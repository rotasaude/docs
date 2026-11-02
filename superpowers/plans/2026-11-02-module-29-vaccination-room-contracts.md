# Módulo 29 — Contratos entre apps

**Spec:** `superpowers/specs/2026-11-02-module-29-vaccination-room-design.md` · **ADR:** `adr/0047.md` · **Pesquisa:** `pesquisa/2026-11-02-sala-de-vacina-29.md`

Fonte única dos formatos entre `api`, `dashboard`, `wpda` e `maintenance`.

Convenções herdadas dos contratos do 21a ao 28 (o 28 §16 é o mais recente):
- erros `{ "error", "path"? }`, com `problems[{ "path", "error" }]` trazendo todas as recusas de uma vez;
- step-up (TOTP) onde marcado;
- interruptor desligado: 403 `feature_disabled`;
- JSON em toda escrita por cookie;
- listas com envelope e `next_before`;
- registro de outro escopo pedido por id → 404; lista com escopo alheio → 403;
- leitura nominal com trilha `clinical_record.viewed` e sem polling; contadores sem trilha;
- dado sensível só no corpo, nunca na URL nem no log;
- nenhum dado pessoal ou clínico (vacina, lote, dose) em `inspect`/`to_s`/`pretty_print`/`as_json` de modelo, evento ou log;
- paciente sem CPF como no 24a (`has_cpf`, `cns_masked`);
- imutabilidade por trigger em `db/city_triggers.sql`;
- todo `Platform.audit` novo entra no R18.

## 1. Nomes no código

| Spec | Código |
|---|---|
| interruptores | `vaccination` (`requires: record_mode:record`); `rnds_immunization` (`requires: feature:vaccination credential:rnds`); `rnds_mock` (`requires: feature:rnds_immunization`, só development/test, padrão do `sncr_mock`) |
| credencial | `IntegrationCredential` kind `rnds`; segredo `{ "pfx_base64", "pfx_password", "requester_cnes" }` (cifrado; nunca devolvido) |
| sala | `vaccine_rooms` (uma por unidade), com `stock_location_id` de tipo `vaccine_room` |
| rede de frio municipal | `stock_locations.kind` ganha `cold_chain_central` (uma por cidade) e `vaccine_room`; índice único passa a `(health_unit_id, kind)` |
| equipe da sala | `vaccine_room_staff.role`: `vaccinator` \| `room_lead`; `room_lead` exige CBO `2235*`. Vacinador: CBO `3222*`, `2235*`, `2251*`, `2252*`, `2253*` |
| equipe da rede de frio | `stock_location_staff.role = stock_keeper` (do 27a) |
| dose | `immunizations` |
| situação no envio | `rnds_status`: `not_sent` (interruptor de envio desligado) \| `pending` \| `sending` \| `accepted` \| `rejected` \| `failed` \| `blocked` (validação NT 115) |
| motivo de bloqueio | `professional_cns_not_in_cnes` \| `patient_identity_unverified` \| `patient_without_cns_or_cpf` |
| conferência da caderneta | `vaccination_record_checks.mode`: `transcribed` \| `confirmed_complete` \| `born_after_go_live` |
| situação por vacina | `due` \| `overdue` \| `upcoming` \| `complete` \| `not_applicable` \| `unchecked` |
| item do razão | `stock_lots.item_kind`: `medication` \| `immunobiological`; `immunobiological_code` (BRImunobiologico) |
| movimentos novos | `vial_opening` (saída das doses do frasco multidose), `immunization` (saída de 1 dose, só unidose), `technical_loss` (só relatório; nunca move saldo — a dose já saiu na abertura) |
| terminologias | códigos dos CodeSystems da RNDS sem tradução: `immunobiological`, `dose`, `strategy`, `route`, `site`, `manufacturer`, `care_group`, `maternal_condition` |

## 2. Formas

`<immunobiological>`: `{ "code", "name", "multidose": bool, "doses_per_vial", "default_route", "default_site", "manufacturers": [ { "code", "name", "validity_after_opening_hours" } ] }`.

