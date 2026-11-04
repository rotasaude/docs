# Módulo 31 — Atenção psicossocial (CAPS) (api) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **PRÉ-REQUISITO: módulo 30 em `origin/main` do api** (conferido ao escrever este plano em `dc1c87df`): última migração de cidade `20261103900001_add_sanitary_transport.rb` (`define(version: 2026_11_03_900001)`, `enable_extension "btree_gist"`); última de plataforma `20261027900001_create_notifiable_diseases.rb` (`define(version: 2026_10_27_900001)`); `AddSanitaryTransport::ROLES` (22 papéis), `::HEALTH_UNIT_KINDS = %w[ubs upa hospital other ceo health_management]`, `::SOURCES = %w[attendance consultation lab_report manual tfd]`, referenciados em `spec/models/{create_analytics_migration_spec,create_campaigns_migration_spec,membership_roles_spec,transport_schema_spec}.rb`; `Membership::PRIVILEGED_ROLES` termina em `transport_dispatcher tfd_committee`; `Platform::Features::CATALOG` termina em `tfd`; `spec/adr_pointers_spec.rb` com `VALID_RANGE = (1..48)`; SIGTAP da plataforma = `terminology_releases` (`kind: "sigtap"`, `version` = competência) + `sigtap_procedures` (sem quantidade máxima nem permanência) + `sigtap_procedure_{cbos,cids,instruments}`, escrita só por `Terminology::Import` → `Terminology::SigtapReader#write(release)` (lê `tb_procedimento` pelo `_layout.txt`; `QT_MAXIMA_EXECUCAO` e `QT_DIAS_PERMANENCIA` já estão no layout do `lib/sigtap_sample.rb`, gravados `9999` e `0`), tabelas imutáveis por `terminology_codes_guard()` em `db/platform_triggers.sql`; `Billing::SigtapCache#{release,procedure,instruments,cbos,cids}`; `Billing::Critic::Facts` (7 membros) chamado em `Billing::Record.insert!` e `Billing::Recalculate.outcome`; `Billing::Critic.instrument(codes)` (`02` → `bpa_i`, `01` → `bpa_c`); `Billing::Lines.age(p)` e `Billing::Json`/`Billing::Pending` leem `p.patient` sem guarda; `performed_procedures.patient_id NOT NULL`, `ck_performed_procedures_{domains,source_ref}`; `Billing::FromTfd` + `Billing::TfdProductionJob` (IdempotentConsumer, `Gate.enabled?`); `Billing::Cancel.call(procedure:, reason:, by:)`; `Billing::Periods.assign!(establishment, competence)`; `Billing::PatientContact.for(ids)`; `Billing::Staff.cns(user)`; `clinical_documents.attendance_id NOT NULL`, `citizen_id NOT NULL`, `ck_clinical_documents_{consultation,paper_outside_consultation,cbo_required}`; `ClinicalDocuments::Issue.call(kind:, content:, by:, consultation: nil, attendance: nil, replaces_document_id: nil, now:)`; `document.attendance.health_unit` lido em `ClinicalDocuments::{Pdf,Verification}`, `Signatures::Canonical` (linha 86) e `ClinicalDocuments::Read.document_grant`; `ClinicalDocuments::Issuers.role(cbo)` (`:physician` = `2251/2252/2253`); `Timeline::KINDS` termina em `transport`; `Timeline::Query::{SOURCES,PROFESSIONAL_ONLY_KINDS = %w[dispensation]}`; `Timeline::Cursor.decode` aceita só `Timeline::KINDS`; `PecTimelinesController::KINDS = Timeline::KINDS - %w[transport]`; `Timeline::Sources::ProcedureSource` lê `PerformedProcedure.recorded.where(patient_id:)` sem filtro de fonte; `CareLines::Sources` (linha 135) idem; `Ledi::ConsultationFichaJob.enqueue_for` e `Ledi::ScreeningFichaJob.enqueue_for` sem filtro de unidade; `ClinicalRecord::Access.call(user:, patient:)`; `ClinicalRecord::Trail.viewed!`; `TerritoryRegistry::Teams.for_patients(ids)`; `CnesSnapshot` (plataforma, por `ibge_code`) + `CnesProfessionalBond` (`cns` cifrado não determinístico); `Admin::SmallCount.small?` (1..4); `Professionals::Cns.generate(seed)`; filtros de log já cobrem `:reason`, `:note`, `:text`, `:cns`, `:cpf`, `:phone` (por substring).

**Goal:** O lado api do módulo 31 (F-31.1 a F-31.8, ADR 0049): CAPS como tipo de unidade com modalidade, equipe pela lotação CNES, encaminhamento da UBS, acolhimento com ou sem matrícula, PTS com versões, grade de grupos e "Hoje no CAPS", ações imutáveis com a crítica SIGTAP na hora, receita do 19 a partir da ação, acolhimento noturno com leito exclusivo e limite, RAAS do mês (continuidade, crítica, fechamento, remessa atrás de `layout_version`, ficha espelho), matriciamento com o vínculo visto pela UBS, BPA-C/BPA-I pelo 22, SIGTAP da plataforma ampliado, linha do tempo restrita, fora do LEDI e do cidadão, painel, exclusão LGPD e semente de dev.

**Architecture:** Quatro migrações separadas para que as mudanças transversais fiquem isoladas e revisáveis: uma de plataforma (SIGTAP, `20261104900001`) e três de cidade — o CAPS (`20261104900002`), o 22 (`20261104900003`, `performed_procedures`) e o 19 (`20261104900004`, `clinical_documents`). O prontuário do CAPS é lido só por quem tem `professional_links` ativo no CAPS (`Psychosocial::Access`), sempre com `clinical_record.viewed` `access: "psychosocial"`. A ação é o centro: toda ação nasce por `Psychosocial::RecordAction` (crítica `Psychosocial::ActionRules` sobre a SIGTAP da competência), é imutável no banco e só se cancela; as RAAS do mês (`psychosocial_monthly_records`) apontam para a matrícula e leem as ações do mês; BPA-C/BPA-I seguem evento → `Billing::PsychosocialProductionJob` → `Billing::Record.insert!`. A remessa é um gerador puro (`Psychosocial::RaasFile`) com registro de layouts vazio até o gate; a ficha espelho sai sempre.

**Tech Stack:** Rails 8.1 (API), PostgreSQL 16 (cidade e plataforma; `btree_gist` já habilitado na cidade), Solid Queue, Prawn (ficha espelho), RSpec via `bin/test`. **Nenhuma gem nova.**

**Spec:** `docs/superpowers/specs/2026-11-04-module-31-psychosocial-care-design.md` e `docs/adr/0049.md` (com as "Decisões do coordenador"). Contrato entre apps — **fonte única de formatos**: `docs/superpowers/plans/2026-11-04-module-31-psychosocial-care-contracts.md`; toda diferença está em "Divergências propostas ao contrato", no fim. Pesquisa: `docs/pesquisa/2026-11-04-atencao-psicossocial-31.md`.

## Desvios e precisões

1. **Fuso e mês.** Toda data/hora em ISO 8601; `performed_at`, `started_at`, `next_visit_at` gravados em UTC e lidos no `Time.zone` da cidade (o `CityConnection.with` fixa o fuso). O mês de uma ação é `performed_at.in_time_zone.to_date.beginning_of_month`, gravado em `psychosocial_actions.month` (date, dia 1, CHECK `extract(day from month) = 1`) pelo Ruby: uma ação de 30/11 às 23:30 local é de novembro. A noite de uma permanência é a data local de cada entrada: `started_at.in_time_zone.to_date ... fim.in_time_zone.to_date` (meio-aberto, mínimo 1), com `fim = ended_at || expected_end_at`.
2. **Lotação.** `Psychosocial::Access.link(user, health_unit_id)` = o `ProfessionalLink` ativo mais antigo (`started_at, id`) do profissional do usuário naquele CAPS; o CBO da ação é o dele. Exige papel `psychosocial_staff` ou `psychosocial_lead` (senão 403 `missing_role`) e vínculo (senão 403 `not_on_caps_team`). `psychosocial_lead` sem vínculo no CAPS também recebe `not_on_caps_team` (a coordenação é lotada). Registro de outro CAPS por id → 404; lista de CAPS não lotado → 403.
3. **Instrumento da ação** (`Psychosocial::Instrument.for(codes, enrolled:)`, sobre os `instrument_code` da SIGTAP da competência do mês da ação): com algum participante matriculado e `09` listado → `raas_psi`; senão `02` → `bpa_i`; senão `01` → `bpa_c`; `09` sem participante matriculado → `enrollment_required`; nenhum dos três → `instrument_mismatch`. Gravado na ação (`instrument`); a ação nunca muda de instrumento.
4. **Crítica na hora** (`Psychosocial::ActionRules.problems`): todos os problemas de uma vez, em `problems[]` com `code` e `participant_index` (índice no corpo) quando couber, na ordem do contrato §4. Regras: `cbo_incompatible` (CBO do vínculo fora de `sigtap_procedure_cbos`; relação vazia não restringe); `quantity_exceeded` (`max_quantity` não nulo e `quantity > max_quantity`; `quantity` fora de 1..999 → `invalid_request` `path: "quantity"`); `performed_in_future` (`performed_at > now`); `month_closed` (fechamento ativo do CAPS no mês); `enrollment_required` (ação `raas_psi` sem participante matriculado, ou participante `{enrollment_id}` de matrícula fechada ou de outro CAPS — esse com `participant_index`); `authorization_required` (atributo `036` na SIGTAP e `authorization_number` vazio; formato 13 dígitos, senão `invalid_request`); `family_only_for_family_care` (participante `{family_name}` com código fora de `family_care_codes`, com `participant_index`); `bpa_i_single_patient` (ação `bpa_i` sem exatamente um participante com paciente — D3). Código fora da SIGTAP ativa da competência → `instrument_mismatch`.
5. **Mês fechado.** Inserir ação em mês com fechamento ativo é recusado para todos (`month_closed`; o trigger também recusa); a coordenação reabre antes. Cancelar ação de mês fechado e não exportado: só `psychosocial_lead` (403 `lead_required`); de mês exportado: 409 `already_exported`.
6. **RAAS do mês.** `Psychosocial::MonthlyRecords.ensure!(enrollment, month)` cria o registro `open` (índice único `(enrollment_id, month)`, `ON CONFLICT DO NOTHING`) ao matricular (mês do início), ao gravar ação com o participante matriculado (mês da ação) e no `Psychosocial::OpenMonthJob` (todo dia, mês corrente, toda matrícula aberta: a continuidade do dia 1º é a primeira execução do mês). Encerrar a matrícula grava `destination` e `concluded_on` no registro do mês de `closed_on` (criado se faltar).
7. **Fechamento.** `Psychosocial::MonthProblems.for(health_unit:, month:)` devolve os problemas do contrato §6 mais `night_stay_open` (D5: permanência aberta com noite no mês). Fechar trava os registros (`closed`) e grava a foto (`snapshot`, JSON cifrado: identificação do paciente, raça/cor, etnia, telefone, endereço, origem só no inicial, CIDs, substâncias, ESF, destino, conclusão e `homeless`). Reabrir apaga nada: marca o fechamento como reaberto (`reopened_at`, motivo cifrado) e volta os registros a `open` com `snapshot` nulo.
8. **Competência de apresentação.** `Billing::Periods.assign!(Billing::Establishment.for_unit(caps), month.strftime("%Y%m"))` — o calendário do 22, sem calendário próprio; gravada em `psychosocial_raas_exports.competence`.
9. **Remessa.** `Psychosocial::RaasFile::LAYOUTS = {}.freeze` (registro de layouts por versão) e `config/psychosocial/raas.yml` `layout_version: null`. Sem layout registrado para a versão configurada → 422 `raas_layout_unavailable`. Os specs registram um layout de teste por `stub_const` para provar o fluxo (estados, competência, trilha, arquivo imutável); o layout oficial entra no gate (D6).
10. **Ficha espelho.** `Psychosocial::MirrorSheetPdf.call(records:, preview:)`: uma página por registro, blocos na ordem do aplicativo RAAS (Identificação, Dados do atendimento, Ações), só os campos que o aplicativo pede; com o mês aberto, a marca "PRÉVIA" no topo de cada página e os dados lidos ao vivo (sem foto).
11. **BPA.** Só ações `bpa_c` e `bpa_i` geram linha; uma linha por ação (`psychosocial_action_id` único). BPA-C: `patient: nil`, `cid10: nil`, `quantity` da ação. BPA-I: o paciente do único participante, `cid10` = CID principal da matrícula quando houver, senão nulo. Estabelecimento = o CAPS (`Billing::Establishment.for_unit`, nunca APS). Profissional = o da ação com o CBO gravado nela. `recorded_by: nil`. O cancelamento da ação cancela a linha na mesma transação (`Billing::Cancel`); linha já exportada em competência fechada → a ação também não cancela (409 `already_exported`).
12. **Receita.** Do 19c/19d por `ClinicalDocuments::Issue.call(..., psychosocial_action:)`: tipos `prescription`, `controlled_notification_record` e `sick_note` (D7); autor = profissional da ação (senão 403 `not_author`); ação não cancelada com exatamente um participante matriculado e `Issuers.role(action.cbo_code) == :physician` (senão 422 `prescriber_required`). `citizen_id` NULO (D8), sem entrar na lista de medicamentos do paciente (a UBS a veria) e sem `attendance_id`.
13. **Trilha.** `Psychosocial::Trail.viewed!(patient:, user:, resource:)` publica `clinical_record.viewed` com `access: "psychosocial"` e `resource` (`enrollment`, `referral`, `action`, `care_plans`, `timeline`, `raas_sheet`, `night_stay`, `link_shared_cases`); listas não gravam trilha.

## Valores fixados por este plano

1. **Interruptor** (depois de `tfd` no `CATALOG`): `Entry.new(key: "psychosocial_care", description: "Atenção psicossocial (CAPS): matrícula, PTS, ações, grupos, acolhimento noturno, RAAS do mês, matriciamento e BPA-C", requires: %w[feature:clinical_record])`.
2. **Papéis:** `psychosocial_staff` e `psychosocial_lead` em `Membership::ROLES` e `PRIVILEGED_ROLES`. Lista nova (24, ordem alfabética) em `AddPsychosocialCare::ROLES`.
3. **`HealthUnit`:** `KINDS = %w[ubs upa hospital other ceo health_management caps]`; `CAPS_MODALITIES = %w[caps_i caps_ii caps_iii caps_i_child caps_ad caps_ad_iii caps_ad_iv]`; `NIGHT_MODALITIES = %w[caps_iii caps_ad_iii caps_ad_iv]`; `SERVICE_CLASSIFICATION = /\A\d{3}\z/`; `#caps?`, `#night_capable?`.
4. **`Psychosocial::Catalog`:** `REFERRAL_PRIORITIES = %w[routine priority]`, `REFERRAL_STATUSES = %w[pending received]`, `RECEPTION_OUTCOMES = %w[enrolled referred_elsewhere no_follow_up]`, `PLACES = %w[caps territory]`, `INSTRUMENTS = %w[raas_psi bpa_c bpa_i]`, `INSTRUMENT_CODES = { "09" => "raas_psi", "02" => "bpa_i", "01" => "bpa_c" }`, `GROUP_KINDS = %w[group workshop assembly]`, `INTENSITIES = %w[intensive semi_intensive non_intensive]`, `MONTHLY_STATUSES = %w[open closed exported]`, `ACTION_ORIGINS = %w[manual reception night_stay matrix_support]`, `WEEKDAYS = (1..7)` (ISO, 1 = segunda), `QUANTITY = (1..999)`, `REASON = (10..500)`, `CLINICAL_TEXT = (1..4000)`, `SHARED_TEXT = (10..4000)`, `GOAL_TEXT = (1..500)`, `MAX_GOALS = 20`, `MAX_SECONDARY_CIDS = 4`, `FAMILY_NAME = (3..120)`, `NAME_MAX = 120`, `TOPIC_MAX = 200`, `BED_LABEL_MAX = 20`, `CAPACITY = (1..100)`, `MAX_EXPECTED_STAY_DAYS = 30`, `AUTHORIZATION = /\A\d{13}\z/`, `REVIEW_DAYS = (30..365)`, `DEFAULT_REVIEW_DAYS = 90`, `DEFAULT_NIGHT_LIMITS = { "caps_iii" => { "consecutive" => 7, "alternate" => 10, "window_days" => 30 }, "caps_ad_iii" => { "total" => 14, "window_days" => 30 }, "caps_ad_iv" => { "total" => 14, "window_days" => 30 } }`, `WINDOW_DAYS = (7..90)`, `TRAIL_ACCESS = "psychosocial"`, `PROBLEM_CODES = %w[patient_cns_missing phone_missing homeless_missing professional_cns_missing professional_not_in_cnes cid_missing no_actions action_out_of_rule night_stay_open]`.
5. **`config/psychosocial/raas.yml`** (`version: 1`, `source: "Manual Operacional RAAS v1.5 / Portaria SAS 854/2012"`, `review: true` — conferência com o aplicativo RAAS vigente é gate):
   - `layout_version: null`; `night_sigtap_code: "0301080020"`; `family_care_codes: ["0301080224"]`.
   - `origins: { "01": "Demanda espontânea", "02": "Atenção básica", "03": "Serviço de urgência e emergência", "04": "Outro CAPS", "05": "Hospital geral", "06": "Hospital psiquiátrico", "07": "Comunidade terapêutica", "08": "Unidade de acolhimento", "09": "Serviço residencial terapêutico", "10": "Rede intersetorial", "11": "Outros" }`.
   - `destinations: { "00": "Permanência", "01": "Alta", "02": "Transferência para outro CAPS", "03": "Encaminhamento para a atenção básica", "04": "Abandono", "05": "Óbito", "06": "Internação", "07": "Outros" }`.
   - `substances: { "01": "Álcool", "02": "Crack", "03": "Cocaína", "04": "Maconha", "05": "Solventes", "06": "Outras", "99": "Nenhuma" }` (`"99"` sozinho; com outro código → `invalid_request` `path: "substances"`).
   - `race_colors`/`ethnicities`: não entram no YAML — `Patient::RACE_COLORS` e a etnia do cadastro (`Patient#ethnicity`).
   - `Psychosocial::Raas.config` lê uma vez por processo e congela; chaves desconhecidas → erro no boot do spec `spec/config/psychosocial_raas_spec.rb`.
