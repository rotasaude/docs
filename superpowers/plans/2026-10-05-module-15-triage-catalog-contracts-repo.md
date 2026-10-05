# Módulo 15 — `offer`, `suggestions` e `gte`/`lte` no schema de protocolo (repo contracts) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Publicar no repo `contracts` os acréscimos opcionais do protocolo do módulo 15 — operadores `gte`/`lte` em `$defs/condition`, bloco `offer` e lista `suggestions` — como `protocols-v1.4.0` (MINOR), com exemplos válidos e inválidos versionados, fonte da cópia que o `api` valida (F-15.2, F-15.3, F-15.6; ADR 0027).

**Architecture:** Duas edições em `protocols/schema.json` (uma por tarefa), exemplos de definição em `protocols/examples/` com um `manifest.json` que diz, por arquivo, `valid` ou o erro esperado (`<data_pointer> <type>` do `json_schemer`), entrada no `protocols/CHANGELOG.md`, versão no `README.md`. O repo não tem código executável nem suíte: a prova é validar os exemplos com o mesmo `json_schemer` 2.5 do `api`, dentro do container, recebendo schema e exemplos pela entrada padrão (o container só monta `apps/api`). Tag anotada `protocols-v1.4.0` e push só com autorização.

**Tech Stack:** JSON Schema draft 2020-12; `json_schemer` 2.5.0 (do bundle do `api`, só para a verificação); `python3` do host para empacotar schema + exemplos num JSON.

**Spec:** `docs/.claude/mod15/superpowers/specs/2026-10-05-module-15-triage-catalog-design.md` §4.1–§4.2 e §11; ADR `docs/.claude/mod15/adr/0027.md`. Contratos entre apps (fonte única do formato): `docs/.claude/mod15/superpowers/plans/2026-10-05-module-15-triage-catalog-contracts.md` §1 e §6. **Este plano roda antes do plano do `api`**, que copia o `schema.json` final byte a byte.

## Global Constraints