`<immunization>` (nominal; trilha): `{ "id", "immunobiological": { "code", "name" }, "dose": { "code", "name" }, "strategy", "care_group", "route", "site", "maternal_condition"|null, "applied_at", "lot": { "id"|null, "number", "manufacturer": { "code"|null, "name" }, "expires_on"|null }, "transcription": bool, "room": { "id", "name", "health_unit": { "id", "name" } }|null, "by": { "name", "cbo" }, "campaign_id"|null, "status": "active"|"cancelled", "cancel_reason"|null, "replaces_id"|null, "replaced_by_id"|null, "rnds": { "status", "sent_at"|null, "accepted_at"|null, "block_reason"|null, "error"|null } }`. `rnds.error` é texto do RNDS sem dado pessoal (o api corta e limpa).

`<card>` (caderneta; nominal; trilha): `{ "patient": { "id", "display_name", "birth_date", "sex", "has_cpf", "cns_masked"|null }, "checked": { "mode", "at", "by_name" }|null, "calendar_version", "items": [ { "immunobiological": { "code", "name" }, "dose": { "code", "name" }, "status", "due_on"|null, "applied": <immunization-resumo>|null } ], "immunizations": [<immunization>] }`. Sem conferência: `status` nunca é `overdue` (vira `unchecked` quando a idade já passou).

`<vial>`: `{ "id", "lot": { "id", "number", "immunobiological_code" }, "opened_at", "expires_at", "doses_per_vial", "doses_used", "doses_left", "status": "open"|"expired"|"closed_out" }`.

## 3. Administração (`municipal_admin`; escrita com step-up)

- Credencial: `PUT /integrations/rnds` (mecanismo do 16; `check` faz a troca de token no EHR-AUTH e grava `last_check_status`).
- `GET/POST/PATCH /vaccination/admin/rooms[/:id]` — `{ "health_unit_id", "name", "active" }`; cria o local `vaccine_room`.
- `POST/DELETE /vaccination/admin/rooms/:id/staff` — `{ "user_id", "role" }`; recusa `cbo_not_allowed` / `room_lead_requires_nurse`.
- `POST /vaccination/admin/cold_chain_central` — cria o local da rede de frio (uma vez; 409 `already_exists`); equipe pelo `/pharmacy/admin/locations/:id/staff` existente (aceito com `vaccination` ligado, mesmo sem `pharmacy`).
- `GET /vaccination/admin/settings` / `PATCH` — `{ "go_live_on" }` (data do go-live da cidade; define `born_after_go_live`).

## 4. Catálogo e calendário (qualquer usuário com acesso à sala; sem trilha)

- `GET /vaccination/catalog` → `{ "calendar_version", "immunobiologicals": [<immunobiological>], "terminologies": { "dose": [..], "strategy": [..], "route": [..], "site": [..], "care_group": [..], "maternal_condition": [..] } }`.
- Calendário em `config/vaccination/calendar-2026.yml`; catálogo em `config/vaccination/immunobiologicals.yml`; terminologias em `config/vaccination/rnds_terminologies.yml`. Mudança de calendário = arquivo novo com versão; a cidade usa o vigente.

## 5. Sala: caderneta e dose

- `POST /vaccination/rooms/:room_id/card` — `{ "patient_id" }` → `<card>` (trilha; o vacinador da sala lê no contexto da sala). Fora da equipe da sala: 403.
- `POST /vaccination/rooms/:room_id/checks` — `{ "patient_id", "mode": "confirmed_complete" }` → `<card>`. `transcribed` é gravado pelo api quando a transcrição fecha a conferência (`"close_check": true`).
- `POST /vaccination/rooms/:room_id/immunizations` — `{ "patient_id", "immunobiological_code", "dose_code", "strategy", "care_group", "route", "site", "maternal_condition"?, "applied_at", "lot_id", "vial_id"?, "campaign_id"? }` → 201 `<immunization>`. Recusas em `problems[]`: `lot_not_in_room`, `lot_expired`, `lot_blocked` (excursão), `vial_expired`, `vial_empty`, `vial_lot_mismatch`, `applied_at_in_future`, `dose_not_in_calendar`, `interval_too_short` (aviso que exige `"override_reason"` — com motivo, aceita), `age_out_of_range` (idem), `sex_not_allowed`, `maternal_condition_required`. Multidose sem `vial_id` → `vial_required`.
- `POST /vaccination/rooms/:room_id/transcriptions` — `{ "patient_id", "items": [ { "immunobiological_code", "dose_code", "applied_on", "lot_number"?, "manufacturer_name"?, "place"? } ], "close_check": bool }` → 201 `{ "immunizations": [..], "card": <card> }`.
- `POST /vaccination/immunizations/:id/cancel` — `{ "reason", "replacement"?: {forma do POST} }` (step-up; autor ou `room_lead`) → `{ "cancelled": <immunization>, "replacement": <immunization>|null }`.
- `GET /vaccination/rooms/:room_id/pending?before=` → doses com `rnds.status` em `blocked|rejected|failed` ou `pending` há mais de 20 h (sem nome na lista; só `patient_id`, vacina, status e motivo; abrir é `POST …/card`).

