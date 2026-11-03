# Módulo 30 — Contratos entre apps

**Spec:** `superpowers/specs/2026-11-03-module-30-sanitary-transport-design.md` · **ADR:** `adr/0048.md` · **Pesquisa:** `pesquisa/2026-11-03-transporte-e-tfd-30.md`

Fonte única dos formatos entre `api`, `dashboard`, `wpda` e `maintenance`.

Convenções herdadas dos contratos do 21a ao 29 (o 29 §15 é o mais recente):
- erros `{ "error", "path"? }`, com `problems[{ "path", "error" }]` trazendo todas as recusas de uma vez;
- step-up (TOTP) onde marcado; papéis privilegiados exigem segundo fator no login;
- interruptor desligado: 403 `feature_disabled` com `"feature"`;
- JSON em toda escrita por cookie;
- listas com envelope e `next_before`;
- registro de outro escopo pedido por id → 404; lista com escopo alheio → 403;
- leitura nominal com trilha `clinical_record.viewed` só onde marcado, sem polling; contadores sem trilha;
- dado sensível só no corpo, nunca na URL nem no log; busca de paciente por CPF por POST;
- nenhum dado pessoal ou clínico (CPF, CNS, nome, CNH, documento do acompanhante, destino de tratamento, CID, justificativa) em `inspect`/`to_s`/`pretty_print`/`as_json` de modelo, evento ou log;
- paciente sem CPF como no 24a (`has_cpf`, `cns_masked`);
- datas e horas em America/Sao_Paulo, aritmética de calendário (lição do 29);
- imutabilidade por trigger em `db/city_triggers.sql`;
- todo `Platform.audit` novo entra no R18.

## 1. Nomes no código

| Spec | Código |
|---|---|
| interruptores | `sanitary_transport` (sem pré-requisito); `tfd` (`requires: feature:sanitary_transport feature:clinical_record`) |
| papéis da cidade | `transport_dispatcher` e `tfd_committee` (privilegiados, em `Membership::ROLES` e `PRIVILEGED_ROLES`, com a trava `ck_memberships_role`); `driver` (não privilegiado) |
| veículo | `transport_vehicles.kind`: `van` \| `minibus` \| `bus` \| `car` \| `ambulance_a`; `renem_code` opcional; `seats` (passageiros, sem o motorista), `wheelchair_spots`, `cnes` (7 dígitos, opcional) |
| motorista | `transport_drivers` (1:1 com `user_id` de papel `driver`): `license_number` (cifrado), `license_category` (`B` `C` `D` `E`), `license_expires_on` |
| destino | `transport_destinations`: `ibge_code` (7 dígitos), `name`, `distance_km` (inteiro > 0; 0 para intramunicipal), `metropolitan` (bool), `active` |
| prestador | `care_providers.ibge_code` (opcional; nulo = cidade própria) |
| unidade de gestão | `health_units.kind` ganha `health_management` (secretaria ou central de regulação) |
| viagem | `transport_trips.status`: `planned` \| `confirmed` \| `in_progress` \| `done` \| `cancelled` |
| origem do passageiro | `transport_trip_passengers.origin`: `regulation` \| `appointment` \| `tfd` \| `series` \| `exception` |
| motivo de exceção | `exception_reason`: `follow_up_return` \| `legacy_patient` \| `scheduling_pending` \| `other` (com `exception_note` obrigatório em `other`) |
| presença | `attendance`: `pending` \| `boarded` \| `no_show`; vale para o paciente e, separado, para o acompanhante |
| processo de TFD | `tfd_cases.status`: `draft` \| `submitted` \| `returned` \| `authorized` \| `denied` \| `scheduled` \| `travelling` \| `completed` \| `cancelled` |
| modal do trecho | `tfd_legs.mode`: `fleet` \| `bus_ticket` \| `air` \| `river`; `direction`: `outbound` \| `return` |
| ajuda de custo | `tfd_allowances.kind`: `meal_overnight` \| `meal_day` (paciente e acompanhante), `ground_km` \| `air_miles` \| `river_miles` |
| fonte do BPA | `performed_procedures.source` ganha `tfd`, com `tfd_leg_id` ou `tfd_allowance_id` |

## 2. Formas