- SemVer por domínio (ADR 0015): só acréscimo opcional → **MINOR**, `protocols-v1.3.0` → `protocols-v1.4.0`. Nada vira obrigatório, nada é removido: toda definição válida em `v1.3.0` continua válida.
- Formato exato do contrato §1: `offer` (`additionalProperties: false`; `title` string 1..60; `summary` string 1..200; `eligibility` `$ref #/$defs/condition`; `retake_after_days` integer 1..3650); `suggestions` (array, `maxItems: 10`; item `additionalProperties: false`, `required: ["protocol", "when"]`; `protocol` string `^[a-z][a-z0-9-]+$`, o mesmo `pattern` do `name` do protocolo; `when` `$ref #/$defs/condition`); `$defs/condition` ganha `gte` e `lte` com o formato de `gt`/`lt` (`$ref #/$defs/condition_operand`).
- O schema **não** distingue variáveis por lugar (`profile.*`, `outcome.*`, `citizen.*`, ids de passo): isso é do gate do `api` (contrato §1, tabela). Nenhum `pattern` de variável entra aqui.
- CHANGELOG obrigatório: sem entrada em `protocols/CHANGELOG.md`, não há tag.
- Tag **anotada**, mensagem `protocols-v1.4.0 — MINOR` (mesmo formato de `protocols-v1.3.0`), criada no commit que chega à `main`.
- O `api` usa uma **cópia** (`apps/api/config/protocols/schema.json`, hoje idêntica a `contracts/protocols/schema.json` v1.3.0): os dois arquivos ficam byte a byte iguais no mesmo ciclo.
- Tipos TypeScript: **nenhum a atualizar aqui.** `contracts/types/` é scaffold (sem tipos de protocolo) e o editor do dashboard trata a definição como `unknown` (`apps/dashboard/src/lib/editor.ts`); tipos de `offer`/`suggestions` no dashboard são do plano do dashboard.
- Design tokens: **não mudar.** `contracts/design-tokens/` é scaffold (só `README.md`); os tokens reais do visual do wpda estão em `apps/wpda/src/theme/tokens.ts` + `apps/wpda/src/theme/global.css` (hoje `tokens.ts` é idêntico ao do dashboard). O plano do dashboard usa esse caminho para a pré-visualização.
- Worktree `contracts/.claude/mod15`, branch `feat/protocols-offer-suggestions` (`contracts/.gitignore` já ignora `/.claude/`). Todos os comandos rodam a partir da raiz do monorepo `/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, com o compose de pé (`docker compose ps` mostra `api running`).
- Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Commits em inglês, Conventional Commits com tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
- Nada de merge na `main`, tag ou push sem autorização explícita do usuário, uma etapa de cada vez.

## Review Focus

1. **Definição publicada sem `offer`/`suggestions` (o caso de toda versão ativa nas cidades):** continua válida. Teste: exemplo `no-offer.json` (Task 1) e o template real `apps/api/config/city_templates/triage_respiratoria.json` passado como caso extra em toda execução da verificação.
2. **`gte`/`lte` valem em todo lugar que usa `$defs/condition`** — não só em `offer`/`suggestions`, mas também em `priority_when` e nas regras de `decision_table`. Quem espera "só no catálogo" se surpreende; o schema aceita, e o `api` precisa avaliar `gte`/`lte` em `Protocols::Condition` **antes** de trocar a cópia do schema (senão um protocolo passa no schema e a condição cai no "operador desconhecido → falso"). Teste: `condition-gte-lte.json` (Task 1) usa `gte`/`lte` em `priority_when`; a ordem fica registrada no CHANGELOG (Task 2).
3. **Nó de condição com dois operadores** (`{ "gte": [...], "lte": [...] }`, erro comum de quem escreve "entre 18 e 64" à mão): continua recusado por `maxProperties: 1`; a forma certa é `all`. Teste: `invalid-condition-two-operators.json` (Task 1) e `reserved-variables.json` com `all` (Task 2).
4. **`offer: {}`** (o editor grava o bloco antes de preencher): válido, e equivale a não ter `offer`. Teste: `offer-empty.json` (Task 2).
5. **Sugestão com `protocol` que começa por dígito (`"1-teste"`):** nunca casaria com um `name` (que exige letra inicial), então o schema recusa com o mesmo `pattern` do `name` (`^[a-z][a-z0-9-]+$`). Teste: `invalid-suggestion-protocol-leading-digit.json` (Task 2), esperado `/suggestions/0/protocol pattern`.

---

### Task 1: `gte` e `lte` em `$defs/condition`, com exemplos e verificação

**Files (repo `contracts`, worktree `contracts/.claude/mod15`):**
- Create: `protocols/examples/manifest.json`
- Create: `protocols/examples/no-offer.json`
- Create: `protocols/examples/condition-gte-lte.json`
- Create: `protocols/examples/invalid-gte-three-items.json`
- Create: `protocols/examples/invalid-condition-two-operators.json`
- Modify: `protocols/schema.json` (linha 182, `oneOf` de `$defs/condition`)
- Modify: `protocols/README.md`

**Interfaces:**
- Produces: `$defs/condition.oneOf` com ramos `{ "required": ["gte"], … }` e `{ "required": ["lte"], … }`, ambos `{ "$ref": "#/$defs/condition_operand" }` (array `[string, string|number]`). Formato do manifesto: `{ "cases": [ { "file": "<nome>.json", "expect": "valid" | "<data_pointer> <type>" } ] }`, em que `<data_pointer>` é o ponteiro JSON do erro (`(root)` se vazio) e `<type>` é o `type` do erro do `json_schemer` (`maxLength`, `required`, `pattern`, `schema` para propriedade a mais…). A Task 2 acrescenta casos a este manifesto; o plano do `api` pode reaproveitar os exemplos.

- [ ] **Step 1: Crie o worktree**

```bash
/opt/homebrew/bin/git -C contracts fetch origin
/opt/homebrew/bin/git -C contracts log --oneline -1 origin/main
/opt/homebrew/bin/git -C contracts worktree add .claude/mod15 -b feat/protocols-offer-suggestions origin/main
```
Expected: `origin/main` em `4c6418c docs: add session contract (session-v1.0.0)` (ou mais novo — se houver commit novo em `protocols/`, pare e reporte); worktree criado.

- [ ] **Step 2: Escreva os exemplos e o manifesto**

```bash
mkdir -p contracts/.claude/mod15/protocols/examples
cd contracts/.claude/mod15/protocols/examples && python3 - <<'EOF'
import json

def base(**extra):
    d = {
        "name": "saude-do-idoso",
        "version": 1,
        "start_step_id": "quedas",
        "steps": [
            {"id": "quedas", "prompt": "Teve alguma queda nos últimos 12 meses?", "answer_type": "boolean",
             "branches": {"true": "medicamentos", "false": "medicamentos"}, "weights": {"true": 5, "false": 0}},
            {"id": "medicamentos", "prompt": "Quantos medicamentos diferentes você toma por dia?", "answer_type": "integer"},
        ],
        "scoring": {"type": "weighted", "thresholds": {"baixa": 0, "alta": 5}, "priority_map": {"baixa": 9, "alta": 5}},
    }
    d.update(extra)
    return d

