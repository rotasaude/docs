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

**Linha do tempo.** Fonte `urgency`, que entra em `Timeline::KINDS` (cidadão e profissional) entra também na página do PEC externo, na forma profissional, e fica fora do LEDI (Q27).

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

## 14. Reconciliação dos planos (2026-11-05)

**Regra:** onde um plano diverge desta seção, vale esta seção. As origens são: A = api, D = dashboard, C = contracts, W = wpda, M = maintenance.

**Q1 — Schema do protocolo** (C1–C6, D13, A4, A19, A20). Prevalece a versão validada do plano do contracts.
- **Estrutura:** `levels` é uma lista ordenada (`prefixItems`) com as 5 cores, cada uma com `max_wait_minutes` inteiro de 0 a 1440. `classification_rules` é uma lista separada de `risk_rules`, e cada regra é `{id, when, color, criterion}`, com `criterion` de até 120 caracteres e no máximo 100 regras.
- **Validação no gate do api:**
  - `id` único → `rule_id`;
  - tempos máximos fora de ordem → `levels_out_of_order`;
  - um protocolo ativo por cidade, conferido pelo `kind`. Não há nome reservado.
- **Avaliação:** quando várias regras casam, vale a cor mais grave; no empate, a primeira regra da lista. As variáveis do `when` são `vitals.*`, `pain_scale`, `profile.age` e `profile.sex`. A queixa não entra na condição.
- **Sugestão:** `POST /urgency/suggest` exige `encounter_id`, por causa da idade e do sexo. Devolve `{suggested: {color, criterion, rule_id} | null, levels}`.
- **Fora do 32a:** o simulador no editor.

**Q2 — Classificação só por enfermeiro** (A14, D21). Só o CBO 2235\* com vínculo ativo na UPA classifica, conforme a COFEN 661/2021. O técnico (3222) recebe 403 `cbo_not_allowed`. Isso corrige o §1.

**Q3 — Dor** (D11). A dor vai em `pain_scale`, de 0 a 10, fora de `vital_signs`. `vital_signs` usa o formato do 18, sem o `pain_score`.

**Q4 — Paciente** (A1, D1).
- `patients` ganha `unidentified` (bool), indexado. A coluna é necessária porque `no_cpf_reason` é cifrado e não dá para buscar nele.
- Paciente novo usa o `NewPatient` do 24a (`full_name`).
- `person_lookup` responde 200 com `patient: null` e `candidates: []`.

**Q5 — Fusão** (A16; amplia R4). Na identificação com paciente existente, passam para o paciente real:
- o encontro, a consulta e os documentos;
- os exames;
- o BPA;
- **a lista de problemas e de medicamentos nascida das consultas do atendimento**, para não perder informação clínica.

O provisório fica com `merged_into_id` e nada é apagado. A guarda de arquitetura cobre toda tabela com `patient_id` (A9).

**Q6 — Fila e detalhe** (A5, A6, A17, D2, D4).
- **Fila:** cada item traz `attendance_id`, `last_call` e `max_wait_minutes`. A fila tem um grupo `in_care[]`.
- **Estouro:** é contado desde a 1ª classificação, com o tempo máximo gravado nela.
- **Detalhe:** `GET /urgency/encounters/:id` grava trilha. Classificação, identificação e desfecho devolvem essa mesma forma.

**Q7 — Chamada** (A3, A24, D6, D8, D19).
- **Caminhos:** os locais de chamada e o "chamar de novo" ficam sob `/attendance`, em `/attendance/attendances/:id/recall`.
- **Local:** `call_location_id` vai no corpo de `Call` e `CallNext`. A lista de locais traz `location_required`.
- **Senha:** o `ticket` também aparece no check-in e na fila do 13.
- **Médico num `walk_in`:** só chama encontro já classificado, e precisa de vínculo 225\*.

