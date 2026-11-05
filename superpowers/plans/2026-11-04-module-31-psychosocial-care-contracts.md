# Módulo 31 — Contratos entre apps

**Spec:** `superpowers/specs/2026-11-04-module-31-psychosocial-care-design.md` · **ADR:** `adr/0049.md` · **Pesquisa:** `pesquisa/2026-11-04-atencao-psicossocial-31.md`

Este contrato é a fonte única para os planos do api, do dashboard, do maintenance e do wpda. Divergência vai para a seção final "Divergências propostas ao contrato" de cada plano, e o coordenador reconcilia aqui (§15).

**Base:** api `dc1c87df`, dashboard `f6fb6e4`, wpda `f923cdc`, maintenance `90237b1`.

**Valores fixos:**

| Item | Valor |
|---|---|
| Porta do api | 3053 |
| Porta do Vite: dashboard | 5211 |
| Porta do Vite: wpda | 5212 |
| Porta do maintenance | 5177, com o serviço do compose parado |
| Migrações | a partir de 20261104900001 |
| Branch | `feat/mod-31-psychosocial-care` |
| Worktree do api | `apps/api/.claude/mod31` |
| Worktrees dos fronts | `rota-saude/.worktrees/<app>-mod31` |

**Mudança transversal.** O módulo toca o 19 (`clinical_documents`), o 22 (`performed_procedures`, `Billing::Critic`) e o SIGTAP da plataforma. A suíte completa do api é obrigatória antes do merge.

## 1. Nomes no código

**Interruptor:** `psychosocial_care`, que requer `feature:clinical_record`. Desligado, a resposta é 403 `{error: "feature_disabled"}`.

**Papéis:** entram em `Membership::ROLES` e em `ck_memberships_role`.
- `psychosocial_staff`: privilegiado (`PRIVILEGED_ROLES`), com segundo fator.
- `psychosocial_lead`: privilegiado.

`psychosocial_lead` implica as permissões de `psychosocial_staff`.

**Lotação:** a pessoa está lotada num CAPS quando tem `professional_links` ativo (`ended_at IS NULL`) com o `health_unit_id` do CAPS. O CBO da ação vem desse vínculo. Sem vínculo ativo no CAPS, a resposta é 403 `not_on_caps_team`.

**Unidade:** `health_units.kind` ganha `caps`. Colunas novas, nulas fora do CAPS (CHECK `kind = 'caps'` ⇔ `caps_modality IS NOT NULL`):

| Coluna | Valores |
|---|---|
| `caps_modality` | `caps_i`, `caps_ii`, `caps_iii`, `caps_i_child`, `caps_ad`, `caps_ad_iii`, `caps_ad_iv` |
| `service_classification` | string de 3 dígitos da classificação do serviço 115 |

`HealthUnit::NIGHT_MODALITIES = %w[caps_iii caps_ad_iii caps_ad_iv]`. Só esses CAPS têm leito.

**Tabelas da cidade** (prefixo `psychosocial_`):

| Tabela | Conteúdo |
|---|---|
| `referrals` | encaminhamento da UBS |
| `receptions` | acolhimento |
| `enrollments` | matrícula |
| `care_plans` | PTS, com versões |
| `groups` | grade semanal |
| `actions` e `action_participants` | ações e participantes |
| `monthly_records` | RAAS do mês |
| `month_closures` | fechamento do mês |
| `raas_exports` | remessa gerada |
| `matrix_supports` e `matrix_cases` | matriciamento |
| `beds` e `night_stays` | leitos e permanências |

**Fontes:**
- `performed_procedures.source` ganha `psychosocial`, com a coluna `psychosocial_action_id`.
- Linha do tempo: fonte `psychosocial` (§10).

**Domínios** (listas fechadas em `config/psychosocial/raas.yml`, com `review: true` até a conferência com o aplicativo RAAS vigente; os códigos são os da RAAS):

| Chave | Conteúdo |
|---|---|
| `origins` | origem do usuário |
| `destinations` | destino; `"00"` = permanência |
| `substances` | álcool, crack, cocaína, maconha, solventes, outras, nenhuma |
| `race_colors`, `ethnicities` | reaproveitar o que o cadastro do cidadão já tiver |
| `care_intensity` | `intensive`, `semi_intensive`, `non_intensive` |

## 2. Formas

- **Erros:** `{error, path}` mais `problems[]` no 422 de validação. Conflitos (409) seguem o padrão do 30.
- **Datas e horas:** ISO 8601, no fuso da cidade. Mês: `"YYYY-MM"`.
- **Referências a pessoas e coisas:**

| Referência | Forma |
|---|---|
| `<patient-ref>` | `{patient_id, name, cns_masked}`, sem CPF. Fora do CAPS, só `{patient_id, name}`. |
| `<pro>` | `{user_id, name, cbo_code}` |
| `<sigtap>` | `{code, name, instrument: "raas_psi"\|"bpa_c"\|"bpa_i"}` |

- **Texto clínico cifrado:** evolução, motivo do encaminhamento, PTS, texto compartilhado. Só sai em `GET` de detalhe para quem pode ler, nunca em lista.
- **Paginação:** cursor, como na linha do tempo.

## 3. Administração (`municipal_admin`, escrita com step-up)

- `POST`/`PATCH /health_units` aceita `kind: "caps"`, `caps_modality` e `service_classification`.
  - `caps_modality` só com `caps`: senão, 422 `caps_modality_mismatch`.
  - `service_classification` precisa de 3 dígitos: senão, 422 `invalid_service_classification`.
- `GET`/`POST`/`PATCH /psychosocial/beds`: leitos do CAPS, `{id, health_unit_id, label, active}`. Só para CAPS de `NIGHT_MODALITIES`: senão, 422 `caps_without_beds`.
- `PATCH /psychosocial/settings`, com `{night_limits: {caps_iii: {consecutive: 7, alternate: 10, window_days: 30}, caps_ad_iii: {total: 14, window_days: 30}, caps_ad_iv: {...}}, care_plan_review_days: 90}`. Gera o evento `psychosocial.settings_changed`.

## 4. Equipe do CAPS (`psychosocial_staff`, escopo = CAPS lotados)

Toda rota recebe `health_unit_id` do CAPS (corpo ou query) e confere a lotação.

**Encaminhamentos** (escrita pela UBS: `health_professional` com vínculo em unidade não CAPS):
- `POST /psychosocial/referrals {patient_id, health_unit_id (CAPS), reason, priority: "routine"|"priority"}`.
- `GET /psychosocial/referrals?health_unit_id&status=pending` (CAPS): lista sem `reason`. O detalhe traz `reason` e grava trilha.

**Acolhimento:** `POST /psychosocial/receptions {health_unit_id, patient_id, referral_id?, at, sigtap_code, notes, outcome: "enrolled"|"referred_elsewhere"|"no_follow_up"}`.
- `enrolled` exige o bloco `enrollment` (abaixo) e cria a matrícula na mesma transação.
- O código de acolhimento vem do SIGTAP com instrumento RAAS-PSI ou BPA-I. Código fora disso: 422 `instrument_mismatch`.

**Matrícula:**
- Criação: `POST /psychosocial/enrollments {health_unit_id, patient_id, started_on, origin, cid_main, cid_secondary[], substances[], esf_covered, esf_cnes?, reference_user_id}`.
  - Já existe matrícula aberta do mesmo paciente nesse CAPS: 409 `enrollment_open`.
  - `esf_cnes` é obrigatório se `esf_covered`, e é pré-preenchido pelo território do 11 quando houver.
- Leitura: `GET /psychosocial/enrollments?health_unit_id&status=open|closed&q=`. O detalhe `GET /psychosocial/enrollments/:id` grava `clinical_record.viewed`.
- Edição: `PATCH /psychosocial/enrollments/:id` aceita CIDs, substâncias, ESF, técnico de referência e `next_visit_at`. `origin` e `started_on` só são editáveis antes da primeira RAAS exportada.
- Encerramento: `POST /psychosocial/enrollments/:id/close {destination, closed_on}`.
  - `destination` diferente de `"00"`; caso contrário, 422 `destination_is_permanence`.
  - `closed_on` não pode ser futuro.
  - Encerrar abre a última RAAS do mês com o destino.

