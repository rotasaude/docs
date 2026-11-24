# Módulo 26b — `clinical-v1.3.0`: laudo de regulação assinável (repo contracts) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Ordem de entrega (contrato §10, Q4):**
> 1. **contracts `clinical-v1.3.0` sem tag (Tasks 0–3)**;
> 2. SHA-256 ao api (Task 4);
> 3. api (Task 15 do plano do api: copia o esquema, os exemplos e o vetor, presos pelo SHA-256);
> 4. **contracts: SHA-256 conferidos, tag e push (Task 5, com autorização, depois da suíte completa do api)**;
> 5. dashboard e maintenance.
>
> Nenhuma task daqui depende do api do 26b rodando. A verificação usa o `json_schemer` do container `api` do compose (o `$R/.claude/check-schema.sh` dos módulos 19c/19d).

**Goal:** Publicar o MINOR `clinical-v1.3.0` com o esquema novo `clinical/regulation-laudo-v1.json` (`rotasaude.regulation_laudo.v1`): o JSON canônico do laudo de solicitação de APAC, OCI e AIH que o profissional assina (CAdES do JCS; o PAdES vai sobre o PDF do api).

**Architecture:** A mesma mecânica do `clinical-v1.1.0` (que acrescentou `clinical-document-v1.json` como MINOR): esquema fechado em todo nível, `$defs` comuns copiados sem mudança de `clinical-document-v1.json`, exemplos com manifesto (`valid` ou `<data_pointer> <type>` do `json_schemer`), vetor RFC 8785 em `examples/canonical/` com o `SHA256SUMS`. Nada dos esquemas e exemplos já publicados muda (documento assinado nunca muda). O esquema confere **forma**: domínios, faixas, ausente × nulo, um principal só, campos da AIH e o mínimo de secundários da OCI. As **relações** ficam com o api: compatibilidade SIGTAP entre principal e secundário, alvo regulável por laudo, o secundário de consulta da OCI, o CID por CBO e o domínio da clínica do SIH.

**Tech Stack:** JSON Schema 2020-12; `json_schemer` 2.5 (container `api`); `python3` do host (JCS com chaves ASCII e números inteiros); `shasum`; git em `/opt/homebrew/bin/git`.

**Spec:** `docs/superpowers/specs/2026-11-25-module-26b-apac-oci-aih-design.md` (decisões 13 e 21). Contrato (fonte única): `docs/superpowers/plans/2026-11-25-module-26b-apac-contracts.md`, §3, §10 e Q4. Plano irmão: `docs/superpowers/plans/2026-11-25-module-26b-apac-api.md` (Valor 3 e Tasks 5 e 15). Molde: `docs/superpowers/plans/2026-11-23-module-35b-zoonoses-contracts-repo.md` (ordem e entrega) e `docs/superpowers/plans/2026-10-09-module-19c-documents-contracts-repo.md` (domínio `clinical` e vetor). **Divergências:** seção final (D1–D3), **resolvidas pela Q12 do §12 do contrato (docs `60908e1`)**, como propostas: cabeçalho comum + `request` + `laudo`, sem o número da autorização; paciente sem CPF → laudo impresso para assinatura à mão; os limites do Global Constraints.

## Global Constraints

