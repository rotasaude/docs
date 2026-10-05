# Módulo 16 — `features` no contrato de sessão (repo contracts) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Publicar no repo `contracts` o campo opcional `features` em `session_user` (`session-v1.1.0`, MINOR), com exemplos válidos e inválidos versionados e verificados pelo mesmo `json_schemer` do `api` (F-16.1; ADR 0028).

**Architecture:** Uma edição em `session/schema.json` (`$defs/session_user.properties.features`), exemplos de corpo de sessão em `session/examples/` com um `manifest.json` no formato que `protocols/examples/` já usa (`valid` ou `<data_pointer> <type>` do `json_schemer`), entrada no `session/CHANGELOG.md` e versão no `README.md`. O repo não tem código executável nem suíte: a prova é validar os exemplos no `json_schemer` 2.5 do container do `api`, recebendo schema e exemplos pela entrada padrão (o container só monta `apps/api`). Merge, tag anotada `session-v1.1.0` e push só com autorização.

**Tech Stack:** JSON Schema draft 2020-12; `json_schemer` 2.5.0 (do bundle do `api`, só para a verificação); `python3` do host para gerar os exemplos e empacotar schema + exemplos num JSON.

**Spec:** `docs/.claude/ciclo2/superpowers/specs/2026-10-05-module-16-record-mode-and-export-design.md` §3.1 e §9 (ordem: `contracts` primeiro); ADR `docs/.claude/ciclo2/adr/0028.md`. Contratos entre apps (fonte única do formato): `docs/.claude/ciclo2/superpowers/plans/2026-10-05-module-16-record-mode-contracts.md` §1 e §7. **Este plano roda antes do plano `api-foundation`**, que passa a emitir `features` em `GET /session`.

## Global Constraints

- SemVer por domínio (ADR 0015): só acréscimo opcional → **MINOR**, `session-v1.0.0` → `session-v1.1.0`. Nada vira obrigatório, nada é removido: todo corpo de sessão válido em `v1.0.0` continua válido.
- Formato exato do contrato §1: `session_user.features` **opcional**, array de string, as chaves de interruptor **ligadas** para a cidade do host (não necessariamente utilizáveis). Ausente na sessão do console de plataforma (`Operators::SessionsController`). Consumidor trata ausente como `[]` e ignora chave desconhecida.
- Restrições do contrato §1 sobre o array: `uniqueItems: true` (uma chave ligada aparece uma vez) e `pattern: "^[a-z][a-z0-9_]*$"` nos itens (as chaves do catálogo, `ledi_export` e `cadsus_lookup`, são snake_case). O `pattern` **não** é uma lista fechada: chave nova do catálogo continua válida sem versão nova do contrato.
- A recusa `403 { "error": "feature_disabled", "feature": "<key>" }` (contrato §1) fica documentada no CHANGELOG; o schema de sessão não descreve corpos de erro e não ganha `$defs` para ela.
- CHANGELOG obrigatório: sem entrada em `session/CHANGELOG.md`, não há tag.
- Tag **anotada**, mensagem `session-v1.1.0 — MINOR` (mesmo formato de `protocols-v1.4.0 — MINOR`), criada no commit que chega à `main`.
- O `api` **não** tem cópia do schema de sessão (diferente de `protocols`): nada a sincronizar lá. Tipos TypeScript: nenhum a atualizar aqui (`contracts/types/` é scaffold; `dashboard` e `admin` têm o tipo de sessão próprio, atualizados pelos planos deles).
- Worktree `contracts/.claude/mod16`, branch `feat/session-features`, a partir de `origin/main` (`contracts/.gitignore` já ignora `/.claude/`). Todos os comandos rodam a partir da raiz do monorepo `/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, com o compose de pé (`docker compose ps` mostra `api running`).
- Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Commits em inglês, Conventional Commits com tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Nunca `git add -A`: caminhos explícitos.
- Nada de merge na `main`, tag ou push sem autorização explícita do usuário, uma etapa de cada vez.

## Review Focus

1. **A sessão que o `api` emite hoje (sem `features`) continua válida.** É o corpo de `SessionsController#serialize` antes do módulo 16, e o de todo api ainda não atualizado durante o rollout. Teste: `city-user-without-features.json` (Task 1), esperado `valid`.
2. **Sessão do console de plataforma** (operador, sem `time_zone` e sem `features`): continua válida. Teste: `platform-console-without-features.json` (Task 1), esperado `valid`.
3. **Chave desconhecida** (api mais novo que o consumidor, ex. `rnds_export`): válida no schema; quem ignora é o consumidor. Teste: `city-user-unknown-feature.json` (Task 1), esperado `valid`.
4. **`features: []`** (cidade sem nada ligado, o estado de toda cidade no primeiro deploy): válido, e não é o mesmo que ausente na leitura do contrato (ausente = sessão de plataforma). Teste: `city-user-features-empty.json` (Task 1), esperado `valid`.
5. **Forma de objeto por engano.** O `GET /cities/:id` do console (contrato §4.2) devolve `features: [{ key, enabled, usable, missing }]`; quem copiar essa forma para a sessão quebra o consumidor que espera string. O schema recusa. Teste: `invalid-features-object-items.json` (Task 1), esperado `/features/0 string`.

