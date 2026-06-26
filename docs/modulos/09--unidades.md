# Módulo 09 — Unidades

- **Estado:** Stub
- **Tipo:** Estratégico (pós-MVP)

## Escopo (preliminar)

Cadastro de unidades de saúde da cidade (UBS, UPA, postos, hospitais
parceiros), com endereço, especialidades atendidas, horário, capacidade.
Base para roteamento de paciente após triagem.

## ADRs governantes

A definir. Provavelmente:

- ADR de modelo de unidade (tipo, hierarquia, vínculo com município).
- ADR de geolocalização (uso ou não para roteamento).

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

_(a detalhar)_

## Histórico
