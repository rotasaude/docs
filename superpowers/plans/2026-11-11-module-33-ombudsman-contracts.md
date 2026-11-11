# Módulo 33 — Contratos entre apps

**Spec:** `superpowers/specs/2026-11-11-module-33-ombudsman-design.md` (decisões 1–19).
**ADR:** `adr/0051.md`.

Este contrato é a **fonte única** dos planos do api, do dashboard, do wpda e do maintenance para o 33. Quando um plano divergir dele, a divergência vai para a seção final "Divergências propostas ao contrato" do próprio plano. O coordenador reconcilia aqui, na §16.

**Bases** (origin/main em 2026-11-11):

| Repo | Commit |
|---|---|
| api | `3e96796d` |
| dashboard | `443faca` |
| wpda | `339e4ac` |
| maintenance | `f2086a1` |

O 33 não mexe no contracts.

**Valores fixos:**

| Item | Valor |
|---|---|
| Porta do api | 3056 |
| Vite do dashboard | 5216 |
| Vite do wpda | 5217 |
| maintenance | 5177, com o serviço do compose parado durante a prova |
| Migrações da cidade | a partir de `20261111900001`; nenhuma de plataforma |
| Branch | `feat/mod-33-ombudsman` |
| Worktree do api | `apps/api/.claude/mod33`, com `config/master.key` copiada |
| Worktrees dos fronts | `rota-saude/.worktrees/<repo>-mod33` |

**Mudança transversal.** O 33 toca `memberships` (CHECK de papel), `exam_files` (finalidade nova), a caixa de avisos do cidadão (`CitizenApi::NoticesController`), a exclusão LGPD (`Citizens::Erase` e `Citizens::RequestErasure`), `CityEncryption::CITY_KEYED_TARGETS`, `Platform::Features` e o primeiro formulário público de escrita. A suíte completa do api é obrigatória antes do merge.

## 1. Nomes no código

**Interruptores** (no fim do `CATALOG` de `Platform::Features`; os specs que fixam a ordem são atualizados):

| Chave | Descrição | `requires` |
|---|---|---|
| `ombudsman` | "Ouvidoria do SUS municipal" | `[]` |
| `ombudsman_anonymous_web` | "Denúncia anônima sem login no canal do cidadão" | `%w[feature:ombudsman]` |

As duas vão ao cidadão em `GET /citizen/session` (não entram em `CITIZEN_HIDDEN`). Sem `ombudsman`, toda rota do módulo responde 403 `feature_disabled`, como os outros gates (Q1).

**Papéis** (em `Membership::ROLES`, com migração de cidade que recria `ck_memberships_role`; no dashboard, `ROLE_OPTIONS`):

| Papel | Privilegiado | Faz |
|---|---|---|
| `ombudsman` | sim (`PRIVILEGED_ROLES`) | tudo do `ombudsman_staff`, mais: ver identidade preservada, prorrogar, arquivar, reabrir, decidir pedido de reabertura, alterar resolutividade, registrar consentimento, configurar respondentes, assuntos e prazos, painel por profissional, relatório anual |
| `ombudsman_staff` | sim (U3) | registrar pela equipe, classificar, encaminhar e cancelar encaminhamento, pedir e registrar complemento, resposta intermediária e conclusiva, comprovante, painel agregado |
| `ombudsman_responder` | não | só os encaminhamentos dos destinos em que está em `ombudsman_responders` |

`departments` é cadastrado pelo `municipal_admin`. O acesso fica num módulo `Ombudsman::Access` (padrão de `Psychosocial::Access`).

**Tabelas novas da cidade:**

- **`departments`:** `name` (2–80, único por cidade sem diferenciar caixa), `active`, `updated_by_user_id`. Nunca é apagada (`rota_no_delete`).
- **`ombudsman_responders`:** `user_id`, `department_id`, `health_unit_id` (CHECK: exatamente um dos dois), `active`, `created_by_user_id`. Único por (`user_id`, destino). Nunca apagada; desligar é `active = false`.
- **`ombudsman_subjects`:** `name` (2–80), `parent_id` (subassunto; só dois níveis), `active`. Semente inicial por migração de cidade (§7).
- **`ombudsman_settings`:** uma linha por cidade. `response_days` (30), `extension_days` (30), `urgent_days` (15), `complement_days` (20), `forwarding_days` (20), `forwarding_extension_days` (20), `warning_days` (5), `anonymous_daily_cap` (50), `updated_by_user_id`. Todos inteiros de 1 a 120, exceto o teto (1 a 1000).
- **`ombudsman_manifestations`:**
  - `protocol` (único, `AAAA-NNNNNN`), `kind` (`complaint` | `report` | `praise` | `suggestion` | `request`), `identification_mode` (`identified` | `preserved` | `anonymous`), `channel` (`web` | `in_person` | `phone` | `ouvidorsus`), `ouvidorsus_number` (só em `ouvidorsus`, até 40);
  - `citizen_id` (nulo no anônimo e no registro sem cidadão), `manifestant_name`, `manifestant_contact`, `manifestant_cpf` (cifrados; preenchidos no registro sem cidadão e na exclusão LGPD, §13);
  - `body` (cifrado, 20–4000);
  - `subject_id`, `referred_health_unit_id`, `referred_professional_id` (do módulo 10), `urgent`;
  - `state` (§2), `received_at`, `cycle_started_on` (data local), `extended` (bool), `extension_reason` (cifrado), `suspended_days` (int), `complement_requested_at`, `complement_used` (bool, por ciclo), `resolution` (`resolved` | `not_resolved` | nulo), `archive_reason`, `duplicate_of_id`, `external_body_name` (até 120, no arquivamento `forwarded_elsewhere`), `concluded_at`, `cycle` (1, 2, …);
  - `created_by_user_id` (nulo no web). Não há `lock_version`: a concorrência usa `expected_state` (§2).