---

## Mapa de arquivos (repo `contracts`, worktree `contracts/.claude/mod16`)

| Arquivo | Responsabilidade | Task |
|---|---|---|
| `session/schema.json` | `features` em `$defs/session_user.properties` | 1 |
| `session/examples/*.json`, `session/examples/manifest.json` | corpos de sessão válidos e inválidos, e o erro esperado de cada um | 1 |
| `session/CHANGELOG.md` | entrada `session-v1.1.0` | 1 |
| `README.md` | versão do domínio `session` e o comando de verificação | 1 |
| — | conferência final, autorização, merge, tag, push | 2 |

---

### Task 1: `features` em `session_user`, com exemplos, CHANGELOG e README (`session-v1.1.0`)

**Files (repo `contracts`, worktree `contracts/.claude/mod16`):**
- Create: `session/examples/manifest.json`
- Create: `session/examples/city-user-without-features.json`, `city-user-features-empty.json`, `city-user-features.json`, `city-user-unknown-feature.json`, `operator-grant-features.json`, `platform-console-without-features.json`
- Create: `session/examples/invalid-features-not-array.json`, `invalid-features-object-items.json`, `invalid-features-duplicate.json`, `invalid-features-bad-key.json`
- Modify: `session/schema.json` (linha 34, `time_zone` em `$defs/session_user.properties`)
- Modify: `session/CHANGELOG.md`
- Modify: `README.md` (tabela de domínios e seção nova `### session`)

**Interfaces:**
- Produces: `$defs/session_user.properties.features` = `{ "type": "array", "uniqueItems": true, "items": { "type": "string", "pattern": "^[a-z][a-z0-9_]*$" } }`, fora de `required`. Formato do manifesto (o mesmo de `protocols/examples/manifest.json`): `{ "cases": [ { "file": "<nome>.json", "expect": "valid" | "<data_pointer> <type>" } ] }`, em que `<data_pointer>` é o ponteiro JSON do erro (`(root)` se vazio) e `<type>` é o `type` do erro do `json_schemer` (`array`, `string`, `uniqueItems`, `pattern`…). Consumido pelo plano `api-foundation` (spec de request de `GET /session` pode validar contra este schema) e pelos planos `dashboard`/`admin` (tipo de sessão com `features?: string[]`).

- [ ] **Step 1: Crie o worktree**

```bash
/opt/homebrew/bin/git -C contracts fetch origin
/opt/homebrew/bin/git -C contracts log --oneline -1 origin/main
/opt/homebrew/bin/git -C contracts worktree add .claude/mod16 -b feat/session-features origin/main
```

