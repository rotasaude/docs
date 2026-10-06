# Módulo 17 — Agenda dos profissionais — design

**Data:** 2026-10-05
**Status:** aprovado (2026-10-05)
**Afeta:**
- `contracts`: schema de protocolo `protocols-v1.5.0` (`scheduling`).
- `apps/api`:
  - cidade: `appointment_types` (base copiada), `schedule_templates`, colunas novas em `appointments`, `appointment_requests`, `professional_shifts`, `professional_links`; extensão `btree_gist` e trava de sobreposição;
  - `Scheduling::Availability`, `Appointments::Book`, `Appointments::FitIn`, pedido gerado pela triagem, `Appointments::RequestReschedule`, `Appointments::RemindJob`;
  - rotas em `/attendance`, `/professionals`, `/citizen/appointments`, `/citizen/notices`.
- `apps/dashboard`: tipos e modelos (Profissionais), fila/marcação/encaixe/agenda da unidade (Atendimento), Minha agenda, regras de agendamento no editor de protocolo.
- `apps/wpda`: pedido no resultado da triagem e em "Meus horários", "Não posso nesse horário", lembrete na caixa de avisos.
- `apps/admin`, `apps/maintenance`: nada.

**ADR:** `docs/adr/0029.md` · **Módulo:** `docs/modulos/17--agenda.md`

**Fora desta entrega:** cidadão escolhendo vaga, semana-padrão, agenda coletiva, lista de espera por cancelamento, indicadores de absenteísmo.

## 1. Ponto de partida

- **`appointments`** (módulo 08, ADR 0019): `request_id` obrigatório, `health_unit_id`, `scheduled_at`, `status` (`scheduled`, `confirmed`, `checked_in`, `cancelled_by_citizen`, `expired`, `no_show`, `moved`), `confirmation_deadline_at`, `cancel_reason` (≥ 10 caracteres quando `cancelled_by_citizen`), `moved_from_appointment_id`. Sem profissional, sem tipo, sem fim.
- **`appointment_requests`**: nasce de atendimento (`origin_attendance_id` obrigatório), `kind` retorno/encaminhamento, `target_unit_id`, `status` `open`/`scheduled`/`closed`.
- **Jobs**: `ExpireUnconfirmedAppointmentsJob`, `MarkNoShowAppointmentsJob`, `SendConfirmationRemindersJob` (lembrete de prazo de confirmação, F-08.7).
- **Turnos** (`professional_shifts`, ADR 0021): por vínculo, instantes, até 24 h, cancelamento com motivo; vínculo (`professional_links`) com unidade e CBO.
- **Caixa de avisos** (`/citizen/notices`): hoje só lê `campaign_recipients`.
- **Unidade de referência**: `Territory::ReferenceUnits.for(neighborhood_id)`.
- **Linguagem de condição e construtor** (ADR 0027): `outcome.*`, `profile.*`, `gte`/`lte`.

## 2. Decisões desta conversa

| # | Decisão |
|---|---|
| 1 | Agenda ligada à triagem; quem marca é sempre a recepção; o cidadão não agenda. |
| 2 | Vagas a partir do turno, com modelo de agenda opcional. |
| 3 | O protocolo decide, por resultado, se gera pedido (tipo, prioridade, prazo) ou só orienta. |
| 4 | Faixas do modelo: demanda do dia, agendável pela recepção, bloqueada. |
| 5 | Vaga ou encaixe (justificativa + limite por turno). |
| 6 | Tipos de atendimento: base da plataforma ampliada pela cidade, com CBOs. |
| 7 | Cidadão pode pedir outro horário; lembrete do horário confirmado (wpda + SMS). |
| 8 | Vagas calculadas, trava de sobreposição no banco. |
| 9 | Transição por unidade: marcação livre só em dia sem turno. |
| 10 | `municipal_admin` edita tipos e modelos; o profissional vê a própria agenda. |

## 3. Tipos de atendimento e modelos

### 3.1 Tipos
- Base em `config/scheduling/appointment_types.yml`: `consulta_medica` (20 min, grupo CBO 2251/2252/2253), `consulta_enfermagem` (15 min, 2235), `consulta_odontologica` (30 min, 2232), `retorno` (15 min, todos os grupos acima). Os grupos são prefixos de CBO.
- `appointment_types` (cidade): `key` (único, `^[a-z][a-z0-9_]{1,40}$`), `name`, `duration_minutes` (5–240), `cbo_prefixes` (array de texto), `active`, `origin` (`platform`|`city`). A migração e o provisionamento copiam a base; tipo `platform` não muda `key` nem `origin`.
- `Scheduling::AppointmentTypes.serves?(type, cbo_code)`.

### 3.2 Modelos
- `schedule_templates` (cidade): `name`, `fit_in_limit` (0–20), `blocks` (jsonb validado): `[{ "starts": "08:00", "ends": "10:00", "kind": "walk_in"|"bookable"|"blocked", "appointment_type_key"?, "slot_minutes"? }]`; faixas sem sobreposição; `bookable` exige tipo; `slot_minutes` padrão = duração do tipo; `active`.
- `professional_shifts.schedule_template_id` (nulo), `professional_links.default_appointment_type_key` (nulo).
- Padrão da cidade para turno sem modelo: `city_profile.default_fit_in_limit` (padrão 2).
- Pré-visualização: `Scheduling::TemplatePreview.call(template, sample_shift)` → vagas e faixas, sem gravar.

