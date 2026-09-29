# Dossiê de verificação — Módulo 11 (Território), F-11.1 a F-11.7

**Data:** 2026-09-29
**ADR governante:** `docs/adr/0023.md` (com a seção Revisão de 2026-09-29)
**Spec:** `docs/superpowers/specs/2026-09-28-module-11-territory-design.md`
**Módulo:** `docs/modulos/11--territorio.md` · **Runbook:** `docs/operacao/rollout-territorio.md`
**Código mergeado (origin/main):** api `2995f95` · dashboard `cbbb75b` · wpda `d1b552f` · docs `a64d493`
**Prova no navegador:** feita com o usuário em 2026-09-29 (Território, CEP, filtro com "< 5"/"oculto", pré-seleção no "encaminhado", bairro e unidade de referência no wpda, relatório público sem unidade e com "Voltar ao início").

## Decisão do usuário (2026-09-29)

O usuário aprovou fechar as duas lacunas antes de subir. Fechadas em api `d03fa02` (só testes; suíte completa 2583/0). Também: dica do filtro cobre "oculto" (dashboard `e810afb`) e o ADR 0023 corrigido para "cinco painéis". Os sete F-IDs sobem para `Verified` e o módulo fica `Fechado`.

## Resumo

| F-ID | Veredito sugerido | Lacuna |
|---|---|---|
| F-11.1 | **Verified** (lacuna fechada) | `down`/`up` da migração agora com spec automatizada (`spec/models/create_territory_migration_spec.rb`, api `d03fa02`) |
| F-11.2 | **Verified** (lacuna fechada) | trava do `ReplaceCoverage` provada com threads reais (`spec/commands/territory/replace_coverage_lock_spec.rb`, api `d03fa02`; mutação sem a trava → vermelho) |
| F-11.3 | Verified | — |
| F-11.4 | Verified | — |
| F-11.5 | Verified | — |
| F-11.6 | Verified | — |
| F-11.7 | Verified | risco de subtração entre campos aceito (spec §10) |
| Critério de fechamento | **cumprido** | — |

---

## F-11.1 — Bairros da cidade: semente própria e edição pelo `municipal_admin`

**1. Requisito.** Cada cidade tem a lista de bairros no seu banco (`neighborhoods`: nome único sem diferenciar maiúsculas, `active`, `source` `seed`/`manual`, `seed_key` estável). Bairro não se apaga: desativa. A lista nasce de uma semente própria versionada no api (`db/seeds/territory/<slug>.yml`), carregada por rake que cria só o que falta, casa por `seed_key` (nunca por nome) e nunca altera nem desativa o que existe. Depois, só o `municipal_admin` cria, renomeia, desativa e reativa no dashboard, sem step-up; o que ele edita prevalece. Cada mudança publica evento de domínio só com ids. (ADR 0023, Decisão, "Origem da lista", "Quem edita", Invariantes, Revisão "Semente"; spec §3.1, §3.4, §3.5, §4.1, §4.2, §5.)

**2. Código.**
- Migração: `apps/api/.claude/mod11/db/city_migrate/20260928100001_create_territory.rb:8-22` (tabela `neighborhoods`; índice único `idx_neighborhoods_name_ci` sobre `lower(name)` L17; `idx_neighborhoods_seed_key` único parcial L18; checks de nome sem espaço nas pontas/não vazio L19-20 e de origem L21-22); `down` L48-61.
- Modelo: `app/models/neighborhood.rb` — `SOURCES`/`NAME_MAX` (L6-7), normalização do nome (L12), validações (L14-15), `active_neighborhoods` (L17), `named` com o mesmo critério do índice (L20-22). Não há rota nem comando de exclusão.
- Comandos: `app/commands/territory/create_neighborhood.rb` (`blank_name` L9, `name_taken` L10 e rescue do índice L18-19, evento `neighborhood.created` só com ids L14-15); `rename_neighborhood.rb` (L7-9, evento sem o nome L13, só-caixa aceito L8-9); `set_neighborhood_active.rb` (idempotente L6, `neighborhood.activated`/`deactivated` L10-11).
- Eventos declarados: `config/initializers/domain_events.rb:67-72`.
- Controller: `app/controllers/territory_controller.rb` — `require_territory_admin` via `CitizenVerificationPolicy#manage?` = `role?(:municipal_admin)` (L53-57; `app/policies/citizen_verification_policy.rb:7-9`), `index` com inativos, `source`, `active` e `units` (L19-22, L75-80), `create`/`update`/`deactivate`/`activate` (L24-42), `ERROR_STATUS` (L11-14), `not_found` (L59-62). Sem step-up.
- Rotas: `config/routes.rb:138-144` (prefixo único `/territory`).
- Semente: `app/services/territory/seed.rb` — casa por `seed_key` (L35), aviso para nome já usado fora da semente sem adotar a chave (L36-38), cria com `source: "seed"` (L40), relatório criados/existentes/avisos (L15, L24-26); `lib/tasks/territory.rake` — `city:territory:seed[slug]` (L23-29), `:all` com classe e mensagem do erro (L33-42), cidade sem arquivo sai sem erro (L11-14), roda via `CityConnection.with` (L16). Arquivos: `db/seeds/territory/curitiba.yml` (75 bairros com `key`) e `maringa.yml` (44).
- Semente de dev: `lib/territory_crew.rb` (endereços e cidadãos com bairros variados, alguns sem; idempotente).
- Dashboard: `apps/dashboard/.claude/mod11/src/lib/api.ts:678-716` (cliente `/territory`: `listNeighborhoods`, `createNeighborhood`, `renameNeighborhood`, `setNeighborhoodActive`); `src/lib/territory.ts` (`normalizeName` sem acento L11-13, `validateNeighborhoodName` L28-33, mensagens em português das recusas L35-51); `src/modules/Territory.tsx` (tabela com nome, origem, estado e nº de unidades ativas L92-125, busca L52-53/L85-88, diálogo criar/renomear L133-168, desativar/reativar L38-50/L113-118, invalida o cache do seletor dos painéis L32-36); menu `src/shell/modules.ts:41-43,73` (grupo "Cidade" só para admin); `src/App.tsx:78`; proxy `vite.config.ts:44`.

**3. Aderência ao ADR 0023.**
- "`neighborhoods` — nome (único sem diferenciar maiúsculas), ativo e origem (`seed` ou `manual`)" — ✓ índice `lower(name)` + checks na migração (L17-22) + `Neighborhood.named`.
- "Bairro não se apaga: desativa" — ✓ não há rota nem comando de exclusão; só `deactivate`/`activate` (routes.rb:142-143).
- "Semente própria versionada no api (`db/seeds/territory/<slug>.yml`)" — ✓ curitiba.yml (75) e maringa.yml (44).
- "Tarefa rake carrega criando só o que falta, reconhecendo cada item por `seed_key`, e não pelo nome; nunca altera nem desativa o que existe" (Decisão + Revisão "Semente") — ✓ `seed.rb:35-40`; invariante idempotente e sem desfazer edição.
- "O `municipal_admin` cria, renomeia, desativa e reativa bairros no dashboard. O que ele edita prevalece" — ✓ `Territory.tsx`; semente não toca bairro existente.
- "Só o `municipal_admin` edita. Não há step-up" — ✓ `require_territory_admin` (territory_controller.rb:53-57), sem `require_step_up`; menu escondido para outros papéis.
- "Cada mudança publica um evento de domínio (ADR 0014)" e "eventos carregam só ids" — ✓ `create_neighborhood.rb:14`, `rename_neighborhood.rb:13`, `set_neighborhood_active.rb:10`; nomes declarados (domain_events.rb:67-72).
- "Bairro inativo não aparece em escolha nova, mas continua no histórico e no filtro" — ✓ `index` do `/territory` lista inativos; `active_neighborhoods` usado nas escolhas (cobertura: ver F-11.2).

**4. Testes.**
- api: `spec/models/neighborhood_spec.rb` (normalização L4, recusas L8, `named` L14, escopo ativo L19); `spec/models/territory_tables_guard_spec.rb` (índice único sem caixa L50, `seed_key` única L55, CHECK de nome/origem L63 — por SQL direto); `spec/commands/territory/neighborhood_commands_spec.rb` (criar manual/semente L9/L18, `blank_name` L24, `name_taken` L31, renomear sem o nome no evento L41, mantém `seed_key` L47, só a caixa L53, mesmo nome sem evento L58, recusas L63, ativar/desativar idempotente L74); `spec/requests/territory_spec.rb` (ciclo completo L14, recusas 422/404 L53-83, 403 `missing_role` para cada papel ≠ `municipal_admin` em todas as rotas L92-114, 401 sem sessão L117); `spec/services/territory/seed_spec.rb` (cria com chave L30, duas vezes não muda L41, não desfaz edição L50, bairro manual homônimo L66, entradas inválidas L76); `spec/services/territory/seed_files_spec.rb` (arquivos existem L8, nomes únicos L17, `key` slug única L23, Curitiba com 75 L37); `spec/tasks/territory_rake_spec.rb` (L38-83); `spec/lib/territory_crew_spec.rb` (L33, L48); `spec/initializers/domain_events_bindings_spec.rb:35-36`.
- Invariantes: `spec/invariants/territory_invariants_spec.rb:266` (semente idempotente e não desfaz edição, nem renomear) e `:303` (eventos só com ids, com nota de mutação).
- Dashboard: `src/modules/Territory.test.tsx` (lista L46, busca sem maiúsculas e acentos L60, criar com nome aparado L68, nome vazio sem chamar API L79, renomear L97, diálogo com foco e Escape L164); `src/lib/territory.test.ts` (`normalizeName` L11, `validateNeighborhoodName` L42, `territoryError` L51, cliente L74); `src/shell/modules.test.ts:80` ("Território só para municipal_admin").

