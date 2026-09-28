# Dossiê de verificação — Módulo 10 (Profissionais), F-10.1 a F-10.5

**Data:** 2026-09-28
**ADR governante:** `docs/.claude/m10-spec/adr/0021.md` (revisado 2026-09-27) e `docs/.claude/m10-spec/adr/0019.md`
**Spec:** `docs/.claude/m10-spec/superpowers/specs/2026-09-27-module-10-professionals-design.md`
**Módulo:** `docs/.claude/m10-spec/modulos/10--profissionais.md`
**Código mergeado:**
- api: `apps/api/.claude/mod10` — origin/main `a2d20c8` (merge `66ab5fc` do feature branch + fix `32b11f1`/`a2d20c8`)
- dashboard: `apps/dashboard/.claude/mod10` — origin/main `f750c41`

**Ledgers usados:** `apps/api/.claude/mod10/.superpowers/sdd/2026-09-27-module-10-professionals-api/progress.md` e `apps/dashboard/.claude/mod10/.superpowers/sdd/2026-09-27-module-10-professionals-dashboard/progress.md`. Não foi encontrado um dossiê de módulo anterior em formato markdown (`docs/.claude/m10-spec/relatorios` e `docs/relatorios` só têm `.gitkeep`); a memória do projeto indica que fechamentos de módulo terminam em um Artifact HTML separado — este documento segue à risca a estrutura pedida pelo solicitante.

---

## F-10.1 — Cadastro do profissional (perfil)

**1. Requisito.** Perfil 1:1 com o usuário que tem o papel `health_professional` (nome profissional, conselho, UF, registro, CNS validado e cifrado, telefone e e-mail de contato opcionais e cifrados). Só o `municipal_admin` cadastra e edita todos os campos; o próprio profissional só edita nome e contato. (ADR 0021, "Decisão" e "Detalhamento"; módulo 10, entregue.)

**2. Código.**
- Migração: `apps/api/.claude/mod10/db/city_migrate/20260928000001_create_professionals.rb:10-27` (tabela `professionals`), índices únicos `idx_professionals_registration` (L21-22) e `idx_professionals_cns` (L23), checks de nome/UF/registro (L24-27).
- Modelo: `app/models/professional.rb` — `FIELDS` (L8) e `SELF_EDITABLE` (L11); `encrypts :cns, deterministic: true` (L13), `encrypts :phone`/`:contact_email` (L14-15); validações L26-32; `cns_masked` (L34).
- Serviço CNS: `app/services/professionals/cns.rb` — `FORMAT` (L9), `valid?` (L13-18, soma ponderada 15..1 mod 11), `mask` (L31-35), `generate` (L23-29, usado pela semente).
- Comandos: `app/commands/professionals/create.rb` (perfil só com papel ativo, L14; único por usuário, L15; evento `professional.created` só com ids, L24) e `app/commands/professionals/update_profile.rb` (mesmo comando serve admin e autoedição via parâmetro `allowed:`; recusa chave fora da lista com `field_not_editable`, L16; evento com só os nomes dos campos alterados, L39-40).
- Controller: `app/controllers/professionals_controller.rb` — `index`/`show`/`create`/`update` (admin, L17-57), `me`/`update_me` (L59-80, `before_action :require_admin, except: %i[me update_me]` em L17); `ERROR_STATUS` L10-14.
- Rotas: `config/routes.rb:120-134`.
- Semente: `lib/professional_crew.rb` — contas `enfermeira@`/`tecnico@`/`novato@` (L17-21), perfis só para `profissional`/`enfermeira`/`tecnico` (L23-27, `novato@` fica sem perfil de propósito), idempotência por `find_or_initialize_by`/checagem de existência (L64-103).
- Dashboard: `src/lib/api.ts:542-620` (tipos e chamadas `getProfessional`, `createProfessional`, `updateProfessional`, `getMyProfessional`, `updateMyProfessional`); `src/modules/professionals/ProfileForm.tsx` (campos sempre editáveis L57-59/83-88 vs. só-admin L60-82, CNS mascarado com "em branco mantém" L79); `src/modules/MyProfile.tsx` (leitura L37-42, edição L43-49, "sem cadastro" L24-30); `src/modules/Professionals.tsx` (lista + painel de pendência L50-118); `src/modules/professionals/ProfessionalDetail.tsx:69-76` (painel Perfil, CNS mascarado).

