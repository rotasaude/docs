# Módulo 19 (19b) — Contratos entre apps

**Spec:** `superpowers/specs/2026-10-08-module-19b-digital-signature-design.md` · **ADR:** `adr/0032.md`

Fonte única dos formatos entre `api`, `dashboard`, `maintenance` e `signer`.
Erros `{ "error": "<reason>" }`; step-up 401 `{ "error": "mfa_required" }`;
interruptor desligado 403 `{ "error": "feature_disabled", "feature": "digital_signature" }`;
escrita devolve o objeto puro. Executa depois do 19a fechado: as formas
`<consultation>` e `<record>` são as do contrato do 19a (com o §9 dele).

## 1. Valores comuns

- `provider`: `vidaas` | `birdid` | `safeid` | `neoid` | `remoteid`.
- Estado do pedido: `pending` | `signed` | `failed` | `returned_to_paper`.
- Modo do documento: `digital` | `manual` | `pending`.
- Validação: `valid` | `invalid` | `indeterminate`.
- Motivos (`reason_code`): `no_session`, `session_expired`, `provider_unavailable`,
  `provider_rejected`, `signer_unavailable`, `verification_failed`,
  `certificate_expired`, `certificate_revoked`, `feature_disabled` (este só em
  `returned_to_paper`), `user_request` (volta ao papel pelo autor).
- `document_type`: `consultation` | `consultation_addendum`.

## 2. Bloco `signature` na consulta e no adendo

`<consultation>` e cada item de `addenda` ganham:
```json
"signature": { "mode": "digital"|"manual"|"pending",
               "request_id"?, "signature_id"?, "signed_at"?, "signer_name"?,
               "verification"?: "valid"|"invalid"|"indeterminate",
               "reason_code"? }
```
`manual` quando o interruptor está desligado, o autor não tinha certificado ativo
na finalização, ou o pedido voltou ao papel. Rascunho não tem o bloco.

## 3. Certificado (`/signature/certificates`, usuário corrente)

Forma `<certificate>`: `{ "id", "provider", "issuer", "serial_number", "not_after", "status", "expires_in_days" }`.
- `GET /signature/certificates/current` → `<certificate>` ou 404 `certificate_not_linked`.
- `POST /signature/certificates/discover` → `{ "providers": [ { "provider", "found": bool } ], "unavailable": [ "<provider>" ] }`.
- `POST /signature/certificates/link { provider, return_to }` (step-up) → `{ "authorize_url" }`; 422 `invalid_provider`.
- `DELETE /signature/certificates/current` (step-up) → 204.

## 4. OAuth e sessão

- `POST /signature/sessions { return_to }` → `{ "authorize_url" }`; 409 `certificate_not_linked`.
- `GET /signature/sessions/current` → `{ "active": bool, "expires_at"?, "provider"? }`.
- `DELETE /signature/sessions/current` → 204.
- `POST /signature/oauth/callback { state, code }` → `{ "purpose": "link"|"session"|"batch", "result": <certificate> | { "expires_at" } | <batch> }`;
  422 `invalid_state`; 409 `authorization_expired`; 403 `authorization_denied`;
  422 `certificate_cpf_mismatch`, `certificate_expired`, `certificate_revoked`;
  503 `provider_unavailable`.
