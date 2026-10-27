# Módulo 25 — Vigilância epidemiológica (api) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **PRÉ-REQUISITO: módulo 24a em `origin/main` do api** (conferido ao escrever este plano em `10c2ab9b`: `Platform::Features::CATALOG` terminando em `territory_registry`; última migração de cidade `20261023900001_add_territory_registry.rb`; `db/city_schema.rb` em `define(version: 2026_10_23_900001)`; última migração de plataforma `20261022900001_create_care_line_templates.rb`, `db/platform_schema.rb` em `define(version: 2026_10_22_900001)`; `spec/adr_pointers_spec.rb` com `VALID_RANGE = (1..42)`; `Membership::ROLES` terminando em `viewer` com `billing_clerk` e `ck_memberships_role` reescrito por `AddProcedureBilling::ROLES` (restaurado à mão em `spec/models/create_analytics_migration_spec.rb` e `create_campaigns_migration_spec.rb`); `consultation.finalized` e `consultation.addendum_added` ligados só a `Signatures::RequestJob`; `Citizens::RequestErasure.examined_patients?` terminando em `territory_registered?(ids)`; `MaintenanceAudit::NAMES` terminando em `maintenance.care_line_template.changed`; `patient_problems.code` `limit: 4`, sem ponto). **A Task 0 confere a base: se um nome mudou, PARE e reporte.**

**Goal:** O lado api do módulo 25 (F-25.1 a F-25.8, ADR 0043): interruptor e papel `surveillance`; lista nacional de notificação compulsória versionada na plataforma (carga da Portaria 11.211/2026 com fonte por linha) e doenças locais da cidade; sugestão de notificação na consulta e rede de segurança na finalização e no adendo; fila da vigilância com notificação manual, ficha no formato da notificação individual do Sinan, andamento até o número oficial; prazos, alertas por e-mail sem dado do paciente e notificação negativa semanal; PDF, CSV e painel por bairro e semana epidemiológica com supressão; retenção LGPD; semente de dev em Curitiba; runbook.

**Architecture:** A lista é dado: versões imutáveis na plataforma (`notifiable_disease_versions` + `notifiable_diseases`, trigger em `db/platform_triggers.sql`), carregadas do YAML que a prova técnica (Task 0) produz e publicadas pelo maintenance; a cidade lê a versão vigente mais as suas `local_notifiable_diseases`. Toda notificação nasce por um único comando (`Surveillance::CreateNotification`) — da consulta, manual ou da rede de segurança —, que aplica a regra de duplicidade sob lock consultivo por paciente e doença, com o índice único parcial "uma aberta por paciente e doença" como última palavra. O andamento vive em `disease_notifications` (estado atual, `version` otimista) e em `disease_notification_events` (imutável). E-mails saem de jobs dedicados fora do consumidor (ADR-0005), só com doença e unidade.

**Tech Stack:** Rails 8.1 (API), PostgreSQL 16 (banco por cidade + plataforma), RSpec via `bin/test`, Prawn (já no Gemfile, PDFs do 19c/21b), GraphQL-Ruby (maintenance), ActionMailer. Nenhuma gem nova.

**Spec:** `docs/superpowers/specs/2026-10-27-module-25-disease-surveillance-design.md` e `docs/adr/0043.md`. Contrato entre apps — **fonte única de formatos**: `docs/superpowers/plans/2026-10-27-module-25-disease-surveillance-contracts.md` (convenções dos contratos do 21a §15, 21b §14, 21c §11, 22 §12–§13, 23 §11–§12 e 24a §14); toda diferença está em "Divergências propostas ao contrato", no fim (prefixo **V**). Pesquisa: `docs/pesquisa/2026-10-05-integracoes/frente-5-vigilancia.md` (A1, A2, A5). Lições que valem aqui: contadores sem trilha; ordem de travas estável; recusar em vez de cortar (texto longo → 422 com `path`); vírgula decimal no PDF; todo `Data`/`Struct`/modelo com dado pessoal sem dado em `inspect`/`to_s`/`pretty_print`/`as_json`. "Todas as recusas de uma vez" e "estrutura HL7 sempre AR" não se aplicam (não há HL7): o validador da ficha devolve a **primeira** recusa com `path`, como o 22 e o 24a.

## Leitura preliminar das regras oficiais (feita ao escrever este plano)

Fontes (a Task 0 baixa, grava SHA-256 e confirma item a item; **o que divergir, ela corrige aqui, no YAML e no contrato antes de qualquer outra task**):
- Portaria GM/MS 11.211/2026 — `https://bvsms.saude.gov.br/bvs/saudelegis/gm/2026/prt11211_14_05_2026.html` (Anexo 1 do Anexo V da PRC 4/2017).
- Ficha de Notificação Individual do Sinan (NINDIV) — `https://portalsinan.saude.gov.br/images/documentos/Agravos/NINDIV/Notificacao_Individual_v5.pdf` (o caminho exato pode ter mudado; o Step 3 da Task 0 parte de `https://portalsinan.saude.gov.br/` e registra a URL real).
- Fichas de investigação de cada agravo (cabeçalho "Agravo/doença" + "Código (CID10)") — `https://portalsinan.saude.gov.br/` (seção de cada agravo).
- Calendário epidemiológico do Sinan — `https://portalsinan.saude.gov.br/calendario-epidemiologico` (ou a página que o portal apontar).

1. **Lista e prazos:** 66 itens; "imediata" = até 24 h (para MS, SES ou SMS — a esfera não muda o prazo do município), "semanal" = até 7 dias. Caxumba: o art. 2º diz imediata, o anexo diz semanal; vale o anexo (resposta do MS no Fala.BR, 13/08/2026) — a linha carrega a nota. Agravos com prazo que depende da forma (dengue: casos semanal, óbito imediata; chikungunya; Zika; Oropouche) entram como **duas doenças** quando a CID não distingue a forma? **Não**: entram como uma doença com o prazo dos **casos** (semanal), e a forma grave/óbito vira doença própria só quando tem CID própria; o resto fica como nota da linha (`notes`) — o médico notifica a forma imediata pela doença própria ou "à mão" (Desvio 3).
2. **CID-10 por doença:** o código que a ficha de investigação do Sinan imprime no cabeçalho (ex.: dengue `A90`, chikungunya `A92.0`, Zika `A92.8`, Oropouche `A93.0`, febre amarela `A95.9`, leptospirose `A27.9`, hantavirose `B33.4`, febre maculosa `A77.9`, tuberculose `A16.9`, hanseníase `A30.9`, sífilis adquirida `A53.9`, sífilis em gestante `O98.1`, sífilis congênita `A50.9`, HIV/AIDS `B24`, hepatites virais `B19`, coqueluche `A37.9`, difteria `A36.9`, sarampo `B05.9`, rubéola `B06.9`, varicela `B01.9`, caxumba `B26.9`, meningite `G03.9`, doença meningocócica `A39.9`, tétano acidental `A35`, tétano neonatal `A33`, raiva humana `A82.9`, atendimento antirrábico `W64`, animal peçonhento `X29`, intoxicação exógena `T65.9`, violência `Y09`, malária `B54`, leishmaniose visceral `B55.0`, tegumentar `B55.1`, Chagas aguda `B57.1`, esquistossomose `B65.9`, febre tifoide `A01.0`, cólera `A00.9`, botulismo `A05.1`, PFA/poliomielite `A80.9`, peste `A20.9`, Mpox `B04`, esporotricose `B42`, toxoplasmose gestacional `O98.6`, congênita `P37.1`). **Mais** as categorias de 3 caracteres que a ficha manda notificar (ex.: tuberculose `A15`–`A19`), com fonte. Gravado **sem ponto** (`A920`), como `patient_problems.code`.
3. **CIAP-2 equivalente:** só quando o código CIAP-2 corresponde a **uma** doença da lista (ex.: `A70` tuberculose, `A71` sarampo, `A72` varicela, `A73` malária, `A74` rubéola, `A78` hanseníase — confirmar a adaptação brasileira —, `B90` HIV, `D71` caxumba, `D72` hepatite viral, `R71` coqueluche, `X70`/`Y70` sífilis adquirida); código ambíguo (ex.: `A77` "outras doenças virais", usado para dengue e outras arboviroses) **fica de fora** (o enfermeiro usa "Notificar suspeita"). Fonte: a tabela CIAP-2 vigente do e-SUS APS (Task 0).
4. **Ficha de notificação individual (NINDIV, v5):** dados gerais (1 tipo de notificação = 2 individual; 2 agravo/doença + código CID-10; 3 data da notificação; 4 UF; 5 município de notificação + IBGE; 6 unidade + CNES; 7 data dos primeiros sintomas); notificação individual (8 nome; 9 nascimento; 10 idade; 11 sexo M/F/I; 12 gestante 1–6, 9; 13 raça/cor 1 branca, 2 preta, 3 amarela, 4 parda, 5 indígena, 9 ignorado; 14 escolaridade 0–10; 15 Cartão SUS; 16 nome da mãe); residência (17 UF; 18 município + IBGE; 19 distrito; 20 bairro; 21 logradouro; 22 número; 23 complemento; 24–25 geocampos; 26 ponto de referência; 27 CEP; 28 telefone; 29 zona 1 urbana, 2 rural, 3 periurbana, 9 ignorado; 30 país, se fora do Brasil); notificador (nome, função, assinatura). **Raça/cor do Sinan ≠ BPA** (no paciente, `03` é parda e `04` amarela).
5. **Semana epidemiológica:** domingo a sábado; a **semana 1 termina no primeiro sábado de janeiro que tem pelo menos 4 dias de janeiro** (= a semana que contém 4 de janeiro); um ano pode ter 53 semanas. Exemplos calculados: `2021-01-02` → `2020-53`; `2021-01-03` → `2021-01`; `2022-01-01` → `2021-52`; `2022-01-02` → `2022-01`; `2026-01-03` → `2025-53`; `2026-01-04` → `2026-01`; `2026-10-27` → `2026-43`; `2027-01-02` → `2026-52`. **Diverge do contrato §1** ("semana 1 = a que contém o primeiro sábado do ano" daria `2022-01-01` → `2022-01`): V1.
6. **Notificação negativa:** toda semana epidemiológica de cada unidade notificadora tem notificação ou negativa (manual de gestão do Sinan, SES-RJ 2025).

## Desvios e precisões

1. **Correspondência de código.** Um código da lista com 4 caracteres casa exato; com 3, casa a categoria (`A15` casa `A150`…`A159` e `A15`). Nenhum código (nem categoria que cubra outro) aparece em duas doenças da mesma versão, nem entre a plataforma e as doenças locais da cidade: um problema dá no máximo uma doença (V3). CIAP-2 casa exato.
2. **O que dispara.** Só item `add` da consulta (rascunho: `draft_items["evaluated_problems"]` com `action == "add"`; finalizada: `consultation_problems` com `action == "add"`, adendos inclusive). `evaluate`, `resolve` e `correct_onset` nunca disparam.
3. **Forma grave/óbito.** Prazo da doença é o da forma "casos" da portaria; a forma imediata com CID própria é doença própria; sem CID própria, fica na nota (`notes`) e a notificação manual pela vigilância cobre (Leitura 1).
4. **Duplicidade (regra única, `Surveillance::Duplicates.blocking`).** Bloqueia sugestão, "possível" e nova notificação: notificação do paciente para a doença com `status ∈ OPEN_STATUSES` (`possible notified investigating`) **ou** com `closed_at` (fechada ou digitada) no fuso da cidade há menos que `repeat_after_days`. `dismissed` não bloqueia (a vigilância disse "não é caso"; evidência nova pode voltar). Concorrência: `pg_advisory_xact_lock` por `surveillance:<patient_id>:<disease_code>` antes da leitura; o índice único parcial `idx_disease_notifications_one_open` decide o resto (`RecordNotUnique` com o nome do índice → `already_notified`).
5. **Prazo e vencida.** `due_at` = instante da suspeita + 24 h (`immediate`) ou + 7 dias (`weekly`). Instante da suspeita: consulta → agora; rede de segurança → `finalized_at` da consulta ou `created_at` do adendo; manual → `suspected_on` às 00:00 no fuso da cidade. **Vencida** (`overdue`) = `due_at < agora` e `status ∈ OVERDUE_STATUSES` (`possible notified`): ninguém pegou o caso para investigar (V6). O prazo da doença é congelado na notificação (`deadline`), como a versão da lista.
6. **Notificador.** Consulta → o autor; manual → quem grava; rede de segurança → nulo até `possible → notified`, quando vira quem aceitou.
7. **Bairro** congelado na criação (`Surveillance::Neighborhood.for(patient)`): bairro da última versão do domicílio do cadastro individual ativo (24a) → `neighborhood_id` do par validado mais recente → nulo ("não informado").
8. **Ficha (`form`).** JSON cifrado com os campos da NINDIV que **não** vêm do paciente nem da notificação (§14 do contrato, Task 0); nasce pré-preenchida (`Surveillance::FormPrefill`): raça/cor do paciente convertida para o Sinan, endereço do paciente (o 24a já o regrava do domicílio), telefone do par validado, ocupação da última versão do cadastro individual (chave da Task 0 no YAML do 24a), zona e país vazios. Nome, nascimento, sexo, CNS e nome da mãe vêm sempre do paciente (só leitura na ficha).
9. **Semana de cada saída.** Painel: semana dos primeiros sintomas (sem ela, a da suspeita) — a da curva epidêmica. CSV e notificação negativa: semana da suspeita (a do lote do Sinan). Painel conta notificações fora de `possible`/`dismissed` e sem classificação `discarded` (V7).
10. **Painel fora do analytics.** Consulta direta no banco da cidade (`Surveillance::Panel`), com a semana própria (`Surveillance::EpiWeek`) — o `Analytics::Params` usa semana ISO (segunda): não reaproveitar. Supressão pela mesma regra (`Admin::SmallCount.small?`), mas o contrato pede `null`, não `{ suppressed: true }`. O ADR cita "semana epidemiológica no analytics": fica aqui, sem tocar o pipeline (V14).
11. **Trilha.** Lista da fila, CSV: um `clinical_record.viewed` por consulta (`access: "surveillance"`, `notification_ids`, `user_id`). Detalhe, PDF, criação manual: um por pessoa (`patient_id`, `person_ref`, `access: "surveillance"`), como o `TerritoryRegistry::Trail.person_viewed!`. Sugestões da consulta: sem trilha (a consulta já deixa). Contadores e painel: sem trilha.
12. **E-mails.** Consumidor de `disease_notification.created` só enfileira (`Surveillance::MailJob`, `CityScopedJob`, dedup por `ProcessedEvent` como o `DispatchMunicipalityAlertJob`); destinatários: membros ativos `surveillance` de usuários não desativados. Texto: doença e unidade, nunca paciente, bairro, data de sintomas ou link com id.
13. **Ordem de travas.** Paciente (`Patient.lock`, quando o caminho já o trava) → lock consultivo paciente×doença → notificação (`lock!`). A transição relê e trava só a notificação.
14. **Doença local.** Código `local_<a-z0-9_>` (até 46 caracteres no total); a plataforma nunca usa o prefixo (V2). Editável (nome, prazo, CIDs, janela); a notificação congela `deadline`; o nome é relido.

## Valores fixados por este plano