files = {
    "no-offer.json": base(),
    "condition-gte-lte.json": base(priority_when=[
        {"when": {"all": [{"gte": ["medicamentos", 5]}, {"lte": ["medicamentos", 30]}]}, "priority": 3}]),
    "invalid-gte-three-items.json": base(priority_when=[
        {"when": {"gte": ["medicamentos", 5, 1]}, "priority": 3}]),
    "invalid-condition-two-operators.json": base(priority_when=[
        {"when": {"gte": ["medicamentos", 5], "lte": ["medicamentos", 30]}, "priority": 3}]),
}
for name, doc in files.items():
    with open(name, "w", encoding="utf-8") as f:
        f.write(json.dumps(doc, indent=2, ensure_ascii=False) + "\n")
EOF
```

Depois crie `contracts/.claude/mod15/protocols/examples/manifest.json`:

```json
{
  "cases": [
    { "file": "no-offer.json", "expect": "valid" },
    { "file": "condition-gte-lte.json", "expect": "valid" },
    { "file": "invalid-gte-three-items.json", "expect": "/priority_when/0/when/gte maxItems" },
    { "file": "invalid-condition-two-operators.json", "expect": "/priority_when/0/when maxProperties" }
  ]
}
```

- [ ] **Step 3: Rode a verificação e veja falhar**

O comando empacota schema + exemplos do manifesto (+ caminhos extras, sempre esperados `valid`) num JSON e valida no `json_schemer` do `api`. A partir da raiz do monorepo:

```bash
python3 -c '
import json, os, sys
schema_path, manifest_path, extras = sys.argv[1], sys.argv[2], sys.argv[3:]
base = os.path.dirname(manifest_path)
cases = [dict(c, doc=json.load(open(os.path.join(base, c["file"])))) for c in json.load(open(manifest_path))["cases"]]
cases += [{"file": p, "expect": "valid", "doc": json.load(open(p))} for p in extras]
print(json.dumps({"schema": json.load(open(schema_path)), "cases": cases}))
' contracts/.claude/mod15/protocols/schema.json contracts/.claude/mod15/protocols/examples/manifest.json apps/api/config/city_templates/triage_respiratoria.json \
| docker compose exec -T -w /rails api bundle exec ruby -rjson -rjson_schemer -e '
input = JSON.parse(STDIN.read)
schemer = JSONSchemer.schema(input["schema"])
passed = input["cases"].count do |c|
  errors = schemer.validate(c["doc"]).map { |e| p = e["data_pointer"]; "#{p.empty? ? "(root)" : p} #{e["type"]}" }.uniq
  ok = c["expect"] == "valid" ? errors.empty? : errors.include?(c["expect"])
  puts "#{ok ? "ok  " : "FAIL"} #{c["file"]} — esperado: #{c["expect"]}; obtido: #{errors.empty? ? "valid" : errors.join(" | ")}"
  ok
end
puts "#{passed}/#{input["cases"].size} casos"
exit(passed == input["cases"].size ? 0 : 1)'
```

Expected (exit 1):
```
ok   no-offer.json — esperado: valid; obtido: valid
FAIL condition-gte-lte.json — esperado: valid; obtido: /priority_when/0/when/all string | …
FAIL invalid-gte-three-items.json — esperado: /priority_when/0/when/gte maxItems; obtido: /priority_when/0/when/gte string | /priority_when/0/when/gte schema | /priority_when/0/when required
ok   invalid-condition-two-operators.json — esperado: /priority_when/0/when maxProperties; obtido: …
ok   apps/api/config/city_templates/triage_respiratoria.json — esperado: valid; obtido: valid
3/5 casos
```
(`gte` ainda não existe: o nó cai no ramo "mapa simples" do `oneOf` e nenhum ramo de condição casa.)

- [ ] **Step 4: Acrescente `gte` e `lte` ao `oneOf` de `$defs/condition`**

Em `contracts/.claude/mod15/protocols/schema.json`, troque

```json
        { "required": ["lt"], "additionalProperties": false, "properties": { "lt": { "$ref": "#/$defs/condition_operand" } } },
```

por

```json
        { "required": ["lt"], "additionalProperties": false, "properties": { "lt": { "$ref": "#/$defs/condition_operand" } } },
        { "required": ["gte"], "additionalProperties": false, "properties": { "gte": { "$ref": "#/$defs/condition_operand" } } },
        { "required": ["lte"], "additionalProperties": false, "properties": { "lte": { "$ref": "#/$defs/condition_operand" } } },