**3. Aderência ao ADR 0021.**
- "Todo perfil pertence a um usuário, e um usuário tem no máximo um perfil" — ✓ atendido: índice único `user_id` na migração (L11) + `Create#already_exists` (create.rb:15) + prova no banco (invariants L23).
- "O profissional edita só nome e contato; conselho, registro e CNS a prefeitura confere" — ✓ atendido: `SELF_EDITABLE` (professional.rb:11) e `update_me` (professionals_controller.rb:75-80) restringem exatamente a esses três campos; `ProfileForm.tsx` esconde os demais quando `selfService`.
- "CNS validado (dígito verificador) e cifrado determinístico" — ✓ atendido: `Cns.valid?` chamado na validação do modelo (professional.rb:32); `encrypts :cns, deterministic: true` (L13) permite o índice único funcionar sobre texto cifrado.
- "Perfil só é criado para usuário com o papel `health_professional` ativo" — ✓ atendido: `create.rb:14`.

**4. Testes.**
- `spec/models/professional_spec.rb`, `spec/services/professionals/cns_spec.rb`, `spec/commands/professionals/create_spec.rb`, `spec/commands/professionals/update_profile_spec.rb`, `spec/requests/professionals_spec.rb`, `spec/lib/professional_crew_spec.rb`.
- Invariantes: `spec/invariants/professional_invariants_spec.rb:23` ("perfil 1:1 com o usuário, garantido pelo banco") e `:32` ("perfil só para quem tem o papel").
- Dashboard: `src/lib/professionals.test.ts:6-18` (CNS válido/inválido), `src/modules/MyProfile.test.tsx:44,54,64`, `src/modules/Professionals.test.tsx:66,82,91` (cadastro a partir da pendência, CNS recusado na tela antes da API), `src/modules/professionals/ProfessionalDetail.test.tsx:56` (CNS mascarado, nunca em claro).

**5. Execução.**
```
docker compose exec -T -w /rails/.claude/mod10 api bundle exec rspec \
  spec/invariants/professional_invariants_spec.rb spec/models/professional_spec.rb \
  spec/models/professional_link_spec.rb spec/models/professional_tables_guard_spec.rb \
  spec/commands/professionals spec/services/professionals \
  spec/requests/professionals_spec.rb spec/requests/professional_links_spec.rb \
  spec/requests/professional_shifts_spec.rb spec/requests/professionals_cbo_spec.rb \
  spec/lib/professional_crew_spec.rb
→ 161 examples, 0 failures
```
```
npx vitest run src/lib/professionals.test.ts src/modules/MyProfile.test.tsx \
  src/modules/Professionals.test.tsx src/modules/professionals/ProfessionalDetail.test.tsx
→ 4 files, 60 tests passed
```
Nenhuma migração pendente relatada pelos bancos de teste.

**6. Lacunas.** Nenhum requisito do ADR/spec para F-10.1 ficou sem código ou sem teste. Riscos aceitos e registrados no ledger (não são lacunas de escopo, mas devem constar): sem auditoria de leitura do CNS; sem paginação na lista de profissionais; `me` continua acessível depois de revogar o papel (o perfil não é apagado, por decisão do ADR). Todos são "declined to judge" explícitos do usuário/revisor na fatia 1.

**7. Veredito sugerido:** **Verified.**

---

## F-10.2 — Vínculo do profissional com unidade(s)

**1. Requisito.** Vínculo profissional↔unidade com início/fim, só por acréscimo (nunca apagado/editado além do encerramento); no máximo um vínculo ativo por (profissional, unidade, CBO); abrir e encerrar exigem step-up; encerrar cancela os turnos futuros do vínculo. (ADR 0021, invariantes.)