1. **Interruptor** (depois de `territory_registry` no `CATALOG`): `Entry.new(key: "surveillance", description: "Vigilância epidemiológica: notificação compulsória, fila da vigilância, notificação negativa e painel por semana epidemiológica", requires: [])`.
2. **`Surveillance::Catalog`:** `DEADLINES = { "immediate" => 24.hours, "weekly" => 7.days }`, `ORIGINS = %w[consultation manual safety_net]`, `STATUSES = %w[possible notified investigating closed entered dismissed]`, `OPEN_STATUSES = %w[possible notified investigating]`, `OVERDUE_STATUSES = %w[possible notified]`, `CLASSIFICATIONS = %w[confirmed discarded inconclusive]`, `TRANSITIONS = { "possible" => %w[notified], "notified" => %w[investigating], "investigating" => %w[closed], "closed" => %w[entered] }`, `DEFAULT_REPEAT_AFTER_DAYS = 30`, `NOTIFIER_CBO_PREFIXES = %w[225 2235]`, `REASON_MAX = 500`, `OFFICIAL_NUMBER = /\A[0-9A-Za-z.\/-]{1,30}\z/`, `PLATFORM_CODE = /\A(?!local_)[a-z][a-z0-9_]{1,47}\z/`, `LOCAL_CODE = /\Alocal_[a-z0-9_]{1,40}\z/`, `CID10 = /\A[A-Z][0-9]{2}[0-9X]?\z/`, `CIAP2 = /\A[A-Z][0-9]{2}\z/`, `NAME_MAX = 120`, `IBGE = /\A[0-9]{7}\z/`, `RACE_TO_SINAN = { "01" => "1", "02" => "2", "03" => "4", "04" => "3", "05" => "5" }` (sem raça → `"9"`), `SEX_TO_SINAN = { "male" => "M", "female" => "F" }` (resto `"I"`), `TRAIL_ACCESS = "surveillance"`.
3. **Listas:** `limit` padrão 50, máximo 100 (inválido → 422 `invalid_limit`); ordem da fila: imediatas abertas e vencidas primeiro (`overdue DESC, deadline = 'immediate' DESC, due_at ASC, id ASC`); `next_before` = cursor opaco da última linha (`Surveillance::Cursor.encode([overdue, immediate, due_at, id])`, base64 url-safe; `decode` inválido → 422 `invalid_cursor`), porque a ordem não é por id.
4. **Painel:** `PANEL_DEFAULT_WEEKS = 12`, `PANEL_MAX_WEEKS = 53`, `SIGNAL_BASELINE_WEEKS = 4`, `SIGNAL_FACTOR = 2`, `SIGNAL_MIN = 5`; sinal = contagem da semana ≥ 5 **e** > 2 × média das 4 semanas anteriores (as anteriores entram pela contagem crua; o sinal só sai verdadeiro em célula não suprimida).
5. **Jobs (`config/recurring.yml`):** `surveillance_overdue` `Surveillance::OverdueJob` "every day at 7:15am America/Sao_Paulo"; `surveillance_negatives` `Surveillance::NegativesReminderJob` "every monday at 7:30am America/Sao_Paulo" (fila `housekeeping`).
6. **Dev:** api do worktree na porta **3047**; worktree `apps/api/.claude/mod25` (com `config/master.key`); branch `feat/mod-25-disease-surveillance`; semente em Curitiba: vigilante "Patrícia Helena Duarte" (`vigilancia@curitiba.demo`, papel `surveillance`, OTP fixo por `DEV_SURVEILLANCE_OTP_SECRET` com padrão `VIGILANCIAEPIDEMIOLOGICASEMENTE2`), doença local `local_acidente_escorpionico_pr` "Acidente escorpiônico (vigilância estadual PR)", semanal, CID `X22` (a Task 0 confere que `X22` não está na lista nacional; se estiver, a semente usa outro agravo estadual do PR com fonte); telefones com prefixo `96666` a partir de `2600`.

## Global Constraints

- git `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta); nunca `git add -A`; commits em inglês, Conventional Commits por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`; merge, push e tag só com autorização do usuário.
- **Primeira ação da execução:** mover os cards F-25.1..F-25.8 do board #1 para In Progress (`gh project item-edit …`, um comando `gh` por chamada, começando com `gh`). Um card só vai para Done quando o ÚLTIMO app dele terminar; Verified depois da prova no navegador.
- Testes só por `bin/test api -w mod25 <specs>` (a partir de `/Users/eduardovrocha/Development/ioit.solutions/rota-saude`); nunca `rspec` direto nem o Postgres do host; sem bancos com sufixo; a suíte completa é pedida à sessão "Testes" (Task 19).
- Banco de cidade: `db/city_migrate/20261027900002_add_disease_surveillance.rb`, dump à mão em `db/city_schema.rb` (`define(version: 2026_10_27_900002)`); triggers em `db/city_triggers.sql`, executado no fim do `up` (o dump não representa trigger). Plataforma: `db/platform_migrate/20261027900001_create_notifiable_diseases.rb`, `db/platform_schema.rb` (`define(version: 2026_10_27_900001)`), trigger em `db/platform_triggers.sql`. Ambas irreversíveis. **O banco de dev só é migrado/semeado na prova no navegador, com autorização do usuário.**
- Specs de requisição com `type: :request`; nunca simule cidade inalcançável com URL real.
- Erros `{ "error": "<reason>" }` (com `path` ou `field` quando o contrato manda); interruptor 403 `{ "error": "feature_disabled", "feature": "surveillance" }` (na consulta, também `clinical_record`); step-up 401 `mfa_required`; Content-Type JSON em toda escrita por cookie (o `Authentication` já recusa com 415 `json_required`); registro de outro escopo por id → 404; lista com escopo alheio → 403.
- **CPF, CNS, nome, nome da mãe, nascimento, endereço, telefone, doença do paciente, data de sintomas, ficha e motivo nunca em log, evento, e-mail, Analytics, mensagem de erro, URL ou `inspect`.** Só no corpo JSON (`request.request_parameters`). E-mail de alerta só com doença e unidade. Filtros de log novos: `form`, `official_number`, `reason`, `symptoms_on`, `disease_code`, `new_patient`, `residence_ibge`.
- Eventos declarados em `config/initializers/domain_events.rb` e em `spec/initializers/domain_events_bindings_spec.rb`, payload só com ids e booleanos (nunca `disease_code`: a doença do paciente é dado sensível); `DomainEvents.publish(` é multilinha — confira cada chamada inteira (memória "Grep de linha única engana").
- `Platform.audit` novo (`notifiable_disease_list.published`, `maintenance.notifiable_disease_list.published`) em `MaintenanceAudit::NAMES` (o de manutenção) e em `R18_PLATFORM_EVENT_NAMES`; nenhuma chave de payload com `name`, `email`, `cpf`, `phone`, `body` (fragmentos do R18).
- Nunca `Current.city =` solto: leia `Current.city`; em seed/rake/job, `CityConnection.with`/`EachCityJob`/`CityScopedJob`.
- Cifra por cidade: todo `encrypts` novo em `CityEncryption::CITY_KEYED_TARGETS` (guarda `spec/architecture/city_encrypted_attributes_guard_spec.rb`); colunas cifradas listadas no `rota_reencryption_only(...)` do trigger.
- `spec/adr_pointers_spec.rb` sobe para `VALID_RANGE = (1..43)`.
- Papel novo: `ck_memberships_role` reescrito pela migração de cidade com `AddDiseaseSurveillance::ROLES`; os dois specs de migração que restauram a lista à mão passam a usar a constante nova.

## Review Focus

1. **Raça/cor do paciente (códigos do BPA, `03` parda e `04` amarela) vai para a ficha e o PDF pelos códigos do Sinan (`4` parda, `3` amarela)** — uma cópia direta troca parda por amarela em toda ficha. Teste na Task 7 ("raça/cor 03 do paciente nasce 4 na ficha") e na Task 15 (PDF).
2. **Consulta finalizada ao mesmo tempo que o médico clica "Notificar"** (o consumidor da rede de segurança e o POST da consulta concorrendo): uma notificação só (a da consulta, ou a `possible` e o 409 `already_notified`), nunca duas abertas nem 500. Teste na Task 9 (thread com `pop(timeout:)` e cleanup no `after`, memória do módulo 09).
3. **Notificação na virada do ano e semana 53:** caso com primeiros sintomas em `2026-01-03` cai na `2025-53` no painel e no CSV; a notificação negativa de `2025-53` é aceita e a de `2026-53` é 422 `invalid_week`. Teste na Task 2 (EpiWeek) e na Task 14.
4. **Adendo que só reavalia** um CID de notificação já ativo (`evaluate`) não gera "possível"; adendo que **acrescenta** um CID da lista gera uma, e o redespacho do mesmo evento não gera outra. Teste na Task 9.
5. **Supressão do painel vazando pelo total ou pelo sinal:** bairro com 3 casos sai `null`, e a linha "não informado" com 1–4 também; o sinal nunca é `true` numa célula `null`. Teste na Task 16.

## Mapa de arquivos

**Criar**
- `vendor/surveillance/2026-10-27/` — páginas e PDFs oficiais baixados na Task 0 + `SOURCE` (URLs, data, SHA-256).
- `config/surveillance/notifiable-diseases.yml` (carga inicial, fonte por linha), `config/surveillance/sinan-form.yml` (campos da NINDIV que a ficha guarda).
- `db/platform_migrate/20261027900001_create_notifiable_diseases.rb`, `db/city_migrate/20261027900002_add_disease_surveillance.rb`.
- `app/models/{notifiable_disease_version,notifiable_disease,local_notifiable_disease,disease_notification,disease_notification_event,weekly_negative_report}.rb`.
- `app/services/surveillance/{gate,catalog,epi_week,disease_list,diseases,matcher,duplicates,neighborhood,form,form_prefill,json,trail,cursor,queue,panel,pdf,csv,recipients}.rb`; `app/services/surveillance/disease_list/{load,publish}.rb`.
- `app/commands/surveillance/{create_notification,notify_from_consultation,create_manual,save_form,transition,dismiss,save_local_disease,record_negative}.rb`, `app/commands/surveillance/safety_net.rb`.
- `app/controllers/concerns/surveillance_gate.rb`; `app/controllers/surveillance/{base,diseases,local_diseases,notifications,transitions,weekly_negatives,exports,panel,person_lookups,duplicates,consultation_suggestions}_controller.rb`.
- `app/jobs/surveillance/{safety_net_job,alert_job,mail_job,overdue_job,negatives_reminder_job}.rb`; `app/mailers/surveillance_mailer.rb` + `app/views/surveillance_mailer/{immediate,overdue,negatives}.{text,html}.erb`.
- `app/graphql/maintenance/types/{notifiable_disease_type,notifiable_disease_version_type,notifiable_disease_input}.rb`, `app/graphql/maintenance/mutations/publish_notifiable_disease_list.rb`.
- `lib/surveillance_crew.rb`; `lib/tasks/surveillance.rake` (`surveillance:load_disease_list`).
- Specs: `spec/services/surveillance/{disease_yaml,epi_week,disease_list,diseases,matcher,duplicates,neighborhood,form,form_prefill,panel,pdf,csv}_spec.rb`, `spec/models/surveillance_schema_spec.rb`, `spec/requests/maintenance/notifiable_diseases_spec.rb`, `spec/requests/surveillance/{diseases,local_diseases,consultation,manual,queue,transitions,weekly_negatives,exports,panel}_spec.rb`, `spec/commands/surveillance/{create_notification,safety_net}_spec.rb`, `spec/jobs/surveillance/{safety_net_job,mail_job,overdue_job,negatives_reminder_job}_spec.rb`, `spec/commands/citizens/surveillance_erasure_spec.rb`, `spec/lib/surveillance_crew_spec.rb`, `spec/support/surveillance_helpers.rb`.

**Modificar**
- `app/services/platform/features.rb` (CATALOG), `spec/adr_pointers_spec.rb` (1..43), `config/initializers/filter_parameter_logging.rb`.
- `app/models/membership.rb` (ROLES, PRIVILEGED_ROLES), `spec/models/membership_roles_spec.rb`, `spec/models/create_analytics_migration_spec.rb`, `spec/models/create_campaigns_migration_spec.rb`.
- `db/city_schema.rb`, `db/city_triggers.sql`, `db/platform_schema.rb`, `db/platform_triggers.sql`, `app/services/city_encryption.rb`.
- `app/models/health_unit.rb`, `app/controllers/health_units_controller.rb` (`notifying`).
- `app/events/maintenance_audit.rb`, `spec/events/platform_event_payload_guard_spec.rb`, `app/graphql/maintenance/types/{query_type,mutation_type}.rb`, `spec/architecture/maintenance_schema_spec.rb`.
- `config/routes.rb`, `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`, `config/recurring.yml`.
- `app/commands/citizens/request_erasure.rb`.
- `spec/support/clinical_record_helpers.rb` (`CID10` ganha `A90`, `A920`, `A169`, `B24`, `X22`).
- `db/seeds.rb` (SurveillanceCrew depois do TerritoryRegistryCrew; carga da lista na plataforma).

---

### Task 0: Prova técnica — portaria, CIDs das fichas, CIAP-2, ficha individual e semana epidemiológica

**Files:**
- Create: `vendor/surveillance/2026-10-27/*` + `SOURCE`, `config/surveillance/notifiable-diseases.yml`, `config/surveillance/sinan-form.yml`, `app/services/surveillance/catalog.rb` (só as constantes de "Valores fixados" item 2), `spec/services/surveillance/disease_yaml_spec.rb`
- Modify: este plano (seção "Leitura preliminar", se algo divergir)
- Entregável fora do api: o **§14 "Ficha de notificação individual"** em `/Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/docs-mod25/superpowers/plans/2026-10-27-module-25-disease-surveillance-contracts.md` (não commitado; a coordenadora publica)

**Interfaces:**
- Produces: `config/surveillance/notifiable-diseases.yml` com a forma abaixo (lida por `Surveillance::DiseaseList::Load`, Task 3); `config/surveillance/sinan-form.yml` (lido por `Surveillance::Form`, Task 11); `Surveillance::Catalog` com as constantes do item 2.

Forma da lista (uma entrada por doença; `sources` obrigatório em toda linha):

```yaml
ordinance: "Portaria GM/MS nº 11.211, de 13/05/2026 (Anexo 1 do Anexo V da PRC 4/2017)"
effective_from: "2026-05-14"
diseases:
  - code: dengue
    name: "Dengue (casos)"
    deadline: weekly
    cid10: [A90]
    ciap2: []
    repeat_after_days: 30
    sinan_form: dengue_chikungunya
    notes: "Óbito por dengue é imediata (notificar à mão)."
    sources:
      deadline: "prt11211 Anexo 1, item 'Dengue — casos' (vendor/surveillance/2026-10-27/prt11211.html)"
      cid10: "Ficha de investigação Dengue/Chikungunya, cabeçalho 'CID10 A90' (…/dengue_chikungunya.pdf)"
      ciap2: "sem código CIAP-2 específico (A77 é ambíguo)"
```

Forma da ficha (`sinan-form.yml`; chave = nome em snake_case inglês; `field` = número do campo na NINDIV):

