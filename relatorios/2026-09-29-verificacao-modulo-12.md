# Dossiê de verificação — Módulo 12 (Campanhas), F-12.1 a F-12.7

**Data:** 2026-09-29
**ADR governante:** `docs/adr/0024.md` (com a seção Revisão de 2026-09-29)
**Spec:** `docs/superpowers/specs/2026-09-29-module-12-campaigns-design.md`
**Módulo:** `docs/modulos/12--campanhas.md` · **Runbook:** `docs/operacao/rollout-campanhas.md`
**Código mergeado (origin/main):** api `4425347` · dashboard `c69a932` · wpda `32e6420` · docs `7c588d6`
**Prova no navegador:** feita com o usuário em 2026-09-29 (construtor com prévia e "menos de 5", envio com step-up, aviso no wpda com CPF mascarado e selo, preferências, SMS no log com texto fixo e link sem id, alerta de SMS sem provedor).

## Decisão do usuário (2026-09-29)

O usuário mandou fechar as lacunas antes de subir e escolheu corrigir já o SMS parado (em vez de deixá-lo como gate de go-live). Fechadas no api (`4425347..476084d`, 8 commits; suíte completa **2760/0**), com revisão independente (opus) "With fixes" → rodada de correção → re-revisão com todos os achados ADDRESSED:

- `f741b4a` — spec do `down`/`up` da migração de campanhas (e o `down` falha com `campaign_manager` existente) — F-12.1, F-12.3.
- `a380424` — `appointment_no_show` ignora os outros 5 status (mutação: 5 cidadãos em vez de 1) — F-12.2.
- `87bc994` — `DispatchJob` espera o `RecipientsFreezeLock` (threads reais; mutação sem `acquire!` → vermelho) — F-12.4.
- `bd59ced` — gateway não configurado às 22h e às 06h vira `unavailable` na hora, com 1 evento — F-12.6.
- `fe83a9d` — provedor que falha no meio do lote publica `campaign.sms_unavailable` uma vez por campanha — F-12.6.
- `a0a53e6` + `5f5366f` — `DueJob` reenfileira SMS parado (`pending` > 10 min; `deferred` na janela), com teto de 48 h e um lote por campanha (trava por campanha + `SKIP LOCKED`, provados com threads) — F-12.6.
- `476084d` — `SKIP LOCKED` segue provado sob a trava; comentários das guardas.

Docs: ADR 0024 "Consequências" corrigido (ligar sem provedor é permitido e avisado) e Revisão com o SMS parado; runbook com o significado de `Unavailable` no backend real e o reenfileiramento de 48 h.

## Resumo

| F-ID | Veredito sugerido | Lacuna |
|---|---|---|
| F-12.1 | **Verified** (lacuna fechada) | spec do `down`/`up` da migração (api `f741b4a`) |
| F-12.2 | **Verified** (lacuna fechada) | teste negativo do status no `appointment_no_show` (api `a380424`) |
| F-12.3 | **Verified** (lacuna fechada) | mesma spec da migração (api `f741b4a`) |
| F-12.4 | **Verified** (lacuna fechada) | trava do `DispatchJob` provada com threads (api `87bc994`) |
| F-12.5 | Verified | — |
| F-12.6 | **Verified** (lacunas fechadas) | `unavailable` fora da janela testado; evento no meio do lote; SMS parado reenfileirado com teto de 48 h (api `bd59ced`, `fe83a9d`, `a0a53e6`, `5f5366f`, `476084d`) |
| F-12.7 | Verified | — |
| Critério de fechamento | **cumprido** | suíte de invariante cobre os 8 itens com mutação; runbook existe; frase do ADR corrigida |

Arquivos de spec do api na corrida única: `spec/commands/campaigns/draft_commands_spec.rb`, `spec/commands/citizens/update_contact_preferences_spec.rb`, `spec/commands/grant_role_spec.rb`, `spec/commands/invite_member_privileged_roles_spec.rb`, `spec/initializers/domain_events_bindings_spec.rb`, `spec/invariants/campaign_invariants_spec.rb`, `spec/jobs/anonymize_revoked_triage_job_spec.rb`, `spec/jobs/campaigns/dispatch_job_spec.rb`, `spec/jobs/campaigns/due_job_concurrency_spec.rb`, `spec/jobs/campaigns/due_job_spec.rb`, `spec/jobs/campaigns/sms_batch_job_spec.rb`, `spec/models/campaign_tables_guard_spec.rb`, `spec/models/membership_roles_spec.rb`, `spec/policies/campaign_policy_spec.rb`, `spec/requests/admin/api/membership_gate_spec.rb`, `spec/requests/campaign_manager_grant_spec.rb`, `spec/requests/campaign_sms_setting_spec.rb`, `spec/requests/campaign_transitions_spec.rb`, `spec/requests/campaigns_spec.rb`, `spec/requests/citizen_api/contact_preferences_spec.rb`, `spec/requests/citizen_api/notices_spec.rb`, `spec/requests/setup_privileged_role_step_up_spec.rb`, `spec/services/campaigns/audience_schema_spec.rb`, `spec/services/campaigns/audience_spec.rb`, `spec/services/campaigns/audience_validation_spec.rb`, `spec/services/campaigns/criteria/care_criteria_spec.rb`, `spec/services/campaigns/criteria/triage_criteria_spec.rb`, `spec/services/campaigns/forget_revoked_recipients_lock_spec.rb`, `spec/services/campaigns/presenter_spec.rb`, `spec/services/campaigns/sms_text_spec.rb`, `spec/services/sms_gateway_spec.rb`

---

## F-12.1 — Papel `campaign_manager` e chave de SMS por cidade

**1. Requisito.** Um papel novo, `campaign_manager`, entra em `Membership::ROLES` e em `PRIVILEGED_ROLES`: conceder pede step-up e o mantenedor não concede (ADR 0024, Decisão "Quem envia" e Consequências; spec §3.5, D5). `city_profile.campaigns_sms_enabled` nasce `false` (ADR, Decisão; spec §3.4). Só o `municipal_admin` liga ou desliga a chave, com step-up. Ligar sem provedor é permitido, e a resposta e a tela avisam (`GET/PUT /campaigns/sms_setting` → `{ enabled, gateway_configured }`; spec §6.1 e §7 "Chave de SMS"). A mudança publica `city.campaigns_sms_toggled` (spec §5.7). No dashboard, o menu "Campanhas" aparece para `campaign_manager` e `municipal_admin`. O `municipal_admin` sem o papel vê só a chave. `campaign_manager` aparece em Equipe entre os papéis concedíveis, com step-up (spec §7). Revisão de 2026-09-29: o `campaign_manager` também lê `/admin/api`.

**2. Código.**
- Migração: `apps/api/.claude/mod12/db/city_migrate/20260929100001_create_campaigns.rb:8-14` (troca do CHECK `ck_memberships_role` para incluir `campaign_manager`, L96-100) e `:16` (`campaigns_sms_enabled` NOT NULL default false); `down` L82-91 (falha de propósito se já houver membership `campaign_manager`, comentário L89).
- Papel: `app/models/membership.rb:7-8` (ROLES) e `:18-21` (PRIVILEGED_ROLES, com a justificativa).
- Step-up genérico de papel privilegiado: `app/controllers/setup_controller.rb:33` (convite), `:79` (conceder), `:96` (revogar), `:153-154` (`privileged_role?`). O mantenedor é recusado para todo papel privilegiado: `app/commands/grant_role.rb:21-26` (allowlist `actor_kind == "user"`) e `app/commands/invite_member.rb:27`.
- Política: `app/policies/campaign_policy.rb:5-15` (`manage?` só `campaign_manager`; `read_sms_setting?` `campaign_manager` ou `municipal_admin`; `write_sms_setting?` só `municipal_admin`).
- Controller da chave: `app/controllers/campaign_sms_settings_controller.rb:10-14` (show), `:16-27` (update: papel → step-up → booleano estrito `invalid_setting` → comando; falha → 409), `:39-41` (`{ enabled, gateway_configured }`).
- Comando: `app/commands/campaigns/set_sms_enabled.rb:8-17` (`CityProfile.lock`, evento só quando muda, com `enabled` e `by_user_id`, sem PII; `city_profile_missing`).
- Leitura da chave: `app/services/campaigns/sms_setting.rb:5-7` (sem `city_profile` vale desligada).
- Rotas: `config/routes.rb:155-156`. Evento declarado: `config/initializers/domain_events.rb:84`.
- Gate do `/admin/api` (Revisão "Leitura dos painéis"): `app/controllers/admin/api/base_controller.rb:42` (qualquer membership ativa).
- Dashboard: `apps/dashboard/.claude/mod12/src/shell/modules.ts:38,72-80` (grupo "Comunicação" para `municipal_admin` ou `campaign_manager`); `src/modules/Campaigns.tsx:20-22,44-56` (o admin sem o papel vê só `SmsSettingPanel` e um texto que aponta para Equipe, e nunca chama `GET /campaigns`); `src/modules/campaigns/SmsSettingPanel.tsx:15-16` (aviso literal da spec), `:52` (aviso quando `!gateway_configured`), `:54-72` (`SensitiveAction requiresStepUp`); `src/lib/api.ts:780,831-837` (cliente); `src/lib/team.ts:78,83` (papel concedível e marcado como privilegiado); `src/modules/Team.tsx:152-154,305-323` (tornar/remover gestor de campanhas com step-up).

**3. Aderência ao ADR 0024.**
- "`city_profile` ganha a chave `campaigns_sms_enabled`, desligada por padrão": ✓ migração L16; `campaign_tables_guard_spec.rb:95`.
- "Um papel novo, `campaign_manager`" / "entra em `Membership::ROLES` e em `PRIVILEGED_ROLES`": ✓ `membership.rb:7-8,20-21`; CHECK do banco na migração L96-100.
- "conceder o papel também [pede step-up] (é privilegiado)": ✓ `setup_controller.rb:79` via `PRIVILEGED_ROLES`; teste específico do `campaign_manager` em `campaign_manager_grant_spec.rb`. Convite e revogação passam pelo mesmo `privileged_role?` (L33, L96).
- "Só o `municipal_admin` liga o SMS da cidade": ✓ `campaign_policy.rb:13-15`; `campaign_manager` leva 403 no PUT (spec L54-56).
- "Cada cidade nasce com o SMS desligado; **ligar exige provedor configurado na plataforma**, e a tela avisa quando não há" (Consequências): **✗ no texto / ✓ na intenção.** O código deixa ligar sem provedor (`set_sms_enabled.rb:2-3`), como manda a spec §6.1 ("ligar com gateway não configurado é permitido"), e a tela avisa (`SmsSettingPanel.tsx:52`). A Revisão ("Sem provedor: SMS vira `unavailable` na hora") pressupõe esse comportamento, mas não corrige a frase das Consequências. É divergência de redação do ADR, não do código.
- Revisão "o `campaign_manager` também lê `/admin/api`": ✓ `base_controller.rb:42`; `spec/requests/admin/api/membership_gate_spec.rb:43` percorre `Membership::ROLES`, e isso inclui o papel novo.
- "Nenhum payload de evento carrega CPF, telefone ou lista de cidadãos": ✓ `city.campaigns_sms_toggled` só leva `enabled` e `by_user_id` (`campaign_sms_setting_spec.rb:38-39`; invariante L129).

**4. Testes.**
- api:
  - `spec/models/membership_roles_spec.rb:21` (papel conhecido e privilegiado).
  - `spec/models/campaign_tables_guard_spec.rb:95` (chave nasce `false`) e `:99` (o banco aceita o papel).
  - `spec/policies/campaign_policy_spec.rb:8` (só `campaign_manager` gerencia; loop em todos os outros ROLES), `:15` (quem lê e quem muda a chave), `:21` (sem usuário: nada).
  - `spec/requests/campaign_manager_grant_spec.rb:14` (sem step-up → 401 `mfa_required`, papel não concedido) e `:22` (com step-up → 201).
  - `spec/requests/campaign_sms_setting_spec.rb:22` (leitura por `campaign_manager` e `municipal_admin`; `viewer` → 403), `:33` (liga com step-up; evento uma vez só, sem PII), `:43` (sem provedor: `gateway_configured: false`), `:49` (sem step-up → 401; `campaign_manager` → 403; `"sim"`/`nil`/`1` → 422 sem mudar), `:66` (409 `city_profile_missing`).
  - `spec/commands/grant_role_spec.rb:45,60` e `spec/commands/invite_member_privileged_roles_spec.rb:17` (mantenedor e ator desconhecido recusados em cada `PRIVILEGED_ROLES`, o que inclui `campaign_manager`).
  - `spec/requests/setup_privileged_role_step_up_spec.rb:84,114` (step-up no convite e na revogação, provado com `protocol_reviewer`).
  - `spec/requests/admin/api/membership_gate_spec.rb:43`.
  - `spec/invariants/campaign_invariants_spec.rb:129` (evento `city.campaigns_sms_toggled` sem PII).
  - `spec/initializers/domain_events_bindings_spec.rb:46`.
