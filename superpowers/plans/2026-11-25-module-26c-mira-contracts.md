# Módulo 26c — Contratos entre apps

**Fonte única** do que o api, o dashboard e o maintenance combinam no 26c. As linhas do §10 prevalecem sobre o texto das seções anteriores. Os planos de cada repositório (`-api.md`, `-dashboard.md`, `-maintenance.md`) seguem este documento. Uma divergência vai para a seção "Divergências propostas ao contrato" do plano, e a decisão volta aqui como Q-n.

- **Spec:** `docs/superpowers/specs/2026-11-25-module-26c-mira-design.md` (aprovada em 2026-11-25; decisões 1–18).
- **Pesquisa:** `docs/pesquisa/2026-11-23-mira-regulacao-rnds-26c.md` (IG RIRA `br.gov.saude.rira.fhir#1.0.0-release`; achados 1–15).
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
- `SECRET_KEYS`: `pfx_base64`, `pfx_password`, `requester_cnes`, `requester_id` (o identificador do solicitante do DATASUS, usado no system `BRRNDS-{requester_id}`) e `responsible_cns` (15 dígitos).
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
- **CI:** um job novo `rira-validate` em `.github/workflows/ci.yml`, com `setup-java` e o validador oficial do HL7 (`validator_cli.jar`, com versão fixa e cache), mais o pacote `br.gov.saude.rira.fhir#1.0.0-release`.
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
