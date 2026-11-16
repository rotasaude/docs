# Módulo 35a — `field-sync` v1.3.0 (repo contracts) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Ordem de entrega (contrato §19):** **contracts v1.3.0 sem tag (Tasks 0–5)** → SHA-256 ao api (Task 6) → api (copia os esquemas daqui, presos por SHA-256) → **contracts: SHA-256 conferidos, tag e push (Task 7, com autorização)** → dashboard e maintenance → campo (gera os tipos pela tag). Nenhuma task daqui depende do api do 35a rodando; o `check.sh` usa o `json_schemer` do container `api` do compose, e a Task 4 lê o `CollectiveActivities::Json.domains` do api da `main`.

**Goal:** Publicar o MINOR `field-sync-v1.3.0` (compatível com o v1.2.0) com o lado de campo do 35a (F-35.10; ADR 0054): `features.vector_control` na sessão; `vector_inspection` na visita; os itens `ovitrap_installation` e `ovitrap_collection` e os seus resultados; `vector_inspection_id` no resultado da visita; `vector` no `<household>`; as entidades `city_block`, `ovitrap`, `liraa_item` e `transmission_block` no download; `vector_control_domains` na referência; a seção `vector_control` e os códigos novos de `errors.item` em `domains-v1.json`; e os dois achados do contracts#2 (Q3): `street_type` em código nos exemplos e o vocabulário de `rules` da atividade na forma que o api serve.

**Architecture:** A mesma mecânica do v1.0.0–v1.2.0, sem ferramenta nova. Os esquemas mudam à mão. Os exemplos saem de `field-sync/tools/make_examples.py` (determinístico) para `examples/` + `manifest.json`. `field-sync/tools/check.sh` valida o manifesto com o `json_schemer` 2.5 do api. `field-sync/tools/check_domains.py` confere os `$defs` repetidos iguais, os `enum` contra `domains-v1.json` e os erros conhecidos, e ganha a seção `vector_control`, as regras da atividade e o logradouro em código. O esquema confere **forma** (tipos, domínios, faixas, objetos fechados, pares um-ou-outro). As **relações** do §3 (situação × desfecho × listas, atividade × imóvel × vínculo, método × produto, contagens entre si, horários, motivos derivados) ficam em dado (`domains-v1.json`, `vector_control.rules`) e são do api, que devolve o código do §17: o app e o dashboard veem o mesmo código. Erro de esquema sob `/body/vector_inspection` o api devolve `rejected vector_inspection_invalid` com o mesmo `path` (CV-4, como a Q2 do 24c para `location`). `$defs/household` continua idêntico na resposta do envio e no download, e por isso o `vector` entra nos dois.

**Tech Stack:** JSON Schema 2020-12; `json_schemer` 2.5 (container `api`); `python3` do host; `shasum`; git em `/opt/homebrew/bin/git`.

**Spec:** `docs/superpowers/specs/2026-11-14-module-35-zoonoses-endemics-design.md` (decisões 1–7, coordenador 23–30); ADR `docs/adr/0054.md`. **Contrato (fonte única):** `docs/superpowers/plans/2026-11-16-module-35a-vector-control-contracts.md` §3, §6, §9, §13, §17 e §20 (Q3 prevalece). Base: `field-sync-v1.2.0` (contracts `f9d6df2`). Molde: `docs/superpowers/plans/2026-11-13-module-24c-field-app-contracts-repo.md` (e o do 24b, `2026-11-12-module-24b-home-visit-contracts-repo.md`). Issue: rotasaude/contracts#2 (os dois achados do 24c; os seis itens adiados **continuam adiados**, Q3, embora o último comentário da issue diga "junto dos seis"). As diferenças com o contrato estão em **"Divergências propostas ao contrato"** (CV-1 a CV-14), no fim; todas foram **decididas** no §20 do contrato (Q6, Q7, Q9, Q10, Q11 e Q13; docs `c2d755c`), e o plano aplica as decisões, inclusive o que veio dos outros planos: `sample_label_taken` e as chaves de `limits` (Q7), os `path` da Task 2 do plano do dashboard como fonte única (Q9) e a semana epidemiológica como `"YYYY-WW"` (Q10).

## Global Constraints

- Repo `contracts` na raiz do monorepo (`$R/contracts`, remote `git@github.com:rotasaude/contracts.git`), `origin/main` em `f9d6df2` ou mais novo; tags `field-sync-v1.0.0`, `-v1.1.0` e `-v1.2.0` existem; nenhuma `field-sync-v1.3.0`.
- Tag `field-sync-v1.3.0`, anotada, mensagem `field-sync-v1.3.0 — MINOR`. `session.contract` continua `"field-sync-v1"` (const). Sem entrada no CHANGELOG, não há tag.
- **Compatível com o v1.2.0** (contrato §13): nada que o v1.2.0 aceitava no **pedido** passa a ser recusado (`vector_inspection` e os itens de ovitrampa são novos e opcionais); nas **respostas**, nada novo é obrigatório (`features.vector_control`, `vector`, `vector_inspection_id` e `vector_control_domains` são opcionais). A única resposta que aperta é `collective_activity_domains.rules` da referência, que o api já serve nessa forma (Q3). Todo exemplo válido do v1.2.0 continua válido.
- Convenções de arquivo: `$id` = `https://rota-saude/schemas/field-sync/<arquivo>`, `title` = `rotasaude.field_sync.<nome>.v1`; fechado em todo nível (`additionalProperties: false`), salvo as seções de dicionário (K-2 do 24b) e os blocos `home_visit_domains`/`collective_activity_domains` da referência fora do `rules`.
- Valores do contrato copiados sem mudança: atividades `li` `li_t` `pe` `ie` `transmission_block` `liraa`; `work_type` `routine` `pending`; situações `worked` `closed` `refused` `vacant`; grupos `A1` `A2` `B` `C` `D1` `D2` `E`; métodos `mechanical` `focal` `perifocal` `spatial`; risco `green` `yellow` `red` `gray` `purple`; condição `ok` `damaged` `lost` `dry`; ovitrampa `active` `removed` `lost`; contagens 0–999; cargas > 0, ≤ 999, duas casas; amostra `number` 1–99 e larvas 1–10; ovos 0–9999; quarteirão `^[0-9A-Za-z-]{1,10}$`; ciclo de UBV `number` 1–3; unidade do produto `g` `ml` `un`; `uses` ⊆ `focal` `perifocal` `spatial`, não vazio; cadência 7–365 dias.
- Semana epidemiológica (Q10): onde aparecer no `field-sync`, é texto `"YYYY-WW"` (`^[0-9]{4}-[0-9]{2}$`), nunca o inteiro `YYYYWW` do banco. Nenhum esquema da v1.3.0 a expõe: o corpo `ovitrap` do §13 não tem semana, e o plano não a acrescenta.
- `limits` (Q7(3)): `deposit_count_max` 999, `charges_max` 999, `samples_max` 99, `larvae_collected_max` 10, `measures_max` 20; `deposits` com no máximo 7 itens (um por grupo).
- LGPD (contrato §17): nada do 35a no download ou na referência leva nome, CPF, CNS, telefone, endereço de pessoa, caso ou notificação. `transmission_block` nunca leva caso; `ovitrap` nunca leva morador. Os objetos fechados são a guarda, e há exemplo inválido para isso.
- Dados dos exemplos fictícios, como no v1. `street_type` sempre em código DNE (`"081"` = RUA), nunca o rótulo (Q3).
- `proof` de `domains-v1.json` não muda (`fcf23c87`): a conferência de `vector_control` contra o api é o `domains_spec` do api do 35a (CV-14).
- Worktree **fora de `contracts/`**: `$R/.worktrees/contracts-mod35a`, branch `feat/mod-35a-vector-control` a partir de `origin/main` (contrato, valores fixos). Comandos a partir de `$R`, com o compose de pé (`docker compose ps api` rodando).
- Testes: este repo não tem `bin/test`; a verificação é `check.sh` + `check_domains.py`. Nada aqui roda `rspec` nem `vitest`.
- Git: `/opt/homebrew/bin/git`. Nunca `git add -A` nem `git add .`. Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Merge, tag, push e comentário na issue só com autorização explícita do usuário, uma etapa por vez; antes do push, `origin/main..main` só com os commits desta entrega.
- Board #1: o contracts entra em F-35.10 (contracts, api, campo). O card está em `Not Started`; quem o move para In Progress é o plano do api (a Task 0 daqui só confere).
- Execução coordenada: ao fim de cada task, uma linha à sessão "Próximos módulos" (`SendMessage`).

## Ambiente de execução