**2. Código.**
- Migração: `db/city_migrate/20260928000001_create_professionals.rb:29-45` (tabela `professional_links`), índice único parcial `idx_professional_links_one_active` (L39-40, `WHERE ended_at IS NULL`), checks `ck_professional_links_cbo_code`/`_ending`/`_order` (L41-45).
- Trigger de só-acréscimo: `db/city_triggers.sql:443-466` (`rota_professional_link_guard`) — DELETE sempre recusado (L449-451), UPDATE só aceito quando apenas as colunas de encerramento mudam a partir de `NULL` (L452-463); instalado como `professional_links_guard` (`BEFORE UPDATE OR DELETE`, L516-518).
- Modelo: `app/models/professional_link.rb` — `scope :active` (L10), `active?` (L17), validação de `cbo_code` só em `on: :create` (L15, para não invalidar vínculos antigos com CBO depreciado).
- Comandos: `app/commands/professionals/open_link.rb` (trava unidade→profissional→vínculo, L14-16; `already_linked`, L22-24; evento `professional.linked` só com ids, L28-29) e `app/commands/professionals/end_link.rb` (`link.lock!` — FOR UPDATE — espera chamada/turno em curso, L8; `ended_at = max(Time.current, started_at)`, L14; cancela só turnos futuros, L25-31; evento `professional.unlinked` com `cancelled_shift_ids`, L16-18).
- Controller: `app/controllers/professional_links_controller.rb` — `before_action :require_admin` (L17); step-up obrigatório em `create` (L22) e `end_link` (L37); `ERROR_STATUS` L12-15.
- Rotas: `config/routes.rb:127,132`.
- Dashboard: `src/modules/professionals/ProfessionalDetail.tsx:78-116` (tabela de vínculos, ação só quando `!ended_at`, L85-90); `OpenLink` L161-223 (`SensitiveAction requiresStepUp`, L207-217); `EndLink` L233-291 (`SensitiveAction requiresStepUp`, L276; prévia de turnos futuros a cancelar, L240-260; contagem real pós-confirmação, L262-273).

**3. Aderência ao ADR 0021.**
- "`professional_links` só aceita acréscimo; encerrar grava data e hora, nada é apagado" — ✓ atendido: trigger `rota_professional_link_guard` (city_triggers.sql:443-466) + prova de banco em `professional_tables_guard_spec.rb` + `professional_invariants_spec.rb:37`.
- "No máximo um vínculo ativo por (profissional, unidade, CBO)" — ✓ atendido: índice único parcial (migração L39-40) + `OpenLink#already_linked` (open_link.rb:22-24) + `professional_invariants_spec.rb:48`.
- "Abrir e encerrar vínculo pedem step-up" — ✓ atendido: `professional_links_controller.rb:22,37` (`require_step_up! unless reauthenticated_recently?`) + dashboard `SensitiveAction requiresStepUp` nos dois fluxos.
- "Encerrar cancela, na mesma transação, os turnos do vínculo que ainda não começaram" — ✓ atendido: `EndLink.cancel_future_shifts` filtra `starts_at > now` (end_link.rb:25-31); confirmado por `professional_invariants_spec.rb:80` ("não existe turno válido futuro em vínculo encerrado").
- "Nenhum payload de evento carrega dado sensível" — ✓ atendido: `professional.linked`/`professional.unlinked` carregam só ids (open_link.rb:28-29; end_link.rb:16-18); confirmado por `professional_invariants_spec.rb:162`.

**4. Testes.**
- `spec/models/professional_link_spec.rb`, `spec/models/professional_tables_guard_spec.rb` (trigger via SQL bruto), `spec/commands/professionals/open_link_spec.rb`, `spec/commands/professionals/end_link_spec.rb`, `spec/commands/professionals/link_lock_spec.rb` (concorrência encerrar×chamar/turno), `spec/requests/professional_links_spec.rb` (inclui `mfa_required` sem TOTP recente).
- Invariantes: `professional_invariants_spec.rb:37,48,80,93` (403 em toda rota de escrita para outros papéis, com step-up aberto).
- Dashboard: `ProfessionalDetail.test.tsx:62` (abre vínculo via step-up), `:77` (pede código quando não há janela aberta), `:88` (erro ao listar unidades não fica silencioso), `:95` (prévia de turnos a cancelar), `:108` (contagem real pós-encerramento), `:273-296` (busca de CBO).