`<vehicle>`: `{ "id", "plate", "renavam", "model", "kind", "renem_code"|null, "seats", "wheelchair_spots", "cnes"|null, "active" }`.

`<driver>`: `{ "id", "user": { "id", "name" }, "license_category", "license_expires_on", "license_status": "valid"|"expiring"|"expired" }` (`expiring` = vence em até 30 dias; `license_number` nunca sai, só `"license_number_last4"`).

`<destination>`: `{ "id", "ibge_code", "name", "distance_km", "metropolitan", "active" }`.

`<passenger>` (central e motorista; **sem dado clínico**): `{ "id", "patient": { "id", "display_name", "has_cpf", "cns_masked"|null, "phone_masked"|null }, "stop_id", "destination": { "provider_name"|null, "address"|null, "destination_id" }, "appointment_at"|null, "takes_seat", "wheelchair", "companion": { "name", "document_last4"|null }|null, "origin", "exception_reason"|null, "attendance", "companion_attendance"|null }`. Nunca procedimento, especialidade, CID ou o texto do pedido.

`<trip>`: `{ "id", "date", "departure_at", "return_at", "vehicle": <vehicle-resumo>|null, "driver": { "id", "name" }|null, "status", "stops": [ { "id", "position", "kind": "boarding"|"destination", "destination_id"|null, "label" } ], "passengers": [<passenger>], "occupancy": { "seats_used", "seats", "wheelchair_used", "wheelchair_spots" }, "series_id"|null, "log": <trip-log>|null }`.

`<trip-log>`: `{ "odometer_start", "odometer_end", "fuel": [ { "liters", "amount_cents", "station" } ], "expenses": [ { "kind", "amount_cents", "note" } ], "closed_at", "closed_by_name" }`.

`<tfd-case>` (comissão e médico autor; **nominal com trilha**): `{ "id", "patient": {…}, "status", "requested_by": { "name", "cbo" }, "consultation_id"|null, "regulation_request_id"|null, "report": { "sigtap_code", "sigtap_name", "cid10", "justification", "companion_indicated", "companion_reason"|null, "transport_mode" }, "destination": <destination>, "provider": { "id", "name" }|null, "appointment_at"|null, "inpatient": [ { "from", "to" } ], "legs": [<tfd-leg>], "allowances": [<tfd-allowance>], "payments": [<tfd-payment>], "accountability": { "closed_at", "document": { "id", "filename" } }|null, "events": [ { "kind", "at", "by_name", "reason"|null } ] }`.

`<tfd-leg>`: `{ "id", "direction", "mode", "date", "distance_km", "trip_passenger_id"|null, "ticket": { "amount_cents", "carrier" }|null, "done": bool }`.

`<tfd-allowance>`: `{ "id", "person": "patient"|"companion", "kind", "sigtap_code", "quantity", "table_cents", "complement_cents", "total_cents" }`.

`<tfd-payment>`: `{ "id", "paid_on", "amount_cents", "method": "cash"|"transfer"|"pix"|"check", "receipt_number", "by_name" }`.

## 3. Administração (`municipal_admin`; escrita com step-up)

- `GET/PATCH /transport/admin/settings` — `{ "tfd_min_distance_km": 50, "companion_rule": { "max_age_child": 15, "min_age_elderly": 60 }, "tfd_billing_health_unit_id", "allowance_complements": { "<sigtap_code>": <cents> } }`. `tfd_billing_health_unit_id` precisa ser unidade `health_management` com CNES.
- Papéis: pelo mecanismo existente de membros (`/team`), com os papéis novos.

## 4. Central de transporte (`transport_dispatcher`)