```yaml
version: "NINDIV v5"
fields:
  pregnancy: { field: 12, type: code, domain: { "1": "1º trimestre", "2": "2º trimestre", "3": "3º trimestre", "4": "Idade gestacional ignorada", "5": "Não", "6": "Não se aplica", "9": "Ignorado" } }
  race_color: { field: 13, type: code, domain: { "1": "Branca", "2": "Preta", "3": "Amarela", "4": "Parda", "5": "Indígena", "9": "Ignorado" } }
  schooling: { field: 14, type: code, domain: { "0": "Analfabeto", … "9": "Ignorado", "10": "Não se aplica" } }
  residence_uf: { field: 17, type: uf }
  residence_district: { field: 19, type: text, max: 60 }
  residence_neighborhood: { field: 20, type: text, max: 60 }
  residence_street: { field: 21, type: text, max: 60 }
  residence_number: { field: 22, type: text, max: 6 }
  residence_complement: { field: 23, type: text, max: 30 }
  residence_geo1: { field: 24, type: text, max: 20 }
  residence_geo2: { field: 25, type: text, max: 20 }
  residence_reference: { field: 26, type: text, max: 60 }
  residence_zip: { field: 27, type: digits, length: 8 }
  phone: { field: 28, type: digits, min: 10, max: 11 }
  residence_zone: { field: 29, type: code, domain: { "1": "Urbana", "2": "Rural", "3": "Periurbana", "9": "Ignorado" } }
  residence_country: { field: 30, type: text, max: 40 }
  occupation_cbo: { field: null, type: cbo, source: "fichas de investigação (campo 'Ocupação'); a NINDIV v5 não tem — confirmar" }
types: [code, text, digits, uf, cbo]
```

(Limites de texto: os do layout de digitação da ficha, confirmados no PDF; os acima são do plano e a Task 0 corrige.)

- [ ] **Step 1: Mover os cards F-25.1..F-25.8 do board #1 para In Progress** (um `gh project item-edit` por card, comando começando com `gh`; ids dos campos na memória "board-91-features-roadmap").

- [ ] **Step 2: Conferir a base** — `/opt/homebrew/bin/git -C apps/api/.claude/mod25 log -1 --format=%h origin/main` e os nomes do PRÉ-REQUISITO. Divergiu → PARE e reporte.

- [ ] **Step 3: Baixar as fontes** para `vendor/surveillance/2026-10-27/` (diretório novo e vazio; `curl -sSfL`, um arquivo por fonte): a portaria (`prt11211.html`), a NINDIV (`nindiv_v5.pdf`), o calendário epidemiológico do Sinan de 2025 e 2026 (`calendario_2025.pdf`, `calendario_2026.pdf`) e a ficha de investigação de cada doença que tem ficha no Sinan (`<sinan_form>.pdf`). `SOURCE`: `fetched_on: 2026-10-27`, e por arquivo `url:` e `sha256:` (`shasum -a 256`). Fonte que não baixa (portal fora do ar, PDF movido): registrar em `SOURCE` como `unavailable:` com a URL tentada e seguir com a linha marcada `sources.cid10: "pendente"` — a linha **não entra** na carga (o spec do Step 5 recusa `pendente`) e a lacuna vai para a coordenadora.

- [ ] **Step 4: Confirmar a "Leitura preliminar"** item a item (66 itens e prazos com a nota da caxumba; CID de cada ficha; CIAP-2 só quando um-para-um; campos da NINDIV e domínios; regra da semana contra o calendário baixado — `2025-12-28..2026-01-03` = SE 53/2025 e `2026-01-04` = SE 1/2026). Divergência → corrigir a seção deste plano e avisar a coordenadora (SendMessage) antes da Task 1. `repeat_after_days`: 30 por padrão; tuberculose 270, hanseníase 365, HIV/AIDS 36500, sífilis (adquirida, gestante) 90, hepatites virais 36500 — **proposta do plano** (a portaria não fixa; registrar `sources.repeat_after_days: "proposta do plano 25 — confirmar com a vigilância"`).

- [ ] **Step 5: Escrever o teste dos YAML**

```ruby
# spec/services/surveillance/disease_yaml_spec.rb
require "rails_helper"

RSpec.describe "Lista de notificação compulsória e ficha do Sinan (prova técnica)" do
  let(:list) { YAML.safe_load_file(Rails.root.join("config/surveillance/notifiable-diseases.yml")) }
  let(:form) { YAML.safe_load_file(Rails.root.join("config/surveillance/sinan-form.yml")) }
  let(:diseases) { list.fetch("diseases") }

  it "cabeçalho da portaria e todas as linhas válidas, com fonte" do
    expect(list["ordinance"]).to include("11.211")
    expect(Date.iso8601(list["effective_from"])).to eq(Date.new(2026, 5, 14))
    diseases.each do |d|
      expect(d["code"]).to match(Surveillance::Catalog::PLATFORM_CODE)
      expect(Surveillance::Catalog::DEADLINES.keys).to include(d["deadline"])
      expect(d["name"].length).to be_between(1, Surveillance::Catalog::NAME_MAX)
      expect(d["cid10"]).to all(match(Surveillance::Catalog::CID10))
      expect(d["ciap2"]).to all(match(Surveillance::Catalog::CIAP2))
      expect(d["repeat_after_days"]).to be_between(1, 36_500)
      expect(d.dig("sources", "deadline")).to be_present
      expect(d.dig("sources", "cid10")).to be_present.and(satisfy { |s| !s.include?("pendente") })
    end
    expect(diseases.map { |d| d["code"] }.uniq.size).to eq(diseases.size)
  end

  it "nenhum código cobre outro entre doenças (um problema dá no máximo uma doença)" do
    owners = diseases.flat_map { |d| d["cid10"].map { |c| [ c, d["code"] ] } }
    owners.combination(2).each do |(a, da), (b, db)|
      next if da == db

      expect(a.start_with?(b) || b.start_with?(a)).to be(false), "#{a} (#{da}) × #{b} (#{db})"
    end
    ciap = diseases.flat_map { |d| d["ciap2"] }
    expect(ciap.uniq.size).to eq(ciap.size)
    expect(ciap).not_to include("A77")
  end

  it "valores conferidos na prova" do
    by_code = diseases.index_by { |d| d["code"] }
    expect(by_code.fetch("dengue")).to include("deadline" => "weekly", "cid10" => [ "A90" ])
    expect(by_code.fetch("caxumba")["deadline"]).to eq("weekly")
    expect(by_code.values.select { |d| d["deadline"] == "immediate" }.map { |d| d["code"] }).to include("botulismo", "colera", "sarampo", "raiva_humana")
    expect(by_code.fetch("tuberculose")["cid10"]).to include("A15", "A16", "A17", "A18", "A19")
  end

  it "ficha: campos da NINDIV com domínio de raça/cor do Sinan (≠ BPA)" do
    fields = form.fetch("fields")
    expect(fields.fetch("race_color")["domain"]).to eq("1" => "Branca", "2" => "Preta", "3" => "Amarela", "4" => "Parda", "5" => "Indígena", "9" => "Ignorado")
    expect(fields.values.map { |f| f["type"] }.uniq - form.fetch("types")).to be_empty
    expect(fields.keys).to all(match(/\A[a-z][a-z0-9_]*\z/))
  end
end
```

(Os códigos `dengue`, `caxumba`, `botulismo`, `colera`, `sarampo`, `raiva_humana`, `tuberculose` são fixados aqui; o resto da lista segue o mesmo padrão: nome da portaria em snake_case sem acento.)

- [ ] **Step 6: Rodar e ver falhar** — `bin/test api -w mod25 spec/services/surveillance/disease_yaml_spec.rb` → FAIL (YAML ausente).

- [ ] **Step 7: Escrever os dois YAML** a partir das fontes baixadas (os 66 itens; doença com forma imediata e CID própria como doença própria; `notes` quando a forma grave não tem CID). Criar `Surveillance::Catalog` só com as constantes.

- [ ] **Step 8: Rodar** — o spec → PASS.

- [ ] **Step 9: Escrever o §14 do contrato** no worktree `docs-mod25` (criar com `/opt/homebrew/bin/git -C /Users/eduardovrocha/Development/ioit.solutions/rota-saude/docs worktree add ../.worktrees/docs-mod25 origin/main` se não existir): tabela `chave` | campo NINDIV | tipo | domínio (código: descrição) | limite | pré-preenchido de — mais "Achados" (semana epidemiológica, V1; caxumba; CIAP-2 ambíguos fora; doenças sem CID própria na forma grave). Avisar a coordenadora por SendMessage: "contrato 25 §14 pronto no worktree docs-mod25 (não commitado) — publique antes das tasks de formulário do dashboard".

- [ ] **Step 10: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod25 add vendor/surveillance config/surveillance app/services/surveillance/catalog.rb spec/services/surveillance/disease_yaml_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod25 commit -m "feat: vendor the notifiable disease list and Sinan form for module 25" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 1: Interruptor `surveillance` e base das rotas

**Files:**
- Create: `app/services/surveillance/gate.rb`, `app/controllers/concerns/surveillance_gate.rb`, `app/controllers/surveillance/base_controller.rb`
- Modify: `app/services/platform/features.rb`, `spec/adr_pointers_spec.rb`, `config/initializers/filter_parameter_logging.rb`
- Test: `spec/services/surveillance/gate_spec.rb`

**Interfaces:**
- Produces: `Surveillance::Gate::KEY = "surveillance"`, `Gate.usable?(city) -> bool` (rotas, jobs diários), `Gate.enabled?(city) -> bool` (consumidores, dentro da transação da cidade), `Gate.consultation_usable?(city) -> bool` (`usable?` e `ClinicalRecord::Gate.usable?`); concern `SurveillanceGate#require_surveillance!`; `Surveillance::BaseController < ApplicationController` (`include Authentication, MfaStepUp, ::SurveillanceGate`; `wrap_parameters false`; `before_action :require_surveillance!`) com `render_failure(result)` (`{ error, path?, field?, notification_id? }`, status por `FAILURE_STATUS`), `render_error`, `forbid(reason = "missing_role")`, `not_found`, `uuid?`, `body_param(key)`, `body`, `query_only!(*allowed)`, `page_limit` (padrão 50, máx. 100), `surveillance? = Current.user.has_role?("surveillance")`.
- `FAILURE_STATUS = { already_notified: :conflict, duplicate_code: :conflict, stale_version: :conflict, unit_has_notifications: :conflict, already_recorded: :conflict, cpf_taken: :conflict, cns_taken: :conflict, citizen_not_verified: :conflict, cbo_not_allowed: :forbidden, not_author: :forbidden, not_found: :not_found }` (o resto 422).

- [ ] **Step 1: Teste**

```ruby
# spec/services/surveillance/gate_spec.rb
require "rails_helper"

RSpec.describe Surveillance::Gate do
  it "está no catálogo depois de territory_registry, sem pré-requisito" do
    entry = Platform::Features::CATALOG.find { |e| e.key == "surveillance" }
    expect(entry.requires).to eq([])
    expect(Platform::Features::KEYS.last(2)).to eq(%w[territory_registry surveillance])
  end

  it "usable? com o interruptor; consultation_usable? exige também o prontuário" do
    city = clinical_city!(record_mode: "integrated", enabled: false)
    expect(described_class.usable?(city)).to be(false)
    Platform::Features.set!(city: city, key: "surveillance", enabled: true, maintainer: ledi_maintainer!)
    expect(described_class.usable?(city)).to be(true)
    expect(described_class.consultation_usable?(city)).to be(false)
    clinical_city!
    expect(described_class.consultation_usable?(city.reload)).to be(true)
    expect(described_class.usable?(nil)).to be(false)
  end
end
```

- [ ] **Step 2: Rodar** — `bin/test api -w mod25 spec/services/surveillance/gate_spec.rb` → FAIL (`uninitialized constant Surveillance::Gate`).
- [ ] **Step 3: Implementar** a entrada do CATALOG (Valores fixados, item 1, com o comentário "ADR 0043"), o `Gate` (forma do `CareLines::Gate`), o concern (403 `{ error: "feature_disabled", feature: "surveillance" }`), a base, os filtros de log e `VALID_RANGE = (1..43)`.
- [ ] **Step 4: Rodar** — o spec + `spec/adr_pointers_spec.rb spec/services/platform/features_spec.rb` → PASS.
- [ ] **Step 5: Commit** — `feat: add the surveillance switch`.

---

### Task 2: Semana epidemiológica

**Files:**
- Create: `app/services/surveillance/epi_week.rb`
- Test: `spec/services/surveillance/epi_week_spec.rb`

**Interfaces:**
- Produces: `Surveillance::EpiWeek = Data.define(:year, :week)` com `EpiWeek.for(date) -> EpiWeek`, `EpiWeek.parse(string) -> EpiWeek | nil` (`"YYYY-WW"`, recusa semana inexistente: `2026-53` → nil), `#to_s -> "YYYY-WW"`, `#starts_on -> Date` (domingo), `#ends_on -> Date` (sábado), `#range -> Range<Date>`, `#succ`, `#pred`, `#-(n)`, `<=>`; `EpiWeek.current(today = Time.zone.today)`; `EpiWeek.between(from, to) -> [EpiWeek]`.

Algoritmo (Leitura 5): `start_of(year) = d - ((d.wday) % 7)` com `d = Date.new(year, 1, 4)` (o domingo da semana que contém 4/jan); a semana de uma data é a do maior `year ∈ {y+1, y, y-1}` com `start_of(year) <= date`, `week = (date - start_of(year)).to_i / 7 + 1`.

- [ ] **Step 1: Teste**

```ruby
# spec/services/surveillance/epi_week_spec.rb
require "rails_helper"

RSpec.describe Surveillance::EpiWeek do
  {
    "2021-01-02" => "2020-53", "2021-01-03" => "2021-01", "2022-01-01" => "2021-52", "2022-01-02" => "2022-01",
    "2025-12-28" => "2025-53", "2026-01-03" => "2025-53", "2026-01-04" => "2026-01", "2026-10-24" => "2026-42",
    "2026-10-25" => "2026-43", "2026-10-27" => "2026-43", "2027-01-02" => "2026-52", "2027-01-03" => "2027-01"
  }.each do |date, week|
    it("#{date} → #{week}") { expect(described_class.for(Date.iso8601(date)).to_s).to eq(week) }
  end

  it "domingo a sábado; parse recusa semana que não existe; between atravessa a virada" do
    w = described_class.parse("2025-53")
    expect([ w.starts_on, w.ends_on ]).to eq([ Date.new(2025, 12, 28), Date.new(2026, 1, 3) ])
    expect(described_class.parse("2026-53")).to be_nil
    expect(described_class.parse("2026-1")).to be_nil
    expect(w.succ.to_s).to eq("2026-01")
    expect(described_class.between(described_class.parse("2025-52"), described_class.parse("2026-02")).map(&:to_s))
      .to eq(%w[2025-52 2025-53 2026-01 2026-02])
  end
end
```

- [ ] **Step 2: Rodar** → FAIL. **Step 3: Implementar.** **Step 4: Rodar** → PASS.
- [ ] **Step 5: Commit** — `feat: add the Sinan epidemiological week`.

---

### Task 3: Lista da plataforma versionada (migração, modelos, carga e publicação)

**Files:**
- Create: `db/platform_migrate/20261027900001_create_notifiable_diseases.rb`, `app/models/{notifiable_disease_version,notifiable_disease}.rb`, `app/services/surveillance/disease_list.rb`, `app/services/surveillance/disease_list/{load,publish}.rb`, `lib/tasks/surveillance.rake`
- Modify: `db/platform_schema.rb`, `db/platform_triggers.sql`
- Test: `spec/services/surveillance/disease_list_spec.rb`

