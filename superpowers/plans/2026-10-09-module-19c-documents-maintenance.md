# Módulo 19 (19c) — Documentos clínicos (maintenance) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** nenhum além do que já está em `origin/main` (maintenance `0f0a779`, 19b fechado). Para o schema (Task 2) e a prova (Task 3), o **api do 19c** precisa estar rodando na branch dele (porta **3038**) com o interruptor `clinical_documents` no catálogo e os campos do catálogo de medicamentos no `Maintenance::Schema`; e ele é **mergeado antes** deste (contrato §11: api → dashboard → maintenance). A Task 0 confere.

**Goal:** Lado maintenance de F-19.16: o interruptor `clinical_documents` aparece e é ligado/desligado na aba **Funcionalidades** pelo mecanismo genérico do módulo 16 (só se confirma e testa, com o rótulo da chave e do pré-requisito `clinical_record_disabled`), e uma tela nova de plataforma, **Medicamentos**, mostra a versão atual do catálogo (CATMAT, classe 6505), importa o CATMAT e as listas da Anvisa (antimicrobianos da RDC 471 e controlados da Portaria 344) e lista os itens que o analisador não entendeu e ficaram ocultos para revisão.

**Architecture:** Nada muda no liga/desliga: `FeaturesTab` já lista o catálogo que o api devolve e chama `setCityFeature`; entra só o rótulo da chave nova em `src/lib/features.ts`. As regras de exibição do catálogo ficam em `src/lib/medications.ts` (puras: resumo da versão, estado de análise, aviso de lista truncada, frases do resultado e dos erros da importação). A tela `src/screens/MedicationCatalog.tsx` tem **consulta GraphQL própria** (`query MedicationCatalog`) e duas mutations (`ImportMedicationCatalog`, `ImportAnvisaLists`), confirmadas em dois cliques (`ConfirmButton`), uma importação por vez; contra um api sem o 19c, só ela mostra o aviso da ordem de deploy. Fica na casca (`Shell`) como tela de topo, ao lado de Cidades, porque o catálogo é da plataforma (vale para todas as cidades do ambiente), não de uma cidade.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, GraphQL Codegen (client preset, `src/gql/` gerado e commitado), Vitest 2 + Testing Library 16 + user-event 14 (jsdom, sem jest-dom, `vi.stubGlobal("fetch")`).

**Spec:** `docs/superpowers/specs/2026-10-09-module-19c-clinical-documents-design.md` (§3 "Interruptor e bases" e "Catálogo", §7 "Maintenance", §11 F-19.16, §12 "Curadoria real do CATMAT"), `docs/adr/0033.md` e o contrato `docs/superpowers/plans/2026-10-09-module-19c-documents-contracts.md` §8 (a forma exata dos tipos GraphQL está na seção final "Divergências propostas ao contrato", M1).

## Global Constraints

- Interruptor (contrato §8; spec §3): `clinical_documents` pelo mecanismo genérico (`City.features`, `setCityFeature`), `requires: ["feature:clinical_record"]`; faltando, `missing: ["clinical_record_disabled"]`. Ligar não exige o pré-requisito: fica "ligada, falta pré-requisito" (três estados, como no módulo 16). Nenhuma mudança em `FeaturesTab` no liga/desliga.
- Leitura e escrita novas (contrato §8, tipos da Divergência M1), de plataforma, na raiz:
  ```graphql
  medicationCatalog: MedicationCatalog!            # Query
  # MedicationCatalog { currentRelease: MedicationCatalogRelease, reviewItems(first: Int = 50): [MedicationCatalogItem!]! }
  # MedicationCatalogRelease { id: ID!, importedAt: ISO8601DateTime!, itemsCount: Int!, hiddenCount: Int! }
  # MedicationCatalogItem { catmatCode: Int!, sourceDescription: String!, parseStatus: String! }
  importMedicationCatalog: ImportMedicationCatalogPayload   # Mutation, sem argumentos
  # { ok: Boolean!, releaseId: ID, added: Int, changed: Int, removed: Int, errors: [UserError!]! }
  importAnvisaLists: ImportAnvisaListsPayload               # Mutation, sem argumentos
  # { ok: Boolean!, antimicrobials: Int, controlled: Int, errors: [UserError!]! }
  ```
- Regras da tela (spec §3): a importação traz o CATMAT classe 6505 da API aberta do Compras.gov e diz quantos itens **entraram, mudaram e saíram**; item que o analisador não entende fica **oculto** na busca da receita e vai para a lista de revisão (`parseStatus` `review`); as listas da Anvisa marcam antimicrobiano e controlado no catálogo. A lista de revisão pede os primeiros **50** (`REVIEW_PAGE`) e diz quando há mais ocultos do que os mostrados. Uma importação por vez: enquanto uma roda, os dois botões ficam travados.
- Nada de dado de paciente ou de cidade nesta tela: o catálogo é da plataforma. A descrição de origem do CATMAT é dado público e aparece como veio.
- Valor desconhecido (api mais novo) aparece cru — nunca `undefined`/`null` na tela; nulo vira "—".
- **Ordem de deploy: api → dashboard → maintenance** (contrato §11). Contra um api sem o 19c, só a tela Medicamentos mostra o aviso; Cidades, Funcionalidades e o resto seguem.
- Interface tem ciclo próprio: componentes e estilo existentes (`Panel`, `DataTable`, `Tag`, `Button`, `ConfirmButton`, `ErrorState`, `EmptyState`), sem redesign.
- Nunca escreva à mão em arquivo gerado (`schema.graphql`, `src/gql/*`): eles saem do api em execução e do codegen.
- Nunca `git add -A` nem `git add .`: o worktree tem o symlink de `node_modules`. Adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Merge, push e tag só com autorização explícita do usuário.

## Ambiente de execução

- Worktree `apps/maintenance/.claude/mod19c`, branch `feat/mod-19c-documents` a partir de `origin/main` (Task 0). O `.gitignore` do maintenance ignora `node_modules` (também o symlink), mas **não** ignora `.claude/`: o checkout principal lista `.claude/` como não rastreado. É esperado; não adicione.
- Todos os caminhos de arquivo das tasks são relativos ao worktree; os comandos partem da raiz do monorepo (`/Users/eduardovrocha/Development/ioit.solutions/rota-saude`).
- Testes: `cd apps/maintenance/.claude/mod19c && npx vitest run <arquivos>` (o `vitest.config.ts` exclui `e2e/**`).
- Tipos e codegen: `cd apps/maintenance/.claude/mod19c && npx tsc --noEmit` e `npm run codegen`.
- Ambiente de teste: `environment: "jsdom"`, `globals: false` (todo teste de componente chama `afterEach(cleanup)`), sem jest-dom (use `not.toBeNull()`, `toBeNull()`, `.textContent`, `(el as HTMLButtonElement).disabled`). O `vitest.config.ts` não fixa `TZ`: datas saem de `fmtWhen` (`src/lib/analytics.ts`), que já formata em `America/Sao_Paulo` ("09/10/2026, 09:00").
- **`npm run schema:pull` não serve no worktree** (o script sobe três níveis a partir de `scripts/` e escreveria no checkout principal). A Task 2 extrai o SDL pelo `docker compose exec` direto, com destino no worktree.
- Padrão encontrado no código (o brief pedia o "padrão das terminologias do 16 no maintenance"): **não há tela de terminologia no maintenance**. As tabelas do módulo 16 (CID-10, CIAP-2, SIGTAP) entram pelo rake `terminology:import[kind,version,path]` do api (`lib/tasks/terminology.rake`, `Terminology::Import`), e o maintenance só mostra a falta delas como pré-requisito cru (`terminology_missing:<kind>`, `src/lib/features.test.ts`). A importação acionada pela tela é nova neste módulo; o padrão de tela que ela segue é o do `SignaturePlatformPanel` do 19b (consulta própria, de plataforma, aviso de ordem de deploy) e o do liga/desliga do 16 (`ConfirmButton` de dois cliques, frase com o que o api devolveu).
- Login de dev (Task 3): `http://maintenance.localhost:5177`, nunca `localhost:5177`. A conta é o `Maintainer` `dev@local` semeado pelo `db:seed`; o usuário faz o login.

