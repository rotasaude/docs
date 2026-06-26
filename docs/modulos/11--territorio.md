# Módulo 11 — Território

- **Estado:** Stub
- **Tipo:** Estratégico (pós-MVP)

## Escopo (preliminar)

Modelagem territorial da cidade — bairros, microrregiões, áreas de
cobertura de unidade. Base para campanhas (módulo 12) e analytics
(módulo 14) por território, e para roteamento paciente → unidade mais
próxima.

## ADRs governantes

A definir. Provavelmente:

- ADR de granularidade territorial (CEP, polígonos, áreas IBGE).
- ADR de georreferenciamento (PostGIS ou simples).

## Superfícies

| Superfície | Papel previsto |
|---|---|
| `api` | Modelos territoriais, queries geo |
| `admin` | Importação de base territorial nacional |
| `dashboard` | Visualização territorial da cidade |
| `wpda` | — |

## Pré-requisitos

- Nenhum interno; depende de definição estratégica do produto.

## Funcionalidades planejadas

_(a detalhar)_

## Histórico