**5. Execução.** Incluído na mesma corrida do item F-10.1 (161 examples, 0 failures, cobrindo `professional_link_spec`, `professional_tables_guard_spec`, `open_link_spec`, `end_link_spec`, `link_lock_spec`, `professional_links_spec`). Dashboard: `ProfessionalDetail.test.tsx` incluído nos 60 testes verdes acima.

**6. Lacunas.** Nenhuma lacuna de requisito não testado. Riscos aceitos e registrados (parked, não bloqueantes): `lock!` (FOR UPDATE) no profissional serializa `ScheduleShift`/abertura de vínculo via FK KEY SHARE — contenção rara, ledger aceitou trocar por `FOR NO KEY UPDATE` numa limpeza futura; `link_lock_spec` tem um `release.pop` sem timeout numa thread auxiliar (não trava a suíte). Risco documentado no módulo (não é lacuna de F-10.2 isoladamente): desativar unidade não encerra os vínculos dela — sem efeito prático hoje porque unidade inativa não recebe check-in.

**7. Veredito sugerido:** **Verified.**

---

## F-10.3 — Ocupação do profissional por vínculo (CBO)

**1. Requisito.** Cada vínculo carrega um código CBO da lista versionada no api (igual para todas as cidades); abrir vínculo com CBO que exige conselho pede que o perfil seja desse conselho; trocar o conselho do perfil com vínculo ativo que dependa dele é recusado. (ADR 0021, Decisão e D9.)

**2. Código.**
- Lista: `apps/api/.claude/mod10/config/professionals/cbo_saude.yml` — 30 códigos, cada um com `code`/`title`/`council` (nulo = qualquer perfil); comentário final confirma que ACS (515105) e ACE (515140) ficaram fora até o CNES — nenhum dos dois códigos aparece no arquivo (conferido por grep).
- Serviço: `app/services/professionals/cbo.rb` — `PATH` (L6), `all`/memoização (L9-14), `active` (L16), `find` (L18).
- Coerência com o conselho: `app/commands/professionals/open_link.rb:20` (`council_mismatch` se `entry.council && entry.council != professional.council`) e `app/commands/professionals/update_profile.rb:30-36,51-57` (`blocking_cbo_codes`/`council_in_use` quando o novo conselho não cobre um CBO de vínculo ativo).
- Validação de código no modelo: `app/models/professional_link.rb:15` (`on: :create`, para não invalidar vínculos antigos com código depreciado).
- Rota: `GET /professionals/cbo` — `config/routes.rb:126`, `professionals_controller.rb:26-28`.
- Dashboard: `src/lib/professionals.ts` `MESSAGES` (`council_mismatch`, `council_in_use`, texto exato "o conselho não pode mudar enquanto houver vínculo ativo..."); `ProfessionalDetail.tsx` seletor de CBO com busca por código/título (L161-223), teste de CBO que some da busca (L273-296).

**3. Aderência ao ADR 0021.**
- "Abrir vínculo com CBO que exige conselho pede que o perfil seja desse conselho" — ✓ atendido: `open_link.rb:20`.
- "Trocar o conselho enquanto houver vínculo ativo com CBO que exige o conselho atual é recusado" (emenda 2026-09-27) — ✓ atendido: `update_profile.rb:30-36`, `council_in_use` (422).
- "A lista CBO é dado versionado no api, igual para todas as cidades; código nunca sai da lista, `deprecated` não abre vínculo novo mas continua válido nos vínculos existentes" — ✓ atendido: `Cbo.find(...).deprecated` checado em `OpenLink` (não no `ProfessionalLink` model, que só valida `on: :create`).
- "Agente comunitário e agente de endemias ficam fora da lista CBO até o CNES" — ✓ atendido: confirmado por leitura direta do yml (nenhum dos dois códigos presente).