**PTS:**
- `POST /psychosocial/enrollments/:id/care_plans {needs, goals[{text, responsible_user_id, due_on?}], intensity, review_on}` cria a versão N+1. As versões são imutáveis.
- `GET /psychosocial/enrollments/:id/care_plans` devolve as versões. O PTS com `review_on` vencido gera o aviso interno `psychosocial.care_plan_review_due`, entregue pelo job diário no caixa da equipe, nunca ao cidadão.

**Grupos:** `GET`/`POST`/`PATCH /psychosocial/groups {health_unit_id, name, kind: "group"|"workshop"|"assembly", weekday, starts_at, ends_at, responsible_user_id, capacity, sigtap_code, active}`.

**Hoje no CAPS:** `GET /psychosocial/today?health_unit_id&date=` devolve os grupos do dia e as matrículas com `next_visit_at` nessa data. Só `<patient-ref>`, sem nada clínico.

**Ações:**
- Criação: `POST /psychosocial/actions {health_unit_id, sigtap_code, performed_at, place: "caps"|"territory", quantity, notes?, authorization_number?, group_id?, participants: [{enrollment_id} | {patient_id} | {family_name}]}`.
  - O profissional é o usuário atual, com o CBO do vínculo no CAPS.
  - **Crítica na hora**, em 422 com `problems[]`. Cada problema traz `code` e `participant_index` quando couber:

    | Código | Regra |
    |---|---|
    | `instrument_mismatch` | instrumento do código fora de RAAS-PSI, BPA-C ou BPA-I |
    | `cbo_incompatible` | CBO fora da lista do código |
    | `quantity_exceeded` | quantidade acima da máxima |
    | `performed_in_future` | data futura |
    | `month_closed` | mês fechado; só a coordenação corrige |
    | `enrollment_required` | ação RAAS-PSI sem matrícula aberta |
    | `authorization_required` | atributo 036 sem número de autorização |
    | `family_only_for_family_care` | familiar sem cadastro fora do código de atendimento familiar |

  - Ação `raas_psi` exige pelo menos um participante com matrícula. Ação `bpa_c` aceita zero participantes.
- Cancelamento: `POST /psychosocial/actions/:id/cancel {reason}`. A ação é imutável (trigger) e o cancelamento é a única mudança. Mês fechado exige `psychosocial_lead`; senão, 403 `lead_required`.
- Leitura: `GET /psychosocial/actions?enrollment_id` lista sem `notes`. O detalhe traz `notes` e grava trilha.

**Receita:**
- `POST /psychosocial/actions/:id/documents` abre o fluxo de documento do 19 e do 19d com `psychosocial_action_id`.
- Condições: ação individual com um participante matriculado e CBO médico (`225*`). Senão, 422 `prescriber_required`.
- Os tipos são os do 19c e do 19d, com as mesmas regras de assinatura e de SNCR.

## 5. Acolhimento noturno

- **Entrada:** `POST /psychosocial/night_stays {bed_id, enrollment_id, started_at, expected_end_at}`.
  - O banco recusa sobreposição no mesmo leito: 409 `bed_taken`.
  - Paciente com outra permanência aberta: 409 `already_staying`.
- **Limite**, configurado no §3 e contado sobre as noites na janela:
  - Passou do limite: 422 `night_limit_reached`, com `{used, limit}`.
  - `psychosocial_lead` passa do limite com `reassessment: {reason}`, que fica registrado e gera o evento `psychosocial.night_limit_overridden`.
- **Saída:** `POST /psychosocial/night_stays/:id/close {ended_at, reason}`.
- **Fechamento da noite:** cada noite fechada gera uma ação RAAS com o código de acolhimento noturno, em `config/psychosocial/raas.yml`, chave `night_sigtap_code`, a confirmar no SIGTAP (gate). O profissional é quem fecha.
- **Mapa:** `GET /psychosocial/beds/board?health_unit_id&date` devolve os leitos, o ocupante (`<patient-ref>`) e as noites usadas na janela.

## 6. RAAS do mês (`psychosocial_lead` fecha; `psychosocial_staff` lê)

- **`psychosocial_monthly_records`:** um por (matrícula, mês de atendimento). Estados `open` → `closed` → `exported`.
  - A continuidade nasce no job diário `Psychosocial::OpenMonthJob` (EachCityJob, idempotente) no 1º dia do mês, para toda matrícula aberta.
  - A foto da identificação e do atendimento é tirada no fechamento.
  - `homeless` (situação de rua no mês) é obrigatório por mês: `PATCH /psychosocial/monthly_records/:id {homeless}`.
- **Pendências:** `GET /psychosocial/months/:month?health_unit_id` devolve os contadores e `problems[]` por registro. São lidos, nunca gravados:

  | Código | Pendência |
  |---|---|
  | `patient_cns_missing` | CNS do paciente ausente |
  | `phone_missing` | telefone ausente |
  | `homeless_missing` | situação de rua não informada |
  | `professional_cns_missing` | CNS do profissional ausente |
  | `professional_not_in_cnes` | profissional fora do CNES local do 16 |
  | `cid_missing` | CID ausente |
  | `no_actions` | registro sem ação |
  | `action_out_of_rule` | ação fora da regra |

- **Fechamento:** `POST /psychosocial/months/:month/close {health_unit_id}`, com step-up.
  - Com pendência, recusa com 422 `month_has_problems`.
  - Grava `psychosocial_month_closures`, publica `psychosocial.month_closed` e trava as ações do mês (trigger).
- **Reabertura:** `POST /psychosocial/months/:month/reopen {health_unit_id, reason}`, com step-up. Só antes da exportação; depois dela, 409 `already_exported`.
- **Remessa:** `POST /psychosocial/raas_exports {health_unit_id, month}` gera a remessa.
  - A competência de apresentação é o `billing_period` aberto do 22.
  - O gerador é `Psychosocial::RaasFile`, isolado e no padrão do `Billing::BpaFile`. Layout oficial é gate: o gerador fica atrás de `config/psychosocial/raas.yml` → `layout_version`, e com `layout_version: null` a resposta é 422 `raas_layout_unavailable`.
  - `GET /psychosocial/raas_exports/:id/file` baixa a remessa e grava trilha `psychosocial.raas_downloaded`.
- **Ficha espelho:** `GET /psychosocial/months/:month/sheets.pdf?health_unit_id` (todas) e `?enrollment_id=` (uma).
  - Sai na ordem das telas do aplicativo RAAS (identificação, atendimento, ações), com mês fechado ou aberto (marca "PRÉVIA").
  - Grava trilha.
- **Quem baixa remessa e fichas:** `psychosocial_lead` e `billing_clerk`.

## 7. Matriciamento e vínculo

- **Matriciamento:** `POST /psychosocial/matrix_supports {health_unit_id (CAPS), health_team_id (UBS), held_on, topic, participants_user_ids[], sigtap_code}`.
  - Gera uma ação `bpa_c`, sem participante paciente.
- **Caso:** `POST /psychosocial/matrix_supports/:id/cases {patient_id, shared_text}`.
- **Leitura pela UBS:** `GET /patients/:id/psychosocial_link`, para quem já pode abrir o paciente pela leitura contextual do ADR 0031.
  - Devolve `{enrolled: bool, caps: {id, name, phone}, since, reference: {name}, shared_cases: [{held_on, caps_name, shared_text}]}`.
  - `shared_cases` só traz os casos da equipe (`health_team_id`) de unidades em que quem lê tem `professional_links` ativo.
  - Grava trilha só quando `shared_cases` não vem vazio.
  - Sem matrícula aberta: `{enrolled: false}`.
  - Nunca devolve CID, substância, PTS ou evolução.

## 8. BPA (módulo 22)

