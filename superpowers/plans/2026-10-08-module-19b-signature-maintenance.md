# Módulo 19 (19b) — Assinatura digital (maintenance) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito: 19a fechado em `origin/main`** (F-19.1..7 Verified; o interruptor `clinical_record` no catálogo do api). E o **api do 19b** na branch dele, rodando na porta **3037**, com os campos novos no `Maintenance::Schema` (o `schema.graphql` deste app é extraído dele na Task 2) e **mergeado antes** do merge deste. A Task 0 confere os dois.

**Goal:** Lado maintenance de F-19.8: o interruptor `digital_signature` aparece e é ligado/desligado na aba **Funcionalidades** pelo mecanismo genérico do módulo 16 (só se confirma e testa, com o rótulo do pré-requisito `clinical_record_disabled`), e, quando o catálogo da cidade traz `digital_signature`, a aba ganha um quadro **somente leitura** com os prestadores de certificado em nuvem configurados no ambiente (`signatureProviders`: nome, credencial presente, última checagem) e o estado do serviço `signer` (`signerStatus`: alcançável, versão, atualização das LCRs).

**Architecture:** Nada muda no liga/desliga: `FeaturesTab` já lista o catálogo que o api devolve e chama `setCityFeature`. O que entra: dois rótulos de pré-requisito em `src/lib/features.ts` (`clinical_record_disabled` do 19b e `record_mode_not_record` do 19a, que o maintenance ainda mostrava cru); as regras de exibição da assinatura em `src/lib/signature.ts` (nome do prestador, estado do prestador, estado do `signer`, LCR com mais de 24 h); e um componente próprio, `SignaturePlatformPanel`, com **consulta GraphQL própria** (`query SignaturePlatform`, campos de plataforma na raiz `Query`), montado dentro da `FeaturesTab` só quando o catálogo da cidade traz `digital_signature`. Consulta própria pelo mesmo motivo do módulo 16: contra um api sem o 19b, a validação recusa o documento inteiro; assim só o quadro novo explica a ordem de deploy, e a aba, o liga/desliga e o resto da ficha seguem.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, GraphQL Codegen (client preset, `src/gql/` gerado e commitado), Vitest 2 + Testing Library 16 + user-event 14 (jsdom, sem jest-dom, `vi.stubGlobal("fetch")`).

**Spec:** `docs/superpowers/specs/2026-10-08-module-19b-digital-signature-design.md` (§3 e §9 "Maintenance" são deste plano; §13 F-19.8), `docs/adr/0032.md` e o contrato `docs/superpowers/plans/2026-10-08-module-19b-signature-contracts.md` §8 (a forma exata dos tipos GraphQL está na seção "Divergências propostas ao contrato", M1).

## Global Constraints

- Interruptor (contrato §8; spec §3): `digital_signature` pelo mecanismo genérico (`City.features`, `setCityFeature`), `requires: ["feature:clinical_record"]`; faltando, `missing: ["clinical_record_disabled"]`. Ligar não exige o pré-requisito: fica "ligada, falta pré-requisito" (três estados, como no módulo 16). Nenhuma mudança em `FeaturesTab` no liga/desliga.
- Leitura nova (contrato §8, tipos da Divergência M1), na raiz `Query`, de plataforma, sem segredo:
  ```graphql
  signatureProviders: [SignatureProvider!]!   # SignatureProvider { key: String!, configured: Boolean!, lastCheckAt: ISO8601DateTime, lastCheckOk: Boolean }
  signerStatus: SignerStatus!                 # SignerStatus { reachable: Boolean!, version: String, crlUpdatedAt: ISO8601DateTime }
  ```
