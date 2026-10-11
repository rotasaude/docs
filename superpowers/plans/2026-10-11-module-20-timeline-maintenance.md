# Módulo 20 — Linha do tempo do paciente (maintenance) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do módulo 20** rodando na branch dele (`feat/mod-20-timeline`, porta **3040**) com `patient_timeline` e `pec_external_view` no catálogo de interruptores e `cityPecConfig`, `setCityPecOrigin` e `rotateCityPecHmacKey` no `Maintenance::Schema`. Ele é **mergeado antes** deste (contrato §8: api → dashboard e wpda → maintenance). As Tasks 0 e 1 não dependem dele; a Task 2 para se o schema ainda não tiver os campos.

**Goal:** Lado maintenance de F-20.1: os interruptores `patient_timeline` e `pec_external_view` aparecem e são ligados/desligados na aba **Funcionalidades** pelo mecanismo genérico do módulo 16 (só rótulos da chave e dos pré-requisitos), e um quadro **Página do PEC (sistemas externos)** por cidade, com o endereço de origem do PEC (`pec_origin`) editável e a chave HMAC que só se grava ou troca — nunca exibida —, com confirmação de dois cliques.

**Architecture:** Nada muda no liga/desliga: `FeaturesTab` já lista o catálogo do api e chama `setCityFeature`; entram os rótulos em `src/lib/features.ts`. As regras do quadro ficam em `src/lib/pec.ts` (puras: normalizar e conferir a origem, conferir a chave colada, estado da chave, frases de recusa). O quadro `src/screens/PecConfigPanel.tsx` tem **consulta GraphQL própria** (`query CityPecConfig`), no molde do `SncrPlatformPanel` do 19d: contra um api sem o módulo 20 só ele mostra o aviso da ordem de deploy e a aba segue. As duas escritas chamam `gql` direto (sem `useMutation`), para a chave nunca ficar no cache de mutations do TanStack Query; o campo da chave é `type="password"` e é esvaziado depois do sucesso. A `FeaturesTab` monta o quadro quando o catálogo da cidade traz `pec_external_view`.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, GraphQL Codegen (client preset, `src/gql/` gerado e commitado), Vitest 2 + Testing Library 16 + user-event 14 (jsdom, sem jest-dom, `vi.stubGlobal("fetch")`).

**Spec:** `docs/superpowers/specs/2026-10-11-module-20-patient-timeline-design.md` (§3 "Interruptores e a pessoa", §7 "Página do PEC", §8 "maintenance", §9, §12 F-20.1), `docs/adr/0035.md` e o contrato `docs/superpowers/plans/2026-10-11-module-20-timeline-contracts.md` §6 e §7 (o tipo GraphQL exato está em "Divergências propostas ao contrato", M1). Pesquisa: `docs/pesquisa/2026-10-05-integracoes/frente-1-aps-sisab.md` e a documentação oficial dos "sistemas externos" do PEC (integracao.esusaps.bridge.ufsc.tech/sistemas_externos/chave_assinatura.html: "Durante a configuração do sistema externo, o Prontuário Eletrônico e-SUS APS gera uma chave de assinatura" — a chave **vem do PEC** e o mantenedor a cola aqui).

## Global Constraints

- Interruptores (contrato §6; spec §3), pelo mecanismo genérico (`City.features`, `setCityFeature`):
  - `patient_timeline`, sem pré-requisito;
  - `pec_external_view` com `requires: ["feature:patient_timeline", "record_mode:integrated"]`; faltando, `missing` traz `patient_timeline_disabled` e/ou `record_mode_not_integrated` (e, com a Divergência M3, `pec_origin_missing` e `pec_hmac_key_missing`).
  - Ligar não exige o pré-requisito: fica "ligada, falta pré-requisito" (três estados, módulo 16). Nenhuma mudança no fluxo de `setCityFeature`.
- Leitura e escritas novas (contrato §6, forma da Divergência M1):
  ```graphql
  cityPecConfig(citySlug: String!): CityPecConfig            # Query
  # CityPecConfig { pecOrigin: String, hmacKeyConfigured: Boolean!, hmacKeyUpdatedAt: ISO8601DateTime }
  setCityPecOrigin(citySlug: String!, pecOrigin: String!): SetCityPecOriginPayload          # { ok, errors { path message } }
  rotateCityPecHmacKey(citySlug: String!, key: String!): RotateCityPecHmacKeyPayload        # { ok, errors { path message } }
  ```
- A chave HMAC **nunca** volta do api, nunca aparece na tela (o campo é `type="password"`, `autoComplete="off"`), nunca vai para log, `console`, `localStorage`, cache de query nem cache de mutation, e o campo fica vazio depois de gravar. A tela mostra só "configurada" e quando foi trocada.
- `pec_origin` é a **origem** do navegador do PEC (`https://pec.cidade.gov.br`, sem caminho), usada no `Content-Security-Policy: frame-ancestors` da página do PEC (spec §7). Não é o "endereço do PEC" do LEDI (`pec_url`), que segue do operador no console admin; o quadro diz a diferença.
- Valor desconhecido (api mais novo) aparece cru — nunca `undefined`/`null` na tela; nulo vira "não preenchido"/"nunca".
- **Ordem de deploy: api → dashboard e wpda → maintenance** (contrato §8). Contra um api sem o módulo 20 o catálogo não traz `pec_external_view` e o quadro nem aparece; se o catálogo trouxer a chave mas o schema não tiver `cityPecConfig`, só o quadro mostra o aviso da ordem de deploy.
- Interface tem ciclo próprio: componentes e estilo existentes (`Panel`, `Field`, `Button`, `ConfirmButton`, `Tag`, `ErrorState`), sem redesign; o quadro copia a forma do `SncrPlatformPanel`. Nenhum componente comum muda.
- Nunca escreva à mão em arquivo gerado (`schema.graphql`, `src/gql/*`): eles saem do api em execução e do codegen.
- Nunca `git add -A` nem `git add .`: o worktree tem o symlink de `node_modules`. Adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Merge, push e tag só com autorização explícita do usuário.

## Ambiente de execução

- Worktree `apps/maintenance/.claude/mod20`, branch `feat/mod-20-timeline` a partir de `origin/main` (Task 0). O `.gitignore` do maintenance ignora `node_modules` (também o symlink), mas **não** ignora `.claude/`: o checkout principal lista `.claude/` como não rastreado. É esperado; não adicione.
- Caminhos de arquivo das tasks são relativos ao worktree; os comandos partem da raiz do monorepo (`/Users/eduardovrocha/Development/ioit.solutions/rota-saude`).
- Testes: `cd apps/maintenance/.claude/mod20 && npx vitest run <arquivos>`. Tipos e codegen: `npx tsc --noEmit` e `npm run codegen`.
- Ambiente de teste: `environment: "jsdom"`, `globals: false` (todo teste de componente chama `cleanup` no `afterEach`), sem jest-dom (use `not.toBeNull()`, `toBeNull()`, `.textContent`, `(el as HTMLButtonElement).disabled`, `(el as HTMLInputElement).value`). Datas saem de `fmtWhen` (`src/lib/analytics.ts`, `America/Sao_Paulo`: `"2026-10-11T12:00:00Z"` → `"11/10/2026, 09:00"`).
- **`npm run schema:pull` não serve no worktree** (escreveria no checkout principal): a Task 2 extrai o SDL pelo `docker compose exec` com destino no worktree, como no 19d.
- Login de dev (Task 3): `http://maintenance.localhost:5177`, nunca `localhost:5177`. A conta é o `Maintainer` `dev@local` do `db:seed`; o usuário faz o login.

## Review Focus

1. **A chave HMAC vaza da tela.** Depois de gravar, o texto colado não pode continuar no campo, no DOM, no corpo de nenhuma consulta de leitura nem no cache de mutations do TanStack Query; a leitura nunca pede um campo que devolva a chave. Testes: Task 2, "gravar a chave: dois cliques, corpo só na mutation, campo vazio e nada no cache" e "a consulta nunca pede a chave".
2. **Endereço colado com caminho, espaço ou barra final.** O mantenedor cola `https://pec.curitiba.pr.gov.br/esus/#/` ou `pec.curitiba.pr.gov.br`; o `frame-ancestors` precisa da origem pura. A tela normaliza `https://host/` para `https://host`, recusa caminho, query, credencial e esquema que não seja http(s) antes de mandar, e diz como deve ser. Testes: Task 1, "normalizePecOrigin: tabela de casos"; Task 2, "endereço com caminho é recusado na tela; com barra final é normalizado".
3. **Trocar a chave sem querer.** Um clique só nunca troca: o botão vira "confirmar: trocar a chave HMAC", e o aviso diz que a página no PEC passa a recusar até o PEC ter a chave nova. Chave curta demais não habilita o botão. Testes: Task 2, "um clique não troca; o aviso fala do PEC" e "chave curta: botão desligado e o motivo".
4. **`pec_external_view` ligado numa cidade que não é integrada (ou sem a linha do tempo).** Fica "ligada, falta pré-requisito" com o motivo em português, pelo liga/desliga genérico. Testes: Task 1, "pré-requisitos do módulo 20 têm frase"; Task 2, "pec_external_view pelo mecanismo genérico: falta linha do tempo e modo integrado".
5. **Maintenance novo contra api sem o módulo 20 (ou cidade inalcançável).** A validação recusa `CityPecConfig`: só o quadro explica a ordem de deploy e a aba segue; cidade inalcançável mostra o código no quadro, sem `null` na tela. Testes: Task 2, "api sem o módulo 20: o quadro explica a ordem de deploy e a aba segue" e "cidade inalcançável: o código no quadro".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | conferência do main e do api do módulo 20, worktree e base | 0 |
| `src/lib/features.ts`, `src/lib/features.test.ts`, `src/lib/pec.ts`, `src/lib/pec.test.ts` | rótulos das duas chaves e dos pré-requisitos; regras da origem, da chave e das recusas | 1 |
| `schema.graphql`, `src/gql/*` (gerados) | `CityPecConfig`, `cityPecConfig` e as duas mutations | 2 |
| `src/screens/PecConfigPanel.tsx`, `src/screens/PecConfigPanel.test.tsx`, `src/screens/FeaturesTab.tsx`, `src/screens/FeaturesTab.test.tsx`, `README.md` | quadro da página do PEC; montagem na aba com `pec_external_view` no catálogo | 2 |
| — | suíte, codegen:check, build, revisão e prova no navegador | 3 |

