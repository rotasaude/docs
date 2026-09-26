# Módulo 09 — Unidades

- **Estado:** Entregue
- **Tipo:** Estratégico (pós-MVP)

## Escopo

Cadastro das unidades de saúde da cidade, apoio do check-in, do atendimento e
do agendamento (ADR 0018).

**Entregue:** cadastro mínimo (`health_units`: nome, tipo, ativa), mantido
pelo `municipal_admin`; desativar e reativar, com a desativação recusada
enquanto houver atendimento aberto na unidade.

**Fora, por enquanto:** endereço, horário de funcionamento, especialidades,
capacidade e geolocalização. Entram sem refazer a tabela.

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0018 | Cadastro mínimo de unidades (`health_units`: nome, tipo, ativa) como apoio do check-in e do atendimento |

Ainda a decidir: endereço, horário, especialidades e geolocalização (sem refazer a tabela).

## Superfícies

| Superfície | O que aparece |
|---|---|
| `api` | `health_units`, rotas de unidades em `/attendance/units` |
| `admin` | — (cadastro é da cidade) |
| `dashboard` | Cadastro de unidades e escolha da unidade do balcão no módulo Atendimento |
| `wpda` | — (a unidade aparece no horário do cidadão) |

## Dependências

- Módulo 11 (Território) — só quando endereço e geolocalização entrarem.

## Funcionalidades planejadas

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-09.1 | Cadastro mínimo de unidades de saúde (`health_units`: nome, tipo, ativa) mantido pelo `municipal_admin` | api, dashboard | 0018 |
| F-09.2 | Desativar e reativar unidade (desativação recusada com atendimentos abertos) | api, dashboard | 0018 |

## Riscos herdados

- **Operacional (ADR 0018):** o cadastro é mínimo. Sem endereço, horário e
  especialidades, o encaminhamento e a marcação dependem de a recepção saber
  para onde mandar o cidadão.
- **Operacional:** uma unidade desativada some da escolha do balcão; a
  desativação é recusada enquanto houver atendimento aberto, mas pedidos e
  horários futuros para ela precisam de decisão da recepção.
- **Em aberto:** endereço, horário de funcionamento, especialidades,
  capacidade e geolocalização (módulo 11).

## Critério de fechamento do módulo

- F-09.1 e F-09.2 verificadas.
- Suíte de invariante: só o `municipal_admin` cria, edita, desativa e
  reativa unidade; desativação com atendimento aberto é recusada; unidade
  inativa não recebe check-in.

## Histórico
