# Módulo 09 — Unidades

- **Estado:** Fechado
- **Tipo:** MVP

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
- **Operacional:** uma unidade desativada some da escolha do balcão. A
  desativação é recusada enquanto houver atendimento aberto ou pedido de
  agendamento aberto ou marcado para ela (`unit_has_open_attendances`,
  `unit_has_open_requests`); a recepção precisa encerrar esses antes.
- **Em aberto:** endereço, horário de funcionamento, especialidades,
  capacidade e geolocalização (módulo 11).

## Critério de fechamento do módulo

- F-09.1 e F-09.2 verificadas.
- Suíte de invariante: só o `municipal_admin` cria, edita, desativa e
  reativa unidade; desativação com atendimento aberto é recusada; unidade
  inativa não recebe check-in.

## Histórico

- 2026-09-26 — Verificação do módulo (dossiê por F-ID): F-09.1 e F-09.2
  `Verified`. Fechamento no api: a desativação trava a unidade (FOR UPDATE)
  e o check-in (por código e por exceção) e o encaminhamento a travam com
  FOR SHARE, o que fecha a corrida em que um check-in entrava numa unidade
  sendo desativada. Suíte de invariante completa: escritas recusadas para
  `citizen_verifier`, `health_professional` e `viewer`; unidade inativa
  recusada nos dois check-ins; spec de concorrência com threads reais.
  Módulo `Fechado` em 2026-09-27 (api dd24533; suíte 2218/0; invariantes em
  `spec/invariants/health_unit_invariants_spec.rb`, com teste de mutação).
- 2026-09-29 — Endereço da unidade (logradouro, número, complemento, CEP e
  bairro onde fica) entregue pelo módulo 11 (F-11.3, ADR 0023; api `b5e9ba3`),
  sem refazer a tabela; `lock_active!` e a trava da desativação inalterados.
  Resolve em parte o risco do cadastro mínimo (rotasaude/api#28).
