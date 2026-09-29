# Módulo 12 — Campanhas (dashboard) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Telas do módulo 12 no dashboard. O `campaign_manager` lista, cria e edita campanhas com um construtor de público em cartões guiados, vê a contagem ao vivo, envia ou agenda com step-up e acompanha o painel da campanha só com agregados. O `municipal_admin` liga e desliga o SMS da cidade com step-up e concede o papel novo em Equipe.

**Architecture:** O cliente HTTP de `/campaigns` vai para `src/lib/api.ts`, junto com os tipos do contrato. As regras puras ficam fora do React: rótulos, período padrão, validação que espelha o schema do api, montagem do JSON do público e tradução das recusas vão em `src/lib/campaigns.ts`, e a frase do público em `src/lib/audiencePhrase.ts`. As telas moram em `src/modules/campaigns/`, cada uma testável isolada por props. A raiz `src/modules/Campaigns.tsx` só troca de vista (lista, editor, painel), sem URL, como `Professionals`. O step-up é sempre do `SensitiveAction`, que já existe.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, Vitest 2 + Testing Library 16 (jsdom, sem jest-dom, sem msw: `vi.mock("../lib/api")` e `vi.stubGlobal("fetch")`), Vite 5 (proxy de dev).

**Spec:** `docs/superpowers/specs/2026-09-29-module-12-campaigns-design.md` (§6.1, §6.3 e §7 são deste plano; §4 e §5 dão o contexto) e `docs/adr/0024.md`. Os contratos HTTP estão fixados em `mod12-contracts.md`, o mesmo arquivo que o plano do api usa, e os tipos da Task 1 os copiam literalmente. O plano do api (`docs/superpowers/plans/2026-09-29-module-12-campaigns-api.md`, escrito em paralelo) precisa estar **mergeado antes** do merge deste.

## Global Constraints

- Rotas: todas sob `/campaigns`, sessão municipal. Precisa de **uma** entrada nova no proxy do Vite.
  - `GET /campaigns` → `{ campaigns: CampaignSummary[] }`
  - `GET /campaigns/options` → `{ protocols, tiers, outcomes, neighborhoods: {id,name}[], units: {id,name}[] }`
  - `POST /campaigns/preview` `{ audience }` → `{ citizens, phones }` ou `{ below_minimum: true }`
  - `POST /campaigns` → 201 `{ campaign }`
  - `GET /campaigns/:id` → `{ campaign }`
  - `PATCH /campaigns/:id` → `{ campaign }`
  - `POST /campaigns/:id/{send,schedule,unschedule,cancel}` → `{ campaign }`, todas com step-up. `schedule` leva `{ send_at }`.
  - `GET /campaigns/sms_setting` → `{ enabled, gateway_configured }`
  - `PUT /campaigns/sms_setting` `{ enabled }`, com step-up, só `municipal_admin`.
- Recusas: 403 `missing_role` (sem papel); 401 `mfa_required` (sem step-up, quem trata é o `SensitiveAction`); 404 `not_found`. Os 422:
  - `invalid_audience` e `invalid_campaign`, ambos com `details: [{ path, message }]`;
  - `below_minimum`, `not_editable`, `invalid_transition`, `invalid_send_at`.
- **Escrita sem corpo vai com `body: "{}"`.** A API recusa com 415 `json_required` toda escrita por cookie sem `application/json` (`apps/api/app/controllers/concerns/authentication.rb:81-89`). Isso vale para `send`, `unschedule` e `cancel`.
- Público, versão 1: `{ version: 1, geo, clinical: { all: Criterion[] } }`.
  - `neighborhood_ids` tem de 1 a 50 itens, e são no máximo 7 critérios.
  - Datas em `YYYY-MM-DD`, inclusivas, com `from <= to` e `to` nunca no futuro no fuso da cidade.
  - O schema recusa campo extra. Um campo opcional em branco vai **ausente**, nunca `""` nem `null`.
- Mínimo de 5 **telefones distintos**. A prévia `{ below_minimum: true }` mostra "menos de 5 — ajuste o público" e trava o envio.
- A contagem é "≈ N pessoas (M telefones)", com debounce de **500 ms**.
- `send_at`: pelo menos **5 minutos** à frente e no máximo **90 dias**, digitado em hora de parede da cidade (`parseCityLocal`, `America/Sao_Paulo`).
- O SMS só sai entre **8h e 20h**. O texto do SMS é fixo, e a tela nunca o edita (D6).
- Painel: só agregados. **Nenhuma lista de destinatários** em nenhuma tela (invariante do ADR 0024).
- Texto de alerta (spec §7, literal): "SMS não enviado: a plataforma ainda não tem provedor de SMS".
- Aviso da chave (spec §7, literal): "A plataforma ainda não tem provedor de SMS: os avisos saem, o SMS fica pendente como não enviado".
- Menu "Campanhas": visível para `campaign_manager` e `municipal_admin`. O `municipal_admin` sem o papel vê só a chave de SMS e **nunca** chama `GET /campaigns`, que responderia 403.
- `campaign_manager` é papel privilegiado: conceder, revogar e convidar pedem step-up (ADR 0024, `Membership::PRIVILEGED_ROLES`).
- Testes que dependem de "hoje" fixam o relógio com `vi.useFakeTimers({ toFake: [ "Date" ] })` + `vi.setSystemTime(...)` em `beforeEach`, e `vi.useRealTimers()` em `afterEach`.
- Nunca `git add -A` nem `git add .`: o worktree tem o symlink de `node_modules`. Adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git`.

## Ambiente de execução

- Crie o worktree a partir de `origin/main` e ligue o `node_modules` do checkout principal. `/.claude/` já está no `.gitignore` do dashboard.

  ```bash
  cd apps/dashboard
  /opt/homebrew/bin/git fetch origin
  /opt/homebrew/bin/git worktree add .claude/mod12 -b feat/mod-12-campaigns origin/main
  ln -s ../../node_modules .claude/mod12/node_modules
  ```

- Testes (script `test` = `vitest run`):

  ```bash
  cd apps/dashboard/.claude/mod12 && npx vitest run <arquivos>
  ```

- Checagem de tipos antes de cada commit (script `typecheck` = `tsc --noEmit`):

  ```bash
  cd apps/dashboard/.claude/mod12 && npx tsc --noEmit
  ```

- Base do ambiente de teste:
  - o `vitest.config.ts` já fixa `TZ=America/Sao_Paulo`;
  - não há `setupFiles` nem jest-dom (use `toBeTruthy`, `not.toBeNull()` e `toBeNull()`);
  - `globals: false`, então todo teste de componente chama `afterEach(cleanup)`.
- O `SensitiveAction` só mostra o botão de confirmar depois que o `AuthProvider` carregou a sessão. Nos testes, procure o botão de confirmar com `findByRole`, nunca com `getByRole`.

## Review Focus

1. **Enviar dentro dos 500 ms depois de mexer no público.** A contagem na tela ainda é a do público anterior, e o envio não pode ser liberado por ela: enquanto a chave digitada não é a consultada, o contador diz "calculando…" e o botão fica travado. Testes:
   - Task 6, "público mudou depois da contagem: volta a calculando";
   - Task 8, "envio travado enquanto a contagem não alcançou o público atual".
2. **Campo opcional deixado em branco** ("qualquer unidade", nenhum tipo de pedido marcado). Ele vai como chave ausente, e não como `""`, `null` ou `[]`, que o schema do api recusa como campo inválido. Teste: Task 2, "campo opcional em branco sai do JSON".
3. **Data e fuso.**
   - `"2026-07-01"` aparece como `01/07/2026`, sem voltar um dia por leitura em UTC.
   - Às 23h30 em São Paulo (dia seguinte em UTC), "hoje" continua sendo o dia da cidade, e o limite "não pode terminar no futuro" não aceita amanhã.
   - Testes: Task 2, "hoje é o dia da cidade, mesmo às 23h30"; Task 3, "data sem deslocar o dia".
4. **Rascunho com bairro que depois foi desativado.** `options` só traz bairros ativos. O id que sobrou no rascunho não pode sumir da tela e continuar contando no público: ele aparece marcado como "(bairro inativo)" e dá para desmarcar. A frase diz "(bairro inativo)". Testes:
   - Task 5, "rascunho com bairro fora da lista: aparece e sai ao desmarcar";
   - Task 3, "bairro que não está na lista".
5. **Horário agendado que passou da janela enquanto o modal estava aberto** (a pessoa demorou no código do autenticador). O servidor responde `invalid_send_at`. A mensagem aparece traduzida e o diálogo continua aberto, sem virar "não foi possível concluir". Teste: Task 8, "servidor recusa invalid_send_at: mensagem traduzida e o diálogo continua".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| `vite.config.ts`, `README.md` | proxy de `/campaigns` | 1 |
| `src/lib/api.ts` | tipos do contrato e cliente de `/campaigns` | 1 |
| `src/test/campaignFixtures.tsx` | opções, campanha, sessão e `renderWithProviders` dos testes | 1 |
| `src/lib/campaigns.ts` | chaves de cache, rótulos, "hoje" da cidade, período padrão, validação, `buildAudience`, `validateSendAt`, erros | 2 |
| `src/lib/audiencePhrase.ts` | público em frase (`describeAudience`, `fmtDay`, `joinPt`) | 3 |
| `src/modules/campaigns/styles.ts` | estilos comuns das telas de campanha | 4 |
| `src/modules/campaigns/CriterionCard.tsx` | um cartão por critério, com os campos do tipo | 4 |
| `src/modules/campaigns/AudienceBuilder.tsx` | passo 1 (recorte) e passo 2 (cartões, "e também") | 5 |
| `src/lib/useDebouncedValue.ts`, `src/modules/campaigns/useAudiencePreview.ts`, `AudienceCounter.tsx` | contagem ao vivo com debounce | 6 |
| `src/modules/campaigns/CampaignEditor.tsx` | título, texto, pré-visualização, público, contador, linha do SMS, salvar rascunho | 7 |
| `src/modules/campaigns/SendDialog.tsx`, `CampaignEditor.tsx` | modal de envio/agendamento com step-up e botão "Revisar e enviar…" | 8 |
| `src/modules/campaigns/CampaignLifecycleDialog.tsx`, `CampaignPanel.tsx`, `CampaignEditor.tsx` | painel da campanha, desagendar e cancelar | 9 |
| `src/modules/campaigns/SmsSettingPanel.tsx` | chave de SMS da cidade | 10 |
| `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/App.tsx`, `src/modules/Campaigns.tsx`, `src/modules/campaigns/CampaignList.tsx` | menu por papel, raiz e lista | 11 |
| `src/lib/team.ts`, `src/lib/team.test.ts`, `src/modules/Team.tsx`, `src/modules/Team.test.tsx` | papel `campaign_manager` em Equipe | 12 |
| — | suíte inteira, build, revisão e prova no navegador | 13 |

---

### Task 1: Cliente de `/campaigns`, tipos do contrato e proxy

**Files:**
- Modify: `vite.config.ts`, `README.md`, `src/lib/api.ts` (fim do arquivo)
- Create: `src/test/campaignFixtures.tsx`
- Test: `src/lib/api.campaigns.test.ts`

**Interfaces:**
- Consumes: `jsonFetch`, `AttendanceOutcome` e `SessionUser`, que já estão em `src/lib/api.ts`.
- Produces (em `src/lib/api.ts`):
  - tipos `CampaignStatus`, `SmsStatus`, `RequestKind = "return" | "referral"`, `AudienceGeo`, `CriterionPeriod { from; to }`, `Criterion` (união dos 7 `kind`), `CriterionKind`, `Audience`, `CampaignSummary`, `CampaignStats { read_count; sms: Record<SmsStatus, number> }`, `Campaign`, `NamedRef { id; name }`, `CampaignOptions`, `AudiencePreview`, `SmsSetting`, `CampaignFields { title; body; audience }`;
  - `listCampaigns(): Promise<CampaignSummary[]>`, `getCampaignOptions(): Promise<CampaignOptions>`, `previewAudience(audience): Promise<AudiencePreview>`;
  - `createCampaign(fields): Promise<Campaign>`, `getCampaign(id): Promise<Campaign>`, `updateCampaign(id, fields: Partial<CampaignFields>): Promise<Campaign>`;
  - `sendCampaign(id)`, `scheduleCampaign(id, sendAtIso)`, `unscheduleCampaign(id)` e `cancelCampaign(id)`, todas `Promise<Campaign>`;
  - `getSmsSetting(): Promise<SmsSetting>`, `setSmsSetting(enabled): Promise<SmsSetting>`.
- Produces (em `src/test/campaignFixtures.tsx`): `OPTIONS: CampaignOptions`, `CITY_AUDIENCE: Audience`, `campaign(overrides?) → Campaign`, `sessionWith(roles, overrides?) → SessionUser`, `renderWithProviders(ui) → { client, ...RenderResult }`.

- [ ] **Step 1: Escreva as fixtures de teste**

```tsx
// src/test/campaignFixtures.tsx
// Dados e harness comuns aos testes do módulo 12. Cada arquivo de teste faz o
// próprio vi.mock("…/lib/api"); este módulo importa só tipos e o AuthProvider,
// que passa a usar a versão mockada do api do arquivo que o importou.
import type { ReactElement, ReactNode } from "react";
import { render } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { AuthProvider } from "../lib/auth";
import type { Audience, Campaign, CampaignOptions, SessionUser } from "../lib/api";

export const OPTIONS: CampaignOptions = {
  protocols: [ "Dor torácica", "Febre" ],
  tiers: [ "vermelha", "amarela" ],
  outcomes: [ "discharged", "referred", "return", "left" ],
  neighborhoods: [ { id: "n1", name: "Boqueirão" }, { id: "n2", name: "Xaxim" }, { id: "n3", name: "Centro" } ],
  units: [ { id: "u1", name: "UBS Centro" }, { id: "u2", name: "UPA Boqueirão" } ]
};

export const CITY_AUDIENCE: Audience = { version: 1, geo: { scope: "city" }, clinical: { all: [] } };

export function campaign(overrides: Partial<Campaign> = {}): Campaign {
  return {
    id: "c1", title: "Vacinação contra a gripe", status: "draft", send_at: null, dispatched_at: null,
    recipients_count: null, body: "Vacinação no sábado, das 8h às 17h, na sua UBS.", audience: CITY_AUDIENCE,
    failure_reason: null, sms_enabled: null, phones_count: null, created_at: "2026-09-28T10:00:00-03:00", stats: null,
    ...overrides
  };
}

// `mfa_verified_at` = agora: a janela de step-up está aberta. Passe
// `{ mfa_verified_at: null }` para forçar o pedido do código.
export function sessionWith(roles: string[], overrides: Partial<SessionUser> = {}): SessionUser {
  return {
    id: "u-1", email_address: "campanhas@curitiba.demo", operator: false,
    memberships: roles.map((role) => ({ municipality_id: "m1", municipality_name: "Curitiba", municipality_uf: "PR", role })),
    mfa_enrolled: true, mfa_verified_at: new Date().toISOString(), ...overrides
  };
}

export function renderWithProviders(ui: ReactElement) {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  const wrapper = ({ children }: { children: ReactNode }) =>
    <QueryClientProvider client={client}><AuthProvider>{children}</AuthProvider></QueryClientProvider>;
  return { client, ...render(ui, { wrapper }) };
}
```

- [ ] **Step 2: Escreva o teste do cliente**

```ts
// src/lib/api.campaigns.test.ts
import { afterEach, describe, expect, it, vi } from "vitest";
import {
  cancelCampaign, createCampaign, getCampaign, getCampaignOptions, getSmsSetting, listCampaigns, previewAudience,
  scheduleCampaign, sendCampaign, setSmsSetting, unscheduleCampaign, updateCampaign
} from "./api";
import { CITY_AUDIENCE, OPTIONS, campaign } from "../test/campaignFixtures";

afterEach(() => vi.unstubAllGlobals());

function stub(body: unknown, status = 200) {
  const fn = vi.fn(async (_input: RequestInfo | URL, _init?: RequestInit) =>
    new Response(JSON.stringify(body), { status, headers: { "Content-Type": "application/json" } }));
  vi.stubGlobal("fetch", fn);
  return fn;
}
const call = (fn: ReturnType<typeof stub>, i = 0) => fn.mock.calls[i] as [ string, RequestInit ];
const contentType = (init: RequestInit) => (init.headers as Record<string, string>)["Content-Type"];

describe("cliente /campaigns", () => {
  it("lista desembrulha { campaigns } e manda o cookie", async () => {
    const fn = stub({ campaigns: [ campaign() ] });
    expect((await listCampaigns()).map((c) => c.title)).toEqual([ "Vacinação contra a gripe" ]);
    expect(call(fn)[0]).toBe("/campaigns");
    expect(call(fn)[1].credentials).toBe("include");
  });

  it("opções vêm soltas, sem envelope", async () => {
    const fn = stub(OPTIONS);
    expect((await getCampaignOptions()).tiers).toEqual([ "vermelha", "amarela" ]);
    expect(call(fn)[0]).toBe("/campaigns/options");
  });

  it("prévia manda o público e devolve a contagem ou below_minimum", async () => {
    const fn = stub({ citizens: 12, phones: 9 });
    expect(await previewAudience(CITY_AUDIENCE)).toEqual({ citizens: 12, phones: 9 });
    expect(call(fn)[0]).toBe("/campaigns/preview");
    expect(call(fn)[1].method).toBe("POST");
    expect(JSON.parse(call(fn)[1].body as string)).toEqual({ audience: CITY_AUDIENCE });

    stub({ below_minimum: true });
    expect(await previewAudience(CITY_AUDIENCE)).toEqual({ below_minimum: true });
  });

  it("cria com POST e desembrulha { campaign } do 201", async () => {
    const fn = stub({ campaign: campaign() }, 201);
    const fields = { title: "Vacinação contra a gripe", body: "Vacinação no sábado.", audience: CITY_AUDIENCE };
    expect((await createCampaign(fields)).id).toBe("c1");
    expect(call(fn)[0]).toBe("/campaigns");
    expect(call(fn)[1].method).toBe("POST");
    expect(JSON.parse(call(fn)[1].body as string)).toEqual(fields);
  });

  it("lê e edita no id escapado; editar é PATCH", async () => {
    const fn = stub({ campaign: campaign({ id: "c/1" }) });
    await getCampaign("c/1");
    await updateCampaign("c/1", { title: "Novo título" });
    expect(call(fn, 0)[0]).toBe("/campaigns/c%2F1");
    expect(call(fn, 1)[0]).toBe("/campaigns/c%2F1");
    expect(call(fn, 1)[1].method).toBe("PATCH");
    expect(JSON.parse(call(fn, 1)[1].body as string)).toEqual({ title: "Novo título" });
  });

  it("enviar, desagendar e cancelar mandam '{}' como JSON (a API recusa escrita sem JSON com 415)", async () => {
    const fn = stub({ campaign: campaign({ status: "sending" }) });
    await sendCampaign("c1");
    await unscheduleCampaign("c1");
    await cancelCampaign("c1");
    expect([ 0, 1, 2 ].map((i) => call(fn, i)[0])).toEqual([
      "/campaigns/c1/send", "/campaigns/c1/unschedule", "/campaigns/c1/cancel"
    ]);
    for (const i of [ 0, 1, 2 ]) {
      expect(call(fn, i)[1].method).toBe("POST");
      expect(call(fn, i)[1].body).toBe("{}");
      expect(contentType(call(fn, i)[1])).toBe("application/json");
    }
  });

  it("agendar manda send_at", async () => {
    const fn = stub({ campaign: campaign({ status: "scheduled", send_at: "2026-09-30T12:00:00.000Z" }) });
    expect((await scheduleCampaign("c1", "2026-09-30T12:00:00.000Z")).status).toBe("scheduled");
    expect(call(fn)[0]).toBe("/campaigns/c1/schedule");
    expect(JSON.parse(call(fn)[1].body as string)).toEqual({ send_at: "2026-09-30T12:00:00.000Z" });
  });

  it("chave de SMS: GET lê, PUT grava enabled", async () => {
    const fn = stub({ enabled: true, gateway_configured: false });
    expect(await getSmsSetting()).toEqual({ enabled: true, gateway_configured: false });
    await setSmsSetting(true);
    expect(call(fn, 0)[0]).toBe("/campaigns/sms_setting");
    expect(call(fn, 1)[0]).toBe("/campaigns/sms_setting");
    expect(call(fn, 1)[1].method).toBe("PUT");
    expect(JSON.parse(call(fn, 1)[1].body as string)).toEqual({ enabled: true });
  });
});
```

- [ ] **Step 3: Rode e veja falhar**

Run: `cd apps/dashboard/.claude/mod12 && npx vitest run src/lib/api.campaigns.test.ts`
Expected: FAIL com "listCampaigns is not a function" (ou erro de import de tipo em `campaignFixtures.tsx`).

- [ ] **Step 4: Cliente e tipos no fim de `src/lib/api.ts`**

```ts
// ─── Campanhas (módulo 12, ADR 0024; spec 2026-09-29 §6.1) ──────────────────
// Tudo sob /campaigns (sessão municipal, banco da cidade) — uma entrada no
// proxy de dev. Nenhuma resposta traz lista de destinatários: só contagens.
// Escrita sem corpo leva "{}" de propósito: a API recusa com 415
// (json_required) escrita por cookie sem application/json.
const CAMPAIGNS_BASE = import.meta.env.VITE_CAMPAIGNS_BASE || "/campaigns";

export type CampaignStatus = "draft" | "scheduled" | "sending" | "sent" | "cancelled" | "failed";
export type SmsStatus = "not_opted_in" | "duplicate_phone" | "pending" | "deferred" | "sent" | "failed" | "unavailable";
export type RequestKind = "return" | "referral";

export type AudienceGeo =
  | { scope: "city" }
  | { scope: "unit"; health_unit_id: string }
  | { scope: "neighborhoods"; neighborhood_ids: string[] };

export interface CriterionPeriod { from: string; to: string }

export type Criterion =
  | ({ kind: "protocol_period"; protocol_name: string } & CriterionPeriod)
  | ({ kind: "triage_tier"; tiers: string[] } & CriterionPeriod)
  | ({ kind: "triage_incomplete" } & CriterionPeriod)
  | ({ kind: "attendance_outcome"; outcomes: AttendanceOutcome[]; health_unit_id?: string } & CriterionPeriod)
  | ({ kind: "triaged_not_attended" } & CriterionPeriod)
  | ({ kind: "appointment_no_show" } & CriterionPeriod)
  | { kind: "appointment_request_open"; kinds?: RequestKind[]; target_unit_id?: string };

export type CriterionKind = Criterion["kind"];

export interface Audience { version: 1; geo: AudienceGeo; clinical: { all: Criterion[] } }

export interface CampaignSummary {
  id: string;
  title: string;
  status: CampaignStatus;
  send_at: string | null;
  dispatched_at: string | null;
  recipients_count: number | null;
}

// null antes do envio; depois, as 7 chaves de SmsStatus sempre presentes.
export interface CampaignStats { read_count: number; sms: Record<SmsStatus, number> }

export interface Campaign extends CampaignSummary {
  body: string;
  audience: Audience;
  failure_reason: "below_minimum" | null;
  sms_enabled: boolean | null;
  phones_count: number | null;
  created_at: string;
  stats: CampaignStats | null;
}

export interface NamedRef { id: string; name: string }
export interface CampaignOptions {
  protocols: string[];
  tiers: string[];
  outcomes: string[];
  neighborhoods: NamedRef[];
  units: NamedRef[];
}
export type AudiencePreview = { citizens: number; phones: number } | { below_minimum: true };
export interface SmsSetting { enabled: boolean; gateway_configured: boolean }
export interface CampaignFields { title: string; body: string; audience: Audience }

const campaignPost = (body: unknown = {}): RequestInit => ({ method: "POST", body: JSON.stringify(body) });

function campaignPath(id: string, action?: string): string {
  return `${CAMPAIGNS_BASE}/${encodeURIComponent(id)}${action ? `/${action}` : ""}`;
}

export async function listCampaigns(): Promise<CampaignSummary[]> {
  return (await jsonFetch<{ campaigns: CampaignSummary[] }>(CAMPAIGNS_BASE)).campaigns;
}

export async function getCampaignOptions(): Promise<CampaignOptions> {
  return jsonFetch<CampaignOptions>(`${CAMPAIGNS_BASE}/options`);
}

export async function previewAudience(audience: Audience): Promise<AudiencePreview> {
  return jsonFetch<AudiencePreview>(`${CAMPAIGNS_BASE}/preview`, campaignPost({ audience }));
}

export async function createCampaign(fields: CampaignFields): Promise<Campaign> {
  return (await jsonFetch<{ campaign: Campaign }>(CAMPAIGNS_BASE, campaignPost(fields))).campaign;
}

export async function getCampaign(id: string): Promise<Campaign> {
  return (await jsonFetch<{ campaign: Campaign }>(campaignPath(id))).campaign;
}

export async function updateCampaign(id: string, fields: Partial<CampaignFields>): Promise<Campaign> {
  return (await jsonFetch<{ campaign: Campaign }>(campaignPath(id), {
    method: "PATCH", body: JSON.stringify(fields)
  })).campaign;
}

export async function sendCampaign(id: string): Promise<Campaign> {
  return (await jsonFetch<{ campaign: Campaign }>(campaignPath(id, "send"), campaignPost())).campaign;
}

export async function scheduleCampaign(id: string, sendAt: string): Promise<Campaign> {
  return (await jsonFetch<{ campaign: Campaign }>(campaignPath(id, "schedule"), campaignPost({ send_at: sendAt }))).campaign;
}

export async function unscheduleCampaign(id: string): Promise<Campaign> {
  return (await jsonFetch<{ campaign: Campaign }>(campaignPath(id, "unschedule"), campaignPost())).campaign;
}

export async function cancelCampaign(id: string): Promise<Campaign> {
  return (await jsonFetch<{ campaign: Campaign }>(campaignPath(id, "cancel"), campaignPost())).campaign;
}

export async function getSmsSetting(): Promise<SmsSetting> {
  return jsonFetch<SmsSetting>(`${CAMPAIGNS_BASE}/sms_setting`);
}