- `GET/POST/PATCH /transport/vehicles[/:id]`; `GET/POST/PATCH /transport/drivers[/:id]` (`{ "user_id", "license_number", "license_category", "license_expires_on" }`; `user_id` precisa ter papel `driver`); `GET/POST/PATCH /transport/destinations[/:id]`.
- `GET /transport/trips?date=&vehicle_id=` → `{ "items": [<trip>] }`.
- `POST /transport/trips` — `{ "date", "departure_at", "return_at", "vehicle_id"?, "driver_id"?, "stops": [..] }` → 201 `<trip>`. `PATCH /transport/trips/:id` (mesmos campos). `POST /transport/trips/:id/confirm` exige veículo e motorista. Recusas em `problems[]`: `vehicle_overlap`, `driver_overlap` (exclusion constraint; 422, nunca 500), `driver_license_expired`, `driver_license_category` (veículo com `seats > 8` exige `D` ou `E`), `vehicle_inactive`, `return_before_departure`.
- `POST /transport/trips/:id/cancel` — `{ "reason" }` (avisa os passageiros).
- `POST /transport/scheduled_care` — `{ "date", "destination_id"? }` → atendimentos marcados na data ainda sem viagem: `{ "items": [ { "patient": {…}, "origin", "origin_id", "appointment_at", "destination": {…} } ] }`. Sem dado clínico.
- `POST /transport/trips/:id/passengers` — `{ "patient_id", "origin", "origin_id"?, "exception_reason"?, "exception_note"?, "stop_id", "destination_id", "provider_name"?, "address"?, "appointment_at"?, "takes_seat": true, "wheelchair": false, "companion"?: { "name", "document"?, "reason"? } }` → 201 `<passenger>`. Recusas: `seats_full`, `wheelchair_full`, `origin_required` (sem origem nem motivo), `companion_not_allowed` (fora da regra sem `reason`; no TFD, sem indicação no laudo), `patient_already_on_trip`, `origin_mismatch` (o atendimento não é desse paciente).
- `DELETE /transport/trips/:id/passengers/:pid` — `{ "reason" }`.
- `GET /transport/trips/:id/manifest.pdf` — lista de passageiros (§2 `<passenger>`, sem dado clínico; sem CPF, só nome, embarque, destino, hora, acompanhante, cadeira).
- Séries: `GET/POST/PATCH /transport/series[/:id]` — `{ "patient_id", "destination_id", "provider_name"?, "weekdays": [1,3,5], "appointment_time": "07:00", "departure_time": "05:30", "starts_on", "ends_on"?, "wheelchair", "companion"? }`; `POST /transport/series/:id/suspend` — `{ "from_on", "reason" }` (tira o paciente das viagens futuras). Job `Transport::MaterializeSeriesJob` diário (EachCityJob): 21 dias à frente; regra da spec (viagem do dia e destino com vaga → senão viagem `planned` sem escala); sem vaga em viagem `confirmed` → pendência em `GET /transport/pending` (`{ "items": [ { "series_id", "date", "reason": "no_seat"|"no_wheelchair_spot" } ] }`).
- `GET /transport/no_shows?from=&to=` → faltas por paciente (contagem e datas, sem destino clínico).

## 5. Motorista (`driver`; só as próprias viagens)

- `GET /transport/my_trips?date=` → `{ "items": [<trip>] }` (só onde `driver_id` é o dele; `<passenger>` sem `cns_masked` e sem `has_cpf`).
- `POST /transport/my_trips/:id/start` → `in_progress` (só `confirmed`, só no dia).
- `POST /transport/my_trips/:id/attendance` — `{ "items": [ { "passenger_id", "attendance", "companion_attendance"? } ] }`.
- `POST /transport/my_trips/:id/close` — `{ "odometer_start", "odometer_end", "fuel": [..], "expenses": [..] }` → `done`. Recusas: `odometer_invalid`, `attendance_pending` (todo passageiro precisa de `boarded`/`no_show`). Depois de `done`, presença e log são imutáveis (trigger); correção só pela central com motivo (`POST /transport/trips/:id/attendance_correction`, step-up).

## 6. TFD

