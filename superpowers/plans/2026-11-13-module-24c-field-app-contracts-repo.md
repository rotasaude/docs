# Módulo 24c — `field-sync` v1.2.0 (repo contracts) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Ordem de entrega (contrato §11):** prova técnica do campo → **contracts v1.2.0 sem tag (Tasks 0–4)** → api (copia os esquemas daqui, presos por SHA-256) → **contracts: SHA-256 conferidos e tag (Task 5, com autorização)** → campo → dashboard → maintenance → catálogo. **Em paralelo, depois da prova técnica do campo:** domínio `map-style/` e tag `map-style-v1.0.0` (Task 6, com autorização; Q11) — antes da Task 15 do campo e da Task 2 do dashboard, que o copiam por lock. Nenhuma task daqui depende do api rodando, salvo o `check.sh` (usa o `json_schemer` do container `api`).

**Goal:** Publicar o MINOR `field-sync-v1.2.0` (compatível com o v1.1.0): item `family` no envio; `family_client_uuid`; `location` no domicílio e na visita; `version` no `accepted` (obrigatório) e no `duplicate` (opcional) dos cadastros; `map` e `reference_version` na sessão; o esquema novo `reference-response-v1.json`; em `domains-v1.json`, `errors.http` com `device_suspended`, `map_not_found`, `range_not_satisfiable` e os códigos de borda da cidade, `errors.item` com `invalid_location` e `family_not_found`, e os limites de `location`; exemplos, README e CHANGELOG (F-24.19, F-24.20, F-24.21, F-24.24; ADR 0053). E o domínio novo `map-style/` (`map-style-v1.0.0`: estilo `light` e `dark` em português, glyphs e sprites; Q11).

**Architecture:** Mesma mecânica do v1.0.0/v1.1.0, sem ferramenta nova: os esquemas mudam à mão; os exemplos saem de `field-sync/tools/make_examples.py` (determinístico) para `examples/` + `manifest.json`; `field-sync/tools/check.sh` valida o manifesto com o `json_schemer` 2.5 do api; `field-sync/tools/check_domains.py` confere `$defs` repetidos iguais, `enum` = `domains-v1.json` e erros conhecidos. O `$defs/location` nasce idêntico em `upload-request-v1.json`, `upload-response-v1.json` e `changes-response-v1.json` (o `check_domains.py` cobra), com `accuracy_m` obrigatório só quando `source` é `gps` (`if`/`then`, Q3), e o `$defs/household` da resposta e do download continua idêntico (ganha `location`). A regra "visita só `gps`" fica em dado (`domains-v1.json`, `location.visit_sources`), decidida pelo api — o esquema aceita `gps` e `map` nos dois lugares. Erro de esquema sob `/body/location` o api devolve como `rejected invalid_location` com o mesmo `path` (Q2); os exemplos inválidos de `location` provam a forma, não o código. O `map-style/` segue a mecânica do `design-tokens/` (gerador Node, `dist/` versionado, `node --test`, consumo por lock com SHA-256).

**Tech Stack:** JSON Schema 2020-12; `json_schemer` 2.5 (container `api`); `python3` do host; `shasum`; Node 22 do host (só `map-style/`, com `@protomaps/basemaps` fixado pela prova do campo); git em `/opt/homebrew/bin/git`.

**Spec:** `docs/superpowers/specs/2026-11-13-module-24c-field-app-design.md` (decisões 8, 11, 13, 16, 19, 24; §3 "contracts"). **Contrato (fonte única):** `docs/superpowers/plans/2026-11-13-module-24c-field-app-contracts.md` §1, §2, §3.1–§3.5, §4, §10, §11 e §13 (Q1–Q49, que prevalece). Base: `field-sync-v1.1.0` (contracts `673bb14`). Molde: `docs/superpowers/plans/2026-11-12-module-24b-home-visit-contracts-repo.md`. As lacunas C-1 a C-10 e a D-1 do dashboard foram decididas no §13 (Q1–Q11) e o plano já as aplica; o resumo está em **"Decisões aplicadas (§13 do contrato)"**, no fim.

## Global Constraints

- Repo `contracts` na raiz do monorepo (`$R/contracts`, remote `git@github.com:rotasaude/contracts.git`), `origin/main` em `673bb14` ou mais novo, tags `field-sync-v1.0.0` e `field-sync-v1.1.0` existentes, nenhuma `field-sync-v1.2.0`.
- Tag `field-sync-v1.2.0`, anotada, mensagem `field-sync-v1.2.0 — MINOR`. `session.contract` continua `"field-sync-v1"` (const). Sem entrada no CHANGELOG, não há tag.
- Arquivos novos seguem a convenção: `$id` = `https://rota-saude/schemas/field-sync/<arquivo>`, `title` = `rotasaude.field_sync.<nome>.v1`; fechado em todo nível (`additionalProperties: false`), salvo as seções de dicionário (K-2 do 24b) e os blocos de domínio da referência (§3.5: "seções de dicionário ficam `type: object`").
- **Compatível com o v1.1.0:** nada que o v1.1.0 aceitava no **pedido** passa a ser recusado (`location`, `family`, `family_client_uuid` são opcionais/novos); nas **respostas**, campos novos podem ser obrigatórios (o api é o único produtor e passa a emiti-los no mesmo deploy).
- `location` (contrato §4 e Q3): `latitude` −90..90 e `longitude` −180..180 com até 8 casas (`multipleOf: 0.00000001`, conferido com o `json_schemer` do api: `-25.42847123` passa, `-25.428471234` falha; o Ajv do app valida igual com `multipleOfPrecision: 8`, Q49); `accuracy_m` 0..10000 com até 1 casa (`multipleOf: 0.1`), **obrigatório com `source: "gps"` e opcional com `"map"`**; `source` `gps` | `map`; `latitude`, `longitude` e `source` sempre presentes. Na visita, `location` é objeto (ausente = sem coordenada; `null` → `invalid_location`); no domicílio, objeto ou `null` (apaga).
- Dados dos exemplos fictícios, como no v1 (coordenadas de Curitiba inventadas: `-25.42847123`, `-49.27325411`).
- Worktree **fora de `contracts/`**: `$R/.worktrees/contracts-mod24c`, branch `feat/mod-24c-field-app` a partir de `origin/main`. Comandos a partir de `$R`, com o compose de pé (`docker compose ps api` rodando).
- Testes: este repo não tem `bin/test`; a verificação é `check.sh` + `check_domains.py`. Nada aqui roda `rspec` nem `vitest`.
- Git: `/opt/homebrew/bin/git`. Nunca `git add -A` nem `git add .`. Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Merge, tag e push só com autorização explícita do usuário, uma etapa por vez; antes do push, `origin/main..main` só com os commits desta entrega.
- Board #1: o contracts entra em F-24.19 (`device_suspended`), F-24.20 (`map` na sessão e `map-style/`), F-24.21 (`family`, referência) e F-24.24 (`location`) — Q10 (spec §8 corrigido). Os cards estão em `Not Started`; quem move para In Progress é o plano do api (Task 0 aqui só confere).
- Execução coordenada: ao fim de cada task, uma linha à sessão "Próximos módulos" (`SendMessage`).