6. **SIGTAP de dev** (`lib/sigtap_sample.rb`, `PSYCHOSOCIAL_CODES`; conferência no SIGTAP oficial é gate): `0301080208` atend. individual (09, máx. 31), `0301080216` grupo (09, máx. 31), `0301080224` familiar (09, máx. 31), `0301080240` domiciliar (09, máx. 31), `0301080283` práticas expressivas (09), `0301080291` crise (09), `0301080348` reabilitação (09), `0301080194` acolhimento diurno (09), `0301080020` acolhimento noturno (09, máx. 1, permanência 14), `0301080232` acolhimento inicial (02 e 09, máx. 1), `0301080259` articulação de redes (01), `0301080267` protagonismo (01), `0301080305` matriciamento AB (01), `0301080313` redução de danos (01), `0301080321` residencial terapêutico (01), `0301080399` matriciamento urgência (01); CBOs `225133 225125 251510 223505 251605 223905 322205` em todos; atributo `036` só no `0301080291` (para o teste; no oficial, gate); `REGISTRIES` ganha `"09" => "RAAS (PSICOSSOCIAL)"`.
7. **Dev:** api do worktree na porta **3053**; worktree `apps/api/.claude/mod31` (com `config/master.key`); branch `feat/mod-31-psychosocial-care`; migrações `db/platform_migrate/20261104900001_add_sigtap_procedure_limits.rb`, `db/city_migrate/20261104900002_add_psychosocial_care.rb`, `20261104900003_add_psychosocial_billing_source.rb`, `20261104900004_add_psychosocial_documents.rb`. Semente (Task 17): Curitiba — `psychosocial_care` ligado; "CAPS II Boa Vista" (`caps_ii`, classificação `001`) e "CAPS AD III Centro Vida" (`caps_ad_iii`, classificação `001`, leitos L1–L4), CNES de dev; coordenação `caps@curitiba.dev` (`psychosocial_lead`, enfermeira 223505 no AD III e no II), psiquiatra `capsmed@curitiba.dev` (225133, nos dois), psicóloga `capspsi@curitiba.dev` (251510, CAPS II), técnica de enfermagem `capstec@curitiba.dev` (322205, AD III) — todos `psychosocial_staff`, TOTP fixo; quatro usuários fictícios com CPF e CNS válidos (`Professionals::Cns.generate`): "Renato Alves Pereira" (F200, CAPS II, PTS em dia), "Luciana Rocha Mendes" (F321, CAPS II, PTS vencido), "Marcos Vinícius Teixeira" (F102, substâncias 01, AD III, permanência aberta no L2), "Patrícia Gomes Lima" (F142, substâncias 02 e 04, AD III); grupo "Oficina de artesanato" (quarta 14:00–16:00, `0301080216`) e "Assembleia de usuários" (sexta 10:00); um matriciamento com a equipe da UBS do `CareLinesCrew` com um caso compartilhado do Renato; vínculos CNES dos quatro profissionais no retrato do CNES de Curitiba.

## Global Constraints

- git `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta); nunca `git add -A` (sempre os arquivos da task, por nome); commits em inglês, Conventional Commits por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`; merge, push e tag só com autorização do usuário.
- **Primeira ação da execução:** mover os cards F-31.1..F-31.8 do board #1 para In Progress (`gh project item-edit …`, um comando `gh` por chamada, começando com `gh`). Card vai para Done quando o ÚLTIMO app dele terminar (F-31.1 também dashboard e maintenance; F-31.2..F-31.7 também dashboard; F-31.8 também dashboard e wpda); Verified depois da prova no navegador.
- Testes só por `bin/test api -w mod31 <specs>` (a partir de `/Users/eduardovrocha/Development/ioit.solutions/rota-saude`); nunca `rspec` direto nem o Postgres do host. Mudança transversal (19, 22, SIGTAP da plataforma, `ck_memberships_role`, `spec/support`) → **suíte completa obrigatória antes do merge**, pedida à sessão "Testes" (Task 18).
- Migrações: dump à mão em `db/city_schema.rb` (`define(version: 2026_11_04_900004)`) e `db/platform_schema.rb` (`define(version: 2026_11_04_900001)`); triggers em `db/city_triggers.sql`/`db/platform_triggers.sql`, executados no fim do `up` de quem os muda. Todas irreversíveis. **O banco de dev só é migrado/semeado na prova no navegador, com autorização do usuário.**
- Specs de requisição com `type: :request`; nunca simule cidade inalcançável com URL real.
- Erros `{ "error", "path"? }` + `problems: [ { code, participant_index? } ]` no 422 da crítica da ação e `problems: [ { path, error } ]` nas recusas de campo; 409 de conflito no padrão do 30; interruptor 403 `{ "error": "feature_disabled", "feature": "psychosocial_care" }`; step-up 401 `mfa_required`; Content-Type JSON em toda escrita por cookie; listas com envelope `{ items, next_before? }`.
- **CPF, CNS, nome, CID, substância e qualquer texto clínico do CAPS nunca em log, evento, Analytics, erro ou URL** (busca `q` só no corpo do `GET` como query `q` já filtrada; ids na URL). Nada do CAPS no wpda, em aviso, SMS, LEDI, na linha do tempo do cidadão e do PEC. Modelos novos com dado do paciente sobrescrevem `inspect`/`to_s`/`pretty_print`/`as_json` (só `id`). Filtro de log novo: `/\A(cid_main|cid_secondary|substances|needs|goals|family_name|topic|participants|reassessment|shared_cases)\z/` (`reason`, `notes`, `shared_text` já caem em `:reason`, `:note`, `:text`).
- Eventos declarados em `config/initializers/domain_events.rb` e em `spec/initializers/domain_events_bindings_spec.rb`, payload só com ids (contrato §13); `DomainEvents.publish(` é multilinha — confira cada chamada inteira. Nenhum `Platform.audit` novo (R18 intacto); se surgir, entra em `R18_PLATFORM_EVENT_NAMES`.
- Nunca `Current.city =` solto; interruptores lidos **fora** da transação da cidade.
- Cifra por cidade: todo `encrypts` novo em `CityEncryption::CITY_KEYED_TARGETS` (guarda `spec/architecture/city_encrypted_attributes_guard_spec.rb`); coluna cifrada de tabela imutável no padrão `rota_reencryption_only(...)`.
- `spec/adr_pointers_spec.rb` sobe para `VALID_RANGE = (1..49)`.
- Confira nomes reais no código antes de usá-los (o PRÉ-REQUISITO lista os conferidos).

## Review Focus

1. **Grupo com um participante de matrícula de outro CAPS** (ou encerrada ontem): 422 com `{ code: "enrollment_required", participant_index: 2 }`, nada gravado — nem a ação, nem participante, nem RAAS do mês, nem evento; os demais participantes não recebem a ação "parcial". Teste na Task 7.
2. **Ação às 23:30 de 30/11 (hora local; 02:30 UTC de 01/12)** com o mês de novembro já fechado e dezembro aberto: recusada com `month_closed` (o mês é o local, não o UTC); às 23:30 de 30/11 com novembro aberto, conta na RAAS de novembro e a crítica lê a SIGTAP de `202611`. Teste na Task 7.
3. **Permanência noturna que atravessa a virada do mês:** fechar o mês anterior com o usuário ainda no leito é recusado (`night_stay_open`); fechar a permanência depois gera uma ação por noite, cada uma no mês da sua noite; nenhuma noite cai num mês fechado. Teste na Task 9.
4. **SIGTAP importada antes do módulo** (`max_quantity`, `max_stay_days` e atributos nulos/ausentes) e pacote antigo sem o arquivo de atributos: a importação passa (zero atributos), a crítica não inventa limite (nulo = sem restrição) e nenhuma ação é recusada por `quantity_exceeded`/`authorization_required` por falta de dado. Teste na Task 1.
5. **Recálculo do 22 no fechamento da competência** com uma linha BPA-C sem paciente cujo código perdeu o instrumento `01` na SIGTAP da competência: o fechamento não estoura CHECK nem 500 — a linha fica `not_bpa`, não faturável (`not_bpa_instrument`), fora do arquivo. Teste na Task 12.

## Mapa de arquivos

**Criar** (sob `apps/api/.claude/mod31/`)
- Migrações: `db/platform_migrate/20261104900001_add_sigtap_procedure_limits.rb`; `db/city_migrate/{20261104900002_add_psychosocial_care,20261104900003_add_psychosocial_billing_source,20261104900004_add_psychosocial_documents}.rb`.
- `app/models/sigtap_procedure_attribute.rb`; `app/models/{psychosocial_setting,psychosocial_bed,psychosocial_group,psychosocial_referral,psychosocial_reception,psychosocial_enrollment,psychosocial_care_plan,psychosocial_care_plan_alert,psychosocial_action,psychosocial_action_participant,psychosocial_monthly_record,psychosocial_month_closure,psychosocial_raas_export,psychosocial_matrix_support,psychosocial_matrix_case,psychosocial_night_stay}.rb`.
- `config/psychosocial/raas.yml`.
- `app/services/psychosocial/{gate,catalog,raas,access,trail,json,settings,instrument,action_rules,monthly_records,month_problems,cnes_check,night_limit,raas_file,mirror_sheet_pdf,today,dashboard,link,anonymize}.rb`.
- `app/commands/psychosocial/{save_settings,save_bed,create_referral,record_reception,create_enrollment,update_enrollment,close_enrollment,save_care_plan,save_group,record_action,cancel_action,start_night_stay,close_night_stay,update_monthly_record,close_month,reopen_month,export_raas,record_matrix_support,add_matrix_case}.rb`; `app/commands/billing/from_psychosocial.rb`.
- `app/controllers/concerns/psychosocial_gate.rb`; `app/controllers/psychosocial/{base,settings,beds,referrals,receptions,enrollments,care_plans,groups,today,actions,documents,night_stays,monthly_records,months,raas_exports,matrix_supports,timelines,dashboard}_controller.rb`; `app/controllers/patient_psychosocial_links_controller.rb`.
- `app/jobs/psychosocial/{open_month_job,care_plan_review_job}.rb`; `app/jobs/billing/psychosocial_production_job.rb`.
- `app/services/timeline/sources/{psychosocial_source,psychosocial_link_source}.rb`.
- `lib/psychosocial_crew.rb`; `spec/support/psychosocial_helpers.rb`; specs listados em cada task.

**Modificar**
- Plataforma/SIGTAP: `app/services/terminology/sigtap_reader.rb`, `app/services/billing/sigtap_cache.rb`, `app/models/sigtap_procedure.rb`, `db/platform_schema.rb`, `db/platform_triggers.sql`, `lib/sigtap_sample.rb`.
- Cidade: `app/services/platform/features.rb`, `app/models/{membership,health_unit,performed_procedure,clinical_document}.rb`, `app/controllers/health_units_controller.rb`, `app/services/city_encryption.rb`, `db/city_schema.rb`, `db/city_triggers.sql`, `config/routes.rb`, `config/recurring.yml`, `config/initializers/{domain_events,filter_parameter_logging}.rb`.
- 22: `app/commands/billing/record.rb`, `app/services/billing/{critic,lines,json,pending}.rb`.
- 19: `app/commands/clinical_documents/issue.rb`, `app/services/clinical_documents/{read,pdf,verification,json}.rb`, `app/services/signatures/canonical.rb`, `app/controllers/clinical_documents_controller.rb` (linha 119).
- Sigilo: `app/services/timeline.rb`, `app/services/timeline/{query,cursor}.rb`, `app/services/timeline/sources/{clinical_document_source,controlled_notification_source,procedure_source,dispensation_source}.rb`, `app/services/timeline/citizen_documents.rb`, `app/controllers/pec_timelines_controller.rb`, `app/services/care_lines/sources.rb`, `app/jobs/ledi/{consultation_ficha_job,screening_ficha_job}.rb`, `app/commands/citizens/{request_erasure,erase}.rb`.
- Specs existentes: `spec/adr_pointers_spec.rb`, `spec/models/{create_analytics_migration_spec,create_campaigns_migration_spec,membership_roles_spec,transport_schema_spec}.rb`, `spec/initializers/domain_events_bindings_spec.rb`, `spec/services/platform/features_spec.rb`, `spec/services/timeline/query_spec.rb`, `db/seeds.rb` (chama a crew).

---

### Task 0: Worktree, cards e conferência do pré-requisito

**Files:** nenhum código.

- [ ] **Step 1:** `gh project item-edit` de F-31.1..F-31.8 para In Progress (um comando `gh` por chamada).
- [ ] **Step 2:** `cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/api && /opt/homebrew/bin/git fetch origin && /opt/homebrew/bin/git worktree add .claude/mod31 -b feat/mod-31-psychosocial-care origin/main && cp config/master.key .claude/mod31/config/master.key`.
- [ ] **Step 3:** conferir o PRÉ-REQUISITO com `grep` no worktree (`VALID_RANGE`, `AddSanitaryTransport::ROLES` nos quatro specs, `HealthUnit::KINDS`, `PerformedProcedure::SOURCES`, `Critic::Facts.new` nos dois chamadores, `ck_clinical_documents_*`, `Timeline::KINDS`, `PecTimelinesController::KINDS`, `QT_MAXIMA_EXECUCAO` no `SigtapSample::LAYOUTS`); diferença vira nota no topo do plano antes da Task 1.
- [ ] **Step 4:** `bin/test api -w mod31 spec/adr_pointers_spec.rb spec/services/platform/features_spec.rb spec/models/membership_roles_spec.rb spec/services/terminology` → PASS (linha de base).

---

### Task 1: SIGTAP da plataforma — quantidade máxima, permanência e atributos (transversal, §12)

**Files:**
- Create: `db/platform_migrate/20261104900001_add_sigtap_procedure_limits.rb`, `app/models/sigtap_procedure_attribute.rb`, `spec/services/terminology/sigtap_reader_limits_spec.rb`, `spec/services/billing/sigtap_cache_limits_spec.rb`
- Modify: `db/platform_schema.rb`, `db/platform_triggers.sql` (a tabela nova entra no `FOREACH` do `terminology_codes_guard`), `app/services/terminology/sigtap_reader.rb`, `app/services/billing/sigtap_cache.rb`, `lib/sigtap_sample.rb`

**Interfaces:**
- Produces: colunas `sigtap_procedures.max_quantity integer NULL` e `max_stay_days integer NULL` (CHECK `> 0` quando não nulas); tabela `sigtap_procedure_attributes` (`release_id uuid NOT NULL`, `procedure_code varchar(10)`, `attribute_code varchar(3)`, índice único `idx_sigtap_procedure_attributes_unique (release_id, procedure_code, attribute_code)`, imutável pelo mesmo guard); `SigtapReader::ATTRIBUTE_FILE = "rl_procedimento_detalhe"`, `ATTRIBUTE_COLUMN = "CO_DETALHE"`, `ATTRIBUTE = /\A\d{3}\z/` (D1: nome do arquivo a conferir no pacote oficial); `QT_MAXIMA_EXECUCAO` `0`/`9999`/vazio → nulo; `QT_DIAS_PERMANENCIA` `0`/`9999`/vazio → nulo; arquivo de atributos ausente no ZIP → `counts[:sigtap_procedure_attributes] = 0`, sem erro; `Billing::SigtapCache#max_quantity(release, code) -> Integer|nil`, `#max_stay_days(release, code) -> Integer|nil`, `#attributes(release, code) -> Set<String>`; `SigtapSample.write_to(dir, competence:)` escreve `rl_procedimento_detalhe` e `tb_detalhe` e os códigos do Valor fixado 6 (com `MAX_QUANTITIES`, `STAY_DAYS`, `ATTRIBUTES`), `SigtapSample::PSYCHOSOCIAL_CODES`.

- [ ] **Step 1: Write the failing tests** (trecho de `sigtap_reader_limits_spec.rb`):

```ruby
RSpec.describe Terminology::SigtapReader do
  def import(dir, competence = "202611")
    Terminology::Import.call(kind: "sigtap", version: competence, path: dir, by: "spec")
  end

  it "grava quantidade máxima, permanência e atributos; 9999 e 0 viram nulo" do
    result = Dir.mktmpdir { |dir| import(SigtapSample.write_to(dir, competence: "202611")) }
    release = result.payload[:release]
    night = SigtapProcedure.find_by!(release_id: release.id, code: "0301080020")
    expect([ night.max_quantity, night.max_stay_days ]).to eq([ 1, 14 ])
    consult = SigtapProcedure.find_by!(release_id: release.id, code: "0301010064")
    expect([ consult.max_quantity, consult.max_stay_days ]).to eq([ nil, nil ])
    expect(SigtapProcedureAttribute.where(release_id: release.id, procedure_code: "0301080291").pluck(:attribute_code)).to eq(%w[036])
    expect(result.payload[:counts]).to include(sigtap_procedure_attributes: 1)
  end

  it "Review Focus 4: pacote sem o arquivo de atributos importa com zero atributos" do
    result = Dir.mktmpdir do |dir|
      SigtapSample.write_to(dir, competence: "202611")
      FileUtils.rm(Dir.glob(File.join(dir, "rl_procedimento_detalhe*")))
      import(dir)
    end
    expect(result).to be_ok
    expect(result.payload[:counts]).to include(sigtap_procedure_attributes: 0)
  end

  it "atributo com código fora de 3 dígitos recusa a release inteira" do
    result = Dir.mktmpdir do |dir|
      SigtapSample.write_to(dir, competence: "202611")
      path = File.join(dir, "rl_procedimento_detalhe.txt")
      File.binwrite(path, File.binread(path).sub("036", "3X6"))
      import(dir)
    end
    expect(result.reason).to eq(:invalid_file)
    expect(TerminologyRelease.where(kind: "sigtap", version: "202611", status: "active")).to be_empty
  end

  it "linha de atributo de release ativa é imutável" do
    release = Dir.mktmpdir { |dir| import(SigtapSample.write_to(dir, competence: "202611")) }.payload[:release]
    row = SigtapProcedureAttribute.find_by!(release_id: release.id)
    expect { row.update_column(:attribute_code, "009") }.to raise_error(ActiveRecord::StatementInvalid, /immutable/)
  end
end
```