- **`ombudsman_events`:** `ombudsman_manifestation_id`, `kind` (§2), `actor_user_id` (nulo quando o ator é o cidadão ou o sistema), `actor` (`staff` | `citizen` | `system`), `text` (cifrado, opcional), `data` (jsonb sem dado pessoal: ids, rótulos, estado anterior e novo), `visible_to_citizen` (bool), `created_at`. Só acréscimos.
- **`ombudsman_forwardings`:** `ombudsman_manifestation_id`, `department_id` ou `health_unit_id` (CHECK de um), `extract` (cifrado, 20–4000), `due_on`, `extended`, `extension_reason` (cifrado), `state` (`pending` | `replied` | `declined` | `cancelled`), `reply` (cifrado), `replied_by_user_id`, `replied_at`, `created_by_user_id`.
- **`ombudsman_identity_accesses`:** `ombudsman_manifestation_id`, `user_id`, `justification` (cifrado, 10–500), `scope` (`identity` | `attachment`), `exam_file_id` (no `attachment`), `created_at`. Só acréscimos.
- **`ombudsman_attachments`:** `ombudsman_manifestation_id`, `exam_file_id` (único), `uploaded_by` (`citizen` | `staff`), `event_id` (o complemento que trouxe o anexo, opcional).
- **`ombudsman_reopen_requests`:** `ombudsman_manifestation_id`, `reason` (cifrado, 10–2000), `state` (`pending` | `accepted` | `rejected`), `decision_text` (cifrado), `decided_by_user_id`, `decided_at`. Um `pending` por manifestação.
- **`ombudsman_satisfaction_responses`:** `ombudsman_manifestation_id` (único), `met` (`yes` | `partly` | `no`), `clear` (bool), `score` (1–5), `created_at`.
- **`ombudsman_notices`:** `citizen_id`, `ombudsman_manifestation_id`, `kind` (`interim_response` | `complement_requested` | `final_response` | `archived` | `reopen_decided`), `read_at`, `created_at`. Só `read_at` muda (padrão de `transport_trip_notices`).
- **`ombudsman_annual_reports`:** `year` (único), `recurring_problems`, `actions_taken`, `notes` (texto simples, até 8000 cada, não cifrado: é documento público), `state` (`draft` | `published`), `snapshot` (jsonb dos números, gravado na publicação), `published_by_user_id`, `published_at`.

**Sequence:** `ombudsman_protocol_seq`, criada em `db/city_triggers.sql`. O protocolo é `"#{ano local de received_at}-#{format('%06d', nextval)}"`, sem reiniciar por ano. A denúncia anônima também recebe protocolo, que nunca é mostrado ao cidadão.

**Mudanças em tabelas existentes:**

| Tabela | Mudança |
|---|---|
| `exam_files` | `PURPOSES` e o CHECK ganham `ombudsman_attachment` |
| `memberships` | `ck_memberships_role` ganha os três papéis |

**Eventos de domínio** (`DomainEvents.publish`, só ids, ligados em `config/initializers/domain_events.rb` quando houver assinante): `ombudsman.manifestation_received`, `ombudsman.forwarded`, `ombudsman.forwarding_replied`, `ombudsman.concluded` (`answered` ou `archived`, no `data` só o estado), `ombudsman.reopened`, `ombudsman.identity_viewed`, `ombudsman.anonymous_cap_reached`. Um spec de sigilo fixa a lista exata e confere que nenhum payload tem texto, nome, CPF, assunto ou unidade.

## 2. Estados, ações e prazos

**Estados de `ombudsman_manifestations.state`:**

| Estado | Significado |
|---|---|
| `received` | registrada, ainda sem classificação |
| `in_analysis` | classificada, com a ouvidoria |
| `forwarded` | há pelo menos um encaminhamento `pending` |
| `awaiting_complement` | complemento pedido; prazo suspenso |
| `answered` | resposta conclusiva enviada |
| `archived` | arquivada com motivo |

**Ações** (todas com `expected_state` no corpo; se o estado atual for outro, 409 `manifestation_changed`, padrão das Q15/Q16 do 32b):