## Ambiente de execução

- `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `G=/opt/homebrew/bin/git`, `C=$R/.worktrees/contracts-mod24c`.
- Caminhos das tasks relativos a `$C`.
- Verificação (de `$R`): `$C/field-sync/tools/check.sh [esquema]`, `python3 $C/field-sync/tools/check_domains.py [--release]`, `python3 $C/field-sync/tools/make_examples.py` (regera e imprime a contagem por suíte).

## Review Focus

1. **Coordenada com 9 casas, só latitude, fora da faixa, GPS sem precisão ou `null` na visita** passando no esquema — o app gravaria e o api recusaria tarde (ou a ficha LEDI sairia com ponto inventado). Testes: `invalid-location-latitude-only.json`, `invalid-location-latitude-out-of-range.json`, `invalid-location-nine-decimals.json`, `invalid-location-gps-without-accuracy.json`, `invalid-home-visit-location-null.json`; ponto do mapa sem `accuracy_m` continua válido (`upload-household-location.json`) (Task 1).
2. **`accepted` de cadastro sem `version`** (ou `version` em visita/atividade): o app não teria como atualizar a base, e o próximo item do mesmo domicílio iria com versão vencida para sempre. O `duplicate` sem `version` é válido (recibo anterior à coluna `result_version`, Q5). Testes: `invalid-response-household-accepted-without-version.json`, `invalid-response-home-visit-with-version.json`, `upload-response-duplicate-without-version.json` (Task 1).
3. **Família citando duas vezes** (`household_id` e `household_client_uuid`; responsável com `patient_id` e `patient_client_uuid`; cadastro com `family_id` e `family_client_uuid`). Testes: `invalid-family-two-households.json`, `invalid-family-responsible-two-patients.json`, `invalid-registration-two-families.json` (Task 1).
4. **Exemplo antigo que vira válido sem ninguém notar:** `invalid-upload-unknown-type.json` usa `"type": "family"`, que passa a existir; sem troca, a suíte quebraria ou (pior) o exemplo deixaria de provar "tipo desconhecido". Teste: o arquivo passa a usar `"visit_sheet"` com o mesmo `expect` `/items/0/type enum` (Task 1).
5. **Dado de pessoa na referência offline** (CPF, CNS de profissional, cidadão) e **domínio que perde a ordem do LEDI** (objeto `{ "117": … }` no lugar da lista, Q27). Testes: `invalid-reference-professional-with-cpf.json` (`/team_professionals/0/cpf schema`), `invalid-reference-domain-as-object.json` (Task 2).
6. **Estilo de mapa que chama terceiro** (o estilo padrão do Protomaps aponta glyphs e sprites para `protomaps.github.io`) ou fonte pedida pelas camadas e ausente da cópia. Testes: `map-style/test/style.test.mjs`, "nenhuma URL http(s)" e "toda fonte das camadas existe em `dist/fonts`" (Task 6).

## Mapa de arquivos

| Arquivo (worktree `$C`) | Mudança | Task |
|---|---|---|
| `field-sync/session-response-v1.json` | `map`, `reference_version` (obrigatórios) | 1 |
| `field-sync/upload-request-v1.json` | `$defs/location`; item `family` + `$defs/family_body`; `family_client_uuid`; `location` em `household_body` e `home_visit_body` | 1 |
| `field-sync/upload-response-v1.json` | `$defs/location`; `location` em `$defs/household`; `family` nos `type`; `version` nos resultados | 1 |
| `field-sync/changes-response-v1.json` | `$defs/location`; `location` em `$defs/household` | 1 |
| `field-sync/tools/make_examples.py`, `field-sync/examples/*` | exemplos novos e ajustados | 1, 2 |
| `field-sync/reference-response-v1.json` | novo (`GET /field/v1/reference`) | 2 |
| `field-sync/domains-v1.json` | `errors.http`, `errors.item`, `location` | 2 |
| `field-sync/tools/check_domains.py` | referência na lista de esquemas; `location` × `domains-v1.json` | 2 |
| `field-sync/README.md`, `field-sync/CHANGELOG.md`, `README.md` (raiz) | regras do §3–§5 em texto curto; entrada v1.2.0; versão na tabela | 3 |
| — | SHA-256 ao api e parada | 4 |
| — | verificação final, merge, tag, push (com autorização) | 5 |
| `map-style/{package.json,package-lock.json,build.mjs,src/manifest.json,dist/**,test/style.test.mjs,README.md,CHANGELOG.md}`, `README.md` (raiz) | domínio novo, tag `map-style-v1.0.0` (com autorização) | 6 |

**Estratégia de teste:** cada mudança de esquema entra com seus exemplos válidos e inválidos no gerador **antes** do esquema (Step 1 de cada task); o `check.sh` falha nos inválidos novos com o esquema velho e passa com o novo. O `check_domains.py` ganha a conferência de `location` (inclusive `accuracy_m.required_for` × o `if`/`then` do esquema) e cobre a referência nos `$defs`. Contagens: hoje 87 casos (34 válidos); ao fim da Task 1, 114; ao fim da Task 2, **120 casos (49 válidos, 71 inválidos)** — o número exato é o que o `make_examples.py` imprime, e o README o repete. O `map-style/` tem teste próprio (`node --test map-style/test/*.test.mjs`) escrito antes do gerador (Task 6).

---

### Task 0: Base, cards e worktree

**Files:** nenhum.

**Interfaces:**
- Consumes: `contracts` `origin/main`; board #1 (`PVT_kwDOEbfGRc4BbxiB`).
- Produces: `$C` em `feat/mod-24c-field-app`; base anotada.

- [ ] **Step 1: Cards** (um `gh` simples por chamada)

```bash
gh project item-list 1 --owner rotasaude --limit 600 --format json --jq '.items[] | select(.title | test("^F-24\\.(19|2[014]) ")) | {id, title, status}'
```

Expected: os quatro cards existem (em 2026-11-13: `Not Started`). Ausente: **pare e reporte**. Não mova (o plano do api move).

- [ ] **Step 2: Base e worktree**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/contracts fetch -q origin
$G -C $R/contracts log --oneline -1 origin/main
$G -C $R/contracts tag -l 'field-sync-*'
$G -C $R/contracts worktree add $R/.worktrees/contracts-mod24c -b feat/mod-24c-field-app origin/main
cd $R && $R/.worktrees/contracts-mod24c/field-sync/tools/check.sh | tail -1 && python3 $R/.worktrees/contracts-mod24c/field-sync/tools/check_domains.py
```

Expected: `673bb14 feat: add too_many_requests to field sync item errors` ou mais novo; tags `field-sync-v1.0.0` e `field-sync-v1.1.0` (sem `v1.2.0`); `87/87 casos`; `defs-iguais`, `dominios-iguais`, `erros-conhecidos`, `prova: fcf23c87 (2026-11-12)`. Diferente: **pare e reporte** (outro módulo mexeu no `field-sync/`).

**Relato:** "contracts-mod24c Task 0: worktree em feat/mod-24c-field-app sobre <sha>; 87/87; cards F-24.19/20/21/24 conferidos."

---

### Task 1: Família, coordenada, `version` no resultado e sessão com mapa

**Files (worktree `$C`):**
- Modify: `field-sync/session-response-v1.json`, `field-sync/upload-request-v1.json`, `field-sync/upload-response-v1.json`, `field-sync/changes-response-v1.json`
- Modify: `field-sync/tools/make_examples.py`; regenerate `field-sync/examples/`

**Interfaces:**
- Consumes: os esquemas do v1.1.0; `TerritoryRegistry::SaveFamily` do api (`app/commands/territory_registry/save_family.rb`: corpo `{ responsible: { patient_id }, income_band, members_count, living_since, moved_out, version }`).
- Produces (`field-sync-v1.2.0`):
  - `$defs/location` (idêntico nos três arquivos de envio/resposta/download):
    ```json
    { "type": "object", "additionalProperties": false, "required": ["latitude", "longitude", "source"],
      "properties": {
        "latitude":   { "type": "number", "minimum": -90,  "maximum": 90,  "multipleOf": 0.00000001 },
        "longitude":  { "type": "number", "minimum": -180, "maximum": 180, "multipleOf": 0.00000001 },
        "accuracy_m": { "type": "number", "minimum": 0, "maximum": 10000, "multipleOf": 0.1 },
        "source":     { "enum": ["gps", "map"] } },
      "if": { "required": ["source"], "properties": { "source": { "const": "gps" } } },
      "then": { "required": ["accuracy_m"] } }
    ```
    `accuracy_m` obrigatório com `source: "gps"` e opcional com `"map"` (Q3: o app manda a resolução do toque; o dashboard não manda).
  - `upload-request-v1.json`: `item.type` ganha `"family"` e o `allOf` ganha o ramo `family` → `$defs/family_body`; `household_body.location` = `{ "anyOf": [ { "$ref": "#/$defs/location" }, { "type": "null" } ] }`; `home_visit_body.location` = `{ "$ref": "#/$defs/location" }`; `home_visit_body.family_client_uuid` e `individual_registration_body.family_client_uuid` (`$ref uuid`) com `{ "not": { "required": ["family_id", "family_client_uuid"] } }` no `allOf` de cada um; descrição da visita sem "latitude e longitude ficam para o 24c".
  - `$defs/family_body`:
    ```json
    { "type": "object", "additionalProperties": false,
      "required": ["version", "income_band", "members_count", "living_since"],
      "properties": {
        "household_id": { "$ref": "#/$defs/uuid" }, "household_client_uuid": { "$ref": "#/$defs/uuid" },
        "family_id": { "$ref": "#/$defs/uuid" },
        "version": { "type": "integer", "minimum": 1 },
        "responsible": { "type": "object", "additionalProperties": false, "minProperties": 1, "maxProperties": 1,
          "properties": { "patient_id": { "$ref": "#/$defs/uuid" }, "patient_client_uuid": { "$ref": "#/$defs/uuid" } } },
        "income_band": { "type": ["string", "null"] }, "members_count": { "type": ["integer", "null"], "minimum": 0 },
        "living_since": { "type": ["string", "null"] }, "moved_out": { "type": "boolean" } },
      "oneOf": [ { "required": ["household_id"] }, { "required": ["household_client_uuid"] } ] }
    ```
    `responsible` aninhado como no 24a (Q1; não existe `responsible_patient_client_uuid` plano); obrigatório só sem `family_id` é regra do api (`invalid_family` `responsible.patient_id`), não do esquema. `responsible.patient_client_uuid` pode citar um item `patient` **ou** um `individual_registration` com `new_patient` (Q26: o livro do api resolve os dois); o esquema não distingue, o README diz.
  - `upload-response-v1.json`: `result.type` e `stale_version.type` ganham `"family"`; `result.version` (`integer ≥ 1`); `$defs/household` ganha `location` (obrigatório, `anyOf [location, null]`); três ramos novos no `allOf` do `result` (Q5):
    1. `status = accepted` **e** `type ∈ {household, family, individual_registration}` → `required: ["version"]` (no `duplicate` desses três, `version` é permitido e opcional: recibo anterior à coluna `result_version` não a tem);
    2. `status ∈ {accepted, duplicate}` **e** `type ∉` esses três → `version: false`;
    3. `status ∈ {rejected, stale_version}` → `version: false`;
    e o ramo `stale_version` + `type: family` → `current: { "$ref": "#/$defs/household" }`. Em `stale_version` de `family` (nova ou existente), `id` = id do **domicílio** (Q4).
  - `changes-response-v1.json`: `$defs/household` idêntico ao da resposta (com `location` obrigatório e anulável).
  - `session-response-v1.json`: `required` + `map`, `reference_version`:
    ```json
    "map": { "anyOf": [ { "type": "null" }, { "type": "object", "additionalProperties": false,
             "required": ["version", "size_bytes", "sha256"],
             "properties": { "version": { "$ref": "#/$defs/date" }, "size_bytes": { "type": "integer", "minimum": 1 },
                             "sha256": { "type": "string", "pattern": "^[0-9a-f]{64}$" } } } ] },
    "reference_version": { "type": "string", "pattern": "^[0-9a-f]{64}$" }
    ```

- [ ] **Step 1: Exemplos primeiro (gerador)**

Em `make_examples.py`:
- Constantes: `LOC_GPS = {"latitude": -25.42847123, "longitude": -49.27325411, "accuracy_m": 8.5, "source": "gps"}`, `LOC_MAP = {"latitude": -25.42851, "longitude": -49.2733, "source": "map"}`, `MAP_SHA = "ab" * 32`, `REF_SHA = "cd" * 32`, `FAM2` (uuid novo, mesmo formato dos outros).
- `household()` ganha `"location": dict(LOC_GPS)` (todos os exemplos que o usam passam a trazer a chave). `session_acs()` ganha `"map": {"version": "2026-11-01", "size_bytes": 41234567, "sha256": MAP_SHA}, "reference_version": REF_SHA`.
- `invalid-upload-unknown-type.json` passa a `{"client_uuid": cu(1), "type": "visit_sheet", "body": {}}` (mesmo `expect`).
- Novos (arquivo → `expect`):

| Suíte | Arquivo | Conteúdo | `expect` |
|---|---|---|---|
| session | `session-without-map.json` | `session_acs()` com `map: None` | `valid` |
| session | `invalid-session-without-reference-version.json` | sem `reference_version` | `(root) required` |
| session | `invalid-session-map-short-sha.json` | `map.sha256 = "ab"` | `/map/sha256 pattern` |
| upload-request | `upload-family-new.json` | lote: `household` novo `cu(30)` (sem `household_id`), `patient` `cu(31)`, `individual_registration` `cu(32)` (`patient_client_uuid` `cu(31)`, `household_client_uuid` `cu(30)`, `is_responsible: True`), `family` `cu(33)` (`household_client_uuid` `cu(30)`, `version` 1, `responsible: {patient_client_uuid: cu(31)}`, `income_band` "3", `members_count` 2, `living_since` "2026-11") | `valid` |
| upload-request | `upload-family-new-from-registration.json` | lote (Q26): `household` novo `cu(30)`, `individual_registration` `cu(35)` com `new_patient` (com `social_name`) e `household_client_uuid` `cu(30)`, `is_responsible: True`; `family` `cu(36)` (`household_client_uuid` `cu(30)`, `version` 1, `responsible: {patient_client_uuid: cu(35)}`, `income_band` "3", `members_count` 1, `living_since` "2026-11") | `valid` |
| upload-request | `upload-family-update.json` | `family` `cu(34)`: `household_id` HH, `family_id` FAM, `version` 4, `income_band` "4", `members_count` 3, `living_since` "2019-03", `moved_out` False | `valid` |
| upload-request | `upload-household-location.json` | `household_update()` + `location: LOC_MAP` | `valid` |
| upload-request | `upload-household-location-cleared.json` | `household_update()` + `location: None` | `valid` |
| upload-request | `upload-home-visit-location.json` | `hv_resident()` + `location: LOC_GPS` | `valid` |
| upload-request | `upload-home-visit-family-client-uuid.json` | lote: `family` `cu(33)` de `upload-family-new` + visita `subject: "family_responsible"`, `household_id` HH, `family_client_uuid` `cu(33)` | `valid` |
| upload-request | `invalid-location-latitude-only.json` | visita com `location: {"latitude": -25.42847123, "source": "gps"}` | `/items/0/body/location required` |
| upload-request | `invalid-location-latitude-out-of-range.json` | `location.latitude = -91` | `/items/0/body/location/latitude minimum` |
| upload-request | `invalid-location-nine-decimals.json` | `location.latitude = -25.428471234` | `/items/0/body/location/latitude multipleOf` |
| upload-request | `invalid-location-gps-without-accuracy.json` | visita com `location: {"latitude": -25.42847123, "longitude": -49.27325411, "source": "gps"}` (Q3) | `/items/0/body/location required` |
| upload-request | `invalid-home-visit-location-null.json` | visita com `location: None` | `/items/0/body/location object` |
| upload-request | `invalid-family-two-households.json` | `upload-family-update` + `household_client_uuid` | `/items/0/body oneOf` |
| upload-request | `invalid-family-responsible-two-patients.json` | `responsible: {patient_id: PAT, patient_client_uuid: cu(31)}` | `/items/0/body/responsible maxProperties` |
| upload-request | `invalid-registration-two-families.json` | `registration_create()` + `family_client_uuid` `cu(33)` | `/items/0/body not` |
| upload-response | `upload-response-accepted-with-version.json` | `accepted` household HH `version` 4; `duplicate` family FAM `version` 5; `accepted` individual_registration REG `version` 2; `accepted` home_visit VISIT (sem `version`) | `valid` |
| upload-response | `upload-response-duplicate-without-version.json` | `duplicate` household HH sem `version` (recibo legado, Q5) | `valid` |
| upload-response | `upload-response-stale-family.json` | `stale_version` `type` family, `id` HH, `current_version` 5, `current` = `household()` com `version` 5 | `valid` |
| upload-response | `upload-response-invalid-location.json` | `rejected` household `invalid_location` `path` `location.latitude` | `valid` |
| upload-response | `upload-response-family-not-found.json` | `rejected` family `family_not_found` `path` `family_id` | `valid` |
| upload-response | `invalid-response-household-accepted-without-version.json` | `accepted` household sem `version` | `/results/0 required` |
| upload-response | `invalid-response-home-visit-with-version.json` | `accepted` home_visit com `version` 1 | `/results/0/version schema` |
| changes | `changes-household-location-null.json` | `page_first()` com `items[1].body.location = None` | `valid` |
| changes | `invalid-changes-household-without-location.json` | `page_first()` sem `items[1].body.location` | `/items/1/body required` |

Rode `python3 $C/field-sync/tools/make_examples.py`. Expected: `session-response-v1.json 8 · upload-request-v1.json 54 · upload-response-v1.json 34 · changes-response-v1.json 18 · total 114`.

- [ ] **Step 2: Rode e veja falhar**

Run: `cd $R && $C/field-sync/tools/check.sh | grep -c '^FAIL'`
Expected: os inválidos novos de `location`, `family` e `version` falham (o esquema velho os aceita ou recusa por outro motivo), e os válidos novos com `family`, `location`, `map` ou `version` falham por `schema`/`enum`. O `invalid-upload-unknown-type.json` continua `ok`.

- [ ] **Step 3: Mude os quatro esquemas** pelas Interfaces. `$defs/location` copiado byte a byte nos três arquivos (o `check_domains.py` cobra); `$defs/household` igual em `upload-response-v1.json` e `changes-response-v1.json`.

- [ ] **Step 4: Rode e veja passar**

Run: `cd $R && $C/field-sync/tools/check.sh | tail -1 && python3 $C/field-sync/tools/check_domains.py`
Expected: `114/114 casos`; `defs-iguais`; `dominios-iguais`; `ERROS FORA DA LISTA: upload-response-invalid-location.json: invalid_location; upload-response-family-not-found.json: family_not_found` (exit 1 — a Task 2 acrescenta os códigos; é o teste que falha dela). Se um `expect` sair com outro `type` do `json_schemer` (ex.: `anyOf` no lugar de `object` no `invalid-home-visit-location-null.json`, ou o ponteiro do `then` no `invalid-location-gps-without-accuracy.json`), ajuste só o `expect` e anote no relato.

- [ ] **Step 5: Commit**

```bash
cd $C && $G add field-sync/session-response-v1.json field-sync/upload-request-v1.json field-sync/upload-response-v1.json field-sync/changes-response-v1.json field-sync/tools/make_examples.py field-sync/examples
$G status --short
$G commit -m "feat: add family items, coordinates, result versions and the map manifest to field sync" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato:** "contracts-mod24c Task 1: family, location (accuracy_m só com gps), version (duplicate opcional) e map no field-sync; 114/114; códigos novos entram na Task 2."

---

### Task 2: Referência offline e domínios

**Files (worktree `$C`):**
- Create: `field-sync/reference-response-v1.json`
- Modify: `field-sync/domains-v1.json`, `field-sync/tools/check_domains.py`, `field-sync/tools/make_examples.py`; regenerate `field-sync/examples/`

**Interfaces:**
- Consumes: as formas do api que a referência reaproveita — `HomeVisits::Json.domains` (`app/services/home_visits/json.rb:152`), `CollectiveActivities::Json.domains` (`app/services/collective_activities/json.rb:95`), `config/ledi/registration_dictionaries.yml` (chaves de topo `version`, `types`, `domains`, `string_domains`, `sections`).
- Produces:
  - `reference-response-v1.json` (`title` `rotasaude.field_sync.reference_response.v1`), fechado no topo:
    - `required`: todas as chaves abaixo;
    - `reference_version`: `^[0-9a-f]{64}$`;
    - `registration_dictionaries`: `{ type: object, required: ["version", "domains", "sections"] }` (conteúdo do YAML sem `string_domains`, na forma do gerador do dashboard `ledi-dictionaries.mjs`, Q27): `domains` = objeto cujo valor de cada chave é **lista** `[{ code: integer | string, label: string }]` na ordem do YAML (fechado no item; objeto `{ "<code>": label }` recusado, porque o JS reordena chaves inteiras); `sections` aberto (K-2: fonte é o YAML);
    - `string_domains`: fechado, `required ["uf", "tipo_logradouro"]`, cada um lista de `{ code: string, label: string }` fechados;
    - `neighborhoods`: lista de `{ id: uuid, name: string≥1 }`;
    - `health_units`: lista de `{ id: uuid, cnes: string^[0-9]{7}$ | null, name: string≥1 }` (Q6: unidade sem CNES vem com `null`; a atividade a recusa, o app a trata como o dashboard trata `has_cnes: false`);
    - `team_professionals`: lista de `{ team_id: uuid, professional_id: uuid, name: string≥1, cbo: ^[0-9]{6}$, has_cns: boolean }` fechados (Q6: a função do dashboard `professional_memberships` filtrada a `in_team`, por isso sem `in_team`; `has_cns` obrigatório);
    - `home_visit_domains`, `collective_activity_domains`: `{ type: object }` com `required` das chaves de topo atuais (`motivo_visita desfecho turno tipo_glicemia tipo_de_imovel non_residential_property_types non_residential_reasons male_forbidden_reasons without_person_allowed_reasons responsible_document_reasons reasons_max measurement_ranges` e `activity_types meeting_topics target_audience health_topics health_practices procedures rules`), sem fechar (fonte é o YAML); um objeto só, não por equipe — o recorte por equipe fica em `team_professionals.team_id` (Q7).
  - `domains-v1.json`:
    - `errors.http` + `device_suspended`, `map_not_found`, `range_not_satisfiable` e os códigos de borda da cidade que o `/field/v1` pode devolver: `unknown_city`, `city_suspended`, `city_schema_behind` (Q8; todos com frase no app);
    - `errors.item` + `invalid_location`, `family_not_found` (`cpf_or_reason_required` **já está** desde o v1.0.0 — nada a fazer, Q9);
    - chave nova `location`: `{ "latitude": { "min": -90, "max": 90 }, "longitude": { "min": -180, "max": 180 }, "decimals": 8, "accuracy_m": { "min": 0, "max": 10000, "decimals": 1, "required_for": ["gps"] }, "sources": ["gps", "map"], "visit_sources": ["gps"] }` (Q3).
  - `check_domains.py`: `SCHEMAS` + `reference-response-v1.json`; par novo `location` comparando `domains.location` com `upload-request-v1.json#/$defs/location` (mínimos, máximos, `multipleOf` = `10**-decimals`, `source.enum` = `sources`, `accuracy_m.required_for` = o `const` do `if` com `then.required` = `["accuracy_m"]`).

- [ ] **Step 1: Exemplos da referência (gerador)**

Função `reference()` no gerador: `reference_version` `REF_SHA`; `registration_dictionaries` `{"version": "8.7.0", "domains": {"renda_familiar": [{"code": 3, "label": "1/2 a 1 salário mínimo"}], "abastecimento_de_agua": [{"code": 117, "label": "Rede encanada até o domicílio"}, {"code": 214, "label": "Poço/nascente no domicílio"}, {"code": 118, "label": "Cisterna"}]}, "sections": {"living_conditions": []}}` (a ordem 117, 214, 118 é a do YAML: prova que a lista não é reordenada); `string_domains` `{"uf": [{"code": "41", "label": "PARANÁ"}], "tipo_logradouro": [{"code": "081", "label": "RUA"}]}`; um bairro, duas unidades (uma com `cnes` "2384299", outra `None`), um profissional (`TEAM`, `PROF2`, "Carla Mendes", "515105", `has_cns` True); `home_visit_domains` e `collective_activity_domains` com as chaves exigidas (valores mínimos: listas vazias, `reasons_max` 36, `measurement_ranges` `{}`, `rules` `{}`).

Suíte nova `("reference-response-v1.json", {...})`:

| Arquivo | Conteúdo | `expect` |
|---|---|---|
| `reference.json` | `reference()` | `valid` |
| `invalid-reference-version-short.json` | `reference_version = "cd"` | `/reference_version pattern` |
| `invalid-reference-professional-with-cpf.json` | `team_professionals[0].cpf = "39053344705"` | `/team_professionals/0/cpf schema` |
| `invalid-reference-professional-without-has-cns.json` | sem `team_professionals[0].has_cns` (Q6) | `/team_professionals/0 required` |
| `invalid-reference-domain-as-object.json` | `registration_dictionaries.domains.renda_familiar = {"3": "1/2 a 1 salário mínimo"}` (Q27) | `/registration_dictionaries/domains/renda_familiar array` |
| `invalid-reference-without-health-units.json` | sem `health_units` | `(root) required` |

Rode o gerador. Expected: `… · reference-response-v1.json 6 · total 120`. (Use sempre a contagem impressa pelo gerador.)

- [ ] **Step 2: Rode e veja falhar**

Run: `cd $R && $C/field-sync/tools/check.sh reference-response-v1.json; python3 $C/field-sync/tools/check_domains.py`
Expected: o `check.sh` não acha o esquema (erro de arquivo); `ERROS FORA DA LISTA: … invalid_location; … family_not_found`.

- [ ] **Step 3: Escreva `reference-response-v1.json`, atualize `domains-v1.json` e `check_domains.py`** pelas Interfaces.

- [ ] **Step 4: Rode e veja passar**

Run: `cd $R && $C/field-sync/tools/check.sh | tail -1 && python3 $C/field-sync/tools/check_domains.py --release; echo "exit $?"`
Expected: `120/120 casos`; `defs-iguais`, `dominios-iguais` (com o par `location`), `erros-conhecidos`, `prova: fcf23c87 (2026-11-12)`, `exit 0`. Teste do verificador: troque `decimals` de `location` para 7 em `domains-v1.json`, rode, veja `DOMINIOS DIFERENTES: location` e exit 1, e desfaça; troque `accuracy_m.required_for` para `["map"]`, veja o mesmo erro, e desfaça.

- [ ] **Step 5: Commit**

```bash
cd $C && $G add field-sync/reference-response-v1.json field-sync/domains-v1.json field-sync/tools/check_domains.py field-sync/tools/make_examples.py field-sync/examples
$G commit -m "feat: add the offline reference schema and the coordinate, map and suspension codes to field sync domains" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato:** "contracts-mod24c Task 2: reference-response-v1.json (has_cns, domínios em lista), domains com device_suspended/map_not_found/range_not_satisfiable/unknown_city/city_suspended/city_schema_behind/invalid_location/family_not_found e location; 120/120; --release verde."

---

### Task 3: README e CHANGELOG

**Files:** Modify `field-sync/README.md`, `field-sync/CHANGELOG.md`, `README.md` (raiz).

**Interfaces:**
- Consumes: Tasks 1–2.
- Produces: texto que o app e o api citam; a tabela da raiz com `field-sync-v1.2.0`.

- [ ] **Step 1: `field-sync/README.md`**, regras curtas (uma por item; sem repetir o esquema):
  - tabela de arquivos + `reference-response-v1.json` (`GET /field/v1/reference`);
  - **Aparelho:** `401 device_revoked` = revogação manual (`user` ou `admin`; o app apaga tudo); `401 device_suspended` = revogação por substituição (`replaced`, Q17) ou automática (`membership_ended`/`user_disabled`), sem dizer qual (o app guarda a fila e apaga o território); `invalid_token` o app trata como `device_suspended`;
  - **Sessão:** `map` (`null` sem arquivo publicado; durante uma regeração ou depois de uma falha continua o publicado, Q30; `sha256` = `ETag` do `GET /field/v1/map`, que aceita um `Range` por requisição, `If-Range` e `If-None-Match`, responde `416 range_not_satisfiable` com `Content-Range: bytes */<tamanho>` e `404 map_not_found` sem mapa, Q34) e `reference_version` (resumo opaco: o app compara, nunca recalcula; muda → `GET /field/v1/reference`);
  - **Família (`family`):** só ACS (`cbo_not_allowed`); sem `family_id` = nova, com = atualização; `version` = do **domicílio**; responsável em `responsible.patient_id` ou `responsible.patient_client_uuid`; se o responsável tem cadastro individual ativo no domicílio, ele entra na família (L28 do 24a, Q25); quando é pessoa nova, o cadastro dele vai antes na fila; `family_id` desconhecido ou de outro domicílio → `family_not_found`; `stale_version` traz o `<household>` atual (`id` = domicílio, Q4); troca de responsável fora do app; a versão nova do cadastro do responsável gravada pela família não volta no `accepted` — o próximo envio desse cadastro recebe `stale_version` e a mescla resolve (Q35);
  - **Citação de pessoa (`patient_client_uuid`):** cita um item `patient` ou um `individual_registration` que criou a pessoa (`new_patient`); o `client_uuid` do cadastro vira o paciente criado por ele (Q26);
  - **`family_client_uuid`** no cadastro individual e na visita, no lugar de `family_id` (no máximo um);
  - **`version` no resultado:** obrigatório no `accepted` de `household`, `family`, `individual_registration` (versão do domicílio para os dois primeiros, do cadastro para o terceiro); no `duplicate`, a gravada naquele recebimento, ausente em recibo anterior ao v1.2.0 (o app marca a linha para recarga, sem mescla; Q5); os outros tipos nunca;
  - **Coordenada (`location`):** limites do §4; `latitude`, `longitude` e `source` obrigatórios; `accuracy_m` obrigatório com `gps` e opcional com `map` (Q3); ausente = mantém (domicílio) ou sem ponto; `null` só no domicílio (apaga; na visita → `invalid_location`); visita só `gps` (`visit_sources`, regra do api: `map` → `invalid_location`); mudar só o ponto gera versão e ficha de atualização; nunca em URL, log ou evento; erro de esquema sob `/body/location` volta `rejected invalid_location` com o mesmo `path`, os demais erros de esquema seguem `invalid_request` (Q2); `multipleOf: 1e-8` no Ajv exige `multipleOfPrecision: 8` (Q49);
  - **Referência:** domínios do cadastro como listas `[{ code, label }]` na ordem do YAML (Q27); `team_professionals` só da equipe, com `has_cns`; `health_units.cnes` `null` = sem CNES (Q6); domínios da atividade num bloco só (Q7);
  - **Códigos HTTP:** `errors.http` lista todo código que o `/field/v1` devolve, inclusive os de borda da cidade (Q8);
  - **Download:** `<household>` com `location` (`null` sem ponto);
  - contagem dos exemplos (a do gerador) e os comandos com `120/120`.
  - **Consumo:** "os cinco arquivos" → "os seis arquivos" (com `reference-response-v1.json`); o app gera os tipos TypeScript pela tag `field-sync-v1.2.0` (contrato §2).

- [ ] **Step 2: `field-sync/CHANGELOG.md`**, no topo: `## field-sync-v1.2.0 — (não publicada até a tag) — MINOR` com uma linha por arquivo do contrato §2 (o mesmo conteúdo do Step 1, em lista), a troca do `invalid-upload-unknown-type.json` (`family` → `visit_sheet`) e "compatível com o v1.1.0: nada do pedido v1.1.0 passa a ser recusado".