- dashboard:
  - `src/modules/campaigns/SmsSettingPanel.test.tsx:26` (sem aviso com provedor), `:34` (aviso literal sem provedor), `:41` (ligar com step-up, aviso mantido), `:55` (janela fechada pede código), `:69` (403), `:78` (409 traduzido), `:88` (falha de leitura).
  - `src/modules/Campaigns.test.tsx:52` (admin sem o papel: só a chave, nunca pede a lista), `:60` (os dois papéis), `:92` (sem papel).
  - `src/shell/modules.test.ts:94` (menu só para `campaign_manager` e `municipal_admin`) e `:104` (gestor não ganha Equipe nem Território).
  - `src/modules/Team.test.tsx:399` (tornar e remover com step-up), `:419` (janela fechada pede código), `:433` (convite como gestor é privilegiado).
  - `src/lib/team.test.ts:67` (privilegiados iguais aos da API) e `:117`.
  - `src/lib/api.campaigns.test.ts:86` (GET/PUT da chave).
  - `src/lib/campaigns.test.ts:202` (`invalid_setting`/`city_profile_missing` traduzidos).

**5. Execução.**
Corrida única do api (os 31 arquivos de spec apontados pelos três grupos, api em `4425347` = origin/main):
```
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec <31 arquivos: união das listas dos três grupos>
```
```
Finished in 23.5 seconds (files took 1.61 seconds to load)
190 examples, 0 failures
```


dashboard (worktree `apps/dashboard/.claude/mod12`, `c69a932`):
```
npx vitest run src/modules/campaigns/SmsSettingPanel.test.tsx src/modules/campaigns/CampaignPanel.test.tsx \
  src/modules/campaigns/SendDialog.test.tsx src/modules/campaigns/CampaignEditor.test.tsx src/modules/Campaigns.test.tsx \
  src/modules/Team.test.tsx src/lib/team.test.ts src/shell/modules.test.ts src/lib/api.campaigns.test.ts \
  src/lib/campaigns.test.ts src/lib/audiencePhrase.test.ts
```
```
 ✓ src/shell/modules.test.ts (12 tests)
 ✓ src/lib/campaigns.test.ts (27 tests)
 ✓ src/lib/team.test.ts (12 tests)
 ✓ src/modules/campaigns/SmsSettingPanel.test.tsx (7 tests)
 ✓ src/modules/campaigns/SendDialog.test.tsx (9 tests)
 ✓ src/modules/Campaigns.test.tsx (8 tests)
 ✓ src/lib/api.campaigns.test.ts (8 tests)
 ✓ src/modules/campaigns/CampaignPanel.test.tsx (12 tests)
 ✓ src/lib/audiencePhrase.test.ts (8 tests)
 ✓ src/modules/Team.test.tsx (32 tests)
 ✓ src/modules/campaigns/CampaignEditor.test.tsx (21 tests)
 Test Files  11 passed (11)
      Tests  156 passed (156)
```
(Uma só corrida serve para F-12.1, F-12.3 e F-12.7.) Prova no navegador feita com o usuário, segundo `modulos/12--campanhas.md`, Histórico ("alerta de SMS sem provedor").

**6. Lacunas.**
- **Nenhum teste roda o `down` da migração** (lacuna de teste). A spec §3 pede migração "só de expansão e reversível". O `down` existe (`20260929100001_create_campaigns.rb:82-91`), mas nenhum spec o executa: o grep por `CreateCampaigns` ou `20260929100001` em `spec/` volta vazio, e o `task-1-report.md` não registra `down`/`up`. É a mesma lacuna que o módulo 11 teve na F-11.1, fechada lá depois com `create_territory_migration_spec.rb`. Vale para a migração inteira, então também para F-12.3.
- **Divergência de redação no ADR 0024.** As Consequências dizem "ligar exige provedor configurado na plataforma", mas o código, a spec §6.1 e a Revisão deixam ligar sem provedor. Sugestão: corrigir a frase na Revisão. Não é lacuna de código.
- Menores (lacuna de teste, baixo risco): revogar e convidar `campaign_manager` com step-up só têm prova na API pelo caminho genérico (`protocol_reviewer`). O dashboard cobre os dois com o papel novo. O ledger do api registra ainda (Task 14) que o caminho de **desligar** a chave não tem teste na API (o `SetSmsEnabled` é simétrico) e que falta um PUT sem sessão ou por `viewer`.

**7. Veredito sugerido:** **Done com lacuna**: o `down` da migração nunca foi exercitado (spec §3, "reversível"). Todo o resto do requisito tem código, teste e prova no navegador. Com a spec de `down`/`up` e a frase do ADR corrigida, vira **Verified**.

---

---

## F-12.2 — Construtor de público: 3 recortes, 7 critérios, mínimo de 5 telefones e prévia

**1. Requisito.** O público é um JSON versionado (`audience` v1: `version`, `geo`, `clinical.all`) que combina **por E** um recorte geográfico (cidade toda, área de uma unidade de referência via `neighborhood_coverages`, ou lista de bairros; ADR 0023) com zero a sete critérios clínicos de tipos fixos. O schema recusa `kind` desconhecido, campo extra, período invertido, `to` no futuro, `neighborhood_ids` vazio ou com mais de 50, e mais de 7 critérios. Períodos são datas inclusivas no fuso da cidade. Triagem chega ao cidadão por `triages.conversation_id → conversations.citizen_id`; triagem sem cidadão não conta. Cidadão sem bairro só entra em `city`. Revogado = conversa mais recente (por `created_at`, desempate por `id`) com consentimento revogado; nunca entra. O mínimo conta **telefones distintos**: `< 5` → prévia `{ below_minimum: true }` sem números, envio recusado (422 `below_minimum`). `GET /campaigns/options` devolve protocolos e faixas de triagens concluídas, bairros e unidades ativos. Dashboard: construtor A (cartões, "e também", contador com debounce de 500 ms, "menos de 5 — ajuste o público" travando o envio, público em frase no modal). Revisão do ADR: recorte com bairro/unidade inativos é recusado (`inactive_or_unknown`) na prévia, criação, edição, envio e agendamento; até 20 faixas de 50 caracteres; até 50 bairros. (ADR 0024 Decisão "Público", "Mínimo de 5", "Consentimento", Invariantes 1–2, Revisão "Recorte ativo" e "Limites"; spec §4.1–§4.4, §6.1, §7, §9.1, §9.3; D3, D4, D8, D10, D11, D12.)

**2. Código.**
- api (worktree `apps/api/.claude/mod12`, `4425347`):
  - Schema: `app/services/campaigns/audience_schema.rb` — limites (L11-14), recortes (L16), 7 kinds com obrigatórios/opcionais (L18-26), raiz com `version` = 1 e chaves exatas (L30-39), recorte (L56-71), `clinical.all` lista e ≤ 7 (L73-85), critério (kind, chaves extras, campos, L87-99), campos por tipo (L101-115), listas vazias/demais/repetidas (L117-127), período futuro/invertido (L130-137).
  - Recorte ativo: `app/services/campaigns/audience_validation.rb:10-28` (bairro inativo ou inexistente por item; unidade inativa; formato antes do banco).
  - Resolução: `app/services/campaigns/audience.rb` — `REVOKED_SQL` com `DISTINCT ON (citizen_id) … ORDER BY citizen_id, created_at DESC, id DESC` (L11-20), interseção por `where(id: …)` por critério e `NOT IN (revogados)` (L26-32), `summary` com `distinct.count(:phone)` sobre a coluna cifrada determinística (L34-37; `citizen.rb:9`), `below_minimum?`/`preview` (L39-46), recortes (L51-59).
  - Critérios: `app/services/campaigns/criteria.rb` (registro L7-19, `period` inclusivo `beginning_of_day..end_of_day` em `Time.zone` L22-25, `triage_citizens` filtrando conversa sem cidadão L29-31) e `criteria/*.rb` (um arquivo por kind; SQL citado na seção 3).
  - Mínimo: `Campaign::MINIMUM_PHONES = 5` (`app/models/campaign.rb:11`); `Campaigns::SendGate.failure_for` (`app/commands/campaigns/send_gate.rb:5-10`) usado por `Send` (L11-12) e `Schedule` (L15-16); no congelamento, `dispatch_job.rb:46-49` (rollback abaixo de 5 — F-12.4).
  - Controller: `app/controllers/campaigns_controller.rb` — `options` (L26-35), `preview` (L37-43: 422 `invalid_audience` com `details`, senão contagens ou `below_minimum`), papel `campaign_manager` em toda rota (L18, L83-85); `Create`/`Update` validam o público (`app/commands/campaigns/create.rb:11-12`, `update.rb:16-21`). Rotas: `config/routes.rb:150-162`.
- dashboard (worktree `apps/dashboard/.claude/mod12`, `c69a932`):
  - Cliente: `src/lib/api.ts:794` (`getCampaignOptions`), `:798` (`previewAudience`).
  - Regras locais: `src/lib/campaigns.ts` — `PREVIEW_DEBOUNCE_MS = 500`, `MAX_CRITERIA = 7`, `MAX_NEIGHBORHOODS = 50` (L18-20), `newCriterion` (L110), `validateCriterion` (L125), `validateGeo` (L137-149), mensagem `below_minimum` (L223).
  - Construtor: `src/modules/campaigns/AudienceBuilder.tsx` — 3 recortes em rádio (L22-26, L63-71), seletor de unidade com unidade inativa visível (L42-43, L72-82), lista de bairros com filtro, contador "N de no máximo 50" e bairro inativo visível e desmarcável (L123-171), cartões com "e também" (L93-99), lista dos 7 tipos (L101-105), trava em 7 (L109, L114); `CriterionCard.tsx` (campos por tipo).
  - Contador: `src/modules/campaigns/useAudiencePreview.ts:17-32` (debounce pela chave JSON; enquanto a chave digitada ≠ a consultada o estado é `loading`, nunca a contagem anterior); `AudienceCounter.tsx` — textos (L4-13; "menos de 5 — ajuste o público" L10; "≈ N pessoas (M telefones)" L11), `previewAllowsSend` só em `ok` (L16-18).
  - Editor: `src/modules/campaigns/CampaignEditor.tsx` — debounce padrão 500 (L55), prévia (L76), envio só com contagem `ok` (L106, L136, L212), aviso "o envio libera quando a contagem mostrar pelo menos 5 telefones" (L221), frase no diálogo (L186).
  - Frase: `src/lib/audiencePhrase.ts:63-67` (recorte + critérios unidos por "e também"; id fora de `options` vira "(bairro inativo)"/"(unidade inativa)", L22-24); `SendDialog.tsx:85` (`below_minimum` devolve ao editor).

**3. Aderência ao ADR 0024 (e spec §4) — um veredito por critério.**

*Formato e schema*
- "O JSON nasce com `clinical: { all: [...] }`, para que grupos 'OU' entrem depois como `any`" — ✓ `audience_schema.rb:76` exige `all` e só `all` (`any` hoje é `unknown_key`; entra sem migração de dado).
- Recusas da §4.1 — ✓ kind desconhecido (`invalid_kind` L91), campo extra (`unknown_key` L52), período invertido (`inverted_period` L134), `to` no futuro (`future_date` L133), `neighborhood_ids` vazio/> 50 (`empty`/`too_many` L119-120), > 7 critérios (`too_many` L81); também `version ≠ 1`, UUID inválido, data inválida, valores fora de `Attendance::OUTCOMES`/`AppointmentRequest::KINDS`, repetidos. Caminho do erro como ponteiro JSON.
- Revisão "Limites: até 20 faixas por critério, 50 caracteres cada; até 50 bairros" — ✓ `MAX_TIERS` (L13), `text?(v, 50)` (L104), `MAX_NEIGHBORHOODS` (L12). Menor do ledger (Task 4, não bloqueante): checagem de repetidos diferencia caixa (`u` e `u.upcase` passam como distintos); `1.0` é aceito como `version` 1.

