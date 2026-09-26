# Módulo 13 — Acompanhamento

- **Estado:** Em andamento (primeira fatia entregue)
- **Tipo:** Estratégico (pós-MVP)

## Escopo (preliminar)

Continuidade do cuidado após triagem/consulta — ficha de acompanhamento
do paciente, evolução, retorno agendado, lembretes de medicação,
follow-up automatizado por WhatsApp.

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0018 | Check-in na unidade, atendimento e desfecho a partir da triagem |
| 0019 | Chamada pelo profissional, estados do atendimento, desfecho de retorno e pedido de agendamento |

Ainda a decidir: ficha de acompanhamento, evolução e follow-up automatizado.

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

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-13.1 | Código de check-in no `wpda` ("Cheguei na unidade"), com finalidade separada do código de validação | api, wpda | 0018 |
| F-13.2 | Check-in na recepção por código + CPF do documento (triagem de até 3 dias; valida o cadastro declarado no mesmo passo) | api, dashboard | 0018, 0017 |
| F-13.3 | Check-in por exceção: busca por CPF com motivo registrado | api, dashboard | 0018 |
| F-13.4 | Atendimento com estados `waiting` → `in_care` → `closed` e fila da unidade pela prioridade da triagem | api, dashboard | 0018, 0019 |
| F-13.5 | Chamada do cidadão pelo `health_professional` (chamar o próximo ou um específico) | api, dashboard | 0019 |
| F-13.6 | Desfecho do atendimento: atendido; encaminhado (unidade e/ou descrição); retorno; saiu sem atendimento | api, dashboard | 0018, 0019 |

## Histórico
