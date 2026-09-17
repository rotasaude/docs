# ADR pointer reconciliation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Reescrever as 225 referências a ADR do `apps/api` para a numeração do corpus v2 (0001..0015), decidindo caso a caso, com o arquivo v1 restaurado no repo `docs` e a decisão registrada numa tabela auditável.

**Architecture:** Três camadas. (1) Restaurar `_v1/` e `_refundacao/` no repo `rotasaude/docs` para que a adjudicação seja reproduzível. (2) Gerar um inventário mecânico, adjudicar cada ocorrência contra o texto v1 e o v2, e commitar a decisão em `docs/adr/RECONCILIACAO-PONTEIROS.md`. (3) Aplicar a reescrita por camada do `apps/api`, com uma spec de guarda que falha enquanto sobrar ref fora da faixa.

**Tech Stack:** Ruby (script de inventário + RSpec), Markdown, git/gh. Specs rodam no container `api-dev`.

**Spec:** `docs/superpowers/specs/2026-09-11-adr-pointer-reconciliation-design.md`

## Global Constraints

- **Commits em inglês**, em todas as apps.
- **`apps/api` está na branch `fix/migrations-owner-ddl-as-admin`.** Commite nessa branch. Use `git -C apps/api ...`.
- **Nunca edite `rota-saude/docs/` para o entregável.** Essa pasta **não é um clone** e já está desatualizada em relação a `rotasaude/docs` (a cópia local ainda tem a frase antiga do DE-PARA, com um caminho absoluto circular). Todo trabalho de documentação vai num clone fresco — a Task 1 cria em `/tmp/rota-docs`.
- **As specs do api rodam DENTRO do container:** `docker exec api-dev bundle exec rspec <path>`. Suba com `./start.sh` se `docker ps` não listar `api-dev`.
- **Faixa válida do corpus v2: 0001..0015.** Qualquer outro número é v1 por definição.
- **Estilo:** mantenha a forma hifenizada `ADR-00NN` que o código já usa (diff mínimo). Se um comentário citar dois ADRs v1 que colapsam no mesmo v2 (`ADR-0016 e ADR-0017` → ambos 0009), **deduplique**: escreva `ADR-0009` uma vez só.
- **Encoding:** os arquivos são UTF-8 com comentários acentuados. Todo `File.readlines` precisa de `encoding: "UTF-8"` explícito — sem isso o script morre com `invalid byte sequence in US-ASCII`. (Verificado na prática.)
- **Deferido (NÃO faça):** trocar número por slug; editar o texto dos ADRs v2; tocar em `rota-saude/scripts/*.rb` (fora de repo — é o item 5); reconciliar a cópia local `rota-saude/docs/` (item 5); criar CI (item 3).

## Procedimento de adjudicação

Isto é o método que as Tasks 4–11 aplicam. Leia antes de começar a Task 4.

Para **cada ocorrência** do inventário:

1. **Leia o contexto real** — o comentário inteiro e as linhas de código que ele anota. Um ponteiro não se decide pelo número; se decide pelo que o código faz.
2. **Se o número for ≥ 0016:** é v1 sem ambiguidade. Aplique o mapa abaixo. Se cair num split, escolha pelo que o código faz.
3. **Se o número for ≤ 0015:** é ambíguo. Abra os dois candidatos e compare:
   - `/tmp/rota-docs/_v1/adr/00NN.md` — o que esse número queria dizer no v1;
   - `/tmp/rota-docs/docs/adr/00NN.md` — o que quer dizer no v2.
   Aquele cuja decisão descreve o que o código faz é o correto.
   - Se o v1 casa → é ref v1, mapeie para o v2 correspondente.
   - Se o v2 casa → já era v2, **não mexa**, e registre na tabela como `já v2`.
   - Se nenhum casa, ou os dois casam: registre `DÚVIDA` na tabela, **deixe o código como está**, e leve para revisão humana. Não chute.

**Mapa v1 → v2:**

| v1 | v2 | v1 | v2 | v1 | v2 |
|---|---|---|---|---|---|
| 0001 | 0001 | 0010 | 0007 | 0019 | 0003 |
| 0002 | 0001 | 0011 | **SPLIT** | 0020 | **SPLIT** |
| 0003 | 0004 | 0012 | 0008 | 0021 | 0007 |
| 0004 | 0004 | 0013 | 0009 | 0022 | 0011 |
| 0005 | 0005 | 0014 | 0005 | 0023 | 0012 |
| 0006 | 0004 | 0015 | 0009 | 0024 | 0013 |
| 0007 | 0010 | 0016 | 0009 | 0025 | 0002 |
| 0008 | 0006 | 0017 | 0009 | 0026 | 0015 |
| 0009 | 0014 | 0018 | 0002 | | |

**Splits:**

