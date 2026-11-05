# Módulo 32a — Contratos entre apps

**Spec:** `superpowers/specs/2026-11-05-module-32-urgency-design.md` (decisões 1–23 e R1–R7) · **ADR:** `adr/0050.md` · **Pesquisa:** `pesquisa/2026-11-05-urgencia-upa-32.md`

Este contrato é a **fonte única** para os planos do contracts, api, dashboard, wpda e maintenance. Quando um plano divergir dele, a divergência vai para a seção final "Divergências propostas ao contrato" do próprio plano, e o coordenador reconcilia aqui, na §14. Este documento cobre só o **32a**; o 32b terá contrato próprio.

**Bases:** api `34ce852f`, dashboard `3671781`, wpda `54b44bf`, maintenance `ec86846`, contracts `protocols-v1.6.0`.

**Valores fixos:**

| Item | Valor |
|---|---|
| Porta do api | 3054 |
| Vite do dashboard | 5213 |
| Vite do wpda | 5214 |
| maintenance | 5177, com o serviço do compose parado |
| Migrações | a partir de `20261105900001` |
| Branch | `feat/mod-32a-urgency` (no contracts, `feat/protocols-urgency`) |
| Worktree do api | `apps/api/.claude/mod32a` |
| Worktrees dos fronts e do contracts | `rota-saude/.worktrees/<repo>-mod32a` |

**Mudança transversal.** O 32a toca `attendances` (13), `patients`, `consultations` (19a), `performed_procedures` (22), o motor de protocolo (18) e a linha do tempo. A suíte completa do api é obrigatória antes do merge.

## 1. Nomes no código

**Interruptores:**
- `urgency_care`, que requer `feature:clinical_record`;
- `waiting_room_display`, independente.

Desligados, as rotas respondem 403 `feature_disabled`. O 32b acrescenta `urgency_stock_deduction`.

**Unidade.** Vale para `health_units.kind = "upa"`, que já existe. Duas colunas novas:
- `ticket_prefix`: de 1 a 3 letras maiúsculas, padrão `"U"` na UPA e `"A"` nas demais;
- `show_wait_times`: bool, padrão `true` na UPA e `false` nas demais.

A UPA não faz a escuta do 18. `Screenings::Scope` e `ActiveProtocol` ignoram unidade `upa`, e a escuta numa UPA responde 422 `screening_not_in_upa`.

**Papéis.** Nenhum papel novo; os existentes e o vínculo com a unidade (`professional_links`) bastam:

| Ação | Papel e CBO |
|---|---|
| Recepção | `citizen_verifier` |
| Classificação | `health_professional` com CBO 2235\* ou 3222\* |
| Atendimento médico | `health_professional` com CBO 225\* |
| Configuração | `municipal_admin` |

**Tabelas novas da cidade:**
- `urgency_encounters`: 1:1 com `attendance`, `consultation_id` opcional, estado `waiting_classification` | `classified` | `in_care` | `closed`;
- `urgency_classifications`: só acréscimos;
- `urgency_outcomes`: 1:1 com o encontro;
- `call_locations`;
- `attendance_calls`: só acréscimos;
- `waiting_room_displays`.

**Mudanças em tabelas existentes:**

| Tabela | Mudança |
|---|---|
| `attendances` | `origin_kind` (`triage`, `appointment`, `walk_in`), que substitui o CHECK `ck_attendances_origin` por um que exige o par certo |
| `attendances` | `check_in_method` ganha `urgency_reception` |
| `attendances` | `patient_id`, obrigatório em `walk_in` |
| `attendances` | `citizen_id` aceita nulo só em `walk_in` |
| `attendances` | `ticket` (texto), único por (unidade, dia local) quando presente |
| `patients` | `merged_into_id` |
| `patients` | `no_cpf_reason` ganha o valor `"unidentified"` |
| `patients` | `unidentified_description` (cifrado) |
| `consultations` | `kind` ganha `urgency`; CHECKs de `care_type` e dos campos dentais ajustados |
| `performed_procedures` | `source` ganha `urgency`, com `urgency_encounter_id` |

