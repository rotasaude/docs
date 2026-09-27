# Módulos — Índice

> Mapa dos módulos funcionais do rota-saúde. Cada módulo é um corte vertical
> de funcionalidade (escopo, superfícies, F-IDs, ADRs governantes, critério de
> fechamento). Os arquivos são numerados `NN--<slug>.md`; os links abaixo
> apontam para o **nome real do arquivo** em disco.

## Convenções

- **Numeração:** `01`–`14`, dois dígitos, prefixo do nome do arquivo.
- **Estado:** mede o andamento dos F-IDs do módulo no board, independente do
  tipo:
  - `Stub` — esboço, sem F-IDs.
  - `Planejado` — tem F-IDs, e nenhum está implementado.
  - `Em andamento` — parte dos F-IDs está `Done` ou `Verified`, parte não.
  - `Entregue` — todos os F-IDs estão `Done` ou `Verified`, mas o critério de
    fechamento ainda não foi cumprido (em geral, falta a verificação humana).
  - `Fechado` — critério de fechamento cumprido: todos os F-IDs `Verified` e as
    suítes de invariante presentes.

  Um F-ID novo num módulo `Entregue` o devolve a `Em andamento`.
- **Tipo:** `MVP` (escopo do piloto) ou `Estratégico (pós-MVP)`.
- **F-IDs:** funcionalidades têm ID `F-NN.M`, único por módulo. Todo módulo
  fora de `Stub` carrega F-IDs, ADRs governantes e critério de fechamento,
  seja MVP ou pós-MVP.
- **ADRs governantes:** decisões arquiteturais que regem o módulo, em
  [`docs/adr/`](../adr/). `brief` = brief formal do dashboard, **item em
  aberto** (ver [`docs/adr/README.md`](../adr/README.md)); o rascunho está
  preservado no corpus v1 de arquivo.

## Índice

| # | Módulo | Estado | Tipo | ADRs governantes |
|---|---|---|---|---|
| 01 | [WhatsApp](01--whatsapp.md) | Fechado | MVP | [0005](../adr/0005.md), [0007](../adr/0007.md), [0013](../adr/0013.md) |
| 02 | [Conversação](02--conversacao.md) | Fechado | MVP | [0005](../adr/0005.md), [0008](../adr/0008.md), [0017](../adr/0017.md) |
| 03 | [Triagem](03--triagem.md) | Fechado | MVP | [0009](../adr/0009.md), [0016](../adr/0016.md), [0010](../adr/0010.md), [0020](../adr/0020.md) |
| 04 | [Relatórios](04--relatorios.md) | Fechado | MVP | [0010](../adr/0010.md) |
| 05 | [Dashboard](05--dashboard.md) | Fechado | MVP | [0022](../adr/0022.md), [0010](../adr/0010.md), [0020](../adr/0020.md), brief |
| 06 | [Identidade/Acesso](06--identidade-acesso.md) | Fechado | MVP | [0011](../adr/0011.md), [0012](../adr/0012.md), [0013](../adr/0013.md) |
| 07 | [LGPD/Auditoria](07--lgpd-auditoria.md) | Entregue | MVP | [0013](../adr/0013.md), [0014](../adr/0014.md), [0020](../adr/0020.md) |
| 08 | [Agendamento](08--agendamento.md) | Entregue | MVP | [0019](../adr/0019.md) |
| 09 | [Unidades](09--unidades.md) | Entregue | MVP | [0018](../adr/0018.md) |
| 10 | [Profissionais](10--profissionais.md) | Planejado | Estratégico (pós-MVP) | [0021](../adr/0021.md), [0019](../adr/0019.md) |
| 11 | [Território](11--territorio.md) | Stub | Estratégico (pós-MVP) | a definir |
| 12 | [Campanhas](12--campanhas.md) | Stub | Estratégico (pós-MVP) | a definir |
| 13 | [Acompanhamento](13--acompanhamento.md) | Entregue | MVP | [0018](../adr/0018.md), [0019](../adr/0019.md) |
| 14 | [Analytics](14--analytics.md) | Stub | Estratégico (pós-MVP) | a definir |

## Mapa módulo × superfície (MVP)

Legenda: ✓ = o módulo tem superfície no app · — = não toca o app. Cada app é um
repositório próprio (ADR 0002).

