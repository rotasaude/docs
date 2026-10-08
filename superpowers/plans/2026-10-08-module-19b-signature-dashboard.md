# Módulo 19 (19b) — Assinatura digital (dashboard) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito: 19a fechado em `origin/main`** (dashboard e api; F-19.1..7 Verified). Este plano mexe em peças que o plano do dashboard do 19a (`docs/superpowers/plans/2026-10-07-module-19-consultation-dashboard.md`) cria: a seção do prontuário em `src/lib/api.ts` (`Consultation`, `Addendum`, `fetchConsultationPdf`), `src/lib/consultation.ts` (`CONSULTATION_KEY`), `src/modules/consultation/ConsultationView.tsx`, `src/test/consultationFixtures.ts` (`finalized()`), o item `clinical-record` em `src/shell/modules.ts`/`src/App.tsx`, `clinical_record` em `src/lib/features.ts` e o proxy `/clinical_record`. A Task 0 confere isso; se faltar, **pare**. E o **api do 19b** precisa estar rodando na branch dele (porta **3037**) para a prova (Task 9) e **mergeado antes** do merge deste.

**Goal:** Telas da assinatura digital no painel da cidade (F-19.9, F-19.10, F-19.12, F-19.13 e F-19.14, lado dashboard): Conta → Assinatura digital (procurar o certificado pelo CPF, vincular, trocar, desvincular com step-up, aviso de vencimento em 30 dias); selo da sessão de assinatura no topo (abrir e encerrar); retorno do prestador em `/dashboard/signature/callback`, que troca `state`+`code` no api e devolve a pessoa à tela de onde saiu; marcador de assinatura na consulta e em cada adendo, com "Ver o que foi assinado", "Baixar PDF assinado", "Baixar .p7s" e "Revalidar"; Pendentes de assinatura com "Assinar todas" e "Voltar ao papel" (motivo ≥ 10), e o contador no menu; aviso de como sai o impresso; e o Painel de assinatura do `municipal_admin` (só leitura).

**Architecture:** O cliente HTTP vai para uma seção nova no fim de `src/lib/api.ts`, com os tipos copiados do contrato do 19b (§1–§7), e o bloco `signature?` entra nos tipos `Consultation` e `Addendum` do 19a. As regras de tela ficam em `src/lib/signature.ts` (puras: quem pode, rótulos, avisos, frases das recusas, resumo do retorno, leitura da URL de retorno; e duas funções de navegador: `goToProvider` e `saveBlob`). As telas novas: `src/modules/SignatureAccount.tsx` (Conta), `src/modules/SignaturePending.tsx` (Atendimento), `src/modules/SignatureOverview.tsx` (Equipe, admin), `src/modules/SignatureCallback.tsx` (retorno do prestador, fora da casca), `src/modules/signature/SignatureMarker.tsx` e `SignatureDetail.tsx` (dentro da consulta do 19a), `src/shell/SignatureSessionBadge.tsx` (selo no topo) e o hook `src/hooks/usePendingSignatures.ts` (lista e contador). As existentes ganham o mínimo: `modules.ts` (três itens, `moduleFromPath`, `withPendingCount`), `App.tsx` (rotas e `initialModule`), `main.tsx` (desvio para o retorno do prestador), `AppHeader.tsx` (selo e contador), `ConsultationView.tsx` (marcadores e aviso do impresso), `features.ts` (`digital_signature`) e o proxy do Vite (`/signature`). O dashboard nunca decide o que o api garante: o OAuth (PKCE, `code_verifier`, token do prestador) é todo do api; a tela só segue o `authorize_url` e devolve `state`+`code`.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, Vitest 2 + Testing Library 16 (jsdom, sem jest-dom, sem msw: `vi.mock("../lib/api")` e `vi.stubGlobal("fetch")`), Vite 5 (base `/dashboard/`, proxy de dev).

**Spec:** `docs/superpowers/specs/2026-10-08-module-19b-digital-signature-design.md` (§9 é deste plano; §3, §5, §7, §8 e §10 dão as regras), `docs/adr/0032.md` e o contrato `docs/superpowers/plans/2026-10-08-module-19b-signature-contracts.md` (§1–§7; a seção "Divergências propostas ao contrato" no fim deste plano diz o que ele ainda não fixa).

## Global Constraints

- Rotas (contrato §3–§7; sessão da cidade, cookie): `GET /signature/certificates/current` (404 `certificate_not_linked`), `POST /signature/certificates/discover`, `POST /signature/certificates/link { provider, return_to }` (step-up; 422 `invalid_provider`), `DELETE /signature/certificates/current` (step-up; 204); `POST /signature/sessions { return_to }` (409 `certificate_not_linked`), `GET /signature/sessions/current` → `{ active, expires_at?, provider? }`, `DELETE /signature/sessions/current`; `POST /signature/oauth/callback { state, code }` → `{ purpose, result }` (422 `invalid_state`, `certificate_cpf_mismatch`, `certificate_expired`, `certificate_revoked`; 409 `authorization_expired`; 403 `authorization_denied`; 503 `provider_unavailable`); `GET /signature/requests?status=pending` → `{ items }`; `POST /signature/requests/:id/return_to_paper { reason }` (422 `invalid_reason`; 403 `not_author`; 409 `not_pending`); `POST /signature/batches { request_ids?, return_to }` → `{ authorize_url, count }` (409 `nothing_pending`, `certificate_not_linked`); `GET /signature/signatures/:id`, `/pdf`, `/package`, `POST …/verify` (403 `out_of_context`/`opening_required` como no 19a); `GET /signature/admin/overview?from=&to=` (`municipal_admin`).
- Erros `{ "error": "<reason>" }`; step-up 401 `{ "error": "mfa_required" }` (quem trata é o `SensitiveAction`); interruptor desligado 403 `{ "error": "feature_disabled", "feature": "digital_signature" }` (ou `"clinical_record"`, pré-requisito); escrita devolve o objeto puro.
- Valores (contrato §1): `provider` ∈ `vidaas` | `birdid` | `safeid` | `neoid` | `remoteid`; pedido `pending` | `signed` | `failed` | `returned_to_paper`; modo `digital` | `manual` | `pending`; validação `valid` | `invalid` | `indeterminate`; motivos `no_session`, `session_expired`, `provider_unavailable`, `provider_rejected`, `signer_unavailable`, `verification_failed`, `certificate_expired`, `certificate_revoked`, `feature_disabled`, `user_request`; `document_type` `consultation` | `consultation_addendum`. Valor desconhecido (api mais novo) aparece cru, nunca some nem vira `undefined`.
- Regras da tela (spec §5, §9): aviso de vencimento quando faltam ≤ 30 dias; motivo da volta ao papel ≥ 10 caracteres (sem os espaços das pontas); lote de até 50 documentos (sem ids = os pendentes mais antigos); "pendente há mais de 24 h" no painel do admin; período do painel padrão de 30 dias, no fuso da cidade.
- Retorno do prestador: a rota do dashboard é `<BASE_URL>signature/callback` = **`/dashboard/signature/callback`** (Divergência D1: `/signature` é prefixo do api). `return_to` é caminho do dashboard `/<id do módulo>` (o dashboard navega por estado, não por URL; Divergência D3) e casa com `/^\/[^\/]/`.
- Segurança e LGPD (spec §10): `state` e `code` só no corpo do POST e a URL de retorno é limpa **antes** do POST; nada de token, `code_verifier` ou texto clínico no navegador além do que a tela mostra; nada em `console`, `localStorage`/`sessionStorage`, URL ou nome de arquivo baixado (só ids); o conteúdo assinado é lido com `gcTime: 0`; a recepção e quem não é `health_professional` não chama nenhuma rota de assinatura do profissional; o painel é só do `municipal_admin`; o operador nunca vê nada disso.
- Quem vê: Conta → Assinatura digital, selo e Pendentes de assinatura só para `health_professional` com `digital_signature` na sessão (`canSign`); Painel de assinatura só para `municipal_admin` com `digital_signature` (`canSeeSignatureOverview`); operador nunca.
- Interface tem ciclo próprio: use os componentes e estilos existentes (`Panel`, `PageHeader`, `DataTable`, `Tag`, `KeyValue`, `EmptyState`, `SensitiveAction`, `formStyles`), sem redesign. Nenhum componente comum muda.
- Testes que dependem de "agora" fixam o relógio com `vi.useFakeTimers({ toFake: [ "Date" ] })` (mais `setInterval`/`clearInterval` no selo da sessão) + `vi.setSystemTime(new Date(NOW19B))` (`NOW19B = "2026-10-08T10:00:00-03:00"`) e `vi.useRealTimers()` no `afterEach`. Funções puras recebem `nowMs` como argumento. Navegar para o prestador é a prop `redirect` (padrão `goToProvider`), trocada por `vi.fn()` nos testes.
- Nunca `git add -A` nem `git add .`: o worktree tem o symlink de `node_modules`. Adicione arquivos pelo caminho.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Merge, push e tag só com autorização explícita do usuário.

## Ambiente de execução

- Worktree `apps/dashboard/.claude/mod19b`, branch `feat/mod-19b-signature` a partir de `origin/main` (Task 0). Todos os caminhos de arquivo das tasks são relativos a ele; os comandos partem da raiz do monorepo (`/Users/eduardovrocha/Development/ioit.solutions/rota-saude`).
- Testes (script `test` = `vitest run`): `cd apps/dashboard/.claude/mod19b && npx vitest run <arquivos>`.
- Tipos antes de cada commit (`typecheck` = `tsc --noEmit`; `noUnusedLocals` ligado e o `tsconfig` inclui os testes): `cd apps/dashboard/.claude/mod19b && npx tsc --noEmit`.
- Ambiente de teste: `vitest.config.ts` fixa `TZ=America/Sao_Paulo` e não define `base` (nos testes `import.meta.env.BASE_URL` é `/`; por isso `readSignatureCallback` recebe a base como argumento); sem `setupFiles` nem jest-dom (use `not.toBeNull()`, `toBeNull()`, `toBe()`); `globals: false`, então todo teste de componente chama `cleanup` no `afterEach`.
- Os diffs abaixo têm o contexto do arquivo **depois do merge do 19a** e das tasks anteriores. Confira o trecho "antes" no arquivo real; se o 19a mudou algo no caminho, aplique a mesma intenção sobre o texto que estiver lá. Aplique à mão ou com `git apply` a partir de um arquivo no scratchpad (nunca dentro do worktree).
- O api do 19b roda em dev na porta **3037** (plano do api do 19b). O Vite do worktree sobe **num container da rede do compose**, na **5187**, com o proxy para `http://api:3037` (Task 9).

## Review Focus

1. **O retorno do prestador é trocado duas vezes.** O `state` é de uso único: sob o `StrictMode` o efeito roda duas vezes, e um recarregar ou o "voltar" do navegador refaz a URL com `code`. A segunda troca daria 422 `invalid_state` e a tela diria "falhou" sobre um vínculo que deu certo. A tela troca o código **uma** vez e limpa a URL antes do POST. Testes: Task 8, "StrictMode: troca o código uma vez só" e "vínculo: limpa a URL antes de trocar o código, diz o resultado e volta à Conta".
2. **A sessão de assinatura vence com a tela aberta.** O turno acaba às 22:00 e a médica continua no painel: o selo precisa passar a "sem sessão de assinatura" na hora, sem recarregar, e oferecer abrir outra — senão ela finaliza achando que vai assinar e a consulta fica pendente. Teste: Task 5, "a sessão vence com a tela aberta: o selo muda sozinho".
3. **O documento muda de estado entre a leitura da lista e o clique.** O job assinou a consulta (sessão aberta em outra aba) enquanto a médica escrevia o motivo de "Voltar ao papel", ou "Assinar todas" chega com nada pendente: 409 `not_pending`/`nothing_pending` relê a lista e diz o que houve, sem erro parado. Testes: Task 4, "not_pending fecha o formulário, diz e relê a lista" e "nothing_pending relê a lista e diz que não há pendentes".
4. **Certificado vencendo, vencido, revogado ou de outro CPF.** O aviso aparece 30 dias antes (com a data), o vencido e o revogado dizem o que fazer, e o certificado de outra pessoa autorizado no prestador volta com uma frase clara. Testes: Task 2, "aviso do certificado: longe, 30 dias, amanhã, hoje, vencido e revogado"; Task 3, "vence em 12 dias: avisa com a data"; Task 8, "certificado de outro CPF: diz o motivo".
5. **Interruptor desligado depois de haver assinaturas, e assinatura que deixou de ser válida.** As consultas pendentes voltam ao papel com `feature_disabled` e as já assinadas continuam visíveis; uma revalidação pode dar `indeterminate`. O marcador mostra "assinatura à mão (papel) · assinatura digital desligada na cidade" e "assinatura digital indeterminada", sem esconder; os itens de menu somem sem a funcionalidade na sessão. Testes: Task 6, "voltou ao papel porque o interruptor foi desligado" e "Revalidar devolve indeterminada: o estado muda na tela"; Tasks 3, 4 e 7, os testes de menu.

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | conferência do 19a em `origin/main`, worktree, base | 0 |
| `src/lib/api.ts`, `src/lib/api.signature.test.ts`, `src/test/signatureFixtures.ts`, `src/lib/features.ts`, `src/lib/features.test.ts`, `vite.config.ts`, `README.md` | tipos do contrato; bloco `signature` na consulta e no adendo; cliente; interruptor `digital_signature`; proxy `/signature` | 1 |
| `src/lib/signature.ts`, `src/lib/signature.test.ts` | quem pode; rótulos; avisos; frases; resumo e destino do retorno; URL de retorno; painel; navegador (`goToProvider`, `saveBlob`) | 2 |
| `src/modules/SignatureAccount.tsx` (+ teste), `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/App.tsx` | Conta → Assinatura digital | 3 |
| `src/hooks/usePendingSignatures.ts`, `src/modules/SignaturePending.tsx` (+ teste), `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/App.tsx` | Pendentes de assinatura, lote e volta ao papel | 4 |
| `src/shell/SignatureSessionBadge.tsx` (+ teste), `src/shell/AppHeader.tsx`, `src/shell/modules.ts`, `src/shell/modules.test.ts` | selo da sessão no topo; contador no menu | 5 |
| `src/modules/signature/SignatureMarker.tsx`, `src/modules/signature/SignatureDetail.tsx` (+ testes), `src/modules/consultation/ConsultationView.tsx`, `src/modules/consultation/ConsultationView.signature.test.tsx` | marcador na consulta e no adendo; o que foi assinado, downloads, revalidar; aviso do impresso | 6 |
| `src/modules/SignatureOverview.tsx` (+ teste), `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/App.tsx` | Painel de assinatura do admin | 7 |
| `src/modules/SignatureCallback.tsx` (+ teste), `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/main.tsx`, `src/App.tsx` | retorno do prestador e volta à tela de origem | 8 |
| — | suíte, build, revisão e prova no navegador | 9 |

**Estratégia de teste:** regras puras com tabela de casos (Task 2); cliente HTTP com `fetch` falso conferindo URL, método e corpo — e que `state`, `code`, motivo e CPF nunca vão na URL (Task 1); cada tela com `vi.mock("../lib/api")` cobrindo o caminho feliz, cada recusa que muda o comportamento da tela e as bordas do Review Focus; o selo com relógio falso avançando; o retorno do prestador dentro de `<StrictMode>`; o marcador testado dentro da `ConsultationView` real do 19a. Navegar para o prestador e baixar arquivo são trocados por dublês (`redirect`, `URL.createObjectURL`, `HTMLAnchorElement.prototype.click`). A prova final é no navegador, contra o api do 19b com o PSC falso (Task 9).

---

### Task 0: Conferência do 19a, worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do dashboard com o 19a mergeado.
- Produces: worktree `apps/dashboard/.claude/mod19b` na branch `feat/mod-19b-signature`; a base anotada (arquivos e testes verdes).

- [ ] **Step 1: Confira que o 19a está em `origin/main`**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude
/opt/homebrew/bin/git -C apps/dashboard fetch origin
for f in src/modules/consultation/ConsultationView.tsx src/lib/consultation.ts src/test/consultationFixtures.ts src/modules/ClinicalRecord.tsx; do
  /opt/homebrew/bin/git -C apps/dashboard cat-file -e origin/main:$f && echo "ok $f"
done
/opt/homebrew/bin/git -C apps/dashboard show origin/main:src/lib/features.ts | grep -c '"clinical_record"'
/opt/homebrew/bin/git -C apps/dashboard show origin/main:src/lib/api.ts | grep -n "export interface Addendum\|export interface Consultation \|export async function fetchConsultationPdf"
/opt/homebrew/bin/git -C apps/dashboard show origin/main:src/lib/consultation.ts | grep -n "CONSULTATION_KEY"
/opt/homebrew/bin/git -C apps/dashboard log --oneline -1 origin/main
```

Expected: quatro linhas `ok …`, contagem ≥ 1, as três declarações de `api.ts` e a de `CONSULTATION_KEY`. Se faltar qualquer uma, **pare** e avise: o 19a não foi mergeado (o 19b só executa depois do 19a fechado, spec "Pré-requisito de execução").

Confira também que o 19a deixou a `ConsultationView` com o cabeçalho e a lista de adendos que a Task 6 altera:

```bash
/opt/homebrew/bin/git -C apps/dashboard show origin/main:src/modules/consultation/ConsultationView.tsx | grep -n 'Consulta de \${fmtDateTime(c.finalized_at)}\|Adendo de \${a.author_name}\|aria-label="adendos"'
```

Expected: as três linhas. Se o 19a as escreveu diferente, anote: a Task 6 aplica a mesma intenção (marcador logo depois do cabeçalho e dentro de cada adendo) sobre o texto real.

- [ ] **Step 2: Crie o worktree e ligue o `node_modules`** (`/.claude/` já está no `.gitignore` do dashboard)

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude
/opt/homebrew/bin/git -C apps/dashboard worktree add .claude/mod19b -b feat/mod-19b-signature origin/main
ln -s ../../node_modules apps/dashboard/.claude/mod19b/node_modules
cd apps/dashboard/.claude/mod19b && npx vitest run 2>&1 | tail -3 && npx tsc --noEmit && echo tsc-ok
```

Expected: tudo verde e `tsc-ok`. **Anote** a base (arquivos e testes): é a de `origin/main` depois do 19a. A Task 9 espera a base mais **10 arquivos de teste novos**.

---

### Task 1: Cliente HTTP da assinatura, tipos do contrato, interruptor e proxy

**Files:**
- Modify: `src/lib/api.ts` (`Addendum`, `Consultation` do 19a; seção nova no fim do arquivo)
- Modify: `src/lib/features.ts`, `src/lib/features.test.ts`, `vite.config.ts`, `README.md`
- Create: `src/test/signatureFixtures.ts`
- Test: `src/lib/api.signature.test.ts`

**Interfaces:**
- Consumes: `jsonFetch`, `postProfessional`, `ApiError`, `errorCode`, `SessionUser` (já em `api.ts`); `sessionWith` (`src/test/campaignFixtures.tsx`).
- Produces:
  - tipos: `SignatureProvider`, `SignatureMode`, `SignatureRequestStatus`, `SignatureVerification`, `SignatureReason`, `SignatureDocumentType`, `CertificateStatus`, `SignatureFileKind` (`"pdf" | "package"`), `SignatureBlock`, `SignerCertificate`, `CertificateDiscovery`, `AuthorizeRedirect`, `SignatureSessionState`, `BatchResult`, `OAuthCallbackResult` (união por `purpose`, com `return_to?`, Divergência D2), `SignatureRequest`, `SignatureDetail`, `OverviewProfessional`, `OverviewSignatureRow`, `SignatureOverview`;
  - campos: `Consultation.signature?: SignatureBlock`, `Addendum.signature?: SignatureBlock`;
  - funções: `getCurrentCertificate(): Promise<SignerCertificate | null>` (404 `certificate_not_linked` → `null`), `discoverCertificates(): Promise<CertificateDiscovery>`, `linkCertificate(provider, returnTo): Promise<AuthorizeRedirect>`, `unlinkCertificate(): Promise<void>`, `openSignatureSession(returnTo): Promise<AuthorizeRedirect>`, `getSignatureSession(): Promise<SignatureSessionState>`, `closeSignatureSession(): Promise<void>`, `completeSignatureOAuth(state, code): Promise<OAuthCallbackResult>`, `listPendingSignatures(): Promise<SignatureRequest[]>`, `returnToPaper(id, reason): Promise<SignatureRequest>`, `startSignatureBatch(returnTo, requestIds?): Promise<AuthorizeRedirect & { count: number }>`, `getSignature(id): Promise<SignatureDetail>`, `verifySignature(id): Promise<SignatureDetail>`, `fetchSignatureFile(id, kind: SignatureFileKind): Promise<Blob>`, `getSignatureOverview({ from, to }): Promise<SignatureOverview>`;
  - `FeatureKey` com `"digital_signature"` (rótulo "Assinatura digital");
  - fixtures: `NOW19B`, `signer(roles?, over?)` (sessão `us1` com `clinical_record` e `digital_signature`), `certificate()`, `signatureBlock()`, `pendingRequest()`, `signatureDetail()`, `overview()`.

- [ ] **Step 1: Write the failing test**

Fixtures (usadas nesta e nas próximas tasks):

```ts
// src/test/signatureFixtures.ts
// Dados comuns aos testes do módulo 19b (assinatura digital). Relógio dos
// testes: quinta, 2026-10-08 10:00 em São Paulo (-03:00). A autora é `us1`,
// a mesma das fixtures da consulta do 19a.
import type {
  SessionUser, SignatureBlock, SignatureDetail, SignatureOverview, SignatureRequest, SignerCertificate
} from "../lib/api";
import { sessionWith } from "./campaignFixtures";

export const NOW19B = "2026-10-08T10:00:00-03:00";

// Profissional com as duas funcionalidades ligadas; janela de step-up aberta
// (sessionWith põe `mfa_verified_at` = agora). `{ mfa_verified_at: null }`
// força o pedido do código.
export function signer(roles: string[] = [ "health_professional" ], over: Partial<SessionUser> = {}): SessionUser {
  return sessionWith(roles, {
    id: "us1", email_address: "medica@curitiba.demo", features: [ "clinical_record", "digital_signature" ], ...over
  });
}

// 2026-10-08 → 2027-03-15: 158 dias.
export function certificate(over: Partial<SignerCertificate> = {}): SignerCertificate {
  return {
    id: "sc1", provider: "vidaas", issuer: "AC VALID RFB v5", serial_number: "5A3F09",
    not_after: "2027-03-15T23:59:59-03:00", status: "active", expires_in_days: 158, ...over
  };
}

export function signatureBlock(over: Partial<SignatureBlock> = {}): SignatureBlock {
  return {
    mode: "digital", request_id: "sr1", signature_id: "sg1", signed_at: "2026-10-07T10:21:00-03:00",
    signer_name: "Helena Prado", verification: "valid", ...over
  };
}

export function pendingRequest(over: Partial<SignatureRequest> = {}): SignatureRequest {
  return {
    id: "sr1", document_type: "consultation", document_id: "cs1", consultation_id: "cs1",
    patient_display_name: "Joana Lima", finalized_at: "2026-10-07T10:20:00-03:00", status: "pending",
    reason_code: "no_session", attempts: 0, ...over
  };
}

export function signatureDetail(over: Partial<SignatureDetail> = {}): SignatureDetail {
  return {
    id: "sg1", document_type: "consultation", document_id: "cs1", signed_at: "2026-10-07T10:21:00-03:00",
    signer_name: "Helena Prado", signer_cpf_masked: "***.456.789-**", policy: "AD-RB", verification: "valid",
    verification_reasons: [], verified_at: "2026-10-08T09:00:00-03:00",
    content: { schema: "rotasaude.consultation.v1", consultation: { id: "cs1", assessment: "Diabetes descompensado." } },
    ...over
  };
}

export function overview(over: Partial<SignatureOverview> = {}): SignatureOverview {
  return {
    professionals: [
      { user_id: "us1", name: "Helena Prado", certificate_status: "active", not_after: "2027-03-15T23:59:59-03:00",
        pending_count: 0, oldest_pending_at: null },
      { user_id: "us2", name: "Lúcia Prado", certificate_status: "expiring", not_after: "2026-10-20T23:59:59-03:00",
        pending_count: 3, oldest_pending_at: "2026-10-06T16:00:00-03:00" },
      { user_id: "us3", name: "Rafael Souza", certificate_status: "none", not_after: null,
        pending_count: 0, oldest_pending_at: null }
    ],
    documents_by_mode: { digital: 42, manual: 17, pending: 3 },
    invalid_or_indeterminate: [
      { signature_id: "sg9", document_type: "consultation_addendum", signer_name: "Lúcia Prado",
        verification: "indeterminate", verified_at: "2026-10-08T08:00:00-03:00" }
    ],
    ...over
  };
}
```

Teste do cliente:

