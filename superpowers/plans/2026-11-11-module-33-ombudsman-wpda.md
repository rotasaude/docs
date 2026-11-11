# Módulo 33 — Ouvidoria do SUS municipal (wpda) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do 33** na branch `feat/mod-33-ombudsman` (worktree `apps/api/.claude/mod33`, porta **3056**) com as rotas do contrato §3 e §4, a 4ª fonte da caixa de avisos (§3, "Avisos") e `ombudsman`/`ombudsman_anonymous_web` em `features` da sessão do cidadão (§1), na forma das Divergências W1–W14. Ordem (contrato §15): api → dashboard → **wpda** → maintenance. As Tasks 0–9 não dependem do api rodando (dublês de `fetch`, `vi.spyOn(citizenApi, …)` e Playwright com rotas respondidas); a prova da Task 10 depende.

**Goal:** Lado wpda de F-33.2, F-33.3 e F-33.8:
- **tela inicial** com a entrada "Ouvidoria": "Fazer uma manifestação" (entra com CPF e código, se ainda não entrou) e "Denúncia anônima" (sem login; só com `anonymous_web`);
- **denúncia anônima** sem login, com o aviso "Sem identificação, você não recebe protocolo nem acompanha a resposta." antes do envio, sem protocolo e sem cookie;
- **nova manifestação** identificada ou com identidade preservada, dentro da máquina de telas do `Flow`: escolha da pessoa (se o celular tem mais de uma), tipo, modo, unidade, texto e até 3 anexos (base64);
- **"Minhas manifestações"**: lista e detalhe, com complemento, pedido de reabertura e pesquisa de satisfação;
- **aviso de ouvidoria** na caixa de avisos (`NoticesStep`), com rótulo próprio e sem texto.

**Architecture:**
- **Cliente** (`src/lib/citizenApi.ts`): bloco "Ouvidoria (módulo 33)" com os tipos, as normalizações e nove métodos. As duas rotas públicas (`public_config`, `anonymous_reports`) passam por uma função nova `publicCall`, com `credentials: "omit"` e **sem** disparar `citizen:unauthenticated`. Assim a denúncia nunca leva o cookie da sessão, mesmo com o cidadão logado (W7). `normalizeNotice` ganha o tipo `ombudsman`.
- **Regras puras** em `src/lib/ombudsman.ts`: rótulos, frases de prazo, contagem do texto, conferência de anexo, leitura base64 e mensagens de erro próprias do módulo (o `feature_disabled` de `ui.tsx` fala do histórico).
- **Peças novas**, no molde de `MyTripsStep` (leitura num `useEffect` com `alive`, **sem polling**) e de `PeopleStep` (modos internos, guarda síncrona `inFlight`):
  - `OmbudsmanFields.tsx` (`TextArea`, `UnitSelect`, `AttachmentPicker`);
  - `OmbudsmanEntry.tsx` (bloco da tela inicial);
  - `AnonymousReportStep.tsx`;
  - `NewManifestationStep.tsx`;
  - `MyManifestationsStep.tsx`;
  - `ManifestationDetailStep.tsx`.
- **Mudam o mínimo:**
  - `Flow.tsx`: quatro estados novos, a rota `ouvidoria` e a regra do topo;
  - `route.ts`: a rota `ouvidoria`;
  - `PhoneStep.tsx`: `purpose` e `children`;
  - `PeopleStep.tsx`: botão "Ouvidoria";
  - `NoticesStep.tsx`: item e detalhe do aviso `ombudsman`;
  - `README.md`.
- **`sessionFeatures.ts` não muda:** o `Flow` lê `hasSessionFeature("ombudsman")` e `hasSessionFeature("ombudsman_anonymous_web")` pelas constantes de `lib/ombudsman.ts`.

**Tech Stack:** React 18 + TypeScript, Vitest 2 + Testing Library 16 + user-event 14 + jest-dom (jsdom, `src/test/setup.ts` com `cleanup` e `stubDevice`), `vi.stubGlobal("fetch")` e `vi.spyOn(citizenApi, …)`, Playwright (5 perfis, `e2e/`), Vite 5 (base `/wpda/`).

**Spec:** `docs/superpowers/specs/2026-11-11-module-33-ombudsman-design.md`, nestas partes:
- §1 (antiobjetivos: texto, nome ou CPF em URL, log ou evento; manifestação recusada por falta de identificação);
- §2, decisões 2, 3, 5, 6, 12, 16, 17 e 19;
- §4 "wpda", §5 (formulário público; exclusão LGPD), §6 "wpda", §8 F-33.2, F-33.3 e F-33.8.

