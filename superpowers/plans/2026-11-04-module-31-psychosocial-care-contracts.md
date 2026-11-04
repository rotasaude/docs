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