**Estratégia de teste:** regras puras com tabela de casos (Task 1); o quadro com `fetch` falso que responde por nome de operação (`query CityPecConfig`, `mutation SetCityPecOrigin`, `mutation RotateCityPecHmacKey`), cobrindo leitura, as duas escritas, as recusas do api, o api sem o módulo 20, a cidade inalcançável e os vazamentos da chave (DOM, corpo da leitura, cache de mutations) (Task 2); os dois interruptores pelo fluxo **existente** de dois cliques da `FeaturesTab`, sem mudar o código do liga/desliga (Task 2). A prova final é no navegador, contra o api do módulo 20 (Task 3).

---

### Task 0: Conferência do main e do api do módulo 20, worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do maintenance (19d entregue, `7db3311` ou mais novo); a branch do api do módulo 20 rodando na 3040.
- Produces: worktree `apps/maintenance/.claude/mod20` na branch `feat/mod-20-timeline`; a base anotada.

- [ ] **Step 1: Confira o main e o api do módulo 20**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude
/opt/homebrew/bin/git -C apps/maintenance fetch origin
/opt/homebrew/bin/git -C apps/maintenance log --oneline -3 origin/main
/opt/homebrew/bin/git -C apps/maintenance show origin/main:src/lib/features.ts | grep -n 'controlled_prescriptions_disabled\|city_unreachable: \|sncr_mock: "SNCR'
/opt/homebrew/bin/git -C apps/maintenance show origin/main:src/screens/FeaturesTab.tsx | grep -n 'import { CONTROLLED_KEY\|<SncrPlatformPanel sncrMock'
/opt/homebrew/bin/git -C apps/maintenance show origin/main:README.md | grep -n 'Receita de controlado e SNCR (módulo 19d)'
docker compose exec -T -w /rails/.claude/mod20 api bin/rails runner \
  'puts Maintenance::Schema.to_definition' 2>/dev/null | grep -n -E "cityPecConfig|setCityPecOrigin|rotateCityPecHmacKey|type CityPecConfig"
```

Expected:
- `origin/main` em `7db3311` ou mais novo;
- em `features.ts`: a linha de `controlled_prescriptions_disabled`, a de `city_unreachable` (a Task 1 insere antes dela) e a de `sncr_mock`;
- em `FeaturesTab.tsx`: o import de `CONTROLLED_KEY` e a linha `<SncrPlatformPanel sncrMock={sncrSimulated} />` (âncoras da Task 2);
- no README, o parágrafo do 19d (a Task 2 escreve depois dele);
- as quatro linhas do schema do api do módulo 20. Sem elas, as Tasks 0 e 1 seguem; **pare antes da Task 2** e reporte.

Se uma âncora mudou, aplique a mesma intenção sobre o texto real.

- [ ] **Step 2: Crie o worktree e ligue o `node_modules`**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/maintenance
/opt/homebrew/bin/git worktree add .claude/mod20 -b feat/mod-20-timeline origin/main
ln -s ../../node_modules .claude/mod20/node_modules
cd .claude/mod20 && npx vitest run 2>&1 | tail -3 && npx tsc --noEmit && echo tsc-ok
```

Expected: suíte verde e `tsc-ok`. **Anote** a base (arquivos e testes): a Task 3 espera a base mais **2 arquivos de teste novos** (`pec.test.ts`, `PecConfigPanel.test.tsx`).

---

### Task 1: Rótulos dos interruptores e regras do quadro do PEC

**Files:**
- Modify: `src/lib/features.ts`, `src/lib/features.test.ts`
- Create: `src/lib/pec.ts`
- Test: `src/lib/pec.test.ts`

**Interfaces:**
- Consumes: `Tone` (`src/theme/tokens.ts`), `fmtWhen` (`src/lib/analytics.ts`), `featureLabel`, `missingText`, `missingSummary`, `UserErrorView` (existentes em `features.ts`).
- Produces:
  - `featureLabel("patient_timeline")` → "Linha do tempo do paciente"; `featureLabel("pec_external_view")` → "Página para os sistemas externos do PEC";
  - `missingText("patient_timeline_disabled")` → "linha do tempo (patient_timeline) desligada"; `missingText("record_mode_not_integrated")` → "modo de prontuário diferente de integrated"; `missingText("pec_origin_missing")` → "endereço de origem do PEC não preenchido"; `missingText("pec_hmac_key_missing")` → "chave HMAC do PEC não gravada";
  - em `src/lib/pec.ts`:
    ```ts
    export const PEC_EXTERNAL_KEY = "pec_external_view";
    export const HMAC_KEY_MIN = 16;
    export const HMAC_KEY_MAX = 512;
    export const PEC_KEY_ROTATE_WARNING: string;
    export const PEC_ORIGIN_HELP: string;
    export function normalizePecOrigin(text: string): string | null;
    export function pecOriginProblem(text: string): string | null;
    export function hmacKeyProblem(text: string): string | null;
    export type PecConfigView = { pecOrigin?: string | null; hmacKeyConfigured: boolean; hmacKeyUpdatedAt?: string | null };
    export function pecOriginText(c: PecConfigView): string;
    export function hmacKeyStatus(c: PecConfigView): { label: string; tone: Tone };
    export function hmacKeyUpdatedText(c: PecConfigView): string;
    export function pecConfigErrorText(errors: UserErrorView[]): string;
    ```

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/pec.test.ts
import { describe, expect, it } from "vitest";
import {
  HMAC_KEY_MIN, PEC_KEY_ROTATE_WARNING, hmacKeyProblem, hmacKeyStatus, hmacKeyUpdatedText, normalizePecOrigin,
  pecConfigErrorText, pecOriginProblem, pecOriginText
} from "./pec";

describe("normalizePecOrigin: tabela de casos", () => {
  it.each([
    [ "https://pec.curitiba.pr.gov.br", "https://pec.curitiba.pr.gov.br" ],
    [ "https://pec.curitiba.pr.gov.br/", "https://pec.curitiba.pr.gov.br" ],
    [ "  https://pec.curitiba.pr.gov.br  ", "https://pec.curitiba.pr.gov.br" ],
    [ "http://10.0.0.5:8080", "http://10.0.0.5:8080" ],
    [ "HTTPS://PEC.Curitiba.PR.gov.br", "https://pec.curitiba.pr.gov.br" ],
    [ "https://pec.curitiba.pr.gov.br:443", "https://pec.curitiba.pr.gov.br" ]
  ])("aceita %s → %s", (input, origin) => {
    expect(normalizePecOrigin(input)).toBe(origin);
  });

  it.each([
    "", "   ", "pec.curitiba.pr.gov.br", "https://pec.curitiba.pr.gov.br/esus", "https://pec.curitiba.pr.gov.br/#/",
    "https://pec.curitiba.pr.gov.br/?a=1", "ftp://pec.curitiba.pr.gov.br", "https://user:senha@pec.curitiba.pr.gov.br",
    "javascript:alert(1)"
  ])("recusa %j", (input) => {
    expect(normalizePecOrigin(input)).toBeNull();
  });
});

describe("pecOriginProblem", () => {
  it("vazio, com caminho e válido", () => {
    expect(pecOriginProblem("")).toBe("informe o endereço de origem do PEC");
    expect(pecOriginProblem("https://pec.curitiba.pr.gov.br/esus"))
      .toBe("use só a origem do PEC, como https://pec.cidade.gov.br — sem caminho, sem # e sem ?");
    expect(pecOriginProblem("https://pec.curitiba.pr.gov.br/")).toBeNull();
  });
});

describe("hmacKeyProblem", () => {
  it("vazia, curta, com espaço no meio, com acento, longa e válida (as pontas são aparadas)", () => {
    expect(hmacKeyProblem("   ")).toBe("cole a chave de assinatura gerada pelo PEC");
    expect(hmacKeyProblem("abc")).toBe(`a chave tem de ter ao menos ${HMAC_KEY_MIN} caracteres`);
    expect(hmacKeyProblem("abcdefgh ijklmnopq")).toBe("a chave não pode ter espaço no meio — confira o que foi colado");
    expect(hmacKeyProblem("chave-de-assinatura-ç")).toBe("a chave só tem letras sem acento, números e símbolos — confira o que foi colado");
    expect(hmacKeyProblem("a".repeat(513))).toBe("a chave tem de ter no máximo 512 caracteres");
    expect(hmacKeyProblem("a".repeat(512))).toBeNull();
    expect(hmacKeyProblem("  0123456789abcdef0123  \n")).toBeNull();
  });
});

