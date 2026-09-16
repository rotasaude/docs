# Reconciliação dos ponteiros de ADR (numeração v1 → v2)

**Date:** 2026-09-11
**Status:** Approved (análise → decisão do usuário → plan written)
**Module:** governança do corpus (não é um módulo funcional)
**Touches:** repo `rotasaude/docs` (restauração do arquivo v1) + `apps/api` (225 ponteiros em 140 arquivos)
**Não toca:** `admin`, `dashboard`, `wpda` (não citam ADR), `contracts/`

## Problem

A refundação do corpus (v1, 26 ADRs com cadeia de emendas → v2, 15 ADRs
lineares) renumerou tudo. O código **não foi reconciliado**: `apps/api` tem
**225 referências a ADR em 140 arquivos**, quase todas na numeração v1.

O problema não é cosmético. Os números baixos **colidem**: existem tanto no v1
quanto no v2, com significados diferentes.

| Onde | O que o código diz | O que o leitor encontra no v2 | O que ele deveria ler |
|---|---|---|---|
| `config/routes.rb` | "Webhook do WhatsApp (ADR-0010)" | 0010 = CQRS: projeções e snapshots | 0007 (ingestão WhatsApp) |
| `config/routes.rb` | "Relatório público (ADR-0007)" | 0007 = ingestão WhatsApp | 0010 (CQRS/snapshots) |
| `app/protocols/outcome.rb` | "Ver ADR-0015" | 0015 = contratos e versionamento | 0009 (motor de protocolos) |
| `app/models/triage.rb` | "Ver ADR-0006 (commands)" | 0006 = filas e prioridade clínica | 0004 (commands e atomicidade) |

Um ponteiro fora de faixa (`ADR-0022`) pelo menos falha alto — o arquivo não
existe. Um ponteiro colidido manda o leitor para a decisão errada **com
confiança total**. É a pior falha possível num corpus lido por agentes.

## Current state (verified)

### Inventário

225 refs / 140 arquivos em `apps/api` (inclui `deploy/` e os `.md` da raiz), por camada:

| Camada | refs | arquivos |
|---|---:|---:|
| `config` | 33 | 12 |
| `db` | 32 | 25 |
| `app/jobs` | 28 | 20 |
| `app/models` | 27 | 17 |
| `app/controllers` | 22 | 14 |
| `app/commands` | 21 | 11 |
| `app/protocols` | 18 | 13 |
| `.md` da raiz | 9 | 2 |
| `app/events` | 6 | 3 |
| `app/services` | 5 | 4 |
| `spec` | 5 | 5 |
| `deploy` | 5 | 3 |
| `app/queries` | 5 | 4 |
| `app/auth` | 3 | 2 |
| `lib` | 3 | 2 |
| `app/mailers` | 2 | 2 |
| `app/policies` | 1 | 1 |

Por classe de ambiguidade:

- **114 refs fora da faixa 0001..0015** (`ADR-0016`..`ADR-0026`) — inequivocamente v1 (esses números não
  existem no v2). Destas, as ~15 de `ADR-0020` caem num **split**.
