# Módulo 16 — Modo de prontuário e exportação (admin) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** No console de plataforma (`apps/admin`), o operador abre a **ficha da cidade** a partir de "Cidades", edita o modo de prontuário, o código IBGE e o endereço do PEC (com confirmação ao mudar o modo e as recusas 422 em português), vê os interruptores da cidade só para leitura (ligado, utilizável, o que falta; quem liga é o `maintenance`) e acompanha a tela **Produção das cidades**, com os alertas `attention`/`critical` e o aviso de SIGTAP não importada.

**Architecture:** Cliente e tipos novos em `src/lib/api.ts` e `src/lib/types.ts`, copiados do arquivo de contratos (§4). As regras ficam fora do React, em funções puras testadas: modo, validação, payload, recusas e rótulos dos interruptores em `src/lib/recordSettings.ts`; prazo, ordem, alertas e aviso da SIGTAP em `src/lib/cityProduction.ts`. A ficha (`src/modules/cities/CityRecord.tsx`) abre de dentro de `Cities.tsx` por estado local, no mesmo modelo de navegação sem roteador do console; o formulário é `RecordSettingsForm.tsx` e os interruptores, `CityFeatures.tsx`. A tela de produção é um módulo novo de navegação (`city_production`), no padrão do `CityAnalytics` do módulo 14.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, Vitest 2 + Testing Library 16 (jsdom, sem jest-dom, sem msw: `vi.mock("../../lib/api")` e `vi.stubGlobal("fetch")`), Vite 5 (proxy de dev).

**Spec:** `docs/superpowers/specs/2026-10-05-module-16-record-mode-and-export-design.md` (§3.1, §3.2, §4 último item, §6.5 "Console" e §9 "Front: admin" são deste plano), `docs/adr/0028.md` e o arquivo de contratos `docs/superpowers/plans/2026-10-05-module-16-record-mode-contracts.md` (§2 e §4 são deste plano). Os tipos da Task 1 copiam o contrato literalmente. Os planos do api (fundação **e** exportador) precisam estar **mergeados antes** do merge deste.

## Global Constraints

- Rotas usadas (todas no `PlatformConsoleHost`, sessão de operador com MFA; nenhuma é nova no proxy salvo `/city_production`):
  - `GET /cities` → `{ data: CityRow[] }`; cada linha ganha `record_mode`, `ibge_code`, `pec_url`, `city_reachable`, `features` (contratos §4.2);
  - `GET /cities/:id` → objeto solto (sem envelope) com os campos do item da lista (`id`, `slug`, `name`, `uf`, `status`, `schema_version`, `time_zone`, `created_at`) e os mesmos campos novos;
  - `PATCH /cities/:id/record_settings` com qualquer subconjunto de `{ record_mode, ibge_code, pec_url }` → 200 `{ city }`; 422 `invalid_record_mode`, `invalid_ibge_code`, `invalid_pec_url`; 503 `city_unreachable` quando `ibge_code` veio e o banco da cidade não responde; 404 `{ "error": "not_found" }` para cidade inexistente;
  - `GET /city_production` → `{ data: { cities, terminology } }` (mesmo envelope de `/cities` e `/city_analytics`); `competences` vem com a corrente primeiro; `terminology.sigtap_alert` é calculado no api.
- `ibge_code` tem **uma** fonte: o `city_profile.ibge_code` do banco da cidade, o mesmo que o "Provisionar cidade" já preenche (`POST /cities`). O PATCH grava lá. A ficha mostra e edita esse campo como **o** código IBGE da cidade — não existe variante "da plataforma" e a tela não fala em duas fontes.
- Banco da cidade inalcançável: a resposta é 200 com `ibge_code: null` e `city_reachable: false`. Nesse estado o campo IBGE fica **bloqueado** (com o motivo na tela) e o PATCH nunca leva `ibge_code` — `null` ali não quer dizer "vazio".
- `record_mode` ∈ `off` | `integrated` | `record`. Rótulos: Desligado, Integrado ao PEC, Prontuário Rota Saúde.
- `ibge_code`: 7 dígitos ou vazio. `pec_url`: só `https://`, sem usuário nem senha na URL, ou vazio. Campo vazio vai como `null` (`null` limpa; o api também trata `""` como `null`). `record_mode` nunca vai nulo. O PATCH leva **só os campos que mudaram**; sem mudança, nada é enviado.
- Mudar `record_mode` sempre pede confirmação na tela antes do PATCH. Mudar só IBGE ou PEC não pede.
- Interruptores: o console **só lê** (`features: [{ key, enabled, usable, missing }]`). Nenhum controle de ligar/desligar no admin (ADR 0028, "Interruptores escritos também pelo console `admin`" foi rejeitado). A tela diz que quem liga é o mantenedor, no `maintenance`.
- Catálogo de interruptores (contratos §2): `ledi_export` (`record_mode_off`, `pec_url_missing`, `ibge_code_missing`, `credential_missing:ledi`, `credential_unauthorized:ledi`) e `cadsus_lookup` (`credential_missing:cadsus`, `credential_unauthorized:cadsus`). Chave ou código desconhecido aparece cru, nunca quebra a tela.
- `alert`: `none` | `attention` (≤ 5 dias úteis com pendente/recusada) | `critical` (`record` e zero aceitas a ≤ 3 dias úteis). Competência `AAAAMM` aparece como `MM/AAAA`; `deadline_on` (`YYYY-MM-DD`) é formatado por texto, nunca por `new Date(...)` (meia-noite UTC vira o dia anterior em São Paulo).
- Aviso da SIGTAP (spec §4): a regra do dia 5 é do **api** (`terminology.sigtap_alert`). O console não calcula data: `sigtap_alert: true` → alerta; `sigtap_imported: false` sem alerta → aviso neutro; importada → nada.
- O `admin` consome os mesmos `/admin/api/*` do dashboard; este módulo **não** mexe neles nem nos módulos legados.
- Interface tem ciclo próprio: use os componentes e estilos existentes (`PageHeader`, `Panel`, `DataTable`, `Tag`, `EmptyState`, `ErrorState`, `Skeleton`, o `Field`/`inputStyle` do `RegisterChannel`), sem redesign.
- Nenhum dado de cidadão passa pelo console. A URL do PEC nunca entra em mensagem de erro (a auditoria do api também não leva valor de URL, contratos §4.1).
- Base do ambiente de teste: `vitest.config.ts` usa `jsdom`, `globals: false` (todo teste de componente chama `afterEach(cleanup)`), sem `setupFiles` nem jest-dom (use `toBeTruthy`, `toBeNull`, `.textContent`).
- Nunca `git add -A` nem `git add .`: o worktree tem o symlink de `node_modules`. Adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta).

## Ambiente de execução

- Worktree a partir de `origin/main`, com o `node_modules` do checkout principal ligado (o `.claude/` do admin não está no `.gitignore`; ele já aparece como não rastreado no checkout principal e nunca entra em commit):

  ```bash
  cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/admin
  /opt/homebrew/bin/git fetch origin
  /opt/homebrew/bin/git worktree add .claude/mod16 -b feat/mod-16-record-mode origin/main
  ln -s ../../node_modules .claude/mod16/node_modules
  ```

- Testes (script `test` = `vitest run`):

  ```bash
  cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/admin/.claude/mod16 && npx vitest run <arquivos>
  ```

- Tipos antes de cada commit (script `typecheck` = `tsc --noEmit`):

  ```bash
  cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/admin/.claude/mod16 && npx tsc --noEmit
  ```

- O api do módulo 16 roda em dev na porta **3033** (isolado do api da `main` em 3030 e do módulo 15 em 3032); a prova no navegador aponta o proxy do Vite para ele (Task 7).

## Review Focus

1. **Endereço do PEC com usuário e senha** (`https://admin:segredo@pec.cidade.gov.br`), ou `http://`. O operador cola o que recebeu da cidade. Esperado: a tela recusa antes de chamar o api, a mensagem é fixa e não repete a URL nem a senha, e nada é enviado. Testes:
   - Task 2, "pecUrlProblem refuses http, credentials and garbage";
   - Task 4, "PEC with user and password is refused before the API and never echoed".
2. **Salvar sem mudar o modo**, inclusive depois de trocar o modo e voltar ao original, e **Cancelar** a confirmação. Esperado: nenhuma confirmação quando o modo não muda, o PATCH nunca reenvia `record_mode` igual, "Nada mudou." quando não há diferença, e Cancelar não manda nada e mantém o que foi escolhido. Testes:
   - Task 2, "recordSettingsPatch sends only what changed";
   - Task 4, "changing only the IBGE code saves without confirmation", "mode back to the original: no confirmation and nothing sent" e "cancel sends nothing and keeps the chosen mode".
3. **Interruptores depois de salvar.** O operador preenche o PEC e espera ver "endereço do PEC" sumir do que falta. Esperado: a ficha mostra o `features` que veio na resposta do PATCH (o servidor decide o que falta, nunca a tela), e a lista de cidades é recarregada com o modo novo. Teste:
   - Task 4, "after saving, the switches follow the server and the city list is refreshed".
4. **Prazo nas bordas e cidade sem competência**: `business_days_left` 1, 0 e negativo, e cidade (`off`, recém-ativada) sem nenhuma competência. Esperado: "1 dia útil", "último dia", "prazo vencido", e uma linha "sem fichas" em vez de sumir ou quebrar. Testes:
   - Task 5, "deadlineText at the edges" e "productionRows keeps a city without competences";
   - Task 6, "deadline edges and a city without competences".
5. **Banco da cidade inalcançável** (`city_reachable: false`, `ibge_code: null`). O operador vê o IBGE vazio e digitaria de novo um código que já existe. Esperado: o campo IBGE fica bloqueado com o motivo, o PATCH de modo/PEC continua funcionando e nunca leva `ibge_code`; na lista, nada quebra. Testes:
   - Task 2, "recordSettingsPatch never sends the IBGE code of an unreachable city";
   - Task 4, "unreachable city: IBGE locked, mode and PEC still save without it".
---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| `src/lib/types.ts`, `src/lib/api.ts`, `vite.config.ts`, `README.md` | tipos do contrato §4; `getCity`, `updateCityRecordSettings`, `listCityProduction`; proxy de `/city_production` | 1 |
| `src/lib/recordSettings.ts` | rótulos de modo, aviso de mudança, formulário, validação, payload, recusas, rótulos e estado dos interruptores | 2 |
| `src/hooks/useCities.ts`, `src/modules/Cities.tsx`, `src/modules/cities/CityRecord.tsx`, `src/modules/cities/CityFeatures.tsx` | coluna "Prontuário", botão "Ficha", ficha com interruptores só leitura | 3 |
| `src/modules/cities/RecordSettingsForm.tsx`, `src/modules/cities/CityRecord.tsx`, `README.md` | edição de modo, IBGE e PEC com confirmação | 4 |
| `src/lib/cityProduction.ts` | competência, prazo, alerta, ordem das linhas, contagem, aviso da SIGTAP | 5 |
| `src/modules/production/CityProduction.tsx`, `src/shell/modules.ts`, `src/App.tsx`, `README.md` | tela "Produção das cidades" e item de navegação | 6 |
| — | suíte, build, revisão e prova no navegador | 7 |

---

### Task 1: Tipos do contrato, cliente HTTP e proxy

**Files:**
- Modify: `src/lib/types.ts` (fim do arquivo, bloco "Cidades")
- Modify: `src/lib/api.ts` (import do topo e fim do arquivo)
- Modify: `vite.config.ts`, `README.md`
- Test: `src/lib/recordSettingsApi.test.ts`

**Interfaces:**
- Consumes: `jsonFetch`, `ApiError` (existentes em `src/lib/api.ts`).
- Produces:
  - em `types.ts`: `RecordMode`, `CityFeatureState`, `CityRecordFields`, `CityDetail`; `CityRow` ganha `record_mode?`, `ibge_code?`, `pec_url?`, `features?`;
  - em `api.ts`: `RecordSettingsPatch`, `getCity(cityId: string): Promise<CityDetail>`, `updateCityRecordSettings(cityId: string, patch: RecordSettingsPatch): Promise<CityDetail>`, `ProductionAlert`, `CompetenceSummary`, `CityProductionCity`, `CityProductionTerminology`, `CityProductionData`, `listCityProduction(): Promise<CityProductionData>`.

- [ ] **Step 1: Write the failing test**

Create `src/lib/recordSettingsApi.test.ts`:

```ts
import { describe, it, expect, vi, afterEach } from "vitest";
import {
  getCity, updateCityRecordSettings, listCityProduction, ApiError, type CityProductionData
} from "./api";
import type { CityDetail } from "./types";

const DETAIL: CityDetail = {
  id: "c-cwb", slug: "curitiba", name: "Curitiba", uf: "PR", status: "active", schema_version: "1",
  time_zone: "America/Sao_Paulo", created_at: "2026-09-01T00:00:00Z",
  record_mode: "integrated", ibge_code: "4106902", pec_url: "https://pec.curitiba.pr.gov.br", city_reachable: true,
  features: [ { key: "ledi_export", enabled: true, usable: false, missing: [ "credential_missing:ledi" ] } ]
};

const PRODUCTION: CityProductionData = {
  cities: [ {
    slug: "curitiba", name: "Curitiba", record_mode: "record",
    competences: [ {
      competence: "202610", deadline_on: "2026-11-14", business_days_left: 7,
      accepted: 120, rejected: 3, pending: 10, failed: 0, alert: "none"
    } ]
  } ],
  terminology: { sigtap_current_competence: "202610", sigtap_imported: true, sigtap_alert: false }
};

function respond(body: unknown, status = 200) {
  return vi.fn().mockResolvedValue(new Response(JSON.stringify(body), { status }));
}

describe("module 16 console client", () => {
  afterEach(() => vi.unstubAllGlobals());

  it("getCity GETs /cities/:id with the session cookie and returns the bare object", async () => {
    const fetchMock = respond(DETAIL);
    vi.stubGlobal("fetch", fetchMock);

    await expect(getCity("c-cwb")).resolves.toEqual(DETAIL);
    const [ url, init ] = fetchMock.mock.calls[0];
    expect(url).toBe("/cities/c-cwb");
    expect(init.credentials).toBe("include");
    expect(init.method ?? "GET").toBe("GET");
  });

  it("updateCityRecordSettings PATCHes only the given fields and unwraps city", async () => {
    const fetchMock = respond({ city: DETAIL });
    vi.stubGlobal("fetch", fetchMock);

    await expect(updateCityRecordSettings("c-cwb", { ibge_code: "4106902", pec_url: null })).resolves.toEqual(DETAIL);
    const [ url, init ] = fetchMock.mock.calls[0];
    expect(url).toBe("/cities/c-cwb/record_settings");
    expect(init.method).toBe("PATCH");
    expect(JSON.parse(init.body)).toEqual({ ibge_code: "4106902", pec_url: null });
    expect(init.headers[ "Content-Type" ]).toBe("application/json");
    expect(init.credentials).toBe("include");
  });

  it("updateCityRecordSettings raises ApiError 404 not_found for an unknown city", async () => {
    vi.stubGlobal("fetch", respond({ error: "not_found" }, 404));

    const err = await updateCityRecordSettings("nope", { pec_url: null }).catch((e) => e);
    expect(err).toBeInstanceOf(ApiError);
    expect((err as ApiError).status).toBe(404);
    expect((err as ApiError).body).toEqual({ error: "not_found" });
  });

  it("updateCityRecordSettings raises ApiError with the server code on 422", async () => {
    vi.stubGlobal("fetch", respond({ error: "invalid_ibge_code" }, 422));

    const err = await updateCityRecordSettings("c-cwb", { ibge_code: "4106902" }).catch((e) => e);
    expect(err).toBeInstanceOf(ApiError);
    expect((err as ApiError).status).toBe(422);
    expect((err as ApiError).body).toEqual({ error: "invalid_ibge_code" });
  });

  it("listCityProduction GETs /city_production and unwraps data", async () => {
    const fetchMock = respond({ data: PRODUCTION });
    vi.stubGlobal("fetch", fetchMock);

    await expect(listCityProduction()).resolves.toEqual(PRODUCTION);
    const [ url, init ] = fetchMock.mock.calls[0];
    expect(url).toBe("/city_production");
    expect(init.credentials).toBe("include");
  });

  it("listCityProduction raises ApiError 401 when the session is missing", async () => {
    vi.stubGlobal("fetch", vi.fn().mockResolvedValue(new Response("", { status: 401 })));

    const err = await listCityProduction().catch((e) => e);
    expect(err).toBeInstanceOf(ApiError);
    expect((err as ApiError).status).toBe(401);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run src/lib/recordSettingsApi.test.ts`
Expected: FAIL — `getCity`/`updateCityRecordSettings`/`listCityProduction` não existem no módulo (e `CityDetail` não existe em `types.ts`).

- [ ] **Step 3: Add the types**

Em `src/lib/types.ts`, substitua o bloco final `CityRow` por:

```ts
// ─── Cidades (catálogo do console, Plano 6) ──────────────────────────────────
export interface CityRow {
  id: string;
  slug: string;
  name: string;
  uf: string | null;
  status: string;
  schema_version: string | null;
  // Fuso IANA da cidade (api#27); ausente em api antigo.
  time_zone?: string;
  created_at: string;
  // Módulo 16 (contratos §4.2); ausentes em api antigo.
  record_mode?: RecordMode;
  ibge_code?: string | null;
  pec_url?: string | null;
  city_reachable?: boolean;
  features?: CityFeatureState[];
}

// ─── Módulo 16 (ADR 0028): modo de prontuário e interruptores ───────────────
// record_mode, ibge_code e pec_url são do operador (PATCH record_settings).
// features é SÓ LEITURA aqui: quem liga e desliga é o maintenance.
export type RecordMode = "off" | "integrated" | "record";

export interface CityFeatureState {
  key: string;
  enabled: boolean;
  usable: boolean;
  missing: string[];
}

// ibge_code mora no city_profile do banco da cidade (fonte única). Com o banco
// inalcançável, o api manda ibge_code: null e city_reachable: false — null aí
// NÃO é "vazio", e a ficha bloqueia a edição do IBGE.
export interface CityRecordFields {
  record_mode: RecordMode;
  ibge_code: string | null;
  pec_url: string | null;
  city_reachable: boolean;
  features: CityFeatureState[];
}

// GET /cities/:id (objeto solto, com os campos do item da lista) e o `city` de
// PATCH /cities/:id/record_settings.
export interface CityDetail extends Omit<CityRow, keyof CityRecordFields>, CityRecordFields {}
```

- [ ] **Step 4: Add the client**

Em `src/lib/api.ts`, troque o import do topo:

```ts
import type { CityRow } from "./types";
```

por:

```ts
import type { CityDetail, CityRow, RecordMode } from "./types";
```

e acrescente no fim do arquivo:

```ts
// ─── Módulo 16 (ADR 0028; contratos §4) ─────────────────────────────────────
// Ficha da cidade: GET /cities/:id (objeto solto) e
// PATCH /cities/:id/record_settings (qualquer subconjunto; null limpa o campo).
// null limpa pec_url/ibge_code; record_mode nunca é nulo. 404 { error: "not_found" }.
// 422: invalid_record_mode, invalid_ibge_code, invalid_pec_url. 503 city_unreachable
// quando ibge_code veio e o banco da cidade não responde: o IBGE mora no
// city_profile da cidade (fonte única, a mesma do provisionamento).
export interface RecordSettingsPatch {
  record_mode?: RecordMode;
  ibge_code?: string | null;
  pec_url?: string | null;
}

export async function getCity(cityId: string): Promise<CityDetail> {
  return jsonFetch<CityDetail>(`/cities/${encodeURIComponent(cityId)}`);
}

export async function updateCityRecordSettings(cityId: string, patch: RecordSettingsPatch): Promise<CityDetail> {
  const res = await jsonFetch<{ city: CityDetail }>(`/cities/${encodeURIComponent(cityId)}/record_settings`, {
    method: "PATCH",
    body: JSON.stringify(patch)
  });
  return res.city;
}

// GET /city_production (contratos §4.3) — resumo por cidade da competência
// corrente e da anterior (corrente primeiro), no envelope { data }. Entregue
// pelo plano api-exporter. sigtap_alert vem calculado do api (dia ≥ 5).
export type ProductionAlert = "none" | "attention" | "critical";

export interface CompetenceSummary {
  competence: string;          // AAAAMM
  deadline_on: string;         // YYYY-MM-DD
  business_days_left: number;
  accepted: number;
  rejected: number;
  pending: number;
  failed: number;
  alert: ProductionAlert;
}

export interface CityProductionCity {
  slug: string;
  name: string;
  record_mode: RecordMode;
  competences: CompetenceSummary[];
}

export interface CityProductionTerminology {
  sigtap_current_competence: string;
  sigtap_imported: boolean;
  sigtap_alert: boolean;
}

export interface CityProductionData {
  cities: CityProductionCity[];
  terminology: CityProductionTerminology;
}

export async function listCityProduction(): Promise<CityProductionData> {
  const res = await jsonFetch<{ data: CityProductionData }>("/city_production");
  return res.data;
}
```

- [ ] **Step 5: Proxy and README**

Em `vite.config.ts`, depois da linha de comentário `//   /city_analytics                 → indicadores publicados pelas cidades (só leitura)`, acrescente:

```ts
//   /city_production                → produção e-SUS de todas as cidades (só leitura, módulo 16)
```

e, no objeto `proxy`, troque:

```ts
      "/city_analytics": proxy(TARGET)
```

por:

```ts
      "/city_analytics": proxy(TARGET),
      "/city_production": proxy(TARGET)
```

(`/cities` já está no proxy e cobre `/cities/:id/record_settings`.)

Em `README.md`, troque `` `/unknown_channels` e `/city_analytics` para `` por `` `/unknown_channels`, `/city_analytics` e `/city_production` para ``.

- [ ] **Step 6: Run test to verify it passes**

Run: `npx vitest run src/lib/recordSettingsApi.test.ts && npx tsc --noEmit`
Expected: PASS (6 testes) e `tsc` sem erro.

- [ ] **Step 7: Commit**

```bash
/opt/homebrew/bin/git add src/lib/types.ts src/lib/api.ts src/lib/recordSettingsApi.test.ts vite.config.ts README.md
/opt/homebrew/bin/git commit -m "$(cat <<'EOF'
feat: add the module 16 console client for record settings and city production

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 2: Regras do modo, do IBGE, do PEC e dos interruptores

**Files:**
- Create: `src/lib/recordSettings.ts`
- Test: `src/lib/recordSettings.test.ts`

**Interfaces:**
- Consumes: `ApiError`, `apiErrorCode`, `RecordSettingsPatch` (`api.ts`); `CityDetail`, `CityFeatureState`, `RecordMode` (`types.ts`).
- Produces:
  - `RECORD_MODES: { value: RecordMode; label: string; hint: string }[]`;
  - `recordModeLabel(mode: string | null | undefined): string`;
  - `modeChangeWarning(from: string, to: string): string`;
  - `type RecordSettingsValues = { record_mode: string; ibge_code: string; pec_url: string }`;
  - `type RecordSettingsField = "ibge_code" | "pec_url"`;
  - `FIELD_PROBLEMS: Record<RecordSettingsField, string>`;
  - `type RecordFields = Pick<CityDetail, "record_mode" | "ibge_code" | "pec_url"> & { city_reachable?: boolean }`;
  - `formFrom(city: RecordFields): RecordSettingsValues`;
  - `ibgeLocked(city: RecordFields): boolean` (`city_reachable === false`); `IBGE_LOCKED_TEXT: string`;
  - `pecUrlProblem(raw: string): boolean`;
  - `validateRecordSettings(values: RecordSettingsValues): RecordSettingsField[]`;
  - `recordSettingsPatch(city, values): RecordSettingsPatch`;
  - `changesMode(patch: RecordSettingsPatch): boolean`;
  - `describeRecordSettingsError(err: unknown): string`;
  - `featureLabel(key: string): string`, `missingLabel(code: string): string`;
  - `featureState(f: CityFeatureState): { kind: "off" | "usable" | "blocked"; text: string; tone: "neutral" | "ok" | "warn" }`.

- [ ] **Step 1: Write the failing test**

Create `src/lib/recordSettings.test.ts`:

```ts
import { describe, it, expect } from "vitest";
import { ApiError } from "./api";
import {
  FIELD_PROBLEMS, changesMode, describeRecordSettingsError, featureLabel, featureState, formFrom, ibgeLocked,
  missingLabel, modeChangeWarning, pecUrlProblem, recordModeLabel, recordSettingsPatch, validateRecordSettings
} from "./recordSettings";

const CITY = { record_mode: "off" as const, ibge_code: null, pec_url: null };

describe("recordModeLabel", () => {
  it("names the three modes and shows unknown values raw", () => {
    expect(recordModeLabel("off")).toBe("Desligado");
    expect(recordModeLabel("integrated")).toBe("Integrado ao PEC");
    expect(recordModeLabel("record")).toBe("Prontuário Rota Saúde");
    expect(recordModeLabel("hybrid")).toBe("hybrid");
    expect(recordModeLabel(undefined)).toBe("—");
  });
});

describe("modeChangeWarning", () => {
  it("names both modes and the consequence of the target mode", () => {
    const text = modeChangeWarning("off", "record");
    expect(text).toContain("de \"Desligado\" para \"Prontuário Rota Saúde\"");
    expect(text).toContain("o repasse da cidade passa a depender das fichas enviadas");
    expect(text).toContain("auditoria da plataforma");
    expect(modeChangeWarning("record", "off")).toContain("nenhuma ficha sai para o e-SUS");
  });
});

describe("pecUrlProblem refuses http, credentials and garbage", () => {
  it.each([
    [ "", false ],
    [ "https://pec.curitiba.pr.gov.br", false ],
    [ "  https://pec.curitiba.pr.gov.br:8443/  ", false ],
    [ "http://pec.curitiba.pr.gov.br", true ],
    [ "https://admin:segredo@pec.curitiba.pr.gov.br", true ],
    [ "https://admin@pec.curitiba.pr.gov.br", true ],
    [ "pec.curitiba.pr.gov.br", true ],
    [ "ftp://pec.curitiba.pr.gov.br", true ]
  ])("%j → problem=%s", (raw, problem) => {
    expect(pecUrlProblem(raw)).toBe(problem);
  });
});

describe("validateRecordSettings", () => {
  it("accepts empty fields and a 7-digit IBGE code", () => {
    expect(validateRecordSettings({ record_mode: "off", ibge_code: "", pec_url: "" })).toEqual([]);
    expect(validateRecordSettings({ record_mode: "off", ibge_code: " 4106902 ", pec_url: "" })).toEqual([]);
  });

  it("flags a short IBGE code and a bad PEC address", () => {
    expect(validateRecordSettings({ record_mode: "off", ibge_code: "41069", pec_url: "http://x" }))
      .toEqual([ "ibge_code", "pec_url" ]);
  });

  it("has a fixed message per field that never repeats what was typed", () => {
    expect(FIELD_PROBLEMS.ibge_code).toBe("o código IBGE tem 7 dígitos");
    expect(FIELD_PROBLEMS.pec_url).toBe("o endereço do PEC começa com https:// e não leva usuário nem senha");
  });
});

