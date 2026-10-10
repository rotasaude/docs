# Módulo 19d — Receita de controlado e de antimicrobiano com SNCR (dashboard) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito: 19c entregue em `origin/main`** (o dashboard do 19c — aba Documentos, `PrescriptionForm`, `MedicationSearch`, `ConsultationDocuments`, `CancelDocument`, `src/lib/clinicalDocuments.ts`, `src/test/documentFixtures.ts` — mergeado depois do api do 19c). Hoje o `origin/main` do dashboard está em `8a0c0de` (19b) e a branch `feat/mod-19c-documents` do dashboard ainda não existe: este plano é escrito contra o **plano** do dashboard do 19c (`docs/superpowers/plans/2026-10-09-module-19c-documents-dashboard.md`, blocos Produces e o código das Tasks 1–6) e o contrato do 19c (§12 prevalece). A **Task 0** confere que o 19c chegou e **para** se não chegou. O **api do 19d** precisa estar rodando na branch dele (porta **3039**, com o `fake-sncr` do compose na **8092**) para a prova (Task 10) e é **mergeado antes** deste (contrato §11: contracts → api → dashboard → maintenance).

**Goal:** Telas do 19d no painel da cidade (F-19.25 a F-19.29, lado dashboard): **Conta → SNCR** (saldo por tipo, pedidos do mês, "Obter números do SNCR", aviso de saldo baixo e de numeração simulada) com a rota de retorno `/dashboard/sncr/callback`; **endereço e telefone** do profissional em Meu perfil; na receita do 19c, a **categoria calculada**, a **identificação e o endereço do paciente** na receita de controle especial, os **limites** (3 substâncias C1, 60/180 dias), o aviso **"vai sair em papel"** antes de emitir e com o `paper_reason` depois, e a **faixa de numeração simulada**; **"Notificação (papel)"** na aba Documentos; e o **Painel do SNCR** do `municipal_admin`.

**Architecture:** O cliente HTTP ganha uma seção nova no fim de `src/lib/api.ts` (tipos do contrato do 19d §1–§7 e as funções de `/sncr`, do contato e da Notificação) e alguns campos opcionais nos tipos do 19c (`controlled_list`/`anticonvulsant` no item do catálogo, `category`/`sncr`/`patient_identification`/`prescriber_contact` no conteúdo da receita, `paper_reason` e `short_code` nulo no documento). As regras de tela ficam em `src/lib/controlled.ts` (puras: quem pode, categoria pelos itens, bloqueio por lista, limites, rascunho de endereço e identificação ↔ corpo, previsão de papel, frases do estoque, do retorno do gov.br, da emissão e das recusas, Notificação). Componentes novos: `SncrAccount` e `SncrCallback` (padrão da `SignatureAccount`/`SignatureCallback` do 19b), `SncrOverview` (padrão da `SignatureOverview`), `professionals/ContactPanel`, `documents/AddressFields`, `documents/PatientIdentificationFields`, `documents/NotificationRecordForm`. As telas do 19c ganham o mínimo: `MedicationSearch` (motivo de bloqueio por item), `PrescriptionForm` (contexto `controlled` opcional — sem ele, nada muda), `ConsultationDocuments` (botão da Notificação, aviso da emissão, categoria e número na lista), `CancelDocument` (aviso de que o SNCR não cancela), `ConsultationWorkspace` (paciente no `IssueContext`), `clinicalDocuments.ts` (frases novas), `MyProfile`, `modules.ts`, `App.tsx`, `main.tsx`, `features.ts` e o proxy do Vite (`/sncr`). O dashboard nunca decide o que o api garante: categoria, modo de emissão, número, limites e quem prescreve são do api; a tela espelha para não oferecer o que ele recusaria e mostra o que ele devolveu.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, Vitest 2 + Testing Library 16 (jsdom, sem jest-dom, sem msw: `vi.mock("../lib/api")` e `vi.stubGlobal("fetch")`), Vite 5 (base `/dashboard/`, proxy de dev).

**Spec:** `docs/superpowers/specs/2026-10-10-module-19d-controlled-prescriptions-design.md` (§7 é deste plano; §3–§6 e §8 dão as regras; §11 F-19.25..29), `docs/adr/0034.md` e o contrato `docs/superpowers/plans/2026-10-10-module-19d-controlled-contracts.md` (§1–§7; o que ele ainda não fixa está em "Divergências propostas ao contrato", no fim). Pesquisa: `docs/pesquisa/2026-10-10-receita-controlada-19d.md`.

## Global Constraints

- Rotas (contrato §2–§7; sessão da cidade, cookie):
  - `GET /sncr/stock` → `{ rce: { free, used, voided, requests_this_month }, ret: {…}, low_threshold: 50, simulated }`;
  - `POST /sncr/requests { kind, return_to }` → `{ authorize_url, state }` (409 `monthly_limit_reached`; 403 `cbo_not_allowed`, `registration_mismatch`; plano do api, D2 e D5);
  - `POST /sncr/oauth/callback { state, session_id }` ou `{ state, error }` (plano do api, D1) → `{ kind, received, batch_id, return_to }` (422 `invalid_state`; 409 `authorization_expired`; 403 `authorization_denied`; 503 `sncr_unavailable`, `sncr_exhausted`);
  - `GET /sncr/admin/overview` → `{ professionals: [ { user_id, name, rce_free, ret_free, low, last_request_at } ], simulated }`;
  - `GET`/`PUT /attendance/professional_profile/contact` → `{ address, phone }` (422 `invalid_address` com `field`, `invalid_phone`);
  - `POST /attendance/consultations/:id/documents` com a receita do 19c (+ `patient_identification` na de controle especial) ou `{ kind: "controlled_notification_record", content }`; erros novos 422 `mixed_categories`, `requires_notification` (com `index`), `not_supported` (com `index`), `too_many_c1_substances`, `duration_exceeded` (com `index`), `patient_identification_required` (com `field`), `prescriber_address_missing`, `item_not_in_notification_list`, `notification_type_mismatch`, `invalid_content` (com `field`); 403 `cbo_not_allowed`;
  - `GET /attendance/consultations/:id/patient_identification` → `{ patient_identification|null }` (a última da receita do paciente; plano do api, D4).
- Rota de retorno: o serviço de login da Anvisa (gov.br do prescritor) devolve a pessoa a `/dashboard/sncr/callback?session_id=` (ou `error=`), sem o nosso `state` (plano do api, D1/D2). O `state` que o `POST /sncr/requests` devolve fica no `sessionStorage` da aba **só entre a ida e a volta** e é lido e apagado uma vez; se a URL de volta trouxer `state`, ele vale. A URL é limpa **antes** do POST; a troca roda uma vez (padrão do `SignatureCallback` do 19b). `return_to` = `"/sncr"` (caminho de módulo, Divergência D3 do 19b).
- Valores (contrato §1): `category` ∈ `common` | `special_control` | `antimicrobial`; `sncr.kind` ∈ `rce` | `ret`; estado do número `free` | `used` | `voided`; `notification_type` ∈ `A` | `B` | `B2`; `controlled_list` ∈ `A1`,`A2`,`A3`,`B1`,`B2`,`C1`,`C2`,`C3`,`C4`,`C5` (C4, antirretrovirais, como C2/C3: plano do api, D8); `paper_reason` ∈ `no_certificate` | `signature_unavailable` | `no_sncr_number` | `nurse_antimicrobial` | `feature_disabled`. Valor desconhecido aparece cru, nunca some nem vira `undefined`.
- Interruptores (spec §3): `controlled_prescriptions` (exige `clinical_documents`) libera tudo deste plano; desligado, a receita e a aba Documentos são exatamente as do 19c (controlado bloqueado com "receita de controle especial — 19d"). `sncr_mock` ligado (só fora de produção) → `SNCR_SIMULATED_NOTICE` ("Numeração simulada — sem validade") na Conta → SNCR, na receita, na lista de documentos, no retorno do gov.br e no painel do admin. 403 `feature_disabled` com `feature: "controlled_prescriptions"` → "a receita de controlado está desligada nesta cidade".
- Regras da tela (spec §4–§6, espelho do api): categoria pelos itens — C1/C5 = controle especial, antimicrobiano = antimicrobiano, os dois juntos = misturada (não emite); listas A1–A3/B1/B2 bloqueadas na receita ("registre a Notificação de papel"); C2/C3/C4 bloqueadas ("fora do Rota Saúde"); controle especial: até 3 substâncias C1 diferentes, duração obrigatória em todo item e ≤ 60 dias (≤ 180 para anticonvulsivante; plano do api, D15), identificação do paciente (sempre o CPF do cadastro, que existe por regra do 19a — sem "não possui CPF", decisão do usuário) e endereço completo, endereço e telefone do prescritor (na de antimicrobiano, só quando ela sairia digital: plano do api, D12); validade 30 dias (controle especial) e 10 (antimicrobiano), 2 vias; Notificação: tipo pela lista (A1–A3 → A, B1 → B, B2 → B2), número do talão, UF da numeração, uma substância, duração ≤ 30 (A) ou ≤ 60 (B/B2). A previsão "vai sair em papel" (sem certificado ativo, sem assinatura na cidade, sem número no estoque, enfermagem com antimicrobiano) é **previsão**: o modo é o que o api devolve, e o aviso depois da emissão usa o `paper_reason` dele.
- Quem: Conta → SNCR é do `health_professional` com `controlled_prescriptions` (o api recusa enfermeiro com `cbo_not_allowed`); "Notificação (papel)" só médico (`2251`–`2253`) e dentista (`2232`); Painel do SNCR só `municipal_admin`; o operador nunca vê nada disto.
- Segurança e LGPD (spec §8): CPF, endereço, telefone, número SNCR, `state` e `session_id` só no corpo de POST/PUT, nunca na URL que a tela monta nem em `console`, `localStorage` ou nome de arquivo; o `session_id` nunca é guardado; o `state` (que só vale com a sessão do próprio usuário no api) fica no `sessionStorage` só entre a ida e a volta, e sai de lá na volta; o token do SNCR nunca passa pelo navegador; leituras com dado pessoal (`contact`, `patient_identification`, documentos) com `gcTime: 0`; a tela recebe o CPF do paciente **mascarado** (do prontuário) e nunca o envia (Divergência D2).
- Toda escrita por cookie leva `Content-Type: application/json` (o `jsonFetch` põe sozinho quando há corpo; `PUT` com corpo). Nenhuma escrita do 19d pede step-up (Divergência D14); cancelar continua com o step-up do 19c.
- Interface tem ciclo próprio: use os componentes e estilos existentes (`Panel`, `PageHeader`, `DataTable`, `Tag`, `KeyValue`, `EmptyState`, `SensitiveAction`, `formStyles`, os `formStyles` de `src/modules/documents/`), sem redesign. Nenhum componente comum muda.
- Testes que dependem de "agora" fixam o relógio com `vi.useFakeTimers({ toFake: [ "Date" ] })` + `vi.setSystemTime(new Date(NOW19D))` (`NOW19D = "2026-10-10T10:00:00-03:00"`) e `vi.useRealTimers()` no `afterEach`; a sessão de teste é criada **depois** do `setSystemTime` (o `mfa_verified_at` dela é "agora").
- Nunca `git add -A` nem `git add .`: o worktree tem o symlink de `node_modules`. Adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Merge, push e tag só com autorização explícita do usuário.

## Ambiente de execução

- Worktree `apps/dashboard/.claude/mod19d`, branch `feat/mod-19d-controlled` a partir de `origin/main` (Task 0). Todos os caminhos de arquivo das tasks são relativos a ele; os comandos partem da raiz do monorepo (`/Users/eduardovrocha/Development/ioit.solutions/rota-saude`).
- Testes (script `test` = `vitest run`): `cd apps/dashboard/.claude/mod19d && npx vitest run <arquivos>`.
- Tipos antes de cada commit (`typecheck` = `tsc --noEmit`; `noUnusedLocals` ligado e o `tsconfig` inclui os testes): `cd apps/dashboard/.claude/mod19d && npx tsc --noEmit`.
- Ambiente de teste: `vitest.config.ts` fixa `TZ=America/Sao_Paulo` e não define `base` (`import.meta.env.BASE_URL` é `/`); sem `setupFiles` nem jest-dom (use `not.toBeNull()`, `toBeNull()`, `toBe()`, `(el as HTMLInputElement).value`, `(el as HTMLButtonElement).disabled`); `globals: false`, então todo teste de componente chama `cleanup` no `afterEach`.
- Os trechos "antes" dos diffs sobre arquivos do 19c vêm do **plano** do dashboard do 19c (código das Tasks 1–6). Confira o trecho no arquivo real (a Task 0 manda); se ele mudou na execução do 19c, aplique a mesma intenção sobre o texto que estiver lá. Os trechos sobre arquivos do 19b vêm de `origin/main` `8a0c0de`.
- O api do 19d roda em dev na porta **3039** (plano do api do 19d). O Vite do worktree sobe **num container da rede do compose**, na **5189**, com o proxy para `http://api:3039` (Task 10).

## Review Focus

1. **O estoque zera (ou o certificado vence) entre abrir a receita e emitir.** A tela previu "digital", mas o api devolve `issue_mode: "paper"` com `paper_reason: "no_sncr_number"`. A frase depois da emissão diz que saiu em papel, **por quê** e que são 2 vias para imprimir e assinar à mão — nunca "a assinatura entra na sua fila". Teste: Task 8, "previsão digital, mas o api devolve papel (no_sncr_number): diz o motivo e manda imprimir 2 vias".
2. **A volta do login do SNCR dá errado.** A pessoa demora mais de 30 s, recusa no gov.br, cola de novo a URL de volta (o `state` já saiu do `sessionStorage`) ou volta noutra aba, ou o SNCR está fora/esgotado. A tela diz o que houve em português, que **nenhum número** foi pedido (ou gasto), troca o `session_id` uma vez só e volta à Conta → SNCR. Testes: Task 5, "autorização vencida (409 authorization_expired)…", "SNCR esgotado (503 sncr_exhausted) e state já usado…", "recusa no gov.br com state…", "sem state (página de retorno recarregada, outra aba)…", "StrictMode: troca o session_id uma vez só".
3. **Receita que o api recusaria por regra de controle.** Sertralina (C1) com amoxicilina na mesma receita, quatro substâncias C1, 90 dias de sertralina ou 200 de carbamazepina (anticonvulsivante, até 180): a tela diz antes de mandar; e a recusa do api (`too_many_c1_substances`, `duration_exceeded` com `index`) aparece no formulário, que fica preenchido (endereço do paciente incluído). Testes: Task 7, "mistura controle especial e antimicrobiano…", "quatro substâncias C1…", "duração acima de 60 dias, e de 180 no anticonvulsivante", "recusa do api fica no formulário, com o endereço".
4. **Medicamento de Notificação ou fora do Rota Saúde na busca da receita.** Morfina (A1) e clonazepam (B1) aparecem bloqueados com o caminho ("registre a Notificação de papel"); isotretinoína (C2) bloqueada como "fora do Rota Saúde"; e na Notificação, item fora das listas A/B ou tipo que não bate com a lista do item não sai. Testes: Task 7, "A/B bloqueados com o caminho da Notificação; C2/C3 fora; C1 entra"; Task 6, "tipo que não bate com a lista do item…" e "item fora das listas A/B fica bloqueado na busca".
5. **Cancelar uma receita digital numerada.** O SNCR não tem cancelamento: o diálogo avisa que o número fica anulado (não volta ao estoque) e que a farmácia só vê o cancelamento na conferência do Rota Saúde; depois, o saldo é relido. Teste: Task 8, "cancelar RCE: avisa que o SNCR não cancela e relê o estoque".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | conferência do main (19c entregue) e do api do 19d, worktree, base | 0 |
| `src/lib/api.ts`, `src/lib/api.controlled.test.ts`, `src/test/controlledFixtures.ts`, `src/lib/features.ts`, `src/lib/features.test.ts`, `vite.config.ts`, `README.md` | tipos do contrato do 19d e campos novos nos do 19c; cliente; interruptor `controlled_prescriptions`; proxy `/sncr` | 1 |
| `src/lib/controlled.ts`, `src/lib/controlled.test.ts`, `src/lib/clinicalDocuments.ts`, `src/lib/clinicalDocuments.test.ts` | regras de tela do 19d; frases novas e o rótulo da Notificação nas regras do 19c | 2 |
| `src/modules/documents/AddressFields.tsx`, `src/modules/professionals/ContactPanel.tsx` (+ teste), `src/modules/MyProfile.tsx`, `src/modules/MyProfile.contact.test.tsx` | endereço e telefone do prescritor em Meu perfil | 3 |
| `src/modules/SncrAccount.tsx` (+ teste), `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/App.tsx` | Conta → SNCR | 4 |
| `src/modules/SncrCallback.tsx` (+ teste), `src/main.tsx` | retorno do gov.br em `/dashboard/sncr/callback` | 5 |
| `src/modules/documents/MedicationSearch.tsx`, `src/modules/documents/MedicationSearch.test.tsx`, `src/modules/documents/NotificationRecordForm.tsx` (+ teste) | motivo de bloqueio por item na busca; formulário da Notificação de papel | 6 |
| `src/modules/documents/PatientIdentificationFields.tsx`, `src/modules/documents/PrescriptionForm.tsx`, `src/modules/documents/PrescriptionForm.controlled.test.tsx` | receita: categoria, identificação do paciente, prescritor, limites, previsão de papel, faixa de simulado | 7 |
| `src/modules/documents/ConsultationDocuments.tsx`, `src/modules/documents/CancelDocument.tsx`, `src/modules/consultation/ConsultationWorkspace.tsx`, `src/modules/documents/ConsultationDocuments.controlled.test.tsx` | aba Documentos: Notificação, aviso da emissão com `paper_reason`, categoria e número na lista, cancelamento | 8 |
| `src/modules/SncrOverview.tsx` (+ teste), `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/App.tsx` | Painel do SNCR do admin | 9 |
| — | suíte, build, revisão e prova no navegador | 10 |

**Estratégia de teste:** regras puras com tabela de casos (Task 2); cliente HTTP com `fetch` falso conferindo URL, método, cabeçalho e corpo — e que `state`, `session_id`, CPF e endereço nunca vão na URL (Task 1); cada componente com `vi.mock("../lib/api")` cobrindo o caminho feliz, cada recusa que muda o comportamento da tela e as bordas do Review Focus; o formulário da receita testado solto (o `onSubmit` é um `vi.fn()`, contexto `controlled` passado por prop) e o contêiner da aba com dublês da API; o retorno do gov.br com `StrictMode` e espiões de `console`/storage. A prova final é no navegador, contra o api do 19d com o SNCR e o PSC simulados (Task 10).

---

### Task 0: Conferência do main (19c entregue), worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do dashboard com o 19c.
- Produces: worktree `apps/dashboard/.claude/mod19d` na branch `feat/mod-19d-controlled`; a base anotada (arquivos e testes verdes).

- [ ] **Step 1: Confira que o 19c chegou e as peças que este plano altera**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude
/opt/homebrew/bin/git -C apps/dashboard fetch origin
/opt/homebrew/bin/git -C apps/dashboard log --oneline -5 origin/main
for f in src/lib/clinicalDocuments.ts src/test/documentFixtures.ts src/modules/documents/PrescriptionForm.tsx \
         src/modules/documents/MedicationSearch.tsx src/modules/documents/ConsultationDocuments.tsx \
         src/modules/documents/CancelDocument.tsx src/modules/documents/formStyles.ts \
         src/modules/consultation/ConsultationWorkspace.tsx src/modules/SignatureCallback.tsx src/modules/MyProfile.tsx; do
  /opt/homebrew/bin/git -C apps/dashboard cat-file -e origin/main:$f && echo "ok $f"
done
/opt/homebrew/bin/git -C apps/dashboard show origin/main:src/lib/api.ts | grep -n \
  "export interface CatalogItemRef\|export interface PrescriptionContent\|export interface PrescriptionInput\|export interface ClinicalDocument\|const attendanceDocPath\|export interface AuthorizeRedirect\|short_code: string"
/opt/homebrew/bin/git -C apps/dashboard show origin/main:src/lib/features.ts | grep -n "export type FeatureKey"
/opt/homebrew/bin/git -C apps/dashboard show origin/main:src/lib/clinicalDocuments.ts | grep -n \
  "export interface PrescriptionDraft\|if (it.medication?.controlled)\|export function prescriptionProblems\|const KIND_LABEL\|if (feature !== null) return DOCUMENTS_DISABLED\|case \"prescription\": {"
/opt/homebrew/bin/git -C apps/dashboard show origin/main:src/modules/documents/ConsultationDocuments.tsx | grep -n \
  "export interface IssueContext\|setNotice(issuedNotice(doc))\|<span className=\"mono\">{d.short_code}</span>\|{issue && canCancel(d, user?.id) && ("