- [ ] **Step 3: `README.md` da raiz:** na linha de `field-sync/`, `field-sync-v1.1.0` → `field-sync-v1.2.0` e a descrição ganha "família, coordenadas, mapa e referência offline (ADR 0053)"; no parágrafo `### field-sync`, "os cinco arquivos" → "os seis arquivos".

- [ ] **Step 4: Verificação**

Run: `cd $R && $C/field-sync/tools/check.sh | tail -1 && python3 $C/field-sync/tools/check_domains.py --release && grep -c "device_suspended\|replaced\|family_client_uuid\|reference-response-v1\|visit_sources\|multipleOfPrecision" $C/field-sync/README.md`
Expected: `120/120 casos`; exit 0; contagem ≥ 6.

- [ ] **Step 5: Commit**

```bash
cd $C && $G add field-sync/README.md field-sync/CHANGELOG.md README.md
$G commit -m "docs: document field-sync v1.2.0 rules and changelog" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: SHA-256 ao api e parada

**Files:** nenhum.

**Interfaces:**
- Produces: hash da branch e SHA-256 dos **seis** arquivos para o api copiar em `config/field_sync/` (contrato §2; hoje são cinco).

- [ ] **Step 1: Verificação e SHA-256**

```bash
cd $R && $C/field-sync/tools/check.sh | tail -1 && python3 $C/field-sync/tools/check_domains.py --release
$G -C $C log --oneline origin/main..HEAD
$G -C $C diff --stat origin/main..HEAD -- protocols session clinical exams lab billing events types design-tokens | tail -1
(cd $C/field-sync && shasum -a 256 session-response-v1.json upload-request-v1.json upload-response-v1.json changes-response-v1.json reference-response-v1.json domains-v1.json)
```

Expected: `120/120`; exit 0; três commits; nada fora de `field-sync/` e do `README.md`; seis SHA-256.

- [ ] **Step 2: Pare.** Sem merge nem push. Mande ao coordenador o `HEAD`, o caminho `$C/field-sync/` e os seis SHA-256. O api copia daqui (worktree local); toda mudança pedida pelo api (nova Q, Q50+) volta para as Tasks 1–3 e gera SHA-256 novos.

**Relato:** "contracts-mod24c Task 4: branch em <sha>, 120/120; SHA-256: <lista>; aguardando o api."

---

### Task 5: Entrega — SHA-256 conferidos, merge, tag e push (com autorização)

**Files:** `field-sync/CHANGELOG.md` (data). Merge, tag e push só com autorização.

**Interfaces:**
- Consumes: o api com as cópias presas aos SHA-256 finais (relato do api ao coordenador).
- Produces: `field-sync-v1.2.0` em `origin` (o campo gera os tipos TypeScript a partir desta tag).

- [ ] **Step 1: Verificação final** — o bloco da Task 4, Step 1. Expected: os seis SHA-256 **iguais** aos que o spec do api prende (confirmados pelo coordenador). Diferente: pare.

- [ ] **Step 2: Data no CHANGELOG** (`(não publicada até a tag)` → `<AAAA-MM-DD>`) e commit:

```bash
cd $C && $G add field-sync/CHANGELOG.md
$G commit -m "docs: date the field-sync-v1.2.0 changelog entry" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

