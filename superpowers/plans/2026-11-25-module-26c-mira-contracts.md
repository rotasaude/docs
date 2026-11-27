# Módulo 26c — Contratos entre apps

**Fonte única** do que o api, o dashboard e o maintenance combinam no 26c. As linhas do §10 prevalecem sobre o texto das seções anteriores. Os planos de cada repositório (`-api.md`, `-dashboard.md`, `-maintenance.md`) seguem este documento. Uma divergência vai para a seção "Divergências propostas ao contrato" do plano, e a decisão volta aqui como Q-n.

- **Spec:** `docs/superpowers/specs/2026-11-25-module-26c-mira-design.md` (aprovada em 2026-11-25; decisões 1–18).
- **Pesquisa:** `docs/pesquisa/2026-11-23-mira-regulacao-rnds-26c.md` (IG RIRA `br.gov.saude.rira.fhir#1.0.0`; achados 1–15).
- **Pacote RIRA (conferido na Task 1, api 4ec88cda):** o `package.json` traz `version: "1.0.0"`. O IG foi refeito em 2026-11-25, e o SHA-256 do `package.tgz` passou a `2f63d06588cce209e6c26f5ab7caae93ef09fc617efb54f1eb97d2e5d450c860`. As cardinalidades citadas nos planos continuam valendo. O fixture versionado no api guarda o nome `br.gov.saude.rira.fhir-1.0.0-release.tgz`, e o job `rira-validate` carrega o pacote por ele.
- **Base:** 26a (ADR 0044) e RNDS do 29 (ADR 0047). Nada deles muda, salvo o que este documento diz.

**Convenções herdadas:**
- erros `{ "error" }` com `path`;
- step-up `401 mfa_required`;
- `403 feature_disabled` com `feature`;
- listas `{ items, next_before }`;
- `expected_version` → 409 `<entidade>_changed`;
- segredo e dado do paciente (CPF, CNS, CID, justificativa) nunca em log, URL, evento ou resposta de lista.

**Bases:** origin/main de cada repo no dia da Task 0. O 26c é executado depois do 27b.

**Valores fixos:**

| Item | Valor |
|---|---|
| Porta do api | 3063 |
| Vite do dashboard | 5224 |
| maintenance | 5177, com o serviço do compose parado |
| Migrações da cidade | as seguintes às do 27b (confira na Task 0) |
| Branch | `feat/mod-26c-mira` |
| Worktrees | `apps/api/.claude/mod26c` (com `master.key`) e `rota-saude/.worktrees/<repo>-mod26c` |
| Simulado | `lib/fake_rnds` ampliado (porta 8094) |
| Testes | `bin/test api -w mod26c …`, `bin/test <app> -w <repo>-mod26c …`; suíte completa pela sessão "Testes" |

**Mudança transversal.** O 26c toca:
- migração de cidade e `CITY_KEYED_TARGETS`;
- `Platform::Features`;
- `IntegrationCredential`;
- a regulação do 26a (eventos, ofertas, prestadores);
- o `fake_rnds` e o interruptor `rnds_mock` do 29;
- a CI do api, com um job novo.

A suíte completa do api é obrigatória.

## 1. Interruptor e credencial

**Interruptor** (fim do `CATALOG`): `rnds_regulation`, rótulo "Envio da regulação à RNDS", com `requires: %w[feature:care_regulation credential:rnds_regulation ibge_code uf]`. Entra em `CITIZEN_HIDDEN`.
- `Regulation::RndsGate.usable?(city)` = utilizável.
- O `rnds_mock` passa a exigir `rnds_immunization` **ou** `rnds_regulation` (motivo `rnds_disabled`). Com ele ligado, os dois envios vão ao simulado.