- `return_to` é caminho relativo do dashboard (`/^\/[^\/\\]/`, recusa `//` e `/\` — redirecionamento aberto); a rota de retorno
  do dashboard é `/signature/callback?state=&code=` (ou `error=`).

## 5. Pendentes e lote

Forma `<request>`: `{ "id", "document_type", "document_id", "consultation_id", "patient_display_name", "finalized_at", "status", "reason_code"?, "attempts" }`.
- `GET /signature/requests?status=pending` → `{ "items": [<request>] }` (só do autor, até 200, mais antigas primeiro).
- `POST /signature/requests/:id/return_to_paper { reason }` → `<request>`; 422 `invalid_reason` (< 10); 403 `not_author`; 409 `not_pending`.
- `POST /signature/batches { request_ids?, return_to }` (sem ids = todas pendentes, até 50) → `{ "authorize_url", "count" }`; 409 `nothing_pending`, `certificate_not_linked`.
- Resultado do lote (no callback): `<batch>` = `{ "signed": n, "failed": [ { "request_id", "reason_code" } ] }`.

## 6. Assinatura

- `GET /signature/signatures/:id` → `{ "id", "document_type", "document_id", "signed_at", "signer_name", "signer_cpf_masked", "policy": "AD-RB", "verification", "verification_reasons": [], "verified_at", "content": <JSON canônico legível> }`;
  trilha `clinical_record.viewed`; fora de contexto 403 `opening_required` (ver §13).
- `GET /signature/signatures/:id/pdf` → `application/pdf` (PAdES).
- `GET /signature/signatures/:id/package` → `application/zip` (`document.json` + `document.json.p7s`).
- `POST /signature/signatures/:id/verify` → mesma forma do GET, revalidada.
- Impresso do 19a: `GET /attendance/consultations/:id/print` devolve o PDF
  assinado quando o modo é `digital` (consulta sem adendo assinado depois) e o
  impresso do 19a nos demais casos.

## 7. Admin (`municipal_admin`)

`GET /signature/admin/overview?from=&to=` →
```json
{ "professionals": [ { "user_id", "name", "certificate_status": "active"|"none"|"expiring", "not_after"?, "pending_count", "oldest_pending_at"? } ],
  "documents_by_mode": { "digital", "manual", "pending" },
  "invalid_or_indeterminate": [ { "signature_id", "document_type", "signer_name", "verification", "verified_at" } ] }
```
`expiring` = vence em ≤ 30 dias; período no fuso da cidade, padrão 30 dias.

## 8. Maintenance

- Interruptor `digital_signature` pelo mecanismo genérico (`missing`:
  `clinical_record_disabled`).
- Leitura nova na API GraphQL de manutenção: `signatureProviders { key configured lastCheckAt lastCheckOk }`
  e `signerStatus { reachable version crlUpdatedAt }` (plataforma, sem segredo).

## 9. Serviço `signer` (interno; api/worker → signer)

`Authorization: Bearer <SIGNER_TOKEN>`; JSON; corpo nunca logado.
- `POST /prepare { kind: "cades"|"pades", document_base64, certificate_der_base64, policy: "AD-RB" }`
  → `{ "to_be_signed_sha256_base64", "prepared_state" }` (estado opaco, base64).
- `POST /assemble { kind, prepared_state, signature_value_base64 }` →
  `{ "signature_base64", "validation_material_base64" }` (p7s destacado ou PDF assinado).
- `POST /verify { kind, document_base64?, signature_base64 }` →
  `{ "status": "valid"|"invalid"|"indeterminate", "signer_cpf", "signer_name", "policy_oid", "signed_at", "reasons": [] }`.
- `GET /health` → `{ "version", "crl_updated_at" }`.
- Erros: 400 `invalid_request`, 401 `unauthorized`, 422 `invalid_certificate`,
  `invalid_signature_value`, 500 `internal`.

## 10. JSON canônico (`contracts`)

`schemas/clinical/consultation-v1.json` e `consultation-addendum-v1.json`
(JSON Schema 2020-12), serializados em RFC 8785. Campos: `schema`, `city`
(`ibge_code`, `name`), `unit` (`cnes`, `name`), `professional` (`name`, `cpf`,
`cbo_code`, `council`), `patient` (`display_name`, `cpf`, `birth_date`),
`consultation` (`id`, `started_at`, `finalized_at`, `care_type`, S/O/A/P,
`vitals`, `evaluated_problems` com `terminology`/`code`/`release`/`action`,
`conducts`, `exam_requests`, `outcome`); adendo: `consultation_id`, `id`,
`created_at`, `reason`, `text`, `changes`, `previous_sha256`. Tag
`clinical-v1.0.0`.

## 11. Eventos (só ids)

| Nome | Payload |
|---|---|
| `signature.certificate_linked` / `signature.certificate_unlinked` | `certificate_id, user_id, provider` |
| `signature.session_opened` | `session_id, user_id, provider` |
| `signature.signed` | `signature_id, request_id, document_type, document_id` |
| `signature.failed` | `request_id, reason_code` |
| `signature.returned_to_paper` | `request_id, reason_code` |
| `signature.verified` | `signature_id, verification` |

## 12. Ordem de entrega

0. Prova técnica (Task 0 do plano do `signer`).
1. `contracts` `clinical-v1.0.0`.
2. `signer` (repositório novo, com autorização).
3. `api` (porta de dev sugerida 3037; migrações de cidade acima das do 19a).
4. `dashboard`. 5. `maintenance`.

## 13. Acréscimos da escrita dos planos (2026-10-08)

Valem sobre as seções acima e sobre os planos.

**contracts**
- Caminhos: `clinical/consultation-v1.json`, `clinical/consultation-addendum-v1.json`
  e vetor de canonicalização em `clinical/examples/canonical/` (o api prova o
  gerador RFC 8785 byte a byte contra ele). Não há prefixo `schemas/`.
- O adendo leva o mesmo cabeçalho da consulta (profissional = autor do adendo)
  e o conteúdo num objeto `addendum`. `label` de problemas e exames entram no
  JSON. `council { name, state, registration_number }`;
  `outcome { code: discharged|referred|return, referral_unit_cnes?, referral_note? }`;
  texto vazio = `null`; horários UTC com `Z`; CPF só dígitos; `ibge_code`,
  `cnes` e `birth_date` anuláveis.
- `previous_sha256` = hash canônico do documento **assinado** anterior da mesma
  consulta (consulta ou adendo); sem nenhum assinado, o hash canônico da consulta.

**signer**
- `/verify`: `document_base64` obrigatório em CAdES e ignorado em PAdES;
  `reasons` de vocabulário fechado (plano do signer); revogado depois do
  `signingTime` = `indeterminate`, antes = `invalid`.
- `validation_material` = CMS `.p7c` com a cadeia e as LCRs; `crl_updated_at` =
  último download bem-sucedido de LCR.
- `/health` também exige o token; rota desconhecida 404 `not_found`; corpo > 48 MB
  400; certificado não-RSA 422; PDF ilegível 400; `/dev-pki/*` só em dev.
- PAdES invisível: o rodapé NGS2 é desenhado pelo api no PDF antes do `/prepare`.
- Implementação: Demoiselle 4.6.2 `CAdESSigner#prepareSignedAttributes` +
  BouncyCastle para o `SignerInfo` com o RAW do PSC; PDFBox para o PAdES.
  Políticas CAdES AD-RB v2.4 (`2.16.76.1.7.1.1.2.4`) e PAdES AD-RB v1.3
  (`2.16.76.1.7.1.11.1.3`). Log do Demoiselle (CPF/nome do titular) desligado.
- Env de dev do api: `SIGNER_URL`, `SIGNER_TOKEN`, `SIGNER_DEV_PKI_DIR=/signer-dev-pki`
  (volume `signer-dev-pki`, e-CPF de teste no formato `DevPki`).

**api**
- Interruptor: pré-requisito genérico `feature:<key>` → `<key>_disabled`.
- Callback aceita `{ state, error }` → 403 `authorization_denied` (consome o
  state) e devolve `return_to` (guardado com o state) em todos os casos.
- Rota de retorno do dashboard: `/dashboard/signature/callback`; a
  `redirect_uri` de cada pedido é `https://<host do dashboard da cidade>/dashboard/signature/callback`.
  **Decidido pelo usuário (2026-10-08): um endereço por cidade** (sem retorno
  único no `auth.*`); registrar o endereço de cada cidade em cada PSC é passo de
  go-live por cidade. `return_to` = `/<id do módulo>` do dashboard.
- `reason_code` ganha `certificate_cpf_mismatch`. `failed` fica reservado (não
  é produzido nesta entrega).
- 409 `professional_cpf_missing` em discover, link, sessions e batches; 422
  `invalid_provider` em sessions e batches quando o PSC do certificado deixou de
  estar configurado.
- Leitura de assinatura guardada é regida por `clinical_record` (não por
  `digital_signature`): fora de contexto 403 `opening_required`; id inexistente
  404 `not_found`.
- Lote: falha do PSC/`signer` durante a assinatura vira item em `failed` com 200;
  só a troca do código falhada dá 503; pedido travado pelo job fica fora do
  resultado. `GET /signature/requests` lista sempre as pendentes;
  `return_to_paper` com id inexistente 404.
- Overview: 403 `missing_role`, 422 `invalid_period`, padrão 30 dias.
- `signer_name` do bloco = nome cadastrado do autor; o nome do certificado só no
  rodapé do PDF e no conteúdo.
- `expires_in_days` = dias inteiros no fuso da cidade, negativo depois do
  vencimento; `patient_display_name` pode ser nulo.
- **Decidido pelo usuário (2026-10-08):** o vínculo confere CPF, validade, uso
  da chave **e revogação**, pela rota nova do `signer`
  `POST /certificates/check { certificate_der_base64 }` →
  `{ "status": "valid"|"invalid"|"indeterminate", "signer_cpf", "not_after", "reasons": [] }`
  (mesmos erros do §9; cadeia desconhecida = 200 `invalid` com `untrusted_chain`;
  LCR indisponível = `indeterminate` com `revocation_unavailable`). No vínculo:
  revogado/vencido → 422 `certificate_revoked`/`certificate_expired`;
  `untrusted_chain` → 422 `certificate_untrusted`; `indeterminate` aceita e a
  revogação volta a ser conferida na primeira assinatura.
- **Decidido pelo usuário (2026-10-08):** lote assina em ordem cronológica; dois
  adendos da mesma consulta no mesmo lote formam cadeia (o segundo leva o hash
  canônico do primeiro); se o primeiro falhar, o segundo fica `pending` com o
  mesmo motivo.

**maintenance**
- Na raiz `Query`: `signatureProviders: [SignatureProvider!]!`
  (`key: String!, configured: Boolean!, lastCheckAt: ISO8601DateTime, lastCheckOk: Boolean`)
  e `signerStatus: SignerStatus!` (`reachable: Boolean!, version: String, crlUpdatedAt: ISO8601DateTime`),
  sem levantar erro. "Última checagem" = última chamada real do api ao PSC.
- Rótulos `clinical_record_disabled` e `record_mode_not_record` entram no app.

**Ajuste vindo do 19a (2026-10-09; ADR 0031, Revisão)**
- A leitura do conteúdo assinado, do PDF e do pacote segue as regras novas do
  19a: a **autora** sempre (trilha `author`); o `municipal_admin` só lê o
  conteúdo (sem PDF nem pacote), com step-up e trilha `administrative`; outro
  profissional em contexto ou por abertura (`opening_required` fora disso).
  O plano do api do 19b deve usar o `ClinicalRecord::Access` revisado pela
  Task 21 do api do 19a, não a regra antiga.
- Adendo é só da autora (sem `opening_id` de terceiro): o assinante do adendo é
  sempre a autora da consulta.

**Decisões do usuário durante a execução (2026-10-09)**
- **PSC simulado primeiro.** O 19b é implementado e verificado inteiro contra o
  PSC simulado (API v0 do ITI) e a DevPki do `signer`. A Task 0 do plano do
  `signer` (sandbox VIDaaS → validar.iti.gov.br) deixa de ser o início e vira
  **gate de go-live** (rotasaude/api#53, Crítica). Enquanto o gate não passar,
  o 19b termina como `Entregue`, não `Fechado`; o critério "assinatura real
  aprovada no validar.iti.gov.br" passa para o gate.
- Autorizados: criar o repositório `rotasaude/signer` já; editar o
  `docker-compose.yml` da raiz (`signer` e `fake-psc`).
- **Escopo novo — interruptor `signature_psc_mock`** (cidade, mecanismo
  genérico, escrito só pelo maintenance; `requires: ["feature:digital_signature"]`,
  missing `digital_signature_disabled`):
  - só existe fora de produção: em produção o catálogo não o oferece e o api
    recusa ligar;
  - ligado: a cidade usa só o PSC simulado; desligado: só os PSC reais
    configurados no ambiente;
  - `provider` ganha o valor `simulated`; toda tela e exportação que mostra a
    assinatura diz "simulada — sem validade jurídica";
  - aviso visível no maintenance (aba Funcionalidades), no dashboard (telas de
    assinatura e selo) e na tela do PSC simulado ("PSC SIMULADO — desenvolvimento");
  - a semente de dev liga o interruptor em Curitiba;
  - substitui o `config.x.signature_dev_providers` do plano do api (o falso não
    responde mais como `vidaas` em development).
- Ordem: contracts-repo (tag `clinical-v1.0.0`) → `signer` sem a Task 0 → api →
  dashboard → maintenance.
- **JSON canônico (decidido pelo usuário, 2026-10-09, antes da tag
  `clinical-v1.0.0`):** cada item de `exam_requests` (consulta) e de
  `changes.exam_requests` (adendo) leva `competence` (AAAAMM, obrigatório),
  vindo de `consultation_exam_requests.sigtap_competence`.
- Construtor do api: lê `addendum.item_changes` (a coluna do 19a); emite
  `changes.exam_requests` quando a lista mudou, inclusive vazia (`[]` = todos
  cancelados); omite `bmi` quando nulo; `care_type`, condutas e `height_cm` como
  inteiros.
- **`signer` entregue (revisão final, head 8b5410b):** `SIGNER_ENV` obrigatório
  (`development` | `staging` | `production`; `prod` recusado); em produção
  também recusa `-Dsigner.extraTrustAnchors`. `/prepare`: LCR indisponível →
  500 `internal` (transitório — o api trata como `signer_unavailable` e tenta de
  novo); certificado revogado → 422; PDF cujo estado passaria de 32 MiB → 400.
  A conferência de revogação autentica a LCR e cobre cada elo da cadeia
  (falha → `indeterminate` com `revocation_unavailable`).
- **Bloco e leitura da assinatura (Task 14 do api):** o bloco `signature` no modo
  `digital` e as respostas de `GET`/`POST .../verify` de `/signature/signatures/:id`
  trazem sempre `simulated: true|false` e `provider`. `POST .../verify` exige
  `Content-Type: application/json` (corpo `{}`). O `municipal_admin` só lê o
  conteúdo; fora de contexto → 403 `opening_required`.
- **api do 19b concluído (7beafed, revisão final):** `simulated` sempre booleano
  e `provider` nos payloads do §6; `expires_in_days` em dias de calendário,
  no máximo -1 depois de vencer, também no painel do admin; `return_to` recusa
  barra invertida; volta ao papel → 409 `not_pending` se a assinatura estiver em
  andamento há até 5 s; lote com certificado de outro PSC → 403
  `authorization_denied`; logo depois de finalizar, o bloco mostra `mode:
  "manual"` até o job criar o pedido (o dashboard relê).