`sigtap_cache_limits_spec.rb`: com a release importada, `cache.max_quantity(release, "0301080208") == 31`, `cache.attributes(release, "0301080291") == Set["036"]`, `cache.attributes(release, "0301080208") == Set[]`; uma release antiga (gravada direto sem as colunas, nulas) → `max_quantity` nil e `attributes` vazio; a segunda leitura do mesmo código não faz consulta (`expect { … }.not_to exceed_query_limit(0)` ou contagem de `sql.active_record`, no padrão do spec existente do cache).

- [ ] **Step 2:** `bin/test api -w mod31 spec/services/terminology/sigtap_reader_limits_spec.rb spec/services/billing/sigtap_cache_limits_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.** Migração: `add_column` × 2 com CHECK `ck_sigtap_procedures_limits` (`(max_quantity IS NULL OR max_quantity > 0) AND (max_stay_days IS NULL OR max_stay_days > 0)`), `create_table :sigtap_procedure_attributes` (sem `id` uuid: `force: :cascade` como as outras relações, `bigint`), `execute File.read(Rails.root.join("db/platform_triggers.sql"))`. Leitor: `procedure_row` ganha `max_quantity: limit(r["QT_MAXIMA_EXECUCAO"], code)`, `max_stay_days: limit(r["QT_DIAS_PERMANENCIA"], code)` (`limit` = inteiro; `0`, `9999` ou vazio → nil; não numérico → `Import::Invalid`); o stream de atributos só roda se o arquivo `ATTRIBUTE_FILE.txt` existir no arquivo (`@archive` já sabe listar — confira `OfficialArchive#each_line` e acrescente `#include?(pattern)` se faltar). Cache: memo como os outros (`:max_quantity`, `:max_stay_days` lidos da `procedure`; `:attributes` do scope).
- [ ] **Step 4:** mesmos specs + `spec/services/terminology spec/services/billing spec/lib` → PASS.
- [ ] **Step 5: Commit** `feat: import sigtap maximum quantity, stay days and complementary attributes`.

---

### Task 2: Esquema da cidade do CAPS, modelos, triggers e helpers

**Files:**
- Create: `db/city_migrate/20261104900002_add_psychosocial_care.rb`; os 16 modelos `psychosocial_*` do mapa; `spec/support/psychosocial_helpers.rb`; `spec/models/psychosocial_schema_spec.rb`
- Modify: `db/city_schema.rb`, `db/city_triggers.sql`, `app/models/{membership,health_unit}.rb`, `app/services/city_encryption.rb`, `config/initializers/filter_parameter_logging.rb`, `spec/adr_pointers_spec.rb`, `spec/models/{create_analytics_migration_spec,create_campaigns_migration_spec,membership_roles_spec,transport_schema_spec}.rb`

**Interfaces:**
- Produces (tabelas, `id: :uuid` salvo nota; FKs com `foreign_key: true`; todas com `created_at`):
  - `psychosocial_settings` (smallint `id` = 1, inserida na migração; `night_limits jsonb` default = `DEFAULT_NIGHT_LIMITS` CHECK `jsonb_typeof = 'object'`; `care_plan_review_days int` default 90 CHECK 30..365; `updated_by_user_id`; `updated_at`).
  - `health_units`: `caps_modality` (string nulo), `service_classification` (string 3 nulo); `ck_health_units_kind` + `caps`; `ck_health_units_caps` (`(kind = 'caps') = (caps_modality IS NOT NULL) AND (caps_modality IS NULL OR caps_modality = ANY(...)) AND (service_classification IS NULL OR (kind = 'caps' AND service_classification ~ '^[0-9]{3}$'))`).
  - `psychosocial_beds` (`health_unit_id`, `label` 20, `active`, `updated_by_user_id`, timestamps; único `(health_unit_id, label)`).
  - `psychosocial_groups` (`health_unit_id`, `name` 120, `kind`, `weekday smallint` CHECK 1..7, `starts_at time`, `ends_at time` CHECK `> starts_at`, `responsible_user_id`, `capacity` CHECK 1..100, `sigtap_code` 10, `active`, `updated_by_user_id`, timestamps).
  - `psychosocial_referrals` (`patient_id`, `health_unit_id` (CAPS), `from_health_unit_id`, `reason` cifrado nulo, `priority`, `status` default `pending`, `referred_by_user_id`, `received_at`, `anonymized_at`, timestamps; índice `(health_unit_id, status, created_at)`).
  - `psychosocial_receptions` (`health_unit_id`, `patient_id`, `psychosocial_referral_id` nulo único, `at`, `sigtap_code` 10, `notes` cifrado nulo, `outcome`, `recorded_by_user_id`, `anonymized_at`, `created_at`).
  - `psychosocial_enrollments` (`health_unit_id`, `patient_id`, `psychosocial_reception_id` nulo, `started_on`, `origin` 2, `cid_main` cifrado, `cid_secondary` cifrado (JSON), `substances` cifrado (JSON), `esf_covered` bool, `esf_cnes` 7 nulo CHECK `esf_covered = (esf_cnes IS NOT NULL)`, `reference_user_id`, `next_visit_at` nulo, `destination` 2 nulo, `closed_on` nulo, `closed_by_user_id` nulo, timestamps; CHECK `(closed_on IS NULL) = (destination IS NULL) AND (destination IS NULL OR destination <> '00')`; único parcial `idx_psychosocial_enrollments_open (patient_id, health_unit_id) WHERE closed_on IS NULL`; índice `(health_unit_id, next_visit_at)`).
  - `psychosocial_care_plans` (`psychosocial_enrollment_id`, `version` CHECK ≥ 1, `needs` cifrado, `goals` cifrado (JSON), `intensity`, `review_on`, `created_by_user_id`, `created_at`; único `(enrollment, version)`).
  - `psychosocial_care_plan_alerts` (`psychosocial_care_plan_id` único, `created_at`).
  - `psychosocial_actions` (`health_unit_id`, `sigtap_code` 10, `instrument`, `performed_at`, `month date` CHECK dia 1, `place`, `quantity` CHECK 1..999, `notes` cifrado nulo, `authorization_number` 13 nulo, `psychosocial_group_id` nulo, `professional_user_id`, `cbo_code` 6, `origin` default `manual`, `psychosocial_night_stay_id` nulo, `cancelled_at`, `cancel_reason` cifrado, `cancelled_by_user_id`, `created_at`; CHECK `(cancelled_at IS NULL) = (cancel_reason IS NULL) AND (cancelled_at IS NULL) = (cancelled_by_user_id IS NULL)`; índices `(health_unit_id, month)`).
  - `psychosocial_action_participants` (`psychosocial_action_id`, `position smallint`, `psychosocial_enrollment_id` nulo, `patient_id` nulo, `family_name` cifrado nulo, `created_at`; CHECK `num_nonnulls(patient_id, family_name) = 1 AND (psychosocial_enrollment_id IS NULL OR patient_id IS NOT NULL)`; único `(action, position)`; índices `(psychosocial_enrollment_id)`, `(patient_id)`).
  - `psychosocial_monthly_records` (`psychosocial_enrollment_id`, `health_unit_id`, `month date` CHECK dia 1, `homeless` bool nulo, `destination` 2 nulo, `concluded_on` nulo, `status` default `open`, `snapshot` cifrado nulo, `closed_at`, `psychosocial_raas_export_id` nulo, timestamps; único `(enrollment, month)`; CHECK `(status = 'open') = (snapshot IS NULL)`).
  - `psychosocial_month_closures` (`health_unit_id`, `month`, `closed_by_user_id`, `closed_at`, `reopened_at`, `reopened_by_user_id`, `reopen_reason` cifrado; único parcial `(health_unit_id, month) WHERE reopened_at IS NULL`).
  - `psychosocial_raas_exports` (`health_unit_id`, `month`, `competence` 6, `billing_period_id`, `layout_version` 20, `file_data` cifrado (Base64), `sha256` 64, `records_count`, `created_by_user_id`, `created_at`; único `(health_unit_id, month)`).
  - `psychosocial_matrix_supports` (`health_unit_id` (CAPS), `health_team_id`, `held_on`, `topic` cifrado, `participant_user_ids uuid[]` CHECK `cardinality >= 1`, `sigtap_code` 10, `psychosocial_action_id` único, `recorded_by_user_id`, `created_at`).
  - `psychosocial_matrix_cases` (`psychosocial_matrix_support_id`, `patient_id`, `shared_text` cifrado, `created_by_user_id`, `created_at`; índice `(patient_id)`).
  - `psychosocial_night_stays` (`psychosocial_bed_id`, `health_unit_id`, `psychosocial_enrollment_id`, `patient_id`, `started_at`, `expected_end_at` CHECK `> started_at`, `ended_at` nulo CHECK `> started_at`, `end_reason` cifrado nulo, `reassessment_reason` cifrado nulo, `reassessed_by_user_id` nulo, `started_by_user_id`, `closed_by_user_id`, timestamps; `EXCLUDE USING gist (psychosocial_bed_id WITH =, tsrange(started_at, COALESCE(ended_at, 'infinity'::timestamp)) WITH &&)` `excl_psychosocial_night_stays_bed`; único parcial `idx_psychosocial_night_stays_open_patient (patient_id) WHERE ended_at IS NULL`; CHECK `(ended_at IS NULL) = (end_reason IS NULL)`, `(reassessment_reason IS NULL) = (reassessed_by_user_id IS NULL)`).
- Produces (travas): `ck_memberships_role` com `AddPsychosocialCare::ROLES`; triggers no bloco "Módulo 31 (ADR 0049)" de `db/city_triggers.sql`: `rota_psychosocial_action_guard()` (DELETE recusado; INSERT recusado com fechamento ativo de `(health_unit_id, month)` — `'psychosocial_actions: month closed'`; UPDATE só das colunas de cancelamento, uma vez, ou sob `rota.reencrypting`); `rota_append_only()` em `psychosocial_action_participants`, `psychosocial_care_plans`, `psychosocial_care_plan_alerts`, `psychosocial_matrix_supports`, `psychosocial_matrix_cases`, `psychosocial_raas_exports` (cifradas via `rota_reencryption_only`); `rota_psychosocial_monthly_record_guard()` (DELETE recusado; `exported` nunca muda; `enrollment_id`/`month`/`health_unit_id` nunca mudam); `rota_psychosocial_night_stay_guard()` (DELETE recusado; fechada nunca muda, salvo re-cifra; aberta só recebe o fechamento); `rota_psychosocial_month_closure_guard()` (só `reopened_*` uma vez); `rota_no_delete()` em `psychosocial_referrals`, `psychosocial_receptions`, `psychosocial_enrollments`, `psychosocial_beds`, `psychosocial_groups`, `psychosocial_settings`; TRUNCATE recusado nas 16 tabelas.
- Produces (helpers, `spec/support/psychosocial_helpers.rb`): `psychosocial_city!(billing: false)` (`clinical_city!` ou, com `billing: true`, `billing_city!` com `psychosocial_sigtap!` ANTES; liga `psychosocial_care`; `PsychosocialSetting.ensure_defaults!`); `psychosocial_sigtap!(competences = [hoje, mês anterior])` (as releases com o recorte do Valor fixado 6, pelo `Terminology::Import` + `SigtapSample`); `caps!(modality: "caps_ii", name: "CAPS #{hex}", cnes: billing_cnes)`; `caps_staff!(unit, cbo: "251510", lead: false)` (usuário `psychosocial_staff` ou `psychosocial_lead` + `Professional` + `ProfessionalLink` ativo, `mfa_verified_at` recente); `caps_patient!(cid: "F200", cns: true)` (paciente com par validado e telefone, `Professionals::Cns.generate` quando `cns`); `enrollment!(unit, patient: caps_patient!, reference: caps_staff!(unit), started_on: Date.current.beginning_of_month)` (gravação direta + RAAS do mês); `bed!(unit, label: "L1")`; `ubs_doctor!(ubs)` (`health_professional` com vínculo 225125 na UBS); `psychosocial_admin!`.

- [ ] **Step 1: Write the failing test** `spec/models/psychosocial_schema_spec.rb`:

```ruby
RSpec.describe "Esquema do módulo 31", type: :model do
  let!(:city) { psychosocial_city! }
  let(:caps) { caps!(modality: "caps_ad_iii") }

  it "o banco recusa duas permanências que se cruzam no mesmo leito; encostadas valem" do
    bed = bed!(caps)
    a = enrollment!(caps)
    b = enrollment!(caps)
    stay = ->(enrollment, from, to) do
      PsychosocialNightStay.create!(psychosocial_bed: bed, health_unit: caps, psychosocial_enrollment: enrollment, patient_id: enrollment.patient_id,
                                    started_at: from, expected_end_at: to, ended_at: to, end_reason: "Alta do acolhimento",
                                    started_by_user: a.reference_user, closed_by_user: a.reference_user)
    end
    stay.call(a, 2.days.ago.change(hour: 19), 1.day.ago.change(hour: 8))
    expect { stay.call(b, 1.day.ago.change(hour: 8), 1.day.ago.change(hour: 20)) }.not_to raise_error
    expect { stay.call(b, 1.day.ago.change(hour: 7), 1.day.ago.change(hour: 9)) }
      .to raise_error(ActiveRecord::ExclusionViolation, /excl_psychosocial_night_stays_bed/)
  end

  it "uma matrícula aberta por paciente e CAPS; encerrada libera" do
    first = enrollment!(caps)
    expect { enrollment!(caps, patient: first.patient) }.to raise_error(ActiveRecord::RecordNotUnique, /idx_psychosocial_enrollments_open/)
    first.update!(destination: "01", closed_on: Date.current, closed_by_user: first.reference_user)
    expect { enrollment!(caps, patient: first.patient) }.not_to raise_error
  end

  it "destino 00 nunca encerra; CAPS exige modalidade; modalidade só no CAPS" do
    e = enrollment!(caps)
    expect { e.update!(destination: "00", closed_on: Date.current, closed_by_user: e.reference_user) }.to raise_error(ActiveRecord::StatementInvalid)
    expect { HealthUnit.create!(name: "CAPS sem modalidade", kind: "caps") }.to raise_error(ActiveRecord::StatementInvalid, /ck_health_units_caps/)
    expect { HealthUnit.create!(name: "UBS com modalidade", kind: "ubs", caps_modality: "caps_ii") }.to raise_error(ActiveRecord::StatementInvalid)
  end

  it "ação é imutável: só o cancelamento, uma vez; mês fechado recusa inserção" do
    staff = caps_staff!(caps)
    action = PsychosocialAction.create!(health_unit: caps, sigtap_code: "0301080208", instrument: "raas_psi", performed_at: 1.hour.ago,
                                        month: Date.current.beginning_of_month, place: "caps", quantity: 1, professional_user: staff, cbo_code: "251510")
    expect { action.update_column(:quantity, 2) }.to raise_error(ActiveRecord::StatementInvalid)
    action.update!(cancelled_at: Time.current, cancel_reason: "Registrada no usuário errado", cancelled_by_user: staff)
    expect { action.update!(cancel_reason: "Outro motivo qualquer") }.to raise_error(ActiveRecord::StatementInvalid)
    PsychosocialMonthClosure.create!(health_unit: caps, month: Date.current.beginning_of_month, closed_by_user: staff, closed_at: Time.current)
    expect { action.dup.tap { |copy| copy.assign_attributes(cancelled_at: nil, cancel_reason: nil, cancelled_by_user: nil) }.save! }
      .to raise_error(ActiveRecord::StatementInvalid, /month closed/)
  end

  it "versão do PTS e participante nunca mudam nem somem" do
    e = enrollment!(caps)
    plan = PsychosocialCarePlan.create!(psychosocial_enrollment: e, version: 1, needs: "Isolamento social", goals: [].to_json,
                                        intensity: "intensive", review_on: Date.current + 90, created_by_user: e.reference_user)
    expect { plan.update_column(:intensity, "non_intensive") }.to raise_error(ActiveRecord::StatementInvalid)
    expect { plan.delete }.to raise_error(ActiveRecord::StatementInvalid)
  end

  it "papéis entram nas travas; nada clínico em inspect/as_json" do
    expect(Membership::PRIVILEGED_ROLES).to include("psychosocial_staff", "psychosocial_lead")
    e = enrollment!(caps)
    expect([ e.inspect, e.as_json.to_s ].join).not_to include("F200")
    expect(e.as_json).to eq("id" => e.id)
  end
end
```