**Credencial** `rnds_regulation` (novo `kind`):
- `SECRET_KEYS`: `pfx_base64`, `pfx_password`, `requester_cnes`, `requester_id` (o identificador do solicitante do DATASUS, de 1 a 20 dígitos pela Q13, usado no system `BRRNDS-{requester_id}`) e `responsible_cns` (15 dígitos).
- Gravada pelo `municipal_admin` em `PUT /integrations/credentials/rnds_regulation`, com step-up, no formato da `rnds` e com os dois campos a mais. O teste de conexão pede um token novo.
- Auditoria: `Platform.audit("rnds_regulation.credential_set")`, nome novo no R18.

**Cliente:** o `Rnds::Client` passa a receber a credencial pelo `kind` (`Rnds::Client.for(city, kind:)`). As URLs e o mTLS são os mesmos. O cache de token separa por `kind`. O `Authorization` leva o `responsible_cns` da credencial.

## 2. Estado por pedido

**Tabela nova da cidade `regulation_rnds_syncs`** (1:1, único por `regulation_request_id`):
- `bundle_identifier` (uuid, fixo);
- `last_rnds_id`, `last_accepted_at`, `sent_status`, `sent_digest` (SHA-256 do conteúdo canônico enviado);
- `desired_status`, `desired_digest`, `pending_steps` (jsonb: lista ordenada de `{ status, at }`, só para preservar a falta);
- `status`: `idle` | `queued` | `sending` | `blocked` | `rejected` | `failed` | `excluded` | `erased`;
- `blocked_reasons` (text[]), `last_error_codes` (text[]);
- `attempts`, `next_attempt_at`, `initial_load` (bool), `enqueued_at`.

**Payload:** a última remessa enviada fica em `regulation_rnds_deliveries` (`regulation_rnds_sync_id`, `kind` `create`|`replace`|`delete`, `payload` cifrado, `rnds_id`, `http_status`, `error_codes`, `sent_at`). O payload é apagado 30 dias depois do aceite.

**Correspondência de situação** (`Regulation::RndsStatus.for(request)`):

| Situação no 26a | Situação no MIRA |
|---|---|
| `draft` | não vai |
| `pending`, `authorized` | `pending` |
| `returned` | `returned-to-requester` |
| `scheduled` | `booked` |
| `completed` | `attended` |
| `denied`, `cancelled` | `cancelled` |
| `closed` pela segunda falta | `absence` |
| `closed` por outro motivo | `cancelled` |
| `external` | não vai, salvo a configuração dos externos ligada: espera → `pending`, com `scheduled_at` → `booked`, com resultado → `attended` |

**Passos** (decisão 12): a primeira falta, que é o andamento `missed` sem mudança de situação no pedido, empurra `absence` para `pending_steps`, antes do `pending`. Os outros passos intermediários se juntam no último.

**Gatilhos** (`Regulation::RndsSyncJob`, `IdempotentConsumer`). Ele se liga a:
- os 11 `regulation_request.*`;
- `care_offer.changed` e `care_provider.changed`;
- `appointment.*` (os do 17);
- `care_regulation.settings_changed`;
- `regulation_rnds.settings_changed` (novo);
- mudanças de CNES da unidade (`health_unit.changed`, se não existir, criar).

Ele recalcula `desired_status`, `pending_steps` e `desired_digest`. Quando diferem do último aceite, a linha vira `queued`.

**Saída da regulação municipal** (decisão 13): quando o pedido vira `external` com a configuração desligada, ou quando a configuração é desligada, o pedido recebe uma substituição com `excluded` e fica `excluded`. Volta a enviar se voltar a ser municipal.

**Paciente apagado:** o `Citizens::Erase` já retém o paciente com pedido de regulação (`request_erasure.rb:166`), então a cadeia continua normal. O estado `erased` fica reservado para o caso de o pedido perder a identificação por outro caminho: nada é enviado e o pedido fica fora do alarme (decisão 17).

**Carga inicial** (decisão 15): ao ligar o interruptor, `Regulation::RndsInitialLoadJob` cria as linhas dos pedidos não terminais e dos terminais encerrados nos últimos 30 dias, com `initial_load: true`. Eles saem no ritmo `Regulation::RndsLimits::INITIAL_RATE_PER_MINUTE = 60`. Com o interruptor desligado ou sem credencial, as linhas são mantidas, mas nada sai.