## Review Focus

1. **A importação demora e o mantenedor clica de novo.** A API do Compras.gov pagina os 6.778 itens; a mutation pode levar minutos. Um segundo clique (no mesmo botão ou no das listas da Anvisa) dispararia outra importação por cima. Enquanto uma roda, os dois botões ficam travados e a tela diz que está importando e para não fechar a aba. Teste: Task 2, "importar o CATMAT pede confirmação, trava os dois botões enquanto roda e diz o resultado".
2. **Compras.gov fora do ar ou importação recusada.** O api devolve `ok: false` com o motivo; a tela diz que **nada mudou** (a versão atual continua a mesma) em português, não o código cru; um código desconhecido aparece cru. Testes: Task 1, "erros da importação: conhecidos traduzidos, desconhecido cru"; Task 2, "Compras.gov fora do ar: diz que nada mudou e a versão continua a mesma".
3. **Itens que saíram do CATMAT.** "Saíram 12" assusta quem lê como "sumiram das receitas". A frase diz que eles deixam de aparecer na busca da receita e que receitas já emitidas não mudam. Teste: Task 1, "resultado do CATMAT: entraram, mudaram, saíram; com saída, explica".
4. **Muitos itens ocultos e só 50 na tela.** Com 312 ocultos, a lista mostra 50; sem aviso, o mantenedor acharia que são só esses. A tela diz "mostrando 50 de 312 itens ocultos". Testes: Task 1, "aviso de lista truncada"; Task 2, "mais ocultos do que a página: avisa quantos faltam".
5. **Maintenance novo contra api sem o 19c, e catálogo nunca importado.** A validação recusa `query MedicationCatalog` inteira (`undefinedField`): só a tela Medicamentos explica a ordem de deploy. Sem nenhuma importação (`currentRelease: null`), a tela diz que a receita só tem texto livre até a primeira, sem `null` na tela. Testes: Task 2, "api sem o 19c: explica a ordem de deploy" e "nenhuma importação ainda e nada para revisar".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | conferência do main e do api do 19c, worktree e base | 0 |
| `src/lib/features.ts`, `src/lib/features.test.ts`, `src/lib/medications.ts`, `src/lib/medications.test.ts` | rótulo de `clinical_documents`; regras de exibição do catálogo e da importação | 1 |
| `schema.graphql`, `src/gql/*` (gerados) | tipos novos do api | 2 |
| `src/screens/MedicationCatalog.tsx`, `src/screens/MedicationCatalog.test.tsx`, `src/screens/Shell.tsx`, `src/screens/Shell.test.tsx`, `src/screens/FeaturesTab.test.tsx`, `README.md` | tela Medicamentos (versão, importações, revisão); item novo na navegação; o interruptor pelo mecanismo genérico | 2 |
| — | suíte, codegen:check, build, revisão e prova no navegador | 3 |

**Estratégia de teste:** regras de exibição com tabela de casos (Task 1); a tela com `fetch` falso que responde por nome de operação (`query MedicationCatalog`, `mutation ImportMedicationCatalog`, `mutation ImportAnvisaLists`), cobrindo o caminho feliz, a importação em curso (promessa adiada), a recusa, a lista truncada, o catálogo vazio e o api sem o 19c (Task 2); o interruptor `clinical_documents` testado pelo fluxo existente de dois cliques da `FeaturesTab`, sem mudar o código do liga/desliga; a casca com o item novo. A prova final é no navegador, contra o api do 19c (Task 3).

---

### Task 0: Conferência do main e do api do 19c, worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do maintenance (`0f0a779`); a branch do api do 19c rodando na 3038.
- Produces: worktree `apps/maintenance/.claude/mod19c` na branch `feat/mod-19c-documents`; a base anotada.

- [ ] **Step 1: Confira o main do maintenance e o api do 19c no ar**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude
/opt/homebrew/bin/git -C apps/maintenance fetch origin
/opt/homebrew/bin/git -C apps/maintenance log --oneline -1 origin/main
/opt/homebrew/bin/git -C apps/maintenance show origin/main:src/screens/Shell.tsx | grep -n 'export type Screen'
/opt/homebrew/bin/git -C apps/maintenance show origin/main:src/lib/features.ts | grep -n 'clinical_record_disabled\|signature_psc_mock'
grep -rn "clinical_documents" apps/api/.claude/mod19c/app/services/platform 2>/dev/null | head -3
docker compose exec -T -w /rails/.claude/mod19c api bin/rails runner \
  'puts Maintenance::Schema.to_definition' 2>/dev/null | grep -n -E "medicationCatalog|importMedicationCatalog|importAnvisaLists"
```

Expected: `origin/main` em `0f0a779 fix: name the simulated PSC notice outside production and correct the deploy-order note` (ou mais novo; se `src/screens/Shell.tsx`, `src/lib/features.ts` ou `src/screens/FeaturesTab.test.tsx` mudaram, confira os trechos que as Tasks 1 e 2 trocam); `export type Screen = "cities" | "maintainers" | "tokens" | "audit";`; as duas linhas de `features.ts`; ao menos uma linha com `clinical_documents` no catálogo do api do 19c; e as três linhas do schema. Se o api do 19c ainda não tiver os campos, as Tasks 0 e 1 seguem; **pare antes da Task 2** e reporte.

- [ ] **Step 2: Crie o worktree e ligue o `node_modules`**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/maintenance
/opt/homebrew/bin/git worktree add .claude/mod19c -b feat/mod-19c-documents origin/main
ln -s ../../node_modules .claude/mod19c/node_modules
cd .claude/mod19c && npx vitest run 2>&1 | tail -3 && npx tsc --noEmit && echo tsc-ok
```

Expected: suíte verde e `tsc-ok`. **Anote** a base (arquivos e testes): a Task 3 espera a base mais **2 arquivos de teste novos** (`medications.test.ts` e `MedicationCatalog.test.tsx`).

---

### Task 1: Rótulo do interruptor e regras de exibição do catálogo

**Files:**
- Modify: `src/lib/features.ts`, `src/lib/features.test.ts`
- Create: `src/lib/medications.ts`
- Test: `src/lib/medications.test.ts`

**Interfaces:**
- Consumes: `Tone` de `src/theme/tokens.ts`; `fmtWhen` de `src/lib/analytics.ts`; `featureLabel`, `missingText` (existentes).
- Produces:
  - `featureLabel("clinical_documents")` → "Documentos clínicos (atestado, declaração, receita, requisição)";
  - em `src/lib/medications.ts`:
    ```ts
    export const REVIEW_PAGE = 50;
    export type ReleaseView = { id: string; importedAt: string; itemsCount: number; hiddenCount: number };
    export type ReviewItemView = { catmatCode: number; sourceDescription: string; parseStatus: string };
    export type ImportErrorView = { path?: string | null; message: string };
    export type CatalogImportView = { ok: boolean; releaseId?: string | null; added?: number | null; changed?: number | null; removed?: number | null; errors: ImportErrorView[] };
    export type AnvisaImportView = { ok: boolean; antimicrobials?: number | null; controlled?: number | null; errors: ImportErrorView[] };
    export const IMPORT_RUNNING: string;                           // aviso enquanto uma importação roda
    export function countText(n: number | null | undefined): string; // "6.778"; nulo → "—"
    export function releaseText(r: ReleaseView | null | undefined): string;
    export function parseStatusView(status: string): { label: string; tone: Tone };
    export function reviewNote(shown: number, hidden: number): string | null;
    export function importErrorText(errors: ImportErrorView[]): string;
    export function catalogImportText(r: CatalogImportView): string;
    export function anvisaImportText(r: AnvisaImportView): string;
    ```

- [ ] **Step 1: Write the failing test**

Em `src/lib/features.test.ts`, no `describe("featureLabel", …)` existente (o que testa `signature_psc_mock`), acrescente o caso; se não houver esse `describe`, acrescente-o no fim do arquivo:

```ts
describe("featureLabel do 19c", () => {
  it("documentos clínicos têm nome em português; o pré-requisito já tem frase", () => {
    expect(featureLabel("clinical_documents")).toBe("Documentos clínicos (atestado, declaração, receita, requisição)");
    expect(missingText("clinical_record_disabled")).toBe("prontuário da atenção primária (clinical_record) desligado");
  });
});
```

(`featureLabel` e `missingText` já estão no `import` do arquivo; se `featureLabel` não estiver, acrescente-o ao `import { … } from "./features"`.)

```ts
// src/lib/medications.test.ts
import { describe, expect, it } from "vitest";
import {
  REVIEW_PAGE, anvisaImportText, catalogImportText, countText, importErrorText, parseStatusView, releaseText, reviewNote
} from "./medications";

describe("countText", () => {
  it("formata em pt-BR e nulo vira travessão", () => {
    expect(countText(6778)).toBe("6.778");
    expect(countText(0)).toBe("0");
    expect(countText(null)).toBe("—");
    expect(countText(undefined)).toBe("—");
  });
});

describe("releaseText", () => {
  it("resume a versão atual", () => {
    expect(releaseText({ id: "r1", importedAt: "2026-10-09T12:00:00Z", itemsCount: 6778, hiddenCount: 312 }))
      .toBe("importado em 09/10/2026, 09:00 · 6.778 itens · 312 ocultos para revisão");
  });

  it("sem importação ainda: diz o que isso significa", () => {
    expect(releaseText(null)).toBe("nenhuma importação ainda — até a primeira, a receita só aceita texto livre");
    expect(releaseText(undefined)).toBe("nenhuma importação ainda — até a primeira, a receita só aceita texto livre");
  });
});

describe("parseStatusView", () => {
  it("ok, para revisar e desconhecido cru", () => {
    expect(parseStatusView("ok")).toEqual({ label: "entendido", tone: "ok" });
    expect(parseStatusView("review")).toEqual({ label: "para revisar", tone: "warn" });
    expect(parseStatusView("ambiguous")).toEqual({ label: "ambiguous", tone: "neutral" });
  });
});

describe("reviewNote", () => {
  it("aviso de lista truncada", () => {
    expect(REVIEW_PAGE).toBe(50);
    expect(reviewNote(50, 312)).toBe("mostrando 50 de 312 itens ocultos");
    expect(reviewNote(12, 12)).toBeNull();
    expect(reviewNote(0, 0)).toBeNull();
  });
});

describe("importErrorText", () => {
  it("erros da importação: conhecidos traduzidos, desconhecido cru", () => {
    expect(importErrorText([ { path: "catalog", message: "source_unavailable" } ]))
      .toBe("o Compras.gov não respondeu — nada mudou; tente de novo mais tarde");
    expect(importErrorText([ { message: "incomplete_source" } ]))
      .toBe("o Compras.gov devolveu menos itens do que anunciou — nada mudou; tente de novo mais tarde");
    expect(importErrorText([ { message: "suspicious_drop" } ]))
      .toBe("o CATMAT veio com menos da metade dos itens ativos — nada mudou; confira a fonte antes de importar de novo");
    expect(importErrorText([ { message: "invalid_file" } ])).toBe("um arquivo das listas da Anvisa está inválido — nada mudou");
    expect(importErrorText([ { message: "import_failed" } ])).toBe("a importação falhou — nada mudou");
    expect(importErrorText([ { message: "import_in_progress" } ])).toBe("já há uma importação em andamento — espere ela terminar");
    expect(importErrorText([ { message: "parser_crashed" } ])).toBe("parser_crashed");
    expect(importErrorText([ { message: "import_in_progress" }, { message: "outra coisa" } ]))
      .toBe("já há uma importação em andamento — espere ela terminar; outra coisa");
    expect(importErrorText([])).toBe("não foi possível concluir — tente de novo");
  });
});

describe("catalogImportText", () => {
  it("resultado do CATMAT: entraram, mudaram, saíram; com saída, explica", () => {
    expect(catalogImportText({ ok: true, releaseId: "r2", added: 40, changed: 7, removed: 0, errors: [] }))
      .toBe("Catálogo importado: 40 entraram, 7 mudaram, 0 saíram.");
    expect(catalogImportText({ ok: true, releaseId: "r2", added: 1203, changed: 0, removed: 12, errors: [] }))
      .toBe("Catálogo importado: 1.203 entraram, 0 mudaram, 12 saíram. Os que saíram deixam de aparecer na busca da receita; receitas já emitidas não mudam.");
  });

  it("contagem nula vira travessão; recusa diz o motivo", () => {
    expect(catalogImportText({ ok: true, releaseId: null, added: null, changed: null, removed: null, errors: [] }))
      .toBe("Catálogo importado: — entraram, — mudaram, — saíram.");
    expect(catalogImportText({ ok: false, errors: [ { message: "source_unavailable" } ] }))
      .toBe("o Compras.gov não respondeu — nada mudou; tente de novo mais tarde");
  });
});

describe("anvisaImportText", () => {
  it("diz quantos foram marcados, ou o motivo da recusa", () => {
    expect(anvisaImportText({ ok: true, antimicrobials: 118, controlled: 402, errors: [] }))
      .toBe("Listas da Anvisa importadas: 118 antimicrobianos (RDC 471) e 402 controlados (Portaria 344) marcados no catálogo.");
    expect(anvisaImportText({ ok: false, errors: [ { message: "invalid_file" } ] }))
      .toBe("um arquivo das listas da Anvisa está inválido — nada mudou");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/maintenance/.claude/mod19c && npx vitest run src/lib/features.test.ts src/lib/medications.test.ts`
Expected: FAIL — `featureLabel("clinical_documents")` devolve `null` e `./medications` não existe.

- [ ] **Step 3: Write minimal implementation**

Em `src/lib/features.ts`, troque o mapa `FEATURE_LABELS`:

```ts
// Nome em português de algumas chaves do catálogo (a chave crua continua na tela).
const FEATURE_LABELS: Record<string, string> = {
  signature_psc_mock: "PSC simulado (desenvolvimento)",
  // Módulo 19c: exige clinical_record (missing: clinical_record_disabled, já traduzido acima).
  clinical_documents: "Documentos clínicos (atestado, declaração, receita, requisição)"
};
```