```

- [ ] **Step 5: Rode a verificação e veja passar**

Run: o mesmo comando do Step 3; depois `python3 -m json.tool contracts/.claude/mod15/protocols/schema.json > /dev/null && echo json-ok`
Expected: cinco linhas `ok`, `5/5 casos`, exit 0; e `json-ok`.

- [ ] **Step 6: Documente os exemplos em `protocols/README.md`**

Troque o conteúdo de `contracts/.claude/mod15/protocols/README.md` por:

````markdown
# protocols — JSON Schema da definição

`schema.json` é o contrato único da definição de protocolo (ADR 0009), consumido pelo
motor Ruby (`Protocols::Validator`) e pelo preview TS. Versionado por `protocols-vX.Y.Z`.

## Exemplos

`examples/` guarda definições válidas e inválidas. `examples/manifest.json` diz, por
arquivo, `valid` ou o erro esperado no formato `<data_pointer> <type>` do
`json_schemer` (o mesmo validador do `api`). Exemplo novo entra no manifesto no mesmo
commit da mudança de schema que ele prova.

Para verificar, a partir da raiz do monorepo, com o compose de pé (o container do
`api` só monta `apps/api`, por isso tudo entra pela entrada padrão; caminhos depois do
manifesto são casos extras esperados `valid`):

```bash
python3 -c '
import json, os, sys
schema_path, manifest_path, extras = sys.argv[1], sys.argv[2], sys.argv[3:]
base = os.path.dirname(manifest_path)
cases = [dict(c, doc=json.load(open(os.path.join(base, c["file"])))) for c in json.load(open(manifest_path))["cases"]]
cases += [{"file": p, "expect": "valid", "doc": json.load(open(p))} for p in extras]
print(json.dumps({"schema": json.load(open(schema_path)), "cases": cases}))
' contracts/protocols/schema.json contracts/protocols/examples/manifest.json apps/api/config/city_templates/triage_respiratoria.json \
| docker compose exec -T -w /rails api bundle exec ruby -rjson -rjson_schemer -e '
input = JSON.parse(STDIN.read)
schemer = JSONSchemer.schema(input["schema"])
passed = input["cases"].count do |c|
  errors = schemer.validate(c["doc"]).map { |e| p = e["data_pointer"]; "#{p.empty? ? "(root)" : p} #{e["type"]}" }.uniq
  ok = c["expect"] == "valid" ? errors.empty? : errors.include?(c["expect"])
  puts "#{ok ? "ok  " : "FAIL"} #{c["file"]} — esperado: #{c["expect"]}; obtido: #{errors.empty? ? "valid" : errors.join(" | ")}"
  ok
end
puts "#{passed}/#{input["cases"].size} casos"
exit(passed == input["cases"].size ? 0 : 1)'
```
````

- [ ] **Step 7: Commit**

```bash
/opt/homebrew/bin/git -C contracts/.claude/mod15 add protocols/schema.json protocols/README.md protocols/examples
/opt/homebrew/bin/git -C contracts/.claude/mod15 status --short
/opt/homebrew/bin/git -C contracts/.claude/mod15 commit -m "feat: add gte and lte operators to the protocols condition grammar

Same operand shape as gt and lt. Adds protocols/examples with a manifest of
valid and invalid definitions, checked with the api's json_schemer.

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```
Expected: `status --short` lista só `protocols/schema.json`, `protocols/README.md` e os cinco arquivos de `protocols/examples/`.

### Task 2: `offer` e `suggestions` na raiz, CHANGELOG e README (`protocols-v1.4.0`)

**Files (worktree `contracts/.claude/mod15`):**
- Create: `protocols/examples/offer-full.json`, `offer-empty.json`, `suggestions-gte-lte.json`, `reserved-variables.json`
- Create: `protocols/examples/invalid-offer-title-too-long.json`, `invalid-offer-title-empty.json`, `invalid-offer-summary-too-long.json`, `invalid-offer-retake-zero.json`, `invalid-offer-retake-too-big.json`, `invalid-offer-extra-property.json`, `invalid-eligibility-unknown-operator.json`, `invalid-suggestion-without-when.json`, `invalid-suggestion-protocol-uppercase.json`, `invalid-suggestion-protocol-leading-digit.json`, `invalid-suggestions-too-many.json`
- Modify: `protocols/examples/manifest.json`
- Modify: `protocols/schema.json` (fim de `properties`, depois de `priority_when`)
- Modify: `protocols/CHANGELOG.md`
- Modify: `README.md`

**Interfaces:**
- Consumes: `gte`/`lte` em `$defs/condition` e o manifesto/comando de verificação da Task 1.
- Produces: `properties.offer` e `properties.suggestions` exatamente como no contrato §1; `protocols-v1.4.0` no CHANGELOG e no README. Consumidos pelo plano do `api` (cópia literal em `config/protocols/schema.json`; gate por lugar) e, via `api`, pelo painel "Oferta e sugestões" do dashboard.

- [ ] **Step 1: Escreva os exemplos novos**

A partir da raiz do monorepo:

```bash
cd contracts/.claude/mod15/protocols/examples && python3 - <<'EOF'
import json

