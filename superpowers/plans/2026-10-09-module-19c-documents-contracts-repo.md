# Módulo 19c — JSON canônico dos documentos clínicos (repo contracts) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito: nenhum além do que já está em `origin/main`.** O domínio `clinical` existe desde `clinical-v1.0.0` (commit `ff7df1d`, módulo 19b). Este plano é o **passo 1** da ordem de entrega do 19c (contrato §11): o plano do `api` copia o esquema e o vetor daqui. A Task 0 confere a base.

**Goal:** Publicar no repo `contracts` o esquema JSON 2020-12 do JSON canônico do documento clínico (`rotasaude.clinical_document.v1`: atestado, declaração de comparecimento, receita comum e requisição de exames), com exemplos válidos e inválidos por tipo, o vetor de canonicalização RFC 8785 com `SHA256SUMS`, README, CHANGELOG e a tag `clinical-v1.1.0` (F-19.19 a F-19.22, lado assinatura; ADR 0033).

**Architecture:** Um arquivo de esquema novo, `clinical/clinical-document-v1.json`, fechado em todo nível, com o **mesmo cabeçalho** de `consultation-v1.json` (os `$defs` comuns são copiados sem mudança e conferidos iguais por comando) mais `document` (`id`, `kind`, `issued_at`, `replaces_document_id`) e `content`, escolhido por `document.kind` com `if`/`then` (o `json_schemer` aponta o campo exato, sem `oneOf` opaco). Os exemplos saem de um script Python e são verificados com o `json_schemer` do container do `api`, pelo manifesto `{ file, expect }` de `protocols/`, `session/` e do próprio `clinical/`. O vetor JCS acrescenta dois documentos ao `examples/canonical/` existente. É um MINOR: nada do `v1.0.0` muda. Merge, tag anotada e push só com autorização.

**Tech Stack:** JSON Schema draft 2020-12; `json_schemer` (bundle do `api`, só para a verificação); `python3` do host (gerar exemplos, empacotar esquema + exemplos, gerar o JCS); `shasum`.

**Spec:** `docs/superpowers/specs/2026-10-09-module-19c-clinical-documents-design.md` (§4, §5, §6 "Assinatura", §10 passo 1); ADR `docs/adr/0033.md`; contrato entre apps (fonte única dos formatos): `docs/superpowers/plans/2026-10-09-module-19c-documents-contracts.md` §1, §2 e §9 (o que ele ainda não fixa está na seção final "Divergências propostas ao contrato"); padrão do domínio: `contracts/clinical/README.md` e `CHANGELOG.md` em `origin/main` (`clinical-v1.0.0`) e o plano `docs/superpowers/plans/2026-10-08-module-19b-signature-contracts-repo.md`.

## Global Constraints

- Repo `contracts` na raiz do monorepo: `/Users/eduardovrocha/Development/ioit.solutions/rota-saude/contracts` (remote `git@github.com:rotasaude/contracts.git`). Não existe `apps/contracts`.
- Arquivo novo: `clinical/clinical-document-v1.json`, `schema` = `"rotasaude.clinical_document.v1"` (`const`). Tag `clinical-v1.1.0`, anotada, mensagem `clinical-v1.1.0 — MINOR` (esquema novo, nada removido; ADR 0015). Os dois esquemas do `v1.0.0` e os exemplos deles **não mudam** (documento assinado nunca muda).
- Mesmas convenções de `consultation-v1` (contrato §9): instantes em UTC com segundos e `Z`, sem fração; datas `AAAA-MM-DD`; CPF, CNES, IBGE e CBO só dígitos; CNPJ sem máscara, no formato alfanumérico da IN RFB 2.229/2024 (12 caracteres `[0-9A-Z]` + 2 dígitos; o numérico é o caso particular); texto vazio é `null`, nunca `""`; inteiros onde for código (`catmat_code`, `year`, `copies`, `days`, `duration_days`, `position`); `quantity` é número > 0 (até 2 casas decimais no `api`).
- `$defs` comuns com `consultation-v1.json` (`uuid`, `utc_datetime`, `date`, `cpf`, `clinical_text`, `city`, `unit`, `professional`, `patient`, `exam_requests`) **idênticos byte a byte na forma JSON** — conferido por comando.
- Valores do contrato (§1, §2): `kind` ∈ `sick_note` | `attendance_declaration` | `prescription` | `exam_requisition`; `sick_note.type` ∈ `leave` | `companion`; `companion_reason` ∈ `clt_473_x` | `clt_473_xi` | `clt_473_xii` | `other`; `period` ∈ `morning` | `afternoon` | `full_day`; `route` ∈ `oral` | `sublingual` | `topical` | `ophthalmic` | `otic` | `nasal` | `inhalation` | `vaginal` | `rectal` | `intramuscular` | `intravenous` | `subcutaneous` | `other`; `copies` ∈ 1 | 2.
- Regras do domínio que o esquema garante (spec §4–§5): CID no atestado só com `cid_authorized: true`; atestado de acompanhante nunca com CID; afastamento exige dias e início; declaração com período **ou** hora de chegada, nunca os dois; item da receita com catálogo **ou** texto livre, nunca os dois; receita de enfermagem (`nursing_protocol`) exige `city_cnpj` e só itens do catálogo; antimicrobiano = 2 vias e `valid_until`, comum = 1 via; requisição com ao menos um exame, cada um com a competência SIGTAP.
- O esquema **não** confere: CBO autorizado por tipo, protocolo vigente, dose máxima, item controlado, DV do CNPJ, data plausível, relação entre `prescription.antimicrobial` e os itens — isso é do `api`.
- CHANGELOG obrigatório (`clinical/CHANGELOG.md`): sem entrada, não há tag.
- Worktree `contracts/.claude/mod19c`, branch `feat/mod-19c-documents` a partir de `origin/main` (`contracts/.gitignore` já ignora `/.claude/`). Comandos a partir da raiz do monorepo, com o compose de pé (`docker compose ps` mostra `api-dev` no ar).
- Nenhum dado real nos exemplos: nomes fictícios; CPFs e CNPJ com dígito verificador válido, inventados (CPFs 52998224725, 39053344705, 16899535009, os mesmos do `v1.0.0`; CNPJ 11222333000181).
- Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Nunca `git add -A`: caminhos explícitos. Commits em inglês, Conventional Commits com tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Merge na `main`, tag e push só com autorização explícita do usuário, uma etapa de cada vez.

## Review Focus

1. **CID entrando no atestado sem autorização, ou no atestado de acompanhante.** O paciente que não autorizou o diagnóstico teria o CID assinado e impresso; o acompanhante nunca leva CID. O esquema recusa e aponta o campo. Testes: `invalid-sick-note-cid-not-authorized.json` (`/content/cid_authorized const`) e `invalid-sick-note-companion-with-cid.json` (`/content/cid10 schema`) (Task 1).
2. **Conteúdo de um tipo sob o `kind` de outro** (a receita mandada como atestado por um gerador com defeito). O `if`/`then` aplica o conteúdo do `kind` e recusa as chaves alheias, em vez de aceitar qualquer objeto. Teste: `invalid-sick-note-with-prescription-content.json` (`/content/items schema`) (Task 1).
3. **Receita de enfermagem com texto livre ou sem o CNPJ da cidade** (COFEN 801/2026). O assinado seria uma prescrição fora do protocolo. Testes: `invalid-prescription-nurse-free-text.json` (`/content/items/0 required`) e `invalid-prescription-nurse-without-cnpj.json` (`/content required`) (Task 1).
4. **Duas representações para o mesmo valor**: hora com fuso em `issued_at`, CNPJ ou CPF com máscara, código CATMAT como texto (`"BR0267656"`), hora de chegada sem zero à esquerda (`8:15`). Dariam dois hashes para o mesmo documento. Testes: `invalid-issued-at-offset.json`, `invalid-prescription-cnpj-masked.json`, `invalid-professional-cpf-masked.json`, `invalid-prescription-catmat-string.json`, `invalid-declaration-time-format.json` (Task 1).
5. **Acento, emoji e cedilha nos bytes assinados** (nome social com emoji, "Dipirona sódica", "cápsula") e chave ausente vs. `null` (`duration_days` ausente no item de texto livre, `note: null`). O vetor fixa os bytes e o SHA-256. Teste: `examples/canonical/SHA256SUMS` com `shasum -a 256 -c` e o JCS regerado comparado byte a byte (Task 1, Step 8).