describe("recordSettingsPatch sends only what changed", () => {
  it("returns an empty patch when nothing changed", () => {
    expect(recordSettingsPatch(CITY, formFrom(CITY))).toEqual({});
  });

  it("does not resend the mode when it went back to the original", () => {
    const values = { ...formFrom(CITY), record_mode: "off", ibge_code: "4106902" };
    expect(recordSettingsPatch(CITY, values)).toEqual({ ibge_code: "4106902" });
  });

  it("trims and sends empty as null to clear a field", () => {
    const city = { record_mode: "integrated" as const, ibge_code: "4106902", pec_url: "https://pec.a.gov.br" };
    const values = { record_mode: "integrated", ibge_code: " 4106902 ", pec_url: "  " };
    expect(recordSettingsPatch(city, values)).toEqual({ pec_url: null });
  });

  it("recordSettingsPatch never sends the IBGE code of an unreachable city", () => {
    const city = { record_mode: "off" as const, ibge_code: null, pec_url: null, city_reachable: false };
    expect(ibgeLocked(city)).toBe(true);
    expect(ibgeLocked(CITY)).toBe(false);
    const values = { record_mode: "integrated", ibge_code: "4106902", pec_url: "https://pec.a.gov.br" };
    expect(recordSettingsPatch(city, values)).toEqual({ record_mode: "integrated", pec_url: "https://pec.a.gov.br" });
  });

  it("sends the new mode, and changesMode sees it", () => {
    const patch = recordSettingsPatch(CITY, { ...formFrom(CITY), record_mode: "record" });
    expect(patch).toEqual({ record_mode: "record" });
    expect(changesMode(patch)).toBe(true);
    expect(changesMode({ ibge_code: "4106902" })).toBe(false);
  });
});

describe("describeRecordSettingsError", () => {
  it.each([
    [ new ApiError(422, { error: "invalid_record_mode" }, "422"), "modo de prontuário inválido" ],
    [ new ApiError(422, { error: "invalid_ibge_code" }, "422"), "o código IBGE tem 7 dígitos" ],
    [ new ApiError(422, { error: "invalid_pec_url" }, "422"), "o endereço do PEC começa com https:// e não leva usuário nem senha" ],
    [ new ApiError(503, { error: "city_unreachable" }, "503"), "o banco da cidade não respondeu; o código IBGE não foi gravado — tente de novo" ],
    [ new Error("503 on /cities/c-cwb/record_settings"), "503 on /cities/c-cwb/record_settings" ],
    [ new ApiError(422, { error: "something_new" }, "422"), "dados inválidos" ],
    [ new ApiError(404, { error: "not_found" }, "404"), "cidade não encontrada" ],
    [ new Error("network down"), "network down" ]
  ])("%#: maps to %j", (err, message) => {
    expect(describeRecordSettingsError(err)).toBe(message);
  });
});

