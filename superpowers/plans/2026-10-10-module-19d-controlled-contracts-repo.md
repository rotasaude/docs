# Módulo 19d — JSON canônico da receita de controlado e de antimicrobiano (repo contracts) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito: 19c entregue em `origin/main`.** Do lado `contracts`, o 19c já está publicado: `origin/main` em `ea08b05` com a tag `clinical-v1.1.0` (`clinical/clinical-document-v1.json`, 51 exemplos, vetor com `prescription-doctor.jcs`, `sick-note-leave.jcs` e `prescription-dosage-form-null.jcs`). Este plano é o **passo 1** da ordem de entrega do 19d (contrato §11) e não depende do `api`, do `dashboard` nem do `maintenance` do 19c estarem em `main` — só do esquema publicado. A Task 0 confere a base e para se ela não estiver como aqui.

**Goal:** Estender o JSON canônico do documento clínico (`rotasaude.clinical_document.v1`) para a receita de controle especial (RCE) e a de antimicrobiano (RET) do 19d — `category`, `sncr`, `patient_identification` e `prescriber_contact` na receita —, com exemplos válidos e inválidos, o vetor JCS com uma RCE e uma RET, README, CHANGELOG e a tag `clinical-v1.2.0` (F-19.26 e F-19.27, lado assinatura; ADR 0034).

**Architecture:** Só o `$defs.prescription` de `clinical/clinical-document-v1.json` muda, e só por acréscimo: quatro chaves opcionais novas e quatro `$defs` novos (`address`, `sncr`, `patient_identification`, `prescriber_contact`). A ausência de `category` é a receita assinada antes do 19d (forma da `clinical-v1.1.0`) e continua válida com os mesmos bytes; com `category`, regras `if`/`then` por categoria (o `json_schemer` aponta o campo exato). A regra de vias muda de "antimicrobiano = 2, senão 1" para "antimicrobiano **ou** controle especial = 2, senão 1", o que não muda nada para os documentos sem `category`. Os exemplos novos saem de um script Python a partir de dois exemplos válidos já publicados e entram no mesmo manifesto; a verificação é a do 19c (`json_schemer` do container do `api`). MINOR: nenhum documento válido na `clinical-v1.1.0` deixa de ser válido. Merge, tag anotada e push só com autorização.

**Tech Stack:** JSON Schema draft 2020-12; `json_schemer` (bundle do `api`, só para a verificação); `python3` do host (patch do esquema, exemplos, JCS); `shasum`.

**Spec:** `docs/superpowers/specs/2026-10-10-module-19d-controlled-prescriptions-design.md` (§5, §6 "Assinatura", §10 passo 1); ADR `docs/adr/0034.md`; contrato entre apps (fonte única dos formatos): `docs/superpowers/plans/2026-10-10-module-19d-controlled-contracts.md` §1, §2 e §9 (o que ele não fixa está em "Divergências propostas ao contrato", no fim); padrão: a tag `clinical-v1.1.0` e o plano `docs/superpowers/plans/2026-10-09-module-19c-documents-contracts-repo.md` (§12 do contrato do 19c prevalece sobre ele).

## Global Constraints

- Repo `contracts` na raiz do monorepo: `/Users/eduardovrocha/Development/ioit.solutions/rota-saude/contracts` (remote `git@github.com:rotasaude/contracts.git`). Não existe `apps/contracts`.
- Arquivo alterado: `clinical/clinical-document-v1.json` (o `schema` continua `"rotasaude.clinical_document.v1"`). Tag `clinical-v1.2.0`, anotada, mensagem `clinical-v1.2.0 — MINOR` (contrato §9). `consultation-v1.json`, `consultation-addendum-v1.json`, os exemplos deles e os **51 exemplos e 5 vetores já publicados não mudam** (documento assinado nunca muda; o `api` fixa os hashes do vetor).
- Valores do contrato (§1, §2): no canônico, `category` ∈ `special_control` | `antimicrobial` e **ausente na receita comum** (uma só representação; o `api` não grava `category: "common"` no canônico — plano do api, D7); `sncr.kind` ∈ `rce` | `ret`; `sncr = { kind, number, simulated }`; `patient_identification = { cpf, address }` (sempre o CPF do cadastro, que existe por regra do 19a; sem "não possui CPF" nem passaporte — decisão do usuário, 2026-10-10); `prescriber_contact = { address, phone }`; `address = { street, number, complement?, district, city, uf, zip }`. `controlled_notification_record` **não** é assinável e não entra no esquema (contrato §9).
- Regras que o esquema garante (spec §5–§6; contrato §2): `special_control` exige `sncr` do tipo `rce`, `patient_identification`, `prescriber_contact`, 2 vias e `valid_until`, `antimicrobial: false` (receita e itens) e nunca protocolo de enfermagem; `antimicrobial` exige `sncr` do tipo `ret`, `prescriber_contact` e `antimicrobial: true` (2 vias e `valid_until` pela regra que já existe), `patient_identification` opcional, nunca protocolo de enfermagem; sem `category` (receita comum, antes ou depois do 19d), nenhum campo do 19d.
- Uma representação por valor (convenção do domínio): CEP só com 8 dígitos; UF com 2 letras maiúsculas; telefone só com dígitos (DDD + número, 10 ou 11); CPF só com dígitos; número SNCR como o SNCR entregou, sem espaço; complemento **ausente** quando não há (nunca `""`); `patient_identification` ausente na RET quando o prescritor não a preencheu.
- O esquema **não** confere: lista da Portaria 344 de cada item, até 3 substâncias C1, duração ≤ 60/180 dias, validade = emissão + 30/10 dias, que o CPF de `patient_identification` é o do cabeçalho, que o número é do prescritor, que `simulated` é falso em produção — isso é do `api`.
- CHANGELOG obrigatório (`clinical/CHANGELOG.md`): sem entrada, não há tag.
- Worktree `contracts/.claude/mod19d`, branch `feat/mod-19d-controlled` a partir de `origin/main` (`contracts/.gitignore` já ignora `/.claude/`). Comandos a partir da raiz do monorepo, com o compose de pé (`docker compose ps` mostra `api-dev` no ar). Os scripts ficam no scratchpad da sessão (`$SCRATCH`), nunca no worktree.
- Nenhum dado real nos exemplos: nomes, endereços e telefone fictícios; CPFs com dígito verificador válido inventados (os mesmos do `v1.0.0`); números SNCR inventados no formato do manual (`AAMM.T-UF.NNNNNNN`, o mesmo do plano do api), marcados `simulated: true`.
- Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Nunca `git add -A`: caminhos explícitos. Commits em inglês, Conventional Commits com tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Merge na `main`, tag e push só com autorização explícita do usuário, uma etapa de cada vez.

## Review Focus

