# Módulo 14 — `analytic` no schema de protocolo (repo contracts) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Publicar no repo `contracts` a propriedade opcional `analytic` da pergunta do protocolo — só válida com `answer_type` `boolean` ou `enum` — como `protocols-v1.2.0` (MINOR), fonte da cópia que o `api` valida (F-14.6, ADR 0025).

**Architecture:** Uma edição em `protocols/schema.json` (`$defs/step`: propriedade `analytic` + `if`/`then` que restringe `answer_type` quando `analytic` é `true`), entrada no `protocols/CHANGELOG.md`, versão no `README.md` e tag anotada `protocols-v1.2.0` depois do merge. O repo não tem código executável nem suíte: a prova é validar o arquivo com o mesmo `json_schemer` do `api`, dentro do container, lendo o schema pela entrada padrão.

**Tech Stack:** JSON Schema draft 2020-12; `json_schemer` 2.5 (do bundle do `api`, só para a verificação).

**Spec:** `docs/.claude/mod14/superpowers/specs/2026-09-30-module-14-analytics-design.md` §7 e `docs/.claude/mod14/adr/0025.md`. Contratos entre apps: `docs/.claude/mod14/superpowers/plans/2026-09-30-module-14-analytics-contracts.md` §4. O plano do api (`2026-09-30-module-14-analytics-api.md`, Task 7) copia este arquivo; **este plano roda antes dele**.

## Global Constraints

- SemVer por domínio (ADR 0015): mudança só aditiva (campo opcional) → **MINOR**, `protocols-v1.1.0` → `protocols-v1.2.0`. Nada vira obrigatório, nada é removido: toda definição válida em `v1.1.0` continua válida.
- CHANGELOG obrigatório: sem entrada em `protocols/CHANGELOG.md`, não há tag.
- Tag **anotada**, mensagem `protocols-v1.2.0 — MINOR` (mesmo formato de `protocols-v1.1.0`), criada no commit que chega à `main`, só com autorização do usuário.
- O `api` usa uma **cópia** (`apps/api/config/protocols/schema.json`): os dois arquivos precisam ficar byte a byte iguais no mesmo ciclo.
- Commits em inglês; o commit de release do domínio pode começar pela tag (`protocols-v1.2.0: …`, regra do README). Terminam com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git`.
- Nada de push, merge ou tag sem autorização explícita do usuário, uma etapa de cada vez.

## Review Focus

1. **Definição antiga, sem `analytic`:** continua válida (é o caso de toda versão já publicada nas cidades). Teste: Task 1, caso "sem analytic (v1.1.0)".
2. **`analytic: false` numa pergunta `integer`/`text`:** válida — o editor pode gravar `false` ao desmarcar. Teste: Task 1, caso "integer com analytic false".
3. **`analytic` com valor que não é booleano (`"sim"`, `1`):** recusado. Teste: Task 1, casos "analytic string" e "analytic número".
4. **`enum` marcado sem `options`:** o `if`/`then` não pode esconder a regra existente; a obrigatoriedade de `options` continua sendo do validador semântico do `api`, como antes. Teste: Task 1, caso "enum analytic" (com `options`) — e nenhum caso novo afrouxa `options`.
5. **Cópia divergente no `api`:** o `diff` final entre os dois arquivos é vazio. Teste: Task 1, Step 7.

---

### Task 1: `analytic` em `$defs/step`, CHANGELOG e README

**Files (repo `contracts`, worktree `contracts/.claude/mod14`, branch `feat/protocols-analytic`):**
- Modify: `protocols/schema.json`
- Modify: `protocols/CHANGELOG.md`
- Modify: `README.md`

**Interfaces:**
- Produces: `$defs/step.properties.analytic` (`boolean`, opcional) e a regra `if { analytic: const true } then { answer_type ∈ [boolean, enum] }`; `protocols-v1.2.0` no CHANGELOG e no README. Consumido pela Task 7 do plano do api (cópia literal) e pelo editor do dashboard (via api).

- [ ] **Step 1: Crie o worktree**

A partir da raiz do monorepo (`/Users/eduardovrocha/Development/ioit.solutions/rota-saude`; `contracts/.gitignore` já ignora `/.claude/`):

```bash
/opt/homebrew/bin/git -C contracts fetch origin
/opt/homebrew/bin/git -C contracts worktree add .claude/mod14 -b feat/protocols-analytic origin/main
```

- [ ] **Step 2: Escreva a verificação e veja falhar**

O `json_schemer` vem do bundle do `api` (o container só monta `apps/api`), então o schema do `contracts` entra pela entrada padrão. A partir da raiz do monorepo:

```bash
docker compose exec -T -w /rails api bundle exec ruby -rjson -rjson_schemer -e '
schema = JSONSchemer.schema(JSON.parse(STDIN.read))
doc = ->(step) { { "name" => "contrato", "version" => 1, "start_step_id" => "s1",
                   "steps" => [ { "id" => "s1", "prompt" => "?" }.merge(step) ] } }