export async function setSmsSetting(enabled: boolean): Promise<SmsSetting> {
  return jsonFetch<SmsSetting>(`${CAMPAIGNS_BASE}/sms_setting`, { method: "PUT", body: JSON.stringify({ enabled }) });
}
```

- [ ] **Step 5: Proxy e README**

Em `vite.config.ts`, na lista do comentário, depois da linha de `/territory`:

```ts
//   /campaigns  → campanhas (módulo 12; campaign_manager, e a chave de SMS também para municipal_admin).
```

e, no objeto `proxy`, troque:

```ts
      "/territory": proxy(TARGET)
```

por:

```ts
      "/territory": proxy(TARGET),
      "/campaigns": proxy(TARGET)
```

Em `README.md`, troque `` `/auth`, `/setup`, `/protocols`, `/mfa`, `/attendance`, `/professionals` e `/territory` para `` por `` `/auth`, `/setup`, `/protocols`, `/mfa`, `/attendance`, `/professionals`, `/territory` e `/campaigns` para ``.

- [ ] **Step 6: Rode, cheque tipos e faça commit**

Run: `cd apps/dashboard/.claude/mod12 && npx vitest run src/lib/api.campaigns.test.ts && npx tsc --noEmit`
Expected: 8 testes PASS; tsc sem erros.

```bash
cd apps/dashboard/.claude/mod12
/opt/homebrew/bin/git add vite.config.ts README.md src/lib/api.ts src/lib/api.campaigns.test.ts src/test/campaignFixtures.tsx
/opt/homebrew/bin/git commit -m "feat: add campaigns API client and contract types

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Regras das campanhas — rótulos, período, validação, JSON do público e recusas

**Files:**
- Create: `src/lib/campaigns.ts`
- Test: `src/lib/campaigns.test.ts`

**Interfaces:**
- Consumes (da Task 1): `ApiError`, `Audience`, `AudienceGeo`, `CampaignStatus`, `Criterion`, `CriterionKind`, `SmsStatus`. De `src/lib/format.ts`: `TIMEZONE`, `parseCityLocal`.
- Produces (em `src/lib/campaigns.ts`):
  - `CAMPAIGN_MANAGER_ROLE = "campaign_manager"`;
  - chaves de cache: `CAMPAIGNS_KEY`, `CAMPAIGN_OPTIONS_KEY`, `SMS_SETTING_KEY`, `PREVIEW_KEY`, `campaignKey(id)`;
  - limites: `PREVIEW_DEBOUNCE_MS = 500`, `MAX_CRITERIA = 7`, `MAX_NEIGHBORHOODS = 50`, `DEFAULT_PERIOD_DAYS = 30`, `TITLE_MIN/TITLE_MAX/BODY_MIN/BODY_MAX`, `SEND_AT_MIN_MS`, `SEND_AT_MAX_MS`;
  - rótulos: `STATUS_LABEL`, `STATUS_TONE`, `FAILURE_LABEL`, `SMS_STATUS_ORDER`, `SMS_STATUS_LABEL`, `CRITERION_LABEL`, `CRITERION_KINDS`, `OUTCOME_LABEL`, `REQUEST_KIND_LABEL`;
  - tipos `CriterionDraft { key; criterion }`, `AudienceDraft { geo; criteria: CriterionDraft[] }`, `SendAtCheck`;
  - rascunho: `nextCriterionKey()`, `emptyAudienceDraft()`, `draftFromAudience(a)`, `isEditable(status)`;
  - datas: `todayInCity(now?) → "YYYY-MM-DD"`, `addDays(date, n)`;
  - regras: `newCriterion(kind, today) → Criterion`, `validateCriterion(c, today) → string | null`, `validateGeo(geo) → string | null`, `audienceProblem(draft, today) → string | null`, `buildAudience(draft) → Audience`, `validateCampaignFields(title, body) → string | null`, `validateSendAt(value, now?) → SendAtCheck`;
  - erros: `campaignErrorCode(err) → string | null`, `campaignError(err) → string`, `campaignErrorOrNull(err) → string | null`.

- [ ] **Step 1: Escreva os testes**

```ts
// src/lib/campaigns.test.ts
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { ApiError, type Criterion } from "./api";
import {
  CRITERION_KINDS, addDays, audienceProblem, buildAudience, campaignError, campaignErrorCode, campaignErrorOrNull,
  draftFromAudience, emptyAudienceDraft, isEditable, newCriterion, todayInCity, validateCampaignFields,
  validateCriterion, validateGeo, validateSendAt, type AudienceDraft
} from "./campaigns";

const TODAY = "2026-09-29";
const draftOf = (geo: AudienceDraft["geo"], criteria: Criterion[] = []): AudienceDraft =>
  ({ geo, criteria: criteria.map((criterion, i) => ({ key: `k${i}`, criterion })) });

describe("hoje na cidade", () => {
  beforeEach(() => vi.useFakeTimers({ toFake: [ "Date" ] }));
  afterEach(() => vi.useRealTimers());

  it("hoje é o dia da cidade, mesmo às 23h30 (já é amanhã em UTC)", () => {
    vi.setSystemTime(new Date("2026-09-29T23:30:00-03:00"));
    expect(todayInCity()).toBe("2026-09-29");
    expect(validateCriterion({ kind: "appointment_no_show", from: "2026-09-29", to: "2026-09-30" }, todayInCity()))
      .toBe("o período não pode terminar no futuro");
  });

  it("addDays atravessa mês sem depender do fuso", () => {
    expect(addDays("2026-09-29", -29)).toBe("2026-08-31");
    expect(addDays("2026-12-31", 1)).toBe("2027-01-01");
  });
});

describe("newCriterion", () => {
  it("os 7 tipos, na ordem do menu", () => {
    expect(CRITERION_KINDS).toEqual([
      "protocol_period", "triage_tier", "triage_incomplete", "attendance_outcome",
      "triaged_not_attended", "appointment_no_show", "appointment_request_open"
    ]);
  });

  it("período padrão: os últimos 30 dias, terminando hoje", () => {
    expect(newCriterion("appointment_no_show", TODAY)).toEqual({ kind: "appointment_no_show", from: "2026-08-31", to: TODAY });
    expect(newCriterion("protocol_period", TODAY)).toEqual({ kind: "protocol_period", protocol_name: "", from: "2026-08-31", to: TODAY });
    expect(newCriterion("triage_tier", TODAY)).toMatchObject({ tiers: [] });
    expect(newCriterion("attendance_outcome", TODAY)).toMatchObject({ outcomes: [] });
  });

  it("pedido de agendamento aberto não tem período", () => {
    expect(newCriterion("appointment_request_open", TODAY)).toEqual({ kind: "appointment_request_open" });
  });
});

describe("validateCriterion", () => {
  it("período: vazio, invertido e no futuro", () => {
    expect(validateCriterion({ kind: "triage_incomplete", from: "", to: TODAY }, TODAY)).toBe("informe o período (de e até)");
    expect(validateCriterion({ kind: "triage_incomplete", from: "2026-09-10", to: "2026-09-01" }, TODAY))
      .toBe("a data inicial é depois da final");
    expect(validateCriterion({ kind: "triage_incomplete", from: "2026-09-01", to: "2026-09-30" }, TODAY))
      .toBe("o período não pode terminar no futuro");
    expect(validateCriterion({ kind: "triage_incomplete", from: TODAY, to: TODAY }, TODAY)).toBeNull();
  });

  it("campos obrigatórios de cada tipo", () => {
    expect(validateCriterion(newCriterion("protocol_period", TODAY), TODAY)).toBe("escolha o protocolo");
    expect(validateCriterion(newCriterion("triage_tier", TODAY), TODAY)).toBe("marque ao menos uma faixa");
    expect(validateCriterion(newCriterion("attendance_outcome", TODAY), TODAY)).toBe("marque ao menos um desfecho");
    expect(validateCriterion(newCriterion("appointment_request_open", TODAY), TODAY)).toBeNull();
  });
});

describe("validateGeo e audienceProblem", () => {
  it("unidade sem escolha e bairros vazios ou demais", () => {
    expect(validateGeo({ scope: "city" })).toBeNull();
    expect(validateGeo({ scope: "unit", health_unit_id: "" })).toBe("escolha a unidade de referência");
    expect(validateGeo({ scope: "neighborhoods", neighborhood_ids: [] })).toBe("escolha ao menos um bairro");
    const many = Array.from({ length: 51 }, (_, i) => `b${i}`);
    expect(validateGeo({ scope: "neighborhoods", neighborhood_ids: many })).toBe("no máximo 50 bairros");
  });

  it("diz qual cartão está incompleto", () => {
    expect(audienceProblem(draftOf({ scope: "city" }, [ newCriterion("triage_tier", TODAY) ]), TODAY))
      .toBe("Faixa da triagem: marque ao menos uma faixa");
    expect(audienceProblem(emptyAudienceDraft(), TODAY)).toBeNull();
  });

  it("mais de 7 critérios", () => {
    const eight = Array.from({ length: 8 }, () => newCriterion("appointment_no_show", TODAY));
    expect(audienceProblem(draftOf({ scope: "city" }, eight), TODAY)).toBe("no máximo 7 critérios");
  });
});

describe("buildAudience", () => {
  it("monta a versão 1 sem as chaves dos cartões", () => {
    const draft = draftOf({ scope: "neighborhoods", neighborhood_ids: [ "n1", "n2", "n1" ] },
      [ { kind: "appointment_no_show", from: "2026-07-01", to: "2026-09-29" } ]);
    expect(buildAudience(draft)).toEqual({
      version: 1,
      geo: { scope: "neighborhoods", neighborhood_ids: [ "n1", "n2" ] },
      clinical: { all: [ { kind: "appointment_no_show", from: "2026-07-01", to: "2026-09-29" } ] }
    });
  });

  it("campo opcional em branco sai do JSON (o schema recusa campo inválido)", () => {
    const draft = draftOf({ scope: "city" }, [
      { kind: "attendance_outcome", outcomes: [ "referred" ], health_unit_id: undefined, from: TODAY, to: TODAY },
      { kind: "appointment_request_open", kinds: [], target_unit_id: undefined }
    ]);
    const json = JSON.parse(JSON.stringify(buildAudience(draft)));
    expect(json.clinical.all[0]).toEqual({ kind: "attendance_outcome", outcomes: [ "referred" ], from: TODAY, to: TODAY });
    expect(json.clinical.all[1]).toEqual({ kind: "appointment_request_open" });
    expect("health_unit_id" in json.clinical.all[0]).toBe(false);
  });

  it("opcional preenchido vai junto", () => {
    const draft = draftOf({ scope: "unit", health_unit_id: "u1" }, [
      { kind: "appointment_request_open", kinds: [ "return" ], target_unit_id: "u2" }
    ]);
    expect(buildAudience(draft).clinical.all[0]).toEqual({ kind: "appointment_request_open", kinds: [ "return" ], target_unit_id: "u2" });
  });

  it("draftFromAudience ida e volta, com chaves únicas", () => {
    const audience = buildAudience(draftOf({ scope: "city" }, [
      newCriterion("appointment_no_show", TODAY), newCriterion("triage_incomplete", TODAY)
    ]));
    const draft = draftFromAudience(audience);
    expect(new Set(draft.criteria.map((d) => d.key)).size).toBe(2);
    expect(buildAudience(draft)).toEqual(audience);
  });
});

describe("campos da campanha e estado", () => {
  it("título de 3 a 120 e texto de 10 a 2000, aparados", () => {
    expect(validateCampaignFields("  ab ", "texto suficiente")).toBe("o título precisa ter de 3 a 120 caracteres");
    expect(validateCampaignFields("a".repeat(121), "texto suficiente")).toBe("o título precisa ter de 3 a 120 caracteres");
    expect(validateCampaignFields("Vacina", " curto    ")).toBe("o texto precisa ter de 10 a 2000 caracteres");
    expect(validateCampaignFields("Vacina", "x".repeat(2001))).toBe("o texto precisa ter de 10 a 2000 caracteres");
    expect(validateCampaignFields(" Vacina ", "Vacinação no sábado.")).toBeNull();
  });

  it("só rascunho é editável", () => {
    expect(isEditable("draft")).toBe(true);
    for (const s of [ "scheduled", "sending", "sent", "cancelled", "failed" ] as const) expect(isEditable(s)).toBe(false);
  });
});

describe("validateSendAt", () => {
  const now = new Date("2026-09-29T10:00:00-03:00");

  it("hora de parede da cidade vira instante", () => {
    expect(validateSendAt("2026-09-29T14:30", now)).toEqual({ ok: true, iso: "2026-09-29T17:30:00.000Z" });
  });

  it("de 5 minutos a 90 dias", () => {
    expect(validateSendAt("2026-09-29T10:04", now)).toEqual({ ok: false, message: "agende para pelo menos 5 minutos a partir de agora" });
    expect(validateSendAt("2026-09-29T10:05", now).ok).toBe(true);
    expect(validateSendAt("2026-12-28T10:00", now).ok).toBe(true);
    expect(validateSendAt("2026-12-28T10:01", now)).toEqual({ ok: false, message: "agende para no máximo 90 dias a partir de agora" });
  });

  it("vazio", () => {
    expect(validateSendAt("", now)).toEqual({ ok: false, message: "informe data e hora do envio" });
  });
});

describe("recusas", () => {
  const err = (status: number, body: unknown) => new ApiError(status, body, String(status));

  it("traduz os códigos do contrato", () => {
    expect(campaignError(err(422, { error: "below_minimum" }))).toBe("o público tem menos de 5 telefones — ajuste o público");
    expect(campaignError(err(422, { error: "not_editable" }))).toBe("esta campanha não é mais rascunho — volte à lista e abra de novo");
    expect(campaignError(err(422, { error: "invalid_transition" })))
      .toBe("a campanha mudou de estado enquanto você decidia — volte à lista e abra de novo");
    expect(campaignError(err(422, { error: "invalid_send_at" })))
      .toBe("horário fora da janela: agende de 5 minutos a 90 dias a partir de agora");
    expect(campaignError(err(422, { error: "invalid_campaign", details: [] })))
      .toBe("confira o título (3 a 120 caracteres) e o texto (10 a 2000)");
    expect(campaignError(err(404, { error: "not_found" }))).toBe("campanha não encontrada — volte à lista");
    expect(campaignError(err(403, { error: "missing_role" }))).toBe("seu papel não permite esta ação");
  });

  it("invalid_audience nomeia o caminho do campo", () => {
    expect(campaignError(err(422, { error: "invalid_audience", details: [ { path: "geo.neighborhood_ids", message: "x" } ] })))
      .toBe("público inválido — confira geo.neighborhood_ids");
    expect(campaignError(err(422, { error: "invalid_audience" }))).toBe("público inválido — confira o recorte e os critérios");
  });

  it("401, rede e código desconhecido", () => {
    expect(campaignError(err(401, {}))).toBe("sessão expirada — entre de novo");
    expect(campaignError(new Error("rede"))).toBe("não foi possível concluir — tente de novo");
    expect(campaignError(err(500, "boom"))).toBe("não foi possível concluir — tente de novo");
  });

  it("variante do SensitiveAction devolve null para o que ela não conhece", () => {
    expect(campaignErrorOrNull(err(401, { error: "mfa_required" }))).toBeNull();
    expect(campaignErrorOrNull(new Error("rede"))).toBeNull();
    expect(campaignErrorOrNull(err(422, { error: "outra_coisa" }))).toBeNull();
    expect(campaignErrorOrNull(err(422, { error: "below_minimum" }))).toBe("o público tem menos de 5 telefones — ajuste o público");
  });

  it("campaignErrorCode lê o código do corpo", () => {
    expect(campaignErrorCode(err(422, { error: "below_minimum" }))).toBe("below_minimum");
    expect(campaignErrorCode(new Error("x"))).toBeNull();
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `cd apps/dashboard/.claude/mod12 && npx vitest run src/lib/campaigns.test.ts`
Expected: FAIL com "Failed to resolve import ./campaigns".

- [ ] **Step 3: Implemente**

```ts
// src/lib/campaigns.ts
// Regras do módulo 12 sem React (spec 2026-09-29 §4 e §7; ADR 0024): chaves
// de cache, rótulos, período padrão dos critérios, validação que espelha o
// schema do api, montagem do JSON do público e tradução das recusas.
import {
  ApiError, type Audience, type AudienceGeo, type CampaignStatus, type Criterion, type CriterionKind, type SmsStatus
} from "./api";
import { TIMEZONE, parseCityLocal } from "./format";

export const CAMPAIGN_MANAGER_ROLE = "campaign_manager";

export const CAMPAIGNS_KEY = [ "campaigns" ] as const;
export const CAMPAIGN_OPTIONS_KEY = [ "campaignOptions" ] as const;
export const SMS_SETTING_KEY = [ "campaignSmsSetting" ] as const;
export const PREVIEW_KEY = [ "campaignPreview" ] as const;
export const campaignKey = (id: string) => [ "campaign", id ] as const;

export const PREVIEW_DEBOUNCE_MS = 500;
export const MAX_CRITERIA = 7;
export const MAX_NEIGHBORHOODS = 50;
export const DEFAULT_PERIOD_DAYS = 30;
export const TITLE_MIN = 3;
export const TITLE_MAX = 120;
export const BODY_MIN = 10;
export const BODY_MAX = 2000;
const MINUTE_MS = 60_000;
export const SEND_AT_MIN_MS = 5 * MINUTE_MS;
export const SEND_AT_MAX_MS = 90 * 24 * 60 * MINUTE_MS;

export const STATUS_LABEL: Record<CampaignStatus, string> = {
  draft: "rascunho", scheduled: "agendada", sending: "enviando", sent: "enviada", cancelled: "cancelada", failed: "falhou"
};
export const STATUS_TONE: Record<CampaignStatus, string | undefined> = {
  draft: undefined, scheduled: "info", sending: "info", sent: "ok", cancelled: undefined, failed: "down"
};
export const FAILURE_LABEL: Record<string, string> = {
  below_minimum: "o público tinha menos de 5 telefones no momento do envio"
};

export const SMS_STATUS_ORDER: SmsStatus[] = [
  "sent", "pending", "deferred", "failed", "unavailable", "duplicate_phone", "not_opted_in"
];
export const SMS_STATUS_LABEL: Record<SmsStatus, string> = {
  sent: "enviado",
  pending: "na fila",
  deferred: "aguardando o horário (8h–20h)",
  failed: "falhou",
  unavailable: "não enviado — sem provedor de SMS",
  duplicate_phone: "telefone já recebeu por outra pessoa",
  not_opted_in: "sem opt-in de SMS"
};

export const CRITERION_LABEL: Record<CriterionKind, string> = {
  protocol_period: "Protocolo e período",
  triage_tier: "Faixa da triagem",
  triage_incomplete: "Triagem não concluída",
  attendance_outcome: "Desfecho de atendimento",
  triaged_not_attended: "Triado e não atendido",
  appointment_no_show: "Falta em agendamento",
  appointment_request_open: "Pedido de agendamento aberto"
};
export const CRITERION_KINDS = Object.keys(CRITERION_LABEL) as CriterionKind[];

export const OUTCOME_LABEL: Record<string, string> = {
  discharged: "atendido e liberado", referred: "encaminhado", return: "retorno", left: "saiu sem atendimento"
};
export const REQUEST_KIND_LABEL: Record<string, string> = { return: "retorno", referral: "encaminhamento" };

// ─── Rascunho do público ─────────────────────────────────────────────────────
// Cada cartão tem uma chave só da tela (React key); ela nunca vai à API.
export interface CriterionDraft { key: string; criterion: Criterion }
export interface AudienceDraft { geo: AudienceGeo; criteria: CriterionDraft[] }

let keySeq = 0;
export function nextCriterionKey(): string {
  keySeq += 1;
  return `criterion-${keySeq}`;
}

export function emptyAudienceDraft(): AudienceDraft {
  return { geo: { scope: "city" }, criteria: [] };
}

export function draftFromAudience(audience: Audience): AudienceDraft {
  return { geo: audience.geo, criteria: audience.clinical.all.map((criterion) => ({ key: nextCriterionKey(), criterion })) };
}

export function isEditable(status: CampaignStatus): boolean {
  return status === "draft";
}

// ─── Datas ──────────────────────────────────────────────────────────────────
// "Hoje" é o dia da CIDADE: às 23h30 em São Paulo já é amanhã em UTC, e o
// limite "não pode terminar no futuro" do api é no fuso da cidade (spec §4.1).
const dayParts = new Intl.DateTimeFormat("en-US", { timeZone: TIMEZONE, year: "numeric", month: "2-digit", day: "2-digit" });

export function todayInCity(now: Date = new Date()): string {
  const p = Object.fromEntries(dayParts.formatToParts(now).map((x) => [ x.type, x.value ]));
  return `${p.year}-${p.month}-${p.day}`;
}

// Aritmética de dia em UTC ao meio-dia: nenhum fuso desloca a data.
export function addDays(date: string, days: number): string {
  const d = new Date(`${date}T12:00:00Z`);
  d.setUTCDate(d.getUTCDate() + days);
  return d.toISOString().slice(0, 10);
}

// ─── Critérios ──────────────────────────────────────────────────────────────
export function newCriterion(kind: CriterionKind, today: string): Criterion {
  const from = addDays(today, -(DEFAULT_PERIOD_DAYS - 1));
  switch (kind) {
    case "protocol_period": return { kind, protocol_name: "", from, to: today };
    case "triage_tier": return { kind, tiers: [], from, to: today };
    case "triage_incomplete": return { kind, from, to: today };
    case "attendance_outcome": return { kind, outcomes: [], from, to: today };
    case "triaged_not_attended": return { kind, from, to: today };
    case "appointment_no_show": return { kind, from, to: today };
    case "appointment_request_open": return { kind };
  }
}

// Espelha o schema do api (spec §4.1). Datas em "YYYY-MM-DD" comparam como
// texto.
export function validateCriterion(c: Criterion, today: string): string | null {
  if ("from" in c) {
    if (!c.from || !c.to) return "informe o período (de e até)";
    if (c.from > c.to) return "a data inicial é depois da final";
    if (c.to > today) return "o período não pode terminar no futuro";
  }
  if (c.kind === "protocol_period" && !c.protocol_name) return "escolha o protocolo";
  if (c.kind === "triage_tier" && c.tiers.length === 0) return "marque ao menos uma faixa";
  if (c.kind === "attendance_outcome" && c.outcomes.length === 0) return "marque ao menos um desfecho";
  return null;
}

export function validateGeo(geo: AudienceGeo): string | null {
  if (geo.scope === "unit" && !geo.health_unit_id) return "escolha a unidade de referência";
  if (geo.scope === "neighborhoods") {
    if (geo.neighborhood_ids.length === 0) return "escolha ao menos um bairro";
    if (geo.neighborhood_ids.length > MAX_NEIGHBORHOODS) return `no máximo ${MAX_NEIGHBORHOODS} bairros`;
  }
  return null;
}

export function audienceProblem(draft: AudienceDraft, today: string): string | null {
  const geo = validateGeo(draft.geo);
  if (geo) return geo;
  if (draft.criteria.length > MAX_CRITERIA) return `no máximo ${MAX_CRITERIA} critérios`;
  for (const { criterion } of draft.criteria) {
    const problem = validateCriterion(criterion, today);
    if (problem) return `${CRITERION_LABEL[criterion.kind]}: ${problem}`;
  }
  return null;
}

// ─── JSON do público ────────────────────────────────────────────────────────
// O schema do api recusa campo extra e valor inválido: opcional em branco
// SAI do objeto (nunca "", null ou []), e bairro repetido vira um só.
function cleanGeo(geo: AudienceGeo): AudienceGeo {
  switch (geo.scope) {
    case "city": return { scope: "city" };
    case "unit": return { scope: "unit", health_unit_id: geo.health_unit_id };
    case "neighborhoods": return { scope: "neighborhoods", neighborhood_ids: [ ...new Set(geo.neighborhood_ids) ] };
  }
}

function cleanCriterion(c: Criterion): Criterion {
  if (c.kind === "attendance_outcome") {
    const { health_unit_id, ...rest } = c;
    return health_unit_id ? { ...rest, health_unit_id } : rest;
  }
  if (c.kind === "appointment_request_open") {
    return {
      kind: c.kind,
      ...(c.kinds && c.kinds.length > 0 ? { kinds: c.kinds } : {}),
      ...(c.target_unit_id ? { target_unit_id: c.target_unit_id } : {})
    };
  }
  return c;
}

export function buildAudience(draft: AudienceDraft): Audience {
  return {
    version: 1,
    geo: cleanGeo(draft.geo),
    clinical: { all: draft.criteria.map((d) => cleanCriterion(d.criterion)) }
  };
}

// ─── Campanha ───────────────────────────────────────────────────────────────
export function validateCampaignFields(title: string, body: string): string | null {
  const t = title.trim().length;
  const b = body.trim().length;
  if (t < TITLE_MIN || t > TITLE_MAX) return `o título precisa ter de ${TITLE_MIN} a ${TITLE_MAX} caracteres`;
  if (b < BODY_MIN || b > BODY_MAX) return `o texto precisa ter de ${BODY_MIN} a ${BODY_MAX} caracteres`;
  return null;
}

export type SendAtCheck = { ok: true; iso: string } | { ok: false; message: string };

// Valor de <input type="datetime-local"> lido como hora de parede da cidade.
// É uma previsão: o api confere de novo no clique, e `invalid_send_at` volta
// traduzido por campaignError.
export function validateSendAt(value: string, now: Date = new Date()): SendAtCheck {
  const at = parseCityLocal(value);
  if (!at) return { ok: false, message: "informe data e hora do envio" };
  const delta = at.getTime() - now.getTime();
  if (delta < SEND_AT_MIN_MS) return { ok: false, message: "agende para pelo menos 5 minutos a partir de agora" };
  if (delta > SEND_AT_MAX_MS) return { ok: false, message: "agende para no máximo 90 dias a partir de agora" };
  return { ok: true, iso: at.toISOString() };
}

// ─── Recusas ────────────────────────────────────────────────────────────────
const MESSAGES: Record<string, string> = {
  missing_role: "seu papel não permite esta ação",
  invalid_campaign: `confira o título (${TITLE_MIN} a ${TITLE_MAX} caracteres) e o texto (${BODY_MIN} a ${BODY_MAX})`,
  below_minimum: "o público tem menos de 5 telefones — ajuste o público",
  not_editable: "esta campanha não é mais rascunho — volte à lista e abra de novo",
  invalid_transition: "a campanha mudou de estado enquanto você decidia — volte à lista e abra de novo",
  invalid_send_at: "horário fora da janela: agende de 5 minutos a 90 dias a partir de agora",
  not_found: "campanha não encontrada — volte à lista"
};
const GENERIC = "não foi possível concluir — tente de novo";

