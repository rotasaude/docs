# Módulo 35b — `field-sync` v1.4.0 (repo contracts) Implementation Plan

> **Nota do coordenador (Q29, docs d4728f1):** a descrição do `tutor_client_uuid` passa a citar um item `patient` ou um `individual_registration` com `new_patient` do mesmo lote. Entra o código `tutor_not_found` (path `tutor_client_uuid`) em `domains-v1.json`, com o seu par no `check_domains.py`.

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Ordem de entrega (contrato §18):**
> 1. **contracts v1.4.0 sem tag (Tasks 0–4)**;
> 2. SHA-256 ao api e ao campo (Task 5);
> 3. api (copia os esquemas daqui, presos por SHA-256);
> 4. **contracts: SHA-256 conferidos, tag e push (Task 6, com autorização)**;
> 5. dashboard, wpda e maintenance;
> 6. campo (gera os tipos pela tag).
>
> Nenhuma task daqui depende do api do 35b rodando. O `check.sh` usa o `json_schemer` do container `api` do compose.

**Goal:** Publicar o MINOR `field-sync-v1.4.0` (compatível com o v1.3.0) com o lado de campo do 35b (ADR 0054; contrato §11). Ele traz:
- `features.zoonoses` na sessão;
- os itens `animal`, `rabies_vaccination`, `leish_collection` e `collar`, com os resultados deles (`label` da coleta, `version` e `stale_version` do animal);
- as entidades `animal` e `leish_survey` no download e o `removal` delas;
- `zoonoses_domains` na referência;
- a seção `zoonoses` e os códigos novos de `errors.item` em `domains-v1.json`.

**Architecture:** É a mesma mecânica do v1.0.0–v1.3.0, sem ferramenta nova.
- Os esquemas mudam à mão.
- Os exemplos saem de `field-sync/tools/make_examples.py` (determinístico) para `examples/` + `manifest.json`.
- `field-sync/tools/check.sh` valida o manifesto com o `json_schemer` 2.5 do api.
- `field-sync/tools/check_domains.py` confere os `$defs` repetidos iguais, os `enum` contra `domains-v1.json` e os erros conhecidos. Ele ganha os pares `zoonoses.*`.

O esquema confere **forma**: tipos, domínios, faixas, objetos fechados, pares um-ou-outro e as chaves proibidas por modo (criação × edição, comunitário × tutor). As **relações** ficam com o api, que devolve o código de item. São elas:
- espécie × item (coleta e coleira só em cão);
- campanha aberta e inquérito aberto;
- inquérito que cobre o quarteirão;
- janela de datas;
- unicidade de microchip e registro;
- estado do animal;
- CBO.

O dado dessas relações fica em `domains-v1.json`, `zoonoses.rules`. `$defs/animal` é idêntico no download e no `current` do `stale_version`, como o `$defs/household`.

**Tech Stack:** JSON Schema 2020-12; `json_schemer` 2.5 (container `api`); `python3` do host; `shasum`; git em `/opt/homebrew/bin/git`.

**Spec:** `docs/superpowers/specs/2026-11-14-module-35-zoonoses-endemics-design.md` (decisões 16, 20, 23, 29–32); ADR `docs/adr/0054.md`.
- **Contrato (fonte única):** `docs/superpowers/plans/2026-11-23-module-35b-zoonoses-contracts.md`, §11, com §1, §2, §3, §6, §12 e §16; o §19 (Q1–Q27) prevalece; as que este plano aplica são Q10, Q11, Q19–Q23.
- **Base:** `field-sync-v1.3.0` (contracts `c022e80`; `origin/main` em `6ce25c2`).
- **Molde:** `docs/superpowers/plans/2026-11-16-module-35a-vector-control-contracts-repo.md`.
- **Divergências:** as diferenças com o contrato estão em **"Divergências propostas ao contrato"** (D1 a D11), no fim. Todas foram **decididas** no §19 do contrato (Q10–Q27), e o plano aplica as decisões, inclusive onde elas diferem da proposta: `household_ids` no `leish_survey` e lápide de `animal`/`leish_survey` (Q21), `adverse_note` só com `adverse_event` (Q20) e tutor opcional também com domicílio (Q22).

## Global Constraints

- **Repo e base:**
  - repo `contracts` na raiz do monorepo (`$R/contracts`, remote `git@github.com:rotasaude/contracts.git`);
  - `origin/main` em `6ce25c2` ou mais novo;
  - tags `field-sync-v1.0.0` a `-v1.3.0` existem, e nenhuma `field-sync-v1.4.0`.
- **Tag:** `field-sync-v1.4.0`, anotada, com a mensagem `field-sync-v1.4.0 — MINOR`. `session.contract` continua `"field-sync-v1"` (const). Sem entrada no CHANGELOG, não há tag.
- **Compatível com o v1.3.0** (contrato §11):
  - nada que o v1.3.0 aceitava no **pedido** passa a ser recusado (os quatro itens são novos);
  - nas **respostas**, nada novo é obrigatório em tipo antigo (`features.zoonoses`, `zoonoses_domains`, `label` e os tipos novos só aparecem com o 35b);
  - todo exemplo válido do v1.3.0 (61) continua válido;
  - `invalid-upload-unknown-type.json` (`visit_sheet`) continua desconhecido.
- **Convenções de arquivo:**
  - `$id` = `https://rota-saude/schemas/field-sync/<arquivo>` e `title` = `rotasaude.field_sync.<nome>.v1`;
  - objeto fechado em todo nível (`additionalProperties: false`), salvo as seções de dicionário (K-2 do 24b).
- **Tipos novos no fim**, na ordem do contrato:
  - `item.type` e `result.type`: `animal`, `rabies_vaccination`, `leish_collection`, `collar` (no `result`, antes do `null`);
  - `change_item.type`: `animal`, `leish_survey`;
  - `removal.type`: `animal`, `leish_survey`.
- **Valores do contrato, copiados sem mudança:**
  - espécie do animal `dog` `cat` `other`;
  - sexo `male` `female` `unknown`;
  - porte `small` `medium` `large` (nulo no download);
  - estado `active` `deceased` `missing` `transferred`;
  - nome 1–60, pelagem 1–40, raça 1–60, microchip de 15 dígitos e registro 1–20;
  - `adverse_note` 10–500;
  - DPP `reactive` `non_reactive` `invalid`;
  - tipo de inquérito `sample` `census` `focus`;
  - finalidade do produto `rabies_vaccine` `collar`;
  - `collar_months` 1–12.
- **LGPD** (contrato §16, Q9):
  - o número do microchip nunca vai ao app, só `has_microchip`;
  - o tutor sai só como `{ patient_id, display_name }`;
  - nada do 35b no download ou na referência leva CPF, CNS, telefone, endereço de pessoa, notificação ou demanda.

  Os objetos fechados são a guarda, e há exemplo inválido para isso.
- **Dados dos exemplos:** fictícios, como no v1. `street_type` em código (`"081"`), e a guarda `logradouro-em-codigo` continua.
- **`proof`:** o de `domains-v1.json` não muda (`fcf23c87`), como na CV-14 do 35a. A conferência de `zoonoses` contra o api é o `domains_spec` do api do 35b.
- **Worktree fora de `contracts/`:** `$R/.worktrees/contracts-mod35b`, na branch `feat/mod-35b-zoonoses`, a partir de `origin/main` (contrato, valores fixos). Os comandos rodam a partir de `$R`, com o compose de pé (`docker compose ps api` rodando).
  - O worktree `contracts-mod35a` fica de pé: este plano não o usa nem o remove.
- **Testes:** este repo não tem `bin/test`. A verificação é `check.sh` + `check_domains.py`. Nada aqui roda `rspec` nem `vitest`.
- **Git:**
  - sempre `/opt/homebrew/bin/git`; nunca `git add -A` nem `git add .`;
  - commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`;
  - merge, tag e push só com autorização explícita do usuário, uma etapa por vez;
  - antes do push, `origin/main..main` só com os commits desta entrega.
- **Execução coordenada:** ao fim de cada task, uma linha à sessão "Próximos módulos" (`SendMessage`).

## Ambiente de execução

- `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `G=/opt/homebrew/bin/git` e `C=$R/.worktrees/contracts-mod35b`.
- Os caminhos das tasks são relativos a `$C`.
- Verificação (de `$R`):
  - `$C/field-sync/tools/check.sh [esquema]`;
  - `python3 $C/field-sync/tools/check_domains.py [--release]`;
  - `python3 $C/field-sync/tools/make_examples.py`, que regera os exemplos e imprime a contagem por suíte. O número que vale é sempre o impresso.

## Review Focus

1. **Microchip ou dado do tutor no aparelho.** Um `animal` com `microchip`, ou um `tutor` com CPF, quebraria a Q9 e o §16, no download ou no `current` do `stale_version`. Testes (Task 2):
   - `invalid-changes-animal-with-microchip.json` (`/items/1/body/microchip schema`);
   - `invalid-changes-animal-tutor-with-cpf.json` (`/items/1/body/tutor/cpf schema`);
   - `invalid-response-stale-animal-with-microchip.json` (`/results/0/current/microchip schema`).
2. **Derivado do api vindo do app.** A etiqueta da coleta, a modalidade da vacinação, o `neutered` (castração é só do veterinário, §7) ou o vencimento da coleira no corpo seriam uma segunda fonte. Testes (Task 1): `invalid-leish-collection-label-in-body.json`, `invalid-rabies-vaccination-modality-in-body.json`, `invalid-animal-neutered-in-body.json` e `invalid-collar-due-on-in-body.json`.
3. **Modo errado no cadastro do animal.** Os casos são:
   - microchip ou registro na edição (o app não conhece o número, Q9);
   - `status` na criação;
   - tutor num animal comunitário (CHECK do §1);
   - dois tutores;
   - edição sem `version`.

   Testes (Task 1): `invalid-animal-microchip-on-update.json`, `-status-on-create`, `-community-with-tutor`, `-two-tutors` e `-version-without-id`.
4. **Etiqueta ausente ou fora do formato no resultado da coleta.** O app mostraria "Etiqueta depois de enviar" para sempre (§12), ou imprimiria um rótulo que o laboratório não lê. Testes (Task 1): `invalid-response-leish-collection-without-label.json`, `invalid-response-label-lowercase-hex.json` e `invalid-response-label-on-collar.json`.
5. **Produto da zoonose na lista do controle vetorial, ou domínio diferente de `domains-v1.json`.**
   - O api serve hoje todos os produtos ativos em `vector_control_domains.products` (D1). A vacina ou a coleira quebrariam a referência de toda cidade com `vector_control`.
   - Testes (Task 3): `invalid-reference-vector-product-use-collar.json`; os pares `zoonoses.*` e `zoonoses.product_uses(disjoint)` no `check_domains.py`, com o teste do verificador (troque um código, veja `DOMINIOS DIFERENTES`).

   A compatibilidade com o app do 35a (ignora tipo e `removal` desconhecidos; campo `dbc54b3`, `src/sync/download.ts:20-68`) não se testa aqui. Ela vai para o README (Task 4).