describe("estado da configuração", () => {
  it("origem: o valor ou 'não preenchido'", () => {
    expect(pecOriginText({ pecOrigin: "https://pec.curitiba.pr.gov.br", hmacKeyConfigured: false })).toBe("https://pec.curitiba.pr.gov.br");
    expect(pecOriginText({ pecOrigin: null, hmacKeyConfigured: false })).toBe("não preenchido");
    expect(pecOriginText({ hmacKeyConfigured: false })).toBe("não preenchido");
  });

  it("chave: configurada ou não, sem nunca mostrar a chave", () => {
    expect(hmacKeyStatus({ hmacKeyConfigured: true, hmacKeyUpdatedAt: "2026-10-11T12:00:00Z" }))
      .toEqual({ label: "chave configurada", tone: "ok" });
    expect(hmacKeyStatus({ hmacKeyConfigured: false })).toEqual({ label: "chave não gravada — o PEC não consegue abrir a página", tone: "warn" });
    expect(hmacKeyUpdatedText({ hmacKeyConfigured: true, hmacKeyUpdatedAt: "2026-10-11T12:00:00Z" })).toBe("trocada em 11/10/2026, 09:00");
    expect(hmacKeyUpdatedText({ hmacKeyConfigured: true, hmacKeyUpdatedAt: null })).toBe("trocada em —");
    expect(hmacKeyUpdatedText({ hmacKeyConfigured: false })).toBe("nunca gravada");
  });

  it("o aviso de troca fala do PEC e da recusa", () => {
    expect(PEC_KEY_ROTATE_WARNING).toMatch(/PEC/);
    expect(PEC_KEY_ROTATE_WARNING).toMatch(/recusa/);
  });
});

describe("pecConfigErrorText", () => {
  it("traduz os códigos do contrato e mostra o resto cru", () => {
    expect(pecConfigErrorText([ { path: "pecOrigin", message: "invalid_pec_origin" } ]))
      .toBe("o api recusou o endereço — use só a origem do PEC, como https://pec.cidade.gov.br");
    expect(pecConfigErrorText([ { path: "key", message: "invalid_key" } ])).toBe("o api recusou a chave — confira o que foi colado");
    expect(pecConfigErrorText([ { path: null, message: "unknown_city" } ])).toBe("cidade inexistente");
    expect(pecConfigErrorText([ { path: null, message: "algo novo" } ])).toBe("algo novo");
    expect(pecConfigErrorText([])).toBe("não foi possível concluir — tente de novo");
  });
});
```

Em `src/lib/features.test.ts`, no `describe("missingText e missingSummary")`, depois do teste "pré-requisitos do 19d têm frase":

```ts
  it("pré-requisitos do módulo 20 têm frase", () => {
    expect(missingText("patient_timeline_disabled")).toBe("linha do tempo (patient_timeline) desligada");
    expect(missingText("record_mode_not_integrated")).toBe("modo de prontuário diferente de integrated");
    expect(missingText("pec_origin_missing")).toBe("endereço de origem do PEC não preenchido");
    expect(missingText("pec_hmac_key_missing")).toBe("chave HMAC do PEC não gravada");
    expect(missingSummary([ "patient_timeline_disabled", "record_mode_not_integrated" ]))
      .toBe("linha do tempo (patient_timeline) desligada; modo de prontuário diferente de integrated");
  });
```

e, logo depois do teste "as chaves do 19d têm nome em português":

```ts
  it("as chaves do módulo 20 têm nome em português", () => {
    expect(featureLabel("patient_timeline")).toBe("Linha do tempo do paciente");
    expect(featureLabel("pec_external_view")).toBe("Página para os sistemas externos do PEC");
  });
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/maintenance/.claude/mod20 && npx vitest run src/lib/pec.test.ts src/lib/features.test.ts`
Expected: FAIL — `./pec` não existe; `missingText("patient_timeline_disabled")` devolve o código cru; `featureLabel("patient_timeline")` devolve `null`.

- [ ] **Step 3: Write minimal implementation**

```ts
// src/lib/pec.ts
// Regras do quadro "Página do PEC (sistemas externos)" (módulo 20, F-20.1;
// spec §3 e §7; contrato §6). Por cidade, gravados só pelo maintenance:
// a ORIGEM do PEC (para o frame-ancestors da página) e a chave HMAC que o
// próprio PEC gera na configuração do sistema externo. A chave nunca volta
// do api e nunca aparece aqui: a tela só sabe se ela existe e quando mudou.
import type { Tone } from "../theme/tokens";
import { fmtWhen } from "./analytics";
import type { UserErrorView } from "./features";

export const PEC_EXTERNAL_KEY = "pec_external_view";
export const HMAC_KEY_MIN = 16;
export const HMAC_KEY_MAX = 512;

export const PEC_ORIGIN_HELP =
  "Endereço de onde a equipe abre o PEC no navegador (só a origem). Não é o endereço do LEDI, que o operador cadastra no console admin.";
export const PEC_KEY_ROTATE_WARNING =
  "Gravar uma chave nova invalida a anterior na hora: até o PEC da cidade ser configurado com a mesma chave, a página do Rota Saúde dentro do PEC recusa todas as aberturas.";