Expected: `origin/main` em `a912c1c feat: add offer and suggestions to the protocols schema (protocols-v1.4.0)` (ou mais novo — se houver commit novo em `session/`, pare e reporte); worktree criado.

- [ ] **Step 2: Escreva os exemplos**

Os corpos imitam `SessionsController#serialize` (usuário da cidade), `#serialize_operator_grant` (operador por grant, dentro da cidade) e `Operators::SessionsController` (console de plataforma, sem `time_zone`). A partir da raiz do monorepo:

```bash
mkdir -p contracts/.claude/mod16/session/examples
cd contracts/.claude/mod16/session/examples && python3 - <<'EOF'
import json
def user(**extra):
    d = {"id": "0b9f6c1e-3d2a-4f7b-9c1e-5a6b7c8d9e0f", "email_address": "admin@curitiba.demo",
         "mfa_enrolled": True, "operator": False, "mfa_verified_at": "2026-10-05T14:03:00Z",
         "memberships": [{"city_slug": "curitiba", "city_name": "Curitiba", "city_uf": "PR", "role": "municipal_admin"}],
         "time_zone": "America/Sao_Paulo"}
    d.update(extra)
    return d
def console():
    d = user(email_address="operador@rotasaude.dev", operator=True, mfa_verified_at=None, memberships=[])
    del d["time_zone"]
    return d
files = {
    "city-user-without-features.json": user(),
    "city-user-features-empty.json": user(features=[]),
    "city-user-features.json": user(features=["ledi_export", "cadsus_lookup"]),
    "city-user-unknown-feature.json": user(features=["rnds_export"]),
    "operator-grant-features.json": user(operator=True, mfa_verified_at=None, memberships=[], features=["ledi_export"]),
    "platform-console-without-features.json": console(),
    "invalid-features-not-array.json": user(features="ledi_export"),
    "invalid-features-object-items.json": user(features=[{"key": "ledi_export", "enabled": True}]),
    "invalid-features-duplicate.json": user(features=["ledi_export", "ledi_export"]),
    "invalid-features-bad-key.json": user(features=["LEDI-Export"]),
}
for name, doc in files.items():
    with open(name, "w", encoding="utf-8") as f:
        f.write(json.dumps(doc, indent=2, ensure_ascii=False) + "\n")
EOF
```

Depois crie `contracts/.claude/mod16/session/examples/manifest.json`:

```json
{
  "cases": [
    { "file": "city-user-without-features.json", "expect": "valid" },
    { "file": "city-user-features-empty.json", "expect": "valid" },
    { "file": "city-user-features.json", "expect": "valid" },
    { "file": "city-user-unknown-feature.json", "expect": "valid" },
    { "file": "operator-grant-features.json", "expect": "valid" },
    { "file": "platform-console-without-features.json", "expect": "valid" },
    { "file": "invalid-features-not-array.json", "expect": "/features array" },
    { "file": "invalid-features-object-items.json", "expect": "/features/0 string" },
    { "file": "invalid-features-duplicate.json", "expect": "/features uniqueItems" },
    { "file": "invalid-features-bad-key.json", "expect": "/features/0 pattern" }
  ]
}
```

- [ ] **Step 3: Rode a verificação e veja falhar**

O comando é o de `protocols/README.md`, apontado para o schema e o manifesto de sessão. A partir da raiz do monorepo:

```bash
python3 -c '
import json, os, sys
schema_path, manifest_path, extras = sys.argv[1], sys.argv[2], sys.argv[3:]
base = os.path.dirname(manifest_path)
cases = [dict(c, doc=json.load(open(os.path.join(base, c["file"])))) for c in json.load(open(manifest_path))["cases"]]
cases += [{"file": p, "expect": "valid", "doc": json.load(open(p))} for p in extras]
print(json.dumps({"schema": json.load(open(schema_path)), "cases": cases}))
' contracts/.claude/mod16/session/schema.json contracts/.claude/mod16/session/examples/manifest.json \
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

Expected (exit 1; conferido contra a `v1.0.0` em 2026-10-05):

```
ok   city-user-without-features.json — esperado: valid; obtido: valid
ok   city-user-features-empty.json — esperado: valid; obtido: valid
ok   city-user-features.json — esperado: valid; obtido: valid
ok   city-user-unknown-feature.json — esperado: valid; obtido: valid
ok   operator-grant-features.json — esperado: valid; obtido: valid
ok   platform-console-without-features.json — esperado: valid; obtido: valid
FAIL invalid-features-not-array.json — esperado: /features array; obtido: valid
FAIL invalid-features-object-items.json — esperado: /features/0 string; obtido: valid
FAIL invalid-features-duplicate.json — esperado: /features uniqueItems; obtido: valid
FAIL invalid-features-bad-key.json — esperado: /features/0 pattern; obtido: valid
6/10 casos
```

(`additionalProperties: true` em `session_user`: hoje qualquer `features` passa sem ser conferido.)

- [ ] **Step 4: Acrescente `features` a `$defs/session_user.properties`**

Em `contracts/.claude/mod16/session/schema.json`, troque

```json
        "time_zone": { "type": "string", "description": "Fuso IANA da cidade do host; as telas formatam hora nele. Ausente na sessão aberta no console de plataforma (Operators::SessionsController)." }
      }
```

por

```json
        "time_zone": { "type": "string", "description": "Fuso IANA da cidade do host; as telas formatam hora nele. Ausente na sessão aberta no console de plataforma (Operators::SessionsController)." },
        "features": {
          "type": "array",
          "uniqueItems": true,
          "items": { "type": "string", "pattern": "^[a-z][a-z0-9_]*$" },
          "description": "Chaves de interruptor de funcionalidade LIGADAS para a cidade do host (ADR 0028); ligada não quer dizer utilizável — o que falta vem da rota da funcionalidade. Ausente na sessão aberta no console de plataforma. Consumidor trata ausente como [] e ignora chave desconhecida.",
          "examples": [["ledi_export", "cadsus_lookup"]]
        }
      }
```

`required` de `session_user` **não** muda.

- [ ] **Step 5: Rode a verificação e veja passar**

Run: o mesmo comando do Step 3; depois `python3 -m json.tool contracts/.claude/mod16/session/schema.json > /dev/null && echo json-ok`
Expected: dez linhas `ok`, `10/10 casos`, exit 0; e `json-ok`.

Confira também que o diff do schema só acrescenta (a única linha "removida" é a do `time_zone`, que ganha a vírgula):

```bash
/opt/homebrew/bin/git -C contracts/.claude/mod16 diff --stat session/schema.json
/opt/homebrew/bin/git -C contracts/.claude/mod16 diff session/schema.json | grep '^-[^-]'
```

Expected: `1 file changed, 8 insertions(+), 1 deletion(-)`; a única linha `-` é a do `"time_zone"`.

- [ ] **Step 6: Entrada no CHANGELOG**

Em `contracts/.claude/mod16/session/CHANGELOG.md`, insira logo depois da linha `# Changelog — session` (e da linha em branco que a segue):

```markdown
## session-v1.1.0 — 2026-10-05 — MINOR

`session_user` ganha `features` (opcional): array com as chaves de interruptor
de funcionalidade **ligadas** para a cidade do host (ADR 0028, módulo 16). Hoje
as chaves do catálogo do `api` são `ledi_export` e `cadsus_lookup`.

- **Ligada não quer dizer utilizável.** O que falta (modo de prontuário,
  endereço do PEC, código IBGE, credencial) o consumidor pergunta à rota da
  funcionalidade; a sessão não carrega pré-requisito.
- **Ausente** na sessão aberta no console de plataforma
  (`Operators::SessionsController`). A sessão de operador por grant, dentro da
  cidade, traz `features` como a de usuário.
- **Tolerância do consumidor:** ausente vale `[]`; chave desconhecida é
  ignorada. O schema confere a forma (string snake_case, sem repetição), não a
  lista de chaves: chave nova do catálogo não pede versão nova deste contrato.
- **Recusa** de rota de funcionalidade desligada:
  `403 { "error": "feature_disabled", "feature": "<key>" }`.

Nada removido, nada passou a obrigatório: toda sessão válida em `v1.0.0`
continua válida. Exemplos válidos e inválidos em `session/examples/`.

```