1. **Receita assinada antes do 19d.** Uma receita comum ou de antimicrobiano assinada na `clinical-v1.1.0` (sem `category`) tem de continuar válida e com os mesmos bytes, senão a verificação de um documento já assinado quebra. Testes: os 51 casos antigos continuam `ok` no manifesto (Task 1, Step 4) e os cinco vetores antigos continuam `OK` no `SHA256SUMS` (Task 1, Step 8).
2. **RCE sem número SNCR, sem identificação do paciente ou sem o contato do prescritor.** A farmácia receberia uma receita de controle especial digital que a norma não aceita. Testes: `invalid-special-control-without-sncr.json`, `invalid-special-control-without-patient-identification.json` e `invalid-special-control-without-prescriber-contact.json` (`/content required`); `invalid-ret-without-sncr.json` e `invalid-ret-without-prescriber-contact.json` (Task 1).
3. **Número do tipo errado ou numa receita que não leva número.** Um número de RET numa RCE (ou o contrário), ou um `sncr` numa receita comum ou sem `category`, seria um número gasto à toa e um documento enganoso. Testes: `invalid-special-control-sncr-ret.json`, `invalid-ret-sncr-rce.json` (`/content/sncr/kind const`), `invalid-prescription-without-category-with-sncr.json` (`/content/sncr schema`) e `invalid-prescription-category-common.json` (`/content/category enum`) (Task 1).
4. **Duas representações para o mesmo valor** no endereço e no telefone: CEP com hífen, UF minúscula, telefone com máscara, CPF com máscara, complemento vazio, número SNCR com espaço. Dariam dois hashes para a mesma receita. Testes: `invalid-address-zip-masked.json`, `invalid-address-uf-lowercase.json`, `invalid-prescriber-phone-masked.json`, `invalid-patient-identification-cpf-masked.json`, `invalid-address-complement-empty.json`, `invalid-sncr-number-with-space.json` (Task 1).
5. **Identificação do paciente sem CPF.** A RCE é sempre com o CPF do cadastro: sem a chave, com `null` ou com a chave `no_cpf` de um gerador antigo, a farmácia receberia uma receita sem identificação. Testes: `invalid-patient-identification-without-cpf.json` (`/content/patient_identification required`), `invalid-patient-identification-cpf-null.json` (`/content/patient_identification/cpf string`) e `invalid-patient-identification-no-cpf-key.json` (`/content/patient_identification/no_cpf schema`) (Task 1).

---

## Mapa de arquivos

| Arquivo (repo `contracts`) | Mudança | Task |
|---|---|---|
| `clinical/clinical-document-v1.json` | `$defs.prescription` estendido; `$defs` novos `address`, `sncr`, `patient_identification`, `prescriber_contact`; descrição da raiz | 1 |
| `clinical/examples/clinical-document/*.json` + `manifest.json` | 31 exemplos novos (3 válidos, 28 inválidos); manifesto 51 → 82 casos | 1 |
| `clinical/examples/canonical/prescription-special-control.jcs`, `prescription-ret.jcs`, `SHA256SUMS` | vetor RFC 8785 (dois arquivos novos; `SHA256SUMS` ganha duas linhas) | 1 |
| `clinical/README.md` | regra de evolução (MINOR por acréscimo), receita por categoria, vetor | 1 |
| `clinical/CHANGELOG.md` | entrada `clinical-v1.2.0` | 1 |
| `README.md` (raiz) | versão e descrição do domínio `clinical` | 1 |

**Estratégia de teste:** o manifesto `{ file, expect }` do domínio, rodado pelo verificador `.claude/check-schema.sh` (o `json_schemer` do `api`): primeiro contra o esquema da `clinical-v1.1.0` (vermelho: os casos novos falham — `52/82`), depois contra o esquema estendido (verde: `82/82`). Cada inválido tem um defeito só e o erro esperado (`<data_pointer> <type>`) tem de estar entre os que o validador devolve. Os `$defs` comuns com `consultation-v1.json` e os do 19c que não mudam são conferidos iguais por comando. O vetor JCS é gerado e conferido byte a byte e por `shasum -a 256 -c`. Todos os números esperados abaixo (contagens e SHA-256) foram obtidos executando estes mesmos passos ao escrever o plano.

---

### Task 0: Conferir a base do contracts e criar o worktree

Esta task não escreve nada no repo.

**Files:** nenhum.

**Interfaces:**
- Consumes: `contracts` `origin/main` em `ea08b05` com a tag `clinical-v1.1.0`; o `api` no ar (só para o `json_schemer`); o verificador `.claude/check-schema.sh` da raiz do monorepo (criado no 19b, fora de qualquer repo).
- Produces: worktree `contracts/.claude/mod19d` na branch `feat/mod-19d-controlled`; a base anotada (51/31/11 casos e 5 vetores `OK`).

- [ ] **Step 1: Confira a main, as tags e o esquema publicado**

A partir da raiz do monorepo:

```bash
/opt/homebrew/bin/git -C contracts fetch origin --tags
/opt/homebrew/bin/git -C contracts log --oneline -1 origin/main
/opt/homebrew/bin/git -C contracts rev-parse 'clinical-v1.1.0^{commit}'
/opt/homebrew/bin/git -C contracts tag -l 'clinical-*'
/opt/homebrew/bin/git -C contracts show origin/main:clinical/clinical-document-v1.json | grep -c '"category"\|"sncr"'
/opt/homebrew/bin/git -C contracts show origin/main:clinical/examples/canonical/SHA256SUMS
test -x .claude/check-schema.sh && echo "verificador ok"
```

Expected: `origin/main` em `ea08b05 fix: tighten clinical document schema and examples (clinical-v1.1.0)` (ou mais novo); `clinical-v1.1.0` aponta para `ea08b050e4db…`; as tags `clinical-v1.0.0` e `clinical-v1.1.0` (sem `clinical-v1.2.0`); `0` (o esquema ainda não tem os campos do 19d); as cinco linhas do `SHA256SUMS` (`consultation-full`, `addendum-structured`, `prescription-doctor`, `sick-note-leave`, `prescription-dosage-form-null`); `verificador ok`. Se a `main` estiver mais nova com mudança em `clinical/`, ou se `clinical-v1.2.0` já existir, **pare** e reporte: os SHA-256 e contagens deste plano supõem o esquema de `ea08b05`. Se o verificador faltar, crie-o como no plano do 19c (`2026-10-09-module-19c-documents-contracts-repo.md`, Task 0, Step 2).

- [ ] **Step 2: Crie o worktree e anote a base**

```bash
/opt/homebrew/bin/git -C contracts worktree add .claude/mod19d -b feat/mod-19d-controlled origin/main
W=contracts/.claude/mod19d/clinical
.claude/check-schema.sh $W/clinical-document-v1.json $W/examples/clinical-document/manifest.json | tail -1
.claude/check-schema.sh $W/consultation-v1.json $W/examples/consultation/manifest.json | tail -1
.claude/check-schema.sh $W/consultation-addendum-v1.json $W/examples/consultation-addendum/manifest.json | tail -1
(cd $W/examples/canonical && shasum -a 256 -c SHA256SUMS)
```

Expected: `51/51 casos`, `31/31 casos`, `11/11 casos` e cinco `OK`.

---

### Task 1: `clinical-document-v1.json` com categoria e SNCR — esquema, exemplos, vetor JCS, README e CHANGELOG (`clinical-v1.2.0`)

**Files (repo `contracts`, worktree `contracts/.claude/mod19d`):**
- Modify: `clinical/clinical-document-v1.json`
- Create: 31 arquivos em `clinical/examples/clinical-document/` (lista no Step 1)
- Modify: `clinical/examples/clinical-document/manifest.json`
- Create: `clinical/examples/canonical/prescription-special-control.jcs`, `clinical/examples/canonical/prescription-ret.jcs`
- Modify: `clinical/examples/canonical/SHA256SUMS`, `clinical/README.md`, `clinical/CHANGELOG.md`, `README.md`

