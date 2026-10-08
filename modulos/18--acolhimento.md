# Módulo 18 — Acolhimento

- **Estado:** Fechado
- **Tipo:** Ciclo 2

## Escopo

Escuta inicial entre o check-in e a chamada do profissional: queixa em CIAP-2,
sinais vitais, classificação de risco CAB 28 sugerida por regras assinadas e
destino (ADR 0030). Primeira ficha LEDI real do Rota Saúde. Resolve a pendência
de LGPD da fila de fichas (rotasaude/api#43).

**Entregue (F-18.1 a F-18.8):**
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

## Funcionalidades

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
- Runbook do acolhimento (escopo, regras, modo integrado):
  [`operacao/rollout-acolhimento.md`](../operacao/rollout-acolhimento.md).

## Histórico

- 2026-10-05 — Registrado como `Stub` no portfólio do Ciclo 2.
- 2026-10-07 — Escopo decidido com o usuário; ADR 0030 e spec
  `superpowers/specs/2026-10-07-module-18-screening-design.md`. F-18.1 a
  F-18.8 criados (board #1, `Not Started`). Módulo passa a `Planejado`.
- 2026-10-08 — Implementado e publicado em `main`:
  - contracts `0c6c753` (tag `protocols-v1.6.0`; a raiz escolhe a variante por
    `if`/`then`/`else` sobre `kind`, não por `oneOf`; os documentos válidos são
    os mesmos);
  - api `eb732e8` (28 commits sobre `8cd8454`; suíte 4125/0 em `a25b216`;
    semente CIAP-2 de dev em `139c846` e `eb732e8`);
  - dashboard `4bf895b` (18 commits sobre `ab00e4f`; 1361 testes).

  Migrações de cidade `20261007300001` (escutas, desfechos, escopo, trava) e
  `20261007300002` (fila LEDI com códigos; **irreversível**: converte
  `last_error` em `last_error_codes` e apaga o texto). Suíte de invariante
  `spec/invariants/screening_invariants_spec.rb`; runbook
  [`operacao/rollout-acolhimento.md`](../operacao/rollout-acolhimento.md).
  rotasaude/api#43 resolvida pela F-18.7.

  **Prova no navegador aprovada** (Curitiba, UBS Jardim das Flores, 2026-10-08):
  - fila do acolhimento;
  - escuta com K86 e PA 185/110 → sugestão vermelha e IMC;
  - justificativa pedida ao mudar a cor;
  - `same_day` → topo vermelho da fila do profissional;
  - `schedule` → pedido `kind: screening` na fila do módulo 17;
  - reavaliação → amarelo;
  - escuta dentro do atendimento chamado;
  - varredor das 23h recuperou atendimentos fechados e registrou "não gerada"
    (`unit_without_cnes`, `professional_without_team`);
  - Produção e-SUS com códigos e "Gerar de novo" sob step-up.

  **Decisões (contrato §9–§10, revisão do ADR 0030):**
  - técnico (CBO 3222) sem aferição gera a Ficha de Procedimentos só com
    `statusEscutaInicialOrientacao = true`;
  - ficha perdida com a exportação inutilizável no fechamento é gerada depois
    pelo varredor das 23h, limitada à competência atual e à anterior;
  - a faixa "aguardando acolhimento" da fila do profissional só vale com
    protocolo `acolhimento` ativo; sem ele, a ordem é a do módulo 13;
  - `awaiting_screening` na fila (marcador neutro, visível à recepção); escuta
    aninhada em `attendance` no `call`/`call_next`; `simulate_screening` sem
    `alerts`/`bmi`;
  - desfechos novos com rótulo ("agendado pelo acolhimento", "orientado no
    acolhimento"), fora do formulário de encerramento;
  - ADR 0028 revisado quanto à fila de fichas (`last_error_codes`, purga de 90
    dias, exclusão limpa a fila, `replaces_outbox_id` na regeneração).

  F-18.1 a F-18.8 `Verified`; módulo `Fechado`.

  **Em aberto:**
  - rotasaude/api#48 (SIGTAP `0214010015`/`0301100039` e ficha só com a marca
    num PEC real; Alta, antes do go-live);
  - rotasaude/api#49 (Analytics não conta `scheduled_from_screening` nem
    `oriented`);
  - rotasaude/api#50 (escuta aponta para o pedido `moved` depois de esvaziar a
    unidade);
  - regras de cor do modelo e da semente precisam de revisão clínica antes de
    cidade real;
  - títulos CIAP-2 da semente vêm de memória (só dev; conferir com a tabela
    oficial);
  - critério de campanha pelo pedido de acolhimento (opcional);
  - achados visuais vão para o ciclo de interface.
