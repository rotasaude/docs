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
