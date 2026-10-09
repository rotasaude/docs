# Módulo 19 (19a) — Contratos entre apps

**Spec:** `superpowers/specs/2026-10-07-module-19-consultation-design.md` · **ADR:** `adr/0031.md`

Fonte única dos formatos entre `api` e `dashboard` (o `maintenance` só ganha a
chave `clinical_record` pelo mecanismo genérico do ADR 0028). Erros
`{ "error": "<reason>" }`; step-up 401 `{ "error": "mfa_required" }`;
interruptor desligado 403 `{ "error": "feature_disabled", "feature": "clinical_record" }`;
escrita devolve o objeto puro.

## 1. Valores comuns

- Problema: `{ "id", "terminology": "ciap2"|"cid10", "code", "label", "status": "active"|"resolved", "onset_on"|null, "onset_precision": "day"|"month"|"year"|null, "resolved_on"|null }`.
- Sinais vitais: mesma forma `vitals` do contrato do módulo 18.
- Condutas e tipos de atendimento: códigos fixados pela Task 1 do plano do api
  (`config/ledi/consultation_mapping.yml`); o `api` expõe a lista em §4.
- Nome de exibição: `display_name` = nome social se houver, senão nome completo.

## 2. Validação presencial (existente, muda)

`POST /attendance/verifications` aceita `full_name` (obrigatório, 3–200),
`social_name` (opcional, ≤ 200), `mother_name` (opcional, ≤ 200). 422
`invalid_full_name`, `invalid_social_name`, `invalid_mother_name`.
`POST /attendance/lookup` devolve `citizen.names: { full_name_set: bool,
display_name }` (nunca os três valores para quem só faz balcão sem validar).
`POST /attendance/verifications/:id/names` (completar nomes de par já validado,
papel `citizen_verifier`) `{ full_name, social_name?, mother_name? }` → o par.

## 3. Paciente e prontuário

Forma do prontuário (`record`):
```json
{ "patient": { "id", "display_name", "full_name", "social_name"|null, "age", "sex", "cpf_masked" },
  "access": "in_context"|"justified",
  "problems": [ <problema> ],
  "today_screening": <screening do módulo 18>|null,
  "consultations": [ { "id", "finalized_at", "author_name", "cbo_label", "care_type_label",
                       "problems": [ <problema avaliado> ], "addenda_count" } ] }
```
- `GET /attendance/attendances/:id/record` → `<record>` em contexto; 403
  `out_of_context`, `missing_role`; 409 `citizen_not_verified`.
- `GET /clinical_record/patients/:id` → `<record>` só com abertura válida; 403
  `opening_required`.
- `POST /clinical_record/openings` (step-up) `{ cpf, reason_code, reason_note? }`
  → `{ "opening_id", "patient_id", "expires_at" }`; 404 `patient_not_found`; 422
  `invalid_reason`.
- `GET /clinical_record/openings?from=&to=&user_id=` (`municipal_admin`) →
  `{ "items": [ { "id", "user_name", "cpf_masked", "reason_code", "created_at", "expires_at" } ] }`.

## 4. Consulta

Forma (`consultation`):
```json
{ "id", "attendance_id", "patient_id", "status": "draft"|"finalized",
  "author": { "id", "name" }, "cbo_code",
  "subjective", "objective", "assessment", "plan", "vitals": {...},
  "care_type", "evaluated_problems": [ { "problem_id"|null, "terminology", "code", "label",
                                         "action": "evaluate"|"add"|"resolve"|"correct_onset",
                                         "onset_on"?, "onset_precision"? } ],
  "conducts": [ "<código>" ], "exam_requests": [ { "sigtap_code", "label", "cid10_justification"? } ],
  "started_at", "finalized_at"|null,
  "addenda": [ { "id", "author_name", "created_at", "reason", "text", "changes": {...} } ] }
```
- `GET /attendance/consultation_options` → `{ "care_types": [{code,label}], "conducts": [{code,label}], "cid10_allowed_for_cbo": bool }` (para o usuário corrente).
- `POST /attendance/attendances/:id/consultation` → 201 `<consultation>` (`draft`);
  409 `citizen_not_verified`, `not_in_care`, `not_caller`, `already_exists`; 403
  `cbo_not_allowed`, `feature_disabled`.
- `PATCH /attendance/consultations/:id` (autosave; só o autor; só `draft`) → 200
  `<consultation>`; 409 `not_draft`; 403 `not_author`; 422 `implausible_vital`
  (com `field`), `text_too_long` (com `field`; 20.000 por campo).