- Prestadores (spec §3): catálogo fixo `vidaas`, `birdid`, `safeid`, `neoid`, `remoteid`; nomes "VIDaaS", "BirdID", "SafeID", "NeoID", "RemoteID"; chave desconhecida (api mais novo) aparece crua. Sem credencial = "não habilitado". O maintenance **só lê**: nada de `client_id`, `client_secret`, `base_url` nem `SIGNER_TOKEN` na tela, na consulta ou no schema.
- Estados na tela: prestador — "sem credencial — não habilitado" / "habilitado, nunca checado" / "habilitado, última checagem ok" / "habilitado, última checagem falhou"; `signer` — "inalcançável" / "no ar, LCRs nunca atualizadas" / "no ar, LCRs com mais de 24 h" / "no ar". Valor nulo nunca vira `undefined`/`null` na tela ("nunca", "—").
- Os prestadores e o `signer` valem para **todas as cidades do ambiente**: o quadro diz isso no título; ele aparece na aba da cidade porque é onde se decide ligar o interruptor.
- **Ordem de deploy: api → dashboard → maintenance** (contrato §12). Contra um api sem o 19b, só o quadro novo mostra o aviso; e ele só aparece quando o catálogo traz `digital_signature`, que também é do api do 19b.
- Interface tem ciclo próprio: componentes e estilo existentes (`Panel`, `DataTable`, `Tag`, `Button`, `ErrorState`, `EmptyState`), sem redesign.
- Nunca escreva à mão em arquivo gerado (`schema.graphql`, `src/gql/*`): eles saem do api em execução e do codegen.
- Nunca `git add -A` nem `git add .`: o worktree tem o symlink de `node_modules`. Adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git`. Merge, push e tag só com autorização explícita do usuário.

## Ambiente de execução

- Worktree `apps/maintenance/.claude/mod19b`, branch `feat/mod-19b-signature` a partir de `origin/main` (Task 0). O `.gitignore` do maintenance ignora `node_modules` (também o symlink), mas **não** ignora `.claude/`: o checkout principal lista `.claude/` como não rastreado. É esperado; não adicione.
- Todos os caminhos de arquivo das tasks são relativos ao worktree; os comandos partem da raiz do monorepo (`/Users/eduardovrocha/Development/ioit.solutions/rota-saude`).
- Testes: `cd apps/maintenance/.claude/mod19b && npx vitest run <arquivos>` (o `vitest.config.ts` exclui `e2e/**`).
- Tipos e codegen: `cd apps/maintenance/.claude/mod19b && npx tsc --noEmit` e `npm run codegen`.
- Ambiente de teste: `environment: "jsdom"`, `globals: false` (todo teste de componente chama `afterEach(cleanup)`), sem jest-dom (use `not.toBeNull()`, `toBeNull()`, `.textContent`). O `vitest.config.ts` não fixa `TZ`: datas saem de `fmtWhen` (`src/lib/analytics.ts`), que já formata em `America/Sao_Paulo` ("05/10/2026, 10:00").
- **`npm run schema:pull` não serve no worktree** (o script sobe três níveis a partir de `scripts/` e escreveria no checkout principal). A Task 2 extrai o SDL pelo `docker compose exec` direto, com destino no worktree.
- Login de dev (Task 3): `http://maintenance.localhost:5177`, nunca `localhost:5177`. A conta é o `Maintainer` `dev@local` semeado pelo `db:seed`; o usuário faz o login.

## Review Focus

1. **Maintenance novo contra api sem o 19b.** A validação recusa `query SignaturePlatform` inteira (`undefinedField`). O quadro explica a ordem de deploy; a aba Funcionalidades, o liga/desliga e as outras abas seguem. Teste: Task 2, "api sem o 19b: o quadro explica a ordem de deploy e a aba segue".
2. **Ligar `digital_signature` numa cidade sem o prontuário ligado.** O api aceita (ligada ≠ utilizável); a tela diz que ficou ligada **mas não utilizável** e que falta o prontuário da atenção primária — em português, não `clinical_record_disabled` cru. Testes: Task 1, "pré-requisitos do prontuário e da assinatura têm frase"; Task 2, "digital_signature pelo mecanismo genérico: falta o prontuário e liga com dois cliques".
3. **`signer` fora do ar ou com LCR velha.** Com o `signer` inalcançável, nada é assinado (as consultas ficam pendentes); com a LCR de dias atrás, a validação de revogação fica fraca. O quadro precisa dizer "inalcançável" ou "LCRs com mais de 24 h" em destaque, não "no ar". Testes: Task 1, "signer: inalcançável, LCR nunca atualizada, velha e em dia"; Task 2, "signer inalcançável e prestador com checagem falha aparecem em destaque".
4. **Segredo na tela.** O quadro mostra só nome, se há credencial e a última checagem — nunca `client_id`, `client_secret`, `base_url` ou token, nem `undefined`/`null`. Testes: Task 2, "lista prestadores e signer sem segredo e sem undefined/null" e Step 1 (o SDL não tem campo de segredo).
5. **Prestador desconhecido ou campos nulos** (api mais novo com `pscnovo`; prestador nunca checado; `signer` sem versão). Aparece cru, "nunca" ou "—". Testes: Task 1, "nome com a chave; prestador desconhecido aparece cru" e "nulos viram 'nunca' e '—'".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | conferência do 19a e do api do 19b, worktree e base | 0 |
| `src/lib/features.ts`, `src/lib/features.test.ts`, `src/lib/signature.ts`, `src/lib/signature.test.ts` | rótulos dos pré-requisitos; nome e estado dos prestadores; estado do `signer` | 1 |
| `schema.graphql`, `src/gql/*` (gerados) | tipos novos do api | 2 |
| `src/screens/SignaturePlatformPanel.tsx`, `src/screens/SignaturePlatformPanel.test.tsx`, `src/screens/FeaturesTab.tsx`, `src/screens/FeaturesTab.test.tsx`, `README.md` | consulta `SignaturePlatform`, o quadro, montado na aba quando há `digital_signature`; o interruptor pelo mecanismo genérico | 2 |
| — | suíte, codegen:check, build, revisão e prova no navegador | 3 |

**Estratégia de teste:** regras de exibição com tabela de casos (Task 1); o quadro e a aba com `fetch` falso que responde por nome de operação (`query CityFeatures`, `query SignaturePlatform`, `mutation SetCityFeature`), cobrindo o caminho feliz, o api sem o 19b, o `signer` fora do ar, os nulos e a ausência de segredo (Task 2); o interruptor `digital_signature` testado pelo fluxo existente de dois cliques, sem mudar o código do liga/desliga. A prova final é no navegador, contra o api do 19b (Task 3).

---

### Task 0: Conferência do 19a e do api do 19b, worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do api com o 19a; a branch do api do 19b rodando na 3037.
- Produces: worktree `apps/maintenance/.claude/mod19b` na branch `feat/mod-19b-signature`; a base anotada.

- [ ] **Step 1: Confira o 19a em `origin/main` do api e o api do 19b no ar**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude
/opt/homebrew/bin/git -C apps/api fetch origin
/opt/homebrew/bin/git -C apps/api grep -n "clinical_record" origin/main -- app/services/platform | head -5
docker compose exec -T -w /rails/.claude/mod19b api bin/rails runner \
  'puts Platform::Features::CATALOG.map { |f| f[:key] rescue f.key }.inspect' 2>/dev/null || true
```

Expected: ao menos uma linha com `clinical_record` no catálogo de `origin/main` (19a) e, no api do 19b, a lista do catálogo com `digital_signature`. Se o 19a não estiver em `origin/main`, **pare** (o 19b só executa depois do 19a fechado). Se o comando do catálogo falhar porque o `CATALOG` tem outra forma, confira o nome no `app/services/platform/` do worktree do api (`grep -rn "digital_signature" apps/api/.claude/mod19b/app/services/platform`); o que importa é `digital_signature` estar lá.

- [ ] **Step 2: Crie o worktree e ligue o `node_modules`**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/maintenance
/opt/homebrew/bin/git fetch origin
/opt/homebrew/bin/git log --oneline -1 origin/main
/opt/homebrew/bin/git worktree add .claude/mod19b -b feat/mod-19b-signature origin/main
ln -s ../../node_modules .claude/mod19b/node_modules
cd .claude/mod19b && npx vitest run 2>&1 | tail -3 && npx tsc --noEmit && echo tsc-ok
```

Expected: `origin/main` em `3f7a758 fix: report the outcome when the feature switch tab gets no feature or a city field error` (ou mais novo; se `src/screens/FeaturesTab.tsx` ou `src/lib/features.ts` mudaram, confira os trechos que as Tasks 1 e 2 trocam). Suíte verde e `tsc-ok`. **Anote** a base (arquivos e testes): a Task 3 espera a base mais **2 arquivos de teste novos** (`signature.test.ts` e `SignaturePlatformPanel.test.tsx`).

---

### Task 1: Rótulos dos pré-requisitos e regras de exibição da assinatura

**Files:**
- Modify: `src/lib/features.ts`, `src/lib/features.test.ts`
- Create: `src/lib/signature.ts`
- Test: `src/lib/signature.test.ts`

**Interfaces:**
- Consumes: `Tone` de `src/theme/tokens.ts`; `fmtWhen` de `src/lib/analytics.ts`; `missingText` (existente).
- Produces:
  - `missingText("clinical_record_disabled")` → "prontuário da atenção primária (clinical_record) desligado"; `missingText("record_mode_not_record")` → "modo de prontuário diferente de record";
  - em `src/lib/signature.ts`:
    ```ts
    export function providerName(key: string): string;                       // "VIDaaS (vidaas)"; desconhecido → cru
    export type ProviderView = { key: string; configured: boolean; lastCheckAt?: string | null; lastCheckOk?: boolean | null };
    export function providerStatus(p: ProviderView): { label: string; tone: Tone };
    export function lastCheckText(p: ProviderView): string;                  // fmtWhen ou "nunca"
    export type SignerView = { reachable: boolean; version?: string | null; crlUpdatedAt?: string | null };
    export const CRL_STALE_MS: number;                                       // 24 h
    export function signerStatus(s: SignerView, nowMs: number): { label: string; tone: Tone };
    export function signerDetail(s: SignerView): string;
    ```

- [ ] **Step 1: Write the failing test**

Em `src/lib/features.test.ts`, dentro de `describe("missingText e missingSummary")`:

```ts
  it("pré-requisitos do prontuário e da assinatura têm frase", () => {
    expect(missingText("record_mode_not_record")).toBe("modo de prontuário diferente de record");
    expect(missingText("clinical_record_disabled")).toBe("prontuário da atenção primária (clinical_record) desligado");
    expect(missingSummary([ "clinical_record_disabled" ])).toBe("prontuário da atenção primária (clinical_record) desligado");
  });
```

```ts
// src/lib/signature.test.ts
import { describe, expect, it } from "vitest";
import { CRL_STALE_MS, lastCheckText, providerName, providerStatus, signerDetail, signerStatus } from "./signature";

const NOW = Date.parse("2026-10-08T13:00:00Z");

describe("prestadores", () => {
  it("nome com a chave; prestador desconhecido aparece cru", () => {
    expect(providerName("vidaas")).toBe("VIDaaS (vidaas)");
    expect(providerName("remoteid")).toBe("RemoteID (remoteid)");
    expect(providerName("pscnovo")).toBe("pscnovo");
  });

  it("estado: sem credencial, nunca checado, checagem ok e checagem falha", () => {
    expect(providerStatus({ key: "safeid", configured: false, lastCheckAt: null, lastCheckOk: null }))
      .toEqual({ label: "sem credencial — não habilitado", tone: "neutral" });
    expect(providerStatus({ key: "birdid", configured: true, lastCheckAt: null, lastCheckOk: null }))
      .toEqual({ label: "habilitado, nunca checado", tone: "info" });
    expect(providerStatus({ key: "vidaas", configured: true, lastCheckAt: "2026-10-08T12:00:00Z", lastCheckOk: true }))
      .toEqual({ label: "habilitado, última checagem ok", tone: "ok" });
    expect(providerStatus({ key: "neoid", configured: true, lastCheckAt: "2026-10-08T12:00:00Z", lastCheckOk: false }))
      .toEqual({ label: "habilitado, última checagem falhou", tone: "down" });
  });

  it("nulos viram 'nunca' e '—'", () => {
    expect(lastCheckText({ key: "birdid", configured: true, lastCheckAt: null })).toBe("nunca");
    expect(lastCheckText({ key: "vidaas", configured: true, lastCheckAt: "2026-10-08T12:00:00Z" })).toBe("08/10/2026, 09:00");
    expect(signerDetail({ reachable: true, version: null, crlUpdatedAt: null })).toBe("versão — · LCRs atualizadas em —");
  });
});

describe("signer", () => {
  it("signer: inalcançável, LCR nunca atualizada, velha e em dia", () => {
    expect(signerStatus({ reachable: false }, NOW)).toEqual({ label: "inalcançável", tone: "down" });
    expect(signerStatus({ reachable: true, version: "1.0.0", crlUpdatedAt: null }, NOW))
      .toEqual({ label: "no ar, LCRs nunca atualizadas", tone: "warn" });
    expect(signerStatus({ reachable: true, version: "1.0.0", crlUpdatedAt: new Date(NOW - CRL_STALE_MS - 60_000).toISOString() }, NOW))
      .toEqual({ label: "no ar, LCRs com mais de 24 h", tone: "warn" });
    expect(signerStatus({ reachable: true, version: "1.0.0", crlUpdatedAt: "2026-10-08T09:00:00Z" }, NOW))
      .toEqual({ label: "no ar", tone: "ok" });
  });

  it("detalhe: versão e hora das LCRs; inalcançável diz o efeito", () => {
    expect(signerDetail({ reachable: true, version: "1.2.0", crlUpdatedAt: "2026-10-08T09:00:00Z" }))
      .toBe("versão 1.2.0 · LCRs atualizadas em 08/10/2026, 06:00");
    expect(signerDetail({ reachable: false, version: null, crlUpdatedAt: null }))
      .toBe("o api não alcançou o serviço signer — as assinaturas ficam pendentes até ele voltar");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/maintenance/.claude/mod19b && npx vitest run src/lib/features.test.ts src/lib/signature.test.ts`
Expected: FAIL — `expected 'record_mode_not_record' to be 'modo de prontuário diferente de record'` e `Failed to resolve import "./signature"`.

- [ ] **Step 3: Write minimal implementation**

Em `src/lib/features.ts`:

```diff
--- a/src/lib/features.ts
+++ b/src/lib/features.ts
@@ const MISSING_LABELS: Record<string, string> = {
   "credential_missing:cadsus": "credencial CADSUS não cadastrada",
   "credential_unauthorized:cadsus": "credencial CADSUS recusada no último teste",
+  // Módulo 19 (19a): clinical_record só é utilizável no modo record.
+  record_mode_not_record: "modo de prontuário diferente de record",
+  // Módulo 19b: digital_signature exige clinical_record ligado e utilizável.
+  clinical_record_disabled: "prontuário da atenção primária (clinical_record) desligado",
   // O api não conseguiu ler o banco da cidade: usable vem false sem que se
```

```ts
// src/lib/signature.ts
// Regras de exibição da assinatura digital no maintenance (módulo 19b, F-19.8;
// spec §3 e §9; contrato §8). Só leitura: os prestadores e o serviço `signer`
// são da plataforma (todas as cidades do ambiente). Nenhum segredo passa por
// aqui — o api só diz se a credencial existe e como foi a última checagem.
import type { Tone } from "../theme/tokens";
import { fmtWhen } from "./analytics";

const PROVIDER_NAMES: Record<string, string> = {
  vidaas: "VIDaaS", birdid: "BirdID", safeid: "SafeID", neoid: "NeoID", remoteid: "RemoteID"
};

export function providerName(key: string): string {
  const name = PROVIDER_NAMES[key];
  return name ? `${name} (${key})` : key;
}

export type ProviderView = { key: string; configured: boolean; lastCheckAt?: string | null; lastCheckOk?: boolean | null };

export function providerStatus(provider: ProviderView): { label: string; tone: Tone } {
  if (!provider.configured) return { label: "sem credencial — não habilitado", tone: "neutral" };
  if (provider.lastCheckOk === true) return { label: "habilitado, última checagem ok", tone: "ok" };
  if (provider.lastCheckOk === false) return { label: "habilitado, última checagem falhou", tone: "down" };
  return { label: "habilitado, nunca checado", tone: "info" };
}

export function lastCheckText(provider: ProviderView): string {
  return provider.lastCheckAt ? fmtWhen(provider.lastCheckAt) : "nunca";
}

export type SignerView = { reachable: boolean; version?: string | null; crlUpdatedAt?: string | null };

// LCR com mais de um dia: a verificação de revogação perde força (spec §5
// "Validação"). É aviso de tela, não regra do api.
export const CRL_STALE_MS = 24 * 60 * 60 * 1000;

export function signerStatus(signer: SignerView, nowMs: number): { label: string; tone: Tone } {
  if (!signer.reachable) return { label: "inalcançável", tone: "down" };
  if (!signer.crlUpdatedAt) return { label: "no ar, LCRs nunca atualizadas", tone: "warn" };
  const at = Date.parse(signer.crlUpdatedAt);
  if (Number.isNaN(at) || nowMs - at > CRL_STALE_MS) return { label: "no ar, LCRs com mais de 24 h", tone: "warn" };
  return { label: "no ar", tone: "ok" };
}

export function signerDetail(signer: SignerView): string {
  if (!signer.reachable) return "o api não alcançou o serviço signer — as assinaturas ficam pendentes até ele voltar";
  return `versão ${signer.version ?? "—"} · LCRs atualizadas em ${fmtWhen(signer.crlUpdatedAt)}`;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/maintenance/.claude/mod19b && npx vitest run src/lib/features.test.ts src/lib/signature.test.ts && npx tsc --noEmit`
Expected: PASS; `tsc` limpo.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/maintenance/.claude/mod19b add src/lib/features.ts src/lib/features.test.ts src/lib/signature.ts src/lib/signature.test.ts
/opt/homebrew/bin/git -C apps/maintenance/.claude/mod19b commit -m "feat: label the clinical record and digital signature prerequisites and signature platform states

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Schema, codegen, quadro de prestadores e `signer`, e o interruptor na aba

**Files:**
- Create: `src/screens/SignaturePlatformPanel.tsx`
- Modify: `src/screens/FeaturesTab.tsx`, `src/screens/FeaturesTab.test.tsx`, `README.md`
- Test: `src/screens/SignaturePlatformPanel.test.tsx`
- Gerados: `schema.graphql`, `src/gql/*`

**Interfaces:**
- Consumes: tudo de `src/lib/signature.ts` (Task 1); `gql` (`src/lib/api.ts`); `graphql` (`src/gql`); `GraphQLRefusal` (`src/lib/errors.ts`); `Panel`, `Button`, `Tag`, `DataTable`, `EmptyState`, `ErrorState`. Do api do 19b: `Query.signatureProviders`, `Query.signerStatus` (contrato §8, tipos da Divergência M1); `digital_signature` no catálogo de `City.features`.
- Produces: `export function SignaturePlatformPanel(): JSX.Element` — `Panel` "Assinatura digital — prestadores e signer (todas as cidades do ambiente)" com "atualizar", a tabela (Prestador, Estado, Última checagem) e o `signer` (estado e detalhe); operação com nome fixo `query SignaturePlatform`, chave `[ "platform", "signature" ]`. `FeaturesTab` monta o quadro depois de "Funcionalidades" quando `city.features` traz `digital_signature`.
- **Depende do plano do api do 19b** ter os dois campos no `Maintenance::Schema` na branch em execução.

- [ ] **Step 1: Extraia o schema do api do 19b**

O api do 19b roda na worktree dele (`apps/api/.claude/mod19b`, com o `config/master.key` copiado, como diz o plano do api); o container monta `apps/api` em `/rails`. Da raiz do monorepo:

```bash
docker compose exec -T -w /rails/.claude/mod19b api bin/rails runner 'puts Maintenance::Schema.to_definition' \
  > apps/maintenance/.claude/mod19b/schema.graphql
# (se o api do 19b já estiver mergeado em main, tire o `-w /rails/.claude/mod19b`)

grep -n -E "^type SignatureProvider|^type SignerStatus|  signatureProviders: \[SignatureProvider!\]!|  signerStatus: SignerStatus!" \
  apps/maintenance/.claude/mod19b/schema.graphql
sed -n '/^type SignatureProvider {/,/^}/p;/^type SignerStatus {/,/^}/p' apps/maintenance/.claude/mod19b/schema.graphql
grep -n -i -E "secret|client_?id|base_?url|token" apps/maintenance/.claude/mod19b/schema.graphql | grep -i -E "signature|signer" ; echo "exit $?"
```

Expected: as quatro linhas; os tipos impressos exatamente como a Divergência M1 — `SignatureProvider { configured: Boolean!, key: String!, lastCheckAt: ISO8601DateTime, lastCheckOk: Boolean }` e `SignerStatus { crlUpdatedAt: ISO8601DateTime, reachable: Boolean!, version: String }` (o SDL ordena os campos por nome) — e o último `grep` sem linha (`exit 1`): nenhum campo de segredo nos tipos da assinatura.

Se faltar algum, **pare e reporte**: não escreva à mão em arquivo gerado. Se vierem com nome, tipo ou nulidade diferente da M1, pare e reporte também: um dos dois planos precisa mudar antes de seguir.

Depois confira que só o schema mudou e gere os tipos:

```bash
cd apps/maintenance/.claude/mod19b && /opt/homebrew/bin/git diff --stat
```

Expected: só `schema.graphql`, com os tipos do 19b (e o que o api do 19a/19b tiver acrescentado; se aparecerem outras mudanças, reporte as linhas antes de seguir). O codegen roda no Step 3, depois do documento novo existir.

- [ ] **Step 2: Write the failing test**

```tsx
// src/screens/SignaturePlatformPanel.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, render, screen, waitFor, within } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";
import { SignaturePlatformPanel } from "./SignaturePlatformPanel";

afterEach(() => { cleanup(); vi.useRealTimers(); vi.unstubAllGlobals(); });

function reply(status: number, body: unknown) {
  return new Response(JSON.stringify(body), { status, headers: { "Content-Type": "application/json" } });
}
function bodyOf(call: unknown[]): { query: string } {
  return JSON.parse((call[1] as RequestInit).body as string);
}

const PROVIDERS = [
  { key: "vidaas", configured: true, lastCheckAt: "2026-10-08T12:00:00Z", lastCheckOk: true },
  { key: "birdid", configured: true, lastCheckAt: null, lastCheckOk: null },
  { key: "safeid", configured: false, lastCheckAt: null, lastCheckOk: null },
  { key: "pscnovo", configured: true, lastCheckAt: "2026-10-08T11:00:00Z", lastCheckOk: false }
];
const SIGNER = { reachable: true, version: "1.2.0", crlUpdatedAt: "2026-10-08T09:00:00Z" };

function platformReply(providers: unknown[] = PROVIDERS, signer: unknown = SIGNER) {
  return reply(200, { data: { signatureProviders: providers, signerStatus: signer } });
}

function renderPanel() {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  function wrapper({ children }: { children: ReactNode }) {
    return <QueryClientProvider client={client}>{children}</QueryClientProvider>;
  }
  return render(<SignaturePlatformPanel />, { wrapper });
}

describe("SignaturePlatformPanel", () => {
  let fetchMock: ReturnType<typeof vi.fn>;

  beforeEach(() => {
    vi.useFakeTimers({ toFake: [ "Date" ] });
    vi.setSystemTime(new Date("2026-10-08T13:00:00Z"));
    fetchMock = vi.fn();
    vi.stubGlobal("fetch", fetchMock);
  });

  it("lista prestadores e signer sem segredo e sem undefined/null", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(platformReply()));
    renderPanel();

    expect(await screen.findByText("VIDaaS (vidaas)")).not.toBeNull();
    expect(screen.getByText("habilitado, última checagem ok")).not.toBeNull();
    expect(screen.getByText("08/10/2026, 09:00")).not.toBeNull();
    expect(screen.getByText("BirdID (birdid)")).not.toBeNull();
    expect(screen.getByText("habilitado, nunca checado")).not.toBeNull();
    expect(screen.getAllByText("nunca")).toHaveLength(2);
    expect(screen.getByText("SafeID (safeid)")).not.toBeNull();
    expect(screen.getByText("sem credencial — não habilitado")).not.toBeNull();
    expect(screen.getByText("pscnovo")).not.toBeNull();

    const signer = screen.getByRole("group", { name: "serviço signer" });
    expect(within(signer).getByText("no ar")).not.toBeNull();
    expect(within(signer).getByText("versão 1.2.0 · LCRs atualizadas em 08/10/2026, 06:00")).not.toBeNull();

    expect(bodyOf(fetchMock.mock.calls[0]).query).toMatch(/query SignaturePlatform/);
    expect(document.body.textContent).not.toMatch(/undefined|null|secret|client_id|base_url|token/i);
  });

  it("signer inalcançável e prestador com checagem falha aparecem em destaque", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(platformReply(PROVIDERS, { reachable: false, version: null, crlUpdatedAt: null })));
    renderPanel();
    const signer = await screen.findByRole("group", { name: "serviço signer" });
    expect(within(signer).getByText("inalcançável")).not.toBeNull();
    expect(within(signer).getByText("o api não alcançou o serviço signer — as assinaturas ficam pendentes até ele voltar")).not.toBeNull();
    expect(screen.getByText("habilitado, última checagem falhou")).not.toBeNull();
  });

  it("LCR com mais de 24 h vira aviso", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(platformReply(PROVIDERS, { ...SIGNER, crlUpdatedAt: "2026-10-06T09:00:00Z" })));
    renderPanel();
    expect(await screen.findByText("no ar, LCRs com mais de 24 h")).not.toBeNull();
  });

  it("nenhum prestador no catálogo: diz", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(platformReply([])));
    renderPanel();
    expect(await screen.findByText("nenhum prestador no catálogo")).not.toBeNull();
  });

  it("api sem o 19b: explica a ordem de deploy", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(reply(200, {
      errors: [ {
        message: "Field 'signatureProviders' doesn't exist on type 'Query'",
        extensions: { code: "undefinedField", typeName: "Query", fieldName: "signatureProviders" }
      } ]
    })));
    renderPanel();
    expect((await screen.findByRole("alert")).textContent)
      .toMatch(/o api do módulo 19b precisa subir antes do maintenance/);
  });

  it("atualizar relê", async () => {
    const user = userEvent.setup();
    fetchMock.mockImplementation(() => Promise.resolve(platformReply()));
    renderPanel();
    await screen.findByText("VIDaaS (vidaas)");
    await user.click(screen.getByRole("button", { name: "atualizar" }));
    await waitFor(() => expect(fetchMock).toHaveBeenCalledTimes(2));
  });
});
```

E, no fim de `src/screens/FeaturesTab.test.tsx` (usa os `reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab`, `Feature` e `LEDI_OFF` que o arquivo já tem):

```tsx
describe("FeaturesTab — assinatura digital (módulo 19b)", () => {
  let fetchMock: ReturnType<typeof vi.fn>;

  beforeEach(() => {
    fetchMock = vi.fn();
    vi.stubGlobal("fetch", fetchMock);
  });

  afterEach(() => vi.unstubAllGlobals());

  const SIGNATURE_OFF: Feature = {
    key: "digital_signature", description: "Assinatura digital ICP-Brasil da consulta", enabled: false, usable: false,
    missing: [ "clinical_record_disabled" ], changedAt: null, changedBy: null
  };
  const platform = () => reply(200, { data: {
    signatureProviders: [ { key: "vidaas", configured: true, lastCheckAt: null, lastCheckOk: null } ],
    signerStatus: { reachable: true, version: "1.0.0", crlUpdatedAt: new Date().toISOString() }
  } });

  it("digital_signature pelo mecanismo genérico: falta o prontuário e liga com dois cliques", async () => {
    const user = userEvent.setup();
    const turnedOn: Feature = { ...SIGNATURE_OFF, enabled: true, changedAt: "2026-10-08T13:00:00Z", changedBy: "dev@local" };
    let listed = [ SIGNATURE_OFF ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("query SignaturePlatform")) return Promise.resolve(platform());
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [ turnedOn ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: turnedOn } } }));
      }
      return Promise.resolve(featuresReply(listed, { recordMode: "record" }));
    });
    renderTab();

    expect(await screen.findByText("digital_signature")).not.toBeNull();
    expect(screen.getByText("prontuário da atenção primária (clinical_record) desligado")).not.toBeNull();

    await user.click(screen.getByRole("button", { name: "ligar" }));
    expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(0);
    await user.click(screen.getByRole("button", { name: "confirmar: ligar digital_signature" }));
    await waitFor(() => expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(1));
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[0]).variables)
      .toEqual({ citySlug: "sp", key: "digital_signature", enabled: true });
    expect((await screen.findByRole("status")).textContent).toBe(
      "digital_signature ligada, mas ainda não utilizável — falta: prontuário da atenção primária (clinical_record) desligado."
    );
  });

  it("com digital_signature no catálogo mostra os prestadores do ambiente; sem ela, nem consulta", async () => {
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("query SignaturePlatform")) return Promise.resolve(platform());
      return Promise.resolve(featuresReply([ SIGNATURE_OFF ]));
    });
    renderTab();
    expect(await screen.findByRole("region", { name: "Assinatura digital — prestadores e signer (todas as cidades do ambiente)" }))
      .not.toBeNull();
    expect(await screen.findByText("VIDaaS (vidaas)")).not.toBeNull();
    cleanup();

    fetchMock.mockReset();
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply([ LEDI_OFF ])));
    renderTab();
    await screen.findByText("ledi_export");
    expect(calls(fetchMock, "query SignaturePlatform")).toHaveLength(0);
  });

  it("api sem o 19b: o quadro explica a ordem de deploy e a aba segue", async () => {
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("query SignaturePlatform")) {
        return Promise.resolve(reply(200, { errors: [ {
          message: "Field 'signatureProviders' doesn't exist on type 'Query'",
          extensions: { code: "undefinedField", typeName: "Query", fieldName: "signatureProviders" }
        } ] }));
      }
      return Promise.resolve(featuresReply([ SIGNATURE_OFF ]));
    });
    renderTab();
    expect((await screen.findByRole("alert")).textContent).toMatch(/o api do módulo 19b precisa subir antes do maintenance/);
    expect(screen.getByText("digital_signature")).not.toBeNull();
    expect(screen.getByRole("button", { name: "ligar" })).not.toBeNull();
  });
});
```

(Os imports do topo de `FeaturesTab.test.tsx` — `cleanup`, `screen`, `waitFor`, `userEvent`, `vi`, `beforeEach`, `afterEach` — já estão lá. A região com o título é a `section` que o `SignaturePlatformPanel` põe em volta do `Panel`, que não tem `aria-label`.)

- [ ] **Step 3: Run test to verify it fails**

Run: `cd apps/maintenance/.claude/mod19b && npx vitest run src/screens/SignaturePlatformPanel.test.tsx src/screens/FeaturesTab.test.tsx`
Expected: FAIL — `Failed to resolve import "./SignaturePlatformPanel"`; na aba, o quadro não aparece (`Unable to find role="region" and name "Assinatura digital — …"`). O teste do liga/desliga de `digital_signature` já passa (mecanismo genérico, rótulo da Task 1): é a confirmação de F-19.8 no maintenance.

- [ ] **Step 4: Write minimal implementation**

```tsx
// src/screens/SignaturePlatformPanel.tsx
// Prestadores de certificado em nuvem e serviço `signer` (módulo 19b, F-19.8;
// spec §3 e §9; contrato §8). Só leitura e de PLATAFORMA: vale para todas as
// cidades do ambiente. As credenciais ficam nas credenciais cifradas do api;
// aqui só aparece se existem e como foi a última checagem.
//
// Consulta PRÓPRIA, como a aba Funcionalidades no módulo 16: contra um api
// sem o 19b a validação recusa o documento inteiro, e só este quadro cai.
import { useQuery } from "@tanstack/react-query";
import type { CSSProperties } from "react";
import { gql } from "../lib/api";
import { graphql } from "../gql";
import { GraphQLRefusal } from "../lib/errors";
import {
  lastCheckText, providerName, providerStatus, signerDetail, signerStatus, type ProviderView
} from "../lib/signature";
import { Panel } from "../components/Panel";
import { Button } from "../components/Button";
import { Tag } from "../components/Tag";
import { DataTable } from "../components/DataTable";
import { EmptyState } from "../components/EmptyState";
import { ErrorState } from "../components/ErrorState";

const SignaturePlatformQuery = graphql(`
  query SignaturePlatform {
    signatureProviders { key configured lastCheckAt lastCheckOk }
    signerStatus { reachable version crlUpdatedAt }
  }
`);

const TITLE = "Assinatura digital — prestadores e signer (todas as cidades do ambiente)";
const OLD_API_CODES = new Set([ "undefinedField" ]);

function requestErrorText(err: unknown): string {
  if (err instanceof GraphQLRefusal && OLD_API_CODES.has(err.code)) {
    return "esta API ainda não tem a assinatura digital — o api do módulo 19b precisa subir antes do maintenance";
  }
  return err instanceof Error ? err.message : "erro inesperado";
}

export function SignaturePlatformPanel() {
  const query = useQuery({
    queryKey: [ "platform", "signature" ],
    queryFn: () => gql(SignaturePlatformQuery),
    staleTime: Infinity
  });
  const data = query.data?.data ?? null;
  const providers: ProviderView[] = data?.signatureProviders ?? [];
  const signer = data?.signerStatus ?? null;
  const fieldError = query.data?.fieldErrors[0];
  const status = signer ? signerStatus(signer, Date.now()) : null;

  return (
    <section aria-label={TITLE}>
      <Panel title={TITLE} actions={<Button onClick={() => void query.refetch()} busy={query.isFetching}>atualizar</Button>}>
        <p style={note}>
          As credenciais dos prestadores ficam nas credenciais cifradas do api, por ambiente; aqui só aparece se existem.
          Prestador sem credencial não é oferecido aos profissionais.
        </p>
        {query.isPending && <p>carregando…</p>}
        {query.isError && <ErrorState message={requestErrorText(query.error)} />}
        {fieldError && <ErrorState message={`${fieldError.code} — ${fieldError.message}`} />}
        {data && (
          <>
            {providers.length === 0 ? <EmptyState message="nenhum prestador no catálogo" /> : (
              <DataTable<ProviderView>
                columns={[
                  { key: "provider", label: "Prestador", render: (p) => providerName(p.key) },
                  {
                    key: "state", label: "Estado",
                    render: (p) => {
                      const s = providerStatus(p);
                      return <Tag tone={s.tone}>{s.label}</Tag>;
                    }
                  },
                  { key: "lastCheck", label: "Última checagem", render: (p) => lastCheckText(p) }
                ]}
                rows={providers}
                rowKey={(p) => p.key}
              />
            )}
            {signer && status && (
              <div role="group" aria-label="serviço signer" style={signerRow}>
                <strong style={{ fontSize: 12.5 }}>Serviço signer</strong>
                <Tag tone={status.tone}>{status.label}</Tag>
                <span style={note}>{signerDetail(signer)}</span>
              </div>
            )}
          </>
        )}
      </Panel>
    </section>
  );
}

const note: CSSProperties = { margin: 0, fontSize: 11.5, color: "var(--ink3)" };
const signerRow: CSSProperties = { display: "flex", gap: 8, alignItems: "center", flexWrap: "wrap" };
```

(O `Panel` do maintenance não põe `aria-label`; a `section` de fora dá o nome da região aos testes e aos leitores de tela, sem mexer no componente comum.)

Em `src/screens/FeaturesTab.tsx`:

```diff
--- a/src/screens/FeaturesTab.tsx
+++ b/src/screens/FeaturesTab.tsx
@@
 import { EmptyState } from "../components/EmptyState";
 import { ErrorState } from "../components/ErrorState";
+import { SignaturePlatformPanel } from "./SignaturePlatformPanel";
@@
                 rows={city.features}
                 rowKey={(f) => f.key}
               />
             )}
           </Panel>
+
+          {/* Módulo 19b: prestadores e signer (plataforma), só onde se decide
+              ligar a assinatura digital. Consulta própria (SignaturePlatformPanel). */}
+          {city.features.some((f) => f.key === "digital_signature") && <SignaturePlatformPanel />}
         </>
       )}
