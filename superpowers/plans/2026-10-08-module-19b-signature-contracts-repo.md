# Módulo 19b — JSON canônico da consulta e do adendo (repo contracts) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **PRÉ-REQUISITO: 19a fechado em `origin/main`** (api e dashboard; F-19.1..7 Verified). O esquema copia valores que o 19a fixa no código (tipos de atendimento, condutas, desfechos, limites de texto e de motivo, conselhos). A Task 0 confere isso antes de tudo.

**Goal:** Publicar no repo `contracts` o domínio novo `clinical` — os esquemas JSON 2020-12 do JSON canônico assinado da consulta (`rotasaude.consultation.v1`) e do adendo (`rotasaude.consultation_addendum.v1`), com exemplos válidos e inválidos, um vetor de canonicalização RFC 8785 e a tag `clinical-v1.0.0` (F-19.11, F-19.13; ADR 0032).

**Architecture:** Dois arquivos de esquema fechados (`additionalProperties: false` em todo nível: o que não está no esquema não entra no que se assina) com o mesmo `$defs` (cabeçalho cidade/unidade/profissional/paciente e os itens clínicos), conferido idêntico por um comando. Os exemplos são gerados por um script Python e verificados com o `json_schemer` 2.5 do `api`, dentro do container, pelo mesmo manifesto `{ file, expect }` de `protocols/` e `session/`. O vetor de canonicalização (bytes JCS + SHA-256) dá ao `api` e a quem mais gerar o JSON canônico uma referência byte a byte. Merge, tag anotada e push só com autorização.

**Tech Stack:** JSON Schema draft 2020-12; `json_schemer` 2.5 (bundle do `api`, só para a verificação); `python3` do host (gerar exemplos, empacotar schema + exemplos, gerar o JCS); `shasum`.

**Spec:** `docs/superpowers/specs/2026-10-08-module-19b-digital-signature-design.md` §7; ADR `docs/adr/0032.md`; contrato entre apps (fonte única dos formatos): `docs/superpowers/plans/2026-10-08-module-19b-signature-contracts.md` §10 e §12 (passo 1); contrato do 19a `docs/superpowers/plans/2026-10-07-module-19-consultation-contracts.md` §4 e §9 (formas da consulta e do adendo de onde o JSON canônico sai).

## Global Constraints

- Repo `contracts` na raiz do monorepo: `/Users/eduardovrocha/Development/ioit.solutions/rota-saude/contracts` (remote `git@github.com:rotasaude/contracts.git`). Não existe `apps/contracts`.
- Domínio novo `clinical/` na raiz do repo, como `protocols/` e `session/`; tag `clinical-vX.Y.Z` (ADR 0015). Primeira versão: `clinical-v1.0.0`, anotada, mensagem `clinical-v1.0.0 — MAJOR` (primeira versão do domínio, como `session-v1.0.0`). Ver Divergência C1 (caminho).
- `schema`: `"rotasaude.consultation.v1"` e `"rotasaude.consultation_addendum.v1"` (spec §7), `const` no esquema.
- Serialização assinada: RFC 8785 (JCS), UTF-8. O esquema não garante a canonicalização; garante só formas que têm **uma** representação: datas UTC `AAAA-MM-DDTHH:MM:SSZ` (sem fração, sem fuso), datas `AAAA-MM-DD`, CPF/CNES/IBGE/CBO só dígitos, texto vazio = `null` (nunca `""`), medidas ausentes = chave ausente em `vitals`.
- Valores copiados do 19a (conferidos na Task 0 em `apps/api` `origin/main`): `care_type` ∈ {1, 2, 5, 6}; condutas ∈ {1, 2, 4–12, 14}, 1 a 12, sem repetição; até 50 problemas avaliados (mínimo 1 na consulta); exames só SIGTAP do grupo 02, até 100; textos S/O/A/P e do adendo até 20.000; motivo do adendo 10–500; desfecho da consulta ∈ {`discharged`, `referred`, `return`} (o `left` do balcão não fecha consulta); conselhos = `Professional::COUNCILS`.
- O esquema não confere plausibilidade de sinais vitais, CID-10 por CBO, nome reservado, nem relação entre campos (isso é do `api`).
- CHANGELOG obrigatório (`clinical/CHANGELOG.md`): sem entrada, não há tag.
- Worktree `contracts/.claude/mod19b`, branch `feat/mod-19b-signature` a partir de `origin/main` (`contracts/.gitignore` já ignora `/.claude/`). Comandos a partir da raiz do monorepo, com o compose de pé (`docker compose ps` mostra `api-dev running`).
- Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Nunca `git add -A`: caminhos explícitos. Commits em inglês, Conventional Commits com tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Merge na `main`, tag e push só com autorização explícita do usuário, uma etapa de cada vez.
- Nenhum dado real nos exemplos: nomes fictícios, CPFs com dígito verificador válido mas inventados.

## Review Focus

1. **Hora com fuso ou com fração de segundo** (`2026-10-08T10:41:37-03:00`, `...37.123Z`) — o mesmo instante teria dois textos e dois hashes; o esquema recusa e aponta o campo. Testes: `invalid-finalized-at-offset.json` e `invalid-finalized-at-fraction.json` (Task 1), esperado `/consultation/finalized_at pattern`.
2. **CPF formatado ou mascarado** (`390.533.447-05`) — o CPF do assinado é comparado com o do certificado; o esquema só aceita 11 dígitos. Teste: `invalid-patient-cpf-masked.json` (Task 1), esperado `/patient/cpf pattern`.
3. **Campo a mais entrando no assinado** (bloco `signature` na raiz, medida desconhecida em `vitals`, chave nova em `changes`, `consultation` dentro do adendo) — o que não está no esquema não pode ser assinado. Testes: `invalid-extra-root-key.json` (`/signature schema`), `invalid-vitals-unknown-key.json` (`/consultation/vitals/mood schema`), `invalid-changes-unknown-key.json` (`/addendum/changes/care_type schema`), `invalid-changes-with-consultation-key.json` (`/consultation schema`) (Task 1).
4. **Texto vazio em vez de `null`** — `""` e `null` dariam dois hashes para "campo vazio"; o esquema exige `null`. Testes: `invalid-subjective-empty.json` (`/consultation/subjective minLength`) e `invalid-text-empty.json` (`/addendum/text minLength`) (Task 1).
5. **Acento, emoji e decimal na serialização** — nome social com acento e emoji, `temperature_c: 36.7`, `weight_kg: 81.5`; o vetor JCS fixa os bytes e o SHA-256 que qualquer canonicalizador tem de reproduzir. Teste: `clinical/examples/canonical/SHA256SUMS` conferido com `shasum -a 256 -c` e o JCS regerado comparado byte a byte (Task 1, Step 8).

## Mapa de arquivos

| Arquivo (repo `contracts`) | Mudança | Task |
|---|---|---|
| `clinical/consultation-v1.json` | novo: esquema da consulta | 1 |
| `clinical/consultation-addendum-v1.json` | novo: esquema do adendo (mesmo `$defs`) | 1 |
| `clinical/examples/consultation/*.json` + `manifest.json` | 28 exemplos (2 válidos, 26 inválidos) | 1 |
| `clinical/examples/consultation-addendum/*.json` + `manifest.json` | 10 exemplos (2 válidos, 8 inválidos) | 1 |
| `clinical/examples/canonical/consultation-full.jcs`, `addendum-structured.jcs`, `SHA256SUMS` | vetor RFC 8785 | 1 |
| `clinical/README.md` | regras do JSON canônico e o comando de verificação | 1 |
| `clinical/CHANGELOG.md` | entrada `clinical-v1.0.0` | 1 |
| `README.md` (raiz) | domínio novo na tabela e parágrafo `### clinical` | 1 |