describe("switches", () => {
  it("names the known switches and shows unknown keys raw", () => {
    expect(featureLabel("ledi_export")).toBe("Exportação para o e-SUS (LEDI)");
    expect(featureLabel("cadsus_lookup")).toBe("Consulta ao CADSUS");
    expect(featureLabel("rnds_export")).toBe("rnds_export");
  });

  it("says who fixes each missing prerequisite and shows unknown codes raw", () => {
    expect(missingLabel("record_mode_off")).toBe("modo de prontuário desligado (aqui)");
    expect(missingLabel("pec_url_missing")).toBe("endereço do PEC (aqui)");
    expect(missingLabel("ibge_code_missing")).toBe("código IBGE (aqui)");
    expect(missingLabel("credential_missing:ledi")).toBe("credencial do PEC (a cidade cadastra em Integrações)");
    expect(missingLabel("credential_unauthorized:ledi")).toBe("credencial do PEC recusada (a cidade troca em Integrações)");
    expect(missingLabel("credential_missing:cadsus")).toBe("credencial do CADSUS (a cidade cadastra em Integrações)");
    expect(missingLabel("credential_unauthorized:cadsus")).toBe("credencial do CADSUS recusada (a cidade troca em Integrações)");
    expect(missingLabel("something_new")).toBe("something_new");
  });

  it("tells off, usable and enabled-but-blocked apart", () => {
    expect(featureState({ key: "k", enabled: false, usable: false, missing: [] }))
      .toEqual({ kind: "off", text: "desligado", tone: "neutral" });
    expect(featureState({ key: "k", enabled: true, usable: true, missing: [] }))
      .toEqual({ kind: "usable", text: "ligado · utilizável", tone: "ok" });
    expect(featureState({ key: "k", enabled: true, usable: false, missing: [ "pec_url_missing" ] }))
      .toEqual({ kind: "blocked", text: "ligado · não utilizável", tone: "warn" });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run src/lib/recordSettings.test.ts`
Expected: FAIL — `Failed to resolve import "./recordSettings"`.

- [ ] **Step 3: Write the implementation**

Create `src/lib/recordSettings.ts`:

```ts
// Regras da ficha da cidade (módulo 16, ADR 0028; contratos §2 e §4).
// Puras: nenhum React, nenhum fetch.
//
// O operador edita record_mode, ibge_code e pec_url. Os interruptores
// (features) são SÓ LEITURA no console: quem liga e desliga é o maintenance.
// O que falta para um interruptor funcionar (`missing`) vem do servidor; a
// tela só traduz os códigos.

import { ApiError, apiErrorCode, type RecordSettingsPatch } from "./api";
import type { CityDetail, CityFeatureState, RecordMode } from "./types";

export const RECORD_MODES: { value: RecordMode; label: string; hint: string }[] = [
  { value: "off", label: "Desligado", hint: "nenhuma ficha sai para o e-SUS" },
  {
    value: "integrated",
    label: "Integrado ao PEC",
    hint: "o PEC da cidade é o prontuário; o Rota Saúde envia só o que ele mesmo registra"
  },
  {
    value: "record",
    label: "Prontuário Rota Saúde",
    hint: "o Rota Saúde é o prontuário; o repasse da cidade passa a depender das fichas enviadas"
  }
];

export function recordModeLabel(mode: string | null | undefined): string {
  if (!mode) return "—";
  return RECORD_MODES.find((m) => m.value === mode)?.label ?? mode;
}

export function modeChangeWarning(from: string, to: string): string {
  const hint = RECORD_MODES.find((m) => m.value === to)?.hint;
  const consequence = hint ? ` Com o modo novo, ${hint}.` : "";
  return `Mudar o modo de prontuário de "${recordModeLabel(from)}" para "${recordModeLabel(to)}"?${consequence} ` +
    "A mudança fica registrada na auditoria da plataforma.";
}

export interface RecordSettingsValues {
  record_mode: string;
  ibge_code: string;
  pec_url: string;
}

export type RecordSettingsField = "ibge_code" | "pec_url";

// Mensagens fixas: nunca repetem o que foi digitado (a URL pode trazer senha).
export const FIELD_PROBLEMS: Record<RecordSettingsField, string> = {
  ibge_code: "o código IBGE tem 7 dígitos",
  pec_url: "o endereço do PEC começa com https:// e não leva usuário nem senha"
};

// city_reachable opcional: api antigo não manda; ausente = alcançável.
export type RecordFields = Pick<CityDetail, "record_mode" | "ibge_code" | "pec_url"> & { city_reachable?: boolean };

// Banco da cidade inalcançável: o api manda ibge_code null, que NÃO é "vazio".
export function ibgeLocked(city: RecordFields): boolean {
  return city.city_reachable === false;
}

export const IBGE_LOCKED_TEXT =
  "o banco da cidade não respondeu: o código IBGE não pode ser lido nem editado agora";

export function formFrom(city: RecordFields): RecordSettingsValues {
  return { record_mode: city.record_mode, ibge_code: city.ibge_code ?? "", pec_url: city.pec_url ?? "" };
}

const IBGE = /^\d{7}$/;

export function pecUrlProblem(raw: string): boolean {
  const value = raw.trim();
  if (value === "") return false;
  let url: URL;
  try {
    url = new URL(value);
  } catch {
    return true;
  }
  return url.protocol !== "https:" || url.username !== "" || url.password !== "";
}

export function validateRecordSettings(values: RecordSettingsValues): RecordSettingsField[] {
  const bad: RecordSettingsField[] = [];
  const ibge = values.ibge_code.trim();
  if (ibge !== "" && !IBGE.test(ibge)) bad.push("ibge_code");
  if (pecUrlProblem(values.pec_url)) bad.push("pec_url");
  return bad;
}

function orNull(raw: string): string | null {
  const value = raw.trim();
  return value === "" ? null : value;
}

// Só o que mudou. O modo igual ao atual nunca vai, e vazio vira null (limpa).
// Com a cidade inalcançável, ibge_code nunca vai.
export function recordSettingsPatch(city: RecordFields, values: RecordSettingsValues): RecordSettingsPatch {
  const patch: RecordSettingsPatch = {};
  if (values.record_mode !== city.record_mode) patch.record_mode = values.record_mode as RecordMode;
  const ibge = orNull(values.ibge_code);
  if (!ibgeLocked(city) && ibge !== (city.ibge_code ?? null)) patch.ibge_code = ibge;
  const pec = orNull(values.pec_url);
  if (pec !== (city.pec_url ?? null)) patch.pec_url = pec;
  return patch;
}

export function changesMode(patch: RecordSettingsPatch): boolean {
  return patch.record_mode !== undefined;
}

const ERROR_MESSAGES: Record<string, string> = {
  invalid_record_mode: "modo de prontuário inválido",
  invalid_ibge_code: FIELD_PROBLEMS.ibge_code,
  invalid_pec_url: FIELD_PROBLEMS.pec_url
};

// 503 city_unreachable: o IBGE grava no city_profile, no banco da cidade.
const CITY_UNREACHABLE = "o banco da cidade não respondeu; o código IBGE não foi gravado — tente de novo";

export function describeRecordSettingsError(err: unknown): string {
  if (err instanceof ApiError) {
    if (err.status === 404) return "cidade não encontrada";
    if (err.status === 422) return ERROR_MESSAGES[apiErrorCode(err) ?? ""] ?? "dados inválidos";
    if (err.status === 503 && apiErrorCode(err) === "city_unreachable") return CITY_UNREACHABLE;
  }
  return (err as Error).message;
}

const FEATURE_LABELS: Record<string, string> = {
  ledi_export: "Exportação para o e-SUS (LEDI)",
  cadsus_lookup: "Consulta ao CADSUS"
};

export function featureLabel(key: string): string {
  return FEATURE_LABELS[key] ?? key;
}

// "(aqui)": o operador resolve nesta ficha. Credencial é da cidade (dashboard,
// tela Integrações, municipal_admin com step-up).
const MISSING_LABELS: Record<string, string> = {
  record_mode_off: "modo de prontuário desligado (aqui)",
  pec_url_missing: "endereço do PEC (aqui)",
  ibge_code_missing: "código IBGE (aqui)",
  "credential_missing:ledi": "credencial do PEC (a cidade cadastra em Integrações)",
  "credential_unauthorized:ledi": "credencial do PEC recusada (a cidade troca em Integrações)",
  "credential_missing:cadsus": "credencial do CADSUS (a cidade cadastra em Integrações)",
  "credential_unauthorized:cadsus": "credencial do CADSUS recusada (a cidade troca em Integrações)"
};

export function missingLabel(code: string): string {
  return MISSING_LABELS[code] ?? code;
}

export interface FeatureStateView {
  kind: "off" | "usable" | "blocked";
  text: string;
  tone: "neutral" | "ok" | "warn";
}

export function featureState(f: CityFeatureState): FeatureStateView {
  if (!f.enabled) return { kind: "off", text: "desligado", tone: "neutral" };
  if (f.usable) return { kind: "usable", text: "ligado · utilizável", tone: "ok" };
  return { kind: "blocked", text: "ligado · não utilizável", tone: "warn" };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run src/lib/recordSettings.test.ts && npx tsc --noEmit`
Expected: PASS e `tsc` sem erro.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add src/lib/recordSettings.ts src/lib/recordSettings.test.ts
/opt/homebrew/bin/git commit -m "$(cat <<'EOF'
feat: add record mode, IBGE, PEC and switch rules for the city record

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 3: Ficha da cidade com os interruptores (só leitura)

**Files:**
- Modify: `src/hooks/useCities.ts`
- Modify: `src/modules/Cities.tsx` (arquivo inteiro abaixo)
- Create: `src/modules/cities/CityRecord.tsx`, `src/modules/cities/CityFeatures.tsx`
- Test: `src/modules/cities/CityRecord.test.tsx`

**Interfaces:**
- Consumes: `getCity` (Task 1); `CityRow`, `CityFeatureState` (Task 1); `recordModeLabel`, `featureLabel`, `featureState`, `missingLabel` (Task 2).
- Produces:
  - `cityKey(id: string): readonly ["city", string]` e `useCity(id: string)` em `src/hooks/useCities.ts`;
  - `CityRecord({ city: CityRow; onBack: () => void })`;
  - `CityFeatures({ features: CityFeatureState[] | undefined })`;
  - botões com nome acessível `Ficha de <nome>` e `Voltar para Cidades`.

- [ ] **Step 1: Write the failing test**

Create `src/modules/cities/CityRecord.test.tsx`:

```tsx
import { describe, it, expect, vi, afterEach, beforeEach } from "vitest";
import { render, screen, cleanup, fireEvent, within } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";

vi.mock("../../lib/api", async (importOriginal) => {
  const actual = await importOriginal<typeof import("../../lib/api")>();
  return { ...actual, listCities: vi.fn(), getCity: vi.fn(), createCityGrant: vi.fn() };
});

import { listCities, getCity, ApiError } from "../../lib/api";
import type { CityDetail, CityRow } from "../../lib/types";
import { Cities } from "../Cities";

const citiesMock = vi.mocked(listCities);
const cityMock = vi.mocked(getCity);

const CITIES: CityRow[] = [
  { id: "c-cwb", slug: "curitiba", name: "Curitiba", uf: "PR", status: "active", schema_version: "1",
    created_at: "2026-09-01T00:00:00Z", record_mode: "off" },
  { id: "c-mga", slug: "maringa", name: "Maringá", uf: "PR", status: "active", schema_version: "1",
    created_at: "2026-09-02T00:00:00Z", record_mode: "integrated" }
];

const DETAIL: CityDetail = {
  id: "c-cwb", slug: "curitiba", name: "Curitiba", uf: "PR", status: "active", schema_version: "1",
  time_zone: "America/Sao_Paulo", created_at: "2026-09-01T00:00:00Z",
  record_mode: "off", ibge_code: null, pec_url: null, city_reachable: true,
  features: [
    { key: "ledi_export", enabled: false, usable: false,
      missing: [ "record_mode_off", "pec_url_missing", "ibge_code_missing", "credential_missing:ledi" ] },
    { key: "cadsus_lookup", enabled: true, usable: true, missing: [] }
  ]
};

function renderScreen() {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  return render(<QueryClientProvider client={client}><Cities /></QueryClientProvider>);
}

async function openRecord() {
  fireEvent.click(await screen.findByRole("button", { name: "Ficha de Curitiba" }));
}

describe("City record (read-only switches)", () => {
  beforeEach(() => {
    citiesMock.mockReset();
    cityMock.mockReset();
    citiesMock.mockResolvedValue(CITIES);
    cityMock.mockResolvedValue(DETAIL);
  });
  afterEach(() => { cleanup(); });

  it("lists the record mode of each city", async () => {
    renderScreen();
    expect(await screen.findByText("Desligado")).toBeTruthy();
    expect(screen.getByText("Integrado ao PEC")).toBeTruthy();
    expect(screen.getByText("Prontuário")).toBeTruthy();
  });

  it("opens the record of the chosen city with its switches, read-only", async () => {
    renderScreen();
    await openRecord();

    expect(await screen.findByRole("heading", { name: "Curitiba · PR" })).toBeTruthy();
    expect(cityMock).toHaveBeenCalledWith("c-cwb");

    const ledi = await screen.findByRole("listitem", { name: "Exportação para o e-SUS (LEDI)" });
    expect(ledi.textContent).toContain("desligado");
    expect(ledi.textContent).toContain(
      "falta: modo de prontuário desligado (aqui); endereço do PEC (aqui); código IBGE (aqui); " +
      "credencial do PEC (a cidade cadastra em Integrações)"
    );
    const cadsus = screen.getByRole("listitem", { name: "Consulta ao CADSUS" });
    expect(cadsus.textContent).toContain("ligado · utilizável");
    expect(cadsus.textContent).not.toContain("falta:");

    expect(screen.getByText(/Quem liga e desliga os interruptores é o mantenedor, no maintenance/)).toBeTruthy();
    expect(screen.queryByRole("checkbox")).toBeNull();
    expect(screen.queryByRole("switch")).toBeNull();
    expect(screen.queryByRole("button", { name: /ligar|desligar/i })).toBeNull();
  });

  it("enabled but not usable says what is still missing", async () => {
    cityMock.mockResolvedValue({
      ...DETAIL,
      features: [ { key: "ledi_export", enabled: true, usable: false, missing: [ "credential_unauthorized:ledi" ] } ]
    });
    renderScreen();
    await openRecord();

    const ledi = await screen.findByRole("listitem", { name: "Exportação para o e-SUS (LEDI)" });
    expect(ledi.textContent).toContain("ligado · não utilizável");
    expect(ledi.textContent).toContain("credencial do PEC recusada (a cidade troca em Integrações)");
  });

  it("unknown switch and unknown missing code appear raw", async () => {
    cityMock.mockResolvedValue({
      ...DETAIL,
      features: [ { key: "rnds_export", enabled: false, usable: false, missing: [ "something_new" ] } ]
    });
    renderScreen();
    await openRecord();

    const item = await screen.findByRole("listitem", { name: "rnds_export" });
    expect(item.textContent).toContain("something_new");
  });

  it("an api without features shows an explained empty state", async () => {
    cityMock.mockResolvedValue({ ...DETAIL, features: [] });
    renderScreen();
    await openRecord();

    expect(await screen.findByText("Nenhum interruptor informado.")).toBeTruthy();
  });

  it("goes back to the list", async () => {
    renderScreen();
    await openRecord();
    fireEvent.click(await screen.findByRole("button", { name: "Voltar para Cidades" }));

    expect(await screen.findByRole("button", { name: "Ficha de Maringá" })).toBeTruthy();
  });

  it("shows the error state when the record fails to load", async () => {
    cityMock.mockRejectedValue(new ApiError(500, "", "500 on /cities/c-cwb"));
    renderScreen();
    await openRecord();

    expect(await screen.findByText("Falha ao carregar")).toBeTruthy();
    const table = screen.queryByRole("table");
    expect(table).toBeNull();
    expect(within(document.body).getByText("500 on /cities/c-cwb")).toBeTruthy();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run src/modules/cities/CityRecord.test.tsx`
Expected: FAIL — não há coluna "Prontuário" nem botão "Ficha de Curitiba".

- [ ] **Step 3: Hook**

Substitua `src/hooks/useCities.ts` por:

```ts
import { useQuery } from "@tanstack/react-query";
import { getCity, listCities } from "../lib/api";

// Catálogo da plataforma: não depende de período nem de cidade ativa.
export function useCities() {
  return useQuery({ queryKey: [ "cities" ], queryFn: listCities, staleTime: 30_000 });
}

// Ficha da cidade (módulo 16). Chave "city", separada de "cities": invalidar
// a lista não derruba a ficha que acabou de receber a resposta do PATCH.
export const cityKey = (id: string) => [ "city", id ] as const;

export function useCity(id: string) {
  return useQuery({ queryKey: cityKey(id), queryFn: () => getCity(id) });
}
```

- [ ] **Step 4: Switches panel**

Create `src/modules/cities/CityFeatures.tsx`:

```tsx
// Interruptores da cidade (módulo 16, ADR 0028) — SÓ LEITURA. Quem liga e
// desliga é o mantenedor, no maintenance (única porta de escrita). Aqui o
// operador vê o estado e o que falta, e resolve o que é dele na própria ficha
// (modo, IBGE, PEC). O `missing` vem do servidor.

import type { CityFeatureState } from "../../lib/types";
import { featureLabel, featureState, missingLabel } from "../../lib/recordSettings";
import { Tag } from "../../components/Tag";
import { EmptyState } from "../../components/EmptyState";

export function CityFeatures({ features }: { features: CityFeatureState[] | undefined }) {
  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 12 }}>
      <p style={{ fontSize: 12, color: "var(--ink3)", margin: 0 }}>
        Quem liga e desliga os interruptores é o mantenedor, no maintenance. O console só mostra o
        estado e o que falta para cada funcionalidade funcionar.
      </p>
      {!features || features.length === 0 ? (
        <EmptyState title="Nenhum interruptor informado." sub="o api não devolveu interruptores para esta cidade" />
      ) : (
        <ul style={{ listStyle: "none", margin: 0, padding: 0, display: "flex", flexDirection: "column", gap: 10 }}>
          {features.map((f) => {
            const state = featureState(f);
            const missing = f.missing ?? [];
            return (
              <li
                key={f.key}
                aria-label={featureLabel(f.key)}
                style={{ display: "flex", flexDirection: "column", gap: 4, paddingBottom: 10, borderBottom: "1px solid var(--rule)" }}
              >
                <div style={{ display: "flex", alignItems: "center", gap: 8, flexWrap: "wrap" }}>
                  <span style={{ fontSize: 13, fontWeight: 600, color: "var(--ink)" }}>{featureLabel(f.key)}</span>
                  <span className="mono" style={{ fontSize: 10.5, color: "var(--ink3)" }}>{f.key}</span>
                  <Tag tone={state.tone}>{state.text}</Tag>
                </div>
                {missing.length > 0 && (
                  <div style={{ fontSize: 12, color: "var(--ink2)" }}>
                    {`falta: ${missing.map(missingLabel).join("; ")}`}
                  </div>
                )}
              </li>
            );
          })}
        </ul>
      )}
    </div>
  );
}
```

- [ ] **Step 5: City record**

Create `src/modules/cities/CityRecord.tsx`:

```tsx
// Ficha da cidade (módulo 16, ADR 0028). Abre de "Cidades" pelo botão
// "Ficha". Lê GET /cities/:id. O cabeçalho usa nome e UF da linha da lista,
// que já está carregada (o show devolve os mesmos campos).

import { useCity } from "../../hooks/useCities";
import type { CityRow } from "../../lib/types";
import { PageHeader } from "../../components/PageHeader";
import { Panel } from "../../components/Panel";
import { ErrorState } from "../../components/ErrorState";
import { Skeleton } from "../../components/Skeleton";
import { CityFeatures } from "./CityFeatures";

interface Props {
  city: CityRow;
  onBack: () => void;
}

export function CityRecord({ city, onBack }: Props) {
  const { data, isLoading, isError, error, refetch } = useCity(city.id);

  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 16 }}>
      <div>
        <button type="button" aria-label="Voltar para Cidades" onClick={onBack} style={backBtn}>
          ← Cidades
        </button>
      </div>
      <PageHeader title={`${city.name}${city.uf ? ` · ${city.uf}` : ""}`} sub="ficha da cidade" />
      {isLoading && <Skeleton rows={4} />}
      {isError && <ErrorState message={(error as Error)?.message || "Erro"} onRetry={() => refetch()} />}
      {data && (
        <Panel title="Interruptores" sub="só leitura · quem liga é o maintenance">
          <CityFeatures features={data.features} />
        </Panel>
      )}
    </div>
  );
}

const backBtn: React.CSSProperties = {
  padding: "5px 10px", borderRadius: 8, border: "1px solid var(--rule2)",
  background: "var(--panel)", color: "var(--ink)", fontSize: 12, cursor: "pointer"
};
```

- [ ] **Step 6: Cities list**

Substitua `src/modules/Cities.tsx` por:

```tsx
// Cidades do catálogo (operador). Sem métricas cross-tenant: o console mostra
// estado de provisionamento e leva o operador para dentro da cidade (Plano 6,
// spec §5). Módulo 16: "Ficha" abre a ficha da cidade (modo de prontuário,
// IBGE, PEC e interruptores), por estado local — o console não tem roteador.
import { useState } from "react";
import { useCities } from "../hooks/useCities";
import { enterCity, sortedForDisplay } from "../lib/cities";
import { recordModeLabel } from "../lib/recordSettings";
import { PageHeader } from "../components/PageHeader";
import { DataTable, type Column } from "../components/DataTable";
import { StatusDot } from "../components/StatusDot";
import { EmptyState } from "../components/EmptyState";
import { ErrorState } from "../components/ErrorState";
import { Skeleton } from "../components/Skeleton";
import { fmtTime } from "../lib/format";
import type { CityRow } from "../lib/types";
import { CityRecord } from "./cities/CityRecord";

export function Cities() {
  const { data, isLoading, isError, error, refetch } = useCities();
  const [ busySlug, setBusySlug ] = useState<string | null>(null);
  const [ failure, setFailure ] = useState<string | null>(null);
  const [ openId, setOpenId ] = useState<string | null>(null);

  // Pelo id: quando a lista recarrega, a ficha aberta recebe a linha nova.
  const open = openId ? data?.find((c) => c.id === openId) : undefined;
  if (open) return <CityRecord city={open} onBack={() => setOpenId(null)} />;

  async function onEnter(city: CityRow) {
    setBusySlug(city.slug);
    setFailure(null);
    try {
      await enterCity(city.slug, (url) => { window.location.href = url; });
    } catch {
      setFailure(`Não foi possível entrar em ${city.name}. O grant vale 60 s — tente de novo.`);
    } finally {
      setBusySlug(null);
    }
  }

  const cols: Column<CityRow>[] = [
    { label: "Cidade", w: "1.4fr", render: (c) => <span>{c.name}{c.uf ? ` · ${c.uf}` : ""}</span> },
    { label: "Slug", w: "1fr", render: (c) => <span className="mono">{c.slug}</span> },
    { label: "Status", w: "0.9fr", render: (c) => (
        <span style={{ display: "inline-flex", alignItems: "center", gap: 6 }}>
          <StatusDot level={c.status === "active" ? "ok" : "warn"} /> {c.status}
        </span>
      ) },
    { label: "Prontuário", w: "1.1fr", render: (c) => recordModeLabel(c.record_mode) },
    { label: "Schema", w: "0.7fr", render: (c) => <span className="mono">{c.schema_version ?? "—"}</span> },
    { label: "Fuso", w: "1fr", render: (c) => <span className="mono">{c.time_zone ?? "—"}</span> },
    { label: "Criada", w: "1fr", render: (c) => c.created_at ? fmtTime(c.created_at) : "—" },
    { label: "", w: "1.2fr", align: "right", render: (c) => (
        <span style={{ display: "inline-flex", gap: 6 }}>
          <button
            type="button"
            aria-label={`Ficha de ${c.name}`}
            onClick={() => setOpenId(c.id)}
            style={enterBtn}
          >
            Ficha
          </button>
          <button
            type="button"
            disabled={c.status !== "active" || busySlug === c.slug}
            onClick={() => { void onEnter(c); }}
            style={enterBtn}
          >
            {busySlug === c.slug ? "Entrando…" : "Entrar"}
          </button>
        </span>
      ) }
  ];

  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 16 }}>
      <PageHeader title="Cidades" sub="catálogo da plataforma" />
      {isLoading && <Skeleton rows={6} />}
      {isError && <ErrorState message={(error as Error)?.message || "Erro"} onRetry={() => refetch()} />}
      {failure && <p role="alert" style={{ color: "var(--down)", fontSize: 12, margin: 0 }}>{failure}</p>}
      {data && data.length === 0 && (
        <EmptyState title="Nenhuma cidade provisionada" sub="use Setup → Provisionar cidade para criar a primeira" />
      )}
      {data && data.length > 0 && (
        <DataTable<CityRow> cols={cols} rows={sortedForDisplay(data)} rowKey={(c) => c.id} />
      )}
    </div>
  );
}

const enterBtn: React.CSSProperties = {
  padding: "5px 10px", borderRadius: 8, border: "1px solid var(--rule2)",
  background: "var(--panel)", color: "var(--ink)", fontSize: 12, cursor: "pointer"
};
```

- [ ] **Step 7: Run tests**

Run: `npx vitest run src/modules/cities/CityRecord.test.tsx src/lib/cities.test.ts src/modules/setup/RegisterChannel.test.tsx && npx tsc --noEmit`
Expected: PASS (as telas que já usavam `useCities` continuam verdes) e `tsc` sem erro.

- [ ] **Step 8: Commit**

```bash
/opt/homebrew/bin/git add src/hooks/useCities.ts src/modules/Cities.tsx src/modules/cities/CityRecord.tsx src/modules/cities/CityFeatures.tsx src/modules/cities/CityRecord.test.tsx
/opt/homebrew/bin/git commit -m "$(cat <<'EOF'
feat: open a city record from the console with its read-only switches

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 4: Edição do modo, do IBGE e do PEC com confirmação