```ts
// src/lib/api.signature.test.ts
import { afterEach, describe, expect, it, vi } from "vitest";
import {
  ApiError, closeSignatureSession, completeSignatureOAuth, discoverCertificates, fetchSignatureFile, getCurrentCertificate,
  getSignature, getSignatureOverview, getSignatureSession, linkCertificate, listPendingSignatures, openSignatureSession,
  returnToPaper, startSignatureBatch, unlinkCertificate, verifySignature
} from "./api";
import { certificate, overview, pendingRequest, signatureDetail } from "../test/signatureFixtures";

afterEach(() => vi.unstubAllGlobals());

function stub(body: unknown, status = 200) {
  const fn = vi.fn(async (_input: RequestInfo | URL, _init?: RequestInit) =>
    new Response(body === undefined ? null : JSON.stringify(body), { status, headers: { "Content-Type": "application/json" } }));
  vi.stubGlobal("fetch", fn);
  return fn;
}
const call = (fn: ReturnType<typeof stub>, i = 0) => fn.mock.calls[i] as [ string, RequestInit ];
const sent = (fn: ReturnType<typeof stub>, i = 0) => JSON.parse(call(fn, i)[1].body as string);

describe("cliente da assinatura digital — certificado", () => {
  it("lê o certificado corrente; 404 certificate_not_linked vira null; outra recusa sobe", async () => {
    const fn = stub(certificate());
    expect((await getCurrentCertificate())?.provider).toBe("vidaas");
    expect(call(fn)[0]).toBe("/signature/certificates/current");
    expect(call(fn)[1].method).toBeUndefined();
    expect(call(fn)[1].credentials).toBe("include");

    stub({ error: "certificate_not_linked" }, 404);
    expect(await getCurrentCertificate()).toBeNull();

    stub({ error: "feature_disabled", feature: "digital_signature" }, 403);
    await expect(getCurrentCertificate()).rejects.toBeInstanceOf(ApiError);
  });

  it("procurar, vincular (provider e return_to no corpo) e desvincular", async () => {
    let fn = stub({ providers: [ { provider: "vidaas", found: true } ], unavailable: [] });
    expect((await discoverCertificates()).providers[0].found).toBe(true);
    expect(call(fn)[0]).toBe("/signature/certificates/discover");
    expect(call(fn)[1].method).toBe("POST");

    fn = stub({ authorize_url: "https://psc.example/authorize?x=1" });
    expect((await linkCertificate("vidaas", "/signature")).authorize_url).toBe("https://psc.example/authorize?x=1");
    expect(call(fn)[0]).toBe("/signature/certificates/link");
    expect(sent(fn)).toEqual({ provider: "vidaas", return_to: "/signature" });

    fn = stub(undefined, 204);
    await unlinkCertificate();
    expect(call(fn)[0]).toBe("/signature/certificates/current");
    expect(call(fn)[1].method).toBe("DELETE");
  });
});

describe("cliente da assinatura digital — sessão e retorno do prestador", () => {
  it("abrir, ler e encerrar a sessão", async () => {
    let fn = stub({ authorize_url: "https://psc.example/authorize?s=1" });
    expect((await openSignatureSession("/attendance")).authorize_url).toBe("https://psc.example/authorize?s=1");
    expect(call(fn)[0]).toBe("/signature/sessions");
    expect(sent(fn)).toEqual({ return_to: "/attendance" });

    fn = stub({ active: true, expires_at: "2026-10-08T22:00:00-03:00", provider: "vidaas" });
    expect((await getSignatureSession()).active).toBe(true);
    expect(call(fn)[0]).toBe("/signature/sessions/current");

    fn = stub(undefined, 204);
    await closeSignatureSession();
    expect(call(fn)[0]).toBe("/signature/sessions/current");
    expect(call(fn)[1].method).toBe("DELETE");
  });

  it("state e code vão no corpo, nunca na URL", async () => {
    const fn = stub({ purpose: "session", result: { expires_at: "2026-10-08T22:00:00-03:00" } });
    const out = await completeSignatureOAuth("st-1", "code-1");
    expect(out.purpose).toBe("session");
    expect(call(fn)[0]).toBe("/signature/oauth/callback");
    expect(call(fn)[1].method).toBe("POST");
    expect(sent(fn)).toEqual({ state: "st-1", code: "code-1" });
  });
});

describe("cliente da assinatura digital — pendentes e lote", () => {
  it("lista as pendentes e volta ao papel com o motivo no corpo", async () => {
    let fn = stub({ items: [ pendingRequest() ] });
    expect(await listPendingSignatures()).toHaveLength(1);
    expect(call(fn)[0]).toBe("/signature/requests?status=pending");

    fn = stub(pendingRequest({ status: "returned_to_paper", reason_code: "user_request" }));
    expect((await returnToPaper("sr/1", "paciente pediu o papel")).status).toBe("returned_to_paper");
    expect(call(fn)[0]).toBe("/signature/requests/sr%2F1/return_to_paper");
    expect(sent(fn)).toEqual({ reason: "paciente pediu o papel" });
  });

  it("lote: sem ids manda só o return_to; com ids, os dois", async () => {
    let fn = stub({ authorize_url: "https://psc.example/authorize?b=1", count: 3 });
    expect((await startSignatureBatch("/signature-pending")).count).toBe(3);
    expect(call(fn)[0]).toBe("/signature/batches");
    expect(sent(fn)).toEqual({ return_to: "/signature-pending" });

    fn = stub({ authorize_url: "https://psc.example/authorize?b=2", count: 1 });
    await startSignatureBatch("/signature-pending", [ "sr1" ]);
    expect(sent(fn)).toEqual({ request_ids: [ "sr1" ], return_to: "/signature-pending" });
  });
});

describe("cliente da assinatura digital — assinatura e painel", () => {
  it("ler e revalidar a assinatura", async () => {
    let fn = stub(signatureDetail());
    expect((await getSignature("sg1")).policy).toBe("AD-RB");
    expect(call(fn)[0]).toBe("/signature/signatures/sg1");

    fn = stub(signatureDetail({ verification: "indeterminate" }));
    expect((await verifySignature("sg1")).verification).toBe("indeterminate");
    expect(call(fn)[0]).toBe("/signature/signatures/sg1/verify");
    expect(call(fn)[1].method).toBe("POST");
  });

  it("baixa PDF e pacote com a sessão; recusa vira ApiError com o código", async () => {
    let fn = vi.fn(async (_input: RequestInfo | URL, _init?: RequestInit) =>
      new Response("%PDF-1.7", { status: 200, headers: { "Content-Type": "application/pdf" } }));
    vi.stubGlobal("fetch", fn);
    expect((await fetchSignatureFile("sg1", "pdf")).size).toBe(8);
    expect(fn.mock.calls[0][0]).toBe("/signature/signatures/sg1/pdf");
    expect((fn.mock.calls[0][1] as RequestInit).credentials).toBe("include");
    expect(((fn.mock.calls[0][1] as RequestInit).headers as Record<string, string>).Accept).toBe("application/pdf");

    fn = vi.fn(async (_input: RequestInfo | URL, _init?: RequestInit) =>
      new Response("PK", { status: 200, headers: { "Content-Type": "application/zip" } }));
    vi.stubGlobal("fetch", fn);
    await fetchSignatureFile("sg1", "package");
    expect(fn.mock.calls[0][0]).toBe("/signature/signatures/sg1/package");
    expect(((fn.mock.calls[0][1] as RequestInit).headers as Record<string, string>).Accept).toBe("application/zip");

    stub({ error: "out_of_context" }, 403);
    const err = await fetchSignatureFile("sg1", "pdf").catch((e: unknown) => e);
    expect(err).toBeInstanceOf(ApiError);
    expect((err as ApiError).body).toEqual({ error: "out_of_context" });
  });

  it("painel do admin: só as datas na URL", async () => {
    const fn = stub(overview());
    expect((await getSignatureOverview({ from: "2026-09-08", to: "2026-10-08" })).documents_by_mode.digital).toBe(42);
    expect(call(fn)[0]).toBe("/signature/admin/overview?from=2026-09-08&to=2026-10-08");
  });
});
```

E o rótulo do interruptor, em `src/lib/features.test.ts` (o 19a já acrescentou as duas linhas de `clinical_record`):

```diff
--- a/src/lib/features.test.ts
+++ b/src/lib/features.test.ts
@@ describe("features da sessão (contratos §1)", () => {
     expect(featureLabel("clinical_record")).toBe("Prontuário da atenção primária");
     expect(hasFeature({ features: [ "clinical_record" ] }, "clinical_record")).toBe(true);
+    expect(featureLabel("digital_signature")).toBe("Assinatura digital");
+    expect(hasFeature({ features: [ "digital_signature" ] }, "digital_signature")).toBe(true);
     expect(featureLabel("rnds_sync")).toBe("rnds_sync");
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod19b && npx vitest run src/lib/api.signature.test.ts src/lib/features.test.ts`
Expected: FAIL — `getCurrentCertificate is not a function` (e as outras funções novas) e `expected 'digital_signature' to be 'Assinatura digital'`.

- [ ] **Step 3: Write minimal implementation**

Em `src/lib/api.ts`, o bloco nos tipos do 19a:

```diff
--- a/src/lib/api.ts
+++ b/src/lib/api.ts
@@ export interface AddendumChanges { evaluated_problems?: EvaluatedProblem[]; conducts?: string[]; exam_requests?: ExamRequest[] }
-export interface Addendum { id: string; author_name: string; created_at: string; reason: string; text: string; changes: AddendumChanges | null }
+export interface Addendum {
+  id: string; author_name: string; created_at: string; reason: string; text: string; changes: AddendumChanges | null;
+  // Módulo 19b (contrato 2026-10-08 §2): calculado pelo api; ausente em api sem o 19b.
+  signature?: SignatureBlock;
+}
@@ export interface Consultation {
   evaluated_problems: EvaluatedProblem[]; conducts: string[]; exam_requests: ExamRequest[];
   started_at: string; finalized_at: string | null; addenda: Addendum[];
+  // Módulo 19b (contrato 2026-10-08 §2): só na finalizada; o rascunho não tem.
+  signature?: SignatureBlock;
 }
```

E a seção nova no **fim** do arquivo, depois da seção do 19a:

```ts
// ─── Assinatura digital (módulo 19b, ADR 0032; contrato 2026-10-08 §1–§7) ───
// As rotas levam só ids; `state`, `code`, motivo e `return_to` vão no corpo.
// O token do prestador e o code_verifier nunca passam pelo navegador: o api
// guarda os dois; o dashboard só segue o authorize_url e devolve state+code.

const SIGNATURE_BASE = import.meta.env.VITE_SIGNATURE_BASE || "/signature";

export type SignatureProvider = "vidaas" | "birdid" | "safeid" | "neoid" | "remoteid";
export type SignatureMode = "digital" | "manual" | "pending";
export type SignatureRequestStatus = "pending" | "signed" | "failed" | "returned_to_paper";
export type SignatureVerification = "valid" | "invalid" | "indeterminate";
export type SignatureReason =
  | "no_session" | "session_expired" | "provider_unavailable" | "provider_rejected" | "signer_unavailable"
  | "verification_failed" | "certificate_expired" | "certificate_revoked" | "feature_disabled" | "user_request";
export type SignatureDocumentType = "consultation" | "consultation_addendum";
export type CertificateStatus = "active" | "replaced" | "unlinked" | "revoked" | "expired";
export type SignatureFileKind = "pdf" | "package";

export interface SignatureBlock {
  mode: SignatureMode; request_id?: string; signature_id?: string; signed_at?: string; signer_name?: string;
  verification?: SignatureVerification; reason_code?: SignatureReason;
}
export interface SignerCertificate {
  id: string; provider: SignatureProvider; issuer: string; serial_number: string; not_after: string;
  status: CertificateStatus; expires_in_days: number;
}
export interface CertificateDiscovery {
  providers: { provider: SignatureProvider; found: boolean }[]; unavailable: SignatureProvider[];
}
export interface AuthorizeRedirect { authorize_url: string }
export interface SignatureSessionState { active: boolean; expires_at?: string; provider?: SignatureProvider }
export interface BatchResult { signed: number; failed: { request_id: string; reason_code: SignatureReason }[] }
// `return_to`: Divergência D2 (o api devolve o caminho que guardou com o state).
export type OAuthCallbackResult =
  | { purpose: "link"; result: SignerCertificate; return_to?: string }
  | { purpose: "session"; result: { expires_at: string }; return_to?: string }
  | { purpose: "batch"; result: BatchResult; return_to?: string };
export interface SignatureRequest {
  id: string; document_type: SignatureDocumentType; document_id: string; consultation_id: string;
  patient_display_name: string | null; finalized_at: string; status: SignatureRequestStatus;
  reason_code?: SignatureReason | null; attempts: number;
}
export interface SignatureDetail {
  id: string; document_type: SignatureDocumentType; document_id: string; signed_at: string; signer_name: string;
  signer_cpf_masked: string; policy: string; verification: SignatureVerification; verification_reasons: string[];
  verified_at: string; content: Record<string, unknown>;
}
export interface OverviewProfessional {
  user_id: string; name: string; certificate_status: "active" | "none" | "expiring";
  not_after?: string | null; pending_count: number; oldest_pending_at?: string | null;
}
export interface OverviewSignatureRow {
  signature_id: string; document_type: SignatureDocumentType; signer_name: string;
  verification: SignatureVerification; verified_at: string;
}
export interface SignatureOverview {
  professionals: OverviewProfessional[];
  documents_by_mode: Record<SignatureMode, number>;
  invalid_or_indeterminate: OverviewSignatureRow[];
}

const signaturePath = (...parts: string[]) => [ SIGNATURE_BASE, ...parts.map(encodeURIComponent) ].join("/");

export async function getCurrentCertificate(): Promise<SignerCertificate | null> {
  try {
    return await jsonFetch<SignerCertificate>(signaturePath("certificates", "current"));
  } catch (err) {
    if (err instanceof ApiError && err.status === 404 && errorCode(err) === "certificate_not_linked") return null;
    throw err;
  }
}

export function discoverCertificates(): Promise<CertificateDiscovery> {
  return jsonFetch(signaturePath("certificates", "discover"), postProfessional({}));
}

// Step-up: quem trata 401 mfa_required é o SensitiveAction.
export function linkCertificate(provider: SignatureProvider, returnTo: string): Promise<AuthorizeRedirect> {
  return jsonFetch(signaturePath("certificates", "link"), postProfessional({ provider, return_to: returnTo }));
}

export async function unlinkCertificate(): Promise<void> {
  await jsonFetch<void>(signaturePath("certificates", "current"), { method: "DELETE" });
}

export function openSignatureSession(returnTo: string): Promise<AuthorizeRedirect> {
  return jsonFetch(signaturePath("sessions"), postProfessional({ return_to: returnTo }));
}

export function getSignatureSession(): Promise<SignatureSessionState> {
  return jsonFetch(signaturePath("sessions", "current"));
}

export async function closeSignatureSession(): Promise<void> {
  await jsonFetch<void>(signaturePath("sessions", "current"), { method: "DELETE" });
}

export function completeSignatureOAuth(state: string, code: string): Promise<OAuthCallbackResult> {
  return jsonFetch(signaturePath("oauth", "callback"), postProfessional({ state, code }));
}

export async function listPendingSignatures(): Promise<SignatureRequest[]> {
  const payload = await jsonFetch<{ items: SignatureRequest[] }>(`${signaturePath("requests")}?status=pending`);
  return payload.items;
}

export function returnToPaper(id: string, reason: string): Promise<SignatureRequest> {
  return jsonFetch(signaturePath("requests", id, "return_to_paper"), postProfessional({ reason }));
}

export function startSignatureBatch(returnTo: string, requestIds?: string[]): Promise<AuthorizeRedirect & { count: number }> {
  const body = requestIds ? { request_ids: requestIds, return_to: returnTo } : { return_to: returnTo };
  return jsonFetch(signaturePath("batches"), postProfessional(body));
}

// Ler gera a trilha `clinical_record.viewed` no api.
export function getSignature(id: string): Promise<SignatureDetail> {
  return jsonFetch(signaturePath("signatures", id));
}

export function verifySignature(id: string): Promise<SignatureDetail> {
  return jsonFetch(signaturePath("signatures", id, "verify"), postProfessional({}));
}

// PDF e pacote lidos com fetch (e não por navegação) para a tela mostrar a
// recusa (403 out_of_context/opening_required) em vez de uma página de erro.
async function binaryFetch(url: string, accept: string): Promise<Blob> {
  const res = await fetch(url, { credentials: "include", headers: { Accept: accept } });
  if (!res.ok) {
    const text = await res.text().catch(() => "");
    let body: unknown = text;
    if (text) { try { body = JSON.parse(text); } catch { /* deixa string */ } }
    throw new ApiError(res.status, body, `${res.status} on ${url}`);
  }
  return res.blob();
}

export function fetchSignatureFile(id: string, kind: SignatureFileKind): Promise<Blob> {
  return binaryFetch(signaturePath("signatures", id, kind), kind === "pdf" ? "application/pdf" : "application/zip");
}

export function getSignatureOverview(q: { from: string; to: string }): Promise<SignatureOverview> {
  const params = new URLSearchParams({ from: q.from, to: q.to });
  return jsonFetch(`${signaturePath("admin", "overview")}?${params.toString()}`);
}
```

Em `src/lib/features.ts` (sobre o estado do 19a):

```diff
--- a/src/lib/features.ts
+++ b/src/lib/features.ts
@@
-export type FeatureKey = "ledi_export" | "cadsus_lookup" | "clinical_record";
+export type FeatureKey = "ledi_export" | "cadsus_lookup" | "clinical_record" | "digital_signature";
@@ const FEATURE_LABEL: Record<string, string> = {
   cadsus_lookup: "Consulta ao CADSUS na validação presencial",
-  clinical_record: "Prontuário da atenção primária"
+  clinical_record: "Prontuário da atenção primária",
+  digital_signature: "Assinatura digital"
 };
```

Em `vite.config.ts` (sobre o estado do 19a):

```diff
--- a/vite.config.ts
+++ b/vite.config.ts
@@
 //   /clinical_record → prontuário fora do atendimento (módulo 19: abertura justificada e relatório).
+//   /signature  → assinatura digital (módulo 19b). O prestador devolve a pessoa a
+//                 /dashboard/signature/callback (base do Vite), que NÃO casa com este
+//                 prefixo: o retorno é do dashboard, e só o POST /signature/oauth/callback vai ao api.
 //
@@
       "/production": proxy(TARGET),
-      "/clinical_record": proxy(TARGET)
+      "/clinical_record": proxy(TARGET),
+      "/signature": proxy(TARGET)
     }
```

Em `README.md` (sobre o estado do 19a):

```diff
--- a/README.md
+++ b/README.md
@@
 `/auth`, `/setup`, `/protocols`, `/mfa`, `/attendance`, `/professionals`, `/territory`, `/campaigns`, `/triage_catalog`,
-`/integrations`, `/cnes`, `/production` e `/clinical_record` para
+`/integrations`, `/cnes`, `/production`, `/clinical_record` e `/signature` para
 `VITE_API_PROXY_TARGET` com `changeOrigin: false`. **Não troque para `true`**:
```

E, logo depois desse parágrafo:

```markdown
O retorno do prestador de assinatura (módulo 19b) chega em
`/dashboard/signature/callback?state=…&code=…`, servido pelo próprio Vite (a
base é `/dashboard/`); a tela limpa a URL e troca o código no api por
`POST /signature/oauth/callback`. O `redirect_uri` cadastrado no prestador
(e no PSC falso de dev) é esse endereço no host da cidade.
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod19b && npx vitest run src/lib/api.signature.test.ts src/lib/features.test.ts && npx tsc --noEmit`
Expected: PASS (9 testes no cliente; `features.test.ts` verde); `tsc` limpo.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19b add src/lib/api.ts src/lib/api.signature.test.ts src/test/signatureFixtures.ts src/lib/features.ts src/lib/features.test.ts vite.config.ts README.md
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19b status --short
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19b commit -m "feat: add the digital signature api client and contract types

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 2: Regras de tela da assinatura

**Files:**
- Create: `src/lib/signature.ts`
- Test: `src/lib/signature.test.ts`

**Interfaces:**
- Consumes: tipos e `errorCode` (Task 1; `api.ts`); `describeActionError` (`src/lib/actionErrors.ts`); `hasFeature`, `featureDisabledKey` (`src/lib/features.ts`); `cityDateFormat`, `cityIsoDate`, `fmtDateTime`, `fmtHourMinute` (`src/lib/format.ts`); `Tone` (`src/theme/tokens.ts`).
- Produces (em `src/lib/signature.ts`):
  ```ts
  export const CERTIFICATE_KEY = "signatureCertificate", SESSION_KEY = "signatureSession", PENDING_KEY = "signaturePending",
               SIGNATURE_KEY = "signatureDetail", OVERVIEW_KEY = "signatureOverview";
  export const RETURN_TO = { account: "/signature", pending: "/signature-pending" } as const;
  export const EXPIRING_DAYS = 30, RETURN_REASON_MIN = 10, BATCH_LIMIT = 50, OVERDUE_MS = 86_400_000;
  export const SIGNATURE_DISABLED: string;           // "a assinatura digital está desligada nesta cidade"
  export const CERTIFICATE_HELP: string;             // CFM / COFEN
  export const RETURNED_TO_PAPER: string;            // frase depois da volta ao papel
  export function canSign(user): boolean;            // !operador && health_professional && digital_signature
  export function canSeeSignatureOverview(user): boolean; // !operador && municipal_admin && digital_signature
  export function providerLabel(p: string): string;
  export function fmtDay(iso?: string | null): string;          // "15/03/2027" no fuso da cidade
  export interface Notice { tone: Tone; text: string }
  export function certificateNotice(c: SignerCertificate): Notice | null;
  export function sessionBadge(s: SignatureSessionState | undefined, nowMs: number): { active: boolean; label: string };
  export function reasonLabel(code?: string | null): string;
  export function documentLabel(t: string): string;             // "consulta" | "adendo" | cru
  export interface Marker { label: string; tone: Tone; detail: string | null }
  export function signatureMarker(b: SignatureBlock): Marker;
  export function printHint(b: SignatureBlock | undefined, hasAddenda: boolean): string | null;
  export function returnToPaperProblem(reason: string): string | null;
  export function signatureError(err: unknown): string;
  export function batchSummary(r: BatchResult): string;
  export function callbackSummary(r: OAuthCallbackResult): string;
  export function callbackLanding(r: OAuthCallbackResult): string;   // caminho "/<módulo>"
  export function oauthErrorPhrase(error: string | null): string;
  export interface SignatureCallbackParams { state: string | null; code: string | null; error: string | null }
  export function readSignatureCallback(pathname?: string, search?: string, base?: string): SignatureCallbackParams | null;
  export function clearSignatureCallbackFromUrl(base?: string): void;
  export function signatureFileName(id: string, kind: SignatureFileKind): string;
  export const CERTIFICATE_STATUS_VIEW, VERIFICATION_VIEW;           // { label, tone } por valor
  export function isPendingOverdue(oldest: string | null | undefined, nowMs: number): boolean;
  export function overviewSummary(o: SignatureOverview, nowMs: number): { withCertificate: number; withoutCertificate: number; expiring: number; pendingOverdue: number };
  export function defaultOverviewPeriod(nowMs: number): { from: string; to: string };
  export function goToProvider(url: string): void;              // window.location.assign
  export function saveBlob(blob: Blob, filename: string): void; // <a download> e revoga em 60 s
  ```

- [ ] **Step 1: Write the failing test**