**Interfaces:**
- Consumes: o manifesto `{ "cases": [ { "file", "expect" } ] }` e o verificador da Task 0; os exemplos publicados `prescription-doctor.json` e `prescription-antimicrobial.json` (bases dos novos).
- Produces (`clinical-v1.2.0`), consumido pelo plano do `api` do 19d (gerador do JSON canônico da receita e seu spec contra o vetor; o `api` copia o esquema para `config/clinical/clinical-document-v1.json`, como nos 19b/19c) e pelo `signer` só como bytes opacos:
  - `content` da receita ganha, todos opcionais: `category: "special_control"|"antimicrobial"` (ausente na comum), `sncr: { kind: "rce"|"ret", number: "AAMM.T-UF.NNNNNNN", simulated: boolean }`, `patient_identification: { cpf: "<11 dígitos>", address: <address> }`, `prescriber_contact: { address: <address>, phone: "<10–11 dígitos>" }`;
  - `<address> = { street (1–120), number (1–10), complement? (1–60), district (1–80), city (1–80), uf: "^[A-Z]{2}$", zip: "^[0-9]{8}$" }` (os limites do plano do api);
  - regras por categoria do Global Constraints; sem `category` = forma da `clinical-v1.1.0`;
  - vetor: `clinical/examples/canonical/prescription-special-control.jcs` (`25538e7f…`) e `prescription-ret.jcs` (`e4397d40…`), com as linhas novas no `SHA256SUMS`;
  - o que **não** entra no canônico: a lista da Portaria 344 do item (`controlled_list`, `anticonvulsant`), o `paper_reason`, o `controlled_notification_record`.

- [ ] **Step 1: Escreva os exemplos novos e acrescente-os ao manifesto**

A partir da raiz do monorepo (`$SCRATCH` é o diretório de scratchpad da sessão):

```bash
cat > "$SCRATCH/gen_controlled_examples.py" <<'PY'
import copy, json, os, sys

root = sys.argv[1]  # .../clinical/examples
DOCS = os.path.join(root, "clinical-document")


def load(name):
    with open(os.path.join(DOCS, name), encoding="utf-8") as f:
        return json.load(f)


DOCTOR = load("prescription-doctor.json")
ANTIMICROBIAL = load("prescription-antimicrobial.json")

SERTRALINE = {"id": "3f0e1d2c-3b4a-4596-8877-665544332214", "catmat_code": 267503,
              "label": "Sertralina, cloridrato 50 mg, comprimido", "active_ingredient": "cloridrato de sertralina",
              "strength": "50 mg", "dosage_form": "comprimido"}
PATIENT_ADDRESS = {"street": "Rua Marechal Deodoro", "number": "630", "complement": "apto 12",
                   "district": "Centro", "city": "Curitiba", "uf": "PR", "zip": "80010010"}
PRESCRIBER_CONTACT = {"address": {"street": "Rua Ouvidor Pardinho", "number": "28", "district": "Rebouças",
                                  "city": "Curitiba", "uf": "PR", "zip": "80230040"},
                      "phone": "4133501234"}


def special_control():
    d = copy.deepcopy(DOCTOR)
    d["document"]["id"] = "7c1e2a3b-4d5e-4f60-8a1b-2c3d4e5f6a79"
    d["content"] = {
        "items": [
            {"position": 1, "catalog_item": copy.deepcopy(SERTRALINE), "printed_description": "Cloridrato de sertralina 50 mg — comprimido",
             "quantity": 60, "quantity_unit": "comprimido", "route": "oral",
             "dosage_instructions": "Tomar 1 comprimido pela manhã.", "duration_days": 60,
             "continuous": True, "antimicrobial": False}],
        "catalog_release": "2026-10-09", "antimicrobial": False, "copies": 2, "valid_until": "2026-11-08",
        "category": "special_control",
        "sncr": {"kind": "rce", "number": "2610.1-41.0001234", "simulated": True},
        "patient_identification": {"cpf": d["patient"]["cpf"], "address": copy.deepcopy(PATIENT_ADDRESS)},
        "prescriber_contact": copy.deepcopy(PRESCRIBER_CONTACT)}
    return d


def ret():
    d = copy.deepcopy(ANTIMICROBIAL)
    d["document"]["id"] = "7c1e2a3b-4d5e-4f60-8a1b-2c3d4e5f6a7a"
    d["content"]["category"] = "antimicrobial"
    d["content"]["sncr"] = {"kind": "ret", "number": "2610.2-41.0000501", "simulated": True}
    d["content"]["prescriber_contact"] = copy.deepcopy(PRESCRIBER_CONTACT)
    return d


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


def v(base, *changes):
    d = base()
    for path, value in changes:
        setp(d, path, value)
    return d


PI = ["content", "patient_identification"]
PC = ["content", "prescriber_contact"]
cases = {
    "prescription-special-control.json": (special_control(), "valid"),
    "prescription-ret.json": (ret(), "valid"),
    "prescription-ret-with-patient-identification.json": (v(ret, (PI, {"cpf": "39053344705", "address": copy.deepcopy(PATIENT_ADDRESS)})), "valid"),
    # categoria
    "invalid-prescription-category-common.json": (v(lambda: copy.deepcopy(DOCTOR), (["content", "category"], "common")), "/content/category enum"),
    "invalid-prescription-category-unknown.json": (v(special_control, (["content", "category"], "controlled")), "/content/category enum"),
    "invalid-prescription-without-category-with-sncr.json": (v(lambda: copy.deepcopy(DOCTOR), (["content", "sncr"], {"kind": "rce", "number": "2610.1-41.0001234", "simulated": False})), "/content/sncr schema"),
    # receita de controle especial
    "invalid-special-control-without-sncr.json": (v(special_control, (["content", "sncr"], DELETE)), "/content required"),
    "invalid-special-control-sncr-ret.json": (v(special_control, (["content", "sncr", "kind"], "ret")), "/content/sncr/kind const"),
    "invalid-special-control-without-patient-identification.json": (v(special_control, (PI, DELETE)), "/content required"),
    "invalid-special-control-without-prescriber-contact.json": (v(special_control, (PC, DELETE)), "/content required"),
    "invalid-special-control-one-copy.json": (v(special_control, (["content", "copies"], 1)), "/content/copies const"),
    "invalid-special-control-without-valid-until.json": (v(special_control, (["content", "valid_until"], DELETE)), "/content required"),
    "invalid-special-control-antimicrobial-item.json": (v(special_control, (["content", "items", 0, "antimicrobial"], True)), "/content/items/0/antimicrobial const"),
    "invalid-special-control-nursing-protocol.json": (v(special_control, (["content", "nursing_protocol"], {"id": "5b4a3c2d-1e0f-4a9b-8c7d-6e5f4a3b2c1d", "title": "Protocolo", "number": "007/2025", "year": 2025, "version_id": "5b4a3c2d-1e0f-4a9b-8c7d-6e5f4a3b2c1e"}), (["content", "city_cnpj"], "11222333000181")), "/content/nursing_protocol schema"),
    # receita de antimicrobiano
    "invalid-ret-without-sncr.json": (v(ret, (["content", "sncr"], DELETE)), "/content required"),
    "invalid-ret-sncr-rce.json": (v(ret, (["content", "sncr", "kind"], "rce")), "/content/sncr/kind const"),
    "invalid-ret-flag-false.json": (v(ret, (["content", "antimicrobial"], False)), "/content/antimicrobial const"),
    "invalid-ret-without-prescriber-contact.json": (v(ret, (PC, DELETE)), "/content required"),
    # número SNCR
    "invalid-sncr-number-with-space.json": (v(special_control, (["content", "sncr", "number"], "2610.1-41 0001234")), "/content/sncr/number pattern"),
    "invalid-sncr-number-digits-only.json": (v(special_control, (["content", "sncr", "number"], "26101410001234")), "/content/sncr/number pattern"),
    "invalid-sncr-without-simulated.json": (v(special_control, (["content", "sncr", "simulated"], DELETE)), "/content/sncr required"),
    # identificação do paciente
    "invalid-patient-identification-cpf-masked.json": (v(special_control, (PI + ["cpf"], "390.533.447-05")), "/content/patient_identification/cpf pattern"),
    "invalid-patient-identification-without-cpf.json": (v(special_control, (PI + ["cpf"], DELETE)), "/content/patient_identification required"),
    "invalid-patient-identification-cpf-null.json": (v(special_control, (PI + ["cpf"], None)), "/content/patient_identification/cpf string"),
    "invalid-patient-identification-no-cpf-key.json": (v(special_control, (PI + ["no_cpf"], True)), "/content/patient_identification/no_cpf schema"),
    "invalid-patient-identification-extra-key.json": (v(special_control, (PI + ["name"], "Joana")), "/content/patient_identification/name schema"),
    # endereço e telefone
    "invalid-address-zip-masked.json": (v(special_control, (PI + ["address", "zip"], "80010-010")), "/content/patient_identification/address/zip pattern"),
    "invalid-address-uf-lowercase.json": (v(special_control, (PI + ["address", "uf"], "pr")), "/content/patient_identification/address/uf pattern"),
    "invalid-address-complement-empty.json": (v(special_control, (PI + ["address", "complement"], "")), "/content/patient_identification/address/complement minLength"),
    "invalid-address-without-district.json": (v(special_control, (PC + ["address", "district"], DELETE)), "/content/prescriber_contact/address required"),
    "invalid-prescriber-phone-masked.json": (v(special_control, (PC + ["phone"], "(41) 3350-1234")), "/content/prescriber_contact/phone pattern"),
}

manifest_path = os.path.join(DOCS, "manifest.json")
manifest = json.load(open(manifest_path, encoding="utf-8"))["cases"]
known = {c["file"] for c in manifest}
for name, (d, expect) in cases.items():
    assert name not in known, name
    with open(os.path.join(DOCS, name), "w", encoding="utf-8") as f:
        f.write(json.dumps(d, indent=2, ensure_ascii=False) + "\n")
    manifest.append({"file": name, "expect": expect})
with open(manifest_path, "w", encoding="utf-8") as f:
    f.write("{\n  \"cases\": [\n" + ",\n".join("    " + json.dumps(c, ensure_ascii=False) for c in manifest) + "\n  ]\n}\n")
print(len(cases), sum(1 for _, e in cases.values() if e == "valid"), len(manifest))
PY
python3 -I "$SCRATCH/gen_controlled_examples.py" contracts/.claude/mod19d/clinical/examples
```

