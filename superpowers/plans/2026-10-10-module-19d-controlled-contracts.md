# Módulo 19 (19d) — Contratos entre apps

**Spec:** `superpowers/specs/2026-10-10-module-19d-controlled-prescriptions-design.md` · **ADR:** `adr/0034.md`

Estende o contrato do 19c (`2026-10-09-module-19c-documents-contracts.md`, §12
prevalece). Pré-requisito de execução: **19c entregue em origin/main**. Mesmas
convenções: `{ "error": "<reason>" }` (com `field`/`index`), step-up 401
`mfa_required`, interruptor 403 `{ "error": "feature_disabled", "feature":
"controlled_prescriptions" }`, Content-Type JSON em toda escrita por cookie
(inclusive DELETE), leitura por `ClinicalRecord::Access`.

## 1. Valores comuns

- `category` (receita): `common` | `special_control` | `antimicrobial`.
- `sncr.kind`: `rce` | `ret`. Estado do número: `free` | `used` | `voided`.
- `notification_type`: `A` | `B` | `B2`.
- Lista Portaria 344 no item do catálogo: `controlled_list` ∈ `A1`,`A2`,`A3`,`B1`,`B2`,`C1`,`C2`,`C3`,`C5`, mais `anticonvulsant: bool`.

## 2. Receita (estende o §2/§3 do 19c)

`content` da `prescription` ganha:
```json
{ "category": "special_control"|"antimicrobial"|"common",
  "sncr": { "kind": "rce"|"ret", "number": "<string>", "simulated": bool }|null,
  "patient_identification": { "cpf": "<11 dígitos>"|null, "no_cpf": bool, "passport"?: "<string>",
                               "address": { "street", "number", "complement"?, "district", "city", "uf", "zip" } }|null,
  "prescriber_contact": { "address": {…}, "phone": "<string>" }|null }
```
- `category` é calculada pelo api (a entrada não a envia; se enviar, é ignorada).
- `special_control`: `patient_identification` (com `address`; `cpf` ou `no_cpf`) e `prescriber_contact` obrigatórios; `antimicrobial` digital: `prescriber_contact` obrigatório, `patient_identification` opcional.
- `sncr` presente só no modo `digital` de `special_control`/`antimicrobial`; `null` em papel.
- Validade impressa: `valid_until` = emissão + 30 dias (`special_control`) / + 10 dias (`antimicrobial`).
- Erros novos em `POST /attendance/consultations/:id/documents`: 422
  `mixed_categories`, `requires_notification` (com `index`), `not_supported`
  (C2/C3, com `index`), `too_many_c1_substances`, `duration_exceeded` (com
  `index`), `patient_identification_required` (com `field`),
  `prescriber_address_missing`; 403 `cbo_not_allowed` (enfermeiro em
  `special_control`).
- Resposta da emissão traz `issue_mode` e, quando caiu para papel, `paper_reason`:
  `no_certificate` | `signature_unavailable` | `no_sncr_number` | `nurse_antimicrobial` | `feature_disabled`.

## 3. Registro da Notificação

- `POST /attendance/consultations/:id/documents { kind: "controlled_notification_record", content }`
  com `content`: `{ notification_type, paper_number, numbering_uf, item: { catalog_item: { id }, quantity, quantity_unit, route, dosage_instructions, duration_days } }`
  → 201 `<document>` (sem `short_code` nem `verification_url`; `issue_mode: "paper"`;
  `signature: null`). 422 `item_not_in_notification_list`,
  `notification_type_mismatch` (item A com tipo B etc.), `duration_exceeded`
  (A ≤ 30; B/B2 ≤ 60), `invalid_content` (com `field`); 403 `cbo_not_allowed`.
- Cancelamento pelo `POST /attendance/documents/:id/cancel` do 19c.

## 4. Estoque SNCR (usuário corrente)

