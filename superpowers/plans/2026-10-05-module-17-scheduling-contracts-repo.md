# Módulo 17 — `scheduling` no schema de protocolo (repo contracts) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Publicar no repo `contracts` o bloco opcional `scheduling` da definição de protocolo — regras `{ when, appointment_type, priority, due_in_days }` que fazem a conclusão da triagem gerar um pedido de agendamento — como `protocols-v1.5.0` (MINOR), com exemplos válidos e inválidos versionados, fonte da cópia que o `api` valida (F-17.5; ADR 0029).

**Architecture:** Uma edição em `protocols/schema.json` (propriedade nova na raiz, depois de `suggestions`), exemplos de definição em `protocols/examples/` acrescentados ao `manifest.json` que já existe desde `protocols-v1.4.0` (`valid` ou `<data_pointer> <type>` do `json_schemer`), entrada no `protocols/CHANGELOG.md` e versão no `README.md` da raiz. O repo não tem código executável nem suíte: a prova é validar os exemplos com o mesmo `json_schemer` 2.5 do `api`, dentro do container, recebendo schema e exemplos pela entrada padrão (o container só monta `apps/api`) — o comando é o de `protocols/README.md`. Merge, tag anotada `protocols-v1.5.0` e push só com autorização.

**Tech Stack:** JSON Schema draft 2020-12; `json_schemer` 2.5.0 (do bundle do `api`, só para a verificação); `python3` do host para gerar os exemplos e empacotar schema + exemplos num JSON.

**Spec:** `docs/.claude/ciclo2/superpowers/specs/2026-10-05-module-17-scheduling-design.md` §5.1 e §11; ADR `docs/.claude/ciclo2/adr/0029.md`. Contratos entre apps (fonte única do formato): `docs/.claude/ciclo2/superpowers/plans/2026-10-05-module-17-scheduling-contracts.md` §1 e §7. **Este plano roda antes do plano do `api`**, que copia o `schema.json` final byte a byte para `apps/api/config/protocols/schema.json`.

## Global Constraints

- SemVer por domínio (ADR 0015): só acréscimo opcional → **MINOR**, `protocols-v1.4.0` → `protocols-v1.5.0`. Nada vira obrigatório, nada é removido: toda definição válida em `v1.4.0` continua válida.
- Formato exato do contrato §1, sem acréscimo: `scheduling` = `{ "type": "array", "maxItems": 10, "items": { "type": "object", "additionalProperties": false, "required": ["when", "appointment_type", "priority", "due_in_days"], "properties": { "when": { "$ref": "#/$defs/condition" }, "appointment_type": { "type": "string", "pattern": "^[a-z][a-z0-9_]{1,40}$" }, "priority": { "enum": ["routine", "priority"] }, "due_in_days": { "type": "integer", "minimum": 1, "maximum": 365 } } } }`. Sem `minItems`: `scheduling: []` é válido e equivale a não ter o bloco ("só orientação", ADR 0029).
- O `pattern` de `appointment_type` é o mesmo da `key` de `appointment_types` da cidade (spec §3.1): sublinhado, nunca hífen (o hífen é do `name` do protocolo).
- O schema **não** confere variáveis do `when` (`outcome.*`, `profile.*`, ids de passo) nem se o tipo existe na cidade: isso é do gate do `api` (contrato §1 — tipo inexistente é aviso, não bloqueio). Nenhum `pattern` de variável e nenhuma lista de tipos entram aqui.
- `when` é **só** a condição estruturada (`$ref #/$defs/condition`), como em `offer.eligibility` e `suggestions[].when`; a forma antiga de mapa simples (`{ "tier": "alta" }`), aceita em `priority_when` e nas regras de `decision_table`, é recusada aqui.
- CHANGELOG obrigatório: sem entrada em `protocols/CHANGELOG.md`, não há tag.
- Tag **anotada**, mensagem `protocols-v1.5.0 — MINOR` (mesmo formato de `protocols-v1.4.0`), criada no commit que chega à `main`.
- O `api` usa uma **cópia** (`apps/api/config/protocols/schema.json`, hoje idêntica a `contracts/protocols/schema.json` v1.4.0 — conferido em 2026-10-05 com `diff`): os dois arquivos ficam byte a byte iguais no mesmo ciclo. O `dashboard` só passa a gravar `scheduling` depois que a cópia do `api` for trocada (senão o `api` recusa o rascunho com `/scheduling schema`).
- Tipos TypeScript e design tokens: **nada a atualizar aqui** (`contracts/types/` e `contracts/design-tokens/` são scaffold; o editor do dashboard trata a definição como `unknown`).
- Eventos do módulo (contrato §6) são da cidade (tenant-scoped) e ficam fora de `events/EVENTS.md`, como no módulo 15: este plano não abre `events-v2.2.0`.
- Worktree `contracts/.claude/mod17`, branch `feat/protocols-scheduling` (`contracts/.gitignore` já ignora `/.claude/`). Todos os comandos rodam a partir da raiz do monorepo `/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, com o compose de pé (`docker compose ps` mostra `api running`).
- Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Nunca `git add -A`: caminhos explícitos. Commits em inglês, Conventional Commits com tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`.
- Nada de merge na `main`, tag ou push sem autorização explícita do usuário, uma etapa de cada vez.
- Módulo 16 em paralelo: o plano `contracts` dele mexe só em `session/` (já publicado como `session-v1.1.0`, `401639c`); este plano não toca `session/`. Se a `main` do `contracts` andar entre o Step 1 da Task 1 e o merge, rebase da branch e nova verificação (Task 2, Step 3).