type CampaignErrorBody = { error?: string; details?: { path?: string; message?: string }[] };

function errorBody(err: ApiError): CampaignErrorBody {
  return (err.body && typeof err.body === "object" ? err.body : {}) as CampaignErrorBody;
}

function translate(body: CampaignErrorBody): string | null {
  if (body.error === "invalid_audience") {
    const path = body.details?.[0]?.path;
    return path ? `público inválido — confira ${path}` : "público inválido — confira o recorte e os critérios";
  }
  return (body.error && MESSAGES[body.error]) || null;
}

export function campaignErrorCode(err: unknown): string | null {
  if (!(err instanceof ApiError)) return null;
  const code = errorBody(err).error;
  return typeof code === "string" ? code : null;
}

export function campaignError(err: unknown): string {
  if (!(err instanceof ApiError)) return GENERIC;
  if (err.status === 401) return "sessão expirada — entre de novo";
  return translate(errorBody(err)) ?? GENERIC;
}

// Para SensitiveAction.translateError: null deixa a frase padrão dele (rede,
// sessão expirada, mfa_required) aparecer.
export function campaignErrorOrNull(err: unknown): string | null {
  if (!(err instanceof ApiError) || err.status === 401) return null;
  return translate(errorBody(err));
}
```

- [ ] **Step 4: Rode, cheque tipos e faça commit**

Run: `cd apps/dashboard/.claude/mod12 && npx vitest run src/lib/campaigns.test.ts && npx tsc --noEmit`
Expected: PASS; tsc sem erros.

```bash
cd apps/dashboard/.claude/mod12
/opt/homebrew/bin/git add src/lib/campaigns.ts src/lib/campaigns.test.ts
/opt/homebrew/bin/git commit -m "feat: add campaign audience rules, validation and error messages

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Público em frase

**Files:**
- Create: `src/lib/audiencePhrase.ts`
- Test: `src/lib/audiencePhrase.test.ts`

**Interfaces:**
- Consumes:
  - da Task 1: `Audience`, `AudienceGeo`, `Criterion`, `NamedRef`;
  - da Task 2: `OUTCOME_LABEL`, `REQUEST_KIND_LABEL`.
- Produces:
  - `PhraseLookup { neighborhoods: NamedRef[]; units: NamedRef[] }` (um `CampaignOptions` serve);
  - `fmtDay("YYYY-MM-DD") → "DD/MM/AAAA"`;
  - `joinPt(items, last = "e")`;
  - `describeAudience(audience, lookup) → string`.

- [ ] **Step 1: Escreva os testes**

```ts
// src/lib/audiencePhrase.test.ts
import { describe, expect, it } from "vitest";
import type { Audience, Criterion } from "./api";
import { describeAudience, fmtDay, joinPt } from "./audiencePhrase";
import { OPTIONS } from "../test/campaignFixtures";

const aud = (geo: Audience["geo"], all: Criterion[] = []): Audience => ({ version: 1, geo, clinical: { all } });
const city = (c: Criterion) => describeAudience(aud({ scope: "city" }, [ c ]), OPTIONS);

describe("fmtDay e joinPt", () => {
  it("data sem deslocar o dia (nunca lida como UTC)", () => {
    expect(fmtDay("2026-07-01")).toBe("01/07/2026");
    expect(fmtDay("2026-12-31")).toBe("31/12/2026");
  });

  it("lista em português", () => {
    expect(joinPt([ "A" ])).toBe("A");
    expect(joinPt([ "A", "B" ])).toBe("A e B");
    expect(joinPt([ "A", "B", "C" ])).toBe("A, B e C");
    expect(joinPt([ "A", "B" ], "ou")).toBe("A ou B");
  });
});

describe("describeAudience", () => {
  it("o exemplo da spec", () => {
    expect(describeAudience(aud({ scope: "neighborhoods", neighborhood_ids: [ "n1", "n2" ] },
      [ { kind: "appointment_no_show", from: "2026-07-01", to: "2026-09-29" } ]), OPTIONS))
      .toBe("moradores de Boqueirão e Xaxim que faltaram a um agendamento entre 01/07/2026 e 29/09/2026");
  });

  it("recortes sem critério", () => {
    expect(describeAudience(aud({ scope: "city" }), OPTIONS)).toBe("cidadãos de toda a cidade");
    expect(describeAudience(aud({ scope: "unit", health_unit_id: "u1" }), OPTIONS))
      .toBe("moradores da área de cobertura de UBS Centro");
    expect(describeAudience(aud({ scope: "neighborhoods", neighborhood_ids: [ "n1", "n2", "n3" ] }), OPTIONS))
      .toBe("moradores de Boqueirão, Xaxim e Centro");
  });

  it("bairro que não está na lista (desativado depois do rascunho)", () => {
    expect(describeAudience(aud({ scope: "neighborhoods", neighborhood_ids: [ "n1", "sumiu" ] }), OPTIONS))
      .toBe("moradores de Boqueirão e (bairro inativo)");
    expect(describeAudience(aud({ scope: "unit", health_unit_id: "sumiu" }), OPTIONS))
      .toBe("moradores da área de cobertura de (unidade inativa)");
  });

  it("uma frase por tipo de critério", () => {
    const p = { from: "2026-07-01", to: "2026-07-31" };
    expect(city({ kind: "protocol_period", protocol_name: "Febre", ...p }))
      .toBe("cidadãos de toda a cidade que fizeram triagem concluída pelo protocolo Febre entre 01/07/2026 e 31/07/2026");
    expect(city({ kind: "triage_tier", tiers: [ "vermelha", "amarela" ], ...p }))
      .toBe("cidadãos de toda a cidade que tiveram triagem concluída na faixa vermelha ou amarela entre 01/07/2026 e 31/07/2026");
    expect(city({ kind: "triage_incomplete", ...p }))
      .toBe("cidadãos de toda a cidade que deixaram uma triagem sem concluir entre 01/07/2026 e 31/07/2026");
    expect(city({ kind: "attendance_outcome", outcomes: [ "referred" ], health_unit_id: "u2", ...p }))
      .toBe("cidadãos de toda a cidade que tiveram atendimento encerrado como encaminhado em UPA Boqueirão entre 01/07/2026 e 31/07/2026");
    expect(city({ kind: "triaged_not_attended", ...p }))
      .toBe("cidadãos de toda a cidade que fizeram triagem concluída entre 01/07/2026 e 31/07/2026 e não foram atendidos depois dela");
    expect(city({ kind: "appointment_request_open" }))
      .toBe("cidadãos de toda a cidade que têm pedido de agendamento aberto");
    expect(city({ kind: "appointment_request_open", kinds: [ "return" ], target_unit_id: "u1" }))
      .toBe("cidadãos de toda a cidade que têm pedido de agendamento de retorno aberto para UBS Centro");
  });

  it("período de um dia só", () => {
    expect(city({ kind: "appointment_no_show", from: "2026-07-01", to: "2026-07-01" }))
      .toBe("cidadãos de toda a cidade que faltaram a um agendamento em 01/07/2026");
  });

  it("critérios somados por E, como no construtor", () => {
    expect(describeAudience(aud({ scope: "neighborhoods", neighborhood_ids: [ "n2" ] }, [
      { kind: "appointment_no_show", from: "2026-07-01", to: "2026-07-31" },
      { kind: "appointment_request_open" }
    ]), OPTIONS)).toBe(
      "moradores de Xaxim que faltaram a um agendamento entre 01/07/2026 e 31/07/2026 " +
      "e também têm pedido de agendamento aberto"
    );
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `cd apps/dashboard/.claude/mod12 && npx vitest run src/lib/audiencePhrase.test.ts`
Expected: FAIL com "Failed to resolve import ./audiencePhrase".

- [ ] **Step 3: Implemente**

```ts
// src/lib/audiencePhrase.ts
// O público em português (spec 2026-09-29 §7): no modal de envio e no painel
// da campanha. Os nomes vêm de GET /campaigns/options (bairros e unidades
// ATIVOS); id que não está lá é nomeado como inativo, nunca some da frase.
import type { Audience, AudienceGeo, Criterion, CriterionPeriod, NamedRef } from "./api";
import { OUTCOME_LABEL, REQUEST_KIND_LABEL } from "./campaigns";

export interface PhraseLookup { neighborhoods: NamedRef[]; units: NamedRef[] }

// "YYYY-MM-DD" → "DD/MM/AAAA" por texto: new Date("2026-07-01") seria meia-noite
// UTC, que em São Paulo ainda é 30/06.
export function fmtDay(date: string): string {
  const m = /^(\d{4})-(\d{2})-(\d{2})$/.exec(date);
  return m ? `${m[3]}/${m[2]}/${m[1]}` : date;
}

export function joinPt(items: string[], last = "e"): string {
  if (items.length <= 1) return items[0] ?? "";
  return `${items.slice(0, -1).join(", ")} ${last} ${items[items.length - 1]}`;
}

function nameOf(list: NamedRef[], id: string, fallback: string): string {
  return list.find((x) => x.id === id)?.name ?? fallback;
}

function period(p: CriterionPeriod): string {
  return p.from === p.to ? `em ${fmtDay(p.from)}` : `entre ${fmtDay(p.from)} e ${fmtDay(p.to)}`;
}

function geoPhrase(geo: AudienceGeo, lookup: PhraseLookup): string {
  switch (geo.scope) {
    case "city": return "cidadãos de toda a cidade";
    case "unit": return `moradores da área de cobertura de ${nameOf(lookup.units, geo.health_unit_id, "(unidade inativa)")}`;
    case "neighborhoods":
      return `moradores de ${joinPt(geo.neighborhood_ids.map((id) => nameOf(lookup.neighborhoods, id, "(bairro inativo)")))}`;
  }
}

function criterionPhrase(c: Criterion, lookup: PhraseLookup): string {
  const unit = (id: string) => nameOf(lookup.units, id, "(unidade inativa)");
  switch (c.kind) {
    case "protocol_period":
      return `fizeram triagem concluída pelo protocolo ${c.protocol_name} ${period(c)}`;
    case "triage_tier":
      return `tiveram triagem concluída na faixa ${joinPt(c.tiers, "ou")} ${period(c)}`;
    case "triage_incomplete":
      return `deixaram uma triagem sem concluir ${period(c)}`;
    case "attendance_outcome": {
      const outcomes = joinPt(c.outcomes.map((o) => OUTCOME_LABEL[o] ?? o), "ou");
      return `tiveram atendimento encerrado como ${outcomes}${c.health_unit_id ? ` em ${unit(c.health_unit_id)}` : ""} ${period(c)}`;
    }
    case "triaged_not_attended":
      return `fizeram triagem concluída ${period(c)} e não foram atendidos depois dela`;
    case "appointment_no_show":
      return `faltaram a um agendamento ${period(c)}`;
    case "appointment_request_open": {
      const kinds = c.kinds && c.kinds.length > 0 ? ` de ${joinPt(c.kinds.map((k) => REQUEST_KIND_LABEL[k] ?? k), "ou")}` : "";
      return `têm pedido de agendamento${kinds} aberto${c.target_unit_id ? ` para ${unit(c.target_unit_id)}` : ""}`;
    }
  }
}

export function describeAudience(audience: Audience, lookup: PhraseLookup): string {
  const geo = geoPhrase(audience.geo, lookup);
  const criteria = audience.clinical.all.map((c) => criterionPhrase(c, lookup));
  return criteria.length === 0 ? geo : `${geo} que ${criteria.join(" e também ")}`;
}
```

- [ ] **Step 4: Rode, cheque tipos e faça commit**

Run: `cd apps/dashboard/.claude/mod12 && npx vitest run src/lib/audiencePhrase.test.ts && npx tsc --noEmit`
Expected: PASS; tsc sem erros.

```bash
cd apps/dashboard/.claude/mod12
/opt/homebrew/bin/git add src/lib/audiencePhrase.ts src/lib/audiencePhrase.test.ts
/opt/homebrew/bin/git commit -m "feat: describe a campaign audience as a Portuguese sentence

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Cartão de critério clínico

**Files:**
- Create: `src/modules/campaigns/styles.ts`, `src/modules/campaigns/CriterionCard.tsx`
- Test: `src/modules/campaigns/CriterionCard.test.tsx`

**Interfaces:**
- Consumes:
  - da Task 1: `AttendanceOutcome`, `CampaignOptions`, `Criterion`, `CriterionPeriod`, `NamedRef`, `RequestKind`;
  - da Task 2: `CRITERION_LABEL`, `OUTCOME_LABEL`, `REQUEST_KIND_LABEL`, `validateCriterion`;
  - de `src/lib/territory.ts`: `sortByName`.
- Produces:
  - `CriterionCard(props: CriterionCardProps)`, com `CriterionCardProps { criterion; options; today; onChange(next: Criterion); onRemove() }`;
  - `toggleValue<T>(list, value) → T[]`;
  - estilos em `styles.ts`: `labelStyle`, `alertStyle`, `noteStyle`, `warnStyle`, `cardStyle`, `fieldsetStyle`, `legendStyle`, `checkStyle`, `rowStyle`, `columnStyle`.
  - O cartão é uma `section` com `aria-label` = rótulo do tipo, e o botão de remover tem o nome `Remover <rótulo>`.

- [ ] **Step 1: Escreva os testes**

```tsx
// src/modules/campaigns/CriterionCard.test.tsx
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen } from "@testing-library/react";
import type { Criterion } from "../../lib/api";
import { CriterionCard } from "./CriterionCard";
import { OPTIONS } from "../../test/campaignFixtures";

afterEach(cleanup);
const TODAY = "2026-09-29";
const P = { from: "2026-09-01", to: "2026-09-29" };

function renderCard(criterion: Criterion, options = OPTIONS) {
  const onChange = vi.fn();
  const onRemove = vi.fn();
  render(<CriterionCard criterion={criterion} options={options} today={TODAY} onChange={onChange} onRemove={onRemove} />);
  return { onChange, onRemove };
}

describe("CriterionCard", () => {
  it("protocolo: escolhe da lista de opções", () => {
    const { onChange } = renderCard({ kind: "protocol_period", protocol_name: "", ...P });
    expect(screen.getByRole("region", { name: "Protocolo e período" })).toBeTruthy();
    expect(screen.getByRole("alert").textContent).toBe("escolha o protocolo");
    fireEvent.change(screen.getByLabelText("Protocolo"), { target: { value: "Febre" } });
    expect(onChange).toHaveBeenCalledWith({ kind: "protocol_period", protocol_name: "Febre", ...P });
  });

  it("protocolo: sem triagem concluída na cidade, diz isso", () => {
    renderCard({ kind: "protocol_period", protocol_name: "", ...P }, { ...OPTIONS, protocols: [] });
    expect(screen.getByText("nenhum protocolo com triagem concluída ainda")).toBeTruthy();
  });

  it("faixa: marca e desmarca", () => {
    const { onChange } = renderCard({ kind: "triage_tier", tiers: [ "amarela" ], ...P });
    fireEvent.click(screen.getByRole("checkbox", { name: "vermelha" }));
    expect(onChange).toHaveBeenLastCalledWith({ kind: "triage_tier", tiers: [ "amarela", "vermelha" ], ...P });
    fireEvent.click(screen.getByRole("checkbox", { name: "amarela" }));
    expect(onChange).toHaveBeenLastCalledWith({ kind: "triage_tier", tiers: [], ...P });
  });

  it("desfecho: rótulos em português e unidade opcional", () => {
    const { onChange } = renderCard({ kind: "attendance_outcome", outcomes: [ "referred" ], ...P });
    expect((screen.getByRole("checkbox", { name: "encaminhado" }) as HTMLInputElement).checked).toBe(true);
    fireEvent.change(screen.getByLabelText("Unidade do atendimento (opcional)"), { target: { value: "u1" } });
    expect(onChange).toHaveBeenLastCalledWith({ kind: "attendance_outcome", outcomes: [ "referred" ], health_unit_id: "u1", ...P });
  });

  it("desfecho: voltar para 'qualquer unidade' limpa o campo", () => {
    const { onChange } = renderCard({ kind: "attendance_outcome", outcomes: [ "referred" ], health_unit_id: "u1", ...P });
    fireEvent.change(screen.getByLabelText("Unidade do atendimento (opcional)"), { target: { value: "" } });
    expect(onChange.mock.calls[0][0].health_unit_id).toBeUndefined();
  });

  it("pedido aberto: sem período, tipo e destino opcionais", () => {
    const { onChange } = renderCard({ kind: "appointment_request_open" });
    expect(screen.queryByLabelText("De")).toBeNull();
    fireEvent.click(screen.getByRole("checkbox", { name: "retorno" }));
    expect(onChange).toHaveBeenLastCalledWith({ kind: "appointment_request_open", kinds: [ "return" ] });
    fireEvent.change(screen.getByLabelText("Unidade de destino (opcional)"), { target: { value: "u2" } });
    expect(onChange).toHaveBeenLastCalledWith({ kind: "appointment_request_open", target_unit_id: "u2" });
  });

  it("período: não deixa escolher depois de hoje e explica a recusa", () => {
    const { onChange } = renderCard({ kind: "appointment_no_show", ...P });
    const until = screen.getByLabelText("Até") as HTMLInputElement;
    expect(until.max).toBe(TODAY);
    fireEvent.change(until, { target: { value: "2026-09-20" } });
    expect(onChange).toHaveBeenCalledWith({ kind: "appointment_no_show", from: "2026-09-01", to: "2026-09-20" });
    cleanup();

    renderCard({ kind: "appointment_no_show", from: "2026-09-01", to: "2026-09-30" });
    expect(screen.getByRole("alert").textContent).toBe("o período não pode terminar no futuro");
    cleanup();

    renderCard({ kind: "appointment_no_show", from: "2026-09-10", to: "2026-09-01" });
    expect(screen.getByRole("alert").textContent).toBe("a data inicial é depois da final");
  });

  it("remover", () => {
    const { onRemove } = renderCard({ kind: "appointment_no_show", ...P });
    fireEvent.click(screen.getByRole("button", { name: "Remover Falta em agendamento" }));
    expect(onRemove).toHaveBeenCalledTimes(1);
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `cd apps/dashboard/.claude/mod12 && npx vitest run src/modules/campaigns/CriterionCard.test.tsx`
Expected: FAIL com "Failed to resolve import ./CriterionCard".

- [ ] **Step 3: Estilos comuns**

```ts
// src/modules/campaigns/styles.ts
// Estilos das telas de campanha (mesma linguagem de Territory e Team).
import type { CSSProperties } from "react";