**Q8 — TV** (A12, A18, D7, D17, D18, D20).
- **Tempos por cor:** no 32a, só na UPA, com o flag da unidade e o do dispositivo ligados. A UBS mostra só as chamadas.
- **Forma das TVs:** inclui `paired` e `last_seen_at`.
- **Token:** fica no `localStorage` do aparelho, é guardado no servidor só como SHA-256 e deixa de valer só quando a TV é revogada. O pareamento e o board funcionam sem cookie e sem CSRF.
- **Tempos:** `wait_times[].median_minutes` é um número ou `"lt5_patients"`.

**Q9 — Senha por dia** (A13). Coluna `attendances.ticket_on` (data local). A senha é única por (unidade, `ticket_on`).

**Q10 — Atendimento e desfecho** (A8, A9, A10, A22, A23, D5, D14, M3).
- **Consulta:** o `kind` da consulta vem do encontro. Finalizar a consulta `urgency` usa `{}`, não aceita `outcome` e não fecha o atendimento. `consultation_options` oferece `kinds: ["urgency"]` ao CBO 225\* na UPA.
- **Quem registra o desfecho:** o médico registra qualquer desfecho. A recepção e a enfermagem só registram `left_without_care`.
- **Identificação no desfecho:** alta e transferência exigem paciente identificado. Óbito e evasão não.
- **Ligação com o 13:**
  - o 13 aceita `in_care → left` quando a evasão é registrada;
  - a `referral_note` recebe só "Retorno à UBS em até N dias";
  - `transferred` vira `referred` regulado.
- **Check-in numa UPA:** check-in por código ou por exceção com `urgency_care` ligado → 422 `urgency_reception_required`.
- **Fechamento:** `Attendances::Close` exige o desfecho só quando `urgency_care` está utilizável. Com o interruptor desligado, o fechamento segue o 13.

**Q11 — Contrarreferência** (A7, D10). `GET /urgency/counter_referrals?health_unit_id`, para a UBS, sem o motivo e sem evento novo. O dashboard mostra a lista "Retornos pedidos pela urgência".

**Q12 — BPA** (A11). A pendência do provisório aparece sem linha (`procedure_id: null`, `fix: "patient"`). A tela de pendências do faturamento aceita esse caso.

**Q13 — Cidadão** (A21, W1–W4).
- A entrada `urgency` do cidadão traz:
  - chegada, unidade e desfecho em texto simples, pela tabela de W3;
  - `return_guidance` sem o motivo.
- Ficam nulos `professional_name`, `status` e `document`.
- Só aparecem encontros fechados.
- **Duplicidade:** `AttendanceSource` não emite o `walk_in`, para a visita não aparecer duas vezes.
- **Óbito:** não aparece no histórico do cidadão. Ele seria visto por quem acompanha a pessoa, e esse não é o canal para essa notícia.

**Q14 — Maintenance** (M1–M3).
- Rótulos: "Urgência e emergência (UPA)" e "Painel de chamada na TV da sala de espera".
- O maintenance só traduz os nomes. Não há cascata entre os interruptores.

**Q15 — Detalhes do dashboard.**
- **Códigos de erro** (D12): o dashboard tem texto para `not_classified`, `invalid_pairing_code`, `destination_required`, `return_guidance_required`, `encounter_closed`, `not_unidentified`, `cbo_not_allowed`, `patient_not_found` e `invalid_ticket_prefix`. `problems[]` segue o formato do P23 do 31.
- **Limites de texto** (D16): o api define.
- **Painel do gestor** (A15, D9): a forma segue o plano do api.
- **Cor laranja** (D22): o tema não tem esse papel. O tom de urgente vem sempre acompanhado do texto "laranja". Um token novo é decisão do ciclo de interface.
- **Telas** (D): a mesa da UPA fica dentro do Atendimento. "Senhas, locais e TVs" fica no grupo "Urgência" e só aparece com `waiting_room_display`.

### Acréscimos da execução (2026-11-09)