Expected: `31 3 82` (31 exemplos novos, 3 válidos; o manifesto passa a ter 82 casos). Os 51 casos antigos ficam nas primeiras linhas do manifesto, sem mudança (o script lê o manifesto, confere que nenhum nome novo já existe e acrescenta no fim). Os novos são:
- válidos: `prescription-special-control.json` (RCE com sertralina — lista C1 —, CPF do cadastro, endereço do paciente com complemento, contato do prescritor sem complemento), `prescription-ret.json` (RET com amoxicilina, sem `patient_identification`), `prescription-ret-with-patient-identification.json`;
- inválidos (um defeito cada; erro esperado no manifesto): `invalid-prescription-category-common`, `invalid-prescription-category-unknown`, `invalid-prescription-without-category-with-sncr`, `invalid-special-control-without-sncr`, `invalid-special-control-sncr-ret`, `invalid-special-control-without-patient-identification`, `invalid-special-control-without-prescriber-contact`, `invalid-special-control-one-copy`, `invalid-special-control-without-valid-until`, `invalid-special-control-antimicrobial-item`, `invalid-special-control-nursing-protocol`, `invalid-ret-without-sncr`, `invalid-ret-sncr-rce`, `invalid-ret-flag-false`, `invalid-ret-without-prescriber-contact`, `invalid-sncr-number-with-space`, `invalid-sncr-number-digits-only`, `invalid-sncr-without-simulated`, `invalid-patient-identification-cpf-masked`, `invalid-patient-identification-without-cpf`, `invalid-patient-identification-cpf-null`, `invalid-patient-identification-no-cpf-key`, `invalid-patient-identification-extra-key`, `invalid-address-zip-masked`, `invalid-address-uf-lowercase`, `invalid-address-complement-empty`, `invalid-address-without-district`, `invalid-prescriber-phone-masked` (todos `.json`).

- [ ] **Step 2: Rode a verificação contra o esquema da `v1.1.0` e veja falhar**

```bash
W=contracts/.claude/mod19d/clinical
.claude/check-schema.sh $W/clinical-document-v1.json $W/examples/clinical-document/manifest.json | grep -c '^FAIL'
.claude/check-schema.sh $W/clinical-document-v1.json $W/examples/clinical-document/manifest.json | tail -1
```

Expected (exit 1 no segundo): `30` e `52/82 casos`. Os 51 antigos passam; dos 31 novos, só `invalid-prescription-without-category-with-sncr.json` passa por acaso (o esquema antigo recusa `sncr` como chave desconhecida, `/content/sncr schema`); os três válidos novos falham com `/content/category schema`.

- [ ] **Step 3: Estenda o esquema**

O patch troca o `$defs.prescription` inteiro (do `    "prescription": {` até a linha antes de `    "exam_requisition": {`) pelo bloco novo seguido dos quatro `$defs` novos, e acrescenta a receita do 19d à descrição da raiz. Nada mais no arquivo muda.

