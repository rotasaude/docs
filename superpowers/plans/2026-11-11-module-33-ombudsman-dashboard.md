# Módulo 33 — Ouvidoria do SUS municipal (dashboard) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do 33** (`feat/mod-33-ombudsman`, worktree `apps/api/.claude/mod33`, porta de dev **3056**) com as rotas do contrato §5, §6, §7 e §11 (e as Divergências D1–D24 deste plano que o plano do api adotar), os interruptores `ombudsman` e `ombudsman_anonymous_web` na sessão, os três papéis em `Membership::ROLES` e a semente de assuntos (§7). Ordem de entrega do contrato §15: api → dashboard → wpda → maintenance. As Tasks 0–12 não dependem do api (dublês de `src/lib/api` com fixtures nas formas do contrato e das Divergências); a prova da Task 13 depende.

**Goal:** Lado dashboard do 33: a área **"Ouvidoria"** no menu, visível com o interruptor `ombudsman` e o papel; a **Caixa** (lista com filtros, cor de prazo e paginação por cursor); a **Manifestação** (dados, identidade conforme o modo, texto, linha do tempo, encaminhamentos com extrato e resposta, anexos, pedido de reabertura) com **todas as ações do §2**, cada uma com `expected_state` e o tratamento do 409 `manifestation_changed`; a **identidade com justificativa** (só o ouvidor, diálogo próprio, nada em cache); a **Nova manifestação** pela equipe (presencial, telefone, OuvidorSUS; os três modos; busca do cidadão pelo CPF por POST; anexos) com **comprovante para impressão**; as **Respostas à ouvidoria** do ponto de resposta (só o extrato); a **Configuração** (respondentes, assuntos, prazos) e os **Setores da secretaria** do `municipal_admin`; o **Painel** (com o recorte por profissional só do ouvidor) e o **Relatório anual** com publicação e impressão; o aviso fixo do extrato; os três papéis no convite da Equipe.

**Architecture:** O cliente HTTP ganha a seção "Ouvidoria (módulo 33)" no fim de `src/lib/api.ts` (tipos = formas do contrato §5–§7, §11 e das Divergências). As regras de tela ficam puras em `src/lib/ombudsman.ts` (portões, papéis, rótulos, cores, **quais ações cabem em cada estado e papel** — espelho da tabela do §2 —, validações e as frases de **todos** os códigos de erro). As telas ficam em `src/modules/ombudsman/` com as telas-raiz `src/modules/Ombudsman*.tsx` e `src/modules/Departments.tsx`. O dashboard nunca decide o que o api garante: máquina de estados, prazo, cor, sigilo da identidade, trilha de acesso, respondente do destino, supressão e imutabilidade são do api; a tela só não oferece nem envia o que já sabe que será recusado, e sempre manda o `expected_state` que leu.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, Vitest 2 + Testing Library 16 (jsdom, sem jest-dom, sem msw: `vi.mock("…/lib/api")`), Vite 5 (base `/dashboard/`), ícones `@tabler/icons-react` 3.

**Spec:** `docs/superpowers/specs/2026-11-11-module-33-ombudsman-design.md` (decisões 1–19; §4 telas; §5 segurança e LGPD; §6 testes; §8 F-33.1, F-33.4–F-33.7, F-33.9), `docs/adr/0051.md` e o contrato `docs/superpowers/plans/2026-11-11-module-33-ombudsman-contracts.md` (§1, §2, §5–§8, §11, §13, §14). O que o contrato não fixa — ou fixa de um jeito que a tela não consegue usar — está em **"Divergências propostas ao contrato"** (D1–D24); nada foi resolvido em silêncio.

## Global Constraints