```ts
// src/lib/signature.test.ts
import { describe, expect, it } from "vitest";
import { ApiError } from "./api";
import {
  CERTIFICATE_STATUS_VIEW, SIGNATURE_DISABLED, VERIFICATION_VIEW, batchSummary, callbackLanding, callbackSummary,
  canSeeSignatureOverview, canSign, certificateNotice, defaultOverviewPeriod, documentLabel, isPendingOverdue,
  oauthErrorPhrase, overviewSummary, printHint, providerLabel, readSignatureCallback, reasonLabel, returnToPaperProblem,
  sessionBadge, signatureError, signatureFileName, signatureMarker
} from "./signature";
import { NOW19B, certificate, overview, signatureBlock } from "../test/signatureFixtures";

const NOW = Date.parse(NOW19B);
const user = (roles: string[], features: string[] = [ "digital_signature" ], operator = false) =>
  ({ operator, memberships: roles.map((role) => ({ role })), features });

describe("quem pode", () => {
  it("assinar: profissional com digital_signature, nunca operador", () => {
    expect(canSign(user([ "health_professional" ]))).toBe(true);
    expect(canSign(user([ "health_professional" ], [ "clinical_record" ]))).toBe(false);
    expect(canSign(user([ "health_professional" ], [ "digital_signature" ], true))).toBe(false);
    expect(canSign(user([ "municipal_admin", "citizen_verifier" ]))).toBe(false);
    expect(canSign(null)).toBe(false);
  });

  it("painel: municipal_admin com digital_signature, nunca operador", () => {
    expect(canSeeSignatureOverview(user([ "municipal_admin" ]))).toBe(true);
    expect(canSeeSignatureOverview(user([ "municipal_admin" ], []))).toBe(false);
    expect(canSeeSignatureOverview(user([ "health_professional" ]))).toBe(false);
    expect(canSeeSignatureOverview(user([ "municipal_admin" ], [ "digital_signature" ], true))).toBe(false);
  });
});

describe("certificado", () => {
  it("rótulo do prestador; desconhecido sai cru", () => {
    expect(providerLabel("vidaas")).toBe("VIDaaS");
    expect(providerLabel("birdid")).toBe("BirdID");
    expect(providerLabel("novopsc")).toBe("novopsc");
  });

  it("aviso do certificado: longe, 30 dias, amanhã, hoje, vencido e revogado", () => {
    expect(certificateNotice(certificate())).toBeNull();
    expect(certificateNotice(certificate({ not_after: "2026-11-07T23:59:59-03:00", expires_in_days: 30 }))).toEqual(
      { tone: "warn", text: "o certificado vence em 30 dias (07/11/2026) — renove no prestador e vincule de novo" });
    expect(certificateNotice(certificate({ not_after: "2026-10-09T23:59:59-03:00", expires_in_days: 1 }))?.text)
      .toBe("o certificado vence amanhã (09/10/2026) — renove no prestador e vincule de novo");
    expect(certificateNotice(certificate({ not_after: "2026-10-08T23:59:59-03:00", expires_in_days: 0 }))?.text)
      .toBe("o certificado vence hoje (08/10/2026) — renove no prestador e vincule de novo");
    expect(certificateNotice(certificate({ not_after: "2026-10-06T23:59:59-03:00", expires_in_days: -2 }))).toEqual({
      tone: "down",
      text: "certificado vencido em 06/10/2026 — renove no prestador e vincule de novo; até lá, as consultas ficam pendentes"
    });
    expect(certificateNotice(certificate({ status: "expired", expires_in_days: 0, not_after: "2026-10-08T00:00:00-03:00" }))?.tone)
      .toBe("down");
    expect(certificateNotice(certificate({ status: "revoked" }))).toEqual({
      tone: "down", text: "certificado revogado — vincule outro certificado; até lá, as consultas ficam pendentes"
    });
  });
});

describe("sessão de assinatura", () => {
  it("ativa só enquanto expires_at está no futuro", () => {
    expect(sessionBadge({ active: true, expires_at: "2026-10-08T18:40:00-03:00" }, NOW))
      .toEqual({ active: true, label: "assinatura ativa até 18:40" });
    expect(sessionBadge({ active: true, expires_at: "2026-10-08T09:59:00-03:00" }, NOW))
      .toEqual({ active: false, label: "sem sessão de assinatura" });
    expect(sessionBadge({ active: false }, NOW).active).toBe(false);
    expect(sessionBadge({ active: true }, NOW).active).toBe(false);
    expect(sessionBadge(undefined, NOW).label).toBe("sem sessão de assinatura");
  });
});

describe("marcador, motivos e impresso", () => {
  it("digital: válida, inválida, indeterminada", () => {
    expect(signatureMarker(signatureBlock())).toEqual(
      { label: "assinada digitalmente", tone: "ok", detail: "Helena Prado · 07/10/2026, 10:21" });
    expect(signatureMarker(signatureBlock({ verification: "invalid" }))).toMatchObject({ label: "assinatura digital inválida", tone: "down" });
    expect(signatureMarker(signatureBlock({ verification: "indeterminate" }))).toMatchObject(
      { label: "assinatura digital indeterminada", tone: "warn" });
  });

  it("pendente com e sem motivo; à mão com o motivo da volta ao papel", () => {
    expect(signatureMarker({ mode: "pending", request_id: "sr1", reason_code: "no_session" })).toEqual(
      { label: "assinatura pendente", tone: "warn", detail: "sem sessão de assinatura aberta" });
    expect(signatureMarker({ mode: "pending", request_id: "sr1" }).detail).toBe("assinando…");
    expect(signatureMarker({ mode: "manual", reason_code: "feature_disabled" })).toEqual(
      { label: "assinatura à mão (papel)", tone: "neutral", detail: "assinatura digital desligada na cidade" });
    expect(signatureMarker({ mode: "manual" }).detail).toBeNull();
  });

  it("motivo desconhecido sai cru; ausente vira travessão", () => {
    expect(reasonLabel("session_expired")).toBe("a sessão de assinatura venceu");
    expect(reasonLabel("quota_exceeded")).toBe("quota_exceeded");
    expect(reasonLabel(null)).toBe("—");
  });

  it("documento e nome do arquivo baixado (só o id)", () => {
    expect(documentLabel("consultation")).toBe("consulta");
    expect(documentLabel("consultation_addendum")).toBe("adendo");
    expect(documentLabel("prescription")).toBe("prescription");
    expect(signatureFileName("sg1", "pdf")).toBe("assinatura-sg1.pdf");
    expect(signatureFileName("sg1", "package")).toBe("assinatura-sg1.zip");
  });

  it("aviso do impresso", () => {
    expect(printHint(undefined, false)).toBeNull();
    expect(printHint(signatureBlock(), false)).toBe("o impresso é o PDF assinado digitalmente");
    expect(printHint(signatureBlock(), true))
      .toBe("com adendo, o impresso é o do prontuário; os documentos assinados estão em “Ver o que foi assinado”");
    expect(printHint({ mode: "pending" }, false)).toBe("o impresso sai com espaço para assinatura à mão");
    expect(printHint({ mode: "manual" }, true)).toBe("o impresso sai com espaço para assinatura à mão");
  });

  it("motivo da volta ao papel: ao menos 10 caracteres sem os espaços das pontas", () => {
    expect(returnToPaperProblem("curto")).toBe("descreva o motivo com pelo menos 10 caracteres");
    expect(returnToPaperProblem("   nove 99   ")).toBe("descreva o motivo com pelo menos 10 caracteres");
    expect(returnToPaperProblem("paciente pediu o papel")).toBeNull();
  });
});

describe("frases das recusas", () => {
  const err = (status: number, body: unknown) => new ApiError(status, body, String(status));

  it("códigos do contrato viram frases; nunca o código cru", () => {
    expect(signatureError(err(422, { error: "certificate_cpf_mismatch" })))
      .toBe("o certificado autorizado não é do seu CPF — escolha no prestador o certificado em seu nome");
    expect(signatureError(err(409, { error: "certificate_not_linked" })))
      .toBe("vincule um certificado em Conta → Assinatura digital antes de assinar");
    expect(signatureError(err(422, { error: "invalid_state" })))
      .toBe("este retorno do prestador não vale mais (já usado ou vencido) — comece de novo");
    expect(signatureError(err(409, { error: "not_pending" }))).toBe("este documento não está mais pendente — a lista foi atualizada");
    expect(signatureError(err(503, { error: "provider_unavailable" }))).toBe("o prestador não respondeu — tente de novo em alguns minutos");
  });

  it("interruptor: o da assinatura e o do prontuário têm frases próprias", () => {
    expect(signatureError(err(403, { error: "feature_disabled", feature: "digital_signature" }))).toBe(SIGNATURE_DISABLED);
    expect(signatureError(err(403, { error: "feature_disabled", feature: "clinical_record" })))
      .toBe("o prontuário está desligado nesta cidade");
  });

  it("o resto cai na tradução comum", () => {
    expect(signatureError(err(500, "boom"))).toBe("não foi possível concluir — tente de novo");
    expect(signatureError(new Error("rede"))).toBe("não foi possível concluir — tente de novo");
    expect(signatureError(err(401, { error: "unauthenticated" }))).toBe("sessão expirada — entre de novo");
  });
});

describe("retorno do prestador", () => {
  it("lê state, code e error só na rota de retorno, sob a base do dashboard", () => {
    expect(readSignatureCallback("/dashboard/signature/callback", "?state=st1&code=c1", "/dashboard/"))
      .toEqual({ state: "st1", code: "c1", error: null });
    expect(readSignatureCallback("/dashboard/signature/callback/", "?state=st1&error=access_denied", "/dashboard/"))
      .toEqual({ state: "st1", code: null, error: "access_denied" });
    expect(readSignatureCallback("/dashboard/", "?state=st1&code=c1", "/dashboard/")).toBeNull();
    expect(readSignatureCallback("/signature/callback", "?state=st1&code=c1", "/dashboard/")).toBeNull();
  });

  it("erro do prestador vira frase", () => {
    expect(oauthErrorPhrase("access_denied")).toBe("a autorização foi negada no prestador — nada foi alterado");
    expect(oauthErrorPhrase(null)).toBe("o prestador não concluiu a autorização — comece de novo");
  });

  it("resumo do que voltou", () => {
    expect(callbackSummary({ purpose: "link", result: certificate() }))
      .toBe("Certificado VIDaaS vinculado, válido até 15/03/2027.");
    expect(callbackSummary({ purpose: "session", result: { expires_at: "2026-10-08T22:00:00-03:00" } }))
      .toBe("Sessão de assinatura aberta até 22:00.");
    expect(batchSummary({ signed: 3, failed: [] })).toBe("3 documentos assinados.");
    expect(batchSummary({ signed: 1, failed: [] })).toBe("1 documento assinado.");
    expect(batchSummary({ signed: 2, failed: [ { request_id: "sr9", reason_code: "provider_unavailable" } ] }))
      .toBe("2 documentos assinados; 1 não assinado — fica em Pendentes de assinatura.");
    expect(batchSummary({ signed: 0, failed: [
      { request_id: "sr8", reason_code: "provider_rejected" }, { request_id: "sr9", reason_code: "provider_rejected" } ] }))
      .toBe("nenhum documento assinado; 2 não assinados — ficam em Pendentes de assinatura.");
  });

  it("destino: o return_to devolvido, senão o padrão do propósito", () => {
    expect(callbackLanding({ purpose: "session", result: { expires_at: "x" }, return_to: "/attendance" })).toBe("/attendance");
    expect(callbackLanding({ purpose: "link", result: certificate() })).toBe("/signature");
    expect(callbackLanding({ purpose: "batch", result: { signed: 1, failed: [] } })).toBe("/signature-pending");
    expect(callbackLanding({ purpose: "session", result: { expires_at: "x" } })).toBe("/overview");
    expect(callbackLanding({ purpose: "session", result: { expires_at: "x" }, return_to: "//evil.example" })).toBe("/overview");
  });
});

describe("painel do admin", () => {
  it("pendente há mais de 24 h", () => {
    expect(isPendingOverdue("2026-10-06T16:00:00-03:00", NOW)).toBe(true);
    expect(isPendingOverdue("2026-10-07T10:30:00-03:00", NOW)).toBe(false);
    expect(isPendingOverdue(null, NOW)).toBe(false);
  });

  it("resumo e período padrão de 30 dias no fuso da cidade", () => {
    expect(overviewSummary(overview(), NOW)).toEqual({ withCertificate: 2, withoutCertificate: 1, expiring: 1, pendingOverdue: 1 });
    expect(defaultOverviewPeriod(NOW)).toEqual({ from: "2026-09-08", to: "2026-10-08" });
  });

  it("rótulos de estado", () => {
    expect(CERTIFICATE_STATUS_VIEW.expiring).toEqual({ label: "vence em até 30 dias", tone: "warn" });
    expect(VERIFICATION_VIEW.indeterminate).toEqual({ label: "indeterminada", tone: "warn" });
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod19b && npx vitest run src/lib/signature.test.ts`
Expected: FAIL — `Failed to resolve import "./signature"`.

- [ ] **Step 3: Write minimal implementation**

```ts
// src/lib/signature.ts
// Regras de tela da assinatura digital (módulo 19b; spec §5, §9 e §10; ADR
// 0032; contrato 2026-10-08). Quem decide é o api: aqui só se traduz o que ele
// devolveu e se avisa antes de enviar. Nada clínico, nenhum token e nenhum
// `code` passa por aqui além do que a tela mostra ou devolve ao api.
import {
  errorCode,
  type BatchResult, type OAuthCallbackResult, type OverviewProfessional, type SignatureBlock, type SignatureFileKind,
  type SignatureOverview, type SignatureSessionState, type SignatureVerification, type SignerCertificate
} from "./api";
import { describeActionError } from "./actionErrors";
import { featureDisabledKey, hasFeature } from "./features";
import { cityDateFormat, cityIsoDate, fmtDateTime, fmtHourMinute } from "./format";
import type { Tone } from "../theme/tokens";

export const CERTIFICATE_KEY = "signatureCertificate";
export const SESSION_KEY = "signatureSession";
export const PENDING_KEY = "signaturePending";
export const SIGNATURE_KEY = "signatureDetail";
export const OVERVIEW_KEY = "signatureOverview";

// Caminhos `/<id do módulo>` que o dashboard manda como return_to (Divergência D3).
export const RETURN_TO = { account: "/signature", pending: "/signature-pending" } as const;

export const EXPIRING_DAYS = 30;
export const RETURN_REASON_MIN = 10;
export const BATCH_LIMIT = 50;
export const OVERDUE_MS = 24 * 60 * 60 * 1000;

export const SIGNATURE_DISABLED = "a assinatura digital está desligada nesta cidade";
export const CERTIFICATE_HELP =
  "Médicos têm certificado em nuvem gratuito pelo CFM. Para a enfermagem, o certificado é fornecido pelo município (COFEN 754/2024).";
export const RETURNED_TO_PAPER = "voltou ao papel — imprima a consulta e assine à mão";

const GENERIC = "não foi possível concluir — tente de novo";

// ─── Quem pode ────────────────────────────────────────────────────────────────

type SessionLike = { operator?: boolean; memberships?: { role: string }[]; features?: unknown } | null | undefined;
const rolesOf = (user: SessionLike) => user?.memberships?.map((m) => m.role) ?? [];

export function canSign(user: SessionLike): boolean {
  return !!user && !user.operator && rolesOf(user).includes("health_professional") && hasFeature(user, "digital_signature");
}

export function canSeeSignatureOverview(user: SessionLike): boolean {
  return !!user && !user.operator && rolesOf(user).includes("municipal_admin") && hasFeature(user, "digital_signature");
}

// ─── Certificado e sessão ─────────────────────────────────────────────────────

const PROVIDER_LABEL: Record<string, string> = {
  vidaas: "VIDaaS", birdid: "BirdID", safeid: "SafeID", neoid: "NeoID", remoteid: "RemoteID"
};

export function providerLabel(provider: string): string {
  return PROVIDER_LABEL[provider] ?? provider;
}

const DAY: Intl.DateTimeFormatOptions = { day: "2-digit", month: "2-digit", year: "numeric" };

export function fmtDay(iso: string | null | undefined): string {
  if (!iso) return "—";
  const d = new Date(iso);
  return Number.isNaN(d.getTime()) ? "—" : cityDateFormat(DAY).format(d);
}

export interface Notice { tone: Tone; text: string }

const RENEW = "renove no prestador e vincule de novo";
const UNTIL_THEN = "até lá, as consultas ficam pendentes";

export function certificateNotice(cert: SignerCertificate): Notice | null {
  if (cert.status === "revoked") {
    return { tone: "down", text: `certificado revogado — vincule outro certificado; ${UNTIL_THEN}` };
  }
  if (cert.status === "expired" || cert.expires_in_days < 0) {
    return { tone: "down", text: `certificado vencido em ${fmtDay(cert.not_after)} — ${RENEW}; ${UNTIL_THEN}` };
  }
  if (cert.expires_in_days <= EXPIRING_DAYS) {
    const when = cert.expires_in_days === 0 ? "vence hoje"
      : cert.expires_in_days === 1 ? "vence amanhã" : `vence em ${cert.expires_in_days} dias`;
    return { tone: "warn", text: `o certificado ${when} (${fmtDay(cert.not_after)}) — ${RENEW}` };
  }
  return null;
}

// Previsão a partir do último GET: a sessão vale até `expires_at`, e o selo
// muda sozinho quando o relógio passa dele (o api é quem decide de verdade).
export function sessionBadge(session: SignatureSessionState | undefined, nowMs: number): { active: boolean; label: string } {
  const until = session?.active && session.expires_at ? Date.parse(session.expires_at) : Number.NaN;
  if (!Number.isNaN(until) && until > nowMs) {
    return { active: true, label: `assinatura ativa até ${fmtHourMinute(session!.expires_at)}` };
  }
  return { active: false, label: "sem sessão de assinatura" };
}

// ─── Marcador, motivos e impresso ─────────────────────────────────────────────

const REASON_LABEL: Record<string, string> = {
  no_session: "sem sessão de assinatura aberta",
  session_expired: "a sessão de assinatura venceu",
  provider_unavailable: "o prestador não respondeu",
  provider_rejected: "o prestador recusou a assinatura",
  signer_unavailable: "o serviço de assinatura não respondeu",
  verification_failed: "a assinatura não passou na verificação",
  certificate_expired: "certificado vencido",
  certificate_revoked: "certificado revogado",
  feature_disabled: "assinatura digital desligada na cidade",
  user_request: "voltou ao papel a pedido do autor"
};

export function reasonLabel(code: string | null | undefined): string {
  if (!code) return "—";
  return REASON_LABEL[code] ?? code;
}

const DOCUMENT_LABEL: Record<string, string> = { consultation: "consulta", consultation_addendum: "adendo" };

export function documentLabel(type: string): string {
  return DOCUMENT_LABEL[type] ?? type;
}

export interface Marker { label: string; tone: Tone; detail: string | null }

export function signatureMarker(block: SignatureBlock): Marker {
  if (block.mode === "digital") {
    const who = [ block.signer_name, block.signed_at ? fmtDateTime(block.signed_at) : null ].filter(Boolean).join(" · ") || null;
    if (block.verification === "invalid") return { label: "assinatura digital inválida", tone: "down", detail: who };
    if (block.verification === "indeterminate") return { label: "assinatura digital indeterminada", tone: "warn", detail: who };
    return { label: "assinada digitalmente", tone: "ok", detail: who };
  }
  if (block.mode === "pending") {
    return { label: "assinatura pendente", tone: "warn", detail: block.reason_code ? reasonLabel(block.reason_code) : "assinando…" };
  }
  return { label: "assinatura à mão (papel)", tone: "neutral", detail: block.reason_code ? reasonLabel(block.reason_code) : null };
}

// O api decide qual impresso sai (contrato §6): PDF assinado quando a consulta
// é digital e não tem adendo depois; o impresso do 19a nos demais casos.
export function printHint(block: SignatureBlock | undefined, hasAddenda: boolean): string | null {
  if (!block) return null;
  if (block.mode === "digital" && !hasAddenda) return "o impresso é o PDF assinado digitalmente";
  if (block.mode === "digital") {
    return "com adendo, o impresso é o do prontuário; os documentos assinados estão em “Ver o que foi assinado”";
  }
  return "o impresso sai com espaço para assinatura à mão";
}

export function returnToPaperProblem(reason: string): string | null {
  return reason.trim().length < RETURN_REASON_MIN ? `descreva o motivo com pelo menos ${RETURN_REASON_MIN} caracteres` : null;
}

export function signatureFileName(id: string, kind: SignatureFileKind): string {
  return `assinatura-${id}.${kind === "pdf" ? "pdf" : "zip"}`;
}

// ─── Frases das recusas ───────────────────────────────────────────────────────

const ERRORS: Record<string, string> = {
  certificate_not_linked: "vincule um certificado em Conta → Assinatura digital antes de assinar",
  certificate_not_found: "nenhum certificado em nuvem encontrado para o seu CPF",
  certificate_cpf_mismatch: "o certificado autorizado não é do seu CPF — escolha no prestador o certificado em seu nome",
  certificate_expired: "o certificado está vencido — renove no prestador e vincule de novo",
  certificate_revoked: "o certificado foi revogado — vincule outro certificado",
  provider_unavailable: "o prestador não respondeu — tente de novo em alguns minutos",
  authorization_denied: "a autorização foi negada no prestador — nada foi alterado",
  authorization_expired: "a autorização demorou demais e venceu — comece de novo",
  invalid_state: "este retorno do prestador não vale mais (já usado ou vencido) — comece de novo",
  invalid_provider: "este prestador não está habilitado — escolha outro",
  not_author: "só quem escreveu o documento pode assiná-lo ou voltá-lo ao papel",
  already_signed: "este documento já foi assinado",
  not_pending: "este documento não está mais pendente — a lista foi atualizada",
  nothing_pending: "não há documentos pendentes — a lista foi atualizada",
  invalid_reason: `descreva o motivo com pelo menos ${RETURN_REASON_MIN} caracteres`,
  out_of_context: "fora do atendimento, abra o prontuário com motivo para ver o que foi assinado",
  opening_required: "a abertura justificada terminou — abra o prontuário de novo",
  missing_role: "seu papel não permite esta ação"
};

export function signatureError(err: unknown): string {
  const feature = featureDisabledKey(err);
  if (feature === "clinical_record") return "o prontuário está desligado nesta cidade";
  if (feature !== null) return SIGNATURE_DISABLED;
  const code = errorCode(err);
  if (code && ERRORS[code]) return ERRORS[code];
  const described = describeActionError(err);
  return "message" in described ? described.message : GENERIC;
}

// ─── Retorno do prestador ─────────────────────────────────────────────────────

const plural = (n: number, one: string, many: string) => (n === 1 ? one : many);

export function batchSummary(result: BatchResult): string {
  const n = result.signed;
  const signed = n === 0 ? "nenhum documento assinado" : `${n} ${plural(n, "documento assinado", "documentos assinados")}`;
  const m = result.failed.length;
  if (m === 0) return `${signed}.`;
  return `${signed}; ${m} ${plural(m, "não assinado — fica", "não assinados — ficam")} em Pendentes de assinatura.`;
}

export function callbackSummary(result: OAuthCallbackResult): string {
  if (result.purpose === "link") {
    return `Certificado ${providerLabel(result.result.provider)} vinculado, válido até ${fmtDay(result.result.not_after)}.`;
  }
  if (result.purpose === "session") return `Sessão de assinatura aberta até ${fmtHourMinute(result.result.expires_at)}.`;
  return batchSummary(result.result);
}

const DEFAULT_LANDING: Record<OAuthCallbackResult["purpose"], string> = {
  link: RETURN_TO.account, session: "/overview", batch: RETURN_TO.pending
};

export function callbackLanding(result: OAuthCallbackResult): string {
  return result.return_to && /^\/[^/]/.test(result.return_to) ? result.return_to : DEFAULT_LANDING[result.purpose];
}

export function oauthErrorPhrase(error: string | null): string {
  return error === "access_denied"
    ? "a autorização foi negada no prestador — nada foi alterado"
    : "o prestador não concluiu a autorização — comece de novo";
}

export interface SignatureCallbackParams { state: string | null; code: string | null; error: string | null }

const CALLBACK_PATH = "signature/callback";

// Rota de retorno sob a base do dashboard (Divergência D1). Pura: a base vem
// por argumento (nos testes, import.meta.env.BASE_URL é "/").
export function readSignatureCallback(
  pathname: string = window.location.pathname,
  search: string = window.location.search,
  base: string = import.meta.env.BASE_URL
): SignatureCallbackParams | null {
  if (pathname.replace(/\/+$/, "") !== `${base}${CALLBACK_PATH}`) return null;
  const params = new URLSearchParams(search);
  const read = (key: string) => {
    const value = params.get(key);
    return value && value.trim() !== "" ? value : null;
  };
  return { state: read("state"), code: read("code"), error: read("error") };
}

// O `code` não pode ficar no histórico: troca a entrada pela base do dashboard.
export function clearSignatureCallbackFromUrl(base: string = import.meta.env.BASE_URL): void {
  window.history.replaceState({}, "", base);
}

// ─── Painel do admin ──────────────────────────────────────────────────────────

export const CERTIFICATE_STATUS_VIEW: Record<OverviewProfessional["certificate_status"], { label: string; tone: Tone }> = {
  active: { label: "ativo", tone: "ok" },
  expiring: { label: "vence em até 30 dias", tone: "warn" },
  none: { label: "sem certificado", tone: "neutral" }
};

export const VERIFICATION_VIEW: Record<SignatureVerification, { label: string; tone: Tone }> = {
  valid: { label: "válida", tone: "ok" },
  invalid: { label: "inválida", tone: "down" },
  indeterminate: { label: "indeterminada", tone: "warn" }
};

export function isPendingOverdue(oldest: string | null | undefined, nowMs: number): boolean {
  if (!oldest) return false;
  const at = Date.parse(oldest);
  return !Number.isNaN(at) && nowMs - at > OVERDUE_MS;
}

export function overviewSummary(o: SignatureOverview, nowMs: number) {
  const ps = o.professionals;
  return {
    withCertificate: ps.filter((p) => p.certificate_status !== "none").length,
    withoutCertificate: ps.filter((p) => p.certificate_status === "none").length,
    expiring: ps.filter((p) => p.certificate_status === "expiring").length,
    pendingOverdue: ps.filter((p) => isPendingOverdue(p.oldest_pending_at, nowMs)).length
  };
}

export function defaultOverviewPeriod(nowMs: number): { from: string; to: string } {
  return { from: cityIsoDate(new Date(nowMs - 30 * 86_400_000)), to: cityIsoDate(new Date(nowMs)) };
}

// ─── Navegador ────────────────────────────────────────────────────────────────

// Vai ao prestador na mesma aba: o retorno volta para /dashboard/signature/callback.
export function goToProvider(url: string): void {
  window.location.assign(url);
}

// Download pelo endereço `blob:`; o nome do arquivo leva só o id (nunca nome
// de paciente), e o endereço é revogado depois de 60 s.
export function saveBlob(blob: Blob, filename: string): void {
  const url = URL.createObjectURL(blob);
  const anchor = document.createElement("a");
  anchor.href = url;
  anchor.download = filename;
  anchor.rel = "noopener";
  document.body.appendChild(anchor);
  anchor.click();
  anchor.remove();
  setTimeout(() => URL.revokeObjectURL(url), 60_000);
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod19b && npx vitest run src/lib/signature.test.ts && npx tsc --noEmit`
Expected: PASS; `tsc` limpo.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19b add src/lib/signature.ts src/lib/signature.test.ts
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19b commit -m "feat: add the digital signature screen rules

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 3: Conta → Assinatura digital (vínculo, troca, desvínculo e aviso de vencimento)