**Interfaces:**
- Produces (plataforma, imutável — trigger `rota_notifiable_diseases_immutable` em UPDATE/DELETE nas duas tabelas, no padrão de `care_line_templates`):
  - `notifiable_disease_versions`: `id uuid`, `version int` (único, ≥ 1), `ordinance string(200)` null false, `effective_from date` null false, `published_by_maintainer_id uuid` null, `created_at`.
  - `notifiable_diseases`: `id uuid`, `list_version_id → notifiable_disease_versions` null false, `code string(48)` (CHECK `PLATFORM_CODE` em SQL: `code ~ '^[a-z][a-z0-9_]{1,47}$' AND code !~ '^local_'`), `name string(120)`, `deadline string` (CHECK `immediate|weekly`), `cid10 jsonb` (array), `ciap2 jsonb` (array), `repeat_after_days int` (CHECK 1..36500), `sinan_form string(60)` null, `notes string(300)` null, `sources jsonb` null false; único `(list_version_id, code)`.
- `Surveillance::DiseaseList.current(today:) -> NotifiableDiseaseVersion | nil` (maior `version` com `effective_from <= today`); `DiseaseList::Load.call(path = DEFAULT_PATH) -> { created: 0|1 }` (cria a versão 1 só se a tabela está vazia; nunca sobrescreve a curadoria); `DiseaseList::Publish.call(ordinance:, effective_from:, diseases:, maintainer: nil) -> Result` (`ok(version:)` ou `fail(:invalid_list, details: { path: "diseases.3.cid10.0" })`; lock consultivo `notifiable_disease_versions`; `Platform.audit("notifiable_disease_list.published", notifiable_disease_version_id:, version:, diseases_count:)`); a validação de cada doença é a mesma regra do spec da Task 0 (formato, códigos, nenhum código cobrindo outro) em `DiseaseList.validate(diseases) -> Result`.

- [ ] **Step 1: Teste**

```ruby
# spec/services/surveillance/disease_list_spec.rb
require "rails_helper"

RSpec.describe Surveillance::DiseaseList do
  let(:entry) { { "code" => "dengue", "name" => "Dengue (casos)", "deadline" => "weekly", "cid10" => %w[A90], "ciap2" => [],
                  "repeat_after_days" => 30, "sinan_form" => nil, "notes" => nil, "sources" => { "cid10" => "ficha" } } }

  it "carga idempotente da prova técnica; vigência escolhe a versão" do
    expect(described_class::Load.call).to eq(created: 1)
    expect(described_class::Load.call).to eq(created: 0)
    v1 = NotifiableDiseaseVersion.find_by!(version: 1)
    expect(v1.diseases.find_by!(code: "dengue").cid10).to eq(%w[A90])
    v2 = described_class::Publish.call(ordinance: "Portaria X", effective_from: Date.new(2027, 1, 1), diseases: [ entry ]).payload[:version]
    expect(described_class.current(today: Date.new(2026, 12, 31))).to eq(v1)
    expect(described_class.current(today: Date.new(2027, 1, 1))).to eq(v2)
    expect(PlatformEvent.where(name: "notifiable_disease_list.published").last.payload.keys).not_to include(a_string_including("name"))
  end

  it "recusa com path: código repetido, categoria cobrindo subcategoria de outra doença, prazo desconhecido" do
    other = entry.merge("code" => "febre_x", "cid10" => %w[A900])
    expect(described_class::Publish.call(ordinance: "P", effective_from: Date.current, diseases: [ entry, other ]).details)
      .to eq(path: "diseases.1.cid10.0")
    bad = entry.merge("deadline" => "monthly")
    expect(described_class::Publish.call(ordinance: "P", effective_from: Date.current, diseases: [ bad ]).details).to eq(path: "diseases.0.deadline")
    expect(described_class::Publish.call(ordinance: "P", effective_from: Date.current, diseases: [ entry.merge("code" => "local_x") ]).details)
      .to eq(path: "diseases.0.code")
  end

  it "versão gravada é imutável no banco" do
    described_class::Load.call
    expect { NotifiableDisease.first.update_columns(name: "x") }.to raise_error(ActiveRecord::StatementInvalid, /immutable/)
  end
end
```

- [ ] **Step 2: Rodar** — `bin/test api -w mod25 spec/services/surveillance/disease_list_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** a migração (`execute File.read(platform_triggers.sql)` no fim do `up`; `down` irreversível), o dump, os modelos (`< PlatformRecord`, `readonly? = persisted?`, `NotifiableDiseaseVersion has_many :diseases`), `Load`, `Publish`, `validate`, e a rake `surveillance:load_disease_list` (chama `Load.call` e imprime a contagem).
- [ ] **Step 4: Rodar** — o spec + `spec/services/surveillance/disease_yaml_spec.rb` → PASS.
- [ ] **Step 5: Commit** — `feat: version the notifiable disease list on the platform`.

---

### Task 4: Maintenance — listar e publicar a lista (step-up e auditoria)

**Files:**
- Create: `app/graphql/maintenance/types/{notifiable_disease_type,notifiable_disease_version_type,notifiable_disease_input}.rb`, `app/graphql/maintenance/mutations/publish_notifiable_disease_list.rb`
- Modify: `app/graphql/maintenance/types/{query_type,mutation_type}.rb`, `app/events/maintenance_audit.rb`, `spec/events/platform_event_payload_guard_spec.rb`, `spec/architecture/maintenance_schema_spec.rb`
- Test: `spec/requests/maintenance/notifiable_diseases_spec.rb`

**Interfaces:**
- Consumes: `Surveillance::DiseaseList.current`, `DiseaseList::Publish.call` (Task 3).
- Produces (GraphQL): `notifiableDiseases: [NotifiableDiseaseVersion!]!` (da mais nova; `NotifiableDiseaseVersion { version ordinance effectiveFrom current createdAt publishedBy diseases }`, `current` = vigente hoje em `America/Sao_Paulo`; `publishedBy` = e-mail do mantenedor ou nulo); `NotifiableDisease { code name deadline cid10 ciap2 repeatAfterDays sinanForm notes sources }`; `publishNotifiableDiseaseList(ordinance: String!, effectiveFrom: ISO8601Date!, diseases: [NotifiableDiseaseInput!]!, code: String!) → { ok errors { path message } version }` — `step_up!(code)` como `CreateMaintenanceToken`; `audited(event: "maintenance.notifiable_disease_list.published", module_name: "surveillance")`; erro de validação → `Rejected` com o `path` em camelCase (`diseases.1.cid10.0`).

- [ ] **Step 1: Teste** (`spec/requests/maintenance/notifiable_diseases_spec.rb`, padrão de `care_line_templates_spec.rb` para sessão, e do spec de tokens para o TOTP do step-up — `travel 31.seconds` antes de gerar o código novo):
  - lista a versão 1 da carga com `current: true` e a doença `dengue` (`cid10: ["A90"]`);
  - publicar com TOTP válido grava a versão 2 (`ok: true`, `version: 2`), audita `maintenance.notifiable_disease_list.published` (attempted + ok) e `notifiable_disease_list.published`, e não abre cidade (`expect(Maintenance::CityReader).not_to receive(:call)`);
  - TOTP errado → `ok: false`, `errors: [{ path: "code" }]`, nenhuma versão nova;
  - lista inválida → `ok: false`, `errors: [{ path: "diseases.0.deadline" }]`.
- [ ] **Step 2: Rodar** → FAIL.
- [ ] **Step 3: Implementar**; declarar os dois nomes em `MaintenanceAudit::NAMES` e `R18_PLATFORM_EVENT_NAMES`; declarar os tipos e campos novos no `EXPECTED_TYPES` da guarda de schema.
- [ ] **Step 4: Rodar** — o spec + `spec/architecture/maintenance_schema_spec.rb spec/events/platform_event_payload_guard_spec.rb spec/events/maintenance_audit_spec.rb` → PASS.
- [ ] **Step 5: Commit** — `feat: publish the notifiable disease list from the maintenance API`.

---

### Task 5: Migração de cidade, modelos, papel `surveillance` e unidade notificadora

**Files:**
- Create: `db/city_migrate/20261027900002_add_disease_surveillance.rb`, `app/models/{local_notifiable_disease,disease_notification,disease_notification_event,weekly_negative_report}.rb`, `spec/models/surveillance_schema_spec.rb`, `spec/support/surveillance_helpers.rb`
- Modify: `db/city_schema.rb`, `db/city_triggers.sql`, `app/services/city_encryption.rb`, `app/models/membership.rb`, `spec/models/membership_roles_spec.rb`, `spec/models/create_analytics_migration_spec.rb`, `spec/models/create_campaigns_migration_spec.rb`, `app/models/health_unit.rb`, `app/controllers/health_units_controller.rb`, `spec/support/clinical_record_helpers.rb`

**Interfaces:**
- Produces (colunas; `enc` = `encrypts` com a chave da cidade, coluna `text`):
  - `memberships`: `ck_memberships_role` com `AddDiseaseSurveillance::ROLES` (= `AddProcedureBilling::ROLES` + `surveillance`, ordenada).
  - `health_units.notifying boolean null false default false`; o `up` marca `true` em `kind = 'ubs' AND active`.
  - `local_notifiable_diseases`: `id`, `code string(46)` único (CHECK `^local_[a-z0-9_]{1,40}$`), `name string(120)`, `deadline` (CHECK), `cid10 jsonb`, `ciap2 jsonb`, `repeat_after_days int` (CHECK 1..36500), `active bool default true`, `updated_by_user_id → users`, timestamps.
  - `disease_notifications`: `id`, `patient_id → patients` null false, `disease_code string(48)`, `disease_list_version int` null (nulo = local), `deadline` (CHECK), `origin` (CHECK `ORIGINS`), `status` (CHECK `STATUSES`), `classification` null (CHECK `CLASSIFICATIONS`; CHECK `(classification IS NOT NULL) = (status IN ('closed','entered'))`), `suspected_on date` null false, `symptoms_on date` null (CHECK `symptoms_on IS NULL OR symptoms_on <= suspected_on`), `due_at timestamptz` null false, `health_unit_id → health_units` null false, `unit_cnes string(7)` null, `notifier_user_id → users` null, `consultation_id → consultations` null, `addendum_id → consultation_addenda` null, `form enc` null false, `official_number string(30)` null (CHECK `(official_number IS NOT NULL) = (status = 'entered')`), `resident_elsewhere bool default false`, `residence_ibge string(7)` null (CHECK `resident_elsewhere = (residence_ibge IS NOT NULL) AND (residence_ibge IS NULL OR residence_ibge ~ '^[0-9]{7}$')`), `neighborhood_id → neighborhoods` null, `closed_at`, `entered_at`, `dismissed_at` (timestamptz null), `version int default 1` (≥ 1), timestamps. Índices: **único parcial `(patient_id, disease_code) WHERE status IN ('possible','notified','investigating')` `idx_disease_notifications_one_open`**; `(status, due_at)`; `(health_unit_id, suspected_on)`; `suspected_on`; `patient_id`; `consultation_id`.
  - `disease_notification_events`: `id`, `notification_id` null false, `status string` (CHECK `STATUSES`), `classification` null, `by_user_id` null, `reason enc` null, `created_at`.
  - `weekly_negative_reports`: `id`, `health_unit_id` null false, `week string(7)` (CHECK `^[0-9]{4}-[0-9]{2}$`), `recorded_by_user_id` null false, `created_at`; único `(health_unit_id, week)` `idx_weekly_negative_reports_unit_week`.
- Triggers (bloco `DO $do$ … IF to_regclass('public.disease_notifications') IS NOT NULL …`): `rota_disease_notification_guard` (BEFORE UPDATE OR DELETE: DELETE recusado; `patient_id`, `disease_code`, `disease_list_version`, `origin`, `consultation_id`, `addendum_id`, `health_unit_id`, `created_at` imutáveis; `status` saindo de `entered` ou `dismissed` recusado — fora isso, regra no código); `disease_notification_events` com `rota_reencryption_only('reason')`; `weekly_negative_reports` com `rota_append_only()` em UPDATE/DELETE; `local_notifiable_diseases` com `rota_no_delete()`; TRUNCATE recusado nas quatro.
- Modelos: `DiseaseNotification` (`serialize :form, coder: JSON` + `encrypts :form`; `belongs_to :patient, :health_unit`, `:notifier_user, :consultation, :addendum, :neighborhood` opcionais; `has_many :events, -> { order(:created_at, :id) }`; `scope :open_notifications`; `overdue?(now = Time.current)`; `inspect` só com `id`, `status`, `version`); `DiseaseNotificationEvent` (`encrypts :reason`; `inspect` sem `reason`); `LocalNotifiableDisease`, `WeeklyNegativeReport`. `CITY_KEYED_TARGETS` + `DiseaseNotification#form`, `DiseaseNotificationEvent#reason`.
- `Membership::ROLES` + `surveillance`; `PRIVILEGED_ROLES` + `surveillance` (comentário: "ADR 0043: lê nome e doença de notificação compulsória — step-up; o mantenedor não concede").
- `HealthUnitsController`: `create`/`update` aceitam `notifying` (bool; ausente na criação → `kind == "ubs"`; não bool → 422 `invalid_notifying`); `unit_json` devolve `notifying` (V18).
- Helpers (`spec/support/surveillance_helpers.rb`): `surveillance_city!(consultation: true)` (cidade com `clinical_record` e `surveillance`, lista carregada, `cid10_release!`, `sigtap_release!`, `ciap2_release!(ScreeningHelpers::CIAP2.merge("A70" => "Tuberculose", "B90" => "Infecção por HIV/SIDA"))` — o padrão do `care_city!`), `surveillance_officer!` (`staff_with(..., "surveillance")`), `notification!(patient:, disease_code: "dengue", status: "notified", **over)` (pelo comando da Task 7), `surveillance_patient!(n, **over)` (paciente com par validado, pelo `Patients::Resolve`).

- [ ] **Step 1: Teste** (`spec/models/surveillance_schema_spec.rb`):

```ruby
require "rails_helper"

RSpec.describe "Esquema da vigilância (ADR 0043)" do
  before { surveillance_city! }
  let(:patient) { surveillance_patient!(1) }
  let(:unit) { HealthUnit.create!(name: "UBS Vigia #{SecureRandom.hex(2)}", kind: "ubs") }

  def row(**over)
    DiseaseNotification.create!({ patient: patient, disease_code: "dengue", disease_list_version: 1, deadline: "weekly", origin: "manual",
                                  status: "notified", suspected_on: Date.current, due_at: 7.days.from_now, health_unit: unit,
                                  form: {} }.merge(over))
  end

  it "uma aberta por paciente e doença; fechada não conta" do
    first = row
    expect { row }.to raise_error(ActiveRecord::RecordNotUnique, /idx_disease_notifications_one_open/)
    first.update!(status: "closed", classification: "discarded", closed_at: Time.current)
    expect { row }.not_to raise_error
  end

  it "CHECKs: classificação só fechada; número oficial só digitada; IBGE com residente fora" do
    expect { row(status: "investigating", classification: "confirmed") }.to raise_error(ActiveRecord::StatementInvalid, /ck_disease_notifications/)
    expect { row(official_number: "1234567") }.to raise_error(ActiveRecord::StatementInvalid)
    expect { row(resident_elsewhere: true) }.to raise_error(ActiveRecord::StatementInvalid)
  end

  it "triggers: sem DELETE, paciente imutável, digitada não volta, andamento só acréscimo, negativa imutável" do
    n = row
    expect { n.delete }.to raise_error(ActiveRecord::StatementInvalid)
    expect { n.update_columns(patient_id: surveillance_patient!(2).id) }.to raise_error(ActiveRecord::StatementInvalid)
    n.update!(status: "closed", classification: "confirmed", closed_at: Time.current)
    n.update!(status: "entered", official_number: "1234567", entered_at: Time.current)
    expect { n.update_columns(status: "closed", official_number: nil) }.to raise_error(ActiveRecord::StatementInvalid)
    event = DiseaseNotificationEvent.create!(notification_id: n.id, status: "entered", by_user_id: nil, reason: "x")
    expect { event.update_columns(status: "closed") }.to raise_error(ActiveRecord::StatementInvalid)
    neg = WeeklyNegativeReport.create!(health_unit: unit, week: "2026-42", recorded_by_user_id: surveillance_officer!.id)
    expect { neg.delete }.to raise_error(ActiveRecord::StatementInvalid)
  end

  it "papel surveillance aceito pelo banco e privilegiado; unidade de APS ativa nasce notificadora" do
    expect { surveillance_officer! }.not_to raise_error
    expect(Membership::PRIVILEGED_ROLES).to include("surveillance")
    expect(DiseaseNotification.new(form: { "pregnancy" => "5" }).inspect).not_to include("pregnancy")
  end
end
```

  E em `spec/requests/health_units_spec.rb` (existente) um exemplo: `POST /attendance/units` com `kind: "ubs"` devolve `notifying: true`; com `kind: "upa"`, `false`; `POST /attendance/units/:id` com `notifying: "sim"` → 422 `invalid_notifying`.