**4. Testes.**
- `spec/services/professionals/cbo_spec.rb` (ledger registra: "CBO: 30 códigos conferidos, nenhum errado" na revisão da fatia 2), `spec/commands/professionals/open_link_spec.rb` (casos de `council_mismatch`), `spec/commands/professionals/update_profile_spec.rb` (casos de `council_in_use`), `spec/requests/professionals_cbo_spec.rb` (rota `GET /professionals/cbo`).
- Dashboard: `professionals.test.ts:45-55` (tradução das recusas nomeadas, incluindo `council_in_use` em L52).

**5. Execução.** Incluído na mesma corrida de 161 examples (0 failures) acima — `cbo_spec.rb`, `open_link_spec.rb`, `update_profile_spec.rb`, `professionals_cbo_spec.rb`. Dashboard: `professionals.test.ts` incluído nos 60 testes verdes.

**6. Lacunas.** Nenhuma. Risco documentado (não é lacuna, é característica aceita do desenho): lista CBO é estática/versionada — acrescentar uma ocupação nova exige um commit, não uma tela de administração; o módulo já registra isso como risco aceito ("Lista CBO incompleta para alguma ocupação local").

**7. Veredito sugerido:** **Verified.**

---

## F-10.4 — Turnos com data por vínculo

**1. Requisito.** Turno gravado como instantes (`starts_at`/`ends_at`), até 24h, sem sobrepor outro turno do mesmo profissional em qualquer unidade; só por acréscimo (cancelar grava data/hora/motivo, nunca apaga); não se lança turno em vínculo encerrado; turno nunca bloqueia ato clínico. (ADR 0021, Decisão e invariantes.)

**2. Código.**
- Migração: `db/city_migrate/20260928000001_create_professionals.rb:47-68` (tabela `professional_shifts`); check `ck_professional_shifts_window` (`ends_at > starts_at AND <= 24h`, L58-60); check `ck_professional_shifts_cancelling` (tudo-ou-nada no trio de cancelamento, L61-65); `EXCLUDE USING gist (professional_id WITH =, tsrange(starts_at, ends_at) WITH &&) WHERE cancelled_at IS NULL` (`excl_professional_shifts_overlap`, L66-67) — a garantia de não-sobreposição é do banco, não da aplicação.
- Trigger: `db/city_triggers.sql:468-510` (`rota_professional_shift_guard`) — no INSERT, trava o vínculo `FOR SHARE` e confere `professional_id` e vínculo não encerrado (L482-491); DELETE sempre recusado (L493-495); UPDATE só aceita o trio de cancelamento uma vez (L496-507); instalado como `professional_shifts_guard` (`BEFORE INSERT OR UPDATE OR DELETE`, L521-524).
- Modelo: `app/models/professional_shift.rb` — `MAX_DURATION = 24.hours`, `LINK_ENDED_REASON`, `MAX_REASON = 200` (L4-6), `scope :valid_shifts` (L13).
- Comandos: `app/commands/professionals/schedule_shift.rb` (valida formato e duração antes do banco, L8-10; trava o vínculo `FOR SHARE`, L13; recusa `link_ended`/`invalid_shift`, L14-15; `rescue ActiveRecord::ExclusionViolation` nomeia o turno em conflito via `conflict_for`, L23-43) e `app/commands/professionals/cancel_shift.rb` (`shift.lock!`, `already_cancelled`).
- Controller: `app/controllers/professional_shifts_controller.rb` — sem step-up ("turno não muda quem pode fazer o quê", D5); `DEFAULT_DAYS=14`, `MAX_DAYS=62`.
- Rotas: `config/routes.rb:128,129,133`.
- Dashboard: `ProfessionalDetail.tsx:118-155` (janela de 14 dias, navegação por semana); `ScheduleShift` L297-349 (`shiftWindow()` de `professionals.ts:36-42` calcula o dia seguinte quando fim ≤ início); `CancelShiftPanel` L351-380 (motivo obrigatório).