- **Repo e base:** `contracts` na raiz do monorepo (`$R/contracts`, remote `git@github.com:rotasaude/contracts.git`), `origin/main` em `a199ff0` ou mais novo; tags `clinical-v1.0.0`, `-v1.1.0` e `-v1.2.0` (`3adc39a`) existem, e nenhuma `clinical-v1.3.0`.
- **Tag:** `clinical-v1.3.0`, anotada, mensagem `clinical-v1.3.0 — MINOR`. Sem entrada no CHANGELOG, não há tag. A tag só sai com autorização do usuário e **depois** da suíte completa verde do api.
- **Nada publicado muda:** `consultation-v1.json`, `consultation-addendum-v1.json`, `clinical-document-v1.json`, os 127 exemplos (31 + 11 + 85) e os 7 vetores do `SHA256SUMS` ficam byte a byte iguais (o api prende os hashes).
- **Convenções do domínio (README):** `$id` = `https://rota-saude/schemas/clinical/regulation-laudo-v1.json`, `title` = `rotasaude.regulation_laudo.v1`, `schema` `const`; `additionalProperties: false` em todo nível; instantes UTC com segundos e `Z`; datas `AAAA-MM-DD`; CPF, CNES, IBGE e CBO só dígitos; texto vazio é `null`, nunca `""`; campo que não se aplica ao caso é chave **ausente**, texto que sempre se aplica está sempre presente e é `null` quando vazio.
- **`$defs` comuns** (`uuid`, `utc_datetime`, `date`, `cpf`, `city`, `unit`, `professional`, `patient`) **idênticos** aos de `clinical-document-v1.json` (Q12).
- **Valores do contrato e do plano do api, copiados sem mudança:** `authorization_kind` `apac` `oci` `aih` (o `bpa` não tem laudo); `role` `principal` `secondary`; `character` `elective` `urgent`; `clinical_summary` 20–4000; `findings` até 4000; `quantity` 1–999; 1 a 30 procedimentos; OCI com o principal `09…` e ao menos 2 secundários; SIGTAP de 10 dígitos com a competência `AAAAMM` e o rótulo, como os exames de `consultation-v1.json`; CID-10 `^[A-Z][0-9]{2}[0-9X]?$` (sem ponto, como em `performed_procedures`); clínica do SIH `^[0-9]{2}$` (Q12).
- git `/opt/homebrew/bin/git`; nunca `git add -A`. Commits em inglês, Conventional Commits com o tipo por extenso: `git commit -m "<tipo>: <descrição>" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`. Merge, tag e push só com autorização.

## Ambiente de execução

