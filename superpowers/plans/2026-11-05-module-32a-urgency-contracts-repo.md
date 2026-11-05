# Módulo 32a — variante `urgency_classification` no schema de protocolo (repo contracts) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Publicar no repo `contracts` a variante de classificação de risco da UPA, `{ name, version, kind: "urgency_classification", levels: [5 cores com max_wait_minutes], classification_rules: [ { id, when, color, criterion } ] }`, como `protocols-v1.7.0` (MINOR). Nenhuma triagem nem acolhimento existente perde a validade, e os exemplos válidos e inválidos ficam versionados. É a fonte da cópia que o `api` valida (F-32.1; ADR 0050; contrato §3).

**Architecture:** Uma edição em `protocols/schema.json`, no molde da `v1.6.0`. O `then` da raiz (com `kind`) vira um segundo `if`/`then`/`else`: `kind: "urgency_classification"` vai para `$defs/urgency_classification`; qualquer outro `kind` continua indo para `$defs/screening`, como hoje. A raiz ganha `levels` e `classification_rules` em `properties`. O `const` de `kind` sai da raiz e vai para dentro de `$defs/screening`. Assim um `kind` desconhecido continua dando exatamente `/kind const`, como na `v1.6.0`. Cada forma diz o que é seu e o que é proibido: triagem e acolhimento recusam `levels` e `classification_rules`, e a urgência recusa `risk_rules` e todos os blocos da triagem. O repo não tem código executável nem suíte: a prova é validar os exemplos com o `json_schemer` do `api`, dentro do container, pelo comando de `protocols/README.md`. Merge, tag anotada `protocols-v1.7.0` e push só com autorização.

**Tech Stack:** JSON Schema draft 2020-12 (`prefixItems`, `if`/`then`/`else`); `json_schemer` do bundle do `api` (só para verificar); `python3` do host para gerar os exemplos e editar o schema.

**Spec:** `docs/superpowers/specs/2026-11-05-module-32-urgency-design.md` (decisões 1 e 8, R5, §8 F-32.1, §9 "Protocolo da cidade" e "Licença do Manchester"); `docs/adr/0050.md`. O contrato entre apps `docs/superpowers/plans/2026-11-05-module-32a-urgency-contracts.md` (§1 "Cores", §3 e §13 "Ordem de entrega") é a fonte única. As diferenças estão em "Divergências propostas ao contrato" (C1–C6). Molde: `docs/superpowers/plans/2026-10-07-module-18-screening-contracts-repo.md`. **Este plano roda antes do plano do `api`**, que copia o `schema.json` final byte a byte para `apps/api/config/protocols/schema.json`.

## Global Constraints

- SemVer por domínio (ADR 0015): só acréscimo, então **MINOR**, de `protocols-v1.6.0` para `protocols-v1.7.0`. Toda definição válida em `v1.6.0` continua válida. Toda inválida continua com o **mesmo conjunto** de erros (só a ordem de um caso muda, ver Review Focus 1).
- Cores (contrato §1): `red`, `orange`, `yellow`, `green`, `blue`, nessa ordem de prioridade. `levels` tem exatamente essas 5, nessa ordem, cada uma com `max_wait_minutes` inteiro de 0 a 1440 (C3). Os padrões 0/10/60/120/240 ficam no exemplo `urgency-default.json` e no CHANGELOG, **não** no schema (sem `default`).
- Regra (contrato §3, C1, C2): `{ "id", "when", "color", "criterion" }`, as quatro obrigatórias e nada além. `id` com `^[a-z][a-z0-9_-]{0,39}$`; `when` só como condição estruturada (`$ref #/$defs/condition`, o mapa simples é recusado); `color` é uma das 5 cores; `criterion` é texto de 1 a 120 caracteres. A lista tem de 1 a 100 regras.
- A escala do acolhimento (`risk_rules`, 4 cores, CAB 28) **não muda**: `orange` continua inválido em `risk_rules`.
- O schema **não** confere as variáveis do `when`, a unicidade de `id`, a ordem crescente dos tempos, "um protocolo ativo de urgência por cidade" nem qual regra vence. Tudo isso é do gate e do motor do `api` (C2, C3, C4, C6).
- Nenhum conteúdo licenciado do Manchester entra nos exemplos (spec decisão 1). Os critérios dos exemplos são genéricos e servem só para provar o formato.
- CHANGELOG obrigatório: sem entrada em `protocols/CHANGELOG.md`, não há tag. A tag é **anotada**, com a mensagem `protocols-v1.7.0 — MINOR`, criada no commit que chega à `main`.
- O `api` usa uma **cópia** (`apps/api/config/protocols/schema.json`). Em 2026-11-05 ela era idêntica a `contracts/protocols/schema.json` `v1.6.0`: foi conferido com `diff` entre os dois `origin/main`. As duas ficam byte a byte iguais no mesmo ciclo. O `dashboard` só grava protocolo de urgência depois que a cópia do `api` for trocada.
- Os eventos do módulo (contrato §12) são da cidade e ficam fora de `events/EVENTS.md`. `types/` e `design-tokens/` não mudam.
- Worktree em `/Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/contracts-mod32a`, branch `feat/protocols-urgency`. Os comandos rodam a partir da raiz do monorepo, com o compose de pé (`docker compose ps` mostra `api running`).
- Git: `/opt/homebrew/bin/git`. Nunca `git add -A` nem `git add .`: caminhos explícitos. Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Merge, tag e push só com autorização explícita do usuário, uma etapa de cada vez.
- Board: a primeira ação da execução é mover F-32.1..F-32.6 do board #1 para In Progress (Task 1, Step 1). O contracts é dono de F-32.1 junto com o api, o dashboard e o maintenance. O card vai para Done quando o último dos quatro terminar, e para Verified depois da prova no navegador.

