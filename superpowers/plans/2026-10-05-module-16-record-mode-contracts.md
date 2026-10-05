# Módulo 16 — Contratos entre apps

**Spec:** `superpowers/specs/2026-10-05-module-16-record-mode-and-export-design.md` · **ADR:** `adr/0028.md`

Fonte única dos formatos usados por mais de um app. Os planos (`contracts`,
`api-foundation`, `api-exporter`, `dashboard`, `admin`, `maintenance`) citam
este arquivo; mudar um formato começa aqui.

Erros HTTP sempre `{ "error": "<reason>" }`. Step-up: 401
`{ "error": "mfa_required" }` (padrão existente).

## 1. Sessão — `contracts/session` `session-v1.1.0` (MINOR)

`session_user` ganha `features` (opcional, array de string; `uniqueItems`, cada item `^[a-z][a-z0-9_]*$`): chaves de
interruptor **ligadas** para a cidade do host. Ausente na sessão do console de
plataforma. Consumidor trata ausente como `[]` e ignora chave desconhecida.

```json
"features": ["ledi_export", "cadsus_lookup"]
```

Recusa de rota de funcionalidade desligada:
`403 { "error": "feature_disabled", "feature": "ledi_export" }`.

## 2. Catálogo de interruptores (código do `api`)

| Chave | Pré-requisitos (`missing` quando faltam) |
|---|---|
| `ledi_export` | `record_mode_off`, `pec_url_missing`, `ibge_code_missing`, `credential_missing:ledi`, `credential_unauthorized:ledi` |
| `cadsus_lookup` | `credential_missing:cadsus`, `credential_unauthorized:cadsus` |

## 3. API de manutenção (GraphQL)

**Código IBGE:** a fonte única é o `city_profile.ibge_code` que já existe no
banco da cidade (gravado pelo operador no provisionamento, `POST /cities`, e
exposto em `City.profile.ibgeCode`). **Não nasce** `cities.ibge_code` na
plataforma; a importação do CNES lê o IBGE de cada cidade ativa no banco dela.

```graphql
type CityFeature { key: String!, description: String!, enabled: Boolean!,
                   usable: Boolean!, missing: [String!]!,
                   changedAt: ISO8601DateTime, changedBy: String }
# em City:
features: [CityFeature!]!
recordMode: String!          # off | integrated | record (só leitura aqui)
# o IBGE continua em City.profile.ibgeCode (existente)
# mutation (padrão das mutations existentes, BaseMutation#audited):
setCityFeature(citySlug: String!, key: String!, enabled: Boolean!):
  { ok: Boolean!, errors: [UserError!]!, feature: CityFeature }
```
`enabled`, `changedAt`, `changedBy` vêm da plataforma; `usable`/`missing` leem o
banco da cidade e, se ela estiver inalcançável, degradam para `usable: false`,
`missing: ["city_unreachable"]` — o liga/desliga continua funcionando.
Erros: `unknown_city`, `unknown_feature`. Auditoria
`city.feature_changed` (`city_id`, `key`, `enabled`, `maintainer_id`).

## 4. Console de plataforma (`PlatformConsoleHost`, módulo `operators`)

### 4.1 `PATCH /cities/:id/record_settings`
Corpo: `{ "record_mode", "ibge_code", "pec_url" }` (qualquer subconjunto).
`ibge_code` grava no `city_profile` do banco da cidade (fonte única).
- 422 `invalid_record_mode`, `invalid_ibge_code` (7 dígitos), `invalid_pec_url`
  (só `https://`, sem credencial na URL); 503 `city_unreachable` se `ibge_code`
  veio e o banco da cidade não responde;
- 200 `{ "city": <cidade de GET /cities/:id, com record_mode, ibge_code, pec_url, features> }`.
Auditoria `city.record_settings_changed` (`city_id`, campos mudados, nunca
valores de URL).

### 4.2 `GET /cities/:id` e `GET /cities` (existentes, ganham campos)
`record_mode`, `ibge_code`, `pec_url`, `city_reachable`,
`features: [{ key, enabled, usable, missing }]`. O `show` passa a devolver
também os campos do item da lista (`name`, `uf`, `created_at`…). Com o banco da
cidade inalcançável: `ibge_code: null` e `city_reachable: false` (o console
bloqueia a edição do IBGE nesse estado), sem falhar a resposta.

**Regras do PATCH (§4.1):** `null` limpa `pec_url` e `ibge_code`; `""` é tratado
como `null`; `record_mode` nunca é nulo (422 `invalid_record_mode`). Cidade
inexistente: 404 `{ "error": "not_found" }`. Auditoria: um só evento,
`city.record_settings_changed`.

### 4.3 `GET /city_production`
Resumo por cidade da competência corrente e da anterior (corrente primeiro),
no envelope `{ "data": ... }` como `/cities` e `/city_analytics`. Entregue pelo
plano **api-exporter** (lê a `ledi_outbox`):