- [ ] **Step 3: Pare e peça autorização**, uma etapa por vez: (1) merge em `main`; (2) tag `field-sync-v1.2.0`; (3) push da `main` e da tag. Reporte o hash, `120/120`, os SHA-256 e as decisões aplicadas (Q1–Q10, Q26, Q27, Q35, Q49).

- [ ] **Step 4: Com autorização**

```bash
$G -C $R/contracts fetch origin && $G -C $R/contracts log --oneline main..origin/main
$G -C $R/contracts checkout main && $G -C $R/contracts merge --ff-only feat/mod-24c-field-app
$G -C $R/contracts tag -a field-sync-v1.2.0 -m "field-sync-v1.2.0 — MINOR"
$G -C $R/contracts log --oneline origin/main..main
$G -C $R/contracts push origin main && $G -C $R/contracts push origin field-sync-v1.2.0
$G -C $R/contracts worktree remove $R/.worktrees/contracts-mod24c && $G -C $R/contracts branch -d feat/mod-24c-field-app
```

Expected: `main..origin/main` vazio (senão: rebase, refaça o Step 1); fast-forward; `origin/main..main` só com os commits deste plano; push aceito (bypass do aviso de PR, como nos módulos anteriores).

**Relato:** "contracts-mod24c Task 5: field-sync-v1.2.0 publicada (main <sha>, tag pushada)."