## 6. RNDS

- Fila `rnds_outbox` (cidade): uma entrada por dose e por cancelamento; idempotente por `(source, kind)`; payload FHIR cifrado com a chave da cidade e anulado no aceite; guarda `rnds_id`, `status`, `attempts`, `next_attempt_at`, `error`, `deadline_at` (`applied_at`/`created_at` + 24 h).
- `Rnds::DeliverJob`: EachCityJob a cada minuto, só com `rnds_immunization` utilizável; backoff do LEDI; `limits_concurrency`.
- Cancelamento só sai depois do `accepted` do original; enquanto isso fica `pending`.
- Validação NT 115 antes de montar: CNS do vacinador no CNES importado; cidadão com CPF ou CNS e, com `cadsus_lookup` ligado, conferido no CADSUS. Falha → `blocked` com motivo; corrigido o cadastro, o job tenta de novo.
- Alerta (mecanismo do 16): entrada passando de `deadline_at` sem aceite → aviso ao `room_lead` e ao `municipal_admin` (só contagens).
- `rnds_mock`: servidor falso (container `fake-rnds`, padrão do fake-sncr) com token, aceite, recusa 422 por marcador e timeout.
- Perfil RIA-R 2.0: o plano do api baixa o perfil oficial e fixa os ids de StructureDefinition num YAML; enquanto não houver, o spec valida contra o esqueleto documentado e a pendência vai para o gate.

## 7. Estoque de imunobiológicos

Reaproveita as rotas do 27a (`/pharmacy/receipts`, `/openings`, `/positions`, `/lots/:id/movements`, `/transfers`, `/adjustments`), que passam a aceitar locais `cold_chain_central` e `vaccine_room` com `vaccination` ligado (mesmo sem `pharmacy`) e item `{ "immunobiological_code" }` no lugar de `catalog_item_id`. Quem pode:
- rede de frio: `stock_keeper` (entrada, transferência, ajuste);
- sala: qualquer `vaccinator` recebe transferência; ajuste, perda física e inventário só `room_lead`.

Novas:
- `POST /vaccination/rooms/:room_id/vials` — `{ "lot_id" }` → 201 `<vial>`; move `vial_opening` de `doses_per_vial` doses. `expires_at` = aberto + `validity_after_opening_hours` do fabricante do lote. Unidose não abre frasco: a dose aplicada é a saída `immunization` de 1 dose.
- `GET /vaccination/rooms/:room_id/vials?status=open`.
- `POST /vaccination/vials/:id/close_out` — `{ "reason" }` (`room_lead`): frasco descartado antes do vencimento; sobra vira perda técnica.
- `GET /vaccination/rooms/:room_id/monthly_summary?month=YYYY-MM` (e `cold_chain_central`) → por imunobiológico e lote: saldo inicial, entradas, transferências, frascos abertos, doses aplicadas, perda técnica, perda física (por motivo), saldo final. Só contagens.

## 8. Cadeia de frio

- `GET/POST/PATCH /vaccination/cold_chain/equipments[/:id]` — `{ "location_id", "name", "kind": "refrigerator"|"chamber"|"freezer", "min_c": 2, "max_c": 8, "active" }` (`room_lead` / `stock_keeper`).
- `POST /vaccination/cold_chain/equipments/:id/readings` — `{ "read_at", "current_c", "min_c", "max_c" }` (qualquer da equipe). Fora da faixa → 201 com `"excursion_id"` e os lotes do local ficam `blocked`.
- `POST /vaccination/cold_chain/excursions/:id/decide` — `{ "lots": [ { "lot_id", "decision": "use"|"discard", "reason" } ] }` (`room_lead`/`stock_keeper`; step-up); `discard` gera perda física com o limite de aprovação do 27a.
- `GET /vaccination/cold_chain/status?location_id=` → equipamentos com última leitura, leituras do dia (0/1/2), excursões abertas.