- `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `G=/opt/homebrew/bin/git`, `C=$R/.worktrees/contracts-mod35a`.
- Caminhos das tasks relativos a `$C`.
- Verificação (de `$R`): `$C/field-sync/tools/check.sh [esquema]`, `python3 $C/field-sync/tools/check_domains.py [--release]`, `python3 $C/field-sync/tools/make_examples.py` (regera e imprime a contagem por suíte; o número que vale é sempre o impresso).

## Review Focus

1. **Caso ou pessoa no download do 35a.** Um `transmission_block` com `cases`, notificação ou paciente, ou uma `ovitrap` com morador, passaria para o aparelho e quebraria o §17. Teste: `invalid-changes-transmission-block-with-case.json` (`/items/4/body/cases schema`); o bloqueio `proposed` também não pode sair (`invalid-changes-transmission-block-proposed.json`) (Task 2).
2. **Derivado ou contagem vindos do app.** `recovered`, a etiqueta da amostra ou `egg_count` no corpo seriam uma segunda fonte para o que o api calcula, e a contagem de ovos não é do app (§14). Testes: `invalid-vector-inspection-recovered-in-body.json`, `invalid-ovitrap-collection-egg-count.json` (Task 1).
3. **Carga com 3 casas, carga 0, contagem 1000, 11 larvas, 8 depósitos, hora sem fuso.** O aparelho gravaria e o api recusaria tarde, ou a baixa de estoque sairia com quantidade inventada. Testes: `invalid-vector-inspection-charges-three-decimals.json`, `-charges-zero`, `-deposit-1000`, `-larvae-11`, `-8-deposits`, `-entered-at-without-offset` (Task 1). Para o Ajv do app, `multipleOf: 0.01` precisa de `multipleOfPrecision` (o app já usa 8, Q49; README).
4. **Compatibilidade com o v1.2.0.** A sessão sem `vector_control`, o `<household>` sem `vector` e todos os 49 válidos antigos continuam válidos, e o `invalid-upload-unknown-type.json` (`visit_sheet`) continua desconhecido. Teste: a contagem de válidos sobe de 49 para 61 sem nenhum antigo virar `FAIL` (Tasks 1–4). O app v1.2.0 **não** ignora tipo desconhecido no download (CV-1): isso não se testa aqui, e por isso a regra de ligar o interruptor vai para o README (Task 5).
5. **Domínio do esquema diferente do `domains-v1.json`**, ou vocabulário antigo das regras da atividade servido na referência. O app montaria o boletim com um código que o esquema recusa, ou leria `by_activity_type`, que o api não manda. Testes: `check_domains.py` com os pares `vector_control.*` (teste do verificador: troque um código, veja `DOMINIOS DIFERENTES`) (Task 3); `invalid-reference-activity-rules-old-vocabulary.json` e `invalid-reference-activity-by-type-old-value.json` (Task 4).

## Mapa de arquivos

| Arquivo (worktree `$C`) | Mudança | Task |
|---|---|---|
| `field-sync/session-response-v1.json` | `features.vector_control` (opcional) | 1 |
| `field-sync/upload-request-v1.json` | `home_visit_body.vector_inspection` + `$defs` do boletim; itens `ovitrap_installation`, `ovitrap_collection` | 1 |
| `field-sync/upload-response-v1.json` | tipos novos; `vector_inspection_id`; `$defs/household` + `vector` | 1, 2 |
| `field-sync/changes-response-v1.json` | `$defs/household` + `vector`; tipos `city_block`, `ovitrap`, `liraa_item`, `transmission_block` | 2 |
| `field-sync/reference-response-v1.json` | `vector_control_domains` (3); `collective_activity_domains.rules` fechado (4) | 3, 4 |
| `field-sync/domains-v1.json` | `vector_control`; `errors.item` (3); `collective_activity.rules` no vocabulário do api (4) | 3, 4 |
| `field-sync/tools/check_domains.py` | pares `vector_control.*` (3); regras da atividade e logradouro em código (4) | 3, 4 |
| `field-sync/tools/make_examples.py`, `field-sync/examples/*` | exemplos novos; `ADDRESS.street_type = "081"` | 1–4 |
| `field-sync/README.md`, `field-sync/CHANGELOG.md`, `README.md` (raiz) | regras, entrada v1.3.0, versão na tabela | 5 |
| — | SHA-256 ao api e parada | 6 |
| — | verificação final, merge, tag, push, comentário no contracts#2 (com autorização) | 7 |

**Estratégia de teste:** cada mudança de esquema entra com seus exemplos válidos e inválidos no gerador **antes** do esquema (Step 1 de cada task). O `check.sh` falha nos novos com o esquema velho e passa com o novo. O `check_domains.py` ganha os pares antes dos dados (Task 3) e as guardas antes da correção (Task 4). Contagens: hoje 124 (49 válidos); ao fim da Task 1, 154; da Task 2, 161; da Task 3, 163; da Task 4, **165 casos (61 válidos, 104 inválidos)**. O número exato é o que o `make_examples.py` imprime, e o README o repete.

---

### Task 0: Base, card, decisões e worktree

**Files:** nenhum.

**Interfaces:**
- Consumes: `contracts` `origin/main`; board #1 (`PVT_kwDOEbfGRc4BbxiB`); o §20 do contrato.
- Produces: `$C` em `feat/mod-35a-vector-control`; a base anotada.

- [ ] **Step 1: Card** (um `gh` simples por chamada)

```bash
gh project item-list 1 --owner rotasaude --limit 800 --format json --jq '.items[] | select(.title | test("^F-35\\.10 ")) | {id, title, status}'
```

Expected: o card existe (`Not Started` ou `In Progress`). Ausente: **pare e reporte**. Não mova.

- [ ] **Step 2: Decisões.** Confira que o §20 do contrato tem as linhas Q6–Q13 (docs `c2d755c` ou mais novo) e que nenhuma Q posterior mudou o que este plano aplica (CV-1 a CV-14 e os itens de Q7, Q9 e Q10). Se uma Q nova mudou algo, ajuste a task indicada **antes** de começar e reporte ao coordenador.

- [ ] **Step 3: Base e worktree**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/contracts fetch -q origin
$G -C $R/contracts log --oneline -1 origin/main
$G -C $R/contracts tag -l 'field-sync-*'
$G -C $R/contracts worktree add $R/.worktrees/contracts-mod35a -b feat/mod-35a-vector-control origin/main
cd $R && $R/.worktrees/contracts-mod35a/field-sync/tools/check.sh | tail -1 && python3 $R/.worktrees/contracts-mod35a/field-sync/tools/check_domains.py
for f in $R/.worktrees/contracts-mod35a/field-sync/*-v1.json; do cmp -s $f $R/apps/api/config/field_sync/$(basename $f) && echo "igual $(basename $f)" || echo "DIFERENTE $(basename $f)"; done
```

Expected: `f9d6df2 docs: date the field-sync-v1.2.0 changelog entry` ou mais novo; tags `v1.0.0`, `v1.1.0` e `v1.2.0`, sem `v1.3.0`; `124/124 casos`; `defs-iguais`, `dominios-iguais`, `erros-conhecidos`, `prova: fcf23c87 (2026-11-12)`; os seis arquivos `igual` à cópia do api. Diferente: **pare e reporte** (outro módulo mexeu no `field-sync/`).

**Relato:** "contracts-mod35a Task 0: worktree em feat/mod-35a-vector-control sobre <sha>; 124/124; F-35.10 conferido; CV-1..14 conferidas em Q6/Q7/Q9/Q10/Q11/Q13."

---

### Task 1: Sessão, boletim na visita e itens de ovitrampa

**Files (worktree `$C`):**
- Modify: `field-sync/session-response-v1.json`, `field-sync/upload-request-v1.json`, `field-sync/upload-response-v1.json`
- Modify: `field-sync/tools/make_examples.py`; regenerate `field-sync/examples/`

**Interfaces:**
- Consumes: os esquemas do v1.2.0; contrato §3 (boletim), §6 e §13 (itens), §9 (etiqueta, só README).
- Produces (`field-sync-v1.3.0`):
  - `session-response-v1.json`: `features.properties.vector_control = { "type": "boolean", "description": "Vigilância ambiental de endemias (35a). Opcional: ausente = false (api anterior ao 35a)." }`; `features.required` **não muda**.
  - `upload-request-v1.json`:
    - `item.properties.type.enum` + `"ovitrap_installation"`, `"ovitrap_collection"` (no fim); `item.allOf` + um ramo `if type const → then body $ref` para cada um;
    - `home_visit_body.properties.vector_inspection = { "$ref": "#/$defs/vector_inspection" }` (ausente = sem boletim; `null` não é aceito), e a descrição da visita cita o boletim;
    - `$defs` novos:
      ```json
      "vector_inspection": { "type": "object", "additionalProperties": false,
        "required": ["activity", "work_type", "situation", "educational_action"],
        "properties": {
          "activity": { "enum": ["li", "li_t", "pe", "ie", "transmission_block", "liraa"] },
          "work_type": { "enum": ["routine", "pending"] },
          "situation": { "enum": ["worked", "closed", "refused", "vacant"] },
          "educational_action": { "type": "boolean" },
          "entered_at": { "$ref": "#/$defs/datetime" }, "left_at": { "$ref": "#/$defs/datetime" },
          "city_block_id": { "$ref": "#/$defs/uuid" },
          "risk_class": { "enum": ["green", "yellow", "red", "gray", "purple"] },
          "liraa_draw_item_id": { "$ref": "#/$defs/uuid" }, "transmission_block_id": { "$ref": "#/$defs/uuid" },
          "deposits": { "type": "array", "maxItems": 7, "items": { "$ref": "#/$defs/vector_deposit" } },
          "measures": { "type": "array", "maxItems": 20, "items": { "$ref": "#/$defs/vector_measure" } },
          "samples": { "type": "array", "maxItems": 99, "items": { "$ref": "#/$defs/vector_sample" } } } },
      "deposit_group": { "enum": ["A1", "A2", "B", "C", "D1", "D2", "E"] },
      "deposit_count": { "type": "integer", "minimum": 0, "maximum": 999 },
      "vector_deposit": { "type": "object", "additionalProperties": false,
        "required": ["group", "existing", "with_water", "with_larvae", "eliminated", "treated"],
        "properties": { "group": { "$ref": "#/$defs/deposit_group" }, "existing": { "$ref": "#/$defs/deposit_count" },
          "with_water": { "$ref": "#/$defs/deposit_count" }, "with_larvae": { "$ref": "#/$defs/deposit_count" },
          "eliminated": { "$ref": "#/$defs/deposit_count" }, "treated": { "$ref": "#/$defs/deposit_count" } } },
      "vector_measure": { "type": "object", "additionalProperties": false, "required": ["method"],
        "properties": { "method": { "enum": ["mechanical", "focal", "perifocal", "spatial"] },
          "vector_product_id": { "$ref": "#/$defs/uuid" },
          "charges": { "type": "number", "exclusiveMinimum": 0, "maximum": 999, "multipleOf": 0.01 } } },
      "vector_sample": { "type": "object", "additionalProperties": false, "required": ["number", "deposit_group", "larvae_collected"],
        "properties": { "number": { "type": "integer", "minimum": 1, "maximum": 99 },
          "deposit_group": { "$ref": "#/$defs/deposit_group" },
          "larvae_collected": { "type": "integer", "minimum": 1, "maximum": 10 } } },
      "ovitrap_number": { "type": "string", "pattern": "^[0-9A-Za-z-]{1,12}$" },
      "ovitrap_installation_body": { "type": "object", "additionalProperties": false,
        "required": ["number", "consent", "installed_on"],
        "properties": { "household_id": { "$ref": "#/$defs/uuid" }, "household_client_uuid": { "$ref": "#/$defs/uuid" },
          "number": { "$ref": "#/$defs/ovitrap_number" }, "consent": { "type": "boolean" },
          "installed_on": { "$ref": "#/$defs/date" } },
        "oneOf": [{ "required": ["household_id"] }, { "required": ["household_client_uuid"] }] },
      "ovitrap_collection_body": { "type": "object", "additionalProperties": false,
        "required": ["collected_on", "condition"],
        "properties": { "ovitrap_id": { "$ref": "#/$defs/uuid" }, "ovitrap_client_uuid": { "$ref": "#/$defs/uuid" },
          "collected_on": { "$ref": "#/$defs/date" }, "condition": { "enum": ["ok", "damaged", "lost", "dry"] },
          "reinstall": { "type": "boolean" }, "remove": { "type": "boolean" } },
        "oneOf": [{ "required": ["ovitrap_id"] }, { "required": ["ovitrap_client_uuid"] }] }
      ```
    - Descrições curtas, sem repetir o README: no boletim, "relações do §3 são do api (domains-v1.json, vector_control.rules); listas ausentes = vazias; erro de esquema aqui dentro → rejected vector_inspection_invalid com o mesmo path (CV-4)"; em `consent`, "false → rejected consent_required (regra do api, CV-9)"; em `reinstall`/`remove`, "padrão true/false; com remove true, reinstall é ignorado (CV-9)".
  - `upload-response-v1.json`:
    - `result.properties.type.enum` + `"ovitrap_installation"`, `"ovitrap_collection"` (antes do `null`);
    - `result.properties.vector_inspection_id = { "$ref": "#/$defs/uuid", "description": "Só no accepted/duplicate de home_visit com boletim (opcional)." }`;
    - `result.allOf` + `{ "if": { "required": ["status", "type"], "properties": { "status": { "enum": ["accepted", "duplicate"] }, "type": { "const": "home_visit" } } }, "else": { "properties": { "vector_inspection_id": false } } }`;
    - **sem `ovitrap_id`** (CV-3): o `id` do `accepted`/`duplicate` de `ovitrap_installation` e de `ovitrap_collection` é o id da **ovitrampa**; a descrição de `result.id` diz isso. Os ramos de `version` do v1.2.0 já proíbem `version` nos tipos novos.

- [ ] **Step 1: Exemplos primeiro (gerador)**

Em `make_examples.py`:
- Constantes novas (uuid fixos): `BLOCK = "a1b2c3d4-e5f6-4a7b-8c9d-0e1f2a3b4c5d"`, `BLOCK2 = "b2c3d4e5-f6a7-4b8c-9d0e-1f2a3b4c5d6e"`, `OVI = "c3d4e5f6-a7b8-4c9d-8e0f-2a3b4c5d6e7f"`, `OVI2 = "d5e6f7a8-b9c0-4d1e-8f2a-3b4c5d6e7f80"`, `LIRAA_ITEM = "d4e5f6a7-b8c9-4d0e-9f1a-3b4c5d6e7f8a"`, `SURVEY = "e5f6a7b8-c9d0-4e1f-8a2b-4c5d6e7f8a9b"`, `TBLOCK = "f6a7b8c9-d0e1-4f2a-9b3c-5d6e7f8a9b0c"`, `PRODUCT = "0a1b2c3d-4e5f-4a6b-8c7d-9e0f1a2b3c4d"`, `PRODUCT2 = "1b2c3d4e-5f6a-4b7c-9d8e-0f1a2b3c4d5e"`, `VINSP = "2c3d4e5f-6a7b-4c8d-8e9f-1a2b3c4d5e6f"`; `VI = B + ["vector_inspection"]`.
- `vi_worked()`: `{"activity": "li_t", "work_type": "routine", "situation": "worked", "educational_action": True, "entered_at": "2026-11-12T09:30:00-03:00", "left_at": "2026-11-12T09:52:00-03:00", "risk_class": "yellow", "deposits": [{"group": "A1", "existing": 3, "with_water": 2, "with_larvae": 1, "eliminated": 1, "treated": 1}, {"group": "D2", "existing": 4, "with_water": 2, "with_larvae": 1, "eliminated": 4, "treated": 0}], "measures": [{"method": "mechanical"}, {"method": "focal", "vector_product_id": PRODUCT, "charges": 2.25}], "samples": [{"number": 1, "deposit_group": "A1", "larvae_collected": 8}, {"number": 2, "deposit_group": "D2", "larvae_collected": 5}]}` (coerente com as regras 5–8 do §3; `occurred_at` 09:40 fica entre a entrada e a saída).
- `hv_ace()`: `{"team_id": TEAM, "occurred_at": "2026-11-12T09:40:00-03:00", "household_id": HH, "subject": "family_responsible", "family_id": FAM, "outcome": 1, "reasons": [], "shared": False, "vector_inspection": vi_worked()}` (os motivos 34–37 saem do boletim, regra 9).
- `upload_vi()` = `up(item(60, "home_visit", hv_ace()))`; `upload_ovitrap_installation()` = `up(item(63, "ovitrap_installation", {"household_id": HH, "number": "OVT-014", "consent": True, "installed_on": "2026-11-12"}))`; `upload_ovitrap_collection()` = `up(item(66, "ovitrap_collection", {"ovitrap_id": OVI, "collected_on": "2026-11-12", "condition": "ok", "reinstall": True, "remove": False}), item(67, "ovitrap_collection", {"ovitrap_id": OVI2, "collected_on": "2026-11-12", "condition": "lost", "remove": True}))`.
- `resp_vector_rejected()`: um `rejected` por código novo de item (os 20 do §17 e o `sample_label_taken` da Q7) e três herdados, `client_uuid` `cu(70)` em diante. Os `path` são os da Task 2 do plano do dashboard (Q9), e onde ele não fixa, os da CV-8.

| `type` | `error` | `path` |
|---|---|---|
| home_visit | `cbo_not_allowed` | `vector_inspection` |
| home_visit | `vector_inspection_invalid` | `vector_inspection.liraa_draw_item_id` |
| home_visit | `situation_outcome_mismatch` | `vector_inspection.situation` |
| home_visit | `inspection_not_worked` | `vector_inspection.deposits` |
| home_visit | `activity_property_mismatch` | `vector_inspection.activity` |
| home_visit | `liraa_item_not_found` | `vector_inspection.liraa_draw_item_id` |
| home_visit | `transmission_block_not_active` | `vector_inspection.transmission_block_id` |
| home_visit | `deposit_counts_inconsistent` | `vector_inspection.deposits[0].with_larvae` |
| home_visit | `product_not_allowed` | `vector_inspection.measures[1].vector_product_id` |
| home_visit | `samples_exceed_larvae` | `vector_inspection.samples[1]` |
| home_visit | `sample_label_taken` | `vector_inspection.samples[0]` |
| home_visit | `inspection_times_invalid` | `vector_inspection.entered_at` |
| home_visit | `reason_derived` | `reasons` |
| home_visit | `block_already_set` | `vector_inspection.city_block_id` |
| home_visit | `household_without_neighborhood` | `vector_inspection.city_block_id` |
| home_visit | `block_not_in_household_neighborhood` | `vector_inspection.city_block_id` |
| ovitrap_installation | `cbo_not_allowed` | — |
| ovitrap_installation | `household_without_block` | `household_id` |
| ovitrap_installation | `consent_required` | `consent` |
| ovitrap_installation | `ovitrap_number_taken` | `number` |
| ovitrap_collection | `not_found` | `ovitrap_client_uuid` |
| ovitrap_collection | `collection_out_of_range` | `collected_on` |
| ovitrap_collection | `reading_not_open` | `ovitrap_id` |
| ovitrap_collection | `ovitrap_not_active` | `ovitrap_id` |

- Novos (arquivo → `expect`):

| Suíte | Arquivo | Conteúdo | `expect` |
|---|---|---|---|
| session | `session-endemic-agent.json` | `session_agent()` (CBO 515140) com `features.vector_control: True` | `valid` |
| session | `invalid-session-vector-control-string.json` | `features.vector_control = "true"` | `/features/vector_control boolean` |
| upload-request | `upload-home-visit-vector-inspection.json` | `upload_vi()` | `valid` |
| upload-request | `upload-home-visit-vector-inspection-pe.json` | `item(61)`: `household_id` HH2, `subject` none, `outcome` 1, `reasons` [], `occurred_at` 14:05; boletim `pe`/`routine`/`worked`, `educational_action` False, entrada 13:50 e saída 14:20, `city_block_id` BLOCK, um depósito D1 (12/5/0/0/5), medida `perifocal` PRODUCT2 `charges` 0.5, **sem `samples`** (ausente = vazia) | `valid` |
| upload-request | `upload-home-visit-vector-inspection-closed.json` | `item(62)`: HH, `subject` none, `outcome` 3, `reasons` []; boletim `liraa`/`routine`/`closed`, `educational_action` False, entrada 10:14, saída 10:16, `liraa_draw_item_id` LIRAA_ITEM, **sem listas** | `valid` |
| upload-request | `upload-ovitrap-installation.json` | `upload_ovitrap_installation()` | `valid` |
| upload-request | `upload-ovitrap-same-batch.json` | instalação `cu(64)` (`household_id` HH, `"OVT-015"`) + coleta `cu(65)` com `ovitrap_client_uuid` `cu(64)`, `collected_on` "2026-11-17", `ok` | `valid` |
| upload-request | `upload-ovitrap-collection.json` | `upload_ovitrap_collection()` | `valid` |
| upload-request | `invalid-vector-inspection-null.json` | `vector_inspection: None` | `/items/0/body/vector_inspection object` |
| upload-request | `invalid-vector-inspection-activity-unknown.json` | `activity = "lit"` | `/items/0/body/vector_inspection/activity enum` |
| upload-request | `invalid-vector-inspection-recovered-in-body.json` | `recovered = True` | `/items/0/body/vector_inspection/recovered schema` |
| upload-request | `invalid-vector-inspection-entered-at-without-offset.json` | `entered_at = "2026-11-12T09:30:00"` | `/items/0/body/vector_inspection/entered_at pattern` |
| upload-request | `invalid-vector-inspection-risk-class-blue.json` | `risk_class = "blue"` | `/items/0/body/vector_inspection/risk_class enum` |
| upload-request | `invalid-vector-inspection-deposit-group-f.json` | `deposits[0].group = "F"` | `/items/0/body/vector_inspection/deposits/0/group enum` |
| upload-request | `invalid-vector-inspection-deposit-1000.json` | `deposits[0].existing = 1000` | `/items/0/body/vector_inspection/deposits/0/existing maximum` |
| upload-request | `invalid-vector-inspection-8-deposits.json` | os 7 grupos + um A1 repetido (todas as contagens 0) | `/items/0/body/vector_inspection/deposits maxItems` |
| upload-request | `invalid-vector-inspection-charges-three-decimals.json` | `measures[1].charges = 1.234` | `/items/0/body/vector_inspection/measures/1/charges multipleOf` |
| upload-request | `invalid-vector-inspection-charges-zero.json` | `measures[1].charges = 0` | `/items/0/body/vector_inspection/measures/1/charges exclusiveMinimum` |
| upload-request | `invalid-vector-inspection-larvae-11.json` | `samples[0].larvae_collected = 11` | `/items/0/body/vector_inspection/samples/0/larvae_collected maximum` |
| upload-request | `invalid-ovitrap-installation-two-households.json` | instalação + `household_client_uuid` `cu(30)` | `/items/0/body oneOf` |
| upload-request | `invalid-ovitrap-installation-number-13-chars.json` | `number = "OVT-012345678"` | `/items/0/body/number pattern` |
| upload-request | `invalid-ovitrap-installation-without-consent.json` | sem `consent` | `/items/0/body required` |
| upload-request | `invalid-ovitrap-collection-two-ovitraps.json` | coleta `cu(66)` + `ovitrap_client_uuid` `cu(64)` | `/items/0/body oneOf` |
| upload-request | `invalid-ovitrap-collection-condition-wet.json` | `condition = "wet"` | `/items/0/body/condition enum` |
| upload-request | `invalid-ovitrap-collection-egg-count.json` | coleta + `egg_count: 12` | `/items/0/body/egg_count schema` |
| upload-response | `upload-response-vector-accepted.json` | `accepted` home_visit VISIT com `vector_inspection_id` VINSP; `accepted` ovitrap_installation `id` OVI; `accepted` ovitrap_collection `id` OVI; `duplicate` home_visit VISIT sem `vector_inspection_id` | `valid` |
| upload-response | `upload-response-vector-rejected.json` | `resp_vector_rejected()` | `valid` |
| upload-response | `invalid-response-vector-inspection-id-on-patient.json` | `accepted` patient PAT2 com `vector_inspection_id` VINSP | `/results/0/vector_inspection_id schema` |
| upload-response | `invalid-response-vector-inspection-id-on-rejected.json` | `resp_vector_rejected()` com `results[0].vector_inspection_id` VINSP | `/results/0/vector_inspection_id schema` |
| upload-response | `invalid-response-ovitrap-id.json` | `accepted` ovitrap_installation OVI + `ovitrap_id` OVI (CV-3) | `/results/0/ovitrap_id schema` |

Rode `python3 $C/field-sync/tools/make_examples.py`. Expected: `session-response-v1.json 10 · upload-request-v1.json 79 · upload-response-v1.json 41 · changes-response-v1.json 18 · reference-response-v1.json 6 · total 154`.

- [ ] **Step 2: Rode e veja falhar**

Run: `cd $R && $C/field-sync/tools/check.sh | grep '^FAIL' | wc -l`
Expected: os válidos novos falham (`schema` em `vector_control`, `vector_inspection`, `vector_inspection_id` e `enum` no `type`), e os inválidos novos falham por outro motivo (`schema` ou `enum` no lugar do `expect`). Nenhum dos 124 antigos falha.

- [ ] **Step 3: Mude os três esquemas** pelas Interfaces.

- [ ] **Step 4: Rode e veja passar**

Run: `cd $R && $C/field-sync/tools/check.sh | tail -1 && python3 $C/field-sync/tools/check_domains.py`
Expected: `154/154 casos`; `defs-iguais`; `dominios-iguais`; `ERROS FORA DA LISTA: upload-response-vector-rejected.json: vector_inspection_invalid; …` com os 21 códigos novos (exit 1: a Task 3 os acrescenta, e esse é o teste que falha dela). Se um `expect` sair com outro `type` do `json_schemer`, ajuste só o `expect` e anote no relato.

- [ ] **Step 5: Commit**

```bash
cd $C && $G add field-sync/session-response-v1.json field-sync/upload-request-v1.json field-sync/upload-response-v1.json field-sync/tools/make_examples.py field-sync/examples
$G status --short
$G commit -m "feat: add the vector inspection and the ovitrap items to field sync uploads" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato:** "contracts-mod35a Task 1: features.vector_control, vector_inspection, ovitrap_installation/collection e vector_inspection_id (id da ovitrampa no resultado, CV-3); 154/154; códigos novos entram na Task 3."

---

### Task 2: Perfil do imóvel e entidades novas no download

**Files (worktree `$C`):**
- Modify: `field-sync/changes-response-v1.json`, `field-sync/upload-response-v1.json`
- Modify: `field-sync/tools/make_examples.py`; regenerate `field-sync/examples/`

**Interfaces:**
- Consumes: contrato §13 (entidades do download), §2 (risco), §6 (ovitrampa), §7 (itens do LIRAa), §8 (bloqueio).
- Produces:
  - Nos **dois** arquivos, idênticos (o `check_domains.py` cobra): `$defs/household.properties.vector = { "anyOf": [{ "$ref": "#/$defs/household_vector" }, { "type": "null" }], "description": "Perfil de vigilância (35a), só leitura, na forma de GET /vector_control/households/:id. Acrescentado pelo FieldSync; o TerritoryRegistry::Json.household do 24a não muda (Q8). Ausente = gate vector_control não utilizável; null = sem perfil (Q6, CV-10). Nunca vai no item household." }` (fora do `required`) e
    ```json
    "household_vector": { "type": "object", "additionalProperties": false,
      "required": ["city_block_id", "risk_class", "risk_assessed_on", "last_worked_on", "last_situation"],
      "properties": { "city_block_id": { "$ref": "#/$defs/uuid_or_null" },
        "risk_class": { "enum": ["green", "yellow", "red", "gray", "purple", null] },
        "risk_assessed_on": { "$ref": "#/$defs/date_or_null" }, "last_worked_on": { "$ref": "#/$defs/date_or_null" },
        "last_situation": { "enum": ["worked", "closed", "refused", "vacant", null] } } }
    ```
  - `changes-response-v1.json`:
    - `change_item.properties.type.enum` + `"city_block"`, `"ovitrap"`, `"liraa_item"`, `"transmission_block"`; `change_item.allOf` + quatro ramos `{ "if": { type const X }, "then": { "required": ["version"], "properties": { "body": { "$ref": "#/$defs/X" } } } }`;
    - `$defs/ovitrap_number`: **byte a byte** o da Task 1;
    - `$defs` novos, todos fechados e com todas as chaves em `required`:
      - `city_block`: `id` uuid, `neighborhood_id` uuid, `number` `{ "type": "string", "pattern": "^[0-9A-Za-z-]{1,10}$" }`, `active` boolean, `neighbor_ids` `{ "type": "array", "uniqueItems": true, "items": uuid }`, `version` integer ≥ 1;
      - `ovitrap`: `id`, `number` (`ovitrap_number`), `household_id` uuid, `status` `{ "enum": ["active", "removed", "lost"] }`, `open_reading` `anyOf [ { fechado, required ["installed_on"], installed_on date }, null ]`, `last_reading` `anyOf [ { fechado, required ["collected_on", "condition", "egg_count"], collected_on date, condition enum ["ok", "damaged", "lost", "dry"], egg_count { "type": ["integer", "null"], "minimum": 0, "maximum": 9999 } }, null ]`, `version`;
      - `liraa_item`: `id`, `survey_id` uuid, `survey_name` `{ "type": "string", "minLength": 1, "maxLength": 80 }`, `household_id` uuid, `city_block_id` uuid, `status` `{ "enum": ["pending", "inspected", "substituted", "closed"] }` (`closed` = o levantamento fechou), `version`;
      - `transmission_block`: `id`, `status` `{ "enum": ["active", "closed"] }` (proposta e descarte nunca saem), `block_ids` `{ "type": "array", "minItems": 1, "uniqueItems": true, "items": uuid }`, `cycles` `{ "type": "array", "maxItems": 3, "items": { fechado, required ["number", "starts_on", "ends_on"], number integer 1–3, starts_on date, ends_on date } }`, `version`. A descrição diz "nunca leva caso, notificação nem paciente (§8, §17)".
    - `removal` não muda (o fim vem pelo `status`, §13).

- [ ] **Step 1: Exemplos primeiro (gerador)**

Funções: `household_vector()` = `{"city_block_id": BLOCK, "risk_class": "yellow", "risk_assessed_on": "2026-11-03", "last_worked_on": "2026-11-03", "last_situation": "worked"}`; `city_block()` = `{"id": BLOCK, "neighborhood_id": cu(40), "number": "12A", "active": True, "neighbor_ids": [BLOCK2], "version": 2}`; `ovitrap()` = `{"id": OVI, "number": "OVT-014", "household_id": HH, "status": "active", "open_reading": {"installed_on": "2026-11-12"}, "last_reading": {"collected_on": "2026-11-12", "condition": "ok", "egg_count": 37}, "version": 5}`; `liraa_item()` = `{"id": LIRAA_ITEM, "survey_id": SURVEY, "survey_name": "LIRAa novembro 2026", "household_id": HH, "city_block_id": BLOCK, "status": "pending", "version": 1}`; `transmission_block()` = `{"id": TBLOCK, "status": "active", "block_ids": [BLOCK, BLOCK2], "cycles": [{"number": 1, "starts_on": "2026-11-10", "ends_on": "2026-11-14"}], "version": 3}`.

`page_vector()`: `reset` False, itens `[household HH v3 com body = household() + vector, city_block BLOCK v2, ovitrap OVI v5, liraa_item v1, transmission_block v3]` (índices 0–4), `removals` [], `who_to_visit` None, `has_more` True.

| Suíte | Arquivo | Conteúdo | `expect` |
|---|---|---|---|
| changes | `changes-vector-page.json` | `page_vector()` | `valid` |
| changes | `changes-vector-closed.json` | última página (`has_more` False, `who_to_visit()`): household com `vector: None`; ovitrap `removed` com `open_reading` None e `last_reading.egg_count` None; liraa_item `closed`; transmission_block `closed` com três ciclos | `valid` |
| changes | `invalid-changes-transmission-block-with-case.json` | `items[4].body.cases = [{"disease": "dengue", "case_on": "2026-11-08"}]` | `/items/4/body/cases schema` |
| changes | `invalid-changes-transmission-block-proposed.json` | `items[4].body.status = "proposed"` | `/items/4/body/status enum` |
| changes | `invalid-changes-ovitrap-without-version.json` | sem `items[2].version` | `/items/2 required` |
| changes | `invalid-changes-city-block-number-with-space.json` | `items[1].body.number = "12 A"` | `/items/1/body/number pattern` |
| changes | `invalid-changes-household-vector-risk-blue.json` | `items[0].body.vector.risk_class = "blue"` | `/items/0/body/vector/risk_class enum` |

Rode o gerador. Expected: `… · changes-response-v1.json 25 · … · total 161`.

- [ ] **Step 2: Rode e veja falhar** — `cd $R && $C/field-sync/tools/check.sh changes-response-v1.json | grep -c '^FAIL'` · Expected: os dois válidos e os cinco inválidos novos falham (`enum` no `type`, `schema` em `vector`).

- [ ] **Step 3: Mude os dois esquemas** pelas Interfaces; `$defs/household` e `$defs/household_vector` copiados byte a byte entre `upload-response-v1.json` e `changes-response-v1.json`.

- [ ] **Step 4: Rode e veja passar** — `cd $R && $C/field-sync/tools/check.sh | tail -1 && python3 $C/field-sync/tools/check_domains.py | head -2` · Expected: `161/161 casos`; `defs-iguais`; `dominios-iguais`. Se o `invalid-changes-household-vector-risk-blue.json` sair com `anyOf` em `/items/0/body/vector` no lugar do `enum`, ajuste só o `expect` e anote.

- [ ] **Step 5: Commit**

```bash
cd $C && $G add field-sync/changes-response-v1.json field-sync/upload-response-v1.json field-sync/tools/make_examples.py field-sync/examples
$G commit -m "feat: add the household vector profile and the vector control entities to the field sync feed" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato:** "contracts-mod35a Task 2: vector no <household> (ausente/null, CV-10), city_block, ovitrap, liraa_item e transmission_block sem caso; 161/161."

---

### Task 3: Domínios de vigilância na referência e em `domains-v1.json`

**Files (worktree `$C`):**
- Modify: `field-sync/reference-response-v1.json`, `field-sync/domains-v1.json`, `field-sync/tools/check_domains.py`, `field-sync/tools/make_examples.py`; regenerate `field-sync/examples/`

**Interfaces:**
- Consumes: os `$defs` das Tasks 1–2; contrato §3, §6, §13 (`vector_control_domains`), §17 (códigos).
- Produces:
  - `domains-v1.json`, chave nova `vector_control` (depois de `location`). Listas `{ code, label }` na ordem abaixo; rótulos da CV-7 (o risco é o do §2):

    | Chave | Códigos e rótulos |
    |---|---|
    | `activities` | `li` "LI — Levantamento de índice"; `li_t` "LI+T — Levantamento de índice e tratamento"; `pe` "PE — Ponto estratégico"; `ie` "IE — Imóvel especial"; `transmission_block` "Bloqueio de transmissão"; `liraa` "LIRAa" |
    | `work_types` | `routine` "Rotina"; `pending` "Recuperação de pendência" |
    | `situations` | `worked` "Trabalhado"; `closed` "Fechado"; `refused` "Recusado"; `vacant` "Desocupado" |
    | `deposit_groups` | `A1` "A1 — Depósito de água elevado"; `A2` "A2 — Depósito de armazenamento ao nível do solo"; `B` "B — Pequeno depósito móvel"; `C` "C — Depósito fixo"; `D1` "D1 — Pneu e outro material rodante"; `D2` "D2 — Lixo, sucata e entulho"; `E` "E — Depósito natural" |
    | `measure_methods` | `mechanical` "Mecânico (eliminação)"; `focal` "Focal (larvicida)"; `perifocal` "Perifocal (adulticida residual)"; `spatial` "Espacial (UBV)" |
    | `risk_classes` | `green` "Verde (baixo)"; `yellow` "Amarelo (médio)"; `red` "Vermelho (alto)"; `gray` "Cinza (difícil acesso, com agendamento)"; `purple` "Roxo (difícil acesso, exige intervenção legal ou VISA)" |
    | `product_units` | `g` "g"; `ml` "ml"; `un` "unidade" |
    | `ovitrap_conditions` | `ok` "Íntegra"; `damaged` "Danificada"; `lost` "Perdida"; `dry` "Seca" |
    | `ovitrap_statuses` | `active` "Ativa"; `removed` "Retirada"; `lost` "Perdida" |
    | `liraa_item_statuses` | `pending` "Pendente"; `inspected` "Inspecionado"; `substituted` "Substituído"; `closed` "Levantamento fechado" |
    | `transmission_block_statuses` | `active` "Ativo"; `closed` "Fechado" |

    mais `"limits": { "deposit_count_max": 999, "charges_max": 999, "samples_max": 99, "larvae_collected_max": 10, "measures_max": 20 }` (a forma do `limits` da referência, Q7(3)) e as regras do §3 e do §6 em dado:
    ```json
    "rules": {
      "registrar_cbos": ["515140"],
      "household_required": true,
      "situation_outcome": { "worked": 1, "refused": 2, "closed": 3, "vacant": 3 },
      "not_worked_empty": ["deposits", "measures", "samples"],
      "not_worked_educational_action": false,
      "activity_property_types": { "pe": ["4"], "ie": ["5", "6", "7", "8", "9", "10", "11", "12", "13"] },
      "activity_links": { "liraa": "liraa_draw_item_id", "transmission_block": "transmission_block_id" },
      "deposit_count_not_above": { "with_water": "existing", "with_larvae": "with_water", "eliminated": "existing", "treated": "with_water" },
      "measure_without_product": ["mechanical"],
      "charges_decimals": 2,
      "samples_per_group_not_above": "with_larvae",
      "times": { "together": true, "max_hours": 12, "same_day_as_visit": true },
      "derived_reasons": { "34": "educational_action", "35": "with_larvae", "36": "eliminated_or_mechanical", "37": "focal" },
      "sample_label": "upcase(client_uuid da visita[0,8]) + \"-\" + number com 2 dígitos",
      "sample_label_collision": "sample_label_taken",
      "ovitrap_collection_reinstall_default": true,
      "ovitrap_remove_ignores_reinstall": true }
    ```
  - `domains-v1.json` `errors.item` + (no fim, nesta ordem) `vector_inspection_invalid`, `situation_outcome_mismatch`, `inspection_not_worked`, `activity_property_mismatch`, `liraa_item_not_found`, `transmission_block_not_active`, `deposit_counts_inconsistent`, `product_not_allowed`, `samples_exceed_larvae`, `inspection_times_invalid`, `reason_derived`, `block_already_set`, `household_without_neighborhood`, `block_not_in_household_neighborhood`, `household_without_block`, `consent_required`, `ovitrap_number_taken`, `collection_out_of_range`, `reading_not_open`, `ovitrap_not_active`, `sample_label_taken` (os 20 de item do §17 e o da Q7(1); os de rota do dashboard ficam fora, inclusive o `invalid_transition`, que é HTTP do dashboard, Q9). `description` do arquivo cita o 35a.
  - `reference-response-v1.json`: `properties.vector_control_domains = { "$ref": "#/$defs/vector_control_domains", "description": "Só com o gate vector_control utilizável (§13)." }` (fora do `required`); `$defs/uuid_or_null` **byte a byte** o dos outros arquivos; e
    ```json
    "vector_control_domains": { "type": "object", "additionalProperties": false,
      "required": ["activities", "work_types", "situations", "deposit_groups", "measure_methods", "risk_classes", "products", "cadences", "parameters", "limits"],
      "properties": {
        "activities": { "type": "array", "minItems": 1, "items": { "$ref": "#/$defs/string_entry" } },
        "work_types": …, "situations": …, "deposit_groups": …, "measure_methods": …, "risk_classes": … (iguais a activities),
        "products": { "type": "array", "items": { "type": "object", "additionalProperties": false,
          "required": ["id", "code", "name", "unit", "charge_size", "uses"],
          "properties": { "id": { "$ref": "#/$defs/uuid" }, "code": { "type": "string", "minLength": 1, "maxLength": 10 },
            "name": { "type": "string", "minLength": 1, "maxLength": 80 }, "unit": { "enum": ["g", "ml", "un"] },
            "charge_size": { "type": "number", "exclusiveMinimum": 0, "multipleOf": 0.01 },
            "uses": { "type": "array", "minItems": 1, "uniqueItems": true, "items": { "enum": ["focal", "perifocal", "spatial"] } } } } },
        "cadences": { "type": "array", "items": { "type": "object", "additionalProperties": false,
          "required": ["neighborhood_id", "risk_class", "strategic_point", "days"],
          "properties": { "neighborhood_id": { "$ref": "#/$defs/uuid_or_null" },
            "risk_class": { "enum": ["green", "yellow", "red", "gray", "purple", null] },
            "strategic_point": { "type": "boolean" }, "days": { "type": "integer", "minimum": 7, "maximum": 365 } } } },
        "parameters": { "type": "object", "additionalProperties": false,
          "required": ["default_days", "strategic_point_days", "ovitrap_collection_days_min", "ovitrap_collection_days_max"],
          "properties": { cada uma { "type": "integer", "minimum": 1 } } },
        "limits": { "type": "object", "additionalProperties": false, "required": ["deposit_count_max", "charges_max", "samples_max", "larvae_collected_max", "measures_max"],
          "properties": { cada uma { "type": "integer", "minimum": 1 } } } } }
    ```
  - `check_domains.py`: com `vc = dom["vector_control"]`, `vi = req["vector_inspection"]["properties"]`, `chg = s["changes-response-v1.json"]["$defs"]` e `nn(enum) = sorted(x for x in enum if x is not None)`, os pares novos (nome → domínio × esquema):
    - `vector_control.activities` × `vi.activity.enum`; `.work_types` × `vi.work_type.enum`; `.situations` × `vi.situation.enum` e × `nn(chg.household_vector.last_situation.enum)`; `.risk_classes` × `vi.risk_class.enum` e × `nn(chg.household_vector.risk_class.enum)`; `.deposit_groups` × `req.deposit_group.enum`; `.measure_methods` × `req.vector_measure.method.enum`; `.ovitrap_conditions` × `req.ovitrap_collection_body.condition.enum` e × a `condition` do `last_reading` de `chg.ovitrap`; `.ovitrap_statuses` × `chg.ovitrap.status.enum`; `.liraa_item_statuses` × `chg.liraa_item.status.enum`; `.transmission_block_statuses` × `chg.transmission_block.status.enum`;
    - `vector_control.limits`: `deposit_count_max` = `req.deposit_count.maximum`; `charges_max` = `vector_measure.charges.maximum`; `samples_max` = `vector_sample.number.maximum` = `vi.samples.maxItems`; `larvae_collected_max` = `vector_sample.larvae_collected.maximum`; `measures_max` = `vi.measures.maxItems`; `len(deposit_groups)` = `vi.deposits.maxItems` (7); `10 ** -rules.charges_decimals` = `charges.multipleOf`;
    - `vector_control.rules`: `registrar_cbos` ⊆ `home_visit.rules.registrar_cbos`; as chaves de `derived_reasons` (int) ⊆ `req.visit_reason.enum`; os valores de `activity_property_types` ⊆ `req.property_type.enum`.
    A docstring do arquivo ganha "vector_control".

- [ ] **Step 1: Pares primeiro** — escreva os pares de `check_domains.py` pelas Interfaces.

- [ ] **Step 2: Rode e veja falhar** — `python3 $C/field-sync/tools/check_domains.py; echo "exit $?"` · Expected: `KeyError: 'vector_control'` (exit ≠ 0).

- [ ] **Step 3: Dados e exemplos** — escreva `vector_control` e os 21 códigos em `domains-v1.json`. No gerador, `reference_vector()` = `reference()` + `vector_control_domains`, com as seis listas **lidas de `domains-v1.json`** (`vector_control.activities` … `risk_classes`; uma fonte só), `products` `[{"id": PRODUCT, "code": "SPN", "name": "Espinosade 7,48% DT (larvicida)", "unit": "un", "charge_size": 1, "uses": ["focal"]}, {"id": PRODUCT2, "code": "CDM", "name": "Clotianidina + deltametrina PM (adulticida residual)", "unit": "g", "charge_size": 12.5, "uses": ["perifocal"]}]`, `cadences` `[{"neighborhood_id": None, "risk_class": None, "strategic_point": True, "days": 15}, {"neighborhood_id": cu(40), "risk_class": "red", "strategic_point": False, "days": 30}]`, `parameters` `{"default_days": 60, "strategic_point_days": 15, "ovitrap_collection_days_min": 3, "ovitrap_collection_days_max": 10}` e `limits` = `vector_control.limits` lido do arquivo.

| Suíte | Arquivo | Conteúdo | `expect` |
|---|---|---|---|
| reference | `reference-vector-control.json` | `reference_vector()` | `valid` |
| reference | `invalid-reference-product-without-uses.json` | `vector_control_domains.products[0].uses = []` | `/vector_control_domains/products/0/uses minItems` |

Rode o gerador (`… · reference-response-v1.json 8 · total 163`) e `cd $R && $C/field-sync/tools/check.sh reference-response-v1.json`. Expected: os dois novos `FAIL` (`/vector_control_domains schema`); `check_domains.py` já mostra `dominios-iguais` e `erros-conhecidos`.

- [ ] **Step 4: Mude `reference-response-v1.json`** pelas Interfaces.

- [ ] **Step 5: Rode e veja passar**

Run: `cd $R && $C/field-sync/tools/check.sh | tail -1 && python3 $C/field-sync/tools/check_domains.py --release; echo "exit $?"`
Expected: `163/163 casos`; `defs-iguais`, `dominios-iguais`, `erros-conhecidos`, `prova: fcf23c87 (2026-11-12)`, `exit 0`. Teste do verificador: troque `li_t` por `lit` em `vector_control.activities`, rode, veja `DOMINIOS DIFERENTES: vector_control.activities` e exit 1, e desfaça; troque `limits.larvae_collected_max` para 11, veja o mesmo erro com `vector_control.limits`, e desfaça.

- [ ] **Step 6: Commit**

```bash
cd $C && $G add field-sync/reference-response-v1.json field-sync/domains-v1.json field-sync/tools/check_domains.py field-sync/tools/make_examples.py field-sync/examples
$G commit -m "feat: add vector control domains and item error codes to field sync reference and domains" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato:** "contracts-mod35a Task 3: vector_control_domains na referência, vector_control (rótulos CV-7, regras em dado) e 21 códigos em errors.item; 163/163; --release verde."

---

### Task 4: Achados do contracts#2 — logradouro em código e regras da atividade (Q3)

**Files (worktree `$C`):**
- Modify: `field-sync/reference-response-v1.json`, `field-sync/domains-v1.json`, `field-sync/tools/check_domains.py`, `field-sync/tools/make_examples.py`; regenerate `field-sync/examples/`

**Interfaces:**
- Consumes: `CollectiveActivities::Json.domains[:rules]` do api (`app/services/collective_activities/json.rb:95-118`, api `469e2e68`): `list_max`, `by_type`, `health_topics_with_meeting_topic`, `practices_only_in_type`, `procedure_practices`, `smoking_practices`, `participant_measurements`; `"081": "RUA"` em `config/ledi/registration_dictionaries.yml` do api.
- Produces:
  - `domains-v1.json`, `collective_activity.rules` (CV-12): as sete chaves **iguais** ao JSON do api, nesta forma —
    ```json
    "list_max": { "meeting_topics": 7, "target_audience": 17, "health_topics": 16, "health_practices": 14 },
    "by_type": { "1": { "meeting_topics": "required", "target_audience": "forbidden", "health_topics": "forbidden", "health_practices": "forbidden", "participants": "optional" }, … "7": { … } },
    "health_topics_with_meeting_topic": [6],
    "practices_only_in_type": [{ "type": 6, "practices": [2, 9, 33] }],
    "procedure_practices": [30],
    "smoking_practices": [25, 26, 27, 28],
    "participant_measurements": { "weight_kg": { "min": 0.5, "max": 500, "decimals": 3 }, "height_cm": { "min": 20, "max": 250, "decimals": 1 } }
    ```
    (`by_type` 1–7 = o `by_activity_type` atual com `health_topics` `"forbidden"` nos tipos 1–3; a exceção do tema 6 fica só em `health_topics_with_meeting_topic`). Saem `by_activity_type`, `practices_only_in_activity_type`, `procedure_code_with_practice`, `smoking_with_practices` e o `ledi` de `participant_measurements`; `values` perde `with_meeting_topic_6`. As outras chaves (`values`, `procedure_code_domain`, `pse_only_health_topics`, `participants_count`, `altered_count`, `participants_required_identifier_types`, `participant_without_identifier_in_other_types`, `unidentified_patient`, `place_text_max_length`, `place_inep_allowed`, `mistaken_reason_length`, `registrar_cbos_table`) ficam.
  - `reference-response-v1.json`: `collective_activity_domains.properties.rules = { "$ref": "#/$defs/collective_activity_rules" }` e
    ```json
    "collective_activity_rules": { "type": "object", "additionalProperties": false,
      "required": ["list_max", "by_type", "health_topics_with_meeting_topic", "practices_only_in_type", "procedure_practices", "smoking_practices", "participant_measurements"],
      "properties": {
        "list_max": { "type": "object", "additionalProperties": false, "required": ["meeting_topics", "target_audience", "health_topics", "health_practices"],
          "properties": { cada uma { "type": "integer", "minimum": 1 } } },
        "by_type": { "type": "object", "minProperties": 1, "propertyNames": { "pattern": "^[1-7]$" },
          "additionalProperties": { "type": "object", "additionalProperties": false,
            "required": ["meeting_topics", "target_audience", "health_topics", "health_practices", "participants"],
            "properties": { cada uma { "enum": ["required", "forbidden", "optional"] } } } },
        "health_topics_with_meeting_topic": { "type": "array", "items": { "type": "integer" } },
        "practices_only_in_type": { "type": "array", "items": { "type": "object", "additionalProperties": false, "required": ["type", "practices"],
          "properties": { "type": { "type": "integer" }, "practices": { "type": "array", "items": { "type": "integer" } } } } },
        "procedure_practices": { "type": "array", "items": { "type": "integer" } },
        "smoking_practices": { "type": "array", "items": { "type": "integer" } },
        "participant_measurements": { "type": "object", "additionalProperties": false, "required": ["weight_kg", "height_cm"],
          "properties": { "weight_kg": { "$ref": "#/$defs/measurement_range" }, "height_cm": { "$ref": "#/$defs/measurement_range" } } } } },
    "measurement_range": { "type": "object", "additionalProperties": false, "required": ["min", "max", "decimals"],
      "properties": { "min": { "type": "number" }, "max": { "type": "number" }, "decimals": { "type": "integer", "minimum": 0 } } }
    ```
  - `make_examples.py`: `ADDRESS["street_type"] = "081"`; `reference()` passa a pôr em `collective_activity_domains.rules` as sete chaves **lidas de `domains-v1.json`** (`{k: ca_rules[k] for k in REFERENCE_RULE_KEYS}`), e `reference_vector()` herda.
  - `check_domains.py`, duas guardas novas:
    - `regras-da-atividade`: constante `REFERENCE_RULE_KEYS` (as sete chaves, na ordem do api; a mesma constante no gerador); todas existem em `collective_activity.rules`; as chaves de `by_type` = `str` dos códigos de `collective_activity_body.activity_type.enum`; todo valor de `by_type` ∈ {`required`, `forbidden`, `optional`}; e, quando `reference-response-v1.json` tiver `$defs/collective_activity_rules`, o `required` dele = `REFERENCE_RULE_KEYS`. Falha imprime `REGRAS DA ATIVIDADE: <motivo>`.
    - `logradouro-em-codigo`: todo `street_type` de todo exemplo (busca recursiva na chave) é `None` ou casa `^[0-9]{3}$`. Falha imprime `LOGRADOURO FORA DO CODIGO: <arquivos>`.
  - O esquema de `address.street_type` **não muda** (CV-11).

- [ ] **Step 1: Guardas primeiro** — escreva as duas guardas em `check_domains.py` e rode `python3 $C/field-sync/tools/check_domains.py; echo "exit $?"`. Expected: `REGRAS DA ATIVIDADE: faltam by_type, health_topics_with_meeting_topic, …` e `LOGRADOURO FORA DO CODIGO:` com 28 arquivos ou mais (os que usam `ADDRESS`), exit 1.

- [ ] **Step 2: Exemplos (gerador)** — `ADDRESS` e `reference()` pelas Interfaces, e os inválidos:

| Suíte | Arquivo | Conteúdo | `expect` |
|---|---|---|---|
| reference | `invalid-reference-activity-rules-old-vocabulary.json` | `collective_activity_domains.rules = {"by_activity_type": {}}` | `/collective_activity_domains/rules required` |
| reference | `invalid-reference-activity-by-type-old-value.json` | `rules.by_type["1"].health_topics = "with_meeting_topic_6"` | `/collective_activity_domains/rules/by_type/1/health_topics enum` |

Não rode o gerador ainda (ele lê as chaves novas de `domains-v1.json`).

- [ ] **Step 3: `domains-v1.json`** pelas Interfaces. Confira contra o api servido:

```bash
cd $R && docker compose exec -T api bin/rails runner 'puts JSON.generate(CollectiveActivities::Json.domains[:rules])' \
  | python3 -c 'import json,sys; a=json.loads(sys.stdin.read().strip().splitlines()[-1]); d=json.load(open(sys.argv[1]))["collective_activity"]["rules"]; print("igual ao api" if set(a)<=set(d) and all(d[k]==a[k] for k in a) else "DIFERENTE: " + ", ".join(k for k in a if d.get(k)!=a[k]))' $C/field-sync/domains-v1.json
grep -n '"081": "RUA"' $R/apps/api/config/ledi/registration_dictionaries.yml
```

Expected: `igual ao api`; a linha do `081`. Se o container do api não estiver na `main` (outra sessão trocou de branch), confira pelo `git show origin/main:app/services/collective_activities/json.rb` do api e anote. Depois rode o gerador: `… · reference-response-v1.json 10 · total 165`.

- [ ] **Step 4: Rode e veja falhar** — `cd $R && $C/field-sync/tools/check.sh reference-response-v1.json | grep '^FAIL'` · Expected: os dois inválidos novos falham (o `rules` ainda é aberto); `reference.json` e `reference-vector-control.json` passam.

- [ ] **Step 5: Mude `reference-response-v1.json`** pelas Interfaces.

- [ ] **Step 6: Rode e veja passar**

Run: `cd $R && $C/field-sync/tools/check.sh | tail -1 && python3 $C/field-sync/tools/check_domains.py --release; echo "exit $?"; grep -rl '"RUA"' $C/field-sync/examples | grep -v reference`
Expected: `165/165 casos`; `defs-iguais`, `dominios-iguais`, `regras-da-atividade`, `logradouro-em-codigo`, `erros-conhecidos`, `prova: fcf23c87 (2026-11-12)`, `exit 0`; o `grep` não lista nada (`"RUA"` só como rótulo de `tipo_logradouro` na referência).

- [ ] **Step 7: Commit**

```bash
cd $C && $G add field-sync/reference-response-v1.json field-sync/domains-v1.json field-sync/tools/check_domains.py field-sync/tools/make_examples.py field-sync/examples
$G commit -m "fix: use street type codes in field sync examples and align the collective activity rules with the api" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato:** "contracts-mod35a Task 4: contracts#2 — street_type '081' nos exemplos (com guarda), rules da atividade na forma do api (domains e esquema da referência); 165/165 (61 válidos)."

---

### Task 5: README e CHANGELOG

**Files:** Modify `field-sync/README.md`, `field-sync/CHANGELOG.md`, `README.md` (raiz).

**Interfaces:**
- Consumes: Tasks 1–4.
- Produces: o texto que o app e o api citam; a tabela da raiz com `field-sync-v1.3.0`.

- [ ] **Step 1: `field-sync/README.md`**, regras curtas (uma por item, sem repetir o esquema), todas marcadas "(v1.3.0)":
  - **Interruptor:** `features.vector_control` (ausente = false). Desligado → itens `rejected feature_disabled`; o que foi gravado fica.
  - **Compatibilidade (Q6),** com este texto: o api só emite as entidades novas (`city_block`, `ovitrap`, `liraa_item`, `transmission_block`) e o `<household>.vector` para cidade com o gate `vector_control` utilizável; ligar ou desligar o interruptor chega a todo aparelho da cidade como `reset: true`; o app anterior ao do 35a não ignora tipo desconhecido no download e quebra com as entidades novas, por isso **nenhum aparelho com app anterior ao do 35a pode estar numa cidade com `vector_control` ligado** (gate de go-live); o app do 35a ignora tipo desconhecido e zera o cursor ao migrar o esquema local para a v2. O api aceita o envio da v1.2.0 sem mudança.
  - **Boletim (`vector_inspection`):** só o agente de endemias (CBO 515140) nas microáreas atribuídas (`cbo_not_allowed`, `path` `vector_inspection`); exige domicílio (`household_id` ou `household_client_uuid`; sem ele → `vector_inspection_invalid` `path` `household_id`, CV-6); listas ausentes = vazias; as relações estão em `vector_control.rules`, e o api decide com os códigos do §17; erro de esquema dentro do boletim → `rejected vector_inspection_invalid` com o mesmo `path` (CV-4); os motivos 34–37 não vêm no `reasons` (`reason_derived`): o api os deriva; a etiqueta da amostra é `upcase(client_uuid[0,8]) + "-" + NN`, calculada igual no app e no api, e colisão → `sample_label_taken` (código de item; 422 no dashboard, Q13(3)), `path` `vector_inspection.samples[i]` (Q7(1); o app só troca as etiquetas por ação explícita do agente); os `path` das recusas são os da Task 2 do plano do dashboard (Q9); `charges` com até 2 casas (o Ajv precisa de `multipleOfPrecision`, como na Q49).
  - **Domicílio pelo agente de endemias (Q13(1)):** com `vector_control` utilizável, o item `household` deixa de ser só do ACS: o agente de endemias (515140) cadastra e edita, nas microáreas atribuídas a ele, domicílio **só** de tipo não residencial (`2 3 4 5 6 12 13`), sem família nem morador. Tipo residencial, troca para residencial, `family` e `individual_registration` → `rejected cbo_not_allowed`, como hoje. O esquema não muda (autoria não é forma), e por isso não há exemplo novo.
  - **Ovitrampa:** `ovitrap_installation` (domicílio por id ou `client_uuid`; `consent` false → `consent_required`) e `ovitrap_collection` (por `ovitrap_id` ou `ovitrap_client_uuid`, que resolve pelo livro de idempotência; falta → `not_found` `path` `ovitrap_client_uuid`); `reinstall` padrão true e ignorado com `remove: true` (o app manda `reinstall: !remove`), sem acoplamento no esquema; `condition: lost` + `remove: true` = perdida (CV-9). O `id` do resultado das duas é o da **ovitrampa** (CV-3). A contagem de ovos nunca vem do app.
  - **Resultado:** `vector_inspection_id` só no `accepted`/`duplicate` de `home_visit` com boletim.
  - **Download:** `<household>.vector` (ausente = gate não utilizável; `null` = sem perfil; **só leitura**, nunca vai no item `household`, CV-10); mudança de perfil reemite o `<household>` com a mesma `version` do 24a (o app sobrescreve); `city_block`, `ovitrap`, `liraa_item` (`closed` = levantamento fechado) e `transmission_block` (só `active` e `closed`; nunca caso nem notificação), cada um com `version`; o fim vem pelo `status`, sem `removal` novo.
  - **Referência:** `vector_control_domains` só com o gate utilizável; `collective_activity_domains.rules` na forma de `CollectiveActivities::Json.domains` (Q3).
  - **`street_type`:** é o código DNE de `string_domains.tipo_logradouro` (`"081"`), nunca o rótulo; o esquema aceita texto, e quem recusa é o api (`invalid_household`, `path` `address.street_type`; CV-11).
  - contagem dos exemplos (a do gerador: 165, 61 válidos) e os comandos com `165/165`; `check_domains.py` lista `regras-da-atividade` e `logradouro-em-codigo`.
  - **Consumo:** o app gera os tipos pela tag `field-sync-v1.3.0`.

- [ ] **Step 2: `field-sync/CHANGELOG.md`**, no topo: `## field-sync-v1.3.0 — (não publicada até a tag) — MINOR`, "Vigilância ambiental de endemias (módulo 35a, ADR 0054). Compatível com o v1.2.0: nada do pedido v1.2.0 passa a ser recusado; o novo nas respostas é opcional." e uma linha por arquivo (o conteúdo das Tasks 1–4), mais a seção "Pendências do contracts#2 (Q3)": os dois achados resolvidos e "os seis itens adiados continuam adiados".

- [ ] **Step 3: `README.md` da raiz:** na linha de `field-sync/`, `field-sync-v1.2.0` → `field-sync-v1.3.0`; a descrição ganha "vigilância ambiental de endemias" e "(ADRs 0041, 0052, 0053 e 0054)"; no parágrafo `### field-sync`, a frase "desde o v1.3.0 (ADR 0054), também o boletim do agente de endemias, as ovitrampas, o LIRAa e o bloqueio".

- [ ] **Step 4: Verificação**

Run: `cd $R && $C/field-sync/tools/check.sh | tail -1 && python3 $C/field-sync/tools/check_domains.py --release && grep -c "vector_inspection_invalid\|não residencial\|reset: true\|ovitrap_client_uuid\|multipleOfPrecision\|\"081\"\|só leitura" $C/field-sync/README.md`
Expected: `165/165 casos`; exit 0; contagem ≥ 6.

- [ ] **Step 5: Commit**

```bash
cd $C && $G add field-sync/README.md field-sync/CHANGELOG.md README.md
$G commit -m "docs: document field-sync v1.3.0 rules and changelog" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: SHA-256 ao api e parada

**Files:** nenhum.

**Interfaces:**
- Produces: o `HEAD` da branch, os seis SHA-256 e a lista do que muda no api.

- [ ] **Step 1: Verificação e SHA-256**

```bash
cd $R && $C/field-sync/tools/check.sh | tail -1 && python3 $C/field-sync/tools/check_domains.py --release
$G -C $C log --oneline origin/main..HEAD
$G -C $C diff --stat origin/main..HEAD -- protocols session clinical exams lab billing events types design-tokens map-style | tail -1
(cd $C/field-sync && shasum -a 256 session-response-v1.json upload-request-v1.json upload-response-v1.json changes-response-v1.json reference-response-v1.json domains-v1.json)
```

Expected: `165/165`; exit 0; cinco commits; nada fora de `field-sync/` e do `README.md`; seis SHA-256.

- [ ] **Step 2: Pare.** Sem merge nem push. Mande ao coordenador o `HEAD`, o caminho `$C/field-sync/`, os seis SHA-256 e o que o api tem de mudar junto:
  1. `FieldSync::Schemas::SHA256` e os literais do `spec/services/field_sync/schemas_spec.rb` (124 → 165 exemplos; 49 → 61 válidos);
  2. a cópia de `examples/` em `spec/fixtures/field_sync/examples/` (165 arquivos + `manifest.json`);
  3. `spec/services/field_sync/domains_spec.rb`: as regras da atividade com as chaves novas, comparadas ao `CollectiveActivities::Json.domains[:rules]` em JSON (sem `ledi`); `vector_control` contra `VectorControl::Catalog` (códigos, rótulos, ordem e `limits`); os 21 códigos em `errors.item` (com `sample_label_taken`);
  4. `FieldSync::Catalog::ITEM_TYPES` + `ovitrap_installation`, `ovitrap_collection` (o exemplo que fixa `.last == "family"` muda);
  5. as decisões que só o api cumpre: CV-1 e CV-2 / Q6 (emitir o novo só com o gate utilizável; o interruptor no `scope_digest`, `reset: true` na troca), CV-3 (`id` = ovitrampa), CV-4 (erro de esquema no boletim → `vector_inspection_invalid`), CV-6, CV-8, CV-9 e CV-10.

  Toda mudança pedida pelo api volta às Tasks 1–5 e gera SHA-256 novos.

**Relato:** "contracts-mod35a Task 6: branch em <sha>, 165/165; SHA-256: <lista>; aguardando o api."

---

### Task 7: Entrega — SHA-256 conferidos, merge, tag, push e issue (com autorização)

**Files:** `field-sync/CHANGELOG.md` (data). Merge, tag, push e comentário só com autorização.

**Interfaces:**
- Consumes: o api com as cópias presas aos SHA-256 finais (relato do api ao coordenador).
- Produces: `field-sync-v1.3.0` em `origin` (o campo gera os tipos a partir desta tag).

- [ ] **Step 1: Verificação final** — o bloco da Task 6, Step 1. Expected: os seis SHA-256 **iguais** aos que o spec do api prende (confirmados pelo coordenador). Diferente: pare.

- [ ] **Step 2: Data no CHANGELOG** (`(não publicada até a tag)` → `<AAAA-MM-DD>`) e commit:

```bash
cd $C && $G add field-sync/CHANGELOG.md
$G commit -m "docs: date the field-sync-v1.3.0 changelog entry" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

O CHANGELOG muda o SHA-256 só dele, não o dos seis arquivos presos.

- [ ] **Step 3: Pare e peça autorização**, uma etapa por vez: (1) merge em `main`; (2) tag `field-sync-v1.3.0`; (3) push da `main` e da tag; (4) comentário no contracts#2. Reporte o hash, `165/165`, os SHA-256 e as decisões aplicadas (CV-1 a CV-14 com as Q-n).

- [ ] **Step 4: Com autorização**

```bash
$G -C $R/contracts fetch origin && $G -C $R/contracts log --oneline main..origin/main
$G -C $R/contracts checkout main && $G -C $R/contracts merge --ff-only feat/mod-35a-vector-control
$G -C $R/contracts tag -a field-sync-v1.3.0 -m "field-sync-v1.3.0 — MINOR"
$G -C $R/contracts log --oneline origin/main..main
$G -C $R/contracts push origin main && $G -C $R/contracts push origin field-sync-v1.3.0
$G -C $R/contracts worktree remove $R/.worktrees/contracts-mod35a && $G -C $R/contracts branch -d feat/mod-35a-vector-control
```

Expected: `main..origin/main` vazio (senão: rebase e refaça o Step 1); fast-forward; `origin/main..main` só com os commits deste plano; push aceito (bypass do aviso de PR, como nos módulos anteriores).

- [ ] **Step 5: Comentário no contracts#2** (só com a autorização da etapa 4), um `gh` simples: `gh issue comment 2 --repo rotasaude/contracts --body-file <arquivo no scratchpad>`, com o texto: os dois achados resolvidos em `field-sync-v1.3.0` (`street_type` em código nos exemplos, com guarda no `check_domains.py`, e o esquema sem restrição, CV-11; `rules` da atividade na forma do api em `domains-v1.json` e fechado na referência), e os seis itens que continuam adiados. A issue fica aberta.

**Relato:** "contracts-mod35a Task 7: field-sync-v1.3.0 publicada (main <sha>, tag pushada); contracts#2 comentada."

---

## Self-review (feito ao escrever o plano)

1. **Cobertura do §13:** `session-response-v1.json` (Task 1); `upload-request-v1.json` (Task 1: boletim fechado e os dois itens, com `household_client_uuid` e `ovitrap_client_uuid`); `upload-response-v1.json` (Task 1: `vector_inspection_id`; `ovitrap_id` substituído pelo `id`, CV-3); `changes-response-v1.json` (Task 2: `vector` e as quatro entidades com os corpos da tabela do §13); `reference-response-v1.json` (Task 3: os dez blocos de `vector_control_domains`); `domains-v1.json` (Task 3: `vector_control` e os códigos de item do §17); `examples/` (válidos e inválidos de cada novidade); Q3 (Task 4). §3: as regras 1–11 estão em dado em `vector_control.rules`, e a forma está no esquema. §6 e §9: só o que chega ao app (coleta, condição, etiqueta no README). §17 e Q7: os 20 códigos de item e o `sample_label_taken`; os de rota do dashboard (inclusive `invalid_transition`, Q9) ficam fora de `errors.item`, de propósito. Q10: nenhuma semana epidemiológica no `field-sync`.
2. **Passos:** os valores lidos na execução são só hashes, SHA-256, ids do board e o JSON do api na Task 4 (conferência, não insumo). Os rótulos da CV-7 estão fixados aqui.
3. **Consistência:** `$defs/ovitrap_number`, `$defs/household`, `$defs/household_vector` e `$defs/uuid_or_null` iguais entre arquivos, cobrados pelo comando. Os nomes dos exemplos são os mesmos no gerador, no Review Focus e nos relatos. Contagens: 124 → 154 → 161 → 163 → 165. Os códigos de `resp_vector_rejected()` são os 21 da Task 3, com os `path` do plano do dashboard (Q9).
4. **Review Focus:** as cinco linhas têm teste nas Tasks 1–4, salvo o comportamento do app v1.2.0 (CV-1), que é do campo e vai para o README.
5. **Proporção:** o plano dá as formas de esquema que o contrato só nomeia e deixa o resto (descrições, ordem de chaves) para o executor.

## Divergências propostas ao contrato

Todas decididas pelo coordenador em 2026-11-16 (docs `aa4f3c5`; segunda rodada em `c2d755c`, Q13), como recomendado; a coluna final diz a Q.

| Código | Onde | Divergência | Resolução (aplicada no plano) | Decisão |
|---|---|---|---|---|
| CV-1 | §13 "Compatibilidade" | O contrato supõe que o app v1.2.0 ignora tipo desconhecido no download. No campo `8429bd5`, `applyPage` (`src/sync/download.ts:20-27`) manda todo tipo que não é `micro_area` nem `household` para `putRegistration`, que lê `body.patient.id` → `TypeError`. A transação da página falha e o download nunca termina. O app também não valida esquema em execução (`src/contract/validate.ts` é só de teste), então o `vector` e o `features.vector_control` não o quebram. | (a) O api emite as quatro entidades e o `<household>.vector` **só com `VectorControl::Gate.usable?`**. (b) README: a cidade só liga `vector_control` quando todos os aparelhos dela estiverem no app com esquema local v2; vira gate de go-live do 35a (board #2). (c) O plano do campo troca o `else` do `applyPage` por um ramo por tipo e **ignora tipo desconhecido** (teste com `{ type: "visit_sheet" }`). Alternativa descartada: um parâmetro de capacidade no `GET /changes`, que é mais trabalho e cuja cobertura também dependeria da atualização do app. | **Decidida: Q6.** |
| CV-2 | §13 "Entidades novas" | O feed é incremental por cursor. O que já existia quando o gate ligou, ou quando o aparelho atualizou o app, nunca seria reemitido. O contrato só diz que "toda escrita de `version` reemite". | Ligar ou desligar `vector_control` é troca de escopo: o próximo `GET /changes` de todo aparelho da cidade responde `reset: true` (a mecânica é do api, por exemplo uma marca do gate no cursor). A migração 1→2 do campo zera o cursor (download completo). README (Task 5). | **Decidida: Q6.** |
| CV-3 | §13 `upload-response` | O `ovitrap_id` no `accepted` da instalação repete o `id`. E a citação `ovitrap_client_uuid` "resolve pelo livro de idempotência", que guarda o `id` do resultado, então o `id` **tem** de ser o da ovitrampa. | Não há `ovitrap_id`. O `id` do `accepted`/`duplicate` de `ovitrap_installation` e de `ovitrap_collection` é o da ovitrampa, e `invalid-response-ovitrap-id.json` cobra isso. Se o contrato mantiver o campo: `ovitrap_id` opcional só no `accepted`/`duplicate` da instalação (Task 1 troca o inválido por um válido). | **Decidida: Q7(2).** |
| CV-4 | §13, §17 | O contrato não diz o código de um erro de **forma** dentro do boletim (enum, faixa, chave desconhecida). | Erro de esquema sob `/body/vector_inspection` → `rejected vector_inspection_invalid` com o mesmo `path`, como a Q2 do 24c fez para `location`. Os outros erros de esquema seguem `invalid_request`. O esquema deixa fora as relações do §3 (inclusive o par entrada/saída e método × produto), para o app e o dashboard verem o mesmo código. | **Decidida: Q7(7), Q11.** |
| CV-5 | §3 | O contrato não diz quais chaves do boletim são obrigatórias, nem o tamanho de `measures`. | Obrigatórias: `activity`, `work_type`, `situation`, `educational_action`. As listas são opcionais (ausente = vazia, como na atividade coletiva). `entered_at`/`left_at` são opcionais no esquema, e o par é regra 8 do api. O depósito leva as seis chaves. `deposits` ≤ 7 (um por grupo), `samples` ≤ 99 e `measures` ≤ 20 (`measures_max`). Chaves de `limits` da Q7(3): `deposit_count_max`, `charges_max`, `samples_max`, `larvae_collected_max`, `measures_max`. | **Decidida: Q7(3)(4).** |
| CV-6 | §3 | O boletim pressupõe domicílio e perfil, mas a visita do 24b aceita imóvel sem domicílio (`micro_area_id` + `property_type`). | Boletim sem `household_id` nem `household_client_uuid` → `rejected vector_inspection_invalid`, `path` `household_id` (`vector_control.rules.household_required: true`). É relação, então fica fora do esquema. | **Decidida: Q10.** |
| CV-7 | §13 `vector_control_domains` | O contrato só dá os rótulos do risco (§2). Faltam os de atividade, tipo de trabalho, situação, grupo de depósito, método, unidade, condição e status. | A tabela da Task 3 (PNCD/Diretrizes de 2009, encurtada) entra no contrato. `domains-v1.json` é a fonte, e o `VectorControl::Catalog` do api e o dashboard usam os mesmos. | **Decidida: Q11.** |
| CV-8 | §3, §6, §17 | Vários códigos não têm `path` no contrato. | `situation_outcome_mismatch` → `vector_inspection.situation`; `inspection_not_worked` → um por campo indevido (`vector_inspection.deposits` \| `.measures` \| `.samples` \| `.educational_action`); `activity_property_mismatch` → `vector_inspection.activity`; `liraa_item_not_found` → `.liraa_draw_item_id`; `transmission_block_not_active` → `.transmission_block_id`; `product_not_allowed` → `.measures[i].vector_product_id` (o mecânico com produto ou cargas é `vector_inspection_invalid` no campo, como no dashboard); `inspection_times_invalid` → sempre `.entered_at`; `sample_label_taken` → `.samples[i]`; `block_already_set`, `household_without_neighborhood` e `block_not_in_household_neighborhood` → `.city_block_id`; `household_without_block` → `household_id`/`household_client_uuid`; `consent_required` → `consent`; `ovitrap_number_taken` → `number`; `collection_out_of_range` → `collected_on`; `reading_not_open` e `ovitrap_not_active` → `ovitrap_id`/`ovitrap_client_uuid`. | **Decidida: Q11; `path` pela Task 2 do dashboard (Q9); os da ovitrampa confirmados na Q13(2).** |
| CV-9 | §6, §13 | `consent: true` como `const` faria o `false` voltar `invalid_request`, não o `consent_required` do contrato. E `remove: true` com `reinstall: true` não está definido. | `consent` é boolean obrigatório, e `false` → `consent_required` (api). `reinstall` padrão true e `remove` padrão false; com `remove: true`, `reinstall` é ignorado (nenhuma leitura nova). | **Decidida: Q7(5).** |
| CV-10 | §13 `<household>.vector` | O contrato não diz quando o `vector` falta. E o `$defs/household` tem de ser igual na resposta do envio (o `current` do `stale_version`), regra que o `check_domains.py` cobra. | Ausente = gate não utilizável; `null` = gate utilizável e imóvel sem perfil. Entra nos dois arquivos. É só leitura: o item `household` (fechado) recusa `vector` com `invalid_request`, e o app tem de tirá-lo antes de enviar (plano do campo). | **Decidida: Q6(1), Q11.** |
| CV-11 | Q3 / contracts#2 | A issue sugere "se for o caso, restringir o esquema ao código". | Não restringir na v1.3.0. Isso mudaria a recusa do app de `invalid_household` (`path` `address.street_type`) para `invalid_request`, e apertaria o pedido numa MINOR. Ficam os exemplos com `"081"`, a guarda `logradouro-em-codigo` e a frase no README. | **Decidida: Q11.** |
| CV-12 | Q3 / contracts#2 | Trocar o vocabulário de `rules` renomeia chaves de dado numa MINOR, e o `participant_measurements` do api não tem `ledi`. | Só o `domains_spec` do api lê essas chaves (o campo lê a referência, Q76(6) do 24c), e o plano do api o atualiza na mesma entrega. As sete chaves ficam **iguais** ao JSON do api (sem `ledi`). As chaves que o api não serve ficam. O esquema da referência fecha `rules` (aperto de resposta, que o api já cumpre). | **Decidida: Q11.** |
| CV-13 | §1 (`ovitraps.number`) | O contrato dá "texto 1–12" sem padrão; o quarteirão tem padrão. | `^[0-9A-Za-z-]{1,12}$`, o mesmo do quarteirão, no envio e no download (`$defs/ovitrap_number`). O api aplica o mesmo na instalação pelo dashboard. | **Decidida: Q7(6).** |
| CV-14 | §13 `domains-v1.json` | `proof` registra a prova técnica do 24b. Atualizá-lo para um commit do api do 35a criaria um ciclo, porque o api prende o SHA-256 deste arquivo. | `proof` fica `fcf23c87`. A conferência de `vector_control` contra o api é o `domains_spec` do api do 35a (Task 6, item 3). | **Decidida: Q11.** |

Observação, sem divergência: o último comentário do contracts#2 diz que os dois achados entram "junto dos seis itens adiados". O contrato (Q3) mantém os seis adiados, e o plano segue o contrato.
