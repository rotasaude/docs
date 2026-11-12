# Módulo 24b — Contrato de sincronização do app de campo v1 (repo contracts) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Ordem de entrega (contrato §15):** contracts (esquemas, **sem a tag**) → api (Task 1 = prova técnica, que publica o acréscimo do §10–§11) → contracts (domínios conferidos, Task 4) → tag `field-sync-v1.0.0` (Task 5, só com autorização). As Tasks 0–3 não dependem do api.

**Goal:** Publicar no repo `contracts` o domínio novo `field-sync/`: os esquemas JSON 2020-12 da sincronização v1 do app de campo (sessão, envio em lote, resultado por item, páginas do download), os domínios e as regras cruzadas da visita e da atividade coletiva **em dado** (`domains-v1.json`), exemplos válidos e inválidos verificados pelo `json_schemer` do `api`, o README do domínio e a linha no README da raiz, e a tag `field-sync-v1.0.0` depois da prova técnica do api (F-24.17; ADRs 0041 e 0052).

**Architecture:** Quatro esquemas fechados (`additionalProperties: false`), um por mensagem do §9.3. Os `$defs` comuns (uuid, datas, endereço, `<household>`, `<family>`, `<registration>`) são repetidos idênticos nos arquivos que os usam e conferidos por comando, como no `clinical/`. Uniões por tipo (item do envio, resultado, item do download, motivo de "quem visitar") são `allOf` de `if`/`then` pelo discriminador, nunca `oneOf`, para o erro apontar o campo. Os exemplos são gerados por `field-sync/tools/make_examples.py` (determinístico, como o `billing/tools/`), listados num manifesto `{ suites: [ { schema, cases: [ { file, expect } ] } ] }` (formato de suítes do `lab/` e do `exams/`, com o `expect` exato `"<data_pointer> <type>"` do `clinical/`) e verificados por `field-sync/tools/check.sh` (empacota em Python e valida com o `json_schemer` 2.5 do container do `api`, como o comando do `session/`). `field-sync/tools/check_domains.py` confere que os `$defs` repetidos são iguais e que cada `enum` do envio é exatamente o domínio de `domains-v1.json`. A tag só sai com `domains-v1.json` conferido contra a prova técnica do api (`proof` preenchido).

**Tech Stack:** JSON Schema draft 2020-12; `json_schemer` 2.5 (bundle do `api`, só para a verificação); `python3` do host (gerar exemplos, empacotar, conferir domínios); `shasum`; git em `/opt/homebrew/bin/git`.

**Spec:** `docs/superpowers/specs/2026-11-12-module-24b-home-visit-collective-activity-design.md` (decisões 4A, 6A, 7A; coordenador 18, 19; §6 "contracts"); ADR `docs/adr/0052.md` (e `0041.md`, núcleo e ferramenta); contrato entre apps (fonte única dos formatos): `docs/superpowers/plans/2026-11-12-module-24b-home-visit-contracts.md` §1, §3–§5, §8, §9 (em especial §9.3 e §9.4), §10, §11, §13, §15; formas do 24a: `docs/superpowers/plans/2026-10-23-module-24a-territory-registry-contracts.md` §2 e §14 (L6, L7, L21, L30, L31), conferidas no código em `ro-api/app/services/territory_registry/json.rb` (api `d36aef3`). Molde: `docs/superpowers/plans/2026-10-08-module-19b-signature-contracts-repo.md`. Diferenças em "Divergências propostas ao contrato", K-1 a K-16.

**Reconciliação (§16 do contrato, 2026-11-12):** K-1, K-2, K-3, K-8, K-9, K-10, K-13, K-14, K-15 aceitas (Q26); K-4 e K-5 → Q27 (família fora do v1, documentada; `household_client_uuid` e `patient_client_uuid` aprovados — a Task 4 Step 5 aplica); **K-6 rejeitada** (Q15): o pedido usa local plano, `place_health_unit_id` ou `place_text`, exatamente um (a resposta da equipe segue aninhada) — já aplicado na Task 1 abaixo, marcado "(Q15)"; K-7 aceita (Q15), e `duplicates_reviewed` só por participante; K-11, K-12 → Q13; K-16 → Q28, corrigido no Step 6 da Task 2. Domínios provisórios ajustados à Q3 (`tipo_glicemia` na ordem do dicionário; público-alvo com 19 e 20). `domains-v1.json` mantém a estrutura desta Task 2 (é do app); o api serve ao dashboard a forma do §7 do contrato a partir do mesmo YAML e confere os códigos contra este arquivo (Q13).

## Global Constraints

- Repo `contracts` na raiz do monorepo: `/Users/eduardovrocha/Development/ioit.solutions/rota-saude/contracts` (remote `git@github.com:rotasaude/contracts.git`), `origin/main` em `b0d1f87` ou mais novo. Não existe `apps/contracts`.
- Domínio novo `field-sync/` na raiz do repo, como `clinical/`, `lab/` e `billing/`; tag `field-sync-v1.0.0`, anotada, mensagem `field-sync-v1.0.0 — MAJOR` (primeira versão do domínio). CHANGELOG obrigatório (`field-sync/CHANGELOG.md`): sem entrada, não há tag.
- Esquemas: `$id` = `https://rota-saude/schemas/field-sync/<arquivo>` e `title` = `rotasaude.field_sync.<nome>.v1` (convenção do repo; o contrato §9.4 põe o nome no `$id`, K-1). `contract` da sessão = `"field-sync-v1"` (`const`).
- Fechado em todo nível, **salvo** as seções de dicionário do 24a (`living_conditions`, `identification`, `sociodemographic`, `health_conditions`, `street_situation`), que são `type: object` (K-2): as chaves e os domínios delas moram em `config/ledi/registration_dictionaries.yml` do api e no contrato do 24a §15.
- Formas que o esquema fixa: uuid em hexadecimal minúsculo; instantes ISO 8601 **com fuso explícito** (`Z` ou `±hh:mm`; o api devolve `iso8601` no fuso da cidade, K-3); datas `AAAA-MM-DD`; CPF 11 dígitos e CNS 15 dígitos, sem máscara; `property_type` e `no_cpf_reason` como texto (códigos do 24a, K-14); códigos LEDI como inteiros.
- O esquema confere forma: tipos, domínios, faixas, objetos fechados, o par de medidas (`dependentRequired`) e as formas por `status`/`type`. **Não** confere relação entre campos (desfecho × motivos, imóvel × pessoa, tipo de atividade × temas, quem pode o quê, CPF × justificativa): isso está em dado em `domains-v1.json` (`rules`) e é decidido pelo api, que devolve o item `rejected` com o código (K-12).
- Nenhum dado real nos exemplos: nomes fictícios; CPFs com dígito verificador válido mas inventados (os do 19b: `39053344705`, `11144477735`); CNS provisório `702000000000001` (dígito válido pelo algoritmo dos CNS iniciados em 7, inventado).
- Worktree **fora de `contracts/`**: `rota-saude/.worktrees/contracts-mod24b`, branch `feat/mod-24b-home-visits` a partir de `origin/main` (contrato, valores fixos). Comandos a partir da raiz do monorepo, com o compose de pé (`docker compose ps` mostra o serviço `api` rodando).
- Testes: este repo não tem `bin/test`; a verificação é `field-sync/tools/check.sh` e `field-sync/tools/check_domains.py`. Nada aqui roda `rspec` nem `vitest`.
- Git: `/opt/homebrew/bin/git`. Nunca `git add -A` nem `git add .`: caminhos explícitos. Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Merge na `main`, tag e push só com autorização explícita do usuário, uma etapa de cada vez; antes do push, `origin/main..main` só pode ter os commits desta entrega (sessões paralelas).
- Board: F-24.17 (contracts, api, dashboard). O card já está em In Progress pelo plano do api; a Task 0 só confere. Done quando as três partes estiverem entregues.
- Execução coordenada: ao fim de cada task, relato de uma linha à sessão "Próximos módulos" (`SendMessage`).

## Ambiente de execução