- [ ] **Step 2: Rodar** — `bin/test api -w mod25 spec/models/surveillance_schema_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** a migração (`replace_check` do papel como a do 22; `up` termina com `execute File.read(city_triggers.sql)`; `down` irreversível), o dump, os modelos, os triggers, a cifra, o `Membership`, a unidade, os helpers e os códigos novos no `CID10` de teste; nos dois specs de migração, `restore_exam_roles` passa a usar `AddDiseaseSurveillance::ROLES` (o comentário cita o 25).
- [ ] **Step 4: Rodar** — o spec + `spec/models/membership_roles_spec.rb spec/models/create_analytics_migration_spec.rb spec/models/create_campaigns_migration_spec.rb spec/services/city_schema_spec.rb spec/architecture/city_encrypted_attributes_guard_spec.rb spec/requests/health_units_spec.rb` → PASS.
- [ ] **Step 5: Commit** — `feat: add the disease surveillance city schema and the surveillance role`.

---

### Task 6: Doenças da cidade — lista vigente, correspondência e doenças locais

**Files:**
- Create: `app/services/surveillance/{diseases,matcher}.rb`, `app/commands/surveillance/save_local_disease.rb`, `app/controllers/surveillance/{diseases,local_diseases}_controller.rb`
- Modify: `config/routes.rb`, `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`
- Test: `spec/services/surveillance/matcher_spec.rb`, `spec/requests/surveillance/local_diseases_spec.rb`

**Interfaces:**
- Produces: `Surveillance::Disease = Data.define(:code, :name, :deadline, :cid10, :ciap2, :repeat_after_days, :sinan_form, :local, :list_version)` com `#to_json_hash` (forma `<disease>` do contrato §2); `Surveillance::Diseases.all(today: Time.zone.today) -> [Disease]` (versão vigente + locais ativas, por nome); `Diseases.find(code, today:) -> Disease | nil`; `Surveillance::Matcher.disease_for(terminology, code, diseases) -> Disease | nil` (Desvio 1); `SaveLocalDisease.call(attrs:, by:, disease: nil) -> Result` (`fail(:invalid_disease, details: { path: })`, `fail(:duplicate_code)` quando o código existe ou qualquer CID/CIAP colide com a plataforma vigente ou com outra local — `path` do código que colide); evento `local_disease.changed` (`local_disease_id`, `user_id`).
- Rotas (escopo `/surveillance`, módulo `surveillance`): `get "diseases"` (`surveillance` ou `health_professional`, sem step-up), `get "admin/local_diseases"`, `post "admin/local_diseases"`, `patch "admin/local_diseases/:id"` (`surveillance`; escrita com `require_step_up!`). Corpo da escrita: `{ code, name, deadline, cid10, ciap2, repeat_after_days, active }` (`code` só no POST).

- [ ] **Step 1: Testes**

```ruby
# spec/services/surveillance/matcher_spec.rb
require "rails_helper"

RSpec.describe Surveillance::Matcher do
  let(:dengue) { Surveillance::Disease.new(code: "dengue", name: "Dengue", deadline: "weekly", cid10: %w[A90], ciap2: [], repeat_after_days: 30, sinan_form: nil, local: false, list_version: 1) }
  let(:tb) { Surveillance::Disease.new(code: "tuberculose", name: "Tuberculose", deadline: "weekly", cid10: %w[A15 A16], ciap2: %w[A70], repeat_after_days: 270, sinan_form: nil, local: false, list_version: 1) }

  it "4 caracteres exato; 3 caracteres cobre a categoria; CIAP-2 exato; terminologia separada" do
    list = [ dengue, tb ]
    expect(described_class.disease_for("cid10", "A90", list)).to eq(dengue)
    expect(described_class.disease_for("cid10", "A169", list)).to eq(tb)
    expect(described_class.disease_for("cid10", "A16", list)).to eq(tb)
    expect(described_class.disease_for("cid10", "A91", list)).to be_nil
    expect(described_class.disease_for("ciap2", "A70", list)).to eq(tb)
    expect(described_class.disease_for("ciap2", "A90", list)).to be_nil
  end
end
```

  `spec/requests/surveillance/local_diseases_spec.rb` (`type: :request`): vigilante com step-up cria `local_acidente_escorpionico_pr` (201, forma `<disease>` com `local: true`, `list_version: nil`) e aparece em `GET /surveillance/diseases` junto com `dengue`; sem step-up → 401 `mfa_required`; código sem prefixo → 422 `invalid_disease` `path: "code"`; CID `A90` (da plataforma) → 409 `duplicate_code` `path: "cid10.0"`; mesmo código → 409; profissional de saúde lê `diseases` e recebe 403 em `admin/local_diseases`; interruptor desligado → 403 `feature_disabled`; o evento `local_disease.changed` só com ids.
- [ ] **Step 2: Rodar** → FAIL. **Step 3: Implementar** (a doença local passa pela mesma validação de formato da `DiseaseList.validate`). **Step 4: Rodar** → PASS.
- [ ] **Step 5: Commit** — `feat: serve the city's notifiable diseases and local additions`.

---

### Task 7: Criação de notificação — duplicidade, prazo, bairro e ficha pré-preenchida

**Files:**
- Create: `app/commands/surveillance/create_notification.rb`, `app/services/surveillance/{duplicates,neighborhood,form_prefill}.rb`
- Modify: `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`
- Test: `spec/commands/surveillance/create_notification_spec.rb`, `spec/services/surveillance/{duplicates,neighborhood,form_prefill}_spec.rb`

**Interfaces:**
- Consumes: `Surveillance::Diseases.find`, `Catalog` (Tasks 0, 6).
- Produces:
  - `Surveillance::Duplicates.blocking(patient_id:, disease:, today:) -> DiseaseNotification | nil` (Desvio 4).
  - `Surveillance::Neighborhood.for(patient) -> neighborhood_id | nil` (Desvio 7).
  - `Surveillance::FormPrefill.for(patient) -> Hash` (só chaves do `sinan-form.yml`; `race_color` por `RACE_TO_SINAN`, `"9"` sem raça; `residence_*` do endereço do paciente; `phone` do par validado mais recente, só dígitos sem `+55`; `occupation_cbo` da última versão do cadastro individual ativo, se houver).
  - `Surveillance::CreateNotification.call(patient:, disease:, origin:, unit:, by: nil, suspected_at:, symptoms_on: nil, consultation: nil, addendum: nil, status: nil, resident_elsewhere: false, residence_ibge: nil) -> Result` — `ok(notification:)`, `fail(:already_notified, details: { notification_id: })`, `fail(:invalid_symptoms_on)` (depois da suspeita), `fail(:invalid_residence)`. Status inicial: `safety_net` → `possible`; os outros → `notified`. Grava o evento `notified|possible` em `disease_notification_events` e publica `disease_notification.created` (`notification_id`, `immediate: bool`, `origin`). Travas: Desvio 13.
  - Binding: `DomainEvents.bind "disease_notification.created", to: [ Surveillance::AlertJob ]` (o job nasce na Task 13; até lá, a linha entra com `to: []` e a Task 13 troca).

- [ ] **Step 1: Teste**

```ruby
# spec/commands/surveillance/create_notification_spec.rb
require "rails_helper"

RSpec.describe Surveillance::CreateNotification do
  include ActiveSupport::Testing::TimeHelpers
  before { surveillance_city! }
  let(:patient) { surveillance_patient!(3) }
  let(:unit) { HealthUnit.create!(name: "UBS Norte #{SecureRandom.hex(2)}", kind: "ubs", cnes: "7654321") }
  let(:dengue) { Surveillance::Diseases.find("dengue", today: Time.zone.today) }
  let(:officer) { surveillance_officer! }

  def create(**over)
    described_class.call(**{ patient: patient, disease: dengue, origin: "manual", unit: unit, by: officer, suspected_at: Time.current }.merge(over))
  end

  it "prazo semanal e imediato a partir da suspeita; CNES congelado; evento só com ids" do
    n = create.payload[:notification]
    expect(n.due_at).to be_within(1.second).of(7.days.from_now)
    expect([ n.status, n.unit_cnes, n.notifier_user_id ]).to eq([ "notified", "7654321", officer.id ])
    event = DomainEvent.where(name: "disease_notification.created").last
    expect(event.payload).to eq("notification_id" => n.id, "immediate" => false, "origin" => "manual")
  end

  it "aberta bloqueia; fechada há menos que a janela bloqueia; depois da janela e dismissed liberam" do
    first = create.payload[:notification]
    expect(create.reason).to eq(:already_notified)
    expect(create.details).to eq(notification_id: first.id)
    first.update!(status: "closed", classification: "confirmed", closed_at: Time.current)
    expect(create.reason).to eq(:already_notified)
    travel_to(31.days.from_now) { expect(create).to be_ok }
  end

  it "possível descartada não bloqueia; rede de segurança nasce possible sem notificador" do
    possible = create(origin: "safety_net", by: nil).payload[:notification]
    expect([ possible.status, possible.notifier_user_id ]).to eq([ "possible", nil ])
    possible.update!(status: "dismissed", dismissed_at: Time.current)
    expect(create).to be_ok
  end

  it "raça/cor 03 do paciente nasce 4 na ficha; sintomas depois da suspeita recusados" do
    patient.update!(race_color: "03", address_street: "Rua das Acácias", address_zip: "81020000")
    n = create.payload[:notification]
    expect(n.form).to include("race_color" => "4", "residence_street" => "Rua das Acácias", "residence_zip" => "81020000")
    expect(create(symptoms_on: Date.tomorrow, disease: Surveillance::Diseases.find("tuberculose", today: Date.current)).reason)
      .to eq(:invalid_symptoms_on)
  end
end
```

  `neighborhood_spec`: domicílio do cadastro ativo vence o bairro do par; sem cadastro, o bairro do par validado mais recente; sem nenhum, nil (helpers do 24a `registry_registration!`/`registry_household!`). `duplicates_spec`: janela em dias no fuso da cidade (fechada às 23h de ontem conta como ontem).
- [ ] **Step 2: Rodar** → FAIL. **Step 3: Implementar.** **Step 4: Rodar** → PASS.
- [ ] **Step 5: Commit** — `feat: create disease notifications with the duplicate window`.

---

### Task 8: Na consulta — sugestão e "Notificar suspeita"

**Files:**
- Create: `app/controllers/surveillance/consultation_suggestions_controller.rb`, `app/commands/surveillance/notify_from_consultation.rb`, `app/controllers/surveillance/notifications_controller.rb` (só `create`, ramo da consulta), `app/services/surveillance/json.rb` (forma `<notification>`)
- Modify: `config/routes.rb`
- Test: `spec/requests/surveillance/consultation_spec.rb`

**Interfaces:**
- Consumes: `CreateNotification`, `Matcher`, `Duplicates` (Tasks 6–7).
- Produces:
  - `GET /attendance/consultations/:id/notification_suggestions` (no escopo `/attendance`, junto das rotas da consulta — V5) → `{ suggestions: [ { disease: <disease>, matched_code } ] }`: autora da consulta (rascunho ou finalizada; senão 403 `not_author`), `health_professional`, `Gate.consultation_usable?` (senão 403 `feature_disabled` com `feature: "surveillance"`); itens `add` (Desvio 2) com doença, sem `Duplicates.blocking`; uma por doença; **nunca** trilha nem escrita.
  - `Surveillance::NotifyFromConsultation.call(consultation:, disease_code:, symptoms_on:, by:) -> Result` — autora; vínculo da consulta com CBO começando por `NOTIFIER_CBO_PREFIXES` (senão `cbo_not_allowed`); consulta não descartada; doença da lista (senão `invalid_disease`); unidade = `consultation.attendance.health_unit`; `suspected_at: Time.current`; `origin: "consultation"`. Vale para **qualquer** doença da lista (o "Notificar suspeita" à mão), não só a sugerida.
  - `POST /surveillance/notifications` com `consultation_id` → 201 `<notification>` (sem `form`? **com** `form` e `events`: é a forma nominal; trilha por pessoa); 409 `already_notified` com `notification_id` só se a pessoa que chama é `surveillance` (para o médico, sem o id — V8); nunca toca a consulta (finalização segue igual).
  - `Surveillance::Json.notification(n, full: true) -> Hash` (forma do contrato §2 + V5; `patient` por `Patients::IdentifierJson` + `display_name`; `overdue` por `n.overdue?`), `Json.list_item(n)` (sem `form` e `events`).

- [ ] **Step 1: Teste** (`spec/requests/surveillance/consultation_spec.rb`, `type: :request`):