- [ ] **Step 2:** `bin/test api -w mod31 spec/models/psychosocial_schema_spec.rb` → FAIL (`psychosocial_city!` indefinido).
- [ ] **Step 3: Implementar** a migração no padrão da do 30 (constantes espelhadas no topo — `ROLES`, `HEALTH_UNIT_KINDS` (+ `caps`), `CAPS_MODALITIES` e as listas do Valor fixado 4 —, `replace_check`, `quoted`, `add_exclusion_constraint ... using: :gist`, `INSERT INTO psychosocial_settings (id, night_limits, updated_at) VALUES (1, '<DEFAULT_NIGHT_LIMITS em JSON>', now()) ON CONFLICT DO NOTHING`, `execute File.read(Rails.root.join("db/city_triggers.sql"))` no fim, `down` irreversível). Modelos: `encrypts` (todas as colunas "cifrado" acima) em `CITY_KEYED_TARGETS`; `cid_secondary`, `substances`, `goals` com `serialize ..., coder: JSON` sobre o texto cifrado; os modelos com paciente (`referral`, `reception`, `enrollment`, `care_plan`, `action`, `action_participant`, `monthly_record`, `matrix_case`, `night_stay`, `raas_export`) com `inspect`/`to_s`/`pretty_print`/`as_json` só `id`; `PsychosocialAction#cancelled?`; `PsychosocialEnrollment#open?`. `HealthUnit` (Valor fixado 3, validações espelham o CHECK). `Membership` com o comentário do padrão (`psychosocial_staff`/`psychosocial_lead`: leem CID, substâncias, evolução e PTS do CAPS — step-up; o mantenedor não concede). Os quatro specs que citam `AddSanitaryTransport::{ROLES,SOURCES,HEALTH_UNIT_KINDS}` passam a `AddPsychosocialCare::ROLES`/`::HEALTH_UNIT_KINDS` (o `transport_schema_spec` compara `AddSanitaryTransport::HEALTH_UNIT_KINDS` com `HealthUnit::KINDS - %w[caps]` e `::SOURCES` com `PerformedProcedure::SOURCES - %w[psychosocial]`, preparando a Task 12); `membership_roles_spec` ganha os dois papéis. Dump à mão; filtro de log; `VALID_RANGE = (1..49)`.
- [ ] **Step 4:** `bin/test api -w mod31 spec/models/psychosocial_schema_spec.rb spec/models/membership_roles_spec.rb spec/models/create_analytics_migration_spec.rb spec/models/create_campaigns_migration_spec.rb spec/models/transport_schema_spec.rb spec/architecture/city_encrypted_attributes_guard_spec.rb spec/adr_pointers_spec.rb spec/requests/health_units_spec.rb` → PASS.
- [ ] **Step 5: Commit** `feat: add psychosocial care schema with bed exclusion and immutability triggers`.

---

### Task 3: Interruptor, portão, lotação, trilha, configuração, CAPS no cadastro e leitos

**Files:**
- Create: `app/services/psychosocial/{gate,catalog,raas,access,trail,json,settings}.rb`, `config/psychosocial/raas.yml`, `app/controllers/concerns/psychosocial_gate.rb`, `app/controllers/psychosocial/{base,settings,beds}_controller.rb`, `app/commands/psychosocial/{save_settings,save_bed}.rb`, `spec/services/psychosocial/access_spec.rb`, `spec/requests/psychosocial/admin_spec.rb`, `spec/config/psychosocial_raas_spec.rb`
- Modify: `app/services/platform/features.rb`, `app/controllers/health_units_controller.rb` (`assign_caps`: `caps_modality`, `service_classification`; `unit_json` com as duas), `config/routes.rb`, `config/initializers/domain_events.rb` + bindings spec (`psychosocial.settings_changed`, `to: []`), `spec/services/platform/features_spec.rb`

**Interfaces:**
- Produces: `Psychosocial::Gate.usable?(city)` (`KEY = "psychosocial_care"`); `PsychosocialGate#require_psychosocial!`; `Psychosocial::BaseController` (no padrão do `Tfd::BaseController`: `render_failure`, `render_refusal`, `forbid`, `not_found`, `uuid?`, `body`, `body_param`, `query_only!`; `FAILURE_STATUS` com `not_on_caps_team`/`missing_role`/`lead_required`/`not_author` → 403, `not_found` → 404, `enrollment_open`/`bed_taken`/`already_staying`/`already_exported`/`bed_label_taken` → 409; o resto 422); `#staff_unit!(health_unit_id) -> HealthUnit|nil` (renderiza 422 `invalid_request` `path: "health_unit_id"` sem uuid/CAPS ativo, 403 `missing_role`/`not_on_caps_team` sem lotação) e `#require_lead!`; `Psychosocial::Access.link(user, health_unit_id) -> ProfessionalLink|nil` (Desvio 2), `.staffed_unit_ids(user) -> Array<uuid>`, `.staff?(user)`, `.lead?(user)`, `.can_read?(user, record) -> Boolean` (record com `health_unit_id`); `Psychosocial::Trail.viewed!(patient:, user:, resource:)` (Desvio 13); `Psychosocial::Raas.config -> Hash` (Valor fixado 5), `.label(:origins, code)`, `.valid?(:destinations, code)`; `Psychosocial::Settings.current -> PsychosocialSetting`; `Psychosocial::Json.{patient_ref(patient, caps: true), pro(user, cbo_code), sigtap(code, instrument, competence), bed(bed), unit(unit)}` (formas do contrato §2; `patient_ref` fora do CAPS sem `cns_masked`); rotas `GET/PATCH /psychosocial/settings` (`municipal_admin`; PATCH com step-up; `{ night_limits, care_plan_review_days }`; chave desconhecida, faixa fora do Valor fixado 4 ou modalidade fora de `NIGHT_MODALITIES` → 422 `invalid_request` com `path`), `GET/POST/PATCH /psychosocial/beds[/:id]` (`municipal_admin`, escrita com step-up; `?health_unit_id=` obrigatório no GET; CAPS fora de `NIGHT_MODALITIES` → 422 `caps_without_beds`; rótulo repetido → 409 `bed_label_taken`).

- [ ] **Step 1: Write the failing tests** (trechos):

```ruby
RSpec.describe Psychosocial::Access do
  let!(:city) { psychosocial_city! }
  let(:caps) { caps! }
  let(:other) { caps!(modality: "caps_ad") }

  it "lotação = vínculo ativo no CAPS; o mais antigo dá o CBO" do
    user = caps_staff!(caps, cbo: "251510")
    link_professional!(user, caps, cbo: "223505")
    expect(described_class.link(user, caps.id).cbo_code).to eq("251510")
    expect(described_class.link(user, other.id)).to be_nil
    ProfessionalLink.active.where(health_unit_id: caps.id).find_each { |l| l.update!(ended_at: Time.current, ended_by_user: user) }
    expect(described_class.link(user, caps.id)).to be_nil
  end

  it "sem papel do CAPS não há lotação, mesmo com vínculo" do
    doctor = ubs_doctor!(caps)
    expect(described_class.link(doctor, caps.id)).to be_nil
  end
end

RSpec.describe "CAPS — administração", type: :request do
  let!(:city) { psychosocial_city! }
  before { sign_in_as(psychosocial_admin!) }

  it "cadastra CAPS com modalidade e classificação; modalidade fora do CAPS e classificação inválida recusadas" do
    json_post "/health_units", name: "CAPS AD III Centro", kind: "caps", caps_modality: "caps_ad_iii", service_classification: "001"
    expect(json_body).to include("kind" => "caps", "caps_modality" => "caps_ad_iii", "service_classification" => "001")
    json_post "/health_units", name: "UBS X", kind: "ubs", caps_modality: "caps_ii"
    expect(json_body["error"]).to eq("caps_modality_mismatch")
    json_post "/health_units", name: "CAPS Y", kind: "caps", caps_modality: "caps_ii", service_classification: "1A"
    expect(json_body["error"]).to eq("invalid_service_classification")
  end

  it "leito só em CAPS III, AD III e AD IV; limites validados; interruptor desligado → 403" do
    json_post "/psychosocial/beds", health_unit_id: caps!(modality: "caps_ii").id, label: "L1"
    expect(json_body["error"]).to eq("caps_without_beds")
    json_post "/psychosocial/beds", health_unit_id: caps!(modality: "caps_iii").id, label: "L1"
    expect(response).to have_http_status(:created)
    json_patch "/psychosocial/settings", night_limits: { caps_ii: { total: 3, window_days: 30 } }
    expect(json_body["path"]).to eq("night_limits.caps_ii")
    expect { json_patch "/psychosocial/settings", care_plan_review_days: 60 }
      .to change { DomainEvent.where(name: "psychosocial.settings_changed").count }.by(1)
    Platform::Features.set!(city: city, key: "psychosocial_care", enabled: false, maintainer: ledi_maintainer!)
    get "/psychosocial/settings"
    expect(json_body).to eq("error" => "feature_disabled", "feature" => "psychosocial_care")
  end
end
```

`psychosocial_raas_spec.rb`: o YAML carrega; `destinations["00"] == "Permanência"`; `layout_version` é nulo; `night_sigtap_code` e `family_care_codes` com 10 dígitos; `review: true`. `features_spec.rb`: o catálogo termina em `psychosocial_care`, com `requires: %w[feature:clinical_record]`.

- [ ] **Step 2:** `bin/test api -w mod31 spec/services/psychosocial/access_spec.rb spec/requests/psychosocial/admin_spec.rb spec/config/psychosocial_raas_spec.rb spec/services/platform/features_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.** Entrada do Valor fixado 1; rotas `scope "/psychosocial", module: :psychosocial, as: :psychosocial`. `HealthUnitsController`: `caps_modality` só com `kind: "caps"` (senão `caps_modality_mismatch`), CAPS sem modalidade → `caps_modality_mismatch`; `service_classification` fora de `/\A\d{3}\z/` → `invalid_service_classification`; trocar o kind de `caps` para outro limpa as duas. `SaveSettings.call(attrs:, by:)` → `Result.ok(settings:)`; `SaveBed.call(attrs:, by:, bed: nil)`.
- [ ] **Step 4:** mesmos specs + `spec/requests/health_units_spec.rb` → PASS.
- [ ] **Step 5: Commit** `feat: add psychosocial care switch, caps units, team scope and beds`.

---

### Task 4: Encaminhamento, acolhimento e matrícula

**Files:**
- Create: `app/commands/psychosocial/{create_referral,record_reception,create_enrollment,update_enrollment,close_enrollment}.rb`, `app/services/psychosocial/monthly_records.rb`, `app/controllers/psychosocial/{referrals,receptions,enrollments}_controller.rb`, `spec/requests/psychosocial/{referrals,receptions,enrollments}_spec.rb`
- Modify: `app/services/psychosocial/json.rb`, `config/routes.rb`, `config/initializers/domain_events.rb` + bindings spec (`psychosocial.referral_created`, `.reception_recorded`, `.enrolled`, `.enrollment_closed`, `to: []`)

**Interfaces:**
- Consumes: Tasks 2–3; `Psychosocial::RecordAction` da Task 7 (o acolhimento grava a ação por ela — esta task cria a recepção e a matrícula; a ação do acolhimento entra na Task 7, que estende `RecordReception`).
- Produces: `Psychosocial::CreateReferral.call(attrs:, by:)` (Desvio: o encaminhador é `health_professional` com vínculo ativo numa unidade **não** CAPS — o mais antigo é `from_health_unit_id`; sem vínculo → 403 `missing_role`; precisa poder abrir o paciente, `ClinicalRecord::Access.call(user:, patient:)` permitido, senão 403 `out_of_context`; `reason` 10..4000; CAPS de destino ativo, senão `invalid_request` `path: "health_unit_id"`); `Psychosocial::CreateEnrollment.call(attrs:, by:, reception: nil)` (regras do contrato §4; `origin` em `origins`; `cid_main` CID-10 válido na release ativa — `ClinicalTerms.cid10_code`, inválido → `invalid_request` `path: "cid_main"`; até 4 secundários; `substances` não vazio, `"99"` sozinho; `esf_cnes` 7 dígitos se `esf_covered`; `reference_user_id` lotado no CAPS, senão `path: "reference_user_id"`; `started_on` não futuro; aberta → 409 `enrollment_open`; cria a RAAS do mês do início); `Psychosocial::UpdateEnrollment.call(enrollment:, attrs:, by:)` (`origin`/`started_on` só sem RAAS `exported` — senão `invalid_request` `path` = a chave); `Psychosocial::CloseEnrollment.call(enrollment:, destination:, closed_on:, by:)` (Desvio 6; `"00"` → `destination_is_permanence`; futuro → `invalid_request` `path: "closed_on"`; já fechada → 409 `already_closed`); `Psychosocial::MonthlyRecords.ensure!(enrollment, month) -> PsychosocialMonthlyRecord` (Desvio 6; mês com fechamento ativo → devolve o existente sem criar); `GET /psychosocial/prefill?patient_id=&health_unit_id=` → `{ esf_covered, esf_cnes }` pelo `TerritoryRegistry::Teams.for_patients` (equipe `kind == "70"` ativa → CNES da unidade dela; senão `{ esf_covered: false, esf_cnes: null }`; D4); rotas e formas do contrato §4: `POST /psychosocial/referrals`, `GET /psychosocial/referrals?health_unit_id&status=pending` (`{ id, patient: <patient-ref>, priority, status, from_unit: { id, name }, created_at }`, sem `reason`), `GET /psychosocial/referrals/:id` (com `reason`; trilha `resource: "referral"`), `POST /psychosocial/receptions`, `POST/GET/PATCH /psychosocial/enrollments[/:id]` (lista: `{ id, patient, started_on, reference: <pro>, next_visit_at, status, care_plan_review_on }`, busca `q` por nome só em matrículas do CAPS, sem trilha; detalhe com CIDs, substâncias, origem e ESF, trilha `resource: "enrollment"`), `POST /psychosocial/enrollments/:id/close`.

- [ ] **Step 1: Write the failing tests** (trecho de `enrollments_spec.rb`):

```ruby
RSpec.describe "CAPS — matrícula", type: :request do
  let!(:city) { psychosocial_city! }
  let(:caps) { caps! }
  let(:staff) { caps_staff!(caps) }
  let(:patient) { caps_patient! }
  def body(**over)
    { health_unit_id: caps.id, patient_id: patient.id, started_on: Date.current.iso8601, origin: "02", cid_main: "F20.0",
      cid_secondary: [ "F10.2" ], substances: [ "01" ], esf_covered: false, reference_user_id: staff.id }.merge(over)
  end

  before { sign_in_as(staff) }

  it "matricula, cria a RAAS do mês e recusa a segunda aberta no mesmo CAPS" do
    expect { json_post "/psychosocial/enrollments", body }.to change(PsychosocialMonthlyRecord, :count).by(1)
    expect(json_body).to include("status" => "open", "cid_main" => "F200")
    json_post "/psychosocial/enrollments", body
    expect([ response.status, json_body["error"] ]).to eq([ 409, "enrollment_open" ])
  end

  it "recusas todas de uma vez; ESF exige o CNES da USF" do
    json_post "/psychosocial/enrollments", body(origin: "77", cid_main: "XYZ", substances: [ "99", "01" ], esf_covered: true)
    expect(json_body["problems"].map { |p| p["path"] }).to contain_exactly("origin", "cid_main", "substances", "esf_cnes")
  end

  it "lista sem CID e sem trilha; detalhe com CID e com trilha; outro CAPS → 403 e 404" do
    json_post "/psychosocial/enrollments", body
    id = json_body["id"]
    expect { get "/psychosocial/enrollments", params: { health_unit_id: caps.id } }
      .not_to(change { DomainEvent.where(name: "clinical_record.viewed").count })
    expect(response.body).not_to include("F200")
    expect { get "/psychosocial/enrollments/#{id}" }.to change { DomainEvent.where(name: "clinical_record.viewed").count }.by(1)
    expect(DomainEvent.where(name: "clinical_record.viewed").last.payload).to include("access" => "psychosocial", "resource" => "enrollment")
    sign_in_as(caps_staff!(caps!(modality: "caps_ad")))
    get "/psychosocial/enrollments", params: { health_unit_id: caps.id }
    expect(json_body["error"]).to eq("not_on_caps_team")
    get "/psychosocial/enrollments/#{id}"
    expect(response).to have_http_status(:not_found)
  end

  it "encerrar: 00 recusado, data futura recusada; o destino vai à RAAS do mês do encerramento" do
    json_post "/psychosocial/enrollments", body
    id = json_body["id"]
    json_post "/psychosocial/enrollments/#{id}/close", destination: "00", closed_on: Date.current.iso8601
    expect(json_body["error"]).to eq("destination_is_permanence")
    json_post "/psychosocial/enrollments/#{id}/close", destination: "01", closed_on: Date.current.iso8601
    record = PsychosocialMonthlyRecord.find_by!(psychosocial_enrollment_id: id, month: Date.current.beginning_of_month)
    expect([ record.destination, record.concluded_on ]).to eq([ "01", Date.current ])
  end
end
```

`referrals_spec.rb`: o médico da UBS em contexto (atendimento `in_care` chamado por ele) encaminha; sem contexto → 403 `out_of_context`; a equipe do CAPS lista sem `reason` (o texto não aparece no corpo) e o detalhe traz `reason` com trilha; o evento `psychosocial.referral_created` só tem `referral_id`. `receptions_spec.rb`: `outcome: "enrolled"` sem bloco `enrollment` → `invalid_request` `path: "enrollment"`; com o bloco, cria recepção e matrícula na mesma transação (falha na matrícula desfaz a recepção); `referral_id` de outro paciente → `invalid_request` `path: "referral_id"`; o encaminhamento usado passa a `received`; `at` futuro → `invalid_request`.

- [ ] **Step 2:** `bin/test api -w mod31 spec/requests/psychosocial/referrals_spec.rb spec/requests/psychosocial/receptions_spec.rb spec/requests/psychosocial/enrollments_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.** Comandos `call(...) -> Result`; a recepção trava o encaminhamento (`FOR UPDATE`) e confere `pending`; `RecordReception` nesta task grava a recepção e, com `enrolled`, chama `CreateEnrollment` (a ação do acolhimento entra na Task 7). `RecordReception` já valida o `sigtap_code` contra a SIGTAP da competência de `at` (instrumento `09` ou `02`; senão `instrument_mismatch`).
- [ ] **Step 4:** mesmos specs → PASS.
- [ ] **Step 5: Commit** `feat: add caps referrals, receptions and enrollments with monthly raas records`.