| Módulo | `api` | `admin` | `dashboard` | `wpda` |
|---|:---:|:---:|:---:|:---:|
| 01 WhatsApp | ✓ | ✓ | ✓ | — |
| 02 Conversação | ✓ | — | ✓ | ✓ |
| 03 Triagem | ✓ | ✓¹ | ✓ | ✓ |
| 04 Relatórios | ✓ | — | ✓ | ✓ |
| 05 Dashboard | ✓ | ✓ | ✓ | — |
| 06 Identidade/Acesso | ✓ | ✓ | ✓ | ✓ |
| 07 LGPD/Auditoria | ✓ | ✓ | ✓ | — |
| 08 Agendamento | ✓ | — | ✓ | ✓ |
| 09 Unidades | ✓ | — | ✓ | — |
| 13 Acompanhamento | ✓ | — | ✓ | ✓ |

¹ Módulo 03: a autoria e as assinaturas são da cidade (`dashboard`); o
`admin` só lista os protocolos de cada cidade. O mantenedor publica e ativa
pelo app `maintenance` quando as assinaturas da cidade já existem, sem nunca
assinar (ADR 0016).

## Recorte MVP × pós-MVP

- **MVP (01–09 e 13):** WhatsApp, Conversação, Triagem, Relatórios, Dashboard,
  Identidade/Acesso, LGPD/Auditoria, Agendamento, Unidades e Acompanhamento.
  Escopo do piloto; especificados com F-IDs e critério de fechamento. 08, 09 e
  13 entraram juntos (2026-09-26): são a fatia presencial dos ADRs 0018 e 0019
  (unidade → check-in e desfecho → pedido e horário) e dependem uns dos outros.
- **Pós-MVP (10, 11, 12, 14):** Profissionais, Território, Campanhas e
  Analytics. Entram no ciclo quando ganham ADR e F-IDs: o 10 está planejado
  (ADR 0021), e 11, 12 e 14 seguem como stubs.

## Mapa ADR → módulos governados

Referência reversa do índice acima (stubs ainda não declaram ADR governante).

| ADR | Módulos |
|---|---|
| [0005](../adr/0005.md) | 01, 02 |
| [0007](../adr/0007.md) | 01 |
| [0008](../adr/0008.md) | 02 |
| [0009](../adr/0009.md) | 03 |
| [0010](../adr/0010.md) | 04, 05 |
| [0011](../adr/0011.md) | 06 |
| [0012](../adr/0012.md) | 06 |
| [0013](../adr/0013.md) | 01, 06, 07 |
| [0014](../adr/0014.md) | 07 |
| [0017](../adr/0017.md) | 02 |
| [0018](../adr/0018.md) | 09, 13 |
| [0019](../adr/0019.md) | 08, 10, 13 |
| [0020](../adr/0020.md) | 05, 07 |
| [0021](../adr/0021.md) | 10 |
| [0022](../adr/0022.md) | 05 |

> ADRs adicionais aparecem nas tabelas de F-IDs dos módulos (ex.: 0004, 0005)
> como apoio a funcionalidades específicas, sem serem governantes do módulo
> como um todo. Todos os ADRs citados existem em [`docs/adr/`](../adr/).

## ADRs transversais

Fundacionais / cross-cutting: regem a plataforma como um todo (infra,
contratos, isolamento) e, em geral, não pertencem a um módulo único.

| ADR | Tema | Alcance |
|---|---|---|
| [0001](../adr/0001.md) | Platform & application stack | infra de fila + papéis — todos os jobs |
| [0002](../adr/0002.md) | Repository topology (multi-repo) | organização inteira |
| [0020](../adr/0020.md) | Database per city (substitui o 0003) | todo dado de cidade |
| [0004](../adr/0004.md) | Domain events, commands & write atomicity | todo evento/escrita |
| [0005](../adr/0005.md) | Consumer idempotency & side-effect isolation | todo consumer |
| [0006](../adr/0006.md) | Queues & clinical priority | roteamento de jobs |

> `0020` e `0005` são transversais **e** governam módulos específicos
> (05, 07 / 01, 02); listados aqui pelo alcance universal e no índice pelo papel
> concreto. O cluster aplicado (`0007`–`0014`) **não** é transversal — governa
> módulos concretos e por isso vive no índice, não aqui.

## Dependências entre módulos (MVP)

| Módulo | Depende de |
|---|---|
| 01 WhatsApp | 06, 07 |
| 02 Conversação | 01, 03, 07 |
| 03 Triagem | 02, 04, 06, 07 |
| 04 Relatórios | 01, 03, 06 |
| 05 Dashboard | todos os MVP, 06 |
| 06 Identidade/Acesso | 03, 07 |
| 07 LGPD/Auditoria | todos (banco por cidade, ADR 0020) |
| 08 Agendamento | 06, 09, 13 |
| 09 Unidades | — |
| 13 Acompanhamento | 03, 06, 08, 09 |