- **v1 0011** (payload bruto: criptografia e retenção) → **0013** se o trecho fala de chave, criptografia ou custódia de secret; **0014** se fala de retenção ou purga.
- **v1 0020** (tenant em eventos e jobs) → **0003** se é o mecanismo de tenant (`with_tenant`, `SET LOCAL`, RLS); **0004** se é o publish carimbado com tenant; **0005** se é a idempotência do consumidor.

**Registre cada ocorrência** como uma linha em `docs/adr/RECONCILIACAO-PONTEIROS.md`:

```
| apps/api/app/models/triage.rb:2 | ADR-0006 | ADR-0004 | v1 0006 = camada de commands; o comentário diz "Manipulado APENAS via commands" |
```

---

### Task 1: Restaurar o arquivo v1 e o `_refundacao` no repo `docs`

**Files:**
- Clone: `/tmp/rota-docs` (de `rotasaude/docs`)
- Create: `/tmp/rota-docs/_v1/README.md`, `/tmp/rota-docs/_v1/adr/*` (25 ADRs + README + nota)
- Create: `/tmp/rota-docs/_refundacao/*` (9 documentos)
- Modify: `/tmp/rota-docs/docs/README.md:23`, `:41`; `/tmp/rota-docs/docs/adr/README.md:93`; `/tmp/rota-docs/docs/adr/DE-PARA.md:5`

**Interfaces:**
- Produces: `/tmp/rota-docs/_v1/adr/00NN.md` para NN em 01..25 — o texto v1 que as Tasks 4–11 consultam para adjudicar.
- Produces: `/tmp/rota-docs/docs/adr/00NN.md` para NN em 01..15 — o texto v2, já no clone.

- [ ] **Step 1: Clonar o repo e conferir o estado**

```bash
rm -rf /tmp/rota-docs && gh repo clone rotasaude/docs /tmp/rota-docs
grep -rn '_refundacao' /tmp/rota-docs/docs
```

Expected: três linhas — `docs/README.md:23`, `docs/README.md:41`, `docs/adr/README.md:93` — todas apontando para um `_refundacao/` que não existe no clone.

- [ ] **Step 2: Restaurar o corpus v1**

```bash
OLD=~/Development/ioit.solutions/boxed/rota-saude-old
mkdir -p /tmp/rota-docs/_v1/adr
cp $OLD/docs/adr/0*.md $OLD/docs/adr/README.md $OLD/docs/adr/nota-revisao-operacional.md /tmp/rota-docs/_v1/adr/
ls /tmp/rota-docs/_v1/adr | wc -l
```

Expected: `27` (0001..0025, README.md, nota-revisao-operacional.md). Não copie `prompts/` — é material de processo, não decisão.

- [ ] **Step 3: Restaurar o `_refundacao`**

```bash
cd /tmp && unzip -o ~/Development/ioit.solutions/boxed/rota-saude-old/refundacao.zip -d /tmp/unz -x '__MACOSX/*'
rm -rf /tmp/unz/__MACOSX
mv /tmp/unz/_refundacao /tmp/rota-docs/_refundacao
ls /tmp/rota-docs/_refundacao
```

Expected: `PROVENIENCIA.md`, `00-destilado.md`, `01-mapa-consolidacao.md`, `06-validacao-movimento-1.md`, `07-inventario-codigo.md`, `08-plano-reconciliacao.md`, `09-execucao-api.md`, `10-execucao-contracts.md`, `decisao-lifecycle-protocolo.md`. Nenhum arquivo começando com `._`.

- [ ] **Step 4: Escrever o README do arquivo**

Crie `/tmp/rota-docs/_v1/README.md`:

```markdown
# Corpus v1 — arquivo read-only

Os 25 ADRs originais (com cadeia de emendas) que a refundação consolidou nos 15
ADRs lineares de [`docs/adr/`](../docs/adr/README.md). Preservados como
**arquivo histórico**: não são decisões vigentes e não devem ser citados por
código novo.

Para saber onde cada ADR v1 foi parar, veja
[`docs/adr/DE-PARA.md`](../docs/adr/DE-PARA.md). Para ver como a consolidação
foi feita, veja [`_refundacao/`](../_refundacao/PROVENIENCIA.md).

O ADR v1 0026 nunca existiu como arquivo — era uma decisão não-arquivada,
absorvida direto pelo v2 0015 (contratos e versionamento).
```

- [ ] **Step 5: Consertar o ponteiro do DE-PARA**

Em `/tmp/rota-docs/docs/adr/DE-PARA.md`, substitua a linha 5:

```
órfã. O corpus v1 é preservado read-only no histórico local do projeto (fora deste repositório).
```

por:

```
órfã. O corpus v1 é preservado read-only em [`_v1/adr/`](../../_v1/README.md).
```