---

## Mapa de arquivos

| Arquivo (repo `contracts`) | Mudança | Task |
|---|---|---|
| `clinical/clinical-document-v1.json` | novo: esquema do documento clínico | 1 |
| `clinical/examples/clinical-document/*.json` + `manifest.json` | 46 exemplos (10 válidos, 36 inválidos) | 1 |
| `clinical/examples/canonical/prescription-doctor.jcs`, `sick-note-leave.jcs`, `SHA256SUMS` | vetor RFC 8785 (dois arquivos novos; `SHA256SUMS` ganha duas linhas) | 1 |
| `clinical/README.md` | o documento clínico: tabela, regras, verificação, vetor | 1 |
| `clinical/CHANGELOG.md` | entrada `clinical-v1.1.0` | 1 |
| `README.md` (raiz) | versão e descrição do domínio `clinical` | 1 |

---

### Task 0: Conferir a base do contracts e criar o worktree

Esta task não escreve nada no repo.

**Files:** nenhum.

**Interfaces:**
- Consumes: `contracts` `origin/main` em `ff7df1d` com a tag `clinical-v1.0.0`; o `api` no ar (só para o `json_schemer`).
- Produces: worktree `contracts/.claude/mod19c` na branch `feat/mod-19c-documents`; o verificador `.claude/check-schema.sh` da raiz do monorepo.

- [ ] **Step 1: Confira a main, a tag e os conselhos**

A partir da raiz do monorepo:

```bash
/opt/homebrew/bin/git -C contracts fetch origin --tags
/opt/homebrew/bin/git -C contracts log --oneline -1 origin/main
/opt/homebrew/bin/git -C contracts rev-parse 'clinical-v1.0.0^{commit}'
/opt/homebrew/bin/git -C contracts tag -l 'clinical-*'
/opt/homebrew/bin/git -C contracts ls-tree --name-only origin/main clinical/
/opt/homebrew/bin/git -C apps/api grep -n "COUNCILS =" origin/main -- app/models/professional.rb
```

Expected: `origin/main` em `ff7df1d feat: require SIGTAP competence in signed exam requests` (ou mais novo, sem `clinical/clinical-document-v1.json`); `clinical-v1.0.0` aponta para `ff7df1d5cf2c…`; só a tag `clinical-v1.0.0`; `clinical/` com `CHANGELOG.md`, `README.md`, `consultation-addendum-v1.json`, `consultation-v1.json`, `examples`; `COUNCILS = %w[CRM COREN CRO CRF CRP CREFITO CRN CRFa CRESS CRBM CREF CRMV]`. Se já existir `clinical-v1.1.0` ou `clinical-document-v1.json`, **pare** e reporte. Se `COUNCILS` mudou, pare: o `$defs.professional` copiado do `v1.0.0` ficaria diferente do código.

- [ ] **Step 2: Crie o worktree e confira o verificador**

```bash
/opt/homebrew/bin/git -C contracts worktree add .claude/mod19c -b feat/mod-19c-documents origin/main
test -x .claude/check-schema.sh && echo "verificador ok" || echo "FALTA o verificador"
```

Se faltar o verificador (o plano do 19b o criou em `.claude/check-schema.sh` da raiz do monorepo, fora de qualquer repo), crie-o:

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
```

Depois, a base (os exemplos do `v1.0.0` continuam verdes):

```bash
.claude/check-schema.sh contracts/.claude/mod19c/clinical/consultation-v1.json contracts/.claude/mod19c/clinical/examples/consultation/manifest.json | tail -1
.claude/check-schema.sh contracts/.claude/mod19c/clinical/consultation-addendum-v1.json contracts/.claude/mod19c/clinical/examples/consultation-addendum/manifest.json | tail -1
(cd contracts/.claude/mod19c/clinical/examples/canonical && shasum -a 256 -c SHA256SUMS)
```

Expected: `31/31 casos`, `11/11 casos`, `consultation-full.jcs: OK` e `addendum-structured.jcs: OK`.

---

### Task 1: `clinical-document-v1.json` — esquema, exemplos, vetor JCS, README e CHANGELOG (`clinical-v1.1.0`)

**Files (repo `contracts`, worktree `contracts/.claude/mod19c`):**
- Create: `clinical/clinical-document-v1.json`
- Create: `clinical/examples/clinical-document/` (46 exemplos + `manifest.json`)
- Create: `clinical/examples/canonical/prescription-doctor.jcs`, `clinical/examples/canonical/sick-note-leave.jcs`
- Modify: `clinical/examples/canonical/SHA256SUMS`, `clinical/README.md`, `clinical/CHANGELOG.md`, `README.md`

**Interfaces:**
- Consumes: o manifesto `{ "cases": [ { "file", "expect" } ] }` e o verificador da Task 0; os `$defs` de `clinical/consultation-v1.json` (`clinical-v1.0.0`).
- Produces (`clinical-v1.1.0`), consumido pelo plano do `api` do 19c (gerador do JSON canônico do documento e seu spec; o `api` copia o esquema para `config/clinical/clinical-document-v1.json`, como fez com os do 19b) e pelo `signer` só como bytes opacos:
  - raiz `{ schema, city, unit, professional, patient, document, content }`, todos obrigatórios; `document = { id, kind, issued_at, replaces_document_id|null }`;
  - `content` por `kind`:
    - `sick_note = { type, days?, start_on?, companion_name?, companion_kinship?, companion_reason?, cid10?: { code, label }, cid_authorized, note|null }`;
    - `attendance_declaration = { date, arrived_at?: "HH:MM", left_at?: "HH:MM", period?, unit_name, companion_name?, issuer_registration? }`;
    - `prescription = { items: [ { position, catalog_item?: { id, catmat_code: integer, label, active_ingredient, strength, dosage_form }, free_text?, printed_description, quantity: number, quantity_unit, route, dosage_instructions, duration_days?, continuous, antimicrobial, reason_problem_id? } ], catalog_release|null, nursing_protocol?: { id, title, number, year: integer, version_id }, city_cnpj?, antimicrobial, copies: 1|2, valid_until? }`;
    - `exam_requisition = { exams: [ { sigtap_code, competence, label, cid10_justification? } ], note|null }`;
  - opcional = chave **ausente** quando não se aplica; texto livre que sempre se aplica (`note`) = presente, `null` quando vazio;
  - vetor: `clinical/examples/canonical/{prescription-doctor,sick-note-leave}.jcs`, com as linhas novas no `SHA256SUMS`.

- [ ] **Step 1: Escreva os exemplos e o manifesto**

A partir da raiz do monorepo (o script fica no scratchpad da sessão, nunca no worktree; `$SCRATCH` é o diretório de scratchpad):

```bash
cat > "$SCRATCH/gen_clinical_document_examples.py" <<'PY'
import copy, json, os, sys

root = sys.argv[1]  # .../clinical/examples
DOCS = os.path.join(root, "clinical-document")
os.makedirs(DOCS, exist_ok=True)

HEADER = {
    "city": {"ibge_code": "4106902", "name": "Curitiba"},
    "unit": {"cnes": "0015466", "name": "UBS Ouvidor Pardinho"},
    "professional": {"name": "Ana Beatriz Souza", "cpf": "52998224725", "cbo_code": "225142",
                     "council": {"name": "CRM", "state": "PR", "registration_number": "45678"}},
    "patient": {"display_name": "Joana D'Arc Conceição 👩🏽‍⚕️", "cpf": "39053344705", "birth_date": "1961-03-14"},
}
NURSE = {"name": "Carla Mendes", "cpf": "16899535009", "cbo_code": "223565",
         "council": {"name": "COREN", "state": "PR", "registration_number": "123456"}}


def doc(kind, content, doc_id, replaces=None):
    return {"schema": "rotasaude.clinical_document.v1", **copy.deepcopy(HEADER),
            "document": {"id": doc_id, "kind": kind, "issued_at": "2026-10-09T13:41:37Z",
                         "replaces_document_id": replaces},
            "content": copy.deepcopy(content)}


