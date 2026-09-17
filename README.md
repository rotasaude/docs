# Rota Saúde — Corpus de governança (v2)

Porta de entrada do **corpus consolidado** do Rota Saúde — o repositório `docs`
da topologia multi-repo (ADR 0002). Aqui vivem as decisões de arquitetura, o mapa
de módulos, o ciclo de desenvolvimento e os runbooks de operação. Não deploya; é
lido por humanos e por agentes.

## O que é este corpus

Este é o **sucessor consolidado (v2)** do corpus anterior. Ele foi refundado por
um corte limpo:

- **Numeração nova e linear.** Os ADRs v2 (`0001`–`0015`) têm numeração própria,
  sem cadeia de emendas. Cada ADR decide **uma** coisa e descreve seu **estado
  final** — nada de "emendado por", nada de ler três documentos para entender uma
  decisão.
- **Idioma único: inglês.** Todo modelo, tabela, classe, comando, evento e
  contrato usa nomes em inglês (`triages`, `consents`, `Triage`, `CompleteTriage`).
  A convenção é fixada no [ADR 0001](adr/0001.md).
- **História preservada como arquivo.** O corpus anterior (26 documentos com
  cadeia de emendas) permanece **intocado** na pasta de origem, como arquivo
  histórico read-only. A correspondência v1→v2 vive em
  [`_refundacao/`](_refundacao/PROVENIENCIA.md).

> **Estado:** EM CONSTRUÇÃO. O scaffold está erguido (Etapa 2). Os ADRs são
> escritos na Etapa 3; módulos, ciclo e operação são reconciliados na Etapa 5.

## Navegação

| Seção | O que é | Estado |
|---|---|---|
| [`adr/`](adr/README.md) | Decisões de arquitetura (15 ADRs lineares) | Etapa 3 |
| [`modulos/`](modulos/README.md) | Mapa de módulos do ciclo de desenvolvimento | Etapa 5 |
| [`ciclo-desenvolvimento.md`](ciclo-desenvolvimento.md) | Como o trabalho flui dos ADRs ao código | Etapa 5 |
| [`operacao/`](operacao/README.md) | Runbooks operacionais | Etapa 5 |
| `relatorios/` | Relatórios de drift (gerados, não migrados) | — |

## Proveniência

Como este corpus foi construído a partir do v1 está documentado em
[`_refundacao/`](_refundacao/PROVENIENCIA.md): o destilado dos ADRs antigos
(`00-destilado.md`) e o mapa de consolidação aprovado (`01-mapa-consolidacao.md`).
Essa pasta pode ser arquivada depois que a refundação concluir.