- Nomes no código (contrato §1–§2): interruptores `ombudsman` e `ombudsman_anonymous_web` (o dashboard só lê o primeiro; o segundo ganha só rótulo); papéis `ombudsman`, `ombudsman_staff`, `ombudsman_responder`; tipos `complaint` | `report` | `praise` | `suggestion` | `request`; modos `identified` | `preserved` | `anonymous`; canais `web` | `in_person` | `phone` | `ouvidorsus`; estados `received` | `in_analysis` | `forwarded` | `awaiting_complement` | `answered` | `archived`; cores `green` | `yellow` | `red` | `paused` (nulo nos estados finais); motivos de arquivamento `duplicate` | `no_complement` | `lack_of_clarity` | `lack_of_civility` | `improper` | `loss_of_object` | `forwarded_elsewhere`; resolutividade `resolved` | `not_resolved`. **Código desconhecido aparece cru, nunca some.**
- **Interruptor:** sem `ombudsman` na sessão, o grupo "Ouvidoria" some inteiro e toda tela do módulo mostra `OMBUDSMAN_DISABLED` sem ler nada; a recusa `feature_disabled` do api (403 hoje, ver D1) vira a mesma frase. O dashboard nunca escreve interruptor; as duas chaves ganham só rótulo em `features.ts`.
- **Quem faz o quê** (contrato §1): `staff` = `ombudsman_staff` **ou** `ombudsman` (o ouvidor faz tudo do técnico); `lead` = `ombudsman`; `responder` = `ombudsman_responder`; `admin` = `municipal_admin` (só Setores). O respondente sem `staff` vê **só** "Respostas à ouvidoria" (o api responde 403 ao resto, §6). O ponto de resposta nunca fala com o cidadão (antiobjetivo).
- **Identidade** (contrato §8, decisão 12): no `identified`, a manifestação mostra `{ name, cpf_masked, phone_masked }` como o api manda; no `preserved` e no `anonymous`, a tela mostra **só** a frase `IDENTITY_PRESERVED` / `ANONYMOUS_NO_IDENTITY` e **lê campo a campo**: um `name`, `cpf`, `phone` ou `manifestant_*` que o api mande por engano não chega ao DOM. A identidade completa vem só de `POST …/identity` com justificativa, fica **em estado local do diálogo** (nunca no cache do TanStack Query, nunca em storage, nunca em `console`) e some ao fechar; reabrir pede justificativa nova (nova linha na trilha). Anexo de `preserved`/`anonymous` só abre pelo ouvidor, com justificativa.
- **Extrato:** o campo do extrato tem, antes do envio, o aviso fixo do contrato §14 `EXTRACT_NOTICE = "Não escreva o nome nem dados de quem fez a manifestação."` **e** o `FrozenTextNotice` do api#32 (o extrato não muda depois de enviado, contrato §13), os dois por `aria-describedby`. Nenhum componente de `src/components` muda (D2).
- **Texto congelado:** todo texto livre que o api grava sem volta (corpo da manifestação, extrato, resposta do ponto, textos das ações, justificativas, motivo) tem o `FrozenTextNotice` antes do envio (decisão do api#32; helper `expectFrozenNotice` de `src/test/frozenNotice.ts`). Exceção: os três textos do relatório anual, que são documento público editável até a publicação, levam `PUBLIC_REPORT_NOTICE`.
- **Dado pessoal só no corpo:** CPF, nome, contato, texto da manifestação, extrato, resposta, motivo e justificativa nunca na URL, em mensagem de erro, em `console`, em Analytics ou em storage. Na URL vão só ids, os filtros da Caixa (`state[]`, `color[]`, `kind[]`, `channel[]`, `subject_id`, `health_unit_id`, `q` = protocolo), o cursor, `from`/`to` do painel e o ano do relatório. Busca de pessoa por `POST /ombudsman/person_lookup` (D6). **Nenhum storage** no módulo.
- **Concorrência** (contrato §2): toda ação da manifestação leva `expected_state` = o `state` que a tela leu. 409 `manifestation_changed` (ou `forwarding_changed`, D10) → a frase `MANIFESTATION_CHANGED`, **uma** releitura do detalhe, nenhum reenvio automático; o texto digitado fica no formulário se a ação ainda couber no estado novo, senão o formulário fecha. Nenhum polling, nenhum `refetchInterval` novo.
- **Todas as recusas de uma vez:** a tela confere o formulário inteiro antes de enviar e mostra cada problema; a recusa do api com `problems[]` mostra todas, traduzidas por `ombudsmanRefusals`; sem `problems`, a do topo. Todo código do contrato e das Divergências tem frase em `OMBUDSMAN_ERRORS` (o teste da Task 2 cobra a lista inteira); código novo do api aparece como "recusado pelo servidor (<código>)".
- **Recusar em vez de cortar** (contrato §1 e D11): texto da manifestação 20–4000; extrato 20–4000; resposta do ponto 10–4000; motivo de recusa e de prorrogação do ponto 10–500; texto ao cidadão (complemento pedido, intermediária, conclusiva) 20–4000; justificativa de prorrogação, de mudança de resolutividade, de reabertura de ofício e de identidade 10–500; decisão do pedido de reabertura 10–2000; consentimento 10–500; `ouvidorsus_number` até 40; `external_body_name` 2–120; nome de setor e de assunto 2–80; prazos 1–120 e teto anônimo 1–1000, inteiros; textos do relatório até 8000; anexo PDF, JPEG ou PNG, até 10 MB, até 3.
- **Sem duplo envio:** botão de escrita desabilitado enquanto a chamada corre (`busy` + `useRef`), e a resposta de uma ação de uma manifestação já trocada é ignorada (`seq`).
- Toda escrita por cookie com `Content-Type: application/json` (o `jsonFetch` põe quando há corpo; toda escrita deste módulo tem corpo — `{}` quando não há campo).
- **Lição da CI (commit `443faca`):** todo teste que faz `vi.mock("…/lib/api", async (orig) => ({ ...(await orig()), … }))` **inclui `fetchCurrentSession: vi.fn()`** com `mockResolvedValue(<usuário>)` — o `AuthProvider` do `renderWithProviders` lê a sessão, e sem o mock o `fetch("/session")` real rejeita depois do teste e derruba o vitest na CI com tudo verde. A Task 13 varre isso.
- "Agora" é `new Date()`; telas recebem `now` por prop quando a cor ou o prazo dependem do dia, e os testes passam `NOW33 = new Date("2026-11-11T10:00:00-03:00")`. Sem relógio falso neste módulo.
- Interface (ciclo de interface): shell em `src/shell`, toda tela nova em `NAV_GROUPS` **e** em `CATALOG.screens` (o `catalog.test.ts` cobra ícone e descrição, inclusive do grupo), `DataTable` com `card: "title"`/`"actions"` para o telefone, `responsive.css` intocado, cores só por papéis do tema (`Tag` com `tone`: `ok`, `warn`, `down`, `info`, `neutral`), sem cor literal, sem `outline: "none"`, sem `textOverflow` sem `title` (guardas `src/theme/*.test.ts`). A cor do prazo **sempre com texto** ("perto do prazo"), nunca só tom. Componentes existentes (`PageHeader`, `Panel`, `DataTable`, `Tag`, `EmptyState`, `StatTile`, `KeyValue`, `SensitiveAction`, `ConfirmButton`, `ActionButton`, `ActionMessage`, `ReasonHint`, `FrozenTextNotice`, `formStyles`, os estilos de `src/modules/exams/styles.ts`, `fmtCount` de `src/lib/smallCount.ts`). Achado só visual não vira tarefa.
- Nunca `git add -A` nem `git add .`. Adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Merge, push e tag só com autorização explícita do usuário.
- Board: a primeira ação da execução é mover os cards **F-33.1, F-33.4, F-33.5, F-33.6, F-33.7 e F-33.9** do board #1 para **In Progress** (Task 0); um card vai a **Done** só quando o **último** app dele termina (F-33.1: api, dashboard e maintenance; os demais: api e dashboard) e a **Verified** depois da prova no navegador (Task 13).
- **Relato ao coordenador:** ao fim de cada task, uma linha à sessão "Próximos módulos" (via `SendMessage`), no formato `dashboard · dashboard-mod33/feat/mod-33-ombudsman · Task N · <o que ficou pronto> · <divergência nova, se houver>`.

## Ambiente de execução

- Worktree em `/Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/dashboard-mod33` (**nunca** dentro de `apps/dashboard`: o compose monta `./apps/dashboard:/app` e um `node_modules` ali dentro derruba o Vite do `dashboard-dev` com React duplicado), branch `feat/mod-33-ombudsman` a partir de `origin/main` (`443faca`), com `npm ci`. Caminhos de arquivo relativos ao worktree.
- `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `W=$R/.worktrees/dashboard-mod33`, `G=/opt/homebrew/bin/git`.
- Testes: `$R/bin/test dashboard -w dashboard-mod33 <arquivos>` (o runner único; uma execução por vez; a saída é o resumo, o log fica em `.test-runs/`). Nunca `vitest` direto. Tipos antes de cada commit: `cd $W && npx tsc --noEmit`.
- Ambiente de teste: `vitest.config.ts` fixa `TZ=America/Sao_Paulo`; sem jest-dom (`not.toBeNull()`, `toBe()`, `(el as HTMLInputElement).value`); `globals: false`, todo teste de componente chama `cleanup` no `afterEach`. `renderWithProviders` e `sessionWith` vêm de `src/test/campaignFixtures.tsx`; as fixtures do 33 vêm de `src/test/ombudsmanFixtures.ts` (Task 1).

## Review Focus

1. **A identidade de quem pediu sigilo aparece para quem não é o ouvidor, ou fica na memória da tela.** Espera-se: técnico (`ombudsman_staff`) numa manifestação `preserved` vê só "Identidade preservada…", mesmo que o detalhe traga `identity.name` ou `manifestant_cpf` por engano; o botão "Ver identidade" só existe para o `ombudsman`; a justificativa (10–500) é obrigatória; uma chamada só; a identidade some do DOM ao fechar e nenhuma chave do `queryClient` a contém; reabrir pede justificativa nova e faz nova chamada. Teste: Task 8, "preservada: técnico não vê; ouvidor vê com justificativa; fechar apaga; nada no cache".
2. **Dois técnicos agem na mesma manifestação.** Espera-se: o corpo de toda ação leva o `expected_state` lido; 409 `manifestation_changed` mostra a frase, relê o detalhe **uma** vez, não reenvia, mantém o texto se a ação ainda cabe e fecha o formulário se não cabe; o duplo clique envia uma vez. Teste: Task 6, "409 manifestation_changed: frase, uma releitura, texto mantido ou formulário fechado".
3. **O extrato leva a identidade ao setor.** Espera-se: o campo do extrato tem os dois avisos antes do envio; o extrato de 19 caracteres não sai; destino sem respondente ativo não é escolhível e o 422 `destination_without_responder` mostra a frase; a resposta conclusiva com encaminhamento pendente fica bloqueada com o motivo e o 422 `forwarding_pending` também tem frase. Teste: Task 6, "encaminhar: avisos no extrato, destino sem respondente, conclusiva bloqueada".
4. **O ponto de resposta vê mais do que o extrato.** Espera-se: a caixa do respondente mostra destino, extrato, prazo com cor e estado — nunca protocolo, texto original, assunto, identidade ou anexos, mesmo que o api os mande (a lista lê campo a campo); responder, recusar e prorrogar enviam uma vez; o encaminhamento cancelado no meio (409) relê a lista. Teste: Task 10, "inbox lê campo a campo; responder uma vez; cancelado no meio".
5. **O comprovante impresso e o relatório anual levam dado pessoal ou texto da manifestação.** Espera-se: o comprovante tem só protocolo, tipo, canal, data de recebimento e prazo — nunca o texto, o nome, o CPF ou o contato digitados; a anônima não oferece comprovante; a janela bloqueada mostra a frase; o relatório impresso tem só os números (com "< 5") e os três textos do ouvidor. Teste: Task 9, "comprovante só com protocolo, tipo, canal, datas; anônima sem comprovante", e Task 12, "relatório impresso".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | board, worktree, base, âncoras, ícones, conferência do api | 0 |
| `src/lib/api.ts`, `src/lib/api.ombudsman.test.ts`, `src/lib/features.ts`, `src/lib/features.test.ts`, `vite.config.ts`, `src/test/ombudsmanFixtures.ts` | tipos e cliente HTTP do 33; interruptores; proxy; dados de teste | 1 |
| `src/lib/ombudsman.ts`, `src/lib/ombudsman.test.ts`, `src/modules/psychosocial/useFormProblems.tsx`, `src/modules/psychosocial/useFormProblems.test.tsx` | portões, papéis, rótulos, cores, ações por estado, validações, frases de todos os códigos; o molde de formulário aceita outro tradutor de recusa | 2 |
| `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/shell/modules.ombudsman.test.ts`, `src/shell/catalog.ts`, `src/App.tsx`, `src/lib/team.ts`, `src/lib/team.test.ts`, `src/modules/ombudsman/OmbudsmanGate.tsx`, `src/modules/ombudsman/OmbudsmanGate.test.tsx`, cascas `src/modules/Ombudsman{Manifestations,New,Replies,Panel,AnnualReport,Settings}.tsx`, `src/modules/Departments.tsx` | grupo "Ouvidoria", telas-portão, papéis no convite (F-33.1) | 3 |
| `src/modules/OmbudsmanManifestations.tsx`, `src/modules/ombudsman/ManifestationList.tsx`, `src/modules/ombudsman/ManifestationFilters.tsx`, `src/modules/ombudsman/DeadlineTag.tsx`, `src/modules/ombudsman/ManifestationList.test.tsx` | Caixa: filtros, cor de prazo, cursor (F-33.5, F-33.6) | 4 |
| `src/modules/ombudsman/ManifestationView.tsx`, `src/modules/ombudsman/IdentityBlock.tsx`, `src/modules/ombudsman/Timeline.tsx`, `src/modules/ombudsman/Forwardings.tsx`, `src/modules/ombudsman/Attachments.tsx`, `src/modules/ombudsman/openFile.ts`, `src/modules/ombudsman/ManifestationView.test.tsx` | Manifestação: dados, identidade por modo, texto, linha do tempo, encaminhamentos, anexos, pedido de reabertura (F-33.5, F-33.7) | 5 |
| `src/modules/ombudsman/ActionsPanel.tsx`, `src/modules/ombudsman/actions/{ClassifyForm,ForwardForm,ExtractField,RequestComplementForm,ComplementForm,ResponseForm}.tsx`, `src/modules/ombudsman/useManifestationAction.ts`, `src/modules/ombudsman/ActionsStaff.test.tsx` | ações do técnico com `expected_state` e 409 (F-33.5, F-33.6) | 6 |
| `src/modules/ombudsman/actions/{ExtendForm,ArchiveForm,ReopenForm,ResolutionForm,ReopenDecisionForm,ConsentForm}.tsx`, `src/modules/ombudsman/ActionsLead.test.tsx` | ações do ouvidor (F-33.6, F-33.7) | 7 |
| `src/modules/ombudsman/IdentityDialog.tsx`, `src/modules/ombudsman/IdentityDialog.test.tsx` | identidade com justificativa; anexo protegido (F-33.7) | 8 |
| `src/modules/OmbudsmanNew.tsx`, `src/modules/ombudsman/NewManifestationForm.tsx`, `src/modules/ombudsman/PersonPick.tsx`, `src/modules/ombudsman/AttachmentPicker.tsx`, `src/modules/ombudsman/Receipt.tsx`, `src/modules/ombudsman/printSheet.ts`, `src/modules/ombudsman/NewManifestation.test.tsx`, `src/modules/ombudsman/printSheet.test.ts` | registro pela equipe e comprovante para impressão (F-33.4) | 9 |
| `src/modules/OmbudsmanReplies.tsx`, `src/modules/ombudsman/ReplyCard.tsx`, `src/modules/OmbudsmanReplies.test.tsx` | caixa do ponto de resposta (F-33.5) | 10 |
| `src/modules/OmbudsmanSettings.tsx`, `src/modules/ombudsman/{RespondersPanel,SubjectsPanel,DeadlinesPanel}.tsx`, `src/modules/Departments.tsx`, `src/modules/OmbudsmanSettings.test.tsx`, `src/modules/Departments.test.tsx` | respondentes, assuntos, prazos; setores da secretaria (F-33.1) | 11 |
| `src/modules/OmbudsmanPanel.tsx`, `src/modules/OmbudsmanAnnualReport.tsx`, `src/modules/OmbudsmanPanel.test.tsx`, `src/modules/OmbudsmanAnnualReport.test.tsx` | painel com supressão, recorte por profissional, relatório anual com publicação e impressão (F-33.9) | 12 |
| — | suíte, build, varredura, revisão, prova no navegador | 13 |

**Estratégia de teste:** cliente HTTP com `fetch` falso (o molde de `src/lib/api.ombudsman.test.ts` é o `src/lib/api.psychosocial.test.ts`: `stub`, `last`, `bodyOf`) conferindo URL, método e corpo — sobretudo o que **não** pode ir na URL (CPF, nome, texto, extrato, justificativa) e o `expected_state` em toda ação (Task 1); regras puras com tabela de casos, inclusive a matriz estado × papel × ação do §2 e a cobertura de **todos** os códigos de erro (Task 2); menu e portão por papel e interruptor, e os papéis no convite (Task 3); cada tela pelo ponto de entrada real com `renderWithProviders`, `vi.mock` do api **com `fetchCurrentSession`**, sessão por papel e fixtures do contrato, cobrindo o caminho feliz, cada recusa que muda a tela, o duplo envio, o 409 e o que **não** pode aparecer (identidade preservada, protocolo no ponto de resposta, texto no comprovante) (Tasks 4–12); `tsc`, build, a suíte inteira e as varreduras, e a prova no navegador com o api real na 3056 (Task 13). O dashboard não testa o que é do api (máquina de estados, prazo, cor, trilha, supressão, imutabilidade, avisos ao cidadão, wpda).

---

### Task 0: Board, worktree, base, âncoras, ícones e conferência do api

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do dashboard (`443faca` ou mais novo); o board #1 (`rotasaude`, `PVT_kwDOEbfGRc4BbxiB`, campo Status `PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`); o worktree do api `apps/api/.claude/mod33`.
- Produces: cards em In Progress; worktree `$W` na branch `feat/mod-33-ombudsman`; a base anotada; as rotas e as formas do api anotadas.

- [ ] **Step 1: Mova os cards para In Progress** (um comando `gh` simples por vez — sem `$(...)` nem laço)

```bash
gh project item-list 1 --owner rotasaude --limit 500 --format json --jq '.items[] | select(.title | test("^F-33\\.[1-9] ")) | {id, title, status}'
gh project field-list 1 --owner rotasaude --format json --jq '.fields[] | select(.name == "Status") | {id, options}'
```

Para cada um de F-33.1, F-33.4, F-33.5, F-33.6, F-33.7 e F-33.9 que ainda não estiver em In Progress ou além:

```bash
gh project item-edit --id <PVTI_do_card> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id de In Progress>
```

Expected: os seis cards em In Progress ou além. Se algum F-33.x não existir no board, **pare e pergunte** (F-ID novo é decisão do usuário).

- [ ] **Step 2: Confira as âncoras que este plano altera**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/dashboard fetch -q origin
$G -C $R/apps/dashboard log --oneline -3 origin/main
$G -C $R/apps/dashboard show origin/main:src/lib/api.ts | grep -n '^// ─── Urgência — observação, prescrição e checagem (módulo 32b\|^async function binaryFetch\|^function readBase64\|^const withQuery = \|^export async function fetchCurrentSession\|^export interface SignatureFile'
$G -C $R/apps/dashboard show origin/main:src/lib/features.ts | grep -n '  urgency_stock_deduction: "Baixa de estoque na medicação da UPA"\|  | "urgency_stock_deduction";'
$G -C $R/apps/dashboard show origin/main:src/lib/team.ts | grep -n '  { role: "psychosocial_lead", label: "Coordenação do CAPS" },\|^const PRIVILEGED_ROLES = '
$G -C $R/apps/dashboard show origin/main:src/lib/team.test.ts | grep -n 'oferece os 23 papéis'
$G -C $R/apps/dashboard show origin/main:src/shell/modules.ts | grep -n '  | "urgency-panel" | "waiting-room-settings" | "urgency-beds";\|    { id: "urgency-beds", label: "Leitos de observação"\|  const wrOn = canUseWaitingRoom(user);\|      if (item.id === "urgency-beds") return urgOn && isAdmin;'
$G -C $R/apps/dashboard show origin/main:src/shell/modules.test.ts | grep -n 'NAV_GROUPS.length - '
$G -C $R/apps/dashboard show origin/main:src/shell/catalog.ts | grep -n '    "Urgência": {\|    "waiting-room-settings": {\|IconBrain, IconAmbulance'
$G -C $R/apps/dashboard show origin/main:src/App.tsx | grep -n '    case "waiting-room-settings":\|^export function useCapsHandoff\|^export function renderModule'
$G -C $R/apps/dashboard show origin/main:vite.config.ts | grep -n '"\^/waiting_room(/|\$)": proxy(TARGET)\|/waiting_room → TV'
$G -C $R/apps/dashboard show origin/main:src/modules/psychosocial/useFormProblems.tsx | grep -n '^export function useFormProblems()\|      const refusals = psyRefusals(err);'
$G -C $R/apps/dashboard show origin/main:src/components/FrozenTextNotice.tsx | grep -n '^export const FROZEN_TEXT_NOTICE\|^export function FrozenTextNotice'
$G -C $R/apps/dashboard show origin/main:src/lib/smallCount.ts | grep -n '^export function fmtCount\|^export const SUPPRESSED_LABEL'
$G -C $R/apps/dashboard show origin/main:src/modules/documents/printDocument.ts | grep -n '^export const POPUP_BLOCKED'
```

Expected: cada `grep` acha a linha pedida. Se alguma mudou, anote: a task aplica a mesma intenção sobre o texto real.

- [ ] **Step 3: Crie o worktree fora de `apps/`, anote a base e os ícones**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/dashboard worktree add -q $R/.worktrees/dashboard-mod33 -b feat/mod-33-ombudsman origin/main
cd $R/.worktrees/dashboard-mod33 && npm ci --silent && npx tsc --noEmit && echo tsc-ok
$R/bin/test dashboard -w dashboard-mod33
ls node_modules/@tabler/icons-react/dist/esm/icons | grep -E '^Icon(MessageReport|MailForward|Sitemap|Inbox|FilePlus|ChartBar|FileReport|Settings)\.mjs$'
```

Expected: suíte verde, `tsc-ok` e os oito ícones (falta algum → escolha outro que exista e use o mesmo na Task 3). **Anote** a base (arquivos e testes). A Task 13 espera a base mais **16 arquivos de teste novos** (listados lá).

- [ ] **Step 4: Confira as rotas e formas do api do 33**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/api log --oneline -5 feat/mod-33-ombudsman 2>/dev/null || echo "api do 33 ainda sem branch"
$G -C $R/apps/api show feat/mod-33-ombudsman:config/routes.rb 2>/dev/null | grep -nE 'ombudsman|departments'
$G -C $R/apps/api show feat/mod-33-ombudsman:app/models/membership.rb 2>/dev/null | grep -n 'ombudsman'
$G -C $R/apps/api grep -n 'feature_disabled' feat/mod-33-ombudsman -- 'app/controllers/ombudsman/*' 2>/dev/null | head
```

Expected: as rotas do contrato §5–§7 e §11 e os três papéis. Para cada Divergência (D1–D24) anote se o api adotou a forma proposta ou outra; **onde o api diz outra coisa, vale o api** — a task dona ajusta tipo, fixture e teste antes de seguir (registre na descrição do commit). Sem branch do api ainda: siga pelas formas deste plano e repita este passo antes da Task 13.

**Relato ao coordenador:** `dashboard · dashboard-mod33/feat/mod-33-ombudsman · Task 0 · worktree e base (<n> arquivos, <m> testes) · <divergências que o api já resolveu ou contrariou>`.

---

### Task 1: Tipos, cliente HTTP, interruptores, proxy e dados de teste

**Files:**
- Modify: `src/lib/api.ts` (seção nova no fim), `src/lib/features.ts`, `src/lib/features.test.ts`, `vite.config.ts`
- Create: `src/test/ombudsmanFixtures.ts`
- Test: `src/lib/api.ombudsman.test.ts`

**Interfaces:**
- Consumes: `jsonFetch`, `ApiError`, `withQuery`, `binaryFetch`, `readBase64`, `SignatureFile`, `SessionUser` (existentes em `src/lib/api.ts`); `SmallCount` (`src/lib/types.ts`); `sessionWith` de `src/test/campaignFixtures.tsx`.
- Produces (todas em `src/lib/api.ts`, exportadas):
  - Tipos: `OmbudsmanKind`, `IdentificationMode`, `OmbudsmanChannel`, `ManifestationState`, `DeadlineColor = "green" | "yellow" | "red" | "paused"`, `ArchiveReason`, `Resolution`, `ForwardingState = "pending" | "replied" | "declined" | "cancelled"`, `ReopenRequestState`, `OmbudsmanActionName = "classify" | "forward" | "request_complement" | "complement" | "extend" | "interim_response" | "final_response" | "archive" | "reopen" | "change_resolution"`.
  - Formas (contrato §5–§7, §11 + D): `OmbudsmanRef { id: string; name: string }`; `OmbudsmanSubjectRef { id; name; parent: OmbudsmanRef | null }` (D12); `OmbudsmanDestination { kind: "department" | "health_unit"; id; name }` (D13); `OmbudsmanListItem { id; protocol; kind; identification_mode; channel; state; subject: OmbudsmanSubjectRef | null; health_unit: OmbudsmanRef | null; urgent: boolean; received_at; due_on: string | null; color: DeadlineColor | null; pending_forwardings: number }`; `OmbudsmanPage { items: OmbudsmanListItem[]; next_before: string | null }` (D14); `OmbudsmanListQuery { state?: string[]; color?: string[]; kind?: string[]; channel?: string[]; subject_id?: string | null; health_unit_id?: string | null; q?: string; before?: string | null }`; `OmbudsmanIdentitySummary { name: string | null; cpf_masked: string | null; phone_masked: string | null; contact: string | null }` (D15); `OmbudsmanEvent { id; kind: string; actor: "staff" | "citizen" | "system"; actor_name: string | null; at: string; text: string | null; data: Record<string, unknown>; visible_to_citizen: boolean }` (D16); `OmbudsmanForwarding { id; destination: OmbudsmanDestination; extract: string; due_on: string; color: DeadlineColor | null; extended: boolean; state: ForwardingState; reply: string | null; replied_at: string | null; replied_by_name: string | null; created_at: string }`; `OmbudsmanAttachment { id; uploaded_by: "citizen" | "staff"; content_type: string; created_at: string }` (D17, sem `filename`); `OmbudsmanReopenRequest { id; reason: string; state: ReopenRequestState; created_at; decision_text: string | null; decided_at: string | null }`; `OmbudsmanManifestation extends OmbudsmanListItem { ouvidorsus_number: string | null; body: string; identity: OmbudsmanIdentitySummary | "preserved"; identity_available: boolean; referred_professional: OmbudsmanRef | null; cycle: number; cycle_started_on: string; extended: boolean; complement_used: boolean; complement_due_on: string | null; resolution: Resolution | null; archive_reason: ArchiveReason | null; duplicate_of: { id: string; protocol: string } | null; external_body_name: string | null; concluded_at: string | null; events: OmbudsmanEvent[]; forwardings: OmbudsmanForwarding[]; attachments: OmbudsmanAttachment[]; reopen_request: OmbudsmanReopenRequest | null }` (D15, D18); `OmbudsmanFullIdentity { name: string | null; cpf: string | null; phone: string | null; manifestant_name: string | null; manifestant_contact: string | null; manifestant_cpf: string | null }`; `OmbudsmanOptions { units: (OmbudsmanRef & { has_active_responder: boolean })[]; departments: (OmbudsmanRef & { has_active_responder: boolean })[]; subjects: OmbudsmanSubject[] }` (D5); `OmbudsmanSubject { id; name; parent_id: string | null; active: boolean }`; `OmbudsmanPerson { citizen_id: string | null; display_name: string | null; cpf_masked: string }` (D6); `NewManifestationInput` (contrato §5, campos opcionais só quando preenchidos); `OmbudsmanCreated { id; protocol: string | null; due_on: string | null }`; `OmbudsmanReceipt { protocol; kind; received_at; due_on; channel }`; `OmbudsmanInboxItem { id; destination: OmbudsmanDestination; extract; due_on; color: DeadlineColor | null; state: ForwardingState; extended: boolean; created_at }`; `Department { id; name; active: boolean }`; `OmbudsmanResponder { id; user: { id: string; email: string }; destination: OmbudsmanDestination; active: boolean }`; `ResponderCandidate { user_id: string; email: string }` (D7); `OmbudsmanSettings` (os oito inteiros do contrato §1); `OmbudsmanDashboard` (D19); `ProfessionalPanelRow { professional: OmbudsmanRef; kind: OmbudsmanKind; count: number }`; `OmbudsmanAnnualReport { year: number; state: "draft" | "published"; recurring_problems: string; actions_taken: string; notes: string; numbers: OmbudsmanDashboard; published_at: string | null; published_by: { name: string } | null }` (D20).
  - Funções: `listOmbudsmanManifestations(q: OmbudsmanListQuery): Promise<OmbudsmanPage>`, `getOmbudsmanManifestation(id)`, `createOmbudsmanManifestation(i: NewManifestationInput): Promise<OmbudsmanCreated>`, `runOmbudsmanAction(id, action: OmbudsmanActionName, body: Record<string, unknown> & { expected_state: ManifestationState }): Promise<OmbudsmanManifestation>` (D9), `cancelOmbudsmanForwarding(id): Promise<OmbudsmanManifestation>`, `decideReopenRequest(id, i: { decision: "accept" | "reject"; text: string }): Promise<OmbudsmanManifestation>`, `revealOmbudsmanIdentity(id, justification): Promise<OmbudsmanFullIdentity>`, `recordIdentityConsent(id, i: { granted: boolean; text: string }): Promise<OmbudsmanManifestation>`, `getOmbudsmanReceipt(id): Promise<OmbudsmanReceipt>`, `uploadOmbudsmanAttachment(file: File): Promise<{ id: string }>`, `getOmbudsmanAttachmentUrl(id, justification?: string): Promise<string>`, `fetchOmbudsmanFile(url): Promise<SignatureFile>`, `lookupOmbudsmanPerson(cpf): Promise<OmbudsmanPerson>`, `getOmbudsmanOptions(): Promise<OmbudsmanOptions>`, `listOmbudsmanProfessionals(unitId): Promise<OmbudsmanRef[]>` (D8), `listOmbudsmanInbox(): Promise<OmbudsmanInboxItem[]>`, `replyOmbudsmanInbox(id, body)`, `declineOmbudsmanInbox(id, reason)`, `extendOmbudsmanInbox(id, reason)` (as três → `OmbudsmanInboxItem`), `listDepartments()`, `createDepartment(name)`, `updateDepartment(id, i: { name: string; active: boolean })`, `listOmbudsmanResponders()`, `listResponderCandidates()`, `createOmbudsmanResponder(i: { user_id: string } & ({ department_id: string } | { health_unit_id: string }))`, `updateOmbudsmanResponder(id, i: { active: boolean })`, `listOmbudsmanSubjects()`, `createOmbudsmanSubject(i: { name: string; parent_id: string | null })`, `updateOmbudsmanSubject(id, i: { name: string; active: boolean })`, `getOmbudsmanSettings()`, `updateOmbudsmanSettings(s: OmbudsmanSettings)`, `getOmbudsmanDashboard(q: { from: string; to: string })`, `getOmbudsmanProfessionalsPanel(q: { from: string; to: string }): Promise<ProfessionalPanelRow[]>`, `getAnnualReport(year: number)`, `updateAnnualReport(year, i: { recurring_problems: string; actions_taken: string; notes: string })`, `publishAnnualReport(year)`.
  - `src/test/ombudsmanFixtures.ts`: `NOW33`, `TODAY33 = "2026-11-11"`, `ombudsmanUser(roles, features?)`, `UNITS33`, `DEPARTMENTS33`, `SUBJECTS33`, `options(o?)`, `listItem(o?)`, `page(o?)`, `manifestation(o?)`, `forwarding(o?)`, `event(o?)`, `inboxItem(o?)`, `settings33(o?)`, `dashboard33(o?)`, `annualReport(o?)`, `receipt(o?)`, `fullIdentity(o?)`.

- [ ] **Step 1: Escreva as fixtures** `src/test/ombudsmanFixtures.ts` (o único arquivo com corpo completo nesta task)

```ts
// Dados dos testes do módulo 33 (ouvidoria). Agora: quarta, 2026-11-11, 10h. Formas do contrato §5–§7, §11 e das Divergências.
import type {
  OmbudsmanAnnualReport, OmbudsmanDashboard, OmbudsmanEvent, OmbudsmanForwarding, OmbudsmanFullIdentity, OmbudsmanInboxItem,
  OmbudsmanListItem, OmbudsmanManifestation, OmbudsmanOptions, OmbudsmanPage, OmbudsmanReceipt, OmbudsmanSettings, OmbudsmanSubject, SessionUser
} from "../lib/api";
import { sessionWith } from "./campaignFixtures";

export const NOW33 = new Date("2026-11-11T10:00:00-03:00");
export const TODAY33 = "2026-11-11";
export const ombudsmanUser = (roles: string[], features: string[] = [ "ombudsman" ]): SessionUser => sessionWith(roles, { features });

export const UNITS33 = [ { id: "ubs1", name: "UBS Centro", has_active_responder: true }, { id: "upa1", name: "UPA Boqueirão", has_active_responder: false } ];
export const DEPARTMENTS33 = [ { id: "dep1", name: "Assistência Farmacêutica", has_active_responder: true }, { id: "dep2", name: "Regulação", has_active_responder: false } ];
export const SUBJECTS33: OmbudsmanSubject[] = [
  { id: "s1", name: "Atendimento", parent_id: null, active: true }, { id: "s11", name: "Demora", parent_id: "s1", active: true },
  { id: "s12", name: "Conduta de profissional", parent_id: "s1", active: true }, { id: "s6", name: "Transporte", parent_id: null, active: true },
  { id: "s9", name: "Antigo", parent_id: null, active: false }
];
export const options = (o: Partial<OmbudsmanOptions> = {}): OmbudsmanOptions => ({ units: UNITS33, departments: DEPARTMENTS33, subjects: SUBJECTS33, ...o });

export const listItem = (o: Partial<OmbudsmanListItem> = {}): OmbudsmanListItem => ({
  id: "m1", protocol: "2026-000123", kind: "complaint", identification_mode: "identified", channel: "web", state: "in_analysis",
  subject: { id: "s11", name: "Demora", parent: { id: "s1", name: "Atendimento" } }, health_unit: { id: "ubs1", name: "UBS Centro" },
  urgent: false, received_at: "2026-11-03T09:12:00-03:00", due_on: "2026-12-03", color: "green", pending_forwardings: 0, ...o
});
export const page = (o: Partial<OmbudsmanPage> = {}): OmbudsmanPage => ({ items: [ listItem() ], next_before: null, ...o });

export const event = (o: Partial<OmbudsmanEvent> = {}): OmbudsmanEvent => ({
  id: "e1", kind: "received", actor: "citizen", actor_name: null, at: "2026-11-03T09:12:00-03:00", text: null, data: {}, visible_to_citizen: true, ...o
});
export const forwarding = (o: Partial<OmbudsmanForwarding> = {}): OmbudsmanForwarding => ({
  id: "f1", destination: { kind: "department", id: "dep1", name: "Assistência Farmacêutica" },
  extract: "Usuária relata falta de losartana na farmácia da UBS Centro há duas semanas.", due_on: "2026-11-25", color: "green",
  extended: false, state: "pending", reply: null, replied_at: null, replied_by_name: null, created_at: "2026-11-05T14:00:00-03:00", ...o
});
export const manifestation = (o: Partial<OmbudsmanManifestation> = {}): OmbudsmanManifestation => ({
  ...listItem(), ouvidorsus_number: null,
  body: "Fui à UBS Centro três vezes e não havia losartana. Preciso do remédio para a pressão.",
  identity: { name: "Maria Lima", cpf_masked: "***.***.***-25", phone_masked: "(41) *****-1234", contact: null }, identity_available: true,
  referred_professional: null, cycle: 1, cycle_started_on: "2026-11-03", extended: false, complement_used: false, complement_due_on: null,
  resolution: null, archive_reason: null, duplicate_of: null, external_body_name: null, concluded_at: null,
  events: [ event(), event({ id: "e2", kind: "classified", actor: "staff", actor_name: "Ana Souza", at: "2026-11-04T08:30:00-03:00", visible_to_citizen: false }) ],
  forwardings: [], attachments: [], reopen_request: null, ...o
});
export const preserved = (o: Partial<OmbudsmanManifestation> = {}) =>
  manifestation({ id: "m2", protocol: "2026-000124", kind: "report", identification_mode: "preserved", identity: "preserved", ...o });

export const fullIdentity = (o: Partial<OmbudsmanFullIdentity> = {}): OmbudsmanFullIdentity => ({
  name: "Joana Prado", cpf: "529.982.247-25", phone: "(41) 99876-1234", manifestant_name: null, manifestant_contact: null, manifestant_cpf: null, ...o
});
export const inboxItem = (o: Partial<OmbudsmanInboxItem> = {}): OmbudsmanInboxItem => ({
  id: "f1", destination: { kind: "department", id: "dep1", name: "Assistência Farmacêutica" },
  extract: "Usuária relata falta de losartana na farmácia da UBS Centro há duas semanas.", due_on: "2026-11-25", color: "green",
  state: "pending", extended: false, created_at: "2026-11-05T14:00:00-03:00", ...o
});
export const settings33 = (o: Partial<OmbudsmanSettings> = {}): OmbudsmanSettings => ({
  response_days: 30, extension_days: 30, urgent_days: 15, complement_days: 20, forwarding_days: 20, forwarding_extension_days: 20,
  warning_days: 5, anonymous_daily_cap: 50, ...o
});
export const receipt = (o: Partial<OmbudsmanReceipt> = {}): OmbudsmanReceipt =>
  ({ protocol: "2026-000130", kind: "complaint", received_at: "2026-11-11T09:58:00-03:00", due_on: "2026-12-11", channel: "in_person", ...o });
export const dashboard33 = (o: Partial<OmbudsmanDashboard> = {}): OmbudsmanDashboard => ({
  from: "2026-11-01", to: "2026-11-11", total: 42,
  by_kind: [ { key: "complaint", count: 30 }, { key: "praise", count: { suppressed: true } } ],
  by_channel: [ { key: "web", count: 28 }, { key: "phone", count: 14 } ],
  by_state: [ { key: "answered", count: 20 }, { key: "in_analysis", count: 22 } ],
  by_subject: [ { id: "s1", name: "Atendimento", count: 18 } ], by_unit: [ { id: "ubs1", name: "UBS Centro", count: 12 } ],
  on_time_rate: 85.0, avg_days_to_conclude: 17.4, resolution_rate: { suppressed: true },
  satisfaction: { responses: 9, met: { yes: 6, partly: { suppressed: true }, no: { suppressed: true } }, clear: { yes: 7, no: { suppressed: true } }, avg_score: 4.1 },
  ...o
});
export const annualReport = (o: Partial<OmbudsmanAnnualReport> = {}): OmbudsmanAnnualReport => ({
  year: 2026, state: "draft", recurring_problems: "", actions_taken: "", notes: "", numbers: dashboard33({ from: "2026-01-01", to: "2026-12-31" }),
  published_at: null, published_by: null, ...o
});
```

(O `OmbudsmanDashboard` das fixtures é a forma da D19; o `as` não entra nas fixtures.)

- [ ] **Step 2: Escreva os testes do cliente** `src/lib/api.ombudsman.test.ts` (mesmo `fetch` falso de `src/lib/api.psychosocial.test.ts`: `stub`, `last`, `bodyOf`)

```ts
// O que nunca pode ir na URL: CPF, nome, contato, texto, extrato e justificativa.
const SECRET_IN_URL = /52998224725|529\.982|Maria|Joana|losartana|pressão|justific|41 9/;

it("caixa: filtros repetidos e cursor na URL; nada além disso", async () => {
  stub({ items: [ listItem() ], next_before: "c2" });
  const p = await listOmbudsmanManifestations({ state: [ "in_analysis", "forwarded" ], color: [ "red" ], kind: [], subject_id: "s1", q: "2026-000123", before: "c1" });
  expect(last().url).toBe("/ombudsman/manifestations?state%5B%5D=in_analysis&state%5B%5D=forwarded&color%5B%5D=red&subject_id=s1&q=2026-000123&before=c1");
  expect(p.next_before).toBe("c2");
});
it("toda ação leva expected_state no corpo e vai ao caminho da ação", async () => {
  stub(manifestation());
  await runOmbudsmanAction("m1", "forward", { expected_state: "in_analysis", department_id: "dep1", extract: "x".repeat(20) });
  expect(last().url).toBe("/ombudsman/manifestations/m1/forward");
  expect(last().init.method).toBe("POST");
  expect(bodyOf()).toEqual({ expected_state: "in_analysis", department_id: "dep1", extract: "x".repeat(20) });
});
it("identidade, consentimento e anexo protegido: justificativa só no corpo", async () => {
  stub(fullIdentity());
  await revealOmbudsmanIdentity("m2", "conferir denúncia repetida do mesmo autor");
  expect(last().url).toBe("/ombudsman/manifestations/m2/identity");
  expect(bodyOf()).toEqual({ justification: "conferir denúncia repetida do mesmo autor" });
  stub({ url: "/ombudsman/files/tok1" });
  expect(await getOmbudsmanAttachmentUrl("a1", "abrir laudo anexado")).toBe("/ombudsman/files/tok1");
  expect(bodyOf()).toEqual({ justification: "abrir laudo anexado" });
  stub({ url: "/ombudsman/files/tok2" });
  await getOmbudsmanAttachmentUrl("a2");
  expect(bodyOf()).toEqual({});
  for (const c of calls) expect(c.url).not.toMatch(SECRET_IN_URL);
});
it("registro pela equipe: só os campos preenchidos; busca de pessoa por POST", async () => {
  stub({ citizen_id: "c1", display_name: "Maria Lima", cpf_masked: "***.***.***-25" });
  await lookupOmbudsmanPerson("52998224725");
  expect(last()).toMatchObject({ url: "/ombudsman/person_lookup", init: { method: "POST" } });
  expect(bodyOf()).toEqual({ cpf: "52998224725" });
  stub({ id: "m9", protocol: "2026-000130", due_on: "2026-12-11" });
  await createOmbudsmanManifestation({ kind: "complaint", identification_mode: "identified", channel: "phone", citizen_id: "c1",
    body: "Fui à UBS Centro três vezes e não havia losartana.", manifestant_name: "  ", health_unit_id: null });
  expect(bodyOf()).toEqual({ kind: "complaint", identification_mode: "identified", channel: "phone", citizen_id: "c1",
    body: "Fui à UBS Centro três vezes e não havia losartana." });
  for (const c of calls) expect(c.url).not.toMatch(SECRET_IN_URL);
});
it("anexo: tipo e base64, sem nome de arquivo (D17)", async () => {
  stub({ id: "a1" });
  await uploadOmbudsmanAttachment(new File([ "%PDF-1.4" ], "RG Maria Lima.pdf", { type: "application/pdf" }));
  expect(last().url).toBe("/ombudsman/attachments");
  expect(Object.keys(bodyOf()).sort()).toEqual([ "content_type", "data" ]);
  expect(JSON.stringify(bodyOf())).not.toMatch(/Maria/);
});
it("ponto de resposta, encaminhamento cancelado e decisão do pedido", async () => {
  stub(inboxItem({ state: "replied" }));
  await replyOmbudsmanInbox("f1", "Reposição feita em 10/11; a UBS já tem o medicamento.");
  expect(last().url).toBe("/ombudsman/inbox/f1/reply");
  expect(bodyOf()).toEqual({ body: "Reposição feita em 10/11; a UBS já tem o medicamento." });
  stub(manifestation());
  await cancelOmbudsmanForwarding("f1");
  expect(last()).toMatchObject({ url: "/ombudsman/forwardings/f1/cancel", init: { method: "POST", body: "{}" } });
  await decideReopenRequest("r1", { decision: "reject", text: "A resposta já tratou o ponto levantado." });
  expect(last().url).toBe("/ombudsman/reopen_requests/r1/decide");
});
it("configuração, painel e relatório pelos caminhos do contrato §7 e §11", async () => {
  stub({ items: [] });
  await listDepartments();
  expect(last().url).toBe("/departments");
  stub({ id: "d1", name: "Ouvidoria", active: false });
  await updateDepartment("d1", { name: "Ouvidoria", active: false });
  expect(last()).toMatchObject({ url: "/departments/d1", init: { method: "PATCH" } });
  stub(settings33());
  await updateOmbudsmanSettings(settings33({ warning_days: 3 }));
  expect(last()).toMatchObject({ url: "/ombudsman/settings", init: { method: "PATCH" } });
  stub(dashboard33());
  await getOmbudsmanDashboard({ from: "2026-11-01", to: "2026-11-11" });
  expect(last().url).toBe("/ombudsman/dashboard?from=2026-11-01&to=2026-11-11");
  stub(annualReport({ state: "published" }));
  await publishAnnualReport(2026);
  expect(last()).toMatchObject({ url: "/ombudsman/annual_reports/2026/publish", init: { method: "POST", body: "{}" } });
});
```

E em `src/lib/features.test.ts`: `featureLabel("ombudsman")` é `"Ouvidoria do SUS municipal"` e `featureLabel("ombudsman_anonymous_web")` é `"Denúncia anônima sem login no canal do cidadão"` (contrato §1).

- [ ] **Step 3: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod33 src/lib/api.ombudsman.test.ts src/lib/features.test.ts`
Expected: FAIL (funções e rótulos inexistentes).

- [ ] **Step 4: Implemente**

- `api.ts`, seção `// ─── Ouvidoria (módulo 33; ADR 0051; contrato 2026-11-11 §5–§7, §11) ───` no fim, com o comentário de topo no molde do 31 (na URL só ids, filtros, cursor, datas do painel e ano; o resto no corpo; listas em `{ items }`; toda escrita com corpo). Bases `OMBUDSMAN_BASE = import.meta.env.VITE_OMBUDSMAN_BASE || "/ombudsman"` e `DEPARTMENTS_BASE = import.meta.env.VITE_DEPARTMENTS_BASE || "/departments"`; `ombPath`, `ombWrite(method, body = {})` e `ombCompact` (tira opcional vazio — `undefined`, `null`, `""` ou só espaços — e apara o que vai) no molde de `psyPath`/`psyWrite`/`psyCompact`. A query da Caixa usa `URLSearchParams.append("state[]", v)` para cada valor das listas, depois os escalares preenchidos, nessa ordem (`state`, `color`, `kind`, `channel`, `subject_id`, `health_unit_id`, `q`, `before`).
- `uploadOmbudsmanAttachment`: `readBase64(file)` e corpo `{ content_type: file.type, data }` — **sem `filename`** (D17, mesmo motivo do TFD: o nome do arquivo pode trazer o nome da pessoa).
- `fetchOmbudsmanFile(url)`: `binaryFetch(url.replace(/^\/ombudsman(?=\/)/, OMBUDSMAN_BASE), "application/pdf,image/jpeg,image/png")`, como `fetchTfdAccountabilityFile`.
- `getOmbudsmanAttachmentUrl(id, justification?)`: corpo `{ justification }` só quando recebida, senão `{}`.
- Listas desembrulham `{ items }`; `getOmbudsmanOptions` e o detalhe vêm como objeto.
- `features.ts`: as duas chaves em `FeatureKey` e `FEATURE_LABEL`, com o comentário "Módulo 33 (ADR 0051): ombudsman sem pré-requisito; ombudsman_anonymous_web exige ombudsman".
- `vite.config.ts`: `"^/ombudsman(/|$)": proxy(TARGET)` e `"^/departments(/|$)": proxy(TARGET)`, e as duas linhas no comentário do topo (`/ombudsman → ouvidoria (módulo 33: caixa, manifestação, ponto de resposta, configuração, painel, relatório anual)`, `/departments → setores da secretaria (módulo 33)`).

- [ ] **Step 5: Rode e veja passar; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod33 src/lib/api.ombudsman.test.ts src/lib/features.test.ts src/lib/api.test.ts && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
cd $W && $G add src/lib/api.ts src/lib/api.ombudsman.test.ts src/lib/features.ts src/lib/features.test.ts vite.config.ts src/test/ombudsmanFixtures.ts
$G commit -m "feat: add the ombudsman API client, feature labels and dev proxy" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** `dashboard · dashboard-mod33/feat/mod-33-ombudsman · Task 1 · cliente HTTP do 33 e fixtures · formas que o api contrariou (D5–D20)`.

---

### Task 2: Regras de tela da ouvidoria

**Files:**
- Create: `src/lib/ombudsman.ts`
- Modify: `src/modules/psychosocial/useFormProblems.tsx`
- Test: `src/lib/ombudsman.test.ts`, `src/modules/psychosocial/useFormProblems.test.tsx`

**Interfaces:**
- Consumes: tipos da Task 1; `hasFeature`, `featureDisabledKey`, `featureLabel` (`src/lib/features.ts`); `NOMINAL_QUERY`, `STATIC_QUERY` (`src/lib/careLines.ts`); `Problem`, `Refusal` (`src/lib/pharmacy.ts`); `textProblem` (`src/lib/psychosocial.ts`); `isValidCpf` (`src/lib/attendance.ts`); `ApiError`, `errorCode` (`src/lib/api.ts`).
- Produces (em `src/lib/ombudsman.ts`):
  - Frases: `OMBUDSMAN_DISABLED = "a ouvidoria está desligada nesta cidade"`, `OMBUDSMAN_NO_ROLE = "seu papel não permite usar a ouvidoria"`, `IDENTITY_PRESERVED = "Identidade preservada — só o ouvidor vê, com justificativa registrada."`, `ANONYMOUS_NO_IDENTITY = "Denúncia anônima, sem dados de quem a fez."`, `EXTRACT_NOTICE = "Não escreva o nome nem dados de quem fez a manifestação."` (contrato §14, exato), `ANONYMOUS_INTERNAL = "Denúncia anônima: a conclusão fica interna e não vai a ninguém."`, `MANIFESTATION_CHANGED = "a manifestação mudou enquanto a tela estava aberta — os dados foram relidos; confira antes de repetir"`, `PUBLIC_REPORT_NOTICE = "Este texto será publicado no site da cidade. Não escreva nome, CPF nem detalhe que identifique alguém."`, `NO_RECEIPT_ANONYMOUS = "A denúncia anônima não tem comprovante."`.
  - Chaves: `OMB_LIST_KEY = "ombudsmanList"`, `OMB_DETAIL_KEY = "ombudsmanDetail"`, `OMB_OPTIONS_KEY = "ombudsmanOptions"`, `OMB_INBOX_KEY = "ombudsmanInbox"`, `OMB_SETTINGS_KEY`, `OMB_SUBJECTS_KEY`, `OMB_RESPONDERS_KEY`, `DEPARTMENTS_KEY`, `OMB_DASHBOARD_KEY`, `OMB_REPORT_KEY`; reexporta `NOMINAL_QUERY` e `STATIC_QUERY`.
  - Portões: `canUseOmbudsman(user)` (não operador e `ombudsman`); `ombudsmanRoles(user): { lead: boolean; staff: boolean; responder: boolean; admin: boolean }` (`staff = lead || ombudsman_staff`).
  - Rótulos (`pick`, cru quando desconhecido): `kindLabel` (Reclamação, Denúncia, Elogio, Sugestão, Solicitação), `modeLabel` (Identificada, Identidade preservada, Anônima), `channelLabel` (Canal do cidadão, Presencial, Telefone, OuvidorSUS), `stateLabel` (recebida, em análise, encaminhada, aguardando complemento, respondida, arquivada), `colorLabel` (no prazo, perto do prazo, vencida, prazo suspenso) e `COLOR_TONE = { green: "ok", yellow: "warn", red: "down", paused: "info" }`, `archiveReasonLabel` (Duplicidade, Sem complemento no prazo, Falta de clareza, Falta de urbanidade, Manifestação imprópria, Perda de objeto, Encaminhada a outro órgão), `resolutionLabel` (resolvida, não resolvida), `forwardingStateLabel` (aguardando resposta, respondido, recusado pelo destino, cancelado), `eventKindLabel` (D16), `destinationLabel(d)` ("Setor: …" / "Unidade: …").
  - Ações: `ACTIONS_BY_STATE` (a tabela do §2), `availableActions(m: OmbudsmanManifestation, roles): OmbudsmanActionName[]` e `blockedReason(m, action): string | null` (o botão aparece desabilitado com o motivo: `final_response` com `pending_forwardings > 0` → "há encaminhamento pendente — cancele ou aguarde a resposta"; `request_complement` com `complement_used` → "o complemento já foi pedido neste ciclo"; `extend` com `extended` → "a prorrogação já foi usada neste ciclo"); `needsResolution(kind)` (`complaint`, `request`, `report`); `canSeeIdentityButton(m, roles)` (`lead` e modo `preserved`/`anonymous` com `identity_available`); `canIssueReceipt(m)` (não anônima).
  - Validações (cada uma devolve `Problem[]`, todos de uma vez): `newManifestationProblems(d: NewDraft)`, `forwardProblems(d, options)`, `textActionProblems(action, d)`, `archiveProblems(d)`, `finalResponseProblems(d, kind)`, `settingsProblems(s)`, `nameProblem(name)` (2–80), `attachmentProblems(files: File[])` (tipo, 10 MB, até 3), `reportTextProblems(r)` (até 8000).
  - Erros: `OMBUDSMAN_ERRORS: Record<string, string>`, `ombudsmanError(err): string`, `ombudsmanRefusals(err): Refusal[]`, `isChanged(err): boolean` (409 `manifestation_changed` ou `forwarding_changed`).
- Produces (em `useFormProblems.tsx`): `useFormProblems(opts?: { refusals?(err: unknown): PsyRefusal[]; isStale?(err: unknown): boolean })` — sem `opts`, o comportamento de hoje (CAPS); a ouvidoria passa `{ refusals: (e) => ombudsmanRefusals(e).map((r) => ({ ...r, participantIndex: null })), isStale: isChanged }`.

- [ ] **Step 1: Escreva os testes**

`src/lib/ombudsman.test.ts`:

```ts
const EVERY_CODE = [
  // contrato §1–§7
  "feature_disabled", "missing_role", "manifestation_changed", "destination_without_responder", "complement_already_used",
  "complement_not_allowed", "already_extended", "forwarding_pending", "identification_mode", "payload_too_large", "rate_limited",
  // Divergência D11: códigos que o contrato implica sem nome
  "forwarding_changed", "resolution_required", "resolution_not_allowed", "subject_required", "subject_not_leaf", "subject_depth",
  "duplicate_required", "external_body_required", "justification_required", "interim_not_allowed", "receipt_not_allowed",
  "reopen_request_not_pending", "name_taken", "already_responder", "report_published", "link_expired", "person_not_found",
  "citizen_not_verified", "attachment_limit", "attachment_type", "attachment_too_large", "destination_inactive", "not_responder"
];
it("todo código tem frase própria; código novo aparece cru", () => {
  for (const code of EVERY_CODE) {
    expect(OMBUDSMAN_ERRORS[code], code).toBeTruthy();
    expect(ombudsmanError(new ApiError(422, { error: code }, "x"))).toBe(OMBUDSMAN_ERRORS[code]);
  }
  expect(ombudsmanError(new ApiError(422, { error: "brand_new" }, "x"))).toBe("recusado pelo servidor (brand_new)");
  expect(ombudsmanError(new ApiError(403, { error: "feature_disabled", feature: "ombudsman" }, "x"))).toBe(OMBUDSMAN_DISABLED);
  expect(ombudsmanError(new ApiError(404, { error: "feature_disabled" }, "x"))).toBe(OMBUDSMAN_DISABLED); // D1: aceita os dois
});
it("frases fixas dos códigos que mudam a tela", () => {
  expect(OMBUDSMAN_ERRORS.manifestation_changed).toBe(MANIFESTATION_CHANGED);
  expect(OMBUDSMAN_ERRORS.destination_without_responder).toBe("este destino não tem respondente ativo — cadastre um na Configuração da ouvidoria");
  expect(OMBUDSMAN_ERRORS.forwarding_pending).toBe("há encaminhamento pendente — cancele ou aguarde a resposta");
  expect(OMBUDSMAN_ERRORS.complement_already_used).toBe("o complemento já foi pedido neste ciclo");
  expect(OMBUDSMAN_ERRORS.already_extended).toBe("a prorrogação já foi usada neste ciclo");
  expect(EXTRACT_NOTICE).toBe("Não escreva o nome nem dados de quem fez a manifestação.");
});
it("papéis: o ouvidor é também técnico; respondente só responde", () => {
  expect(ombudsmanRoles(ombudsmanUser([ "ombudsman" ]))).toEqual({ lead: true, staff: true, responder: false, admin: false });
  expect(ombudsmanRoles(ombudsmanUser([ "ombudsman_staff" ]))).toEqual({ lead: false, staff: true, responder: false, admin: false });
  expect(ombudsmanRoles(ombudsmanUser([ "ombudsman_responder" ])).staff).toBe(false);
  expect(canUseOmbudsman(ombudsmanUser([ "ombudsman" ], []))).toBe(false);
  expect(canUseOmbudsman({ ...ombudsmanUser([ "ombudsman" ]), operator: true })).toBe(false);
});
it("matriz estado × papel do §2", () => {
  const staff = ombudsmanRoles(ombudsmanUser([ "ombudsman_staff" ]));
  const lead = ombudsmanRoles(ombudsmanUser([ "ombudsman" ]));
  expect(availableActions(manifestation({ state: "received" }), staff)).toEqual([ "classify", "interim_response" ]);
  expect(availableActions(manifestation({ state: "in_analysis" }), staff))
    .toEqual([ "classify", "forward", "request_complement", "interim_response", "final_response" ]);
  expect(availableActions(manifestation({ state: "in_analysis" }), lead))
    .toEqual([ "classify", "forward", "request_complement", "extend", "interim_response", "final_response", "archive" ]);
  expect(availableActions(manifestation({ state: "forwarded" }), staff)).toEqual([ "forward", "interim_response" ]);
  expect(availableActions(manifestation({ state: "awaiting_complement" }), lead)).toEqual([ "complement", "extend", "interim_response", "archive" ]);
  expect(availableActions(manifestation({ state: "answered" }), lead)).toEqual([ "reopen", "change_resolution" ]);
  expect(availableActions(manifestation({ state: "answered", kind: "praise" }), lead)).toEqual([ "reopen" ]);
  expect(availableActions(manifestation({ state: "archived" }), staff)).toEqual([]);
  // anônima: nem complemento nem intermediária
  expect(availableActions(preserved({ identification_mode: "anonymous", state: "in_analysis" }), staff)).toEqual([ "classify", "forward", "final_response" ]);
});
it("motivos de bloqueio", () => {
  expect(blockedReason(manifestation({ pending_forwardings: 1 }), "final_response")).toBe(OMBUDSMAN_ERRORS.forwarding_pending);
  expect(blockedReason(manifestation({ complement_used: true }), "request_complement")).toBe(OMBUDSMAN_ERRORS.complement_already_used);
  expect(blockedReason(manifestation({ extended: true }), "extend")).toBe(OMBUDSMAN_ERRORS.already_extended);
  expect(blockedReason(manifestation(), "final_response")).toBeNull();
});
it("registro pela equipe: anônima só denúncia e sem pessoa; texto 20–4000; OuvidorSUS com número", () => {
  const base = { kind: "complaint", mode: "identified", channel: "in_person", ouvidorsusNumber: "", person: null, manual: { name: "", contact: "", cpf: "" },
    body: "curto", healthUnitId: null, files: [] } as NewDraft;
  expect(newManifestationProblems(base).map((p) => p.path)).toEqual([ "person", "body" ]);
  expect(newManifestationProblems({ ...base, mode: "anonymous", body: "x".repeat(20) }).map((p) => p.path)).toEqual([ "identification_mode" ]);
  expect(newManifestationProblems({ ...base, kind: "report", mode: "anonymous", body: "x".repeat(20), manual: { name: "Ana", contact: "", cpf: "" } })
    .map((p) => p.path)).toEqual([ "manifestant_name" ]);
  expect(newManifestationProblems({ ...base, channel: "ouvidorsus", manual: { name: "Ana Reis", contact: "41 3333-0000", cpf: "" }, body: "x".repeat(20) })
    .map((p) => p.path)).toEqual([ "ouvidorsus_number" ]);
  expect(newManifestationProblems({ ...base, manual: { name: "Ana Reis", contact: "", cpf: "123" }, body: "x".repeat(20) })
    .map((p) => p.path)).toEqual([ "manifestant_cpf" ]);
});
it("encaminhar e concluir", () => {
  expect(forwardProblems({ destination: null, extract: "x".repeat(19) }, options()).map((p) => p.path)).toEqual([ "destination", "extract" ]);
  expect(forwardProblems({ destination: { kind: "department", id: "dep2" }, extract: "x".repeat(20) }, options())[0].text)
    .toBe(OMBUDSMAN_ERRORS.destination_without_responder);
  expect(finalResponseProblems({ text: "x".repeat(20), resolution: null }, "complaint").map((p) => p.path)).toEqual([ "resolution" ]);
  expect(finalResponseProblems({ text: "x".repeat(20), resolution: "resolved" }, "praise").map((p) => p.path)).toEqual([ "resolution" ]);
  expect(archiveProblems({ reason: "duplicate", duplicateOfId: null, externalBodyName: "" }).map((p) => p.path)).toEqual([ "duplicate_of_id" ]);
  expect(archiveProblems({ reason: "forwarded_elsewhere", duplicateOfId: null, externalBodyName: "M" }).map((p) => p.path)).toEqual([ "external_body_name" ]);
});
it("prazos e anexos", () => {
  expect(settingsProblems(settings33({ response_days: 0, anonymous_daily_cap: 1001, warning_days: 2.5 })).map((p) => p.path))
    .toEqual([ "response_days", "warning_days", "anonymous_daily_cap" ]);
  const big = new File([ new Uint8Array(10 * 1024 * 1024 + 1) ], "a.pdf", { type: "application/pdf" });
  const gif = new File([ "x" ], "a.gif", { type: "image/gif" });
  expect(attachmentProblems([ big, gif ]).map((p) => p.text)).toEqual([ OMBUDSMAN_ERRORS.attachment_too_large, OMBUDSMAN_ERRORS.attachment_type ]);
  const pdf = () => new File([ "x" ], "a.pdf", { type: "application/pdf" });
  expect(attachmentProblems([ pdf(), pdf(), pdf(), pdf() ])[0].text).toBe(OMBUDSMAN_ERRORS.attachment_limit);
});
it("rótulos crus para código desconhecido", () => {
  expect(kindLabel("simplification")).toBe("simplification");
  expect(eventKindLabel("ledger_rewritten")).toBe("ledger_rewritten");
  expect(colorLabel("yellow")).toBe("perto do prazo");
});
```

Em `src/modules/psychosocial/useFormProblems.test.tsx`, um caso novo: com `opts.refusals` e `opts.isStale`, a recusa vai pelo tradutor recebido e o `stale` é chamado quando `isStale(err)`; sem `opts`, os casos que já existem seguem iguais.

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod33 src/lib/ombudsman.test.ts src/modules/psychosocial/useFormProblems.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `ombudsman.ts`: cabeçalho no molde de `src/lib/psychosocial.ts` (o que é do api, o que a tela só não envia; nenhuma frase interpola nome, CPF, contato, texto ou extrato). Frases de `OMBUDSMAN_ERRORS` (as do teste exatas; as demais em minúscula: `missing_role` → `OMBUDSMAN_NO_ROLE`; `complement_not_allowed` "a denúncia anônima não recebe pedido de complemento"; `identification_mode` "anônima só para denúncia, e sem dados de pessoa"; `payload_too_large` "o texto ou o anexo passou do tamanho aceito"; `rate_limited` "muitas tentativas seguidas — espere um pouco"; `forwarding_changed` "o encaminhamento mudou enquanto a tela estava aberta — os dados foram relidos"; `resolution_required` "informe se a demanda foi resolvida"; `resolution_not_allowed` "elogio e sugestão não têm resolutividade"; `subject_required` "escolha o assunto"; `subject_not_leaf` "escolha um subassunto — este assunto tem subdivisões"; `subject_depth` "subassunto não pode ter subassunto"; `duplicate_required` "informe o protocolo da manifestação original"; `external_body_required` "informe o órgão para onde foi encaminhada"; `justification_required` "escreva a justificativa (pelo menos 10 caracteres)"; `interim_not_allowed` "a denúncia anônima não recebe resposta intermediária"; `receipt_not_allowed` `NO_RECEIPT_ANONYMOUS` em minúscula; `reopen_request_not_pending` "este pedido de reabertura já foi decidido"; `name_taken` "já existe um item com esse nome"; `already_responder` "esta pessoa já responde por este destino"; `report_published` "o relatório deste ano já foi publicado e não muda"; `link_expired` "o endereço do anexo venceu — abra de novo"; `person_not_found` "ninguém com este CPF — registre nome e contato na manifestação"; `citizen_not_verified` "este CPF ainda não foi confirmado — registre nome e contato na manifestação"; `attachment_limit` "até 3 anexos"; `attachment_type` "o anexo tem de ser PDF, JPEG ou PNG"; `attachment_too_large` "cada anexo tem até 10 MB"; `destination_inactive` "este destino foi desativado"; `not_responder` "você não responde por este destino"; `feature_disabled` → `OMBUDSMAN_DISABLED`). `ombudsmanError`: `feature_disabled` com status 403 **ou** 404 (D1) — `featureDisabledKey` só reconhece 403, então o 404 é conferido aqui — com outra chave → `"<rótulo> está desligado nesta cidade"`; 401 → frase da sessão do `attendanceError`; código conhecido → a frase; outro → `recusado pelo servidor (<código>)`; fora do formato `^[a-z0-9_]{1,64}$` → "recusado pelo servidor" (sem eco, como `psyRefusals`). `ombudsmanRefusals`: `problems[]` com `error` ou `code` e `path` (só no formato de `PATH_FORMAT`), cada um com a frase; sem `problems`, uma recusa do topo. `availableActions` segue a ordem `classify, forward, request_complement, complement, extend, interim_response, final_response, archive, reopen, change_resolution` e filtra por papel (`extend`, `archive`, `reopen`, `change_resolution` só `lead`), por modo (`request_complement`, `complement` e `interim_response` nunca no anônimo) e por tipo (`change_resolution` só com `needsResolution`). As ações que só têm `blockedReason` ficam na lista (o botão aparece desabilitado com o motivo).
- `useFormProblems.tsx`: o hook aceita `opts`; `psyRefusals` e `isStale` viram os padrões. Nenhuma tela do CAPS muda.

- [ ] **Step 4: Rode e veja passar; regressão do CAPS; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod33 src/lib/ombudsman.test.ts src/modules/psychosocial src/modules/Caps && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/ombudsman.ts src/lib/ombudsman.test.ts src/modules/psychosocial/useFormProblems.tsx src/modules/psychosocial/useFormProblems.test.tsx
$G commit -m "feat: add ombudsman screen rules, action matrix and error texts" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** `dashboard · dashboard-mod33/feat/mod-33-ombudsman · Task 2 · regras de tela e matriz de ações do §2 · <códigos do api que faltavam na lista>`.

---

### Task 3: Grupo "Ouvidoria", telas-portão e papéis no convite (F-33.1)

**Files:**
- Modify: `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/shell/catalog.ts`, `src/App.tsx`, `src/lib/team.ts`, `src/lib/team.test.ts`
- Create: `src/modules/ombudsman/OmbudsmanGate.tsx`, cascas `src/modules/OmbudsmanManifestations.tsx`, `OmbudsmanNew.tsx`, `OmbudsmanReplies.tsx`, `OmbudsmanPanel.tsx`, `OmbudsmanAnnualReport.tsx`, `OmbudsmanSettings.tsx`, `Departments.tsx` (o conteúdo vem nas Tasks 4–12)
- Test: `src/shell/modules.ombudsman.test.ts`, `src/modules/ombudsman/OmbudsmanGate.test.tsx`

**Interfaces:**
- Consumes: `canUseOmbudsman`, `ombudsmanRoles`, `OMBUDSMAN_DISABLED`, `OMBUDSMAN_NO_ROLE` (Task 2).
- Produces: `ModuleId` ganha `"ombudsman-manifestations" | "ombudsman-new" | "ombudsman-replies" | "ombudsman-panel" | "ombudsman-annual-report" | "ombudsman-settings" | "departments"`; grupo `{ label: "Ouvidoria", items: [ { id: "ombudsman-manifestations", label: "Caixa da ouvidoria", icon: "☰" }, { id: "ombudsman-new", label: "Nova manifestação", icon: "✎" }, { id: "ombudsman-replies", label: "Respostas à ouvidoria", icon: "↩" }, { id: "ombudsman-panel", label: "Painel da ouvidoria", icon: "▦" }, { id: "ombudsman-annual-report", label: "Relatório anual", icon: "▤" }, { id: "ombudsman-settings", label: "Configuração da ouvidoria", icon: "⚙" }, { id: "departments", label: "Setores da secretaria", icon: "⌂" } ] }` depois de "Urgência"; `OmbudsmanGate({ title, sub, allow(roles): boolean, children(roles): ReactNode })`; o handoff `useOmbudsmanHandoff(active)` no molde de `useCapsHandoff` (`{ manifestationId: string | null; openManifestation(id: string): void }`, apagado ao sair da Caixa); `INVITE_ROLES` com `{ role: "ombudsman", label: "Ouvidor" }`, `{ role: "ombudsman_staff", label: "Equipe da ouvidoria" }`, `{ role: "ombudsman_responder", label: "Ponto de resposta da ouvidoria" }` antes de `municipal_admin`; `ombudsman` em `PRIVILEGED_ROLES` (D3).

- [ ] **Step 1: Escreva os testes**

`src/shell/modules.ombudsman.test.ts`:

```ts
const ids = (u: Parameters<typeof navGroupsFor>[0]) => navGroupsFor(u).find((g) => g.label === "Ouvidoria")?.items.map((i) => i.id) ?? [];
const STAFF = [ "ombudsman-manifestations", "ombudsman-new", "ombudsman-panel" ];
describe("menu da ouvidoria (módulo 33)", () => {
  it("técnico: caixa, nova e painel; ouvidor: também relatório e configuração", () => {
    expect(ids(ombudsmanUser([ "ombudsman_staff" ]))).toEqual(STAFF);
    expect(ids(ombudsmanUser([ "ombudsman" ]))).toEqual([ ...STAFF, "ombudsman-annual-report", "ombudsman-settings" ]);
  });
  it("respondente só as respostas; respondente e técnico, as duas coisas", () => {
    expect(ids(ombudsmanUser([ "ombudsman_responder" ]))).toEqual([ "ombudsman-replies" ]);
    expect(ids(ombudsmanUser([ "ombudsman_responder", "ombudsman_staff" ]))).toEqual([ "ombudsman-manifestations", "ombudsman-new", "ombudsman-replies", "ombudsman-panel" ]);
  });
  it("admin só os setores; sem o interruptor o grupo some", () => {
    expect(ids(ombudsmanUser([ "municipal_admin" ]))).toEqual([ "departments" ]);
    expect(ids(ombudsmanUser([ "ombudsman", "municipal_admin" ], []))).toEqual([]);
    expect(navGroupsFor(ombudsmanUser([ "health_professional" ])).some((g) => g.label === "Ouvidoria")).toBe(false);
  });
  it("catálogo: grupo e telas com ícone e descrição", () => {
    expect(CATALOG.groups["Ouvidoria"].description).toBe("Manifestações do cidadão, encaminhamentos, prazos e relatório anual");
    expect(CATALOG.screens["ombudsman-manifestations"].description).toBe("Manifestações por estado e cor de prazo");
    expect(CATALOG.screens["ombudsman-replies"].description).toBe("Encaminhamentos do seu setor ou unidade");
    expect(CATALOG.screens["departments"].description).toBe("Setores da secretaria que recebem encaminhamentos");
  });
});
```

`src/modules/ombudsman/OmbudsmanGate.test.tsx` (mock do api com `fetchCurrentSession`): sem o interruptor mostra `OMBUDSMAN_DISABLED` e não chama nada do api além da sessão; papel errado mostra `OMBUDSMAN_NO_ROLE`; papel certo mostra o corpo.

Em `src/lib/team.test.ts`: o título passa a "oferece os 26 papéis…", a lista ordenada ganha `"ombudsman", "ombudsman_responder", "ombudsman_staff"`, os rótulos acima, `isPrivilegedRole("ombudsman")` `true`, `isPrivilegedRole("ombudsman_staff")` e `isPrivilegedRole("ombudsman_responder")` `false` (contrato §1; ver D3). Em `src/shell/modules.test.ts`: os deslocamentos `NAV_GROUPS.length - 21` e `- 20` sobem 1 e as duas listas ganham `expect(groups.some((g) => g.label === "Ouvidoria")).toBe(false)`.

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod33 src/shell/modules.ombudsman.test.ts src/modules/ombudsman/OmbudsmanGate.test.tsx src/lib/team.test.ts`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `modules.ts`: ids, grupo e, em `navGroupsFor`, depois de `const wrOn = canUseWaitingRoom(user);`: `// Módulo 33 (ADR 0051): ombudsman na sessão; o respondente sem ombudsman_staff só responde (contrato §6)` + `const ombOn = canUseOmbudsman(user); const omb = ombudsmanRoles(user);` e, antes de `integrations`: `if (item.id === "ombudsman-manifestations" || item.id === "ombudsman-new" || item.id === "ombudsman-panel") return ombOn && omb.staff;`, `if (item.id === "ombudsman-replies") return ombOn && omb.responder;`, `if (item.id === "ombudsman-annual-report" || item.id === "ombudsman-settings") return ombOn && omb.lead;`, `if (item.id === "departments") return ombOn && isAdmin;`.
- `catalog.ts`: grupo com `IconMessageReport`; `ombudsman-manifestations` `IconInbox`, `ombudsman-new` `IconFilePlus`, `ombudsman-replies` `IconMailForward`, `ombudsman-panel` `IconChartBar`, `ombudsman-annual-report` `IconFileReport`, `ombudsman-settings` `IconSettings`, `departments` `IconSitemap`, com as descrições do teste (as outras: nova "Registro presencial, por telefone ou do OuvidorSUS, com comprovante"; painel "Volume, prazos, resolutividade e satisfação"; relatório "Relatório de gestão anual para publicar"; configuração "Respondentes, assuntos e prazos").
- `App.tsx`: os sete `case` (as telas com escrita sensível recebem `onGoToSecurity={() => setActive("security")}`), e `useOmbudsmanHandoff(active)` no molde de `useCapsHandoff`: `ombudsman-new` recebe `onOpenManifestation={(id) => { omb.openManifestation(id); setActive("ombudsman-manifestations"); }}` e `ombudsman-manifestations` recebe `initialId={omb.manifestationId}`.
- `OmbudsmanGate.tsx`: molde do `PsychosocialGate` sem escolha de unidade (`PageHeader` + `EmptyState` com as frases; corpo com `children(roles)`).
- `team.ts`: os três papéis e `ombudsman` em `PRIVILEGED_ROLES`, com o comentário "Módulo 33 (ADR 0051): o ouvidor vê a identidade preservada — privilegiado (step-up para conceder); equipe e respondente não (contrato §1)".
- Cascas: cada tela com `OmbudsmanGate` e `EmptyState`.

- [ ] **Step 4: Rode e veja passar; regressão do shell e da Equipe; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod33 src/shell src/modules/ombudsman src/lib/team.test.ts src/modules/Team.test.tsx src/App.navigation.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/shell/modules.ts src/shell/modules.test.ts src/shell/modules.ombudsman.test.ts src/shell/catalog.ts src/App.tsx src/lib/team.ts src/lib/team.test.ts \
  src/modules/ombudsman/OmbudsmanGate.tsx src/modules/ombudsman/OmbudsmanGate.test.tsx src/modules/OmbudsmanManifestations.tsx src/modules/OmbudsmanNew.tsx \
  src/modules/OmbudsmanReplies.tsx src/modules/OmbudsmanPanel.tsx src/modules/OmbudsmanAnnualReport.tsx src/modules/OmbudsmanSettings.tsx src/modules/Departments.tsx
$G commit -m "feat: add the ombudsman menu group, screen gates and the three ombudsman roles" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** `dashboard · dashboard-mod33/feat/mod-33-ombudsman · Task 3 · grupo Ouvidoria e papéis no convite (ombudsman privilegiado) · D3 se o api tornar ombudsman_staff privilegiado`.

---

### Task 4: Caixa da ouvidoria — filtros, cor de prazo e cursor (F-33.5, F-33.6)

**Files:**
- Create: `src/modules/ombudsman/ManifestationList.tsx`, `src/modules/ombudsman/ManifestationFilters.tsx`, `src/modules/ombudsman/DeadlineTag.tsx`
- Modify: `src/modules/OmbudsmanManifestations.tsx`
- Test: `src/modules/ombudsman/ManifestationList.test.tsx`

**Interfaces:**
- Consumes: `listOmbudsmanManifestations`, `getOmbudsmanOptions` (Task 1); rótulos, `COLOR_TONE`, `OMB_LIST_KEY`, `OMB_OPTIONS_KEY`, `STATIC_QUERY` (Task 2); `DataTable`, `Tag`.
- Produces: `DeadlineTag({ color, dueOn })` ("perto do prazo · 03/12" com o tom; nada quando `color` é nulo); `ManifestationList({ onOpen(id: string): void })`; `OmbudsmanManifestations({ initialId?: string | null; now?: Date; onGoToSecurity? })` — lista e, com um id escolhido, a `ManifestationView` (Task 5) no lugar, com "Voltar à caixa".

- [ ] **Step 1: Escreva os testes**

```tsx
vi.mock("../../lib/api", async (orig) => ({ ...(await orig<typeof api>()), fetchCurrentSession: vi.fn(),
  listOmbudsmanManifestations: vi.fn(), getOmbudsmanOptions: vi.fn(), getOmbudsmanManifestation: vi.fn() }));
afterEach(() => { cleanup(); vi.clearAllMocks(); });
function mount(user = ombudsmanUser([ "ombudsman_staff" ])) {
  vi.mocked(api.fetchCurrentSession).mockResolvedValue(user);
  vi.mocked(api.getOmbudsmanOptions).mockResolvedValue(options());
  return renderWithProviders(<OmbudsmanManifestations now={NOW33} />);
}
it("lista com protocolo, tipo, modo, estado, assunto, unidade e prazo com texto da cor", async () => {
  vi.mocked(api.listOmbudsmanManifestations).mockResolvedValue(page({ items: [
    listItem(), listItem({ id: "m3", protocol: "2026-000125", color: "red", due_on: "2026-11-09", urgent: true, pending_forwardings: 2 }),
    listItem({ id: "m4", protocol: "2026-000126", state: "awaiting_complement", color: "paused" }),
    listItem({ id: "m5", protocol: "2026-000127", state: "answered", color: null, due_on: null })
  ] }));
  mount();
  const row = (await screen.findByText("2026-000125")).closest("[role='row']") as HTMLElement;
  expect(within(row).getByText(/vencida/)).not.toBeNull();
  expect(within(row).getByText("urgente")).not.toBeNull();
  expect(within(row).getByText("2 encaminhamentos pendentes")).not.toBeNull();
  expect(screen.getByText(/prazo suspenso/)).not.toBeNull();
  const done = screen.getByText("2026-000127").closest("[role='row']") as HTMLElement;
  expect(within(done).queryByText(/prazo/)).toBeNull();
});
it("a lista não mostra texto nem identidade, mesmo que o api mande", async () => {
  vi.mocked(api.listOmbudsmanManifestations).mockResolvedValue(page({ items: [
    { ...listItem(), body: "Fui à UBS Centro três vezes", identity: { name: "Maria Lima" }, manifestant_name: "Maria Lima" } as never
  ] }));
  mount();
  await screen.findByText("2026-000123");
  expect(screen.queryByText(/Maria Lima|três vezes/)).toBeNull();
});
it("filtros viram a consulta; protocolo só no campo q; trocar filtro recomeça do início", async () => {
  vi.mocked(api.listOmbudsmanManifestations).mockResolvedValue(page({ next_before: "c2" }));
  mount();
  await screen.findByText("2026-000123");
  fireEvent.click(screen.getByLabelText("vencida"));
  fireEvent.change(screen.getByLabelText("Assunto"), { target: { value: "s1" } });
  fireEvent.change(screen.getByLabelText("Protocolo"), { target: { value: "2026-000123" } });
  fireEvent.click(screen.getByRole("button", { name: "Filtrar" }));
  await waitFor(() => expect(vi.mocked(api.listOmbudsmanManifestations).mock.calls.at(-1)![0])
    .toMatchObject({ color: [ "red" ], subject_id: "s1", q: "2026-000123", before: null }));
});
it("Carregar mais usa o cursor e acrescenta", async () => {
  vi.mocked(api.listOmbudsmanManifestations).mockResolvedValueOnce(page({ next_before: "c2" }))
    .mockResolvedValueOnce(page({ items: [ listItem({ id: "m8", protocol: "2026-000099" }) ], next_before: null }));
  mount();
  fireEvent.click(await screen.findByRole("button", { name: "Carregar mais" }));
  expect(await screen.findByText("2026-000099")).not.toBeNull();
  expect(screen.getByText("2026-000123")).not.toBeNull();
  expect(vi.mocked(api.listOmbudsmanManifestations).mock.calls[1][0].before).toBe("c2");
  expect(screen.queryByRole("button", { name: "Carregar mais" })).toBeNull();
});
it("clicar abre a manifestação; respondente sem técnico nem chega a ler", async () => {
  vi.mocked(api.listOmbudsmanManifestations).mockResolvedValue(page());
  vi.mocked(api.getOmbudsmanManifestation).mockResolvedValue(manifestation());
  mount();
  fireEvent.click(await screen.findByRole("button", { name: "Abrir 2026-000123" }));
  expect(await screen.findByRole("heading", { name: /2026-000123/ })).not.toBeNull();
  cleanup();
  mount(ombudsmanUser([ "ombudsman_responder" ]));
  expect(await screen.findByText(OMBUDSMAN_NO_ROLE)).not.toBeNull();
  expect(api.listOmbudsmanManifestations).toHaveBeenCalledTimes(1);
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod33 src/modules/ombudsman/ManifestationList.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `ManifestationFilters.tsx`: grupos de caixas para estado, cor (as quatro, com o texto), tipo e canal; seletores de assunto (só os de primeiro nível ativos, mais "todos") e unidade (das `options`); campo "Protocolo" (`AAAA-NNNNNN`, sem outro texto); botão "Filtrar" (a consulta só muda no clique, não a cada tecla).
- `ManifestationList.tsx`: `useInfiniteQuery` com chave `[ OMB_LIST_KEY, filtros ]`, `STATIC_QUERY`, `getNextPageParam: (p) => p.next_before`; o `DataTable` monta cada linha **campo a campo** (`protocol`, `kind`, `identification_mode`, `channel`, `state`, `subject`, `health_unit`, `urgent`, `received_at`, `due_on`, `color`, `pending_forwardings`), nunca espalhando o item; colunas com `card: "title"` no protocolo e `"actions"` no "Abrir <protocolo>"; "urgente" como `Tag tone="down"`; `pending_forwardings > 0` com o plural; "Carregar mais" só com `next_before`. Vazio: "Nenhuma manifestação com esses filtros".
- `OmbudsmanManifestations.tsx`: `OmbudsmanGate` com `allow: (r) => r.staff`; o id escolhido (ou o `initialId` do handoff) abre a `ManifestationView` (casca com o título até a Task 5); "Voltar à caixa" volta à lista e remove o detalhe do cache (`queryClient.removeQueries({ queryKey: [ OMB_DETAIL_KEY ] })`).

- [ ] **Step 4: Rode e veja passar; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod33 src/modules/ombudsman && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/OmbudsmanManifestations.tsx src/modules/ombudsman/ManifestationList.tsx src/modules/ombudsman/ManifestationFilters.tsx \
  src/modules/ombudsman/DeadlineTag.tsx src/modules/ombudsman/ManifestationList.test.tsx
$G commit -m "feat: add the ombudsman inbox with filters, deadline colors and cursor paging" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** `dashboard · dashboard-mod33/feat/mod-33-ombudsman · Task 4 · caixa da ouvidoria · forma do cursor e dos filtros (D12, D14)`.

---

### Task 5: Manifestação — dados, identidade por modo, texto, linha do tempo, encaminhamentos e anexos (F-33.5, F-33.7)

**Files:**
- Create: `src/modules/ombudsman/ManifestationView.tsx`, `src/modules/ombudsman/IdentityBlock.tsx`, `src/modules/ombudsman/Timeline.tsx`, `src/modules/ombudsman/Forwardings.tsx`, `src/modules/ombudsman/Attachments.tsx`, `src/modules/ombudsman/openFile.ts`
- Test: `src/modules/ombudsman/ManifestationView.test.tsx`

**Interfaces:**
- Consumes: `getOmbudsmanManifestation`, `getOmbudsmanAttachmentUrl`, `fetchOmbudsmanFile` (Task 1); rótulos, `IDENTITY_PRESERVED`, `ANONYMOUS_NO_IDENTITY`, `NOMINAL_QUERY`, `OMB_DETAIL_KEY`, `ombudsmanError` (Task 2); `DeadlineTag` (Task 4); `POPUP_BLOCKED` (`src/modules/documents/printDocument.ts`); `KeyValue`, `Panel`, `Tag`.
- Produces: `ManifestationView({ id, roles, now, onBack, onGoToSecurity? })`; `IdentityBlock({ m, roles, onReveal(): void })` (o botão só para `canSeeIdentityButton`; o diálogo vem na Task 8); `Timeline({ events })`; `Forwardings({ m, roles, onCancel(f): void })` (o cancelar vem na Task 6); `Attachments({ m, roles, onProtected(a): void })`; `openFile(getUrl: () => Promise<string>, errorText): Promise<string | null>` (janela aberta no clique, antes do `await`, como `printDocument`; `blob:` com `revokeObjectURL` em 60 s; janela bloqueada → `POPUP_BLOCKED` sem ler o arquivo).

- [ ] **Step 1: Escreva os testes**

```tsx
vi.mock("../../lib/api", async (orig) => ({ ...(await orig<typeof api>()), fetchCurrentSession: vi.fn(),
  getOmbudsmanManifestation: vi.fn(), getOmbudsmanAttachmentUrl: vi.fn(), fetchOmbudsmanFile: vi.fn() }));
function mount(m: OmbudsmanManifestation, roles = [ "ombudsman_staff" ]) {
  const user = ombudsmanUser(roles);
  vi.mocked(api.fetchCurrentSession).mockResolvedValue(user);
  vi.mocked(api.getOmbudsmanManifestation).mockResolvedValue(m);
  return renderWithProviders(<ManifestationView id={m.id} roles={ombudsmanRoles(user)} now={NOW33} onBack={() => {}} />);
}
it("identificada: dados, identidade mascarada, texto e prazo", async () => {
  mount(manifestation({ urgent: true, referred_professional: { id: "p1", name: "Dr. Caio" } }));
  expect(await screen.findByRole("heading", { name: "Manifestação 2026-000123" })).not.toBeNull();
  expect(screen.getByText("Reclamação")).not.toBeNull();
  expect(screen.getByText("Maria Lima")).not.toBeNull();
  expect(screen.getByText("***.***.***-25")).not.toBeNull();
  expect(screen.getByText(/não havia losartana/)).not.toBeNull();
  expect(screen.getByText("Atendimento › Demora")).not.toBeNull();
  expect(screen.getByText("Dr. Caio")).not.toBeNull();
  expect(screen.queryByRole("button", { name: "Ver identidade" })).toBeNull();
});
it("preservada: o técnico vê a frase e nada mais, mesmo com dado vazado no detalhe", async () => {
  mount({ ...preserved(), name: "Joana Prado", manifestant_cpf: "529.982.247-25", identity: "preserved" } as never);
  expect(await screen.findByText(IDENTITY_PRESERVED)).not.toBeNull();
  expect(screen.queryByText(/Joana|529\.982/)).toBeNull();
  expect(screen.queryByRole("button", { name: "Ver identidade" })).toBeNull();
});
it("preservada: o ouvidor vê o botão; anônima sem dados não oferece nada", async () => {
  mount(preserved(), [ "ombudsman" ]);
  expect(await screen.findByRole("button", { name: "Ver identidade" })).not.toBeNull();
  cleanup();
  mount(preserved({ identification_mode: "anonymous", identity_available: false }), [ "ombudsman" ]);
  expect(await screen.findByText(ANONYMOUS_NO_IDENTITY)).not.toBeNull();
  expect(screen.queryByRole("button", { name: "Ver identidade" })).toBeNull();
});
it("linha do tempo com quem fez, o que vai ao cidadão marcado, e código desconhecido cru", async () => {
  mount(manifestation({ events: [ event(), event({ id: "e3", kind: "interim_response", actor: "staff", actor_name: "Ana Souza",
    text: "Estamos verificando com a farmácia.", visible_to_citizen: true }), event({ id: "e4", kind: "brand_new_event", actor: "system" }) ] }));
  const item = (await screen.findByText("Estamos verificando com a farmácia.")).closest("li") as HTMLElement;
  expect(within(item).getByText("Resposta intermediária")).not.toBeNull();
  expect(within(item).getByText("Ana Souza")).not.toBeNull();
  expect(within(item).getByText("enviada ao cidadão")).not.toBeNull();
  expect(screen.getByText("brand_new_event")).not.toBeNull();
});
it("encaminhamentos com destino, extrato, prazo e resposta", async () => {
  mount(manifestation({ state: "forwarded", pending_forwardings: 1, forwardings: [ forwarding(),
    forwarding({ id: "f2", state: "replied", reply: "Reposição feita.", replied_by_name: "Bruno Melo", color: null }) ] }));
  expect(await screen.findByText("Setor: Assistência Farmacêutica")).not.toBeNull();
  expect(screen.getAllByText(/falta de losartana/).length).toBe(2);
  expect(screen.getByText("Reposição feita.")).not.toBeNull();
  expect(screen.getByText("aguardando resposta")).not.toBeNull();
});
it("anexo da identificada abre sem justificativa; janela bloqueada não lê", async () => {
  const win = { location: { href: "" }, close: vi.fn() };
  const open = vi.spyOn(window, "open").mockReturnValue(win as never);
  vi.mocked(api.getOmbudsmanAttachmentUrl).mockResolvedValue("/ombudsman/files/tok1");
  vi.mocked(api.fetchOmbudsmanFile).mockResolvedValue({ blob: new Blob([ "x" ]), filename: null });
  URL.createObjectURL = vi.fn(() => "blob:1"); URL.revokeObjectURL = vi.fn();
  mount(manifestation({ attachments: [ { id: "a1", uploaded_by: "citizen", content_type: "application/pdf", created_at: "2026-11-03T09:12:00-03:00" } ] }));
  fireEvent.click(await screen.findByRole("button", { name: "Abrir anexo 1 (PDF)" }));
  await waitFor(() => expect(win.location.href).toBe("blob:1"));
  expect(vi.mocked(api.getOmbudsmanAttachmentUrl).mock.calls[0]).toEqual([ "a1" ]);
  open.mockReturnValue(null);
  fireEvent.click(screen.getByRole("button", { name: "Abrir anexo 1 (PDF)" }));
  expect(await screen.findByText(POPUP_BLOCKED)).not.toBeNull();
  expect(api.getOmbudsmanAttachmentUrl).toHaveBeenCalledTimes(1);
});
it("o detalhe sai do cache ao sair (leitura nominal, gcTime 0)", async () => {
  const { unmount } = mount(manifestation());
  await screen.findByText(/não havia losartana/);
  unmount();
  expect(queryClientOf().getQueryCache().findAll({ queryKey: [ OMB_DETAIL_KEY ] }).length).toBe(0);
});
```

(`queryClientOf()`: confira na Task 0 como os testes do 31 leem o cliente do `renderWithProviders`; se ele não for exposto, o teste passa um `QueryClient` próprio pelo mesmo provedor.)

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod33 src/modules/ombudsman/ManifestationView.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `ManifestationView.tsx`: `useQuery([ OMB_DETAIL_KEY, id ], …, NOMINAL_QUERY)`; título "Manifestação <protocolo>"; `KeyValue` para tipo, modo, canal (com o número do OuvidorSUS), recebida em, prazo (`DeadlineTag`, e a data do complemento quando `awaiting_complement`), urgente, assunto ("pai › filho"), unidade, profissional, ciclo, prorrogada, resolutividade, motivo do arquivamento (com o protocolo original ou o órgão externo); `IdentityBlock`; o texto (`body`) num `Panel` "Texto da manifestação"; `Timeline`; `Forwardings`; `Attachments`; o pedido de reabertura (motivo, estado, decisão); e o lugar do `ActionsPanel` (Task 6).
- `IdentityBlock.tsx`: lê campo a campo; `identified` → `name`, `cpf_masked`, `phone_masked`, `contact` do `identity` (D15); outros modos → só a frase, e o botão "Ver identidade" para `canSeeIdentityButton`.
- `Timeline.tsx`: `<ol>` com rótulo do tipo, data, `actor_name` (ou "cidadão"/"sistema"), o texto quando há, e a marca "enviada ao cidadão" quando `visible_to_citizen`. `data` não é exibido.
- `Attachments.tsx`: "Anexo N (PDF|JPEG|PNG)" pelo `content_type`, "enviado pelo cidadão"/"pela equipe"; na identificada, `openFile(() => getOmbudsmanAttachmentUrl(a.id), ombudsmanError)`; na preservada/anônima, só o ouvidor vê o botão "Abrir anexo N (justificar)" que chama `onProtected(a)` (Task 8); o técnico vê "anexo protegido — só o ouvidor abre".
- `openFile.ts`: o molde de `printDocument` (janela no clique; `fetchOmbudsmanFile(url)`; `blob:`; 410 `link_expired` com a frase).

- [ ] **Step 4: Rode e veja passar; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod33 src/modules/ombudsman && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/ombudsman/ManifestationView.tsx src/modules/ombudsman/IdentityBlock.tsx src/modules/ombudsman/Timeline.tsx \
  src/modules/ombudsman/Forwardings.tsx src/modules/ombudsman/Attachments.tsx src/modules/ombudsman/openFile.ts src/modules/ombudsman/ManifestationView.test.tsx
$G commit -m "feat: show an ombudsman manifestation with identity by mode, timeline, forwardings and attachments" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** `dashboard · dashboard-mod33/feat/mod-33-ombudsman · Task 5 · detalhe da manifestação · forma do detalhe, eventos e anexos (D15–D18)`.

---

### Task 6: Ações do técnico — classificar, encaminhar, cancelar, complemento, respostas (F-33.5, F-33.6)

**Files:**
- Create: `src/modules/ombudsman/ActionsPanel.tsx`, `src/modules/ombudsman/useManifestationAction.ts`, `src/modules/ombudsman/actions/ClassifyForm.tsx`, `ForwardForm.tsx`, `ExtractField.tsx`, `RequestComplementForm.tsx`, `ComplementForm.tsx`, `ResponseForm.tsx`
- Modify: `src/modules/ombudsman/ManifestationView.tsx`, `src/modules/ombudsman/Forwardings.tsx`
- Test: `src/modules/ombudsman/ActionsStaff.test.tsx`

**Interfaces:**
- Consumes: `runOmbudsmanAction`, `cancelOmbudsmanForwarding`, `getOmbudsmanOptions`, `listOmbudsmanProfessionals`, `uploadOmbudsmanAttachment` (Task 1); `availableActions`, `blockedReason`, `forwardProblems`, `textActionProblems`, `finalResponseProblems`, `needsResolution`, `EXTRACT_NOTICE`, `ANONYMOUS_INTERNAL`, `MANIFESTATION_CHANGED`, `isChanged`, `ombudsmanRefusals` (Task 2); `useFormProblems` com `opts` (Task 2); `FrozenTextNotice`, `ReasonHint`, `ConfirmButton`, `ActionButton`.
- Produces: `ActionsPanel({ m, roles, onChanged(next: OmbudsmanManifestation | null): void })` — um botão por ação de `availableActions` (desabilitado com `blockedReason`), uma ação aberta por vez; `useManifestationAction(m, onChanged)` → `{ run(action, body): Promise<void>; changedNotice: string | null }` — põe `expected_state: m.state` em **todo** corpo, guarda `seq`, e no 409 `manifestation_changed` mostra a frase, pede **uma** releitura (`onChanged(null)` → `refetch`) e não reenvia; `ExtractField({ id, value, onChange, aria })` — `<textarea>` com `aria-describedby` = `<id>-extract-notice <id>-frozen` (+ o erro quando há), a frase `EXTRACT_NOTICE` e o `FrozenTextNotice`.

- [ ] **Step 1: Escreva os testes**

```tsx
vi.mock("../../lib/api", async (orig) => ({ ...(await orig<typeof api>()), fetchCurrentSession: vi.fn(), getOmbudsmanManifestation: vi.fn(),
  runOmbudsmanAction: vi.fn(), cancelOmbudsmanForwarding: vi.fn(), getOmbudsmanOptions: vi.fn(), listOmbudsmanProfessionals: vi.fn() }));
// mount(m, roles): fetchCurrentSession, getOmbudsmanOptions → options(), listOmbudsmanProfessionals → [], getOmbudsmanManifestation → m
it("classificar: subassunto obrigatório; o corpo leva expected_state e só o que mudou", async () => {
  vi.mocked(api.runOmbudsmanAction).mockResolvedValue(manifestation());
  mount(manifestation({ state: "received", subject: null }));
  fireEvent.click(await screen.findByRole("button", { name: "Classificar" }));
  fireEvent.change(screen.getByLabelText("Assunto"), { target: { value: "s1" } });
  fireEvent.click(screen.getByRole("button", { name: "Salvar classificação" }));
  expect(await screen.findByText(OMBUDSMAN_ERRORS.subject_not_leaf)).not.toBeNull();
  fireEvent.change(screen.getByLabelText("Subassunto"), { target: { value: "s11" } });
  fireEvent.click(screen.getByLabelText("Urgente"));
  fireEvent.click(screen.getByRole("button", { name: "Salvar classificação" }));
  await waitFor(() => expect(vi.mocked(api.runOmbudsmanAction).mock.calls[0])
    .toEqual([ "m1", "classify", { expected_state: "received", subject_id: "s11", urgent: true } ]));
});
it("encaminhar: avisos no extrato antes do envio, destino sem respondente indisponível, extrato curto não sai", async () => {
  vi.mocked(api.runOmbudsmanAction).mockResolvedValue(manifestation({ state: "forwarded" }));
  mount(manifestation());
  fireEvent.click(await screen.findByRole("button", { name: "Encaminhar" }));
  const extract = screen.getByLabelText("Extrato para o destino");
  expectFrozenNotice(extract);
  const ids = extract.getAttribute("aria-describedby")!.split(/\s+/).map((i) => document.getElementById(i)?.textContent);
  expect(ids).toContain(EXTRACT_NOTICE);
  expect((screen.getByRole("option", { name: /Regulação/ }) as HTMLOptionElement).disabled).toBe(true);
  fireEvent.change(screen.getByLabelText("Destino"), { target: { value: "department:dep1" } });
  fireEvent.change(extract, { target: { value: "x".repeat(19) } });
  fireEvent.click(screen.getByRole("button", { name: "Encaminhar agora" }));
  expect(api.runOmbudsmanAction).not.toHaveBeenCalled();
  fireEvent.change(extract, { target: { value: "Usuária relata falta de losartana na farmácia da UBS Centro." } });
  fireEvent.click(screen.getByRole("button", { name: "Encaminhar agora" }));
  fireEvent.click(screen.getByRole("button", { name: "Encaminhar agora" }));
  await waitFor(() => expect(api.runOmbudsmanAction).toHaveBeenCalledTimes(1));
  expect(vi.mocked(api.runOmbudsmanAction).mock.calls[0][2])
    .toEqual({ expected_state: "in_analysis", department_id: "dep1", extract: "Usuária relata falta de losartana na farmácia da UBS Centro." });
});
it("destination_without_responder do api: a frase no destino e o extrato mantido", async () => {
  vi.mocked(api.runOmbudsmanAction).mockRejectedValue(new ApiError(422, { error: "destination_without_responder" }, "x"));
  mount(manifestation());
  await forward("department:dep1", "Usuária relata falta de losartana na farmácia da UBS Centro.");
  expect(await screen.findByText(OMBUDSMAN_ERRORS.destination_without_responder)).not.toBeNull();
  expect((screen.getByLabelText("Extrato para o destino") as HTMLTextAreaElement).value).toMatch(/losartana/);
});
it("409 manifestation_changed: frase, uma releitura, texto mantido se a ação ainda cabe", async () => {
  vi.mocked(api.getOmbudsmanManifestation).mockResolvedValueOnce(manifestation()).mockResolvedValueOnce(manifestation({ state: "forwarded" }));
  vi.mocked(api.runOmbudsmanAction).mockRejectedValue(new ApiError(409, { error: "manifestation_changed" }, "x"));
  mount(manifestation());
  await forward("department:dep1", "Usuária relata falta de losartana na farmácia da UBS Centro.");
  expect(await screen.findByText(MANIFESTATION_CHANGED)).not.toBeNull();
  await waitFor(() => expect(api.getOmbudsmanManifestation).toHaveBeenCalledTimes(2));
  expect((screen.getByLabelText("Extrato para o destino") as HTMLTextAreaElement).value).toMatch(/losartana/); // forward cabe em forwarded
  expect(api.runOmbudsmanAction).toHaveBeenCalledTimes(1);
});
it("409 manifestation_changed: formulário fecha quando a ação não cabe mais", async () => {
  vi.mocked(api.getOmbudsmanManifestation).mockResolvedValueOnce(manifestation()).mockResolvedValueOnce(manifestation({ state: "archived", color: null }));
  vi.mocked(api.runOmbudsmanAction).mockRejectedValue(new ApiError(409, { error: "manifestation_changed" }, "x"));
  mount(manifestation());
  await sendInterim("Estamos verificando com a farmácia e retornamos.");
  expect(await screen.findByText(MANIFESTATION_CHANGED)).not.toBeNull();
  await waitFor(() => expect(screen.queryByLabelText("Texto ao cidadão")).toBeNull());
});
it("conclusiva bloqueada com encaminhamento pendente; resolutividade só onde cabe; anônima interna", async () => {
  mount(manifestation({ pending_forwardings: 1 }));
  expect((await screen.findByRole("button", { name: "Resposta conclusiva" }) as HTMLButtonElement).disabled).toBe(true);
  expect(screen.getByText(OMBUDSMAN_ERRORS.forwarding_pending)).not.toBeNull();
  cleanup();
  vi.mocked(api.runOmbudsmanAction).mockResolvedValue(manifestation({ state: "answered" }));
  mount(manifestation({ kind: "praise" }));
  fireEvent.click(await screen.findByRole("button", { name: "Resposta conclusiva" }));
  expect(screen.queryByLabelText("Resolvida?")).toBeNull();
  cleanup();
  mount(preserved({ identification_mode: "anonymous", state: "in_analysis" }));
  fireEvent.click(await screen.findByRole("button", { name: "Resposta conclusiva" }));
  expect(screen.getByText(ANONYMOUS_INTERNAL)).not.toBeNull();
  expect(screen.getByLabelText("Conclusão interna")).not.toBeNull();
});
it("cancelar encaminhamento pendente: confirmação em dois passos e uma chamada", async () => {
  vi.mocked(api.cancelOmbudsmanForwarding).mockResolvedValue(manifestation());
  mount(manifestation({ state: "forwarded", pending_forwardings: 1, forwardings: [ forwarding() ] }));
  fireEvent.click(await screen.findByRole("button", { name: "Cancelar encaminhamento" }));
  fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));
  await waitFor(() => expect(api.cancelOmbudsmanForwarding).toHaveBeenCalledWith("f1"));
});
it("pedido de complemento: uma vez por ciclo, nunca na anônima; registrar complemento recebido", async () => {
  mount(manifestation({ complement_used: true }));
  expect((await screen.findByRole("button", { name: "Pedir complemento" }) as HTMLButtonElement).disabled).toBe(true);
  cleanup();
  vi.mocked(api.runOmbudsmanAction).mockResolvedValue(manifestation());
  mount(manifestation({ state: "awaiting_complement", color: "paused", complement_used: true }));
  fireEvent.click(await screen.findByRole("button", { name: "Registrar complemento recebido" }));
  fireEvent.change(screen.getByLabelText("Complemento recebido"), { target: { value: "Por telefone: a falta foi nos dias 2, 4 e 9 de novembro." } });
  fireEvent.click(screen.getByRole("button", { name: "Registrar" }));
  await waitFor(() => expect(vi.mocked(api.runOmbudsmanAction).mock.calls[0].slice(0, 2)).toEqual([ "m1", "complement" ]));
});
```

(`forward(dest, text)` e `sendInterim(text)`: auxiliares do arquivo que abrem a ação, preenchem e enviam.)

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod33 src/modules/ombudsman/ActionsStaff.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `useManifestationAction.ts`: `run(action, fields)` → `runOmbudsmanAction(m.id, action, { expected_state: m.state, ...fields })`; sucesso → `onChanged(next)` (o detalhe vai ao cache pelo `setQueryData` da chave do detalhe e a lista é invalidada); `isChanged(err)` → `changedNotice = MANIFESTATION_CHANGED`, `onChanged(null)` (uma releitura) e a recusa não vai aos campos; resposta de `seq` velho é ignorada. O formulário aberto continua aberto se a ação ainda estiver em `availableActions(<novo>)`; senão o `ActionsPanel` o fecha e deixa a frase.
- `ClassifyForm.tsx`: assunto (primeiro nível, ativos) e subassunto (filhos ativos do escolhido); se o assunto tem filhos, o subassunto é obrigatório (`subject_not_leaf`); tipo, unidade, profissional (`listOmbudsmanProfessionals(unitId)`, D8) e urgente; o corpo leva `subject_id` sempre e só os outros campos que mudaram (o api grava o antes e o depois, §2).
- `ForwardForm.tsx`: destino num `<select>` com dois `optgroup` ("Setores", "Unidades") e valor `department:<id>`/`health_unit:<id>`; sem respondente ativo, a opção fica `disabled` com " — sem respondente"; `ExtractField`; corpo `{ department_id }` ou `{ health_unit_id }`, mais `extract`.
- `RequestComplementForm.tsx` ("Texto ao cidadão", 20–4000, com `FrozenTextNotice`), `ComplementForm.tsx` ("Complemento recebido", 20–4000, anexos pelo `AttachmentPicker` da Task 9 quando existir — até lá só texto), `ResponseForm.tsx` (intermediária: "Texto ao cidadão"; conclusiva: "Texto ao cidadão" ou, na anônima, "Conclusão interna" com `ANONYMOUS_INTERNAL`, e "Resolvida?" com `resolved`/`not_resolved` só quando `needsResolution(kind)`).
- `Forwardings.tsx`: "Cancelar encaminhamento" (`ConfirmButton`, só `pending`, só `staff`) → `cancelOmbudsmanForwarding(f.id)`; 409 `forwarding_changed` → frase e releitura.
- Toda frase da recusa por `ombudsmanRefusals`; nenhuma interpola texto digitado.

- [ ] **Step 4: Rode e veja passar; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod33 src/modules/ombudsman && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/ombudsman/ActionsPanel.tsx src/modules/ombudsman/useManifestationAction.ts src/modules/ombudsman/actions/ClassifyForm.tsx \
  src/modules/ombudsman/actions/ForwardForm.tsx src/modules/ombudsman/actions/ExtractField.tsx src/modules/ombudsman/actions/RequestComplementForm.tsx \
  src/modules/ombudsman/actions/ComplementForm.tsx src/modules/ombudsman/actions/ResponseForm.tsx src/modules/ombudsman/ManifestationView.tsx \
  src/modules/ombudsman/Forwardings.tsx src/modules/ombudsman/ActionsStaff.test.tsx
$G commit -m "feat: add ombudsman staff actions with expected state and stale handling" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** `dashboard · dashboard-mod33/feat/mod-33-ombudsman · Task 6 · ações do técnico com expected_state e 409 · corpo das ações e retorno (D9, D10)`.

---

### Task 7: Ações do ouvidor — prorrogar, arquivar, reabrir, resolutividade, pedido de reabertura, consentimento (F-33.6, F-33.7)

**Files:**
- Create: `src/modules/ombudsman/actions/ExtendForm.tsx`, `ArchiveForm.tsx`, `ReopenForm.tsx`, `ResolutionForm.tsx`, `ReopenDecisionForm.tsx`, `ConsentForm.tsx`
- Modify: `src/modules/ombudsman/ActionsPanel.tsx`, `src/modules/ombudsman/ManifestationView.tsx`
- Test: `src/modules/ombudsman/ActionsLead.test.tsx`

**Interfaces:**
- Consumes: `runOmbudsmanAction`, `decideReopenRequest`, `recordIdentityConsent`, `listOmbudsmanManifestations` (Task 1); `archiveProblems`, `archiveReasonLabel`, `useManifestationAction` (Tasks 2 e 6).
- Produces: as seis formas, todas pelo `useManifestationAction` (menos a decisão e o consentimento, que têm rota própria e tratam 409 `reopen_request_not_pending` relendo).

- [ ] **Step 1: Escreva os testes**

```tsx
it("prorrogar: justificativa 10–500 vai ao cidadão; uma vez por ciclo", async () => {
  vi.mocked(api.runOmbudsmanAction).mockResolvedValue(manifestation({ extended: true }));
  mount(manifestation(), [ "ombudsman" ]);
  fireEvent.click(await screen.findByRole("button", { name: "Prorrogar prazo" }));
  const field = screen.getByLabelText("Justificativa (vai ao cidadão)");
  expectFrozenNotice(field);
  fireEvent.change(field, { target: { value: "curta" } });
  expect((screen.getByRole("button", { name: "Prorrogar" }) as HTMLButtonElement).disabled).toBe(true);
  fireEvent.change(field, { target: { value: "Aguardamos o inventário da farmácia central." } });
  fireEvent.click(screen.getByRole("button", { name: "Prorrogar" }));
  await waitFor(() => expect(vi.mocked(api.runOmbudsmanAction).mock.calls[0])
    .toEqual([ "m1", "extend", { expected_state: "in_analysis", reason: "Aguardamos o inventário da farmácia central." } ]));
  cleanup();
  mount(manifestation({ extended: true }), [ "ombudsman" ]);
  expect((await screen.findByRole("button", { name: "Prorrogar prazo" }) as HTMLButtonElement).disabled).toBe(true);
});
it("o técnico não vê as ações do ouvidor", async () => {
  mount(manifestation({ state: "answered", color: null, concluded_at: "2026-11-10T10:00:00-03:00" }));
  await screen.findByRole("heading", { name: /2026-000123/ });
  for (const name of [ "Reabrir", "Alterar resolutividade", "Arquivar", "Prorrogar prazo" ]) expect(screen.queryByRole("button", { name })).toBeNull();
});
it("arquivar: motivo da lista; duplicidade acha o original pelo protocolo; outro órgão pede o nome", async () => {
  vi.mocked(api.listOmbudsmanManifestations).mockResolvedValue(page({ items: [ listItem({ id: "m0", protocol: "2026-000100" }) ] }));
  vi.mocked(api.runOmbudsmanAction).mockResolvedValue(manifestation({ state: "archived" }));
  mount(manifestation(), [ "ombudsman" ]);
  fireEvent.click(await screen.findByRole("button", { name: "Arquivar" }));
  fireEvent.change(screen.getByLabelText("Motivo"), { target: { value: "duplicate" } });
  fireEvent.change(screen.getByLabelText("Protocolo da manifestação original"), { target: { value: "2026-000100" } });
  fireEvent.click(screen.getByRole("button", { name: "Buscar" }));
  expect(await screen.findByText("Original: 2026-000100")).not.toBeNull();
  fireEvent.click(screen.getByRole("button", { name: "Arquivar agora" }));
  fireEvent.click(await screen.findByRole("button", { name: "Confirmar" }));
  await waitFor(() => expect(vi.mocked(api.runOmbudsmanAction).mock.calls[0][2])
    .toEqual({ expected_state: "in_analysis", reason: "duplicate", duplicate_of_id: "m0" }));
  expect(vi.mocked(api.listOmbudsmanManifestations).mock.calls[0][0]).toEqual({ q: "2026-000100" });
});
it("reabrir de ofício com justificativa; mudar resolutividade com justificativa", async () => {
  vi.mocked(api.runOmbudsmanAction).mockResolvedValue(manifestation({ state: "in_analysis", cycle: 2 }));
  mount(manifestation({ state: "answered", color: null, resolution: "resolved" }), [ "ombudsman" ]);
  fireEvent.click(await screen.findByRole("button", { name: "Alterar resolutividade" }));
  fireEvent.click(screen.getByLabelText("não resolvida"));
  fireEvent.change(screen.getByLabelText("Justificativa"), { target: { value: "A falta voltou na semana seguinte." } });
  fireEvent.click(screen.getByRole("button", { name: "Salvar resolutividade" }));
  await waitFor(() => expect(vi.mocked(api.runOmbudsmanAction).mock.calls[0][1]).toBe("change_resolution"));
  expect(vi.mocked(api.runOmbudsmanAction).mock.calls[0][2]).toEqual({ expected_state: "answered", resolution: "not_resolved", reason: "A falta voltou na semana seguinte." });
});
it("pedido de reabertura do cidadão: aceitar reabre; recusar exige texto; já decidido relê", async () => {
  const req = { id: "r1", reason: "A falta continua.", state: "pending", created_at: "2026-11-10T08:00:00-03:00", decision_text: null, decided_at: null } as const;
  vi.mocked(api.decideReopenRequest).mockRejectedValueOnce(new ApiError(409, { error: "reopen_request_not_pending" }, "x"));
  mount(manifestation({ state: "answered", color: null, reopen_request: req }), [ "ombudsman" ]);
  expect(await screen.findByText("A falta continua.")).not.toBeNull();
  fireEvent.click(screen.getByLabelText("Recusar"));
  fireEvent.change(screen.getByLabelText("Texto ao cidadão"), { target: { value: "A reposição foi confirmada pela farmácia em 10/11." } });
  fireEvent.click(screen.getByRole("button", { name: "Decidir pedido" }));
  expect(await screen.findByText(OMBUDSMAN_ERRORS.reopen_request_not_pending)).not.toBeNull();
  await waitFor(() => expect(api.getOmbudsmanManifestation).toHaveBeenCalledTimes(2));
});
it("consentimento para revelar fora da ouvidoria: registra sem mostrar nada", async () => {
  vi.mocked(api.recordIdentityConsent).mockResolvedValue(preserved());
  mount(preserved(), [ "ombudsman" ]);
  fireEvent.click(await screen.findByRole("button", { name: "Registrar consentimento do cidadão" }));
  fireEvent.click(screen.getByLabelText("O cidadão consentiu"));
  fireEvent.change(screen.getByLabelText("Como o consentimento foi dado"), { target: { value: "Por telefone, em 11/11, ao ouvidor." } });
  fireEvent.click(screen.getByRole("button", { name: "Registrar" }));
  await waitFor(() => expect(api.recordIdentityConsent).toHaveBeenCalledWith("m2", { granted: true, text: "Por telefone, em 11/11, ao ouvidor." }));
  expect(screen.queryByText(/Joana/)).toBeNull();
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod33 src/modules/ombudsman/ActionsLead.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `ExtendForm.tsx` (corpo `{ reason }`, D9), `ReopenForm.tsx` (`{ reason }`), `ResolutionForm.tsx` (`{ resolution, reason }`), todos com `ReasonHint` (mínimo 10) e `FrozenTextNotice`.
- `ArchiveForm.tsx`: motivo em `<select>` com `archiveReasonLabel` (sem `no_complement`, que é do job — D21); `duplicate` → campo "Protocolo da manifestação original" + "Buscar" (`listOmbudsmanManifestations({ q })`, aceita só um resultado diferente da própria); `forwarded_elsewhere` → "Órgão" (2–120); `ConfirmButton` "Arquivar agora" com a pergunta "Arquivar? O cidadão recebe o aviso com o motivo.".
- `ReopenDecisionForm.tsx`: no bloco do pedido pendente, `accept`/`reject` e o texto (10–2000, `FrozenTextNotice`) → `decideReopenRequest`; 409 `reopen_request_not_pending` → frase e releitura.
- `ConsentForm.tsx`: só em `preserved`, só o ouvidor; `granted` (consentiu / recusou) e o texto (10–500) → `recordIdentityConsent`; a resposta é o detalhe, que não traz identidade.
- `ActionsPanel.tsx`: a ordem dos botões segue `availableActions`.

- [ ] **Step 4: Rode e veja passar; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod33 src/modules/ombudsman && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/ombudsman/actions/ExtendForm.tsx src/modules/ombudsman/actions/ArchiveForm.tsx src/modules/ombudsman/actions/ReopenForm.tsx \
  src/modules/ombudsman/actions/ResolutionForm.tsx src/modules/ombudsman/actions/ReopenDecisionForm.tsx src/modules/ombudsman/actions/ConsentForm.tsx \
  src/modules/ombudsman/ActionsPanel.tsx src/modules/ombudsman/ManifestationView.tsx src/modules/ombudsman/ActionsLead.test.tsx
$G commit -m "feat: add ombudsman lead actions for extension, archiving, reopening and identity consent" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** `dashboard · dashboard-mod33/feat/mod-33-ombudsman · Task 7 · ações do ouvidor · nomes dos campos de texto das ações (D9) e no_complement fora da lista (D21)`.

---

### Task 8: Identidade com justificativa e anexo protegido (F-33.7)

**Files:**
- Create: `src/modules/ombudsman/IdentityDialog.tsx`
- Modify: `src/modules/ombudsman/ManifestationView.tsx`, `src/modules/ombudsman/IdentityBlock.tsx`, `src/modules/ombudsman/Attachments.tsx`
- Test: `src/modules/ombudsman/IdentityDialog.test.tsx`

**Interfaces:**
- Consumes: `revealOmbudsmanIdentity`, `getOmbudsmanAttachmentUrl` (Task 1); `ombudsmanError` (Task 2); `openFile` (Task 5); `useDialog`, `ReasonHint`, `FrozenTextNotice`.
- Produces: `IdentityDialog({ manifestationId, scope: "identity" | "attachment", attachment?: OmbudsmanAttachment, onClose(): void })` — `role="dialog" aria-modal="true" aria-label="Ver identidade"` (ou "Abrir anexo protegido"), foco preso por `useDialog`; o campo "Justificativa (fica registrada)" 10–500; "Ver identidade" → `revealOmbudsmanIdentity` uma vez; a identidade fica **só** em `useState` do diálogo e é zerada ao fechar e ao desmontar; no escopo `attachment`, "Abrir anexo" → `openFile(() => getOmbudsmanAttachmentUrl(a.id, justification), …)` e o diálogo fecha.

- [ ] **Step 1: Escreva os testes**

```tsx
it("preservada: o ouvidor vê com justificativa; fechar apaga; reabrir pede de novo; nada no cache", async () => {
  vi.mocked(api.revealOmbudsmanIdentity).mockResolvedValue(fullIdentity());
  const { client } = mountView(preserved(), [ "ombudsman" ]);
  fireEvent.click(await screen.findByRole("button", { name: "Ver identidade" }));
  const dialog = screen.getByRole("dialog", { name: "Ver identidade" });
  const field = within(dialog).getByLabelText("Justificativa (fica registrada)");
  expectFrozenNotice(field);
  const go = within(dialog).getByRole("button", { name: "Ver identidade" }) as HTMLButtonElement;
  fireEvent.change(field, { target: { value: "curta" } });
  expect(go.disabled).toBe(true);
  fireEvent.change(field, { target: { value: "conferir se é o mesmo autor da denúncia 2026-000101" } });
  fireEvent.click(go); fireEvent.click(go);
  expect(await within(dialog).findByText("Joana Prado")).not.toBeNull();
  expect(within(dialog).getByText("529.982.247-25")).not.toBeNull();
  expect(api.revealOmbudsmanIdentity).toHaveBeenCalledTimes(1);
  expect(JSON.stringify(client.getQueryCache().getAll().map((q) => q.state.data))).not.toMatch(/Joana|529\.982/);
  fireEvent.click(within(dialog).getByRole("button", { name: "Fechar" }));
  expect(screen.queryByText("Joana Prado")).toBeNull();
  fireEvent.click(screen.getByRole("button", { name: "Ver identidade" }));
  expect((within(screen.getByRole("dialog")).getByLabelText("Justificativa (fica registrada)") as HTMLTextAreaElement).value).toBe("");
});
it("registro sem cidadão: mostra os manifestant_* e os vazios como 'não informado'", async () => {
  vi.mocked(api.revealOmbudsmanIdentity).mockResolvedValue(fullIdentity({ name: null, cpf: null, phone: null, manifestant_name: "José Alves", manifestant_contact: "(41) 3333-0000" }));
  mountView(preserved({ channel: "phone" }), [ "ombudsman" ]);
  await reveal("ligou para confirmar a denúncia por telefone");
  expect(screen.getByText("José Alves")).not.toBeNull();
  expect(screen.getAllByText("não informado").length).toBeGreaterThan(0);
});
it("403 do api: a frase, sem identidade; Escape fecha e devolve o foco", async () => {
  vi.mocked(api.revealOmbudsmanIdentity).mockRejectedValue(new ApiError(403, { error: "missing_role" }, "x"));
  mountView(preserved(), [ "ombudsman" ]);
  await reveal("conferir se é o mesmo autor da denúncia");
  expect(await screen.findByText(OMBUDSMAN_NO_ROLE)).not.toBeNull();
  fireEvent.keyDown(screen.getByRole("dialog"), { key: "Escape" });
  await waitFor(() => expect(document.activeElement?.textContent).toBe("Ver identidade"));
});
it("anexo protegido: só o ouvidor, com justificativa no corpo do link", async () => {
  const win = { location: { href: "" }, close: vi.fn() };
  vi.spyOn(window, "open").mockReturnValue(win as never);
  vi.mocked(api.getOmbudsmanAttachmentUrl).mockResolvedValue("/ombudsman/files/tok9");
  vi.mocked(api.fetchOmbudsmanFile).mockResolvedValue({ blob: new Blob([ "x" ]), filename: null });
  URL.createObjectURL = vi.fn(() => "blob:9"); URL.revokeObjectURL = vi.fn();
  const att = { id: "a9", uploaded_by: "citizen", content_type: "image/png", created_at: "2026-11-03T09:12:00-03:00" } as const;
  mountView(preserved({ attachments: [ att ] }), [ "ombudsman" ]);
  fireEvent.click(await screen.findByRole("button", { name: "Abrir anexo 1 (PNG) (justificar)" }));
  const dialog = screen.getByRole("dialog", { name: "Abrir anexo protegido" });
  fireEvent.change(within(dialog).getByLabelText("Justificativa (fica registrada)"), { target: { value: "ver a foto da sala citada na denúncia" } });
  fireEvent.click(within(dialog).getByRole("button", { name: "Abrir anexo" }));
  await waitFor(() => expect(vi.mocked(api.getOmbudsmanAttachmentUrl).mock.calls[0]).toEqual([ "a9", "ver a foto da sala citada na denúncia" ]));
  cleanup();
  mountView(preserved({ attachments: [ att ] }), [ "ombudsman_staff" ]);
  expect(await screen.findByText("anexo protegido — só o ouvidor abre")).not.toBeNull();
});
```

(`mountView` devolve o `QueryClient` usado; `reveal(text)` abre o diálogo, escreve e clica.)

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod33 src/modules/ombudsman/IdentityDialog.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `IdentityDialog.tsx`: `busy` + `useRef`; a chamada direta à função do api (sem `useQuery`/`useMutation` com cache de resultado); a resposta é lida campo a campo para `KeyValue` ("Nome", "CPF", "Telefone", "Nome informado", "Contato informado", "CPF informado"; nulo → "não informado"); a nota "Este acesso foi registrado com a sua justificativa." depois de ver; "Fechar" e Escape zeram o estado e devolvem o foco ao botão que abriu; nada em `console`. Recusa por `ombudsmanError`, sem eco.
- `IdentityBlock.tsx` e `Attachments.tsx`: abrem o diálogo pelo escopo.

- [ ] **Step 4: Rode e veja passar; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod33 src/modules/ombudsman && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/ombudsman/IdentityDialog.tsx src/modules/ombudsman/IdentityDialog.test.tsx src/modules/ombudsman/ManifestationView.tsx \
  src/modules/ombudsman/IdentityBlock.tsx src/modules/ombudsman/Attachments.tsx
$G commit -m "feat: reveal preserved identity and protected attachments only with a recorded justification" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** `dashboard · dashboard-mod33/feat/mod-33-ombudsman · Task 8 · identidade com justificativa · step-up na identidade (D22) se o api pedir`.

---

### Task 9: Nova manifestação pela equipe e comprovante para impressão (F-33.4)

**Files:**
- Create: `src/modules/ombudsman/NewManifestationForm.tsx`, `src/modules/ombudsman/PersonPick.tsx`, `src/modules/ombudsman/AttachmentPicker.tsx`, `src/modules/ombudsman/Receipt.tsx`, `src/modules/ombudsman/printSheet.ts`
- Modify: `src/modules/OmbudsmanNew.tsx`, `src/modules/ombudsman/actions/ComplementForm.tsx` (anexos)
- Test: `src/modules/ombudsman/NewManifestation.test.tsx`, `src/modules/ombudsman/printSheet.test.ts`

**Interfaces:**
- Consumes: `createOmbudsmanManifestation`, `lookupOmbudsmanPerson`, `uploadOmbudsmanAttachment`, `getOmbudsmanReceipt`, `getOmbudsmanOptions` (Task 1); `newManifestationProblems`, `attachmentProblems`, rótulos, `NO_RECEIPT_ANONYMOUS` (Task 2); `maskCpf`, `isValidCpf` (`src/lib/attendance.ts`); `POPUP_BLOCKED`; `FrozenTextNotice`.
- Produces: `printSheet(title: string, rows: [ string, string ][], paragraphs?: { title: string; text: string }[]): string | null` — abre a janela no clique, monta o documento **por DOM** (`createElement` + `textContent`, nunca `innerHTML`), chama `print()`; janela bloqueada → `POPUP_BLOCKED`; `NewDraft` (Task 2); `OmbudsmanNew({ onOpenManifestation(id): void })`; `Receipt({ created, onNew(): void, onOpen(): void })`.

- [ ] **Step 1: Escreva os testes**

`src/modules/ombudsman/printSheet.test.ts`:

```ts
it("monta o documento por texto, imprime e não usa innerHTML", () => {
  const doc = document.implementation.createHTMLDocument("");
  const win = { document: doc, print: vi.fn(), focus: vi.fn(), close: vi.fn() };
  vi.spyOn(window, "open").mockReturnValue(win as never);
  expect(printSheet("Comprovante", [ [ "Protocolo", "<b>2026-000130</b>" ] ])).toBeNull();
  expect(doc.body.textContent).toContain("<b>2026-000130</b>");
  expect(doc.querySelector("b")).toBeNull();
  expect(win.print).toHaveBeenCalled();
  vi.spyOn(window, "open").mockReturnValue(null);
  expect(printSheet("x", [])).toBe(POPUP_BLOCKED);
});
```

`src/modules/ombudsman/NewManifestation.test.tsx`:

```tsx
vi.mock("../../lib/api", async (orig) => ({ ...(await orig<typeof api>()), fetchCurrentSession: vi.fn(), getOmbudsmanOptions: vi.fn(),
  createOmbudsmanManifestation: vi.fn(), lookupOmbudsmanPerson: vi.fn(), uploadOmbudsmanAttachment: vi.fn(), getOmbudsmanReceipt: vi.fn() }));
const BODY = "Fui à UBS Centro três vezes e não havia losartana.";
it("identificada por CPF: busca por POST, citizen_id no corpo, comprovante sem texto nem nome", async () => {
  vi.mocked(api.lookupOmbudsmanPerson).mockResolvedValue({ citizen_id: "c1", display_name: "Maria Lima", cpf_masked: "***.***.***-25" });
  vi.mocked(api.createOmbudsmanManifestation).mockResolvedValue({ id: "m9", protocol: "2026-000130", due_on: "2026-12-11" });
  vi.mocked(api.getOmbudsmanReceipt).mockResolvedValue(receipt());
  const doc = document.implementation.createHTMLDocument("");
  vi.spyOn(window, "open").mockReturnValue({ document: doc, print: vi.fn(), focus: vi.fn(), close: vi.fn() } as never);
  mount();
  fireEvent.change(await screen.findByLabelText("Tipo"), { target: { value: "complaint" } });
  fireEvent.click(screen.getByLabelText("Presencial"));
  fireEvent.change(screen.getByLabelText("CPF de quem manifesta"), { target: { value: "529.982.247-25" } });
  fireEvent.click(screen.getByRole("button", { name: "Buscar pelo CPF" }));
  expect(await screen.findByText("Maria Lima")).not.toBeNull();
  const body = screen.getByLabelText("Texto da manifestação");
  expectFrozenNotice(body);
  fireEvent.change(body, { target: { value: BODY } });
  fireEvent.click(screen.getByRole("button", { name: "Registrar manifestação" }));
  fireEvent.click(screen.getByRole("button", { name: "Registrar manifestação" }));
  await waitFor(() => expect(api.createOmbudsmanManifestation).toHaveBeenCalledTimes(1));
  expect(vi.mocked(api.createOmbudsmanManifestation).mock.calls[0][0])
    .toEqual({ kind: "complaint", identification_mode: "identified", channel: "in_person", citizen_id: "c1", body: BODY });
  expect(await screen.findByText("2026-000130")).not.toBeNull();
  fireEvent.click(screen.getByRole("button", { name: "Imprimir comprovante" }));
  await waitFor(() => expect(doc.body.textContent).toContain("2026-000130"));
  expect(doc.body.textContent).toMatch(/Reclamação.*Presencial/s);
  expect(doc.body.textContent).not.toMatch(/losartana|Maria|529|\*\*\*/);
  expect(screen.queryByLabelText("CPF de quem manifesta")).toBeNull(); // o formulário foi limpo
});
it("pessoa sem cadastro no canal do cidadão: nome e contato só na manifestação", async () => {
  vi.mocked(api.lookupOmbudsmanPerson).mockRejectedValue(new ApiError(404, { error: "person_not_found" }, "x"));
  vi.mocked(api.createOmbudsmanManifestation).mockResolvedValue({ id: "m9", protocol: "2026-000131", due_on: "2026-12-11" });
  mount();
  await fillBasics({ kind: "request", channel: "Telefone" });
  fireEvent.change(screen.getByLabelText("CPF de quem manifesta"), { target: { value: "529.982.247-25" } });
  fireEvent.click(screen.getByRole("button", { name: "Buscar pelo CPF" }));
  expect(await screen.findByText(OMBUDSMAN_ERRORS.person_not_found)).not.toBeNull();
  fireEvent.change(screen.getByLabelText("Nome informado"), { target: { value: "José Alves" } });
  fireEvent.change(screen.getByLabelText("Contato informado"), { target: { value: "(41) 3333-0000" } });
  fireEvent.change(screen.getByLabelText("Texto da manifestação"), { target: { value: BODY } });
  fireEvent.click(screen.getByRole("button", { name: "Registrar manifestação" }));
  await waitFor(() => expect(vi.mocked(api.createOmbudsmanManifestation).mock.calls[0][0]).toEqual({ kind: "request", identification_mode: "identified",
    channel: "phone", manifestant_name: "José Alves", manifestant_contact: "(41) 3333-0000", manifestant_cpf: "52998224725", body: BODY }));
});
it("anônima: só com denúncia, esconde os campos de pessoa e não oferece comprovante", async () => {
  vi.mocked(api.createOmbudsmanManifestation).mockResolvedValue({ id: "m10", protocol: "2026-000132", due_on: "2026-12-11" });
  mount();
  fireEvent.change(await screen.findByLabelText("Tipo"), { target: { value: "complaint" } });
  expect((screen.getByLabelText("Anônima (só denúncia)") as HTMLInputElement).disabled).toBe(true);
  fireEvent.change(screen.getByLabelText("Tipo"), { target: { value: "report" } });
  fireEvent.click(screen.getByLabelText("Anônima (só denúncia)"));
  expect(screen.queryByLabelText("CPF de quem manifesta")).toBeNull();
  fireEvent.click(screen.getByLabelText("Telefone"));
  fireEvent.change(screen.getByLabelText("Texto da manifestação"), { target: { value: "Há cobrança indevida por consulta na UBS Centro." } });
  fireEvent.click(screen.getByRole("button", { name: "Registrar manifestação" }));
  expect(await screen.findByText(NO_RECEIPT_ANONYMOUS)).not.toBeNull();
  expect(screen.queryByRole("button", { name: "Imprimir comprovante" })).toBeNull();
  expect(api.getOmbudsmanReceipt).not.toHaveBeenCalled();
});
it("OuvidorSUS pede o número; anexos: tipo, tamanho e no máximo 3, enviados antes do registro", async () => {
  vi.mocked(api.uploadOmbudsmanAttachment).mockResolvedValueOnce({ id: "a1" });
  vi.mocked(api.createOmbudsmanManifestation).mockResolvedValue({ id: "m11", protocol: "2026-000133", due_on: "2026-12-11" });
  mount();
  await fillBasics({ kind: "complaint", channel: "OuvidorSUS" });
  fireEvent.click(screen.getByRole("button", { name: "Registrar manifestação" }));
  expect(await screen.findByText("informe o número do OuvidorSUS (até 40 caracteres)")).not.toBeNull();
  fireEvent.change(screen.getByLabelText("Número do OuvidorSUS"), { target: { value: "136-2026-998877" } });
  fireEvent.change(screen.getByLabelText("Anexos"), { target: { files: [ new File([ "x" ], "a.gif", { type: "image/gif" }) ] } });
  expect(screen.getByText(OMBUDSMAN_ERRORS.attachment_type)).not.toBeNull();
  fireEvent.change(screen.getByLabelText("Anexos"), { target: { files: [ new File([ "%PDF" ], "laudo.pdf", { type: "application/pdf" }) ] } });
  await fillPerson();
  fireEvent.click(screen.getByRole("button", { name: "Registrar manifestação" }));
  await waitFor(() => expect(vi.mocked(api.createOmbudsmanManifestation).mock.calls[0][0])
    .toMatchObject({ channel: "ouvidorsus", ouvidorsus_number: "136-2026-998877", attachment_ids: [ "a1" ] }));
});
it("Abrir a manifestação leva à Caixa pelo handoff", async () => {
  const onOpen = vi.fn();
  vi.mocked(api.createOmbudsmanManifestation).mockResolvedValue({ id: "m9", protocol: "2026-000130", due_on: "2026-12-11" });
  vi.mocked(api.getOmbudsmanReceipt).mockResolvedValue(receipt());
  mount(onOpen);
  await registerSimple();
  fireEvent.click(await screen.findByRole("button", { name: "Abrir a manifestação" }));
  expect(onOpen).toHaveBeenCalledWith("m9");
});
```

(`fillBasics`, `fillPerson`, `registerSimple`: auxiliares do arquivo.)

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod33 src/modules/ombudsman/NewManifestation.test.tsx src/modules/ombudsman/printSheet.test.ts`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `NewManifestationForm.tsx`: tipo; modo (`identified`, `preserved`, `anonymous` — este desabilitado fora de `report`); canal (`in_person`, `phone`, `ouvidorsus` — o `web` não aparece, contrato §5); "Número do OuvidorSUS" só em `ouvidorsus`; pessoa (fora do anônimo) pelo `PersonPick`; unidade referida (opcional, das `options`); "Texto da manifestação" 20–4000 com `FrozenTextNotice`; anexos pelo `AttachmentPicker`. No envio: confere tudo (`newManifestationProblems`), sobe os anexos um a um (`uploadOmbudsmanAttachment`; recusa de um anexo para tudo, sem registrar), depois `createOmbudsmanManifestation` com o corpo montado só com o preenchido (CPF só dígitos); `busy` + `useRef`. Sucesso: o formulário é **zerado** (nenhum nome, CPF ou texto fica no estado) e o `Receipt` aparece.
- `PersonPick.tsx`: "CPF de quem manifesta" (`maskCpf`, `isValidCpf`) + "Buscar pelo CPF" → `lookupOmbudsmanPerson`; achado com `citizen_id` → nome e CPF mascarado, "Trocar"; achado **sem** `citizen_id` (pessoa sem uso do canal do cidadão, D6) ou 404/409 → a frase e os campos "Nome informado", "Contato informado" (e o CPF digitado vai como `manifestant_cpf`).
- `AttachmentPicker.tsx`: `<input type="file" multiple accept="application/pdf,image/jpeg,image/png" aria-label="Anexos">`, lista "Anexo N (tipo)" sem o nome do arquivo, remover, `attachmentProblems`.
- `Receipt.tsx`: na anônima, `NO_RECEIPT_ANONYMOUS` e "Registrar outra"; senão, protocolo grande, prazo, "Imprimir comprovante" → `getOmbudsmanReceipt(id)` e `printSheet("Comprovante de manifestação — Ouvidoria da Saúde", [ ["Protocolo", …], ["Tipo", kindLabel], ["Canal", channelLabel], ["Recebida em", fmtDateTime], ["Prazo de resposta", fmtDay] ], [ { title: "", text: "Guarde este número para acompanhar a sua manifestação." } ])`; "Abrir a manifestação" (`onOpen`) e "Registrar outra".
- `printSheet.ts`: `window.open("", "_blank")` antes de qualquer trabalho; `doc.title`, `h1`, uma `<table>` de linhas e os parágrafos, tudo por `textContent`; `win.focus(); win.print()`. Sem estilo com cor (o documento impresso usa as cores do navegador).
- `ComplementForm.tsx`: ganha o `AttachmentPicker` e manda `attachment_ids` quando há.

- [ ] **Step 4: Rode e veja passar; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod33 src/modules/ombudsman && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/OmbudsmanNew.tsx src/modules/ombudsman/NewManifestationForm.tsx src/modules/ombudsman/PersonPick.tsx src/modules/ombudsman/AttachmentPicker.tsx \
  src/modules/ombudsman/Receipt.tsx src/modules/ombudsman/printSheet.ts src/modules/ombudsman/actions/ComplementForm.tsx \
  src/modules/ombudsman/NewManifestation.test.tsx src/modules/ombudsman/printSheet.test.ts
$G commit -m "feat: register staff manifestations with a printable receipt" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** `dashboard · dashboard-mod33/feat/mod-33-ombudsman · Task 9 · registro pela equipe e comprovante · busca do cidadão (D6) e anexos sem nome (D17)`.

---

### Task 10: Respostas à ouvidoria — caixa do ponto de resposta (F-33.5)

**Files:**
- Create: `src/modules/ombudsman/ReplyCard.tsx`
- Modify: `src/modules/OmbudsmanReplies.tsx`
- Test: `src/modules/OmbudsmanReplies.test.tsx`

**Interfaces:**
- Consumes: `listOmbudsmanInbox`, `replyOmbudsmanInbox`, `declineOmbudsmanInbox`, `extendOmbudsmanInbox` (Task 1); `destinationLabel`, `forwardingStateLabel`, `ombudsmanRefusals`, `isChanged`, `OMB_INBOX_KEY`, `NOMINAL_QUERY` (Task 2); `DeadlineTag` (Task 4).
- Produces: `OmbudsmanReplies({ now? })` (portão `allow: (r) => r.responder`); `ReplyCard({ item, onDone(next): void, onStale(): void })` — destino, data, prazo com cor, estado, o extrato; em `pending`: "Responder à ouvidoria" (10–4000, `FrozenTextNotice`), "Não é do nosso destino" (motivo 10–500), "Prorrogar prazo" (motivo 10–500, só sem `extended`).

- [ ] **Step 1: Escreva os testes**

```tsx
vi.mock("../lib/api", async (orig) => ({ ...(await orig<typeof api>()), fetchCurrentSession: vi.fn(), listOmbudsmanInbox: vi.fn(),
  replyOmbudsmanInbox: vi.fn(), declineOmbudsmanInbox: vi.fn(), extendOmbudsmanInbox: vi.fn() }));
it("lê campo a campo: destino, extrato, prazo e estado — nunca protocolo, texto, assunto, identidade ou anexos", async () => {
  vi.mocked(api.listOmbudsmanInbox).mockResolvedValue([ { ...inboxItem(), protocol: "2026-000123", body: "Fui à UBS três vezes",
    subject: { name: "Demora" }, identity: { name: "Maria Lima" }, attachments: [ { id: "a1" } ] } as never ]);
  mount();
  expect(await screen.findByText(/falta de losartana/)).not.toBeNull();
  expect(screen.getByText("Setor: Assistência Farmacêutica")).not.toBeNull();
  expect(screen.queryByText(/2026-000123|três vezes|Demora|Maria Lima|Anexo/)).toBeNull();
});
it("responder: aviso de texto congelado, uma chamada no duplo clique, o cartão vira respondido", async () => {
  vi.mocked(api.listOmbudsmanInbox).mockResolvedValue([ inboxItem() ]);
  vi.mocked(api.replyOmbudsmanInbox).mockResolvedValue(inboxItem({ state: "replied", color: null }));
  mount();
  fireEvent.click(await screen.findByRole("button", { name: "Responder à ouvidoria" }));
  const field = screen.getByLabelText("Resposta à ouvidoria");
  expectFrozenNotice(field);
  fireEvent.change(field, { target: { value: "Reposição feita em 10/11; a UBS já tem o medicamento." } });
  fireEvent.click(screen.getByRole("button", { name: "Enviar resposta" }));
  fireEvent.click(screen.getByRole("button", { name: "Enviar resposta" }));
  await waitFor(() => expect(api.replyOmbudsmanInbox).toHaveBeenCalledTimes(1));
  expect(await screen.findByText("respondido")).not.toBeNull();
});
it("recusar com motivo; prorrogar só uma vez", async () => {
  vi.mocked(api.listOmbudsmanInbox).mockResolvedValue([ inboxItem(), inboxItem({ id: "f2", extended: true }) ]);
  vi.mocked(api.declineOmbudsmanInbox).mockResolvedValue(inboxItem({ state: "declined", color: null }));
  mount();
  const [ first, second ] = await screen.findAllByRole("article");
  expect(within(second).queryByRole("button", { name: "Prorrogar prazo" })).toBeNull();
  fireEvent.click(within(first).getByRole("button", { name: "Não é do nosso destino" }));
  fireEvent.change(within(first).getByLabelText("Motivo"), { target: { value: "Quem cuida é a farmácia central." } });
  fireEvent.click(within(first).getByRole("button", { name: "Devolver à ouvidoria" }));
  await waitFor(() => expect(api.declineOmbudsmanInbox).toHaveBeenCalledWith("f1", "Quem cuida é a farmácia central."));
});
it("encaminhamento cancelado no meio: frase, uma releitura, texto não reenviado", async () => {
  vi.mocked(api.listOmbudsmanInbox).mockResolvedValueOnce([ inboxItem() ]).mockResolvedValueOnce([]);
  vi.mocked(api.replyOmbudsmanInbox).mockRejectedValue(new ApiError(409, { error: "forwarding_changed" }, "x"));
  mount();
  await sendReply("Reposição feita em 10/11; a UBS já tem o medicamento.");
  expect(await screen.findByText(OMBUDSMAN_ERRORS.forwarding_changed)).not.toBeNull();
  await waitFor(() => expect(api.listOmbudsmanInbox).toHaveBeenCalledTimes(2));
  expect(api.replyOmbudsmanInbox).toHaveBeenCalledTimes(1);
});
it("técnico sem papel de respondente não lê a caixa", async () => {
  mount(ombudsmanUser([ "ombudsman_staff" ]));
  expect(await screen.findByText(OMBUDSMAN_NO_ROLE)).not.toBeNull();
  expect(api.listOmbudsmanInbox).not.toHaveBeenCalled();
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod33 src/modules/OmbudsmanReplies.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `OmbudsmanReplies.tsx`: `useQuery([ OMB_INBOX_KEY ], listOmbudsmanInbox, NOMINAL_QUERY)` (o extrato pode ter dado de saúde); pendentes primeiro (o api ordena; a tela não reordena), depois os outros; vazio: "Nenhum encaminhamento para o seu destino".
- `ReplyCard.tsx`: `<article aria-label="Encaminhamento de <data>">` lendo campo a campo (`destination`, `extract`, `due_on`, `color`, `state`, `extended`, `created_at`); uma ação aberta por vez; `busy` + `useRef`; sucesso → o item novo substitui o antigo no cache (`setQueryData`); `isChanged(err)` → a frase e `refetch` uma vez; o texto não é reenviado. A nota fixa "A ouvidoria responde ao cidadão. Não entre em contato com quem fez a manifestação." no topo (antiobjetivo da spec).

- [ ] **Step 4: Rode e veja passar; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod33 src/modules/OmbudsmanReplies.test.tsx src/modules/ombudsman && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/OmbudsmanReplies.tsx src/modules/ombudsman/ReplyCard.tsx src/modules/OmbudsmanReplies.test.tsx
$G commit -m "feat: add the responder inbox that shows only the extract and answers the ombudsman" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** `dashboard · dashboard-mod33/feat/mod-33-ombudsman · Task 10 · caixa do ponto de resposta · 409 forwarding_changed e forma do item (D10, D13)`.

---

### Task 11: Configuração da ouvidoria e Setores da secretaria (F-33.1)

**Files:**
- Create: `src/modules/ombudsman/RespondersPanel.tsx`, `src/modules/ombudsman/SubjectsPanel.tsx`, `src/modules/ombudsman/DeadlinesPanel.tsx`
- Modify: `src/modules/OmbudsmanSettings.tsx`, `src/modules/Departments.tsx`
- Test: `src/modules/OmbudsmanSettings.test.tsx`, `src/modules/Departments.test.tsx`

**Interfaces:**
- Consumes: as funções de configuração da Task 1; `nameProblem`, `settingsProblems`, `OMB_*_KEY`, `DEPARTMENTS_KEY`, `STATIC_QUERY` (Task 2); `DataTable`, `ConfirmButton`, `ActionMessage`.
- Produces: `OmbudsmanSettings()` (portão `lead`) com três `Panel` — "Respondentes" (lista por destino, ativar/desativar; nova ligação: pessoa entre os candidatos com o papel `ombudsman_responder` (D7) e destino setor ou unidade; 409 `already_responder`), "Assuntos" (árvore de dois níveis; novo assunto ou subassunto de um de primeiro nível; renomear; desativar; 422 `subject_depth`, `name_taken`), "Prazos" (os oito números com rótulo e unidade, um "Salvar prazos" com o corpo inteiro); `Departments()` (portão `admin`): lista, novo setor, renomear, desativar (nunca apagar — `rota_no_delete`).

- [ ] **Step 1: Escreva os testes**

`src/modules/OmbudsmanSettings.test.tsx`:

```tsx
it("prazos: valores do api, recusa fora de 1–120 e grava o corpo inteiro", async () => {
  vi.mocked(api.updateOmbudsmanSettings).mockImplementation(async (s) => s);
  mount();
  const field = await screen.findByLabelText("Resposta ao cidadão (dias)");
  expect((field as HTMLInputElement).value).toBe("30");
  fireEvent.change(field, { target: { value: "0" } });
  fireEvent.change(screen.getByLabelText("Teto diário de denúncias anônimas"), { target: { value: "1001" } });
  fireEvent.click(screen.getByRole("button", { name: "Salvar prazos" }));
  expect(screen.getAllByRole("alert").length).toBe(2);
  expect(api.updateOmbudsmanSettings).not.toHaveBeenCalled();
  fireEvent.change(field, { target: { value: "20" } });
  fireEvent.change(screen.getByLabelText("Teto diário de denúncias anônimas"), { target: { value: "80" } });
  fireEvent.click(screen.getByRole("button", { name: "Salvar prazos" }));
  await waitFor(() => expect(api.updateOmbudsmanSettings).toHaveBeenCalledWith(settings33({ response_days: 20, anonymous_daily_cap: 80 })));
  expect(await screen.findByText("Prazos salvos.")).not.toBeNull();
});
it("assuntos: subassunto só sob assunto de primeiro nível; desativado aparece marcado", async () => {
  vi.mocked(api.createOmbudsmanSubject).mockResolvedValue({ id: "s13", name: "Acolhimento", parent_id: "s1", active: true });
  mount();
  expect(await screen.findByText("Antigo")).not.toBeNull();
  expect(screen.getByText("desativado")).not.toBeNull();
  const parent = screen.getByLabelText("Dentro de");
  expect([ ...(parent as HTMLSelectElement).options ].map((o) => o.text)).toEqual([ "— assunto de primeiro nível —", "Atendimento", "Transporte" ]);
  fireEvent.change(parent, { target: { value: "s1" } });
  fireEvent.change(screen.getByLabelText("Nome do assunto"), { target: { value: "Acolhimento" } });
  fireEvent.click(screen.getByRole("button", { name: "Adicionar assunto" }));
  await waitFor(() => expect(api.createOmbudsmanSubject).toHaveBeenCalledWith({ name: "Acolhimento", parent_id: "s1" }));
});
it("respondentes: candidatos com o papel e destino; já ligado dá a frase", async () => {
  vi.mocked(api.listResponderCandidates).mockResolvedValue([ { user_id: "u7", email: "bruno@curitiba.demo" } ]);
  vi.mocked(api.createOmbudsmanResponder).mockRejectedValue(new ApiError(409, { error: "already_responder" }, "x"));
  mount();
  fireEvent.change(await screen.findByLabelText("Pessoa"), { target: { value: "u7" } });
  fireEvent.change(screen.getByLabelText("Destino do respondente"), { target: { value: "department:dep1" } });
  fireEvent.click(screen.getByRole("button", { name: "Ligar respondente" }));
  await waitFor(() => expect(api.createOmbudsmanResponder).toHaveBeenCalledWith({ user_id: "u7", department_id: "dep1" }));
  expect(await screen.findByText(OMBUDSMAN_ERRORS.already_responder)).not.toBeNull();
});
it("o técnico não abre a configuração", async () => {
  mount(ombudsmanUser([ "ombudsman_staff" ]));
  expect(await screen.findByText(OMBUDSMAN_NO_ROLE)).not.toBeNull();
  expect(api.getOmbudsmanSettings).not.toHaveBeenCalled();
});
```

`src/modules/Departments.test.tsx`:

```tsx
it("admin cria, renomeia e desativa; nome repetido dá a frase; não há apagar", async () => {
  vi.mocked(api.listDepartments).mockResolvedValue([ { id: "dep1", name: "Assistência Farmacêutica", active: true } ]);
  vi.mocked(api.createDepartment).mockRejectedValueOnce(new ApiError(422, { error: "name_taken" }, "x"));
  vi.mocked(api.updateDepartment).mockResolvedValue({ id: "dep1", name: "Assistência Farmacêutica", active: false });
  mount(ombudsmanUser([ "municipal_admin" ]));
  fireEvent.change(await screen.findByLabelText("Nome do setor"), { target: { value: "assistência farmacêutica" } });
  fireEvent.click(screen.getByRole("button", { name: "Adicionar setor" }));
  expect(await screen.findByText(OMBUDSMAN_ERRORS.name_taken)).not.toBeNull();
  fireEvent.click(screen.getByRole("button", { name: "Desativar Assistência Farmacêutica" }));
  fireEvent.click(await screen.findByRole("button", { name: "Confirmar" }));
  await waitFor(() => expect(api.updateDepartment).toHaveBeenCalledWith("dep1", { name: "Assistência Farmacêutica", active: false }));
  expect(screen.queryByRole("button", { name: /Apagar|Excluir/ })).toBeNull();
});
it("nome com 1 caractere não sai; sem o papel de admin, a frase", async () => {
  vi.mocked(api.listDepartments).mockResolvedValue([]);
  mount(ombudsmanUser([ "municipal_admin" ]));
  fireEvent.change(await screen.findByLabelText("Nome do setor"), { target: { value: "A" } });
  fireEvent.click(screen.getByRole("button", { name: "Adicionar setor" }));
  expect(api.createDepartment).not.toHaveBeenCalled();
  cleanup();
  mount(ombudsmanUser([ "ombudsman" ]));
  expect(await screen.findByText(OMBUDSMAN_NO_ROLE)).not.toBeNull();
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod33 src/modules/OmbudsmanSettings.test.tsx src/modules/Departments.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- Rótulos dos prazos: "Resposta ao cidadão (dias)", "Prorrogação (dias)", "Manifestação urgente (dias)", "Prazo do complemento (dias)", "Resposta do destino (dias)", "Prorrogação do destino (dias)", "Aviso de prazo perto do fim (dias antes)", "Teto diário de denúncias anônimas"; a nota fixa "Os prazos padrão vêm do modelo federal; confira o decreto municipal de ouvidoria (gate do go-live)."
- Respondentes em `DataTable` (pessoa, destino, situação, "Desativar"/"Reativar" com `ConfirmButton`); destino no mesmo `<select>` agrupado da Task 6, sem o filtro de respondente.
- Assuntos: lista em dois níveis com renomear inline e "Desativar"/"Reativar"; o seletor "Dentro de" só traz os de primeiro nível ativos.
- `Departments.tsx`: `DataTable` com nome, situação, "Renomear", "Desativar <nome>"/"Reativar <nome>" (`ConfirmButton`); campo "Nome do setor" (2–80) e "Adicionar setor".
- Toda recusa por `ombudsmanRefusals`; `busy` + `useRef` em cada botão.

- [ ] **Step 4: Rode e veja passar; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod33 src/modules/OmbudsmanSettings.test.tsx src/modules/Departments.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/OmbudsmanSettings.tsx src/modules/ombudsman/RespondersPanel.tsx src/modules/ombudsman/SubjectsPanel.tsx src/modules/ombudsman/DeadlinesPanel.tsx \
  src/modules/Departments.tsx src/modules/OmbudsmanSettings.test.tsx src/modules/Departments.test.tsx
$G commit -m "feat: configure ombudsman responders, subjects and deadlines, and city departments" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** `dashboard · dashboard-mod33/feat/mod-33-ombudsman · Task 11 · configuração e setores · candidatos a respondente (D7) e gate de /departments (D23)`.

---

### Task 12: Painel da ouvidoria e Relatório anual com publicação e impressão (F-33.9)

**Files:**
- Modify: `src/modules/OmbudsmanPanel.tsx`, `src/modules/OmbudsmanAnnualReport.tsx`
- Test: `src/modules/OmbudsmanPanel.test.tsx`, `src/modules/OmbudsmanAnnualReport.test.tsx`

**Interfaces:**
- Consumes: `getOmbudsmanDashboard`, `getOmbudsmanProfessionalsPanel`, `getAnnualReport`, `updateAnnualReport`, `publishAnnualReport` (Task 1); rótulos, `reportTextProblems`, `PUBLIC_REPORT_NOTICE`, `OMB_DASHBOARD_KEY`, `OMB_REPORT_KEY`, `STATIC_QUERY` (Task 2); `printSheet` (Task 9); `fmtCount`, `isSuppressed`, `SUPPRESSED_LABEL` (`src/lib/smallCount.ts`); `StatTile`, `DataTable`, `ConfirmButton`.
- Produces: `OmbudsmanPanel({ now? })` (portão `staff`; período "De"/"Até", padrão do dia 1 do mês até hoje; totais por tipo, canal, estado, assunto e unidade; no prazo, tempo médio, resolutividade; satisfação; aba "Por profissional" só para o `lead`); `OmbudsmanAnnualReport({ now? })` (portão `lead`; ano; números do ano; três textos; salvar rascunho; publicar; imprimir); `numbersRows(d: OmbudsmanDashboard): [ string, string ][]` (exportada, usada pelo painel e pela impressão).

- [ ] **Step 1: Escreva os testes**

`src/modules/OmbudsmanPanel.test.tsx`:

```tsx
it("totais com '< 5' e 'oculto', nunca zero no lugar do suprimido", async () => {
  vi.mocked(api.getOmbudsmanDashboard).mockResolvedValue(dashboard33());
  mount();
  expect(await screen.findByText("42")).not.toBeNull();
  const praise = screen.getByText("Elogio").closest("[role='row']") as HTMLElement;
  expect(within(praise).getByText(SUPPRESSED_LABEL)).not.toBeNull();
  expect(screen.getByText("Resolutividade").parentElement!.textContent).toMatch(/oculto/);
  expect(screen.getByText(/85,0\s?%/)).not.toBeNull();
  expect(screen.getByText(/17,4 dias/)).not.toBeNull();
  expect(screen.getByText(/4,1 de 5/)).not.toBeNull();
});
it("período vai à consulta; o fim antes do início não sai", async () => {
  vi.mocked(api.getOmbudsmanDashboard).mockResolvedValue(dashboard33());
  mount();
  await screen.findByText("42");
  expect(vi.mocked(api.getOmbudsmanDashboard).mock.calls[0][0]).toEqual({ from: "2026-11-01", to: "2026-11-11" });
  fireEvent.change(screen.getByLabelText("De"), { target: { value: "2026-11-12" } });
  fireEvent.click(screen.getByRole("button", { name: "Atualizar" }));
  expect(screen.getByText("o início vem antes do fim")).not.toBeNull();
  expect(api.getOmbudsmanDashboard).toHaveBeenCalledTimes(1);
});
it("por profissional só para o ouvidor", async () => {
  vi.mocked(api.getOmbudsmanDashboard).mockResolvedValue(dashboard33());
  vi.mocked(api.getOmbudsmanProfessionalsPanel).mockResolvedValue([ { professional: { id: "p1", name: "Dr. Caio" }, kind: "complaint", count: 3 } ]);
  mount(ombudsmanUser([ "ombudsman_staff" ]));
  await screen.findByText("42");
  expect(screen.queryByRole("tab", { name: "Por profissional" })).toBeNull();
  expect(api.getOmbudsmanProfessionalsPanel).not.toHaveBeenCalled();
  cleanup();
  mount(ombudsmanUser([ "ombudsman" ]));
  fireEvent.click(await screen.findByRole("tab", { name: "Por profissional" }));
  expect(await screen.findByText("Dr. Caio")).not.toBeNull();
});
```

`src/modules/OmbudsmanAnnualReport.test.tsx`:

```tsx
it("rascunho: textos com o aviso público, até 8000, salvar manda os três", async () => {
  vi.mocked(api.getAnnualReport).mockResolvedValue(annualReport());
  vi.mocked(api.updateAnnualReport).mockImplementation(async (_y, i) => annualReport(i));
  mount();
  const field = await screen.findByLabelText("Problemas recorrentes");
  expect(field.getAttribute("aria-describedby")!.split(/\s+/).map((i) => document.getElementById(i)?.textContent)).toContain(PUBLIC_REPORT_NOTICE);
  fireEvent.change(field, { target: { value: "x".repeat(8001) } });
  fireEvent.click(screen.getByRole("button", { name: "Salvar rascunho" }));
  expect(api.updateAnnualReport).not.toHaveBeenCalled();
  fireEvent.change(field, { target: { value: "Falta de medicamentos na UBS Centro." } });
  fireEvent.change(screen.getByLabelText("Providências adotadas"), { target: { value: "Nova rotina de reposição semanal." } });
  fireEvent.click(screen.getByRole("button", { name: "Salvar rascunho" }));
  await waitFor(() => expect(api.updateAnnualReport).toHaveBeenCalledWith(2026,
    { recurring_problems: "Falta de medicamentos na UBS Centro.", actions_taken: "Nova rotina de reposição semanal.", notes: "" }));
});
it("publicar: dois passos; depois só leitura e o 'publicado em'", async () => {
  vi.mocked(api.getAnnualReport).mockResolvedValue(annualReport({ recurring_problems: "Falta de medicamentos." }));
  vi.mocked(api.publishAnnualReport).mockResolvedValue(annualReport({ state: "published", recurring_problems: "Falta de medicamentos.",
    published_at: "2026-11-11T10:00:00-03:00", published_by: { name: "Rita Prado" } }));
  mount();
  fireEvent.click(await screen.findByRole("button", { name: "Publicar relatório de 2026" }));
  fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));
  expect(await screen.findByText(/Publicado em 11\/11\/2026 por Rita Prado/)).not.toBeNull();
  expect(screen.queryByRole("button", { name: "Salvar rascunho" })).toBeNull();
  expect((screen.getByLabelText("Problemas recorrentes") as HTMLTextAreaElement).readOnly).toBe(true);
});
it("report_published do api relê e passa a só leitura", async () => {
  vi.mocked(api.getAnnualReport).mockResolvedValueOnce(annualReport()).mockResolvedValueOnce(annualReport({ state: "published" }));
  vi.mocked(api.updateAnnualReport).mockRejectedValue(new ApiError(409, { error: "report_published" }, "x"));
  mount();
  fireEvent.click(await screen.findByRole("button", { name: "Salvar rascunho" }));
  expect(await screen.findByText(OMBUDSMAN_ERRORS.report_published)).not.toBeNull();
  await waitFor(() => expect(screen.queryByRole("button", { name: "Salvar rascunho" })).toBeNull());
});
it("imprimir: números com '< 5' e os três textos; nada pessoal", async () => {
  vi.mocked(api.getAnnualReport).mockResolvedValue(annualReport({ recurring_problems: "Falta de medicamentos." }));
  const doc = document.implementation.createHTMLDocument("");
  vi.spyOn(window, "open").mockReturnValue({ document: doc, print: vi.fn(), focus: vi.fn(), close: vi.fn() } as never);
  mount();
  fireEvent.click(await screen.findByRole("button", { name: "Imprimir" }));
  expect(doc.body.textContent).toContain("Relatório de gestão da Ouvidoria da Saúde — 2026");
  expect(doc.body.textContent).toContain("Falta de medicamentos.");
  expect(doc.body.textContent).toContain(SUPPRESSED_LABEL);
});
```

- [ ] **Step 2: Rode e veja falhar**

Run: `$R/bin/test dashboard -w dashboard-mod33 src/modules/OmbudsmanPanel.test.tsx src/modules/OmbudsmanAnnualReport.test.tsx`
Expected: FAIL.

- [ ] **Step 3: Implemente**

- `OmbudsmanPanel.tsx`: `useQuery([ OMB_DASHBOARD_KEY, from, to ], …, STATIC_QUERY)`; período só muda no "Atualizar"; `StatTile` para total, no prazo (`fmtCount(v, "%")`), tempo médio ("N dias"), resolutividade; tabelas para os cinco recortes (rótulo do código, contagem por `fmtCount`) e a satisfação (`met` sim/em parte/não, `clear` sim/não, média "N de 5", respostas); a nota `SUPPRESSED_HINT` adaptada: "Contagens de 1 a 4 aparecem como “< 5”; taxas e médias calculadas sobre elas aparecem como “oculto”."; abas `role="tablist"` "Resumo" e, só para `lead`, "Por profissional" (lida só ao abrir a aba).
- `OmbudsmanAnnualReport.tsx`: ano (atual e anterior); `getAnnualReport(year)`; os números do ano pelo `numbersRows`; três `<textarea>` ("Problemas recorrentes", "Providências adotadas", "Observações", até 8000, com `PUBLIC_REPORT_NOTICE` por `aria-describedby`; `readOnly` publicado); "Salvar rascunho"; `ConfirmButton` "Publicar relatório de <ano>" com a pergunta "Publicar? Depois de publicado, o relatório não muda."; "Imprimir" → `printSheet("Relatório de gestão da Ouvidoria da Saúde — <ano>", numbersRows(r.numbers), [ três parágrafos com título ])`. 409 `report_published` → frase e releitura.

- [ ] **Step 4: Rode e veja passar; tipos**

Run: `$R/bin/test dashboard -w dashboard-mod33 src/modules/OmbudsmanPanel.test.tsx src/modules/OmbudsmanAnnualReport.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/OmbudsmanPanel.tsx src/modules/OmbudsmanAnnualReport.tsx src/modules/OmbudsmanPanel.test.tsx src/modules/OmbudsmanAnnualReport.test.tsx
$G commit -m "feat: add the ombudsman panel and the annual management report with publishing and print" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** `dashboard · dashboard-mod33/feat/mod-33-ombudsman · Task 12 · painel e relatório anual · forma do painel e do relatório (D19, D20)`.

---

### Task 13: Suíte, build, varredura, revisão e prova no navegador (api real na 3056)

**Files:** nenhum novo (correções que a prova pedir entram nas tasks donas, com teste).

- [ ] **Step 1: Suíte inteira, tipos e build**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; W=$R/.worktrees/dashboard-mod33
$R/bin/test dashboard -w dashboard-mod33
cd $W && npx tsc --noEmit && npx vite build >/dev/null && echo build-ok
```

Expected: verde, `build-ok`, e a contagem = base da Task 0 + **16 arquivos de teste novos**: `src/lib/api.ombudsman.test.ts`, `src/lib/ombudsman.test.ts`, `src/shell/modules.ombudsman.test.ts`, `src/modules/ombudsman/OmbudsmanGate.test.tsx`, `src/modules/ombudsman/ManifestationList.test.tsx`, `src/modules/ombudsman/ManifestationView.test.tsx`, `src/modules/ombudsman/ActionsStaff.test.tsx`, `src/modules/ombudsman/ActionsLead.test.tsx`, `src/modules/ombudsman/IdentityDialog.test.tsx`, `src/modules/ombudsman/NewManifestation.test.tsx`, `src/modules/ombudsman/printSheet.test.ts`, `src/modules/OmbudsmanReplies.test.tsx`, `src/modules/OmbudsmanSettings.test.tsx`, `src/modules/Departments.test.tsx`, `src/modules/OmbudsmanPanel.test.tsx`, `src/modules/OmbudsmanAnnualReport.test.tsx` (o `useFormProblems.test.tsx` ganha casos, não arquivo). Contagem que dobra = artefato de build descoberto pelo vitest: confira antes de seguir. **Erro não tratado** no resumo com tudo verde = mock sem `fetchCurrentSession` (Step 2).

- [ ] **Step 2: Varredura de dado sensível, storage, polling e mocks**

```bash
cd $W
O="src/modules/ombudsman src/modules/Ombudsman*.tsx src/modules/Departments.tsx src/lib/ombudsman.ts"
grep -rnE "refetchInterval|setInterval" $O && echo "ERRO: polling" || echo ok-polling
grep -rnE "localStorage|sessionStorage|indexedDB|console\.(log|info|warn|error)" $O && echo "ERRO: storage/log" || echo ok-storage
grep -rnE "innerHTML|dangerouslySetInnerHTML" $O && echo "ERRO: html cru" || echo ok-html
sed -n '/Ouvidoria (módulo 33/,$p' src/lib/api.ts | grep -nE "withQuery\(|\?\$\{" | grep -vE "ombListQuery|from|year" && echo "revisar: URL" || echo ok-url
grep -rnE "\.\.\.(item|row|m|f|identity|r)\b" src/modules/ombudsman/ManifestationList.tsx src/modules/ombudsman/ReplyCard.tsx src/modules/ombudsman/IdentityBlock.tsx && echo "revisar: resposta espalhada" || echo ok-spread
grep -rnE "useQuery|useMutation|setQueryData" src/modules/ombudsman/IdentityDialog.tsx && echo "ERRO: identidade no cache" || echo ok-identity
for f in $(grep -rlE 'vi\.mock\("\.\./(\.\./)?lib/api", async' src/modules/ombudsman src/modules/Ombudsman*.test.tsx src/modules/Departments.test.tsx src/shell/modules.ombudsman.test.ts 2>/dev/null); do
  grep -q "fetchCurrentSession" "$f" || echo "ERRO: $f sem fetchCurrentSession"; done; echo ok-mocks
```

Expected: `ok-polling`, `ok-storage`, `ok-html`, `ok-url`, `ok-spread`, `ok-identity`, `ok-mocks` sem nenhuma linha `ERRO`.

- [ ] **Step 3: Revisão do ramo inteiro** (superpowers:requesting-code-review), com foco nas cinco linhas do Review Focus, na identidade (campo a campo, diálogo sem cache), no `expected_state` de toda ação, no extrato (dois avisos), no ponto de resposta (só o extrato), na impressão (por `textContent`, sem dado pessoal) e no `useFormProblems` (as telas do CAPS seguem iguais).

- [ ] **Step 4: Prova no navegador** (só com o api do 33 mergeado ou no worktree dele, na 3056, e o banco de dev migrado e semeado **com autorização do usuário**)

Suba o Vite do worktree na rede do compose, porta **5216**, com proxy para `http://api:3056` (mesmo comando da prova do 32a, trocando porta e worktree), abra `curitiba.localhost:5216/dashboard/` e percorra, com `ombudsman` ligado, com quatro contas da semente (admin, ouvidor, técnico, respondente; senhas de dev podem ser mostradas, nunca as de staging):
1. Admin: Setores da secretaria — criar "Assistência Farmacêutica", renomear, desativar e reativar; Equipe — convidar com os três papéis (o `ombudsman` pede step-up).
2. Ouvidor: Configuração — respondente do setor e de uma UBS; um subassunto novo; mudar o aviso para 3 dias.
3. Técnico: Nova manifestação identificada por CPF (presencial), por telefone sem cadastro (nome e contato), preservada e anônima (denúncia); imprimir o comprovante (conferir a janela: sem texto nem nome); OuvidorSUS com número; anexo PDF.
4. Caixa: filtros por estado, cor e protocolo; "Carregar mais"; abrir a identificada e a preservada (o técnico vê "Identidade preservada…").
5. Classificar, encaminhar ao setor (os dois avisos no extrato; o destino sem respondente indisponível); numa segunda aba, o mesmo técnico tenta outra ação com a tela velha → `manifestation_changed`.
6. Respondente: Respostas à ouvidoria — só o extrato; prorrogar; responder. Técnico: a resposta aparece; resposta conclusiva com resolutividade.
7. Pedir complemento; responder pelo wpda (prova do wpda) ou registrar o recebido; resposta intermediária; prorrogação pelo ouvidor.
8. Ouvidor: ver a identidade da preservada com justificativa; fechar e reabrir (pede de novo); abrir o anexo protegido; registrar consentimento.
9. Arquivar por duplicidade (achar o original pelo protocolo); reabrir de ofício; decidir um pedido de reabertura vindo do wpda; mudar a resolutividade.
10. Painel com "< 5" e o recorte por profissional (só o ouvidor); Relatório anual — salvar, imprimir, publicar (dois passos) e conferir a só leitura.
11. Telefone (375×812) na Caixa e nas Respostas (cartões do `DataTable`), e só teclado nos passos 5, 6 e 8 (o diálogo prende o foco e o Escape devolve); ao terminar, `resize_window` preset desktop.

Anote cada passo com o que viu. Achado que muda comportamento vira teste na task dona antes da correção; achado só visual vai para o usuário (não vira tarefa).

- [ ] **Step 5: Board e encerramento**

Cards do dashboard prontos: F-33.4, F-33.5, F-33.6, F-33.7 e F-33.9 → **Done** se o api já terminou o deles; F-33.1 espera o maintenance (o último app move). **Verified** só depois da prova do Step 4, com o usuário. Merge, push e tag só com autorização (antes do merge na main, a suíte do dashboard inteira neste worktree já basta; a suíte completa do api é da sessão "Testes" e do plano do api). Worktree: `$G -C $R/apps/dashboard worktree remove $W` depois do merge.

**Relato ao coordenador:** `dashboard · dashboard-mod33/feat/mod-33-ombudsman · Task 13 · suíte <n>/<n>, build ok, varreduras ok, prova <passos ok/achados> · cards movidos`.

---

## Divergências propostas ao contrato

Cada uma diz o que a tela precisa, a evidência no código real e a proposta. Se o plano do api adotar outra forma, vale a do api (Task 0 Step 4).

- **D1 — Status do `feature_disabled`.** O §1 diz "404 `feature_disabled`, como os outros gates", mas os outros gates do api respondem **403** (`app/controllers/campaigns_controller.rb:17`, `attendances_controller.rb:14`, `consultations_controller.rb:214`, `tfd/base_controller.rb:2`, `dental_care/base_controller.rb:2`), e o dashboard só reconhece 403 (`src/lib/features.ts:77`, `featureDisabledKey`). Proposta: as rotas da equipe (§5–§7, §11) respondem **403** `{ error: "feature_disabled", feature }`; o 404 fica só para as rotas públicas do §4, se o api quiser esconder a existência. O plano aceita os dois (Task 2), mas o contrato deve dizer um.
- **D2 — O aviso do extrato e o `FrozenTextNotice`.** O `FrozenTextNotice` tem texto fixo ("Depois de enviado, este texto não pode ser alterado. Não escreva nome, telefone, CPF ou outros dados pessoais.", `src/components/FrozenTextNotice.tsx:4`), sem prop de texto, e a frase do §14 é outra. Proposta: o campo do extrato mostra **os dois** (o extrato também não muda, §13), por `aria-describedby`, sem mexer em `src/components`; a frase do contrato vira `EXTRACT_NOTICE` em `src/lib/ombudsman.ts`.
- **D3 — `ombudsman_staff` não privilegiado.** O §1 põe só o `ombudsman` em `PRIVILEGED_ROLES`, mas o técnico lê o texto da manifestação (dado de saúde, decisão 12) e a identidade no `identified`; todo papel que lê dado nominal é privilegiado no api (`app/models/membership.rb:43–46`: `pharmacy_assistant`, `transport_dispatcher`, `psychosocial_staff`) e no dashboard (`src/lib/team.ts:112`). Proposta: `ombudsman_staff` também privilegiado (step-up para conceder); `ombudsman_responder` não (só o extrato). O plano segue o contrato até a decisão.
- **D4 — `ROLE_OPTIONS` não existe.** O §1 diz "no dashboard, `ROLE_OPTIONS`"; a lista real é `INVITE_ROLES` em `src/lib/team.ts:79`, com `PRIVILEGED_ROLES` na linha 112. Proposta: corrigir o nome no contrato.
- **D5 — A equipe da ouvidoria não lê unidades nem assuntos ativos para escolher.** `GET /attendance/units` exige verificador, profissional ou admin (`app/controllers/health_units_controller.rb:140–143`) e `units/all` exige admin (linha 14); o ouvidor e o técnico não têm nenhum. O encaminhamento precisa saber se o destino tem respondente ativo (senão 422). Proposta: `GET /ombudsman/options` (staff, ombudsman) → `{ units: [{id, name, has_active_responder}], departments: [{id, name, has_active_responder}] (ativos), subjects: [{id, name, parent_id, active}] }`.
- **D6 — Busca do cidadão pelo CPF.** O §5 diz que o plano do api indica a busca; cada módulo tem a própria `POST /<módulo>/person_lookup` (`config/routes.rb:475, 588, 631, 687, 751`), que devolve `patient_id` (`app/controllers/psychosocial/person_lookups_controller.rb:7`), mas a manifestação guarda `citizen_id` (§1), e um paciente pode não ter cidadão (`Patient has_many :citizens`, `app/models/patient.rb:31`). Proposta: `POST /ombudsman/person_lookup { cpf }` → `{ citizen_id | null, display_name, cpf_masked }`; sem `citizen_id` (ou 404 `person_not_found`, 409 `citizen_not_verified`), a tela registra pelos `manifestant_*`. CPF só no corpo.
- **D7 — Candidatos a respondente.** O ouvidor liga pessoas a destinos, mas a lista de membros (`GET /setup/memberships`) é só do `municipal_admin` (`app/controllers/setup_controller.rb:126–145`). Proposta: `GET /ombudsman/responder_candidates` (ombudsman) → `{ items: [{ user_id, email }] }` com quem tem `ombudsman_responder` ativo.
- **D8 — Profissional referido.** O §1 tem `referred_professional_id` "do módulo 10", sem rota de escolha para a ouvidoria. Proposta: `GET /ombudsman/professionals?health_unit_id=` → `{ items: [{ id, name }] }` (vínculos ativos na unidade); só id na URL.
- **D9 — Corpo e retorno das ações.** O §5 lista as ações mas não os campos. Proposta (todas com `expected_state`, todas devolvendo o detalhe do §5): `classify { subject_id, kind?, health_unit_id?, referred_professional_id?, urgent? }`; `forward { department_id | health_unit_id, extract }`; `request_complement { text }`; `complement { text, attachment_ids? }`; `extend { reason }`; `interim_response { text }`; `final_response { text, resolution? }`; `archive { reason, duplicate_of_id?, external_body_name? }`; `reopen { reason }`; `change_resolution { resolution, reason }`. Os textos entram em `filter_parameters` (§13).
- **D10 — Concorrência no encaminhamento e no ponto de resposta.** `POST /ombudsman/forwardings/:id/cancel` e `/ombudsman/inbox/:id/(reply|decline|extend)` não têm `expected_state`; o respondente pode responder um encaminhamento já cancelado. Proposta: o api responde 409 `forwarding_changed` quando o encaminhamento não está `pending`; nenhum corpo novo.
- **D11 — Códigos e limites sem nome.** Códigos que o contrato implica: `forwarding_changed`, `resolution_required`, `resolution_not_allowed`, `subject_required`, `subject_not_leaf` (assunto com filhos), `subject_depth` (subassunto de subassunto), `duplicate_required`, `external_body_required`, `justification_required`, `interim_not_allowed` (anônima), `receipt_not_allowed` (anônima), `reopen_request_not_pending`, `name_taken`, `already_responder`, `report_published`, `link_expired`, `person_not_found`, `citizen_not_verified`, `attachment_limit`, `attachment_type`, `attachment_too_large`, `destination_inactive`, `not_responder`; `problems[]` como `{ error, path }` (o dashboard aceita `code`). Limites que o contrato não dá: resposta do ponto 10–4000; motivo de recusa e de prorrogação do ponto 10–500; texto ao cidadão 20–4000; justificativas (prorrogação, resolutividade, reabertura de ofício) 10–500; decisão do pedido 10–2000; consentimento 10–500; `external_body_name` 2–120.
- **D12 — Forma do assunto na lista.** O §5 diz `subject` sem forma; o painel agrupa pelo primeiro nível. Proposta: `subject: { id, name, parent: { id, name } | null } | null` e `health_unit: { id, name } | null`.
- **D13 — Forma do destino.** O §6 diz `destination` sem forma. Proposta: `{ kind: "department" | "health_unit", id, name }`, igual no detalhe, na caixa do respondente e na configuração.
- **D14 — Cursor da Caixa.** O §5 diz "paginação por cursor como nas listas do 31": proposta explícita `?before=<cursor>&limit=` → `{ items, next_before }` (`src/lib/api.ts:6309`, `psyPage`), e os filtros de lista como `state[]=…` repetidos.
- **D15 — Identidade no `identified` sem cidadão.** O §5 dá `{ name, cpf_masked, phone_masked }`, mas o registro pela equipe sem `citizen_id` tem `manifestant_name` e `manifestant_contact` (não telefone). Proposta: `identity: { name, cpf_masked, phone_masked, contact } | "preserved"` (no `identified`, os `manifestant_*` preenchem `name` e `contact`), e `identity_available: boolean` para a anônima sem nenhum dado não oferecer "Ver identidade" à toa.
- **D16 — Eventos.** O §1 diz `kind (§2)`, mas o §2 não enumera os tipos de evento. Proposta: `received`, `classified`, `forwarded`, `forwarding_cancelled`, `forwarding_replied`, `forwarding_declined`, `forwarding_extended`, `complement_requested`, `complement_received`, `extended`, `interim_response`, `final_response`, `archived`, `reopened`, `reopen_requested`, `reopen_decided`, `resolution_changed`, `identity_consent_recorded`; no detalhe da equipe, `{ id, kind, actor, actor_name, at, text, data, visible_to_citizen }` (o nome de quem agiu é para a equipe; o cidadão nunca o vê, §3).
- **D17 — Nome do arquivo do anexo.** O §3/§5 manda `{ filename, content_type, data }`, mas o nome do arquivo pode trazer o nome da pessoa ("RG Maria.pdf") e o técnico veria isso numa manifestação `preserved`; o TFD já tirou o nome pelo mesmo motivo (`src/lib/api.ts:6045`, "D11: só o tipo e o conteúdo em base64"). Proposta: o upload da equipe não manda `filename` e o detalhe da equipe não devolve `filename` (o api gera um nome por id, como `Files.generated_name`).
- **D18 — Rota do arquivo do anexo.** O §5 diz "URL curta assinada (padrão de `tfd/files_controller.rb`)" sem a rota do download. Proposta: `POST /ombudsman/attachments/:id/link` → `{ url: "/ombudsman/files/<token>" }` e `GET /ombudsman/files/:token` (5 min, presa ao usuário; 410 `link_expired`).
- **D19 — Forma do painel.** Proposta: `GET /ombudsman/dashboard?from&to` → `{ from, to, total, by_kind: [{key, count}], by_channel, by_state, by_subject: [{id, name, count}], by_unit: [{id, name, count}], on_time_rate, avg_days_to_conclude, resolution_rate, satisfaction: { responses, met: {yes, partly, no}, clear: {yes, no}, avg_score } }`, com a supressão do `Analytics::Suppression`, que hoje devolve `{ suppressed: true }` (`app/services/analytics/suppression.rb:7`, `Admin::SmallCount::SUPPRESSED`), não o `"lt5"` do 31/32a; o dashboard lê pelo `fmtCount` de `src/lib/smallCount.ts`. `GET /ombudsman/dashboard/professionals` → `{ items: [{ professional: {id, name}, kind, count }] }`.
- **D20 — Relatório de um ano sem rascunho.** O §11 não diz o que `GET /ombudsman/annual_reports/:year` devolve quando nada foi salvo, nem de onde vêm os números antes da publicação. Proposta: sempre 200 `{ year, state: "draft" | "published", recurring_problems, actions_taken, notes, numbers, published_at, published_by: { name } | null }` (rascunho vazio sem gravar nada; `numbers` ao vivo no rascunho e o `snapshot` depois de publicado); PATCH num publicado → 409 `report_published`.
- **D21 — `no_complement` pelo ouvidor.** O §2 lista `no_complement` entre os motivos e diz "ombudsman (e o job em `no_complement`)". Proposta: o motivo `no_complement` é só do job (422 se o ouvidor o mandar); a tela não o oferece.
- **D22 — Step-up na identidade.** O §5 exige justificativa em `POST …/identity`, sem dizer se pede o segundo fator (a decisão 13 diz que o ouvidor é "privilegiado, com segundo fator"). Proposta: sem step-up por leitura (a trilha com justificativa já registra); se o api exigir, a tela trata o `mfa_required` pelo mesmo caminho do `SensitiveAction`.
- **D23 — `/departments` e o interruptor.** O §7 põe `departments` fora de `/ombudsman` ("genérico, para outros módulos", spec §3) e o §1 diz que "toda rota do módulo" responde `feature_disabled` sem `ombudsman`. Proposta: `/departments` sem gate de interruptor (é cadastro da cidade); o dashboard mostra "Setores da secretaria" só com `ombudsman`, por ser o único uso hoje.
- **D24 — Filtro por "responsável".** A spec §4 pede filtro "por responsável" na Caixa, mas o modelo do §1 não tem responsável pela manifestação. Proposta: sem esse filtro nesta entrega (a spec ajusta), ou um `assigned_user_id` no contrato se o usuário quiser.

## Self-review

1. **Cobertura da spec (dashboard):** decisão 2 → Tasks 2 e 9; 3 e 12 → Tasks 5 e 8; 4 e 7 → Task 9; 9 e 10 → Tasks 2, 6 e 7; 11 → Tasks 6, 10 e 11; 13 → Tasks 2 e 3; 14 → Tasks 6 e 11; 15 → Tasks 4, 10 e 11; 18 → Task 12; 19 → Tasks 1 e 3. Decisões 5, 6, 8, 16 e 17 são do api e do wpda. §4 telas → Tasks 4–12; §5 → Global Constraints e Task 13 Step 2; §6 dashboard ("testes de componente; fluxo da caixa até a resposta") → Tasks 4–12 e a prova da Task 13.
2. **Contrato §5, §6, §7, §11 e §14:** cada rota tem função na Task 1 e tela nas Tasks 4–12; o aviso fixo do extrato está nas Tasks 2 e 6; o comprovante na Task 9; a impressão do relatório na Task 12.
3. **Tipos:** `OmbudsmanManifestation`, `OmbudsmanListItem`, `OmbudsmanInboxItem`, `OmbudsmanOptions` e as funções da Task 1 são os mesmos nas Tasks 2–12; `availableActions` (Task 2) governa o `ActionsPanel` (Tasks 6–7); `useManifestationAction` (Task 6) serve às Tasks 6 e 7; `printSheet` (Task 9) serve às Tasks 9 e 12; `openFile` (Task 5) às Tasks 5 e 8.
4. **Review Focus:** as cinco linhas têm teste na task dona (8; 6; 6; 10; 9 e 12).
5. **CI:** todo `vi.mock` do api com `importOriginal` inclui `fetchCurrentSession` (exemplos das Tasks 4, 6, 9, 10) e a Task 13 varre.
6. **Proporção:** o código do plano é teste, fixture e frase fixa; implementação por assinatura e valores.