```json
{ "data": { "cities": [ { "slug": "curitiba", "name": "Curitiba", "record_mode": "record",
  "competences": [ { "competence": "202610", "deadline_on": "2026-11-14",
     "business_days_left": 7, "accepted": 120, "rejected": 3, "pending": 10,
     "failed": 0, "alert": "none" } ] } ],
  "terminology": { "sigtap_current_competence": "202610", "sigtap_imported": true,
                   "sigtap_alert": false } } }
```
`sigtap_alert` é calculado no `api` (dia ≥ 5 do mês em `America/Sao_Paulo` sem a
SIGTAP da competência corrente).
`alert`: `none` | `attention` (≤ 5 dias úteis com pendente/recusada) |
`critical` (`record` e zero aceitas a ≤ 3 dias úteis).

## 5. Cidade (sessão municipal, banco da cidade)

### 5.1 Integrações (`municipal_admin`)
- `GET /integrations` →
  `{ "record_mode", "pec_url_set": bool, "ibge_code_set": bool,
     "credentials": [ { "kind": "ledi"|"cadsus", "set": bool, "set_at", "set_by",
       "last_check_at", "last_check_status": "ok"|"unauthorized"|"unreachable"|"error"|null,
       "last_check_message" } ],
     "features": [ { "key", "enabled", "usable", "missing": [...] } ] }`
  — nunca devolve segredo.
- `PUT /integrations/credentials/:kind` (step-up) corpo `{ "username", "password" }`
  → 200 com o item de `credentials`; 422 `invalid_credential` (campo vazio),
  `unknown_kind`.
- `POST /integrations/credentials/:kind/check` → executa o teste e devolve o item
  atualizado; 409 `credential_missing`.

### 5.2 CNES (`municipal_admin`)
- `GET /cnes` →
  `{ "snapshot": { "competence", "imported_at" } | null,
     "proposals": [ { "id", "kind": "unit"|"team"|"member", "action": "link"|"create"|"end",
        "local": {...} | null, "cnes": {...}, "confidence": "exact"|"probable" } ],
     "divergences": [ { "kind": "no_bond_in_cnes"|"cbo_mismatch"|"team_inactive_in_cnes"|"unit_without_cnes",
        "subject": { "type", "id", "label" }, "detail" } ] }`
  `local`/`cnes` mostram nome, CNES/INE/CBO e CPF/CNS **mascarados**.
- `POST /cnes/apply` (step-up) corpo `{ "proposal_ids": [...] }` →
  `{ "applied": n, "skipped": [ { "id", "reason": "stale" } ] }` (proposta que
  mudou desde a leitura é pulada).
- `GET /cnes` sem retrato importado: `snapshot: null`, listas vazias.

### 5.3 Produção e-SUS (`municipal_admin`; leitura também para `analyst`)
- `GET /production?competence=AAAAMM` (padrão: corrente) →
  `{ "competence", "deadline_on", "business_days_left", "alert",
     "counts": { "accepted", "rejected", "pending", "sending", "failed" },
     "rejections": [ { "message", "count" } ],
     "fichas": [ { "id", "ficha_type", "status", "attempts", "last_error", "created_at", "accepted_at" } ] }`
  (`fichas` paginado, 50 por página, `?page=`). Exige `ledi_export` ligado.
- `POST /production/fichas/:id/resend` (step-up) → 200 com a ficha; 409
  `not_rejected`. Regra de `uuid` definida pela prova técnica.

### 5.4 CADSUS na validação presencial
- `POST /attendance/cadsus_lookup` corpo `{ "code" }` (o mesmo código do
  `POST /attendance/lookup`) → exige `cadsus_lookup` utilizável;
  `{ "found": bool, "cns_masked", "birth_date_matches": bool|null,
     "sex_matches": bool|null }` — nunca devolve nome, mãe ou endereço.
  Erros: 503 `cadsus_unavailable`, 404 `not_found`.
- A confirmação da validação (`Citizens::Verify`) ganha `cadsus_confirmed: bool`;
  com `true`, grava `citizens.cns` e `citizens.cadsus_checked_at` da última
  consulta da mesma sessão (até 10 min).

## 6. Eventos e auditoria (só ids)

| Nome | Onde | Payload |
|---|---|---|
| `city.feature_changed` | plataforma (`Platform.audit`) | `city_id, key, enabled, maintainer_id` |
| `city.record_settings_changed` | plataforma | `city_id, fields` |
| `terminology.release_activated` | plataforma | `kind, version` |
| `cnes.snapshot_imported` | plataforma | `competence, ibge_codes_count` |
| `integration_credential.changed` | cidade | `kind, user_id` |
| `cnes.proposals_applied` | cidade | `user_id, count` |
| `ledi.ficha_accepted` / `ledi.ficha_rejected` | cidade | `outbox_id, ficha_type, competence` |
| `citizen.cadsus_looked_up` | cidade | `citizen_id, user_id, found` |

## 7. Ordem de entrega

1. `contracts` `session-v1.1.0` (push com autorização).
2. `api-foundation` (interruptores, modo, credenciais, terminologias, CNES, CADSUS).
3. `api-exporter` (prova técnica primeiro; depende da fundação).
4. `maintenance` depois da fundação; `admin` e `dashboard` depois do exportador
   (Produção) — as telas de Integrações, CNES e CADSUS podem começar com a fundação.
