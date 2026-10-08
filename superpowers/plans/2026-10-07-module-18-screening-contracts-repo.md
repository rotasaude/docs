# Módulo 18 — variante `screening` no schema de protocolo (repo contracts) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Publicar no repo `contracts` a variante de acolhimento da definição de protocolo — `{ name, version, kind: "screening", risk_rules: [ { when, color } ] }`, com 1 a 50 regras e cores `red`/`yellow`/`green`/`blue` — como `protocols-v1.6.0` (MINOR), sem tirar a validade de nenhuma triagem existente, com exemplos válidos e inválidos versionados. É a fonte da cópia que o `api` valida (F-18.2; ADR 0030).

**Architecture:** Uma edição em `protocols/schema.json`: a raiz perde o `required` fixo e ganha `oneOf` entre duas formas em `$defs` — `triage` (o `required` de hoje, sem `kind` e sem `risk_rules`) e `screening` (`required` da variante e `false` para tudo que é da triagem) —, e `properties` ganha `kind` (`const: "screening"`) e `risk_rules`. As propriedades continuam na raiz, com o `additionalProperties: false` de hoje; as duas formas só dizem o que é obrigatório e o que é proibido. Assim os erros de uma triagem continuam exatamente os de `v1.5.0` (o erro de propriedade aparece uma vez, na raiz) e uma definição que não é objeto continua dando só `(root) object`. Exemplos novos em `protocols/examples/` entram no `manifest.json`; entrada no `protocols/CHANGELOG.md` e versão no `README.md` da raiz. O repo não tem código executável nem suíte: a prova é validar os exemplos com o `json_schemer` 2.5 do `api`, dentro do container, recebendo schema e exemplos pela entrada padrão (comando de `protocols/README.md`). Merge, tag anotada `protocols-v1.6.0` e push só com autorização.

*Nota de 2026-10-07: na execução, o `oneOf` da raiz virou `if`/`then`/`else` sobre `kind` (ver C3). Onde este plano diz `oneOf` entre as formas, leia `if`/`then`/`else`.*

**Tech Stack:** JSON Schema draft 2020-12; `json_schemer` 2.5.0 (do bundle do `api`, só para a verificação); `python3` do host para gerar os exemplos e empacotar schema + exemplos num JSON.

**Spec:** `docs/.claude/ciclo2/superpowers/specs/2026-10-07-module-18-screening-design.md` §3.3 e §11; ADR `docs/.claude/ciclo2/adr/0030.md`. Contratos entre apps (fonte única do formato): `docs/.claude/ciclo2/superpowers/plans/2026-10-07-module-18-screening-contracts.md` §1, §2 e §8. **Este plano roda antes do plano do `api`**, cuja Task 2 copia o `schema.json` final byte a byte para `apps/api/config/protocols/schema.json`.

## Global Constraints