Base: contracts `b0d1f87`, tag `protocols-v1.7.0`, publicada com autorização do usuário.

**Q16 — Nome e ativação do protocolo (C-3, C-4).**
- **Nome:** o gate recusa um `name` que já pertence a um protocolo de outro `kind`, com `name_kind_conflict`. Assim, ativar a urgência nunca desliga a triagem.
- **Ativação:** só um protocolo de urgência fica ativo por cidade, garantido por índice único parcial em `definition->>'kind'`. Ativar um novo rebaixa o ativo anterior do mesmo `kind`, como já acontece numa troca de versão.

**Q17 — Fusão com item repetido (C-10).**
- Se o paciente real já tem ativo o mesmo problema (código) ou o mesmo medicamento, a linha do provisório fica no provisório e não é movida.
- O retorno informa quantos itens foram movidos e quantos ficaram (`moved` e `kept_duplicates`), e a trilha `urgency.patient_identified` registra os dois números.
- O histórico guardado não muda.

**Q18 — Estouro após reclassificação (C-12).** O tempo conta desde a 1ª classificação. O limite é o `max_wait_minutes` gravado na classificação **vigente**. Quem passa de verde para laranja depois de 60 minutos de espera estoura na hora. Isso corrige a letra do Q6.

**Q19 — Detalhe sem acesso clínico (C-14).** Quando quem lê não tem acesso clínico, como a recepção ao identificar ou registrar evasão, o detalhe do encontro omite:
- `complaint`, `vital_signs`, `pain_scale`;
- `justification`, `criterion`;
- os campos cifrados do desfecho.

`patient.description` continua no detalhe. Todo detalhe com campo clínico ou com a descrição grava trilha.

**Q20 — Retornos à UBS (C-27).** Os itens de `counter_referrals` trazem `return_days`. Leem `citizen_verifier` e `health_professional` vinculados à UBS.

**Q21 — Fusão: receitas externas e colisões (Tasks 3 e 9).**
- **Receitas externas:** `external_prescriptions` do paciente provisório passa para o paciente real na fusão, com o mesmo destino dos documentos, porque a farmácia dispensa contra o paciente.
- **Colisão em chave única:** a regra do Q17 vale para qualquer linha que bateria em chave única. Isso inclui `walk_in` aberto e notificação aberta. A linha fica no provisório e entra em `kept_duplicates`.
- **Guarda de recifragem:** a exceção de fusão vale só para `exam_results` e `dispensations`. A trilha `clinical_record_openings` não é tocada.

**Q22 — Protocolo: nome e dor (Task 5; amplia o Q16).**
- **Nome do protocolo de urgência:**
  - Ao salvar o rascunho, o gate também confere `name_kind_conflict`. Assim, um rascunho de urgência não sobrescreve uma triagem nem as assinaturas dela.
  - O nome "acolhimento" é reservado à escuta do 18 e recusado para urgência.
- **Dor obrigatória:** em toda classificação, `pain_scale` (0–10) é obrigatório. Sem ele → 422 `pain_scale_required`. Assim, a dor nunca fica "não avaliada", e uma regra como `not gt pain_scale 0` não sugere azul por falta de dado. A semente e os exemplos não dependem do comportamento com valor ausente.

**Q23 — Paciente novo com conferência de duplicado (Task 6; corrige o Q4).**
- **Comandos:** `arrivals` e `identify` criam o paciente novo por `Patients::CreateReviewed` do 24a, como a farmácia, a vacina e a vigilância. A conferência usa o `match_key`, travado contra criação simultânea.
- **Duplicado suspeito:** a resposta é 409 `duplicate_suspected` (`path: new_patient`) com os candidatos. O dashboard mostra os candidatos, e a recepção escolhe um deles ou reenvia com `duplicates_reviewed: true`.
- **Urgência:** quem está em estado grave não espera a conferência. O caminho do não identificado (provisório) continua sem ela, e a identificação acontece depois.
- **Prévia da cor:** `POST /urgency/suggest` com `pain_scale` nulo devolve `suggested: null` e os `levels`.
- **Corpo da requisição:** `arrivals` e as demais escritas recusam chave desconhecida, como no P23 do 31.