---

### Task 0: Conferir que o 19a está na main do api

Esta task não escreve nada. O esquema repete valores do código do 19a; se o 19a não fechou ou mudou um valor, o esquema nasce errado.

**Files:** nenhum.

**Interfaces:**
- Consumes: `apps/api` `origin/main` com o 19a mergeado.
- Produces: confirmação dos valores das Global Constraints (ou parada).

- [ ] **Step 1: Confira o 19a em `origin/main` do api**

A partir da raiz do monorepo:

```bash
/opt/homebrew/bin/git -C apps/api fetch origin
/opt/homebrew/bin/git -C apps/api log --oneline -1 origin/main
for f in app/models/consultation.rb app/models/consultation_addendum.rb app/services/ledi/consultation_mapping.rb \
         config/ledi/consultation_mapping.yml app/services/clinical_record/access.rb spec/support/clinical_record_helpers.rb; do
  /opt/homebrew/bin/git -C apps/api cat-file -e "origin/main:$f" && echo "ok $f" || echo "FALTA $f"
done
/opt/homebrew/bin/git -C apps/api grep -n "MAX_TEXT\|MIN_REASON\|MAX_REASON" origin/main -- app/models/consultation.rb app/models/consultation_addendum.rb
/opt/homebrew/bin/git -C apps/api grep -n "CLOSE_OUTCOMES\|COUNCILS =" origin/main -- app/models/attendance.rb app/models/professional.rb
/opt/homebrew/bin/git -C apps/api show origin/main:config/ledi/consultation_mapping.yml | grep -n -A3 "care_types\|conducts\|max_conducts\|max_exams\|exam_group_prefix"
```

Expected: todos `ok`; `MAX_TEXT = 20_000`, `MIN_REASON = 10`, `MAX_REASON = 500`; `CLOSE_OUTCOMES = %w[discharged referred return left]`; `COUNCILS = %w[CRM COREN CRO CRF CRP CREFITO CRN CRFa CRESS CRBM CREF CRMV]`; no YAML, tipos de atendimento 1, 2, 5, 6, condutas 1, 2, 4–12 e 14, 12 condutas e 100 exames no máximo, grupo `02`. **Pare** e reporte ao coordenador se algo `FALTA` (19a não mergeado) ou se um valor divergir (ajuste o esquema e este plano antes de seguir).

- [ ] **Step 2: Confira a main do contracts e crie o worktree**

```bash
/opt/homebrew/bin/git -C contracts fetch origin
/opt/homebrew/bin/git -C contracts log --oneline -1 origin/main
/opt/homebrew/bin/git -C contracts ls-tree --name-only origin/main | grep -x clinical || echo "sem clinical"
/opt/homebrew/bin/git -C contracts tag -l 'clinical-*'
/opt/homebrew/bin/git -C contracts worktree add .claude/mod19b -b feat/mod-19b-signature origin/main
```

Expected: `origin/main` em `0c6c753 fix: route protocol variants by kind to keep triage errors unchanged` ou mais novo; `sem clinical`; nenhuma tag `clinical-*`; worktree criado. Se `clinical/` ou a tag já existirem, pare e reporte.

---

### Task 1: Domínio `clinical` — esquemas, exemplos, vetor JCS, README e CHANGELOG (`clinical-v1.0.0`)

**Files (repo `contracts`, worktree `contracts/.claude/mod19b`):**
- Create: `clinical/consultation-v1.json`, `clinical/consultation-addendum-v1.json`
- Create: `clinical/examples/consultation/` (28 exemplos + `manifest.json`), `clinical/examples/consultation-addendum/` (10 exemplos + `manifest.json`)
- Create: `clinical/examples/canonical/consultation-full.jcs`, `clinical/examples/canonical/addendum-structured.jcs`, `clinical/examples/canonical/SHA256SUMS`
- Create: `clinical/README.md`, `clinical/CHANGELOG.md`
- Modify: `README.md`

**Interfaces:**
- Consumes: o manifesto `{ "cases": [ { "file", "expect" } ] }` e o comando de verificação de `protocols/README.md` (o `json_schemer` do container do `api`).
- Produces (`clinical-v1.0.0`):
  - `clinical/consultation-v1.json` — raiz `{ schema, city, unit, professional, patient, consultation }`; `consultation = { id, started_at, finalized_at, care_type, subjective, objective, assessment, plan, vitals, evaluated_problems, conducts, exam_requests, outcome }`.
  - `clinical/consultation-addendum-v1.json` — raiz `{ schema, city, unit, professional, patient, addendum }`; `addendum = { consultation_id, id, created_at, reason, text, changes, previous_sha256 }`; `changes ⊆ { evaluated_problems, conducts, exam_requests }`.
  - `$defs` comuns (idênticos nos dois): `uuid`, `utc_datetime`, `date`, `cpf`, `clinical_text`, `city {ibge_code|null, name}`, `unit {cnes|null, name}`, `professional {name, cpf, cbo_code, council {name, state, registration_number}}`, `patient {display_name, cpf, birth_date|null}`, `vitals`, `evaluated_problem {terminology, code, label, release, action, onset_on?, onset_precision?}`, `conducts`, `exam_requests [{sigtap_code, label, cid10_justification?}]`, `outcome {code, referral_unit_cnes?, referral_note?}`.
  - Vetor: `clinical/examples/canonical/{consultation-full,addendum-structured}.jcs` e `SHA256SUMS`. Consumido pelo plano do `api` (gerador do JSON canônico e seu spec; o `api` copia os dois esquemas para a pasta de configuração que o plano dele fixar) e pelo `signer` só como bytes opacos.

- [ ] **Step 1: Escreva os exemplos e os manifestos**

A partir da raiz do monorepo:

```bash
mkdir -p contracts/.claude/mod19b/clinical/examples && python3 - contracts/.claude/mod19b/clinical/examples <<'PY'
import copy, json, os, sys

root = sys.argv[1]  # .../clinical/examples
CONS = os.path.join(root, "consultation")
ADD = os.path.join(root, "consultation-addendum")
os.makedirs(CONS, exist_ok=True)
os.makedirs(ADD, exist_ok=True)

HEADER = {
    "city": {"ibge_code": "4106902", "name": "Curitiba"},
    "unit": {"cnes": "0015466", "name": "UBS Ouvidor Pardinho"},
    "professional": {"name": "Ana Beatriz Souza", "cpf": "52998224725", "cbo_code": "225142",
                     "council": {"name": "CRM", "state": "PR", "registration_number": "45678"}},
    "patient": {"display_name": "Joana D'Arc Conceição 👩🏽‍⚕️", "cpf": "39053344705", "birth_date": "1961-03-14"},
}

def consultation_full():
    return {"schema": "rotasaude.consultation.v1", **copy.deepcopy(HEADER), "consultation": {
        "id": "0b6f3c1e-8a2d-4c47-9f1e-2d3a4b5c6d7e",
        "started_at": "2026-10-08T13:20:05Z", "finalized_at": "2026-10-08T13:41:37Z",
        "care_type": 5,
        "subjective": "Refere sede excessiva há 3 semanas.\nNega febre.",
        "objective": "BEG, corada, hidratada.",
        "assessment": "DM2 descompensado; HAS controlada.",
        "plan": "Ajustar metformina; retorno em 30 dias com exames.",
        "vitals": {"systolic": 132, "diastolic": 84, "heart_rate": 78, "respiratory_rate": 16,
                   "temperature_c": 36.7, "spo2": 97, "capillary_glucose": 245, "glucose_moment": "random",
                   "weight_kg": 81.5, "height_cm": 162, "bmi": 31.1, "pain_score": 0},
        "evaluated_problems": [
            {"terminology": "ciap2", "code": "T90", "label": "Diabetes não insulino-dependente", "release": "2",
             "action": "evaluate"},
            {"terminology": "cid10", "code": "I10", "label": "Hipertensão essencial (primária)", "release": "2008",
             "action": "add", "onset_on": "2019-01-01", "onset_precision": "year"}],
        "conducts": [1, 9],
        "exam_requests": [
            {"sigtap_code": "0202010503", "label": "Dosagem de hemoglobina glicosilada", "cid10_justification": "E119"},
            {"sigtap_code": "0202010317", "label": "Dosagem de creatinina"}],
        "outcome": {"code": "referred", "referral_unit_cnes": "2384299", "referral_note": "Avaliação oftalmológica."}}}

def consultation_minimal():
    d = consultation_full()
    d["city"]["ibge_code"] = None
    d["unit"]["cnes"] = None
    d["patient"] = {"display_name": "Maria Silva", "cpf": "11144477735", "birth_date": None}
    c = d["consultation"]
    c.update(care_type=1, subjective=None, objective=None, plan=None, assessment="Sem queixas.", vitals={},
             evaluated_problems=[{"terminology": "ciap2", "code": "A98", "label": "Medicina preventiva",
                                  "release": "2", "action": "evaluate"}],
             conducts=[1], exam_requests=[], outcome={"code": "discharged"})
    return d

def addendum_text_only():
    return {"schema": "rotasaude.consultation_addendum.v1", **copy.deepcopy(HEADER), "addendum": {
        "consultation_id": "0b6f3c1e-8a2d-4c47-9f1e-2d3a4b5c6d7e",
        "id": "5e1d2c3b-4a59-4687-b7c8-d9e0f1a2b3c4",
        "created_at": "2026-10-09T09:02:11Z",
        "reason": "Resultado de exame chegou depois da consulta.",
        "text": "HbA1c 9,2%. Mantida a conduta.",
        "changes": {},
        "previous_sha256": "9f2c4b7e1d0a8c6f3e5b2a1d4c7f0e9b8a6d3c2f1e0b9a8d7c6f5e4d3c2b1a09"}}

def addendum_structured():
    d = addendum_text_only()
    d["professional"] = {"name": "Carla Mendes", "cpf": "16899535009", "cbo_code": "223565",
                         "council": {"name": "COREN", "state": "PR", "registration_number": "123456"}}
    d["addendum"]["changes"] = {
        "evaluated_problems": [{"terminology": "ciap2", "code": "T90", "label": "Diabetes não insulino-dependente",
                                "release": "2", "action": "correct_onset", "onset_on": "2018-06-01", "onset_precision": "month"}],
        "conducts": [1, 9, 12],
        "exam_requests": [{"sigtap_code": "0202010503", "label": "Dosagem de hemoglobina glicosilada"}]}
    return d

def set_path(doc, path, value):
    cur = doc
    for k in path[:-1]:
        cur = cur[k]
    if value is DELETE:
        del cur[path[-1]]
    else:
        cur[path[-1]] = value
    return doc

DELETE = object()

def variant(base, path, value):
    return set_path(base(), path, value)

cons = {
    "consultation-full.json": (consultation_full(), "valid"),
    "consultation-minimal.json": (consultation_minimal(), "valid"),
    "invalid-schema-version.json": (variant(consultation_full, ["schema"], "rotasaude.consultation.v2"), "/schema const"),
    "invalid-addendum-schema-name.json": (variant(consultation_full, ["schema"], "rotasaude.consultation_addendum.v1"), "/schema const"),
    "invalid-extra-root-key.json": (variant(consultation_full, ["signature"], {"mode": "digital"}), "/signature schema"),
    "invalid-without-patient.json": (variant(consultation_full, ["patient"], DELETE), "(root) required"),
    "invalid-patient-cpf-masked.json": (variant(consultation_full, ["patient", "cpf"], "390.533.447-05"), "/patient/cpf pattern"),
    "invalid-professional-without-cpf.json": (variant(consultation_full, ["professional", "cpf"], DELETE), "/professional required"),
    "invalid-council-unknown.json": (variant(consultation_full, ["professional", "council", "name"], "CRX"), "/professional/council/name enum"),
    "invalid-cbo-five-digits.json": (variant(consultation_full, ["professional", "cbo_code"], "22514"), "/professional/cbo_code pattern"),
    "invalid-unit-cnes-short.json": (variant(consultation_full, ["unit", "cnes"], "15466"), "/unit/cnes pattern"),
    "invalid-finalized-at-offset.json": (variant(consultation_full, ["consultation", "finalized_at"], "2026-10-08T10:41:37-03:00"), "/consultation/finalized_at pattern"),
    "invalid-finalized-at-fraction.json": (variant(consultation_full, ["consultation", "finalized_at"], "2026-10-08T13:41:37.123Z"), "/consultation/finalized_at pattern"),
    "invalid-subjective-empty.json": (variant(consultation_full, ["consultation", "subjective"], ""), "/consultation/subjective minLength"),
    "invalid-plan-too-long.json": (variant(consultation_full, ["consultation", "plan"], "x" * 20001), "/consultation/plan maxLength"),
    "invalid-without-plan-key.json": (variant(consultation_full, ["consultation", "plan"], DELETE), "/consultation required"),
    "invalid-care-type-4.json": (variant(consultation_full, ["consultation", "care_type"], 4), "/consultation/care_type enum"),
    "invalid-no-problem.json": (variant(consultation_full, ["consultation", "evaluated_problems"], []), "/consultation/evaluated_problems minItems"),
    "invalid-problem-without-release.json": (set_path(consultation_full(), ["consultation", "evaluated_problems", 0, "release"], DELETE), "/consultation/evaluated_problems/0 required"),
    "invalid-problem-terminology.json": (set_path(consultation_full(), ["consultation", "evaluated_problems", 0, "terminology"], "snomed"), "/consultation/evaluated_problems/0/terminology enum"),
    "invalid-no-conduct.json": (variant(consultation_full, ["consultation", "conducts"], []), "/consultation/conducts minItems"),
    "invalid-conduct-3.json": (variant(consultation_full, ["consultation", "conducts"], [3]), "/consultation/conducts/0 enum"),
    "invalid-conduct-string.json": (variant(consultation_full, ["consultation", "conducts"], ["1"]), "/consultation/conducts/0 enum"),
    "invalid-conduct-duplicate.json": (variant(consultation_full, ["consultation", "conducts"], [1, 1]), "/consultation/conducts uniqueItems"),
    "invalid-exam-group-03.json": (set_path(consultation_full(), ["consultation", "exam_requests", 0, "sigtap_code"], "0301010072"), "/consultation/exam_requests/0/sigtap_code pattern"),
    "invalid-outcome-left.json": (variant(consultation_full, ["consultation", "outcome", "code"], "left"), "/consultation/outcome/code enum"),
    "invalid-vitals-unknown-key.json": (set_path(consultation_full(), ["consultation", "vitals", "mood"], "ok"), "/consultation/vitals/mood schema"),
    "invalid-vitals-systolic-string.json": (set_path(consultation_full(), ["consultation", "vitals", "systolic"], "132"), "/consultation/vitals/systolic integer"),
}
adds = {
    "addendum-text-only.json": (addendum_text_only(), "valid"),
    "addendum-structured.json": (addendum_structured(), "valid"),
    "invalid-consultation-schema-name.json": (variant(addendum_text_only, ["schema"], "rotasaude.consultation.v1"), "/schema const"),
    "invalid-without-previous-sha256.json": (variant(addendum_text_only, ["addendum", "previous_sha256"], DELETE), "/addendum required"),
    "invalid-previous-sha256-uppercase.json": (variant(addendum_text_only, ["addendum", "previous_sha256"], "9F2C4B7E1D0A8C6F3E5B2A1D4C7F0E9B8A6D3C2F1E0B9A8D7C6F5E4D3C2B1A09"), "/addendum/previous_sha256 pattern"),
    "invalid-reason-short.json": (variant(addendum_text_only, ["addendum", "reason"], "curto"), "/addendum/reason minLength"),
    "invalid-text-empty.json": (variant(addendum_text_only, ["addendum", "text"], ""), "/addendum/text minLength"),
    "invalid-changes-unknown-key.json": (variant(addendum_text_only, ["addendum", "changes"], {"care_type": 2}), "/addendum/changes/care_type schema"),
    "invalid-changes-empty-conducts.json": (variant(addendum_text_only, ["addendum", "changes"], {"conducts": []}), "/addendum/changes/conducts minItems"),
    "invalid-changes-with-consultation-key.json": (variant(addendum_text_only, ["consultation"], {"id": "0b6f3c1e-8a2d-4c47-9f1e-2d3a4b5c6d7e"}), "/consultation schema"),
}

for folder, cases in ((CONS, cons), (ADD, adds)):
    manifest = []
    for name, (doc, expect) in cases.items():
        with open(os.path.join(folder, name), "w", encoding="utf-8") as f:
            f.write(json.dumps(doc, indent=2, ensure_ascii=False) + "\n")
        manifest.append({"file": name, "expect": expect})
    with open(os.path.join(folder, "manifest.json"), "w", encoding="utf-8") as f:
        f.write("{\n  \"cases\": [\n" + ",\n".join("    " + json.dumps(c, ensure_ascii=False) for c in manifest) + "\n  ]\n}\n")
print(len(cons), len(adds))
PY
```

