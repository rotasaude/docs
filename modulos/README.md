# Módulos — Índice

> Mapa dos módulos funcionais do rota-saúde. Cada módulo é um corte vertical
> de funcionalidade (escopo, superfícies, F-IDs, ADRs governantes, critério de
> fechamento). Os arquivos são numerados `NN--<slug>.md`; os links abaixo
> apontam para o **nome real do arquivo** em disco.

## Convenções

- **Numeração:** `01`–`14`, dois dígitos, prefixo do nome do arquivo.
- **Estado:** `Planejado` (no ciclo MVP) ou `Stub` (esboço pós-MVP, a detalhar).
- **Tipo:** `MVP` (escopo do piloto) ou `Estratégico (pós-MVP)`.
- **F-IDs:** funcionalidades têm ID `F-NN.M`, único por módulo. Os módulos
  MVP (01–07) carregam F-IDs, ADRs governantes e critério de fechamento; os
  stubs (08–14) ainda não.
- **ADRs governantes:** decisões arquiteturais que regem o módulo, em
  [`docs/adr/`](../adr/). `brief` = brief formal do dashboard, **item em
  aberto** (ver [`docs/adr/README.md`](../adr/README.md)); o rascunho está
  preservado no corpus v1 de arquivo.

## Índice

| # | Módulo | Estado | Tipo | ADRs governantes |
|---|---|---|---|---|
| 01 | [WhatsApp](01--whatsapp.md) | Planejado | MVP | [0005](../adr/0005.md), [0007](../adr/0007.md), [0013](../adr/0013.md) |
| 02 | [Conversação](02--conversacao.md) | Planejado | MVP | [0005](../adr/0005.md), [0008](../adr/0008.md) |
| 03 | [Triagem](03--triagem.md) | Planejado | MVP | [0009](../adr/0009.md) |
| 04 | [Relatórios](04--relatorios.md) | Planejado | MVP | [0010](../adr/0010.md) |
| 05 | [Dashboard](05--dashboard.md) | Planejado | MVP | [0003](../adr/0003.md), [0010](../adr/0010.md), brief |
| 06 | [Identidade/Acesso](06--identidade-acesso.md) | Planejado | MVP | [0011](../adr/0011.md), [0012](../adr/0012.md), [0013](../adr/0013.md) |
| 07 | [LGPD/Auditoria](07--lgpd-auditoria.md) | Planejado | MVP | [0003](../adr/0003.md), [0013](../adr/0013.md), [0014](../adr/0014.md) |
| 08 | [Agendamento](08--agendamento.md) | Stub | Estratégico (pós-MVP) | a definir |
| 09 | [Unidades](09--unidades.md) | Stub | Estratégico (pós-MVP) | a definir |
| 10 | [Profissionais](10--profissionais.md) | Stub | Estratégico (pós-MVP) | a definir |
| 11 | [Território](11--territorio.md) | Stub | Estratégico (pós-MVP) | a definir |
| 12 | [Campanhas](12--campanhas.md) | Stub | Estratégico (pós-MVP) | a definir |
| 13 | [Acompanhamento](13--acompanhamento.md) | Stub | Estratégico (pós-MVP) | a definir |
| 14 | [Analytics](14--analytics.md) | Stub | Estratégico (pós-MVP) | a definir |

## Mapa módulo × superfície (MVP)

Legenda: ✓ = o módulo tem superfície no app · — = não toca o app. Cada app é um
repositório próprio (ADR 0002).

| Módulo | `api` | `admin` | `dashboard` | `wpda` |
|---|:---:|:---:|:---:|:---:|
| 01 WhatsApp | ✓ | ✓ | ✓ | — |
| 02 Conversação | ✓ | — | ✓ | — |
| 03 Triagem | ✓ | —¹ | ✓ | ✓ |
| 04 Relatórios | ✓ | — | ✓ | ✓ |
| 05 Dashboard | ✓ | — | ✓ | — |
| 06 Identidade/Acesso | ✓ | ✓ | ✓ | ✓ |
| 07 LGPD/Auditoria | ✓ | ✓ | ✓ | — |

¹ Módulo 03: a autoria de protocolo é da cidade (`dashboard`); o
`admin` atua só como backstop do `platform_operator` (ver módulo 06).

## Recorte MVP × pós-MVP

- **MVP (01–07):** WhatsApp, Conversação, Triagem, Relatórios, Dashboard,
  Identidade/Acesso, LGPD/Auditoria. Escopo do piloto; especificados com F-IDs
  e critério de fechamento.
- **Pós-MVP (08–14):** Agendamento, Unidades, Profissionais, Território,
  Campanhas, Acompanhamento, Analytics. Stubs estratégicos; escopo preliminar,
  ADRs a definir quando entrarem no ciclo.

## Mapa ADR → módulos governados

Referência reversa do índice acima (apenas módulos MVP; stubs ainda não
declaram ADR governante).

| ADR | Módulos |
|---|---|
| [0003](../adr/0003.md) | 05, 07 |
| [0005](../adr/0005.md) | 01, 02 |
| [0007](../adr/0007.md) | 01 |
| [0008](../adr/0008.md) | 02 |
| [0009](../adr/0009.md) | 03 |
| [0010](../adr/0010.md) | 04, 05 |
| [0011](../adr/0011.md) | 06 |
| [0012](../adr/0012.md) | 06 |
| [0013](../adr/0013.md) | 01, 06, 07 |
| [0014](../adr/0014.md) | 07 |

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
| [0003](../adr/0003.md) | Multi-tenant isolation (RLS) | toda tabela de tenant |
| [0004](../adr/0004.md) | Domain events, commands & write atomicity | todo evento/escrita |
| [0005](../adr/0005.md) | Consumer idempotency & side-effect isolation | todo consumer |
| [0006](../adr/0006.md) | Queues & clinical priority | roteamento de jobs |

> `0003` e `0005` são transversais **e** governam módulos específicos
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
| 07 LGPD/Auditoria | todos (RLS universal) |