```ts
// src/lib/medications.ts
// Regras de exibição do catálogo de medicamentos da plataforma (módulo 19c,
// F-19.16; spec §3; contrato §8). O catálogo vem do CATMAT (classe 6505,
// Compras.gov) e as marcas de antimicrobiano e controlado das listas da
// Anvisa; quem importa, compara e decide o que fica oculto é o api. Aqui só
// se traduz o que ele devolveu. Valor desconhecido aparece cru; nulo vira "—".
import type { Tone } from "../theme/tokens";
import { fmtWhen } from "./analytics";

export const REVIEW_PAGE = 50;

export type ReleaseView = { id: string; importedAt: string; itemsCount: number; hiddenCount: number };
export type ReviewItemView = { catmatCode: number; sourceDescription: string; parseStatus: string };
export type ImportErrorView = { path?: string | null; message: string };
export type CatalogImportView = {
  ok: boolean; releaseId?: string | null; added?: number | null; changed?: number | null; removed?: number | null;
  errors: ImportErrorView[];
};
export type AnvisaImportView = { ok: boolean; antimicrobials?: number | null; controlled?: number | null; errors: ImportErrorView[] };

export const IMPORT_RUNNING = "importando… pode levar alguns minutos — não feche esta aba";

const counter = new Intl.NumberFormat("pt-BR", { maximumFractionDigits: 0 });

export function countText(n: number | null | undefined): string {
  return typeof n === "number" ? counter.format(n) : "—";
}

export function releaseText(r: ReleaseView | null | undefined): string {
  if (!r) return "nenhuma importação ainda — até a primeira, a receita só aceita texto livre";
  return `importado em ${fmtWhen(r.importedAt)} · ${countText(r.itemsCount)} itens · ${countText(r.hiddenCount)} ocultos para revisão`;
}

const PARSE_STATUS: Record<string, { label: string; tone: Tone }> = {
  ok: { label: "entendido", tone: "ok" },
  review: { label: "para revisar", tone: "warn" }
};

export function parseStatusView(status: string): { label: string; tone: Tone } {
  return PARSE_STATUS[status] ?? { label: status, tone: "neutral" };
}

export function reviewNote(shown: number, hidden: number): string | null {
  return hidden > shown ? `mostrando ${countText(shown)} de ${countText(hidden)} itens ocultos` : null;
}

// Códigos propostos ao contrato (Divergência M2); o api pode mandar outros.
const IMPORT_ERRORS: Record<string, string> = {
  source_unavailable: "o Compras.gov não respondeu — nada mudou; tente de novo mais tarde",
  incomplete_source: "o Compras.gov devolveu menos itens do que anunciou — nada mudou; tente de novo mais tarde",
  suspicious_drop: "o CATMAT veio com menos da metade dos itens ativos — nada mudou; confira a fonte antes de importar de novo",
  invalid_file: "um arquivo das listas da Anvisa está inválido — nada mudou",
  import_failed: "a importação falhou — nada mudou",
  import_in_progress: "já há uma importação em andamento — espere ela terminar"
};

export function importErrorText(errors: ImportErrorView[]): string {
  if (errors.length === 0) return "não foi possível concluir — tente de novo";
  return errors.map((e) => IMPORT_ERRORS[e.message] ?? e.message).join("; ");
}

export function catalogImportText(r: CatalogImportView): string {
  if (!r.ok) return importErrorText(r.errors);
  const base = `Catálogo importado: ${countText(r.added)} entraram, ${countText(r.changed)} mudaram, ${countText(r.removed)} saíram.`;
  return (r.removed ?? 0) > 0
    ? `${base} Os que saíram deixam de aparecer na busca da receita; receitas já emitidas não mudam.`
    : base;
}

export function anvisaImportText(r: AnvisaImportView): string {
  if (!r.ok) return importErrorText(r.errors);
  return `Listas da Anvisa importadas: ${countText(r.antimicrobials)} antimicrobianos (RDC 471) e ` +
    `${countText(r.controlled)} controlados (Portaria 344) marcados no catálogo.`;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/maintenance/.claude/mod19c && npx vitest run src/lib/features.test.ts src/lib/medications.test.ts && npx tsc --noEmit`
Expected: PASS e `tsc` sem erro.

- [ ] **Step 5: Commit**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/maintenance/.claude/mod19c
/opt/homebrew/bin/git add src/lib/features.ts src/lib/features.test.ts src/lib/medications.ts src/lib/medications.test.ts
/opt/homebrew/bin/git commit -m "feat: add display rules for the medication catalog and label the clinical documents switch

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Schema, codegen, tela Medicamentos e o interruptor na aba

**Files:**
- Create: `src/screens/MedicationCatalog.tsx`
- Modify: `src/screens/Shell.tsx`, `src/screens/Shell.test.tsx`, `src/screens/FeaturesTab.test.tsx`, `README.md`
- Test: `src/screens/MedicationCatalog.test.tsx`
- Gerados: `schema.graphql`, `src/gql/*`

**Interfaces:**
- Consumes: tudo de `src/lib/medications.ts` (Task 1); `gql` (`src/lib/api.ts`); `graphql` (`src/gql`); `GraphQLRefusal` (`src/lib/errors.ts`); `Panel`, `Button`, `ConfirmButton`, `Tag`, `DataTable`, `EmptyState`, `ErrorState`. Do api do 19c: `Query.medicationCatalog`, `Mutation.importMedicationCatalog`, `Mutation.importAnvisaLists` (contrato §8, tipos da Divergência M1); `clinical_documents` no catálogo de `City.features`.
- Produces: `export function MedicationCatalog(): JSX.Element` — dois `Panel`: "Catálogo de medicamentos — CATMAT, classe 6505 (todas as cidades do ambiente)" (versão atual, "atualizar", "importar o CATMAT", "importar as listas da Anvisa", aviso de importação em curso, resultado) e "Itens para revisar" (CATMAT, descrição de origem, situação). Operações com nome fixo `query MedicationCatalog`, `mutation ImportMedicationCatalog`, `mutation ImportAnvisaLists`; chave de cache `[ "platform", "medicationCatalog" ]`. `Screen` ganha `"medications"` e a navegação o item "Medicamentos" logo depois de "Cidades".
- **Depende do plano do api do 19c** ter os campos no `Maintenance::Schema` na branch em execução.

- [ ] **Step 1: Extraia o schema do api do 19c**

O api do 19c roda na worktree dele (`apps/api/.claude/mod19c`, com o `config/master.key` copiado, como diz o plano do api); o container monta `apps/api` em `/rails`. Da raiz do monorepo:

```bash
docker compose exec -T -w /rails/.claude/mod19c api bin/rails runner 'puts Maintenance::Schema.to_definition' \
  > apps/maintenance/.claude/mod19c/schema.graphql
# (se o api do 19c já estiver mergeado em main, tire o `-w /rails/.claude/mod19c`)

grep -n -E "  medicationCatalog: MedicationCatalog!|  importMedicationCatalog: ImportMedicationCatalogPayload|  importAnvisaLists: ImportAnvisaListsPayload" \
  apps/maintenance/.claude/mod19c/schema.graphql
for t in MedicationCatalog MedicationCatalogRelease MedicationCatalogItem ImportMedicationCatalogPayload ImportAnvisaListsPayload; do
  sed -n "/^type $t {/,/^}/p" apps/maintenance/.claude/mod19c/schema.graphql
done
```

Expected: as três linhas e os cinco tipos exatamente como a Divergência M1 (o SDL ordena os campos por nome e pode trazer descrições entre aspas triplas): `MedicationCatalog { currentRelease: MedicationCatalogRelease, reviewItems(first: Int = 50): [MedicationCatalogItem!]! }`, `MedicationCatalogRelease { hiddenCount: Int!, id: ID!, importedAt: ISO8601DateTime!, itemsCount: Int! }`, `MedicationCatalogItem { catmatCode: Int!, parseStatus: String!, sourceDescription: String! }`, `ImportMedicationCatalogPayload { added: Int, changed: Int, errors: [UserError!]!, ok: Boolean!, releaseId: ID, removed: Int }`, `ImportAnvisaListsPayload { antimicrobials: Int, controlled: Int, errors: [UserError!]!, ok: Boolean! }`.

Se faltar algum, **pare e reporte**: não escreva à mão em arquivo gerado. Se vierem com nome, tipo ou nulidade diferente da M1, pare e reporte também: um dos dois planos precisa mudar antes de seguir.

```bash
cd apps/maintenance/.claude/mod19c && /opt/homebrew/bin/git diff --stat
```

Expected: só `schema.graphql`, com os tipos do 19c (e o que o api do 19c tiver acrescentado; se aparecerem outras mudanças, reporte as linhas antes de seguir). O codegen roda no Step 3, depois dos documentos novos existirem.

- [ ] **Step 2: Write the failing test**

