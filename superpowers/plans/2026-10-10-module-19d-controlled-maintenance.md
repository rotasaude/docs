# Módulo 19d — Receita de controlado e SNCR (maintenance) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito: 19c entregue em `origin/main`** (o maintenance do 19c — rótulo de `clinical_documents` e a tela Medicamentos — mergeado depois do api e do dashboard do 19c). A base de hoje é `0f0a779` (19b); o 19c ainda está em execução noutra sessão. A **Task 0** confere que o 19c chegou e **para** se não chegou. Para o schema (Task 2) e a prova (Task 3), o **api do 19d** precisa estar rodando na branch dele (porta **3039**) com `controlled_prescriptions` e `sncr_mock` no catálogo de interruptores e `sncrStatus` no `Maintenance::Schema`; ele é **mergeado antes** deste (contrato §11: contracts → api → dashboard → maintenance).

**Goal:** Lado maintenance de F-19.24: os interruptores `controlled_prescriptions` e `sncr_mock` aparecem e são ligados/desligados na aba **Funcionalidades** pelo mecanismo genérico do módulo 16 (só rótulos da chave e dos pré-requisitos `clinical_documents_disabled` e `controlled_prescriptions_disabled`), com o aviso de numeração simulada no padrão do `signature_psc_mock` do 19b, e um quadro só leitura **SNCR da Anvisa** com `sncrStatus` (configurado, alcançável, última checagem, simulado disponível no ambiente).

**Architecture:** Nada muda no liga/desliga: `FeaturesTab` já lista o catálogo que o api devolve e chama `setCityFeature`; entram só os rótulos em `src/lib/features.ts`. As regras de exibição do SNCR ficam em `src/lib/sncr.ts` (puras: aviso de modo pelo estado **ligado** de `sncr_mock`, estado do SNCR real, texto da última checagem e da disponibilidade do simulado). O quadro `src/screens/SncrPlatformPanel.tsx` tem **consulta GraphQL própria** (`query SncrPlatform`), como o `SignaturePlatformPanel` do 19b: contra um api sem o 19d só ele mostra o aviso da ordem de deploy e a aba segue. A `FeaturesTab` monta o quadro quando o catálogo da cidade traz `controlled_prescriptions` e mostra o aviso de numeração simulada quando `sncr_mock` está ligado. Em produção o api nem põe `sncr_mock` no catálogo (e responde `unknown_feature` se alguém tentar ligar); para a tela é o mesmo que desligado.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, GraphQL Codegen (client preset, `src/gql/` gerado e commitado), Vitest 2 + Testing Library 16 + user-event 14 (jsdom, sem jest-dom, `vi.stubGlobal("fetch")`).

**Spec:** `docs/superpowers/specs/2026-10-10-module-19d-controlled-prescriptions-design.md` (§3 "Interruptores e SNCR", §7 "Maintenance", §11 F-19.24), `docs/adr/0034.md` e o contrato `docs/superpowers/plans/2026-10-10-module-19d-controlled-contracts.md` §8 (a forma exata do tipo GraphQL está na seção final "Divergências propostas ao contrato", M1).

## Global Constraints

- Interruptores (contrato §8; spec §3), pelo mecanismo genérico (`City.features`, `setCityFeature`):
  - `controlled_prescriptions` com `requires: ["feature:clinical_documents"]`; faltando, `missing: ["clinical_documents_disabled"]`;
  - `sncr_mock` com `requires: ["feature:controlled_prescriptions"]`; faltando, `missing: ["controlled_prescriptions_disabled"]`; **só fora de produção** — em produção não aparece no catálogo e a mutation responde `unknown_feature`.
  - Ligar não exige o pré-requisito: fica "ligada, falta pré-requisito" (três estados, como no módulo 16). Nenhuma mudança no fluxo de `setCityFeature` da `FeaturesTab`.
- Leitura nova (contrato §8, tipo da Divergência M1), de plataforma, na raiz:
  ```graphql
  sncrStatus: SncrStatus!   # Query
  # SncrStatus { configured: Boolean!, reachable: Boolean!, lastCheckAt: ISO8601DateTime, simulatedAvailable: Boolean! }
  ```
- Aviso de simulado: pelo estado **ligado** de `sncr_mock` (não pelo "utilizável"), na aba (`role="note"`, nome "modo do SNCR") e no quadro (`role="note"`, nome "modo do SNCR desta cidade"). Texto: `SNCR_MOCK_NOTICE`. Desligado ou ausente do catálogo: o quadro diz `SNCR_REAL_NOTICE` e a aba não avisa.
- Só leitura e sem segredo: a configuração do SNCR (`sncr.{base_url, auth_url, maintainer_cnpj}` — sem `client_id`/`client_secret`, plano do api do 19d, D3) fica nas credenciais cifradas do api; a tela mostra só configurado/alcançável/última checagem. Nenhum número SNCR, CPF, CNPJ, token ou endereço passa por aqui.
- Valor desconhecido (api mais novo) aparece cru — nunca `undefined`/`null` na tela; nulo vira "nunca"/"—".
- **Ordem de deploy: contracts → api → dashboard → maintenance** (contrato §11). Contra um api sem o 19d o catálogo não traz `controlled_prescriptions` e o quadro nem aparece; se o catálogo trouxer a chave mas o schema não tiver `sncrStatus` (api desencontrado), só o quadro mostra o aviso da ordem de deploy.
- Interface tem ciclo próprio: componentes e estilo existentes (`Panel`, `DataTable`, `Tag`, `Button`, `ConfirmButton`, `ErrorState`, `EmptyState`), sem redesign; o quadro novo copia a forma do `SignaturePlatformPanel`.
- Nunca escreva à mão em arquivo gerado (`schema.graphql`, `src/gql/*`): eles saem do api em execução e do codegen.
- Nunca `git add -A` nem `git add .`: o worktree tem o symlink de `node_modules`. Adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Merge, push e tag só com autorização explícita do usuário.

## Ambiente de execução

- Worktree `apps/maintenance/.claude/mod19d`, branch `feat/mod-19d-controlled` a partir de `origin/main` (Task 0). O `.gitignore` do maintenance ignora `node_modules` (também o symlink), mas **não** ignora `.claude/`: o checkout principal lista `.claude/` como não rastreado. É esperado; não adicione.
- Todos os caminhos de arquivo das tasks são relativos ao worktree; os comandos partem da raiz do monorepo (`/Users/eduardovrocha/Development/ioit.solutions/rota-saude`).
- Testes: `cd apps/maintenance/.claude/mod19d && npx vitest run <arquivos>` (o `vitest.config.ts` exclui `e2e/**`).
- Tipos e codegen: `cd apps/maintenance/.claude/mod19d && npx tsc --noEmit` e `npm run codegen`.
- Ambiente de teste: `environment: "jsdom"`, `globals: false` (todo teste de componente chama `cleanup` no `afterEach`), sem jest-dom (use `not.toBeNull()`, `toBeNull()`, `.textContent`, `(el as HTMLButtonElement).disabled`). Datas saem de `fmtWhen` (`src/lib/analytics.ts`), que formata em `America/Sao_Paulo` (`"2026-10-10T12:00:00Z"` → `"10/10/2026, 09:00"`).
- **`npm run schema:pull` não serve no worktree** (o script sobe três níveis a partir de `scripts/` e escreveria no checkout principal). A Task 2 extrai o SDL pelo `docker compose exec` direto, com destino no worktree, como no 19c.
- Login de dev (Task 3): `http://maintenance.localhost:5177`, nunca `localhost:5177`. A conta é o `Maintainer` `dev@local` semeado pelo `db:seed`; o usuário faz o login.

## Review Focus

