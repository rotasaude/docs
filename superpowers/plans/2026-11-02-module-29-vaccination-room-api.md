# Módulo 29 — Sala de vacina (api) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **PRÉ-REQUISITO: módulo 27a em `origin/main` do api** (conferido ao escrever este plano em `7e086d78`): última migração de cidade `20261029900001_add_pharmacy_stock.rb`; `Platform::Features::CATALOG` terminando em `pharmacy`; `IntegrationCredential::KINDS = %w[ledi cadsus]` (CHECK `ck_integration_credentials_kind`) com segredo `{ username, password }`; rotas `PUT /integrations/credentials/:kind` e `POST /integrations/credentials/:kind/check`; `Pharmacy::Ledger.record!(location:, lot:, kind:, quantity:, by:, document:, reason:, package:, at:)` com `DOCUMENTS` de 4 tipos e `ck_stock_movements_document`; `stock_lots.catalog_item_id`/`catmat_code` NOT NULL; `stock_locations` com `ck_stock_locations_kind` (`central unit`), `idx_stock_locations_unit` (único por unidade) e `idx_stock_locations_one_central`; `Pharmacy::TransfersController#create` só a partir de `kind: "central"`; `Pharmacy::RequestAdjustment.call(location:, lot:, kind:, counted_quantity:, quantity:, reason:, by:, settings:, now:)` (perda aplicada na hora); `Timeline::KINDS` terminando em `dispensation`; `CareLines::Catalog::UNTRACKED_SOURCE = /\A(acs_visit|oral_health|vaccine:.+)\z/`; `Campaigns::Criteria::KINDS` com 7 critérios; `Surveillance::CreateNotification.call(patient:, disease:, origin:, unit:, suspected_at:, by:, …)`; `spec/adr_pointers_spec.rb` com `VALID_RANGE = (1..45)`; `config/recurring.yml` com `ledi_deliver`. O módulo 28 roda em paralelo (`feat/mod-28-oral-health`, migração `20261030900001`, ADR 46): **não dependa dele**; os pontos de conflito previsíveis estão no fim de cada task que toca arquivo comum (Task 1, 2, 17).

**Goal:** O lado api do módulo 29 (F-29.1 a F-29.8, ADR 0047): sala de vacina com equipe, catálogo/calendário/terminologias versionados, caderneta conferida com situação materializada, registro/transcrição/cancelamento da dose, fila RNDS com RIA-R 2.0 e validação NT 115, estoque de imunobiológicos no razão do 27a (frascos, perdas, resumo mensal), cadeia de frio, endpoints do cidadão, linha do tempo, linhas de cuidado, campanhas, ESAVI e painel.

**Architecture:** Tudo no banco da cidade numa migração só (`20261102900001`), atrás de `vaccination` (requer `record_mode:record`); o envio é um `EachCityJob` sobre a tabela `rnds_outbox` no padrão do LEDI (claim com SKIP LOCKED, backoff, payload cifrado e anulado no aceite), e o transporte troca o certificado A1 da cidade (mTLS) por token no EHR-AUTH ou fala com o `FakeRnds::App` quando `rnds_mock` está utilizável. O estoque de vacina é o razão do 27a com uma "chave de item" (`catalog_item_id` OU `immunobiological_code`) e dois tipos de local novos; o calendário é Ruby puro sobre YAML, e o resultado por paciente fica em `vaccination_statuses` (cópia recalculável) para os critérios SQL de campanha.

**Tech Stack:** Rails 8.1 (API), PostgreSQL 16 (cidade + plataforma), Solid Queue, `Net::HTTP` + `OpenSSL::PKCS12` (stdlib), RSpec via `bin/test`. **Nenhuma gem nova.**

**Spec:** `docs/superpowers/specs/2026-11-02-module-29-vaccination-room-design.md` e `docs/adr/0047.md` (com as "Decisões do coordenador" e a decisão 3a). Contrato entre apps — **fonte única de formatos**: `docs/superpowers/plans/2026-11-02-module-29-vaccination-room-contracts.md`; toda diferença está em "Divergências propostas ao contrato", no fim. Pesquisa: `docs/pesquisa/2026-11-02-sala-de-vacina-29.md` e `docs/pesquisa/2026-10-05-integracoes/frente-2-identidade-rnds.md`.

## Desvios e precisões