## Review Focus

1. **Triagem e acolhimento existentes com os mesmos erros.** O editor do `dashboard` mostra ao autor `schema: <ponteiro> <tipo>`. Se o roteamento novo despejasse erros da forma de urgência num acolhimento com defeito, o autor veria blocos válidos como proibidos. Teste: os 68 casos da `v1.6.0` e o template `apps/api/config/city_templates/triage_respiratoria.json`, em toda execução (Task 1, Steps 4 e 6), com as mesmas expectativas. O Step 6 compara a saída dos 68 casos com a da `v1.6.0`: só `invalid-triage-kind-triage.json` muda, e só na ordem.
2. **Acolhimento convertido em urgência (ou o contrário).** Quem copia um protocolo deixa `risk_rules` para trás, ou põe `levels` no acolhimento. O schema recusa e aponta o bloco. Testes: `invalid-urgency-with-risk-rules.json` (`/risk_rules schema`), `invalid-screening-with-levels.json` e `invalid-screening-with-classification-rules.json` (Task 1).
3. **Níveis incompletos, a mais ou fora de ordem.** São 4 cores, 6 cores, laranja antes de vermelho, mapa em vez de lista. A fila da UPA ordena por essa lista, e cor faltando deixaria paciente sem tempo máximo. Testes: `invalid-urgency-levels-four.json` (`/levels minItems`), `invalid-urgency-levels-six.json` (`/levels maxItems`), `invalid-urgency-levels-out-of-order.json` (`/levels/0/color const`) e `invalid-urgency-levels-not-array.json` (`/levels array`) (Task 1).
4. **Tempo máximo estranho.** Negativo, fracionário, em texto, acima de 24 h, ou zero em várias cores. Os limites 0 e 1440 valem; o resto é recusado. Testes: `urgency-wait-bounds.json` (`valid`) e `invalid-urgency-wait-negative.json`, `-too-big`, `-fraction`, `-string` (Task 1).
5. **Regra sem o que a enfermagem precisa ver.** Sem `criterion`, critério vazio ou com mais de 120 caracteres, sem `id` (o `rule_id` do `Urgency::Suggest`), ou com cor fora da escala de 5 (`purple`, `Red`). Testes: `invalid-urgency-rule-without-criterion.json`, `-without-id.json`, `-criterion-empty.json`, `-criterion-too-long.json`, `-rule-color-purple.json` e `-rule-color-uppercase.json` (Task 1).

## Mapa de arquivos

| Arquivo (repo `contracts`) | Mudança | Task |
|---|---|---|
| `protocols/schema.json` | `then` da raiz com `if` sobre `kind`; `properties.levels`, `properties.classification_rules`; `kind` sem `const` na raiz; `$defs/urgency_classification`, `$defs/urgency_level`; triagem e acolhimento recusam os blocos novos | 1 |
| `protocols/examples/*.json` | 38 exemplos novos (4 válidos, 34 inválidos) | 1 |
| `protocols/examples/manifest.json` | 38 casos novos (106 no total) | 1 |
| `protocols/CHANGELOG.md` | entrada `protocols-v1.7.0` | 1 |
| `README.md` | versão na tabela e frase em `### protocols` | 1 |

**Estratégia de teste:** manifesto com o erro esperado por arquivo (`<data_pointer> <type>` do `json_schemer`), mais o template real da triagem. Primeiro roda contra o schema `v1.6.0`, que deve falhar em 27 casos, e depois contra o novo, que deve passar em todos. A saída dos 68 casos antigos é comparada linha a linha com a da `v1.6.0`. Os números deste plano (`80/107` e `107/107`) foram conferidos no container em 2026-11-05, numa cópia de rascunho.

---

### Task 1: Variante `urgency_classification`, exemplos, CHANGELOG e README (`protocols-v1.7.0`)

**Files (worktree `.worktrees/contracts-mod32a`):**
- Create: `protocols/examples/urgency-default.json`, `urgency-one-rule.json`, `urgency-hundred-rules.json`, `urgency-wait-bounds.json`.
- Create: os 34 inválidos da tabela do Step 3.
- Modify: `protocols/examples/manifest.json`, `protocols/schema.json`, `protocols/CHANGELOG.md`, `README.md`.