**5. Execução.**
```
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker compose exec -T -w /rails/.claude/mod11 api bundle exec rspec \
  spec/models/neighborhood_spec.rb spec/models/territory_tables_guard_spec.rb \
  spec/commands/territory/neighborhood_commands_spec.rb spec/commands/territory/replace_coverage_spec.rb \
  spec/requests/territory_spec.rb spec/services/territory/seed_spec.rb spec/services/territory/seed_files_spec.rb \
  spec/tasks/territory_rake_spec.rb spec/lib/territory_crew_spec.rb spec/requests/health_units_address_spec.rb \
  spec/requests/health_units_spec.rb spec/invariants/territory_invariants_spec.rb \
  spec/initializers/domain_events_bindings_spec.rb spec/invariants/health_unit_invariants_spec.rb
```
```
Finished in 35.8 seconds (files took 1.85 seconds to load)
109 examples, 0 failures
```
(api em `2995f95`, contido em `origin/main`; só avisos de depreciação de `:unprocessable_entity` do Rack na saída.)

```
cd apps/dashboard/.claude/mod11 && npx vitest run src/modules/Territory.test.tsx src/lib/territory.test.ts \
  src/lib/viacep.test.ts src/lib/unitAddress.test.ts src/modules/attendance/UnitForm.test.tsx \
  src/modules/attendance/Units.test.tsx src/shell/modules.test.ts
```
```
 ✓ src/shell/modules.test.ts (10 tests)
 ✓ src/lib/viacep.test.ts (8 tests)
 ✓ src/lib/territory.test.ts (12 tests)
 ✓ src/lib/unitAddress.test.ts (8 tests)
 ✓ src/modules/attendance/UnitForm.test.tsx (13 tests)
 ✓ src/modules/attendance/Units.test.tsx (11 tests)
 ✓ src/modules/Territory.test.tsx (15 tests)
 Test Files  7 passed (7)
      Tests  77 passed (77)
```
(dashboard em `cbbb75b`, contido em `origin/main`.) Prova no navegador pelo usuário (2026-09-29): tela Território com lista, busca sem acento, criar e desativar — aprovada.

**6. Lacunas.**
- **Migração "reversível" sem teste do `down`.** A spec §3 exige migração "só de expansão e reversível"; `down` existe (`20260928100001_create_territory.rb:48-61`), mas nenhum spec o executa. O ledger do api registrou a regra "exercitar down e depois up" para a Task 16, e ela não foi cumprida (grep em `spec/` não acha `CreateTerritory` nem a versão da migração). A lacuna vale para a migração inteira (também F-11.2 e F-11.3), e fica registrada aqui.
- Menores adiados (ledger, não bloqueantes): os testes de renomear/desativar no dashboard não conferem o recarregamento da lista; guardas `is_a?(String)` e rescue `RecordNotUnique → name_taken` duplicados no controller e nos comandos; o controller de território reaproveita `CitizenVerificationPolicy#manage?` (explicado no comentário L54-55, mas acopla a autorização do território à política da validação presencial).
- Risco herdado (módulo, não lacuna de código): Maringá tem 44 bairros conferidos na base de CEP, não a lista oficial completa que a spec §3.4 previa; a prefeitura completa pelo dashboard.

**7. Veredito sugerido:** **Done com lacuna** — `down` da migração de território nunca exercitado (spec §3 "reversível"). Todo o resto do requisito tem código, teste verde e prova no navegador.

---

## F-11.2 — Cobertura: quais unidades atendem cada bairro

**1. Requisito.** `neighborhood_coverages` guarda o par (bairro, unidade), único; ligar cria a linha, desligar apaga. O `municipal_admin` substitui o conjunto de unidades de um bairro, e só unidades ativas e existentes entram. Bairro inativo não recebe cobertura. `ReplaceCoverage` trava o bairro (`FOR UPDATE`), calcula adicionados e removidos e publica `neighborhood.coverage_changed` só com ids. A semente liga cobertura só em bairro criado na própria carga. No dashboard, a cobertura é editada em caixas de seleção das unidades ativas. (ADR 0023, Decisão e Invariantes "Bairro inativo não aparece em escolha nova (cobertura)"; spec §3.2, §3.4, §3.5, §4.1, §4.2, §5.)

**2. Código.**
- Migração: `db/city_migrate/20260928100001_create_territory.rb:24-30` (tabela, FKs, índice único `idx_neighborhood_coverages_pair`).
- Modelo: `app/models/neighborhood_coverage.rb:3-6`; associação `Neighborhood has_many :coverages / :health_units` (`neighborhood.rb:9-10`).
- Comando: `app/commands/territory/replace_coverage.rb` — normaliza ids (downcase + uniq, L9), `lock("FOR UPDATE")` no bairro (L12), rollback em falha (L14), `inactive_neighborhood` (L20), `inactive_unit` quando algum id não é unidade ativa (L21), diferença adicionados/removidos (L23-27), evento só com ids e só se mudou (L28-32).
- Controller: `territory_controller.rb:44-49` (`coverage` recusa corpo que não é lista de strings com `inactive_unit`), JSON com `units: [{id, name, active}]` (L78).
- Rota: `config/routes.rb:144`.
- Semente: `app/services/territory/seed.rb:44-48` (cobertura só no bairro criado nesta carga; unidade ativa achada pelo nome, sem maiúsculas, L51-55; não achada = aviso).
- Dashboard: `src/lib/api.ts:712-716` (`replaceCoverage` com `health_unit_ids`); `src/modules/Territory.tsx:170-237` (`CoverageEditor`: caixas das unidades ativas marcadas pelo que já cobre L171-173, aviso de unidade desativada que sai ao salvar L178/L223-227, substitui o conjunto L189-199); botão "Cobertura" só em bairro ativo (L107-112); coluna "Unidades" conta só as ativas (L97-99).

**3. Aderência ao ADR 0023.**
- "`neighborhood_coverages` — par (bairro, unidade), único" — ✓ índice único (migração L29-30), provado por SQL direto.
- "Ligar a cobertura cria a linha; desligar a apaga" — ✓ `replace_coverage.rb:26-27` (`delete_all`/`create!`).
- "Cada mudança publica um evento de domínio, que é a trilha" — ✓ `neighborhood.coverage_changed` com `added_unit_ids`/`removed_unit_ids` (L29-31); sem evento quando nada muda.
- "Bairro inativo não aparece em escolha nova (cobertura)" — ✓ `inactive_neighborhood` (L20); dashboard esconde o botão em bairro inativo e fecha o editor ao desativar.
- "A unidade de referência nunca inclui unidade inativa" (lado da cobertura) — ✓ só unidade ativa entra (L21); unidade desativada depois de coberta continua listada com `active: false` e o dashboard a retira ao salvar. (A leitura da referência é de F-11.5.)
- "Só o `municipal_admin` edita a cobertura, sem step-up" — ✓ mesmo `before_action` de F-11.1; 403 testado na rota `coverage`.
- "A carga da semente nunca altera nem desativa o que existe" (cobertura editada prevalece) — ✓ `seed.rb:35,44-48`.
- Spec §4.2 "trava o bairro (`FOR UPDATE`)" — ✓ no código (L12); sem teste (ver Lacunas).

**4. Testes.**
- api: `spec/commands/territory/replace_coverage_spec.rb` (substitui e publica adicionados/removidos só com ids L12; mesmo conjunto em outra ordem e com repetição não publica L25; lista vazia remove tudo L31; unidade inativa, inexistente ou id não-UUID → `inactive_unit` e nada muda L37; id em caixa alta normalizado, sem remover e readicionar L47; bairro inativo L57); `spec/requests/territory_spec.rb` (cobre no ciclo L14; unidade desativada depois de coberta continua listada com `active false` L42; 422 `inactive_unit` com unidade inativa, inexistente ou corpo sem lista L67; 422 `inactive_neighborhood` L77; 403 na rota de cobertura para todo papel ≠ admin L92-114); `spec/models/territory_tables_guard_spec.rb:69` (par único no banco); `spec/services/territory/seed_spec.rb:30,50` (cobertura da semente só de unidade ativa; cobertura editada não é refeita).
- Invariantes: `territory_invariants_spec.rb:44` (bairro inativo não entra em cobertura), `:266` (semente não refaz cobertura esvaziada no admin), `:303` (evento de cobertura sem nome de unidade).
- Dashboard: `Territory.test.tsx` (bairro inativo sem botão de cobertura L130; caixas das ativas marcadas e substituição do conjunto L137; unidade desativada sai com aviso L151; foco e Escape no editor L173; desativar fecha o editor L183; `inactive_unit` traduzido L192); `territory.test.ts:89` (`health_unit_ids` no corpo).