**3. Aderência ao ADR 0021.**
- "Turnos com instantes de início e fim, até 24h, sem sobrepor" — ✓ atendido pelo CHECK + EXCLUDE (autoridade é o banco); `ScheduleShift` só nomeia o conflito, não decide.
- "`professional_shifts` só aceita acréscimo; cancelar grava data e hora e motivo, sem apagar" — ✓ atendido pelo trigger + CHECK tudo-ou-nada.
- "Não existe turno em vínculo encerrado" — ✓ atendido em duas camadas: aplicação (`schedule_shift.rb:14`, `link_ended`) e banco (trigger INSERT, city_triggers.sql:489).
- "Turno nunca bloqueia ato clínico" — ✓ atendido por desenho: `ClinicalAuthorization` (F-10.5) nunca consulta `professional_shifts` (confirmado por leitura do arquivo inteiro).

**4. Testes.**
- `spec/commands/professionals/schedule_shift_spec.rb`, `spec/commands/professionals/cancel_shift_spec.rb`, `spec/models/professional_tables_guard_spec.rb` (EXCLUDE/CHECK/trigger testados via SQL direto), `spec/requests/professional_shifts_spec.rb` (inclui o teste da borda exata de 62 dias, commit `86f582b`, "test: cover the exact 62-day shift range boundary").
- Invariantes: `professional_invariants_spec.rb:65` ("só por acréscimo, sem sobreposição, até 24h"), `:80` ("não existe turno válido futuro em vínculo encerrado"), `:148` ("turno nunca bloqueia ato clínico").
- Dashboard: `ProfessionalDetail.test.tsx:120` (pagina semana), `:126` (motivo obrigatório), `:141` (plantão que cruza a meia-noite, `vi.setSystemTime`), `:153` (sobreposição nomeando o turno em conflito), `:165-218` (fluxo pós-salvar), `:218-260` (prévia de turnos futuros).

**5. Execução.** Incluído na mesma corrida de 161 examples (0 failures) — `schedule_shift_spec`, `cancel_shift_spec`, `professional_shifts_spec`, `professional_tables_guard_spec`. Dashboard: `ProfessionalDetail.test.tsx` incluído nos 60 testes verdes.

**6. Lacunas.** Nenhuma. Risco deferido e aceito (cosmético): `conflict_for` relê o turno em conflito sem lock, então sob concorrência poderia nomear um conflito diferente do real na mensagem de erro — a exclusão em si (fonte de verdade) não é afetada, só o texto da recusa.

**7. Veredito sugerido:** **Verified.**

---

## F-10.5 — Chamada e desfecho só por profissional vinculado

**1. Requisito.** Chamar e registrar desfecho clínico exigem papel `health_professional` **e** vínculo ativo com a unidade do atendimento; faltando um, a API recusa nomeando `missing_role` ou `missing_link`; "saiu sem atendimento" continua sem exigir vínculo (ato de balcão); a fila mostra o nome profissional de quem chamou. (ADR 0021 "A regra da chamada"; ADR 0019.)