**Interfaces:**
- Consumes: `$defs/condition` (com `gte`/`lte`, `v1.4.0`); o roteamento por `kind` da `v1.6.0`; `protocols/examples/screening-one-rule.json` e `no-offer.json` como base dos inválidos de acolhimento e triagem; o comando de `protocols/README.md`.
- Produces: `protocols-v1.7.0`, com:
  - `properties.levels`: 5 itens `{ color, max_wait_minutes }` pela ordem `red, orange, yellow, green, blue`;
  - `properties.classification_rules`: de 1 a 100 itens `{ id, when, color, criterion }`;
  - `$defs/urgency_classification` e `$defs/urgency_level`.

  O plano do `api` consome o schema como cópia literal. O motor lê `levels[i].max_wait_minutes` e `classification_rules[i].{id, color, criterion}`, e o `Urgency::Suggest` devolve `rule_id` = `id`. O editor do `dashboard` consome via `api`. O formato do manifesto não muda.

- [ ] **Step 1: Cards e worktree**

Um `gh` simples por chamada, sem `$(...)` nem laço:

```bash
gh project item-list 1 --owner rotasaude --limit 500 --format json --jq '.items[] | select(.title | test("^F-32\\.[1-6] ")) | {id, title, status}'
gh project field-list 1 --owner rotasaude --format json --jq '.fields[] | select(.name == "Status") | {id, options}'
gh project item-edit --id <PVTI_do_card> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id_In_Progress>
```

Depois, a partir da raiz do monorepo:

```bash
G=/opt/homebrew/bin/git
$G -C contracts fetch origin && $G -C contracts log --oneline -1 origin/main
$G -C contracts worktree add ../.worktrees/contracts-mod32a -b feat/protocols-urgency origin/main
diff .worktrees/contracts-mod32a/protocols/schema.json <($G -C apps/api show origin/main:config/protocols/schema.json) && echo identicos
```

Expected:
- os seis cards em In Progress ou além (card ausente: **pare e reporte**);
- `origin/main` em `2f0c3e9` ou mais novo. Commit novo em `protocols/`: pare e reporte;
- `identicos`.

- [ ] **Step 2: Escreva os exemplos**

Salve o gerador em `$SP/c32-gen.py` (`SP` = o scratchpad da sessão que executa, fora do repo) e rode com `python3 -I <gerador> .worktrees/contracts-mod32a/protocols/examples`. Os exemplos são o contrato do formato, então o conteúdo é este:

```python
import json, os, sys, copy
d = sys.argv[1]
def w(name, doc):
    with open(os.path.join(d, name), "w") as f:
        json.dump(doc, f, indent=2, ensure_ascii=False); f.write("\n")
LEVELS = [ {"color": c, "max_wait_minutes": m} for c, m in
           [("red", 0), ("orange", 10), ("yellow", 60), ("green", 120), ("blue", 240)] ]
RULES = [
  {"id": "spo2-baixa", "when": {"lt": ["vitals.spo2", 90]}, "color": "red", "criterion": "Saturação de oxigênio abaixo de 90%"},
  {"id": "pas-muito-alta", "when": {"any": [{"gte": ["vitals.systolic", 220]}, {"lte": ["vitals.systolic", 80]}]},
   "color": "orange", "criterion": "Pressão sistólica de 220 ou mais, ou de 80 ou menos"},
  {"id": "dor-intensa", "when": {"gte": ["pain_scale", 8]}, "color": "orange", "criterion": "Dor intensa (8 a 10)"},
  {"id": "febre-alta", "when": {"gte": ["vitals.temperature_c", 39.5]}, "color": "yellow", "criterion": "Temperatura de 39,5 °C ou mais"},
  {"id": "idoso-dor-moderada", "when": {"all": [{"gte": ["profile.age", 60]}, {"gte": ["pain_scale", 4]}]},
   "color": "yellow", "criterion": "60 anos ou mais com dor moderada"},
  {"id": "dor-leve", "when": {"in": ["pain_scale", [1, 2, 3]]}, "color": "green", "criterion": "Dor leve (1 a 3)"},
  {"id": "sem-dor", "when": {"not": {"gt": ["pain_scale", 0]}}, "color": "blue", "criterion": "Sem dor e sinais vitais sem alerta"}
]
BASE = {"name": "classificacao-urgencia", "version": 1, "kind": "urgency_classification", "levels": LEVELS, "classification_rules": RULES}
def v(**over):
    doc = copy.deepcopy(BASE)
    for k, val in over.items():
        if val is None: doc.pop(k, None)
        else: doc[k] = val
    return doc
def lv(i, **over):
    levels = copy.deepcopy(LEVELS); levels[i].update(over); return levels
def rl(**over):
    r = copy.deepcopy(RULES[0]); r.update(over)
    for k in [k for k, val in over.items() if val is None]: r.pop(k)
    return [r]
many = lambda n: [ {"id": f"regra-{i:03d}", "when": {"gte": ["pain_scale", i % 11]}, "color": LEVELS[i % 5]["color"],
                    "criterion": f"Regra {i}"} for i in range(1, n + 1) ]
w("urgency-default.json", BASE)
w("urgency-one-rule.json", v(classification_rules=rl()))
w("urgency-hundred-rules.json", v(classification_rules=many(100)))
w("urgency-wait-bounds.json", v(levels=[{"color": c, "max_wait_minutes": m} for c, m in
    [("red", 0), ("orange", 0), ("yellow", 1440), ("green", 1440), ("blue", 1440)]], schema_version=1))
w("invalid-urgency-without-levels.json", v(levels=None))
w("invalid-urgency-without-rules.json", v(classification_rules=None))
w("invalid-urgency-levels-four.json", v(levels=LEVELS[:4]))
w("invalid-urgency-levels-six.json", v(levels=LEVELS + [{"color": "white", "max_wait_minutes": 480}]))
w("invalid-urgency-levels-out-of-order.json", v(levels=[LEVELS[1], LEVELS[0]] + LEVELS[2:]))
w("invalid-urgency-levels-not-array.json", v(levels={"red": 0, "orange": 10, "yellow": 60, "green": 120, "blue": 240}))
w("invalid-urgency-level-without-wait.json", v(levels=[l if i != 2 else {"color": "yellow"} for i, l in enumerate(LEVELS)]))
w("invalid-urgency-level-extra-property.json", v(levels=lv(0, label="Emergência")))
w("invalid-urgency-wait-negative.json", v(levels=lv(0, max_wait_minutes=-1)))
w("invalid-urgency-wait-too-big.json", v(levels=lv(4, max_wait_minutes=1441)))
w("invalid-urgency-wait-fraction.json", v(levels=lv(1, max_wait_minutes=10.5)))
w("invalid-urgency-wait-string.json", v(levels=lv(3, max_wait_minutes="120")))
w("invalid-urgency-empty-rules.json", v(classification_rules=[]))
w("invalid-urgency-too-many-rules.json", v(classification_rules=many(101)))
w("invalid-urgency-rule-color-purple.json", v(classification_rules=rl(color="purple")))
w("invalid-urgency-rule-color-uppercase.json", v(classification_rules=rl(color="Red")))
w("invalid-urgency-rule-without-criterion.json", v(classification_rules=rl(criterion=None)))
w("invalid-urgency-rule-without-id.json", v(classification_rules=rl(id=None)))
w("invalid-urgency-rule-without-when.json", v(classification_rules=rl(when=None)))
w("invalid-urgency-rule-id-uppercase.json", v(classification_rules=rl(id="SpO2")))
w("invalid-urgency-criterion-empty.json", v(classification_rules=rl(criterion="")))
w("invalid-urgency-criterion-too-long.json", v(classification_rules=rl(criterion="x" * 121)))
w("invalid-urgency-rule-extra-property.json", v(classification_rules=[dict(RULES[0], max_wait_minutes=0)]))
w("invalid-urgency-when-simple-map.json", v(classification_rules=rl(when={"vitals.spo2": "89"})))
w("invalid-urgency-with-risk-rules.json", v(risk_rules=[{"when": {"lt": ["vitals.spo2", 90]}, "color": "red"}]))
w("invalid-urgency-with-steps.json", v(start_step_id="q1", steps=[{"id": "q1", "prompt": "Tem falta de ar?", "answer_type": "boolean"}]))
w("invalid-urgency-with-scoring.json", v(scoring={"type": "weighted", "thresholds": {"low": 0}}))
w("invalid-urgency-with-offer.json", v(offer={"title": "Urgência"}))
w("invalid-urgency-with-scheduling.json", v(scheduling=[]))
w("invalid-urgency-kind-typo.json", v(kind="urgency"))
scr = json.load(open(os.path.join(d, "screening-one-rule.json")))
w("invalid-screening-with-levels.json", dict(scr, levels=LEVELS))
w("invalid-screening-with-classification-rules.json", dict(scr, classification_rules=RULES[:1]))
tri = json.load(open(os.path.join(d, "no-offer.json")))
w("invalid-triage-with-levels.json", dict(tri, levels=LEVELS))
w("invalid-triage-with-classification-rules.json", dict(tri, classification_rules=RULES[:1]))
```

Expected: `ls .worktrees/contracts-mod32a/protocols/examples | grep -c "urgency\|with-levels\|with-classification"` dá `38`.

- [ ] **Step 3: Acrescente os casos ao manifesto**

No fim de `cases`, depois de `invalid-triage-without-steps.json`, deixe uma linha em branco (como no bloco do 18) e acrescente estes 38 casos, na ordem:

| Arquivo | `expect` |
|---|---|
| `urgency-default.json`, `urgency-one-rule.json`, `urgency-hundred-rules.json`, `urgency-wait-bounds.json` | `valid` |
| `invalid-urgency-without-levels.json`, `invalid-urgency-without-rules.json` | `(root) required` |
| `invalid-urgency-levels-four.json` | `/levels minItems` |
| `invalid-urgency-levels-six.json` | `/levels maxItems` |
| `invalid-urgency-levels-out-of-order.json` | `/levels/0/color const` |
| `invalid-urgency-levels-not-array.json` | `/levels array` |
| `invalid-urgency-level-without-wait.json` | `/levels/2 required` |
| `invalid-urgency-level-extra-property.json` | `/levels/0/label schema` |
| `invalid-urgency-wait-negative.json` | `/levels/0/max_wait_minutes minimum` |
| `invalid-urgency-wait-too-big.json` | `/levels/4/max_wait_minutes maximum` |
| `invalid-urgency-wait-fraction.json` | `/levels/1/max_wait_minutes integer` |
| `invalid-urgency-wait-string.json` | `/levels/3/max_wait_minutes integer` |
| `invalid-urgency-empty-rules.json` | `/classification_rules minItems` |
| `invalid-urgency-too-many-rules.json` | `/classification_rules maxItems` |
| `invalid-urgency-rule-color-purple.json`, `invalid-urgency-rule-color-uppercase.json` | `/classification_rules/0/color enum` |
| `invalid-urgency-rule-without-criterion.json`, `invalid-urgency-rule-without-id.json`, `invalid-urgency-rule-without-when.json` | `/classification_rules/0 required` |
| `invalid-urgency-rule-id-uppercase.json` | `/classification_rules/0/id pattern` |
| `invalid-urgency-criterion-empty.json` | `/classification_rules/0/criterion minLength` |
| `invalid-urgency-criterion-too-long.json` | `/classification_rules/0/criterion maxLength` |
| `invalid-urgency-rule-extra-property.json` | `/classification_rules/0/max_wait_minutes schema` |
| `invalid-urgency-when-simple-map.json` | `/classification_rules/0/when required` |
| `invalid-urgency-with-risk-rules.json` | `/risk_rules schema` |
| `invalid-urgency-with-steps.json` | `/steps schema` |
| `invalid-urgency-with-scoring.json` | `/scoring schema` |
| `invalid-urgency-with-offer.json` | `/offer schema` |
| `invalid-urgency-with-scheduling.json` | `/scheduling schema` |
| `invalid-urgency-kind-typo.json` | `/kind const` |
| `invalid-screening-with-levels.json`, `invalid-triage-with-levels.json` | `/levels schema` |
| `invalid-screening-with-classification-rules.json`, `invalid-triage-with-classification-rules.json` | `/classification_rules schema` |

Cada caso é uma linha `{ "file": "<arquivo>", "expect": "<expect>" },` no estilo do arquivo. Confira com `python3 -c 'import json;print(len(json.load(open(".worktrees/contracts-mod32a/protocols/examples/manifest.json"))["cases"]))'`. Expected: `106`.

- [ ] **Step 4: Rode a verificação e veja falhar**

Rode o comando de `protocols/README.md` com os caminhos do worktree (`.worktrees/contracts-mod32a/protocols/schema.json`, `.worktrees/contracts-mod32a/protocols/examples/manifest.json`, `apps/api/config/city_templates/triage_respiratoria.json`) e guarde a saída em `$SP/c32-v160.txt`.

Expected (exit 1):
- os 68 casos da `v1.6.0` e o template `ok`;
- 11 dos novos `ok`, porque já falham na `v1.6.0` pelo motivo esperado (`/<bloco> schema` da raiz, `(root) required` ou `/kind const`);
- 27 `FAIL`, entre eles os 4 válidos, com `obtido: (root) required | /kind const | /levels schema | /classification_rules schema`;
- última linha `80/107 casos`.

- [ ] **Step 5: A variante no schema**

Em `.worktrees/contracts-mod32a/protocols/schema.json`, seis trocas exatas. Cada trecho antigo aparece uma vez só.

(a) Roteamento na raiz. Troque `  "then": { "$ref": "#/$defs/screening" },` por:

```json
  "then": {
    "if": { "required": ["kind"], "properties": { "kind": { "const": "urgency_classification" } } },
    "then": { "$ref": "#/$defs/urgency_classification" },
    "else": { "$ref": "#/$defs/screening" }
  },
```

(b) `kind` na raiz perde o `const`, que vai para o acolhimento em (e):

```json
    "kind": {
      "description": "Ausente = triagem. screening = acolhimento (ADR 0030): só risk_rules. urgency_classification = classificação de risco da UPA (ADR 0050): só levels e classification_rules. O valor é conferido na forma escolhida."
    },
```

(c) Depois de `risk_rules`, no fim de `properties`:

```json
    "levels": {
      "type": "array",
      "description": "As 5 cores da urgência, na ordem de prioridade, com o tempo máximo de espera de cada uma (ADR 0050).",
      "minItems": 5,
      "maxItems": 5,
      "prefixItems": [
        { "$ref": "#/$defs/urgency_level", "properties": { "color": { "const": "red" } } },
        { "$ref": "#/$defs/urgency_level", "properties": { "color": { "const": "orange" } } },
        { "$ref": "#/$defs/urgency_level", "properties": { "color": { "const": "yellow" } } },
        { "$ref": "#/$defs/urgency_level", "properties": { "color": { "const": "green" } } },
        { "$ref": "#/$defs/urgency_level", "properties": { "color": { "const": "blue" } } }
      ],
      "items": false
    },
    "classification_rules": {
      "type": "array",
      "minItems": 1,
      "maxItems": 100,
      "items": {
        "type": "object",
        "additionalProperties": false,
        "required": ["id", "when", "color", "criterion"],
        "properties": {
          "id":        { "type": "string", "pattern": "^[a-z][a-z0-9_-]{0,39}$" },
          "when":      { "$ref": "#/$defs/condition" },
          "color":     { "enum": ["red", "orange", "yellow", "green", "blue"] },
          "criterion": { "type": "string", "minLength": 1, "maxLength": 120 }
        }
      }
    }
```

(d) Em `$defs/triage`, a linha `"properties": { "risk_rules": false }` passa a ser `"properties": { "risk_rules": false, "levels": false, "classification_rules": false }`.

(e) Em `$defs/screening`, no começo de `properties`:

```json
        "kind":            { "const": "screening" },
        "levels":          false,
        "classification_rules": false,
```

(f) Em `$defs`, antes de `"step"`:

```json
    "urgency_classification": {
      "description": "Classificação de risco da UPA (kind: urgency_classification, ADR 0050): 5 cores com tempo máximo e regras que sugerem a cor com o critério.",
      "required": ["name", "version", "kind", "levels", "classification_rules"],
      "properties": {
        "risk_rules":      false,
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
    "urgency_level": {
      "type": "object",
      "additionalProperties": false,
      "required": ["color", "max_wait_minutes"],
      "properties": {
        "color":            { "type": "string" },
        "max_wait_minutes": { "type": "integer", "minimum": 0, "maximum": 1440 }
      }
    },
```

Confira: `python3 -c 'import json;json.load(open(".worktrees/contracts-mod32a/protocols/schema.json"));print("json-ok")'`. Expected: `json-ok`.

- [ ] **Step 6: Rode a verificação e veja passar**

Rode o mesmo comando do Step 4 e guarde a saída em `$SP/c32-v170.txt`. Depois compare os casos antigos: `diff <(head -68 $SP/c32-v160.txt) <(head -68 $SP/c32-v170.txt)`.

Expected:
- exit 0 e última linha `107/107 casos`;
- o `diff` mostra uma única troca, a de `invalid-triage-kind-triage.json`, em que `/kind const` sai primeiro; o conjunto de erros é o mesmo;
- `kind: 5` num acolhimento continua dando só `/name pattern | /kind const` ou `/kind const`, sem `/kind string`. Por isso `kind` não tem `type` na raiz.

Qualquer outra diferença nos 68 casos: **pare e reporte**.

- [ ] **Step 7: CHANGELOG**

No topo de `protocols/CHANGELOG.md`, depois de `# Changelog — protocols`:

```markdown
## protocols-v1.7.0 — 2026-11-05 — MINOR
- Variante de classificação de risco da UPA (ADR 0050): `kind:
  "urgency_classification"` com `levels` e `classification_rules`.
  - `levels`: exatamente as 5 cores `red`, `orange`, `yellow`, `green`,
    `blue`, nessa ordem de prioridade, cada uma `{ color, max_wait_minutes }`,
    com o tempo máximo inteiro de 0 a 1440 minutos. O padrão sugerido é
    0/10/60/120/240 (exemplo `urgency-default.json`); o schema não impõe valor.
  - `classification_rules`: de 1 a 100 regras `{ id, when, color, criterion }`,
    as quatro obrigatórias. `id` segue `^[a-z][a-z0-9_-]{0,39}$` e é o
    `rule_id` da sugestão; `when` é a condição estruturada (`$defs/condition`;
    o mapa simples não vale); `color` é uma das 5 cores; `criterion` (1 a 120
    caracteres) é a frase curta que diz à enfermagem por que a cor foi sugerida.
  - A variante não tem `risk_rules`, `start_step_id`, `steps`, `scoring`,
    `recommendations`, `priority_when`, `offer`, `suggestions` nem `scheduling`.
    Parte do conteúdo assinado (ADR 0016).
- Roteamento: com `kind: "urgency_classification"`, `$defs/urgency_classification`;
  com outro `kind`, `$defs/screening`, como na `v1.6.0`; sem `kind`, triagem. O
  `const` de `kind` foi da raiz para o acolhimento, então um `kind` desconhecido
  continua dando só `/kind const`. Triagem e acolhimento recusam `levels` e
  `classification_rules`. A escala do acolhimento (`risk_rules`, 4 cores do CAB 28)
  não muda.
- O schema não confere as variáveis do `when` (`vitals.*`, `pain_scale`,
  `profile.*`), a unicidade de `id`, a ordem crescente dos tempos, qual regra
  vence quando várias casam, nem "um protocolo ativo de urgência por cidade".
  Quem confere é o gate e o motor do `api`.
- Nenhum conteúdo licenciado (fluxogramas do Manchester) entra nos exemplos: os
  critérios são genéricos e só provam o formato.
- `examples/`: 38 exemplos novos (4 válidos, 34 inválidos) no mesmo manifesto.
- Expand: nada foi removido e nada vira obrigatório para triagem ou acolhimento.
  Toda definição válida em `v1.6.0` continua válida, com o mesmo conjunto de
  erros quando inválida, por isso MINOR. O `api` atualiza a cópia
  (`config/protocols/schema.json`) no mesmo ciclo, antes de o `dashboard` passar a
  gravar protocolos de urgência.
```