**5. Execução.** Incluído nas mesmas corridas de F-11.1: api **109 examples, 0 failures** (inclui `replace_coverage_spec.rb`, `territory_spec.rb`, `territory_tables_guard_spec.rb`, `seed_spec.rb`, `territory_invariants_spec.rb`); dashboard **77 passed** (inclui `Territory.test.tsx`, `territory.test.ts`). Prova no navegador pelo usuário (2026-09-29): cobertura editada na tela Território — aprovada.

**6. Lacunas.**
- **Trava `FOR UPDATE` sem teste de concorrência.** A spec §4.2 exige que `ReplaceCoverage` trave o bairro para que duas edições simultâneas não calculem a diferença sobre o mesmo estado; o código faz isso (`replace_coverage.rb:12`), mas nenhum spec exercita duas substituições concorrentes. O módulo 10, por exemplo, tem `link_lock_spec.rb` para o mesmo tipo de regra. Se alguém tirar o `lock`, a suíte continua verde.
- Herda a lacuna do `down` da migração (registrada em F-11.1).
- Menor adiado (ledger dashboard T4): corrida silenciosa se a lista de unidades ativas mudar com o editor aberto (a API recusa com `inactive_unit`, traduzido; não há perda silenciosa de dado).

**7. Veredito sugerido:** **Done com lacuna** — trava `FOR UPDATE` de `ReplaceCoverage` sem teste de concorrência (spec §4.2).

---

## F-11.3 — Endereço da unidade, com consulta de CEP pelo navegador

**1. Requisito.** `health_units` ganha endereço em texto, todo opcional: logradouro (até 160), número (até 20), complemento (até 80), CEP (exatamente 8 dígitos) e o bairro onde a unidade **fica** (pode não estar entre os que atende). `create`/`update` de `/attendance/units` aceitam os campos; CEP inválido = 422 `invalid_zip`; bairro inexistente = 422 `invalid_neighborhood`; as leituras devolvem os campos. Só o `municipal_admin` edita. O CEP é consultado pelo **navegador** do dashboard, direto no ViaCEP (timeout de 5 s); o api nunca chama serviço de CEP. Sucesso preenche o logradouro e mostra o bairro do CEP só como sugestão (pré-seleciona bairro da lista se o nome bate, sem diferenciar maiúsculas e acentos). Erro, `{erro: true}` ou timeout: aviso e preenchimento à mão. Se houver CSP no deploy, liberar `connect-src https://viacep.com.br`. (ADR 0023, Decisão "`health_units` ganha endereço" e "CEP", Invariante "O `api` nunca chama serviço externo de CEP", "Quem edita"; ADR 0018; spec §3.3, §4.1, §5.)

**2. Código.**
- Migração: `db/city_migrate/20260928100001_create_territory.rb:32-38` (colunas com limites, CHECK `ck_health_units_address_zip` `^[0-9]{8}$`, FK `neighborhood_id`).
- Modelo: `app/models/health_unit.rb` — `belongs_to :neighborhood, optional: true` (L13), normalização (vazio → nil; CEP sem ponto, hífen e espaço) (L20-22), validações de tamanho e formato (L26-29). `lock_active!` do módulo 09 preservado (L38-40).
- Controller: `app/controllers/health_units_controller.rb` — `ADDRESS_FIELDS` (L9); `assign_address` só muda as chaves presentes no corpo (L90-103, preserva o endereço quando o cliente manda só nome e tipo), recusa valor não escalar (L95), bairro inexistente (inativo aceito como localização) (L96-98); códigos `invalid_zip`/`invalid_neighborhood` (L105-107, L114-115); leituras devolvem os campos (`unit_json` L121-125); escrita só `require_admin` (L12; `attendance_access.rb:17-19` → `manage?` = `municipal_admin`).
- Rotas: `config/routes.rb:95-100` (inalteradas, módulo 09).
- Ausência do ViaCEP no api: nenhum arquivo em `app/`, `lib/`, `config/` ou `db/` contém "viacep" (grep e invariante).
- Semente de dev: `lib/territory_crew.rb:13-28,50-59` (CEP real do bairro, não sobrescreve endereço preenchido).
- Dashboard: `src/lib/viacep.ts` (URL `https://viacep.com.br/ws/<cep>/json/` L17, `credentials: "omit"`, `VIACEP_TIMEOUT_MS = 5000` L5 com `AbortController` L12-13, `{erro}`/HTTP/rede/timeout → `{ok:false}` L18-27, nunca rejeita); `src/lib/unitAddress.ts` (máscara L14-17, `zipError` L19-22, payload vazio → `null` L34-45, `formatAddress` L47-52); `src/modules/attendance/UnitForm.tsx` (consulta ao completar 8 dígitos L44-60; descarta resposta de CEP já trocado L47/L51; logradouro só é sobrescrito se o CEP trouxe rua L57; pré-seleção só de bairro **ativo** e só se nenhum foi escolhido L54/L58; aviso de falha L91-93; "bairro segundo o CEP: X" L94-96; CEP incompleto bloqueia L64-65; bairro atual inativo aparece marcado L42/L115); `src/modules/attendance/Units.tsx` (lista com endereço e nome do bairro L75/L107; edição parte do endereço atual L113; envia o endereço em create/update L55-63); `src/lib/territory.ts:22-26` (`matchNeighborhood` sem acento e só ativos); `src/lib/api.ts:383-415` (tipos e chamadas com endereço). CSP: `nginx.conf:9-10` registra que hoje não há CSP e que `connect-src` precisa liberar o ViaCEP se um dia houver.

**3. Aderência ao ADR 0023 (e 0018).**
- "`health_units` ganha endereço em texto (logradouro, número, complemento, CEP) e o bairro onde a unidade fica, que pode não estar entre os que ela atende. Tudo opcional" — ✓ migração L32-38; `neighborhood_id` é independente da cobertura; bairro inativo aceito como localização.
- "O módulo 09 ganha o endereço da unidade, sem refazer a tabela" (Consequências; ADR 0018) — ✓ só `add_column`/`add_reference`; `health_unit_invariants_spec.rb` do módulo 09 continua verde.
- "O endereço pode ser preenchido pela consulta de CEP, que o navegador do dashboard faz direto no ViaCEP" — ✓ `viacep.ts:17`.
- "O `api` não consulta serviço de CEP" (Invariante) — ✓ grep e invariante `territory_invariants_spec.rb:294`.
- "O bairro vindo do CEP é só sugestão: vale o bairro escolhido da lista da cidade" — ✓ `UnitForm.tsx:54-58` (só sugere bairro ativo; não troca escolha feita; o que vai à API é o `neighborhood_id` da lista).
- "O dashboard depende do ViaCEP, com o preenchimento à mão como saída" — ✓ toda falha vira `{ok:false}` e aviso; campos continuam livres.
- "Só o `municipal_admin` edita o endereço da unidade, sem step-up" — ✓ `require_admin` nas escritas; 403 para atendente e profissional em `health_units_spec.rb:83,108`.

**4. Testes.**
- api: `spec/requests/health_units_address_spec.rb` (cria com endereço e leituras devolvem L16; update sem as chaves preserva o endereço L31; `null` apaga L40; bairro inativo aceito como localização L47; `invalid_zip` com CEP fora de 8 dígitos ou não texto, nada gravado L54; `invalid_neighborhood` com inexistente, não-UUID ou não texto L62; logradouro acima de 160 → `invalid_unit` L71); `spec/models/territory_tables_guard_spec.rb:76` (CHECK do CEP no banco); `spec/requests/health_units_spec.rb:83,108` (403 nas escritas para quem não é admin); `spec/lib/territory_crew_spec.rb:33,48` (endereço de CEP real, idempotente, não sobrescreve).
- Invariantes: `territory_invariants_spec.rb:294` ("o api nunca chama serviço de CEP", varredura de `{app,lib,config,db}` com nota de mutação).
- Dashboard: `src/lib/viacep.test.ts` (consulta direta sem cookie L10; `{erro: true}` e `{erro: "true"}` L20; rede L30; CEP sem 8 dígitos não chama a rede L40; desiste em 5 s L44-58); `src/lib/unitAddress.test.ts` (máscara, `zipError`, payload com `null`, API antiga sem endereço, `formatAddress`); `src/modules/attendance/UnitForm.test.tsx` (sucesso com sugestão e pré-seleção L34; sem maiúsculas e acentos L45; bairro inativo não pré-selecionado L53; não troca bairro escolhido L61; CEP geral não apaga a rua L70; `{erro: true}` com aviso L82; timeout L98; resposta de CEP trocado descartada L114; CEP incompleto bloqueia L134; bairro atual inativo marcado L155); `src/modules/attendance/Units.test.tsx` (edição só de nome mantém o endereço no payload L56-68; mostra endereço com nome do bairro L136; cria com bairro sugerido pelo CEP L141); `src/lib/territory.test.ts:31,96` (nunca sugere bairro inativo; `createUnit` com endereço achatado).