**Files:**
- Create: `src/modules/cities/RecordSettingsForm.tsx`
- Modify: `src/modules/cities/CityRecord.tsx` (arquivo inteiro abaixo), `README.md`
- Test: `src/modules/cities/RecordSettingsForm.test.tsx`

**Interfaces:**
- Consumes: `updateCityRecordSettings`, `RecordSettingsPatch` (Task 1); `CityDetail` (Task 1); `RECORD_MODES`, `FIELD_PROBLEMS`, `formFrom`, `validateRecordSettings`, `recordSettingsPatch`, `changesMode`, `modeChangeWarning`, `describeRecordSettingsError`, `RecordSettingsValues`, `RecordSettingsField` (Task 2); `cityKey`, `useCity` (Task 3).
- Consumes também: `ibgeLocked`, `IBGE_LOCKED_TEXT` (Task 2).
- Produces: `RecordSettingsForm({ city: CityDetail; onSaved: (city: CityDetail) => void })`; campos rotulados `Modo de prontuário`, `Código IBGE · 7 dígitos`, `Endereço do PEC · https://, sem usuário nem senha`; botões `Salvar`, `Confirmar mudança`, `Cancelar`; grupo `Confirmar mudança de modo`.

- [ ] **Step 1: Write the failing test**

Create `src/modules/cities/RecordSettingsForm.test.tsx`:

```tsx
import { describe, it, expect, vi, afterEach, beforeEach } from "vitest";
import { render, screen, cleanup, fireEvent } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";

vi.mock("../../lib/api", async (importOriginal) => {
  const actual = await importOriginal<typeof import("../../lib/api")>();
  return {
    ...actual, listCities: vi.fn(), getCity: vi.fn(), updateCityRecordSettings: vi.fn(), createCityGrant: vi.fn()
  };
});

import { listCities, getCity, updateCityRecordSettings, ApiError } from "../../lib/api";
import type { CityDetail, CityRow } from "../../lib/types";
import { FIELD_PROBLEMS, IBGE_LOCKED_TEXT } from "../../lib/recordSettings";
import { Cities } from "../Cities";

const citiesMock = vi.mocked(listCities);
const cityMock = vi.mocked(getCity);
const updateMock = vi.mocked(updateCityRecordSettings);

const CITIES: CityRow[] = [
  { id: "c-cwb", slug: "curitiba", name: "Curitiba", uf: "PR", status: "active", schema_version: "1",
    created_at: "2026-09-01T00:00:00Z", record_mode: "off" }
];

const DETAIL: CityDetail = {
  id: "c-cwb", slug: "curitiba", name: "Curitiba", uf: "PR", status: "active", schema_version: "1",
  time_zone: "America/Sao_Paulo", created_at: "2026-09-01T00:00:00Z",
  record_mode: "off", ibge_code: null, pec_url: null, city_reachable: true,
  features: [
    { key: "ledi_export", enabled: false, usable: false,
      missing: [ "record_mode_off", "pec_url_missing", "ibge_code_missing", "credential_missing:ledi" ] },
    { key: "cadsus_lookup", enabled: false, usable: false, missing: [ "credential_missing:cadsus" ] }
  ]
};

function renderScreen() {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  return render(<QueryClientProvider client={client}><Cities /></QueryClientProvider>);
}

function field(label: RegExp) {
  return screen.getByLabelText(label) as HTMLInputElement | HTMLSelectElement;
}

async function openRecord() {
  fireEvent.click(await screen.findByRole("button", { name: "Ficha de Curitiba" }));
  await screen.findByLabelText(/^Modo de prontuário/);
}

function save() {
  fireEvent.click(screen.getByRole("button", { name: "Salvar" }));
}

describe("RecordSettingsForm", () => {
  beforeEach(() => {
    citiesMock.mockReset();
    cityMock.mockReset();
    updateMock.mockReset();
    citiesMock.mockResolvedValue(CITIES);
    cityMock.mockResolvedValue(DETAIL);
  });
  afterEach(() => { cleanup(); });

  it("opens with the current values", async () => {
    cityMock.mockResolvedValue({ ...DETAIL, record_mode: "integrated", ibge_code: "4106902", pec_url: "https://pec.a.gov.br" });
    renderScreen();
    await openRecord();

    expect(field(/^Modo de prontuário/).value).toBe("integrated");
    expect(field(/^Código IBGE/).value).toBe("4106902");
    expect(field(/^Endereço do PEC/).value).toBe("https://pec.a.gov.br");
  });

  it("changing only the IBGE code saves without confirmation and sends only that field", async () => {
    updateMock.mockResolvedValue({ ...DETAIL, ibge_code: "4106902" });
    renderScreen();
    await openRecord();

    fireEvent.change(field(/^Código IBGE/), { target: { value: "4106902" } });
    save();

    expect(await screen.findByText("Salvo.")).toBeTruthy();
    expect(updateMock).toHaveBeenCalledTimes(1);
    expect(updateMock).toHaveBeenCalledWith("c-cwb", { ibge_code: "4106902" });
    expect(screen.queryByRole("group", { name: "Confirmar mudança de modo" })).toBeNull();
  });

  it("changing the mode asks for confirmation and only sends after confirming", async () => {
    updateMock.mockResolvedValue({ ...DETAIL, record_mode: "record" });
    renderScreen();
    await openRecord();

    fireEvent.change(field(/^Modo de prontuário/), { target: { value: "record" } });
    save();

    const confirm = await screen.findByRole("group", { name: "Confirmar mudança de modo" });
    expect(confirm.textContent).toContain("de \"Desligado\" para \"Prontuário Rota Saúde\"");
    expect(confirm.textContent).toContain("o repasse da cidade passa a depender das fichas enviadas");
    expect(updateMock).not.toHaveBeenCalled();

    fireEvent.click(screen.getByRole("button", { name: "Confirmar mudança" }));

    expect(await screen.findByText("Salvo.")).toBeTruthy();
    expect(updateMock).toHaveBeenCalledWith("c-cwb", { record_mode: "record" });
    expect(screen.queryByRole("group", { name: "Confirmar mudança de modo" })).toBeNull();
  });

  it("cancel sends nothing and keeps the chosen mode", async () => {
    renderScreen();
    await openRecord();

    fireEvent.change(field(/^Modo de prontuário/), { target: { value: "integrated" } });
    save();
    await screen.findByRole("group", { name: "Confirmar mudança de modo" });
    fireEvent.click(screen.getByRole("button", { name: "Cancelar" }));

    expect(screen.queryByRole("group", { name: "Confirmar mudança de modo" })).toBeNull();
    expect(updateMock).not.toHaveBeenCalled();
    expect(field(/^Modo de prontuário/).value).toBe("integrated");
  });

  it("mode back to the original: no confirmation and nothing sent", async () => {
    renderScreen();
    await openRecord();

    fireEvent.change(field(/^Modo de prontuário/), { target: { value: "record" } });
    fireEvent.change(field(/^Modo de prontuário/), { target: { value: "off" } });
    save();

    expect(await screen.findByText("Nada mudou.")).toBeTruthy();
    expect(screen.queryByRole("group", { name: "Confirmar mudança de modo" })).toBeNull();
    expect(updateMock).not.toHaveBeenCalled();
  });

  it("PEC with user and password is refused before the API and never echoed", async () => {
    const { container } = renderScreen();
    await openRecord();

    fireEvent.change(field(/^Endereço do PEC/), { target: { value: "https://admin:segredo@pec.curitiba.pr.gov.br" } });
    save();

    expect(await screen.findByText(FIELD_PROBLEMS.pec_url)).toBeTruthy();
    expect(updateMock).not.toHaveBeenCalled();
    expect(container.textContent).not.toContain("segredo");
  });

  it("a short IBGE code is refused before the API", async () => {
    renderScreen();
    await openRecord();

    fireEvent.change(field(/^Código IBGE/), { target: { value: "41069" } });
    save();

    expect(await screen.findByText(FIELD_PROBLEMS.ibge_code)).toBeTruthy();
    expect(updateMock).not.toHaveBeenCalled();
  });

  it.each([
    [ "city_unreachable", new ApiError(503, { error: "city_unreachable" }, "503"), "o banco da cidade não respondeu; o código IBGE não foi gravado — tente de novo" ],
    [ "invalid_ibge_code", new ApiError(422, { error: "invalid_ibge_code" }, "422"), "o código IBGE tem 7 dígitos" ],
    [ "invalid_pec_url", new ApiError(422, { error: "invalid_pec_url" }, "422"), FIELD_PROBLEMS.pec_url ],
    [ "invalid_record_mode", new ApiError(422, { error: "invalid_record_mode" }, "422"), "modo de prontuário inválido" ],
    [ "404", new ApiError(404, { error: "not_found" }, "404"), "cidade não encontrada" ],
    [ "network", new Error("network down"), "network down" ]
  ])("shows the refusal (%s) and keeps the form", async (_name, err, message) => {
    updateMock.mockRejectedValue(err);
    renderScreen();
    await openRecord();

    fireEvent.change(field(/^Código IBGE/), { target: { value: "4106902" } });
    save();

    expect((await screen.findByRole("alert")).textContent).toBe(message);
    expect(screen.queryByText("Salvo.")).toBeNull();
    expect(field(/^Código IBGE/).value).toBe("4106902");
  });

  it("unreachable city: IBGE locked, mode and PEC still save without it", async () => {
    cityMock.mockResolvedValue({ ...DETAIL, ibge_code: null, city_reachable: false });
    updateMock.mockResolvedValue({ ...DETAIL, ibge_code: null, city_reachable: false, pec_url: "https://pec.a.gov.br" });
    renderScreen();
    await openRecord();

    const ibge = field(/^Código IBGE/) as HTMLInputElement;
    expect(ibge.disabled).toBe(true);
    expect(screen.getByText(IBGE_LOCKED_TEXT)).toBeTruthy();

    fireEvent.change(field(/^Endereço do PEC/), { target: { value: "https://pec.a.gov.br" } });
    save();

    expect(await screen.findByText("Salvo.")).toBeTruthy();
    expect(updateMock).toHaveBeenCalledWith("c-cwb", { pec_url: "https://pec.a.gov.br" });
  });

  it("after saving, the switches follow the server and the city list is refreshed", async () => {
    citiesMock.mockReset();
    citiesMock
      .mockResolvedValueOnce(CITIES)
      .mockResolvedValue([ { ...CITIES[0], record_mode: "record" } ]);
    updateMock.mockResolvedValue({
      ...DETAIL,
      record_mode: "record", ibge_code: "4106902", pec_url: "https://pec.curitiba.pr.gov.br",
      features: [
        { key: "ledi_export", enabled: false, usable: false, missing: [ "credential_missing:ledi" ] },
        DETAIL.features[1]
      ]
    });
    renderScreen();
    await openRecord();

    fireEvent.change(field(/^Modo de prontuário/), { target: { value: "record" } });
    fireEvent.change(field(/^Código IBGE/), { target: { value: "4106902" } });
    fireEvent.change(field(/^Endereço do PEC/), { target: { value: "https://pec.curitiba.pr.gov.br" } });
    save();
    fireEvent.click(await screen.findByRole("button", { name: "Confirmar mudança" }));
    await screen.findByText("Salvo.");

    expect(updateMock).toHaveBeenCalledWith("c-cwb", {
      record_mode: "record", ibge_code: "4106902", pec_url: "https://pec.curitiba.pr.gov.br"
    });
    const ledi = screen.getByRole("listitem", { name: "Exportação para o e-SUS (LEDI)" });
    expect(ledi.textContent).not.toContain("endereço do PEC");
    expect(ledi.textContent).not.toContain("modo de prontuário desligado");
    expect(ledi.textContent).toContain("credencial do PEC (a cidade cadastra em Integrações)");

    fireEvent.click(screen.getByRole("button", { name: "Voltar para Cidades" }));
    expect(await screen.findByText("Prontuário Rota Saúde")).toBeTruthy();
    expect(citiesMock.mock.calls.length).toBeGreaterThanOrEqual(2);
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run src/modules/cities/RecordSettingsForm.test.tsx`
Expected: FAIL — não há campo "Modo de prontuário" na ficha.

- [ ] **Step 3: Form**

Create `src/modules/cities/RecordSettingsForm.tsx`:

```tsx
// Modo de prontuário, código IBGE e endereço do PEC da cidade (módulo 16,
// ADR 0028; contratos §4.1). Decisão de contrato, do operador, auditada no api.
//
// - Mudar o modo sempre pede confirmação antes do PATCH; IBGE e PEC não.
// - O PATCH leva só o que mudou; vazio vai como null (limpa o campo).
// - A URL do PEC é conferida antes de sair (https, sem usuário nem senha) e
//   nenhuma mensagem repete o que foi digitado.
// - Banco da cidade inalcançável (city_reachable: false): o IBGE fica
//   bloqueado; o null que veio não é "vazio".

import { useState, type FormEvent, type ReactNode } from "react";
import { updateCityRecordSettings, type RecordSettingsPatch } from "../../lib/api";
import type { CityDetail } from "../../lib/types";
import {
  FIELD_PROBLEMS, IBGE_LOCKED_TEXT, RECORD_MODES, changesMode, describeRecordSettingsError, formFrom, ibgeLocked,
  modeChangeWarning, recordSettingsPatch, validateRecordSettings, type RecordSettingsField, type RecordSettingsValues
} from "../../lib/recordSettings";

interface Props {
  city: CityDetail;
  onSaved: (city: CityDetail) => void;
}

export function RecordSettingsForm({ city, onSaved }: Props) {
  const [ values, setValues ] = useState<RecordSettingsValues>(() => formFrom(city));
  const [ bad, setBad ] = useState<RecordSettingsField[]>([]);
  const [ pending, setPending ] = useState<RecordSettingsPatch | null>(null);
  const [ busy, setBusy ] = useState(false);
  const [ error, setError ] = useState<string | null>(null);
  const [ notice, setNotice ] = useState<string | null>(null);

  function set<K extends keyof RecordSettingsValues>(key: K, value: string) {
    setValues((v) => ({ ...v, [ key ]: value }));
    setNotice(null);
  }

  function submit(e: FormEvent) {
    e.preventDefault();
    setError(null);
    setNotice(null);
    const problems = validateRecordSettings(values);
    setBad(problems);
    if (problems.length > 0) return;
    const patch = recordSettingsPatch(city, values);
    if (Object.keys(patch).length === 0) {
      setNotice("Nada mudou.");
      return;
    }
    if (changesMode(patch)) {
      setPending(patch);
      return;
    }
    void send(patch);
  }

  async function send(patch: RecordSettingsPatch) {
    setBusy(true);
    try {
      const updated = await updateCityRecordSettings(city.id, patch);
      setValues(formFrom(updated));
      setNotice("Salvo.");
      onSaved(updated);
    } catch (err) {
      setError(describeRecordSettingsError(err));
    } finally {
      setPending(null);
      setBusy(false);
    }
  }

  const locked = busy || pending !== null;
  const ibgeOff = ibgeLocked(city);
  const hint = RECORD_MODES.find((m) => m.value === values.record_mode)?.hint;

  return (
    <form onSubmit={submit} style={{ display: "flex", flexDirection: "column", gap: 12, maxWidth: 520 }}>
      <Field label="Modo de prontuário">
        <select
          value={values.record_mode}
          onChange={(e) => set("record_mode", e.target.value)}
          disabled={locked}
          style={inputStyle}
        >
          {RECORD_MODES.map((m) => <option key={m.value} value={m.value}>{m.label}</option>)}
        </select>
      </Field>
      {hint && <p style={{ fontSize: 12, color: "var(--ink3)", margin: 0 }}>{hint}</p>}

      <Field
        label="Código IBGE"
        hint="7 dígitos"
        error={ibgeOff ? IBGE_LOCKED_TEXT : bad.includes("ibge_code") ? FIELD_PROBLEMS.ibge_code : undefined}
      >
        <input
          value={values.ibge_code}
          onChange={(e) => set("ibge_code", e.target.value)}
          inputMode="numeric"
          disabled={locked || ibgeOff}
          className="mono"
          style={inputStyle}
        />
      </Field>

      <Field
        label="Endereço do PEC"
        hint="https://, sem usuário nem senha"
        error={bad.includes("pec_url") ? FIELD_PROBLEMS.pec_url : undefined}
      >
        <input
          value={values.pec_url}
          onChange={(e) => set("pec_url", e.target.value)}
          placeholder="https://pec.cidade.gov.br"
          disabled={locked}
          className="mono"
          style={inputStyle}
        />
      </Field>

      {pending && (
        <div
          role="group"
          aria-label="Confirmar mudança de modo"
          style={{ padding: "10px 12px", borderRadius: 8, background: "var(--warn-bg)", color: "var(--ink)", fontSize: 12, display: "flex", flexDirection: "column", gap: 8 }}
        >
          <p style={{ margin: 0 }}>{modeChangeWarning(city.record_mode, pending.record_mode ?? values.record_mode)}</p>
          <div style={{ display: "flex", gap: 8 }}>
            <button type="button" onClick={() => { void send(pending); }} disabled={busy} style={btnPrimary(busy)}>
              {busy ? "Salvando…" : "Confirmar mudança"}
            </button>
            <button type="button" onClick={() => setPending(null)} disabled={busy} style={btnGhost}>
              Cancelar
            </button>
          </div>
        </div>
      )}

      {error && (
        <div role="alert" style={{ padding: "8px 10px", borderRadius: 6, background: "var(--down-bg)", color: "var(--down)", fontSize: 12 }}>
          {error}
        </div>
      )}
      {notice && (
        <div role="status" style={{ fontSize: 12, color: "var(--ink2)" }}>{notice}</div>
      )}

      <button type="submit" disabled={locked} style={btnPrimary(locked)}>
        {busy && !pending ? "Salvando…" : "Salvar"}
      </button>
    </form>
  );
}

function Field({ label, hint, error, children }: { label: string; hint?: string; error?: string; children: ReactNode }) {
  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 4 }}>
      <label style={{ display: "flex", flexDirection: "column", gap: 4 }}>
        <span className="mono" style={{ fontSize: 10.5, color: "var(--ink3)", textTransform: "uppercase", letterSpacing: 0.6 }}>
          {label}{hint ? ` · ${hint}` : ""}
        </span>
        {children}
      </label>
      {error && <span style={{ fontSize: 11.5, color: "var(--down)" }}>{error}</span>}
    </div>
  );
}

const inputStyle: React.CSSProperties = {
  padding: "9px 10px", fontSize: 13, fontFamily: "var(--font-sans)",
  color: "var(--ink)", background: "var(--panel)",
  border: "1px solid var(--rule2)", borderRadius: 8, outline: "none"
};

const btnGhost: React.CSSProperties = {
  padding: "8px 12px", borderRadius: 8, border: "1px solid var(--rule2)",
  background: "var(--panel)", color: "var(--ink)", fontSize: 13, cursor: "pointer"
};

function btnPrimary(disabled: boolean): React.CSSProperties {
  return {
    padding: "10px 12px", borderRadius: 8, border: "none",
    background: disabled ? "var(--ink3)" : "var(--ink)",
    color: "var(--panel)", fontFamily: "var(--font-sans)", fontSize: 13, fontWeight: 600,
    cursor: disabled ? "default" : "pointer",
    opacity: disabled ? 0.7 : 1
  };
}
```

Confira que `--warn-bg` existe em `src/theme/global.css` (`grep -n "warn-bg" src/theme/global.css`). Se não existir, use `var(--sunken)` no fundo do grupo de confirmação.

- [ ] **Step 4: Mount it in the record**

Substitua `src/modules/cities/CityRecord.tsx` por:

```tsx
// Ficha da cidade (módulo 16, ADR 0028). Abre de "Cidades" pelo botão
// "Ficha". Lê GET /cities/:id. O cabeçalho usa nome e UF da linha da lista,
// que já está carregada (o show devolve os mesmos campos). Depois de salvar, a ficha passa a mostrar a cidade que
// o PATCH devolveu (inclusive o `features`, que o servidor recalcula) e a
// lista de cidades é recarregada.

import { useQueryClient } from "@tanstack/react-query";
import { cityKey, useCity } from "../../hooks/useCities";
import type { CityDetail, CityRow } from "../../lib/types";
import { PageHeader } from "../../components/PageHeader";
import { Panel } from "../../components/Panel";
import { ErrorState } from "../../components/ErrorState";
import { Skeleton } from "../../components/Skeleton";
import { CityFeatures } from "./CityFeatures";
import { RecordSettingsForm } from "./RecordSettingsForm";

interface Props {
  city: CityRow;
  onBack: () => void;
}

export function CityRecord({ city, onBack }: Props) {
  const qc = useQueryClient();
  const { data, isLoading, isError, error, refetch } = useCity(city.id);

  function onSaved(updated: CityDetail) {
    qc.setQueryData(cityKey(city.id), updated);
    void qc.invalidateQueries({ queryKey: [ "cities" ] });
  }

  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 16 }}>
      <div>
        <button type="button" aria-label="Voltar para Cidades" onClick={onBack} style={backBtn}>
          ← Cidades
        </button>
      </div>
      <PageHeader title={`${city.name}${city.uf ? ` · ${city.uf}` : ""}`} sub="ficha da cidade" />
      {isLoading && <Skeleton rows={4} />}
      {isError && <ErrorState message={(error as Error)?.message || "Erro"} onRetry={() => refetch()} />}
      {data && (
        <>
          <Panel title="Prontuário e e-SUS" sub="modo · código IBGE · endereço do PEC">
            <RecordSettingsForm city={data} onSaved={onSaved} />
          </Panel>
          <Panel title="Interruptores" sub="só leitura · quem liga é o maintenance">
            <CityFeatures features={data.features} />
          </Panel>
        </>
      )}
    </div>
  );
}

const backBtn: React.CSSProperties = {
  padding: "5px 10px", borderRadius: 8, border: "1px solid var(--rule2)",
  background: "var(--panel)", color: "var(--ink)", fontSize: 12, cursor: "pointer"
};
```

- [ ] **Step 5: README**

Em `README.md`, logo depois do item "**Cidades**: lista o catálogo e oferece **Entrar** …" (que termina em "`domain_events` da cidade)."), acrescente:

```markdown
- **Ficha da cidade** (botão **Ficha** em Cidades, módulo 16, ADR 0028):
  modo de prontuário (`off`, `integrated`, `record`), código IBGE e endereço
  HTTPS do PEC, por `PATCH /cities/:id/record_settings`. Mudar o modo pede
  confirmação. Os interruptores da cidade (`ledi_export`, `cadsus_lookup`)
  aparecem **só para leitura**, com o que falta: quem liga é o mantenedor, no
  `maintenance`; credenciais são da cidade (dashboard, Integrações).
```

- [ ] **Step 6: Run tests**

Run: `npx vitest run src/modules/cities && npx tsc --noEmit`
Expected: PASS (`CityRecord.test.tsx` e `RecordSettingsForm.test.tsx`) e `tsc` sem erro.

- [ ] **Step 7: Commit**

```bash
/opt/homebrew/bin/git add src/modules/cities/RecordSettingsForm.tsx src/modules/cities/RecordSettingsForm.test.tsx src/modules/cities/CityRecord.tsx README.md
/opt/homebrew/bin/git commit -m "$(cat <<'EOF'
feat: edit record mode, IBGE code and PEC address from the city record

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 5: Regras da produção das cidades

**Files:**
- Create: `src/lib/cityProduction.ts`
- Test: `src/lib/cityProduction.test.ts`

**Interfaces:**
- Consumes: `CityProductionData`, `CityProductionCity`, `CityProductionTerminology`, `CompetenceSummary` (Task 1).
- Produces:
  - `fmtCompetence(c: string): string` (`"202610"` → `"10/2026"`);
  - `fmtDay(iso: string): string` (`"2026-11-14"` → `"14/11/2026"`);
  - `deadlineText(c: CompetenceSummary): string`;
  - `alertView(alert: string): { label: string; tone: "neutral" | "warn" | "down" }`;
  - `interface ProductionRow { key: string; slug: string; name: string; record_mode: string; competence: CompetenceSummary | null }`;
  - `productionRows(data: CityProductionData): ProductionRow[]`;
  - `countAlerts(data: CityProductionData): { critical: number; attention: number }`;
  - `sigtapNotice(t: CityProductionTerminology): { tone: "warn" | "info"; text: string } | null` (a regra do dia 5 é do api: `sigtap_alert`).

- [ ] **Step 1: Write the failing test**

Create `src/lib/cityProduction.test.ts`:

```ts
import { describe, it, expect } from "vitest";
import type { CityProductionData, CompetenceSummary } from "./api";
import {
  alertView, countAlerts, deadlineText, fmtCompetence, fmtDay, productionRows, sigtapNotice
} from "./cityProduction";

function comp(over: Partial<CompetenceSummary>): CompetenceSummary {
  return {
    competence: "202610", deadline_on: "2026-11-14", business_days_left: 7,
    accepted: 0, rejected: 0, pending: 0, failed: 0, alert: "none", ...over
  };
}

const DATA: CityProductionData = {
  cities: [
    { slug: "aurora", name: "Aurora", record_mode: "integrated",
      competences: [ comp({ competence: "202609", deadline_on: "2026-10-14", business_days_left: 6 }), comp({ alert: "attention" }) ] },
    { slug: "maringa", name: "Maringá", record_mode: "off", competences: [] },
    { slug: "curitiba", name: "Curitiba", record_mode: "record",
      competences: [ comp({ alert: "critical", business_days_left: 2 }) ] },
    { slug: "bela", name: "Bela", record_mode: "integrated", competences: [ comp({}) ] }
  ],
  terminology: { sigtap_current_competence: "202610", sigtap_imported: false, sigtap_alert: false }
};

describe("formatting", () => {
  it("competence as MM/AAAA and days by text", () => {
    expect(fmtCompetence("202610")).toBe("10/2026");
    expect(fmtCompetence("2026-10")).toBe("2026-10");
    expect(fmtDay("2026-11-14")).toBe("14/11/2026");
  });
});

describe("deadlineText at the edges", () => {
  it.each([
    [ 7, "14/11/2026 · 7 dias úteis" ],
    [ 1, "14/11/2026 · 1 dia útil" ],
    [ 0, "14/11/2026 · último dia" ],
    [ -1, "14/11/2026 · prazo vencido" ]
  ])("%i business days left → %j", (left, text) => {
    expect(deadlineText(comp({ business_days_left: left }))).toBe(text);
  });
});

describe("alertView", () => {
  it("labels and tones, unknown raw", () => {
    expect(alertView("none")).toEqual({ label: "sem alerta", tone: "neutral" });
    expect(alertView("attention")).toEqual({ label: "atenção", tone: "warn" });
    expect(alertView("critical")).toEqual({ label: "crítico", tone: "down" });
    expect(alertView("panic")).toEqual({ label: "panic", tone: "neutral" });
  });
});

describe("productionRows", () => {
  it("puts the worst alert first, then by name, newest competence first", () => {
    expect(productionRows(DATA).map((r) => r.key)).toEqual([
      "curitiba:202610",
      "aurora:202610", "aurora:202609",
      "bela:202610",
      "maringa:none"
    ]);
  });

  it("productionRows keeps a city without competences", () => {
    const row = productionRows(DATA).find((r) => r.slug === "maringa");
    expect(row).toEqual({ key: "maringa:none", slug: "maringa", name: "Maringá", record_mode: "off", competence: null });
  });

  it("countAlerts counts each city once by its worst competence", () => {
    expect(countAlerts(DATA)).toEqual({ critical: 1, attention: 1 });
  });
});