| Ação | De | Para | Quem | Regras |
|---|---|---|---|---|
| `classify` | `received`, `in_analysis` | `in_analysis` | staff | `subject_id` obrigatório (subassunto ou assunto sem filhos); `kind`, unidade, profissional e `urgent` podem mudar; cada mudança vira evento `classified` com o antes e o depois no `data` |
| `forward` | `in_analysis`, `forwarded` | `forwarded` | staff | destino ativo com pelo menos um respondente ativo, senão 422 `destination_without_responder`; extrato obrigatório; `due_on = hoje + forwarding_days` |
| `cancel_forwarding` | `forwarded` | `in_analysis` se não sobrar `pending` | staff | só `pending` |
| responder `reply` / `decline` | — | `in_analysis` se não sobrar `pending` | responder | `decline` com motivo (não é do destino) |
| `request_complement` | `in_analysis` | `awaiting_complement` | staff | uma vez por ciclo (`complement_used`), senão 422 `complement_already_used`; nunca no anônimo (422 `complement_not_allowed`); texto obrigatório, vai ao cidadão |
| complemento (cidadão ou staff) | `awaiting_complement` | `in_analysis` | cidadão no wpda; staff registra o recebido por outro canal | `suspended_days += dias corridos entre o pedido e a resposta` |
| `extend` | `in_analysis`, `forwarded`, `awaiting_complement` | — (marcador) | ombudsman | uma vez por ciclo, senão 422 `already_extended`; justificativa obrigatória, vai ao cidadão como resposta intermediária |
| `interim_response` | qualquer aberto | — | staff | texto ao cidadão; não mexe no prazo; não no anônimo |
| `final_response` | `in_analysis` | `answered` | staff | texto obrigatório; `resolution` obrigatória em `complaint`, `request` e `report`, proibida nos outros; recusa se houver encaminhamento `pending` (422 `forwarding_pending`); no anônimo o texto fica interno |
| `archive` | `received`, `in_analysis`, `awaiting_complement` | `archived` | ombudsman (e o job em `no_complement`) | motivo da lista abaixo; `duplicate` exige `duplicate_of_id` de outra manifestação; `forwarded_elsewhere` exige `external_body_name` |
| `reopen` | `answered`, `archived` | `in_analysis` | ombudsman | `cycle += 1`; novo `cycle_started_on = hoje`; zera `extended`, `complement_used`, `suspended_days`, `resolution`; pode ser de ofício ou ao aceitar pedido |
| `change_resolution` | `answered` | — | ombudsman | justificativa obrigatória; evento |

**Motivos de arquivamento:** `duplicate`, `no_complement`, `lack_of_clarity`, `lack_of_civility`, `improper`, `loss_of_object`, `forwarded_elsewhere`.

**Pedido de reabertura do cidadão:** no wpda, até 30 dias depois de `concluded_at`, uma vez por ciclo, com motivo. O ombudsman aceita (faz `reopen`) ou recusa com texto, que vai ao cidadão (aviso `reopen_decided`). Fora do prazo: 422 `reopen_window_closed`.

**Prazo** (dias corridos, na data local da cidade; fonte única em SQL, `Ombudsman::Deadlines.due_on_sql`, com espelho em Ruby, padrão de `CareRegulation::Deadlines`):
- `due_on = cycle_started_on + (urgent ? urgent_days : response_days) + (extended ? extension_days : 0) + suspended_days`;
- em `awaiting_complement`, o prazo fica congelado: a cor é `paused` e o vencimento mostrado é o recalculado na volta;
- `color`: `green`; `yellow` quando `due_on - hoje <= warning_days`; `red` quando `hoje > due_on`; `paused`; e nada nos estados finais;
- complemento vence em `complement_requested_at::date + complement_days`; o job arquiva com `no_complement` no dia seguinte.
- Encaminhamento: `due_on` gravado; `extend` do responder uma vez (+`forwarding_extension_days`), com motivo; cor pela mesma regra.

## 3. API do cidadão (wpda), com sessão

Base `/citizen/ombudsman`, sessão de `CitizenAuthentication`, só JSON (415 fora disso). A sessão é do telefone: o cidadão escolhe `citizen_id` entre os da sessão, senão 403 `not_own_citizen`. O par declarado basta.