1. **Fontes dos códigos (lidas ao escrever o plano).** Imunobiológico, dose, via, local de aplicação, grupo de atendimento, condição maternal e estratégia vêm do **dicionário do LEDI APS 8.7.0** (`integracao.esusaps.bridge.ufsc.tech/ledi/documentacao/referencias/dicionario.html`, seções `#imunobiologico`, `#dose`, `#viaadministracao-para-imunobiol-gicos`, `#localaplicacao`, `#grupodeatendimento`, `#estrategiavacinacao` — coluna **código RNDS**; condição maternal 1 Nenhuma, 2 Gestante, 3 Puérpera em `dicionario-fv.html`). Os 16 códigos do C2 foram conferidos também na **nota metodológica do C2** (09 HepB, 17 Hib, 22 VIP, 24 SCR, 26 VPC10, 29 penta acelular, 39 DTP/Hib, 42 penta, 43 hexa, 46 DTP, 47 DTPa infantil, 56 SCRV, 58 tetra acelular, 59 VPC13, 106 VPC15, 107 VPC20). Os CodeSystems da RNDS (`BRImunobiologico`, `BRDose`, `BRViaAdministracao`, `BRLocalAplicacao`, `BRGrupoAtendimento`, `BRCondicaoMaternal`, `BRFabricante`) **não abriram** (guia SPA): todo código que não seja os 16 do C2 e as estratégias leva `review: true` no YAML, e a igualdade LEDI = RNDS é **gate de homologação**. `BRFabricante` não foi achado: fabricantes nascem com `code: null` e `review: true`.
2. **Calendário curado sem o PDF lido linha a linha.** O YAML de `calendar-2026.yml` (Task 4) reproduz o esquema da criança do PNI conhecido em 2025–2026 e as entradas de adolescente, adulto e gestante usadas pelas linhas de cuidado; **toda entrada nasce `review: true`** e com `source:` apontando a página do PDF a conferir. Revisão de enfermagem (zerar `review`) é gate de go-live; em produção o envio fica bloqueado enquanto houver `review: true` (Desvio 18).
3. **Chave de item no razão.** `stock_lots`, `stock_movements`, `stock_minimums` e `stock_openings` ganham `item_kind` (`medication` | `immunobiological`, só em `stock_lots`) e `immunobiological_code`; `catalog_item_id`/`catmat_code` viram nulos com `CHECK num_nonnulls(catalog_item_id, immunobiological_code) = 1` (lotes: `item_kind = 'immunobiological'` ⇔ `immunobiological_code IS NOT NULL` ⇔ `catalog_item_id IS NULL AND catmat_code IS NULL`). Índices únicos parciais para o imunobiológico (`(immunobiological_code, lot_number)`, `(stock_location_id, immunobiological_code)` em mínimos e aberturas). Lote de imunobiológico ganha `manufacturer_code` (string 20, nulo) e `dispensing_unit = "dose"`. `Pharmacy::Ledger.record!` copia o código do lote para o movimento.
4. **Locais.** `stock_locations.kind` aceita `cold_chain_central` (um por cidade, índice parcial) e `vaccine_room` (exige `health_unit_id`); o índice `idx_stock_locations_unit` vira `(health_unit_id, kind)`. `ck_stock_locations_unit` passa a `(kind IN ('unit','vaccine_room')) = (health_unit_id IS NOT NULL)`. Vacina só entra em `cold_chain_central`/`vaccine_room`; medicamento nunca entra neles (`item_not_allowed_here`).
5. **Rotas do 27a reaproveitadas (contrato §7).** `Pharmacy::BaseController` troca `before_action :require_pharmacy!` por `require_stock_feature!` (concern `StockGate`): passa se `pharmacy` OU `vaccination` utilizável; cada ação, ao carregar o local, chama `stock_feature_for!(location)` (`central`/`unit` → `pharmacy`; `cold_chain_central`/`vaccine_room` → `vaccination`; 403 `feature_disabled` com a `feature` do local). Rotas só de farmácia (receitas, dispensação, devolução, busca de pessoa, duplicados, receita externa, itens, mínimos, admin/settings) mantêm `require_pharmacy!`. Linha de entrada com `immunobiological_code` no lugar de `catalog_item_id` (e `manufacturer_code` obrigatório, da lista do produto); `POST /pharmacy/transfers` ganha `from_location_id` opcional (ausente = `central`; `cold_chain_central` só para `vaccine_room`). Papéis na sala vêm de `vaccine_room_staff` (Desvio 6); na central de frio, `stock_keeper` de `stock_location_staff`.
6. **Quem é da sala.** `Vaccination::Access.room_roles(user:, room:)` = papéis do vínculo ativo em `vaccine_room_staff` **E** Membership ativa `health_professional` **E** `ProfessionalLink` ativo na unidade da sala com CBO de vacinador (`3222*`, `2235*`, `2251*`, `2252*`, `2253*`); `room_lead` exige o CBO do vínculo `2235*`. Nenhum papel novo. Mapa para as rotas do 27a: `vaccinator` recebe transferência; `room_lead` também faz ajuste, perda, inventário de abertura e decide excursão.
7. **Movimentos novos.** `vial_opening` (−doses do frasco, documento `vial_opening_id`), `immunization` (−1, só unidose, documento `immunization_id`), `immunization_reversal` (+1, devolve a dose de unidose cancelada, documento `immunization_id`). **`technical_loss` não é movimento** (a dose já saiu na abertura): é `vial_openings.technical_loss_doses`, gravado no fechamento ou no vencimento (D3).
8. **Frasco.** `expires_at = opened_at + validity_after_opening_hours` do fabricante do lote (YAML); `doses_used` = imunizações ativas com o `vial_opening_id`; `status` derivado: `closed_out` se fechado; `expired` se `expires_at <= now` (o job grava `technical_loss_doses` e `closed_at`); senão `open`. Dose multidose com frasco vencido → `vial_expired`; com `doses_used == doses_per_vial` → `vial_empty`.
9. **Dose.** `applied_at` com fuso, nunca futura (`applied_at > now + 1.minute` → `applied_at_in_future`). Imutável (trigger); cancelamento grava `cancelled_at`, `cancelled_by_user_id` e `cancel_reason` (cifrado) uma vez. Substituição: a nova dose leva `replaces_id`. Avisos `interval_too_short` e `age_out_of_range` voltam como recusa 422 até vir `override_reason` (10..500, cifrado), que grava a dose.
10. **Caderneta conferida.** `vaccination_record_checks` imutável; `born_after_go_live` é gravado pelo `Vaccination::Statuses.refresh!` (autor nulo, sala nula) quando o paciente nasceu em ou depois de `vaccination_settings.go_live_on`. `transcribed` só pela transcrição com `close_check: true`. Sem conferência: nenhum `overdue` (vira `unchecked`).
11. **Situação materializada.** `vaccination_statuses` (paciente × série): `status`, `next_dose_code`, `due_on`, `doses_applied`, `age_months`, `checked`, `computed_on`. Recalculada na dose, no cancelamento, na transcrição, na conferência e pelo job diário (todos os pacientes com nascimento conhecido, em lotes de 500). `DELETE`+`INSERT` por paciente (não é registro, é cópia). Linhas `not_applicable` não são gravadas.
12. **Fila RNDS.** Uma entrada por dose (`kind: application`) e por cancelamento (`kind: cancellation`), índice único `(immunization_id, kind)`. Nasce `pending` com `payload` nulo; o job valida (NT 115), monta e cifra o payload, envia. `blocked` revalida a cada hora. Cancelamento só é reivindicado quando a entrada `application` do original está `accepted`; original `rejected`/`failed` cancelado → a entrada de cancelamento nem nasce (nada a desfazer na RNDS) e a dose sai das pendências. Desistência (`failed`) depois de **15 dias** da primeira tentativa (prazo "sem conectividade" da Portaria 5.663), não 24 h.
13. **Prazo e alerta.** `deadline_at` = `applied_at + 24h` (aplicação); `created_at + 24h` (transcrição e cancelamento). "Alerta (mecanismo do 16)" = contagem calculada na leitura (como `Ledi::Alert`): `late` = não `accepted` e `deadline_at < now`; sai no envelope das pendências da sala (`counts`) e no painel. Sem notificação ativa (D4).
14. **Transporte.** `GET {auth_url}/api/token` com mTLS (PKCS#12 da credencial) → `access_token` (cache em memória do processo por cidade e `set_at`, 25 min); `POST {ehr_url}/api/fhir/r4/Bundle` com `Content-Type: application/fhir+json`, `X-Authorization-Server: Bearer <token>`, `Authorization: <CNS do vacinador>`. 201 + `Location: …/Bundle/<id>` → `rnds_id`; 400/422 → `rejected` com o `OperationOutcome.issue[].diagnostics` limpo (`Rnds::Errors.sanitize`: corta 200, troca sequências de 11+ dígitos por `***`); 401/403 → credencial `unauthorized`, lote devolvido sem contar tentativa; 5xx/timeout → backoff. URLs da plataforma em `Rails.application.credentials.rnds` (`auth_url`, `ehr_url` homologação; `ehr_url_template` produção com `%{uf}`). O simulado: `FAKE_RNDS_URL` (auth e ehr na mesma base, http, sem mTLS).
15. **Cancelamento na RNDS (não confirmado em normativo).** Novo Bundle com `Immunization.status = "entered-in-error"` e `Composition.relatesTo = [{ code: "replaces", targetIdentifier: { value: <rnds_id do original> } }]`. Gate de homologação (D5).
16. **NT 115.** `professional_cns_not_in_cnes`: o CNS do vacinador não está em `CnesProfessionalBond` do retrato mais recente do IBGE da cidade com o CNES da unidade da sala (o CNS do vínculo é cifrado sem chave de busca: compara em Ruby, carregando só os vínculos do CNES); `patient_without_cns_or_cpf`; `patient_identity_unverified`: com `cadsus_lookup` utilizável, `Cadsus::Client.for(city).lookup(cpf)` nulo. CADSUS indisponível não bloqueia: a entrada volta a `pending` com backoff. Conferência do CADSUS guardada por entrada (`identity_checked_at`), uma vez.
17. **ESAVI.** `disease_notifications` ganha `immunization_id` (FK, nulo). `POST /vaccination/immunizations/:id/esavi` chama `Surveillance::CreateNotification` com `origin: "manual"`, `disease: Surveillance::Diseases.find("esavi")`, unidade da sala da dose, `suspected_at: now`, `by:` quem pede; já aberta → 200 com o `notification_id` existente (o 25 deixa uma aberta por paciente × doença).
18. **Catálogo em revisão.** `rnds_immunization` ganha o requisito `vaccination_catalog_reviewed` (só fora das `SIMULATION_ENVS`): falta `vaccination_catalog_under_review` enquanto qualquer YAML do módulo tiver `review: true`. Em dev/test/staging o requisito não existe (D6).
19. **Exclusão LGPD.** `Citizens::RequestErasure.examined_patients?` ganha `vaccinated?(ids)` = `Immunization.where(patient_id: ids).exists?` (transcrição inclusive). Conferência e situação materializada não retêm.
20. **Trilha.** `Vaccination::Trail.person_viewed!(user:, patient_id:)` = `clinical_record.viewed` com `access: "vaccination"`, `person_ref: "patient:<id>"`, uma por pessoa mostrada (card, check, dose, transcrição, cancelamento, busca de pessoa). Painel, resumo mensal, pendências (sem nome), frio e catálogo: sem trilha.

## Valores fixados por este plano

1. **Interruptores** (depois de `pharmacy` no `CATALOG`):
   - `Entry.new(key: "vaccination", description: "Sala de vacina: caderneta, registro da dose, calendário, estoque de imunobiológicos e cadeia de frio", requires: %w[record_mode:record])`
   - `Entry.new(key: "rnds_immunization", description: "Envio das doses aplicadas à RNDS (RIA-R 2.0)", requires: %w[feature:vaccination credential:rnds vaccination_catalog_reviewed])` — `vaccination_catalog_reviewed` só cobrado fora de `SIMULATION_ENVS` (Desvio 18).
   - `Entry.new(key: "rnds_mock", description: "RNDS simulada (desenvolvimento) — envio sem validade", requires: %w[feature:rnds_immunization], environments: SIMULATION_ENVS)`
2. **Credencial `rnds`:** segredo `{ "pfx_base64", "pfx_password", "requester_cnes" }`; `pfx_base64` até 20 000 caracteres, decodifica e abre com a senha (`OpenSSL::PKCS12.new`), certificado não vencido; `pfx_password` 1..200; `requester_cnes` `/\A\d{7}\z/`. Recusas: `invalid_credential` (forma), `invalid_certificate` (não abre), `certificate_expired`. `Platform.audit("rnds.credential_set", city_id:)` na troca.
3. **`Vaccination::Catalog`:** `LOCATION_KINDS = %w[cold_chain_central vaccine_room]`, `ROOM_ROLES = %w[vaccinator room_lead]`, `VACCINATOR_CBO_PREFIXES = %w[3222 2235 2251 2252 2253]`, `ROOM_LEAD_CBO_PREFIX = "2235"`, `CHECK_MODES = %w[transcribed confirmed_complete born_after_go_live]`, `ITEM_STATUSES = %w[due overdue upcoming complete not_applicable unchecked]`, `RNDS_STATUSES = %w[not_sent pending sending accepted rejected failed blocked]`, `OUTBOX_STATUSES = %w[pending sending accepted rejected failed blocked]`, `BLOCK_REASONS = %w[professional_cns_not_in_cnes patient_identity_unverified patient_without_cns_or_cpf]`, `EQUIPMENT_KINDS = %w[refrigerator chamber freezer]`, `VIAL_STATUSES = %w[open expired closed_out]`, `EXCURSION_DECISIONS = %w[use discard]`, `REASON = (10..500)`, `PENDING_AFTER = 20.hours`, `DEADLINE = 24.hours`, `GIVE_UP_AFTER = 15.days`, `MAX_TRANSCRIPTION_ITEMS = 40`, `TRAIL_ACCESS = "vaccination"`, `DEFAULT_RANGE_C = [2, 8]`, `READINGS_PER_DAY = 2`, `TEXT_MAX = 120`.
4. **YAML** (todos com `version:` e `source:`; Task 4): `config/vaccination/immunobiologicals.yml`, `config/vaccination/rnds_terminologies.yml`, `config/vaccination/calendar-2026.yml`, `config/vaccination/c2_schedule.yml`, `config/vaccination/physical_loss_reasons.yml`; `config/rnds/ria_r_2_0.yml` (Task 10).
5. **Dev:** api do worktree na porta **3051**; worktree `apps/api/.claude/mod29` (com `config/master.key`); branch `feat/mod-29-vaccination-room`; migração de cidade `20261102900001_add_vaccination_room.rb`; sem migração de plataforma (o catálogo de interruptores e de credenciais é código; o CHECK da credencial é da cidade). `fake-rnds` na porta **8094** (`FAKE_RNDS_URL=http://fake-rnds:8094`). Semente (Task 21): Curitiba — "Sala de vacina UBS Jardim das Flores" na `CareLinesCrew::UNIT`, técnica "Vera Vacinadora" (`322205`, CNS no retrato do CNES) e enfermeira "Rita Responsável" (`223505`, `room_lead`); central "Rede de frio municipal" com o almoxarife do 27a; 4 lotes (penta 42, VIP 22, SCR 24, BCG 15); geladeira "Geladeira 1"; `go_live_on` = hoje − 1 ano; credencial `rnds` com PFX autoassinado gerado na semente (`CN=Rota Saude Simulado`); `vaccination`, `rnds_immunization`, `rnds_mock` ligados.

## Global Constraints

- git `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta); nunca `git add -A`; commits em inglês, Conventional Commits por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`; merge, push e tag só com autorização do usuário.
- **Primeira ação da execução:** mover os cards F-29.1..F-29.8 do board #1 para In Progress (`gh project item-edit …`, um comando `gh` por chamada, começando com `gh`). Card vai para Done quando o ÚLTIMO app dele terminar (todos têm dashboard; F-29.1 também maintenance; F-29.7 também wpda); Verified depois da prova no navegador.
- Testes só por `bin/test api -w mod29 <specs>` (a partir de `/Users/eduardovrocha/Development/ioit.solutions/rota-saude`); nunca `rspec` direto nem o Postgres do host; a suíte completa é pedida à sessão "Testes" (Task 21). `spec/support/vaccination_helpers.rb` é arquivo novo de `spec/support` → a suíte completa é obrigatória no fim.
- Banco de cidade: `db/city_migrate/20261102900001_add_vaccination_room.rb`, dump à mão em `db/city_schema.rb` (`define(version: 2026_11_02_900001)`); triggers em `db/city_triggers.sql`, executado no fim do `up`. Irreversível. **O banco de dev só é migrado/semeado na prova no navegador, com autorização do usuário.**
- Specs de requisição com `type: :request`; nunca simule cidade inalcançável com URL real.
- Erros `{ "error" }` (com `path` e `problems: [ { path, error } ]` quando há recusas de campo, todas de uma vez); interruptor 403 `{ "error": "feature_disabled", "feature": "<key>" }`; step-up 401 `mfa_required`; Content-Type JSON em toda escrita por cookie; registro de outro escopo por id → 404; lista com escopo alheio → 403; listas com envelope e `next_before`.
- **CPF, CNS, nome, vacina/lote/dose do paciente, motivos, justificativas, certificado e senha nunca em log, evento, Analytics, erro ou URL.** Modelos novos com dado do paciente sobrescrevem `inspect`/`to_s`/`pretty_print`/`as_json` (só `id`). Filtros de log novos (Task 1): `/\A(immunobiological_code|immunobiological_codes|dose_code|lot_id|lot_number|vial_id|manufacturer_code|manufacturer_name|strategy|care_group|route|site|maternal_condition|applied_at|applied_on|place|items|replacement|override_reason|pfx_base64|pfx_password|requester_cnes|current_c|min_c|max_c)\z/`.
- Eventos declarados em `config/initializers/domain_events.rb` (`to: []`) e em `spec/initializers/domain_events_bindings_spec.rb`, payload só com ids e códigos de lista fechada; `DomainEvents.publish(` é multilinha — confira cada chamada inteira. `Platform.audit("rnds.credential_set")` entra em `R18_PLATFORM_EVENT_NAMES`.
- Nunca `Current.city =` solto; interruptores lidos **fora** da transação da cidade.
- Cifra por cidade: todo `encrypts` novo em `CityEncryption::CITY_KEYED_TARGETS` (guarda `spec/architecture/city_encrypted_attributes_guard_spec.rb`); coluna cifrada de tabela imutável no `rota_reencryption_only(...)`.
- `spec/adr_pointers_spec.rb` sobe para `VALID_RANGE = (1..47)` (o 28 sobe para 46; quem mergear depois resolve para 47).
- Nenhum papel novo (`Membership::ROLES` intacto). Painel: `municipal_admin` e `analyst`.
- A sala nunca exige agendamento nem número mínimo de pessoas para abrir frasco (NT 236/2026): nenhuma regra de abertura além de lote válido, não bloqueado e com saldo.

## Review Focus

1. **Dose de unidose cancelada e substituída pelo mesmo lote** (a técnica errou só a dose, não o lote): o saldo do lote termina igual ao de uma dose só (a reversão devolve 1, a substituta tira 1), e o frasco multidose nunca recebe reversão. Teste na Task 9.
2. **Caderneta não conferida de criança com 1 ano sem nenhuma dose registrada** (o caso mais comum no go-live): nenhum item `overdue`, os passados viram `unchecked`, o critério `vaccine_overdue` não a convoca e `vaccination_record_unchecked` convoca. Testes na Task 5 e na Task 18.
3. **RNDS recusando a dose (422) e a técnica cancelando para corrigir:** a dose original recusada nunca é "desfeita" na RNDS (nenhuma entrada de cancelamento nasce), a substituta vai normalmente e a pendência da original some da lista. Teste na Task 13.
4. **Geladeira fora da faixa no meio do expediente com frasco aberto** (leitura 9,5 °C): os lotes do local ficam bloqueados para nova dose E para abrir frasco, o frasco já aberto também recusa dose (`lot_blocked`), e a decisão `use` libera tudo sem perda. Teste na Task 14.
5. **Paciente sem CPF e com CNS** (paciente do 24a): o envio vai com o CNS como identificador e não bloqueia; sem os dois, `blocked` com `patient_without_cns_or_cpf`, e o cidadão não vê a dose como "enviada". Teste na Task 11.

## Mapa de arquivos

**Criar** (sob `apps/api/.claude/mod29/`)
- `db/city_migrate/20261102900001_add_vaccination_room.rb`.
- `config/vaccination/{immunobiologicals,rnds_terminologies,calendar-2026,c2_schedule,physical_loss_reasons}.yml`; `config/rnds/ria_r_2_0.yml`.
- `app/models/{vaccine_room,vaccine_room_staff,vaccination_setting,vaccination_record_check,vaccination_status,immunization,rnds_outbox_entry,vial_opening,stock_lot_block,cold_chain_equipment,cold_chain_reading,cold_chain_excursion}.rb`.
- `app/services/vaccination/{gate,catalog,immunobiologicals,terminologies,calendar,schedule,statuses,access,card,json,trail,c2,stock_positions,monthly_summary,cold_chain_status,dashboard,citizen_view}.rb`.
- `app/services/rnds/{gate,mock,config,client,bundle,profile,skeleton,validation,delivery,backoff,errors,alert,enqueue}.rb`; `app/services/stock/domain.rb`.
- `app/commands/vaccination/{save_room,link_room_staff,unlink_room_staff,create_cold_chain_central,save_settings,record_check,record_immunization,record_transcription,cancel_immunization,open_vial,close_out_vial,save_equipment,record_reading,decide_excursion,notify_esavi}.rb`.
- `app/controllers/concerns/{vaccination_gate,stock_gate}.rb`; `app/controllers/vaccination/{base,catalog,cards,checks,person_lookups,immunizations,transcriptions,pending,vials,summaries,cold_chain,dashboard}_controller.rb`; `app/controllers/vaccination/admin/{rooms,room_staff,cold_chain_central,settings}_controller.rb`; `app/controllers/citizen_api/vaccinations_controller.rb`.
- `app/jobs/rnds/deliver_job.rb`; `app/jobs/vaccination/{refresh_statuses_job,vial_expiry_job}.rb`.
- `app/services/timeline/sources/vaccination_source.rb`; `app/services/campaigns/criteria/{vaccine_overdue,vaccination_record_unchecked}.rb`.
- `lib/fake_rnds/{app.rb,config.ru}`; `lib/vaccination_crew.rb`.
- `spec/support/vaccination_helpers.rb`; specs listados em cada task.

**Modificar**
- `app/services/platform/features.rb`, `app/models/integration_credential.rb`, `app/commands/integrations/{set_credential,check_connection}.rb`, `app/controllers/integrations_controller.rb`, `app/models/{stock_lot,stock_movement,stock_location,disease_notification,membership}.rb` (só comentário em `membership`), `app/services/pharmacy/{ledger,catalog,access,positions,json,statement}.rb`, `app/commands/pharmacy/{record_entry,send_transfer,request_adjustment}.rb`, `app/controllers/pharmacy/{base,entries,transfers,adjustments,positions,statements}_controller.rb`, `app/services/city_encryption.rb`, `app/services/timeline.rb`, `app/services/timeline/query.rb`, `app/services/care_lines/{catalog,evaluate,sources,preview,rules}.rb` e quem chama `Evaluate.call`, `config/care_lines/catalog-base.yml`, `app/services/campaigns/{criteria,audience_schema,presenter}.rb`, `app/commands/citizens/request_erasure.rb`, `db/city_schema.rb`, `db/city_triggers.sql`, `config/routes.rb`, `config/recurring.yml`, `config/initializers/{domain_events,filter_parameter_logging}.rb`, `spec/adr_pointers_spec.rb`, `spec/events/platform_event_payload_guard_spec.rb`, `spec/initializers/domain_events_bindings_spec.rb`, `spec/requests/integrations_spec.rb`, `spec/services/platform/features_spec.rb`, `db/seeds.rb` (chama a crew), `/Users/eduardovrocha/Development/ioit.solutions/rota-saude/docker-compose.yml` (fora de git: serviço `fake-rnds` e `FAKE_RNDS_URL`).

---

### Task 0: Worktree, cards e conferência do pré-requisito

**Files:** nenhum código.

- [ ] **Step 1:** `gh project item-edit` de F-29.1..F-29.8 para In Progress (um comando `gh` por chamada).
- [ ] **Step 2:** `cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/api && /opt/homebrew/bin/git fetch origin && /opt/homebrew/bin/git worktree add .claude/mod29 -b feat/mod-29-vaccination-room origin/main && cp config/master.key .claude/mod29/config/master.key`.
- [ ] **Step 3:** conferir o PRÉ-REQUISITO (cabeçalho) com `grep` no worktree; qualquer diferença vira nota no topo do plano antes da Task 1.
- [ ] **Step 4:** `bin/test api -w mod29 spec/adr_pointers_spec.rb spec/services/platform/features_spec.rb` → PASS (linha de base).

---

### Task 1: Esquema da cidade, modelos, triggers e helpers

**Files:**
- Create: `db/city_migrate/20261102900001_add_vaccination_room.rb`; os 12 modelos de `app/models/` do mapa; `spec/support/vaccination_helpers.rb`; `spec/models/vaccination_schema_spec.rb`
- Modify: `db/city_schema.rb`, `db/city_triggers.sql`, `app/models/{stock_lot,stock_movement,stock_location,disease_notification}.rb`, `app/services/city_encryption.rb`, `config/initializers/filter_parameter_logging.rb`, `spec/adr_pointers_spec.rb`

**Interfaces:**
- Produces (tabelas): `vaccination_settings` (smallint id=1, `go_live_on date`, `updated_by_user_id`, `updated_at`); `vaccine_rooms` (`health_unit_id` único, `stock_location_id` único NOT NULL, `name` 120, `active`, `updated_by_user_id`, timestamps); `vaccine_room_staff` (`vaccine_room_id`, `user_id`, `role` em `ROOM_ROLES`, `granted_by_user_id`, `granted_at`, `revoked_at`, `revoked_by_user_id`; único parcial ativo `(room, user, role)`); `immunizations` (`patient_id`, `vaccine_room_id` nulo só na transcrição sem sala, `health_unit_id`, `vaccinator_user_id`, `vaccinator_cbo` 6, `immunobiological_code` 10, `dose_code` 10, `strategy` 10, `care_group` 10 nulo na transcrição, `route` 10 nulo na transcrição, `site` 10 nulo na transcrição, `maternal_condition` 10 nulo, `applied_at timestamptz`, `transcription bool`, `stock_lot_id` nulo, `lot_number_text`/`manufacturer_text`/`place_text` cifrados (transcrição), `vial_opening_id` nulo, `campaign_id` nulo, `replaces_id` nulo único, `override_reason` cifrado nulo, `cancelled_at`, `cancelled_by_user_id`, `cancel_reason` cifrado, `calendar_version` 20, `created_at`); `rnds_outbox` (`immunization_id`, `kind` `application|cancellation`, `status` em `OUTBOX_STATUSES`, `payload` cifrado nulo, `rnds_id` 100, `block_reason`, `error` 200, `attempts`, `next_attempt_at`, `first_attempt_at`, `last_attempted_at`, `sent_at`, `accepted_at`, `identity_checked_at`, `deadline_at`, timestamps; único `(immunization_id, kind)`); `vial_openings` (`vaccine_room_id`, `stock_lot_id`, `opened_by_user_id`, `opened_at`, `expires_at`, `doses_per_vial`, `closed_at`, `closed_by_user_id` nulo, `close_reason` cifrado nulo, `technical_loss_doses` nulo); `stock_lot_blocks` (`stock_location_id`, `stock_lot_id`, `cold_chain_excursion_id`, `released_at`, `decision`); `cold_chain_equipments` (`stock_location_id`, `name` 120, `kind`, `min_c`/`max_c numeric(4,1)`, `active`, timestamps); `cold_chain_readings` (`cold_chain_equipment_id`, `read_at`, `current_c`/`min_c`/`max_c numeric(4,1)`, `recorded_by_user_id`, `created_at`); `cold_chain_excursions` (`cold_chain_reading_id` único, `stock_location_id`, `decided_at`, `decided_by_user_id`); `vaccination_record_checks` (`patient_id`, `vaccine_room_id` nulo, `checked_by_user_id` nulo só em `born_after_go_live`, `mode`, `checked_at`); `vaccination_statuses` (`patient_id`, `series` 40, `immunobiological_code`, `next_dose_code`, `status`, `due_on`, `doses_applied`, `age_months`, `checked`, `computed_on`; único `(patient_id, series)`; índice `(status, immunobiological_code)`).
- Produces (colunas): `stock_lots.item_kind` (default `medication`), `immunobiological_code`, `manufacturer_code`; `stock_movements.immunobiological_code`, `vial_opening_id`, `immunization_id`; `stock_minimums.immunobiological_code`; `stock_openings.immunobiological_code`; `stock_adjustments.loss_code` (nulo; obrigatório para lote imunobiológico em `loss`/`expiry` — validado no comando, Task 6); `disease_notifications.immunization_id`.
- Produces (helpers de spec): `vaccination_city!(enabled: true, rnds: false)` (nesta task = `clinical_city!`, sem interruptor: o catálogo só existe na Task 2, que passa a ligar `vaccination` e, com `rnds: true`, `rnds_immunization` + `rnds_mock` + `rnds_credential!`), `self_signed_pfx(password:, not_after:)`, `create_immunization!(room, **over)` e `cold_chain_equipment!(room)` (gravação direta no banco, sem comando), `create_unit` com `cnes:` opcional, `vaccine_room!(unit = create_unit("UBS Vacina #{SecureRandom.hex(2)}"))`, `room_member!(room, role: "vaccinator", cbo: "322205")`, `vaccine_patient!(birth_date:, sex: "female", cpf: true, cns: true)`, `vaccine_lot!(room, code: "42", doses: 10, lot_number: "L#{SecureRandom.hex(3).upcase}", expires_on: Time.zone.today + 180, manufacturer_code: nil)`, `cold_chain_central!`, `rnds_credential!(cnes: "1234567")`.

- [ ] **Step 1: Write the failing test** `spec/models/vaccination_schema_spec.rb`:

```ruby
RSpec.describe "Esquema do módulo 29", type: :model do
  let!(:city) { vaccination_city! }
  let(:room) { vaccine_room! }

  it "lote de imunobiológico sem CATMAT; medicamento sem código de imunobiológico" do
    by = room.updated_by_user
    expect { StockLot.create!(item_kind: "immunobiological", immunobiological_code: "42", lot_number: "P1", expires_on: Date.tomorrow, dispensing_unit: "dose", created_by_user: by) }.not_to raise_error
    expect { StockLot.create!(item_kind: "immunobiological", immunobiological_code: "42", catmat_code: 1, catalog_item_id: SecureRandom.uuid, lot_number: "P2", expires_on: Date.tomorrow, dispensing_unit: "dose", created_by_user: by) }
      .to raise_error(ActiveRecord::StatementInvalid, /ck_stock_lots_item/)
  end

  it "um vaccine_room por unidade, e a unidade aceita unit e vaccine_room juntos" do
    expect(StockLocation.where(health_unit_id: room.health_unit_id).pluck(:kind)).to eq([ "vaccine_room" ])
    expect { StockLocation.create!(kind: "vaccine_room", name: "Outra", health_unit: room.health_unit, updated_by_user: room.updated_by_user) }
      .to raise_error(ActiveRecord::RecordNotUnique)
  end

  it "dose imutável: UPDATE fora do cancelamento e DELETE recusados; cancelar é uma vez" do
    dose = create_immunization!(room)
    expect { dose.update_column(:dose_code, "2") }.to raise_error(ActiveRecord::StatementInvalid, /immunizations/)
    dose.update!(cancelled_at: Time.current, cancelled_by_user: dose.vaccinator_user, cancel_reason: "Lote digitado errado na sala")
    expect { dose.update!(cancelled_at: 1.minute.from_now) }.to raise_error(ActiveRecord::StatementInvalid, /cancelled once/)
    expect { dose.delete }.to raise_error(ActiveRecord::StatementInvalid)
  end

  it "leitura de temperatura e conferência são só acréscimo" do
    reading = ColdChainReading.create!(cold_chain_equipment: cold_chain_equipment!(room), read_at: Time.current, current_c: 5, min_c: 3, max_c: 7, recorded_by_user: room.updated_by_user)
    expect { reading.update_column(:current_c, 4) }.to raise_error(ActiveRecord::StatementInvalid)
  end

  it "payload da fila vai a nulo no aceite, e só no aceite" do
    entry = RndsOutboxEntry.create!(immunization: create_immunization!(room), kind: "application", status: "pending", payload: "{}", next_attempt_at: Time.current, deadline_at: 1.day.from_now)
    expect { entry.update!(status: "accepted", accepted_at: Time.current) }.to raise_error(ActiveRecord::StatementInvalid, /payload/)
  end

  it "nada pessoal em inspect/as_json" do
    dose = create_immunization!(room)
    expect([ dose.inspect, dose.as_json.to_s ].join).not_to include("42")
    expect(dose.as_json).to eq("id" => dose.id)
  end
end
```

(`create_immunization!(room, **over)` e `cold_chain_equipment!(room)` entram nos helpers: dose aplicada direta no banco, sem comando.)

- [ ] **Step 2:** `bin/test api -w mod29 spec/models/vaccination_schema_spec.rb` → FAIL (`vaccination_city!` indefinido).
- [ ] **Step 3: Implementar** a migração (o padrão da do 27a: constantes espelhadas, `replace_check`, `execute File.read("db/city_triggers.sql")` no fim, `down` irreversível). Constraints-chave: `ck_stock_lots_item` (Desvio 3); `ck_stock_movements_item` (`num_nonnulls(catalog_item_id, immunobiological_code) = 1`); `ck_stock_movements_kind` com `vial_opening immunization immunization_reversal`; sinal (`vial_opening`, `immunization` negativos; `immunization_reversal` positivo); `ck_stock_movements_document` com 6 documentos (`vial_opening_id` ⇔ `vial_opening`; `immunization_id` ⇔ `immunization`/`immunization_reversal`); `ck_stock_locations_kind`/`ck_stock_locations_unit` e `idx_stock_locations_unit` → `(health_unit_id, kind)`; `idx_stock_locations_one_cold_chain_central`; `ck_immunizations_transcription` (`transcription` ⇔ `stock_lot_id IS NULL`; aplicada exige `vaccine_room_id`, `care_group`, `route`, `site`); `ck_immunizations_cancel` (os três campos juntos); `ck_rnds_outbox_payload_accepted` (`status <> 'accepted' OR payload IS NULL`); `ck_rnds_outbox_block` (`(status = 'blocked') = (block_reason IS NOT NULL)`); `ck_vaccination_record_checks_author` (`(mode = 'born_after_go_live') = (checked_by_user_id IS NULL)`); `ck_cold_chain_equipments_range` (`min_c < max_c`). Triggers: `rota_immunization_guard()` (só `cancelled_at`, `cancelled_by_user_id`, `cancel_reason` mudam, uma vez; re-cifra sob `rota.reencrypting` para `lot_number_text`, `manufacturer_text`, `place_text`, `override_reason`, `cancel_reason`), `rota_vial_opening_guard()` (só `closed_at`, `closed_by_user_id`, `close_reason`, `technical_loss_doses`, uma vez), `rota_append_only()` em `cold_chain_readings`, `vaccination_record_checks`, `cold_chain_excursions` (exceto `decided_at`/`decided_by_user_id` uma vez — guarda própria), `vaccine_room_staff` no padrão do `rota_stock_location_staff_guard`; `rota_no_delete()` em `vaccine_rooms`, `cold_chain_equipments`, `rnds_outbox`, `stock_lot_blocks`; TRUNCATE recusado em todas. Modelos com `encrypts` (lista acima + `RndsOutboxEntry#payload`) entram em `CITY_KEYED_TARGETS`; `Immunization`, `RndsOutboxEntry`, `VialOpening`, `VaccinationRecordCheck`, `VaccinationStatus` com `inspect`/`as_json` só id. Dump à mão do `db/city_schema.rb`; filtros de log; `VALID_RANGE = (1..47)`.
- [ ] **Step 4:** `bin/test api -w mod29 spec/models/vaccination_schema_spec.rb spec/architecture/city_encrypted_attributes_guard_spec.rb spec/adr_pointers_spec.rb spec/requests/pharmacy` → PASS (o 27a segue verde com as colunas nulas).
- [ ] **Step 5: Commit** `feat: add vaccination room schema, models and immutability triggers` (só os arquivos da task, por nome).

Conflito previsto com o 28: `db/city_schema.rb` (versão do `define`), `db/city_triggers.sql` (bloco novo no fim), `spec/adr_pointers_spec.rb`. Quem mergear depois renumera o `define` para a maior versão.

---

### Task 2: Interruptores e credencial `rnds`

**Files:**
- Create: `app/services/vaccination/{gate,catalog}.rb` (o `Catalog` nasce aqui com as constantes do Valor fixado 3 e `under_review? = false`), `app/services/rnds/{gate,mock}.rb`, `app/controllers/concerns/vaccination_gate.rb`, `spec/services/vaccination/gate_spec.rb`, `spec/commands/integrations/set_rnds_credential_spec.rb`
- Modify: `app/services/platform/features.rb`, `app/models/integration_credential.rb`, `app/commands/integrations/set_credential.rb`, `app/controllers/integrations_controller.rb`, `spec/requests/integrations_spec.rb:67`, `spec/services/platform/features_spec.rb:77-79`, `spec/events/platform_event_payload_guard_spec.rb` (`rnds.credential_set` em `R18_PLATFORM_EVENT_NAMES`)

**Interfaces:**
- Produces: `Vaccination::Gate::KEY = "vaccination"`, `Vaccination::Gate.usable?(city) -> Boolean`; `Rnds::Gate::KEY = "rnds_immunization"`, `.usable?(city)`; `Rnds::Mock::KEY = "rnds_mock"`, `.on?(city, env: Rails.env)`; concern `VaccinationGate#require_vaccination!` (403 `feature_disabled`); `IntegrationCredential::KINDS = %w[ledi cadsus rnds]`, `#pfx_bytes`, `#pfx_password`, `#requester_cnes`; `Integrations::SetCredential.call(kind:, secret:, by:, city:)` (assinatura nova: `secret` é o Hash do corpo; `ledi`/`cadsus` continuam `{ username, password }`).
- `Platform::Features.missing_for` ganha `"vaccination_catalog_reviewed"` → `"vaccination_catalog_under_review"` quando `Vaccination::Catalog.under_review?` (nasce `false` aqui; a Task 4 troca a implementação), cobrado só se `env` não está em `SIMULATION_ENVS`.

- [ ] **Step 1: Write the failing tests:**

```ruby
# spec/services/vaccination/gate_spec.rb
RSpec.describe Vaccination::Gate do
  it "vaccination exige record_mode record; rnds_immunization exige vaccination e credencial; rnds_mock só fora de produção" do
    city = clinical_city!(record_mode: "integrated")
    Platform::Features.set!(city: city, key: "vaccination", enabled: true, maintainer: ledi_maintainer!)
    expect(Platform::Features.missing(city, "vaccination")).to eq([ "record_mode_not_record" ])
    city.update!(record_mode: "record")
    expect(described_class.usable?(city)).to be(true)
    Platform::Features.set!(city: city, key: "rnds_immunization", enabled: true, maintainer: ledi_maintainer!)
    expect(Platform::Features.missing(city, "rnds_immunization")).to eq([ "credential_missing:rnds" ])
    expect(Platform::Features.find("rnds_mock", env: "production")).to be_nil
  end

  it "em produção o catálogo em revisão bloqueia o envio" do
    city = vaccination_city!(rnds: true)
    allow(Vaccination::Catalog).to receive(:under_review?).and_return(true)
    expect(Platform::Features.missing(city, "rnds_immunization", env: "production")).to include("vaccination_catalog_under_review")
    expect(Platform::Features.missing(city, "rnds_immunization", env: "test")).to eq([])
  end
end
```

```ruby
# spec/commands/integrations/set_rnds_credential_spec.rb
RSpec.describe Integrations::SetCredential do
  let!(:city) { vaccination_city! }
  let(:admin) { pharmacy_admin! }
  let(:pfx) { self_signed_pfx(password: "segredo", not_after: 1.year.from_now) } # helper da Task 1

  it "grava o PFX cifrado, nunca devolve, e audita na plataforma só com city_id" do
    result = described_class.call(kind: "rnds", secret: { "pfx_base64" => Base64.strict_encode64(pfx), "pfx_password" => "segredo", "requester_cnes" => "1234567" }, by: admin, city: city)
    expect(result).to be_ok
    expect(IntegrationCredential.find_by(kind: "rnds").requester_cnes).to eq("1234567")
    expect(PlatformEvent.where(name: "rnds.credential_set").last.payload).to eq("city_id" => city.id)
  end

  it "recusa forma, PFX que não abre e certificado vencido" do
    bad = ->(secret) { described_class.call(kind: "rnds", secret: secret, by: admin, city: city).reason }
    expect(bad.call({ "pfx_base64" => "x", "pfx_password" => "segredo" })).to eq(:invalid_credential)
    expect(bad.call({ "pfx_base64" => Base64.strict_encode64("lixo"), "pfx_password" => "s", "requester_cnes" => "1234567" })).to eq(:invalid_certificate)
    old = self_signed_pfx(password: "s", not_after: 1.day.ago)
    expect(bad.call({ "pfx_base64" => Base64.strict_encode64(old), "pfx_password" => "s", "requester_cnes" => "1234567" })).to eq(:certificate_expired)
  end
end
```

E em `spec/requests/integrations_spec.rb:67` o kind desconhecido passa a `"sncr"`; novo exemplo: `PUT /integrations/credentials/rnds` com o corpo do PFX → 200 sem `pfx_base64` nem `pfx_password` no JSON e o log da requisição sem a senha (`capture_log`). `features_spec.rb:77-79` troca `"rnds"` por `"no_such_feature"`.

- [ ] **Step 2:** `bin/test api -w mod29 spec/services/vaccination/gate_spec.rb spec/commands/integrations/set_rnds_credential_spec.rb spec/requests/integrations_spec.rb spec/services/platform/features_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.** `IntegrationCredential#secret_shape` por kind (`SECRET_KEYS = { "ledi" => %w[password username], "cadsus" => %w[password username], "rnds" => %w[pfx_base64 pfx_password requester_cnes] }`, limites do Valor fixado 2); a abertura do PKCS#12 fica em `Integrations::SetCredential` (fora do modelo, `rescue OpenSSL::PKCS12::PKCS12Error` → `invalid_certificate`, nunca a mensagem). O controlador passa `request.request_parameters.slice(*SECRET_KEYS.fetch(kind))`. `Platform.audit` só para `rnds`, depois do commit da cidade. CHECK da cidade já aceita `rnds` (Task 1).
- [ ] **Step 4:** mesmos specs → PASS; `spec/events/platform_event_payload_guard_spec.rb` → PASS.
- [ ] **Step 5: Commit** `feat: add vaccination switches and the rnds credential`.

---

### Task 3: Salas, equipe, central de frio e go-live

**Files:**
- Create: `app/services/vaccination/{access,json}.rb`, `app/commands/vaccination/{save_room,link_room_staff,unlink_room_staff,create_cold_chain_central,save_settings}.rb`, `app/controllers/vaccination/base_controller.rb`, `app/controllers/vaccination/admin/{rooms,room_staff,cold_chain_central,settings}_controller.rb`, `spec/requests/vaccination/admin_spec.rb`, `spec/services/vaccination/access_spec.rb`
- Modify: `config/routes.rb`, `app/controllers/pharmacy/admin/staff_controller.rb` (aceita local `cold_chain_central` com `vaccination` utilizável, papel só `stock_keeper`), `config/initializers/domain_events.rb` + bindings spec (`vaccine_room.saved`, `vaccine_room_staff.linked`, `vaccine_room_staff.revoked`)

**Interfaces:**
- Produces: `Vaccination::Access.room_roles(user:, room:) -> Set<String>`; `.allowed?(user:, room:, roles:) -> Boolean`; `.vaccinator_link(user:, room:) -> ProfessionalLink | nil` (vínculo ativo com CBO de vacinador na unidade); `.rooms_for(user) -> Array<VaccineRoom>`; `Vaccination::Json.room(room) -> { id, name, active, health_unit: { id, name }, location_id }`; rotas `GET/POST/PATCH /vaccination/admin/rooms[/:id]`, `POST/DELETE /vaccination/admin/rooms/:id/staff`, `POST /vaccination/admin/cold_chain_central`, `GET/PATCH /vaccination/admin/settings`; `Vaccination::BaseController` (sessão, `MfaStepUp`, `require_vaccination!`, `render_failure`, `body_param`, `query_only!` — os do `Pharmacy::BaseController`).

- [ ] **Step 1: Write the failing test** (trechos):

```ruby
RSpec.describe "Sala de vacina — administração", type: :request do
  let!(:city) { vaccination_city! }
  let(:admin) { pharmacy_admin!.tap { |u| u.update!(mfa_verified_at: Time.current) } }
  let(:unit) { create_unit("UBS Jardim das Flores") }

  it "cria a sala com o local vaccine_room; segunda sala na unidade → 409 unit_taken" do
    sign_in_as(admin)
    json_post "/vaccination/admin/rooms", health_unit_id: unit.id, name: "Sala de vacina"
    expect(response).to have_http_status(:created)
    expect(StockLocation.find(json_body["location_id"]).kind).to eq("vaccine_room")
    json_post "/vaccination/admin/rooms", health_unit_id: unit.id, name: "Outra"
    expect([ response.status, json_body["error"] ]).to eq([ 409, "unit_taken" ])
  end

  it "equipe: técnico vira vaccinator; room_lead só enfermeiro; CBO fora da lista recusado" do
    room = vaccine_room!(unit)
    tecnica = screener!(unit, cbo: "322205")
    farmaceutico = screener!(unit, cbo: "223405")
    sign_in_as(admin)
    json_post "/vaccination/admin/rooms/#{room.id}/staff", user_id: tecnica.id, role: "room_lead"
    expect([ response.status, json_body["error"] ]).to eq([ 422, "room_lead_requires_nurse" ])
    json_post "/vaccination/admin/rooms/#{room.id}/staff", user_id: farmaceutico.id, role: "vaccinator"
    expect(json_body["error"]).to eq("cbo_not_allowed")
    json_post "/vaccination/admin/rooms/#{room.id}/staff", user_id: tecnica.id, role: "vaccinator"
    expect(response).to have_http_status(:created)
  end

  it "central de frio uma vez; equipe pelo /pharmacy/admin existente mesmo sem pharmacy" do
    sign_in_as(admin)
    json_post "/vaccination/admin/cold_chain_central", name: "Rede de frio municipal"
    central = StockLocation.find_by!(kind: "cold_chain_central")
    json_post "/vaccination/admin/cold_chain_central", name: "De novo"
    expect([ response.status, json_body["error"] ]).to eq([ 409, "already_exists" ])
    keeper = staff_with("frio@cidade.gov.br", "stock_keeper")
    json_post "/pharmacy/admin/locations/#{central.id}/staff", user_id: keeper.id, role: "stock_keeper"
    expect(response).to have_http_status(:created)
  end

  it "go_live_on; vaccination desligado → 403 feature_disabled; escrita sem step-up → 401" do
    sign_in_as(admin)
    json_patch "/vaccination/admin/settings", go_live_on: "2026-11-02"
    expect(json_body).to eq("go_live_on" => "2026-11-02")
    Platform::Features.set!(city: city, key: "vaccination", enabled: false, maintainer: ledi_maintainer!)
    get "/vaccination/admin/rooms"
    expect(json_body).to eq("error" => "feature_disabled", "feature" => "vaccination")
  end
end
```

`spec/services/vaccination/access_spec.rb`: vínculo revogado, Membership `health_professional` revogada ou `ProfessionalLink` encerrado → `room_roles` vazio (cada um sozinho).

- [ ] **Step 2:** `bin/test api -w mod29 spec/requests/vaccination/admin_spec.rb spec/services/vaccination/access_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.** `SaveRoom` cria `StockLocation(kind: "vaccine_room", name: "<nome> — <unidade>")` e a sala na mesma transação; desativar a sala desativa o local (recusa `location_has_stock` se saldo > 0 ou frasco aberto). `LinkRoomStaff` confere o CBO do `ProfessionalLink` ativo do usuário na unidade (`cbo_not_allowed`; sem vínculo → `professional_link_missing`). Recusas: `unit_taken`, `already_linked`, `already_exists` → 409. Escritas com step-up.
- [ ] **Step 4:** mesmos specs + `spec/requests/pharmacy/admin_spec.rb` → PASS.
- [ ] **Step 5: Commit** `feat: add vaccine rooms, room staff and the cold chain central`.

---

### Task 4: Catálogo, terminologias e calendário (YAML)

**Files:**
- Create: `config/vaccination/{immunobiologicals,rnds_terminologies,calendar-2026,c2_schedule,physical_loss_reasons}.yml`, `app/services/vaccination/{immunobiologicals,terminologies,calendar}.rb`, `app/controllers/vaccination/catalog_controller.rb`, `spec/services/vaccination/catalog_files_spec.rb`, `spec/requests/vaccination/catalog_spec.rb`
- Modify: `app/services/vaccination/catalog.rb` (`under_review?` real), `config/routes.rb`

**Interfaces:**
- Produces: `Vaccination::Immunobiologicals.all -> Array<Product>`, `.find(code) -> Product | nil`, `Product = Data.define(:code, :name, :multidose, :doses_per_vial, :default_route, :default_site, :manufacturers, :review)`, `Manufacturer = Data.define(:code, :key, :name, :validity_after_opening_hours, :review)`, `Product#manufacturer(code_or_key)`; `Vaccination::Terminologies.list(kind) -> Array<{ code:, name: }>`, `.valid?(kind, code)`, `.name(kind, code)`, `KINDS = %w[dose strategy route site care_group maternal_condition]`; `Vaccination::Calendar.current -> Calendar::Version(version:, due_window_days:, series:)`, `Series = Data.define(:key, :public, :accepts, :doses, :sex, :repeat_every_days)`, `Dose = Data.define(:dose_code, :min_age_days, :max_age_days, :recommended_age_days, :min_interval_days, :recommended_interval_days, :review, :source)`, `Version#entry(immunobiological_code, dose_code) -> [Series, Dose] | nil`; `Vaccination::Catalog.under_review? -> Boolean` (algum `review: true` nos cinco YAML); `GET /vaccination/catalog` (contrato §4; `immunobiologicals[].manufacturers[].code` pode ser `null`).

- [ ] **Step 1: Escrever os YAML.** Formas (os valores das listas são os do Desvio 1; abaixo, as linhas que decidem):

`rnds_terminologies.yml` — `version: 1`, `source: "LEDI APS 8.7.0, dicionario.html (2026-11-02)"`, cada kind com `review:`: `strategy` (códigos **RNDS** 1 Rotina … 14 Vacinação escolar, `review: false`), `dose` (1..10, 36..42, 57, 58 — os de rotina; `review: true`), `route` (1, 2, 3, 5, 6, 7; `review: true`), `site` (1..22; `review: true`), `care_group` (os 113 do dicionário, `review: true`), `maternal_condition` (1 Nenhuma, 2 Gestante, 3 Puérpera; `review: true`).

`immunobiologicals.yml` — `version: 1`; um produto por código. Os de rotina com frasco (Desvio 1 e pesquisa §5; `review: true` em todos, exceto `name`/`code` dos 16 do C2):

```yaml
- { code: "15", name: "Vacina BCG", multidose: true, doses_per_vial: 10, default_route: "2", default_site: "14",
    manufacturers: [ { key: serum, name: "Serum Institute of India", code: null, validity_after_opening_hours: 6, review: true },
                     { key: fiocruz, name: "Fiocruz", code: null, validity_after_opening_hours: 8, review: true } ], review: true }
- { code: "09", name: "Vacina hepatite B", multidose: true, doses_per_vial: 10, default_route: "3", default_site: "18",
    manufacturers: [ { key: butantan, name: "Instituto Butantan", code: null, validity_after_opening_hours: 240, review: true } ], review: true }
- { code: "42", name: "Vacina penta (DTP/HB/Hib)", multidose: false, doses_per_vial: 1, default_route: "3", default_site: "18", manufacturers: [ … ], review: true }
- { code: "22", name: "Vacina poliomielite inativada (VIP)", multidose: true, doses_per_vial: 10, … validity_after_opening_hours: 672 … }
- { code: "26", name: "Vacina pneumo 10", multidose: true, doses_per_vial: 4, … 672 … }
- { code: "45", name: "Vacina rotavírus", multidose: false, doses_per_vial: 1, default_route: "5", default_site: "1", … }
- { code: "41", name: "Vacina meningo C", multidose: false, … }
- { code: "74", name: "Vacina meningo ACWY", multidose: false, … }
- { code: "14", name: "Vacina febre amarela", multidose: true, doses_per_vial: 5, … 6 … }
- { code: "24", name: "Vacina tríplice viral", multidose: true, doses_per_vial: 10, … 6 … }
- { code: "56", name: "Vacina tetraviral (SCRV)", multidose: false, … }
- { code: "55", name: "Vacina hepatite A infantil", multidose: false, … }
- { code: "46", name: "Vacina DTP", multidose: true, doses_per_vial: 10, … 672 … }
- { code: "34", name: "Vacina varicela", multidose: false, … }
- { code: "33", name: "Vacina influenza trivalente", multidose: true, doses_per_vial: 10, … 168 … }
- { code: "67", name: "Vacina HPV quadrivalente", multidose: false, … }
- { code: "57", name: "Vacina dTpa adulto", multidose: false, … }
- { code: "25", name: "Vacina difteria e tétano adulto", multidose: true, doses_per_vial: 10, … 672 … }
```

Nas linhas abreviadas acima: `manufacturers` = `[ { key: <fabricante público do PNI para o produto>, name:, code: null, validity_after_opening_hours: <o número da linha; 0 em unidose>, review: true } ]`; `default_route`/`default_site` = `"3"`/`"18"` (IM, vasto lateral da coxa direita) salvo indicado; rotavírus oral (`"5"`/`"1"`), SCR/FA/varicela/tetraviral subcutânea (`"6"`/`"14"`), adulto IM deltoide (`"3"`/`"2"`). Inclui também, mais os 16 do C2 que não são de rotina (17, 29, 39, 43, 47, 58, 59, 106, 107) com `routine: false` (aceitos em transcrição, fora do estoque padrão). Unidose: `validity_after_opening_hours: 0` (não abre frasco).

`calendar-2026.yml` — `version: "2026.1"`, `due_window_days: 30`, `source: "Calendário Nacional de Vacinação 2026 (PDFs por ciclo de vida, gov.br/saude … calendario-tecnico)"`; séries (todas `review: true`; idades em dias; `min_interval_days` a partir da dose anterior da série):

| série | public | accepts | doses (dose_code: recomendada / mín / máx / intervalo mín / recomendado) |
|---|---|---|---|
| `bcg` | child | 15 | 9: 0 / 0 / 1825 |
| `hepb_birth` | child | 09 | 8: 0 / 0 / 30 |
| `penta` | child | 42, 43 | 1: 60 / 42 / 2555 · 2: 120 / – / 2555 / 30 / 60 · 3: 180 / – / 2555 / 30 / 60 |
| `vip` | child | 22, 43, 29, 58 | 1: 60 / 42 / 1825 · 2: 120 / 30 / 60 · 3: 180 / 30 / 60 · 6 (1º reforço): 450 / 180 após a 3 |
| `pcv10` | child | 26, 59, 106, 107 | 1: 60 / 42 / 1825 · 2: 120 / 30 / 60 · 6: 365 / 60 |
| `rota` | child | 45 | 1: 60 / 45 / 105 · 2: 120 / 30 / 60, máx 239 |
| `menc` | child | 41 | 1: 90 / 42 / 1825 · 2: 150 / 30 / 60 |
| `menacwy_child` | child | 74 | 6: 365 / 60 após menc 2 |
| `yellow_fever` | child | 14 | 1: 270 / 270 / 1825 · 6: 1460 / 30 |
| `mmr` | child | 24, 56 | 1: 365 / 365 · 2: 450 / 30 (SCRV conta como 2) |
| `hepa` | child | 55 | 9: 450 / 450 / 1825 |
| `dtp_booster` | child | 46, 47 | 6: 450 / 180 após penta 3 · 7: 1460 / 180 |
| `varicella` | child | 34, 56 | 2: 1460 / 30 |
| `hpv` | adolescent | 67 | 9: 3287 / 3287 / 5478, sex: null |
| `dtpa_pregnancy` | pregnant | 57 | 8: gestação ≥ 20 semanas (regra `pregnant_from_week: 20`) |
| `dt_adult` | adult | 25 | 6: `repeat_every_days: 3650` a partir de 2555 dias |
| `influenza` | child/elderly/pregnant | 33 | 8: `repeat_every_days: 365`, child 180..2190, elderly ≥ 21900 |

`c2_schedule.yml` — `version: 1`, `source: "Nota metodológica C2 (SAPS, 19/06/2026), item g e Quadro 05"`, `review: true` (interpretação por componente):

```yaml
min_interval_days: 30
groups:
  - { key: penta, doses: 3, components: [diphtheria, tetanus, pertussis, hepb, hib] }
  - { key: vip, doses: 3, components: [polio] }
  - { key: mmr, doses: 2, components: [measles, mumps, rubella], min_age_days: 365 }
  - { key: pneumo, doses: 2, components: [pneumo] }
codes:
  "09": [hepb]
  "17": [hib]
  "22": [polio]
  "24": [measles, mumps, rubella]
  "26": [pneumo]
  "29": [diphtheria, tetanus, pertussis, polio, hib]
  "39": [diphtheria, tetanus, pertussis, hib]
  "42": [diphtheria, tetanus, pertussis, hepb, hib]
  "43": [diphtheria, tetanus, pertussis, hepb, polio, hib]
  "46": [diphtheria, tetanus, pertussis]
  "47": [diphtheria, tetanus, pertussis]
  "56": [measles, mumps, rubella]
  "58": [diphtheria, tetanus, pertussis, polio]
  "59": [pneumo]
  "106": [pneumo]
  "107": [pneumo]
```

`physical_loss_reasons.yml` — `version: 1`, `source: "NT 77/2025-CGGI/DPNI (motivos do SIES)"`, `review: true`: `broken`, `expired`, `equipment_failure`, `power_outage`, `transport_failure`, `temperature_excursion`, `wrong_procedure`, `other` (com `name` pt-BR).

- [ ] **Step 2: Write the failing tests** `spec/services/vaccination/catalog_files_spec.rb`:

```ruby
RSpec.describe "YAML do módulo 29" do
  it "todo produto tem fabricante; unidose não tem prazo após aberto; códigos únicos" do
    products = Vaccination::Immunobiologicals.all
    expect(products.map(&:code).uniq.size).to eq(products.size)
    expect(products.select(&:multidose)).to all(satisfy { |p| p.manufacturers.all? { |m| m.validity_after_opening_hours.positive? } })
  end

  it "toda dose do calendário existe nas terminologias e todo código aceito existe no catálogo" do
    Vaccination::Calendar.current.series.each do |series|
      expect(series.accepts).to all(satisfy { |code| Vaccination::Immunobiologicals.find(code) })
      expect(series.doses.map(&:dose_code)).to all(satisfy { |code| Vaccination::Terminologies.valid?("dose", code) })
    end
  end

  it "os 16 códigos do C2 estão no catálogo com o nome da nota" do
    expect(Vaccination::Immunobiologicals.find("42").name).to eq("Vacina penta (DTP/HB/Hib)")
    expect(YAML.load_file(Rails.root.join("config/vaccination/c2_schedule.yml"))["codes"].keys)
      .to match_array(%w[09 17 22 24 26 29 39 42 43 46 47 56 58 59 106 107])
  end

  it "toda entrada tem review e source; under_review? enquanto houver review true" do
    expect(Vaccination::Catalog.under_review?).to be(true)
  end
end
```

`spec/requests/vaccination/catalog_spec.rb`: vacinador da sala → 200 com `calendar_version: "2026.1"` e as seis terminologias; usuário sem sala nem central → 403 `missing_role`.

- [ ] **Step 3:** `bin/test api -w mod29 spec/services/vaccination/catalog_files_spec.rb spec/requests/vaccination/catalog_spec.rb` → FAIL.
- [ ] **Step 4: Implementar** os loaders (carregados uma vez, congelados, `YAML.safe_load_file`), a rota e o controlador (`Access.rooms_for(user).any? || stock_keeper` da central).
- [ ] **Step 5:** → PASS. **Commit** `feat: add the curated vaccination catalog, terminologies and 2026 calendar`.

---

### Task 5: Motor do calendário, situação materializada, conferência e caderneta

**Files:**
- Create: `app/services/vaccination/{schedule,statuses,card,trail}.rb`, `app/commands/vaccination/record_check.rb`, `app/controllers/vaccination/{cards,checks,person_lookups}_controller.rb`, `app/jobs/vaccination/refresh_statuses_job.rb`, `spec/services/vaccination/schedule_spec.rb`, `spec/services/vaccination/statuses_spec.rb`, `spec/requests/vaccination/cards_spec.rb`
- Modify: `config/routes.rb`, `config/recurring.yml` (`vaccination_refresh_statuses`: `every day at 3am`), eventos (`vaccination.record_checked`)

**Interfaces:**
- Consumes: `Vaccination::Calendar.current` (Task 4).
- Produces: `Vaccination::Schedule.call(birth_date:, sex:, pregnant_since:, doses:, checked:, today:, calendar:) -> Array<Item>` com `Item = Data.define(:series, :immunobiological_code, :dose_code, :status, :due_on, :applied_id)` e `doses` = `Array<{ id:, code:, dose_code:, on: Date }>` (só ativas); `Vaccination::Statuses.refresh!(patient, today: Time.zone.today) -> void` (grava `born_after_go_live` quando cabe, depois `DELETE`+`INSERT` das linhas não `not_applicable`); `Vaccination::Statuses.checked?(patient_id) -> Boolean`; `Vaccination::Card.call(patient:, today:) -> Hash` (`<card>` do contrato §2); `Vaccination::Trail.person_viewed!(user:, patient_id:)`; `POST /vaccination/rooms/:room_id/card`, `POST /vaccination/rooms/:room_id/checks`, `POST /vaccination/rooms/:room_id/person_lookup`.

Regras do `Schedule` (as que o teste não determina sozinho):
- dose aplicada da série (qualquer código de `accepts`) casa com a dose de mesmo `dose_code`; sem ela, a série anda pela ordem (a 1ª dose aplicada sem código casado conta para a menor dose pendente);
- `due_on = max(nascimento + recommended_age_days, anterior.on + recommended_interval_days)`; dose anterior pendente → `upcoming` com `due_on: nil`;
- idade > `max_age_days` sem a dose → `not_applicable`; série fora do `public` pela idade/sexo → não entra;
- `today < due_on` → `upcoming`; `due_on <= today < due_on + due_window_days` → `due`; depois → `overdue`, ou `unchecked` sem conferência;
- `repeat_every_days`: a última dose + período; nenhuma dose → devida a partir da idade mínima;
- `dtpa_pregnancy` só com `pregnant_since` (início estimado da gestação pela inscrição ativa em `pregnancy_c3`, `anchor_date` da DUM) e a partir de `pregnant_since + 140 dias`.

- [ ] **Step 1: Write the failing tests** `spec/services/vaccination/schedule_spec.rb` (casos do PDF a conferir pela revisão):

```ruby
RSpec.describe Vaccination::Schedule do
  let(:calendar) { Vaccination::Calendar.current }
  let(:born) { Date.new(2026, 1, 10) }
  def run(today:, doses: [], checked: true, sex: "female", pregnant_since: nil)
    described_class.call(birth_date: born, sex: sex, pregnant_since: pregnant_since, doses: doses, checked: checked, today: today, calendar: calendar)
  end
  def item(items, series, dose) = items.find { |i| i.series == series && i.dose_code == dose }

  it "penta 1 devida aos 2 meses; em atraso depois da janela; não conferida vira unchecked" do
    expect(item(run(today: born + 60), "penta", "1")).to have_attributes(status: "due", due_on: born + 60)
    expect(item(run(today: born + 95), "penta", "1").status).to eq("overdue")
    expect(item(run(today: born + 95, checked: false), "penta", "1").status).to eq("unchecked")
  end

  it "penta 2 aprazada pelo intervalo recomendado a partir da dose real" do
    items = run(today: born + 80, doses: [ { id: "a", code: "42", dose_code: "1", on: born + 75 } ])
    expect(item(items, "penta", "1")).to have_attributes(status: "complete", applied_id: "a")
    expect(item(items, "penta", "2")).to have_attributes(status: "upcoming", due_on: born + 135)
  end

  it "hexa transcrita conta para penta e VIP" do
    items = run(today: born + 70, doses: [ { id: "h", code: "43", dose_code: "1", on: born + 61 } ])
    expect([ item(items, "penta", "1").status, item(items, "vip", "1").status ]).to eq(%w[complete complete])
  end

  it "rotavírus 1 depois de 105 dias de vida não se aplica mais" do
    expect(item(run(today: born + 120), "rota", "1").status).to eq("not_applicable")
  end

  it "dTpa da gestante só a partir da 20ª semana e só para gestante" do
    adult = Date.new(1996, 5, 1)
    items = described_class.call(birth_date: adult, sex: "female", pregnant_since: Date.new(2026, 6, 1), doses: [], checked: true,
                                 today: Date.new(2026, 10, 25), calendar: calendar)
    expect(item(items, "dtpa_pregnancy", "8")).to have_attributes(status: "due", due_on: Date.new(2026, 10, 19))
  end
end
```

`statuses_spec.rb`: (a) paciente nascido depois de `go_live_on` → `refresh!` grava a conferência `born_after_go_live` uma vez e linhas `checked: true`; (b) paciente sem nascimento → nenhuma linha; (c) `age_months` = meses completos em `today`. `cards_spec.rb`: vacinador da sala → `<card>` com `checked: null`, nenhum item `overdue` (Review Focus 2: criança de 1 ano sem dose → itens passados `unchecked`), e um `clinical_record.viewed` com `access: "vaccination"`; `POST …/checks` `confirmed_complete` → `checked.mode`; fora da equipe → 403; `patient_id` inexistente → 404; `person_lookup` com CPF e com CNS (forma do 27a, sem CPF nem CNS inteiros).

- [ ] **Step 2:** `bin/test api -w mod29 spec/services/vaccination/schedule_spec.rb spec/services/vaccination/statuses_spec.rb spec/requests/vaccination/cards_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.** `RefreshStatusesJob` (`prepend EachCityJob`; `Patient.in_batches(of: 500)`; só com `Vaccination::Gate.usable?`). `PersonLookupsController` reaproveita os privados do `Pharmacy::PersonLookupsController` extraindo `Patients::LookupByIdentifier.call(cpf:, cns:) -> Result` (mesmas recusas e o `Patients::Resolve`).
- [ ] **Step 4:** → PASS + `spec/requests/pharmacy/lookup_spec.rb` (extração).
- [ ] **Step 5: Commit** `feat: compute the vaccination schedule and serve the checked card`.

---

### Task 6: Razão do 27a generalizado para imunobiológico

**Files:**
- Create: `app/services/stock/domain.rb`, `app/controllers/concerns/stock_gate.rb`, `app/services/vaccination/stock_positions.rb`, `spec/requests/vaccination/stock_spec.rb`
- Modify: `app/services/pharmacy/{ledger,catalog,access,json,statement}.rb`, `app/commands/pharmacy/{record_entry,send_transfer,request_adjustment}.rb`, `app/controllers/pharmacy/{base,entries,transfers,adjustments,positions,statements}_controller.rb`

**Interfaces:**
- Produces: `Stock::Domain.for(location) -> "medication" | "immunobiological"`; `Stock::Domain.feature_for(location) -> "pharmacy" | "vaccination"`; `StockGate#require_stock_feature!`, `#stock_feature_for!(location) -> Boolean` (renderiza 403 e devolve false); `Pharmacy::Ledger::DOCUMENTS` com `VialOpening => :vial_opening` e `Immunization => :immunization`; `Pharmacy::Catalog::MOVEMENT_KINDS` + `vial_opening immunization immunization_reversal` (positivo: `immunization_reversal`; negativos: `vial_opening`, `immunization`); `Pharmacy::Access.roles_at(user:, location:)` devolve, num `vaccine_room`, `{"stock_keeper"}` ao `room_lead` e `{"pharmacy_assistant"}` ao `vaccinator` (mapeamento interno para reaproveitar as listas de papéis do 27a: recebe transferência; ajuste/perda/inventário só `room_lead`); `Pharmacy::Json.lot(lot, catalog:)` para imunobiológico → `{ id, item: { immunobiological_code, name, dispensing_unit: "dose" }, lot_number, expires_on, manufacturer, manufacturer_code }`; `Vaccination::StockPositions.call(location_ids:, limit:, today:, settings:, before:)` com a forma `<position>` do 27a e `item` acima; `RecordEntry` linha `{ immunobiological_code, lot_number, expires_on, manufacturer_code, quantity }` (sem `package`); `RequestAdjustment` com `loss_code:` (obrigatório para lote imunobiológico em `loss`/`expiry`, da `physical_loss_reasons.yml`).
- Recusas novas: `item_not_allowed_here` (vacina em local de medicamento e vice-versa; `path` da linha), `immunobiological_unknown`, `manufacturer_not_in_catalog`, `lot_blocked` (Task 14 preenche os bloqueios; a checagem nasce aqui: transferência e ajuste de lote com `stock_lot_blocks` aberto no local de origem).

- [ ] **Step 1: Write the failing test** (trechos de `spec/requests/vaccination/stock_spec.rb`):

```ruby
RSpec.describe "Estoque de imunobiológicos pelas rotas do 27a", type: :request do
  let!(:city) { vaccination_city! }                       # pharmacy DESLIGADO
  let!(:central) { cold_chain_central! }
  let(:keeper) { pharmacy_staff!(central, "stock_keeper") }
  let(:room) { vaccine_room! }

  def line(**over) = { immunobiological_code: "42", lot_number: "PEN001", expires_on: (Date.current + 200).iso8601, manufacturer_code: nil, quantity: "100" }.merge(over)

  it "entrada na central em doses, transferência para a sala, recebimento pela técnica" do
    sign_in_as(keeper)
    json_post "/pharmacy/receipts", location_id: central.id, document: { kind: "invoice", number: "GUIA-77", supplier: "Regional de Saúde" }, items: [ line ]
    expect(response).to have_http_status(:created)
    lot = StockLot.sole
    expect(lot).to have_attributes(item_kind: "immunobiological", immunobiological_code: "42", catalog_item_id: nil, dispensing_unit: "dose")
    json_post "/pharmacy/transfers", from_location_id: central.id, to_location_id: room.stock_location_id, items: [ { lot_id: lot.id, quantity: "20" } ]
    expect(response).to have_http_status(:created)
    sign_in_as(room_member!(room, role: "vaccinator"))
    json_post "/pharmacy/transfers/#{StockTransfer.sole.id}/receive", items: [ { lot_id: lot.id, quantity_received: "20" } ]
    expect(balance_of(room.stock_location, lot)).to eq(20)
  end

  it "medicamento na central de frio → item_not_allowed_here; vacina na farmácia da unidade → idem" do
    sign_in_as(keeper)
    json_post "/pharmacy/receipts", location_id: central.id, document: { kind: "invoice", number: "N1", supplier: "S" }, items: [ { catalog_item_id: SecureRandom.uuid, lot_number: "X", expires_on: "2027-01-01", quantity: "1" } ]
    expect(json_body["problems"]).to include("path" => "items[0]", "error" => "item_not_allowed_here")
  end

  it "rotas de farmácia seguem 403 com pharmacy desligado; posição da sala sem trilha" do
    sign_in_as(keeper)
    json_post "/pharmacy/dispensations", {}
    expect(json_body).to eq("error" => "feature_disabled", "feature" => "pharmacy")
  end

  it "perda física de vacina exige loss_code da lista; só o room_lead" do
    lot = vaccine_lot!(room, doses: 10)
    sign_in_as(room_member!(room, role: "vaccinator"))
    json_post "/pharmacy/adjustments", location_id: room.stock_location_id, lot_id: lot.id, kind: "loss", quantity: "1", reason: "Frasco caiu no chão da sala"
    expect(response).to have_http_status(:forbidden)
    sign_in_as(room_member!(room, role: "room_lead", cbo: "223505").tap { |u| u.update!(mfa_verified_at: Time.current) })
    json_post "/pharmacy/adjustments", location_id: room.stock_location_id, lot_id: lot.id, kind: "loss", quantity: "1", reason: "Frasco caiu no chão da sala"
    expect(json_body["problems"]).to include("path" => "loss_code", "error" => "required")
  end
end
```

- [ ] **Step 2:** `bin/test api -w mod29 spec/requests/vaccination/stock_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.** Positions: `Pharmacy::PositionsController` despacha para `Vaccination::StockPositions` quando todos os locais pedidos são de vacina; mistura de domínios na mesma lista → 422 `location_id`. `Vaccination::StockPositions` reusa a SQL do 27a com a chave `immunobiological_code` (CTE própria; sem mínimos por enquanto: `minimum: null`).
- [ ] **Step 4:** `bin/test api -w mod29 spec/requests/vaccination/stock_spec.rb spec/requests/pharmacy spec/services/pharmacy spec/commands/pharmacy` → PASS (o 27a inteiro, pela mudança de gate e de ledger).
- [ ] **Step 5: Commit** `feat: carry immunobiologicals on the pharmacy ledger`.

---

### Task 7: Frascos multidose e perda técnica

**Files:**
- Create: `app/commands/vaccination/{open_vial,close_out_vial}.rb`, `app/controllers/vaccination/vials_controller.rb`, `app/jobs/vaccination/vial_expiry_job.rb`, `spec/requests/vaccination/vials_spec.rb`, `spec/jobs/vaccination/vial_expiry_job_spec.rb`
- Modify: `config/routes.rb`, `config/recurring.yml` (`vaccination_vial_expiry`: `every 15 minutes`), eventos (`vaccination.vial_opened`, `vaccination.vial_closed`)

**Interfaces:**
- Produces (helpers de spec): `open_vial!(room, lot:, by: nil)` (pelo comando) e `apply_from_vial!(vial, count:)` (doses gravadas direto no banco com `vial_opening_id`).
- Produces: `Vaccination::OpenVial.call(room:, lot:, by:, now:) -> Result.ok(vial:) | fail(:lot_not_in_room|:lot_expired|:lot_blocked|:not_multidose|:insufficient_balance)`; `VialOpening#status(now = Time.current)`, `#doses_used`, `#doses_left`; `Vaccination::Json.vial(vial, now:)` (contrato §2); `Vaccination::CloseOutVial.call(vial:, reason:, by:, now:)` (`room_lead`; `already_closed` 409); rotas `POST /vaccination/rooms/:room_id/vials`, `GET /vaccination/rooms/:room_id/vials?status=open|expired|closed_out`, `POST /vaccination/vials/:id/close_out`.

- [ ] **Step 1: Write the failing tests:**

```ruby
it "abre frasco: sai doses_per_vial do lote, vence pelo fabricante" do
  lot = vaccine_lot!(room, code: "22", doses: 20, manufacturer_code: nil) # VIP, 10 doses, 672 h
  sign_in_as(vaccinator)
  travel_to(Time.zone.parse("2026-11-03 08:00")) { json_post "/vaccination/rooms/#{room.id}/vials", lot_id: lot.id }
  expect(json_body).to include("doses_per_vial" => 10, "doses_used" => 0, "doses_left" => 10, "status" => "open",
                               "expires_at" => Time.zone.parse("2026-12-01 08:00").iso8601)
  expect(balance_of(room.stock_location, lot)).to eq(10)
end

it "unidose não abre frasco; lote de outra sala → 404" do
  json_post "/vaccination/rooms/#{room.id}/vials", lot_id: vaccine_lot!(room, code: "42").id
  expect(json_body["error"]).to eq("not_multidose")
end

it "vencimento grava a perda técnica com as doses que sobraram" do
  vial = open_vial!(room, lot: vaccine_lot!(room, code: "24", doses: 10)) # SCR, 6 h
  apply_from_vial!(vial, count: 3)
  travel 7.hours do
    Vaccination::VialExpiryJob.perform_now
    expect(vial.reload).to have_attributes(technical_loss_doses: 7)
    expect(vial.status).to eq("expired")
  end
end
```

- [ ] **Step 2:** → FAIL. **Step 3:** implementar (trava: lote → local → frasco, a ordem do `Pharmacy::Ledger`). **Step 4:** → PASS.
- [ ] **Step 5: Commit** `feat: open multidose vials and record technical loss`.

---

### Task 8: Registro da dose aplicada

**Files:**
- Create: `app/commands/vaccination/record_immunization.rb`, `app/services/rnds/enqueue.rb`, `app/controllers/vaccination/immunizations_controller.rb`, `spec/commands/vaccination/record_immunization_spec.rb`, `spec/requests/vaccination/immunizations_spec.rb`
- Modify: `app/services/vaccination/json.rb`, `config/routes.rb`, eventos (`vaccination.immunization_recorded`)

**Interfaces:**
- Consumes: `Vaccination::Access.vaccinator_link` (Task 3), `Calendar` (4), `Schedule`/`Statuses.refresh!` (5), `Pharmacy::Ledger` (6), `VialOpening` (7).
- Produces (helpers de spec): `record!(room, patient, lot: nil, vial: nil, by: nil, **attrs)` (pelo comando; padrão penta D1 com lote unidose novo) e `cancel!(immunization, replacement: false, by: nil)` (Task 9).
- Produces: `Vaccination::RecordImmunization.call(room:, patient:, attrs:, by:, now: Time.current) -> Result.ok(immunization:) | fail(:invalid, details: { problems: [...] })`, `attrs` com chaves String do contrato §5; `Rnds::Enqueue.call(immunization, kind: "application" | "cancellation") -> RndsOutboxEntry | nil` (cria `pending` com `next_attempt_at: now`, `deadline_at` do Desvio 13; idempotente pelo índice único); `Vaccination::Json.immunization(record, entry:, rnds_usable:)` (forma `<immunization>`; `rnds.status` = `not_sent` quando a entrada está `pending` e `rnds_usable` é falso); `POST /vaccination/rooms/:room_id/immunizations` → 201.

Ordem das recusas (todas em `problems`, na ordem dos campos do contrato): terminologias (`invalid_value` por campo), `dose_not_in_calendar` (estratégia `1` com par fora do calendário; outras estratégias aceitam qualquer `dose_code` da terminologia), `applied_at_in_future`, lote (`lot_not_in_room`, `lot_expired` — validade < dia de `applied_at` na cidade, `lot_blocked`), `vial_required`/`vial_lot_mismatch`/`vial_expired`/`vial_empty`, `sex_not_allowed`, `maternal_condition_required` (sexo `female` sem `maternal_condition`), e por último os avisos `interval_too_short`/`age_out_of_range` (só sem `override_reason`). Gravação: trava lote → local → (frasco); unidose grava o movimento `immunization`; `Statuses.refresh!`; `Rnds::Enqueue`; trilha; evento só com `immunization_id`, `room_id`, `user_id`.

- [ ] **Step 1: Write the failing tests** (comando):

```ruby
RSpec.describe Vaccination::RecordImmunization do
  let!(:city) { vaccination_city! }
  let(:room) { vaccine_room! }
  let(:vaccinator) { room_member!(room, role: "vaccinator") }
  let(:child) { vaccine_patient!(birth_date: Date.current - 61, sex: "female") }
  let(:lot) { vaccine_lot!(room, code: "42", doses: 5) }
  def attrs(**over) = { "immunobiological_code" => "42", "dose_code" => "1", "strategy" => "1", "care_group" => "121", "route" => "3", "site" => "18",
                        "maternal_condition" => "1", "applied_at" => Time.current.iso8601, "lot_id" => lot.id }.merge(over.transform_keys(&:to_s))
  def call(**over) = described_class.call(room: room, patient: child, attrs: attrs(**over), by: vaccinator)

  it "grava a dose, tira 1 do lote unidose, materializa a situação e entra na fila" do
    result = call
    dose = result.payload[:immunization]
    expect(dose).to have_attributes(vaccinator_cbo: "322205", transcription: false, calendar_version: "2026.1")
    expect(balance_of(room.stock_location, lot)).to eq(4)
    expect(VaccinationStatus.find_by(patient_id: child.id, series: "penta")).to have_attributes(doses_applied: 1, next_dose_code: "2")
    expect(RndsOutboxEntry.sole).to have_attributes(kind: "application", status: "pending", payload: nil)
  end

  it "junta todas as recusas" do
    other = vaccine_lot!(vaccine_room!, code: "42")
    result = call(applied_at: 1.hour.from_now.iso8601, lot_id: other.id, route: "99")
    expect(result.details[:problems].map { |p| p[:error] }).to eq(%w[invalid_value applied_at_in_future lot_not_in_room])
  end

  it "intervalo curto pede motivo; com motivo grava" do
    call
    expect(call(dose_code: "2").details[:problems]).to eq([ { path: "dose_code", error: "interval_too_short" } ])
    expect(call(dose_code: "2", override_reason: "Viagem da família para fora do país")).to be_ok
  end

  it "multidose sem frasco → vial_required; frasco de outro lote → vial_lot_mismatch" do
    vip = vaccine_lot!(room, code: "22", doses: 20)
    expect(call(immunobiological_code: "22", lot_id: vip.id).details[:problems]).to include({ path: "vial_id", error: "vial_required" })
  end
end
```

Request spec: 201 com a forma `<immunization>` e `rnds.status: "not_sent"` (envio desligado); fora da equipe 403; `patient_id` inexistente 404; nada da vacina no log (`capture_log` sem `"42"` nem o número do lote).

- [ ] **Step 2:** → FAIL. **Step 3:** implementar. **Step 4:** → PASS.
- [ ] **Step 5: Commit** `feat: record the administered dose against the room stock`.

---

### Task 9: Transcrição, cancelamento e pendências da sala

**Files:**
- Create: `app/commands/vaccination/{record_transcription,cancel_immunization}.rb`, `app/controllers/vaccination/{transcriptions,pending}_controller.rb`, `spec/commands/vaccination/{record_transcription,cancel_immunization}_spec.rb`, `spec/requests/vaccination/pending_spec.rb`
- Modify: `app/controllers/vaccination/immunizations_controller.rb` (`cancel`), `config/routes.rb`, eventos (`vaccination.immunization_cancelled`)

**Interfaces:**
- Produces: `Vaccination::RecordTranscription.call(room:, patient:, items:, close_check:, by:, now:) -> Result.ok(immunizations:, card:)` (itens com `immunobiological_code`, `dose_code`, `applied_on` passado, `lot_number?`, `manufacturer_name?`, `place?`; até `MAX_TRANSCRIPTION_ITEMS`; `applied_at` = `applied_on` 12:00 na cidade; estratégia `1`; sem estoque); `Vaccination::CancelImmunization.call(immunization:, reason:, replacement:, by:, now:) -> Result.ok(cancelled:, replacement:)` (autor ou `room_lead` da sala; step-up no controlador; `already_cancelled` 409); `POST /vaccination/immunizations/:id/cancel`; `GET /vaccination/rooms/:room_id/pending?before=&limit=` → `{ "items": [ { "immunization_id", "patient_id", "immunobiological": { code, name }, "rnds": { status, block_reason, error } } ], "counts": { "blocked", "rejected", "failed", "late" }, "next_before" }`.

Regras: cancelamento numa transação: grava os três campos; unidose com lote → movimento `immunization_reversal` (+1) no local da sala; `Rnds::Enqueue(kind: "cancellation")` só se a entrada `application` não está `rejected`/`failed` (Desvio 12); substituição via `RecordImmunization` com `replaces_id`, na mesma transação (falha da substituta desfaz o cancelamento); `Statuses.refresh!`.

- [ ] **Step 1: Write the failing tests** — inclui o Review Focus 1:

```ruby
it "unidose cancelada e substituída pelo mesmo lote: o lote perde uma dose só" do
  dose = record!(room, child, lot: lot)                       # lote com 5
  result = described_class.call(immunization: dose, reason: "Dose registrada como D1 por engano",
                                replacement: replacement_attrs(dose, dose_code: "2"), by: lead)
  expect(result).to be_ok
  expect(balance_of(room.stock_location, lot)).to eq(4)
  expect(result.payload[:replacement].replaces_id).to eq(dose.id)
  expect(RndsOutboxEntry.where(immunization_id: dose.id).pluck(:kind)).to match_array(%w[application cancellation])
end

it "multidose cancelada não devolve dose ao lote; o frasco volta a ter a dose" do
  vial = open_vial!(room, lot: vaccine_lot!(room, code: "22", doses: 10))
  dose = record!(room, child, lot: vial.stock_lot, vial: vial)
  described_class.call(immunization: dose, reason: "Paciente errado na busca", replacement: nil, by: lead)
  expect(StockMovement.where(kind: "immunization_reversal")).to be_empty
  expect(vial.reload.doses_left).to eq(10)
end

it "transcrição com close_check grava a conferência transcribed e libera o atraso" do
  result = Vaccination::RecordTranscription.call(room: room, patient: child, close_check: true, by: vaccinator,
                                                 items: [ { "immunobiological_code" => "15", "dose_code" => "9", "applied_on" => (Date.current - 55).iso8601, "place" => "Maternidade" } ])
  expect(VaccinationRecordCheck.sole.mode).to eq("transcribed")
  expect(result.payload[:immunizations].sole).to have_attributes(transcription: true, stock_lot_id: nil)
end
```

`pending_spec.rb`: dose `pending` há 21 h entra; há 19 h não; `blocked` entra com `block_reason`; sem nome nem CPF no JSON; `counts.late` conta só as que passaram de `deadline_at`; vacinador de outra sala → 403.

- [ ] **Step 2:** → FAIL. **Step 3:** implementar. **Step 4:** → PASS (+ `spec/commands/vaccination/record_immunization_spec.rb`).
- [ ] **Step 5: Commit** `feat: transcribe vaccination cards and cancel doses with replacement`.

---

### Task 10: Montagem do RIA-R 2.0

**Files:**
- Create: `config/rnds/ria_r_2_0.yml`, `app/services/rnds/{profile,bundle,skeleton}.rb`, `spec/services/rnds/bundle_spec.rb`, `spec/fixtures/rnds/ria_r_application.json` (o Bundle esperado de um caso fixo)
- Modify: nenhum

**Interfaces:**
- Produces: `Rnds::Profile.current -> Profile` (`composition_profile`, `immunization_profile`, `code_systems: { immunobiological:, dose:, strategy:, route:, site:, care_group:, maternal_condition:, manufacturer: }`, `extensions: { strategy:, care_group:, maternal_condition:, transcription:, origin_country: }`, `identifiers: { cpf:, cns:, cnes:, practitioner_cns: }`, `confirmed: Boolean`); `Rnds::Bundle.build(immunization:, kind:, original_rnds_id: nil, cnes:, now:) -> Hash` (JSON FHIR, chaves String); `Rnds::Skeleton.problems(bundle) -> Array<String>` (caminhos ausentes).

- [ ] **Step 1: Baixar o perfil.** Tentar o pacote do guia FHIR da RNDS (Portal de Serviços DATASUS → RIA-R 2.0, StructureDefinitions `BRRegistroImunobiologicoAdministrado` e `BRImunobiologicoAdministrado`); se abrir, copiar os canônicos e as URLs de extensão para `config/rnds/ria_r_2_0.yml` com `confirmed: true` e `source:` com a data; se não abrir, gravar os canônicos conhecidos (`http://www.saude.gov.br/fhir/r4/StructureDefinition/BRRegistroImunobiologicoAdministrado-2.0`, `…/BRImunobiologicoAdministrado-2.0`, CodeSystems `http://www.saude.gov.br/fhir/r4/CodeSystem/BRImunobiologico` etc.) com `confirmed: false`, e a pendência fica no gate de homologação (D5).
- [ ] **Step 2: Write the failing test:**

```ruby
RSpec.describe Rnds::Bundle do
  let!(:city) { vaccination_city! }
  let(:dose) { create_immunization!(vaccine_room!, immunobiological_code: "42", dose_code: "1", strategy: "1", care_group: "121", route: "3", site: "18", maternal_condition: "1", applied_at: Time.zone.parse("2026-11-03 09:15")) }

  it "documento com Composition primeiro e Immunization referenciada; códigos sem tradução" do
    bundle = described_class.build(immunization: dose, kind: "application", cnes: "1234567", now: Time.zone.parse("2026-11-03 09:20"))
    expect(bundle["type"]).to eq("document")
    composition, immunization = bundle["entry"].map { |e| e["resource"] }
    expect(composition["resourceType"]).to eq("Composition")
    expect(composition["section"].first["entry"].first["reference"]).to eq(bundle["entry"].last["fullUrl"])
    expect(immunization.dig("vaccineCode", "coding", 0)).to eq("system" => Rnds::Profile.current.code_systems[:immunobiological], "code" => "42")
    expect(immunization["occurrenceDateTime"]).to eq("2026-11-03T09:15:00-03:00")
    expect(immunization["lotNumber"]).to eq(dose.stock_lot.lot_number)
    expect(immunization["primarySource"]).to be(true)
    expect(Rnds::Skeleton.problems(bundle)).to eq([])
  end

  it "transcrição: primarySource false, flag de transcrição, lote opcional" do
    transcribed = create_immunization!(nil, transcription: true, stock_lot: nil, lot_number_text: nil)
    immunization = described_class.build(immunization: transcribed, kind: "application", cnes: "1234567", now: Time.current)["entry"].last["resource"]
    expect(immunization).to include("primarySource" => false)
    expect(immunization).not_to have_key("lotNumber")
  end

  it "paciente sem CPF vai com o CNS; cancelamento substitui o documento aceito" do
    bundle = described_class.build(immunization: dose, kind: "cancellation", original_rnds_id: "abc-123", cnes: "1234567", now: Time.current)
    expect(bundle["entry"].first["resource"]["relatesTo"]).to eq([ { "code" => "replaces", "targetIdentifier" => { "value" => "abc-123" } } ])
    expect(bundle["entry"].last["resource"]["status"]).to eq("entered-in-error")
  end
end
```

E o caso fixo comparado inteiro com `spec/fixtures/rnds/ria_r_application.json` (o executor escreve a fixture a partir do perfil do Step 1; é ela que a homologação confere).

- [ ] **Step 3:** → FAIL. **Step 4:** implementar (o `Skeleton` lista os campos 1..1 da Portaria Conjunta 25/2023, pesquisa §1). **Step 5:** → PASS. **Commit** `feat: build the RIA-R 2.0 immunization bundle`.

---

### Task 11: Validação NT 115

**Files:**
- Create: `app/services/rnds/validation.rb`, `spec/services/rnds/validation_spec.rb`

**Interfaces:**
- Produces: `Rnds::Validation.call(immunization:, entry:, city:) -> :ok | :retry_later | [:blocked, reason]` (`reason` em `BLOCK_REASONS`); grava `entry.identity_checked_at` quando o CADSUS confirma.

- [ ] **Step 1: Write the failing test** — Review Focus 5 e o Desvio 16:

```ruby
RSpec.describe Rnds::Validation do
  let!(:city) { vaccination_city!.tap { cnes_city! } }
  let(:room) { vaccine_room!(create_unit("UBS Vacina", cnes: "1234567")) }
  let(:vaccinator) { room_member!(room, role: "vaccinator") }
  before { cnes_snapshot!(bonds: [ { cnes: "1234567", cbo_code: "322205", cns: vaccinator.professional.cns, cpf: nil, ine: nil } ]) }

  it "paciente só com CNS passa; sem CPF e sem CNS bloqueia" do
    with_cns = record!(room, vaccine_patient!(birth_date: Date.current - 70, cpf: false, cns: true), by: vaccinator)
    expect(described_class.call(immunization: with_cns, entry: with_cns.rnds_entries.first, city: city)).to eq(:ok)
    none = record!(room, vaccine_patient!(birth_date: Date.current - 70, cpf: false, cns: false), by: vaccinator)
    expect(described_class.call(immunization: none, entry: none.rnds_entries.first, city: city)).to eq([ :blocked, "patient_without_cns_or_cpf" ])
  end

  it "vacinador fora do retrato do CNES bloqueia" do
    other = room_member!(room, role: "vaccinator", cbo: "322230")
    dose = record!(room, vaccine_patient!(birth_date: Date.current - 70), by: other)
    expect(described_class.call(immunization: dose, entry: dose.rnds_entries.first, city: city)).to eq([ :blocked, "professional_cns_not_in_cnes" ])
  end

  it "com cadsus_lookup: CPF não achado bloqueia; CADSUS fora do ar só adia" do
    enable_cadsus!(city)
    dose = record!(room, vaccine_patient!(birth_date: Date.current - 70), by: vaccinator)
    allow(Cadsus::Client).to receive(:for).and_return(double(lookup: nil))
    expect(described_class.call(immunization: dose, entry: dose.rnds_entries.first, city: city)).to eq([ :blocked, "patient_identity_unverified" ])
    allow(Cadsus::Client).to receive(:for).and_raise(Cadsus::Unavailable)
    expect(described_class.call(immunization: dose, entry: dose.rnds_entries.first, city: city)).to eq(:retry_later)
  end
end
```

(`create_unit` ganha `cnes:` opcional no helper da Task 1 se ainda não tiver; `enable_cadsus!` = credencial `cadsus` + interruptor.)

- [ ] **Step 2:** → FAIL. **Step 3:** implementar (vínculos do retrato carregados uma vez por chamada do job, memoizados por CNES). **Step 4:** → PASS. **Step 5: Commit** `feat: validate identity and professional before sending to the rnds`.

---

### Task 12: Transporte, servidor falso e teste da credencial

**Files:**
- Create: `app/services/rnds/{config,client,errors}.rb`, `lib/fake_rnds/{app.rb,config.ru}`, `spec/lib/fake_rnds_spec.rb`, `spec/services/rnds/client_spec.rb`
- Modify: `app/commands/integrations/check_connection.rb` (`rnds` → `Rnds::Client.for(city).token!`: `ok` "Token da RNDS emitido" / `unauthorized` "A RNDS recusou o certificado" / `unreachable` "RNDS inalcançável"), `/Users/eduardovrocha/Development/ioit.solutions/rota-saude/docker-compose.yml` (fora de git: serviço `fake-rnds` no padrão do `fake-sncr`, `command: bundle exec puma -b tcp://0.0.0.0:8094 lib/fake_rnds/config.ru`, porta `127.0.0.1:8094:8094`; `FAKE_RNDS_URL: ${FAKE_RNDS_URL:-http://fake-rnds:8094}` no ambiente do api)

**Interfaces:**
- Produces: `Rnds::Config.for_city(city) -> Settings(auth_url:, ehr_url:, simulated:) | nil` (`inspect` sem URL); `Rnds::Client.for(city) -> Client` (lê a credencial dentro de `CityConnection.with`); `Client#token! -> String`; `Client#post_bundle(bundle, practitioner_cns:) -> Response(status: :accepted | :rejected | :unauthorized | :transient, rnds_id:, error:)`; `Rnds::Errors.sanitize(text) -> String`; `FakeRnds::App` (`GET /api/token`, `POST /api/fhir/r4/Bundle`; `lotNumber == "RECUSA"` → 422 `OperationOutcome` com `diagnostics: "Lote inválido para o CPF 12345678901"`; `"TIMEOUT"` → `failures` de teste; `failures` como no `FakeSncr::App`; `log`; `accepted` com os Bundles recebidos).

- [ ] **Step 1: Write the failing tests** (`client_spec.rb` com `stub_request(...).to_rack(FakeRnds::App.new)`):

```ruby
it "token e envio: 201 vira accepted com o id do Location" do
  response = client.post_bundle(bundle, practitioner_cns: "700000000000005")
  expect(response).to have_attributes(status: :accepted, rnds_id: fake.accepted.last[:id])
  expect(fake.log.last).to eq([ "POST", "/api/fhir/r4/Bundle", { "authorization" => "700000000000005", "x-authorization-server" => "Bearer #{fake.issued_tokens.last}" } ])
end

it "422 vira rejected com a mensagem limpa, sem o CPF" do
  bundle["entry"].last["resource"]["lotNumber"] = "RECUSA"
  expect(client.post_bundle(bundle, practitioner_cns: "700000000000005").error).to eq("Lote inválido para o CPF ***")
end

it "5xx e recusa de conexão viram transient; 401 vira unauthorized" do
  fake.failures.push(503, :refused, 401)
  expect(3.times.map { client.post_bundle(bundle, practitioner_cns: "700000000000005").status }).to eq(%i[transient transient unauthorized])
end
```

`spec/lib/fake_rnds_spec.rb` no padrão do `fake_sncr_spec.rb` (rackup sobe; token; recusa por marcador). `spec/requests/integrations_spec.rb`: `POST /integrations/credentials/rnds/check` com o simulado → `last_check_status: "ok"`.

- [ ] **Step 2:** → FAIL. **Step 3:** implementar (`Net::HTTP` com `cert`/`key` do PKCS#12 e `verify_mode` padrão; `open_timeout 5`, `read_timeout 20`; nenhuma mensagem de exceção sai). **Step 4:** → PASS.
- [ ] **Step 5: Commit** `feat: talk to the rnds with the city certificate and add the simulated rnds`.

---

### Task 13: Fila, job de envio e alerta

**Files:**
- Create: `app/services/rnds/{delivery,backoff,alert}.rb`, `app/jobs/rnds/deliver_job.rb`, `spec/jobs/rnds/deliver_job_spec.rb`, `spec/services/rnds/alert_spec.rb`
- Modify: `app/models/rnds_outbox_entry.rb` (`claim!`, `release!`, `release_stale!`, `accept!`, `reject!`, `block!`, `retry_later!` no padrão do `LediOutboxEntry`), `config/recurring.yml` (`rnds_deliver`: `every minute`), eventos (`rnds.immunization_accepted`, `rnds.immunization_rejected`)

**Interfaces:**
- Produces: `RndsOutboxEntry.claim!(limit:)` (só `pending`/`blocked` vencidas; `cancellation` só com o `application` do mesmo `immunization_id` `accepted`); `Rnds::Backoff.wait(attempts)` (1, 2, 4… min até 2 h), `GIVE_UP_AFTER = 15.days`; `Rnds::Delivery.new(city).run(entries) -> :done | :paused`; `Rnds::DeliverJob` (`prepend EachCityJob`, `limits_concurrency to: 1, key: "rnds_deliver", duration: 10.minutes`, só com `Rnds::Gate.usable?`); `Rnds::Alert.counts(scope = RndsOutboxEntry.all, now:) -> { pending:, blocked:, rejected:, failed:, late: }`.

- [ ] **Step 1: Write the failing tests** — inclui Review Focus 3:

```ruby
RSpec.describe Rnds::DeliverJob do
  let!(:city) { vaccination_city!(rnds: true) }   # rnds_immunization + rnds_mock + credencial simulada
  let(:fake) { FakeRnds::App.new }
  before { stub_request(:any, /fake-rnds/).to_rack(fake) }

  it "aceita: guarda o id, anula o payload, publica só ids" do
    dose = record!(room, child)
    described_class.perform_now
    expect(dose.rnds_entries.sole).to have_attributes(status: "accepted", payload: nil, rnds_id: fake.accepted.last[:id])
    expect(DomainEvent.where(name: "rnds.immunization_accepted").last.payload.keys).to match_array(%w[outbox_id immunization_id])
  end

  it "cancelamento espera o aceite do original" do
    dose = record!(room, child)
    cancel!(dose)
    fake.failures.push(503)
    described_class.perform_now
    expect(dose.rnds_entries.pluck(:kind, :status)).to match_array([ %w[application pending], %w[cancellation pending] ])
    travel 2.minutes do
      described_class.perform_now
      described_class.perform_now
    end
    expect(dose.rnds_entries.pluck(:status).uniq).to eq([ "accepted" ])
  end

  it "recusada e cancelada pela técnica: nenhum cancelamento nasce; a substituta vai" do
    dose = record!(room, child, lot: vaccine_lot!(room, code: "42", lot_number: "RECUSA"))
    described_class.perform_now
    expect(dose.rnds_entries.sole.status).to eq("rejected")
    replacement = cancel!(dose, replacement: true)
    described_class.perform_now
    expect(dose.rnds_entries.pluck(:kind)).to eq([ "application" ])
    expect(replacement.rnds_entries.sole.status).to eq("accepted")
    expect(pending_ids(room)).not_to include(dose.id)
  end

  it "bloqueada fica blocked com o motivo e revalida depois de 1 h" do
    dose = record!(room, vaccine_patient!(birth_date: Date.current - 70, cpf: false, cns: false))
    described_class.perform_now
    expect(dose.rnds_entries.sole).to have_attributes(status: "blocked", block_reason: "patient_without_cns_or_cpf", payload: nil)
  end

  it "credencial recusada: lote devolvido sem contar tentativa e credencial marcada" do
    fake.failures.push(401)
    record!(room, child)
    described_class.perform_now
    expect(RndsOutboxEntry.sole).to have_attributes(status: "pending", attempts: 0)
    expect(IntegrationCredential.find_by(kind: "rnds").last_check_status).to eq("unauthorized")
  end

  it "envio desligado: nada sai; a dose fica pending e aparece como not_sent" do
    Platform::Features.set!(city: city, key: "rnds_immunization", enabled: false, maintainer: ledi_maintainer!)
    record!(room, child)
    described_class.perform_now
    expect(fake.log).to be_empty
  end
end
```

`alert_spec.rb`: `late` conta `pending`/`blocked`/`rejected`/`failed` com `deadline_at` passado; `sending` não conta.

- [ ] **Step 2:** → FAIL. **Step 3:** implementar (validação → `Bundle.build` → `payload` cifrado → `post_bundle`; HTTP fora de transação; o `ensure` devolve o que sobrou em `sending`). **Step 4:** → PASS (+ Tasks 8, 9, 11, 12).
- [ ] **Step 5: Commit** `feat: deliver immunizations to the rnds with retries and deadline counts`.

---

### Task 14: Cadeia de frio

**Files:**
- Create: `app/commands/vaccination/{save_equipment,record_reading,decide_excursion}.rb`, `app/services/vaccination/cold_chain_status.rb`, `app/controllers/vaccination/cold_chain_controller.rb`, `spec/requests/vaccination/cold_chain_spec.rb`
- Modify: `config/routes.rb`, eventos (`vaccination.cold_chain_excursion`, `vaccination.cold_chain_excursion_decided`)

**Interfaces:**
- Produces: rotas do contrato §8; `Vaccination::RecordReading.call(equipment:, attrs:, by:, now:) -> Result.ok(reading:, excursion:)` (fora da faixa se `current_c`, `min_c` ou `max_c` < `min_c` do equipamento ou > `max_c`; cria `cold_chain_excursions` e um `stock_lot_blocks` por lote com saldo > 0 no local, e os de frasco aberto); `Vaccination::DecideExcursion.call(excursion:, lots:, by:, settings:, now:)` (`use` libera; `discard` → `Pharmacy::RequestAdjustment` `kind: "loss"`, `loss_code: "temperature_excursion"`, quantidade = saldo, e fecha os frascos abertos do lote com `close_reason`; todos os lotes bloqueados precisam vir; `lots_missing`); `Vaccination::ColdChainStatus.call(location:, today:) -> { equipments: [ { id, name, kind, min_c, max_c, last_reading, readings_today } ], open_excursions: [ { id, read_at, lots: [ { lot_id, immunobiological_code, lot_number } ] } ] }`; `lot_blocked` passa a valer em `RecordImmunization`, `OpenVial`, transferência e ajuste (não em perda por `discard`).

- [ ] **Step 1: Write the failing test** — Review Focus 4:

```ruby
it "9,5 °C com frasco aberto bloqueia dose, frasco novo e transferência; decisão use libera sem perda" do
  vial = open_vial!(room, lot: vip = vaccine_lot!(room, code: "22", doses: 20))
  json_post "/vaccination/cold_chain/equipments/#{fridge.id}/readings", read_at: Time.current.iso8601, current_c: "9.5", min_c: "4.0", max_c: "9.5"
  excursion_id = json_body.fetch("excursion_id")
  expect(record_via_api(vial: vial)["problems"]).to include("path" => "lot_id", "error" => "lot_blocked")
  json_post "/vaccination/rooms/#{room.id}/vials", lot_id: vip.id
  expect(json_body["error"]).to eq("lot_blocked")
  sign_in_as(lead_with_step_up)
  json_post "/vaccination/cold_chain/excursions/#{excursion_id}/decide", lots: [ { lot_id: vip.id, decision: "use", reason: "Termômetro conferido, sem perda" } ]
  expect(response).to have_http_status(:ok)
  expect(record_via_api(vial: vial)).to include("status" => "active")
  expect(StockAdjustment.count).to eq(0)
end

it "discard dá perda física de todo o saldo e fecha os frascos do lote" do
  decide!(excursion_id, vip, "discard")
  expect(StockAdjustment.sole).to have_attributes(kind: "loss", status: "applied", loss_code: "temperature_excursion")
  expect(balance_of(room.stock_location, vip)).to eq(0)
  expect(vial.reload).to have_attributes(technical_loss_doses: 10)
  expect(vial.status).to eq("closed_out")
end

it "status do dia: leituras do dia e excursões abertas, sem dado pessoal" do
  get "/vaccination/cold_chain/status", params: { location_id: room.stock_location_id }
  expect(json_body["equipments"].sole).to include("readings_today" => 1)
  expect(json_body["open_excursions"].sole["lots"].sole).to include("lot_id" => vip.id, "immunobiological_code" => "22")
  expect(response.body).not_to include(child.full_name.to_s)
end
```

- [ ] **Step 2:** → FAIL. **Step 3:** implementar (`room_lead`/`stock_keeper` editam equipamento e decidem; qualquer da equipe lê e registra). **Step 4:** → PASS (+ Tasks 6–9).
- [ ] **Step 5: Commit** `feat: log cold chain temperatures and block lots on excursion`.

---

### Task 15: Resumo mensal

**Files:**
- Create: `app/services/vaccination/monthly_summary.rb`, `app/controllers/vaccination/summaries_controller.rb`, `spec/services/vaccination/monthly_summary_spec.rb`
- Modify: `config/routes.rb` (`GET /vaccination/rooms/:room_id/monthly_summary`, `GET /vaccination/cold_chain_central/monthly_summary`)

**Interfaces:**
- Produces: `Vaccination::MonthlySummary.call(location:, month: "YYYY-MM") -> { month:, location:, rows: [ { immunobiological: { code, name }, lot: { id, number, expires_on }, opening_balance, received, transferred_in, transferred_out, vials_opened, doses_applied, technical_loss, physical_loss: { "<loss_code>" => n }, closing_balance } ] }` (inteiros em doses; mês no fuso da cidade; `doses_applied` = imunizações ativas não transcritas da sala no mês; `technical_loss` = soma dos frascos fechados no mês). Sem trilha.

- [ ] **Step 1: Write the failing test:** cenário de um mês com entrada 100, transferência 20 para a sala, 2 frascos VIP abertos, 13 doses, um frasco vencido com 7 sobrando, perda física `broken` 1 → linha da sala `{ opening_balance: 0, transferred_in: 20, vials_opened: 2, doses_applied: 13, technical_loss: 7, physical_loss: { "broken" => 10 }, closing_balance: 0 }` (perda de um frasco fechado = 10 doses) e `closing_balance == opening + entradas − saídas` em todas as linhas; mês inválido → 422 `month`.
- [ ] **Step 2–4:** FAIL → implementar (SQL agregada por `stock_lot_id`) → PASS.
- [ ] **Step 5: Commit** `feat: summarize monthly immunobiological movement for SIES and SIPNI`.

---

### Task 16: Caderneta do cidadão e linha do tempo

**Files:**
- Create: `app/services/vaccination/citizen_view.rb`, `app/controllers/citizen_api/vaccinations_controller.rb`, `app/services/timeline/sources/vaccination_source.rb`, `spec/requests/citizen_api/vaccinations_spec.rb`, `spec/services/timeline/sources/vaccination_source_spec.rb`
- Modify: `app/services/timeline.rb` (`KINDS` + `vaccination`), `app/services/timeline/query.rb` (`SOURCES`), `config/routes.rb` (`get "vaccinations", to: "vaccinations#index"` no escopo `/citizen`)

**Interfaces:**
- Produces: `GET /citizen/vaccinations?citizen_id=` (par validado do celular da sessão, padrão do `RegulationRequestsController`; `vaccination` utilizável) → forma do contrato §9; `rnds: "sent"` só com a entrada `application` `accepted`, senão `"sending"` (bloqueada e recusada inclusive); `upcoming` = itens `due`/`upcoming`/`overdue` com `due_on`, só com caderneta conferida; trilha `access: "citizen"` quando há paciente. `Timeline::Sources::VaccinationSource.entries(person:, before:, limit:, from:, to:)` → `Entry(kind: "vaccination", occurred_at: applied_at, title: "Vacina <nome> — <dose>", summary: nil, status: "applied", detail_ref: { type: "immunization", id: }, citizen: { title:, summary: nil, status: "applied", status_label: "Aplicada", professional_name: nil })`; canceladas fora; não consulta o interruptor (registro retido, como a dispensação).

- [ ] **Step 1: Write the failing tests:** cidadão validado vê dose aplicada e transcrita, sem lote/vacinador/cancelada; caderneta não conferida → `upcoming: []` e `checked: false`; par não validado → 403 `not_verified`; `citizen_id` de outra sessão → 404; linha do tempo do profissional traz `kind: "vaccination"` com o título do contrato e a do cidadão também; `Timeline::Query.citizen_kinds(verified: true, documents: false)` inclui `vaccination`.
- [ ] **Step 2–4:** FAIL → implementar → PASS (+ `spec/services/timeline`, `spec/requests/citizen_api/history_spec.rb`).
- [ ] **Step 5: Commit** `feat: show vaccinations to the citizen and on the patient timeline`.

---

### Task 17: Linhas de cuidado medem vacina

**Files:**
- Create: `app/services/vaccination/c2.rb`, `spec/services/vaccination/c2_spec.rb`, `spec/services/care_lines/vaccine_practices_spec.rb`
- Modify: `app/services/care_lines/{catalog,evaluate,sources,preview,rules}.rb` e cada chamador de `CareLines::Evaluate.call`, `config/care_lines/catalog-base.yml` (conferir 57, 33, 67 contra o catálogo do Desvio 1: dTpa adulto, influenza trivalente, HPV quadrivalente — iguais; anotar a conferência no comentário do YAML)

**Interfaces:**
- Produces: `CareLines::Catalog::UNTRACKED_SOURCE = /\A(acs_visit|oral_health)\z/`; `CareLines::Catalog.untracked?(source, vaccination: false)` (`vaccine:*` untracked só com `vaccination: false`); `CareLines::Catalog::STATUSES` + `unchecked`; `CareLines::Evaluate.call(practices:, person:, anchors:, events:, today:, vaccination: false)` — prática `vaccine:*` com `vaccination: true` e sem o marcador `CareLines::Sources::VACCINE_CHECKED` nos eventos → `PracticeStatus(tracked: false, status: "unchecked")`; `Sources.load` carrega `vaccine:<code>` (datas das imunizações ativas, transcrição inclusive) e `vaccine:c2_schedule` (uma data: o dia em que `Vaccination::C2.met_on` fechou) e o marcador quando há conferência; `Vaccination::C2.met_on(doses:, birth_date:) -> Date | nil` (por componente, intervalo mínimo 30 dias, SCR só a partir de 365 dias de vida).
- Chamadores passam `vaccination: Vaccination::Gate.usable?(Current.city)`, lido fora de transação.

- [ ] **Step 1: Write the failing tests:**

```ruby
RSpec.describe Vaccination::C2 do
  let(:born) { Date.new(2025, 1, 1) }
  def dose(code, on) = { code: code, on: on }

  it "penta por componente com hexa no meio; SCR antes de 12 meses não conta" do
    doses = [ dose("42", born + 60), dose("43", born + 120), dose("42", born + 180),
              dose("22", born + 61), dose("22", born + 121),
              dose("26", born + 60), dose("26", born + 120),
              dose("24", born + 300), dose("24", born + 370), dose("56", born + 450) ]
    expect(described_class.met_on(doses: doses, birth_date: born)).to eq(born + 450)
  end

  it "intervalo menor que 30 dias não conta a segunda dose" do
    doses = [ dose("26", born + 60), dose("26", born + 80) ]
    expect(described_class.met_on(doses: doses, birth_date: born)).to be_nil
  end
end
```

`vaccine_practices_spec.rb`: com `vaccination: false` a prática `c3_f` segue `untracked`; com `vaccination: true` e sem conferência → `unchecked` e fora do `worst`; conferida e com dTpa 57 na janela → `ok`; transcrição conta.

- [ ] **Step 2–4:** FAIL → implementar → PASS (+ `spec/services/care_lines`, `spec/requests/care_lines`, `spec/requests/citizen_api/care_spec.rb`).
- [ ] **Step 5: Commit** `feat: measure vaccine milestones in care lines`.

Conflito previsto com o 28: `CareLines::Catalog::UNTRACKED_SOURCE` (o 28 pode tirar `oral_health`); resolver deixando só o que nenhum dos dois mede.

---

### Task 18: Critérios de campanha e convocados vacinados

**Files:**
- Create: `app/services/campaigns/criteria/{vaccine_overdue,vaccination_record_unchecked}.rb`, `spec/services/campaigns/vaccination_criteria_spec.rb`
- Modify: `app/services/campaigns/{criteria,audience_schema,presenter}.rb`

**Interfaces:**
- Produces: `AudienceSchema::CRITERIA["vaccine_overdue"] = { required: %w[immunobiological_codes], optional: %w[age_min_months age_max_months] }`, `["vaccination_record_unchecked"] = { required: [], optional: %w[age_min_months age_max_months] }`; validação: códigos do catálogo (até 20, sem repetição), idades inteiras 0..1200 e `min <= max` (`inverted_range`); `Criteria::VaccineOverdue.relation(params)` = `Citizen.where(patient_id: VaccinationStatus.where(status: "overdue", checked: true, immunobiological_code: codes)[.where(age_months: min..max)].select(:patient_id)).select(:id)`; `VaccinationRecordUnchecked` = cidadãos cujo paciente tem linha em `vaccination_statuses` com `checked: false` (na faixa); `Campaigns::Presenter.full` ganha `vaccinated_count` (destinatários cujo paciente tem imunização ativa de um dos códigos com `applied_at >= campaign_recipients.created_at`; `null` sem critério `vaccine_overdue`).

- [ ] **Step 1: Write the failing test** — Review Focus 2 (lado campanha): criança de 1 ano sem conferência não entra em `vaccine_overdue` e entra em `vaccination_record_unchecked`; depois da conferência `confirmed_complete` + `refresh!`, entra em `vaccine_overdue` pela penta; faixa `age_min_months: 24` a exclui; `vaccinated_count` passa de 0 para 1 depois da dose registrada após o envio; esquema recusa `immunobiological_codes: ["999"]` com `/clinical/all/0/immunobiological_codes/0` `invalid_value`.
- [ ] **Step 2–4:** FAIL → implementar → PASS (+ `spec/services/campaigns`, `spec/requests/campaigns_spec.rb`).
- [ ] **Step 5: Commit** `feat: target overdue and unchecked vaccination cards in campaigns`.

---

### Task 19: ESAVI a partir da dose

**Files:**
- Create: `app/commands/vaccination/notify_esavi.rb`, `spec/requests/vaccination/esavi_spec.rb`
- Modify: `app/controllers/vaccination/immunizations_controller.rb` (`esavi`), `app/commands/surveillance/create_notification.rb` (aceita `immunization:` opcional e grava `immunization_id`), `config/routes.rb`

**Interfaces:**
- Produces: `Vaccination::NotifyEsavi.call(immunization:, by:, now:) -> Result.ok(notification_id:, created: Boolean)`; `POST /vaccination/immunizations/:id/esavi` → 201 `{ "notification_id" }` (nova) ou 200 (já aberta); `surveillance` desligado → 403 `feature_disabled` com `feature: "surveillance"`; dose cancelada → 409 `immunization_cancelled`. Quem pode: equipe da sala da dose ou papel `surveillance`.

- [ ] **Step 1: Write the failing test:** cria a notificação `esavi` com `immunization_id`, `origin: "manual"`, unidade da sala; segunda chamada → 200 com o mesmo id; nenhum evento novo com vacina/lote (o `disease_notification.created` do 25 já existe, só ids).
- [ ] **Step 2–4:** FAIL → implementar → PASS (+ `spec/commands/surveillance/create_notification_spec.rb`).
- [ ] **Step 5: Commit** `feat: open an ESAVI notification bound to the dose`.

---

### Task 20: Painel de vacinação e exclusão LGPD

**Files:**
- Create: `app/services/vaccination/dashboard.rb`, `app/controllers/vaccination/dashboard_controller.rb`, `spec/requests/vaccination/dashboard_spec.rb`
- Modify: `app/commands/citizens/request_erasure.rb` (`vaccinated?`), `spec/commands/citizens/request_erasure_spec.rb`, `config/routes.rb`

**Interfaces:**
- Produces: `GET /vaccination/dashboard?month=YYYY-MM&health_unit_id=` (`municipal_admin`, `analyst`) → `{ "doses_by_immunobiological": [ { code, name, room_id, room_name, count } ], "c2_coverage_by_team": [ { team_id, team_name, children, complete } ], "rnds": { pending, blocked, rejected, late }, "losses": { technical, physical }, "unchecked_cards" }`; `c2_coverage_by_team`: crianças de 12 a 23 meses da população da equipe (`CareLines::TeamPopulation.members(team:)`), `complete` = `Vaccination::C2.met_on` não nulo; `unchecked_cards` = pacientes com linha `checked: false`. Supressão de contagem pequena como no painel do 23 (mesma regra; reaproveitar o helper de lá).

- [ ] **Step 1: Write the failing test:** contagens de um cenário com duas salas; `health_unit_id` filtra; vacinador → 403; nenhum nome/CPF no JSON; pedido de exclusão de quem tem dose (transcrita inclusive) → retido.
- [ ] **Step 2–4:** FAIL → implementar → PASS.
- [ ] **Step 5: Commit** `feat: add the vaccination panel and retain doses on erasure`.

---

### Task 21: Semente de dev, guarda de eventos e suíte completa

**Files:**
- Create: `lib/vaccination_crew.rb`, `spec/lib/vaccination_crew_spec.rb`
- Modify: `db/seeds.rb`

**Interfaces:**
- Consumes: tudo acima. Produces: `VaccinationCrew.seed!(city)` idempotente (Valores fixados 5), PFX autoassinado gerado na hora (`OpenSSL::PKey::RSA.new(2048)`, `OpenSSL::PKCS12.create`), CNS da técnica incluído num retrato do CNES de dev.

- [ ] **Step 1:** spec da crew: duas rodadas → mesma contagem de salas, lotes, equipamentos; `rnds_mock` utilizável; uma dose de exemplo de criança conferida, aceita pela RNDS simulada quando o job roda com o `FakeRnds::App` em `to_rack`.
- [ ] **Step 2–4:** FAIL → implementar → PASS.
- [ ] **Step 5:** `bin/test api -w mod29 spec/initializers/domain_events_bindings_spec.rb spec/events spec/architecture spec/adr_pointers_spec.rb spec/config` → PASS; `rg -n "DomainEvents.publish\(" app/commands/vaccination app/services/rnds app/models/rnds_outbox_entry.rb -A4` conferindo cada chamada inteira (só ids e códigos).
- [ ] **Step 6: Commit** `chore: seed a dev vaccination room with the simulated rnds`.
- [ ] **Step 7:** pedir à sessão "Testes" (`SendMessage`): `api · mod29/feat/mod-29-vaccination-room · --full · módulo 29 (migração de cidade, spec/support, gates de estoque do 27a)`. Falha → diagnóstico nesta sessão, correção com teste, nova rodada.
- [ ] **Step 8:** prova no navegador só com autorização do usuário (migrar e semear o dev; subir `fake-rnds`); depois disso as tasks do dashboard. Gates de go-live a registrar no card (board #2): homologação RNDS (perfil RIA-R 2.0, cancelamento por substituição, cabeçalhos e validade do token, e-CNPJ × e-CPF, CNES como chave), códigos RNDS × LEDI (dose, via, local, grupo, condição maternal, fabricante), revisão de enfermagem do calendário e dos prazos após abertura, motivos de perda do SIES.

---

## Divergências propostas ao contrato

- **D1 — Rota da credencial.** O contrato escreve `PUT /integrations/rnds`; a rota do 16 é `PUT /integrations/credentials/:kind` (e `POST …/:kind/check`). Proposta: `PUT /integrations/credentials/rnds` com o corpo `{ pfx_base64, pfx_password, requester_cnes }`.
- **D2 — Busca de pessoa na sala.** O contrato começa em `POST …/card { patient_id }` e não diz como a sala acha o `patient_id`. Proposta: `POST /vaccination/rooms/:room_id/person_lookup` `{ cpf }` ou `{ cns }` → `{ patient_id, display_name, has_cpf, cns_masked }`, recusas e trilha do `POST /pharmacy/person_lookup`.
- **D3 — `technical_loss` não é movimento.** O §1 lista `technical_loss` entre os "movimentos novos"; como não move saldo (e o CHECK recusa quantidade 0), fica em `vial_openings.technical_loss_doses` e no resumo mensal. Entra um movimento que o contrato não tem: `immunization_reversal` (+1 dose ao cancelar dose de unidose com lote), senão a substituição consome duas doses de um lote.
- **D4 — Alerta de 24 h.** "Mecanismo do 16" é contagem calculada na leitura (como `Ledi::Alert`), não aviso ativo: `counts.late` no envelope de `GET …/pending` (que ganha `counts` e `items`) e `rnds.late` no painel. Proposta: registrar isso no §6.
- **D5 — Cancelamento na RNDS e perfil.** Cancelamento = Bundle substituto (`relatesTo.replaces` + `Immunization.status = entered-in-error`), não confirmado; e o original `rejected`/`failed` cancelado não gera entrada de cancelamento. Ambos vão ao gate de homologação.
- **D6 — Requisito novo de `rnds_immunization`.** Fora de dev/test/staging, `vaccination_catalog_reviewed` (falta `vaccination_catalog_under_review` enquanto houver `review: true` nos YAML). O maintenance mostra o motivo pelo mecanismo genérico.
- **D7 — Detalhe da linha do tempo.** O 20 usa `detail_ref: { "type", "id" }`, não `{ "kind", "id" }`. Proposta: `{ "type": "immunization", "id" }`.
- **D8 — `GET /citizen/vaccinations?citizen_id=`.** As rotas do cidadão desde o 20 levam `citizen_id` do par validado (sessão com vários pares). Proposta: query `citizen_id` obrigatória, 404/403 `not_verified` como em "Meus encaminhamentos".
- **D9 — Forma do lote de imunobiológico nas rotas do 27a.** O contrato não fixa: `lot.item = { immunobiological_code, name, dispensing_unit: "dose" }`, `lot.manufacturer_code` (pode ser `null` enquanto o `BRFabricante` não for importado), linha de entrada sem `package`, e `POST /pharmacy/transfers` com `from_location_id` opcional; perda/vencimento de vacina com `loss_code` da `physical_loss_reasons.yml`.
- **D10 — Avisos com motivo.** `interval_too_short`/`age_out_of_range` são recusas 422 em `problems` até vir `override_reason` (10..500); o motivo fica cifrado na dose e não volta na forma `<immunization>`.
- **D11 — ESAVI já aberto.** O 25 permite uma notificação aberta por paciente × doença: segunda chamada devolve 200 com o `notification_id` existente (não cria outra).
- **D12 — Desistência do envio em 15 dias.** `failed` só depois de 15 dias da primeira tentativa (prazo da sala sem conectividade), não nas 24 h do LEDI; o prazo de 24 h continua sendo `deadline_at` (alerta).
- **D13 — Códigos.** Os códigos de terminologia são os do dicionário do LEDI APS (estratégia na coluna RNDS); a equivalência com os CodeSystems da RNDS fica no gate. `immunobiologicals[].manufacturers[].code` pode ser `null`.
