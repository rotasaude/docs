# Setup do board de execução — Rota Saúde

Guia operacional do board v2 na organização `rotasaude`. Materializa a decisão
do ADR 0002 ("Board de execução: Project v2 no nível da organização, agregando
issues de todos os repos. Não é repositório.") e os gates do
[ciclo de desenvolvimento](../ciclo-desenvolvimento.md).

Anexos referenciados neste documento:

- `setup-labels.sh` — script bash que cria o set de labels padrão em todos os
  repos da org.
- `.github/ISSUE_TEMPLATE/feature.yml` — template de issue de funcionalidade,
  herdado por todos os repos via repositório `.github` da org.
- `funcionalidades-mvp.csv` — roadmap das 91 funcionalidades MVP (01–07).

## Pré-requisitos

- `gh` CLI autenticado com escopo `project` (`gh auth refresh -s project`).
- Permissão `admin` na org `rotasaude` para criar projeto, labels e arquivos
  no repo `.github`.
- `jq` instalado (alguns passos precisam parsear JSON do `gh`).

Verificação:

```bash
gh auth status                                   # logado?
gh auth status | grep -q "'project'" && echo ok  # escopo project presente?
gh api orgs/rotasaude -q .login                  # acesso à org?
```

## Parte 1 — Project v2 na org

### 1.1 Criar o projeto

```bash
gh project create \
  --owner rotasaude \
  --title "Rota Saúde — Execução"
```

A saída devolve o número do projeto e a URL. Guarde o número em uma variável
de shell para os próximos passos:

```bash
export PROJECT=<número-que-saiu-acima>
export OWNER=rotasaude
```

A partir daqui, todos os comandos `gh project` usam `$PROJECT` e `$OWNER`.

### 1.2 Custom fields

Três campos novos. O quarto eixo — `App` — **não** vira custom field; vira
label do repo. Razão: GitHub Projects v2 não suporta multi-select nativo,
e a maioria das funcionalidades MVP toca mais de uma aplicação (ver
`funcionalidades-mvp.csv`). Labels de repo fazem multi-app naturalmente.

```bash
# Módulo (single-select, 14 opções)
gh project field-create $PROJECT \
  --owner $OWNER \
  --name "Module" \
  --data-type SINGLE_SELECT \
  --single-select-options "mod-01,mod-02,mod-03,mod-04,mod-05,mod-06,mod-07,mod-08,mod-09,mod-10,mod-11,mod-12,mod-13,mod-14"

# Tipo (single-select, 4 opções)
gh project field-create $PROJECT \
  --owner $OWNER \
  --name "Type" \
  --data-type SINGLE_SELECT \
  --single-select-options "feature,chore,hotfix,adr"

# Target release (texto livre, ex.: triagem-v0.2)
gh project field-create $PROJECT \
  --owner $OWNER \
  --name "Target release" \
  --data-type TEXT
```

Para conferir:

```bash
gh project field-list $PROJECT --owner $OWNER
```

### 1.3 Editar o field `Status` — pela UI

**Limitação conhecida do CLI:** o field built-in `Status` não pode ser
editado via `gh`. Os passos abaixo são manuais na UI do board, **uma vez só**.

1. Abrir o board em `https://github.com/orgs/rotasaude/projects/$PROJECT`.
2. `Settings` (canto superior direito) → `Custom fields` → `Status` → `Edit`.
3. Renomear / substituir as opções padrão para refletir os quatro estados
   do ciclo:

| Estado | Cor sugerida | Significado | Gate de entrada |
|---|---|---|---|
| `Não iniciado` | cinza | issue com escopo completo (passo 1 do ciclo) | escopo aprovado |
| `Em curso` | amarelo | PR aberto, em implementação | branch criada, PR draft |
| `Implementado` | azul | PR merged, 3 camadas de teste passando | code review + CI verde |
| `Verificado` | verde | doc atualizada, ADR-pai referencia | revisão humana final |

4. Remover `Done`. **Não** é o mesmo que `Verificado` — o ciclo separa
   intencionalmente "implementado" de "verificado" para que dívida de
   verificação não se esconda. Release não é estado de card, é tag git por
   marco de módulo.

### 1.4 Views — pela UI

**Limitação conhecida do CLI:** views não podem ser criadas via `gh`.
Os passos são manuais, uma vez só. Total: 12 views.

Para cada view, `+ New view` no rodapé, escolher `Table` (ou `Board` para a
de "Em curso"), aplicar filtro, salvar.

#### Views por app (filtra por label do repo)

| Nome da view | Tipo | Filtro | Agrupar por |
|---|---|---|---|
| `App: api` | Table | `label:"app:api"` | `Module` |
| `App: admin` | Table | `label:"app:admin"` | `Module` |
| `App: dashboard` | Table | `label:"app:dashboard"` | `Module` |
| `App: wpda` | Table | `label:"app:wpda"` | `Module` |

#### Views por módulo (sete, uma por módulo MVP)

| Nome da view | Tipo | Filtro | Agrupar por |
|---|---|---|---|
| `Módulo: 01 WhatsApp` | Table | `Module:mod-01` | `Status` |
| `Módulo: 02 Conversação` | Table | `Module:mod-02` | `Status` |
| `Módulo: 03 Triagem` | Table | `Module:mod-03` | `Status` |
| `Módulo: 04 Relatórios` | Table | `Module:mod-04` | `Status` |
| `Módulo: 05 Dashboard` | Table | `Module:mod-05` | `Status` |
| `Módulo: 06 Identidade/Acesso` | Table | `Module:mod-06` | `Status` |
| `Módulo: 07 LGPD/Auditoria` | Table | `Module:mod-07` | `Status` |

(Views dos módulos 08–14 podem ser criadas quando saírem de stub.)

#### View geral

| Nome da view | Tipo | Filtro | Agrupar por |
|---|---|---|---|
| `Em curso (todos)` | Board | `Status:"Em curso"` | (sem agrupamento — kanban) |

## Parte 2 — Labels nos repos

Set de labels obrigatório, replicado nos seis repos da org (`api`, `admin`,
`dashboard`, `wpda`, `contracts`, `docs`). Labels são por-repo no GitHub —
não há label de org. O script `setup-labels.sh` rodado uma vez cria tudo.

Set:

| Família | Labels |
|---|---|
| Módulo | `mod-01` … `mod-14` |
| App | `app:api`, `app:admin`, `app:dashboard`, `app:wpda`, `app:contracts`, `app:docs` |
| Tipo | `type:feature`, `type:chore`, `type:hotfix`, `type:adr` |
| Risco | `risco:baixo`, `risco:medio`, `risco:alto` |

Rodar:

```bash
chmod +x setup-labels.sh
./setup-labels.sh
```

O script é idempotente — rodar duas vezes não duplica nem quebra.

## Parte 3 — Template de issue de funcionalidade

Vive em `.github/ISSUE_TEMPLATE/feature.yml` **no repositório `.github` da
org**. O GitHub propaga automaticamente esse template para todos os outros
repos da org que não tenham template próprio — exatamente o "workflows
reutilizáveis no `.github` da org" do ADR 0002.

Conteúdo no arquivo anexo `feature.yml`. Para instalar:

```bash
# clonar o repo .github da org
git clone git@github.com:rotasaude/.github.git
cd .github
mkdir -p .github/ISSUE_TEMPLATE
cp /caminho/para/feature.yml .github/ISSUE_TEMPLATE/feature.yml
git add .github/ISSUE_TEMPLATE/feature.yml
git commit -m "chore: template de issue de funcionalidade"
git push origin main
```

Campos obrigatórios derivam direto do passo 1 do ciclo de desenvolvimento:
módulo, superfície(s), ADRs governantes, critério de aceite, camadas de
teste, out-of-scope. Sem qualquer um deles, a issue não sai de
`Não iniciado` — invariante do ciclo.

## Parte 4 — Roadmap das 91 funcionalidades MVP

Em `funcionalidades-mvp.csv` (anexo). Schema:

| Coluna | Conteúdo |
|---|---|
| `f_id` | `F-NN.M` |
| `title` | título da funcionalidade (idêntico ao módulo) |
| `module` | `mod-NN` |
| `apps` | lista `\|`-separada (ex.: `api\|dashboard`) |
| `adrs` | lista `,`-separada (ex.: `0009,0012`) |
| `type` | sempre `feature` no MVP |

### Por que não importar para o board agora

O CSV é **fonte da verdade do escopo planejado**, não inventário de tarefas
abertas. Três razões para não jogar as 91 linhas como draft items no board
desde já:

1. **Multi-app vira labels, não custom field.** Funcionalidades como F-06.9
   (`api`+`admin`+`dashboard`) não cabem num field single-select. Como labels
   são por-repo, só fazem sentido quando a issue existe num repo — não em
   draft.
2. **Draft no board pollui métricas.** Profundidade da fila `Em curso`,
   tempo médio em cada estado, etc., ficam falseados se 91 items estiverem
   parados em `Não iniciado` por meses.
3. **A issue real nasce quando o trabalho começa.** No passo 1 do ciclo,
   escopo completo é pré-requisito de `Não iniciado`. Backfillar 91 issues
   sem escopo já viola o gate.

### Fluxo recomendado

Quando uma funcionalidade entrar no ciclo (passo 1):

1. Olhar a linha do CSV para confirmar F-ID, módulo, apps e ADRs.
2. Abrir issue **em cada repo afetado** com título `F-NN.M — <titulo>`
   (mesmo título em todos), corpo via template (Parte 3), labels
   `mod-NN`, `app:<repo>`, `type:feature`.
3. Adicionar cada issue ao board:
   ```bash
   gh project item-add $PROJECT --owner $OWNER --url <url-da-issue>
   ```
4. Vincular as issues entre si (cross-references no corpo: `Parte de
   rotasaude/api#42`).

Para issues que tocam um único repo (a maioria das do `api`), o passo 2 é só
uma issue.

## Convenções operacionais

### Título de issue

Sempre `F-NN.M — <título exato do módulo>`. O F-ID é o vínculo entre board
e `docs/modulos/`. Exemplo: `F-06.9 — Convites`.

### Multi-app

Labels `app:*` no repo + adicionar a issue ao board. Cada repo afetado tem
sua própria issue, mesmo F-ID, vinculadas por cross-reference. **Não** tente
representar multi-app num único custom field do project.

### Referência a ADR no PR

Sempre por URL completa
(`https://github.com/rotasaude/docs/blob/main/docs/adr/0009.md`),
nunca por path relativo — invariante do ADR 0002 (código e ADR vivem em
repos distintos).

### Sub-issues vs issues independentes

Funcionalidade pequena (1 PR, 1 dev, < 1 dia) = uma issue. Funcionalidade
grande que se quebra em PRs separados = issue-mãe (a "feature") + sub-issues
(tarefas), todas com mesmo `mod-NN` e `app:*`. Sub-issues herdam o F-ID
da mãe (`F-06.9.1`, `F-06.9.2`, etc.) — convenção informal, não obrigatória.

### Closes de issues no PR

Toda PR fecha exatamente uma issue via `Closes #N` na descrição. PR sem
issue vinculada não passa do review — não é hotfix nem chore.

### Hotfix

Não passa pelo board. Branch `hotfix/<n>`, PR direto, label `type:hotfix`
na issue retrospectiva (se virou regra), ADR retroativo se for o caso.
Está documentado em ["O que NÃO entra neste ciclo"](../ciclo-desenvolvimento.md).

## Validação do setup

Depois de tudo, conferir:

```bash
# Project existe?
gh project list --owner rotasaude

# Fields corretos?
gh project field-list $PROJECT --owner $OWNER

# Labels no api?
gh label list --repo rotasaude/api | grep -E '^(mod-|app:|type:|risco:)'
```

Spot-check funcional: criar uma issue-teste no repo `api` usando o template,
verificar que ela aparece nas views `App: api` e `Módulo: 0X` corretas,
deletar a issue-teste.

## O que este setup NÃO faz

Honesto, para evitar surpresa:

- **Não automatiza** mover cards entre estados quando um PR é mergeado.
  Isso seria uma GitHub Action a parte. Se virar dor, abre issue
  `type:chore` no `.github`.
- **Não importa** as 91 funcionalidades MVP automaticamente para o board.
  Pelas razões da Parte 4 — board acompanha trabalho em curso, não escopo
  planejado.
- **Não configura** branch protection nem required checks. Isso é setup
  de cada repo, separado, e deveria virar um runbook próprio em
  `docs/operacao/`.
- **Não cria** webhook nem integração com nada externo (Slack, etc.).
  Notificação operacional vive em outro lugar do corpus (item em aberto
  do ADR 0006).