- Variáveis: `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `G=/opt/homebrew/bin/git`, `C=$R/.worktrees/contracts-mod24b`.
- Caminhos das tasks relativos a `$C` (ex.: `field-sync/upload-request-v1.json` = `$C/field-sync/upload-request-v1.json`), salvo quando começam por `$R`.
- Verificação (sempre de `$R`): `$C/field-sync/tools/check.sh` (todas as suítes) ou `$C/field-sync/tools/check.sh <esquema>` (uma suíte); `python3 $C/field-sync/tools/check_domains.py` (com `--release` antes da tag).

## Review Focus

1. **Turno ou cliente_uuid no corpo da visita** (`shift`, `client_uuid` dentro de `body`): o turno é derivado de `occurred_at` pelo api e o uuid vai no item; aceitar os dois daria duas fontes. Testes: `invalid-home-visit-shift-in-body.json` (`/items/0/body/shift schema`) e o `client_uuid` fora do `body` coberto pelo fechamento do item (Task 1).
2. **Hora sem fuso** (`2026-11-12T09:40:00`): o dia da cidade e o turno sairiam errados no app offline. Teste: `invalid-home-visit-occurred-at-without-offset.json` (`/items/0/body/occurred_at pattern`) (Task 1).
3. **Dado de pessoa onde o contrato proíbe**: CPF em claro no candidato a duplicado, no responsável da família ou no paciente do `<registration>` baixado; CID num motivo de "quem visitar"; INEP no local da atividade; latitude na visita. Testes: `invalid-response-candidate-with-cpf.json`, `invalid-response-stale-household-with-cpf.json`, `invalid-changes-registration-with-cpf.json`, `invalid-changes-care-line-reason-with-cid.json`, `invalid-collective-activity-inep.json`, `invalid-home-visit-latitude.json` (Task 1).
4. **Resultado incoerente com o `status`** (`accepted` sem `id`, `rejected` com `id`, `stale_version` sem `current`, `candidates` fora de `duplicate_suspected`) e **"quem visitar" fora da última página** (ou faltando nela): o app decide o que apagar e o que reenviar por esses campos. Testes: `invalid-response-accepted-without-id.json`, `invalid-response-rejected-with-id.json`, `invalid-response-stale-without-current.json`, `invalid-response-candidates-without-duplicate-suspected.json`, `invalid-changes-who-null-on-last-page.json`, `invalid-changes-who-on-middle-page.json` (Task 1).
5. **Domínio do esquema diferente do `domains-v1.json`** (ou tag com domínio não conferido na prova técnica): o app montaria o formulário com um código que o esquema recusa. Testes: `check_domains.py` imprime `dominios-iguais` (Task 2) e `--release` falha com `proof: null` (Task 2, Step 3; Task 5, Step 1).

## Mapa de arquivos

| Arquivo (repo `contracts`, worktree `$C`) | Mudança | Task |
|---|---|---|
| `field-sync/tools/make_examples.py` | novo: gera os 56 exemplos e o manifesto | 1 |
| `field-sync/tools/check.sh` | novo: valida o manifesto com o `json_schemer` do api | 1 |
| `field-sync/examples/*.json` + `manifest.json` | 56 exemplos (20 válidos, 36 inválidos) | 1 |
| `field-sync/session-response-v1.json` | novo: `GET /field/v1/session` | 1 |
| `field-sync/upload-request-v1.json` | novo: `POST /field/v1/uploads` (pedido) | 1 |
| `field-sync/upload-response-v1.json` | novo: `POST /field/v1/uploads` (resposta 200) | 1 |
| `field-sync/changes-response-v1.json` | novo: `GET /field/v1/changes` | 1 |
| `field-sync/domains-v1.json` | novo: domínios do §10–§11 e regras do §3–§4 em dado (`proof: null`) | 2 |
| `field-sync/tools/check_domains.py` | novo: `$defs` iguais, `enum` = domínio, erros conhecidos, `--release` | 2 |
| `field-sync/README.md`, `field-sync/CHANGELOG.md` | novos | 2 |
| `README.md` (raiz) | domínio novo na tabela, parágrafo `### field-sync`, prefixo da tag | 2 |
| — | entrega das cópias ao api (SHA-256) | 3 |
| `field-sync/domains-v1.json`, `field-sync/upload-request-v1.json`, `field-sync/CHANGELOG.md` (e exemplos, se um código mudar) | domínios conferidos na prova técnica (`proof` preenchido) e decisões Q-n | 4 |
| — | verificação final, merge, tag, push (com autorização) | 5 |

**Estratégia de teste:** cada esquema nasce de exemplos escritos antes (Task 1, Step 1), um defeito por arquivo inválido, com o erro esperado exato (`<data_pointer> <type>` do `json_schemer`); os esquemas provisórios (`{ "type": "object" }`) fazem os 36 inválidos falharem (Step 2) e os definitivos fazem os 56 passarem (Step 8). A consistência entre arquivos (`$defs` repetidos, `enum` × `domains-v1.json`, códigos de erro dos exemplos × `errors.item`) é um comando próprio (Task 2). O api testa as respostas reais contra a **cópia** destes esquemas em `config/field_sync/`, presa por SHA-256 (plano do api); este plano entrega os SHA-256 (Task 3 e Task 4).

---

### Task 0: Base, card e worktree

Esta task não escreve nada.

**Files:** nenhum.

**Interfaces:**
- Consumes: `contracts` `origin/main` (`b0d1f87` ou mais novo); board #1 (`PVT_kwDOEbfGRc4BbxiB`, campo Status `PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`).
- Produces: `$C` em `feat/mod-24b-home-visits`; a base anotada.

- [ ] **Step 1: Conferir o card F-24.17** (um `gh` simples por chamada, sem `$(...)` nem laço)

```bash
gh project item-list 1 --owner rotasaude --limit 500 --format json --jq '.items[] | select(.title | test("^F-24\\.17 ")) | {id, title, status}'
```

Expected: F-24.17 em In Progress ou além (movido pelo plano do api). Se estiver em Todo, mova:

```bash
gh project field-list 1 --owner rotasaude --format json --jq '.fields[] | select(.name == "Status") | {id, options}'
gh project item-edit --id <PVTI_do_card> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id_In_Progress>
```

Card ausente: **pare e reporte**.

- [ ] **Step 2: Base do contracts e worktree**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/contracts fetch -q origin
$G -C $R/contracts log --oneline -1 origin/main
$G -C $R/contracts ls-tree --name-only origin/main | grep -x field-sync || echo "sem field-sync"
$G -C $R/contracts tag -l 'field-sync-*'
$G -C $R/contracts show origin/main:README.md | grep -n 'Nove domínios\|billing/\](billing\|### Fora deste repo\|`lab-vX.Y.Z`'
$G -C $R/contracts worktree add $R/.worktrees/contracts-mod24b -b feat/mod-24b-home-visits origin/main
docker compose -f $R/docker-compose.yml ps api
```

Expected:
- `origin/main` em `b0d1f87 feat: add urgency classification variant to the protocols schema (protocols-v1.7.0)` ou mais novo;
- `sem field-sync` e nenhuma tag `field-sync-*`;
- no README: "Nove domínios", a linha de `billing/` na tabela, o título `### Fora deste repo, por enquanto` e o prefixo `` `lab-vX.Y.Z` `` (âncoras da Task 2, Step 6);
- worktree criado; serviço `api` rodando.

Se `field-sync/` ou a tag já existirem, ou se o README tiver outro número de domínios (outro módulo entrou antes), **pare e reporte** (a Task 2 ajusta as âncoras).

**Relato ao coordenador:** "contracts-mod24b Task 0: worktree em feat/mod-24b-home-visits sobre <sha>; F-24.17 em In Progress; sem field-sync nem tag."

---

### Task 1: Esquemas da sincronização v1 com exemplos

**Files (worktree `$C`):**
- Create: `field-sync/tools/make_examples.py`, `field-sync/tools/check.sh`
- Create: `field-sync/examples/` (56 exemplos + `manifest.json`, gerados)
- Create: `field-sync/session-response-v1.json`, `field-sync/upload-request-v1.json`, `field-sync/upload-response-v1.json`, `field-sync/changes-response-v1.json`

**Interfaces:**
- Consumes: as formas do contrato do 24b §9.3 e do 24a §2 (código: `TerritoryRegistry::Json.micro_area`, `.household`, `.family`, `.registration`, `.duplicate`; `TerritoryRegistry::Address::KEYS`; `TerritoryRegistry::Catalog::PROPERTY_TYPES`, `NO_CPF_REASONS`, `STATUSES`).
- Produces (`field-sync-v1.0.0`):
  - `session-response-v1.json` — `{ contract, server_time, user {id,name}, professional {id,cbo,has_cns}, teams [{id,ine,name,micro_areas [{id,code,name,scope}],out_of_area_allowed}], features {home_visits,collective_activities,territory_registry}, min_date }`.
  - `upload-request-v1.json` — `{ items: [ { client_uuid, type, body } ] }` (1–50); `body` por `type`: `$defs/home_visit_body`, `home_visit_withdrawal_body`, `collective_activity_body`, `patient_body`, `household_body`, `individual_registration_body`.
  - `upload-response-v1.json` — `{ results: [ $defs/result ] }`; `result = { client_uuid, status, type, id?, error?, path?, candidates?, current_version?, current? }` com as formas por `status`.
  - `changes-response-v1.json` — `{ reset, items: [ $defs/change_item ], removals: [ $defs/removal ], who_to_visit, next_cursor, has_more }`.
  - `$defs` comuns (idênticos onde repetidos): `uuid`, `uuid_or_null`, `datetime`, `datetime_or_null`, `date`, `date_or_null`, `micro_area_code`, `micro_area_ref`, `team_ref`, `registered_by`, `property_type`, `address`, `household`, `family`, `registration`.
  - Consumido pelo plano do api (cópias em `config/field_sync/`, presas por SHA-256, e as respostas testadas contra elas) e pelo 24c (app de campo).

- [ ] **Step 1: Escreva o gerador dos exemplos e gere**

Crie `field-sync/tools/make_examples.py`:

```python
#!/usr/bin/env python3
"""Gera field-sync/examples/*.json e examples/manifest.json (determinístico).

Uso: python3 field-sync/tools/make_examples.py (de qualquer pasta).
Um defeito por arquivo inválido; o expect é o erro exato do json_schemer
("<data_pointer> <type>", "(root)" na raiz). Dados fictícios; os CPFs têm
dígito verificador válido e são inventados; o CNS 702000000000001 é
provisório (início 7) e inventado.
"""
import copy
import json
import os

HERE = os.path.dirname(os.path.abspath(__file__))
OUT = os.path.join(os.path.dirname(HERE), "examples")
DELETE = object()


def cu(n):
    return f"c0ffee{n:02d}-0000-4000-8000-{n:012d}"


TEAM = "3f1c2a4b-5d6e-4f70-8a9b-0c1d2e3f4a5b"
MA1 = "6a7b8c9d-0e1f-4a2b-8c3d-4e5f6a7b8c9d"
MA2 = "7b8c9d0e-1f2a-4b3c-9d4e-5f6a7b8c9d0e"
HH = "8c9d0e1f-2a3b-4c4d-8e5f-6a7b8c9d0e1f"
HH2 = "9e0f1a2b-3c4d-4e5f-8a6b-7c8d9e0f1a2b"
FAM = "9d0e1f2a-3b4c-4d5e-9f6a-7b8c9d0e1f2a"
PAT = "0e1f2a3b-4c5d-4e6f-8a7b-8c9d0e1f2a3b"
PAT2 = "1f2a3b4c-5d6e-4f7a-9b8c-9d0e1f2a3b4c"
PAT3 = "2b3c4d5e-6f7a-4b8c-9d0e-1f2a3b4c5d6e"
REG = "2a3b4c5d-6e7f-4a8b-8c9d-0e1f2a3b4c5d"
REG2 = "3c4d5e6f-7a8b-4c9d-8e0f-1a2b3c4d5e6f"
REG3 = "4d5e6f7a-8b9c-4d0e-9f1a-2b3c4d5e6f7a"
USER = "3b4c5d6e-7f8a-4b9c-9d0e-1f2a3b4c5d6e"
PROF = "4c5d6e7f-8a9b-4c0d-8e1f-2a3b4c5d6e7f"
PROF2 = "5d6e7f8a-9b0c-4d1e-9f2a-3b4c5d6e7f8a"
UNIT = "6e7f8a9b-0c1d-4e2f-8a3b-4c5d6e7f8a9b"
VISIT = "7f8a9b0c-1d2e-4f3a-9b4c-5d6e7f8a9b0c"
ACT = "8a9b0c1d-2e3f-4a4b-8c5d-6e7f8a9b0c1d"
CURSOR = "eyJ2IjoxLCJzaW5jZSI6IjIwMjYtMTEtMTJUMTI6MDA6MDBaIn0"

TEAM_REF = {"id": TEAM, "ine": "0001234567", "name": "ESF Vila Nova"}
MA1_REF = {"id": MA1, "code": "03"}
ADDRESS = {"zip": "80010000", "street_type": "RUA", "street": "das Flores", "number": "120", "no_number": False,
           "complement": None, "neighborhood_id": None, "district": "Centro", "reference": "Perto da praça"}
NEW_PATIENT = {"full_name": "Benedita Aparecida Rocha", "birth_date": "1948-05-02", "sex": "female",
               "mother_name": "Maria José Rocha", "no_cpf_reason": "2", "cns": "702000000000001"}
WITHDRAWAL_REASON = "Visita registrada no domicílio errado."


def set_path(doc, path, value):
    cur = doc
    for key in path[:-1]:
        cur = cur[key]
    if value is DELETE:
        del cur[path[-1]]
    else:
        cur[path[-1]] = value
    return doc


def variant(base, path, value):
    return set_path(base(), path, value)


# ---------- GET /field/v1/session ----------

def session_acs():
    return {"contract": "field-sync-v1", "server_time": "2026-11-12T12:00:00Z",
            "user": {"id": USER, "name": "Rosângela Alves"},
            "professional": {"id": PROF, "cbo": "515105", "has_cns": True},
            "teams": [{"id": TEAM, "ine": "0001234567", "name": "ESF Vila Nova",
                       "micro_areas": [{"id": MA1, "code": "03", "name": "Vila Nova — rua das Flores", "scope": "responsible"}],
                       "out_of_area_allowed": True}],
            "features": {"home_visits": True, "collective_activities": True, "territory_registry": True},
            "min_date": "2026-10-01"}


def session_agent():
    d = session_acs()
    d["user"]["name"] = "Carlos Menezes"
    d["professional"] = {"id": PROF2, "cbo": "515140", "has_cns": False}
    d["teams"][0]["micro_areas"] = [{"id": MA2, "code": "07", "name": "Jardim Esperança", "scope": "assigned"}]
    d["teams"][0]["out_of_area_allowed"] = False
    d["features"]["collective_activities"] = False
    return d


# ---------- POST /field/v1/uploads (pedido) ----------

def hv_resident():
    return {"team_id": TEAM, "occurred_at": "2026-11-12T09:40:00-03:00", "household_id": HH, "subject": "resident",
            "patient_id": PAT, "outcome": 1, "reasons": [11, 12], "shared": False,
            "measurements": {"systolic": 138, "diastolic": 88, "weight_kg": 81.5, "height_cm": 162.0,
                             "glucose_mg_dl": 142, "glucose_type": 2},
            "note": "Pediu para a equipe voltar à tarde."}


def hv_non_residential():
    return {"team_id": TEAM, "occurred_at": "2026-11-12T14:05:00-03:00", "micro_area_id": MA1, "property_type": "4",
            "subject": "none", "outcome": 1, "reasons": [34, 37], "shared": False}


def collective():
    return {"team_id": TEAM, "held_on": "2026-11-11", "shift": 2, "activity_type": 5,
            "place_text": "Salão da Associação de Moradores da Vila Nova",  # Q15
            "participants_count": 12, "altered_count": 1, "target_audience": [12], "health_topics": [4],
            "health_practices": [20], "professionals": [{"professional_id": PROF2}],
            "participants": [{"patient_id": PAT, "altered_evaluation": True, "measurements": {"weight_kg": 81.5, "height_cm": 162.0}},
                             {"patient_client_uuid": cu(10), "altered_evaluation": False}]}


def household_update():
    return {"household_id": HH, "version": 3, "micro_area_id": MA1, "property_type": "1", "refused": False,
            "contact_phone": "41999990000", "address": copy.deepcopy(ADDRESS), "living_conditions": {}}


def registration_create():
    return {"patient_id": PAT2, "household_id": HH, "family_id": FAM, "is_responsible": False, "identification": {},
            "patient_demographics": {"race_color": "01", "ethnicity": None}, "sociodemographic": {},
            "health_conditions": {}, "refused": False}


def item(n, kind, body):
    return {"client_uuid": cu(n), "type": kind, "body": body}


def up(*items):
    return {"items": list(items)}


def upload_hv():
    return up(item(1, "home_visit", hv_resident()))


def upload_hv_nr():
    return up(item(2, "home_visit", hv_non_residential()))


def upload_withdrawal():
    return up(item(3, "home_visit_withdrawal", {"visit_client_uuid": cu(1), "reason": WITHDRAWAL_REASON}))


def upload_ca():
    return up(item(4, "collective_activity", collective()))


def upload_patient():
    return up(item(5, "patient", {"new_patient": copy.deepcopy(NEW_PATIENT), "duplicates_reviewed": False}))


def upload_household():
    return up(item(6, "household", household_update()))


def upload_registration():
    return up(item(7, "individual_registration", registration_create()))


def upload_mixed():
    visit = hv_resident()
    del visit["patient_id"], visit["measurements"], visit["note"]
    visit["patient_client_uuid"] = cu(10)
    return up(item(10, "patient", {"new_patient": copy.deepcopy(NEW_PATIENT)}), item(11, "home_visit", visit),
              item(12, "collective_activity", collective()))


B = ["items", 0, "body"]

# ---------- formas do download (24a §2, json.rb) ----------

def household():
    return {"id": HH, "micro_area": dict(MA1_REF), "address": copy.deepcopy(ADDRESS), "property_type": "1",
            "contact_phone": "41999990000", "refused": False, "status": "active", "version": 3,
            "registered_at": "2026-10-28T10:15:00-03:00", "registered_by": {"name": "Rosângela Alves"},
            "families": [{"id": FAM, "responsible": {"patient_id": PAT, "display_name": "Joana Conceição"},
                          "income_band": "3", "members_count": 3, "living_since": "2019-03-01", "moved_out": False,
                          "responsible_exited": False}],
            "living_conditions": {}}


def registration():
    return {"id": REG,
            "patient": {"id": PAT, "display_name": "Joana Conceição", "full_name": "Joana D'Arc Conceição",
                        "mother_name": "Maria Conceição", "birth_date": "1961-03-14", "sex": "female", "has_cpf": True,
                        "no_cpf_reason": None, "cns_masked": None, "identity_locked": False},
            "patient_demographics": {"race_color": "03", "ethnicity": None},
            "household_id": HH, "family_id": FAM, "is_responsible": True, "team": dict(TEAM_REF), "micro_area": dict(MA1_REF),
            "identification": {}, "sociodemographic": {}, "health_conditions": {}, "street_situation": None,
            "refused": False, "exit": None, "status": "valid", "version": 2,
            "registered_at": "2026-10-28T10:20:00-03:00", "registered_by": {"name": "Rosângela Alves"},
            "identity_changed_at": None, "identity_review_pending": False}


def micro_area(ident=MA1, code="03", name="Vila Nova — rua das Flores"):
    return {"id": ident, "team": dict(TEAM_REF), "code": code, "name": name,
            "acs": {"professional_id": PROF, "name": "Rosângela Alves"}, "active": True}


def who_to_visit():
    return {"generated_at": "2026-11-12T12:00:00Z", "micro_areas": [{"micro_area_id": MA1, "people": [
        {"patient_id": PAT, "display_name": "Joana Conceição", "age": 65,
         "household": {"id": HH, "address_short": "Rua das Flores, 120"}, "family_id": FAM, "micro_area": dict(MA1_REF),
         "last_visit_on": "2026-08-20",
         "reasons": [{"code": "care_line", "care_line": "Pessoa com hipertensão",
                      "practice": "2 visitas do ACS nos últimos 12 meses", "status": "overdue", "due_on": "2026-11-01"}]},
        {"patient_id": PAT2, "display_name": "Benedita Rocha", "age": 78,
         "household": {"id": HH, "address_short": "Rua das Flores, 120"}, "family_id": FAM, "micro_area": dict(MA1_REF),
         "last_visit_on": None,
         "reasons": [{"code": "registration_due", "status": "due_soon", "due_on": "2026-12-20"}, {"code": "never_visited"}]}]}]}


# ---------- POST /field/v1/uploads (resposta) ----------

def results(*rs):
    return {"results": list(rs)}


def done(n, kind, ident, status="accepted"):
    return {"client_uuid": cu(n), "status": status, "type": kind, "id": ident}


def resp_accepted():
    return results(done(10, "patient", PAT2), done(11, "home_visit", VISIT), done(12, "collective_activity", ACT))


def resp_duplicate():
    return results(done(11, "home_visit", VISIT, "duplicate"))


def resp_rejected():
    return results(
        {"client_uuid": cu(1), "status": "rejected", "type": "home_visit", "error": "visit_date_out_of_range", "path": "occurred_at"},
        {"client_uuid": cu(4), "status": "rejected", "type": "collective_activity", "error": "feature_disabled"},
        {"client_uuid": cu(5), "status": "rejected", "type": "patient", "error": "client_uuid_conflict"})


def resp_duplicate_suspected():
    return results({"client_uuid": cu(10), "status": "rejected", "type": "patient", "error": "duplicate_suspected",
                    "candidates": [{"patient_id": PAT2, "display_name": "Benedita Rocha", "birth_date": "1948-05-02", "has_cpf": False},
                                   {"patient_id": PAT3, "display_name": "Benedita A. Rocha", "birth_date": "1948-05-02",
                                    "has_cpf": True, "registration_id": REG3}]})


def resp_stale_household():
    current = household()
    current["version"] = 4
    return results({"client_uuid": cu(6), "status": "stale_version", "type": "household", "id": HH,
                    "current_version": 4, "current": current})


def resp_stale_registration():
    current = registration()
    current["version"] = 3
    return results({"client_uuid": cu(7), "status": "stale_version", "type": "individual_registration", "id": REG,
                    "current_version": 3, "current": current})


# ---------- GET /field/v1/changes ----------

def page_first():
    return {"reset": False,
            "items": [{"type": "micro_area", "id": MA1, "body": micro_area()},
                      {"type": "household", "id": HH, "version": 3, "body": household()},
                      {"type": "registration", "id": REG, "version": 2, "body": registration()}],
            "removals": [], "who_to_visit": None, "next_cursor": CURSOR, "has_more": True}


def page_last():
    return {"reset": False, "items": [], "removals": [], "who_to_visit": who_to_visit(), "next_cursor": CURSOR, "has_more": False}


def page_removals():
    return {"reset": False, "items": [],
            "removals": [{"type": "household", "id": HH2, "reason": "moved"},
                         {"type": "registration", "id": REG2, "reason": "erased"},
                         {"type": "registration", "id": REG3, "reason": "out_of_scope"}],
            "who_to_visit": {"generated_at": "2026-11-12T12:00:00Z", "micro_areas": [{"micro_area_id": MA1, "people": []}]},
            "next_cursor": CURSOR, "has_more": False}


def page_reset():
    return {"reset": True, "items": [{"type": "micro_area", "id": MA2, "body": micro_area(MA2, "07", "Jardim Esperança")}],
            "removals": [], "who_to_visit": None, "next_cursor": CURSOR, "has_more": True}


SUITES = [
    ("session-response-v1.json", {
        "session-acs.json": (session_acs(), "valid"),
        "session-assigned-agent.json": (session_agent(), "valid"),
        "invalid-session-contract-v2.json": (variant(session_acs, ["contract"], "field-sync-v2"), "/contract const"),
        "invalid-session-scope-team.json": (variant(session_acs, ["teams", 0, "micro_areas", 0, "scope"], "team"), "/teams/0/micro_areas/0/scope enum"),
        "invalid-session-token-key.json": (variant(session_acs, ["token"], "rsf_x"), "/token schema"),
    }),
    ("upload-request-v1.json", {
        "upload-home-visit-resident.json": (upload_hv(), "valid"),
        "upload-home-visit-non-residential.json": (upload_hv_nr(), "valid"),
        "upload-home-visit-withdrawal.json": (upload_withdrawal(), "valid"),
        "upload-collective-activity.json": (upload_ca(), "valid"),
        "upload-patient.json": (upload_patient(), "valid"),
        "upload-household-update.json": (upload_household(), "valid"),
        "upload-individual-registration.json": (upload_registration(), "valid"),
        "upload-mixed-batch.json": (upload_mixed(), "valid"),
        "invalid-upload-empty.json": (up(), "/items minItems"),
        "invalid-upload-51-items.json": (up(*[item(n, "home_visit_withdrawal", {"visit_client_uuid": cu(1), "reason": WITHDRAWAL_REASON}) for n in range(1, 52)]), "/items maxItems"),
        "invalid-upload-unknown-type.json": (up({"client_uuid": cu(1), "type": "family", "body": {}}), "/items/0/type enum"),
        "invalid-upload-client-uuid-uppercase.json": (variant(upload_hv, ["items", 0, "client_uuid"], cu(1).upper()), "/items/0/client_uuid pattern"),
        "invalid-home-visit-shift-in-body.json": (variant(upload_hv, B + ["shift"], 1), "/items/0/body/shift schema"),
        "invalid-home-visit-occurred-at-without-offset.json": (variant(upload_hv, B + ["occurred_at"], "2026-11-12T09:40:00"), "/items/0/body/occurred_at pattern"),
        "invalid-home-visit-outcome-4.json": (variant(upload_hv, B + ["outcome"], 4), "/items/0/body/outcome enum"),
        "invalid-home-visit-reason-26.json": (variant(upload_hv, B + ["reasons"], [26]), "/items/0/body/reasons/0 enum"),
        "invalid-home-visit-systolic-alone.json": (variant(upload_hv, B + ["measurements"], {"systolic": 138}), "/items/0/body/measurements dependentRequired"),
        "invalid-home-visit-note-too-long.json": (variant(upload_hv, B + ["note"], "x" * 501), "/items/0/body/note maxLength"),
        "invalid-home-visit-latitude.json": (variant(upload_hv, B + ["latitude"], -25.43), "/items/0/body/latitude schema"),
        "invalid-withdrawal-reason-short.json": (variant(upload_withdrawal, B + ["reason"], "errado"), "/items/0/body/reason minLength"),
        "invalid-collective-activity-pse-topic-18.json": (variant(upload_ca, B + ["health_topics"], [18]), "/items/0/body/health_topics/0 enum"),
        "invalid-collective-activity-two-places.json": (variant(upload_ca, B + ["place_health_unit_id"], UNIT), "/items/0/body oneOf"),  # Q15
        "invalid-collective-activity-inep.json": (variant(upload_ca, B + ["inep"], "41000000"), "/items/0/body/inep schema"),  # Q15
        "invalid-patient-cpf-masked.json": (variant(upload_patient, B + ["new_patient", "cpf"], "390.533.447-05"), "/items/0/body/new_patient/cpf pattern"),
        "invalid-household-version-without-id.json": (variant(upload_household, B + ["household_id"], DELETE), "/items/0/body dependentRequired"),
        "invalid-household-property-type-integer.json": (variant(upload_household, B + ["property_type"], 1), "/items/0/body/property_type enum"),
    }),
    ("upload-response-v1.json", {
        "upload-response-accepted.json": (resp_accepted(), "valid"),
        "upload-response-duplicate.json": (resp_duplicate(), "valid"),
        "upload-response-rejected.json": (resp_rejected(), "valid"),
        "upload-response-duplicate-suspected.json": (resp_duplicate_suspected(), "valid"),
        "upload-response-stale-household.json": (resp_stale_household(), "valid"),
        "upload-response-stale-registration.json": (resp_stale_registration(), "valid"),
        "invalid-response-accepted-without-id.json": (variant(resp_accepted, ["results", 0, "id"], DELETE), "/results/0 required"),
        "invalid-response-status-queued.json": (variant(resp_accepted, ["results", 0, "status"], "queued"), "/results/0/status enum"),
        "invalid-response-candidate-with-cpf.json": (variant(resp_duplicate_suspected, ["results", 0, "candidates", 0, "cpf"], "11144477735"), "/results/0/candidates/0/cpf schema"),
        "invalid-response-stale-without-current.json": (variant(resp_stale_household, ["results", 0, "current"], DELETE), "/results/0 required"),
        "invalid-response-rejected-with-id.json": (variant(resp_rejected, ["results", 0, "id"], VISIT), "/results/0/id schema"),
        "invalid-response-candidates-without-duplicate-suspected.json": (variant(resp_duplicate_suspected, ["results", 0, "error"], "cpf_taken"), "/results/0/error const"),
        "invalid-response-stale-household-with-cpf.json": (variant(resp_stale_household, ["results", 0, "current", "families", 0, "responsible", "cpf"], "39053344705"), "/results/0/current/families/0/responsible/cpf schema"),
    }),
    ("changes-response-v1.json", {
        "changes-first-page.json": (page_first(), "valid"),
        "changes-last-page.json": (page_last(), "valid"),
        "changes-removals.json": (page_removals(), "valid"),
        "changes-reset.json": (page_reset(), "valid"),
        "invalid-changes-who-null-on-last-page.json": (variant(page_last, ["who_to_visit"], None), "/who_to_visit object"),
        "invalid-changes-who-on-middle-page.json": (variant(page_first, ["who_to_visit"], who_to_visit()), "/who_to_visit null"),
        "invalid-changes-removal-reason-death.json": (variant(page_removals, ["removals", 0, "reason"], "death"), "/removals/0/reason enum"),
        "invalid-changes-household-without-version.json": (variant(page_first, ["items", 1, "version"], DELETE), "/items/1 required"),
        "invalid-changes-micro-area-with-version.json": (variant(page_first, ["items", 0, "version"], 1), "/items/0/version schema"),
        "invalid-changes-registration-with-cpf.json": (variant(page_first, ["items", 2, "body", "patient", "cpf"], "39053344705"), "/items/2/body/patient/cpf schema"),
        "invalid-changes-care-line-reason-with-cid.json": (variant(page_last, ["who_to_visit", "micro_areas", 0, "people", 0, "reasons", 0, "cid10"], "I10"), "/who_to_visit/micro_areas/0/people/0/reasons/0/cid10 schema"),
        "invalid-changes-cursor-null.json": (variant(page_first, ["next_cursor"], None), "/next_cursor string"),
    }),
]


def main():
    os.makedirs(OUT, exist_ok=True)
    manifest = {"suites": []}
    for schema, cases in SUITES:
        entries = []
        for name, (doc, expect) in cases.items():
            with open(os.path.join(OUT, name), "w", encoding="utf-8") as f:
                f.write(json.dumps(doc, indent=2, ensure_ascii=False) + "\n")
            entries.append({"file": name, "expect": expect})
        manifest["suites"].append({"schema": schema, "cases": entries})
    with open(os.path.join(OUT, "manifest.json"), "w", encoding="utf-8") as f:
        f.write(json.dumps(manifest, indent=2, ensure_ascii=False) + "\n")
    total = sum(len(c) for _, c in SUITES)
    print(" · ".join(f"{s} {len(c)}" for s, c in SUITES) + f" · total {total}")


if __name__ == "__main__":
    main()
```

Gere (de `$R`):

```bash
C=/Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/contracts-mod24b
mkdir -p $C/field-sync/tools && python3 $C/field-sync/tools/make_examples.py
ls $C/field-sync/examples | wc -l
grep -c '"expect": "valid"' $C/field-sync/examples/manifest.json
```

Expected: `session-response-v1.json 5 · upload-request-v1.json 26 · upload-response-v1.json 13 · changes-response-v1.json 12 · total 56`; `57` arquivos (56 + manifesto); `20` válidos.

- [ ] **Step 2: Verificador e esquemas provisórios; veja falhar**

Crie `field-sync/tools/check.sh` (é o comando do `session/README.md`, com suítes):

```bash
#!/usr/bin/env bash
# Valida field-sync/examples/manifest.json contra os esquemas com o
# json_schemer do api. Rode da raiz do monorepo, com o compose de pé.
# Uso: field-sync/tools/check.sh [esquema.json ...]   (sem argumento: todas as suítes)
set -euo pipefail
dir="$(cd "$(dirname "$0")/.." && pwd)"
python3 -c '
import json, os, sys
d, only = sys.argv[1], set(sys.argv[2:])
ex = os.path.join(d, "examples")
m = json.load(open(os.path.join(ex, "manifest.json"), encoding="utf-8"))
suites = []
for s in m["suites"]:
    if only and s["schema"] not in only:
        continue
    cases = [dict(c, doc=json.load(open(os.path.join(ex, c["file"]), encoding="utf-8"))) for c in s["cases"]]
    suites.append({"schema": s["schema"], "schema_doc": json.load(open(os.path.join(d, s["schema"]), encoding="utf-8")), "cases": cases})
print(json.dumps({"suites": suites}))
' "$dir" "$@" \
| docker compose exec -T -w /rails api bundle exec ruby -rjson -rjson_schemer -e '
input = JSON.parse(STDIN.read)
total = 0
passed = 0
input["suites"].each do |s|
  schemer = JSONSchemer.schema(s["schema_doc"])
  s["cases"].each do |c|
    total += 1
    errors = schemer.validate(c["doc"]).map { |e| p = e["data_pointer"]; "#{p.empty? ? "(root)" : p} #{e["type"]}" }.uniq
    ok = c["expect"] == "valid" ? errors.empty? : errors.include?(c["expect"])
    passed += 1 if ok
    puts "#{ok ? "ok  " : "FAIL"} #{s["schema"]} #{c["file"]} — esperado: #{c["expect"]}; obtido: #{errors.empty? ? "valid" : errors.join(" | ")}"
  end
end
puts "#{passed}/#{total} casos"
exit(passed == total ? 0 : 1)'
```

```bash
C=/Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/contracts-mod24b
chmod +x $C/field-sync/tools/check.sh
for f in session-response-v1 upload-request-v1 upload-response-v1 changes-response-v1; do
  printf '{ "$schema": "https://json-schema.org/draft/2020-12/schema", "type": "object" }\n' > $C/field-sync/$f.json
done
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude && $C/field-sync/tools/check.sh | tail -1
```

Expected (exit 1): os 20 válidos `ok`, os 36 inválidos `FAIL ... obtido: valid`; última linha `20/56 casos`.

- [ ] **Step 3: O esquema da sessão**

Substitua `field-sync/session-response-v1.json` por:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://rota-saude/schemas/field-sync/session-response-v1.json",
  "title": "rotasaude.field_sync.session_response.v1",
  "description": "GET /field/v1/session (contrato do 24b §9.3): quem é o aparelho, as microáreas que ele baixa (ACS: as de que é responsável; outros CBOs da Tabela 7: as atribuídas pelo enfermeiro, 6A) e os interruptores. Sem trilha. Fechado.",
  "type": "object",
  "additionalProperties": false,
  "required": ["contract", "server_time", "user", "professional", "teams", "features", "min_date"],
  "properties": {
    "contract": { "const": "field-sync-v1" },
    "server_time": { "$ref": "#/$defs/datetime" },
    "user": {
      "type": "object",
      "additionalProperties": false,
      "required": ["id", "name"],
      "properties": {
        "id": { "$ref": "#/$defs/uuid" },
        "name": { "type": "string", "minLength": 1 }
      }
    },
    "professional": {
      "type": "object",
      "additionalProperties": false,
      "required": ["id", "cbo", "has_cns"],
      "properties": {
        "id": { "$ref": "#/$defs/uuid" },
        "cbo": { "type": "string", "pattern": "^[0-9]{6}$", "description": "CBO do vínculo ativo (Tabela 7 do LEDI)." },
        "has_cns": { "type": "boolean", "description": "Sem CNS a ficha de visita sai 'não gerada' (professional_without_cns)." }
      }
    },
    "teams": {
      "type": "array",
      "minItems": 1,
      "items": {
        "type": "object",
        "additionalProperties": false,
        "required": ["id", "ine", "name", "micro_areas", "out_of_area_allowed"],
        "properties": {
          "id": { "$ref": "#/$defs/uuid" },
          "ine": { "type": "string", "minLength": 1 },
          "name": { "type": "string", "minLength": 1 },
          "micro_areas": {
            "type": "array",
            "items": {
              "type": "object",
              "additionalProperties": false,
              "required": ["id", "code", "name", "scope"],
              "properties": {
                "id": { "$ref": "#/$defs/uuid" },
                "code": { "$ref": "#/$defs/micro_area_code" },
                "name": { "type": "string", "minLength": 1 },
                "scope": { "enum": ["responsible", "assigned"], "description": "responsible: o ACS da microárea; assigned: atribuída pelo enfermeiro (6A). Não há 'team': enfermeiro e médico não têm aparelho." }
              }
            }
          },
          "out_of_area_allowed": { "type": "boolean", "description": "Só o ACS registra fora de área (C3)." }
        }
      }
    },
    "features": {
      "type": "object",
      "additionalProperties": false,
      "required": ["home_visits", "collective_activities", "territory_registry"],
      "properties": {
        "home_visits": { "type": "boolean" },
        "collective_activities": { "type": "boolean" },
        "territory_registry": { "type": "boolean" }
      }
    },
    "min_date": { "$ref": "#/$defs/date", "description": "Primeiro dia aceito pelo prazo do SIAPS (§3, C2)." }
  },
  "$defs": {
    "uuid": { "type": "string", "pattern": "^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$" },
    "datetime": { "type": "string", "pattern": "^[0-9]{4}-[0-9]{2}-[0-9]{2}T[0-9]{2}:[0-9]{2}:[0-9]{2}(\\.[0-9]{1,6})?(Z|[+-][0-9]{2}:[0-9]{2})$", "description": "ISO 8601 com segundos e fuso explícito (Z ou ±hh:mm); nunca hora local sem fuso." },
    "date": { "type": "string", "pattern": "^[0-9]{4}-[0-9]{2}-[0-9]{2}$" },
    "micro_area_code": { "type": "string", "pattern": "^[0-9]{2}$" }
  }
}
```

- [ ] **Step 4: O esquema do pedido de envio**

Substitua `field-sync/upload-request-v1.json` por:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://rota-saude/schemas/field-sync/upload-request-v1.json",
  "title": "rotasaude.field_sync.upload_request.v1",
  "description": "POST /field/v1/uploads (contrato do 24b §9.3): de 1 a 50 itens, processados em ordem, cada um na sua transação; idempotente pelo client_uuid do item. O esquema confere a forma (tipos, domínios, faixas, objetos fechados); as relações entre campos (desfecho × motivos, imóvel × pessoa, tipo de atividade × temas, quem pode o quê) estão em dado em domains-v1.json e são decididas pelo api, que devolve o item rejected com o código.",
  "type": "object",
  "additionalProperties": false,
  "required": ["items"],
  "properties": {
    "items": { "type": "array", "minItems": 1, "maxItems": 50, "items": { "$ref": "#/$defs/item" } }
  },
  "$defs": {
    "uuid": { "type": "string", "pattern": "^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$" },
    "uuid_or_null": { "type": ["string", "null"], "pattern": "^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$" },
    "datetime": { "type": "string", "pattern": "^[0-9]{4}-[0-9]{2}-[0-9]{2}T[0-9]{2}:[0-9]{2}:[0-9]{2}(\\.[0-9]{1,6})?(Z|[+-][0-9]{2}:[0-9]{2})$", "description": "ISO 8601 com segundos e fuso explícito (Z ou ±hh:mm); nunca hora local sem fuso." },
    "date": { "type": "string", "pattern": "^[0-9]{4}-[0-9]{2}-[0-9]{2}$" },
    "property_type": { "enum": ["1", "2", "3", "4", "5", "6", "7", "8", "9", "10", "11", "12", "13", "99"], "description": "TerritoryRegistry::Catalog::PROPERTY_TYPES (texto, como no 24a). Não residencial: 2, 3, 4, 5, 6, 12, 13." },
    "no_cpf_reason": { "enum": ["2", "3", "4", "5", "6", "7", "8", "9", "10", "11", "12", "13"], "description": "TerritoryRegistry::Catalog::NO_CPF_REASONS (justificativa de não ter CPF, contrato do 24a §15)." },
    "address": {
      "type": "object",
      "additionalProperties": false,
      "description": "Chaves de TerritoryRegistry::Address::KEYS (24a L6). Objeto vazio no domicílio recusado sem endereço. Tamanhos são do api (invalid_household com path).",
      "properties": {
        "zip": { "type": ["string", "null"], "pattern": "^[0-9]{8}$" },
        "street_type": { "type": ["string", "null"] },
        "street": { "type": ["string", "null"] },
        "number": { "type": ["string", "null"] },
        "no_number": { "type": ["boolean", "null"] },
        "complement": { "type": ["string", "null"] },
        "neighborhood_id": { "$ref": "#/$defs/uuid_or_null" },
        "district": { "type": ["string", "null"] },
        "reference": { "type": ["string", "null"] }
      }
    },
    "item": {
      "type": "object",
      "additionalProperties": false,
      "required": ["client_uuid", "type", "body"],
      "properties": {
        "client_uuid": { "$ref": "#/$defs/uuid", "description": "Gerado pelo app, um por item; chave de idempotência (field_sync_receipts). Só itens aceitos vão ao livro." },
        "type": { "enum": ["home_visit", "home_visit_withdrawal", "collective_activity", "patient", "household", "individual_registration"] },
        "body": { "type": "object" }
      },
      "allOf": [
        { "if": { "required": ["type"], "properties": { "type": { "const": "home_visit" } } }, "then": { "properties": { "body": { "$ref": "#/$defs/home_visit_body" } } } },
        { "if": { "required": ["type"], "properties": { "type": { "const": "home_visit_withdrawal" } } }, "then": { "properties": { "body": { "$ref": "#/$defs/home_visit_withdrawal_body" } } } },
        { "if": { "required": ["type"], "properties": { "type": { "const": "collective_activity" } } }, "then": { "properties": { "body": { "$ref": "#/$defs/collective_activity_body" } } } },
        { "if": { "required": ["type"], "properties": { "type": { "const": "patient" } } }, "then": { "properties": { "body": { "$ref": "#/$defs/patient_body" } } } },
        { "if": { "required": ["type"], "properties": { "type": { "const": "household" } } }, "then": { "properties": { "body": { "$ref": "#/$defs/household_body" } } } },
        { "if": { "required": ["type"], "properties": { "type": { "const": "individual_registration" } } }, "then": { "properties": { "body": { "$ref": "#/$defs/individual_registration_body" } } } }
      ]
    },
    "home_visit_body": {
      "type": "object",
      "additionalProperties": false,
      "description": "Corpo do POST /home_visits (§6) sem client_uuid (vai no item). shift nunca vem: o api deriva de occurred_at no fuso da cidade. patient_client_uuid cita um item patient aceito antes (C11). latitude e longitude ficam para o 24c.",
      "required": ["team_id", "occurred_at", "subject", "outcome", "reasons", "shared"],
      "properties": {
        "team_id": { "$ref": "#/$defs/uuid" },
        "occurred_at": { "$ref": "#/$defs/datetime" },
        "household_id": { "$ref": "#/$defs/uuid" },
        "micro_area_id": { "$ref": "#/$defs/uuid" },
        "out_of_area": { "type": "boolean" },
        "property_type": { "$ref": "#/$defs/property_type" },
        "subject": { "enum": ["resident", "family_responsible", "none"] },
        "patient_id": { "$ref": "#/$defs/uuid" },
        "patient_client_uuid": { "$ref": "#/$defs/uuid" },
        "new_patient": { "$ref": "#/$defs/new_patient" },
        "family_id": { "$ref": "#/$defs/uuid" },
        "duplicates_reviewed": { "type": "boolean" },
        "outcome": { "enum": [1, 2, 3], "description": "1 Realizada, 2 Recusada, 3 Ausente." },
        "reasons": { "type": "array", "uniqueItems": true, "items": { "$ref": "#/$defs/visit_reason" } },
        "shared": { "type": "boolean" },
        "measurements": { "$ref": "#/$defs/visit_measurements" },
        "note": { "type": "string", "minLength": 1, "maxLength": 500, "description": "Observação interna: nunca vai ao LEDI, à linha do tempo nem ao cidadão." }
      }
    },
    "visit_reason": {
      "enum": [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22, 23, 24, 25, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38],
      "description": "MotivoVisita (domains-v1.json, home_visit.motivo_visita). Não existe o 26."
    },
    "visit_measurements": {
      "type": "object",
      "additionalProperties": false,
      "minProperties": 1,
      "description": "Só as medidas aferidas (§3). Casas decimais e as proibições (desfecho 2 e 3, não residencial) são do api.",
      "properties": {
        "weight_kg": { "type": "number", "minimum": 0.5, "maximum": 500 },
        "height_cm": { "type": "number", "minimum": 20, "maximum": 250 },
        "systolic": { "type": "integer", "minimum": 0, "maximum": 999 },
        "diastolic": { "type": "integer", "minimum": 0, "maximum": 999 },
        "temperature_c": { "type": "number", "minimum": 20, "maximum": 45 },
        "glucose_mg_dl": { "type": "integer", "minimum": 0, "maximum": 800 },
        "glucose_type": { "enum": [0, 1, 2, 3] }
      },
      "dependentRequired": {
        "systolic": ["diastolic"],
        "diastolic": ["systolic"],
        "glucose_mg_dl": ["glucose_type"],
        "glucose_type": ["glucose_mg_dl"]
      }
    },
    "new_patient": {
      "type": "object",
      "additionalProperties": false,
      "required": ["full_name", "birth_date", "sex"],
      "description": "Registro rápido (§5, 7A): cria o paciente na transação do item. cpf ou no_cpf_reason (regra do api: cpf_or_reason_required).",
      "properties": {
        "full_name": { "type": "string", "minLength": 1 },
        "birth_date": { "$ref": "#/$defs/date" },
        "sex": { "enum": ["male", "female"] },
        "mother_name": { "type": "string", "minLength": 1 },
        "cpf": { "type": "string", "pattern": "^[0-9]{11}$" },
        "no_cpf_reason": { "$ref": "#/$defs/no_cpf_reason" },
        "cns": { "type": "string", "pattern": "^[0-9]{15}$" }
      }
    },
    "registration_new_patient": {
      "type": "object",
      "additionalProperties": false,
      "required": ["full_name", "birth_date", "sex"],
      "description": "new_patient do cadastro individual do 24a (§5), que aceita também social_name.",
      "properties": {
        "full_name": { "type": "string", "minLength": 1 },
        "social_name": { "type": "string", "minLength": 1 },
        "birth_date": { "$ref": "#/$defs/date" },
        "sex": { "enum": ["male", "female"] },
        "mother_name": { "type": "string", "minLength": 1 },
        "cpf": { "type": "string", "pattern": "^[0-9]{11}$" },
        "no_cpf_reason": { "$ref": "#/$defs/no_cpf_reason" },
        "cns": { "type": "string", "pattern": "^[0-9]{15}$" }
      }
    },
    "home_visit_withdrawal_body": {
      "type": "object",
      "additionalProperties": false,
      "required": ["reason"],
      "description": "Retirada (§2): visit_client_uuid (visita enviada pelo app) ou visit_id, um dos dois (regra do api). Só com a folha aberta.",
      "properties": {
        "visit_client_uuid": { "$ref": "#/$defs/uuid" },
        "visit_id": { "$ref": "#/$defs/uuid" },
        "reason": { "type": "string", "minLength": 10, "maxLength": 500 }
      }
    },
    "collective_activity_body": {
      "type": "object",
      "additionalProperties": false,
      "description": "Corpo do POST /collective_activities (§7): a atividade inteira, sem rascunho. Lista ausente = vazia. Local plano (Q15): place_health_unit_id (unidade ativa com CNES) ou place_text, exatamente um; INEP nunca (sem PSE). O tema 18 não existe aqui. duplicates_reviewed só dentro de cada participante (Q15).",
      "required": ["team_id", "held_on", "shift", "activity_type", "participants_count", "altered_count"],
      "oneOf": [{ "required": ["place_health_unit_id"] }, { "required": ["place_text"] }],
      "properties": {
        "team_id": { "$ref": "#/$defs/uuid" },
        "held_on": { "$ref": "#/$defs/date" },
        "shift": { "enum": [1, 2, 3] },
        "activity_type": { "enum": [1, 2, 3, 4, 5, 6, 7] },
        "place_health_unit_id": { "$ref": "#/$defs/uuid" },
        "place_text": { "type": "string", "minLength": 1, "maxLength": 250 },
        "participants_count": { "type": "integer", "minimum": 1, "maximum": 999 },
        "altered_count": { "type": "integer", "minimum": 0, "maximum": 999 },
        "meeting_topics": { "type": "array", "uniqueItems": true, "items": { "enum": [1, 2, 3, 4, 5, 6, 7, 8] } },
        "target_audience": { "type": "array", "uniqueItems": true, "items": { "enum": [1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 12, 13, 14, 15, 16, 17, 18, 19, 20] } },
        "health_topics": { "type": "array", "uniqueItems": true, "items": { "enum": [1, 4, 5, 6, 7, 8, 10, 13, 14, 15, 16, 17, 19, 21, 29, 31, 32] } },
        "health_practices": { "type": "array", "uniqueItems": true, "items": { "enum": [2, 3, 9, 11, 12, 20, 22, 23, 24, 25, 26, 27, 28, 30, 33, 34] } },
        "procedure_code": { "type": "string", "pattern": "^[0-9]{10}$" },
        "professionals": {
          "type": "array",
          "uniqueItems": true,
          "items": {
            "type": "object",
            "additionalProperties": false,
            "required": ["professional_id"],
            "properties": { "professional_id": { "$ref": "#/$defs/uuid" } }
          }
        },
        "participants": { "type": "array", "maxItems": 999, "items": { "$ref": "#/$defs/participant" } }
      }
    },
    "participant": {
      "type": "object",
      "additionalProperties": false,
      "description": "patient_id, patient_client_uuid ou new_patient, um só (regra do api). duplicates_reviewed acompanha o new_patient (Q15).",
      "properties": {
        "patient_id": { "$ref": "#/$defs/uuid" },
        "patient_client_uuid": { "$ref": "#/$defs/uuid" },
        "new_patient": { "$ref": "#/$defs/new_patient" },
        "duplicates_reviewed": { "type": "boolean" },
        "altered_evaluation": { "type": "boolean" },
        "measurements": {
          "type": "object",
          "additionalProperties": false,
          "minProperties": 1,
          "properties": {
            "weight_kg": { "type": "number", "minimum": 0.5, "maximum": 500 },
            "height_cm": { "type": "number", "minimum": 20, "maximum": 250 }
          }
        },
        "smoking": {
          "type": "object",
          "additionalProperties": false,
          "minProperties": 1,
          "properties": {
            "quit": { "type": "boolean" },
            "dropped_out": { "type": "boolean" }
          }
        }
      }
    },
    "patient_body": {
      "type": "object",
      "additionalProperties": false,
      "required": ["new_patient"],
      "properties": {
        "new_patient": { "$ref": "#/$defs/new_patient" },
        "duplicates_reviewed": { "type": "boolean" }
      }
    },
    "household_body": {
      "type": "object",
      "additionalProperties": false,
      "description": "Corpo do POST/PATCH /territory/households do 24a (§4, L6, L7); na atualização, household_id e version (a esperada; vencida → stale_version). Família tem rota própria e não entra no v1 (K-4). living_conditions: seção do dicionário do 24a (§15), conferida pelo api.",
      "required": ["micro_area_id", "property_type"],
      "properties": {
        "household_id": { "$ref": "#/$defs/uuid" },
        "version": { "type": "integer", "minimum": 1 },
        "micro_area_id": { "$ref": "#/$defs/uuid" },
        "address": { "$ref": "#/$defs/address" },
        "property_type": { "$ref": "#/$defs/property_type" },
        "living_conditions": { "type": "object" },
        "contact_phone": { "type": ["string", "null"] },
        "refused": { "type": "boolean" },
        "status": { "enum": ["active", "inactive"] }
      },
      "dependentRequired": { "household_id": ["version"], "version": ["household_id"] }
    },
    "individual_registration_body": {
      "type": "object",
      "additionalProperties": false,
      "description": "Corpo do POST/PATCH /territory/registrations do 24a (§5, L4, L7); na atualização, registration_id e version. identification, sociodemographic, health_conditions e street_situation: seções do dicionário do 24a (§15), conferidas pelo api.",
      "properties": {
        "registration_id": { "$ref": "#/$defs/uuid" },
        "version": { "type": "integer", "minimum": 1 },
        "patient_id": { "$ref": "#/$defs/uuid" },
        "citizen_id": { "$ref": "#/$defs/uuid" },
        "new_patient": { "$ref": "#/$defs/registration_new_patient" },
        "duplicates_reviewed": { "type": "boolean" },
        "household_id": { "$ref": "#/$defs/uuid" },
        "family_id": { "$ref": "#/$defs/uuid" },
        "is_responsible": { "type": "boolean" },
        "team_id": { "$ref": "#/$defs/uuid" },
        "identification": { "type": "object" },
        "patient_demographics": {
          "type": "object",
          "additionalProperties": false,
          "properties": {
            "race_color": { "type": ["string", "null"] },
            "ethnicity": { "type": ["string", "null"] }
          }
        },
        "sociodemographic": { "type": "object" },
        "health_conditions": { "type": "object" },
        "street_situation": { "type": ["object", "null"] },
        "refused": { "type": "boolean" }
      },
      "dependentRequired": { "registration_id": ["version"], "version": ["registration_id"] }
    }
  }
}
```

- [ ] **Step 5: As formas comuns do download (`$defs` repetidos)**

Os dois esquemas de resposta (Steps 6 e 7) levam **exatamente** este bloco dentro de `$defs`, além dos `$defs` próprios de cada um. `uuid`, `uuid_or_null`, `datetime`, `date`, `property_type` e `address` são os mesmos do Step 4 (repetidos aqui por inteiro, sem mudar uma vírgula); `check_domains.py` (Task 2) confere a igualdade.

```json
    "uuid": { "type": "string", "pattern": "^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$" },
    "uuid_or_null": { "type": ["string", "null"], "pattern": "^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$" },
    "datetime": { "type": "string", "pattern": "^[0-9]{4}-[0-9]{2}-[0-9]{2}T[0-9]{2}:[0-9]{2}:[0-9]{2}(\\.[0-9]{1,6})?(Z|[+-][0-9]{2}:[0-9]{2})$", "description": "ISO 8601 com segundos e fuso explícito (Z ou ±hh:mm); nunca hora local sem fuso." },
    "datetime_or_null": { "type": ["string", "null"], "pattern": "^[0-9]{4}-[0-9]{2}-[0-9]{2}T[0-9]{2}:[0-9]{2}:[0-9]{2}(\\.[0-9]{1,6})?(Z|[+-][0-9]{2}:[0-9]{2})$" },
    "date": { "type": "string", "pattern": "^[0-9]{4}-[0-9]{2}-[0-9]{2}$" },
    "date_or_null": { "type": ["string", "null"], "pattern": "^[0-9]{4}-[0-9]{2}-[0-9]{2}$" },
    "micro_area_code": { "type": "string", "pattern": "^[0-9]{2}$" },
    "micro_area_ref": {
      "type": "object",
      "additionalProperties": false,
      "required": ["id", "code"],
      "properties": { "id": { "$ref": "#/$defs/uuid" }, "code": { "$ref": "#/$defs/micro_area_code" } }
    },
    "team_ref": {
      "type": "object",
      "additionalProperties": false,
      "required": ["id", "ine", "name"],
      "properties": {
        "id": { "$ref": "#/$defs/uuid" },
        "ine": { "type": "string", "minLength": 1 },
        "name": { "type": "string", "minLength": 1 }
      }
    },
    "registered_by": {
      "type": "object",
      "additionalProperties": false,
      "required": ["name"],
      "properties": { "name": { "type": ["string", "null"] } }
    },
    "property_type": { "enum": ["1", "2", "3", "4", "5", "6", "7", "8", "9", "10", "11", "12", "13", "99"], "description": "TerritoryRegistry::Catalog::PROPERTY_TYPES (texto, como no 24a). Não residencial: 2, 3, 4, 5, 6, 12, 13." },
    "address": {
      "type": "object",
      "additionalProperties": false,
      "description": "Chaves de TerritoryRegistry::Address::KEYS (24a L6). Objeto vazio no domicílio recusado sem endereço. Tamanhos são do api (invalid_household com path).",
      "properties": {
        "zip": { "type": ["string", "null"], "pattern": "^[0-9]{8}$" },
        "street_type": { "type": ["string", "null"] },
        "street": { "type": ["string", "null"] },
        "number": { "type": ["string", "null"] },
        "no_number": { "type": ["boolean", "null"] },
        "complement": { "type": ["string", "null"] },
        "neighborhood_id": { "$ref": "#/$defs/uuid_or_null" },
        "district": { "type": ["string", "null"] },
        "reference": { "type": ["string", "null"] }
      }
    },
    "household": {
      "type": "object",
      "additionalProperties": false,
      "description": "<household> do 24a (§2; L6, L7, L30; TerritoryRegistry::Json.household com full: true): com families e living_conditions. Responsável da família só com patient_id e display_name, nunca CPF.",
      "required": ["id", "micro_area", "address", "property_type", "contact_phone", "refused", "status", "version", "registered_at", "registered_by", "families", "living_conditions"],
      "properties": {
        "id": { "$ref": "#/$defs/uuid" },
        "micro_area": { "$ref": "#/$defs/micro_area_ref" },
        "address": { "$ref": "#/$defs/address" },
        "property_type": { "$ref": "#/$defs/property_type" },
        "contact_phone": { "type": ["string", "null"] },
        "refused": { "type": "boolean" },
        "status": { "enum": ["active", "inactive"] },
        "version": { "type": "integer", "minimum": 1 },
        "registered_at": { "$ref": "#/$defs/datetime" },
        "registered_by": { "$ref": "#/$defs/registered_by" },
        "families": { "type": "array", "items": { "$ref": "#/$defs/family" } },
        "living_conditions": { "type": "object", "description": "Seção do dicionário do 24a (§15): chaves e domínios do api, não deste esquema (K-2)." }
      }
    },
    "family": {
      "type": "object",
      "additionalProperties": false,
      "required": ["id", "responsible", "income_band", "members_count", "living_since", "moved_out", "responsible_exited"],
      "properties": {
        "id": { "$ref": "#/$defs/uuid" },
        "responsible": {
          "type": "object",
          "additionalProperties": false,
          "required": ["patient_id", "display_name"],
          "properties": {
            "patient_id": { "$ref": "#/$defs/uuid" },
            "display_name": { "type": "string", "minLength": 1 }
          }
        },
        "income_band": { "type": ["string", "null"] },
        "members_count": { "type": "integer", "minimum": 0 },
        "living_since": { "type": ["string", "null"] },
        "moved_out": { "type": "boolean" },
        "responsible_exited": { "type": "boolean" }
      }
    },
    "registration": {
      "type": "object",
      "additionalProperties": false,
      "description": "<registration> do 24a (§2; L4, L7, L9, L21, L24; TerritoryRegistry::Json.registration). CPF nunca: só has_cpf; CNS só mascarado.",
      "required": ["id", "patient", "patient_demographics", "household_id", "family_id", "is_responsible", "team", "micro_area", "identification", "sociodemographic", "health_conditions", "street_situation", "refused", "exit", "status", "version", "registered_at", "registered_by", "identity_changed_at", "identity_review_pending"],
      "properties": {
        "id": { "$ref": "#/$defs/uuid" },
        "patient": {
          "type": "object",
          "additionalProperties": false,
          "required": ["id", "display_name", "full_name", "mother_name", "birth_date", "sex", "has_cpf", "no_cpf_reason", "cns_masked", "identity_locked"],
          "properties": {
            "id": { "$ref": "#/$defs/uuid" },
            "display_name": { "type": "string", "minLength": 1 },
            "full_name": { "type": "string", "minLength": 1 },
            "mother_name": { "type": ["string", "null"] },
            "birth_date": { "$ref": "#/$defs/date_or_null" },
            "sex": { "enum": ["male", "female", null] },
            "has_cpf": { "type": "boolean" },
            "no_cpf_reason": { "enum": ["2", "3", "4", "5", "6", "7", "8", "9", "10", "11", "12", "13", null] },
            "cns_masked": { "type": ["string", "null"] },
            "identity_locked": { "type": "boolean" }
          }
        },
        "patient_demographics": {
          "type": "object",
          "additionalProperties": false,
          "required": ["race_color", "ethnicity"],
          "properties": {
            "race_color": { "type": ["string", "null"] },
            "ethnicity": { "type": ["string", "null"] }
          }
        },
        "household_id": { "$ref": "#/$defs/uuid_or_null" },
        "family_id": { "$ref": "#/$defs/uuid_or_null" },
        "is_responsible": { "type": "boolean" },
        "team": { "$ref": "#/$defs/team_ref" },
        "micro_area": { "anyOf": [{ "$ref": "#/$defs/micro_area_ref" }, { "type": "null" }] },
        "identification": { "type": "object" },
        "sociodemographic": { "type": "object" },
        "health_conditions": { "type": "object" },
        "street_situation": { "type": ["object", "null"] },
        "refused": { "type": "boolean" },
        "exit": {
          "type": ["object", "null"],
          "additionalProperties": false,
          "required": ["reason", "on", "death_on", "death_certificate"],
          "properties": {
            "reason": { "enum": ["moved", "death"] },
            "on": { "$ref": "#/$defs/date" },
            "death_on": { "$ref": "#/$defs/date_or_null" },
            "death_certificate": { "type": ["string", "null"] }
          }
        },
        "status": { "enum": ["valid", "due_soon", "expired", "missing", "missing_identifier", "refused", "exited"] },
        "version": { "type": "integer", "minimum": 1 },
        "registered_at": { "$ref": "#/$defs/datetime" },
        "registered_by": { "$ref": "#/$defs/registered_by" },
        "identity_changed_at": { "$ref": "#/$defs/datetime_or_null" },
        "identity_review_pending": { "type": "boolean" }
      }
    }
```

- [ ] **Step 6: O esquema da resposta do envio**

Substitua `field-sync/upload-response-v1.json` por (o comentário `«bloco do Step 5»` é substituído pelo bloco do Step 5, por inteiro; o arquivo final é JSON puro):

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://rota-saude/schemas/field-sync/upload-response-v1.json",
  "title": "rotasaude.field_sync.upload_response.v1",
  "description": "Resposta 200 do POST /field/v1/uploads (contrato do 24b §9.3): um resultado por item, na ordem do pedido. accepted e duplicate trazem o id gravado; rejected traz o código (e path, e candidates só em duplicate_suspected); stale_version traz a versão atual do domicílio ou do cadastro para o app resolver o conflito (24c). O envelope inválido é 422 { error: invalid_request, path }, fora deste esquema.",
  "type": "object",
  "additionalProperties": false,
  "required": ["results"],
  "properties": {
    "results": { "type": "array", "maxItems": 50, "items": { "$ref": "#/$defs/result" } }
  },
  "$defs": {
    «bloco do Step 5»,
    "error_code": { "type": "string", "pattern": "^[a-z][a-z0-9_]*$", "maxLength": 64, "description": "Código do contrato do 24b §13, dos herdados e do 24a. Os conhecidos estão em domains-v1.json (errors.item); código novo do api é MINOR e o app mostra uma frase genérica (K-8)." },
    "candidate": {
      "type": "object",
      "additionalProperties": false,
      "required": ["patient_id", "display_name", "birth_date", "has_cpf"],
      "description": "Candidato a duplicado (§5): sem CPF, CNS nem mãe. registration_id só vem do cadastro individual do 24a (L31), K-9.",
      "properties": {
        "patient_id": { "$ref": "#/$defs/uuid" },
        "display_name": { "type": "string", "minLength": 1 },
        "birth_date": { "$ref": "#/$defs/date_or_null" },
        "has_cpf": { "type": "boolean" },
        "registration_id": { "$ref": "#/$defs/uuid_or_null" }
      }
    },
    "result": {
      "type": "object",
      "additionalProperties": false,
      "required": ["client_uuid", "status", "type"],
      "properties": {
        "client_uuid": { "$ref": "#/$defs/uuid" },
        "status": { "enum": ["accepted", "duplicate", "rejected", "stale_version"] },
        "type": { "enum": ["home_visit", "home_visit_withdrawal", "collective_activity", "patient", "household", "individual_registration"] },
        "id": { "$ref": "#/$defs/uuid" },
        "error": { "$ref": "#/$defs/error_code" },
        "path": { "type": "string", "minLength": 1 },
        "candidates": { "type": "array", "maxItems": 10, "items": { "$ref": "#/$defs/candidate" } },
        "current_version": { "type": "integer", "minimum": 1 },
        "current": { "type": "object" }
      },
      "allOf": [
        { "if": { "required": ["status"], "properties": { "status": { "enum": ["accepted", "duplicate"] } } },
          "then": { "required": ["id"], "properties": { "error": false, "path": false, "candidates": false, "current_version": false, "current": false } } },
        { "if": { "required": ["status"], "properties": { "status": { "const": "rejected" } } },
          "then": { "required": ["error"], "properties": { "id": false, "current_version": false, "current": false } } },
        { "if": { "required": ["status"], "properties": { "status": { "const": "stale_version" } } },
          "then": { "required": ["id", "current_version", "current"], "properties": { "type": { "enum": ["household", "individual_registration"] }, "error": false, "path": false, "candidates": false } } },
        { "if": { "required": ["status", "type"], "properties": { "status": { "const": "stale_version" }, "type": { "const": "household" } } },
          "then": { "properties": { "current": { "$ref": "#/$defs/household" } } } },
        { "if": { "required": ["status", "type"], "properties": { "status": { "const": "stale_version" }, "type": { "const": "individual_registration" } } },
          "then": { "properties": { "current": { "$ref": "#/$defs/registration" } } } },
        { "if": { "required": ["candidates"] },
          "then": { "properties": { "error": { "const": "duplicate_suspected" } } } }
      ]
    }
  }
}
```

- [ ] **Step 7: O esquema das páginas do download**

Substitua `field-sync/changes-response-v1.json` por (mesma regra do `«bloco do Step 5»`):

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://rota-saude/schemas/field-sync/changes-response-v1.json",
  "title": "rotasaude.field_sync.changes_response.v1",
  "description": "GET /field/v1/changes (contrato do 24b §9.3): uma página do download incremental do escopo do aparelho, em ordem (updated_at, id). reset: true = o escopo mudou desde o cursor; o app apaga o que baixou (nunca o que falta enviar) e aplica as páginas. who_to_visit vem só na última página (has_more: false), sempre completo; nas outras, null. next_cursor é opaco e sempre vem: o app o guarda para a próxima sincronização (K-10).",
  "type": "object",
  "additionalProperties": false,
  "required": ["reset", "items", "removals", "who_to_visit", "next_cursor", "has_more"],
  "properties": {
    "reset": { "type": "boolean" },
    "items": { "type": "array", "maxItems": 500, "items": { "$ref": "#/$defs/change_item" } },
    "removals": { "type": "array", "items": { "$ref": "#/$defs/removal" } },
    "who_to_visit": { "type": ["object", "null"] },
    "next_cursor": { "type": "string", "minLength": 1, "maxLength": 1024, "pattern": "^[A-Za-z0-9_-]+$", "description": "base64url sem padding; nenhum dado pessoal." },
    "has_more": { "type": "boolean" }
  },
  "allOf": [
    { "if": { "required": ["has_more"], "properties": { "has_more": { "const": true } } },
      "then": { "properties": { "who_to_visit": { "type": "null" } } },
      "else": { "properties": { "who_to_visit": { "$ref": "#/$defs/who_to_visit" } } } }
  ],
  "$defs": {
    «bloco do Step 5»,
    "micro_area": {
      "type": "object",
      "additionalProperties": false,
      "description": "<micro_area> do 24a (§2; TerritoryRegistry::Json.micro_area).",
      "required": ["id", "team", "code", "name", "acs", "active"],
      "properties": {
        "id": { "$ref": "#/$defs/uuid" },
        "team": { "$ref": "#/$defs/team_ref" },
        "code": { "$ref": "#/$defs/micro_area_code" },
        "name": { "type": "string", "minLength": 1 },
        "acs": {
          "type": ["object", "null"],
          "additionalProperties": false,
          "required": ["professional_id", "name"],
          "properties": {
            "professional_id": { "$ref": "#/$defs/uuid" },
            "name": { "type": ["string", "null"] }
          }
        },
        "active": { "type": "boolean" }
      }
    },
    "change_item": {
      "type": "object",
      "additionalProperties": false,
      "required": ["type", "id", "body"],
      "properties": {
        "type": { "enum": ["micro_area", "household", "registration"] },
        "id": { "$ref": "#/$defs/uuid" },
        "version": { "type": "integer", "minimum": 1 },
        "body": { "type": "object" }
      },
      "allOf": [
        { "if": { "required": ["type"], "properties": { "type": { "const": "micro_area" } } },
          "then": { "properties": { "version": false, "body": { "$ref": "#/$defs/micro_area" } } } },
        { "if": { "required": ["type"], "properties": { "type": { "const": "household" } } },
          "then": { "required": ["version"], "properties": { "body": { "$ref": "#/$defs/household" } } } },
        { "if": { "required": ["type"], "properties": { "type": { "const": "registration" } } },
          "then": { "required": ["version"], "properties": { "body": { "$ref": "#/$defs/registration" } } } }
      ]
    },
    "removal": {
      "type": "object",
      "additionalProperties": false,
      "required": ["type", "id", "reason"],
      "description": "De field_sync_tombstones (moved, erased) ou da troca de escopo (out_of_scope).",
      "properties": {
        "type": { "enum": ["household", "registration"] },
        "id": { "$ref": "#/$defs/uuid" },
        "reason": { "enum": ["moved", "erased", "out_of_scope"] }
      }
    },
    "who_to_visit": {
      "type": "object",
      "additionalProperties": false,
      "required": ["generated_at", "micro_areas"],
      "properties": {
        "generated_at": { "$ref": "#/$defs/datetime" },
        "micro_areas": {
          "type": "array",
          "items": {
            "type": "object",
            "additionalProperties": false,
            "required": ["micro_area_id", "people"],
            "properties": {
              "micro_area_id": { "$ref": "#/$defs/uuid" },
              "people": { "type": "array", "items": { "$ref": "#/$defs/who_item" } }
            }
          }
        }
      }
    },
    "who_item": {
      "type": "object",
      "additionalProperties": false,
      "description": "Item de 'quem visitar' (§8). Nunca CID, problema ou condição.",
      "required": ["patient_id", "display_name", "age", "household", "family_id", "micro_area", "last_visit_on", "reasons"],
      "properties": {
        "patient_id": { "$ref": "#/$defs/uuid" },
        "display_name": { "type": "string", "minLength": 1 },
        "age": { "type": "integer", "minimum": 0 },
        "household": {
          "type": ["object", "null"],
          "additionalProperties": false,
          "required": ["id", "address_short"],
          "properties": {
            "id": { "$ref": "#/$defs/uuid" },
            "address_short": { "type": "string", "minLength": 1 }
          }
        },
        "family_id": { "$ref": "#/$defs/uuid_or_null" },
        "micro_area": { "$ref": "#/$defs/micro_area_ref" },
        "last_visit_on": { "$ref": "#/$defs/date_or_null" },
        "reasons": { "type": "array", "minItems": 1, "items": { "$ref": "#/$defs/who_reason" } }
      }
    },
    "who_reason": {
      "type": "object",
      "additionalProperties": false,
      "required": ["code"],
      "properties": {
        "code": { "enum": ["care_line", "registration_due", "never_visited"] },
        "care_line": { "type": "string", "minLength": 1, "description": "Nome da linha de cuidado." },
        "practice": { "type": "string", "minLength": 1, "description": "Rótulo da boa prática." },
        "status": { "type": "string" },
        "due_on": { "$ref": "#/$defs/date" }
      },
      "allOf": [
        { "if": { "required": ["code"], "properties": { "code": { "const": "care_line" } } },
          "then": { "required": ["care_line", "practice", "status", "due_on"], "properties": { "status": { "enum": ["overdue", "due_soon"] } } } },
        { "if": { "required": ["code"], "properties": { "code": { "const": "registration_due" } } },
          "then": { "required": ["status", "due_on"], "properties": { "status": { "enum": ["due_soon", "expired"] }, "care_line": false, "practice": false } } },
        { "if": { "required": ["code"], "properties": { "code": { "const": "never_visited" } } },
          "then": { "properties": { "care_line": false, "practice": false, "status": false, "due_on": false } } }
      ]
    }
  }
}
```

- [ ] **Step 8: Rode a verificação e veja passar**

```bash
C=/Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/contracts-mod24b
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude
for f in session-response-v1 upload-request-v1 upload-response-v1 changes-response-v1; do python3 -m json.tool $C/field-sync/$f.json > /dev/null && echo "json ok $f"; done
$C/field-sync/tools/check.sh | tail -1
$C/field-sync/tools/check.sh | grep -c '^FAIL' || true
```

Expected: quatro `json ok` (nenhum `«bloco do Step 5»` sobrou); `56/56 casos` (exit 0); `0`.

Se um inválido falhar com o **mesmo** `data_pointer` e outro `type` (ex.: o `json_schemer` reportar o `then` em vez da folha), **não** troque o `expect` sozinho: pare, mostre a linha `obtido:` e reporte ao coordenador (o `expect` é a promessa ao app de qual campo está errado). Se falhar com outro `data_pointer`, o esquema está errado: corrija o esquema.

- [ ] **Step 9: Commit**

```bash
C=/Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/contracts-mod24b; G=/opt/homebrew/bin/git
$G -C $C add field-sync/session-response-v1.json field-sync/upload-request-v1.json field-sync/upload-response-v1.json \
  field-sync/changes-response-v1.json field-sync/examples field-sync/tools/make_examples.py field-sync/tools/check.sh
$G -C $C status --short | grep -v '^A ' || echo "só adições"
$G -C $C commit -m "feat: add field sync v1 schemas for session, upload and changes

New field-sync domain with the JSON Schema 2020-12 of the field app sync v1
(module 24b, ADR 0052): the session, the upload batch with one body per
item type, the per-item results (accepted, duplicate, rejected,
stale_version) and the incremental download pages with removals and the
who-to-visit list. Closed at every level except the 24a dictionary
sections. Examples are generated by a deterministic script and checked
with the api json_schemer through a suite manifest.

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

Expected: `só adições`; o commit traz 4 esquemas, 57 arquivos em `examples/` e 2 em `tools/` (63 arquivos novos).

**Relato ao coordenador:** "contracts-mod24b Task 1: 4 esquemas field-sync v1 + 56 exemplos verdes (56/56) commitados (<sha>)."

---

### Task 2: Domínios em dado, conferência entre arquivos, README e CHANGELOG

**Files (worktree `$C`):**
- Create: `field-sync/domains-v1.json`, `field-sync/tools/check_domains.py`, `field-sync/README.md`, `field-sync/CHANGELOG.md`
- Modify: `README.md`

**Interfaces:**
- Consumes: os domínios do contrato do 24b §10 e §11 (provisórios até a prova técnica do api), as regras do §3 e do §4, os códigos do §13; os rótulos da pesquisa `docs/pesquisa/2026-11-12-visita-e-coletiva-24b.md` §2 e §3.
- Produces: `domains-v1.json` com `schema = "rotasaude.field_sync.domains.v1"`, `proof: null`, `home_visit {turno, desfecho, motivo_visita [{group,label,items}], tipo_de_imovel, tipo_glicemia, rules}`, `collective_activity {tipo_atividade_coletiva, temas_para_reuniao, temas_para_saude, praticas_em_saude, publico_alvo, rules}`, `patient {sexo, justificativa_nao_possui_cpf}`, `errors {http, item}`; `check_domains.py` (`defs-iguais`, `dominios-iguais`, `erros-conhecidos`, `prova: …`; `--release` falha com `proof: null`).

- [ ] **Step 1: Escreva o verificador e veja falhar**

Crie `field-sync/tools/check_domains.py`:

```python
#!/usr/bin/env python3
"""Conferência entre os arquivos de field-sync/ (sem docker).

1. Todo $defs repetido em dois esquemas é igual.
2. Cada enum do pedido de envio é exatamente o domínio de domains-v1.json,
   e as faixas das medidas são as de domains-v1.json.
3. Todo error dos exemplos válidos de resposta está em errors.item.
--release: falha também com "proof": null (a tag só sai depois da prova técnica do api).
Uso: python3 field-sync/tools/check_domains.py [--release]
"""
import itertools
import json
import os
import sys

D = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
SCHEMAS = ["session-response-v1.json", "upload-request-v1.json", "upload-response-v1.json", "changes-response-v1.json"]


def load(name):
    with open(os.path.join(D, name), encoding="utf-8") as f:
        return json.load(f)


def main():
    bad = []
    s = {n: load(n) for n in SCHEMAS}
    for a, b in itertools.combinations(SCHEMAS, 2):
        for key in sorted(set(s[a].get("$defs", {})) & set(s[b].get("$defs", {}))):
            if s[a]["$defs"][key] != s[b]["$defs"][key]:
                bad.append(f"$defs/{key}: {a} != {b}")
    print("defs-iguais" if not bad else "DEFS DIFERENTES: " + "; ".join(bad))

    dom = load("domains-v1.json")
    req = s["upload-request-v1.json"]["$defs"]
    hv, ca = dom["home_visit"], dom["collective_activity"]
    ca_props = req["collective_activity_body"]["properties"]
    meas = req["visit_measurements"]["properties"]

    def codes(entries):
        return sorted(e["code"] for e in entries)

    pairs = {
        "motivo_visita": (sorted(i["code"] for g in hv["motivo_visita"] for i in g["items"]), sorted(req["visit_reason"]["enum"])),
        "desfecho": (codes(hv["desfecho"]), sorted(req["home_visit_body"]["properties"]["outcome"]["enum"])),
        "tipo_de_imovel": (codes(hv["tipo_de_imovel"]), sorted(req["property_type"]["enum"])),
        "tipo_glicemia": (codes(hv["tipo_glicemia"]), sorted(meas["glucose_type"]["enum"])),
        "turno": (codes(hv["turno"]), sorted(ca_props["shift"]["enum"])),
        "justificativa_nao_possui_cpf": (sorted(dom["patient"]["justificativa_nao_possui_cpf"]["codes"]), sorted(req["no_cpf_reason"]["enum"])),
        "sexo": (codes(dom["patient"]["sexo"]), sorted(req["new_patient"]["properties"]["sex"]["enum"])),
        "tipo_atividade_coletiva": (codes(ca["tipo_atividade_coletiva"]), sorted(ca_props["activity_type"]["enum"])),
        "temas_para_reuniao": (codes(ca["temas_para_reuniao"]), sorted(ca_props["meeting_topics"]["items"]["enum"])),
        "publico_alvo": (codes(ca["publico_alvo"]), sorted(ca_props["target_audience"]["items"]["enum"])),
        "temas_para_saude": (codes(ca["temas_para_saude"]), sorted(ca_props["health_topics"]["items"]["enum"])),
        "praticas_em_saude": (codes(ca["praticas_em_saude"]), sorted(ca_props["health_practices"]["items"]["enum"])),
    }
    for key, rule in hv["rules"]["measurements"].items():
        if "min" in rule:
            pairs[f"measurements.{key}"] = ([rule["min"], rule["max"]], [meas[key]["minimum"], meas[key]["maximum"]])
    diff = [k for k, (x, y) in pairs.items() if x != y]
    print("dominios-iguais" if not diff else "DOMINIOS DIFERENTES: " + ", ".join(diff))
    bad += diff

    known = set(dom["errors"]["item"])
    manifest = load(os.path.join("examples", "manifest.json"))
    unknown = []
    for suite in manifest["suites"]:
        if suite["schema"] != "upload-response-v1.json":
            continue
        for case in suite["cases"]:
            if case["expect"] != "valid":
                continue
            for r in load(os.path.join("examples", case["file"]))["results"]:
                if "error" in r and r["error"] not in known:
                    unknown.append(f'{case["file"]}: {r["error"]}')
    print("erros-conhecidos" if not unknown else "ERROS FORA DA LISTA: " + "; ".join(unknown))
    bad += unknown

    proof = dom["proof"]
    print("prova: " + ("pendente" if proof is None else f'{proof["api_commit"]} ({proof["date"]})'))
    if "--release" in sys.argv and proof is None:
        bad.append("proof")
    sys.exit(1 if bad else 0)


if __name__ == "__main__":
    main()
```

Run: `python3 $C/field-sync/tools/check_domains.py`. Expected: FAIL com `FileNotFoundError` em `domains-v1.json` (exit 1), depois de imprimir `defs-iguais` (prova de que os `$defs` repetidos na Task 1 são iguais; se sair `DEFS DIFERENTES`, corrija o esquema citado antes de seguir).

- [ ] **Step 2: Os domínios em dado**

Crie `field-sync/domains-v1.json` (provisório: códigos do contrato do 24b §10–§11 e rótulos da pesquisa §2–§3; a Task 4 confere contra a prova técnica e preenche `proof`):

```json
{
  "schema": "rotasaude.field_sync.domains.v1",
  "description": "Domínios e regras cruzadas da visita (contrato do 24b §3 e §10) e da atividade coletiva (§4 e §11), em dado, para o app de campo montar os formulários sem copiar regra à mão. Códigos LEDI inteiros (IDL 8.7.0); os do 24a em texto. Quem decide é o api: isto é o espelho. proof: null até a conferência com a prova técnica do api (dicionario-fvd.html, dicionario-fac.html e o gen-rb); a tag só sai com proof preenchido.",
  "proof": null,
  "home_visit": {
    "turno": [
      { "code": 1, "label": "Manhã" },
      { "code": 2, "label": "Tarde" },
      { "code": 3, "label": "Noite" }
    ],
    "desfecho": [
      { "code": 1, "label": "Realizada" },
      { "code": 2, "label": "Recusada" },
      { "code": 3, "label": "Ausente" }
    ],
    "motivo_visita": [
      { "group": "TIPO_VISITA", "label": "Tipo de visita", "items": [
        { "code": 1, "label": "Cadastramento/Atualização" },
        { "code": 29, "label": "Visita periódica" } ] },
      { "group": "BUSCA_ATIVA", "label": "Busca ativa", "items": [
        { "code": 2, "label": "Consulta" },
        { "code": 3, "label": "Exame" },
        { "code": 4, "label": "Vacina" },
        { "code": 30, "label": "Condicionalidades do Bolsa Família" } ] },
      { "group": "ACOMPANHAMENTO", "label": "Acompanhamento", "items": [
        { "code": 5, "label": "Gestante" },
        { "code": 6, "label": "Puérpera" },
        { "code": 7, "label": "Recém-nascido" },
        { "code": 8, "label": "Criança" },
        { "code": 9, "label": "Pessoa com desnutrição" },
        { "code": 10, "label": "Pessoa em reabilitação ou com deficiência" },
        { "code": 11, "label": "Pessoa com hipertensão" },
        { "code": 12, "label": "Pessoa com diabetes" },
        { "code": 13, "label": "Pessoa com asma" },
        { "code": 14, "label": "Pessoa com DPOC/enfisema" },
        { "code": 15, "label": "Pessoa com câncer" },
        { "code": 16, "label": "Pessoa com outras doenças crônicas" },
        { "code": 17, "label": "Pessoa com hanseníase" },
        { "code": 18, "label": "Pessoa com tuberculose" },
        { "code": 32, "label": "Sintomáticos respiratórios" },
        { "code": 33, "label": "Tabagista" },
        { "code": 19, "label": "Domiciliados/acamados" },
        { "code": 20, "label": "Condições de vulnerabilidade social" },
        { "code": 21, "label": "Condicionalidades do Bolsa Família" },
        { "code": 22, "label": "Saúde mental" },
        { "code": 23, "label": "Usuário de álcool" },
        { "code": 24, "label": "Usuário de outras drogas" },
        { "code": 38, "label": "Pessoa idosa" } ] },
      { "group": "CONTROLE_AMBIENTAL_E_VETORIAL", "label": "Controle ambiental e vetorial", "items": [
        { "code": 34, "label": "Ação educativa" },
        { "code": 35, "label": "Imóvel com foco" },
        { "code": 36, "label": "Ação mecânica" },
        { "code": 37, "label": "Tratamento focal" } ] },
      { "group": "OUTROS", "label": "Outros", "items": [
        { "code": 25, "label": "Egresso de internação" },
        { "code": 27, "label": "Convite para atividades coletivas/campanha de saúde" },
        { "code": 31, "label": "Orientação/prevenção" },
        { "code": 28, "label": "Outros" } ] }
    ],
    "tipo_de_imovel": [
      { "code": "1", "label": "Domicílio" },
      { "code": "2", "label": "Comércio" },
      { "code": "3", "label": "Terreno baldio" },
      { "code": "4", "label": "Ponto estratégico" },
      { "code": "5", "label": "Escola" },
      { "code": "6", "label": "Creche" },
      { "code": "7", "label": "Abrigo" },
      { "code": "8", "label": "Instituição de longa permanência para idosos" },
      { "code": "9", "label": "Unidade prisional" },
      { "code": "10", "label": "Unidade de medida socioeducativa" },
      { "code": "11", "label": "Delegacia" },
      { "code": "12", "label": "Estabelecimento religioso" },
      { "code": "13", "label": "CASAI" },
      { "code": "99", "label": "Outros" }
    ],
    "tipo_glicemia": [
      { "code": 0, "label": "Jejum" },
      { "code": 1, "label": "Pós-prandial" },
      { "code": 2, "label": "Pré-prandial" },
      { "code": 3, "label": "Não especificado" }
    ],
    "rules": {
      "shift_from_local_time": [
        { "shift": 1, "before": "12:00" },
        { "shift": 2, "before": "18:00" },
        { "shift": 3, "before": "24:00" }
      ],
      "registrar_cbos": ["515105", "515310", "515140", "515130", "515125", "322255", "515120"],
      "out_of_area_cbos": ["515105"],
      "non_residential_property_types": ["2", "3", "4", "5", "6", "12", "13"],
      "non_residential_without_person": true,
      "non_residential_allowed_reasons": [1, 27, 28, 31, 34, 35, 36, 37],
      "outcomes_without_reasons_and_measurements": [2, 3],
      "reasons_forbidden_for_sex": { "male": [5, 6] },
      "measurements": {
        "weight_kg": { "min": 0.5, "max": 500, "decimals": 3, "ledi": "pesoAcompanhamentoNutricional" },
        "height_cm": { "min": 20, "max": 250, "decimals": 1, "ledi": "alturaAcompanhamentoNutricional" },
        "systolic": { "min": 0, "max": 999, "decimals": 0, "requires": "diastolic", "ledi": "pressaoSistolica" },
        "diastolic": { "min": 0, "max": 999, "decimals": 0, "requires": "systolic", "ledi": "pressaoDiastolica" },
        "temperature_c": { "min": 20, "max": 45, "decimals": 1, "ledi": "temperatura" },
        "glucose_mg_dl": { "min": 0, "max": 800, "decimals": 0, "requires": "glucose_type", "ledi": "glicemia" },
        "glucose_type": { "requires": "glucose_mg_dl", "ledi": "tipoGlicemia" }
      },
      "note_max_length": 500,
      "withdrawal_reason_length": { "min": 10, "max": 500 },
      "visits_per_sheet": 99
    }
  },
  "collective_activity": {
    "tipo_atividade_coletiva": [
      { "code": 1, "label": "Reunião de equipe" },
      { "code": 2, "label": "Reunião com outras equipes de saúde" },
      { "code": 3, "label": "Reunião intersetorial/conselho local de saúde/controle social" },
      { "code": 4, "label": "Educação em saúde" },
      { "code": 5, "label": "Atendimento em grupo" },
      { "code": 6, "label": "Avaliação/procedimento coletivo" },
      { "code": 7, "label": "Mobilização social" }
    ],
    "temas_para_reuniao": [
      { "code": 1, "label": "Questões administrativas/funcionamento" },
      { "code": 2, "label": "Processo de trabalho" },
      { "code": 3, "label": "Diagnóstico do território/monitoramento do território" },
      { "code": 4, "label": "Planejamento/monitoramento das ações da equipe" },
      { "code": 5, "label": "Discussão de caso/projeto terapêutico singular" },
      { "code": 6, "label": "Educação permanente" },
      { "code": 7, "label": "Outros" },
      { "code": 8, "label": "Vigilância em saúde" }
    ],
    "temas_para_saude": [
      { "code": 29, "label": "Ações de combate ao Aedes aegypti" },
      { "code": 19, "label": "Agravos negligenciados" },
      { "code": 1, "label": "Alimentação saudável" },
      { "code": 4, "label": "Autocuidado de pessoas com doenças crônicas" },
      { "code": 5, "label": "Cidadania e direitos humanos" },
      { "code": 7, "label": "Prevenção ao uso de álcool, tabaco e outras drogas" },
      { "code": 8, "label": "Envelhecimento/climatério/andropausa" },
      { "code": 10, "label": "Plantas medicinais/fitoterapia" },
      { "code": 13, "label": "Prevenção da violência e promoção da cultura da paz" },
      { "code": 14, "label": "Saúde ambiental" },
      { "code": 15, "label": "Saúde bucal" },
      { "code": 6, "label": "Saúde do trabalhador" },
      { "code": 16, "label": "Saúde mental" },
      { "code": 17, "label": "Saúde sexual e reprodutiva" },
      { "code": 31, "label": "Amamentação" },
      { "code": 32, "label": "Alimentação complementar" },
      { "code": 21, "label": "Outros" }
    ],
    "praticas_em_saude": [
      { "code": 20, "label": "Antropometria" },
      { "code": 2, "label": "Aplicação tópica de flúor" },
      { "code": 23, "label": "Desenvolvimento da linguagem" },
      { "code": 9, "label": "Escovação dental supervisionada" },
      { "code": 11, "label": "Práticas corporais e atividade física" },
      { "code": 25, "label": "PNCT sessão 1" },
      { "code": 26, "label": "PNCT sessão 2" },
      { "code": 27, "label": "PNCT sessão 3" },
      { "code": 28, "label": "PNCT sessão 4" },
      { "code": 34, "label": "PNCT sessão de manutenção" },
      { "code": 22, "label": "Saúde auditiva" },
      { "code": 3, "label": "Saúde ocular" },
      { "code": 24, "label": "Verificação da situação vacinal" },
      { "code": 12, "label": "Outras" },
      { "code": 30, "label": "Outro procedimento coletivo" },
      { "code": 33, "label": "Entrega de kit de saúde bucal" }
    ],
    "publico_alvo": [
      { "code": 1, "label": "Comunidade em geral" },
      { "code": 2, "label": "Criança de 0 a 3 anos" },
      { "code": 3, "label": "Criança de 4 a 5 anos" },
      { "code": 4, "label": "Criança de 6 a 11 anos" },
      { "code": 5, "label": "Adolescente" },
      { "code": 6, "label": "Mulher" },
      { "code": 7, "label": "Gestante" },
      { "code": 8, "label": "Homem" },
      { "code": 9, "label": "Familiares" },
      { "code": 10, "label": "Pessoa idosa" },
      { "code": 12, "label": "Pessoas com doenças crônicas" },
      { "code": 13, "label": "Usuário de tabaco" },
      { "code": 14, "label": "Usuário de álcool" },
      { "code": 15, "label": "Usuário de outras drogas" },
      { "code": 16, "label": "Pessoas com sofrimento ou transtorno mental" },
      { "code": 17, "label": "Profissional de educação" },
      { "code": 18, "label": "Outros", "pending": "confirmar na prova técnica do api (Q3)" },
      { "code": 19, "label": "Adolescente em medida socioeducativa (meio aberto)", "pending": "rótulo a conferir no dicionario-fac.html (Q3)" },
      { "code": 20, "label": "Adolescente em medida socioeducativa (meio fechado)", "pending": "rótulo a conferir no dicionario-fac.html (Q3)" }
    ],
    "rules": {
      "by_activity_type": {
        "1": { "meeting_topics": "required", "target_audience": "forbidden", "health_topics": "with_meeting_topic_6", "health_practices": "forbidden", "participants": "optional" },
        "2": { "meeting_topics": "required", "target_audience": "forbidden", "health_topics": "with_meeting_topic_6", "health_practices": "forbidden", "participants": "optional" },
        "3": { "meeting_topics": "required", "target_audience": "forbidden", "health_topics": "with_meeting_topic_6", "health_practices": "forbidden", "participants": "optional" },
        "4": { "meeting_topics": "forbidden", "target_audience": "required", "health_topics": "required", "health_practices": "forbidden", "participants": "optional" },
        "5": { "meeting_topics": "forbidden", "target_audience": "required", "health_topics": "required", "health_practices": "optional", "participants": "required" },
        "6": { "meeting_topics": "forbidden", "target_audience": "required", "health_topics": "optional", "health_practices": "required", "participants": "required" },
        "7": { "meeting_topics": "forbidden", "target_audience": "required", "health_topics": "required", "health_practices": "forbidden", "participants": "optional" }
      },
      "values": {
        "required": "obrigatório",
        "forbidden": "proibido",
        "optional": "opcional",
        "with_meeting_topic_6": "proibido, salvo com o tema de reunião 6 (Educação permanente)"
      },
      "practices_only_in_activity_type": { "6": [2, 9, 33] },
      "procedure_code_with_practice": 30,
      "smoking_with_practices": [25, 26, 27, 28],
      "pse_only_health_topics": [18],
      "participants_count": { "min": 1, "max": 999 },
      "altered_count_equals_altered_participants_when_listed": true,
      "participants_required_identifier_types": [5, 6],
      "place_text_max_length": 250,
      "place_inep_allowed": false,
      "mistaken_reason_length": { "min": 10, "max": 500 },
      "registrar_cbos_table": "Tabela 5 do LEDI (config/ledi/registration_cbos.yml do api, chave collective_activity)"
    }
  },
  "patient": {
    "sexo": [
      { "code": "female", "label": "Feminino" },
      { "code": "male", "label": "Masculino" }
    ],
    "justificativa_nao_possui_cpf": {
      "codes": ["2", "3", "4", "5", "6", "7", "8", "9", "10", "11", "12", "13"],
      "labels": "contrato do 24a §15 (config/ledi/registration_dictionaries.yml do api)"
    }
  },
  "errors": {
    "http": ["invalid_token", "device_revoked", "feature_disabled", "invalid_request", "invalid_cursor", "invalid_limit", "too_many_requests", "payload_too_large"],
    "item": [
      "invalid_request", "feature_disabled", "client_uuid_conflict", "not_found", "cbo_not_allowed", "out_of_scope",
      "invalid_home_visit", "visit_date_out_of_range", "non_residential_with_person", "reasons_forbidden", "reason_not_allowed",
      "measurements_forbidden", "patient_in_other_household", "family_without_responsible", "day_closed", "visit_changed",
      "invalid_collective_activity", "activity_date_out_of_range", "duplicate_participant", "professional_without_cns",
      "duplicate_suspected", "cpf_taken", "cns_taken",
      "invalid_household", "invalid_family", "invalid_registration", "cpf_or_reason_required", "already_registered",
      "identity_locked", "address_from_household", "invalid_ethnicity", "citizen_not_verified", "invalid_transition"
    ]
  }
}
```

- [ ] **Step 3: Rode a conferência e veja passar**

```bash
C=/Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/contracts-mod24b
python3 -m json.tool $C/field-sync/domains-v1.json > /dev/null && echo "json ok"
python3 $C/field-sync/tools/check_domains.py; echo "exit $?"
python3 $C/field-sync/tools/check_domains.py --release; echo "exit $?"
```

Expected: `json ok`; `defs-iguais`, `dominios-iguais`, `erros-conhecidos`, `prova: pendente`, `exit 0`; com `--release`, as mesmas linhas e `exit 1` (a tag fica barrada até a Task 4).

- [ ] **Step 4: README do domínio**

Crie `field-sync/README.md`:

````markdown
# field-sync — sincronização do app de campo (v1)

Contrato entre o `api` e o app de campo offline (módulo 24c), entregue pelo
módulo 24b (ADRs 0041 e 0052). O app não é fonte de verdade: ele envia o que
registrou e baixa o escopo do aparelho. Base `/field/v1` no host da cidade.

| Arquivo | `title` | O que é |
|---|---|---|
| `session-response-v1.json` | `rotasaude.field_sync.session_response.v1` | `GET /field/v1/session` |
| `upload-request-v1.json` | `rotasaude.field_sync.upload_request.v1` | `POST /field/v1/uploads` (pedido) |
| `upload-response-v1.json` | `rotasaude.field_sync.upload_response.v1` | `POST /field/v1/uploads` (resposta 200) |
| `changes-response-v1.json` | `rotasaude.field_sync.changes_response.v1` | `GET /field/v1/changes` |
| `domains-v1.json` | `rotasaude.field_sync.domains.v1` | domínios e regras cruzadas da visita e da atividade, em dado |

## Regras

- **Aparelho.** Cadastrado no dashboard com segundo fator; o token
  (`rsf_` + 32 bytes em base64url) aparece uma vez, no QR
  `rotasaude-field:v1:<base64url de { host, token }>`. Vai só em
  `Authorization: Bearer rsf_…`, nunca na URL. Ausente ou desconhecido →
  `401 invalid_token`; revogado (pelo dono, pelo administrador, fim de vínculo
  ou usuário desativado) → `401 device_revoked`. 120 requisições por minuto;
  só JSON (415); corpo até 2 MB (413). Interruptor `home_visits` desligado →
  `403 feature_disabled`.
- **Escopo.** O ACS baixa as microáreas de que é responsável; os outros CBOs
  da Tabela 7, as que o enfermeiro atribui (`scope` `responsible` ou
  `assigned`). Enfermeiro e médico não têm aparelho.
- **Envio.** De 1 a 50 itens, processados em ordem, cada um na sua transação:
  um item recusado não derruba o lote. Idempotente pelo `client_uuid` do item:
  repetido → `duplicate` com o mesmo `id`; o mesmo `client_uuid` com outro
  `type` → `rejected` `client_uuid_conflict`. Só itens aceitos entram no livro
  de idempotência: o reenvio de um `rejected` ou `stale_version` é processado
  de novo. Um item pode citar o `client_uuid` de um `patient` aceito antes
  (`patient_client_uuid`), no mesmo lote ou em outro. O domicílio e o
  cadastro com `version` vencida voltam `stale_version` com a versão atual.
- **Download.** Por cursor opaco (`next_cursor`, sempre presente: o app o guarda
  para a próxima vez); páginas de até 500 itens; sem cursor = carga completa.
  `reset: true` = o escopo mudou: apague o que foi baixado (nunca o que falta
  enviar) e aplique as páginas. `removals` vêm das lápides (`moved`, `erased`)
  e da troca de escopo (`out_of_scope`). "Quem visitar" vem só na última
  página (`has_more: false`), completo; nas outras, `null`. Toda página com
  pessoa grava trilha no api.
- **Formas.** Fechadas em todo nível, salvo as seções de dicionário do 24a
  (`living_conditions`, `identification`, `sociodemographic`,
  `health_conditions`, `street_situation`), cujas chaves e domínios são os do
  contrato do 24a §15. Instantes com fuso explícito (`Z` ou `±hh:mm`; o api
  responde no fuso da cidade); datas `AAAA-MM-DD`; CPF e CNS só dígitos;
  `property_type` e `no_cpf_reason` em texto (24a); códigos LEDI inteiros.
  O turno da visita nunca vem do app: o api deriva da hora. CPF nunca desce
  (só `has_cpf`; CNS só mascarado).
- **O esquema confere forma, não regra.** As relações entre campos (desfecho ×
  motivos, imóvel × pessoa, tipo de atividade × temas, quem pode registrar)
  estão em `domains-v1.json` (`rules`) para o app guiar o formulário, e quem
  decide é o api (`rejected` com o código e o `path`). Código de erro novo do
  api é MINOR: o app mostra uma frase genérica.

## Exemplos e verificação

`examples/` tem 56 exemplos (20 válidos, 36 inválidos, um defeito por arquivo),
gerados por `tools/make_examples.py` e listados em `examples/manifest.json`
(`{ suites: [ { schema, cases: [ { file, expect } ] } ] }`; `expect` é
`valid` ou o erro `<data_pointer> <type>` do `json_schemer`). A partir da raiz
do monorepo, com o compose de pé:

```bash
contracts/field-sync/tools/check.sh                           # todas as suítes (56/56)
contracts/field-sync/tools/check.sh upload-request-v1.json    # uma suíte
python3 contracts/field-sync/tools/check_domains.py           # $defs iguais, enum = domains, erros conhecidos
python3 contracts/field-sync/tools/check_domains.py --release # idem, e falha sem a prova técnica (proof)
```

## Consumo

O `api` copia os quatro esquemas para `config/field_sync/` e prende o SHA-256 de
cada um num spec (como `config/lab/lab-report-v1.json`); as respostas reais de
`/field/v1/*` são validadas contra essas cópias. Mudou um esquema? Atualize os
dois lugares no mesmo ciclo, com entrada no CHANGELOG. O app de campo (24c)
consome os esquemas e `domains-v1.json` pela tag.
````

- [ ] **Step 5: CHANGELOG do domínio**

Crie `field-sync/CHANGELOG.md`:

```markdown
# Changelog — field-sync

## field-sync-v1.0.0 — (não publicada até a tag) — MAJOR

Primeira versão do domínio (ADR 0052, módulo 24b): a sincronização do app de
campo offline (24c) com o api.

- `session-response-v1.json`: usuário, profissional (CBO, `has_cns`), equipes
  com as microáreas do escopo do aparelho (`responsible` ou `assigned`),
  interruptores e `min_date`.
- `upload-request-v1.json`: lote de 1 a 50 itens `{ client_uuid, type, body }`
  — `home_visit`, `home_visit_withdrawal`, `collective_activity`, `patient`
  (registro rápido), `household` e `individual_registration` (corpos do 24a,
  com a versão esperada na atualização); `patient_client_uuid` cita um
  paciente criado offline.
- `upload-response-v1.json`: um resultado por item — `accepted`, `duplicate`,
  `rejected` (código, `path`, `candidates` em `duplicate_suspected`) e
  `stale_version` (versão atual do domicílio ou do cadastro).
- `changes-response-v1.json`: página do download incremental com
  microáreas, domicílios (famílias e condições de moradia), cadastros,
  remoções, `reset`, "quem visitar" na última página e cursor opaco.
- `domains-v1.json`: domínios da ficha de visita (tipo 8) e de atividade
  coletiva (tipo 6) e as regras cruzadas, em dado.
- Fechados em todo nível, salvo as seções de dicionário do 24a; instantes com
  fuso explícito; CPF nunca desce.
- `examples/`: 56 exemplos (20 válidos, 36 inválidos) e `manifest.json`;
  `tools/`: gerador e verificadores.
```

- [ ] **Step 6: README da raiz**

Em `README.md` (worktree `$C`):

(a) Na abertura de "## Domínios", troque `Nove domínios` por `Dez domínios`.

(b) (Q28) Na mesma tabela, corrija as versões vigentes: `lab-v1.0.0` → `lab-v1.1.0` e `billing-v1.0.0` → `billing-v1.1.0` (confira antes com `$G -C $C tag -l 'lab-v*' 'billing-v*'`); na lista de versionamento, acrescente o prefixo `billing-vX.Y.Z` ao lado dos outros. Vai no mesmo commit dos itens (c) e (d).

(c) Na tabela "Domínios", logo depois da linha de `billing/`, acrescente:

```markdown
| [`field-sync/`](field-sync/README.md) | Sincronização do app de campo v1: sessão, envio em lote idempotente e download incremental por microárea (ADRs 0041 e 0052) | `field-sync-v1.0.0` | Materializado |
```

(d) Logo antes de `### Fora deste repo, por enquanto`, acrescente:

```markdown
### field-sync

`session-response-v1.json`, `upload-request-v1.json`,
`upload-response-v1.json` e `changes-response-v1.json` (JSON Schema 2020-12)
descrevem a sincronização do app de campo offline (módulo 24c) com o `api`,
entregue no módulo 24b (ADR 0052): a sessão do aparelho, o envio em lote
idempotente pelo `client_uuid` de cada item e o download incremental pelas
microáreas do aparelho. `domains-v1.json` traz os domínios e as regras
cruzadas da visita domiciliar e da atividade coletiva em dado. O `api` usa
**cópias** em `config/field_sync/`, presas por SHA-256. Exemplos e
verificação em [`field-sync/README.md`](field-sync/README.md).
```

(e) Em "## Versionamento (ADR 0015)", na lista de prefixos, troque `` `lab-vX.Y.Z`, `` por `` `lab-vX.Y.Z`, `field-sync-vX.Y.Z`, `` (e acrescente `` `billing-vX.Y.Z`, `` se ainda faltar, Q28).

Confira: `grep -n 'field-sync\|Dez domínios' $C/README.md`. Expected: a abertura, a linha da tabela, o parágrafo (título e link) e o prefixo.

- [ ] **Step 7: Commit**

```bash
C=/Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/contracts-mod24b; G=/opt/homebrew/bin/git
$G -C $C add field-sync/domains-v1.json field-sync/tools/check_domains.py field-sync/README.md field-sync/CHANGELOG.md README.md
$G -C $C status --short
$G -C $C commit -m "feat: add field sync domains, cross-file checks and docs

domains-v1.json mirrors the home visit and collective activity domains and
cross-field rules as data so the field app builds its forms without
copying rules by hand; proof stays null until the api technical proof.
check_domains.py pins repeated \$defs, schema enums against the domains and
known error codes, and blocks the release without the proof.

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

Expected: `status --short` com 4 arquivos `A` e `README.md` `M`, nada mais.

**Relato ao coordenador:** "contracts-mod24b Task 2: domains-v1.json (proof: null), check_domains e READMEs commitados (<sha>); defs-iguais, dominios-iguais, erros-conhecidos; --release barrado até a prova."

---

### Task 3: Entrega das cópias ao api e parada

**Files:** nenhum.

**Interfaces:**
- Consumes: a branch com os commits das Tasks 1 e 2.
- Produces: o hash da branch e o SHA-256 dos quatro esquemas para o plano do api copiar em `config/field_sync/` (contrato §9.4).

- [ ] **Step 1: Verificação e SHA-256**

```bash
C=/Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/contracts-mod24b; G=/opt/homebrew/bin/git
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude
$C/field-sync/tools/check.sh | tail -1
python3 $C/field-sync/tools/check_domains.py | tail -1
$G -C $C log --oneline origin/main..HEAD
$G -C $C diff --stat origin/main..HEAD -- protocols session clinical exams lab billing events types design-tokens | tail -1
(cd $C/field-sync && shasum -a 256 session-response-v1.json upload-request-v1.json upload-response-v1.json changes-response-v1.json domains-v1.json)
```

Expected: `56/56 casos`; `prova: pendente`; dois commits; nenhuma linha no `diff --stat` (nada fora de `field-sync/` e do `README.md`); cinco SHA-256.

- [ ] **Step 2: Pare.** Não faça merge nem push. Mande ao coordenador o hash do `HEAD`, o caminho `$C/field-sync/` e os cinco SHA-256: o plano do api copia os esquemas daqui (worktree local, sem esperar a tag), e a Task 4 deste plano só começa quando a prova técnica do api (Task 1 dele) publicar o acréscimo do §10–§11 no contrato.

**Relato ao coordenador:** "contracts-mod24b Task 3: branch em <sha>, 56/56; SHA-256 dos esquemas: <lista>; aguardando a prova técnica do api para conferir domains-v1.json."

---

### Task 4: Domínios conferidos com a prova técnica do api

**Pré-requisito:** a Task 1 do plano do api concluída e o acréscimo publicado no contrato do 24b (§10 e §11: `tipoGlicemia`, "Outros" do público-alvo, cada regra cruzada do §4, `smoking` na prática 34), e as decisões Q-n do coordenador sobre K-1 a K-16 na §16 do contrato.

**Files (worktree `$C`):**
- Modify: `field-sync/domains-v1.json`, `field-sync/CHANGELOG.md`
- Modify (só se um código, uma faixa ou uma Q-n mudar a forma): `field-sync/upload-request-v1.json`, `field-sync/tools/make_examples.py`, `field-sync/examples/`, `field-sync/README.md`

**Interfaces:**
- Consumes: o acréscimo do §10–§11 (com o commit do api da prova) e a §16 do contrato.
- Produces: `domains-v1.json` com `proof = { "api_commit": "<sha>", "date": "AAAA-MM-DD", "contract": "docs <sha> §10–§11" }`; os SHA-256 novos para o api.

- [ ] **Step 1: Diferenças entre o provisório e a prova**

Leia o acréscimo do §10–§11 e anote, para cada domínio de `domains-v1.json` e cada linha de `rules.by_activity_type`, o que mudou. No mínimo: `tipo_glicemia` (códigos e rótulos), o código do "Outros" de `publico_alvo` (18 ou 74), `health_topics` nos tipos 1–3 com o tema 6 e no tipo 6, `smoking_with_practices` (entra o 34?), e os limites das medidas.

- [ ] **Step 2: Atualize o dado e o esquema (o teste vem antes)**

1. Em `domains-v1.json`, aplique as diferenças do Step 1; tire a chave `pending` do "Outros"; preencha `proof`.
2. Rode `python3 $C/field-sync/tools/check_domains.py --release`. Expected: `DOMINIOS DIFERENTES: <os domínios que mudaram>` (exit 1) se algum código mudou; se nada mudou, `dominios-iguais` e `prova: <sha>` (exit 0) e pule para o item 5.
3. Em `upload-request-v1.json`, ajuste o `enum` de cada domínio citado (ex.: `target_audience.items.enum` com 74 no lugar de 18; `visit_measurements.properties.glucose_type.enum`).
4. Se um exemplo usa um código que saiu (os válidos usam `reasons` 11, 12, 34, 37; `target_audience` 12; `health_topics` 4; `health_practices` 20; `glucose_type` 2; `property_type` "1" e "4"), troque o código em `make_examples.py` e regere (`python3 $C/field-sync/tools/make_examples.py`).
5. Para cada Q-n que mude a forma — a Q27 aprovou a K-5: `household_client_uuid` em `home_visit_body` e `individual_registration_body`, e `patient_client_uuid` em `individual_registration_body` (um só entre `household_id` e `household_client_uuid`; idem paciente) — acrescente a propriedade, um exemplo válido e um inválido no gerador, e a regra no README.

- [ ] **Step 3: Rode tudo e veja passar**

```bash
C=/Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/contracts-mod24b
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude
$C/field-sync/tools/check.sh | tail -1
python3 $C/field-sync/tools/check_domains.py --release; echo "exit $?"
```

Expected: `N/N casos` (56, ou mais com as Q-n); `defs-iguais`, `dominios-iguais`, `erros-conhecidos`, `prova: <sha> (<data>)`, `exit 0`.

- [ ] **Step 4: CHANGELOG.** No título, troque `(não publicada até a tag)` pela data da Task 5 só na Task 5; aqui, acrescente ao fim da entrada `field-sync-v1.0.0` a linha `- Domínios conferidos com a prova técnica do api (<sha>, contrato do 24b §10–§11): <resumo das mudanças, ou "sem mudança">.`

- [ ] **Step 5: Commit**

```bash
C=/Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/contracts-mod24b; G=/opt/homebrew/bin/git
$G -C $C add field-sync/domains-v1.json field-sync/CHANGELOG.md
# só se mudaram:
# $G -C $C add field-sync/upload-request-v1.json field-sync/tools/make_examples.py field-sync/examples field-sync/README.md
$G -C $C status --short
$G -C $C commit -m "feat: confirm field sync domains against the LEDI technical proof

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
(cd $C/field-sync && shasum -a 256 session-response-v1.json upload-request-v1.json upload-response-v1.json changes-response-v1.json domains-v1.json)
```

Expected: `status` só com os arquivos adicionados; cinco SHA-256. Mande os que mudaram ao coordenador (o api recopia e atualiza o spec do SHA-256 no mesmo ciclo).

**Relato ao coordenador:** "contracts-mod24b Task 4: domains-v1.json conferido com a prova do api (<sha api>), --release verde; SHA-256 novos: <lista ou 'sem mudança nos esquemas'>."

---

### Task 5: Entrega — verificação final, merge, tag e push (com autorização)

**Files:** `field-sync/CHANGELOG.md` (data da tag). Só leitura além disso; merge, tag e push com autorização.

**Interfaces:**
- Consumes: a branch com as Tasks 1, 2 e 4; o api com as cópias presas aos SHA-256 finais.
- Produces: a tag `field-sync-v1.0.0` em `origin` (contrato §15).

- [ ] **Step 1: Verificação final**

```bash
C=/Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/contracts-mod24b; G=/opt/homebrew/bin/git
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude
$C/field-sync/tools/check.sh | tail -1
python3 $C/field-sync/tools/check_domains.py --release; echo "exit $?"
$G -C $C log --oneline origin/main..HEAD
$G -C $C diff --stat origin/main..HEAD -- protocols session clinical exams lab billing events types design-tokens | tail -1
(cd $C/field-sync && shasum -a 256 session-response-v1.json upload-request-v1.json upload-response-v1.json changes-response-v1.json)
```

Expected: `N/N casos`; `exit 0` com `prova: <sha>`; três commits (ou mais, com Q-n); nenhuma linha fora de `field-sync/`; os SHA-256 **iguais** aos que o spec do api prende (confira com a sessão do api pelo coordenador). Diferente: pare.

- [ ] **Step 2: Data no CHANGELOG e commit**

Em `field-sync/CHANGELOG.md`, troque `## field-sync-v1.0.0 — (não publicada até a tag) — MAJOR` por `## field-sync-v1.0.0 — <AAAA-MM-DD da tag> — MAJOR`.

```bash
$G -C $C add field-sync/CHANGELOG.md
$G -C $C commit -m "docs: date the field-sync-v1.0.0 changelog entry

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

- [ ] **Step 3: Pare e peça autorização.** Reporte ao usuário: o hash, `N/N` exemplos verdes, o commit do api da prova e as divergências K-1 a K-16 com as decisões Q-n. Pergunte, uma de cada vez: (1) merge em `main` local; (2) tag anotada `field-sync-v1.0.0`; (3) push da `main` e da tag.

- [ ] **Step 4: Com autorização — merge, tag e push**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/contracts fetch origin
$G -C $R/contracts log --oneline main..origin/main
$G -C $R/contracts checkout main
$G -C $R/contracts merge --ff-only feat/mod-24b-home-visits
$G -C $R/contracts tag -a field-sync-v1.0.0 -m "field-sync-v1.0.0 — MAJOR"
$G -C $R/contracts log --oneline origin/main..main
$G -C $R/contracts push origin main
$G -C $R/contracts push origin field-sync-v1.0.0
$G -C $R/contracts worktree remove $R/.worktrees/contracts-mod24b
$G -C $R/contracts branch -d feat/mod-24b-home-visits
```

Expected: `main..origin/main` vazio (senão, pare: rebase da branch sobre a nova `origin/main`, refaça o Step 1); merge fast-forward; `origin/main..main` só com os commits deste plano; push da `main` e da tag aceitos (o aviso de "PR obrigatório" passa por bypass, como nos módulos anteriores). Se as partes do api e do dashboard do F-24.17 estiverem Done, mova o card para Done (opção `98236657`).

**Relato ao coordenador:** "contracts-mod24b Task 5: field-sync-v1.0.0 publicada (main <sha>, tag pushada); worktree removido."

---

## Self-review (feito ao escrever o plano)

1. **Cobertura do contrato §9.4:** `README.md` e `CHANGELOG.md` (Task 2, Steps 4–5); `session-response-v1.json`, `upload-request-v1.json`, `upload-response-v1.json`, `changes-response-v1.json` (Task 1, Steps 3–7); `domains-v1.json` (Task 2, Step 2; conferido na Task 4); `examples/` + `manifest.json` com um válido por tipo de item (sete arquivos de tipo único e um lote misto), um por resultado (`accepted`, `duplicate`, `rejected` — inclusive `duplicate_suspected` —, `stale_version` de domicílio e de cadastro), página com `removals`, página com `reset`, e 36 inválidos (Task 1, Step 1); JSON Schema 2020-12 fechado (salvo K-2); cópia no api presa por SHA-256 (Tasks 3–5); a tag só depois da prova (Task 2, Step 3 `--release`; Task 4; Task 5); linha no README da raiz (Task 2, Step 6). §9.3: sessão, 1–50 itens, seis tipos, `patient_client_uuid`, idempotência e `client_uuid_conflict`, formas dos resultados, `stale_version` com `current`, download com `reset`, `removals` (três motivos), "quem visitar" só na última página, cursor opaco, `limit` até 500 (`maxItems`). §8: forma do item e dos três motivos, sem CID.
2. **Placeholders:** os esquemas, o gerador, os verificadores e o dado estão no texto. Valores lidos na execução: ids do card (Task 0), hashes e SHA-256 (Tasks 3–5), o acréscimo da prova técnica e as Q-n (Task 4, que lista o que conferir e os exemplos afetados). O único marcador textual, `«bloco do Step 5»`, é substituído pelo bloco dado no Step 5 e conferido pelo `json.tool` do Step 8.
3. **Consistência:** nomes dos 56 exemplos iguais no gerador, no manifesto, no Review Focus e nas contagens (5 + 26 + 13 + 12; 20 válidos); `$defs` repetidos iguais por comando; `enum` = `domains-v1.json` por comando; códigos de erro dos exemplos em `errors.item`. Os `expect` seguem o formato que o `clinical/` já usa com o mesmo `json_schemer` (`schema` para propriedade a mais ou `false`, o nome do tipo para `type`, o nome da palavra-chave nos demais). Não rodei a verificação ao escrever o plano (só leitura); o Step 8 da Task 1 tem a regra para um `type` diferente do esperado.
4. **Review Focus:** as cinco linhas têm teste nas Tasks 1 e 2.

## Divergências propostas ao contrato

> **Decididas na §16 do contrato (2026-11-12):** K-1..K-3, K-8..K-10, K-13..K-15 → Q26; K-4, K-5 → Q27; K-6 → Q15 (**rejeitada**: local plano no pedido); K-7 → Q15; K-11, K-12 → Q13; K-16 → Q28 (corrigida no plano, sem card).

- **K-1 — `$id` e nome.** O §9.4 pede `$id` `rotasaude.field_sync.<nome>.v1`. No repo, `$id` é URL (`https://rota-saude/schemas/<domínio>/<arquivo>`) e o nome vai em `title` (`clinical/`, `lab/`, `exams/`); um `$id` que não é URI deixa o `json_schemer` resolver `$ref` contra uma base relativa. Proposta: `$id` URL e `title` = `rotasaude.field_sync.<nome>.v1` (ex.: `rotasaude.field_sync.upload_request.v1`).
- **K-2 — Seções do 24a abertas.** O §9.4 pede esquema fechado, e o §9.3 reusa as formas do 24a. As seções de dicionário (`living_conditions`, `identification`, `sociodemographic`, `health_conditions`, `street_situation`) têm dezenas de chaves que moram em `registration_dictionaries.yml` do api; copiá-las criaria uma segunda fonte. Proposta: tudo fechado, salvo essas cinco seções (`type: object`); `address` e `patient_demographics` ficam fechados (chaves conhecidas).
- **K-3 — Instante com fuso.** O contrato não fixa a forma dos instantes. O api devolve `iso8601` no fuso da cidade (`-03:00`), não em UTC. Proposta: aceitar `Z` ou `±hh:mm`, com segundos; o `occurred_at` enviado pelo app **exige** fuso (sem fuso, o dia da cidade e o turno ficam ambíguos).
- **K-4 — Família fora do envio.** O §9.3 não tem item de família, e família tem rota própria no 24a. Offline, o app não cria família nova; visita `family_responsible` e cadastro com `family_id` só para famílias já baixadas. Proposta: documentar no v1 e acrescentar `family` (corpo do `POST /territory/households/:id/families`) num MINOR quando o 24c pedir; ou incluir já no v1 (exige rota no api).
- **K-5 — Citar domicílio criado offline.** Só `patient_client_uuid` existe. Um domicílio novo enviado no mesmo lote não pode ser citado pela visita nem pelo cadastro, e o cadastro de pessoa criada offline não aceita `patient_client_uuid`. Proposta: `household_client_uuid` em `home_visit` e `individual_registration`, e `patient_client_uuid` em `individual_registration`, com a mesma regra (item aceito antes). Até a Q-n, o esquema segue o contrato; a Task 4 aplica.
- **K-6 — `place` no pedido.** O §7 dá só a forma da resposta (`{ health_unit: { id, name } } | { text }`). Proposta para o pedido: `{ health_unit_id } | { text }`, exatamente uma chave (`maxProperties: 1`).
- **K-7 — Listas da atividade opcionais no pedido.** Ausente = vazia (o tipo decide o que é proibido); na resposta, as quatro listas sempre vêm.
- **K-8 — `error` por padrão, não por `enum`.** Um `enum` fechado faria todo código novo do api quebrar o esquema. Proposta: `^[a-z][a-z0-9_]*$` no esquema e a lista conhecida em `domains-v1.json` (`errors.http`, `errors.item`), conferida contra os exemplos.
- **K-9 — `registration_id` no candidato.** O §5 dá `{ patient_id, display_name, birth_date, has_cpf }`, mas o item `individual_registration` passa pelo 24a, cujo candidato traz `registration_id` (L31). Proposta: `registration_id` opcional (uuid ou `null`).
- **K-10 — `next_cursor` e `who_to_visit`.** O §9.3 não diz se `next_cursor` vem na última página. Proposta: sempre vem (o app guarda para a próxima sincronização); `who_to_visit` é `null` se e só se `has_more: true`.
- **K-11 — Quem escreve `domains-v1.json`.** O §10 diz que a Task 1 do api publica o resultado "nesta seção (e em `domains-v1.json`)". Pela regra das sessões por app, o repo `contracts` é desta sessão. Proposta: o api publica a tabela no contrato; este plano (Task 4) atualiza `domains-v1.json`, preenche `proof` e manda os SHA-256 novos.
- **K-12 — Relações fora do esquema.** O esquema não confere desfecho × motivos, imóvel × pessoa, tipo × temas, um-de-três da pessoa nem CPF × justificativa; ficam em `domains-v1.json` (`rules`) e no api, com os códigos do §13. Exceções no esquema: os pares de medida (`dependentRequired`) e `household_id`/`registration_id` com `version`.
- **K-13 — `scope` da sessão.** O §6 tem `scope` `responsible | assigned | team`; no aparelho só existem `responsible` e `assigned` (enfermeiro e médico não têm aparelho). Proposta: `enum` com os dois.
- **K-14 — Códigos do 24a em texto.** O §1 diz "`property_type` (texto)", e o §10 diz "`tipoDeImovel` ← `property_type` (inteiro)". Proposta: no contrato do app, texto (`"1"`…`"99"`, como o 24a grava e devolve); o inteiro é só na ficha. Idem `no_cpf_reason`.
- **K-15 — Envelope de erro.** A recusa do envelope (`422 invalid_request` com `path`) e os erros HTTP (`401`, `403`, `413`, `415`, `429`) seguem a convenção `{ "error" }` e não têm esquema próprio; os códigos estão em `errors.http`.
- **K-16 — README da raiz desatualizado (achado, fora do escopo).** A tabela da raiz mostra `lab-v1.0.0` e `billing-v1.0.0`, mas as tags `lab-v1.1.0` e `billing-v1.1.0` existem, e o prefixo `billing-vX.Y.Z` falta na lista de versionamento. Este plano não mexe nisso; proposta: card de pendência no board #2 para o dono do contracts.