def sick_note_leave():
    return doc("sick_note", {
        "type": "leave", "days": 3, "start_on": "2026-10-09",
        "cid10": {"code": "J069", "label": "Infecção aguda das vias aéreas superiores não especificada"},
        "cid_authorized": True, "note": None}, "7c1e2a3b-4d5e-4f60-8a1b-2c3d4e5f6a71")


def sick_note_companion():
    return doc("sick_note", {
        "type": "companion", "start_on": "2026-10-09", "companion_name": "Marcos Conceição",
        "companion_kinship": "filho", "companion_reason": "clt_473_xi", "cid_authorized": False,
        "note": "Acompanhou o filho de 4 anos na consulta."}, "7c1e2a3b-4d5e-4f60-8a1b-2c3d4e5f6a72",
        replaces="7c1e2a3b-4d5e-4f60-8a1b-2c3d4e5f6a70")


def declaration_period():
    return doc("attendance_declaration", {
        "date": "2026-10-09", "period": "morning", "unit_name": "UBS Ouvidor Pardinho"},
        "7c1e2a3b-4d5e-4f60-8a1b-2c3d4e5f6a73")


def declaration_times():
    return doc("attendance_declaration", {
        "date": "2026-10-09", "arrived_at": "08:15", "left_at": "10:30", "unit_name": "UBS Ouvidor Pardinho",
        "companion_name": "Marcos Conceição"}, "7c1e2a3b-4d5e-4f60-8a1b-2c3d4e5f6a74")


METFORMIN = {"id": "3f0e1d2c-3b4a-4596-8877-665544332211", "catmat_code": 267656,
             "label": "Metformina, cloridrato 850 mg, comprimido", "active_ingredient": "cloridrato de metformina",
             "strength": "850 mg", "dosage_form": "comprimido"}
DIPYRONE = {"id": "3f0e1d2c-3b4a-4596-8877-665544332212", "catmat_code": 268144,
            "label": "Dipirona sódica 500 mg/mL, solução oral", "active_ingredient": "dipirona sódica",
            "strength": "500 mg/mL", "dosage_form": "solução oral"}
AMOXICILLIN = {"id": "3f0e1d2c-3b4a-4596-8877-665544332213", "catmat_code": 267621,
               "label": "Amoxicilina 500 mg, cápsula", "active_ingredient": "amoxicilina",
               "strength": "500 mg", "dosage_form": "cápsula"}


def prescription_doctor():
    return doc("prescription", {
        "items": [
            {"position": 1, "catalog_item": METFORMIN, "printed_description": "Cloridrato de metformina 850 mg — comprimido",
             "quantity": 60, "quantity_unit": "comprimido", "route": "oral",
             "dosage_instructions": "Tomar 1 comprimido após o café e 1 após o jantar.", "duration_days": 30,
             "continuous": True, "antimicrobial": False, "reason_problem_id": "9a8b7c6d-5e4f-4a3b-9c2d-1e0f9a8b7c6d"},
            {"position": 2, "free_text": "Soro fisiológico 0,9% para lavagem nasal", "printed_description": "Soro fisiológico 0,9% para lavagem nasal",
             "quantity": 1, "quantity_unit": "frasco", "route": "nasal",
             "dosage_instructions": "Aplicar 2 jatos em cada narina 3 vezes ao dia.", "continuous": False,
             "antimicrobial": False}],
        "catalog_release": "2026-10-09", "antimicrobial": False, "copies": 1}, "7c1e2a3b-4d5e-4f60-8a1b-2c3d4e5f6a75")


def prescription_nurse():
    d = doc("prescription", {
        "items": [
            {"position": 1, "catalog_item": DIPYRONE, "printed_description": "Dipirona sódica 500 mg/mL — solução oral",
             "quantity": 1, "quantity_unit": "frasco", "route": "oral",
             "dosage_instructions": "Tomar 20 gotas até de 6 em 6 horas se dor ou febre.", "duration_days": 3,
             "continuous": False, "antimicrobial": False}],
        "catalog_release": "2026-10-09",
        "nursing_protocol": {"id": "5b4a3c2d-1e0f-4a9b-8c7d-6e5f4a3b2c1d", "title": "Protocolo de enfermagem na atenção primária",
                             "number": "007/2025", "year": 2025, "version_id": "5b4a3c2d-1e0f-4a9b-8c7d-6e5f4a3b2c1e"},
        "city_cnpj": "11222333000181", "antimicrobial": False, "copies": 1}, "7c1e2a3b-4d5e-4f60-8a1b-2c3d4e5f6a76")
    d["professional"] = copy.deepcopy(NURSE)
    return d


def prescription_antimicrobial():
    return doc("prescription", {
        "items": [
            {"position": 1, "catalog_item": AMOXICILLIN, "printed_description": "Amoxicilina 500 mg — cápsula",
             "quantity": 21, "quantity_unit": "cápsula", "route": "oral",
             "dosage_instructions": "Tomar 1 cápsula de 8 em 8 horas por 7 dias.", "duration_days": 7,
             "continuous": False, "antimicrobial": True}],
        "catalog_release": "2026-10-09", "antimicrobial": True, "copies": 2, "valid_until": "2026-10-19"},
        "7c1e2a3b-4d5e-4f60-8a1b-2c3d4e5f6a77")


def exam_requisition():
    return doc("exam_requisition", {
        "exams": [
            {"sigtap_code": "0202010503", "competence": "202609", "label": "Dosagem de hemoglobina glicosilada",
             "cid10_justification": "E119"},
            {"sigtap_code": "0202010317", "competence": "202609", "label": "Dosagem de creatinina"}],
        "note": None}, "7c1e2a3b-4d5e-4f60-8a1b-2c3d4e5f6a78")


DELETE = object()


def setp(d, path, value):
    cur = d
    for k in path[:-1]:
        cur = cur[k]
    if value is DELETE:
        del cur[path[-1]]
    else:
        cur[path[-1]] = value
    return d


def v(base, path, value):
    return setp(base(), path, value)