- **Quais ações vão ao BPA:** só as `bpa_c` e `bpa_i`. Vão como `performed_procedures` com `source: "psychosocial"` e `psychosocial_action_id`, pelo padrão evento → job → `Billing::Record.insert!`, como no `FromLab` e no `FromTfd`. As `raas_psi` nunca vão.
- **BPA-C:** uma linha por ação, com `quantity` = quantidade da ação, `patient_id: null` e `cid10: null`.
  - CHECK: `patient_id IS NOT NULL OR (source = 'psychosocial' AND instrument = 'bpa_c')`.
  - `Billing::Critic` e `Billing::Lines` aceitam linha sem paciente nesse caso. Os demais casos não mudam.
- **BPA-I:** leva o paciente.
- **Cancelamento:** cancelar a ação cancela a linha, pelo caminho que o 22 já tem.

## 9. Documentos (módulo 19)

- **Coluna nova:** `clinical_documents.psychosocial_action_id` (uuid, FK).
- **Constraints a ajustar:**
  - `attendance_id` passa a aceitar nulo.
  - CHECK: `num_nonnulls(consultation_id, psychosocial_action_id) <= 1`.
  - CHECK: `attendance_id IS NOT NULL OR psychosocial_action_id IS NOT NULL`.
  - `ck_clinical_documents_consultation` e `ck_clinical_documents_paper_outside_consultation` passam a aceitar `psychosocial_action_id IS NOT NULL` como alternativa a `consultation_id`.
  - `ck_clinical_documents_cbo_required` vale também para `psychosocial_action_id`.
- **Mantém-se:** verificação pública, assinatura (19b) e SNCR (19d), sem outra mudança.
- **Fica fora** de: documentos do cidadão no wpda (`Timeline::CitizenDocuments` e rotas do `citizen_api`), linha do tempo geral (`clinical_document`, `controlled_notification`) e LEDI.

## 10. Linha do tempo, LEDI e cidadão

- **Fonte `Timeline::Sources::PsychosocialSource`:**
  - Entra em `Timeline::Query::SOURCES`. Não entra em `Timeline::KINDS`, nem nas visões do cidadão e do PEC externo (`PROFESSIONAL_ONLY_KINDS` não basta).
  - Fica numa lista nova, `CAPS_ONLY_KINDS`, servida só por `GET /psychosocial/enrollments/:id/timeline`, para a equipe lotada.
  - Na linha do tempo geral entra só um marcador sem texto: "Acompanhamento no CAPS <nome>", com início e fim.
- **Documentos do CAPS:** `ClinicalDocumentSource` e `ControlledNotificationSource` filtram `psychosocial_action_id IS NULL`.
- **LEDI:** nada do CAPS é enfileirado (`Ledi::Enqueue` e os jobs de ficha ignoram unidade `caps`, ações e documentos com `psychosocial_action_id`).
- **wpda:** nenhuma rota nova. As rotas existentes não podem devolver nada com origem CAPS. O spec do wpda só confere isso.
- **SMS e avisos:** o módulo não cria aviso ao cidadão.

## 11. Painel

`GET /psychosocial/dashboard?health_unit_id?`, para `psychosocial_lead`, `municipal_admin` e `analyst`. Traz só contagens, com "menos de 5" para valor de 1 a 4:
- matrículas ativas por CAPS e modalidade;
- novas no mês;
- encerradas por destino;
- PTS vencidos;
- ações por instrumento;
- ocupação dos leitos (noites/leitos-dia);
- situação da RAAS do mês.

## 12. SIGTAP (plataforma)

- **Tabelas:**
  - `sigtap_procedures` ganha `max_quantity` (integer) e `max_stay_days` (integer, nulos permitidos).
  - Tabela nova `sigtap_procedure_attributes (release_id, code, attribute_code)`, com os atributos complementares (009, 035, 036 e os demais que vierem).
- **Importador:** o importador do SIGTAP da plataforma (localizar no código pela carga de `sigtap_releases`) passa a ler os campos de quantidade máxima e permanência do arquivo de procedimentos e o relacionamento de atributos complementares do pacote oficial.
- **Cache:** `Billing::SigtapCache` ganha `max_quantity`, `max_stay_days` e `attributes`.
- **Instrumento:** o instrumento 09 (RAAS Atenção Psicossocial) já vem em `sigtap_procedure_instruments`. Confirmar o código usado na tabela.

## 13. Eventos (só ids, nunca CID, texto ou nome)

| Evento | Payload |
|---|---|
| `psychosocial.referral_created` | `referral_id` |
| `psychosocial.reception_recorded` | `reception_id` |
| `psychosocial.enrolled` | `enrollment_id` |
| `psychosocial.enrollment_closed` | `enrollment_id` |
| `psychosocial.care_plan_saved` | `care_plan_id` |
| `psychosocial.action_recorded` | `action_id` |
| `psychosocial.action_cancelled` | `action_id` |
| `psychosocial.night_stay_started` | `night_stay_id` |
| `psychosocial.night_stay_closed` | `night_stay_id` |
| `psychosocial.night_limit_overridden` | `night_stay_id` |
| `psychosocial.month_closed` | `health_unit_id`, `month` |
| `psychosocial.month_reopened` | `health_unit_id`, `month` |
| `psychosocial.raas_exported` | `raas_export_id` |
| `psychosocial.raas_downloaded` | `raas_export_id` |
| `psychosocial.matrix_support_recorded` | `matrix_support_id` |
| `psychosocial.settings_changed` | (sem payload) |

`Platform.audit` novo entra em R18.

## 14. LGPD e ordem de entrega

- **Cifra:** campos clínicos com `encrypts`, que cobrem:
  - CIDs, substâncias, `notes`, `reason` do encaminhamento;
  - PTS, `shared_text`, motivos de cancelamento e de reavaliação.
- **Fora de URL, log, evento, Analytics e mensagem de erro:** CPF, CNS, nome, CID, substância e texto clínico.
- **Exclusão (ADR 0026):** matrícula, ações, PTS, RAAS e permanências são registro de saúde e ficam retidos. Encaminhamento sem acolhimento e acolhimento `no_follow_up` sem ação seguem a regra da triagem sem atendimento (anonimizar). (P17)
- **Trilha:** toda leitura nominal de detalhe grava `clinical_record.viewed`. Listas não gravam trilha.
- **Ordem de entrega:** api → dashboard → maintenance → wpda (só o spec de ausência).

## 15. Reconciliação dos planos (2026-11-04)

**Regra:** onde um plano diverge desta seção, vale esta seção. O api implementa todas as rotas abaixo, inclusive as que só o plano do dashboard pediu. Cada linha cita a origem (A = api, D = dashboard, W = wpda, M = maintenance).

**P1 — SIGTAP (A-D1):**
- A release é `terminology_releases` com `kind: "sigtap"`, não `sigtap_releases`.
- Os atributos vêm de `rl_procedimento_detalhe` (`CO_DETALHE`). O nome do arquivo é conferido no pacote oficial, como gate.
- Arquivo ausente significa zero atributos.
- `max_quantity` com `0` ou `9999` significa sem limite.

**P2 — BPA-C sem paciente (A-D2):** o CHECK é `patient_id IS NOT NULL OR psychosocial_action_id IS NOT NULL`. Ele não depende do instrumento, porque o `Billing::Recalculate` regrava o instrumento.

**P3 — Instrumento da ação (A-D3):** o instrumento é resolvido pela SIGTAP, nesta ordem:

| Condição | Instrumento |
|---|---|
| `09` com participante matriculado | RAAS |
| senão, `02` | BPA-I |
| senão, `01` | BPA-C |

BPA-I com mais de um paciente → 422 `bpa_i_single_patient`.

**P4 — Leitura e opções:** todas as rotas desta tabela são `GET` sob `/psychosocial`.

| Rota | Conteúdo | Origem |
|---|---|---|
| `units` | CAPS de lotação; para `billing_clerk`, `municipal_admin` e `analyst`, todos | D-D1 |
| `options?health_unit_id` | listas do `raas.yml`, procedimentos permitidos já filtrados pelo CBO de quem pede (com as marcas `family_care` e `collective`), a equipe do CAPS e os `health_teams` | D-D2 |
| `prefill?patient_id&health_unit_id` | ESF e USF a partir do território do 11 e do 24a | A-D4, D-D5 |
| `settings` | configuração do §3 | D-D10 |
| `raas_exports?health_unit_id&month` | lista de remessas; o faturista baixa sem ler a RAAS nominal | D-D11 |
| `matrix_supports?health_unit_id` e `matrix_supports/:id` | matriciamento: a lista sem texto; o detalhe com os casos e trilha | D-D12, A-D13 |
| `actions/:id/documents` | documentos da ação | D-D8 |
| `care_plans/due?health_unit_id` | PTS vencidos | A-D9 |