/opt/homebrew/bin/git -C apps/dashboard show origin/main:src/modules/consultation/ConsultationWorkspace.tsx | grep -n "issue={user?.id === consultation.author.id"
/opt/homebrew/bin/git -C apps/dashboard show origin/main:vite.config.ts | grep -n '"/clinical_documents": proxy(TARGET)'
```

Expected:
- `origin/main` com os commits do dashboard do 19c (mais novo que `8a0c0de`); dez linhas `ok …`. **Se `clinicalDocuments.ts` ou `PrescriptionForm.tsx` não existirem, pare e reporte**: o 19c ainda não foi entregue (pré-requisito).
- em `api.ts`: as declarações `CatalogItemRef`, `PrescriptionContent`, `PrescriptionInput`, `ClinicalDocument` (com `short_code: string; verification_url: string;`), `attendanceDocPath` e `AuthorizeRedirect`;
- `export type FeatureKey = "ledi_export" | "cadsus_lookup" | "clinical_record" | "digital_signature" | "clinical_documents";`;
- em `clinicalDocuments.ts`: `PrescriptionDraft`, a linha `if (it.medication?.controlled) out.push(…)` dentro de `prescriptionProblems`, `KIND_LABEL`, a linha do `DOCUMENTS_DISABLED` em `documentError` e o `case "prescription": {` de `reissueDraft`;
- em `ConsultationDocuments.tsx`: `IssueContext`, `setNotice(issuedNotice(doc))`, a coluna do código e o botão "Cancelar e emitir outro"; em `ConsultationWorkspace.tsx`, a linha do `issue=`; no `vite.config.ts`, a linha do proxy do 19c.

Se alguma linha mudou na execução do 19c, anote: as tasks aplicam a mesma intenção sobre o texto real.

- [ ] **Step 2: Confira o api do 19d no ar**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude
/opt/homebrew/bin/git -C apps/api branch --list feat/mod-19d-controlled
docker compose exec -T -w /rails/.claude/mod19d api bin/rails routes 2>/dev/null | grep -E "sncr|professional_profile|patient_identification" | head
```

Expected: a branch do api do 19d e as rotas `/sncr/stock`, `/sncr/requests`, `/sncr/oauth/callback`, `/sncr/admin/overview`, `/attendance/professional_profile/contact` e `/attendance/consultations/:id/patient_identification`. Sem o api do 19d, as Tasks 1–9 seguem (os testes não dependem dele); a Task 10 espera.

- [ ] **Step 3: Crie o worktree e ligue o `node_modules`** (`/.claude/` já está no `.gitignore` do dashboard)

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude
/opt/homebrew/bin/git -C apps/dashboard worktree add .claude/mod19d -b feat/mod-19d-controlled origin/main
ln -s ../../node_modules apps/dashboard/.claude/mod19d/node_modules
cd apps/dashboard/.claude/mod19d && npx vitest run 2>&1 | tail -3 && npx tsc --noEmit && echo tsc-ok
```

Expected: tudo verde e `tsc-ok`. **Anote** a base (arquivos e testes). A Task 10 espera a base mais **10 arquivos de teste novos**.

---

### Task 1: Tipos do 19d, cliente HTTP, interruptor, proxy e fixtures

**Files:**
- Modify: `src/lib/api.ts` (campos opcionais nos tipos do 19c; seção nova no fim do arquivo)
- Modify: `src/lib/features.ts`, `src/lib/features.test.ts`, `vite.config.ts`, `README.md`
- Create: `src/test/controlledFixtures.ts`
- Test: `src/lib/api.controlled.test.ts`

**Interfaces:**
- Consumes: `jsonFetch`, `postProfessional`, `attendanceDocPath`, `ATTENDANCE_BASE`, `ApiError`, `MedicationRoute`, `PrescriptionItem`, `ClinicalDocument`, `CatalogItemRef` (já em `api.ts`); `sessionWith` (`campaignFixtures`); `searchItem`, `catalogItem`, `prescriptionItem`, `prescriptionDoc`, `AMOXICILLIN` (`documentFixtures`, 19c).
- Produces:
  - tipos: `PrescriptionCategory`, `SncrKind`, `ControlledList`, `NotificationType`, `PaperReason`, `SncrAddress`, `SncrNumberRef`, `PatientIdentification`, `PatientIdentificationInput`, `PrescriberContact`, `ProfessionalContact`, `NotificationRecordInput`, `NotificationRecordContent`, `RecordedDocumentKind`, `SncrStockKind`, `SncrStock`, `SncrAuthorizeRedirect`, `SncrCallbackResult`, `SncrOverviewProfessional`, `SncrOverview`;
  - campos novos nos tipos do 19c: `CatalogItemRef.controlled_list?: ControlledList | null`, `CatalogItemRef.anticonvulsant?: boolean`; `PrescriptionContent.category?`, `.sncr?`, `.patient_identification?`, `.prescriber_contact?`; `PrescriptionInput.patient_identification?`; `ClinicalDocument.kind: RecordedDocumentKind`, `.short_code: string | null`, `.verification_url: string | null`, `.paper_reason?: PaperReason | string | null`; `DocumentContent` inclui `NotificationRecordContent`;
  - funções: `getSncrStock(): Promise<SncrStock>`, `requestSncrNumbers(kind: SncrKind, returnTo: string): Promise<SncrAuthorizeRedirect>` (`{ authorize_url, state }`), `completeSncrOAuth(state: string, answer: string | { error: string }): Promise<SncrCallbackResult>` (a string é o `session_id`), `getSncrOverview(): Promise<SncrOverview>`, `getProfessionalContact(): Promise<ProfessionalContact>`, `saveProfessionalContact(contact: { address: SncrAddress; phone: string }): Promise<ProfessionalContact>`, `recordNotification(consultationId: string, content: NotificationRecordInput): Promise<ClinicalDocument>`, `getLastPatientIdentification(consultationId: string): Promise<PatientIdentification | null>`;
  - `FeatureKey` com `"controlled_prescriptions"` (rótulo "Receita de controle especial e de antimicrobiano (SNCR)");
  - fixtures (`src/test/controlledFixtures.ts`): `NOW19D`, `TODAY19D`, `controlledUser(roles?, over?)` (sessão `us1` com `clinical_record`, `digital_signature`, `clinical_documents` e `controlled_prescriptions`), `SERTRALINE`, `AMITRIPTYLINE`, `FLUOXETINE`, `CARBAMAZEPINE` (C1; a última anticonvulsivante), `MORPHINE` (A1), `CLONAZEPAM_B1` (B1), `ISOTRETINOIN` (C2), `SERTRALINE_REF` (`CatalogItemRef`), `address(over?)`, `prescriberContact(over?)`, `contact(over?)`, `stock(over?)`, `rceDoc(over?)`, `retDoc(over?)`, `notificationDoc(over?)`, `sncrOverview(over?)`.

- [ ] **Step 1: Write the failing test**

Fixtures (usadas nesta e nas próximas tasks):

```ts
// src/test/controlledFixtures.ts
// Dados comuns aos testes do módulo 19d (receita de controlado, SNCR).
// Relógio: sábado, 2026-10-10 10:00 em São Paulo (-03:00). A autora é `us1`
// (médica, CBO 225142), a mesma das fixtures do 19a/19b/19c. Endereços,
// telefone e números SNCR são fictícios; o CPF do paciente vem mascarado.
import type {
  CatalogItemRef, ClinicalDocument, MedicationSearchItem, PrescriberContact, PrescriptionContent, ProfessionalContact,
  SessionUser, SncrAddress, SncrOverview, SncrStock
} from "../lib/api";
import { sessionWith } from "./campaignFixtures";
import { catalogItem, prescriptionDoc, prescriptionItem, searchItem, AMOXICILLIN } from "./documentFixtures";

export const NOW19D = "2026-10-10T10:00:00-03:00";
export const TODAY19D = "2026-10-10";

// Janela de step-up aberta (sessionWith põe `mfa_verified_at` = agora): crie a
// sessão DEPOIS de fixar o relógio.
export function controlledUser(roles: string[] = [ "health_professional" ], over: Partial<SessionUser> = {}): SessionUser {
  return sessionWith(roles, {
    id: "us1", email_address: "medica@curitiba.demo",
    features: [ "clinical_record", "digital_signature", "clinical_documents", "controlled_prescriptions" ], ...over
  });
}

const c1 = (over: Partial<MedicationSearchItem>) =>
  searchItem({ controlled: true, controlled_list: "C1", anticonvulsant: false, antimicrobial: false, in_network: true, ...over });

export const SERTRALINE = c1({
  id: "ci10", catmat_code: 267503, label: "Sertralina, cloridrato 50 mg, comprimido", active_ingredient: "cloridrato de sertralina",
  strength: "50 mg", dosage_form: "comprimido"
});
export const AMITRIPTYLINE = c1({
  id: "ci11", catmat_code: 267504, label: "Amitriptilina, cloridrato 25 mg, comprimido", active_ingredient: "cloridrato de amitriptilina",
  strength: "25 mg", dosage_form: "comprimido"
});
export const FLUOXETINE = c1({
  id: "ci12", catmat_code: 267505, label: "Fluoxetina, cloridrato 20 mg, cápsula", active_ingredient: "cloridrato de fluoxetina",
  strength: "20 mg", dosage_form: "cápsula"
});
export const CARBAMAZEPINE = c1({
  id: "ci13", catmat_code: 267506, label: "Carbamazepina 200 mg, comprimido", active_ingredient: "carbamazepina",
  strength: "200 mg", dosage_form: "comprimido", anticonvulsant: true
});
export const MORPHINE = searchItem({
  id: "ci14", catmat_code: 267507, label: "Morfina, sulfato 10 mg, comprimido", active_ingredient: "sulfato de morfina",
  strength: "10 mg", dosage_form: "comprimido", controlled: true, controlled_list: "A1", in_network: false
});
export const CLONAZEPAM_B1 = searchItem({
  id: "ci3", catmat_code: 269999, label: "Clonazepam 2 mg, comprimido", active_ingredient: "clonazepam",
  strength: "2 mg", dosage_form: "comprimido", controlled: true, controlled_list: "B1", in_network: false
});
export const ISOTRETINOIN = searchItem({
  id: "ci15", catmat_code: 267508, label: "Isotretinoína 20 mg, cápsula", active_ingredient: "isotretinoína",
  strength: "20 mg", dosage_form: "cápsula", controlled: true, controlled_list: "C2", in_network: false
});
export const AMOXICILLIN_19D: MedicationSearchItem = { ...AMOXICILLIN, controlled_list: null, anticonvulsant: false };
export const SERTRALINE_REF: CatalogItemRef = catalogItem({
  id: "ci10", catmat_code: 267503, label: "Sertralina, cloridrato 50 mg, comprimido", active_ingredient: "cloridrato de sertralina",
  strength: "50 mg", dosage_form: "comprimido", controlled_list: "C1", anticonvulsant: false
});

export function address(over: Partial<SncrAddress> = {}): SncrAddress {
  return { street: "Rua Marechal Deodoro", number: "630", complement: "apto 12", district: "Centro", city: "Curitiba", uf: "PR",
    zip: "80010010", ...over };
}

export function prescriberContact(over: Partial<PrescriberContact> = {}): PrescriberContact {
  return {
    address: { street: "Rua Ouvidor Pardinho", number: "28", district: "Rebouças", city: "Curitiba", uf: "PR", zip: "80230040" },
    phone: "4133501234", ...over
  };
}

export function contact(over: Partial<ProfessionalContact> = {}): ProfessionalContact {
  return { ...prescriberContact(), ...over };
}

export function stock(over: Partial<SncrStock> = {}): SncrStock {
  return {
    rce: { free: 812, used: 180, voided: 8, requests_this_month: 1 },
    ret: { free: 40, used: 960, voided: 0, requests_this_month: 2 },
    low_threshold: 50, simulated: false, ...over
  };
}

function rceContent(over: Partial<PrescriptionContent> = {}): PrescriptionContent {
  return {
    items: [ prescriptionItem({ catalog_item: SERTRALINE_REF, printed_description: "Cloridrato de sertralina 50 mg — comprimido",
      quantity: 60, dosage_instructions: "1 comprimido pela manhã", duration_days: 60, continuous: true }) ],
    antimicrobial: false, copies: 2, valid_until: "2026-11-09", category: "special_control",
    sncr: { kind: "rce", number: "2610.1-41.0001234", simulated: false },
    patient_identification: { cpf: "39053344705", address: address() },
    prescriber_contact: prescriberContact(), ...over
  };
}

export function rceDoc(over: Partial<ClinicalDocument> = {}, content: Partial<PrescriptionContent> = {}): ClinicalDocument {
  return prescriptionDoc({
    id: "doc20", issue_mode: "digital", issued_at: "2026-10-10T09:40:00-03:00", short_code: "R5C8E2N7K4",
    content: rceContent(content), signature: { mode: "pending", request_id: "sr20" }, paper_reason: null, ...over
  });
}

export function retDoc(over: Partial<ClinicalDocument> = {}): ClinicalDocument {
  return prescriptionDoc({
    id: "doc21", issue_mode: "digital", issued_at: "2026-10-10T09:45:00-03:00", short_code: "T4N9W2K7R3",
    content: {
      items: [ prescriptionItem({ catalog_item: catalogItem({ id: "ci2", label: "Amoxicilina 500 mg, cápsula" }), antimicrobial: true,
        quantity: 21, quantity_unit: "cápsula", duration_days: 7, continuous: false }) ],
      antimicrobial: true, copies: 2, valid_until: "2026-10-20", category: "antimicrobial",
      sncr: { kind: "ret", number: "2610.2-41.0000501", simulated: false }, prescriber_contact: prescriberContact()
    },
    signature: { mode: "pending", request_id: "sr21" }, paper_reason: null, ...over
  });
}

export function notificationDoc(over: Partial<ClinicalDocument> = {}): ClinicalDocument {
  return prescriptionDoc({
    id: "doc22", kind: "controlled_notification_record", issue_mode: "paper", issued_at: "2026-10-10T09:50:00-03:00",
    short_code: null, verification_url: null, signature: null,
    content: { notification_type: "B", paper_number: "PR0012345", numbering_uf: "PR",
      item: prescriptionItem({ catalog_item: catalogItem({ id: "ci3", label: "Clonazepam 2 mg, comprimido", controlled_list: "B1" }),
        quantity: 30, dosage_instructions: "1 comprimido à noite", duration_days: 30, continuous: true }) },
    ...over
  });
}

export function sncrOverview(over: Partial<SncrOverview> = {}): SncrOverview {
  return {
    professionals: [
      { user_id: "us1", name: "Dra. Helena Prado", rce_free: 812, ret_free: 40, low: true, last_request_at: "2026-10-02T14:10:00-03:00" },
      { user_id: "us2", name: "Dr. Caio Mendes", rce_free: 950, ret_free: 700, low: false, last_request_at: "2026-10-05T09:00:00-03:00" },
      { user_id: "us3", name: "Dra. Bruna Lopes", rce_free: 0, ret_free: 0, low: true, last_request_at: null }
    ],
    simulated: false, ...over
  };
}
```

```ts
// src/lib/api.controlled.test.ts
import { afterEach, describe, expect, it, vi } from "vitest";
import {
  completeSncrOAuth, getLastPatientIdentification, getProfessionalContact, getSncrOverview, getSncrStock, recordNotification,
  requestSncrNumbers, saveProfessionalContact, type NotificationRecordInput
} from "./api";
import { address, contact, notificationDoc, sncrOverview, stock } from "../test/controlledFixtures";

afterEach(() => vi.unstubAllGlobals());

function stub(body: unknown, status = 200) {
  const fn = vi.fn(async (_input: RequestInfo | URL, _init?: RequestInit) =>
    new Response(body === undefined ? null : JSON.stringify(body), { status, headers: { "Content-Type": "application/json" } }));
  vi.stubGlobal("fetch", fn);
  return fn;
}
const call = (fn: ReturnType<typeof stub>, i = 0) => fn.mock.calls[i] as [ string, RequestInit ];
// Toda escrita por cookie exige JSON (CSRF do api): método, header e corpo.
function expectJsonWrite(fn: ReturnType<typeof stub>, method: string, body: unknown, i = 0) {
  expect(call(fn, i)[1].method).toBe(method);
  expect((call(fn, i)[1].headers as Record<string, string>)["Content-Type"]).toBe("application/json");
  expect(JSON.parse(call(fn, i)[1].body as string)).toEqual(body);
}

describe("cliente do SNCR", () => {
  it("lê o estoque do usuário corrente com a sessão", async () => {
    const fn = stub(stock());
    expect((await getSncrStock()).rce.free).toBe(812);
    expect(call(fn)[0]).toBe("/sncr/stock");
    expect(call(fn)[1].credentials).toBe("include");
  });

  it("pede números com kind e return_to no corpo e devolve o authorize_url e o state", async () => {
    const fn = stub({ authorize_url: "https://sncr-auth.example.br/auth/login?client_url=x", state: "ST-1" });
    expect(await requestSncrNumbers("rce", "/sncr")).toEqual({ authorize_url: "https://sncr-auth.example.br/auth/login?client_url=x", state: "ST-1" });
    expect(call(fn)[0]).toBe("/sncr/requests");
    expectJsonWrite(fn, "POST", { kind: "rce", return_to: "/sncr" });
  });

  it("devolve state e session_id (ou o erro da volta) só no corpo", async () => {
    let fn = stub({ kind: "ret", received: 1000, batch_id: "b1", return_to: "/sncr" });
    expect((await completeSncrOAuth("ST-1", "SID-1")).received).toBe(1000);
    expect(call(fn)[0]).toBe("/sncr/oauth/callback");
    expectJsonWrite(fn, "POST", { state: "ST-1", session_id: "SID-1" });
    expect(call(fn)[0]).not.toMatch(/ST-1|SID-1/);
    fn = stub({ error: "authorization_denied", return_to: "/sncr" }, 403);
    await expect(completeSncrOAuth("ST-2", { error: "access_denied" })).rejects.toMatchObject({ status: 403 });
    expectJsonWrite(fn, "POST", { state: "ST-2", error: "access_denied" });
  });

  it("painel do admin", async () => {
    const fn = stub(sncrOverview());
    expect((await getSncrOverview()).professionals).toHaveLength(3);
    expect(call(fn)[0]).toBe("/sncr/admin/overview");
  });
});

describe("cliente do contato do profissional e da Notificação", () => {
  it("contato: lê e grava com PUT JSON; endereço e telefone só no corpo", async () => {
    let fn = stub(contact());
    expect((await getProfessionalContact()).phone).toBe("4133501234");
    expect(call(fn)[0]).toBe("/attendance/professional_profile/contact");
    fn = stub(contact());
    const body = { address: address(), phone: "4133501234" };
    await saveProfessionalContact(body);
    expect(call(fn)[0]).toBe("/attendance/professional_profile/contact");
    expectJsonWrite(fn, "PUT", body);
  });

  it("registra a Notificação pela rota de documentos da consulta", async () => {
    const fn = stub(notificationDoc(), 201);
    const content: NotificationRecordInput = {
      notification_type: "B", paper_number: "PR0012345", numbering_uf: "PR",
      item: { catalog_item: { id: "ci3" }, quantity: 30, quantity_unit: "comprimido", route: "oral",
        dosage_instructions: "1 comprimido à noite", duration_days: 30 }
    };
    expect((await recordNotification("cs1", content)).short_code).toBeNull();
    expect(call(fn)[0]).toBe("/attendance/consultations/cs1/documents");
    expectJsonWrite(fn, "POST", { kind: "controlled_notification_record", content });
  });

  it("última identificação do paciente pela consulta: a identificação, null, ou null no 404; outra recusa sobe", async () => {
    const fn = stub({ patient_identification: { cpf: "39053344705", address: address() } });
    expect((await getLastPatientIdentification("cs1"))?.address.zip).toBe("80010010");
    expect(call(fn)[0]).toBe("/attendance/consultations/cs1/patient_identification");
    stub({ patient_identification: null });
    expect(await getLastPatientIdentification("cs1")).toBeNull();
    stub({ error: "not_found" }, 404);
    expect(await getLastPatientIdentification("cs1")).toBeNull();
    stub({ error: "not_author" }, 403);
    await expect(getLastPatientIdentification("cs1")).rejects.toMatchObject({ status: 403 });
  });
});
```

Em `src/lib/features.test.ts`, no teste "rótulo conhecido em português; desconhecido sai como a chave", logo depois da linha de `clinical_documents` (19c):

```ts
    expect(featureLabel("controlled_prescriptions")).toBe("Receita de controle especial e de antimicrobiano (SNCR)");
    expect(hasFeature({ features: [ "controlled_prescriptions" ] }, "controlled_prescriptions")).toBe(true);
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod19d && npx vitest run src/lib/api.controlled.test.ts src/lib/features.test.ts`
Expected: FAIL — as funções novas não existem em `./api`, a fixture não compila (`controlled_list` fora de `CatalogItemRef`) e `featureLabel("controlled_prescriptions")` devolve a chave crua.

- [ ] **Step 3: Write minimal implementation**

Em `src/lib/api.ts`, nos tipos do 19c (seção "Documentos clínicos"):

(a) `CatalogItemRef` ganha os dois campos do catálogo do 19d (Divergência D1; ausentes no api do 19c):

```ts
export interface CatalogItemRef {
  id: string; catmat_code: number; label: string; active_ingredient: string; strength: string; dosage_form: string;
  // Módulo 19d (Divergência D1): lista da Portaria 344 e marca de anticonvulsivante.
  controlled_list?: ControlledList | null; anticonvulsant?: boolean;
}
```

(b) `PrescriptionContent` e `PrescriptionInput`:

```ts
export interface PrescriptionContent {
  items: PrescriptionItem[];
  nursing_protocol?: { id: string; title: string; number: string; year: number; version_id: string };
  city_cnpj?: string; antimicrobial: boolean; copies: 1 | 2; valid_until?: string;
  // Módulo 19d (contrato §2): calculados pelo api; `sncr` só no modo digital.
  category?: PrescriptionCategory; sncr?: SncrNumberRef | null;
  patient_identification?: PatientIdentification | null; prescriber_contact?: PrescriberContact | null;
}
```

```ts
export interface PrescriptionInput {
  items: PrescriptionItemInput[]; nursing_protocol?: { version_id: string };
  // Módulo 19d: só na receita de controle especial (Divergência D2: sem CPF; o api usa o do cadastro).
  patient_identification?: PatientIdentificationInput;
}
```

(c) `DocumentContent` e `ClinicalDocument`:

```ts
export type DocumentContent = SickNoteContent | DeclarationContent | PrescriptionContent | ExamRequisitionContent | NotificationRecordContent;
```

```ts
export interface ClinicalDocument {
  id: string; kind: RecordedDocumentKind; status: DocumentStatus; issue_mode: IssueMode; issued_at: string;
  cancelled_at: string | null; cancel_reason: string | null; author: DocumentAuthor;
  patient: { id: string; display_name: string } | null; consultation_id: string | null; attendance_id: string;
  // Nulos no registro da Notificação de papel (contrato do 19d §3).
  short_code: string | null; verification_url: string | null; replaces_document_id: string | null;
  content: DocumentContent; signature: SignatureBlock | null;
  // Módulo 19d (contrato §2; Divergência D8): por que saiu em papel.
  paper_reason?: PaperReason | string | null;
}
```

No fim de `src/lib/api.ts`:

```ts
// ─── Receita de controlado e SNCR (módulo 19d, ADR 0034; contrato 2026-10-10 §1–§7) ───
// CPF, endereço, telefone, número SNCR, `state` e `session_id` só no corpo de
// POST/PUT, nunca em URL: as rotas levam só ids. O token do SNCR nunca passa
// pelo navegador: o serviço de login da Anvisa faz o gov.br e devolve um
// `session_id` de uso único (30 s), que o api troca pelo token e usa no pedido
// dos números, no mesmo fluxo (plano do api, D1). Toda escrita leva JSON (o
// jsonFetch põe o Content-Type quando há corpo).

const SNCR_BASE = import.meta.env.VITE_SNCR_BASE || "/sncr";

export type PrescriptionCategory = "common" | "special_control" | "antimicrobial";
export type SncrKind = "rce" | "ret";
export type ControlledList = "A1" | "A2" | "A3" | "B1" | "B2" | "C1" | "C2" | "C3" | "C4" | "C5";
export type NotificationType = "A" | "B" | "B2";
export type PaperReason = "no_certificate" | "signature_unavailable" | "no_sncr_number" | "nurse_antimicrobial" | "feature_disabled";
export type RecordedDocumentKind = DocumentKind | "controlled_notification_record";

export interface SncrAddress {
  street: string; number: string; complement?: string | null; district: string; city: string; uf: string; zip: string;
}
export interface SncrNumberRef { kind: SncrKind; number: string; simulated: boolean }
// Sempre o CPF do cadastro (regra do 19a); sem "não possui CPF" (decisão do usuário, 2026-10-10).
export interface PatientIdentification { cpf: string; address: SncrAddress }
// Divergência D2: a entrada não leva o CPF — o api usa o do cadastro.
export interface PatientIdentificationInput { address: SncrAddress }
export interface PrescriberContact { address: SncrAddress; phone: string }
// Divergência D4: sem cadastro, os dois vêm nulos (200, não 404).
export interface ProfessionalContact { address: SncrAddress | null; phone: string | null }

export interface NotificationRecordInput {
  notification_type: NotificationType; paper_number: string; numbering_uf: string;
  item: {
    catalog_item: { id: string }; quantity: number; quantity_unit: string; route: MedicationRoute;
    dosage_instructions: string; duration_days: number;
  };
}
// Divergência D13: a saída repete a entrada com o item na forma do 19c.
export interface NotificationRecordContent {
  notification_type: NotificationType; paper_number: string; numbering_uf: string; item: PrescriptionItem;
}

export interface SncrStockKind { free: number; used: number; voided: number; requests_this_month: number }
export interface SncrStock { rce: SncrStockKind; ret: SncrStockKind; low_threshold: number; simulated: boolean }
// O `state` volta na resposta: a volta da Anvisa não o traz (plano do api, D2).
export interface SncrAuthorizeRedirect { authorize_url: string; state: string }
export interface SncrCallbackResult { kind: SncrKind; received: number; batch_id: string; return_to?: string }
export interface SncrOverviewProfessional {
  user_id: string; name: string; rce_free: number; ret_free: number; low: boolean; last_request_at: string | null;
}
export interface SncrOverview { professionals: SncrOverviewProfessional[]; simulated: boolean }

const sncrPath = (...parts: string[]) => [ SNCR_BASE, ...parts.map(encodeURIComponent) ].join("/");
const CONTACT_PATH = `${ATTENDANCE_BASE}/professional_profile/contact`;

export function getSncrStock(): Promise<SncrStock> {
  return jsonFetch(sncrPath("stock"));
}

// Vai ao login da Anvisa (gov.br do próprio prescritor); o api guarda o state.
export function requestSncrNumbers(kind: SncrKind, returnTo: string): Promise<SncrAuthorizeRedirect> {
  return jsonFetch(sncrPath("requests"), postProfessional({ kind, return_to: returnTo }));
}

// `answer` é o `session_id` da volta. A recusa também vai ao api, como
// `{ state, error }` (ele consome o state e devolve o return_to), como no 19b (R11).
export function completeSncrOAuth(state: string, answer: string | { error: string }): Promise<SncrCallbackResult> {
  const body = typeof answer === "string" ? { state, session_id: answer } : { state, error: answer.error };
  return jsonFetch(sncrPath("oauth", "callback"), postProfessional(body));
}

export function getSncrOverview(): Promise<SncrOverview> {
  return jsonFetch(sncrPath("admin", "overview"));
}

export function getProfessionalContact(): Promise<ProfessionalContact> {
  return jsonFetch(CONTACT_PATH);
}

export function saveProfessionalContact(contact: { address: SncrAddress; phone: string }): Promise<ProfessionalContact> {
  return jsonFetch(CONTACT_PATH, { method: "PUT", body: JSON.stringify(contact) });
}

export function recordNotification(consultationId: string, content: NotificationRecordInput): Promise<ClinicalDocument> {
  return jsonFetch(attendanceDocPath("consultations", consultationId, "documents"),
    postProfessional({ kind: "controlled_notification_record", content }));
}

// "Reaproveitar o último" (spec §5; plano do api, D4): a identificação da
// última receita do paciente, pela consulta (autora). 404 = nenhuma: a tela diz.
export async function getLastPatientIdentification(consultationId: string): Promise<PatientIdentification | null> {
  try {
    return (await jsonFetch<{ patient_identification: PatientIdentification | null }>(
      attendanceDocPath("consultations", consultationId, "patient_identification"))).patient_identification;
  } catch (err) {
    if (err instanceof ApiError && err.status === 404) return null;
    throw err;
  }
}
```

(`ATTENDANCE_BASE`, `postProfessional` e `attendanceDocPath` são constantes declaradas antes no arquivo.)

Em `src/lib/features.ts`:

```ts
export type FeatureKey =
  | "ledi_export" | "cadsus_lookup" | "clinical_record" | "digital_signature" | "clinical_documents" | "controlled_prescriptions";
```

e, no `FEATURE_LABEL`, depois de `clinical_documents: "Documentos clínicos"`:

```ts
  clinical_documents: "Documentos clínicos",
  controlled_prescriptions: "Receita de controle especial e de antimicrobiano (SNCR)"
```

Em `vite.config.ts`, no comentário do topo, depois da linha de `/clinical_documents` (19c):

```ts
//   /sncr       → números do SNCR (módulo 19d: estoque, pedido pelo gov.br, retorno e painel do admin). O gov.br
//                 devolve a pessoa a /dashboard/sncr/callback (base do Vite), que NÃO casa com este prefixo:
//                 o retorno é do dashboard, e só o POST /sncr/oauth/callback vai ao api.
```

e no `proxy`, depois de `"/clinical_documents": proxy(TARGET),`:

```ts
      "/sncr": proxy(TARGET),
```

No `README.md`, na tabela "## Módulos", logo antes da linha de **Conta**:

```markdown
| Conta → SNCR (módulo 19d) | Saldo de números SNCR por tipo (RCE, RET), pedidos do mês, "Obter números do SNCR" pelo gov.br (retorno em `/dashboard/sncr/callback`) | `health_professional` (médico, dentista) com `controlled_prescriptions` |
| Atendimento → consulta → Documentos (módulo 19d) | Receita de controle especial e de antimicrobiano (categoria, identificação do paciente, limites, aviso de papel, número SNCR) e "Notificação (papel)" | `health_professional` com `controlled_prescriptions` |
| Equipe → Painel do SNCR (módulo 19d) | Saldo de números por prescritor e saldo baixo, só leitura | `municipal_admin` com `controlled_prescriptions` |
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod19d && npx vitest run src/lib/api.controlled.test.ts src/lib/features.test.ts && npx tsc --noEmit`
Expected: PASS e `tsc` sem erro. `short_code` e `verification_url` anuláveis só são lidos em JSX no código do 19c (a coluna "Código" da lista e a frase da declaração na recepção), onde `null` desenha nada; se o `tsc` acusar outro uso (texto concatenado, por exemplo), troque por `?? "—"` naquele ponto.

- [ ] **Step 5: Commit**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/dashboard/.claude/mod19d
/opt/homebrew/bin/git add src/lib/api.ts src/lib/api.controlled.test.ts src/test/controlledFixtures.ts \
  src/lib/features.ts src/lib/features.test.ts vite.config.ts README.md
/opt/homebrew/bin/git commit -m "feat: add the SNCR and controlled prescription API client, feature key and dev proxy

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Regras de tela do 19d e frases novas nas do 19c

**Files:**
- Create: `src/lib/controlled.ts`
- Modify: `src/lib/clinicalDocuments.ts`, `src/lib/clinicalDocuments.test.ts`
- Test: `src/lib/controlled.test.ts`

**Interfaces:**
- Consumes: os tipos da Task 1; de `clinicalDocuments.ts` (19c): `issuedNotice`, `kindLabel`, `professionalKind`, `documentError`, `ItemDraft`, `ItemMedication`, `PrescriptionDraft`, `DOCUMENTS_DISABLED`; `featureDisabledKey`, `hasFeature`, `sessionFeatures` (`features.ts`); `describeActionError` (`actionErrors.ts`); `fmtNumber` (`format.ts`); `onlyDigits` (`attendance.ts`); `maskCep` (`unitAddress.ts`); `Tone` (`theme/tokens.ts`).
- Produces:
  - em `clinicalDocuments.ts`: `CONTROLLED_DISABLED`; `kindLabel("controlled_notification_record")` → "Notificação de receita (papel)"; `PrescriptionDraft.identification?: PatientIdentification | null` (preenchido por `reissueDraft` a partir do documento cancelado); `prescriptionProblems(d, nurse, controlledAllowed = false)` (com `true`, não acusa controlado — quem decide é `controlledProblems`); `documentError` com as recusas do 19d;
  - em `controlled.ts` (todas puras):
    ```ts
    export const SNCR_STOCK_KEY = "sncrStock";            // [ SNCR_STOCK_KEY, userId ]
    export const SNCR_OVERVIEW_KEY = "sncrOverview";
    export const CONTACT_KEY = "professionalContact";     // [ CONTACT_KEY, userId ]
    export const SNCR_RETURN_TO = "/sncr";
    export const MONTHLY_REQUEST_LIMIT = 3; SPECIAL_CONTROL_MAX_C1 = 3; SPECIAL_CONTROL_MAX_DAYS = 60; ANTICONVULSANT_MAX_DAYS = 180;
    export const NOTIFICATION_MAX_DAYS: Record<NotificationType, number>;
    export const SNCR_SIMULATED_NOTICE, SNCR_NO_CANCEL_NOTE, NOTIFICATION_NOTE, MIXED_CATEGORIES, PRESCRIBER_MISSING: string;
    export const CATEGORY_NOTE: Record<"special_control" | "antimicrobial", string>;
    export const UFS: string[];
    export function controlledOn(user): boolean; canRequestSncr(user): boolean; canSeeSncrOverview(user): boolean;
    export function usesSimulatedSncr(user): boolean; prescribesControlled(cbo): boolean;
    export type ItemClass = "common" | "special_control" | "antimicrobial" | "notification" | "unsupported" | "unclassified";
    export function itemClass(m): ItemClass; prescriptionCategory(items: ItemDraft[]): PrescriptionCategory | "mixed";
    export function controlledBlockReason(m): string | null; controlledTagLabel(m): string | null; maxDaysFor(m): number;
    export function categoryLabel(c: string): string; documentCategory(doc): PrescriptionCategory | null; documentSncr(doc): SncrNumberRef | null;
    export function documentTitle(doc): string; sncrLabel(s): string;
    export interface AddressDraft { street; number; complement; district; city; uf; zip }   // strings
    export const EMPTY_ADDRESS_DRAFT; addressDraftFrom(a): AddressDraft; addressProblem(d, who): string | null; addressInput(d): SncrAddress; addressLine(a): string;
    export interface IdentificationDraft { address: AddressDraft }
    export const EMPTY_IDENTIFICATION; identificationDraftFrom(pi): IdentificationDraft; identificationProblem(d): string | null; identificationInput(d): PatientIdentificationInput;
    export function phoneProblem(text): string | null; formatPhone(digits): string;
    export function controlledProblems(d: { items: ItemDraft[] }, identification: IdentificationDraft): string[];
    export function sncrKindFor(category): SncrKind | null;
    export interface ForecastInput { category; cboCode; signatureOn; certificate: SignerCertificate | null | undefined; stock: SncrStock | undefined }
    export function paperForecast(i: ForecastInput): PaperReason | null; paperReasonLabel(r): string;
    export function controlledIssuedNotice(doc: ClinicalDocument): string;
    export interface StockRow { kind; label; free; used; voided; requests; limitReached; notice: { tone: Tone; text: string } | null }
    export function stockRows(s: SncrStock): StockRow[];
    export interface SncrCallbackParams { state: string | null; sessionId: string | null; error: string | null }   // state só se a URL trouxer
    export function readSncrCallback(pathname?, search?, base?): SncrCallbackParams | null;   // pura: só lê a URL
    export const SNCR_STATE_STORAGE_KEY = "rotasaude.sncr.state";
    export function rememberSncrState(state: string, storage?: Storage | null): void; takeSncrState(storage?: Storage | null): string | null;
    export function sncrCallbackSummary(r: SncrCallbackResult): string; sncrCallbackLanding(r): string; sncrOauthErrorPhrase(error): string;
    export function notificationTypeFor(list): NotificationType | null; NOTIFICATION_TYPE_LABEL;
    export interface NotificationDraft { type: NotificationType | ""; paperNumber; numberingUf; medication: ItemMedication | null; quantity; quantityUnit; route: MedicationRoute | ""; dosage; durationDays }
    export function emptyNotification(uf: string): NotificationDraft; notificationProblems(d): string[]; notificationInput(d): NotificationRecordInput;
    export function sortOverview(list): SncrOverviewProfessional[]; overviewSummary(o: SncrOverview): { professionals: number; low: number; neverRequested: number };
    export function controlledError(err: unknown): string;
    ```

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/controlled.test.ts
import { afterEach, describe, expect, it, vi } from "vitest";
import { ApiError } from "./api";
import { itemFromMedication, issuedNotice, type ItemDraft } from "./clinicalDocuments";
import {
  CATEGORY_NOTE, CONTROLLED_DISABLED_TEXT, EMPTY_IDENTIFICATION, MIXED_CATEGORIES, SNCR_SIMULATED_NOTICE, addressDraftFrom, addressInput,
  addressLine, addressProblem, canRequestSncr, canSeeSncrOverview, controlledBlockReason, controlledError, controlledIssuedNotice,
  controlledOn, controlledProblems, controlledTagLabel, documentTitle, emptyNotification, formatPhone, identificationDraftFrom,
  identificationInput, notificationInput, notificationProblems, notificationTypeFor, overviewSummary, paperForecast, paperReasonLabel,
  phoneProblem, prescribesControlled, prescriptionCategory, readSncrCallback, rememberSncrState, sncrCallbackLanding, sncrCallbackSummary, sncrLabel,
  sncrOauthErrorPhrase, sortOverview, stockRows, takeSncrState, usesSimulatedSncr, type IdentificationDraft
} from "./controlled";
import { DIPYRONE, prescriptionDoc } from "../test/documentFixtures";
import {
  AMITRIPTYLINE, AMOXICILLIN_19D, CARBAMAZEPINE, CLONAZEPAM_B1, FLUOXETINE, ISOTRETINOIN, MORPHINE, SERTRALINE, address, controlledUser,
  notificationDoc, rceDoc, retDoc, sncrOverview, stock
} from "../test/controlledFixtures";
import { certificate } from "../test/signatureFixtures";

afterEach(() => vi.useRealTimers());

const err = (status: number, body: unknown) => new ApiError(status, body, `${status}`);
const item = (m: Parameters<typeof itemFromMedication>[0], over: Partial<ItemDraft> = {}): ItemDraft =>
  ({ ...itemFromMedication(m), quantity: "30", route: "oral", dosage: "1 comprimido ao dia", durationDays: "30", ...over });
const FILLED: IdentificationDraft = { address: addressDraftFrom(address()) };

describe("quem pode", () => {
  it("receita de controlado: cidade com documentos e controlado, nunca operador", () => {
    expect(controlledOn(controlledUser())).toBe(true);
    expect(controlledOn(controlledUser([ "health_professional" ], { features: [ "clinical_documents" ] }))).toBe(false);
    expect(controlledOn(controlledUser([ "health_professional" ], { features: [ "controlled_prescriptions" ] }))).toBe(false);
    expect(controlledOn(controlledUser([ "health_professional" ], { operator: true }))).toBe(false);
    expect(controlledOn(null)).toBe(false);
  });

  it("SNCR do profissional; painel do admin; simulado com sncr_mock", () => {
    expect(canRequestSncr(controlledUser())).toBe(true);
    expect(canRequestSncr(controlledUser([ "municipal_admin" ]))).toBe(false);
    expect(canSeeSncrOverview(controlledUser([ "municipal_admin" ]))).toBe(true);
    expect(canSeeSncrOverview(controlledUser())).toBe(false);
    expect(usesSimulatedSncr(controlledUser())).toBe(false);
    expect(usesSimulatedSncr(controlledUser([ "health_professional" ], {
      features: [ "clinical_documents", "controlled_prescriptions", "sncr_mock" ] }))).toBe(true);
    expect(usesSimulatedSncr(controlledUser([ "health_professional" ], { features: [ "sncr_mock" ] }))).toBe(false);
  });

  it("controlado e Notificação só médico e dentista", () => {
    expect(prescribesControlled("225142")).toBe(true);
    expect(prescribesControlled("223208")).toBe(true);
    expect(prescribesControlled("223565")).toBe(false);
    expect(prescribesControlled(null)).toBe(false);
  });
});

describe("categoria pelos itens", () => {
  it("comum, controle especial, antimicrobiano e misturada", () => {
    expect(prescriptionCategory([])).toBe("common");
    expect(prescriptionCategory([ item(DIPYRONE) ])).toBe("common");
    expect(prescriptionCategory([ item(SERTRALINE), item(DIPYRONE) ])).toBe("special_control");
    expect(prescriptionCategory([ item(AMOXICILLIN_19D), item(DIPYRONE) ])).toBe("antimicrobial");
    expect(prescriptionCategory([ item(SERTRALINE), item(AMOXICILLIN_19D) ])).toBe("mixed");
  });

  it("A/B bloqueados com o caminho da Notificação; C2/C3/C4 fora; sem lista, bloqueado; C1 entra", () => {
    expect(controlledBlockReason(MORPHINE)).toBe("Notificação de receita (listas A/B) — registre a de papel em “Notificação (papel)”");
    expect(controlledBlockReason(CLONAZEPAM_B1)).toBe("Notificação de receita (listas A/B) — registre a de papel em “Notificação (papel)”");
    expect(controlledBlockReason(ISOTRETINOIN)).toBe("lista C2 (retinoide, talidomida ou antirretroviral) — fora do Rota Saúde");
    expect(controlledBlockReason({ ...ISOTRETINOIN, controlled_list: "C4" })).toBe("lista C4 (retinoide, talidomida ou antirretroviral) — fora do Rota Saúde");
    expect(controlledBlockReason({ ...SERTRALINE, controlled_list: null }))
      .toBe("controlado sem a lista da Portaria 344 no catálogo — não entra na receita");
    expect(controlledBlockReason(SERTRALINE)).toBeNull();
    expect(controlledBlockReason(AMOXICILLIN_19D)).toBeNull();
    expect(controlledTagLabel(SERTRALINE)).toBe("lista C1");
    expect(controlledTagLabel(CARBAMAZEPINE)).toBe("lista C1 · anticonvulsivante");
    expect(controlledTagLabel(DIPYRONE)).toBeNull();
  });
});

describe("limites da receita de controle especial", () => {
  it("misturada diz para separar; antimicrobiano e comum não têm limite aqui", () => {
    expect(controlledProblems({ items: [ item(SERTRALINE, { durationDays: "60" }), item(AMOXICILLIN_19D) ] }, FILLED)).toEqual([ MIXED_CATEGORIES ]);
    expect(controlledProblems({ items: [ item(AMOXICILLIN_19D, { durationDays: "" }) ] }, EMPTY_IDENTIFICATION)).toEqual([]);
    expect(controlledProblems({ items: [ item(DIPYRONE) ] }, EMPTY_IDENTIFICATION)).toEqual([]);
  });

  it("até 3 substâncias C1 diferentes", () => {
    const four = [ SERTRALINE, AMITRIPTYLINE, FLUOXETINE, CARBAMAZEPINE ].map((m) => item(m));
    expect(controlledProblems({ items: four }, FILLED)).toEqual([ "no máximo 3 substâncias da lista C1 por receita" ]);
    expect(controlledProblems({ items: four.slice(0, 3) }, FILLED)).toEqual([]);
    // A mesma substância em duas apresentações conta uma vez.
    expect(controlledProblems({ items: [ item(SERTRALINE), item({ ...SERTRALINE, id: "ci99" }), item(FLUOXETINE), item(AMITRIPTYLINE) ] }, FILLED))
      .toEqual([]);
  });

  it("duração obrigatória e até 60 dias; 180 no anticonvulsivante", () => {
    expect(controlledProblems({ items: [ item(SERTRALINE, { durationDays: "" }) ] }, FILLED))
      .toEqual([ "item 1: informe a duração em dias (até 60)" ]);
    expect(controlledProblems({ items: [ item(SERTRALINE, { durationDays: "90" }) ] }, FILLED))
      .toEqual([ "item 1: duração acima de 60 dias" ]);
    expect(controlledProblems({ items: [ item(DIPYRONE), item(CARBAMAZEPINE, { durationDays: "120" }) ] }, FILLED)).toEqual([]);
    expect(controlledProblems({ items: [ item(DIPYRONE), item(CARBAMAZEPINE, { durationDays: "200" }) ] }, FILLED))
      .toEqual([ "item 2: duração acima de 180 dias (anticonvulsivante)" ]);
    // Na receita de controle especial, todo item leva duração (plano do api, D15), até o comum.
    expect(controlledProblems({ items: [ item(SERTRALINE), item(DIPYRONE, { durationDays: "" }) ] }, FILLED))
      .toEqual([ "item 2: informe a duração em dias (até 60)" ]);
  });

  it("item bloqueado que chegou pela renovação aparece com o motivo, em qualquer categoria", () => {
    expect(controlledProblems({ items: [ item(MORPHINE) ] }, EMPTY_IDENTIFICATION))
      .toEqual([ "item 1: Notificação de receita (listas A/B) — registre a de papel em “Notificação (papel)”" ]);
  });

  it("controle especial pede o endereço do paciente", () => {
    expect(controlledProblems({ items: [ item(SERTRALINE) ] }, EMPTY_IDENTIFICATION))
      .toEqual([ "endereço do paciente: informe o logradouro" ]);
  });
});

describe("endereço, identificação e telefone", () => {
  it("rascunho ↔ corpo: CEP só dígitos, complemento ausente quando vazio, UF maiúscula", () => {
    const draft = addressDraftFrom(address());
    expect(draft.zip).toBe("80010-010");
    expect(addressInput({ ...draft, complement: "  " })).toEqual({
      street: "Rua Marechal Deodoro", number: "630", district: "Centro", city: "Curitiba", uf: "PR", zip: "80010010"
    });
    expect(addressInput(draft).complement).toBe("apto 12");
    expect(addressDraftFrom(null).street).toBe("");
  });

  it("o que falta no endereço, um de cada vez", () => {
    const ok = addressDraftFrom(address());
    expect(addressProblem(ok, "endereço")).toBeNull();
    expect(addressProblem({ ...ok, number: "" }, "endereço")).toBe("endereço: informe o número (ou s/n)");
    expect(addressProblem({ ...ok, district: " " }, "endereço")).toBe("endereço: informe o bairro");
    expect(addressProblem({ ...ok, uf: "" }, "endereço")).toBe("endereço: escolha a UF");
    expect(addressProblem({ ...ok, zip: "8001-001" }, "endereço")).toBe("endereço: o CEP precisa ter 8 dígitos");
  });

  it("linha do endereço para a tela", () => {
    expect(addressLine(address())).toBe("Rua Marechal Deodoro, 630 — apto 12 · Centro · Curitiba/PR · 80010-010");
    expect(addressLine(null)).toBe("—");
  });

  it("identificação: só o endereço no corpo (o CPF é o do cadastro, no api)", () => {
    expect(identificationInput(FILLED)).toEqual({ address: addressInput(FILLED.address) });
    expect(identificationInput(FILLED)).not.toHaveProperty("cpf");
    expect(identificationDraftFrom({ cpf: "39053344705", address: address() })).toEqual({ address: addressDraftFrom(address()) });
    expect(identificationDraftFrom(null)).toEqual(EMPTY_IDENTIFICATION);
  });

  it("telefone: DDD e número, 10 ou 11 dígitos", () => {
    expect(phoneProblem("(41) 3350-1234")).toBeNull();
    expect(phoneProblem("41 99876-5432")).toBeNull();
    expect(phoneProblem("3350-1234")).toBe("telefone: DDD e número, 10 ou 11 dígitos");
    expect(phoneProblem("(09) 3350-1234")).toBe("telefone: DDD inválido");
    expect(formatPhone("4133501234")).toBe("(41) 3350-1234");
    expect(formatPhone("41998765432")).toBe("(41) 99876-5432");
    expect(formatPhone(null)).toBe("—");
  });
});

describe("previsão de papel (o api decide)", () => {
  const base = { category: "special_control" as const, cboCode: "225142", signatureOn: true, certificate: certificate(), stock: stock() };

  it("digital quando tudo existe; comum e misturada não preveem", () => {
    expect(paperForecast(base)).toBeNull();
    expect(paperForecast({ ...base, category: "common" })).toBeNull();
    expect(paperForecast({ ...base, category: "mixed" })).toBeNull();
  });

  it("os motivos, na ordem do api", () => {
    expect(paperForecast({ ...base, category: "antimicrobial", cboCode: "223565" })).toBe("nurse_antimicrobial");
    expect(paperForecast({ ...base, signatureOn: false })).toBe("signature_unavailable");
    expect(paperForecast({ ...base, certificate: null })).toBe("no_certificate");
    expect(paperForecast({ ...base, certificate: certificate({ status: "expired" }) })).toBe("no_certificate");
    expect(paperForecast({ ...base, stock: stock({ rce: { free: 0, used: 1000, voided: 0, requests_this_month: 1 } }) })).toBe("no_sncr_number");
    expect(paperForecast({ ...base, category: "antimicrobial", stock: stock({ ret: { free: 0, used: 0, voided: 0, requests_this_month: 0 } }) }))
      .toBe("no_sncr_number");
  });

  it("enquanto carrega (certificado ou estoque indefinidos), não prevê papel", () => {
    expect(paperForecast({ ...base, certificate: undefined, stock: undefined })).toBeNull();
  });

  it("frase de cada motivo; desconhecido cru", () => {
    expect(paperReasonLabel("no_sncr_number")).toBe("sem número SNCR no seu estoque — obtenha números em Conta → SNCR");
    expect(paperReasonLabel("no_certificate")).toBe("você não tem certificado digital ativo — vincule em Conta → Assinatura digital");
    expect(paperReasonLabel("vigilance_rule")).toBe("vigilance_rule");
    expect(paperReasonLabel(null)).toBe("—");
  });
});

describe("documento emitido", () => {
  it("RCE digital com número (e simulado)", () => {
    expect(controlledIssuedNotice(rceDoc())).toBe(
      "Receita de controle especial emitida em modo digital com o número SNCR RCE nº 2610.1-41.0001234: a assinatura entra na sua fila; o PDF assinado sai depois dela.");
    expect(controlledIssuedNotice(rceDoc({}, { sncr: { kind: "rce", number: "2610.1-41.0001234", simulated: true } }))).toBe(
      "Receita de controle especial emitida em modo digital com o número SNCR RCE nº 2610.1-41.0001234 (numeração simulada — sem validade): a assinatura entra na sua fila; o PDF assinado sai depois dela.");
  });

  it("papel com o motivo do api; comum como no 19c; Notificação registrada", () => {
    expect(controlledIssuedNotice(retDoc({ issue_mode: "paper", paper_reason: "no_certificate", signature: null }))).toBe(
      "Receita de antimicrobiano emitida em modo papel (você não tem certificado digital ativo — vincule em Conta → Assinatura digital): imprima as 2 vias e assine à mão.");
    expect(controlledIssuedNotice(prescriptionDoc())).toBe(issuedNotice(prescriptionDoc()));
    expect(controlledIssuedNotice(notificationDoc())).toBe(
      "Notificação de papel registrada: o medicamento entra na lista de medicamentos em uso. Guarde o talão; nada é impresso aqui.");
  });

  it("título e número na lista", () => {
    expect(documentTitle(rceDoc())).toBe("Receita de controle especial");
    expect(documentTitle(retDoc())).toBe("Receita de antimicrobiano");
    expect(documentTitle(prescriptionDoc())).toBe("Receita");
    expect(documentTitle(notificationDoc())).toBe("Notificação de receita (papel)");
    expect(sncrLabel({ kind: "ret", number: "2610.2-41.0000501" })).toBe("RET nº 2610.2-41.0000501");
    expect(SNCR_SIMULATED_NOTICE).toBe("Numeração simulada — sem validade");
    expect(CATEGORY_NOTE.special_control).toMatch(/válida por 30 dias/);
    expect(CATEGORY_NOTE.antimicrobial).toMatch(/válida por 10 dias/);
  });
});

describe("estoque", () => {
  it("saldo baixo, zerado e limite do mês", () => {
    const [ rce, ret ] = stockRows(stock());
    expect(rce).toMatchObject({ kind: "rce", label: "Receita de controle especial (RCE)", free: 812, requests: 1, limitReached: false, notice: null });
    expect(ret.notice).toEqual({ tone: "warn", text: "saldo baixo: menos de 50 números — obtenha mais antes que acabe" });
    const [ empty ] = stockRows(stock({ rce: { free: 0, used: 2000, voided: 3, requests_this_month: 3 } }));
    expect(empty.notice).toEqual({ tone: "down", text: "sem números: as receitas de controle especial saem em papel" });
    expect(empty.limitReached).toBe(true);
  });
});

describe("retorno do gov.br", () => {
  it("lê session_id, error (e o state, se vier) só no caminho de retorno, sob a base", () => {
    expect(readSncrCallback("/dashboard/sncr/callback", "?session_id=sid1", "/dashboard/")).toEqual({ state: null, sessionId: "sid1", error: null });
    expect(readSncrCallback("/dashboard/sncr/callback/", "?state=s1&session_id=sid1", "/dashboard/"))
      .toEqual({ state: "s1", sessionId: "sid1", error: null });
    expect(readSncrCallback("/dashboard/sncr/callback", "?error=access_denied", "/dashboard/")).toEqual({ state: null, sessionId: null, error: "access_denied" });
    expect(readSncrCallback("/dashboard/sncr/callback", "?session_id=%20", "/dashboard/")).toEqual({ state: null, sessionId: null, error: null });
    expect(readSncrCallback("/dashboard/signature/callback", "?session_id=sid1", "/dashboard/")).toBeNull();
    expect(readSncrCallback("/dashboard/", "?session_id=sid1", "/dashboard/")).toBeNull();
  });

  it("o state fica no sessionStorage só entre a ida e a volta, e sai na primeira leitura", () => {
    window.sessionStorage.clear();
    rememberSncrState("st1");
    expect(window.sessionStorage.getItem("rotasaude.sncr.state")).toBe("st1");
    expect(takeSncrState()).toBe("st1");
    expect(window.sessionStorage.getItem("rotasaude.sncr.state")).toBeNull();
    expect(takeSncrState()).toBeNull();
    // Storage indisponível (aba privada, política): não quebra.
    const broken = { getItem: () => { throw new Error("blocked"); }, setItem: () => { throw new Error("blocked"); }, removeItem: () => undefined } as unknown as Storage;
    expect(() => rememberSncrState("st1", broken)).not.toThrow();
    expect(takeSncrState(broken)).toBeNull();
  });

  it("resumo, destino e recusa do gov.br", () => {
    expect(sncrCallbackSummary({ kind: "rce", received: 1000, batch_id: "b1" }))
      .toBe("1.000 números recebidos do SNCR para receita de controle especial (RCE).");
    expect(sncrCallbackSummary({ kind: "ret", received: 1, batch_id: "b1" }))
      .toBe("1 número recebido do SNCR para receita de antimicrobiano (RET).");
    expect(sncrCallbackLanding({ return_to: "/sncr" })).toBe("/sncr");
    expect(sncrCallbackLanding({ return_to: "//evil.example" })).toBe("/sncr");
    expect(sncrCallbackLanding(undefined)).toBe("/sncr");
    expect(sncrOauthErrorPhrase("access_denied")).toBe("a autorização foi negada no gov.br — nenhum número foi pedido");
    expect(sncrOauthErrorPhrase(null)).toBe("o gov.br não concluiu a autorização — comece de novo");
  });
});

describe("Notificação de papel", () => {
  it("tipo pela lista do item", () => {
    expect(notificationTypeFor("A1")).toBe("A");
    expect(notificationTypeFor("A3")).toBe("A");
    expect(notificationTypeFor("B1")).toBe("B");
    expect(notificationTypeFor("B2")).toBe("B2");
    expect(notificationTypeFor("C1")).toBeNull();
    expect(notificationTypeFor(null)).toBeNull();
  });

  const filled = () => ({ ...emptyNotification("PR"), type: "B" as const, paperNumber: "PR0012345", medication: CLONAZEPAM_B1,
    quantity: "30", quantityUnit: "comprimido", route: "oral" as const, dosage: "1 comprimido à noite", durationDays: "30" });

  it("o que falta, tipo que não bate e duração por tipo", () => {
    expect(notificationProblems(filled())).toEqual([]);
    expect(notificationProblems(emptyNotification("PR"))).toEqual([
      "escolha o tipo da Notificação", "informe o número impresso no talão", "escolha o medicamento (listas A ou B)",
      "informe a quantidade (número inteiro)", "escolha a via", "escreva a posologia", "informe a duração em dias"
    ]);
    expect(notificationProblems({ ...filled(), type: "A" })).toEqual([ "o medicamento é da lista B1: a Notificação é B" ]);
    expect(notificationProblems({ ...filled(), medication: SERTRALINE })).toEqual([ "o medicamento escolhido não é de Notificação (listas A ou B)" ]);
    expect(notificationProblems({ ...filled(), type: "A", medication: MORPHINE, durationDays: "45" }))
      .toEqual([ "duração acima de 30 dias para a Notificação A" ]);
    expect(notificationProblems({ ...filled(), durationDays: "60" })).toEqual([]);
    expect(notificationProblems({ ...filled(), numberingUf: "" })).toEqual([ "escolha a UF da numeração" ]);
  });

  it("corpo do POST", () => {
    expect(notificationInput(filled())).toEqual({
      notification_type: "B", paper_number: "PR0012345", numbering_uf: "PR",
      item: { catalog_item: { id: "ci3" }, quantity: 30, quantity_unit: "comprimido", route: "oral",
        dosage_instructions: "1 comprimido à noite", duration_days: 30 }
    });
  });
});

describe("painel do admin", () => {
  it("saldo baixo primeiro, depois por nome; contagens", () => {
    expect(sortOverview(sncrOverview().professionals).map((p) => p.user_id)).toEqual([ "us3", "us1", "us2" ]);
    expect(overviewSummary(sncrOverview())).toEqual({ professionals: 3, low: 2, neverRequested: 1 });
  });
});

describe("frases das recusas do SNCR", () => {
  it("cada código do contrato; interruptor; desconhecido genérico", () => {
    expect(controlledError(err(409, { error: "monthly_limit_reached" })))
      .toBe("você já fez os 3 pedidos deste mês para este tipo — o limite volta no mês que vem");
    expect(controlledError(err(409, { error: "authorization_expired" })))
      .toBe("a autorização demorou demais e venceu — comece de novo; o pedido ao SNCR precisa sair em até 30 segundos");
    expect(controlledError(err(503, { error: "sncr_exhausted" })))
      .toBe("o SNCR está sem números para a sua UF agora — tente mais tarde; até lá, as receitas saem em papel");
    expect(controlledError(err(503, { error: "sncr_unavailable" }))).toBe("o SNCR não respondeu — tente de novo em alguns minutos; nenhum número foi gasto");
    expect(controlledError(err(403, { error: "cbo_not_allowed" }))).toBe("só médicos e dentistas pedem números do SNCR e prescrevem controlado");
    expect(controlledError(err(422, { error: "invalid_address", field: "address.zip" }))).toBe("endereço inválido: confira o CEP");
    expect(controlledError(err(422, { error: "invalid_phone" }))).toBe("telefone inválido — DDD e número, 10 ou 11 dígitos");
    expect(controlledError(err(403, { error: "registration_mismatch" })))
      .toBe("a conta gov.br usada não bate com a sua inscrição no conselho, ou você não tem cadastro de prescritor no SNCR — entre com a sua própria conta gov.br");
    expect(controlledError(err(403, { error: "feature_disabled", feature: "controlled_prescriptions" }))).toBe(CONTROLLED_DISABLED_TEXT);
    expect(controlledError(err(500, ""))).not.toMatch(/undefined/);
  });
});
```

Em `src/lib/clinicalDocuments.test.ts`, no fim do arquivo:

```ts
describe("frases e regras do 19d nas regras do 19c", () => {
  it("rótulo da Notificação e o interruptor do controlado", () => {
    expect(kindLabel("controlled_notification_record")).toBe("Notificação de receita (papel)");
    expect(documentError(err(403, { error: "feature_disabled", feature: "controlled_prescriptions" })))
      .toBe("a receita de controlado está desligada nesta cidade");
    expect(documentError(err(403, { error: "feature_disabled", feature: "clinical_documents" }))).toBe(DOCUMENTS_DISABLED);
  });

  it("recusas novas da emissão, com a posição do item quando vem", () => {
    expect(documentError(err(422, { error: "mixed_categories" })))
      .toBe("não misture controle especial e antimicrobiano na mesma receita — emita uma receita para cada");
    expect(documentError(err(422, { error: "duration_exceeded", index: 1 })))
      .toBe("item 2: duração acima do permitido (controle especial: 60 dias, 180 para anticonvulsivante; Notificação A: 30; B: 60)");
    expect(documentError(err(422, { error: "requires_notification", index: 0 })))
      .toBe("item 1: este medicamento é de Notificação (listas A/B): registre a Notificação de papel em vez da receita");
    expect(documentError(err(422, { error: "too_many_c1_substances" }))).toBe("no máximo 3 substâncias da lista C1 por receita");
    expect(documentError(err(422, { error: "prescriber_address_missing" })))
      .toBe("falta o seu endereço e telefone — cadastre em Meu perfil antes de emitir");
    expect(documentError(err(422, { error: "patient_identification_required", field: "address.zip" })))
      .toBe("falta na identificação do paciente: CEP");
    expect(documentError(err(422, { error: "notification_type_mismatch" }))).toBe("o tipo da Notificação não bate com a lista do medicamento");
    expect(documentError(err(422, { error: "invalid_content", field: "paper_number" }))).toBe("confira o campo: número do talão");
  });

  it("com o controlado liberado, a receita não acusa o C1 (quem confere é controlledProblems)", () => {
    const sertraline = { ...itemFromMedication({ ...catalogItem({ id: "ci10" }), controlled: true, controlled_list: "C1" }),
      quantity: "30", route: "oral" as const, dosage: "1 cp", durationDays: "60" };
    expect(prescriptionProblems({ items: [ sertraline ], protocolVersionId: "" }, false))
      .toEqual([ `item 1: ${CONTROLLED_BLOCKED}` ]);
    expect(prescriptionProblems({ items: [ sertraline ], protocolVersionId: "" }, false, true)).toEqual([]);
  });

  it("cancelar e emitir outro leva a identificação do paciente da receita cancelada", () => {
    const pi = { cpf: "39053344705", address: { street: "Rua A", number: "1", district: "Centro", city: "Curitiba", uf: "PR", zip: "80010010" } };
    const doc = prescriptionDoc({ content: { items: [ prescriptionItem() ], antimicrobial: false, copies: 2, category: "special_control", patient_identification: pi } });
    const next = reissueDraft(doc);
    expect(next?.kind).toBe("prescription");
    expect(next?.kind === "prescription" ? next.draft.identification : undefined).toEqual(pi);
  });
});
```

(Se `err`, `catalogItem`, `prescriptionItem`, `prescriptionDoc`, `CONTROLLED_BLOCKED`, `DOCUMENTS_DISABLED`, `itemFromMedication`, `kindLabel`, `prescriptionProblems`, `reissueDraft` ou `documentError` ainda não estiverem importados no topo do teste do 19c, acrescente-os nos imports existentes.)

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod19d && npx vitest run src/lib/controlled.test.ts src/lib/clinicalDocuments.test.ts`
Expected: FAIL — `./controlled` não existe; `kindLabel("controlled_notification_record")` devolve a chave crua; as frases novas saem genéricas; `prescriptionProblems` não aceita o terceiro argumento.

- [ ] **Step 3: Write minimal implementation**

Em `src/lib/clinicalDocuments.ts`:

(a) nos imports de `./api`, acrescente `type PatientIdentification`;

(b) depois de `export const NO_PROTOCOL = …`:

```ts
// Módulo 19d: interruptor `controlled_prescriptions` desligado.
export const CONTROLLED_DISABLED = "a receita de controlado está desligada nesta cidade";
```

(c) no `KIND_LABEL`:

```ts
const KIND_LABEL: Record<string, string> = {
  sick_note: "Atestado", attendance_declaration: "Declaração de comparecimento", prescription: "Receita",
  exam_requisition: "Requisição de exames", controlled_notification_record: "Notificação de receita (papel)"
};
```

(d) `PrescriptionDraft`:

```ts
// `identification`: módulo 19d — a do documento cancelado em "cancelar e emitir outro" (a tela converte).
export interface PrescriptionDraft { items: ItemDraft[]; protocolVersionId: string; identification?: PatientIdentification | null }
```

(e) em `prescriptionProblems`, a assinatura e a linha do controlado:

```ts
export function prescriptionProblems(d: PrescriptionDraft, nurse: boolean, controlledAllowed = false): string[] {
```

```ts
    if (it.medication?.controlled && !controlledAllowed) out.push(`${n}: ${CONTROLLED_BLOCKED}`);
```

(f) em `reissueDraft`, o `case "prescription"`:

```ts
    case "prescription": {
      const c = doc.content as PrescriptionContent;
      return { kind: "prescription", draft: {
        items: c.items.map(itemFromPrescription), protocolVersionId: c.nursing_protocol?.version_id ?? "",
        identification: c.patient_identification ?? null
      } };
    }
```

(g) no `FIELD_LABEL`, depois de `note: "observação"`:

```ts
  note: "observação", paper_number: "número do talão", numbering_uf: "UF da numeração", notification_type: "tipo da Notificação",
  item: "medicamento", duration_days: "duração em dias"
```

(h) no `ERRORS`, depois de `missing_role`:

```ts
  missing_role: "seu papel não permite esta ação",
  // Módulo 19d (contrato §2–§3).
  mixed_categories: "não misture controle especial e antimicrobiano na mesma receita — emita uma receita para cada",
  requires_notification: "este medicamento é de Notificação (listas A/B): registre a Notificação de papel em vez da receita",
  not_supported: "medicamento das listas C2, C3 ou C4 (retinoide, talidomida, antirretroviral) não sai pelo Rota Saúde",
  too_many_c1_substances: "no máximo 3 substâncias da lista C1 por receita",
  duration_exceeded: "duração acima do permitido (controle especial: 60 dias, 180 para anticonvulsivante; Notificação A: 30; B: 60)",
  prescriber_address_missing: "falta o seu endereço e telefone — cadastre em Meu perfil antes de emitir",
  item_not_in_notification_list: "o medicamento não é de Notificação (listas A ou B)",
  notification_type_mismatch: "o tipo da Notificação não bate com a lista do medicamento"
```

(i) em `documentError`, troque as duas primeiras linhas do interruptor e acrescente o caso da identificação, logo depois do bloco de `invalid_content`:

```ts
  const feature = featureDisabledKey(err);
  if (feature === "clinical_record") return "o prontuário está desligado nesta cidade";
  if (feature === "controlled_prescriptions") return CONTROLLED_DISABLED;
  if (feature !== null) return DOCUMENTS_DISABLED;
```

```ts
  if (code === "patient_identification_required") {
    const field = typeof body?.field === "string" ? body.field : "";
    return `falta na identificação do paciente: ${IDENTIFICATION_FIELD[field.split(".").pop() ?? ""] ?? "o endereço"}`;
  }
```

e, antes de `const ERRORS`:

```ts
const IDENTIFICATION_FIELD: Record<string, string> = {
  cpf: "CPF do cadastro do paciente", address: "endereço",
  street: "logradouro", number: "número", district: "bairro", city: "cidade", uf: "UF", zip: "CEP"
};
```

Crie `src/lib/controlled.ts`:

```ts
// src/lib/controlled.ts
// Regras de tela da receita de controlado e de antimicrobiano e do SNCR
// (módulo 19d; spec §4–§7; ADR 0034; contrato 2026-10-10 §1–§7). Quem decide
// é o api (categoria, modo, número, limites, quem prescreve): aqui só se
// espelha para não oferecer o que ele recusaria e se traduz o que ele
// devolveu. Nenhuma frase leva CPF, endereço, número SNCR, state ou session_id —
// só rótulos, posições ("item 2") e o número quando a própria tela o mostra.
import {
  errorCode,
  type ClinicalDocument, type MedicationRoute, type NotificationRecordInput, type NotificationType, type PaperReason,
  type PatientIdentification, type PatientIdentificationInput, type PrescriptionCategory, type PrescriptionContent, type SignerCertificate,
  type SncrAddress, type SncrCallbackResult, type SncrKind, type SncrNumberRef, type SncrOverview, type SncrOverviewProfessional,
  type SncrStock
} from "./api";
import { describeActionError } from "./actionErrors";
import { featureDisabledKey, hasFeature, sessionFeatures } from "./features";
import { fmtNumber } from "./format";
import { onlyDigits } from "./attendance";
import { maskCep } from "./unitAddress";
import { CONTROLLED_DISABLED, issuedNotice, kindLabel, professionalKind, type ItemDraft, type ItemMedication } from "./clinicalDocuments";
import type { Tone } from "../theme/tokens";

export const SNCR_STOCK_KEY = "sncrStock";
export const SNCR_OVERVIEW_KEY = "sncrOverview";
export const CONTACT_KEY = "professionalContact";
// Caminho do módulo "sncr" (return_to; Divergência D3 do 19b).
export const SNCR_RETURN_TO = "/sncr";

export const MONTHLY_REQUEST_LIMIT = 3;
export const SPECIAL_CONTROL_MAX_C1 = 3;
export const SPECIAL_CONTROL_MAX_DAYS = 60;
export const ANTICONVULSANT_MAX_DAYS = 180;
export const NOTIFICATION_MAX_DAYS: Record<NotificationType, number> = { A: 30, B: 60, B2: 60 };

export const CONTROLLED_DISABLED_TEXT = CONTROLLED_DISABLED;
export const SNCR_SIMULATED_NOTICE = "Numeração simulada — sem validade";
export const SNCR_NO_CANCEL_NOTE =
  "O número SNCR fica anulado e não volta ao seu estoque. O SNCR não tem cancelamento: a farmácia só vê que a receita foi cancelada na conferência do Rota Saúde.";
export const NOTIFICATION_NOTE =
  "Registro da Notificação de papel (talão da VISA): não gera PDF nem assinatura e não aparece na página de conferência; o medicamento entra na lista de medicamentos em uso.";
export const MIXED_CATEGORIES = "não misture controle especial e antimicrobiano na mesma receita — emita uma receita para cada";
// Plano do api, D12: na de controle especial, sempre; na de antimicrobiano, quando ela sairia digital.
export const PRESCRIBER_MISSING = "falta o seu endereço e telefone — cadastre em Meu perfil antes de emitir esta receita";
export const CATEGORY_NOTE: Record<"special_control" | "antimicrobial", string> = {
  special_control:
    "receita de controle especial: 2 vias (1ª via — farmácia, 2ª via — paciente), válida por 30 dias; digital com número SNCR quando você tem certificado e número no estoque; senão, papel",
  antimicrobial:
    "receita de antimicrobiano: 2 vias (1ª via — farmácia, 2ª via — paciente), válida por 10 dias; digital com número SNCR quando você tem certificado e número no estoque; senão, papel"
};
export const UFS = [
  "AC", "AL", "AM", "AP", "BA", "CE", "DF", "ES", "GO", "MA", "MG", "MS", "MT", "PA", "PB", "PE", "PI", "PR", "RJ", "RN", "RO", "RR",
  "RS", "SC", "SE", "SP", "TO"
];

const GENERIC = "não foi possível concluir — tente de novo";
const positiveInt = (text: string): number | null => {
  const t = text.trim();
  return /^\d+$/.test(t) && Number(t) >= 1 ? Number(t) : null;
};

// ─── Quem pode ────────────────────────────────────────────────────────────────

type SessionLike = { operator?: boolean; memberships?: { role: string }[]; features?: unknown } | null | undefined;
const rolesOf = (user: SessionLike) => user?.memberships?.map((m) => m.role) ?? [];

export function controlledOn(user: SessionLike): boolean {
  return !!user && !user.operator && hasFeature(user, "clinical_documents") && hasFeature(user, "controlled_prescriptions");
}

export function canRequestSncr(user: SessionLike): boolean {
  return controlledOn(user) && rolesOf(user).includes("health_professional");
}

export function canSeeSncrOverview(user: SessionLike): boolean {
  return controlledOn(user) && rolesOf(user).includes("municipal_admin");
}

// Interruptor `sncr_mock` (maintenance) ligado sobre o controlado.
export function usesSimulatedSncr(user: SessionLike): boolean {
  const features = sessionFeatures(user);
  return features.includes("controlled_prescriptions") && features.includes("sncr_mock");
}

export function prescribesControlled(cbo: string | null | undefined): boolean {
  const who = professionalKind(cbo);
  return who === "physician" || who === "dentist";
}

// ─── Categoria e bloqueio por lista (Portaria 344) ────────────────────────────

export type ItemClass = "common" | "special_control" | "antimicrobial" | "notification" | "unsupported" | "unclassified";
type Classifiable = { antimicrobial?: boolean; controlled?: boolean; controlled_list?: string | null; anticonvulsant?: boolean };

const NOTIFICATION_LISTS = [ "A1", "A2", "A3", "B1", "B2" ];
// C4 (antirretrovirais) como C2/C3: plano do api, D8.
const UNSUPPORTED_LISTS = [ "C2", "C3", "C4" ];
const SPECIAL_LISTS = [ "C1", "C5" ];

export function itemClass(m: Classifiable | null | undefined): ItemClass {
  const list = m?.controlled_list ?? null;
  if (list && NOTIFICATION_LISTS.includes(list)) return "notification";
  if (list && UNSUPPORTED_LISTS.includes(list)) return "unsupported";
  if (list && SPECIAL_LISTS.includes(list)) return "special_control";
  if (m?.controlled) return "unclassified";
  if (m?.antimicrobial) return "antimicrobial";
  return "common";
}

export function prescriptionCategory(items: ItemDraft[]): PrescriptionCategory | "mixed" {
  const classes = items.map((it) => itemClass(it.medication));
  const special = classes.includes("special_control");
  const antimicrobial = classes.includes("antimicrobial");
  if (special && antimicrobial) return "mixed";
  if (special) return "special_control";
  if (antimicrobial) return "antimicrobial";
  return "common";
}

export function controlledBlockReason(m: Classifiable): string | null {
  switch (itemClass(m)) {
    case "notification": return "Notificação de receita (listas A/B) — registre a de papel em “Notificação (papel)”";
    case "unsupported": return `lista ${m.controlled_list} (retinoide, talidomida ou antirretroviral) — fora do Rota Saúde`;
    case "unclassified": return "controlado sem a lista da Portaria 344 no catálogo — não entra na receita";
    default: return null;
  }
}

export function controlledTagLabel(m: Classifiable): string | null {
  if (!m.controlled_list) return null;
  return `lista ${m.controlled_list}${m.anticonvulsant ? " · anticonvulsivante" : ""}`;
}

export function maxDaysFor(m: Classifiable | null | undefined): number {
  return m?.anticonvulsant ? ANTICONVULSANT_MAX_DAYS : SPECIAL_CONTROL_MAX_DAYS;
}

const CATEGORY_LABEL: Record<string, string> = {
  common: "receita comum", special_control: "receita de controle especial", antimicrobial: "receita de antimicrobiano"
};

export function categoryLabel(category: string): string {
  return CATEGORY_LABEL[category] ?? category;
}

export function documentCategory(doc: ClinicalDocument): PrescriptionCategory | null {
  return doc.kind === "prescription" ? ((doc.content as PrescriptionContent).category ?? null) : null;
}

export function documentSncr(doc: ClinicalDocument): SncrNumberRef | null {
  return doc.kind === "prescription" ? ((doc.content as PrescriptionContent).sncr ?? null) : null;
}

export function documentTitle(doc: ClinicalDocument): string {
  const category = documentCategory(doc);
  if (category === "special_control") return "Receita de controle especial";
  if (category === "antimicrobial") return "Receita de antimicrobiano";
  return kindLabel(doc.kind);
}

export function sncrLabel(s: Pick<SncrNumberRef, "kind" | "number">): string {
  return `${s.kind.toUpperCase()} nº ${s.number}`;
}

// ─── Endereço, identificação do paciente e telefone ───────────────────────────

export interface AddressDraft { street: string; number: string; complement: string; district: string; city: string; uf: string; zip: string }
export const EMPTY_ADDRESS_DRAFT: AddressDraft = { street: "", number: "", complement: "", district: "", city: "", uf: "", zip: "" };

export function addressDraftFrom(a: SncrAddress | null | undefined): AddressDraft {
  if (!a) return EMPTY_ADDRESS_DRAFT;
  return { street: a.street, number: a.number, complement: a.complement ?? "", district: a.district, city: a.city, uf: a.uf, zip: maskCep(a.zip) };
}

export function addressProblem(d: AddressDraft, who: string): string | null {
  if (!d.street.trim()) return `${who}: informe o logradouro`;
  if (!d.number.trim()) return `${who}: informe o número (ou s/n)`;
  if (!d.district.trim()) return `${who}: informe o bairro`;
  if (!d.city.trim()) return `${who}: informe a cidade`;
  if (!UFS.includes(d.uf)) return `${who}: escolha a UF`;
  if (onlyDigits(d.zip).length !== 8) return `${who}: o CEP precisa ter 8 dígitos`;
  return null;
}

// Uma forma por valor (contracts C4): CEP só dígitos, complemento ausente quando vazio.
export function addressInput(d: AddressDraft): SncrAddress {
  const complement = d.complement.trim();
  return {
    street: d.street.trim(), number: d.number.trim(), ...(complement ? { complement } : {}), district: d.district.trim(),
    city: d.city.trim(), uf: d.uf, zip: onlyDigits(d.zip)
  };
}

export function addressLine(a: SncrAddress | null | undefined): string {
  if (!a) return "—";
  const street = `${a.street}, ${a.number}`;
  return [ a.complement ? `${street} — ${a.complement}` : street, a.district, `${a.city}/${a.uf}`, maskCep(a.zip) ].join(" · ");
}

// O CPF é sempre o do cadastro (regra do 19a; sem "não possui CPF"): a tela
// só edita o endereço do paciente.
export interface IdentificationDraft { address: AddressDraft }
export const EMPTY_IDENTIFICATION: IdentificationDraft = { address: EMPTY_ADDRESS_DRAFT };

export function identificationDraftFrom(pi: PatientIdentification | null | undefined): IdentificationDraft {
  return pi ? { address: addressDraftFrom(pi.address) } : EMPTY_IDENTIFICATION;
}

export function identificationProblem(d: IdentificationDraft): string | null {
  return addressProblem(d.address, "endereço do paciente");
}

// Divergência D2: o CPF não vai no corpo — o api usa o do cadastro do paciente.
export function identificationInput(d: IdentificationDraft): PatientIdentificationInput {
  return { address: addressInput(d.address) };
}

export function phoneProblem(text: string): string | null {
  const d = onlyDigits(text);
  if (d.length !== 10 && d.length !== 11) return "telefone: DDD e número, 10 ou 11 dígitos";
  if (Number(d.slice(0, 2)) < 11) return "telefone: DDD inválido";
  return null;
}

export function formatPhone(digits: string | null | undefined): string {
  if (!digits) return "—";
  const d = onlyDigits(digits);
  if (d.length === 11) return `(${d.slice(0, 2)}) ${d.slice(2, 7)}-${d.slice(7)}`;
  if (d.length === 10) return `(${d.slice(0, 2)}) ${d.slice(2, 6)}-${d.slice(6)}`;
  return d;
}

// ─── Limites (espelho dos 422 do api) ─────────────────────────────────────────

export function controlledProblems(d: { items: ItemDraft[] }, identification: IdentificationDraft): string[] {
  const out: string[] = [];
  d.items.forEach((it, i) => {
    const reason = it.medication ? controlledBlockReason(it.medication) : null;
    if (reason) out.push(`item ${i + 1}: ${reason}`);
  });
  const category = prescriptionCategory(d.items);
  if (category === "mixed") return [ ...out, MIXED_CATEGORIES ];
  if (category !== "special_control") return out;
  const c1 = new Set(d.items.filter((it) => it.medication?.controlled_list === "C1")
    .map((it) => it.medication!.active_ingredient.trim().toLowerCase()));
  if (c1.size > SPECIAL_CONTROL_MAX_C1) out.push(`no máximo ${SPECIAL_CONTROL_MAX_C1} substâncias da lista C1 por receita`);
  // Todo item da receita de controle especial leva duração (plano do api, D15).
  d.items.forEach((it, i) => {
    const max = maxDaysFor(it.medication);
    const days = positiveInt(it.durationDays);
    if (it.durationDays.trim() === "") out.push(`item ${i + 1}: informe a duração em dias (até ${max})`);
    else if (days !== null && days > max) {
      out.push(`item ${i + 1}: duração acima de ${max} dias${max === ANTICONVULSANT_MAX_DAYS ? " (anticonvulsivante)" : ""}`);
    }
  });
  const identificationIssue = identificationProblem(identification);
  if (identificationIssue) out.push(identificationIssue);
  return out;
}

// ─── Previsão de papel (o modo é do api) ──────────────────────────────────────

export function sncrKindFor(category: PrescriptionCategory | "mixed"): SncrKind | null {
  if (category === "special_control") return "rce";
  if (category === "antimicrobial") return "ret";
  return null;
}

export interface ForecastInput {
  category: PrescriptionCategory | "mixed"; cboCode: string; signatureOn: boolean;
  certificate: SignerCertificate | null | undefined; stock: SncrStock | undefined;
}

// Indefinido = ainda carregando: não prevê papel por falta de dado.
export function paperForecast(i: ForecastInput): PaperReason | null {
  const kind = sncrKindFor(i.category);
  if (!kind) return null;
  if (kind === "ret" && professionalKind(i.cboCode) === "nurse") return "nurse_antimicrobial";
  if (!i.signatureOn) return "signature_unavailable";
  if (i.certificate === null || (i.certificate && i.certificate.status !== "active")) return "no_certificate";
  if (i.stock && i.stock[kind].free <= 0) return "no_sncr_number";
  return null;
}

const PAPER_REASON_LABEL: Record<string, string> = {
  no_certificate: "você não tem certificado digital ativo — vincule em Conta → Assinatura digital",
  signature_unavailable: "a assinatura digital não está disponível na cidade",
  no_sncr_number: "sem número SNCR no seu estoque — obtenha números em Conta → SNCR",
  nurse_antimicrobial: "a receita de antimicrobiano da enfermagem sai sempre em papel",
  feature_disabled: "a receita de controlado digital está desligada na cidade"
};

export function paperReasonLabel(reason: string | null | undefined): string {
  if (!reason) return "—";
  return PAPER_REASON_LABEL[reason] ?? reason;
}

// ─── Depois da emissão ────────────────────────────────────────────────────────

export function controlledIssuedNotice(doc: ClinicalDocument): string {
  if (doc.kind === "controlled_notification_record") {
    return "Notificação de papel registrada: o medicamento entra na lista de medicamentos em uso. Guarde o talão; nada é impresso aqui.";
  }
  const category = documentCategory(doc);
  if (category !== "special_control" && category !== "antimicrobial") return issuedNotice(doc);
  const what = documentTitle(doc);
  if (doc.issue_mode === "digital") {
    const sncr = documentSncr(doc);
    const number = sncr ? ` com o número SNCR ${sncrLabel(sncr)}${sncr.simulated ? ` (${SNCR_SIMULATED_NOTICE.toLowerCase()})` : ""}` : "";
    return `${what} emitida em modo digital${number}: a assinatura entra na sua fila; o PDF assinado sai depois dela.`;
  }
  const reason = doc.paper_reason ? ` (${paperReasonLabel(doc.paper_reason)})` : "";
  return `${what} emitida em modo papel${reason}: imprima as 2 vias e assine à mão.`;
}

// ─── Estoque ──────────────────────────────────────────────────────────────────

const SNCR_KIND_LABEL: Record<SncrKind, string> = {
  rce: "Receita de controle especial (RCE)", ret: "Receita de antimicrobiano (RET)"
};
const OUT_OF_NUMBERS: Record<SncrKind, string> = {
  rce: "sem números: as receitas de controle especial saem em papel",
  ret: "sem números: as receitas de antimicrobiano saem em papel"
};

export interface StockRow {
  kind: SncrKind; label: string; free: number; used: number; voided: number; requests: number; limitReached: boolean;
  notice: { tone: Tone; text: string } | null;
}

export function stockRows(s: SncrStock): StockRow[] {
  return ([ "rce", "ret" ] as const).map((kind) => {
    const k = s[kind];
    const notice: StockRow["notice"] = k.free <= 0 ? { tone: "down", text: OUT_OF_NUMBERS[kind] }
      : k.free < s.low_threshold ? { tone: "warn", text: `saldo baixo: menos de ${s.low_threshold} números — obtenha mais antes que acabe` }
      : null;
    return {
      kind, label: SNCR_KIND_LABEL[kind], free: k.free, used: k.used, voided: k.voided, requests: k.requests_this_month,
      limitReached: k.requests_this_month >= MONTHLY_REQUEST_LIMIT, notice
    };
  });
}

// ─── Retorno do gov.br ────────────────────────────────────────────────────────

// A volta do login da Anvisa traz `session_id` (ou `error`) e não traz o nosso
// `state` (plano do api, D1/D2); se um dia trouxer, vale o da URL.
export interface SncrCallbackParams { state: string | null; sessionId: string | null; error: string | null }

const SNCR_CALLBACK_PATH = "sncr/callback";
export const SNCR_STATE_STORAGE_KEY = "rotasaude.sncr.state";

function sessionStore(): Storage | null {
  try { return window.sessionStorage; } catch { return null; }
}

// O `state` (só vale com a sessão do próprio usuário no api) fica nesta aba
// entre a ida e a volta; a volta o tira na primeira leitura. Storage
// indisponível não quebra: a volta só não acha o state e pede para começar de novo.
export function rememberSncrState(state: string, storage: Storage | null = sessionStore()): void {
  try { storage?.setItem(SNCR_STATE_STORAGE_KEY, state); } catch { /* sem storage: a volta pede para recomeçar */ }
}

export function takeSncrState(storage: Storage | null = sessionStore()): string | null {
  try {
    const value = storage?.getItem(SNCR_STATE_STORAGE_KEY) ?? null;
    storage?.removeItem(SNCR_STATE_STORAGE_KEY);
    return value && value.trim() !== "" ? value : null;
  } catch {
    return null;
  }
}

// Rota de retorno sob a base do dashboard. Pura (só lê a URL; o StrictMode
// chama o inicializador do useState duas vezes): a base vem por argumento
// (nos testes, import.meta.env.BASE_URL é "/").
export function readSncrCallback(
  pathname: string = window.location.pathname,
  search: string = window.location.search,
  base: string = import.meta.env.BASE_URL
): SncrCallbackParams | null {
  if (pathname.replace(/\/+$/, "") !== `${base}${SNCR_CALLBACK_PATH}`) return null;
  const params = new URLSearchParams(search);
  const read = (key: string) => {
    const value = params.get(key);
    return value && value.trim() !== "" ? value : null;
  };
  return { state: read("state"), sessionId: read("session_id"), error: read("error") };
}

const KIND_FOR_SUMMARY: Record<string, string> = {
  rce: "receita de controle especial (RCE)", ret: "receita de antimicrobiano (RET)"
};

export function sncrCallbackSummary(r: SncrCallbackResult): string {
  const noun = r.received === 1 ? "número recebido" : "números recebidos";
  return `${fmtNumber(r.received)} ${noun} do SNCR para ${KIND_FOR_SUMMARY[r.kind] ?? r.kind}.`;
}

export function sncrCallbackLanding(r: { return_to?: unknown } | null | undefined): string {
  const to = r?.return_to;
  return typeof to === "string" && /^\/[^/\\]/.test(to) ? to : SNCR_RETURN_TO;
}

export function sncrOauthErrorPhrase(error: string | null): string {
  return error === "access_denied"
    ? "a autorização foi negada no gov.br — nenhum número foi pedido"
    : "o gov.br não concluiu a autorização — comece de novo";
}

// ─── Notificação de papel ─────────────────────────────────────────────────────

export function notificationTypeFor(list: string | null | undefined): NotificationType | null {
  if (list === "A1" || list === "A2" || list === "A3") return "A";
  if (list === "B1") return "B";
  if (list === "B2") return "B2";
  return null;
}

export const NOTIFICATION_TYPE_LABEL: Record<NotificationType, string> = {
  A: "Notificação A (amarela — listas A1, A2, A3)", B: "Notificação B (azul — lista B1)", B2: "Notificação B2 (azul — lista B2)"
};

export interface NotificationDraft {
  type: NotificationType | ""; paperNumber: string; numberingUf: string; medication: ItemMedication | null; quantity: string;
  quantityUnit: string; route: MedicationRoute | ""; dosage: string; durationDays: string;
}

export function emptyNotification(uf: string): NotificationDraft {
  return { type: "", paperNumber: "", numberingUf: UFS.includes(uf) ? uf : "", medication: null, quantity: "", quantityUnit: "", route: "",
    dosage: "", durationDays: "" };
}

export function notificationProblems(d: NotificationDraft): string[] {
  const out: string[] = [];
  if (!d.type) out.push("escolha o tipo da Notificação");
  if (!d.paperNumber.trim()) out.push("informe o número impresso no talão");
  if (!UFS.includes(d.numberingUf)) out.push("escolha a UF da numeração");
  if (!d.medication) out.push("escolha o medicamento (listas A ou B)");
  else {
    const expected = notificationTypeFor(d.medication.controlled_list);
    if (!expected) out.push("o medicamento escolhido não é de Notificação (listas A ou B)");
    else if (d.type && expected !== d.type) out.push(`o medicamento é da lista ${d.medication.controlled_list}: a Notificação é ${expected}`);
  }
  if (positiveInt(d.quantity) === null) out.push("informe a quantidade (número inteiro)");
  if (d.medication && !d.quantityUnit.trim()) out.push("informe a unidade da quantidade");
  if (!d.route) out.push("escolha a via");
  if (!d.dosage.trim()) out.push("escreva a posologia");
  const days = positiveInt(d.durationDays);
  if (days === null) out.push("informe a duração em dias");
  else if (d.type && days > NOTIFICATION_MAX_DAYS[d.type]) out.push(`duração acima de ${NOTIFICATION_MAX_DAYS[d.type]} dias para a Notificação ${d.type}`);
  return out;
}

export function notificationInput(d: NotificationDraft): NotificationRecordInput {
  return {
    notification_type: d.type as NotificationType, paper_number: d.paperNumber.trim(), numbering_uf: d.numberingUf,
    item: {
      catalog_item: { id: d.medication!.id }, quantity: positiveInt(d.quantity) ?? 0, quantity_unit: d.quantityUnit.trim(),
      route: d.route as MedicationRoute, dosage_instructions: d.dosage.trim(), duration_days: positiveInt(d.durationDays) ?? 0
    }
  };
}

// ─── Painel do admin ──────────────────────────────────────────────────────────

export function sortOverview(list: SncrOverviewProfessional[]): SncrOverviewProfessional[] {
  return [ ...list ].sort((a, b) => Number(b.low) - Number(a.low) || a.name.localeCompare(b.name, "pt-BR"));
}

export function overviewSummary(o: SncrOverview): { professionals: number; low: number; neverRequested: number } {
  return {
    professionals: o.professionals.length,
    low: o.professionals.filter((p) => p.low).length,
    neverRequested: o.professionals.filter((p) => !p.last_request_at).length
  };
}

// ─── Frases das recusas ───────────────────────────────────────────────────────

const ADDRESS_FIELD: Record<string, string> = {
  street: "o logradouro", number: "o número", complement: "o complemento", district: "o bairro", city: "a cidade", uf: "a UF", zip: "o CEP"
};

const ERRORS: Record<string, string> = {
  monthly_limit_reached: "você já fez os 3 pedidos deste mês para este tipo — o limite volta no mês que vem",
  cbo_not_allowed: "só médicos e dentistas pedem números do SNCR e prescrevem controlado",
  professional_cpf_missing: "seu cadastro está sem CPF — procure a administração da cidade",
  registration_mismatch: "a conta gov.br usada não bate com a sua inscrição no conselho, ou você não tem cadastro de prescritor no SNCR — entre com a sua própria conta gov.br",
  invalid_state: "este retorno do gov.br não vale mais (já usado ou vencido) — comece de novo",
  authorization_expired: "a autorização demorou demais e venceu — comece de novo; o pedido ao SNCR precisa sair em até 30 segundos",
  authorization_denied: "a autorização foi negada no gov.br — nenhum número foi pedido",
  sncr_unavailable: "o SNCR não respondeu — tente de novo em alguns minutos; nenhum número foi gasto",
  sncr_exhausted: "o SNCR está sem números para a sua UF agora — tente mais tarde; até lá, as receitas saem em papel",
  invalid_phone: "telefone inválido — DDD e número, 10 ou 11 dígitos",
  missing_role: "seu papel não permite esta ação"
};

export function controlledError(err: unknown): string {
  if (featureDisabledKey(err) !== null) return CONTROLLED_DISABLED;
  const code = errorCode(err);
  if (code === "invalid_address") {
    const body = (err as { body?: { field?: unknown } }).body;
    const field = typeof body?.field === "string" ? body.field.split(".").pop() ?? "" : "";
    return `endereço inválido: confira ${ADDRESS_FIELD[field] ?? "o endereço"}`;
  }
  if (code && ERRORS[code]) return ERRORS[code];
  const described = describeActionError(err);
  return "message" in described ? described.message : GENERIC;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod19d && npx vitest run src/lib/controlled.test.ts src/lib/clinicalDocuments.test.ts && npx tsc --noEmit`
Expected: PASS e `tsc` sem erro. Se `controlledError(err(500, ""))` sair diferente, confira o que `describeActionError` devolve para 500 (`src/lib/actionErrors.ts`): a frase dele é a esperada (o teste só exige que não haja `undefined`).

- [ ] **Step 5: Commit**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/dashboard/.claude/mod19d
/opt/homebrew/bin/git add src/lib/controlled.ts src/lib/controlled.test.ts src/lib/clinicalDocuments.ts src/lib/clinicalDocuments.test.ts
/opt/homebrew/bin/git commit -m "feat: add screen rules for controlled prescriptions, SNCR stock and paper notifications

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Endereço e telefone do prescritor em Meu perfil

**Files:**
- Create: `src/modules/documents/AddressFields.tsx`, `src/modules/professionals/ContactPanel.tsx`
- Modify: `src/modules/MyProfile.tsx`
- Test: `src/modules/professionals/ContactPanel.test.tsx`, `src/modules/MyProfile.contact.test.tsx`

**Interfaces:**
- Consumes: `getProfessionalContact`, `saveProfessionalContact`, `getMyProfessional` (Task 1 e módulo 10); `CONTACT_KEY`, `AddressDraft`, `EMPTY_ADDRESS_DRAFT`, `UFS`, `addressDraftFrom`, `addressProblem`, `addressInput`, `addressLine`, `phoneProblem`, `formatPhone`, `controlledOn`, `controlledError` (Task 2); `onlyDigits` (`attendance.ts`); `maskCep` (`unitAddress.ts`); `Panel`, `KeyValue`, `formStyles` (comum) e `src/modules/documents/formStyles.ts` (19c).
- Produces:
  - `AddressFields({ legend: string; value: AddressDraft; onChange(next: AddressDraft): void })` — `<fieldset aria-label={legend}>` com "Logradouro", "Número", "Complemento", "Bairro", "Cidade", "UF" (`<select>` com as 27 UFs) e "CEP" (máscara `00000-000`);
  - `ContactPanel()` — `Panel` "Endereço e telefone na receita de controle especial": mostra o endereço e o telefone do `GET …/contact` (ou que faltam, com `role="note"`), "Cadastrar endereço e telefone"/"Alterar endereço e telefone" abre o formulário `aria-label="Endereço e telefone do prescritor"` ("Salvar", "Cancelar"); grava com `PUT` e diz "Endereço e telefone salvos.";
  - `MyProfile` desenha `<ContactPanel />` depois de "Nome e contato" quando `controlledOn(user)`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/professionals/ContactPanel.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getProfessionalContact: vi.fn(), saveProfessionalContact: vi.fn() };
});

import * as api from "../../lib/api";
import { ApiError } from "../../lib/api";
import { ContactPanel } from "./ContactPanel";
import { renderWithProviders } from "../../test/campaignFixtures";
import { contact, controlledUser } from "../../test/controlledFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

function fillForm(phone = "(41) 3350-1234") {
  const form = screen.getByRole("form", { name: "Endereço e telefone do prescritor" });
  const w = within(form);
  fireEvent.change(w.getByLabelText("Logradouro"), { target: { value: "Rua Ouvidor Pardinho" } });
  fireEvent.change(w.getByLabelText("Número"), { target: { value: "28" } });
  fireEvent.change(w.getByLabelText("Bairro"), { target: { value: "Rebouças" } });
  fireEvent.change(w.getByLabelText("Cidade"), { target: { value: "Curitiba" } });
  fireEvent.change(w.getByLabelText("UF"), { target: { value: "PR" } });
  fireEvent.change(w.getByLabelText("CEP"), { target: { value: "80230040" } });
  fireEvent.change(w.getByLabelText("Telefone"), { target: { value: phone } });
  return form;
}

afterEach(cleanup);

describe("ContactPanel", () => {
  beforeEach(() => {
    vi.clearAllMocks();
    mocked(api.fetchCurrentSession).mockResolvedValue(controlledUser());
  });

  it("sem cadastro: diz que a receita de controle especial não sai e cadastra com PUT só com dígitos", async () => {
    mocked(api.getProfessionalContact).mockResolvedValue({ address: null, phone: null });
    mocked(api.saveProfessionalContact).mockResolvedValue(contact());
    renderWithProviders(<ContactPanel />);

    expect((await screen.findByRole("note")).textContent)
      .toBe("sem endereço e telefone, a receita de controle especial não sai — cadastre aqui");
    fireEvent.click(screen.getByRole("button", { name: "Cadastrar endereço e telefone" }));
    const form = fillForm();
    expect((within(form).getByLabelText("CEP") as HTMLInputElement).value).toBe("80230-040");
    fireEvent.click(within(form).getByRole("button", { name: "Salvar" }));

    await waitFor(() => expect(api.saveProfessionalContact).toHaveBeenCalledWith({
      address: { street: "Rua Ouvidor Pardinho", number: "28", district: "Rebouças", city: "Curitiba", uf: "PR", zip: "80230040" },
      phone: "4133501234"
    }));
    expect((await screen.findByRole("status")).textContent).toBe("Endereço e telefone salvos.");
    expect(screen.getByText("Rua Ouvidor Pardinho, 28 · Rebouças · Curitiba/PR · 80230-040")).not.toBeNull();
    expect(screen.getByText("(41) 3350-1234")).not.toBeNull();
    expect(screen.queryByRole("note")).toBeNull();
  });

  it("CEP incompleto ou telefone sem DDD: diz na tela e não manda", async () => {
    mocked(api.getProfessionalContact).mockResolvedValue({ address: null, phone: null });
    renderWithProviders(<ContactPanel />);
    fireEvent.click(await screen.findByRole("button", { name: "Cadastrar endereço e telefone" }));
    const form = fillForm("3350-1234");
    fireEvent.click(within(form).getByRole("button", { name: "Salvar" }));
    expect((await within(form).findByRole("alert")).textContent).toBe("telefone: DDD e número, 10 ou 11 dígitos");
    fireEvent.change(within(form).getByLabelText("CEP"), { target: { value: "8023" } });
    fireEvent.click(within(form).getByRole("button", { name: "Salvar" }));
    expect(within(form).getByRole("alert").textContent).toBe("endereço: o CEP precisa ter 8 dígitos");
    expect(api.saveProfessionalContact).not.toHaveBeenCalled();
  });

  it("recusa do api (invalid_address com field) aparece no formulário, que fica preenchido", async () => {
    mocked(api.getProfessionalContact).mockResolvedValue({ address: null, phone: null });
    mocked(api.saveProfessionalContact).mockRejectedValue(new ApiError(422, { error: "invalid_address", field: "zip" }, "422"));
    renderWithProviders(<ContactPanel />);
    fireEvent.click(await screen.findByRole("button", { name: "Cadastrar endereço e telefone" }));
    const form = fillForm();
    fireEvent.click(within(form).getByRole("button", { name: "Salvar" }));
    expect((await within(form).findByRole("alert")).textContent).toBe("endereço inválido: confira o CEP");
    expect((within(form).getByLabelText("Logradouro") as HTMLInputElement).value).toBe("Rua Ouvidor Pardinho");
  });

  it("com cadastro: mostra e altera a partir do que existe", async () => {
    mocked(api.getProfessionalContact).mockResolvedValue(contact());
    renderWithProviders(<ContactPanel />);
    expect(await screen.findByText("Rua Ouvidor Pardinho, 28 · Rebouças · Curitiba/PR · 80230-040")).not.toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Alterar endereço e telefone" }));
    const form = screen.getByRole("form", { name: "Endereço e telefone do prescritor" });
    expect((within(form).getByLabelText("Bairro") as HTMLInputElement).value).toBe("Rebouças");
    expect((within(form).getByLabelText("Telefone") as HTMLInputElement).value).toBe("(41) 3350-1234");
    fireEvent.click(within(form).getByRole("button", { name: "Cancelar" }));
    expect(screen.queryByRole("form", { name: "Endereço e telefone do prescritor" })).toBeNull();
  });

  it("interruptor desligado no meio do caminho: diz", async () => {
    mocked(api.getProfessionalContact).mockRejectedValue(new ApiError(403, { error: "feature_disabled", feature: "controlled_prescriptions" }, "403"));
    renderWithProviders(<ContactPanel />);
    expect((await screen.findByRole("alert")).textContent).toBe("a receita de controlado está desligada nesta cidade");
  });
});
```

```tsx
// src/modules/MyProfile.contact.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, screen } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getMyProfessional: vi.fn(), getProfessionalContact: vi.fn() };
});

import * as api from "../lib/api";
import { MyProfile } from "./MyProfile";
import { renderWithProviders } from "../test/campaignFixtures";
import { contact, controlledUser } from "../test/controlledFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const PROFESSIONAL = {
  professional: { id: "pr1", user_id: "us1", email_address: "medica@curitiba.demo", professional_name: "Dra. Helena Prado",
    council: "CRM", council_state: "PR", registration_number: "45678", cns_masked: "***.****.****.1234", phone: null, contact_email: null },
  links: [], shifts: []
};

afterEach(cleanup);

describe("Meu perfil — endereço e telefone da receita (19d)", () => {
  beforeEach(() => {
    vi.clearAllMocks();
    mocked(api.getMyProfessional).mockResolvedValue(PROFESSIONAL);
    mocked(api.getProfessionalContact).mockResolvedValue(contact());
  });

  it("com controlled_prescriptions, mostra o painel do endereço e telefone", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(controlledUser());
    renderWithProviders(<MyProfile />);
    expect(await screen.findByText("Endereço e telefone na receita de controle especial")).not.toBeNull();
  });

  it("sem o interruptor, nem consulta o contato", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(controlledUser([ "health_professional" ], { features: [ "clinical_documents" ] }));
    renderWithProviders(<MyProfile />);
    expect(await screen.findByText("Dados conferidos pela prefeitura")).not.toBeNull();
    expect(screen.queryByText("Endereço e telefone na receita de controle especial")).toBeNull();
    expect(api.getProfessionalContact).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod19d && npx vitest run src/modules/professionals/ContactPanel.test.tsx src/modules/MyProfile.contact.test.tsx`
Expected: FAIL — `./ContactPanel` não existe e Meu perfil não desenha o painel.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/documents/AddressFields.tsx
// Endereço completo (módulo 19d; Lei 5.991, art. 35; contrato §2 e §5): o do
// paciente na receita de controle especial e o do prescritor no perfil. Os
// campos são texto; a conversão para o corpo (CEP só dígitos, complemento
// omitido quando vazio) é de `addressInput`.
import type { CSSProperties } from "react";
import { UFS, type AddressDraft } from "../../lib/controlled";
import { maskCep } from "../../lib/unitAddress";
import { inputStyle } from "../../components/formStyles";
import { labelStyle, rowStyle } from "./formStyles";

interface Props { legend: string; value: AddressDraft; onChange(next: AddressDraft): void }

export function AddressFields({ legend, value, onChange }: Props) {
  const set = (patch: Partial<AddressDraft>) => onChange({ ...value, ...patch });
  return (
    <fieldset aria-label={legend} style={box}>
      <legend style={legendStyle}>{legend}</legend>
      <div style={rowStyle}>
        <label style={{ ...labelStyle, flex: 2 }}>
          Logradouro
          <input value={value.street} style={inputStyle} onChange={(e) => set({ street: e.target.value })} />
        </label>
        <label style={labelStyle}>
          Número
          <input value={value.number} style={inputStyle} onChange={(e) => set({ number: e.target.value })} />
        </label>
        <label style={labelStyle}>
          Complemento
          <input value={value.complement} style={inputStyle} onChange={(e) => set({ complement: e.target.value })} />
        </label>
      </div>
      <div style={rowStyle}>
        <label style={labelStyle}>
          Bairro
          <input value={value.district} style={inputStyle} onChange={(e) => set({ district: e.target.value })} />
        </label>
        <label style={labelStyle}>
          Cidade
          <input value={value.city} style={inputStyle} onChange={(e) => set({ city: e.target.value })} />
        </label>
        <label style={labelStyle}>
          UF
          <select value={value.uf} style={inputStyle} onChange={(e) => set({ uf: e.target.value })}>
            <option value="">escolha</option>
            {UFS.map((uf) => <option key={uf} value={uf}>{uf}</option>)}
          </select>
        </label>
        <label style={labelStyle}>
          CEP
          <input inputMode="numeric" value={value.zip} style={inputStyle} onChange={(e) => set({ zip: maskCep(e.target.value) })} />
        </label>
      </div>
    </fieldset>
  );
}

const box: CSSProperties = { display: "flex", flexDirection: "column", gap: 8, margin: 0, padding: 10, border: "1px solid var(--rule)", borderRadius: 6 };
const legendStyle: CSSProperties = { fontSize: 12, color: "var(--ink2)", padding: "0 4px" };
```

```tsx
// src/modules/professionals/ContactPanel.tsx
// Endereço e telefone do prescritor (módulo 19d; spec §5 e §7; contrato §5):
// impressos na receita de controle especial (obrigatórios; sem eles o api
// recusa com prescriber_address_missing) e na de antimicrobiano. Só o próprio
// profissional lê e grava. Leitura com gcTime 0 (dado pessoal).
import { useState, type CSSProperties, type FormEvent } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { getProfessionalContact, saveProfessionalContact } from "../../lib/api";
import { useAuth } from "../../lib/auth";
import { onlyDigits } from "../../lib/attendance";
import {
  CONTACT_KEY, EMPTY_ADDRESS_DRAFT, addressDraftFrom, addressInput, addressLine, addressProblem, controlledError, formatPhone, phoneProblem,
  type AddressDraft
} from "../../lib/controlled";
import { Panel } from "../../components/Panel";
import { KeyValue } from "../../components/KeyValue";
import { buttonStyle, disabledButtonStyle, inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { AddressFields } from "../documents/AddressFields";
import { alertStyle, labelStyle, mutedStyle, rowStyle } from "../documents/formStyles";

export function ContactPanel() {
  const { user } = useAuth();
  const queryClient = useQueryClient();
  const key = [ CONTACT_KEY, user?.id ?? null ];
  const query = useQuery({ queryKey: key, queryFn: getProfessionalContact, gcTime: 0, retry: false });
  const [ editing, setEditing ] = useState(false);
  const [ address, setAddress ] = useState<AddressDraft>(EMPTY_ADDRESS_DRAFT);
  const [ phone, setPhone ] = useState("");
  const [ problem, setProblem ] = useState<string | null>(null);
  const [ saved, setSaved ] = useState(false);
  const [ busy, setBusy ] = useState(false);
  const data = query.data;
  const missing = !!data && (!data.address || !data.phone);

  function startEditing() {
    setAddress(addressDraftFrom(data?.address));
    setPhone(data?.phone ? formatPhone(data.phone) : "");
    setProblem(null); setSaved(false); setEditing(true);
  }

  async function save(event: FormEvent) {
    event.preventDefault();
    if (busy) return;
    const found = addressProblem(address, "endereço") ?? phoneProblem(phone);
    if (found) { setProblem(found); return; }
    setBusy(true); setProblem(null);
    try {
      const next = await saveProfessionalContact({ address: addressInput(address), phone: onlyDigits(phone) });
      queryClient.setQueryData(key, next);
      setEditing(false); setSaved(true);
    } catch (err) {
      setProblem(controlledError(err));
    } finally {
      setBusy(false);
    }
  }

  return (
    <Panel title="Endereço e telefone na receita de controle especial" sub="impressos na receita de controle especial e na de antimicrobiano">
      <div style={body}>
        {query.isPending && <p className="mono" style={mutedStyle}>carregando…</p>}
        {query.isError && <p role="alert" style={alertStyle}>{controlledError(query.error)}</p>}
        {saved && <p role="status" style={statusStyle}>Endereço e telefone salvos.</p>}
        {data && !editing && (
          <>
            <div style={grid}>
              <KeyValue k="Endereço" v={addressLine(data.address)} mono={false} />
              <KeyValue k="Telefone" v={formatPhone(data.phone)} />
            </div>
            {missing && (
              <p role="note" style={{ ...mutedStyle, fontWeight: 600, color: "var(--warn)" }}>
                sem endereço e telefone, a receita de controle especial não sai — cadastre aqui
              </p>
            )}
            <div>
              <button type="button" style={missing ? buttonStyle : secondaryButtonStyle} onClick={startEditing}>
                {missing ? "Cadastrar endereço e telefone" : "Alterar endereço e telefone"}
              </button>
            </div>
          </>
        )}
        {editing && (
          <form aria-label="Endereço e telefone do prescritor" onSubmit={(e) => void save(e)} style={body}>
            <AddressFields legend="Endereço de atendimento" value={address} onChange={setAddress} />
            <label style={labelStyle}>
              Telefone
              <input inputMode="tel" value={phone} style={inputStyle} onChange={(e) => setPhone(e.target.value)} />
            </label>
            {problem && <p role="alert" style={alertStyle}>{problem}</p>}
            <div style={rowStyle}>
              <button type="submit" disabled={busy} style={busy ? disabledButtonStyle : buttonStyle}>Salvar</button>
              <button type="button" disabled={busy} style={secondaryButtonStyle} onClick={() => setEditing(false)}>Cancelar</button>
            </div>
          </form>
        )}
      </div>
    </Panel>
  );
}

const body: CSSProperties = { display: "flex", flexDirection: "column", gap: 10 };
const grid: CSSProperties = { display: "flex", gap: 24, flexWrap: "wrap" };
const statusStyle: CSSProperties = { margin: 0, fontSize: 12.5, fontWeight: 600 };
```

(O `<form aria-label=…>` tem papel `form` acessível só com nome; os testes o acham por `getByRole("form", { name })`.)

Em `src/modules/MyProfile.tsx`:

(a) imports, depois de `import { ProfileForm } from "./professionals/ProfileForm";`:

```tsx
import { ContactPanel } from "./professionals/ContactPanel";
import { controlledOn } from "../lib/controlled";
```

(b) logo depois do `</Panel>` de "Nome e contato":

```tsx
      {/* Módulo 19d: endereço e telefone impressos na receita de controle especial. */}
      {controlledOn(user) && <ContactPanel />}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod19d && npx vitest run src/modules/professionals/ContactPanel.test.tsx src/modules/MyProfile.contact.test.tsx src/modules/MyProfile.test.tsx && npx tsc --noEmit`
Expected: PASS (o teste antigo de Meu perfil continua: a sessão dele não tem `controlled_prescriptions`) e `tsc` sem erro.

- [ ] **Step 5: Commit**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/dashboard/.claude/mod19d
/opt/homebrew/bin/git add src/modules/documents/AddressFields.tsx src/modules/professionals/ContactPanel.tsx \
  src/modules/professionals/ContactPanel.test.tsx src/modules/MyProfile.tsx src/modules/MyProfile.contact.test.tsx
/opt/homebrew/bin/git commit -m "feat: let prescribers keep the address and phone printed on special control prescriptions

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Conta → SNCR

**Files:**
- Create: `src/modules/SncrAccount.tsx`
- Modify: `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/App.tsx`
- Test: `src/modules/SncrAccount.test.tsx`

**Interfaces:**
- Consumes: `getSncrStock`, `requestSncrNumbers`, `errorCode` (Task 1); `SNCR_STOCK_KEY`, `SNCR_RETURN_TO`, `SNCR_SIMULATED_NOTICE`, `MONTHLY_REQUEST_LIMIT`, `CONTROLLED_DISABLED_TEXT`, `canRequestSncr`, `controlledOn`, `usesSimulatedSncr`, `stockRows`, `controlledError`, `rememberSncrState` (Task 2); `goToProvider` (`signature.ts`, 19b); `fmtNumber`; `toneColor` (`theme/tokens.ts`).
- Produces:
  - `SncrAccount({ redirect?(url: string): void })` — `PageHeader` "SNCR"; `Panel` "Números do SNCR" com uma `<section aria-label="<rótulo do tipo>">` por tipo (Livres, Usados, Anulados, Pedidos no mês "N de 3"), o aviso de saldo baixo/zerado (`role="status"`), o botão "Obter números do SNCR" (nome acessível `Obter números do SNCR: <rótulo do tipo>`; guarda o `state` devolvido no `sessionStorage` e vai ao `authorize_url`) ou a frase do limite do mês; a faixa `SNCR_SIMULATED_NOTICE` com `sncr_mock` ou estoque simulado;
  - `ModuleId` `"sncr"`, item "SNCR" (ícone `№`) no grupo **Conta**, visível com `canRequestSncr(user)`; `App` desenha `<SncrAccount />`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/SncrAccount.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getSncrStock: vi.fn(), requestSncrNumbers: vi.fn() };
});

import * as api from "../lib/api";
import { ApiError } from "../lib/api";
import { SncrAccount } from "./SncrAccount";
import { renderWithProviders } from "../test/campaignFixtures";
import { controlledUser, stock } from "../test/controlledFixtures";
import { CONTROLLED_DISABLED_TEXT, SNCR_SIMULATED_NOTICE, SNCR_STATE_STORAGE_KEY } from "../lib/controlled";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const RCE = "Receita de controle especial (RCE)";
const RET = "Receita de antimicrobiano (RET)";

afterEach(cleanup);

describe("SncrAccount", () => {
  let redirect: ReturnType<typeof vi.fn>;

  beforeEach(() => {
    vi.clearAllMocks();
    redirect = vi.fn();
    mocked(api.fetchCurrentSession).mockResolvedValue(controlledUser());
    mocked(api.getSncrStock).mockResolvedValue(stock());
  });

  it("saldo por tipo, pedidos do mês e saldo baixo", async () => {
    renderWithProviders(<SncrAccount redirect={redirect} />);
    const rce = await screen.findByRole("region", { name: RCE });
    expect(within(rce).getByText("812")).not.toBeNull();
    expect(within(rce).getByText("1 de 3")).not.toBeNull();
    expect(within(rce).queryByRole("status")).toBeNull();
    const ret = screen.getByRole("region", { name: RET });
    expect(within(ret).getByText("40")).not.toBeNull();
    expect(within(ret).getByRole("status").textContent).toBe("saldo baixo: menos de 50 números — obtenha mais antes que acabe");
    expect(document.body.textContent).not.toMatch(/undefined|null|Numeração simulada/);
  });

  it("Obter números do SNCR guarda o state e vai ao login da Anvisa com kind e return_to", async () => {
    window.sessionStorage.clear();
    mocked(api.requestSncrNumbers).mockResolvedValue({ authorize_url: "https://sncr-auth.example.br/auth/login?client_url=x", state: "st9" });
    renderWithProviders(<SncrAccount redirect={redirect} />);
    fireEvent.click(await screen.findByRole("button", { name: `Obter números do SNCR: ${RET}` }));
    await waitFor(() => expect(redirect).toHaveBeenCalledWith("https://sncr-auth.example.br/auth/login?client_url=x"));
    expect(api.requestSncrNumbers).toHaveBeenCalledWith("ret", "/sncr");
    expect(window.sessionStorage.getItem(SNCR_STATE_STORAGE_KEY)).toBe("st9");
    window.sessionStorage.clear();
  });

  it("sem números: diz que sai em papel; com 3 pedidos no mês, sem botão", async () => {
    mocked(api.getSncrStock).mockResolvedValue(stock({ rce: { free: 0, used: 3000, voided: 0, requests_this_month: 3 } }));
    renderWithProviders(<SncrAccount redirect={redirect} />);
    const rce = await screen.findByRole("region", { name: RCE });
    expect(within(rce).getByRole("status").textContent).toBe("sem números: as receitas de controle especial saem em papel");
    expect(within(rce).queryByRole("button")).toBeNull();
    expect(within(rce).getByText("limite de 3 pedidos por mês atingido — o próximo pedido fica para o mês que vem")).not.toBeNull();
  });

  it("limite do mês alcançado noutra aba (409): diz e relê o saldo", async () => {
    mocked(api.requestSncrNumbers).mockRejectedValue(new ApiError(409, { error: "monthly_limit_reached" }, "409"));
    renderWithProviders(<SncrAccount redirect={redirect} />);
    fireEvent.click(await screen.findByRole("button", { name: `Obter números do SNCR: ${RCE}` }));
    expect((await screen.findByRole("alert")).textContent).toBe("você já fez os 3 pedidos deste mês para este tipo — o limite volta no mês que vem");
    await waitFor(() => expect(api.getSncrStock).toHaveBeenCalledTimes(2));
    expect(redirect).not.toHaveBeenCalled();
    expect((screen.getByRole("button", { name: `Obter números do SNCR: ${RCE}` }) as HTMLButtonElement).disabled).toBe(false);
  });

  it("enfermeira (403 cbo_not_allowed): diz quem pede", async () => {
    mocked(api.getSncrStock).mockRejectedValue(new ApiError(403, { error: "cbo_not_allowed" }, "403"));
    renderWithProviders(<SncrAccount redirect={redirect} />);
    expect((await screen.findByRole("alert")).textContent).toBe("só médicos e dentistas pedem números do SNCR e prescrevem controlado");
  });

  it("SNCR simulado: faixa sem validade", async () => {
    mocked(api.getSncrStock).mockResolvedValue(stock({ simulated: true }));
    renderWithProviders(<SncrAccount redirect={redirect} />);
    expect(await screen.findByText(SNCR_SIMULATED_NOTICE)).not.toBeNull();
  });

  it("interruptor desligado: diz, sem consultar", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(controlledUser([ "health_professional" ], { features: [ "clinical_documents" ] }));
    renderWithProviders(<SncrAccount redirect={redirect} />);
    expect(await screen.findByText(CONTROLLED_DISABLED_TEXT)).not.toBeNull();
    expect(api.getSncrStock).not.toHaveBeenCalled();
  });
});
```

Em `src/shell/modules.test.ts`, dentro do `describe("navGroupsFor")`, no fim:

```ts
    it("Conta → SNCR: só profissional com controlled_prescriptions (e clinical_documents), nunca operador", () => {
      const ids = (u: Parameters<typeof navGroupsFor>[0]) => navGroupsFor(u).flatMap((g) => g.items.map((i) => i.id));
      const pro = { operator: false, memberships: [ { role: "health_professional" } ] };
      expect(ids({ ...pro, features: [ "clinical_documents", "controlled_prescriptions" ] })).toContain("sncr");
      expect(ids({ ...pro, features: [ "clinical_documents" ] })).not.toContain("sncr");
      expect(ids({ operator: false, memberships: [ { role: "municipal_admin" } ], features: [ "clinical_documents", "controlled_prescriptions" ] }))
        .not.toContain("sncr");
      expect(ids({ ...pro, operator: true, features: [ "clinical_documents", "controlled_prescriptions" ] })).not.toContain("sncr");
      expect(labelFor("sncr")).toBe("SNCR");
      expect(moduleFromPath("/sncr")).toBe("sncr");
    });
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod19d && npx vitest run src/modules/SncrAccount.test.tsx src/shell/modules.test.ts`
Expected: FAIL — `./SncrAccount` não existe e `"sncr"` não está no catálogo de módulos.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/SncrAccount.tsx
// Conta → SNCR (módulo 19d, F-19.25; spec §4 e §7; contrato §4): o estoque
// pessoal de números do SNCR por tipo (RCE, RET), os pedidos do mês (até 3,
// blocos de 1.000) e "Obter números do SNCR", que leva ao login da Anvisa com
// o gov.br do próprio prescritor. O token e o pedido ao SNCR são do api (30 s);
// a tela guarda o `state` nesta aba (a volta não o traz; plano do api, D2) e
// segue o authorize_url. Sem número, as receitas saem em papel.
import { useState, type CSSProperties } from "react";
import { useQuery } from "@tanstack/react-query";
import { errorCode, getSncrStock, requestSncrNumbers, type SncrKind } from "../lib/api";
import { useAuth } from "../lib/auth";
import { fmtNumber } from "../lib/format";
import { goToProvider } from "../lib/signature";
import {
  CONTROLLED_DISABLED_TEXT, MONTHLY_REQUEST_LIMIT, SNCR_RETURN_TO, SNCR_SIMULATED_NOTICE, SNCR_STOCK_KEY, canRequestSncr, controlledError,
  controlledOn, rememberSncrState, stockRows, usesSimulatedSncr
} from "../lib/controlled";
import { PageHeader } from "../components/PageHeader";
import { Panel } from "../components/Panel";
import { KeyValue } from "../components/KeyValue";
import { Tag } from "../components/Tag";
import { EmptyState } from "../components/EmptyState";
import { buttonStyle, disabledButtonStyle } from "../components/formStyles";
import { toneColor } from "../theme/tokens";

const HOW_IT_WORKS =
  "Você entra no login do SNCR da Anvisa com a sua conta gov.br e o Rota Saúde pede, em seu nome, um bloco de 1.000 números do tipo escolhido " +
  "(até 3 pedidos por mês). Cada receita digital de controle especial ou de antimicrobiano usa um número; sem número, a receita sai em papel.";

interface Props { redirect?(url: string): void }

export function SncrAccount({ redirect = goToProvider }: Props) {
  const { user } = useAuth();
  const allowed = canRequestSncr(user);
  const query = useQuery({ queryKey: [ SNCR_STOCK_KEY, user?.id ?? null ], queryFn: getSncrStock, enabled: allowed, retry: false });
  const [ asking, setAsking ] = useState<SncrKind | null>(null);
  const [ error, setError ] = useState<string | null>(null);

  if (!allowed) {
    return (
      <div style={page}>
        <PageHeader title="SNCR" sub="conta · números de receita da Anvisa" />
        <EmptyState title={controlledOn(user) ? "só profissionais de saúde pedem números do SNCR" : CONTROLLED_DISABLED_TEXT} />
      </div>
    );
  }

  async function ask(kind: SncrKind) {
    if (asking) return;
    setAsking(kind); setError(null);
    try {
      const { authorize_url, state } = await requestSncrNumbers(kind, SNCR_RETURN_TO);
      rememberSncrState(state);
      redirect(authorize_url);
    } catch (err) {
      setError(controlledError(err));
      if (errorCode(err) === "monthly_limit_reached") void query.refetch();
      setAsking(null);
    }
  }

  const simulated = usesSimulatedSncr(user) || query.data?.simulated === true;

  return (
    <div style={page}>
      <PageHeader title="SNCR" sub="conta · números de receita da Anvisa" />
      <Panel title="Números do SNCR" sub="estoque pessoal para a receita digital de controle especial e de antimicrobiano">
        <div style={body}>
          {simulated && <div><Tag tone="warn" mono={false}>{SNCR_SIMULATED_NOTICE}</Tag></div>}
          <p style={muted}>{HOW_IT_WORKS}</p>
          {query.isPending && <p style={muted}>carregando…</p>}
          {query.isError && <p role="alert" style={alert}>{controlledError(query.error)}</p>}
          {error && <p role="alert" style={alert}>{error}</p>}
          {query.data && stockRows(query.data).map((row) => (
            <section key={row.kind} aria-label={row.label} style={kindBox}>
              <strong style={{ fontSize: 13 }}>{row.label}</strong>
              <div style={grid}>
                <KeyValue k="Livres" v={fmtNumber(row.free)} />
                <KeyValue k="Usados" v={fmtNumber(row.used)} />
                <KeyValue k="Anulados" v={fmtNumber(row.voided)} />
                <KeyValue k="Pedidos no mês" v={`${row.requests} de ${MONTHLY_REQUEST_LIMIT}`} />
              </div>
              {row.notice && <p role="status" style={{ ...text, color: toneColor(row.notice.tone).fg }}>{row.notice.text}</p>}
              {row.limitReached ? (
                <p style={muted}>{`limite de ${MONTHLY_REQUEST_LIMIT} pedidos por mês atingido — o próximo pedido fica para o mês que vem`}</p>
              ) : (
                <div>
                  <button type="button" aria-label={`Obter números do SNCR: ${row.label}`} disabled={asking !== null}
                    style={asking !== null ? disabledButtonStyle : buttonStyle} onClick={() => void ask(row.kind)}>
                    Obter números do SNCR
                  </button>
                </div>
              )}
            </section>
          ))}
        </div>
      </Panel>
    </div>
  );
}

const page: CSSProperties = { display: "flex", flexDirection: "column", gap: 16 };
const body: CSSProperties = { display: "flex", flexDirection: "column", gap: 12 };
const kindBox: CSSProperties = { display: "flex", flexDirection: "column", gap: 8, padding: 12, border: "1px solid var(--rule)", borderRadius: 8 };
const grid: CSSProperties = { display: "grid", gridTemplateColumns: "repeat(auto-fit, minmax(120px, 1fr))", gap: 12 };
const text: CSSProperties = { margin: 0, fontSize: 13 };
const muted: CSSProperties = { margin: 0, fontSize: 12, color: "var(--ink3)" };
const alert: CSSProperties = { margin: 0, fontSize: 12.5, color: "var(--down)" };
```

(`fmtNumber(812)` é "812"; `fmtNumber(40)` é "40" — os testes procuram esses textos dentro de cada seção.)

Em `src/shell/modules.ts`:

(a) import, depois do de `../lib/signature`:

```ts
import { canRequestSncr } from "../lib/controlled";
```

(b) no `ModuleId`, acrescente `| "sncr"` à última linha da união (depois de `"signature-overview"` e do `"clinical-documents-admin"` do 19c);

(c) no grupo **Conta**:

```ts
  { label: "Conta", items: [
    { id: "security", label: "Segurança", icon: "⚿" },
    { id: "my-profile", label: "Meu perfil", icon: "☺" },
    { id: "signature", label: "Assinatura digital", icon: "✍" },
    { id: "sncr", label: "SNCR", icon: "№" }
  ]}
```

(d) no filtro dos itens, depois da linha de `signature-overview`:

```ts
      // Módulo 19d: estoque de números do SNCR, do profissional, com controlled_prescriptions.
      if (item.id === "sncr") return canRequestSncr(user);
```

Em `src/App.tsx`: `import { SncrAccount } from "./modules/SncrAccount";` e, no `switch`, depois de `case "signature-overview": …`:

```tsx
    case "sncr":           return <SncrAccount />;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod19d && npx vitest run src/modules/SncrAccount.test.tsx src/shell/modules.test.ts && npx tsc --noEmit`
Expected: PASS (os testes antigos de `modules.test.ts` continuam: o item novo some sem a sessão e sem o interruptor, e o grupo Conta já existia) e `tsc` sem erro.

- [ ] **Step 5: Commit**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/dashboard/.claude/mod19d
/opt/homebrew/bin/git add src/modules/SncrAccount.tsx src/modules/SncrAccount.test.tsx src/shell/modules.ts src/shell/modules.test.ts src/App.tsx
/opt/homebrew/bin/git commit -m "feat: add the account SNCR screen with number stock and the gov.br request

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Retorno do login do SNCR em `/dashboard/sncr/callback`

**Files:**
- Create: `src/modules/SncrCallback.tsx`
- Modify: `src/main.tsx`
- Test: `src/modules/SncrCallback.test.tsx`

**Interfaces:**
- Consumes: `completeSncrOAuth`, `ApiError` (Task 1); `SNCR_STOCK_KEY`, `SNCR_SIMULATED_NOTICE`, `SncrCallbackParams`, `readSncrCallback`, `takeSncrState`, `sncrCallbackSummary`, `sncrCallbackLanding`, `sncrOauthErrorPhrase`, `controlledError`, `usesSimulatedSncr` (Task 2); `clearSignatureCallbackFromUrl` (`signature.ts`, 19b — troca a entrada do histórico pela base, serve para os dois retornos); `labelFor`, `moduleFromPath`, `ModuleId` (`shell/modules.ts`).
- Produces: `SncrCallback({ params: SncrCallbackParams; onDone(landing: ModuleId): void; clearUrl?(): void; takeState?(): string | null })` — `<section aria-label="Retorno do gov.br — números do SNCR">`; numa única vez (ref que sobrevive ao StrictMode): limpa a URL, tira o `state` do `sessionStorage` (vale o da URL, se vier), manda `{ state, session_id }` (ou `{ state, error }`), relê o estoque, mostra o resumo (`role="status"`) ou a recusa (`role="alert"`) e o botão "Continuar" / "Voltar a SNCR"; `main.tsx` desenha a tela quando `readSncrCallback()` casa, como faz com o retorno da assinatura.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/SncrCallback.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor } from "@testing-library/react";
import { StrictMode } from "react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), completeSncrOAuth: vi.fn() };
});

import * as api from "../lib/api";
import { ApiError } from "../lib/api";
import { SncrCallback } from "./SncrCallback";
import { renderWithProviders } from "../test/campaignFixtures";
import { controlledUser } from "../test/controlledFixtures";
import { SNCR_SIMULATED_NOTICE, SNCR_STATE_STORAGE_KEY, SNCR_STOCK_KEY } from "../lib/controlled";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const BACK = { state: null, sessionId: "sid1", error: null };
const RECEIVED = { kind: "rce", received: 1000, batch_id: "b1", return_to: "/sncr" };

afterEach(() => { cleanup(); window.sessionStorage.clear(); });

describe("SncrCallback", () => {
  let onDone: ReturnType<typeof vi.fn>;
  let clearUrl: ReturnType<typeof vi.fn>;

  beforeEach(() => {
    vi.clearAllMocks();
    mocked(api.completeSncrOAuth).mockReset();
    mocked(api.fetchCurrentSession).mockResolvedValue(controlledUser());
    window.sessionStorage.setItem(SNCR_STATE_STORAGE_KEY, "st1");
    onDone = vi.fn();
    clearUrl = vi.fn();
  });

  it("limpa a URL, usa o state guardado (e o apaga), diz quantos números chegaram, relê o estoque e volta à Conta → SNCR", async () => {
    mocked(api.completeSncrOAuth).mockResolvedValue(RECEIVED);
    const { client } = renderWithProviders(<SncrCallback params={BACK} onDone={onDone} clearUrl={clearUrl} />);
    const invalidate = vi.spyOn(client, "invalidateQueries");
    expect((await screen.findByRole("status")).textContent).toBe("1.000 números recebidos do SNCR para receita de controle especial (RCE).");
    expect(api.completeSncrOAuth).toHaveBeenCalledWith("st1", "sid1");
    expect(clearUrl.mock.invocationCallOrder[0]).toBeLessThan(mocked(api.completeSncrOAuth).mock.invocationCallOrder[0]);
    expect(window.sessionStorage.getItem(SNCR_STATE_STORAGE_KEY)).toBeNull();
    await waitFor(() => expect(invalidate).toHaveBeenCalledWith({ queryKey: [ SNCR_STOCK_KEY ] }));
    fireEvent.click(screen.getByRole("button", { name: "Continuar" }));
    expect(onDone).toHaveBeenCalledWith("sncr");
  });

  it("state na URL vale sobre o guardado", async () => {
    mocked(api.completeSncrOAuth).mockResolvedValue(RECEIVED);
    renderWithProviders(<SncrCallback params={{ state: "url-st", sessionId: "sid1", error: null }} onDone={onDone} clearUrl={clearUrl} />);
    await screen.findByRole("status");
    expect(api.completeSncrOAuth).toHaveBeenCalledWith("url-st", "sid1");
    expect(window.sessionStorage.getItem(SNCR_STATE_STORAGE_KEY)).toBeNull();
  });

  it("StrictMode: troca o session_id uma vez só", async () => {
    mocked(api.completeSncrOAuth).mockResolvedValue(RECEIVED);
    renderWithProviders(<StrictMode><SncrCallback params={BACK} onDone={onDone} clearUrl={clearUrl} /></StrictMode>);
    await screen.findByRole("status");
    expect(api.completeSncrOAuth).toHaveBeenCalledTimes(1);
    expect(clearUrl).toHaveBeenCalledTimes(1);
  });

  it("session_id não vai ao console nem ao storage", async () => {
    const spies = [ "log", "info", "warn", "error", "debug" ].map((m) =>
      vi.spyOn(console, m as "log").mockImplementation(() => undefined));
    window.localStorage.clear();
    mocked(api.completeSncrOAuth).mockResolvedValue(RECEIVED);
    renderWithProviders(<SncrCallback params={{ state: null, sessionId: "SID-XYZ", error: null }} onDone={onDone} clearUrl={clearUrl} />);
    await screen.findByRole("status");
    const logged = spies.flatMap((s) => s.mock.calls.flat().map(String)).join(" ");
    expect(logged).not.toMatch(/SID-XYZ|st1/);
    const stored = [ window.localStorage, window.sessionStorage ]
      .flatMap((s) => Object.keys(s).map((k) => `${k}=${s.getItem(k)}`)).join(" ");
    expect(stored).not.toMatch(/SID-XYZ|st1/);
    spies.forEach((s) => s.mockRestore());
  });

  it("autorização vencida (409 authorization_expired): diz que nada foi pedido e volta à Conta → SNCR", async () => {
    mocked(api.completeSncrOAuth).mockRejectedValue(new ApiError(409, { error: "authorization_expired", return_to: "/sncr" }, "409"));
    renderWithProviders(<SncrCallback params={BACK} onDone={onDone} clearUrl={clearUrl} />);
    expect((await screen.findByRole("alert")).textContent)
      .toBe("a autorização demorou demais e venceu — comece de novo; o pedido ao SNCR precisa sair em até 30 segundos");
    expect(screen.queryByRole("status")).toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Voltar a SNCR" }));
    expect(onDone).toHaveBeenCalledWith("sncr");
  });

  it("SNCR esgotado (503 sncr_exhausted) e state já usado (422 invalid_state)", async () => {
    mocked(api.completeSncrOAuth).mockRejectedValue(new ApiError(503, { error: "sncr_exhausted" }, "503"));
    renderWithProviders(<SncrCallback params={BACK} onDone={onDone} clearUrl={clearUrl} />);
    expect((await screen.findByRole("alert")).textContent)
      .toBe("o SNCR está sem números para a sua UF agora — tente mais tarde; até lá, as receitas saem em papel");
    cleanup();
    window.sessionStorage.setItem(SNCR_STATE_STORAGE_KEY, "st1");
    mocked(api.completeSncrOAuth).mockRejectedValue(new ApiError(422, { error: "invalid_state" }, "422"));
    renderWithProviders(<SncrCallback params={BACK} onDone={onDone} clearUrl={clearUrl} />);
    expect((await screen.findByRole("alert")).textContent).toBe("este retorno do gov.br não vale mais (já usado ou vencido) — comece de novo");
  });

  it("recusa no gov.br com state: avisa o api com o erro e diz que nada foi pedido", async () => {
    mocked(api.completeSncrOAuth).mockRejectedValue(new ApiError(403, { error: "authorization_denied", return_to: "/sncr" }, "403"));
    renderWithProviders(<SncrCallback params={{ state: null, sessionId: null, error: "access_denied" }} onDone={onDone} clearUrl={clearUrl} />);
    expect((await screen.findByRole("alert")).textContent).toBe("a autorização foi negada no gov.br — nenhum número foi pedido");
    expect(api.completeSncrOAuth).toHaveBeenCalledWith("st1", { error: "access_denied" });
  });

  it("sem state (página de retorno recarregada, outra aba): nem chama o api", async () => {
    window.sessionStorage.clear();
    renderWithProviders(<SncrCallback params={BACK} onDone={onDone} clearUrl={clearUrl} />);
    expect((await screen.findByRole("alert")).textContent).toBe("o gov.br não concluiu a autorização — comece de novo");
    expect(api.completeSncrOAuth).not.toHaveBeenCalled();
    expect(clearUrl).toHaveBeenCalledTimes(1);
  });

  it("SNCR simulado: faixa sem validade no retorno", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(controlledUser([ "health_professional" ], {
      features: [ "clinical_documents", "controlled_prescriptions", "sncr_mock" ] }));
    mocked(api.completeSncrOAuth).mockResolvedValue(RECEIVED);
    renderWithProviders(<SncrCallback params={BACK} onDone={onDone} clearUrl={clearUrl} />);
    await screen.findByRole("status");
    expect(await screen.findByText(SNCR_SIMULATED_NOTICE)).not.toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod19d && npx vitest run src/modules/SncrCallback.test.tsx`
Expected: FAIL — `./SncrCallback` não existe.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/SncrCallback.tsx
// Retorno do login do SNCR (módulo 19d, F-19.25; spec §3; contrato §4; plano
// do api, D1/D2). O serviço de login da Anvisa faz o gov.br do prescritor e
// devolve a pessoa a /dashboard/sncr/callback?session_id= (ou error=), sem o
// nosso `state`: ele ficou no sessionStorage desta aba na ida. O api troca o
// session_id pelo token (30 s, nunca guardado) e pede o bloco no mesmo fluxo.
// Tudo roda UMA vez (um ref sobrevive ao StrictMode): a URL é limpa e o state
// sai do storage ANTES do POST. `state` e `session_id` só vão no corpo do POST.
import { useEffect, useRef, useState, type CSSProperties } from "react";
import { useQueryClient } from "@tanstack/react-query";
import { ApiError, completeSncrOAuth } from "../lib/api";
import { useAuth } from "../lib/auth";
import { clearSignatureCallbackFromUrl } from "../lib/signature";
import {
  SNCR_SIMULATED_NOTICE, SNCR_STOCK_KEY, controlledError, sncrCallbackLanding, sncrCallbackSummary, sncrOauthErrorPhrase,
  takeSncrState, usesSimulatedSncr, type SncrCallbackParams
} from "../lib/controlled";
import { labelFor, moduleFromPath, type ModuleId } from "../shell/modules";
import { buttonStyle } from "../components/formStyles";
import { Tag } from "../components/Tag";

type View =
  | { kind: "working" }
  | { kind: "done"; text: string; landing: ModuleId }
  | { kind: "failed"; text: string; landing: ModuleId };

interface Props {
  params: SncrCallbackParams;
  onDone(landing: ModuleId): void;
  clearUrl?(): void;
  takeState?(): string | null;
}

// O api devolve o return_to guardado com o state também nas recusas depois de
// achar o state do próprio usuário (Divergência D6). Sem ele, Conta → SNCR.
function landingFromError(err: unknown): ModuleId {
  const body = err instanceof ApiError ? (err.body as { return_to?: unknown } | null | undefined) : null;
  return moduleFromPath(sncrCallbackLanding(body ?? undefined)) ?? "sncr";
}

export function SncrCallback({ params, onDone, clearUrl = clearSignatureCallbackFromUrl, takeState = takeSncrState }: Props) {
  const queryClient = useQueryClient();
  const { user } = useAuth();
  const started = useRef(false);
  const [ view, setView ] = useState<View>({ kind: "working" });

  useEffect(() => {
    if (started.current) return;
    started.current = true;
    clearUrl();
    const stored = takeState();
    const state = params.state ?? stored;
    const { sessionId, error } = params;
    if (error && state) {
      void (async () => {
        let landing: ModuleId = "sncr";
        try {
          landing = moduleFromPath(sncrCallbackLanding(await completeSncrOAuth(state, { error }))) ?? "sncr";
        } catch (err) {
          landing = landingFromError(err);
        }
        setView({ kind: "failed", text: sncrOauthErrorPhrase(error), landing });
      })();
      return;
    }
    if (error || !state || !sessionId) {
      setView({ kind: "failed", text: sncrOauthErrorPhrase(error), landing: "sncr" });
      return;
    }
    void (async () => {
      try {
        const result = await completeSncrOAuth(state, sessionId);
        void queryClient.invalidateQueries({ queryKey: [ SNCR_STOCK_KEY ] });
        setView({ kind: "done", text: sncrCallbackSummary(result), landing: moduleFromPath(sncrCallbackLanding(result)) ?? "sncr" });
      } catch (err) {
        setView({ kind: "failed", text: controlledError(err), landing: landingFromError(err) });
      }
    })();
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, []);

  return (
    <div style={page}>
      <section aria-label="Retorno do gov.br — números do SNCR" style={panel}>
        <strong>Números do SNCR</strong>
        {usesSimulatedSncr(user) && <div><Tag tone="warn" mono={false}>{SNCR_SIMULATED_NOTICE}</Tag></div>}
        {view.kind === "working" && <p style={text}>pedindo os números ao SNCR…</p>}
        {view.kind === "done" && (
          <>
            <p role="status" style={text}>{view.text}</p>
            <div><button type="button" style={buttonStyle} onClick={() => onDone(view.landing)}>Continuar</button></div>
          </>
        )}
        {view.kind === "failed" && (
          <>
            <p role="alert" style={alert}>{view.text}</p>
            <div><button type="button" style={buttonStyle} onClick={() => onDone(view.landing)}>
              {view.landing === "overview" ? "Voltar ao painel" : `Voltar a ${labelFor(view.landing)}`}
            </button></div>
          </>
        )}
      </section>
    </div>
  );
}

const page: CSSProperties = { minHeight: "100vh", display: "flex", alignItems: "center", justifyContent: "center", padding: 24 };
const panel: CSSProperties = {
  display: "flex", flexDirection: "column", gap: 12, padding: 20, width: "100%", maxWidth: 440,
  border: "1px solid var(--rule)", borderRadius: 10, background: "var(--panel)"
};
const text: CSSProperties = { margin: 0, fontSize: 13 };
const alert: CSSProperties = { margin: 0, fontSize: 13, color: "var(--down)" };
```

Em `src/main.tsx`:

(a) imports, depois dos do 19b:

```tsx
import { SncrCallback } from "./modules/SncrCallback";
import { readSncrCallback, type SncrCallbackParams } from "./lib/controlled";
```

(b) em `AppRoot`, depois do `useState` de `signatureReturn`:

```tsx
  // Módulo 19d: volta do login do SNCR. Lido uma vez e puro (só a URL); a tela
  // de retorno limpa a URL, tira o state do sessionStorage e troca o session_id
  // depois do login.
  const [ sncrReturn, setSncrReturn ] = useState<SncrCallbackParams | null>(() => readSncrCallback());
```

(c) troque o conteúdo do `QueryClientProvider`:

```tsx
    <QueryClientProvider client={queryClient}>
      {signatureReturn
        ? <SignatureCallback params={signatureReturn} onDone={(next) => { setSignatureReturn(null); setLanding(next); }} />
        : sncrReturn
          ? <SncrCallback params={sncrReturn} onDone={(next) => { setSncrReturn(null); setLanding(next); }} />
          : <App initialModule={landing} />}
    </QueryClientProvider>
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod19d && npx vitest run src/modules/SncrCallback.test.tsx src/modules/SignatureCallback.test.tsx && npx tsc --noEmit`
Expected: PASS (o retorno da assinatura do 19b continua igual) e `tsc` sem erro.

- [ ] **Step 5: Commit**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/dashboard/.claude/mod19d
/opt/homebrew/bin/git add src/modules/SncrCallback.tsx src/modules/SncrCallback.test.tsx src/main.tsx
/opt/homebrew/bin/git commit -m "feat: handle the SNCR login return at /dashboard/sncr/callback

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Motivo de bloqueio na busca e o formulário da Notificação de papel

**Files:**
- Modify: `src/modules/documents/MedicationSearch.tsx`, `src/modules/documents/MedicationSearch.test.tsx`
- Create: `src/modules/documents/NotificationRecordForm.tsx`
- Test: `src/modules/documents/NotificationRecordForm.test.tsx`

**Interfaces:**
- Consumes: `searchMedications`, `MedicationSearchItem`, `NotificationRecordInput`, `NotificationType`, `MedicationRoute` (Task 1); `NOTIFICATION_NOTE`, `NOTIFICATION_TYPE_LABEL`, `UFS`, `emptyNotification`, `notificationInput`, `notificationProblems`, `notificationTypeFor`, `NotificationDraft` (Task 2); `ROUTES`, `ROUTE_LABEL`, `documentError`, `CONTROLLED_BLOCKED` (19c); `src/modules/documents/formStyles.ts` (19c).
- Produces:
  - `MedicationSearch` ganha a prop opcional `blockReason?(item: MedicationSearchItem): string | null`: quando passada, decide o bloqueio de cada item no lugar de `blockControlled` (motivo não nulo = botão desligado + selo `down` com o motivo); sem ela, tudo como no 19c. O controlado liberado ganha o selo `controlado · lista <lista>` (ou "controlado" sem lista);
  - `NotificationRecordForm({ unitId?: string; defaultUf: string; onSubmit(content: NotificationRecordInput): Promise<void>; onCancel(): void; searchDelayMs?: number })` — `<form aria-label="Notificação de papel">` com `NOTIFICATION_NOTE` (`role="note"`), "Tipo da Notificação", "Número do talão", "UF da numeração" (padrão: a UF da cidade), a busca "Buscar medicamento (listas A e B)" (só listas A/B entram; o tipo vem da lista quando ainda não foi escolhido), "Trocar medicamento", "Quantidade", "Unidade", "Via", "Duração em dias", "Posologia"; botões "Registrar Notificação" e "Cancelar"; o que falta numa lista `role="alert"`; a recusa do `onSubmit` aparece no formulário, que fica preenchido.

- [ ] **Step 1: Write the failing test**

Em `src/modules/documents/MedicationSearch.test.tsx` (19c), acrescente o import

```tsx
import { MORPHINE, SERTRALINE } from "../../test/controlledFixtures";
```

e, no fim do arquivo:

```tsx
describe("MedicationSearch — motivo de bloqueio por item (19d)", () => {
  afterEach(cleanup);

  function renderWithReason(onPick = vi.fn()) {
    const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
    render(
      <QueryClientProvider client={client}>
        <MedicationSearch label="Buscar medicamento" blockControlled delayMs={0} onPick={onPick}
          blockReason={(item) => (item.controlled_list === "A1" ? "registre a Notificação de papel" : null)} />
      </QueryClientProvider>
    );
    return onPick;
  }

  it("o motivo decide: A1 bloqueado com o motivo; C1 liberado com a lista no selo", async () => {
    (api.searchMedications as ReturnType<typeof vi.fn>).mockResolvedValue([ MORPHINE, SERTRALINE ]);
    const onPick = renderWithReason();
    fireEvent.change(screen.getByLabelText("Buscar medicamento"), { target: { value: "mo" } });
    const list = await screen.findByRole("list", { name: "resultados: Buscar medicamento" });
    expect((within(list).getByRole("button", { name: MORPHINE.label }) as HTMLButtonElement).disabled).toBe(true);
    expect(within(list).getByText("registre a Notificação de papel")).not.toBeNull();
    expect(within(list).getByText("controlado · lista C1")).not.toBeNull();
    fireEvent.click(within(list).getByRole("button", { name: SERTRALINE.label }));
    expect(onPick).toHaveBeenCalledWith(SERTRALINE);
  });
});
```

```tsx
// src/modules/documents/NotificationRecordForm.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor, within } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, searchMedications: vi.fn() };
});

import * as api from "../../lib/api";
import { ApiError, type MedicationSearchItem, type NotificationRecordInput } from "../../lib/api";
import { NOTIFICATION_NOTE } from "../../lib/controlled";
import { NotificationRecordForm } from "./NotificationRecordForm";
import { CLONAZEPAM_B1, MORPHINE, SERTRALINE } from "../../test/controlledFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

function renderForm(onSubmit: (c: NotificationRecordInput) => Promise<void> = vi.fn(async () => {})) {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  render(
    <QueryClientProvider client={client}>
      <NotificationRecordForm unitId="u1" defaultUf="PR" onSubmit={onSubmit} onCancel={vi.fn()} searchDelayMs={0} />
    </QueryClientProvider>
  );
  return onSubmit as ReturnType<typeof vi.fn>;
}

async function pick(med: MedicationSearchItem) {
  mocked(api.searchMedications).mockResolvedValueOnce([ med ]);
  fireEvent.change(screen.getByLabelText("Buscar medicamento (listas A e B)"), { target: { value: med.label.slice(0, 6) } });
  fireEvent.click(await screen.findByRole("button", { name: med.label }));
}

function fill(days = "30") {
  fireEvent.change(screen.getByLabelText("Número do talão"), { target: { value: "PR0012345" } });
  fireEvent.change(screen.getByLabelText("Quantidade"), { target: { value: "30" } });
  fireEvent.change(screen.getByLabelText("Via"), { target: { value: "oral" } });
  fireEvent.change(screen.getByLabelText("Duração em dias"), { target: { value: days } });
  fireEvent.change(screen.getByLabelText("Posologia"), { target: { value: "1 comprimido à noite" } });
}

afterEach(cleanup);

describe("NotificationRecordForm", () => {
  beforeEach(() => { vi.clearAllMocks(); mocked(api.searchMedications).mockReset(); });

  it("registra: o tipo vem da lista do item, a UF é a da cidade e o corpo vai certo", async () => {
    const onSubmit = renderForm();
    expect(screen.getByRole("note").textContent).toBe(NOTIFICATION_NOTE);
    await pick(CLONAZEPAM_B1);
    expect((screen.getByLabelText("Tipo da Notificação") as HTMLSelectElement).value).toBe("B");
    expect((screen.getByLabelText("UF da numeração") as HTMLSelectElement).value).toBe("PR");
    expect((screen.getByLabelText("Unidade") as HTMLInputElement).value).toBe("comprimido");
    fill();
    fireEvent.click(screen.getByRole("button", { name: "Registrar Notificação" }));
    await waitFor(() => expect(onSubmit).toHaveBeenCalledWith({
      notification_type: "B", paper_number: "PR0012345", numbering_uf: "PR",
      item: { catalog_item: { id: "ci3" }, quantity: 30, quantity_unit: "comprimido", route: "oral",
        dosage_instructions: "1 comprimido à noite", duration_days: 30 }
    }));
  });

  it("item fora das listas A/B fica bloqueado na busca", async () => {
    mocked(api.searchMedications).mockResolvedValue([ SERTRALINE, MORPHINE ]);
    renderForm();
    fireEvent.change(screen.getByLabelText("Buscar medicamento (listas A e B)"), { target: { value: "mo" } });
    const list = await screen.findByRole("list", { name: "resultados: Buscar medicamento (listas A e B)" });
    expect((within(list).getByRole("button", { name: SERTRALINE.label }) as HTMLButtonElement).disabled).toBe(true);
    expect(within(list).getByText("não é de Notificação (listas A/B)")).not.toBeNull();
    expect((within(list).getByRole("button", { name: MORPHINE.label }) as HTMLButtonElement).disabled).toBe(false);
  });

  it("tipo que não bate com a lista do item: diz e não manda", async () => {
    const onSubmit = renderForm();
    await pick(CLONAZEPAM_B1);
    fireEvent.change(screen.getByLabelText("Tipo da Notificação"), { target: { value: "A" } });
    fill();
    fireEvent.click(screen.getByRole("button", { name: "Registrar Notificação" }));
    expect(screen.getByRole("alert").textContent).toBe("o medicamento é da lista B1: a Notificação é B");
    expect(onSubmit).not.toHaveBeenCalled();
  });

  it("Notificação A até 30 dias", async () => {
    const onSubmit = renderForm();
    await pick(MORPHINE);
    fill("45");
    fireEvent.click(screen.getByRole("button", { name: "Registrar Notificação" }));
    expect(screen.getByRole("alert").textContent).toBe("duração acima de 30 dias para a Notificação A");
    expect(onSubmit).not.toHaveBeenCalled();
  });

  it("recusa do api fica no formulário, que fica preenchido", async () => {
    renderForm(vi.fn(async () => { throw new ApiError(422, { error: "notification_type_mismatch" }, "422"); }));
    await pick(CLONAZEPAM_B1);
    fill();
    fireEvent.click(screen.getByRole("button", { name: "Registrar Notificação" }));
    expect((await screen.findByRole("alert")).textContent).toBe("o tipo da Notificação não bate com a lista do medicamento");
    expect((screen.getByLabelText("Número do talão") as HTMLInputElement).value).toBe("PR0012345");
  });

  it("trocar o medicamento volta para a busca", async () => {
    renderForm();
    await pick(CLONAZEPAM_B1);
    fireEvent.click(screen.getByRole("button", { name: "Trocar medicamento" }));
    expect(screen.getByLabelText("Buscar medicamento (listas A e B)")).not.toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod19d && npx vitest run src/modules/documents/MedicationSearch.test.tsx src/modules/documents/NotificationRecordForm.test.tsx`
Expected: FAIL — `MedicationSearch` não conhece `blockReason` (o `tsc` acusaria a prop) e `./NotificationRecordForm` não existe.

- [ ] **Step 3: Write minimal implementation**

Em `src/modules/documents/MedicationSearch.tsx`:

(a) no comentário do topo, troque a frase `Na receita, o controlado aparece (a médica precisa` … `Na lista de uso e na REMUME, entra.` por:

```tsx
// primeiro ("na rede"). Na receita, o controlado aparece (a médica precisa
// saber que existe); sem o 19d, não entra; com o 19d (`blockReason`), entra o
// C1/C5 e as listas A/B/C2/C3/C4 ficam bloqueadas com o motivo. Na lista de uso e
// na REMUME, entra.
```

(b) nas `Props`, depois de `delayMs?: number;`:

```tsx
  // Módulo 19d: decide o bloqueio de cada item (motivo não nulo = bloqueado).
  blockReason?(item: MedicationSearchItem): string | null;
```

(c) na desestruturação: `export function MedicationSearch({ label, unitId, blockControlled, onPick, delayMs = 300, blockReason }: Props) {`

(d) no `map` dos resultados, troque `const blocked = blockControlled && item.controlled;` por

```tsx
            const reason = blockReason ? blockReason(item) : (blockControlled && item.controlled ? CONTROLLED_BLOCKED : null);
            const blocked = reason !== null;
```

e a linha do selo do controlado (`{item.controlled && <Tag tone="down" …>…</Tag>}`) por

```tsx
                {reason !== null && <Tag tone="down" mono={false}>{reason}</Tag>}
                {reason === null && item.controlled && (
                  <Tag tone="warn" mono={false}>{item.controlled_list ? `controlado · lista ${item.controlled_list}` : "controlado"}</Tag>
                )}
```

(Sem `blockReason` e com `blockControlled`, o motivo é `CONTROLLED_BLOCKED`, como no 19c; sem os dois, o controlado mostra "controlado", como no 19c.)

```tsx
// src/modules/documents/NotificationRecordForm.tsx
// Registro da Notificação de receita de papel (módulo 19d, F-19.28; spec §5;
// contrato §3): o médico ou dentista preenche a Notificação A/B/B2 no talão da
// VISA e registra aqui o número, a UF da numeração e o medicamento (uma
// substância das listas A ou B). Não gera PDF nem assinatura; entra na lista de
// medicamentos em uso. O api confere tipo × lista e a duração.
import { useState, type FormEvent } from "react";
import type { MedicationRoute, MedicationSearchItem, NotificationRecordInput, NotificationType } from "../../lib/api";
import { ROUTES, ROUTE_LABEL, documentError } from "../../lib/clinicalDocuments";
import {
  NOTIFICATION_NOTE, NOTIFICATION_TYPE_LABEL, UFS, emptyNotification, notificationInput, notificationProblems, notificationTypeFor,
  type NotificationDraft
} from "../../lib/controlled";
import { Tag } from "../../components/Tag";
import { buttonStyle, disabledButtonStyle, inputStyle, secondaryButtonStyle } from "../../components/formStyles";
import { MedicationSearch } from "./MedicationSearch";
import { alertStyle, formBox, labelStyle, mutedStyle, rowStyle } from "./formStyles";

const TYPES: NotificationType[] = [ "A", "B", "B2" ];
const NOT_NOTIFICATION = "não é de Notificação (listas A/B)";
const onlyNotificationLists = (item: MedicationSearchItem) => (notificationTypeFor(item.controlled_list) ? null : NOT_NOTIFICATION);

interface Props {
  unitId?: string;
  defaultUf: string;
  onSubmit(content: NotificationRecordInput): Promise<void>;
  onCancel(): void;
  searchDelayMs?: number;
}

export function NotificationRecordForm({ unitId, defaultUf, onSubmit, onCancel, searchDelayMs }: Props) {
  const [ d, setD ] = useState<NotificationDraft>(() => emptyNotification(defaultUf));
  const [ busy, setBusy ] = useState(false);
  const [ problems, setProblems ] = useState<string[]>([]);
  const set = (patch: Partial<NotificationDraft>) => setD((prev) => ({ ...prev, ...patch }));

  function pick(m: MedicationSearchItem) {
    setD((prev) => ({
      ...prev, medication: m, type: prev.type || (notificationTypeFor(m.controlled_list) ?? ""), quantityUnit: prev.quantityUnit || m.dosage_form
    }));
  }

  async function submit(event: FormEvent) {
    event.preventDefault();
    if (busy) return;
    const found = notificationProblems(d);
    if (found.length > 0) { setProblems(found); return; }
    setBusy(true); setProblems([]);
    try {
      await onSubmit(notificationInput(d));
    } catch (err) {
      setProblems([ documentError(err) ]);
    } finally {
      setBusy(false);
    }
  }

  return (
    <form aria-label="Notificação de papel" onSubmit={(e) => void submit(e)} style={formBox}>
      <strong>Notificação de receita (papel)</strong>
      <p role="note" style={mutedStyle}>{NOTIFICATION_NOTE}</p>
      <div style={rowStyle}>
        <label style={labelStyle}>
          Tipo da Notificação
          <select value={d.type} style={inputStyle} onChange={(e) => set({ type: e.target.value as NotificationType | "" })}>
            <option value="">escolha</option>
            {TYPES.map((t) => <option key={t} value={t}>{NOTIFICATION_TYPE_LABEL[t]}</option>)}
          </select>
        </label>
        <label style={labelStyle}>
          Número do talão
          <input value={d.paperNumber} style={inputStyle} onChange={(e) => set({ paperNumber: e.target.value })} />
        </label>
        <label style={labelStyle}>
          UF da numeração
          <select value={d.numberingUf} style={inputStyle} onChange={(e) => set({ numberingUf: e.target.value })}>
            <option value="">escolha</option>
            {UFS.map((uf) => <option key={uf} value={uf}>{uf}</option>)}
          </select>
        </label>
      </div>

      {d.medication ? (
        <div style={rowStyle}>
          <strong style={{ fontSize: 12.5 }}>{d.medication.label}</strong>
          {d.medication.controlled_list && <Tag tone="warn" mono={false}>{`lista ${d.medication.controlled_list}`}</Tag>}
          <button type="button" style={secondaryButtonStyle} onClick={() => set({ medication: null })}>Trocar medicamento</button>
        </div>
      ) : (
        <MedicationSearch label="Buscar medicamento (listas A e B)" unitId={unitId} blockControlled={false}
          blockReason={onlyNotificationLists} delayMs={searchDelayMs} onPick={pick} />
      )}

      <div style={rowStyle}>
        <label style={labelStyle}>
          Quantidade
          <input inputMode="numeric" value={d.quantity} style={inputStyle} onChange={(e) => set({ quantity: e.target.value })} />
        </label>
        <label style={labelStyle}>
          Unidade
          <input value={d.quantityUnit} style={inputStyle} onChange={(e) => set({ quantityUnit: e.target.value })} />
        </label>
        <label style={labelStyle}>
          Via
          <select value={d.route} style={inputStyle} onChange={(e) => set({ route: e.target.value as MedicationRoute | "" })}>
            <option value="">escolha</option>
            {ROUTES.map((r) => <option key={r} value={r}>{ROUTE_LABEL[r]}</option>)}
          </select>
        </label>
        <label style={labelStyle}>
          Duração em dias
          <input inputMode="numeric" value={d.durationDays} style={inputStyle} onChange={(e) => set({ durationDays: e.target.value })} />
        </label>
      </div>
      <label style={labelStyle}>
        Posologia
        <textarea value={d.dosage} rows={2} style={inputStyle} onChange={(e) => set({ dosage: e.target.value })} />
      </label>

      {problems.length > 0 && (
        <ul role="alert" style={{ ...alertStyle, paddingLeft: 18 }}>
          {problems.map((p) => <li key={p}>{p}</li>)}
        </ul>
      )}
      <div style={rowStyle}>
        <button type="submit" disabled={busy} style={busy ? disabledButtonStyle : buttonStyle}>Registrar Notificação</button>
        <button type="button" disabled={busy} style={secondaryButtonStyle} onClick={onCancel}>Cancelar</button>
      </div>
    </form>
  );
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod19d && npx vitest run src/modules/documents/MedicationSearch.test.tsx src/modules/documents/NotificationRecordForm.test.tsx src/modules/documents/PrescriptionForm.test.tsx && npx tsc --noEmit`
Expected: PASS (os testes do 19c da busca e da receita continuam: sem `blockReason`, nada mudou) e `tsc` sem erro.

- [ ] **Step 5: Commit**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/dashboard/.claude/mod19d
/opt/homebrew/bin/git add src/modules/documents/MedicationSearch.tsx src/modules/documents/MedicationSearch.test.tsx \
  src/modules/documents/NotificationRecordForm.tsx src/modules/documents/NotificationRecordForm.test.tsx
/opt/homebrew/bin/git commit -m "feat: add per-item block reasons to the medication search and the paper notification record form

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: Receita de controle especial e de antimicrobiano no formulário da receita

**Files:**
- Create: `src/modules/documents/PatientIdentificationFields.tsx`
- Modify: `src/modules/documents/PrescriptionForm.tsx`
- Test: `src/modules/documents/PrescriptionForm.controlled.test.tsx`

**Interfaces:**
- Consumes: `getProfessionalContact`, `getSncrStock`, `getCurrentCertificate`, `getLastPatientIdentification` (Tasks 1 e 19b); de `controlled.ts` (Task 2): `CATEGORY_NOTE`, `CONTACT_KEY`, `PRESCRIBER_MISSING`, `SNCR_SIMULATED_NOTICE`, `SNCR_STOCK_KEY`, `addressDraftFrom`, `addressLine`, `categoryLabel`, `controlledBlockReason`, `controlledProblems`, `formatPhone`, `identificationDraftFrom`, `identificationInput`, `paperForecast`, `paperReasonLabel`, `prescriptionCategory`, `IdentificationDraft`; `prescriptionProblems(d, nurse, controlledAllowed)` e `PrescriptionDraft.identification` (Task 2); `CERTIFICATE_KEY` (`signature.ts`); `AddressFields` (Task 3); `MedicationSearch` com `blockReason` (Task 6).
- Produces:
  - `export interface ControlledContext { userId: string | null; simulated: boolean; signatureOn: boolean; consultationId?: string; patientCpfMasked?: string }`;
  - `PrescriptionForm` ganha a prop opcional `controlled?: ControlledContext` — **sem ela, tudo como no 19c**. Com ela: a busca usa `controlledBlockReason`; a categoria aparece (`role="status"`, nome "categoria da receita": "categoria: receita de controle especial" | "categoria: receita de antimicrobiano" | "categoria: receita comum" | "categoria: misturada — separe em duas receitas"); a regra da categoria (`role="note"`, nome "regras da categoria", `CATEGORY_NOTE`) no lugar do `ANTIMICROBIAL_NOTE` do 19c; a previsão (`role="note"`, nome "vai sair em papel": "Esta receita vai sair em papel: <motivo>."); a faixa `SNCR_SIMULATED_NOTICE`; na de controle especial, `PatientIdentificationFields` e, nas duas, o grupo "prescritor" com o endereço e o telefone do perfil (ou `PRESCRIBER_MISSING` na de controle especial); o corpo da de controle especial leva `patient_identification` (sem CPF);
  - `PatientIdentificationFields({ value: IdentificationDraft; onChange(next): void; cpfMasked?: string; onUseLast?(): void; lastNote?: string | null })` — `<fieldset aria-label="Identificação do paciente">` com "CPF do cadastro: <mascarado>", "Usar a última identificação" e o `AddressFields` "Endereço do paciente".

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/documents/PrescriptionForm.controlled.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor, within } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, searchMedications: vi.fn(), listNursingProtocols: vi.fn(), getProfessionalContact: vi.fn(), getSncrStock: vi.fn(),
    getCurrentCertificate: vi.fn(), getLastPatientIdentification: vi.fn() };
});

import * as api from "../../lib/api";
import { ApiError, type MedicationSearchItem, type PrescriptionInput } from "../../lib/api";
import { ANTIMICROBIAL_NOTE, CONTROLLED_BLOCKED, itemFromPrescription, type PrescriptionDraft } from "../../lib/clinicalDocuments";
import { CATEGORY_NOTE, MIXED_CATEGORIES, PRESCRIBER_MISSING, SNCR_SIMULATED_NOTICE } from "../../lib/controlled";
import { PrescriptionForm, type ControlledContext } from "./PrescriptionForm";
import { prescriptionItem } from "../../test/documentFixtures";
import { certificate } from "../../test/signatureFixtures";
import {
  AMITRIPTYLINE, AMOXICILLIN_19D, CARBAMAZEPINE, CLONAZEPAM_B1, FLUOXETINE, ISOTRETINOIN, MORPHINE, SERTRALINE, SERTRALINE_REF, TODAY19D,
  address, contact, stock
} from "../../test/controlledFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const EMPTY: PrescriptionDraft = { items: [], protocolVersionId: "" };
const CONTEXT: ControlledContext = { userId: "us1", simulated: false, signatureOn: true, consultationId: "cs1", patientCpfMasked: "***.533.447-**" };
const PAPER = "vai sair em papel";

function renderForm(opts: { controlled?: ControlledContext | null; onSubmit?: (c: PrescriptionInput) => Promise<void>; initial?: PrescriptionDraft } = {}) {
  const onSubmit = opts.onSubmit ?? vi.fn(async (_c: PrescriptionInput) => {});
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  render(
    <QueryClientProvider client={client}>
      <PrescriptionForm cboCode="225142" unitId="u1" today={TODAY19D} initial={opts.initial ?? EMPTY} onSubmit={onSubmit}
        onCancel={vi.fn()} searchDelayMs={0} controlled={opts.controlled === null ? undefined : (opts.controlled ?? CONTEXT)} />
    </QueryClientProvider>
  );
  return onSubmit as ReturnType<typeof vi.fn>;
}

async function pick(med: MedicationSearchItem) {
  mocked(api.searchMedications).mockResolvedValueOnce([ med ]);
  fireEvent.change(screen.getByLabelText("Buscar medicamento"), { target: { value: med.label.slice(0, 6) } });
  fireEvent.click(await screen.findByRole("button", { name: med.label }));
}

function fillItem(n: number, days: string) {
  const li = screen.getByRole("listitem", { name: `item ${n}` });
  fireEvent.change(within(li).getByLabelText(`Quantidade ${n}`), { target: { value: "30" } });
  fireEvent.change(within(li).getByLabelText(`Via ${n}`), { target: { value: "oral" } });
  fireEvent.change(within(li).getByLabelText(`Duração em dias ${n}`), { target: { value: days } });
  fireEvent.change(within(li).getByLabelText(`Posologia ${n}`), { target: { value: "1 comprimido ao dia" } });
}

function fillAddress() {
  const w = within(screen.getByRole("group", { name: "Identificação do paciente" }));
  fireEvent.change(w.getByLabelText("Logradouro"), { target: { value: "Rua Marechal Deodoro" } });
  fireEvent.change(w.getByLabelText("Número"), { target: { value: "630" } });
  fireEvent.change(w.getByLabelText("Bairro"), { target: { value: "Centro" } });
  fireEvent.change(w.getByLabelText("Cidade"), { target: { value: "Curitiba" } });
  fireEvent.change(w.getByLabelText("UF"), { target: { value: "PR" } });
  fireEvent.change(w.getByLabelText("CEP"), { target: { value: "80010-010" } });
}

const emit = () => fireEvent.click(screen.getByRole("button", { name: "Emitir receita" }));
const ADDRESS_BODY = { street: "Rua Marechal Deodoro", number: "630", district: "Centro", city: "Curitiba", uf: "PR", zip: "80010010" };

afterEach(cleanup);

describe("PrescriptionForm — controlado e antimicrobiano (19d)", () => {
  beforeEach(() => {
    vi.clearAllMocks();
    mocked(api.searchMedications).mockReset();
    mocked(api.listNursingProtocols).mockResolvedValue([]);
    mocked(api.getProfessionalContact).mockResolvedValue(contact());
    mocked(api.getSncrStock).mockResolvedValue(stock());
    mocked(api.getCurrentCertificate).mockResolvedValue(certificate());
    mocked(api.getLastPatientIdentification).mockResolvedValue({ cpf: "39053344705", address: address() });
  });

  it("controle especial: categoria, regras, CPF do cadastro, endereço do paciente e o corpo sem CPF", async () => {
    const onSubmit = renderForm();
    await pick(SERTRALINE);
    fillItem(1, "60");
    expect((await screen.findByRole("status", { name: "categoria da receita" })).textContent).toBe("categoria: receita de controle especial");
    expect(screen.getByRole("note", { name: "regras da categoria" }).textContent).toBe(CATEGORY_NOTE.special_control);
    const id = screen.getByRole("group", { name: "Identificação do paciente" });
    expect(within(id).getByText("***.533.447-**")).not.toBeNull();
    expect(await within(screen.getByRole("group", { name: "prescritor" }))
      .findByText("Rua Ouvidor Pardinho, 28 · Rebouças · Curitiba/PR · 80230-040 · (41) 3350-1234")).not.toBeNull();

    emit();
    expect(screen.getByRole("alert").textContent).toContain("endereço do paciente: informe o logradouro");
    expect(onSubmit).not.toHaveBeenCalled();

    fillAddress();
    emit();
    await waitFor(() => expect(onSubmit).toHaveBeenCalledTimes(1));
    expect(onSubmit.mock.calls[0][0]).toEqual({
      items: [ { catalog_item: { id: "ci10" }, quantity: 30, quantity_unit: "comprimido", route: "oral",
        dosage_instructions: "1 comprimido ao dia", duration_days: 60, continuous: false } ],
      patient_identification: { address: ADDRESS_BODY }
    });
    expect(onSubmit.mock.calls[0][0].patient_identification).not.toHaveProperty("cpf");
  });

  it("antimicrobiano: categoria e regra do 19d, sem identificação do paciente e sem o aviso de papel do 19c", async () => {
    const onSubmit = renderForm();
    await pick(AMOXICILLIN_19D);
    fillItem(1, "7");
    expect((await screen.findByRole("status", { name: "categoria da receita" })).textContent).toBe("categoria: receita de antimicrobiano");
    expect(screen.getByRole("note", { name: "regras da categoria" }).textContent).toBe(CATEGORY_NOTE.antimicrobial);
    expect(screen.queryByText(ANTIMICROBIAL_NOTE)).toBeNull();
    expect(screen.queryByRole("group", { name: "Identificação do paciente" })).toBeNull();
    emit();
    await waitFor(() => expect(onSubmit).toHaveBeenCalledTimes(1));
    expect(onSubmit.mock.calls[0][0]).not.toHaveProperty("patient_identification");
  });

  it("mistura controle especial e antimicrobiano: diz para separar e não manda", async () => {
    const onSubmit = renderForm();
    await pick(SERTRALINE);
    await pick(AMOXICILLIN_19D);
    fillItem(1, "60");
    fillItem(2, "7");
    expect(screen.getByRole("status", { name: "categoria da receita" }).textContent).toBe("categoria: misturada — separe em duas receitas");
    emit();
    expect(screen.getByRole("alert").textContent).toContain(MIXED_CATEGORIES);
    expect(onSubmit).not.toHaveBeenCalled();
  });

  it("quatro substâncias C1: diz o limite e não manda", async () => {
    const onSubmit = renderForm();
    for (const med of [ SERTRALINE, AMITRIPTYLINE, FLUOXETINE, CARBAMAZEPINE ]) await pick(med);
    [ 1, 2, 3, 4 ].forEach((n) => fillItem(n, "30"));
    fillAddress();
    emit();
    expect(screen.getByRole("alert").textContent).toBe("no máximo 3 substâncias da lista C1 por receita");
    expect(onSubmit).not.toHaveBeenCalled();
  });

  it("duração acima de 60 dias, e de 180 no anticonvulsivante", async () => {
    const onSubmit = renderForm();
    await pick(SERTRALINE);
    await pick(CARBAMAZEPINE);
    fillItem(1, "90");
    fillItem(2, "120");
    fillAddress();
    emit();
    expect(screen.getByRole("alert").textContent).toBe("item 1: duração acima de 60 dias");
    fillItem(1, "60");
    fillItem(2, "200");
    emit();
    expect(screen.getByRole("alert").textContent).toBe("item 2: duração acima de 180 dias (anticonvulsivante)");
    expect(onSubmit).not.toHaveBeenCalled();
  });

  it("A/B bloqueados com o caminho da Notificação; C2/C3/C4 fora; C1 entra", async () => {
    mocked(api.searchMedications).mockResolvedValue([ MORPHINE, CLONAZEPAM_B1, ISOTRETINOIN, SERTRALINE ]);
    renderForm();
    fireEvent.change(screen.getByLabelText("Buscar medicamento"), { target: { value: "ab" } });
    const list = await screen.findByRole("list", { name: "resultados: Buscar medicamento" });
    for (const med of [ MORPHINE, CLONAZEPAM_B1, ISOTRETINOIN ]) {
      expect((within(list).getByRole("button", { name: med.label }) as HTMLButtonElement).disabled).toBe(true);
    }
    expect(within(list).getAllByText("Notificação de receita (listas A/B) — registre a de papel em “Notificação (papel)”")).toHaveLength(2);
    expect(within(list).getByText("lista C2 (retinoide, talidomida ou antirretroviral) — fora do Rota Saúde")).not.toBeNull();
    expect((within(list).getByRole("button", { name: SERTRALINE.label }) as HTMLButtonElement).disabled).toBe(false);
    expect(within(list).getByText("controlado · lista C1")).not.toBeNull();
  });

  it("vai sair em papel: sem número, sem certificado, sem assinatura na cidade", async () => {
    mocked(api.getSncrStock).mockResolvedValue(stock({ rce: { free: 0, used: 1000, voided: 0, requests_this_month: 1 } }));
    renderForm();
    await pick(SERTRALINE);
    expect((await screen.findByRole("note", { name: PAPER })).textContent)
      .toBe("Esta receita vai sair em papel: sem número SNCR no seu estoque — obtenha números em Conta → SNCR.");
    cleanup();

    mocked(api.getSncrStock).mockResolvedValue(stock());
    mocked(api.getCurrentCertificate).mockResolvedValue(null);
    renderForm();
    await pick(SERTRALINE);
    expect((await screen.findByRole("note", { name: PAPER })).textContent)
      .toBe("Esta receita vai sair em papel: você não tem certificado digital ativo — vincule em Conta → Assinatura digital.");
    cleanup();

    renderForm({ controlled: { ...CONTEXT, signatureOn: false } });
    await pick(SERTRALINE);
    expect((await screen.findByRole("note", { name: PAPER })).textContent)
      .toBe("Esta receita vai sair em papel: a assinatura digital não está disponível na cidade.");
    // As duas primeiras telas consultaram o certificado; sem assinatura na cidade, a terceira nem consulta.
    expect(api.getCurrentCertificate).toHaveBeenCalledTimes(2);
  });

  it("com certificado e número, não prevê papel", async () => {
    renderForm();
    await pick(SERTRALINE);
    await waitFor(() => expect(api.getSncrStock).toHaveBeenCalled());
    await waitFor(() => expect(api.getCurrentCertificate).toHaveBeenCalled());
    expect(screen.queryByRole("note", { name: PAPER })).toBeNull();
  });

  it("usar a última identificação preenche o endereço; sem anterior, diz", async () => {
    renderForm();
    await pick(SERTRALINE);
    fireEvent.click(screen.getByRole("button", { name: "Usar a última identificação" }));
    const id = screen.getByRole("group", { name: "Identificação do paciente" });
    await waitFor(() => expect((within(id).getByLabelText("Logradouro") as HTMLInputElement).value).toBe("Rua Marechal Deodoro"));
    expect((within(id).getByLabelText("CEP") as HTMLInputElement).value).toBe("80010-010");
    expect(within(id).queryByLabelText(/não possui CPF/)).toBeNull();
    expect(api.getLastPatientIdentification).toHaveBeenCalledWith("cs1");
    cleanup();

    mocked(api.getLastPatientIdentification).mockResolvedValue(null);
    renderForm();
    await pick(SERTRALINE);
    fireEvent.click(screen.getByRole("button", { name: "Usar a última identificação" }));
    expect(await screen.findByText("nenhuma receita anterior deste paciente com identificação")).not.toBeNull();
  });

  it("sem endereço e telefone do prescritor: avisa e não emite (controle especial sempre; antimicrobiano quando sairia digital)", async () => {
    mocked(api.getProfessionalContact).mockResolvedValue({ address: null, phone: null });
    const onSubmit = renderForm();
    await pick(SERTRALINE);
    fillItem(1, "30");
    fillAddress();
    expect(await within(screen.getByRole("group", { name: "prescritor" })).findByText(PRESCRIBER_MISSING)).not.toBeNull();
    emit();
    expect(screen.getByRole("alert").textContent).toBe(PRESCRIBER_MISSING);
    expect(onSubmit).not.toHaveBeenCalled();
    cleanup();

    const onSubmitRet = renderForm();
    await pick(AMOXICILLIN_19D);
    fillItem(1, "7");
    expect(await within(screen.getByRole("group", { name: "prescritor" })).findByText(PRESCRIBER_MISSING)).not.toBeNull();
    await waitFor(() => expect(api.getCurrentCertificate).toHaveBeenCalled());
    emit();
    expect(screen.getByRole("alert").textContent).toBe(PRESCRIBER_MISSING);
    expect(onSubmitRet).not.toHaveBeenCalled();
    cleanup();

    // Sem certificado a RET já sairia em papel: o api não exige o contato (plano do api, D12) e a tela deixa emitir.
    mocked(api.getCurrentCertificate).mockResolvedValue(null);
    const onSubmitPaper = renderForm();
    await pick(AMOXICILLIN_19D);
    fillItem(1, "7");
    await screen.findByRole("note", { name: PAPER });
    emit();
    await waitFor(() => expect(onSubmitPaper).toHaveBeenCalledTimes(1));
  });

  it("SNCR simulado: faixa sem validade", async () => {
    renderForm({ controlled: { ...CONTEXT, simulated: true } });
    await pick(SERTRALINE);
    expect(await screen.findByText(SNCR_SIMULATED_NOTICE)).not.toBeNull();
  });

  it("recusa do api fica no formulário, com o endereço", async () => {
    renderForm({ onSubmit: vi.fn(async () => { throw new ApiError(422, { error: "too_many_c1_substances" }, "422"); }) });
    await pick(SERTRALINE);
    fillItem(1, "30");
    fillAddress();
    emit();
    expect((await screen.findByRole("alert")).textContent).toBe("no máximo 3 substâncias da lista C1 por receita");
    const id = screen.getByRole("group", { name: "Identificação do paciente" });
    expect((within(id).getByLabelText("Logradouro") as HTMLInputElement).value).toBe("Rua Marechal Deodoro");
  });

  it("cancelar e emitir outro: a identificação vem preenchida do documento cancelado", async () => {
    renderForm({ initial: {
      items: [ itemFromPrescription(prescriptionItem({ catalog_item: SERTRALINE_REF, duration_days: 60 })) ], protocolVersionId: "",
      identification: { cpf: "39053344705", address: address() }
    } });
    const id = await screen.findByRole("group", { name: "Identificação do paciente" });
    expect((within(id).getByLabelText("Complemento") as HTMLInputElement).value).toBe("apto 12");
  });

  it("sem o 19d (sem contexto): o controlado continua bloqueado como no 19c", async () => {
    mocked(api.searchMedications).mockResolvedValue([ SERTRALINE ]);
    renderForm({ controlled: null });
    fireEvent.change(screen.getByLabelText("Buscar medicamento"), { target: { value: "se" } });
    const list = await screen.findByRole("list", { name: "resultados: Buscar medicamento" });
    expect((within(list).getByRole("button", { name: SERTRALINE.label }) as HTMLButtonElement).disabled).toBe(true);
    expect(within(list).getByText(CONTROLLED_BLOCKED)).not.toBeNull();
    expect(screen.queryByRole("status", { name: "categoria da receita" })).toBeNull();
    expect(api.getSncrStock).not.toHaveBeenCalled();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod19d && npx vitest run src/modules/documents/PrescriptionForm.controlled.test.tsx`
Expected: FAIL — `PrescriptionForm` não tem a prop `controlled` (e o `ControlledContext` não existe); o controlado C1 fica bloqueado.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/documents/PatientIdentificationFields.tsx
// Identificação do paciente na receita de controle especial (módulo 19d; spec
// §5; contrato §2): o CPF é o do cadastro (a tela só mostra mascarado e nunca
// o envia — o api usa o do paciente; sem "não possui CPF", decisão do usuário)
// e o endereço completo, digitado ou o da última receita do paciente.
import type { CSSProperties } from "react";
import type { IdentificationDraft } from "../../lib/controlled";
import { secondaryButtonStyle } from "../../components/formStyles";
import { AddressFields } from "./AddressFields";
import { mutedStyle, rowStyle } from "./formStyles";

interface Props {
  value: IdentificationDraft;
  onChange(next: IdentificationDraft): void;
  cpfMasked?: string;
  onUseLast?(): void;
  lastNote?: string | null;
}

export function PatientIdentificationFields({ value, onChange, cpfMasked, onUseLast, lastNote }: Props) {
  return (
    <fieldset aria-label="Identificação do paciente" style={box}>
      <legend style={legendStyle}>Identificação do paciente (receita de controle especial)</legend>
      <p style={mutedStyle}>CPF do cadastro: <span className="mono">{cpfMasked ?? "—"}</span></p>
      {onUseLast && (
        <div style={rowStyle}>
          <button type="button" style={secondaryButtonStyle} onClick={onUseLast}>Usar a última identificação</button>
          {lastNote && <small style={mutedStyle}>{lastNote}</small>}
        </div>
      )}
      <AddressFields legend="Endereço do paciente" value={value.address} onChange={(address) => onChange({ ...value, address })} />
    </fieldset>
  );
}

const box: CSSProperties = { display: "flex", flexDirection: "column", gap: 8, margin: 0, padding: 10, border: "1px solid var(--rule)", borderRadius: 6 };
const legendStyle: CSSProperties = { fontSize: 12, fontWeight: 600, color: "var(--ink2)", padding: "0 4px" };
```

Em `src/modules/documents/PrescriptionForm.tsx` (código do 19c, Task 5 do plano do dashboard do 19c):

(a) comentário do topo — acrescente ao fim:

```tsx
// Módulo 19d (com `controlled`): C1/C5 entram (receita de controle especial,
// com identificação do paciente e contato do prescritor), A/B e C2/C3/C4 ficam
// bloqueados com o motivo, a categoria é calculada pelos itens e a tela prevê
// quando a receita vai sair em papel. Sem `controlled`, tudo como acima.
```

(b) imports — troque a linha de `../../lib/api` e acrescente as novas:

```tsx
import {
  getCurrentCertificate, getLastPatientIdentification, getProfessionalContact, getSncrStock, listNursingProtocols,
  type MedicationRoute, type PrescriptionInput
} from "../../lib/api";
import { CERTIFICATE_KEY } from "../../lib/signature";
import {
  CATEGORY_NOTE, CONTACT_KEY, PRESCRIBER_MISSING, SNCR_SIMULATED_NOTICE, SNCR_STOCK_KEY, addressLine, categoryLabel,
  controlledBlockReason, controlledProblems, formatPhone, identificationDraftFrom, identificationInput, paperForecast, paperReasonLabel,
  prescriptionCategory, type IdentificationDraft
} from "../../lib/controlled";
import { PatientIdentificationFields } from "./PatientIdentificationFields";
```

(c) `Props` e a assinatura:

```tsx
// Módulo 19d: contexto da receita de controlado (sessão e paciente). Sem ele, a receita do 19c.
export interface ControlledContext {
  userId: string | null; simulated: boolean; signatureOn: boolean; consultationId?: string; patientCpfMasked?: string;
}

interface Props {
  cboCode: string;
  unitId?: string;
  today: string;
  initial: PrescriptionDraft;
  onSubmit(content: PrescriptionInput): Promise<void>;
  onCancel(): void;
  searchDelayMs?: number;
  replacing?: boolean;
  controlled?: ControlledContext;
}

export function PrescriptionForm({ cboCode, unitId, today, initial, onSubmit, onCancel, searchDelayMs, replacing, controlled }: Props) {
```

(d) logo depois de `const chosen = …;`:

```tsx
  // ─── Módulo 19d ───
  const category = controlled ? prescriptionCategory(d.items) : "common";
  const needsSncr = category === "special_control" || category === "antimicrobial";
  const [ identification, setIdentification ] = useState<IdentificationDraft>(() => identificationDraftFrom(initial.identification));
  const [ lastNote, setLastNote ] = useState<string | null>(null);
  const contact = useQuery({ queryKey: [ CONTACT_KEY, controlled?.userId ?? null ], queryFn: getProfessionalContact,
    enabled: !!controlled && needsSncr, gcTime: 0, retry: false });
  const stock = useQuery({ queryKey: [ SNCR_STOCK_KEY, controlled?.userId ?? null ], queryFn: getSncrStock,
    enabled: !!controlled && needsSncr && !nurse, retry: false });
  const certificate = useQuery({ queryKey: [ CERTIFICATE_KEY, controlled?.userId ?? undefined ], queryFn: getCurrentCertificate,
    enabled: !!controlled && needsSncr && !nurse && controlled.signatureOn, retry: false });
  const contactMissing = contact.isSuccess && (!contact.data.address || !contact.data.phone);
  const forecast = controlled ? paperForecast({
    category, cboCode, signatureOn: controlled.signatureOn, certificate: certificate.isSuccess ? certificate.data : undefined,
    stock: stock.data
  }) : null;
  // Plano do api, D12: o contato é exigido na de controle especial e na de
  // antimicrobiano que sairia digital.
  const contactBlocks = contactMissing && (category === "special_control" || (category === "antimicrobial" && !forecast));
  const simulated = !!controlled && needsSncr && !forecast && (controlled.simulated || stock.data?.simulated === true);

  async function fillLastIdentification() {
    if (!controlled?.consultationId) return;
    setLastNote(null);
    try {
      const last = await getLastPatientIdentification(controlled.consultationId);
      if (last) setIdentification(identificationDraftFrom(last));
      else setLastNote("nenhuma receita anterior deste paciente com identificação");
    } catch (err) {
      setLastNote(documentError(err));
    }
  }
```

(e) em `submit`, troque `const found = prescriptionProblems(d, nurse);` e o `await onSubmit(prescriptionInput(d));` por:

```tsx
    const found = [ ...prescriptionProblems(d, nurse, !!controlled), ...(controlled ? controlledProblems(d, identification) : []) ];
    if (controlled && contactBlocks) found.push(PRESCRIBER_MISSING);
```

```tsx
      const content = prescriptionInput(d);
      await onSubmit(controlled && category === "special_control"
        ? { ...content, patient_identification: identificationInput(identification) } : content);
```

(f) a busca do médico e do dentista:

```tsx
          <MedicationSearch label="Buscar medicamento" unitId={unitId} blockControlled delayMs={searchDelayMs}
            blockReason={controlled ? controlledBlockReason : undefined} onPick={(m) => addItem(itemFromMedication(m))} />
```

(g) troque a linha do aviso de antimicrobiano do 19c (`{hasAntimicrobial(d) && <p role="note" …>{ANTIMICROBIAL_NOTE}</p>}`) por:

```tsx
      {!controlled && hasAntimicrobial(d) && <p role="note" style={{ ...mutedStyle, fontWeight: 600, color: "var(--warn)" }}>{ANTIMICROBIAL_NOTE}</p>}
      {controlled && d.items.length > 0 && (
        <p role="status" aria-label="categoria da receita" style={mutedStyle}>
          {category === "mixed" ? "categoria: misturada — separe em duas receitas" : `categoria: ${categoryLabel(category)}`}
        </p>
      )}
      {controlled && (category === "special_control" || category === "antimicrobial") && (
        <p role="note" aria-label="regras da categoria" style={{ ...mutedStyle, fontWeight: 600 }}>{CATEGORY_NOTE[category]}</p>
      )}
      {forecast && (
        <p role="note" aria-label="vai sair em papel" style={{ ...mutedStyle, fontWeight: 600, color: "var(--warn)" }}>
          {`Esta receita vai sair em papel: ${paperReasonLabel(forecast)}.`}
        </p>
      )}
      {simulated && <div><Tag tone="warn" mono={false}>{SNCR_SIMULATED_NOTICE}</Tag></div>}
```

(h) logo depois do bloco da lista de itens (`{d.items.length === 0 ? … : ( <ol …> … </ol> )}`) e antes da lista de problemas:

```tsx
      {controlled && category === "special_control" && (
        <PatientIdentificationFields value={identification} onChange={setIdentification} cpfMasked={controlled.patientCpfMasked}
          onUseLast={controlled.consultationId ? () => void fillLastIdentification() : undefined} lastNote={lastNote} />
      )}
      {controlled && needsSncr && (
        <div role="group" aria-label="prescritor" style={{ display: "flex", flexDirection: "column", gap: 2 }}>
          <strong style={{ fontSize: 12 }}>Prescritor (endereço e telefone do seu perfil)</strong>
          {contact.isError && <p role="alert" style={alertStyle}>{documentError(contact.error)}</p>}
          {contact.isSuccess && !contactMissing && (
            <p style={mutedStyle}>{`${addressLine(contact.data.address)} · ${formatPhone(contact.data.phone)}`}</p>
          )}
          {contactBlocks && (
            <p role="note" style={{ ...mutedStyle, fontWeight: 600, color: "var(--warn)" }}>{PRESCRIBER_MISSING}</p>
          )}
        </div>
      )}
```

(i) em `ItemFields`, logo depois do selo de antimicrobiano (`{item.medication?.antimicrobial && <Tag …>antimicrobiano</Tag>}`):

```tsx
        {item.medication?.controlled_list && <Tag tone="warn" mono={false}>{`lista ${item.medication.controlled_list}`}</Tag>}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod19d && npx vitest run src/modules/documents/PrescriptionForm.controlled.test.tsx src/modules/documents/PrescriptionForm.test.tsx && npx tsc --noEmit`
Expected: PASS (os testes do 19c da receita continuam: eles não passam `controlled`) e `tsc` sem erro. No teste "sem endereço e telefone do prescritor", a lista `role="alert"` traz só `PRESCRIBER_MISSING` porque os itens e o endereço do paciente estão completos; se aparecer outra linha, confira o `fillItem`.

- [ ] **Step 5: Commit**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/dashboard/.claude/mod19d
/opt/homebrew/bin/git add src/modules/documents/PatientIdentificationFields.tsx src/modules/documents/PrescriptionForm.tsx \
  src/modules/documents/PrescriptionForm.controlled.test.tsx
/opt/homebrew/bin/git commit -m "feat: compute the prescription category and collect special control data in the prescription form

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 8: Aba Documentos — Notificação, aviso da emissão, categoria e número na lista, cancelamento

**Files:**
- Modify: `src/modules/documents/ConsultationDocuments.tsx`, `src/modules/documents/CancelDocument.tsx`, `src/modules/consultation/ConsultationWorkspace.tsx`
- Test: `src/modules/documents/ConsultationDocuments.controlled.test.tsx`

**Interfaces:**
- Consumes: `recordNotification`, `NotificationRecordInput` (Task 1); `SNCR_STOCK_KEY`, `SNCR_SIMULATED_NOTICE`, `SNCR_NO_CANCEL_NOTE`, `controlledIssuedNotice`, `controlledOn`, `documentSncr`, `documentTitle`, `paperReasonLabel`, `prescribesControlled`, `sncrLabel`, `usesSimulatedSncr` (Task 2); `NotificationRecordForm` (Task 6); `PrescriptionForm` com `controlled` e `ControlledContext` (Task 7); `hasFeature` (`features.ts`); `MEDICATIONS_KEY`, `withDocument` (19c).
- Produces:
  - `IssueContext` ganha `patientCpfMasked?: string` (o `ConsultationWorkspace` passa o do prontuário); o `ControlledContext` leva o `consultationId` (para "Usar a última identificação");
  - com `controlled_prescriptions`: o botão "Notificação (papel)" (médico e dentista) abre o `NotificationRecordForm`; a frase depois de emitir é `controlledIssuedNotice` (digital com o número, papel com o `paper_reason`); emitir uma receita numerada, ou cancelar uma, relê o estoque (`[ SNCR_STOCK_KEY ]`); registrar a Notificação relê os medicamentos em uso;
  - na lista: o título da receita pela categoria ("Receita de controle especial", "Receita de antimicrobiano"), o número (`RCE nº …`), a faixa `SNCR_SIMULATED_NOTICE` quando simulado, o motivo do papel na coluna "Modo"; o registro da Notificação sem "Imprimir" nem "Cancelar e emitir outro" e com "—" no código;
  - `CancelDocument` avisa `SNCR_NO_CANCEL_NOTE` quando o documento tem número SNCR e o `CANCEL_WARNING` também no registro da Notificação.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/documents/ConsultationDocuments.controlled.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), listConsultationDocuments: vi.fn(), issueDocument: vi.fn(), recordNotification: vi.fn(),
    cancelDocument: vi.fn(), getPrescriptionRenewal: vi.fn(), fetchDocumentPdf: vi.fn(), searchMedications: vi.fn(), searchTerminology: vi.fn(),
    listNursingProtocols: vi.fn(), getProfessionalContact: vi.fn(), getSncrStock: vi.fn(), getCurrentCertificate: vi.fn(),
    getLastPatientIdentification: vi.fn() };
});

import * as api from "../../lib/api";
import { MEDICATIONS_KEY } from "../../lib/clinicalDocuments";
import { SNCR_NO_CANCEL_NOTE, SNCR_SIMULATED_NOTICE, SNCR_STOCK_KEY } from "../../lib/controlled";
import { ConsultationDocuments } from "./ConsultationDocuments";
import { renderWithProviders } from "../../test/campaignFixtures";
import { prescriptionDoc, prescriptionItem } from "../../test/documentFixtures";
import { certificate } from "../../test/signatureFixtures";
import {
  CLONAZEPAM_B1, NOW19D, SERTRALINE_REF, contact, controlledUser, notificationDoc, rceDoc, stock
} from "../../test/controlledFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const SIMULATED_RCE = { sncr: { kind: "rce" as const, number: "2610.1-41.0001234", simulated: true } };
const ADDRESS_BODY = { street: "Rua Marechal Deodoro", number: "630", district: "Centro", city: "Curitiba", uf: "PR", zip: "80010010" };

function renderDocs(cboCode = "225142") {
  return renderWithProviders(
    <ConsultationDocuments consultationId="cs1" searchDelayMs={0}
      issue={{ cboCode, unitId: "u1", patientCpfMasked: "***.533.447-**" }} />
  );
}

function fillAddress() {
  const w = within(screen.getByRole("group", { name: "Identificação do paciente" }));
  fireEvent.change(w.getByLabelText("Logradouro"), { target: { value: "Rua Marechal Deodoro" } });
  fireEvent.change(w.getByLabelText("Número"), { target: { value: "630" } });
  fireEvent.change(w.getByLabelText("Bairro"), { target: { value: "Centro" } });
  fireEvent.change(w.getByLabelText("Cidade"), { target: { value: "Curitiba" } });
  fireEvent.change(w.getByLabelText("UF"), { target: { value: "PR" } });
  fireEvent.change(w.getByLabelText("CEP"), { target: { value: "80010010" } });
}

async function renewSertraline() {
  mocked(api.getPrescriptionRenewal).mockResolvedValue([ prescriptionItem({ catalog_item: SERTRALINE_REF, quantity: 60,
    dosage_instructions: "1 comprimido pela manhã", duration_days: 60, continuous: true }) ]);
  fireEvent.click(await screen.findByRole("button", { name: "Renovar receita" }));
  await screen.findByRole("group", { name: "Identificação do paciente" });
  fillAddress();
}

afterEach(() => { cleanup(); vi.useRealTimers(); });

describe("ConsultationDocuments — controlado e SNCR (19d)", () => {
  beforeEach(() => {
    vi.clearAllMocks();
    vi.useFakeTimers({ toFake: [ "Date" ] });
    vi.setSystemTime(new Date(NOW19D));
    mocked(api.fetchCurrentSession).mockResolvedValue(controlledUser());
    mocked(api.listConsultationDocuments).mockResolvedValue([]);
    mocked(api.listNursingProtocols).mockResolvedValue([]);
    mocked(api.getProfessionalContact).mockResolvedValue(contact());
    mocked(api.getSncrStock).mockResolvedValue(stock());
    mocked(api.getCurrentCertificate).mockResolvedValue(certificate());
    mocked(api.getLastPatientIdentification).mockResolvedValue(null);
  });

  it("lista: RCE com título, número e faixa de simulado; Notificação sem Imprimir nem 'emitir outro'", async () => {
    mocked(api.listConsultationDocuments).mockResolvedValue([ rceDoc({}, SIMULATED_RCE), notificationDoc() ]);
    renderDocs();
    expect(await screen.findByText("Receita de controle especial")).not.toBeNull();
    expect(screen.getByText("RCE nº 2610.1-41.0001234")).not.toBeNull();
    expect(screen.getByText(SNCR_SIMULATED_NOTICE)).not.toBeNull();
    expect(screen.getByText("Notificação de receita (papel)")).not.toBeNull();
    expect(await screen.findByRole("button", { name: /^Cancelar Notificação de receita \(papel\) de/ })).not.toBeNull();
    expect(screen.queryByRole("button", { name: /^Imprimir Notificação/ })).toBeNull();
    expect(screen.queryByRole("button", { name: /^Cancelar e emitir outro: Notificação/ })).toBeNull();
    expect(screen.getByRole("button", { name: /^Imprimir Receita de/ })).not.toBeNull();
  });

  it("renovar a RCE: emite digital, diz o número (simulado) e relê o estoque", async () => {
    mocked(api.issueDocument).mockResolvedValue(rceDoc({}, SIMULATED_RCE));
    const { client } = renderDocs();
    const invalidate = vi.spyOn(client, "invalidateQueries");
    await renewSertraline();
    fireEvent.click(screen.getByRole("button", { name: "Emitir receita" }));
    expect(await screen.findByText("Receita de controle especial emitida em modo digital com o número SNCR RCE nº 2610.1-41.0001234 " +
      "(numeração simulada — sem validade): a assinatura entra na sua fila; o PDF assinado sai depois dela.")).not.toBeNull();
    expect(api.issueDocument).toHaveBeenCalledWith("cs1", { kind: "prescription", content: {
      items: [ { catalog_item: { id: "ci10" }, quantity: 60, quantity_unit: "comprimido", route: "oral",
        dosage_instructions: "1 comprimido pela manhã", duration_days: 60, continuous: true } ],
      patient_identification: { address: ADDRESS_BODY }
    } }, undefined);
    expect(invalidate).toHaveBeenCalledWith({ queryKey: [ SNCR_STOCK_KEY ] });
  });

  it("previsão digital, mas o api devolve papel (no_sncr_number): diz o motivo e manda imprimir 2 vias", async () => {
    mocked(api.issueDocument).mockResolvedValue(rceDoc({ issue_mode: "paper", paper_reason: "no_sncr_number", signature: null }, { sncr: null }));
    renderDocs();
    await renewSertraline();
    await waitFor(() => expect(api.getSncrStock).toHaveBeenCalled());
    expect(screen.queryByRole("note", { name: "vai sair em papel" })).toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Emitir receita" }));
    const notice = await screen.findByText("Receita de controle especial emitida em modo papel (sem número SNCR no seu estoque — " +
      "obtenha números em Conta → SNCR): imprima as 2 vias e assine à mão.");
    expect(notice.textContent).not.toMatch(/assinatura entra na sua fila/);
    expect(screen.getByText("sem número SNCR no seu estoque — obtenha números em Conta → SNCR")).not.toBeNull();
  });

  it("Notificação (papel): registra, diz e relê os medicamentos em uso", async () => {
    mocked(api.searchMedications).mockResolvedValue([ CLONAZEPAM_B1 ]);
    mocked(api.recordNotification).mockResolvedValue(notificationDoc());
    const { client } = renderDocs();
    const invalidate = vi.spyOn(client, "invalidateQueries");
    fireEvent.click(await screen.findByRole("button", { name: "Notificação (papel)" }));
    const form = screen.getByRole("form", { name: "Notificação de papel" });
    fireEvent.change(within(form).getByLabelText("Buscar medicamento (listas A e B)"), { target: { value: "clona" } });
    fireEvent.click(await within(form).findByRole("button", { name: CLONAZEPAM_B1.label }));
    fireEvent.change(within(form).getByLabelText("Número do talão"), { target: { value: "PR0012345" } });
    fireEvent.change(within(form).getByLabelText("Quantidade"), { target: { value: "30" } });
    fireEvent.change(within(form).getByLabelText("Via"), { target: { value: "oral" } });
    fireEvent.change(within(form).getByLabelText("Duração em dias"), { target: { value: "30" } });
    fireEvent.change(within(form).getByLabelText("Posologia"), { target: { value: "1 comprimido à noite" } });
    fireEvent.click(within(form).getByRole("button", { name: "Registrar Notificação" }));

    await waitFor(() => expect(api.recordNotification).toHaveBeenCalledWith("cs1", {
      notification_type: "B", paper_number: "PR0012345", numbering_uf: "PR",
      item: { catalog_item: { id: "ci3" }, quantity: 30, quantity_unit: "comprimido", route: "oral",
        dosage_instructions: "1 comprimido à noite", duration_days: 30 }
    }));
    expect(await screen.findByText("Notificação de papel registrada: o medicamento entra na lista de medicamentos em uso. " +
      "Guarde o talão; nada é impresso aqui.")).not.toBeNull();
    expect(invalidate).toHaveBeenCalledWith({ queryKey: [ MEDICATIONS_KEY ] });
    expect(screen.queryByRole("form", { name: "Notificação de papel" })).toBeNull();
  });

  it("sem o interruptor, ou para a enfermagem: sem 'Notificação (papel)'", async () => {
    mocked(api.listConsultationDocuments).mockResolvedValue([ prescriptionDoc() ]);
    mocked(api.fetchCurrentSession).mockResolvedValue(controlledUser([ "health_professional" ], { features: [ "clinical_record", "clinical_documents" ] }));
    renderDocs();
    // O "Cancelar" da autora só aparece com a sessão carregada.
    expect(await screen.findByRole("button", { name: /^Cancelar Receita de/ })).not.toBeNull();
    expect(screen.queryByRole("button", { name: "Notificação (papel)" })).toBeNull();
    cleanup();

    mocked(api.fetchCurrentSession).mockResolvedValue(controlledUser());
    renderDocs("223565");
    expect(await screen.findByRole("button", { name: /^Cancelar Receita de/ })).not.toBeNull();
    expect(screen.queryByRole("button", { name: "Notificação (papel)" })).toBeNull();
  });

  it("cancelar RCE: avisa que o SNCR não cancela e relê o estoque", async () => {
    mocked(api.listConsultationDocuments).mockResolvedValue([ rceDoc() ]);
    mocked(api.cancelDocument).mockResolvedValue(rceDoc({ status: "cancelled", cancelled_at: "2026-10-10T10:00:00-03:00",
      cancel_reason: "dose errada na receita" }));
    const { client } = renderDocs();
    const invalidate = vi.spyOn(client, "invalidateQueries");
    fireEvent.click(await screen.findByRole("button", { name: /^Cancelar Receita de/ }));
    expect(await screen.findByText(SNCR_NO_CANCEL_NOTE)).not.toBeNull();
    fireEvent.change(screen.getByLabelText("Motivo do cancelamento"), { target: { value: "dose errada na receita" } });
    fireEvent.click(screen.getByRole("button", { name: "Confirmar cancelamento" }));
    await waitFor(() => expect(api.cancelDocument).toHaveBeenCalledWith("doc20", "dose errada na receita"));
    await waitFor(() => expect(invalidate).toHaveBeenCalledWith({ queryKey: [ SNCR_STOCK_KEY ] }));
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod19d && npx vitest run src/modules/documents/ConsultationDocuments.controlled.test.tsx`
Expected: FAIL — a lista mostra "Receita" sem número, não há "Notificação (papel)", a frase da emissão é a do 19c e o diálogo de cancelamento não fala do SNCR.

- [ ] **Step 3: Write minimal implementation**

Em `src/modules/documents/ConsultationDocuments.tsx` (código do 19c, Task 6 do plano do dashboard do 19c):

(a) comentário do topo — acrescente ao fim:

```tsx
// Módulo 19d (com `controlled_prescriptions`): "Notificação (papel)" para médico
// e dentista; a receita de controle especial e a de antimicrobiano com número
// SNCR; a frase depois de emitir diz o número ou por que saiu em papel; emitir
// ou cancelar uma receita numerada relê o estoque do SNCR.
```

(b) imports — troque a linha de `../../lib/api`, **tire `issuedNotice`** do import de `../../lib/clinicalDocuments` (passa a ser usado por `controlledIssuedNotice`) e acrescente:

```tsx
import {
  getPrescriptionRenewal, issueDocument, listConsultationDocuments, recordNotification,
  type ClinicalDocument, type DocumentInput, type DocumentKind, type NotificationRecordInput
} from "../../lib/api";
import { hasFeature } from "../../lib/features";
import {
  SNCR_SIMULATED_NOTICE, SNCR_STOCK_KEY, controlledIssuedNotice, controlledOn, documentSncr, documentTitle, paperReasonLabel,
  prescribesControlled, sncrLabel, usesSimulatedSncr
} from "../../lib/controlled";
import { Tag } from "../../components/Tag";
import { NotificationRecordForm } from "./NotificationRecordForm";
import type { ControlledContext } from "./PrescriptionForm";
```

(c) `IssueContext`:

```tsx
// patientCpfMasked: módulo 19d (identificação do paciente na receita de controle especial).
export interface IssueContext { cboCode: string; unitId?: string; patientCpfMasked?: string }
```

(d) depois de `const kinds = issue ? kindsFor(issue.cboCode) : [];`:

```tsx
  // Módulo 19d.
  const controlled = controlledOn(user);
  const canNotify = !!issue && controlled && prescribesControlled(issue.cboCode);
  const [ notifying, setNotifying ] = useState(false);
  const cityUf = user?.memberships?.find((m) => m.city_uf)?.city_uf ?? "";
  const controlledContext: ControlledContext | undefined = issue && controlled ? {
    userId: user?.id ?? null, simulated: usesSimulatedSncr(user), signatureOn: hasFeature(user, "digital_signature"),
    consultationId, patientCpfMasked: issue.patientCpfMasked
  } : undefined;
```

(e) em `open(next)` e em `askCancel(doc, reissue)`, acrescente `setNotifying(false);` na primeira linha de cada uma; e acrescente, depois de `askCancel`:

```tsx
  function startNotification() {
    setNotice(null); setError(null); setCancelling(null); setForm(null); setNotifying(true);
  }

  // A recusa sobe para o formulário, que a mostra e fica preenchido.
  async function recordIt(content: NotificationRecordInput) {
    const doc = await recordNotification(consultationId, content);
    queryClient.setQueryData<ClinicalDocument[]>(key, (list) => withDocument(list ?? [], doc));
    void queryClient.invalidateQueries({ queryKey: [ MEDICATIONS_KEY ] });
    setNotifying(false);
    setNotice(controlledIssuedNotice(doc));
  }
```

(f) em `issueIt`, troque `setNotice(issuedNotice(doc));` por:

```tsx
    if (documentSncr(doc)) void queryClient.invalidateQueries({ queryKey: [ SNCR_STOCK_KEY ] });
    setNotice(controlledIssuedNotice(doc));
```

(g) nas colunas, troque a de "Documento", a de "Modo" e a de "Código":

```tsx
    { label: "Documento", w: "1.4fr", render: (d) => {
      const sncr = documentSncr(d);
      return (
        <span style={cell}>
          <span>{`${documentTitle(d)}${d.replaces_document_id ? " · substitui um cancelado" : ""}`}</span>
          {sncr && <span className="mono" style={muted}>{sncrLabel(sncr)}</span>}
          {sncr?.simulated && <Tag tone="warn" mono={false}>{SNCR_SIMULATED_NOTICE}</Tag>}
        </span>
      );
    } },
```

```tsx
    { label: "Modo", w: "0.6fr", render: (d) => d.paper_reason ? (
      <span style={cell}>
        <span>{issueModeLabel(d.issue_mode)}</span>
        <span style={muted}>{paperReasonLabel(d.paper_reason)}</span>
      </span>
    ) : issueModeLabel(d.issue_mode) },
```

```tsx
    { label: "Código", w: "1fr", render: (d) => <span className="mono">{d.short_code ?? "—"}</span> },
```

e, na coluna de ações, as condições de "Imprimir" e de "Cancelar e emitir outro" (o registro da Notificação não tem PDF nem substituto):

```tsx
        {d.status === "issued" && d.kind !== "controlled_notification_record" && (
```

```tsx
        {issue && canCancel(d, user?.id) && d.kind !== "controlled_notification_record" && (
```

(h) no grupo "emitir documento", depois do botão "Renovar receita":

```tsx
          {canNotify && (
            <button type="button" style={secondaryButtonStyle} onClick={startNotification}>Notificação (papel)</button>
          )}
```

(i) passe o contexto à receita e desenhe o formulário da Notificação, logo depois do `ExamRequisitionForm`:

```tsx
      {issue && form?.kind === "prescription" && (
        <PrescriptionForm cboCode={issue.cboCode} unitId={issue.unitId} today={today} initial={form.draft} replacing={!!form.replaces}
          searchDelayMs={searchDelayMs} controlled={controlledContext} onCancel={() => setForm(null)}
          onSubmit={(content) => issueIt({ kind: "prescription", content })} />
      )}
```

```tsx
      {issue && notifying && (
        <NotificationRecordForm unitId={issue.unitId} defaultUf={cityUf} searchDelayMs={searchDelayMs}
          onCancel={() => setNotifying(false)} onSubmit={recordIt} />
      )}
```

(j) no `onDone` do `CancelDocument`, logo depois do `setQueryData`:

```tsx
            if (documentSncr(cancelledDoc)) void queryClient.invalidateQueries({ queryKey: [ SNCR_STOCK_KEY ] });
```

(k) no fim do arquivo:

```tsx
const cell: CSSProperties = { display: "flex", flexDirection: "column", gap: 2, alignItems: "flex-start" };
```

Em `src/modules/documents/CancelDocument.tsx`:

(a) import:

```tsx
import { SNCR_NO_CANCEL_NOTE, documentSncr } from "../../lib/controlled";
```

(b) troque a linha do aviso da receita (`{doc.kind === "prescription" && <p …>{CANCEL_WARNING}</p>}`) por:

```tsx
          {(doc.kind === "prescription" || doc.kind === "controlled_notification_record") && (
            <p style={{ ...text, fontWeight: 600 }}>{CANCEL_WARNING}</p>
          )}
          {documentSncr(doc) && <p style={{ ...text, fontWeight: 600 }}>{SNCR_NO_CANCEL_NOTE}</p>}
```

Em `src/modules/consultation/ConsultationWorkspace.tsx`, na linha do `issue=` da `ConsultationDocuments`:

```tsx
              issue={user?.id === consultation.author.id ? {
                cboCode: consultation.cbo_code, unitId: props.unit.id, patientCpfMasked: record.data.patient.cpf_masked
              } : undefined} />
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod19d && npx vitest run src/modules/documents src/modules/consultation src/modules/MyConsultations.documents.test.tsx && npx tsc --noEmit`
Expected: PASS — o teste novo e os do 19c (lista, emissão, renovação, cancelamento, "Cancelar e emitir outro", Minhas consultas e o encaixe na consulta: as sessões deles não têm `controlled_prescriptions`, e para documentos sem categoria o título, a frase e as colunas são as mesmas) — e `tsc` sem erro.

- [ ] **Step 5: Commit**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/dashboard/.claude/mod19d
/opt/homebrew/bin/git add src/modules/documents/ConsultationDocuments.tsx src/modules/documents/CancelDocument.tsx \
  src/modules/consultation/ConsultationWorkspace.tsx src/modules/documents/ConsultationDocuments.controlled.test.tsx
/opt/homebrew/bin/git commit -m "feat: record paper notifications and show SNCR numbers, categories and paper reasons in the documents tab

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 9: Painel do SNCR do admin

**Files:**
- Create: `src/modules/SncrOverview.tsx`
- Modify: `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/App.tsx`
- Test: `src/modules/SncrOverview.test.tsx`

**Interfaces:**
- Consumes: `getSncrOverview`, `SncrOverviewProfessional` (Task 1); `SNCR_OVERVIEW_KEY`, `SNCR_SIMULATED_NOTICE`, `CONTROLLED_DISABLED_TEXT`, `canSeeSncrOverview`, `controlledOn`, `controlledError`, `overviewSummary`, `sortOverview`, `usesSimulatedSncr` (Task 2); `fmtNumber`, `fmtDateTime`; `PageHeader`, `Panel`, `KeyValue`, `DataTable`, `Tag`, `EmptyState`.
- Produces:
  - `SncrOverview()` — `PageHeader` "Painel do SNCR" (sub "cidade · números de receita da Anvisa · só leitura"); `Panel` "Prescritores" com "Prescritores", "Com saldo baixo", "Nunca pediram" e a tabela (Profissional, RCE livres, RET livres, Saldo — `saldo baixo`/`ok` —, Último pedido — data ou "nunca pediu"), saldo baixo primeiro; a faixa `SNCR_SIMULATED_NOTICE`; nenhum CPF, número SNCR ou dado de paciente;
  - `ModuleId` `"sncr-overview"`, item "Painel do SNCR" (ícone `№`) no grupo **Equipe**, depois de "Painel de assinatura", visível com `canSeeSncrOverview(user)`; `App` desenha `<SncrOverview />`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/SncrOverview.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, screen } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getSncrOverview: vi.fn() };
});

import * as api from "../lib/api";
import { ApiError } from "../lib/api";
import { SncrOverview } from "./SncrOverview";
import { renderWithProviders } from "../test/campaignFixtures";
import { controlledUser, sncrOverview } from "../test/controlledFixtures";
import { CONTROLLED_DISABLED_TEXT, SNCR_SIMULATED_NOTICE } from "../lib/controlled";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

afterEach(cleanup);

describe("SncrOverview", () => {
  beforeEach(() => {
    vi.clearAllMocks();
    mocked(api.fetchCurrentSession).mockResolvedValue(controlledUser([ "municipal_admin" ]));
    mocked(api.getSncrOverview).mockResolvedValue(sncrOverview());
  });

  it("saldo por prescritor, saldo baixo primeiro, quem nunca pediu e as contagens", async () => {
    renderWithProviders(<SncrOverview />);
    expect(await screen.findByText("Dra. Bruna Lopes")).not.toBeNull();
    expect(screen.getAllByText(/^Dra?\. /).map((e) => e.textContent)).toEqual([ "Dra. Bruna Lopes", "Dra. Helena Prado", "Dr. Caio Mendes" ]);
    expect(screen.getAllByText("saldo baixo")).toHaveLength(2);
    expect(screen.getByText("nunca pediu")).not.toBeNull();
    expect(screen.getByText("812")).not.toBeNull();
    expect(screen.getByText("Com saldo baixo")).not.toBeNull();
    expect(screen.getByText("Nunca pediram")).not.toBeNull();
    expect(document.body.textContent).not.toMatch(/undefined|null|Numeração simulada/);
  });

  it("SNCR simulado: faixa sem validade", async () => {
    mocked(api.getSncrOverview).mockResolvedValue(sncrOverview({ simulated: true }));
    renderWithProviders(<SncrOverview />);
    expect(await screen.findByText(SNCR_SIMULATED_NOTICE)).not.toBeNull();
  });

  it("quem não é admin, ou cidade sem o interruptor: diz, sem consultar", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(controlledUser());
    renderWithProviders(<SncrOverview />);
    expect(await screen.findByText("só o administrador municipal vê este painel")).not.toBeNull();
    cleanup();
    mocked(api.fetchCurrentSession).mockResolvedValue(controlledUser([ "municipal_admin" ], { features: [ "clinical_documents" ] }));
    renderWithProviders(<SncrOverview />);
    expect(await screen.findByText(CONTROLLED_DISABLED_TEXT)).not.toBeNull();
    expect(api.getSncrOverview).not.toHaveBeenCalled();
  });

  it("interruptor desligado depois de abrir (403): diz", async () => {
    mocked(api.getSncrOverview).mockRejectedValue(new ApiError(403, { error: "feature_disabled", feature: "controlled_prescriptions" }, "403"));
    renderWithProviders(<SncrOverview />);
    expect((await screen.findByRole("alert")).textContent).toBe(CONTROLLED_DISABLED_TEXT);
  });
});
```

Em `src/shell/modules.test.ts`, dentro do `describe("navGroupsFor")`, no fim:

```ts
    it("Equipe → Painel do SNCR: só municipal_admin com controlled_prescriptions", () => {
      const ids = (u: Parameters<typeof navGroupsFor>[0]) => navGroupsFor(u).flatMap((g) => g.items.map((i) => i.id));
      const admin = { operator: false, memberships: [ { role: "municipal_admin" } ] };
      expect(ids({ ...admin, features: [ "clinical_documents", "controlled_prescriptions" ] })).toContain("sncr-overview");
      expect(ids({ ...admin, features: [ "clinical_documents" ] })).not.toContain("sncr-overview");
      expect(ids({ operator: false, memberships: [ { role: "health_professional" } ], features: [ "clinical_documents", "controlled_prescriptions" ] }))
        .not.toContain("sncr-overview");
      expect(labelFor("sncr-overview")).toBe("Painel do SNCR");
    });
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod19d && npx vitest run src/modules/SncrOverview.test.tsx src/shell/modules.test.ts`
Expected: FAIL — `./SncrOverview` não existe e `"sncr-overview"` não está no catálogo.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/SncrOverview.tsx
// Painel do SNCR do municipal_admin (módulo 19d, F-19.29; spec §7; contrato
// §6): o saldo de números SNCR de cada prescritor (RCE e RET), quem está com
// saldo baixo e quem nunca pediu. Só leitura. Nenhum número SNCR, CPF ou dado
// de paciente: o admin vê só contagens.
import type { CSSProperties } from "react";
import { useQuery } from "@tanstack/react-query";
import { getSncrOverview, type SncrOverviewProfessional } from "../lib/api";
import { useAuth } from "../lib/auth";
import { fmtDateTime, fmtNumber } from "../lib/format";
import {
  CONTROLLED_DISABLED_TEXT, SNCR_OVERVIEW_KEY, SNCR_SIMULATED_NOTICE, canSeeSncrOverview, controlledError, controlledOn, overviewSummary,
  sortOverview, usesSimulatedSncr
} from "../lib/controlled";
import { PageHeader } from "../components/PageHeader";
import { Panel } from "../components/Panel";
import { KeyValue } from "../components/KeyValue";
import { DataTable, type Column } from "../components/DataTable";
import { Tag } from "../components/Tag";
import { EmptyState } from "../components/EmptyState";

const SUB = "cidade · números de receita da Anvisa · só leitura";

export function SncrOverview() {
  const { user } = useAuth();
  const allowed = canSeeSncrOverview(user);
  const query = useQuery({ queryKey: [ SNCR_OVERVIEW_KEY ], queryFn: getSncrOverview, enabled: allowed, retry: false });

  if (!allowed) {
    return (
      <div style={page}>
        <PageHeader title="Painel do SNCR" sub={SUB} />
        <EmptyState title={controlledOn(user) ? "só o administrador municipal vê este painel" : CONTROLLED_DISABLED_TEXT} />
      </div>
    );
  }

  const data = query.data;
  const summary = data ? overviewSummary(data) : null;
  const cols: Column<SncrOverviewProfessional>[] = [
    { label: "Profissional", w: "2fr", render: (p) => p.name },
    { label: "RCE livres", w: "1fr", align: "right", render: (p) => fmtNumber(p.rce_free) },
    { label: "RET livres", w: "1fr", align: "right", render: (p) => fmtNumber(p.ret_free) },
    { label: "Saldo", w: "1fr", render: (p) => p.low
      ? <Tag tone="warn" mono={false}>saldo baixo</Tag>
      : <Tag tone="ok" mono={false}>ok</Tag> },
    { label: "Último pedido", w: "1.4fr", render: (p) => (p.last_request_at ? fmtDateTime(p.last_request_at) : "nunca pediu") }
  ];

  return (
    <div style={page}>
      <PageHeader title="Painel do SNCR" sub={SUB} />
      {(usesSimulatedSncr(user) || data?.simulated === true) && <div><Tag tone="warn" mono={false}>{SNCR_SIMULATED_NOTICE}</Tag></div>}
      {query.isPending && <p style={muted}>carregando…</p>}
      {query.isError && <p role="alert" style={alert}>{controlledError(query.error)}</p>}
      {data && summary && (
        <Panel title="Prescritores" sub="números livres de cada médico e dentista; saldo baixo segundo o api (Divergência D11)">
          <div style={body}>
            <div style={grid}>
              <KeyValue k="Prescritores" v={String(summary.professionals)} />
              <KeyValue k="Com saldo baixo" v={String(summary.low)} />
              <KeyValue k="Nunca pediram" v={String(summary.neverRequested)} />
            </div>
            <DataTable cols={cols} rows={sortOverview(data.professionals)} rowKey={(p) => p.user_id}
              empty="nenhum médico ou dentista com números do SNCR" />
          </div>
        </Panel>
      )}
    </div>
  );
}

const page: CSSProperties = { display: "flex", flexDirection: "column", gap: 16 };
const body: CSSProperties = { display: "flex", flexDirection: "column", gap: 12 };
const grid: CSSProperties = { display: "grid", gridTemplateColumns: "repeat(auto-fit, minmax(160px, 1fr))", gap: 12 };
const muted: CSSProperties = { margin: 0, fontSize: 12, color: "var(--ink3)" };
const alert: CSSProperties = { margin: 0, fontSize: 12.5, color: "var(--down)" };
```

Troque o `sub` do `Panel` por `"números livres de cada médico e dentista; saldo baixo = menos de 50 números livres de um dos tipos"` quando o plano do api confirmar a regra da Divergência D11 (até lá, o texto acima não promete o número).

Em `src/shell/modules.ts`:

(a) import: `import { canRequestSncr, canSeeSncrOverview } from "../lib/controlled";` (no lugar do import da Task 4);

(b) no `ModuleId`, acrescente `| "sncr-overview"` ao lado de `"sncr"`;

(c) no grupo **Equipe**, depois de `signature-overview`:

```ts
    { id: "signature-overview", label: "Painel de assinatura", icon: "✍" },
    { id: "sncr-overview", label: "Painel do SNCR", icon: "№" }
```

(d) no filtro dos itens, depois da linha de `sncr`:

```ts
      // F-19.29: painel só leitura do municipal_admin, com controlled_prescriptions.
      if (item.id === "sncr-overview") return canSeeSncrOverview(user);
```

Em `src/App.tsx`: `import { SncrOverview } from "./modules/SncrOverview";` e, no `switch`, depois de `case "sncr": …`:

```tsx
    case "sncr-overview":  return <SncrOverview />;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod19d && npx vitest run src/modules/SncrOverview.test.tsx src/shell/modules.test.ts && npx tsc --noEmit`
Expected: PASS (o grupo Equipe já era só do admin; o item novo some sem o interruptor) e `tsc` sem erro.

- [ ] **Step 5: Commit**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/dashboard/.claude/mod19d
/opt/homebrew/bin/git add src/modules/SncrOverview.tsx src/modules/SncrOverview.test.tsx src/shell/modules.ts src/shell/modules.test.ts src/App.tsx
/opt/homebrew/bin/git commit -m "feat: add the municipal admin SNCR panel with per-prescriber number balance

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 10: Suíte, build, revisão e prova no navegador

**Files:** nenhum novo.

**Interfaces:**
- Consumes: as Tasks 1–9 na branch `feat/mod-19d-controlled`; o api do 19d rodando na **3039** com a semente do 19d, o `fake-sncr` (8092), o PSC simulado e o `signer` do compose.
- Produces: branch pronta para merge (depois do merge do api do 19d).

- [ ] **Step 1: Suíte, tipos e build**

Run: `cd apps/dashboard/.claude/mod19d && npx vitest run && npx tsc --noEmit && npm run build`
Expected: tudo verde: a base anotada na Task 0 mais **10 arquivos de teste novos** (`api.controlled`, `controlled`, `ContactPanel`, `MyProfile.contact`, `SncrAccount`, `SncrCallback`, `NotificationRecordForm`, `PrescriptionForm.controlled`, `ConsultationDocuments.controlled`, `SncrOverview`). Um número de arquivos que dobra é artefato de build descoberto pelo vitest: pare e reporte. A CI roda os mesmos três; `dist/` está no `.gitignore`.

```bash
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19d status --short
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19d log --stat origin/main..HEAD | grep -c node_modules
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19d diff origin/main --stat -- src/components
```

Expected: `status` vazio, a contagem `0` e nenhum arquivo de `src/components` mudado (interface sem redesign).

- [ ] **Step 2: Nada vaza**

```bash
cd apps/dashboard/.claude/mod19d
grep -rn "console\." src/lib/controlled.ts src/modules/SncrAccount.tsx src/modules/SncrCallback.tsx src/modules/SncrOverview.tsx \
  src/modules/professionals/ContactPanel.tsx src/modules/documents/AddressFields.tsx src/modules/documents/PatientIdentificationFields.tsx \
  src/modules/documents/NotificationRecordForm.tsx
grep -rn "localStorage\|sessionStorage" src/lib/controlled.ts src/modules/SncrAccount.tsx src/modules/SncrCallback.tsx \
  src/modules/professionals/ContactPanel.tsx src/modules/documents/PatientIdentificationFields.tsx
sed -n '/─── Receita de controlado e SNCR (módulo 19d/,$p' src/lib/api.ts | grep -n "URLSearchParams\|?state\|?session_id\|?cpf\|?zip\|?phone"
```

Expected: os três sem saída — nenhum `console`, nenhum armazenamento do navegador, nada de `state`, `session_id`, CPF, CEP ou telefone em URL.

- [ ] **Step 3: Revisão**

Um subagente revisor lê `origin/main..HEAD` do dashboard contra a spec do 19d (§3–§8, §11), o ADR 0034 e o contrato do 19d (§1–§7). Pontos de atenção:
- sem `controlled_prescriptions`, a receita, a busca, a aba Documentos, Meu perfil e o menu são exatamente os do 19c (os testes do 19c continuam verdes sem mudança);
- a volta do login do SNCR troca o `session_id` uma vez, limpa a URL e tira o `state` do `sessionStorage` antes do POST, nunca guarda o `session_id`; o token do SNCR nunca aparece no dashboard;
- a tela nunca decide o modo: a previsão de papel é só aviso, e a frase depois da emissão vem do `issue_mode`/`paper_reason` do api; o CPF do paciente nunca vai no corpo (Divergência D2);
- C1/C5 entram, A/B e C2/C3/C4 ficam bloqueados com o motivo; a de controle especial não sai sem endereço do paciente, sem contato do prescritor, misturada com antimicrobiano, com mais de 3 substâncias C1 ou acima de 60/180 dias; a recusa do api aparece no formulário, que fica preenchido;
- a Notificação de papel não tem "Imprimir" nem "Cancelar e emitir outro"; cancelar uma receita numerada avisa que o SNCR não cancela e relê o estoque;
- a faixa "Numeração simulada — sem validade" aparece com `sncr_mock` ou número simulado em todas as telas do 19d;
- toda escrita leva JSON; nenhuma escrita do 19d pede step-up; leituras com dado pessoal com `gcTime: 0`;
- interface sem redesign: só componentes existentes; nenhum componente comum mudou.

- [ ] **Step 4: Prova no navegador (com o usuário)**

Depende do api do 19d rodando na porta **3039** (plano do api do 19d: servidor do worktree no container, semente com Curitiba em `record_mode = record`, `clinical_record`, `digital_signature`, `signature_psc_mock`, `clinical_documents`, `controlled_prescriptions` e `sncr_mock` ligados em dev, o catálogo com as listas da Portaria 344 por item, a médica da semente com CPF e certificado simulado, uma enfermeira, e o `fake-sncr` no compose). O Vite do worktree sobe num container da rede do compose, para alcançar `http://api:3039`:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose run -d --rm --no-deps --name dashboard-mod19d -p 5189:5189 \
  -w /app/.claude/mod19d -e VITE_API_PROXY_TARGET=http://api:3039 \
  dashboard npx vite --port 5189 --host 0.0.0.0
docker logs -f dashboard-mod19d   # espere o "ready"; Ctrl+C sai do log, o container segue
```

(Se o plano do api subir o servidor do 19d num container próprio, use o nome dele no lugar de `api`. O `redirect_uri` do gov.br simulado precisa apontar para `http://curitiba.localhost:5189/dashboard/sncr/callback` — o plano do api fixa isso no `fake-sncr`.)

Abra `http://curitiba.localhost:5189/dashboard/`. O usuário faz o login; não digite senha nem TOTP (as da semente de dev podem ser mostradas se ele pedir). Confira com screenshot:
- como a médica, **Meu perfil** → "Endereço e telefone na receita de controle especial": sem cadastro, o aviso; cadastrar com CEP incompleto (recusa na tela) e depois completo → "Endereço e telefone salvos.";
- **Conta → SNCR**: a faixa "Numeração simulada — sem validade", RCE e RET com 0 livres ("sem números: … saem em papel"); "Obter números do SNCR" (RCE) → o login simulado do SNCR (`fake-sncr`) → volta a `/dashboard/sncr/callback?session_id=…`, a URL fica limpa, "1.000 números recebidos do SNCR para receita de controle especial (RCE)." → "Continuar" → Conta → SNCR com 1.000 livres e "1 de 3"; repetir para RET; colar a URL de volta antiga na barra do navegador → "o gov.br não concluiu a autorização — comece de novo" (o `state` já saiu do `sessionStorage`) e nenhum número novo;
- na consulta, aba **Documentos** → **Receita**: buscar morfina e clonazepam (bloqueados com "Notificação de receita (listas A/B) …"), isotretinoína ("fora do Rota Saúde"), sertralina (entra, "lista C1"); categoria "receita de controle especial", a regra (2 vias, 30 dias), a identificação do paciente com o CPF mascarado, "Usar a última identificação" (a primeira vez: "nenhuma receita anterior…"), o prescritor com o endereço do perfil; incluir amoxicilina → "misturada — separe em duas receitas" e a recusa ao emitir; tirar a amoxicilina, 90 dias → recusa; 60 dias e o endereço → "Receita de controle especial emitida em modo digital com o número SNCR RCE nº … (numeração simulada — sem validade)…"; a linha mostra o título, o número, a faixa e "assinatura pendente" → em até 15 s "assinada digitalmente"; **Imprimir** → o PDF no leiaute da Anvisa com o número, as 2 vias e a faixa "NUMERAÇÃO SIMULADA — SEM VALIDADE";
- **Receita** só com amoxicilina → "receita de antimicrobiano", emitida digital com "RET nº …"; Conta → SNCR mostra 1 usado em cada tipo;
- **Notificação (papel)** → clonazepam (o tipo vira B sozinho), número do talão, 30 dias → "Notificação de papel registrada…"; a linha sem "Imprimir" e sem "Cancelar e emitir outro"; o clonazepam aparece em "Medicamentos em uso"; trocar o tipo para A antes de registrar → a recusa na tela;
- **Cancelar** a RCE → o diálogo avisa que o número fica anulado e que o SNCR não cancela; confirmar (step-up) → "cancelado em …"; Conta → SNCR com 1 anulado;
- desvincular o certificado (Conta → Assinatura digital) e abrir uma receita com sertralina → "Esta receita vai sair em papel: você não tem certificado digital ativo…"; emitir → "… emitida em modo papel (você não tem certificado digital ativo …): imprima as 2 vias e assine à mão." e o motivo na coluna "Modo"; vincular de novo depois;
- como a enfermeira: sem "Notificação (papel)"; Conta → SNCR com a recusa "só médicos e dentistas…";
- como `admin@curitiba.demo`, **Equipe → Painel do SNCR**: a médica com os números livres, quem nunca pediu, saldo baixo, a faixa de simulado;
- no maintenance, desligar `controlled_prescriptions` em Curitiba e recarregar o dashboard: Conta → SNCR, o Painel do SNCR, o painel de endereço em Meu perfil e "Notificação (papel)" somem; a receita volta a bloquear o controlado como no 19c ("receita de controle especial — 19d").

Depois: `docker rm -f dashboard-mod19d`. Religue o que a prova desligou (interruptores e certificado como estavam antes).

- [ ] **Step 5:** **Pare.** O merge do dashboard só vem depois do merge do api do 19d, e só com autorização explícita do usuário (merge, push e cards, uma etapa de cada vez). Ordem de deploy: contracts → api → dashboard → maintenance (contrato §11). Antes do push, confira `origin/main..main` no dashboard e publique só os commits desta entrega. Comandos, quando autorizados:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/dashboard
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

Formatos que este plano precisou e que o contrato do 19d não traz (ou traz de outro jeito). O plano do `api` do 19d (`2026-10-10-module-19d-controlled-api.md`, escrito em paralelo) já fixou parte deles nas Divergências D1–D18 dele; onde ele fixou, este plano segue e diz isso. Quem decide a forma final é o `api`; o dashboard se ajusta ao que ele entregar.

1. **D1 — `controlled_list` e `anticonvulsant` no `catalog_item` HTTP.** O contrato (§1) põe a lista da Portaria 344 "no item do catálogo", sem dizer em quais respostas; o plano do api (D18) os põe na busca e na REMUME. A tela precisa deles **também** nos itens de `GET …/renewal` e no `content.items` dos documentos (renovar uma RCE e "cancelar e emitir outro" partem desses itens; sem a lista, a sertralina renovada viraria "comum" na tela). Proposta: os dois campos em todo `catalog_item` que o api devolve. Não entram no JSON canônico.
2. **D2 — `patient_identification` de entrada sem `cpf`.** O dashboard só tem o CPF **mascarado** do prontuário (`RecordPatient.cpf_masked`) e não deve redigitar um CPF que o api já tem. Proposta: entrada `{ address }` (o api preenche `cpf` com o do cadastro); a saída e o canônico têm `{ cpf, address }`. **Decidido pelo usuário (2026-10-10):** sem "não possui CPF" no 19d — nada de `no_cpf` nem `passport`; o CPF do cadastro existe por regra do 19a.
3. **D3 — "Reaproveitar o último".** Segue o plano do api (D4): `GET /attendance/consultations/:id/patient_identification` → `{ patient_identification|null }` (a da última receita do paciente, para a autora da consulta). A tela reaproveita o endereço da identificação; 404 vira "nenhuma receita anterior…".
4. **D4 — Contato sem cadastro.** Segue o plano do api (D16): o telefone é o `professionals.phone` do módulo 10. Proposta complementar: com perfil e sem endereço, `GET …/contact` → `200 { address: null, phone: <o do módulo 10>|null }` (não 404); 404 `not_found` só sem perfil de profissional.
5. **D5 — O que é "misturar".** O contrato diz `mixed_categories`, sem dizer o quê. Proposta (a tela segue): misturada = controle especial (C1/C5) **e** antimicrobiano na mesma receita; item comum vai junto na categoria do controlado ou do antimicrobiano (como no 19c, em que o antimicrobiano levava os comuns para as 2 vias).
6. **D6 — Volta do login do SNCR.** **Decidido (coordenador):** `state` no `sessionStorage`, ou no `client_url` se a prova técnica permitir. Segue o plano do api (D1/D2): a volta traz `session_id` (não `code`) e não traz o nosso `state`; `POST /sncr/requests` devolve `{ authorize_url, state }` e o dashboard guarda o `state` no `sessionStorage` da aba entre a ida e a volta (e o apaga na volta). Isso contraria o padrão do 19b ("nada em storage"); por isso a proposta: se a prova técnica (Task 0 do api) mostrar que o `client_url` preserva a query string, o api põe o `state` no próprio `client_url` (`…/sncr/callback?state=…`) e o storage sai — a tela já usa o `state` da URL quando ele vier. O `session_id` nunca é guardado. `return_to` = `"/sncr"` (caminho de módulo, como a D3 do 19b); as recusas depois de achar o `state` (409/403/503) trazem `return_to`.
7. **D7 — Receita de antimicrobiano sem o contato do prescritor.** Segue o plano do api (D12): exigido só quando ela sairia digital (422 `prescriber_address_missing`); a tela bloqueia nesse caso e deixa emitir quando a previsão já é papel.
8. **D8 — `paper_reason` no `<document>`.** **Decidido (coordenador):** gravado no documento e devolvido na leitura (o plano do api, D10, que só o devolvia no `POST`, se ajusta). Forma: gravar e devolver no `<document>` também no `GET …/documents`, para a coluna "Modo" dizer por que um documento saiu em papel depois de recarregar a tela. A tela funciona dos dois jeitos (sem o campo, a coluna mostra só "papel").
9. **D9 — Erros do pedido de números.** Segue o plano do api (D5): 403 `registration_mismatch` (frase nova) e sem o 409 `professional_cpf_missing` (a frase fica, inofensiva). Proposta complementar: `GET /sncr/stock` também responde 403 `cbo_not_allowed` para quem não é médico nem dentista (o item do menu aparece para todo `health_professional`, porque o CBO é do vínculo, não da sessão).
10. **D10 — Uma forma por valor na entrada.** CEP só com 8 dígitos, UF maiúscula, complemento omitido quando vazio, telefone só com dígitos (DDD + número) — igual ao canônico (`contracts`, C4). O plano do api aceita pontuação e normaliza; a tela já manda normalizado.
11. **D11 — `low` do painel do admin.** O contrato dá `low: bool` sem a regra. Proposta: `true` quando `rce_free` **ou** `ret_free` < `low_threshold` (50), o mesmo limiar de `GET /sncr/stock`; `last_request_at: null` = nunca pediu; a ordem da lista fica com o dashboard (saldo baixo primeiro, depois nome).
12. **D12 — Lista C4.** Segue o plano do api (D8): C4 (antirretrovirais) é bloqueada na receita como C2/C3 (`not_supported`); o tipo `ControlledList` ganha `C4`.
13. **D13 — Conteúdo de saída do registro da Notificação.** O contrato (§3) só dá a entrada. Proposta: `content = { notification_type, paper_number, numbering_uf, item: <item do 19c> }`, `issue_mode: "paper"`, `signature: null`, `short_code: null`, `verification_url: null` (o plano do api, D11, já fixa os dois nulos e o 404 no impresso — a tela nem oferece "Imprimir").
14. **D14 — Step-up.** **Decidido (coordenador):** sem step-up. Nenhuma escrita do 19d pede step-up: pedir números passa pelo gov.br do próprio prescritor, e o contato é dado do próprio usuário. Cancelar continua com o step-up do 19c.
15. **D15 — Duração em todo item da RCE.** Segue o plano do api (D15): `duration_days` obrigatório em todo item da receita de controle especial, inclusive o comum que vai junto; a tela já não deixa mandar sem.

## Self-review

- **Cobertura (spec §7 e §11; contrato §2–§7):**
  - F-19.25 — Conta → SNCR (saldo por tipo, pedidos do mês, "Obter números do SNCR", saldo baixo, simulado): Tasks 2 e 4; retorno `/dashboard/sncr/callback`: Tasks 2 e 5;
  - F-19.26 — receita de controle especial: categoria calculada, identificação e endereço do paciente, contato do prescritor, limites (3 C1, 60/180 dias), aviso "vai sair em papel" antes e `paper_reason` depois, faixa de simulado, número na lista, cancelamento com número anulado: Tasks 2, 3, 6, 7 e 8;
  - F-19.27 — receita de antimicrobiano digital (RET): categoria, regra de 10 dias, contato, previsão de papel (enfermagem sempre papel), número: Tasks 2, 7 e 8;
  - F-19.28 — "Registrar Notificação de papel" na aba Documentos (tipo × lista, duração por tipo, sem PDF, entra nos medicamentos em uso): Tasks 2, 6 e 8;
  - F-19.29 — Painel do SNCR do admin: Tasks 2 e 9;
  - contato do profissional (endereço e telefone): Task 3; interruptor `controlled_prescriptions` e proxy `/sncr`: Task 1; LGPD (spec §8): Global Constraints, Tasks 1 e 5 (teste de console/storage) e Task 10 (Step 2); prova no navegador (spec §9): Task 10.
  - Fora desta entrega, de propósito: o PDF no leiaute da Anvisa e a página pública de conferência (são do api; a Task 10 só os confere).
- **Placeholders:** nenhum. Arquivos novos vêm inteiros; os do 19c e do 19b, por trecho com contexto (a Task 0 manda conferir o texto real, que vem do plano do 19c enquanto ele não estiver em `main`). A única frase condicional é o `sub` do painel do admin (Task 9), que espera a regra da D11 do api.
- **Consistência de nomes:** `SNCR_STOCK_KEY`, `SNCR_OVERVIEW_KEY`, `CONTACT_KEY`, `SNCR_RETURN_TO` (Task 2) usados nas Tasks 3, 4, 5, 7, 8 e 9; `ControlledContext` (Task 7) montado na Task 8; `IssueContext.patientCpfMasked` (Task 8) vindo do `ConsultationWorkspace` e `ControlledContext.consultationId`; `rememberSncrState`/`takeSncrState` (Task 2) na Conta → SNCR (Task 4) e na volta (Task 5); `blockReason` (Task 6) usado pela receita (Task 7) e pela Notificação (Task 6); `AddressFields` (Task 3) em `ContactPanel` e `PatientIdentificationFields` (Task 7); `CONTROLLED_DISABLED` (`clinicalDocuments.ts`) e o apelido `CONTROLLED_DISABLED_TEXT` (`controlled.ts`) com o mesmo texto; ids de módulo `sncr` e `sncr-overview` iguais em `modules.ts`, `App.tsx`, `SncrCallback` (`moduleFromPath("/sncr")`) e nos testes; fixtures `NOW19D`, `TODAY19D`, `controlledUser`, `SERTRALINE`…`ISOTRETINOIN`, `SERTRALINE_REF`, `AMOXICILLIN_19D`, `address`, `prescriberContact`, `contact`, `stock`, `rceDoc`, `retDoc`, `notificationDoc`, `sncrOverview` (Task 1) nas Tasks 2–9.
- **Review Focus:** 1 — Task 8 ("previsão digital, mas o api devolve papel…"); 2 — Task 5 ("autorização vencida…", "SNCR esgotado… e state já usado…", "recusa no gov.br com state…", "StrictMode…"); 3 — Task 7 ("mistura…", "quatro substâncias C1…", "duração acima de 60 dias…", "recusa do api fica no formulário, com o endereço"); 4 — Task 7 ("A/B bloqueados…") e Task 6 ("tipo que não bate…", "item fora das listas A/B…"); 5 — Task 8 ("cancelar RCE: avisa que o SNCR não cancela e relê o estoque").