```ruby
require "rails_helper"

RSpec.describe "Vigilância na consulta", type: :request do
  before { surveillance_city! }
  let(:unit) { HealthUnit.create!(name: "UBS Consulta #{SecureRandom.hex(2)}", kind: "ubs") }
  let(:doctor) { doctor!(unit) }
  let(:citizen) { verified_citizen!(41) }
  let(:dengue_add) { [ { "terminology" => "cid10", "code" => "A90", "action" => "add" } ] }

  def draft!(problems, who: citizen)
    c = started_consultation!(unit: unit, doctor: doctor, citizen: who)
    Consultations::SaveDraft.call(consultation: c, params: draft_body("evaluated_problems" => problems), by: doctor)
    c.reload
  end

  it "rascunho com A90 add sugere dengue sem trilha; notificar devolve 201 e a sugestão some" do
    c = draft!(dengue_add)
    sign_in_as(doctor)
    trail = DomainEvent.where(name: "clinical_record.viewed").count
    get "/attendance/consultations/#{c.id}/notification_suggestions"
    expect(json_body["suggestions"].map { |s| [ s.dig("disease", "code"), s["matched_code"] ] }).to eq([ %w[dengue A90] ])
    expect(DomainEvent.where(name: "clinical_record.viewed").count).to eq(trail)
    post "/surveillance/notifications", params: { consultation_id: c.id, disease_code: "dengue", symptoms_on: Date.yesterday.iso8601 }.to_json,
                                        headers: { "Content-Type" => "application/json" }
    expect(response).to have_http_status(:created)
    expect(json_body).to include("origin" => "consultation", "status" => "notified", "symptoms_on" => Date.yesterday.iso8601)
    get "/attendance/consultations/#{c.id}/notification_suggestions"
    expect(json_body["suggestions"]).to eq([])
    post "/surveillance/notifications", params: { consultation_id: c.id, disease_code: "dengue" }.to_json, headers: { "Content-Type" => "application/json" }
    expect(response).to have_http_status(:conflict)
    expect(json_body).to eq("error" => "already_notified")
  end

  it "sem item add não sugere; outra autora 403; surveillance desligado 403 feature_disabled; finalizar sem notificar segue" do
    c = draft!([ { "terminology" => "ciap2", "code" => "T90", "action" => "add" } ])
    sign_in_as(doctor)
    get "/attendance/consultations/#{c.id}/notification_suggestions"
    expect(json_body["suggestions"]).to eq([])
    sign_in_as(doctor!(unit))
    get "/attendance/consultations/#{c.id}/notification_suggestions"
    expect(response).to have_http_status(:forbidden)
    Platform::Features.set!(city: Current.city, key: "surveillance", enabled: false, maintainer: ledi_maintainer!)
    sign_in_as(doctor)
    get "/attendance/consultations/#{c.id}/notification_suggestions"
    expect(json_body).to eq("error" => "feature_disabled", "feature" => "surveillance")
    expect(Consultations::Finalize.call(consultation: draft!(dengue_add, who: verified_citizen!(43)), outcome_params: { "outcome" => "discharged" }, by: doctor)).to be_ok
  end

  it "enfermeiro (223505) notifica pelo CIAP-2; vínculo fora de 225*/2235* → cbo_not_allowed" do
    nurse = doctor!(unit, cbo: "223505")
    c = started_consultation!(unit: unit, doctor: nurse, citizen: citizen)
    Consultations::SaveDraft.call(consultation: c, params: draft_body("evaluated_problems" => [ { "terminology" => "ciap2", "code" => "A70", "action" => "add" } ]), by: nurse)
    sign_in_as(nurse)
    get "/attendance/consultations/#{c.id}/notification_suggestions"
    expect(json_body["suggestions"].map { |s| s.dig("disease", "code") }).to eq(%w[tuberculose])
    psychologist = doctor!(unit, cbo: "251510") # o 19a aceita na consulta (tabela do MIAI); a notificação não
    other = started_consultation!(unit: unit, doctor: psychologist, citizen: verified_citizen!(42))
    sign_in_as(psychologist)
    post "/surveillance/notifications", params: { consultation_id: other.id, disease_code: "tuberculose" }.to_json, headers: { "Content-Type" => "application/json" }
    expect(response).to have_http_status(:forbidden)
    expect(json_body).to eq("error" => "cbo_not_allowed")
  end
end
```
- [ ] **Step 2: Rodar** → FAIL. **Step 3: Implementar.** **Step 4: Rodar** — o spec + `spec/requests/consultations_spec.rb` → PASS (a consulta não muda).
- [ ] **Step 5: Commit** — `feat: suggest and record a notification from the consultation`.

---

### Task 9: Rede de segurança — consumidor de `consultation.finalized` e `consultation.addendum_added`

**Files:**
- Create: `app/commands/surveillance/safety_net.rb`, `app/jobs/surveillance/safety_net_job.rb`
- Modify: `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`
- Test: `spec/commands/surveillance/safety_net_spec.rb`, `spec/jobs/surveillance/safety_net_job_spec.rb`

**Interfaces:**
- Consumes: `CreateNotification`, `Matcher` (Tasks 6–7).
- Produces: `Surveillance::SafetyNet.call(consultation:, addendum: nil) -> { created: Integer }` — itens `ConsultationProblem` `action: "add"` daquela consulta (sem adendo) ou daquele adendo; uma `possible` por doença casada sem `Duplicates.blocking`; `suspected_at` = `consultation.finalized_at` ou `addendum.created_at`; unidade = a do atendimento; `already_notified` é silêncio (contado como 0). `Surveillance::SafetyNetJob < ApplicationJob` (`include IdempotentConsumer`, `queue_as :default`, `handle(consultation_id:, addendum_id: nil, **)`; `return unless Gate.enabled?(Current.city)` — Desvio: o consumidor roda na transação da cidade, como o `CareLines::ProblemChangedJob`; a consulta sem `clinical_record` não existe, então não relê o prontuário). Bindings: `"consultation.finalized", to: [ Signatures::RequestJob, Surveillance::SafetyNetJob ]` e o mesmo para `consultation.addendum_added` (comentário "ADR 0043").

- [ ] **Step 1: Teste**

```ruby
# spec/commands/surveillance/safety_net_spec.rb
require "rails_helper"

RSpec.describe Surveillance::SafetyNet do
  before { surveillance_city! }
  let(:unit) { HealthUnit.create!(name: "UBS Rede #{SecureRandom.hex(2)}", kind: "ubs") }
  let(:doctor) { doctor!(unit) }

  def finalized!(n, problems)
    finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(n), "evaluated_problems" => problems)
  end

  it "finalizada com A90 add e sem notificação vira possible; redespacho não duplica" do
    c = finalized!(51, [ { "terminology" => "cid10", "code" => "A90", "action" => "add" } ])
    expect(described_class.call(consultation: c)).to eq(created: 1)
    expect(described_class.call(consultation: c)).to eq(created: 0)
    n = DiseaseNotification.find_by!(consultation_id: c.id)
    expect([ n.status, n.origin, n.notifier_user_id ]).to eq([ "possible", "safety_net", nil ])
    expect(n.due_at).to be_within(1.second).of(c.finalized_at + 7.days)
  end

  it "notificada na consulta não gera possible; adendo que acrescenta gera; adendo que só reavalia não" do
    c = finalized!(52, [ { "terminology" => "ciap2", "code" => "T90", "action" => "add" } ])
    added = Consultations::AddAddendum.call(consultation: c, by: doctor, reason: "Resultado do exame chegou", text: "NS1 reagente.",
                                            changes: { "evaluated_problems" => [ { "terminology" => "cid10", "code" => "A90", "action" => "add" } ] })
    expect(described_class.call(consultation: c, addendum: added.payload[:addendum])).to eq(created: 1)
    again = Consultations::AddAddendum.call(consultation: c, by: doctor, reason: "Reavaliação do quadro", text: "Melhora.",
                                            changes: { "evaluated_problems" => [ { "terminology" => "cid10", "code" => "A90", "action" => "evaluate",
                                                                                   "problem_id" => PatientProblem.find_by!(code: "A90").id } ] })
    expect(described_class.call(consultation: c, addendum: again.payload[:addendum])).to eq(created: 0)
  end
end
```

  `spec/jobs/surveillance/safety_net_job_spec.rb`: o binding existe (`DomainEvents.registry`), o job com interruptor desligado não cria nada, e **a corrida da Review Focus 2**: duas threads — uma chama `NotifyFromConsultation` na consulta finalizada, a outra `SafetyNet.call` — com `Queue#pop(timeout: 5)` como barreira, terminam com exatamente uma notificação aberta e nenhuma exceção (cleanup das linhas comitadas no `after`, como `spec/support/committed_rows_cleanup.rb`).
- [ ] **Step 2: Rodar** → FAIL. **Step 3: Implementar.** **Step 4: Rodar** — os dois + `spec/jobs/signatures/request_job_spec.rb spec/initializers/domain_events_bindings_spec.rb` → PASS.
- [ ] **Step 5: Commit** — `feat: flag possible missed notifications on consultation finalization`.

---

### Task 10: Notificação manual e busca de pessoa da vigilância

**Files:**
- Create: `app/commands/surveillance/create_manual.rb`, `app/controllers/surveillance/{person_lookups,duplicates}_controller.rb`, `app/services/surveillance/trail.rb`
- Modify: `app/controllers/surveillance/notifications_controller.rb` (ramo manual), `config/routes.rb`
- Test: `spec/requests/surveillance/manual_spec.rb`

**Interfaces:**
- Consumes: `Patients::Resolve.call(citizen)`, `Patients::CreateTyped.call(attrs:, by:)`, `Patients::Duplicates.call(full_name:, birth_date:, mother_name:, limit:)`, `Patients::Pairs.of_cpf`, `Patients::IdentifierJson.call` (24a); `CreateNotification` (Task 7).
- Produces:
  - `Surveillance::Trail.person_viewed!(user:, patient_id:, citizen_id: nil)`, `Trail.listed!(user:, notification_ids:)` (Desvio 11; `access: "surveillance"`).
  - `Surveillance::CreateManual.call(params:, by:) -> Result` — exatamente um de `patient_id` | `citizen_id` | `new_patient` (senão 422 `invalid_patient` `path: "patient_id"`); `citizen_id` → `Patients::Resolve` (par não validado → 409 `citizen_not_verified`); `new_patient` → `CreateTyped` (recusas dele com `path: "new_patient.<campo>"`, `cpf_taken`, `cns_taken`); `disease_code` da lista; `suspected_on` ISO, não futura; `symptoms_on` opcional ≤ `suspected_on`; `unit_id` de unidade ativa (qualquer unidade; senão 422 `invalid_unit`); `resident_elsewhere` bool e `residence_ibge` 7 dígitos ≠ IBGE da cidade (`CityProfile.current.ibge_code`) quando verdadeiro (senão 422 `invalid_residence`). Paciente criado e notificação na **mesma transação** (falha na notificação desfaz o paciente).
  - `POST /surveillance/notifications` sem `consultation_id` → ramo manual (`surveillance`; 201 `<notification>` completa; trilha por pessoa).
  - `POST /surveillance/person_lookup` `{ cpf }` e `POST /surveillance/patients/duplicates` `{ full_name, birth_date, mother_name? }` — mesma forma e mesmas recusas do 24a (§14 L3 e §5 do contrato do 24a), papel `surveillance`, **sem** exigir `territory_registry`; 30 por minuto por usuário; query com qualquer chave → 422 `invalid_request`; trilha por pessoa mostrada (V4).

- [ ] **Step 1: Teste** (`spec/requests/surveillance/manual_spec.rb`, `type: :request`): vigilante notifica (a) paciente existente por `patient_id`; (b) par validado por `citizen_id` (cria o paciente; `patient.created`); (c) `new_patient` sem CPF com `no_cpf_reason: "5"` e CNS válido (201; `patient.has_cpf: false`, `cns_masked` com 4 dígitos); (d) residente em outro município com `residence_ibge: "4113700"` (Londrina) → `resident_elsewhere: true`; `residence_ibge` igual ao da cidade → 422 `invalid_residence`; (e) `new_patient` com nome de 2 letras → 422 `invalid_patient` `path: "new_patient.full_name"` e nenhum paciente criado; (f) profissional de saúde sem `surveillance` → 403; (g) `person_lookup` com CPF na query → 422 `invalid_request`, no corpo → `{ patient_id, citizen_id, display_name, has_cpf, cns_masked }` com trilha `access: "surveillance"`; (h) nenhum log da requisição contém o CPF, o nome nem `dengue` (`capture_log`).
- [ ] **Step 2: Rodar** → FAIL. **Step 3: Implementar.** **Step 4: Rodar** — o spec + `spec/commands/patients/create_typed_spec.rb` → PASS.
- [ ] **Step 5: Commit** — `feat: record manual notifications by the surveillance team`.

---

### Task 11: Fila, detalhe, ficha e contadores

**Files:**
- Create: `app/services/surveillance/{queue,cursor,form}.rb`, `app/commands/surveillance/save_form.rb`
- Modify: `app/controllers/surveillance/notifications_controller.rb` (`index`, `show`, `update`, `count`), `config/routes.rb`
- Test: `spec/services/surveillance/form_spec.rb`, `spec/requests/surveillance/queue_spec.rb`

**Interfaces:**
- Consumes: `config/surveillance/sinan-form.yml` (Task 0), `Surveillance::Json`, `Trail` (Tasks 8, 10).
- Produces:
  - `Surveillance::Form.validate(hash) -> Result` (`ok(form:)` normalizado — strings `squish`, vazio vira ausente; ou `fail(:invalid_form, details: { path: "form.<chave>" })`: chave desconhecida, tipo, domínio, tamanho — texto longo recusado, nunca cortado).
  - `Surveillance::SaveForm.call(notification:, form:, version:, by:) -> Result` (`stale_version` se `version` ≠; recusa em `entered`/`dismissed` → 422 `invalid_transition`; grava inteiro — o PATCH manda a ficha toda —, `version += 1`, evento `disease_notification.updated` só com `notification_id`).
  - `Surveillance::Queue.call(filters:, limit:, before:, now:) -> { rows:, next_before: }`; filtros: `status` (um de `STATUSES`; padrão: todos menos `entered` e `dismissed`), `disease` (código), `overdue` (`"true"`), `unit_id` (uuid), `without_official_number` (`"true"` = `status = 'closed'`, o relatório "sem número", V12); inválido → 422 `invalid_filter` com `field`.
  - Rotas: `get "notifications"`, `get "notifications/count"` (literal antes de `:id`), `get "notifications/:id"`, `patch "notifications/:id"` — todas `surveillance`. `count` → `{ possible, notified, investigating, overdue, immediate_open, without_official_number }` sem trilha (`immediate_open` = imediatas em `OPEN_STATUSES`).

- [ ] **Step 1: Testes** — `form_spec`: `{ "pregnancy" => "7" }` → `path: "form.pregnancy"`; `{ "residence_street" => "x" * 61 }` → `path: "form.residence_street"`; chave `cpf` → `path: "form.cpf"`; `{ "phone" => "(41) 99999-0000" }` → normalizado para `"41999990000"`. `queue_spec` (`type: :request`):

```ruby
it "imediata vencida no topo; lista sem form com uma trilha; detalhe com form e trilha por pessoa; count sem trilha" do
  weekly = notification!(patient: surveillance_patient!(61), disease_code: "dengue")
  late = travel_to(2.days.ago) { notification!(patient: surveillance_patient!(62), disease_code: "sarampo") }
  sign_in_as(surveillance_officer!)
  get "/surveillance/notifications"
  expect(json_body["notifications"].map { |n| n["id"] }).to eq([ late.id, weekly.id ])
  expect(json_body["notifications"].first).to include("overdue" => true).and(satisfy { |n| !n.key?("form") })
  expect(DomainEvent.where(name: "clinical_record.viewed").last.payload).to include("access" => "surveillance", "notification_ids" => [ late.id, weekly.id ])
  before_count = DomainEvent.count
  get "/surveillance/notifications/count"
  expect(json_body).to include("notified" => 2, "overdue" => 1, "immediate_open" => 1)
  expect(DomainEvent.count).to eq(before_count)
  get "/surveillance/notifications/#{weekly.id}"
  expect(json_body.keys).to include("form", "events", "version")
  patch "/surveillance/notifications/#{weekly.id}", params: { form: { pregnancy: "5" }, version: 1 }.to_json, headers: { "Content-Type" => "application/json" }
  expect(json_body["version"]).to eq(2)
  patch "/surveillance/notifications/#{weekly.id}", params: { form: {}, version: 1 }.to_json, headers: { "Content-Type" => "application/json" }
  expect(response).to have_http_status(:conflict)
end
```

  Mais: paginação com `next_before` estável entre duas páginas; id de outra cidade/inexistente → 404; `analyst` → 403; filtro `status=foo` → 422 `invalid_filter` `field: "status"`.
- [ ] **Step 2: Rodar** → FAIL. **Step 3: Implementar.** **Step 4: Rodar** → PASS.
- [ ] **Step 5: Commit** — `feat: list and complete disease notifications in the surveillance queue`.