**5. Execução.** Incluído nas mesmas corridas de F-11.1: api **109 examples, 0 failures** (inclui `health_units_address_spec.rb`, `health_units_spec.rb`, `health_unit_invariants_spec.rb`, `territory_invariants_spec.rb`, `territory_crew_spec.rb`); dashboard **77 passed** (inclui `viacep.test.ts` 8, `unitAddress.test.ts` 8, `UnitForm.test.tsx` 13, `Units.test.tsx` 11). Complemento por grep no fonte do api:
```
grep -rn -i "viacep" app lib config   # sem resultado
```
Prova no navegador pelo usuário (2026-09-29): endereço da unidade com consulta de CEP, e endereço mantido ao editar só o nome — aprovada.

**6. Lacunas.** Nenhuma lacuna de requisito sem código ou sem teste. Registros não bloqueantes:
- A spec §5 prevê liberar `connect-src https://viacep.com.br` "se houver CSP"; hoje não há CSP no `nginx.conf` do dashboard (só o comentário L9-10). Condição ainda não disparada, sem teste possível hoje; vale como item de deploy.
- Menores adiados (ledger): `address_number` numérico em JSON é recusado como `invalid_unit` (só aceita texto; o dashboard sempre manda texto); em `Units.tsx`, falha ao listar bairros é engolida (o seletor fica só com "—", sem alerta); dois testes do `UnitForm` usam espera por tempo.
- Herda a lacuna do `down` da migração (registrada em F-11.1; a migração também cria as colunas de endereço).

**7. Veredito sugerido:** **Verified.**

---

## F-11.4 — Bairro declarado pelo cidadão, copiado de forma imutável na triagem

**1. Requisito.** O cidadão declara o bairro uma vez, na escolha da pessoa no wpda (lista de bairros ativos com busca), com "Prefiro não informar", e pode trocar depois ("Trocar bairro"). O bairro é gravado sob a mesma regra do CPF: nada antes do consentimento vigente (ADR 0017). Bairro inválido é recusado **antes** de criar o cidadão (`RegisterPerson`). A triagem copia o bairro do cidadão no `INSERT` e ele nunca muda depois; a única exceção é virar `NULL` na anonimização por revogação de consentimento. Trocar o bairro não muda triagens anteriores. Evento `citizen.neighborhood_changed` só com ids. (ADR 0023, "Quando o cidadão declara", "Por que copiar na triagem", Invariantes e Revisão "Revogação"; spec §3.3, §4.1, §4.2, §6.)

**2. Código.**
- Migração: `apps/api/.claude/mod11/db/city_migrate/20260928100001_create_territory.rb:40` (`citizens.neighborhood_id`, FK + índice), `:42-43` (`triages.neighborhood_id`, FK + índice `idx_triages_neighborhood_created`), `:45` (executa `db/city_triggers.sql`); `down` derruba trigger/função (L49-50).
- Trigger: `db/city_triggers.sql:529-561` — `rota_triage_neighborhood_guard` (L539-548): qualquer mudança de `neighborhood_id` levanta erro, salvo `NEW.neighborhood_id IS NULL AND NEW.status = 'aborted_by_revocation'` (L542-544); instalado como `triages_neighborhood_immutable` `BEFORE UPDATE` só se a coluna existe (L551-561).
- Modelos: `app/models/citizen.rb:16` e `app/models/triage.rb:7` (`belongs_to :neighborhood, optional: true`); `app/models/neighborhood.rb:17` (`scope :active_neighborhoods`).
- Comando: `app/commands/citizens/set_neighborhood.rb:8-21` — `nil`/`""` = "prefiro não informar" (L9), recusa inativo/inexistente com `invalid_neighborhood` (L10), `lock!` + evento só quando muda, só ids (L13-18).
- Cópia: `app/commands/start_triage.rb:22` (`neighborhood_id: conversation.citizen&.neighborhood_id` no `create!` — única escrita).
- Anonimização: `app/jobs/anonymize_revoked_triage_job.rb:9-13` (`neighborhood_id: nil` junto com o conteúdo clínico, só em `aborted_by_revocation`).
- Controllers: `app/controllers/citizen_api/conversations_controller.rb:16-18` (consentimento primeiro), `:23-24` (`requested_neighborhood_id`, bairro inválido = 422 **antes** de `resolve_citizen`), `:26` (`resolve_citizen` → `RegisterPerson`, L81-91), `:31-34` (grava só se a pessoa ainda não tem bairro, antes de `StartConversation`), `:95-102`; `app/controllers/citizen_api/people_controller.rb:8-11` (`people` com `neighborhood`), `:13-26` (troca; sem a chave ou não-texto = 422; CPF fora da sessão = 404); `app/controllers/citizen_api/neighborhoods_controller.rb:6-9` (só ativos, por nome).
- Rotas: `config/routes.rb:169-170`. Evento declarado: `config/initializers/domain_events.rb:72`.
- wpda: `apps/wpda/.claude/mod11/src/lib/territory.ts:16-24` (busca sem acento/maiúsculas); `src/modules/citizen/NeighborhoodPicker.tsx:8-22` (lista com busca, "Prefiro não informar" L16); `src/modules/citizen/PeopleStep.tsx:56-68` (começa com o bairro; 422 relê a lista e pergunta de novo), `:77-83` (pergunta uma vez a quem não tem; cidade sem bairros não pergunta), `:96-112` ("Trocar bairro", `null` tira), `:145-153` (bairro na lista de pessoas + botão "Trocar bairro de …"); `src/lib/citizenApi.ts:183-191` (`neighborhoods`, `setNeighborhood`, `start` só manda `neighborhood_id` quando há); `src/modules/citizen/Flow.tsx:58-60` (422 `invalid_neighborhood` sem erro genérico).

**3. Aderência ao ADR 0023.**
- "Nada é gravado antes do consentimento vigente" — ✓ atendido: `conversations_controller.rb:16-18` confere a versão antes de validar o bairro e de `RegisterPerson`; `neighborhood_spec.rb:52` prova 409 sem `Citizen` e sem evento.
- "Bairro inválido recusado antes de criar o cidadão" (spec §4.2) — ✓ atendido: `conversations_controller.rb:23-24` antes de `:26`; `neighborhood_spec.rb:44` (inativo, inexistente, não-UUID, lista → 422 e `Citizen.count == 0`).
- "'Prefiro não informar' é permitido" — ✓ atendido: `null`/ausente/`""` tratados como sem bairro (controller L97, comando L9); wpda `NeighborhoodPicker.tsx:16`.
- "Pode trocar depois; trocar não muda triagens anteriores" — ✓ atendido: rota `POST /citizen/people/:id/neighborhood` + trigger; `neighborhood_spec.rb:75`, `start_triage_neighborhood_spec.rb:13`.
- "Pessoa que já tem bairro: o do início é ignorado" (spec §4.2) — ✓ atendido: `conversations_controller.rb:31`; `neighborhood_spec.rb:66`.
- "Bairro inativo não aparece em escolha nova" — ✓ atendido: `neighborhoods_controller.rb:7` e `set_neighborhood.rb:10`; `territory_invariants_spec.rb:44`.
- "Cópia na triagem imutável; única exceção NULL na revogação" — ✓ atendido: trigger `city_triggers.sql:539-561` + job `anonymize_revoked_triage_job.rb:12`; provas `territory_tables_guard_spec.rb:22,30,43` e `territory_invariants_spec.rb:18,32`.
- "Eventos só com ids" — ✓ atendido: `set_neighborhood.rb:18` (`citizen_id`, `from_id`, `to_id`); `territory_invariants_spec.rb:303`.

**4. Testes.**
- api: `spec/commands/citizens/set_neighborhood_spec.rb:10,23,30`; `spec/commands/start_triage_neighborhood_spec.rb:13,22` (cópia; troca posterior não muda; sem cidadão/sem bairro = nulo); `spec/requests/citizen_api/neighborhood_spec.rb:21,26,36,44,52,59,66,75,88,95,105,115`; `spec/models/territory_tables_guard_spec.rb:22,30,43` (trigger via SQL); `spec/jobs/anonymize_revoked_triage_job_spec.rb:84`; `spec/invariants/territory_invariants_spec.rb:18,32,44,303`.
- wpda: `src/lib/territory.test.ts:15` (sem acento); `src/modules/citizen/NeighborhoodPicker.test.tsx:10,23`; `src/modules/citizen/PeopleStep.test.tsx:34,48,73,83,92,106,122,133,162,182,197,253`; `src/modules/citizen/Flow.test.tsx:127`; `src/lib/citizenApi.test.ts:183,193,198,206,213,220,229`.
- Prova no navegador (usuário, 2026-09-29): cidadão escolheu o bairro com busca sem acento; a triagem copiou; a lista de pessoas mostra o bairro e "Trocar bairro".