**Busca de paciente (D-D3):** `POST /psychosocial/person_lookup {cpf|cns}`, no padrão do T30 do módulo 30, mais o `/patients/duplicates` que já existe. Nunca por query.

**P5 — Nada de nome em URL (D-D4):** `GET /psychosocial/enrollments` não aceita `q=`. O dashboard filtra a lista localmente.

**P6 — Formas de leitura (D-D6, D-D20):** as formas de lista e de detalhe seguem o plano do api. Se o dashboard precisar de um campo que o api não dá, o api acrescenta o campo, nunca texto clínico em lista.
- `month` é `"YYYY-MM"`.
- `PATCH /psychosocial/monthly_records/:id {homeless: bool}` vale só para registro `open`. Fora disso, 409 `record_not_open`.

**P7 — Acolhimento (D-D7):**
- A resposta é `{reception_id, enrollment_id|null}`.
- Encaminhamento que não está mais pendente → 409 `referral_not_pending`.

**P8 — Documentos pela ação (A-D7, A-D8, D-D8):**
- **Tipos:** só `prescription`, `controlled_notification_record` e `sick_note`.
- **Lista de medicamentos:** a receita do CAPS não entra na lista de medicamentos do paciente (`PatientMedication`).
- **Constraints de `clinical_documents`:**
  - `num_nonnulls(attendance_id, psychosocial_action_id) = 1`;
  - `citizen_id IS NULL` exatamente quando `psychosocial_action_id IS NOT NULL`. Isso tira o documento do wpda, da linha do tempo e das consultas por par.
- **Leitura:** pelas rotas do 19, o documento do CAPS só é lido por quem está lotado no CAPS; para os demais, 404.
- **Impressão e cancelamento:** pelas rotas que já existem, com `health_unit_id`.

**P9 — Unidade (D-D9):** a rota real do cadastro é `/attendance/units`. Ela ganha os campos do §3 e mantém o step-up que já tem.

**P10 — Vínculo (D-D13):** a rota é `GET /psychosocial/patients/:id/link`, não `/patients/:id/psychosocial_link`. A forma é a do §7. Fora do contexto de leitura → 403 `out_of_context`.

**P11 — Painel (D-D14, D-D19):**
- `GET /psychosocial/dashboard?month&health_unit_id?`.
- Valor de 1 a 4 sai como `"lt5"`.
- O rótulo do destino vem de `options`.

**P12 — Limites de texto (D-D15):**

| Campo | Limite (caracteres) |
|---|---|
| `notes` | 4000 |
| `reason` (encaminhamento e cancelamento) | 2000 |
| `shared_text` | 2000 |
| `needs` do PTS | 4000 |
| cada objetivo do PTS | 500 |
| `topic` | 200 |
| `family_name` | 120 |

Acima do limite → 422 `too_long`, com `path`.

**P13 — Problemas (D-D16, D-D17):**
- **Forma:** `problems[]` é `{error, path, participant_index?}`. A chave é `error`, não `code`, como no resto da API.
- **Códigos novos:**
  - `esf_cnes_required`, `closed_on_in_future`, `origin_locked`;
  - `already_closed`, `already_cancelled`, `bed_label_taken`;
  - `month_open` (exportar mês aberto, 409);
  - `out_of_context`, `action_rejected`, `record_not_open`.
- **`night_limit_reached`:** leva `{used, limit, window_days}`.

**P14 — Fechamento do mês (A-D5, A-D11):**
- `night_stay_open` é pendência que bloqueia: uma permanência aberta na virada do mês bloqueia o fechamento daquele mês.
- `no_actions` vira **aviso**, não bloqueia. O registro sem ação no mês não entra na remessa nem nas fichas, mas continua em "permanência" no mês seguinte.

**P15 — Remessa (A-D6, A-D12):**
- O registro de layouts entra vazio, então a resposta é `raas_layout_unavailable` até o gate.
- Exportar exige mês fechado (409 `month_open`).
- Geram e baixam a remessa `psychosocial_lead` e `billing_clerk`, com step-up para gerar.

**P16 — Alertas do PTS (A-D9):** tabela `psychosocial_care_plan_alerts`, criada pelo job diário, mais a rota `care_plans/due`. Nada vai ao cidadão.

**P17 — LGPD (A-D10):** sai do §14 a frase "familiar sem cadastro … anonimizado com o titular". O titular com ação é sempre retido, então o `family_name` fica retido com a ação.

**P18 — Linha do tempo (A-D13, D-D18, W-W2):**
- Todo kind do CAPS começa com `psychosocial`.
- O marcador da linha do tempo geral é o kind `psychosocial_link`, com `detail_ref: null`. Ele só aparece na visão profissional e fica fora do PEC e do cidadão.
- `CAPS_ONLY_KINDS` reúne os demais kinds `psychosocial_*`.
- O wpda filtra por prefixo `psychosocial` em `citizenApi.history()`, como defesa extra.

**P19 — Sessão do cidadão (W-W1):**
- `Platform::Features::CITIZEN_HIDDEN = %w[psychosocial_care]`. A chave sai de `GET /citizen/session` (`citizen_api/sessions_controller.rb#session_json`).
- Usamos lista de exclusão, não de permissão, para não mexer nas chaves que já existem. Todo módulo futuro com interruptor sensível acrescenta a sua chave aqui.

**P20 — Resumo de saúde do cidadão (W-W3):** o CAPS não grava problema nem medicamento no `Timeline::CitizenSummary` (P8). Um spec confirma isso.

**P21 — Provas de ausência no api (W-W4):** a Task 18 do plano do api tem request specs com semente do CAPS para:
- `/citizen/history`, com e sem `kinds[]`;
- `/citizen/documents/:id/pdf`, que deve dar 404;
- `/citizen/health_summary`, `/citizen/notices`, `/citizen/appointments` e `/citizen/session`.

Também tem o teste de que nenhum SMS nem aviso é enfileirado.

**P22 — Maintenance (M-M1 a M-M3):**
- O rótulo é "Atenção psicossocial (CAPS)".
- O nome do interruptor é traduzido no maintenance (`FEATURE_LABELS`).
- Não há cascata: desligar o prontuário deixa `psychosocial_care` ligado e "falta pré-requisito", e as rotas respondem 403 `feature_disabled`.

### Acréscimos da execução (2026-11-04)

**P23 — Crítica da ação (R1).** A recusa responde `{error: "action_rejected", problems: [{error, path, participant_index?}]}`. O `path` de cada código:

| Código | `path` |
|---|---|
| `instrument_mismatch` | `sigtap_code` |
| `cbo_incompatible` | `sigtap_code` |
| `quantity_exceeded` | `quantity` |
| `performed_in_future` | `performed_at` |
| `month_closed` | `performed_at` |
| `enrollment_required` | `participants[i]` |
| `authorization_required` | `authorization_number` |
| `family_only_for_family_care` | `participants[i].family_name` |
| `bpa_i_single_patient` | `participants` |

**P24 — Pendências do mês (R2).** O 422 `month_has_problems` leva `problems: [{error, path: "records.<monthly_record_id>"}]`. O `GET /psychosocial/months/:month` traz `records[].problems` e `records[].warnings`. `no_actions` é um aviso.

**P25 — Mínimos de texto (R3).**
- `reason` e `shared_text`: no mínimo 10 caracteres.
- `family_name`: no mínimo 3.
- Motivo de reabertura, de reavaliação e de saída do leito: de 10 a 500.
- Abaixo do mínimo → 422 `invalid_request`. Acima do máximo → 422 `too_long`.

**P26 — Cadastro de unidade (R4).** `/attendance/units` continua sem step-up. O §3 fica corrigido: a escrita da unidade CAPS segue as regras que essa rota já tem.