**Q24 — Fila, chamada e exceção da fusão (Task 8; esclarece o Q21).**
- **Exceção da fusão:** a lista de exceção da guarda de recifragem passa a ser `exam_results`, `dispensations`, `external_prescriptions` e `external_prescription_items`, este último se tiver `patient_id`. O "só" do Q21 existe para proteger as trilhas: `clinical_record_openings` e as demais continuam recusadas.
- **Fila do 13 com a urgência desligada:** a trava da fila ignora `walk_in`. Assim, um `walk_in` na frente da fila não trava o "Chamar próximo".
- **"Chamar próximo" na UPA:** sem nenhum paciente classificado, a chamada cai para a fila do 13 sem `walk_in`. Os atendimentos abertos antes de ligar a urgência continuam podendo ser chamados.

**Q25 — Fusão com atendimento aberto e identificação depois do fechamento (Task 9).**
- **Dois atendimentos abertos:** se o paciente real já tem um atendimento `walk_in` aberto na mesma UPA, a identificação é recusada com 409 `encounter_open` antes de mover qualquer registro.
  - O `attendance` é a raiz do encontro, não um item do prontuário. Por isso não entra na regra de `kept_duplicates` do Q17 e do Q21.
  - Para resolver, fecha-se um dos dois atendimentos. Depois disso a fusão funciona normalmente.
- **Novo desfecho `registered_in_error`:** serve para o atendimento aberto por engano ou em duplicidade.
  - Não gera BPA.
  - Fica fora das contagens de evasão e de desfecho do painel.
  - Não aparece para o cidadão.
  - Recepção, enfermagem e médico podem registrá-lo, com motivo obrigatório de pelo menos 10 caracteres.
  - Fecha o atendimento do 13 como `left`.
- **Identificação de encontro fechado:** é permitida. Um óbito sem identificação, por exemplo, pode ser identificado no dia seguinte (Review Focus 5). A regra de vez única continua valendo.
- **Ajustes internos aprovados:**
  - a ordem de lock é attendance → paciente;
  - `urgency_encounters.provisional_patient_id` passa a ser durável.
- **Pendência de ciclo:** gravadores de outros módulos aceitam o `patient_id` de um paciente já fundido. Vira card no board #2 no fechamento, aberto só com autorização do usuário.

**Q26 — Desfecho e contrarreferência (Task 11).**
- **Recepção e a lista de retornos:** `counter_referrals` para `citizen_verifier` não é filtrada por unidade, assim como a fila do 13. Restringir a leitura por unidade mudaria o 13 também, então fica como pendência de ciclo, no mesmo card da recepção sem vínculo de unidade.
- **`walk_in` com a urgência desligada:** um `walk_in` aberto antes de desligar `urgency_care` e encaminhado à UBS não gera pedido do 17 nem contrarreferência. Fica como pendência de produto, de risco baixo: com o interruptor desligado, não nascem `walk_in` novos.
- **Texto da `referral_note`:** concorda o número com o substantivo ("Retorno à UBS em até 1 dia", "em até 7 dias").
- **`NON_CARE_OUTCOMES`:** é o gancho do `registered_in_error` no BPA (Task 13), na linha do tempo (14) e no painel (15).

**Q27 — Linha do tempo: PEC externo e resumo do cidadão (Task 14; corrige o §1 e o §10).**
- **PEC externo:** quem usa a página da linha do tempo no PEC (ADR 0035) é o profissional da atenção básica, e a passagem pela UPA importa para a continuidade do cuidado. Por isso:
  - a fonte `urgency` **entra** na página do PEC, na forma profissional: data, unidade, desfecho, cor e CID;
  - também entram as linhas de BPA e os documentos do `walk_in`;
  - não aparecem `registered_in_error` nem o paciente provisório.

  Sair do PEC era decisão errada do §1 e foi retirada. A urgência continua fora do **LEDI**, porque a UPA não é atenção básica.