```bash
cat > "$SCRATCH/prescription_block.json" <<'EOF'
    "prescription": {
      "type": "object",
      "additionalProperties": false,
      "description": "Receita. catalog_release é a versão do catálogo de medicamentos vigente na emissão (null quando todos os itens são texto livre). Receita de enfermagem (nursing_protocol) leva o CNPJ da cidade e só itens do catálogo. Antimicrobiano: 2 vias e validade; senão 1 via. Desde clinical-v1.2.0 (ADR 0034): category, calculada pelos itens, só na receita de controle especial (special_control, listas C1/C5: número SNCR rce, identificação do paciente, contato do prescritor, 2 vias e validade) e na de antimicrobiano digital (antimicrobial: número SNCR ret e contato do prescritor). Sem category = receita comum, antes ou depois do 19d (forma da clinical-v1.1.0, uma só representação).",
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
        "valid_until": { "$ref": "#/$defs/date" },
        "category": { "enum": ["special_control", "antimicrobial"], "description": "Ausente na receita comum (o api não grava category: common no canônico)." },
        "sncr": { "$ref": "#/$defs/sncr" },
        "patient_identification": { "$ref": "#/$defs/patient_identification" },
        "prescriber_contact": { "$ref": "#/$defs/prescriber_contact" }
      },
      "allOf": [
        {
          "if": {
            "anyOf": [
              { "required": ["antimicrobial"], "properties": { "antimicrobial": { "const": true } } },
              { "required": ["category"], "properties": { "category": { "const": "special_control" } } }
            ]
          },
          "then": { "required": ["valid_until"], "properties": { "copies": { "const": 2 } } },
          "else": { "properties": { "copies": { "const": 1 } } }
        },
        {
          "if": { "required": ["nursing_protocol"] },
          "then": {
            "required": ["city_cnpj"],
            "properties": { "items": { "items": { "required": ["catalog_item"] } } }
          }
        },
        {
          "if": { "not": { "required": ["category"] } },
          "then": { "properties": { "sncr": false, "patient_identification": false, "prescriber_contact": false } }
        },
        {
          "if": { "required": ["category"], "properties": { "category": { "const": "special_control" } } },
          "then": {
            "required": ["sncr", "patient_identification", "prescriber_contact"],
            "properties": {
              "antimicrobial": { "const": false },
              "nursing_protocol": false,
              "sncr": { "properties": { "kind": { "const": "rce" } } },
              "items": { "items": { "properties": { "antimicrobial": { "const": false } } } }
            }
          }
        },
        {
          "if": { "required": ["category"], "properties": { "category": { "const": "antimicrobial" } } },
          "then": {
            "required": ["sncr", "prescriber_contact"],
            "properties": {
              "antimicrobial": { "const": true },
              "nursing_protocol": false,
              "sncr": { "properties": { "kind": { "const": "ret" } } }
            }
          }
        }
      ]
    },
EOF
cat > "$SCRATCH/new_defs.json" <<'EOF'
    "address": {
      "type": "object",
      "additionalProperties": false,
      "description": "Endereço completo (Lei 5.991/1973, art. 35; RDC 1.000/2025): logradouro, número (texto: aceita 's/n'), complemento (ausente quando não há), bairro, cidade, UF e CEP só com dígitos.",
      "required": ["street", "number", "district", "city", "uf", "zip"],
      "properties": {
        "street": { "type": "string", "minLength": 1, "maxLength": 120 },
        "number": { "type": "string", "minLength": 1, "maxLength": 10 },
        "complement": { "type": "string", "minLength": 1, "maxLength": 60 },
        "district": { "type": "string", "minLength": 1, "maxLength": 80 },
        "city": { "type": "string", "minLength": 1, "maxLength": 80 },
        "uf": { "type": "string", "pattern": "^[A-Z]{2}$" },
        "zip": { "type": "string", "pattern": "^[0-9]{8}$" }
      }
    },
    "sncr": {
      "type": "object",
      "additionalProperties": false,
      "description": "Número do SNCR (Anvisa) consumido na emissão: rce na receita de controle especial, ret na de antimicrobiano. number no formato do SNCR AAMM.T-UF.NNNNNNN (ano e mês, 1 RCE ou 2 RET, código IBGE da UF, sequência; ex.: 2610.1-41.0001234). simulated: true quando veio do SNCR simulado (fora de produção, sem validade).",
      "required": ["kind", "number", "simulated"],
      "properties": {
        "kind": { "enum": ["rce", "ret"] },
        "number": { "type": "string", "pattern": "^[0-9]{4}\\.[0-9]-[0-9]{2}\\.[0-9]{7}$" },
        "simulated": { "type": "boolean" }
      }
    },
    "patient_identification": {
      "type": "object",
      "additionalProperties": false,
      "description": "Identificação do paciente na receita de controle especial (e, se o prescritor quiser, na de antimicrobiano): o CPF do cadastro do paciente (sempre existe, regra do 19a) e o endereço completo.",
      "required": ["cpf", "address"],
      "properties": {
        "cpf": { "$ref": "#/$defs/cpf" },
        "address": { "$ref": "#/$defs/address" }
      }
    },
    "prescriber_contact": {
      "type": "object",
      "additionalProperties": false,
      "description": "Endereço completo e telefone do prescritor (perfil do profissional), impressos na receita de controle especial e na de antimicrobiano.",
      "required": ["address", "phone"],
      "properties": {
        "address": { "$ref": "#/$defs/address" },
        "phone": { "type": "string", "pattern": "^[0-9]{10,11}$", "description": "DDD e número, só dígitos." }
      }
    },
EOF
cat > "$SCRATCH/patch_schema.py" <<'PY'
import sys
path, block_path, defs_path = sys.argv[1:4]
text = open(path, encoding="utf-8").read()
start = text.index('    "prescription": {\n')
end = text.index('    "exam_requisition": {\n')
old_desc = "atestado, declaração de comparecimento, receita comum e requisição de exames. É o conteúdo"
assert text.count(old_desc) == 1
new_desc = "atestado, declaração de comparecimento, receita (comum; de controle especial e de antimicrobiano com número SNCR desde clinical-v1.2.0, ADR 0034) e requisição de exames. É o conteúdo"
block = open(block_path, encoding="utf-8").read()
defs = open(defs_path, encoding="utf-8").read()
text = text[:start] + block + defs + text[end:]
text = text.replace(old_desc, new_desc)
open(path, "w", encoding="utf-8").write(text)
print("ok")
PY
python3 -I "$SCRATCH/patch_schema.py" contracts/.claude/mod19d/clinical/clinical-document-v1.json \
  "$SCRATCH/prescription_block.json" "$SCRATCH/new_defs.json"
python3 -I -c 'import json, sys; json.load(open(sys.argv[1])); print("json ok")' contracts/.claude/mod19d/clinical/clinical-document-v1.json
```

Expected: `ok` e `json ok`.

O que o bloco novo faz, regra por regra (para a revisão):
- `allOf[0]` — vias: antimicrobiano **ou** `category: "special_control"` = 2 vias e `valid_until`; senão 1 via. Sem `category` é exatamente a regra da `v1.1.0`.
- `allOf[1]` — enfermagem: igual à `v1.1.0`.
- `allOf[2]` — sem `category` (receita comum): nenhum campo do 19d (`sncr`, `patient_identification`, `prescriber_contact` são `false`). `category` só aceita `special_control` e `antimicrobial`: `"common"` é recusado, para a receita comum ter uma só representação.
- `allOf[3]` — `special_control`: exige `sncr` (`kind: "rce"`), `patient_identification` e `prescriber_contact`; `antimicrobial: false` na receita e em cada item; sem `nursing_protocol`.
- `allOf[4]` — `antimicrobial`: exige `sncr` (`kind: "ret"`) e `prescriber_contact`; `antimicrobial: true`; sem `nursing_protocol`; `patient_identification` opcional.
- `patient_identification`: CPF de 11 dígitos obrigatório e o endereço; nenhuma outra chave (sem `no_cpf` nem passaporte).

- [ ] **Step 4: Rode a verificação e veja passar**

```bash
W=contracts/.claude/mod19d/clinical
.claude/check-schema.sh $W/clinical-document-v1.json $W/examples/clinical-document/manifest.json | grep -v '^ok'
```

Expected: só `82/82 casos` (exit 0). Alguns inválidos trazem mais de um erro, e o manifesto só exige que o esperado **esteja** na lista — entre os novos, `invalid-prescription-category-unknown.json` (também `/content/copies const`: sem categoria conhecida a receita volta a pedir 1 via) e `invalid-ret-flag-false.json` (também `/content/copies const`, porque sem antimicrobiano a receita volta a pedir 1 via). (Conferido ao escrever o plano contra o `json_schemer` do `api-dev`: `82/82`.)

- [ ] **Step 5: Confira o cabeçalho comum, os `$defs` do 19c e que nada fora do esquema da receita mudou**