---

### Task 5: PTS com versões e revisão vencida

**Files:**
- Create: `app/commands/psychosocial/save_care_plan.rb`, `app/controllers/psychosocial/care_plans_controller.rb`, `app/jobs/psychosocial/care_plan_review_job.rb`, `spec/requests/psychosocial/care_plans_spec.rb`, `spec/jobs/psychosocial/care_plan_review_job_spec.rb`
- Modify: `app/services/psychosocial/json.rb`, `config/routes.rb`, `config/recurring.yml` (`psychosocial_care_plan_review`: `every day at 7am America/Sao_Paulo`, fila `housekeeping`), `config/initializers/domain_events.rb` + bindings spec (`psychosocial.care_plan_saved`, `psychosocial.care_plan_review_due`, `to: []`)

**Interfaces:**
- Produces: `Psychosocial::SaveCarePlan.call(enrollment:, attrs:, by:)` (matrícula aberta, senão 409 `already_closed`; `needs` 1..4000; `goals` 1..20 itens `{ text 1..500, responsible_user_id (lotado no CAPS), due_on? }`; `intensity` em `INTENSITIES`; `review_on` > hoje e ≤ hoje + 365; versão = máxima + 1 sob trava da matrícula); `POST /psychosocial/enrollments/:id/care_plans`, `GET /psychosocial/enrollments/:id/care_plans` (`{ items: [ { id, version, needs, goals, intensity, review_on, created_by: <pro>, created_at } ] }` da mais nova para a mais antiga; trilha `resource: "care_plans"`); `GET /psychosocial/care_plans/due?health_unit_id=` (D9: matrículas abertas cuja última versão tem `review_on < hoje` ou que não têm PTS e começaram há mais de `care_plan_review_days`; para `psychosocial_lead` todas do CAPS, para os demais só as em que é técnico de referência; `{ items: [ { enrollment_id, patient: <patient-ref>, review_on|null } ] }`, sem trilha); `Psychosocial::CarePlanReviewJob` (EachCityJob, só com o portão utilizável; publica `psychosocial.care_plan_review_due` `care_plan_id` uma vez por versão vencida — `psychosocial_care_plan_alerts` com `insert_all ... unique_by`).

- [ ] **Step 1: Write the failing tests:** versão 1 e 2 imutáveis, a 2 com `version: 2`; responsável fora da equipe → `path: "goals[0].responsible_user_id"`; `review_on` no passado → `path: "review_on"`; matrícula encerrada → 409; o GET grava uma trilha; `due` mostra o vencido ao técnico de referência e à coordenação, e não a outro técnico; o job, rodado duas vezes no mesmo dia, publica um evento só por versão vencida e nenhum para a revisão em dia.
- [ ] **Step 2:** `bin/test api -w mod31 spec/requests/psychosocial/care_plans_spec.rb spec/jobs/psychosocial/care_plan_review_job_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** mesmos specs → PASS.
- [ ] **Step 5: Commit** `feat: add versioned care plans with overdue review alerts`.

---

### Task 6: Grade de grupos e "Hoje no CAPS"

**Files:**
- Create: `app/commands/psychosocial/save_group.rb`, `app/services/psychosocial/today.rb`, `app/controllers/psychosocial/{groups,today}_controller.rb`, `spec/requests/psychosocial/{groups,today}_spec.rb`
- Modify: `app/services/psychosocial/json.rb`, `config/routes.rb`

**Interfaces:**
- Produces: `Psychosocial::SaveGroup.call(attrs:, by:, group: nil)` (contrato §4; `responsible_user_id` lotado; `sigtap_code` com instrumento `09` ou `01` na SIGTAP da competência de hoje, senão `instrument_mismatch`; `starts_at`/`ends_at` `"HH:MM"`); `GET/POST/PATCH /psychosocial/groups[/:id]` (lista `?health_unit_id` com `?active=true`); `Psychosocial::Today.call(unit:, date:) -> { date, groups: [ { id, name, kind, starts_at, ends_at, responsible: <pro>, capacity } ], visits: [ { enrollment_id, patient: <patient-ref>, next_visit_at, reference: <pro> } ] }` (grupos ativos com `weekday == date.cwday`; matrículas abertas do CAPS com `next_visit_at` na data local, ordenadas pela hora); `GET /psychosocial/today?health_unit_id&date=` (`date` padrão hoje; inválida → `invalid_request` `path: "date"`; nada clínico; sem trilha).

- [ ] **Step 1: Write the failing tests:** grade com fim antes do início → `path: "ends_at"`; grupo de quarta aparece em `today` numa quarta e não numa quinta; `next_visit_at` às 23:30 locais entra no dia local (não no dia UTC seguinte); o corpo de `today` não traz `cid_main`, `substances` nem `notes`; equipe de outro CAPS → 403.
- [ ] **Step 2:** `bin/test api -w mod31 spec/requests/psychosocial/groups_spec.rb spec/requests/psychosocial/today_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** mesmos specs → PASS.
- [ ] **Step 5: Commit** `feat: add caps weekly groups and today view`.

---

### Task 7: Ações — crítica na hora, coletivas, cancelamento e o acolhimento como ação

**Files:**
- Create: `app/services/psychosocial/{instrument,action_rules}.rb`, `app/commands/psychosocial/{record_action,cancel_action}.rb`, `app/controllers/psychosocial/actions_controller.rb`, `spec/services/psychosocial/action_rules_spec.rb`, `spec/requests/psychosocial/actions_spec.rb`
- Modify: `app/commands/psychosocial/record_reception.rb` (grava a ação do acolhimento), `app/services/psychosocial/json.rb`, `config/routes.rb`, `config/initializers/domain_events.rb` + bindings spec (`psychosocial.action_recorded` — consumidor entra na Task 12 —, `psychosocial.action_cancelled`, `to: []`)

**Interfaces:**
- Consumes: `Billing::SigtapCache#{release,procedure,instruments,cbos,max_quantity,attributes}` (Task 1); `Psychosocial::Access.link` (Task 3); `MonthlyRecords.ensure!` (Task 4).
- Produces: `Psychosocial::Instrument.for(codes, enrolled:) -> String | :enrollment_required | :instrument_mismatch` (Desvio 3); `Psychosocial::ActionRules.problems(input, unit:, link:, now:, cache: Billing::SigtapCache.new) -> { instrument:, problems: Array<{ code:, participant_index: }> }` (Desvio 4; `input` = `{ sigtap_code, performed_at (Time), quantity, authorization_number, participants: [ { enrollment: PsychosocialEnrollment|nil, patient_id:, family_name: } ] }`); `Psychosocial::RecordAction.call(attrs:, unit:, by:, link:, origin: "manual", night_stay: nil, now: Time.current) -> Result.ok(action:) | fail(:rejected, details: { problems: })` (uma transação: ação, participantes na ordem, `MonthlyRecords.ensure!` para cada matriculado, evento `psychosocial.action_recorded` `action_id`); `Psychosocial::CancelAction.call(action:, reason:, by:)` (Desvio 5; motivo 10..500; já cancelada → 409 `already_cancelled`; na Task 12 ganha o cancelamento da linha BPA); rotas: `POST /psychosocial/actions` (corpo do contrato §4; 422 `{ error: "action_rejected", problems: [...] }`), `GET /psychosocial/actions?enrollment_id=` (lista sem `notes` e sem trilha: `{ id, sigtap: <sigtap>, instrument, performed_at, place, quantity, professional: <pro>, group_id, participants_count, cancelled }`), `GET /psychosocial/actions/:id` (com `notes`, participantes `{ enrollment_id, patient: <patient-ref> }` ou `{ family_name }`; trilha `resource: "action"` para cada paciente participante), `POST /psychosocial/actions/:id/cancel`. `RecordReception` passa a criar a ação do acolhimento (`origin: "reception"`, participante = a matrícula criada ou `{ patient_id }`; o instrumento sai do Desvio 3 — sem matrícula e só `09` → `enrollment_required`).

- [ ] **Step 1: Write the failing tests** (trecho de `actions_spec.rb`):

```ruby
RSpec.describe "CAPS — ações", type: :request do
  let!(:city) { psychosocial_city! }
  let(:caps) { caps! }
  let(:staff) { caps_staff!(caps, cbo: "251510") }
  let(:a) { enrollment!(caps, reference: staff) }
  let(:b) { enrollment!(caps, reference: staff) }

  def action(**over)
    { health_unit_id: caps.id, sigtap_code: "0301080216", performed_at: 1.hour.ago.iso8601, place: "caps", quantity: 1,
      notes: "Grupo de convivência; boa participação", participants: [ { enrollment_id: a.id }, { enrollment_id: b.id } ] }.merge(over)
  end

  before { sign_in_as(staff) }

  it "coletiva: uma ação, um participante por matrícula, cada RAAS do mês recebe a ação" do
    expect { json_post "/psychosocial/actions", action }.to change(PsychosocialAction, :count).by(1)
      .and change(PsychosocialActionParticipant, :count).by(2)
    expect(json_body).to include("instrument" => "raas_psi", "participants_count" => 2)
    expect(DomainEvent.where(name: "psychosocial.action_recorded").last.payload.keys).to contain_exactly("action_id")
  end

  it "Review Focus 1: participante de outro CAPS recusa tudo, com o índice" do
    foreign = enrollment!(caps!(modality: "caps_ad"))
    expect { json_post "/psychosocial/actions", action(participants: [ { enrollment_id: a.id }, { enrollment_id: b.id }, { enrollment_id: foreign.id } ]) }
      .not_to change(PsychosocialAction, :count)
    expect(json_body["problems"]).to eq([ { "code" => "enrollment_required", "participant_index" => 2 } ])
    expect(DomainEvent.where(name: "psychosocial.action_recorded")).to be_empty
  end

  it "crítica completa de uma vez: CBO, quantidade, data futura, autorização e familiar fora do atendimento familiar" do
    json_post "/psychosocial/actions", action(sigtap_code: "0301080291", quantity: 40, performed_at: 1.hour.from_now.iso8601,
                                              participants: [ { enrollment_id: a.id }, { family_name: "Mãe do usuário" } ])
    expect(json_body["problems"].map { |p| p["code"] })
      .to include("quantity_exceeded", "performed_in_future", "authorization_required", "family_only_for_family_care")
    sign_in_as(caps_staff!(caps, cbo: "515105")) # CBO fora da lista do código (o vínculo é só acréscimo; não se edita)
    json_post "/psychosocial/actions", action
    expect(json_body["problems"].map { |p| p["code"] }).to eq(%w[cbo_incompatible])
  end

  it "Review Focus 2: 23:30 locais de 30/11 são de novembro" do
    travel_to Time.zone.local(2026, 12, 1, 10, 0) do
      late = Time.zone.local(2026, 11, 30, 23, 30)
      PsychosocialMonthClosure.create!(health_unit: caps, month: Date.new(2026, 11, 1), closed_by_user: staff, closed_at: Time.current)
      json_post "/psychosocial/actions", action(performed_at: late.iso8601)
      expect(json_body["problems"].map { |p| p["code"] }).to eq(%w[month_closed])
      PsychosocialMonthClosure.last.update!(reopened_at: Time.current, reopened_by_user: staff, reopen_reason: "Correção de ação lançada")
      json_post "/psychosocial/actions", action(performed_at: late.iso8601)
      expect(PsychosocialAction.find(json_body["id"]).month).to eq(Date.new(2026, 11, 1))
    end
  end

  it "BPA-C aceita zero participantes; RAAS sem matrícula não" do
    json_post "/psychosocial/actions", action(sigtap_code: "0301080259", participants: [])
    expect(json_body["instrument"]).to eq("bpa_c")
    json_post "/psychosocial/actions", action(participants: [ { patient_id: caps_patient!.id } ])
    expect(json_body["problems"].map { |p| p["code"] }).to eq(%w[enrollment_required])
  end

  it "cancelar exige motivo; mês fechado só a coordenação; lista sem notes, detalhe com trilha" do
    json_post "/psychosocial/actions", action
    id = json_body["id"]
    get "/psychosocial/actions", params: { enrollment_id: a.id }
    expect(response.body).not_to include("boa participação")
    expect { get "/psychosocial/actions/#{id}" }.to change { DomainEvent.where(name: "clinical_record.viewed").count }.by(2)
    PsychosocialMonthClosure.create!(health_unit: caps, month: Date.current.beginning_of_month, closed_by_user: staff, closed_at: Time.current)
    json_post "/psychosocial/actions/#{id}/cancel", reason: "Lançada no grupo errado"
    expect([ response.status, json_body["error"] ]).to eq([ 403, "lead_required" ])
    sign_in_as(caps_staff!(caps, lead: true))
    json_post "/psychosocial/actions/#{id}/cancel", reason: "Lançada no grupo errado"
    expect(PsychosocialAction.find(id)).to be_cancelled
  end
end
```

`action_rules_spec.rb`: `Instrument.for(%w[02 09], enrolled: true) == "raas_psi"`, `(…, enrolled: false) == "bpa_i"`, `(%w[09], enrolled: false) == :enrollment_required`, `(%w[06], …) == :instrument_mismatch`; a release antiga sem `max_quantity`/atributos (Review Focus 4) não gera `quantity_exceeded` nem `authorization_required`; a crítica lê a SIGTAP da competência do mês da ação (release do mês anterior para ação do mês anterior). Spec do acolhimento: recepção `referred_elsewhere` com `0301080232` grava ação `bpa_i` com o paciente; com um código só `09` → `enrollment_required`.

- [ ] **Step 2:** `bin/test api -w mod31 spec/services/psychosocial/action_rules_spec.rb spec/requests/psychosocial/actions_spec.rb spec/requests/psychosocial/receptions_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.** `RecordAction` relê as matrículas com `FOR SHARE` dentro da transação e confere de novo "aberta e do CAPS"; `ActiveRecord::StatementInvalid` com `month closed` vira `month_closed` (corrida com o fechamento, nunca 500). Notas 1..4000 opcionais; `place` em `PLACES`; `group_id` do mesmo CAPS.
- [ ] **Step 4:** mesmos specs → PASS.
- [ ] **Step 5: Commit** `feat: record immutable caps actions with immediate sigtap critique`.

---

### Task 8: Acolhimento noturno — leitos, permanências, limite e reavaliação

**Files:**
- Create: `app/services/psychosocial/night_limit.rb`, `app/commands/psychosocial/{start_night_stay,close_night_stay}.rb`, `app/controllers/psychosocial/night_stays_controller.rb`, `spec/services/psychosocial/night_limit_spec.rb`, `spec/requests/psychosocial/night_stays_spec.rb`
- Modify: `app/controllers/psychosocial/beds_controller.rb` (`board`), `app/services/psychosocial/json.rb`, `config/routes.rb`, `config/initializers/domain_events.rb` + bindings spec (`psychosocial.night_stay_started`, `.night_stay_closed`, `.night_limit_overridden`, `to: []`)

**Interfaces:**
- Consumes: `RecordAction` (Task 7), `Psychosocial::Settings` (Task 3).
- Produces: `Psychosocial::NightLimit.nights(stay_or_range) -> Array<Date>` (Desvio 1); `.check(patient_id:, modality:, new_nights:, settings:, except_stay_id: nil) -> nil | { used:, limit:, rule: "total"|"consecutive"|"alternate" }` (janela de `window_days` terminando na última noite nova; noites do paciente em qualquer CAPS da cidade; CAPS III: passa se a maior sequência > `consecutive` ou o total > `alternate`; AD III/IV: total > `total`); `Psychosocial::StartNightStay.call(attrs:, by:, link:)` (leito ativo de CAPS de `NIGHT_MODALITIES`; matrícula aberta do mesmo CAPS; `expected_end_at` > `started_at` e ≤ 30 dias; `started_at` não futuro mais que 12 h; limite excedido → 422 `night_limit_reached` com `{ used, limit }`, salvo `psychosocial_lead` com `reassessment: { reason 10..500 }` — grava `reassessment_reason`/`reassessed_by_user_id` e publica `psychosocial.night_limit_overridden`; constraint de leito → 409 `bed_taken`; índice do paciente → 409 `already_staying`; Ruby confere antes e a constraint é a rede contra a corrida); `Psychosocial::CloseNightStay.call(stay:, ended_at:, reason:, by:, link:, reassessment: nil)` (`ended_at` > `started_at` e não futuro; se as noites reais passarem do limite e não houve reavaliação, a mesma regra do início; uma ação RAAS por noite, `origin: "night_stay"`, `sigtap_code` = `night_sigtap_code`, `performed_at` = 20:00 local da noite, profissional = quem fecha, numa transação com o fechamento — noite num mês fechado → 422 `month_closed` e nada fecha); `GET /psychosocial/beds/board?health_unit_id&date` → `{ items: [ { bed: <bed>, stay: { id, patient: <patient-ref>, started_at, expected_end_at } | null, nights_used: Integer|null, limit: Integer|null } ] }` (trilha só ao abrir o detalhe `GET /psychosocial/night_stays/:id`, `resource: "night_stay"`); `POST /psychosocial/night_stays`, `POST /psychosocial/night_stays/:id/close`.

- [ ] **Step 1: Write the failing tests** (trecho):

```ruby
RSpec.describe Psychosocial::NightLimit do
  let!(:city) { psychosocial_city! }

  it "AD III: 14 noites em 30; a 15ª passa do limite" do
    patient = caps_patient!
    nights = (0...14).map { |i| Date.new(2026, 11, 1) + i }
    allow(described_class).to receive(:existing_nights).and_return(nights)
    expect(described_class.check(patient_id: patient.id, modality: "caps_ad_iii", new_nights: [ Date.new(2026, 11, 20) ],
                                 settings: Psychosocial::Settings.current)).to eq(used: 15, limit: 14, rule: "total")
  end

  it "CAPS III: 7 corridas ou 10 intercaladas" do
    run = (1..7).map { |d| Date.new(2026, 11, d) }
    expect(described_class.check(patient_id: caps_patient!.id, modality: "caps_iii", new_nights: run + [ Date.new(2026, 11, 8) ],
                                 settings: Psychosocial::Settings.current)).to include(rule: "consecutive", limit: 7)
  end