- `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `G=/opt/homebrew/bin/git`, `C=$R/.worktrees/contracts-mod26b`. Os caminhos das tasks são relativos a `$C`.
- Verificação (de `$R`, com o compose de pé): `$R/.claude/check-schema.sh <esquema> <manifesto>` (imprime `ok`/`FAIL` por caso e `N/M casos`); vetor: `(cd $C/clinical/examples/canonical && shasum -a 256 -c SHA256SUMS)`.

## Review Focus

1. **O mesmo laudo com uma chave opcional ausente × nula dá dois hashes:** `cid_secondary` e `findings` sem valor têm de estar presentes e `null`; `character` e `clinic` fora da AIH têm de estar ausentes. Exemplos `invalid-laudo-without-cid-secondary-key.json` e `invalid-apac-with-clinic.json` (Task 1).
2. **Dois principais, ou nenhum:** o pedido com dois `principal` assinado mudaria o que a autorização numera. Exemplos `invalid-two-principals.json` e `invalid-no-principal.json` (Task 1).
3. **AIH sem caráter ou sem clínica** passa a vida inteira sem esses campos no que foi assinado. Exemplo `invalid-aih-without-character.json` (Task 1).
4. **Texto com acento, cedilha, emoji e quebra de linha** na justificativa: o vetor tem de reproduzir os bytes em UTF-8 cru. Vetor `laudo-oci.jcs` (Task 2).
5. **Deriva entre a cópia do api e a tag:** os SHA-256 entregues na Task 4 são os que o api prende; mudança depois disso volta às Tasks 1–3 e gera SHA-256 novos (Task 5, Step 1).

## Mapa de arquivos

| Arquivo | O que muda | Task |
|---|---|---|
| `clinical/regulation-laudo-v1.json` | esquema novo | 1 |
| `clinical/examples/regulation-laudo/*.json` + `manifest.json` | 20 exemplos (4 válidos, 16 inválidos) | 1 |
| `clinical/examples/canonical/{laudo-oci,laudo-aih}.jcs`, `SHA256SUMS` | vetor (duas linhas novas no fim) | 2 |
| `clinical/README.md`, `clinical/CHANGELOG.md` | tabela, regras do laudo, vetor; entrada `clinical-v1.3.0` | 3 |

---

### Task 0: Base e worktree

**Files:** nenhum.

**Interfaces:**
- Produces: `$C` em `feat/mod-26b-apac` e a base anotada.

- [ ] **Step 1: Base e worktree**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git; C=$R/.worktrees/contracts-mod26b
$G -C $R/contracts fetch -q origin
$G -C $R/contracts log --oneline -1 origin/main
$G -C $R/contracts tag -l 'clinical-*'
$G -C $R/contracts worktree add $C -b feat/mod-26b-apac origin/main
cd $R && for p in consultation:consultation-v1 consultation-addendum:consultation-addendum-v1 clinical-document:clinical-document-v1; do $R/.claude/check-schema.sh $C/clinical/${p#*:}.json $C/clinical/examples/${p%%:*}/manifest.json | tail -1; done
(cd $C/clinical/examples/canonical && shasum -a 256 -c SHA256SUMS)
$G -C $R/apps/api fetch -q origin
for f in consultation-v1 consultation-addendum-v1 clinical-document-v1; do $G -C $R/apps/api show origin/main:config/clinical/$f.json | cmp -s - $C/clinical/$f.json && echo "igual $f" || echo "DIFERENTE $f"; done
```

Expected: `a199ff0` ou mais novo; tags `clinical-v1.0.0` a `-v1.2.0`, sem `-v1.3.0`; `31/31 casos`, `11/11 casos`, `85/85 casos`; os sete `.jcs: OK`; os três `igual`. Diferente: **pare e reporte** (outro módulo mexeu no `clinical/`).

- [ ] **Step 2: Decisões.** Confira que o §12 do contrato tem a Q12 (D1–D3) e que nenhuma Q posterior mudou o que este plano aplica; se mudou, ajuste a task indicada antes de começar e reporte.

**Relato:** "contracts-mod26b Task 0: worktree em feat/mod-26b-apac sobre <sha>; 31/31, 11/11, 85/85; vetor OK; cópias do api iguais; D1–D3 conferidas na Q12."

---

### Task 1: `regulation-laudo-v1.json` e exemplos

**Files:**
- Create: `clinical/regulation-laudo-v1.json`, `clinical/examples/regulation-laudo/manifest.json` e os 20 exemplos abaixo.

**Interfaces:**
- Produces (forma do canônico, Q12):
  - raiz `required: ["schema", "city", "unit", "professional", "patient", "request", "laudo"]`; `schema` `const` `rotasaude.regulation_laudo.v1`; `city`, `unit` (a unidade solicitante), `professional` (o solicitante, que assina), `patient` pelos `$defs` comuns;
  - `request` `{ id (uuid do pedido), authorization_kind (enum apac|oci|aih), requested_at (utc_datetime: a promoção), procedures }`, todos obrigatórios; `procedures` 1–30 itens `{ sigtap_code ^[0-9]{10}$, competence ^[0-9]{4}(0[1-9]|1[0-2])$, label (string ≥ 1), role (enum), quantity (inteiro 1–999) }`, todos obrigatórios; exatamente um `principal` (`contains` com `minContains: 1`, `maxContains: 1`);
  - `laudo` `{ clinical_summary (string 20–4000), cid_principal ($defs/cid10), cid_secondary ($defs/cid10 | null), findings (string 1–4000 | null), character (enum), clinic (^[0-9]{2}$) }`; obrigatórios `clinical_summary`, `cid_principal`, `cid_secondary`, `findings`;
  - `allOf` na raiz: `authorization_kind` `aih` → `laudo` exige `character` e `clinic`; senão → `laudo` sem as duas (`not` `anyOf` `required`); `oci` → o principal casa `^09` e há ≥ 2 `secondary` (`contains` com `minContains: 2`); `apac`/`aih` → o principal casa `^0[234]`;
  - `$defs` novos: `cid10`, `procedure`; os comuns copiados de `clinical-document-v1.json`;
  - `description` do esquema: o que é, ADR do 26b (número na execução), "o número da autorização não entra: sai depois, na fila", e as relações que ficam com o api.

- [ ] **Step 1: Escreva os exemplos e o manifesto.** Válidos (`valid`): `laudo-oci.json` (OCI `09…` com a consulta `0301010072` e outro secundário; justificativa com acento, cedilha, emoji e `\n`); `laudo-aih.json` (`0407030026`, `character` `elective`, `clinic` `01`, `cid_secondary` preenchido); `laudo-apac.json` (`0305010107`, quantidade 12); `laudo-apac-minimal.json` (`cid_secondary` e `findings` `null`, nascimento `null`). Inválidos (cada um muda **uma** coisa de um válido; esperado no ponteiro indicado, com o tipo que o `json_schemer` 2.5 devolver no Step 3):
  - `invalid-schema-name.json` → `/schema const`;
  - `invalid-extra-root-key.json` (chave `authorization` na raiz) → `/authorization schema`;
  - `invalid-kind-bpa.json` → `/request/authorization_kind enum`;
  - `invalid-requested-at-offset.json` (`-03:00`) → `/request/requested_at pattern`;
  - `invalid-summary-short.json` (19 caracteres) → `/laudo/clinical_summary minLength`;
  - `invalid-cid-dotted.json` (`I10.0`) → `/laudo/cid_principal pattern`;
  - `invalid-laudo-without-cid-secondary-key.json` → `/laudo required`;
  - `invalid-findings-empty.json` (`""`) → `/laudo/findings minLength`;
  - `invalid-aih-without-character.json` → `/laudo required`;
  - `invalid-apac-with-clinic.json` → `/laudo not`;
  - `invalid-clinic-letters.json` (`"AB"`) → `/laudo/clinic pattern`;
  - `invalid-two-principals.json` e `invalid-no-principal.json` → `/request/procedures <tipo do contains>`;
  - `invalid-oci-one-secondary.json` e `invalid-oci-principal-not-09.json` → `/request/procedures <tipo do contains>`;
  - `invalid-procedure-quantity-zero.json` → `/request/procedures/0/quantity minimum`.
- [ ] **Step 2: Rode antes do esquema** `cd $R && $R/.claude/check-schema.sh $C/clinical/regulation-laudo-v1.json $C/clinical/examples/regulation-laudo/manifest.json` → falha (arquivo ausente).
- [ ] **Step 3: Escreva o esquema.** Rode o mesmo comando. Onde o tipo de erro do `contains` vier diferente do anotado, grave no manifesto o tipo devolvido, **desde que o ponteiro seja o indicado**; ponteiro diferente = esquema errado.
- [ ] **Step 4: Confira** → `20/20 casos`; `$defs` comuns iguais:

```bash
python3 -I - "$C/clinical" <<'PY'
import json, sys
d = sys.argv[1]
a = json.load(open(f"{d}/clinical-document-v1.json"))["$defs"]; b = json.load(open(f"{d}/regulation-laudo-v1.json"))["$defs"]
common = ["uuid", "utc_datetime", "date", "cpf", "city", "unit", "professional", "patient"]
print("defs-comuns-iguais" if all(a[k] == b[k] for k in common) else [k for k in common if a[k] != b[k]])
PY
$G -C $C status --short -- clinical/consultation-v1.json clinical/consultation-addendum-v1.json clinical/clinical-document-v1.json clinical/examples/consultation clinical/examples/consultation-addendum clinical/examples/clinical-document
```

Expected: `defs-comuns-iguais` e o `status` vazio.
- [ ] **Step 5: Commit**

```bash
cd $C && $G add clinical/regulation-laudo-v1.json clinical/examples/regulation-laudo
$G commit -m "feat: add the regulation laudo canonical schema (clinical-v1.3.0)" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato:** "contracts-mod26b Task 1: regulation-laudo-v1.json (APAC, OCI, AIH; um principal; AIH com caráter e clínica; OCI 09 com 2 secundários); 20/20; $defs comuns iguais."

---

### Task 2: Vetor de canonicalização

**Files:**
- Create: `clinical/examples/canonical/laudo-oci.jcs`, `clinical/examples/canonical/laudo-aih.jcs`
- Modify: `clinical/examples/canonical/SHA256SUMS` (duas linhas no fim)

- [ ] **Step 1: Escreva o vetor.** O JCS destes exemplos é o `json.dumps` com chaves ordenadas, sem espaços e sem escapar não-ASCII (chaves ASCII e números inteiros, como no 19c):

```bash
(cd $C/clinical/examples && python3 -I - <<'PY'
import json
for source, target in (("regulation-laudo/laudo-oci.json", "canonical/laudo-oci.jcs"),
                       ("regulation-laudo/laudo-aih.json", "canonical/laudo-aih.jcs")):
    doc = json.load(open(source, encoding="utf-8"))
    data = json.dumps(doc, sort_keys=True, separators=(",", ":"), ensure_ascii=False, allow_nan=False).encode("utf-8")
    open(target, "wb").write(data)
    print(target, len(data))
PY
)
(cd $C/clinical/examples/canonical && shasum -a 256 laudo-oci.jcs laudo-aih.jcs >> SHA256SUMS && shasum -a 256 -c SHA256SUMS)
```

Expected: os nove `OK`; as sete primeiras linhas do `SHA256SUMS` iguais às da base (`$G -C $C diff origin/main -- clinical/examples/canonical/SHA256SUMS` só com as duas linhas acrescentadas).
- [ ] **Step 2: Commit** `cd $C && $G add clinical/examples/canonical && $G commit -m "test: add the regulation laudo canonicalization vectors" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato:** "contracts-mod26b Task 2: vetor laudo-oci.jcs e laudo-aih.jcs; SHA256SUMS com 9 linhas, as 7 antigas intactas."

---

### Task 3: README e CHANGELOG

**Files:** Modify `clinical/README.md`, `clinical/CHANGELOG.md`.

- [ ] **Step 1: README.** Título com o ADR do 26b; frase de abertura com "o laudo de solicitação de APAC, OCI ou AIH"; linha nova na tabela (`regulation-laudo-v1.json` · `rotasaude.regulation_laudo.v1` · laudo de regulação assinado pelo solicitante na promoção do pedido); seção "Laudo de regulação" com: um principal; secundários com a competência e o rótulo da SIGTAP no ato; AIH com `character` e `clinic`, ausentes nos outros tipos; `cid_secondary` e `findings` sempre presentes e `null` quando vazios; o número da autorização não entra (sai depois); o que o esquema não confere (compatibilidade SIGTAP, alvo por laudo, o secundário de consulta da OCI, CID por CBO, domínio da clínica do SIH, que o `cid_principal` é o CID do pedido); o comando de verificação com o manifesto novo; os dois vetores na lista do vetor.
- [ ] **Step 2: CHANGELOG.** Entrada no topo `## clinical-v1.3.0 — (não publicada até a tag) — MINOR`: esquema novo (nada do `v1.2.0` muda); forma do laudo e regras de presença; `$defs` comuns idênticos; `examples/regulation-laudo/`: 20 exemplos (4 válidos, 16 inválidos); vetor `laudo-oci.jcs` e `laudo-aih.jcs`.
- [ ] **Step 3: Confira** os quatro manifestos (`31/31`, `11/11`, `85/85`, `20/20`) e o vetor (nove `OK`).
- [ ] **Step 4: Commit** `cd $C && $G add clinical/README.md clinical/CHANGELOG.md && $G commit -m "docs: document the regulation laudo schema and clinical-v1.3.0" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`.

**Relato:** "contracts-mod26b Task 3: README (regras do laudo) e CHANGELOG clinical-v1.3.0 sem data; 4 manifestos verdes."

---

### Task 4: SHA-256 ao api, e parada

**Files:** nenhum.

- [ ] **Step 1: Verificação e SHA-256**

```bash
cd $R && $R/.claude/check-schema.sh $C/clinical/regulation-laudo-v1.json $C/clinical/examples/regulation-laudo/manifest.json | tail -1
$G -C $C log --oneline origin/main..HEAD
$G -C $C diff --stat origin/main..HEAD -- field-sync protocols session exams lab billing events types design-tokens map-style | tail -1
(cd $C/clinical && shasum -a 256 regulation-laudo-v1.json examples/regulation-laudo/manifest.json examples/canonical/laudo-oci.jcs examples/canonical/laudo-aih.jcs)
```

Expected: `20/20 casos`; três commits; nada fora de `clinical/`; quatro SHA-256.
- [ ] **Step 2: Pare.** Sem merge nem push. Mande ao coordenador o `HEAD`, o caminho `$C/clinical/` e os quatro SHA-256, com o que o api muda junto (Task 15 do plano do api): `config/clinical/regulation-laudo-v1.json`; `spec/fixtures/clinical/regulation-laudo/` (20 exemplos + manifesto); `spec/fixtures/clinical/canonical/{laudo-oci,laudo-aih}.jcs` e as duas linhas do `SHA256SUMS`; `Signatures::Canonical::LAUDO_SCHEMA` em `SCHEMA_FILES`; o `canonical_vector_spec.rb` com os dois vetores e o manifesto novo; e a Q12 (paciente sem CPF não abre assinatura: o laudo sai impresso). Mudança pedida pelo api volta às Tasks 1–3 e gera SHA-256 novos.

**Relato:** "contracts-mod26b Task 4: branch em <sha>, 20/20; SHA-256: <lista>; aguardando o api."

---

### Task 5: Entrega — SHA-256 conferidos, merge, tag e push (com autorização, depois do api)

**Files:** `clinical/CHANGELOG.md` (data). Merge, tag e push só com autorização.

- [ ] **Step 1: Verificação final.** O bloco da Task 4, Step 1, mais os quatro manifestos e o vetor. Expected: os quatro SHA-256 **iguais** aos que o spec do api prende, confirmados pelo coordenador depois da suíte completa verde do api. Diferente: pare.
- [ ] **Step 2: Data no CHANGELOG** (`(não publicada até a tag)` → `<AAAA-MM-DD>`) e `cd $C && $G add clinical/CHANGELOG.md && $G commit -m "docs: date the clinical-v1.3.0 changelog entry" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`. O CHANGELOG não está entre os arquivos presos.
- [ ] **Step 3: Pare e peça autorização**, uma etapa por vez: merge em `main`; tag `clinical-v1.3.0`; push da `main` e da tag. Reporte o hash, `20/20`, os SHA-256 e a Q12 aplicada (D1–D3).
- [ ] **Step 4: Com autorização**

```bash
$G -C $R/contracts fetch origin && $G -C $R/contracts log --oneline main..origin/main
$G -C $R/contracts checkout main && $G -C $R/contracts merge --ff-only feat/mod-26b-apac
$G -C $R/contracts tag -a clinical-v1.3.0 -m "clinical-v1.3.0 — MINOR"
$G -C $R/contracts log --oneline origin/main..main
$G -C $R/contracts push origin main && $G -C $R/contracts push origin clinical-v1.3.0
$G -C $R/contracts worktree remove $C && $G -C $R/contracts branch -d feat/mod-26b-apac
```

Expected: `main..origin/main` vazio (senão rebase e refaça o Step 1); fast-forward; `origin/main..main` só com os commits deste plano; push aceito (bypass do aviso de PR, como nos módulos anteriores). Avise o api para trocar o comentário da cópia de "branch <sha>" para "tag clinical-v1.3.0".

**Relato:** "contracts-mod26b Task 5: clinical-v1.3.0 publicada (main <sha>, tag pushada)."

---

## Self-review

1. **Cobertura:** §10 e Q4 — esquema novo no `clinical` (1), MINOR com tag depois do api (5); §3 — tipos, laudo (`clinical_summary` 20–4000, CID principal e secundário, achados, caráter e clínica da AIH) e procedimentos com papel e quantidade (1); OCI com principal 09 e 2 secundários (1); assinatura em JCS (2).
2. **Nomes:** `rotasaude.regulation_laudo.v1`, `clinical/regulation-laudo-v1.json`, `examples/regulation-laudo/`, `laudo-oci.jcs`, `laudo-aih.jcs` — os mesmos da Task 15 do plano do api.
3. **Proporção:** quatro arquivos de esquema/vetor e dois de texto; nenhuma ferramenta nova.

## Divergências propostas ao contrato

> D1–D3 **resolvidas pela Q12 do §12 do contrato (docs `60908e1`)**, como propostas.

D1. **[Resolvida pela Q12.]** **A forma do canônico não está no contrato.** O §10 nomeia o esquema e o domínio, e o §3 lista os campos do laudo e dos procedimentos, mas não diz o que mais entra no que se assina. **Proposta:** o cabeçalho comum do domínio (cidade, unidade solicitante, solicitante, paciente), `request` (`id`, `authorization_kind`, `requested_at`, procedimentos com competência e rótulo SIGTAP do ato, como os exames de `consultation-v1.json`) e `laudo`; **sem** o número da autorização (sai depois, na fila), sem a `justification` do 26a e sem a versão do laudo (Task 1).

D2. **[Resolvida pela Q12.] Vale a proposta: o laudo sai impresso, para assinatura à mão.** **Paciente sem CPF.** O `$defs/patient` comum exige CPF (`clinical-document-v1.json`, `$defs.patient.required`), e o pedido de OCI identifica o paciente por CPF ou CNS (pesquisa, linha 32); o BPA-I do api já aceita paciente só com CNS (`app/services/billing/critic.rb:87-89`). **Proposta:** manter o `patient` comum idêntico (uma representação no domínio); o laudo de paciente sem CPF não abre assinatura e sai impresso para assinar à mão, como o laudo sem certificado (Task 15 do plano do api). A alternativa é um `patient` próprio com CPF ou CNS, que quebraria o "`$defs` idênticos" do README.

D3. **[Resolvida pela Q12.]** **Limites e domínios que o contrato não fixa.** O §3 não dá o tamanho de `findings`, a faixa de `quantity`, o número máximo de procedimentos nem o domínio de `clinic` ("código da clínica do SIH"). **Proposta:** `findings` até 4000 (o mesmo da justificativa), `quantity` 1–999 (o `PerformedProcedure::QUANTITY` do api), até 30 procedimentos, e `clinic` só na forma (`^[0-9]{2}$`) no esquema, com o domínio conferido pelo api em `config/regulation/sih_clinics.yml` (Task 0 do api).