```bash
python3 -I -c '
import json, sys
c = json.load(open(sys.argv[1]))["$defs"]; d = json.load(open(sys.argv[2]))["$defs"]; o = json.loads(sys.argv[3])["$defs"]
shared = [k for k in d if k in c]
print(shared)
print("defs-comuns-iguais" if all(c[k] == d[k] for k in shared) else "DEFS DIFERENTES: " + ", ".join(k for k in shared if c[k] != d[k]))
print("defs-19c-intactos" if all(o[k] == d[k] for k in o if k != "prescription") else "MUDOU: " + ", ".join(k for k in o if k != "prescription" and o[k] != d[k]))
print(sorted(set(d) - set(o)))
' contracts/.claude/mod19d/clinical/consultation-v1.json contracts/.claude/mod19d/clinical/clinical-document-v1.json \
  "$(/opt/homebrew/bin/git -C contracts show origin/main:clinical/clinical-document-v1.json)"
/opt/homebrew/bin/git -C contracts/.claude/mod19d status --short -- clinical/consultation-v1.json clinical/consultation-addendum-v1.json \
  clinical/examples/consultation clinical/examples/consultation-addendum
/opt/homebrew/bin/git -C contracts/.claude/mod19d diff --stat -- clinical/examples/clinical-document | tail -1
```

Expected: `['uuid', 'utc_datetime', 'date', 'cpf', 'clinical_text', 'city', 'unit', 'professional', 'patient', 'exam_requests']`, `defs-comuns-iguais`, `defs-19c-intactos`, `['address', 'patient_identification', 'prescriber_contact', 'sncr']`; o `status` vazio; e o `diff --stat` de `clinical/examples/clinical-document` com **um** arquivo modificado (o `manifest.json`, só com linhas acrescentadas) — os 51 exemplos publicados não mudaram.

- [ ] **Step 6: Escreva o vetor JCS**

O JCS (RFC 8785) destes dois exemplos é o `json.dumps` do Python com chaves ordenadas, sem espaços e sem escapar não-ASCII — vale porque todas as chaves são ASCII e todos os números são inteiros, como no vetor do 19c.

```bash
(cd contracts/.claude/mod19d/clinical/examples && python3 -I - <<'PY'
import json
for source, target in (("clinical-document/prescription-special-control.json", "canonical/prescription-special-control.jcs"),
                       ("clinical-document/prescription-ret.json", "canonical/prescription-ret.jcs")):
    doc = json.load(open(source, encoding="utf-8"))
    data = json.dumps(doc, sort_keys=True, separators=(",", ":"), ensure_ascii=False, allow_nan=False).encode("utf-8")
    open(target, "wb").write(data)
    print(target, len(data))
PY
)
(cd contracts/.claude/mod19d/clinical/examples/canonical && shasum -a 256 prescription-special-control.jcs prescription-ret.jcs >> SHA256SUMS && cat SHA256SUMS)
```

Expected:
```
canonical/prescription-special-control.jcs 1617
canonical/prescription-ret.jcs 1390
4caa5160378b1ed8a3a29a6338af1709dd6adbf7744a79e2996e5019d91f4e70  consultation-full.jcs
6ba35a6bd3dab76bdbe515769608fe30473dd3acc15e280672b8b41e54259279  addendum-structured.jcs
62bd59faef736229bfc314adb643a128fa557f4434c6277085f031efa1c170f5  prescription-doctor.jcs
171c46109f960cdebb7c78dba67a29d2d6e548cb84bd29961145a8cb3bc77afc  sick-note-leave.jcs
6394b4339ef9a6d9869c5bfe6a96985e7b0812d7323906c8136778dabb31a8a5  prescription-dosage-form-null.jcs
25538e7fb3c406a8a84a785069307297097b338c460d15ed2599cf7df2bb2e0a  prescription-special-control.jcs
e4397d409d2e9bc5a625e6e8bce4ab0d1539e4f1d7f4c2256c7812a95f170464  prescription-ret.jcs
```
(Valores conferidos ao escrever o plano. As cinco primeiras linhas são do 19c e anteriores e não mudam. Se um SHA-256 novo sair diferente, o Step 1 não gerou os exemplos exatamente como aqui: pare e compare.)

- [ ] **Step 7: README e CHANGELOG do domínio**

Em `contracts/.claude/mod19d/clinical/README.md`:

(a) Troque o título `# clinical — JSON canônico assinado (ADRs 0032 e 0033)` por `# clinical — JSON canônico assinado (ADRs 0032, 0033 e 0034)`.

(b) Na tabela, troque a descrição da linha de `clinical-document-v1.json` por `Documento clínico da consulta: atestado, declaração de comparecimento, receita (comum, de controle especial e de antimicrobiano), requisição de exames (ADRs 0033 e 0034)`.

(c) Em "## Regras", troque o primeiro item (`- **Fechado em todo nível** (\`additionalProperties: false\`): o que não está no` / `esquema não entra no que se assina. Campo novo = esquema novo (\`v2\`).`) por:

```markdown
- **Fechado em todo nível** (`additionalProperties: false`): o que não está no
  esquema não entra no que se assina. **Evolução**: acrescentar campo opcional
  compatível (todo documento já assinado continua válido, com os mesmos bytes)
  é MINOR, no mesmo esquema — foi o caso da categoria e do SNCR na receita,
  `clinical-v1.2.0`; só mudança incompatível (campo novo obrigatório, campo
  removido, significado mudado) pede esquema novo (`v2`).
```

(d) Em "## Documento clínico (`clinical-document-v1.json`)", logo depois do item **Receita** (o que termina em `` `valid_until`; comum: 1 via. ``), acrescente:

```markdown
- **Categoria e SNCR (ADR 0034, `clinical-v1.2.0`)**: a receita assinada desde o
  19d traz `category`, calculada pelo `api` a partir dos itens, só quando não é
  comum (a receita comum, antes ou depois do 19d, não tem `category`: uma só
  representação). `special_control` (receita de controle especial, listas C1/C5): o número
  SNCR do tipo `rce` consumido na emissão (`sncr`, com `simulated: true` quando
  veio do SNCR simulado, fora de produção), a identificação do paciente (o CPF
  do cadastro, sempre, e o endereço completo), o endereço e o
  telefone do prescritor, 2 vias e `valid_until`. `antimicrobial`: o número do
  tipo `ret` e o contato do prescritor (identificação do paciente opcional), 2
  vias e `valid_until`. Número no formato do SNCR `AAMM.T-UF.NNNNNNN`. Endereço:
  CEP com 8 dígitos, UF maiúscula, complemento ausente quando não há; telefone
  só com dígitos (DDD + número). A receita em papel não tem JSON canônico, então
  `sncr` nunca aparece nulo aqui; o registro da Notificação de papel
  (`controlled_notification_record`) não é assinado e não tem esquema. O esquema
  não confere a lista da Portaria 344 de cada item, o número de substâncias C1,
  a duração, a validade nem que o CPF é o do cabeçalho: isso é do `api`.
```

(e) Em "## Vetor de canonicalização", troque a primeira frase (`` `examples/canonical/` tem o JCS de `consultation-full.json`, `` … `` `prescription-dosage-form-null.json` e o `SHA256SUMS`. ``) por:

```markdown
`examples/canonical/` tem o JCS de `consultation-full.json`,
`addendum-structured.json`, `prescription-doctor.json`, `sick-note-leave.json`,
`prescription-dosage-form-null.json`, `prescription-special-control.json` e
`prescription-ret.json` e o `SHA256SUMS`.
```

