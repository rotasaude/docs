# Módulo 20 — Contratos entre apps

**Spec:** `superpowers/specs/2026-10-11-module-20-patient-timeline-design.md` · **ADR:** `adr/0035.md`

Fonte única dos formatos entre `api`, `dashboard`, `wpda` e `maintenance`.
Convenções do projeto: `{ "error": "<reason>" }`; step-up 401 `mfa_required`;
interruptor 403 `{ "error": "feature_disabled", "feature": "patient_timeline" }`;
Content-Type JSON em toda escrita por cookie (inclusive DELETE); leitura do
profissional por `ClinicalRecord::Access` (generalizado para a pessoa).

## 1. Valores comuns

- `kind`: `triage` | `attendance` | `screening` | `appointment` | `consultation` |
  `clinical_document` | `controlled_notification` | `campaign`.
- `access` da trilha `clinical_record.viewed`: + `timeline` | `citizen` | `pec_external`.

## 2. Entrada

Profissional (`<entry>`):
```json
{ "id": "<kind>:<source_id>", "kind", "occurred_at", "unit": { "id", "name" }|null,
  "professional": { "name", "cbo_code" }|null, "title", "summary"|null, "status"|null,
  "detail_ref": { "type", "id" } }
```
Cidadão (`<citizen_entry>`): `{ "id", "kind", "occurred_at", "unit": { "name" }|null, "professional_name"|null, "title", "summary"|null, "status"|null, "document"?: { "id", "short_code", "verification_url", "downloadable": bool } }`.
Nunca no cidadão: texto clínico livre, CID sem nome, número SNCR, dado de outra pessoa.

## 3. Profissional (`/attendance`)

- `GET /attendance/timeline?patient_id=|attendance_id=|opening_id=&before=&limit=&kinds[]=&from=&to=`
  → `{ "person": { "display_name", "partial": bool, "patient_id"|null }, "entries": [<entry>], "next_before"|null }`.
  `limit` padrão 50, máximo 100; `before` é cursor opaco (`occurred_at|kind|id`).
  403 `out_of_context`, `opening_required`, `missing_role`; 422 `invalid_cursor`,
  `invalid_period`; 404 `not_found`.
- `partial: true` quando a cidade não está no modo `record`.

## 4. Cidadão (sessão do wpda)

- `GET /citizen/history?before=&limit=` → `{ "verified": bool, "partial": bool, "entries": [<citizen_entry>], "next_before"|null }`.
  Par só declarado: `verified: false`, só `triage` e `appointment` do próprio par.
- `GET /citizen/health_summary` → `{ "problems": [ { "terminology", "code", "label", "since"|null } ], "medications": [ { "label", "dosage_summary"|null, "since"|null } ], "exam_requests": [ { "sigtap_code", "label", "requested_at" } ], "vitals": { "blood_pressure"?: { "systolic", "diastolic", "measured_at" }, "weight_kg"?: { "value", "measured_at" }, "capillary_glucose"?: { "value", "moment", "measured_at" } } }`;
  403 `not_verified`; 404 `not_available` (sem paciente ou sem `clinical_record`).
- `GET /citizen/documents/:id/pdf` → `application/pdf` (assinado quando digital e
  assinado; senão o do papel); 404 `not_found` (documento de outra pessoa,
  cancelado ou inexistente — sem distinguir); 403 `not_verified`.

## 5. Página do PEC (sem sessão do Rota Saúde, host da cidade)

- `GET /pec/timeline?documentoCidadao=&documentoProfissional=&cnes=&pec_timestamp=&<assinatura>`
  (nomes exatos conferidos na Task 0 contra a documentação oficial) → HTML só
  leitura; cria sessão curta (cookie isolado, 15 min, `SameSite=None; Secure`).
  Falha → HTML genérico com código: `pec_disabled`, `pec_signature_invalid`,
  `pec_expired`, `pec_replayed`, `pec_unit_unknown`, `pec_professional_not_linked`,
  `pec_citizen_not_verified`. `Content-Security-Policy: frame-ancestors <pec_origin>`.
- `GET /pec/timeline/more?before=` (com a sessão curta) → fragmento HTML com a próxima página.

## 6. Maintenance (GraphQL, token humano)

- Interruptores `patient_timeline` e `pec_external_view` (missing:
  `patient_timeline_disabled`, `record_mode_not_integrated`).
- `cityPecConfig(citySlug) { pecOrigin, hmacKeyConfigured: Boolean!, hmacKeyUpdatedAt }`;
  mutations `setCityPecOrigin(citySlug, pecOrigin)` e
  `rotateCityPecHmacKey(citySlug, key)` → `{ ok, errors }` (a chave nunca volta).

## 7. Eventos (só ids)

| Nome | Payload |
|---|---|
| `clinical_record.viewed` | + `access: timeline/citizen/pec_external`, `person_ref` |
| `pec.external_view_opened` | `user_id, unit_id, patient_id|null` |
| `pec.external_view_denied` | `reason_code` |
| `city.pec_config_changed` (plataforma) | `city_id, field` |

## 8. Ordem de entrega

0. Conferir na documentação oficial dos "sistemas externos" do PEC o formato exato da chamada e da assinatura.
1. `api` (porta de dev sugerida 3040; migrações a partir de 20261011800001).
2. `dashboard` (Vite 5190) e `wpda` (Vite 5191). 3. `maintenance`.