**Files:**
- Create: `src/modules/SignatureAccount.tsx`
- Modify: `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/App.tsx`
- Test: `src/modules/SignatureAccount.test.tsx`

**Interfaces:**
- Consumes: `getCurrentCertificate`, `discoverCertificates`, `linkCertificate`, `unlinkCertificate`, tipos (Task 1); `CERTIFICATE_KEY`, `RETURN_TO`, `SIGNATURE_DISABLED`, `CERTIFICATE_HELP`, `canSign`, `certificateNotice`, `fmtDay`, `goToProvider`, `providerLabel`, `signatureError` (Task 2); `useAuth`, `hasFeature`, `SensitiveAction`, `PageHeader`, `Panel`, `KeyValue`, `EmptyState`, `formStyles`, `toneColor`.
- Produces:
  - `SignatureAccount({ onGoToSecurity?(): void; redirect?(url: string): void })` — página "Assinatura digital": `Panel` "Certificado" (prestador, emissor, número de série, válido até; aviso de vencimento; "Trocar certificado" e "Desvincular" com step-up) ou, sem certificado, "Você continua assinando no papel…" com a orientação e "Procurar meu certificado"; `Panel` "Procurar certificado pelo CPF" ("Procurar", lista "certificados encontrados" com "Vincular <prestador>", "não encontrado em: …", "sem resposta: …") e o `SensitiveAction` "Vincular certificado <prestador>" (step-up, `confirmLabel` "Ir ao prestador") que pede o `authorize_url` com `return_to: "/signature"` e chama `redirect`;
  - `ModuleId` com `"signature"`; item "Assinatura digital" no grupo Conta, só com `canSign(user)`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/SignatureAccount.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return {
    ...real, fetchCurrentSession: vi.fn(), stepUpMfa: vi.fn(), getCurrentCertificate: vi.fn(),
    discoverCertificates: vi.fn(), linkCertificate: vi.fn(), unlinkCertificate: vi.fn()
  };
});

import * as api from "../lib/api";
import { ApiError } from "../lib/api";
import { SignatureAccount } from "./SignatureAccount";
import { renderWithProviders } from "../test/campaignFixtures";
import { certificate, signer } from "../test/signatureFixtures";
import { CERTIFICATE_HELP, SIGNATURE_DISABLED } from "../lib/signature";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const PAPER = "Você continua assinando no papel: as consultas saem para impressão e assinatura à mão.";

afterEach(cleanup);