end
```

`night_stays_spec.rb`: entrada no leito ocupado → 409 `bed_taken`; paciente já no leito L1 entrando no L2 → 409 `already_staying`; a 15ª noite no AD III → 422 `night_limit_reached` `{ used: 15, limit: 14 }` para a técnica e aceita para a coordenação com `reassessment.reason` (evento `night_limit_overridden` só com `night_stay_id`); fechar uma permanência de 3 noites cria 3 ações `0301080020` com quantidade 1 e `month` de cada noite; o mapa traz o ocupante sem CID; CAPS II → `caps_without_beds`.

- [ ] **Step 2:** `bin/test api -w mod31 spec/services/psychosocial/night_limit_spec.rb spec/requests/psychosocial/night_stays_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.** `existing_nights(patient_id, window, except_stay_id)` lê `psychosocial_night_stays` do paciente que tocam a janela (abertas com `expected_end_at`).
- [ ] **Step 4:** mesmos specs → PASS.
- [ ] **Step 5: Commit** `feat: add caps night reception with exclusive beds and night limits`.

---

### Task 9: RAAS do mês — continuidade, situação de rua, crítica, fechamento e reabertura

**Files:**
- Create: `app/jobs/psychosocial/open_month_job.rb`, `app/services/psychosocial/{month_problems,cnes_check}.rb`, `app/commands/psychosocial/{update_monthly_record,close_month,reopen_month}.rb`, `app/controllers/psychosocial/{monthly_records,months}_controller.rb`, `spec/jobs/psychosocial/open_month_job_spec.rb`, `spec/services/psychosocial/month_problems_spec.rb`, `spec/requests/psychosocial/months_spec.rb`
- Modify: `config/recurring.yml` (`psychosocial_open_month`: `every day at 1am America/Sao_Paulo`, `housekeeping`), `app/services/psychosocial/json.rb`, `config/routes.rb`, `config/initializers/domain_events.rb` + bindings spec (`psychosocial.month_closed`, `.month_reopened`, `to: []`)

**Interfaces:**
- Consumes: `ActionRules` (Task 7) para `action_out_of_rule`; `Billing::PatientContact.for`, `Billing::Staff.cns`.
- Produces: `Psychosocial::OpenMonthJob` (EachCityJob; portão utilizável; `insert_all` de `(enrollment, mês corrente)` para toda matrícula aberta, `unique_by: :index_psychosocial_monthly_records_on_enrollment_and_month`; log só com a contagem); `Psychosocial::CnesCheck.bonded?(cns:, cbo:, cnes:) -> Boolean` (último `CnesSnapshot` do `CityProfile.current.ibge_code`; `CnesProfessionalBond` com `cnes` e `cbo_code` iguais e `cns` decifrado igual; sem retrato → false); `Psychosocial::MonthProblems.for(health_unit:, month:) -> { counts: { records:, open:, closed:, exported:, with_problems: }, records: [ { id, enrollment_id, patient: <patient-ref>, status, homeless, problems: [String] } ] }` (Desvio 7; registro sem ação não cancelada no mês → `no_actions`; ação cuja crítica de hoje devolve problema → `action_out_of_rule`; profissional de ação sem CNS → `professional_cns_missing`; fora do CNES → `professional_not_in_cnes`; `night_stay_open` — D5); `PATCH /psychosocial/monthly_records/:id { homeless: Boolean }` (só `open`, senão 409 `month_closed`); `GET /psychosocial/months/:month?health_unit_id` (`month` `"YYYY-MM"`, senão `invalid_request` `path: "month"`; lead e staff); `Psychosocial::CloseMonth.call(unit:, month:, by:)` (lead, step-up; problemas → 422 `month_has_problems` com `problems` = registros com problema; trava a unidade e os registros `FOR UPDATE`; grava a foto; fechamento ativo já existe → 409 `already_closed`); `Psychosocial::ReopenMonth.call(unit:, month:, reason:, by:)` (lead, step-up; motivo 10..500; exportado → 409 `already_exported`); `POST /psychosocial/months/:month/{close,reopen}`.

- [ ] **Step 1: Write the failing tests** (trecho de `months_spec.rb`):

```ruby
RSpec.describe "CAPS — RAAS do mês", type: :request do
  let!(:city) { psychosocial_city! }
  let(:caps) { caps!(modality: "caps_ad_iii") }
  let(:lead) { caps_staff!(caps, lead: true) }
  let(:month) { Date.current.beginning_of_month }

  before { sign_in_as(lead) }

  it "pendências lidas, nunca gravadas; fechar com pendência recusa" do
    e = enrollment!(caps, patient: caps_patient!(cns: false))
    get "/psychosocial/months/#{month.strftime('%Y-%m')}", params: { health_unit_id: caps.id }
    expect(json_body["records"].first["problems"]).to include("patient_cns_missing", "homeless_missing", "no_actions")
    json_post "/psychosocial/months/#{month.strftime('%Y-%m')}/close", health_unit_id: caps.id
    expect(json_body["error"]).to eq("month_has_problems")
    expect(PsychosocialMonthClosure.count).to eq(0)
    expect(e.reload).to be_open
  end

  it "Review Focus 3: permanência aberta com noite no mês bloqueia o fechamento" do
    travel_to Time.zone.local(2026, 12, 1, 9, 0) do
      e = enrollment!(caps, started_on: Date.new(2026, 11, 3))
      complete_month!(e, Date.new(2026, 11, 1))
      PsychosocialNightStay.create!(psychosocial_bed: bed!(caps), health_unit: caps, psychosocial_enrollment: e, patient_id: e.patient_id,
                                    started_at: Time.zone.local(2026, 11, 29, 19), expected_end_at: Time.zone.local(2026, 12, 3, 8), started_by_user: lead)
      json_post "/psychosocial/months/2026-11/close", health_unit_id: caps.id
      expect(json_body["problems"].first["problems"]).to eq(%w[night_stay_open])
    end
  end

  it "fecha sem pendência: foto, evento, ações travadas; reabre com motivo antes da remessa" do
    e = enrollment!(caps)
    complete_month!(e, month)
    json_post "/psychosocial/months/#{month.strftime('%Y-%m')}/close", health_unit_id: caps.id
    expect(PsychosocialMonthlyRecord.find_by!(psychosocial_enrollment_id: e.id, month: month)).to have_attributes(status: "closed")
    expect(DomainEvent.where(name: "psychosocial.month_closed").last.payload).to eq("health_unit_id" => caps.id, "month" => month.strftime("%Y-%m"))
    json_post "/psychosocial/months/#{month.strftime('%Y-%m')}/reopen", health_unit_id: caps.id, reason: "curto"
    expect(json_body["path"]).to eq("reason")
    json_post "/psychosocial/months/#{month.strftime('%Y-%m')}/reopen", health_unit_id: caps.id, reason: "Ação lançada com CBO errado"
    expect(PsychosocialMonthlyRecord.find_by!(psychosocial_enrollment_id: e.id, month: month).status).to eq("open")
  end

  it "fechar exige a coordenação e step-up" do
    sign_in_as(caps_staff!(caps))
    json_post "/psychosocial/months/#{month.strftime('%Y-%m')}/close", health_unit_id: caps.id
    expect(json_body["error"]).to eq("lead_required")
    sign_in_as(lead)
    lead.update!(mfa_verified_at: 10.minutes.ago)
    json_post "/psychosocial/months/#{month.strftime('%Y-%m')}/close", health_unit_id: caps.id
    expect(json_body["error"]).to eq("mfa_required")
  end
end
```

`complete_month!(enrollment, month)` (helper novo em `psychosocial_helpers.rb`): `homeless: false`, uma ação `0301080208` do profissional com vínculo CNES no retrato (`cnes_snapshot!` com o bond do CNS dele), paciente com CNS e telefone. `open_month_job_spec.rb`: no dia 1º cria o registro do mês para a matrícula aberta e nenhum para a encerrada; duas execuções no mesmo dia → um registro; portão desligado → nada. `month_problems_spec.rb`: profissional sem bond no retrato → `professional_not_in_cnes`; sem retrato → idem; ação cancelada não conta como ação.

- [ ] **Step 2:** `bin/test api -w mod31 spec/jobs/psychosocial/open_month_job_spec.rb spec/services/psychosocial/month_problems_spec.rb spec/requests/psychosocial/months_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** mesmos specs + `spec/requests/psychosocial` → PASS.
- [ ] **Step 5: Commit** `feat: add monthly raas records with continuity job, critique, closing and reopening`.

---

### Task 10: Remessa RAAS atrás de `layout_version` e ficha espelho

**Files:**
- Create: `app/services/psychosocial/{raas_file,mirror_sheet_pdf}.rb`, `app/commands/psychosocial/export_raas.rb`, `app/controllers/psychosocial/raas_exports_controller.rb`, `spec/services/psychosocial/raas_file_spec.rb`, `spec/requests/psychosocial/raas_exports_spec.rb`, `spec/services/psychosocial/mirror_sheet_pdf_spec.rb`
- Modify: `app/controllers/psychosocial/months_controller.rb` (`sheets`), `config/routes.rb`, `config/initializers/domain_events.rb` + bindings spec (`psychosocial.raas_exported`, `.raas_downloaded`, `to: []`)

**Interfaces:**
- Produces: `Psychosocial::RaasFile` — `Layout = Data.define(:version, :writer)` (writer: `->(header, records) { String }`), `LAYOUTS = {}.freeze`, `.available?(version) -> Boolean`, `.build(version:, header:, records:) -> String` (`header` = `{ cnes, competence, month, unit_name }`; `records` = fotos dos registros com as ações não canceladas do mês; `KeyError` → nunca chamado sem `available?`); `Psychosocial::ExportRaas.call(unit:, month:, by:) -> Result.ok(export:)` (Desvio 8 e 9; só mês fechado — senão 409 `month_open`; já exportado → 409 `already_exported`; sem layout → 422 `raas_layout_unavailable`; grava `psychosocial_raas_exports` com `file_data` em Base64 cifrado, `sha256`, `records_count`, registros → `exported` com o id; evento `psychosocial.raas_exported` `raas_export_id`); `POST /psychosocial/raas_exports { health_unit_id, month }` e `GET /psychosocial/raas_exports/:id/file` (`application/octet-stream`, nome `RAAS-<cnes>-<competência>.txt`; evento `psychosocial.raas_downloaded` e trilha `clinical_record.viewed` por paciente com `resource: "raas_file"`) — **quem:** `psychosocial_lead` lotado ou `billing_clerk` (sem lotação; CAPS ativo), com step-up; `Psychosocial::MirrorSheetPdf.call(records:, preview:) -> String` (Desvio 10); `GET /psychosocial/months/:month/sheets.pdf?health_unit_id=&enrollment_id=` (mesmos papéis; trilha por paciente `resource: "raas_sheet"`).

- [ ] **Step 1: Write the failing tests:** sem layout → 422 `raas_layout_unavailable` e nada gravado; com `stub_const("Psychosocial::RaasFile::LAYOUTS", { "test-1" => layout_de_teste })` e `layout_version: "test-1"`: mês aberto → 409 `month_open`; fechado → 201, competência = a do `Billing::Periods.assign!` (com a competência do mês fechada no 22, a seguinte aberta), registros `exported`, reabrir → 409 `already_exported`, segunda remessa → 409; o download grava `raas_downloaded` e as trilhas; `health_professional` → 403; `billing_clerk` baixa sem lotação. `mirror_sheet_pdf_spec.rb`: o texto do PDF (por `PDF::Reader`, já usado nos specs do 19c) tem "Identificação", "Dados do atendimento" e "Ações" nessa ordem, "PRÉVIA" só com o mês aberto, o CID principal e as datas das ações, e não tem `notes`, PTS nem texto de matriciamento; `enrollment_id` de outro CAPS → 404.
- [ ] **Step 2:** `bin/test api -w mod31 spec/services/psychosocial/raas_file_spec.rb spec/requests/psychosocial/raas_exports_spec.rb spec/services/psychosocial/mirror_sheet_pdf_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.** PDF no padrão do `ClinicalDocuments::Pdf`/`Transport::ManifestPdf` (Prawn, fonte embutida do projeto); nenhum dado no nome do arquivo além de CNES e mês.
- [ ] **Step 4:** mesmos specs → PASS.
- [ ] **Step 5: Commit** `feat: add raas export behind the layout gate and the mirror sheet pdf`.

---

### Task 11: Matriciamento, caso compartilhado e o vínculo visto pela UBS

**Files:**
- Create: `app/commands/psychosocial/{record_matrix_support,add_matrix_case}.rb`, `app/services/psychosocial/link.rb`, `app/controllers/psychosocial/matrix_supports_controller.rb`, `app/controllers/patient_psychosocial_links_controller.rb`, `spec/requests/psychosocial/matrix_supports_spec.rb`, `spec/requests/patient_psychosocial_link_spec.rb`
- Modify: `config/routes.rb` (`get "/patients/:id/psychosocial_link"`), `config/initializers/domain_events.rb` + bindings spec (`psychosocial.matrix_support_recorded`, `to: []`)

**Interfaces:**
- Consumes: `RecordAction` (Task 7; a ação `bpa_c` nasce aqui com `origin: "matrix_support"`, zero participantes).
- Produces: `Psychosocial::RecordMatrixSupport.call(attrs:, unit:, by:, link:)` (contrato §7; `health_team_id` de equipe ativa de unidade **não** CAPS; `participants_user_ids` 1..20, todos lotados no CAPS; `topic` 1..200; `held_on` não futuro; `sigtap_code` com instrumento `01`, senão `instrument_mismatch`; numa transação: a ação e o matriciamento); `Psychosocial::AddMatrixCase.call(support:, patient_id:, shared_text:, by:)` (`shared_text` 10..4000; autor lotado no CAPS do matriciamento); `Psychosocial::Link.for(patient:, viewer:) -> Hash` (forma do contrato §7; matrícula aberta mais antiga quando houver mais de uma — `caps: { id, name, phone }` com o telefone da unidade; `reference: { name }`; `shared_cases` = casos do paciente cujo `health_team.health_unit_id` está entre as unidades de vínculo ativo do leitor); `GET /patients/:id/psychosocial_link` (portão `psychosocial_care`; `ClinicalRecord::Access.call(user:, patient:)` permitido, senão 403 com o motivo; trilha `clinical_record.viewed` `access: "psychosocial"` `resource: "link_shared_cases"` só com `shared_cases` não vazio); `POST /psychosocial/matrix_supports`, `GET /psychosocial/matrix_supports?health_unit_id` (sem `topic`), `POST /psychosocial/matrix_supports/:id/cases`.

- [ ] **Step 1: Write the failing tests** (trecho de `patient_psychosocial_link_spec.rb`):

```ruby
RSpec.describe "UBS — vínculo com o CAPS", type: :request do
  let!(:city) { psychosocial_city! }
  let(:caps) { caps! }
  let(:ubs) { HealthUnit.create!(name: "UBS Boa Vista", kind: "ubs") }
  let(:team) { HealthTeam.create!(health_unit: ubs, ine: "0000123456", kind: "70", name: "eSF Boa Vista") }
  let(:doctor) { ubs_doctor!(ubs) }
  let(:enrollment) { enrollment!(caps) }

  it "em contexto, a UBS vê só o vínculo e o caso da própria equipe, com trilha só quando há caso" do
    in_context!(doctor, ubs, enrollment.patient)
    sign_in_as(doctor)
    expect { get "/patients/#{enrollment.patient_id}/psychosocial_link" }
      .not_to(change { DomainEvent.where(name: "clinical_record.viewed").count })
    expect(json_body).to include("enrolled" => true, "shared_cases" => [])
    expect(json_body["caps"]).to include("id" => caps.id, "name" => caps.name)
    matrix_case!(caps, team, enrollment.patient, "Discutido manejo da insônia com a equipe")
    get "/patients/#{enrollment.patient_id}/psychosocial_link"
    expect(json_body["shared_cases"].first["shared_text"]).to eq("Discutido manejo da insônia com a equipe")
    expect(response.body).not_to include("F200")
    expect(json_body.keys).to contain_exactly("enrolled", "caps", "since", "reference", "shared_cases")
  end

  it "fora de contexto → 403; outra equipe não vê o caso; sem matrícula → enrolled false" do
    sign_in_as(doctor)
    get "/patients/#{enrollment.patient_id}/psychosocial_link"
    expect(json_body["error"]).to eq("out_of_context")
    other = HealthUnit.create!(name: "UBS Centro", kind: "ubs")
    other_doctor = ubs_doctor!(other)
    matrix_case!(caps, team, enrollment.patient, "Texto da equipe Boa Vista apenas")
    in_context!(other_doctor, other, enrollment.patient)
    sign_in_as(other_doctor)
    get "/patients/#{enrollment.patient_id}/psychosocial_link"
    expect(json_body["shared_cases"]).to eq([])
    in_context!(other_doctor, other, caps_patient!.tap { |p| @lone = p })
    get "/patients/#{@lone.id}/psychosocial_link"
    expect(json_body).to eq("enrolled" => false)
  end
end
```

