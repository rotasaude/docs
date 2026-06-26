# Módulo 13 — Acompanhamento

- **Estado:** Stub
- **Tipo:** Estratégico (pós-MVP)

## Escopo (preliminar)

Continuidade do cuidado após triagem/consulta — ficha de acompanhamento
do paciente, evolução, retorno agendado, lembretes de medicação,
follow-up automatizado por WhatsApp.

## ADRs governantes

A definir. Provavelmente:

- ADR de modelo de ficha (formato, versionamento, retenção LGPD).
- ADR de continuidade conversacional (relação com a máquina de estados
  do MVP).

## Superfícies

| Superfície | Papel previsto |
|---|---|
| `api` | Modelo de ficha, jobs de follow-up |
| `admin` | — |
| `dashboard` | Visão de ficha por paciente, evolução |
| `wpda` | Paciente vê própria ficha, recebe lembretes |

## Pré-requisitos

- Módulo 04 (Relatórios) — ficha é evolução do snapshot.
- Módulo 08 (Agendamento) — retorno agendado.
- Módulo 10 (Profissionais) — atribuição de responsável.

## Funcionalidades planejadas

_(a detalhar)_

## Histórico
