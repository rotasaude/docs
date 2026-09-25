# Rota Saúde — Corpus de governança (v2)

Porta de entrada da documentação do Rota Saúde, o repositório `docs` da
topologia multi-repo (ADR 0002). Aqui vivem as decisões de arquitetura, o mapa
de módulos, as specs e os planos de cada entrega, o ciclo de desenvolvimento e
os runbooks de operação. Não deploya; é lido por humanos e por agentes.

## O produto em uma tela

O Rota Saúde faz a triagem de saúde do cidadão em nome da prefeitura e leva o
resultado até o atendimento presencial. Cada cidade é um tenant com banco
próprio. O ecossistema é um backend e quatro frontends, cada um no seu
repositório da org `rotasaude`:

| Repo | Papel | Quem usa |
|---|---|---|
| `api` | Backend único (Rails 8): regra de negócio, bancos, filas, todas as APIs | — |
| `wpda` | Canal web do cidadão (ADR 0017) e relatório público por token | Cidadão |
| `dashboard` | Painel da cidade: operação, protocolos assinados (ADR 0016), equipe, atendimento presencial (ADR 0018) | Equipe da prefeitura |
| `admin` | Console da plataforma: catálogo e provisionamento de cidades, entrada numa cidade por grant | Operador da plataforma |
| `maintenance` | Acesso técnico a todas as cidades via GraphQL; só development/staging | Mantenedor (superusuário) |
| `contracts` | Contratos compartilhados: eventos, schema de protocolo, tipos e design tokens (ADR 0015) | — |
| `docs` | Este corpus | — |

O README de cada app descreve o papel dele em detalhe, como rodar em dev e as
suas armadilhas.

## Navegação

| Caminho | O que é |
|---|---|
| [`adr/`](adr/README.md) | Decisões de arquitetura: 20 ADRs lineares (`0001`–`0020`), mais os itens em aberto |
| [`superpowers/specs/`](superpowers/specs/) | Specs de design de cada entrega (`AAAA-MM-DD-<tema>-design.md`) |
| [`superpowers/plans/`](superpowers/plans/) | Planos de implementação, tarefa por tarefa, derivados das specs |
| [`modulos/`](modulos/README.md) | Mapa dos 14 módulos funcionais: escopo, superfícies, F-IDs, ADRs, critério de fechamento |
| [`funcionalidades-mvp.csv`](funcionalidades-mvp.csv) | As 91 funcionalidades do MVP (F-IDs), espelhadas no board do GitHub Project #1 |
| [`ciclo-desenvolvimento.md`](ciclo-desenvolvimento.md) | Como o trabalho flui do ADR ao PR (ADR → módulo → funcionalidade → tarefa) |
| [`operacao/`](operacao/README.md) | Runbooks operacionais; a maior parte ainda a produzir |
| `relatorios/` | Relatórios de drift (gerados; vazio hoje) |
| [`_v1/`](_v1/README.md) | Arquivo read-only do corpus anterior (25 ADRs com cadeia de emendas) |
| [`_refundacao/`](_refundacao/PROVENIENCIA.md) | Como o v1 virou v2: destilado, mapa de consolidação, validação e reconciliação com o código |

### Como os artefatos se relacionam

- **ADR** responde "por que decidimos assim". É imutável: uma mudança de
  decisão vira ADR novo, nunca edição.
- **Spec** desenha uma entrega concreta à luz dos ADRs. Quando a entrega
  muda uma decisão, a spec aponta o ADR que precisa nascer.
- **Plano** quebra a spec em tarefas com teste, na ordem de execução.
- **Módulo** e **funcionalidade** organizam o trabalho no board; o
  `ciclo-desenvolvimento.md` descreve o fluxo.

Os runbooks que já estão maduros moram hoje no README do `api` (ciclo de vida
da cidade, migração, worker por cidade, chave de cifra por cidade, restore),
porque nasceram junto do código. `operacao/` é o destino deles quando forem
extraídos.

## Convenções do corpus

- **Numeração linear.** Cada ADR decide **uma** coisa e descreve o **estado
  final**, sem cadeia de emendas.
- **Identificadores em inglês.** Todo modelo, tabela, classe, comando, evento e
  contrato usa nomes em inglês (`triages`, `consents`, `CompleteTriage`),
  conforme o [ADR 0001](adr/0001.md). A prosa é em português.
- **Commits em inglês**, no formato Conventional Commits com o tipo por
  extenso (`feat`, `fix`, `refactor`, `docs`…), em todos os repos.

## Isolamento entre cidades

O Rota Saúde usa **um banco por cidade** ([ADR 0020](adr/0020.md)), que
substituiu o isolamento por Row-Level Security do [ADR 0003](adr/0003.md). Os
ADRs 0011, 0012 e 0013 ainda descrevem o modelo anterior em alguns pontos
(identidade global, operador como membership de cidade nula, chave única); onde
conflitarem com o ADR 0020, ele vence.

## Proveniência

O v2 nasceu de um corte limpo do v1: numeração nova, idioma único, sem emendas.
O v1 fica intocado em [`_v1/`](_v1/README.md), e a correspondência entre os dois
está em [`adr/DE-PARA.md`](adr/DE-PARA.md). Os ponteiros de ADR no código foram
reconciliados para o v2 em 2026-09-11
([`adr/RECONCILIACAO-PONTEIROS.md`](adr/RECONCILIACAO-PONTEIROS.md)).