**5. Execução.**
```
docker compose exec -T -w /rails/.claude/mod11 api bundle exec rspec \
  spec/commands/citizens/set_neighborhood_spec.rb spec/commands/start_triage_neighborhood_spec.rb \
  spec/requests/citizen_api/neighborhood_spec.rb spec/requests/citizen_api/reference_units_spec.rb \
  spec/services/territory/reference_units_spec.rb spec/requests/attendance_reference_units_spec.rb \
  spec/models/attendance_territory_spec.rb spec/jobs/anonymize_revoked_triage_job_spec.rb \
  spec/invariants/territory_invariants_spec.rb spec/models/territory_tables_guard_spec.rb
→ Finished in 14.28 seconds (files took 2.54 seconds to load)
→ 62 examples, 0 failures
```
```
cd apps/wpda/.claude/mod11 && npx vitest run src/lib/territory.test.ts src/modules/ReferenceUnits.test.tsx \
  src/modules/Report.test.tsx src/modules/citizen/Flow.test.tsx src/modules/citizen/NeighborhoodPicker.test.tsx \
  src/modules/citizen/PeopleStep.test.tsx src/modules/citizen/ResultStep.test.tsx src/App.test.tsx src/lib/citizenApi.test.ts
→ Test Files  9 passed (9)
→ Tests  102 passed (102)
```
(api em `2995f95`, wpda em `d1b552f`, ambos contidos em `origin/main`.)

**6. Lacunas.** Nenhum requisito sem código ou sem teste. Observações registradas no ledger do api (Task 8/final review, minor deferred), não bloqueantes: (a) sem teste de id não-UUID e de `""` na rota `people/:id/neighborhood` — o caso não-UUID está coberto no nível do comando (`set_neighborhood_spec.rb:30`); (b) o bairro gravado no início persiste se `StartConversation` falhar depois (o consentimento já foi conferido, então não fere a LGPD); (c) conversa **retomada** mantém a triagem já criada com bairro nulo mesmo que o bairro seja informado agora — coerente com "cópia só na criação", documentado; (d) a exceção do trigger é por estado (`aborted_by_revocation`), não pelo job: qualquer UPDATE para NULL nessa linha passa, o que é exatamente a regra do ADR.

**7. Veredito sugerido:** **Verified.**

---

## F-11.5 — Unidade de referência no resultado da triagem (área logada)

**1. Requisito.** O resultado da triagem, na área logada do cidadão, mostra o bloco "Sua unidade de referência" (nome, tipo e endereço) com as unidades **ativas** que cobrem o bairro **copiado na triagem**, calculadas na hora e nunca gravadas; várias = todas; nenhuma = bloco ausente; informa, nunca restringe. O relatório público `/r/:token` **nunca** mostra a unidade nem o bairro; ele ganhou "Voltar ao início". O bloco e as ações ficam dentro do cartão do resultado. (ADR 0023, "Unidade de referência", Invariantes, Revisão "wpda"; spec §4.1, §4.2, §6.)

**2. Código.**
- Fonte única: `apps/api/.claude/mod11/app/services/territory/reference_units.rb:10-16` (`for`: ativas, cobertura do bairro, por nome; bairro nulo = `[]`), `:31-39` (`as_json_list` com `address` em objeto).
- Área logada: `app/controllers/citizen_api/triages_controller.rb:21-29` (`GET /citizen/triages/:id` acrescenta `reference_units` do `triage.neighborhood_id`, não do bairro atual).
- Relatório público: `app/controllers/reports_controller.rb:15-26` — resposta fixa com `tier`, `priority`, `recommendation`, `completed_at`, `expires_at`; nenhum campo territorial. Rota `config/routes.rb:157`.
- wpda: `src/modules/ReferenceUnits.tsx:9-33` (bloco; vazio/null = nada; singular/plural; texto "Você pode procurar qualquer unidade de saúde"); `src/lib/territory.ts:4-13,31-52` (rótulo do tipo, endereço/CEP); `src/modules/citizen/ResultStep.tsx:25-26` (unidades vêm de `citizenApi.triage`), `:46-47` (bloco + ações como `children` do `Report`, dentro do `<article>` — `src/modules/Report.tsx:63`), `:49-53` (antes do relatório ficar pronto); `src/App.tsx:9` (rota pública: `Report` só com `BackHomeLink`); `src/modules/BackHomeLink.tsx:3-12` ("Voltar ao início" para `import.meta.env.BASE_URL`); `src/lib/citizenApi.ts:85,145` (`reference_units` normalizado para `[]`).

**3. Aderência ao ADR 0023.**
- "Conjunto das unidades ativas que cobrem o bairro, calculado na hora e nunca gravado" — ✓ atendido: `reference_units.rb:13-15`, nenhuma coluna nova; `reference_units_spec.rb:53` e `territory_invariants_spec.rb:59` (unidade desativada some).
- "Do bairro copiado na triagem" (spec §4.2) — ✓ atendido: `triages_controller.rb:27`; `reference_units_spec.rb:46` (trocar o bairro depois não muda a referência da triagem antiga).
- "Mostrada no resultado da triagem, na área logada" — ✓ atendido: `ResultStep.tsx:46-53`; `ResultStep.test.tsx:70,77,90`.
- "Nenhuma = bloco some; várias = todas" — ✓ atendido: `ReferenceUnits.tsx:10-11`; `ReferenceUnits.test.tsx:15,20,29`; `ResultStep.test.tsx:105`.
- "O relatório público nunca mostra a unidade de referência nem o bairro" — ✓ atendido em duas camadas: api (`reports_controller.rb:19-25`; `territory_invariants_spec.rb:91` e `reference_units_spec.rb:62` varrem o corpo por `reference_units`, `neighborhood`, id e nome do bairro e da unidade) e wpda (`App.tsx:9`; `Report.test.tsx:106` — mesmo que a API mande `reference_units`, a tela pública não mostra).
- "Relatório público ganhou 'Voltar ao início'" (Revisão) — ✓ atendido: `BackHomeLink.tsx`; `App.test.tsx:8` (aponta para a base do wpda, dentro do cartão, sem unidade nem bairro).
- "Bloco e ações dentro do cartão do resultado" (Revisão) — ✓ atendido: `ResultStep.tsx:47`; `ResultStep.test.tsx:90`.
- "Informa, nunca restringe" — ✓ atendido: bloco só leitura com o aviso de livre escolha (`ReferenceUnits.tsx:29-31`); nenhuma regra de acesso lê a referência.

**4. Testes.**
- api: `spec/services/territory/reference_units_spec.rb:15,21,27`; `spec/requests/citizen_api/reference_units_spec.rb:29,40,46,53,62`; `spec/invariants/territory_invariants_spec.rb:59,91`.
- wpda: `src/modules/ReferenceUnits.test.tsx:15,20,29,38`; `src/modules/citizen/ResultStep.test.tsx:70,77,90,105`; `src/modules/Report.test.tsx:106`; `src/App.test.tsx:8`; `src/lib/territory.test.ts:69`; `src/lib/citizenApi.test.ts:234`.
- Prova no navegador (usuário, 2026-09-29): resultado logado mostra "Sua unidade de referência — UBS Jardim das Flores" dentro do cartão; relatório público sem unidade/bairro e com "Voltar ao início".

**5. Execução.** Incluído nas corridas do F-11.4: api **62 examples, 0 failures** (inclui `reference_units_spec.rb` dos dois tipos e `territory_invariants_spec.rb`); wpda **9 files, 102 tests passed** (inclui `ReferenceUnits.test.tsx` 6, `ResultStep.test.tsx` 6, `Report.test.tsx` 10, `App.test.tsx` 1).

**6. Lacunas.** Nenhuma. Minors aceitos no ledger do wpda, fora do requisito: metadados do `Report` em 12/13px (abaixo da regra de 18px, não interativos); o bloco abaixo da dobra foi julgado na prova do navegador e aprovado.

**7. Veredito sugerido:** **Verified.**

---

## F-11.6 — Unidade de referência pré-selecionada no desfecho "encaminhado"

**1. Requisito.** Cada linha da fila da unidade (`GET /attendance/units/:id/queue`, que alimenta o formulário de desfecho) traz `reference_unit_ids`: unidades ativas que cobrem o bairro da triagem do atendimento (pela triagem ou pela cadeia horário → pedido → triagem raiz); sem triagem, o bairro atual do cidadão; **nunca** a própria unidade do atendimento. No dashboard, o desfecho "encaminhado" já vem com a primeira de referência por nome; as demais sobem com a etiqueta "referência"; se só a própria unidade for de referência, nada vem escolhido; o profissional pode trocar. A pré-seleção nunca vaza para outro desfecho. (ADR 0023, "Unidade de referência", Consequências, Revisão "Desfecho"; spec §4.1, §5.)