describe("sigtapNotice follows the api flags", () => {
  const T = DATA.terminology;

  it("nothing when SIGTAP is imported, even with a stale alert flag", () => {
    expect(sigtapNotice({ ...T, sigtap_imported: true, sigtap_alert: true })).toBeNull();
  });

  it("info while the api does not raise the alert", () => {
    const notice = sigtapNotice(T);
    expect(notice?.tone).toBe("info");
    expect(notice?.text).toBe("SIGTAP da competência 10/2026 ainda não importada (vira alerta no dia 5).");
  });

  it("alert when the api says so", () => {
    const notice = sigtapNotice({ ...T, sigtap_alert: true });
    expect(notice?.tone).toBe("warn");
    expect(notice?.text).toBe(
      "SIGTAP da competência 10/2026 não importada. Até a importação, a conferência de procedimentos " +
      "usa a versão anterior. Importe no api com terminology:import."
    );
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `npx vitest run src/lib/cityProduction.test.ts`
Expected: FAIL — `Failed to resolve import "./cityProduction"`.

- [ ] **Step 3: Write the implementation**

Create `src/lib/cityProduction.ts`:

```ts
// Regras da tela "Produção das cidades" (módulo 16, spec §6.5; contratos
// §4.3). Puras: nenhum React, nenhum fetch, nenhuma data.
//
// O alerta de cada competência vem pronto do api (none | attention |
// critical) e o da SIGTAP também (sigtap_alert: dia ≥ 5 em America/Sao_Paulo,
// calculado no api); a tela só ordena, rotula e conta. As competências já vêm
// com a corrente primeiro; a ordenação aqui é rede de segurança.

import type {
  CityProductionCity, CityProductionData, CityProductionTerminology, CompetenceSummary
} from "./api";

export function fmtCompetence(c: string): string {
  return /^\d{6}$/.test(c) ? `${c.slice(4, 6)}/${c.slice(0, 4)}` : c;
}

// "YYYY-MM-DD" → "DD/MM/YYYY" por texto: new Date("2026-11-14") é meia-noite
// UTC, que em São Paulo ainda é dia 13.
export function fmtDay(iso: string): string {
  const [ y, m, d ] = iso.split("-");
  return `${d}/${m}/${y}`;
}

export function deadlineText(c: CompetenceSummary): string {
  const day = fmtDay(c.deadline_on);
  const left = c.business_days_left;
  if (left < 0) return `${day} · prazo vencido`;
  if (left === 0) return `${day} · último dia`;
  if (left === 1) return `${day} · 1 dia útil`;
  return `${day} · ${left} dias úteis`;
}

export interface AlertView {
  label: string;
  tone: "neutral" | "warn" | "down";
}

const ALERTS: Record<string, AlertView> = {
  none: { label: "sem alerta", tone: "neutral" },
  attention: { label: "atenção", tone: "warn" },
  critical: { label: "crítico", tone: "down" }
};

export function alertView(alert: string): AlertView {
  return ALERTS[alert] ?? { label: alert, tone: "neutral" };
}

export interface ProductionRow {
  key: string;
  slug: string;
  name: string;
  record_mode: string;
  competence: CompetenceSummary | null;
}

const RANK: Record<string, number> = { critical: 0, attention: 1, none: 2 };

function worst(city: CityProductionCity): number {
  return Math.min(2, ...city.competences.map((c) => RANK[c.alert] ?? 2));
}

// Cidade com alerta pior primeiro, depois por nome; dentro da cidade, a
// competência mais nova primeiro. Cidade sem competência vira UMA linha
// (competence: null) — nunca some da tabela.
export function productionRows(data: CityProductionData): ProductionRow[] {
  const cities = [ ...data.cities ].sort(
    (a, b) => worst(a) - worst(b) || a.name.localeCompare(b.name, "pt-BR")
  );
  return cities.flatMap((city) => {
    const base = { slug: city.slug, name: city.name, record_mode: city.record_mode };
    if (city.competences.length === 0) return [ { key: `${city.slug}:none`, ...base, competence: null } ];
    return [ ...city.competences ]
      .sort((x, y) => y.competence.localeCompare(x.competence))
      .map((c) => ({ key: `${city.slug}:${c.competence}`, ...base, competence: c }));
  });
}

export function countAlerts(data: CityProductionData): { critical: number; attention: number } {
  let critical = 0;
  let attention = 0;
  for (const city of data.cities) {
    const w = worst(city);
    if (w === 0) critical += 1;
    else if (w === 1) attention += 1;
  }
  return { critical, attention };
}

export interface SigtapNotice {
  tone: "warn" | "info";
  text: string;
}

export function sigtapNotice(t: CityProductionTerminology): SigtapNotice | null {
  if (t.sigtap_imported) return null;
  const competence = fmtCompetence(t.sigtap_current_competence);
  if (t.sigtap_alert) {
    return {
      tone: "warn",
      text: `SIGTAP da competência ${competence} não importada. Até a importação, a conferência de ` +
        "procedimentos usa a versão anterior. Importe no api com terminology:import."
    };
  }
  return { tone: "info", text: `SIGTAP da competência ${competence} ainda não importada (vira alerta no dia 5).` };
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `npx vitest run src/lib/cityProduction.test.ts && npx tsc --noEmit`
Expected: PASS e `tsc` sem erro.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add src/lib/cityProduction.ts src/lib/cityProduction.test.ts
/opt/homebrew/bin/git commit -m "$(cat <<'EOF'
feat: add deadline, alert and SIGTAP notice rules for city production

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 6: Tela "Produção das cidades" e navegação

**Files:**
- Create: `src/modules/production/CityProduction.tsx`
- Modify: `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/App.tsx`, `README.md`
- Test: `src/modules/production/CityProduction.test.tsx`

**Interfaces:**
- Consumes: `listCityProduction`, `CityProductionData` (Task 1); `productionRows`, `countAlerts`, `deadlineText`, `fmtCompetence`, `alertView`, `sigtapNotice`, `ProductionRow` (Task 5); `recordModeLabel` (Task 2).
- Produces: `CityProduction()`; `ModuleId` ganha `"city_production"`; grupo de navegação `e-SUS` com o item `Produção das cidades`.

- [ ] **Step 1: Write the failing tests**

Create `src/modules/production/CityProduction.test.tsx`:

```tsx
import { describe, it, expect, vi, afterEach, beforeEach } from "vitest";
import { render, screen, cleanup } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";

vi.mock("../../lib/api", async (importOriginal) => {
  const actual = await importOriginal<typeof import("../../lib/api")>();
  return { ...actual, listCityProduction: vi.fn() };
});

import { listCityProduction, ApiError, type CityProductionData, type CompetenceSummary } from "../../lib/api";
import { CityProduction } from "./CityProduction";

const mocked = vi.mocked(listCityProduction);

function comp(over: Partial<CompetenceSummary>): CompetenceSummary {
  return {
    competence: "202610", deadline_on: "2026-11-14", business_days_left: 7,
    accepted: 120, rejected: 3, pending: 10, failed: 0, alert: "none", ...over
  };
}

const DATA: CityProductionData = {
  cities: [
    { slug: "maringa", name: "Maringá", record_mode: "integrated",
      competences: [ comp({ alert: "attention", business_days_left: 0 }),
                     comp({ competence: "202609", deadline_on: "2026-10-14", business_days_left: -1, accepted: 1234 }) ] },
    { slug: "curitiba", name: "Curitiba", record_mode: "record",
      competences: [ comp({ alert: "critical", business_days_left: 1, accepted: 0 }) ] },
    { slug: "nova", name: "Nova", record_mode: "off", competences: [] }
  ],
  terminology: { sigtap_current_competence: "202610", sigtap_imported: true, sigtap_alert: false }
};

function renderScreen() {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  return render(<QueryClientProvider client={client}><CityProduction /></QueryClientProvider>);
}

function rowsText(): string[] {
  return screen.getAllByRole("row").slice(1).map((r) => r.textContent ?? "");
}

describe("CityProduction", () => {
  beforeEach(() => { mocked.mockReset(); });
  afterEach(() => { cleanup(); });

  it("lists critical first, with mode, competence, counts and alert", async () => {
    mocked.mockResolvedValue(DATA);
    renderScreen();

    expect(await screen.findByText("1 em alerta crítico · 1 em atenção")).toBeTruthy();
    const rows = rowsText();
    expect(rows[0]).toContain("Curitiba");
    expect(rows[0]).toContain("Prontuário Rota Saúde");
    expect(rows[0]).toContain("10/2026");
    expect(rows[0]).toContain("crítico");
    expect(rows[1]).toContain("Maringá");
    expect(rows[1]).toContain("atenção");
    expect(rows[2]).toContain("09/2026");
    expect(rows[2]).toContain("1.234");
    for (const label of [ "Aceitas", "Recusadas", "Pendentes", "Falhas", "Prazo", "Alerta" ]) {
      expect(screen.getByText(label)).toBeTruthy();
    }
  });

  it("deadline edges and a city without competences", async () => {
    mocked.mockResolvedValue(DATA);
    renderScreen();

    await screen.findByText("1 em alerta crítico · 1 em atenção");
    const rows = rowsText();
    expect(rows[0]).toContain("14/11/2026 · 1 dia útil");
    expect(rows[1]).toContain("14/11/2026 · último dia");
    expect(rows[2]).toContain("14/10/2026 · prazo vencido");
    expect(rows[3]).toContain("Nova");
    expect(rows[3]).toContain("Desligado");
    expect(rows[3]).toContain("sem fichas");
  });

  it("explains attention and critical", async () => {
    mocked.mockResolvedValue(DATA);
    renderScreen();

    expect(await screen.findByText(/Atenção: até 5 dias úteis do prazo com ficha pendente ou recusada/)).toBeTruthy();
    expect(screen.getByText(/Crítico: modo Prontuário Rota Saúde e nenhuma ficha aceita a até 3 dias úteis do prazo/)).toBeTruthy();
  });

  it("SIGTAP notice: alert when the api flags it, nothing when imported", async () => {
    mocked.mockResolvedValue({ ...DATA, terminology: { sigtap_current_competence: "202610", sigtap_imported: false, sigtap_alert: true } });
    renderScreen();

    expect((await screen.findByRole("alert")).textContent).toContain("SIGTAP da competência 10/2026 não importada");
    cleanup();

    mocked.mockResolvedValue({ ...DATA, terminology: { sigtap_current_competence: "202610", sigtap_imported: false, sigtap_alert: false } });
    renderScreen();
    expect((await screen.findByRole("status")).textContent).toContain("ainda não importada (vira alerta no dia 5)");
    expect(screen.queryByRole("alert")).toBeNull();
    cleanup();

    mocked.mockResolvedValue(DATA);
    renderScreen();
    await screen.findByText("1 em alerta crítico · 1 em atenção");
    expect(screen.queryByText(/SIGTAP/)).toBeNull();
  });

  it("shows the empty state when there is no active city", async () => {
    mocked.mockResolvedValue({ ...DATA, cities: [] });
    renderScreen();

    expect(await screen.findByText("Nenhuma cidade ativa.")).toBeTruthy();
    expect(screen.queryByRole("table")).toBeNull();
  });

  it("shows the error state when the request fails", async () => {
    mocked.mockRejectedValue(new ApiError(500, "", "500 on /city_production"));
    renderScreen();

    expect(await screen.findByText("Falha ao carregar")).toBeTruthy();
    expect(screen.getByText("500 on /city_production")).toBeTruthy();
    expect(screen.queryByRole("table")).toBeNull();
  });
});
```

Em `src/shell/modules.test.ts`, acrescente no fim:

```ts
describe("NAV_GROUPS e-SUS", () => {
  const group = NAV_GROUPS.find((g) => g.label === "e-SUS");
  const item = group?.items.find((i) => i.id === "city_production");

  it("lists Produção das cidades in its own group", () => {
    expect(item?.label).toBe("Produção das cidades");
  });

  it("shows it to operators only", () => {
    expect(item?.visible?.({ operator: true, memberships: [] })).toBe(true);
    expect(item?.visible?.({ operator: false, memberships: [ { role: "municipal_admin" } ] })).toBe(false);
  });
});
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `npx vitest run src/modules/production/CityProduction.test.tsx src/shell/modules.test.ts`
Expected: FAIL — `Failed to resolve import "./CityProduction"` e o grupo `e-SUS` não existe.

- [ ] **Step 3: Screen**

Create `src/modules/production/CityProduction.tsx`:

```tsx
// Produção das cidades (módulo 16, spec §6.5; contratos §4.3) — só leitura.
// Fala com GET /city_production (PlatformConsoleHost, operador): resumo por
// cidade da competência corrente e da anterior (aceitas, recusadas,
// pendentes, falhas, prazo e alerta). Nenhuma ficha nem dado de cidadão chega
// ao console. O aviso de SIGTAP não importada vale para todas as cidades.

import { useQuery } from "@tanstack/react-query";
import { listCityProduction } from "../../lib/api";
import {
  alertView, countAlerts, deadlineText, fmtCompetence, productionRows, sigtapNotice, type ProductionRow
} from "../../lib/cityProduction";
import { recordModeLabel } from "../../lib/recordSettings";
import { fmtNumber } from "../../lib/format";
import { PageHeader } from "../../components/PageHeader";
import { DataTable, type Column } from "../../components/DataTable";
import { EmptyState } from "../../components/EmptyState";
import { ErrorState } from "../../components/ErrorState";
import { Skeleton } from "../../components/Skeleton";
import { Tag } from "../../components/Tag";

type CountKey = "accepted" | "rejected" | "pending" | "failed";

function count(r: ProductionRow, key: CountKey): string {
  return r.competence ? fmtNumber(r.competence[key]) : "—";
}

export function CityProduction() {
  const { data, isLoading, isError, error, refetch } = useQuery({
    queryKey: [ "city_production" ],
    queryFn: listCityProduction,
    staleTime: 60_000
  });

  const notice = data ? sigtapNotice(data.terminology) : null;
  const alerts = data ? countAlerts(data) : null;

  const cols: Column<ProductionRow>[] = [
    { label: "Cidade", w: "1.2fr", render: (r) => r.name },
    { label: "Modo", w: "1.1fr", render: (r) => recordModeLabel(r.record_mode) },
    { label: "Competência", w: "0.8fr", render: (r) => (
        r.competence ? <span className="mono">{fmtCompetence(r.competence.competence)}</span> : "sem fichas"
      ) },
    { label: "Prazo", w: "1.4fr", render: (r) => (r.competence ? deadlineText(r.competence) : "—") },
    { label: "Aceitas", w: "0.6fr", align: "right", render: (r) => count(r, "accepted") },
    { label: "Recusadas", w: "0.7fr", align: "right", render: (r) => count(r, "rejected") },
    { label: "Pendentes", w: "0.7fr", align: "right", render: (r) => count(r, "pending") },
    { label: "Falhas", w: "0.6fr", align: "right", render: (r) => count(r, "failed") },
    { label: "Alerta", w: "0.8fr", render: (r) => {
        if (!r.competence) return "—";
        const view = alertView(r.competence.alert);
        return <Tag tone={view.tone}>{view.label}</Tag>;
      } }
  ];

  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 16 }}>
      <PageHeader title="Produção das cidades" sub="e-SUS APS · competência corrente e anterior" />
      <p style={{ fontSize: 12, color: "var(--ink3)", margin: 0 }}>
        Fichas enviadas ao PEC de cada cidade. O prazo é o 10º dia útil do mês seguinte à competência.
        Atenção: até 5 dias úteis do prazo com ficha pendente ou recusada. Crítico: modo Prontuário Rota
        Saúde e nenhuma ficha aceita a até 3 dias úteis do prazo.
      </p>

      {notice && (
        <div
          role={notice.tone === "warn" ? "alert" : "status"}
          style={{
            padding: "8px 10px", borderRadius: 6, fontSize: 12,
            background: notice.tone === "warn" ? "var(--down-bg)" : "var(--sunken)",
            color: notice.tone === "warn" ? "var(--down)" : "var(--ink2)"
          }}
        >
          {notice.text}
        </div>
      )}

      {isLoading && <Skeleton rows={6} />}
      {isError && <ErrorState message={(error as Error)?.message || "Erro"} onRetry={() => refetch()} />}

      {data && data.cities.length === 0 && <EmptyState title="Nenhuma cidade ativa." />}

      {data && alerts && data.cities.length > 0 && (
        <>
          <p className="mono" style={{ fontSize: 11, color: "var(--ink2)", margin: 0 }}>
            {`${alerts.critical} em alerta crítico · ${alerts.attention} em atenção`}
          </p>
          <DataTable<ProductionRow> cols={cols} rows={productionRows(data)} rowKey={(r) => r.key} />
        </>
      )}
    </div>
  );
}
```

O teste "lists critical first" espera `1.234` para 1234: `fmtNumber` usa `Intl.NumberFormat("pt-BR")`, que separa milhar com ponto.

- [ ] **Step 4: Navigation**

Em `src/shell/modules.ts`, troque:

```ts
  // Módulo 14 (ADR 0025)
  | "city_analytics";
```

por:

```ts
  // Módulo 14 (ADR 0025)
  | "city_analytics"
  // Módulo 16 (ADR 0028)
  | "city_production";
```

e, no fim de `NAV_GROUPS`, depois do grupo `Analytics`, acrescente o grupo (cuidado com a vírgula depois do `}` do grupo `Analytics`):

```ts
  {
    label: "e-SUS",
    items: [
      {
        id: "city_production",
        label: "Produção das cidades",
        icon: "⇪",
        visible: (u) => u.operator
      }
    ]
  }
```

Em `src/App.tsx`, acrescente o import depois do de `CityAnalytics`:

```ts
import { CityProduction } from "./modules/production/CityProduction";
```

e o caso no `switch`, depois de `case "city_analytics": return <CityAnalytics />;`:

```ts
    case "city_production": return <CityProduction />;
```

- [ ] **Step 5: README**

Em `README.md`, depois do item "**Analytics das cidades** …" (termina em "O console nunca vê bairro, unidade, protocolo ou pergunta."), acrescente:

```markdown
- **Produção das cidades**: `GET /city_production`, só leitura (módulo 16).
  Por cidade, a competência corrente e a anterior: aceitas, recusadas,
  pendentes, falhas, prazo (10º dia útil do mês seguinte) e alerta
  (`attention`, `critical`). Avisa quando a SIGTAP da competência corrente não
  foi importada (o api levanta o alerta a partir do dia 5, em America/Sao_Paulo).
```

- [ ] **Step 6: Run tests**

Run: `npx vitest run src/modules/production src/shell/modules.test.ts && npx tsc --noEmit`
Expected: PASS e `tsc` sem erro (o `switch` de `renderModule` cobre o `ModuleId` novo).

- [ ] **Step 7: Commit**

```bash
/opt/homebrew/bin/git add src/modules/production/CityProduction.tsx src/modules/production/CityProduction.test.tsx src/shell/modules.ts src/shell/modules.test.ts src/App.tsx README.md
/opt/homebrew/bin/git commit -m "$(cat <<'EOF'
feat: add the city production screen with deadline alerts and SIGTAP notice

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>
EOF
)"
```

---

### Task 7: Suíte, build, revisão e prova no navegador

- [ ] **Step 1: O admin não escreve interruptores nem toca nos módulos legados**

Run (no worktree):

```bash
grep -rn "setCityFeature\|city_features\|/admin/api" src --include=*.ts --include=*.tsx | grep -v "^src/lib/api.ts:.*VITE_ADMIN_API_BASE\|^src/lib/api.ts:.*/admin/api\"" 
/opt/homebrew/bin/git diff --stat origin/main..HEAD -- src/modules/Overview.tsx src/modules/Ingestion.tsx src/modules/Conversations.tsx src/modules/Consent.tsx src/modules/Triages.tsx src/modules/Classification.tsx src/modules/Protocols.tsx src/modules/Queues.tsx src/modules/Events.tsx src/modules/Health.tsx src/hooks
```

Expected: o `grep` não acha nenhuma escrita de interruptor; o `diff --stat` lista só `src/hooks/useCities.ts`.

- [ ] **Step 2: Suíte, tipos e build**

Run: `npx vitest run && npx tsc --noEmit && npm run build`
Expected: tudo verde (a CI roda `typecheck`, `test` e `build`). `dist/` está no `.gitignore`.

```bash
/opt/homebrew/bin/git status --short
/opt/homebrew/bin/git log --stat origin/main..HEAD | grep -c node_modules
```

Expected: `status` vazio e a contagem `0`. Se o symlink aparecer, **não** o adicione.

- [ ] **Step 3: Revisão**

Um subagente revisor lê `origin/main..HEAD` do admin contra a spec (§3.1, §3.2, §4, §6.5, §9), o ADR 0028 e o arquivo de contratos (§2 e §4). Pontos de atenção:
- nenhum controle de ligar/desligar interruptor; a ficha diz que quem liga é o `maintenance`;
- mudar o modo sempre passa pela confirmação; IBGE e PEC sozinhos não;
- o PATCH leva só o que mudou; vazio vira `null`; `record_mode` igual nunca vai;
- a URL do PEC com usuário/senha nunca sai da tela e nenhuma mensagem repete o valor digitado;
- `missing` vem do servidor, nunca calculado no front; depois de salvar, a ficha usa a resposta do PATCH;
- datas `YYYY-MM-DD` e competências formatadas por texto; o console não calcula o "dia 5" (usa `sigtap_alert`);
- com `city_reachable: false` o IBGE fica bloqueado e o PATCH nunca leva `ibge_code`;
- tipos idênticos ao contrato §4 (nomes de campo e códigos de erro).

- [ ] **Step 4: Prova no navegador (com o usuário)**

Rode o api da branch do módulo 16 na porta **3033**, com a semente do módulo (spec §10: Curitiba `4106902` e Maringá `4115200`, `record_mode=off`, SIGTAP reduzida ativa) e o exportador (para `/city_production`). Depois o Vite do worktree apontando para ele:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/admin/.claude/mod16 && VITE_API_PROXY_TARGET=http://localhost:3033 npx vite --port 5181 --host 0.0.0.0
```

Abra `http://admin.localhost:5181/admin/` (o host `admin.localhost` é o que faz o Rails tratar a requisição como console). O usuário faz o login do operador de dev; não digite senha nem TOTP. Confira com screenshot:
- Cidades: coluna "Prontuário" com "Desligado"; "Ficha de Curitiba" abre a ficha com IBGE `4106902`;
- interruptores: `ledi_export` desligado com "modo de prontuário desligado (aqui)", "endereço do PEC (aqui)" e a credencial; a frase sobre o `maintenance`; nenhum botão de ligar;
- digitar `https://admin:x@pec.exemplo.gov.br` → recusa na tela e nenhuma requisição `PATCH` na aba de rede;
- `https://pec.exemplo.gov.br` + modo "Integrado ao PEC" → confirmação com o texto do modo → "Confirmar mudança" → "Salvo."; os itens "(aqui)" somem do que falta;
- o IBGE que aparece na ficha é o mesmo do perfil da cidade (o do provisionamento); trocar para `4106903` e salvar → "Salvo."; voltar a `4106902`;
- voltar ao modo "Desligado" (de novo com confirmação), para deixar a semente como estava;
- Produção das cidades: as duas cidades, competência corrente e anterior, prazo e alerta; sem SIGTAP da competência corrente na semente, conferir o aviso (neutro antes do dia 5, alerta a partir dele, conforme `sigtap_alert` do api).

- [ ] **Step 5:** **Pare.** O merge do admin só vem depois do merge do api (fundação **e** exportador, por causa de `/city_production`), e só com autorização explícita do usuário. Ordem de deploy (spec §11): contracts → api → maintenance, admin, dashboard.

---

## Nota: incorporado ao contrato

As nove divergências levantadas na escrita deste plano foram resolvidas no arquivo de contratos (§4 e §7); o plano segue o texto atual:

1. **§4.3:** `GET /city_production` usa o envelope `{ data }`, como `/cities` e `/city_analytics` (`listCityProduction` desembrulha).
2. **§4.3:** o api manda `terminology.sigtap_alert` (dia ≥ 5 em `America/Sao_Paulo`); o console não calcula data (`sigtapNotice` só lê as flags).
3. **§4.2:** o `show` devolve também os campos do item da lista (`name`, `uf`, `created_at`…); `CityDetail` estende `CityRow`.
4. **§4.2, regras do PATCH:** `null` limpa `pec_url` e `ibge_code`, `""` vale `null`, `record_mode` nunca é nulo.
5. **§4.2:** cidade inexistente responde 404 `{ "error": "not_found" }` ("cidade não encontrada").
6. **§4.1/§4.2:** o IBGE tem fonte única no `city_profile` da cidade; com o banco inalcançável a resposta é 200 com `ibge_code: null` e `city_reachable: false`, e a ficha bloqueia o IBGE (Review Focus 5); o PATCH pode responder 503 `city_unreachable`.
7. **§4.1/§6:** um só evento de auditoria, `city.record_settings_changed` (a spec foi corrigida); o admin não emite evento.
8. **§7:** o admin mergeia depois do exportador (Task 7, Step 5).
9. **§4.3:** `competences` vem com a corrente primeiro; o front mantém a ordenação como rede de segurança (`productionRows`).

## Self-review

- **Cobertura da spec:**
  - §3.2 e contratos §4.1, edição de modo, IBGE e PEC na ficha da cidade, com confirmação ao mudar o modo e os 422: Tasks 1, 2 e 4;
  - contratos §4.1/§4.2, IBGE de fonte única (`city_profile`), bloqueado com a cidade inalcançável e 503 `city_unreachable`: Tasks 2 e 4;
  - §3.1 e contratos §4.2, interruptores no console só leitura (ligado, utilizável, o que falta; quem liga é o `maintenance`): Tasks 2 e 3;
  - §6.5 "Console: tabela de todas as cidades com o mesmo resumo" e contratos §4.3 (alertas `attention`/`critical`): Tasks 5 e 6;
  - §4 "Alerta no console: dia 5 sem SIGTAP da competência corrente" (decidido no api, `sigtap_alert`; mostrado no console): Tasks 5 e 6;
  - §9 "Front: admin (modo/IBGE/PEC, visão geral, alerta SIGTAP)": Tasks 2–6; prova no navegador: Task 7.
- **Placeholders:** nenhum. As Tasks 3 e 4 substituem `Cities.tsx` e `CityRecord.tsx` inteiros; as demais alterações em arquivos existentes são trechos exatos (de/para).
- **Consistência de nomes:**
  - `RecordMode`, `CityFeatureState`, `CityRecordFields`, `CityDetail` (Task 1, `types.ts`) usados nas Tasks 2, 3, 4;
  - `RecordSettingsPatch`, `getCity`, `updateCityRecordSettings`, `CompetenceSummary`, `CityProductionCity`, `CityProductionTerminology`, `CityProductionData`, `listCityProduction` (Task 1, `api.ts`) usados nas Tasks 2, 3, 4, 5, 6;
  - `RECORD_MODES`, `recordModeLabel`, `modeChangeWarning`, `RecordSettingsValues`, `RecordSettingsField`, `FIELD_PROBLEMS`, `formFrom`, `pecUrlProblem`, `validateRecordSettings`, `recordSettingsPatch`, `changesMode`, `describeRecordSettingsError`, `featureLabel`, `missingLabel`, `featureState` (Task 2) usados nas Tasks 3, 4 e 6;
  - `cityKey`, `useCity` (Task 3) usados na Task 4;
  - `fmtCompetence`, `fmtDay`, `deadlineText`, `alertView`, `ProductionRow`, `productionRows`, `countAlerts`, `sigtapNotice` (Task 5) usados na Task 6;
  - `RecordFields`, `ibgeLocked`, `IBGE_LOCKED_TEXT` (Task 2) usados na Task 4.
- **Review Focus:** cada item tem teste na task dona:
  - 1: Tasks 2 e 4;
  - 2: Tasks 2 e 4;
  - 3: Task 4;
  - 4: Tasks 5 e 6;
  - 5: Tasks 2 e 4.
