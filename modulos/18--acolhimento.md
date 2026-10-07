# Módulo 18 — Acolhimento

- **Estado:** Planejado
- **Tipo:** Ciclo 2

## Escopo

Escuta inicial entre o check-in e a chamada do profissional: queixa em CIAP-2,
sinais vitais, classificação de risco CAB 28 sugerida por regras assinadas e
destino (ADR 0030). Primeira ficha LEDI real do Rota Saúde. Resolve a pendência
de LGPD da fila de fichas (rotasaude/api#43).

**Planejado (F-18.1 a F-18.8):**
- escuta inicial com queixa CIAP-2, sinais vitais e escopo por unidade
  (`walk_in` padrão, `all`);
- classificação CAB 28 com sugestão por protocolo assinado `screening`;
- destino (consulta no dia, agendar, orientação, encaminhar) e exceção na trava
  do módulo 13;
- filas do acolhimento e do profissional por cor; reavaliação;
- fichas LEDI por CBO, com confirmação prévia do layout;
- fichas não geradas por falta de identificação, com "gerar de novo";
- LGPD da fila de fichas: códigos no lugar de texto, retenção de 90 dias,
  exclusão;
- trilha de leitura da escuta.

**Fora, por enquanto:** escuta domiciliar (24), escuta visível ao cidadão (20),
painel de espera por cor, CIAP-2 sugerido do texto.

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0030 | Escuta, classificação, destino, ficha por CBO, LGPD da fila, trilha de leitura |
| 0018, 0019 | Check-in, atendimento, desfecho, fila |
| 0028 | Exportação LEDI e terminologias (revisado pelo 0030) |
| 0029 | Pedido de agendamento |
| 0026 | Revogação e exclusão |

## Superfícies

| Superfície | O que aparece |
|---|---|
| `contracts` | `protocols-v1.6.0` (variante `screening`) |
| `api` | `screenings`, `screening_revisions`, `screening_scope`, desfechos novos, `Screenings::*`, fichas da escuta, `ledi_generation_failures`, `last_error_codes`, purga de 90 dias |
| `admin` | — |
| `dashboard` | Acolhimento, fila com cor, escopo da unidade, protocolo `screening`, fichas não geradas |
| `wpda` | — |

## Pré-requisitos

- Módulos 13 (atendimento), 15 (perfil e construtor), 16 (exportação,
  terminologias, equipes), 17 (pedido de agendamento).

## Funcionalidades planejadas

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-18.1 | Escuta inicial: queixa em CIAP-2, sinais vitais e escopo por unidade | api, dashboard | 0030 |
| F-18.2 | Classificação de risco CAB 28 com sugestão por protocolo assinado de acolhimento | contracts, api, dashboard | 0030, 0016 |
| F-18.3 | Destino do acolhimento e exceção na trava do módulo 13 | api, dashboard | 0030, 0019 |
| F-18.4 | Filas do acolhimento e do profissional por cor; reavaliação | api, dashboard | 0030 |
| F-18.5 | Fichas LEDI da escuta por CBO, com confirmação prévia do layout | api | 0030, 0028 |
| F-18.6 | Fichas não geradas por falta de identificação, com motivo e "gerar de novo" | api, dashboard | 0030 |
| F-18.7 | LGPD na fila de fichas (api#43): códigos, retenção de 90 dias, exclusão | api | 0030, 0026, 0028 |
| F-18.8 | Trilha de leitura da escuta | api | 0030 |

## Riscos herdados

- Confirmação do layout pode mudar o mapeamento CBO → ficha.
- Aceite real do PEC ainda não observado (api#41).
- Dupla contagem no modo integrado.
- Regras iniciais de cor precisam de revisão clínica antes de valer.

## Critério de fechamento do módulo

- F-18.1 a F-18.8 verificadas.
- Suíte de invariante (`spec/invariants/screening_invariants_spec.rb`).
- api#43 fechada.
- Runbook do acolhimento (escopo, regras, modo integrado).

## Histórico

- 2026-10-05 — Registrado como `Stub` no portfólio do Ciclo 2.
- 2026-10-07 — Escopo decidido com o usuário; ADR 0030 e spec
  `superpowers/specs/2026-10-07-module-18-screening-design.md`. F-18.1 a
  F-18.8 criados (board #1, `Not Started`). Módulo passa a `Planejado`.