```tsx
// src/screens/MedicationCatalog.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, render, screen, waitFor, within } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";
import { MedicationCatalog } from "./MedicationCatalog";

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

const RELEASE = { id: "r1", importedAt: "2026-10-09T12:00:00Z", itemsCount: 6778, hiddenCount: 2 };
const REVIEW = [
  { catmatCode: 271234, sourceDescription: "PARACETAMOL, CONCENTRACAO: 200MG/ML, FORMA FARMACEUTICA:", parseStatus: "review" },
  { catmatCode: 270001, sourceDescription: "KIT DE MEDICAMENTOS DIVERSOS", parseStatus: "review" }
];

function catalogReply(release: unknown = RELEASE, review: unknown[] = REVIEW) {
  return reply(200, { data: { medicationCatalog: { currentRelease: release, reviewItems: review } } });
}

type Route = (body: { query: string; variables: Record<string, unknown> }) => Promise<Response> | Response;

function route(fetchMock: ReturnType<typeof vi.fn>, handlers: Record<string, Route>) {
  fetchMock.mockImplementation((_url: string, init: RequestInit) => {
    const body = JSON.parse(init.body as string);
    const name = Object.keys(handlers).find((op) => body.query.includes(op));
    return Promise.resolve(name ? handlers[name](body) : reply(500, {}));
  });
}

function renderScreen() {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  function wrapper({ children }: { children: ReactNode }) {
    return <QueryClientProvider client={client}>{children}</QueryClientProvider>;
  }
  return render(<MedicationCatalog />, { wrapper });
}

describe("MedicationCatalog", () => {
  let fetchMock: ReturnType<typeof vi.fn>;

  beforeEach(() => {
    fetchMock = vi.fn();
    vi.stubGlobal("fetch", fetchMock);
  });

  it("mostra a versão atual e os itens para revisar, sem undefined/null", async () => {
    route(fetchMock, { "query MedicationCatalog": () => catalogReply() });
    renderScreen();

    expect(await screen.findByText("importado em 09/10/2026, 09:00 · 6.778 itens · 2 ocultos para revisão")).not.toBeNull();
    const review = screen.getByRole("region", { name: "Itens para revisar" });
    expect(within(review).getByText("271234")).not.toBeNull();
    expect(within(review).getByText("KIT DE MEDICAMENTOS DIVERSOS")).not.toBeNull();
    expect(within(review).getAllByText("para revisar")).toHaveLength(2);
    expect(bodyOf(calls(fetchMock, "query MedicationCatalog")[0]).variables).toEqual({ first: 50 });
    expect(document.body.textContent).not.toMatch(/undefined|null/);
  });

  it("nenhuma importação ainda e nada para revisar", async () => {
    route(fetchMock, { "query MedicationCatalog": () => catalogReply(null, []) });
    renderScreen();
    expect(await screen.findByText("nenhuma importação ainda — até a primeira, a receita só aceita texto livre")).not.toBeNull();
    expect(screen.getByText("nenhum item para revisar")).not.toBeNull();
    expect(document.body.textContent).not.toMatch(/undefined|null/);
  });

  it("mais ocultos do que a página: avisa quantos faltam", async () => {
    const many = Array.from({ length: 50 }, (_, i) => ({ catmatCode: 300000 + i, sourceDescription: `ITEM ${i}`, parseStatus: "review" }));
    route(fetchMock, { "query MedicationCatalog": () => catalogReply({ ...RELEASE, hiddenCount: 312 }, many) });
    renderScreen();
    expect(await screen.findByText("mostrando 50 de 312 itens ocultos")).not.toBeNull();
  });

  it("importar o CATMAT pede confirmação, trava os dois botões enquanto roda e diz o resultado", async () => {
    const user = userEvent.setup();
    let finish: (r: Response) => void = () => {};
    route(fetchMock, {
      "query MedicationCatalog": () => catalogReply(),
      "mutation ImportMedicationCatalog": () => new Promise<Response>((resolve) => { finish = resolve; })
    });
    renderScreen();

    await user.click(await screen.findByRole("button", { name: "importar o CATMAT" }));
    expect(calls(fetchMock, "mutation ImportMedicationCatalog")).toHaveLength(0);
    await user.click(screen.getByRole("button", { name: "confirmar: importar o CATMAT agora" }));

    expect((await screen.findByRole("status")).textContent).toBe("importando… pode levar alguns minutos — não feche esta aba");
    expect((screen.getByRole("button", { name: "importar as listas da Anvisa" }) as HTMLButtonElement).disabled).toBe(true);
    expect((screen.getByRole("button", { name: "…" }) as HTMLButtonElement).disabled).toBe(true);
    expect(calls(fetchMock, "mutation ImportMedicationCatalog")).toHaveLength(1);

    finish(reply(200, { data: { importMedicationCatalog: {
      ok: true, releaseId: "r2", added: 40, changed: 7, removed: 12, errors: []
    } } }));

    expect(await screen.findByText(
      "Catálogo importado: 40 entraram, 7 mudaram, 12 saíram. Os que saíram deixam de aparecer na busca da receita; receitas já emitidas não mudam."
    )).not.toBeNull();
    await waitFor(() => expect(calls(fetchMock, "query MedicationCatalog").length).toBeGreaterThanOrEqual(2));
    expect((screen.getByRole("button", { name: "importar as listas da Anvisa" }) as HTMLButtonElement).disabled).toBe(false);
  });

  it("Compras.gov fora do ar: diz que nada mudou e a versão continua a mesma", async () => {
    const user = userEvent.setup();
    route(fetchMock, {
      "query MedicationCatalog": () => catalogReply(),
      "mutation ImportMedicationCatalog": () => reply(200, { data: { importMedicationCatalog: {
        ok: false, releaseId: null, added: null, changed: null, removed: null,
        errors: [ { path: "catalog", message: "source_unavailable" } ]
      } } })
    });
    renderScreen();
    await user.click(await screen.findByRole("button", { name: "importar o CATMAT" }));
    await user.click(screen.getByRole("button", { name: "confirmar: importar o CATMAT agora" }));
    expect((await screen.findByRole("alert")).textContent)
      .toBe("o Compras.gov não respondeu — nada mudou; tente de novo mais tarde");
    expect(screen.getByText("importado em 09/10/2026, 09:00 · 6.778 itens · 2 ocultos para revisão")).not.toBeNull();
  });

  it("importar as listas da Anvisa: diz quantos foram marcados", async () => {
    const user = userEvent.setup();
    route(fetchMock, {
      "query MedicationCatalog": () => catalogReply(),
      "mutation ImportAnvisaLists": () => reply(200, { data: { importAnvisaLists: {
        ok: true, antimicrobials: 118, controlled: 402, errors: []
      } } })
    });
    renderScreen();
    await user.click(await screen.findByRole("button", { name: "importar as listas da Anvisa" }));
    await user.click(screen.getByRole("button", { name: "confirmar: importar as listas da Anvisa agora" }));
    expect(await screen.findByText(
      "Listas da Anvisa importadas: 118 antimicrobianos (RDC 471) e 402 controlados (Portaria 344) marcados no catálogo."
    )).not.toBeNull();
  });

  it("recusa do envelope (payload nulo): mostra o código e a mensagem do api", async () => {
    const user = userEvent.setup();
    route(fetchMock, {
      "query MedicationCatalog": () => catalogReply(),
      "mutation ImportAnvisaLists": () => reply(200, {
        data: { importAnvisaLists: null },
        errors: [ { message: "sessão sem permissão de escrita", path: [ "importAnvisaLists" ], extensions: { code: "FORBIDDEN" } } ]
      })
    });
    renderScreen();
    await user.click(await screen.findByRole("button", { name: "importar as listas da Anvisa" }));
    await user.click(screen.getByRole("button", { name: "confirmar: importar as listas da Anvisa agora" }));
    expect((await screen.findByRole("alert")).textContent).toBe("FORBIDDEN — sessão sem permissão de escrita");
  });

  it("api sem o 19c: explica a ordem de deploy", async () => {
    route(fetchMock, { "query MedicationCatalog": () => reply(200, {
      errors: [ {
        message: "Field 'medicationCatalog' doesn't exist on type 'Query'",
        extensions: { code: "undefinedField", typeName: "Query", fieldName: "medicationCatalog" }
      } ]
    }) });
    renderScreen();
    expect((await screen.findByRole("alert")).textContent)
      .toMatch(/o api do módulo 19c precisa subir antes do maintenance/);
  });
});
```

Em `src/screens/FeaturesTab.test.tsx`, acrescente dentro do `describe("FeaturesTab", …)` (o arquivo já tem `reply`, `bodyOf`, `calls`, `featuresReply`, `renderTab` e o tipo `Feature`):