## 4. Vagas, marcação e encaixe

### 4.1 Cálculo (`Scheduling::Availability`, puro sobre dados carregados)
`Availability.for(unit:, from:, to:, appointment_type:)` → `[ { professional_id, shift_id, starts_at, ends_at } ]`:
1. turnos não cancelados da unidade que cruzam o período, cujo vínculo tem CBO servido pelo tipo;
2. faixas: do modelo do turno (horas no fuso da cidade, recortadas pelo turno); sem modelo, o turno inteiro como `bookable` do tipo padrão do vínculo → senão o da base pelo CBO → senão nenhuma vaga;
3. recorte das faixas `bookable` do tipo pela duração da vaga (sobra no fim da faixa é descartada);
4. remove vagas sobrepostas a horários ativos (`scheduled`, `confirmed`, `checked_in`) do profissional, de qualquer `booking_kind`, e vagas cujo início já passou.

### 4.2 Horário (`appointments`, colunas novas)
`professional_id`, `appointment_type_key`, `ends_at`, `shift_id`, `booking_kind` (`slot`|`fit_in`|`legacy`; existentes viram `legacy`), `fit_in_reason` (obrigatório e ≥ 10 caracteres quando `fit_in`), `reschedule_requested` (bool), `reminded_at`.
- `CHECK`: `slot`/`fit_in` exigem profissional, tipo, fim e turno; `legacy` não.
- `EXCLUDE USING gist (professional_id WITH =, tstzrange(scheduled_at, ends_at) WITH &&) WHERE (booking_kind = 'slot' AND status IN ('scheduled','confirmed','checked_in'))`.

### 4.3 Comandos
- `Appointments::Book.call(request:, professional:, starts_at:, type:, by:)`: confere que a vaga existe em `Availability`; cidadão sem outro horário ativo sobreposto (`citizen_busy`); grava `slot`; violação da trava → `slot_taken`.
- `Appointments::FitIn.call(request:, professional:, shift:, starts_at:, type:, reason:, by:)`: `shift.lock!`; início e fim dentro do turno; tipo serve o CBO; conta encaixes ativos do turno contra o limite (`fit_in_limit`); grava `fit_in`.
- Remarcar pela recepção: o fluxo existente (`moved`) passa a usar `Book`/`FitIn`.
- `legacy`: só quando a unidade não tem nenhum turno não cancelado que cruze o dia; senão `use_slots`.
- Turno cancelado: horários dele ficam; a fila mostra o pedido como "precisa remarcar" (`needs_reschedule`, derivado).

## 5. Pedido gerado pela triagem

### 5.1 Protocolo (`protocols-v1.5.0`)
```json
"scheduling": [ { "when": <condição>, "appointment_type": "consulta_medica",
                  "priority": "priority"|"routine", "due_in_days": 1..365 } ]
```
Até 10 regras; `when` usa `outcome.*`, `profile.*` e ids de passo; gate avisa tipo inexistente na cidade.

### 5.2 Pedido (`appointment_requests`)
- `kind` ganha `triage`; `origin_attendance_id` opcional; `origin_triage_id`; `CHECK` exatamente uma origem; `target_unit_id` opcional (fila "sem unidade"); `appointment_type_key`, `priority` (`routine`|`priority`), `due_on`; `reschedule_reason_code`, `reschedule_note`, `preferred_period` (`morning`|`afternoon`|`any`), `reschedule_count`.
- Pedidos de retorno/encaminhamento: tipo `retorno`, prioridade `routine`, `due_on` do que a recepção já usa hoje (data indicada no desfecho; sem data, +30 dias).
- `Triages::Schedule.call(triage, outcome)` na conclusão: urgente → nada; primeira regra que casa; unidade = primeira unidade de referência do bairro da triagem; sem unidade → `target_unit_id` nulo; pedido aberto do mesmo tipo para o cidadão → atualiza prazo (o menor) e prioridade (a maior) e registra `origin_triage_id` adicional em `appointment_request_triages`.
- Revogação: pedido de triagem `open` sem horário fecha com `closed_reason = consent_revoked`.

### 5.3 Fila
`GET /attendance/requests?unit_id=` ordena por atrasado, `due_on`, prioridade; marcas: `overdue`, `reschedule_requested`, `needs_reschedule`, origem. `GET /attendance/requests?unassigned=true` = fila "sem unidade"; `POST /attendance/requests/:id/assign_unit`.

## 6. Cidadão

