# Módulo 17 — Agenda dos profissionais

- **Estado:** Planejado
- **Tipo:** Ciclo 2

## Escopo

A unidade passa a ter agenda por profissional, recortada dos turnos, e a
triagem pode gerar pedido de agendamento com tipo, prioridade e prazo (ADR
0029). Quem marca é sempre a recepção; o cidadão confirma, cancela, pede outro
horário e recebe lembrete.

**Planejado (F-17.1 a F-17.9):**
- tipos de atendimento: base da plataforma copiada para a cidade, ajustável e
  ampliável, com os CBOs que atendem;
- modelos de agenda no turno (demanda do dia, agendável, bloqueada) e tipo
  padrão do vínculo;
- vagas calculadas e marcação em vaga, com trava de sobreposição no banco;
- encaixe com justificativa e limite por turno;
- pedido de agendamento gerado pela triagem (`scheduling` no protocolo);
- fila da recepção por atraso, prazo e prioridade, e fila "sem unidade";
- "Não posso nesse horário" no wpda;
- lembrete do horário confirmado (caixa de avisos e SMS de texto fixo);
- agendas da unidade e do profissional.

**Fora, por enquanto:** cidadão escolhendo vaga, semana-padrão, agenda
coletiva, lista de espera por cancelamento, indicadores de absenteísmo.

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0029 | Agenda por turno com modelo, vagas calculadas, encaixe, pedido da triagem, remarcação pedida, lembrete |
| 0019 | Pedido e horário, confirmação, expiração, falta |
| 0021 | Profissionais, vínculos, turnos |
| 0023 | Unidade de referência do bairro |
| 0024 | Caixa de avisos e SMS de texto fixo |
| 0027 | Linguagem de condição e construtor visual |

## Superfícies

| Superfície | O que aparece |
|---|---|
| `contracts` | `protocols-v1.5.0` (`scheduling`) |
| `api` | `appointment_types`, `schedule_templates`, colunas novas em `appointments`/`appointment_requests`/turnos/vínculos, `Scheduling::Availability`, `Appointments::Book`/`FitIn`/`RequestReschedule`/`RemindJob`, `Triages::Schedule` |
| `admin` | — |
| `dashboard` | Tipos e modelos (Profissionais), fila, marcação, encaixe e agenda da unidade (Atendimento), Minha agenda, painel Agendamento no editor de protocolo |
| `wpda` | Pedido no resultado e em "Meus horários", "Não posso nesse horário", lembrete na caixa de avisos |

## Pré-requisitos

- Módulos 08 (pedido e horário), 10 (turnos), 11 (unidade de referência), 12
  (caixa de avisos e SMS) e 15 (construtor de condições).

## Funcionalidades planejadas

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-17.1 | Tipos de atendimento: base da plataforma e da cidade | api, dashboard | 0029 |
| F-17.2 | Modelos de agenda e tipo padrão do vínculo | api, dashboard | 0029, 0021 |
| F-17.3 | Vagas calculadas e marcação em vaga, com trava de sobreposição | api, dashboard | 0029 |
| F-17.4 | Encaixe com justificativa e limite | api, dashboard | 0029 |
| F-17.5 | Pedido de agendamento gerado pela triagem | contracts, api, dashboard, wpda | 0029, 0027 |
| F-17.6 | Fila da recepção com prazo e prioridade, e fila "sem unidade" | api, dashboard | 0029, 0023 |
| F-17.7 | "Não posso nesse horário" no wpda | api, wpda, dashboard | 0029, 0019 |
| F-17.8 | Lembrete do horário confirmado (aviso e SMS) | api, wpda | 0029, 0024 |
| F-17.9 | Agendas da unidade e do profissional | api, dashboard | 0029 |

## Riscos herdados

- Horários sobrepostos herdados bloqueiam a migração até alguém decidir.
- Turno cancelado deixa horários "precisa remarcar" dependentes da recepção.
- Prazo do protocolo é previsão; atraso não avisa o cidadão.

## Critério de fechamento do módulo

- F-17.1 a F-17.9 verificadas.
- Suíte de invariante (`spec/invariants/scheduling_invariants_spec.rb`) com os
  invariantes do ADR 0029.
- Runbook da recepção (vaga, encaixe, fila "sem unidade", transição).

## Histórico

- 2026-10-05 — Registrado como `Stub` no portfólio do Ciclo 2.
- 2026-10-05 — Escopo decidido com o usuário; ADR 0029 e spec
  `superpowers/specs/2026-10-05-module-17-scheduling-design.md`. F-17.1 a
  F-17.9 criados (board #1, `Not Started`). Módulo passa a `Planejado`.