## 9. Cidadão (wpda)

- `GET /citizen/vaccinations` → `{ "checked": bool, "items": [ { "immunobiological_name", "dose_name", "applied_on", "place_name"|null, "transcription", "rnds": "sent"|"sending" } ], "upcoming": [ { "immunobiological_name", "dose_name", "due_on" } ] }`. Sem lote, sem vacinador, sem cancelados. `upcoming` só com caderneta conferida (senão, `[]` e o texto pede para levar a caderneta à unidade). Sem polling.

## 10. Linha do tempo, linhas de cuidado, campanhas, vigilância

- Linha do tempo: fonte `vaccination` (fecha api#71): `{ "kind": "vaccination", "at", "title": "Vacina <nome> — <dose>", "summary": null, "detail_ref": { "kind": "immunization", "id" } }` no dashboard; no wpda, `title` igual e sem detalhe.
- Linhas de cuidado: `vaccine:<código>` medido por dose ativa (inclusive transcrição); `vaccine:c2_schedule` pela nota do C2; caderneta não conferida → `unchecked` (exibido "não conferido", fora de "pendente").
- Campanhas: critérios `vaccine_overdue` `{ "immunobiological_codes": [..], "age_min_months"?, "age_max_months"? }` (só conferidas) e `vaccination_record_unchecked` `{ "age_min_months"?, "age_max_months"? }`, ambos relação SQL sobre `vaccination_statuses`/`vaccination_record_checks`; a campanha mostra `vaccinated_count` (destinatários com dose do imunobiológico depois do envio).
- Vigilância: `POST /vaccination/immunizations/:id/esavi` → cria a notificação `esavi` do 25 com `immunization_id` e devolve `{ "notification_id" }`; a tela abre a notificação.

## 11. Painel (sem trilha; só contagens)

`GET /vaccination/dashboard?month=&health_unit_id=` → `{ "doses_by_immunobiological": [..], "c2_coverage_by_team": [..], "rnds": { "pending", "blocked", "rejected", "late" }, "losses": { "technical", "physical" }, "unchecked_cards" }`.

## 12. Maintenance

Só os três interruptores no mecanismo genérico; nenhuma tela nova.

## 13. Eventos (só ids)

`vaccination.immunization_recorded`, `vaccination.immunization_cancelled`, `vaccination.record_checked`, `vaccination.vial_opened`, `vaccination.cold_chain_excursion`, `rnds.immunization_accepted`, `rnds.immunization_rejected`. `Platform.audit`: `rnds.credential_set` (R18).

## 14. Ordem de entrega

api → dashboard → wpda → maintenance. Migração de cidade a partir de `20261102900001`. Worktree `apps/api/.claude/mod29`, branch `feat/mod-29-vaccination-room`; frontends em `.worktrees/<app>-mod29`.

## 15. Acréscimos (revisão dos planos, 2026-11-02)

Planos: api (Tasks 0–21), dashboard (0–16), wpda (0–7), maintenance (0–3). Valem sobre o texto acima quando divergem.

- **V1 — Credencial:** `PUT /integrations/credentials/rnds` e `POST …/rnds/check` (rota real do 16). Corpo `{ "pfx_base64", "pfx_password", "requester_cnes" }`; recusas `invalid_certificate`, `certificate_expired`, `invalid_requester_cnes`; certificado ≤ 100 KB. `IntegrationCredential` valida o segredo por tipo.
- **V2 — Locais da pessoa:** `GET /vaccination/my_locations` → salas e rede de frio da pessoa com o papel em cada; o `stock_keeper` da rede de frio recebe também as salas com `roles: []` (destinos de transferência). `GET /vaccination/admin/rooms` → `{ "rooms", "cold_chain_central" }`.
- **V3 — Busca do cidadão:** `POST /vaccination/rooms/:room_id/person_lookup` na forma do 27a, e duplicados por `/patients/duplicates`. `POST …/card` aceita `{ "patient_id" }` ou `{ "new_patient", "duplicates_reviewed"? }` (recém-nascido sem cadastro; forma do 24a; 409 `duplicate_suspected`).
- **V4 — Razão:** `technical_loss` não é movimento: fica em `vial_openings.technical_loss_doses`, calculado no vencimento ou no descarte. Entra o movimento `immunization_reversal` (+1 ao cancelar dose de unidose). O item de imunobiológico vem com `catalog_item_id`/`catmat_code` nulos, `immunobiological_code`, `name` e `dispensing_unit: "dose"`; o lote ganha `manufacturer_code` (pode ser nulo); `LotBalance` ganha `blocked`. Entrada e inventário inicial de vacina levam `immunobiological_code`, `manufacturer_code` e `package` (doses por frasco × frascos). Transferência aceita `from_location_id` opcional; perda de vacina leva `loss_code` da lista de motivos do SIES.
- **V5 — Avisos com motivo:** `interval_too_short` e `age_out_of_range` são 422 com `"overridable": true` até vir `override_reason` (10–500, cifrado, na dose). Qualquer aviso futuro que aceite motivo vem marcado do mesmo jeito.
- **V6 — Cancelar e substituir:** atômico; problemas da substituição com `path: "replacement.*"`. A dose ganha `can_cancel`, calculado pelo api.
- **V7 — Envio à RNDS:** o cancelamento é um Bundle substituto (`relatesTo.replaces` + `entered-in-error`), a confirmar na homologação; original recusado ou `failed` que for cancelado não gera envio. `failed` só depois de 15 dias da primeira tentativa (prazo da sala sem conectividade); as 24 h continuam como prazo do alerta.
- **V8 — Alerta de 24 h:** segue o mecanismo do 16, que é calculado na leitura: `counts.late` nas pendências da sala e `rnds.late` no painel, com faixa de aviso para `room_lead` e `municipal_admin` quando `late > 0`. Pendências no envelope `{ "items", "counts", "next_before" }`.
- **V9 — Catálogo revisado:** os códigos vêm do dicionário do LEDI 8.7.0; os 16 códigos do C2 estão confirmados na nota metodológica; o resto e o calendário inteiro nascem com `review: true`. `rnds_immunization` ganha o requisito `vaccination_catalog_reviewed`, cobrado só fora de development/test/staging. A equivalência com os CodeSystems da RNDS e os códigos de fabricante (podem ser nulos) são gate de homologação.
- **V10 — Linha do tempo:** a entrada usa a forma real do 20: `occurred_at` e `detail_ref: { "type": "immunization", "id" }`. Nova leitura nominal `GET /vaccination/immunizations/:id` (trilha).
- **V11 — Cidadão (wpda):** `GET /citizen/vaccinations?citizen_id=` (padrão das rotas do cidadão); cadastro só declarado → 403 `not_verified`; `vaccination` em `features` da sessão do cidadão, e desligado → 403 `feature_disabled`; a entrada do histórico vem com `professional_name`, `status`, `summary` e `document` nulos, sem doses canceladas; `unchecked` chega em `/citizen/care`; datas `AAAA-MM-DD`, listas em ordem cronológica, e uma próxima dose com data passada é permitida (sem a palavra "atrasada").
- **V12 — ESAVI:** se já há notificação `esavi` aberta para o paciente, devolve **200** com o `notification_id` existente e `"existing": true` (idempotente; a tela abre a mesma notificação). Vigilância desligada → 403 `feature_disabled`.
- **V13 — Campanhas:** `GET /campaigns/options` ganha `immunobiologicals` só com `vaccination` ligado; a campanha devolve `vaccinated_count`.
- **V14 — Formas:** frascos no envelope `{ "vials" }`; resumo mensal da rede de frio em `/vaccination/cold_chain_central/monthly_summary`; cadeia de frio, pendências e painel nas formas do plano do dashboard (D8–D10, D13); terminologias `{ "code", "name" }`; `applied` da caderneta `{ "id", "applied_at", "transcription" }`. Limites: motivos 10–500, nomes 3–120, lote ≤ 40.
- **V15 — Maintenance:** rótulos "Sala de vacina", "Envio das vacinas à RNDS (Ministério da Saúde)", "RNDS simulada (desenvolvimento)"; o maintenance traduz os códigos de `missing` `vaccination_disabled`, `rnds_immunization_disabled`, `credential_missing:rnds`, `credential_unauthorized:rnds` e `vaccination_catalog_reviewed`; aviso de RNDS simulada como linha `role="note"` na aba, no molde do `fake_lab`.
- **V16 — Semente de dev:** a semente cria um certificado de teste aceito pelo `fake-rnds` (porta 8094) e a credencial `rnds` da cidade de dev, para a prova no navegador não travar.

### Acréscimos da execução (2026-11-02)

- **V17 — Ordem das listas do cidadão (corrige o V11):** `items` em `applied_on` decrescente (o mais recente primeiro); `upcoming` em `due_on` crescente, com datas nulas no fim.
- **V18 — Precedências confirmadas:** o V4 vence a "entrada sem `package`" do D9 do plano do api; o certificado do V1 tem limite de 100 KB **decodificados**; ESAVI repetido é 200 com `existing` (V12), não 409; a busca de pessoa fica na rota da sala (V3).
- **V19 — Fabricante por chave (corrige o V4):** `manufacturers[]` ganha `key` (string estável, obrigatória, p. ex. `butantan`, `serum`), e `code` (BRFabricante) pode ser nulo. Entrada, inventário inicial e lote levam **`manufacturer_key`**, nunca o código; o lote não grava o código. A montagem do RIA-R resolve o código pelo catálogo no momento do envio, então lotes antigos passam a enviar o código real assim que o catálogo o tiver, sem migração. Código nulo no envio: em development/test/staging o `fake-rnds` aceita; fora deles, o requisito `vaccination_catalog_reviewed` (V9) já impede o envio. O dashboard usa `key` como valor do select. Transcrição continua com `manufacturer_name` em texto livre. A troca dos códigos nulos pelos reais é gate de homologação, junto com o BRFabricante.
- **V20 — Grupos de atendimento:** valem os 108 do dicionário LEDI 8.7.0, com os inativos marcados `active: false` (recusados em dose nova, aceitos em leitura). Nomes de fabricante e prazos após aberto que o plano não fixou nascem com `review: true`.
- **V21 — Duplicados na sala (corrige o V3):** `POST /vaccination/rooms/:room_id/patients/duplicates` `{ "full_name", "birth_date", "mother_name"? }`, com o serviço, a forma de resposta e a trilha do 24a (o mesmo padrão do `/pharmacy/patients/duplicates` do 27a). Portão próprio: `vaccination` utilizável e pessoa na equipe da sala; não depende de `territory_registry`. A rota do 24a não muda.
- **V22 — Códigos de recusa da dose:** entram `lot_product_mismatch` (lote de outro imunobiológico; erro clínico que a tela precisa nomear) e `vial_closed_out` (frasco descartado; não confundir com `vial_expired`). Os demais casos sem código próprio reaproveitam `invalid_value`, `insufficient_balance` e `invalid_reason`. Os eventos do módulo levam `vaccine_room_id` (só ids).
- **V23 — Transcrição e dose cancelada antes do envio (estende o V7):** a transcrição recusa `applied_on_in_future` e `applied_before_birth` com `path: "items[i].applied_on"`. Dose cancelada **nunca tentada** na RNDS (sem `first_attempt_at`: `pending` sem tentativa, `blocked`, ou envio desligado) não vai à RNDS, nem o cancelamento dela; a entrada da fila sai das pendências. Timeout conta como tentada (desfecho desconhecido): o cancelamento espera o desfecho do original, como no V7. `counts.late` só some com `rnds_immunization` desligado (credencial quebrada continua contando) e inclui `sending`.
- **V24 — Fechamento da excursão:** a excursão só fecha quando todos os lotes têm conduta aplicada. Novo `decide` pede só os lotes sem conduta; lote com descarte pendente → `adjustment_pending`; descarte recusado devolve o lote para nova conduta (`use` ou novo `discard`); o lote segue bloqueado até a conduta aplicada. Cada lote em `open_excursions[].lots[]` ganha `pending_adjustment_id|null` ("aguardando aprovação do descarte"). `stock_adjustments.loss_code` passa a `varchar(40)` (motivos do SIES de até 21 caracteres), na migração do 29.
- **V25 — Resumo mensal (acrescenta ao D8):** cada linha ganha `vial_doses` (doses saídas por abertura de frasco), `unit_doses` (doses unidose aplicadas) e `adjustment` (inventário, com sinal), mantendo todas as chaves do D8. A conta é `closing = opening + entradas − saídas`; `vials_opened` continua contando frascos. Perda técnica vem dos frascos; perda pendente de aprovação fica de fora.