- `POST /attendance/consultations/:id/finalize` `{ outcome: <corpo do close existente> }`
  → 200 `<consultation>`; 422 `no_problem_evaluated`, `no_conduct`,
  `assessment_or_plan_required`, `patient_name_missing`,
  `cid10_not_allowed_for_cbo`, mais os erros do close existente.
- `GET /attendance/consultations/:id` → `<consultation>` (rascunho só para o autor;
  finalizada em contexto ou abertura); trilha `clinical_record.viewed`.
- `POST /attendance/consultations/:id/addenda` `{ reason, text, changes?, opening_id? }`
  → 201 com o adendo; 409 `not_finalized`; 403 `opening_required`; 422
  `invalid_reason`.
- `GET /attendance/consultations/:id/print` → `application/pdf`; 409
  `not_finalized`, `patient_name_missing`.

## 5. Busca de terminologia

`POST /attendance/ciap2/search` (módulo 18) passa a aceitar
`{ q, terminology: "ciap2"|"cid10" }` (padrão `ciap2`); mesma resposta.
`POST /attendance/sigtap/search { q }` → `{ items: [{ code, label }] }` (competência
ativa); 503 `terminology_unavailable`.

## 6. Produção (módulo 16/18)

Ficha `correction_pending` aparece em `GET /production` com
`status: "correction_pending"` e não conta como pendente de envio.

## 7. Eventos (só ids)

| Nome | Payload |
|---|---|
| `patient.created` / `patient.linked` | `patient_id, citizen_id` |
| `patient_problem.changed` | `patient_problem_id, kind, consultation_id|addendum_id` |
| `consultation.started` / `consultation.finalized` | `consultation_id, attendance_id` |
| `consultation.addendum_added` | `consultation_id, addendum_id` |
| `clinical_record.viewed` | `patient_id, user_id, access, reason_code` |
| `clinical_record.opened` | `opening_id, patient_id, user_id, reason_code` |

## 8. Ordem de entrega

1. `api` (porta de dev sugerida 3036).
2. `dashboard` depois do `api`.
3. Interruptor `clinical_record` ligado pelo maintenance só em dev/staging.

## 9. Acréscimos da escrita dos planos (2026-10-07)

Valem sobre as seções acima e sobre os planos.

**Do plano do dashboard**
- `POST /attendance/attendances/:id/consultation` 409 `already_exists` traz `consultation_id`.
- Completar nomes de par já validado acontece no check-in: o `:id` de
  `POST /attendance/verifications/:id/names` é a validação ativa;
  `POST /attendance/check_ins/lookup` devolve `citizen.names`
  (`{ full_name_set, display_name }`) e `citizen.verification_id`;
  `POST /attendance/check_ins` devolve `verification_id` quando valida. A
  resposta de `.../names` é `{ citizen: { id, cpf_masked, verification_level, names } }`.
- Cada item de `GET /attendance/units/:id/queue` ganha `display_name`.
- Adendo `changes`: `evaluated_problems` = eventos novos; `conducts` e
  `exam_requests` = listas finais, só quando mudaram; `opening_id` aceito também
  da autora.
- `PATCH /attendance/consultations/:id` recebe todos os campos editáveis a cada
  salvamento.
- Compatibilidade de deploy: `POST /attendance/verifications` sem a chave
  `full_name` (cliente antigo) é aceito e não grava nome; com a chave, o nome é
  obrigatório (3–200). A consulta bloqueia com `patient_name_missing` até completar.
- `<record>.consultations` só lista finalizadas; `cid10_justification` segue a
  mesma regra de CBO dos problemas.
- Impresso: `application/pdf` inline no sucesso; `{ error }` em JSON nas recusas.

**Do plano do api**
- `missing` do interruptor ganha `record_mode_not_record`.
- 422 a mais (com `index` nos itens): `invalid_text` (com `field`),
  `invalid_problem`, `invalid_onset`, `cid10_sex_incompatible`, `invalid_conduct`,
  `invalid_exam`, `invalid_care_type`, `text_required`, `invalid_changes`,
  `invalid_period`, `invalid_terminology`. Pressão incompleta no autosave =
  `implausible_vital` com `field` do lado que falta.
- `POST /attendance/attendances/:id/close` → 409 `consultation_in_progress` quando
  há consulta em rascunho no atendimento.