| Rota | Corpo / resposta |
|---|---|
| `GET /citizen/ombudsman/manifestations` | `{ manifestations: [{ id, protocol, kind, identification_mode, state, received_at, due_on, color, cpf_masked? , has_unread }] }`, de todos os cidadãos da sessão, mais novo primeiro. Nada do anônimo (não tem `citizen_id`) |
| `GET /citizen/ombudsman/manifestations/:id` | `{ id, protocol, kind, identification_mode, state, received_at, due_on, body, health_unit: {id,name}?, attachments: [{id, filename}], timeline: [{ kind, at, text? }], can_complement, can_request_reopen, can_answer_satisfaction }`. `timeline` traz só eventos `visible_to_citizen`: recebida, pedido de complemento, complemento enviado, prorrogação (com a justificativa), resposta intermediária, resposta conclusiva, arquivamento (com o rótulo do motivo), pedido e decisão de reabertura. Nunca o extrato, o encaminhamento, o assunto interno nem o nome de quem respondeu |
| `POST /citizen/ombudsman/manifestations` | `{ citizen_id, kind, identification_mode: "identified" \| "preserved", body, health_unit_id?, attachment_ids? }` → 201 `{ id, protocol, due_on }`. `anonymous` aqui → 422 `identification_mode` |
| `POST /citizen/ombudsman/manifestations/:id/complement` | `{ body, attachment_ids? }` → 200; só em `awaiting_complement` |
| `POST /citizen/ombudsman/manifestations/:id/reopen_request` | `{ reason }` → 201 |
| `POST /citizen/ombudsman/manifestations/:id/satisfaction` | `{ met, clear, score }` → 201; só em `answered`, até 30 dias de `concluded_at`, uma vez (409 `already_answered`) |
| `POST /citizen/ombudsman/attachments` | `{ filename, content_type, data }` (base64) → 201 `{ id }`. PDF, JPEG ou PNG por magic bytes, até 10 MB, até 3 por manifestação e por complemento. Pelo `Exams::Files` com `purpose = ombudsman_attachment`, dono = a sessão do cidadão; órfão apagado em 24 h |

**Limites** (`rate_limit` com `RateLimitStore`): criar manifestação 10 por dia por sessão; anexos 20 por hora por sessão.

**Avisos:** `ombudsman_notices` vira a 4ª fonte de `CitizenApi::NoticesController`, com `{ id, kind: "ombudsman", event: <kind>, protocol, created_at, read }` e entra no `unread_count`. O aviso não tem texto. O `read` procura também nessa fonte.

## 4. API pública (denúncia anônima), sem sessão

| Rota | Corpo / resposta |
|---|---|
| `GET /citizen/ombudsman/public_config` | `{ anonymous_web: bool, units: [{ id, name }] }` (unidades ativas). 404 `feature_disabled` sem `ombudsman` |
| `POST /citizen/ombudsman/anonymous_reports` | `{ body, health_unit_id? }` → 201 `{ ok: true }`, **sem protocolo**. 404 `feature_disabled` sem `ombudsman_anonymous_web` |

- Só `kind = report`, `identification_mode = anonymous`, `channel = web`.
- Só JSON; corpo acima de 16 KB → 413 `payload_too_large`; qualquer chave fora de `body` e `health_unit_id` → 422 com o `path`.
- `rate_limit` por IP: 3 por hora e 10 por dia → 429 `rate_limited`.
- Teto da cidade: `anonymous_daily_cap` por dia local. Acima → 429 `daily_cap_reached`, e o primeiro bloqueio do dia publica `ombudsman.anonymous_cap_reached` (sem dado), que manda um e-mail só com a contagem ao `ombudsman`.
- `body` já está em `filter_parameters`; o spec confere que o texto não aparece no log da requisição.
- Nenhum cookie é criado; o endpoint não lê nem cria `citizen_session`.

## 5. API da equipe (dashboard)

Base `/ombudsman`, sessão de equipe, gate `ombudsman`, `Ombudsman::Access`.

| Rota | Quem | Notas |
|---|---|---|
| `GET /ombudsman/manifestations` | staff, ombudsman | filtros `state[]`, `color[]`, `kind[]`, `channel[]`, `subject_id`, `health_unit_id`, `q` (só protocolo); paginação por cursor como nas listas do 31; item: `{ id, protocol, kind, identification_mode, channel, state, subject, health_unit, urgent, received_at, due_on, color, pending_forwardings }` |
| `GET /ombudsman/manifestations/:id` | staff, ombudsman | dados, `body`, eventos, encaminhamentos com extrato e resposta, anexos (sem conteúdo), pedido de reabertura. Identidade: no `identified`, `{ name, cpf_masked, phone_masked }`; no `preserved` e `anonymous`, `identity: "preserved"` |
| `POST /ombudsman/manifestations` | staff, ombudsman | registro pela equipe: `{ kind, identification_mode, channel: in_person \| phone \| ouvidorsus, ouvidorsus_number?, citizen_id?, manifestant_name?, manifestant_contact?, manifestant_cpf?, body, health_unit_id?, attachment_ids? }`. `anonymous` só com `report` e sem dados de pessoa. Busca do cidadão pelo CPF reusa a busca de paciente que já existe (o plano do api indica qual) |
| `POST /ombudsman/manifestations/:id/<ação>` | §2 | `classify`, `forward`, `request_complement`, `complement` (registro do recebido por outro canal), `extend`, `interim_response`, `final_response`, `archive`, `reopen`, `change_resolution`; cada um com `expected_state` |
| `POST /ombudsman/forwardings/:id/cancel` | staff | |
| `POST /ombudsman/reopen_requests/:id/decide` | ombudsman | `{ decision: accept \| reject, text }` |
| `POST /ombudsman/manifestations/:id/identity` | ombudsman | `{ justification }` → identidade completa (`name`, `cpf`, `phone`, `manifestant_*`); grava `ombudsman_identity_accesses` e `ombudsman.identity_viewed`. Outros papéis → 403 |
| `POST /ombudsman/manifestations/:id/identity_consent` | ombudsman | `{ granted: bool, text }`, evento `identity_consent_recorded`; não revela nada a ninguém |
| `GET /ombudsman/manifestations/:id/receipt` | staff | `{ protocol, kind, received_at, due_on, channel }` para o comprovante; nunca na anônima |
| `POST /ombudsman/attachments` | staff | igual ao do cidadão, dono = o usuário |
| `POST /ombudsman/attachments/:id/link` | staff, ombudsman | URL curta assinada (padrão de `tfd/files_controller.rb`). Manifestação `preserved` ou `anonymous`: só ombudsman, com `{ justification }`, e grava acesso `attachment` |