**P27 — Telefone da unidade (R5).**
- `health_units.phone`: só dígitos, de 10 a 11, pode ser nulo. Entra na migração 900002 e é aceito em `/attendance/units`.
- O cadastro do dashboard ganha o campo.
- O vínculo devolve `phone: null` quando o telefone não está cadastrado.

**P28 — Assinatura do documento do CAPS (R6, opção a).** `ck_signature_requests_consultation` e `Signatures::Canonical` passam a aceitar o documento com `psychosocial_action_id`, na migração 900004. O 19b assina como qualquer documento. É superfície transversal, coberta pela suíte completa.

**P29 — Rotas de documento do 19 (R7, opção b).**
- `ClinicalDocumentsController` aceita `psychosocial_staff` e `psychosocial_lead` em `show`, `print` e `cancel`, só para documento com `psychosocial_action_id`.
- A decisão é do grant de lotação no CAPS. Fora dele → 404.
- A equipe do CAPS não recebe `health_professional`.

**P30 — Busca de paciente (R8).**
- `POST /psychosocial/person_lookup` aceita `cpf` **ou** `cns`, no padrão do `Vaccination::PersonLookupsController`, por causa do usuário em situação de rua sem CPF.
- A busca de duplicados é `POST /psychosocial/patients/duplicates`.

**P31 — Procedimentos coletivos (R9).**
- A marca `collective` em `options` vem de `collective_codes` no `raas.yml` (`review: true`).
- Em dev: `0301080216`, `0301080283` e `0301080348`.

**P32 — Unidade CAPS fora do check-in e da agenda (R11).** O check-in do módulo 09 e a marcação do módulo 17 em unidade `kind = caps` são recusados com 422 `caps_unit_not_allowed`, por uma guarda em cada comando. Sem essa guarda, o nome da unidade CAPS apareceria no wpda.

**P33 — Configuração e acolhimento (R12, R13).**
- `GET /psychosocial/settings` é lido por `municipal_admin` e `psychosocial_lead`.
- O acolhimento responde só `{reception_id, enrollment_id|null}`.

**P34 — Guardas do cadastro de unidade (Task 3).** O `/attendance/units` ganha estas recusas:

| Código | Situação |
|---|---|
| 409 `unit_has_caps_records` | CAPS com qualquer registro `psychosocial_*` deixaria de ser `caps` |
| 409 `caps_has_beds` | CAPS noturno com leito ativo passaria para uma modalidade sem leito |
| 409 `unit_has_attendances` | unidade com atendimento ou agenda viraria `caps`; o nome do CAPS iria para o histórico do cidadão |
| 422 `invalid_phone` | telefone inválido |

O dashboard tem texto para cada código.

**P35 — Forma de `options` (Task 3).**
- **Campos:** além do P4, `options` traz `night_sigtap_code`, `competence`, `places`, `group_kinds`, `reception_outcomes` e `referral_priorities`.
- **Procedimentos:** o filtro combina três condições:
  - instrumento `01`, `02` ou `09`;
  - CBO do vínculo de quem pede;
  - prefixo de `procedure_prefixes` no `raas.yml` (`review: true`; em dev, `030108`).
- **`units`:** lista só os CAPS ativos.

**P36 — Detalhes de topo nos erros (Task 3).** Os detalhes que podem subir ao topo do erro estão numa lista de permissão: `used`, `limit`, `window_days` e `rule`. O `rule` é a regra do limite noturno que estourou.

**P37 — Matrícula e mês fechado (Task 4).** A data de início ou de encerramento num mês já fechado é recusada com 422 `month_closed`:

| Operação | Recusa quando | `path` |
|---|---|---|
| Criar ou editar | `started_on` cai num mês fechado | `started_on` |
| Matricular no acolhimento | `started_on` cai num mês fechado | `enrollment.started_on` |
| Editar `started_on` | o mês de início atual está fechado | `started_on` |
| Encerrar | `closed_on` cai num mês fechado | `closed_on` |

- `PATCH` e `close` devolvem o detalhe e gravam `clinical_record.viewed`.
- `PATCH` sem nenhum campo → 422 `invalid_request`.
- As RAAS dos meses depois do `closed_on` ficam fora da crítica, do fechamento, das fichas e da remessa.

**P38 — Prefill e trilha (Task 4).**
- **Prefill:** o `prefill` usa `CareLines::TeamPopulation`: primeiro o cadastro do 24a, senão o território do 11.
  - Só a eSF (tipo 70) conta como cobertura de ESF.
  - A resposta traz também `team`.
  - eSF sem CNES → `esf_covered: true` e `esf_cnes: null`.
- **Encaminhamento:** no detalhe, `referred_by` é `{user_id, name}`, sem `cbo_code`.
- **Trilha:** todo `clinical_record.viewed` do CAPS leva `resource`. `person_lookup` e `patient_duplicates` também gravam a trilha.

**P39 — PTS e grupos (Tasks 5 e 6).**
- **Evento novo:** `psychosocial.care_plan_review_due` (`care_plan_id`) entra na tabela do §13.
- **PTS vencidos:** em `care_plans/due`, `psychosocial_lead` vê o CAPS inteiro. Os demais veem só as matrículas em que são referência. A rota não grava trilha.
- **Grupo:**
  - o JSON do grupo devolve `sigtap_code`, e o nome do procedimento vem de `options`;
  - `PATCH` sem mudança → 200 com o grupo atual;
  - o código e o responsável só são conferidos quando novos ou trocados.
- **`today`:** usa o dia da semana e o dia locais da cidade. Não grava trilha.

**P40 — Ações (Task 7).**
- **Matrícula aberta na data da ação:** a matrícula só conta como aberta se `started_on` for igual ou anterior à data local de `performed_at`. Caso contrário, `enrollment_required` em `participants[i]`, sem criar RAAS fantasma.
  - No acolhimento, `at` antes de `started_on` dá `{error: "enrollment_required", path: "at"}`.
- **Crítica no acolhimento:** responde 422 `action_rejected` com os paths do acolhimento. `performed_at` vira `at`, e os demais vão para `sigtap_code`.
- **Campos:**
  - `quantity` ausente vale 1.
  - `participants` é obrigatório: array, que pode ser vazio, com no máximo 100 itens.
  - Participante repetido → `invalid_request` em `participants[i]`.
  - Matrícula inexistente ou de outro CAPS → `enrollment_required` indexado.
  - Os ids são normalizados para minúsculas.
- **`enrollment_required` sem índice** (`path: "participants"`): ação só RAAS sem nenhum participante matriculado.
- **Cancelamento:** qualquer pessoa lotada no CAPS cancela. Em mês fechado, só a coordenação. Ação já exportada não é cancelada (409).
- **Dashboard:** manda sempre `enrollment_id` quando o paciente tem matrícula. Só `{patient_id}` não liga a ação à RAAS.

**P41 — Acolhimento noturno (Task 8).**

*Ações na saída*
- A saída grava uma ação RAAS por noite, no mês daquela noite, às 20:00 locais, na mesma transação.
- Se alguma noite cai em mês fechado, a saída responde 422 `month_closed` e nada fica fechado.
- Uma data que já tem ação noturna não cancelada do mesmo paciente no mesmo CAPS não gera outra ação.

*Entrada*

| Situação | Resposta | `path` |
|---|---|---|
| `started_at` antes do `started_on` da matrícula | 422 `enrollment_required` | `enrollment_id` |
| primeira noite em mês fechado | 422 `month_closed` | `started_at` |

*Encerrar matrícula*
- Matrícula com permanência aberta não pode ser encerrada: 409 `night_stay_open`.

*Limite*
- A conta é o maior total entre todas as janelas de `window_days` que contêm alguma noite nova, isto é, "N em qualquer janela de 30". Isso cobre também a entrada retroativa.
- A recusa é 422 `night_limit_reached`, com `{used, limit, window_days, rule}`.
- Reavaliação desnecessária é ignorada, e a resposta traz `reassessment: null`.
- Uma permanência reavaliada na entrada não confere o limite de novo na saída, porque o banco aceita uma reavaliação só.