**Linha do tempo.** Fonte `urgency`, que entra em `Timeline::KINDS` (cidadão e profissional) e fica fora do PEC externo e do LEDI.

**Cores.** `red`, `orange`, `yellow`, `green`, `blue`, nessa ordem de prioridade. Tempos máximos padrão, em minutos: 0, 10, 60, 120 e 240.

## 2. Formas

- **Erros:** `{error, path}` mais `problems[]`, no padrão do 31 (P23).
- **Datas:** ISO 8601, no fuso da cidade.
- **Referência a paciente:** `<patient-ref>` = `{patient_id, name, cns_masked, unidentified: bool, description?}`. `description` só aparece para não identificado, e só no detalhe.
- **Sem nome em URL:** a busca de pessoa é sempre `POST` com corpo.

## 3. Protocolo (contracts → api)

- **contracts:**
  - `protocols/schema.json` ganha `kind: "urgency_classification"`, com `levels` (as 5 cores e `max_wait_minutes` de cada uma) e as regras no mesmo formato de condição do 18, que sugerem a cor e o `criterion` (texto curto que explica por que aquela cor).
  - Exemplos válidos e inválidos.
  - Tag `protocols-v1.7.0`.
- **api:**
  - O motor de protocolo assinado do 18 aceita o kind novo, com rascunho, revisão, assinatura e ativação.
  - Há um protocolo ativo de urgência por cidade.
  - `Urgency::Suggest` recebe a queixa, os sinais vitais e a dor, e devolve `{color, criterion, rule_id}`.
  - Sem protocolo ativo, a classificação é manual, `suggested: null`, com justificativa obrigatória.

## 4. Chegada (recepção, `citizen_verifier`)

**Busca de pessoa.** `POST /urgency/person_lookup {cpf | cns}`, no padrão do 29/30/31. Devolve `{patient: <patient-ref> | null, candidates: []}` e grava trilha.

**Chegada.** `POST /urgency/arrivals {health_unit_id, patient_id? , new_patient?: {name, birth_date, sex, cpf?, cns?, no_cpf_reason?}, unidentified?: {description}}`.
- Vale exatamente uma das três formas: paciente existente, paciente novo ou não identificado.
- Unidade que não é UPA → 422 `not_an_upa`.
- CAPS → 422 `caps_unit_not_allowed`, pelo P32 do 31.
- Na mesma transação, cria o `Patient` (novo ou provisório), o `attendance` (`walk_in`, `waiting`, `ticket` se a cidade tiver `waiting_room_display`) e o `urgency_encounter` (`waiting_classification`).
- O mesmo paciente com encontro aberto na mesma UPA → 409 `encounter_open`.
- Resposta: `{attendance_id, encounter_id, ticket | null, patient: <patient-ref>}`.

**Identificação.** `POST /urgency/encounters/:id/identify {patient_id | new_patient}`, só para paciente provisório.
- Se o paciente escolhido já existe, os registros mudam de dono (R4) e o provisório recebe `merged_into_id`.
- Se for novo, o provisório recebe os dados e deixa de ser `unidentified`.
- Uma vez só: já identificado → 409 `already_identified`.
- Grava a trilha `urgency.patient_identified`, com os ids.

## 5. Classificação (enfermagem)

`POST /urgency/encounters/:id/classifications {complaint, vital_signs: {…}, pain_scale 0..10, color, justification?}`.
- Os sinais vitais usam o mesmo formato e os mesmos limites do 18 (`Screenings::VitalSigns`).
- A resposta traz `suggested: {color, criterion} | null`.
- Cor diferente da sugerida exige `justification` com 10 caracteres ou mais; sem ela → 422 `justification_required`.
- A primeira classificação leva o encontro a `classified`. As seguintes são reclassificações, só acréscimos. A cor vigente é a última.
- Prévia sem gravar: `POST /urgency/suggest`.

