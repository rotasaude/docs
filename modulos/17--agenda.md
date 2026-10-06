# Módulo 17 — Agenda dos profissionais

- **Estado:** Fechado
- **Tipo:** Ciclo 2

## Escopo

A unidade passa a ter agenda por profissional, recortada dos turnos, e a
triagem pode gerar pedido de agendamento com tipo, prioridade e prazo (ADR
0029). Quem marca é sempre a recepção; o cidadão confirma, cancela, pede outro
horário e recebe lembrete.

**Entregue (F-17.1 a F-17.9):**
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

## Funcionalidades

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
- 2026-10-06 — Implementado e publicado em `main`:
  - contracts `8298709` (tag `protocols-v1.5.0`);
  - api `8cd8454` (36 commits sobre o módulo 16; suíte 3890/0);
  - dashboard `ab00e4f` (1217 testes), wpda `20e5e85` (460 testes).

  Migração de cidade `20261006210001` (renumerada para ficar depois da
  `20261006200001` do módulo 16: `city:dev_up` marca como aplicada, sem rodar,
  toda versão menor que a maior registrada). Suíte de invariante
  `spec/invariants/scheduling_invariants_spec.rb`; runbook
  [`operacao/agenda-recepcao.md`](../operacao/agenda-recepcao.md). Os bancos de
  teste de cidade aceitam sufixo (`ROTA_TEST_DB_SUFFIX`) para sessões paralelas.

  **Prova no navegador aprovada** (Curitiba, dev): modelo "Manhã" e
  pré-visualização; triagem "Saúde do idoso" gerando pedido na fila "sem
  unidade"; atribuição de unidade; marcação em vaga; dois encaixes e o terceiro
  recusado (`409 fit_in_limit`, também direto na API); "Não posso nesse horário"
  devolvendo o pedido à fila com o mesmo prazo; lembrete da véspera na caixa de
  avisos; resultado da triagem com `status: "scheduled"`.

  **Decisões (contrato §8–§11):** fusão com pedido marcado que encurta o prazo
  vira `needs_reschedule`; exclusão cancela horários e fecha pedidos vivos
  (conta como cancelamento do cidadão no analytics); ordem de travas unidade →
  cidadão → horários → pedido → turno; prazo vencido escondido em "Seus
  agendamentos".

  F-17.1 a F-17.9 `Verified`; módulo `Fechado`.

  **Em aberto:**
  - rotasaude/api#45 (remarcação pedida em unidade desativada);
  - rotasaude/api#46 (lembrete continua na caixa após cancelamento);
  - rotasaude/api#47 (exclusão não limpa `cancel_reason`/`dismiss_reason`, anterior ao 17);
  - provedor de SMS com timeout curto antes do go-live (o envio acontece com o horário travado);
  - remover a chave antiga `appointments` da agenda da unidade depois do deploy do dashboard;
  - pedido `needs_reschedule` em unidade sem turno não tem caminho de marcação livre;
  - endereço da unidade no wpda sem o bairro (ciclo de interface).