cases = {
    "sick-note-leave.json": (sick_note_leave(), "valid"),
    "sick-note-companion.json": (sick_note_companion(), "valid"),
    "attendance-declaration-period.json": (declaration_period(), "valid"),
    "attendance-declaration-times.json": (declaration_times(), "valid"),
    "prescription-doctor.json": (prescription_doctor(), "valid"),
    "prescription-nurse.json": (prescription_nurse(), "valid"),
    "prescription-antimicrobial.json": (prescription_antimicrobial(), "valid"),
    "exam-requisition.json": (exam_requisition(), "valid"),
    "prescription-nurse-alphanumeric-cnpj.json": (v(prescription_nurse, ["content", "city_cnpj"], "12ABC34501DE35"), "valid"),
    "prescription-fractional-quantity.json": (v(prescription_doctor, ["content", "items", 1, "quantity"], 1.5), "valid"),
    # cabeçalho
    "invalid-schema-name.json": (v(sick_note_leave, ["schema"], "rotasaude.consultation.v1"), "/schema const"),
    "invalid-extra-root-key.json": (v(sick_note_leave, ["signature"], {"mode": "digital"}), "/signature schema"),
    "invalid-kind-unknown.json": (v(sick_note_leave, ["document", "kind"], "referral"), "/document/kind enum"),
    "invalid-issued-at-offset.json": (v(sick_note_leave, ["document", "issued_at"], "2026-10-09T10:41:37-03:00"), "/document/issued_at pattern"),
    "invalid-without-replaces-key.json": (v(sick_note_leave, ["document", "replaces_document_id"], DELETE), "/document required"),
    "invalid-patient-null.json": (v(sick_note_leave, ["patient"], None), "/patient object"),
    "invalid-professional-cpf-masked.json": (v(sick_note_leave, ["professional", "cpf"], "529.982.247-25"), "/professional/cpf pattern"),
    # atestado
    "invalid-sick-note-cid-not-authorized.json": (v(sick_note_leave, ["content", "cid_authorized"], False), "/content/cid_authorized const"),
    "invalid-sick-note-companion-with-cid.json": (v(sick_note_companion, ["content", "cid10"], {"code": "Z762", "label": "Supervisão de saúde de outras crianças"}), "/content/cid10 schema"),
    "invalid-sick-note-leave-without-days.json": (v(sick_note_leave, ["content", "days"], DELETE), "/content required"),
    "invalid-sick-note-leave-zero-days.json": (v(sick_note_leave, ["content", "days"], 0), "/content/days minimum"),
    "invalid-sick-note-leave-with-companion.json": (v(sick_note_leave, ["content", "companion_name"], "Marcos Conceição"), "/content/companion_name schema"),
    "invalid-sick-note-companion-reason-unknown.json": (v(sick_note_companion, ["content", "companion_reason"], "clt_473_iv"), "/content/companion_reason enum"),
    "invalid-sick-note-note-empty.json": (v(sick_note_leave, ["content", "note"], ""), "/content/note minLength"),
    "invalid-sick-note-with-prescription-content.json": (v(sick_note_leave, ["content"], prescription_doctor()["content"]), "/content/items schema"),
    # declaração
    "invalid-declaration-period-and-times.json": (v(declaration_period, ["content", "arrived_at"], "08:15"), "/content/arrived_at schema"),
    "invalid-declaration-no-period-no-time.json": (v(declaration_period, ["content", "period"], DELETE), "/content required"),
    "invalid-declaration-time-format.json": (v(declaration_times, ["content", "arrived_at"], "8:15"), "/content/arrived_at pattern"),
    "invalid-declaration-period-evening.json": (v(declaration_period, ["content", "period"], "evening"), "/content/period enum"),
    # receita
    "invalid-prescription-no-items.json": (v(prescription_doctor, ["content", "items"], []), "/content/items minItems"),
    "invalid-prescription-item-catalog-and-free-text.json": (v(prescription_doctor, ["content", "items", 0, "free_text"], "Metformina"), "/content/items/0/free_text schema"),
    "invalid-prescription-item-without-medication.json": (v(prescription_doctor, ["content", "items", 1, "free_text"], DELETE), "/content/items/1 required"),
    "invalid-prescription-route-unknown.json": (v(prescription_doctor, ["content", "items", 0, "route"], "transdermal"), "/content/items/0/route enum"),
    "invalid-prescription-quantity-zero.json": (v(prescription_doctor, ["content", "items", 0, "quantity"], 0), "/content/items/0/quantity exclusiveMinimum"),
    "invalid-prescription-catmat-string.json": (v(prescription_doctor, ["content", "items", 0, "catalog_item", "catmat_code"], "BR0267656"), "/content/items/0/catalog_item/catmat_code integer"),
    "invalid-prescription-without-catalog-release-key.json": (v(prescription_doctor, ["content", "catalog_release"], DELETE), "/content required"),
    "invalid-prescription-nurse-free-text.json": (v(prescription_nurse, ["content", "items", 0], {
        "position": 1, "free_text": "Dipirona gotas", "printed_description": "Dipirona gotas", "quantity": 1,
        "quantity_unit": "frasco", "route": "oral", "dosage_instructions": "20 gotas se dor.", "continuous": False,
        "antimicrobial": False}), "/content/items/0 required"),
    "invalid-prescription-nurse-without-cnpj.json": (v(prescription_nurse, ["content", "city_cnpj"], DELETE), "/content required"),
    "invalid-prescription-cnpj-masked.json": (v(prescription_nurse, ["content", "city_cnpj"], "11.222.333/0001-81"), "/content/city_cnpj pattern"),
    "invalid-prescription-antimicrobial-one-copy.json": (v(prescription_antimicrobial, ["content", "copies"], 1), "/content/copies const"),
    "invalid-prescription-antimicrobial-without-valid-until.json": (v(prescription_antimicrobial, ["content", "valid_until"], DELETE), "/content required"),
    "invalid-prescription-common-two-copies.json": (v(prescription_doctor, ["content", "copies"], 2), "/content/copies const"),
    # requisição de exames
    "invalid-exam-requisition-empty.json": (v(exam_requisition, ["content", "exams"], []), "/content/exams minItems"),
    "invalid-exam-requisition-without-competence.json": (v(exam_requisition, ["content", "exams", 0, "competence"], DELETE), "/content/exams/0 required"),
    "invalid-exam-requisition-group-03.json": (v(exam_requisition, ["content", "exams", 0, "sigtap_code"], "0301010072"), "/content/exams/0/sigtap_code pattern"),
    "invalid-exam-requisition-without-note-key.json": (v(exam_requisition, ["content", "note"], DELETE), "/content required"),
}

manifest = []
for name, (d, expect) in cases.items():
    with open(os.path.join(DOCS, name), "w", encoding="utf-8") as f:
        f.write(json.dumps(d, indent=2, ensure_ascii=False) + "\n")
    manifest.append({"file": name, "expect": expect})
with open(os.path.join(DOCS, "manifest.json"), "w", encoding="utf-8") as f:
    f.write("{\n  \"cases\": [\n" + ",\n".join("    " + json.dumps(c, ensure_ascii=False) for c in manifest) + "\n  ]\n}\n")
print(len(cases), sum(1 for _, e in cases.values() if e == "valid"))
PY
mkdir -p contracts/.claude/mod19c/clinical/examples
python3 -I "$SCRATCH/gen_clinical_document_examples.py" contracts/.claude/mod19c/clinical/examples
```

Expected: `46 10` (46 exemplos, 10 válidos — entre eles a receita de enfermagem com CNPJ alfanumérico e um item com quantidade fracionária). Um defeito por arquivo inválido; o manifesto fica em `clinical/examples/clinical-document/manifest.json`. O `doc()` copia o conteúdo (`copy.deepcopy`): sem isso, a variante `invalid-prescription-catmat-string.json` estragaria o item compartilhado e os exemplos válidos de receita sairiam inválidos.

- [ ] **Step 2: Escreva um esquema provisório e veja a verificação falhar**

```bash
printf '{ "$schema": "https://json-schema.org/draft/2020-12/schema", "type": "object" }\n' \
  > contracts/.claude/mod19c/clinical/clinical-document-v1.json
.claude/check-schema.sh contracts/.claude/mod19c/clinical/clinical-document-v1.json \
  contracts/.claude/mod19c/clinical/examples/clinical-document/manifest.json | tail -3