---

### Task 6: Domínio `map-style/` e tag `map-style-v1.0.0` (Q11, Q12, Q18)

Roda **depois da prova técnica do campo** (Task 1 do campo, resultado publicado na seção "Prova técnica do campo" do contrato) e **antes** da Task 15 do campo e da Task 2 do dashboard, que copiam o domínio por lock. Independente das Tasks 0–5 (branch e worktree próprios). Merge, tag e push só com autorização do usuário.

**Files (worktree `$S=$R/.worktrees/contracts-mapstyle`, branch `feat/map-style-v1` a partir de `origin/main`):**
- Create: `map-style/package.json` (privado, `"type": "module"`, `@protomaps/basemaps` com a versão exata da prova, `save-exact`), `map-style/package-lock.json`, `map-style/build.mjs`, `map-style/src/manifest.json`, `map-style/dist/{style-light.json,style-dark.json,manifest.json}`, `map-style/dist/fonts/<pilha>/<faixa>.pbf`, `map-style/dist/fonts/OFL.txt`, `map-style/dist/sprites/{light,dark}{.json,.png,@2x.json,@2x.png}`, `map-style/test/style.test.mjs`, `map-style/README.md`, `map-style/CHANGELOG.md`
- Modify: `README.md` (raiz: linha do domínio na tabela, "Dez domínios" → "Onze domínios"), `.gitignore` (`map-style/node_modules/`)