export const labelStyle: CSSProperties = { display: "flex", flexDirection: "column", gap: 4, fontSize: 12, color: "var(--ink2)" };
export const alertStyle: CSSProperties = { margin: 0, fontSize: 12.5, color: "var(--down)" };
export const noteStyle: CSSProperties = { margin: 0, fontSize: 12, color: "var(--ink3)" };
export const warnStyle: CSSProperties = {
  margin: 0, padding: "8px 10px", fontSize: 13, fontWeight: 600, color: "var(--warn)",
  border: "1px solid var(--warn)", borderRadius: 6
};
export const cardStyle: CSSProperties = {
  display: "flex", flexDirection: "column", gap: 10, padding: 12,
  border: "1px solid var(--rule)", borderRadius: 8, background: "var(--panel)"
};
export const fieldsetStyle: CSSProperties = {
  display: "flex", flexWrap: "wrap", gap: 10, margin: 0, padding: "8px 10px",
  border: "1px solid var(--rule)", borderRadius: 6
};
export const legendStyle: CSSProperties = { fontSize: 12, color: "var(--ink2)", padding: "0 4px" };
export const checkStyle: CSSProperties = { display: "flex", gap: 6, alignItems: "center", fontSize: 12.5 };
export const rowStyle: CSSProperties = { display: "flex", gap: 8, alignItems: "flex-end", flexWrap: "wrap" };
export const columnStyle: CSSProperties = { display: "flex", flexDirection: "column", gap: 12 };
```

- [ ] **Step 4: Cartão**

```tsx
// src/modules/campaigns/CriterionCard.tsx
import type { AttendanceOutcome, CampaignOptions, Criterion, CriterionPeriod, NamedRef, RequestKind } from "../../lib/api";
import { CRITERION_LABEL, OUTCOME_LABEL, REQUEST_KIND_LABEL, validateCriterion } from "../../lib/campaigns";
import { sortByName } from "../../lib/territory";
import { inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { alertStyle, cardStyle, checkStyle, fieldsetStyle, labelStyle, legendStyle, noteStyle, rowStyle } from "./styles";

// Um cartão por critério clínico (spec 2026-09-29 §7; construtor A, D10).
// Cada tipo tem os próprios campos, e o problema do cartão aparece nele,
// com a mesma regra que trava a contagem (validateCriterion).
export interface CriterionCardProps {
  criterion: Criterion;
  options: CampaignOptions;
  today: string;
  onChange(next: Criterion): void;
  onRemove(): void;
}

const REQUEST_KINDS: RequestKind[] = [ "return", "referral" ];

export function toggleValue<T>(list: T[], value: T): T[] {
  return list.includes(value) ? list.filter((v) => v !== value) : [ ...list, value ];
}

export function CriterionCard({ criterion: c, options, today, onChange, onRemove }: CriterionCardProps) {
  const title = CRITERION_LABEL[c.kind];
  const problem = validateCriterion(c, today);

  return (
    <section aria-label={title} style={cardStyle}>
      <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", gap: 8 }}>
        <strong style={{ fontSize: 13 }}>{title}</strong>
        <button type="button" aria-label={`Remover ${title}`} onClick={onRemove} style={secondaryButtonStyle}>remover</button>
      </div>

      {c.kind === "protocol_period" && (
        options.protocols.length === 0
          ? <p style={noteStyle}>nenhum protocolo com triagem concluída ainda</p>
          : (
            <label style={labelStyle}>
              Protocolo
              <select value={c.protocol_name} onChange={(e) => onChange({ ...c, protocol_name: e.target.value })} style={inputStyle}>
                <option value="">escolha…</option>
                {options.protocols.map((p) => <option key={p} value={p}>{p}</option>)}
              </select>
            </label>
          )
      )}

      {c.kind === "triage_tier" && (
        <CheckboxGroup legend="Faixas" values={options.tiers} selected={c.tiers} label={(t) => t}
          emptyText="nenhuma faixa registrada ainda"
          onToggle={(t) => onChange({ ...c, tiers: toggleValue(c.tiers, t) })} />
      )}

      {c.kind === "attendance_outcome" && (
        <>
          <CheckboxGroup legend="Desfechos" values={options.outcomes} selected={c.outcomes}
            label={(o) => OUTCOME_LABEL[o] ?? o} emptyText="nenhum desfecho disponível"
            onToggle={(o) => onChange({ ...c, outcomes: toggleValue(c.outcomes, o as AttendanceOutcome) })} />
          <UnitSelect label="Unidade do atendimento (opcional)" value={c.health_unit_id} units={options.units}
            onChange={(id) => onChange({ ...c, health_unit_id: id })} />
        </>
      )}

      {c.kind === "appointment_request_open" && (
        <>
          <CheckboxGroup legend="Tipo do pedido (opcional — nenhum marcado vale os dois)" values={REQUEST_KINDS}
            selected={c.kinds ?? []} label={(k) => REQUEST_KIND_LABEL[k] ?? k}
            onToggle={(k) => onChange({ ...c, kinds: toggleValue<RequestKind>(c.kinds ?? [], k as RequestKind) })} />
          <UnitSelect label="Unidade de destino (opcional)" value={c.target_unit_id} units={options.units}
            onChange={(id) => onChange({ ...c, target_unit_id: id })} />
        </>
      )}

      {"from" in c && (
        <PeriodFields period={c} today={today} onChange={(p) => onChange({ ...c, ...p })} />
      )}

      {problem && <p role="alert" style={alertStyle}>{problem}</p>}
    </section>
  );
}

function CheckboxGroup({ legend, values, selected, label, emptyText, onToggle }: {
  legend: string; values: string[]; selected: string[]; label(v: string): string; emptyText?: string; onToggle(v: string): void;
}) {
  return (
    <fieldset style={fieldsetStyle}>
      <legend style={legendStyle}>{legend}</legend>
      {values.length === 0 && emptyText && <p style={noteStyle}>{emptyText}</p>}
      {values.map((v) => (
        <label key={v} style={checkStyle}>
          <input type="checkbox" checked={selected.includes(v)} onChange={() => onToggle(v)} />
          {label(v)}
        </label>
      ))}
    </fieldset>
  );
}

// "" = qualquer unidade: sai como undefined, e buildAudience tira a chave.
function UnitSelect({ label, value, units, onChange }: {
  label: string; value: string | undefined; units: NamedRef[]; onChange(id: string | undefined): void;
}) {
  return (
    <label style={labelStyle}>
      {label}
      <select value={value ?? ""} onChange={(e) => onChange(e.target.value || undefined)} style={inputStyle}>
        <option value="">qualquer unidade</option>
        {sortByName(units).map((u) => <option key={u.id} value={u.id}>{u.name}</option>)}
      </select>
    </label>
  );
}

function PeriodFields({ period, today, onChange }: {
  period: CriterionPeriod; today: string; onChange(p: CriterionPeriod): void;
}) {
  return (
    <div style={rowStyle}>
      <label style={labelStyle}>
        De
        <input type="date" value={period.from} max={today}
          onChange={(e) => onChange({ from: e.target.value, to: period.to })} style={inputStyle} />
      </label>
      <label style={labelStyle}>
        Até
        <input type="date" value={period.to} max={today}
          onChange={(e) => onChange({ from: period.from, to: e.target.value })} style={inputStyle} />
      </label>
    </div>
  );
}
```

- [ ] **Step 5: Rode, cheque tipos e faça commit**

Run: `cd apps/dashboard/.claude/mod12 && npx vitest run src/modules/campaigns/CriterionCard.test.tsx && npx tsc --noEmit`
Expected: PASS; tsc sem erros.

```bash
cd apps/dashboard/.claude/mod12
/opt/homebrew/bin/git add src/modules/campaigns/styles.ts src/modules/campaigns/CriterionCard.tsx src/modules/campaigns/CriterionCard.test.tsx
/opt/homebrew/bin/git commit -m "feat: add clinical criterion cards for the campaign audience builder

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Construtor de público — recorte e cartões

**Files:**
- Create: `src/modules/campaigns/AudienceBuilder.tsx`
- Test: `src/modules/campaigns/AudienceBuilder.test.tsx`

**Interfaces:**
- Consumes:
  - da Task 2: `AudienceDraft`, `CRITERION_KINDS`, `CRITERION_LABEL`, `MAX_CRITERIA`, `MAX_NEIGHBORHOODS`, `newCriterion`, `nextCriterionKey`, `validateGeo`;
  - da Task 4: `CriterionCard`, `toggleValue` e os estilos;
  - de `territory.ts`: `normalizeName`, `sortByName`.
- Produces: `AudienceBuilder(props: AudienceBuilderProps)`, com `AudienceBuilderProps { draft: AudienceDraft; options: CampaignOptions; today: string; onChange(next: AudienceDraft) }`. É controlado: o estado mora em quem o usa.
- Nomes acessíveis que as Tasks 7, 8 e 11 usam:
  - radios "Cidade toda", "Unidade de referência" e "Bairros";
  - select "Unidade";
  - checkboxes com o nome de cada bairro, e "Filtrar bairros";
  - botão "Adicionar critério clínico";
  - texto "e também" entre cartões.

- [ ] **Step 1: Escreva os testes**

```tsx
// src/modules/campaigns/AudienceBuilder.test.tsx
import { afterEach, describe, expect, it } from "vitest";
import { cleanup, fireEvent, render, screen, within } from "@testing-library/react";
import { useState } from "react";
import type { CampaignOptions, Criterion } from "../../lib/api";
import { emptyAudienceDraft, type AudienceDraft } from "../../lib/campaigns";
import { AudienceBuilder } from "./AudienceBuilder";
import { OPTIONS } from "../../test/campaignFixtures";

afterEach(cleanup);
const TODAY = "2026-09-29";
let last: AudienceDraft;

function Harness({ initial, options = OPTIONS }: { initial: AudienceDraft; options?: CampaignOptions }) {
  const [ draft, setDraft ] = useState(initial);
  last = draft;
  return <AudienceBuilder draft={draft} options={options} today={TODAY} onChange={(d) => { last = d; setDraft(d); }} />;
}
const withCriteria = (criteria: Criterion[], geo: AudienceDraft["geo"] = { scope: "city" }): AudienceDraft =>
  ({ geo, criteria: criteria.map((criterion, i) => ({ key: `k${i}`, criterion })) });
const P = { from: "2026-09-01", to: "2026-09-29" };

describe("AudienceBuilder — recorte", () => {
  it("começa na cidade toda, sem critérios", () => {
    render(<Harness initial={emptyAudienceDraft()} />);
    expect((screen.getByRole("radio", { name: "Cidade toda" }) as HTMLInputElement).checked).toBe(true);
    expect(screen.queryByText("e também")).toBeNull();
    expect(screen.queryByRole("alert")).toBeNull();
  });

  it("unidade de referência: pede a unidade e grava o id", () => {
    render(<Harness initial={emptyAudienceDraft()} />);
    fireEvent.click(screen.getByRole("radio", { name: "Unidade de referência" }));
    expect(screen.getByRole("alert").textContent).toBe("escolha a unidade de referência");
    fireEvent.change(screen.getByLabelText("Unidade"), { target: { value: "u1" } });
    expect(last.geo).toEqual({ scope: "unit", health_unit_id: "u1" });
    expect(screen.queryByRole("alert")).toBeNull();
  });

  it("bairros: seleção múltipla, com contagem", () => {
    render(<Harness initial={emptyAudienceDraft()} />);
    fireEvent.click(screen.getByRole("radio", { name: "Bairros" }));
    expect(screen.getByRole("alert").textContent).toBe("escolha ao menos um bairro");
    fireEvent.click(screen.getByRole("checkbox", { name: "Xaxim" }));
    fireEvent.click(screen.getByRole("checkbox", { name: "Boqueirão" }));
    expect(last.geo).toEqual({ scope: "neighborhoods", neighborhood_ids: [ "n2", "n1" ] });
    expect(screen.getByText("2 de no máximo 50 escolhidos")).toBeTruthy();
  });

  it("filtro de bairros ignora acento e maiúscula", () => {
    render(<Harness initial={withCriteria([], { scope: "neighborhoods", neighborhood_ids: [] })} />);
    fireEvent.change(screen.getByLabelText("Filtrar bairros"), { target: { value: "BOQUEIRAO" } });
    expect(screen.getByRole("checkbox", { name: "Boqueirão" })).toBeTruthy();
    expect(screen.queryByRole("checkbox", { name: "Xaxim" })).toBeNull();
  });

  it("rascunho com bairro fora da lista: aparece e sai ao desmarcar", () => {
    render(<Harness initial={withCriteria([], { scope: "neighborhoods", neighborhood_ids: [ "n1", "sumiu" ] })} />);
    const inactive = screen.getByRole("checkbox", { name: "(bairro inativo)" }) as HTMLInputElement;
    expect(inactive.checked).toBe(true);
    fireEvent.click(inactive);
    expect(last.geo).toEqual({ scope: "neighborhoods", neighborhood_ids: [ "n1" ] });
    expect(screen.queryByRole("checkbox", { name: "(bairro inativo)" })).toBeNull();
  });

  it("com 50 bairros escolhidos, os outros travam", () => {
    const many = Array.from({ length: 51 }, (_, i) => ({ id: `b${i}`, name: `Bairro ${String(i).padStart(2, "0")}` }));
    render(<Harness options={{ ...OPTIONS, neighborhoods: many }}
      initial={withCriteria([], { scope: "neighborhoods", neighborhood_ids: many.slice(0, 50).map((n) => n.id) })} />);
    expect((screen.getByRole("checkbox", { name: "Bairro 50" }) as HTMLInputElement).disabled).toBe(true);
    expect((screen.getByRole("checkbox", { name: "Bairro 00" }) as HTMLInputElement).disabled).toBe(false);
  });
});

describe("AudienceBuilder — critérios", () => {
  it("adicionar abre os 7 tipos e cria o cartão com o período padrão", () => {
    render(<Harness initial={emptyAudienceDraft()} />);
    fireEvent.click(screen.getByRole("button", { name: "Adicionar critério clínico" }));
    const menu = within(screen.getByRole("group", { name: "Tipos de critério" }));
    expect(menu.getAllByRole("button").map((b) => b.textContent)).toEqual([
      "Protocolo e período", "Faixa da triagem", "Triagem não concluída", "Desfecho de atendimento",
      "Triado e não atendido", "Falta em agendamento", "Pedido de agendamento aberto", "Fechar lista"
    ]);
    fireEvent.click(menu.getByRole("button", { name: "Falta em agendamento" }));
    const card = within(screen.getByRole("region", { name: "Falta em agendamento" }));
    expect((card.getByLabelText("De") as HTMLInputElement).value).toBe("2026-08-31");
    expect((card.getByLabelText("Até") as HTMLInputElement).value).toBe(TODAY);
    expect(screen.queryByRole("group", { name: "Tipos de critério" })).toBeNull();
  });

  it("'e também' entre os cartões", () => {
    render(<Harness initial={withCriteria([
      { kind: "appointment_no_show", ...P }, { kind: "triage_incomplete", ...P }, { kind: "appointment_request_open" }
    ])} />);
    expect(screen.getAllByText("e também")).toHaveLength(2);
  });

  it("remover tira só aquele cartão", () => {
    render(<Harness initial={withCriteria([
      { kind: "appointment_no_show", ...P }, { kind: "triage_incomplete", ...P }, { kind: "appointment_request_open" }
    ])} />);
    fireEvent.click(screen.getByRole("button", { name: "Remover Triagem não concluída" }));
    expect(last.criteria.map((d) => d.criterion.kind)).toEqual([ "appointment_no_show", "appointment_request_open" ]);
    expect(screen.getAllByText("e também")).toHaveLength(1);
  });

  it("mudar um cartão não mexe no outro", () => {
    render(<Harness initial={withCriteria([ { kind: "appointment_no_show", ...P }, { kind: "triage_incomplete", ...P } ])} />);
    const second = within(screen.getByRole("region", { name: "Triagem não concluída" }));
    fireEvent.change(second.getByLabelText("De"), { target: { value: "2026-09-10" } });
    expect(last.criteria[0].criterion).toEqual({ kind: "appointment_no_show", ...P });
    expect(last.criteria[1].criterion).toEqual({ kind: "triage_incomplete", from: "2026-09-10", to: "2026-09-29" });
  });

  it("no máximo 7", () => {
    render(<Harness initial={withCriteria(Array.from({ length: 7 }, () => ({ kind: "appointment_no_show" as const, ...P })))} />);
    expect((screen.getByRole("button", { name: "Adicionar critério clínico" }) as HTMLButtonElement).disabled).toBe(true);
    expect(screen.getByText("no máximo 7 critérios")).toBeTruthy();
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `cd apps/dashboard/.claude/mod12 && npx vitest run src/modules/campaigns/AudienceBuilder.test.tsx`
Expected: FAIL com "Failed to resolve import ./AudienceBuilder".

- [ ] **Step 3: Implemente**

```tsx
// src/modules/campaigns/AudienceBuilder.tsx
import { Fragment, useState } from "react";
import type { AudienceGeo, CampaignOptions, Criterion, CriterionKind, NamedRef } from "../../lib/api";
import {
  CRITERION_KINDS, CRITERION_LABEL, MAX_CRITERIA, MAX_NEIGHBORHOODS, newCriterion, nextCriterionKey, validateGeo,
  type AudienceDraft
} from "../../lib/campaigns";
import { normalizeName, sortByName } from "../../lib/territory";
import { buttonStyle, disabledButtonStyle, inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { CriterionCard, toggleValue } from "./CriterionCard";
import { alertStyle, checkStyle, columnStyle, fieldsetStyle, labelStyle, legendStyle, noteStyle } from "./styles";

// Construtor A (spec 2026-09-29 §7; D3, D10): 1. recorte geográfico, 2. zero a
// sete critérios clínicos somados por E. Controlado: o rascunho mora no editor.
export interface AudienceBuilderProps {
  draft: AudienceDraft;
  options: CampaignOptions;
  today: string;
  onChange(next: AudienceDraft): void;
}

const SCOPES: { scope: AudienceGeo["scope"]; label: string }[] = [
  { scope: "city", label: "Cidade toda" },
  { scope: "unit", label: "Unidade de referência" },
  { scope: "neighborhoods", label: "Bairros" }
];

function geoFor(scope: AudienceGeo["scope"]): AudienceGeo {
  if (scope === "unit") return { scope, health_unit_id: "" };
  if (scope === "neighborhoods") return { scope, neighborhood_ids: [] };
  return { scope: "city" };
}

export function AudienceBuilder({ draft, options, today, onChange }: AudienceBuilderProps) {
  const [ adding, setAdding ] = useState(false);
  const geo = draft.geo;
  const geoProblem = validateGeo(geo);
  const full = draft.criteria.length >= MAX_CRITERIA;

  function setGeo(next: AudienceGeo) {
    onChange({ ...draft, geo: next });
  }
  function add(kind: CriterionKind) {
    setAdding(false);
    onChange({ ...draft, criteria: [ ...draft.criteria, { key: nextCriterionKey(), criterion: newCriterion(kind, today) } ] });
  }
  function replace(key: string, criterion: Criterion) {
    onChange({ ...draft, criteria: draft.criteria.map((d) => (d.key === key ? { key, criterion } : d)) });
  }
  function remove(key: string) {
    onChange({ ...draft, criteria: draft.criteria.filter((d) => d.key !== key) });
  }

  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 16 }}>
      <fieldset style={{ ...fieldsetStyle, flexDirection: "column", flexWrap: "nowrap" }}>
        <legend style={legendStyle}>1. Recorte geográfico</legend>
        <div role="radiogroup" aria-label="Recorte geográfico" style={{ display: "flex", gap: 16, flexWrap: "wrap" }}>
          {SCOPES.map((s) => (
            <label key={s.scope} style={checkStyle}>
              <input type="radio" name="campaign-geo-scope" checked={geo.scope === s.scope}
                onChange={() => { if (geo.scope !== s.scope) setGeo(geoFor(s.scope)); }} />
              {s.label}
            </label>
          ))}
        </div>
        {geo.scope === "unit" && (
          <label style={{ ...labelStyle, maxWidth: 360 }}>
            Unidade
            <select value={geo.health_unit_id} onChange={(e) => setGeo({ scope: "unit", health_unit_id: e.target.value })} style={inputStyle}>
              <option value="">escolha…</option>
              {sortByName(options.units).map((u) => <option key={u.id} value={u.id}>{u.name}</option>)}
            </select>
          </label>
        )}
        {geo.scope === "neighborhoods" && (
          <NeighborhoodChecklist selected={geo.neighborhood_ids} neighborhoods={options.neighborhoods}
            onChange={(ids) => setGeo({ scope: "neighborhoods", neighborhood_ids: ids })} />
        )}
        {geoProblem && <p role="alert" style={alertStyle}>{geoProblem}</p>}
      </fieldset>

      <section aria-label="2. Critérios clínicos" style={columnStyle}>
        <strong style={{ fontSize: 13 }}>2. Critérios clínicos</strong>
        <p style={noteStyle}>opcional — quem entra precisa atender a todos os critérios ao mesmo tempo</p>
        {draft.criteria.map((d, i) => (
          <Fragment key={d.key}>
            {i > 0 && <p className="mono" style={{ margin: 0, fontSize: 11, color: "var(--ink3)", textAlign: "center" }}>e também</p>}
            <CriterionCard criterion={d.criterion} options={options} today={today}
              onChange={(c) => replace(d.key, c)} onRemove={() => remove(d.key)} />
          </Fragment>
        ))}
        {adding ? (
          <div role="group" aria-label="Tipos de critério" style={{ display: "flex", gap: 6, flexWrap: "wrap" }}>
            {CRITERION_KINDS.map((kind) => (
              <button key={kind} type="button" style={secondaryButtonStyle} onClick={() => add(kind)}>{CRITERION_LABEL[kind]}</button>
            ))}
            <button type="button" style={secondaryButtonStyle} onClick={() => setAdding(false)}>Fechar lista</button>
          </div>
        ) : (
          <div>
            <button type="button" disabled={full} style={full ? disabledButtonStyle : buttonStyle} onClick={() => setAdding(true)}>
              Adicionar critério clínico
            </button>
          </div>
        )}
        {full && <p style={noteStyle}>no máximo {MAX_CRITERIA} critérios</p>}
      </section>
    </div>
  );
}

// Bairro do rascunho que não está mais em `options` (desativado depois) fica
// VISÍVEL e desmarcável: sumir da tela e continuar no público seria contar
// gente que a pessoa não vê.
function NeighborhoodChecklist({ selected, neighborhoods, onChange }: {
  selected: string[]; neighborhoods: NamedRef[]; onChange(ids: string[]): void;
}) {
  const [ filter, setFilter ] = useState("");
  const known = new Set(neighborhoods.map((n) => n.id));
  const unknown = selected.filter((id) => !known.has(id));
  const key = normalizeName(filter);
  const visible = sortByName(neighborhoods).filter((n) => !key || normalizeName(n.name).includes(key));
  const full = selected.length >= MAX_NEIGHBORHOODS;

  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 8 }}>
      <label style={{ ...labelStyle, maxWidth: 320 }}>
        Filtrar bairros
        <input value={filter} onChange={(e) => setFilter(e.target.value)} style={inputStyle} />
      </label>
      <p className="mono" style={noteStyle}>{selected.length} de no máximo {MAX_NEIGHBORHOODS} escolhidos</p>
      {unknown.length > 0 && (
        <div role="group" aria-label="Bairros fora da lista" style={{ display: "flex", flexDirection: "column", gap: 4 }}>
          <p style={alertStyle}>bairros deste rascunho que não estão mais ativos — desmarque para tirar do público</p>
          {unknown.map((id) => (
            <label key={id} style={checkStyle}>
              <input type="checkbox" checked onChange={() => onChange(selected.filter((x) => x !== id))} />
              (bairro inativo)
            </label>
          ))}
        </div>
      )}
      <div role="group" aria-label="Lista de bairros"
        style={{ display: "grid", gridTemplateColumns: "repeat(auto-fill, minmax(180px, 1fr))", gap: 6 }}>
        {visible.map((n) => {
          const checked = selected.includes(n.id);
          return (
            <label key={n.id} style={checkStyle}>
              <input type="checkbox" checked={checked} disabled={!checked && full}
                onChange={() => onChange(toggleValue(selected, n.id))} />
              {n.name}
            </label>
          );
        })}
        {visible.length === 0 && (
          <p style={noteStyle}>
            {neighborhoods.length === 0 ? "nenhum bairro ativo — cadastre em Cidade → Território" : "nenhum bairro com esse nome"}
          </p>
        )}
      </div>
    </div>
  );
}
```

- [ ] **Step 4: Rode, cheque tipos e faça commit**

Run: `cd apps/dashboard/.claude/mod12 && npx vitest run src/modules/campaigns/AudienceBuilder.test.tsx && npx tsc --noEmit`
Expected: PASS; tsc sem erros.

```bash
cd apps/dashboard/.claude/mod12
/opt/homebrew/bin/git add src/modules/campaigns/AudienceBuilder.tsx src/modules/campaigns/AudienceBuilder.test.tsx
/opt/homebrew/bin/git commit -m "feat: add the guided campaign audience builder

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Contagem ao vivo com debounce

**Files:**
- Create: `src/lib/useDebouncedValue.ts`, `src/modules/campaigns/useAudiencePreview.ts`, `src/modules/campaigns/AudienceCounter.tsx`
- Test: `src/lib/useDebouncedValue.test.ts`, `src/modules/campaigns/AudienceCounter.test.tsx`

**Interfaces:**
- Consumes:
  - da Task 1: `previewAudience`, `Audience`;
  - da Task 2: `PREVIEW_DEBOUNCE_MS`, `PREVIEW_KEY`, `campaignError`;
  - `fmtNumber` de `format.ts`.
- Produces:
  - `useDebouncedValue<T>(value, delayMs): T`;
  - `PreviewState = { kind: "incomplete" } | { kind: "loading" } | { kind: "error"; message } | { kind: "below_minimum" } | { kind: "ok"; citizens; phones }`;
  - `useAudiencePreview(audience: Audience | null, delayMs = PREVIEW_DEBOUNCE_MS): PreviewState`;
  - `previewText(state, problem) → string`;
  - `previewAllowsSend(state) → state is { kind: "ok" … }`;
  - `AudienceCounter({ state, problem })`, um `role="status"` com nome "Contagem do público".

- [ ] **Step 1: Escreva os testes**

```ts
// src/lib/useDebouncedValue.test.ts
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { act, renderHook } from "@testing-library/react";
import { useDebouncedValue } from "./useDebouncedValue";

describe("useDebouncedValue", () => {
  beforeEach(() => vi.useFakeTimers({ toFake: [ "setTimeout", "clearTimeout" ] }));
  afterEach(() => vi.useRealTimers());

  it("só muda depois de 500 ms parado, e cada mudança reinicia a espera", () => {
    const { result, rerender } = renderHook(({ v }) => useDebouncedValue(v, 500), { initialProps: { v: "a" } });
    expect(result.current).toBe("a");
    rerender({ v: "b" });
    act(() => { vi.advanceTimersByTime(499); });
    expect(result.current).toBe("a");
    rerender({ v: "c" });
    act(() => { vi.advanceTimersByTime(499); });
    expect(result.current).toBe("a");
    act(() => { vi.advanceTimersByTime(1); });
    expect(result.current).toBe("c");
  });
});
```

```tsx
// src/modules/campaigns/AudienceCounter.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, render, renderHook, screen, waitFor } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, previewAudience: vi.fn() };
});

import * as api from "../../lib/api";
import { ApiError, type Audience } from "../../lib/api";
import { AudienceCounter, previewAllowsSend, previewText } from "./AudienceCounter";
import { useAudiencePreview } from "./useAudiencePreview";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const byNeighborhoods = (...ids: string[]): Audience =>
  ({ version: 1, geo: { scope: "neighborhoods", neighborhood_ids: ids }, clinical: { all: [] } });

function wrapper({ children }: { children: ReactNode }) {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  return <QueryClientProvider client={client}>{children}</QueryClientProvider>;
}

describe("useAudiencePreview", () => {
  beforeEach(() => mocked(api.previewAudience).mockReset());

  it("público incompleto nunca chama a API", () => {
    const { result } = renderHook(() => useAudiencePreview(null, 30), { wrapper });
    expect(result.current).toEqual({ kind: "incomplete" });
    expect(api.previewAudience).not.toHaveBeenCalled();
  });

  it("só a última versão do público vai à API", async () => {
    mocked(api.previewAudience).mockResolvedValue({ citizens: 12, phones: 9 });
    const { result, rerender } = renderHook(({ a }) => useAudiencePreview(a, 30),
      { wrapper, initialProps: { a: byNeighborhoods("n1") } });
    await waitFor(() => expect(result.current).toEqual({ kind: "ok", citizens: 12, phones: 9 }));
    mocked(api.previewAudience).mockClear();

    rerender({ a: byNeighborhoods("n1", "n2") });
    rerender({ a: byNeighborhoods("n1", "n2", "n3") });
    await waitFor(() => expect(result.current.kind).toBe("ok"));
    expect(api.previewAudience).toHaveBeenCalledTimes(1);
    expect(api.previewAudience).toHaveBeenCalledWith(byNeighborhoods("n1", "n2", "n3"));
  });

  it("público mudou depois da contagem: volta a calculando até a nova chave assentar", async () => {
    mocked(api.previewAudience).mockResolvedValue({ citizens: 12, phones: 9 });
    const { result, rerender } = renderHook(({ a }) => useAudiencePreview(a, 30),
      { wrapper, initialProps: { a: byNeighborhoods("n1") } });
    await waitFor(() => expect(result.current.kind).toBe("ok"));

    rerender({ a: byNeighborhoods("n1", "n2") });
    expect(result.current).toEqual({ kind: "loading" });
    expect(previewAllowsSend(result.current)).toBe(false);
  });

  it("below_minimum e erro da API", async () => {
    mocked(api.previewAudience).mockResolvedValueOnce({ below_minimum: true });
    const first = renderHook(() => useAudiencePreview(byNeighborhoods("n1"), 30), { wrapper });
    await waitFor(() => expect(first.result.current).toEqual({ kind: "below_minimum" }));

    mocked(api.previewAudience).mockRejectedValueOnce(
      new ApiError(422, { error: "invalid_audience", details: [ { path: "geo.neighborhood_ids", message: "x" } ] }, "422"));
    const second = renderHook(() => useAudiencePreview(byNeighborhoods("n9"), 30), { wrapper });
    await waitFor(() => expect(second.result.current)
      .toEqual({ kind: "error", message: "público inválido — confira geo.neighborhood_ids" }));
  });
});

describe("AudienceCounter", () => {
  it("frases de cada estado", () => {
    expect(previewText({ kind: "ok", citizens: 1234, phones: 987 }, null)).toBe("≈ 1.234 pessoas (987 telefones)");
    expect(previewText({ kind: "below_minimum" }, null)).toBe("menos de 5 — ajuste o público");
    expect(previewText({ kind: "loading" }, null)).toBe("calculando…");
    expect(previewText({ kind: "incomplete" }, "escolha ao menos um bairro")).toBe("complete o público: escolha ao menos um bairro");
    expect(previewText({ kind: "error", message: "falhou" }, null)).toBe("falhou");
  });

  it("só 'ok' libera o envio", () => {
    expect(previewAllowsSend({ kind: "ok", citizens: 5, phones: 5 })).toBe(true);
    for (const s of [ { kind: "loading" }, { kind: "below_minimum" }, { kind: "incomplete" } ] as const) {
      expect(previewAllowsSend(s)).toBe(false);
    }
  });

  it("renderiza como status nomeado", () => {
    render(<AudienceCounter state={{ kind: "below_minimum" }} problem={null} />);
    expect(screen.getByRole("status", { name: "Contagem do público" }).textContent).toBe("menos de 5 — ajuste o público");
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `cd apps/dashboard/.claude/mod12 && npx vitest run src/lib/useDebouncedValue.test.ts src/modules/campaigns/AudienceCounter.test.tsx`
Expected: FAIL com "Failed to resolve import ./useDebouncedValue" e "./AudienceCounter".

- [ ] **Step 3: Implemente**

```ts
// src/lib/useDebouncedValue.ts
import { useEffect, useState } from "react";

// O valor só muda depois de `delayMs` sem mudança. Passe um valor primitivo
// (ex.: JSON.stringify) — objeto novo a cada render reiniciaria a espera.
export function useDebouncedValue<T>(value: T, delayMs: number): T {
  const [ debounced, setDebounced ] = useState(value);
  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delayMs);
    return () => clearTimeout(id);
  }, [ value, delayMs ]);
  return debounced;
}
```

```ts
// src/modules/campaigns/useAudiencePreview.ts
import { useQuery } from "@tanstack/react-query";
import { previewAudience, type Audience } from "../../lib/api";
import { PREVIEW_DEBOUNCE_MS, PREVIEW_KEY, campaignError } from "../../lib/campaigns";
import { useDebouncedValue } from "../../lib/useDebouncedValue";

export type PreviewState =
  | { kind: "incomplete" }
  | { kind: "loading" }
  | { kind: "error"; message: string }
  | { kind: "below_minimum" }
  | { kind: "ok"; citizens: number; phones: number };

// Contagem ao vivo (spec §7). O público vira uma chave de texto, e só a chave
// que ficou parada por `delayMs` chega à API. Enquanto a chave digitada não é
// a consultada, o estado é "loading" — nunca a contagem do público anterior,
// que liberaria o envio de um público que ninguém contou.
export function useAudiencePreview(audience: Audience | null, delayMs: number = PREVIEW_DEBOUNCE_MS): PreviewState {
  const key = audience ? JSON.stringify(audience) : null;
  const settledKey = useDebouncedValue(key, delayMs);
  const query = useQuery({
    queryKey: [ ...PREVIEW_KEY, settledKey ],
    queryFn: () => previewAudience(JSON.parse(settledKey as string) as Audience),
    enabled: settledKey !== null,
    retry: false
  });

  if (key === null) return { kind: "incomplete" };
  if (settledKey !== key) return { kind: "loading" };
  if (query.isPending) return { kind: "loading" };
  if (query.isError) return { kind: "error", message: campaignError(query.error) };
  if ("below_minimum" in query.data) return { kind: "below_minimum" };
  return { kind: "ok", citizens: query.data.citizens, phones: query.data.phones };
}
```

```tsx
// src/modules/campaigns/AudienceCounter.tsx
import { fmtNumber } from "../../lib/format";
import type { PreviewState } from "./useAudiencePreview";