**2. Código.**
- Bairro do caso: `apps/api/.claude/mod11/app/models/attendance.rb:26-28` (`root_triage`: a própria ou a do pedido do horário) e `:38-41` (`territory_neighborhood_id`: o copiado na triagem raiz; sem triagem raiz, o bairro atual do cidadão; triagem sem bairro continua sem bairro).
- Lote sem N+1: `app/services/territory/reference_units.rb:19-29` (`ids_by_neighborhood`, só ativas, por nome).
- Fila: `app/controllers/attendances_controller.rb:23-28` (calcula as referências de `waiting + in_care`), `:64-71` (`reference_unit_ids: refs.fetch(...) - [ a.health_unit_id ]`, L70). Rota `config/routes.rb:101`.
- Encerramento sem mudança de regra (ADR 0019): `attendances_controller.rb` `close` repassa `referral_unit_id` a `Attendances::Close`.
- Dashboard: `apps/dashboard/.claude/mod11/src/lib/api.ts:451` (`reference_unit_ids?`); `src/lib/attendance.ts:48-57` (`splitReferenceUnits`: remove a própria unidade de novo, por defesa, ignora id fora das ativas, ordena por nome pt-BR); `src/modules/attendance/UnitQueue.tsx:258-260` (`referralChoice ?? referenceUnits[0]?.id ?? ""`: sugestão até o profissional mexer), `:272-273` (unidade só enviada quando o desfecho é `referred`), `:306-314` (select com "—", as de referência com "· referência" no topo, depois as outras).

**3. Aderência ao ADR 0023 (e 0019).**
- "O dashboard a pré-seleciona como destino no desfecho 'encaminhado', e o profissional pode trocar" — ✓ atendido: `UnitQueue.tsx:260,309`; `UnitQueue.test.tsx:98,111,122`.
- "A pré-seleção nunca escolhe a própria unidade do atendimento" (Revisão) — ✓ atendido em duas camadas: api `attendances_controller.rb:70` e dashboard `attendance.ts:52`; provas `attendance_reference_units_spec.rb:21,36`, `territory_invariants_spec.rb:71`, `attendance.test.ts:72`, `UnitQueue.test.tsx:156`.
- "Bairro do caso: o da triagem; sem triagem, o atual do cidadão" (spec §4.1/D7) — ✓ atendido: `attendance.rb:38-41`; `attendance_territory_spec.rb:10,15,19,25`.
- "Nunca inclui unidade inativa" — ✓ atendido: `reference_units.rb:24`; `attendance_reference_units_spec.rb:54`; dashboard ignora id fora das ativas (`attendance.test.ts:67`, `UnitQueue.test.tsx:167`).
- "Informa e sugere; nunca restringe" / "o ADR 0019 não muda de regra" — ✓ atendido: todas as unidades ativas continuam no select, incluindo a própria (`UnitQueue.test.tsx:207`); `Attendances::Close` não foi alterado para a referência.
- Pré-seleção não vaza para outro desfecho — ✓ atendido: `UnitQueue.tsx:272-273`; `UnitQueue.test.tsx:132` (desfecho padrão sem unidade) e `:141` (`it.each` "encaminhado com referência pré-selecionada, trocado para Retorno / Atendido e liberado: não manda unidade").

**4. Testes.**
- api: `spec/requests/attendance_reference_units_spec.rb:21,36,46,54`; `spec/models/attendance_territory_spec.rb:10,15,19,25`; `spec/services/territory/reference_units_spec.rb:21`; `spec/invariants/territory_invariants_spec.rb:71`.
- dashboard: `src/lib/attendance.test.ts:61,67,72`; `src/modules/attendance/UnitQueue.test.tsx:98,111,122,132,141,156,167` (bloco "unidade de referência (módulo 11)") e regressões do encaminhamento `:207,216,249,279`.
- Prova no navegador (usuário, 2026-09-29): cidadão de Santa Felicidade atendido na UPA 24h Centro → destino pré-selecionado "UBS Jardim das Flores"; cidadão do Centro (bairro coberto pela própria unidade) → nada pré-selecionado.

**5. Execução.**
- api: incluído na corrida de **62 examples, 0 failures** (inclui `attendance_reference_units_spec.rb`, `attendance_territory_spec.rb`, `territory_invariants_spec.rb`).
```
cd apps/dashboard/.claude/mod11 && npx vitest run src/lib/attendance.test.ts src/modules/attendance/UnitQueue.test.tsx
→ ✓ src/lib/attendance.test.ts (8 tests)
→ ✓ src/modules/attendance/UnitQueue.test.tsx (33 tests)
→ Test Files  2 passed (2)
→ Tests  41 passed (41)
```
(dashboard em `cbbb75b`, contido em `origin/main`.)

**6. Lacunas.** Nenhuma. A lacuna de teste que o ledger do dashboard registrou na Task 6 ("no test for referred→switch-to-Retorno/Atendido leak path") foi fechada na onda final — o teste existe e passa (`UnitQueue.test.tsx:141`, `it.each` com os dois desfechos).

**7. Veredito sugerido:** **Verified.**

---

## F-11.7 — Filtro de bairro nos painéis, com supressão de contagens de 1 a 4

**1. Requisito.** Os cinco painéis com cidadão de `/admin/api` (Visão geral, Classificação, Triagens, Relatórios, Conversas; Filas fica fora por ser fila de jobs) aceitam `neighborhood_id=<uuid>` ou `neighborhood_id=none`; parâmetro inválido = 422 `invalid_neighborhood`; a resposta ganha `filter: { neighborhood: {id,name} | "none" | null }`. A lista do seletor vem de `GET /admin/api/neighborhoods`, com a mesma autorização dos painéis (todos os papéis que os leem), inclusive inativos marcados. Com o filtro ligado, todo número de 1 a 4 sai `{ suppressed: true }` (0 continua 0). Decisões da Revisão do ADR 0023 (2026-09-29): listas de amostra (`sampleTriages`, linhas de Relatórios) vêm `null` quando **qualquer** contagem ou ponto de série do painel é suprimido; taxa cujo numerador é de 1 a 4 também é suprimida, mesmo com total visível; o dashboard mostra "< 5" para contagem e "oculto" para taxa/média; o console `admin` nunca manda o parâmetro e não é afetado. Risco residual aceito: subtração entre campos da mesma resposta (spec §10). (ADR 0023, "Painéis por bairro", "Supressão", Revisão; spec §4.1, §4.3, §5; módulo 11, F-11.7.)

**2. Código.**
- api — objeto único do filtro: `app/queries/admin/neighborhood_filter.rb` — `parse` (L20-28: vazio = desligado; `none`; UUID de bairro existente, ativo ou inativo; o resto levanta `Invalid`), recortes `triages` (L47-49, bairro copiado), `conversations` (L51-53, bairro atual do cidadão, `none` inclui conversa sem cidadão), `report_snapshots` (L55-57, via triagem), `count` (L59-64), `series` (L66-71), `over` (L74-76), `share` (L80-85, suprime se numerador **ou** total for 1-4), `list` (L88-90).
- `app/queries/admin/small_count.rb:4-17` — `SUPPRESSED = { suppressed: true }` congelado, `RANGE = 1..4`, `wrap` aplicado depois de agregar.
- `app/controllers/admin/api/base_controller.rb` — `rescue_from ... Invalid` → 422 (L36, L102-104); `neighborhood_filter` (L60-62); `render_envelope(..., filter:)` acrescenta `filter.neighborhood` (L66-72).
- Controllers dos cinco painéis passam o filtro e o descritor: `overview_controller.rb:3-4`, `classification_controller.rb:3-4`, `triages_controller.rb:3-4`, `reports_controller.rb:6`, `conversations_controller.rb:3-4`. `queues_controller.rb:7` não usa filtro.
- Queries:
  - `overview_query.rb` — KPIs `done`/`active`/`urgent` com `count` + `series` (L41-87); `completion` com `share(completed, started, rate)` (L89-105, tom neutro quando suprimido); `failed` (jobs) ignora o filtro de propósito (L110-122); `delta` sempre `nil`.
  - `classification_query.rb` — tiers, urgentes, `urgentTrend`, pivô por protocolo (com apelidos `low/medium/high` e `priorityTrue/priorityTrend` também suprimidos, L103-104), `byMode` com `share` (L108-120); amostra `null` se o total for pequeno **ou** se `suppressed_anywhere?` achar supressão em tiers, urgentes, qualquer ponto de `urgentTrend`, protocolo ou modo (L44-46, L66-76).
  - `triages_query.rb` — série, `started`, `completed`, `completionRate` por `share` (L29-34), `byProtocol` com `count`/`share` (L40-54).
  - `reports_query.rb` — `total` por `count`; lista `null` se o total estiver suprimido **ou** se algum grupo por tier ou por protocolo tiver 1-4 linhas (L28-44).
  - `conversations_query.rb` — `live`, funil, saídas, `liveActive` por `count` (L36-48); `abandonRate` por `share` (L54-59); `avgToCompleteMin` por `over(total, …)` (L61-72).