## 3. Envio

**`Regulation::RndsDeliverJob`** (`EachCityJob`, a cada minuto, `config/recurring.yml`, `limits_concurrency key: "rnds_regulation_deliver"`):
- faz o claim de linhas `queued` com `SKIP LOCKED`, uma por pedido;
- monta o próximo documento (o primeiro passo de `pending_steps`, ou o desejado);
- confere, envia e registra.
- **Primeiro envio:** `POST /api/fhir/r4/Bundle`, documento novo.
- **Seguintes:** substituição, com `Composition.relatesTo { code: replaces, targetReference: Composition/{last_rnds_id} }` e o mesmo `Bundle.identifier.value`.
- **Timeout sem resposta:** antes de reenviar, `GET /identifier?system=…&value={bundle_identifier}&docType=…` descobre se o documento já foi aceito.
- **`DELETE`:** só na reconciliação, quando a consulta acha um documento duplicado.

**Documento** (`Regulation::RiraBundle.build(request:, sync:, status:, now:)`):
- Bundle `document` com 4 entradas, nas URLs canônicas do IG, em `config/rnds/rira_1_0.yml` com `confirmed: true`:
  - Composition `BRRegulacaoAssistencial` (`type` RA, `event` 1..1 com a situação e `period.end`);
  - Appointment `BRAgendamentoRegulacaoAssistencial` (sempre presente; `proposed` no `pending`);
  - ServiceRequest `BRRequisicaoRegulacaoAssistencial` (`intent` proposal, `authoredOn` = `requested_at`, `requester` = CNES da unidade solicitante, `code` = SIGTAP, `priority`, `category`);
  - Condition `BRProblemaDiagnostico` (CID-10 ou CIAP-2).
- **Paciente:** por identifier, CPF ou CNS (`BRIndividuo-1.0`).
- **Executante:** CNES do prestador. Na rede própria, o CNES da unidade.
- **`scheduled_at`** → `Appointment.start` (decisão 18). A correspondência fica no perfil, `date_fields`.
- **Modalidade:** fixa em `09`.
- **CBO do executante** (decisão 14), na ordem:
  1. o `professional_links.cbo_code` do horário da agenda (`appointments.shift_id → professional_shifts.professional_link_id`);
  2. o `care_offers.performer_cbo` da autorização;
  3. o `care_providers.default_performer_cbo`.
  - É exigido só em `booked` e `attended`, nos grupos SIGTAP 03 e 04.

**Configuração da cidade:** tabela nova `regulation_rnds_settings`, linha única:
- `priority_by_color` (jsonb `{ red: urgent, yellow: routine, green: routine, blue: routine }`);
- `specialty_sigtap` (jsonb `{ <CBO>: <SIGTAP> }`, padrão `0301010072` para todo CBO sem entrada);
- `send_external` (bool, padrão `false`);
- `version`.
- Gravada em `PUT /regulation/admin/rnds_settings` pelo `municipal_admin`, com `expected_version` → 409 `rnds_settings_changed`. Publica `regulation_rnds.settings_changed`.

**Colunas novas no 26a:**
- `care_offers.performer_cbo` (6 dígitos, opcional);
- `care_providers.default_performer_cbo` (6 dígitos, opcional).
- Gravadas pelas rotas de admin que já existem (`PATCH /regulation/admin/offers/:id` e o de prestador).

**Conferência** (`Regulation::RndsValidation`, no molde do `Rnds::Validation`). Devolve `:ok` ou `[:blocked, reasons]`. Os motivos são:
- `missing_diagnosis`;
- `requester_unit_without_cnes`, `performer_without_cnes`;
- `patient_without_cpf_or_cns`;
- `performer_cbo_missing`;
- `specialty_without_sigtap`;
- `skeleton:<campo>`, do esqueleto `Regulation::RiraSkeleton`.

A verificação de parados roda de hora em hora (`BLOCKED_RECHECK`, como na vacina).