## Review Focus

1. **`appointment_type` com hífen (`"consulta-medica"`)** — o autor acostumado ao `name` do protocolo escreve hífen; a `key` do tipo usa sublinhado e o pedido nunca casaria com um tipo. O schema recusa. Teste: `invalid-scheduling-type-hyphen.json` (Task 1), esperado `/scheduling/0/appointment_type pattern`.
2. **`when` na forma antiga de mapa simples (`{ "tier": "alta" }`)** — aceita em `priority_when`, mas aqui o gate do `api` avalia só a condição estruturada; aceitar no schema deixaria uma regra que nunca casa. Recusada. Teste: `invalid-scheduling-when-simple-map.json` (Task 1), esperado `/scheduling/0/when required`.
3. **`due_in_days` como texto (`"30"`) ou fracionário (`1.5`)** — o editor monta o número a partir de um campo de texto; o schema exige inteiro. Testes: `invalid-scheduling-due-string.json` e `invalid-scheduling-due-fraction.json` (Task 1), esperado `/scheduling/0/due_in_days integer`.
4. **`scheduling: []`** (o editor grava a lista antes de criar a primeira regra) — válido, equivale a "só orientação". Teste: `scheduling-empty.json` (Task 1), esperado `valid`.
5. **Definição publicada sem `scheduling`, e a que tem `offer`/`suggestions`/`scheduling` juntos** — continuam válidas (toda versão ativa nas cidades hoje não tem o bloco). Testes: os 19 casos de `v1.4.0` no manifesto, o template real `apps/api/config/city_templates/triage_respiratoria.json` como caso extra em toda execução e `scheduling-with-offer-and-suggestions.json` (Task 1).

## Mapa de arquivos

| Arquivo (repo `contracts`) | Mudança | Task |
|---|---|---|
| `protocols/schema.json` | `properties.scheduling` depois de `suggestions` | 1 |
| `protocols/examples/*.json` | 22 exemplos novos (5 válidos, 17 inválidos) | 1 |
| `protocols/examples/manifest.json` | 22 casos novos (41 no total) | 1 |
| `protocols/CHANGELOG.md` | entrada `protocols-v1.5.0` | 1 |
| `README.md` | versão na tabela e parágrafo de `### protocols` | 1 |

---

### Task 1: `scheduling` na raiz, exemplos, CHANGELOG e README (`protocols-v1.5.0`)