**Interfaces:**
- Consumes: da seção "Prova técnica do campo" do contrato — a versão de `@protomaps/basemaps`, o commit de `protomaps/basemaps-assets`, as pilhas de fonte e faixas usadas (`Noto Sans Regular`, `Noto Sans Medium`, `Noto Sans Italic`; `0-255`, `256-511`, `8192-8447`) e os sprites; os arquivos de fonte e sprite que a prova guardou no scratchpad (Task 1 do campo, Step 7).
- Produces (tag `map-style-v1.0.0`; os mesmos nomes no campo e no dashboard):
  - `dist/style-<tema>.json` (`light`, `dark`): `version: 8`; `sources.protomaps = { "type": "vector", "url": "__PMTILES__", "attribution": "© OpenStreetMap" }`; `glyphs: "__GLYPHS__/{fontstack}/{range}.pbf"`; `sprite: "__SPRITE__/<tema>"`; `layers` = `layers("protomaps", namedFlavor(<tema>), { lang: "pt" })`. Os três marcadores (`__PMTILES__`, `__GLYPHS__`, `__SPRITE__`) são trocados pelo app que consome (o campo por `pmtiles://file://…` e o caminho local escolhido na prova; o dashboard por `pmtiles://<origem>/territory/map` e `<base>map/fonts`, `<base>map/sprites`).
  - `dist/manifest.json`: `{ "basemaps": "<versão>", "assets": "<commit do basemaps-assets>", "lang": "pt", "themes": ["light", "dark"], "fonts": { "<pilha>": ["0-255", "256-511", "8192-8447"] }, "sprites": ["light", "dark"], "placeholders": ["__PMTILES__", "__GLYPHS__", "__SPRITE__"] }`.
  - `build.mjs`: `build(): { "style-light.json": object, "style-dark.json": object, "manifest.json": object }` puro; só o CLI grava `dist/`. Fontes e sprites não são gerados: são copiados da prova e conferidos pelo teste.
  - **Consumo** (README): o app copia `map-style/dist` da tag para a sua pasta de cópia e grava um lock no formato do `tokens.lock` (`{ "ref": "map-style-v1.0.0", "files": { "<caminho>": "<sha256>" } }`); um teste do app falha se a cópia divergir do lock. O campo copia `light` e `dark`; o dashboard só `light` (Q18).