- Pedido (médico CBO `225*` com `health_professional`): `POST /tfd/cases` — `{ "patient_id", "consultation_id"?, "regulation_request_id"?, "report": {…}, "destination_id", "provider_id"? }` → 201 `<tfd-case>` em `draft`; `PATCH /tfd/cases/:id` (só `draft`/`returned`, só o autor); `POST /tfd/cases/:id/submit`. Recusas: `distance_below_minimum`, `metropolitan_destination`, `primary_care_procedure` (SIGTAP do grupo da atenção básica, lista em `config/tfd/primary_care_groups.yml`), `cid_required`, `justification_required`.
- Comissão (`tfd_committee`, step-up nas decisões): `GET /tfd/cases?status=` → lista sem dado clínico (`{ id, patient.display_name, destination.name, status, submitted_at }`); `POST /tfd/cases/:id/read` → `<tfd-case>` (trilha); `POST /tfd/cases/:id/decide` — `{ "decision": "authorize"|"return"|"deny", "reason" }`; `authorize` exige `appointment_at` já informado (`appointment_required`); `POST /tfd/cases/:id/appointment` — `{ "appointment_at", "provider_id"? }`; `POST /tfd/cases/:id/inpatient` — `{ "from", "to"? }`.
- Trechos: `POST /tfd/cases/:id/legs` — `{ "direction", "mode", "date", "trip_passenger_id"?, "ticket"? }` (`fleet` exige `trip_passenger_id` de uma viagem com o paciente; `distance_km` vem do destino); `POST /tfd/legs/:id/done` (trecho de passagem realizado; o de frota fica `done` com `boarded`).
- Ajuda de custo: calculada pelo api ao marcar a volta `done` (par ida/volta): `meal_overnight` por noite fora menos noites internado, `meal_day` quando ida e volta no mesmo dia; acompanhante só se viajou (`companion_attendance = boarded` ou trecho de passagem com acompanhante); valor = SIGTAP da competência + complemento da cidade. `GET /tfd/cases/:id/allowances`.
- Pagamento: `POST /tfd/cases/:id/payments` — `{ "paid_on", "amount_cents", "method", "receipt_number" }` (imutável).
- Prestação de contas: `POST /tfd/cases/:id/accountability` — multipart com o comprovante (PDF/JPG/PNG, até 10 MB, armazenamento do 19c) → `completed`.
- Consulta (19a): `GET /tfd/prefill?consultation_id=` → `{ "patient_id", "cid10"|null, "sigtap_code"|null }` para o botão "pedir TFD".

## 7. BPA (módulo 22)

- Evento `tfd.leg_done` e `tfd.allowances_calculated` → `Billing::TfdProductionJob` → `Billing::Record.insert!` com `source: "tfd"`, instrumento BPA-I, `performed_on` = data do trecho, profissional = médico do laudo, estabelecimento = `tfd_billing_health_unit_id`.
- Deslocamento: `quantity` = `ceil(distance_km / 50)` por trecho e por pessoa (terrestre); aéreo por 200 milhas; fluvial por 27 milhas náuticas. Trecho de frota também fatura (gate: confirmar no SIGTAP oficial).
- Códigos em `config/tfd/sigtap_codes.yml` (lista do 08.03.01 da pesquisa); crítica do 22 sem a regra `establishment_is_aps` para `health_management`.
- Idempotente por `(tfd_leg_id|tfd_allowance_id, código, pessoa)`; cancelamento do processo depois do BPA fechado não apaga a linha (segue a regra do 22).

## 8. Cidadão (wpda)

- `GET /citizen/trips` → `{ "items": [ { "id", "date", "departure_at", "boarding": { "label" }, "destination": { "name", "provider_name"|null }, "status", "companion": bool } ], "tfd": [ { "id", "status", "destination_name", "appointment_at"|null, "allowance_total_cents"|null, "paid": bool } ] }`. Sem procedimento, sem CID e sem veículo ou motorista. Sem polling. Trilha da leitura do cidadão (como `GET /citizen/vaccinations`).
- Avisos: `transport_trip_notices` na caixa do wpda (`/citizen/notices` existente passa a incluir o tipo `trip`); SMS pelo `SmsGateway` com as regras do 17 (opt-in, janela 8h–20h), texto fixo sem destino clínico ("Sua viagem de <data> sai às <hora> de <embarque>").

## 9. Linha do tempo e painel

- Linha do tempo: fonte `transport`: `{ "kind": "transport", "at", "title": "Viagem para <cidade>", "summary": null, "detail_ref": { "kind": "trip_passenger", "id" } }`; TFD: `"title": "TFD para <cidade>"`. No wpda, igual, sem detalhe.
- `GET /transport/dashboard?month=` (gestor, `transport_dispatcher`, `tfd_committee`; sem trilha) → `{ "trips", "occupancy_pct", "no_shows", "km", "fuel_cents", "tfd_by_status": {…}, "tfd_paid_cents", "exceptions" }`. Contagens abaixo de 5 vêm como `"lt5"`.