- SemVer por domínio (ADR 0015): só acréscimo → **MINOR**, `protocols-v1.5.0` → `protocols-v1.6.0`. Nada vira obrigatório para a triagem, nada é removido: toda definição válida em `v1.5.0` continua válida, com os mesmos erros quando inválida.
- Formato do contrato §1: `kind` ausente = triagem (compatível); `kind: "screening"` exige `name`, `version` e `risk_rules` (1–50) e proíbe `steps`, `scoring`, `offer`, `suggestions`, `scheduling`. Regra = `{ "when": <condição>, "color": "red" | "yellow" | "green" | "blue" }`, as duas chaves obrigatórias, nada além delas.
- `when` é **só** a condição estruturada (`$ref #/$defs/condition`), como em `suggestions[].when` e `scheduling[].when`; o mapa simples (`{ "vitals.systolic": "180" }`) é recusado.
- O schema **não** confere as variáveis do `when` (`vitals.*`, `complaint.ciap2`, `profile.*`), nem o nome reservado `acolhimento`, nem "uma versão `active` por nome": isso é do gate do `api` (contrato §1). Nenhum `pattern` de variável nem `const` de nome entram aqui.
- `schema_version` continua inteiro (`minimum: 1`), como hoje, nas duas formas (ver Divergência C1).
- CHANGELOG obrigatório: sem entrada em `protocols/CHANGELOG.md`, não há tag.
- Tag **anotada**, mensagem `protocols-v1.6.0 — MINOR` (mesmo formato das anteriores), criada no commit que chega à `main`.
- O `api` usa uma **cópia** (`apps/api/config/protocols/schema.json`, hoje idêntica a `contracts/protocols/schema.json` `v1.5.0` — conferido em 2026-10-07 com `diff` entre os dois `origin/main`): os dois arquivos ficam byte a byte iguais no mesmo ciclo. O `dashboard` só grava protocolo `screening` depois que a cópia do `api` for trocada.
- Tipos TypeScript e design tokens: nada a atualizar (`types/` e `design-tokens/` são scaffold). Eventos do módulo (contrato §7) são da cidade e ficam fora de `events/EVENTS.md`, como nos módulos 15 e 17.
- Worktree `contracts/.claude/mod18`, branch `feat/protocols-screening` (`contracts/.gitignore` já ignora `/.claude/`). Todos os comandos rodam a partir da raiz do monorepo `/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, com o compose de pé (`docker compose ps` mostra `api running`).
- Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Nunca `git add -A`: caminhos explícitos. Commits em inglês, Conventional Commits com tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
- Nada de merge na `main`, tag ou push sem autorização explícita do usuário, uma etapa de cada vez.

## Review Focus

1. **Triagem existente continua válida e com os mesmos erros** — publicar a variante não pode tornar inválida nenhuma versão ativa nas cidades, nem trocar a lista de erros que o editor mostra (o `api` repassa `schema: <ponteiro> <tipo>` ao autor; um `oneOf` ingênuo despejaria erros da outra forma em toda triagem com defeito). Testes: os 41 casos de `v1.5.0` no manifesto e o template real `apps/api/config/city_templates/triage_respiratoria.json` em toda execução (Task 1, Steps 4 e 6), com as mesmas expectativas de antes.
2. **Acolhimento com sobra de triagem** (`steps`, `scoring`, `offer`, `suggestions`, `scheduling`, e também `recommendations`) — quem converte uma triagem em acolhimento deixa blocos para trás; o schema recusa apontando o bloco. Testes: `invalid-screening-with-steps.json` … `invalid-screening-with-recommendations.json` (Task 1), esperado `/<bloco> schema`.
3. **Triagem com pedaço do acolhimento** (`risk_rules` sem `kind`, ou `kind: "triage"`) — recusada, para a triagem nunca carregar regra de cor que ninguém avalia. Testes: `invalid-triage-with-risk-rules.json` (`/risk_rules schema`) e `invalid-triage-kind-triage.json` (`/kind const`) (Task 1).
4. **Cor fora da escala ou escrita de outro jeito** (`orange`, `Red`) — a escala é fixa (CAB 28). Testes: `invalid-screening-color-orange.json` e `invalid-screening-color-uppercase.json` (Task 1), esperado `/risk_rules/0/color enum`.
5. **Limites da lista e decimal na condição** — 0 regras, 51 regras, a 50ª, e `{ "gte": ["vitals.temperature_c", 37.8] }`. Testes: `invalid-screening-empty-risk-rules.json` (`/risk_rules minItems`), `invalid-screening-too-many-rules.json` (`/risk_rules maxItems`), `screening-fifty-rules.json` e `screening-cab28.json` (`valid`) (Task 1).

## Mapa de arquivos

| Arquivo (repo `contracts`) | Mudança | Task |
|---|---|---|
| `protocols/schema.json` | raiz com `oneOf` (`$defs/triage`, `$defs/screening`); `properties.kind`, `properties.risk_rules` | 1 |
| `protocols/examples/*.json` | 26 exemplos novos (4 válidos, 22 inválidos) | 1 |
| `protocols/examples/manifest.json` | 26 casos novos (67 no total) | 1 |
| `protocols/CHANGELOG.md` | entrada `protocols-v1.6.0` | 1 |
| `README.md` | versão na tabela e parágrafo de `### protocols` | 1 |

---

### Task 1: variante `screening`, exemplos, CHANGELOG e README (`protocols-v1.6.0`)

**Files (repo `contracts`, worktree `contracts/.claude/mod18`):**
- Create: `protocols/examples/screening-cab28.json`, `screening-one-rule.json`, `screening-fifty-rules.json`, `screening-other-name.json`
- Create: `protocols/examples/invalid-screening-without-risk-rules.json`, `invalid-screening-without-name.json`, `invalid-screening-name-uppercase.json`, `invalid-screening-kind-unknown.json`, `invalid-screening-schema-version-string.json`, `invalid-screening-rules-not-array.json`, `invalid-screening-empty-risk-rules.json`, `invalid-screening-too-many-rules.json`, `invalid-screening-color-orange.json`, `invalid-screening-color-uppercase.json`, `invalid-screening-rule-without-color.json`, `invalid-screening-rule-without-when.json`, `invalid-screening-rule-extra-property.json`, `invalid-screening-when-simple-map.json`, `invalid-screening-with-steps.json`, `invalid-screening-with-scoring.json`, `invalid-screening-with-recommendations.json`, `invalid-screening-with-offer.json`, `invalid-screening-with-suggestions.json`, `invalid-screening-with-scheduling.json`, `invalid-triage-with-risk-rules.json`, `invalid-triage-kind-triage.json`
- Modify: `protocols/examples/manifest.json`
- Modify: `protocols/schema.json` (topo da raiz; fim de `properties`; começo de `$defs`)
- Modify: `protocols/CHANGELOG.md`
- Modify: `README.md`

**Interfaces:**
- Consumes: `$defs/condition` (com `gte`/`lte`, `protocols-v1.4.0`), o manifesto e o comando de verificação de `protocols/README.md`, e `protocols/examples/scheduling-full.json` (`v1.5.0`) como base das triagens inválidas.
- Produces: `protocols-v1.6.0` — `properties.kind` (`const: "screening"`), `properties.risk_rules` (1–50 de `{ when: $defs/condition, color: enum }`), `$defs/triage`, `$defs/screening`, e a raiz com `oneOf` entre as duas. Consumido pelo plano do `api` (cópia literal em `config/protocols/schema.json`; o gate confere variáveis e o nome reservado) e, via `api`, pelo editor do `dashboard` (painel "Regras de cor"). Formato do manifesto inalterado: `{ "cases": [ { "file": "<nome>.json", "expect": "valid" | "<data_pointer> <type>" } ] }`.

- [ ] **Step 1: Crie o worktree**

```bash
/opt/homebrew/bin/git -C contracts fetch origin
/opt/homebrew/bin/git -C contracts log --oneline -1 origin/main
/opt/homebrew/bin/git -C contracts worktree add .claude/mod18 -b feat/protocols-screening origin/main
diff contracts/.claude/mod18/protocols/schema.json apps/api/config/protocols/schema.json && echo identicos
```
Expected: `origin/main` em `8298709 feat: add scheduling rules to the protocols schema (protocols-v1.5.0)` (ou mais novo — se houver commit novo em `protocols/`, pare e reporte); worktree criado; `identicos` (se o checkout do `api` estiver em outra branch, compare com `/opt/homebrew/bin/git -C apps/api show origin/main:config/protocols/schema.json`).

- [ ] **Step 2: Escreva os exemplos**

A partir da raiz do monorepo. As triagens inválidas partem de `scheduling-full.json` (exemplo válido de `v1.5.0`):

```bash
cd contracts/.claude/mod18/protocols/examples && python3 - <<'EOF'
import json

def rule(**over):
    r = {"when": {"gte": ["vitals.systolic", 180]}, "color": "red"}
    r.update(over)
    return r

def base(**extra):
    d = {"name": "acolhimento", "version": 1, "kind": "screening", "risk_rules": [rule()]}
    d.update(extra)
    return d

def without(key):
    d = base()
    del d[key]
    return d

def triage(**extra):
    with open("scheduling-full.json", encoding="utf-8") as f:
        d = json.load(f)
    d.update(extra)
    return d

files = {
    "screening-cab28.json": base(schema_version=1, risk_rules=[
        {"when": {"any": [{"gte": ["vitals.systolic", 180]}, {"lt": ["vitals.spo2", 90]}]}, "color": "red"},
        {"when": {"any": [{"gte": ["vitals.temperature_c", 39]}, {"gte": ["vitals.capillary_glucose", 300]}]}, "color": "yellow"},
        {"when": {"all": [{"gte": ["profile.age", 60]}, {"eq": ["profile.sex", "female"]}, {"gte": ["vitals.pain_score", 8]}]}, "color": "yellow"},
        {"when": {"all": [{"eq": ["vitals.glucose_moment", "fasting"]}, {"gte": ["vitals.capillary_glucose", 126]}]}, "color": "green"},
        {"when": {"gte": ["vitals.temperature_c", 37.8]}, "color": "green"},
        {"when": {"in": ["complaint.ciap2", ["A98", "A97"]]}, "color": "blue"}]),
    "screening-one-rule.json": base(),
    "screening-fifty-rules.json": base(risk_rules=[rule(when={"gte": ["vitals.heart_rate", 100 + i]}) for i in range(50)]),
    "screening-other-name.json": base(name="acolhimento-pediatrico"),
    "invalid-screening-without-risk-rules.json": without("risk_rules"),
    "invalid-screening-without-name.json": without("name"),
    "invalid-screening-name-uppercase.json": base(name="Acolhimento"),
    "invalid-screening-kind-unknown.json": base(kind="home_visit"),
    "invalid-screening-schema-version-string.json": base(schema_version="1.6.0"),
    "invalid-screening-rules-not-array.json": base(risk_rules=rule()),
    "invalid-screening-empty-risk-rules.json": base(risk_rules=[]),
    "invalid-screening-too-many-rules.json": base(risk_rules=[rule(when={"gte": ["vitals.heart_rate", 100 + i]}) for i in range(51)]),
    "invalid-screening-color-orange.json": base(risk_rules=[rule(color="orange")]),
    "invalid-screening-color-uppercase.json": base(risk_rules=[rule(color="Red")]),
    "invalid-screening-rule-without-color.json": base(risk_rules=[{"when": {"gte": ["vitals.systolic", 180]}}]),
    "invalid-screening-rule-without-when.json": base(risk_rules=[{"color": "red"}]),
    "invalid-screening-rule-extra-property.json": base(risk_rules=[rule(priority=1)]),
    "invalid-screening-when-simple-map.json": base(risk_rules=[rule(when={"vitals.systolic": "180"})]),
    "invalid-screening-with-steps.json": base(start_step_id="q", steps=[{"id": "q", "prompt": "Tem febre?", "answer_type": "boolean"}]),
    "invalid-screening-with-scoring.json": base(scoring={"type": "weighted", "thresholds": {"baixa": 0}}),
    "invalid-screening-with-recommendations.json": base(recommendations={"red": {"title": "Atendimento imediato", "body": "Procure a equipe."}}),
    "invalid-screening-with-offer.json": base(offer={"title": "Acolhimento"}),
    "invalid-screening-with-suggestions.json": base(suggestions=[{"protocol": "saude-do-idoso", "when": {"lt": ["vitals.spo2", 90]}}]),
    "invalid-screening-with-scheduling.json": base(scheduling=[{"when": {"gte": ["vitals.systolic", 140]}, "appointment_type": "consulta_medica", "priority": "routine", "due_in_days": 7}]),
    "invalid-triage-with-risk-rules.json": triage(risk_rules=[rule()]),
    "invalid-triage-kind-triage.json": triage(kind="triage"),
}
for name, doc in files.items():
    with open(name, "w", encoding="utf-8") as f:
        f.write(json.dumps(doc, indent=2, ensure_ascii=False) + "\n")
print(len(files))
EOF
```
Expected: `26`. (`screening-other-name.json` é válido de propósito: o nome reservado `acolhimento` é regra do gate do `api`, não do schema.)

- [ ] **Step 3: Acrescente os casos ao manifesto**

Em `contracts/.claude/mod18/protocols/examples/manifest.json`, troque o último caso

```json
    { "file": "invalid-scheduling-too-many.json", "expect": "/scheduling maxItems" }
  ]
}
```

por

```json
    { "file": "invalid-scheduling-too-many.json", "expect": "/scheduling maxItems" },
    { "file": "screening-cab28.json", "expect": "valid" },
    { "file": "screening-one-rule.json", "expect": "valid" },
    { "file": "screening-fifty-rules.json", "expect": "valid" },
    { "file": "screening-other-name.json", "expect": "valid" },
    { "file": "invalid-screening-without-risk-rules.json", "expect": "(root) required" },
    { "file": "invalid-screening-without-name.json", "expect": "(root) required" },
    { "file": "invalid-screening-name-uppercase.json", "expect": "/name pattern" },
    { "file": "invalid-screening-kind-unknown.json", "expect": "/kind const" },
    { "file": "invalid-screening-schema-version-string.json", "expect": "/schema_version integer" },
    { "file": "invalid-screening-rules-not-array.json", "expect": "/risk_rules array" },
    { "file": "invalid-screening-empty-risk-rules.json", "expect": "/risk_rules minItems" },
    { "file": "invalid-screening-too-many-rules.json", "expect": "/risk_rules maxItems" },
    { "file": "invalid-screening-color-orange.json", "expect": "/risk_rules/0/color enum" },
    { "file": "invalid-screening-color-uppercase.json", "expect": "/risk_rules/0/color enum" },
    { "file": "invalid-screening-rule-without-color.json", "expect": "/risk_rules/0 required" },
    { "file": "invalid-screening-rule-without-when.json", "expect": "/risk_rules/0 required" },
    { "file": "invalid-screening-rule-extra-property.json", "expect": "/risk_rules/0/priority schema" },
    { "file": "invalid-screening-when-simple-map.json", "expect": "/risk_rules/0/when required" },
    { "file": "invalid-screening-with-steps.json", "expect": "/steps schema" },
    { "file": "invalid-screening-with-scoring.json", "expect": "/scoring schema" },
    { "file": "invalid-screening-with-recommendations.json", "expect": "/recommendations schema" },
    { "file": "invalid-screening-with-offer.json", "expect": "/offer schema" },
    { "file": "invalid-screening-with-suggestions.json", "expect": "/suggestions schema" },
    { "file": "invalid-screening-with-scheduling.json", "expect": "/scheduling schema" },
    { "file": "invalid-triage-with-risk-rules.json", "expect": "/risk_rules schema" },
    { "file": "invalid-triage-kind-triage.json", "expect": "/kind const" }
  ]
}
```

Confira: `python3 -c 'import json;print(len(json.load(open("contracts/.claude/mod18/protocols/examples/manifest.json"))["cases"]))'` → `67`.

- [ ] **Step 4: Rode a verificação e veja falhar**

O comando de `protocols/README.md`, apontado para o worktree. A partir da raiz do monorepo:

```bash
python3 -c '
import json, os, sys
schema_path, manifest_path, extras = sys.argv[1], sys.argv[2], sys.argv[3:]
base = os.path.dirname(manifest_path)
cases = [dict(c, doc=json.load(open(os.path.join(base, c["file"])))) for c in json.load(open(manifest_path))["cases"]]
cases += [{"file": p, "expect": "valid", "doc": json.load(open(p))} for p in extras]
print(json.dumps({"schema": json.load(open(schema_path)), "cases": cases}))
' contracts/.claude/mod18/protocols/schema.json contracts/.claude/mod18/protocols/examples/manifest.json apps/api/config/city_templates/triage_respiratoria.json \
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

Expected (exit 1): os 41 casos de `v1.5.0`, o template e 5 dos novos `ok` (os que já falhavam em `v1.5.0` pelo motivo esperado: `without-risk-rules`, `without-name`, `name-uppercase`, `schema-version-string` e `triage-with-risk-rules`); 21 `FAIL` — os 4 válidos com `obtido: /kind schema | /risk_rules schema | (root) required`; última linha `47/68 casos`.

- [ ] **Step 5: A variante no schema**

Em `contracts/.claude/mod18/protocols/schema.json`, duas trocas.

(a) No topo da raiz, troque

```json
  "type": "object",
  "required": ["name", "version", "start_step_id", "steps"],
  "additionalProperties": false,
  "properties": {
```

por

```json
  "type": "object",
  "additionalProperties": false,
  "oneOf": [
    { "$ref": "#/$defs/triage" },
    { "$ref": "#/$defs/screening" }
  ],
  "properties": {
```

(b) No fim de `properties` e começo de `$defs`, troque

```json
          "due_in_days":      { "type": "integer", "minimum": 1, "maximum": 365 }
        }
      }
    }
  },
  "$defs": {
    "step": {
```

por

```json
          "due_in_days":      { "type": "integer", "minimum": 1, "maximum": 365 }
        }
      }
    },
    "kind": {
      "const": "screening",
      "description": "Ausente = triagem. screening = acolhimento (ADR 0030): só risk_rules."
    },
    "risk_rules": {
      "type": "array",
      "minItems": 1,
      "maxItems": 50,
      "items": {
        "type": "object",
        "additionalProperties": false,
        "required": ["when", "color"],
        "properties": {
          "when":  { "$ref": "#/$defs/condition" },
          "color": { "enum": ["red", "yellow", "green", "blue"] }
        }
      }
    }
  },
  "$defs": {
    "triage": {
      "description": "Triagem (kind ausente): perguntas, pontuação e blocos opcionais; nunca risk_rules.",
      "required": ["name", "version", "start_step_id", "steps"],
      "not": { "required": ["kind"] },
      "properties": { "risk_rules": false }
    },
    "screening": {
      "description": "Acolhimento (kind: screening, ADR 0030): só regras de cor da escala do CAB 28.",
      "required": ["name", "version", "kind", "risk_rules"],
      "properties": {
        "start_step_id":   false,
        "steps":           false,
        "scoring":         false,
        "recommendations": false,
        "priority_when":   false,
        "offer":           false,
        "suggestions":     false,
        "scheduling":      false
      }
    },
    "step": {
```

(Os dois trechos "antes" são únicos no arquivo. Por que assim e não com a triagem inteira dentro de `$defs/triage`: com as propriedades na raiz, o erro de uma propriedade sai uma vez só e a triagem com defeito mostra no editor os mesmos erros de `v1.5.0`; o `not: { required: ["kind"] }` em vez de `"kind": false` faz uma definição que não é objeto continuar dando só `(root) object`, que a spec `spec/requests/authoring/simulate_offer_spec.rb:73` do `api` compara por igualdade.)

- [ ] **Step 6: Rode a verificação e veja passar**

Run: o mesmo comando do Step 4; depois `python3 -m json.tool contracts/.claude/mod18/protocols/schema.json > /dev/null && echo json-ok`
Expected: 68 linhas `ok`, `68/68 casos`, exit 0; e `json-ok`. (Conferido em 2026-10-07 contra o `json_schemer` 2.5.0 do container, numa cópia de rascunho do schema e dos exemplos: `47/68` antes e `68/68` depois. Para conferir a afirmação do Review Focus 1, o mesmo `json_schemer` dá, com o schema novo: `"x"` → `["(root) object"]`; triagem com `name` inválido e passo sem `answer_type` → `["/name pattern", "/steps/0 required"]`, igual a `v1.5.0`.)

- [ ] **Step 7: CHANGELOG**

Em `contracts/.claude/mod18/protocols/CHANGELOG.md`, logo abaixo de `# Changelog — protocols` (antes da entrada `protocols-v1.5.0`), insira:

```markdown

## protocols-v1.6.0 — 2026-10-07 — MINOR
- Variante de acolhimento (ADR 0030): `kind: "screening"` com `risk_rules` —
  de 1 a 50 regras `{ when, color }`, as duas obrigatórias; `when` é a condição
  estruturada (`$defs/condition`; o mapa simples não vale) e `color` é
  `red`, `yellow`, `green` ou `blue` (escala do Caderno de Atenção Básica nº 28,
  nessa ordem de gravidade). Vale a cor mais grave entre as regras que casarem
  (regra do `api`). A variante não tem `start_step_id`, `steps`, `scoring`,
  `recommendations`, `priority_when`, `offer`, `suggestions` nem `scheduling`.
  Parte do conteúdo assinado (ADR 0016).
- `kind` ausente continua sendo triagem, com os mesmos campos obrigatórios; a
  triagem não aceita `risk_rules` nem `kind`. A raiz passa a ter `oneOf` entre
  `$defs/triage` e `$defs/screening`; as propriedades continuam na raiz, então
  os erros de uma triagem inválida são os mesmos de `v1.5.0`.
- O schema não confere as variáveis do `when` (`vitals.*`, `complaint.ciap2`,
  `profile.*`), o nome reservado `acolhimento` nem "uma versão ativa por nome":
  quem confere é o gate do `api`.
- `examples/`: 26 exemplos novos (4 válidos, 22 inválidos) no mesmo manifesto.
- Expand: nada vira obrigatório para a triagem e nada foi removido — toda
  definição válida em `v1.5.0` continua válida, por isso MINOR. O `api`
  atualiza a cópia (`config/protocols/schema.json`) no mesmo ciclo, antes do
  `dashboard` passar a gravar protocolos de acolhimento.
```

- [ ] **Step 8: README da raiz**

Em `contracts/.claude/mod18/README.md`:
- na tabela "Domínios", `` `protocols-v1.5.0` `` passa a `` `protocols-v1.6.0` ``;
- no parágrafo de `### protocols`, o trecho

```markdown
conclusão da triagem (ADR 0029). Exemplos válidos e inválidos ficam em
`protocols/examples/`.
```

passa a

```markdown
conclusão da triagem (ADR 0029). `v1.6.0` acrescentou a variante de
acolhimento (`kind: "screening"`, ADR 0030): só `risk_rules`, as regras que
sugerem a cor da escuta inicial. Exemplos válidos e inválidos ficam em
`protocols/examples/`.
```

Confira: `grep -n 'protocols-v1' contracts/.claude/mod18/README.md` mostra só `protocols-v1.6.0` na tabela (e o `protocols-vX.Y.Z` genérico da seção de versionamento).

- [ ] **Step 9: Commit**

```bash
/opt/homebrew/bin/git -C contracts/.claude/mod18 add protocols/schema.json protocols/CHANGELOG.md protocols/examples README.md
/opt/homebrew/bin/git -C contracts/.claude/mod18 status --short
/opt/homebrew/bin/git -C contracts/.claude/mod18 commit -m "feat: add screening variant to the protocols schema (protocols-v1.6.0)

kind: screening definitions carry only risk_rules, 1 to 50 { when, color }
rules on the CAB 28 colour scale (ADR 0030). The root becomes a oneOf between
the triage and screening shapes while properties stay at the root, so an
invalid triage reports the same errors as before. Variables and the reserved
acolhimento name stay with the api gate. MINOR per ADR 0015: every v1.5.0
definition stays valid.

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```
Expected: `status --short` lista `protocols/schema.json`, `protocols/CHANGELOG.md`, `README.md`, `protocols/examples/manifest.json` e os 26 exemplos novos — nada mais.

### Task 2: Entrega — verificação final e parada antes do merge

**Files:** nenhum novo. Só leitura e, com autorização, merge/tag/push.

**Interfaces:**
- Consumes: a branch `feat/protocols-screening` com o commit da Task 1.
- Produces: o hash do commit da Task 1 para o plano do `api` citar; depois de autorizado, a tag `protocols-v1.6.0` em `origin` — pré-requisito do plano do `api` (contrato §8, passo 1).

- [ ] **Step 1: Verificação final da branch**

```bash
/opt/homebrew/bin/git -C contracts/.claude/mod18 log --oneline origin/main..HEAD
/opt/homebrew/bin/git -C contracts/.claude/mod18 diff --stat origin/main..HEAD
/opt/homebrew/bin/git -C contracts/.claude/mod18 diff origin/main..HEAD -- protocols/schema.json | grep '^-' | grep -v '^---'
```
Expected: um commit (`feat: add screening variant to the protocols schema (protocols-v1.6.0)`); `diff --stat` só em `README.md`, `protocols/CHANGELOG.md`, `protocols/schema.json` e `protocols/examples/` (27 arquivos: manifesto + 26 exemplos); as linhas removidas do schema são só `  "required": ["name", "version", "start_step_id", "steps"],` (movida para `$defs/triage`) e, no máximo, um `    }` que vira `    },`. Rode de novo o comando de verificação da Task 1, Step 4: `68/68 casos`.

- [ ] **Step 2: Pare e peça autorização**

Reporte ao usuário: hash do commit, `68/68 casos`, as Divergências C1–C3 abaixo e a lista do que falta. **Não execute nada abaixo sem um "sim" explícito do usuário para cada etapa** (a) merge na `main` local, (b) tag, (c) push. Antes do push, confira `origin/main..main`: outra sessão pode ter commits locais no `contracts`; publique só os desta entrega.

- [ ] **Step 3 (só com autorização): Merge fast-forward na main**

```bash
/opt/homebrew/bin/git -C contracts fetch origin
/opt/homebrew/bin/git -C contracts status -sb
/opt/homebrew/bin/git -C contracts checkout main
/opt/homebrew/bin/git -C contracts merge --ff-only feat/protocols-screening
```
Expected: `## main...origin/main` sem divergência antes; fast-forward de um commit. Se não for fast-forward, pare e reporte: `git -C contracts/.claude/mod18 rebase origin/main`; conflito só pode aparecer em `README.md` (tabela de versões) ou `protocols/CHANGELOG.md` (entrada nova no topo — mantenha as duas, a mais nova primeiro); repita a verificação (`68/68` ou mais) antes de tentar de novo.

- [ ] **Step 4 (só com autorização): Tag anotada**

```bash
/opt/homebrew/bin/git -C contracts tag -a protocols-v1.6.0 -m "protocols-v1.6.0 — MINOR"
/opt/homebrew/bin/git -C contracts show protocols-v1.6.0 --stat --format='%an %s' | head -12
```
Expected: `tag protocols-v1.6.0`, `protocols-v1.6.0 — MINOR`, e o commit `feat: add screening variant to the protocols schema (protocols-v1.6.0)`.

- [ ] **Step 5 (só com autorização): Push da main e da tag**

```bash
/opt/homebrew/bin/git -C contracts log --oneline origin/main..main
/opt/homebrew/bin/git -C contracts push origin main
/opt/homebrew/bin/git -C contracts push origin protocols-v1.6.0
```
Expected: `origin/main..main` mostra só o commit desta entrega; o push publica a `main` e a tag.

- [ ] **Step 6: Limpe o worktree (depois do merge)**

```bash
/opt/homebrew/bin/git -C contracts worktree remove .claude/mod18
/opt/homebrew/bin/git -C contracts branch -d feat/protocols-screening
```

- [ ] **Step 7: Confira a cópia do api (depois da Task 2 do plano do api)**

Run: `diff contracts/protocols/schema.json apps/api/.claude/mod18/config/protocols/schema.json && echo identicos`
Expected: `identicos`.

---

## Divergências propostas ao contrato

1. **C1 — `schema_version` do exemplo do contrato §1.** O exemplo escreve `"schema_version": "1.6.0"` (texto), mas `schema_version` é inteiro desde `v1.0.0` e o contrato não pede mudança de tipo. Copiado como está, o exemplo é recusado (`/schema_version integer`, provado por `invalid-screening-schema-version-string.json`). Proposta: corrigir o exemplo do contrato para `"schema_version": 1` (ou tirar a chave). Aceitar texto seria widening de tipo fora do escopo.
2. **C2 — o que a variante proíbe.** O contrato lista `steps`, `scoring`, `offer`, `suggestions`, `scheduling`. Como a raiz tem `additionalProperties: false` e cada forma diz o que é seu, a variante também recusa `start_step_id`, `recommendations` e `priority_when` (são da triagem e não teriam efeito no acolhimento). Proposta: o contrato §1 dizer "só `name`, `version`, `schema_version`, `kind` e `risk_rules`".
3. **C3 — forma do `oneOf` (observação, sem mudar formato).** O contrato diz "via `oneOf` com a definição de triagem existente". Mover a triagem inteira para dentro de um ramo do `oneOf` foi testado e descartado: o `json_schemer` (formato `classic`, o que o `api` usa) devolve os erros dos dois ramos, e toda triagem com defeito passaria a mostrar no editor `schema: /steps schema`, `/start_step_id schema` etc. (dizendo que blocos válidos são proibidos), e uma definição que não é objeto daria `(root) object` duas vezes, quebrando `spec/requests/authoring/simulate_offer_spec.rb:73`. O desenho do Step 5 mantém o `oneOf` com as propriedades na raiz e os erros de hoje. Resta um efeito pequeno para o plano do `api`: definição `{}` (ou ausente) dá `(root) required` duas vezes (uma por forma); `Protocols::Validation::Schema.call` pode aplicar `.uniq` antes de prefixar `schema: `.

   *Resultado (2026-10-07): o `oneOf` também foi trocado. A tag publicada (`protocols-v1.6.0`, contracts `0c6c753`, "route protocol variants by kind") escolhe a forma por `if`/`then`/`else` sobre `kind` — com `kind`, `$defs/screening`; sem `kind`, `$defs/triage` —, então só a forma escolhida reporta erros e o `(root) required` duplicado desaparece. Os documentos válidos e inválidos são os mesmos do `oneOf`. O contrato §1 foi corrigido.*

## Self-review (feito ao escrever o plano)

- **Cobertura:** contrato §1 (variante, `kind` ausente compatível, `risk_rules` 1–50 com cores, proibições) → Task 1 Step 5 e exemplos; §2 (cores) → enum; §8 passo 1 (tag com autorização) → Task 2. Spec §3.3 ("até 50 regras", "mesmo ciclo de assinatura", nome reservado e variáveis no gate) → schema + CHANGELOG; spec §11 passo 1 → Task 2.
- **Fixtures:** válidos — conjunto CAB 28 com `any`/`all`/`in`, decimal (`37.8`), `profile.*`, `vitals.glucose_moment` e `complaint.ciap2`; uma regra; 50 regras; outro nome. Inválidos — cada obrigatório ausente, nome fora do padrão, `kind` desconhecido, `schema_version` texto, lista que não é lista, 0 e 51 regras, cor fora da escala e em maiúscula, regra sem cor, sem `when`, com chave extra, `when` de mapa simples, cada bloco da triagem dentro do acolhimento, `risk_rules` numa triagem e `kind: "triage"`.
- **Placeholders:** nenhum. Script, manifesto, trocas exatas no schema, comando, CHANGELOG, README e commit completos. Saídas esperadas (`47/68` e `68/68`, ponteiros e tipos) conferidas no container em 2026-10-07 numa cópia de rascunho.
- **Consistência:** nomes do manifesto = nomes do script; comando = o de `protocols/README.md`; tag no formato `protocols-vX.Y.Z — MINOR`; branch e worktree os pedidos (`feat/protocols-screening`, `contracts/.claude/mod18`).
- **Review Focus:** os cinco itens têm caso no manifesto (Task 1), e o item 1 também o template real em toda execução.