*Mapa dos leitos*
- Forma: `{date, window_days, items: [{bed, stay: {id, patient, started_at, expected_end_at} | null, nights_used, limit}]}`. Substitui o `{beds: [...]}` da fixture do dashboard.
- No CAPS III, o `limit` mostrado é o das noites intercaladas.
- A permanência aberta conta as noites até `expected_end_at`.

*Trilha*
- O detalhe e a saída gravam trilha com `resource: night_stay`. A saída grava porque devolve a reavaliação.
- A entrada e o mapa não gravam.

**P42 — RAAS do mês (Task 9).**
- **Mês futuro:** fechar um mês futuro → 422 `invalid_request`, com `path: month`.
- **Resposta do mês:** o `GET /psychosocial/months/:month` também traz `status`, `closure`, `actions_count` e `counts.actions`.
- **Matrícula retroativa:** vale para criação, acolhimento e `PATCH` de `started_on`.
  - Abre a RAAS de cada mês entre `started_on` e o mês corrente.
  - Se algum mês do intervalo estiver fechado → 422 `month_closed`, com `path` `started_on` (ou `enrollment.started_on` no acolhimento).
- **Limites de `started_on`:**
  - no mínimo o 1º dia do mês de 12 meses antes do corrente (`MAX_RETROACTIVE_MONTHS = 12`);
  - nunca antes do nascimento;
  - fora disso → 422 `invalid_request`, no mesmo `path`.
  - Um `PATCH` que reenvia o mesmo `started_on` é aceito sem efeito.
- **Leito:** a entrada no leito também garante a RAAS do mês da entrada.

**P43 — Remessa e ficha espelho (Task 10).**

**Remessa**
- Mês fechado sem nenhum registro com ação RAAS → 422 `nothing_to_export`. Nada é gravado e o mês não trava.
- O aviso `no_actions` conta só ações `raas_psi` não canceladas. Um registro que só tem BPA:
  - recebe o aviso;
  - fica fora da remessa e das fichas;
  - não impede o fechamento do mês.
- Mês já exportado → 409 `already_exported`.
- Remessa de CAPS desativado continua listável e baixável. Gerar remessa exige CAPS ativo.

**Fichas espelho**
- Quem lê:
  - `psychosocial_lead` lotado no CAPS;
  - `billing_clerk`, só de mês **fechado**. Em mês aberto, ele recebe 409 `month_open`, por minimização: a prévia ao vivo é da coordenação.
  - `psychosocial_staff` recebe 403 `lead_required`.
- Gerar fichas exige CAPS ativo.
- Ficha com mais de uma página repete o cabeçalho "Ações (continuação)" com o nome e o CNS mascarado.

**Trilha**
- Download da remessa: `clinical_record.viewed` com `resource: raas_file`, uma entrada por paciente.
- Ficha: trilha com `resource: raas_sheet`, uma entrada por paciente.

**P44 — Matriciamento e vínculo (Task 11).**

*Vínculo visto pela UBS* (`GET /psychosocial/patients/:id/link`)
- A ordem das checagens evita que a resposta revele quem está no CAPS: primeiro 403 `out_of_context`, pelo `ClinicalRecord::Access`, e só depois qualquer consulta ao CAPS.
- **Sem matrícula aberta** (por exemplo, depois da alta para a atenção básica), a resposta é `{enrolled: false}`. Se houver caso compartilhado com a equipe de quem lê, vem também `shared_cases: [...]`, e só nesse caso a leitura grava trilha. Isso corrige o §7.
- **Interruptor desligado:** a UBS recebe 403 `feature_disabled`. O dashboard trata como "sem vínculo", sem mostrar erro.

*Matriciamento cancelado*
- A ação do matriciamento pode ser cancelada pela rota de ações; a linha do BPA cai junto (Task 12).
- A lista e o detalhe do matriciamento mostram `action_cancelled_at`.
- `add_case` num matriciamento cancelado → 409 `already_cancelled`.
- O vínculo visto pela UBS exclui os casos de matriciamento cancelado.

*Formas*
- A lista e o detalhe trazem também `health_team`, `participants`, `sigtap`, `action_id`, `cases_count` e `created_at`.
- Na crítica, `performed_at` aparece como `held_on`; os demais erros, como `sigtap_code`.
- Código que não resolve para BPA-C → 422 `instrument_mismatch`.

*Risco:* o nome de um CAPS AD no vínculo revela, de forma indireta, uso de substância. Resolvido pelo P46.

**P45 — BPA pelo 22 (Task 12).**
- **CID na BPA-I:** a linha leva o CID principal da matrícula.
- **Cancelamento pelo CAPS:** cancelar a ação cancela a linha. No faturamento, o motivo é o texto fixo "Ação do CAPS cancelada", sem texto do CAPS.
- **Linha já exportada:** recusa com 409 `already_exported`.
- **Cancelamento pelo faturista:** em `/billing/procedures/:id/cancel`, ele cancela a linha direto e a ação do CAPS continua de pé, como nas linhas `tfd` e `lab`. O motivo digitado ali é do faturamento: o runbook orienta a não pôr informação clínica nele.
- **Pendências:** linha sem paciente que vire BPA-I aparece em `pending` com `fix: "patient"` e `patient: null`. O dashboard do faturamento tolera esse caso.
- **Antes do merge:** a linha BPA-I do CAPS só sai da linha do tempo (`ProcedureSource`) e de `CareLines::Sources` com a Task 14.
- **Exclusão:** linha BPA-I do CAPS retém o paciente na exclusão. A Task 16 confere.
- **Gate:** `PATIENTLESS_AGE = 0` na BPA-C ainda precisa ser conferido no layout oficial.

**P46 — Vínculo genérico, sem o nome do CAPS (decisão do usuário, 2026-11-04).** Este item substitui partes do §7, do §10 e do P44.

O nome de um CAPS AD revela, de forma indireta, o uso de substância. Por isso, fora da equipe do CAPS, nenhuma superfície mostra o nome nem a modalidade da unidade.

- **Vínculo para a UBS** (`GET /psychosocial/patients/:id/link`): devolve `{enrolled, caps: {phone}, since, reference: {name}, shared_cases: [{held_on, shared_text}]}`.
  - Sem `caps.id`, `caps.name`, `caps_modality` e `caps_name`.
  - O texto da tela é "Em acompanhamento no CAPS desde …".
- **Marcador da linha do tempo geral** (`psychosocial_link`): "Acompanhamento no CAPS", sem o nome, com início e fim.
- **Outras superfícies fora do CAPS:** o mesmo vale para eventos, painel de quem não é do CAPS e qualquer superfície futura. O painel do CAPS para `psychosocial_lead`, `municipal_admin` e `analyst` continua por unidade, porque é agregado e não nominal.
- **Specs:** um spec confere que nenhuma resposta vista pela UBS contém o nome da unidade CAPS.

**P47 — Documentos pela ação (Task 13).**
- **Criação:** o `POST /psychosocial/actions/:id/documents` devolve o documento no topo, como no 19. Se a ação estiver cancelada → 409 `already_cancelled`.
- **Imprimir e cancelar:** o `health_unit_id` é aceito e ignorado. Quem decide é a lotação no CAPS.
- **Assinatura:** as rotas `/signature/*` e o `signable_by` aceitam `psychosocial_staff` e `psychosocial_lead`, limitados aos pedidos de documentos do CAPS de que a pessoa é autora. É a mesma lógica do P29.
- **Detalhe da assinatura:** `GET /signature/signatures/:id` devolve 404 para documento do CAPS. O dashboard não liga para esse detalhe; o PDF assinado sai pelo `print` do 19.
- **Cancelamento:** cancelam a autora lotada e também o `psychosocial_lead` lotado no CAPS, com trilha (E1, opção a). Assim, uma receita errada continua cancelável depois que a autora sai do CAPS.
- **Farmácia:** vê a receita do CAPS para dispensar. A Task 14 tira a dispensação do CAPS da linha do tempo e da "última retirada" da lista de medicamentos da UBS.