Faça o mesmo em `/tmp/rota-docs/docs/adr/README.md` (por volta da linha 92), trocando "no histórico local do projeto (fora deste repositório)" por "em [`_v1/adr/`](../../_v1/README.md)".

- [ ] **Step 6: Verificar que todo link resolve**

```bash
cd /tmp/rota-docs
for f in docs/README.md docs/adr/README.md docs/adr/DE-PARA.md; do
  grep -oE '\]\(([^)]+)\)' $f | sed 's/](//;s/)//' | grep -v '^http' | while read -r rel; do
    target=$(cd "$(dirname $f)" && cd "$(dirname "$rel")" 2>/dev/null && pwd)/$(basename "$rel")
    [ -e "$target" ] || echo "QUEBRADO: $f -> $rel"
  done
done
```

Expected: nenhuma linha `QUEBRADO`. Se aparecer alguma, conserte o link antes de commitar.

- [ ] **Step 7: Commit e push**

```bash
cd /tmp/rota-docs
git add _v1 _refundacao docs/adr/DE-PARA.md docs/adr/README.md
git commit -m "Restore the v1 ADR archive and the refundacao provenance folder"
git push
```

---

### Task 2: Inventário mecânico dos ponteiros

**Files:**
- Create: `apps/api/script/adr_pointer_inventory.rb`
- Create (gerado, não commitado no api): `/tmp/adr-inventory.tsv`

**Interfaces:**
- Produces: `/tmp/adr-inventory.tsv` — TSV com colunas `file`, `line`, `ref`, `context`, uma linha por ocorrência. É a entrada das Tasks 3–11.

- [ ] **Step 1: Escrever o script**

Crie `apps/api/script/adr_pointer_inventory.rb`:

```ruby
#!/usr/bin/env ruby
# frozen_string_literal: true
# Inventário de ponteiros de ADR no apps/api. Rode da raiz do repo api:
#   ruby script/adr_pointer_inventory.rb > /tmp/adr-inventory.tsv
#
# encoding: "UTF-8" é obrigatório — os comentários são acentuados e o default
# externo do ambiente pode ser US-ASCII, o que faz o scan levantar
# ArgumentError: invalid byte sequence.
ROOTS = %w[app config db lib spec deploy].freeze
VALID = (1..15).freeze

paths = (ROOTS.flat_map { |r| Dir.glob("#{r}/**/*.{rb,yml,yaml,erb,rake,md}") } + Dir.glob("*.md"))
        .sort.uniq

puts %w[file line ref in_range context].join("\t")
paths.each do |path|
  File.readlines(path, encoding: "UTF-8").each_with_index do |line, i|
    line.scan(/ADR[-\s]?(\d{4})/).flatten.uniq.each do |num|
      puts [path, i + 1, "ADR-#{num}", VALID.cover?(num.to_i), line.strip].join("\t")
    end
  end
end
```

- [ ] **Step 2: Rodar e conferir os totais**

```bash
cd apps/api && ruby script/adr_pointer_inventory.rb > /tmp/adr-inventory.tsv
tail -n +2 /tmp/adr-inventory.tsv | wc -l
tail -n +2 /tmp/adr-inventory.tsv | cut -f1 | sort -u | wc -l
tail -n +2 /tmp/adr-inventory.tsv | cut -f4 | sort | uniq -c
```

Expected: `225` ocorrências, `140` arquivos, `114 false` / `111 true`. Se os números divergirem, o repo mudou desde o levantamento — reconfira antes de seguir, não ajuste o script para bater com o esperado.

- [ ] **Step 3: Commit do script**

```bash
git -C apps/api add script/adr_pointer_inventory.rb
git -C apps/api commit -m "Add a script that inventories ADR pointers across the api"
```

---

### Task 3: Guarda de regressão (spec que começa vermelha)

**Files:**
- Create: `apps/api/spec/adr_pointers_spec.rb`

**Interfaces:**
- Consumes: nada. Lê o próprio repo.
- Produces: uma spec que falha enquanto existir ref fora de 0001..0015. As Tasks 4–11 a deixam verde.

**Limite honesto:** esta guarda pega só refs **fora da faixa**. Ela **não** pega um `ADR-0009` que continuou com o sentido v1. Essa classe é protegida pela tabela de adjudicação e pela revisão, não por automação — não escreva no comentário da spec que ela garante mais do que garante.

- [ ] **Step 1: Write the failing test**

Crie `apps/api/spec/adr_pointers_spec.rb`:

```ruby
require "rails_helper"

# Guarda de numeração do corpus de ADRs. O v2 vai de 0001 a 0015; qualquer
# outro número é da numeração v1, que foi aposentada.
#
# Escopo desta guarda: ela pega ponteiro FORA da faixa. Ela NÃO pega um
# ADR-0009 que continuou querendo dizer o 0009 do v1 — números baixos existem
# nas duas numerações com sentidos diferentes. Contra essa classe o que vale é
# docs/adr/RECONCILIACAO-PONTEIROS.md (no repo docs) e a revisão.
RSpec.describe "ADR pointers" do
  ROOTS = %w[app config db lib spec deploy].freeze
  VALID_RANGE = (1..15).freeze
  SELF_PATH = "spec/adr_pointers_spec.rb"

  def out_of_range
    Dir.chdir(Rails.root) do
      paths = (ROOTS.flat_map { |r| Dir.glob("#{r}/**/*.{rb,yml,yaml,erb,rake,md}") } +
               Dir.glob("*.md")).sort.uniq - [SELF_PATH]

      paths.flat_map do |path|
        File.readlines(path, encoding: "UTF-8").each_with_index.flat_map do |line, i|
          line.scan(/ADR[-\s]?(\d{4})/).flatten.uniq
              .reject { |num| VALID_RANGE.cover?(num.to_i) }
              .map { |num| "#{path}:#{i + 1} → ADR-#{num}" }
        end
      end
    end
  end

  it "only points at ADRs that exist in the v2 corpus (0001..0015)" do
    offenders = out_of_range
    expect(offenders).to eq([]),
      "#{offenders.size} ponteiro(s) fora do corpus v2:\n#{offenders.join("\n")}"
  end
end
```

- [ ] **Step 2: Run test to verify it fails**

Run: `docker exec api-dev bundle exec rspec spec/adr_pointers_spec.rb`
Expected: FAIL, listando **114** ponteiros fora da faixa.

- [ ] **Step 3: Commit a guarda vermelha**

Commitar uma spec que falha é deliberado: ela é o placar das Tasks 4–11.

```bash
git -C apps/api add spec/adr_pointers_spec.rb
git -C apps/api commit -m "Add a failing guard spec for out-of-corpus ADR pointers"
```

---

### Task 4: Adjudicar e reescrever — motor, eventos, políticas, mailers, auth

**Files:**
- Modify: todos os arquivos com ref em `apps/api/app/protocols/` (18 refs / 13 arquivos), `app/events/` (6/3), `app/mailers/` (2/2), `app/auth/` (3/2), `app/policies/` (1/1) — **30 refs / 21 arquivos**
- Modify: `/tmp/rota-docs/docs/adr/RECONCILIACAO-PONTEIROS.md` (criar nesta task)

**Interfaces:**
- Consumes: `/tmp/adr-inventory.tsv` (Task 2), `/tmp/rota-docs/_v1/adr/` e `/tmp/rota-docs/docs/adr/` (Task 1).
- Produces: `docs/adr/RECONCILIACAO-PONTEIROS.md` com cabeçalho e as linhas desta camada. As Tasks 5–11 só apendam.

- [ ] **Step 1: Filtrar o inventário desta camada**

```bash
grep -E '^app/(protocols|events|mailers|auth|policies)/' /tmp/adr-inventory.tsv | cut -f1,2,3,5
```

Expected: 30 linhas.

- [ ] **Step 2: Criar a tabela de reconciliação**

Crie `/tmp/rota-docs/docs/adr/RECONCILIACAO-PONTEIROS.md`:

```markdown
# Reconciliação dos ponteiros de ADR no código (v1 → v2)

Registro de decisão, uma linha por ocorrência. A reescrita dos comentários em
`rotasaude/api` foi **derivada desta tabela** — se um ponteiro do código não
bate com a linha aqui, a tabela é a fonte de verdade e o código está errado.

Método e mapa v1→v2: `docs/adr/DE-PARA.md`. Texto v1: [`_v1/adr/`](../../_v1/README.md).

Legenda de `Decisão`: `v1→v2` (era v1, reescrito) · `já v2` (não mexido) ·
`DÚVIDA` (não mexido, pendente de revisão humana).

| Ocorrência | Antes | Depois | Decisão | Justificativa |
|---|---|---|---|---|
```

- [ ] **Step 3: Adjudicar cada uma das 30 ocorrências**

Siga o **Procedimento de adjudicação** no topo deste plano, ocorrência por
ocorrência. Para cada uma, apende uma linha na tabela.

Exemplo trabalhado, para calibrar o nível de rigor esperado —
`app/protocols/outcome.rb:1` diz `# Value Object de saída do motor. Imutável. Ver ADR-0015.`:

- v1 0015 = "Classificação: prioridade e explicabilidade" → o DE-PARA diz que
  virou o `Outcome` do motor, no v2 0009. Casa perfeitamente: o arquivo **é** o
  `Outcome`.
- v2 0015 = "Contracts & versioning" → não tem nada a ver com um value object
  do motor.
- Decisão: `v1→v2`, `ADR-0015` → `ADR-0009`.
- Linha: `| apps/api/app/protocols/outcome.rb:1 | ADR-0015 | ADR-0009 | v1→v2 | v1 0015 (classificação/explicabilidade) é o Outcome do motor, consolidado no v2 0009 |`