## 10. Maintenance

Só os dois interruptores no mecanismo genérico; nenhuma tela nova.

## 11. Eventos (só ids)

`transport.trip_confirmed`, `transport.trip_cancelled`, `transport.passenger_added`, `transport.passenger_removed`, `transport.trip_changed`, `transport.trip_done`, `tfd.case_submitted`, `tfd.case_decided`, `tfd.leg_done`, `tfd.allowances_calculated`, `tfd.payment_recorded`. Sem `Platform.audit` novo (tudo na cidade); se surgir, vai para o R18.

## 12. LGPD

- `RequestErasure`: `tfd_cases` e o que pende dele (trechos, passageiros com `origin: tfd`, ajudas, pagamentos) retidos; demais passageiros anonimizados (paciente → casca, acompanhante apagado), presença mantida.
- Laudo: `report` cifrado; leitura só `tfd_committee` e o médico autor, sempre com trilha.

## 13. Ordem de entrega

api → dashboard → wpda → maintenance. Migração de cidade a partir de `20261103900001`. Worktree `apps/api/.claude/mod30`, branch `feat/mod-30-sanitary-transport`; frontends em `.worktrees/<app>-mod30`. Portas: api 3052, Vite dashboard 5209, wpda 5210, maintenance 5177 com o serviço do compose parado.

## 14. Acréscimos (revisão dos planos, 2026-11-03)

Reconciliação das divergências dos quatro planos. Onde um plano diverge deste §14, vale o §14; o executor acrescenta ao plano do api as rotas que só o dashboard pediu (T2, T3, T7, T9).

- **T1 — Linha do tempo:** campos reais `occurred_at` e `detail_ref.type` (padrão do 20/29), não `at`/`kind`. A fonte `transport` vem com `detail_ref: null` no dashboard e no wpda (não há tela de detalhe); `unit`, `professional_name` (nunca o motorista), `status` e `document` nulos; `occurred_at` = `departure_at` da viagem ou data do trecho de passagem feito. Entram passageiros `boarded` e trechos de TFD feitos; viagem cancelada e falta ficam fora. TFD usa o mesmo `kind: "transport"` com título "TFD para <cidade>".
- **T2 — Paciente na central:** `POST /transport/person_lookup { "cpf" }` e `POST /transport/patients/duplicates`, na forma do 24a/V21, para `transport_dispatcher`; a central não cria paciente.
- **T3 — Motoristas candidatos:** `GET /transport/driver_candidates` → `{ "items": [ { "user_id", "name" } ] }` (usuários com papel `driver` sem cadastro); recusas `user_not_driver` (422) e `user_already_driver` (409).
- **T4 — Paradas:** escrita `{ "id"?, "position", "kind", "destination_id"?, "label" }`; o PATCH substitui a lista; parada com passageiro removida → `stop_in_use`. "Mudou a parada" (aviso) = mudou o `label` da parada do passageiro.
- **T5 — Atendimentos marcados:** `POST /transport/scheduled_care { "date", "destination_id"?, "health_unit_id"? }`; horários do 17 só entram com `health_unit_id`. Item: `{ "patient", "origin", "origin_id", "appointment_at", "destination": { "destination_id"|null, "provider_name"|null, "address"|null } }`; chave estável `origin` + `origin_id`; o texto livre `schedule_place` do 26a nunca vira `provider_name`.
- **T6 — Série:** escrita exige `return_time` e `boarding_label`; `weekdays` ISO (1 = segunda … 7 = domingo). Leitura `<series>`: campos da escrita + `"status": "active"|"suspended"|"ended"`, `"suspended_from"|null`.
- **T7 — Opções do TFD:** `GET /tfd/options` → `{ "destinations": [<destination>], "providers": [ { "id", "name", "ibge_code" } ], "min_distance_km" }`, para o médico e a comissão.
- **T8 — Processo de TFD:**
  - o laudo exige `consultation_id` ou `regulation_request_id`;
  - o CBO `225*` do autor fica gravado no processo, e o BPA usa esse CBO;
  - `POST /tfd/cases/:id/cancel { "reason" }`, pela comissão, com step-up;
  - toda escrita do TFD devolve o `<tfd-case>` inteiro;
  - `<tfd-case>` ganha `trip_passengers: [ { "id", "date", "departure_at", "attendance", "companion_attendance" } ]`.