e, no fim do parágrafo, troque `` (`note`, `duration_days`) e `dosage_form: null`. `` por `` (`note`, `duration_days`, `complement`, `patient_identification`), `dosage_form: null` e o número SNCR com ponto e hífen. ``.

Em `contracts/.claude/mod19d/clinical/CHANGELOG.md`, logo depois de `# Changelog — clinical`:

```markdown

## clinical-v1.2.0 — 2026-10-10 — MINOR

Receita de controle especial e de antimicrobiano com número SNCR (ADR 0034,
módulo 19d). Nada do `v1.1.0` muda: toda receita assinada sem `category`
continua válida, com os mesmos bytes.

- `clinical-document-v1.json`, receita: `category` (`special_control`,
  `antimicrobial`; ausente na receita comum), `sncr` (`kind` `rce`/`ret`, `number`,
  `simulated`), `patient_identification` (CPF do cadastro e
  endereço) e `prescriber_contact` (endereço e telefone), todos opcionais,
  com as regras de cada categoria; `$defs` novos `address`, `sncr`,
  `patient_identification`, `prescriber_contact`.
- Regra de vias: antimicrobiano ou controle especial = 2 vias e validade;
  senão 1 via.
- `examples/clinical-document/`: 31 exemplos novos (3 válidos, 28 inválidos);
  82 no total (15 válidos, 67 inválidos).
- Vetor de canonicalização: `prescription-special-control.jcs` e
  `prescription-ret.jcs` em `examples/canonical/`, com as linhas novas no
  `SHA256SUMS`.
- README: acrescentar campo opcional compatível é MINOR; só mudança
  incompatível pede `v2`.
```

- [ ] **Step 8: Rode a prova do vetor e do README**

```bash
C=contracts/.claude/mod19d/clinical/examples
(cd $C/canonical && shasum -a 256 -c SHA256SUMS)
python3 -I -c '
import json, sys
for source, target in zip(sys.argv[1::2], sys.argv[2::2]):
    doc = json.load(open(source, encoding="utf-8"))
    data = json.dumps(doc, sort_keys=True, separators=(",", ":"), ensure_ascii=False, allow_nan=False).encode("utf-8")
    print(target.rsplit("/", 1)[-1], "jcs-igual" if data == open(target, "rb").read() else "JCS DIFERENTE")
' $C/clinical-document/prescription-special-control.json $C/canonical/prescription-special-control.jcs \
  $C/clinical-document/prescription-ret.json $C/canonical/prescription-ret.jcs
grep -o '"sncr":{[^}]*}' $C/canonical/prescription-special-control.jcs
grep -c '"complement"' $C/canonical/prescription-ret.jcs
grep -c '"patient_identification"' $C/canonical/prescription-ret.jcs
grep -n 'clinical-v1.2.0\|0034' contracts/.claude/mod19d/clinical/README.md contracts/.claude/mod19d/clinical/CHANGELOG.md
```

Expected: sete `OK`; `prescription-special-control.jcs jcs-igual` e `prescription-ret.jcs jcs-igual`; `"sncr":{"kind":"rce","number":"2610.1-41.0001234","simulated":true}`; `0` e `0` (na RET o contato do prescritor não tem complemento e a identificação do paciente está ausente: chave ausente, não `null`); e as linhas do README (título, regra de evolução, categoria) e do CHANGELOG com `clinical-v1.2.0`/`0034`.

- [ ] **Step 9: README da raiz**

Em `contracts/.claude/mod19d/README.md`:

(a) Na tabela "Domínios", troque a linha de `clinical/` por:

```markdown
| [`clinical/`](clinical/README.md) | JSON canônico assinado da consulta, do adendo e dos documentos clínicos (ADRs 0032, 0033 e 0034) | `clinical-v1.2.0` | Materializado |
```