```tsx
  it("clinical_documents pelo mecanismo genérico: nome em português, falta o prontuário e liga com dois cliques", async () => {
    const user = userEvent.setup();
    const DOCS_OFF: Feature = {
      key: "clinical_documents", description: "Documentos clínicos da consulta", enabled: false, usable: false,
      missing: [ "clinical_record_disabled" ], changedAt: null, changedBy: null
    };
    fetchMock.mockImplementation((_url: string, init: RequestInit) => {
      const body = JSON.parse(init.body as string);
      if (body.query.includes("mutation SetCityFeature")) {
        return Promise.resolve(reply(200, { data: { setCityFeature: {
          ok: true, errors: [],
          feature: { ...DOCS_OFF, enabled: true, changedAt: "2026-10-09T12:00:00Z", changedBy: "dev@local" }
        } } }));
      }
      return Promise.resolve(featuresReply([ DOCS_OFF ]));
    });
    renderTab();

    expect(await screen.findByText("clinical_documents")).not.toBeNull();
    expect(screen.getByText("Documentos clínicos (atestado, declaração, receita, requisição)")).not.toBeNull();
    expect(screen.getByText("prontuário da atenção primária (clinical_record) desligado")).not.toBeNull();

    await user.click(screen.getByRole("button", { name: "ligar" }));
    await user.click(screen.getByRole("button", { name: "confirmar: ligar clinical_documents" }));

    expect(await screen.findByText(
      "clinical_documents ligada, mas ainda não utilizável — falta: prontuário da atenção primária (clinical_record) desligado."
    )).not.toBeNull();
    const sent = calls(fetchMock, "mutation SetCityFeature");
    expect(sent).toHaveLength(1);
    expect(bodyOf(sent[0]).variables).toEqual({ citySlug: "sp", key: "clinical_documents", enabled: true });
  });
```

Em `src/screens/Shell.test.tsx`, no primeiro teste, troque o nome e a lista das telas:

```tsx
  it("mostra o e-mail do mantenedor, a navegação com as cinco telas e 'sair' encerra a sessão", async () => {
```

```tsx
    for (const label of [ "Cidades", "Medicamentos", "Mantenedores", "Tokens", "Auditoria" ]) {
```

- [ ] **Step 3: Run test to verify it fails, then generate types**

Run: `cd apps/maintenance/.claude/mod19c && npx vitest run src/screens/MedicationCatalog.test.tsx src/screens/FeaturesTab.test.tsx src/screens/Shell.test.tsx`
Expected: FAIL — `./MedicationCatalog` não existe e a navegação não tem "Medicamentos". O teste novo da `FeaturesTab` já **passa** (o mecanismo é genérico e o rótulo entrou na Task 1); é esperado.

- [ ] **Step 4: Write minimal implementation**

```tsx
// src/screens/MedicationCatalog.tsx
// Catálogo de medicamentos da plataforma (módulo 19c, F-19.16; spec §3 e §7
// "Maintenance"; contrato §8). Vale para TODAS as cidades do ambiente: o api
// traz o CATMAT (classe 6505) da API aberta do Compras.gov, compara com a
// versão atual (entrou/mudou/saiu) e esconde da busca da receita o item que o
// analisador não entendeu; as listas da Anvisa marcam antimicrobiano (RDC 471)
// e controlado (Portaria 344). Uma importação por vez.
//
// Consulta PRÓPRIA, como o quadro de assinatura do 19b: contra um api sem o
// 19c a validação recusa o documento inteiro, e só esta tela cai.
import { useState, type CSSProperties } from "react";
import { useMutation, useQuery, useQueryClient } from "@tanstack/react-query";
import { gql } from "../lib/api";
import { graphql } from "../gql";
import { GraphQLRefusal } from "../lib/errors";
import {
  IMPORT_RUNNING, REVIEW_PAGE, anvisaImportText, catalogImportText, importErrorText, parseStatusView, releaseText, reviewNote,
  type ReviewItemView
} from "../lib/medications";
import { Panel } from "../components/Panel";
import { Button } from "../components/Button";
import { ConfirmButton } from "../components/ConfirmButton";
import { Tag } from "../components/Tag";
import { DataTable } from "../components/DataTable";
import { EmptyState } from "../components/EmptyState";
import { ErrorState } from "../components/ErrorState";

const MedicationCatalogQuery = graphql(`
  query MedicationCatalog($first: Int!) {
    medicationCatalog {
      currentRelease { id importedAt itemsCount hiddenCount }
      reviewItems(first: $first) { catmatCode sourceDescription parseStatus }
    }
  }
`);

const ImportMedicationCatalogMutation = graphql(`
  mutation ImportMedicationCatalog {
    importMedicationCatalog { ok releaseId added changed removed errors { path message } }
  }
`);

const ImportAnvisaListsMutation = graphql(`
  mutation ImportAnvisaLists {
    importAnvisaLists { ok antimicrobials controlled errors { path message } }
  }
`);

const TITLE = "Catálogo de medicamentos — CATMAT, classe 6505 (todas as cidades do ambiente)";
const QUERY_KEY = [ "platform", "medicationCatalog" ];
const OLD_API_CODES = new Set([ "undefinedField" ]);

function requestErrorText(err: unknown): string {
  if (err instanceof GraphQLRefusal && OLD_API_CODES.has(err.code)) {
    return "esta API ainda não tem o catálogo de medicamentos — o api do módulo 19c precisa subir antes do maintenance";
  }
  return err instanceof Error ? err.message : "erro inesperado";
}

type Outcome = { tone: "done" | "failed"; text: string };

export function MedicationCatalog() {
  const queryClient = useQueryClient();
  const query = useQuery({
    queryKey: QUERY_KEY,
    queryFn: () => gql(MedicationCatalogQuery, { first: REVIEW_PAGE }),
    staleTime: Infinity
  });
  const [ outcome, setOutcome ] = useState<Outcome | null>(null);

  function settle(text: string, ok: boolean) {
    setOutcome({ tone: ok ? "done" : "failed", text });
    if (ok) void queryClient.invalidateQueries({ queryKey: QUERY_KEY });
  }

  // Recusa do envelope (FORBIDDEN, …): o payload vem nulo e o motivo em `errors`.
  function refused(fieldErrors: GraphQLRefusal[]): string {
    const refusal = fieldErrors[0];
    return refusal ? `${refusal.code} — ${refusal.message}` : importErrorText([]);
  }

  const catalogImport = useMutation({
    mutationFn: () => gql(ImportMedicationCatalogMutation),
    onMutate: () => setOutcome(null),
    onSuccess: ({ data, fieldErrors }) => {
      const payload = data?.importMedicationCatalog ?? null;
      if (!payload) { settle(refused(fieldErrors), false); return; }
      settle(catalogImportText(payload), payload.ok);
    },
    onError: (err) => settle(requestErrorText(err), false)
  });

  const anvisaImport = useMutation({
    mutationFn: () => gql(ImportAnvisaListsMutation),
    onMutate: () => setOutcome(null),
    onSuccess: ({ data, fieldErrors }) => {
      const payload = data?.importAnvisaLists ?? null;
      if (!payload) { settle(refused(fieldErrors), false); return; }
      settle(anvisaImportText(payload), payload.ok);
    },
    onError: (err) => settle(requestErrorText(err), false)
  });

  const running = catalogImport.isPending || anvisaImport.isPending;
  const catalog = query.data?.data?.medicationCatalog ?? null;
  const fieldError = query.data?.fieldErrors[0];
  const review: ReviewItemView[] = catalog?.reviewItems ?? [];
  const note = catalog?.currentRelease ? reviewNote(review.length, catalog.currentRelease.hiddenCount) : null;

  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 12 }}>
      <section aria-label={TITLE}>
        <Panel title={TITLE} actions={<Button onClick={() => void query.refetch()} busy={query.isFetching}>atualizar</Button>}>
          <p style={muted}>
            A importação traz o CATMAT da API aberta do Compras.gov e diz o que entrou, mudou e saiu. Item que o analisador não
            entende fica oculto na busca da receita e aparece em "Itens para revisar". As listas da Anvisa marcam antimicrobianos
            (receita em papel, 2 vias) e controlados (bloqueados até o 19d).
          </p>
          {query.isPending && <p>carregando…</p>}
          {query.isError && <ErrorState message={requestErrorText(query.error)} />}
          {fieldError && <ErrorState message={`${fieldError.code} — ${fieldError.message}`} />}
          {catalog && <p style={{ margin: 0, fontSize: 12.5 }}>{releaseText(catalog.currentRelease)}</p>}
          {catalog && (
            <div style={row}>
              <ConfirmButton
                label="importar o CATMAT"
                confirmLabel="confirmar: importar o CATMAT agora"
                busy={catalogImport.isPending}
                disabled={running}
                onConfirm={() => catalogImport.mutate()}
              />
              <ConfirmButton
                label="importar as listas da Anvisa"
                confirmLabel="confirmar: importar as listas da Anvisa agora"
                busy={anvisaImport.isPending}
                disabled={running}
                onConfirm={() => anvisaImport.mutate()}
              />
            </div>
          )}
          {running && <p role="status" style={{ margin: 0, fontSize: 12.5, fontWeight: 600 }}>{IMPORT_RUNNING}</p>}
          {outcome?.tone === "done" && <p role="status" style={{ margin: 0, fontSize: 12.5, color: "var(--ok)" }}>{outcome.text}</p>}
          {outcome?.tone === "failed" && <ErrorState message={outcome.text} />}
        </Panel>
      </section>

      {catalog && (
        <section aria-label="Itens para revisar">
          <Panel title="Itens para revisar">
            {note && <p style={muted}>{note}</p>}
            {review.length === 0 ? <EmptyState message="nenhum item para revisar" /> : (
              <DataTable<ReviewItemView>
                columns={[
                  { key: "catmatCode", label: "CATMAT", render: (i) => String(i.catmatCode) },
                  { key: "sourceDescription", label: "Descrição de origem", render: (i) => i.sourceDescription },
                  {
                    key: "parseStatus", label: "Situação",
                    render: (i) => {
                      const s = parseStatusView(i.parseStatus);
                      return <Tag tone={s.tone}>{s.label}</Tag>;
                    }
                  }
                ]}
                rows={review}
                rowKey={(i) => i.catmatCode}
              />
            )}
          </Panel>
        </section>
      )}
    </div>
  );
}

const muted: CSSProperties = { margin: 0, fontSize: 11.5, color: "var(--ink3)" };
const row: CSSProperties = { display: "flex", gap: 8, flexWrap: "wrap" };
```