- **T9 — Médico autor:** `GET /tfd/my_cases` (envelope, sem dado clínico); abre por `POST /tfd/cases/:id/read` com trilha. A fila da comissão `GET /tfd/cases` vem em `{ "items", "next_before" }`, do mais antigo para o mais novo.
- **T10 — Trecho:**
  - `<tfd-leg>` ganha `companion: bool` e `outbound_leg_id`: a volta se pareia sozinha com a ida aberta, e sem ida dá `no_outbound_leg`;
  - marcar como feito um trecho de frota pela rota de passagem dá `fleet_leg_by_trip`;
  - recusas `tfd_leg_done`, `legs_pending` (prestação de contas com trecho aberto), `inpatient_open` e `inpatient_overlap`.
  - A internação fica em `tfd_cases.inpatient_stays` (jsonb).
- **T11 — Ajuda de custo e BPA:** o deslocamento também vira linha de `tfd_allowances` (`ground_km` etc.). A linha do BPA sempre aponta `tfd_allowance_id` (único), com `tfd_leg_id` como rastro; os dois eventos levam `tfd_leg_id` e vão ao mesmo job. O desconto da internação vale só para o paciente; o acompanhante recebe as noites (gate: manual da UF). `Billing::Critic` não muda (`establishment_is_aps` só marca `ubs`).
- **T12 — Unidade de faturamento:** `authorize` recusa `billing_unit_missing` com `procedure_billing` ligado e sem `tfd_billing_health_unit_id`; o PATCH de settings recusa `billing_unit_invalid`. `GET /transport/admin/settings` ganha `allowance_codes: [ { "code", "name", "kind" } ]` (só leitura). A unidade de faturamento **não** é `requires` do catálogo.
- **T13 — Prestação de contas:** JSON base64 `{ "content_type", "data", "filename"? }` (não multipart), até 10 MB decodificados, PDF/JPG/PNG; armazenamento `Exams::Files` do 21a; `filename` cifrado (o api gera um se faltar); download por URL curta `POST /tfd/cases/:id/accountability/url` → `GET /tfd/files/:token`.
- **T14 — Cidadão:** `GET /citizen/trips?citizen_id=` de um par validado (outro celular → 404; par só declarado → 403 `not_verified`). `sanitary_transport` e `tfd` vêm em `features` da sessão; transporte desligado → 403 `feature_disabled`; `tfd` desligado → `"tfd": []`. TFD em `draft` nunca aparece; `returned` aparece como "Em revisão pelo seu médico"; motivo, laudo e procedimento nunca. Itens por `departure_at`; `date` como data; horas ISO com fuso; `allowance_total_cents` inteiro (nulo antes do cálculo); `paid` quando os pagamentos alcançam o total.
- **T15 — Avisos:**
  - **Quando sai:** só em viagem `confirmed` ou `in_progress`; o `confirm` avisa todos os passageiros; o lembrete da véspera sai das 17h às 20h, como no 17.
  - **Forma em `/citizen/notices`:** `{ "kind": "trip", "id", "event": "added"|"departure_changed"|"boarding_changed"|"cancelled"|"removed"|"reminder", "date", "departure_at", "boarding": { "label" }, "read", "cpf_masked" }`, sem destino nem prestador (a caixa mostra pares só declarados).
  - **SMS:** um texto por `event`, com o prefixo do 17; exige `sms_opt_in` e respeita `notices_muted` e `appointment_reminders_muted`.
- **T16 — Painel:** contagem de 1 a 4 sai `"lt5"`; 0 continua 0; somas nunca suprimidas.
- **T17 — Acompanhante:** a regra da CIT fica como idade ≤ `max_age_child` (15) ou ≥ `min_age_elderly` (60), na data da viagem.
- **T18 — Formas e limites:**
  - `<passenger>.destination` ganha `name`;
  - `no_shows`: `{ "items": [ { "patient": { "id", "display_name" }, "count", "dates" } ] }`;
  - `expenses[].kind`: `toll` | `parking` | `meal` | `other`;
  - correção de presença `{ "items", "reason" }`, com step-up;
  - formas do 26a: `CareProvider.ibge_code` em `/regulation/providers`; `care_provider_id?` no `POST …/schedule`; `schedule.care_provider` na leitura.
  - Limites de texto:
    - motivos: 10–500
    - justificativa: 20–2000
    - nome do acompanhante: 3–120
    - documento do acompanhante: até 20
    - parada: 3–120
    - posto, empresa e observação: até 120
    - recibo: 1–40
    - modelo do veículo: 2–80