export function previewText(state: PreviewState, problem: string | null): string {
  if (problem) return `complete o público: ${problem}`;
  switch (state.kind) {
    case "incomplete": return "complete o público para ver a contagem";
    case "loading": return "calculando…";
    case "error": return state.message;
    case "below_minimum": return "menos de 5 — ajuste o público";
    case "ok": return `≈ ${fmtNumber(state.citizens)} pessoas (${fmtNumber(state.phones)} telefones)`;
  }
}

// Mínimo de 5 telefones (D8): só uma contagem do público ATUAL libera.
export function previewAllowsSend(state: PreviewState): state is Extract<PreviewState, { kind: "ok" }> {
  return state.kind === "ok";
}

export function AudienceCounter({ state, problem }: { state: PreviewState; problem: string | null }) {
  const blocked = problem !== null || state.kind === "below_minimum" || state.kind === "error";
  return (
    <p role="status" aria-live="polite" aria-label="Contagem do público"
      style={{ margin: 0, fontSize: 14, fontWeight: 600, color: blocked ? "var(--down)" : "var(--ink)" }}>
      {previewText(state, problem)}
    </p>
  );
}
```

- [ ] **Step 4: Rode, cheque tipos e faça commit**

Run: `cd apps/dashboard/.claude/mod12 && npx vitest run src/lib/useDebouncedValue.test.ts src/modules/campaigns/AudienceCounter.test.tsx && npx tsc --noEmit`
Expected: PASS; tsc sem erros.

```bash
cd apps/dashboard/.claude/mod12
/opt/homebrew/bin/git add src/lib/useDebouncedValue.ts src/lib/useDebouncedValue.test.ts src/modules/campaigns/useAudiencePreview.ts src/modules/campaigns/AudienceCounter.tsx src/modules/campaigns/AudienceCounter.test.tsx
/opt/homebrew/bin/git commit -m "feat: add live debounced audience count for campaigns

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: Editor — título, texto, pré-visualização, público e rascunho

**Files:**
- Create: `src/modules/campaigns/CampaignEditor.tsx`
- Test: `src/modules/campaigns/CampaignEditor.test.tsx`

**Interfaces:**
- Consumes:
  - da Task 1: `createCampaign`, `getCampaign`, `getCampaignOptions`, `getSmsSetting`, `updateCampaign`, `Campaign`, `CampaignOptions`;
  - da Task 2: `BODY_MAX`, `CAMPAIGNS_KEY`, `CAMPAIGN_OPTIONS_KEY`, `PREVIEW_DEBOUNCE_MS`, `SMS_SETTING_KEY`, `TITLE_MAX`, `audienceProblem`, `buildAudience`, `campaignError`, `campaignKey`, `draftFromAudience`, `emptyAudienceDraft`, `isEditable`, `todayInCity`, `validateCampaignFields`, `AudienceDraft`;
  - da Task 5: `AudienceBuilder`;
  - da Task 6: `AudienceCounter`, `useAudiencePreview`.
- Produces: `CampaignEditor(props: CampaignEditorProps)`, com `CampaignEditorProps { campaignId: string | null; onBack(): void; previewDelayMs?: number }`. A Task 8 acrescenta `onLeftDraft` e `onGoToSecurity`.
- Nomes acessíveis:
  - campos "Título" e "Texto do aviso";
  - `article` "Pré-visualização no wpda";
  - botões "Salvar rascunho" e "Voltar à lista";
  - `status` "rascunho salvo".

- [ ] **Step 1: Escreva os testes**

```tsx
// src/modules/campaigns/CampaignEditor.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return {
    ...real, fetchCurrentSession: vi.fn(), stepUpMfa: vi.fn(), getCampaignOptions: vi.fn(), getCampaign: vi.fn(),
    getSmsSetting: vi.fn(), previewAudience: vi.fn(), createCampaign: vi.fn(), updateCampaign: vi.fn(),
    sendCampaign: vi.fn(), scheduleCampaign: vi.fn(), cancelCampaign: vi.fn()
  };
});

import * as api from "../../lib/api";
import { ApiError, type Audience } from "../../lib/api";
import { CampaignEditor } from "./CampaignEditor";
import { OPTIONS, campaign, renderWithProviders, sessionWith } from "../../test/campaignFixtures";

afterEach(() => { cleanup(); vi.useRealTimers(); });
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const BOQUEIRAO: Audience = { version: 1, geo: { scope: "neighborhoods", neighborhood_ids: [ "n1" ] }, clinical: { all: [] } };

function renderEditor(campaignId: string | null = null) {
  const onBack = vi.fn();
  renderWithProviders(<CampaignEditor campaignId={campaignId} onBack={onBack} previewDelayMs={0} />);
  return { onBack };
}

async function fillContent(title = " Vacina da gripe ", body = "Vacinação no sábado, das 8h às 17h.") {
  fireEvent.change(await screen.findByLabelText("Título"), { target: { value: title } });
  fireEvent.change(screen.getByLabelText("Texto do aviso"), { target: { value: body } });
}

describe("CampaignEditor", () => {
  beforeEach(() => {
    vi.resetAllMocks();
    vi.useFakeTimers({ toFake: [ "Date" ] });
    vi.setSystemTime(new Date("2026-09-29T10:00:00-03:00"));
    mocked(api.fetchCurrentSession).mockResolvedValue(sessionWith([ "campaign_manager" ]));
    mocked(api.getCampaignOptions).mockResolvedValue(OPTIONS);
    mocked(api.getSmsSetting).mockResolvedValue({ enabled: false, gateway_configured: false });
    mocked(api.previewAudience).mockResolvedValue({ citizens: 12, phones: 9 });
  });

  it("nova campanha: salva com texto aparado e o público montado; o segundo salvar edita", async () => {
    mocked(api.createCampaign).mockResolvedValue(campaign({ id: "c9", audience: BOQUEIRAO }));
    mocked(api.updateCampaign).mockResolvedValue(campaign({ id: "c9", audience: BOQUEIRAO }));
    renderEditor();
    await fillContent();
    fireEvent.click(screen.getByRole("radio", { name: "Bairros" }));
    fireEvent.click(screen.getByRole("checkbox", { name: "Boqueirão" }));

    fireEvent.click(screen.getByRole("button", { name: "Salvar rascunho" }));
    await waitFor(() => expect(api.createCampaign).toHaveBeenCalledWith({
      title: "Vacina da gripe", body: "Vacinação no sábado, das 8h às 17h.", audience: BOQUEIRAO
    }));
    expect((await screen.findByText("rascunho salvo")).getAttribute("role")).toBe("status");

    fireEvent.click(screen.getByRole("button", { name: "Salvar rascunho" }));
    await waitFor(() => expect(api.updateCampaign).toHaveBeenCalledWith("c9", expect.objectContaining({ title: "Vacina da gripe" })));
    expect(api.createCampaign).toHaveBeenCalledTimes(1);
  });

  it("título ou texto curto: não chama a API", async () => {
    renderEditor();
    await fillContent("ab");
    fireEvent.click(screen.getByRole("button", { name: "Salvar rascunho" }));
    expect((await screen.findByText("o título precisa ter de 3 a 120 caracteres")).getAttribute("role")).toBe("alert");
    expect(api.createCampaign).not.toHaveBeenCalled();
  });

  it("público incompleto: não salva e diz o que falta", async () => {
    renderEditor();
    await fillContent();
    fireEvent.click(screen.getByRole("radio", { name: "Bairros" }));
    fireEvent.click(screen.getByRole("button", { name: "Salvar rascunho" }));
    expect(await screen.findByText("complete o público antes de salvar: escolha ao menos um bairro")).toBeTruthy();
    expect(api.createCampaign).not.toHaveBeenCalled();
  });

  it("rascunho existente: carrega título, texto e cartões", async () => {
    mocked(api.getCampaign).mockResolvedValue(campaign({
      audience: { ...BOQUEIRAO, clinical: { all: [ { kind: "appointment_no_show", from: "2026-07-01", to: "2026-09-29" } ] } }
    }));
    renderEditor("c1");
    expect(((await screen.findByLabelText("Título")) as HTMLInputElement).value).toBe("Vacinação contra a gripe");
    const card = within(screen.getByRole("region", { name: "Falta em agendamento" }));
    expect((card.getByLabelText("De") as HTMLInputElement).value).toBe("2026-07-01");
    expect((screen.getByRole("checkbox", { name: "Boqueirão" }) as HTMLInputElement).checked).toBe(true);
  });

  it("pré-visualização mostra o texto como texto, com as quebras de linha", async () => {
    renderEditor();
    await fillContent("Aviso", "<b>Atenção</b>\nsegunda linha");
    const preview = screen.getByRole("article", { name: "Pré-visualização no wpda" });
    expect(preview.querySelector("b")).toBeNull();
    expect(preview.textContent).toContain("<b>Atenção</b>\nsegunda linha");
    expect((preview.querySelector("p") as HTMLElement).style.whiteSpace).toBe("pre-wrap");
  });

  it("contador: contagem da API e 'menos de 5'", async () => {
    renderEditor();
    expect(await screen.findByText("≈ 12 pessoas (9 telefones)")).toBeTruthy();
    cleanup();

    mocked(api.previewAudience).mockResolvedValue({ below_minimum: true });
    renderEditor();
    expect(await screen.findByText("menos de 5 — ajuste o público")).toBeTruthy();
  });

  it("linha informativa do SMS da cidade", async () => {
    renderEditor();
    expect(await screen.findByText("SMS nesta cidade: desligado")).toBeTruthy();
  });

  it("recusa not_editable aparece traduzida", async () => {
    mocked(api.getCampaign).mockResolvedValue(campaign({ audience: BOQUEIRAO }));
    mocked(api.updateCampaign).mockRejectedValue(new ApiError(422, { error: "not_editable" }, "422"));
    renderEditor("c1");
    await screen.findByLabelText("Título");
    fireEvent.click(screen.getByRole("button", { name: "Salvar rascunho" }));
    expect(await screen.findByText("esta campanha não é mais rascunho — volte à lista e abra de novo")).toBeTruthy();
  });

  it("campanha que não é mais rascunho não abre no editor", async () => {
    mocked(api.getCampaign).mockResolvedValue(campaign({ status: "scheduled" }));
    renderEditor("c1");
    expect(await screen.findByText("esta campanha não é mais rascunho — volte à lista")).toBeTruthy();
    expect(screen.queryByLabelText("Título")).toBeNull();
  });

  it("voltar à lista", async () => {
    const { onBack } = renderEditor();
    fireEvent.click(await screen.findByRole("button", { name: "Voltar à lista" }));
    expect(onBack).toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `cd apps/dashboard/.claude/mod12 && npx vitest run src/modules/campaigns/CampaignEditor.test.tsx`
Expected: FAIL com "Failed to resolve import ./CampaignEditor".

- [ ] **Step 3: Implemente**

```tsx
// src/modules/campaigns/CampaignEditor.tsx
import { useState, type CSSProperties } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import {
  createCampaign, getCampaign, getCampaignOptions, getSmsSetting, updateCampaign, type Campaign, type CampaignOptions
} from "../../lib/api";
import {
  BODY_MAX, CAMPAIGNS_KEY, CAMPAIGN_OPTIONS_KEY, PREVIEW_DEBOUNCE_MS, SMS_SETTING_KEY, TITLE_MAX, audienceProblem,
  buildAudience, campaignError, campaignKey, draftFromAudience, emptyAudienceDraft, isEditable, todayInCity,
  validateCampaignFields, type AudienceDraft
} from "../../lib/campaigns";
import { Panel } from "../../components/Panel";
import { buttonStyle, disabledButtonStyle, inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { AudienceBuilder } from "./AudienceBuilder";
import { AudienceCounter } from "./AudienceCounter";
import { useAudiencePreview } from "./useAudiencePreview";
import { alertStyle, columnStyle, labelStyle, noteStyle, rowStyle } from "./styles";

// Editor de rascunho (spec 2026-09-29 §7). Só `draft` é editável: outro
// status mostra a recusa e manda de volta à lista. O público inválido não
// salva — o schema do api recusaria (422 invalid_audience).
export interface CampaignEditorProps {
  campaignId: string | null;
  onBack(): void;
  previewDelayMs?: number;
}

export function CampaignEditor(props: CampaignEditorProps) {
  const { campaignId, onBack } = props;
  const options = useQuery({ queryKey: CAMPAIGN_OPTIONS_KEY, queryFn: getCampaignOptions });
  const existing = useQuery({
    queryKey: campaignKey(campaignId ?? "new"),
    queryFn: () => getCampaign(campaignId as string),
    enabled: campaignId !== null
  });
  const back = <button type="button" style={secondaryButtonStyle} onClick={onBack}>Voltar à lista</button>;

  if (options.isError) return <Panel title="Campanha" right={back}><p role="alert" style={alertStyle}>{campaignError(options.error)}</p></Panel>;
  if (existing.isError) return <Panel title="Campanha" right={back}><p role="alert" style={alertStyle}>{campaignError(existing.error)}</p></Panel>;
  if (options.isPending || (campaignId !== null && existing.isPending)) {
    return <Panel title="Campanha" right={back}><p className="mono" style={noteStyle}>carregando…</p></Panel>;
  }
  const initial = campaignId === null ? null : existing.data ?? null;
  if (initial && !isEditable(initial.status)) {
    return <Panel title="Campanha" right={back}><p role="alert" style={alertStyle}>esta campanha não é mais rascunho — volte à lista</p></Panel>;
  }
  return <EditorForm {...props} initial={initial} options={options.data} />;
}

function EditorForm({ initial, options, onBack, previewDelayMs = PREVIEW_DEBOUNCE_MS }: CampaignEditorProps & {
  initial: Campaign | null; options: CampaignOptions;
}) {
  const queryClient = useQueryClient();
  const sms = useQuery({ queryKey: SMS_SETTING_KEY, queryFn: getSmsSetting });
  const [ id, setId ] = useState<string | null>(initial?.id ?? null);
  const [ title, setTitle ] = useState(initial?.title ?? "");
  const [ body, setBody ] = useState(initial?.body ?? "");
  const [ draft, setDraft ] = useState<AudienceDraft>(() => (initial ? draftFromAudience(initial.audience) : emptyAudienceDraft()));
  const [ busy, setBusy ] = useState(false);
  const [ error, setError ] = useState<string | null>(null);
  const [ notice, setNotice ] = useState<string | null>(null);

  const today = todayInCity();
  const problem = audienceProblem(draft, today);
  const audience = problem ? null : buildAudience(draft);
  const preview = useAudiencePreview(audience, previewDelayMs);

  function fail(message: string) {
    setError(message);
    setNotice(null);
  }

  async function save(): Promise<Campaign | null> {
    if (busy) return null;
    const invalid = validateCampaignFields(title, body);
    if (invalid) { fail(invalid); return null; }
    if (!audience) { fail(`complete o público antes de salvar: ${problem}`); return null; }
    const fields = { title: title.trim(), body: body.trim(), audience };
    setBusy(true); setError(null); setNotice(null);
    try {
      const saved = id ? await updateCampaign(id, fields) : await createCampaign(fields);
      setId(saved.id);
      queryClient.setQueryData(campaignKey(saved.id), saved);
      void queryClient.invalidateQueries({ queryKey: CAMPAIGNS_KEY });
      setNotice("rascunho salvo");
      return saved;
    } catch (err) {
      fail(campaignError(err));
      return null;
    } finally {
      setBusy(false);
    }
  }

  const smsLine = sms.data
    ? `SMS nesta cidade: ${sms.data.enabled ? "ligado" : "desligado"}`
    : sms.isError ? "SMS nesta cidade: não foi possível consultar" : "SMS nesta cidade: consultando…";

  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 16 }}>
      <Panel title={id ? "Editar campanha" : "Nova campanha"} sub="rascunho"
        right={<button type="button" style={secondaryButtonStyle} onClick={onBack}>Voltar à lista</button>}>
        <div style={columnStyle}>
          {error && <p role="alert" style={alertStyle}>{error}</p>}
          {notice && <p role="status" style={{ margin: 0, fontSize: 12.5 }}>{notice}</p>}
          <label style={labelStyle}>
            Título
            <input value={title} maxLength={TITLE_MAX} onChange={(e) => setTitle(e.target.value)} style={inputStyle} />
          </label>
          <span className="mono" style={noteStyle}>{title.trim().length}/{TITLE_MAX}</span>
          <label style={labelStyle}>
            Texto do aviso
            <textarea value={body} rows={8} maxLength={BODY_MAX} onChange={(e) => setBody(e.target.value)}
              style={{ ...inputStyle, resize: "vertical", fontFamily: "inherit" }} />
          </label>
          <span className="mono" style={noteStyle}>{body.trim().length}/{BODY_MAX} · texto simples; as quebras de linha ficam</span>
        </div>
      </Panel>

      <Panel title="Como aparece no wpda" sub="pré-visualização">
        <article aria-label="Pré-visualização no wpda" style={wpdaPreviewStyle}>
          <h3 style={{ margin: 0, fontSize: 20 }}>{title.trim() || "Título do aviso"}</h3>
          <p style={{ margin: "8px 0 0", whiteSpace: "pre-wrap" }}>{body || "O texto do aviso aparece aqui."}</p>
        </article>
      </Panel>

      <Panel title="Público" sub="recorte · critérios clínicos">
        <div style={columnStyle}>
          <AudienceBuilder draft={draft} options={options} today={today} onChange={setDraft} />
          <AudienceCounter state={preview} problem={problem} />
          <p style={noteStyle}>{smsLine}</p>
        </div>
      </Panel>

      <div style={rowStyle}>
        <button type="button" disabled={busy} onClick={() => void save()} style={busy ? disabledButtonStyle : buttonStyle}>
          Salvar rascunho
        </button>
      </div>
    </div>
  );
}

// Espelha as regras da casa do wpda: texto ≥ 18px.
const wpdaPreviewStyle: CSSProperties = {
  maxWidth: 420, padding: 16, fontSize: 18, lineHeight: 1.5,
  border: "1px solid var(--rule)", borderRadius: 12, background: "var(--sunken)"
};
```

- [ ] **Step 4: Rode, cheque tipos e faça commit**

Run: `cd apps/dashboard/.claude/mod12 && npx vitest run src/modules/campaigns/CampaignEditor.test.tsx && npx tsc --noEmit`
Expected: PASS; tsc sem erros.

```bash
cd apps/dashboard/.claude/mod12
/opt/homebrew/bin/git add src/modules/campaigns/CampaignEditor.tsx src/modules/campaigns/CampaignEditor.test.tsx
/opt/homebrew/bin/git commit -m "feat: add campaign draft editor with wpda preview and live count

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 8: Enviar ou agendar com step-up

**Files:**
- Create: `src/modules/campaigns/SendDialog.tsx`
- Modify: `src/modules/campaigns/CampaignEditor.tsx`
- Test: `src/modules/campaigns/SendDialog.test.tsx`, `src/modules/campaigns/CampaignEditor.test.tsx`

**Interfaces:**
- Consumes:
  - da Task 1: `scheduleCampaign`, `sendCampaign`, `Campaign`;
  - da Task 2: `CAMPAIGNS_KEY`, `PREVIEW_KEY`, `campaignErrorCode`, `campaignErrorOrNull`, `campaignKey`, `validateSendAt`;
  - da Task 3: `describeAudience`;
  - da Task 6: `previewAllowsSend`;
  - `SensitiveAction`, `fmtDateTime` e `fmtNumber`, que já existem.
- Produces:
  - `SendDialog(props: SendDialogProps)`, com `SendDialogProps { campaign; phrase; counts: { citizens; phones }; smsEnabled: boolean | null; onDone(c: Campaign); onBelowMinimum(); onCancel(); onGoToSecurity() }`;
  - `channelsText(smsEnabled) → string`;
  - `CampaignEditorProps` ganha `onLeftDraft(campaign: Campaign): void` e `onGoToSecurity(): void`.
- Nomes acessíveis:
  - diálogo "Como enviar", com radios "Enviar agora" e "Agendar para", campo "Data e hora (horário da cidade)" e botão "Continuar";
  - depois, o `SensitiveAction` "Enviar campanha" (confirma com "Enviar agora") ou "Agendar campanha" (confirma com "Agendar");
  - no editor, o botão "Revisar e enviar…".

- [ ] **Step 1: Testes do diálogo**

```tsx
// src/modules/campaigns/SendDialog.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), stepUpMfa: vi.fn(), sendCampaign: vi.fn(), scheduleCampaign: vi.fn() };
});

import * as api from "../../lib/api";
import { ApiError } from "../../lib/api";
import { SendDialog, channelsText } from "./SendDialog";
import { campaign, renderWithProviders, sessionWith } from "../../test/campaignFixtures";

afterEach(() => { cleanup(); vi.useRealTimers(); });
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const PHRASE = "moradores de Boqueirão que faltaram a um agendamento entre 01/07/2026 e 29/09/2026";

function renderDialog(smsEnabled: boolean | null = true) {
  const handlers = { onDone: vi.fn(), onBelowMinimum: vi.fn(), onCancel: vi.fn(), onGoToSecurity: vi.fn() };
  renderWithProviders(<SendDialog campaign={campaign()} phrase={PHRASE} counts={{ citizens: 12, phones: 9 }}
    smsEnabled={smsEnabled} {...handlers} />);
  return handlers;
}

function chooseSchedule(value: string) {
  fireEvent.click(screen.getByRole("radio", { name: "Agendar para" }));
  fireEvent.change(screen.getByLabelText("Data e hora (horário da cidade)"), { target: { value } });
  fireEvent.click(screen.getByRole("button", { name: "Continuar" }));
}

describe("SendDialog", () => {
  beforeEach(() => {
    vi.resetAllMocks();
    vi.useFakeTimers({ toFake: [ "Date" ] });
    vi.setSystemTime(new Date("2026-09-29T10:00:00-03:00"));
    mocked(api.fetchCurrentSession).mockResolvedValue(sessionWith([ "campaign_manager" ]));
  });

  it("agora: mostra o público em frase, a contagem e os canais; confirma e devolve a campanha", async () => {
    const sent = campaign({ status: "sending" });
    mocked(api.sendCampaign).mockResolvedValue(sent);
    const { onDone } = renderDialog();
    fireEvent.click(screen.getByRole("button", { name: "Continuar" }));

    // O botão só aparece depois que a sessão carregou; antes disso a mesma
    // região mostra "exige um autenticador cadastrado".
    const confirmButton = await screen.findByRole("button", { name: "Enviar agora" });
    const confirm = screen.getByRole("region", { name: "Enviar campanha" });
    expect(confirm.textContent).toContain(PHRASE);
    expect(confirm.textContent).toContain("≈ 12 pessoas (9 telefones)");
    expect(confirm.textContent).toContain("aviso no wpda + SMS para quem aceitou receber (o SMS só sai entre 8h e 20h)");
    fireEvent.click(confirmButton);

    await waitFor(() => expect(api.sendCampaign).toHaveBeenCalledWith("c1"));
    expect(onDone).toHaveBeenCalledWith(sent);
  });

  it("janela de verificação fechada: pede o código antes de enviar", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(sessionWith([ "campaign_manager" ], { mfa_verified_at: null }));
    mocked(api.stepUpMfa).mockResolvedValue(undefined);
    mocked(api.sendCampaign).mockResolvedValue(campaign({ status: "sending" }));
    renderDialog();
    fireEvent.click(screen.getByRole("button", { name: "Continuar" }));
    fireEvent.change(await screen.findByLabelText("Código do autenticador"), { target: { value: "123456" } });
    fireEvent.click(screen.getByRole("button", { name: "Enviar agora" }));

    await waitFor(() => expect(api.sendCampaign).toHaveBeenCalled());
    expect(api.stepUpMfa).toHaveBeenCalledWith("123456");
  });

  it("agendar: recusa horário a menos de 5 minutos, sem chamar a API", () => {
    renderDialog();
    chooseSchedule("2026-09-29T10:03");
    expect(screen.getByRole("alert").textContent).toBe("agende para pelo menos 5 minutos a partir de agora");
    expect(screen.queryByRole("region", { name: "Agendar campanha" })).toBeNull();
  });

  it("agendar: manda o instante lido no fuso da cidade", async () => {
    const scheduled = campaign({ status: "scheduled", send_at: "2026-09-29T17:30:00.000Z" });
    mocked(api.scheduleCampaign).mockResolvedValue(scheduled);
    const { onDone } = renderDialog();
    chooseSchedule("2026-09-29T14:30");

    const confirmButton = await screen.findByRole("button", { name: "Agendar" });
    expect(screen.getByRole("region", { name: "Agendar campanha" }).textContent).toMatch(/29\/09\/2026.*14:30/);
    fireEvent.click(confirmButton);
    await waitFor(() => expect(api.scheduleCampaign).toHaveBeenCalledWith("c1", "2026-09-29T17:30:00.000Z"));
    expect(onDone).toHaveBeenCalledWith(scheduled);
  });

  it("servidor recusa invalid_send_at: mensagem traduzida e o diálogo continua", async () => {
    mocked(api.scheduleCampaign).mockRejectedValue(new ApiError(422, { error: "invalid_send_at" }, "422"));
    const { onDone } = renderDialog();
    chooseSchedule("2026-09-29T10:06");
    fireEvent.click(await screen.findByRole("button", { name: "Agendar" }));

    expect(await screen.findByText("horário fora da janela: agende de 5 minutos a 90 dias a partir de agora")).toBeTruthy();
    expect(screen.getByRole("region", { name: "Agendar campanha" })).toBeTruthy();
    expect(onDone).not.toHaveBeenCalled();
  });

  it("below_minimum: devolve ao editor, sem chamar onDone", async () => {
    mocked(api.sendCampaign).mockRejectedValue(new ApiError(422, { error: "below_minimum" }, "422"));
    const { onDone, onBelowMinimum } = renderDialog();
    fireEvent.click(screen.getByRole("button", { name: "Continuar" }));
    fireEvent.click(await screen.findByRole("button", { name: "Enviar agora" }));

    await waitFor(() => expect(onBelowMinimum).toHaveBeenCalled());
    expect(onDone).not.toHaveBeenCalled();
  });

  it("canais", () => {
    expect(channelsText(false)).toBe("aviso no wpda (SMS desligado nesta cidade)");
    expect(channelsText(null)).toBe("aviso no wpda (não foi possível consultar o SMS da cidade)");
  });

  it("Esc e Cancelar fecham", () => {
    const { onCancel } = renderDialog();
    fireEvent.keyDown(screen.getByRole("dialog", { name: "Como enviar" }), { key: "Escape" });
    fireEvent.click(screen.getByRole("button", { name: "Cancelar" }));
    expect(onCancel).toHaveBeenCalledTimes(2);
  });
});
```