Expected: `28 10`. Os dois válidos de cada esquema mais os inválidos, um defeito por arquivo; os manifestos ficam em `clinical/examples/consultation/manifest.json` e `clinical/examples/consultation-addendum/manifest.json`. Os CPFs dos exemplos (52998224725, 39053344705, 11144477735, 16899535009) têm dígito verificador válido e são inventados.

- [ ] **Step 2: Escreva esquemas provisórios e veja a verificação falhar**

```bash
for f in consultation-v1 consultation-addendum-v1; do
  printf '{ "$schema": "https://json-schema.org/draft/2020-12/schema", "type": "object" }\n' > contracts/.claude/mod19b/clinical/$f.json
done
```

Crie o verificador de uso local (fora do repo; é o comando de `protocols/README.md` com os caminhos como argumentos):

```bash
mkdir -p .claude && cat > .claude/check-schema.sh <<'SH'
#!/usr/bin/env bash
# uso: check.sh <schema> <manifest>  (a partir da raiz do monorepo, compose de pé)
python3 -c '
import json, os, sys
schema_path, manifest_path, extras = sys.argv[1], sys.argv[2], sys.argv[3:]
base = os.path.dirname(manifest_path)
cases = [dict(c, doc=json.load(open(os.path.join(base, c["file"])))) for c in json.load(open(manifest_path))["cases"]]
cases += [{"file": p, "expect": "valid", "doc": json.load(open(p))} for p in extras]
print(json.dumps({"schema": json.load(open(schema_path)), "cases": cases}))
' "$1" "$2" \
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
SH
chmod +x .claude/check-schema.sh
.claude/check-schema.sh contracts/.claude/mod19b/clinical/consultation-v1.json contracts/.claude/mod19b/clinical/examples/consultation/manifest.json
.claude/check-schema.sh contracts/.claude/mod19b/clinical/consultation-addendum-v1.json contracts/.claude/mod19b/clinical/examples/consultation-addendum/manifest.json
```

Expected (exit 1 nas duas): os 2 válidos `ok`, todos os inválidos `FAIL ... obtido: valid`; últimas linhas `2/28 casos` e `2/10 casos`.

- [ ] **Step 3: O esquema da consulta**

