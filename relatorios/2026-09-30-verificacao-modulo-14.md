# Dossiê de verificação — Módulo 14 (Analytics), F-14.1 a F-14.9

- **Data:** 2026-09-30
- **Base verificada:** api `ecadd09`, dashboard `c4fd035`, admin `6436f23`,
  maintenance `baf3219`, contracts `3747c80` (`protocols-v1.2.0`)
- **Autoridade:** ADR 0025, spec
  `superpowers/specs/2026-09-30-module-14-analytics-design.md`, contratos
  `superpowers/plans/2026-09-30-module-14-analytics-contracts.md`

## Como foi feito

Três agentes de leitura, um por grupo de F-IDs (14.1/14.6/14.7; 14.2–14.5;
14.8/14.9 + critério de fechamento), com veredito por F-ID (READY, GAP-TEST,
GAP-IMPL) e evidência em quatro camadas: implementação, especificação
comportamental, invariante e regressão. Conferência manual da prova de mutação
nos relatórios da execução (ledger SDD do api, Task 19 e leva de correção
final). Prova no navegador feita antes, na entrega (ver histórico do módulo).

## Resumo

| F-ID | Veredito | Lacunas principais |
|---|---|---|
| F-14.1 Consolidação | GAP-TEST | liberação da trava depois de falha; rake `:all` e caminho de sucesso; rebuild não afirma a republicação; purga de runs na borda de 90 dias |
| F-14.2 Papel e área | GAP-TEST | varredura recursiva 1–4 fraca fora de `triage.started`; 422 via HTTP (`protocol_version` sem nome, unidade inválida na qualidade, 61 meses, parâmetro em array); membership inativa → 403; borda de 36 h |
| F-14.3 Demanda | GAP-TEST | pedidos abertos/fechados nunca afirmados ocultos; granularidade mensal com célula oculta |
| F-14.4 Qualidade | GAP-TEST | estado vazio (sem consolidação) sem spec de request; taxas por unidade fora da suíte de invariante |
| F-14.5 Calibração | GAP-TEST + 1 ajuste | sem guarda de "nunca consolidado" (devolve dado com `as_of` nulo, ao contrário das outras frentes); tier nulo sem teste |
| F-14.6 Perguntas `analytic` | GAP-TEST | nada fixa a cópia do schema no api igual à do contracts (idênticas hoje, conferido à mão); recusa 422 de `analytic` em integer só testada no `Gate`, não via request |
| F-14.7 Epidemiologia | GAP-TEST | borda de fuso do `epi.answer`; opção removida numa versão nova some da série sem teste |
| F-14.8 Publicação + console | READY | atomicidade do apaga-e-regrava na plataforma sem spec; taxa oculta pelo denominador só no `Suppression`; **semana corrente incompleta é publicada** (decisão) |
| F-14.9 Maintenance | READY | `lastError` sem DETAIL provado no `Run`, não pelo campo GraphQL; `stale: true` não afirmado via GraphQL |

Nenhum GAP-IMPL.

## Critério de fechamento — `spec/invariants/analytics_invariants_spec.rb`

Todas as invariantes do ADR 0025 têm seção própria e mutação registrada. A
execução das mutações está nos relatórios da Task 19 e da leva de correção
final (ledger SDD do api):

| Invariante (ADR 0025) | Seção | Mutação que ficou vermelha |
|---|---|---|
| Sem coluna de pessoa nas 3 tabelas | `:37` | `citizen_id` em `analytics_daily_facts` |
| Nenhum valor de 1 a 4 sai do banco da cidade | `:48` | `cell` sem `wrap`; `Publish` sem `cell` |
| Total/taxa oculto quando parte oculta | `:90` | `group` → `cell`; `row` sem grupo; `group_rate` → `rate`; `triages_total` → `cell` |
| Epidemiologia só de pergunta marcada boolean/enum | `:189` | sem filtro `analytic`; `answer_type` → `TRUE OR` |
| Consolidação idempotente | `:204` | sem `delete_all` da janela |
| Revogação não altera fato fora da janela | `:221` | `delete_all` sem janela |
| Purga só acima de 5 anos | `:244` | `...` → `..`; retenção de 1 ano |
| Só `analyst`/`municipal_admin`; operador nunca | `:257` | papel extra em `ROLES`; grant sem recusa |
| Plataforma nunca abre banco de cidade para indicador | `:284` | `CityConnection.with` no console e no GraphQL |

- Runbook [`operacao/analytics.md`](../operacao/analytics.md): ordem de rollout,
  `city:migrate:all` antes do tráfego, rebuild depois do deploy, monitoramento
  por `analyticsStatus`, retenção, revogação e rollback.
- CI: `bundle exec rspec` sem filtro roda `spec/invariants/`.

## Divergências de texto

- Contratos §0 e spec §6.3 dizem que qualquer célula de `triages.*` esconde
  os três `triages_total`; o código (correto, `ecadd09`) usa as partes de cada
  um. O texto precisa ser corrigido.
- Spec §4.1 diz 2h; o job roda às 2h30 (depois do `sweep_abandoned`). A tela
  do admin diz "2h".

## Decisões pendentes

1. Semana corrente incompleta publicada na plataforma.
2. Quais lacunas fechar antes de subir para `Verified`.

## Riscos que continuam

- Cruzamento entre tabelas diferentes da mesma resposta (ADR 0025, em aberto).
- No semanal, cidade pequena vê quase todo total oculto (consequência da regra
  do total do grupo).
- Varredura completa de `triages` no job noturno (sem índice de
  `created_at`/`completed_at`).
- Fuso fixo `America/Sao_Paulo` (api#27).