**2. Código.**
- Serviço: `app/services/professionals/clinical_authorization.rb` (arquivo inteiro, 20 linhas) — `check(user:, health_unit_id:)`: papel primeiro (`:missing_role`, L12), depois vínculo ativo travado `FOR SHARE OF professional_links` escopado à unidade (L14-16), `:ok`/`:missing_link` (L17). Confirmado por leitura completa: **nenhuma referência a `professional_shifts`** no arquivo.
- Integração: `app/commands/attendances/call.rb:9-11` (depois de `attendance.lock!`, antes da checagem de `:already_called`); `app/commands/attendances/call_next.rb:10` (atalho sem lock, antes do laço — a checagem autoritativa é a de `Call`, que retrava); `app/commands/attendances/close.rb:21-26` (depois de `attendance.lock!`/`HealthUnit.lock_active!`, só quando `outcome != "left"` — L24, confirmando que "saiu sem atendimento" pula a checagem).
- Nomes na fila: `app/controllers/attendances_controller.rb:63,73` (`called_by_name`/`closed_by_name`) e `staff_name` (L85-89: `user.professional&.professional_name || user.email_address`).
- Situação do cadastro: `app/services/professionals/status.rb` (`missing_profile`/`missing_link`/`ok`) usado em `GET /setup/memberships` (`app/controllers/setup_controller.rb:126-139`) e em `GET /professionals/pending`.
- Dashboard: `src/modules/Attendance.tsx:43-70` (`canCareRole`, `myProfessional` query com `refetchInterval`, `canCare`, `careBlocked`); `src/modules/attendance/UnitQueue.tsx` — botões "Chamar"/"Chamar próximo"/desfecho só renderizados com `canCare` (L124,166,207); `handleClinicalRefusal` (L38-50, chamado em L81,99,276) trata `missing_link`/`missing_role` invalidando a consulta `myProfessional` e, no caso de `missing_role`, recarregando a sessão; `src/modules/Team.tsx:123-132` (etiqueta "sem perfil"/"sem vínculo" na Equipe, com link para Profissionais).

**3. Aderência ao ADR 0021 / ADR 0019.**
- "Chamar e registrar desfecho clínico exigem o papel `health_professional` e vínculo ativo com a unidade" — ✓ atendido nos três pontos de entrada (`Call`, `CallNext`, `Close`).
- "Faltando um dos dois, a API recusa dizendo qual falta (`missing_role` ou `missing_link`)" — ✓ atendido: símbolos exatos retornados e mapeados para 403 nos controllers.
- "'Saiu sem atendimento' continua... sem vínculo, porque é ato de balcão" — ✓ atendido: `close.rb:24` pula a checagem quando `outcome == "left"`.
- "O turno nunca bloqueia ato clínico" — ✓ atendido por desenho (serviço não toca `professional_shifts`).
- "A fila mostra o nome profissional de quem chamou" — ✓ atendido, com fallback ao e-mail quando não há perfil.
- Trava `FOR SHARE` no vínculo ordena o encerramento contra a chamada (ADR: "a trava do vínculo... ordena o encerramento contra a chamada") — ✓ atendido e testado por concorrência (`clinical_lock_spec.rb`, `link_lock_spec.rb`).

**4. Testes.**
- `spec/services/professionals/clinical_authorization_spec.rb`, `spec/commands/professionals/clinical_lock_spec.rb` (concorrência `EndLink` × `Call`), `spec/commands/attendances/call_spec.rb`, `spec/commands/attendances/close_spec.rb`, `spec/requests/attendances_spec.rb`, `spec/requests/attendance_spec.rb`, `spec/requests/setup_list_memberships_spec.rb` (`professional_status`).
- Invariantes: `professional_invariants_spec.rb:93` (403 em toda rota de escrita, com step-up aberto, para cada outro papel), `:123` ("sem papel: missing_role"), `:127` ("sem vínculo, vínculo em outra unidade, vínculo encerrado: missing_link"), `:137` ("desfecho clínico sem vínculo: missing_link; left pela recepção: ok"), `:148` ("turno nunca bloqueia ato clínico"), `:162` (nenhum evento `professional.*` carrega dado sensível).
- Dashboard: `src/modules/Attendance.test.tsx:218,237,249,253,265,291,297,306,317,351,360`; `src/modules/attendance/UnitQueue.test.tsx:88-314` (blocos `canCare`/`recepção (sem canCare)`, `missing_link`/`missing_role` no 403, `careBlocked`); `src/modules/Team.test.tsx` (etiqueta de pendência).