- [ ] **Step 8: README da raiz**

Em `README.md`:
- na tabela, troque `protocols-v1.6.0` por `protocols-v1.7.0`;
- em `### protocols`, depois da frase da `v1.6.0` ("... as regras que sugerem a cor da escuta inicial."), acrescente: `` `v1.7.0` acrescentou a classificação de risco da UPA (`kind: "urgency_classification"`, ADR 0050): as 5 cores com o tempo máximo de cada uma (`levels`) e as regras que sugerem a cor com o critério (`classification_rules`).``

Confira com `grep -n "v1.7.0" .worktrees/contracts-mod32a/README.md`. Expected: duas linhas.

- [ ] **Step 9: Commit**

```bash
cd .worktrees/contracts-mod32a
/opt/homebrew/bin/git add protocols/schema.json protocols/CHANGELOG.md protocols/examples/manifest.json README.md \
  protocols/examples/urgency-*.json protocols/examples/invalid-urgency-*.json \
  protocols/examples/invalid-screening-with-levels.json protocols/examples/invalid-screening-with-classification-rules.json \
  protocols/examples/invalid-triage-with-levels.json protocols/examples/invalid-triage-with-classification-rules.json
/opt/homebrew/bin/git status --short
/opt/homebrew/bin/git commit -m "feat: add urgency classification variant to the protocols schema (protocols-v1.7.0)

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

Expected: antes do commit, `status --short` lista só os 4 arquivos modificados e os 38 exemplos novos.

---

### Task 2: Entrega — verificação final e parada antes do merge

**Files:** nenhum novo. Só leitura e, com autorização, merge, tag e push.

**Interfaces:**
- Consumes: a branch `feat/protocols-urgency` com o commit da Task 1.
- Produces: o hash do commit, para o plano do `api` citar. Depois de autorizado, a tag `protocols-v1.7.0` em `origin`, que é pré-requisito do plano do `api` (contrato §13).

- [ ] **Step 1: Verificação final da branch**

```bash
G=/opt/homebrew/bin/git; W=.worktrees/contracts-mod32a
$G -C $W log --oneline origin/main..HEAD
$G -C $W diff --stat origin/main..HEAD | tail -1
$G -C $W diff origin/main..HEAD -- protocols/schema.json | grep '^-' | grep -v '^---'
```

Expected:
- um commit;
- `42 files changed`;
- as linhas removidas do schema são só quatro: o `"then"` de uma linha, o `"const": "screening",` da raiz, a descrição antiga de `kind` e o `"properties": { "risk_rules": false }` da triagem.

Rode de novo o Step 6 da Task 1. Expected: `107/107 casos`.

- [ ] **Step 2: Pare e peça autorização**

Reporte ao usuário o hash, `107/107`, as Divergências C1–C6 e o que falta. **Não execute nada abaixo sem um "sim" explícito do usuário para cada etapa:** (a) merge na `main` local, (b) tag, (c) push.

- [ ] **Step 3 (só com autorização): Merge fast-forward**

```bash
G=/opt/homebrew/bin/git
$G -C contracts fetch origin && $G -C contracts status -sb
$G -C contracts checkout main && $G -C contracts merge --ff-only feat/protocols-urgency
```

Expected: `## main...origin/main` sem divergência, e fast-forward de um commit.

Se não for fast-forward, pare e reporte. Depois, `$G -C .worktrees/contracts-mod32a rebase origin/main`. O conflito só pode aparecer na tabela do `README.md` ou no topo do `protocols/CHANGELOG.md`: mantenha as duas entradas, a mais nova primeiro. Repita a verificação (`107/107` ou mais).

- [ ] **Step 4 (só com autorização): Tag anotada**

```bash
/opt/homebrew/bin/git -C contracts tag -a protocols-v1.7.0 -m "protocols-v1.7.0 — MINOR"
/opt/homebrew/bin/git -C contracts show protocols-v1.7.0 --format='%s' -s | head -3
```