## 6. Fila e chamada

**Fila.** `GET /urgency/queue?health_unit_id` devolve dois grupos:
- `awaiting_classification[]`: `{encounter_id, ticket, patient: <patient-ref>, arrived_at, waited_minutes}`;
- `by_color[]`: os mesmos campos mais `color` e `overdue: bool`, ordenados por cor e chegada.

Nenhum texto clínico. A fila não grava trilha, porque é polling.

**Chamada.**
- `Attendances::CallNext` na UPA segue a ordem por cor. Quem ainda não foi classificado não é chamado pelo médico.
- O enfermeiro chama a classificação por `POST /urgency/encounters/:id/call_for_classification {call_location_id?}`. A chamada não muda o estado do `attendance`.
- **Local da chamada:** com `waiting_room_display` ligado e TV ativa na unidade, `call_location_id` é obrigatório em `Call`, `CallNext` e na chamada de classificação. Sem ele → 422 `call_location_required`.
- Cada chamada grava `attendance_calls {attendance_id, ticket, call_location_id, kind: "classification"|"care", called_at, by}`.
- "Chamar de novo": `POST /attendances/:id/recall {call_location_id}` grava outra chamada sem mudar o estado.

## 7. Atendimento médico e desfecho

**Atendimento.** O médico é chamado → a consulta do 19a é aberta com `kind: "urgency"`, presa ao `attendance`. Rascunho, finalização, adendos, CID, exames do 21, documentos do 19 e sugestão de notificação do 25 seguem o 19a sem mudança.
- O paciente provisório é atendido normalmente.
- A consulta `urgency` fica fora do LEDI e da produção da APS.

**Desfecho.** `POST /urgency/encounters/:id/outcome {kind, destination?, vacancy_request?, return_guidance?: {days, reason}}`.
- Valores de `kind`: `discharged`, `discharged_to_primary_care`, `transferred`, `left_without_care`, `left_against_advice` e `death`.
- `transferred` exige `destination`.
- `discharged_to_primary_care` exige `return_guidance`.
- `left_without_care` vale em qualquer estado antes do fechamento.
- Os demais exigem a consulta finalizada; sem ela → 422 `consultation_not_finalized`.
- O desfecho fecha o `attendance` com o `outcome` equivalente do 13:

| Desfecho | `outcome` do 13 |
|---|---|
| `discharged`, `death` | `discharged` |
| `discharged_to_primary_care` | `referred`, com `referral_note` = a orientação |
| `transferred` | `referred` (regulado) |
| `left_*` | `left` |

**Contrarreferência.** No `discharged_to_primary_care`, a UBS de referência do paciente (território do 11/24a, `CareLines::TeamPopulation`) recebe um aviso interno com `{encounter_id}`. A linha do tempo profissional mostra a orientação.

**Fechamento pelo 13.** `Attendances::Close` numa UPA exige o desfecho do 32. Sem ele → 422 `urgency_outcome_required`.

## 8. TV da sala de espera (`waiting_room_display`)

**Administração** (`municipal_admin`, com step-up):
- `GET/POST/PATCH /attendance/call_locations {health_unit_id, name, active}`;
- `GET/POST/PATCH /waiting_room/displays {health_unit_id, name, active, show_wait_times}`;
- `POST /waiting_room/displays/:id/pairing_code`: código de 6 dígitos que vale 10 min;
- `POST /waiting_room/displays/:id/revoke`.