## 6. Ponto de resposta

| Rota | Notas |
|---|---|
| `GET /ombudsman/inbox` | encaminhamentos dos destinos em que o usuário é respondente ativo: `{ id, destination, extract, due_on, color, state, created_at }`. Nunca o protocolo, a identidade, o texto original, o assunto nem os anexos da manifestação |
| `POST /ombudsman/inbox/:id/reply` | `{ body }`; só `pending` |
| `POST /ombudsman/inbox/:id/decline` | `{ reason }` |
| `POST /ombudsman/inbox/:id/extend` | `{ reason }`; uma vez |

O respondente que não tem também `ombudsman_staff` não lê nenhuma rota do §5 (403).

## 7. Configuração

| Rota | Quem |
|---|---|
| `GET/POST/PATCH /departments` | `municipal_admin` (leitura também para staff e ombudsman) |
| `GET/POST/PATCH /ombudsman/responders` | ombudsman |
| `GET/POST/PATCH /ombudsman/subjects` | ombudsman |
| `GET/PATCH /ombudsman/settings` | ombudsman |

Semente de assuntos (migração de cidade, `active = true`; a cidade edita): "Atendimento" (demora, conduta de profissional, acolhimento), "Acesso" (agendamento, fila, falta de profissional), "Medicamentos" (falta, dispensação), "Exames" (agendamento, resultado), "Estrutura" (limpeza, instalações), "Transporte", "Vigilância em saúde", "Outros". A lista oficial do Manual de Tipificação do OuvidorSUS 3 é pendência (§15).

## 8. Identidade preservada

- "Identidade" = `citizen.full_name`, CPF, telefone e os três `manifestant_*`.
- Em `preserved` e `anonymous`, nenhuma resposta do §5 traz identidade fora de `POST .../identity`, nem para o `ombudsman`.
- Anexos dessas manifestações só abrem pelo ombudsman, com justificativa.
- O `body` fica visível à equipe da ouvidoria em qualquer modo (decisão 12).
- Nada de identidade no extrato: a interface avisa, como o `FrozenTextNotice` do api#32. Não há detecção automática.

## 9. Avisos e SMS ao cidadão

- Cria aviso (`ombudsman_notices`) para o `citizen_id` em: pedido de complemento, prorrogação e resposta intermediária (`interim_response`), resposta conclusiva, arquivamento, decisão de reabertura.
- SMS opcional pelo padrão de `Transport::Notify#sms_allowed?` (cidade com SMS ligado, provedor configurado, janela de 8h às 20h, opt-in, sem silêncio), texto fixo, sem protocolo: "Rota Saúde: há novidade na sua manifestação à Ouvidoria da Saúde. Veja no canal do cidadão." Fora da janela, não envia (não reagenda).
- Registro pela equipe sem `citizen_id`: nenhum aviso; a equipe retorna por telefone.

## 10. Pesquisa de satisfação

Rótulos: "Sua demanda foi atendida?" (Sim, Em parte, Não); "A resposta foi fácil de entender?" (Sim, Não); "De 1 a 5, qual sua satisfação com a ouvidoria?". Só manifestações com `citizen_id`, em `answered`, até 30 dias de `concluded_at`, uma por manifestação. A resposta não tem texto livre.

## 11. Painel e relatório anual

- `GET /ombudsman/dashboard?from=&to=` (staff, ombudsman): totais por `kind`, `channel`, `state`, assunto (primeiro nível), unidade referida; `on_time_rate` (concluídas até `due_on`), `avg_days_to_conclude`, `resolution_rate` (sobre as que têm resolutividade), satisfação (`met` por opção, `clear` sim/não, média do `score`, número de respostas). Contagens de 1 a 4 escondidas com `Analytics::Suppression` (`group` esconde o total se alguma parte for escondida).
- `GET /ombudsman/dashboard/professionals?from=&to=` (ombudsman): contagem por profissional referido e tipo, sem supressão.
- `GET/PATCH /ombudsman/annual_reports/:year` e `POST /ombudsman/annual_reports/:year/publish` (ombudsman). A publicação grava `snapshot` com os números do §11 para o ano, com a mesma supressão; depois disso o relatório não muda (trigger). A tela tem versão para impressão; a cidade publica no próprio site.