- [ ] **Step 2: Testes novos no editor**

Em `src/modules/campaigns/CampaignEditor.test.tsx`, troque `renderEditor` por:

```tsx
function renderEditor(campaignId: string | null = null, previewDelayMs = 0) {
  const onBack = vi.fn();
  const onLeftDraft = vi.fn();
  renderWithProviders(<CampaignEditor campaignId={campaignId} onBack={onBack} onLeftDraft={onLeftDraft}
    onGoToSecurity={vi.fn()} previewDelayMs={previewDelayMs} />);
  return { onBack, onLeftDraft };
}
```

e acrescente, dentro do `describe("CampaignEditor")`:

```tsx
  describe("enviar", () => {
    const sendButton = () => screen.getByRole("button", { name: "Revisar e enviar…" }) as HTMLButtonElement;

    it("envio travado com menos de 5 telefones", async () => {
      mocked(api.previewAudience).mockResolvedValue({ below_minimum: true });
      renderEditor();
      await screen.findByText("menos de 5 — ajuste o público");
      expect(sendButton().disabled).toBe(true);
    });

    it("envio travado enquanto a contagem não alcançou o público atual", async () => {
      mocked(api.getCampaign).mockResolvedValue(campaign({ audience: BOQUEIRAO }));
      renderEditor("c1", 200);
      await screen.findByText("≈ 12 pessoas (9 telefones)");
      expect(sendButton().disabled).toBe(false);

      fireEvent.click(screen.getByRole("checkbox", { name: "Xaxim" }));
      expect(sendButton().disabled).toBe(true);
      expect(screen.getByRole("status", { name: "Contagem do público" }).textContent).toBe("calculando…");
      await waitFor(() => expect(sendButton().disabled).toBe(false));
    });

    it("revisar e enviar: salva antes, abre o diálogo com a frase e sai do rascunho", async () => {
      mocked(api.getCampaign).mockResolvedValue(campaign({ audience: BOQUEIRAO }));
      mocked(api.updateCampaign).mockResolvedValue(campaign({ audience: BOQUEIRAO }));
      const sent = campaign({ audience: BOQUEIRAO, status: "sending" });
      mocked(api.sendCampaign).mockResolvedValue(sent);
      const { onLeftDraft } = renderEditor("c1");
      await screen.findByText("≈ 12 pessoas (9 telefones)");

      fireEvent.click(sendButton());
      await waitFor(() => expect(api.updateCampaign).toHaveBeenCalled());
      const dialog = await screen.findByRole("dialog", { name: "Como enviar" });
      expect(dialog.textContent).toContain("moradores de Boqueirão");
      fireEvent.click(screen.getByRole("button", { name: "Continuar" }));
      fireEvent.click(await screen.findByRole("button", { name: "Enviar agora" }));

      await waitFor(() => expect(onLeftDraft).toHaveBeenCalledWith(sent));
    });

    it("below_minimum no envio: volta ao editor com a mensagem e recalcula", async () => {
      mocked(api.getCampaign).mockResolvedValue(campaign({ audience: BOQUEIRAO }));
      mocked(api.updateCampaign).mockResolvedValue(campaign({ audience: BOQUEIRAO }));
      mocked(api.previewAudience).mockResolvedValueOnce({ citizens: 12, phones: 9 }).mockResolvedValue({ below_minimum: true });
      mocked(api.sendCampaign).mockRejectedValue(new ApiError(422, { error: "below_minimum" }, "422"));
      const { onLeftDraft } = renderEditor("c1");
      await screen.findByText("≈ 12 pessoas (9 telefones)");

      fireEvent.click(sendButton());
      fireEvent.click(await screen.findByRole("button", { name: "Continuar" }));
      fireEvent.click(await screen.findByRole("button", { name: "Enviar agora" }));

      expect(await screen.findByText("o público ficou com menos de 5 telefones desde a contagem — ajuste o público")).toBeTruthy();
      expect(screen.queryByRole("region", { name: "Enviar campanha" })).toBeNull();
      expect(await screen.findByText("menos de 5 — ajuste o público")).toBeTruthy();
      expect(onLeftDraft).not.toHaveBeenCalled();
    });
  });
```

- [ ] **Step 3: Rode e veja falhar**

Run: `cd apps/dashboard/.claude/mod12 && npx vitest run src/modules/campaigns/SendDialog.test.tsx src/modules/campaigns/CampaignEditor.test.tsx`
Expected: FAIL. `./SendDialog` não resolve, e o editor não tem o botão "Revisar e enviar…" (o tsc também acusa `onLeftDraft` desconhecido).

- [ ] **Step 4: Diálogo**

```tsx
// src/modules/campaigns/SendDialog.tsx
import { useRef, useState } from "react";
import { scheduleCampaign, sendCampaign, type Campaign } from "../../lib/api";
import { campaignErrorCode, campaignErrorOrNull, validateSendAt } from "../../lib/campaigns";
import { fmtDateTime, fmtNumber } from "../../lib/format";
import { SensitiveAction } from "../../components/SensitiveAction";
import { buttonStyle, inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { alertStyle, cardStyle, checkStyle, labelStyle, rowStyle } from "./styles";

// Envio (spec 2026-09-29 §7, D5, D9): primeiro escolhe agora ou agendar (a
// data é validada aqui, antes de gastar um código TOTP), depois confirma com
// step-up no SensitiveAction. `below_minimum` não é erro do diálogo: o
// público encolheu, e a pessoa volta ao editor (onBelowMinimum).
export interface SendDialogProps {
  campaign: Campaign;
  phrase: string;
  counts: { citizens: number; phones: number };
  smsEnabled: boolean | null;
  onDone(campaign: Campaign): void;
  onBelowMinimum(): void;
  onCancel(): void;
  onGoToSecurity(): void;
}

type Stage = { kind: "choose" } | { kind: "now" } | { kind: "schedule"; iso: string };

export function channelsText(smsEnabled: boolean | null): string {
  if (smsEnabled === true) return "aviso no wpda + SMS para quem aceitou receber (o SMS só sai entre 8h e 20h)";
  if (smsEnabled === false) return "aviso no wpda (SMS desligado nesta cidade)";
  return "aviso no wpda (não foi possível consultar o SMS da cidade)";
}

export function SendDialog({ campaign, phrase, counts, smsEnabled, onDone, onBelowMinimum, onCancel, onGoToSecurity }: SendDialogProps) {
  const [ stage, setStage ] = useState<Stage>({ kind: "choose" });
  const [ mode, setMode ] = useState<"now" | "schedule">("now");
  const [ when, setWhen ] = useState("");
  const [ whenError, setWhenError ] = useState<string | null>(null);
  const result = useRef<Campaign | null>(null);
  const belowMinimum = useRef(false);

  function proceed() {
    if (mode === "now") { setStage({ kind: "now" }); return; }
    const check = validateSendAt(when, new Date());
    if (!check.ok) { setWhenError(check.message); return; }
    setWhenError(null);
    setStage({ kind: "schedule", iso: check.iso });
  }

  const summary = (whenText: string | null) => (
    <span style={{ display: "flex", flexDirection: "column", gap: 4 }}>
      <span><strong>Público:</strong> {phrase}</span>
      <span><strong>Contagem:</strong> ≈ {fmtNumber(counts.citizens)} pessoas ({fmtNumber(counts.phones)} telefones)</span>
      <span><strong>Canais:</strong> {channelsText(smsEnabled)}</span>
      {whenText && <span><strong>Quando:</strong> {whenText}</span>}
    </span>
  );

  if (stage.kind !== "choose") {
    const scheduling = stage.kind === "schedule";
    return (
      <SensitiveAction
        title={scheduling ? "Agendar campanha" : "Enviar campanha"}
        description={summary(stage.kind === "schedule" ? fmtDateTime(stage.iso) : "agora")}
        requiresStepUp
        confirmLabel={scheduling ? "Agendar" : "Enviar agora"}
        run={async () => {
          belowMinimum.current = false;
          try {
            result.current = stage.kind === "schedule"
              ? await scheduleCampaign(campaign.id, stage.iso)
              : await sendCampaign(campaign.id);
          } catch (err) {
            if (campaignErrorCode(err) === "below_minimum") { belowMinimum.current = true; return; }
            throw err;
          }
        }}
        onDone={() => (belowMinimum.current ? onBelowMinimum() : onDone(result.current as Campaign))}
        onCancel={onCancel}
        onGoToSecurity={onGoToSecurity}
        translateError={campaignErrorOrNull}
      />
    );
  }

  return (
    <section role="dialog" aria-label="Como enviar" style={cardStyle}
      onKeyDown={(e) => { if (e.key === "Escape") onCancel(); }}>
      <strong>Enviar “{campaign.title}”</strong>
      <div style={{ fontSize: 12.5, color: "var(--ink2)" }}>{summary(null)}</div>
      <div role="radiogroup" aria-label="Quando enviar" style={{ display: "flex", gap: 16 }}>
        <label style={checkStyle}>
          <input type="radio" name="campaign-send-mode" checked={mode === "now"} onChange={() => setMode("now")} />
          Enviar agora
        </label>
        <label style={checkStyle}>
          <input type="radio" name="campaign-send-mode" checked={mode === "schedule"} onChange={() => setMode("schedule")} />
          Agendar para
        </label>
      </div>
      {mode === "schedule" && (
        <label style={{ ...labelStyle, maxWidth: 280 }}>
          Data e hora (horário da cidade)
          <input type="datetime-local" value={when} onChange={(e) => setWhen(e.target.value)} style={inputStyle} />
        </label>
      )}
      {whenError && <p role="alert" style={alertStyle}>{whenError}</p>}
      <div style={rowStyle}>
        <button type="button" style={buttonStyle} onClick={proceed}>Continuar</button>
        <button type="button" style={secondaryButtonStyle} onClick={onCancel}>Cancelar</button>
      </div>
    </section>
  );
}
```

- [ ] **Step 5: Editor com o botão e o diálogo**

Em `src/modules/campaigns/CampaignEditor.tsx`:

1. Imports:
   - acrescente `PREVIEW_KEY` ao import de `../../lib/campaigns`;
   - troque `import { AudienceCounter } from "./AudienceCounter";` por `import { AudienceCounter, previewAllowsSend } from "./AudienceCounter";`;
   - acrescente:

```tsx
import { describeAudience } from "../../lib/audiencePhrase";
import { SendDialog } from "./SendDialog";
```

2. Troque a interface:

```tsx
export interface CampaignEditorProps {
  campaignId: string | null;
  onBack(): void;
  onLeftDraft(campaign: Campaign): void;
  onGoToSecurity(): void;
  previewDelayMs?: number;
}
```

3. Troque a assinatura de `EditorForm`:

```tsx
function EditorForm({ initial, options, onBack, onLeftDraft, onGoToSecurity, previewDelayMs = PREVIEW_DEBOUNCE_MS }: CampaignEditorProps & {
  initial: Campaign | null; options: CampaignOptions;
}) {
```

4. Depois de `const [ notice, setNotice ] = …`, acrescente:

```tsx
  // A contagem é congelada ao abrir o diálogo: o que a pessoa confirma é o
  // número que ela viu, mesmo que o público mude atrás do modal.
  const [ reviewing, setReviewing ] = useState<{ campaign: Campaign; counts: { citizens: number; phones: number } } | null>(null);
```

5. Depois da função `save`, acrescente:

```tsx
  async function review() {
    if (!previewAllowsSend(preview)) return;
    const counts = { citizens: preview.citizens, phones: preview.phones };
    const saved = await save();
    if (saved) setReviewing({ campaign: saved, counts });
  }

  function leftDraft(c: Campaign) {
    setReviewing(null);
    queryClient.setQueryData(campaignKey(c.id), c);
    void queryClient.invalidateQueries({ queryKey: CAMPAIGNS_KEY });
    onLeftDraft(c);
  }

  function belowMinimum() {
    setReviewing(null);
    fail("o público ficou com menos de 5 telefones desde a contagem — ajuste o público");
    void queryClient.invalidateQueries({ queryKey: PREVIEW_KEY });
  }

  const canSend = previewAllowsSend(preview) && !busy;
```

6. Troque o bloco final `<div style={rowStyle}>…Salvar rascunho…</div>` por:

```tsx
      {reviewing && (
        <SendDialog
          key={reviewing.campaign.id}
          campaign={reviewing.campaign}
          phrase={describeAudience(reviewing.campaign.audience, options)}
          counts={reviewing.counts}
          smsEnabled={sms.data?.enabled ?? null}
          onDone={leftDraft}
          onBelowMinimum={belowMinimum}
          onCancel={() => setReviewing(null)}
          onGoToSecurity={onGoToSecurity}
        />
      )}

      <div style={rowStyle}>
        <button type="button" disabled={busy} onClick={() => void save()} style={busy ? disabledButtonStyle : secondaryButtonStyle}>
          Salvar rascunho
        </button>
        <button type="button" disabled={!canSend} onClick={() => void review()} style={canSend ? buttonStyle : disabledButtonStyle}>
          Revisar e enviar…
        </button>
        {!canSend && !busy && <span style={noteStyle}>o envio libera quando a contagem mostrar pelo menos 5 telefones</span>}
      </div>
```

- [ ] **Step 6: Rode, cheque tipos e faça commit**

Run: `cd apps/dashboard/.claude/mod12 && npx vitest run src/modules/campaigns/SendDialog.test.tsx src/modules/campaigns/CampaignEditor.test.tsx && npx tsc --noEmit`
Expected: PASS; tsc sem erros.

```bash
cd apps/dashboard/.claude/mod12
/opt/homebrew/bin/git add src/modules/campaigns/SendDialog.tsx src/modules/campaigns/SendDialog.test.tsx src/modules/campaigns/CampaignEditor.tsx src/modules/campaigns/CampaignEditor.test.tsx
/opt/homebrew/bin/git commit -m "feat: send or schedule a campaign with step-up from the editor

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 9: Painel da campanha, desagendar e cancelar

**Files:**
- Create: `src/modules/campaigns/CampaignLifecycleDialog.tsx`, `src/modules/campaigns/CampaignPanel.tsx`
- Modify: `src/modules/campaigns/CampaignEditor.tsx`
- Test: `src/modules/campaigns/CampaignPanel.test.tsx`, `src/modules/campaigns/CampaignEditor.test.tsx`

**Interfaces:**
- Consumes:
  - da Task 1: `cancelCampaign`, `getCampaign`, `getCampaignOptions`, `unscheduleCampaign`, `Campaign`, `CampaignStats`, `SmsStatus`;
  - da Task 2: `CAMPAIGNS_KEY`, `CAMPAIGN_OPTIONS_KEY`, `FAILURE_LABEL`, `SMS_STATUS_LABEL`, `SMS_STATUS_ORDER`, `STATUS_LABEL`, `STATUS_TONE`, `campaignError`, `campaignErrorOrNull`, `campaignKey`;
  - da Task 3: `describeAudience`.
- Produces:
  - `LifecycleAction = "unschedule" | "cancel"`;
  - `CampaignLifecycleDialog({ campaign: Pick<Campaign, "id" | "title">; action; onDone(c: Campaign); onCancel(); onGoToSecurity() })`;
  - `SENDING_REFETCH_MS = 5000`;
  - `CampaignPanel({ campaignId; onBack(); onEdit(id: string); onGoToSecurity() })`;
  - `statusLine(c: Campaign) → string`;
  - `GATEWAY_ALERT`, com o texto literal da spec.
- Nomes acessíveis:
  - botões "Desagendar…" e "Cancelar campanha…";
  - os `SensitiveAction` "Desagendar campanha" e "Cancelar campanha", que confirmam com "Desagendar" e "Cancelar campanha";
  - painel "Resultado".

- [ ] **Step 1: Escreva os testes do painel**

```tsx
// src/modules/campaigns/CampaignPanel.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return {
    ...real, fetchCurrentSession: vi.fn(), stepUpMfa: vi.fn(), getCampaign: vi.fn(), getCampaignOptions: vi.fn(),
    unscheduleCampaign: vi.fn(), cancelCampaign: vi.fn()
  };
});

import * as api from "../../lib/api";
import { ApiError, type CampaignStats } from "../../lib/api";
import { CampaignPanel, SENDING_REFETCH_MS } from "./CampaignPanel";
import { OPTIONS, campaign, renderWithProviders, sessionWith } from "../../test/campaignFixtures";

afterEach(() => { cleanup(); vi.useRealTimers(); });
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

const sms = (over: Partial<CampaignStats["sms"]> = {}): CampaignStats["sms"] => ({
  not_opted_in: 60, duplicate_phone: 2, pending: 0, deferred: 0, sent: 58, failed: 0, unavailable: 0, ...over
});
const sent = (over: Parameters<typeof campaign>[0] = {}) => campaign({
  status: "sent", dispatched_at: "2026-09-29T10:00:00-03:00", recipients_count: 120, phones_count: 98, sms_enabled: true,
  audience: { version: 1, geo: { scope: "neighborhoods", neighborhood_ids: [ "n1", "n2" ] },
    clinical: { all: [ { kind: "appointment_no_show", from: "2026-07-01", to: "2026-09-29" } ] } },
  stats: { read_count: 30, sms: sms() }, ...over
});

function renderPanel() {
  const onBack = vi.fn();
  const onEdit = vi.fn();
  renderWithProviders(<CampaignPanel campaignId="c1" onBack={onBack} onEdit={onEdit} onGoToSecurity={vi.fn()} />);
  return { onBack, onEdit };
}

describe("CampaignPanel", () => {
  beforeEach(() => {
    vi.resetAllMocks();
    mocked(api.fetchCurrentSession).mockResolvedValue(sessionWith([ "campaign_manager" ]));
    mocked(api.getCampaignOptions).mockResolvedValue(OPTIONS);
  });

  it("enviada: destinatários, telefones, lidos com percentual e o público em frase", async () => {
    mocked(api.getCampaign).mockResolvedValue(sent());
    renderPanel();
    const result = within(await screen.findByRole("region", { name: "Resultado" }));
    expect(result.getByText("120")).toBeTruthy();
    expect(result.getByText("98")).toBeTruthy();
    expect(result.getByText("30 (25%)")).toBeTruthy();
    expect(await screen.findByText(/moradores de Boqueirão e Xaxim que faltaram a um agendamento/)).toBeTruthy();
  });

  it("SMS por status, sem nenhuma lista de pessoas", async () => {
    mocked(api.getCampaign).mockResolvedValue(sent());
    renderPanel();
    const result = within(await screen.findByRole("region", { name: "Resultado" }));
    expect(result.getByText("enviado")).toBeTruthy();
    expect(result.getByText("58")).toBeTruthy();
    expect(result.getByText("sem opt-in de SMS")).toBeTruthy();
    expect(result.queryByText(/\*\*\*/)).toBeNull();
  });

  it("alerta destacado quando o SMS ficou sem provedor", async () => {
    mocked(api.getCampaign).mockResolvedValue(sent({ stats: { read_count: 0, sms: sms({ sent: 0, unavailable: 58 }) } }));
    renderPanel();
    const alerts = await screen.findAllByRole("alert");
    expect(alerts.some((a) => a.textContent?.startsWith("SMS não enviado: a plataforma ainda não tem provedor de SMS"))).toBe(true);
  });

  it("alerta quando algum SMS falhou", async () => {
    mocked(api.getCampaign).mockResolvedValue(sent({ stats: { read_count: 0, sms: sms({ failed: 3 }) } }));
    renderPanel();
    expect(await screen.findByText("3 SMS falharam no envio")).toBeTruthy();
  });

  it("SMS desligado no envio: diz isso e não mostra a tabela", async () => {
    mocked(api.getCampaign).mockResolvedValue(sent({ sms_enabled: false, stats: { read_count: 1, sms: sms({ sent: 0 }) } }));
    renderPanel();
    expect(await screen.findByText("SMS desligado nesta cidade no momento do envio: só o aviso no wpda.")).toBeTruthy();
    expect(screen.queryByText("sem opt-in de SMS")).toBeNull();
  });

  it("falhou: mostra o motivo", async () => {
    mocked(api.getCampaign).mockResolvedValue(campaign({ status: "failed", failure_reason: "below_minimum" }));
    renderPanel();
    expect(await screen.findByText("Não enviada: o público tinha menos de 5 telefones no momento do envio")).toBeTruthy();
    expect(screen.queryByRole("region", { name: "Resultado" })).toBeNull();
  });

  it("agendada: desagendar com step-up volta ao editor", async () => {
    mocked(api.getCampaign).mockResolvedValue(campaign({ status: "scheduled", send_at: "2026-09-30T12:00:00.000Z" }));
    mocked(api.unscheduleCampaign).mockResolvedValue(campaign({ status: "draft" }));
    const { onEdit } = renderPanel();
    expect(await screen.findByText(/agendada para 30\/09\/2026.*09:00/)).toBeTruthy();
    fireEvent.click(screen.getByRole("button", { name: "Desagendar…" }));
    fireEvent.click(await screen.findByRole("button", { name: "Desagendar" }));
    await waitFor(() => expect(api.unscheduleCampaign).toHaveBeenCalledWith("c1"));
    expect(onEdit).toHaveBeenCalledWith("c1");
  });

  it("agendada: cancelar com step-up fica no painel, agora cancelada", async () => {
    mocked(api.getCampaign).mockResolvedValue(campaign({ status: "scheduled", send_at: "2026-09-30T12:00:00.000Z" }));
    mocked(api.cancelCampaign).mockResolvedValue(campaign({ status: "cancelled" }));
    const { onEdit } = renderPanel();
    fireEvent.click(await screen.findByRole("button", { name: "Cancelar campanha…" }));
    fireEvent.click(await screen.findByRole("button", { name: "Cancelar campanha" }));
    await waitFor(() => expect(api.cancelCampaign).toHaveBeenCalledWith("c1"));
    // A etiqueta de status e o subtítulo do painel dizem "cancelada".
    expect((await screen.findAllByText("cancelada")).length).toBe(2);
    expect(screen.queryByRole("button", { name: "Desagendar…" })).toBeNull();
    expect(onEdit).not.toHaveBeenCalled();
  });

  it("recusa invalid_transition aparece traduzida", async () => {
    mocked(api.getCampaign).mockResolvedValue(campaign({ status: "scheduled", send_at: "2026-09-30T12:00:00.000Z" }));
    mocked(api.unscheduleCampaign).mockRejectedValue(new ApiError(422, { error: "invalid_transition" }, "422"));
    renderPanel();
    fireEvent.click(await screen.findByRole("button", { name: "Desagendar…" }));
    fireEvent.click(await screen.findByRole("button", { name: "Desagendar" }));
    expect(await screen.findByText("a campanha mudou de estado enquanto você decidia — volte à lista e abra de novo")).toBeTruthy();
  });

  it("enviando: relê sozinho até sair de 'enviando'", async () => {
    vi.useFakeTimers({ toFake: [ "setTimeout", "clearTimeout", "setInterval", "clearInterval" ], shouldAdvanceTime: true });
    mocked(api.getCampaign).mockResolvedValueOnce(campaign({ status: "sending" })).mockResolvedValue(sent());
    renderPanel();
    expect(await screen.findByText("enviando")).toBeTruthy();
    await vi.advanceTimersByTimeAsync(SENDING_REFETCH_MS);
    expect(await screen.findByRole("region", { name: "Resultado" })).toBeTruthy();
  });
});
```

- [ ] **Step 2: Teste novo no editor**

Em `src/modules/campaigns/CampaignEditor.test.tsx`, dentro do `describe("CampaignEditor")`:

```tsx
  it("rascunho salvo pode ser cancelado, com step-up", async () => {
    mocked(api.getCampaign).mockResolvedValue(campaign({ audience: BOQUEIRAO }));
    const cancelled = campaign({ audience: BOQUEIRAO, status: "cancelled" });
    mocked(api.cancelCampaign).mockResolvedValue(cancelled);
    const { onLeftDraft } = renderEditor("c1");
    fireEvent.click(await screen.findByRole("button", { name: "Cancelar campanha…" }));
    fireEvent.click(await screen.findByRole("button", { name: "Cancelar campanha" }));
    await waitFor(() => expect(api.cancelCampaign).toHaveBeenCalledWith("c1"));
    expect(onLeftDraft).toHaveBeenCalledWith(cancelled);
  });

  it("rascunho novo, ainda não salvo, não tem 'Cancelar campanha…'", async () => {
    renderEditor();
    await screen.findByLabelText("Título");
    expect(screen.queryByRole("button", { name: "Cancelar campanha…" })).toBeNull();
  });
```

- [ ] **Step 3: Rode e veja falhar**

Run: `cd apps/dashboard/.claude/mod12 && npx vitest run src/modules/campaigns/CampaignPanel.test.tsx src/modules/campaigns/CampaignEditor.test.tsx`
Expected: FAIL. `./CampaignPanel` não resolve, e o editor não tem "Cancelar campanha…".

- [ ] **Step 4: Diálogo de desagendar e cancelar**

```tsx
// src/modules/campaigns/CampaignLifecycleDialog.tsx
import { useRef } from "react";
import { cancelCampaign, unscheduleCampaign, type Campaign } from "../../lib/api";
import { campaignErrorOrNull } from "../../lib/campaigns";
import { SensitiveAction } from "../../components/SensitiveAction";

// Desagendar (scheduled → draft) e cancelar (draft/scheduled → cancelled),
// ambos com step-up (spec §5.1, D5).
export type LifecycleAction = "unschedule" | "cancel";

const COPY: Record<LifecycleAction, { title: string; description: string; confirm: string }> = {
  unschedule: {
    title: "Desagendar campanha",
    description: "A campanha volta a rascunho e não sai no horário marcado. Depois dá para editar e enviar de novo.",
    confirm: "Desagendar"
  },
  cancel: {
    title: "Cancelar campanha",
    description: "A campanha é cancelada: não sai, e não pode mais ser editada nem enviada.",
    confirm: "Cancelar campanha"
  }
};