cases = {
  "sem analytic (v1.1.0)"        => [ doc.({ "answer_type" => "integer" }), true ],
  "boolean analytic"             => [ doc.({ "answer_type" => "boolean", "analytic" => true }), true ],
  "enum analytic"                => [ doc.({ "answer_type" => "enum", "options" => [ "a", "b" ], "analytic" => true }), true ],
  "integer com analytic false"   => [ doc.({ "answer_type" => "integer", "analytic" => false }), true ],
  "text com analytic false"      => [ doc.({ "answer_type" => "text", "analytic" => false }), true ],
  "integer analytic"             => [ doc.({ "answer_type" => "integer", "analytic" => true }), false ],
  "text analytic"                => [ doc.({ "answer_type" => "text", "analytic" => true }), false ],
  "analytic string"              => [ doc.({ "answer_type" => "boolean", "analytic" => "sim" }), false ],
  "analytic número"              => [ doc.({ "answer_type" => "boolean", "analytic" => 1 }), false ]
}
failed = cases.reject { |_name, (definition, expected)| schema.valid?(definition) == expected }.keys
abort("FALHOU: #{failed.join(", ")}") if failed.any?
puts "ok: #{cases.size} casos"
' < contracts/.claude/mod14/protocols/schema.json
```

Expected: `FALHOU: boolean analytic, enum analytic, integer com analytic false, text com analytic false` — hoje `$defs/step` tem `additionalProperties: false` e recusa qualquer `analytic`.

- [ ] **Step 3: Edite o schema**

Em `contracts/.claude/mod14/protocols/schema.json`, dentro de `$defs.step`:

Antes:
```json
        "weights": {
          "type": "object",
          "additionalProperties": { "type": "integer" },
          "description": "answer_value -> peso numérico (usado por scoring weighted)"
        }
      }
    },
    "scoring_weighted": {
```

Depois:
```json
        "weights": {
          "type": "object",
          "additionalProperties": { "type": "integer" },
          "description": "answer_value -> peso numérico (usado por scoring weighted)"
        },
        "analytic": {
          "type": "boolean",
          "description": "Usar em Analytics (ADR 0025): respostas agregadas por bairro, nunca por pessoa. Só boolean e enum."
        }
      },
      "if": {
        "required": ["analytic"],
        "properties": { "analytic": { "const": true } }
      },
      "then": {
        "properties": { "answer_type": { "enum": ["boolean", "enum"] } }
      }
    },
    "scoring_weighted": {
```

- [ ] **Step 4: Rode a verificação e veja passar**

Run: o mesmo comando do Step 2, e depois `python3 -m json.tool contracts/.claude/mod14/protocols/schema.json > /dev/null && echo json-ok`
Expected: `ok: 9 casos` e `json-ok`.

- [ ] **Step 5: CHANGELOG e README**

Em `protocols/CHANGELOG.md`, logo abaixo de `# Changelog — protocols` (antes da entrada `v1.1.0`):

```markdown

## protocols-v1.2.0 — 2026-09-30 — MINOR
- `analytic` (opcional, boolean) na pergunta (`$defs/step`): marca a pergunta para o Analytics
  (ADR 0025) — as respostas dela aparecem agregadas por bairro, nunca por pessoa. Só vale `true`
  com `answer_type` `boolean` ou `enum` (`if`/`then` no passo); `integer` e `text` com
  `analytic: true` são recusados. Ausente equivale a `false`.
- A marca é parte da versão do protocolo e passa pelo ciclo assinado (ADR 0016).
- Expand: nada vira obrigatório e nada foi removido — toda definição válida em `v1.1.0` continua
  válida, por isso MINOR. O `api` atualiza a cópia (`config/protocols/schema.json`) no mesmo ciclo,
  antes do `dashboard` passar a gravar a marca.
```

Em `README.md`:
- na tabela "Domínios", `` `protocols-v1.1.0` `` passa a `` `protocols-v1.2.0` ``;
- no parágrafo de `### protocols`, o trecho

```markdown
pelo `api`, que valida contra ele. `v1.1.0`
acrescentou `recommendations`, `priority_when` e a gramática de condições
(`$defs/condition`), sem remover nada.
```

passa a

```markdown
pelo `api`, que valida contra ele. `v1.1.0`
acrescentou `recommendations`, `priority_when` e a gramática de condições
(`$defs/condition`), sem remover nada. `v1.2.0` acrescentou `analytic` na
pergunta (Analytics, ADR 0025), válido só em `boolean` e `enum`.
```

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git -C contracts/.claude/mod14 add protocols/schema.json protocols/CHANGELOG.md README.md
/opt/homebrew/bin/git -C contracts/.claude/mod14 commit -m "protocols-v1.2.0: add the analytic flag to protocol questions

Optional boolean on \$defs/step, accepted as true only for boolean and enum
answer types (ADR 0025). Additive: every v1.1.0 definition stays valid.

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

- [ ] **Step 7: Confira a cópia do api (depois da Task 7 do plano do api)**

Run: `diff contracts/.claude/mod14/protocols/schema.json apps/api/.claude/mod14/config/protocols/schema.json && echo identicos`
Expected: `identicos`.

### Task 2: Merge, tag `protocols-v1.2.0` e push (só com autorização)

- [ ] **Step 1: Pare e peça autorização** ao usuário para: (a) levar `feat/protocols-analytic` à `main` do `contracts`, (b) criar a tag, (c) fazer push. Uma etapa de cada vez. Antes do push, confira `origin/main..main` (outra sessão pode ter commits locais; publique só o desta).

- [ ] **Step 2: Merge (fast-forward) na main**

```bash
/opt/homebrew/bin/git -C contracts fetch origin
/opt/homebrew/bin/git -C contracts checkout main
/opt/homebrew/bin/git -C contracts merge --ff-only feat/protocols-analytic
```
Expected: fast-forward de um commit. Se não for fast-forward, pare e reporte.

- [ ] **Step 3: Tag anotada no commit da main**

```bash
/opt/homebrew/bin/git -C contracts tag -a protocols-v1.2.0 -m "protocols-v1.2.0 — MINOR"
/opt/homebrew/bin/git -C contracts show protocols-v1.2.0 --stat --format='%an %s' | head -8
```
Expected: `tag protocols-v1.2.0`, `protocols-v1.2.0 — MINOR`, e o commit do Step 6 com os três arquivos.

- [ ] **Step 4: Push da main e da tag**

```bash
/opt/homebrew/bin/git -C contracts log --oneline origin/main..main
/opt/homebrew/bin/git -C contracts push origin main
/opt/homebrew/bin/git -C contracts push origin protocols-v1.2.0
```
Expected: `origin/main..main` mostra só o commit desta entrega; o push publica a main e a tag.

- [ ] **Step 5: Limpe o worktree**

```bash
/opt/homebrew/bin/git -C contracts worktree remove .claude/mod14
/opt/homebrew/bin/git -C contracts branch -d feat/protocols-analytic
```

---

## Self-review (feito ao escrever o plano)

- **Cobertura da spec:** §7 (propriedade opcional `analytic`; recusa em `integer`/`text`; fonte no `contracts`, cópia no api) → Task 1; contratos §4 → Task 1; ADR 0015 (MINOR, CHANGELOG, tag por domínio) → Tasks 1 e 2. A marca pelo ciclo assinado e a caixa do editor são dos planos do api e do dashboard.
- **Placeholders:** nenhum; o JSON, o CHANGELOG, o README e o script de verificação estão completos.
- **Consistência:** o bloco "Depois" do Step 3 é o mesmo da Task 7 do plano do api; a mensagem da tag segue a de `protocols-v1.1.0`.
- **Review Focus:** os cinco itens têm caso no script (Step 2) ou o `diff` (Step 7).