Confira, antes de rodar, que `ErrorState` desenha `role="alert"` (os testes do 19b já dependem disso: `src/components/ErrorState.tsx`).

Em `src/screens/Shell.tsx`:

```tsx
import { Audit } from "./Audit";
import { MedicationCatalog } from "./MedicationCatalog";
```

```tsx
// Casca do mantenedor logado (Task 6): cabeçalho com e-mail e navegação,
// corpo com a tela ativa. Mantenedores (Task 7), Tokens (Task 8) e
// Auditoria (Task 9) já existem. Módulo 19c: Medicamentos, o catálogo da
// plataforma (vale para todas as cidades do ambiente).
export type Screen = "cities" | "medications" | "maintainers" | "tokens" | "audit";

const NAV: { key: Screen; label: string }[] = [
  { key: "cities", label: "Cidades" },
  { key: "medications", label: "Medicamentos" },
  { key: "maintainers", label: "Mantenedores" },
  { key: "tokens", label: "Tokens" },
  { key: "audit", label: "Auditoria" }
];
```

e no `<main>`, logo depois do bloco de `screen === "cities"`:

```tsx
        {screen === "medications" && <MedicationCatalog />}
```

Gere os tipos:

```bash
cd apps/maintenance/.claude/mod19c && npm run codegen && /opt/homebrew/bin/git status --short src/gql
```

Expected: `src/gql/gql.ts` e `src/gql/graphql.ts` modificados, com `MedicationCatalogQuery`, `ImportMedicationCatalogMutation` e `ImportAnvisaListsMutation`.

No `README.md`, na tabela "## Telas", logo depois da linha de **Cidades**:

```markdown
| **Medicamentos** | Catálogo de medicamentos da plataforma (módulo 19c): versão atual do CATMAT (classe 6505), "importar o CATMAT" (Compras.gov; diz o que entrou, mudou e saiu), "importar as listas da Anvisa" (antimicrobianos e controlados) e os itens ocultos para revisão. Vale para todas as cidades do ambiente; uma importação por vez |
```

E, no fim da seção "### Funcionalidades" (logo antes do bloco `> **Ordem de deploy:**` dela), acrescente:

```markdown
**Documentos clínicos (módulo 19c).** O interruptor `clinical_documents`
("Documentos clínicos (atestado, declaração, receita, requisição)") aparece na
mesma lista e liga como os outros; ele precisa do `clinical_record` ligado e
utilizável. O catálogo de medicamentos que a receita usa é da plataforma e fica
na tela **Medicamentos**. Ordem de deploy: api → dashboard → maintenance;
contra um api sem o 19c, só a tela Medicamentos mostra o aviso.
```

- [ ] **Step 5: Run test to verify it passes**

Run: `cd apps/maintenance/.claude/mod19c && npx vitest run src/screens/MedicationCatalog.test.tsx src/screens/FeaturesTab.test.tsx src/screens/Shell.test.tsx && npx tsc --noEmit && npm run codegen:check`
Expected: PASS, `tsc` sem erro e `codegen:check` verde.

- [ ] **Step 6: Commit**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/maintenance/.claude/mod19c
/opt/homebrew/bin/git add schema.graphql src/gql/gql.ts src/gql/graphql.ts src/screens/MedicationCatalog.tsx \
  src/screens/MedicationCatalog.test.tsx src/screens/Shell.tsx src/screens/Shell.test.tsx src/screens/FeaturesTab.test.tsx README.md
/opt/homebrew/bin/git status --short
/opt/homebrew/bin/git commit -m "feat: add the medication catalog screen with CATMAT and Anvisa imports to maintenance

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

Expected: `status --short` antes do commit sem nada fora da lista (se `src/gql/index.ts` mudou, acrescente-o pelo caminho).

---

### Task 3: Suíte, codegen:check, build, revisão e prova no navegador

**Files:** nenhum novo.

**Interfaces:**
- Consumes: as Tasks 1–2 na branch `feat/mod-19c-documents`; o api do 19c rodando na 3038 com a semente.
- Produces: branch pronta para merge (depois do merge do api e do dashboard do 19c).

- [ ] **Step 1: Suíte, tipos, codegen e build**

Run: `cd apps/maintenance/.claude/mod19c && npx vitest run && npx tsc --noEmit && npm run codegen:check && npm run build`
Expected: tudo verde (a CI roda os quatro): a base anotada na Task 0 mais **2 arquivos de teste novos**. Um número que dobra é artefato de build descoberto pelo vitest: pare e reporte.

```bash
cd apps/maintenance/.claude/mod19c && /opt/homebrew/bin/git status --short && /opt/homebrew/bin/git log --stat origin/main..HEAD | grep -c node_modules
```

Expected: `status` vazio e a contagem `0`.

- [ ] **Step 2: Nada de cidade na tela de plataforma e nenhum campo novo no topo**