- **T19 — Recusas:**
  - `GET /transport/trips` exige `date`;
  - 422: `vehicle_required`, `driver_required`, `trip_not_open` (escrita em viagem `done`/`cancelled`), `trip_not_today` e `invalid_transition` (no `start`);
  - 409: `plate_taken`, `renavam_taken`, `ibge_taken`.
- **T20 — Tabelas além do ADR:** `transport_settings`, `transport_pending_items`, `transport_attendance_corrections`.
- **T21 — Maintenance:**
  - nomes "Transporte sanitário" e "Tratamento Fora de Domicílio (TFD)", iguais no dashboard;
  - o maintenance traduz o `missing` `sanitary_transport_disabled`;
  - desligar o transporte não desliga o TFD em cascata: o TFD fica "ligado, falta pré-requisito" e responde 403 até o transporte voltar.

### Acréscimos da execução (2026-11-03)

- **T22 — Data da viagem:** sem CHECK no banco; `date` é derivada de `departure_at` no fuso da cidade, em Ruby (o dump do esquema é único e o fuso varia por cidade).
- **T23 — Passageiro anonimizado:** a exclusão LGPD primeiro retira o paciente das viagens abertas; o passageiro anonimizado fica congelado, e remover ou corrigir dá 422 `passenger_anonymized`. A exclusão também anonimiza o acompanhante gravado nas séries.
- **T24 — Nomes:** a coluna `tfd_payments.method` chama-se `payment_method`; o JSON continua `"method"`.
- **T25 — Cancelamento do TFD:** só a comissão cancela, com step-up (T8); o autor não cancela o próprio rascunho.
- **T26 — BPA:** `character` da linha do TFD é `elective`, como no laboratório.
- **T27 — Evento de configuração:** o `PATCH /transport/admin/settings` publica `transport.settings_changed` (só o id da cidade e as chaves alteradas, sem valores), como `vaccination.settings_changed` no 29; entra no §11.
- **T28 — Cadastros:**
  - o §11 ganha `transport.vehicle_saved`, `transport.driver_saved` e `transport.destination_saved`, só com ids;
  - `<driver>` ganha `"active"`, e `GET /transport/drivers?active=true` filtra por ele;
  - o `POST …/schedule` do 26a substitui o agendamento inteiro: reagendar sem `care_provider_id` limpa o prestador, então o dashboard (Task 7) reenvia o prestador ao reagendar;
  - falta ou sincronização da agenda limpa o `care_provider_id` junto com o agendamento.
- **T29 — Viagem confirmada:**
  - motorista inativo é recusado com `driver_inactive` (422), no padrão de `vehicle_inactive`;
  - numa viagem `confirmed`, todo PATCH valida as regras do `confirm` sobre o estado resultante, então trocar o motorista ou o veículo que ficou inválido é sempre aceito;
  - `cancel` nunca passa por essas regras;
  - `transport.trip_cancelled` sai em todo cancelamento, e o aviso ao cidadão só sai para viagem que já foi confirmada (T15).
- **T30 — Paciente na central (precisa o T2):**
  - O `person_lookup` segue a regra comum do 27a/29: achando um cidadão com par validado e sem paciente, materializa o paciente.
  - "A central não cria paciente" quer dizer que ela não cadastra pessoa nova do zero.
  - `POST /transport/patients/duplicates` devolve `{ "candidates" }`, o envelope real do 27a/29.
- **T31 — Agenda do 17 em `scheduled_care`:**
  - O item sai com `destination_id: null`, `provider_name` = nome da unidade e `address` = endereço da unidade. Nada clínico: sem profissional nem especialidade.
  - Com `destination_id` no filtro, os itens do 17 não aparecem (são intramunicipais).