**Files (repo `contracts`, worktree `contracts/.claude/mod17`):**
- Create: `protocols/examples/scheduling-full.json`, `scheduling-empty.json`, `scheduling-due-bounds.json`, `scheduling-city-type.json`, `scheduling-with-offer-and-suggestions.json`
- Create: `protocols/examples/invalid-scheduling-without-when.json`, `invalid-scheduling-without-type.json`, `invalid-scheduling-without-priority.json`, `invalid-scheduling-without-due.json`, `invalid-scheduling-type-hyphen.json`, `invalid-scheduling-type-uppercase.json`, `invalid-scheduling-type-one-char.json`, `invalid-scheduling-type-too-long.json`, `invalid-scheduling-priority-urgent.json`, `invalid-scheduling-due-zero.json`, `invalid-scheduling-due-too-big.json`, `invalid-scheduling-due-string.json`, `invalid-scheduling-due-fraction.json`, `invalid-scheduling-extra-property.json`, `invalid-scheduling-when-simple-map.json`, `invalid-scheduling-not-array.json`, `invalid-scheduling-too-many.json`
- Modify: `protocols/examples/manifest.json`
- Modify: `protocols/schema.json` (fim de `properties`, depois de `suggestions`)
- Modify: `protocols/CHANGELOG.md`
- Modify: `README.md`

**Interfaces:**
- Consumes: `$defs/condition` (com `gte`/`lte`, `protocols-v1.4.0`), o manifesto e o comando de verificação de `protocols/README.md`.
- Produces: `properties.scheduling` exatamente como no contrato §1; `protocols-v1.5.0` no CHANGELOG e no README. Consumidos pelo plano do `api` (cópia literal em `config/protocols/schema.json`; `Triages::Schedule` lê `definition["scheduling"]`; gate de variáveis e de tipo inexistente) e, via `api`, pelo painel "Agendamento" do editor de protocolo no `dashboard`. Formato do manifesto inalterado: `{ "cases": [ { "file": "<nome>.json", "expect": "valid" | "<data_pointer> <type>" } ] }`.

- [ ] **Step 1: Crie o worktree**

```bash
/opt/homebrew/bin/git -C contracts fetch origin
/opt/homebrew/bin/git -C contracts log --oneline -1 origin/main
/opt/homebrew/bin/git -C contracts worktree add .claude/mod17 -b feat/protocols-scheduling origin/main
```
Expected: `origin/main` em `401639c feat: add enabled feature keys to the session contract (session-v1.1.0)` (ou mais novo — se houver commit novo em `protocols/`, pare e reporte); worktree criado. Confira também que a cópia do `api` está igual à base: `diff contracts/.claude/mod17/protocols/schema.json apps/api/config/protocols/schema.json && echo identicos` → `identicos`.

- [ ] **Step 2: Escreva os exemplos**

A partir da raiz do monorepo (o `base()` é o mesmo protocolo dos exemplos de `v1.4.0`):

```bash
cd contracts/.claude/mod17/protocols/examples && python3 - <<'EOF'
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

def rule(**over):
    r = {"when": {"eq": ["outcome.tier", "alta"]}, "appointment_type": "consulta_medica",
         "priority": "priority", "due_in_days": 7}
    r.update(over)
    return r

def without(key):
    r = rule()
    del r[key]
    return r

files = {
    "scheduling-full.json": base(scheduling=[
        rule(),
        {"when": {"all": [{"gte": ["profile.age", 60]}, {"gte": ["medicamentos", 5]}]},
         "appointment_type": "consulta_enfermagem", "priority": "routine", "due_in_days": 30}]),
    "scheduling-empty.json": base(scheduling=[]),
    "scheduling-due-bounds.json": base(scheduling=[rule(due_in_days=1), rule(due_in_days=365, priority="routine")]),
    "scheduling-city-type.json": base(scheduling=[rule(appointment_type="grupo_hipertensos_2")]),
    "scheduling-with-offer-and-suggestions.json": base(
        offer={"title": "Saúde do idoso", "eligibility": {"gte": ["profile.age", 60]}, "retake_after_days": 365},
        suggestions=[{"protocol": "saude-mental-aprofundada", "when": {"gte": ["outcome.score", 15]}}],
        scheduling=[rule(priority="routine", due_in_days=30)]),
    "invalid-scheduling-without-when.json": base(scheduling=[without("when")]),
    "invalid-scheduling-without-type.json": base(scheduling=[without("appointment_type")]),
    "invalid-scheduling-without-priority.json": base(scheduling=[without("priority")]),
    "invalid-scheduling-without-due.json": base(scheduling=[without("due_in_days")]),
    "invalid-scheduling-type-hyphen.json": base(scheduling=[rule(appointment_type="consulta-medica")]),
    "invalid-scheduling-type-uppercase.json": base(scheduling=[rule(appointment_type="Consulta_Medica")]),
    "invalid-scheduling-type-one-char.json": base(scheduling=[rule(appointment_type="c")]),
    "invalid-scheduling-type-too-long.json": base(scheduling=[rule(appointment_type="c" + "x" * 41)]),
    "invalid-scheduling-priority-urgent.json": base(scheduling=[rule(priority="urgent")]),
    "invalid-scheduling-due-zero.json": base(scheduling=[rule(due_in_days=0)]),
    "invalid-scheduling-due-too-big.json": base(scheduling=[rule(due_in_days=366)]),
    "invalid-scheduling-due-string.json": base(scheduling=[rule(due_in_days="30")]),
    "invalid-scheduling-due-fraction.json": base(scheduling=[rule(due_in_days=1.5)]),
    "invalid-scheduling-extra-property.json": base(scheduling=[rule(unit_id=12)]),
    "invalid-scheduling-when-simple-map.json": base(scheduling=[rule(when={"tier": "alta"})]),
    "invalid-scheduling-not-array.json": base(scheduling=rule()),
    "invalid-scheduling-too-many.json": base(scheduling=[rule(due_in_days=i + 1) for i in range(11)]),
}
for name, doc in files.items():
    with open(name, "w", encoding="utf-8") as f:
        f.write(json.dumps(doc, indent=2, ensure_ascii=False) + "\n")
print(len(files))
EOF
```
Expected: `22`. (`"c" + "x" * 41` tem 42 caracteres: o `pattern` aceita de 2 a 41.)