- [ ] **Step 4: Aplicar a reescrita nos 21 arquivos**

Edite cada arquivo trocando o ponteiro pelo decidido na tabela. Deduplique
quando dois v1 colapsarem no mesmo v2 na mesma linha.

- [ ] **Step 5: Verificar**

```bash
docker exec api-dev bundle exec rspec spec/adr_pointers_spec.rb
grep -rE 'ADR[-\s]?00(1[6-9]|2[0-9])' apps/api/app/protocols apps/api/app/events apps/api/app/mailers apps/api/app/auth apps/api/app/policies
```

Expected: a spec ainda falha (sobram as outras camadas), mas com **menos** ofensores que 114. O `grep` não retorna nada nestes diretórios.

Rode também a suíte inteira — a reescrita é só de comentário, nada pode quebrar:

Run: `docker exec api-dev bundle exec rspec`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git -C apps/api add app/protocols app/events app/mailers app/auth app/policies
git -C apps/api commit -m "Point engine, events, mailers, auth and policies at v2 ADR numbers"
cd /tmp/rota-docs && git add docs/adr/RECONCILIACAO-PONTEIROS.md && git commit -m "Record ADR pointer adjudication for the engine layer" && git push
```

---

### Task 5: Adjudicar e reescrever — commands e services

**Files:**
- Modify: `apps/api/app/commands/` (21 refs / 11 arquivos), `apps/api/app/services/` (5/4) — **26 refs / 15 arquivos**
- Modify: `/tmp/rota-docs/docs/adr/RECONCILIACAO-PONTEIROS.md` (apendar)

**Interfaces:**
- Consumes: a tabela criada na Task 4; o inventário da Task 2.
- Produces: mais 26 linhas na tabela.

- [ ] **Step 1: Filtrar o inventário desta camada**

```bash
grep -E '^app/(commands|services)/' /tmp/adr-inventory.tsv | cut -f1,2,3,5
```

Expected: 26 linhas.

- [ ] **Step 2: Adjudicar as 26 ocorrências**

Siga o **Procedimento de adjudicação**. Atenção especial a dois casos desta camada:

- `app/commands/municipality_channels/rotate_token.rb:4` cita `ADR-0011/0023`.
  `0023` é ≥0016 → v1 → 0012 (RBAC). `0011` é ambíguo e cai num **split**: o
  comentário fala de auditar **sem o valor do token**, ou seja custódia de
  secret → v2 **0013**, não 0014.
- Todo command é espinha de escrita (muta + publica evento). Ponteiros a v1
  0003/0004/0006 nesta camada colapsam todos em v2 **0004** — deduplique.

- [ ] **Step 3: Aplicar a reescrita nos 15 arquivos**

Edite cada arquivo conforme a tabela.

- [ ] **Step 4: Verificar**

```bash
grep -rE 'ADR[-\s]?00(1[6-9]|2[0-9])' apps/api/app/commands apps/api/app/services
docker exec api-dev bundle exec rspec
```

Expected: `grep` vazio; suíte verde.

- [ ] **Step 5: Commit**

```bash
git -C apps/api add app/commands app/services
git -C apps/api commit -m "Point commands and services at v2 ADR numbers"
cd /tmp/rota-docs && git add docs/adr/RECONCILIACAO-PONTEIROS.md && git commit -m "Record ADR pointer adjudication for commands and services" && git push
```

---

### Task 6: Adjudicar e reescrever — jobs

**Files:**
- Modify: `apps/api/app/jobs/` — **28 refs / 20 arquivos**
- Modify: `/tmp/rota-docs/docs/adr/RECONCILIACAO-PONTEIROS.md` (apendar)

**Interfaces:**
- Consumes: a tabela; o inventário.
- Produces: mais 28 linhas na tabela.

- [ ] **Step 1: Filtrar o inventário desta camada**

```bash
grep -E '^app/jobs/' /tmp/adr-inventory.tsv | cut -f1,2,3,5
```

Expected: 28 linhas.

- [ ] **Step 2: Adjudicar as 28 ocorrências**

Siga o **Procedimento de adjudicação**. Esta é a camada com mais armadilha,
porque é onde os refs `≤0015` já v2 se concentram:

- `app/jobs/purge_domain_events_job.rb:2` cita `ADR-0005/0014`. O job purga
  `domain_events` por retenção. v2 0014 = "Audit log, retention & LGPD" →
  **casa**. v1 0014 = "resposta ao cidadão fora do lock" → não casa. Decisão:
  **`já v2`, não mexer**. Registre assim na tabela.
- Os jobs de purga (`purge_*`) e o `anonymize_revoked_triage_job` tendem a ser
  v2 0014. Os `concerns/idempotent_consumer.rb` e `concerns/tenant_scoped_job.rb`
  tendem a ser v1 0005/0020 → v2 0005 e 0003. **Confira cada um**, não
  generalize a partir desta dica.

- [ ] **Step 3: Aplicar a reescrita nos 20 arquivos**

Edite cada arquivo conforme a tabela. Os marcados `já v2` ficam intocados.

- [ ] **Step 4: Verificar**

```bash
grep -rE 'ADR[-\s]?00(1[6-9]|2[0-9])' apps/api/app/jobs
docker exec api-dev bundle exec rspec
```

Expected: `grep` vazio; suíte verde.

- [ ] **Step 5: Commit**

```bash
git -C apps/api add app/jobs
git -C apps/api commit -m "Point jobs at v2 ADR numbers"
cd /tmp/rota-docs && git add docs/adr/RECONCILIACAO-PONTEIROS.md && git commit -m "Record ADR pointer adjudication for jobs" && git push
```

---

### Task 7: Adjudicar e reescrever — models e queries

**Files:**
- Modify: `apps/api/app/models/` (27 refs / 17 arquivos), `apps/api/app/queries/` (5/4) — **32 refs / 21 arquivos**
- Modify: `/tmp/rota-docs/docs/adr/RECONCILIACAO-PONTEIROS.md` (apendar)

**Interfaces:**
- Consumes: a tabela; o inventário.
- Produces: mais 32 linhas na tabela.

- [ ] **Step 1: Filtrar o inventário desta camada**

```bash
grep -E '^app/(models|queries)/' /tmp/adr-inventory.tsv | cut -f1,2,3,5
```

Expected: 32 linhas.

- [ ] **Step 2: Adjudicar as 32 ocorrências**

Siga o **Procedimento de adjudicação**. Caso trabalhado desta camada —
`app/models/triage.rb:2` diz `# Ver ADR-0006 (commands) e ADR-0013 (motor de protocolos).`:

- O próprio comentário nomeia o tema de cada número: "commands" e "motor de
  protocolos". v1 0006 = camada de commands → v2 **0004**. v1 0013 = motor de
  protocolos → v2 **0009**. Os dois são v1.
- Resultado: `# Ver ADR-0004 (commands) e ADR-0009 (motor de protocolos).`

Nas queries, `Admin::*Query` são projeções de leitura — ponteiros a v1 0007
(CQRS) viram v2 **0010**.

- [ ] **Step 3: Aplicar a reescrita nos 21 arquivos**

- [ ] **Step 4: Verificar**

```bash
grep -rE 'ADR[-\s]?00(1[6-9]|2[0-9])' apps/api/app/models apps/api/app/queries
docker exec api-dev bundle exec rspec
```

Expected: `grep` vazio; suíte verde.

- [ ] **Step 5: Commit**

```bash
git -C apps/api add app/models app/queries
git -C apps/api commit -m "Point models and queries at v2 ADR numbers"
cd /tmp/rota-docs && git add docs/adr/RECONCILIACAO-PONTEIROS.md && git commit -m "Record ADR pointer adjudication for models and queries" && git push
```

---

### Task 8: Adjudicar e reescrever — controllers

**Files:**
- Modify: `apps/api/app/controllers/` — **22 refs / 14 arquivos**
- Modify: `/tmp/rota-docs/docs/adr/RECONCILIACAO-PONTEIROS.md` (apendar)

**Interfaces:**
- Consumes: a tabela; o inventário.
- Produces: mais 22 linhas na tabela.

- [ ] **Step 1: Filtrar o inventário desta camada**

```bash
grep -E '^app/controllers/' /tmp/adr-inventory.tsv | cut -f1,2,3,5
```

Expected: 22 linhas.

- [ ] **Step 2: Adjudicar as 22 ocorrências**

Siga o **Procedimento de adjudicação**. Nesta camada concentram-se os refs a
v1 0022 (auth/sessão/MFA → v2 **0011**), v1 0023 (RBAC → v2 **0012**) e v1 0018
(topologia dos 4 apps → v2 **0002**). O `admin/api/base_controller.rb` e o
namespace read-only citam v1 0018 → v2 0002.

- [ ] **Step 3: Aplicar a reescrita nos 14 arquivos**

- [ ] **Step 4: Verificar**

```bash
grep -rE 'ADR[-\s]?00(1[6-9]|2[0-9])' apps/api/app/controllers
docker exec api-dev bundle exec rspec
```

Expected: `grep` vazio; suíte verde.

- [ ] **Step 5: Commit**

```bash
git -C apps/api add app/controllers
git -C apps/api commit -m "Point controllers at v2 ADR numbers"
cd /tmp/rota-docs && git add docs/adr/RECONCILIACAO-PONTEIROS.md && git commit -m "Record ADR pointer adjudication for controllers" && git push
```