## Mapa de arquivos

| Arquivo (worktree `$C`) | Mudança | Task |
|---|---|---|
| `field-sync/session-response-v1.json` | `features.zoonoses` (opcional) | 1 |
| `field-sync/upload-request-v1.json` | itens `animal`, `rabies_vaccination`, `leish_collection`, `collar` e os quatro `*_body` | 1 |
| `field-sync/upload-response-v1.json` | tipos novos, `label`, `version` do animal (1); `stale_version` do animal + `$defs/animal` (2) | 1, 2 |
| `field-sync/changes-response-v1.json` | tipos `animal` e `leish_survey`; `removal` + `animal`, `leish_survey` | 2 |
| `field-sync/reference-response-v1.json` | `zoonoses_domains`; `$defs/date` | 3 |
| `field-sync/domains-v1.json` | seção `zoonoses`; nove códigos em `errors.item` | 3 |
| `field-sync/tools/check_domains.py` | pares `zoonoses.*` | 3 |
| `field-sync/tools/make_examples.py`, `field-sync/examples/*` | exemplos novos | 1–3 |
| `field-sync/README.md`, `field-sync/CHANGELOG.md`, `README.md` (raiz) | regras, entrada v1.4.0, versão na tabela | 4 |
| — | SHA-256 ao api e ao campo, e parada | 5 |
| — | verificação final, merge, tag e push (com autorização) | 6 |

**Estratégia de teste:**
- Cada mudança de esquema entra com seus exemplos válidos e inválidos no gerador **antes** do esquema (Step 1 de cada task). O `check.sh` falha nos novos com o esquema velho e passa com o novo.
- O `check_domains.py` ganha os pares antes dos dados (Task 3).
- **Regra de exemplos do contrato (§11), lida assim (D11):**
  - cada item e cada tipo novo leva ao menos um válido;
  - leva também um inválido **por chave de `required`** de cada objeto fechado novo do corpo (gerado em laço, `invalid-<x>-without-<chave>.json`);
  - e um inválido **por `additionalProperties: false`** (chave estranha escolhida para provar uma guarda);
  - na sessão e na referência, os inválidos são dirigidos, como no 35a.
- **Contagens:**

  | Momento | Casos | Válidos |
  |---|---|---|
  | Hoje | 168 | 61 |
  | Fim da Task 1 | 218 | 72 |
  | Fim da Task 2 | 257 | 75 |
  | Fim da Task 3 | **265** | **77** (188 inválidos) |

  O número exato é o que o `make_examples.py` imprime, e o README o repete.

---

### Task 0: Base, decisões e worktree

**Files:** nenhum.

**Interfaces:**
- Consumes:
  - `contracts` `origin/main`;
  - board #1 (`PVT_kwDOEbfGRc4BbxiB`);
  - o §19 do contrato.
- Produces: `$C` em `feat/mod-35b-zoonoses` e a base anotada.

- [ ] **Step 1: Card** (um `gh` simples por chamada)

```bash
gh project item-list 1 --owner rotasaude --limit 900 --format json --jq '.items[] | select(.title | test("^F-35\\.1[1-8] ")) | {id, title, status}'
```

Expected: os cards F-35.11 a F-35.18 existem.
- Anote o do app de campo e do `field-sync` (o que cita contracts ou campo). Não mova.
- Card ausente, ou nenhum que cite o contracts: **pare e reporte**.

- [ ] **Step 2: Decisões.**
  - Confira que o §19 do contrato tem as linhas Q10–Q27 e que nenhuma Q posterior mudou o que este plano aplica (D1–D11 e as Q11, Q20, Q21 e Q22).
  - Se uma Q nova mudou algo, ajuste a task indicada **antes** de começar e reporte ao coordenador.

