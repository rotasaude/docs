# Dossiê de verificação — Módulo 07 (LGPD/Auditoria), F-07.15 e F-07.16 (ADR 0026)

**Data:** 2026-10-01
**ADR governante:** `docs/adr/0026.md`; também `0008` (consentimento), `0014` (trilha), `0025` (Analytics)
**Spec:** `docs/superpowers/specs/2026-10-01-lgpd-revocation-and-erasure-design.md`
**Módulo:** `docs/modulos/07--lgpd-auditoria.md` · **Runbook:** `docs/operacao/atender-requisicao-lgpd.md`
**Código verificado (origin/main antes das correções):** api `eafe586` · dashboard `723ccf1`
**Branch das correções:** api `mod-7/verify-adr-0026`

## Como foi feito

- Um agente montou o mapa de evidência por F-ID (código com arquivo:linha, spec que prova cada invariante do ADR 0026, aderência aos ADRs 0008/0014/0025). Os achados de risco alto foram conferidos no código antes de entrar aqui.
- Specs que tocam revogação e exclusão, em `main`: **412 exemplos, 0 falhas** (jobs da revogação e dos consumidores de triagem, `revoke_consent`, check-in, `citizens/*erasure*`, `erasure_requests`, guarda de bairro, `spec/invariants`, `spec/cities`, `requests/citizen_api`).
- Passada na tela do dashboard em dev (curitiba), descrita abaixo.

### Antes da passada: dev atrasado

As cidades de dev ainda estavam na migração `20260930200001`, e as mensagens tinham 98 telefones em claro: o passo de rollout `city:encrypt_message_phones:all` (fechamento do módulo 07, 2026-09-27) nunca tinha rodado em dev. Sem ele, a exclusão buscaria o telefone cifrado e deixaria as linhas antigas para trás. Rodados `city:migrate:all` (→ `20261001000002`) e `city:encrypt_message_phones:all` (curitiba 70, maringa 28; agora 0 em claro).

## Passada na tela (dev, curitiba)

| Passo | Quem | Resultado |
|---|---|---|
| Pedir exclusão de CPF com atendimento | `recepcao@` (verificador) | "Cadastro retido por base legal: há registro de atendimento. Nada foi apagado." |
| Pedir exclusão de CPF de teste, nunca atendido, com triagem concluída | `recepcao@` | "Pedido registrado. Um administrador precisa confirmar." |
| Pedir de novo o mesmo CPF | `recepcao@` | "Já existe um pedido pendente para este CPF." |
| Ver pedidos pendentes | `admin@` (municipal_admin) | Lista com data, quem pediu, nº de cadastros e celular mascarado; sem CPF |
| Confirmar exclusão | `admin@` | Painel "A exclusão é irreversível…" + código do autenticador → "Cadastro excluído."; lista vazia |
| Conferência no banco | — | CPF e celular viraram marcador, `erased_at`; nenhum cadastro pelo CPF; sessões do telefone apagadas; triagem anonimizada; consentimentos revogados; telefone da conversa vira marcador; pedido `confirmed` (pedido por `recepcao@`, decidido por `admin@`) com CPF em marcador; eventos só com `request_id` e `origin` |

Achados só visuais (ficam para o ciclo de interface): o item "Atendimento" do menu sobrepõe "Operação" na largura de 1024 px; a tabela de pedidos desalinha o cabeçalho "Celular"; a página desloca na horizontal ao abrir a confirmação.

## F-07.15 — Revogação