- [ ] **Step 7: README — versão e comando de verificação**

Em `contracts/.claude/mod16/README.md`, troque a linha da tabela de domínios

```markdown
| [`session/`](session/CHANGELOG.md) | Corpo da sessão (`GET /session`) e escopo do envelope de `/admin/api` | `session-v1.0.0` | Materializado |
```

por

```markdown
| [`session/`](session/CHANGELOG.md) | Corpo da sessão (`GET /session`) e escopo do envelope de `/admin/api` | `session-v1.1.0` | Materializado |
```

E insira, logo antes da linha `### Fora deste repo, por enquanto`:

````markdown
### session

`schema.json` descreve o corpo de `GET /session` (e de `POST /session` e
`POST /session/grant`) e o `data.scope` do envelope de `/admin/api`. `v1.1.0`
acrescentou `features` (chaves de interruptor ligadas na cidade do host, ADR
0028). O `api` não tem cópia deste schema. Exemplos válidos e inválidos ficam
em `session/examples/`, com o mesmo formato de manifesto de
`protocols/examples/`; verifique, a partir da raiz do monorepo e com o compose
de pé:

```bash
python3 -c '
import json, os, sys
schema_path, manifest_path, extras = sys.argv[1], sys.argv[2], sys.argv[3:]
base = os.path.dirname(manifest_path)
cases = [dict(c, doc=json.load(open(os.path.join(base, c["file"])))) for c in json.load(open(manifest_path))["cases"]]
cases += [{"file": p, "expect": "valid", "doc": json.load(open(p))} for p in extras]
print(json.dumps({"schema": json.load(open(schema_path)), "cases": cases}))
' contracts/session/schema.json contracts/session/examples/manifest.json \
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

- [ ] **Step 8: Commit**

```bash
/opt/homebrew/bin/git -C contracts/.claude/mod16 add README.md session/schema.json session/CHANGELOG.md session/examples
/opt/homebrew/bin/git -C contracts/.claude/mod16 status --short
/opt/homebrew/bin/git -C contracts/.claude/mod16 commit -m "feat: add enabled feature keys to the session contract (session-v1.1.0)

Optional features array on session_user: the feature switch keys turned on
for the host city (ADR 0028). Absent on the platform console session;
consumers read absent as an empty list and ignore unknown keys. MINOR per
ADR 0015: nothing removed or made required. Adds session/examples with a
manifest checked with the api's json_schemer.

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

Expected: antes do commit, `status --short` lista só `README.md`, `session/schema.json`, `session/CHANGELOG.md` e os 11 arquivos de `session/examples/` (manifesto + 10 exemplos).

---

### Task 2: Conferência final, autorização e publicação

**Files:** nenhum novo.

**Interfaces:**
- Consumes: o commit da Task 1 na branch `feat/session-features`.
- Produces: `main` do `contracts` com o commit e a tag anotada `session-v1.1.0`, publicadas em `origin` — pré-requisito do plano `api-foundation` (contrato §7, passo 1).

- [ ] **Step 1: Confira a branch inteira**

```bash
/opt/homebrew/bin/git -C contracts/.claude/mod16 log --oneline origin/main..HEAD
/opt/homebrew/bin/git -C contracts/.claude/mod16 diff --stat origin/main..HEAD
```

