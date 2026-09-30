# Dossiê de verificação — Módulo 13 (Acompanhamento), F-13.1 a F-13.6

**Data:** 2026-09-30
**ADRs governantes:** `docs/adr/0018.md` e `docs/adr/0019.md` (ambos com a seção Revisão de 2026-09-30); `docs/adr/0021.md` (regra da chamada) e `docs/adr/0017.md` (código do balcão, validação)
**Spec:** `docs/superpowers/specs/2026-09-24-citizen-attendance-check-in-design.md`
**Módulo:** `docs/modulos/13--acompanhamento.md` · **Runbook:** `docs/operacao/atendimento.md`
**Código verificado (origin/main antes das correções):** api `476084d` · dashboard `c69a932` · wpda `32e6420`
**Branch das correções:** `fix/mod-13-verification` (api, dashboard, wpda) · `docs/mod-13-verification` (docs)

## Como foi feito

Três agentes montaram o dossiê por F-ID (F-13.1–.3 check-in; F-13.4–.5 fila e chamada; F-13.6 e critério de fechamento), lendo código, ADRs e specs. Linha de base: as specs do módulo passavam (168/0 na primeira corrida; 172 na contagem do implementador, com o mesmo conjunto). Todas as F-IDs estavam implementadas e aderentes aos ADRs; as lacunas eram de teste, mais o critério de fechamento, que **não** era cumprido (não havia suíte de invariante agrupada; a varredura de payload excluía `attendance.called` e `attendance.closed`; "CPF nunca na URL" não tinha teste nas rotas de check-in; não havia runbook).

## Decisão do usuário (2026-09-30)

Fechar todas as lacunas antes de subir, e ainda:

- **Trilha LGPD da busca por exceção:** `attendance.exception_searched` ganha `citizen_ids` e `health_unit_id` (ids, nunca CPF).
- **Mensagens de corrida:** check-in duplicado em corrida devolve `already_checked_in` com unidade e hora, pelo código e pela exceção; outras violações de unicidade não são rotuladas assim.
- **"Chamar próximo" sob disputa:** `FOR UPDATE SKIP LOCKED`; o dashboard recarrega a fila em `already_called`.
- **Nasce `waiting` no banco** e **desempate estável** na fila (prioridade, chegada, id).
- **Encaminhar para a própria unidade** continua permitido.
- **Runbook** do módulo: `operacao/atendimento.md`.

## Correções (TDD; cada teste visto vermelho por mutação ou antes da correção)

api (`fix/mod-13-verification`):

- `de84e15` — trigger `attendances_born_waiting` (migração de cidade `20260930100001_guard_attendance_insert`, com spec de `down`/`up`); `lib/campaign_history.rb` passa a criar o atendimento `waiting` e percorrer as transições reais.
- `4ec59af` — `exception_searched` com `citizen_ids` e `health_unit_id` (só quando é unidade existente: um CPF digitado no campo da unidade não vaza para o evento).
- `1b36463` — corrida de check-in rotulada pelo índice violado (`PG_DIAG_CONSTRAINT_NAME`); `already_checked_in` com `unit_name` e `checked_in_at` nos dois caminhos; validação concorrente do mesmo cidadão num savepoint (o check-in segue, `verified: false`).
- `bb509d4` — fila ordenada no SQL (prioridade da triagem raiz `NULLS LAST`, `checked_in_at`, `id`), compartilhada por `UnitQueue.waiting` e `CallNext`; `CallNext` com `FOR UPDATE OF attendances SKIP LOCKED`, provado com threads.
- `d566146` — suíte `spec/invariants/attendance_invariants_spec.rb` (itens a–i abaixo).
- `5f54549`, `a66a12a`, `4f55bc0` — testes que faltavam por F-ID.
- `92a34e3` — ajustes da revisão (comentários da fila e do `campaign_history`; limpeza robusta da spec com threads).

dashboard: `e60f39d` (recarrega a fila em `already_called` do "Chamar próximo"), `59903d5` (mensagens de triagem e corrida na exceção), `b1e0f37` (comentário da revisão).
wpda: `4bbad51` (mensagens de recusa do "Cheguei na unidade"), `4d07728` (status reais da API nos mocks).
docs: Revisão nos ADRs 0018 e 0019; runbook `operacao/atendimento.md` (listado no README); spec do check-in atualizada (payload da busca); este dossiê.