```

Expected (exit 1): os 10 válidos `ok`, todos os inválidos `FAIL ... obtido: valid`; última linha `10/46 casos`.

- [ ] **Step 3: Escreva o esquema**

Substitua `contracts/.claude/mod19c/clinical/clinical-document-v1.json` por:

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://rota-saude/schemas/clinical/clinical-document-v1.json",
  "title": "rotasaude.clinical_document.v1",
  "description": "JSON canônico do documento clínico emitido na consulta (ADR 0033; módulo 19c): atestado, declaração de comparecimento, receita comum e requisição de exames. É o conteúdo assinado em CAdES destacado AD-RB, serializado em RFC 8785 (JCS), quando o documento sai em modo digital; o PAdES vai sobre o PDF. Mesmo cabeçalho e mesmas convenções de consultation-v1.json (UTC com segundos e 'Z', CPF/CNES/IBGE/CBO só dígitos, texto vazio null, inteiros onde for código). O conteúdo é escolhido por document.kind. Fechado (additionalProperties: false) porque o que não está no esquema não pode entrar no que se assina.",
  "type": "object",
  "additionalProperties": false,
  "required": ["schema", "city", "unit", "professional", "patient", "document", "content"],
  "properties": {
    "schema": { "const": "rotasaude.clinical_document.v1" },
    "city": { "$ref": "#/$defs/city" },
    "unit": { "$ref": "#/$defs/unit" },
    "professional": { "$ref": "#/$defs/professional" },
    "patient": { "$ref": "#/$defs/patient" },
    "document": {
      "type": "object",
      "additionalProperties": false,
      "required": ["id", "kind", "issued_at", "replaces_document_id"],
      "properties": {
        "id": { "$ref": "#/$defs/uuid" },
        "kind": { "enum": ["sick_note", "attendance_declaration", "prescription", "exam_requisition"] },
        "issued_at": { "$ref": "#/$defs/utc_datetime" },
        "replaces_document_id": { "anyOf": [{ "$ref": "#/$defs/uuid" }, { "type": "null" }], "description": "O documento cancelado que este substitui (\"cancelar e emitir outro\"); null quando não substitui nenhum." }
      }
    },
    "content": { "type": "object" }
  },
  "allOf": [
    {
      "if": { "required": ["document"], "properties": { "document": { "required": ["kind"], "properties": { "kind": { "const": "sick_note" } } } } },
      "then": { "properties": { "content": { "$ref": "#/$defs/sick_note" } } }
    },
    {
      "if": { "required": ["document"], "properties": { "document": { "required": ["kind"], "properties": { "kind": { "const": "attendance_declaration" } } } } },
      "then": { "properties": { "content": { "$ref": "#/$defs/attendance_declaration" } } }
    },
    {
      "if": { "required": ["document"], "properties": { "document": { "required": ["kind"], "properties": { "kind": { "const": "prescription" } } } } },
      "then": { "properties": { "content": { "$ref": "#/$defs/prescription" } } }
    },
    {
      "if": { "required": ["document"], "properties": { "document": { "required": ["kind"], "properties": { "kind": { "const": "exam_requisition" } } } } },
      "then": { "properties": { "content": { "$ref": "#/$defs/exam_requisition" } } }
    }
  ],
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
    "exam_requests": {
      "type": "array",
      "maxItems": 100,
      "items": {
        "type": "object",
        "additionalProperties": false,
        "required": ["sigtap_code", "competence", "label"],
        "properties": {
          "sigtap_code": { "type": "string", "pattern": "^02[0-9]{8}$" },
          "competence": { "type": "string", "pattern": "^[0-9]{4}(0[1-9]|1[0-2])$", "description": "Competência SIGTAP (AAAAMM) da tabela de onde saiu o rótulo, vigente no ato." },
          "label": { "type": "string", "minLength": 1 },
          "cid10_justification": { "type": "string", "minLength": 1, "maxLength": 8 }
        }
      }
    },
    "local_time": { "type": "string", "pattern": "^([01][0-9]|2[0-3]):[0-5][0-9]$", "description": "Hora de parede da cidade (HH:MM, 24 h), no dia de `date`." },
    "person_name": { "type": "string", "minLength": 1, "maxLength": 200 },
    "sick_note": {
      "type": "object",
      "additionalProperties": false,
      "description": "Atestado. leave = afastamento (dias e início); companion = acompanhante (nome, parentesco e motivo da CLT art. 473), nunca com CID. CID só com cid_authorized: true (autorização do paciente registrada).",
      "required": ["type", "cid_authorized", "note"],
      "properties": {
        "type": { "enum": ["leave", "companion"] },
        "days": { "type": "integer", "minimum": 1 },
        "start_on": { "$ref": "#/$defs/date" },
        "companion_name": { "$ref": "#/$defs/person_name" },
        "companion_kinship": { "type": "string", "minLength": 1 },
        "companion_reason": { "enum": ["clt_473_x", "clt_473_xi", "clt_473_xii", "other"] },
        "cid10": {
          "type": "object",
          "additionalProperties": false,
          "required": ["code", "label"],
          "properties": {
            "code": { "type": "string", "minLength": 1, "maxLength": 8 },
            "label": { "type": "string", "minLength": 1 }
          }
        },
        "cid_authorized": { "type": "boolean" },
        "note": { "$ref": "#/$defs/clinical_text" }
      },
      "allOf": [
        {
          "if": { "properties": { "type": { "const": "leave" } } },
          "then": {
            "required": ["days", "start_on"],
            "properties": { "companion_name": false, "companion_kinship": false, "companion_reason": false }
          }
        },
        {
          "if": { "properties": { "type": { "const": "companion" } } },
          "then": {
            "required": ["companion_name", "companion_kinship", "companion_reason"],
            "properties": { "cid10": false }
          }
        },
        {
          "if": { "required": ["cid10"] },
          "then": { "properties": { "cid_authorized": { "const": true } } }
        }
      ]
    },
    "attendance_declaration": {
      "type": "object",
      "additionalProperties": false,
      "description": "Declaração de comparecimento: o dia e o período (manhã, tarde, dia todo) OU a hora de chegada (e a de saída, se houver), nunca os dois.",
      "required": ["date", "unit_name"],
      "properties": {
        "date": { "$ref": "#/$defs/date" },
        "arrived_at": { "$ref": "#/$defs/local_time" },
        "left_at": { "$ref": "#/$defs/local_time" },
        "period": { "enum": ["morning", "afternoon", "full_day"] },
        "unit_name": { "type": "string", "minLength": 1 },
        "companion_name": { "$ref": "#/$defs/person_name" },
        "issuer_registration": { "type": "string", "minLength": 1 }
      },
      "if": { "required": ["period"] },
      "then": { "properties": { "arrived_at": false, "left_at": false } },
      "else": { "required": ["arrived_at"] }
    },
    "catalog_item": {
      "type": "object",
      "additionalProperties": false,
      "required": ["id", "catmat_code", "label", "active_ingredient", "strength", "dosage_form"],
      "properties": {
        "id": { "$ref": "#/$defs/uuid" },
        "catmat_code": { "type": "integer", "minimum": 1, "description": "Código do item no CATMAT (codigoItem do Compras.gov), inteiro." },
        "label": { "type": "string", "minLength": 1 },
        "active_ingredient": { "type": "string", "minLength": 1, "description": "Denominação Comum Brasileira (DCB)." },
        "strength": { "type": "string", "minLength": 1 },
        "dosage_form": { "type": ["string", "null"], "minLength": 1, "description": "Forma farmacêutica; null quando o CATMAT não traz (decisão do usuário, 2026-10-10)." }
      }
    },
    "prescription_item": {
      "type": "object",
      "additionalProperties": false,
      "description": "Item do catálogo OU texto livre, nunca os dois.",
      "required": ["position", "printed_description", "quantity", "quantity_unit", "route", "dosage_instructions", "continuous", "antimicrobial"],
      "properties": {
        "position": { "type": "integer", "minimum": 1 },
        "catalog_item": { "$ref": "#/$defs/catalog_item" },
        "free_text": { "type": "string", "minLength": 1 },
        "printed_description": { "type": "string", "minLength": 1 },
        "quantity": { "type": "number", "exclusiveMinimum": 0, "description": "Quantidade na unidade de quantity_unit; pode ter até 2 casas decimais (o JCS escreve a forma mais curta)." },
        "quantity_unit": { "type": "string", "minLength": 1 },
        "route": { "enum": ["oral", "sublingual", "topical", "ophthalmic", "otic", "nasal", "inhalation", "vaginal", "rectal", "intramuscular", "intravenous", "subcutaneous", "other"] },
        "dosage_instructions": { "type": "string", "minLength": 1 },
        "duration_days": { "type": "integer", "minimum": 1 },
        "continuous": { "type": "boolean" },
        "antimicrobial": { "type": "boolean" },
        "reason_problem_id": { "$ref": "#/$defs/uuid" }
      },
      "if": { "required": ["catalog_item"] },
      "then": { "properties": { "free_text": false } },
      "else": { "required": ["free_text"] }
    },
    "prescription": {
      "type": "object",
      "additionalProperties": false,
      "description": "Receita comum. catalog_release é a versão do catálogo de medicamentos vigente na emissão (null quando todos os itens são texto livre). Receita de enfermagem (nursing_protocol) leva o CNPJ da cidade e só itens do catálogo. Antimicrobiano: 2 vias e validade; senão 1 via.",
      "required": ["items", "catalog_release", "antimicrobial", "copies"],
      "properties": {
        "items": { "type": "array", "minItems": 1, "items": { "$ref": "#/$defs/prescription_item" } },
        "catalog_release": { "type": ["string", "null"], "minLength": 1 },
        "nursing_protocol": {
          "type": "object",
          "additionalProperties": false,
          "required": ["id", "title", "number", "year", "version_id"],
          "properties": {
            "id": { "$ref": "#/$defs/uuid" },
            "title": { "type": "string", "minLength": 1 },
            "number": { "type": "string", "minLength": 1 },
            "year": { "type": "integer", "minimum": 1900, "maximum": 2999 },
            "version_id": { "$ref": "#/$defs/uuid" }
          }
        },
        "city_cnpj": { "type": "string", "pattern": "^[0-9A-Z]{12}[0-9]{2}$", "description": "CNPJ sem máscara: 12 caracteres [0-9A-Z] e 2 dígitos verificadores (IN RFB 2.229/2024; o numérico é o caso particular)." },
        "antimicrobial": { "type": "boolean" },
        "copies": { "enum": [1, 2] },
        "valid_until": { "$ref": "#/$defs/date" }
      },
      "allOf": [
        {
          "if": { "properties": { "antimicrobial": { "const": true } } },
          "then": { "required": ["valid_until"], "properties": { "copies": { "const": 2 } } },
          "else": { "properties": { "copies": { "const": 1 } } }
        },
        {
          "if": { "required": ["nursing_protocol"] },
          "then": {
            "required": ["city_cnpj"],
            "properties": { "items": { "items": { "required": ["catalog_item"] } } }
          }
        }
      ]
    },
    "exam_requisition": {
      "type": "object",
      "additionalProperties": false,
      "description": "Requisição de exames: os pedidos de exame da consulta (SIGTAP grupo 02, com a competência gravada no ato).",
      "required": ["exams", "note"],
      "properties": {
        "exams": { "$ref": "#/$defs/exam_requests", "minItems": 1 },
        "note": { "$ref": "#/$defs/clinical_text" }
      }
    }
  }
}
```