Também `docs/adr/0051.md` e o contrato `docs/superpowers/plans/2026-11-11-module-33-ombudsman-contracts.md` §1, §3, §4, §9, §10, §14 e §15 (diferenças em W1–W14). Herdados: ADR 0017 (par CPF + celular), ADR 0024 (caixa de avisos, que não depende do termo), ciclo de interface (decisão 9: fluxo só no celular; wpda#2: desktop só em dev e staging), lição do polling (`ResultStep`, 82dddf1) e o plano do wpda do 30 (molde do e2e e da prova).

## Global Constraints

- **Rotas** (contrato §3 e §4, base `/citizen/ombudsman`). Toda escrita é JSON:
  - com sessão: `GET manifestations`, `GET manifestations/:id`, `POST manifestations`, `POST manifestations/:id/complement`, `POST manifestations/:id/reopen_request`, `POST manifestations/:id/satisfaction`, `POST attachments`;
  - sem sessão (`publicCall`, `credentials: "omit"`): `GET public_config`, `POST anonymous_reports`.
- **Denúncia anônima** (contrato §4; spec decisão 6):
  - o corpo é **exatamente** `{ body }` ou `{ body, health_unit_id }`. Nenhuma outra chave, nem `kind`;
  - nenhum cookie vai junto (`credentials: "omit"`; provado no e2e);
  - a tela nunca mostra protocolo, nem que o api mande um (defesa);
  - o aviso `ANONYMOUS_WARNING` ("Sem identificação, você não recebe protocolo nem acompanha a resposta.") fica à vista **antes** do botão de envio;
  - não tem anexo nem contato.
- **Nova manifestação** (contrato §3; spec decisão 5): só os modos `identified` e `preserved`; o `citizen_id` vai no corpo e é sempre uma pessoa de `citizenApi.people()`. No modo `preserved`, a dica pede para não escrever nome, CPF, telefone nem endereço no texto (o `body` é visível à equipe em qualquer modo, contrato §8).
- **Texto:**
  - `body` de 20 a 4.000 caracteres, e motivo de reabertura de 10 a 2.000;
  - contados em code points depois do `trim` (`[...s.trim()].length`, que bate com o `String#length` do Ruby, W9);
  - vai o texto aparado.
- **Anexos** (contrato §3):
  - PDF, JPEG ou PNG (`file.type`), até 10 MB, até 3 por manifestação e por complemento, conferidos no cliente **antes** do envio. Os bytes mágicos são do api;
  - o nome enviado é sempre genérico (`anexo-1.pdf`, `anexo-2.jpg`), nunca o nome do arquivo do aparelho (W8);
  - remover um anexo só o tira da lista: o órfão some no api em 24 h.
- **O cidadão nunca vê** o encaminhamento, o extrato, o assunto interno nem quem respondeu. O cliente descarta todo campo fora do contrato. `forwarded` aparece como "Em análise" (W5).
- **Prazo:**
  - `due_on` é data de calendário: `fmtCalendarDate`, nunca `new Date`;
  - `paused` (ou `awaiting_complement`) diz que o prazo está suspenso;
  - `red`, ou hoje (`cityToday()`) depois de `due_on`, diz que o prazo venceu;
  - estado final, sem prazo.
- **Na URL**, só o id (UUID) da manifestação, no caminho. Nada em query string, `localStorage`, `sessionStorage` ou `console`. O texto digitado mora só no estado do React: um F5 ou a sessão vencida o perdem, de propósito.
- **Sem polling.** Cada tela lê ao abrir e relê depois de uma ação. Toda resposta que chega depois do desmonte é ignorada (`alive`). O único trabalho assíncrono novo é a leitura do arquivo (`FileReader`) e o upload, e os dois respeitam `alive`. Nenhum `setTimeout` novo; se um aparecer na revisão, precisa de `clearTimeout` no desmonte (lição do 82dddf1).
- **Interruptores:**
  - sem `ombudsman` (sessão) ou com `public_config` em 404, nenhuma entrada de ouvidoria aparece. O resto do canal fica como hoje;
  - sem `anonymous_web`/`ombudsman_anonymous_web`, não aparece "Denúncia anônima".
- **Termo:** a ouvidoria **não** depende do termo da triagem (base legal: obrigação legal, spec §5), como a caixa de avisos (ADR 0024). Estados com `consentVersion: string | null` (W1).
- **Interface do canal** (subprojetos 4 e 6e):
  - `Screen`, `BigButton`, `ErrorText`, `RadioGroup`, `Field` e `ConfirmInline` de `ui.tsx`. Nenhum componente comum muda; as peças novas ficam em `OmbudsmanFields.tsx`;
  - alvos de 48 px, texto de 18 px, uma pergunta por tela;
  - cores só por papel do tema (`var(--warn-bg)`, `var(--accent-bg)`, `var(--rule2)`, `var(--ink2)`), checadas por `src/theme/guards.test.ts`;
  - fluxo só no celular (`isPhone`), com o desktop liberado só em dev e staging (`desktopAllowed`, `WPDA_ALLOW_DESKTOP`). A denúncia anônima mora no `Flow` e segue o mesmo bloqueio (W13). `e2e/device-gate.spec.ts` continua verde.
- **Relógio:** os testes com "agora" fixam o relógio (`vi.useFakeTimers({ toFake: [ "Date" ] })`, `2026-11-12T10:00:00-03:00`); a suíte roda em `TZ=America/Sao_Paulo`; o Playwright em `timezoneId: "UTC"`.
- **Git e board:**
  - nunca `git add -A` nem `git add .`;
  - commits em inglês, Conventional Commits com o tipo por extenso, `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`;
  - git `/opt/homebrew/bin/git`. Merge, push e tag só com autorização;
  - board: a primeira ação é mover F-33.1..F-33.9 do board #1 para In Progress (Task 0). O wpda é dono de F-33.2, F-33.3 e F-33.8 com o api. Cada card vai para Done quando o último dos dois terminar, e para Verified depois da prova (Task 10).

## Ambiente de execução

- `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `W=$R/.worktrees/wpda-mod33`, `G=/opt/homebrew/bin/git`. O worktree fica **fora de `apps/`**, na branch `feat/mod-33-ombudsman`. Os caminhos são relativos a `$W`.
- Testes: `bin/test wpda -w wpda-mod33 <arquivos>`. Tipos: `cd $W && npx tsc --noEmit`. E2e: `cd $W && WPDA_URL=http://curitiba.localhost:5217 npx playwright test <arquivos>`, com o Vite da Task 9 no ar.
- O api do 33 roda em dev na **3056**, no container `api-dev` (rede `rota-saude_default`, apelido `api`). O Vite do worktree sobe num container dessa rede na **5217**, com `node_modules` em volume próprio. `/citizen` já está no proxy (`vite.config.ts`). O banco de dev só é migrado e semeado **com autorização do usuário** (plano do api).

## Review Focus

1. **Denúncia anônima que identifique quem denunciou.** Riscos: cookie da sessão no POST, chave a mais no corpo, protocolo na tela, telefone ou CPF da sessão no texto da tela. Testes:
   - Task 1, "anonymousReport: corpo exato, sem cookie e sem citizen:unauthenticated";
   - Task 4, "denúncia anônima: aviso antes do envio; enviada sem protocolo";
   - Task 9, e2e "Denúncia anônima: sem cookie, sem protocolo, sem rolagem lateral".
2. **Identidade preservada vazando pelo anexo ou pelo texto.** O nome do arquivo do aparelho ("RG João Silva.pdf") não sai; a dica do modo `preserved` pede para não escrever dados pessoais. Testes:
   - Task 2, "attachmentFilename: genérico, pela ordem e pelo tipo";
   - Task 3, "AttachmentPicker: sobe com nome genérico, recusa tipo, tamanho e o 4º";
   - Task 5, "preservada: dica de não escrever dados pessoais".
3. **Encaminhamento ou dado interno chegando ao cidadão** (`forwarded`, `extract`, `subject`, `forwardings`, `responder_name` num api com defeito). Some na borda, e `forwarded` vira "Em análise". Testes:
   - Task 1, "manifestation: só os campos do contrato";
   - Task 2, "stateLabel: forwarded é Em análise";
   - Task 6, "detalhe: nada do encaminhamento".
4. **Pessoa errada no `citizen_id`.** Com várias pessoas, a escolha é obrigatória e vai no corpo; com uma, vai a única; sem nenhuma, a tela leva ao cadastro do CPF (W2). Testes: Task 5, "duas pessoas: escolhe e manda o citizen_id escolhido", "uma pessoa: vai direto ao tipo" e "nenhuma pessoa: leva ao cadastro do CPF".
5. **Toque duplo e resposta tardia.** Dois toques em "Enviar" fazem um POST só; a resposta que chega depois de sair da tela não escreve. Testes:
   - Task 4, "dois toques: um POST";
   - Task 5, "dois toques: uma manifestação";
   - Task 3, "upload que termina depois do desmonte não escreve".
6. **Ações fora de hora** (complemento depois do prazo, reabertura fora dos 30 dias, pesquisa já respondida). A tela mostra a frase própria e relê o detalhe. Teste: Task 6, "409/422 nas ações: frase própria e releitura".
7. **Entrada sem o interruptor ou com o api antigo** (`public_config` em 404; sessão sem `ombudsman`). Nada da ouvidoria aparece e o resto do canal fica igual. Testes: Task 4, "public_config 404: nada"; Task 7, "sem ombudsman na sessão: Para quem é igual".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | cards, âncoras, rotas do api, worktree, base | 0 |
| `src/lib/citizenApi.ts`, `src/lib/citizenApi.ombudsman.test.ts` | `publicCall`, os nove métodos, as normalizações, o aviso `ombudsman` | 1 |
| `src/lib/ombudsman.ts`, `src/lib/ombudsman.test.ts` | rótulos, prazos, contagem, anexos, mensagens | 2 |
| `src/modules/citizen/OmbudsmanFields.tsx`, `src/modules/citizen/OmbudsmanFields.test.tsx` | `TextArea`, `UnitSelect`, `AttachmentPicker` | 3 |
| `src/modules/citizen/OmbudsmanEntry.tsx`, `src/modules/citizen/AnonymousReportStep.tsx`, `src/modules/citizen/AnonymousReportStep.test.tsx` | entrada da tela inicial e denúncia anônima | 4 |
| `src/modules/citizen/NewManifestationStep.tsx`, `src/modules/citizen/NewManifestationStep.test.tsx` | nova manifestação | 5 |
| `src/modules/citizen/MyManifestationsStep.tsx`, `src/modules/citizen/ManifestationDetailStep.tsx`, `src/modules/citizen/MyManifestationsStep.test.tsx` | lista, detalhe, complemento, reabertura, pesquisa | 6 |
| `src/lib/route.ts`, `src/lib/route.test.ts`, `src/modules/citizen/Flow.tsx`, `src/modules/citizen/PhoneStep.tsx`, `src/modules/citizen/PeopleStep.tsx`, `src/modules/citizen/Flow.ombudsman.test.tsx`, `README.md` | estados, rota `ouvidoria`, entradas, README | 7 |
| `src/modules/citizen/NoticesStep.tsx`, `src/modules/citizen/NoticesStep.ombudsman.test.tsx` | aviso de ouvidoria na caixa | 8 |
| `e2e/ombudsman.spec.ts` | formulário anônimo nos celulares | 9 |
| — | suíte, build, e2e, revisão, prova | 10 |

**Estratégia de teste:**
- **Cliente** (Task 1), com `fetch` falso. Confere a URL, o corpo exato, `credentials`, o descarte de campos fora do contrato e o aviso `ombudsman`.
- **Regras puras** (Task 2), com tabela de casos: rótulos, prazos (verde, amarelo, vencido, suspenso, final, data inválida), contagem com emoji e espaços, anexos e mensagens.
- **Peças de formulário** (Task 3), com `user-event`: contador, `user.upload` com arquivo válido, de outro tipo, grande e o 4º, e o desmonte no meio do upload.
- **Telas** (Tasks 4–6, 8), com dublês de `citizenApi`: o caminho feliz, cada recusa do api e o descarte.
- **Navegação real pelo `Flow`** (Task 7): sem sessão (tela inicial → anônima → volta; "Fazer uma manifestação" → login → ouvidoria), com sessão, sem o interruptor, e a rota `/wpda/ouvidoria`.
- **Playwright nos celulares** (Task 9): `iphone` e `pixel` em UTC, com um cookie `citizen_session` plantado para provar que a denúncia não o leva. Sem rolagem lateral.
- **Prova contra o api do 33** (Task 10): os três modos, complemento, aviso e pesquisa.

---

### Task 0: Cards, âncoras, rotas do api, worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do wpda (`339e4ac` ou mais novo); board #1 (`PVT_kwDOEbfGRc4BbxiB`, Status `PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`); o worktree do api `apps/api/.claude/mod33`.
- Produces: os cards em In Progress; `$W`; a base anotada; a forma real das rotas do api.

- [ ] **Step 1: Cards F-33.1..F-33.9 para In Progress.** Um `gh` simples por chamada, sem `$(...)` nem laço:

```bash
gh project item-list 1 --owner rotasaude --limit 500 --format json --jq '.items[] | select(.title | test("^F-33\\.[1-9] ")) | {id, title, status}'
gh project field-list 1 --owner rotasaude --format json --jq '.fields[] | select(.name == "Status") | {id, options}'
gh project item-edit --id <PVTI_do_card> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id_In_Progress>
```

Expected: os nove em In Progress ou além. Card ausente: **pare e reporte**.

- [ ] **Step 2: Âncoras e o que o api do 33 fixou**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git; A=$R/apps/api/.claude/mod33
$G -C $R/apps/wpda fetch -q origin && $G -C $R/apps/wpda log --oneline -3 origin/main
$G -C $R/apps/wpda show origin/main:src/lib/citizenApi.ts | grep -n '^async function call<T>\|^export type Notice = \|^function normalizeNotice\|if (n.kind !== undefined && n.kind !== "campaign") return null\|^// ─── Minhas viagens (módulo 30\|^  trips: async'
$G -C $R/apps/wpda show origin/main:src/lib/route.ts | grep -n '^export type AppRoute\|^const ROUTES'
$G -C $R/apps/wpda show origin/main:src/modules/citizen/Flow.tsx | grep -n '| { at: "phone" }\|| { at: "declined" }\|if (route === "avisos")\|if (s.at === "notices") target.current\|const signedIn = \|case "phone":\|case "notices":\|case "declined":'
$G -C $R/apps/wpda show origin/main:src/modules/citizen/PhoneStep.tsx | grep -n 'export function PhoneStep\|<Screen title="Triagem de saúde"'
$G -C $R/apps/wpda show origin/main:src/modules/citizen/PeopleStep.tsx | grep -n 'export function PeopleStep\|Continuar com este CPF'
$G -C $R/apps/wpda show origin/main:src/modules/citizen/NoticesStep.tsx | grep -n '^export const TRIP_NOTICE_HINT\|if (notice.kind === "trip")\|if (current && current.kind === "trip")\|export function NoticesStep'
$G -C $R/apps/wpda show origin/main:README.md | grep -n '^\*\*Urgência (módulo 32a).\*\*\|^### Relatório público'
docker compose -f $R/docker-compose.yml exec -T -w /rails/.claude/mod33 api bin/rails routes 2>/dev/null | grep -E "citizen/(ombudsman|notices|session)"
grep -rn "render_error\|error:" $A/app/controllers/citizen_api/ombudsman* 2>/dev/null | grep -o '"[a-z_]*"' | sort -u
grep -n "ombudsman" $A/app/controllers/citizen_api/notices_controller.rb
```

Expected:
- as âncoras do wpda;
- no api, as nove rotas de `/citizen/ombudsman` e a lista real dos códigos de erro;
- o `ombudsman_json` da caixa de avisos.

Ajuste **antes** de começar, com nota para a Task 10, se a forma real for diferente: envelope, nomes dos campos, códigos de erro (W6, W11), `kind` da linha do tempo (W4), `cpf_masked` no aviso (W3), `forwarded` na visão do cidadão (W5). O ajuste vai em `normalize*` (Task 1), em `ombudsmanMessage`/`timelineTitle` (Task 2) e nas fixtures. Se a sessão não listar `ombudsman`, avise a sessão do api.

- [ ] **Step 3: Worktree e base**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/wpda worktree add -q $R/.worktrees/wpda-mod33 -b feat/mod-33-ombudsman origin/main
cd $R/.worktrees/wpda-mod33 && npm ci --silent && npx tsc --noEmit && echo tsc-ok
$R/bin/test wpda -w wpda-mod33
```

Expected: verde e `tsc-ok`. **Anote** a base. A Task 10 espera:
- **+8 arquivos de teste:** `citizenApi.ombudsman`, `ombudsman`, `OmbudsmanFields`, `AnonymousReportStep`, `NewManifestationStep`, `MyManifestationsStep`, `Flow.ombudsman`, `NoticesStep.ombudsman`, mais os casos novos em `route.test.ts`;
- **+1 e2e** (`e2e/ombudsman.spec.ts`).

**Relato ao coordenador:** "wpda-mod33: worktree e base verdes (<N> testes); rotas e códigos reais do api anotados; ajustes: <lista ou 'nenhum'>".

---

### Task 1: Cliente — ouvidoria em `citizenApi`

**Files:**
- Modify: `src/lib/citizenApi.ts`:
  - `publicCall` logo depois de `call`;
  - bloco "Ouvidoria (módulo 33)" depois do de Minhas viagens;
  - `OmbudsmanNotice` em `Notice` e em `normalizeNotice`;
  - métodos depois de `trips`.
- Test: `src/lib/citizenApi.ombudsman.test.ts`

**Interfaces:**
- Consumes: `call`, `str`, `strOrNull`, `rec`, `ApiError` (existentes).
- Produces:
  - `type ManifestationKind = "complaint" | "report" | "praise" | "suggestion" | "request"`;
  - `type IdentificationMode = "identified" | "preserved"`;
  - `type ManifestationState = "received" | "in_analysis" | "forwarded" | "awaiting_complement" | "answered" | "archived"`;
  - `type DeadlineColor = "green" | "yellow" | "red" | "paused"`;
  - `interface OmbudsmanPublicConfig { anonymous_web: boolean; units: { id: string; name: string }[] }`;
  - `interface CitizenManifestationSummary { id; protocol; kind: string; identification_mode: string; state: string; received_at: string; due_on: string | null; color: DeadlineColor | null; cpf_masked: string | null; has_unread: boolean }`;
  - `interface ManifestationTimelineItem { kind: string; at: string; text: string | null }`;
  - `interface CitizenManifestation { id; protocol; kind; identification_mode; state; received_at; due_on: string | null; body: string; health_unit: { id: string; name: string } | null; attachments: { id: string; filename: string }[]; timeline: ManifestationTimelineItem[]; can_complement: boolean; can_request_reopen: boolean; can_answer_satisfaction: boolean }`;
  - `interface OmbudsmanNotice { kind: "ombudsman"; id: string; event: string; protocol: string | null; created_at: string; read: boolean; cpf_masked: string | null }`, com `Notice = CampaignNotice | ReminderNotice | TripNotice | OmbudsmanNotice`;
  - os métodos de `citizenApi`:
    - `ombudsmanPublicConfig(): Promise<OmbudsmanPublicConfig>`;
    - `sendAnonymousReport(r: { body: string; healthUnitId: string | null }): Promise<void>`;
    - `manifestations(): Promise<CitizenManifestationSummary[]>`;
    - `manifestation(id: string): Promise<CitizenManifestation>`;
    - `createManifestation(m: { citizenId: string; kind: ManifestationKind; mode: IdentificationMode; body: string; healthUnitId: string | null; attachmentIds: string[] }): Promise<{ id: string; protocol: string; due_on: string | null }>`;
    - `complementManifestation(id: string, c: { body: string; attachmentIds: string[] }): Promise<void>`;
    - `requestReopen(id: string, reason: string): Promise<void>`;
    - `answerSatisfaction(id: string, a: { met: "yes" | "partly" | "no"; clear: boolean; score: number }): Promise<void>`;
    - `uploadOmbudsmanAttachment(f: { filename: string; contentType: string; data: string }): Promise<{ id: string }>`.

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/citizenApi.ombudsman.test.ts
// Módulo 33 (contrato §3, §4): ouvidoria no cliente. Só os campos do contrato passam;
// as rotas públicas nunca levam o cookie nem derrubam a sessão.
import { describe, it, expect, vi, afterEach } from "vitest";
import { citizenApi, ApiError } from "./citizenApi";

afterEach(() => { vi.unstubAllGlobals(); vi.restoreAllMocks(); });

function stubFetch(status: number, body: unknown) {
  const fetchMock = vi.fn(async () => new Response(JSON.stringify(body), { status, headers: { "Content-Type": "application/json" } }));
  vi.stubGlobal("fetch", fetchMock);
  return fetchMock;
}
const sent = (m: ReturnType<typeof stubFetch>, i = 0) => {
  const [ url, init ] = m.mock.calls[i] as unknown as [ string, RequestInit ];
  return { url, init, body: init.body ? JSON.parse(init.body as string) : undefined };
};

describe("citizenApi — ouvidoria pública (módulo 33, §4)", () => {
  it("publicConfig: sem cookie; unidades válidas; anonymous_web só com true", async () => {
    const m = stubFetch(200, { anonymous_web: "yes", units: [ { id: "u1", name: "UBS Centro" }, { id: 7 }, { name: "sem id" } ] });
    expect(await citizenApi.ombudsmanPublicConfig()).toEqual({ anonymous_web: false, units: [ { id: "u1", name: "UBS Centro" } ] });
    expect(sent(m).url).toBe("/citizen/ombudsman/public_config");
    expect(sent(m).init.credentials).toBe("omit");
  });

  it("anonymousReport: corpo exato, sem cookie e sem citizen:unauthenticated", async () => {
    const m = stubFetch(201, { ok: true, protocol: "2026-000999" });
    await expect(citizenApi.sendAnonymousReport({ body: "Faltou médico o dia todo na UBS.", healthUnitId: "u1" })).resolves.toBeUndefined();
    expect(sent(m)).toMatchObject({ url: "/citizen/ombudsman/anonymous_reports",
      body: { body: "Faltou médico o dia todo na UBS.", health_unit_id: "u1" } });
    expect(Object.keys(sent(m).body)).toEqual([ "body", "health_unit_id" ]);
    expect(sent(m).init.credentials).toBe("omit");

    stubFetch(201, { ok: true });
    await citizenApi.sendAnonymousReport({ body: "Faltou médico o dia todo na UBS.", healthUnitId: null });
    const fetchMock = vi.mocked(fetch);
    expect(JSON.parse((fetchMock.mock.calls[0][1] as RequestInit).body as string)).toEqual({ body: "Faltou médico o dia todo na UBS." });

    const onUnauth = vi.fn();
    window.addEventListener("citizen:unauthenticated", onUnauth);
    stubFetch(401, { error: "unauthenticated" });
    await expect(citizenApi.sendAnonymousReport({ body: "x".repeat(20), healthUnitId: null })).rejects.toEqual(new ApiError(401, "unauthenticated"));
    expect(onUnauth).not.toHaveBeenCalled();
    window.removeEventListener("citizen:unauthenticated", onUnauth);
  });

  it("anonymousReport: 429 e 404 viram ApiError com o código", async () => {
    stubFetch(429, { error: "daily_cap_reached" });
    await expect(citizenApi.sendAnonymousReport({ body: "x".repeat(20), healthUnitId: null })).rejects.toMatchObject({ status: 429, code: "daily_cap_reached" });
    stubFetch(404, { error: "feature_disabled" });
    await expect(citizenApi.ombudsmanPublicConfig()).rejects.toMatchObject({ status: 404, code: "feature_disabled" });
  });
});

describe("citizenApi — minhas manifestações (módulo 33, §3)", () => {
  const SUMMARY = { id: "m1", protocol: "2026-000123", kind: "complaint", identification_mode: "preserved", state: "forwarded",
    received_at: "2026-11-10T09:00:00-03:00", due_on: "2026-12-10", color: "green", cpf_masked: "***.982.247-**", has_unread: true,
    subject: { name: "Atendimento" }, forwardings: [ { destination: "UBS Centro" } ] };

  it("manifestations: lista com os campos do contrato; cor desconhecida vira null", async () => {
    const m = stubFetch(200, { manifestations: [ SUMMARY, { ...SUMMARY, id: "m2", color: "blue", cpf_masked: undefined, has_unread: "1" } ] });
    const list = await citizenApi.manifestations();
    expect(sent(m).url).toBe("/citizen/ombudsman/manifestations");
    expect(list[0]).toEqual({ id: "m1", protocol: "2026-000123", kind: "complaint", identification_mode: "preserved", state: "forwarded",
      received_at: "2026-11-10T09:00:00-03:00", due_on: "2026-12-10", color: "green", cpf_masked: "***.982.247-**", has_unread: true });
    expect(list[1]).toMatchObject({ color: null, cpf_masked: null, has_unread: false });
  });

  it("manifestation: só os campos do contrato", async () => {
    const m = stubFetch(200, { id: "m1", protocol: "2026-000123", kind: "complaint", identification_mode: "identified", state: "awaiting_complement",
      received_at: "2026-11-10T09:00:00-03:00", due_on: "2026-12-10", body: "Esperei 4 horas.", health_unit: { id: "u1", name: "UBS Centro" },
      attachments: [ { id: "a1", filename: "anexo-1.pdf", exam_file_id: "x" }, { filename: "sem id" } ],
      timeline: [ { kind: "received", at: "2026-11-10T09:00:00-03:00" }, { kind: "complement_requested", at: "2026-11-11T09:00:00-03:00", text: "Qual o horário?", actor_user: "Ana" } ],
      can_complement: true, can_request_reopen: "true", can_answer_satisfaction: false,
      extract: "Cidadã relata espera", subject: { name: "Atendimento" }, forwardings: [ { reply: "x" } ], responder_name: "Carlos" });
    const d = await citizenApi.manifestation("m1");
    expect(sent(m).url).toBe("/citizen/ombudsman/manifestations/m1");
    expect(d).toEqual({ id: "m1", protocol: "2026-000123", kind: "complaint", identification_mode: "identified", state: "awaiting_complement",
      received_at: "2026-11-10T09:00:00-03:00", due_on: "2026-12-10", body: "Esperei 4 horas.", health_unit: { id: "u1", name: "UBS Centro" },
      attachments: [ { id: "a1", filename: "anexo-1.pdf" } ],
      timeline: [ { kind: "received", at: "2026-11-10T09:00:00-03:00", text: null },
        { kind: "complement_requested", at: "2026-11-11T09:00:00-03:00", text: "Qual o horário?" } ],
      can_complement: true, can_request_reopen: false, can_answer_satisfaction: false });
    expect(JSON.stringify(d)).not.toMatch(/extract|Atendimento|Carlos|Ana|forwardings/);
  });

  it("escritas: corpo exato, citizen_id só no corpo, URL sem dado", async () => {
    let m = stubFetch(201, { id: "m9", protocol: "2026-000124", due_on: "2026-12-12" });
    expect(await citizenApi.createManifestation({ citizenId: "p2", kind: "report", mode: "preserved", body: "Texto com mais de vinte letras.",
      healthUnitId: null, attachmentIds: [ "a1" ] })).toEqual({ id: "m9", protocol: "2026-000124", due_on: "2026-12-12" });
    expect(sent(m)).toMatchObject({ url: "/citizen/ombudsman/manifestations", body: { citizen_id: "p2", kind: "report",
      identification_mode: "preserved", body: "Texto com mais de vinte letras.", attachment_ids: [ "a1" ] } });
    expect(sent(m).body).not.toHaveProperty("health_unit_id");
    expect(sent(m).init.credentials).toBe("same-origin");

    m = stubFetch(200, {});
    await citizenApi.complementManifestation("m1", { body: "O horário foi 7h da manhã, na recepção.", attachmentIds: [] });
    expect(sent(m)).toMatchObject({ url: "/citizen/ombudsman/manifestations/m1/complement", body: { body: "O horário foi 7h da manhã, na recepção." } });
    expect(sent(m).body).not.toHaveProperty("attachment_ids");

    m = stubFetch(201, {});
    await citizenApi.requestReopen("m1", "Ninguém me ligou de volta.");
    expect(sent(m)).toMatchObject({ url: "/citizen/ombudsman/manifestations/m1/reopen_request", body: { reason: "Ninguém me ligou de volta." } });

    m = stubFetch(201, {});
    await citizenApi.answerSatisfaction("m1", { met: "partly", clear: true, score: 4 });
    expect(sent(m).body).toEqual({ met: "partly", clear: true, score: 4 });

    m = stubFetch(201, { id: "a7" });
    expect(await citizenApi.uploadOmbudsmanAttachment({ filename: "anexo-1.pdf", contentType: "application/pdf", data: "JVBERi0=" })).toEqual({ id: "a7" });
    expect(sent(m).body).toEqual({ filename: "anexo-1.pdf", content_type: "application/pdf", data: "JVBERi0=" });
    for (const c of vi.mocked(fetch).mock.calls) expect(String(c[0])).not.toContain("?");
  });

  it("notices: aviso ombudsman com evento, protocolo e data; sem texto", async () => {
    stubFetch(200, { unread_count: 2, notices: [
      { kind: "ombudsman", id: "o1", event: "final_response", protocol: "2026-000123", created_at: "2026-11-11T14:00:00-03:00", read: false,
        body: "Sua reclamação foi resolvida", text: "x" },
      { kind: "ombudsman", id: "o2", created_at: "2026-11-11T14:00:00-03:00" },
      { kind: "ombudsman", id: "o3" }
    ] });
    const r = await citizenApi.notices();
    expect(r.notices).toEqual([
      { kind: "ombudsman", id: "o1", event: "final_response", protocol: "2026-000123", created_at: "2026-11-11T14:00:00-03:00", read: false, cpf_masked: null },
      { kind: "ombudsman", id: "o2", event: "", protocol: null, created_at: "2026-11-11T14:00:00-03:00", read: false, cpf_masked: null }
    ]);
  });
});
```

- [ ] **Step 2: Run** `bin/test wpda -w wpda-mod33 src/lib/citizenApi.ombudsman.test.ts`. Expected: FAIL (os métodos não existem; o aviso `ombudsman` é descartado).

- [ ] **Step 3: Implemente**
  - **`publicCall<T>(method, path, body?)`**, logo depois de `call`, com o comentário `// Módulo 33 (contrato §4; W7): rotas públicas da ouvidoria. Nunca levam o cookie da sessão (a denúncia anônima não se liga a ninguém) e um 401 não derruba a sessão.`. Igual a `call`, com três diferenças:
    - `credentials: "omit"`;
    - sem o `dispatchEvent` do 401;
    - `204` ou corpo vazio viram `undefined`.
  - **Bloco "Ouvidoria (módulo 33, ADR 0051; contrato §3 e §4)"** depois do de Minhas viagens, com os tipos da Interface e as funções:
    - `normalizePublicConfig`: `anonymous_web === true`; unidades só com `id` e `name` texto;
    - `normalizeManifestationSummary`: `color` só se for uma das quatro, senão `null`; `has_unread === true`; `cpf_masked` por `strOrNull`;
    - `normalizeManifestation`: monta campo a campo, como o `normalizeTrips`; anexo sem `id` texto some; `timeline[].text` por `strOrNull`; os três `can_*` só com `=== true`;
    - um comentário de uma linha que diz que o extrato, o encaminhamento, o assunto e quem respondeu nunca passam da borda.
  - **Métodos**, depois de `trips`:
    - `ombudsmanPublicConfig` e `sendAnonymousReport` usam `publicCall`. O corpo da denúncia é `{ body, ...(healthUnitId ? { health_unit_id: healthUnitId } : {}) }`, e a resposta é descartada (nunca devolve protocolo);
    - os outros usam `call`. `attachment_ids` e `health_unit_id` só vão quando têm valor; o id na URL passa por `encodeURIComponent`.
  - **`normalizeNotice`:** antes da linha `if (n.kind !== undefined && n.kind !== "campaign") return null`, o caso `n.kind === "ombudsman"` devolve `{ kind: "ombudsman", id, event: n.event ?? "", protocol: strOrNull(n.protocol), created_at, read, cpf_masked }`. Sem `created_at` texto, devolve `null`. Em `RawNotice`, acrescente `protocol?: string | null; created_at?: string`. `str`, `strOrNull` e `rec` estão declarados mais abaixo no arquivo, no bloco do 29: mova as três constantes para logo depois de `ApiError`, sem mudar o corpo delas.

- [ ] **Step 4: Run** o mesmo comando, `bin/test wpda -w wpda-mod33 src/lib` e `cd $W && npx tsc --noEmit`. Expected: PASS. Os `citizenApi.*` antigos continuam iguais (inclusive "notices: tipo do CAPS fica fora").

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/citizenApi.ts src/lib/citizenApi.ombudsman.test.ts
$G commit -m "feat: add the ombudsman routes to the citizen api client

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** "wpda-mod33 Task 1: cliente da ouvidoria pronto; rotas públicas com credentials omit; aviso `ombudsman` normalizado".

---

### Task 2: Regras puras — `src/lib/ombudsman.ts`

**Files:**
- Create: `src/lib/ombudsman.ts`
- Test: `src/lib/ombudsman.test.ts`

**Interfaces:**
- Consumes: `ApiError`, `ManifestationState`, `DeadlineColor` (Task 1); `fmtCalendarDate`, `cityToday` (`format.ts`); `messageFor` (`ui.tsx`).
- Produces:
  - constantes: `OMBUDSMAN_FEATURE = "ombudsman"`, `ANONYMOUS_WEB_FEATURE = "ombudsman_anonymous_web"`, `ANONYMOUS_WARNING`, `PRESERVED_HINT`, `BODY_MIN = 20`, `BODY_MAX = 4000`, `REASON_MIN = 10`, `REASON_MAX = 2000`, `MAX_ATTACHMENTS = 3`, `MAX_ATTACHMENT_BYTES = 10 * 1024 * 1024`, `ATTACHMENT_TYPES`, `OMBUDSMAN_NOTICE_HINT`;
  - opções: `KIND_OPTIONS` (valor, rótulo, explicação), `MODE_OPTIONS`, `SATISFACTION` (as três perguntas do contrato §10, com as opções);
  - funções:
    - `kindLabel(kind)`, `stateLabel(state)`, `deadlineText({ state, due_on, color }, today = cityToday())`;
    - `timelineTitle(kind)`, `noticeTitle(event)`;
    - `textLength(s)`, `bodyError(s)`, `reasonError(s)`;
    - `attachmentError(file, count)`, `attachmentFilename(index, contentType)`, `readAsBase64(file)`;
    - `ombudsmanMessage(error)`.

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/ombudsman.test.ts
import { describe, it, expect } from "vitest";
import { ApiError } from "./citizenApi";
import {
  ANONYMOUS_WARNING, attachmentError, attachmentFilename, bodyError, deadlineText, kindLabel, noticeTitle, ombudsmanMessage,
  readAsBase64, reasonError, stateLabel, textLength, timelineTitle
} from "./ombudsman";

describe("ombudsman — textos do contrato", () => {
  it("aviso da denúncia anônima, palavra por palavra (contrato §14)", () => {
    expect(ANONYMOUS_WARNING).toBe("Sem identificação, você não recebe protocolo nem acompanha a resposta.");
  });

  it("kindLabel e stateLabel; forwarded é Em análise; desconhecido não mostra o código", () => {
    expect([ "complaint", "report", "praise", "suggestion", "request" ].map(kindLabel))
      .toEqual([ "Reclamação", "Denúncia", "Elogio", "Sugestão", "Solicitação" ]);
    expect([ "received", "in_analysis", "forwarded", "awaiting_complement", "answered", "archived", "xpto" ].map(stateLabel))
      .toEqual([ "Recebida", "Em análise", "Em análise", "Aguardando seu complemento", "Respondida", "Arquivada", "Em andamento" ]);
    expect(kindLabel("xpto")).toBe("Manifestação");
  });

  it("timelineTitle e noticeTitle; desconhecido vira neutro", () => {
    expect(timelineTitle("received")).toBe("Manifestação recebida");
    expect(timelineTitle("complement_requested")).toBe("A Ouvidoria pediu um complemento");
    expect(timelineTitle("final_response")).toBe("Resposta da Ouvidoria");
    expect(timelineTitle("forwarded")).toBe("Atualização da Ouvidoria");
    expect(noticeTitle("final_response")).toBe("Sua manifestação foi respondida");
    expect(noticeTitle("complement_requested")).toBe("A Ouvidoria pediu um complemento");
    expect(noticeTitle("")).toBe("Aviso da Ouvidoria");
  });
});

describe("deadlineText", () => {
  const TODAY = "2026-11-12";
  it.each([
    [ { state: "in_analysis", due_on: "2026-12-10", color: "green" }, "Resposta prevista até 10/12/2026" ],
    [ { state: "forwarded", due_on: "2026-11-15", color: "yellow" }, "Resposta prevista até 15/11/2026" ],
    [ { state: "in_analysis", due_on: "2026-11-11", color: "red" }, "O prazo de resposta venceu em 11/11/2026" ],
    [ { state: "in_analysis", due_on: "2026-11-11", color: null }, "O prazo de resposta venceu em 11/11/2026" ],
    [ { state: "awaiting_complement", due_on: "2026-12-10", color: "paused" }, "Prazo suspenso até você enviar o complemento" ],
    [ { state: "answered", due_on: "2026-12-10", color: null }, null ],
    [ { state: "archived", due_on: null, color: null }, null ],
    [ { state: "received", due_on: "10/12/2026", color: "green" }, null ]
  ])("%o → %s", (m, text) => expect(deadlineText(m as never, TODAY)).toBe(text));
});

describe("texto e motivo", () => {
  it("conta code points depois do trim (emoji conta 1, como o Ruby)", () => {
    expect(textLength("  abc  ")).toBe(3);
    expect(textLength("😀".repeat(20))).toBe(20);
  });
  it("bodyError e reasonError nos limites", () => {
    expect(bodyError(" ".repeat(30))).toBe("Escreva pelo menos 20 caracteres.");
    expect(bodyError("a".repeat(19))).toBe("Escreva pelo menos 20 caracteres.");
    expect(bodyError("a".repeat(20))).toBeNull();
    expect(bodyError("a".repeat(4000))).toBeNull();
    expect(bodyError("a".repeat(4001))).toBe("Escreva no máximo 4.000 caracteres.");
    expect(reasonError("a".repeat(9))).toBe("Escreva pelo menos 10 caracteres.");
    expect(reasonError("a".repeat(2001))).toBe("Escreva no máximo 2.000 caracteres.");
  });
});

describe("anexos", () => {
  const file = (type: string, size = 100) => { const f = new File([ "x" ], "RG João Silva.pdf", { type }); Object.defineProperty(f, "size", { value: size }); return f; };
  it("attachmentError: tipo, tamanho e quantidade", () => {
    expect(attachmentError(file("application/pdf"), 0)).toBeNull();
    expect(attachmentError(file("image/jpeg"), 2)).toBeNull();
    expect(attachmentError(file("image/png", 10 * 1024 * 1024), 0)).toBeNull();
    expect(attachmentError(file("image/png", 10 * 1024 * 1024 + 1), 0)).toBe("O arquivo passa de 10 MB.");
    expect(attachmentError(file("image/gif"), 0)).toBe("Envie um PDF, uma foto JPEG ou PNG.");
    expect(attachmentError(file("application/pdf"), 3)).toBe("No máximo 3 anexos.");
  });
  it("attachmentFilename: genérico, pela ordem e pelo tipo", () => {
    expect(attachmentFilename(0, "application/pdf")).toBe("anexo-1.pdf");
    expect(attachmentFilename(1, "image/jpeg")).toBe("anexo-2.jpg");
    expect(attachmentFilename(2, "image/png")).toBe("anexo-3.png");
  });
  it("readAsBase64: só o conteúdo, sem o prefixo data:", async () => {
    expect(await readAsBase64(new File([ "%PDF-1" ], "a.pdf", { type: "application/pdf" }))).toBe("JVBERi0x");
  });
});

describe("ombudsmanMessage", () => {
  it.each([
    [ "feature_disabled", "A Ouvidoria não está disponível agora nesta cidade." ],
    [ "rate_limited", "Você enviou muitas denúncias em pouco tempo. Tente mais tarde." ],
    [ "daily_cap_reached", "O canal de denúncias recebeu muitas mensagens hoje. Tente mais tarde." ],
    [ "too_many_requests", "Muitos envios em pouco tempo. Tente mais tarde." ],
    [ "payload_too_large", "O texto ficou grande demais. Encurte e tente de novo." ],
    [ "http_413", "O texto ficou grande demais. Encurte e tente de novo." ],
    [ "not_own_citizen", "Escolha uma das pessoas deste celular." ],
    [ "invalid_file", "Envie um PDF, uma foto JPEG ou PNG, de até 10 MB." ],
    [ "manifestation_changed", "A manifestação mudou. Veja como ela está agora." ],
    [ "reopen_window_closed", "O prazo de 30 dias para pedir a reabertura terminou." ],
    [ "already_answered", "Você já respondeu a pesquisa desta manifestação." ]
  ])("%s", (code, text) => expect(ombudsmanMessage(new ApiError(422, code))).toBe(text));
  it("o resto cai em messageFor", () => {
    expect(ombudsmanMessage(new ApiError(500, "boom"))).toBe("Algo deu errado. Tente de novo.");
    expect(ombudsmanMessage(new TypeError("offline"))).toBe("Sem conexão. Verifique a internet e tente de novo.");
  });
});
```

- [ ] **Step 2: Run** `bin/test wpda -w wpda-mod33 src/lib/ombudsman.test.ts`. Expected: FAIL (o módulo não existe).

- [ ] **Step 3: Implemente** `src/lib/ombudsman.ts`. Cabeçalho: `// Ouvidoria no canal do cidadão (módulo 33, ADR 0051; contrato §3, §4, §10, §14). Regras de tela puras; nada aqui grava, loga ou monta URL.`
  - **Tabelas:**
    - `KIND_LABEL`;
    - `STATE_LABEL`, com `forwarded: "Em análise"` e o comentário `// W5: o cidadão não sabe do encaminhamento (contrato §3)`;
    - `TIMELINE_TITLE` (W4), com `received` "Manifestação recebida", `complement_requested` "A Ouvidoria pediu um complemento", `complement_sent` "Você enviou o complemento", `extended` "Prazo prorrogado", `interim_response` "Resposta parcial da Ouvidoria", `final_response` "Resposta da Ouvidoria", `archived` "Manifestação arquivada", `reopen_requested` "Você pediu a reabertura", `reopen_decided` "Pedido de reabertura respondido", `reopened` "Manifestação reaberta"; o resto vira "Atualização da Ouvidoria";
    - `NOTICE_TITLE` (contrato §1 `ombudsman_notices.kind`), com `interim_response` "Novidade na sua manifestação", `complement_requested` "A Ouvidoria pediu um complemento", `final_response` "Sua manifestação foi respondida", `archived` "Sua manifestação foi arquivada", `reopen_decided` "Pedido de reabertura respondido"; o resto vira "Aviso da Ouvidoria".
  - As buscas usam `Object.prototype.hasOwnProperty.call`, como `tripNoticeTitle`.
  - **`KIND_OPTIONS`**, com a explicação curta de cada tipo: Reclamação ("algo deu errado no atendimento"), Denúncia ("uma irregularidade"), Elogio, Sugestão e Solicitação ("pedir uma providência").
  - **`MODE_OPTIONS`:**
    - `identified`, "Identificada", explicada como "a equipe da Ouvidoria vê quem você é";
    - `preserved`, "Com identidade preservada", explicada como "só o ouvidor pode ver quem você é, e cada vez que ele abre fica registrado".
  - **`PRESERVED_HINT`:** "Para manter sua identidade preservada, não escreva no texto seu nome, CPF, telefone ou endereço." Para os outros modos, a dica é `FROZEN_TEXT_NOTICE` de `ui.tsx`.
  - **`deadlineText`**, nesta ordem:
    1. estado final (`answered`, `archived`) dá `null`;
    2. `awaiting_complement` ou `color === "paused"` dá "Prazo suspenso até você enviar o complemento";
    3. `due_on` que não casa `^\d{4}-\d{2}-\d{2}$` dá `null`;
    4. `color === "red"`, ou `today > due_on` pela ordem do texto, dá "O prazo de resposta venceu em <data>";
    5. senão, "Resposta prevista até <data>".
  - **Texto:** `textLength = [...s.trim()].length`, com o comentário do W9. `bodyError` e `reasonError` comparam com os limites e formatam o máximo com `toLocaleString("pt-BR")`.
  - **Anexos:**
    - `ATTACHMENT_TYPES = { "application/pdf": "pdf", "image/jpeg": "jpg", "image/png": "png" }`;
    - `attachmentError` confere, nesta ordem, a quantidade, o tipo e o tamanho;
    - `attachmentFilename(i, type) = \`anexo-${i + 1}.${ATTACHMENT_TYPES[type]}\``, com o comentário `// W8: nunca o nome do arquivo do aparelho, que pode trazer o nome de quem pediu sigilo.`;
    - `readAsBase64` usa `FileReader.readAsDataURL` e devolve o que vem depois da primeira vírgula; o erro rejeita.
  - **`ombudsmanMessage`:** o mapa do teste mais `too_many_attachments` ("No máximo 3 anexos."), `already_requested` ("Você já pediu a reabertura desta manifestação.") e `satisfaction_window_closed` ("O prazo para responder a pesquisa terminou."). Os códigos `body`, `invalid_body`, `reason` e `invalid_reason` devolvem a frase de tamanho (W6). O resto vai para `messageFor`.

- [ ] **Step 4: Run** o mesmo comando e `cd $W && npx tsc --noEmit`. Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/ombudsman.ts src/lib/ombudsman.test.ts
$G commit -m "feat: add the ombudsman screen rules for the citizen channel

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** "wpda-mod33 Task 2: rótulos, prazos e mensagens da ouvidoria; `forwarded` aparece como 'Em análise' (W5); nome de anexo genérico (W8)".

---

### Task 3: Peças de formulário — `OmbudsmanFields.tsx`

**Files:**
- Create: `src/modules/citizen/OmbudsmanFields.tsx`
- Test: `src/modules/citizen/OmbudsmanFields.test.tsx`

**Interfaces:**
- Consumes: `ErrorText`, `BigButton` (`ui.tsx`); `attachmentError`, `attachmentFilename`, `readAsBase64`, `ombudsmanMessage`, `BODY_MAX`, `MAX_ATTACHMENTS` (Task 2); `citizenApi.uploadOmbudsmanAttachment` (Task 1).
- Produces:
  - `TextArea(props: { label: string; hint?: string; error?: string; value: string; onChange: (v: string) => void; max: number; disabled?: boolean })`: `label` ligado ao campo; `hint` e `error` em `aria-describedby`, como em `Field`; contador "<n> de <max>" com `aria-live="polite"`; 6 linhas, `minHeight` de 160, texto de 18 px, borda `var(--rule2)` ou `var(--down)` com erro;
  - `UnitSelect(props: { units: { id: string; name: string }[]; value: string | null; onChange: (id: string | null) => void })`: `<select>` nativo de 56 px com o rótulo "Unidade de saúde (opcional)" e a primeira opção "Nenhuma unidade específica" (valor `""`, que volta como `null`). Sem unidades, não desenha nada;
  - `export interface PickedAttachment { id: string; filename: string }`;
  - `AttachmentPicker(props: { items: PickedAttachment[]; onChange: (items: PickedAttachment[]) => void; disabled?: boolean })`:
    - `<input type="file" accept="application/pdf,image/jpeg,image/png">` com o rótulo "Anexar arquivo (opcional)" e a dica "Até 3 arquivos, PDF, JPEG ou PNG, de até 10 MB cada.";
    - a lista dos anexos, cada um com "Remover";
    - o input some quando há 3 anexos.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/citizen/OmbudsmanFields.test.tsx
import { describe, it, expect, vi, afterEach } from "vitest";
import { render, screen, waitFor } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { useState } from "react";
import { AttachmentPicker, TextArea, UnitSelect, type PickedAttachment } from "./OmbudsmanFields";
import { citizenApi } from "../../lib/citizenApi";

afterEach(() => { vi.restoreAllMocks(); });

function Picker({ initial = [] as PickedAttachment[] }) {
  const [ items, setItems ] = useState(initial);
  return <AttachmentPicker items={items} onChange={setItems} />;
}
const pdf = (name = "RG João Silva.pdf") => new File([ "%PDF-1" ], name, { type: "application/pdf" });

describe("OmbudsmanFields", () => {
  it("TextArea: rótulo, dica, contador e erro descrevem o campo", async () => {
    function T() { const [ v, setV ] = useState(""); return <TextArea label="O que aconteceu?" hint="Dica" error={v ? undefined : "Erro"} value={v} onChange={setV} max={4000} />; }
    const user = userEvent.setup();
    render(<T />);
    const box = screen.getByRole("textbox", { name: "O que aconteceu?" });
    expect(box).toHaveAccessibleDescription("Dica Erro");
    await user.type(box, "Oi");
    expect(screen.getByText("2 de 4.000")).toBeInTheDocument();
  });

  it("UnitSelect: opcional; vazio volta null; sem unidades não aparece", async () => {
    const onChange = vi.fn();
    const user = userEvent.setup();
    const { rerender } = render(<UnitSelect units={[ { id: "u1", name: "UBS Centro" } ]} value={null} onChange={onChange} />);
    await user.selectOptions(screen.getByRole("combobox", { name: "Unidade de saúde (opcional)" }), "u1");
    expect(onChange).toHaveBeenLastCalledWith("u1");
    await user.selectOptions(screen.getByRole("combobox"), "");
    expect(onChange).toHaveBeenLastCalledWith(null);
    rerender(<UnitSelect units={[]} value={null} onChange={onChange} />);
    expect(screen.queryByRole("combobox")).toBeNull();
  });

  it("AttachmentPicker: sobe com nome genérico, recusa tipo, tamanho e o 4º", async () => {
    const upload = vi.spyOn(citizenApi, "uploadOmbudsmanAttachment").mockResolvedValueOnce({ id: "a1" });
    const user = userEvent.setup({ applyAccept: false });
    render(<Picker initial={[ { id: "a0", filename: "anexo-1.pdf" } ]} />);
    await user.upload(screen.getByLabelText("Anexar arquivo (opcional)"), pdf());
    expect(await screen.findByText("anexo-2.pdf")).toBeInTheDocument();
    expect(upload).toHaveBeenCalledWith({ filename: "anexo-2.pdf", contentType: "application/pdf", data: "JVBERi0x" });
    expect(document.body.textContent).not.toMatch(/João/);

    await user.upload(screen.getByLabelText("Anexar arquivo (opcional)"), new File([ "x" ], "a.gif", { type: "image/gif" }));
    expect(screen.getByRole("alert")).toHaveTextContent("Envie um PDF, uma foto JPEG ou PNG.");
    const big = pdf(); Object.defineProperty(big, "size", { value: 10 * 1024 * 1024 + 1 });
    await user.upload(screen.getByLabelText("Anexar arquivo (opcional)"), big);
    expect(screen.getByRole("alert")).toHaveTextContent("O arquivo passa de 10 MB.");
    expect(upload).toHaveBeenCalledTimes(1);

    upload.mockResolvedValueOnce({ id: "a2" });
    await user.upload(screen.getByLabelText("Anexar arquivo (opcional)"), pdf());
    await screen.findByText("anexo-3.pdf");
    expect(screen.queryByLabelText("Anexar arquivo (opcional)")).toBeNull();
    await user.click(screen.getByRole("button", { name: "Remover anexo-2.pdf" }));
    expect(screen.queryByText("anexo-2.pdf")).toBeNull();
    expect(screen.getByLabelText("Anexar arquivo (opcional)")).toBeInTheDocument();
  });

  it("AttachmentPicker: recusa do api vira frase; upload que termina depois do desmonte não escreve", async () => {
    const { ApiError } = await import("../../lib/citizenApi");
    vi.spyOn(citizenApi, "uploadOmbudsmanAttachment").mockRejectedValueOnce(new ApiError(429, "too_many_requests"));
    const user = userEvent.setup();
    const view = render(<Picker />);
    await user.upload(screen.getByLabelText("Anexar arquivo (opcional)"), pdf());
    expect(await screen.findByRole("alert")).toHaveTextContent("Muitos envios em pouco tempo. Tente mais tarde.");

    let resolve!: (v: { id: string }) => void;
    vi.spyOn(citizenApi, "uploadOmbudsmanAttachment").mockReturnValueOnce(new Promise((r) => { resolve = r; }));
    const errors = vi.spyOn(console, "error");
    await user.upload(screen.getByLabelText("Anexar arquivo (opcional)"), pdf());
    await waitFor(() => expect(screen.getByRole("status")).toHaveTextContent("Enviando o anexo…"));
    view.unmount();
    resolve({ id: "late" });
    await Promise.resolve();
    expect(errors).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run** `bin/test wpda -w wpda-mod33 src/modules/citizen/OmbudsmanFields.test.tsx`. Expected: FAIL (o arquivo não existe).

- [ ] **Step 3: Implemente** `OmbudsmanFields.tsx`. Cabeçalho: `// Campos da Ouvidoria (módulo 33): texto longo, unidade opcional e anexos. Peças locais: nenhum componente comum de ui.tsx muda.`
  - `TextArea` usa `useId` para os ids do rótulo, da dica, do erro e do contador, no mesmo esquema do `Field`. O contador mostra `textLength(value)` (Task 2) formatado em pt-BR.
  - `AttachmentPicker`:
    - **Guardas:** `alive` (`useRef(true)`, que vira `false` no cleanup do `useEffect`) e `busy` (`useState`), para um upload por vez.
    - **No `onChange` do input:**
      1. pega `files[0]` e zera `e.target.value`, para o mesmo arquivo poder ser escolhido de novo;
      2. confere com `attachmentError(file, items.length)`;
      3. lê com `readAsBase64` e sobe com `filename: attachmentFilename(items.length, file.type)`;
      4. com o `id`, chama `onChange([...items, { id, filename }])`, só se `alive.current`.
    - **Mensagens:** o erro vem por `ombudsmanMessage` num `ErrorText`. "Enviando o anexo…" aparece com `role="status"` enquanto `busy`.
    - **"Remover"** é um `button` de link (48 px, como o `linkStyle` de `PeopleStep`), com `aria-label="Remover <nome>"`.
    - Os nomes ficam na ordem; a remoção não renumera os outros. O nome já foi enviado ao api, e um número repetido não identifica ninguém.

- [ ] **Step 4: Run** o mesmo comando, mais `src/theme`, e `cd $W && npx tsc --noEmit`. Expected: PASS (as guardas de cor continuam verdes).

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/citizen/OmbudsmanFields.tsx src/modules/citizen/OmbudsmanFields.test.tsx
$G commit -m "feat: add the text, unit and attachment fields for the ombudsman forms

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** "wpda-mod33 Task 3: campos de texto, unidade e anexos (base64, nome genérico, até 3, 10 MB) prontos".

---

### Task 4: Entrada da tela inicial e denúncia anônima

**Files:**
- Create: `src/modules/citizen/OmbudsmanEntry.tsx`, `src/modules/citizen/AnonymousReportStep.tsx`
- Test: `src/modules/citizen/AnonymousReportStep.test.tsx` (cobre as duas peças)

**Interfaces:**
- Consumes: `citizenApi.ombudsmanPublicConfig`, `citizenApi.sendAnonymousReport` (Task 1); `ANONYMOUS_WARNING`, `bodyError`, `ombudsmanMessage`, `BODY_MAX`, `REPORT_HINT` (Task 2); `TextArea`, `UnitSelect` (Task 3); `Screen`, `BigButton`, `ErrorText` (`ui.tsx`).
- Produces:
  - `OmbudsmanEntry({ onManifest, onAnonymous }: { onManifest: () => void; onAnonymous: () => void })`: lê `public_config` uma vez (`alive`). Com erro (404 `feature_disabled`, rede), não desenha nada. Com a configuração, desenha `<section aria-labelledby>` com o `h2` "Ouvidoria", a frase "Reclamações, denúncias, elogios, sugestões e pedidos à Secretaria de Saúde.", o botão secundário "Fazer uma manifestação" e, só com `anonymous_web`, o botão secundário "Denúncia anônima";
  - `AnonymousReportStep({ onBack }: { onBack: () => void })`: o formulário e a confirmação.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/citizen/AnonymousReportStep.test.tsx
// Módulo 33 (contrato §4, §14; spec decisão 6): entrada da Ouvidoria e denúncia anônima sem login.
import { describe, it, expect, vi, afterEach } from "vitest";
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { AnonymousReportStep } from "./AnonymousReportStep";
import { OmbudsmanEntry } from "./OmbudsmanEntry";
import { citizenApi, ApiError } from "../../lib/citizenApi";

afterEach(() => { vi.restoreAllMocks(); });

const CONFIG = { anonymous_web: true, units: [ { id: "u1", name: "UBS Centro" } ] };
const TEXT = "Na UBS Centro cobraram consulta, o que é proibido no SUS.";

describe("OmbudsmanEntry", () => {
  it("com anonymous_web: os dois caminhos", async () => {
    vi.spyOn(citizenApi, "ombudsmanPublicConfig").mockResolvedValue(CONFIG);
    const onManifest = vi.fn(); const onAnonymous = vi.fn();
    const user = userEvent.setup();
    render(<OmbudsmanEntry onManifest={onManifest} onAnonymous={onAnonymous} />);
    expect(await screen.findByRole("heading", { level: 2, name: "Ouvidoria" })).toBeInTheDocument();
    await user.click(screen.getByRole("button", { name: "Fazer uma manifestação" }));
    await user.click(screen.getByRole("button", { name: "Denúncia anônima" }));
    expect(onManifest).toHaveBeenCalledOnce(); expect(onAnonymous).toHaveBeenCalledOnce();
  });

  it("sem anonymous_web: só a manifestação", async () => {
    vi.spyOn(citizenApi, "ombudsmanPublicConfig").mockResolvedValue({ ...CONFIG, anonymous_web: false });
    render(<OmbudsmanEntry onManifest={vi.fn()} onAnonymous={vi.fn()} />);
    expect(await screen.findByRole("button", { name: "Fazer uma manifestação" })).toBeInTheDocument();
    expect(screen.queryByRole("button", { name: "Denúncia anônima" })).toBeNull();
  });

  it("public_config 404: nada", async () => {
    const spy = vi.spyOn(citizenApi, "ombudsmanPublicConfig").mockRejectedValue(new ApiError(404, "feature_disabled"));
    const { container } = render(<OmbudsmanEntry onManifest={vi.fn()} onAnonymous={vi.fn()} />);
    await vi.waitFor(() => expect(spy).toHaveBeenCalled());
    await Promise.resolve();
    expect(container).toBeEmptyDOMElement();
  });
});

describe("AnonymousReportStep", () => {
  it("denúncia anônima: aviso antes do envio; enviada sem protocolo", async () => {
    vi.spyOn(citizenApi, "ombudsmanPublicConfig").mockResolvedValue(CONFIG);
    const send = vi.spyOn(citizenApi, "sendAnonymousReport").mockResolvedValue(undefined);
    const user = userEvent.setup();
    render(<AnonymousReportStep onBack={vi.fn()} />);
    expect(await screen.findByRole("heading", { level: 1, name: "Denúncia anônima" })).toBeInTheDocument();
    const warning = screen.getByText("Sem identificação, você não recebe protocolo nem acompanha a resposta.");
    const button = screen.getByRole("button", { name: "Enviar denúncia" });
    expect(warning.compareDocumentPosition(button) & Node.DOCUMENT_POSITION_FOLLOWING).toBeTruthy();
    await user.selectOptions(screen.getByRole("combobox", { name: "Unidade de saúde (opcional)" }), "u1");
    await user.click(screen.getByRole("textbox", { name: "O que você quer denunciar?" }));
    await user.paste(`  ${TEXT}  `);
    await user.click(button);
    expect(send).toHaveBeenCalledWith({ body: TEXT, healthUnitId: "u1" });
    expect(await screen.findByRole("heading", { level: 1, name: "Denúncia enviada" })).toBeInTheDocument();
    expect(document.body.textContent).not.toMatch(/[Pp]rotocolo \d|2026-\d{6}/);
    expect(screen.queryByRole("button", { name: /anexo/i })).toBeNull();
  });

  it("texto curto: erro no campo e nada enviado", async () => {
    vi.spyOn(citizenApi, "ombudsmanPublicConfig").mockResolvedValue(CONFIG);
    const send = vi.spyOn(citizenApi, "sendAnonymousReport");
    const user = userEvent.setup();
    render(<AnonymousReportStep onBack={vi.fn()} />);
    await user.click(await screen.findByRole("textbox", { name: "O que você quer denunciar?" }));
    await user.paste("curto");
    await user.click(screen.getByRole("button", { name: "Enviar denúncia" }));
    expect(screen.getByRole("textbox")).toHaveAccessibleDescription(/Escreva pelo menos 20 caracteres\./);
    expect(screen.getByRole("textbox")).toHaveFocus();
    expect(send).not.toHaveBeenCalled();
  });

  it("dois toques: um POST", async () => {
    vi.spyOn(citizenApi, "ombudsmanPublicConfig").mockResolvedValue(CONFIG);
    let resolve!: () => void;
    const send = vi.spyOn(citizenApi, "sendAnonymousReport").mockReturnValue(new Promise<void>((r) => { resolve = r; }));
    const user = userEvent.setup();
    render(<AnonymousReportStep onBack={vi.fn()} />);
    await user.click(await screen.findByRole("textbox", { name: "O que você quer denunciar?" }));
    await user.paste(TEXT);
    await user.dblClick(screen.getByRole("button", { name: "Enviar denúncia" }));
    resolve();
    await screen.findByRole("heading", { level: 1, name: "Denúncia enviada" });
    expect(send).toHaveBeenCalledTimes(1);
  });

  it.each([
    [ new ApiError(429, "rate_limited"), "Você enviou muitas denúncias em pouco tempo. Tente mais tarde." ],
    [ new ApiError(429, "daily_cap_reached"), "O canal de denúncias recebeu muitas mensagens hoje. Tente mais tarde." ],
    [ new ApiError(413, "payload_too_large"), "O texto ficou grande demais. Encurte e tente de novo." ]
  ])("recusa %o: frase própria e o texto continua", async (err, text) => {
    vi.spyOn(citizenApi, "ombudsmanPublicConfig").mockResolvedValue(CONFIG);
    vi.spyOn(citizenApi, "sendAnonymousReport").mockRejectedValue(err);
    const user = userEvent.setup();
    render(<AnonymousReportStep onBack={vi.fn()} />);
    await user.click(await screen.findByRole("textbox", { name: "O que você quer denunciar?" }));
    await user.paste(TEXT);
    await user.click(screen.getByRole("button", { name: "Enviar denúncia" }));
    expect(await screen.findByRole("alert")).toHaveTextContent(text);
    expect(screen.getByRole("textbox")).toHaveValue(TEXT);
  });

  it("anonymous_web desligado (ou 404): explica e só volta", async () => {
    vi.spyOn(citizenApi, "ombudsmanPublicConfig").mockResolvedValue({ ...CONFIG, anonymous_web: false });
    const onBack = vi.fn();
    const user = userEvent.setup();
    render(<AnonymousReportStep onBack={onBack} />);
    expect(await screen.findByText("A denúncia anônima não está disponível agora nesta cidade.")).toBeInTheDocument();
    expect(screen.queryByRole("textbox")).toBeNull();
    await user.click(screen.getByRole("button", { name: "Voltar" }));
    expect(onBack).toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run** `bin/test wpda -w wpda-mod33 src/modules/citizen/AnonymousReportStep.test.tsx`. Expected: FAIL (os arquivos não existem).

- [ ] **Step 3: Implemente**
  - **`OmbudsmanEntry.tsx`:** um `useEffect` com `alive`. O estado é `config | null`, e o erro deixa `null` sem mensagem: a entrada só some, e a tela inicial continua a do celular. Os dois botões são `BigButton variant="secondary"`, porque o principal da tela segue "Receber código".
  - **`AnonymousReportStep.tsx`:**
    - **Leitura:** `public_config` com `alive`. Sem `anonymous_web` ou com erro, mostra só a frase `ANONYMOUS_DISABLED` ("A denúncia anônima não está disponível agora nesta cidade.") e "Voltar".
    - **Formulário:**
      - `Screen title="Denúncia anônima"`;
      - um parágrafo `var(--warn-bg)` com `ANONYMOUS_WARNING` e outro, simples, com "Use este caminho só para denúncias. Para reclamar, elogiar, sugerir ou pedir, faça uma manifestação.";
      - `UnitSelect` e o `TextArea` "O que você quer denunciar?", com o `hint` `REPORT_HINT` ("Conte o que aconteceu, onde e quando. Não escreva seu nome, CPF ou telefone.");
      - no rodapé, o `ANONYMOUS_WARNING` de novo, logo acima de "Enviar denúncia" (principal) e "Voltar" (secundário). É a frase "antes do envio" do contrato §14, à vista sem rolar.
    - **Envio:**
      - a guarda síncrona `inFlight` (`useRef`) e `busy`;
      - `bodyError` falhando põe o erro no campo e o foco nele (WP-07);
      - senão, `sendAnonymousReport({ body: text.trim(), healthUnitId })`, e o sucesso troca para o modo `sent`;
      - o erro aparece com `ombudsmanMessage` num `ErrorText`, sem apagar o texto.
    - **Confirmação:** `Screen title="Denúncia enviada"`, com "Recebemos a sua denúncia. Ela será analisada pela Ouvidoria da Saúde." e "Como você não se identificou, não há protocolo nem acompanhamento por aqui.", e o botão "Voltar ao início" (`onBack`). Nada da resposta do api é lido.
  - Coloque `ANONYMOUS_DISABLED` e `REPORT_HINT` em `lib/ombudsman.ts` (Task 2), com um caso no `ombudsman.test.ts`.

- [ ] **Step 4: Run** o mesmo comando, mais `src/lib/ombudsman.test.ts src/theme`, e `cd $W && npx tsc --noEmit`. Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/citizen/OmbudsmanEntry.tsx src/modules/citizen/AnonymousReportStep.tsx src/modules/citizen/AnonymousReportStep.test.tsx src/lib/ombudsman.ts src/lib/ombudsman.test.ts
$G commit -m "feat: add the ombudsman entry and the anonymous report form without login

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** "wpda-mod33 Task 4: entrada da Ouvidoria e denúncia anônima prontas (aviso antes do envio, sem protocolo, sem cookie); ainda sem ligar ao Flow".

---

### Task 5: Nova manifestação — `NewManifestationStep`

**Files:**
- Create: `src/modules/citizen/NewManifestationStep.tsx`
- Test: `src/modules/citizen/NewManifestationStep.test.tsx`

**Interfaces:**
- Consumes: `citizenApi.people`, `citizenApi.ombudsmanPublicConfig`, `citizenApi.createManifestation` (Task 1); `KIND_OPTIONS`, `MODE_OPTIONS`, `PRESERVED_HINT`, `bodyError`, `ombudsmanMessage`, `BODY_MAX` (Task 2); `TextArea`, `UnitSelect`, `AttachmentPicker` (Task 3); `personLine` (`PeopleStep.tsx`); `Screen`, `BigButton`, `ErrorText`, `RadioGroup`, `FROZEN_TEXT_NOTICE` (`ui.tsx`); `fmtCalendarDate`.
- Produces: `NewManifestationStep({ onBack, onDone, onRegister, onAnonymous }: { onBack: () => void; onDone: (id: string) => void; onRegister: () => void; onAnonymous?: () => void })`. Modos internos, uma pergunta por tela:
  1. `who`, só com 2 ou mais pessoas: "Quem está fazendo a manifestação?", com um `RadioGroup` das pessoas (`CPF <mascarado>` e `personLine`);
  2. `kind`: "Que tipo de manifestação?", com `RadioGroup` de `KIND_OPTIONS`. Com `report` escolhido e `onAnonymous`, aparece o link "Prefere não se identificar? Faça uma denúncia anônima";
  3. `mode`: "Como você quer se identificar?", com `RadioGroup` de `MODE_OPTIONS` e a explicação de cada um;
  4. `text`: `UnitSelect`, o `TextArea` "Conte o que aconteceu" (`hint` `PRESERVED_HINT` no `preserved`, `FROZEN_TEXT_NOTICE` no `identified`) e o `AttachmentPicker`;
  5. `sent`: "Manifestação registrada", com "Protocolo <protocolo>", "Resposta prevista até <dd/mm/aaaa>" e o botão "Ver minha manifestação" (`onDone(id)`).

  Cada modo tem "Continuar" e "Voltar". O "Voltar" do primeiro modo chama `onBack`. Com 0 pessoas, a tela "Antes, cadastre o seu CPF" leva a `onRegister` (W2).

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/citizen/NewManifestationStep.test.tsx
// Módulo 33 (contrato §3, §14; spec decisões 3, 5, 12): nova manifestação identificada ou preservada.
import { describe, it, expect, vi, afterEach } from "vitest";
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { NewManifestationStep } from "./NewManifestationStep";
import { citizenApi, ApiError, type Person } from "../../lib/citizenApi";

afterEach(() => { vi.restoreAllMocks(); });

const P1: Person = { id: "p1", cpf_masked: "***.982.247-**", verification_level: "declared", neighborhood: null, profile: null };
const P2: Person = { id: "p2", cpf_masked: "***.111.444-**", verification_level: "verified", neighborhood: null, profile: null };
const TEXT = "Esperei quatro horas na recepção e ninguém me chamou.";

function stub(people: Person[]) {
  vi.spyOn(citizenApi, "people").mockResolvedValue({ people });
  vi.spyOn(citizenApi, "ombudsmanPublicConfig").mockResolvedValue({ anonymous_web: true, units: [ { id: "u1", name: "UBS Centro" } ] });
  return vi.spyOn(citizenApi, "createManifestation").mockResolvedValue({ id: "m9", protocol: "2026-000124", due_on: "2026-12-12" });
}
async function fillAndSend(user: ReturnType<typeof userEvent.setup>, kind = "Reclamação", mode = "Identificada") {
  await user.click(await screen.findByRole("radio", { name: new RegExp(`^${kind}`) }));
  await user.click(screen.getByRole("button", { name: "Continuar" }));
  await user.click(await screen.findByRole("radio", { name: new RegExp(`^${mode}`) }));
  await user.click(screen.getByRole("button", { name: "Continuar" }));
  await user.click(await screen.findByRole("textbox", { name: "Conte o que aconteceu" }));
  await user.paste(TEXT);
}

describe("NewManifestationStep", () => {
  it("uma pessoa: vai direto ao tipo; manda o corpo exato e mostra o protocolo", async () => {
    const create = stub([ P1 ]);
    const onDone = vi.fn();
    const user = userEvent.setup();
    render(<NewManifestationStep onBack={vi.fn()} onDone={onDone} onRegister={vi.fn()} />);
    expect(await screen.findByRole("heading", { level: 1, name: "Que tipo de manifestação?" })).toBeInTheDocument();
    await fillAndSend(user);
    await user.selectOptions(screen.getByRole("combobox", { name: "Unidade de saúde (opcional)" }), "u1");
    await user.click(screen.getByRole("button", { name: "Enviar manifestação" }));
    expect(create).toHaveBeenCalledWith({ citizenId: "p1", kind: "complaint", mode: "identified", body: TEXT, healthUnitId: "u1", attachmentIds: [] });
    expect(await screen.findByRole("heading", { level: 1, name: "Manifestação registrada" })).toBeInTheDocument();
    expect(screen.getByText("Protocolo 2026-000124")).toBeInTheDocument();
    expect(screen.getByText("Resposta prevista até 12/12/2026")).toBeInTheDocument();
    await user.click(screen.getByRole("button", { name: "Ver minha manifestação" }));
    expect(onDone).toHaveBeenCalledWith("m9");
  });

  it("duas pessoas: escolhe e manda o citizen_id escolhido", async () => {
    const create = stub([ P1, P2 ]);
    const user = userEvent.setup();
    render(<NewManifestationStep onBack={vi.fn()} onDone={vi.fn()} onRegister={vi.fn()} />);
    expect(await screen.findByRole("heading", { level: 1, name: "Quem está fazendo a manifestação?" })).toBeInTheDocument();
    await user.click(screen.getByRole("button", { name: "Continuar" }));
    expect(screen.getByRole("alert")).toHaveTextContent("Escolha a pessoa.");
    await user.click(screen.getByRole("radio", { name: /\*\*\*\.111\.444-\*\*/ }));
    await user.click(screen.getByRole("button", { name: "Continuar" }));
    await fillAndSend(user, "Sugestão");
    await user.click(screen.getByRole("button", { name: "Enviar manifestação" }));
    expect(create).toHaveBeenCalledWith(expect.objectContaining({ citizenId: "p2", kind: "suggestion" }));
  });

  it("nenhuma pessoa: leva ao cadastro do CPF", async () => {
    stub([]);
    const onRegister = vi.fn();
    const user = userEvent.setup();
    render(<NewManifestationStep onBack={vi.fn()} onDone={vi.fn()} onRegister={onRegister} />);
    expect(await screen.findByRole("heading", { level: 1, name: "Antes, cadastre o seu CPF" })).toBeInTheDocument();
    await user.click(screen.getByRole("button", { name: "Cadastrar meu CPF" }));
    expect(onRegister).toHaveBeenCalled();
  });

  it("preservada: dica de não escrever dados pessoais", async () => {
    stub([ P1 ]);
    const user = userEvent.setup();
    render(<NewManifestationStep onBack={vi.fn()} onDone={vi.fn()} onRegister={vi.fn()} />);
    await fillAndSend(user, "Denúncia", "Com identidade preservada");
    expect(screen.getByRole("textbox", { name: "Conte o que aconteceu" }))
      .toHaveAccessibleDescription(/não escreva no texto seu nome, CPF, telefone ou endereço/);
  });

  it("denúncia com o anônimo ligado: oferece o caminho anônimo", async () => {
    stub([ P1 ]);
    const onAnonymous = vi.fn();
    const user = userEvent.setup();
    render(<NewManifestationStep onBack={vi.fn()} onDone={vi.fn()} onRegister={vi.fn()} onAnonymous={onAnonymous} />);
    expect(screen.queryByRole("button", { name: /denúncia anônima/ })).toBeNull();
    await user.click(await screen.findByRole("radio", { name: /^Denúncia/ }));
    await user.click(screen.getByRole("button", { name: "Prefere não se identificar? Faça uma denúncia anônima" }));
    expect(onAnonymous).toHaveBeenCalled();
  });

  it("dois toques: uma manifestação; recusa do api mantém o texto", async () => {
    const create = stub([ P1 ]);
    create.mockRejectedValueOnce(new ApiError(403, "not_own_citizen"));
    const user = userEvent.setup();
    render(<NewManifestationStep onBack={vi.fn()} onDone={vi.fn()} onRegister={vi.fn()} />);
    await fillAndSend(user);
    await user.click(screen.getByRole("button", { name: "Enviar manifestação" }));
    expect(await screen.findByRole("alert")).toHaveTextContent("Escolha uma das pessoas deste celular.");
    expect(screen.getByRole("textbox")).toHaveValue(TEXT);
    await user.dblClick(screen.getByRole("button", { name: "Enviar manifestação" }));
    await screen.findByRole("heading", { level: 1, name: "Manifestação registrada" });
    expect(create).toHaveBeenCalledTimes(2);
  });
});
```

- [ ] **Step 2: Run** `bin/test wpda -w wpda-mod33 src/modules/citizen/NewManifestationStep.test.tsx`. Expected: FAIL (o arquivo não existe).

- [ ] **Step 3: Implemente** `NewManifestationStep.tsx`. Cabeçalho: `// Nova manifestação (módulo 33; contrato §3, §14): uma pergunta por tela. O texto mora só no estado (nunca em storage, URL ou console).`
  - **Leitura:** `people` e `public_config`, cada um com `alive`. A falha do `public_config` dá `units = []`, e a unidade some.
  - **Estado:** `{ mode, citizenId, kind, idMode, healthUnitId, text, attachments, result }`. A pessoa única entra direto em `citizenId`.
  - **Validação** a cada "Continuar": `who` sem escolha dá "Escolha a pessoa."; `kind` sem escolha, "Escolha o tipo."; `mode` sem escolha, "Escolha como se identificar.". O erro vai no `RadioGroup` (A-18).
  - **Envio:** guarda `inFlight` e `bodyError`. O corpo é `{ citizenId, kind, mode: idMode, body: text.trim(), healthUnitId, attachmentIds: attachments.map(a => a.id) }`. O erro vem por `ombudsmanMessage`, sem limpar nada.
  - **"Voltar"** no meio dos modos não apaga o que foi preenchido.

- [ ] **Step 4: Run** o mesmo comando, mais `src/theme`, e `cd $W && npx tsc --noEmit`. Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/citizen/NewManifestationStep.tsx src/modules/citizen/NewManifestationStep.test.tsx
$G commit -m "feat: add the identified and preserved ombudsman manifestation form

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** "wpda-mod33 Task 5: nova manifestação (pessoa, tipo, modo, unidade, texto, anexos) pronta; celular sem CPF leva ao cadastro (W2, aguardando decisão)".

---

### Task 6: "Minhas manifestações" — lista e detalhe

**Files:**
- Create: `src/modules/citizen/MyManifestationsStep.tsx`, `src/modules/citizen/ManifestationDetailStep.tsx`
- Test: `src/modules/citizen/MyManifestationsStep.test.tsx` (cobre as duas)

**Interfaces:**
- Consumes: `citizenApi.manifestations`, `manifestation`, `complementManifestation`, `requestReopen`, `answerSatisfaction` (Task 1); `kindLabel`, `stateLabel`, `deadlineText`, `timelineTitle`, `bodyError`, `reasonError`, `SATISFACTION`, `ombudsmanMessage` (Task 2); `TextArea`, `AttachmentPicker` (Task 3); `Screen`, `BigButton`, `ErrorText`, `RadioGroup` (`ui.tsx`); `fmtDate`, `fmtDateTime`.
- Produces:
  - `MyManifestationsStep({ onBack, onNew, onOpen, onAnonymous }: { onBack: () => void; onNew: () => void; onOpen: (id: string) => void; onAnonymous?: () => void })`. Título "Minhas manifestações". Cada item é um `button` com `aria-label` "<Tipo> — protocolo <protocolo>" e mostra:
    - tipo, protocolo e data de recebimento;
    - o estado num selo;
    - a frase do prazo;
    - "Identidade preservada", quando é o caso;
    - "Para o CPF …", quando o api manda;
    - "novo", com `has_unread`.

    A lista vazia diz "Você ainda não fez nenhuma manifestação por aqui.". Sempre aparece a nota "Denúncias anônimas não aparecem aqui, porque não ficam ligadas a ninguém." No rodapé, "Nova manifestação" (principal), "Denúncia anônima" (secundário, com `onAnonymous`) e "Voltar".
  - `ManifestationDetailStep({ manifestationId, onBack }: { manifestationId: string; onBack: () => void })`. Título "Manifestação <protocolo>". Mostra:
    - tipo, estado, prazo e unidade;
    - "O que você escreveu" (`pre-wrap`, `overflowWrap: "anywhere"`);
    - os anexos (só os nomes);
    - "Andamento": `timeline` em lista, com `timelineTitle`, data e hora, e o `text` quando houver;
    - as ações, uma de cada vez, cada uma num `ConfirmInline` ou num formulário que abre:
      - `can_complement`: "Enviar complemento", com `TextArea` e `AttachmentPicker`;
      - `can_request_reopen`: "Pedir reabertura", com o motivo;
      - `can_answer_satisfaction`: "Responder a pesquisa", com as 3 perguntas.

    Depois de cada ação aceita, relê o detalhe e mostra `role="status"` com "Complemento enviado.", "Pedido de reabertura enviado." ou "Obrigado por responder.".

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/citizen/MyManifestationsStep.test.tsx
// Módulo 33 (contrato §3, §10, §14; spec decisões 16 e 17): lista e detalhe das manifestações.
import { describe, it, expect, vi, afterEach, beforeEach } from "vitest";
import { render, screen, within } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { MyManifestationsStep } from "./MyManifestationsStep";
import { ManifestationDetailStep } from "./ManifestationDetailStep";
import { citizenApi, ApiError, type CitizenManifestation } from "../../lib/citizenApi";

beforeEach(() => { vi.useFakeTimers({ toFake: [ "Date" ] }); vi.setSystemTime(new Date("2026-11-12T10:00:00-03:00")); });
afterEach(() => { vi.useRealTimers(); vi.restoreAllMocks(); });

const ITEM = { id: "m1", protocol: "2026-000123", kind: "complaint", identification_mode: "preserved", state: "forwarded",
  received_at: "2026-11-10T09:00:00-03:00", due_on: "2026-12-10", color: "green" as const, cpf_masked: "***.982.247-**", has_unread: true };
const DETAIL: CitizenManifestation = { id: "m1", protocol: "2026-000123", kind: "complaint", identification_mode: "identified",
  state: "awaiting_complement", received_at: "2026-11-10T09:00:00-03:00", due_on: "2026-12-10", body: "Esperei quatro horas.",
  health_unit: { id: "u1", name: "UBS Centro" }, attachments: [ { id: "a1", filename: "anexo-1.pdf" } ],
  timeline: [ { kind: "received", at: "2026-11-10T09:00:00-03:00", text: null },
    { kind: "complement_requested", at: "2026-11-11T09:00:00-03:00", text: "Em que horário foi?" } ],
  can_complement: true, can_request_reopen: false, can_answer_satisfaction: false };
const opts = () => userEvent.setup({ advanceTimers: vi.advanceTimersByTime });

describe("MyManifestationsStep", () => {
  it("lista: tipo, protocolo, Em análise (nunca encaminhada), prazo, preservada, CPF e novo", async () => {
    vi.spyOn(citizenApi, "manifestations").mockResolvedValue([ ITEM ]);
    const onOpen = vi.fn();
    const user = opts();
    render(<MyManifestationsStep onBack={vi.fn()} onNew={vi.fn()} onOpen={onOpen} />);
    const item = await screen.findByRole("button", { name: "Reclamação — protocolo 2026-000123" });
    for (const t of [ "Em análise", "Resposta prevista até 10/12/2026", "Identidade preservada", "Para o CPF ***.982.247-**", "novo" ])
      expect(within(item).getByText(t)).toBeInTheDocument();
    expect(document.body.textContent).not.toMatch(/[Ee]ncaminha/);
    await user.click(item);
    expect(onOpen).toHaveBeenCalledWith("m1");
  });

  it("vazia: frase própria, nota do anônimo e Nova manifestação", async () => {
    vi.spyOn(citizenApi, "manifestations").mockResolvedValue([]);
    const onNew = vi.fn();
    const user = opts();
    render(<MyManifestationsStep onBack={vi.fn()} onNew={onNew} onOpen={vi.fn()} />);
    expect(await screen.findByText("Você ainda não fez nenhuma manifestação por aqui.")).toBeInTheDocument();
    expect(screen.getByText(/Denúncias anônimas não aparecem aqui/)).toBeInTheDocument();
    expect(screen.queryByRole("button", { name: "Denúncia anônima" })).toBeNull();
    await user.click(screen.getByRole("button", { name: "Nova manifestação" }));
    expect(onNew).toHaveBeenCalled();
  });

  it("feature_disabled: frase da Ouvidoria, não a do histórico", async () => {
    vi.spyOn(citizenApi, "manifestations").mockRejectedValue(new ApiError(404, "feature_disabled"));
    render(<MyManifestationsStep onBack={vi.fn()} onNew={vi.fn()} onOpen={vi.fn()} />);
    expect(await screen.findByRole("alert")).toHaveTextContent("A Ouvidoria não está disponível agora nesta cidade.");
  });
});

describe("ManifestationDetailStep", () => {
  it("detalhe: texto, unidade, anexos e andamento; nada do encaminhamento", async () => {
    vi.spyOn(citizenApi, "manifestation").mockResolvedValue(DETAIL);
    render(<ManifestationDetailStep manifestationId="m1" onBack={vi.fn()} />);
    expect(await screen.findByRole("heading", { level: 1, name: "Manifestação 2026-000123" })).toBeInTheDocument();
    for (const t of [ "Esperei quatro horas.", "UBS Centro", "anexo-1.pdf", "Manifestação recebida", "A Ouvidoria pediu um complemento",
      "Em que horário foi?", "Prazo suspenso até você enviar o complemento" ]) expect(screen.getByText(t)).toBeInTheDocument();
    expect(document.body.textContent).not.toMatch(/[Ee]ncaminha|extrato|Atendimento/);
  });

  it("complemento: envia o texto aparado e relê", async () => {
    const get = vi.spyOn(citizenApi, "manifestation").mockResolvedValueOnce(DETAIL)
      .mockResolvedValueOnce({ ...DETAIL, state: "in_analysis", can_complement: false });
    const send = vi.spyOn(citizenApi, "complementManifestation").mockResolvedValue(undefined);
    const user = opts();
    render(<ManifestationDetailStep manifestationId="m1" onBack={vi.fn()} />);
    await user.click(await screen.findByRole("button", { name: "Enviar complemento" }));
    await user.click(screen.getByRole("textbox", { name: "Seu complemento" }));
    await user.paste("  Foi às 7h, na recepção da UBS Centro.  ");
    await user.click(screen.getByRole("button", { name: "Enviar o complemento" }));
    expect(send).toHaveBeenCalledWith("m1", { body: "Foi às 7h, na recepção da UBS Centro.", attachmentIds: [] });
    expect(await screen.findByRole("status")).toHaveTextContent("Complemento enviado.");
    expect(get).toHaveBeenCalledTimes(2);
    expect(screen.queryByRole("button", { name: "Enviar complemento" })).toBeNull();
  });

  it("pesquisa: as três perguntas do contrato; manda met, clear e score", async () => {
    vi.spyOn(citizenApi, "manifestation").mockResolvedValue({ ...DETAIL, state: "answered", can_complement: false, can_answer_satisfaction: true });
    const answer = vi.spyOn(citizenApi, "answerSatisfaction").mockResolvedValue(undefined);
    const user = opts();
    render(<ManifestationDetailStep manifestationId="m1" onBack={vi.fn()} />);
    await user.click(await screen.findByRole("button", { name: "Responder a pesquisa" }));
    const met = screen.getByRole("group", { name: "Sua demanda foi atendida?" });
    await user.click(within(met).getByRole("radio", { name: "Em parte" }));
    await user.click(within(screen.getByRole("group", { name: "A resposta foi fácil de entender?" })).getByRole("radio", { name: "Sim" }));
    await user.click(within(screen.getByRole("group", { name: "De 1 a 5, qual sua satisfação com a ouvidoria?" })).getByRole("radio", { name: /^4/ }));
    await user.click(screen.getByRole("button", { name: "Enviar respostas" }));
    expect(answer).toHaveBeenCalledWith("m1", { met: "partly", clear: true, score: 4 });
    expect(screen.queryByRole("textbox")).toBeNull();
  });

  it("reabertura: motivo de 10 a 2.000", async () => {
    vi.spyOn(citizenApi, "manifestation").mockResolvedValue({ ...DETAIL, state: "answered", can_complement: false, can_request_reopen: true });
    const reopen = vi.spyOn(citizenApi, "requestReopen").mockResolvedValue(undefined);
    const user = opts();
    render(<ManifestationDetailStep manifestationId="m1" onBack={vi.fn()} />);
    await user.click(await screen.findByRole("button", { name: "Pedir reabertura" }));
    await user.click(screen.getByRole("textbox", { name: "Por que você quer reabrir?" }));
    await user.paste("curto");
    await user.click(screen.getByRole("button", { name: "Enviar o pedido" }));
    expect(screen.getByRole("textbox")).toHaveAccessibleDescription(/pelo menos 10 caracteres/);
    await user.paste(" — a resposta não resolveu");
    await user.click(screen.getByRole("button", { name: "Enviar o pedido" }));
    expect(reopen).toHaveBeenCalledWith("m1", "curto — a resposta não resolveu");
  });

  it.each([
    [ "complementManifestation", "Enviar complemento", "Seu complemento", "Enviar o complemento", new ApiError(409, "manifestation_changed"), "A manifestação mudou. Veja como ela está agora." ],
    [ "requestReopen", "Pedir reabertura", "Por que você quer reabrir?", "Enviar o pedido", new ApiError(422, "reopen_window_closed"), "O prazo de 30 dias para pedir a reabertura terminou." ]
  ] as const)("409/422 nas ações: frase própria e releitura (%s)", async (method, open, field, submit, err, text) => {
    const get = vi.spyOn(citizenApi, "manifestation").mockResolvedValue({ ...DETAIL, can_request_reopen: true });
    vi.spyOn(citizenApi, method).mockRejectedValue(err);
    const user = opts();
    render(<ManifestationDetailStep manifestationId="m1" onBack={vi.fn()} />);
    await user.click(await screen.findByRole("button", { name: open }));
    await user.click(screen.getByRole("textbox", { name: field }));
    await user.paste("Texto com mais de vinte caracteres aqui.");
    await user.click(screen.getByRole("button", { name: submit }));
    expect(await screen.findByRole("alert")).toHaveTextContent(text);
    expect(get).toHaveBeenCalledTimes(2);
  });
});
```

- [ ] **Step 2: Run** `bin/test wpda -w wpda-mod33 src/modules/citizen/MyManifestationsStep.test.tsx`. Expected: FAIL (os arquivos não existem).

- [ ] **Step 3: Implemente**
  - **`MyManifestationsStep.tsx`** (molde do `MyTripsStep`): leitura com `alive`, `card` e `list` iguais. O selo do estado usa `var(--accent-bg)` em aberto e `var(--rule2)` em final. O "novo" segue o `newStyle` de `NoticesStep` (`var(--accent)`/`var(--on-accent)`).
  - **`ManifestationDetailStep.tsx`:**
    - **Leitura:** `load()` com `alive` (`useRef`, cleanup no desmonte). `deadlineText` recebe `{ state, due_on, color: null }`, porque o detalhe não traz `color` (W10).
    - **Ações:** `open: "complement" | "reopen" | "satisfaction" | null` abre uma por vez. Cada formulário tem a guarda `inFlight`.
      - **Complemento:** `bodyError` e `AttachmentPicker`, com os itens zerados a cada abertura.
      - **Reabertura:** `reasonError`.
      - **Pesquisa:** os três `RadioGroup` de `SATISFACTION`. `met` tem as opções `yes`, `partly` e `no`, com os rótulos "Sim", "Em parte" e "Não". `clear` tem "Sim" e "Não", que viram `true` e `false`. `score` tem as opções 1 a 5, com os rótulos "1 (muito insatisfeito)", "2", "3", "4" e "5 (muito satisfeito)". Sem as três respostas, aparece o erro "Responda as três perguntas." no primeiro grupo vazio.
    - **Falha da ação:** a frase vai de `ombudsmanMessage` para um `ErrorText` fora do formulário, o formulário fecha e o detalhe é relido. Vale para 409, 422 e `already_answered`. **Sucesso:** a frase de status e a releitura.
    - **Fim da tela:** "Voltar às manifestações" (`onBack`). Sem polling.

- [ ] **Step 4: Run** o mesmo comando, mais `src/theme`, e `cd $W && npx tsc --noEmit`. Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/citizen/MyManifestationsStep.tsx src/modules/citizen/ManifestationDetailStep.tsx src/modules/citizen/MyManifestationsStep.test.tsx
$G commit -m "feat: add my manifestations with complement, reopen request and satisfaction survey

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** "wpda-mod33 Task 6: 'Minhas manifestações' (lista, detalhe, complemento, reabertura, pesquisa) pronta; sem polling".

---

### Task 7: Ligar ao `Flow` — estados, rota `ouvidoria` e entradas

**Files:**
- Modify:
  - `src/lib/route.ts`: `AppRoute` e `ROUTES` ganham `"ouvidoria"`;
  - `src/lib/route.test.ts`: casos novos;
  - `src/modules/citizen/Flow.tsx`: quatro estados, `enter`, `onUnauthenticated`, a URL, o topo e os `case`;
  - `src/modules/citizen/PhoneStep.tsx`: as props `purpose`, `onCancel` e `children`;
  - `src/modules/citizen/PeopleStep.tsx`: a prop `onOmbudsman?` e o botão "Ouvidoria";
  - `README.md`.
- Test: `src/modules/citizen/Flow.ombudsman.test.tsx`

**Interfaces:**
- Consumes: as peças das Tasks 4–6; `hasSessionFeature`, `OMBUDSMAN_FEATURE`, `ANONYMOUS_WEB_FEATURE`.
- Produces:
  - `AppRoute = "avisos" | "preferencias" | "ouvidoria"`;
  - estados novos do `Flow`:
    - `{ at: "phone"; purpose?: "ombudsman" }`;
    - `{ at: "ombudsman"; consentVersion: string | null }`;
    - `{ at: "ombudsman-new"; consentVersion: string | null }`;
    - `{ at: "ombudsman-detail"; consentVersion: string | null; manifestationId: string }`;
    - `{ at: "anonymous-report"; consentVersion: string | null; session: boolean }`;
  - `PhoneStep({ onSent, purpose, onCancel, children })`. Com `purpose === "ombudsman"`, o título é "Ouvidoria", a frase é "Para fazer uma manifestação, entre com o seu celular. Vamos mandar um código por SMS." e aparece o botão secundário "Voltar" (`onCancel`). Sem `purpose`, a tela é a de hoje mais os `children` (a entrada da Ouvidoria) depois do erro;
  - `PeopleStep({ …, onOmbudsman? })`: com a prop, um `BigButton variant="secondary"` "Ouvidoria" depois de "Continuar com este CPF".

- [ ] **Step 1: Write the failing tests**

Em `src/lib/route.test.ts`, acrescente `[ "/wpda/ouvidoria", "/wpda/", "ouvidoria" ]` ao primeiro `it.each` e `expect(pathFor("ouvidoria", "/wpda/")).toBe("/wpda/ouvidoria")` ao `pathFor`.

```tsx
// src/modules/citizen/Flow.ombudsman.test.tsx
// Módulo 33 (contrato §14): a Ouvidoria na máquina de telas, sem e com sessão.
import { describe, it, expect, vi, afterEach, beforeEach } from "vitest";
import { render, screen, within } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { Flow } from "./Flow";
import { citizenApi, ApiError } from "../../lib/citizenApi";
import { setSessionFeatures } from "../../lib/sessionFeatures";

beforeEach(() => {
  vi.useFakeTimers({ toFake: [ "Date" ] });
  vi.setSystemTime(new Date("2026-11-12T10:00:00-03:00"));
  vi.stubEnv("BASE_URL", "/wpda/");
});
afterEach(() => { vi.useRealTimers(); vi.unstubAllEnvs(); vi.restoreAllMocks(); setSessionFeatures(null); window.history.replaceState(null, "", "/"); });

const PERSON = { id: "p1", cpf_masked: "***.982.247-**", verification_level: "declared" as const, neighborhood: null, profile: null };

function api({ session, features = [ "ombudsman", "ombudsman_anonymous_web" ], anonymous = true }:
  { session: boolean; features?: string[]; anonymous?: boolean }) {
  const sessionOk = async () => { setSessionFeatures(features); return { phone_masked: "(**) *****-5432", features }; };
  if (session) vi.spyOn(citizenApi, "currentSession").mockImplementation(sessionOk);
  else vi.spyOn(citizenApi, "currentSession").mockRejectedValue(new ApiError(401, "unauthenticated"));
  vi.spyOn(citizenApi, "requestCode").mockResolvedValue({ status: "sent", resend_after: 60 });
  vi.spyOn(citizenApi, "verifyCode").mockImplementation(async () => sessionOk());
  vi.spyOn(citizenApi, "consentTerm").mockResolvedValue({ version: "1", body: "Termo" });
  vi.spyOn(citizenApi, "people").mockResolvedValue({ people: [ PERSON ] });
  vi.spyOn(citizenApi, "neighborhoods").mockResolvedValue([]);
  const notices = vi.spyOn(citizenApi, "notices").mockResolvedValue({ notices: [], unread_count: 0 });
  vi.spyOn(citizenApi, "ombudsmanPublicConfig").mockResolvedValue({ anonymous_web: anonymous, units: [] });
  const list = vi.spyOn(citizenApi, "manifestations").mockResolvedValue([]);
  return { notices, list };
}

describe("Flow — Ouvidoria (módulo 33)", () => {
  it("sem sessão: tela inicial com a Ouvidoria; anônima sem o topo da conta; Voltar ao início", async () => {
    const { notices } = api({ session: false });
    const user = userEvent.setup({ advanceTimers: vi.advanceTimersByTime });
    render(<Flow />);
    await screen.findByRole("heading", { level: 1, name: "Triagem de saúde" });
    expect(await screen.findByRole("heading", { level: 2, name: "Ouvidoria" })).toBeInTheDocument();
    await user.click(screen.getByRole("button", { name: "Denúncia anônima" }));
    expect(await screen.findByRole("heading", { level: 1, name: "Denúncia anônima" })).toBeInTheDocument();
    expect(screen.queryByRole("navigation", { name: "Conta" })).toBeNull();
    expect(notices).not.toHaveBeenCalled();
    await user.click(screen.getByRole("button", { name: "Voltar" }));
    expect(await screen.findByRole("heading", { level: 1, name: "Triagem de saúde" })).toBeInTheDocument();
  });

  it("Fazer uma manifestação: entra com o celular e cai na Ouvidoria, sem o termo da triagem", async () => {
    const { list } = api({ session: false });
    const user = userEvent.setup({ advanceTimers: vi.advanceTimersByTime });
    render(<Flow />);
    await user.click(await screen.findByRole("button", { name: "Fazer uma manifestação" }));
    expect(await screen.findByRole("heading", { level: 1, name: "Ouvidoria" })).toBeInTheDocument();
    await user.type(screen.getByLabelText("Seu celular"), "41998765432");
    await user.click(screen.getByRole("button", { name: "Receber código" }));
    await user.type(await screen.findByLabelText("Código"), "123456");
    await user.click(screen.getByRole("button", { name: "Confirmar" }));
    expect(await screen.findByRole("heading", { level: 1, name: "Minhas manifestações" })).toBeInTheDocument();
    expect(citizenApi.consentTerm).not.toHaveBeenCalled();
    expect(list).toHaveBeenCalled();
    expect(window.location.pathname).toBe("/wpda/ouvidoria");
  });

  it("com sessão: Para quem é → Ouvidoria → Nova manifestação → Voltar", async () => {
    api({ session: true });
    const user = userEvent.setup({ advanceTimers: vi.advanceTimersByTime });
    render(<Flow />);
    await user.click(await screen.findByRole("button", { name: "Concordo" }));
    await user.click(await screen.findByRole("button", { name: "Ouvidoria" }));
    await screen.findByRole("heading", { level: 1, name: "Minhas manifestações" });
    expect(within(screen.getByRole("navigation", { name: "Conta" })).getByRole("button", { name: "Sair" })).toBeInTheDocument();
    await user.click(screen.getByRole("button", { name: "Nova manifestação" }));
    expect(await screen.findByRole("heading", { level: 1, name: "Que tipo de manifestação?" })).toBeInTheDocument();
    await user.click(screen.getByRole("button", { name: "Voltar" }));
    expect(await screen.findByRole("heading", { level: 1, name: "Minhas manifestações" })).toBeInTheDocument();
    await user.click(screen.getByRole("button", { name: "Voltar" }));
    expect(await screen.findByRole("heading", { level: 1, name: "Para quem é esta triagem?" })).toBeInTheDocument();
  });

  it("sem ombudsman na sessão: Para quem é igual", async () => {
    api({ session: true, features: [ "patient_timeline" ] });
    const user = userEvent.setup({ advanceTimers: vi.advanceTimersByTime });
    render(<Flow />);
    await user.click(await screen.findByRole("button", { name: "Concordo" }));
    await screen.findByRole("heading", { level: 1, name: "Para quem é esta triagem?" });
    expect(screen.queryByRole("button", { name: "Ouvidoria" })).toBeNull();
  });

  it("rota /wpda/ouvidoria com sessão: abre direto; sem anonymous na sessão, não há Denúncia anônima", async () => {
    window.history.replaceState(null, "", "/wpda/ouvidoria");
    api({ session: true, features: [ "ombudsman" ] });
    render(<Flow />);
    expect(await screen.findByRole("heading", { level: 1, name: "Minhas manifestações" })).toBeInTheDocument();
    expect(screen.queryByRole("button", { name: "Denúncia anônima" })).toBeNull();
  });

  it("sessão cai na Ouvidoria: volta ao celular e, ao entrar, à Ouvidoria", async () => {
    const { list } = api({ session: true });
    list.mockRejectedValueOnce(new ApiError(401, "unauthenticated"));
    window.history.replaceState(null, "", "/wpda/ouvidoria");
    render(<Flow />);
    window.dispatchEvent(new Event("citizen:unauthenticated"));
    expect(await screen.findByRole("heading", { level: 1, name: "Triagem de saúde" })).toBeInTheDocument();
    const user = userEvent.setup({ advanceTimers: vi.advanceTimersByTime });
    await user.type(screen.getByLabelText("Seu celular"), "41998765432");
    await user.click(screen.getByRole("button", { name: "Receber código" }));
    await user.type(await screen.findByLabelText("Código"), "123456");
    await user.click(screen.getByRole("button", { name: "Confirmar" }));
    expect(await screen.findByRole("heading", { level: 1, name: "Minhas manifestações" })).toBeInTheDocument();
  });
});
```

- [ ] **Step 2: Run** `bin/test wpda -w wpda-mod33 src/lib/route.test.ts src/modules/citizen/Flow.ombudsman.test.tsx`. Expected: FAIL.

- [ ] **Step 3: Implemente**
  - **`route.ts`:** `"ouvidoria"` em `AppRoute` e em `ROUTES`, e o comentário de cabeçalho cita o módulo 33.
  - **`Flow.tsx`:**
    - **Estados:** os quatro novos, e `phone` ganha `purpose?`.
    - **`enter()`:** `route === "ouvidoria"` vai para `{ at: "ombudsman", consentVersion: null }`, sem passar pelo termo (W1).
    - **`onUnauthenticated`:** com `s.at` em `ombudsman`, `ombudsman-new` ou `ombudsman-detail`, `target.current = "ouvidoria"`.
    - **Efeito da URL:** `ombudsman`, `ombudsman-new` e `ombudsman-detail` dão a rota `"ouvidoria"`. O id da manifestação **não** vai à URL; um F5 no detalhe volta à lista. `anonymous-report` com `session: false` fica fora do efeito, como `phone`.
    - **`signedIn`:** passa a ser `state.at === "anonymous-report" ? state.session : state.at !== "boot" && state.at !== "phone" && state.at !== "code"`. Sem sessão, o topo (`NoticesLink`) não pede `/citizen/notices`, e um 401 não derruba a tela.
    - **`const ombudsman = hasSessionFeature(OMBUDSMAN_FEATURE)`** e **`const anonymousWeb = hasSessionFeature(ANONYMOUS_WEB_FEATURE)`**, lidos no render, como os outros interruptores.
    - **`case "phone"`:** `<PhoneStep onSent=… purpose={state.purpose} onCancel={() => { target.current = null; setState({ at: "phone" }); }}>`. Os `children` só existem sem `purpose`: `<OmbudsmanEntry onManifest={() => { target.current = "ouvidoria"; setState({ at: "phone", purpose: "ombudsman" }); }} onAnonymous={() => setState({ at: "anonymous-report", consentVersion: null, session: false })} />`.
    - **`case "people"`:** `onOmbudsman={ombudsman ? () => setState({ at: "ombudsman", consentVersion: state.consentVersion }) : undefined}`.
    - **`case "declined"`:** com `ombudsman`, um `BigButton variant="secondary"` "Ouvidoria" no rodapé, depois de "Ler o termo de novo". Quem recusa a triagem ainda pode falar com a Ouvidoria.
    - **`case "ombudsman"`:** `<MyManifestationsStep>`, com:
      - `onNew`, que vai para `ombudsman-new`;
      - `onOpen`, que vai para `ombudsman-detail`;
      - `onAnonymous` só com `anonymousWeb`, que vai para `anonymous-report` com `session: true`;
      - `onBack`, que vai para `people` com a versão, ou para `consent` sem ela (como em `notices`).
    - **`case "ombudsman-new"`:** `<NewManifestationStep>`, com:
      - `onBack`, que vai para `ombudsman`;
      - `onDone(id)`, que vai para `ombudsman-detail`;
      - `onRegister`, que vai para `consent`, o caminho do cadastro do CPF (W2);
      - `onAnonymous` com `anonymousWeb`.
    - **`case "ombudsman-detail"`:** `<ManifestationDetailStep key={state.manifestationId} …>`, com `onBack` para `ombudsman`.
    - **`case "anonymous-report"`:** `<AnonymousReportStep>`, com `onBack` que volta a `ombudsman` (com sessão) ou a `{ at: "phone" }` (sem sessão).
  - **`PhoneStep.tsx`:** as props novas, opcionais, e nada mais muda.
  - **`PeopleStep.tsx`:** a prop opcional e o botão.
  - **`README.md`:** o parágrafo **Ouvidoria (módulo 33).**, depois do de Urgência, diz que:
    - com `ombudsman` ligado, a tela inicial mostra "Ouvidoria", com "Fazer uma manifestação" (entra com o celular e vai direto à Ouvidoria, sem o termo da triagem) e, com `ombudsman_anonymous_web`, "Denúncia anônima";
    - a denúncia anônima não pede login, não leva o cookie (`publicCall`, `credentials: "omit"`), não tem anexo nem protocolo, e avisa disso antes do envio;
    - a manifestação identificada ou preservada escolhe a pessoa do celular, o tipo, o modo, a unidade, o texto e até 3 anexos com nome genérico;
    - "Minhas manifestações" mostra o protocolo, o estado ("Em análise" também para o que foi encaminhado), o prazo, as respostas, o complemento, o pedido de reabertura e a pesquisa;
    - o endereço `<base>ouvidoria` abre a Ouvidoria depois do login;
    - tudo segue o bloqueio por dispositivo da decisão 9.

- [ ] **Step 4: Run** o mesmo comando, mais `src/modules/citizen src/lib src/theme`, e `cd $W && npx tsc --noEmit`. Expected: PASS. `Flow.route.test.tsx`, `Flow.notices.test.tsx`, `Flow.test.tsx` e `PeopleStep.test.tsx` continuam iguais. Se um teste antigo falhar porque a tela inicial agora pede `public_config`, o motivo é o `fetch` sem dublê: acrescente o spy `vi.spyOn(citizenApi, "ombudsmanPublicConfig").mockRejectedValue(new ApiError(404, "feature_disabled"))` no helper do teste, e não mude a tela.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/route.ts src/lib/route.test.ts src/modules/citizen/Flow.tsx src/modules/citizen/PhoneStep.tsx src/modules/citizen/PeopleStep.tsx src/modules/citizen/Flow.ombudsman.test.tsx README.md
$G commit -m "feat: wire the ombudsman screens into the citizen flow

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

(Se a Step 4 mexeu em helpers de testes antigos, adicione cada arquivo pelo nome no mesmo `add`.)

**Relato ao coordenador:** "wpda-mod33 Task 7: Ouvidoria no Flow (tela inicial, `/wpda/ouvidoria`, Para quem é, recusa do termo); sem depender do termo da triagem (W1)".

---

### Task 8: Aviso de ouvidoria na caixa — `NoticesStep`

**Files:**
- Modify: `src/modules/citizen/NoticesStep.tsx` (item e detalhe do aviso `ombudsman`; prop `onOmbudsman?`), `src/modules/citizen/Flow.tsx` (passa `onOmbudsman` no `case "notices"`)
- Test: `src/modules/citizen/NoticesStep.ombudsman.test.tsx`

**Interfaces:**
- Consumes: `OmbudsmanNotice` (Task 1); `noticeTitle`, `OMBUDSMAN_NOTICE_HINT` (Task 2).
- Produces: `NoticesStep({ onBack, onPreferences, onRead, onOmbudsman })`.
  - O item `ombudsman` mostra o título (`noticeTitle(event)`), o selo "novo", "Protocolo <protocolo>" (quando houver), a data (`fmtDate(created_at)`) e "Para o CPF …" (quando houver).
  - O detalhe `OmbudsmanDetail` mostra o mesmo, mais `OMBUDSMAN_NOTICE_HINT` ('Os detalhes estão em "Minhas manifestações", na Ouvidoria.') e, com `onOmbudsman`, o botão "Ver minhas manifestações".
  - O aviso não tem texto (contrato §3): nada além disso aparece.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/citizen/NoticesStep.ombudsman.test.tsx
import { describe, it, expect, vi, afterEach, beforeEach } from "vitest";
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { NoticesStep } from "./NoticesStep";
import { citizenApi, type Notice } from "../../lib/citizenApi";

beforeEach(() => { vi.useFakeTimers({ toFake: [ "Date" ] }); vi.setSystemTime(new Date("2026-11-12T10:00:00-03:00")); });
afterEach(() => { vi.useRealTimers(); vi.restoreAllMocks(); });

const N: Notice = { kind: "ombudsman", id: "o1", event: "final_response", protocol: "2026-000123",
  created_at: "2026-11-11T14:00:00-03:00", read: false, cpf_masked: "***.982.247-**" };

describe("NoticesStep — aviso da Ouvidoria (módulo 33)", () => {
  it("item: título do evento, protocolo, data, CPF e novo; abre, marca lido e leva às manifestações", async () => {
    vi.spyOn(citizenApi, "notices").mockResolvedValue({ notices: [ N, { ...N, id: "o2", event: "xpto", protocol: null, read: true, cpf_masked: null } ], unread_count: 1 });
    const read = vi.spyOn(citizenApi, "readNotice").mockResolvedValue({ ok: true });
    const onOmbudsman = vi.fn();
    const user = userEvent.setup({ advanceTimers: vi.advanceTimersByTime });
    render(<NoticesStep onBack={vi.fn()} onOmbudsman={onOmbudsman} />);
    const item = await screen.findByRole("button", { name: /Sua manifestação foi respondida/ });
    expect(item).toHaveTextContent("Protocolo 2026-000123");
    expect(item).toHaveTextContent("11/11/2026");
    expect(item).toHaveTextContent("Para o CPF ***.982.247-**");
    expect(item).toHaveTextContent("novo");
    expect(screen.getByRole("button", { name: /Aviso da Ouvidoria/ })).not.toHaveTextContent("Protocolo");
    await user.click(item);
    expect(await screen.findByRole("heading", { level: 1, name: "Sua manifestação foi respondida" })).toBeInTheDocument();
    expect(screen.getByText('Os detalhes estão em "Minhas manifestações", na Ouvidoria.')).toBeInTheDocument();
    expect(read).toHaveBeenCalledWith("o1");
    await user.click(screen.getByRole("button", { name: "Ver minhas manifestações" }));
    expect(onOmbudsman).toHaveBeenCalled();
    expect(document.body.textContent).not.toMatch(/undefined|null/);
  });

  it("sem onOmbudsman (interruptor desligado): sem o botão", async () => {
    vi.spyOn(citizenApi, "notices").mockResolvedValue({ notices: [ N ], unread_count: 1 });
    vi.spyOn(citizenApi, "readNotice").mockResolvedValue({ ok: true });
    const user = userEvent.setup({ advanceTimers: vi.advanceTimersByTime });
    render(<NoticesStep onBack={vi.fn()} />);
    await user.click(await screen.findByRole("button", { name: /Sua manifestação foi respondida/ }));
    expect(screen.queryByRole("button", { name: "Ver minhas manifestações" })).toBeNull();
  });
});
```

- [ ] **Step 2: Run** `bin/test wpda -w wpda-mod33 src/modules/citizen/NoticesStep.ombudsman.test.tsx`. Expected: FAIL (o item cai no ramo de campanha e o título fica vazio).

- [ ] **Step 3: Implemente**
  - **`NoticesStep.tsx`:**
    - o cabeçalho cita o módulo 33;
    - em `ItemContent`, um ramo `notice.kind === "ombudsman"` depois do de `trip`, no mesmo desenho;
    - `OmbudsmanDetail`, no molde de `TripDetail`;
    - no componente, `if (current && current.kind === "ombudsman") return <OmbudsmanDetail … onOmbudsman={onOmbudsman} />`, antes do ramo de campanha.
  - **`Flow.tsx`:** no `case "notices"`, `onOmbudsman={hasSessionFeature(OMBUDSMAN_FEATURE) ? () => setState({ at: "ombudsman", consentVersion: state.consentVersion }) : undefined}`.

- [ ] **Step 4: Run** o mesmo comando, mais `src/modules/citizen/NoticesStep.test.tsx src/modules/citizen/NoticesStep.trips.test.tsx src/modules/citizen/NoticesStep.caps.test.tsx src/modules/citizen/Flow.notices.test.tsx`, e `cd $W && npx tsc --noEmit`. Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/citizen/NoticesStep.tsx src/modules/citizen/NoticesStep.ombudsman.test.tsx src/modules/citizen/Flow.tsx
$G commit -m "feat: show ombudsman notices in the citizen inbox

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** "wpda-mod33 Task 8: aviso de ouvidoria na caixa (título do evento, protocolo, sem texto); aguardo `cpf_masked` no aviso (W3)".

---

### Task 9: Playwright — formulário anônimo nos celulares

**Files:**
- Create: `e2e/ombudsman.spec.ts`

**Interfaces:**
- Consumes: o Vite do worktree na 5217; `playwright.config.ts` (5 perfis, `WPDA_URL`); o molde de `e2e/trips.spec.ts` (`PHONES`, `noOverflow`, `timezoneId: "UTC"`, rotas `/citizen/*` respondidas).
- Produces: a prova, no aparelho, de que a denúncia anônima sai sem login, sem cookie, sem chave a mais e sem protocolo na tela.

- [ ] **Step 1: Suba o Vite do worktree** (fica no ar até a Task 10)

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker run -d --rm --name wpda-mod33 --network rota-saude_default -p 5217:5217 \
  -v $R/.worktrees/wpda-mod33:/app -v wpda-mod33-node-modules:/app/node_modules -w /app \
  -e VITE_API_PROXY_TARGET=http://api:3056 -e WPDA_ALLOW_DESKTOP=1 node:22-alpine \
  sh -c "npm ci --silent --no-audit --no-fund && npx vite --port 5217 --host 0.0.0.0"
docker logs -f wpda-mod33   # espere o "ready"
```

- [ ] **Step 2: Escreva o teste**

```ts
// e2e/ombudsman.spec.ts
// Módulo 33 nos perfis de celular (contrato §4, §14). As rotas /citizen/* são respondidas aqui, sem api.
// Prova a denúncia anônima no aparelho: sem login, o aviso antes do envio, o corpo exato, nenhum cookie
// (mesmo com um citizen_session plantado), nenhum protocolo na tela e sem rolagem lateral.
import { expect, test, type Page, type Request } from "@playwright/test";

test.use({ timezoneId: "UTC" });

const PHONES = new Set([ "iphone", "pixel" ]);
const TEXT = "Na UBS Centro cobraram R$ 50 pela consulta, o que é proibido no SUS.";
const WARNING = "Sem identificação, você não recebe protocolo nem acompanha a resposta.";

async function mockApi(page: Page, opts: { anonymous: boolean; configStatus?: number }) {
  const reports: Request[] = [];
  await page.route((url) => url.pathname.startsWith("/citizen/"), async (route) => {
    const path = new URL(route.request().url()).pathname.replace(/^\/citizen/, "");
    const json = (body: unknown, status = 200) => route.fulfill({ status, contentType: "application/json", body: JSON.stringify(body) });
    switch (path) {
      case "/session": return json({ error: "unauthenticated" }, 401);
      case "/ombudsman/public_config":
        return opts.configStatus === 404 ? json({ error: "feature_disabled" }, 404)
          : json({ anonymous_web: opts.anonymous, units: [ { id: "u1", name: "UBS Centro" } ] });
      case "/ombudsman/anonymous_reports":
        reports.push(route.request());
        return json({ ok: true, protocol: "2026-000999" }, 201);
      default: return json({ error: "not_found" }, 404);
    }
  });
  return reports;
}

const noOverflow = (page: Page) => page.evaluate(() => document.documentElement.scrollWidth - document.documentElement.clientWidth);

test.beforeEach(async ({ page }, info) => {
  test.skip(!PHONES.has(info.project.name), "esta prova roda nos dois celulares (iphone e pixel)");
  await page.clock.setFixedTime(new Date("2026-11-12T10:00:00-03:00"));
});

test("Denúncia anônima: sem login, aviso antes do envio, sem cookie, sem protocolo, sem rolagem lateral", async ({ page, context, baseURL }) => {
  await context.addCookies([ { name: "citizen_session", value: "plantado", url: baseURL as string } ]);
  const reports = await mockApi(page, { anonymous: true });
  await page.goto("/wpda/");
  await expect(page.getByRole("heading", { level: 2, name: "Ouvidoria" })).toBeVisible();
  await page.getByRole("button", { name: "Denúncia anônima", exact: true }).click();
  await expect(page.getByRole("heading", { level: 1, name: "Denúncia anônima" })).toBeVisible();
  await expect(page.getByRole("navigation", { name: "Conta" })).toHaveCount(0);

  await page.getByRole("combobox", { name: "Unidade de saúde (opcional)" }).selectOption("u1");
  await page.getByRole("textbox", { name: "O que você quer denunciar?" }).fill(TEXT);
  const send = page.getByRole("button", { name: "Enviar denúncia" });
  await send.scrollIntoViewIfNeeded();
  await expect(page.getByText(WARNING, { exact: true }).last()).toBeInViewport();
  expect(await noOverflow(page)).toBeLessThanOrEqual(0);
  await send.click();

  await expect(page.getByRole("heading", { level: 1, name: "Denúncia enviada" })).toBeVisible();
  expect(reports).toHaveLength(1);
  const req = reports[0];
  expect(req.method()).toBe("POST");
  expect(new URL(req.url()).search).toBe("");
  expect(req.postDataJSON()).toEqual({ body: TEXT, health_unit_id: "u1" });
  expect((await req.allHeaders())["cookie"]).toBeUndefined();
  expect(await page.locator("body").innerText()).not.toMatch(/2026-000999|[Pp]rotocolo \d/);
  expect(await noOverflow(page)).toBeLessThanOrEqual(0);
});

test("Texto curto: erro no campo e nada enviado", async ({ page }) => {
  const reports = await mockApi(page, { anonymous: true });
  await page.goto("/wpda/");
  await page.getByRole("button", { name: "Denúncia anônima", exact: true }).click();
  await page.getByRole("textbox", { name: "O que você quer denunciar?" }).fill("curto");
  await page.getByRole("button", { name: "Enviar denúncia" }).click();
  await expect(page.getByText("Escreva pelo menos 20 caracteres.")).toBeVisible();
  expect(reports).toHaveLength(0);
});

test("Sem anonymous_web: só a manifestação; public_config 404: nada da Ouvidoria", async ({ page }) => {
  await mockApi(page, { anonymous: false });
  await page.goto("/wpda/");
  await expect(page.getByRole("button", { name: "Fazer uma manifestação" })).toBeVisible();
  await expect(page.getByRole("button", { name: "Denúncia anônima", exact: true })).toHaveCount(0);

  await page.unrouteAll({ behavior: "ignoreErrors" });
  await mockApi(page, { anonymous: true, configStatus: 404 });
  await page.reload();
  await expect(page.getByRole("heading", { level: 1, name: "Triagem de saúde" })).toBeVisible();
  await expect(page.getByRole("heading", { level: 2, name: "Ouvidoria" })).toHaveCount(0);
});
```

- [ ] **Step 3: Run** `cd $W && WPDA_URL=http://curitiba.localhost:5217 npx playwright test e2e/ombudsman.spec.ts e2e/device-gate.spec.ts`. Expected:
  - PASS nos perfis `iphone` e `pixel`; os outros pulam com o motivo;
  - device-gate verde nos 5 perfis.

  Se `allHeaders()["cookie"]` vier preenchido, o `publicCall` não está com `credentials: "omit"`: corrija a Task 1, não o teste.

- [ ] **Step 4: Commit**

```bash
cd $W && $G add e2e/ombudsman.spec.ts
$G commit -m "test: cover the anonymous ombudsman report on phone profiles

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** "wpda-mod33 Task 9: e2e da denúncia anônima verde no iphone e no pixel (sem cookie, sem protocolo, sem rolagem); device-gate verde".

---

### Task 10: Suíte, build, e2e, revisão e prova no navegador

**Files:** nenhum novo.

- [ ] **Step 1: Suíte, tipos, build e e2e**

Run: `bin/test wpda -w wpda-mod33 && cd $W && npx tsc --noEmit && npm run build && WPDA_URL=http://curitiba.localhost:5217 npx playwright test && $G status --short && $G log --stat origin/main..HEAD | grep -c node_modules`

Expected:
- verde, com a base mais 8 arquivos de teste e os casos novos de `route.test.ts`;
- e2e verde;
- `status` vazio;
- contagem `0`.

Se o número de testes dobrar, é artefato de build: pare e reporte.

- [ ] **Step 2: Nada vaza**

```bash
cd $W
grep -n "console\.\|localStorage\|sessionStorage" src/lib/ombudsman.ts src/modules/citizen/OmbudsmanFields.tsx src/modules/citizen/OmbudsmanEntry.tsx \
  src/modules/citizen/AnonymousReportStep.tsx src/modules/citizen/NewManifestationStep.tsx src/modules/citizen/MyManifestationsStep.tsx \
  src/modules/citizen/ManifestationDetailStep.tsx
grep -n "setTimeout\|setInterval" src/modules/citizen/*Manifestation*.tsx src/modules/citizen/AnonymousReportStep.tsx src/modules/citizen/OmbudsmanFields.tsx
grep -n "file\.name" src/modules/citizen/OmbudsmanFields.tsx src/lib/ombudsman.ts
grep -n '"/ombudsman/[^"]*?' src/lib/citizenApi.ts
```

Expected: sem saída nos quatro. Sem console, sem storage, sem timer, sem o nome do arquivo do aparelho e sem query string.

- [ ] **Step 3: Revisão.** Um subagente revisor lê `origin/main..HEAD` contra a spec (decisões 2, 3, 5, 6, 12, 16, 17 e 19; §5; F-33.2, F-33.3, F-33.8) e o contrato (§3, §4, §9, §10, §14). Ele confere:
  - a denúncia anônima: `credentials: "omit"`, corpo exato, aviso antes do envio, sem protocolo, sem anexo;
  - `forwarded` como "Em análise" e nada do encaminhamento;
  - o nome do anexo genérico;
  - o `citizen_id` sempre de `people()`;
  - as guardas `inFlight` e `alive`;
  - o bloqueio por dispositivo intacto (a denúncia mora no `Flow`; `App.tsx` não mudou);
  - cores só por papel do tema;
  - os rótulos da pesquisa iguais aos do §10.

- [ ] **Step 4: Prova no navegador (com o usuário)**

Depende do api do 33 na **3056**, com a semente do plano do api:
- Curitiba com `ombudsman` e `ombudsman_anonymous_web` ligados, e duas unidades;
- o celular de dev com **duas pessoas** (uma validada, uma só declarada);
- um `ombudsman_staff` e um `ombudsman` no dashboard (5216), para responder.

O Vite da Task 9 segue no ar. Abra `http://curitiba.localhost:5217/wpda/` a 375 px, com toque. Guarde os screenshots em `evidencias/2026-11-11-modulo-33/wpda/`. Roteiro:
1. **Sem login:** a tela inicial mostra "Ouvidoria". Faça uma "Denúncia anônima" com unidade: aparece o aviso antes do envio e, depois, "Denúncia enviada" sem protocolo. Na rede (`read_network_requests`), o POST `/citizen/ombudsman/anonymous_reports` vai sem `Cookie` e com o corpo `{ body, health_unit_id }`. No dashboard, a denúncia aparece como `anonymous`, `report`, `web`.
2. **"Fazer uma manifestação":** entre com o celular e o código SMS de dev. A tela cai em "Minhas manifestações", sem o termo. Faça uma **identificada** (pessoa validada, reclamação, unidade, um PDF) e anote o protocolo e o prazo. Faça uma **preservada** (pessoa declarada, denúncia, uma foto). No dashboard, o anexo da preservada se chama `anexo-1.jpg` e a identidade aparece como "preservada".
3. **Complemento:** no dashboard, o técnico pede um complemento na identificada. No wpda, a caixa de avisos mostra "A Ouvidoria pediu um complemento", com o protocolo e sem o texto. "Ver minhas manifestações" leva à lista. O detalhe mostra "Prazo suspenso…" e o pedido. Envie o complemento: o detalhe volta a "Em análise".
4. **Encaminhamento:** o técnico encaminha a um setor. No wpda, o estado continua "Em análise", e nenhuma resposta `/citizen/ombudsman/*` contém `extract`, `forward` nem o nome do setor.
5. **Prorrogação e resposta conclusiva:** o ouvidor prorroga e responde. A linha do tempo mostra "Prazo prorrogado" com a justificativa, e "Resposta da Ouvidoria". O aviso é "Sua manifestação foi respondida".
6. **Pesquisa:** responda Em parte, Sim, 4. Ao abrir de novo, a pesquisa não aparece mais. **Reabertura:** peça a reabertura com motivo; o ouvidor recusa, e o aviso é "Pedido de reabertura respondido".
7. **Desligar o interruptor:** no maintenance (5177, com o serviço do compose parado), desligue `ombudsman_anonymous_web`. A tela inicial perde "Denúncia anônima". Desligue `ombudsman`, saia e entre: nenhuma entrada da Ouvidoria aparece, e `/wpda/ouvidoria` diz "A Ouvidoria não está disponível agora nesta cidade.". Religue os dois.
8. Alto contraste legível; a 375 px, nenhuma tela com rolagem lateral.

Ao terminar: `docker rm -f wpda-mod33 && docker volume rm wpda-mod33-node-modules`.

- [ ] **Step 5: Cards.** Se as partes do api de F-33.2, F-33.3 e F-33.8 estiverem **Done**, mova os três para **Done** (`98236657`), e para **Verified** se a prova passou. Senão, deixe como está e avise a sessão do api.

- [ ] **Step 6: Pare.** O merge só vem depois do api e do dashboard do 33, e só com autorização explícita (merge, push e cards, uma etapa de cada vez). Antes do push, `origin/main..main` só pode ter os commits desta entrega. Quando autorizado: `fetch`, `merge --ff-only feat/mod-33-ombudsman`, `push origin main`, `worktree remove $W`, `branch -d`. O `wpda-dev` (5176) recompila sozinho.

**Relato ao coordenador:** "wpda-mod33 Task 10: suíte <N>/0, build e e2e verdes; prova <passou/pendente: …>; cards F-33.2/.3/.8 em <estado>; pronto para merge depois do api e do dashboard".

---

## Divergências propostas ao contrato

1. **W1 — A Ouvidoria depende do termo da triagem?** O contrato não diz. O `Flow` de hoje leva todo login ao termo da triagem (`enter()` → `consent`), menos a caixa de avisos (ADR 0024; `Flow.tsx:86-92`). A spec §5 diz que a base legal é a obrigação legal, não o consentimento.
   - Proposta: as rotas do §3 **não** conferem consentimento (nada de `no_consent`/`consent_outdated`), e o wpda vai à Ouvidoria sem o termo (Task 7).
   - O plano do api ganha um request spec com uma sessão sem consentimento.
2. **W2 — Celular sem nenhuma pessoa cadastrada.** O §3 exige `citizen_id` entre os da sessão. Só que o par nasce em `POST /citizen/people`, que exige `consent_version` (o termo da triagem; `people_controller.rb:21`). Quem só quer reclamar teria de aceitar o termo da triagem para cadastrar o CPF.
   - O plano implementa o caminho que já existe: "Antes, cadastre o seu CPF" leva ao termo e a "Para quem é" (Task 5, `onRegister`).
   - Alternativa: `POST /citizen/ombudsman/manifestations` aceita `cpf` no lugar de `citizen_id` e cria o par sem o termo da triagem, pela mesma base legal do W1. É mais justo com o cidadão, mas mexe em `Citizens::RegisterPerson` e na regra "nenhum CPF sem o termo".
   - O coordenador decide; o wpda muda só o `onRegister`.
3. **W3 — `cpf_masked` no aviso de ouvidoria.** O §3 define o aviso como `{ id, kind: "ombudsman", event, protocol, created_at, read }`. As outras três fontes mandam `cpf_masked` quando o telefone tem mais de uma pessoa (`notices_controller.rb:13`), e é assim que a caixa diz de quem é o aviso.
   - Proposta: o aviso de ouvidoria leva `cpf_masked` pela mesma regra.
   - O wpda já normaliza e mostra o campo (Tasks 1 e 8).
4. **W4 — Códigos da linha do tempo do cidadão.** O §3 lista os eventos visíveis em palavras, mas não fixa os `kind`.
   - Proposta: `received`, `complement_requested`, `complement_sent`, `extended`, `interim_response`, `final_response`, `archived`, `reopen_requested`, `reopen_decided`, `reopened`.
   - O `text` vem pronto: no `archived`, o rótulo do motivo já em palavras do cidadão (o wpda não traduz código, como no 32a W3); no `extended`, a justificativa.
   - O wpda dá título neutro ("Atualização da Ouvidoria") a `kind` desconhecido (Task 2).
5. **W5 — `forwarded` na lista do cidadão.** O §3 proíbe mostrar o encaminhamento no detalhe, mas a lista e o detalhe devolvem `state`, que pode ser `forwarded`. "Encaminhada" conta ao cidadão que um setor ou unidade foi acionado, e qual é o assunto, se ele souber ler.
   - Proposta: a visão do cidadão devolve `in_analysis` no lugar de `forwarded`.
   - O wpda já mostra os dois como "Em análise" (Task 2, defesa).
6. **W6 — Códigos de erro de validação.** O §3 e o §4 fixam `identification_mode`, `not_own_citizen`, `already_answered` e `reopen_window_closed`, mas não fixam os códigos de: texto fora do tamanho, `kind` inválido, unidade inexistente ou inativa, anexo inválido ou de outro dono, anexos demais, complemento fora de `awaiting_complement`, reabertura já pedida, pesquisa fora dos 30 dias. Também não fixam o tamanho do complemento.
   - Proposta: `invalid_body`, `invalid_kind`, `invalid_health_unit`, `invalid_file` (o de `Exams::Files`), `too_many_attachments`, 409 `manifestation_changed` (o padrão do §2), 409 `already_requested`, 422 `satisfaction_window_closed`.
   - O complemento segue o `body`: 20 a 4.000.
   - O wpda mapeia esses códigos e os nomes de campo (`body`, `reason`), e o resto cai na frase genérica (Task 2).
7. **W7 — Rotas públicas sem cookie.** O §4 diz que o endpoint não lê nem cria `citizen_session`. O wpda vai além: manda as duas rotas públicas com `credentials: "omit"`. Assim o cookie nem chega ao api, mesmo com o cidadão logado ou numa troca de rota no proxy (Task 1, provado no e2e da Task 9). Isto é um complemento, não uma troca.
   - Proposta: o §4 registra que o cliente não manda o cookie.
   - O spec do api confere as duas coisas: com o cookie presente, nada se liga à sessão e nenhum `Set-Cookie` volta.
8. **W8 — Nome do anexo.** O §3 pede `filename` no upload e o devolve no detalhe; o §5 lista os anexos à equipe. O nome do arquivo do aparelho costuma trazer o nome da pessoa ("RG João Silva.pdf"), e isso quebra a identidade preservada, porque os anexos são listados ao técnico.
   - Proposta: o §3 registra que o cliente manda um nome genérico (`anexo-<n>.<ext>`), e o wpda faz isso sempre (Task 2).
   - Defesa no api: nas manifestações `preserved` e `anonymous`, a lista da equipe não mostra o `filename`, como o `Exams::Files.filename` já gera nomes genéricos.
9. **W9 — Como contar 20 e 4.000.** O contrato não diz se o texto é aparado nem em que unidade se conta. O JavaScript conta em UTF-16 (um emoji vale 2) e o Ruby em code points (vale 1).
   - Proposta: o api faz `strip` e conta com `String#length`. O wpda conta com `[...s.trim()].length` e manda o texto aparado (Task 2).
   - Nota sobre o teto de 16 KB do §4: 4.000 code points de 4 bytes dão cerca de 16,1 KB com o envelope, abaixo de 16.384 bytes. Basta o api recusar caractere de controle (que o JSON escapa em 6 bytes), em vez de subir o teto.
10. **W10 — Cor do prazo no detalhe.** A lista traz `color`, o detalhe não (§3). O wpda calcula a frase pelo estado e pela data (`deadlineText`, Task 2), mas o "amarelo" e o "vermelho" ficam com a regra do api só na lista.
    - Proposta: o detalhe também traz `color`, pela mesma `Ombudsman::Deadlines`.
11. **W11 — Códigos de limite de taxa.** O §4 usa 429 `rate_limited` para o anônimo, mas o resto do api usa `too_many_requests` (`sessions_controller.rb:35` e outros 20). O §3 não fixa o código dos limites de criação (10 por dia) nem de anexos (20 por hora).
    - Proposta: o anônimo mantém `rate_limited` e `daily_cap_reached` (frases próprias, que já sabem que é denúncia). Os do §3 usam `too_many_requests`.
    - O wpda mapeia os três (Task 2).
12. **W12 — `feature_disabled` com 404 no 33 e 403 no resto.** As rotas do 26a, do 29 e do 30 respondem 403 `feature_disabled` (`citizenApi.ts:882-893`). O §1 do 33 diz 404. O wpda decide pelo código, não pelo status, então funciona com os dois.
    - Proposta: o §1 registra que a diferença é de propósito ou alinha com o 403. O wpda não muda.
13. **W13 — Denúncia anônima no desktop.** A denúncia mora no `Flow` e segue a decisão 9 do ciclo de interface: em produção, só no celular. No desktop, o cidadão vê "Abra no celular" (`App.tsx`). O relatório é a única exceção que já existe.
    - O plano mantém o bloqueio, como pede o brief.
    - Se a cidade quiser a denúncia pelo computador (por exemplo, um servidor que denuncia do trabalho), a exceção é uma decisão do usuário, nos moldes da do relatório (`App.tsx`, `token ? <Report>`). Abriria `AnonymousReportStep` fora do `isPhone`.
14. **W14 — Tamanho do corpo do anexo.** 10 MB em base64 são cerca de 13,4 MB de JSON. O `nginx.conf` do wpda não faz proxy de `/citizen` (o host de produção faz), e o padrão do nginx é 1 MB (`client_max_body_size`).
    - Proposta: o runbook `operacao/rollout-ouvidoria.md` registra `client_max_body_size 15m` para `POST /citizen/ombudsman/attachments` e `/ombudsman/attachments` no proxy da cidade, e mantém o limite baixo nas outras rotas.
    - Hoje o wpda trataria o 413 do proxy como "O texto ficou grande demais" (`http_413`). No anexo, mostra "Envie um PDF, uma foto JPEG ou PNG, de até 10 MB." (ajuste do `AttachmentPicker`: 413 = `invalid_file`).
15. **W15 — "novo" na lista depois de abrir o detalhe.** `has_unread` (§3) depende dos avisos não lidos, e só a caixa de avisos os marca como lidos (`POST /notices/:id/read`). O cidadão que abre a manifestação pela lista continua vendo "novo" nela e o selo no topo.
    - Proposta: `GET /citizen/ombudsman/manifestations/:id` marca como lidos os avisos daquela manifestação, das pessoas da sessão, de forma idempotente. É o mesmo efeito do `read` da caixa, e não grava trilha.
    - Alternativa: deixar como está e o cidadão limpa pelo "Avisos".
    - Se a proposta for aceita, o wpda só passa a reler o selo (`refreshBadge`) ao sair do detalhe.

## Self-review

- **Cobertura (decisões 2, 3, 5, 6, 12, 16, 17, 19; §5; F-33.2, F-33.3, F-33.8; contrato §3, §4, §9, §10, §14):**
  - entrada na tela inicial, com os dois caminhos e o interruptor: Tasks 4 e 7 (Review Focus 7);
  - denúncia anônima sem login, aviso antes do envio, sem protocolo, sem anexo, sem cookie, com os limites: Tasks 1, 4 e 9 (Review Focus 1; W7, W11, W13);
  - manifestação identificada ou preservada, escolha da pessoa, tipo, modo, unidade, texto e anexos base64: Tasks 3 e 5 (Review Focus 2, 4 e 5; W2, W8, W9);
  - "Minhas manifestações", com complemento, reabertura e pesquisa (os rótulos do §10): Task 6 (Review Focus 3 e 6; W4, W5, W6, W10, W15);
  - o aviso de ouvidoria na caixa, sem texto: Task 8 (W3);
  - SMS (§9): só no api, conferido na prova;
  - `citizenApi.ts`: Task 1. `sessionFeatures`: lido pelo `Flow` com as constantes da Task 2, sem mudar o arquivo;
  - testes: componentes nas Tasks 3–8 e Playwright do formulário anônimo na Task 9;
  - decisão 9 e `WPDA_ALLOW_DESKTOP`: Global Constraints, device-gate nas Tasks 9 e 10 (W13);
  - lição do polling: sem timer novo, `alive` em toda leitura e no upload (Task 3, "upload que termina depois do desmonte não escreve"; Task 10, Step 2).
- **Placeholders:** nenhum. Valores lidos na execução: ids dos cards, a forma real das rotas e dos códigos do api (Task 0, com a regra de ajuste) e a semente (Task 10).
- **Tipos e nomes:** os novos (Tasks 1–3) são usados nas Tasks 4–8. Os existentes foram conferidos em `origin/main` (`339e4ac`):
  - `call`, `ApiError`, `str`, `strOrNull`, `rec` (movidos para cima sem mudar), `normalizeNotice`, `RawNotice`, `Notice`, `NoticesResult`, `Person`, `citizenApi.people`, `readNotice`;
  - `setSessionFeatures`, `hasSessionFeature`, `AppRoute`, `ROUTES`, `routeFromPath`, `pathFor`;
  - `Flow` (`enter`, `target`, `onUnauthenticated`, `signedIn`, os `case` de `phone`, `people`, `notices` e `declined`);
  - `PhoneStep`, `PeopleStep`, `personLine`, `NoticesStep`, `ItemContent`, `TripDetail`, `newStyle`;
  - `Screen`, `BigButton`, `ErrorText`, `RadioGroup`, `Field`, `ConfirmInline`, `FROZEN_TEXT_NOTICE`, `messageFor`;
  - `fmtDate`, `fmtDateTime`, `fmtCalendarDate`, `cityToday`;
  - `MyTripsStep` (molde), `playwright.config.ts` (`WPDA_URL`, 5 perfis), `e2e/trips.spec.ts` (molde) e `Exams::Files` no api (`MAX_BYTES`, `MAGIC`, `store_base64`).
- **Review Focus:** 1 — Tasks 1, 4 e 9; 2 — Tasks 2, 3 e 5; 3 — Tasks 1, 2 e 6; 4 — Task 5; 5 — Tasks 3, 4 e 5; 6 — Task 6; 7 — Tasks 4 e 7.
- **Proporção:** seis peças novas, porque o §14 pede telas novas. As mudanças em telas existentes são props opcionais (`PhoneStep`, `PeopleStep`, `NoticesStep`) e os `case` do `Flow`. Nenhum componente comum de `ui.tsx` muda.