| Item | Evidência |
|---|---|
| Origens `web`/`whatsapp`/`erasure`, evento sem texto do cidadão | `app/commands/revoke_consent.rb:5,8,30`; specs `revoke_consent_spec.rb:44,52` |
| Anonimização, com trava e sem tocar a triagem atendida | `app/commands/triages/anonymize.rb:12-25`; `anonymize_revoked_triage_job_spec.rb:137,148` |
| Bairro vai a nulo com `anonymized_at` | `db/city_triggers.sql:568-590`; `territory_tables_guard_spec.rb:43` |
| Check-in recusa anonimizada ou revogada, com trava e reconferência | `check_in_eligibility.rb:12,22-23`, `check_in.rb:35-37`, `check_in_by_exception.rb:45-47`; specs `check_in_eligibility_spec.rb:56,62`, `check_in_spec.rb:143,154`, `check_in_by_exception_spec.rb:113` |
| Trilha só com referência | `complete_triage.rb:27-28`; `complete_triage_payload_spec.rb:8` |
| Consumidores pulam a anonimizada | relatório, painel, notificação e alerta (`*_job.rb`); specs em cada `*_job_spec.rb` |
| Analytics trata a concluída revogada como revogada (ADR 0025) | `analytics/consolidate/base.rb:15-27`; `analytics_invariants_spec.rb:288` |

**Lacunas:**

- **G1 (alta, quebra invariante):** `AnonymizeRevokedTriageJob` só olha `aborted_by_revocation` e `completed`. Triagem terminada por tempo (`aborted_by_timeout`) ou cancelada (`aborted_by_cancellation`) e depois revogada mantém respostas e bairro. O ADR diz "sem atendimento: anonimizada, concluída ou interrompida".
- G3 (baixa): o payload de `triage.urgent` não tem asserção.
- G4 (baixa): nenhuma spec prova que o relatório emitido não muda na revogação.
- G7 (cosmético): comentário de `anonymize.rb:3` diz que a exclusão apaga atendimento (não apaga); título de exemplo antigo em `anonymize_revoked_triage_job_spec.rb:63`; fixture com chave `reason`.

**Veredito:** **Verified** depois das correções (G1, G3, G4, G7 fechadas).

## F-07.16 — Exclusão do cadastro

| Item | Evidência |
|---|---|
| Rotas em `/attendance`, papéis, step-up, 409 em deadlock | `config/routes.rb:96-99`; `erasure_requests_controller.rb:18-19,37,43-45`; `erasure_requests_spec.rb:33,48,63,122,131,140,159` |
| Pedido: documento, todos os pares, retido se atendido, um pendente por CPF | `citizens/request_erasure.rb:9,19-33`; `request_erasure_spec.rb:18` |
| Confirmação: trava, reconferência, casca, o que sai e o que fica | `citizens/erase.rb:14-74`; `erase_spec.rb:24,81,93,147,156,168,188,198` |
| Recusa com motivo ≥ 10 | `citizens/reject_erasure.rb:12-21`; `reject_erasure_spec.rb` |
| Pedido só com acréscimos, decisão uma vez | `db/city_triggers.sql:672-704`; `citizen_erasure_request_spec.rb:16,21,33,40,52` |
| Dashboard | `ErasureRequest.tsx`, `ErasureRequests.tsx` (step-up por `SensitiveAction`); testes de cada um |

**Lacunas:**