- `GET /sncr/stock` → `{ "rce": { "free", "used", "voided", "requests_this_month" }, "ret": { … }, "low_threshold": 50, "simulated": bool }`.
- `POST /sncr/requests { kind, return_to }` → `{ "authorize_url" }`; 409
  `monthly_limit_reached`; 403 `cbo_not_allowed`; 409 `professional_cpf_missing`.
- `POST /sncr/oauth/callback { state, code }` (ou `{ state, error }`) →
  `{ "kind", "received": n, "batch_id", "return_to" }`; 422 `invalid_state`;
  409 `authorization_expired`; 403 `authorization_denied`; 503
  `sncr_unavailable`, `sncr_exhausted`.
- Rota de retorno do dashboard: `/dashboard/sncr/callback`.

## 5. Perfil do profissional

- `GET/PUT /attendance/professional_profile/contact` → `{ address: {…}, phone }`
  (o próprio usuário); 422 `invalid_address` (com `field`), `invalid_phone`.

## 6. Admin (`municipal_admin`)

- `GET /sncr/admin/overview` → `{ "professionals": [ { user_id, name, rce_free, ret_free, low: bool, last_request_at } ], "simulated": bool }`.

## 7. Página pública (estende o §7 do 19c)

- Ganha `category` e `sncr: { kind, number }` quando houver; nunca medicamento ou CPF.

## 8. Maintenance (GraphQL, token humano)

- Interruptores `controlled_prescriptions` (missing `clinical_documents_disabled`) e
  `sncr_mock` (missing `controlled_prescriptions_disabled`; só fora de produção —
  em produção não aparece no catálogo e a mutation responde `unknown_feature`).
- `sncrStatus { configured: Boolean!, reachable: Boolean!, lastCheckAt: ISO8601DateTime, simulatedAvailable: Boolean! }`.

## 9. JSON canônico (`contracts`)

`clinical/clinical-document-v1.json` ganha, na receita, `category`, `sncr`,
`patient_identification` e `prescriber_contact` (§2) — MINOR, tag
`clinical-v1.2.0`; `controlled_notification_record` não é assinável (sem
canônico). Vetor JCS com uma RCE e uma RET.

## 10. Eventos (só ids)

| Nome | Payload |
|---|---|
| `sncr.numbers_requested` | `user_id, kind, count, simulated` |
| `sncr.number_used` | `document_id, kind` |
| `sncr.number_voided` | `document_id, kind` |
| `controlled_notification.recorded` | `document_id, notification_type` |
| `clinical_document.issued` / `.cancelled` | + `category` |

## 11. Ordem de entrega

0. Prova técnica (troca do token gov.br do SNCR no servidor dentro de 30 s).
1. `contracts` `clinical-v1.2.0`.
2. `api` (porta de dev sugerida 3039; migrações a partir de 20261010700001; `fake-sncr` no compose, porta 8092).
3. `dashboard` (Vite 5189). 4. `maintenance`.

## 12. Acréscimos da escrita dos planos (2026-10-10)

Valem sobre as seções acima e sobre os planos.

**SNCR (manuais oficiais: Manual da API 3ª ed., Instruções de Integração v1.0)**
- A Anvisa faz o OAuth com o gov.br no servidor dela e devolve ao `client_url`
  só `?session_id` (uso único, 30 s, preso ao `Origin` de quem iniciou). Sem
  `code`, sem PKCE do nosso lado, sem `client_id`/`client_secret`.
- Callback: `POST /sncr/oauth/callback { state, session_id }` (ou `{ state, error }`).
  `POST /sncr/requests` devolve `{ authorize_url, state }`; o dashboard guarda o
  `state` em `sessionStorage` (ou no `client_url`, se a prova técnica permitir).
- Credenciais `sncr.{base_url, auth_url, maintainer_cnpj}`.
- **Task 0 do api = sonda real na homologação**, com o usuário fazendo o login
  gov.br. Se a troca do `session_id` pelo servidor dentro de 30 s não for
  confirmada (ou sem acesso à homologação), a execução para e o usuário escolhe:
  seguir com o SNCR simulado (troca real vira gate) ou o plano B (navegador chama
  o SNCR).