def base(**extra):
    d = {
        "name": "saude-do-idoso",
        "version": 1,
        "start_step_id": "quedas",
        "steps": [
            {"id": "quedas", "prompt": "Teve alguma queda nos últimos 12 meses?", "answer_type": "boolean",
             "branches": {"true": "medicamentos", "false": "medicamentos"}, "weights": {"true": 5, "false": 0}},
            {"id": "medicamentos", "prompt": "Quantos medicamentos diferentes você toma por dia?", "answer_type": "integer"},
        ],
        "scoring": {"type": "weighted", "thresholds": {"baixa": 0, "alta": 5}, "priority_map": {"baixa": 9, "alta": 5}},
    }
    d.update(extra)
    return d

files = {
    "offer-full.json": base(offer={
        "title": "Saúde do idoso",
        "summary": "Avaliação anual de quedas, memória e medicamentos.",
        "eligibility": {"gte": ["profile.age", 60]},
        "retake_after_days": 365}),
    "offer-empty.json": base(offer={}),
    "suggestions-gte-lte.json": base(suggestions=[
        {"protocol": "saude-mental-aprofundada",
         "when": {"all": [{"gte": ["outcome.score", 15]}, {"lte": ["outcome.priority", 5]}]}}]),
    "reserved-variables.json": base(
        offer={"eligibility": {"all": [
            {"eq": ["profile.sex", "female"]}, {"gte": ["profile.age", 50]}, {"lte": ["profile.age", 69]}]}},
        suggestions=[{"protocol": "saude-mental-aprofundada", "when": {"any": [
            {"eq": ["outcome.tier", "alta"]}, {"eq": ["profile.sex", "female"]}, {"gte": ["medicamentos", 5]}]}}]),
    "invalid-offer-title-too-long.json": base(offer={"title": "x" * 61}),
    "invalid-offer-title-empty.json": base(offer={"title": ""}),
    "invalid-offer-summary-too-long.json": base(offer={"summary": "x" * 201}),
    "invalid-offer-retake-zero.json": base(offer={"retake_after_days": 0}),
    "invalid-offer-retake-too-big.json": base(offer={"retake_after_days": 3651}),
    "invalid-offer-extra-property.json": base(offer={"title": "Saúde do idoso", "audience": "60+"}),
    "invalid-eligibility-unknown-operator.json": base(offer={"eligibility": {"ge": ["profile.age", 60]}}),
    "invalid-suggestion-without-when.json": base(suggestions=[{"protocol": "saude-mental-aprofundada"}]),
    "invalid-suggestion-protocol-uppercase.json": base(suggestions=[
        {"protocol": "Saude-Mental", "when": {"gte": ["outcome.score", 15]}}]),
    "invalid-suggestion-protocol-leading-digit.json": base(suggestions=[
        {"protocol": "1-teste", "when": {"gte": ["outcome.score", 15]}}]),
    "invalid-suggestions-too-many.json": base(suggestions=[
        {"protocol": "protocolo-%02d" % i, "when": {"gte": ["outcome.score", i]}} for i in range(11)]),
}
for name, doc in files.items():
    with open(name, "w", encoding="utf-8") as f:
        f.write(json.dumps(doc, indent=2, ensure_ascii=False) + "\n")