**Respostas:**
- 201: grava `last_rnds_id` (do `Location`), `sent_status` e `sent_digest`; a linha vira `idle`.
- 422: `rejected`, com os códigos. Volta a `queued` quando o `desired_digest` muda.
- 401/403: pausa a cidade e marca a credencial `unauthorized`.
- Outros: backoff do `Rnds::Backoff`, com `GIVE_UP_AFTER` próprio de 15 dias. Depois disso, `failed`.

## 4. Alarme

`Regulation::RndsAlert.counts(scope, now:)` conta:
- `queued`, `blocked`, `rejected`, `failed`;
- `late`: linha cuja última mudança de situação ou de conteúdo (`enqueued_at`) foi antes do fim do dia anterior e que não está `idle` às 23:00 de hoje no fuso da cidade.
- As linhas com `initial_load` ficam fora do `late` nas primeiras 72 horas.

## 5. Rotas do api

| Rota | Quem |
|---|---|
| `GET /regulation/rnds/status` (contagens e faixa) | `municipal_admin`, `care_regulator` |
| `GET /regulation/rnds/syncs` (filtros `status`, `reason`; lista com o número do pedido, a situação, o motivo e as datas, **sem paciente**) | os mesmos |
| `POST /regulation/rnds/syncs/:id/retry` (só `rejected` ou `failed`: reenfileira sem mudar o conteúdo, para recusa transitória) | `municipal_admin`, com step-up |
| `GET/PUT /regulation/admin/rnds_settings` | leitura pelos mesmos; escrita pelo `municipal_admin` |
| `PUT /integrations/credentials/rnds_regulation`, `POST …/check` | `municipal_admin` (§1) |

Todas exigem `Regulation::RndsGate.usable?`, exceto a credencial e as configurações, que valem antes de ligar.

## 6. Dashboard

- **Grupo "Regulação":** entra o item `regulation-rnds`, "Envio à RNDS". A tela traz:
  - a faixa do alarme;
  - os totais;
  - a lista por motivo, com o atalho para corrigir: a oferta para o CBO, o prestador e a unidade para o CNES, as configurações para o SIGTAP da especialidade e o pedido;
  - "Reenviar" com step-up, só para recusado ou falho.
  - Molde: `VaccinationPending.tsx`.
- **"Prazos e lista do enfermeiro":** ganha a seção "Envio à RNDS", com cor → caráter, SIGTAP por especialidade e "Enviar também os pedidos regulados pelo estado", que traz o aviso "Ligue só se a cidade pactuou com o estado ser a responsável pelo envio desses pedidos."
- **Oferta e prestador:** o campo "CBO do executante".
- **Integrações:** a credencial `rnds_regulation`, com os campos da `rnds` mais "Identificador do solicitante (DATASUS)" e "CNS do responsável pelo envio".
- **Frases:** `REGULATION_RNDS_ERRORS` em `src/lib/regulationRndsErrors.ts`, com cada motivo de parado e os códigos de recusa mais comuns.

## 7. Maintenance

- `FEATURE_LABELS.rnds_regulation` = "Envio da regulação à RNDS".
- `MISSING_LABELS` ganha `credential_missing:rnds_regulation`, `credential_unauthorized:rnds_regulation`, `care_regulation_disabled` (se não existir) e `rnds_disabled`.
- O aviso de simulado do `rnds_mock` passa a citar os dois envios.

## 8. CI, simulado e testes

- **`lib/fake_rnds`:**
  - aceita o Bundle RIRA;
  - simula substituição repetida (EDS-MSG009, 422), `DELETE` (204 ou EHR-ERR825) e `GET /identifier`;
  - o marcador de falha fica no `Bundle.identifier.value` (`RECUSA`, `TIMEOUT`, `DUPLICADO`).
- **CI:** um job novo `rira-validate` em `.github/workflows/ci.yml`, com `setup-java` e o validador oficial do HL7 (`validator_cli.jar`, com versão fixa e cache), mais o pacote `br.gov.saude.rira.fhir#1.0.0`.
  - Ele valida os documentos de exemplo gerados por `bin/rails regulation:rira_examples` (um por situação, mais urgente e eletivo, mais rede própria e externa).
  - Falha a CI se houver erro.