## 12. Job diário

`Ombudsman::DeadlinesJob` (`prepend EachCityJob`, gate `ombudsman`, em `config/recurring.yml` às 7h de Brasília):
1. Arquiva com `no_complement` as manifestações em `awaiting_complement` cujo complemento venceu, com evento `system` e aviso ao cidadão.
2. Manda e-mail ao `ombudsman` e ao `ombudsman_staff` com as contagens de amarelas e vermelhas, e a cada respondente com as contagens dos encaminhamentos dele. Só números, sem protocolo.
3. Idempotente no dia: um índice único por dia numa tabela `ombudsman_daily_runs` (no banco da cidade), no padrão de `care_plan_review_job.rb`.

## 13. LGPD, cifra, imutabilidade e eventos

- **Cifra:** todos os campos marcados como cifrados entram em `CityEncryption::CITY_KEYED_TARGETS`.
- **Imutabilidade** (`db/city_triggers.sql`):
  - `ombudsman_events`, `ombudsman_identity_accesses` e `ombudsman_satisfaction_responses`: `rota_append_only` e `rota_reencryption_only`;
  - `ombudsman_manifestations`: `protocol`, `body`, `channel`, `identification_mode`, `received_at`, `created_by_user_id` não mudam (só recifragem); `citizen_id` só muda para nulo (exclusão);
  - `ombudsman_forwardings`: `extract` não muda; `reply` só passa de nulo a preenchido;
  - `ombudsman_annual_reports` publicado não muda;
  - nenhuma tabela do módulo aceita DELETE, exceto `ombudsman_notices` (exclusão do cidadão) e `ombudsman_attachments` órfão.
- **Exclusão LGPD (ADR 0026):** `Citizens::RequestErasure` marca a manifestação como retida (`ombudsman_record`). `Citizens::Erase` copia `full_name`, CPF e telefone do cidadão para os `manifestant_*` da manifestação, põe `citizen_id = NULL`, apaga os `ombudsman_notices` dele e mantém as respostas de satisfação. A ordem de travas segue a de `Citizens::Erase`. A purga ao fim da guarda (10 anos; denúncia, 20) é pendência.
- **Logs e URLs:** nenhum texto, nome, CPF, assunto ou unidade em URL, log ou evento. Os campos novos que trazem texto (`body`, `extract`, `reason`, `justification`, `text`) entram em `filter_parameters` quando ainda não estão.
- **Trilha:** a leitura do detalhe não grava `clinical_record.viewed` (não é prontuário). A única trilha de leitura é a da identidade e do anexo protegido.

## 14. Telas

**dashboard:** área "Ouvidoria" no `NAV_GROUPS`, visível com o interruptor e o papel.
- Caixa (lista com filtros e cor de prazo).
- Manifestação (dados, linha do tempo, encaminhamentos, ações do §2, identidade com justificativa, anexos).
- Nova manifestação (registro pela equipe e comprovante para impressão).
- Respostas à ouvidoria (`/ombudsman/inbox`), para o respondente.
- Configuração (respondentes, assuntos, prazos) e Setores da secretaria (para o `municipal_admin`).
- Painel e Relatório anual (com impressão).
- Aviso fixo no campo do extrato: "Não escreva o nome nem dados de quem fez a manifestação."

**wpda:**
- Tela inicial: entrada "Ouvidoria" com "Fazer uma manifestação" (fluxo de CPF e código, se ainda não entrou) e "Denúncia anônima" (sem login; só aparece com `anonymous_web`).
- Denúncia anônima: texto do aviso antes do envio ("Sem identificação, você não recebe protocolo nem acompanha a resposta.").
- Nova manifestação: escolha do cidadão (se a sessão tem mais de um), tipo, modo, unidade, texto e anexos.
- "Minhas manifestações": lista e detalhe, com complemento, pedido de reabertura e pesquisa.
- Avisos: rótulo do aviso de ouvidoria.

**maintenance:** rótulos das duas chaves em `src/lib/features.ts` e do `missing` `ombudsman_disabled`.

## 15. Entrega, testes e pendências

- **Ordem:** api → dashboard → wpda → maintenance.
- **Prova no navegador:** registro web identificado, preservado e anônimo; registro pela equipe; encaminhamento e resposta do ponto; complemento; prorrogação; resposta conclusiva; aviso no wpda; satisfação; identidade com justificativa; painel; relatório anual.
- **Suíte completa do api** antes do merge (transversal).
- **Gates de go-live:** decreto municipal de ouvidoria (prazos), SMS.
- **Pendências já abertas:** api#148 (Fala.BR), api#149 (ficha do OuvidorSUS), api#150 (totem e perguntas próprias).
- **Pendências a abrir no fechamento:** purga ao fim da guarda; taxonomia oficial do OuvidorSUS; captcha se o limite não bastar.