- `app/controllers/admin/api/neighborhoods_controller.rb:7-12` + `config/routes.rb:271` — `GET /admin/api/neighborhoods`, herda `BaseController` (sessão + vínculo ativo ou grant de operador), sem envelope `data`, ordenado por nome, inativos com `active: false`.
- dashboard:
  - `src/lib/neighborhoodFilter.ts` — filtro na URL (`?bairro=<uuid>|none`, L12-30), lixo vira "Todos" (L17-20), `neighborhoodParams` (L51-53).
  - `src/components/NeighborhoodPicker.tsx:12-54` — "Todos", "Sem bairro", bairros com "(inativo)"; bairro desconhecido limpa a URL (L18-19); placeholder enquanto a lista carrega/falha (L24, L44); dica da regra (L51).
  - Seletor nos cinco painéis: `Overview.tsx:39`, `Classification.tsx:210`, `Triages.tsx:73`, `Reports.tsx:58`, `Conversations.tsx:70`.
  - Hooks com o bairro na chave de cache e no parâmetro: `useOverview.ts:9-12`, `useClassification.ts:9-12`, `useTriages.ts:9`, `useReports.ts:9-12`, `useConversations.ts:9-12`.
  - Rótulos: `src/lib/smallCount.ts` — `SUPPRESSED_LABEL = "< 5"` (L6), `HIDDEN_LABEL = "oculto"` para `%`/`min` (L15-21), ponto suprimido vira lacuna no gráfico, nunca 0 (L33-35), proporção só quando todas as categorias têm número (L39-41); `src/components/Count.tsx:6-30`; `src/components/StatTile.tsx:25-29` (sem unidade nem delta quando suprimido, L72/L80).
  - Listas ocultas com texto neutro (vale para bairro e para "Sem bairro"): `Classification.tsx:85-86` ("amostra oculta"), `Reports.tsx:28-32` ("lista oculta").
- admin: nenhuma referência a `neighborhood` em `apps/admin/src` (grep vazio) — o console não manda o parâmetro.

**3. Aderência ao ADR 0023 (e Revisão).**
- "Os painéis da cidade que envolvem o cidadão aceitam filtro por bairro" (cinco, Filas fora) — ✓ os cinco controllers passam `neighborhood_filter`; Filas não; o resto de `/admin/api` ignora o parâmetro (request spec L39).
- "O bairro de cada caso vem da cópia na triagem; o que não tem triagem usa o bairro atual do cidadão" — ✓ `triages`/`report_snapshots` pela cópia; `conversations` pelo bairro atual (neighborhood_filter.rb:47-57; spec de query L60-66 prova que a conversa segue o bairro atual).
- "Com o filtro ligado, contagem de 1 a 4 não é mostrada; 0 continua 0" — ✓ `SmallCount.wrap` em todo KPI, contagem, série e pivô; `small?` exclui 0.
- "Sem filtro, a cidade inteira aparece como antes" — ✓ `off` devolve as relações e os números intactos; specs "sem filtro: igual ao de antes" nas cinco queries.
- Revisão — listas `null` quando **qualquer** contagem ou ponto de série é suprimido — ✓ Classificação (L44-46, inclusive `urgentTrend` em qualquer granularidade, corrigido em `67f20b2`); Relatórios (L34-44: total, grupo de tier ou de protocolo). Relatórios não tem série própria; a regra do "grupo dentro da lista" é mais estrita que o ADR.
- Revisão — taxa com numerador 1-4 suprimida mesmo com total visível — ✓ `share` em `completion` (Visão geral), `completionRate` e `byProtocol.share` (Triagens), `byMode.share` (Classificação), `abandonRate` (Conversas); média `avgToCompleteMin` por `over(total)` (o total é a própria contagem da média).
- Revisão — "< 5" para contagem e "oculto" para taxa/média — ✓ `smallCount.ts:6,15-21`, `StatTile.tsx:25-29`, `Count.tsx:7`.
- Revisão — filtro para todos os papéis via `GET /admin/api/neighborhoods` — ✓ `NeighborhoodsController < BaseController`; request spec itera `Membership::ROLES` (L51-52), 403 sem vínculo, 401 sem sessão.
- Bairro inativo "continua no histórico e no filtro dos painéis" — ✓ `parse` aceita inativo; a lista traz inativos marcados; o seletor mostra "(inativo)".
- "O console `admin` segue sem enviá-lo" — ✓ nada em `apps/admin/src`; os 46 testes do admin passam. A resposta sem filtro ganha a chave aditiva `filter: { neighborhood: null }`, que o admin ignora.
- Invariante "Com o filtro de bairro ligado, nenhum painel devolve contagem de 1 a 4" — ✓ varredura recursiva nos cinco painéis (territory_invariants_spec.rb:114-180), com as exceções nomeadas (`urgentMaxPriority`, `priority` da amostra, KPI `failed` pelo id).
- Risco residual aceito (spec §10, Revisão): subtração entre campos da mesma resposta — ✓ registrado como aceito; não é regra que o código precise cumprir (ver Lacunas).

**4. Testes.**
- api:
  - `spec/queries/admin/neighborhood_filter_spec.rb` — `parse` (vazio, `none`, inativo, lixo/UUID inexistente/lista → `Invalid`), recortes (triagem, conversa com `none` incluindo conversa sem cidadão, relatório), conversa seguindo o bairro atual, `count`/`series`/`over`/`share`/`list` com fronteira 4/5 e filtro desligado sem efeito.
  - `spec/queries/admin/overview_classification_filter_spec.rb` — Visão geral (sem filtro = cidade inteira; bairro 1-4: KPI, série e taxa suprimidos e `failed` não; 5+; `completion` suprimida com iniciadas ≥5 e concluídas 1-4; `none`); Classificação (tudo suprimido e amostra `null`; bairro grande com uma categoria de 1 caso → categoria e share suprimidos e amostra `null`; bairro grande sem supressão → amostra aparece; **período por hora com uma hora de 1-4 urgentes → amostra `null`**, L111; sem filtro igual).
  - `spec/queries/admin/triages_reports_conversations_filter_spec.rb` — Triagens (1-4, 5+, `completionRate` por subtração, sem filtro igual); Relatórios (1-4 e `none` → total suprimido e lista `null`; 5+ sem supressão → lista; **split 5+1 de protocolo** e **tier com 1-4 linhas** → lista `null` com total visível; sem filtro igual); Conversas (1-4 com zeros continuando zero; 5+; `abandonRate` por subtração; `none` com conversa do WhatsApp sem cidadão).
  - `spec/requests/admin/api/neighborhood_filter_spec.rb` — para cada um dos cinco painéis: descritor `filter` nos três modos e 422 `invalid_neighborhood`; o resto de `/admin/api` ignora o parâmetro; `GET /admin/api/neighborhoods` para cada papel de `Membership::ROLES`, 403 sem vínculo, 401 sem sessão.
  - Invariantes: `spec/invariants/territory_invariants_spec.rb:114-261` (varredura dos cinco painéis em "bairro com 3 casos", "bairro com 7 casos, um deles único na categoria" e "sem bairro"; prova de que a fixture exercita número pequeno sem filtro; amostra/lista somem e reaparecem; share do modo minoritário; média suprimida).
  - Evidência de mutação (ledger `task-16-report.md` e `final-fix-report.md`): tirar `@filter.count` de `kpi_done` → vermelho; tirar `share` de `by_mode` → vermelho; tirar `over` de `avg_complete_minutes` → vermelho; anular a supressão da amostra (Classificação) e de `suppress_list?` (Relatórios) → vermelho; tirar o ramo de protocolo de `small_group?` → vermelho; `urgentTrend` fora da checagem da amostra → vermelho antes da correção `67f20b2`.
- dashboard: `src/lib/smallCount.test.ts` ("< 5", "oculto" para `%` e `min`, 0 continua 0), `src/components/StatTile.test.tsx:9,18` (taxa e média "oculto" sem unidade nem delta; contagem "< 5"), `src/lib/neighborhoodFilter.test.ts`, `src/components/NeighborhoodPicker.test.tsx` (grava `?bairro=`, dica, "Todos" apaga, placeholder, bairro desconhecido), `src/modules/neighborhoodPanels.test.tsx` (os cinco painéis mandam `neighborhood_id` da URL e mostram o seletor; "Sem bairro" → `none`; filas/saúde/eventos/ingestão da Visão geral **não** levam o bairro; bairro desconhecido volta a "Todos"), `src/modules/panels.test.tsx:80,97` (Triagens: "< 5" e "oculto"; taxa oculta com `started` visível, sem NaN), `src/modules/Classification.test.tsx:157` (contagem "< 5", distribuição vira lista, amostra some), `Reports.test.tsx`, `Conversations.test.tsx`, `Overview.test.tsx`.
- admin: suíte inteira do `apps/admin` (console de `/admin/api`), para provar que não foi afetado.