- [ ] **Step 1: Worktree e insumos da prova**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git; S=$R/.worktrees/contracts-mapstyle
grep -n "Prova técnica do campo" /private/tmp/claude-501/-Users-eduardovrocha-Development-ioit-solutions-rota-saude/fd670630-7d57-4330-811c-0f40fb741325/scratchpad/docs-wt/superpowers/plans/2026-11-13-module-24c-field-app-contracts.md
$G -C $R/contracts fetch -q origin && $G -C $R/contracts tag -l 'map-style-*'
$G -C $R/contracts worktree add $S -b feat/map-style-v1 origin/main
```

Expected: a seção da prova existe, com a versão de `@protomaps/basemaps`, o commit dos assets e as fontes; nenhuma tag `map-style-*`. Sem a seção: **pare** (a prova ainda não terminou).

- [ ] **Step 2: Testes que falham** — `map-style/test/style.test.mjs` (`node:test`, sem dependências):
  - "nenhuma URL `http(s)://` nos dois estilos" (Review Focus 6);
  - "os três marcadores presentes; `sources.protomaps.url === '__PMTILES__'`; atribuição `© OpenStreetMap`";
  - "rótulos em português: toda camada de texto usa `name:pt`";
  - "toda pilha de `text-font` dos dois estilos está em `manifest.fonts`, e cada faixa existe em `dist/fonts/<pilha>/<faixa>.pbf`" (Review Focus 6);
  - "todo sprite de `manifest.sprites` tem `.json`, `.png`, `@2x.json`, `@2x.png`; `fonts/OFL.txt` existe; nenhum arquivo de `dist/` fora do manifesto";
  - "`build()` reproduz `dist/style-*.json` e `dist/manifest.json` byte a byte".