(b) No parágrafo `### clinical`, troque `` `clinical-document-v1.json` (ADR 0033, `clinical-v1.1.0`) descrevem o `` por `` `clinical-document-v1.json` (ADRs 0033 e 0034, `clinical-v1.2.0`) descrevem o `` e `` clínico emitido na consulta (atestado, declaração, receita, requisição de `` por `` clínico emitido na consulta (atestado, declaração, receita comum, de controle especial ou de antimicrobiano, requisição de ``; e a última frase (`Documento assinado nunca muda: esquema novo é versão nova (\`v2\`), nunca edição` / `de \`v1\`.`) por:

```markdown
Documento assinado nunca muda: acrescentar campo opcional compatível é MINOR
na mesma versão; só mudança incompatível pede versão nova (`v2`).
```

Confira: `grep -n 'clinical' contracts/.claude/mod19d/README.md` mostra a linha da tabela com `clinical-v1.2.0` e o parágrafo com `0034`.

- [ ] **Step 10: Commit**

```bash
W=contracts/.claude/mod19d
/opt/homebrew/bin/git -C $W add clinical/clinical-document-v1.json clinical/examples/clinical-document \
  clinical/examples/canonical/prescription-special-control.jcs clinical/examples/canonical/prescription-ret.jcs \
  clinical/examples/canonical/SHA256SUMS clinical/README.md clinical/CHANGELOG.md README.md
/opt/homebrew/bin/git -C $W status --short | awk '{print $1}' | sort | uniq -c
/opt/homebrew/bin/git -C $W commit -m "feat: add prescription category and SNCR number to the clinical document schema (clinical-v1.2.0)

Special control (C1/C5) and antimicrobial prescriptions issued digitally
now carry the SNCR number consumed at issue (rce/ret, flagged when it came
from the simulated SNCR), the patient identification (the CPF on file and
the full address) and the prescriber address and
phone (ADR 0034). All new keys are optional: a prescription signed without
category keeps the clinical-v1.1.0 shape and the same bytes. Per-category
rules with if/then; a common prescription still has no category key.
31 more examples and two more JCS vectors.

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

Expected: o `status --short` antes do commit conta `33 A` (31 exemplos + 2 `.jcs`) e `6 M` (esquema, `manifest.json`, `SHA256SUMS`, `clinical/README.md`, `clinical/CHANGELOG.md`, `README.md`).

---

### Task 2: Entrega — verificação final e parada antes do merge

**Files:** nenhum novo. Só leitura e, com autorização, merge/tag/push.

**Interfaces:**
- Consumes: a branch `feat/mod-19d-controlled` com o commit da Task 1.
- Produces: o hash do commit para o plano do `api` do 19d citar; depois de autorizado, a tag `clinical-v1.2.0` em `origin` (contrato §11, passo 1).

- [ ] **Step 1: Verificação final da branch**

```bash
W=contracts/.claude/mod19d
/opt/homebrew/bin/git -C $W log --oneline origin/main..HEAD
/opt/homebrew/bin/git -C $W diff --stat origin/main..HEAD | tail -1
/opt/homebrew/bin/git -C $W diff origin/main..HEAD -- protocols session events types design-tokens \
  clinical/consultation-v1.json clinical/consultation-addendum-v1.json clinical/examples/consultation clinical/examples/consultation-addendum | wc -l
/opt/homebrew/bin/git -C $W diff --numstat origin/main..HEAD -- clinical/examples/clinical-document/manifest.json
.claude/check-schema.sh $W/clinical/clinical-document-v1.json $W/clinical/examples/clinical-document/manifest.json | tail -1
.claude/check-schema.sh $W/clinical/consultation-v1.json $W/clinical/examples/consultation/manifest.json | tail -1
.claude/check-schema.sh $W/clinical/consultation-addendum-v1.json $W/clinical/examples/consultation-addendum/manifest.json | tail -1
(cd $W/clinical/examples/canonical && shasum -a 256 -c SHA256SUMS)
```

Expected: um commit; `39 files changed`; `0` (nada fora do documento clínico, do vetor e dos READMEs/CHANGELOG); o `numstat` do manifesto com `32` linhas acrescentadas e `1` removida (os 31 casos novos, e a linha do último caso antigo, que ganha a vírgula); `82/82 casos`, `31/31 casos`, `11/11 casos`; sete `OK`.

- [ ] **Step 2: Pare e peça autorização**

Reporte ao usuário: o hash do commit, os 82 casos verdes (51 antigos intactos), os dois SHA-256 novos do vetor e as divergências C2–C4 e C6 abaixo (C1, C5 e C7 já decididas). Pergunte, uma de cada vez: (1) merge em `main` local; (2) tag anotada `clinical-v1.2.0`; (3) push da `main` e da tag. Antes do push, confira `origin/main..main` (outra sessão pode ter commitado em `contracts`; publique só o desta sessão).

- [ ] **Step 3: Com autorização — merge, tag e push**

```bash
/opt/homebrew/bin/git -C contracts fetch origin
/opt/homebrew/bin/git -C contracts log --oneline main..origin/main
/opt/homebrew/bin/git -C contracts checkout main
/opt/homebrew/bin/git -C contracts merge --ff-only feat/mod-19d-controlled
/opt/homebrew/bin/git -C contracts tag -a clinical-v1.2.0 -m "clinical-v1.2.0 — MINOR"
/opt/homebrew/bin/git -C contracts log --oneline origin/main..main
/opt/homebrew/bin/git -C contracts push origin main
/opt/homebrew/bin/git -C contracts push origin clinical-v1.2.0
/opt/homebrew/bin/git -C contracts worktree remove .claude/mod19d
/opt/homebrew/bin/git -C contracts branch -d feat/mod-19d-controlled
```

Expected: `main..origin/main` vazio (senão, pare: rebase da branch sobre a nova `origin/main` e refaça o Step 1); merge fast-forward; `origin/main..main` mostra só o commit da Task 1; push da `main` e da tag aceitos (o aviso de "PR obrigatório" passa por bypass, como nos módulos anteriores). Avise a sessão do `api` do 19d que a tag existe: o `config/clinical/clinical-document-v1.json` do `api` é cópia deste arquivo nesta tag, e o spec do gerador canônico confere os dois vetores novos.

---

## Self-review (feito ao escrever o plano)

1. **Cobertura do contrato §9 e §2:** `category`, `sncr`, `patient_identification` e `prescriber_contact` na receita — Task 1, Step 3; exemplos válidos e inválidos — Step 1; vetor JCS com uma RCE e uma RET — Steps 6 e 8; README e CHANGELOG — Steps 7 e 9; tag `clinical-v1.2.0` com push só com autorização — Task 2; `controlled_notification_record` fora do canônico — Global Constraints e README (d).
2. **Placeholders:** nenhum. O bloco do esquema, os `$defs`, o patch, o gerador, as contagens (`52/82` no vermelho, `82/82` no verde) e os SHA-256 foram executados ao escrever o plano, sobre uma cópia do `clinical/` de `ea08b05`.
3. **Consistência:** os nomes dos exemplos no gerador, no manifesto, no Review Focus e nos Steps 2, 4 e 8 conferem; contagens do commit (34 novos + 6 modificados = 40) e do `diff --stat` batem; `$defs` comuns e os do 19c conferidos iguais por comando (Step 5).
4. **Review Focus:** as cinco linhas têm exemplo no manifesto ou prova do vetor na Task 1.

## Divergências propostas ao contrato

- **C1 — MINOR num esquema "fechado" cuja regra dizia `v2`.** O contrato (§9) pede `clinical-document-v1` estendido como MINOR; o README publicado do domínio dizia "campo novo = esquema novo (`v2`)". Proposta (este plano): campos novos só **opcionais**, cuja ausência mantém o significado da `v1.1.0`; a regra do README passa a dizer isso (Task 1, Step 7c, e README da raiz). **Decidido pelo usuário (2026-10-10):** MINOR na `v1`, tag `clinical-v1.2.0`; acrescentar campo opcional compatível é MINOR, só mudança incompatível pede `v2`.
- **C2 — Ausente × `null` no canônico.** O contrato (§2) dá `sncr`, `patient_identification` e `prescriber_contact` como `…|null` na resposta HTTP. No JSON canônico vale a convenção do 19c (§12 do contrato do 19c): campo que não se aplica é **chave ausente**. Como só a receita digital tem canônico, `sncr` nunca é `null` aqui; `patient_identification` é ausente na RET quando o prescritor não a preencheu; `complement` é ausente quando não há. O gerador do `api` precisa seguir isso para bater com o vetor.
- **C3 — Receita comum sem `category` no canônico.** O contrato diz que `category` sempre existe no `content` (HTTP) e o §1 lista `common`. No canônico, a receita comum **não** tem a chave (antes ou depois do 19d), e o esquema recusa `"common"`: uma só representação, e os vetores da `v1.1.0` continuam byte a byte. Igual à D7 do plano do `api` do 19d (o construtor só põe `category` quando ≠ `common`).
- **C4 — Formas fixadas pelo esquema que o contrato não fixa:** `address.zip` com 8 dígitos sem hífen, `uf` maiúscula, `number` como texto (aceita "s/n"), `complement` opcional e não vazio, limites de tamanho do plano do `api` (logradouro 120, número 10, complemento 60, bairro e cidade 80); `phone` só com dígitos, 10 ou 11; `sncr.number` no formato do manual que o plano do `api` fixou (`^[0-9]{4}\.[0-9]-[0-9]{2}\.[0-9]{7}$`, `AAMM.T-UF.NNNNNNN`; se a homologação mostrar outro formato no bloco de RCE/RET, o padrão é afrouxado num PATCH antes do go-live). O `api` (normalização na entrada) e o dashboard (máscara só na tela) devem gravar exatamente isso.
- **C5 — Sem "não possui CPF".** **Decidido pelo usuário (2026-10-10):** o `patient_identification` não tem `no_cpf` nem `passport`; a receita de controle especial usa sempre o CPF do cadastro do paciente (que existe por regra do 19a). O CPF repete o do cabeçalho; o esquema não confere a igualdade (é do `api`).
- **C6 — Vetor de canonicalização.** O contrato pede "uma RCE e uma RET": `prescription-special-control.jcs` (com `simulated: true`, acento, emoji, complemento presente) e `prescription-ret.jcs` (sem complemento e sem identificação do paciente: chaves ausentes), acrescentados ao `SHA256SUMS` existente, sem pasta nova.

- **C7 — Nome do vetor da RET.** O plano do `api` do 19d (Task 16 e D7) espera `prescription-antimicrobial.jcs`, gerado de `prescription-antimicrobial.json`. Esse exemplo já existe desde a `clinical-v1.1.0` (receita de antimicrobiano **sem** `category`, da era do papel) e não pode mudar. **Decidido (coordenador):** o vetor é `prescription-ret.json` / `prescription-ret.jcs`; o spec do `api` troca o nome na lista dos vetores. O número dos exemplos segue o formato do plano do `api` (`2610.1-41.0001234`, `2610.2-41.0000501`).