- **Specs do api:**
  - cada linha da tabela de situação;
  - a falta dupla;
  - a substituição em ordem;
  - o timeout com `GET /identifier`;
  - a retificação por digest;
  - `excluded`;
  - a carga inicial com ritmo;
  - o alarme com `initial_load`;
  - os motivos de parado;
  - o sigilo (`spec/security/regulation_rnds_secrecy_spec.rb`);
  - a suíte completa.

## 9. Eventos e LGPD

**Eventos de domínio** (só ids):
- `regulation_rnds.queued`
- `regulation_rnds.accepted`
- `regulation_rnds.blocked`
- `regulation_rnds.rejected`
- `regulation_rnds.excluded`
- `regulation_rnds.settings_changed`

**Sigilo:**
- O payload só vai no corpo e é guardado cifrado por 30 dias.
- A lista do dashboard nunca mostra paciente.
- Log e evento ficam sem CPF, CNS, CID nem justificativa.

## 10. Decisões (Q-n)

| # | Decisão |
|---|---|
| Q1 (coordenador, contrato) | A configuração da regulação para o RNDS fica numa tabela própria (`regulation_rnds_settings`). O 26a não tem tabela de settings, e prazos e lista do enfermeiro já têm as deles. |
| Q2 (coordenador, contrato) | O CBO da rede própria vem do vínculo do profissional do horário da agenda (`professional_links.cbo_code`), lido na montagem do documento. Ele não é copiado para o pedido. |
| Q3 (coordenador, contrato) | O `Citizens::Erase` já retém o paciente com pedido de regulação. Por isso a decisão 17 da spec só vale para um caso de borda, e o estado `erased` fica reservado. |
| Q4 (coordenador, contrato) | A CI ganha Java só num job à parte, `rira-validate`. O runtime do api não usa Java, como pede a decisão 3 da spec. |
| Q5 (coordenador, contrato) | O `Rnds::Client` passa a receber a credencial pelo `kind`, e o cache separa por `kind`. O envio da vacina não muda de comportamento. |
| Q6 (api D1, D2, D4, D5, D6, D14) | **Ajustes técnicos aceitos como no plano do api:**<br>- `bundle_identifier` é `varchar(64)`, para caber o marcador do simulado;<br>- o pacote RIRA reconstruído fica versionado no repo, com SHA-256, porque ele não está no registro FHIR;<br>- `RiraBundle.build` é puro sobre `facts:`, com o atalho `.for(request:)`;<br>- a tabela de sync ganha `claimed_at` e os timestamps;<br>- o `docType` do `GET /identifier` fica como hipótese no YAML, a confirmar na homologação (gate);<br>- as exigências do perfil (`Composition.author`; sem `custodian` nem `section.title`; executante em `ServiceRequest.performer`) seguem o IG. |
| Q7 (api D3) | **Externo quer dizer `state_regulated`.** O pedido regulado pelo estado fica fora do envio em qualquer situação (`external`, `scheduled`, `completed`), salvo `send_external` ligado. É ele que recebe o `excluded` da decisão 13. |
| Q8 (api D7, D13) | `erased` só vale para o pedido que já tinha documento aceito e perdeu CPF e CNS. O que nunca foi aceito fica `blocked` (`patient_without_cpf_or_cns`). A linha "`closed` por outro motivo → `cancelled`" fica como defesa. |
| Q9 (api D8; dashboard D3) | `specialty_without_sigtap` vale quando o SIGTAP configurado, ou o padrão, não for regulável ou não estiver vigente na SIGTAP da competência. |
| Q10 (api D9; texto corrigido pela Q19) | As leituras de status e da lista exigem `Regulation::RndsGate.enabled?`, para a tela abrir com a credencial recusada. As configurações e a credencial valem antes de ligar, como no §5. O reenvio e o job exigem `usable?`. |
| Q11 (api D10–D11; dashboard D1–D2) | **Formas.** As contagens trazem também `accepted`. A lista mostra `request_ref`: os 8 primeiros caracteres hex do id do pedido, em maiúsculas, calculados pelo api, com o link para o detalhe. As formas de `/status` e `/syncs` são as do plano do api. |
| Q12 (api D12; dashboard D4–D5) | **Fonte do CBO do executante, nesta ordem:**<br>1. o CBO do alvo, quando o pedido é por especialidade;<br>2. o profissional da agenda;<br>3. o `performer_cbo` da oferta;<br>4. o `default_performer_cbo` do prestador.<br>A oferta nova herda o `performer_cbo` da oferta do mês anterior para o mesmo prestador e alvo. O POST de oferta aceita `performer_cbo`, e o PATCH aceita mudar só ele, sem exigir `quantity`. |
| Q13 (dashboard D6–D9) | **Telas e frases.**<br>- O PUT das configurações pede step-up.<br>- O regulador vê as configurações só para leitura, num resumo da tela "Envio à RNDS". A seção editável fica no admin.<br>- Sem painel da regulação: a faixa do alarme fica só na tela "Envio à RNDS".<br>- Os códigos de recusa e os formatos (CBO de 6 dígitos, SIGTAP de 10, identificador do solicitante, CNS de 15, reenvio) são os do plano do dashboard. |
| Q14 (dashboard D10; maintenance DM1–DM2) | **Pré-requisito "um dos envios".** O motivo é `rnds_sending_disabled`, com a frase "ligue antes um dos envios à RNDS (vacina ou regulação)". O api cria esse requisito alternativo no `missing_for`. O aviso do simulado no maintenance cita só os envios ligados. |
| Q15 (api pré-voo B4) | **Caráter do pedido sem cor decidida.** O `priority` e o `appointmentType` saem de `priority_by_color[effective_color]`, em que `effective_color = color \|\| suggested_color`, como a fila, o CSV e a guia do 26a. Sem nenhuma das duas cores, o pedido fica parado, com motivo, e nada é enviado. |
| Q16 (api pré-voo I4) | **Prazo de desistência.** Os 15 dias do `GIVE_UP_AFTER` contam do `sent_at` da primeira remessa sem aceite depois do último aceite da linha, e não do `enqueued_at`. O tempo parado (sem credencial, desligado ou com motivo) não conta. O "Reenviar" manual de uma linha `failed` reabre a contagem. O `enqueued_at` fica só para o alarme D-1. |
| Q17 (api pré-voo I5) | **Purga do payload.** Toda remessa com `COALESCE(accepted_at, sent_at)` anterior a 30 dias tem o payload apagado, antes do gate, com o interruptor ligado ou não (molde do 27b, R19 e Q30). Os metadados ficam: situação, códigos, identificador e evidência sem segredo. O reenvio depois da purga remonta o documento a partir do estado atual. |
| Q18 (api pré-voo I7; homologação) | **Data da falta.** O `Appointment.start` do documento `absence` vem do `appointments.scheduled_at` do horário `no_show`/`expired` da cadeia (rede própria). O pedido externo vai sem a data. A aceitação pela RNDS entra como pergunta de homologação, junto da decisão 18. |
| Q19 (api pré-voo I10) | **Texto da Q10.** A Q10 foi corrigida para seguir o §5: as configurações e a credencial valem antes de ligar, e só o status e a lista exigem `enabled?`. Os planos do api e do dashboard já seguem o §5. |
| Q20 (api Task 8; homologação) | **Fim do horário.** Os documentos `booked`, `attended` e `absence` levam `Appointment.start` e `Appointment.end`, como pedem as invariantes app-2, app-3, mira-5, mira-7 e mira-10 do pacote. Na rede própria, `end` é o `appointments.ends_at`; no legado, `effective_ends_at = scheduled_at + 15 min`. No externo, `end = scheduled_at + Appointment::LEGACY_SPAN`. A aceitação do `end` estimado pela RNDS é pergunta de homologação. |
| Q21 (api Task 8) | **Data da realização.** `ServiceRequest.occurrenceDateTime` vai em `attended` e `absence` (mira-9, mira-12). Em `attended`, é o `scheduled_at` do horário atendido (rede própria) ou o `scheduled_at` do pedido (externo). Em `absence`, é a mesma data do `start`, a do horário faltado. |
| Q22 (api Task 8; ajusta a Q18 e a I7 do plano) | **Falta com executante e data.** Na rede própria, o `absence` leva o executante (`ServiceRequest.performer`, o CNES da unidade do horário) e as datas do horário `no_show` (mira-10, mira-11, mira-12). A regra "absence sem executante" cai. Só o `no_show` gera falta: o `expired` volta a `authorized` no 26a e não é falta. No pedido externo, o 26a apaga a data e o prestador ao registrar a falta, e por isso o `absence` externo fica parado, com o motivo novo `external_absence_without_schedule`, até a homologação dizer como a RNDS quer esse caso. Isso só acontece com o envio dos pedidos regulados pelo estado ligado (Q7), que vem desligado por padrão. O 26a não muda no 26c. |
| Q23 (api Task 8) | **Documento nunca sai inválido.** O `RiraSkeleton` confere, antes do envio, as invariantes do pacote por situação. Um documento que não passa fica parado, com motivo, e não é enviado. Motivos novos no fim do `BLOCK_REASONS`: `missing_priority` (Q15) e `external_absence_without_schedule` (Q22). As frases do dashboard são "falta a cor do pedido: o regulador ainda não decidiu e não há cor sugerida" e "falta de pedido regulado pelo estado: aguarda a homologação para ser enviada". |
| Q24 (api Task 8, revisão; corrige a premissa da Q22) | **A falta guarda o horário e o prestador.** A falta externa não depende só do `send_external`: o pedido municipal num prestador contratado também passa pelo `Misses.apply!`, que apaga a data e o prestador, e esses pedidos vão por padrão. A migração `20261126900001` (ainda não mergeada) acrescenta à `regulation_request_events` as colunas nulas `scheduled_at`, `ends_at` e `care_provider_id`. O `Misses.apply!` grava nelas, na linha `missed`, o horário e o prestador que está limpando, sem mudar o comportamento do 26a. O `RiraFacts` lê a falta dessa linha nas duas redes, e a rede da falta é a do horário faltado, não a da autorização vigente. O `external_absence_without_schedule` fica só para a falta registrada antes do deploy, sem esses dados. O 26a ganha esse toque (o `Misses.apply!` e o spec dele), coberto pela suíte completa. |
| Q25 (api Task 12, revisão) | **Sem marcadores sintéticos em `error_codes`.** `regulation_rnds_deliveries.error_codes` guarda só códigos da RNDS (`ERROR_CODE`, §9). Entram duas colunas na migração `20261126900001`, que ainda não foi mergeada. (1) `regulation_rnds_syncs.retry_window_started_at` (timestamptz, nula): começa no `sent_at` da primeira remessa sem aceite depois do último aceite, o `GIVE_UP_AFTER` de 15 dias conta dela (Q16), o "Reenviar" manual a zera e ela não sai em nenhuma resposta da API. (2) `regulation_rnds_deliveries.unanswered_closed_at` (timestamptz, nula): é gravada uma vez, só enquanto `http_status` e `accepted_at` são nulos, quando a reconciliação não acha o documento, e a remessa deixa de contar como sem resposta. Com isso, o trigger da resposta gravada uma vez fica coerente. O `DELETE` de duplicado só conta como sucesso no 204 ou no `EHR-ERR825`; outra recusa vai a `last_error_codes`, e a reconciliação tenta de novo, com backoff. Nota (Task 12): toda parada não transitória (recusa, substituição recusada, bloqueio com motivo e pausa por 401/403) zera a `retry_window_started_at`, e a janela recomeça no próximo envio. Assim o tempo parado não conta, como diz a Q16. |