- [ ] **Step 3: Base e worktree**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/contracts fetch -q origin
$G -C $R/contracts log --oneline -1 origin/main
$G -C $R/contracts tag -l 'field-sync-*'
$G -C $R/contracts worktree add $R/.worktrees/contracts-mod35b -b feat/mod-35b-zoonoses origin/main
cd $R && $R/.worktrees/contracts-mod35b/field-sync/tools/check.sh | tail -1 && python3 $R/.worktrees/contracts-mod35b/field-sync/tools/check_domains.py --release
$G -C $R/apps/api fetch -q origin
for f in $R/.worktrees/contracts-mod35b/field-sync/*-v1.json; do $G -C $R/apps/api show origin/main:config/field_sync/$(basename $f) | cmp -s - $f && echo "igual $(basename $f)" || echo "DIFERENTE $(basename $f)"; done
```

Expected:
- `6ce25c2 docs: note that a collection without reinstall or removal is refused` ou mais novo;
- tags `v1.0.0` a `v1.3.0`, sem `v1.4.0`;
- `168/168 casos`;
- `defs-iguais`, `dominios-iguais`, `erros-conhecidos`, `regras-da-atividade`, `logradouro-em-codigo` e `prova: fcf23c87 (2026-11-12)`, com exit 0;
- os seis arquivos `igual` à cópia do api na `main`.

Diferente: **pare e reporte** (outro módulo mexeu no `field-sync/`).

**Relato:** "contracts-mod35b Task 0: worktree em feat/mod-35b-zoonoses sobre <sha>; 168/168; card <F-35.x>; D1..D11 conferidas em Q10–Q27."

---

### Task 1: Sessão, os quatro itens do envio e os seus resultados

**Files (worktree `$C`):**
- Modify: `field-sync/session-response-v1.json`, `field-sync/upload-request-v1.json`, `field-sync/upload-response-v1.json`
- Modify: `field-sync/tools/make_examples.py`; regenerate `field-sync/examples/`

**Interfaces:**
- Consumes: os esquemas do v1.3.0; contrato §1 (colunas), §2, §3, §6, §11 e §12; D2, D3, D4, D5, D7 e D8.
- Produces (`field-sync-v1.4.0`):
  - **`session-response-v1.json`:**
    - `features.properties.zoonoses = { "type": "boolean", "description": "Centro de zoonoses (35b): Zoonoses::Gate.field_usable?. Opcional: ausente = false (api anterior ao 35b)." }`;
    - `features.required` **não muda**.
  - **`upload-request-v1.json`:**
    - `item.properties.type.enum` + `"animal"`, `"rabies_vaccination"`, `"leish_collection"`, `"collar"` (no fim);
    - `item.allOf` + um ramo `if type const → then body $ref` para cada um;
    - `$defs` novos (D4, D5, D7):
      ```json
      "animal_body": { "type": "object", "additionalProperties": false,
        "required": ["species", "sex", "community"],
        "properties": {
          "animal_id": { "$ref": "#/$defs/uuid" }, "version": { "type": "integer", "minimum": 1 },
          "household_id": { "$ref": "#/$defs/uuid" }, "household_client_uuid": { "$ref": "#/$defs/uuid" },
          "species": { "enum": ["dog", "cat", "other"] }, "sex": { "enum": ["male", "female", "unknown"] },
          "size": { "enum": ["small", "medium", "large"] },
          "name": { "type": "string", "minLength": 1, "maxLength": 60 },
          "coat": { "type": "string", "minLength": 1, "maxLength": 40 },
          "breed": { "type": "string", "minLength": 1, "maxLength": 60 },
          "birth_estimate_month": { "type": "string", "pattern": "^[0-9]{4}-(0[1-9]|1[0-2])$" },
          "tutor_patient_id": { "$ref": "#/$defs/uuid" }, "tutor_client_uuid": { "$ref": "#/$defs/uuid" },
          "community": { "type": "boolean" },
          "microchip": { "type": "string", "pattern": "^[0-9]{15}$" },
          "registry_number": { "type": "string", "minLength": 1, "maxLength": 20 },
          "status": { "enum": ["active", "deceased", "missing", "transferred"] },
          "status_on": { "$ref": "#/$defs/date" } },
        "dependentRequired": { "animal_id": ["version"], "version": ["animal_id"], "status": ["status_on"], "status_on": ["status"] },
        "oneOf": [{ "required": ["household_id"] }, { "required": ["household_client_uuid"] }],
        "allOf": [
          { "not": { "required": ["tutor_patient_id", "tutor_client_uuid"] } },
          { "if": { "required": ["community"], "properties": { "community": { "const": true } } },
            "then": { "properties": { "tutor_patient_id": false, "tutor_client_uuid": false } } },
          { "if": { "required": ["animal_id"] },
            "then": { "properties": { "microchip": false, "registry_number": false } },
            "else": { "properties": { "status": false, "status_on": false } } } ] },
      "rabies_vaccination_body": { "type": "object", "additionalProperties": false,
        "required": ["campaign_id", "vaccinated_on", "vector_product_id", "adverse_event"],
        "properties": { "animal_id": uuid, "animal_client_uuid": uuid, "campaign_id": uuid, "vaccinated_on": date,
          "vector_product_id": uuid, "adverse_event": { "type": "boolean" },
          "adverse_note": { "type": "string", "minLength": 10, "maxLength": 500 } },
        "oneOf": [{ "required": ["animal_id"] }, { "required": ["animal_client_uuid"] }],
        "if": { "required": ["adverse_event"], "properties": { "adverse_event": { "const": false } } },
        "then": { "properties": { "adverse_note": false } } },
      "leish_collection_body": { "type": "object", "additionalProperties": false, "required": ["collected_on"],
        "properties": { "leish_survey_id": uuid, "animal_id": uuid, "animal_client_uuid": uuid, "collected_on": date,
          "dpp_result": { "enum": ["reactive", "non_reactive", "invalid"] }, "dpp_on": date },
        "dependentRequired": { "dpp_result": ["dpp_on"], "dpp_on": ["dpp_result"] },
        "oneOf": [{ "required": ["animal_id"] }, { "required": ["animal_client_uuid"] }] },
      "collar_body": { "type": "object", "additionalProperties": false, "required": ["vector_product_id", "placed_on"],
        "properties": { "animal_id": uuid, "animal_client_uuid": uuid, "vector_product_id": uuid, "placed_on": date },
        "oneOf": [{ "required": ["animal_id"] }, { "required": ["animal_client_uuid"] }] }
      ```
      (`uuid`/`date` = `{ "$ref": "#/$defs/uuid" }`/`{ "$ref": "#/$defs/date" }`. Os três corpos que citam o animal repetem `animal_id`/`animal_client_uuid` + `oneOf`, como o `ovitrap_collection_body`; não há `$def` comum.)
    - Descrições curtas:
      - **`animal_body`:** "corpo inteiro na criação e na edição (D4): ausente = vazio, salvo microchip e registry_number, que só vão na criação (Q9); household_client_uuid cita um item household aceito antes; tutor opcional também com domicílio (tutor não identificado, Q22), proibido com community true; tutor_client_uuid cita um item patient aceito antes no mesmo lote (24c, Q22); status/status_on só na edição, e o animal que deixa active sai do download (D6); neutered nunca vem (§7)";
      - **`rabies_vaccination_body`:** "modalidade sempre house (§11); adverse_note só com adverse_event true (Q20); falta de estoque não é recusa (§3); a segunda dose na campanha só o api barra (Q19)";
      - **`leish_collection_body`:** "etiqueta dada pelo api no resultado (label, D2); sem leish_survey_id = avulsa";
      - **`collar_body`:** "due_on é do api (placed_on + collar_months)";
      - **`animal_client_uuid`:** "cita um item animal aceito antes, no mesmo lote ou em outro (resolve pelo livro de idempotência; falta → not_found)".
  - **`upload-response-v1.json`:**
    - `result.properties.type.enum` + os quatro (antes do `null`).
    - `result.properties.label = { "type": "string", "pattern": "^LV-[0-9]{4}-([0-9A-F]{6}|AVUL)-[0-9]{4}$", "description": "Etiqueta da coleta (35b §6, D2): só no accepted/duplicate de leish_collection, e obrigatória ali." }`.
    - `result.allOf` + `{ "if": { "required": ["status", "type"], "properties": { "status": { "enum": ["accepted", "duplicate"] }, "type": { "const": "leish_collection" } } }, "then": { "required": ["label"] }, "else": { "properties": { "label": false } } }`.
    - Versão do animal (D3), nos ramos existentes (`upload-response-v1.json:244` e `:246`): `"animal"` entra nas duas listas `["household", "family", "individual_registration"]`. Assim o `accepted` de `animal` exige `version`, o `duplicate` a aceita e os outros tipos novos a proíbem.
    - As descrições de `id` e `version` citam o 35b. O `id` é o do animal, da vacinação, do teste (`leish_tests`) e da coleira. A `version` vale também para o `animal`.
    - O `stale_version` do animal fica para a Task 2.

- [ ] **Step 1: Exemplos primeiro (gerador)**

Em `make_examples.py`:
- **Helpers novos:**
  - `cz(n)` = `f"c0ffee{n:02d}-0000-4000-9000-{n:012d}"` (client_uuid do 35b; o `cu(n)` já usa 1–93);
  - `zitem(n, kind, body)` e `zdone(n, kind, ident, status="accepted")`, iguais a `item`/`done` com `cz`;
  - `zr(n, kind, error, path=None)`, igual a `vr`.
- **Constantes (uuid fixos):**

  | Nome | uuid |
  |---|---|
  | `ANIMAL` | `a7b8c9d0-e1f2-4a3b-8c4d-6e7f8a9b0c1d` |
  | `ANIMAL2` | `b8c9d0e1-f2a3-4b4c-9d5e-7f8a9b0c1d2e` |
  | `ANIMAL3` | `c9d0e1f2-a3b4-4c5d-8e6f-8a9b0c1d2e3f` |
  | `CAMPAIGN` | `d0e1f2a3-b4c5-4d6e-9f7a-9b0c1d2e3f4a` |
  | `LV_SURVEY` | `f1a2b3c4-d5e6-4f7a-8b8c-0d1e2f3a4b5c` (etiqueta `LV-2026-F1A2B3-…`) |
  | `VACCINE` | `e1f2a3b4-c5d6-4e7f-8a9b-1c2d3e4f5a6b` |
  | `COLLAR_PRODUCT` | `f2a3b4c5-d6e7-4f8a-9b0c-2d3e4f5a6b7c` |
  | `VACC` | `0b1c2d3e-4f5a-4b6c-8d7e-3f4a5b6c7d8e` |
  | `LTEST` | `1c2d3e4f-5a6b-4c7d-9e8f-4a5b6c7d8e9f` |
  | `COLLAR` | `2d3e4f5a-6b7c-4d8e-8f9a-5b6c7d8e9f0a` |

- **Corpos:**
  - `animal_new()` = `{"household_id": HH, "species": "dog", "sex": "female", "size": "medium", "name": "Pipoca", "coat": "Caramelo", "breed": "SRD", "birth_estimate_month": "2022-03", "tutor_patient_id": PAT, "community": False, "microchip": "985112003456789"}`;
  - `animal_update()` = `{"animal_id": ANIMAL2, "version": 2, "household_id": HH, "species": "cat", "sex": "male", "name": "Frajola", "community": False, "status": "missing", "status_on": "2026-11-20"}`;
  - `animal_community()` = `{"household_id": HH2, "species": "dog", "sex": "unknown", "community": True}`;
  - `vacc()` = `{"animal_id": ANIMAL, "campaign_id": CAMPAIGN, "vaccinated_on": "2026-11-21", "vector_product_id": VACCINE, "adverse_event": True, "adverse_note": "Inchaço no local da aplicação."}`;
  - `leish()` = `{"leish_survey_id": LV_SURVEY, "animal_id": ANIMAL, "collected_on": "2026-11-21", "dpp_result": "reactive", "dpp_on": "2026-11-21"}`;
  - `collar()` = `{"animal_id": ANIMAL, "vector_product_id": COLLAR_PRODUCT, "placed_on": "2026-11-21"}`.
- **Lote no mesmo envio** (§17, roteiro 9), `upload_zoonoses_same_batch()`:
  1. `zitem(1, "patient", {"new_patient": NEW_PATIENT})`;
  2. `zitem(2, "animal", animal_new() sem microchip, com tutor_client_uuid cz(1) no lugar de tutor_patient_id)`;
  3. `zitem(3, "rabies_vaccination", vacc() com animal_client_uuid cz(2), sem adverse_note, adverse_event False)`;
  4. `zitem(4, "leish_collection", leish() com animal_client_uuid cz(2))`;
  5. `zitem(5, "collar", collar() com animal_client_uuid cz(2))`.
- **`resp_zoonoses_rejected()`:** um `rejected` por código novo de item e os herdados que o 35b usa, `cz(40)` em diante (D8):

| `type` | `error` | `path` |
|---|---|---|
| animal | `microchip_taken` | `microchip` |
| animal | `registry_number_taken` | `registry_number` |
| animal | `animal_not_active` | `animal_id` |
| animal | `cbo_not_allowed` | — |
| animal | `out_of_scope` | `household_id` |
| animal | `not_found` | `tutor_client_uuid` |
| rabies_vaccination | `already_vaccinated_in_campaign` | `animal_id` |
| rabies_vaccination | `campaign_not_open` | `campaign_id` |
| rabies_vaccination | `product_not_allowed` | `vector_product_id` |
| rabies_vaccination | `date_out_of_range` | `vaccinated_on` |
| leish_collection | `survey_not_open` | `leish_survey_id` |
| leish_collection | `household_not_in_survey` | `leish_survey_id` |
| leish_collection | `animal_not_dog` | `animal_id` |
| leish_collection | `cbo_not_allowed` | — |
| leish_collection | `not_found` | `animal_client_uuid` |
| collar | `animal_not_dog` | `animal_client_uuid` |
| collar | `feature_disabled` | — |

- **`resp_zoonoses_accepted()`:**
  - `zdone(2, "animal", ANIMAL)` + `version` 1;
  - `zdone(3, "rabies_vaccination", VACC)`;
  - `zdone(4, "leish_collection", LTEST)` + `label` `"LV-2026-F1A2B3-0001"`;
  - `zdone(5, "collar", COLLAR)`;
  - `zdone(4, "leish_collection", LTEST, "duplicate")` + o mesmo `label`;
  - `zdone(2, "animal", ANIMAL, "duplicate")` sem `version`;
  - `zdone(6, "leish_collection", cz(60))` + `label` `"LV-2026-AVUL-0007"`.
- **Laço de `required`:** para cada corpo novo, `for k in <required do corpo>: "invalid-<item>-without-<k com hífens>.json": (variant(<upload do item>, B + [k], DELETE), "/items/0/body required")`. Os itens e as chaves:
  - `animal`: `species`, `sex`, `community`;
  - `rabies-vaccination`: `campaign_id`, `vaccinated_on`, `vector_product_id`, `adverse_event`;
  - `leish-collection`: `collected_on`;
  - `collar`: `vector_product_id`, `placed_on`.

  São 10 arquivos.
- **Novos** (arquivo → `expect`):

| Suíte | Arquivo | Conteúdo | `expect` |
|---|---|---|---|
| session | `session-zoonoses-agent.json` | `session_agent()` com `vector_control` e `zoonoses` True | `valid` |
| session | `invalid-session-zoonoses-string.json` | `features.zoonoses = "true"` | `/features/zoonoses boolean` |
| upload-request | `upload-animal-new.json` | `up(zitem(10, "animal", animal_new()))` | `valid` |
| upload-request | `upload-animal-update.json` | `up(zitem(11, "animal", animal_update()))` | `valid` |
| upload-request | `upload-animal-community.json` | `up(zitem(12, "animal", animal_community()))` | `valid` |
| upload-request | `upload-zoonoses-same-batch.json` | `upload_zoonoses_same_batch()` | `valid` |
| upload-request | `upload-rabies-vaccination.json` | `up(zitem(13, "rabies_vaccination", vacc()))` | `valid` |
| upload-request | `upload-leish-collection.json` | `up(zitem(14, "leish_collection", leish()))` | `valid` |
| upload-request | `upload-leish-collection-off-survey.json` | `leish()` sem `leish_survey_id`, `dpp_result` e `dpp_on` | `valid` |
| upload-request | `upload-collar.json` | `up(zitem(15, "collar", collar()))` | `valid` |
| upload-request | (laço) `invalid-animal-without-species.json` … `invalid-collar-without-placed-on.json` | 10 arquivos | `/items/0/body required` |
| upload-request | `invalid-animal-neutered-in-body.json` | `animal_new()` + `neutered: True` | `/items/0/body/neutered schema` |
| upload-request | `invalid-rabies-vaccination-modality-in-body.json` | `vacc()` + `modality: "house"` | `/items/0/body/modality schema` |
| upload-request | `invalid-leish-collection-label-in-body.json` | `leish()` + `label: "LV-2026-F1A2B3-0001"` | `/items/0/body/label schema` |
| upload-request | `invalid-collar-due-on-in-body.json` | `collar()` + `due_on: "2027-05-21"` | `/items/0/body/due_on schema` |
| upload-request | `invalid-animal-two-households.json` | `animal_new()` + `household_client_uuid` `cz(30)` | `/items/0/body oneOf` |
| upload-request | `invalid-animal-without-household.json` | `animal_new()` sem `household_id` | `/items/0/body oneOf` |
| upload-request | `invalid-animal-two-tutors.json` | `animal_new()` + `tutor_client_uuid` `cz(1)` | `/items/0/body not` |
| upload-request | `invalid-animal-community-with-tutor.json` | `animal_new()` com `community` True | `/items/0/body/tutor_patient_id schema` |
| upload-request | `invalid-animal-microchip-on-update.json` | `animal_update()` + `microchip` | `/items/0/body/microchip schema` |
| upload-request | `invalid-animal-microchip-14-digits.json` | `microchip = "98511200345678"` | `/items/0/body/microchip pattern` |
| upload-request | `invalid-animal-status-on-create.json` | `animal_new()` + `status` `deceased` + `status_on` | `/items/0/body/status schema` |
| upload-request | `invalid-animal-version-without-id.json` | `animal_update()` sem `animal_id` | `/items/0/body/version dependentRequired` |
| upload-request | `invalid-animal-birth-month-13.json` | `birth_estimate_month = "2022-13"` | `/items/0/body/birth_estimate_month pattern` |
| upload-request | `invalid-animal-name-61-chars.json` | `name = "P" * 61` | `/items/0/body/name maxLength` |
| upload-request | `invalid-rabies-vaccination-two-animals.json` | `vacc()` + `animal_client_uuid` `cz(2)` | `/items/0/body oneOf` |
| upload-request | `invalid-rabies-vaccination-without-animal.json` | `vacc()` sem `animal_id` | `/items/0/body oneOf` |
| upload-request | `invalid-rabies-vaccination-note-short.json` | `adverse_note = "Inchaço."` | `/items/0/body/adverse_note minLength` |
| upload-request | `invalid-rabies-vaccination-note-without-event.json` | `vacc()` com `adverse_event` False (Q20) | `/items/0/body/adverse_note schema` |
| upload-request | `invalid-leish-collection-without-animal.json` | `leish()` sem `animal_id` | `/items/0/body oneOf` |
| upload-request | `invalid-leish-collection-dpp-without-date.json` | `leish()` sem `dpp_on` | `/items/0/body/dpp_result dependentRequired` |
| upload-request | `invalid-leish-collection-dpp-inconclusive.json` | `dpp_result = "inconclusive"` (é do ELISA) | `/items/0/body/dpp_result enum` |
| upload-request | `invalid-collar-two-animals.json` | `collar()` + `animal_client_uuid` `cz(2)` | `/items/0/body oneOf` |
| upload-request | `invalid-collar-without-animal.json` | `collar()` sem `animal_id` | `/items/0/body oneOf` |
| upload-response | `upload-response-zoonoses-accepted.json` | `resp_zoonoses_accepted()` | `valid` |
| upload-response | `upload-response-zoonoses-rejected.json` | `resp_zoonoses_rejected()` | `valid` |
| upload-response | `invalid-response-leish-collection-without-label.json` | `zdone(4, "leish_collection", LTEST)` sem `label` | `/results/0 required` |
| upload-response | `invalid-response-label-on-collar.json` | `zdone(5, "collar", COLLAR)` + `label` | `/results/0/label schema` |
| upload-response | `invalid-response-label-lowercase-hex.json` | `label = "LV-2026-f1a2b3-0001"` | `/results/0/label pattern` |
| upload-response | `invalid-response-animal-accepted-without-version.json` | `zdone(2, "animal", ANIMAL)` | `/results/0 required` |
| upload-response | `invalid-response-version-on-vaccination.json` | `zdone(3, "rabies_vaccination", VACC)` + `version` 1 | `/results/0/version schema` |

Rode `python3 $C/field-sync/tools/make_examples.py`. Expected: `session-response-v1.json 12 · upload-request-v1.json 121 · upload-response-v1.json 48 · changes-response-v1.json 26 · reference-response-v1.json 11 · total 218`.

- [ ] **Step 2: Rode e veja falhar**

Run: `cd $R && $C/field-sync/tools/check.sh | grep '^FAIL' | wc -l`

Expected:
- os válidos novos falham (`schema` em `zoonoses` e `label`, `enum` no `type`);
- os inválidos novos falham por outro motivo (`enum` no `type` no lugar do `expect`);
- nenhum dos 168 antigos falha.

- [ ] **Step 3: Mude os três esquemas** pelas Interfaces.

- [ ] **Step 4: Rode e veja passar**

Run: `cd $R && $C/field-sync/tools/check.sh | tail -1 && python3 $C/field-sync/tools/check_domains.py`

Expected:
- `218/218 casos`; `defs-iguais`; `dominios-iguais`;
- `ERROS FORA DA LISTA: upload-response-zoonoses-rejected.json: microchip_taken; …`, com os nove códigos novos. É exit 1: a Task 3 os acrescenta, e esse é o teste que falha dela.

Se um `expect` sair com outro `type` do `json_schemer` (o `not` dos tutores, o `false` dentro de `if/then`), ajuste só o `expect` e anote no relato.

- [ ] **Step 5: Commit**

```bash
cd $C && $G add field-sync/session-response-v1.json field-sync/upload-request-v1.json field-sync/upload-response-v1.json field-sync/tools/make_examples.py field-sync/examples
$G status --short
$G commit -m "feat: add the animal, rabies vaccination, leishmaniasis collection and collar items to field sync uploads" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato:** "contracts-mod35b Task 1: features.zoonoses, quatro itens (animal com corpo inteiro, microchip só na criação), label na coleta e version do animal; 218/218; códigos novos entram na Task 3."

---

### Task 2: Animal e inquérito no download; `stale_version` do animal

**Files (worktree `$C`):**
- Modify: `field-sync/changes-response-v1.json`, `field-sync/upload-response-v1.json`
- Modify: `field-sync/tools/make_examples.py`; regenerate `field-sync/examples/`

**Interfaces:**
- Consumes: contrato §11 (corpos do download), §2 (tutor), §6 (coleira), Q9; D3, D6 e D10.
- Produces:
  - **`$defs/animal`, nos dois arquivos, byte a byte** (o `check_domains.py` cobra):
    ```json
    "animal": { "type": "object", "additionalProperties": false,
      "description": "Animal do domicílio (35b §11). Nunca o número do microchip (Q9): só has_microchip. Tutor só { patient_id, display_name }. last_rabies_vaccination_on e collar_due_on são derivados: mudam sem version nova (D10).",
      "required": ["id", "version", "household_id", "species", "name", "sex", "size", "coat", "breed", "birth_estimate_on", "status", "neutered", "tutor", "community", "last_rabies_vaccination_on", "collar_due_on", "has_microchip"],
      "properties": {
        "id": { "$ref": "#/$defs/uuid" }, "version": { "type": "integer", "minimum": 1 },
        "household_id": { "$ref": "#/$defs/uuid" },
        "species": { "enum": ["dog", "cat", "other"] },
        "name": { "type": ["string", "null"], "minLength": 1, "maxLength": 60 },
        "sex": { "enum": ["male", "female", "unknown"] },
        "size": { "enum": ["small", "medium", "large", null] },
        "coat": { "type": ["string", "null"], "minLength": 1, "maxLength": 40 },
        "breed": { "type": ["string", "null"], "minLength": 1, "maxLength": 60 },
        "birth_estimate_on": { "$ref": "#/$defs/date_or_null" },
        "status": { "enum": ["active", "deceased", "missing", "transferred"] },
        "neutered": { "type": "boolean" },
        "tutor": { "anyOf": [
          { "type": "object", "additionalProperties": false, "required": ["patient_id", "display_name"],
            "properties": { "patient_id": { "$ref": "#/$defs/uuid" }, "display_name": { "type": "string", "minLength": 1 } } },
          { "type": "null" }] },
        "community": { "type": "boolean" },
        "last_rabies_vaccination_on": { "$ref": "#/$defs/date_or_null" },
        "collar_due_on": { "$ref": "#/$defs/date_or_null" },
        "has_microchip": { "type": "boolean" } } }
    ```
  - **`changes-response-v1.json`:**
    - `change_item.properties.type.enum` + `"animal"`, `"leish_survey"`;
    - `change_item.allOf` + dois ramos `{ "if": { type const X }, "then": { "required": ["version"], "properties": { "body": { "$ref": "#/$defs/X" } } } }`;
    - `$defs/leish_survey` fechado, com todas as chaves em `required`:
      - `id` uuid;
      - `version` integer ≥ 1;
      - `name` `{ "type": "string", "minLength": 1, "maxLength": 80 }`;
      - `kind` `{ "enum": ["sample", "census", "focus"] }`;
      - `household_ids` `{ "type": "array", "minItems": 1, "uniqueItems": true, "items": uuid }` (Q21: os domicílios do escopo do agente nos quarteirões do inquérito; o tablet não depende do quarteirão do 35a);
      - `starts_on` date;
      - `ends_on` date.

      A descrição: "Só inquéritos open com domicílio no escopo do agente (Q21); household_ids = esses domicílios. O fechamento e a saída do escopo vêm por removal out_of_scope (Q21). Sem quarteirão, população, meta nem resultado."
    - `removal.properties.type.enum` + `"animal"`, `"leish_survey"`. O `reason` **não muda**.
    - A descrição do `removal` ganha: "animal e leish_survey (35b): sempre out_of_scope; o animal sai quando deixa o escopo ou o estado active, e o inquérito quando fecha ou deixa o escopo (Q21)."
  - **`upload-response-v1.json`:**
    - no ramo `stale_version` (`:237`), `type.enum` + `"animal"`;
    - ramo novo `{ "if": { status const stale_version, type const animal }, "then": { "properties": { "current": { "$ref": "#/$defs/animal" } } } }`;
    - a descrição de `status` cita o animal (D3).

- [ ] **Step 1: Exemplos primeiro (gerador)**

**Funções:**
- `animal()` = `{"id": ANIMAL, "version": 2, "household_id": HH, "species": "dog", "name": "Pipoca", "sex": "female", "size": "medium", "coat": "Caramelo", "breed": "SRD", "birth_estimate_on": "2022-03-01", "status": "active", "neutered": False, "tutor": {"patient_id": PAT, "display_name": "Joana Conceição"}, "community": False, "last_rabies_vaccination_on": "2026-11-21", "collar_due_on": "2026-12-01", "has_microchip": True}`;
- `animal_community_body()` = `animal()` com `id` ANIMAL2, `version` 1, `name`/`size`/`coat`/`breed`/`birth_estimate_on`/`last_rabies_vaccination_on`/`collar_due_on` None, `tutor` None, `community` True e `has_microchip` False;
- `leish_survey()` = `{"id": LV_SURVEY, "version": 1, "name": "Inquérito LV Vila Nova 2026", "kind": "sample", "household_ids": [HH, HH2], "starts_on": "2026-11-03", "ends_on": "2026-12-12"}`.

**Páginas:**
- `page_zoonoses()`: `reset` False, `has_more` True, `who_to_visit` None. Itens:
  0. `household` HH v3 (`household()`);
  1. `animal` ANIMAL v2;
  2. `animal` ANIMAL2 v1 (comunitário);
  3. `leish_survey` v1.
- `page_zoonoses_removals()`: última página, `removals` `[{"type": "animal", "id": ANIMAL3, "reason": "out_of_scope"}, {"type": "leish_survey", "id": LV_SURVEY, "reason": "out_of_scope"}]` e `who_to_visit()`.

**Laços de `required`:**
- animal: `for k in animal_required: "invalid-changes-animal-without-<k>.json": (variant(page_zoonoses, ["items", 1, "body", k], DELETE), "/items/1/body required")`, com 17 arquivos;
- tutor: `for k in ["patient_id", "display_name"]: "invalid-changes-animal-tutor-without-<k>.json"` → `/items/1/body/tutor required`, com 2;
- inquérito: `for k in leish_required: "invalid-changes-leish-survey-without-<k>.json"` → `/items/3/body required`, com 7.

| Suíte | Arquivo | Conteúdo | `expect` |
|---|---|---|---|
| changes | `changes-zoonoses-page.json` | `page_zoonoses()` | `valid` |
| changes | `changes-zoonoses-removals.json` | `page_zoonoses_removals()` | `valid` |
| changes | (laços) 26 arquivos | acima | acima |
| changes | `invalid-changes-animal-with-microchip.json` | `items[1].body.microchip = "985112003456789"` | `/items/1/body/microchip schema` |
| changes | `invalid-changes-animal-tutor-with-cpf.json` | `items[1].body.tutor.cpf = "52998224725"` | `/items/1/body/tutor/cpf schema` |
| changes | `invalid-changes-leish-survey-with-target-dogs.json` | `items[3].body.target_dogs = 180` | `/items/3/body/target_dogs schema` |
| changes | `invalid-changes-animal-species-bird.json` | `items[1].body.species = "bird"` | `/items/1/body/species enum` |
| changes | `invalid-changes-animal-without-item-version.json` | sem `items[1].version` | `/items/1 required` |
| changes | `invalid-changes-leish-survey-without-item-version.json` | sem `items[3].version` | `/items/3 required` |
| changes | `invalid-changes-leish-survey-no-households.json` | `items[3].body.household_ids = []` | `/items/3/body/household_ids minItems` |
| changes | `invalid-changes-removal-animal-deceased.json` | `removals[0].reason = "deceased"` | `/removals/0/reason enum` |
| upload-response | `upload-response-stale-animal.json` | `{"client_uuid": cz(11), "status": "stale_version", "type": "animal", "id": ANIMAL, "current_version": 3, "current": animal() com version 3}` | `valid` |
| upload-response | `invalid-response-stale-animal-with-microchip.json` | o anterior + `current.microchip` | `/results/0/current/microchip schema` |
| upload-response | `invalid-response-stale-vaccination.json` | o anterior com `type` `rabies_vaccination` | `/results/0/type enum` |

Rode o gerador. Expected: `… · upload-response-v1.json 51 · changes-response-v1.json 62 · … · total 257`.

- [ ] **Step 2: Rode e veja falhar.** `cd $R && $C/field-sync/tools/check.sh changes-response-v1.json upload-response-v1.json | grep -c '^FAIL'`. Expected: os 3 válidos e os 36 inválidos novos falham (`enum` no `type`, `schema` no `current`).

- [ ] **Step 3: Mude os dois esquemas** pelas Interfaces. `$defs/animal` é copiado byte a byte entre os dois arquivos.

- [ ] **Step 4: Rode e veja passar.** `cd $R && $C/field-sync/tools/check.sh | tail -1 && python3 $C/field-sync/tools/check_domains.py | head -2`. Expected: `257/257 casos`; `defs-iguais`; `dominios-iguais`. Se os inválidos do `tutor` saírem com `anyOf` em `/items/1/body/tutor` no lugar do erro de dentro, ajuste só o `expect` e anote (no 35a, `vector` saiu com o erro de dentro).

- [ ] **Step 5: Commit**

```bash
cd $C && $G add field-sync/changes-response-v1.json field-sync/upload-response-v1.json field-sync/tools/make_examples.py field-sync/examples
$G commit -m "feat: add animals and leishmaniasis surveys to the field sync feed" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato:** "contracts-mod35b Task 2: animal (sem microchip, tutor só id e nome) e leish_survey no download, removal das duas, stale_version do animal; 257/257."

---

### Task 3: Domínios de zoonoses na referência e em `domains-v1.json`

**Files (worktree `$C`):**
- Modify: `field-sync/reference-response-v1.json`, `field-sync/domains-v1.json`, `field-sync/tools/check_domains.py`, `field-sync/tools/make_examples.py`; regenerate `field-sync/examples/`

**Interfaces:**
- Consumes: os `$defs` das Tasks 1–2; contrato §3, §6, §11 e §12; D1, D8 e D9.
- Produces:
  - **`domains-v1.json`**, chave nova `zoonoses` (depois de `vector_control`). As listas `{ code, label }` vão na ordem abaixo, com os rótulos da D9:

    | Chave | Códigos e rótulos |
    |---|---|
    | `species` | `dog` "Cão"; `cat` "Gato"; `other` "Outro" |
    | `sexes` | `male` "Macho"; `female` "Fêmea"; `unknown` "Não informado" |
    | `sizes` | `small` "Pequeno"; `medium` "Médio"; `large` "Grande" |
    | `animal_statuses` | `active` "Ativo"; `deceased` "Óbito"; `missing` "Desaparecido"; `transferred` "Transferido" |
    | `dpp_results` | `reactive` "Reagente"; `non_reactive` "Não reagente"; `invalid` "Inválido" |
    | `leish_survey_kinds` | `sample` "Amostral"; `census` "Censitário"; `focus` "Foco" |
    | `product_uses` | `rabies_vaccine` "Vacina antirrábica animal"; `collar` "Coleira inseticida" |

    mais estes dois blocos:
    ```json
    "limits": { "name_max": 60, "coat_max": 40, "breed_max": 60, "microchip_digits": 15, "registry_number_max": 20,
                "adverse_note_min": 10, "adverse_note_max": 500 },
    "rules": {
      "registrar_cbos": ["515140", "515105"],
      "leish_collection_cbos": ["515140"],
      "dog_only": ["leish_collection", "collar"],
      "vaccination_modality": "house",
      "collar_due_window_days": 15,
      "adverse_note_requires_event": true,
      "leish_label": "LV-<ano>-<6 primeiros hex do id do inquérito, maiúsculos>-<seq de 4 dígitos>; fora de inquérito, LV-<ano>-AVUL-<seq de 4 dígitos>",
      "microchip_on_create_only": true,
      "stock_shortfall_rejects": false }
    ```
  - **`domains-v1.json` `errors.item`:** entram no fim, nesta ordem, `microchip_taken`, `registry_number_taken`, `already_vaccinated_in_campaign`, `campaign_not_open`, `survey_not_open`, `animal_not_dog`, `animal_not_active` (os sete do §11) e `date_out_of_range`, `household_not_in_survey` (D8). A `description` do arquivo cita o 35b.
  - **`reference-response-v1.json`:**
    - `properties.zoonoses_domains = { "$ref": "#/$defs/zoonoses_domains", "description": "Só com Zoonoses::Gate.field_usable? (§11). open_campaign muda o reference_version (D9)." }`, fora do `required`;
    - `$defs/date` **byte a byte** o dos outros arquivos;
    - e o bloco:
    ```json
    "zoonoses_domains": { "type": "object", "additionalProperties": false,
      "required": ["species", "sexes", "sizes", "animal_statuses", "dpp_results", "products", "open_campaign", "parameters", "limits"],
      "properties": {
        "species": { "type": "array", "minItems": 1, "items": { "$ref": "#/$defs/string_entry" } },
        "sexes": …, "sizes": …, "animal_statuses": …, "dpp_results": … (iguais a species),
        "products": { "type": "array", "items": { "type": "object", "additionalProperties": false,
          "required": ["id", "code", "name", "use"],
          "properties": { "id": { "$ref": "#/$defs/uuid" }, "code": { "type": "string", "minLength": 1, "maxLength": 10 },
            "name": { "type": "string", "minLength": 1, "maxLength": 80 }, "use": { "enum": ["rabies_vaccine", "collar"] } } } },
        "open_campaign": { "anyOf": [
          { "type": "object", "additionalProperties": false, "required": ["id", "name", "starts_on", "ends_on"],
            "properties": { "id": { "$ref": "#/$defs/uuid" }, "name": { "type": "string", "minLength": 1, "maxLength": 80 },
              "starts_on": { "$ref": "#/$defs/date" }, "ends_on": { "$ref": "#/$defs/date" } } },
          { "type": "null" }] },
        "parameters": { "type": "object", "additionalProperties": false, "required": ["collar_months"],
          "properties": { "collar_months": { "type": "integer", "minimum": 1, "maximum": 12 } } },
        "limits": { "type": "object", "additionalProperties": false,
          "required": ["name_max", "coat_max", "breed_max", "microchip_digits", "registry_number_max", "adverse_note_min", "adverse_note_max"],
          "properties": { cada uma { "type": "integer", "minimum": 1 } } } } }
    ```
    - `vector_control_domains` **não muda** (D1): `products[].uses` continua `focal` `perifocal` `spatial`.
  - **`check_domains.py`:** usa `zo = dom["zoonoses"]`, `ab = req["animal_body"]["properties"]`, `an = chg["animal"]["properties"]` e `zd = s["reference-response-v1.json"]["$defs"]["zoonoses_domains"]["properties"]`. Os pares novos (nome → domínio × esquema):
    - **listas:**
      - `zoonoses.species` × `ab.species.enum` e × `an.species.enum`;
      - `.sexes` × `ab.sex` e × `an.sex`;
      - `.sizes` × `ab.size` e × `nn(an.size)`;
      - `.animal_statuses` × `ab.status` e × `an.status`;
      - `.dpp_results` × `req.leish_collection_body.dpp_result.enum`;
      - `.leish_survey_kinds` × `chg.leish_survey.kind.enum`;
      - `.product_uses` × `zd.products.items.use.enum`;
      - `zoonoses.product_uses(disjoint)`: a interseção dos códigos de `product_uses` com `vcd.products.items.uses.items.enum` é `[]` (D1).
    - **`zoonoses.limits`:** `name_max` = `ab.name.maxLength` = `an.name.maxLength`. O mesmo vale para `coat_max` e `breed_max`. Também: `registry_number_max` = `ab.registry_number.maxLength`; `adverse_note_min`/`max` = `minLength`/`maxLength` de `rabies_vaccination_body.adverse_note`; e `ab.microchip.pattern` = `f"^[0-9]{{{microchip_digits}}}$"`.
    - **`zoonoses.rules`:**
      - `registrar_cbos` ⊆ `home_visit.rules.registrar_cbos`;
      - `leish_collection_cbos` ⊆ `registrar_cbos`;
      - `dog_only` ⊆ `req.item.type.enum`.

    A docstring ganha "zoonoses (35b)".

- [ ] **Step 1: Pares primeiro.** Escreva os pares de `check_domains.py` pelas Interfaces.

- [ ] **Step 2: Rode e veja falhar.** `python3 $C/field-sync/tools/check_domains.py; echo "exit $?"`. Expected: `KeyError: 'zoonoses'` (exit ≠ 0).

- [ ] **Step 3: Dados e exemplos.**
  - Escreva `zoonoses` e os nove códigos em `domains-v1.json`.
  - No gerador, `zoonoses_domains()` = as cinco listas **lidas de `domains-v1.json`** (`species` … `dpp_results`; uma fonte só) e mais:
    - `products`: `[{"id": VACCINE, "code": "VAR", "name": "Vacina antirrábica animal", "use": "rabies_vaccine"}, {"id": COLLAR_PRODUCT, "code": "CDT4", "name": "Coleira deltametrina 4%", "use": "collar"}]`;
    - `open_campaign`: `{"id": CAMPAIGN, "name": "Campanha antirrábica 2026", "starts_on": "2026-11-03", "ends_on": "2026-11-28"}`;
    - `parameters`: `{"collar_months": 6}`;
    - `limits` = `zoonoses.limits` lido do arquivo.
  - `reference_zoonoses()` = `reference()` + `zoonoses_domains()`, sem `vector_control_domains`: o 35b funciona sem o 35a (§1).

| Suíte | Arquivo | Conteúdo | `expect` |
|---|---|---|---|
| reference | `reference-zoonoses.json` | `reference_zoonoses()` | `valid` |
| reference | `reference-zoonoses-vector-control.json` | `reference_vector()` + `zoonoses_domains()` com `open_campaign` None | `valid` |
| reference | `invalid-reference-zoonoses-without-open-campaign.json` | sem `zoonoses_domains.open_campaign` (ausente ≠ null) | `/zoonoses_domains required` |
| reference | `invalid-reference-zoonoses-product-use-focal.json` | `products[0].use = "focal"` | `/zoonoses_domains/products/0/use enum` |
| reference | `invalid-reference-zoonoses-collar-months-13.json` | `parameters.collar_months = 13` | `/zoonoses_domains/parameters/collar_months maximum` |
| reference | `invalid-reference-zoonoses-campaign-with-goal.json` | `open_campaign.goal_dogs = 1200` | `/zoonoses_domains/open_campaign/goal_dogs schema` |
| reference | `invalid-reference-zoonoses-species-empty.json` | `species = []` | `/zoonoses_domains/species minItems` |
| reference | `invalid-reference-vector-product-use-collar.json` | `reference_vector()` com `vector_control_domains.products[0].uses = ["collar"]` | `/vector_control_domains/products/0/uses/0 enum` |

Rode o gerador (`… · reference-response-v1.json 19 · total 265`) e `cd $R && $C/field-sync/tools/check.sh reference-response-v1.json`.
- Expected: os dois válidos e os cinco inválidos de `zoonoses_domains` dão `FAIL` (`/zoonoses_domains schema`).
- O `invalid-reference-vector-product-use-collar.json` já passa: ele prova a D1 sem mudar o esquema.
- O `check_domains.py` já mostra `dominios-iguais` e `erros-conhecidos`.

- [ ] **Step 4: Mude `reference-response-v1.json`** pelas Interfaces.

- [ ] **Step 5: Rode e veja passar**

Run: `cd $R && $C/field-sync/tools/check.sh | tail -1 && python3 $C/field-sync/tools/check_domains.py --release; echo "exit $?"`

Expected: `265/265 casos`; `defs-iguais`, `dominios-iguais`, `erros-conhecidos`, `regras-da-atividade`, `logradouro-em-codigo`, `prova: fcf23c87 (2026-11-12)` e `exit 0`.

Teste do verificador, desfazendo cada troca depois:
1. troque `unknown` por `unk` em `zoonoses.sexes`: veja `DOMINIOS DIFERENTES: zoonoses.sexes` e exit 1;
2. troque `limits.name_max` para 61: veja o mesmo erro com `zoonoses.limits`;
3. acrescente `"collar"` ao `uses.items.enum` de `vector_control_domains`: veja `zoonoses.product_uses(disjoint)` e `vector_control.measure_methods(reference)`.

- [ ] **Step 6: Commit**

```bash
cd $C && $G add field-sync/reference-response-v1.json field-sync/domains-v1.json field-sync/tools/check_domains.py field-sync/tools/make_examples.py field-sync/examples
$G commit -m "feat: add zoonoses domains and item error codes to field sync reference and domains" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato:** "contracts-mod35b Task 3: zoonoses_domains na referência (open_campaign nulo, produtos com use), zoonoses em domains-v1 (rótulos D9, limits, rules) e 9 códigos; vector_control_domains intacto (D1); 265/265 (77 válidos); --release verde."

---

### Task 4: README e CHANGELOG

**Files:** Modify `field-sync/README.md`, `field-sync/CHANGELOG.md`, `README.md` (raiz).

**Interfaces:**
- Consumes: Tasks 1–3.
- Produces: o texto que o app e o api citam, e a tabela da raiz com `field-sync-v1.4.0`.

- [ ] **Step 1: `field-sync/README.md`.** A tabela de arquivos cita o 35b na linha de `domains-v1.json`. As regras são curtas, uma por item, sem repetir o esquema, e todas marcadas "(v1.4.0)":
  - **Interruptor:** `features.zoonoses` = `Zoonoses::Gate.field_usable?` (ausente = false). Desligado: os itens voltam `rejected feature_disabled`, o app esconde as telas e a fila espera (Q6 do 35a). O que foi gravado fica.
  - **Compatibilidade (§11):**
    - o api emite `animal`, `leish_survey` e `zoonoses_domains` só com `field_usable?`;
    - ligar ou desligar o interruptor chega a todo aparelho da cidade como `reset: true`;
    - o app do 35a ignora tipo e `removal` desconhecidos no download e não valida a sessão nem a referência em execução: ele convive com a cidade e só não mostra os animais;
    - o app anterior ao do 35a quebra. Por isso **nenhum aparelho com app anterior ao do 35a pode estar numa cidade com `zoonoses` ligado** (gate de go-live);
    - o api aceita o envio da v1.3.0 sem mudança.
  - **Animal (D4, D5):**
    - o corpo é inteiro na criação e na edição;
    - a edição leva `animal_id` + `version` (vencida → `stale_version` com o `<animal>` atual, D3);
    - `microchip` e `registry_number` só na criação: o app não recebe o número (Q9) e não o reenvia;
    - `status`/`status_on` só na edição, e quem deixa `active` sai do download;
    - comunitário sem tutor;
    - o tutor é opcional também com domicílio (tutor não identificado); é `tutor_patient_id` (o 515140 escolhe entre os moradores que o tablet tem) ou `tutor_client_uuid` de um item `patient` do mesmo lote (24c; Q22); `community: true` = ninguém responde pelo animal;
    - `neutered` nunca vem (castração é do veterinário, §7);
    - `birth_estimate_month` `"YYYY-MM"`; o api grava o dia 1, e o download traz `birth_estimate_on`;
    - recusas: `microchip_taken`, `registry_number_taken`, `animal_not_active`, `out_of_scope` e `not_found` (`tutor_client_uuid`).
  - **Vacinação:**
    - a modalidade é sempre `house`, e a campanha é o `open_campaign` da referência (`campaign_not_open`);
    - uma dose por animal e campanha (`already_vaccinated_in_campaign`): só o api barra, porque o tablet não sabe a campanha da última dose (Q19);
    - `adverse_note` só com `adverse_event: true` (Q20);
    - data fora da janela → `date_out_of_range`;
    - produto sem `rabies_vaccine` → `product_not_allowed`;
    - o `adverse_event` mostra no app o lembrete do EATAV (§3);
    - **a falta de estoque não é recusa**: o item volta `accepted`.
  - **Coleta de LV:**
    - só o CBO 515140 (`cbo_not_allowed`) e só cão (`animal_not_dog`);
    - o inquérito precisa estar `open` (`survey_not_open`) e cobrir o quarteirão do domicílio (`household_not_in_survey`); sem `leish_survey_id`, a coleta é avulsa;
    - `dpp_result` e `dpp_on` vão juntos;
    - a etiqueta é do api e volta em `label`, obrigatória no `accepted` e no `duplicate` (Q21, formato em `zoonoses.rules.leish_label`); até lá, o app mostra "Etiqueta depois de enviar".
  - **Coleira:** só cão; o `due_on` é do api; a coleira ativa anterior sai `replaced` na mesma transação (§6); a falta de estoque não é recusa. As coleiras a trocar (`collar_due_on` em até 15 dias) têm lista própria no app, fora de "Imóveis a visitar" (Q23).
  - **Referência do item:** `animal_client_uuid` resolve pelo livro de idempotência (falta → `not_found`). O `id` do resultado é o do animal, da vacinação, do teste e da coleira.
  - **Download:**
    - `animal` vem só pelo domicílio do escopo (sem domicílio, não vai) e só `active`;
    - `leish_survey` vem só `open`, com `household_ids` = os domicílios do escopo nos quarteirões dele (Q21); a coleta só aparece nesses domicílios;
    - a saída das duas vem por `removal` `out_of_scope` (Q21);
    - o app **sobrescreve** `animal` e `leish_survey` a cada recebimento, qualquer que seja a `version`, como na Q29: `last_rabies_vaccination_on` e `collar_due_on` mudam sem `version` nova (D10). A fila local nunca é apagada;
    - nunca o número do microchip, e o tutor só com id e nome.
  - **Referência:**
    - `zoonoses_domains` só com `field_usable?`;
    - `open_campaign` `null` sem campanha aberta; abrir ou fechar a campanha muda o `reference_version`, e o app baixa de novo (cache de 5 min do api);
    - `vector_control_domains.products` traz só produtos de controle vetorial: a vacina e a coleira vêm só em `zoonoses_domains.products` (D1).
  - A contagem dos exemplos (a do gerador: 265, 77 válidos) e os comandos com `265/265`.
  - **Consumo:** o app gera os tipos pela tag `field-sync-v1.4.0`.

- [ ] **Step 2: `field-sync/CHANGELOG.md`**, no topo:
  - o título `## field-sync-v1.4.0 — (não publicada até a tag) — MINOR`;
  - a frase "Centro de zoonoses (módulo 35b, ADR 0054). Compatível com o v1.3.0: nada do pedido v1.3.0 passa a ser recusado; o novo nas respostas é opcional ou só de tipo novo.";
  - uma linha por arquivo, com o conteúdo das Tasks 1–3;
  - a linha "Exemplos: 265 (77 válidos, 188 inválidos)".

- [ ] **Step 3: `README.md` da raiz.**
  - Na linha de `field-sync/`: `field-sync-v1.3.0` → `field-sync-v1.4.0`, e a descrição ganha "centro de zoonoses".
  - No parágrafo `### field-sync`, a frase "desde o v1.4.0 (ADR 0054), também o animal com tutor, a vacinação antirrábica casa a casa, a coleta do inquérito de leishmaniose e a coleira".

- [ ] **Step 4: Verificação**

Run: `cd $R && $C/field-sync/tools/check.sh | tail -1 && python3 $C/field-sync/tools/check_domains.py --release && grep -c "birth_estimate_month\|has_microchip\|household_not_in_survey\|Etiqueta depois de enviar\|out_of_scope\|não é recusa\|reset: true" $C/field-sync/README.md`

Expected: `265/265 casos`; exit 0; contagem ≥ 7.

- [ ] **Step 5: Commit**

```bash
cd $C && $G add field-sync/README.md field-sync/CHANGELOG.md README.md
$G commit -m "docs: document field-sync v1.4.0 rules and changelog" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato:** "contracts-mod35b Task 4: README (regras v1.4.0, gate de go-live dos aparelhos) e CHANGELOG; 265/265."

---

### Task 5: SHA-256 ao api e ao campo, e parada

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

Expected:
- `265/265`; exit 0;
- quatro commits;
- nada fora de `field-sync/` e do `README.md`;
- seis SHA-256.

- [ ] **Step 2: Pare.** Sem merge nem push. Mande ao coordenador o `HEAD`, o caminho `$C/field-sync/` e os seis SHA-256 (para o api e para o campo). Mande também o que o api tem de mudar junto:
  1. `FieldSync::Schemas::SHA256` (`app/services/field_sync/schemas.rb:14`) e os literais de `spec/services/field_sync/schemas_spec.rb`: 168 → 265 exemplos (`:41`) e 61 → 77 válidos (`:54`);
  2. a cópia de `examples/` em `spec/fixtures/field_sync/examples/` (265 arquivos + `manifest.json`);
  3. `spec/services/field_sync/domains_spec.rb`: `zoonoses` contra o catálogo do api (códigos, rótulos, ordem, `limits`) e os nove códigos em `errors.item`;
  4. `FieldSync::Catalog::ITEM_TYPES` (`app/services/field_sync/catalog.rb:12`) + `animal`, `rabies_vaccination`, `leish_collection`, `collar`, no fim. O exemplo que fixa `.last` muda. A ordem do api já difere da do esquema em `family`, e isso não muda;
  5. as decisões que só o api cumpre:
     - Q11: filtrar `vector_control_domains.products` aos vetoriais (`app/services/vector_control/json.rb:226-229`); produto de zoonoses com uma finalidade só, `unit` `un` e `charge_size` 1.00; o gate `VectorControl::Gate.supplies_usable?`;
     - Q21: `label` (hex maiúsculo) no `accepted` e no `duplicate`, relido do teste;
     - Q20: `version` no `accepted` do animal e `stale_version` com `current_version`, sem mescla; `adverse_note` só com `adverse_event`;
     - Q21: `removal` `out_of_scope` de `animal` e `leish_survey`, com `TOMBSTONE_ENTITY_TYPES` (`catalog.rb:22`) + `animal` e `leish_survey`; `household_ids` no `leish_survey`;
     - Q20: os `path` e os dois códigos novos;
     - Q20: reemitir o `animal` quando a vacinação ou a coleira mudam;
     - Q10/§11: `features.zoonoses` = `field_usable?`, no `scope_digest`, e `reset: true` na troca.

  Toda mudança pedida pelo api volta às Tasks 1–4 e gera SHA-256 novos.

**Relato:** "contracts-mod35b Task 5: branch em <sha>, 265/265; SHA-256: <lista>; aguardando o api."

---

### Task 6: Entrega — SHA-256 conferidos, merge, tag e push (com autorização, depois do api)

**Files:** `field-sync/CHANGELOG.md` (data). Merge, tag e push só com autorização.

**Interfaces:**
- Consumes: o api com as cópias presas aos SHA-256 finais (relato do api ao coordenador).
- Produces: `field-sync-v1.4.0` em `origin`. O campo gera os tipos a partir desta tag.

- [ ] **Step 1: Verificação final.** Rode o bloco da Task 5, Step 1. Expected: os seis SHA-256 **iguais** aos que o spec do api prende, confirmados pelo coordenador. Diferente: pare.

- [ ] **Step 2: Data no CHANGELOG** (`(não publicada até a tag)` → `<AAAA-MM-DD>`) e commit:

```bash
cd $C && $G add field-sync/CHANGELOG.md
$G commit -m "docs: date the field-sync-v1.4.0 changelog entry" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

O CHANGELOG muda o SHA-256 só dele, não o dos seis arquivos presos.

- [ ] **Step 3: Pare e peça autorização**, uma etapa por vez:
  1. merge em `main`;
  2. tag `field-sync-v1.4.0`;
  3. push da `main` e da tag.

  Reporte o hash, `265/265`, os SHA-256 e as decisões aplicadas (D1 a D11, resolvidas pelas Q11 e Q20–Q22).

- [ ] **Step 4: Com autorização**

```bash
$G -C $R/contracts fetch origin && $G -C $R/contracts log --oneline main..origin/main
$G -C $R/contracts checkout main && $G -C $R/contracts merge --ff-only feat/mod-35b-zoonoses
$G -C $R/contracts tag -a field-sync-v1.4.0 -m "field-sync-v1.4.0 — MINOR"
$G -C $R/contracts log --oneline origin/main..main
$G -C $R/contracts push origin main && $G -C $R/contracts push origin field-sync-v1.4.0
$G -C $R/contracts worktree remove $R/.worktrees/contracts-mod35b && $G -C $R/contracts branch -d feat/mod-35b-zoonoses
```

Expected:
- `main..origin/main` vazio. Se não estiver: rebase e refaça o Step 1;
- fast-forward;
- `origin/main..main` só com os commits deste plano;
- push aceito (bypass do aviso de PR, como nos módulos anteriores).

**Relato:** "contracts-mod35b Task 6: field-sync-v1.4.0 publicada (main <sha>, tag pushada)."

---

## Self-review (feito ao escrever o plano)

1. **Cobertura do §11:**
   - `session-response` (Task 1);
   - `reference-response` (Task 3: os nove blocos de `zoonoses_domains`, fechado);
   - `changes-response` (Task 2: `animal` com as 17 chaves do §11 e `leish_survey` com as 7; `removals` + `animal`, e `leish_survey` pela D6);
   - `upload-request` (Task 1: os quatro corpos fechados);
   - recusas do `domains-v1.json` (Task 3: as sete do §11 + duas da D8);
   - falta de estoque `accepted` (README e `rules.stock_shortfall_rejects`);
   - compatibilidade (Global Constraints e README);
   - exemplos (laços de `required` e um inválido por objeto fechado, D11);
   - docs e SHA-256 ao api e ao campo (Tasks 4–5).

   Do §12, só o que chega ao contrato: CBOs, só cão, janela de 15 dias e etiqueta depois do aceite (`zoonoses.rules`). Do §16: microchip e tutor (Review Focus 1).
2. **Passos:** os valores lidos na execução são só hashes, SHA-256 e ids do board. Os rótulos, `limits` e `rules` da D9 estão fixados aqui.
3. **Consistência:**
   - `$defs/animal` e `$defs/date` são iguais entre arquivos, cobrados pelo `check_domains.py`;
   - os nomes dos exemplos são os mesmos no gerador, no Review Focus e nos relatos;
   - contagens: 168 → 218 → 257 → 265; válidos 61 → 72 → 75 → 77;
   - os nove códigos de `resp_zoonoses_rejected()` são os da Task 3.
4. **Review Focus:** as cinco linhas têm teste nas Tasks 1–3. A exceção é o comportamento do app do 35a, que é do campo e vai para o README.
5. **Proporção:** o plano dá as formas de esquema que o contrato só nomeia. Os laços de `required` ficam como regra, não como lista de arquivos.

## Divergências propostas ao contrato

Todas decididas pelo coordenador em 2026-11-23 (§19 do contrato, Q10–Q27). A coluna "Decisão" diz a Q; onde a decisão difere da proposta, a coluna diz o que o plano aplica.

| Código | Onde | Divergência (com evidência) | Proposta | Task | Decisão |
|---|---|---|---|---|---|
| D1 | §1 (`vector_products.uses`), §3, §11 (`products`) | **O problema.** A vacina e a coleira passam a ser `vector_products` com `uses` `rabies_vaccine`/`collar`, mas `vector_control_domains.products[].uses` é fechado em `focal` `perifocal` `spatial` (`field-sync/reference-response-v1.json:175`). O `check_domains.py:123-124` amarra esse enum aos métodos do controle vetorial. O api serve **todos** os produtos ativos ali (`apps/api` `85dc80ee`, `app/services/vector_control/json.rb:226-229`, `VectorProduct.active_products`). Com a semente do §17, a referência de toda cidade com `vector_control` ligado sairia inválida. **A ambiguidade.** O §11 dá `use` no singular na lista do 35b, e o contrato não diz se um produto pode juntar finalidades. | O api filtra `vector_control_domains.products` aos produtos com finalidade de controle vetorial e manda em `uses` só essas. Um produto com `rabies_vaccine` ou `collar` tem `uses` de **um item só** (validação do api e do dashboard). Assim `zoonoses_domains.products[].use` é inequívoco. O esquema do 35a não muda, e um inválido (`invalid-reference-vector-product-use-collar.json`) e o par `zoonoses.product_uses(disjoint)` provam isso. | 3, 5 | **Resolvida pela Q11** (como proposto; o produto de zoonoses tem também `unit` `un` e `charge_size` 1.00, e as peças do 35a abrem pelo gate `VectorControl::Gate.supplies_usable?`). |
| D2 | §11 `leish_collection` ("volta no `upload-response` (`label`), como a amostra do 35a"); §6 (etiqueta) | No 35a a etiqueta **não** volta no resultado: é calculada igual no app e no api (`domains-v1.json`, `vector_control.rules.sample_label`; `upload-response-v1.json:207-224` não tem `label`). O "como" não tem molde. O §6 também não diz a caixa dos 6 hex nem se o `duplicate` traz a etiqueta. | `result.label`, obrigatório no `accepted` **e** no `duplicate` de `leish_collection` (o api relê o teste pelo `id`) e proibido no resto. O padrão é `^LV-[0-9]{4}-([0-9A-F]{6}\|AVUL)-[0-9]{4}$`: hex **maiúsculo**, os 6 primeiros do id do inquérito, como o `upcase` do 35a. A sequência e o estouro de 9999 são do api. | 1 | **Resolvida pela Q21** (como proposto: hex maiúsculo, `label` obrigatório no `accepted` e no `duplicate`). |
| D3 | §2 (409 `animal_changed`), §11 ("com `version` na edição") | O contrato não diz o que o app recebe depois da escrita nem na versão vencida. `result.version` só é obrigatória em `household`/`family`/`individual_registration` e é proibida nos outros (`upload-response-v1.json:221`, `:244-249`). O `stale_version` só aceita esses três (`:213`, `:237`). | O `accepted` de `animal` traz `version` (obrigatória; opcional no `duplicate`, como na Q5 do 24c). A versão vencida volta `stale_version` com `current` = `$defs/animal` (igual ao do download) e `current_version`. É o par do `409 animal_changed` do dashboard. | 1, 2 | **Resolvida pela Q20** (`stale_version` com `current_version`, como no 24c, sem mescla). |
| D4 | §11 `animal` (só lista domicílio, tutor e microchip); §1 (colunas); §1 "o app manda o mês e o dia vira 1" | O contrato não diz quais chaves vão no corpo, quais são obrigatórias, se a edição é inteira ou parcial, se o app muda `status`, nem o nome e o formato do mês. | **Corpo inteiro** na criação e na edição. Obrigatórias: `species`, `sex`, `community` e o domicílio (`household_id` ou `household_client_uuid`, exatamente um; o agente só cadastra animal de domicílio, §2). Opcionais: `name`, `size`, `coat`, `breed`, `birth_estimate_month` (`"YYYY-MM"`; o download traz `birth_estimate_on` com dia 1) e um tutor no máximo. Proibido o tutor com `community: true` (CHECK do §1, no esquema). `status` + `status_on` só na edição (o app marca óbito ou desaparecido; volta a `active` é `animal_not_active`). `microchip`/`registry_number` só na criação. `neutered` nunca. Alternativa ao mês: `birth_estimate_on` com padrão `^\d{4}-\d{2}-01$`. | 1 | **Resolvida pela Q20 e pela Q22** (como proposto, com `birth_estimate_month`; tutor opcional também com domicílio, `community: true` proíbe tutor; sem `neutered`, Q12/Q20). |
| D5 | §11 (`registry_number_taken` nas recusas) × §11 `animal` (sem `registry_number`) | O código de recusa do app não tem campo que o produza. | `registry_number` opcional **só na criação**, como o microchip, e fora do download (a lista do §11 fica). Alternativa: tirar `registry_number_taken` de `errors.item`. | 1, 3 | **Resolvida pela Q20** (`registry_number` só na criação). |
| D6 | §11 ("`removals` ganha `animal`"; "vêm só os inquéritos `open`") | **O que falta.** O contrato não dá o `reason` da saída do animal. O enum é `moved` `erased` `out_of_scope`, e `out_of_scope` é reservado no v1 (`changes-response-v1.json:326-331`; api `app/services/field_sync/catalog.rb:21-23`). Também não diz como o aparelho fica sabendo que um inquérito fechou. **O que vale hoje.** O app do 35a ignora tipo de `removal` desconhecido (campo `dbc54b3`, `src/sync/download.ts:64-68`). | `removal.type` + `animal` e `leish_survey`, sempre com `out_of_scope`: saiu do escopo, deixou `active`, ou o inquérito fechou. Sem `reason` novo. O `animal` do download é sempre `active`; o enum completo fica para o `stale_version` e o par com `domains-v1.json`. Alternativa para o inquérito: `stage` no corpo, como o `status` do 35a. | 2 | **Resolvida pela Q21**, com diferença: `out_of_scope` como proposto, mas **com lápide** (`TOMBSTONE_ENTITY_TYPES` + `animal` e `leish_survey`), e o `leish_survey` traz `household_ids` no lugar de `city_block_ids`. |
| D7 | §11 (`rabies_vaccination`, `leish_collection`, `collar`) | O contrato lista as chaves, mas não diz quais são obrigatórias nem como `dpp_result` e `dpp_on` se combinam. | **Vacinação:** obrigatórias `campaign_id`, `vaccinated_on`, `vector_product_id` e `adverse_event`; `adverse_note` é opcional. **Coleta:** obrigatória `collected_on`; `leish_survey_id` é opcional (avulsa); `dpp_result` ⇔ `dpp_on` (`dependentRequired`). **Coleira:** tudo obrigatório. Nos três, o animal vai por `animal_id` ou `animal_client_uuid`, exatamente um. | 1 | **Resolvida pela Q20**, com acréscimo: `adverse_note` só com `adverse_event: true` (no esquema, `invalid-rabies-vaccination-note-without-event.json`). |
| D8 | §11 (recusas), §3 (janela de `vaccinated_on`), §12 (inquérito "que cubra o quarteirão do domicílio") | **Faltam os `path`** dos sete códigos. **Faltam dois códigos:** a data fora da janela e o inquérito que não cobre o domicílio não têm código. **Falta dizer** o código do produto errado. | **`path`:** os da tabela da Task 1 (`microchip`, `registry_number`, `animal_id`/`animal_client_uuid`, `campaign_id` e `leish_survey_id`). **Códigos novos:** `date_out_of_range` (o `path` é o campo de data: `vaccinated_on`, `collected_on`, `dpp_on` ou `placed_on`) e `household_not_in_survey` (`path` `leish_survey_id`). **Reaproveitados:** `product_not_allowed` (`vector_product_id`, do 35a), `cbo_not_allowed`, `not_found`, `out_of_scope` e `feature_disabled`. | 1, 3 | **Resolvida pela Q20** (os dois códigos e os `path` como proposto). |
| D9 | §11 (`zoonoses_domains`: listas e `limits` sem conteúdo) | **O que falta.** O contrato não dá os rótulos, as chaves de `limits`, nem um domínio para `leish_survey.kind` e para a finalidade do produto. **O que a ferramenta exige.** O `check_domains.py:67-145` exige que todo enum novo tenha o seu domínio em `domains-v1.json`. **Uma consequência.** O `open_campaign` na referência faz o `reference_version` mudar a cada abertura ou fechamento de campanha. | Seção `zoonoses` em `domains-v1.json`, com os rótulos e as listas da Task 3. `leish_survey_kinds` e `product_uses` ficam só em `domains-v1.json`: o app lê o arquivo pela tag. Os `limits` são `name_max`, `coat_max`, `breed_max`, `microchip_digits`, `registry_number_max`, `adverse_note_min` e `adverse_note_max`, e os `rules` em dado (CBOs, só cão, janela de 15 dias, formato da etiqueta, microchip só na criação, falta de estoque não recusa). O catálogo do api usa os mesmos rótulos, conferidos pelo `domains_spec` (Task 5). A troca do `reference_version` pela campanha fica no README. | 3, 4 | **Resolvida pela Q20** (seção `zoonoses` como proposta). |
| D10 | §11 (`last_rabies_vaccination_on`, `collar_due_on` no `animal`); §1 (`version` +1 a cada escrita da tabela) | Os dois campos vêm de outras tabelas: a vacinação e a coleira não sobem a `version` do animal. O `stale_version` e a regra "versão maior sobrescreve" deixariam o aparelho com a data velha. É o caso da Q29 do 35a (`field-sync/README.md:257-261`). | O api reemite o `animal` com a **mesma** `version` quando muda a vacinação, a coleira ou o nome do tutor. O app sobrescreve `animal` e `leish_survey` a cada recebimento, qualquer que seja a `version`, e nunca apaga a fila. O conflito de edição continua pelo `stale_version` (D3). | 2, 4 | **Resolvida pela Q20** (como proposto). |
| D11 | §11 "Exemplos: cada item e cada tipo novo leva um válido e os inválidos de cada `required` e de cada `additionalProperties`" | Não se diz se a regra vale para os objetos aninhados, para a sessão e a referência, nem se os pares um-ou-outro contam como `required`. | Vale para os corpos dos quatro itens e dos dois tipos e para o objeto aninhado `tutor`. Os inválidos são um por chave de `required` (em laço no gerador) e um por objeto fechado, com uma chave que prova uma guarda. Os pares um-ou-outro levam o caso "sem" e o caso "os dois" (`oneOf`). A sessão e a referência levam inválidos dirigidos, como no 35a. São 265 casos, 77 válidos. | 1–3 | **Resolvida pela Q20** (como proposto; 265 casos, 77 válidos, com o inválido do `adverse_note`). |

Observação, sem divergência: `features.zoonoses` fora do `required` segue o `features.vector_control` (`session-response-v1.json:63-68`). O `zoonoses_domains` opcional no topo segue o `vector_control_domains` (`reference-response-v1.json:8`, `:96`). Os dois estão de acordo com o contrato.