- [ ] **Step 4: Rode a verificação e veja passar**

```bash
.claude/check-schema.sh contracts/.claude/mod19c/clinical/clinical-document-v1.json \
  contracts/.claude/mod19c/clinical/examples/clinical-document/manifest.json
```

Expected: `46/46 casos` (exit 0). Alguns inválidos trazem mais de um erro e isso é esperado — o manifesto exige que o erro esperado **esteja** na lista: `invalid-sick-note-companion-with-cid.json` (também `/content/cid_authorized const`, porque o acompanhante não autorizou) e `invalid-sick-note-with-prescription-content.json` (todas as chaves da receita e o `required` do atestado). (Conferido ao escrever o plano contra o `json_schemer` do `api-dev`, com este esquema e estes exemplos: `46/46`.)

- [ ] **Step 5: Confira o cabeçalho comum e que o `v1.0.0` não mudou**

```bash
python3 -I -c '
import json, sys
c = json.load(open(sys.argv[1]))["$defs"]; d = json.load(open(sys.argv[2]))["$defs"]
shared = [k for k in d if k in c]
print(shared)
print("defs-comuns-iguais" if all(c[k] == d[k] for k in shared) else "DEFS DIFERENTES: " + ", ".join(k for k in shared if c[k] != d[k]))
' contracts/.claude/mod19c/clinical/consultation-v1.json contracts/.claude/mod19c/clinical/clinical-document-v1.json
/opt/homebrew/bin/git -C contracts/.claude/mod19c status --short -- clinical/consultation-v1.json clinical/consultation-addendum-v1.json clinical/examples/consultation clinical/examples/consultation-addendum
```

Expected: `['uuid', 'utc_datetime', 'date', 'cpf', 'clinical_text', 'city', 'unit', 'professional', 'patient', 'exam_requests']`, `defs-comuns-iguais` e o `status` vazio (nada do `v1.0.0` foi tocado).

- [ ] **Step 6: Escreva o vetor JCS**

O JCS (RFC 8785) destes dois exemplos é o `json.dumps` do Python com chaves ordenadas, sem espaços e sem escapar não-ASCII — vale porque todas as chaves são ASCII e todos os números são inteiros (a forma do Python coincide com a do ECMAScript que o JCS exige), como no vetor do `v1.0.0`. Quem implementar o gerador (o `api`) prova o seu contra estes bytes.

```bash
(cd contracts/.claude/mod19c/clinical/examples && python3 -I - <<'PY'
import json
for source, target in (("clinical-document/prescription-doctor.json", "canonical/prescription-doctor.jcs"),
                       ("clinical-document/sick-note-leave.json", "canonical/sick-note-leave.jcs")):
    doc = json.load(open(source, encoding="utf-8"))
    data = json.dumps(doc, sort_keys=True, separators=(",", ":"), ensure_ascii=False, allow_nan=False).encode("utf-8")
    open(target, "wb").write(data)
    print(target, len(data))
PY
)
(cd contracts/.claude/mod19c/clinical/examples/canonical && shasum -a 256 prescription-doctor.jcs sick-note-leave.jcs >> SHA256SUMS && cat SHA256SUMS)
```

Expected:
```
canonical/prescription-doctor.jcs 1515
canonical/sick-note-leave.jcs 747
4caa5160378b1ed8a3a29a6338af1709dd6adbf7744a79e2996e5019d91f4e70  consultation-full.jcs
6ba35a6bd3dab76bdbe515769608fe30473dd3acc15e280672b8b41e54259279  addendum-structured.jcs
62bd59faef736229bfc314adb643a128fa557f4434c6277085f031efa1c170f5  prescription-doctor.jcs
171c46109f960cdebb7c78dba67a29d2d6e548cb84bd29961145a8cb3bc77afc  sick-note-leave.jcs
```
(Valores conferidos ao escrever o plano com estes exemplos. As duas primeiras linhas são do `v1.0.0` e não mudam. Se o SHA-256 novo sair diferente, o Step 1 não gerou os exemplos exatamente como aqui: pare e compare.)

- [ ] **Step 7: README e CHANGELOG do domínio**

Em `contracts/.claude/mod19c/clinical/README.md`:

(a) Troque o título `# clinical — JSON canônico assinado (ADR 0032)` por `# clinical — JSON canônico assinado (ADRs 0032 e 0033)`, e a primeira frase (`O documento que o profissional assina no prontuário. O \`api\` monta o JSON a partir` … `da consulta finalizada (ou do adendo), serializa`) por:

```markdown
O documento que o profissional assina no prontuário. O `api` monta o JSON a partir
da consulta finalizada, do adendo ou do documento clínico emitido na consulta, serializa
```

(as linhas seguintes do parágrafo continuam iguais).

(b) Na tabela, depois da linha do adendo, acrescente:

```markdown
| `clinical-document-v1.json` | `rotasaude.clinical_document.v1` | Documento clínico da consulta: atestado, declaração de comparecimento, receita comum, requisição de exames (ADR 0033) |
```

(c) Em "## Regras", troque `- **Cabeçalho comum**: \`city\`, \`unit\`, \`professional\` (quem assina: o autor da` … `O \`$defs\` é idêntico nos dois arquivos.` por:

```markdown
- **Cabeçalho comum**: `city`, `unit`, `professional` (quem assina: o autor da
  consulta, do adendo ou do documento, com o CPF que tem de ser o do certificado),
  `patient`. O `$defs` é idêntico nos dois arquivos da consulta; o de
  `clinical-document-v1.json` repete, sem mudança, os que tem em comum com eles.
```

(d) Antes de "## Exemplos e verificação", acrescente:

```markdown
## Documento clínico (`clinical-document-v1.json`)

- **Conteúdo por tipo**: `document.kind` escolhe a forma de `content`
  (`sick_note`, `attendance_declaration`, `prescription`, `exam_requisition`); a
  forma de outro tipo é recusada.
- **Ausente × `null`**: campo que não se aplica ao caso (dias no atestado de
  acompanhante, `duration_days` não informada, `cid10` sem autorização) é chave
  **ausente**; texto que sempre se aplica (`note`) está sempre presente e é `null`
  quando vazio.
- **Atestado**: CID só com `cid_authorized: true`; acompanhante (`companion`)
  nunca com CID e sempre com nome, parentesco e motivo da CLT art. 473;
  afastamento (`leave`) com dias e início.
- **Declaração**: o dia mais o período **ou** a hora de chegada (e a de saída,
  se houver), em hora de parede da cidade `HH:MM`. Só a declaração emitida por
  profissional na consulta é assinada; a da recepção sai em papel e não tem JSON
  canônico.
- **Receita**: cada item é do catálogo (com o código CATMAT inteiro) **ou**
  texto livre; `catalog_release` é a versão do catálogo vigente na emissão
  (`null` se todos os itens são texto livre); `quantity` é número (pode ter
  casas decimais; o JCS escreve a forma mais curta). Receita de enfermagem leva
  o protocolo (com a versão) e o CNPJ da cidade (alfanumérico da IN RFB
  2.229/2024, sem máscara), e só itens do catálogo. Antimicrobiano: 2 vias e
  `valid_until`; comum: 1 via.
- **Requisição de exames**: os exames da consulta, cada um com a competência
  SIGTAP gravada no ato (mesmo item de `consultation-v1`).
- O esquema não confere CBO por tipo de documento, protocolo vigente, dose
  máxima, item controlado nem o dígito verificador do CNPJ: isso é do `api`.
```