- [ ] **Step 3: Acrescente os casos ao manifesto**

Em `contracts/.claude/mod17/protocols/examples/manifest.json`, troque a última linha de caso

```json
    { "file": "invalid-suggestions-too-many.json", "expect": "/suggestions maxItems" }
  ]
}
```

por

```json
    { "file": "invalid-suggestions-too-many.json", "expect": "/suggestions maxItems" },
    { "file": "scheduling-full.json", "expect": "valid" },
    { "file": "scheduling-empty.json", "expect": "valid" },
    { "file": "scheduling-due-bounds.json", "expect": "valid" },
    { "file": "scheduling-city-type.json", "expect": "valid" },
    { "file": "scheduling-with-offer-and-suggestions.json", "expect": "valid" },
    { "file": "invalid-scheduling-without-when.json", "expect": "/scheduling/0 required" },
    { "file": "invalid-scheduling-without-type.json", "expect": "/scheduling/0 required" },
    { "file": "invalid-scheduling-without-priority.json", "expect": "/scheduling/0 required" },
    { "file": "invalid-scheduling-without-due.json", "expect": "/scheduling/0 required" },
    { "file": "invalid-scheduling-type-hyphen.json", "expect": "/scheduling/0/appointment_type pattern" },
    { "file": "invalid-scheduling-type-uppercase.json", "expect": "/scheduling/0/appointment_type pattern" },
    { "file": "invalid-scheduling-type-one-char.json", "expect": "/scheduling/0/appointment_type pattern" },
    { "file": "invalid-scheduling-type-too-long.json", "expect": "/scheduling/0/appointment_type pattern" },
    { "file": "invalid-scheduling-priority-urgent.json", "expect": "/scheduling/0/priority enum" },
    { "file": "invalid-scheduling-due-zero.json", "expect": "/scheduling/0/due_in_days minimum" },
    { "file": "invalid-scheduling-due-too-big.json", "expect": "/scheduling/0/due_in_days maximum" },
    { "file": "invalid-scheduling-due-string.json", "expect": "/scheduling/0/due_in_days integer" },
    { "file": "invalid-scheduling-due-fraction.json", "expect": "/scheduling/0/due_in_days integer" },
    { "file": "invalid-scheduling-extra-property.json", "expect": "/scheduling/0/unit_id schema" },
    { "file": "invalid-scheduling-when-simple-map.json", "expect": "/scheduling/0/when required" },
    { "file": "invalid-scheduling-not-array.json", "expect": "/scheduling array" },
    { "file": "invalid-scheduling-too-many.json", "expect": "/scheduling maxItems" }
  ]
}
```