**5. Execução.**
```
docker compose exec -T -w /rails/.claude/mod11 api bundle exec rspec \
  spec/invariants/territory_invariants_spec.rb spec/queries/admin/neighborhood_filter_spec.rb \
  spec/queries/admin/overview_classification_filter_spec.rb \
  spec/queries/admin/triages_reports_conversations_filter_spec.rb \
  spec/requests/admin/api/neighborhood_filter_spec.rb spec/requests/admin
→ 120 examples, 2 failures
```
As 2 falhas (`territory_invariants_spec.rb:204` e `:253`) foram `PG::TRDeadlockDetected` em `create_default_protocol!` (índice `idx_protocol_definitions_name_version_muni`), por disputa dos bancos de teste compartilhados com outros agentes rodando em paralelo. Não é defeito do código. Nova execução do arquivo:
```
docker compose exec -T -w /rails/.claude/mod11 api bundle exec rspec spec/invariants/territory_invariants_spec.rb
→ 17 examples, 0 failures
```
Resultado: 118/120 na primeira passada; os 2 que faltavam passaram na segunda (17/0 no arquivo). Registro do ledger na ponta (`2995f95`): suíte completa 2580/0, 4m49s (~0,112 s/ex).
```
cd apps/dashboard/.claude/mod11 && npx vitest run src/modules/neighborhoodPanels.test.tsx \
  src/components/NeighborhoodPicker.test.tsx src/lib/neighborhoodFilter.test.ts src/lib/smallCount.test.ts \
  src/components/StatTile.test.tsx src/modules/Classification.test.tsx src/modules/Reports.test.tsx \
  src/modules/Conversations.test.tsx src/modules/Overview.test.tsx src/modules/panels.test.tsx
→ 10 files, 66 tests passed
```
```
cd apps/admin && npx vitest run      # HEAD 9a8c0f6, somente leitura
→ 9 files, 46 tests passed
```
Prova no navegador feita pelo usuário (2026-09-29): Classificação filtrada por Batel mostrou "< 5" em todos os números e a amostra oculta com o texto neutro. O histórico do módulo também registra "oculto" em taxas na prova conjunta.

**6. Lacunas.**
- **Risco residual aceito (não bloqueia):** subtração entre campos da mesma resposta. Exemplos concretos no código atual: em Conversas, `live` = `liveActive.awaiting + liveActive.inProgress` (com `live` ≥ 5 e um dos dois ≥ 5, o outro, suprimido, sai por diferença); em Triagens/Visão geral, a soma dos pontos visíveis da série contra o total visível. A decisão está registrada na spec §10, no ADR 0023 (Revisão) e no ledger do api (Task 11/12), para rever com o módulo 14.
- **Ponto cego da varredura (mitigado):** o `small_numbers` da suíte de invariantes só olha `Numeric`. `avgToCompleteMin` sai como string (`BigDecimal#as_json`, ex. `"3.0"`), então a varredura geral não o veria. Um exemplo dedicado cobre o caso (`territory_invariants_spec.rb:253`, com mutação vermelha). Se surgir outro campo decimal suprimível, ele não fica coberto automaticamente.
- **Mutação por amostragem:** o comentário da varredura pede "tirar qualquer `@filter.*` de uma das cinco queries (uma por vez)", mas o ledger registra mutação só em `kpi_done`, `by_mode.share`, `avg_complete_minutes.over`, amostra/lista e ramo de protocolo. Os wrappers de Triagens e o `total` de Relatórios não têm mutação registrada; a fixture (bairro com 3 casos) deve pegá-los, mas isso não foi demonstrado.
- **Texto do ADR:** a seção "Consequências" do ADR 0023 fala em "contratos de **seis** painéis", e a Revisão/spec falam em **cinco** (o sexto seria o novo `GET /admin/api/neighborhoods`, que não é painel). Inconsistência de redação, sem efeito no código.
- **Menores de interface (ledger do dashboard, adiados):** `SUPPRESSED_HINT` cita "< 5" também na dica dos valores "oculto"; na Visão geral, os blocos filas/saúde/eventos/ingestão ficam sem filtro ao lado dos KPIs filtrados (por desenho, porque não são do cidadão, e testado). Não é violação de regra.
- Nenhum requisito do ADR/spec para F-11.7 ficou sem código ou sem teste.

**7. Veredito sugerido:** **Verified** (com o risco residual de subtração registrado como aceito).

---

## Critério de fechamento do módulo 11

Fonte: `modulos/11--territorio.md`, seção "Critério de fechamento do módulo".

**1. F-11.1 a F-11.7 verificadas.** F-11.7: Verified (acima). F-11.1 a F-11.6 dependem dos vereditos das outras partes deste dossiê. Este item só fecha quando todas estiverem Verified.

**2. Suíte de invariantes `spec/invariants/territory_invariants_spec.rb`, com teste de mutação.** O arquivo existe (320 linhas, 17 exemplos) e passa com 17/0 nesta verificação. A evidência de mutação está em `apps/api/.claude/mod11/.superpowers/sdd/2026-09-28-module-11-territory-api/task-16-report.md`: 12 mutações na entrega (`1d411a9`) e mais 2 na rodada de revisão (`5bb009e`), além da prova de que a fixture de jobs com falha pega a isenção acidental. Todas ficaram vermelhas e foram restauradas com `git status` limpo. Some-se a mutação do ramo de protocolo em Relatórios e o vermelho→verde de `urgentTrend` (`final-fix-report.md`). Cobertura de cada invariante do critério:

| Invariante do critério | Exemplo | Mutação registrada | Situação |
|---|---|---|---|
| Bairro da triagem imutável (exceto NULL na revogação) | L18 (UPDATE para outro bairro e para NULL fora da revogação levantam erro; NULL com `aborted_by_revocation` passa) + L32 (revogação zera o bairro) | #1 (trigger sem a condição de status), #2 (`neighborhood_id: nil` tirado do job) | ✓ coberto |
| Bairro inativo fora de escolhas novas | L44 (`ReplaceCoverage` → `inactive_neighborhood`; `Citizens::SetNeighborhood` → `invalid_neighborhood`) | #3 (só o ramo da cobertura) | ✓ coberto. Ressalva menor: o ramo do cidadão (`active_neighborhoods` em `SetNeighborhood`) é citado no comentário, mas não tem mutação registrada; a lista `GET /citizen/neighborhoods` só com ativos fica nas request specs, não nesta suíte |
| Referência sem unidade inativa | L59 (`ReferenceUnits.for` e `ids_by_neighborhood`) | #4 | ✓ coberto |
| Referência sem a própria unidade no desfecho | L71 (`reference_unit_ids` da fila exclui a unidade do atendimento) | #5 | ✓ coberto |
| Nenhum número de 1 a 4 com filtro ligado | L114-261 (varredura dos cinco painéis em três cenários + amostra/lista + share de modo + média) | #7, #8, #9 + 2 da rodada 1 (amostra e lista) | ✓ coberto. Ressalvas: varredura só `Numeric` (média decimal coberta à parte); mutação por amostragem, não em cada wrapper |
| Semente idempotente e sem desfazer edições | L266 (duas cargas iguais; depois de desativar, renomear e esvaziar cobertura, a carga não desfaz nada) | #10 | ✓ coberto |
| Relatório público sem bairro nem referência | L91 (fluxo real do cidadão → `GET /r/:token`, sem chave nem id/nome do bairro ou da unidade) | #6 | ✓ coberto |
| `api` sem chamada ao ViaCEP | L294 (varredura de `app/`, `lib/`, `config/`, `db/`) | #11 | ✓ coberto |
| (extra, invariante do ADR) Eventos só com ids | L303 | #12 | ✓ coberto |

Também registrados no ledger: suíte `spec/invariants` + `spec/architecture` com 276/0, e suíte completa 2580/0 em `2995f95`.

**3. Runbook `operacao/rollout-territorio.md` existe e bate com a realidade.** ✓ Conferido item por item:
- Migração `db/city_migrate/20260928100001_create_territory.rb` existe e é só de expansão.
- `city:migrate:all` existe (`lib/tasks/city.rake:276`).
- `city:territory:seed[slug]` e `city:territory:seed:all` existem (`lib/tasks/territory.rake:23,33`).
- As sementes `db/seeds/territory/curitiba.yml` e `maringa.yml` existem.
- A carga casa por `seed_key` e unidade não achada vira aviso (`app/services/territory/seed.rb:35,53`).
- Os commits citados são os de origin/main: api `2995f95`, dashboard `cbbb75b`, wpda `d1b552f`.
- A tela fica em "Cidade → Território" (`src/shell/modules.ts:41-42`).
- A nota de CSP bate com `nginx.conf:9-10` do dashboard.
- A reversão com `down`/`up` num banco descartável está registrada no `task-16-report.md`.
- A exceção do trigger para `aborted_by_revocation` existe em `db/city_triggers.sql:542`.

Nenhuma divergência encontrada.

**Veredito sugerido do critério de fechamento:** **Atendido nos itens 2 e 3**. O item 1 fica condicionado aos vereditos de F-11.1 a F-11.6 nas outras partes. Se todos forem Verified, o módulo 11 pode passar a **Fechado**. Ressalvas não bloqueantes: risco de subtração aceito, ponto cego `Numeric` da varredura mitigado por exemplo dedicado, mutação por amostragem nos painéis e no ramo do cidadão de "bairro inativo", e "seis" × "cinco" painéis no texto do ADR.