---

### Task 12: Andamento — transições, encerramento, número oficial e descarte da possível

**Files:**
- Create: `app/commands/surveillance/{transition,dismiss}.rb`, `app/controllers/surveillance/transitions_controller.rb`
- Modify: `config/routes.rb`, `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`
- Test: `spec/requests/surveillance/transitions_spec.rb`

**Interfaces:**
- Produces:
  - `Surveillance::Transition.call(notification:, to:, version:, by:, classification: nil, official_number: nil, reason: nil) -> Result` — `TRANSITIONS`; fora → `invalid_transition`; `closed` sem classificação válida → `classification_required`; `entered` sem número válido (`OFFICIAL_NUMBER`) → `official_number_required`; `reason` ≤ `REASON_MAX` (senão 422 `invalid_reason` `field: "reason"`); `possible → notified` grava `notifier_user_id = by`; `closed_at`/`entered_at`; evento imutável com `by_user_id`, `classification`, `reason`; `version += 1`; `disease_notification.transitioned` (`notification_id`, `user_id`).
  - `Surveillance::Dismiss.call(notification:, reason:, version:, by:) -> Result` — só `possible`; `reason` obrigatório 3..500; `dismissed_at`; `disease_notification.dismissed`.
  - Rotas: `post "notifications/:id/transition"`, `post "notifications/:id/dismiss"` (`surveillance`).

- [ ] **Step 1: Teste** (`type: :request`): caminho completo `notified → investigating → closed (confirmed) → entered ("1234567")` com 4 eventos na ordem e autor; `notified → closed` → 422 `invalid_transition`; `investigating → closed` sem classificação → 422 `classification_required`; `closed → entered` com `"12 34"` → 422 `official_number_required`; `version` velha → 409 `stale_version`; `possible → notified` grava o notificador; `dismiss` em `notified` → 422 `invalid_transition`, em `possible` com motivo → `dismissed`, e o motivo não aparece em `DomainEvent.last.payload` nem no log; `entered` some da fila padrão e aparece com `status=entered`; `closed` sem número aparece em `without_official_number=true`.
- [ ] **Step 2: Rodar** → FAIL. **Step 3: Implementar.** **Step 4: Rodar** → PASS.
- [ ] **Step 5: Commit** — `feat: move notifications through investigation, closure and official entry`.

---

### Task 13: Alertas por e-mail — imediata, vencidas e lembrete da negativa

**Files:**
- Create: `app/mailers/surveillance_mailer.rb`, `app/views/surveillance_mailer/{immediate,overdue,negatives}.{text,html}.erb`, `app/jobs/surveillance/{alert_job,mail_job,overdue_job,negatives_reminder_job}.rb`, `app/services/surveillance/recipients.rb`
- Modify: `config/initializers/domain_events.rb` (`disease_notification.created` → `[ Surveillance::AlertJob ]`), `spec/initializers/domain_events_bindings_spec.rb`, `config/recurring.yml`
- Test: `spec/jobs/surveillance/{mail_job,overdue_job,negatives_reminder_job}_spec.rb`

**Interfaces:**
- Produces:
  - `Surveillance::Recipients.emails -> [String]` (membros `surveillance` ativos, usuário sem `deactivated_at`).
  - `Surveillance::AlertJob` (`IdempotentConsumer`; `handle(notification_id:, immediate:, **)`; só `immediate == true` e `Gate.enabled?`; enfileira `MailJob.perform_later(city_slug: Current.city.slug, kind: "immediate", ref: notification_id)` — nada de SMTP no consumidor).
  - `Surveillance::MailJob < ApplicationJob` (`include CityScopedJob`, `queue_as :urgent`, `retry_on` os mesmos erros transitórios do `DispatchMunicipalityAlertJob`; `perform(city_slug:, kind:, ref:)`; dedup `ProcessedEvent(consumer: "surveillance_mail", event_id: "#{kind}:#{ref}")`; sem destinatário → log com contagem 0 e sem erro — o painel ainda mostra o alerta). `kind`: `immediate` (`ref` = id da notificação; lê doença e unidade), `overdue` (`ref` = data ISO; lê só contagens), `negatives` (`ref` = semana `YYYY-WW`; lê os nomes das unidades pendentes).
  - `SurveillanceMailer.immediate(to:, disease_label:, unit_label:, possible:)` — assunto `"[rota-saúde] Nova notificação imediata: <doença>, <unidade>"`; corpo: doença, unidade, "possível notificação não feita — confirmar na fila" quando `possible`, e "abra a fila da vigilância no painel da cidade" (sem link com id). `overdue(to:, count:, immediate_count:)`, `negatives(to:, week:, unit_labels:)`.
  - `Surveillance::OverdueJob` (`prepend EachCityJob`, `Gate.usable?`; contagem > 0 → `MailJob` `overdue`), `Surveillance::NegativesReminderJob` (semana anterior; unidades `notifying` ativas sem negativa nem notificação fora de `possible`/`dismissed` na semana → `MailJob` `negatives`; nenhuma → nada).

- [ ] **Step 1: Teste** — `mail_job_spec`: notificação imediata de sarampo de um paciente com nome "Maria Aparecida da Silva", CPF e bairro → `ActionMailer::Base.deliveries.last` tem `"Sarampo"` e o nome da unidade, e **não** contém o nome, o CPF, o CNS, o bairro, a data de sintomas nem o id da notificação (assunto, texto e HTML); dois membros `surveillance` recebem (um desativado não); rodar o job duas vezes entrega uma vez; notificação semanal não enfileira nada (`AlertJob`). `overdue_job_spec`: uma imediata vencida → um e-mail com "1"; nenhuma → nenhum e-mail. `negatives_reminder_job_spec` (com `travel_to` numa segunda): unidade notificadora sem nada na semana anterior aparece; com negativa ou com caso, não; unidade `notifying: false`, não.
- [ ] **Step 2: Rodar** → FAIL. **Step 3: Implementar** (e as duas entradas do `recurring.yml`, Valores fixados item 5). **Step 4: Rodar** — os três + `spec/initializers/domain_events_bindings_spec.rb` → PASS.
- [ ] **Step 5: Commit** — `feat: alert the surveillance team by e-mail without patient data`.

---

### Task 14: Notificação negativa semanal

**Files:**
- Create: `app/commands/surveillance/record_negative.rb`, `app/controllers/surveillance/weekly_negatives_controller.rb`
- Modify: `config/routes.rb`, `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`
- Test: `spec/requests/surveillance/weekly_negatives_spec.rb`

**Interfaces:**
- Consumes: `EpiWeek` (Task 2).
- Produces: `GET /surveillance/weekly_negatives?week=YYYY-WW` (padrão: a semana anterior à corrente) → `{ week, units: [ { unit: { id, cnes, name }, status: "with_cases"|"negative"|"pending" } ] }` (unidades `notifying` ativas, por nome; `with_cases` vence `negative`); `POST /surveillance/weekly_negatives` `{ unit_id, week }` → 201 `{ week, unit, status: "negative" }`; 409 `unit_has_notifications` (notificação da unidade com `suspected_on` na semana, fora de `possible`/`dismissed`), 409 `already_recorded` (índice único), 422 `future_week` (sábado da semana depois de hoje no fuso da cidade), 422 `invalid_week` (`EpiWeek.parse` nil), 422 `invalid_unit` (inexistente, inativa ou não notificadora); evento `weekly_negative.recorded` (`weekly_negative_report_id`, `user_id`). Só `surveillance`.

- [ ] **Step 1: Teste** (`type: :request`, `travel_to Time.zone.local(2026, 1, 5, 10)` — segunda da SE 01/2026): `2025-53` aceita para a UBS sem casos (201) e a segunda vez dá 409 `already_recorded`; `2026-01` (corrente, sábado 10/01 no futuro) → 422 `future_week`; `2026-53` → 422 `invalid_week`; unidade com notificação de `suspected_on` `2026-01-02` → 409 `unit_has_notifications` para `2025-53`; o GET de `2025-53` mostra `negative`, `with_cases` e `pending` nas três unidades.
- [ ] **Step 2: Rodar** → FAIL. **Step 3: Implementar.** **Step 4: Rodar** → PASS.
- [ ] **Step 5: Commit** — `feat: record weekly negative notifications per unit`.

---

### Task 15: Saídas — ficha em PDF no formato do Sinan e CSV por semana

**Files:**
- Create: `app/services/surveillance/{pdf,csv}.rb`, `app/controllers/surveillance/exports_controller.rb`
- Modify: `config/routes.rb`
- Test: `spec/services/surveillance/{pdf,csv}_spec.rb`, `spec/requests/surveillance/exports_spec.rb`

**Interfaces:**
- Consumes: `Consultations::Print.document(footer, title:)`, `Consultations::Print.safe(text)` (Prawn do 19c); `sinan-form.yml`; `CityProfile.current` (UF, IBGE, nome do município).
- Produces:
  - `Surveillance::Pdf.call(notification) -> String` — título "Ficha de Notificação Individual (Sinan) — para digitação"; blocos e números de campo da NINDIV (1–30 + notificador), cada campo com o número e o rótulo da ficha; códigos do Sinan **e** a descrição (`4 — Parda`); data `dd/mm/aaaa`; idade calculada na data da notificação; texto longo quebra linha, nunca sai da página; rodapé "Gerado pelo Rota Saúde — não substitui o registro no Sinan" e "Página n de N".
  - `Surveillance::Csv.call(week:) -> String` (UTF-8 com BOM, `;`, cabeçalho fixo: `semana;notificacao_id;data_notificacao;data_primeiros_sintomas;doenca_codigo;doenca;cid10_ficha;prazo;situacao;classificacao;numero_oficial;unidade_cnes;unidade;paciente;cns_final;sexo;nascimento;bairro;residente_outro_municipio;ibge_residencia`; uma linha por notificação com `suspected_on` na semana, fora de `possible`/`dismissed`; **sem CPF**; CNS só os 4 últimos dígitos).
  - Rotas: `get "notifications/:id/form.pdf"` (`format: false`, `send_data`, `Cache-Control: no-store`, trilha por pessoa), `get "exports/week.csv"` (`week` obrigatório, inválido → 422 `invalid_week`; `no-store`; trilha `listed!` com os ids). Só `surveillance`.

- [ ] **Step 1: Testes** — `pdf_spec` (texto por `PDF::Inspector`, como os specs do 19c/21b): contém "Notificação Individual", o CID da doença, "13 Raça/Cor" com "4 — Parda" para paciente `03`, o CNES da unidade e o nome do notificador; paciente sem CPF mostra o CNS; nome de 70 caracteres não estoura a página (1 página). `csv_spec`: a notificação da semana sai, a da semana seguinte não, `possible` não; nenhuma célula com CPF (`/\d{11}/` não casa fora de `cns_final`); a coluna `cns_final` tem 4 dígitos. `exports_spec`: `Cache-Control` `no-store`, trilha, analyst → 403, `week=2026-53` → 422.
- [ ] **Step 2: Rodar** → FAIL. **Step 3: Implementar.** **Step 4: Rodar** → PASS.
- [ ] **Step 5: Commit** — `feat: export the Sinan-shaped notification form and the weekly CSV`.

---

### Task 16: Painel por bairro e semana epidemiológica

**Files:**
- Create: `app/services/surveillance/panel.rb`, `app/controllers/surveillance/panel_controller.rb`
- Modify: `config/routes.rb`
- Test: `spec/services/surveillance/panel_spec.rb`, `spec/requests/surveillance/panel_spec.rb`

**Interfaces:**
- Consumes: `EpiWeek`, `Admin::SmallCount.small?` (Desvio 10).
- Produces: `Surveillance::Panel.call(from:, to:, disease: nil) -> { weeks: [String], rows: [ { neighborhood: { id, name }, counts: [Integer|nil], signal: [Boolean] } ], suppressed: true }` — conta (Desvio 9) por `COALESCE(symptoms_on, suspected_on)` agrupada por semana (`EpiWeek` em Ruby sobre as datas agrupadas no SQL por dia) e `neighborhood_id`; inclui as 4 semanas antes de `from` só para o sinal; linhas: bairros com pelo menos um caso no intervalo + "não informado" (`id: nil`, `name: "Não informado"`) por último; demais por nome; contagem 1–4 → `nil`; sinal (Valores fixados, item 4). `GET /surveillance/panel?from_week=&to_week=&disease=` — `surveillance` ou `analyst`; padrão as 12 semanas até a anterior à corrente; intervalo > 53 semanas ou `from > to` → 422 `invalid_range`; semana inválida → 422 `invalid_week`; doença desconhecida → 422 `invalid_disease`; sem trilha.

- [ ] **Step 1: Teste**

```ruby
# spec/services/surveillance/panel_spec.rb
require "rails_helper"

RSpec.describe Surveillance::Panel do
  before { surveillance_city! }
  let(:week) { Surveillance::EpiWeek.parse("2026-40") }

  def cases!(count, neighborhood:, on:)
    count.times { |i| panel_case!(neighborhood: neighborhood, symptoms_on: on, suspected_on: on + 1) }
  end

  it "suprime 1–4 (bairro e não informado), sinal só em célula visível acima do dobro da média das 4 anteriores" do
    centro = Neighborhood.create!(name: "Centro", source: "manual")
    boqueirao = Neighborhood.create!(name: "Boqueirão", source: "manual")
    (36..39).each { |w| cases!(2, neighborhood: centro, on: Surveillance::EpiWeek.new(year: 2026, week: w).starts_on) }
    cases!(6, neighborhood: centro, on: week.starts_on)
    cases!(3, neighborhood: boqueirao, on: week.starts_on)
    cases!(1, neighborhood: nil, on: week.starts_on)
    out = described_class.call(from: week, to: week)
    rows = out[:rows].to_h { |r| [ r[:neighborhood][:name], r ] }
    expect(out[:weeks]).to eq(%w[2026-40])
    expect(rows["Centro"]).to include(counts: [ 6 ], signal: [ true ])
    expect(rows["Boqueirão"]).to include(counts: [ nil ], signal: [ false ])
    expect(rows["Não informado"]).to include(counts: [ nil ], signal: [ false ])
    expect(out[:rows].last[:neighborhood]).to eq(id: nil, name: "Não informado")
  end

  it "semana pelos primeiros sintomas (virada de ano); possible, dismissed e descartada fora" do
    centro = Neighborhood.create!(name: "Centro", source: "manual")
    cases!(5, neighborhood: centro, on: Date.new(2026, 1, 3))
    panel_case!(neighborhood: centro, symptoms_on: Date.new(2026, 1, 3), suspected_on: Date.new(2026, 1, 5), status: "possible")
    out = described_class.call(from: Surveillance::EpiWeek.parse("2025-53"), to: Surveillance::EpiWeek.parse("2026-01"))
    expect(out[:rows].first[:counts]).to eq([ 5, 0 ])
  end
end
```

  (`panel_case!` em `surveillance_helpers.rb`: cria paciente e notificação pelo `CreateNotification` com o bairro já resolvido e, para `status:`, ajusta pelo `Transition`.) `panel_spec` de requisição: analyst lê 200 sem trilha; profissional de saúde 403; `from_week=2026-01&to_week=2027-02` → 422 `invalid_range`.
- [ ] **Step 2: Rodar** → FAIL. **Step 3: Implementar.** **Step 4: Rodar** → PASS.
- [ ] **Step 5: Commit** — `feat: add the surveillance panel by neighborhood and epidemiological week`.