- **T32 — Códigos:**
  - origem inexistente ou em situação errada: 422 `invalid_request` em `origin_id`;
  - passageiro já retirado: 404;
  - `DELETE` do passageiro devolve o `<trip>`.
- **T33 — Séries:**
  - **Dia sem viagem:** só uma pendência não resolvida faz o job pular o dia. Uma pendência resolvida, do paciente retirado ou da viagem cancelada, é reaberta.
  - **Viagem do dia:** "já está numa viagem" conta só passageiro ativo em viagem não cancelada.
  - **Execução do job:** cada dia roda num savepoint próprio. A série fica travada e o estado ativo é conferido de novo, contra a corrida com a suspensão.
  - **Recusas:** `return_before_departure` em `return_time`; `invalid_request` em `from_on` no passado.
  - **Suspensão:** não se desfaz. Para retomar, a central cria uma série nova.
- **T34 — Sem sessão:** rota sem sessão responde 401, não 403, como no resto da API. A conferência fica para a revisão final do api.
- **T35 — Motorista:**
  - o §11 ganha `transport.trip_started` e `transport.attendance_corrected` (só ids);
  - `my_trips` mostra só viagens `confirmed`, `in_progress` e `done`, nunca `planned`;
  - o `start` reconfere a CNH (validade na data e categoria) e recusa com `driver_license_expired` ou `driver_license_category`;
  - correção ou presença fora da situação certa → `invalid_transition`; `attendance_pending` com path `passengers`.
- **T36 — Série e outra origem:** se o paciente já é passageiro ativo, por qualquer origem, de uma viagem não cancelada no mesmo dia, o job da série não cria nem usa outra viagem para ele naquele dia. Corrige na onda final.
- **T37 — Laudo e comissão:**
  - **Fora do escopo:** escrita em caso alheio dá 404, seja de autor alheio, seja de rascunho visto pela comissão (como a leitura).
  - **Prefill:** lê só os problemas da consulta finalizada, e "Pedir TFD" aparece só em consulta finalizada.
  - **Paciente:** PATCH que muda `patient_id` → `invalid_request`.
  - **Prestador:** `appointment` sem `provider_id` mantém o prestador do laudo.
  - **Autorização:** `authorize` recusa `appointment_at` no passado com `appointment_required`.
  - **Noites internado:** são a interseção de `[from, to)` com `[data da ida, data da volta)`; internação aberta na volta conta até a data da volta.
  - **Eventos:** o §11 ganha `tfd.case_created`, `tfd.case_updated` e `tfd.case_cancelled` (só ids).
  - **CID:** o normalizador de CID é compartilhado (`ClinicalTerms.cid10_code`).
- **T38 — Trechos e ajuda de custo:**
  - **Trecho de frota:** o índice único é por `(trip_passenger_id, direction)`, então uma linha de passageiro carrega ida, volta ou as duas (van no mesmo dia, ou viagem partida em A e B). A data do trecho sai da viagem: a ida usa `trip.date` e a volta a data local de `return_at`.
  - **Alimentação:** a do par só é calculada quando o segundo trecho fica `done`, em qualquer ordem.
  - **Acompanhante:** recebe alimentação só se viajou na ida do par. O deslocamento é por trecho. A regra entra no gate do manual da UF.
- **T39 — Pareamento, pagamento e comprovante:**
  - **Trechos:**
    - `leg_in_future` (422) para trecho de passagem com data futura marcado como feito;
    - a volta de frota pareia primeiro com a ida da mesma linha de passageiro; senão vale o T10.
  - **Pagamento:** devolve 200 com o `<tfd-case>`. 201 fica só na criação do caso.
  - **Eventos:** o §11 ganha `tfd.accountability_closed` (só ids).
  - **Comprovante:**
    - baixar grava a trilha `clinical_record.viewed` (acesso `tfd`);
    - o token de `/tfd/files/<token>` e de `/exams/files/<token>` é mascarado no log de requisições;
    - o link curto vale 5 minutos e depois responde 410.
  - **Exclusão LGPD:** o comprovante e o nome do arquivo ficam **retidos** com o processo (prestação de contas, Portaria 55 arts. 15–16; §12), não são apagados.
