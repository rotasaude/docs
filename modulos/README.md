# Módulos — Índice

> Mapa dos módulos funcionais do rota-saúde. Cada módulo é um corte vertical
> de funcionalidade (escopo, superfícies, F-IDs, ADRs governantes, critério de
> fechamento). Os arquivos são numerados `NN--<slug>.md`; os links abaixo
> apontam para o **nome real do arquivo** em disco.

## Convenções

- **Numeração:** `01`–`35`, dois dígitos, prefixo do nome do arquivo.
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
- **Tipo:** `MVP` (módulos 01–14, Ciclo 1) ou `Ciclo 2` (15 em diante). O
  tipo `Estratégico (pós-MVP)` deixou de ser usado em 2026-09-30.
- **F-IDs:** funcionalidades têm ID `F-NN.M`, único por módulo. Todo módulo
  fora de `Stub` carrega F-IDs, ADRs governantes e critério de fechamento.
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
| 07 | [LGPD/Auditoria](07--lgpd-auditoria.md) | Fechado | MVP | [0013](../adr/0013.md), [0014](../adr/0014.md), [0020](../adr/0020.md), [0026](../adr/0026.md) |
| 08 | [Agendamento](08--agendamento.md) | Fechado | MVP | [0019](../adr/0019.md) |
| 09 | [Unidades](09--unidades.md) | Fechado | MVP | [0018](../adr/0018.md) |
| 10 | [Profissionais](10--profissionais.md) | Fechado | MVP | [0021](../adr/0021.md), [0019](../adr/0019.md) |
| 11 | [Território](11--territorio.md) | Fechado | MVP | [0023](../adr/0023.md) |
| 12 | [Campanhas](12--campanhas.md) | Fechado | MVP | [0024](../adr/0024.md) |
| 13 | [Acompanhamento](13--acompanhamento.md) | Fechado | MVP | [0018](../adr/0018.md), [0019](../adr/0019.md) |
| 14 | [Analytics](14--analytics.md) | Fechado | MVP | [0025](../adr/0025.md), [0023](../adr/0023.md), [0016](../adr/0016.md), [0020](../adr/0020.md) |
| 15 | [Triagem por perfil](15--triagem-por-perfil.md) | Fechado | Ciclo 2 | [0027](../adr/0027.md), [0009](../adr/0009.md), [0017](../adr/0017.md) |
| 16 | [Modo de prontuário e exportação](16--modo-de-prontuario.md) | Fechado | Ciclo 2 | [0028](../adr/0028.md), [0007](../adr/0007.md), [0020](../adr/0020.md) |
| 17 | [Agenda dos profissionais](17--agenda.md) | Fechado | Ciclo 2 | [0029](../adr/0029.md), [0019](../adr/0019.md), [0021](../adr/0021.md) |
| 18 | [Acolhimento](18--acolhimento.md) | Planejado | Ciclo 2 | [0030](../adr/0030.md), [0019](../adr/0019.md), [0028](../adr/0028.md) |
| 19 | [Consulta (prontuário da APS)](19--consulta.md) | Planejado | Ciclo 2 | [0031](../adr/0031.md), [0028](../adr/0028.md), [0030](../adr/0030.md) |
| 20 | [Linha do tempo do paciente](20--linha-do-tempo.md) | Stub | Ciclo 2 | — |
| 21 | [Exames e laboratório](21--exames.md) | Stub | Ciclo 2 | — |
| 22 | [Procedimentos e faturamento](22--procedimentos.md) | Stub | Ciclo 2 | — |
| 23 | [Linhas de cuidado](23--linhas-de-cuidado.md) | Stub | Ciclo 2 | — |
| 24 | [Atendimento domiciliar e coletivo](24--domiciliar-e-coletivo.md) | Stub | Ciclo 2 | — |
| 25 | [Vigilância epidemiológica](25--vigilancia-epidemiologica.md) | Stub | Ciclo 2 | — |
| 26 | [Regulação de média e alta complexidade](26--regulacao.md) | Stub | Ciclo 2 | — |
| 27 | [Farmácia e almoxarifado](27--farmacia-e-almoxarifado.md) | Stub | Ciclo 2 | — |
| 28 | [Odontologia](28--odontologia.md) | Stub | Ciclo 2 | — |
| 29 | [Sala de vacina](29--sala-de-vacina.md) | Stub | Ciclo 2 | — |
| 30 | [Transporte sanitário e TFD](30--transporte-e-tfd.md) | Stub | Ciclo 2 | — |
| 31 | [Atenção psicossocial (CAPS)](31--caps.md) | Stub | Ciclo 2 | — |
| 32 | [Urgência (UPA)](32--urgencia.md) | Stub | Ciclo 2 | — |
| 33 | [Ouvidoria](33--ouvidoria.md) | Stub | Ciclo 2 | — |
| 34 | [Telessaúde](34--telessaude.md) | Stub | Ciclo 2 | — |
| 35 | [Zoonoses e endemias](35--zoonoses-e-endemias.md) | Stub | Ciclo 2 | — |

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

## Escopo do MVP (Ciclo 1)

- **MVP = os 14 módulos (01–14)**, decisão do usuário em 2026-09-30. Até
  então, 10, 11, 12 e 14 eram "Estratégico (pós-MVP)"; o recorte anterior era
  01–09 e 13 (08, 09 e 13 tinham entrado juntos em 2026-09-26, como a fatia
  presencial dos ADRs 0018 e 0019).
- O MVP fecha quando os 14 módulos estiverem `Fechado`. **Fechado em
  2026-09-30:** o 14 foi o último (verificação no mesmo dia).
- Itens tirados do Ciclo 1 continuam registrados no módulo, marcados "(fora do
  Ciclo 1)": F-10.6 (CNES) e F-10.7 (escalonamento do alerta urgente).

## Ciclo 2 — plataforma da atenção primária

Decidido no brainstorm de 2026-10-05: o Rota Saúde vira **suíte completa**
para disputar edital municipal, com **modo de prontuário por cidade** (registro
ou integrado ao e-SUS PEC). Contexto externo em
[`pesquisa/2026-10-05-mapa-de-integracoes.md`](../pesquisa/2026-10-05-mapa-de-integracoes.md).

- **Ordem:** 15 → 16 → 17 → 18 → 19 → 20 → 21 → 22 → 23 → 24 → 25 → 26. Os
  módulos 15 e 17 não dependem de integração externa. Do 16 em diante, nada vira
  spec antes das decisões de credencial e do uso do PEC em cada cidade.
- **Lacunas dos editais (27–35):** entraram com a decisão de suíte completa,
  numeradas pela frequência nos 15 editais analisados; a ordem entre elas e o
  roteiro 15–26 ainda será decidida pela matriz de requisitos dos editais.
  Laboratório municipal, faturamento BPA e app offline do ACS ampliam os
  módulos 21, 22 e 24.
- **Subprojeto seguinte ao 15:** editor visual completo do protocolo (módulo 03).
- **Linhas de produto separadas** (fora desta numeração, só intenção):
  hospitalar (prontuário hospitalar e de maternidade, gestão hospitalar) e
  vigilância sanitária.

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
| [0018](../adr/0018.md) | 09, 11, 13 |
| [0019](../adr/0019.md) | 08, 10, 11, 13 |
| [0020](../adr/0020.md) | 05, 07, 14 |
| [0021](../adr/0021.md) | 10 |
| [0022](../adr/0022.md) | 05 |
| [0023](../adr/0023.md) | 11, 12, 14 |
| [0024](../adr/0024.md) | 12 |
| [0025](../adr/0025.md) | 14 |

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