EOF
```

Depois troque `contracts/.claude/mod15/protocols/examples/manifest.json` inteiro por:

```json
{
  "cases": [
    { "file": "no-offer.json", "expect": "valid" },
    { "file": "condition-gte-lte.json", "expect": "valid" },
    { "file": "invalid-gte-three-items.json", "expect": "/priority_when/0/when/gte maxItems" },
    { "file": "invalid-condition-two-operators.json", "expect": "/priority_when/0/when maxProperties" },
    { "file": "offer-full.json", "expect": "valid" },
    { "file": "offer-empty.json", "expect": "valid" },
    { "file": "suggestions-gte-lte.json", "expect": "valid" },
    { "file": "reserved-variables.json", "expect": "valid" },
    { "file": "invalid-offer-title-too-long.json", "expect": "/offer/title maxLength" },
    { "file": "invalid-offer-title-empty.json", "expect": "/offer/title minLength" },
    { "file": "invalid-offer-summary-too-long.json", "expect": "/offer/summary maxLength" },
    { "file": "invalid-offer-retake-zero.json", "expect": "/offer/retake_after_days minimum" },
    { "file": "invalid-offer-retake-too-big.json", "expect": "/offer/retake_after_days maximum" },
    { "file": "invalid-offer-extra-property.json", "expect": "/offer/audience schema" },
    { "file": "invalid-eligibility-unknown-operator.json", "expect": "/offer/eligibility required" },
    { "file": "invalid-suggestion-without-when.json", "expect": "/suggestions/0 required" },
    { "file": "invalid-suggestion-protocol-uppercase.json", "expect": "/suggestions/0/protocol pattern" },
    { "file": "invalid-suggestion-protocol-leading-digit.json", "expect": "/suggestions/0/protocol pattern" },
    { "file": "invalid-suggestions-too-many.json", "expect": "/suggestions maxItems" }
  ]
}
```

- [ ] **Step 2: Rode a verificação e veja falhar**

Run: o comando de verificação do `protocols/README.md`, apontando para o worktree:

```bash
python3 -c '
import json, os, sys
schema_path, manifest_path, extras = sys.argv[1], sys.argv[2], sys.argv[3:]
base = os.path.dirname(manifest_path)
cases = [dict(c, doc=json.load(open(os.path.join(base, c["file"])))) for c in json.load(open(manifest_path))["cases"]]
cases += [{"file": p, "expect": "valid", "doc": json.load(open(p))} for p in extras]
print(json.dumps({"schema": json.load(open(schema_path)), "cases": cases}))
' contracts/.claude/mod15/protocols/schema.json contracts/.claude/mod15/protocols/examples/manifest.json apps/api/config/city_templates/triage_respiratoria.json \
| docker compose exec -T -w /rails api bundle exec ruby -rjson -rjson_schemer -e '
input = JSON.parse(STDIN.read)
schemer = JSONSchemer.schema(input["schema"])
passed = input["cases"].count do |c|
  errors = schemer.validate(c["doc"]).map { |e| p = e["data_pointer"]; "#{p.empty? ? "(root)" : p} #{e["type"]}" }.uniq
  ok = c["expect"] == "valid" ? errors.empty? : errors.include?(c["expect"])
  puts "#{ok ? "ok  " : "FAIL"} #{c["file"]} — esperado: #{c["expect"]}; obtido: #{errors.empty? ? "valid" : errors.join(" | ")}"
  ok