Confira: `python3 -c 'import json;print(len(json.load(open("contracts/.claude/mod17/protocols/examples/manifest.json"))["cases"]))'` → `41`.

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
' contracts/.claude/mod17/protocols/schema.json contracts/.claude/mod17/protocols/examples/manifest.json apps/api/config/city_templates/triage_respiratoria.json \
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

Expected (exit 1): os 19 casos de `v1.4.0` e o template `ok`; os 22 novos `FAIL` com `obtido: /scheduling schema` (propriedade desconhecida na raiz, `additionalProperties: false`); última linha `20/42 casos`.

- [ ] **Step 5: Acrescente `scheduling` a `properties`**

Em `contracts/.claude/mod17/protocols/schema.json`, troque o fim de `suggestions` e o começo de `$defs`:

```json
          "when":     { "$ref": "#/$defs/condition" }
        }
      }
    }
  },
  "$defs": {
```

por

```json
          "when":     { "$ref": "#/$defs/condition" }
        }
      }
    },
    "scheduling": {
      "type": "array",
      "maxItems": 10,
      "items": {
        "type": "object",
        "additionalProperties": false,
        "required": ["when", "appointment_type", "priority", "due_in_days"],
        "properties": {
          "when":             { "$ref": "#/$defs/condition" },
          "appointment_type": { "type": "string", "pattern": "^[a-z][a-z0-9_]{1,40}$" },
          "priority":         { "enum": ["routine", "priority"] },
          "due_in_days":      { "type": "integer", "minimum": 1, "maximum": 365 }
        }
      }
    }
  },
  "$defs": {
```

(O trecho "antes" é único: é o único `"when"` com essa indentação seguido de três fechamentos e de `"$defs"`.)

- [ ] **Step 6: Rode a verificação e veja passar**

Run: o mesmo comando do Step 4; depois `python3 -m json.tool contracts/.claude/mod17/protocols/schema.json > /dev/null && echo json-ok`
Expected: 42 linhas `ok`, `42/42 casos`, exit 0; e `json-ok`. (Conferido em 2026-10-05 contra o `json_schemer` 2.5.0 do container, com a `v1.4.0` atual e com o bloco acima: `20/42` antes e `42/42` depois; `invalid-scheduling-when-simple-map.json` dá `/scheduling/0/when/tier schema | /scheduling/0/when required`.)

- [ ] **Step 7: CHANGELOG**

Em `contracts/.claude/mod17/protocols/CHANGELOG.md`, logo abaixo de `# Changelog — protocols` (antes da entrada `protocols-v1.4.0`), insira:

```markdown

## protocols-v1.5.0 — 2026-10-05 — MINOR
- `scheduling` (opcional, na raiz, até 10 regras): o que a conclusão da triagem
  faz com a agenda (ADR 0029). Cada regra é `{ when, appointment_type, priority,
  due_in_days }`, todos obrigatórios: `when` é a condição estruturada
  (`$defs/condition`; a forma antiga de mapa simples não vale aqui),
  `appointment_type` é a `key` do tipo de atendimento da cidade
  (`^[a-z][a-z0-9_]{1,40}$`, sublinhado, nunca hífen), `priority` é `routine` ou
  `priority`, e `due_in_days` (1..365) é o prazo previsto do pedido. Vale a
  primeira regra que casar; nenhuma regra (ou `scheduling: []`) = só orientação.
  Resultado urgente nunca gera pedido (regra do `api`, não do schema). Parte do
  conteúdo assinado (ADR 0016).
- O schema não confere as variáveis do `when` (`outcome.*`, `profile.*`, ids de
  passo) nem se o tipo existe na cidade: quem confere é o gate do `api` — tipo
  inexistente é aviso, não bloqueio.
- `examples/`: 22 exemplos novos (5 válidos, 17 inválidos) no mesmo manifesto.
- Expand: nada vira obrigatório e nada foi removido — toda definição válida em
  `v1.4.0` continua válida, por isso MINOR. O `api` atualiza a cópia
  (`config/protocols/schema.json`) no mesmo ciclo, antes do `dashboard` passar a
  gravar `scheduling`.
```

- [ ] **Step 8: README da raiz**

Em `contracts/.claude/mod17/README.md`:
- na tabela "Domínios", `` `protocols-v1.4.0` `` passa a `` `protocols-v1.5.0` ``;
- no parágrafo de `### protocols`, o trecho