- Par validado sem paciente ainda: `patient.id: null`, listas vazias, sem trilha.
- Consulta finalizada fora de contexto e sem abertura (inclusive impresso) → 403
  `out_of_context`; rascunho de outro autor → 403 `not_author`;
  `/clinical_record/patients/:id` mantém `opening_required`.
- Iniciar consulta também responde 403 `missing_role`, `missing_link`.
- Aberturas: `POST /clinical_record/openings` → 201; CPF inválido também 404
  `patient_not_found` (sem enumeração); só profissional com vínculo de CBO
  permitido abre; relatório com `from`/`to` no fuso da cidade, até 500 linhas,
  mais novas primeiro.
- Limites: exames só do grupo 02 da SIGTAP, até 100; até 50 problemas e 12
  condutas; tipos de atendimento 1, 2, 5, 6; condutas 1, 2, 4–12 e 14 (rótulos no
  `consultation_mapping.yml`).
- `cid10_allowed_for_cbo` em `consultation_options` considera qualquer vínculo
  permitido do usuário.
- Campos opcionais (`onset_on`, `onset_precision`, `cid10_justification`) só
  aparecem com valor; o problema da lista traz `resolved_on`.
- Encaminhamento vai à ficha só como conduta (o campo `encaminhamentos` do MIAI
  exige tabela de especialidades que o projeto não tem).
- Produção: `correction_pending` aparece em `fichas` com `replaces_outbox_id`
  apontando a aceita e fica fora de `counts`; "não geradas", "gerar de novo" e
  "Reenviar" valem também para `source_type: "Consultation"`.
- Impresso com a gem Prawn (Ruby puro); `pdf-reader` só no grupo de teste.

**Achado da confirmação do layout:** o MIAI e as regras de CBO **não restringem
CID-10 por CBO** (só limitam quais CBOs registram o MIAI). O YAML guarda
`rule: miai_table`; restringir CID-10 a médicos, como o PEC faz, é uma linha no
YAML. **Decidido pelo usuário (2026-10-07): CID-10 só para médicos (grupos
2251–2253), como no PEC; os demais profissionais usam só CIAP-2.** O
`consultation_mapping.yml` usa `rule: physicians_only`.

**Decisões do usuário durante a execução (2026-10-08)**
- Não médico (CBO fora de 2251–2253) pode avaliar ou resolver problema CID-10
  já existente na lista; só não registra CID-10 novo nem acrescenta/troca
  justificativa CID-10 de exame. A ficha LEDI da consulta de não médico
  **omite** os problemas CID-10 e envia só os CIAP-2 (como no PEC).
- Como o MIAI exige ao menos um problema avaliado, `POST /attendance/consultations/:id/finalize`
  ganha 422 `ciap2_required_for_cbo` quando um não médico finaliza sem nenhum
  problema CIAP-2 avaliado. Não se aplica ao adendo.
- `cid10_allowed_for_cbo` = algum vínculo permitido do usuário com CBO de médico.
- Chave desconhecida no `PATCH /attendance/consultations/:id` é ignorada (sem 422).
- O varredor das 23h (`Ledi::ScreeningFichaSweepJob`, módulo 18) passa a cobrir
  também a ficha de consulta de atendimentos fechados (competência atual ou
  anterior).
- Coluna `consultation_addenda.item_changes` (colisão com ActiveModel::Dirty);
  a chave JSON continua `changes`.
- **Leitura da consulta finalizada (2026-10-09, corrige o 403 da autora após a
  finalização):**
  1. **Autora** lê e imprime a própria consulta finalizada a qualquer momento,
     sem atendimento aberto nem abertura; lista nova "Minhas consultas" no
     dashboard. Trilha `clinical_record.viewed` com `access: "author"`, fora do
     relatório de aberturas.
  2. **Administrador da cidade** (`municipal_admin`): consultas por profissional
     com conteúdo completo (SOAP, itens, adendos), **só leitura** (sem imprimir,
     sem adendo), com step-up; trilha `access: "administrative"`, listada no
     relatório de aberturas. Decisão tomada pelo usuário ciente do risco CFM/LGPD.
  3. **Outro profissional**: lê em contexto ou por abertura justificada, como
     antes, mas só visualiza. Imprimir e adendo passam a ser só da autora (403
     `not_author`); some o adendo de terceiro com `opening_id`.
  4. `access` passa a `in_context` | `justified` | `author` | `administrative`.
  Rotas finais: no balanço da Task 21 do api (sessão "MVP Module 19").