- [ ] **Step 3: Rode e veja falhar** — `cd $S/map-style && npm ci --silent && node --test test/` · Expected: FAIL (`build.mjs` e `dist/` ausentes).

- [ ] **Step 4: Implemente** `build.mjs` e `src/manifest.json`; copie as fontes, os sprites `light`/`dark` e a `OFL.txt` guardados pela prova para `dist/`; rode `node build.mjs`.

- [ ] **Step 5: Rode e veja passar** — `node --test test/` · Expected: PASS. Teste do teste: troque um `text-font` por `"Fonte Ausente"` no estilo gerado, rode, veja a falha da pilha ausente, e desfaça.

- [ ] **Step 6: README e CHANGELOG** — `map-style/README.md` (o que é, marcadores, manifesto, **Consumo** pelo lock, regra "nenhum serviço de mapa de terceiros", licenças, como gerar e validar); `map-style/CHANGELOG.md` com `## map-style-v1.0.0 — (não publicada até a tag)`; `README.md` da raiz: linha `[map-style/](map-style/README.md) | Estilo do mapa de campo (light e dark, pt), glyphs e sprites locais (ADR 0053) | map-style-v1.0.0 | Materializado` e "Onze domínios"; `.gitignore` com `map-style/node_modules/`.

- [ ] **Step 7: Commit**

```bash
cd $S && $G add .gitignore README.md map-style/package.json map-style/package-lock.json map-style/build.mjs map-style/src/manifest.json map-style/dist map-style/test/style.test.mjs map-style/README.md map-style/CHANGELOG.md
$G status --short
$G commit -m "feat: add the map-style domain with local Protomaps styles, glyphs and sprites" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

- [ ] **Step 8: Pare e peça autorização**, uma etapa por vez: (1) data no CHANGELOG e merge em `main`; (2) tag `map-style-v1.0.0`; (3) push da `main` e da tag. Com autorização:

```bash
$G -C $R/contracts fetch origin && $G -C $R/contracts log --oneline main..origin/main
$G -C $R/contracts checkout main && $G -C $R/contracts merge --ff-only feat/map-style-v1
$G -C $R/contracts tag -a map-style-v1.0.0 -m "map-style-v1.0.0"
$G -C $R/contracts log --oneline origin/main..main
$G -C $R/contracts push origin main && $G -C $R/contracts push origin map-style-v1.0.0
$G -C $R/contracts worktree remove $S && $G -C $R/contracts branch -d feat/map-style-v1
```

Expected: `main..origin/main` vazio (a branch do `field-sync` pode ter entrado antes: rebase, refaça o Step 5); fast-forward; `origin/main..main` só com os commits desta task.

**Relato:** "contracts map-style Task 6: map-style-v1.0.0 publicada (main <sha>; basemaps <versão>; fontes <n>, sprites light/dark); campo (Task 15) e dashboard (Task 2) podem copiar."

---

## Self-review (feito ao escrever o plano)

1. **Cobertura do §2:** `session-response-v1.json` (Task 1), `upload-request-v1.json` (Task 1: `family`, `family_client_uuid`, `responsible.patient_client_uuid`, `location`), `upload-response-v1.json` (Task 1: `version`), `changes-response-v1.json` (Task 1), `reference-response-v1.json` (Task 2), `domains-v1.json` (Task 2), exemplos de cada novidade do §2 — família nova com `household_client_uuid` e responsável por `client_uuid` (de item `patient` e de cadastro com `new_patient`, Q26), `accepted` com `version`, `duplicate` sem `version`, `stale_version` de `family`, domicílio e visita com `location`, sessão com e sem `map`, referência, inválidos de `location` (só latitude, fora da faixa, 9 casas, GPS sem precisão) — (Tasks 1–2), README/CHANGELOG (Task 3), SHA-256 ao api antes da tag e tag com autorização (Tasks 4–5). Q11: `map-style/` com tag própria (Task 6).
2. **Placeholders:** valores lidos na execução são só hashes, SHA-256, ids do board, as versões e fontes da prova do campo (Task 6).
3. **Consistência:** `$defs/location` e `$defs/household` iguais por comando; `type: family` no item, no resultado e no `stale_version`; nomes dos exemplos iguais no gerador, no Review Focus e nos relatos. Contagens: 87 → 114 (Task 1) → 120 (Task 2); o texto usa a impressa pelo gerador. Os códigos de `errors.http` (Q8) são os mesmos do `HTTP_ERRORS` do api (Task 2 do api) e das frases do campo (Task 7 do campo); os marcadores do `map-style/` são os mesmos do `buildStyle` do campo e do `fieldMapStyle` do dashboard.
4. **Review Focus:** as seis linhas têm teste nas Tasks 1, 2 e 6.

## Decisões aplicadas (§13 do contrato)

| Código | Q | Onde o plano aplica |
|---|---|---|
| C-1 | Q2 | Architecture e README (Task 3): erro de esquema sob `/body/location` → `rejected invalid_location` com o mesmo `path`, no api (Task 9 do api) |
| C-2 | Q1 | `$defs/family_body` com `responsible` aninhado (Task 1) |
| C-3 | Q3 | `$defs/location` com `if`/`then` (`accuracy_m` obrigatório com `gps`), `invalid-location-gps-without-accuracy.json`, `location.accuracy_m.required_for` em `domains-v1.json` (Tasks 1–2) |
| C-4 | Q4 | `id` do `stale_version` de `family` = domicílio (Task 1) |
| C-5 | Q5 | `version` obrigatório no `accepted` e opcional no `duplicate`; `upload-response-duplicate-without-version.json` (Task 1) |
| C-6 | Q6 | `team_professionals[].has_cns`; `health_units[].cnes` anulável; `invalid-reference-professional-without-has-cns.json` (Task 2) |
| C-7 | Q7 | `collective_activity_domains` num bloco só (Task 2) |
| C-8 | Q8 | `errors.http` + `device_suspended`, `map_not_found`, `range_not_satisfiable`, `unknown_city`, `city_suspended`, `city_schema_behind` (Task 2) |
| C-9 | Q9 | nada a acrescentar em `errors.item` (Task 2) |
| C-10 | Q10 | contracts em F-24.19, F-24.20, F-24.21, F-24.24 (Global Constraints) |
| D-1 (dashboard) | Q11, Q12, Q18 | Task 6: `map-style/`, tag `map-style-v1.0.0`, versão do `@protomaps/basemaps` da prova; o dashboard copia só `light` |
| — | Q17 | README: `device_suspended` cobre `replaced` (Task 3) |
| — | Q25, Q26, Q35 | README e `upload-family-new-from-registration.json` (Tasks 1, 3) |
| — | Q27 | `registration_dictionaries.domains` como listas na ordem do YAML; `invalid-reference-domain-as-object.json` (Task 2) |
| — | Q30, Q34 | README da sessão e da entrega do mapa (Task 3) |
| — | Q49 | Global Constraints e README: Ajv com `multipleOfPrecision: 8` (Task 3) |