export function CampaignLifecycleDialog({ campaign, action, onDone, onCancel, onGoToSecurity }: {
  campaign: Pick<Campaign, "id" | "title">;
  action: LifecycleAction;
  onDone(campaign: Campaign): void;
  onCancel(): void;
  onGoToSecurity(): void;
}) {
  const result = useRef<Campaign | null>(null);
  const copy = COPY[action];
  return (
    <SensitiveAction
      title={copy.title}
      description={`“${campaign.title}”: ${copy.description}`}
      requiresStepUp
      confirmLabel={copy.confirm}
      run={async () => {
        result.current = action === "unschedule" ? await unscheduleCampaign(campaign.id) : await cancelCampaign(campaign.id);
      }}
      onDone={() => onDone(result.current as Campaign)}
      onCancel={onCancel}
      onGoToSecurity={onGoToSecurity}
      translateError={campaignErrorOrNull}
    />
  );
}
```

- [ ] **Step 5: Painel**

```tsx
// src/modules/campaigns/CampaignPanel.tsx
import { useState } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { getCampaign, getCampaignOptions, type Campaign, type CampaignStats, type SmsStatus } from "../../lib/api";
import {
  CAMPAIGNS_KEY, CAMPAIGN_OPTIONS_KEY, FAILURE_LABEL, SMS_STATUS_LABEL, SMS_STATUS_ORDER, STATUS_LABEL, STATUS_TONE,
  campaignError, campaignKey
} from "../../lib/campaigns";
import { describeAudience } from "../../lib/audiencePhrase";
import { fmtDateTime, fmtNumber, fmtPercent } from "../../lib/format";
import { Panel } from "../../components/Panel";
import { DataTable } from "../../components/DataTable";
import { Tag } from "../../components/Tag";
import { secondaryButtonStyle } from "../../components/formStyles";
import { CampaignLifecycleDialog, type LifecycleAction } from "./CampaignLifecycleDialog";
import { alertStyle, columnStyle, noteStyle, rowStyle, warnStyle } from "./styles";

// Painel da campanha (spec 2026-09-29 §6.3 e §7): só agregados, nenhuma lista
// de destinatários. Enquanto está "enviando", relê sozinho.
export const SENDING_REFETCH_MS = 5_000;
export const GATEWAY_ALERT = "SMS não enviado: a plataforma ainda não tem provedor de SMS";

export function statusLine(c: Campaign): string {
  switch (c.status) {
    case "draft": return "rascunho";
    case "scheduled": return `agendada para ${fmtDateTime(c.send_at)}`;
    case "sending": return "enviando — congelando o público";
    case "sent": return `enviada em ${fmtDateTime(c.dispatched_at)}`;
    case "failed": return "não enviada";
    case "cancelled": return "cancelada";
  }
}

export interface CampaignPanelProps {
  campaignId: string;
  onBack(): void;
  onEdit(id: string): void;
  onGoToSecurity(): void;
}

export function CampaignPanel({ campaignId, onBack, onEdit, onGoToSecurity }: CampaignPanelProps) {
  const queryClient = useQueryClient();
  const query = useQuery({
    queryKey: campaignKey(campaignId),
    queryFn: () => getCampaign(campaignId),
    refetchInterval: (q) => (q.state.data?.status === "sending" ? SENDING_REFETCH_MS : false)
  });
  const options = useQuery({ queryKey: CAMPAIGN_OPTIONS_KEY, queryFn: getCampaignOptions });
  const [ acting, setActing ] = useState<LifecycleAction | null>(null);
  const back = <button type="button" style={secondaryButtonStyle} onClick={onBack}>Voltar à lista</button>;

  if (query.isError) return <Panel title="Campanha" right={back}><p role="alert" style={alertStyle}>{campaignError(query.error)}</p></Panel>;
  if (query.isPending) return <Panel title="Campanha" right={back}><p className="mono" style={noteStyle}>carregando…</p></Panel>;
  const c = query.data;
  const phrase = options.data
    ? describeAudience(c.audience, options.data)
    : options.isError ? describeAudience(c.audience, { neighborhoods: [], units: [] }) : "carregando o público…";

  function finished(next: Campaign) {
    setActing(null);
    queryClient.setQueryData(campaignKey(next.id), next);
    void queryClient.invalidateQueries({ queryKey: CAMPAIGNS_KEY });
    if (next.status === "draft") onEdit(next.id);
  }

  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 16 }}>
      <Panel title={c.title} sub={statusLine(c)} right={
        <div style={{ display: "flex", gap: 8, alignItems: "center" }}>
          <Tag tone={STATUS_TONE[c.status]}>{STATUS_LABEL[c.status]}</Tag>
          {back}
        </div>
      }>
        <div style={columnStyle}>
          {c.status === "failed" && (
            <p role="alert" style={warnStyle}>
              Não enviada: {FAILURE_LABEL[c.failure_reason ?? ""] ?? "motivo não informado"}
            </p>
          )}
          <p style={{ margin: 0, fontSize: 13 }}><strong>Público:</strong> {phrase}</p>
          {c.status === "scheduled" && !acting && (
            <div style={rowStyle}>
              <button type="button" style={secondaryButtonStyle} onClick={() => setActing("unschedule")}>Desagendar…</button>
              <button type="button" style={secondaryButtonStyle} onClick={() => setActing("cancel")}>Cancelar campanha…</button>
            </div>
          )}
          {acting && (
            <CampaignLifecycleDialog key={acting} campaign={c} action={acting} onDone={finished}
              onCancel={() => setActing(null)} onGoToSecurity={onGoToSecurity} />
          )}
        </div>
      </Panel>

      {c.stats && <Aggregates campaign={c} stats={c.stats} />}

      <Panel title="Aviso" sub="como está no wpda">
        <article aria-label="Texto do aviso" style={{ fontSize: 15, lineHeight: 1.5 }}>
          <p style={{ margin: 0, whiteSpace: "pre-wrap" }}>{c.body}</p>
        </article>
      </Panel>
    </div>
  );
}

function Aggregates({ campaign: c, stats }: { campaign: Campaign; stats: CampaignStats }) {
  const recipients = c.recipients_count ?? 0;
  const readPct = recipients > 0 ? fmtPercent((stats.read_count / recipients) * 100) : "—";
  const unavailable = stats.sms.unavailable ?? 0;
  const failed = stats.sms.failed ?? 0;

  return (
    <Panel title="Resultado" sub="só contagens — nenhuma lista de pessoas">
      <div style={columnStyle}>
        {unavailable > 0 && (
          <p role="alert" style={warnStyle}>
            {GATEWAY_ALERT} ({fmtNumber(unavailable)} {unavailable === 1 ? "pessoa" : "pessoas"})
          </p>
        )}
        {failed > 0 && (
          <p role="alert" style={warnStyle}>{fmtNumber(failed)} SMS {failed === 1 ? "falhou" : "falharam"} no envio</p>
        )}
        <dl style={{ display: "flex", gap: 32, margin: 0, flexWrap: "wrap" }}>
          <Stat label="Destinatários" value={fmtNumber(c.recipients_count)} />
          <Stat label="Telefones" value={fmtNumber(c.phones_count)} />
          <Stat label="Lidos" value={`${fmtNumber(stats.read_count)} (${readPct})`} />
        </dl>
        {c.sms_enabled === false ? (
          <p style={noteStyle}>SMS desligado nesta cidade no momento do envio: só o aviso no wpda.</p>
        ) : (
          <DataTable<SmsStatus>
            cols={[
              { label: "SMS", w: "3fr", render: (s) => SMS_STATUS_LABEL[s] },
              { label: "Pessoas", w: "1fr", align: "right", render: (s) => <span className="mono">{fmtNumber(stats.sms[s] ?? 0)}</span> }
            ]}
            rows={SMS_STATUS_ORDER}
            rowKey={(s) => s}
          />
        )}
      </div>
    </Panel>
  );
}

function Stat({ label, value }: { label: string; value: string }) {
  return (
    <div>
      <dt style={noteStyle}>{label}</dt>
      <dd style={{ margin: 0, fontSize: 20, fontWeight: 700 }}>{value}</dd>
    </div>
  );
}
```

- [ ] **Step 6: Cancelar rascunho no editor**

Em `src/modules/campaigns/CampaignEditor.tsx`:

1. Acrescente o import `import { CampaignLifecycleDialog } from "./CampaignLifecycleDialog";`.
2. Depois do estado `reviewing`, acrescente `const [ cancelling, setCancelling ] = useState(false);`.
3. Logo antes de `<div style={rowStyle}>` (a fileira de botões do fim), acrescente:

```tsx
      {cancelling && id && (
        <CampaignLifecycleDialog
          campaign={{ id, title: title.trim() || initial?.title || "" }}
          action="cancel"
          onDone={(c) => { setCancelling(false); leftDraft(c); }}
          onCancel={() => setCancelling(false)}
          onGoToSecurity={onGoToSecurity}
        />
      )}
```

4. Na fileira de botões, depois do "Revisar e enviar…", acrescente:

```tsx
        {id && (
          <button type="button" disabled={busy} style={secondaryButtonStyle}
            onClick={() => { setReviewing(null); setCancelling(true); }}>
            Cancelar campanha…
          </button>
        )}
```

- [ ] **Step 7: Rode, cheque tipos e faça commit**

Run: `cd apps/dashboard/.claude/mod12 && npx vitest run src/modules/campaigns && npx tsc --noEmit`
Expected: PASS; tsc sem erros.

```bash
cd apps/dashboard/.claude/mod12
/opt/homebrew/bin/git add src/modules/campaigns/CampaignLifecycleDialog.tsx src/modules/campaigns/CampaignPanel.tsx src/modules/campaigns/CampaignPanel.test.tsx src/modules/campaigns/CampaignEditor.tsx src/modules/campaigns/CampaignEditor.test.tsx
/opt/homebrew/bin/git commit -m "feat: add campaign panel with aggregates, SMS alerts, unschedule and cancel

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 10: Chave de SMS da cidade

**Files:**
- Create: `src/modules/campaigns/SmsSettingPanel.tsx`
- Test: `src/modules/campaigns/SmsSettingPanel.test.tsx`

**Interfaces:**
- Consumes:
  - da Task 1: `getSmsSetting`, `setSmsSetting`, `SmsSetting`;
  - da Task 2: `SMS_SETTING_KEY`, `campaignError`, `campaignErrorOrNull`.
- Produces:
  - `GATEWAY_WARNING`, com o texto literal da spec;
  - `SmsSettingPanel({ onGoToSecurity })`, um `Panel` "SMS das campanhas";
  - os botões "Ligar SMS" e "Desligar SMS", que abrem os `SensitiveAction` "Ligar o SMS das campanhas" (confirma com "Ligar") e "Desligar o SMS das campanhas" (confirma com "Desligar").

- [ ] **Step 1: Escreva os testes**

```tsx
// src/modules/campaigns/SmsSettingPanel.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), stepUpMfa: vi.fn(), getSmsSetting: vi.fn(), setSmsSetting: vi.fn() };
});

import * as api from "../../lib/api";
import { ApiError } from "../../lib/api";
import { GATEWAY_WARNING, SmsSettingPanel } from "./SmsSettingPanel";
import { renderWithProviders, sessionWith } from "../../test/campaignFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const renderIt = () => renderWithProviders(<SmsSettingPanel onGoToSecurity={vi.fn()} />);
const panel = () => within(screen.getByRole("region", { name: "SMS das campanhas" }));

describe("SmsSettingPanel", () => {
  beforeEach(() => {
    vi.resetAllMocks();
    mocked(api.fetchCurrentSession).mockResolvedValue(sessionWith([ "municipal_admin" ]));
  });

  it("desligado, com provedor: sem aviso", async () => {
    mocked(api.getSmsSetting).mockResolvedValue({ enabled: false, gateway_configured: true });
    renderIt();
    expect(await screen.findByRole("button", { name: "Ligar SMS" })).toBeTruthy();
    expect(panel().getByText("desligado")).toBeTruthy();
    expect(screen.queryByText(GATEWAY_WARNING)).toBeNull();
  });

  it("sem provedor na plataforma: aviso literal da spec", async () => {
    mocked(api.getSmsSetting).mockResolvedValue({ enabled: false, gateway_configured: false });
    renderIt();
    expect((await screen.findByText(GATEWAY_WARNING)).getAttribute("role")).toBe("note");
    expect(GATEWAY_WARNING).toBe("A plataforma ainda não tem provedor de SMS: os avisos saem, o SMS fica pendente como não enviado");
  });

  it("ligar com step-up: grava, mostra ligado e mantém o aviso de provedor", async () => {
    mocked(api.getSmsSetting).mockResolvedValue({ enabled: false, gateway_configured: false });
    mocked(api.setSmsSetting).mockResolvedValue({ enabled: true, gateway_configured: false });
    renderIt();
    fireEvent.click(await screen.findByRole("button", { name: "Ligar SMS" }));
    fireEvent.click(await screen.findByRole("button", { name: "Ligar" }));

    await waitFor(() => expect(api.setSmsSetting).toHaveBeenCalledWith(true));
    expect((await screen.findByText("SMS ligado")).getAttribute("role")).toBe("status");
    expect(panel().getByText("ligado")).toBeTruthy();
    expect(screen.getByRole("button", { name: "Desligar SMS" })).toBeTruthy();
    expect(screen.getByText(GATEWAY_WARNING)).toBeTruthy();
  });

  it("janela fechada: pede o código", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(sessionWith([ "municipal_admin" ], { mfa_verified_at: null }));
    mocked(api.getSmsSetting).mockResolvedValue({ enabled: true, gateway_configured: true });
    mocked(api.stepUpMfa).mockResolvedValue(undefined);
    mocked(api.setSmsSetting).mockResolvedValue({ enabled: false, gateway_configured: true });
    renderIt();
    fireEvent.click(await screen.findByRole("button", { name: "Desligar SMS" }));
    fireEvent.change(await screen.findByLabelText("Código do autenticador"), { target: { value: "654321" } });
    fireEvent.click(screen.getByRole("button", { name: "Desligar" }));

    await waitFor(() => expect(api.setSmsSetting).toHaveBeenCalledWith(false));
    expect(api.stepUpMfa).toHaveBeenCalledWith("654321");
  });

  it("403: seu papel não permite", async () => {
    mocked(api.getSmsSetting).mockResolvedValue({ enabled: false, gateway_configured: true });
    mocked(api.setSmsSetting).mockRejectedValue(new ApiError(403, { error: "missing_role" }, "403"));
    renderIt();
    fireEvent.click(await screen.findByRole("button", { name: "Ligar SMS" }));
    fireEvent.click(await screen.findByRole("button", { name: "Ligar" }));
    expect(await screen.findByText("seu papel não permite esta ação")).toBeTruthy();
  });

  it("falha ao ler a chave", async () => {
    mocked(api.getSmsSetting).mockRejectedValue(new ApiError(500, "boom", "500"));
    renderIt();
    expect(await screen.findByText("não foi possível concluir — tente de novo")).toBeTruthy();
  });
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `cd apps/dashboard/.claude/mod12 && npx vitest run src/modules/campaigns/SmsSettingPanel.test.tsx`
Expected: FAIL com "Failed to resolve import ./SmsSettingPanel".

- [ ] **Step 3: Implemente**

```tsx
// src/modules/campaigns/SmsSettingPanel.tsx
import { useRef, useState } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { getSmsSetting, setSmsSetting, type SmsSetting } from "../../lib/api";
import { SMS_SETTING_KEY, campaignError, campaignErrorOrNull } from "../../lib/campaigns";
import { Panel } from "../../components/Panel";
import { Tag } from "../../components/Tag";
import { SensitiveAction } from "../../components/SensitiveAction";
import { buttonStyle } from "../../components/formStyles";
import { alertStyle, columnStyle, noteStyle, rowStyle, warnStyle } from "./styles";

// Chave de SMS da cidade (spec 2026-09-29 §7, D1): só o municipal_admin liga
// ou desliga, com step-up. Ligar sem provedor é permitido: os avisos saem e o
// SMS de cada destinatário fica "unavailable" — a tela avisa.
export const GATEWAY_WARNING =
  "A plataforma ainda não tem provedor de SMS: os avisos saem, o SMS fica pendente como não enviado";

export function SmsSettingPanel({ onGoToSecurity }: { onGoToSecurity(): void }) {
  const query = useQuery({ queryKey: SMS_SETTING_KEY, queryFn: getSmsSetting });
  return (
    <Panel title="SMS das campanhas" sub="chave da cidade · só o administrador municipal muda">
      {query.isError
        ? <p role="alert" style={alertStyle}>{campaignError(query.error)}</p>
        : query.isPending
          ? <p className="mono" style={noteStyle}>carregando…</p>
          : <SmsSwitch setting={query.data} onGoToSecurity={onGoToSecurity} />}
    </Panel>
  );
}

function SmsSwitch({ setting, onGoToSecurity }: { setting: SmsSetting; onGoToSecurity(): void }) {
  const queryClient = useQueryClient();
  const [ confirming, setConfirming ] = useState(false);
  const [ done, setDone ] = useState<string | null>(null);
  const result = useRef<SmsSetting | null>(null);
  const turningOn = !setting.enabled;

  return (
    <div style={columnStyle}>
      <div style={{ ...rowStyle, alignItems: "center" }}>
        <span style={{ fontSize: 13 }}>SMS nesta cidade:</span>
        <Tag tone={setting.enabled ? "ok" : undefined}>{setting.enabled ? "ligado" : "desligado"}</Tag>
        {!confirming && (
          <button type="button" style={buttonStyle} onClick={() => { setDone(null); setConfirming(true); }}>
            {turningOn ? "Ligar SMS" : "Desligar SMS"}
          </button>
        )}
      </div>
      <p style={noteStyle}>
        o SMS nunca leva o texto do aviso: só "você tem um aviso novo" e o link do wpda, entre 8h e 20h, para quem aceitou receber
      </p>
      {!setting.gateway_configured && <p role="note" style={warnStyle}>{GATEWAY_WARNING}</p>}
      {done && <p role="status" style={{ margin: 0, fontSize: 12.5 }}>{done}</p>}
      {confirming && (
        <SensitiveAction
          title={turningOn ? "Ligar o SMS das campanhas" : "Desligar o SMS das campanhas"}
          description={turningOn
            ? "Nas próximas campanhas, quem aceitou receber SMS ganha um SMS avisando que há um aviso novo no wpda."
            : "As próximas campanhas saem só como aviso no wpda. Campanha já enviada não muda."}
          requiresStepUp
          confirmLabel={turningOn ? "Ligar" : "Desligar"}
          run={async () => { result.current = await setSmsSetting(turningOn); }}
          onDone={() => {
            const next = result.current as SmsSetting;
            setConfirming(false);
            queryClient.setQueryData(SMS_SETTING_KEY, next);
            setDone(next.enabled ? "SMS ligado" : "SMS desligado");
          }}
          onCancel={() => setConfirming(false)}
          onGoToSecurity={onGoToSecurity}
          translateError={campaignErrorOrNull}
        />
      )}
    </div>
  );
}
```

- [ ] **Step 4: Rode, cheque tipos e faça commit**

Run: `cd apps/dashboard/.claude/mod12 && npx vitest run src/modules/campaigns/SmsSettingPanel.test.tsx && npx tsc --noEmit`
Expected: PASS; tsc sem erros.

```bash
cd apps/dashboard/.claude/mod12
/opt/homebrew/bin/git add src/modules/campaigns/SmsSettingPanel.tsx src/modules/campaigns/SmsSettingPanel.test.tsx
/opt/homebrew/bin/git commit -m "feat: let city admins toggle campaign SMS with step-up

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 11: Menu por papel, lista e raiz do módulo

**Files:**
- Modify: `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/App.tsx`
- Create: `src/modules/campaigns/CampaignList.tsx`, `src/modules/Campaigns.tsx`
- Test: `src/modules/Campaigns.test.tsx`

**Interfaces:**
- Consumes:
  - da Task 1: `listCampaigns`, `CampaignSummary`;
  - da Task 2: `CAMPAIGNS_KEY`, `CAMPAIGN_MANAGER_ROLE`, `STATUS_LABEL`, `STATUS_TONE`, `campaignError`;
  - das Tasks 7 a 10: `CampaignEditor`, `CampaignPanel`, `SmsSettingPanel`;
  - `useAuth`, que já existe.
- Produces:
  - `ModuleId` ganha `"campaigns"`;
  - um grupo novo "Comunicação" com o item `{ id: "campaigns", label: "Campanhas", icon: "✉" }`;
  - `sendLabel(c: CampaignSummary) → string`;
  - `CampaignList({ onNew(); onOpen(c: CampaignSummary) })`;
  - `Campaigns({ onNavigate(id: ModuleId) })`.

- [ ] **Step 1: Testes da navegação**

Em `src/shell/modules.test.ts`:

1. No teste "operador não vê o grupo Conta…", troque

```ts
      expect(groups.some((g) => g.label === "Cidade")).toBe(false);
      expect(groups.length).toBe(NAV_GROUPS.length - 4);
```

por

```ts
      expect(groups.some((g) => g.label === "Cidade")).toBe(false);
      expect(groups.some((g) => g.label === "Comunicação")).toBe(false);
      expect(groups.length).toBe(NAV_GROUPS.length - 5);
```

2. No teste "sem sessão, esconde Equipe, Atendimento e Cidade", troque

```ts
      expect(groups.some((g) => g.label === "Cidade")).toBe(false);
      expect(groups.length).toBe(NAV_GROUPS.length - 3);
```

por

```ts
      expect(groups.some((g) => g.label === "Cidade")).toBe(false);
      expect(groups.some((g) => g.label === "Comunicação")).toBe(false);
      expect(groups.length).toBe(NAV_GROUPS.length - 4);
```

3. Antes do `});` final do arquivo, acrescente:

```ts
  describe("módulo 12 na navegação", () => {
    const user = (roles: string[]) => ({ operator: false, memberships: roles.map((role) => ({ role })) });
    const ids = (u: Parameters<typeof navGroupsFor>[0]) => navGroupsFor(u).flatMap((g) => g.items.map((i) => i.id));

    it("Campanhas para campaign_manager e municipal_admin, e para ninguém mais", () => {
      expect(ids(user([ "campaign_manager" ]))).toContain("campaigns");
      expect(ids(user([ "municipal_admin" ]))).toContain("campaigns");
      for (const role of [ "viewer", "citizen_verifier", "health_professional", "protocol_reviewer", "protocol_publisher" ]) {
        expect(ids(user([ role ]))).not.toContain("campaigns");
      }
      expect(ids(null)).not.toContain("campaigns");
      expect(labelFor("campaigns")).toBe("Campanhas");
    });

    it("gestor de campanhas não ganha Equipe nem Território", () => {
      expect(ids(user([ "campaign_manager" ]))).not.toContain("team");
      expect(ids(user([ "campaign_manager" ]))).not.toContain("territory");
    });
  });
```

- [ ] **Step 2: Testes da raiz e da lista**

```tsx
// src/modules/Campaigns.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return {
    ...real, fetchCurrentSession: vi.fn(), stepUpMfa: vi.fn(), listCampaigns: vi.fn(), getSmsSetting: vi.fn(),
    getCampaignOptions: vi.fn(), getCampaign: vi.fn(), previewAudience: vi.fn()
  };
});

import * as api from "../lib/api";
import { Campaigns } from "./Campaigns";
import { sendLabel } from "./campaigns/CampaignList";
import { OPTIONS, campaign, renderWithProviders, sessionWith } from "../test/campaignFixtures";

afterEach(cleanup);
const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

const draft = campaign({ id: "c1", title: "Rascunho da gripe" });
const scheduled = campaign({ id: "c2", title: "Agendada", status: "scheduled", send_at: "2026-09-30T12:00:00.000Z" });
const sent = campaign({ id: "c3", title: "Enviada", status: "sent", dispatched_at: "2026-09-28T13:00:00.000Z",
  recipients_count: 1234, phones_count: 1000, sms_enabled: false,
  stats: { read_count: 10, sms: { not_opted_in: 1234, duplicate_phone: 0, pending: 0, deferred: 0, sent: 0, failed: 0, unavailable: 0 } } });

function renderAs(roles: string[]) {
  mocked(api.fetchCurrentSession).mockResolvedValue(sessionWith(roles));
  const onNavigate = vi.fn();
  renderWithProviders(<Campaigns onNavigate={onNavigate} />);
  return { onNavigate };
}

describe("Campaigns", () => {
  beforeEach(() => {
    vi.resetAllMocks();
    mocked(api.listCampaigns).mockResolvedValue([ draft, scheduled, sent ]);
    mocked(api.getSmsSetting).mockResolvedValue({ enabled: false, gateway_configured: false });
    mocked(api.getCampaignOptions).mockResolvedValue(OPTIONS);
    mocked(api.previewAudience).mockResolvedValue({ citizens: 12, phones: 9 });
  });

  it("gestor vê a lista com status, envio e destinatários, sem a chave de SMS", async () => {
    renderAs([ "campaign_manager" ]);
    const list = within(await screen.findByRole("region", { name: "Lista" }));
    expect(await list.findByText("Rascunho da gripe")).toBeTruthy();
    expect(list.getByText("rascunho")).toBeTruthy();
    expect(list.getByText(/agendada para 30\/09\/2026.*09:00/)).toBeTruthy();
    expect(list.getByText("1.234")).toBeTruthy();
    expect(screen.queryByRole("region", { name: "SMS das campanhas" })).toBeNull();
  });

  it("municipal_admin sem o papel: só a chave de SMS, e nunca pede a lista", async () => {
    renderAs([ "municipal_admin" ]);
    expect(await screen.findByRole("region", { name: "SMS das campanhas" })).toBeTruthy();
    expect(screen.getByText(/Criar e enviar campanhas é do papel gestor de campanhas/)).toBeTruthy();
    expect(screen.queryByRole("region", { name: "Lista" })).toBeNull();
    expect(api.listCampaigns).not.toHaveBeenCalled();
  });

  it("com os dois papéis: chave e lista", async () => {
    renderAs([ "municipal_admin", "campaign_manager" ]);
    expect(await screen.findByRole("region", { name: "SMS das campanhas" })).toBeTruthy();
    expect(await screen.findByRole("region", { name: "Lista" })).toBeTruthy();
  });

  it("'Nova campanha' abre o editor, e 'Voltar à lista' volta", async () => {
    renderAs([ "campaign_manager" ]);
    fireEvent.click(await screen.findByRole("button", { name: "Nova campanha" }));
    expect(await screen.findByRole("region", { name: "Nova campanha" })).toBeTruthy();
    fireEvent.click(screen.getByRole("button", { name: "Voltar à lista" }));
    expect(await screen.findByRole("region", { name: "Lista" })).toBeTruthy();
  });

  it("rascunho abre no editor; enviada abre no painel", async () => {
    mocked(api.getCampaign).mockImplementation(async (id: string) => (id === "c1" ? draft : sent));
    renderAs([ "campaign_manager" ]);
    fireEvent.click(await screen.findByRole("button", { name: "Abrir Rascunho da gripe" }));
    expect(((await screen.findByLabelText("Título")) as HTMLInputElement).value).toBe("Rascunho da gripe");

    fireEvent.click(screen.getByRole("button", { name: "Voltar à lista" }));
    fireEvent.click(await screen.findByRole("button", { name: "Abrir Enviada" }));
    expect(await screen.findByRole("region", { name: "Resultado" })).toBeTruthy();
  });

  it("sendLabel", () => {
    expect(sendLabel(draft)).toBe("—");
    expect(sendLabel(sent)).toMatch(/28\/09\/2026.*10:00/);
  });
});
```