**5. Execução.**
```
docker compose exec -T -w /rails/.claude/mod10 api bundle exec rspec \
  spec/commands/professionals/clinical_lock_spec.rb \
  spec/services/professionals/clinical_authorization_spec.rb \
  spec/commands/attendances/call_spec.rb spec/commands/attendances/close_spec.rb \
  spec/requests/attendances_spec.rb spec/requests/attendance_spec.rb
→ 55 examples, 0 failures

docker compose exec -T -w /rails/.claude/mod10 api bundle exec rspec \
  spec/requests/setup_list_memberships_spec.rb
→ 2 examples, 0 failures
```
```
npx vitest run src/modules/Attendance.test.tsx
→ 18 tests passed
npx vitest run src/modules/Team.test.tsx src/modules/attendance/UnitQueue.test.tsx
→ 54 tests passed
```
Invariantes completas de F-10.1 a F-10.5 (161 examples) já cobrem os casos acima de `missing_role`/`missing_link`/`left`/turno-não-bloqueia.

**6. Lacunas.** Nenhum requisito de F-10.5 ficou sem teste. Achado e corrigido durante a prova no navegador (não é lacuna em aberto, mas deve constar): bug pré-existente ao módulo 10 (`GET /attendance/units` devolvia 403 para quem só tinha o papel `health_professional`, impedindo o profissional de sequer ver a fila) — corrigido em `a2d20c8` (commit `32b11f1`), já mergeado. Risco aceito e deferido: a checagem de papel em `ClinicalAuthorization` não trava a linha do usuário/membership, então uma revogação de papel exatamente concorrente com uma chamada não serializa perfeitamente (custo se errado: uma chamada rara passa um instante depois da revogação; sem violação de invariante de dados).

**7. Veredito sugerido:** **Verified.**

---

## Seção transversal

**Prova no navegador (2026-09-28).** Conforme o ledger do dashboard: como `municipal_admin` — lista e pendência de Profissionais, ficha com CNS mascarado, turno 19h–07h lançado com prévia de contagem, step-up seguido de abertura de vínculo (profissional "Carla", UBS Vila Esperança) gravados de ponta a ponta; como profissional — "Meu perfil" correto, tela Atendimento com "Chamar próximo" funcionando na UBS Jardim das Flores (unidade vinculada) e mensagem "Você não tem vínculo com esta unidade" na UBS Vila Esperança (unidade sem vínculo). TOTP do admin de dev foi redefinido para o segredo da semente com autorização do usuário; servidores da prova derrubados ao final.

**Bug pré-existente corrigido em `a2d20c8`.** A prova no navegador descobriu que um usuário só com o papel `health_professional` (sem outro papel) recebia 403 em `GET /attendance/units` e por isso nunca chegava a ver a fila — defeito anterior ao módulo 10, na checagem de acesso às unidades. Corrigido no branch `fix/units-list-for-professionals` (commit `32b11f1`), mergeado em `a2d20c8`. Sem essa correção, F-10.5 funcionaria no nível do modelo/comando mas a tela de Atendimento do profissional ficaria inacessível na prática — por isso o fix é condição de facto para o veredito Verified de F-10.5 no dashboard.

**Estado da suíte completa (ledger, não re-executado aqui por instrução do escopo).** api: fatias 1-4 mergeadas com suíte completa 2391/0 (commit `86f582b`, base do merge); invariantes com 11 mutações, todas vermelhas antes da correção (prova de que os testes realmente pegam a quebra). dashboard: 416/416 antes do merge (`66559f7`), mais a correção residual `f750c41`.

---

## Resumo executável

| F-ID | Veredito sugerido |
|---|---|
| F-10.1 (perfil) | Verified |
| F-10.2 (vínculo) | Verified |
| F-10.3 (CBO) | Verified |
| F-10.4 (turnos) | Verified |
| F-10.5 (regra da chamada) | Verified |

Nenhuma lacuna bloqueante encontrada. Riscos aceitos e já registrados nos ledgers (não bloqueiam o Verified, mas devem acompanhar o registro): sem auditoria de leitura do CNS; sem paginação na lista de profissionais; lista CBO estática (mudança via commit, não tela); `conflict_for` de turno relê sem lock (mensagem, não a exclusão); checagem de papel em `ClinicalAuthorization` sem lock de membership (janela de corrida estreita); `lock!` do profissional podendo serializar brevemente `ScheduleShift`/abertura de vínculo.