## 16. Decisões

Reconciliação dos planos de 2026-11-11: api (A1–A31), dashboard (D1–D24), wpda (W1–W15) e maintenance (M1–M4). Onde o plano e esta seção divergem, vale esta seção. Os códigos de erro, nomes e formatos fixados no plano do api valem para os fronts.

**Do usuário:**

| # | Decisão |
|---|---|
| U1 (W2) | A ouvidoria aceita o CPF direto, sem o termo da triagem. Se a sessão do telefone não tem o cidadão, `POST /citizen/ombudsman/manifestations` aceita `{ cpf, full_name }` em vez de `citizen_id`, e o api cria o par declarado (CPF + telefone da sessão) **sem `consent_version`**. O plano do api confere que nenhum fluxo assume termo para todo `Citizen`; a triagem continua exigindo o termo na própria entrada. |
| U2 (W13) | A denúncia anônima abre também no desktop em produção, como exceção ao bloqueio por dispositivo, igual à do relatório público. O resto do wpda segue só no celular. |
| U3 (A2, D3) | `ombudsman_staff` também entra em `PRIVILEGED_ROLES` (segundo fator), porque lê texto de saúde e a identidade no modo `identified`. A tabela de papéis do §1 fica corrigida. |
| U4 (D22) | Abrir a identidade preservada **não** pede step-up: o ombudsman já tem segundo fator, e a justificativa e a trilha valem por abertura. |

**Do coordenador:**