Substitua `contracts/.claude/mod19b/clinical/consultation-v1.json` por:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://rota-saude/schemas/clinical/consultation-v1.json",
  "title": "rotasaude.consultation.v1",
  "description": "JSON canônico da consulta finalizada (ADR 0032; módulo 19b). É o conteúdo assinado em CAdES destacado AD-RB, serializado em RFC 8785 (JCS): o api gera, o signer assina o hash dos bytes JCS. Só datas UTC com segundos e 'Z'; números sem zero à direita. Fechado (additionalProperties: false) porque o que não está no esquema não pode entrar no que se assina.",
  "type": "object",
  "additionalProperties": false,
  "required": ["schema", "city", "unit", "professional", "patient", "consultation"],
  "properties": {
    "schema": { "const": "rotasaude.consultation.v1" },
    "city": { "$ref": "#/$defs/city" },
    "unit": { "$ref": "#/$defs/unit" },
    "professional": { "$ref": "#/$defs/professional" },
    "patient": { "$ref": "#/$defs/patient" },
    "consultation": {
      "type": "object",
      "additionalProperties": false,
      "required": ["id", "started_at", "finalized_at", "care_type", "subjective", "objective", "assessment", "plan",
                   "vitals", "evaluated_problems", "conducts", "exam_requests", "outcome"],
      "properties": {
        "id": { "$ref": "#/$defs/uuid" },
        "started_at": { "$ref": "#/$defs/utc_datetime" },
        "finalized_at": { "$ref": "#/$defs/utc_datetime" },
        "care_type": { "enum": [1, 2, 5, 6], "description": "Tipo de atendimento (config/ledi/consultation_mapping.yml do api, 19a)." },
        "subjective": { "$ref": "#/$defs/clinical_text" },
        "objective": { "$ref": "#/$defs/clinical_text" },
        "assessment": { "$ref": "#/$defs/clinical_text" },
        "plan": { "$ref": "#/$defs/clinical_text" },
        "vitals": { "$ref": "#/$defs/vitals" },
        "evaluated_problems": { "type": "array", "minItems": 1, "maxItems": 50, "items": { "$ref": "#/$defs/evaluated_problem" } },
        "conducts": { "$ref": "#/$defs/conducts" },
        "exam_requests": { "$ref": "#/$defs/exam_requests" },
        "outcome": { "$ref": "#/$defs/outcome" }
      }
    }
  },
  "$defs": {
    "uuid": { "type": "string", "pattern": "^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$" },
    "utc_datetime": { "type": "string", "pattern": "^[0-9]{4}-[0-9]{2}-[0-9]{2}T[0-9]{2}:[0-9]{2}:[0-9]{2}Z$", "description": "ISO 8601 em UTC, com segundos, sem fração, com 'Z'." },
    "date": { "type": "string", "pattern": "^[0-9]{4}-[0-9]{2}-[0-9]{2}$" },
    "cpf": { "type": "string", "pattern": "^[0-9]{11}$", "description": "Só dígitos, sem máscara." },
    "clinical_text": { "type": ["string", "null"], "minLength": 1, "maxLength": 20000, "description": "null quando o campo ficou vazio; nunca string vazia." },
    "city": {
      "type": "object",
      "additionalProperties": false,
      "required": ["ibge_code", "name"],
      "properties": {
        "ibge_code": { "type": ["string", "null"], "pattern": "^[0-9]{7}$" },
        "name": { "type": "string", "minLength": 1 }
      }
    },
    "unit": {
      "type": "object",
      "additionalProperties": false,
      "required": ["cnes", "name"],
      "properties": {
        "cnes": { "type": ["string", "null"], "pattern": "^[0-9]{7}$" },
        "name": { "type": "string", "minLength": 1 }
      }
    },
    "professional": {
      "type": "object",
      "additionalProperties": false,
      "required": ["name", "cpf", "cbo_code", "council"],
      "properties": {
        "name": { "type": "string", "minLength": 1 },
        "cpf": { "$ref": "#/$defs/cpf" },
        "cbo_code": { "type": "string", "pattern": "^[0-9]{6}$" },
        "council": {
          "type": "object",
          "additionalProperties": false,
          "required": ["name", "state", "registration_number"],
          "properties": {
            "name": { "enum": ["CRM", "COREN", "CRO", "CRF", "CRP", "CREFITO", "CRN", "CRFa", "CRESS", "CRBM", "CREF", "CRMV"] },
            "state": { "type": "string", "pattern": "^[A-Z]{2}$" },
            "registration_number": { "type": "string", "pattern": "^[0-9]{1,10}$" }
          }
        }
      }
    },
    "patient": {
      "type": "object",
      "additionalProperties": false,
      "required": ["display_name", "cpf", "birth_date"],
      "properties": {
        "display_name": { "type": "string", "minLength": 1, "maxLength": 200 },
        "cpf": { "$ref": "#/$defs/cpf" },
        "birth_date": { "anyOf": [{ "$ref": "#/$defs/date" }, { "type": "null" }] }
      }
    },
    "vitals": {
      "type": "object",
      "additionalProperties": false,
      "description": "Só as medidas preenchidas; objeto vazio quando nenhuma. Plausibilidade é regra do api.",
      "properties": {
        "systolic": { "type": "integer" },
        "diastolic": { "type": "integer" },
        "heart_rate": { "type": "integer" },
        "respiratory_rate": { "type": "integer" },
        "temperature_c": { "type": "number" },
        "spo2": { "type": "integer" },
        "capillary_glucose": { "type": "integer" },
        "glucose_moment": { "enum": ["fasting", "postprandial", "random"] },
        "weight_kg": { "type": "number" },
        "height_cm": { "type": "number" },
        "bmi": { "type": "number" },
        "pain_score": { "type": "integer", "minimum": 0, "maximum": 10 }
      }
    },
    "evaluated_problem": {
      "type": "object",
      "additionalProperties": false,
      "required": ["terminology", "code", "label", "release", "action"],
      "properties": {
        "terminology": { "enum": ["ciap2", "cid10"] },
        "code": { "type": "string", "minLength": 1, "maxLength": 8 },
        "label": { "type": "string", "minLength": 1 },
        "release": { "type": "string", "minLength": 1, "description": "Versão da tabela (TerminologyRelease#version) vigente no ato." },
        "action": { "enum": ["evaluate", "add", "resolve", "correct_onset"] },
        "onset_on": { "$ref": "#/$defs/date" },
        "onset_precision": { "enum": ["day", "month", "year"] }
      }
    },
    "conducts": {
      "type": "array",
      "minItems": 1,
      "maxItems": 12,
      "uniqueItems": true,
      "items": { "enum": [1, 2, 4, 5, 6, 7, 8, 9, 10, 11, 12, 14] }
    },
    "exam_requests": {
      "type": "array",
      "maxItems": 100,
      "items": {
        "type": "object",
        "additionalProperties": false,
        "required": ["sigtap_code", "label"],
        "properties": {
          "sigtap_code": { "type": "string", "pattern": "^02[0-9]{8}$" },
          "label": { "type": "string", "minLength": 1 },
          "cid10_justification": { "type": "string", "minLength": 1, "maxLength": 8 }
        }
      }
    },
    "outcome": {
      "type": "object",
      "additionalProperties": false,
      "required": ["code"],
      "properties": {
        "code": { "enum": ["discharged", "referred", "return"] },
        "referral_unit_cnes": { "type": "string", "pattern": "^[0-9]{7}$" },
        "referral_note": { "type": "string", "minLength": 1 }
      }
    }
  }
}
```

- [ ] **Step 4: O esquema do adendo (mesmo `$defs`)**

Substitua `contracts/.claude/mod19b/clinical/consultation-addendum-v1.json` por:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://rota-saude/schemas/clinical/consultation-addendum-v1.json",
  "title": "rotasaude.consultation_addendum.v1",
  "description": "JSON canônico do adendo (ADR 0032; módulo 19b), assinado pelo autor do adendo em CAdES destacado AD-RB, serializado em RFC 8785 (JCS). previous_sha256 encadeia o adendo ao documento anterior da mesma consulta. $defs idêntico ao de consultation-v1.json. Fechado (additionalProperties: false).",
  "type": "object",
  "additionalProperties": false,
  "required": ["schema", "city", "unit", "professional", "patient", "addendum"],
  "properties": {
    "schema": { "const": "rotasaude.consultation_addendum.v1" },
    "city": { "$ref": "#/$defs/city" },
    "unit": { "$ref": "#/$defs/unit" },
    "professional": { "$ref": "#/$defs/professional" },
    "patient": { "$ref": "#/$defs/patient" },
    "addendum": {
      "type": "object",
      "additionalProperties": false,
      "required": ["consultation_id", "id", "created_at", "reason", "text", "changes", "previous_sha256"],
      "properties": {
        "consultation_id": { "$ref": "#/$defs/uuid" },
        "id": { "$ref": "#/$defs/uuid" },
        "created_at": { "$ref": "#/$defs/utc_datetime" },
        "reason": { "type": "string", "minLength": 10, "maxLength": 500 },
        "text": { "type": "string", "minLength": 1, "maxLength": 20000 },
        "changes": {
          "type": "object",
          "additionalProperties": false,
          "description": "Contrato do 19a §9: evaluated_problems = eventos novos; conducts e exam_requests = listas finais, só quando mudaram. Objeto vazio quando o adendo é só texto.",
          "properties": {
            "evaluated_problems": { "type": "array", "minItems": 1, "maxItems": 50, "items": { "$ref": "#/$defs/evaluated_problem" } },
            "conducts": { "$ref": "#/$defs/conducts" },
            "exam_requests": { "$ref": "#/$defs/exam_requests" }
          }
        },
        "previous_sha256": { "type": "string", "pattern": "^[0-9a-f]{64}$", "description": "canonical_sha256 do documento assinado anterior da mesma consulta (consulta ou adendo); sem nenhum assinado, o SHA-256 do JSON canônico da consulta." }
      }
    }
  },
  "$defs": {
    "uuid": { "type": "string", "pattern": "^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$" },
    "utc_datetime": { "type": "string", "pattern": "^[0-9]{4}-[0-9]{2}-[0-9]{2}T[0-9]{2}:[0-9]{2}:[0-9]{2}Z$", "description": "ISO 8601 em UTC, com segundos, sem fração, com 'Z'." },
    "date": { "type": "string", "pattern": "^[0-9]{4}-[0-9]{2}-[0-9]{2}$" },
    "cpf": { "type": "string", "pattern": "^[0-9]{11}$", "description": "Só dígitos, sem máscara." },
    "clinical_text": { "type": ["string", "null"], "minLength": 1, "maxLength": 20000, "description": "null quando o campo ficou vazio; nunca string vazia." },
    "city": {
      "type": "object",
      "additionalProperties": false,
      "required": ["ibge_code", "name"],
      "properties": {
        "ibge_code": { "type": ["string", "null"], "pattern": "^[0-9]{7}$" },
        "name": { "type": "string", "minLength": 1 }
      }
    },
    "unit": {
      "type": "object",
      "additionalProperties": false,
      "required": ["cnes", "name"],
      "properties": {
        "cnes": { "type": ["string", "null"], "pattern": "^[0-9]{7}$" },
        "name": { "type": "string", "minLength": 1 }
      }
    },
    "professional": {
      "type": "object",
      "additionalProperties": false,
      "required": ["name", "cpf", "cbo_code", "council"],
      "properties": {
        "name": { "type": "string", "minLength": 1 },
        "cpf": { "$ref": "#/$defs/cpf" },
        "cbo_code": { "type": "string", "pattern": "^[0-9]{6}$" },
        "council": {
          "type": "object",
          "additionalProperties": false,
          "required": ["name", "state", "registration_number"],
          "properties": {
            "name": { "enum": ["CRM", "COREN", "CRO", "CRF", "CRP", "CREFITO", "CRN", "CRFa", "CRESS", "CRBM", "CREF", "CRMV"] },
            "state": { "type": "string", "pattern": "^[A-Z]{2}$" },
            "registration_number": { "type": "string", "pattern": "^[0-9]{1,10}$" }
          }
        }
      }
    },
    "patient": {
      "type": "object",
      "additionalProperties": false,
      "required": ["display_name", "cpf", "birth_date"],
      "properties": {
        "display_name": { "type": "string", "minLength": 1, "maxLength": 200 },
        "cpf": { "$ref": "#/$defs/cpf" },
        "birth_date": { "anyOf": [{ "$ref": "#/$defs/date" }, { "type": "null" }] }
      }
    },
    "vitals": {
      "type": "object",
      "additionalProperties": false,
      "description": "Só as medidas preenchidas; objeto vazio quando nenhuma. Plausibilidade é regra do api.",
      "properties": {
        "systolic": { "type": "integer" },
        "diastolic": { "type": "integer" },
        "heart_rate": { "type": "integer" },
        "respiratory_rate": { "type": "integer" },
        "temperature_c": { "type": "number" },
        "spo2": { "type": "integer" },
        "capillary_glucose": { "type": "integer" },
        "glucose_moment": { "enum": ["fasting", "postprandial", "random"] },
        "weight_kg": { "type": "number" },
        "height_cm": { "type": "number" },
        "bmi": { "type": "number" },
        "pain_score": { "type": "integer", "minimum": 0, "maximum": 10 }
      }
    },
    "evaluated_problem": {
      "type": "object",
      "additionalProperties": false,
      "required": ["terminology", "code", "label", "release", "action"],
      "properties": {
        "terminology": { "enum": ["ciap2", "cid10"] },
        "code": { "type": "string", "minLength": 1, "maxLength": 8 },
        "label": { "type": "string", "minLength": 1 },
        "release": { "type": "string", "minLength": 1, "description": "Versão da tabela (TerminologyRelease#version) vigente no ato." },
        "action": { "enum": ["evaluate", "add", "resolve", "correct_onset"] },
        "onset_on": { "$ref": "#/$defs/date" },
        "onset_precision": { "enum": ["day", "month", "year"] }
      }
    },
    "conducts": {
      "type": "array",
      "minItems": 1,
      "maxItems": 12,
      "uniqueItems": true,
      "items": { "enum": [1, 2, 4, 5, 6, 7, 8, 9, 10, 11, 12, 14] }
    },
    "exam_requests": {
      "type": "array",
      "maxItems": 100,
      "items": {
        "type": "object",
        "additionalProperties": false,
        "required": ["sigtap_code", "label"],
        "properties": {
          "sigtap_code": { "type": "string", "pattern": "^02[0-9]{8}$" },
          "label": { "type": "string", "minLength": 1 },
          "cid10_justification": { "type": "string", "minLength": 1, "maxLength": 8 }
        }
      }
    },
    "outcome": {
      "type": "object",
      "additionalProperties": false,
      "required": ["code"],
      "properties": {
        "code": { "enum": ["discharged", "referred", "return"] },
        "referral_unit_cnes": { "type": "string", "pattern": "^[0-9]{7}$" },
        "referral_note": { "type": "string", "minLength": 1 }
      }
    }
  }
}
```