- [ ] **Step 3: Rode e veja falhar**

Run: `cd apps/dashboard/.claude/mod12 && npx vitest run src/shell/modules.test.ts src/modules/Campaigns.test.tsx`
Expected: FAIL. `./Campaigns` não resolve, e a navegação não tem "campaigns".

- [ ] **Step 4: Navegação**

Em `src/shell/modules.ts`:

1. Troque `| "professionals" | "my-profile" | "territory";` por `| "professionals" | "my-profile" | "territory" | "campaigns";`.
2. Em `NAV_GROUPS`, depois do grupo "Atendimento", acrescente:

```ts
  { label: "Comunicação", items: [
    { id: "campaigns", label: "Campanhas", icon: "✉" }
  ]},
```

3. Em `navGroupsFor`, depois de `const isProfessional = …`, acrescente:

```ts
  // Módulo 12: /campaigns é do campaign_manager; o municipal_admin sem o
  // papel entra para a chave de SMS da cidade (spec 2026-09-29 §7).
  const canCampaigns = isAdmin || roles.includes("campaign_manager");
```

   e, no `filter`, depois da linha de "Cidade":

```ts
    if (group.label === "Comunicação") return canCampaigns;
```

- [ ] **Step 5: Lista**

```tsx
// src/modules/campaigns/CampaignList.tsx
import { useQuery } from "@tanstack/react-query";
import { listCampaigns, type CampaignSummary } from "../../lib/api";
import { CAMPAIGNS_KEY, STATUS_LABEL, STATUS_TONE, campaignError } from "../../lib/campaigns";
import { fmtDateTime, fmtNumber } from "../../lib/format";
import { Panel } from "../../components/Panel";
import { DataTable } from "../../components/DataTable";
import { Tag } from "../../components/Tag";
import { buttonStyle, secondaryButtonStyle } from "../../components/formStyles";
import { alertStyle, noteStyle } from "./styles";

export function sendLabel(c: CampaignSummary): string {
  if (c.status === "scheduled" && c.send_at) return `agendada para ${fmtDateTime(c.send_at)}`;
  if (c.dispatched_at) return fmtDateTime(c.dispatched_at);
  return "—";
}

export function CampaignList({ onNew, onOpen }: { onNew(): void; onOpen(c: CampaignSummary): void }) {
  const list = useQuery({ queryKey: CAMPAIGNS_KEY, queryFn: listCampaigns });
  return (
    <Panel title="Lista" sub="mais novas primeiro"
      right={<button type="button" style={buttonStyle} onClick={onNew}>Nova campanha</button>}>
      {list.isError ? <p role="alert" style={alertStyle}>{campaignError(list.error)}</p>
        : list.isPending ? <p className="mono" style={noteStyle}>carregando…</p>
        : (
          <DataTable<CampaignSummary>
            cols={[
              { label: "Título", w: "3fr", render: (c) => (
                <button type="button" aria-label={`Abrir ${c.title}`} onClick={() => onOpen(c)}
                  style={{ ...secondaryButtonStyle, border: "none", textAlign: "left" }}>
                  {c.title}
                </button>
              ) },
              { label: "Status", w: "1fr", render: (c) => <Tag tone={STATUS_TONE[c.status]}>{STATUS_LABEL[c.status]}</Tag> },
              { label: "Envio", w: "2fr", render: (c) => sendLabel(c) },
              { label: "Destinatários", w: "1fr", align: "right", render: (c) => <span className="mono">{fmtNumber(c.recipients_count)}</span> }
            ]}
            rows={list.data}
            rowKey={(c) => c.id}
            empty="nenhuma campanha ainda — crie a primeira"
          />
        )}
    </Panel>
  );
}
```

- [ ] **Step 6: Raiz e App**

```tsx
// src/modules/Campaigns.tsx
import { useState, type ReactNode } from "react";
import { useAuth } from "../lib/auth";
import { CAMPAIGN_MANAGER_ROLE } from "../lib/campaigns";
import { PageHeader } from "../components/PageHeader";
import { EmptyState } from "../components/EmptyState";
import type { ModuleId } from "../shell/modules";
import { CampaignList } from "./campaigns/CampaignList";
import { CampaignEditor } from "./campaigns/CampaignEditor";
import { CampaignPanel } from "./campaigns/CampaignPanel";
import { SmsSettingPanel } from "./campaigns/SmsSettingPanel";

// Campanhas (módulo 12; ADR 0024; spec 2026-09-29 §7). O campaign_manager
// lista, edita, envia e acompanha; o municipal_admin liga o SMS da cidade.
// Quem é só municipal_admin nunca chama GET /campaigns (a API responderia
// 403). As vistas trocam por estado, sem URL, como em Profissionais.
type View = { kind: "list" } | { kind: "edit"; id: string | null } | { kind: "view"; id: string };

export function Campaigns({ onNavigate }: { onNavigate(id: ModuleId): void }) {
  const { user } = useAuth();
  const roles = user?.memberships?.map((m) => m.role) ?? [];
  const isManager = roles.includes(CAMPAIGN_MANAGER_ROLE);
  const isAdmin = roles.includes("municipal_admin");
  const [ view, setView ] = useState<View>({ kind: "list" });
  const toList = () => setView({ kind: "list" });
  const goToSecurity = () => onNavigate("security");

  let content: ReactNode;
  if (isManager && view.kind === "edit") {
    content = (
      <CampaignEditor key={view.id ?? "new"} campaignId={view.id} onBack={toList}
        onLeftDraft={(c) => setView({ kind: "view", id: c.id })} onGoToSecurity={goToSecurity} />
    );
  } else if (isManager && view.kind === "view") {
    content = (
      <CampaignPanel key={view.id} campaignId={view.id} onBack={toList}
        onEdit={(id) => setView({ kind: "edit", id })} onGoToSecurity={goToSecurity} />
    );
  } else {
    content = (
      <>
        {isAdmin && <SmsSettingPanel onGoToSecurity={goToSecurity} />}
        {isManager ? (
          <CampaignList
            onNew={() => setView({ kind: "edit", id: null })}
            onOpen={(c) => setView(c.status === "draft" ? { kind: "edit", id: c.id } : { kind: "view", id: c.id })}
          />
        ) : isAdmin ? (
          <p style={{ margin: 0, fontSize: 12.5, color: "var(--ink2)" }}>
            Criar e enviar campanhas é do papel gestor de campanhas (conceda em Equipe). Aqui você liga ou desliga o SMS da cidade.
          </p>
        ) : (
          <EmptyState title="seu papel não dá acesso a campanhas" />
        )}
      </>
    );
  }

  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 16 }}>
      <PageHeader title="Campanhas" sub="avisos no wpda · sms opcional" />
      {content}
    </div>
  );
}
```

Em `src/App.tsx`, acrescente `import { Campaigns } from "./modules/Campaigns";` depois do import de `Territory`, e, no `switch`, depois de `case "territory":`:

```tsx
    case "campaigns":      return <Campaigns onNavigate={setActive} />;
```

- [ ] **Step 7: Rode, cheque tipos e faça commit**

Run: `cd apps/dashboard/.claude/mod12 && npx vitest run src/shell/modules.test.ts src/modules/Campaigns.test.tsx src/modules/campaigns && npx tsc --noEmit`
Expected: PASS; tsc sem erros.

```bash
cd apps/dashboard/.claude/mod12
/opt/homebrew/bin/git add src/shell/modules.ts src/shell/modules.test.ts src/App.tsx src/modules/Campaigns.tsx src/modules/Campaigns.test.tsx src/modules/campaigns/CampaignList.tsx
/opt/homebrew/bin/git commit -m "feat: add campaigns module to the dashboard navigation by role

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 12: Papel `campaign_manager` em Equipe

**Files:**
- Modify: `src/lib/team.ts`, `src/lib/team.test.ts`, `src/modules/Team.tsx`, `src/modules/Team.test.tsx`

**Interfaces:**
- Consumes: `CAMPAIGN_MANAGER_ROLE`, da Task 2.
- Produces:
  - `TeamMember` ganha `isCampaignManager: boolean` e `campaignManagerMembershipId: string | null`;
  - `INVITE_ROLES` passa a 8 papéis;
  - `isPrivilegedRole("campaign_manager") === true`;
  - em Equipe, a coluna "Campanhas" com os botões "Tornar gestor de campanhas" e "Remover gestor de campanhas".

- [ ] **Step 1: Testes da lib**

Em `src/lib/team.test.ts`, troque o teste "oferece os 7 papéis da cidade" e o de privilegiados por:

```ts
  it("oferece os 8 papéis da cidade", () => {
    expect(INVITE_ROLES.map((r) => r.role).sort()).toEqual([
      "campaign_manager", "citizen_verifier", "health_professional", "municipal_admin", "protocol_author",
      "protocol_publisher", "protocol_reviewer", "viewer"
    ]);
    expect(INVITE_ROLES.find((r) => r.role === "campaign_manager")?.label).toBe("Gestor de campanhas");
  });

  it("papéis privilegiados são os mesmos que a API protege com step-up", () => {
    for (const role of [ "municipal_admin", "protocol_reviewer", "citizen_verifier", "health_professional", "campaign_manager" ]) {
      expect(isPrivilegedRole(role)).toBe(true);
    }
    for (const role of [ "viewer", "protocol_author", "protocol_publisher" ]) {
      expect(isPrivilegedRole(role)).toBe(false);
    }
  });
```

e acrescente ao fim do arquivo:

```ts
describe("gestor de campanhas", () => {
  it("teamMembers marca o papel e guarda o id da membership", () => {
    const [ ana ] = teamMembers([
      { id: "m-1", user: { id: "u1", email_address: "ana@cidade.gov.br" }, role: "viewer", granted_at: "x" },
      { id: "m-2", user: { id: "u1", email_address: "ana@cidade.gov.br" }, role: "campaign_manager", granted_at: "x" }
    ]);
    expect(ana.isCampaignManager).toBe(true);
    expect(ana.campaignManagerMembershipId).toBe("m-2");
  });
});
```

(Confira que `teamMembers` já está no import do topo de `team.test.ts`; se não estiver, acrescente-o.)

- [ ] **Step 2: Testes da tela**

Em `src/modules/Team.test.tsx`, antes do `});` que fecha `describe("Team")`, acrescente:

```tsx
  describe("gestor de campanhas (módulo 12)", () => {
    it("tornar e remover gestor de campanhas, com step-up", async () => {
      mocked(api.listMemberships).mockResolvedValue([
        membership("ana@cidade.gov.br", "viewer"),
        membership("bia@cidade.gov.br", "campaign_manager", "m-bia-camp")
      ]);
      mocked(api.grantRole).mockResolvedValue(undefined);
      mocked(api.revokeMembership).mockResolvedValue(undefined);
      renderTeam();

      fireEvent.click(await screen.findByRole("button", { name: "Tornar gestor de campanhas" }));
      fireEvent.click(await screen.findByRole("button", { name: "Confirmar" }));
      await waitFor(() => expect(api.grantRole).toHaveBeenCalledWith("u-ana@cidade.gov.br", "campaign_manager"));
      expect((await screen.findByRole("status")).textContent).toBe("ana@cidade.gov.br agora é gestor de campanhas");

      fireEvent.click(await screen.findByRole("button", { name: "Remover gestor de campanhas" }));
      fireEvent.click(await screen.findByRole("button", { name: "Confirmar" }));
      await waitFor(() => expect(api.revokeMembership).toHaveBeenCalledWith("m-bia-camp"));
      expect((await screen.findByRole("status")).textContent).toBe("bia@cidade.gov.br não é mais gestor de campanhas");
    });

    it("janela fechada: conceder pede o código do autenticador", async () => {
      mocked(api.fetchCurrentSession).mockResolvedValue(session({ mfa_verified_at: null }));
      mocked(api.listMemberships).mockResolvedValue([ membership("ana@cidade.gov.br", "viewer") ]);
      mocked(api.stepUpMfa).mockResolvedValue(undefined);
      mocked(api.grantRole).mockResolvedValue(undefined);
      renderTeam();

      fireEvent.click(await screen.findByRole("button", { name: "Tornar gestor de campanhas" }));
      fireEvent.change(await screen.findByLabelText("Código do autenticador"), { target: { value: "112233" } });
      fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));
      await waitFor(() => expect(api.grantRole).toHaveBeenCalledWith("u-ana@cidade.gov.br", "campaign_manager"));
      expect(api.stepUpMfa).toHaveBeenCalledWith("112233");
    });

    it("convite como gestor de campanhas é privilegiado: pede o código", async () => {
      mocked(api.fetchCurrentSession).mockResolvedValue(session({ mfa_verified_at: null }));
      mocked(api.listMemberships).mockResolvedValue([]);
      renderTeam();

      fireEvent.change(await screen.findByLabelText("E-mail da pessoa"), { target: { value: "com@cidade.gov.br" } });
      fireEvent.change(screen.getByLabelText("Papel"), { target: { value: "campaign_manager" } });
      fireEvent.click(screen.getByRole("button", { name: "Convidar" }));
      expect(await screen.findByLabelText("Código do autenticador")).toBeTruthy();
      expect(screen.getByText("Convidar com@cidade.gov.br como Gestor de campanhas.")).toBeTruthy();
    });
  });
```

- [ ] **Step 3: Rode e veja falhar**

Run: `cd apps/dashboard/.claude/mod12 && npx vitest run src/lib/team.test.ts src/modules/Team.test.tsx`
Expected: FAIL. `campaign_manager` não está em `INVITE_ROLES`, e o botão "Tornar gestor de campanhas" não existe.

- [ ] **Step 4: `team.ts`**

1. Troque `import { ApiError, type MembershipRow } from "./api";` por:

```ts
import { ApiError, type MembershipRow } from "./api";
import { CAMPAIGN_MANAGER_ROLE } from "./campaigns";
```

2. Em `TeamMember`, depois de `professionalStatus: …;`, acrescente:

```ts
  isCampaignManager: boolean;
  campaignManagerMembershipId: string | null;
```

3. No objeto inicial de `teamMembers`, troque `isProfessional: false, professionalMembershipId: null, professionalStatus: null` por `isProfessional: false, professionalMembershipId: null, professionalStatus: null, isCampaignManager: false, campaignManagerMembershipId: null`.
4. Depois do `if (row.role === PROFESSIONAL_ROLE) { … }`, acrescente:

```ts
    if (row.role === CAMPAIGN_MANAGER_ROLE) {
      current.isCampaignManager = true;
      current.campaignManagerMembershipId = row.id;
    }
```

5. Troque o comentário `// Os 7 papéis da cidade (Membership::ROLES na API), na ordem do seletor do` por `// Os 8 papéis da cidade (Membership::ROLES na API), na ordem do seletor do`. Depois, em `INVITE_ROLES`, antes da linha de `municipal_admin`, acrescente:

```ts
  { role: CAMPAIGN_MANAGER_ROLE, label: "Gestor de campanhas" },
```

6. Troque a linha de `PRIVILEGED_ROLES` por:

```ts
const PRIVILEGED_ROLES = new Set([ "municipal_admin", REVIEWER_ROLE, VERIFIER_ROLE, PROFESSIONAL_ROLE, CAMPAIGN_MANAGER_ROLE ]);
```

- [ ] **Step 5: `Team.tsx`**

1. Acrescente o import `import { CAMPAIGN_MANAGER_ROLE } from "../lib/campaigns";`.
2. Troque `type Pending = { member: TeamMember; kind: "grant" | "revoke"; role: "reviewer" | "verifier" | "professional" };` por:

```ts
type Pending = { member: TeamMember; kind: "grant" | "revoke"; role: "reviewer" | "verifier" | "professional" | "campaign" };
```

3. Troque `<PageHeader title="Equipe" sub="papéis · revisores de protocolo · atendentes" />` por `<PageHeader title="Equipe" sub="papéis · revisores de protocolo · atendentes · campanhas" />`.
4. Nas `cols` do `DataTable`, depois da coluna "Profissional de saúde", acrescente:

```tsx
                  { label: "Campanhas", w: "auto", align: "right", render: (m) => (
                    m.isCampaignManager
                      ? <button type="button" style={buttonStyle} onClick={() => open(m, "revoke", "campaign")}>Remover gestor de campanhas</button>
                      : <button type="button" style={buttonStyle} onClick={() => open(m, "grant", "campaign")}>Tornar gestor de campanhas</button>
                  ) },
```

5. Depois do bloco `{pending && pending.role === "professional" && ( … )}`, acrescente:

```tsx
      {pending && pending.role === "campaign" && (
        pending.kind === "grant" ? (
          <SensitiveAction
            title="Tornar gestor de campanhas"
            description={`${pending.member.email} poderá montar públicos com dado de saúde e enviar avisos da Secretaria aos cidadãos.`}
            requiresStepUp
            run={async () => { await grantRole(pending.member.userId, CAMPAIGN_MANAGER_ROLE); }}
            onDone={() => finish(`${pending.member.email} agora é gestor de campanhas`)}
            onCancel={() => setPending(null)}
            onGoToSecurity={() => onNavigate("security")}
          />
        ) : (
          // O `!` é seguro: "Remover gestor de campanhas" só existe quando
          // isCampaignManager é true, e teamMembers preenche os dois juntos.
          <SensitiveAction
            title="Remover gestor de campanhas"
            description={`${pending.member.email} deixa de criar e enviar campanhas. As campanhas já enviadas continuam valendo.`}
            requiresStepUp
            run={async () => { await revokeMembership(pending.member.campaignManagerMembershipId!); }}
            onDone={() => finish(`${pending.member.email} não é mais gestor de campanhas`)}
            onCancel={() => setPending(null)}
            onGoToSecurity={() => onNavigate("security")}
          />
        )
      )}
```

6. No comentário do topo de `Team.tsx`, troque `// Escopo: conceder e revogar revisor, atendente e profissional de saúde;` por `// Escopo: conceder e revogar revisor, atendente, profissional de saúde e gestor de campanhas;` e `// convidar pessoa com qualquer um dos 7 papéis` por `// convidar pessoa com qualquer um dos 8 papéis`.

- [ ] **Step 6: Rode, cheque tipos e faça commit**

Run: `cd apps/dashboard/.claude/mod12 && npx vitest run src/lib/team.test.ts src/modules/Team.test.tsx && npx tsc --noEmit`
Expected: PASS (os testes antigos de Equipe continuam verdes); tsc sem erros.

```bash
cd apps/dashboard/.claude/mod12
/opt/homebrew/bin/git add src/lib/team.ts src/lib/team.test.ts src/modules/Team.tsx src/modules/Team.test.tsx
/opt/homebrew/bin/git commit -m "feat: grant and invite the campaign manager role in the team screen

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 13: Suíte, build, revisão e prova no navegador

- [ ] **Step 1: O console `admin` não compartilha tipos com o dashboard**

Run (da raiz do monorepo):

```bash
grep -rn "dashboard/src\|@rota-saude/dashboard" apps/admin/src apps/admin/package.json apps/admin/tsconfig.json
```

Expected: nenhuma linha. O `admin` não muda neste módulo (spec, "Afeta").

- [ ] **Step 2: Suíte, tipos e build**

Run: `cd apps/dashboard/.claude/mod12 && npx vitest run && npx tsc --noEmit && npm run build`
Expected: tudo verde. A CI (`.github/workflows/ci.yml`) roda os mesmos três. `dist/` está no `.gitignore`.

```bash
/opt/homebrew/bin/git status --short
```

Expected: vazio. O symlink `node_modules` não aparece porque é ignorado. Se aparecer, **não** o adicione.

- [ ] **Step 3: Revisão**

Um subagente revisor lê `origin/main..HEAD` do dashboard contra a spec (§6.1, §6.3 e §7), o ADR 0024 e o arquivo de contratos. Pontos de atenção:
- nenhuma tela mostra lista de destinatários, CPF ou telefone, só contagens;
- o JSON do público nunca leva chave de cartão nem opcional vazio (`""`, `null`, `[]`);
- o envio só é liberado por contagem `ok` do público **atual**. Loading, erro, `below_minimum` e debounce pendente travam;
- `send`, `unschedule` e `cancel` mandam `"{}"` com `Content-Type: application/json`;
- send, schedule, unschedule, cancel, a chave de SMS e o papel em Equipe passam todos pelo `SensitiveAction` com `requiresStepUp`;
- o `municipal_admin` sem o papel nunca dispara `GET /campaigns`;
- "hoje" e o `send_at` saem no fuso da cidade, nunca no do navegador;
- nenhum `git add -A` no histórico (`git log --stat origin/main..HEAD` sem `node_modules`).

- [ ] **Step 4: Prova no navegador (com o usuário)**

Rode o api da branch do módulo 12 a partir do worktree dele, na porta que o plano do api definir (no módulo 11 foi `:3031`), com a semente de campanhas aplicada em Curitiba. Depois rode o Vite do worktree apontando para ele:

```bash
cd apps/dashboard/.claude/mod12 && VITE_API_PROXY_TARGET=http://localhost:3031 npx vite --port 5179 --host 0.0.0.0
```

Abra `http://curitiba.localhost:5179/dashboard/`. O usuário faz o login. Não digite senha nem TOTP. Confira com screenshot:
- como `campanhas@curitiba.demo`:
  - Comunicação → Campanhas → Nova campanha;
  - Bairros (dois da semente) + "Falta em agendamento". O contador muda meio segundo depois de cada ajuste;
  - um recorte pequeno mostra "menos de 5 — ajuste o público" e o botão travado;
  - salvar, "Revisar e enviar…", a frase do público, o step-up e o envio;
  - o painel mostra os agregados;
- com `config.x.sms_gateway = nil` e a chave ligada: o alerta "SMS não enviado: a plataforma ainda não tem provedor de SMS";
- como `admin@curitiba.demo`:
  - Campanhas mostra só a chave de SMS e o aviso de provedor. Ligar pede step-up;
  - em Equipe, a coluna "Campanhas" e o convite com "Gestor de campanhas".

- [ ] **Step 5:** **Pare.** O merge do dashboard só vem depois do merge do api, e só com autorização explícita do usuário. A ordem de deploy é api → dashboard → wpda (spec §11).

---

## Self-review

- **Cobertura da spec:**
  - §6.1, as 12 rotas: cliente na Task 1. O uso de cada rota:
    - lista: Task 11;
    - options: Tasks 7 e 9;
    - preview: Task 6;
    - create/patch: Task 7;
    - get: Tasks 7 e 9;
    - send/schedule: Task 8;
    - unschedule/cancel: Task 9;
    - sms_setting GET: Tasks 7 e 10;
    - sms_setting PUT: Task 10.
  - §6.3, agregados: Task 9. Os números vêm de `recipients_count`, `phones_count`, `read_count` e `%`, o SMS por status de `stats.sms`, e `sms_enabled` e `audience` também entram.
  - §7:
    - menu por papel: Task 11;
    - lista: Task 11;
    - editor (título, texto, pré-visualização): Task 7;
    - recorte: Task 5;
    - critérios em cartões com "e também": Tasks 4 e 5;
    - contador com debounce e "menos de 5": Task 6, travando o envio na Task 8;
    - linha do SMS: Task 7;
    - modal com frase, contagem, canais, agora/agendar, step-up e retorno do `below_minimum`: Tasks 3 e 8;
    - painel com alertas `unavailable`/`failed`, frase e motivo de `failed`: Task 9;
    - chave de SMS com step-up e aviso de gateway: Task 10;
    - Equipe: Task 12.
  - §9.3:
    - adicionar e remover cartão: Task 5;
    - validação por tipo: Tasks 2 e 4;
    - contador, "menos de 5" e debounce: Tasks 6 e 8;
    - frase: Task 3;
    - modal com step-up: Task 8;
    - painel `unavailable`: Task 9;
    - chave com aviso: Task 10;
    - papel em Equipe: Task 12.
  - §9.5, prova no navegador: Task 13.
- **Placeholders:** nenhum. As Tasks 8, 9 e 12 alteram arquivos existentes por trechos exatos (de/para), com o código inteiro do trecho novo.
- **Consistência de nomes:**
  - `CAMPAIGNS_KEY`, `campaignKey`, `PREVIEW_KEY`, `SMS_SETTING_KEY` e `CAMPAIGN_OPTIONS_KEY` estão definidos na Task 2 e são usados nas Tasks 6 a 11.
  - `CAMPAIGN_MANAGER_ROLE` está definido na Task 2 e é usado nas Tasks 11 e 12.
  - `AudienceDraft`, `buildAudience`, `audienceProblem` e `draftFromAudience` saem da Task 2 e são usados nas Tasks 5 e 7.
  - `PreviewState`, `previewAllowsSend`, `useAudiencePreview` e `AudienceCounter` saem da Task 6 e são usados nas Tasks 7 e 8.
  - `CampaignEditorProps` ganha `onLeftDraft` e `onGoToSecurity` na Task 8, e a Task 11 os passa.
  - `CampaignLifecycleDialog` recebe `Pick<Campaign, "id" | "title">` na Task 9, tanto no editor quanto no painel.
  - `describeAudience(audience, lookup)` recebe `CampaignOptions`, que é compatível com `PhraseLookup`, nas Tasks 8 e 9.
- **Review Focus:** cada um dos cinco itens tem teste na task dona:
  - 1: Tasks 6 e 8;
  - 2: Task 2;
  - 3: Tasks 2 e 3;
  - 4: Tasks 3 e 5;
  - 5: Task 8.