| # | Decisão |
|---|---|
| Q1 (A1, D1, M3, W12) | Interruptor desligado responde **403 `feature_disabled`**, como todos os portões do api. O §1 e o §4 ficam corrigidos. |
| Q2 (A3, A4, D17, W8) | O dono do anexo fica em `ombudsman_attachments` (`citizen_session_id` sem FK, ou `uploaded_by_user_id`), sem coluna nova em `exam_files` e sem `claim!`. A linha nasce órfã no upload e some em cascata com o `ExamFile` órfão. `filename` é cifrado e **nunca é mostrado**: as telas mostram "Anexo N"; o wpda sempre manda `anexo-N.ext`. |
| Q3 (A5) | Prazo pelo par `overdue_sql` + método Ruby, no padrão de `RegulationRequest`. |
| Q4 (A6, D16, W4) | A lista de `ombudsman_events.kind` e o subconjunto `visible_to_citizen` são os da A6 do plano do api. A equipe vê `actor_name`; o cidadão nunca. O `text` ao cidadão vem pronto (rótulo do motivo de arquivamento já em palavras dele). |
| Q5 (A7) | A manifestação **não** retém o pedido de exclusão (senão o `Erase` nunca roda). O `Erase` copia nome, CPF e telefone para os `manifestant_*`, anula `citizen_id` e apaga os avisos. O §13 fica corrigido. |
| Q6 (A8, D6) | Busca nova `POST /ombudsman/person_lookup { cpf }` → `{ citizen_id, name, cpf_masked }` ou `null`, sobre `Citizen.not_erased`. Não cria paciente nem exige par validado. |
| Q7 (A9, D5, D7, D8) | `GET /ombudsman/options` (staff e ombudsman) traz assuntos, unidades com `has_active_responder`, setores e profissionais ativos com as unidades. `GET /ombudsman/responder_candidates` (ombudsman) traz os usuários da cidade para virar respondente. |
| Q8 (A10, D11, W6, W9) | Textos de ação 10–4000; `body` 20–4000. Contagem em code points depois do `strip`, nos dois lados. Os códigos de validação são os do plano do api. |
| Q9 (A11) | O motivo do `decline` vai em `reply`, com `state = declined`. |
| Q10 (A12) | `ombudsman_reopen_requests.cycle`, único por (manifestação, ciclo). |
| Q11 (A13) | `ombudsman_daily_runs` com `kind` (`deadlines`, `anonymous_cap`), único por (dia, kind). |
| Q12 (A14, D23) | `/departments` sem o portão da ouvidoria. |
| Q13 (A15) | `Ombudsman::Settings.ensure_defaults!` na leitura cria a linha de configuração e a semente de assuntos que faltarem (banco vindo do dump). |
| Q14 (A16, D18) | Download pela rota assinada `GET /ombudsman/files/:token`, com o caminho mascarado em `RequestLogPathMask`. O cidadão **não** baixa anexos (só vê "Anexo N"). |
| Q15 (A17) | O SMS respeita `notices_muted`. |
| Q16 (A18, W3) | O aviso de ouvidoria traz `cpf_masked`, como as outras fontes. |
| Q17 (A19) | O prazo é sempre o da configuração vigente; mudar a configuração muda as abertas. `on_time_rate` usa o ciclo vigente. |
| Q18 (A20) | Ids de unidade e assunto podem aparecer em URL e log (filtros). Nenhum id de unidade, assunto ou profissional vai em payload de evento. O §13 proíbe nomes e textos. |
| Q19 (A21) | 403 `not_own_citizen` no POST; 404 nas rotas por `:id` de outro cidadão. |
| Q20 (A22) | `public_config` lista todas as unidades ativas, CAPS inclusive. |
| Q21 (A23) | `ombudsman.anonymous_cap_reached` com payload `{}`. |
| Q22 (A24) | Registro `identified` ou `preserved` pela equipe exige `citizen_id` ou `manifestant_name`. |
| Q23 (A25, D14) | Listas com `{ items, next_before }`, cursor `before`, filtros repetidos como `state[]`; vale para a caixa e para `/ombudsman/inbox`. |
| Q24 (A26, D15) | O detalhe traz `identity_available` e, no `identified` sem cidadão, `contact`. `POST .../identity` na anônima sem dado responde 422 `no_identity` e não grava acesso. |
| Q25 (A27) | `complement_requested_at` volta a nulo quando o complemento chega. |
| Q26 (A28) | `archive duplicate` aceita `duplicate_of_protocol` ou `duplicate_of_id`. |
| Q27 (A29) | Relatório anual só se publica para ano encerrado (422 `year_not_closed`). |
| Q28 (A30, D19) | Formato do painel do plano do api: escondido é `{ suppressed: true }`; médias com 1 a 4 casos saem escondidas; sem caso, `null`; `by_subject` tem `unclassified`. |
| Q29 (A31) | Aceito: o JSON da anônima é interpretado antes do 413, e o `body` segue filtrado no log. Sem middleware novo. |
| Q30 (D2) | O campo do extrato mostra o `FrozenTextNotice` e o aviso do §14, sem mudar `src/components`. |
| Q31 (D4) | No dashboard, os papéis entram em `INVITE_ROLES` e `PRIVILEGED_ROLES` de `src/lib/team.ts`. |
| Q32 (D9) | O corpo de cada ação é o do plano do api, e toda ação responde com o detalhe atualizado. |
| Q33 (D10) | `cancel` e as ações do respondente levam `expected_state` do encaminhamento → 409 `forwarding_changed`. |
| Q34 (D12, D13) | `subject = { id, name, parent: { id, name } \| null }`; `destination = { kind: department \| health_unit, id, name }`. |
| Q35 (D20) | `GET` do relatório anual sempre 200 (rascunho vazio quando não há); `PATCH` em publicado → 409 `report_published`. |
| Q36 (D21) | `no_complement` não é oferecido na tela; só o job usa. |
| Q37 (D24) | Sai o filtro "responsável" da caixa (não há responsável no modelo). |
| Q38 (M1, M2, M4) | Rótulos do plano do maintenance, sempre diferentes da descrição; o dashboard usa os mesmos. Desligar `ombudsman` não desliga a denúncia em cascata: ela fica "ligada, falta pré-requisito", as rotas respondem `feature_disabled` e nada é apagado. O api tem spec de serviço para isso. |
| Q39 (W1) | A ouvidoria não depende do termo da triagem (ver U1). |
| Q40 (W5) | O cidadão vê `forwarded` como `in_analysis`; o api já devolve `in_analysis` no canal do cidadão. |
| Q41 (W7) | O wpda manda as rotas públicas sem cookie (`credentials: "omit"`), e o api confere que nada se liga à sessão. |
| Q42 (W10) | O detalhe do cidadão traz `color`. |
| Q43 (W11) | Todo limite responde 429 `too_many_requests`, inclusive a anônima; `rate_limited` sai do §4. O teto da cidade segue `daily_cap_reached`. |
| Q44 (W14) | O runbook pede `client_max_body_size 15m` só nas rotas de anexo. |
| Q45 (W15) | O `GET` do detalhe do cidadão marca como lidos os avisos daquela manifestação. |

**Da execução (api, 2026-11-11):**

| # | Decisão |
|---|---|
| Q46 (U1) | O `full_name` informado no par declarado vai para `manifestant_name` da manifestação, nunca para `citizens.full_name` (nome conferido em documento, ADR 0031). O par nasce por `Citizens::RegisterPerson`, sem consentimento. No `Erase`, `manifestant_* = COALESCE(manifestant_*, valor do cidadão)`, para não apagar o nome declarado. |
| Q47 (Q13) | A semente de assuntos só entra com a tabela vazia, nunca item a item (assunto renomeado ou desativado pela cidade não reaparece). A linha de configuração é criada sempre que faltar. |
| Q48 (Q32, §6) | As ações do respondente (`/ombudsman/inbox/:id/reply`, `decline`, `extend`) respondem 200 com o item atualizado do inbox, não com o detalhe. As ações da equipe e o `cancel` respondem com o detalhe. |
| Q49 (Q28) | `unclassified` é o último item de `by_subject`: `{ id: null, name: null, unclassified: true, count: <célula> }`. |
