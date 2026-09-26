# Módulo 09 — Unidades

- **Estado:** Em andamento (primeira fatia entregue)
- **Tipo:** Estratégico (pós-MVP)

## Escopo (preliminar)

Cadastro de unidades de saúde da cidade (UBS, UPA, postos, hospitais
parceiros), com endereço, especialidades atendidas, horário, capacidade.
Base para roteamento de paciente após triagem.

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0018 | Cadastro mínimo de unidades (`health_units`: nome, tipo, ativa) como apoio do check-in e do atendimento |

Ainda a decidir: endereço, horário, especialidades e geolocalização (sem refazer a tabela).

## Superfícies

| Superfície | Papel previsto |
|---|---|
| `api` | Modelo `health_units`, commands de CRUD |
| `admin` | — (cadastro é da cidade) |
| `dashboard` | CRUD de unidades da cidade |
| `wpda` | — (consumido indireto via agendamento) |

## Pré-requisitos

- Módulo 11 (Território) — endereço/geolocalização compartilhados.

## Funcionalidades planejadas

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-09.1 | Cadastro mínimo de unidades de saúde (`health_units`: nome, tipo, ativa) mantido pelo `municipal_admin` | api, dashboard | 0018 |
| F-09.2 | Desativar e reativar unidade (desativação recusada com atendimentos abertos) | api, dashboard | 0018 |

## Histórico