**P48 — Nome do CAPS nos documentos e telefone no vínculo** (decisão do usuário, 2026-11-04).
- **Telefone:** o vínculo visto pela UBS (P46) mantém `caps.phone`, para a UBS poder ligar direto. O risco de identificar a unidade pelo número foi aceito.
- **Verificação pública** (`/v/:token`), para documento com `psychosocial_action_id`: a unidade aparece como "Centro de Atenção Psicossocial", com o município e o CNES. Saem o nome e a modalidade.
- **Atestado** (`sick_note`) do CAPS: o impresso e o PDF assinado também usam "Centro de Atenção Psicossocial", com o município e o CNES, sem o nome nem a modalidade. Assim o empregador não fica sabendo do tratamento.
- **Receita e notificação de controlado:** o impresso mantém a identificação completa do emitente, porque a farmácia precisa dela. Na verificação pública desses documentos vale a forma genérica.
- **Specs:** a verificação pública de qualquer documento do CAPS e o atestado impresso não podem conter o nome nem a modalidade da unidade.

**P49 — Profissional no atestado do CAPS (Task 13; segue a intenção do P48).**
- **F1:** no atestado (`sick_note`) do CAPS, o profissional aparece só com a profissão e o registro, por exemplo "Médico — CRM …". Não aparecem a especialidade, o título ("psiquiatra") nem o CBO. Isso vale no impresso, no PDF assinado e na verificação pública. A especialidade revelaria o tratamento ao empregador, como o nome do CAPS no P48. Um spec confere.
- **F2:** a receita e a notificação de controlado assinadas (`/v/:token/signed.pdf`) continuam completas, com o nome do CAPS e o profissional completo, porque a farmácia precisa deles. O JSON e o HTML da verificação seguem genéricos (P48).
- **Motivo do cancelamento de documento:** segue a regra do 19, de 10 a 500 caracteres. O limite de 10 a 2000 vale só para o cancelamento da ação do CAPS.

**P50 — Notas da Task 13.**
- **Canônico assinado:** o `clinical-document-v1` (contracts `clinical-v1.1.0`) continua com o `cbo_code` real, porque o esquema exige 6 dígitos. Ele não fica público:
  - a verificação pública e o `signed.pdf` do atestado não mostram o CBO;
  - `/signature/signatures/:id` responde 404 para documento do CAPS.

  Tirar o CBO dali exigiria uma tag nova do contracts, e trocá-lo por um genérico falsificaria o documento assinado. Por isso o CBO fica.
- **Nome do profissional no atestado:** fica, porque o CFM exige a identificação do médico.
- **Quem emite documento no CAPS:** só médico (CBO 2251, 2252 ou 2253). Os outros profissionais recebem 422 `prescriber_required`. Declaração de comparecimento por psicólogo e outros profissionais está fora desta entrega.

**P51 — Sigilo fora do diff (Task 14; segue P32, P46 e P48).**

**G1 — CAPS fora dos caminhos automáticos e de cadastro.**
- `Territory::ReplaceCoverage` e `CareRegulation::SaveProvider` recusam unidade `caps` com 422 `caps_unit_not_allowed`.
- `ReferenceUnits` exclui o CAPS, o que protege também os dados que já existem.
- `CareLines::OpenAppointmentRequest` ganha a mesma guarda.
- O 409 `unit_has_attendances` do P34 passa a contar também cobertura de território e prestador.

Sem isso, toda triagem web do bairro abriria um pedido com destino no CAPS, e o nome dele apareceria no wpda.

**G2 — Nome genérico, opção (i).**
- `Timeline::Sources::Labels.unit` é o único ponto de nome de unidade. Unidade `caps` vira "Centro de Atenção Psicossocial", como no P48, em todas as fontes gerais:
  - exame;
  - vacina, inclusive para o cidadão;
  - dispensação;
  - procedimento.
- `ProcedureSource` e `CareLines::Sources` filtram também as linhas de unidade `caps`. Isso inclui o BPA manual lançado antes do módulo.
- O CAPS continua podendo pedir exame, ter sala de vacina e estoque. Só o nome some fora do CAPS.

**Interface e operação.**
- `/citizen/history?kinds[]=psychosocial_*` responde 422 `invalid_kinds`. O wpda e o dashboard não pedem esses kinds.
- O marcador "Acompanhamento no CAPS" continua na linha do tempo profissional mesmo com o interruptor desligado, porque o registro fica retido.
- Runbook: a farmácia não lança a receita impressa do CAPS como "externa", porque nesse caso ela escapa do filtro de dispensação.

**P52 — Painel (Task 15).**

*Acesso*
- `psychosocial_lead` vê só os CAPS em que está lotado. Fora deles → 403 `not_on_caps_team`.
- `municipal_admin` e `analyst` veem todos os CAPS ativos.
- `billing_clerk` e `psychosocial_staff` → 403 `missing_role`.
- Mês futuro → 422, com `path: month`.
- O painel não grava trilha.

*Valores*
- De 1 a 4 sai `"lt5"`. Zero sai `0`.

*Campos*
- `enrollments_active[]` usa `caps_modality`.
- `closed_by_destination` traz todos os destinos diferentes de `"00"`, com 0 quando não houver registro. O rótulo vem de `options`.
- `raas_month` traz `open`, `closed`, `exported` e `no_actions`.

*Definições*
| Indicador | Como é contado |
|---|---|
| Matrículas ativas | abertas no fim do mês; no mês corrente, abertas hoje |
| PTS vencidos | situação de hoje, qualquer que seja o mês pedido |
| Noites | regra do limite noturno até o fim do mês ou até hoje, o que vier antes; a noite de hoje conta |
| Leitos-dia | leitos ativos agora × dias do mês |

*Risco aceito: supressão desfeita por diferença*
A supressão de "menos de 5" pode ser desfeita por subtração:
- o total da RAAS menos as matrículas ativas dá a soma dos encerramentos;
- o total sem filtro, comparado ao filtrado por unidade, isola a unidade suprimida.

Não há supressão complementar. É o mesmo padrão do painel do transporte, e só papéis privilegiados (coordenação, gestão e analista) leem o painel.

**P53 — Nome genérico também nos detalhes (Task 14; aplica P46/P51).**
- **Telas fora do CAPS:** em qualquer tela de outro módulo, uma unidade `caps` aparece como "Centro de Atenção Psicossocial", seguido do CNES quando houver, para todos os leitores. Vale também para os detalhes abertos a partir da linha do tempo:
  - `GET /vaccination/immunizations/:id` e o cartão lido pelo vacinador, que perde também o nome livre da sala;
  - `GET /exams/orders/:id`;
  - as telas da regulação e do laboratório.
- **Telas do CAPS:** as próprias telas do CAPS (`/psychosocial/*`) continuam mostrando o nome real.
- **Ponto único:** `Labels.unit_name` concentra essa regra. `PerformedProcedure.outside_caps` filtra a produção do CAPS.
- **409 `unit_has_attendances`:** passa a contar também a cobertura de território e o prestador. O texto do dashboard fica genérico: "a unidade já tem histórico de atendimento, cobertura ou prestador".
- **Risco baixo aceito:** uma equipe de saúde (`HealthTeam`) numa unidade CAPS não entra no 409, porque o `OpenAppointmentRequest` já barra o caso.

**P54 — Exclusão LGPD (Task 16).**
- **O que retém o paciente:** matrícula; ação, inclusive cancelada; permanência; caso de matriciamento; linha BPA-I; documento do CAPS. Isso vale também para os pares e para os duplicados por CPF.
- **O que é anonimizado:** encaminhamento e acolhimento sem seguimento. Os campos `reason` e `notes` vão a NULL e a linha recebe `anonymized_at`. A anonimização acontece uma vez só, e depois a linha fica congelada por trigger.
- **Como o encaminhamento anonimizado se comporta:**
  - sai da fila;
  - tentar acolhê-lo dá 409 `referral_not_pending`;
  - o detalhe dá 404, sem trilha.
- **Na prática:** só o encaminhamento de um paciente que nunca foi atendido chega a ser anonimizado. O acolhimento sempre gera uma ação, e a ação retém o paciente.