**TV.**
- **Pareamento:** `POST /waiting_room/pair {code}` → `{display_token}`. O token é longo, guardado com hash e revogável.
- **Leitura:** `GET /waiting_room/board` com o cabeçalho `X-Display-Token`, sem cookie de sessão:

  ```
  {unit: {name}, calls: [{ticket, location, kind, called_at}] (últimas 8, até 30 min), wait_times: [{color, median_minutes | "lt5_patients"}] | null, updated_at}
  ```

  - Nada de nome, CPF, cor da pessoa ou queixa.
  - Os tempos são a mediana de espera dos pacientes ainda esperando, por cor. Com menos de 5 pacientes numa cor, sai `"lt5_patients"`.
  - Não grava trilha.
  - Token revogado → 401 `display_revoked`.
- **Dashboard:** rota `/#/tv` em tela cheia, fora do shell. Faz polling a cada 5 s, com `clearTimeout` ao desmontar. Pede o código no primeiro acesso.

## 9. BPA (22)

O evento `urgency.encounter_closed` dispara o job, que grava com `Billing::Record.insert!` e `source: "urgency"`:
- o **acolhimento com classificação de risco**, `config/urgency/sigtap.yml` → `classification_code` (padrão `0301060118`), com o profissional da primeira classificação;
- o **atendimento médico**, `care_code` (padrão `0301060096`), com o médico da consulta.

Regras:
- Os códigos são gate: o arquivo tem `review: true` até a conferência no SIGTAP.
- O instrumento e o CBO vêm da SIGTAP.
- Paciente provisório não identificado não gera linha e fica em `pending` com `fix: "patient"`.
- Evasão antes da classificação não gera linha.

## 10. Linha do tempo, wpda e vigilância

**Linha do tempo.** `Timeline::Sources::UrgencySource`:

| Visão | O que mostra |
|---|---|
| Cidadão | "Atendimento de urgência — <UPA>", data e hora, o desfecho em linguagem simples e a orientação de retorno. Sem CID, cor ou evolução |
| Profissional | também a cor vigente e o CID |

- Paciente provisório não aparece para o cidadão. Depois da identificação, o atendimento aparece no histórico do paciente real.
- **wpda:** nenhuma tela nova. "Meu histórico" recebe o kind `urgency`, e o wpda só precisa do rótulo.
- **Vigilância:** a sugestão de notificação compulsória vem da consulta do 19a (25), sem nada novo.

## 11. Painel do gestor

`GET /urgency/dashboard?month&health_unit_id?`, para `municipal_admin` e `analyst`. Traz só contagens, com `"lt5"` de 1 a 4:
- tempo da porta à classificação e da classificação ao médico, por cor (mediana e p90);
- % de pacientes acima do tempo máximo, por cor;
- atendimentos por cor e por desfecho;
- evasão;
- permanência.

## 12. Eventos (só ids)

| Evento | Payload |
|---|---|
| `urgency.arrived` | `encounter_id` |
| `urgency.classified` | `encounter_id`, `classification_id` |
| `urgency.reclassified` | `encounter_id`, `classification_id` |
| `urgency.patient_identified` | `encounter_id`, `patient_id`, `merged_from_id?` |
| `urgency.encounter_closed` | `encounter_id` |
| `attendance.recalled` | `attendance_id`, `call_id` |
| `waiting_room.display_paired` | `display_id` |
| `waiting_room.display_revoked` | `display_id` |

Todo `Platform.audit` novo entra em R18.

## 13. LGPD, imutabilidade e ordem de entrega

**Cifra.** Ficam cifrados a queixa, a justificativa, `unidentified_description`, `return_guidance.reason`, `destination` e `vacancy_request`.

**Nunca em URL, log ou evento:** CPF, CNS, nome, queixa, CID e descrição.

**Imutabilidade** (trigger em `db/city_triggers.sql`): classificações, desfecho, chamadas e o `merged_into_id` depois de gravado.

**Exclusão (ADR 0026):**
- O encontro com classificação ou consulta é retido.
- O `walk_in` com evasão antes da classificação segue a triagem sem atendimento.

**Ordem de entrega:** contracts (tag) → api → dashboard → wpda → maintenance.