1. **O mantenedor liga `sncr_mock` e esquece.** Médicos da cidade passam a receber números que não valem na farmácia. O aviso tem de estar onde se liga e onde se olha o SNCR, com a palavra "SIMULADO" e "sem validade", e sumir quando desliga. Testes: Task 2, "SNCR simulado ligado: aviso na aba e no quadro", "ligar sncr_mock por dois cliques: após o refetch a aba avisa" e "SNCR simulado desligado: sem aviso na aba; o quadro diz SNCR real".
2. **Produção: `sncr_mock` não existe no catálogo.** A aba não pode quebrar, nem mostrar a linha, nem dizer "simulado"; o quadro diz que o simulado está indisponível no ambiente. Testes: Task 1, "simulado indisponível (produção)"; Task 2, "catálogo sem sncr_mock (produção): nada quebra e nada diz simulado".
3. **Ligar `controlled_prescriptions` sem `clinical_documents`.** Fica "ligada, falta pré-requisito", com o motivo em português, e o liga/desliga é o genérico (dois cliques, frase com o estado devolvido). Testes: Task 1, "pré-requisitos do 19d têm frase"; Task 2, "controlled_prescriptions pelo mecanismo genérico: falta documentos clínicos e liga com dois cliques".
4. **SNCR real sem credencial, fora do ar ou nunca checado.** "Configurado" não é "alcançável": três estados diferentes, com a hora da última checagem, sem `null` na tela e sem nenhum segredo. Testes: Task 1, "estado do SNCR: sem credencial, nunca checado, alcançável, inalcançável"; Task 2, "SNCR configurado e inalcançável aparece em destaque" e "lista o estado sem segredo e sem undefined/null".
5. **Maintenance novo contra api sem o 19d.** Com o catálogo trazendo `controlled_prescriptions` mas o schema sem `sncrStatus`, a validação recusa a consulta do quadro: só ele explica a ordem de deploy, e o liga/desliga continua. Testes: Task 2, "api sem o 19d: o quadro explica a ordem de deploy e a aba segue" (na aba) e "api sem o 19d: explica a ordem de deploy" (no quadro).

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | conferência do main (19c entregue) e do api do 19d, worktree e base | 0 |
| `src/lib/features.ts`, `src/lib/features.test.ts`, `src/lib/sncr.ts`, `src/lib/sncr.test.ts` | rótulos das duas chaves e dos dois pré-requisitos; regras de exibição do SNCR | 1 |
| `schema.graphql`, `src/gql/*` (gerados) | tipo `SncrStatus` e o campo `sncrStatus` | 2 |
| `src/screens/SncrPlatformPanel.tsx`, `src/screens/SncrPlatformPanel.test.tsx`, `src/screens/FeaturesTab.tsx`, `src/screens/FeaturesTab.test.tsx`, `README.md` | quadro do SNCR; aviso de simulado na aba; o quadro montado com `controlled_prescriptions` no catálogo | 2 |
| — | suíte, codegen:check, build, revisão e prova no navegador | 3 |

**Estratégia de teste:** regras de exibição com tabela de casos (Task 1); o quadro com `fetch` falso que responde por nome de operação (`query SncrPlatform`), cobrindo os quatro estados do SNCR real, a disponibilidade do simulado, o api sem o 19d e o "atualizar" (Task 2); os dois interruptores testados pelo fluxo **existente** de dois cliques da `FeaturesTab`, sem mudar o código do liga/desliga, e o aviso de simulado na aba e no quadro (Task 2). A prova final é no navegador, contra o api do 19d (Task 3).

---

### Task 0: Conferência do main (19c entregue) e do api do 19d, worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do maintenance com o 19c; a branch do api do 19d rodando na 3039.
- Produces: worktree `apps/maintenance/.claude/mod19d` na branch `feat/mod-19d-controlled`; a base anotada (arquivos e testes verdes).

- [ ] **Step 1: Confira que o 19c chegou ao main e que o api do 19d está no ar**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude
/opt/homebrew/bin/git -C apps/maintenance fetch origin
/opt/homebrew/bin/git -C apps/maintenance log --oneline -5 origin/main
/opt/homebrew/bin/git -C apps/maintenance cat-file -e origin/main:src/screens/MedicationCatalog.tsx && echo "19c ok"
/opt/homebrew/bin/git -C apps/maintenance show origin/main:src/lib/features.ts | grep -n 'clinical_documents\|signature_psc_mock\|clinical_documents_disabled\|controlled_prescriptions'
/opt/homebrew/bin/git -C apps/maintenance show origin/main:src/screens/FeaturesTab.tsx | grep -n 'SignaturePlatformPanel pscMock\|const pscSimulated\|aria-label="modo do PSC"'
grep -rn "controlled_prescriptions\|sncr_mock" apps/api/.claude/mod19d/app/services/platform 2>/dev/null | head -4
docker compose exec -T -w /rails/.claude/mod19d api bin/rails runner \
  'puts Maintenance::Schema.to_definition' 2>/dev/null | grep -n -E "sncrStatus|type SncrStatus"