**P55 — Perfil profissional da equipe do CAPS (Task 17).**
- **Regra:** `Professionals::Create` (módulo 10) aceita `psychosocial_staff` e `psychosocial_lead` para criar o perfil profissional, com CNS e CBO, e o vínculo CNES (`professional_links`). Esses usuários aparecem na lista de pendentes do admin.
- **Por quê:** sem isso, a prefeitura só conseguiria criar o perfil da equipe do CAPS dando o papel `health_professional`, o que vai contra o P29.
- **Garantia:** um spec negativo confere que ter perfil e vínculo numa UBS não dá permissão clínica fora do CAPS.

**Semente de dev:**
- usuários `caps@`, `capsmed@`, `capspsi@` e `capstec@curitiba.dev`, com TOTP fixo;
- telefones com prefixo 99777.

Se a SIGTAP do CAPS ou os CIDs faltarem, a crew avisa e não semeia, sem interromper o `db:seed`.

**P56 — Preflight do dashboard.**
- **R1:** vale o api real (`328a4351`). Os tipos e as fixtures do dashboard são reescritos a partir dele.
- **R2:** `options.procedures[]` ganha `max_quantity` e `requires_authorization` (atributo 036). Entra na onda final do api.
- **R3:** `today.groups[]` ganha `sigtap_code` e `action_id`, este da ação não cancelada do grupo no dia. Entra na onda final do api.
- **R9:** o dashboard oferece pela ação os três tipos do P8: receita, notificação de controlado e atestado. A notificação de controlado é o caso central do CAPS (psicotrópicos B1, C1), e a decisão 19 da spec já a inclui.
- **R10:**
  - As rotas `/sncr/*` aceitam `psychosocial_staff` e `psychosocial_lead`, limitados aos documentos do CAPS de que a pessoa é autora, no padrão do P47. Entra na onda final do api.
  - No dashboard, `canSign` aceita os dois papéis.
- **R17:** os seletores de unidade do check-in, da agenda e da regulação no dashboard filtram `kind !== "caps"`. Entra neste plano, não como card à parte. A frase do 422 `caps_unit_not_allowed` fica como defesa.
- **R19:** no formulário de encaminhamento, a UBS vê o nome dos CAPS no seletor, porque precisa escolher o destino. A mensagem de sucesso é genérica: "Encaminhamento enviado ao CAPS".
- **R4–R8, R11–R16, R18 e R20:** decisões internas do dashboard, aceitas como estão no preflight.

**P57 — Revisão final do api.**
- **I1 (início da matrícula):**
  - `in_window` também corta os meses anteriores a `started_on`.
  - O `PATCH` que move `started_on` para depois da ação mais antiga é recusado com 422 `started_after_actions`, `path: started_on`.
- **I2:** o P53 vale também no laboratório (`Lab::Json.requisition`, `Lab::ShipmentJson` e `scheduled_place` de `Lab::OpenRequisition`). O nome do CAPS deixa de chegar ao wpda pelo `Exams::CitizenList`.
- **I3 (telefone e situação de rua):**
  - Registro com `homeless = true` e sem telefone: `phone_missing` vira aviso. A remessa e a ficha usam o telefone da unidade CAPS como "telefone de contato", porque o CAPS é o contato legítimo da pessoa em situação de rua.
  - Se a unidade também não tiver telefone, continua sendo pendência.
  - Nos demais casos, `phone_missing` segue como pendência.
  - Gate: conferir se o SIA aceita o telefone da unidade.
- **Registros sem ação:** registro `no_actions` não bloqueia o fechamento por nenhuma pendência, porque fica fora da remessa.
- **Gates para o runbook:**
  - **G2:** a regra de `QT_MAXIMA_EXECUCAO`. Hoje a conferência é por ação; se o SIA limita por competência e paciente, o excesso passa sem aviso. `QT_DIAS_PERMANENCIA` é importado, mas não é usado.
  - **G6:** sem o retrato do CNES da cidade, ou com o CAPS sem CNES, nenhum mês fecha.
- **Correções miúdas:** ficam a critério da sessão.

**P58 — Onda final do api.**
- **Permanência aberta:** `night_stay_open` continua bloqueando o fechamento, mesmo em registro sem ação RAAS. É uma exceção ao P57. Na saída do leito, cada noite vira uma ação daquele mês, e com o mês fechado a saída seria recusada.
- **Pendências em `warnings`:** os códigos de pendência também podem chegar em `warnings`. Por exemplo, `patient_cns_missing` e `homeless_missing` num registro sem ações. O dashboard tem rótulo para todos eles nos dois lugares.
- **SNCR:**
  - O número SNCR só vai na receita controlada digital. A notificação de controlado é sempre em papel e não leva número.
  - A autoria fica garantida na emissão, porque o estoque SNCR é por usuário.
- **Telefone de contato (I3):** a foto do mês grava o telefone do CAPS no campo de telefone do paciente, sem marcar de onde ele veio. Se o layout oficial da RAAS pedir essa distinção, o ajuste entra no gate do layout (G1).

**P59 — Busca de CID e campanhas.**
- **Busca de CID:** `POST /attendance/ciap2/search` aceita `psychosocial_staff` e `psychosocial_lead` só com `terminology: "cid10"`. A CIAP-2 continua exclusiva de `health_professional`, no padrão do P47 e do P56. Implementado em `200206f6`.
- **Campanhas (módulo 12):** o público de uma campanha nunca pode ser uma unidade `caps`. Uma campanha "usuários do CAPS X" mandaria aviso ou SMS que revela o vínculo com o CAPS, contra a decisão 4.
  - O api de campanhas recusa unidade `caps` no público com 422 `caps_unit_not_allowed`.
  - A lista de unidades que alimenta esse seletor exclui o CAPS.
  - O dashboard filtra `kind !== "caps"` no seletor.
- **Telas que mantêm o CAPS:** configuração do admin, REMUME, origem da coleta, sala de vacina e transporte. Nelas o CAPS é destino legítimo, e só a equipe interna as vê. Fora do CAPS, os rótulos seguem o P53.

**P60 — Prazo do PTS e CAPS certo na matrícula (Task 6 do dashboard).**
- **Prazo do PTS:** `GET /psychosocial/options` traz `care_plan_review_days`, porque a equipe não lê `/settings` (P33). O dashboard usa esse prazo para sugerir a revisão do PTS e para marcar "sem PTS, prazo vencido", pela mesma regra do job.
- **Desempenho:** `options.procedures` é lido em lote. Isso resolve o N+1, que deixa de ser resíduo.
- **Do acolhimento para a matrícula:** a passagem leva `{enrollmentId, unitId}`, e o portão abre direto no CAPS certo.
  - Se o CAPS não for de quem lê, a tela não busca nada e não grava trilha.
  - Isso corrige quem está lotado em dois CAPS: antes, a matrícula abria no CAPS errado e a trilha registrava uma leitura que ninguém fez.

**P61 — Prova no navegador.**

*Acolhimento sem matrícula (P2 da prova)*
- Quando o desfecho do acolhimento não é `enrolled`, o dashboard oferece só procedimentos de instrumento `bpa_i`, pelo campo `instrument` de `options.procedures`.
- O 422 do api continua como defesa.

*Acolhimento noturno em CAPS sem leito (P3 da prova)*
- `options` não oferece o `night_sigtap_code` a CAPS fora de `NIGHT_MODALITIES`. A correção é pequena no api, e por isso deixa de ser gate.
- O gate do código SIGTAP do acolhimento noturno continua como estava.

*Leito liberado e contagem de noites (P9 da prova)*
- Conta-se uma noite para cada data D em que `started_at.date ≤ D < ended_at.date`. A permanência aberta conta até `expected_end_at`.
- Permanência que entra e sai no mesmo dia vale 0 noite e não gera ação noturna.
- O `NightLimit` e a ação RAAS da saída seguem essa regra.
- `GET /psychosocial/beds/board`:
  - traz `ended_at` em `stay`;
  - mostra como ocupante só a permanência aberta na data, ou a que cobre a noite da data.
- Permanência encerrada no dia aparece como "saiu às …", sem o botão "Registrar saída".