`in_context!` e `matrix_case!` entram em `psychosocial_helpers.rb` (atendimento `in_care` chamado pelo médico, de par validado do paciente; matriciamento + caso pelos comandos). `matrix_supports_spec.rb`: equipe de unidade CAPS → `path: "health_team_id"`; participante não lotado → `path: "participants_user_ids[1]"`; gravado → uma ação `bpa_c` sem participantes; lista sem `topic`.

- [ ] **Step 2:** `bin/test api -w mod31 spec/requests/psychosocial/matrix_supports_spec.rb spec/requests/patient_psychosocial_link_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** mesmos specs → PASS.
- [ ] **Step 5: Commit** `feat: add matrix support with shared cases and the caps link for primary care`.

---

### Task 12: BPA pelo módulo 22 — fonte `psychosocial`, BPA-C sem paciente (transversal, §8)

**Files:**
- Create: `db/city_migrate/20261104900003_add_psychosocial_billing_source.rb`, `app/commands/billing/from_psychosocial.rb`, `app/jobs/billing/psychosocial_production_job.rb`, `spec/commands/billing/from_psychosocial_spec.rb`, `spec/services/billing/critic_patientless_spec.rb`
- Modify: `db/city_schema.rb`, `app/models/performed_procedure.rb` (`SOURCES` + `psychosocial`, `belongs_to :patient, optional: true`, `belongs_to :psychosocial_action, optional: true`), `app/commands/billing/record.rb` (`insert!` ganha `psychosocial_action: nil`; `patient:` pode ser nil), `app/services/billing/critic.rb`, `app/services/billing/lines.rb`, `app/services/billing/json.rb`, `app/services/billing/pending.rb`, `app/commands/psychosocial/cancel_action.rb`, `config/initializers/domain_events.rb` + bindings spec (`psychosocial.action_recorded` → `[ Billing::PsychosocialProductionJob ]`), `spec/models/transport_schema_spec.rb`

**Interfaces:**
- Produces: `performed_procedures.psychosocial_action_id` (FK, índice único parcial `idx_performed_procedures_psychosocial_action`); `ck_performed_procedures_domains` com `AddPsychosocialBillingSource::SOURCES = %w[attendance consultation lab_report manual tfd psychosocial]`; `ck_performed_procedures_source_ref` + `(source <> 'psychosocial' OR psychosocial_action_id IS NOT NULL)`; `performed_procedures.patient_id` NULL permitido com `ck_performed_procedures_patient` = `patient_id IS NOT NULL OR (source = 'psychosocial' AND psychosocial_action_id IS NOT NULL)` (D2; o contrato fixa `instrument = 'bpa_c'`); `Billing::Critic`: paciente nulo com instrumento `bpa_c` não gera motivo de paciente; com `bpa_i`/`not_bpa`, `missing_patient_id` (o arquivo nunca o leva); `Billing::Lines`: linha BPA-C sem paciente com `age = Billing::Lines::PATIENTLESS_AGE = 0` (gate do layout do BPA-C: idade da ação de equipe); `Billing::Json`/`Billing::Pending`: `patient: nil` quando não há paciente; `Billing::FromPsychosocial.call(action) -> PerformedProcedure|nil` (Desvio 11; nil para `raas_psi`, ação cancelada ou já levada); `Billing::PsychosocialProductionJob` (IdempotentConsumer, `handle(action_id:, **)`, `return unless Gate.enabled?(Current.city)`); `CancelAction` cancela a linha pelo `Billing::Cancel.call(procedure:, reason:, by:)` na mesma transação; `already_exported` do 22 → a ação não cancela (409 `already_exported`).

- [ ] **Step 1: Write the failing tests** (trecho de `from_psychosocial_spec.rb`):

```ruby
RSpec.describe Billing::FromPsychosocial do
  let!(:city) { psychosocial_city!(billing: true) }
  let(:caps) { caps! }
  let(:staff) { caps_staff!(caps, cbo: "251510") }

  it "BPA-C: uma linha por ação, sem paciente, com a quantidade da ação; nunca uma por participante" do
    action = record_action!(caps, staff, sigtap_code: "0301080313", quantity: 3, participants: [ { patient_id: caps_patient!.id }, { patient_id: caps_patient!.id } ])
    line = described_class.call(action)
    expect(line).to have_attributes(source: "psychosocial", patient_id: nil, cid10: nil, quantity: 3, instrument: "bpa_c",
                                    health_unit_id: caps.id, cbo_code: "251510", psychosocial_action_id: action.id)
    expect(described_class.call(action)).to be_nil
    expect(PerformedProcedure.where(psychosocial_action_id: action.id).count).to eq(1)
  end

  it "BPA-I leva o paciente; RAAS nunca vai" do
    patient = caps_patient!
    reception_action = record_action!(caps, staff, sigtap_code: "0301080232", participants: [ { patient_id: patient.id } ])
    expect(described_class.call(reception_action)).to have_attributes(instrument: "bpa_i", patient_id: patient.id)
    raas = record_action!(caps, staff, sigtap_code: "0301080208", participants: [ { enrollment_id: enrollment!(caps).id } ])
    expect(described_class.call(raas)).to be_nil
  end

  it "cancelar a ação cancela a linha; linha já exportada impede o cancelamento" do
    action = record_action!(caps, staff, sigtap_code: "0301080305", participants: [])
    line = described_class.call(action)
    Psychosocial::CancelAction.call(action: action, reason: "Matriciamento lançado em duplicidade", by: staff)
    expect(line.reload.status).to eq("cancelled")
  end

  it "o banco recusa linha sem paciente de outra fonte" do
    expect { raw_procedure!(unit: caps, patient: nil, professional: staff, source: "manual") }
      .to raise_error(ActiveRecord::StatementInvalid, /ck_performed_procedures_patient/)
  end
end
```

`critic_patientless_spec.rb` (Review Focus 5): linha BPA-C sem paciente é faturável; numa competência nova cuja SIGTAP tirou o `01` do código, `Billing::Recalculate.call` grava `instrument: "not_bpa"`, `billable: false`, motivos com `not_bpa_instrument`, sem exceção; `Billing::Close` da competência com essa linha conclui e o arquivo não a leva; com a linha `bpa_c`, `Billing::Lines.build` devolve um `CLine` com `age: 0`. O job: evento `psychosocial.action_recorded` com o portão do BPA desligado não grava; ligado, grava uma vez mesmo reprocessado.

- [ ] **Step 2:** `bin/test api -w mod31 spec/commands/billing/from_psychosocial_spec.rb spec/services/billing/critic_patientless_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.** `record_action!(unit, by, **attrs)` em `psychosocial_helpers.rb` (pelo `RecordAction`, data de hoje às 10:00 locais). Erro do banco no INSERT → `FromPsychosocial::InsertFailed` sem a causa (padrão do `FromTfd`).
- [ ] **Step 4:** mesmos specs + `spec/commands/billing spec/services/billing spec/requests/billing spec/jobs/billing spec/models/transport_schema_spec.rb` → PASS.
- [ ] **Step 5: Commit** `feat: bill caps team actions as patientless bpa-c and receptions as bpa-i`.

---

### Task 13: Receita do 19 a partir da ação do CAPS (transversal, §9)

**Files:**
- Create: `db/city_migrate/20261104900004_add_psychosocial_documents.rb`, `app/controllers/psychosocial/documents_controller.rb`, `spec/requests/psychosocial/documents_spec.rb`
- Modify: `db/city_schema.rb`, `app/models/clinical_document.rb`, `app/commands/clinical_documents/issue.rb`, `app/services/clinical_documents/{read,pdf,verification,json}.rb`, `app/services/signatures/canonical.rb`, `app/controllers/clinical_documents_controller.rb` (linha 119: `where(psychosocial_action_id: nil)`), `config/routes.rb`

**Interfaces:**
- Produces: `clinical_documents.psychosocial_action_id` (uuid, FK, índice); `attendance_id` e `citizen_id` NULL permitidos; CHECKs: `ck_clinical_documents_parent` = `num_nonnulls(attendance_id, psychosocial_action_id) = 1` (D8: mais estrito que o par do contrato §9, e o mesmo efeito); `ck_clinical_documents_citizen` = `(psychosocial_action_id IS NULL) = (citizen_id IS NOT NULL)` (D8); `ck_clinical_documents_consultation` = `consultation_id IS NOT NULL OR psychosocial_action_id IS NOT NULL OR kind = 'attendance_declaration'`; `ck_clinical_documents_paper_outside_consultation` = `consultation_id IS NOT NULL OR psychosocial_action_id IS NOT NULL OR issue_mode = 'paper'`; `ck_clinical_documents_cbo_required` = `(consultation_id IS NULL AND psychosocial_action_id IS NULL) OR cbo_code IS NOT NULL`; `ClinicalDocument#health_unit -> HealthUnit` (`attendance&.health_unit || psychosocial_action&.health_unit`), usado nos quatro leitores do PRÉ-REQUISITO; `ClinicalDocuments::Issue.call(..., psychosocial_action: nil)` (Desvio 12; `PSYCHOSOCIAL_KINDS = %w[prescription controlled_notification_record sick_note]`, outro tipo → `invalid_content` `field: "kind"`; interruptores do 19c/19d e assinatura como na consulta; sem `medications!`/`record_medication!`; evento `clinical_document.issued` com `psychosocial_action_id` no lugar de `consultation_id`; `replaced` compara `psychosocial_action_id`); `ClinicalDocuments::Read.document_grant`: documento com `psychosocial_action_id` → autora, ou `Psychosocial::Access.can_read?(user, action)` → `Grant(kind: :psychosocial)`, senão `deny(:not_found)`; `POST /psychosocial/actions/:id/documents` (corpo `{ kind, content, replaces_document_id? }` do 19c; respostas do `ClinicalDocumentsController#respond_created`) e `GET /psychosocial/actions/:id/documents` (lista da ação, trilha `resource: "documents"`); impressão, cancelamento e verificação pública pelas rotas que já existem.

- [ ] **Step 1: Write the failing tests** (trecho de `documents_spec.rb`):

```ruby
RSpec.describe "CAPS — receita", type: :request do
  let!(:city) { controlled_city!(mock: true, signature: true).tap { psychosocial_switch_on!(_1) } }
  let(:caps) { caps! }
  let(:psychiatrist) { caps_staff!(caps, cbo: "225133") }
  let(:enrollment) { enrollment!(caps, reference: psychiatrist) }
  let(:action) { record_action!(caps, psychiatrist, sigtap_code: "0301080208", participants: [ { enrollment_id: enrollment.id } ]) }

  before { sign_in_as(psychiatrist) }

  it "emite a receita controlada pela ação, sem atendimento e sem cidadão, fora da lista de medicamentos" do
    prescriber_contact!(psychiatrist)
    sncr_stock!(psychiatrist)
    expect { json_post "/psychosocial/actions/#{action.id}/documents", kind: "prescription", content: controlled_content }
      .not_to change(PatientMedication, :count)
    document = ClinicalDocument.find(json_body.dig("document", "id"))
    expect(document).to have_attributes(psychosocial_action_id: action.id, attendance_id: nil, citizen_id: nil, cbo_code: "225133")
    get "/v/#{document.verification_token}"
    expect(response).to have_http_status(:ok)
  end

  it "psicóloga e ação coletiva não emitem; exame só pela consulta" do
    psychologist = caps_staff!(caps, cbo: "251510")
    sign_in_as(psychologist)
    other = record_action!(caps, psychologist, sigtap_code: "0301080208", participants: [ { enrollment_id: enrollment.id } ])
    json_post "/psychosocial/actions/#{other.id}/documents", kind: "prescription", content: controlled_content
    expect(json_body["error"]).to eq("prescriber_required")
    sign_in_as(psychiatrist)
    json_post "/psychosocial/actions/#{action.id}/documents", kind: "exam_requisition", content: {}
    expect(json_body).to include("error" => "invalid_content")
  end

  it "médico da UBS em contexto não lê o documento do CAPS pelas rotas do 19" do
    json_post "/psychosocial/actions/#{action.id}/documents", kind: "sick_note", content: { "days" => 3, "cid10" => nil }
    ubs = HealthUnit.create!(name: "UBS Vila", kind: "ubs")
    doctor = ubs_doctor!(ubs)
    in_context!(doctor, ubs, enrollment.patient)
    sign_in_as(doctor)
    get "/clinical_documents/#{json_body.dig('document', 'id')}"
    expect(response).to have_http_status(:not_found)
  end
end
```

`psychosocial_switch_on!(city)` liga `psychosocial_care` sobre a cidade do 19d (helper novo). Mais: o spec existente do 19c/19d segue verde (`spec/requests/clinical_documents spec/commands/clinical_documents spec/services/signatures`); o canônico da assinatura de um documento do CAPS traz o CNES do CAPS.

- [ ] **Step 2:** `bin/test api -w mod31 spec/requests/psychosocial/documents_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.** Migração: `change_column_null :clinical_documents, :attendance_id, true` e `:citizen_id, true`; `add_reference ... :psychosocial_action`; `replace_check` × 3 + `add_check_constraint` × 2; o trigger do documento não muda (guarda a linha inteira).
- [ ] **Step 4:** mesmos specs + `spec/requests/clinical_documents spec/commands/clinical_documents spec/services/clinical_documents spec/services/signatures spec/requests/document_verifications_spec.rb` → PASS.
- [ ] **Step 5: Commit** `feat: issue clinical documents from caps actions without an attendance`.

---

### Task 14: Linha do tempo restrita, marcador do vínculo e fora do LEDI

**Files:**
- Create: `app/services/timeline/sources/{psychosocial_source,psychosocial_link_source}.rb`, `app/controllers/psychosocial/timelines_controller.rb`, `spec/services/timeline/sources/psychosocial_source_spec.rb`, `spec/requests/psychosocial/timeline_spec.rb`
- Modify: `app/services/timeline.rb` (`KINDS` + `psychosocial_link`), `app/services/timeline/query.rb` (`PROFESSIONAL_ONLY_KINDS = %w[dispensation psychosocial_link]`; `CAPS_ONLY_KINDS = %w[psychosocial]`; `SOURCES` com as duas), `app/services/timeline/cursor.rb` (aceita `Timeline::KINDS + Query::CAPS_ONLY_KINDS`), `app/controllers/pec_timelines_controller.rb` (`KINDS = (Timeline::KINDS - %w[transport psychosocial_link]).freeze`), `app/services/timeline/sources/{clinical_document_source,controlled_notification_source}.rb` e `app/services/timeline/citizen_documents.rb` (`where(psychosocial_action_id: nil)`), `app/services/timeline/sources/procedure_source.rb` e `app/services/care_lines/sources.rb` (`where.not(source: "psychosocial")`), `app/services/timeline/sources/dispensation_source.rb` (fora a dispensação de documento com `psychosocial_action_id`), `app/jobs/ledi/{consultation_ficha_job,screening_ficha_job}.rb` (`enqueue_for` volta sem enfileirar quando `attendance.health_unit.kind == "caps"`), `config/routes.rb`, `spec/services/timeline/query_spec.rb`

**Interfaces:**
- Produces: `Timeline::Sources::PsychosocialSource.entries(person:, before:, limit:, from:, to:)` (kind `psychosocial`; ações não canceladas com o paciente como participante — título = nome SIGTAP, `occurred_at` = `performed_at`; versões do PTS — "PTS versão N", `created_at`; permanências — "Acolhimento noturno", `started_at`; documentos com `psychosocial_action_id` — "Receita"/"Atestado", `issued_at`; `summary`, `citizen` e `document` nulos; `detail_ref: { type: "psychosocial_action"|"psychosocial_care_plan"|"psychosocial_night_stay"|"clinical_document", id }`); `Timeline::Sources::PsychosocialLinkSource.entries(...)` (kind `psychosocial_link`; uma entrada por matrícula, `occurred_at` = início, título "Acompanhamento no CAPS <nome da unidade>", `summary` "Desde dd/mm/aaaa" ou "dd/mm/aaaa a dd/mm/aaaa"; nada clínico; `detail_ref` nulo); `GET /psychosocial/enrollments/:id/timeline?before=&limit=` (só a equipe lotada no CAPS da matrícula; `Timeline::Query.call(person:, kinds: Query::CAPS_ONLY_KINDS + %w[psychosocial_link])`; trilha `resource: "timeline"`).

- [ ] **Step 1: Write the failing tests:** a fonte `psychosocial` traz ação, PTS, permanência e receita do paciente, sem `notes`; `Timeline::Query.citizen_kinds(verified: true, documents: true)` não inclui `psychosocial` nem `psychosocial_link`; `PecTimelinesController::KINDS` também não; o `GET /timelines` do médico da UBS em contexto traz o marcador "Acompanhamento no CAPS …" e nenhuma entrada `psychosocial`, nem `procedure` da linha BPA-I do acolhimento, nem `clinical_document` da receita do CAPS; o cursor de `GET /psychosocial/enrollments/:id/timeline` pagina; outro CAPS → 404; `query_spec.rb:75` passa a conferir `KINDS.last(2) == %w[transport psychosocial_link]`; `Ledi::ConsultationFichaJob.enqueue_for` de consulta num atendimento de unidade `caps` não cria `LediFicha`.
- [ ] **Step 2:** `bin/test api -w mod31 spec/services/timeline spec/requests/psychosocial/timeline_spec.rb spec/requests/timelines_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4:** mesmos specs + `spec/requests/citizen_api spec/requests/pec_timelines_spec.rb spec/jobs/ledi spec/services/care_lines` → PASS.
- [ ] **Step 5: Commit** `feat: keep the caps record in a team-only timeline with a link marker`.