- A mensagem real de "faixa esgotada" não é documentada; o cliente a reconhece
  por `/esgot/` (gate de go-live).
- Formato do número: `^\d{4}\.\d-\d{2}\.\d{7}$` (`AAMM.T-UF.NNNNNNN`).

**Decididos pelo usuário (2026-10-10)**
- Tag `clinical-v1.2.0` é MINOR da v1; o README do contracts passa a dizer que
  campo opcional compatível é MINOR e só mudança incompatível pede v2.
- **Sem "não possui CPF":** `patient_identification = { cpf, address }`; a
  entrada envia só `{ address }` e o api preenche o `cpf` com o do cadastro (que
  existe por regra do 19a). Sem `no_cpf` e sem `passport`.

**Formas e erros**
- Formatos (entrada já normalizada, sem máscara): CEP `^[0-9]{8}$`, UF
  `^[A-Z]{2}$`, telefone `^[0-9]{10,11}$`; fora disso 422 com `field`.
- Rota `GET /attendance/consultations/:id/patient_identification` (reaproveitar
  a última identificação).
- `paper_reason` **gravado** no documento (coluna, só no papel, imutável) e
  devolvido em toda leitura.
- Pedido de números: 403 `registration_mismatch`; sem perfil ou conselho fora de
  CRM/CRO → 403 `cbo_not_allowed` (também em `GET /sncr/stock`); sai o 409
  `professional_cpf_missing`; `kind` inválido → 422 `invalid_content`.
- Lista C4 → `not_supported` (como C2/C3); controlado em texto livre segue recusado.
- `duration_days` obrigatório em todo item da RCE. RET sem contato do prescritor só é recusada quando sairia digital.
- Registro da Notificação: `short_code` e `verification_url` nulos; 404 no impresso e na página pública.
- Volta ao papel também anula o número (`voided`).
- Título do PDF: "RECEITA DE CONTROLE ESPECIAL" (norma da Anvisa).
- Contato: `PUT` devolve `{ address, phone }`; sem endereço → `{ address: null, phone }`; telefone é o `professionals.phone` do módulo 10; 404 sem perfil.
- Saldo e painel são do modo corrente (simulado ou real); `received` conta só números novos; `low` = menos de 50 livres em um dos tipos.
- `controlled_list` e `anticonvulsant` em todo `catalog_item` HTTP (busca, REMUME, renovação, itens). `clinical_document.cancelled` leva `category`.
- Página pública: `sncr.simulated`, aviso de "não dispense" na cancelada, `category` em toda receita.
- Nenhuma escrita do 19d pede step-up.

**JSON canônico (`clinical-v1.2.0`)**
- Receita comum sem `category` e sem chaves nulas (vetores da 1.1.0 seguem byte a
  byte); `sncr`, `patient_identification` e `complement` ausentes quando não se
  aplicam.
- Vetores novos: `prescription-special-control.jcs` (1617 bytes, SHA-256
  `25538e7f…2e0a`) e `prescription-ret.jcs` (1390 bytes, `e4397d40…0464`), no
  `SHA256SUMS` existente; manifesto de 51 para 82 casos.

**Limitação conhecida**
- Dentista não emite no 19c nem no 19d enquanto a consulta do 19a recusar o CBO
  2232 (odontologia é o módulo 28).

**Decidido pelo usuário durante a execução (2026-10-10)**
- A sonda real na homologação do SNCR (Task 0 do api) **não foi feita**. O
  usuário escolheu seguir com o SNCR simulado: a troca do `session_id` pelo
  servidor dentro de 30 s vira **gate de go-live**, junto da prova com o SNCR
  real. Risco registrado: se a troca no servidor não funcionar (o `session_id`
  é preso ao `Origin` do navegador), o plano B (navegador chama o SNCR e repassa
  os números ao api) obriga a refazer `Sncr::Client`, o callback e a tela de
  Conta → SNCR. Registro em `pesquisa/2026-10-10-sncr-token-no-servidor.md`.