## 9. Acréscimos da escrita dos planos (2026-10-11)

Valem sobre as seções acima e sobre os planos.

**PEC ("sistemas externos", documentação oficial lida)**
- Parâmetros reais: `documento-profissional`, `documento-cidadao`, `cnes` (hífen,
  minúsculas); o PEC acrescenta `pec_timestamp` (segundos Unix) e `pec_sign` =
  HMAC-SHA256 em hexadecimal minúsculo da URL completa (protocolo, host, caminho
  e query) até antes de `&pec_sign=`. A chave é gerada pelo PEC ao cadastrar o
  integrador e não é recuperável. `documento-cidadao` só vem quando o sistema
  externo é cadastrado no Atendimento (não no menu).
- Sessão curta: token cifrado de 15 min no cabeçalho `X-Pec-View-Session` (não
  cookie — navegadores bloqueiam cookie de terceiro em iframe); "carregar mais"
  devolve fragmento HTML e o cursor em `X-Next-Before`; sessão inválida → 401.
- Códigos novos: `pec_citizen_missing` (acesso pelo menu), `pec_session_expired`;
  cidadão identificado só por CNS cai em `pec_citizen_not_verified`. Página de
  recusa 403; "carregar mais" 401.
- `pec_external_view` exige também `pec_origin` e `pec_hmac_key` (missing
  `pec_origin_missing`, `pec_hmac_key_missing`).
- Maintenance: `cityPecConfig` anulável com recusa em `errors`;
  `setCityPecOrigin(citySlug, pecOrigin)` aceita `null`; origem
  `scheme://host[:porta]`, `https` (http só fora de produção);
  `rotateCityPecHmacKey(citySlug, key, code)` **exige TOTP**; chave de 16–512
  caracteres ASCII visíveis; recusas `invalid_origin`, `invalid_key`,
  `unknown_city`, `city_unreachable`. Evento `city.pec_config_changed` com
  `maintainer_id`; auditoria `maintenance.city.pec_config_changed`.
- Gates de go-live: o proxy de produção encaminha `/pec/*` ao api preservando
  `Host` e `X-Forwarded-Proto` (a URL assinada é `request.base_url +
  original_fullpath`) — provar em staging; em produção, sem maintenance, origem e
  chave só pelo console.

**Profissional**
- Alvo por prioridade `opening_id` > `patient_id` > `attendance_id`; 409
  `citizen_not_verified` (atendimento de par declarado); 422 `invalid_kinds`; 404
  `not_found` sem alvo ou alvo inexistente; 403 `opening_required` para abertura
  vencida ou de outro usuário.
- Abertura justificada vale com `patient_timeline` ligado mesmo sem prontuário:
  sem paciente, é feita por um par validado (`citizen_id` na abertura) e responde
  `patient_id: null`; `clinical_record.opened` ganha `citizen_id`. Desligado, o
  19a segue igual.
- Fora de contexto, o detalhe só abre a consulta; documento, escuta e pedido
  ficam no resumo da própria entrada.
- `detail_ref`: agendamento → `appointment_request` (o pedido); Notificação →
  `clinical_document`; campanha → `campaign`. `cbo_code` nulo em atendimento e
  agendamento. `title`/`summary` em português; `status` é código fixado por kind
  (a tela traduz).

**Cidadão**
- Rotas por par: `GET /citizen/history?citizen_id=`, `GET /citizen/health_summary?citizen_id=`
  (a sessão do wpda é por telefone e cobre vários pares); PDF
  `/citizen/documents/:id/pdf` conferido contra os pares validados da sessão.
- `kinds[]` e `limit` 1–100 no histórico; `document` sempre presente (`null`
  fora de documento; `downloadable: false` sem PDF); escuta e consulta com
  `status: null`.
- `GET`/`POST /citizen/session` devolvem `features: string[]` (interruptores
  utilizáveis da cidade).
- PDF: 404 para documento digital não assinado, cancelado, de outra pessoa ou
  com `clinical_documents` desligado; sai como anexo; limite 30 por 10 min.
- Resumo e histórico: ordem 403 `feature_disabled` > 404 `not_found` > 403
  `not_verified` > 404 `not_available` (o interruptor é da cidade e não depende
  do `citizen_id`; corrigido na Task 10 do api, ee3eb4e); problema sem nome não aparece; exame sem nome sai como
  "Procedimento <código>"; sinais vitais da consulta ou escuta mais recente. No
  wpda, `not_available` aparece como "sem resumo", sem erro.

**Trilha e migração**
- `clinical_record.viewed` leva `person_ref` (`patient:<id>` | `citizen:<id>`),
  `user_id` nulo no acesso do cidadão e o motivo no acesso justificado.
- Migração de cidade `20261011800002` (a `20261011800001` já existe — api#51);
  sem migração de plataforma.
- **Rota do profissional (Task 8 do api, de5a552):** `kinds[]` escalar ou vazio
  → 422 `invalid_kinds`; os 422 de parâmetro (`invalid_kinds`, `invalid_cursor`,
  `invalid_period`) saem antes da resolução do alvo; pessoa vazia → 404
  `not_found` sem trilha. `occurred_at` sai sem fração de segundo: as telas
  paginam só por `next_before`, sem reconstruir o cursor.