Expected: `protocols-v1.7.0 — MINOR` e o assunto do commit da Task 1.

- [ ] **Step 5 (só com autorização): Push**

```bash
G=/opt/homebrew/bin/git
$G -C contracts log --oneline origin/main..main
$G -C contracts push origin main && $G -C contracts push origin protocols-v1.7.0
```

Expected: `origin/main..main` só com o commit desta entrega (outra sessão pode ter commits locais: publique só os seus).

- [ ] **Step 6: Limpe e confira a cópia do api**

Depois do merge, rode `$G -C contracts worktree remove ../.worktrees/contracts-mod32a && $G -C contracts branch -d feat/protocols-urgency`.

Depois da task de cópia do plano do `api`, rode `diff contracts/protocols/schema.json apps/api/.claude/mod32a/config/protocols/schema.json && echo identicos`. Expected: `identicos`. O card F-32.1 só vai para Done quando api, dashboard e maintenance também terminarem.

---

## Divergências propostas ao contrato

1. **C1 — Nomes dos blocos.** O §3 diz "`levels` … e as regras", sem nome para as regras. Proposta: `classification_rules`, separado de `risk_rules`. Assim a escala de 4 cores do acolhimento fica intacta e um acolhimento convertido não passa como urgência. Também proposta: `levels` como **lista** ordenada (`prefixItems`), não como mapa por cor. A ordem é a prioridade da fila, e o schema recusa cor faltando, repetida ou fora de ordem.
2. **C2 — `id` da regra.** O §3 diz que `Urgency::Suggest` devolve `rule_id`, mas a regra não tem id. Proposta: `id` obrigatório (`^[a-z][a-z0-9_-]{0,39}$`), que é o `rule_id` gravado na classificação. A unicidade dentro do protocolo fica com o gate do `api`, porque o JSON Schema não expressa unicidade de campo. Alternativa: o índice da regra. É mais simples, mas muda de sentido entre versões e quebra o painel do gestor por regra.
3. **C3 — Limites dos tempos.** O §1 dá os padrões e não os limites. Proposta: inteiro de 0 a 1440 no schema. O gate do `api` recusa tempo de cor menos grave menor que o de cor mais grave (por exemplo, `yellow` 30 e `orange` 60), com o erro `levels_out_of_order`. O schema não consegue comparar itens.
4. **C4 — Qual regra vence e quais variáveis valem.** O §3 não diz. Proposta para o motor do `api`:
   - vale a cor mais grave entre as regras que casarem; no empate, a primeira da lista, com o `criterion` e o `id` dela;
   - variáveis do `when`: `vitals.*` com os nomes de `Screenings::VitalSigns`, `pain_scale` (0..10) e `profile.age`/`profile.sex`;
   - a queixa em texto livre (cifrada) **não** entra em condição. O §3 diz que a sugestão "recebe a queixa": proposta de que ela só fica gravada, e que uma queixa codificada (`complaint.ciap2`, como no 18) seja decisão do plano do `api`.
5. **C5 — Tamanhos.** Proposta: até 100 regras (o acolhimento tem 50; a classificação da UPA tem mais critérios) e `criterion` de até 120 caracteres, para caber na tela da enfermagem e no tablet.
6. **C6 — Nome e unicidade.** O acolhimento reservou o nome `acolhimento`. Para a urgência, proposta: nenhum nome reservado (o `name` segue o `pattern` comum). "Um protocolo ativo de urgência por cidade" (§3) é conferido pelo `kind`, no gate do `api`.

## Self-review

- **Cobertura:**
  - contrato §3, `kind` novo com `levels` (5 cores e `max_wait_minutes`): Task 1, Steps 5(c) e 5(f), Review Focus 3 e 4;
  - regras no formato de condição do 18 com cor e `criterion`: Step 5(c), Review Focus 5, C1, C2;
  - exemplos válidos e inválidos: Steps 2 e 3; CHANGELOG e README: Steps 7 e 8;
  - tag `protocols-v1.7.0` só com autorização: Task 2;
  - §1 "Cores" (ordem e padrões): `levels` e `urgency-default.json`;
  - spec decisão 1 (sem conteúdo licenciado): Global Constraints e CHANGELOG;
  - R5 (fora de `ActiveProtocol`): é do `api`, e o roteamento por `kind` permite filtrar.
- **Placeholders:** nenhum. Valores lidos na execução: só os ids dos cards. Os números (`80/107`, `107/107`, `42 files`, a diferença de ordem num caso) foram conferidos no container em 2026-11-05.
- **Consistência:** os nomes da tabela do Step 3 são os do gerador do Step 2; o comando é o de `protocols/README.md`; a tag segue o formato das anteriores; branch e worktree são os do brief.
- **Review Focus:** os cinco itens têm caso no manifesto (Task 1), e o item 1 também a comparação linha a linha do Step 6.
- **Proporção:** o código é só o que é contrato (fixtures e trechos exatos do schema); o resto são comandos e saídas esperadas.