---

### Task 17: Exclusão LGPD — notificação retida

**Files:**
- Modify: `app/commands/citizens/request_erasure.rb`
- Test: `spec/commands/citizens/surveillance_erasure_spec.rb`

**Interfaces:**
- Produces: `Citizens::RequestErasure.examined_patients?(ids)` também verdadeiro com `DiseaseNotification.where(patient_id: ids).exists?` (qualquer situação, `dismissed` inclusive — a possível descartada também é registro da vigilância; comentário "ADR 0043: obrigação legal").

- [ ] **Step 1: Teste** — pedido de quem tem notificação (de qualquer origem, `dismissed` inclusive) nasce `retained`; a reconferência do `Citizens::Erase` também retém; sem notificação, `pending`.
- [ ] **Step 2: Rodar** → FAIL. **Step 3: Implementar.** **Step 4: Rodar** — o spec + `spec/commands/citizens/erase_spec.rb spec/commands/citizens/request_erasure_spec.rb spec/commands/citizens/territory_registry_erasure_spec.rb` → PASS.
- [ ] **Step 5: Commit** — `feat: retain disease notifications on LGPD erasure`.

---

### Task 18: Semente de dev realista (Curitiba) para a prova no navegador

**Files:**
- Create: `lib/surveillance_crew.rb`, `spec/lib/surveillance_crew_spec.rb`
- Modify: `db/seeds.rb` (plataforma: `Surveillance::DiseaseList::Load.call` com `puts`; cidade: bloco depois do `TerritoryRegistryCrew`)

**Interfaces:**
- Consumes: `CareLinesCrew::UNIT`, `CareLinesCrew::PEOPLE`, `TerritoryRegistryCrew` (pessoas com domicílio em bairros de Curitiba), `SignatureCrew.ensure_totp`/`otpauth_uri`, `ClinicalTerms.find`, comandos das Tasks 6–14 (caminho real; o histórico de semanas passadas por `travel_to` em volta do comando, como o `TerritoryRegistryCrew`).
- Produces: `SurveillanceCrew.seed_current_city(slug:, ddd:, password:) -> { switch:, notifications:, negatives:, users: [{ email:, role:, otpauth_uri: }] }` — só Curitiba; `guard_environment!` (nunca produção/staging); idempotente; liga `surveillance` pelo mantenedor `dev@local`; a vigilante (Valores fixados, item 6); a doença local `local_acidente_escorpionico_pr` (CID `X22`); notificações coerentes com o real: 9 casos de dengue (A90) nas SE 38–43/2026 em três bairros cobertos pela UBS — Centro com 1, 1, 2, 1 e 6 (sinal na SE 42), dois bairros com 1–3 (células suprimidas) —, um sarampo imediato aberto há 30 horas (vencida, alerta), uma tuberculose em investigação com ficha preenchida, uma hanseníase encerrada confirmada e digitada (`official_number` de 7 dígitos), uma sífilis encerrada sem número (relatório "sem número"), uma notificação manual de residente em Londrina (`4113700`), um paciente sem CPF (do 24a) com notificação de tuberculose, uma **possível** pela rede de segurança numa consulta finalizada da médica da UBS com `A90` (só se `ClinicalTerms.find("cid10", "A90")` existe no dev; senão, `warn` e segue); negativas: a UBS Jardim das Flores com caso, uma UBS com negativa da SE 42, uma pendente.

- [ ] **Step 1: Teste** (`spec/lib/surveillance_crew_spec.rb`, padrão do `territory_registry_crew_spec.rb`): roda duas vezes e conta o mesmo; o painel da SE 42 tem o sinal no Centro e células `nil` nos outros dois; `count` bate `{ overdue: 1, immediate_open: 1, without_official_number: 1 }`; o residente fora tem `residence_ibge: "4113700"`; nenhum telefone fora do prefixo `96666`; fora de Curitiba devolve `switch` "desligado".
- [ ] **Step 2: Rodar** — `bin/test api -w mod25 spec/lib/surveillance_crew_spec.rb` → FAIL. **Step 3: Implementar** (e os blocos do `db/seeds.rb`). **Step 4: Rodar** — o spec + `spec/lib/territory_registry_crew_spec.rb spec/lib/dev_maintainer_spec.rb` → PASS. **O banco de dev NÃO é migrado nem semeado aqui** (só na prova, com autorização).
- [ ] **Step 5: Commit** — `chore: seed a realistic surveillance scenario in Curitiba for the browser proof`.

---

### Task 19: Revisão, runbook e suíte completa

**Files:**
- Create (worktree `docs-mod25`, não publicado): `/Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/docs-mod25/operacao/rollout-vigilancia.md`

- [ ] **Step 1: Revisão do branch** contra a spec e o contrato (superpowers:requesting-code-review, um revisor no modelo mais capaz): Review Focus 1–5; `/opt/homebrew/bin/git -C apps/api/.claude/mod25 grep -n "DomainEvents.publish(" -- app/commands/surveillance app/services/surveillance app/controllers/surveillance app/jobs/surveillance` conferido chamada inteira (payload só com ids e booleanos; nunca `disease_code`); `Platform.audit(` idem contra o R18; nenhum `Current.city =`; nenhum `disease`/`symptoms` em `Rails.logger`; `R18`/`domain_events_bindings`/`city_encrypted_attributes_guard`/`maintenance_schema`/`adr_pointers` verdes.

- [ ] **Step 2: Runbook** `operacao/rollout-vigilancia.md` (no padrão dos `rollout-*.md`): pré-requisitos (`CityProfile` com IBGE e UF; unidades com CNES; quem terá o papel `surveillance`, concedido por admin com step-up; e-mail de envio configurado); ordem (imagem do api → `bin/migrate` da imagem — **migração de plataforma e de cidade irreversíveis**; `bin/rails surveillance:load_disease_list` uma vez na plataforma; dashboard; maintenance; ligar `surveillance` por cidade pelo maintenance — a sugestão na consulta só aparece com `clinical_record`); conferências pós-deploy (`SELECT version, effective_from FROM notifiable_disease_versions`; triggers `SELECT tgname FROM pg_trigger WHERE tgname LIKE '%disease_notification%' OR tgname LIKE '%weekly_negative%'`; unidades notificadoras = UBS ativas; um e-mail de teste de imediata em staging chega sem dado do paciente); operação (a lista muda por portaria: publicar versão nova no maintenance com a data de vigência; caxumba aguarda retificação; doença estadual = doença local da cidade); o que **não** faz (não envia ao Sinan; digitação manual com o número de volta); LGPD (notificação retém o cadastro); reversão (desligar o interruptor; nada se apaga; o consumidor fica inerte com o interruptor desligado); gates de go-live (lista conferida pela vigilância da cidade piloto, incluindo `repeat_after_days`; ficha impressa conferida com a digitação no Sinan).

- [ ] **Step 3: Escopo fechado verde** — `bin/test api -w mod25 spec/services/surveillance spec/commands/surveillance spec/requests/surveillance spec/jobs/surveillance spec/requests/maintenance/notifiable_diseases_spec.rb spec/models/surveillance_schema_spec.rb spec/models/membership_roles_spec.rb spec/models/create_analytics_migration_spec.rb spec/models/create_campaigns_migration_spec.rb spec/commands/citizens spec/requests/consultations_spec.rb spec/requests/health_units_spec.rb spec/jobs/signatures spec/lib/surveillance_crew_spec.rb spec/architecture spec/initializers spec/events spec/adr_pointers_spec.rb` → PASS.

- [ ] **Step 4: Pedir a suíte completa** à sessão "Testes" (mudança transversal: migração de cidade e de plataforma, papel novo no CHECK, `spec/support`, consumidor novo de `consultation.finalized`): SendMessage `api · .claude/mod25/feat/mod-25-disease-surveillance · --full · migrações 20261027900001 (plataforma) e 20261027900002 (cidade), papel surveillance e módulo 25`. Antes, `docker compose stop worker` (memória "Suíte com worker parado") e `start` depois. Falha → diagnóstico nesta sessão (a "Testes" só responde o resumo).

- [ ] **Step 5: Commit final** (se a revisão mudou algo) — `fix: address module 25 review findings`; avisar a coordenadora: plano do api concluído, runbook no `docs-mod25` (não publicado), §14 do contrato pendente de publicação, cards F-25.x seguem In Progress até o dashboard e o maintenance.

---

## Self-review

- **Cobertura da spec:** §2 decisão 1 (Tasks 0, 3, 4, 6), 2 (8), 3 (9), 4 (10), 5 (5, 10–12), 6 (12), 7 (7, 13, 14), 8 (15, 11 para o relatório sem número), 9 (16), 10 (7, Desvio 7), 11 (1, 8), 12 (17); decisões do coordenador: enfermeiro e CIAP-2 (0, 6, 8), o que dispara e duplicidade (7, 8, 9), paciente da manual (10), outra cidade (10), negativa e unidade notificadora (5, 13, 14), painel e semana (2, 16), lista com fonte (0), doenças estaduais (6). §3 interruptor (1). §4 dados (3, 5). §6 segurança (Global Constraints, 10, 11, 13, 15). §7 testes (todas; virada de ano 2 e 14; e-mail 13; supressão 16; retenção 17; escopo do papel 6, 10, 11). §8 runbook (19). F-25.1: 1, 3–6; F-25.2: 8; F-25.3: 9; F-25.4: 10, 11; F-25.5: 12; F-25.6: 7, 13, 14; F-25.7: 15; F-25.8: 2, 16.
- **Tipos:** `Surveillance::Disease` (Task 6) usado em 7–9 e 16; `CreateNotification.call(patient:, disease:, origin:, unit:, by:, suspected_at:, …)` com a mesma assinatura em 8, 9, 10 e 18; `Duplicates.blocking(patient_id:, disease:, today:)` em 7–9; `EpiWeek.for/parse` em 14–16; `Trail.person_viewed!/listed!` em 10, 11, 15.
- **Proporção:** corpos só onde a regra não se deduz (algoritmo da semana, forma dos YAML); o resto por assinatura, valores e teste.

## Divergências propostas ao contrato

- **V1 — Semana epidemiológica (§1).** A semana 1 **termina no primeiro sábado de janeiro com pelo menos 4 dias de janeiro** (= contém 4/jan), não "contém o primeiro sábado": com a regra do contrato, `2022-01-01` seria `2022-01`; pelo Sinan é `2021-52`. Anos com 53 semanas (`2025-53`). A confirmar no calendário do Sinan (Task 0).
- **V2 — Código de doença local (§3).** `local_<a-z0-9_>` (até 46); o código da plataforma nunca começa por `local_`. Colisão de **CID/CIAP** com a plataforma vigente ou outra local também é 409 `duplicate_code`, com `path` do código.
- **V3 — Correspondência de código (§2, §4).** Códigos sem ponto (como `patient_problems`); CID-10 de 3 caracteres cobre a categoria; nenhum código em duas doenças; CIAP-2 só quando corresponde a uma doença (`A77` fica de fora). Doença com forma imediata e CID própria é doença própria; sem CID própria, `notes` na `<disease>` (campo novo).
- **V4 — Busca de pessoa da vigilância (§5).** `POST /surveillance/person_lookup` e `POST /surveillance/patients/duplicates`, mesma forma e recusas do 24a, papel `surveillance`, sem `territory_registry` (as rotas do 24a exigem `health_professional` e o interruptor do 24a).
- **V5 — Caminho da sugestão e forma da notificação (§2, §4).** A sugestão mora no escopo da consulta: `GET /attendance/consultations/:id/notification_suggestions` (as rotas da consulta estão sob `/attendance`). `<notification>` ganha `deadline` (congelado), `consultation_id`|null, `closed_at`, `entered_at`, `dismissed_at`; `disease` ganha `list_version`; `notifier` é nulo na `possible` da rede de segurança até virar `notified`. `<disease>` ganha `notes`.
- **V6 — Vencida (§5).** `overdue` = `due_at` passado com `status ∈ { possible, notified }`; `due_at` conta do instante da suspeita (consulta: agora; rede de segurança: finalização/adendo; manual: 00:00 de `suspected_on` no fuso da cidade).
- **V7 — O que o painel, o CSV e a negativa contam (§6–§8).** Painel: semana dos primeiros sintomas (sem ela, da suspeita), fora `possible`, `dismissed` e classificação `discarded`. CSV e negativa: semana da suspeita, fora `possible` e `dismissed`. Sinal só em célula visível. Linha "não informado" com `neighborhood.id: null`, por último.
- **V8 — 409 `already_notified` (§4).** `notification_id` só para quem tem `surveillance` (o médico recebe só o código).
- **V9 — Transições (§5).** Lineares (`possible→notified→investigating→closed→entered`); `notified→closed` não existe. `reason` opcional ≤ 500 (cifrado, nunca em evento); `official_number` `[0-9A-Za-z./-]{1,30}`; `dismiss.reason` obrigatório 3..500. Recusas novas: 422 `invalid_reason`, 422 `invalid_filter` (`field`), 422 `invalid_unit`, 422 `invalid_residence`, 422 `invalid_symptoms_on`, 422 `invalid_week`, 422 `invalid_range`, 422 `invalid_disease`.
- **V10 — Negativa (§6).** `future_week` = semana cujo sábado é depois de hoje (a corrente inclusive); `GET` sem `week` = a semana anterior; `unit` = `{ id, cnes, name }`; só `surveillance`.
- **V11 — Eventos (§12).** Payloads: `disease_notification.created` (`notification_id`, `immediate`, `origin`); `.updated`/`.transitioned`/`.dismissed` (`notification_id`, `user_id`); `weekly_negative.recorded` (`weekly_negative_report_id`, `user_id`); `local_disease.changed` (`local_disease_id`, `user_id`). Nunca `disease_code`. Trilha: `clinical_record.viewed` com `access: "surveillance"` (por pessoa ou `notification_ids`). Plataforma: também `maintenance.notifiable_disease_list.published` (auditoria da mutation).
- **V12 — Relatório sem número (§7).** `GET /surveillance/notifications?without_official_number=true` (= `status=closed`); `count.without_official_number` idem.
- **V13 — Maintenance (§3, §11).** `publishNotifiableDiseaseList(ordinance, effectiveFrom, diseases, code)` com o TOTP em `code` (step-up consumido, como `createMaintenanceToken`) → `{ ok, errors { path message }, version }`; a lista inteira a cada publicação; `notifiableDiseases` devolve as versões com `current` (vigente hoje) e `publishedBy`.
- **V14 — Painel (§8).** Consulta própria com a semana do Sinan (o analytics usa semana ISO); padrão 12 semanas até a anterior à corrente, máximo 53; `invalid_range`/`invalid_week`/`invalid_disease`.
- **V15 — CSV (§7).** Colunas fixas da Task 15 (UTF-8 com BOM, `;`); CNS só os 4 últimos dígitos.
- **V16 — Ficha (§2, §14).** `PATCH` manda a ficha inteira; recusa a primeira chave inválida com `path: "form.<chave>"`; `form` em `entered`/`dismissed` → 422 `invalid_transition`. Pré-preenchimento: raça/cor convertida para o Sinan, endereço do paciente, telefone do par, ocupação do cadastro individual.
- **V17 — Unidade notificadora (§6).** `notifying` vai em `POST /attendance/units` e `POST /attendance/units/:id` (as rotas de unidade são POST) e no JSON da unidade; ausente na criação = `kind == "ubs"`; inválido → 422 `invalid_notifying`.