---

### Task 9: Adjudicar e reescrever — config, lib e deploy

**Files:**
- Modify: `apps/api/config/` (33 refs / 12 arquivos), `apps/api/lib/` (3/2), `apps/api/deploy/` (5/3) — **41 refs / 17 arquivos**
- Modify: `/tmp/rota-docs/docs/adr/RECONCILIACAO-PONTEIROS.md` (apendar)

**Interfaces:**
- Consumes: a tabela; o inventário.
- Produces: mais 41 linhas na tabela.

- [ ] **Step 1: Filtrar o inventário desta camada**

```bash
grep -E '^(config|lib|deploy)/' /tmp/adr-inventory.tsv | cut -f1,2,3,5
```

Expected: 41 linhas.

- [ ] **Step 2: Adjudicar as 41 ocorrências**

Siga o **Procedimento de adjudicação**. Dois pontos desta camada merecem
atenção:

- `config/routes.rb` é o arquivo com as colisões mais evidentes: "Webhook do
  WhatsApp (ADR-0010)" é v1 0010 (borda de ingestão) → v2 **0007**; "Relatório
  público (ADR-0007)" é v1 0007 (CQRS) → v2 **0010**. **Os dois trocam de lugar**
  — reescreva com cuidado, é fácil inverter.
- `deploy/SECRETS.md` e `config/initializers/*encryption*` citam v1 0024
  (provisionamento/custódia) → v2 **0013**, e podem citar v1 0011 (split):
  chave/criptografia → **0013**, retenção → **0014**.

- [ ] **Step 3: Aplicar a reescrita nos 17 arquivos**

- [ ] **Step 4: Verificar**

```bash
grep -rE 'ADR[-\s]?00(1[6-9]|2[0-9])' apps/api/config apps/api/lib apps/api/deploy
docker exec api-dev bundle exec rspec
```

Expected: `grep` vazio; suíte verde. Confira também que a app ainda sobe, já que
`config/` foi tocado: `docker exec api-dev bin/rails runner 'puts "boot ok"'`.

- [ ] **Step 5: Commit**

```bash
git -C apps/api add config lib deploy
git -C apps/api commit -m "Point config, lib and deploy docs at v2 ADR numbers"
cd /tmp/rota-docs && git add docs/adr/RECONCILIACAO-PONTEIROS.md && git commit -m "Record ADR pointer adjudication for config, lib and deploy" && git push
```

---

### Task 10: Adjudicar e reescrever — db (migrations e schema)

**Files:**
- Modify: `apps/api/db/` — **32 refs / 25 arquivos**
- Modify: `/tmp/rota-docs/docs/adr/RECONCILIACAO-PONTEIROS.md` (apendar)

**Interfaces:**
- Consumes: a tabela; o inventário.
- Produces: mais 32 linhas na tabela.

**Nota sobre migrations já aplicadas:** editar comentário de migration já rodada
é seguro — o Rails só olha o timestamp e a classe, e nada aqui muda DDL. Mas
**não mexa em nada além dos comentários**: se você se pegar editando um
`add_column` ou um `create_table`, parou no arquivo errado.

- [ ] **Step 1: Filtrar o inventário desta camada**

```bash
grep -E '^db/' /tmp/adr-inventory.tsv | cut -f1,2,3,5
```

Expected: 32 linhas.

- [ ] **Step 2: Adjudicar as 32 ocorrências**

Siga o **Procedimento de adjudicação**. As migrations de RLS e de
`municipality_id` citam v1 0019 (RLS → v2 **0003**) e v1 0020 (split: mecanismo
de tenant → v2 **0003**). As de `domain_events`/`processed_events` citam v1 0003
e 0005 → v2 **0004** e **0005**.

- [ ] **Step 3: Aplicar a reescrita nos 25 arquivos**

- [ ] **Step 4: Verificar**

```bash
grep -rE 'ADR[-\s]?00(1[6-9]|2[0-9])' apps/api/db
docker exec api-dev bin/rails db:migrate:status | tail -5
docker exec api-dev bundle exec rspec
```

Expected: `grep` vazio; todas as migrations seguem `up`; suíte verde.

- [ ] **Step 5: Commit**

```bash
git -C apps/api add db
git -C apps/api commit -m "Point migrations and schema comments at v2 ADR numbers"
cd /tmp/rota-docs && git add docs/adr/RECONCILIACAO-PONTEIROS.md && git commit -m "Record ADR pointer adjudication for db" && git push
```

---

### Task 11: Adjudicar e reescrever — specs e os `.md` da raiz do api

**Files:**
- Modify: `apps/api/spec/` (5 refs / 5 arquivos), `apps/api/README.md` e `apps/api/RECONCILE_admin_console.md` (9 refs / 2 arquivos) — **14 refs / 7 arquivos**
- Modify: `/tmp/rota-docs/docs/adr/RECONCILIACAO-PONTEIROS.md` (apendar)

