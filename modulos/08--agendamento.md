# Módulo 08 — Agendamento

- **Estado:** Stub
- **Tipo:** Estratégico (pós-MVP)

## Escopo (preliminar)

Marcação de consultas pela cidade para os pacientes triados; visualização
e gestão de agenda pelo paciente no wpda. Depende fortemente do módulo
09 (Unidades) e 10 (Profissionais).

## ADRs governantes

A definir quando entrar no ciclo. Provavelmente:

- ADR de modelo de agenda (slots, recorrência, exceções).
- ADR de janela de comunicação (lembrete WhatsApp, confirmação, no-show).

## Superfícies

| Superfície | Papel previsto |
|---|---|
| `api` | Modelo de agenda, comandos, jobs de lembrete |
| `admin` | — |
| `dashboard` | Visão da agenda da cidade, gestão de slots |
| `wpda` | Agendar, ver agendamentos, cancelar, confirmar |

## Pré-requisitos

- Módulo 09 (Unidades) definido.
- Módulo 10 (Profissionais) definido.
- Janela de 24h do WhatsApp resolvida para lembretes (templates).

## Funcionalidades planejadas

_(a detalhar)_

## Histórico
