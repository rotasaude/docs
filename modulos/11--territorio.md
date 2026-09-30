# Módulo 11 — Território

- **Estado:** Fechado
- **Tipo:** MVP

## Escopo

Território da cidade por **bairro declarado**, sem geometria (ADR 0023): a
lista de bairros de cada cidade, quais unidades atendem cada bairro, o bairro
do cidadão e o endereço da unidade. Serve a dois usos no Ciclo 1: indicar a
unidade de referência do cidadão e recortar os painéis por bairro. É também a
base do recorte territorial dos módulos 12 (Campanhas) e 14 (Analytics).

**Entregue (F-11.1 a F-11.7):**
- bairros por cidade: semente própria versionada (casada por `seed_key`) como
  carga inicial; depois o `municipal_admin` cria, renomeia, desativa e
  reativa no dashboard;
- cobertura: quais unidades ativas atendem cada bairro;
- endereço da unidade em texto, com consulta de CEP feita pelo navegador do
  dashboard no ViaCEP (falha = preenchimento à mão);
- bairro declarado pelo cidadão uma vez no wpda, trocável, copiado na triagem
  (imutável; vira nulo só na anonimização por revogação);
- unidade de referência no resultado da triagem, só na área logada do cidadão
  (nunca no relatório público);
- unidade de referência pré-selecionada no desfecho "encaminhado" (nunca a
  própria unidade do atendimento);
- filtro de bairro nos 5 painéis com cidadão, para todos os papéis que os
  leem, com supressão de contagens de 1 a 4.

**Fora, por enquanto:** região acima do bairro, polígonos, distância e
"unidade mais próxima" (PostGIS), importação de bairros por CSV, restrição do
agendamento à unidade de referência, horário de funcionamento da unidade.

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0023 | Bairro declarado sem geometria, semente + edição, cópia na triagem, referência que informa e sugere, filtro com supressão, CEP pelo navegador |
| 0018 | Unidades: o endereço deixado em aberto lá entra por este módulo |
| 0019 | Desfecho "encaminhado" e pedido de agendamento, onde a referência é sugerida |

## Superfícies

| Superfície | O que aparece |
|---|---|
| `api` | `neighborhoods`, `neighborhood_coverages`; bairro em `citizens` e `triages`; endereço em `health_units`; rotas `/territory`; `/citizen/neighborhoods` e bairro da pessoa; `reference_units` e `reference_unit_ids`; `GET /admin/api/neighborhoods` e filtro nos painéis; semente e rake `city:territory:seed` |
| `admin` | — (o console não envia o filtro) |
| `dashboard` | Território (bairros e cobertura), endereço com CEP no formulário de unidade, pré-seleção no desfecho, seletor de bairro nos painéis |
| `wpda` | Bairro na escolha da pessoa, "Trocar bairro", bloco "Sua unidade de referência" |

## Pré-requisitos

- Módulo 09 (Unidades) — cobertura e endereço são da unidade.
- Módulo 13 (Acompanhamento) — desfecho "encaminhado".
- Módulo 05 (Dashboard) — painéis ao vivo (ADR 0022).

## Funcionalidades planejadas

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-11.1 | Bairros da cidade: semente própria e edição pelo `municipal_admin` | api, dashboard | 0023 |
| F-11.2 | Cobertura: quais unidades atendem cada bairro | api, dashboard | 0023 |
| F-11.3 | Endereço da unidade, com consulta de CEP pelo navegador | api, dashboard | 0023, 0018 |
| F-11.4 | Bairro declarado pelo cidadão, copiado de forma imutável na triagem | api, wpda | 0023 |
| F-11.5 | Unidade de referência no resultado da triagem (área logada) | api, wpda | 0023 |
| F-11.6 | Unidade de referência pré-selecionada no desfecho "encaminhado" | api, dashboard | 0023, 0019 |
| F-11.7 | Filtro de bairro nos painéis, com supressão de contagens de 1 a 4 | api, dashboard | 0023, 0022 |

## Riscos herdados

- **Semente:** Maringá começa com 44 bairros reais conferidos na base de CEP,
  não a lista oficial completa; a prefeitura completa pelo dashboard.
- **ViaCEP** fora do ar ou alterado: o preenchimento à mão é sempre possível.
- **Supressão** não impede todo cruzamento (alternar filtro e período); aceito
  no Ciclo 1, com painel restrito a papéis da própria cidade.

## Critério de fechamento do módulo

- F-11.1 a F-11.7 verificadas.
- Suíte de invariante (`spec/invariants/territory_invariants_spec.rb`, com
  teste de mutação): bairro da triagem imutável (exceto NULL na revogação);
  bairro inativo fora de escolhas novas; referência sem unidade inativa e sem
  a própria unidade no desfecho; nenhum número de 1 a 4 com filtro ligado;
  semente idempotente e sem desfazer edições; relatório público sem bairro nem
  referência; `api` sem chamada ao ViaCEP.
- Runbook [`operacao/rollout-territorio.md`](../operacao/rollout-territorio.md).

## Histórico

- 2026-09-28 — Escopo decidido com o usuário; ADR 0023, spec
  `superpowers/specs/2026-09-28-module-11-territory-design.md` e planos
  `superpowers/plans/2026-09-28-module-11-territory-{api,dashboard,wpda}.md`.
  F-11.1 a F-11.7 criados. Módulo passa de `Stub` a `Planejado`.
- 2026-09-29 — F-11.1 a F-11.7 entregues e publicados: api `2995f95` (suíte
  2580/0; invariantes com mutação), dashboard `cbbb75b` (531 testes), wpda
  `d1b552f` (165 testes). Prova no navegador feita com o usuário (Território,
  CEP, filtro com "< 5" e "oculto", pré-seleção no "encaminhado" nunca na
  própria unidade, bairro e unidade de referência no wpda, relatório público
  sem unidade e com "Voltar ao início"). Decisões da execução no ADR 0023,
  seção Revisão. Módulo passa a `Entregue`; falta a verificação (dossiê por
  F-ID).
- 2026-09-29 — Verificação do módulo (dossiê por F-ID,
  [`relatorios/2026-09-29-verificacao-modulo-11.md`](../relatorios/2026-09-29-verificacao-modulo-11.md)):
  duas lacunas de teste fechadas antes de subir (rollback da migração e trava
  da cobertura sob concorrência; api `d03fa02`, suíte 2583/0), dica do filtro
  corrigida (dashboard `e810afb`). F-11.1 a F-11.7 `Verified` por aprovação do
  usuário. Critério de fechamento cumprido; módulo `Fechado`.