- [ ] **Step 5: Rode a verificação e veja passar**

```bash
.claude/check-schema.sh contracts/.claude/mod19b/clinical/consultation-v1.json contracts/.claude/mod19b/clinical/examples/consultation/manifest.json
.claude/check-schema.sh contracts/.claude/mod19b/clinical/consultation-addendum-v1.json contracts/.claude/mod19b/clinical/examples/consultation-addendum/manifest.json
python3 -c '
import json, sys
c = json.load(open(sys.argv[1])); a = json.load(open(sys.argv[2]))
print("defs-iguais" if c["$defs"] == a["$defs"] else "DEFS DIFERENTES")
' contracts/.claude/mod19b/clinical/consultation-v1.json contracts/.claude/mod19b/clinical/consultation-addendum-v1.json
```

Expected: `28/28 casos` e `10/10 casos` (exit 0), e `defs-iguais`. (Conferido ao escrever o plano contra o `json_schemer` do `api-dev`, com estes esquemas e estes exemplos: `28/28` e `10/10`.)

- [ ] **Step 6: Escreva o vetor JCS**

O JCS (RFC 8785) destes exemplos é o `json.dumps` do Python com chaves ordenadas, sem espaços e sem escapar não-ASCII — vale **só** porque todas as chaves são ASCII e os números são inteiros ou decimais curtos (a forma mais curta do Python coincide com a do ECMAScript que o JCS exige). Quem implementar o canonicalizador (o `api`) prova o seu contra estes bytes.

```bash
(cd contracts/.claude/mod19b/clinical/examples && mkdir -p canonical && python3 - <<'PY'
import json
for source, target in (("consultation/consultation-full.json", "canonical/consultation-full.jcs"),
                       ("consultation-addendum/addendum-structured.json", "canonical/addendum-structured.jcs")):
    doc = json.load(open(source, encoding="utf-8"))
    data = json.dumps(doc, sort_keys=True, separators=(",", ":"), ensure_ascii=False, allow_nan=False).encode("utf-8")
    open(target, "wb").write(data)
    print(target, len(data))
PY
)
(cd contracts/.claude/mod19b/clinical/examples/canonical && shasum -a 256 consultation-full.jcs addendum-structured.jcs > SHA256SUMS && cat SHA256SUMS)
```

Expected:
```
canonical/consultation-full.jcs 1600
canonical/addendum-structured.jcs 1068
2a7e6e4c2a99ef899d8c559a6f3267a25f54faf329d636f49fab508848585ac4  consultation-full.jcs
ed005f6c36c0b6d4734b4f6e5bd8b9248fe4cde4be7a4128d3aa6cff132b5b27  addendum-structured.jcs
```
(Valores conferidos ao escrever o plano com estes exemplos. Se o SHA-256 sair diferente, o Step 1 não gerou os exemplos exatamente como aqui: pare e compare.)