- `POST /citizen/appointments/:id/reschedule_request` `{ reason_code: work|health|transport|other, note?, preferred_period }` — horário `scheduled`/`confirmed` antes do início; horário vira `cancelled_by_citizen` (`cancel_reason` = frase fixa "Remarcação pedida pelo cidadão") com `reschedule_requested = true`; pedido volta a `open` com motivo, período e `reschedule_count + 1`; prazo mantido.
- `Appointments::RemindJob` (por cidade, 17h no fuso da cidade): confirmados do dia seguinte sem `reminded_at`; marca `reminded_at`; SMS só com `campaigns_sms_enabled` e opt-in, texto fixo *"Secretaria de Saúde de {cidade}: você tem um compromisso de saúde amanhã. Veja em {link}"*, janela 8h–20h.
- Caixa de avisos: `/citizen/notices` passa a unir campanhas e lembretes (`kind: appointment_reminder`, com tipo, unidade, endereço, hora e profissional).
- Resultado da triagem: *"A Unidade X vai entrar em contato para marcar sua consulta. Prazo previsto: até DD/MM."*

## 7. Telas

- **dashboard / Profissionais** (`municipal_admin`): tipos (ajustar, desativar, criar), modelos (editor de faixas + pré-visualização), modelo no turno, tipo padrão no vínculo.
- **dashboard / Atendimento** (recepção): fila com marcas; marcar a partir do pedido (vagas por dia e profissional) e Encaixe (justificativa, contador); agenda da unidade por dia; fila "sem unidade".
- **dashboard / Minha agenda** (`health_professional`): dia e semana, só leitura.
- **dashboard / editor de protocolo**: painel "Agendamento" com o construtor de condições do módulo 15.
- **wpda**: pedido no resultado e em "Meus horários"; "Não posso nesse horário"; lembrete na caixa de avisos.

## 8. LGPD

Motivo, nota e justificativa fora de evento, log e Analytics; eventos só com ids; SMS de texto fixo e link sem identificador; revogação e exclusão seguem o ADR 0026 mais o fechamento do §5.2.

## 9. Testes

- `Scheduling::Availability`: tabela de casos (sem modelo com padrão do vínculo, pelo CBO, sem correspondência; modelo com três faixas; faixa fora do turno; sobra de faixa; CBO não servido; vaga passada; sobreposição; turno cancelado; virada de dia; fuso).
- Concorrência com threads: mesma vaga (trava); dois encaixes no último lugar do limite; cidadão sobreposto.
- Pedido da triagem: primeira regra, urgente, sem unidade, sem duplicar, tipo inexistente, revogação.
- Cidadão: reschedule (motivo, período, contagem, prazo mantido); lembrete (idempotência, janela, opt-in, chave).
- Transição `legacy`.
- Migração: recusa com sobreposição herdada, listando os ids.
- Invariantes (`spec/invariants/scheduling_invariants_spec.rb`): os do ADR 0029.
- Front: dashboard (editor + pré-visualização, marcação, encaixe, agendas, painel Agendamento) e wpda (botão, motivo/período, avisos), relógio fixo.
- Prova no navegador: modelo → vagas; triagem gera pedido; recepção marca vaga e tenta encaixe acima do limite; cidadão pede outro horário; lembrete na véspera.

## 10. Semente de dev

Base de tipos copiada; um modelo "Manhã: demanda do dia 7–9h, consultas 9–11h, bloqueio 11–12h" em Curitiba; turnos da semana para a médica e a enfermeira da semente; protocolo "Saúde do idoso" com regra de agendamento (rotina, 30 dias).

## 11. Rollout

1. `contracts` `protocols-v1.5.0` (push com autorização).
2. `api`: migração de cidade (verifica sobreposição herdada antes da trava); `VALID_RANGE` → `(1..29)`.
3. `dashboard` e `wpda` depois do `api`.
4. Compatível: unidade sem turnos segue marcando como hoje.

## 12. F-IDs (propostos)

| ID | Funcionalidade | Superfície |
|---|---|---|
| F-17.1 | Tipos de atendimento: base da plataforma e da cidade | api, dashboard |
| F-17.2 | Modelos de agenda e tipo padrão do vínculo | api, dashboard |
| F-17.3 | Vagas calculadas e marcação em vaga, com trava de sobreposição | api, dashboard |
| F-17.4 | Encaixe com justificativa e limite | api, dashboard |
| F-17.5 | Pedido de agendamento gerado pela triagem | contracts, api, dashboard, wpda |
| F-17.6 | Fila da recepção com prazo e prioridade, e fila "sem unidade" | api, dashboard |
| F-17.7 | "Não posso nesse horário" no wpda | api, wpda, dashboard |
| F-17.8 | Lembrete do horário confirmado (aviso e SMS) | api, wpda |
| F-17.9 | Agendas da unidade e do profissional | api, dashboard |

## 13. Riscos

- Horários sobrepostos herdados bloqueiam a migração até alguém decidir.
- Turno cancelado deixa horários "precisa remarcar" que dependem da recepção agir.
- Prazo do protocolo é previsão: fila longa pode atrasar sem aviso ao cidadão (indicador fica para o Analytics).
- Conflito de arquivos com os módulos 15 e 16 em execução (validação presencial, `city_schema.rb`, `adr_pointers_spec.rb`).