*Recortes geográficos*
- `city` = todos os cidadãos — ✓ `audience.rb:54`.
- `neighborhoods` = `citizens.neighborhood_id` na lista — ✓ `audience.rb:55`.
- `unit` = bairros cobertos pela unidade em `neighborhood_coverages` — ✓ `audience.rb:57` (subconsulta `NeighborhoodCoverage.where(health_unit_id:).select(:neighborhood_id)`).
- "Cidadão sem bairro declarado só entra no recorte `city`" — ✓ por construção: `neighborhood_id IS NULL` nunca casa `IN (…)`.
- Revisão "Recorte ativo: prévia, criação, edição, envio e agendamento recusam bairro ou unidade do recorte inativos (`inactive_or_unknown`)" — ✓ `audience_validation.rb:15-27`, chamado em `preview` (controller L39), `Create` (L11), `Update` (L17), `SendGate` (L6) para Send e Schedule. Campanha já agendada congelada mesmo com bairro desativado — fora deste F-ID (DispatchJob não revalida; comportamento declarado na Revisão).

*Critérios clínicos (regra da spec §4.1 × SQL real)*
- **`protocol_period`** — spec: "triagem `completed` com esse protocolo, `completed_at` no período". SQL (`criteria/protocol_period.rb:6-7`): `Triage.where(status: "completed", protocol_name:, completed_at: period)` → `triage_citizens`. ✓ idêntico.
- **`triage_tier`** — spec: "triagem `completed` com `tier` na lista, no período". SQL (`triage_tier.rb:7-8`): `status: "completed", tier: tiers, completed_at: period`. ✓ (a spec não diz qual carimbo; o código usa `completed_at`, coerente com `protocol_period`).
- **`triage_incomplete`** — spec: "triagem `aborted_by_timeout` ou `aborted_by_cancellation` criada no período **e** nenhuma triagem `completed` criada depois dela". SQL (`triage_incomplete.rb:6-22`): `status IN (aborted_by_timeout, aborted_by_cancellation)`, `created_at` no período, `conversations.citizen_id IS NOT NULL`, `NOT EXISTS (triagem completed de conversa do mesmo cidadão com later.created_at > triages.created_at)`. ✓ idêntico; `aborted_by_revocation` fica de fora, como a spec.
- **`attendance_outcome`** — spec: "atendimento `closed` com `outcome` na lista (e na unidade, se dada), `closed_at` no período". SQL (`attendance_outcome.rb:7-9`): `status: "closed", outcome: outcomes, closed_at: period` + `health_unit_id` opcional. ✓ idêntico. (A unidade do critério não é conferida como ativa, por decisão: "filtram histórico", `audience_validation.rb:4-5`.)
- **`triaged_not_attended`** — spec: "triagem `completed` no período sem nenhum atendimento com aquele `triage_id`". SQL (`triaged_not_attended.rb:6-8`): `completed_at` no período e `id NOT IN (SELECT triage_id FROM attendances WHERE triage_id IS NOT NULL)`. ✓ idêntico; o `IS NOT NULL` evita a armadilha do `NOT IN` com NULL (atendimento vindo de agendamento tem `triage_id` nulo, `ck_attendances_origin`).
- **`appointment_no_show`** — spec: "agendamento `no_show` com `scheduled_at` no período". SQL (`appointment_no_show.rb:7`): `status: "no_show", scheduled_at: period`. ✓ idêntico.
- **`appointment_request_open`** — spec: "pedido de agendamento `open` agora", `kinds` e `target_unit_id` opcionais. SQL (`appointment_request_open.rb:6-9`). ✓ idêntico; sem período, e o schema não aceita `from`/`to` nele (L25).
- "Bordas de período inclusivas no fuso da cidade" — ✓ `criteria.rb:22-25` (`beginning_of_day..end_of_day` em `Time.zone`). Ressalva herdada: o "fuso da cidade" é o fixo `America/Sao_Paulo` (`config/application.rb:70`; risco aceito api#27, spec §13). `to` no futuro compara com `Time.zone.today` (schema L133), mesmo fuso.
- "Triagem sem cidadão (conversa antiga de WhatsApp) não conta" — ✓ nos quatro critérios de triagem: `triage_citizens` filtra `citizen_id IS NOT NULL` (`criteria.rb:29-31`); `triage_incomplete` filtra no join (L19). Os três critérios de atendimento/agendamento leem `citizen_id NOT NULL` por schema (`city_schema.rb:32,64,94`).

*Combinação, revogados, contagem, mínimo*
- "Combinado **por E**" — ✓ cada critério é um `where(id: subconsulta)` encadeado sobre o recorte (`audience.rb:28-30`); tudo em SQL (spec §4.3 "sem carregar cidadãos em Ruby").
- "Cidadão cuja conversa mais recente teve o consentimento revogado nunca entra no público" (+ D11 "desempate por id") — ✓ `REVOKED_SQL` (`audience.rb:11-20`) com `ORDER BY citizen_id, created_at DESC, id DESC`; aplicado depois dos critérios (L31). A mesma regra existe duplicada em Ruby em `ForgetRevokedRecipients` (ruling PF8, comentário cruzado L9-10).
- "Nenhuma campanha sai com menos de 5 telefones distintos, com ou sem filtro clínico" / D12 "conta telefones distintos" — ✓ `summary` conta `DISTINCT phone` cifrado determinístico (`audience.rb:36`); o mínimo não depende do filtro clínico.
- "A prévia mostra 'menos de 5' e o envio é recusado" — ✓ prévia `{ below_minimum: true }` sem números (`audience.rb:43-46`); envio e agendamento → 422 `below_minimum` (`send_gate.rb:9`, `ERROR_STATUS` controller L14).
- "Se o público encolher entre a prévia e o envio, a campanha falha sem enviar a ninguém" — ✓ `dispatch_job.rb:46-49` (verificado de fato em F-12.4).
- `GET /campaigns/options` (spec §6.1: "protocolos e faixas já vistos em triagens concluídas, bairros ativos, unidades ativas") — ✓ controller L26-35; traz também `outcomes` (lista fixa). Só `campaign_manager` (L18).
- Privacidade: a prévia só devolve contagens e esconde os números abaixo de 5 ✓; nenhuma rota do construtor devolve ids de cidadão ✓; `options` só traz nomes de protocolo/faixa e bairros/unidades ✓. Risco de subtração entre prévias aceito (spec §13).

*Dashboard (construtor A)*
- "Público — 1. Recorte: cidade toda / unidade de referência (seletor) / bairros (seleção múltipla)" — ✓ `AudienceBuilder.tsx:22-26, 72-86`.
- "2. Critérios clínicos: 'Adicionar critério' abre a lista dos 7; cada um vira cartão com os próprios campos e um 'remover'; entre cartões, 'e também'" — ✓ L93-114 + `CriterionCard.tsx`.
- "Contador ao vivo (`POST /campaigns/preview`, debounce de 500 ms): '≈ N pessoas (M telefones)' ou 'menos de 5 — ajuste o público'" — ✓ `useAudiencePreview.ts`, `AudienceCounter.tsx:10-11`, padrão 500 em `CampaignEditor.tsx:55`.
- "Menos de 5" travando o envio — ✓ `previewAllowsSend` só em `ok` (`AudienceCounter.tsx:16-18`; `CampaignEditor.tsx:136,212`); contagem antiga não libera o público novo (estado `loading` enquanto a chave não assenta).
- "Modal com o público em frase ('moradores de Boqueirão e Xaxim que faltaram a um agendamento entre 01/07 e 30/09')" — ✓ `audiencePhrase.ts` (datas com ano: "01/07/2026", diferença só de formato), usado em `CampaignEditor.tsx:186` e `CampaignPanel.tsx:56`.
- "Recusa `below_minimum` volta ao editor com a mensagem" — ✓ `SendDialog.tsx:85`, `campaigns.ts:223`.

**4. Testes.**
- api — schema: `spec/services/campaigns/audience_schema_spec.rb` (aceita 3 recortes e os 7 kinds L16; símbolos e período de um dia hoje L36; não-objeto, versão, chave extra, faltando L43; escopo, UUID, bairros vazios/repetidos/> 50/não-lista L52; > 7, kind desconhecido, campo extra, faltando, valor fora da lista, faixas vazias, protocolo em branco L69; data inválida, futura, invertida L88).
- api — recorte ativo: `spec/services/campaigns/audience_validation_spec.rb` (bairro inativo/inexistente por item, UUID em caixa alta aceito L11; unidade inativa/inexistente L18; formato antes do banco L28).
- api — critérios de triagem: `spec/services/campaigns/criteria/triage_criteria_spec.rb` (período inclusivo L19; `protocol_period` entra, bordas ±1 s, outro protocolo, não concluída L27, sem cidadão L37; `triage_tier` L46, sem cidadão L57; `triage_incomplete` tempo/cancelamento nas bordas, concluída antes entra, concluída depois sai, revogação não conta L66, sem cidadão L83; `triaged_not_attended` bordas, atendida sai L95, sem cidadão L109).
- api — critérios de atendimento/agendamento: `spec/services/campaigns/criteria/care_criteria_spec.rb` (schema × classes em correspondência L18; `attendance_outcome` desfecho, bordas, unidade L26; `appointment_no_show` bordas L40; `appointment_request_open` aberto × `scheduled`, `kinds`, `target_unit_id` L53).
- api — resolução: `spec/services/campaigns/audience_spec.rb` (3 recortes e sem bairro só em `city` L17; recorte + 2 critérios por E L29; revogação recente exclui e antiga não L46; empate de `created_at` decide pelo maior id L54; telefone compartilhado conta uma vez L62; 4 telefones → `below_minimum`, 5 → contagens L70; 5 CPFs num telefone não chegam ao mínimo L78; relação SQL L84).
- api — request: `spec/requests/campaigns_spec.rb` (prévia 4 → `below_minimum` sem números, 5 → contagens L38; público malformado 422 com caminho em prévia e criação, nada gravado L47; bairro desativado recusado em prévia e edição L66; `options` só de triagens concluídas e bairros/unidades ativos L99; 403 `missing_role` para todo papel ≠ `campaign_manager` em `options` e `preview` L121; 401 sem sessão L140); `spec/requests/campaign_transitions_spec.rb` (envio e agendamento com < 5 → 422 `below_minimum` e segue `draft` L57; bairro desativado → `invalid_audience` no envio L69).
- api — comandos: `spec/commands/campaigns/draft_commands_spec.rb` (público ausente/inválido em `Create` L42; símbolos gravados como texto L50; `Update` com público inválido L65).
- api — invariantes: `spec/invariants/campaign_invariants_spec.rb` (mínimo de 5 telefones, com mutação registrada L32; revogado fora do público L44); congelamento com público encolhido `spec/jobs/campaigns/dispatch_job_spec.rb:30`.
- dashboard: `src/modules/campaigns/AudienceBuilder.test.tsx` (começa em cidade L24; unidade L31; bairros com contagem L40; filtro sem acento L50; bairro/unidade inativos visíveis L57/L66; 50 bairros travam L78; lista dos 7 e período padrão L88; "e também" L103; remover L110; independência dos cartões L119; máximo 7 L127); `AudienceCounter.test.tsx` (incompleto não chama API L33; só a última versão vai à API L39; público mudou → "calculando…" e envio travado L53; `below_minimum` e erro L64; textos L78; só `ok` libera L86); `CriterionCard.test.tsx` (campos e recusas por tipo, L20-86); `CampaignEditor.test.tsx` (contador e "menos de 5" L112; envio travado com < 5 L206; travado até a contagem alcançar o público atual L213; `below_minimum` no envio volta ao editor L243; `invalid_audience` no envio L261/L281); `src/lib/audiencePhrase.test.ts` (exemplo da spec L24; recortes L30; bairro inativo L38; uma frase por tipo L45; dia único L63; E L68); `src/lib/campaigns.test.ts` (7 tipos L32; período padrão L39; validações L52-89; `buildAudience` L91-127); `src/lib/api.campaigns.test.ts` (`options` L28, `preview` L34).

**5. Execução.**
- api: `Corrida única do api (os 31 arquivos de spec apontados pelos três grupos, api em `4425347` = origin/main):
```
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec <31 arquivos: união das listas dos três grupos>
```
```
Finished in 23.5 seconds (files took 1.61 seconds to load)
190 examples, 0 failures
```
`
- dashboard (worktree `apps/dashboard/.claude/mod12`, `c69a932`):
```
npx vitest run src/modules/campaigns/AudienceBuilder.test.tsx src/modules/campaigns/AudienceCounter.test.tsx \
  src/modules/campaigns/CriterionCard.test.tsx src/modules/campaigns/CampaignEditor.test.tsx \
  src/modules/campaigns/SendDialog.test.tsx src/lib/campaigns.test.ts src/lib/api.campaigns.test.ts \
  src/lib/audiencePhrase.test.ts
```
```
 ✓ src/lib/campaigns.test.ts (27 tests)
 ✓ src/lib/audiencePhrase.test.ts (8 tests)
 ✓ src/lib/api.campaigns.test.ts (8 tests)
 ✓ src/modules/campaigns/CriterionCard.test.tsx (9 tests)
 ✓ src/modules/campaigns/SendDialog.test.tsx (9 tests)
 ✓ src/modules/campaigns/AudienceBuilder.test.tsx (12 tests)
 ✓ src/modules/campaigns/AudienceCounter.test.tsx (7 tests)
 ✓ src/modules/campaigns/CampaignEditor.test.tsx (21 tests)
 Test Files  8 passed (8)
      Tests  101 passed (101)
```
- Prova no navegador (spec §9.5: bairro + falta → prévia → enviar): não conferida por esta verificação; cabe ao controlador confirmar com o usuário.

**6. Lacunas.**
- **Lacuna de teste — filtro de status do `appointment_no_show`.** Nenhum exemplo cria um agendamento `confirmed`/`checked_in`/`cancelled_by_citizen`/`expired` com `scheduled_at` no período (`care_criteria_spec.rb:40-47`; o único "não entra" é um pedido sem agendamento). Tirar `status: "no_show"` de `appointment_no_show.rb:7` mantém a suíte verde (confirmado no ledger do api, Task 6 minor). O código está certo; falta o teste que o prove.
- **Não é lacuna — filtro `closed` do `attendance_outcome`.** Também sem exemplo negativo (ledger Task 6), mas `ck_attendances_closing` (`db/city_schema.rb`, check de `attendances`) só permite `outcome` não nulo em `closed`: o filtro é redundante com o banco e não tem como falhar.
- **Menor — revogado × recortes/critérios.** A exclusão de revogados só é testada com o recorte `city` (`audience_spec.rb:46,54`; invariante L44); como o `NOT IN` é aplicado depois de qualquer recorte (`audience.rb:31`), o risco é baixo (ledger Task 7).
- **Menor — debounce de 500 ms.** Os testes do contador usam 30/0/200 ms injetados; nada afirma que o padrão é 500 (`CampaignEditor.tsx:55` / `campaigns.ts:18`). O comportamento de debounce (só a última chave vai à API) está provado.
- **Menores de schema (ledger Task 4):** repetidos em `neighborhood_ids` diferenciam caixa (UUID igual em caixa alta passa como distinto; o SQL trata como o mesmo id, sem efeito na contagem); `version: 1.0` aceito.
- **Risco herdado (não lacuna):** "fuso da cidade" é o fuso fixo da aplicação (api#27); subtração entre prévias (spec §13).
- Privacidade: nada encontrado. A prévia só traz contagens e as esconde abaixo de 5; nenhuma rota do construtor devolve cidadão.

**7. Veredito sugerido:** **Done com lacuna** — o filtro `status = no_show` do critério `appointment_no_show` não tem teste que falhe sem ele (lacuna de teste, não de implementação). Os demais critérios (schema, 3 recortes, 7 critérios, bordas no fuso, triagem sem cidadão, E, revogados com desempate por id, telefones distintos, mínimo na prévia e no envio, `options`, construtor, contador, trava e frase) têm código aderente e teste. Fechar com um exemplo de agendamento não-`no_show` no período em `care_criteria_spec.rb` sobe para **Verified**.

---

## F-12.3 — Criar, agendar, desagendar, cancelar e enviar campanha com step-up

**1. Requisito.**
- Estados: `draft`, `scheduled`, `sending`, `sent`, `cancelled`, `failed` (ADR 0024, Decisão).
- Só `draft` é editável; `scheduled` volta a `draft` por "desagendar", com step-up (spec §5.1).
- Enviar, agendar e cancelar pedem step-up (ADR, "Quem envia"); desagendar também (spec D5, §6.1).
- Enviar: valida o público (≥ 5 telefones), `draft → sending` e enfileira o `DispatchJob`.
- Agendar: `send_at` de 5 min a 90 dias, `draft → scheduled`.
- `Campaigns::DueJob` recorrente, a cada minuto, em cada cidade, com `FOR UPDATE SKIP LOCKED` (spec §5.2).
- Revisão: o DueJob reenfileira campanha presa em `sending` entre 10 min e 24 h. Bairro ou unidade inativos no recorte são recusados na criação, na edição, no envio e no agendamento (`inactive_or_unknown`).
- Recusas: 403 `missing_role`, 401 do `MfaStepUp`, 422 com caminho, 422 `below_minimum` (spec §6.1).
- Imutabilidade por trigger: `campaigns_frozen_after_send` (em `sent`/`cancelled`/`failed` nada muda; em `sending` só o congelamento) e `campaign_recipients_append_only` (spec §3.1, §3.2; ADR, Invariantes "A campanha enviada é imutável…").
- Eventos `campaign.created/scheduled/unscheduled/cancelled` sem PII (spec §5.7).
- Dashboard: editor (só `draft`); diálogo de envio "agora ou agendar" com step-up; `below_minimum` volta ao editor (spec §7).

**2. Código.**
- Tabela e CHECKs: `db/city_migrate/20260929100001_create_campaigns.rb:18-53` (status L37-39, título L40-42, texto L43, `send_at` obrigatório em `scheduled` L44-45, `failure_reason` ⇔ `failed` L46-49, cancelamento L50-53).
- Triggers: `db/city_triggers.sql:570-600` (`rota_campaign_guard`: só `draft` se apaga L572-577; congelada em `sent`/`cancelled`/`failed` L578-580; `sending` só vai a `sent`/`failed` e só com as colunas do congelamento L581-597), `:605-621` (`rota_campaign_recipient_guard`: identidade imutável, `notice_read_at` uma vez só, DELETE livre), `:623-638` (criação dos triggers).
- Modelo: `app/models/campaign.rb:7-13` (STATUSES, limites, `MINIMUM_PHONES = 5`, `SEND_AT_MIN_LEAD`/`MAX_AHEAD`), `:20-21` (tira espaço das pontas).
- Comandos:
  - `app/commands/campaigns/create.rb:6-21` (conteúdo, depois público, depois `campaign.created` só com ids).
  - `update.rb:7-25` (só `draft` → senão `not_editable`; só `title`/`body`/`audience`; sem evento).
  - `send.rb:6-18` (`lock!`, só `draft`, `SendGate`, `draft → sending`, grava `dispatched_by_user`, `DispatchJob.perform_later(city_slug:, campaign_id:)`).
  - `schedule.rb:5-30` (só `draft`; janela 5 min–90 dias; ISO 8601 estrito; `SendGate`; `campaign.scheduled`).
  - `unschedule.rb:4-13` (só `scheduled` → `draft`, zera `send_at` e `dispatched_by_user`).
  - `cancel.rb:5-15` (`draft`/`scheduled` → `cancelled`, com `from_status`).
  - `send_gate.rb:5-10` (revalida o público, com bairro/unidade inativos, e o mínimo).
- Controller: `app/controllers/campaigns_controller.rb:18-19` (papel antes, 404 depois), `:45-55` (criar/editar sem step-up), `:59-73` (quatro transições via `transition`), `:92-96` (step-up antes do comando), `:98-105` (mapa de erros); rotas `config/routes.rb:150-162`.
- DueJob: `app/jobs/campaigns/due_job.rb:19-29` (vencidos com `FOR UPDATE SKIP LOCKED`, `scheduled → sending`, enfileira), `:33-38` (reenfileira `sending` entre 10 min e 24 h); `config/recurring.yml:67-70`.
- Dashboard:
  - `src/modules/campaigns/CampaignEditor.tsx` (rascunho).
  - `src/modules/campaigns/SendDialog.tsx:36-82` ("agora" ou "agendar", `requiresStepUp` L75, `sendCampaign`/`scheduleCampaign` L81-82).
  - `src/modules/campaigns/CampaignLifecycleDialog.tsx:24-40` (desagendar e cancelar com `requiresStepUp` L37).
  - `src/modules/campaigns/CampaignPanel.tsx:81-90` (ações da agendada).
  - `src/lib/campaigns.ts:88` (`isEditable`).
  - `src/lib/api.ts:801-829` (cliente; escrita sempre como JSON).

**3. Aderência ao ADR 0024.**
- "estado (`draft`, `scheduled`, `sending`, `sent`, `cancelled`, `failed`) e os carimbos de quem criou e de quem enviou": ✓ CHECK L37-39; `created_by_user_id` NOT NULL; `dispatched_by_user_id` gravado no Send e no Schedule (`campaign_transitions_spec.rb:36`).
- "`campaign_manager` cria, agenda, cancela e envia": ✓ `before_action :require_campaign_manager` em todas as ações (`campaigns_controller.rb:18`).
- "Enviar, agendar e cancelar pedem step-up": ✓ `campaigns_controller.rb:92-93`; desagendar também, como na spec.
- "Nenhuma campanha sai com menos de 5 telefones distintos… a prévia mostra 'menos de 5' e o envio é recusado": ✓ `SendGate` no Send e no Schedule (422 `below_minimum`); o congelamento com público encolhido é da F-12.4.
- "A campanha enviada é imutável": ✓ trigger `rota_campaign_guard` (L578-580) e a guarda dos destinatários (L610-618).
- Revisão "Recorte ativo: … envio e agendamento recusam bairro ou unidade do recorte inativos": ✓ `SendGate` → `AudienceValidation` (`campaign_transitions_spec.rb:69`). "campanha já agendada cujo bairro é desativado depois ainda é congelada": ✓ o `DueJob` não revalida (`due_job.rb:21-26`).
- Revisão "Concorrência: … o `Campaigns::DueJob` reenfileira campanha presa em `sending` entre 10 min e 24 h": ✓ `due_job.rb:16-17,33-38`.
- "Nenhum payload de evento carrega CPF, telefone ou lista de cidadãos": ✓ `created`, `scheduled`, `unscheduled` e `cancelled` só levam ids, `send_at` e `from_status` (ruling PF7 do ledger).
- "A escrita de campanhas mora num prefixo próprio (`/campaigns`), fora de `/admin/api`": ✓ `routes.rb:150-162`.

**4. Testes.**
- api:
  - `spec/requests/campaigns_spec.rb`:
    - `:16`: cria, edita, lê e lista; chaves exatas da resposta, sem lista.
    - `:47`: público malformado → 422 com caminho, nada gravado.
    - `:66`: bairro desativado → prévia e edição recusam.
    - `:81`: `invalid_campaign`.
    - `:86`: editar fora de `draft` → `not_editable`.
    - `:92`: 404 com id inexistente ou que não é UUID.
    - `:121`: loop por todos os papéis ≠ `campaign_manager` → 403 em listar, ver, opções, prévia, criar e editar.
    - `:140`: 401 sem sessão.
  - `spec/requests/campaign_transitions_spec.rb`:
    - `:30`: enviar → `sending`, `dispatched_by`, `DispatchJob` com `city_slug`/`campaign_id`; reenviar → `invalid_transition`.
    - `:42`: 415 sem JSON.
    - `:48`: sem step-up, 401 nas 4 transições e nada muda.
    - `:57`: `below_minimum` no envio e no agendamento.
    - `:69`: bairro inativo → `invalid_audience` com caminho.
    - `:81`: `send_at` com 4 min, 91 dias, texto, `nil` ou número → `invalid_send_at`; válido → `scheduled` com evento.
    - `:94`: desagendar.
    - `:105`: cancelar de `draft` e de `scheduled`; em `sending` e `sent` → `invalid_transition`.
    - `:124`: papel e 404 antes do step-up.
  - `spec/commands/campaigns/draft_commands_spec.rb`:
    - `:11`: pontas aparadas, quebra de linha, evento só com ids.
    - `:20`: recusas de título e texto, com HTML.
    - `:36`: `'<'` que não abre tag passa.
    - `:42`, `:50`: público.
    - `:59`, `:65`, `:71`: Update parcial, recusas e `not_editable`.
  - `spec/jobs/campaigns/due_job_spec.rb`:
    - `:10`: só o que venceu, inclusive no segundo exato.
    - `:22`: duas execuções seguidas não enfileiram em dobro.
    - `:28`: cancelada ou desagendada não sai.
    - `:45`, `:52`, `:57`: presa em `sending` (11 min reenfileira só com slug e id; 1 min e 25 h não).
    - `:63`: `recurring.yml`.
  - `spec/jobs/campaigns/due_job_concurrency_spec.rb` (threads reais):
    - `:77`: DueJob × DueJob, uma vez só.
    - `:95`: Cancel segurando a linha → o DueJob pula.
    - `:116`: DueJob segurando → o Cancel relê `sending` e recusa.
    - Mutação sem `SKIP LOCKED` → vermelho (`task-13-report.md`).
  - `spec/models/campaign_tables_guard_spec.rb`:
    - `:11`: todos os CHECKs.
    - `:28`: `draft` muda livre.
    - `:33`: `sent` e `cancelled` sem UPDATE nem DELETE.
    - `:50`: `sending` só vai a `sent`/`failed` com as colunas do congelamento.
    - `:61`: destinatário (leitura uma vez, SMS muda, identidade não muda, DELETE permitido).
    - `:78`: par único.
  - `spec/invariants/campaign_invariants_spec.rb`:
    - `:32`: mínimo via `Campaigns::Send`.
    - `:104`: imutabilidade, mutação #8 = remover os triggers.
    - `:129`: eventos das transições sem PII.
    - `:176`: args dos 5 `perform_later`, com Send → Dispatch e DueJob → Dispatch.
  - `spec/initializers/domain_events_bindings_spec.rb:46`.
- dashboard:
  - `src/modules/campaigns/CampaignEditor.test.tsx`:
    - `:47`: criar, e o segundo salvar edita.
    - `:66`, `:74`, `:102`: validação local, com HTML.
    - `:127`: `not_editable`.
    - `:136`: não-rascunho não abre no editor.
    - `:143`: cancelar rascunho com step-up.
    - `:160`, `:181`: diálogos exclusivos; formulário trava com o diálogo aberto.
    - `:206`, `:213`: envio travado com menos de 5 ou com contagem velha.
    - `:225`: revisar e enviar.
    - `:243`, `:261`, `:281`: `below_minimum` e `invalid_audience` voltam ao editor.
  - `src/modules/campaigns/SendDialog.test.tsx`:
    - `:40`: agora.
    - `:59`: janela fechada pede código.
    - `:72`, `:79`: agendar (menos de 5 min recusado local; fuso da cidade).
    - `:92`, `:103`: `invalid_send_at`.
    - `:118`: `below_minimum`.
    - `:133`: Esc.
  - `src/modules/campaigns/CampaignPanel.test.tsx`:
    - `:110`: desagendar com step-up volta ao editor.
    - `:121`: cancelar com step-up.
    - `:134`: `invalid_transition` traduzido.
  - `src/modules/Campaigns.test.tsx:66,74` (navegação lista/editor/painel).
  - `src/lib/campaigns.test.ts:138` (só rascunho editável) e `:144-158` (`validateSendAt` 5 min–90 dias).
  - `src/lib/api.campaigns.test.ts:45,54,64,79` (verbos e corpo JSON das transições).

**5. Execução.**
Corrida única do api (os 31 arquivos de spec apontados pelos três grupos, api em `4425347` = origin/main):
```
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec <31 arquivos: união das listas dos três grupos>
```
```
Finished in 23.5 seconds (files took 1.61 seconds to load)
190 examples, 0 failures
```


dashboard: mesma corrida da F-12.1 (11 arquivos, 156/156 verdes).

**6. Lacunas.**
- **O `down` da migração nunca é exercitado** (lacuna de teste; ver F-12.1). A migração é a mesma.
- **O banco não impõe a ordem das transições** (lacuna de implementação menor, adiada no ledger, Task 1). O trigger deixa `draft`/`scheduled` pular direto para `sent`, `cancelled` ou `failed`: só os comandos garantem `draft → sending → sent`. Também não há guarda de INSERT em `campaign_recipients` para campanha que não está em `sending`. A imutabilidade **depois** do envio, que é o que o ADR exige, está garantida.
- **Papéis ≠ `campaign_manager` nas quatro rotas de transição** (lacuna de teste, baixa). O loop de `campaigns_spec.rb:121` cobre listar, ver, opções, prévia, criar e editar, mas não `send`/`schedule`/`unschedule`/`cancel`. Nas transições só há o caso do `municipal_admin` em `send` (`campaign_transitions_spec.rb:124`). O `before_action` é o mesmo para tudo, então o risco é pequeno.
- Menores do ledger (Task 12), adiados:
  - A falha `invalid_audience` no Schedule não tem teste. O código é compartilhado com o Send via `SendGate`, e o caminho do Send tem teste.
  - O teste de 401 não confere que nenhum job ou evento saiu.
  - O Cancel de `scheduled` mantém `dispatched_by_user_id`.
  - `reauthenticated_recently?` é redundante antes de `require_step_up!`.
- Aceitos no ledger, fora do escopo do veredito:
  - O DueJob reenfileira uma campanha presa em `sending` a cada minuto por até 24 h. É ruído de fila; o `DispatchJob` é idempotente.
  - `wait_for_lock_wait` conta qualquer espera de lock do banco, com risco raro de falso verde.
- Dashboard (ledger final, cosméticos): o sinal `sendAtRefused` persiste depois de outro erro. Depois de `below_minimum`, a prévia é invalidada e não zerada; o servidor revalida.

**7. Veredito sugerido:** **Done com lacuna**: o `down` da migração nunca foi exercitado (a mesma lacuna da F-12.1). A máquina de estados, o step-up, o DueJob (com concorrência real e mutação) e a imutabilidade pós-envio por trigger estão provados. A ordem `draft → sending` sem guarda no banco é risco menor, adiado de propósito, e não bloqueia. Com a spec de `down`/`up`, vira **Verified**.

---

---

## F-12.4 — Público congelado no envio e aviso no wpda (caixa, selo, lido)

**1. Requisito.** No envio, a lista de destinatários é gravada em `campaign_recipients` ("congelada no envio"), com leitura e estado do SMS de cada um; "o SMS e o aviso vão exatamente para as mesmas pessoas" (ADR 0024, Decisão; "Por que congelar no envio"). `DispatchJob` numa transação: `FOR UPDATE` e saída se não `sending` (idempotente); público < 5 → `failed/below_minimum`; grava a chave da cidade; `sms_status` por destinatário (`not_opted_in`, `pending`, `duplicate_phone`); `campaign.dispatched` (spec §5.3). Revisão: SMS vai ao **menor `citizen_id` com opt-in** do telefone; o mínimo é contado **nos telefones das linhas inseridas, num savepoint** (abaixo de 5 desfaz e `failed`); um **advisory lock comum** fecha a corrida congelamento × revogação. Invariante: "A revogação que anonimiza o cidadão apaga as linhas dele em `campaign_recipients`" (spec §5.6). wpda: `GET /citizen/notices` com avisos de todos os cidadãos do telefone, mais novo primeiro, `cpf_masked` só com mais de um cidadão, `read`, `unread_count` (descontando silêncio); `POST /citizen/notices/:id/read` com 404 cruzado (spec §6.2). Selo no topo ao lado de "Sair"; caixa `/avisos` com vazio "Nenhum aviso da Secretaria por enquanto."; link `/avisos` pula o termo e volta depois do login (spec §8; ADR Revisão "wpda").

**2. Código.**
- Congelamento: `api/app/jobs/campaigns/dispatch_job.rb` — `Campaign.lock.find_by` + saída fora de `sending` (L19-20); chave lida no instante (L22); savepoint `transaction(requires_new: true)` (L42) com `RecipientsFreezeLock.acquire!` (L43), INSERT…SELECT (L44), contagem de telefones **nas linhas inseridas** (L45-46), `raise ActiveRecord::Rollback` abaixo de 5 (L47); `sms_status` no próprio INSERT (L57-62: chave desligada ou sem opt-in → `not_opted_in`; `row_number() OVER (PARTITION BY c.phone, COALESCE(p.sms_opt_in,false) ORDER BY c.id) = 1` → `pending`; resto → `duplicate_phone`); `sent` + contagens + `dispatched_at` (L26-27); evento (L28-31); `SmsBatchJob` só se houver `pending` (L32-34), enfileirado depois do commit (`application_job.rb:13`); `fail_below_minimum` (L70-74). A transação externa vem de `CityScopedJob#with_city` (`app/jobs/concerns/city_scoped_job.rb:52-54`).
- Trava: `app/services/campaigns/recipients_freeze_lock.rb:11-14` (`pg_advisory_xact_lock(hashtext('campaign_recipients_freeze'))`); usada em `dispatch_job.rb:43` e `forget_revoked_recipients.rb:20`.
- Revogação: `app/services/campaigns/forget_revoked_recipients.rb:12-22` (só quando a conversa revogada é a mais recente, `created_at DESC, id DESC`; `delete_all`), chamada por `app/jobs/anonymize_revoked_triage_job.rb:16` (consumidor de `consent.revoked`, dentro da transação do `IdempotentConsumer`, `idempotent_consumer.rb:17`).
- Tabela/trigger: `db/city_migrate/20260929100001_create_campaigns.rb:58-69` (`notice_read_at`, `sms_status` com CHECK); `db/city_triggers.sql:632-635` (`campaign_recipients_append_only`).
- Caixa: `app/controllers/citizen_api/notices_controller.rb` — escopo pelo telefone da sessão `current_citizen_session.citizens` (= `Citizen.where(phone:)`, `app/models/citizen_session.rb:46-48`) (L7); `cpf_masked` só com `citizens.size > 1` (L9); ordem `dispatched_at DESC, id DESC` (L11); `unread_count` só dos não silenciados (L10, L14); `read` escopado, 404 fora do telefone, grava só quando NULL (L20-26); só campanhas `sent` (L30-32). Rotas `config/routes.rb:203-204`.
- wpda: `src/lib/citizenApi.ts:251-258` (`notices`, `readNotice`); `src/modules/citizen/NoticesStep.tsx` (vazio L11/L77, lista com "novo" L80-92, CPF mascarado L59/L89, abrir marca lido uma vez L39-51, texto como texto com `pre-wrap` L60-62); `src/modules/citizen/NoticesLink.tsx` (selo com `unread_count`, some em 0, "99+", href `BASE_URL + avisos` L8-47); `src/modules/citizen/Flow.tsx` (barra "Conta" com "Avisos" + "Sair" L153-163; `target` lido da URL L48; `enter()` manda a `/avisos` ou `/preferencias` sem termo L54-60; `signOut` esquece o destino L121-128); `src/lib/route.ts:12-22`.

**3. Aderência ao ADR 0024.**
- "`campaign_recipients` — a lista de destinatários, congelada no envio, com a leitura do aviso e o estado do SMS de cada um" — ✓ `dispatch_job.rb:44,54-67`; migração L58-69.
- "o SMS e o aviso vão exatamente para as mesmas pessoas" — ✓ o SMS só lê `campaign.recipients` (`sms_batch_job.rb:24`).
- Revisão "SMS por telefone: vai ao primeiro cidadão com opt-in daquele telefone (menor `citizen_id` entre os que optaram); os demais ficam `duplicate_phone`" — ✓ `dispatch_job.rb:59` (partição por telefone **e** opt-in).
- Revisão "Mínimo no congelamento: contado nos telefones das linhas inseridas, num savepoint; abaixo de 5, desfaz e a campanha vai a `failed`" — ✓ `dispatch_job.rb:42-49,70-74`.
- "se o público encolher entre a prévia e o envio, a campanha falha sem enviar a ninguém" — ✓ idem; nenhum `SmsBatchJob`.
- "Nenhum SMS chega à gateway … com a chave da cidade desligada no momento do congelamento" — ✓ `dispatch_job.rb:22,58` (sem `pending` com chave desligada).
- Revisão "Concorrência: um advisory lock comum fecha a corrida entre o congelamento e a revogação" — ✓ no código (`dispatch_job.rb:43`, `forget_revoked_recipients.rb:20`); ✗ **sem teste do lado do `DispatchJob`** (ver §6).
- "A revogação que anonimiza o cidadão apaga as linhas dele em `campaign_recipients`" — ✓ `anonymize_revoked_triage_job.rb:16`.
- "A campanha enviada é imutável; dos destinatários só mudam a leitura e o estado do SMS" — ✓ trigger `city_triggers.sql:632-635`; leitura grava só de NULL (`notices_controller.rb:24`).
- "a caixa de avisos mostra os avisos de todas as pessoas do telefone" (Por que telefones distintos) — ✓ `notices_controller.rb:7`.
- Revisão "wpda: … o link do SMS e a entrada em `/avisos` e `/preferencias` pulam o termo; o link 'Avisos' com o selo fica no topo, ao lado de 'Sair'; cidadão sem nome aparece pelo CPF mascarado" — ✓ `Flow.tsx:54-60,153-163`; `NoticesStep.tsx:59,89`.
- Revisão "o login do cidadão é por celular + código (não CPF + código)" — ✓ (ruling do ledger do wpda; spec §8 dizia CPF, superado pela Revisão).
- "Nenhum payload de evento carrega CPF, telefone ou lista de cidadãos" — ✓ `campaign.dispatched` só com contagens, booleano, id do usuário e o `audience` (`dispatch_job.rb:28-31`).

**4. Testes.**
- api `spec/jobs/campaigns/dispatch_job_spec.rb`: congela, grava contagens e a chave, publica 1 evento; rodar de novo não muda nada (idempotência) L17-28; encolheu abaixo de 5 → `failed`, zero linhas, sem lote de SMS L30-40; `sms_status` por destinatário (pending, duplicate_phone, not_opted_in no mesmo telefone e sem opt-in; 7 cidadãos/5 telefones) L42-59; menor `citizen_id` **com opt-in** recebe L61-71; chave desligada → todos `not_opted_in`, sem lote L73-81; revogado depois da prévia não congela L83-90; só age em `sending` L92-98.
- api `spec/services/campaigns/forget_revoked_recipients_lock_spec.rb:23-57` — esquecimento espera o detentor da trava (threads reais, sem fixture transacional). **Só um sentido**: o detentor é `RecipientsFreezeLock.acquire!` genérico, não o `DispatchJob`.
- api `spec/jobs/anonymize_revoked_triage_job_spec.rb:103-120` (revogar a conversa mais recente apaga; conversa antiga não; contadores da campanha não mudam), L122-126 (conversa sem cidadão).
- api `spec/requests/citizen_api/notices_spec.rb`: lista do telefone, mais novo primeiro, sem `cpf_masked` com uma pessoa, `unread_count` L14-27; duas pessoas com `cpf_masked` L29-37; marcar lido idempotente e conta cai L39-50; outro telefone / não-UUID → 404 e nada muda L52-60; silêncio tira do contador só a pessoa silenciada, lista inteira L62-71; 401 sem sessão, 415 sem JSON L73-79; sem N+1 L81-94.
- api `spec/models/campaign_tables_guard_spec.rb:61` (destinatário: só leitura uma vez e SMS mudam; apagar permitido), L78 (um por cidadão e campanha).
- api invariantes `spec/invariants/campaign_invariants_spec.rb`: mínimo (Rollback no congelamento) L32-40; revogado fora L44-50; chave desligada no congelamento L54-60; imutabilidade L104-112; anonimização apaga L116-125; eventos sem PII L129-161.
- wpda `src/modules/citizen/NoticesStep.test.tsx` L36 (título, data, "novo" só no não lido, sem CPF com uma pessoa), L47 (ordem da API), L55 (vazio), L62 (várias pessoas → CPF mascarado), L72 (48 px / 18 px), L79 (erro e "Tentar de novo"), L108 (abre, marca lido uma vez), L123/L135 (onRead só no sucesso), L145 (quebras de linha), L153 (HTML como texto), L161 (lido não chama POST), L168 (sem POST repetido), L177 (falha ao marcar lido).
- wpda `src/modules/citizen/NoticesLink.test.tsx` L16 (selo e href `<base>avisos`), L26, L32 ("99+"), L39 (zero, inclusive todos silenciados → sem selo), L46 (falha sem erro), L54 (abre sem recarregar).
- wpda `src/modules/citizen/Flow.notices.test.tsx` L44 (link ao lado de Sair no termo e em pessoas), L58 (sem sessão sem barra), L73 (ler tira o selo na hora), L93 (caixa sem aceitar o termo), L105 (api sem a rota), L152.
- wpda `src/modules/citizen/Flow.route.test.tsx` L54 (sem sessão: login e volta à caixa, sem termo), L67 (F5 no código), L77, L106/L119 (401 volta ao destino), L131/L146 ("Sair" esquece o destino), L164; `src/lib/route.test.ts`; `src/lib/citizenApi.test.ts` L248-282.

**5. Execução.**
- api: `Corrida única do api (os 31 arquivos de spec apontados pelos três grupos, api em `4425347` = origin/main):
```
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec <31 arquivos: união das listas dos três grupos>
```
```
Finished in 23.5 seconds (files took 1.61 seconds to load)
190 examples, 0 failures
```
`
- wpda:
```
cd apps/wpda/.claude/mod12 && npx vitest run src/modules/citizen/NoticesStep.test.tsx src/modules/citizen/NoticesLink.test.tsx \
  src/modules/citizen/PreferencesStep.test.tsx src/modules/citizen/Flow.notices.test.tsx src/modules/citizen/Flow.route.test.tsx \
  src/lib/route.test.ts src/lib/citizenApi.test.ts src/lib/format.test.ts
```
```
 ✓ src/lib/route.test.ts (15 tests)
 ✓ src/lib/citizenApi.test.ts (30 tests)
 ✓ src/lib/format.test.ts (9 tests)
 ✓ src/modules/citizen/NoticesLink.test.tsx (12 tests)
 ✓ src/modules/citizen/PreferencesStep.test.tsx (11 tests)
 ✓ src/modules/citizen/NoticesStep.test.tsx (16 tests)
 ✓ src/modules/citizen/Flow.notices.test.tsx (7 tests)
 ✓ src/modules/citizen/Flow.route.test.tsx (10 tests)
 Test Files  8 passed (8)
      Tests  110 passed (110)
```
(mesma corrida cobre F-12.4 e F-12.5.)

**6. Lacunas.**
- **Teste (principal):** a trava congelamento × revogação só é provada do lado do esquecimento; nenhum teste exercita o `DispatchJob` segurando a trava. Mutação "apagar `RecipientsFreezeLock.acquire!` de `dispatch_job.rb:43`" fica verde (conferido por leitura: o único spec com a trava usa `acquire!` direto na thread detentora). O código está certo; falta a prova.
- Teste (menor): `unread_count` "0 se todos os cidadãos do telefone silenciaram" (spec §6.2) só tem o caso parcial no api (`notices_spec.rb:62`, uma de duas silenciada); o caso "todos" só existe no front, com a API mockada (`NoticesLink.test.tsx:39`). A regra por pessoa implica o total, então é de baixo risco.
- Teste (menor, ledger T17): desempate por `id DESC` no `ForgetRevokedRecipients` sem teste.
- Implementação (menor, ledger T16): `/citizen/notices` sem paginação.
- Privacidade: nada encontrado. A lista só sai para o próprio telefone; o 404 cruzado está testado; os eventos não têm PII.

**7. Veredito sugerido:** **Done com lacuna**: falta um teste que prove que o `DispatchJob` segura a trava contra a revogação (por exemplo, uma thread segura `RecipientsFreezeLock` e o `DispatchJob` fica esperando; e a mutação sem `acquire!` fica vermelha).

---

---

## F-12.5 — Preferências do cidadão: opt-in de SMS e silenciar avisos

**1. Requisito.** "O SMS exige **opt-in explícito e próprio**, desligado por padrão, e é conferido de novo no momento de cada envio"; "O aviso no wpda não exige opt-in … e ele pode silenciar" (ADR 0024, Consentimento). `citizen_contact_preferences`: `sms_opt_in` default false, `notices_muted` default false, e a ausência de linha vale os dois desligados. Cada mudança publica evento (spec §3.3), e o `citizen.contact_preferences_changed` leva só `citizen_id` e os booleanos (§5.7). `GET /citizen/contact_preferences` lista por pessoa do telefone (`citizen_id`, `cpf_masked`, flags) + `sms_available`. `PUT …/:citizen_id` com cidadão de outro telefone → 404 (§6.2). Revisão: "`sms_available` do cidadão olha só a chave da cidade; opt-in com a chave desligada é guardado e vale quando ela ligar". wpda `/preferencias`: "Receber avisos por SMS" só quando `sms_available`, com o texto "A Secretaria de Saúde pode enviar um SMS avisando que há um aviso novo aqui. Você pode desligar quando quiser."; "Silenciar avisos" tira o selo e mantém a lista (spec §8). Tela vazia com texto aprovado pelo usuário (ledger wpda, commit 32e6420).

**2. Código.**
- Tabela: `api/db/city_migrate/20260929100001_create_campaigns.rb:72-74` (`sms_opt_in` NOT NULL default false, `sms_opt_in_changed_at`, `notices_muted` default false); modelo `app/models/citizen_contact_preference.rb:9-11` (`for` devolve linha nova desligada).
- Comando: `app/commands/citizens/update_contact_preferences.rb` — só chaves conhecidas e booleanas (L11-14); trava a linha (L24); carimba `sms_opt_in_changed_at` só quando o opt-in muda (L27); salva e publica **só quando algo muda**, payload `citizen_id` + dois booleanos (L28-33); corrida na 1ª linha (L17-19).
- Controller: `app/controllers/citizen_api/contact_preferences_controller.rb` — `sms_available: Campaigns::SmsSetting.enabled?` (só a chave, L11); pessoas do telefone (L8); `update` escopado ao telefone → 404 (L17-18); 422 `invalid_preferences` (L21). `app/services/campaigns/sms_setting.rb:5-7`. Rotas `config/routes.rb:206-207`. Evento declarado `config/initializers/domain_events.rb:83`.
- Conferência no envio: `app/jobs/campaigns/sms_batch_job.rb:37-38,54`; e no congelamento `dispatch_job.rb:58`.
- wpda: `src/lib/citizenApi.ts:259-265` (GET; `sms_available` só com `true` explícito; PUT só com o campo mudado); `src/modules/citizen/PreferencesStep.tsx` — `SMS_EXPLANATION` (L10-11, igual à spec §8), `MUTE_EXPLANATION` (L12-13), `EMPTY_PREFERENCES` (L15-16, texto aprovado), interruptor de SMS só com `sms_available` (L75-78), silêncio sempre (L79-81), um PUT por vez (L26, L35-57), `onSaved` relê o selo (L49; `Flow.tsx:227`).

**3. Aderência ao ADR 0024.**
- "O SMS exige opt-in explícito e próprio, desligado por padrão" — ✓ default false na migração; `for` sem linha = false; PUT explícito por pessoa.
- "… e é conferido de novo no momento de cada envio" — ✓ `sms_batch_job.rb:37-38,54` (detalhe em F-12.6).
- "O aviso no wpda não exige opt-in … ele pode silenciar" — ✓ `notices_muted` só tira do `unread_count` (`notices_controller.rb:10,14`); a lista continua.
- "`citizen_contact_preferences` — o opt-in do SMS e o silêncio dos avisos, por cidadão" — ✓ PK `citizen_id`.
- "Nenhum payload de evento carrega CPF, telefone ou lista de cidadãos" — ✓ `update_contact_preferences.rb:30-32`.
- Revisão "`sms_available` do cidadão olha só a chave da cidade; opt-in com a chave desligada é guardado e vale quando ela ligar" — ✓ `contact_preferences_controller.rb:11` (não olha a gateway); o PUT não depende da chave.
- Escopo por telefone (sessão por telefone, ADR 0017) — ✓ `contact_preferences_controller.rb:8,17-18`.

**4. Testes.**
- api `spec/commands/citizens/update_contact_preferences_spec.rb`: liga o opt-in com a hora, silencia, um evento por mudança sem telefone nem CPF (payload exato) L10-20; nada muda → sem linha e sem evento L22-27; repetir o valor não publica L29-33; silenciar não mexe na hora do opt-in L35-38; chave desconhecida ou não booleana → `invalid_preferences` L40-44.
- api `spec/requests/citizen_api/contact_preferences_spec.rb`: lista cada pessoa do telefone com padrões desligados e `sms_available: true` com a chave ligada L10-21; PUT altera e, com a chave desligada, `sms_available: false` e o opt-in fica guardado L23-30; outro telefone / não-UUID → 404 sem criar linha; corpo inválido → 422 L32-42; 401 L44-47.
- api `spec/models/campaign_tables_guard_spec.rb:84` (padrões desligados, uma linha por cidadão).
- api `spec/services/campaigns/sms_text_spec.rb:18-26` (`SmsSetting` desligada sem perfil e por padrão).
- api invariantes `campaign_invariants_spec.rb:54-66` (sem opt-in vigente, nenhum SMS; opt-out depois do congelamento), L129-161 (evento `citizen.contact_preferences_changed` sem CPF/telefone).
- api `spec/jobs/campaigns/sms_batch_job_spec.rb:71-78` (opt-out tardio).
- wpda `src/modules/citizen/PreferencesStep.test.tsx`: tela vazia com o texto aprovado (asserção literal) L25-30; dois interruptores com o estado da API e as explicações, com `SMS_EXPLANATION` literal L36-49; sem SMS na cidade o interruptor some L51-57; liga SMS manda só `sms_opt_in` L59; silenciar manda só `notices_muted` e chama `onSaved` L71; desligar manda false L83; falha volta o interruptor L93; toque duplo = um PUT L105; erro ao carregar L118; 48 px / 18 px L129.
- wpda `src/modules/citizen/Flow.notices.test.tsx:116` (silenciar tira o selo na hora; avisos seguem "novo" na lista); `Flow.route.test.tsx:90` (`/preferencias` direto); `src/lib/citizenApi.test.ts:283-310`.

**5. Execução.**
- api: `Corrida única do api (os 31 arquivos de spec apontados pelos três grupos, api em `4425347` = origin/main):
```
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec <31 arquivos: união das listas dos três grupos>
```
```
Finished in 23.5 seconds (files took 1.61 seconds to load)
190 examples, 0 failures
```
`
- wpda: ver F-12.4 (8 arquivos, 110/110).

**6. Lacunas.**
- Teste (menor, ledger T15): o opt-out numa linha já existente pelo **comando** não tem exemplo próprio (é coberto pelo front `PreferencesStep.test.tsx:83` com API mockada e, no api, pelo helper `opt_in!(citizen, false)`, que escreve direto no modelo). O retry de `RecordNotUnique` não tem teste.
- Implementação (menor, ledger T15): `order(:created_at)` sem desempate por id na lista de pessoas (`contact_preferences_controller.rb:8`).
- Privacidade: nada encontrado. O PUT de outro telefone dá 404 e não cria linha; o evento não tem PII; o SMS não sai sem opt-in.

**7. Veredito sugerido:** **Verified**.

---

---

## F-12.6 — Envio de SMS: gateway plugável, janela 8h–20h e estado por destinatário

**1. Requisito.** "O provedor fica atrás de uma interface (`SmsGateway`) com backends de log, teste e 'não configurado'". "Texto fixo, sempre o mesmo: 'Secretaria de Saúde de {cidade}: você tem um aviso novo. Acesse {link}'. O link leva à caixa de avisos e não carrega identificador de campanha nem de cidadão. O envio só acontece entre 8h e 20h no fuso da cidade" (ADR 0024, SMS). Invariantes: "Nenhum SMS chega à gateway sem opt-in vigente no momento do envio"; "O texto do SMS é sempre o fixo, e o link não carrega identificador". `SmsBatchJob` (spec §5.4): até 100 `pending`/`deferred`; fora da janela → `deferred` e reagenda para as 8h; gateway não configurado → `unavailable` + `campaign.sms_unavailable` uma vez; opt-in conferido de novo; `sent`/`failed` depois de 1 retentativa; falha de um não interrompe o lote; sobrou → reenfileira. §5.5: `:log` com telefone mascarado; argumentos de job sem telefone. Revisão: "Sem provedor: SMS vira `unavailable` na hora, **mesmo fora da janela** de 8h–20h, e publica `campaign.sms_unavailable` uma vez"; "Desligar a chave não interrompe os SMS pendentes de campanha já enviada"; "Antes de ligar SMS com provedor real: resolver o reenvio possível do lote (a transação fica aberta durante a chamada ao provedor)".

**2. Código.**
- Gateway: `api/app/services/sms_gateway.rb` — `deliver`/`configured?` (L9-15); backend por `config.x.sms_gateway` (L17-23); `Log` com `CitizenIdentity::Phone.mask` (L27-33); `Test` (L35-49); `Unconfigured` (`configured? = false`, `deliver` levanta `Unavailable`, L51-57). Config: `config/environments/development.rb:38` (`:log`), `test.rb:21` (`:test`); production/staging sem a chave → `Unconfigured`.
- Texto: `app/services/campaigns/sms_text.rb:7-14` (nome do `city_profile`, senão o do catálogo; link `CityPublicUrl.wpda(city) + "avisos"`, sem id).
- Lote: `app/jobs/campaigns/sms_batch_job.rb` — só campanha `sent` (L21-22); até `BATCH_SIZE = 100` `pending`/`deferred` com `FOR UPDATE SKIP LOCKED` (L14, L24-25); **gateway não configurado antes da janela** (L27) → `mark_unavailable` com evento só se `count.positive?` (L48-51); fora de `WINDOW_HOURS = (8...20)` (L15, L30) → `pending`→`deferred` e `set(wait_until: next_window_start)` (L31-32, L75-78); texto fixo (L36); opt-in relido (L37-38) → `not_opted_in` (L54); `ATTEMPTS = 2` (1 retentativa, L16, L64-73); `sent` + `sms_sent_at` / `failed` + `sms_error` = só a classe do erro (L56-60); `Unavailable` por destinatário → `unavailable` sem evento (L58); reenfileira se sobrou (L40-42). O job **não** relê a chave da cidade (a chave vale no congelamento, ruling T11). Fuso: `Time.current` em `config.time_zone = "America/Sao_Paulo"` (fixo, api#27; risco herdado do módulo).
- Argumentos: todos os `perform_later` levam `city_slug:` e `campaign_id:` (`dispatch_job.rb:33`, `sms_batch_job.rb:32,41`; `Send`/`DueJob` em F-12.3).
- Transação aberta durante o envio: `CityScopedJob#with_city` (`city_scoped_job.rb:52-54`) + `retry_on ActiveRecord::Deadlocked` (`application_job.rb:19`); registrado no ledger como PF13/T10 parked.
- Runbook: `docs/.claude/mod12/operacao/rollout-campanhas.md:35-49` ("Ligar o SMS (gate de go-live)", item 2: risco de SMS duplicado, até 99 mensagens, com chave de idempotência por destinatário ou envio fora da transação); ADR `adr/0024.md:177-180` (Revisão aponta o gate).

**3. Aderência ao ADR 0024.**
- "`SmsGateway` com backends de log, teste e 'não configurado'" — ✓ `sms_gateway.rb:17-57`.
- "Texto fixo, sempre o mesmo …" — ✓ `sms_text.rb:9`.
- "O link leva à caixa de avisos e não carrega identificador" — ✓ `sms_text.rb:12-14`.
- "O envio só acontece entre 8h e 20h no fuso da cidade" — ✓ `sms_batch_job.rb:15,30` (com o fuso fixo como risco registrado).
- Invariante "Nenhum SMS chega à gateway sem opt-in vigente no momento do envio" — ✓ `sms_batch_job.rb:37-38,54`.
- Invariante "… nem com a chave da cidade desligada no momento do congelamento" — ✓ só há `pending` com a chave ligada (`dispatch_job.rb:58`).
- Revisão "Sem provedor: `unavailable` na hora, mesmo fora da janela, e publica `campaign.sms_unavailable` uma vez" — ✓ no código (L27 antes de L30; evento só com `count.positive?`); ✗ **"mesmo fora da janela" sem teste**.
- Revisão "Desligar a chave não interrompe os SMS pendentes de campanha já enviada" — ✓ no código (o lote não lê a chave); ✗ **sem teste**.
- Spec §5.5 "Argumentos de job nunca carregam telefone" — ✓ testado nas 5 filas.
- Spec §3.2 "`sms_error` … nunca contém o telefone" — ✓ `sms_batch_job.rb:59` (só a classe).
- "Nenhum payload de evento carrega CPF, telefone ou lista" — ✓ `campaign.sms_unavailable` = `{campaign_id, count}`.
- Revisão "Antes de ligar SMS com provedor real: resolver o reenvio possível do lote" — ✓ **documentado como gate de go-live** no runbook (`rollout-campanhas.md:35-49`, item 2) e referenciado no ADR (L177-180).
- Consequência "Cada cidade nasce com o SMS desligado; ligar exige provedor configurado" / runbook "o deploy não envia SMS" — ✓ production/staging sem `sms_gateway` (invariante).

**4. Testes.**
- api `spec/services/sms_gateway_spec.rb`: `test` guarda a entrega L4-8; `log` com telefone mascarado `"[sms] (**) *****-5432 Aviso"` L10-18; sem backend: `configured? false` e `Unavailable` L20-26.
- api `spec/services/campaigns/sms_text_spec.rb:6-16` (texto exato com o nome do perfil e o link `/wpda/avisos`; sem perfil usa o catálogo).
- api `spec/jobs/campaigns/sms_batch_job_spec.rb`: dentro da janela envia o texto fixo e marca `sent` com a hora L18-25; 7h59 → `deferred` e reagenda para as 8h; 20h00 → reagenda para as 8h do dia seguinte; às 8h envia L27-43; gateway não configurado: `pending` e `deferred` → `unavailable`, **um** evento em duas execuções (às 10h) L45-51; falha isolada, 2 chamadas, `sms_error` só com a classe, o outro `sent` L53-69; opt-out tardio L71-78; lote cheio reenfileira L80-87; não toca `duplicate_phone` nem campanha não `sent` L89-95.
- api invariantes `spec/invariants/campaign_invariants_spec.rb`: sem opt-in / chave desligada no congelamento L54-66; texto fixo sem UUID, sem título, termina em `/wpda/avisos` L70-80; eventos sem PII (inclui `campaign.sms_unavailable`) L129-161; production/staging sem `sms_gateway` L164-168; argumentos só `city_slug` + `campaign_id` nas 5 filas, incluindo o reagendamento fora da janela e o lote cheio L176-209.
- api `spec/initializers/domain_events_bindings_spec.rb:43-47` (`campaign.sms_unavailable` declarado).

**5. Execução.**
- api: `Corrida única do api (os 31 arquivos de spec apontados pelos três grupos, api em `4425347` = origin/main):
```
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec <31 arquivos: união das listas dos três grupos>
```
```
Finished in 23.5 seconds (files took 1.61 seconds to load)
190 examples, 0 failures
```
`
- dashboard/wpda: não se aplica (F-12.6 é só api; o alerta de `unavailable` no painel é do F-12.7).

**6. Lacunas.**
- **Teste (principal):** a decisão da Revisão "`unavailable` na hora, **mesmo fora da janela**" não tem exemplo. O único teste do gateway ausente roda às 10h (`sms_batch_job_spec.rb:47`). Trocar a ordem de L27 e L30 (janela antes do gateway) passaria na suíte: fora da janela ficaria `deferred` até as 8h.
- **Teste:** "Desligar a chave não interrompe os SMS pendentes" (Revisão; ruling T11) não tem exemplo. Todo teste do lote roda com a chave ligada (`sms_batch_job_spec.rb:12`). Se alguém incluir uma checagem da chave no lote, a suíte continua verde.
- Teste/implementação (menor, ledger T10): o ramo `SmsGateway::Unavailable` por destinatário (`sms_batch_job.rb:58`, gateway que cai no meio do lote) não tem teste e não publica `campaign.sms_unavailable`. A retentativa que falha e depois dá certo também não tem teste.
- **Documentação (gate):** o ledger do api registra que a revisão final classificou "PF13 + T10 parked + stranded SMS (Minor 2)" como gate de go-live do SMS. O runbook tem o SMS duplicado (item 2) ✓, mas não fala do SMS **encalhado**: linhas `pending`/`deferred` cujo `SmsBatchJob` se perdeu (queda entre o commit e a fila, ou a reprogramação das 8h perdida) não são varridas por ninguém; o `DueJob` só varre `sending`. Sugestão: acrescentar ao gate do runbook, ou abrir um card.
- Risco herdado (já registrado no módulo): fuso fixo `America/Sao_Paulo` (api#27).
- Privacidade: nada encontrado. Telefone mascarado no log; `sms_error` só com a classe; os argumentos dos jobs só com ids; texto sem conteúdo nem id.

**7. Veredito sugerido:** **Done com lacuna**: falta o teste de "gateway ausente fora da janela → `unavailable` na hora + 1 evento" (decisão da Revisão). Também faltam, de preferência, o teste de "chave desligada depois do envio não interrompe" e a linha do SMS encalhado no gate do runbook.

---

---

## F-12.7 — Painel da campanha: agregados e alertas de SMS

**1. Requisito.**
- O painel mostra `recipients_count`, `phones_count`, `read_count` (cidadãos com `notice_read_at`), a contagem por `sms_status`, `sms_enabled` e `audience`, e **nenhuma lista de destinatários, em nenhuma rota** (spec §6.3; ADR, Invariantes "Nenhuma resposta do dashboard traz a lista de destinatários: só contagens").
- Revisão: `stats` só existe em campanha `sent`.
- Lidos e SMS são contados ao vivo e encolhem com a revogação (spec §5.6).
- Dashboard (spec §7):
  - destinatários, telefones, lidos (%) e SMS por status;
  - alerta destacado com `unavailable` ("SMS não enviado: a plataforma ainda não tem provedor de SMS") ou com `failed`;
  - o público congelado em frase;
  - `failed` mostra o motivo.

**2. Código.**
- `app/services/campaigns/presenter.rb:7-12` (`summary`: id, título, status, `send_at`, `dispatched_at`, `recipients_count`), `:14-20` (`full`: + `body`, `audience`, `failure_reason`, `sms_enabled`, `phones_count`, `created_at`, `stats`), `:23-31` (`stats` nulo fora de `sent`; `read_count` e as 7 chaves de `sms_status` contados ao vivo com `group(:sms_status).count`).
- `app/controllers/campaigns_controller.rb:21-24` (index), `:49-51` (show) e `:98-105` (toda escrita devolve `Presenter.full`). Nenhuma rota do dashboard serializa `campaign_recipients`. `/admin/api` não tem nada de campanha: o grep por `campaign` em `app/controllers/admin` volta vazio.
- Dashboard:
  - `src/modules/campaigns/CampaignPanel.tsx`:
    - `:21`: texto literal do alerta.
    - `:46`: relê a cada 5 s enquanto `sending`.
    - `:55-57`: frase do público.
    - `:75-79`: motivo de `failed`.
    - `:94`: `stats` ou nota.
    - `:105-142`: `Aggregates` (lidos em % sobre `recipients_count` L107; alerta `unavailable` L114-118; alerta `failed` L119-121; tabela das 7 situações de SMS ou nota de SMS desligado L127-138).
    - `:146-154`: nota sem inventar números.
  - `src/lib/api.ts:755-768` (tipos `CampaignStats` e `Campaign`, sem campo de lista).

**3. Aderência ao ADR 0024.**
- "Nenhuma resposta do dashboard traz a lista de destinatários: só contagens": ✓ Presenter só com contagens. A invariante varre `GET /campaigns`, `GET /campaigns/:id`, `GET /campaigns/options`, `GET /campaigns/sms_setting` e `POST /campaigns/preview` atrás de id/CPF/telefone dos cidadãos e de chaves `recipients`/`citizen_ids` (mutação #7 vermelha).
- Revisão "Agregados: `stats` só existe em campanha `sent`": ✓ `presenter.rb:24`; `presenter_spec.rb:13` percorre `draft`, `scheduled`, `sending`, `failed` e `cancelled`.
- "dos destinatários só mudam a leitura e o estado do SMS" + §5.6 (contagens ao vivo encolhem): ✓ `stats` lê as linhas atuais; `recipients_count`/`phones_count` ficam congelados na campanha.
- Riscos "Chave ligada sem provedor: SMS vira `unavailable`, sem reenvio… o painel avisa": ✓ `CampaignPanel.tsx:114-118`.

**4. Testes.**
- api:
  - `spec/services/campaigns/presenter_spec.rb:4` (rascunho: tudo nulo e chaves exatas do `summary`), `:13` (`stats` nulo em todo status que não é `sent`), `:21` (enviada: `read_count` e as 7 chaves de SMS com valores concretos; JSON sem "citizen").
  - `spec/requests/campaigns_spec.rb:16` (chaves exatas de `full` e de `summary` na resposta HTTP).
  - `spec/invariants/campaign_invariants_spec.rb:84` (varredura das 5 respostas do dashboard, com mutação).
- dashboard, `src/modules/campaigns/CampaignPanel.test.tsx`:
  - `:45`: destinatários, telefones, lidos com percentual e a frase do público.
  - `:55`: SMS por status; exatamente 8 linhas (7 + cabeçalho); nenhum padrão de CPF ou telefone no texto.
  - `:69`: alerta `unavailable` literal.
  - `:76`: alerta de `failed`.
  - `:82`: SMS desligado no envio.
  - `:89`, `:96`: `failed` com motivo, sem região "Resultado".
  - `:103`: `sending` sem números.
  - `:143`: relê enquanto `sending`.

**5. Execução.**
Corrida única do api (os 31 arquivos de spec apontados pelos três grupos, api em `4425347` = origin/main):
```
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose exec -T -w /rails/.claude/mod12 api bundle exec rspec <31 arquivos: união das listas dos três grupos>
```
```
Finished in 23.5 seconds (files took 1.61 seconds to load)
190 examples, 0 failures
```


dashboard: mesma corrida da F-12.1 (`CampaignPanel.test.tsx`: 12/12; total 156/156).

**6. Lacunas.**
- **Nenhuma request spec confere `stats` de uma campanha `sent`** (lacuna de teste, baixa). As contagens concretas só têm teste unitário (`presenter_spec.rb:21`). Na API, `GET /campaigns/:id` de campanha enviada só passa na invariante L84, que confere a **ausência** de lista e não os números. O caminho controller → `Presenter.full` é o mesmo do rascunho (`campaigns_spec.rb:16`), então o risco é pequeno.
- **Encolhimento depois da revogação sem teste** (lacuna de teste, baixa). Nenhum teste junta revogação e `stats`: ninguém prova que `read_count`/`sms` encolhem enquanto `recipients_count` fica. O comportamento vem do desenho (contagem ao vivo sobre linhas apagadas pela F-12.4/§5.6).
- **Percentual de lidos pode enganar** (observação, não lacuna do requisito). O dashboard divide `read_count`, que é ao vivo, por `recipients_count`, que é congelado. Depois de uma revogação o percentual fica levemente subestimado. É coerente com a spec §5.6.
- **Varredura da invariante não cobre as respostas de escrita** (ressalva). Ficam de fora POST, PATCH e as transições, mas todas usam o mesmo `Presenter.full` da rota varrida.

**7. Veredito sugerido:** **Verified**. Os requisitos estão implementados, e a privacidade (nenhuma lista em nenhuma rota) está provada com mutação. As lacunas são de teste de baixo risco: `stats` de `sent` sem request spec e encolhimento pós-revogação sem teste. Vale fechar só se o controlador quiser rigor total.

---

---

## Critério de fechamento — suíte de invariante

Arquivo: `apps/api/.claude/mod12/spec/invariants/campaign_invariants_spec.rb` (210 linhas, 10 exemplos: os 8 do critério, mais "production/staging sem gateway" L164 e "jobs só com slug e id" L176). Evidência de mutação: `apps/api/.claude/mod12/.superpowers/sdd/2026-09-29-module-12-campaigns-api/task-18-report.md`, 13 mutações na entrega e mais 3 na rodada de correção, todas vermelhas e restauradas com `git diff --stat app lib config db` vazio. A mutação da trava entre congelamento e revogação está em `final-fix-report.md`, com a spec própria `spec/services/campaigns/forget_revoked_recipients_lock_spec.rb`.

| Item do critério (`modulos/12--campanhas.md`) | Exemplo | Mutação registrada | Situação |
|---|---|---|---|
| Mínimo de 5 telefones | L32 (5 CPFs num telefone só + 3 = 4 telefones; `Send` → `below_minimum`; `DispatchJob` → `failed`, sem linhas) | #1 (tirar o `Rollback` do congelamento) e #2 (`below_minimum?` → false no `SendGate`) | ✓ coberto, nos dois pontos (envio e congelamento) |
| Revogado fora do público | L44 (congelado e `Audience#citizen_ids`) | #3 (tirar o `NOT IN (REVOKED_SQL)`) | ✓ coberto. Ressalva: a regra "a conversa **mais recente**" (revogação em conversa antiga não exclui) não entra nesta suíte; fica em `audience_spec` |
| Nenhum SMS sem opt-in e chave | L54 (chave desligada no congelamento e ligada depois → zero envios; opt-out tardio → não envia àquele) | #4 (`opted.include?` → true) e #5 (INSERT com `NOT TRUE`) | ✓ coberto |
| Texto fixo sem identificador | L70 (corpo igual a `SmsText.body`, sem UUID, sem título, termina em `/wpda/avisos`) | #6 | ✓ coberto |
| Nenhuma lista de destinatários no dashboard | L84 (5 respostas GET/POST varridas atrás de id/CPF/telefone e de chaves de lista) | #7 (`recipients:` no `Presenter.full`) | ✓ coberto. Ressalva: respostas de escrita (POST/PATCH/transições) fora da varredura, mas usam o mesmo Presenter |
| Campanha enviada imutável | L104 (UPDATE de `body` em `sent`; troca de `citizen_id` no destinatário) | #8 (remover o bloco `DO` dos triggers) | ✓ coberto. Ressalva: a mutação é grossa (tira os dois triggers de uma vez). `cancelled`/`failed`, DELETE e "`notice_read_at` uma vez" ficam em `campaign_tables_guard_spec.rb:33,61`, sem mutação registrada |
| Anonimização apaga destinatários | L116 (`RevokeConsent` + `AnonymizeRevokedTriageJob#handle`) | #9 (tirar a chamada a `ForgetRevokedRecipients`) | ✓ coberto. A corrida com o congelamento tem spec e mutação próprias (`forget_revoked_recipients_lock_spec.rb`; `final-fix-report.md`) |
| Eventos sem PII | L129 (os 9 nomes emitidos de fato, com `match_array`; nenhum CPF, telefone, lista, nem id de cidadão fora de `contact_preferences_changed`) | #10 (`citizen_ids:` em `dispatched`) e #11 (`phone:` em `contact_preferences_changed`) | ✓ coberto |
| (extra) deploy sem gateway | L164 | #12 | ✓ |
| (extra) args de job só slug + id | L176 (5 pontos de `perform_later`) | #13 e mais 3 da rodada 1 (`sms_batch_job.rb:32`, `:41`, `due_job.rb:17`) | ✓ |

**Situação:** os 8 itens do critério estão cobertos e cada um tem mutação registrada que deixa o teste vermelho. Ressalvas, nenhuma bloqueante:
- a mutação da imutabilidade é grossa;
- "conversa mais recente" fica fora da suíte de invariantes;
- a varredura do dashboard cobre as respostas GET e a prévia, e as de escrita não.
