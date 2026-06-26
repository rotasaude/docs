# Módulo 10 — Profissionais

- **Estado:** Stub
- **Tipo:** Estratégico (pós-MVP)

## Escopo (preliminar)

Cadastro de profissionais de saúde da cidade, vínculo com unidades,
especialidades, agenda de atendimento. Base para alocação de paciente
após triagem e para escalonamento de alerta urgente.

## ADRs governantes

A definir. Provavelmente:

- ADR de modelo de profissional (CNS, conselho de classe, registro).
- ADR de integração com cadastro nacional (CNES).

## Superfícies

| Superfície | Papel previsto |
|---|---|
| `api` | Modelo `professionals`, vínculo com `health_units` |
| `admin` | — |
| `dashboard` | CRUD de profissionais da cidade |
| `wpda` | Identificação do profissional na ficha do paciente |

## Pré-requisitos

- Módulo 09 (Unidades) — profissional vincula a unidade(s).
- Módulo 06 (Identidade/Acesso) — profissional pode ou não ser usuário do
  sistema; decisão a tomar.

## Funcionalidades planejadas

_(a detalhar)_

## Histórico