- **G2 (alta, quebra invariante):** um pedido **recusado** guarda o CPF cifrado (determinístico, buscável). Se outro pedido do mesmo CPF for confirmado depois, o recusado continua lá, e "nenhum CPF se recupera depois da exclusão" deixa de valer. (Pedido retido não gera o problema: retido significa atendido, e o cadastro de quem foi atendido nunca é excluído.)
- G5 (baixa, já é a pendência dashboard#9): admin sem autenticador não chega à Segurança pela tela.
- G6 (baixa): a corrida check-in × exclusão não tem teste com concorrência real; a proteção vem da trava de FK e da reconferência sob trava (revisão de 2026-10-01).

**Veredito:** **Verified** depois das correções (G2 e a rotação de chave fechadas; G5 segue como dashboard#9, G6 aceito).

## Aderência aos outros ADRs

- **0008:** consentimento continua só com acréscimo (`rota_consent_guard`), e a exclusão revoga pelo `RevokeConsent`.
- **0014:** trilha imutável, piso de 12 meses no DELETE, payloads novos só com referência. Eventos antigos com `trail` ficam até a purga (aceito no ADR 0026; não há produção).
- **0025:** a regra de consolidação para a concluída revogada continua verdadeira.

## Critério de fechamento do módulo 07 (o que toca revogação e exclusão)

| Invariante | Spec |
|---|---|
| Revogada sem atendimento fica sem conteúdo clínico | `anonymize_revoked_triage_job_spec.rb:137` (G1 amplia aos outros estados) |
| Triagem atendida não muda | `anonymize_revoked_triage_job_spec.rb:148` |
| CPF atendido nunca é excluído | `request_erasure_spec.rb:18`, `erase_spec.rb:168` |
| Duas pessoas e step-up | `erase_spec.rb:188`, `erasure_requests_spec.rb:122,140` |
| Nenhum CPF/telefone recuperável depois | `erase_spec.rb:93` (G2 amplia ao pedido recusado anterior) |
| Pedido só com acréscimos | `citizen_erasure_request_spec.rb` |

## Decisão do usuário (2026-10-01)

Fechar G1 e G2 (e as menores) com TDD antes de subir; o pedido recusado deixa de guardar o CPF.

## Correções (TDD; RED comportamental antes de cada correção)

api (`mod-7/verify-adr-0026`):

- `27ab35a` — G1 e G7: o job anonimiza toda triagem da conversa revogada que não esteja `in_progress` (cobre `aborted_by_timeout` e `aborted_by_cancellation`), sempre por `Triages::Anonymize.call` (trava e reconferência do atendimento); comentário e títulos de spec corrigidos.
- `63d29fd` — G2: a recusa troca o CPF do pedido pelo marcador, na mesma UPDATE; o trigger aceita a troca do `cpf` junto da mudança para `confirmed` **ou** `rejected`. Migração de cidade `20261001000003_tombstone_rejected_erasure_cpf`. A varredura do `erase_spec` ganhou um pedido recusado anterior do mesmo CPF.
- `d2efc15` — G3 e G4: payload exato de `triage.urgent`; o relatório emitido não muda na revogação.
- `a3b18ee` — achado da revisão: a rotação de chave (`CityRekey`) e a re-cifra (`ReencryptionJob`) reescrevem o CPF de todo pedido, e o trigger recusava qualquer mudança em pedido decidido — a cidade com algum pedido decidido não conseguiria rotacionar a chave. Agora um pedido decidido aceita só a troca do `cpf` (re-cifra), como a evidência do consentimento; status, quem decidiu, quando, motivo, quem pediu e o cadastro apresentado continuam imutáveis. Specs de rekey e re-cifra com pedidos `confirmed`, `rejected` e `retained`.
- `14fe0e7` — a triagem `in_progress` não é tocada pelo job (a revogação já a converte).

Revisão independente (opus) e re-revisão: aprovado, nada crítico. Resíduos aceitos: um pedido decidido pode ter o `cpf` reescrito para qualquer valor (necessário para a re-cifra; a coluna segue cifrada e só a camada de comandos escreve); o motivo da recusa é texto livre (orientar na tela a não escrever o CPF).

**Suíte completa do api no branch: 3068/0** (worker parado). Merge publicado: api `73b4a8e`.

### Rollout

- Migrações de cidade até `20261001000003` (`city:migrate:all`).
- Sem backfill: pedidos recusados e triagens interrompidas por tempo/cancelamento revogados entre `eafe586` e esta migração ficam como estão. Não há produção; em dev não há pedido recusado.
- Em dev foram rodados `city:migrate:all` e `city:encrypt_message_phones:all` (este estava pendente desde 2026-09-27).

## Resumo

| F-ID | Veredito | Lacunas (fechadas) |
|---|---|---|
| F-07.15 | **Verified** | G1 (interrompida por tempo/cancelamento), G3, G4, G7, `in_progress` fixado |
| F-07.16 | **Verified** | G2 (CPF no pedido recusado), rotação de chave com pedido decidido |
| Critério de fechamento (revogação e exclusão) | **cumprido** | invariantes acima, com G1 e G2 incluídos nas specs |