```markdown
catálogo), `suggestions` e os operadores `gte`/`lte` (ADR 0027). Exemplos
válidos e inválidos ficam em `protocols/examples/`.
```

passa a

```markdown
catálogo), `suggestions` e os operadores `gte`/`lte` (ADR 0027). `v1.5.0`
acrescentou `scheduling`, as regras que geram pedido de agendamento na
conclusão da triagem (ADR 0029). Exemplos válidos e inválidos ficam em
`protocols/examples/`.
```

Confira: `grep -n 'protocols-v1' contracts/.claude/mod17/README.md` mostra só `protocols-v1.5.0` na tabela (e o `protocols-vX.Y.Z` genérico da seção de versionamento).

- [ ] **Step 9: Commit**

```bash
/opt/homebrew/bin/git -C contracts/.claude/mod17 add protocols/schema.json protocols/CHANGELOG.md protocols/examples README.md
/opt/homebrew/bin/git -C contracts/.claude/mod17 status --short
/opt/homebrew/bin/git -C contracts/.claude/mod17 commit -m "feat: add scheduling rules to the protocols schema (protocols-v1.5.0)

Optional scheduling list at the root: { when, appointment_type, priority,
due_in_days } rules that let a completed triage open an appointment request
(ADR 0029). Variables in when and unknown appointment types stay with the api
gate. MINOR per ADR 0015: nothing removed or made required, every v1.4.0
definition stays valid.

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```
Expected: `status --short` lista `protocols/schema.json`, `protocols/CHANGELOG.md`, `README.md`, `protocols/examples/manifest.json` e os 22 exemplos novos — nada mais.

### Task 2: Entrega — verificação final e parada antes do merge

**Files:** nenhum novo. Só leitura e, com autorização, merge/tag/push.

**Interfaces:**
- Consumes: a branch `feat/protocols-scheduling` com o commit da Task 1.
- Produces: o hash do commit da Task 1 para o plano do `api` citar; depois de autorizado, a tag `protocols-v1.5.0` em `origin` — pré-requisito do plano do `api` (contrato §7, passo 1).

- [ ] **Step 1: Verificação final da branch**

```bash
/opt/homebrew/bin/git -C contracts/.claude/mod17 log --oneline origin/main..HEAD
/opt/homebrew/bin/git -C contracts/.claude/mod17 diff --stat origin/main..HEAD
/opt/homebrew/bin/git -C contracts/.claude/mod17 diff origin/main..HEAD -- protocols/schema.json | grep '^-' | grep -v '^---'
```
Expected: um commit (`feat: add scheduling rules to the protocols schema (protocols-v1.5.0)`); `diff --stat` só em `README.md`, `protocols/CHANGELOG.md`, `protocols/schema.json` e `protocols/examples/` (23 arquivos: manifesto + 22 exemplos); o `grep` de linhas removidas do schema não mostra nada ou, no máximo, a linha `    }` que vira `    },` (o alinhamento do diff varia; nenhuma propriedade removida — MINOR). Rode de novo o comando de verificação da Task 1, Step 4: `42/42 casos`.

- [ ] **Step 2: Pare e peça autorização**

Reporte ao usuário: hash do commit, `42/42 casos`, e a lista do que falta. **Não execute nada abaixo sem um "sim" explícito do usuário para cada etapa** (a) merge na `main` local, (b) tag, (c) push. Antes do push, confira `origin/main..main`: outra sessão (módulo 16) pode ter commits locais no `contracts`; publique só os desta entrega.

- [ ] **Step 3 (só com autorização): Merge fast-forward na main**

```bash
/opt/homebrew/bin/git -C contracts fetch origin
/opt/homebrew/bin/git -C contracts status -sb
/opt/homebrew/bin/git -C contracts checkout main
/opt/homebrew/bin/git -C contracts merge --ff-only feat/protocols-scheduling
```
Expected: `## main...origin/main` sem divergência antes; fast-forward de um commit. Se não for fast-forward (outra sessão publicou em `contracts` desde a Task 1), pare e reporte: `git -C contracts/.claude/mod17 rebase origin/main`, conflito só pode aparecer em `README.md` (tabela de versões) ou `protocols/CHANGELOG.md` (entrada nova no topo — mantenha as duas, a mais nova primeiro), e repita a verificação (`42/42` ou mais) antes de tentar de novo.