// Origem pura (esquema + host + porta): o frame-ancestors não aceita caminho.
// `https://host/` vira `https://host`; caminho, query, fragmento, credencial
// ou esquema que não seja http(s) → null.
export function normalizePecOrigin(text: string): string | null {
  const raw = text.trim();
  if (raw === "") return null;
  let url: URL;
  try { url = new URL(raw); } catch { return null; }
  if (url.protocol !== "https:" && url.protocol !== "http:") return null;
  if (url.username !== "" || url.password !== "") return null;
  if (url.pathname !== "/" || url.search !== "" || url.hash !== "") return null;
  if (/[#?]/.test(raw)) return null;
  return url.origin;
}

export function pecOriginProblem(text: string): string | null {
  if (text.trim() === "") return "informe o endereço de origem do PEC";
  return normalizePecOrigin(text) ? null : "use só a origem do PEC, como https://pec.cidade.gov.br — sem caminho, sem # e sem ?";
}

// As pontas são aparadas (o colar costuma trazer quebra de linha); o resto
// vai como veio. Espelho do api (16–512 caracteres ASCII visíveis); quem
// decide é ele.
export function hmacKeyProblem(text: string): string | null {
  const key = text.trim();
  if (key === "") return "cole a chave de assinatura gerada pelo PEC";
  if (/\s/.test(key)) return "a chave não pode ter espaço no meio — confira o que foi colado";
  if (!/^[\x21-\x7E]+$/.test(key)) return "a chave só tem letras sem acento, números e símbolos — confira o que foi colado";
  if (key.length < HMAC_KEY_MIN) return `a chave tem de ter ao menos ${HMAC_KEY_MIN} caracteres`;
  if (key.length > HMAC_KEY_MAX) return `a chave tem de ter no máximo ${HMAC_KEY_MAX} caracteres`;
  return null;
}

export type PecConfigView = { pecOrigin?: string | null; hmacKeyConfigured: boolean; hmacKeyUpdatedAt?: string | null };

export function pecOriginText(c: PecConfigView): string {
  return c.pecOrigin ? c.pecOrigin : "não preenchido";
}

export function hmacKeyStatus(c: PecConfigView): { label: string; tone: Tone } {
  return c.hmacKeyConfigured
    ? { label: "chave configurada", tone: "ok" }
    : { label: "chave não gravada — o PEC não consegue abrir a página", tone: "warn" };
}

export function hmacKeyUpdatedText(c: PecConfigView): string {
  if (!c.hmacKeyConfigured) return "nunca gravada";
  return `trocada em ${c.hmacKeyUpdatedAt ? fmtWhen(c.hmacKeyUpdatedAt) : "—"}`;
}

const ERROR_LABELS: Record<string, string> = {
  invalid_pec_origin: "o api recusou o endereço — use só a origem do PEC, como https://pec.cidade.gov.br",
  invalid_key: "o api recusou a chave — confira o que foi colado",
  unknown_city: "cidade inexistente",
  city_unreachable: "banco da cidade inalcançável — tente de novo"
};

export function pecConfigErrorText(errors: UserErrorView[]): string {
  if (errors.length === 0) return "não foi possível concluir — tente de novo";
  return errors.map((e) => ERROR_LABELS[e.message] ?? e.message).join("; ");
}
```

Em `src/lib/features.ts`, no `MISSING_LABELS`, logo antes do comentário de `city_unreachable`:

```ts
  // Módulo 20: pec_external_view exige patient_timeline ligado e o modo
  // integrated; com a Divergência M3, também a origem e a chave do PEC.
  patient_timeline_disabled: "linha do tempo (patient_timeline) desligada",
  record_mode_not_integrated: "modo de prontuário diferente de integrated",
  pec_origin_missing: "endereço de origem do PEC não preenchido",
  pec_hmac_key_missing: "chave HMAC do PEC não gravada",
```

e no `FEATURE_LABELS`, depois da linha de `sncr_mock`:

```ts
  patient_timeline: "Linha do tempo do paciente",
  pec_external_view: "Página para os sistemas externos do PEC",
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/maintenance/.claude/mod20 && npx vitest run src/lib/pec.test.ts src/lib/features.test.ts && npx tsc --noEmit`
Expected: PASS e `tsc` sem erro.

- [ ] **Step 5: Commit**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/maintenance/.claude/mod20
/opt/homebrew/bin/git add src/lib/pec.ts src/lib/pec.test.ts src/lib/features.ts src/lib/features.test.ts
/opt/homebrew/bin/git commit -m "feat: label the patient timeline and PEC external view switches and add PEC config rules

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Schema, codegen, quadro da página do PEC e os interruptores na aba

**Files:**
- Gerados: `schema.graphql`, `src/gql/gql.ts`, `src/gql/graphql.ts`
- Create: `src/screens/PecConfigPanel.tsx`
- Modify: `src/screens/FeaturesTab.tsx`, `src/screens/FeaturesTab.test.tsx`, `README.md`
- Test: `src/screens/PecConfigPanel.test.tsx`, `src/screens/FeaturesTab.test.tsx`

**Interfaces:**
- Consumes: `PEC_EXTERNAL_KEY`, `PEC_KEY_ROTATE_WARNING`, `PEC_ORIGIN_HELP`, `normalizePecOrigin`, `pecOriginProblem`, `hmacKeyProblem`, `pecOriginText`, `hmacKeyStatus`, `hmacKeyUpdatedText`, `pecConfigErrorText` (Task 1); `gql` (`src/lib/api.ts`), `graphql` (`src/gql`), `GraphQLRefusal` (`src/lib/errors.ts`), `Panel`, `Field`, `Button`, `ConfirmButton`, `Tag`, `ErrorState`.
- Produces: `export function PecConfigPanel({ slug }: { slug: string }): JSX.Element` — `<section aria-label="Página do PEC (sistemas externos)">` com um `Panel` (botão "atualizar"), o grupo `role="group"` "configuração do PEC" (origem atual, selo da chave, "trocada em"), o formulário da origem (campo "Endereço de origem do PEC", botão "salvar endereço") e o da chave (campo "Chave de assinatura do PEC", `type="password"`, `ConfirmButton` "gravar a chave"/"trocar a chave" → "confirmar: gravar a chave HMAC"/"confirmar: trocar a chave HMAC"), a nota `role="note"` "aviso da troca da chave" com `PEC_KEY_ROTATE_WARNING` e `role="status"` depois de cada escrita. Operações com nome fixo: `query CityPecConfig`, `mutation SetCityPecOrigin`, `mutation RotateCityPecHmacKey`; chave de cache `[ "city", slug, "pec" ]`. A `FeaturesTab` monta o quadro quando o catálogo traz `pec_external_view`.

- [ ] **Step 1: Extraia o schema do api do módulo 20**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose exec -T -w /rails/.claude/mod20 api bin/rails runner 'puts Maintenance::Schema.to_definition' \
  > apps/maintenance/.claude/mod20/schema.graphql
/opt/homebrew/bin/git -C apps/maintenance/.claude/mod20 diff --stat
/opt/homebrew/bin/git -C apps/maintenance/.claude/mod20 diff schema.graphql | grep '^[+-]' | grep -v '^+++\|^---'
```

(Se o plano do api subir o servidor do módulo 20 num container próprio, use o nome dele no lugar de `api`.)

Expected: só `schema.graphql` mudou, com o acréscimo (na ordem que o `to_definition` escrever, mais as descrições `"""…"""`):
```
+type CityPecConfig {
+  hmacKeyConfigured: Boolean!
+  hmacKeyUpdatedAt: ISO8601DateTime
+  pecOrigin: String
+}
+  cityPecConfig(citySlug: String!): CityPecConfig
+  rotateCityPecHmacKey(citySlug: String!, key: String!): RotateCityPecHmacKeyPayload
+  setCityPecOrigin(citySlug: String!, pecOrigin: String!): SetCityPecOriginPayload
+type RotateCityPecHmacKeyPayload {
+  errors: [UserError!]!
+  ok: Boolean!
+}
+type SetCityPecOriginPayload {
+  errors: [UserError!]!
+  ok: Boolean!
+}
```
Se o tipo tiver **qualquer** campo que devolva a chave (`hmacKey: String`, `key`, `secret`), **pare e reporte**: contraria o invariante do ADR 0035. Se aparecerem outras mudanças, reporte as linhas antes de seguir.

- [ ] **Step 2: Write the failing test**

```tsx
// src/screens/PecConfigPanel.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, render, screen, waitFor, within } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";
import { PecConfigPanel } from "./PecConfigPanel";
import { PEC_KEY_ROTATE_WARNING } from "../lib/pec";

afterEach(() => { cleanup(); vi.unstubAllGlobals(); });

function reply(status: number, body: unknown) {
  return new Response(JSON.stringify(body), { status, headers: { "Content-Type": "application/json" } });
}
function bodyOf(call: unknown[]): { query: string; variables: Record<string, unknown> } {
  return JSON.parse((call[1] as RequestInit).body as string);
}
function calls(fetchMock: ReturnType<typeof vi.fn>, operation: string) {
  return fetchMock.mock.calls.filter((call) => bodyOf(call).query.includes(operation));
}

const KEY = "9f2c4e7a1b3d5f60a8c2e4f6b1d3a5c7";
const CONFIGURED = { pecOrigin: "https://pec.curitiba.pr.gov.br", hmacKeyConfigured: true, hmacKeyUpdatedAt: "2026-10-11T12:00:00Z" };
const EMPTY = { pecOrigin: null, hmacKeyConfigured: false, hmacKeyUpdatedAt: null };

function renderPanel() {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  function wrapper({ children }: { children: ReactNode }) {
    return <QueryClientProvider client={client}>{children}</QueryClientProvider>;
  }
  return { client, ...render(<PecConfigPanel slug="curitiba" />, { wrapper }) };
}

describe("PecConfigPanel", () => {
  let fetchMock: ReturnType<typeof vi.fn>;
  let config: typeof CONFIGURED | typeof EMPTY;

  beforeEach(() => {
    config = CONFIGURED;
    fetchMock = vi.fn((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityPecOrigin")) {
        config = { ...config, pecOrigin: body.variables.pecOrigin as string };
        return Promise.resolve(reply(200, { data: { setCityPecOrigin: { ok: true, errors: [] } } }));
      }
      if (body.query.includes("mutation RotateCityPecHmacKey")) {
        config = { ...config, hmacKeyConfigured: true, hmacKeyUpdatedAt: "2026-10-11T13:00:00Z" };
        return Promise.resolve(reply(200, { data: { rotateCityPecHmacKey: { ok: true, errors: [] } } }));
      }
      return Promise.resolve(reply(200, { data: { cityPecConfig: config } }));
    });
    vi.stubGlobal("fetch", fetchMock);
  });

  it("mostra a origem e o estado da chave, sem null", async () => {
    renderPanel();
    const group = await screen.findByRole("group", { name: "configuração do PEC" });
    expect(within(group).getByText("https://pec.curitiba.pr.gov.br")).not.toBeNull();
    expect(within(group).getByText("chave configurada")).not.toBeNull();
    expect(within(group).getByText("trocada em 11/10/2026, 09:00")).not.toBeNull();
    expect(screen.getByRole("button", { name: "trocar a chave" })).not.toBeNull();
    expect(document.body.textContent).not.toMatch(/undefined|null/);
    expect(bodyOf(fetchMock.mock.calls[0]).variables).toEqual({ slug: "curitiba" });
  });

  it("a consulta nunca pede a chave", async () => {
    renderPanel();
    await screen.findByRole("group", { name: "configuração do PEC" });
    const query = bodyOf(calls(fetchMock, "query CityPecConfig")[0]).query;
    expect(query).toMatch(/hmacKeyConfigured/);
    expect(query.replace(/hmacKeyConfigured|hmacKeyUpdatedAt/g, "")).not.toMatch(/hmacKey|secret|\bkey\b/i);
  });

  it("nada gravado: diz que o PEC não abre e oferece 'gravar a chave'", async () => {
    config = EMPTY;
    renderPanel();
    const group = await screen.findByRole("group", { name: "configuração do PEC" });
    expect(within(group).getByText("não preenchido")).not.toBeNull();
    expect(within(group).getByText("chave não gravada — o PEC não consegue abrir a página")).not.toBeNull();
    expect(within(group).getByText("nunca gravada")).not.toBeNull();
    expect(screen.getByRole("button", { name: "gravar a chave" })).not.toBeNull();
  });

  it("endereço com caminho é recusado na tela; com barra final é normalizado", async () => {
    const user = userEvent.setup();
    renderPanel();
    await screen.findByRole("group", { name: "configuração do PEC" });
    const field = screen.getByLabelText("Endereço de origem do PEC") as HTMLInputElement;
    expect(field.value).toBe("https://pec.curitiba.pr.gov.br");

    await user.clear(field);
    await user.type(field, "https://pec.curitiba.pr.gov.br/esus");
    await user.click(screen.getByRole("button", { name: "salvar endereço" }));
    expect((await screen.findByRole("alert")).textContent).toMatch(/sem caminho/);
    expect(calls(fetchMock, "mutation SetCityPecOrigin")).toHaveLength(0);

    await user.clear(field);
    await user.type(field, "https://pec2.curitiba.pr.gov.br/");
    await user.click(screen.getByRole("button", { name: "salvar endereço" }));
    await waitFor(() => expect(calls(fetchMock, "mutation SetCityPecOrigin")).toHaveLength(1));
    expect(bodyOf(calls(fetchMock, "mutation SetCityPecOrigin")[0]).variables)
      .toEqual({ citySlug: "curitiba", pecOrigin: "https://pec2.curitiba.pr.gov.br" });
    expect((await screen.findByRole("status")).textContent).toBe("Endereço de origem do PEC salvo.");
    const group = await screen.findByRole("group", { name: "configuração do PEC" });
    await within(group).findByText("https://pec2.curitiba.pr.gov.br");
  });

  it("recusa do api ao salvar o endereço aparece traduzida", async () => {
    const user = userEvent.setup();
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityPecOrigin")) {
        return Promise.resolve(reply(200, { data: { setCityPecOrigin: { ok: false, errors: [ { path: "pecOrigin", message: "invalid_pec_origin" } ] } } }));
      }
      return Promise.resolve(reply(200, { data: { cityPecConfig: CONFIGURED } }));
    });
    renderPanel();
    await screen.findByRole("group", { name: "configuração do PEC" });
    await user.click(screen.getByRole("button", { name: "salvar endereço" }));
    expect((await screen.findByRole("alert")).textContent)
      .toBe("o api recusou o endereço — use só a origem do PEC, como https://pec.cidade.gov.br");
  });

  it("chave curta: botão desligado e o motivo", async () => {
    const user = userEvent.setup();
    renderPanel();
    await screen.findByRole("group", { name: "configuração do PEC" });
    expect((screen.getByRole("button", { name: "trocar a chave" }) as HTMLButtonElement).disabled).toBe(true);
    await user.type(screen.getByLabelText("Chave de assinatura do PEC"), "curta");
    expect(screen.getByText("a chave tem de ter ao menos 16 caracteres")).not.toBeNull();
    expect((screen.getByRole("button", { name: "trocar a chave" }) as HTMLButtonElement).disabled).toBe(true);
  });

  it("um clique não troca; o aviso fala do PEC", async () => {
    const user = userEvent.setup();
    renderPanel();
    await screen.findByRole("group", { name: "configuração do PEC" });
    expect(screen.getByRole("note", { name: "aviso da troca da chave" }).textContent).toBe(PEC_KEY_ROTATE_WARNING);
    await user.type(screen.getByLabelText("Chave de assinatura do PEC"), KEY);
    await user.click(screen.getByRole("button", { name: "trocar a chave" }));
    expect(calls(fetchMock, "mutation RotateCityPecHmacKey")).toHaveLength(0);
    expect(screen.getByRole("button", { name: "confirmar: trocar a chave HMAC" })).not.toBeNull();
  });

  it("gravar a chave: dois cliques, corpo só na mutation, campo vazio e nada no cache", async () => {
    const user = userEvent.setup();
    const { client } = renderPanel();
    await screen.findByRole("group", { name: "configuração do PEC" });
    const field = screen.getByLabelText("Chave de assinatura do PEC") as HTMLInputElement;
    expect(field.type).toBe("password");
    expect(field.getAttribute("autocomplete")).toBe("off");

    await user.type(field, `  ${KEY}  `);
    await user.click(screen.getByRole("button", { name: "trocar a chave" }));
    await user.click(screen.getByRole("button", { name: "confirmar: trocar a chave HMAC" }));

    await waitFor(() => expect(calls(fetchMock, "mutation RotateCityPecHmacKey")).toHaveLength(1));
    expect(bodyOf(calls(fetchMock, "mutation RotateCityPecHmacKey")[0]).variables).toEqual({ citySlug: "curitiba", key: KEY });
    expect((await screen.findByRole("status")).textContent)
      .toBe("Chave gravada. Configure a mesma chave no PEC da cidade, se ainda não estiver lá.");
    expect((screen.getByLabelText("Chave de assinatura do PEC") as HTMLInputElement).value).toBe("");
    await screen.findByText("trocada em 11/10/2026, 10:00");

    // A chave só viajou no corpo da mutation: nenhuma leitura, nenhum cache, nenhum texto da tela.
    for (const call of fetchMock.mock.calls) {
      const body = bodyOf(call);
      if (!body.query.includes("mutation RotateCityPecHmacKey")) expect(JSON.stringify(body)).not.toContain(KEY);
      expect(String(call[0])).not.toContain(KEY);
    }
    expect(JSON.stringify(client.getMutationCache().getAll().map((m) => m.state.variables))).not.toContain(KEY);
    expect(JSON.stringify(client.getQueryCache().getAll().map((q) => q.state.data))).not.toContain(KEY);
    expect(document.body.innerHTML).not.toContain(KEY);
  });

  it("recusa do api ao gravar a chave: diz o motivo e mantém o campo para corrigir", async () => {
    const user = userEvent.setup();
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation RotateCityPecHmacKey")) {
        return Promise.resolve(reply(200, { data: { rotateCityPecHmacKey: { ok: false, errors: [ { path: "key", message: "invalid_key" } ] } } }));
      }
      return Promise.resolve(reply(200, { data: { cityPecConfig: CONFIGURED } }));
    });
    renderPanel();
    await screen.findByRole("group", { name: "configuração do PEC" });
    await user.type(screen.getByLabelText("Chave de assinatura do PEC"), KEY);
    await user.click(screen.getByRole("button", { name: "trocar a chave" }));
    await user.click(screen.getByRole("button", { name: "confirmar: trocar a chave HMAC" }));
    expect((await screen.findByRole("alert")).textContent).toBe("o api recusou a chave — confira o que foi colado");
    expect((screen.getByLabelText("Chave de assinatura do PEC") as HTMLInputElement).value).toBe(KEY);
    expect(screen.queryByRole("status")).toBeNull();
  });

  it("api sem o módulo 20: explica a ordem de deploy", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(reply(200, {
      errors: [ { message: "Field 'cityPecConfig' doesn't exist on type 'Query'",
        extensions: { code: "undefinedField", typeName: "Query", fieldName: "cityPecConfig" } } ]
    })));
    renderPanel();
    expect((await screen.findByRole("alert")).textContent).toMatch(/o api do módulo 20 precisa subir antes do maintenance/);
    expect(screen.queryByLabelText("Chave de assinatura do PEC")).toBeNull();
  });

  it("cidade inalcançável: o código no quadro", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(reply(200, {
      data: { cityPecConfig: null },
      errors: [ { message: "city database unreachable", path: [ "cityPecConfig" ], extensions: { code: "CITY_UNREACHABLE" } } ]
    })));
    renderPanel();
    expect((await screen.findByRole("alert")).textContent).toBe("CITY_UNREACHABLE — city database unreachable");
    expect(document.body.textContent).not.toMatch(/undefined|null/);
  });

  it("atualizar relê", async () => {
    const user = userEvent.setup();
    renderPanel();
    await screen.findByRole("group", { name: "configuração do PEC" });
    await user.click(screen.getByRole("button", { name: "atualizar" }));
    await waitFor(() => expect(calls(fetchMock, "query CityPecConfig")).toHaveLength(2));
  });
});
```

(`12:00Z` é 09:00 em São Paulo; o "gravar a chave" espera `10:00`, de `13:00Z`.)

Em `src/screens/FeaturesTab.test.tsx`, no fim do arquivo, um `describe` novo:

```tsx
describe("FeaturesTab — linha do tempo e página do PEC (módulo 20)", () => {
  let fetchMock: ReturnType<typeof vi.fn>;

  beforeEach(() => {
    fetchMock = vi.fn();
    vi.stubGlobal("fetch", fetchMock);
  });

  afterEach(() => vi.unstubAllGlobals());

  const TIMELINE_OFF: Feature = {
    key: "patient_timeline", description: "Linha do tempo do paciente (profissional e cidadão)",
    enabled: false, usable: false, missing: [], changedAt: null, changedBy: null
  };
  const PEC_OFF: Feature = {
    key: "pec_external_view", description: "Página para a tela de sistemas externos do e-SUS PEC",
    enabled: false, usable: false, missing: [ "patient_timeline_disabled", "record_mode_not_integrated" ],
    changedAt: null, changedBy: null
  };
  const pecConfig = () => reply(200, { data: {
    cityPecConfig: { pecOrigin: null, hmacKeyConfigured: false, hmacKeyUpdatedAt: null }
  } });

  it("patient_timeline tem rótulo e liga com dois cliques, sem pré-requisito", async () => {
    const user = userEvent.setup();
    const turnedOn: Feature = { ...TIMELINE_OFF, enabled: true, usable: true, changedAt: "2026-10-11T13:00:00Z", changedBy: "dev@local" };
    let listed = [ TIMELINE_OFF ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [ turnedOn ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: turnedOn } } }));
      }
      return Promise.resolve(featuresReply(listed));
    });
    renderTab();
    expect(await screen.findByText("patient_timeline")).not.toBeNull();
    expect(screen.getByText("Linha do tempo do paciente")).not.toBeNull();
    await user.click(screen.getByRole("button", { name: "ligar" }));
    await user.click(screen.getByRole("button", { name: "confirmar: ligar patient_timeline" }));
    await waitFor(() => expect(calls(fetchMock, "mutation SetCityFeature")).toHaveLength(1));
    expect(bodyOf(calls(fetchMock, "mutation SetCityFeature")[0]).variables)
      .toEqual({ citySlug: "sp", key: "patient_timeline", enabled: true });
    expect((await screen.findByRole("status")).textContent).toBe("patient_timeline ligada e utilizável.");
    expect(calls(fetchMock, "query CityPecConfig")).toHaveLength(0);
  });

  it("pec_external_view pelo mecanismo genérico: falta linha do tempo e modo integrado", async () => {
    const user = userEvent.setup();
    const turnedOn: Feature = { ...PEC_OFF, enabled: true, changedAt: "2026-10-11T13:00:00Z", changedBy: "dev@local" };
    let listed = [ PEC_OFF ];
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("query CityPecConfig")) return Promise.resolve(pecConfig());
      if (body.query.includes("mutation SetCityFeature")) {
        listed = [ turnedOn ];
        return Promise.resolve(reply(200, { data: { setCityFeature: { ok: true, errors: [], feature: turnedOn } } }));
      }
      return Promise.resolve(featuresReply(listed, { recordMode: "record" }));
    });
    renderTab();
    expect(await screen.findByText("pec_external_view")).not.toBeNull();
    expect(screen.getByText("Página para os sistemas externos do PEC")).not.toBeNull();
    expect(screen.getByText("linha do tempo (patient_timeline) desligada; modo de prontuário diferente de integrated")).not.toBeNull();

    await user.click(screen.getByRole("button", { name: "ligar" }));
    await user.click(screen.getByRole("button", { name: "confirmar: ligar pec_external_view" }));
    expect((await screen.findByRole("status")).textContent).toBe(
      "pec_external_view ligada, mas ainda não utilizável — falta: linha do tempo (patient_timeline) desligada; modo de prontuário diferente de integrated."
    );
  });

  it("com pec_external_view no catálogo mostra o quadro do PEC; sem ela, nem consulta", async () => {
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("query CityPecConfig")) return Promise.resolve(pecConfig());
      return Promise.resolve(featuresReply([ TIMELINE_OFF, PEC_OFF ]));
    });
    renderTab();
    expect(await screen.findByRole("region", { name: "Página do PEC (sistemas externos)" })).not.toBeNull();
    expect(await screen.findByText("chave não gravada — o PEC não consegue abrir a página")).not.toBeNull();
    expect(bodyOf(calls(fetchMock, "query CityPecConfig")[0]).variables).toEqual({ slug: "sp" });
    cleanup();

    fetchMock.mockReset();
    fetchMock.mockImplementation(() => Promise.resolve(featuresReply([ LEDI_OFF ])));
    renderTab();
    await screen.findByText("ledi_export");
    expect(calls(fetchMock, "query CityPecConfig")).toHaveLength(0);
  });

  it("api sem o módulo 20: o quadro explica a ordem de deploy e a aba segue", async () => {
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("query CityPecConfig")) {
        return Promise.resolve(reply(200, { errors: [ {
          message: "Field 'cityPecConfig' doesn't exist on type 'Query'",
          extensions: { code: "undefinedField", typeName: "Query", fieldName: "cityPecConfig" }
        } ] }));
      }
      return Promise.resolve(featuresReply([ PEC_OFF ]));
    });
    renderTab();
    expect((await screen.findByRole("alert")).textContent).toMatch(/o api do módulo 20 precisa subir antes do maintenance/);
    expect(screen.getByText("pec_external_view")).not.toBeNull();
    expect(screen.getByRole("button", { name: "ligar" })).not.toBeNull();
  });
});
```

(`reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab`, `Feature` e `LEDI_OFF` já estão no topo do arquivo, do módulo 16.)

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/screens/PecConfigPanel.tsx
// Página do PEC (sistemas externos) por cidade (módulo 20, F-20.1; spec §3,
// §7 e §8; contrato §6). Gravados só aqui: a ORIGEM do PEC (frame-ancestors
// da página) e a chave HMAC que o próprio PEC gera. A chave nunca volta do
// api e nunca aparece na tela: o campo é de senha, as escritas chamam `gql`
// direto (sem useMutation, para a chave não ficar no cache de mutations) e o
// campo fica vazio depois de gravar.
//
// Consulta PRÓPRIA, como o SncrPlatformPanel do 19d: contra um api sem o
// módulo 20 a validação recusa o documento inteiro, e só este quadro cai.
import { useEffect, useState, type CSSProperties } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { gql } from "../lib/api";
import { graphql } from "../gql";
import { GraphQLRefusal } from "../lib/errors";
import {
  PEC_KEY_ROTATE_WARNING, PEC_ORIGIN_HELP, hmacKeyProblem, hmacKeyStatus, hmacKeyUpdatedText, normalizePecOrigin,
  pecConfigErrorText, pecOriginProblem, pecOriginText
} from "../lib/pec";
import { Panel } from "../components/Panel";
import { Field } from "../components/Field";
import { Button } from "../components/Button";
import { ConfirmButton } from "../components/ConfirmButton";
import { Tag } from "../components/Tag";
import { ErrorState } from "../components/ErrorState";

const CityPecConfigQuery = graphql(`
  query CityPecConfig($slug: String!) {
    cityPecConfig(citySlug: $slug) { pecOrigin hmacKeyConfigured hmacKeyUpdatedAt }
  }
`);

const SetCityPecOriginMutation = graphql(`
  mutation SetCityPecOrigin($citySlug: String!, $pecOrigin: String!) {
    setCityPecOrigin(citySlug: $citySlug, pecOrigin: $pecOrigin) { ok errors { path message } }
  }
`);

const RotateCityPecHmacKeyMutation = graphql(`
  mutation RotateCityPecHmacKey($citySlug: String!, $key: String!) {
    rotateCityPecHmacKey(citySlug: $citySlug, key: $key) { ok errors { path message } }
  }
`);

const TITLE = "Página do PEC (sistemas externos)";
const OLD_API_CODES = new Set([ "undefinedField" ]);

function requestErrorText(err: unknown): string {
  if (err instanceof GraphQLRefusal && OLD_API_CODES.has(err.code)) {
    return "esta API ainda não tem a página do PEC — o api do módulo 20 precisa subir antes do maintenance";
  }
  return err instanceof Error ? err.message : "erro inesperado";
}

type Payload = { ok: boolean; errors: { path?: string | null; message: string }[] } | null | undefined;

function payloadFailure(payload: Payload, fieldErrors: GraphQLRefusal[]): string | null {
  if (payload == null) {
    const refusal = fieldErrors[0];
    return refusal ? `${refusal.code} — ${refusal.message}` : pecConfigErrorText([]);
  }
  return payload.ok ? null : pecConfigErrorText(payload.errors);
}

export function PecConfigPanel({ slug }: { slug: string }) {
  const queryClient = useQueryClient();
  const queryKey = [ "city", slug, "pec" ];
  const query = useQuery({ queryKey, queryFn: () => gql(CityPecConfigQuery, { slug }), staleTime: Infinity });
  const config = query.data?.data?.cityPecConfig ?? null;
  const fieldError = query.data?.fieldErrors[0];

  const [ originText, setOriginText ] = useState("");
  const [ originTouched, setOriginTouched ] = useState(false);
  const [ keyText, setKeyText ] = useState("");
  const [ busy, setBusy ] = useState<"origin" | "key" | null>(null);
  const [ done, setDone ] = useState<string | null>(null);
  const [ failure, setFailure ] = useState<string | null>(null);

  // O campo começa com o valor gravado; depois de mexer, o que vale é o digitado.
  useEffect(() => {
    if (!originTouched) setOriginText(config?.pecOrigin ?? "");
  }, [ config?.pecOrigin, originTouched ]);

  const keyProblem = keyText === "" ? null : hmacKeyProblem(keyText);
  const configured = config?.hmacKeyConfigured ?? false;

  async function saveOrigin() {
    setDone(null); setFailure(null);
    const problem = pecOriginProblem(originText);
    const origin = normalizePecOrigin(originText);
    if (problem || !origin) { setFailure(problem ?? pecOriginProblem("")); return; }
    setBusy("origin");
    try {
      const { data, fieldErrors } = await gql(SetCityPecOriginMutation, { citySlug: slug, pecOrigin: origin });
      const fail = payloadFailure(data?.setCityPecOrigin, fieldErrors);
      if (fail) { setFailure(fail); return; }
      setOriginTouched(false);
      setDone("Endereço de origem do PEC salvo.");
      void queryClient.invalidateQueries({ queryKey });
    } catch (err) {
      setFailure(err instanceof Error ? err.message : "erro inesperado");
    } finally {
      setBusy(null);
    }
  }

  async function rotateKey() {
    setDone(null); setFailure(null);
    const key = keyText.trim();
    if (hmacKeyProblem(key)) return;
    setBusy("key");
    try {
      const { data, fieldErrors } = await gql(RotateCityPecHmacKeyMutation, { citySlug: slug, key });
      const fail = payloadFailure(data?.rotateCityPecHmacKey, fieldErrors);
      if (fail) { setFailure(fail); return; }
      setKeyText("");
      setDone("Chave gravada. Configure a mesma chave no PEC da cidade, se ainda não estiver lá.");
      void queryClient.invalidateQueries({ queryKey });
    } catch (err) {
      setFailure(err instanceof Error ? err.message : "erro inesperado");
    } finally {
      setBusy(null);
    }
  }

  const status = config ? hmacKeyStatus(config) : null;

  return (
    <section aria-label={TITLE}>
      <Panel title={TITLE} actions={<Button onClick={() => void query.refetch()} busy={query.isFetching}>atualizar</Button>}>
        <p style={note}>{PEC_ORIGIN_HELP}</p>
        {query.isPending && <p>carregando…</p>}
        {query.isError && <ErrorState message={requestErrorText(query.error)} />}
        {fieldError && <ErrorState message={`${fieldError.code} — ${fieldError.message}`} />}
        {config && status && (
          <>
            <div role="group" aria-label="configuração do PEC" style={column}>
              <div style={row}>
                <strong style={{ fontSize: 12.5 }}>Origem</strong>
                <span>{pecOriginText(config)}</span>
              </div>
              <div style={row}>
                <strong style={{ fontSize: 12.5 }}>Chave HMAC</strong>
                <Tag tone={status.tone}>{status.label}</Tag>
                <span style={note}>{hmacKeyUpdatedText(config)}</span>
              </div>
            </div>

            {done && <p role="status" style={{ margin: 0, fontSize: 12.5, color: "var(--ok)" }}>{done}</p>}
            {failure && <ErrorState message={failure} />}

            <div style={column}>
              <Field label="Endereço de origem do PEC" value={originText} autoComplete="off"
                onChange={(v) => { setOriginTouched(true); setOriginText(v); }}
                helpText="Ex.: https://pec.curitiba.pr.gov.br" />
              <div><Button onClick={() => void saveOrigin()} busy={busy === "origin"} disabled={busy !== null}>salvar endereço</Button></div>
            </div>

            <div style={column}>
              <Field label="Chave de assinatura do PEC" type="password" name="pec-hmac-key" autoComplete="off" value={keyText}
                onChange={setKeyText} helpText="Cole a chave que o PEC gerou na configuração do sistema externo. Ela não aparece de novo." />
              {keyProblem && <p style={{ margin: 0, fontSize: 11.5, color: "var(--down)" }}>{keyProblem}</p>}
              <p role="note" aria-label="aviso da troca da chave" style={warnNote}>{PEC_KEY_ROTATE_WARNING}</p>
              <div>
                <ConfirmButton
                  label={configured ? "trocar a chave" : "gravar a chave"}
                  confirmLabel={configured ? "confirmar: trocar a chave HMAC" : "confirmar: gravar a chave HMAC"}
                  busy={busy === "key"}
                  disabled={busy !== null || keyText.trim() === "" || keyProblem !== null}
                  onConfirm={() => void rotateKey()}
                />
              </div>
            </div>
          </>
        )}
      </Panel>
    </section>
  );
}

const note: CSSProperties = { margin: 0, fontSize: 11.5, color: "var(--ink3)" };
const warnNote: CSSProperties = { margin: 0, fontSize: 12, fontWeight: 600, color: "var(--warn)" };
const row: CSSProperties = { display: "flex", gap: 8, alignItems: "center", flexWrap: "wrap" };
const column: CSSProperties = { display: "flex", flexDirection: "column", gap: 6 };
```

O `Field` existente não repassa `autoComplete` se não for passado; ele já aceita `type`, `name` e `autoComplete` (veja `src/components/Field.tsx`). Nenhum componente comum muda.

Em `src/screens/FeaturesTab.tsx`:

(a) nos imports, depois de `import { CONTROLLED_KEY, SNCR_MOCK_NOTICE, sncrModeNotice } from "../lib/sncr";`:

```tsx
import { PecConfigPanel } from "./PecConfigPanel";
import { PEC_EXTERNAL_KEY } from "../lib/pec";
```

(b) depois da linha `{city.features.some((f) => f.key === CONTROLLED_KEY) && <SncrPlatformPanel sncrMock={sncrSimulated} />}`:

```tsx

          {/* Módulo 20: origem e chave HMAC da página do PEC (por cidade), só
              onde se decide ligar pec_external_view. Consulta própria (PecConfigPanel). */}
          {city.features.some((f) => f.key === PEC_EXTERNAL_KEY) && <PecConfigPanel slug={slug} />}
```

No `README.md`, logo depois do parágrafo **Receita de controlado e SNCR (módulo 19d).** (o que termina em "só o quadro mostra o aviso da ordem de deploy e a aba segue."), acrescente:

```markdown

**Linha do tempo e página do PEC (módulo 20).** Os interruptores
`patient_timeline` ("Linha do tempo do paciente", sem pré-requisito) e
`pec_external_view` ("Página para os sistemas externos do PEC", exige
`patient_timeline` ligado e o modo `integrated`) aparecem na mesma lista e
ligam como os outros. Quando o catálogo da cidade traz `pec_external_view`, a
aba mostra o quadro **Página do PEC (sistemas externos)**: a origem do PEC
(`https://pec.cidade.gov.br`, sem caminho — vai no `frame-ancestors` da página;
não é o endereço do LEDI, que é do operador no console admin) e a chave de
assinatura HMAC que o próprio PEC gera. A chave só se grava ou troca, com
confirmação de dois cliques; ela nunca volta do api nem aparece aqui, e o campo
fica vazio depois de gravar. Trocar a chave faz a página recusar as aberturas
até o PEC ter a chave nova. Ordem de deploy: api → dashboard e wpda →
maintenance; contra um api sem o módulo 20 o quadro nem aparece, e se o
catálogo trouxer a chave mas o schema não tiver `cityPecConfig`, só o quadro
mostra o aviso da ordem de deploy.
```

Depois, o codegen:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/maintenance/.claude/mod20 && npm run codegen && /opt/homebrew/bin/git status --short src/gql
grep -n "CityPecConfig\|SetCityPecOrigin\|RotateCityPecHmacKey" src/gql/gql.ts | head -6
```

Expected: `src/gql/gql.ts` e `src/gql/graphql.ts` modificados; os três documentos registrados em `gql.ts`.

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/maintenance/.claude/mod20 && npx vitest run src/screens/PecConfigPanel.test.tsx src/screens/FeaturesTab.test.tsx src/lib/pec.test.ts src/lib/features.test.ts && npx tsc --noEmit && npm run codegen:check`
Expected: PASS (os testes antigos da `FeaturesTab`, inclusive o "Modo e IBGE não têm controle de edição aqui", continuam verdes: nenhum catálogo deles traz `pec_external_view`), `tsc` sem erro e `codegen:check` verde.

- [ ] **Step 5: Commit**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/maintenance/.claude/mod20
/opt/homebrew/bin/git add schema.graphql src/gql/gql.ts src/gql/graphql.ts src/screens/PecConfigPanel.tsx \
  src/screens/PecConfigPanel.test.tsx src/screens/FeaturesTab.tsx src/screens/FeaturesTab.test.tsx README.md
/opt/homebrew/bin/git status --short
/opt/homebrew/bin/git commit -m "feat: configure the PEC origin and HMAC key per city next to the PEC external view switch

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

Expected: `status --short` antes do commit sem nada além dos oito arquivos.

---

### Task 3: Suíte, codegen:check, build, revisão e prova no navegador

**Files:** nenhum novo.

**Interfaces:**
- Consumes: as Tasks 1–2 na branch `feat/mod-20-timeline`; o api do módulo 20 rodando na 3040.
- Produces: branch pronta para merge (depois do merge do api, do dashboard e do wpda do módulo 20).

- [ ] **Step 1: Suíte, tipos, codegen e build**

Run: `cd apps/maintenance/.claude/mod20 && npx vitest run && npx tsc --noEmit && npm run codegen:check && npm run build`
Expected: tudo verde: a base anotada na Task 0 mais **2 arquivos de teste novos** (`pec.test.ts`, `PecConfigPanel.test.tsx`). Um número que dobra é artefato de build descoberto pelo vitest: pare e reporte.

```bash
cd apps/maintenance/.claude/mod20 && /opt/homebrew/bin/git status --short && /opt/homebrew/bin/git log --stat origin/main..HEAD | grep -c node_modules
/opt/homebrew/bin/git diff origin/main --stat -- src/components
```

Expected: `status` vazio, contagem `0` e nenhum arquivo de `src/components` mudado.

- [ ] **Step 2: A chave não vaza**

```bash
cd apps/maintenance/.claude/mod20
grep -n "console\.\|localStorage\|sessionStorage\|useMutation" src/screens/PecConfigPanel.tsx src/lib/pec.ts ; echo "exit $?"
grep -n -E "hmacKey[^CU]|secret" schema.graphql ; echo "exit $?"
```

Expected: os dois `exit 1` — nenhum `console`, armazenamento do navegador ou `useMutation` no quadro; o schema não tem campo que devolva a chave.

- [ ] **Step 3: Revisão**

Um subagente revisor lê `origin/main..HEAD` do maintenance contra a spec (§3, §7, §8, §9, §12 F-20.1), o ADR 0035 e o contrato do módulo 20 §6–§7. Pontos de atenção:
- o liga/desliga de `patient_timeline` e `pec_external_view` é o genérico; os pré-requisitos aparecem em português;
- `CityPecConfig` é consulta própria; contra api antigo só o quadro falha; o quadro só monta com `pec_external_view` no catálogo;
- a chave só viaja no corpo de `RotateCityPecHmacKey`, nunca na leitura, na URL, no cache (query ou mutation), no DOM ou em `console`; o campo é de senha e fica vazio depois de gravar;
- a origem vai normalizada (sem barra final), e caminho/query/fragmento/credencial são recusados antes de mandar;
- trocar a chave pede dois cliques e avisa da recusa até o PEC ter a chave nova;
- `schema.graphql` e `src/gql/*` iguais ao que o api em execução produz; nenhum `git add -A` no histórico.

- [ ] **Step 4: Prova no navegador (com o usuário)**

Depende do api do módulo 20 na porta **3040** (plano do api do módulo 20), com Curitiba em `record_mode = integrated` na semente de dev. O proxy do Vite troca o Host para `maintenance-api.localhost` e injeta o Origin `http://maintenance.localhost:5177`, então o Vite do worktree sobe **na 5177**, num container da rede do compose, depois de parar o serviço `maintenance`:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose stop maintenance
docker compose run -d --rm --no-deps --name maintenance-mod20 -p 5177:5177 \
  -w /app/.claude/mod20 -e VITE_API_PROXY_TARGET=http://api:3040 -e VITE_MAINTENANCE_ENV=development \
  maintenance npx vite --port 5177 --host 0.0.0.0
docker logs -f maintenance-mod20   # espere o "ready"; Ctrl+C sai do log
```

(Se o plano do api subir o servidor do módulo 20 num container próprio, use o nome dele no lugar de `api`.)

Abra `http://maintenance.localhost:5177`. O usuário faz o login do mantenedor; não digite senha nem TOTP. Confira com screenshot:
- Cidades → Curitiba → **Funcionalidades**: `patient_timeline` e `pec_external_view` com os nomes em português; com a linha do tempo desligada, ligar `pec_external_view` → "ligada, falta pré-requisito" / "linha do tempo (patient_timeline) desligada"; ligar `patient_timeline` → "ligada e utilizável" e, depois do refetch, `pec_external_view` utilizável (ou com `pec_origin_missing`/`pec_hmac_key_missing`, se a Divergência M3 for aceita);
- o quadro **Página do PEC (sistemas externos)**: "não preenchido" e "chave não gravada"; colar `https://pec.curitiba.pr.gov.br/esus` → recusa na tela; `http://localhost:8099/` → "Endereço de origem do PEC salvo." e a origem sem a barra; colar uma chave de teste gerada na hora (`openssl rand -hex 32` no terminal do usuário) → "gravar a chave" → "confirmar: gravar a chave HMAC" → "Chave gravada…", o campo vazio, "chave configurada" e "trocada em …"; o DevTools (aba Rede) mostra a chave só no corpo do `RotateCityPecHmacKey`;
- a tela **Auditoria** mostra `city.pec_config_changed` com o campo (`pec_origin`, `pec_hmac_key`), nunca a chave;
- com a chave gravada, a página de teste do PEC simulado (plano do api do módulo 20) abre o iframe; trocar a chave aqui e abrir de novo com a chave antiga → a página recusa com o código genérico.

Depois: `docker rm -f maintenance-mod20 && docker compose start maintenance`. Deixe os interruptores e a configuração do PEC da cidade de dev como o plano do api espera para a prova dele (ou como estavam antes).

- [ ] **Step 5:** **Pare.** O merge do maintenance só vem depois do merge do api, do dashboard e do wpda do módulo 20, e só com autorização explícita do usuário (merge, push e cards, uma etapa de cada vez). Antes do push, confira `origin/main..main` no maintenance e publique só os commits desta entrega. Comandos, quando autorizados:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/maintenance
/opt/homebrew/bin/git fetch origin
/opt/homebrew/bin/git status -sb
/opt/homebrew/bin/git merge --ff-only feat/mod-20-timeline
/opt/homebrew/bin/git log --oneline origin/main..main
/opt/homebrew/bin/git push origin main
/opt/homebrew/bin/git worktree remove .claude/mod20
/opt/homebrew/bin/git branch -d feat/mod-20-timeline
```

---

## Divergências propostas ao contrato

1. **M1 — Tipo, nulidade e lugar do `cityPecConfig`.** O contrato (§6) dá os campos, não o tipo nem o comportamento em falha. Proposta, na raiz `Query` (por cidade, como `city(slug)`), anulável para a recusa do envelope (`CITY_UNREACHABLE`, `CITY_OUT_OF_SCOPE`) vir em `errors` com o `path` `["cityPecConfig"]`:
   ```graphql
   type CityPecConfig { pecOrigin: String, hmacKeyConfigured: Boolean!, hmacKeyUpdatedAt: ISO8601DateTime }
   cityPecConfig(citySlug: String!): CityPecConfig
   setCityPecOrigin(citySlug: String!, pecOrigin: String!): SetCityPecOriginPayload      # { ok, errors: [UserError!]! }
   rotateCityPecHmacKey(citySlug: String!, key: String!): RotateCityPecHmacKeyPayload    # { ok, errors: [UserError!]! }
   ```
   Nenhum campo do schema devolve a chave. Códigos de `UserError.message`: `invalid_pec_origin`, `invalid_key`, `unknown_city`, `city_unreachable`.
2. **M2 — Formato aceito (o mesmo do plano do api, "Valores fixados" 7).** `pecOrigin`: origem `scheme://host[:porta]`, sem caminho, query, fragmento ou credencial; o api grava a origem sem barra final e recusa o resto com `invalid_pec_origin`; `https` sempre e `http` só fora de produção (a tela aceita os dois e mostra a recusa do api). `key`: o api apara as pontas e aceita de 16 a 512 caracteres ASCII visíveis (`/\A[\x21-\x7E]{16,512}\z/`; o PEC gera a chave e a documentação oficial não fixa o formato), senão `invalid_key`. A tela espelha as duas regras para não mandar o que seria recusado.
3. **M3 — Pré-requisitos de `pec_external_view`.** O contrato lista só `patient_timeline_disabled` e `record_mode_not_integrated`, e o plano do api (Desvio 14) mantém isso: sem origem ou sem chave, a página responde `pec_disabled`. Proposta: o catálogo acrescenta `pec_origin_missing` e `pec_hmac_key_missing`, para "ligada e utilizável" significar que o PEC de fato consegue abrir a página. As frases já estão na Task 1; sem a divergência, nada quebra — o quadro do PEC continua dizendo "chave não gravada" e "não preenchido".
4. **M4 — Step-up na troca da chave.** O contrato não pede o TOTP em `rotateCityPecHmacKey`; o maintenance já o pede para ações de efeito parecido (`createMaintenanceToken`, `inviteMaintainer`). Proposta para o api decidir: acrescentar `code: String!` (TOTP do momento) a `rotateCityPecHmacKey`. Este plano segue o contrato atual (confirmação de dois cliques); se a divergência for aceita, a Task 2 ganha o campo "Código do autenticador" com `STEP_UP_CODE_HELP_TEXT`/`StepUpCodeError`, como em `Tokens.tsx`.
5. **M5 — A URL a cadastrar no PEC.** A documentação oficial manda o integrador entregar ao administrador da instalação a URL com as variáveis (`https://<host da cidade>/pec/timeline?documento-profissional=[documentoProfissional]&documento-cidadao=[documentoCidadao]&cnes=[CNES]`, nomes já adotados pelo plano do api, Divergência D1 dele). Proposta para uma entrega futura: `CityPecConfig.externalUrlTemplate: String!`, para o quadro mostrar ao mantenedor exatamente o que copiar para o PEC. Este plano não depende disso.

## Self-review

- **Cobertura (spec §3, §7, §8 "maintenance", §9, §12 F-20.1; contrato §6–§7):** interruptores `patient_timeline` e `pec_external_view` pelo mecanismo genérico, com os pré-requisitos em português — Tasks 1 e 2 (dois cliques sem mudar o fluxo); `pec_origin` editável — Tasks 1 e 2; chave HMAC só gravar/trocar, nunca exibida, com confirmação — Task 2 (e Task 3, Step 2); ordem de deploy — README, consulta própria e testes da Task 2; prova no navegador — Task 3. O evento `city.pec_config_changed` é do api; a prova confere na Auditoria.
- **Placeholders:** nenhum; todo passo de código tem o código. As únicas variáveis são as do ambiente do api do módulo 20 (nome do container, página de teste do PEC simulado), que o plano do api define.
- **Tipos:** `PecConfigView` aceita o tipo gerado da M1 (`pecOrigin?: Maybe<string>`, `hmacKeyUpdatedAt?: Maybe<string>`); `PecConfigPanel({ slug })` é o que a `FeaturesTab` monta; os nomes das operações (`CityPecConfig`, `SetCityPecOrigin`, `RotateCityPecHmacKey`) são os mesmos no componente e nos testes; `PEC_EXTERNAL_KEY` é a chave do contrato.
- **Review Focus:** 1 — Task 2 ("gravar a chave: dois cliques…", "a consulta nunca pede a chave"); 2 — Task 1 ("normalizePecOrigin: tabela de casos") e Task 2 ("endereço com caminho…"); 3 — Task 2 ("um clique não troca…", "chave curta…"); 4 — Task 1 ("pré-requisitos do módulo 20 têm frase") e Task 2 ("pec_external_view pelo mecanismo genérico…"); 5 — Task 2 ("api sem o módulo 20…" no quadro e na aba, "cidade inalcançável…").