Revisão independente (opus): **aprovar com correções**, nada crítico. Confirmou o SQL do `SKIP LOCKED` (joins por PK, sem duplicar nem perder linha; `FOR UPDATE OF` válido; sem deadlock com o `FOR SHARE` do vínculo), o savepoint (`requires_new`), o payload sem CPF e o trigger em cidade nova (`city_triggers.sql` + `city_schema_spec`). Achados menores tratados: comentários, limpeza da spec com threads, status dos mocks do wpda, spec de docs. Aceitos e documentados: "fila vazia" momentânea quando todos os aguardando estão travados por outra chamada (runbook); violação de unicidade não relacionada ao atendimento agora sobe como 500 em vez de virar mensagem de negócio (decisão 2).

**Suíte completa do api no branch: 2826/0** (worker parado). dashboard 670/670 + typecheck limpo; wpda 252/252 + typecheck limpo.

## Resumo

| F-ID | Veredito | Lacunas (fechadas) |
|---|---|---|
| F-13.1 | **Verified** | triagem inelegível; TTL 10 min, 5 tentativas, CPF errado e código vencido com finalidade `check_in`; mensagens do wpda |
| F-13.2 | **Verified** | janela de 72 h exatas; limite 30/10 min (a busca conta); 403 do `health_professional`; corrida → `already_checked_in` |
| F-13.3 | **Verified** | exceção com triagem fora da janela e já atendida; CHECK de método e motivo; trilha com `citizen_ids`/`health_unit_id`; corrida → `already_checked_in` |
| F-13.4 | **Verified** | fila mista (horário pela triagem raiz); isolamento por unidade; sem prioridade vai para o fim; desempate por id; nasce `waiting` no banco |
| F-13.5 | **Verified** | chamar atendimento encerrado; concorrência real com threads; `SKIP LOCKED`; payload de `attendance.called` |
| F-13.6 | **Verified** | payload de `attendance.closed` sem a descrição; `referral_unit_id` inválido na API |
| Critério de fechamento | **cumprido** | suíte de invariante com os 8 itens do módulo + "nasce `waiting`", cada um provado por mutação; runbook existe |

## Critério de fechamento — `spec/invariants/attendance_invariants_spec.rb`

| Item | Onde | Mutação que deixa vermelho |
|---|---|---|
| a. nasce de triagem ou de horário, nunca dos dois nem de nenhum | :56, :63 | sem `ck_attendances_origin` |
| b. no máximo um atendimento por triagem e por horário | :71, :78 | sem cada índice único |
| c. as 10 colunas do check-in imutáveis (waiting e in_care); chamada imutável; encerrado imutável nos 4 desfechos; sem DELETE | :105, :117, :131 | lista de colunas reduzida; sem as checagens do trigger |
| d. desfecho clínico só de `in_care`; `left` só de `waiting` | :156, :168, :186, :200 | sem as checagens de transição |
| e. exceção por CPF sempre grava método e motivo | :208, :216, :225, :242 | sem os CHECKs de motivo e método |
| f. triagens, consentimentos, relatórios e métricas intactos (inclui `left`) | :263; `spec/requests/attendance_contract_spec.rb:85` | desfecho reescrevendo triagem; check-in escrevendo consentimento |
| g. CPF nunca na URL | :414, :424 | rota GET para a busca |
| h. nenhum payload com CPF, celular, motivo ou descrição do encaminhamento (varre `attendance%`, `appointment%`, `citizen.verified`) | :308 | CPF, motivo e nota injetados no payload |
| i. nasce `waiting` | :366–390 | vermelho antes do trigger |

## Riscos que continuam

- Atendimento esquecido não expira (ADR 0019); procedimento no runbook.
- O atendente vê a prioridade da triagem (ampliação deliberada do ADR 0018).
- Encaminhar para a própria unidade é permitido (decisão de 2026-09-30).
- Em aberto no módulo: ficha de acompanhamento, evolução, lembretes, follow-up automatizado.

## Rollout

Migração de cidade só de expansão (trigger de INSERT). Publicar a imagem do api e rodar `bin/rails city:migrate:all` da imagem nova antes de cortar o tráfego; dashboard e wpda depois. Os bancos de teste de cidade foram recarregados com o trigger: até este branch entrar no main, specs de campanha do main quebram contra eles (o `lib/campaign_history.rb` antigo insere atendimento encerrado direto).