- **111 refs dentro da faixa `ADR-0001`..`ADR-0015`** — **mistas**. Verificado: arquivos
  escritos *depois* da refundação ainda usam v1 (`authoring/protocols_controller.rb`,
  30/06, cita `ADR-0022`/`ADR-0019`), mas pelo menos um usa v2
  (`purge_domain_events_job.rb`, 01/07, cita "ADR-0005/0014" — e 0014 **v2** é
  retenção/LGPD, que é exatamente o que o job faz; no v1, 0014 era "resposta ao
  cidadão fora do lock", que não tem nada a ver). **Um sed mecânico corromperia
  esses.**

### O arquivo v1 não estava perdido

Contradizendo o que o próprio corpus afirma, o v1 existe:

- `~/Development/ioit.solutions/boxed/rota-saude-old/docs/adr/` — ADRs
  `0001.md`..`0025.md`, `README.md`, `nota-revisao-operacional.md`, `prompts/`.
  (`0026` nunca foi arquivo: o DE-PARA registra que era "decisão não-arquivada
  no v1", absorvida direto pelo v2 0015.)
- `~/Development/ioit.solutions/boxed/rota-saude-old/refundacao.zip` — o
  `_refundacao/` completo: `PROVENIENCIA.md`, `00-destilado.md`,
  `01-mapa-consolidacao.md`, `06-validacao-movimento-1.md`,
  `07-inventario-codigo.md`, `08-plano-reconciliacao.md`, `09-execucao-api.md`,
  `10-execucao-contracts.md`, `decisao-lifecycle-protocolo.md`. (O zip tem lixo
  `__MACOSX/` a descartar.)

Isso é o que torna a **adjudicação por contexto viável**: dá para ler o texto v1
e decidir se um `ADR-0009` quer dizer "auditoria LGPD" (v1) ou "motor de
protocolos" (v2).

### Links quebrados no repo `docs`

O repo `rotasaude/docs` aponta três vezes para um `_refundacao/` que não existe
nele:

- `docs/README.md:23` e `docs/README.md:41` → `../_refundacao/PROVENIENCIA.md`
- `docs/adr/README.md:93` → `../../_refundacao/PROVENIENCIA.md`

E `docs/adr/DE-PARA.md:5` diz que o v1 está "no histórico local do projeto (fora
deste repositório)" — verdadeiro, mas inacionável: não nomeia onde.

### Cuidado: a cópia local de `docs/` está desatualizada

`rota-saude/docs/` **não é um clone** — é uma cópia solta, e já divergiu do repo:

- local está **atrás** em `docs/adr/DE-PARA.md` e `docs/adr/README.md` (a cópia
  local ainda tem a frase antiga, com um caminho absoluto que aponta para ela
  mesma — um link circular);
- local **não tem** `docs/funcionalidades-mvp.csv` nem `docs/operacao/board-setup.md`
  (estão em `rota-saude/files/`);
- o repo **não tem** `docs/superpowers/` (todos os planos e specs, inclusive
  este, existem só no disco local).

Toda escrita desta entrega tem que ir para um **clone do repo**, não para a
cópia local.

## Decision

**Adjudicação completa** (escolha do usuário): reescrever as 225 referências
para a numeração v2, decidindo caso a caso com o corpus v1 em mãos.

Três entregáveis, não um:

1. **Restaurar o arquivo.** `_v1/` e `_refundacao/` voltam para o repo `docs`,
   e os três links quebrados passam a resolver. Sem isso a adjudicação não é
   reproduzível nem auditável por outra pessoa depois.
2. **Registrar a decisão.** Uma tabela commitada
   (`docs/adr/RECONCILIACAO-PONTEIROS.md`) com uma linha por ocorrência:
   arquivo:linha, ref antiga, ref nova, e **por quê**. A reescrita é derivada
   dessa tabela, não improvisada arquivo a arquivo — é o que torna o resultado
   revisável sem reler os 136 arquivos.
3. **Travar a regressão.** Uma spec (`spec/adr_pointers_spec.rb`) que falha se
   qualquer ref sair da faixa 0001..0015.

### Mapa v1 → v2 (do DE-PARA)

| v1 | v2 | | v1 | v2 |
|---|---|---|---|---|
| 0001 | 0001 | | 0014 | 0005 |
| 0002 | 0001 | | 0015 | 0009 |
| 0003 | 0004 | | 0016 | 0009 |
| 0004 | 0004 | | 0017 | 0009 |
| 0005 | 0005 | | 0018 | 0002 |
| 0006 | 0004 | | 0019 | 0003 |
| 0007 | 0010 | | 0020 | **SPLIT** |
| 0008 | 0006 | | 0021 | 0007 |
| 0009 | 0014 | | 0022 | 0011 |
| 0010 | 0007 | | 0023 | 0012 |
| 0011 | **SPLIT** | | 0024 | 0013 |
| 0012 | 0008 | | 0025 | 0002 |
| 0013 | 0009 | | 0026 | 0015 |

**Splits** (decidir pelo que o código anotado faz):

- **v1 0011** (payload bruto: criptografia e retenção) → **0013** se o trecho é
  sobre chave/criptografia/custódia; **0014** se é sobre retenção/purga.
- **v1 0020** (tenant em eventos e jobs) → **0003** se é o mecanismo
  (`with_tenant`/`SET LOCAL`); **0004** se é o publish carimbado; **0005** se é
  a idempotência do consumidor.

### Limite honesto da guarda

A spec de regressão só pega refs **fora da faixa** 0001..0015. Ela **não** pega
um `ADR-0009` que continuou querendo dizer o 0009 do v1. Essa classe é
inerentemente não-mecanizável: o que a protege é a tabela commitada mais a
revisão. Dizer o contrário seria vender uma garantia que não existe.

## Non-goals

- Não trocar o número por slug estável (opção descartada pelo usuário).
- Não editar o texto dos ADRs v2, nem o DE-PARA além do link de arquivo.
- Não mexer em `rota-saude/scripts/*.rb`, que também citam ADR v1
  (`replay_domain_events.rb` cita `ADR-0009`): esses arquivos não estão em repo
  nenhum. Sai junto com o item 5 (versionar a camada de orquestração).
- Não reconciliar a cópia local `rota-saude/docs/` com o repo — também é item 5.
- Não criar CI (item 3). A spec de guarda é o gancho onde o CI vai pendurar
  depois.