end
puts "#{passed}/#{input["cases"].size} casos"
exit(passed == input["cases"].size ? 0 : 1)'
```

Expected (exit 1): os quatro casos da Task 1 e o template `ok`; os 15 novos `FAIL` com `obtido: /offer schema` ou `/suggestions schema` (propriedade desconhecida na raiz, `additionalProperties: false`); `5/20 casos`.

- [ ] **Step 3: Acrescente `offer` e `suggestions` a `properties`**

Em `contracts/.claude/mod15/protocols/schema.json`, troque o fim de `priority_when` e o começo de `$defs`:

```json
          "priority": { "type": "integer", "minimum": 1, "maximum": 9 }
        }
      }
    }
  },
  "$defs": {
```

por

```json
          "priority": { "type": "integer", "minimum": 1, "maximum": 9 }
        }
      }
    },
    "offer": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "title":             { "type": "string", "minLength": 1, "maxLength": 60 },
        "summary":           { "type": "string", "minLength": 1, "maxLength": 200 },
        "eligibility":       { "$ref": "#/$defs/condition" },
        "retake_after_days": { "type": "integer", "minimum": 1, "maximum": 3650 }
      }
    },
    "suggestions": {
      "type": "array",
      "maxItems": 10,
      "items": {
        "type": "object",
        "additionalProperties": false,
        "required": ["protocol", "when"],
        "properties": {
          "protocol": { "type": "string", "pattern": "^[a-z][a-z0-9-]+$" },
          "when":     { "$ref": "#/$defs/condition" }
        }
      }
    }
  },
  "$defs": {
```

(O trecho "antes" é único: é o único `"priority"` seguido de três fechamentos e de `"$defs"`.)

- [ ] **Step 4: Rode a verificação e veja passar**

Run: o mesmo comando do Step 2; depois `python3 -m json.tool contracts/.claude/mod15/protocols/schema.json > /dev/null && echo json-ok`
Expected: 20 linhas `ok`, `20/20 casos`, exit 0; e `json-ok`.

- [ ] **Step 5: CHANGELOG**

Em `contracts/.claude/mod15/protocols/CHANGELOG.md`, logo abaixo de `# Changelog — protocols` (antes da entrada `protocols-v1.3.0`), insira:

```markdown

## protocols-v1.4.0 — 2026-10-05 — MINOR
- `$defs/condition` ganha `gte` e `lte`, com o mesmo operando de `gt`/`lt`
  (`[operando, valor]`). Valem em todo lugar que usa a condição (`offer`, `suggestions`,
  `priority_when`, regras de `decision_table`): o `api` avalia os dois antes de adotar esta
  versão do schema. "60 anos ou mais" passa a ser `gte`, não `gt 59` (ADR 0027).
- `offer` (opcional, na raiz): o que o catálogo do cidadão mostra e a quem. `title` (1..60),
  `summary` (1..200), `eligibility` (condição) e `retake_after_days` (1..3650), todos
  opcionais; `offer: {}` equivale a não ter o bloco. Parte do conteúdo assinado (ADR 0016).
- `suggestions` (opcional, na raiz, até 10): `{ protocol, when }`, ambos obrigatórios — o
  protocolo que a conclusão desta triagem pode sugerir (`pattern` igual ao do `name`) e a condição para isso.
- O schema não distingue variáveis por lugar: `profile.*` em `offer.eligibility`;
  `profile.*`, `outcome.*` e ids de passo em `suggestions[].when` — quem confere é o gate do
  `api`, assim como "sugestão para o próprio protocolo" e "protocolo inexistente na cidade".
- `examples/` + `examples/manifest.json`: definições válidas e inválidas com o erro esperado,
  verificadas com o `json_schemer` do `api` (comando em `protocols/README.md`).
- Expand: nada vira obrigatório e nada foi removido — toda definição válida em `v1.3.0`
  continua válida, por isso MINOR. O `api` atualiza a cópia (`config/protocols/schema.json`)
  no mesmo ciclo, antes do `dashboard` passar a gravar `offer`/`suggestions`.
```

- [ ] **Step 6: README da raiz**

Em `contracts/.claude/mod15/README.md`:
- na tabela "Domínios", `` `protocols-v1.3.0` `` passa a `` `protocols-v1.4.0` ``;
- no parágrafo de `### protocols`, o trecho

```markdown
pergunta (Analytics, ADR 0025), válido só em `boolean` e `enum`. `v1.3.0` faz o schema exigir `options`
(não vazio) em pergunta `enum`, regra que já constava na descrição.
```

passa a

```markdown
pergunta (Analytics, ADR 0025), válido só em `boolean` e `enum`. `v1.3.0` faz o schema exigir `options`
(não vazio) em pergunta `enum`, regra que já constava na descrição. `v1.4.0`
acrescentou `offer` (título, resumo, elegibilidade e intervalo de repetição do
catálogo), `suggestions` e os operadores `gte`/`lte` (ADR 0027). Exemplos
válidos e inválidos ficam em `protocols/examples/`.
```

- [ ] **Step 7: Commit**

```bash
/opt/homebrew/bin/git -C contracts/.claude/mod15 add protocols/schema.json protocols/CHANGELOG.md protocols/examples README.md
/opt/homebrew/bin/git -C contracts/.claude/mod15 status --short
/opt/homebrew/bin/git -C contracts/.claude/mod15 commit -m "feat: add offer and suggestions to the protocols schema (protocols-v1.4.0)

Optional offer block (title, summary, eligibility, retake_after_days) and an
optional suggestions list of { protocol, when } at the root (ADR 0027). Which
variables each place may use stays with the api gate. MINOR per ADR 0015:
nothing removed or made required, every v1.3.0 definition stays valid.

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```
Expected: `status --short` lista `protocols/schema.json`, `protocols/CHANGELOG.md`, `README.md`, `protocols/examples/manifest.json` e os 15 exemplos novos.

### Task 3: Entrega — verificação final e parada antes do push

**Files:** nenhum novo. Só leitura e, com autorização, merge/tag/push.

**Interfaces:**
- Consumes: a branch `feat/protocols-offer-suggestions` com os commits das Tasks 1 e 2.
- Produces: o hash do commit da Task 2 para o plano do `api` citar; depois de autorizado, a tag `protocols-v1.4.0` em `origin`.

- [ ] **Step 1: Verificação final da branch**

```bash
/opt/homebrew/bin/git -C contracts/.claude/mod15 log --oneline origin/main..HEAD
/opt/homebrew/bin/git -C contracts/.claude/mod15 diff --stat origin/main..HEAD
/opt/homebrew/bin/git -C contracts/.claude/mod15 diff origin/main..HEAD -- protocols/schema.json
```
Expected: dois commits (Task 1 e Task 2); `diff --stat` só em `README.md`, `protocols/README.md`, `protocols/CHANGELOG.md`, `protocols/schema.json` e `protocols/examples/` (20 arquivos: manifesto + 19 exemplos); o diff do schema só tem linhas `+` (nenhuma linha removida — MINOR). Rode também o comando de verificação da Task 2, Step 2: `20/20 casos`.

- [ ] **Step 2: Pare e peça autorização**

Reporte ao usuário: hash dos dois commits, `20/20 casos`, e a lista do que falta. **Não execute nada abaixo sem um "sim" explícito do usuário para cada etapa** (a) merge na `main` local, (b) tag, (c) push. Antes do push, confira `origin/main..main`: outra sessão pode ter commits locais no `contracts`; publique só os desta entrega.

- [ ] **Step 3 (só com autorização): Merge fast-forward na main**

```bash
/opt/homebrew/bin/git -C contracts fetch origin
/opt/homebrew/bin/git -C contracts status -sb
/opt/homebrew/bin/git -C contracts checkout main
/opt/homebrew/bin/git -C contracts merge --ff-only feat/protocols-offer-suggestions
```
Expected: `## main...origin/main` sem divergência antes; fast-forward de dois commits. Se não for fast-forward, pare e reporte.

- [ ] **Step 4 (só com autorização): Tag anotada**

```bash
/opt/homebrew/bin/git -C contracts tag -a protocols-v1.4.0 -m "protocols-v1.4.0 — MINOR"
/opt/homebrew/bin/git -C contracts show protocols-v1.4.0 --stat --format='%an %s' | head -12
```
Expected: `tag protocols-v1.4.0`, `protocols-v1.4.0 — MINOR`, e o commit `feat: add offer and suggestions to the protocols schema (protocols-v1.4.0)`.

- [ ] **Step 5 (só com autorização): Push da main e da tag**

```bash
/opt/homebrew/bin/git -C contracts log --oneline origin/main..main
/opt/homebrew/bin/git -C contracts push origin main
/opt/homebrew/bin/git -C contracts push origin protocols-v1.4.0
```
Expected: `origin/main..main` mostra só os dois commits desta entrega; o push publica a `main` e a tag.

- [ ] **Step 6: Limpe o worktree (depois do merge)**

```bash
/opt/homebrew/bin/git -C contracts worktree remove .claude/mod15
/opt/homebrew/bin/git -C contracts branch -d feat/protocols-offer-suggestions
```

- [ ] **Step 7: Confira a cópia do api (depois da tarefa de schema do plano do api)**

Run: `diff contracts/protocols/schema.json apps/api/.claude/mod15/config/protocols/schema.json && echo identicos`
Expected: `identicos`. (Se o worktree do `api` tiver outro caminho, use o `config/protocols/schema.json` dele.)

---

## Divergências com o contrato (aceitas) (`…-contracts.md` §1)

1. **`pattern` de `suggestions[].protocol` — incorporada.** O contrato passou a usar `^[a-z][a-z0-9-]+$`, igual ao `name` (antes `^[a-z0-9][a-z0-9-]{1,63}$`, que aceitava dígito inicial e cortava em 64). Este plano já usa o `pattern` novo (Task 2, Step 3) e prova com `invalid-suggestion-protocol-leading-digit.json`.
2. **Eventos novos (§5) fora do `contracts`.** `citizen.profile_changed`, `triage.suggested` e `triage_offer.changed` são tenant-scoped; o `events/EVENTS.md` já não cataloga vários eventos tenant-scoped de módulos anteriores (ex.: `citizen.neighborhood_changed`). Este plano não abre `events-v2.2.0`; se o usuário quiser o catálogo em dia, é uma entrega própria.
3. **Tokens do visual do wpda.** A spec §7 fala em "tokens do `contracts`", mas `contracts/design-tokens/` é scaffold; a fonte real é `apps/wpda/src/theme/tokens.ts` (+ `global.css`). O plano do dashboard deve apontar para lá.

## Self-review (feito ao escrever o plano)

- **Cobertura da spec:** §4.1 (`gte`/`lte` no formato de `gt`/`lt`) → Task 1; §4.2 (`offer` opcional e tudo dentro opcional, `title` ≤ 60, `summary` ≤ 200, `suggestions`, CHANGELOG, cópia no `api`) → Task 2; §11.1 e contrato §6.1 (tag `protocols-v1.4.0`, push com autorização) → Task 3. Variáveis por lugar, autossugestão e prefixos reservados são do gate do `api` (contrato §1), registrados no CHANGELOG.
- **Fixtures pedidas:** válidos — offer completo (`offer-full`), offer vazio (`offer-empty`), sugestões com `gte`/`lte` (`suggestions-gte-lte`), variáveis reservadas (`reserved-variables`), protocolo sem `offer` (`no-offer` + template real do `api`); inválidos — title > 60, summary > 200, retake 0, sugestão sem `when`, propriedade extra em `offer`, `protocol` com maiúscula, `protocol` começando por dígito; extras: title vazio, retake > 3650, operador desconhecido, 11 sugestões, `gte` com três itens, nó com dois operadores.
- **Placeholders:** nenhum; schema, exemplos (gerados por script exato), manifesto, comando, CHANGELOG, README e mensagens de commit estão completos. As saídas esperadas (ponteiros e tipos de erro) foram conferidas contra o `json_schemer` 2.5.0 do container em 2026-10-05, com a v1.3.0 atual e com o schema final.
- **Consistência:** nomes de arquivo do manifesto = nomes gerados pelos scripts; o comando de verificação é o mesmo no README e nas Tasks 1–2; mensagem da tag segue `protocols-v1.3.0 — MINOR`.
- **Review Focus:** os cinco itens têm exemplo no manifesto.