Expected: um commit (`feat: add enabled feature keys to the session contract (session-v1.1.0)`); `diff --stat` só em `README.md`, `session/CHANGELOG.md`, `session/schema.json` e `session/examples/` (11 arquivos). Rode de novo o comando de verificação da Task 1, Step 3: `10/10 casos`.

- [ ] **Step 2: Pare e peça autorização**

Reporte ao usuário: hash do commit, `10/10 casos`, e a lista do que falta. **Não execute nada abaixo sem um "sim" explícito do usuário para cada etapa** (a) merge na `main` local, (b) tag, (c) push. Antes do push, confira `origin/main..main`: outra sessão (o módulo 15 corre em paralelo) pode ter commits locais no `contracts`; publique só os desta entrega.

- [ ] **Step 3 (só com autorização): Merge fast-forward na main**

```bash
/opt/homebrew/bin/git -C contracts fetch origin
/opt/homebrew/bin/git -C contracts status -sb
/opt/homebrew/bin/git -C contracts checkout main
/opt/homebrew/bin/git -C contracts merge --ff-only feat/session-features
```

Expected: `## main...origin/main` sem divergência antes; fast-forward de um commit. Se não for fast-forward (outra sessão publicou em `contracts` desde o Step 1 da Task 1), pare e reporte: rebase da branch e nova verificação antes de tentar de novo.

- [ ] **Step 4 (só com autorização): Tag anotada**

```bash
/opt/homebrew/bin/git -C contracts tag -a session-v1.1.0 -m "session-v1.1.0 — MINOR"
/opt/homebrew/bin/git -C contracts show session-v1.1.0 --stat --format='%an %s' | head -12
```

Expected: `tag session-v1.1.0`, `session-v1.1.0 — MINOR`, e o commit `feat: add enabled feature keys to the session contract (session-v1.1.0)`.

- [ ] **Step 5 (só com autorização): Push da main e da tag**

```bash
/opt/homebrew/bin/git -C contracts log --oneline origin/main..main
/opt/homebrew/bin/git -C contracts push origin main
/opt/homebrew/bin/git -C contracts push origin session-v1.1.0
```

Expected: `origin/main..main` mostra só o commit desta entrega; o push publica a `main` e a tag.

- [ ] **Step 6: Limpe o worktree (depois do merge)**

```bash
/opt/homebrew/bin/git -C contracts worktree remove .claude/mod16
/opt/homebrew/bin/git -C contracts branch -d feat/session-features
```

---

## Incorporado ao contrato

As duas divergências levantadas ao escrever este plano foram aceitas no contrato (`…-contracts.md` §1):

1. **`uniqueItems: true` e `pattern: "^[a-z][a-z0-9_]*$"` nos itens de `features`** — agora parte do formato do §1; o `pattern` não fecha a lista de chaves.
2. **`403 feature_disabled` documentado só no CHANGELOG**, sem `$defs` no schema de sessão.

Nenhuma divergência em aberto.

## Self-review (feito ao escrever o plano)

- **Cobertura:** contrato §1 (`features` opcional, ligadas, ausente na sessão de plataforma, ausente = `[]`, chave desconhecida ignorada, recusa 403) → Task 1 (schema, exemplos, CHANGELOG); contrato §7 passo 1 e spec §9 (push com autorização) → Task 2.
- **Placeholders:** nenhum. Schema, exemplos (gerados por script exato), manifesto, comando, CHANGELOG, README e mensagens de commit estão completos. As saídas esperadas (ponteiros e tipos de erro) foram conferidas contra o `json_schemer` 2.5.0 do container em 2026-10-05, com a `v1.0.0` atual (6/10, exit 1) e com o schema final (10/10, exit 0).
- **Consistência:** nomes de arquivo do manifesto = nomes gerados pelo script; o comando é o mesmo no Step 3 e no README (com o caminho de `contracts/session/`); a mensagem da tag segue `protocols-v1.4.0 — MINOR`.
- **Review Focus:** os cinco itens têm caso no manifesto da Task 1.