- [ ] **Step 4 (só com autorização): Tag anotada**

```bash
/opt/homebrew/bin/git -C contracts tag -a protocols-v1.5.0 -m "protocols-v1.5.0 — MINOR"
/opt/homebrew/bin/git -C contracts show protocols-v1.5.0 --stat --format='%an %s' | head -12
```
Expected: `tag protocols-v1.5.0`, `protocols-v1.5.0 — MINOR`, e o commit `feat: add scheduling rules to the protocols schema (protocols-v1.5.0)`.

- [ ] **Step 5 (só com autorização): Push da main e da tag**

```bash
/opt/homebrew/bin/git -C contracts log --oneline origin/main..main
/opt/homebrew/bin/git -C contracts push origin main
/opt/homebrew/bin/git -C contracts push origin protocols-v1.5.0
```
Expected: `origin/main..main` mostra só o commit desta entrega; o push publica a `main` e a tag.

- [ ] **Step 6: Limpe o worktree (depois do merge)**

```bash
/opt/homebrew/bin/git -C contracts worktree remove .claude/mod17
/opt/homebrew/bin/git -C contracts branch -d feat/protocols-scheduling
```

- [ ] **Step 7: Confira a cópia do api (depois da tarefa de schema do plano do api)**

Run: `diff contracts/protocols/schema.json apps/api/.claude/mod17/config/protocols/schema.json && echo identicos`
Expected: `identicos`. (Se o worktree do `api` tiver outro caminho, use o `config/protocols/schema.json` dele.)

---

## Divergências propostas ao contrato

Nenhuma no formato: o bloco é o do contrato §1, byte a byte nas restrições. Duas observações, sem pedir mudança:

1. **Forma antiga de `when` recusada.** O contrato escreve `"when": { "$ref": "#/$defs/condition" }`, o que já exclui o mapa simples aceito em `priority_when`; este plano só deixa isso explícito no CHANGELOG e prova com `invalid-scheduling-when-simple-map.json`. O painel "Agendamento" do dashboard deve gravar sempre a condição estruturada (o construtor do módulo 15 já faz isso).
2. **Eventos (§6) fora do `contracts`.** `appointment.*`, `appointment_request.*`, `appointment_type.changed` e `schedule_template.changed` são da cidade; como no módulo 15, `events/EVENTS.md` não é atualizado. Se o usuário quiser o catálogo em dia, é uma entrega própria (`events-v2.2.0`).

## Self-review (feito ao escrever o plano)

- **Cobertura da spec:** §5.1 (até 10 regras; `when` com `outcome.*`, `profile.*` e ids de passo; `appointment_type`, `priority`, `due_in_days` 1..365; gate avisa tipo inexistente) → Task 1 (schema, exemplos `scheduling-full` com `outcome.tier`, `profile.age` e o passo `medicamentos`, CHANGELOG com o papel do gate); §11.1 e contrato §7.1 (`protocols-v1.5.0`, push com autorização) → Task 2.
- **Fixtures:** válidos — duas regras com variáveis dos três lugares, lista vazia, limites 1 e 365, tipo criado pela cidade (`grupo_hipertensos_2`), o bloco junto de `offer`/`suggestions`; inválidos — cada um dos quatro obrigatórios ausente, tipo com hífen, maiúscula, um caractere e 42 caracteres, prioridade `urgent`, prazo 0, 366, texto e fracionário, propriedade extra, `when` de mapa simples, objeto no lugar da lista e 11 regras.
- **Placeholders:** nenhum; schema, exemplos (script exato), manifesto, comando, CHANGELOG, README e mensagem de commit estão completos. As saídas esperadas (ponteiros e tipos de erro, `20/42` antes e `42/42` depois) foram conferidas contra o `json_schemer` 2.5.0 do container em 2026-10-05, numa cópia de rascunho do schema.
- **Consistência:** nomes do manifesto = nomes gerados pelo script; o comando de verificação é o de `protocols/README.md`; a mensagem da tag segue `protocols-v1.4.0 — MINOR`; branch e worktree são os pedidos (`feat/protocols-scheduling`, `contracts/.claude/mod17`).
- **Review Focus:** os cinco itens têm exemplo no manifesto (Task 1).