(e) No bloco de "## Exemplos e verificação", troque a primeira frase por `` `examples/consultation/`, `examples/consultation-addendum/` e `examples/clinical-document/` seguem o manifesto `` (o resto da frase igual) e acrescente ao bloco `bash`, depois das linhas do adendo:

```bash
# documento clínico:
#   contracts/clinical/clinical-document-v1.json contracts/clinical/examples/clinical-document/manifest.json
```

(f) Troque o parágrafo de "## Vetor de canonicalização" por:

```markdown
`examples/canonical/` tem o JCS de `consultation-full.json`,
`addendum-structured.json`, `prescription-doctor.json` e `sick-note-leave.json` e o
`SHA256SUMS`. Todo gerador do JSON canônico tem de produzir exatamente esses bytes
a partir desses documentos (`shasum -a 256 -c SHA256SUMS` dentro da pasta). O
vetor cobre acento, cedilha e emoji em UTF-8 cru, decimais (`36.7`, `81.5`,
`31.1`), `null` e chave ausente (`note`, `duration_days`).
```

No topo de `contracts/.claude/mod19c/clinical/CHANGELOG.md`, logo depois de `# Changelog — clinical`, acrescente:

```markdown

## clinical-v1.1.0 — 2026-10-09 — MINOR

Documento clínico emitido na consulta (ADR 0033, módulo 19c). Nada do
`v1.0.0` muda.

- `clinical-document-v1.json` (`rotasaude.clinical_document.v1`): o mesmo
  cabeçalho da consulta (cidade, unidade, profissional, paciente), o documento
  (`id`, `kind`, `issued_at`, `replaces_document_id`) e o conteúdo do tipo —
  atestado (afastamento ou acompanhante, CID só com autorização), declaração de
  comparecimento (período ou horário), receita comum (itens do CATMAT ou texto
  livre, versão do catálogo, protocolo de enfermagem e CNPJ, antimicrobiano em
  2 vias) e requisição de exames (SIGTAP com competência).
- Fechado em todo nível; conteúdo escolhido por `document.kind`; `$defs`
  comuns idênticos aos de `consultation-v1.json`.
- `examples/clinical-document/`: 46 exemplos (10 válidos, 36 inválidos).
- Vetor de canonicalização: `prescription-doctor.jcs` e `sick-note-leave.jcs`
  em `examples/canonical/`, com as linhas novas no `SHA256SUMS`.
```

- [ ] **Step 8: Rode a prova do vetor e do README**

```bash
(cd contracts/.claude/mod19c/clinical/examples/canonical && shasum -a 256 -c SHA256SUMS)
python3 -I -c '
import json, sys
for source, target in zip(sys.argv[1::2], sys.argv[2::2]):
    doc = json.load(open(source, encoding="utf-8"))
    data = json.dumps(doc, sort_keys=True, separators=(",", ":"), ensure_ascii=False, allow_nan=False).encode("utf-8")
    print(target.rsplit("/", 1)[-1], "jcs-igual" if data == open(target, "rb").read() else "JCS DIFERENTE")
' contracts/.claude/mod19c/clinical/examples/clinical-document/prescription-doctor.json contracts/.claude/mod19c/clinical/examples/canonical/prescription-doctor.jcs \
  contracts/.claude/mod19c/clinical/examples/clinical-document/sick-note-leave.json contracts/.claude/mod19c/clinical/examples/canonical/sick-note-leave.jcs
grep -c 'Joana D' contracts/.claude/mod19c/clinical/examples/canonical/prescription-doctor.jcs
grep -o '"note":null' contracts/.claude/mod19c/clinical/examples/canonical/sick-note-leave.jcs
grep -c 'duration_days' contracts/.claude/mod19c/clinical/examples/canonical/prescription-doctor.jcs
grep -n 'clinical-document' contracts/.claude/mod19c/clinical/README.md
```

Expected: quatro `OK` (os dois do `v1.0.0` e os dois novos), `prescription-doctor.jcs jcs-igual`, `sick-note-leave.jcs jcs-igual`, `1` (nome com acento e emoji em UTF-8 cru), `"note":null`, `1` (`duration_days` só no item do catálogo; ausente no de texto livre), e as linhas do README com `clinical-document` (tabela, seção, verificação, vetor).

- [ ] **Step 9: README da raiz**

Em `contracts/.claude/mod19c/README.md`:

(a) Na tabela "Domínios", troque a linha de `clinical/` por:

```markdown
| [`clinical/`](clinical/README.md) | JSON canônico assinado da consulta, do adendo e dos documentos clínicos (ADRs 0032 e 0033) | `clinical-v1.1.0` | Materializado |
```

(b) No parágrafo `### clinical`, troque a primeira frase (`` `consultation-v1.json` e `consultation-addendum-v1.json` (JSON Schema 2020-12) `` … `ADR 0032): o \`api\` gera o JSON,`) por:

```markdown
`consultation-v1.json`, `consultation-addendum-v1.json` (JSON Schema 2020-12,
ADR 0032) e `clinical-document-v1.json` (ADR 0033, `clinical-v1.1.0`) descrevem o
que o profissional assina no prontuário — a consulta, o adendo e o documento
clínico emitido na consulta (atestado, declaração, receita, requisição de
exames): o `api` gera o JSON,
```

(o resto do parágrafo continua igual).

Confira: `grep -n 'clinical' contracts/.claude/mod19c/README.md` mostra a linha da tabela com `clinical-v1.1.0` e o parágrafo com `clinical-document-v1.json`.

- [ ] **Step 10: Commit**

```bash
/opt/homebrew/bin/git -C contracts/.claude/mod19c add clinical/clinical-document-v1.json clinical/examples/clinical-document \
  clinical/examples/canonical/prescription-doctor.jcs clinical/examples/canonical/sick-note-leave.jcs \
  clinical/examples/canonical/SHA256SUMS clinical/README.md clinical/CHANGELOG.md README.md
/opt/homebrew/bin/git -C contracts/.claude/mod19c status --short
/opt/homebrew/bin/git -C contracts/.claude/mod19c commit -m "feat: add canonical clinical document schema (clinical-v1.1.0)

New rotasaude.clinical_document.v1 schema for the documents issued in the
consultation (ADR 0033): sick note, attendance declaration, common
prescription and exam requisition. Same header and \$defs as the
consultation schema; the content is picked by document.kind with if/then,
so a validator points at the exact field. Closed at every level, with the
rules the signed bytes must carry (CID only when authorized, catalog item
or free text, nursing prescriptions with protocol and city CNPJ,
antimicrobials in two copies). 46 examples and two more JCS vectors.

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

Expected: `status --short` antes do commit lista o esquema, os 47 arquivos de `clinical/examples/clinical-document/` (46 exemplos + manifesto), os dois `.jcs` novos e `SHA256SUMS`, `clinical/README.md`, `clinical/CHANGELOG.md` e `README.md` — 50 novos + 4 modificados.

---

### Task 2: Entrega — verificação final e parada antes do merge

**Files:** nenhum novo. Só leitura e, com autorização, merge/tag/push.

**Interfaces:**
- Consumes: a branch `feat/mod-19c-documents` com o commit da Task 1.
- Produces: o hash do commit para o plano do `api` do 19c citar; depois de autorizado, a tag `clinical-v1.1.0` em `origin` (contrato §11, passo 1).

- [ ] **Step 1: Verificação final da branch**

```bash
/opt/homebrew/bin/git -C contracts/.claude/mod19c log --oneline origin/main..HEAD
/opt/homebrew/bin/git -C contracts/.claude/mod19c diff --stat origin/main..HEAD | tail -1
/opt/homebrew/bin/git -C contracts/.claude/mod19c diff origin/main..HEAD -- protocols session events types design-tokens \
  clinical/consultation-v1.json clinical/consultation-addendum-v1.json clinical/examples/consultation clinical/examples/consultation-addendum | wc -l