- [ ] **Step 7: README e CHANGELOG do domínio**

Crie `contracts/.claude/mod19b/clinical/README.md`:

````markdown
# clinical — JSON canônico assinado (ADR 0032)

O documento que o profissional assina no prontuário. O `api` monta o JSON a partir
da consulta finalizada (ou do adendo), serializa em **RFC 8785 (JCS)** e o `signer`
monta o CAdES destacado AD-RB sobre esses bytes. O `.p7s` e o `.json` exportados
juntos são validáveis no validar.iti.gov.br.

| Arquivo | `schema` | O que é |
|---|---|---|
| `consultation-v1.json` | `rotasaude.consultation.v1` | Consulta finalizada |
| `consultation-addendum-v1.json` | `rotasaude.consultation_addendum.v1` | Adendo (assinado pelo autor do adendo) |

## Regras

- **Fechado em todo nível** (`additionalProperties: false`): o que não está no
  esquema não entra no que se assina. Campo novo = esquema novo (`v2`).
- **Uma representação por valor**, para o JCS dar um só hash:
  - instantes em UTC, com segundos, sem fração: `2026-10-08T13:41:37Z`;
  - datas `AAAA-MM-DD`; CPF, CNES, IBGE e CBO só com dígitos;
  - texto vazio é `null`, nunca `""`; medida não aferida é chave ausente em `vitals`.
- **Cabeçalho comum**: `city`, `unit`, `professional` (quem assina: o autor da
  consulta ou do adendo, com o CPF que tem de ser o do certificado), `patient`.
  O `$defs` é idêntico nos dois arquivos.
- **Cadeia**: `addendum.previous_sha256` é o SHA-256 (hex minúsculo) do JCS do
  documento assinado anterior da mesma consulta; sem nenhum assinado, o do JCS
  da consulta.
- O esquema não confere plausibilidade de sinais vitais, CID-10 por CBO nem
  relações entre campos: isso é do `api`.

## Exemplos e verificação

`examples/consultation/` e `examples/consultation-addendum/` seguem o manifesto
de `protocols/examples/` (`valid` ou o erro `<data_pointer> <type>` do
`json_schemer`). A partir da raiz do monorepo, com o compose de pé, o comando de
`protocols/README.md` com o esquema e o manifesto de cada um:

```bash
# consulta: troque os dois caminhos do comando de protocols/README.md por
#   contracts/clinical/consultation-v1.json contracts/clinical/examples/consultation/manifest.json
# adendo:
#   contracts/clinical/consultation-addendum-v1.json contracts/clinical/examples/consultation-addendum/manifest.json
```

## Vetor de canonicalização

`examples/canonical/` tem o JCS de `consultation-full.json` e de
`addendum-structured.json` e o `SHA256SUMS`. Todo gerador do JSON canônico tem
de produzir exatamente esses bytes a partir desses documentos
(`shasum -a 256 -c SHA256SUMS` dentro da pasta). O vetor cobre acento e emoji
em UTF-8 cru e decimais (`36.7`, `81.5`, `31.1`).
````

Crie `contracts/.claude/mod19b/clinical/CHANGELOG.md`:

```markdown
# Changelog — clinical

## clinical-v1.0.0 — 2026-10-08 — MAJOR

Primeira versão do domínio (ADR 0032, módulo 19b): o JSON canônico que o
profissional assina no prontuário.

- `consultation-v1.json` (`rotasaude.consultation.v1`): cidade (IBGE), unidade
  (CNES), profissional (nome, CPF, CBO, conselho), paciente (nome de exibição,
  CPF, nascimento) e a consulta finalizada — horários, tipo de atendimento,
  S/O/A/P, sinais vitais, problemas avaliados (terminologia, código, rótulo,
  versão da tabela, ação), condutas, exames (SIGTAP grupo 02) e desfecho.
- `consultation-addendum-v1.json` (`rotasaude.consultation_addendum.v1`): o
  mesmo cabeçalho, com o autor do adendo como profissional, e o adendo —
  motivo, texto, mudanças estruturadas e `previous_sha256` (cadeia).
- Fechados em todo nível; só formas com uma representação (UTC com `Z`, CPF só
  dígitos, `null` para texto vazio), porque o hash assinado é do JCS (RFC 8785).
- `examples/`: 38 exemplos (4 válidos, 34 inválidos) e o vetor de
  canonicalização (`examples/canonical/`, com `SHA256SUMS`).
- Valores tirados do 19a (ADR 0031): tipos de atendimento 1, 2, 5, 6;
  condutas 1, 2, 4–12, 14 (até 12); até 50 problemas e 100 exames; textos até
  20.000; motivo do adendo 10–500; conselhos de `Professional::COUNCILS`.
- Documento assinado nunca muda: mudança de forma será `consultation-v2.json`
  (novo `schema`), com `v1` mantido para verificar o que já foi assinado.
```

- [ ] **Step 8: Rode a prova do vetor**

```bash
(cd contracts/.claude/mod19b/clinical/examples/canonical && shasum -a 256 -c SHA256SUMS)
python3 -c '
import json, sys
doc = json.load(open(sys.argv[1], encoding="utf-8"))
data = json.dumps(doc, sort_keys=True, separators=(",", ":"), ensure_ascii=False, allow_nan=False).encode("utf-8")
print("jcs-igual" if data == open(sys.argv[2], "rb").read() else "JCS DIFERENTE")
' contracts/.claude/mod19b/clinical/examples/consultation/consultation-full.json contracts/.claude/mod19b/clinical/examples/canonical/consultation-full.jcs
grep -c 'Joana D' contracts/.claude/mod19b/clinical/examples/canonical/consultation-full.jcs
```

Expected: `consultation-full.jcs: OK`, `addendum-structured.jcs: OK`, `jcs-igual`, `1` (o nome com acento e emoji saiu em UTF-8 cru, não em `\u`).

- [ ] **Step 9: README da raiz**

Em `contracts/.claude/mod19b/README.md`:

(a) Na frase de abertura de "## Domínios", troque `Cinco domínios` por `Seis domínios`.

(b) Na tabela "Domínios", logo depois da linha de `session/`, acrescente:

```markdown
| [`clinical/`](clinical/README.md) | JSON canônico assinado da consulta e do adendo (ADR 0032) | `clinical-v1.0.0` | Materializado |
```

(c) Logo antes de `### Fora deste repo, por enquanto`, acrescente:

```markdown
### clinical

`consultation-v1.json` e `consultation-addendum-v1.json` (JSON Schema 2020-12)
descrevem o documento que o profissional assina no prontuário (módulo 19b,
ADR 0032): o `api` gera o JSON, serializa em RFC 8785 (JCS) e o `signer` monta
o CAdES destacado sobre esses bytes. Os dois esquemas são fechados e têm o
mesmo `$defs`. Exemplos e o vetor de canonicalização ficam em
`clinical/examples/`; regras e verificação em [`clinical/README.md`](clinical/README.md).
Documento assinado nunca muda: esquema novo é versão nova (`v2`), nunca edição
de `v1`.
```

(d) Em "## Versionamento (ADR 0015)", na lista de prefixos, troque `` `session-vX.Y.Z`, `types-vX.Y.Z` `` por `` `session-vX.Y.Z`, `clinical-vX.Y.Z`, `types-vX.Y.Z` ``.

