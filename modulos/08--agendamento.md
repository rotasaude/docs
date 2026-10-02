# Módulo 08 — Agendamento

- **Estado:** Em andamento
- **Tipo:** MVP

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

Um SMS lembra o cidadão 24h antes do prazo de confirmação, e o `wpda` mostra
na entrada os horários a confirmar (F-08.7).

**Fora, por enquanto:** agenda de vagas publicada pela unidade, o cidadão
escolhendo o horário, lembrete do horário já confirmado, remarcação pedida
pelo cidadão e encaminhamento para fora da rede da cidade.

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0019 | Pedido de agendamento e horário: nascem do desfecho de retorno ou encaminhamento; confirmação com prazo, expiração e falta |

Ainda a decidir: agenda de vagas publicada pela unidade e o cidadão escolhendo o horário (em aberto no ADR 0019).

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
| F-08.7 | Lembrete por SMS 24h antes do prazo de confirmação do horário (opt-out do cidadão) e aviso no wpda dos horários a confirmar | api, wpda | 0019 |

## Riscos herdados

- **Clínica (ADR 0019):** o cancelamento automático pune quem não abre o
  `wpda`. Desde 2026-10-02 há lembrete por SMS (F-08.7), mas ele só sai com
  o provedor contratado e a chave de SMS da cidade ligada; até lá, a marca
  "sem confirmação" na fila segue como rede de proteção (a recepção liga e
  remarca). Ligar o SMS antes do prazo ainda alcança os horários pendentes.
- **Operacional (ADR 0019):** a recepção digita data e hora sem agenda de
  vagas. Desde 2026-10-01 o horário já ocupado na unidade é avisado e só sai
  como encaixe confirmado (api#26); sem profissional nem duração no horário,
  o aviso olha só o início igual.
- **Operacional:** desde 2026-10-02 prazo, falta, "hoje" e a agenda seguem o
  fuso da cidade (api#27; Revisão do ADR 0020). O fuso é definido no
  provisionamento e o banco recusa trocá-lo (api#36).
- **Operacional:** expiração e falta dependem dos jobs do worker da cidade.
  Worker parado deixa horário vencido aberto e pedido fora da fila.
- **Em aberto:** agenda de vagas, escolha do horário pelo cidadão, lembrete do
  horário confirmado, remarcação pedida pelo cidadão e encaminhamento para
  fora da rede.

## Critério de fechamento do módulo

- F-08.1 a F-08.7 verificadas.
- Suíte de invariante: todo horário pertence a um pedido e todo pedido nasce
  de um atendimento; um pedido tem no máximo um horário vivo; horário
  encerrado não muda, e o que foi marcado nunca muda; cancelamento exige
  motivo; expiração e falta devolvem o pedido à fila marcado; check-in de
  horário só no dia, na unidade e com o horário confirmado; nenhum payload de
  evento carrega CPF, celular, motivo ou nota.

## Histórico

- 2026-09-27: módulo fechado, com 6/6 `Verified`. A verificação por F-ID
  achou lacunas reais, consertadas com TDD (api 71a4711, dashboard 72ed9ad,
  wpda bb42097; api 2200/0, dashboard 342, wpda 95):
  - F-08.6: o check-in por exceção respondia `triage_not_eligible` para
    qualquer horário recusado. Agora diz `not_today`, `wrong_unit` (com o nome
    da unidade) ou `appointment_not_eligible`, e o `fulfil` reconfere dia e
    unidade sob o lock, não só o status (api#24).
  - F-08.2: o dashboard lia a data e hora digitadas no fuso do navegador.
    Agora lê como hora da cidade (dashboard#8).
  - F-08.4: o `wpda` oferecia Confirmar depois do prazo, até o job rodar, e
    o servidor recusava. Agora o botão some e a tela avisa. Saiu um texto de
    pedido reaberto que nunca aparecia.
  - Critério cumprido: `spec/invariants/appointment_invariants_spec.rb`
    (api#25) cobre as recusas no banco, o escopo do check-in e uma varredura
    do ciclo inteiro provando que nenhum payload de evento carrega CPF,
    celular, motivo ou nota (conferida injetando a nota num payload).
  - Continuam como riscos, não como pendência dos F-IDs: horário duplicado
    (api#26) e fuso fixo (api#27).
- 2026-10-01: conflito de horário (api#26). A marcação recusa com 409
  `slot_taken` quando a unidade já tem horário vivo no mesmo início e diz
  quantos; a recepção vê o aviso e pode "Marcar mesmo assim", e o evento
  registra o encaixe (`fit_in`). Trava por unidade e início contra duas
  recepções ao mesmo tempo. Decisão na Revisão do ADR 0019.
- 2026-10-02: fuso da cidade (api#27). O dia, o prazo de confirmação, a
  falta depois da meia-noite, o check-in "só hoje" e a agenda seguem o fuso
  da cidade; a recepção digita e vê a hora nele, e o cidadão também. Suíte de
  invariante em `America/Manaus` (`spec/invariants/city_time_zone_invariants_spec.rb`).
- 2026-10-02: lembrete de confirmação (api#39, F-08.7). Um SMS 24h antes do
  prazo, só para horário ainda sem confirmação, dentro das 8h–20h da cidade;
  não exige o opt-in das campanhas, mas respeita o opt-out de lembretes;
  texto fixo sem unidade, data nem motivo. Sem chave de SMS, sem provedor ou
  com opt-out, nada sai e o lembrete segue pendente até o prazo; só a resposta
  do provedor o grava (`appointment_reminders`, só acréscimo, um por horário). No `wpda`, faixa com os horários a confirmar e
  interruptor de lembrete nas preferências. Módulo volta a `Em andamento`
  até a verificação da F-08.7.