**Interfaces:**
- Consumes: a tabela; o inventário.
- Produces: as últimas 14 linhas da tabela.

- [ ] **Step 1: Filtrar o inventário desta camada**

```bash
grep -E '^(spec/|[A-Za-z_]+\.md)' /tmp/adr-inventory.tsv | cut -f1,2,3,5
```

Expected: 14 linhas. **Não** inclua `spec/adr_pointers_spec.rb` — o script já a
inventaria, mas a guarda se exclui, e os números nela são a faixa válida, não
ponteiros.

- [ ] **Step 2: Adjudicar as 14 ocorrências**

Siga o **Procedimento de adjudicação**. `spec/rls/tenant_isolation_spec.rb` diz
"Invariantes do ADR-0019" → v1 0019 (RLS) → v2 **0003**.

- [ ] **Step 3: Aplicar a reescrita nos 7 arquivos**

- [ ] **Step 4: Verificar — a guarda deve ficar VERDE aqui**

```bash
docker exec api-dev bundle exec rspec spec/adr_pointers_spec.rb
```

Expected: **PASS**. Se ainda falhar, a lista de ofensores diz exatamente qual
camada ficou para trás — volte na task correspondente.

Run: `docker exec api-dev bundle exec rspec`
Expected: PASS, suíte inteira.

- [ ] **Step 5: Commit**

```bash
git -C apps/api add spec README.md RECONCILE_admin_console.md
git -C apps/api commit -m "Point specs and api docs at v2 ADR numbers"
cd /tmp/rota-docs && git add docs/adr/RECONCILIACAO-PONTEIROS.md && git commit -m "Record ADR pointer adjudication for specs and api docs" && git push
```

---

### Task 12: Fechamento — conferir a tabela contra o código

**Files:**
- Modify: `/tmp/rota-docs/docs/adr/RECONCILIACAO-PONTEIROS.md` (fecho + pendências)

**Interfaces:**
- Consumes: tudo das tasks anteriores.
- Produces: a tabela final, conferida linha a linha contra o estado do código.

- [ ] **Step 1: Conferir a contagem**

```bash
grep -c '^| apps/api/' /tmp/rota-docs/docs/adr/RECONCILIACAO-PONTEIROS.md
```

Expected: `225`. Menos que isso significa ocorrência não registrada — ache qual
comparando com `cut -f1,2 /tmp/adr-inventory.tsv`.

- [ ] **Step 2: Reinventariar e comparar com a tabela**

```bash
cd apps/api && ruby script/adr_pointer_inventory.rb > /tmp/adr-inventory-after.tsv
tail -n +2 /tmp/adr-inventory-after.tsv | cut -f4 | sort | uniq -c
```

Expected: `225 true`, nenhum `false`.

- [ ] **Step 3: Listar as dúvidas**

```bash
grep 'DÚVIDA' /tmp/rota-docs/docs/adr/RECONCILIACAO-PONTEIROS.md
```

Se houver linhas, apende ao fim da tabela uma seção nomeando-as como pendência
aberta, com o que falta decidir em cada uma. Elas ficam com o ponteiro original
no código — isso é correto, não é dívida escondida, desde que esteja escrito.

- [ ] **Step 4: Fechar a tabela**

Apende ao fim de `RECONCILIACAO-PONTEIROS.md`:

```markdown
## Fecho

- 225 ocorrências inventariadas em 140 arquivos de `rotasaude/api`.
- Guarda de regressão: `apps/api/spec/adr_pointers_spec.rb` — falha se qualquer
  ponteiro sair da faixa 0001..0015.
- Fora do escopo desta reconciliação: `rota-saude/scripts/*.rb` (não estão em
  repositório nenhum) e a cópia local não-versionada de `rota-saude/docs/`.
```

- [ ] **Step 5: Commit final**

```bash
cd /tmp/rota-docs && git add docs/adr/RECONCILIACAO-PONTEIROS.md
git commit -m "Close the ADR pointer reconciliation with the final tally"
git push
```

---

## Verificação final

- [ ] `docker exec api-dev bundle exec rspec` — suíte inteira verde, incluindo a guarda.
- [ ] `grep -rE 'ADR[-\s]?00(1[6-9]|2[0-9])' apps/api` — sem resultado.
- [ ] Cada link de `/tmp/rota-docs/docs/README.md`, `docs/adr/README.md` e `docs/adr/DE-PARA.md` resolve (script da Task 1, Step 6).
- [ ] `gh repo view rotasaude/docs` mostra `_v1/` e `_refundacao/` no HEAD.
- [ ] Amostragem cega: sorteie 10 linhas da tabela, abra o arquivo:linha no código e confirme que o ponteiro bate com a coluna `Depois`.