Confira: `grep -n 'clinical' contracts/.claude/mod19b/README.md` mostra a linha da tabela, o parágrafo e o prefixo.

- [ ] **Step 10: Commit**

```bash
/opt/homebrew/bin/git -C contracts/.claude/mod19b add clinical/consultation-v1.json clinical/consultation-addendum-v1.json \
  clinical/examples/consultation clinical/examples/consultation-addendum clinical/examples/canonical \
  clinical/README.md clinical/CHANGELOG.md README.md
/opt/homebrew/bin/git -C contracts/.claude/mod19b status --short
/opt/homebrew/bin/git -C contracts/.claude/mod19b commit -m "feat: add canonical consultation and addendum schemas (clinical-v1.0.0)

New clinical domain with the JSON Schema 2020-12 of the documents the
professional signs (ADR 0032): rotasaude.consultation.v1 and
rotasaude.consultation_addendum.v1, closed at every level and sharing one
\$defs. Only forms with a single representation are accepted (UTC seconds
with Z, digits-only CPF, null for empty text) so the RFC 8785 bytes are
unique. Examples follow the protocols manifest format, plus a JCS test
vector with its SHA-256.

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

Expected: `status --short` lista só os dois esquemas, `README.md`, `clinical/README.md`, `clinical/CHANGELOG.md`, os 38 exemplos, os 2 manifestos e os 3 arquivos de `canonical/` (47 arquivos novos + 1 modificado).

---

### Task 2: Entrega — verificação final e parada antes do merge

**Files:** nenhum novo. Só leitura e, com autorização, merge/tag/push.

**Interfaces:**
- Consumes: a branch `feat/mod-19b-signature` com o commit da Task 1.
- Produces: o hash do commit para os planos do `api` e do `signer` citarem; depois de autorizado, a tag `clinical-v1.0.0` em `origin` (contrato §12, passo 1).

- [ ] **Step 1: Verificação final da branch**

```bash
/opt/homebrew/bin/git -C contracts/.claude/mod19b log --oneline origin/main..HEAD
/opt/homebrew/bin/git -C contracts/.claude/mod19b diff --stat origin/main..HEAD | tail -1
/opt/homebrew/bin/git -C contracts/.claude/mod19b diff origin/main..HEAD -- protocols session events | wc -l
.claude/check-schema.sh contracts/.claude/mod19b/clinical/consultation-v1.json contracts/.claude/mod19b/clinical/examples/consultation/manifest.json | tail -1
.claude/check-schema.sh contracts/.claude/mod19b/clinical/consultation-addendum-v1.json contracts/.claude/mod19b/clinical/examples/consultation-addendum/manifest.json | tail -1
```

Expected: um commit; `48 files changed`; `0` (nada fora de `clinical/` e do README); `28/28 casos` e `10/10 casos`.

- [ ] **Step 2: Pare e peça autorização**

Reporte ao usuário: o hash do commit, os 38 exemplos verdes, o SHA-256 do vetor e as divergências C1–C5 abaixo. Pergunte, uma de cada vez: (1) merge em `main` local; (2) tag anotada `clinical-v1.0.0`; (3) push da `main` e da tag. Antes do push, confira `origin/main..main` (outra sessão pode ter commitado em `contracts`; publique só o desta sessão).

- [ ] **Step 3: Com autorização — merge, tag e push**

```bash
/opt/homebrew/bin/git -C contracts fetch origin
/opt/homebrew/bin/git -C contracts log --oneline main..origin/main
/opt/homebrew/bin/git -C contracts checkout main
/opt/homebrew/bin/git -C contracts merge --ff-only feat/mod-19b-signature
/opt/homebrew/bin/git -C contracts tag -a clinical-v1.0.0 -m "clinical-v1.0.0 — MAJOR"
/opt/homebrew/bin/git -C contracts log --oneline origin/main..main
/opt/homebrew/bin/git -C contracts push origin main
/opt/homebrew/bin/git -C contracts push origin clinical-v1.0.0
/opt/homebrew/bin/git -C contracts worktree remove .claude/mod19b
/opt/homebrew/bin/git -C contracts branch -d feat/mod-19b-signature
```

Expected: `main..origin/main` vazio (senão, pare: rebase da branch sobre a nova `origin/main` e refaça a Task 2 Step 1); merge fast-forward; `origin/main..main` mostra só o commit da Task 1; push da `main` e da tag aceitos (o aviso de "PR obrigatório" passa por bypass, como nos módulos anteriores).

---

## Self-review (feito ao escrever o plano)

1. **Cobertura do contrato §10:** `schema`, `city` (`ibge_code`, `name`), `unit` (`cnes`, `name`), `professional` (`name`, `cpf`, `cbo_code`, `council`), `patient` (`display_name`, `cpf`, `birth_date`), `consultation` (`id`, `started_at`, `finalized_at`, `care_type`, S/O/A/P, `vitals`, `evaluated_problems` com `terminology`/`code`/`release`/`action`, `conducts`, `exam_requests`, `outcome`); adendo `consultation_id`, `id`, `created_at`, `reason`, `text`, `changes`, `previous_sha256`; tag `clinical-v1.0.0` — todos na Task 1 (Steps 3, 4, 10 e Task 2).
2. **Placeholders:** nenhum; os esquemas, o gerador, o verificador e os SHA-256 esperados estão no texto e foram executados ao escrever o plano.
3. **Consistência:** os nomes dos exemplos no gerador, no manifesto e nas expectativas do Review Focus conferem; `$defs` idêntico conferido por comando (Step 5).
4. **Review Focus:** as cinco linhas têm teste na Task 1.

## Divergências propostas ao contrato

- **C1 — caminho do domínio.** O contrato §10 e o brief dizem `schemas/clinical/consultation-v1.json`. O repo organiza cada domínio na raiz (`protocols/`, `session/`, `events/`) e a tag segue o diretório (`<domínio>-vX.Y.Z`). Este plano usa `clinical/consultation-v1.json` e `clinical/consultation-addendum-v1.json` (tag `clinical-v1.0.0`, como pedido). Os planos do `api` e do `signer` devem citar estes caminhos.
- **C2 — forma do adendo.** O §10 lista os campos do adendo soltos. Aqui o adendo tem o mesmo cabeçalho da consulta (`city`, `unit`, `professional` = autor do adendo, `patient`) e o conteúdo dentro de `addendum`, simétrico a `consultation`.
- **C3 — rótulos no assinado.** `evaluated_problems[].label` e `exam_requests[].label` entram no JSON canônico (rótulo vigente no ato). Sem eles, "ver o que foi assinado" (NGS2.02.04; contrato §6, `content`) mostraria só códigos. `release` é a `TerminologyRelease#version` vigente.
- **C4 — formas fixadas.** `council = { name, state, registration_number }` (colunas de `professionals`); `outcome = { code: discharged|referred|return, referral_unit_cnes?, referral_note? }` (de `Attendances::Close`); S/O/A/P `null` quando vazios; `vitals` só com as medidas presentes (nomes do módulo 18); datas UTC `AAAA-MM-DDTHH:MM:SSZ`; CPF só dígitos; `ibge_code`, `cnes` e `birth_date` podem ser `null` (colunas anuláveis hoje). Exames só do grupo 02 (`^02[0-9]{8}$`).
- **C5 — vetor de canonicalização.** `clinical/examples/canonical/` (bytes JCS + `SHA256SUMS`) não está no contrato; é a prova byte a byte para o gerador do `api`.