```

Expected:
- `origin/main` com os commits do maintenance do 19c (mais novo que `0f0a779`) e `19c ok`. **Se `MedicationCatalog.tsx` não existir, pare e reporte**: o 19c ainda não foi entregue (pré-requisito).
- em `features.ts`: a linha do rótulo de `clinical_documents` (19c) e a de `signature_psc_mock` (19b); **nenhuma** linha com `clinical_documents_disabled` nem `controlled_prescriptions` (se já existirem, a Task 1 só acrescenta o que faltar e o teste dela continua valendo).
- em `FeaturesTab.tsx`: as três linhas do 19b que a Task 2 usa como âncora (`const pscSimulated = …`, o `<p role="note" aria-label="modo do PSC" …>` e o `<SignaturePlatformPanel pscMock={pscSimulated} />`). Se mudaram, aplique a mesma intenção sobre o texto real.
- ao menos uma linha com `controlled_prescriptions`/`sncr_mock` no catálogo do api do 19d e as duas linhas do schema (`sncrStatus: SncrStatus!` e `type SncrStatus {`). Se o api do 19d ainda não tiver o campo, as Tasks 0 e 1 seguem; **pare antes da Task 2** e reporte.

- [ ] **Step 2: Crie o worktree e ligue o `node_modules`**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/maintenance
/opt/homebrew/bin/git worktree add .claude/mod19d -b feat/mod-19d-controlled origin/main
ln -s ../../node_modules .claude/mod19d/node_modules
cd .claude/mod19d && npx vitest run 2>&1 | tail -3 && npx tsc --noEmit && echo tsc-ok
```

Expected: suíte verde e `tsc-ok`. **Anote** a base (arquivos e testes): a Task 3 espera a base mais **2 arquivos de teste novos** (`sncr.test.ts` e `SncrPlatformPanel.test.tsx`).

---

### Task 1: Rótulos dos interruptores e regras de exibição do SNCR

**Files:**
- Modify: `src/lib/features.ts`, `src/lib/features.test.ts`
- Create: `src/lib/sncr.ts`
- Test: `src/lib/sncr.test.ts`

**Interfaces:**
- Consumes: `Tone` de `src/theme/tokens.ts`; `fmtWhen` de `src/lib/analytics.ts`; `featureLabel`, `missingText` (existentes).
- Produces:
  - `featureLabel("controlled_prescriptions")` → "Receita de controle especial e de antimicrobiano (SNCR)"; `featureLabel("sncr_mock")` → "SNCR simulado (desenvolvimento)";
  - `missingText("clinical_documents_disabled")` → "documentos clínicos (clinical_documents) desligados"; `missingText("controlled_prescriptions_disabled")` → "receita de controlado (controlled_prescriptions) desligada";
  - em `src/lib/sncr.ts`:
    ```ts
    export const SNCR_MOCK_KEY = "sncr_mock";
    export const CONTROLLED_KEY = "controlled_prescriptions";
    export const SNCR_MOCK_NOTICE: string;   // "Esta cidade numera … SNCR SIMULADO — numeração sem validade (fora de produção)."
    export const SNCR_REAL_NOTICE: string;
    export function sncrModeNotice(features: { key: string; enabled: boolean }[]): { text: string; tone: Tone; simulated: boolean };
    export type SncrStatusView = { configured: boolean; reachable: boolean; lastCheckAt?: string | null; simulatedAvailable: boolean };
    export function sncrStatusLabel(s: SncrStatusView): { label: string; tone: Tone };
    export function sncrLastCheckText(s: SncrStatusView): string;
    export function simulatedAvailabilityText(s: SncrStatusView): string;
    export function sncrDetail(s: SncrStatusView): string;
    ```

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/sncr.test.ts
import { describe, expect, it } from "vitest";
import {
  SNCR_MOCK_NOTICE, SNCR_REAL_NOTICE, simulatedAvailabilityText, sncrDetail, sncrLastCheckText, sncrModeNotice, sncrStatusLabel
} from "./sncr";

const base = { configured: true, reachable: true, lastCheckAt: "2026-10-10T12:00:00Z", simulatedAvailable: true };

describe("sncrModeNotice", () => {
  it("pelo estado LIGADO de sncr_mock, não pelo utilizável", () => {
    expect(sncrModeNotice([ { key: "sncr_mock", enabled: true } ])).toEqual({ text: SNCR_MOCK_NOTICE, tone: "warn", simulated: true });
    expect(sncrModeNotice([ { key: "sncr_mock", enabled: false } ])).toEqual({ text: SNCR_REAL_NOTICE, tone: "neutral", simulated: false });
  });

  it("sem a chave no catálogo (produção) é o mesmo que desligado", () => {
    expect(sncrModeNotice([ { key: "controlled_prescriptions", enabled: true } ]).simulated).toBe(false);
    expect(sncrModeNotice([]).text).toBe(SNCR_REAL_NOTICE);
  });

  it("o aviso de simulado diz SIMULADO e sem validade", () => {
    expect(SNCR_MOCK_NOTICE).toMatch(/SIMULADO/);
    expect(SNCR_MOCK_NOTICE).toMatch(/sem validade/);
    expect(SNCR_REAL_NOTICE).not.toMatch(/SIMULADO/);
  });
});

describe("estado do SNCR: sem credencial, nunca checado, alcançável, inalcançável", () => {
  it("sem credencial vem antes de tudo", () => {
    expect(sncrStatusLabel({ ...base, configured: false, reachable: false, lastCheckAt: null }))
      .toEqual({ label: "sem credencial — SNCR real não configurado", tone: "neutral" });
  });

  it("configurado e nunca checado não é verde", () => {
    expect(sncrStatusLabel({ ...base, reachable: false, lastCheckAt: null })).toEqual({ label: "configurado, nunca checado", tone: "info" });
  });

  it("alcançável e inalcançável na última checagem", () => {
    expect(sncrStatusLabel(base)).toEqual({ label: "configurado e alcançável", tone: "ok" });
    expect(sncrStatusLabel({ ...base, reachable: false })).toEqual({ label: "configurado, inalcançável na última checagem", tone: "down" });
  });

  it("última checagem: hora da cidade ou 'nunca'", () => {
    expect(sncrLastCheckText(base)).toBe("10/10/2026, 09:00");
    expect(sncrLastCheckText({ ...base, lastCheckAt: null })).toBe("nunca");
    expect(sncrLastCheckText({ ...base, lastCheckAt: undefined })).toBe("nunca");
  });

  it("detalhe do inalcançável diz o que acontece com as receitas", () => {
    expect(sncrDetail({ ...base, reachable: false }))
      .toBe("o api não alcançou o SNCR — os profissionais não conseguem obter números; com o estoque zerado, as receitas saem em papel");
    expect(sncrDetail(base)).toBe("os profissionais obtêm números em Conta → SNCR, pelo gov.br de cada um");
    expect(sncrDetail({ ...base, configured: false, reachable: false, lastCheckAt: null }))
      .toBe("sem credencial, só o SNCR simulado numera (fora de produção); em produção, as receitas de controlado saem em papel");
  });
});

describe("simulado disponível no ambiente", () => {
  it("disponível fora de produção; indisponível (produção)", () => {
    expect(simulatedAvailabilityText(base)).toBe("SNCR simulado disponível neste ambiente (fora de produção)");
    expect(simulatedAvailabilityText({ ...base, simulatedAvailable: false })).toBe("SNCR simulado indisponível neste ambiente (produção)");
  });
});
```

Em `src/lib/features.test.ts`, no `describe("missingText e missingSummary")`, depois do teste "pré-requisitos do prontuário e da assinatura têm frase":

```ts
  it("pré-requisitos do 19d têm frase", () => {
    expect(missingText("clinical_documents_disabled")).toBe("documentos clínicos (clinical_documents) desligados");
    expect(missingText("controlled_prescriptions_disabled")).toBe("receita de controlado (controlled_prescriptions) desligada");
    expect(missingSummary([ "controlled_prescriptions_disabled" ])).toBe("receita de controlado (controlled_prescriptions) desligada");
  });
```

e, no `describe` de `featureLabel` (o teste que confere `featureLabel("signature_psc_mock")`; se ele estiver noutro bloco, ponha logo depois dele):

```ts
  it("as chaves do 19d têm nome em português", () => {
    expect(featureLabel("controlled_prescriptions")).toBe("Receita de controle especial e de antimicrobiano (SNCR)");
    expect(featureLabel("sncr_mock")).toBe("SNCR simulado (desenvolvimento)");
  });
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/maintenance/.claude/mod19d && npx vitest run src/lib/sncr.test.ts src/lib/features.test.ts`
Expected: FAIL — `./sncr` não existe; `missingText("clinical_documents_disabled")` devolve o código cru; `featureLabel("sncr_mock")` devolve `null`.

- [ ] **Step 3: Write minimal implementation**

```ts
// src/lib/sncr.ts
// Regras de exibição do SNCR no maintenance (módulo 19d, F-19.24; spec §3 e
// §7; contrato §8). Só leitura e de PLATAFORMA: as credenciais do SNCR ficam
// nas credenciais cifradas do api, por ambiente; aqui só aparece se existem,
// se o api alcançou o SNCR e quando foi a última checagem. Nenhum segredo,
// nenhum número SNCR, nenhum dado de profissional ou paciente.
import type { Tone } from "../theme/tokens";
import { fmtWhen } from "./analytics";

export const SNCR_MOCK_KEY = "sncr_mock";
export const CONTROLLED_KEY = "controlled_prescriptions";
export const SNCR_MOCK_NOTICE =
  "Esta cidade numera as receitas de controlado com o SNCR SIMULADO — numeração sem validade (fora de produção).";
export const SNCR_REAL_NOTICE =
  "Sem o SNCR simulado: os números vêm do SNCR da Anvisa configurado no ambiente.";

// Aviso pelo estado LIGADO do interruptor (não pelo "utilizável"): ligado, a
// cidade passa a numerar com o simulado. Sem o interruptor no catálogo
// (produção) é o mesmo que desligado.
export function sncrModeNotice(features: { key: string; enabled: boolean }[]): { text: string; tone: Tone; simulated: boolean } {
  const simulated = features.some((f) => f.key === SNCR_MOCK_KEY && f.enabled);
  return simulated
    ? { text: SNCR_MOCK_NOTICE, tone: "warn", simulated: true }
    : { text: SNCR_REAL_NOTICE, tone: "neutral", simulated: false };
}

export type SncrStatusView = { configured: boolean; reachable: boolean; lastCheckAt?: string | null; simulatedAvailable: boolean };

// "Configurado" não é "alcançável": sem checagem ainda, nunca verde.
export function sncrStatusLabel(s: SncrStatusView): { label: string; tone: Tone } {
  if (!s.configured) return { label: "sem credencial — SNCR real não configurado", tone: "neutral" };
  if (!s.lastCheckAt) return { label: "configurado, nunca checado", tone: "info" };
  if (s.reachable) return { label: "configurado e alcançável", tone: "ok" };
  return { label: "configurado, inalcançável na última checagem", tone: "down" };
}

export function sncrLastCheckText(s: SncrStatusView): string {
  return s.lastCheckAt ? fmtWhen(s.lastCheckAt) : "nunca";
}

export function sncrDetail(s: SncrStatusView): string {
  if (!s.configured) {
    return "sem credencial, só o SNCR simulado numera (fora de produção); em produção, as receitas de controlado saem em papel";
  }
  if (s.lastCheckAt && !s.reachable) {
    return "o api não alcançou o SNCR — os profissionais não conseguem obter números; com o estoque zerado, as receitas saem em papel";
  }
  return "os profissionais obtêm números em Conta → SNCR, pelo gov.br de cada um";
}

export function simulatedAvailabilityText(s: SncrStatusView): string {
  return s.simulatedAvailable
    ? "SNCR simulado disponível neste ambiente (fora de produção)"
    : "SNCR simulado indisponível neste ambiente (produção)";
}
```

Em `src/lib/features.ts`, no `MISSING_LABELS`, logo antes do comentário de `city_unreachable`:

```ts
  // Módulo 19d: controlled_prescriptions exige clinical_documents ligado;
  // sncr_mock exige controlled_prescriptions ligado.
  clinical_documents_disabled: "documentos clínicos (clinical_documents) desligados",
  controlled_prescriptions_disabled: "receita de controlado (controlled_prescriptions) desligada",
```

e no `FEATURE_LABELS`, depois da linha de `signature_psc_mock` (e da de `clinical_documents`, que o 19c acrescentou):

```ts
  controlled_prescriptions: "Receita de controle especial e de antimicrobiano (SNCR)",
  sncr_mock: "SNCR simulado (desenvolvimento)",
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/maintenance/.claude/mod19d && npx vitest run src/lib/sncr.test.ts src/lib/features.test.ts && npx tsc --noEmit`
Expected: PASS e `tsc` sem erro.

- [ ] **Step 5: Commit**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/maintenance/.claude/mod19d
/opt/homebrew/bin/git add src/lib/sncr.ts src/lib/sncr.test.ts src/lib/features.ts src/lib/features.test.ts
/opt/homebrew/bin/git commit -m "feat: label the controlled prescription and simulated SNCR switches and add SNCR display rules

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Schema, codegen, quadro do SNCR e os interruptores na aba

**Files:**
- Gerados: `schema.graphql`, `src/gql/gql.ts`, `src/gql/graphql.ts`
- Create: `src/screens/SncrPlatformPanel.tsx`
- Modify: `src/screens/FeaturesTab.tsx`, `src/screens/FeaturesTab.test.tsx`, `README.md`
- Test: `src/screens/SncrPlatformPanel.test.tsx`, `src/screens/FeaturesTab.test.tsx`

**Interfaces:**
- Consumes: `SNCR_MOCK_NOTICE`, `SNCR_REAL_NOTICE`, `CONTROLLED_KEY`, `sncrModeNotice`, `sncrStatusLabel`, `sncrLastCheckText`, `sncrDetail`, `simulatedAvailabilityText`, `SncrStatusView` (Task 1); `gql` (`src/lib/api.ts`), `graphql` (`src/gql`), `GraphQLRefusal` (`src/lib/errors.ts`), `Panel`, `Button`, `Tag`, `ErrorState`.
- Produces: `export function SncrPlatformPanel({ sncrMock }: { sncrMock: boolean }): JSX.Element` — `<section aria-label="SNCR da Anvisa — numeração de receitas (todas as cidades do ambiente)">` com um `Panel` (botão "atualizar"), a nota de modo (`role="note"`, nome "modo do SNCR desta cidade"), a nota das credenciais, e o grupo `role="group"` "estado do SNCR" (selo, última checagem, detalhe, disponibilidade do simulado). Operação com nome fixo `query SncrPlatform`; chave de cache `[ "platform", "sncr" ]`. A `FeaturesTab` mostra a nota `role="note"` "modo do SNCR" com `sncr_mock` ligado e monta o quadro quando o catálogo traz `controlled_prescriptions`.

- [ ] **Step 1: Extraia o schema do api do 19d**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose exec -T -w /rails/.claude/mod19d api bin/rails runner 'puts Maintenance::Schema.to_definition' \
  > apps/maintenance/.claude/mod19d/schema.graphql
/opt/homebrew/bin/git -C apps/maintenance/.claude/mod19d diff --stat
/opt/homebrew/bin/git -C apps/maintenance/.claude/mod19d diff schema.graphql | grep '^[+-]' | grep -v '^+++\|^---'
```

(Se o plano do api subir o servidor do 19d num container próprio, use o nome dele no lugar de `api`.)

Expected: só `schema.graphql` mudou, e só com o acréscimo do campo e do tipo (a ordem das linhas é a que o `to_definition` do graphql-ruby escrever):
```
+  sncrStatus: SncrStatus!
+type SncrStatus {
+  configured: Boolean!
+  lastCheckAt: ISO8601DateTime
+  reachable: Boolean!
+  simulatedAvailable: Boolean!
+}
```
(mais as linhas de descrição `"""…"""` que o api tiver posto). Se aparecerem outras mudanças (tipos do 19c que o `origin/main` do maintenance ainda não tinha, ou outro campo do 19d), reporte as linhas antes de seguir. O codegen roda no Step 3, depois do documento novo existir.

- [ ] **Step 2: Write the failing test**

```tsx
// src/screens/SncrPlatformPanel.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, render, screen, waitFor, within } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";
import { SncrPlatformPanel } from "./SncrPlatformPanel";
import { SNCR_MOCK_NOTICE, SNCR_REAL_NOTICE } from "../lib/sncr";

afterEach(() => { cleanup(); vi.unstubAllGlobals(); });

function reply(status: number, body: unknown) {
  return new Response(JSON.stringify(body), { status, headers: { "Content-Type": "application/json" } });
}
function bodyOf(call: unknown[]): { query: string } {
  return JSON.parse((call[1] as RequestInit).body as string);
}

const STATUS = { configured: true, reachable: true, lastCheckAt: "2026-10-10T12:00:00Z", simulatedAvailable: true };

function statusReply(status: unknown = STATUS) {
  return reply(200, { data: { sncrStatus: status } });
}

function renderPanel(sncrMock = false) {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  function wrapper({ children }: { children: ReactNode }) {
    return <QueryClientProvider client={client}>{children}</QueryClientProvider>;
  }
  return render(<SncrPlatformPanel sncrMock={sncrMock} />, { wrapper });
}

describe("SncrPlatformPanel", () => {
  let fetchMock: ReturnType<typeof vi.fn>;

  beforeEach(() => {
    fetchMock = vi.fn();
    vi.stubGlobal("fetch", fetchMock);
  });

  it("lista o estado sem segredo e sem undefined/null", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(statusReply()));
    renderPanel();

    const group = await screen.findByRole("group", { name: "estado do SNCR" });
    expect(within(group).getByText("configurado e alcançável")).not.toBeNull();
    expect(within(group).getByText("última checagem: 10/10/2026, 09:00")).not.toBeNull();
    expect(within(group).getByText("os profissionais obtêm números em Conta → SNCR, pelo gov.br de cada um")).not.toBeNull();
    expect(within(group).getByText("SNCR simulado disponível neste ambiente (fora de produção)")).not.toBeNull();

    expect(bodyOf(fetchMock.mock.calls[0]).query).toMatch(/query SncrPlatform/);
    expect(bodyOf(fetchMock.mock.calls[0]).query).not.toMatch(/secret|client_?id|cnpj|token|base_?url|auth_?url/i);
    expect(document.body.textContent).not.toMatch(/undefined|null|secret|client_id|base_url|auth_url|cnpj|token/i);
  });

  it("SNCR configurado e inalcançável aparece em destaque", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(statusReply({ ...STATUS, reachable: false })));
    renderPanel();
    const group = await screen.findByRole("group", { name: "estado do SNCR" });
    expect(within(group).getByText("configurado, inalcançável na última checagem")).not.toBeNull();
    expect(within(group).getByText(/o api não alcançou o SNCR/)).not.toBeNull();
  });

  it("sem credencial e nunca checado: diz 'nunca', sem null", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(statusReply({ configured: false, reachable: false, lastCheckAt: null, simulatedAvailable: false })));
    renderPanel();
    const group = await screen.findByRole("group", { name: "estado do SNCR" });
    expect(within(group).getByText("sem credencial — SNCR real não configurado")).not.toBeNull();
    expect(within(group).getByText("última checagem: nunca")).not.toBeNull();
    expect(within(group).getByText("SNCR simulado indisponível neste ambiente (produção)")).not.toBeNull();
    expect(document.body.textContent).not.toMatch(/undefined|null/);
  });

  it("api sem o 19d: explica a ordem de deploy", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(reply(200, {
      errors: [ {
        message: "Field 'sncrStatus' doesn't exist on type 'Query'",
        extensions: { code: "undefinedField", typeName: "Query", fieldName: "sncrStatus" }
      } ]
    })));
    renderPanel();
    expect((await screen.findByRole("alert")).textContent).toMatch(/o api do módulo 19d precisa subir antes do maintenance/);
    expect(screen.getByRole("note", { name: "modo do SNCR desta cidade" }).textContent).toBe(SNCR_REAL_NOTICE);
  });

  it("atualizar relê", async () => {
    const user = userEvent.setup();
    fetchMock.mockImplementation(() => Promise.resolve(statusReply()));
    renderPanel();
    await screen.findByRole("group", { name: "estado do SNCR" });
    await user.click(screen.getByRole("button", { name: "atualizar" }));
    await waitFor(() => expect(fetchMock).toHaveBeenCalledTimes(2));
  });

  it("SNCR simulado ligado: aviso em destaque, mesmo enquanto carrega", async () => {
    fetchMock.mockImplementation(() => new Promise(() => undefined));
    renderPanel(true);
    expect(screen.getByRole("note", { name: "modo do SNCR desta cidade" }).textContent).toBe(SNCR_MOCK_NOTICE);
  });

  it("desligado: diz que usa o SNCR real", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(statusReply()));
    renderPanel(false);
    expect(screen.getByRole("note", { name: "modo do SNCR desta cidade" }).textContent).toBe(SNCR_REAL_NOTICE);
    await screen.findByRole("group", { name: "estado do SNCR" });
    expect(document.body.textContent).not.toMatch(/SIMULADO/);
  });
});
```

Em `src/screens/FeaturesTab.test.tsx`, nos imports do topo, acrescente:

```tsx
import { SNCR_MOCK_NOTICE, SNCR_REAL_NOTICE } from "../lib/sncr";
```

e, no fim do arquivo, um `describe` novo:

```tsx
describe("FeaturesTab — receita de controlado e SNCR (módulo 19d)", () => {
  let fetchMock: ReturnType<typeof vi.fn>;

  beforeEach(() => {
    fetchMock = vi.fn();
    vi.stubGlobal("fetch", fetchMock);
  });

  afterEach(() => vi.unstubAllGlobals());

  const CONTROLLED_OFF: Feature = {
    key: "controlled_prescriptions", description: "Receita de controle especial e de antimicrobiano com número SNCR",
    enabled: false, usable: false, missing: [ "clinical_documents_disabled" ], changedAt: null, changedBy: null
  };
  const SNCR_MOCK_OFF: Feature = {
    key: "sncr_mock", description: "SNCR simulado para desenvolvimento", enabled: false, usable: false,
    missing: [ "controlled_prescriptions_disabled" ], changedAt: null, changedBy: null
  };
  const SNCR_MOCK_ON: Feature = { ...SNCR_MOCK_OFF, enabled: true, changedAt: "2026-10-10T13:00:00Z", changedBy: "dev@local" };
  const sncrPlatform = () => reply(200, { data: {
    sncrStatus: { configured: false, reachable: false, lastCheckAt: null, simulatedAvailable: true }
  } });
  function withCatalog(features: Feature[]) {
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("query SncrPlatform")) return Promise.resolve(sncrPlatform());
      return Promise.resolve(featuresReply(features));
    });
  }

  it("controlled_prescriptions pelo mecanismo genérico: falta documentos clínicos e liga com dois cliques", async () => {
    const user = userEvent.setup();
    const turnedOn: Feature = { ...CONTROLLED_OFF, enabled: true, changedAt: "2026-10-10T13:00:00Z", changedBy: "dev@local" };
    let listed = [ CONTROLLED_OFF ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("query SncrPlatform")) return Promise.resolve(sncrPlatform());
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [ turnedOn ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: turnedOn } } }));
      }
      return Promise.resolve(featuresReply(listed, { recordMode: "record" }));
    });
    renderTab();

    expect(await screen.findByText("controlled_prescriptions")).not.toBeNull();
    expect(screen.getByText("Receita de controle especial e de antimicrobiano (SNCR)")).not.toBeNull();
    expect(screen.getByText("documentos clínicos (clinical_documents) desligados")).not.toBeNull();

    await user.click(screen.getByRole("button", { name: "ligar" }));
    expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(0);
    await user.click(screen.getByRole("button", { name: "confirmar: ligar controlled_prescriptions" }));
    await waitFor(() => expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(1));
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[0]).variables)
      .toEqual({ citySlug: "sp", key: "controlled_prescriptions", enabled: true });
    expect((await screen.findByRole("status")).textContent).toBe(
      "controlled_prescriptions ligada, mas ainda não utilizável — falta: documentos clínicos (clinical_documents) desligados."
    );
  });

  it("com controlled_prescriptions no catálogo mostra o quadro do SNCR; sem ela, nem consulta", async () => {
    withCatalog([ CONTROLLED_OFF ]);
    renderTab();
    expect(await screen.findByRole("region", { name: "SNCR da Anvisa — numeração de receitas (todas as cidades do ambiente)" }))
      .not.toBeNull();
    expect(await screen.findByText("sem credencial — SNCR real não configurado")).not.toBeNull();
    cleanup();

    fetchMock.mockReset();
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply([ LEDI_OFF ])));
    renderTab();
    await screen.findByText("ledi_export");
    expect(calls(fetchMock, "query SncrPlatform")).toHaveLength(0);
  });

  it("api sem o 19d: o quadro explica a ordem de deploy e a aba segue", async () => {
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("query SncrPlatform")) {
        return Promise.resolve(reply(200, { errors: [ {
          message: "Field 'sncrStatus' doesn't exist on type 'Query'",
          extensions: { code: "undefinedField", typeName: "Query", fieldName: "sncrStatus" }
        } ] }));
      }
      return Promise.resolve(featuresReply([ CONTROLLED_OFF ]));
    });
    renderTab();
    expect((await screen.findByRole("alert")).textContent).toMatch(/o api do módulo 19d precisa subir antes do maintenance/);
    expect(screen.getByText("controlled_prescriptions")).not.toBeNull();
    expect(screen.getByRole("button", { name: "ligar" })).not.toBeNull();
  });

  it("sncr_mock tem rótulo e diz o pré-requisito", async () => {
    withCatalog([ CONTROLLED_OFF, SNCR_MOCK_OFF ]);
    renderTab();
    expect(await screen.findByText("sncr_mock")).not.toBeNull();
    expect(screen.getByText("SNCR simulado (desenvolvimento)")).not.toBeNull();
    expect(screen.getByText("receita de controlado (controlled_prescriptions) desligada")).not.toBeNull();
  });

  it("SNCR simulado ligado: aviso na aba e no quadro", async () => {
    withCatalog([ CONTROLLED_OFF, SNCR_MOCK_ON ]);
    renderTab();
    expect((await screen.findByRole("note", { name: "modo do SNCR" })).textContent).toBe(SNCR_MOCK_NOTICE);
    expect(screen.getByRole("note", { name: "modo do SNCR desta cidade" }).textContent).toBe(SNCR_MOCK_NOTICE);
  });

  it("ligar sncr_mock por dois cliques: após o refetch a aba avisa", async () => {
    const user = userEvent.setup();
    let listed = [ CONTROLLED_OFF, SNCR_MOCK_OFF ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("query SncrPlatform")) return Promise.resolve(sncrPlatform());
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [ CONTROLLED_OFF, SNCR_MOCK_ON ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: SNCR_MOCK_ON } } }));
      }
      return Promise.resolve(featuresReply(listed));
    });
    renderTab();

    expect(await screen.findByText("sncr_mock")).not.toBeNull();
    expect(screen.queryByRole("note", { name: "modo do SNCR" })).toBeNull();
    // Duas linhas desligadas (controlled_prescriptions, sncr_mock): a do SNCR simulado é a segunda.
    await user.click(screen.getAllByRole("button", { name: "ligar" })[1]);
    await user.click(await screen.findByRole("button", { name: "confirmar: ligar sncr_mock" }));
    await waitFor(() => expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(1));
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[0]).variables)
      .toEqual({ citySlug: "sp", key: "sncr_mock", enabled: true });
    expect((await screen.findByRole("note", { name: "modo do SNCR" })).textContent).toBe(SNCR_MOCK_NOTICE);
  });

  it("SNCR simulado desligado: sem aviso na aba; o quadro diz SNCR real", async () => {
    withCatalog([ CONTROLLED_OFF, SNCR_MOCK_OFF ]);
    renderTab();
    expect((await screen.findByRole("note", { name: "modo do SNCR desta cidade" })).textContent).toBe(SNCR_REAL_NOTICE);
    expect(screen.queryByRole("note", { name: "modo do SNCR" })).toBeNull();
  });

  it("catálogo sem sncr_mock (produção): nada quebra e nada diz simulado", async () => {
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("query SncrPlatform")) {
        return Promise.resolve(reply(200, { data: {
          sncrStatus: { configured: true, reachable: true, lastCheckAt: "2026-10-10T12:00:00Z", simulatedAvailable: false }
        } }));
      }
      return Promise.resolve(featuresReply([ { ...CONTROLLED_OFF, enabled: true, usable: true, missing: [] } ]));
    });
    renderTab();
    expect((await screen.findByRole("note", { name: "modo do SNCR desta cidade" })).textContent).toBe(SNCR_REAL_NOTICE);
    await screen.findByText("SNCR simulado indisponível neste ambiente (produção)");
    expect(screen.queryByText("sncr_mock")).toBeNull();
    expect(screen.queryByRole("note", { name: "modo do SNCR" })).toBeNull();
    expect(document.body.textContent).not.toMatch(/SIMULADO|undefined|null/);
  });
});
```

(`reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab`, `Feature` e `LEDI_OFF` já estão no topo do arquivo, do módulo 16.)

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/screens/SncrPlatformPanel.tsx
// SNCR da Anvisa (módulo 19d, F-19.24; spec §3 e §7; contrato §8). Só leitura
// e de PLATAFORMA: vale para todas as cidades do ambiente. A configuração
// (endereços do SNCR e CNPJ da mantenedora) fica nas credenciais cifradas do
// api; aqui só aparece se existe, se o api alcançou o SNCR e quando, e se o
// SNCR simulado existe neste ambiente.
//
// Consulta PRÓPRIA, como o SignaturePlatformPanel do 19b: contra um api sem o
// 19d a validação recusa o documento inteiro, e só este quadro cai.
import { useQuery } from "@tanstack/react-query";
import type { CSSProperties } from "react";
import { gql } from "../lib/api";
import { graphql } from "../gql";
import { GraphQLRefusal } from "../lib/errors";
import {
  SNCR_MOCK_NOTICE, SNCR_REAL_NOTICE,
  simulatedAvailabilityText, sncrDetail, sncrLastCheckText, sncrStatusLabel
} from "../lib/sncr";
import { Panel } from "../components/Panel";
import { Button } from "../components/Button";
import { Tag } from "../components/Tag";
import { ErrorState } from "../components/ErrorState";

const SncrPlatformQuery = graphql(`
  query SncrPlatform {
    sncrStatus { configured reachable lastCheckAt simulatedAvailable }
  }
`);

const TITLE = "SNCR da Anvisa — numeração de receitas (todas as cidades do ambiente)";
const OLD_API_CODES = new Set([ "undefinedField" ]);

function requestErrorText(err: unknown): string {
  if (err instanceof GraphQLRefusal && OLD_API_CODES.has(err.code)) {
    return "esta API ainda não tem o SNCR — o api do módulo 19d precisa subir antes do maintenance";
  }
  return err instanceof Error ? err.message : "erro inesperado";
}

export function SncrPlatformPanel({ sncrMock }: { sncrMock: boolean }) {
  const query = useQuery({
    queryKey: [ "platform", "sncr" ],
    queryFn: () => gql(SncrPlatformQuery),
    staleTime: Infinity
  });
  const status = query.data?.data?.sncrStatus ?? null;
  const fieldError = query.data?.fieldErrors[0];
  const label = status ? sncrStatusLabel(status) : null;

  return (
    <section aria-label={TITLE}>
      <Panel title={TITLE} actions={<Button onClick={() => void query.refetch()} busy={query.isFetching}>atualizar</Button>}>
        {/* Estado da cidade, não da consulta: aparece também enquanto carrega ou falha. */}
        <p role="note" aria-label="modo do SNCR desta cidade" style={sncrMock ? warnNote : note}>
          {sncrMock ? SNCR_MOCK_NOTICE : SNCR_REAL_NOTICE}
        </p>
        <p style={note}>
          A configuração do SNCR (cadastro da integração na Anvisa) fica nas credenciais cifradas do api, por ambiente;
          aqui só aparece se existe. Cada número é pedido pelo gov.br do próprio profissional.
        </p>
        {query.isPending && <p>carregando…</p>}
        {query.isError && <ErrorState message={requestErrorText(query.error)} />}
        {fieldError && <ErrorState message={`${fieldError.code} — ${fieldError.message}`} />}
        {status && label && (
          <div role="group" aria-label="estado do SNCR" style={column}>
            <div style={row}>
              <strong style={{ fontSize: 12.5 }}>SNCR real</strong>
              <Tag tone={label.tone}>{label.label}</Tag>
              <span style={note}>{`última checagem: ${sncrLastCheckText(status)}`}</span>
            </div>
            <span style={note}>{sncrDetail(status)}</span>
            <span style={note}>{simulatedAvailabilityText(status)}</span>
          </div>
        )}
      </Panel>
    </section>
  );
}

const note: CSSProperties = { margin: 0, fontSize: 11.5, color: "var(--ink3)" };
const warnNote: CSSProperties = { margin: 0, fontSize: 12.5, fontWeight: 600, color: "var(--warn)" };
const row: CSSProperties = { display: "flex", gap: 8, alignItems: "center", flexWrap: "wrap" };
const column: CSSProperties = { display: "flex", flexDirection: "column", gap: 6 };
```

Em `src/screens/FeaturesTab.tsx`:

(a) nos imports, depois de `import { PSC_MOCK_NOTICE, pscModeNotice } from "../lib/signature";`:

```tsx
import { SncrPlatformPanel } from "./SncrPlatformPanel";
import { CONTROLLED_KEY, SNCR_MOCK_NOTICE, sncrModeNotice } from "../lib/sncr";
```

(b) depois de `const pscSimulated = city ? pscModeNotice(city.features).simulated : false;`:

```tsx
  const sncrSimulated = city ? sncrModeNotice(city.features).simulated : false;
```

(c) no `Panel` "Funcionalidades", logo depois do bloco `{pscSimulated && ( … )}`:

```tsx
            {sncrSimulated && (
              <p role="note" aria-label="modo do SNCR" style={{ margin: 0, fontSize: 12.5, fontWeight: 600, color: "var(--warn)" }}>
                {SNCR_MOCK_NOTICE}
              </p>
            )}
```

(d) depois da linha `{city.features.some((f) => f.key === "digital_signature") && <SignaturePlatformPanel pscMock={pscSimulated} />}`:

```tsx

          {/* Módulo 19d: SNCR da Anvisa (plataforma), só onde se decide ligar a
              receita de controlado. Consulta própria (SncrPlatformPanel). */}
          {city.features.some((f) => f.key === CONTROLLED_KEY) && <SncrPlatformPanel sncrMock={sncrSimulated} />}
```

No `README.md`, logo depois do parágrafo **Assinatura digital (módulo 19b).** (o que termina em "só o quadro mostra o aviso da ordem de deploy e a aba segue."), acrescente:

```markdown

**Receita de controlado e SNCR (módulo 19d).** O interruptor
`controlled_prescriptions` ("Receita de controle especial e de antimicrobiano
(SNCR)") aparece na mesma lista e liga como os outros; ele precisa de
`clinical_documents` ligado (falta: "documentos clínicos (clinical_documents)
desligados"). Quando o catálogo da cidade traz `controlled_prescriptions`, a
aba mostra também o quadro **SNCR da Anvisa — numeração de receitas**, só
leitura e da plataforma: se a credencial do SNCR existe no api, se o api o
alcançou na última checagem (e quando) e se o SNCR simulado existe neste
ambiente. Credenciais, CNPJ da mantenedora, token e números SNCR nunca aparecem
aqui. O interruptor `sncr_mock` ("SNCR simulado (desenvolvimento)") só existe
fora de produção e exige `controlled_prescriptions`; ligado, a aba e o quadro
avisam que a cidade numera com o SNCR simulado, sem validade. Em produção o api
nem o oferece. Ordem de deploy: contracts → api → dashboard → maintenance;
contra um api sem o 19d o catálogo não traz `controlled_prescriptions` e o
quadro nem aparece; se o catálogo trouxer a chave mas o schema não tiver
`sncrStatus`, só o quadro mostra o aviso da ordem de deploy e a aba segue.
```

Depois, o codegen:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/maintenance/.claude/mod19d && npm run codegen && /opt/homebrew/bin/git status --short src/gql
grep -n "SncrPlatform" src/gql/gql.ts | head -3
```

Expected: `src/gql/gql.ts` e `src/gql/graphql.ts` modificados; o documento `query SncrPlatform` registrado em `gql.ts`.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/maintenance/.claude/mod19d && npx vitest run src/screens/SncrPlatformPanel.test.tsx src/screens/FeaturesTab.test.tsx src/lib/sncr.test.ts src/lib/features.test.ts && npx tsc --noEmit && npm run codegen:check`
Expected: PASS (os testes antigos da `FeaturesTab`, inclusive os do 19b e do 19c, continuam verdes: nenhum catálogo deles traz `controlled_prescriptions` nem `sncr_mock`), `tsc` sem erro e `codegen:check` verde.

- [ ] **Step 5: Commit**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/maintenance/.claude/mod19d
/opt/homebrew/bin/git add schema.graphql src/gql/gql.ts src/gql/graphql.ts src/screens/SncrPlatformPanel.tsx \
  src/screens/SncrPlatformPanel.test.tsx src/screens/FeaturesTab.tsx src/screens/FeaturesTab.test.tsx README.md
/opt/homebrew/bin/git status --short
/opt/homebrew/bin/git commit -m "feat: show the SNCR status and the simulated SNCR notice next to the controlled prescription switch

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

Expected: `status --short` antes do commit sem nada além dos oito arquivos (e o `.claude/` não rastreado do checkout principal não aparece aqui, porque o comando roda no worktree).

---

### Task 3: Suíte, codegen:check, build, revisão e prova no navegador

**Files:** nenhum novo.

**Interfaces:**
- Consumes: as Tasks 1–2 na branch `feat/mod-19d-controlled`; o api do 19d rodando na 3039 com a semente e o `fake-sncr` do compose (porta 8092).
- Produces: branch pronta para merge (depois do merge do api e do dashboard do 19d).

- [ ] **Step 1: Suíte, tipos, codegen e build**

Run: `cd apps/maintenance/.claude/mod19d && npx vitest run && npx tsc --noEmit && npm run codegen:check && npm run build`
Expected: tudo verde (a CI roda os quatro): a base anotada na Task 0 mais **2 arquivos de teste novos** (`sncr.test.ts`, `SncrPlatformPanel.test.tsx`). Um número que dobra é artefato de build descoberto pelo vitest: pare e reporte.

```bash
cd apps/maintenance/.claude/mod19d && /opt/homebrew/bin/git status --short && /opt/homebrew/bin/git log --stat origin/main..HEAD | grep -c node_modules
```

Expected: `status` vazio e a contagem `0`.

- [ ] **Step 2: Nenhum segredo e nenhum campo novo no topo**

```bash
cd apps/maintenance/.claude/mod19d
grep -n -i -E "secret|client_?id|base_?url|auth_?url|cnpj|token" src/screens/SncrPlatformPanel.tsx src/lib/sncr.ts | grep -v "^.*//" ; echo "exit $?"
sed -n '/const CityHeaderQuery/,/`);/p' src/screens/CityDetail.tsx | grep -n -i "sncr" ; echo "exit $?"
```

Expected: os dois `exit 1` (nenhuma menção a segredo fora de comentário; o `CityHeader` não pede campo do 19d). O comentário do topo do quadro cita as credenciais **pelo nome** para dizer que ficam no api — é a única menção, e o `grep -v` a tira.

- [ ] **Step 3: Revisão**

Um subagente revisor lê `origin/main..HEAD` do maintenance contra a spec (§3, §7, §11 F-19.24), o ADR 0034 e o contrato do 19d §8. Pontos de atenção:
- o liga/desliga de `controlled_prescriptions` e `sncr_mock` é o genérico (nenhuma linha nova no fluxo de `setCityFeature`); os pré-requisitos aparecem em português;
- `SncrPlatform` é consulta própria; contra api antigo só o quadro falha; o quadro só monta com `controlled_prescriptions` no catálogo;
- o aviso de simulado sai do estado **ligado** de `sncr_mock`, na aba e no quadro, e some quando desliga; em produção (chave ausente) nada diz "simulado";
- nenhum segredo na consulta, no schema ou na tela; nulos viram "nunca"/"—";
- `schema.graphql` e `src/gql/*` iguais ao que o api em execução produz (`codegen:check` verde);
- nenhum `git add -A` no histórico.

- [ ] **Step 4: Prova no navegador (com o usuário)**

Depende do api do 19d na porta **3039** (plano do api do 19d) com o `fake-sncr` do compose (porta 8092) e as credenciais de dev do SNCR simulado. O proxy do Vite troca o Host para `maintenance-api.localhost` e injeta o Origin `http://maintenance.localhost:5177`, então o Vite do worktree sobe **na 5177**, num container da rede do compose (para alcançar `http://api:3039`), depois de parar o serviço `maintenance`:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose stop maintenance
docker compose run -d --rm --no-deps --name maintenance-mod19d -p 5177:5177 \
  -w /app/.claude/mod19d -e VITE_API_PROXY_TARGET=http://api:3039 -e VITE_MAINTENANCE_ENV=development \
  maintenance npx vite --port 5177 --host 0.0.0.0
docker logs -f maintenance-mod19d   # espere o "ready"; Ctrl+C sai do log
```

(Se o plano do api subir o servidor do 19d num container próprio, use o nome dele no lugar de `api`.)

Abra `http://maintenance.localhost:5177` (nunca `localhost:5177`). O usuário faz o login do mantenedor; não digite senha nem TOTP. Confira com screenshot:
- Cidades → Curitiba → **Funcionalidades**: `controlled_prescriptions` na lista, com o nome em português e a descrição do catálogo; com `clinical_documents` desligado, depois de ligar: "ligada, falta pré-requisito" / "documentos clínicos (clinical_documents) desligados"; com ele ligado, "ligada e utilizável";
- `sncr_mock` na lista ("SNCR simulado (desenvolvimento)"), com o pré-requisito "receita de controlado (controlled_prescriptions) desligada" enquanto ela estiver desligada;
- "ligar" → "confirmar: ligar sncr_mock" → a frase do estado devolvido e, depois do refetch, o aviso em destaque "Esta cidade numera as receitas de controlado com o SNCR SIMULADO — numeração sem validade (fora de produção)." na aba e no quadro; "Última mudança" com a hora e `dev@local`; a tela **Auditoria** mostra as mudanças;
- o quadro **SNCR da Anvisa — numeração de receitas**: em dev, o estado da credencial de dev e "SNCR simulado disponível neste ambiente (fora de produção)"; parar o `fake-sncr` (`docker compose stop fake-sncr`), provocar uma checagem pelo caminho que o plano do api definir (pedido de números no dashboard ou a checagem do api) e "atualizar" → "configurado, inalcançável na última checagem" com a hora; religar (`docker compose start fake-sncr`);
- desligar `sncr_mock` → os dois avisos somem e o quadro diz "Sem o SNCR simulado: …";
- nenhuma credencial, CNPJ, token ou número SNCR na tela.

Depois: `docker rm -f maintenance-mod19d && docker compose start maintenance`. Deixe os interruptores da cidade de dev como estavam antes da prova.

- [ ] **Step 5:** **Pare.** O merge do maintenance só vem depois do merge do api e do dashboard do 19d, e só com autorização explícita do usuário (merge, push e cards, uma etapa de cada vez). Antes do push, confira `origin/main..main` no maintenance e publique só os commits desta entrega. Comandos, quando autorizados:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/maintenance
/opt/homebrew/bin/git fetch origin
/opt/homebrew/bin/git status -sb
/opt/homebrew/bin/git merge --ff-only feat/mod-19d-controlled
/opt/homebrew/bin/git log --oneline origin/main..main
/opt/homebrew/bin/git push origin main
/opt/homebrew/bin/git worktree remove .claude/mod19d
/opt/homebrew/bin/git branch -d feat/mod-19d-controlled
```

---

## Divergências propostas ao contrato

1. **M1 — Tipo, nulidade e lugar do `sncrStatus`.** O contrato (§8) dá os campos e tipos, mas não o lugar nem o comportamento em falha. Proposta, na raiz `Query` (é de plataforma, não de cidade; qualquer sessão de manutenção lê), igual ao `signerStatus` do 19b:
   ```graphql
   type SncrStatus { configured: Boolean!, reachable: Boolean!, lastCheckAt: ISO8601DateTime, simulatedAvailable: Boolean! }
   # Query:
   sncrStatus: SncrStatus!   # nunca levanta: sem credencial → configured: false, reachable: false, lastCheckAt: null
   ```
   Se ele levantar, o erro anula o `data` e o quadro mostra só o código do erro.
2. **M2 — O que é a "última checagem".** O contrato não diz. Proposta: a última chamada do api ao SNCR real (autorização do gov.br/troca do token ou pedido de números) com o resultado de transporte (`reachable` = respondeu sem erro de rede/credencial); `lastCheckAt: null` quando nunca houve chamada, e então `reachable: false` sem significado (a tela diz "nunca checado", não "inalcançável"). Sem checagem ativa periódica nesta entrega. O `fake-sncr` não conta como SNCR real.
3. **M3 — `simulatedAvailable`.** Proposta (a do plano do api): `true` quando o ambiente tem o SNCR simulado (fora de produção, com o `fake-sncr` configurado) e por isso oferece `sncr_mock` no catálogo; `false` em produção — independente de alguma cidade ter ligado o simulado.
4. **M4 — Rótulos dos pré-requisitos.** O contrato fixa os códigos `clinical_documents_disabled` (de `controlled_prescriptions`) e `controlled_prescriptions_disabled` (de `sncr_mock`); o plano só acrescenta as frases. Se o api do 19d puser outro pré-requisito em `controlled_prescriptions` (por exemplo, credencial do SNCR em produção), ele aparece cru até ganhar frase — não quebra nada.

## Self-review

- **Cobertura (spec §3, §7 "Maintenance", §11 F-19.24; contrato §8):** interruptores `controlled_prescriptions` e `sncr_mock` pelo mecanismo genérico, com os pré-requisitos em português — Tasks 1 e 2 (testes de dois cliques sem mudar o fluxo); `sncr_mock` só fora de produção (ausente do catálogo → nada quebra, nada diz simulado) — Task 2; aviso de simulado na aba e no quadro, no padrão do `signature_psc_mock` — Tasks 1 e 2; configuração do SNCR só leitura (configurado/alcançável/última checagem/simulado disponível), sem segredo — Tasks 1, 2 e 3 (Step 2); ordem de deploy — README (Task 2), consulta própria e Task 3; prova no navegador — Task 3.
- **Placeholders:** nenhum; todo passo de código tem o código. As únicas variáveis são as do ambiente do api do 19d (nome do container se não for `api`; o caminho que provoca uma checagem do SNCR), que o plano do api define.
- **Tipos:** `SncrStatusView` (Task 1) aceita o tipo gerado da M1 (`lastCheckAt?: Maybe<string>`); `SncrPlatformPanel({ sncrMock })` é o que a `FeaturesTab` monta; o nome da operação `SncrPlatform` é o mesmo no componente e nos testes das duas telas; `CONTROLLED_KEY` e `SNCR_MOCK_KEY` são as chaves do contrato.
- **Review Focus:** 1 — Task 2 ("SNCR simulado ligado…", "ligar sncr_mock por dois cliques…", "SNCR simulado desligado…"); 2 — Task 1 ("disponível fora de produção; indisponível (produção)") e Task 2 ("catálogo sem sncr_mock (produção)…"); 3 — Task 1 ("pré-requisitos do 19d têm frase") e Task 2 ("controlled_prescriptions pelo mecanismo genérico…"); 4 — Task 1 ("estado do SNCR: …") e Task 2 ("SNCR configurado e inalcançável…", "lista o estado sem segredo…"); 5 — Task 2 ("api sem o 19d…" no quadro e na aba).