.claude/check-schema.sh contracts/.claude/mod19c/clinical/clinical-document-v1.json contracts/.claude/mod19c/clinical/examples/clinical-document/manifest.json | tail -1
.claude/check-schema.sh contracts/.claude/mod19c/clinical/consultation-v1.json contracts/.claude/mod19c/clinical/examples/consultation/manifest.json | tail -1
.claude/check-schema.sh contracts/.claude/mod19c/clinical/consultation-addendum-v1.json contracts/.claude/mod19c/clinical/examples/consultation-addendum/manifest.json | tail -1
(cd contracts/.claude/mod19c/clinical/examples/canonical && shasum -a 256 -c SHA256SUMS)
```

Expected: um commit; `54 files changed`; `0` (nada fora do documento clínico, do vetor e dos READMEs/CHANGELOG); `46/46 casos`, `31/31 casos`, `11/11 casos`; quatro `OK`.

- [ ] **Step 2: Pare e peça autorização**

Reporte ao usuário: o hash do commit, os 46 exemplos verdes (e os 42 do `v1.0.0` intactos), os dois SHA-256 novos do vetor e as divergências C1–C6 abaixo. Pergunte, uma de cada vez: (1) merge em `main` local; (2) tag anotada `clinical-v1.1.0`; (3) push da `main` e da tag. Antes do push, confira `origin/main..main` (outra sessão pode ter commitado em `contracts`; publique só o desta sessão).

- [ ] **Step 3: Com autorização — merge, tag e push**

```bash
/opt/homebrew/bin/git -C contracts fetch origin
/opt/homebrew/bin/git -C contracts log --oneline main..origin/main
/opt/homebrew/bin/git -C contracts checkout main
/opt/homebrew/bin/git -C contracts merge --ff-only feat/mod-19c-documents
/opt/homebrew/bin/git -C contracts tag -a clinical-v1.1.0 -m "clinical-v1.1.0 — MINOR"
/opt/homebrew/bin/git -C contracts log --oneline origin/main..main
/opt/homebrew/bin/git -C contracts push origin main
/opt/homebrew/bin/git -C contracts push origin clinical-v1.1.0
/opt/homebrew/bin/git -C contracts worktree remove .claude/mod19c
/opt/homebrew/bin/git -C contracts branch -d feat/mod-19c-documents
```

Expected: `main..origin/main` vazio (senão, pare: rebase da branch sobre a nova `origin/main` e refaça a Task 2 Step 1); merge fast-forward; `origin/main..main` mostra só o commit da Task 1; push da `main` e da tag aceitos (o aviso de "PR obrigatório" passa por bypass, como nos módulos anteriores). Avise a sessão do `api` do 19c que a tag existe: o `config/clinical/clinical-document-v1.json` do `api` é cópia deste arquivo nesta tag.

---

## Self-review (feito ao escrever o plano)

1. **Cobertura do contrato §9 e §2:** `schema`, `city`, `unit`, `professional`, `patient`, `document` (`id`, `kind`, `issued_at`, `replaces_document_id`), `content` dos quatro tipos com os campos do §2, receita com `catmat_code`, `catalog_release` e protocolo; mesmas convenções do `consultation-v1`; vetor em `clinical/examples/canonical/`; tag `clinical-v1.1.0` — Task 1 (Steps 1–9) e Task 2.
2. **Placeholders:** nenhum; o esquema, o gerador, o verificador, as contagens e os SHA-256 esperados estão no texto e foram executados ao escrever o plano (`46/46`, `defs-comuns-iguais`, os dois hashes).
3. **Consistência:** os nomes dos exemplos no gerador, no manifesto, no Review Focus e no Step 8 conferem; `$defs` comuns conferidos iguais por comando (Step 5); contagem de arquivos do commit (50 novos + 4 modificados) e do `diff --stat` (54) batem.
4. **Review Focus:** as cinco linhas têm exemplo no manifesto ou prova do vetor na Task 1.

## Divergências propostas ao contrato

- **C1 — `patient` sempre presente.** O §4 da spec deixa `patient_id` nulo na declaração de quem não é paciente, mas essa declaração é da recepção, que não assina (sai em papel). Só documento de profissional na consulta tem JSON canônico; por isso `patient` é obrigatório e não anulável no esquema. Se um dia a recepção assinar, é `clinical-document-v2`.
- **C2 — ausente × `null`.** O §2 marca opcionais com `?` sem dizer a forma no assinado. Aqui: campo que não se aplica é chave ausente; `note` (atestado e requisição) é sempre presente, `null` quando vazio. O gerador do `api` precisa seguir isso para bater com o vetor.
- **C3 — hora da declaração.** O §2 não diz o formato de `arrived_at`/`left_at`. Aqui são hora de parede da cidade `HH:MM` (24 h) no dia de `date` — o que o papel imprime —, não instantes UTC. O `api` e o dashboard devem usar a mesma forma no corpo do POST.
- **C4 — `catmat_code` inteiro e `catalog_release` na receita.** "Inteiros onde for código" (§9) aplicado ao CATMAT (o `codigoItem` da API do Compras.gov é inteiro; o prefixo "BR" do código BR é de exibição). `catalog_release` fica uma vez por receita (a emissão usa uma só versão do catálogo), `null` quando só há texto livre; o §9 não dizia onde.
- **C5 — formas fixadas pelo esquema que o §2 não fixa:** `days` inteiro ≥ 1 e `quantity` número > 0 (o plano do `api` aceita até 2 casas decimais); `city_cnpj` sem máscara no formato alfanumérico da IN RFB 2.229/2024 (`^[0-9A-Z]{12}[0-9]{2}$`, como o `api`); `nursing_protocol.number` texto (ex.: `007/2025`) e `year` inteiro; `cid10 = { code, label }` sem a versão da tabela (como no §2; a CID-10 da cidade é a de 2008); `antimicrobial: true` ⇒ `copies: 2` e `valid_until`, senão `copies: 1`. O `api` gera exatamente isso.
- **C6 — vetor de canonicalização.** O §9 pede o vetor em `clinical/examples/canonical/`; este plano acrescenta `prescription-doctor.jcs` e `sick-note-leave.jcs` ao `SHA256SUMS` existente (sem pasta nova), e os exemplos do documento ficam em `clinical/examples/clinical-document/`, ao lado dos da consulta e do adendo.
- **C7 — Forma do canônico no plano do `api` do 19c.** O plano do `api` (`2026-10-09-module-19c-documents-api.md`, "Valores fixados" 1) descreve um canônico um pouco diferente: `professional.council` anulável; item da receita achatado (`catmat_code?`, `catalog_release?` por item, `free_text: bool`) em vez de `catalog_item { … }` + `free_text` texto; `nursing_protocol { title, number, year, version }` (versão inteira) em vez de `{ id, title, number, year, version_id }`; declaração sem `issuer_registration`. Este esquema segue o §2 do contrato (formas de `<item>` e do protocolo) e o cabeçalho do `consultation-v1` (`council` obrigatório, como em todo documento assinado). O próprio plano do `api` diz que, se o esquema da tag divergir, **o esquema prevalece** e a Task 18 dele se ajusta; o coordenador confirma antes da tag. Já alinhados com o `api` aqui: `quantity` número, CNPJ alfanumérico, `start_on` opcional no atestado de acompanhante, `note` presente e `null` quando vazio, chave opcional ausente, horas `HH:MM`.

## Decisão do usuário (2026-10-10)

`catalog_item.dosage_form` aceita `null` já na `clinical-v1.1.0` (o CATMAT real
não traz a forma em boa parte dos itens, ex.: losartana 50 mg, 268856). O campo
continua obrigatório (presente), com valor string ou `null`. Acrescente aos
exemplos válidos uma receita com item de `dosage_form: null` e ajuste as
contagens de exemplos do plano; os vetores JCS existentes não mudam.