```bash
cd apps/maintenance/.claude/mod19c
grep -n -E "citySlug|slug" src/screens/MedicationCatalog.tsx ; echo "exit $?"
sed -n '/const CityHeaderQuery/,/`);/p' src/screens/CityDetail.tsx | grep -n -i -E "medication|catmat" ; echo "exit $?"
```

Expected: os dois `exit 1` (a tela não recebe nem manda cidade; o `CityHeader` não pede campo do 19c).

- [ ] **Step 3: Revisão**

Um subagente revisor lê `origin/main..HEAD` do maintenance contra a spec (§3, §7 "Maintenance", §11 F-19.16, §12), o ADR 0033 e o contrato §8. Pontos de atenção:
- o liga/desliga de `clinical_documents` é o genérico (nenhuma linha nova no fluxo de `setCityFeature`); a chave e o pré-requisito aparecem em português;
- `MedicationCatalog` é consulta própria; contra api antigo só ela falha; uma importação por vez; o resultado é o que o api devolveu (nunca o pedido);
- os códigos de erro da importação (Divergência M2) traduzidos e os desconhecidos crus; nulos viram "—";
- `schema.graphql` e `src/gql/*` iguais ao que o api em execução produz (`codegen:check` verde);
- nenhum `git add -A` no histórico.

- [ ] **Step 4: Prova no navegador (com o usuário)**

Depende do api do 19c na porta **3038** (plano do api do 19c), com acesso de saída à internet do container (a importação chama `dadosabertos.compras.gov.br`). O proxy do Vite troca o Host para `maintenance-api.localhost` e injeta o Origin `http://maintenance.localhost:5177`, então o Vite do worktree sobe **na 5177**, num container da rede do compose (para alcançar `http://api:3038`), depois de parar o serviço `maintenance`:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose stop maintenance
docker compose run -d --rm --no-deps --name maintenance-mod19c -p 5177:5177 \
  -w /app/.claude/mod19c -e VITE_API_PROXY_TARGET=http://api:3038 -e VITE_MAINTENANCE_ENV=development \
  maintenance npx vite --port 5177 --host 0.0.0.0
docker logs -f maintenance-mod19c   # espere o "ready"; Ctrl+C sai do log
```

(Se o plano do api subir o servidor do 19c num container próprio, use o nome dele no lugar de `api`.)

Abra `http://maintenance.localhost:5177` (nunca `localhost:5177`). O usuário faz o login do mantenedor; não digite senha nem TOTP. Confira com screenshot:
- **Medicamentos** na navegação; a tela diz "nenhuma importação ainda…" (ou a versão que o api semeou);
- "importar o CATMAT" → "confirmar: importar o CATMAT agora": o aviso "importando…" com os dois botões travados; ao fim, "Catálogo importado: N entraram, …" e a versão atual com a contagem de itens e de ocultos; "Itens para revisar" com a descrição de origem crua e "para revisar"; com mais de 50 ocultos, "mostrando 50 de N itens ocultos";
- "importar as listas da Anvisa" → a frase com as contagens;
- importar de novo logo depois: "0 entraram, 0 mudaram, 0 saíram" (ou o que o CATMAT de fato mudou);
- Cidades → Curitiba → **Funcionalidades**: `clinical_documents` com "Documentos clínicos (atestado, declaração, receita, requisição)"; com `clinical_record` desligado, "ligada, falta pré-requisito" / "prontuário da atenção primária (clinical_record) desligado" depois de ligar; com ele ligado e utilizável, "ligada e utilizável"; a tela **Auditoria** mostra a mudança;
- nenhum `undefined`/`null` na tela.

Depois: `docker rm -f maintenance-mod19c && docker compose start maintenance`. Deixe os interruptores da cidade de dev como estavam antes da prova (o catálogo importado fica: é o que o dashboard do 19c usa na prova dele).

- [ ] **Step 5:** **Pare.** O merge do maintenance só vem depois do merge do api e do dashboard do 19c, e só com autorização explícita do usuário (merge, push e cards, uma etapa de cada vez). Antes do push, confira `origin/main..main` no maintenance e publique só os commits desta entrega. Comandos, quando autorizados:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/maintenance
/opt/homebrew/bin/git fetch origin
/opt/homebrew/bin/git status -sb
/opt/homebrew/bin/git merge --ff-only feat/mod-19c-documents
/opt/homebrew/bin/git log --oneline origin/main..main
/opt/homebrew/bin/git push origin main
/opt/homebrew/bin/git worktree remove .claude/mod19c
/opt/homebrew/bin/git branch -d feat/mod-19c-documents
```

---

## Divergências propostas ao contrato

1. **M1 — Tipos, nulidade e lugar dos campos do §8.** O contrato dá os nomes (`medicationCatalog { currentRelease { id importedAt itemsCount hiddenCount } reviewItems(first) { catmatCode sourceDescription parseStatus } }`, `importMedicationCatalog`, `importAnvisaLists`). Proposta, na raiz (são de plataforma, não de cidade):
   ```graphql
   type MedicationCatalog { currentRelease: MedicationCatalogRelease, reviewItems(first: Int = 50): [MedicationCatalogItem!]! }
   type MedicationCatalogRelease { id: ID!, importedAt: ISO8601DateTime!, itemsCount: Int!, hiddenCount: Int! }
   type MedicationCatalogItem { catmatCode: Int!, sourceDescription: String!, parseStatus: String! }   # parseStatus: "ok" | "review"
   type ImportMedicationCatalogPayload { ok: Boolean!, releaseId: ID, added: Int, changed: Int, removed: Int, errors: [UserError!]! }
   type ImportAnvisaListsPayload { ok: Boolean!, antimicrobials: Int, controlled: Int, errors: [UserError!]! }
   # Query:    medicationCatalog: MedicationCatalog!
   # Mutation: importMedicationCatalog: ImportMedicationCatalogPayload   (sem argumentos)
   #           importAnvisaLists: ImportAnvisaListsPayload               (sem argumentos)
   ```
   `currentRelease` nulo antes da primeira importação; `reviewItems` traz os ocultos com `parseStatus: "review"`, em ordem de código CATMAT, até `first` (a tela pede 50); `hiddenCount` conta todos os ocultos. `catmatCode` inteiro, como no esquema canônico (`contracts` C4).
2. **M2 — Códigos de recusa da importação.** O contrato só diz `errors`. Este plano usa os do plano do `api` do 19c (`UserError.message`, `path: "catalog"`): `source_unavailable`, `incomplete_source`, `suspicious_drop` e `import_failed` (CATMAT) e `invalid_file`/`import_failed` (listas da Anvisa); com `ok: false`, a versão atual não muda. Proposta a mais: `import_in_progress` quando já há uma release `importing` (o plano do `api` não trava duas importações simultâneas; a tela trava os botões, mas outra aba ou outro mantenedor pode disparar). A tela já traduz o código.
3. **M3 — Revisão dos itens é só leitura nesta entrega.** O spec §7 diz "revisão de itens" e o contrato §8 só dá a lista. Corrigir um item (princípio ativo, concentração, forma, tirar do oculto) precisa de uma mutation que o contrato não tem; ficou para a curadoria real do CATMAT (spec §12, pendência de go-live). Se o contrato ganhar `reviewMedicationCatalogItem(catmatCode, …)`, a tela ganha uma ação na linha.
4. **M4 — Importação síncrona e produção.** O contrato devolve as contagens na resposta da mutation, então a importação é síncrona (a tela avisa que pode demorar). Se o api a mover para job, o payload precisa de um estado ("em andamento") e a tela de um "atualizar" até terminar. E o maintenance só existe em development e staging: a carga do catálogo em **produção** precisa de outro caminho (um rake no api, como o `terminology:import` do 16) — fica registrado como pendência de go-live.

## Self-review

- **Cobertura (spec §3, §7 "Maintenance", §11 F-19.16; contrato §8):** interruptor `clinical_documents` pelo mecanismo genérico, com a chave e o pré-requisito em português — Tasks 1 e 2 (teste do liga/desliga sem mudar o fluxo); importação do CATMAT com entrou/mudou/saiu — Tasks 1 e 2; listas da Anvisa — Tasks 1 e 2; revisão de itens (lista dos ocultos, truncada em 50 com aviso) — Tasks 1 e 2 (a ação de corrigir é a Divergência M3); ordem de deploy — consulta própria, README e Task 3; prova no navegador — Task 3.
- **Placeholders:** nenhum; todo passo de código tem o código. As únicas variáveis são as do ambiente do api do 19c (nome do container se não for `api`), que o plano do api define.
- **Tipos:** `ReleaseView`, `ReviewItemView`, `CatalogImportView` e `AnvisaImportView` (Task 1) aceitam os tipos gerados da M1 (`releaseId?: Maybe<string>`, `added?: Maybe<number>`, `errors: Array<{ path?: Maybe<string>, message: string }>`); `MedicationCatalog()` é o que a `Shell` monta; os nomes das operações são os mesmos na tela e nos testes.
- **Review Focus:** 1 — Task 2 ("importar o CATMAT pede confirmação, trava os dois botões…"); 2 — Task 1 ("erros da importação…") e Task 2 ("Compras.gov fora do ar…"); 3 — Task 1 ("resultado do CATMAT…; com saída, explica") e Task 2 (mesmo teste da importação, com `removed: 12`); 4 — Task 1 ("aviso de lista truncada") e Task 2 ("mais ocultos do que a página…"); 5 — Task 2 ("api sem o 19c…" e "nenhuma importação ainda…").