```

Gere os tipos:

```bash
cd apps/maintenance/.claude/mod19b && npm run codegen && /opt/homebrew/bin/git status --short
```

Expected: `src/gql/gql.ts` e `src/gql/graphql.ts` mudados (operação `SignaturePlatform` e os tipos `SignatureProvider`/`SignerStatus`), além de `schema.graphql` e dos arquivos das tasks.

Em `README.md`, no fim da seção "### Funcionalidades" (antes do aviso de ordem de deploy dela):

```markdown
**Assinatura digital (módulo 19b).** O interruptor `digital_signature` aparece
na mesma lista e liga como os outros; ele precisa do `clinical_record` ligado
e utilizável (falta: "prontuário da atenção primária (clinical_record)
desligado"). Quando o catálogo da cidade traz `digital_signature`, a aba
mostra também o quadro **Assinatura digital — prestadores e signer**, só
leitura e da plataforma (vale para todas as cidades do ambiente): cada
prestador de certificado em nuvem (VIDaaS, BirdID, SafeID, NeoID, RemoteID),
se a credencial existe no api e como foi a última checagem; e o serviço
`signer` (no ar, versão, última atualização das LCRs — aviso com mais de
24 h). Credenciais e token nunca aparecem aqui. Ordem de deploy:
api → dashboard → maintenance; contra um api sem o 19b, só o quadro mostra o
aviso.
```

- [ ] **Step 5: Run test to verify it passes**

Run: `cd apps/maintenance/.claude/mod19b && npx vitest run src/screens/SignaturePlatformPanel.test.tsx src/screens/FeaturesTab.test.tsx && npx tsc --noEmit && npm run codegen:check`
Expected: PASS (6 testes no quadro; os testes antigos da aba e os 3 novos verdes); `tsc` limpo; `codegen:check` verde.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git -C apps/maintenance/.claude/mod19b add schema.graphql src/gql/gql.ts src/gql/graphql.ts src/screens/SignaturePlatformPanel.tsx src/screens/SignaturePlatformPanel.test.tsx src/screens/FeaturesTab.tsx src/screens/FeaturesTab.test.tsx README.md
/opt/homebrew/bin/git -C apps/maintenance/.claude/mod19b status --short
/opt/homebrew/bin/git -C apps/maintenance/.claude/mod19b commit -m "feat: show signature providers and signer status next to the digital signature switch

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

(Se o codegen mexeu em outro arquivo de `src/gql/`, inclua-o pelo caminho; `status --short` tem de ficar vazio depois do commit.)

---

### Task 3: Suíte, codegen:check, build, revisão e prova no navegador

**Files:** nenhum novo.

**Interfaces:**
- Consumes: as Tasks 1–2 na branch `feat/mod-19b-signature`; o api do 19b rodando na 3037 com a semente.
- Produces: branch pronta para merge (depois do merge do api e do dashboard do 19b).

- [ ] **Step 1: Suíte, tipos, codegen e build**

Run: `cd apps/maintenance/.claude/mod19b && npx vitest run && npx tsc --noEmit && npm run codegen:check && npm run build`
Expected: tudo verde (a CI roda os quatro): a base anotada na Task 0 mais **2 arquivos de teste novos**. Um número que dobra é artefato de build descoberto pelo vitest: pare e reporte.

```bash
cd apps/maintenance/.claude/mod19b && /opt/homebrew/bin/git status --short && /opt/homebrew/bin/git log --stat origin/main..HEAD | grep -c node_modules
```

Expected: `status` vazio e a contagem `0`.

- [ ] **Step 2: Nenhum segredo e nenhum campo novo no topo**

```bash
cd apps/maintenance/.claude/mod19b
grep -n -i -E "secret|client_?id|base_?url|signer_?token" src/screens/SignaturePlatformPanel.tsx src/lib/signature.ts ; echo "exit $?"
sed -n '/const CityHeaderQuery/,/`);/p' src/screens/CityDetail.tsx | grep -n -E "signature|signer" ; echo "exit $?"
```

Expected: os dois `exit 1` (nenhuma menção a segredo; o `CityHeader` não pede campo do 19b).

- [ ] **Step 3: Revisão**

Um subagente revisor lê `origin/main..HEAD` do maintenance contra a spec (§3, §9, §13 F-19.8), o ADR 0032 e o contrato §8. Pontos de atenção:
- o liga/desliga de `digital_signature` é o genérico (nenhuma linha nova no fluxo de `setCityFeature`); o pré-requisito aparece em português;
- `SignaturePlatform` é consulta própria; contra api antigo só o quadro falha; o quadro só monta com `digital_signature` no catálogo;
- nenhum segredo na consulta, no schema ou na tela; nulos viram "nunca"/"—";
- `schema.graphql` e `src/gql/*` iguais ao que o api em execução produz (`codegen:check` verde);
- nenhum `git add -A` no histórico.

- [ ] **Step 4: Prova no navegador (com o usuário)**

Depende do api do 19b na porta **3037** (plano do api do 19b) com o `signer` do compose e as credenciais de dev do PSC falso. O proxy do Vite troca o Host para `maintenance-api.localhost` e injeta o Origin `http://maintenance.localhost:5177`, então o Vite do worktree sobe **na 5177**, num container da rede do compose (para alcançar `http://api:3037`), depois de parar o serviço `maintenance`:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose stop maintenance
docker compose run -d --rm --no-deps --name maintenance-mod19b -p 5177:5177 \
  -w /app/.claude/mod19b -e VITE_API_PROXY_TARGET=http://api:3037 -e VITE_MAINTENANCE_ENV=development \
  maintenance npx vite --port 5177 --host 0.0.0.0
docker logs -f maintenance-mod19b   # espere o "ready"; Ctrl+C sai do log
```

(Se o plano do api subir o servidor do 19b num container próprio, use o nome dele no lugar de `api`.)

Abra `http://maintenance.localhost:5177` (nunca `localhost:5177`). O usuário faz o login do mantenedor; não digite senha nem TOTP. Confira com screenshot:
- Cidades → Curitiba → **Funcionalidades**: `digital_signature` na lista, com a descrição do catálogo; com `clinical_record` desligado, "ligada, falta pré-requisito" / "prontuário da atenção primária (clinical_record) desligado" depois de ligar; com ele ligado e utilizável, "ligada e utilizável";
- "ligar" → "confirmar: ligar digital_signature" → a frase do estado devolvido; "Última mudança" com a hora e `dev@local`; a tela **Auditoria** mostra a mudança;
- o quadro **Assinatura digital — prestadores e signer**: VIDaaS (e o PSC falso de dev) com "habilitado, …", os sem credencial como "sem credencial — não habilitado", e o `signer` "no ar" com versão e hora das LCRs; parar o `signer` (`docker compose stop signer`) e "atualizar" → "inalcançável"; religar (`docker compose start signer`);
- nenhuma credencial na tela.

Depois: `docker rm -f maintenance-mod19b && docker compose start maintenance`. Deixe os interruptores da cidade de dev como estavam antes da prova.

- [ ] **Step 5:** **Pare.** O merge do maintenance só vem depois do merge do api e do dashboard do 19b, e só com autorização explícita do usuário (merge, push e cards, uma etapa de cada vez). Antes do push, confira `origin/main..main` no maintenance e publique só os commits desta entrega. Comandos, quando autorizados:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/maintenance
/opt/homebrew/bin/git fetch origin
/opt/homebrew/bin/git status -sb
/opt/homebrew/bin/git merge --ff-only feat/mod-19b-signature
/opt/homebrew/bin/git log --oneline origin/main..main
/opt/homebrew/bin/git push origin main
/opt/homebrew/bin/git worktree remove .claude/mod19b
/opt/homebrew/bin/git branch -d feat/mod-19b-signature
```

---

## Divergências propostas ao contrato

1. **M1 — Tipos, nulidade e lugar dos campos do §8.** O contrato dá só os nomes (`signatureProviders { key configured lastCheckAt lastCheckOk }`, `signerStatus { reachable version crlUpdatedAt }`). Proposta, na raiz `Query` (são de plataforma, não de cidade; qualquer sessão de manutenção lê):
   ```graphql
   type SignatureProvider { key: String!, configured: Boolean!, lastCheckAt: ISO8601DateTime, lastCheckOk: Boolean }
   type SignerStatus { reachable: Boolean!, version: String, crlUpdatedAt: ISO8601DateTime }
   # Query:
   signatureProviders: [SignatureProvider!]!   # o catálogo fixo inteiro, configurado ou não, na ordem do catálogo
   signerStatus: SignerStatus!                 # nunca levanta: signer fora do ar → reachable: false, version/crlUpdatedAt nulos
   ```
   Recusada a degradação do `signerStatus` (se ele levantar), o erro anularia o `data` inteiro e o quadro mostraria só o código do erro, sem os prestadores.
2. **M2 — O que é a "última checagem" do prestador.** O contrato não diz. Proposta: a última chamada do api ao prestador (localização de certificado, troca de token ou assinatura) com o resultado (`lastCheckOk` = respondeu sem erro de transporte/credencial); `null`/`null` quando nunca houve chamada. Sem uma checagem ativa periódica nesta entrega.
3. **M3 — Rótulo do pré-requisito do 19a.** O plano do api do 19a fixou `record_mode_not_record` no `missing` de `clinical_record` e disse que o maintenance não muda; a tela mostrava o código cru. Este plano acrescenta a frase ("modo de prontuário diferente de record") junto da do 19b (`clinical_record_disabled`). Não muda o contrato, só registra que o rótulo entrou aqui.

## Self-review

- **Cobertura (spec §3, §9 "Maintenance", §13 F-19.8; contrato §8):** interruptor `digital_signature` pelo mecanismo genérico, com o pré-requisito em português — Tasks 1 e 2 (teste do liga/desliga sem mudar o fluxo); prestadores do ambiente (nome, credencial presente, última checagem) — Tasks 1 e 2; estado do `signer` — Tasks 1 e 2; só leitura e sem segredo — Global Constraints, Task 2 (teste e Step 1) e Task 3 (Step 2); ordem de deploy — README (Task 2), consulta própria e Task 3; prova no navegador — Task 3.
- **Placeholders:** nenhum; todo passo de código tem o código. As únicas variáveis são as do ambiente do api do 19b (nome do container se ele não for `api`), que o plano do api define.
- **Tipos:** `ProviderView`/`SignerView` (Task 1) aceitam os tipos gerados da M1 (`lastCheckAt?: Maybe<string>`, `lastCheckOk?: Maybe<boolean>`, `version?: Maybe<string>`, `crlUpdatedAt?: Maybe<string>`); `SignaturePlatformPanel()` é o que a `FeaturesTab` monta; o nome `SignaturePlatform` é o mesmo nos testes das duas telas.
- **Review Focus:** 1 — Task 2 ("api sem o 19b: o quadro explica…" na aba e no quadro); 2 — Task 1 ("pré-requisitos do prontuário e da assinatura têm frase") e Task 2 ("digital_signature pelo mecanismo genérico…"); 3 — Task 1 ("signer: inalcançável, LCR…") e Task 2 ("signer inalcançável e prestador com checagem falha…", "LCR com mais de 24 h vira aviso"); 4 — Task 2 ("lista prestadores e signer sem segredo…") e Step 1; 5 — Task 1 ("nome com a chave; prestador desconhecido aparece cru", "nulos viram 'nunca' e '—'").
