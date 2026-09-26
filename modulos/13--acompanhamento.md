# Módulo 13 — Acompanhamento

- **Estado:** Entregue
- **Tipo:** Estratégico (pós-MVP)

## Escopo

A passagem da triagem remota para o cuidado presencial (ADRs 0018 e 0019).

**Entregue:** o cidadão gera no `wpda` o código "Cheguei na unidade"; a
recepção faz o check-in pelo código e o CPF do documento (triagem de até 3
dias, validando o cadastro declarado no mesmo passo) ou, sem celular, por
exceção com motivo. O atendimento entra na fila da unidade pela prioridade da
triagem, passa por `waiting` → `in_care` → `closed`, é chamado pelo
`health_professional` e termina com um desfecho: atendido, encaminhado
(unidade e/ou descrição), retorno ou saiu sem atendimento. Retorno e
encaminhamento para unidade da cidade geram pedido de agendamento (módulo 08).
`triages`, consentimentos, relatórios e métricas não mudam por nada disso.

**Fora, por enquanto:** ficha de acompanhamento, evolução, lembretes de
medicação, follow-up automatizado e expiração de atendimentos abertos
esquecidos.

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0018 | Check-in na unidade, atendimento e desfecho a partir da triagem |
| 0019 | Chamada pelo profissional, estados do atendimento, desfecho de retorno e pedido de agendamento |

Ainda a decidir: ficha de acompanhamento, evolução e follow-up automatizado.

## Superfícies

| Superfície | O que aparece |
|---|---|
| `api` | `attendances` (só acréscimos e transições previstas), código de balcão com finalidade, rotas de check-in, fila, chamada e desfecho em `/attendance`, código de check-in em `/citizen/triages/:id/check_in_code` |
| `admin` | — |
| `dashboard` | Módulo Atendimento: check-in (código ou exceção), fila da unidade, chamada e desfecho |
| `wpda` | "Cheguei na unidade" nas triagens do cidadão |

## Dependências

- Módulo 03 (Triagem) — todo atendimento nasce de uma triagem ou de um
  horário, e a cadeia sempre chega a uma triagem.
- Módulo 06 (Identidade/Acesso) — papéis `citizen_verifier` e
  `health_professional`; validação do cadastro no check-in.
- Módulo 09 (Unidades) — onde o atendimento acontece.
- Módulo 08 (Agendamento) — recebe o pedido do desfecho de retorno ou
  encaminhamento.
- Módulo 10 (Profissionais) — só quando o profissional tiver vínculo formal
  com a unidade.

## Funcionalidades planejadas

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-13.1 | Código de check-in no `wpda` ("Cheguei na unidade"), com finalidade separada do código de validação | api, wpda | 0018 |
| F-13.2 | Check-in na recepção por código + CPF do documento (triagem de até 3 dias; valida o cadastro declarado no mesmo passo) | api, dashboard | 0018, 0017 |
| F-13.3 | Check-in por exceção: busca por CPF com motivo registrado | api, dashboard | 0018 |
| F-13.4 | Atendimento com estados `waiting` → `in_care` → `closed` e fila da unidade pela prioridade da triagem | api, dashboard | 0018, 0019 |
| F-13.5 | Chamada do cidadão pelo `health_professional` (chamar o próximo ou um específico) | api, dashboard | 0019 |
| F-13.6 | Desfecho do atendimento: atendido; encaminhado (unidade e/ou descrição); retorno; saiu sem atendimento | api, dashboard | 0018, 0019 |

## Riscos herdados

- **Clínica (ADR 0018):** o atendente passa a ver a prioridade da triagem de
  quem fez check-in, uma ampliação deliberada do que o balcão mostrava.
  Respostas e relatório continuam fora do balcão.
- **Clínica (ADR 0019):** atendimento aberto esquecido não expira; fica na
  fila até alguém encerrar.
- **LGPD (ADR 0018):** o check-in por exceção busca por CPF sem o cidadão
  apresentar o código; o motivo é obrigatório e a busca deixa trilha.
- **Em aberto:** ficha de acompanhamento, evolução, lembretes de medicação,
  follow-up automatizado e vínculo formal do profissional com a unidade
  (módulo 10).

## Critério de fechamento do módulo

- F-13.1 a F-13.6 verificadas.
- Suíte de invariante: todo atendimento nasce de uma triagem ou de um
  horário, nunca dos dois; uma triagem e um horário têm no máximo um
  atendimento cada; check-in nunca muda e atendimento encerrado não muda
  mais; desfecho clínico só a partir de `in_care`, e "saiu sem atendimento"
  só a partir de `waiting`; a exceção por CPF sempre grava método e motivo;
  `triages`, consentimentos, relatórios e métricas não mudam por check-in,
  chamada nem desfecho; o CPF nunca vai na URL; nenhum payload de evento
  carrega CPF, celular, motivo ou descrição do encaminhamento.

## Histórico
