# Módulo 08 — Agendamento

- **Estado:** Em andamento (primeira fatia entregue)
- **Tipo:** Estratégico (pós-MVP)

## Escopo (preliminar)

Marcação de consultas pela cidade para os pacientes triados; visualização
e gestão de agenda pelo paciente no wpda. Depende fortemente do módulo
09 (Unidades) e 10 (Profissionais).

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0019 | Pedido de agendamento e horário: nascem do desfecho de retorno ou encaminhamento; confirmação com prazo, expiração e falta |

Ainda a decidir: agenda de vagas publicada pela unidade, o cidadão escolhendo o horário e lembretes (em aberto no ADR 0019).

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

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-08.1 | Pedido de agendamento nasce do desfecho de retorno ou encaminhamento (fila de pedidos da unidade de destino) | api, dashboard | 0019 |
| F-08.2 | Recepção marca data e hora ou encerra o pedido com justificativa; remarcar cria um horário novo | api, dashboard | 0019 |
| F-08.3 | Agenda do dia da unidade | api, dashboard | 0019 |
| F-08.4 | Cidadão vê, confirma (até 24h antes) ou cancela (com motivo) o horário no `wpda` | api, wpda | 0019 |
| F-08.5 | Horário sem confirmação expira e falta vira `no_show` (jobs); o pedido volta marcado para a fila | api | 0019 |
| F-08.6 | Check-in de horário confirmado (no dia e na unidade) vira atendimento | api, dashboard, wpda | 0019 |

## Histórico