---

### Task 15: Painel do CAPS

**Files:**
- Create: `app/services/psychosocial/dashboard.rb`, `app/controllers/psychosocial/dashboard_controller.rb`, `spec/requests/psychosocial/dashboard_spec.rb`
- Modify: `config/routes.rb`

**Interfaces:**
- Produces: `Psychosocial::Dashboard.call(health_unit_id: nil, month: Date.current.beginning_of_month) -> Hash` (contrato §11; `{ enrollments_active: [ { health_unit_id, name, modality, count } ], new_in_month, closed_by_destination: { "01" => n, … }, care_plans_overdue, actions_by_instrument: { raas_psi, bpa_c, bpa_i }, bed_occupancy: { nights, bed_days }, raas_month: { open, closed, exported } }`, cada contagem por `Admin::SmallCount.wrap`; somas de noites e leitos-dia também); `GET /psychosocial/dashboard?health_unit_id?` (`psychosocial_lead` — só CAPS lotados —, `municipal_admin` e `analyst`; sem trilha; sem nome, CID ou texto).
- [ ] **Step 1: Write the failing tests:** 3 matrículas → `{ "suppressed" => true }`; 6 → 6; 0 → 0; `analyst` lê; `psychosocial_staff` → 403 `missing_role`; o corpo não tem nome de paciente nem CID; `health_unit_id` de CAPS não lotado para a coordenação → 403 `not_on_caps_team`.
- [ ] **Step 2–4:** FAIL → implementar → PASS (`bin/test api -w mod31 spec/requests/psychosocial/dashboard_spec.rb`).
- [ ] **Step 5: Commit** `feat: add the caps dashboard with small-count suppression`.

---

### Task 16: Exclusão LGPD

**Files:**
- Create: `app/services/psychosocial/anonymize.rb`, `spec/commands/citizens/erase_psychosocial_spec.rb`
- Modify: `app/commands/citizens/request_erasure.rb` (`psychosocial_recorded?(ids)` em `examined_patients?`), `app/commands/citizens/erase.rb` (`Psychosocial::Anonymize.patients!(ids)` no ramo não retido), `spec/commands/citizens/request_erasure_spec.rb`, `db/city_triggers.sql` (encaminhamento e acolhimento aceitam a anonimização uma vez: `anonymized_at` e as cifradas a NULL na mesma UPDATE)

**Interfaces:**
- Produces: `Citizens::RequestErasure.psychosocial_recorded?(ids) -> Boolean` (matrícula em qualquer situação, ou participante de ação com o paciente — cancelada inclusive —, ou permanência, ou caso de matriciamento: registro de saúde retido); `Psychosocial::Anonymize.patients!(patient_ids, now: Time.current) -> Integer` (encaminhamentos e acolhimentos dos pacientes não retidos: `reason`/`notes` → NULL, `anonymized_at = now`, onde `anonymized_at IS NULL`; D10).
- [ ] **Step 1: Write the failing tests:** pedido de quem tem matrícula encerrada → `retained`; de quem só foi encaminhado → `confirmed` e o motivo do encaminhamento vira NULL com `anonymized_at`; de quem teve acolhimento `no_follow_up` cuja ação foi cancelada → `retained` (a ação cancelada é registro); o trigger recusa uma segunda anonimização.
- [ ] **Step 2–4:** FAIL → implementar → PASS (`bin/test api -w mod31 spec/commands/citizens`).
- [ ] **Step 5: Commit** `feat: retain caps records and anonymize bare referrals on erasure`.

---

### Task 17: Semente de dev realista

**Files:**
- Create: `lib/psychosocial_crew.rb`, `spec/lib/psychosocial_crew_spec.rb`
- Modify: `lib/sigtap_sample.rb` (já na Task 1; aqui só confirmar), `db/seeds.rb` (`PsychosocialCrew.seed_platform!` depois do `OralHealthCrew.seed_platform!`; `PsychosocialCrew.seed_current_city(slug:, ddd:, password:)` + `summary_lines` depois do `TransportCrew`)

**Interfaces:**
- Consumes: tudo acima; `DevTerminology.ensure_sigtap!(codes: SigtapSample::PSYCHOSOCIAL_CODES, competences:, label: "semente do CAPS: SIGTAP")`; `ProfessionalCrew.ensure_link`-like (confira o nome real; use os comandos de vínculo do 10); `CareLinesCrew` (equipe e UBS); `ScreeningCrew.send(:cpf_for, …)`; `Professionals::Cns.generate`.
- Produces: `PsychosocialCrew.seed_platform!`; `PsychosocialCrew.seed_current_city(slug:, ddd:, password:)` idempotente (Valor fixado 7; só Curitiba; tudo pelos comandos — `SaveBed`, `SaveGroup`, `RecordReception`, `CreateEnrollment`, `SaveCarePlan`, `RecordAction`, `StartNightStay`, `RecordMatrixSupport`, `AddMatrixCase`, `UpdateMonthlyRecord`; o PTS vencido com `review_on` 10 dias atrás é gravado direto, porque o comando recusa data no passado); bonds no último retrato do CNES de Curitiba (`CnesProfessionalBond.create!` se faltar, sem reescrever o retrato); `summary_lines` só com e-mail, papel e otpauth.
- [ ] **Step 1:** spec da crew: duas rodadas → mesmas contagens de CAPS, leitos, grupos, matrículas, PTS, ações e permanências; `psychosocial_care` utilizável; o AD III tem um leito ocupado; o mês corrente do CAPS II sem pendência para o Renato; nenhuma linha da saída com CPF, CNS, nome de paciente ou CID; os CPFs e CNS gerados passam em `CitizenIdentity::Cpf.valid?` e `Professionals::Cns.valid?`.
- [ ] **Step 2–4:** FAIL → implementar → PASS (`bin/test api -w mod31 spec/lib/psychosocial_crew_spec.rb`).
- [ ] **Step 5: Commit** `chore: seed dev caps units, team and fictitious users`.

---

### Task 18: Specs de sigilo, guarda de eventos e suíte completa

**Files:**
- Create: `spec/invariants/psychosocial_secrecy_spec.rb`
- Modify: nenhum código de produção (se um exemplo falhar, a correção vai na task dona e volta aqui)

**Interfaces:**
- Consumes: tudo acima. Produces: a guarda de sigilo do F-31.8.

- [ ] **Step 1: Write the failing test** `spec/invariants/psychosocial_secrecy_spec.rb` — um cenário completo (matrícula com CID `F200` e substância, PTS, ação com nota "Relata ideação", ação coletiva, receita controlada, BPA-I do acolhimento, permanência, matriciamento com caso) e depois:

```ruby
RSpec.describe "Sigilo do CAPS", type: :request do
  let!(:scenario) { full_caps_scenario! } # helper em psychosocial_helpers.rb; devolve { patient:, citizen:, caps:, document: }
  let(:secrets) { [ "F200", "F20.0", "Relata ideação", "Discutido manejo", scenario[:caps].name ] }

  it "nada nas rotas do cidadão" do
    sign_in_citizen!(scenario[:citizen])
    %w[/citizen/history /citizen/documents /citizen/notices /citizen/appointments /citizen/trips].each do |path|
      get path, params: { citizen_id: scenario[:citizen].id }
      expect(response.body).not_to include(*secrets), path
    end
    get "/citizen/documents/#{scenario[:document].id}/pdf", params: { citizen_id: scenario[:citizen].id }
    expect(response).to have_http_status(:not_found)
  end

  it "na linha do tempo geral e no PEC, só o marcador" do
    doctor = scenario[:ubs_doctor]
    sign_in_as(doctor)
    get "/timelines/#{scenario[:patient].id}"
    kinds = json_body["items"].map { |i| i["kind"] }
    expect(kinds).to include("psychosocial_link")
    expect(kinds).not_to include("psychosocial")
    expect(response.body).not_to include("F200", "Relata ideação", "Receita")
    expect(PecTimelinesController::KINDS).not_to include("psychosocial", "psychosocial_link")
  end

  it "nada no LEDI, em aviso ou SMS" do
    expect(LediFicha.where(created_at: scenario[:started_at]..)).to be_empty
    expect(AppointmentNotice.where(created_at: scenario[:started_at]..)).to be_empty
    expect(OutboundMessage.where(created_at: scenario[:started_at]..)).to be_empty
  end

  it "nada em evento, log ou URL" do
    payloads = DomainEvent.where(created_at: scenario[:started_at]..).pluck(:payload).to_json
    expect(payloads).not_to include(*secrets)
    expect(scenario[:log]).not_to include(*secrets)
    expect(scenario[:paths].join).not_to match(/F20|ideação|\d{11}|\d{15}/)
  end
end
```

`full_caps_scenario!` grava o cenário pelos comandos e pelas rotas e devolve `{ patient:, citizen:, caps:, document:, ubs_doctor: (em contexto), started_at:, log: (Rails.logger capturado como o `capture_log` do 19a), paths: (URLs pedidas) }`. Confira as rotas reais do cidadão (`config/routes.rb`, `scope :citizen_api`) e da linha do tempo profissional (`TimelinesController`) e troque os caminhos dos exemplos pelos que existem.

- [ ] **Step 2:** `bin/test api -w mod31 spec/invariants/psychosocial_secrecy_spec.rb` → PASS (se falhar, corrigir na task dona com teste, e rodar de novo).
- [ ] **Step 3:** `bin/test api -w mod31 spec/initializers/domain_events_bindings_spec.rb spec/initializers/filter_parameter_logging_spec.rb spec/events spec/architecture spec/adr_pointers_spec.rb spec/config spec/invariants` → PASS; `rg -n "DomainEvents.publish\(" app/commands/psychosocial app/services/psychosocial app/jobs/psychosocial app/commands/billing/from_psychosocial.rb -A4` conferindo cada chamada inteira (só ids e o `month`).
- [ ] **Step 4: Commit** `test: guard caps secrecy across citizen routes, timelines, ledi, events and logs`.
- [ ] **Step 5:** pedir à sessão "Testes" (`SendMessage`): `api · mod31/feat/mod-31-psychosocial-care · --full · módulo 31 (migração de plataforma SIGTAP, três de cidade, ck_memberships_role, performed_procedures, clinical_documents, Billing::Critic/Lines, Timeline, Citizens::Erase, spec/support)`. Falha → diagnóstico nesta sessão, correção com teste, nova rodada.
- [ ] **Step 6:** nenhum card só do api neste módulo (todos dependem do dashboard). Prova no navegador só com autorização do usuário (migrar plataforma e cidades, semear o dev); depois o plano do dashboard. Gates de go-live a registrar no board #2 e no runbook `operacao/rollout-atencao-psicossocial.md`: layout da remessa RAAS (D6) e homologação da importação no aplicativo RAAS/SIA; códigos, instrumentos, CBOs, quantidade máxima, permanência e atributo `036` dos procedimentos do CAPS no SIGTAP oficial (inclusive acolhimento inicial, diurno e noturno); nome do arquivo de atributos complementares no pacote (D1) e reimportação das competências ativas depois do deploy (as releases antigas ficam sem limites); limite do CAPS III na norma; listas de origem, destino e substâncias da versão vigente do aplicativo; idade do BPA-C sem paciente (`PATIENTLESS_AGE`); modalidade e classificação 115 de cada CAPS no CNES; leitos do CNES; faixa de APAC do gestor.

---

## Divergências propostas ao contrato

- **D1 — Arquivo de atributos do SIGTAP (§12).** O contrato fala em "relacionamento de atributos complementares do pacote oficial" e em localizar o importador "pela carga de `sigtap_releases`"; no código, a release é `terminology_releases` (`kind: "sigtap"`) e o pacote não tem arquivo com "atributo" no nome. Proposta: ler `rl_procedimento_detalhe` (`CO_DETALHE`, 3 dígitos), nome fixado numa constante e conferido no pacote (gate); arquivo ausente = zero atributos, sem recusar a release. `QT_MAXIMA_EXECUCAO` e `QT_DIAS_PERMANENCIA` com `0`/`9999` = sem limite (nulo).
- **D2 — CHECK do BPA-C sem paciente (§8).** `instrument` é recalculado no fechamento da competência do 22 (`Billing::Recalculate`): com `patient_id IS NOT NULL OR (source = 'psychosocial' AND instrument = 'bpa_c')`, uma SIGTAP nova que tire o `01` do código faria o fechamento estourar o CHECK. Proposta: `patient_id IS NOT NULL OR (source = 'psychosocial' AND psychosocial_action_id IS NOT NULL)`; a linha sem paciente que deixar de ser BPA-C fica `not_bpa`, não faturável (`not_bpa_instrument`/`missing_patient_id`), fora do arquivo.
- **D3 — Código novo da crítica.** A ação `bpa_i` precisa de exatamente um paciente (a linha leva um paciente). Proposta: `bpa_i_single_patient` em `problems[]`. E o instrumento sai da SIGTAP com precedência: `09` com participante matriculado → `raas_psi`; senão `02` → `bpa_i`; senão `01` → `bpa_c` (o acolhimento inicial com `02` e `09` vira RAAS na matrícula e BPA-I sem ela).
- **D4 — Pré-preenchimento do ESF.** O contrato diz que `esf_cnes` é pré-preenchido pelo território do 11, sem rota. Proposta: `GET /psychosocial/prefill?patient_id=&health_unit_id=` → `{ esf_covered, esf_cnes }` (cadastro individual ativo do 24a em equipe `70` ativa → CNES da unidade da equipe).
- **D5 — Pendência `night_stay_open`.** Com a permanência aberta atravessando a virada do mês, as noites do mês anterior só viram ação no fechamento da permanência; fechar o mês antes travaria essas ações. Proposta: a pendência `night_stay_open` no §6 (bloqueia o fechamento do mês até a permanência fechar).
- **D6 — Remessa sem layout.** O gerador `Psychosocial::RaasFile` entra com o registro de layouts vazio (`LAYOUTS = {}`), porque o layout oficial não foi obtido; com `layout_version` sem layout registrado, 422 `raas_layout_unavailable`. O fluxo (estados, competência, trilha, arquivo imutável) é provado com um layout de teste. Exportar exige mês fechado (409 `month_open`), e a segunda remessa do mesmo mês é 409 `already_exported` (sem versão nova enquanto o layout não existir).
- **D7 — Tipos de documento pela ação (§4, §9).** "Os tipos são os do 19c e do 19d": a declaração de comparecimento precisa do atendimento e a requisição de exame precisa da consulta. Proposta: `prescription`, `controlled_notification_record` e `sick_note`; os outros → `invalid_content` `field: "kind"`. A receita do CAPS não entra na lista de medicamentos do paciente (a UBS a veria no prontuário).
- **D8 — `citizen_id` e o pai do documento (§9).** `clinical_documents.citizen_id` é NOT NULL e vem do atendimento; o documento do CAPS não tem atendimento. Proposta: `citizen_id` NULO exatamente quando `psychosocial_action_id` existe (`ck_clinical_documents_citizen`), o que já o tira de toda consulta por par do cidadão (wpda, linha do tempo, exclusão), e `num_nonnulls(attendance_id, psychosocial_action_id) = 1` no lugar dos dois CHECKs do contrato (mesmo efeito, uma regra). A leitura pelas rotas do 19 (`/clinical_documents/:id`) exige a lotação no CAPS (senão 404).
- **D9 — "Caixa da equipe" do PTS vencido (§4).** Não existe caixa de avisos da equipe. Proposta: o job publica `psychosocial.care_plan_review_due` uma vez por versão vencida (`psychosocial_care_plan_alerts`, tabela além do ADR) e a "caixa" é `GET /psychosocial/care_plans/due?health_unit_id=` (técnico de referência vê as suas; a coordenação, todas).
- **D10 — Familiar sem cadastro na exclusão (§14).** O familiar só existe em ação com participante matriculado, e o titular com ação é sempre retido; a anonimização "com o titular" nunca acontece. Proposta: tirar a frase do §14; a exclusão anonimiza só o encaminhamento e o acolhimento de quem não tem registro de saúde do CAPS (o acolhimento com ação — cancelada inclusive — já retém).
- **D11 — Pendência `no_actions` bloqueia o mês.** Pelo §6, um usuário em permanência sem nenhuma ação no mês impede o fechamento do CAPS inteiro. O plano implementa como está; proposta para o coordenador: `no_actions` como aviso (o registro fica fora da remessa e da ficha), sem bloquear.
- **D12 — Quem gera a remessa.** O §6 diz quem baixa (`psychosocial_lead` e `billing_clerk`), não quem gera. Proposta: os mesmos dois, com step-up; o `billing_clerk` sem lotação.
- **D13 — Recusas e formas extras.** `already_closed` (matrícula ou mês), `already_cancelled`, `bed_label_taken` (409), `month_open` (409), `out_of_context` no encaminhamento e no vínculo (o encaminhador precisa poder abrir o paciente pelo ADR 0031), 422 da crítica com `error: "action_rejected"`; `GET /psychosocial/night_stays/:id`, `GET /psychosocial/actions/:id/documents`, `GET /psychosocial/enrollments/:id/timeline` (já no §10) e `GET /psychosocial/matrix_supports`. Na linha do tempo geral, o marcador é um kind novo, `psychosocial_link`, em `Timeline::KINDS` e em `PROFESSIONAL_ONLY_KINDS`, fora do PEC; a fonte `psychosocial` fica só em `CAPS_ONLY_KINDS`.