describe("SignatureAccount", () => {
  let redirect: ReturnType<typeof vi.fn>;

  beforeEach(() => {
    vi.clearAllMocks();
    redirect = vi.fn();
    mocked(api.fetchCurrentSession).mockResolvedValue(signer());
    mocked(api.getCurrentCertificate).mockResolvedValue(certificate());
    mocked(api.stepUpMfa).mockResolvedValue(undefined);
  });

  it("certificado vinculado: prestador, emissor, série e validade, sem aviso longe do vencimento", async () => {
    renderWithProviders(<SignatureAccount redirect={redirect} />);
    const panel = await screen.findByRole("region", { name: "Certificado" });
    expect(await within(panel).findByText("VIDaaS")).not.toBeNull();
    expect(within(panel).getByText("AC VALID RFB v5")).not.toBeNull();
    expect(within(panel).getByText("5A3F09")).not.toBeNull();
    expect(within(panel).getByText("15/03/2027")).not.toBeNull();
    expect(within(panel).queryByRole("status")).toBeNull();
    expect(within(panel).getByRole("button", { name: "Trocar certificado" })).not.toBeNull();
  });

  it("vence em 12 dias: avisa com a data", async () => {
    mocked(api.getCurrentCertificate).mockResolvedValue(
      certificate({ not_after: "2026-10-20T23:59:59-03:00", expires_in_days: 12 }));
    renderWithProviders(<SignatureAccount redirect={redirect} />);
    expect((await screen.findByRole("status")).textContent)
      .toBe("o certificado vence em 12 dias (20/10/2026) — renove no prestador e vincule de novo");
  });

  it("sem certificado: continua no papel, com a orientação, e procura pelo CPF", async () => {
    mocked(api.getCurrentCertificate).mockResolvedValue(null);
    mocked(api.discoverCertificates).mockResolvedValue({
      providers: [ { provider: "vidaas", found: true }, { provider: "birdid", found: false } ], unavailable: [ "safeid" ]
    });
    mocked(api.linkCertificate).mockResolvedValue({ authorize_url: "https://psc.example/authorize?x=1" });
    renderWithProviders(<SignatureAccount redirect={redirect} />);

    expect(await screen.findByText(PAPER)).not.toBeNull();
    expect(screen.getByText(CERTIFICATE_HELP)).not.toBeNull();
    fireEvent.click(screen.getByRole("button", { name: "Procurar meu certificado" }));
    fireEvent.click(await screen.findByRole("button", { name: "Procurar" }));

    const found = await screen.findByRole("list", { name: "certificados encontrados" });
    expect(screen.getByText("não encontrado em: BirdID")).not.toBeNull();
    expect(screen.getByText("sem resposta: SafeID — procure de novo em alguns minutos")).not.toBeNull();

    fireEvent.click(within(found).getByRole("button", { name: "Vincular VIDaaS" }));
    const confirm = await screen.findByRole("region", { name: "Vincular certificado VIDaaS" });
    fireEvent.click(within(confirm).getByRole("button", { name: "Ir ao prestador" }));

    await waitFor(() => expect(redirect).toHaveBeenCalledWith("https://psc.example/authorize?x=1"));
    expect(api.linkCertificate).toHaveBeenCalledWith("vidaas", "/signature");
  });

  it("janela de step-up fechada: pede o código antes de ir ao prestador", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(signer(undefined, { mfa_verified_at: null }));
    mocked(api.getCurrentCertificate).mockResolvedValue(null);
    mocked(api.discoverCertificates).mockResolvedValue({ providers: [ { provider: "vidaas", found: true } ], unavailable: [] });
    mocked(api.linkCertificate).mockResolvedValue({ authorize_url: "https://psc.example/authorize?x=2" });
    renderWithProviders(<SignatureAccount redirect={redirect} />);

    fireEvent.click(await screen.findByRole("button", { name: "Procurar meu certificado" }));
    fireEvent.click(await screen.findByRole("button", { name: "Procurar" }));
    fireEvent.click(await screen.findByRole("button", { name: "Vincular VIDaaS" }));
    const confirm = await screen.findByRole("region", { name: "Vincular certificado VIDaaS" });
    fireEvent.change(within(confirm).getByLabelText("Código do autenticador"), { target: { value: "123456" } });
    fireEvent.click(within(confirm).getByRole("button", { name: "Ir ao prestador" }));

    await waitFor(() => expect(redirect).toHaveBeenCalledWith("https://psc.example/authorize?x=2"));
    expect(api.stepUpMfa).toHaveBeenCalledWith("123456");
  });

  it("nenhum certificado encontrado: diz que continua no papel", async () => {
    mocked(api.getCurrentCertificate).mockResolvedValue(null);
    mocked(api.discoverCertificates).mockResolvedValue({
      providers: [ { provider: "vidaas", found: false }, { provider: "birdid", found: false } ], unavailable: []
    });
    renderWithProviders(<SignatureAccount redirect={redirect} />);
    fireEvent.click(await screen.findByRole("button", { name: "Procurar meu certificado" }));
    fireEvent.click(await screen.findByRole("button", { name: "Procurar" }));
    expect(await screen.findByText(
      "nenhum certificado em nuvem encontrado para o seu CPF nos prestadores habilitados — você continua no papel")).not.toBeNull();
    expect(screen.queryByRole("list", { name: "certificados encontrados" })).toBeNull();
  });

  it("desvincular pede confirmação com step-up e volta ao papel", async () => {
    mocked(api.getCurrentCertificate).mockResolvedValueOnce(certificate()).mockResolvedValue(null);
    mocked(api.unlinkCertificate).mockResolvedValue(undefined);
    renderWithProviders(<SignatureAccount redirect={redirect} />);

    fireEvent.click(await screen.findByRole("button", { name: "Desvincular" }));
    const confirm = await screen.findByRole("region", { name: "Desvincular certificado" });
    expect(within(confirm).getByText(/Os documentos já assinados continuam válidos/)).not.toBeNull();
    fireEvent.click(within(confirm).getByRole("button", { name: "Desvincular" }));

    await waitFor(() => expect(api.unlinkCertificate).toHaveBeenCalledTimes(1));
    expect(await screen.findByText(PAPER)).not.toBeNull();
    expect(screen.getByText("certificado desvinculado")).not.toBeNull();
  });

  it("interruptor desligado entre a sessão e a leitura: diz que a assinatura está desligada", async () => {
    mocked(api.getCurrentCertificate).mockRejectedValue(
      new ApiError(403, { error: "feature_disabled", feature: "digital_signature" }, "403"));
    renderWithProviders(<SignatureAccount redirect={redirect} />);
    expect((await screen.findByRole("alert")).textContent).toBe(SIGNATURE_DISABLED);
  });

  it("sem a funcionalidade na sessão: não chama o api", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(signer(undefined, { features: [ "clinical_record" ] }));
    renderWithProviders(<SignatureAccount redirect={redirect} />);
    await waitFor(() => expect(api.fetchCurrentSession).toHaveBeenCalled());
    expect(await screen.findByText(SIGNATURE_DISABLED)).not.toBeNull();
    expect(api.getCurrentCertificate).not.toHaveBeenCalled();
  });
});
```

E o menu, no fim de `src/shell/modules.test.ts`:

```ts
describe("assinatura digital no menu (módulo 19b)", () => {
  const u = (roles: string[], features: string[] = [ "digital_signature" ], operator = false) =>
    ({ operator, memberships: roles.map((role) => ({ role })), features });
  const ids = (user: ReturnType<typeof u> | null) => navGroupsFor(user).flatMap((g) => g.items.map((i) => i.id));

  it("Conta → Assinatura digital: só profissional com digital_signature, nunca operador", () => {
    const conta = NAV_GROUPS.find((g) => g.label === "Conta");
    expect(conta?.items.map((i) => i.id)).toContain("signature");
    expect(labelFor("signature")).toBe("Assinatura digital");
    expect(ids(u([ "health_professional" ]))).toContain("signature");
    expect(ids(u([ "health_professional" ], [ "clinical_record" ]))).not.toContain("signature");
    expect(ids(u([ "municipal_admin" ]))).not.toContain("signature");
    expect(ids(u([ "health_professional" ], [ "digital_signature" ], true))).not.toContain("signature");
    expect(ids(null)).not.toContain("signature");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod19b && npx vitest run src/modules/SignatureAccount.test.tsx src/shell/modules.test.ts`
Expected: FAIL — `Failed to resolve import "./SignatureAccount"`; no menu, `expected [...] to include 'signature'`.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/SignatureAccount.tsx
// Conta → Assinatura digital (módulo 19b; spec §5 "Vínculo" e §9; contrato
// §3). Procura o certificado em nuvem pelo CPF, vincula (step-up + ida ao
// prestador), troca e desvincula (step-up). Sem certificado, a pessoa segue no
// papel. O OAuth é todo do api: aqui só se segue o authorize_url.
import { useState, type CSSProperties } from "react";
import { useMutation, useQuery, useQueryClient } from "@tanstack/react-query";
import {
  discoverCertificates, getCurrentCertificate, linkCertificate, unlinkCertificate,
  type CertificateDiscovery, type SignatureProvider
} from "../lib/api";
import { useAuth } from "../lib/auth";
import { hasFeature } from "../lib/features";
import {
  CERTIFICATE_HELP, CERTIFICATE_KEY, RETURN_TO, SIGNATURE_DISABLED, canSign, certificateNotice, fmtDay, goToProvider,
  providerLabel, signatureError
} from "../lib/signature";
import { SensitiveAction } from "../components/SensitiveAction";
import { PageHeader } from "../components/PageHeader";
import { Panel } from "../components/Panel";
import { KeyValue } from "../components/KeyValue";
import { EmptyState } from "../components/EmptyState";
import { buttonStyle, disabledButtonStyle, secondaryButtonStyle } from "../components/formStyles";
import { toneColor } from "../theme/tokens";

const PAPER = "Você continua assinando no papel: as consultas saem para impressão e assinatura à mão.";
const NONE_FOUND = "nenhum certificado em nuvem encontrado para o seu CPF nos prestadores habilitados — você continua no papel";

interface Props {
  onGoToSecurity?(): void;
  redirect?(url: string): void;
}

export function SignatureAccount({ onGoToSecurity, redirect = goToProvider }: Props) {
  const { user } = useAuth();
  const queryClient = useQueryClient();
  const allowed = canSign(user);
  const cert = useQuery({ queryKey: [ CERTIFICATE_KEY, user?.id ], queryFn: getCurrentCertificate, enabled: allowed });
  const discover = useMutation({ mutationFn: discoverCertificates });
  const [ searching, setSearching ] = useState(false);
  const [ chosen, setChosen ] = useState<SignatureProvider | null>(null);
  const [ unlinking, setUnlinking ] = useState(false);
  const [ unlinked, setUnlinked ] = useState(false);

  if (!allowed) {
    return (
      <div style={page}>
        <PageHeader title="Assinatura digital" sub="conta · certificado ICP-Brasil em nuvem" />
        <EmptyState title={hasFeature(user, "digital_signature") ? "só profissionais de saúde assinam documentos" : SIGNATURE_DISABLED} />
      </div>
    );
  }

  const current = cert.data ?? null;
  const notice = current ? certificateNotice(current) : null;

  function stopSearching() {
    setSearching(false); setChosen(null); discover.reset();
  }

  return (
    <div style={page}>
      <PageHeader title="Assinatura digital" sub="conta · certificado ICP-Brasil em nuvem" />
      <Panel title="Certificado">
        <div style={body}>
          {unlinked && <p style={text}>certificado desvinculado</p>}
          {cert.isPending && <p style={muted}>carregando…</p>}
          {cert.isError && <p role="alert" style={alert}>{signatureError(cert.error)}</p>}
          {cert.isSuccess && current && (
            <>
              <div style={grid}>
                <KeyValue k="Prestador" v={providerLabel(current.provider)} />
                <KeyValue k="Emissor" v={current.issuer} mono={false} />
                <KeyValue k="Número de série" v={current.serial_number} />
                <KeyValue k="Válido até" v={fmtDay(current.not_after)} />
              </div>
              {notice && <p role="status" style={{ ...text, color: toneColor(notice.tone).fg }}>{notice.text}</p>}
              {!searching && !unlinking && (
                <div style={row}>
                  <button type="button" style={secondaryButtonStyle} onClick={() => setSearching(true)}>Trocar certificado</button>
                  <button type="button" style={secondaryButtonStyle} onClick={() => setUnlinking(true)}>Desvincular</button>
                </div>
              )}
              {unlinking && (
                <SensitiveAction
                  title="Desvincular certificado"
                  description="Os documentos já assinados continuam válidos. As próximas consultas voltam ao papel até você vincular outro certificado."
                  requiresStepUp
                  confirmLabel="Desvincular"
                  run={async () => { await unlinkCertificate(); }}
                  onDone={() => {
                    setUnlinking(false); setUnlinked(true);
                    void queryClient.invalidateQueries({ queryKey: [ CERTIFICATE_KEY ] });
                  }}
                  onCancel={() => setUnlinking(false)}
                  onGoToSecurity={onGoToSecurity}
                  translateError={signatureError}
                />
              )}
            </>
          )}
          {cert.isSuccess && !current && (
            <>
              <p style={text}>{PAPER}</p>
              <p style={muted}>{CERTIFICATE_HELP}</p>
              {!searching && (
                <div><button type="button" style={buttonStyle} onClick={() => setSearching(true)}>Procurar meu certificado</button></div>
              )}
            </>
          )}
        </div>
      </Panel>

      {searching && (
        <Panel title="Procurar certificado pelo CPF">
          <div style={body}>
            <p style={muted}>O Rota Saúde procura, pelo seu CPF, um certificado em nuvem em cada prestador habilitado.</p>
            {!discover.data && (
              <div style={row}>
                <button type="button" disabled={discover.isPending} style={discover.isPending ? disabledButtonStyle : buttonStyle}
                  onClick={() => discover.mutate()}>Procurar</button>
                <button type="button" style={secondaryButtonStyle} onClick={stopSearching}>Cancelar</button>
              </div>
            )}
            {discover.isError && <p role="alert" style={alert}>{signatureError(discover.error)}</p>}
            {discover.data && !chosen && <DiscoveryResult data={discover.data} onChoose={setChosen} onCancel={stopSearching} />}
            {chosen && (
              <SensitiveAction
                title={`Vincular certificado ${providerLabel(chosen)}`}
                description="Você vai ao prestador autorizar o uso do certificado. Ele precisa estar emitido para o seu CPF."
                requiresStepUp
                confirmLabel="Ir ao prestador"
                run={async () => {
                  const { authorize_url } = await linkCertificate(chosen, RETURN_TO.account);
                  redirect(authorize_url);
                }}
                onDone={() => undefined}
                onCancel={() => setChosen(null)}
                onGoToSecurity={onGoToSecurity}
                translateError={signatureError}
              />
            )}
          </div>
        </Panel>
      )}
    </div>
  );
}

function DiscoveryResult({ data, onChoose, onCancel }: {
  data: CertificateDiscovery; onChoose(p: SignatureProvider): void; onCancel(): void;
}) {
  const found = data.providers.filter((p) => p.found).map((p) => p.provider);
  const missing = data.providers.filter((p) => !p.found).map((p) => providerLabel(p.provider));
  return (
    <div style={body}>
      {found.length === 0 ? <p style={text}>{NONE_FOUND}</p> : (
        <ul aria-label="certificados encontrados" style={list}>
          {found.map((p) => (
            <li key={p}>
              <button type="button" style={buttonStyle} onClick={() => onChoose(p)}>{`Vincular ${providerLabel(p)}`}</button>
            </li>
          ))}
        </ul>
      )}
      {missing.length > 0 && <p style={muted}>{`não encontrado em: ${missing.join(", ")}`}</p>}
      {data.unavailable.length > 0 && (
        <p style={muted}>{`sem resposta: ${data.unavailable.map(providerLabel).join(", ")} — procure de novo em alguns minutos`}</p>
      )}
      <div><button type="button" style={secondaryButtonStyle} onClick={onCancel}>Fechar</button></div>
    </div>
  );
}

const page: CSSProperties = { display: "flex", flexDirection: "column", gap: 16 };
const body: CSSProperties = { display: "flex", flexDirection: "column", gap: 12, fontSize: 12.5 };
const grid: CSSProperties = { display: "grid", gridTemplateColumns: "repeat(auto-fit, minmax(160px, 1fr))", gap: 12 };
const row: CSSProperties = { display: "flex", gap: 8, flexWrap: "wrap" };
const list: CSSProperties = { margin: 0, padding: 0, listStyle: "none", display: "flex", gap: 8, flexWrap: "wrap" };
const text: CSSProperties = { margin: 0, fontSize: 12.5 };
const muted: CSSProperties = { margin: 0, fontSize: 12, color: "var(--ink3)" };
const alert: CSSProperties = { margin: 0, fontSize: 12.5, color: "var(--down)" };
```

Em `src/shell/modules.ts` (sobre o estado do 19a):

```diff
--- a/src/shell/modules.ts
+++ b/src/shell/modules.ts
@@
 import { hasFeature } from "../lib/features";
+import { canSign } from "../lib/signature";
 export type ModuleId =
@@
-  | "integrations" | "cnes" | "production" | "my-agenda" | "clinical-record";
+  | "integrations" | "cnes" | "production" | "my-agenda" | "clinical-record"
+  | "signature";
@@
   { label: "Conta", items: [
     { id: "security", label: "Segurança", icon: "⚿" },
-    { id: "my-profile", label: "Meu perfil", icon: "☺" }
+    { id: "my-profile", label: "Meu perfil", icon: "☺" },
+    { id: "signature", label: "Assinatura digital", icon: "✍" }
   ]}
@@ export function navGroupsFor(
       if (item.id === "my-agenda") return isProfessional;
+      // Módulo 19b: certificado e sessão de assinatura são do profissional,
+      // só com `digital_signature` ligado (o api recusaria com 403).
+      if (item.id === "signature") return canSign(user);
       if (item.id === "integrations" || item.id === "cnes") return canIntegrations;
```

Em `src/App.tsx`:

```diff
--- a/src/App.tsx
+++ b/src/App.tsx
@@
 import { Production } from "./modules/Production";
+import { SignatureAccount } from "./modules/SignatureAccount";
@@ function renderModule(active: ModuleId, setActive: (id: ModuleId) => void) {
     case "production":     return <Production onGoToSecurity={() => setActive("security")} />;
+    case "signature":      return <SignatureAccount onGoToSecurity={() => setActive("security")} />;
     default:               return <Placeholder title={labelFor(active)} />;
```

(O `case "clinical-record"` do 19a fica onde estiver; a linha nova entra antes do `default`.)

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod19b && npx vitest run src/modules/SignatureAccount.test.tsx src/shell/modules.test.ts && npx tsc --noEmit`
Expected: PASS (8 testes na página; `modules.test.ts` verde, inclusive os testes antigos que contam grupos — nenhum grupo novo); `tsc` limpo.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19b add src/modules/SignatureAccount.tsx src/modules/SignatureAccount.test.tsx src/shell/modules.ts src/shell/modules.test.ts src/App.tsx
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19b commit -m "feat: add the digital signature certificate page to the account menu

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 4: Pendentes de assinatura — lista, "Assinar todas" e "Voltar ao papel"

**Files:**
- Create: `src/hooks/usePendingSignatures.ts`, `src/modules/SignaturePending.tsx`
- Modify: `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/App.tsx`
- Test: `src/modules/SignaturePending.test.tsx`

**Interfaces:**
- Consumes: `listPendingSignatures`, `startSignatureBatch`, `returnToPaper`, `errorCode`, tipos (Task 1); `PENDING_KEY`, `RETURN_TO`, `BATCH_LIMIT`, `RETURNED_TO_PAPER`, `SIGNATURE_DISABLED`, `canSign`, `documentLabel`, `goToProvider`, `reasonLabel`, `returnToPaperProblem`, `signatureError` (Task 2); `CONSULTATION_KEY` (`src/lib/consultation.ts`, 19a); `useAuth`, `hasFeature`, `fmtDateTime`, `PageHeader`, `Panel`, `DataTable`, `EmptyState`, `formStyles`; `ModuleId` (`src/shell/modules.ts`).
- Produces:
  - `PENDING_REFRESH_MS = 60_000`; `usePendingSignatures(enabled: boolean)` — `useQuery` com chave `[ PENDING_KEY ]`, `listPendingSignatures`, relida a cada minuto quando `enabled` (a Task 5 usa a mesma chave para o contador);
  - `SignaturePending({ onNavigate(id: ModuleId): void; redirect?(url: string): void })` — página "Pendentes de assinatura": `Panel` "Pendentes" com "Assinar todas" (lote sem ids, `return_to: "/signature-pending"`), aviso de lote de 50, tabela (paciente, documento, finalizado em, motivo, tentativas, "Voltar ao papel"), formulário `section` "Voltar ao papel" com "Motivo para voltar ao papel" e "Confirmar volta ao papel";
  - `ModuleId` com `"signature-pending"`; item "Pendentes de assinatura" no grupo Atendimento, só com `canSign(user)`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/SignaturePending.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), listPendingSignatures: vi.fn(), startSignatureBatch: vi.fn(), returnToPaper: vi.fn() };
});

import * as api from "../lib/api";
import { ApiError } from "../lib/api";
import { SignaturePending } from "./SignaturePending";
import { renderWithProviders } from "../test/campaignFixtures";
import { pendingRequest, signer } from "../test/signatureFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

afterEach(cleanup);

describe("SignaturePending", () => {
  let redirect: ReturnType<typeof vi.fn>;
  let onNavigate: ReturnType<typeof vi.fn>;

  beforeEach(() => {
    vi.clearAllMocks();
    redirect = vi.fn();
    onNavigate = vi.fn();
    mocked(api.fetchCurrentSession).mockResolvedValue(signer());
    mocked(api.listPendingSignatures).mockResolvedValue([
      pendingRequest(),
      pendingRequest({ id: "sr2", document_type: "consultation_addendum", document_id: "ad1", patient_display_name: "Pedro Alves",
        finalized_at: "2026-10-07T15:00:00-03:00", reason_code: "provider_unavailable", attempts: 3 })
    ]);
  });

  const renderIt = () => renderWithProviders(<SignaturePending onNavigate={onNavigate} redirect={redirect} />);

  it("lista paciente, documento, finalização, motivo e tentativas", async () => {
    renderIt();
    const panel = await screen.findByRole("region", { name: "Pendentes" });
    expect(await within(panel).findByText("Joana Lima")).not.toBeNull();
    expect(within(panel).getByText("consulta")).not.toBeNull();
    expect(within(panel).getByText("07/10/2026, 10:20")).not.toBeNull();
    expect(within(panel).getByText("sem sessão de assinatura aberta")).not.toBeNull();
    expect(within(panel).getByText("Pedro Alves")).not.toBeNull();
    expect(within(panel).getByText("adendo")).not.toBeNull();
    expect(within(panel).getByText("o prestador não respondeu")).not.toBeNull();
    expect(within(panel).getByText("3")).not.toBeNull();
  });

  it("Assinar todas: pede o lote sem ids e vai ao prestador", async () => {
    mocked(api.startSignatureBatch).mockResolvedValue({ authorize_url: "https://psc.example/authorize?b=1", count: 2 });
    renderIt();
    await screen.findByText("Joana Lima");
    fireEvent.click(screen.getByRole("button", { name: "Assinar todas" }));
    await waitFor(() => expect(redirect).toHaveBeenCalledWith("https://psc.example/authorize?b=1"));
    expect(api.startSignatureBatch).toHaveBeenCalledWith("/signature-pending");
  });

  it("mais de 50: avisa que o lote leva as 50 mais antigas", async () => {
    mocked(api.listPendingSignatures).mockResolvedValue(
      Array.from({ length: 51 }, (_, i) => pendingRequest({ id: `sr${i}`, document_id: `cs${i}` })));
    renderIt();
    expect(await screen.findByText(
      "cada lote assina até 50 documentos, os mais antigos primeiro — depois do retorno, clique de novo para os demais")).not.toBeNull();
  });

  it("nothing_pending relê a lista e diz que não há pendentes", async () => {
    mocked(api.startSignatureBatch).mockRejectedValue(new ApiError(409, { error: "nothing_pending" }, "409"));
    renderIt();
    await screen.findByText("Joana Lima");
    mocked(api.listPendingSignatures).mockResolvedValue([]);
    fireEvent.click(screen.getByRole("button", { name: "Assinar todas" }));
    expect(await screen.findByText("não há documentos pendentes — a lista foi atualizada")).not.toBeNull();
    expect(await screen.findByText("nenhum documento esperando a sua assinatura")).not.toBeNull();
    expect(redirect).not.toHaveBeenCalled();
  });

  it("sem certificado: diz e leva à Conta → Assinatura digital", async () => {
    mocked(api.startSignatureBatch).mockRejectedValue(new ApiError(409, { error: "certificate_not_linked" }, "409"));
    renderIt();
    await screen.findByText("Joana Lima");
    fireEvent.click(screen.getByRole("button", { name: "Assinar todas" }));
    expect((await screen.findByRole("alert")).textContent).toBe("vincule um certificado em Conta → Assinatura digital antes de assinar");
    fireEvent.click(screen.getByRole("button", { name: "Vincular certificado" }));
    expect(onNavigate).toHaveBeenCalledWith("signature");
  });

  it("Voltar ao papel com motivo curto: avisa e não envia", async () => {
    renderIt();
    await screen.findByText("Joana Lima");
    fireEvent.click(screen.getAllByRole("button", { name: "Voltar ao papel" })[0]);
    const form = screen.getByRole("region", { name: "Voltar ao papel" });
    fireEvent.change(within(form).getByLabelText("Motivo para voltar ao papel"), { target: { value: "  curto  " } });
    fireEvent.click(within(form).getByRole("button", { name: "Confirmar volta ao papel" }));
    expect(within(form).getByText("descreva o motivo com pelo menos 10 caracteres")).not.toBeNull();
    expect(api.returnToPaper).not.toHaveBeenCalled();
  });

  it("Voltar ao papel: manda o motivo sem espaços das pontas, diz o que fazer e relê", async () => {
    mocked(api.returnToPaper).mockResolvedValue(pendingRequest({ status: "returned_to_paper", reason_code: "user_request" }));
    renderIt();
    await screen.findByText("Joana Lima");
    fireEvent.click(screen.getAllByRole("button", { name: "Voltar ao papel" })[0]);
    const form = screen.getByRole("region", { name: "Voltar ao papel" });
    fireEvent.change(within(form).getByLabelText("Motivo para voltar ao papel"), { target: { value: "  paciente pediu o papel  " } });
    mocked(api.listPendingSignatures).mockResolvedValue([]);
    fireEvent.click(within(form).getByRole("button", { name: "Confirmar volta ao papel" }));

    await waitFor(() => expect(api.returnToPaper).toHaveBeenCalledWith("sr1", "paciente pediu o papel"));
    expect(await screen.findByText("voltou ao papel — imprima a consulta e assine à mão")).not.toBeNull();
    expect(screen.queryByRole("region", { name: "Voltar ao papel" })).toBeNull();
    expect(await screen.findByText("nenhum documento esperando a sua assinatura")).not.toBeNull();
  });

  it("not_pending fecha o formulário, diz e relê a lista", async () => {
    mocked(api.returnToPaper).mockRejectedValue(new ApiError(409, { error: "not_pending" }, "409"));
    renderIt();
    await screen.findByText("Joana Lima");
    fireEvent.click(screen.getAllByRole("button", { name: "Voltar ao papel" })[0]);
    const form = screen.getByRole("region", { name: "Voltar ao papel" });
    fireEvent.change(within(form).getByLabelText("Motivo para voltar ao papel"), { target: { value: "paciente pediu o papel" } });
    const before = mocked(api.listPendingSignatures).mock.calls.length;
    fireEvent.click(within(form).getByRole("button", { name: "Confirmar volta ao papel" }));

    expect(await screen.findByText("este documento não está mais pendente — a lista foi atualizada")).not.toBeNull();
    expect(screen.queryByRole("region", { name: "Voltar ao papel" })).toBeNull();
    await waitFor(() => expect(mocked(api.listPendingSignatures).mock.calls.length).toBeGreaterThan(before));
  });

  it("lista vazia: diz e não oferece lote", async () => {
    mocked(api.listPendingSignatures).mockResolvedValue([]);
    renderIt();
    expect(await screen.findByText("nenhum documento esperando a sua assinatura")).not.toBeNull();
    expect((screen.getByRole("button", { name: "Assinar todas" }) as HTMLButtonElement).disabled).toBe(true);
  });
});
```

E o menu, dentro do `describe("assinatura digital no menu (módulo 19b)")` de `src/shell/modules.test.ts` (Task 3):

```ts
  it("Atendimento → Pendentes de assinatura: só profissional com digital_signature", () => {
    const atendimento = NAV_GROUPS.find((g) => g.label === "Atendimento");
    expect(atendimento?.items.map((i) => i.id)).toContain("signature-pending");
    expect(labelFor("signature-pending")).toBe("Pendentes de assinatura");
    expect(ids(u([ "health_professional" ]))).toContain("signature-pending");
    expect(ids(u([ "health_professional" ], []))).not.toContain("signature-pending");
    expect(ids(u([ "citizen_verifier" ]))).not.toContain("signature-pending");
    expect(ids(u([ "municipal_admin" ]))).not.toContain("signature-pending");
  });
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod19b && npx vitest run src/modules/SignaturePending.test.tsx src/shell/modules.test.ts`
Expected: FAIL — `Failed to resolve import "./SignaturePending"`; no menu, `expected [...] to include 'signature-pending'`.

- [ ] **Step 3: Write minimal implementation**

```ts
// src/hooks/usePendingSignatures.ts
// Pendentes de assinatura do próprio profissional (módulo 19b; contrato §5).
// A página e o contador do menu leem a mesma chave: uma requisição por minuto.
import { useQuery } from "@tanstack/react-query";
import { listPendingSignatures } from "../lib/api";
import { PENDING_KEY } from "../lib/signature";

export const PENDING_REFRESH_MS = 60_000;

export function usePendingSignatures(enabled: boolean) {
  return useQuery({
    queryKey: [ PENDING_KEY ],
    queryFn: listPendingSignatures,
    enabled,
    refetchInterval: enabled ? PENDING_REFRESH_MS : false
  });
}
```

```tsx
// src/modules/SignaturePending.tsx
// Pendentes de assinatura (módulo 19b; spec §5 "Lote" e §9; contrato §5): o
// que espera a assinatura digital do próprio profissional, "Assinar todas"
// (lote de até 50, pelo prestador) e "Voltar ao papel" (motivo ≥ 10). O api
// confere autor e estado; a tela relê a lista quando o mundo mudou.
import { useState, type CSSProperties, type FormEvent } from "react";
import { useMutation, useQueryClient } from "@tanstack/react-query";
import { errorCode, returnToPaper, startSignatureBatch, type SignatureRequest } from "../lib/api";
import { useAuth } from "../lib/auth";
import { hasFeature } from "../lib/features";
import { fmtDateTime } from "../lib/format";
import { CONSULTATION_KEY } from "../lib/consultation";
import {
  BATCH_LIMIT, PENDING_KEY, RETURNED_TO_PAPER, RETURN_TO, SIGNATURE_DISABLED, canSign, documentLabel, goToProvider,
  reasonLabel, returnToPaperProblem, signatureError
} from "../lib/signature";
import { usePendingSignatures } from "../hooks/usePendingSignatures";
import type { ModuleId } from "../shell/modules";
import { PageHeader } from "../components/PageHeader";
import { Panel } from "../components/Panel";
import { DataTable, type Column } from "../components/DataTable";
import { EmptyState } from "../components/EmptyState";
import { buttonStyle, disabledButtonStyle, inputStyle, secondaryButtonStyle } from "../components/formStyles";

const EMPTY = "nenhum documento esperando a sua assinatura";
const BATCH_NOTE = `cada lote assina até ${BATCH_LIMIT} documentos, os mais antigos primeiro — depois do retorno, clique de novo para os demais`;

interface Props {
  onNavigate(id: ModuleId): void;
  redirect?(url: string): void;
}

export function SignaturePending({ onNavigate, redirect = goToProvider }: Props) {
  const { user } = useAuth();
  const queryClient = useQueryClient();
  const allowed = canSign(user);
  const pending = usePendingSignatures(allowed);
  const [ notice, setNotice ] = useState<string | null>(null);
  const [ failure, setFailure ] = useState<{ text: string; linkCertificate: boolean } | null>(null);
  const [ paperFor, setPaperFor ] = useState<SignatureRequest | null>(null);

  const batch = useMutation({
    mutationFn: () => startSignatureBatch(RETURN_TO.pending),
    onMutate: () => { setNotice(null); setFailure(null); },
    onSuccess: ({ authorize_url }) => redirect(authorize_url),
    onError: (err) => {
      const code = errorCode(err);
      if (code === "nothing_pending") {
        setNotice(signatureError(err));
        void queryClient.invalidateQueries({ queryKey: [ PENDING_KEY ] });
        return;
      }
      setFailure({ text: signatureError(err), linkCertificate: code === "certificate_not_linked" });
    }
  });

  if (!allowed) {
    return (
      <div style={page}>
        <PageHeader title="Pendentes de assinatura" sub="atendimento · assinatura digital" />
        <EmptyState title={hasFeature(user, "digital_signature") ? "só profissionais de saúde assinam documentos" : SIGNATURE_DISABLED} />
      </div>
    );
  }

  const items = pending.data ?? [];

  function refresh() {
    void queryClient.invalidateQueries({ queryKey: [ PENDING_KEY ] });
  }

  const cols: Column<SignatureRequest>[] = [
    { label: "Paciente", w: "2fr", render: (r) => r.patient_display_name ?? "—" },
    { label: "Documento", w: "1fr", render: (r) => documentLabel(r.document_type) },
    { label: "Finalizado em", w: "1.4fr", render: (r) => fmtDateTime(r.finalized_at) },
    { label: "Motivo", w: "2fr", render: (r) => reasonLabel(r.reason_code) },
    { label: "Tentativas", w: "0.8fr", align: "right", render: (r) => String(r.attempts) },
    {
      label: "", w: "1.2fr", render: (r) => (
        <button type="button" style={secondaryButtonStyle} onClick={() => { setNotice(null); setPaperFor(r); }}>Voltar ao papel</button>
      )
    }
  ];

  return (
    <div style={page}>
      <PageHeader title="Pendentes de assinatura" sub="atendimento · assinatura digital" />
      <Panel
        title="Pendentes"
        sub="consultas e adendos que esperam a sua assinatura digital"
        right={(
          <button type="button" disabled={items.length === 0 || batch.isPending}
            style={items.length === 0 || batch.isPending ? disabledButtonStyle : buttonStyle}
            onClick={() => batch.mutate()}>Assinar todas</button>
        )}
      >
        <div style={body}>
          {notice && <p role="status" style={text}>{notice}</p>}
          {failure && (
            <div style={row}>
              <p role="alert" style={alert}>{failure.text}</p>
              {failure.linkCertificate && (
                <button type="button" style={secondaryButtonStyle} onClick={() => onNavigate("signature")}>Vincular certificado</button>
              )}
            </div>
          )}
          {items.length > BATCH_LIMIT && <p style={muted}>{BATCH_NOTE}</p>}
          {pending.isPending && <p style={muted}>carregando…</p>}
          {pending.isError && <p role="alert" style={alert}>{signatureError(pending.error)}</p>}
          {pending.isSuccess && <DataTable cols={cols} rows={items} rowKey={(r) => r.id} empty={EMPTY} />}
          {paperFor && (
            <ReturnToPaperForm
              request={paperFor}
              onDone={() => {
                setPaperFor(null); setNotice(RETURNED_TO_PAPER); refresh();
                void queryClient.invalidateQueries({ queryKey: [ CONSULTATION_KEY ] });
              }}
              onStale={(message) => { setPaperFor(null); setNotice(message); refresh(); }}
              onCancel={() => setPaperFor(null)}
            />
          )}
        </div>
      </Panel>
    </div>
  );
}

function ReturnToPaperForm({ request, onDone, onStale, onCancel }: {
  request: SignatureRequest; onDone(): void; onStale(message: string): void; onCancel(): void;
}) {
  const [ reason, setReason ] = useState("");
  const [ problem, setProblem ] = useState<string | null>(null);
  const mutation = useMutation({ mutationFn: (text: string) => returnToPaper(request.id, text) });

  async function submit(event: FormEvent) {
    event.preventDefault();
    if (mutation.isPending) return;
    const found = returnToPaperProblem(reason);
    if (found) { setProblem(found); return; }
    setProblem(null);
    try {
      await mutation.mutateAsync(reason.trim());
      onDone();
    } catch (err) {
      const code = errorCode(err);
      if (code === "not_pending" || code === "not_author") { onStale(signatureError(err)); return; }
      setProblem(signatureError(err));
    }
  }

  return (
    <section aria-label="Voltar ao papel" style={panel}>
      <strong>{`Voltar ao papel: ${documentLabel(request.document_type)} de ${fmtDateTime(request.finalized_at)}`}</strong>
      <p style={muted}>O documento deixa de esperar a assinatura digital e o impresso volta a ter espaço para assinatura à mão. O motivo fica registrado.</p>
      <form onSubmit={(e) => void submit(e)} style={body}>
        <label style={label}>
          Motivo para voltar ao papel
          <textarea value={reason} onChange={(e) => setReason(e.target.value)} rows={3} style={inputStyle} />
        </label>
        {problem && <p role="alert" style={alert}>{problem}</p>}
        <div style={row}>
          <button type="submit" disabled={mutation.isPending} style={mutation.isPending ? disabledButtonStyle : buttonStyle}>
            Confirmar volta ao papel
          </button>
          <button type="button" disabled={mutation.isPending} style={secondaryButtonStyle} onClick={onCancel}>Cancelar</button>
        </div>
      </form>
    </section>
  );
}

const page: CSSProperties = { display: "flex", flexDirection: "column", gap: 16 };
const body: CSSProperties = { display: "flex", flexDirection: "column", gap: 10, fontSize: 12.5 };
const row: CSSProperties = { display: "flex", gap: 8, alignItems: "center", flexWrap: "wrap" };
const panel: CSSProperties = { display: "flex", flexDirection: "column", gap: 10, padding: 16, border: "1px solid var(--rule)", borderRadius: 8 };
const label: CSSProperties = { display: "flex", flexDirection: "column", gap: 4, fontSize: 12, color: "var(--ink2)" };
const text: CSSProperties = { margin: 0, fontSize: 12.5 };
const muted: CSSProperties = { margin: 0, fontSize: 12, color: "var(--ink3)" };
const alert: CSSProperties = { margin: 0, fontSize: 12.5, color: "var(--down)" };
```

Em `src/shell/modules.ts` (sobre a Task 3):

```diff
--- a/src/shell/modules.ts
+++ b/src/shell/modules.ts
@@
   | "integrations" | "cnes" | "production" | "my-agenda" | "clinical-record"
-  | "signature";
+  | "signature" | "signature-pending";
@@
   { label: "Atendimento", items: [
     { id: "attendance", label: "Atendimento", icon: "☑" },
-    { id: "my-agenda", label: "Minha agenda", icon: "◷" }
+    { id: "my-agenda", label: "Minha agenda", icon: "◷" },
+    { id: "signature-pending", label: "Pendentes de assinatura", icon: "⧗" }
   ]},
@@ export function navGroupsFor(
       if (item.id === "signature") return canSign(user);
+      if (item.id === "signature-pending") return canSign(user);
```

(O item `clinical-record` do 19a continua no grupo Atendimento; o novo entra por último.)

Em `src/App.tsx`:

```diff
--- a/src/App.tsx
+++ b/src/App.tsx
@@
 import { SignatureAccount } from "./modules/SignatureAccount";
+import { SignaturePending } from "./modules/SignaturePending";
@@ function renderModule(active: ModuleId, setActive: (id: ModuleId) => void) {
     case "signature":      return <SignatureAccount onGoToSecurity={() => setActive("security")} />;
+    case "signature-pending": return <SignaturePending onNavigate={setActive} />;
     default:               return <Placeholder title={labelFor(active)} />;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod19b && npx vitest run src/modules/SignaturePending.test.tsx src/shell/modules.test.ts && npx tsc --noEmit`
Expected: PASS (9 testes na página); `tsc` limpo.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19b add src/hooks/usePendingSignatures.ts src/modules/SignaturePending.tsx src/modules/SignaturePending.test.tsx src/shell/modules.ts src/shell/modules.test.ts src/App.tsx
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19b commit -m "feat: add the pending signatures page with batch signing and return to paper

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 5: Selo da sessão de assinatura no topo e contador no menu

**Files:**
- Create: `src/shell/SignatureSessionBadge.tsx`
- Modify: `src/shell/AppHeader.tsx`, `src/shell/modules.ts`, `src/shell/modules.test.ts`
- Test: `src/shell/SignatureSessionBadge.test.tsx`

**Interfaces:**
- Consumes: `getSignatureSession`, `openSignatureSession`, `closeSignatureSession`, `errorCode` (Task 1); `SESSION_KEY`, `canSign`, `goToProvider`, `sessionBadge`, `signatureError` (Task 2); `usePendingSignatures` (Task 4); `useAuth`, `Tag`; `ModuleId`, `NavGroupDef`, `navGroupsFor` (`src/shell/modules.ts`).
- Produces:
  - `SignatureSessionBadge({ active: ModuleId; onSelect(id: ModuleId): void; redirect?(url: string): void })` — `<span aria-label="sessão de assinatura">` com o `Tag` "assinatura ativa até HH:MM" (e "encerrar") ou "sem sessão de assinatura" (e "abrir sessão", que pede o `authorize_url` com `return_to: "/<módulo ativo>"`); sem certificado (409 `certificate_not_linked`) leva a Conta → Assinatura digital; relê a sessão a cada minuto e recalcula o selo a cada 30 s; nada para quem não pode assinar;
  - `withPendingCount(groups: NavGroupDef[], count: number): NavGroupDef[]` em `src/shell/modules.ts` — "Pendentes de assinatura (3)" (de 200 em diante, "(200+)");
  - `AppHeader` mostra o selo antes do chip da cidade e o contador no item do menu.

- [ ] **Step 1: Write the failing test**

```tsx
// src/shell/SignatureSessionBadge.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { act, cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getSignatureSession: vi.fn(), openSignatureSession: vi.fn(), closeSignatureSession: vi.fn() };
});

import * as api from "../lib/api";
import { ApiError } from "../lib/api";
import { SignatureSessionBadge } from "./SignatureSessionBadge";
import { renderWithProviders } from "../test/campaignFixtures";
import { NOW19B, signer } from "../test/signatureFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;

afterEach(() => { cleanup(); vi.useRealTimers(); });

describe("SignatureSessionBadge", () => {
  let redirect: ReturnType<typeof vi.fn>;
  let onSelect: ReturnType<typeof vi.fn>;

  beforeEach(() => {
    vi.clearAllMocks();
    vi.useFakeTimers({ toFake: [ "Date", "setInterval", "clearInterval" ] });
    vi.setSystemTime(new Date(NOW19B));
    redirect = vi.fn();
    onSelect = vi.fn();
    mocked(api.fetchCurrentSession).mockResolvedValue(signer());
  });

  const renderIt = () =>
    renderWithProviders(<SignatureSessionBadge active="attendance" onSelect={onSelect} redirect={redirect} />);

  it("sessão ativa: diz até quando e encerra", async () => {
    mocked(api.getSignatureSession).mockResolvedValueOnce({ active: true, expires_at: "2026-10-08T22:00:00-03:00", provider: "vidaas" })
      .mockResolvedValue({ active: false });
    mocked(api.closeSignatureSession).mockResolvedValue(undefined);
    renderIt();
    const badge = await screen.findByLabelText("sessão de assinatura");
    expect(await within(badge).findByText("assinatura ativa até 22:00")).not.toBeNull();
    fireEvent.click(within(badge).getByRole("button", { name: "encerrar" }));
    await waitFor(() => expect(api.closeSignatureSession).toHaveBeenCalledTimes(1));
    expect(await within(badge).findByText("sem sessão de assinatura")).not.toBeNull();
  });

  it("a sessão vence com a tela aberta: o selo muda sozinho", async () => {
    mocked(api.getSignatureSession).mockResolvedValue({ active: true, expires_at: "2026-10-08T10:30:00-03:00", provider: "vidaas" });
    renderIt();
    expect(await screen.findByText("assinatura ativa até 10:30")).not.toBeNull();
    act(() => { vi.advanceTimersByTime(31 * 60_000); });
    expect(await screen.findByText("sem sessão de assinatura")).not.toBeNull();
    expect(screen.getByRole("button", { name: "abrir sessão" })).not.toBeNull();
  });

  it("sem sessão: abre pelo prestador e volta para a tela atual", async () => {
    mocked(api.getSignatureSession).mockResolvedValue({ active: false });
    mocked(api.openSignatureSession).mockResolvedValue({ authorize_url: "https://psc.example/authorize?s=1" });
    renderIt();
    fireEvent.click(await screen.findByRole("button", { name: "abrir sessão" }));
    await waitFor(() => expect(redirect).toHaveBeenCalledWith("https://psc.example/authorize?s=1"));
    expect(api.openSignatureSession).toHaveBeenCalledWith("/attendance");
  });

  it("sem certificado vinculado: leva a Conta → Assinatura digital", async () => {
    mocked(api.getSignatureSession).mockResolvedValue({ active: false });
    mocked(api.openSignatureSession).mockRejectedValue(new ApiError(409, { error: "certificate_not_linked" }, "409"));
    renderIt();
    fireEvent.click(await screen.findByRole("button", { name: "abrir sessão" }));
    await waitFor(() => expect(onSelect).toHaveBeenCalledWith("signature"));
    expect(redirect).not.toHaveBeenCalled();
  });

  it("prestador fora do ar: diz, sem sair da tela", async () => {
    mocked(api.getSignatureSession).mockResolvedValue({ active: false });
    mocked(api.openSignatureSession).mockRejectedValue(new ApiError(503, { error: "provider_unavailable" }, "503"));
    renderIt();
    fireEvent.click(await screen.findByRole("button", { name: "abrir sessão" }));
    expect((await screen.findByRole("alert")).textContent).toBe("o prestador não respondeu — tente de novo em alguns minutos");
  });

  it("quem não pode assinar não vê o selo nem chama o api", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(signer([ "citizen_verifier" ]));
    renderIt();
    await waitFor(() => expect(api.fetchCurrentSession).toHaveBeenCalled());
    expect(screen.queryByLabelText("sessão de assinatura")).toBeNull();
    expect(api.getSignatureSession).not.toHaveBeenCalled();
  });
});
```

E o contador, no `describe("assinatura digital no menu (módulo 19b)")` de `src/shell/modules.test.ts` (acrescente `withPendingCount` ao import do arquivo):

```ts
  it("contador de pendentes no item do menu", () => {
    const groups = navGroupsFor(u([ "health_professional" ]));
    const label = (gs: typeof groups) => gs.flatMap((g) => g.items).find((i) => i.id === "signature-pending")?.label;
    expect(label(withPendingCount(groups, 0))).toBe("Pendentes de assinatura");
    expect(label(withPendingCount(groups, 3))).toBe("Pendentes de assinatura (3)");
    expect(label(withPendingCount(groups, 200))).toBe("Pendentes de assinatura (200+)");
    expect(groups.flatMap((g) => g.items).find((i) => i.id === "signature-pending")?.label).toBe("Pendentes de assinatura");
  });
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod19b && npx vitest run src/shell/SignatureSessionBadge.test.tsx src/shell/modules.test.ts`
Expected: FAIL — `Failed to resolve import "./SignatureSessionBadge"` e `withPendingCount is not a function`.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/shell/SignatureSessionBadge.tsx
// Selo da sessão de assinatura no topo (módulo 19b; spec §5 "Sessão do
// turno" e §9; contrato §4). Mostra até quando a sessão vale e muda sozinho
// quando ela vence (relógio de 30 s, releitura de 1 min). Abrir vai ao
// prestador e volta para a tela de onde saiu (return_to "/<módulo>").
import { useEffect, useState, type CSSProperties } from "react";
import { useMutation, useQuery, useQueryClient } from "@tanstack/react-query";
import { closeSignatureSession, errorCode, getSignatureSession, openSignatureSession } from "../lib/api";
import { useAuth } from "../lib/auth";
import { SESSION_KEY, canSign, goToProvider, sessionBadge, signatureError } from "../lib/signature";
import { Tag } from "../components/Tag";
import type { ModuleId } from "./modules";

const TICK_MS = 30_000;
const REFRESH_MS = 60_000;

interface Props {
  active: ModuleId;
  onSelect(id: ModuleId): void;
  redirect?(url: string): void;
}

export function SignatureSessionBadge({ active, onSelect, redirect = goToProvider }: Props) {
  const { user } = useAuth();
  const allowed = canSign(user);
  const queryClient = useQueryClient();
  const session = useQuery({
    queryKey: [ SESSION_KEY, user?.id ], queryFn: getSignatureSession, enabled: allowed,
    refetchInterval: allowed ? REFRESH_MS : false
  });
  const [ now, setNow ] = useState(() => Date.now());
  const [ failure, setFailure ] = useState<string | null>(null);

  useEffect(() => {
    if (!allowed) return;
    const id = setInterval(() => setNow(Date.now()), TICK_MS);
    return () => clearInterval(id);
  }, [ allowed ]);

  const open = useMutation({
    mutationFn: () => openSignatureSession(`/${active}`),
    onMutate: () => setFailure(null),
    onSuccess: ({ authorize_url }) => redirect(authorize_url),
    onError: (err) => {
      if (errorCode(err) === "certificate_not_linked") { onSelect("signature"); return; }
      setFailure(signatureError(err));
    }
  });

  const close = useMutation({
    mutationFn: closeSignatureSession,
    onMutate: () => setFailure(null),
    onSuccess: () => void queryClient.invalidateQueries({ queryKey: [ SESSION_KEY ] }),
    onError: (err) => setFailure(signatureError(err))
  });

  if (!allowed || !session.isSuccess) return null;
  const badge = sessionBadge(session.data, now);

  return (
    <span aria-label="sessão de assinatura" style={wrap}>
      <Tag tone={badge.active ? "ok" : "neutral"}>{badge.label}</Tag>
      {badge.active ? (
        <button type="button" disabled={close.isPending} style={linkButton} onClick={() => close.mutate()}>encerrar</button>
      ) : (
        <button type="button" disabled={open.isPending} style={linkButton} onClick={() => open.mutate()}>abrir sessão</button>
      )}
      {failure && <span role="alert" style={alert}>{failure}</span>}
    </span>
  );
}

const wrap: CSSProperties = { display: "inline-flex", alignItems: "center", gap: 6, flexShrink: 0 };
const linkButton: CSSProperties = {
  border: "none", background: "transparent", padding: 0, cursor: "pointer", fontSize: 11.5,
  color: "var(--accent)", textDecoration: "underline", whiteSpace: "nowrap"
};
const alert: CSSProperties = { fontSize: 11, color: "var(--down)", maxWidth: 240 };
```

Em `src/shell/modules.ts` (sobre a Task 4), no fim do arquivo:

```ts
// Módulo 19b: contador de pendentes de assinatura no item do menu (a lista
// vem até 200 itens; dali em diante, "200+"). Só o rótulo muda; o NavDropdown
// continua o mesmo.
export function withPendingCount(groups: NavGroupDef[], count: number): NavGroupDef[] {
  if (count <= 0) return groups;
  const shown = count >= 200 ? "200+" : String(count);
  return groups.map((group) => ({
    ...group,
    items: group.items.map((item) => item.id === "signature-pending" ? { ...item, label: `${item.label} (${shown})` } : item)
  }));
}
```

Em `src/shell/AppHeader.tsx`:

```diff
--- a/src/shell/AppHeader.tsx
+++ b/src/shell/AppHeader.tsx
@@
 import { NotificationCenter } from "./NotificationCenter";
-import { navGroupsFor, type ModuleId } from "./modules";
+import { navGroupsFor, withPendingCount, type ModuleId } from "./modules";
+import { SignatureSessionBadge } from "./SignatureSessionBadge";
+import { usePendingSignatures } from "../hooks/usePendingSignatures";
+import { canSign } from "../lib/signature";
 import { PERIOD_OPTIONS, useScope } from "../lib/scope";
@@ export function AppHeader({ active, onSelect, alerts }: Props) {
   const [ openGroup, setOpenGroup ] = useState<string | null>(null);
   const navRef = useRef<HTMLElement | null>(null);
+  // Módulo 19b: a mesma consulta da página Pendentes (uma por minuto).
+  const pending = usePendingSignatures(canSign(auth.user));
@@
-          {navGroupsFor(auth.user).map((g) => {
+          {withPendingCount(navGroupsFor(auth.user), pending.data?.length ?? 0).map((g) => {
@@
+          <SignatureSessionBadge active={active} onSelect={onSelect} />
           <TenantChip user={auth.user} />
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod19b && npx vitest run src/shell/SignatureSessionBadge.test.tsx src/shell/modules.test.ts && npx tsc --noEmit`
Expected: PASS (6 testes no selo; `modules.test.ts` verde); `tsc` limpo.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19b add src/shell/SignatureSessionBadge.tsx src/shell/SignatureSessionBadge.test.tsx src/shell/AppHeader.tsx src/shell/modules.ts src/shell/modules.test.ts
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19b commit -m "feat: show the signing session badge and the pending signatures count in the header

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 6: Marcador na consulta e no adendo; "Ver o que foi assinado", downloads e "Revalidar"

**Files:**
- Create: `src/modules/signature/SignatureMarker.tsx`, `src/modules/signature/SignatureDetail.tsx`
- Modify: `src/modules/consultation/ConsultationView.tsx` (do 19a)
- Test: `src/modules/signature/SignatureMarker.test.tsx`, `src/modules/signature/SignatureDetail.test.tsx`, `src/modules/consultation/ConsultationView.signature.test.tsx`

**Interfaces:**
- Consumes: `getSignature`, `verifySignature`, `fetchSignatureFile`, `errorCode`, tipos (Task 1); `SIGNATURE_KEY`, `VERIFICATION_VIEW`, `documentLabel`, `printHint`, `saveBlob`, `signatureError`, `signatureFileName`, `signatureMarker` (Task 2); `CONSULTATION_KEY` (`src/lib/consultation.ts`, 19a); `ConsultationView` e `ConsultationViewProps` (19a, Task 8 de lá); `finalized`, `options` (`src/test/consultationFixtures.ts`, 19a); `fmtDateTime`, `Tag`, `KeyValue`, `formStyles`.
- Produces:
  - `SignatureMarker({ label: string; block: SignatureBlock; onOpeningRequired?(): void })` — `role="group"` com o nome `label`: o `Tag` do estado, o detalhe (quem e quando, ou o motivo) e, quando digital com `signature_id`, o botão "Ver o que foi assinado"/"Fechar o que foi assinado";
  - `SignatureDetail({ id: string; onOpeningRequired?(): void })` — seção "O que foi assinado": assinado por, CPF mascarado, assinado em, política, estado da validação e motivos, quando foi verificada, o conteúdo canônico (`<pre aria-label="conteúdo assinado">`), "Baixar PDF assinado", "Baixar .p7s" (o pacote `.zip` com `document.json` e `document.json.p7s`) e "Revalidar"; leitura com `gcTime: 0`; 403 `opening_required` chama `onOpeningRequired`;
  - `ConsultationView` mostra `SignatureMarker` "assinatura da consulta" logo depois do cabeçalho, o aviso do impresso (`printHint`) e um "assinatura do adendo" em cada adendo que tem o bloco. Sem o bloco (api sem o 19b), nada muda.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/signature/SignatureDetail.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor, within } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, getSignature: vi.fn(), verifySignature: vi.fn(), fetchSignatureFile: vi.fn() };
});

import * as api from "../../lib/api";
import { ApiError } from "../../lib/api";
import { SignatureDetail } from "./SignatureDetail";
import { signatureDetail } from "../../test/signatureFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
let downloads: string[];

afterEach(() => { cleanup(); vi.restoreAllMocks(); });

function wrap(ui: ReactNode) {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  render(<QueryClientProvider client={client}>{ui}</QueryClientProvider>);
}

describe("SignatureDetail", () => {
  beforeEach(() => {
    for (const fn of [ api.getSignature, api.verifySignature, api.fetchSignatureFile ]) mocked(fn).mockReset();
    mocked(api.getSignature).mockResolvedValue(signatureDetail());
    downloads = [];
    // O jsdom não tem createObjectURL/revokeObjectURL nem navega no clique do <a>.
    Object.defineProperty(URL, "createObjectURL", { configurable: true, value: vi.fn(() => "blob:file-1") });
    Object.defineProperty(URL, "revokeObjectURL", { configurable: true, value: vi.fn() });
    vi.spyOn(HTMLAnchorElement.prototype, "click").mockImplementation(function (this: HTMLAnchorElement) {
      downloads.push(this.download);
    });
  });

  it("mostra quem assinou, quando, a política, a validação e o conteúdo", async () => {
    wrap(<SignatureDetail id="sg1" />);
    const section = await screen.findByRole("region", { name: "O que foi assinado" });
    expect(within(section).getByText("Helena Prado")).not.toBeNull();
    expect(within(section).getByText("***.456.789-**")).not.toBeNull();
    expect(within(section).getByText("07/10/2026, 10:21")).not.toBeNull();
    expect(within(section).getByText("AD-RB")).not.toBeNull();
    expect(within(section).getByText("válida")).not.toBeNull();
    expect(within(section).getByText("verificada em 08/10/2026, 09:00")).not.toBeNull();
    expect(within(section).getByLabelText("conteúdo assinado").textContent).toMatch(/"assessment": "Diabetes descompensado\."/);
    expect(api.getSignature).toHaveBeenCalledWith("sg1");
  });

  it("baixa o PDF assinado e o pacote com o id no nome, nunca o nome do paciente", async () => {
    mocked(api.fetchSignatureFile).mockResolvedValue(new Blob([ "x" ]));
    wrap(<SignatureDetail id="sg1" />);
    fireEvent.click(await screen.findByRole("button", { name: "Baixar PDF assinado" }));
    await waitFor(() => expect(downloads).toEqual([ "assinatura-sg1.pdf" ]));
    expect(api.fetchSignatureFile).toHaveBeenCalledWith("sg1", "pdf");

    fireEvent.click(screen.getByRole("button", { name: "Baixar .p7s" }));
    await waitFor(() => expect(downloads).toEqual([ "assinatura-sg1.pdf", "assinatura-sg1.zip" ]));
    expect(api.fetchSignatureFile).toHaveBeenLastCalledWith("sg1", "package");
  });

  it("Revalidar devolve indeterminada: o estado muda na tela", async () => {
    mocked(api.verifySignature).mockResolvedValue(
      signatureDetail({ verification: "indeterminate", verification_reasons: [ "revocation_unknown" ], verified_at: "2026-10-08T10:05:00-03:00" }));
    wrap(<SignatureDetail id="sg1" />);
    fireEvent.click(await screen.findByRole("button", { name: "Revalidar" }));
    expect(await screen.findByText("revalidada: indeterminada")).not.toBeNull();
    const section = screen.getByRole("region", { name: "O que foi assinado" });
    expect(within(section).getByText("indeterminada")).not.toBeNull();
    expect(within(section).getByText("motivos: revocation_unknown")).not.toBeNull();
    expect(within(section).getByText("verificada em 08/10/2026, 10:05")).not.toBeNull();
    expect(api.verifySignature).toHaveBeenCalledWith("sg1");
  });

  it("fora do contexto: diz o caminho", async () => {
    mocked(api.getSignature).mockRejectedValue(new ApiError(403, { error: "out_of_context" }, "403"));
    wrap(<SignatureDetail id="sg1" />);
    expect((await screen.findByRole("alert")).textContent)
      .toBe("fora do atendimento, abra o prontuário com motivo para ver o que foi assinado");
  });

  it("abertura justificada vencida: avisa quem abriu", async () => {
    mocked(api.getSignature).mockRejectedValue(new ApiError(403, { error: "opening_required" }, "403"));
    const onOpeningRequired = vi.fn();
    wrap(<SignatureDetail id="sg1" onOpeningRequired={onOpeningRequired} />);
    await waitFor(() => expect(onOpeningRequired).toHaveBeenCalled());
  });
});
```

```tsx
// src/modules/signature/SignatureMarker.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, within } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, getSignature: vi.fn() };
});

import * as api from "../../lib/api";
import { SignatureMarker } from "./SignatureMarker";
import { signatureBlock, signatureDetail } from "../../test/signatureFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
afterEach(cleanup);

function wrap(ui: ReactNode) {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  render(<QueryClientProvider client={client}>{ui}</QueryClientProvider>);
}

describe("SignatureMarker", () => {
  beforeEach(() => { mocked(api.getSignature).mockReset(); mocked(api.getSignature).mockResolvedValue(signatureDetail()); });

  it("digital: estado, quem e quando; abre e fecha o que foi assinado", async () => {
    wrap(<SignatureMarker label="assinatura da consulta" block={signatureBlock()} />);
    const group = screen.getByRole("group", { name: "assinatura da consulta" });
    expect(within(group).getByText("assinada digitalmente")).not.toBeNull();
    expect(within(group).getByText("Helena Prado · 07/10/2026, 10:21")).not.toBeNull();
    fireEvent.click(within(group).getByRole("button", { name: "Ver o que foi assinado" }));
    expect(await screen.findByRole("region", { name: "O que foi assinado" })).not.toBeNull();
    fireEvent.click(within(group).getByRole("button", { name: "Fechar o que foi assinado" }));
    expect(screen.queryByRole("region", { name: "O que foi assinado" })).toBeNull();
  });

  it("indeterminada aparece com o estado, sem esconder", () => {
    wrap(<SignatureMarker label="assinatura da consulta" block={signatureBlock({ verification: "indeterminate" })} />);
    expect(screen.getByText("assinatura digital indeterminada")).not.toBeNull();
    expect(screen.getByRole("button", { name: "Ver o que foi assinado" })).not.toBeNull();
  });

  it("pendente e à mão: sem botão de conteúdo", () => {
    wrap(<SignatureMarker label="assinatura do adendo" block={{ mode: "pending", request_id: "sr2", reason_code: "session_expired" }} />);
    expect(screen.getByText("assinatura pendente")).not.toBeNull();
    expect(screen.getByText("a sessão de assinatura venceu")).not.toBeNull();
    expect(screen.queryByRole("button", { name: "Ver o que foi assinado" })).toBeNull();
    expect(api.getSignature).not.toHaveBeenCalled();
  });
});
```

```tsx
// src/modules/consultation/ConsultationView.signature.test.tsx
// O marcador do 19b dentro da ConsultationView real do 19a.
import { afterEach, describe, expect, it, vi } from "vitest";
import { cleanup, render, screen, within } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";

vi.mock("../../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../../lib/api")>();
  return { ...real, fetchConsultationPdf: vi.fn(), getConsultation: vi.fn(), addAddendum: vi.fn(), getSignature: vi.fn() };
});

import { ConsultationView } from "./ConsultationView";
import { finalized, options } from "../../test/consultationFixtures";
import { signatureBlock } from "../../test/signatureFixtures";

afterEach(cleanup);

function wrap(ui: ReactNode) {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  render(<QueryClientProvider client={client}>{ui}</QueryClientProvider>);
}
const addendum = { id: "ad1", author_name: "Enf. Lúcia Prado", created_at: "2026-10-07T11:00:00-03:00",
  reason: "correção do plano", text: "Retorno em 15 dias.", changes: null };
const view = (c: ReturnType<typeof finalized>) =>
  wrap(<ConsultationView consultation={c} options={options()} patientProblems={[]} canAddendum={false}
    onAddendumAdded={vi.fn()} onClose={vi.fn()} />);

describe("ConsultationView com assinatura digital (19b)", () => {
  it("marcador na consulta e em cada adendo; o impresso avisa o caminho do adendo", () => {
    view(finalized({ signature: signatureBlock(), addenda: [
      { ...addendum, signature: { mode: "pending", request_id: "sr2", reason_code: "no_session" } } ] }));
    const consultation = screen.getByRole("group", { name: "assinatura da consulta" });
    expect(within(consultation).getByText("assinada digitalmente")).not.toBeNull();
    const added = screen.getByRole("group", { name: "assinatura do adendo" });
    expect(within(added).getByText("assinatura pendente")).not.toBeNull();
    expect(within(added).getByText("sem sessão de assinatura aberta")).not.toBeNull();
    expect(screen.getByText("com adendo, o impresso é o do prontuário; os documentos assinados estão em “Ver o que foi assinado”"))
      .not.toBeNull();
  });

  it("consulta digital sem adendo: o impresso é o PDF assinado", () => {
    view(finalized({ signature: signatureBlock() }));
    expect(screen.getByText("o impresso é o PDF assinado digitalmente")).not.toBeNull();
  });

  it("voltou ao papel porque o interruptor foi desligado", () => {
    view(finalized({ signature: { mode: "manual", request_id: "sr1", reason_code: "feature_disabled" } }));
    const group = screen.getByRole("group", { name: "assinatura da consulta" });
    expect(within(group).getByText("assinatura à mão (papel)")).not.toBeNull();
    expect(within(group).getByText("assinatura digital desligada na cidade")).not.toBeNull();
    expect(screen.getByText("o impresso sai com espaço para assinatura à mão")).not.toBeNull();
  });

  it("api sem o 19b (sem o bloco): nada de assinatura na tela", () => {
    view(finalized({ addenda: [ addendum ] }));
    expect(screen.queryByRole("group", { name: "assinatura da consulta" })).toBeNull();
    expect(screen.queryByRole("group", { name: "assinatura do adendo" })).toBeNull();
    expect(screen.queryByText(/o impresso/)).toBeNull();
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod19b && npx vitest run src/modules/signature src/modules/consultation/ConsultationView.signature.test.tsx`
Expected: FAIL — `Failed to resolve import "./SignatureDetail"` (e `./SignatureMarker`); na `ConsultationView`, `Unable to find role="group" and name "assinatura da consulta"`.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/signature/SignatureDetail.tsx
// O que foi assinado (módulo 19b; spec §5 "Validação", §7 e §9; contrato §6):
// quem, quando, política, validação e o JSON canônico; PDF assinado, pacote
// (.json + .p7s) e Revalidar. Ler passa pelo controle de acesso do prontuário
// no api (trilha `clinical_record.viewed`); o conteúdo some ao fechar (gcTime 0).
import { useEffect, useState, type CSSProperties } from "react";
import { useQuery, useQueryClient } from "@tanstack/react-query";
import { errorCode, fetchSignatureFile, getSignature, verifySignature, type SignatureFileKind } from "../../lib/api";
import { fmtDateTime } from "../../lib/format";
import { CONSULTATION_KEY } from "../../lib/consultation";
import {
  SIGNATURE_KEY, VERIFICATION_VIEW, saveBlob, signatureError, signatureFileName
} from "../../lib/signature";
import { KeyValue } from "../../components/KeyValue";
import { Tag } from "../../components/Tag";
import { buttonStyle, disabledButtonStyle, secondaryButtonStyle } from "../../components/formStyles";

interface Props {
  id: string;
  onOpeningRequired?(): void;
}

export function SignatureDetail({ id, onOpeningRequired }: Props) {
  const queryClient = useQueryClient();
  const key = [ SIGNATURE_KEY, id ];
  const query = useQuery({ queryKey: key, queryFn: () => getSignature(id), gcTime: 0, staleTime: 0 });
  const [ busy, setBusy ] = useState<SignatureFileKind | "verify" | null>(null);
  const [ message, setMessage ] = useState<string | null>(null);
  const [ failure, setFailure ] = useState<string | null>(null);
  const openingGone = query.isError && errorCode(query.error) === "opening_required";

  useEffect(() => {
    if (openingGone) onOpeningRequired?.();
  }, [ openingGone, onOpeningRequired ]);

  function fail(err: unknown) {
    if (errorCode(err) === "opening_required") onOpeningRequired?.();
    setFailure(signatureError(err));
  }

  async function download(kind: SignatureFileKind) {
    if (busy) return;
    setBusy(kind); setFailure(null); setMessage(null);
    try {
      saveBlob(await fetchSignatureFile(id, kind), signatureFileName(id, kind));
    } catch (err) {
      fail(err);
    } finally {
      setBusy(null);
    }
  }

  async function revalidate() {
    if (busy) return;
    setBusy("verify"); setFailure(null); setMessage(null);
    try {
      const fresh = await verifySignature(id);
      queryClient.setQueryData(key, fresh);
      setMessage(`revalidada: ${(VERIFICATION_VIEW[fresh.verification] ?? { label: fresh.verification }).label}`);
      void queryClient.invalidateQueries({ queryKey: [ CONSULTATION_KEY ] });
    } catch (err) {
      fail(err);
    } finally {
      setBusy(null);
    }
  }

  if (query.isPending) return <p style={muted}>carregando o que foi assinado…</p>;
  if (query.isError) return <p role="alert" style={alert}>{signatureError(query.error)}</p>;

  const s = query.data;
  const verification = VERIFICATION_VIEW[s.verification] ?? { label: s.verification, tone: "neutral" };

  return (
    <section aria-label="O que foi assinado" style={panel}>
      <div style={grid}>
        <KeyValue k="Assinado por" v={s.signer_name} mono={false} />
        <KeyValue k="CPF" v={s.signer_cpf_masked} />
        <KeyValue k="Assinado em" v={fmtDateTime(s.signed_at)} />
        <KeyValue k="Política" v={s.policy} />
      </div>
      <div style={row}>
        <Tag tone={verification.tone}>{verification.label}</Tag>
        <span style={muted}>{`verificada em ${fmtDateTime(s.verified_at)}`}</span>
      </div>
      {s.verification_reasons.length > 0 && <p style={muted}>{`motivos: ${s.verification_reasons.join(", ")}`}</p>}
      <pre aria-label="conteúdo assinado" style={pre}>{JSON.stringify(s.content, null, 2)}</pre>
      {message && <p role="status" style={text}>{message}</p>}
      {failure && <p role="alert" style={alert}>{failure}</p>}
      <div style={row}>
        <button type="button" disabled={busy !== null} style={busy ? disabledButtonStyle : buttonStyle}
          onClick={() => void download("pdf")}>Baixar PDF assinado</button>
        <button type="button" disabled={busy !== null} style={secondaryButtonStyle}
          onClick={() => void download("package")}>Baixar .p7s</button>
        <button type="button" disabled={busy !== null} style={secondaryButtonStyle}
          onClick={() => void revalidate()}>Revalidar</button>
      </div>
      <p style={muted}>O pacote traz o documento (.json) e a assinatura (.p7s). Os dois arquivos podem ser conferidos em validar.iti.gov.br.</p>
    </section>
  );
}

const panel: CSSProperties = { display: "flex", flexDirection: "column", gap: 10, padding: 12, border: "1px solid var(--rule)", borderRadius: 8 };
const grid: CSSProperties = { display: "grid", gridTemplateColumns: "repeat(auto-fit, minmax(150px, 1fr))", gap: 12 };
const row: CSSProperties = { display: "flex", gap: 8, alignItems: "center", flexWrap: "wrap" };
const pre: CSSProperties = {
  margin: 0, padding: 10, maxHeight: 320, overflow: "auto", fontSize: 11.5, background: "var(--sunken)",
  border: "1px solid var(--rule)", borderRadius: 6, whiteSpace: "pre-wrap"
};
const text: CSSProperties = { margin: 0, fontSize: 12.5 };
const muted: CSSProperties = { margin: 0, fontSize: 12, color: "var(--ink3)" };
const alert: CSSProperties = { margin: 0, fontSize: 12.5, color: "var(--down)" };
```

```tsx
// src/modules/signature/SignatureMarker.tsx
// Marcador de assinatura de um documento (módulo 19b; spec §9; contrato §2):
// digital (válida, inválida, indeterminada), pendente (com o motivo) ou à mão.
// Só a assinatura digital tem conteúdo para abrir.
import { useState, type CSSProperties } from "react";
import type { SignatureBlock } from "../../lib/api";
import { signatureMarker } from "../../lib/signature";
import { Tag } from "../../components/Tag";
import { SignatureDetail } from "./SignatureDetail";

interface Props {
  label: string;
  block: SignatureBlock;
  onOpeningRequired?(): void;
}

export function SignatureMarker({ label, block, onOpeningRequired }: Props) {
  const [ open, setOpen ] = useState(false);
  const marker = signatureMarker(block);
  const canOpen = block.mode === "digital" && !!block.signature_id;

  return (
    <div role="group" aria-label={label} style={wrap}>
      <div style={row}>
        <Tag tone={marker.tone}>{marker.label}</Tag>
        {marker.detail && <span style={muted}>{marker.detail}</span>}
        {canOpen && (
          <button type="button" style={linkButton} onClick={() => setOpen((v) => !v)}>
            {open ? "Fechar o que foi assinado" : "Ver o que foi assinado"}
          </button>
        )}
      </div>
      {open && block.signature_id && <SignatureDetail id={block.signature_id} onOpeningRequired={onOpeningRequired} />}
    </div>
  );
}

const wrap: CSSProperties = { display: "flex", flexDirection: "column", gap: 8 };
const row: CSSProperties = { display: "flex", gap: 8, alignItems: "center", flexWrap: "wrap" };
const muted: CSSProperties = { fontSize: 12, color: "var(--ink3)" };
const linkButton: CSSProperties = {
  border: "none", background: "transparent", padding: 0, cursor: "pointer", fontSize: 12,
  color: "var(--accent)", textDecoration: "underline"
};
```

Em `src/modules/consultation/ConsultationView.tsx` (do 19a; confira o texto real, como diz a Task 0):

```diff
--- a/src/modules/consultation/ConsultationView.tsx
+++ b/src/modules/consultation/ConsultationView.tsx
@@
 import { VitalsList } from "./PatientPanel";
 import { AddendumForm } from "./AddendumForm";
+import { SignatureMarker } from "../signature/SignatureMarker";
+import { printHint } from "../../lib/signature";
@@ export function ConsultationView(props: ConsultationViewProps) {
   const conductLabel = (code: string) => codedLabel(options?.conducts, code);
   const addenda = [ ...c.addenda ].sort((a, b) => a.created_at.localeCompare(b.created_at));
+  // Módulo 19b: como sai o impresso (o api escolhe; contrato 19b §6).
+  const hint = printHint(c.signature, c.addenda.length > 0);
@@
       <div style={{ display: "flex", gap: 12, alignItems: "baseline", flexWrap: "wrap" }}>
         <strong>{`Consulta de ${fmtDateTime(c.finalized_at)}`}</strong>
         <span style={muted}>{`${c.author.name} · ${codedLabel(options?.care_types, c.care_type)}`}</span>
       </div>
+      {c.signature && (
+        <SignatureMarker label="assinatura da consulta" block={c.signature} onOpeningRequired={props.onOpeningRequired} />
+      )}
+      {hint && <p style={muted}>{hint}</p>}
@@
             <li key={a.id} style={{ fontSize: 12.5 }}>
               <strong>{`Adendo de ${a.author_name} em ${fmtDateTime(a.created_at)}`}</strong>
+              {a.signature && (
+                <SignatureMarker label="assinatura do adendo" block={a.signature} onOpeningRequired={props.onOpeningRequired} />
+              )}
               <p style={textStyle}>{`Motivo: ${a.reason}`}</p>
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod19b && npx vitest run src/modules/signature src/modules/consultation && npx tsc --noEmit`
Expected: PASS (5 + 3 + 4 testes novos; os testes do 19a em `src/modules/consultation` continuam verdes — as fixtures dele não trazem `signature`); `tsc` limpo.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19b add src/modules/signature/SignatureMarker.tsx src/modules/signature/SignatureDetail.tsx src/modules/signature/SignatureMarker.test.tsx src/modules/signature/SignatureDetail.test.tsx src/modules/consultation/ConsultationView.tsx src/modules/consultation/ConsultationView.signature.test.tsx
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19b commit -m "feat: show the signature state on consultations and addenda with the signed content

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 7: Painel de assinatura do `municipal_admin`

**Files:**
- Create: `src/modules/SignatureOverview.tsx`
- Modify: `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/App.tsx`
- Test: `src/modules/SignatureOverview.test.tsx`

**Interfaces:**
- Consumes: `getSignatureOverview`, tipos (Task 1); `OVERVIEW_KEY`, `SIGNATURE_DISABLED`, `CERTIFICATE_STATUS_VIEW`, `VERIFICATION_VIEW`, `canSeeSignatureOverview`, `defaultOverviewPeriod`, `documentLabel`, `fmtDay`, `isPendingOverdue`, `overviewSummary`, `signatureError` (Task 2); `useAuth`, `hasFeature`, `fmtDateTime`, `PageHeader`, `Panel`, `KeyValue`, `DataTable`, `Tag`, `EmptyState`, `formStyles`.
- Produces:
  - `SignatureOverview()` — página "Painel de assinatura" (só leitura): "De"/"Até" (padrão: últimos 30 dias no fuso da cidade; início depois do fim não consulta e avisa); `Panel` "Profissionais" com o resumo (com certificado, sem certificado, vencendo em 30 dias, pendentes há mais de 24 h) e a tabela (profissional, certificado, válido até, pendentes, mais antiga com "há mais de 24 h"); `Panel` "Documentos por modo" (digital, papel, pendente); `Panel` "Assinaturas inválidas ou indeterminadas";
  - `ModuleId` com `"signature-overview"`; item "Painel de assinatura" no grupo Equipe, só com `canSeeSignatureOverview(user)`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/SignatureOverview.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, screen, waitFor, within } from "@testing-library/react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, fetchCurrentSession: vi.fn(), getSignatureOverview: vi.fn() };
});

import * as api from "../lib/api";
import { SignatureOverview } from "./SignatureOverview";
import { renderWithProviders } from "../test/campaignFixtures";
import { NOW19B, overview, signer } from "../test/signatureFixtures";
import { SIGNATURE_DISABLED } from "../lib/signature";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const valueOf = (label: string) => screen.getByText(label).parentElement?.textContent?.replace(label, "");

afterEach(() => { cleanup(); vi.useRealTimers(); });

describe("SignatureOverview", () => {
  beforeEach(() => {
    vi.clearAllMocks();
    vi.useFakeTimers({ toFake: [ "Date" ] });
    vi.setSystemTime(new Date(NOW19B));
    mocked(api.fetchCurrentSession).mockResolvedValue(signer([ "municipal_admin" ]));
    mocked(api.getSignatureOverview).mockResolvedValue(overview());
  });

  it("pede os últimos 30 dias e resume os profissionais", async () => {
    renderWithProviders(<SignatureOverview />);
    await waitFor(() => expect(api.getSignatureOverview).toHaveBeenCalledWith({ from: "2026-09-08", to: "2026-10-08" }));
    await screen.findByText("Com certificado");
    expect(valueOf("Com certificado")).toBe("2");
    expect(valueOf("Sem certificado")).toBe("1");
    expect(valueOf("Vencendo em 30 dias")).toBe("1");
    expect(valueOf("Pendentes há mais de 24 h")).toBe("1");
  });

  it("tabela de profissionais: certificado, validade, pendentes e a mais antiga atrasada", async () => {
    renderWithProviders(<SignatureOverview />);
    const panel = await screen.findByRole("region", { name: "Profissionais" });
    await within(panel).findByText("Lúcia Prado");
    expect(within(panel).getByText("vence em até 30 dias")).not.toBeNull();
    expect(within(panel).getByText("20/10/2026")).not.toBeNull();
    expect(within(panel).getByText("06/10/2026, 16:00")).not.toBeNull();
    expect(within(panel).getByText("há mais de 24 h")).not.toBeNull();
    expect(within(panel).getByText("sem certificado")).not.toBeNull();
    expect(within(panel).getByText("ativo")).not.toBeNull();
  });

  it("documentos por modo e assinaturas inválidas ou indeterminadas", async () => {
    renderWithProviders(<SignatureOverview />);
    const modes = await screen.findByRole("region", { name: "Documentos por modo" });
    await within(modes).findByText("42");
    expect(within(modes).getByText("17")).not.toBeNull();
    expect(within(modes).getByText("3")).not.toBeNull();
    const invalid = screen.getByRole("region", { name: "Assinaturas inválidas ou indeterminadas" });
    expect(within(invalid).getByText("adendo")).not.toBeNull();
    expect(within(invalid).getByText("Lúcia Prado")).not.toBeNull();
    expect(within(invalid).getByText("indeterminada")).not.toBeNull();
    expect(within(invalid).getByText("08/10/2026, 08:00")).not.toBeNull();
  });

  it("nenhuma inválida: diz", async () => {
    mocked(api.getSignatureOverview).mockResolvedValue(overview({ invalid_or_indeterminate: [] }));
    renderWithProviders(<SignatureOverview />);
    expect(await screen.findByText("nenhuma assinatura inválida ou indeterminada no período")).not.toBeNull();
  });

  it("início depois do fim: avisa e não consulta", async () => {
    renderWithProviders(<SignatureOverview />);
    await waitFor(() => expect(api.getSignatureOverview).toHaveBeenCalledTimes(1));
    fireEvent.change(screen.getByLabelText("De"), { target: { value: "2026-10-09" } });
    expect(await screen.findByText("a data inicial vem depois da final")).not.toBeNull();
    expect(api.getSignatureOverview).toHaveBeenCalledTimes(1);
  });

  it("muda o período e consulta de novo", async () => {
    renderWithProviders(<SignatureOverview />);
    await waitFor(() => expect(api.getSignatureOverview).toHaveBeenCalledTimes(1));
    fireEvent.change(screen.getByLabelText("De"), { target: { value: "2026-10-01" } });
    await waitFor(() => expect(api.getSignatureOverview).toHaveBeenLastCalledWith({ from: "2026-10-01", to: "2026-10-08" }));
  });

  it("sem a funcionalidade, ou sem ser admin: não consulta", async () => {
    mocked(api.fetchCurrentSession).mockResolvedValue(signer([ "municipal_admin" ], { features: [] }));
    renderWithProviders(<SignatureOverview />);
    await waitFor(() => expect(api.fetchCurrentSession).toHaveBeenCalled());
    expect(await screen.findByText(SIGNATURE_DISABLED)).not.toBeNull();
    expect(api.getSignatureOverview).not.toHaveBeenCalled();
  });
});
```

E o menu, no `describe("assinatura digital no menu (módulo 19b)")` de `src/shell/modules.test.ts`:

```ts
  it("Equipe → Painel de assinatura: só municipal_admin com digital_signature", () => {
    const equipe = NAV_GROUPS.find((g) => g.label === "Equipe");
    expect(equipe?.items.map((i) => i.id)).toContain("signature-overview");
    expect(labelFor("signature-overview")).toBe("Painel de assinatura");
    expect(ids(u([ "municipal_admin" ]))).toContain("signature-overview");
    expect(ids(u([ "municipal_admin" ], []))).not.toContain("signature-overview");
    expect(ids(u([ "health_professional" ]))).not.toContain("signature-overview");
  });
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod19b && npx vitest run src/modules/SignatureOverview.test.tsx src/shell/modules.test.ts`
Expected: FAIL — `Failed to resolve import "./SignatureOverview"`; no menu, `expected [...] to include 'signature-overview'`.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/SignatureOverview.tsx
// Painel de assinatura do municipal_admin (módulo 19b, F-19.14; spec §9;
// contrato §7): quem tem certificado, quem vence em 30 dias, pendentes há
// mais de 24 h, documentos por modo e assinaturas inválidas/indeterminadas.
// Só leitura; o período é no fuso da cidade (padrão: 30 dias).
import { useState, type CSSProperties } from "react";
import { useQuery } from "@tanstack/react-query";
import { getSignatureOverview, type OverviewProfessional, type OverviewSignatureRow } from "../lib/api";
import { useAuth } from "../lib/auth";
import { hasFeature } from "../lib/features";
import { fmtDateTime } from "../lib/format";
import {
  CERTIFICATE_STATUS_VIEW, OVERVIEW_KEY, SIGNATURE_DISABLED, VERIFICATION_VIEW, canSeeSignatureOverview,
  defaultOverviewPeriod, documentLabel, fmtDay, isPendingOverdue, overviewSummary, signatureError
} from "../lib/signature";
import { PageHeader } from "../components/PageHeader";
import { Panel } from "../components/Panel";
import { KeyValue } from "../components/KeyValue";
import { DataTable, type Column } from "../components/DataTable";
import { Tag } from "../components/Tag";
import { EmptyState } from "../components/EmptyState";
import { inputStyle } from "../components/formStyles";

export function SignatureOverview() {
  const { user } = useAuth();
  const allowed = canSeeSignatureOverview(user);
  const [ period, setPeriod ] = useState(() => defaultOverviewPeriod(Date.now()));
  const inverted = !!period.from && !!period.to && period.from > period.to;
  const query = useQuery({
    queryKey: [ OVERVIEW_KEY, period.from, period.to ],
    queryFn: () => getSignatureOverview({ from: period.from, to: period.to }),
    enabled: allowed && !inverted && !!period.from && !!period.to
  });

  if (!allowed) {
    return (
      <div style={page}>
        <PageHeader title="Painel de assinatura" sub="cidade · assinatura digital · só leitura" />
        <EmptyState title={hasFeature(user, "digital_signature") ? "só o administrador municipal vê este painel" : SIGNATURE_DISABLED} />
      </div>
    );
  }

  const nowMs = Date.now();
  const data = query.data;
  const summary = data ? overviewSummary(data, nowMs) : null;

  const professionalCols: Column<OverviewProfessional>[] = [
    { label: "Profissional", w: "2fr", render: (p) => p.name },
    {
      label: "Certificado", w: "1.5fr", render: (p) => {
        const view = CERTIFICATE_STATUS_VIEW[p.certificate_status] ?? { label: p.certificate_status, tone: "neutral" };
        return <Tag tone={view.tone}>{view.label}</Tag>;
      }
    },
    { label: "Válido até", w: "1fr", render: (p) => fmtDay(p.not_after) },
    { label: "Pendentes", w: "0.8fr", align: "right", render: (p) => String(p.pending_count) },
    {
      label: "Pendente mais antiga", w: "2fr", render: (p) => p.oldest_pending_at ? (
        <span style={row}>
          <span>{fmtDateTime(p.oldest_pending_at)}</span>
          {isPendingOverdue(p.oldest_pending_at, nowMs) && <Tag tone="warn">há mais de 24 h</Tag>}
        </span>
      ) : "—"
    }
  ];

  const invalidCols: Column<OverviewSignatureRow>[] = [
    { label: "Documento", w: "1fr", render: (r) => documentLabel(r.document_type) },
    { label: "Assinado por", w: "2fr", render: (r) => r.signer_name },
    {
      label: "Estado", w: "1fr", render: (r) => {
        const view = VERIFICATION_VIEW[r.verification] ?? { label: r.verification, tone: "neutral" };
        return <Tag tone={view.tone}>{view.label}</Tag>;
      }
    },
    { label: "Verificado em", w: "1.4fr", render: (r) => fmtDateTime(r.verified_at) }
  ];

  return (
    <div style={page}>
      <PageHeader
        title="Painel de assinatura"
        sub="cidade · assinatura digital · só leitura"
        right={(
          <div style={row}>
            <label style={label}>
              De
              <input type="date" value={period.from} onChange={(e) => setPeriod((p) => ({ ...p, from: e.target.value }))} style={dateInput} />
            </label>
            <label style={label}>
              Até
              <input type="date" value={period.to} onChange={(e) => setPeriod((p) => ({ ...p, to: e.target.value }))} style={dateInput} />
            </label>
          </div>
        )}
      />
      {inverted && <p role="alert" style={alert}>a data inicial vem depois da final</p>}
      {query.isError && <p role="alert" style={alert}>{signatureError(query.error)}</p>}
      {query.isPending && !inverted && <p style={muted}>carregando…</p>}

      {data && summary && (
        <>
          <Panel title="Profissionais" sub="certificado e pendentes de cada profissional">
            <div style={body}>
              <div style={grid}>
                <KeyValue k="Com certificado" v={String(summary.withCertificate)} />
                <KeyValue k="Sem certificado" v={String(summary.withoutCertificate)} />
                <KeyValue k="Vencendo em 30 dias" v={String(summary.expiring)} />
                <KeyValue k="Pendentes há mais de 24 h" v={String(summary.pendingOverdue)} />
              </div>
              <DataTable cols={professionalCols} rows={data.professionals} rowKey={(p) => p.user_id}
                empty="nenhum profissional de saúde na cidade" />
            </div>
          </Panel>

          <Panel title="Documentos por modo" sub="consultas e adendos finalizados no período">
            <div style={grid}>
              <KeyValue k="Digital" v={String(data.documents_by_mode.digital ?? 0)} />
              <KeyValue k="Papel" v={String(data.documents_by_mode.manual ?? 0)} />
              <KeyValue k="Pendente" v={String(data.documents_by_mode.pending ?? 0)} />
            </div>
          </Panel>

          <Panel title="Assinaturas inválidas ou indeterminadas" sub="na última validação">
            <DataTable cols={invalidCols} rows={data.invalid_or_indeterminate} rowKey={(r) => r.signature_id}
              empty="nenhuma assinatura inválida ou indeterminada no período" />
          </Panel>
        </>
      )}
    </div>
  );
}

const page: CSSProperties = { display: "flex", flexDirection: "column", gap: 16 };
const body: CSSProperties = { display: "flex", flexDirection: "column", gap: 12 };
const grid: CSSProperties = { display: "grid", gridTemplateColumns: "repeat(auto-fit, minmax(160px, 1fr))", gap: 12 };
const row: CSSProperties = { display: "flex", gap: 8, alignItems: "center", flexWrap: "wrap" };
const label: CSSProperties = { display: "flex", flexDirection: "column", gap: 2, fontSize: 11, color: "var(--ink3)" };
const dateInput: CSSProperties = { ...inputStyle, marginTop: 0, width: 150 };
const muted: CSSProperties = { margin: 0, fontSize: 12, color: "var(--ink3)" };
const alert: CSSProperties = { margin: 0, fontSize: 12.5, color: "var(--down)" };
```

Em `src/shell/modules.ts` (sobre a Task 5):

```diff
--- a/src/shell/modules.ts
+++ b/src/shell/modules.ts
@@
-import { canSign } from "../lib/signature";
+import { canSeeSignatureOverview, canSign } from "../lib/signature";
@@
-  | "signature" | "signature-pending";
+  | "signature" | "signature-pending" | "signature-overview";
@@
   { label: "Equipe", items: [
     { id: "team", label: "Equipe", icon: "☷" },
-    { id: "professionals", label: "Profissionais", icon: "✚" }
+    { id: "professionals", label: "Profissionais", icon: "✚" },
+    { id: "signature-overview", label: "Painel de assinatura", icon: "✍" }
   ]},
@@ export function navGroupsFor(
       if (item.id === "signature-pending") return canSign(user);
+      // F-19.14: painel só leitura do municipal_admin, com digital_signature.
+      if (item.id === "signature-overview") return canSeeSignatureOverview(user);
```

Em `src/App.tsx`:

```diff
--- a/src/App.tsx
+++ b/src/App.tsx
@@
 import { SignaturePending } from "./modules/SignaturePending";
+import { SignatureOverview } from "./modules/SignatureOverview";
@@ function renderModule(active: ModuleId, setActive: (id: ModuleId) => void) {
     case "signature-pending": return <SignaturePending onNavigate={setActive} />;
+    case "signature-overview": return <SignatureOverview />;
     default:               return <Placeholder title={labelFor(active)} />;
```

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod19b && npx vitest run src/modules/SignatureOverview.test.tsx src/shell/modules.test.ts && npx tsc --noEmit`
Expected: PASS (7 testes na página); `tsc` limpo.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19b add src/modules/SignatureOverview.tsx src/modules/SignatureOverview.test.tsx src/shell/modules.ts src/shell/modules.test.ts src/App.tsx
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19b commit -m "feat: add the digital signature overview for the municipal admin

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 8: Retorno do prestador (`/dashboard/signature/callback`) e volta à tela de origem

**Files:**
- Create: `src/modules/SignatureCallback.tsx`
- Modify: `src/shell/modules.ts`, `src/shell/modules.test.ts`, `src/main.tsx`, `src/App.tsx`
- Test: `src/modules/SignatureCallback.test.tsx`

**Interfaces:**
- Consumes: `completeSignatureOAuth`, `BatchResult` (Task 1); `CERTIFICATE_KEY`, `SESSION_KEY`, `PENDING_KEY`, `SignatureCallbackParams`, `callbackLanding`, `callbackSummary`, `clearSignatureCallbackFromUrl`, `oauthErrorPhrase`, `readSignatureCallback`, `reasonLabel`, `signatureError` (Task 2); `NAV_GROUPS`, `ModuleId` (`src/shell/modules.ts`); `buttonStyle`.
- Produces:
  - `moduleFromPath(path: string | null | undefined): ModuleId | null` em `src/shell/modules.ts` — `/<id>` de um módulo do catálogo, senão `null`;
  - `SignatureCallback({ params: SignatureCallbackParams; onDone(landing: ModuleId): void; clearUrl?(): void })` — seção "Retorno do prestador de assinatura": limpa a URL, troca `state`+`code` **uma vez** (guarda por `ref`, resiste ao `StrictMode`), diz o resultado (`role="status"`, com a lista "não assinados" do lote) e "Continuar" leva ao destino; erro do prestador ou recusa do api (`role="alert"`) com "Voltar ao painel";
  - `App({ initialModule?: ModuleId })` — começa no módulo pedido (padrão "overview");
  - `main.tsx` desvia para `SignatureCallback` quando a URL é a de retorno, depois do login, dentro do `QueryClientProvider`.

- [ ] **Step 1: Write the failing test**

```tsx
// src/modules/SignatureCallback.test.tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, fireEvent, render, screen, waitFor } from "@testing-library/react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { StrictMode, type ReactNode } from "react";

vi.mock("../lib/api", async (importOriginal) => {
  const real = await importOriginal<typeof import("../lib/api")>();
  return { ...real, completeSignatureOAuth: vi.fn() };
});

import * as api from "../lib/api";
import { ApiError } from "../lib/api";
import { SignatureCallback } from "./SignatureCallback";
import { certificate } from "../test/signatureFixtures";

const mocked = (fn: unknown) => fn as ReturnType<typeof vi.fn>;
const OK = { state: "st1", code: "c1", error: null };

afterEach(cleanup);

function wrap(ui: ReactNode) {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  render(<QueryClientProvider client={client}>{ui}</QueryClientProvider>);
}

describe("SignatureCallback", () => {
  let onDone: ReturnType<typeof vi.fn>;
  let clearUrl: ReturnType<typeof vi.fn>;

  beforeEach(() => {
    mocked(api.completeSignatureOAuth).mockReset();
    onDone = vi.fn();
    clearUrl = vi.fn();
  });

  it("vínculo: limpa a URL antes de trocar o código, diz o resultado e volta à Conta", async () => {
    mocked(api.completeSignatureOAuth).mockResolvedValue({ purpose: "link", result: certificate(), return_to: "/signature" });
    wrap(<SignatureCallback params={OK} onDone={onDone} clearUrl={clearUrl} />);
    expect((await screen.findByRole("status")).textContent).toBe("Certificado VIDaaS vinculado, válido até 15/03/2027.");
    expect(api.completeSignatureOAuth).toHaveBeenCalledWith("st1", "c1");
    expect(clearUrl.mock.invocationCallOrder[0]).toBeLessThan(mocked(api.completeSignatureOAuth).mock.invocationCallOrder[0]);
    fireEvent.click(screen.getByRole("button", { name: "Continuar" }));
    expect(onDone).toHaveBeenCalledWith("signature");
  });

  it("StrictMode: troca o código uma vez só", async () => {
    mocked(api.completeSignatureOAuth).mockResolvedValue({ purpose: "session", result: { expires_at: "2026-10-08T22:00:00-03:00" } });
    wrap(<StrictMode><SignatureCallback params={OK} onDone={onDone} clearUrl={clearUrl} /></StrictMode>);
    expect(await screen.findByText("Sessão de assinatura aberta até 22:00.")).not.toBeNull();
    expect(api.completeSignatureOAuth).toHaveBeenCalledTimes(1);
    expect(clearUrl).toHaveBeenCalledTimes(1);
  });

  it("sessão sem return_to devolvido: volta à visão geral", async () => {
    mocked(api.completeSignatureOAuth).mockResolvedValue({ purpose: "session", result: { expires_at: "2026-10-08T22:00:00-03:00" } });
    wrap(<SignatureCallback params={OK} onDone={onDone} clearUrl={clearUrl} />);
    fireEvent.click(await screen.findByRole("button", { name: "Continuar" }));
    expect(onDone).toHaveBeenCalledWith("overview");
  });

  it("lote: quantos foram assinados, o que ficou e volta às Pendentes", async () => {
    mocked(api.completeSignatureOAuth).mockResolvedValue({
      purpose: "batch", return_to: "/signature-pending",
      result: { signed: 2, failed: [ { request_id: "sr9", reason_code: "provider_unavailable" } ] }
    });
    wrap(<SignatureCallback params={OK} onDone={onDone} clearUrl={clearUrl} />);
    expect((await screen.findByRole("status")).textContent)
      .toBe("2 documentos assinados; 1 não assinado — fica em Pendentes de assinatura.");
    expect(screen.getByRole("list", { name: "não assinados" }).textContent).toBe("o prestador não respondeu");
    fireEvent.click(screen.getByRole("button", { name: "Continuar" }));
    expect(onDone).toHaveBeenCalledWith("signature-pending");
  });

  it("autorização negada no prestador: não chama o api e diz que nada mudou", async () => {
    wrap(<SignatureCallback params={{ state: "st1", code: null, error: "access_denied" }} onDone={onDone} clearUrl={clearUrl} />);
    expect((await screen.findByRole("alert")).textContent).toBe("a autorização foi negada no prestador — nada foi alterado");
    expect(api.completeSignatureOAuth).not.toHaveBeenCalled();
    expect(clearUrl).toHaveBeenCalledTimes(1);
    fireEvent.click(screen.getByRole("button", { name: "Voltar ao painel" }));
    expect(onDone).toHaveBeenCalledWith("overview");
  });

  it("retorno já usado (invalid_state): diz para começar de novo", async () => {
    mocked(api.completeSignatureOAuth).mockRejectedValue(new ApiError(422, { error: "invalid_state" }, "422"));
    wrap(<SignatureCallback params={OK} onDone={onDone} clearUrl={clearUrl} />);
    expect((await screen.findByRole("alert")).textContent)
      .toBe("este retorno do prestador não vale mais (já usado ou vencido) — comece de novo");
  });

  it("certificado de outro CPF: diz o motivo", async () => {
    mocked(api.completeSignatureOAuth).mockRejectedValue(new ApiError(422, { error: "certificate_cpf_mismatch" }, "422"));
    wrap(<SignatureCallback params={OK} onDone={onDone} clearUrl={clearUrl} />);
    expect((await screen.findByRole("alert")).textContent)
      .toBe("o certificado autorizado não é do seu CPF — escolha no prestador o certificado em seu nome");
  });

  it("sem state ou code: não chama o api", async () => {
    wrap(<SignatureCallback params={{ state: null, code: "c1", error: null }} onDone={onDone} clearUrl={clearUrl} />);
    expect((await screen.findByRole("alert")).textContent).toBe("o prestador não concluiu a autorização — comece de novo");
    await waitFor(() => expect(api.completeSignatureOAuth).not.toHaveBeenCalled());
  });
});
```

E o destino, no `describe("assinatura digital no menu (módulo 19b)")` de `src/shell/modules.test.ts` (acrescente `moduleFromPath` ao import):

```ts
  it("moduleFromPath: só /<id> de módulo do catálogo", () => {
    expect(moduleFromPath("/signature")).toBe("signature");
    expect(moduleFromPath("/signature-pending")).toBe("signature-pending");
    expect(moduleFromPath("/attendance")).toBe("attendance");
    expect(moduleFromPath("/attendance?x=1")).toBe("attendance");
    expect(moduleFromPath("/nao-existe")).toBeNull();
    expect(moduleFromPath("//evil.example")).toBeNull();
    expect(moduleFromPath("https://evil.example/signature")).toBeNull();
    expect(moduleFromPath(null)).toBeNull();
  });
```

- [ ] **Step 2: Run test to verify it fails**

Run: `cd apps/dashboard/.claude/mod19b && npx vitest run src/modules/SignatureCallback.test.tsx src/shell/modules.test.ts`
Expected: FAIL — `Failed to resolve import "./SignatureCallback"` e `moduleFromPath is not a function`.

- [ ] **Step 3: Write minimal implementation**

```tsx
// src/modules/SignatureCallback.tsx
// Retorno do prestador de assinatura (módulo 19b; spec §5; contrato §4). O
// prestador devolve a pessoa a /dashboard/signature/callback?state=&code=
// (ou error=). O `state` é de USO ÚNICO: a troca roda uma vez só — um ref
// sobrevive ao ciclo monta/desmonta/monta do StrictMode, como no
// useGrantEntry — e a URL é limpa antes do POST, para o `code` não ficar no
// histórico nem ser trocado de novo num recarregar.
import { useEffect, useRef, useState, type CSSProperties } from "react";
import { useQueryClient } from "@tanstack/react-query";
import { completeSignatureOAuth, type BatchResult } from "../lib/api";
import {
  CERTIFICATE_KEY, PENDING_KEY, SESSION_KEY, callbackLanding, callbackSummary, clearSignatureCallbackFromUrl,
  oauthErrorPhrase, reasonLabel, signatureError, type SignatureCallbackParams
} from "../lib/signature";
import { moduleFromPath, type ModuleId } from "../shell/modules";
import { buttonStyle } from "../components/formStyles";

type View =
  | { kind: "working" }
  | { kind: "done"; text: string; failed: BatchResult["failed"]; landing: ModuleId }
  | { kind: "failed"; text: string };

interface Props {
  params: SignatureCallbackParams;
  onDone(landing: ModuleId): void;
  clearUrl?(): void;
}

export function SignatureCallback({ params, onDone, clearUrl = clearSignatureCallbackFromUrl }: Props) {
  const queryClient = useQueryClient();
  const started = useRef(false);
  const [ view, setView ] = useState<View>({ kind: "working" });

  useEffect(() => {
    if (started.current) return;
    started.current = true;
    clearUrl();
    const { state, code, error } = params;
    if (error || !state || !code) {
      setView({ kind: "failed", text: oauthErrorPhrase(error) });
      return;
    }
    void (async () => {
      try {
        const result = await completeSignatureOAuth(state, code);
        for (const key of [ CERTIFICATE_KEY, SESSION_KEY, PENDING_KEY ]) void queryClient.invalidateQueries({ queryKey: [ key ] });
        setView({
          kind: "done",
          text: callbackSummary(result),
          failed: result.purpose === "batch" ? result.result.failed : [],
          landing: moduleFromPath(callbackLanding(result)) ?? "overview"
        });
      } catch (err) {
        setView({ kind: "failed", text: signatureError(err) });
      }
    })();
    // eslint-disable-next-line react-hooks/exhaustive-deps
  }, []);

  return (
    <div style={page}>
      <section aria-label="Retorno do prestador de assinatura" style={panel}>
        <strong>Assinatura digital</strong>
        {view.kind === "working" && <p style={text}>concluindo a autorização…</p>}
        {view.kind === "done" && (
          <>
            <p role="status" style={text}>{view.text}</p>
            {view.failed.length > 0 && (
              <ul aria-label="não assinados" style={list}>
                {view.failed.map((f) => <li key={f.request_id}>{reasonLabel(f.reason_code)}</li>)}
              </ul>
            )}
            <div><button type="button" style={buttonStyle} onClick={() => onDone(view.landing)}>Continuar</button></div>
          </>
        )}
        {view.kind === "failed" && (
          <>
            <p role="alert" style={alert}>{view.text}</p>
            <div><button type="button" style={buttonStyle} onClick={() => onDone("overview")}>Voltar ao painel</button></div>
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
const list: CSSProperties = { margin: 0, paddingLeft: 18, fontSize: 12.5 };
const text: CSSProperties = { margin: 0, fontSize: 13 };
const alert: CSSProperties = { margin: 0, fontSize: 13, color: "var(--down)" };
```

Em `src/shell/modules.ts` (sobre a Task 7), depois de `labelFor`:

```ts
// Módulo 19b: o `return_to` que o dashboard manda ao api é "/<id do módulo>"
// (a navegação é por estado, não por URL). Só caminho relativo de um módulo
// do catálogo vale; qualquer outra coisa é null (e quem chama cai na visão geral).
export function moduleFromPath(path: string | null | undefined): ModuleId | null {
  if (!path || !/^\/[^/]/.test(path)) return null;
  const id = path.slice(1).split(/[?#/]/)[0];
  return NAV_GROUPS.some((g) => g.items.some((i) => i.id === id)) ? (id as ModuleId) : null;
}
```

Em `src/App.tsx`:

```diff
--- a/src/App.tsx
+++ b/src/App.tsx
@@
-export function App() {
+// `initialModule`: a tela de onde a pessoa saiu para o prestador de
+// assinatura (módulo 19b), devolvida pelo retorno do OAuth.
+export function App({ initialModule }: { initialModule?: ModuleId } = {}) {
   const [ period, setPeriod ] = useState<PeriodKey>("7d");
-  const [ active, setActive ] = useState<ModuleId>("overview");
+  const [ active, setActive ] = useState<ModuleId>(initialModule ?? "overview");
```

Em `src/main.tsx`:

```diff
--- a/src/main.tsx
+++ b/src/main.tsx
@@
 import { AcceptInvitation } from "./modules/AcceptInvitation";
+import { SignatureCallback } from "./modules/SignatureCallback";
 import { AuthProvider, useAuth } from "./lib/auth";
 import { useSessionQueryClient } from "./lib/sessionQueryClient";
 import { clearEntryFromUrl, readEntryFromUrl, type Entry } from "./lib/entry";
 import { useGrantEntry } from "./lib/use_grant_entry";
+import { readSignatureCallback, type SignatureCallbackParams } from "./lib/signature";
+import type { ModuleId } from "./shell/modules";
 import "./theme/global.css";
 
 function AppRoot() {
   const auth = useAuth();
   const [ entry, setEntry ] = useState<Entry | null>(() => readEntryFromUrl());
+  // Módulo 19b: retorno do prestador de assinatura. Lido uma vez (puro); a
+  // tela de retorno limpa a URL e troca o código depois do login.
+  const [ signatureReturn, setSignatureReturn ] = useState<SignatureCallbackParams | null>(() => readSignatureCallback());
+  const [ landing, setLanding ] = useState<ModuleId | undefined>(undefined);
@@
   return (
     <QueryClientProvider client={queryClient}>
-      <App />
+      {signatureReturn
+        ? <SignatureCallback params={signatureReturn} onDone={(next) => { setSignatureReturn(null); setLanding(next); }} />
+        : <App initialModule={landing} />}
     </QueryClientProvider>
   );
```

(Anônimo no retorno — a sessão caiu enquanto estava no prestador — vê o `Login` de sempre; depois de entrar, o retorno segue, porque `signatureReturn` continua no estado e a URL só é limpa pela tela de retorno.)

- [ ] **Step 4: Run test to verify it passes**

Run: `cd apps/dashboard/.claude/mod19b && npx vitest run src/modules/SignatureCallback.test.tsx src/shell/modules.test.ts && npx tsc --noEmit`
Expected: PASS (8 testes no retorno; `modules.test.ts` verde); `tsc` limpo.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19b add src/modules/SignatureCallback.tsx src/modules/SignatureCallback.test.tsx src/shell/modules.ts src/shell/modules.test.ts src/main.tsx src/App.tsx
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19b commit -m "feat: handle the signature provider callback and return to the original screen

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 9: Suíte, build, revisão e prova no navegador

**Files:** nenhum novo.

**Interfaces:**
- Consumes: as Tasks 1–8 na branch `feat/mod-19b-signature`; o api do 19b rodando na **3037** com o PSC falso e o `signer` do compose.
- Produces: branch pronta para merge (depois do merge do api do 19b).

- [ ] **Step 1: Suíte, tipos e build**

Run: `cd apps/dashboard/.claude/mod19b && npx vitest run && npx tsc --noEmit && npm run build`
Expected: tudo verde: a base anotada na Task 0 mais **10 arquivos de teste novos** (`api.signature`, `signature`, `SignatureAccount`, `SignaturePending`, `SignatureSessionBadge`, `SignatureMarker`, `SignatureDetail`, `ConsultationView.signature`, `SignatureOverview`, `SignatureCallback`). Um número de arquivos que dobra é artefato de build descoberto pelo vitest: pare e reporte. A CI roda os mesmos três; `dist/` está no `.gitignore`.

```bash
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19b status --short
/opt/homebrew/bin/git -C apps/dashboard/.claude/mod19b log --stat origin/main..HEAD | grep -c node_modules
```

Expected: `status` vazio e a contagem `0`. Se o symlink aparecer, **não** o adicione.

- [ ] **Step 2: Nada vaza**

```bash
cd apps/dashboard/.claude/mod19b
grep -rn "console\." src/lib/signature.ts src/hooks/usePendingSignatures.ts src/modules/Signature*.tsx src/modules/signature src/shell/SignatureSessionBadge.tsx
grep -rn "localStorage\|sessionStorage" src/lib/signature.ts src/modules/Signature*.tsx src/modules/signature src/shell/SignatureSessionBadge.tsx
grep -n "state=\|code=\|reason=\|cpf=" src/lib/api.ts | grep -v "status=pending"
grep -rn "patient_display_name\|signer_name" src/lib/signature.ts | grep -i "download\|filename"
```

Expected: os quatro sem saída — nenhum `console`, nenhum armazenamento do navegador, nada de `state`, `code`, motivo ou CPF em URL, e o nome do arquivo baixado leva só o id.

- [ ] **Step 3: Revisão**

Um subagente revisor lê `origin/main..HEAD` do dashboard contra a spec (§3, §5, §7, §9, §10), o ADR 0032 e o contrato do 19b (§1–§7). Pontos de atenção:
- o retorno do prestador troca o código uma vez só, limpa a URL antes do POST e não depende de `localStorage`; erro do prestador não chama o api;
- vincular e desvincular só pelo `SensitiveAction` com step-up; abrir sessão e lote sem step-up (contrato §4–§5);
- nenhum token, `code_verifier`, CPF completo ou texto clínico fora do que a tela mostra; conteúdo assinado com `gcTime: 0`; arquivos baixados com o id no nome;
- itens de menu, selo e contador só para quem pode (`canSign`, `canSeeSignatureOverview`); nenhuma rota de assinatura chamada por quem não pode (os testes conferem com `not.toHaveBeenCalled`);
- o marcador não some quando a assinatura fica inválida/indeterminada nem quando o interruptor é desligado; sem o bloco (api antigo), a `ConsultationView` do 19a fica igual;
- 409 `not_pending`/`nothing_pending` relê a lista; 409 `certificate_not_linked` leva à Conta;
- interface sem redesign: só componentes existentes; nenhum componente comum mudou (`git diff origin/main --stat -- src/components` vazio).

- [ ] **Step 4: Prova no navegador (com o usuário)**

Depende do api do 19b rodando na porta **3037** (plano do api do 19b: servidor do worktree no container, PSC falso de dev, `signer` no compose, semente com Curitiba em `record_mode = record`, `clinical_record` e `digital_signature` ligados em dev, a médica da semente com CPF e um certificado de teste no PSC falso, e o `redirect_uri` do PSC falso apontando para `http://curitiba.localhost:5187/dashboard/signature/callback`). O Vite do worktree sobe num container da rede do compose, para alcançar `http://api:3037` (se o plano do api subir o servidor do 19b num container próprio, use o nome dele no lugar de `api`):

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose run -d --rm --no-deps --name dashboard-mod19b -p 5187:5187 \
  -w /app/.claude/mod19b -e VITE_API_PROXY_TARGET=http://api:3037 \
  dashboard npx vite --port 5187 --host 0.0.0.0
docker logs -f dashboard-mod19b   # espere o "ready"; Ctrl+C sai do log, o container segue
```

Abra `http://curitiba.localhost:5187/dashboard/`. O usuário faz o login; não digite senha nem TOTP (as da semente de dev podem ser mostradas se ele pedir). Confira com screenshot:
- como a médica, Conta → **Assinatura digital**: "Você continua assinando no papel…"; "Procurar meu certificado" → "Procurar" → "Vincular VIDaaS" → step-up → o PSC falso → volta a `/dashboard/signature/callback`, a URL fica `/dashboard/` e aparece "Certificado VIDaaS vinculado, válido até …" → "Continuar" volta à Conta com o certificado;
- no topo, "sem sessão de assinatura" → "abrir sessão" → PSC falso → "Sessão de assinatura aberta até …" → "Continuar" volta à tela de onde saiu; o selo diz "assinatura ativa até …";
- finalizar uma consulta (fluxo do 19a): o marcador "assinatura pendente · assinando…" e, depois de recarregar a consulta, "assinada digitalmente"; "Ver o que foi assinado" com o conteúdo, "Baixar PDF assinado" e "Baixar .p7s" (os arquivos `assinatura-<id>.pdf`/`.zip`), "Revalidar"; o aviso "o impresso é o PDF assinado digitalmente" e "Imprimir" abre o PDF assinado;
- "encerrar" a sessão, finalizar outra consulta → "assinatura pendente · sem sessão de assinatura aberta"; **Pendentes de assinatura (1)** no menu; "Assinar todas" → PSC falso → "1 documento assinado." → "Continuar" volta às Pendentes, vazias;
- outra pendente → "Voltar ao papel" com motivo curto (recusa na tela) e depois válido → some da lista; a consulta mostra "assinatura à mão (papel) · voltou ao papel a pedido do autor" e "Imprimir" sai com espaço para assinatura à mão;
- adendo numa consulta assinada (com sessão) → o adendo ganha o próprio marcador; o aviso do impresso muda para "com adendo…";
- como `admin@curitiba.demo` (municipal_admin), Equipe → **Painel de assinatura**: a médica com certificado, os pendentes, os documentos por modo;
- no maintenance, desligar `digital_signature` em Curitiba e recarregar o dashboard: os itens de menu e o selo somem; a consulta assinada continua mostrando "assinada digitalmente".

Depois: `docker rm -f dashboard-mod19b`. Religue o que a prova desligou (interruptores da cidade como estavam antes).

- [ ] **Step 5:** **Pare.** O merge do dashboard só vem depois do merge do api do 19b, e só com autorização explícita do usuário (merge, push e cards, uma etapa de cada vez). Ordem de deploy: api → dashboard → maintenance (contrato §12). Antes do push, confira `origin/main..main` no dashboard e publique só os commits desta entrega. Comandos, quando autorizados:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/dashboard
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

Formatos que este plano precisou e que o contrato do 19b não traz (ou traz de outro jeito). O plano já está escrito com a proposta; se uma for recusada, o ajuste fica dito.

1. **D1 — Rota de retorno sob a base do dashboard.** O contrato §4 diz que a rota de retorno é `/signature/callback?state=&code=`. O dashboard é servido em `/dashboard/` (Vite `base`), e `/signature` é o prefixo das rotas do api (proxy de dev; o mesmo vale para o roteamento de produção): o prestador devolveria a pessoa ao api, não à tela. Proposta: a rota é **`/dashboard/signature/callback`** e o `redirect_uri` cadastrado em cada PSC (e no PSC falso de dev) é `https://<host da cidade>/dashboard/signature/callback`. Recusada (rota na raiz), o roteamento precisaria de exceção para `GET /signature/callback` ir ao dashboard, e o proxy do Vite, de um `bypass`.
2. **D2 — `return_to` devolvido no callback.** O contrato recebe `return_to` em `link`, `sessions` e `batches`, mas a resposta de `POST /signature/oauth/callback` não o devolve, e o dashboard não tem como saber para onde voltar (a navegação é por estado). Proposta: `{ "purpose", "result", "return_to" }` — o api guarda o `return_to` com o `state` (`signature_oauth_states`) e o devolve na troca. Recusada, o dashboard usa o padrão do propósito (`link` → Conta, `batch` → Pendentes, `session` → visão geral; `callbackLanding`), e quem abriu a sessão no meio de um atendimento volta à visão geral.
3. **D3 — Forma do `return_to`.** O contrato só diz "caminho relativo do dashboard (`/^\/[^\/]/`)". Proposta: `"/<id do módulo>"` (ex.: `/signature`, `/signature-pending`, `/attendance`), sem a base e sem consulta; o api só confere o formato e devolve igual. O dashboard aceita só ids do catálogo (`moduleFromPath`), então um `return_to` adulterado cai na visão geral.
4. **D4 — `expires_in_days` e certificado vencido.** O contrato não diz o arredondamento nem o que vem depois do vencimento. Proposta: dias inteiros até `not_after` no fuso da cidade (0 = vence hoje), **negativo** depois do vencimento, e `status: "expired"` quando o api já marcou. O dashboard trata os dois como vencido (`certificateNotice`).
5. **D5 — `GET /signature/signatures/:id` (e `pdf`, `package`, `verify`) fora do atendimento.** O contrato diz "403 `out_of_context` como no 19a". Proposta: vale a mesma regra do `GET` da consulta no 19a — a abertura justificada ativa do usuário para o paciente basta (sem parâmetro), e 403 `opening_required` quando ela venceu; o dashboard então avisa a tela da abertura (`onOpeningRequired`).
6. **D6 — `patient_display_name` nulo.** O `<request>` traz `patient_display_name`; paciente validado sem nome (19a) não tem nome de exibição. Proposta: `string | null` (a tela mostra "—").

## Self-review

- **Cobertura (spec §9 e contrato §2–§7):**
  - Conta → Assinatura digital: vínculo por CPF (discover → link com step-up → prestador → retorno), troca (mesmo fluxo), desvínculo com step-up, aviso 30 dias antes, "você continua no papel" com a orientação CFM/COFEN — Tasks 2, 3 e 8;
  - selo da sessão no topo (abrir, encerrar, vencer com a tela aberta) — Tasks 2 e 5; retorno `/dashboard/signature/callback` com `POST /signature/oauth/callback` e volta ao `return_to` — Tasks 2 e 8;
  - marcador na consulta e no adendo; "Ver o que foi assinado", "Baixar PDF assinado", "Baixar .p7s", "Revalidar" — Tasks 2 e 6;
  - Pendentes de assinatura com "Assinar todas" (lote de 50) e "Voltar ao papel" (≥ 10) e o contador no menu — Tasks 2, 4 e 5;
  - impresso: o api escolhe (contrato §6); a tela avisa como ele sai — Tasks 2 e 6;
  - painel do admin (com/sem certificado, vencendo em 30 dias, pendentes > 24 h, documentos por modo, inválidas/indeterminadas), só leitura — Tasks 2 e 7;
  - interruptor `digital_signature` na sessão e proxy `/signature` — Task 1; spec §10 (LGPD na tela) — Global Constraints e Task 9; spec §11 (Vitest nas telas) — Tasks 1–8; prova manual — Task 9.
- **Placeholders:** nenhum. Arquivos novos vêm inteiros; os existentes, por diff com contexto do estado depois do 19a (a Task 0 manda conferir os trechos do 19a).
- **Consistência de nomes:** `CERTIFICATE_KEY`, `SESSION_KEY`, `PENDING_KEY`, `SIGNATURE_KEY`, `OVERVIEW_KEY`, `RETURN_TO`, `canSign`, `canSeeSignatureOverview`, `signatureError`, `goToProvider`, `saveBlob`, `signatureFileName`, `signatureMarker`, `printHint`, `readSignatureCallback`, `callbackLanding`, `callbackSummary` (Task 2) usados nas Tasks 3–8; `usePendingSignatures` (Task 4) na Task 5; `SignatureDetail` (Task 6) dentro do `SignatureMarker` (Task 6); `moduleFromPath`, `withPendingCount` (Tasks 8 e 5) em `modules.ts`; ids de módulo `signature`, `signature-pending`, `signature-overview` iguais em `modules.ts`, `App.tsx`, `RETURN_TO` e nos testes; fixtures `NOW19B`, `signer`, `certificate`, `signatureBlock`, `pendingRequest`, `signatureDetail`, `overview` (Task 1) nas Tasks 2–8.
- **Review Focus:** 1 — Task 8 ("StrictMode: troca o código uma vez só", "vínculo: limpa a URL antes de trocar o código…"); 2 — Task 5 ("a sessão vence com a tela aberta…"); 3 — Task 4 ("not_pending fecha o formulário…", "nothing_pending relê a lista…"); 4 — Task 2 ("aviso do certificado…"), Task 3 ("vence em 12 dias…") e Task 8 ("certificado de outro CPF…"); 5 — Task 6 ("voltou ao papel porque o interruptor foi desligado", "Revalidar devolve indeterminada…") e Tasks 3, 4 e 7 (menu).