- **Link da entrada profissional:** a entrada aponta para a consulta do 19a (`{type: "consultation"}`), que é o único caminho até a nota da UPA.
- **Resumo de saúde do cidadão** (`Timeline::CitizenSummary`):
  - ignora os problemas nascidos de consulta `urgency`, para manter o "sem CID" do §10 e do Q13;
  - um problema crônico identificado na UPA chega ao cidadão quando a UBS o registra.
- **Visão do cidadão:** só mostra um encontro quando ele é elegível. Nada aparece por padrão.

**Q28 — Painel e exclusão (Tasks 15 e 16; os pontos 1 e 2 da exclusão aguardam o usuário).**
- **Evasão no painel:** conta só `left_without_care`. `left_against_advice` aparece só em `by_outcome`.
- **% acima do tempo máximo:** o cálculo inclui quem foi chamado e quem saiu sem atendimento depois de classificado. Quem saiu depois de estourar o tempo entra como estouro. Sem isso, o número sairia baixo.
- **Descrição física na exclusão:** é apagada quando o provisório foi identificado como paciente novo e passou a ter CPF.
- **Provisório fundido:** a linha fundida mantém a descrição física, que não tem nome nem CPF. Isso está aceito e documentado.
- **Recusa nova:** 409 `patient_in_urgency` quando a pessoa ainda espera na UPA, com `walk_in` aberto e sem classificação. O pedido de exclusão fica pendente, como no `patient_on_trip`. O dashboard tem o texto da recusa.

**Q29 — Lista de provisórios para identificar (conferência prévia do dashboard).**
- **Rota nova no api:** `GET /urgency/provisional_encounters?health_unit_id`, para a recepção da UPA. Lista os encontros com paciente provisório ainda não identificado, abertos ou fechados nos últimos 7 dias.
- **Cada item:** `{encounter_id, attendance_id, arrived_at, closed_at, outcome_kind, patient: <patient-ref com description>}`. Nenhum dado clínico.
- **Trilha:** a leitura grava trilha, porque leva a descrição física.
- **Objetivo:** permite identificar, no dia seguinte, o óbito de alguém sem documento (RF5, Q25). Entra como task extra do api, antes da suíte completa.
- **Dashboard:** aplica as decisões da conferência prévia como estão. A Task 13 do dashboard ajusta o `PendingPanel` do faturamento para a linha sem `procedure_id` (Q12).

**Q30 — Limite do "registrado por engano" (revisão final do api; restringe o Q25).** `registered_in_error` só é aceito enquanto o encontro não tiver consulta `urgency` finalizada. Com a consulta finalizada, a resposta é 422 `consultation_finalized`, e o caso fecha com outro desfecho registrado pelo médico. Assim, um cuidado que aconteceu não pode ter o BPA apagado nem sumir da visão do cidadão, do painel e do PEC. Na screening do 18, `POST /attendance/screenings/suggest` num `walk_in` de UPA responde 422 `screening_not_in_upa`.

**Q31 — Alcance da exclusão (decisão do usuário, 2026-11-09; fecha os pontos 1 e 2 do Q28).**
- **Ponto 1, opção A:** o `Citizens::Erase` passa a apagar o nome completo, o nome social, o nome da mãe, o nascimento, o sexo e o CNS de todo paciente não retido, em qualquer módulo. São os mesmos dados que o par do cidadão já perdia. O ADR 0026 recebeu uma nota de revisão com essa mudança. A retenção continua tudo ou nada por pedido.
- **Ponto 2, opção B:** o CPF do paciente excluído continua como está no 32a. A correção fica para uma rodada própria de LGPD, no card rotasaude/api#131 (board #2, Alta, antes do go-live).
- **Suíte completa:** liberada.
