# Módulo 08 — Agendamento

- **Estado:** Entregue
- **Tipo:** Estratégico (pós-MVP)

## Escopo

Continuidade presencial marcada: quando um atendimento termina em **retorno**
ou **encaminhamento** para uma unidade da cidade, nasce um **pedido de
agendamento** na unidade de destino (ADR 0019).

**Entregue:** a recepção marca data e hora (ou encerra o pedido com
justificativa), vê a agenda do dia e remarca criando um horário novo. O
cidadão vê o horário no `wpda` e confirma até 24h antes ou cancela com
motivo. Horário sem confirmação expira, falta vira `no_show`, e nos dois
casos o pedido volta marcado para a fila: ninguém sai dela sem uma pessoa
decidir. No dia, o check-in do horário confirmado vira atendimento.

**Fora, por enquanto:** agenda de vagas publicada pela unidade, o cidadão
escolhendo o horário, lembrete antes do prazo, remarcação pedida pelo
cidadão e encaminhamento para fora da rede da cidade.

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0019 | Pedido de agendamento e horário: nascem do desfecho de retorno ou encaminhamento; confirmação com prazo, expiração e falta |

Ainda a decidir: agenda de vagas publicada pela unidade, o cidadão escolhendo o horário e lembretes (em aberto no ADR 0019).

## Superfícies

| Superfície | O que aparece |
|---|---|
| `api` | `appointment_requests`, `appointments` (só acréscimos e transições previstas), rotas de pedidos, marcação e agenda em `/attendance`, rotas do cidadão em `/citizen/appointments`, jobs de expiração e de falta |
| `admin` | — |
| `dashboard` | Módulo Atendimento: fila de pedidos da unidade, marcação e agenda do dia |
| `wpda` | Horários do cidadão: ver, confirmar, cancelar e gerar o código de check-in do horário |

## Dependências

- Módulo 13 (Acompanhamento) — o pedido nasce do desfecho de um atendimento,
  e o check-in do horário abre um atendimento novo.
- Módulo 09 (Unidades) — a unidade de destino do pedido.
- Módulo 06 (Identidade/Acesso) — o par (CPF, celular) do atendimento de
  origem é quem vê o horário no `wpda`.
- Módulo 10 (Profissionais) — só para a agenda de vagas futura.

## Funcionalidades planejadas

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-08.1 | Pedido de agendamento nasce do desfecho de retorno ou encaminhamento (fila de pedidos da unidade de destino) | api, dashboard | 0019 |
| F-08.2 | Recepção marca data e hora ou encerra o pedido com justificativa; remarcar cria um horário novo | api, dashboard | 0019 |
| F-08.3 | Agenda do dia da unidade | api, dashboard | 0019 |
| F-08.4 | Cidadão vê, confirma (até 24h antes) ou cancela (com motivo) o horário no `wpda` | api, wpda | 0019 |
| F-08.5 | Horário sem confirmação expira e falta vira `no_show` (jobs); o pedido volta marcado para a fila | api | 0019 |
| F-08.6 | Check-in de horário confirmado (no dia e na unidade) vira atendimento | api, dashboard, wpda | 0019 |

## Riscos herdados

- **Clínica (ADR 0019):** sem lembrete, o cancelamento automático pune quem
  não abre o `wpda`. A marca "sem confirmação" na fila é a rede de proteção:
  a recepção pode ligar e remarcar.
- **Operacional (ADR 0019):** a recepção digita data e hora sem agenda de
  vagas; nada impede marcar dois cidadãos no mesmo horário.
- **Operacional:** expiração e falta dependem dos jobs do worker da cidade.
  Worker parado deixa horário vencido aberto e pedido fora da fila.
- **Em aberto:** agenda de vagas, escolha do horário pelo cidadão, lembrete,
  remarcação pedida pelo cidadão e encaminhamento para fora da rede.

## Critério de fechamento do módulo

- F-08.1 a F-08.6 verificadas.
- Suíte de invariante: todo horário pertence a um pedido e todo pedido nasce
  de um atendimento; um pedido tem no máximo um horário vivo; horário
  encerrado não muda, e o que foi marcado nunca muda; cancelamento exige
  motivo; expiração e falta devolvem o pedido à fila marcado; check-in de
  horário só no dia, na unidade e com o horário confirmado; nenhum payload de
  evento carrega CPF, celular, motivo ou nota.

## Histórico
