# Módulo 15 — Triagem direcionada por perfil (api) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Perfil do par (nascimento, sexo, identidade de gênero; `declared`/`verified`), linguagem de condição com `gte`/`lte` e variáveis reservadas, oferta assinada no protocolo (`offer`) e sugestões (`suggestions`), catálogo da cidade (`triage_offers`), catálogo do cidadão, início de triagem por nome, sugestões na conclusão com expiração preguiçosa, simulador e contadores — o lado api de F-15.1 a F-15.8 (ADR 0027).

**Architecture:** Uma migração de cidade só de expansão acrescenta o perfil cifrado em `citizens` e cria `triage_offers`, `triage_suggestions` (trigger de transição, índice único parcial) e `triage_offer_daily_counts` (contador agregado de "oferecida"). A linguagem de condição continua pura (`Protocols::Condition` + `Protocols::ConditionContext`); o gate ganha `Validation::Offer` (variáveis por lugar). `Triages::Offer.evaluate` é uma função pura sobre dados já carregados; `Triages::Offer.for` só carrega e chama. `StartTriage` recebe o nome do protocolo, trava o cidadão e confere a oferta; `CompleteTriage` chama `Triages::Suggest`; `Triages::Catalog` monta o catálogo do cidadão, expira sugestões e conta "oferecida". O dashboard lê/escreve o catálogo por `TriageCatalogController` (step-up) e simula por `POST /authoring/protocols/simulate_offer`.

**Tech Stack:** Rails 8.1 (API), PostgreSQL 16 (banco por cidade), RSpec, json_schemer, Active Record Encryption (chave por cidade, ADR 0007).

**Spec:** `docs/.claude/mod15/superpowers/specs/2026-10-05-module-15-triage-catalog-design.md` e `docs/.claude/mod15/adr/0027.md` (leia os dois antes de começar). Contratos entre apps: `docs/.claude/mod15/superpowers/plans/2026-10-05-module-15-triage-catalog-contracts.md` — os planos do `contracts`, do wpda e do dashboard foram escritos contra ele: **não mude nomes, formatos nem códigos de erro**; as poucas precisões que o código real obrigou estão no fim, em "Divergências propostas ao contrato". O schema novo nasce no repo `contracts` (tag `protocols-v1.4.0`, outro plano); a Task 2 copia o arquivo de lá (o acréscimo está transcrito na Task 2 para conferência).

## Desvios da spec (e precisões de contrato)

Onde a spec é omissa ou o código real obrigou a escolher:

1. **Contador "oferecida" precisa de onde morar.** A spec (§7) conta "catálogos montados com ele `available`", mas nenhuma tabela guarda isso. Nasce `triage_offer_daily_counts (day, protocol_name, offered)`, agregada e **sem coluna de pessoa**: cada `GET /citizen/people/:id/catalog` soma 1 a cada protocolo mostrado como `available` ou `suggested` no dia (fuso da cidade). Não identifica ninguém e não precisa de limpeza na exclusão.
2. **Valores da condição viram texto no contexto.** `Protocols::Condition` compara `eq`/`in` por string e `gt`/`lt` por `Float()`. Para não mudar a semântica de respostas antigas, `Protocols::ConditionContext.build` grava as variáveis reservadas como texto (`"62"`, `"female"`, `"5"`); os operadores numéricos convertem como sempre. Respostas cujo id começa por prefixo reservado são descartadas do contexto (o gate já recusa esse id).
3. **Avisos do gate fora do motor puro.** "Sugestão para `name` que não existe hoje na cidade" exige banco; o motor (`app/protocols/`) é puro (`spec/invariants/protocol_engine_purity_spec.rb`). O aviso sai de `Protocols::SuggestionTargets.warnings` (`app/services/protocols/`) e aparece como `warnings` na resposta de `POST /authoring/protocols/gate` (só quando há aviso) e de `simulate_offer` — acréscimo ao contrato, ver Divergências.
4. **Protocolo inexistente ou inativo pedido pelo cidadão é `not_offered` (409)**, não `no_protocol` (503): para quem tem cidadão, a regra de oferta decide antes. `no_protocol` continua só para o WhatsApp (conversa sem cidadão) e para o nome padrão.
5. **Lock na ordem cidadão → conversa.** `Citizens::Erase` trava os pares e depois mexe nas conversas; `Citizens::StartConversation` passa a travar o cidadão **antes** da conversa (e `StartTriage` trava o mesmo cidadão de novo, sem custo, na mesma transação). `Triages::Suggest` (dentro do lock da conversa em `SubmitAnswer`) **não** trava o cidadão: a corrida entre duas conclusões é resolvida pelo índice único parcial (`ON CONFLICT` via `rescue RecordNotUnique` em savepoint).
6. **Protocolo em andamento fica fora de `suggested`/`available`** no catálogo do cidadão: ele aparece só em `in_progress` (pedir o mesmo protocolo retoma). A sugestão pendente dele não expira por isso.
7. **Validação presencial grava o perfil no par que ela marca `verified`** (contratos §4.4: "todos os pares do CPF que a validação marca `verified`"). Hoje `Citizens::Verify` marca exatamente um par — o do código —, então é nele que o perfil conferido é gravado; os outros pares do mesmo CPF não mudam. O check-in por código (`Citizens::Verify.record!`, usado por `Attendances::CheckIn`) não recebe perfil e não muda `profile_source`. Revogar a validação (`RevokeVerification`) não mexe no perfil.
8. **`gender_identity` na validação presencial:** chave ausente mantém o valor declarado; chave presente (inclusive `null`) grava o que veio.
9. **Intervalo de repetição** conta da data local (fuso da cidade) de `created_at` da última triagem `completed` do par naquele `protocol_name`: `next_available_on = last_completed_on + retake_after_days`; `recent` enquanto `hoje < next_available_on`.
10. **Contadores da aba do dashboard:** janela = hoje e os 29 dias anteriores (30 dias locais). `started` = triagens criadas com o `protocol_name` (todos os canais, inclusive revogadas, como o Analytics conta "iniciada"); `completed` = `Triage.counted_completed` (exclui revogada); `from_suggestion` = sugestões `taken` resolvidas na janela; `offered` = soma de `triage_offer_daily_counts`. Cada valor de 1 a 4 vira `null` (`Admin::SmallCount.small?`).
11. **Restrição da cidade:** além de árvore e variáveis, o tamanho do JSON é limitado a 4096 bytes (`invalid_restriction`). Uma restrição inválida gravada por fora (SQL) nunca derruba o catálogo do cidadão: a avaliação é total e dá falso (protocolo fora de oferta).
12. **`POST /citizen/conversations` com `cpf` (compatibilidade) continua**, mas um par novo nasce sem perfil e recebe 409 `profile_required`: na prática, o wpda novo usa `POST /citizen/people` e `citizen_id`. As specs existentes que começavam por CPF passam a criar o par com perfil antes (Task 10).

## Global Constraints

- Tudo de domínio no banco de cada cidade: migração em `db/city_migrate`, dump à mão em `db/city_schema.rb`, triggers em `db/city_triggers.sql` (o dump em Ruby não representa trigger; a migração termina com `execute File.read(Rails.root.join("db/city_triggers.sql"))`). Migração só de expansão. Rollout roda `city:migrate:all`; **nunca** migrar fora do rake (cidade trava em 503).
- `birth_date`, `sex`, `gender_identity` cifrados com a chave da cidade (`encrypts` **sem** `deterministic`, ADR 0007) e listados em `CityEncryption::CITY_KEYED_TARGETS` (`spec/architecture/city_encrypted_attributes_guard_spec.rb`). `profile_source` em claro, com `CHECK`. Idade **nunca** é gravada: `Citizen#age(on:)` calcula na leitura, no fuso da cidade (`Time.zone.today` dentro de `CityConnection.with`).
- Valores, exatamente: `sex` ∈ `female`, `male`; `gender_identity` ∈ `cis_woman`, `cis_man`, `trans_woman`, `trans_man`, `travesti`, `non_binary`, `other` ou `null`; `profile_source` ∈ `declared`, `verified`; `birth_date` `YYYY-MM-DD`, não futura, idade ≤ 130.
- Variáveis reservadas: prefixos `profile.`, `outcome.`, `citizen.`; variáveis `profile.age`, `profile.sex`, `outcome.tier`, `outcome.score`, `outcome.priority`, `citizen.neighborhood_id`. Por lugar: `offer.eligibility` → `profile.age`, `profile.sex`; `suggestions[].when` → `profile.age`, `profile.sex`, `outcome.tier`, `outcome.score`, `outcome.priority` + ids de passo do protocolo; restrição do catálogo → `profile.age`, `profile.sex`, `citizen.neighborhood_id`; qualquer outro lugar (decision_table, priority_when) → nenhuma.
- A linguagem continua **total**: variável ausente, operando malformado ou nó inválido → `false`, nunca exceção. A chamada antiga `Protocols::Condition.eval(node, answers)` continua valendo.
- `gte`/`lte` passam a valer em **todo** lugar de condição assim que a cópia do schema for atualizada: a Task 1 (avaliação e validação de `gte`/`lte`) vem **antes** da Task 2 (cópia do schema). `suggestions[].protocol` segue o mesmo padrão do `name`: `^[a-z][a-z0-9-]+$`.
- Eventos de domínio novos, exatamente e só com ids: `citizen.profile_changed` `{ citizen_id }`, `triage.suggested` `{ triage_id, suggestion_id, protocol_name }`, `triage_offer.changed` `{ protocol_name, user_id }`; declarados em `config/initializers/domain_events.rb` com `to: []` e na `spec/initializers/domain_events_bindings_spec.rb`. Nenhum `Platform.audit` novo (logo, nada em `R18_PLATFORM_EVENT_NAMES`). Nenhum evento, log ou URL carrega data de nascimento, idade, sexo ou identidade de gênero; `filter_parameters` ganha `:birth_date`, `:sex`, `:gender_identity`.
- Erros sempre `{ "error": "<reason>" }`; step-up pelo `MfaStepUp` existente (401 `{ "error": "mfa_required" }`, contratos §4.2); papel ausente 403 `missing_role`.
- `Current.city` **nunca** é atribuído em `app/` ou `lib/` (`spec/architecture/current_city_assignment_spec.rb`); semente usa o `Current.set`/`CityConnection.with` que `db/seeds.rb` já abre.
- Catálogo vazio (nenhuma linha em `triage_offers`, nenhum `offer` nos protocolos) = comportamento de hoje: todo protocolo ativo sem `offer.eligibility` está em oferta.
- Specs de request precisam de `type: :request` (inferência desligada). Arquivo novo em `spec/support/` precisa de `require_relative` em `spec/rails_helper.rb`. Nada de data fixa contra o relógio real em spec de banco: datas derivadas de `Time.zone.today`; a função pura (`Triages::Offer.evaluate`) recebe `on:` e pode usar data fixa.
- `spec/adr_pointers_spec.rb`: `VALID_RANGE = (1..27)`.
- Commits em inglês, Conventional Commits com o tipo por extenso (`feat`, `fix`, `refactor`, `test`, `chore`...), terminando com a linha `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta).

## Ambiente de execução

- Antes de tocar `apps/api`, avise a sessão dona do api (sessão "API"). Ordem de merge: `contracts` (tag `protocols-v1.4.0`) → api → wpda e dashboard.
- Crie o worktree (a partir da raiz do monorepo, `/Users/eduardovrocha/Development/ioit.solutions/rota-saude`):

  ```bash
  /opt/homebrew/bin/git -C apps/api fetch origin
  /opt/homebrew/bin/git -C apps/api worktree add .claude/mod15 -b feat/mod-15-triage-catalog origin/main
  cp apps/api/config/master.key apps/api/.claude/mod15/config/master.key
  ```

- `./apps/api` é montado em `/rails` no container do serviço `api` (`docker-compose.yml` da raiz); o worktree é `/rails/.claude/mod15`. Todo comando Rails/RSpec roda no container, a partir da raiz do monorepo:

  ```bash
  docker compose exec -T -w /rails/.claude/mod15 api bundle exec rspec <arquivos>
  ```

- Todo `git add`/`git commit` usa `-C apps/api/.claude/mod15` (caminhos relativos ao worktree).
- Depois da migração de cidade (Task 4): `DROP DATABASE` dos dois bancos de teste de cidade (`rota_saude_test_city_a`, `rota_saude_test_city_b`) e `docker compose exec -T -w /rails/.claude/mod15 api bin/rails city:test_databases`; a paridade (`spec/services/city_schema_spec.rb`) quebra se o banco de teste estiver fora do schema. Ao voltar para a main, repita (o banco de teste fica à frente).
- Suíte completa só com o worker parado e sem outra sessão rodando suíte completa:

  ```bash
  docker compose stop worker
  docker compose exec -T -w /rails/.claude/mod15 api bundle exec rspec
  docker compose start worker
  ```

- Prova no navegador e os planos do wpda/dashboard: o api do worktree sobe na porta **3032** (Task 21), sem derrubar o servidor principal.

## Review Focus

1. **Aniversário e virada do dia no fuso da cidade:** quem faz 60 anos hoje vê "Saúde do idoso" desde a meia-noite local (não a UTC), quem faz amanhã não vê, e quem nasceu em 29/02 faz aniversário em 01/03 nos anos comuns. Testes: Task 5 ("idade no aniversário, na véspera e em 29/02") e Task 9 ("faz 60 hoje no fuso de Manaus").
2. **Pares que dividem celular ou CPF não se enxergam:** avó e neto no mesmo celular têm catálogos diferentes; o mesmo CPF em dois celulares não cruza perfil nem sugestão; o histórico de par verificado mostra a triagem do outro par **sem** sugestões. Testes: Task 12 ("dois pares no mesmo celular", "mesmo CPF em dois celulares"), Task 13 ("triagem de outro par do CPF verificado: suggestions []"), Task 19 (invariante 7).
3. **A cidade pausa (ou o período acaba) depois da sugestão:** a sugestão pendente vira `expired` na próxima leitura do catálogo, some de `GET /citizen/triages/:id` e iniciar responde 409 `not_offered`. Testes: Task 12 ("pausa expira a sugestão"), Task 13 ("protocolo pausado some do resultado"), Task 10 ("pausado: not_offered").
4. **Dois toques ou duas abas iniciando ao mesmo tempo:** o mesmo protocolo gera uma triagem só (a outra retoma); protocolos diferentes geram uma triagem e um 409 `triage_in_progress`. Testes: Task 10 (spec com threads).
5. **Restrição malformada vinda do dashboard ou do banco:** operador desconhecido, variável de passo, `outcome.*`, sexo inválido, uuid inválido, JSON acima de 4096 bytes → 422 `invalid_restriction`, nada gravado; uma restrição quebrada gravada por SQL deixa o protocolo fora de oferta, sem 500 no catálogo. Testes: Task 14 (tabela de restrições inválidas) e Task 9 ("restrição quebrada no banco").

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| `app/protocols/condition.rb`, `app/protocols/condition_context.rb`, `app/protocols/validation/condition.rb` | `gte`/`lte`, contexto com variáveis reservadas, validação por lugar | 1 |
| `config/protocols/schema.json`, `app/protocols/validation/offer.rb`, `app/protocols/gate.rb` | schema 1.4.0 e gate de `offer`/`suggestions` | 2 |
| `app/services/protocols/suggestion_targets.rb`, `app/controllers/authoring/protocols_controller.rb` | aviso de protocolo sugerido inexistente | 3 |
| `db/city_migrate/20261005100001_add_triage_catalog.rb`, `db/city_schema.rb`, `db/city_triggers.sql`, `app/models/triage_offer.rb`, `app/models/triage_suggestion.rb`, `app/models/triage_offer_daily_count.rb`, `spec/support/triage_catalog_helpers.rb`, `spec/adr_pointers_spec.rb` | dados | 4 |
| `app/models/citizen.rb`, `app/services/city_encryption.rb`, `config/initializers/filter_parameter_logging.rb`, `app/commands/citizens/profile_json.rb` | perfil no modelo | 5 |
| `app/commands/citizens/profile_values.rb`, `app/commands/citizens/set_profile.rb` | perfil declarado | 6 |
| `app/controllers/citizen_api/people_controller.rb`, `app/controllers/citizen_api/base_controller.rb`, `app/commands/citizens/register_person.rb` | API do perfil | 7 |
| `app/commands/citizens/verify.rb`, `app/controllers/attendance_controller.rb` | perfil conferido no balcão | 8 |
| `app/services/triages/offer.rb` | regra de oferta | 9 |
| `app/commands/start_triage.rb`, `app/commands/citizens/start_conversation.rb`, `app/controllers/citizen_api/conversations_controller.rb` | início por nome | 10 |
| `app/commands/triages/suggest.rb`, `app/commands/complete_triage.rb` | sugestões na conclusão | 11 |
| `app/services/triages/catalog.rb`, `config/routes.rb` | catálogo do cidadão | 12 |
| `app/controllers/citizen_api/triages_controller.rb` | sugestões no resultado | 13 |
| `app/commands/triages/set_offer.rb` | linha do catálogo da cidade | 14 |
| `app/services/triages/counters.rb` | contadores com supressão | 15 |
| `app/services/triages/catalog_admin.rb`, `app/controllers/triage_catalog_controller.rb`, `app/policies/protocol_policy.rb` | aba do dashboard | 16 |
| `app/services/protocols/condition_text.rb`, `app/services/protocols/simulate_offer.rb` | simulador | 17 |
| `app/commands/citizens/erase.rb`, `app/commands/revoke_consent.rb` | LGPD | 18 |
| `spec/invariants/triage_catalog_invariants_spec.rb` | invariantes do ADR 0027 | 19 |
| `lib/triage_catalog_crew.rb`, `db/seeds.rb` | semente de dev | 20 |

---

## Fatia 1 — Linguagem de condição e schema (F-15.2, F-15.3)

### Task 1: `gte`/`lte`, contexto com variáveis reservadas e validação por lugar

**Files:**
- Modify: `app/protocols/condition.rb`
- Create: `app/protocols/condition_context.rb`
- Modify: `app/protocols/validation/condition.rb`
- Modify: `spec/invariants/protocol_engine_purity_spec.rb` (núcleo puro ganha `condition_context.rb`)
- Test: `spec/protocols/condition_spec.rb`, `spec/protocols/condition_context_spec.rb`, `spec/protocols/validation/condition_spec.rb`

**Interfaces:**
- Produces:
  - `Protocols::Condition::OPERATORS == %w[eq in gt lt gte lte all any not]`; `Protocols::Condition.eval(node, context) -> Boolean` (total).
  - `Protocols::ConditionContext::RESERVED_PREFIXES == %w[profile. outcome. citizen.]`; `Protocols::ConditionContext.reserved?(name) -> Boolean`; `Protocols::ConditionContext.build(answers: {}, profile: {}, outcome: {}, citizen: {}) -> Hash{String => Object}` (perfil `{ age:, sex: }`, resultado `{ tier:, score:, priority: }`, cidadão `{ neighborhood_id: }`, chaves símbolo ou string; valores reservados como texto; `nil` some).
  - `Protocols::Validation::Condition.errors(node, by_id, variables: []) -> Array<String>`; `Protocols::Validation::Condition::VARIABLES` (nome → `:number`/`:sex`/`:text`/`:uuid`).

- [ ] **Step 1: Escreva as specs que falham**

Acrescente ao fim de `spec/protocols/condition_spec.rb` (antes do `end` final):

```ruby
  # ADR 0027: gte/lte com o mesmo guarda de gt/lt.
  it "gte/lte: inclusive numeric comparison, total on bad operands" do
    expect(ev({ "gte" => ["idade", 70] })).to be(true)
    expect(ev({ "gte" => ["idade", 71] })).to be(false)
    expect(ev({ "lte" => ["idade", 70] })).to be(true)
    expect(ev({ "lte" => ["idade", 69] })).to be(false)
    expect(ev({ "gte" => ["ausente", 1] })).to be(false)
    expect(ev({ "gte" => ["febre", 1] })).to be(false)
    expect(ev({ "gte" => ["idade", "abc"] })).to be(false)
    expect(ev({ "gte" => "idade" })).to be(false)
    expect(ev({ "lte" => ["idade"] })).to be(false)
  end

  it "evaluates reserved variables from a ConditionContext and keeps the answers-only call" do
    context = Protocols::ConditionContext.build(
      answers: answers, profile: { age: 60, sex: "female" }, outcome: { tier: "media", score: 15, priority: 5 }
    )
    expect(described_class.eval({ "all" => [{ "gte" => ["profile.age", 60] }, { "eq" => ["profile.sex", "female"] }] }, context)).to be(true)
    expect(described_class.eval({ "gte" => ["outcome.score", 15] }, context)).to be(true)
    expect(described_class.eval({ "eq" => ["outcome.priority", 5] }, context)).to be(true)
    expect(described_class.eval({ "eq" => ["febre", "true"] }, context)).to be(true)
    expect(described_class.eval({ "gte" => ["profile.age", 60] }, answers)).to be(false)
  end
```

Crie `spec/protocols/condition_context_spec.rb`:

```ruby
require "rails_helper"

# ADR 0027 (spec 2026-10-05 §4.1): o contexto plano junta respostas e variáveis
# reservadas. Valores reservados viram texto (eq/in comparam texto; gt/gte
# convertem com Float), nil some (variável ausente → falso), e uma resposta com
# id de prefixo reservado nunca sobrescreve a variável.
RSpec.describe Protocols::ConditionContext do
  it "builds a flat context with stringified reserved variables" do
    context = described_class.build(
      answers: { "q1" => "true", "profile.age" => "999" },
      profile: { age: 62, sex: "female" },
      outcome: { tier: "media", score: 15, priority: 5 },
      citizen: { neighborhood_id: "0b6f6c1e-9f1a-4d8b-9a4c-1f2e3d4c5b6a" }
    )
    expect(context).to eq(
      "q1" => "true", "profile.age" => "62", "profile.sex" => "female",
      "outcome.tier" => "media", "outcome.score" => "15", "outcome.priority" => "5",
      "citizen.neighborhood_id" => "0b6f6c1e-9f1a-4d8b-9a4c-1f2e3d4c5b6a"
    )
  end

  it "drops nil values and accepts string keys" do
    context = described_class.build(profile: { "age" => nil, "sex" => "male" }, outcome: nil, citizen: "x")
    expect(context).to eq("profile.sex" => "male")
  end

  it "knows the reserved prefixes" do
    expect(described_class.reserved?("profile.age")).to be(true)
    expect(described_class.reserved?("outcome.x")).to be(true)
    expect(described_class.reserved?("citizen.neighborhood_id")).to be(true)
    expect(described_class.reserved?("profiles")).to be(false)
  end
end
```

Acrescente ao fim de `spec/protocols/validation/condition_spec.rb` (antes do `end` final):

```ruby
  describe "gte/lte and reserved variables (ADR 0027)" do
    it "accepts gte/lte on an integer step and rejects them elsewhere" do
      expect(errs({ "gte" => ["idade", 60] })).to eq([])
      expect(errs({ "lte" => ["idade", 5] })).to eq([])
      expect(errs({ "gte" => ["febre", 1] })).to include(a_string_including("requires an integer step"))
      expect(errs({ "lte" => ["idade", "x"] })).to include("condition 'lte' threshold must be numeric")
    end

    it "rejects any reserved variable when the place allows none (decision_table, priority_when)" do
      expect(errs({ "gte" => ["profile.age", 60] })).to eq(["condition variable 'profile.age' is not allowed here"])
      expect(errs({ "eq" => ["outcome.tier", "alta"] })).to eq(["condition variable 'outcome.tier' is not allowed here"])
    end

    def place(node, variables) = described_class.errors(node, by_id, variables: variables)

    it "checks type and value of an allowed variable" do
      vars = %w[profile.age profile.sex outcome.tier outcome.score citizen.neighborhood_id]
      expect(place({ "gte" => ["profile.age", 60] }, vars)).to eq([])
      expect(place({ "in" => ["profile.sex", %w[female male]] }, vars)).to eq([])
      expect(place({ "eq" => ["outcome.tier", "qualquer"] }, vars)).to eq([])
      expect(place({ "in" => ["citizen.neighborhood_id", ["0b6f6c1e-9f1a-4d8b-9a4c-1f2e3d4c5b6a"]] }, vars)).to eq([])
      expect(place({ "eq" => ["profile.sex", "outro"] }, vars)).to eq(["condition 'eq' invalid value 'outro' for profile.sex"])
      expect(place({ "gt" => ["profile.sex", 1] }, vars)).to eq(["condition 'gt' requires a numeric variable, got profile.sex"])
      expect(place({ "in" => ["citizen.neighborhood_id", ["nao-uuid"]] }, vars))
        .to eq(["condition 'in' invalid value 'nao-uuid' for citizen.neighborhood_id"])
      expect(place({ "eq" => ["profile.age", "sessenta"] }, vars)).to eq(["condition 'eq' invalid value 'sessenta' for profile.age"])
      expect(place({ "gte" => ["profile.height", 1] }, vars)).to eq(["condition variable 'profile.height' is not allowed here"])
    end

    it "treats a reserved key in a legacy map as eq, and rejects empty all/any" do
      expect(place({ "profile.sex" => "female" }, %w[profile.sex])).to eq([])
      expect(place({ "profile.sex" => "x" }, %w[profile.sex])).to eq(["condition 'eq' invalid value 'x' for profile.sex"])
      expect(errs({ "all" => [] })).to eq(["condition 'all' operand must be a non-empty array"])
      expect(errs({ "any" => "x" })).to eq(["condition 'any' operand must be a non-empty array"])
    end
  end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod15 api bundle exec rspec spec/protocols/condition_spec.rb spec/protocols/condition_context_spec.rb spec/protocols/validation/condition_spec.rb`
Expected: FAIL (`uninitialized constant Protocols::ConditionContext`; `gte` dá `false`; `unknown keyword: :variables`).

- [ ] **Step 3: Escreva o contexto**

```ruby
# app/protocols/condition_context.rb
# Contexto plano da linguagem de condição (ADR 0009, ADR 0027; spec 2026-10-05
# §4.1). Módulo puro: quem chama calcula a idade (Citizen#age) e passa valores.
# Variáveis reservadas viram texto — eq/in comparam texto e gt/gte/lt/lte
# convertem com Float, como sempre fizeram com as respostas. nil não entra
# (variável ausente → condição falsa). Resposta cujo id começa por prefixo
# reservado é descartada: o gate recusa esse id, e a variável sempre vence.
module Protocols
  module ConditionContext
    RESERVED_PREFIXES = %w[profile. outcome. citizen.].freeze

    module_function

    def reserved?(name) = name.to_s.start_with?(*RESERVED_PREFIXES)

    def build(answers: {}, profile: {}, outcome: {}, citizen: {})
      context = {}
      (answers.is_a?(Hash) ? answers : {}).each { |key, value| context[key.to_s] = value unless reserved?(key) }
      profile = symbolize(profile)
      outcome = symbolize(outcome)
      citizen = symbolize(citizen)
      put(context, "profile.age", profile[:age])
      put(context, "profile.sex", profile[:sex])
      put(context, "outcome.tier", outcome[:tier])
      put(context, "outcome.score", outcome[:score])
      put(context, "outcome.priority", outcome[:priority])
      put(context, "citizen.neighborhood_id", citizen[:neighborhood_id])
      context
    end

    def symbolize(hash) = hash.is_a?(Hash) ? hash.transform_keys(&:to_sym) : {}

    def put(context, key, value)
      context[key] = value.to_s unless value.nil?
    end
  end
end
```

- [ ] **Step 4: Ensine `gte`/`lte` ao avaliador**

Substitua `app/protocols/condition.rb` inteiro por:

```ruby
# Avaliador de condições do motor de protocolos. Módulo puro — ver ADR-0009.
# eq/in/gt/lt/gte/lte/all/any/not; um `when` que não seja nó-operador de chave
# única é tratado como mapa legado {step_id => value} (AND de eq). Runtime
# total: nunca levanta — operando ausente/não-numérico ou nó inválido => false.
# Cada branch de operador é type-guarded (operand precisa ser Array/Hash conforme
# o operador); operando malformado (nil, String, Integer, array curto) => false.
#
# ADR 0027: o segundo argumento é um contexto plano — as respostas
# {step_id => valor}, como sempre, ou o hash de Protocols::ConditionContext,
# que junta as variáveis reservadas profile.*/outcome.*/citizen.*.
module Protocols
  module Condition
    OPERATORS = %w[eq in gt lt gte lte all any not].freeze

    module_function

    def eval(node, context)
      return false unless node.is_a?(Hash)
      return legacy_all_eq(node, context) unless operator_node?(node)

      op, operand = node.first
      case op.to_s
      when "eq"  then operand.is_a?(Array) && context[operand[0].to_s] == operand[1].to_s
      when "in"  then operand.is_a?(Array) && Array(operand[1]).map(&:to_s).include?(context[operand[0].to_s])
      when "gt"  then compare(context, operand) { |value, threshold| value > threshold }
      when "lt"  then compare(context, operand) { |value, threshold| value < threshold }
      when "gte" then compare(context, operand) { |value, threshold| value >= threshold }
      when "lte" then compare(context, operand) { |value, threshold| value <= threshold }
      when "all" then operand.is_a?(Array) && operand.all? { |n| eval(n, context) }
      when "any" then operand.is_a?(Array) && operand.any? { |n| eval(n, context) }
      when "not" then operand.is_a?(Hash) && !eval(operand, context)
      else false
      end
    end

    # NOTE: colisão conhecida — um `when` legado de chave única para um step com
    # nome de operador (ex.: {"eq" => "true"}) é lido como o OPERADOR eq, não como
    # mapa legado. O gate recusa step id == nome de operador
    # (Validation::Condition.step_id_collision_errors). Runtime é total.
    def operator_node?(node)
      node.size == 1 && OPERATORS.include?(node.keys.first.to_s)
    end

    def legacy_all_eq(map, context)
      return false if map.empty?
      map.all? { |step_id, expected| context[step_id.to_s] == expected.to_s }
    end

    def compare(context, operand)
      return false unless operand.is_a?(Array) && operand.size >= 2

      numeric(context[operand[0].to_s]) { |value| yield value, Float(operand[1]) }
    end

    def numeric(raw)
      yield Float(raw)
    rescue ArgumentError, TypeError
      false
    end
  end
end
```

(`compare` exige 2 elementos: `{"lte" => ["idade"]}` dá falso. O `Float(operand[1])` que levanta dentro do bloco cai no `rescue` de `numeric`.)

- [ ] **Step 5: Validação por lugar**

Substitua `app/protocols/validation/condition.rb` inteiro por:

```ruby
# Valida um nó de condição (when) no PUBLISH (Gate). Semântico, além do JSON
# Schema: operador conhecido, gt/lt/gte/lte só em step integer ou variável
# numérica, eq/in no conjunto permitido, step referenciado existe. Mapa legado
# {step=>val} = validação por-par. Ver ADR-0009. (chore de validação —
# F-03.2/F-03.6)
#
# ADR 0027 (spec 2026-10-05 §4.3): `variables:` diz quais variáveis reservadas
# o LUGAR aceita (contratos §1). O padrão é nenhuma: decision_table e
# priority_when continuam só com passos.
module Protocols
  module Validation
    module Condition
      OPERATORS = Protocols::Condition::OPERATORS
      NUMERIC_OPERATORS = %w[gt lt gte lte].freeze
      VARIABLES = {
        "profile.age" => :number, "profile.sex" => :sex, "outcome.tier" => :text,
        "outcome.score" => :number, "outcome.priority" => :number, "citizen.neighborhood_id" => :uuid
      }.freeze
      SEXES = %w[female male].freeze
      UUID = /\A\h{8}-\h{4}-\h{4}-\h{4}-\h{12}\z/

      module_function

      def errors(node, by_id, variables: [])
        return ["condition must be a non-empty object"] unless node.is_a?(Hash) && node.any?
        return legacy_errors(node, by_id, variables) unless operator_node?(node)

        op, operand = node.first
        case op.to_s
        when "eq", "in" then eq_in_errors(op.to_s, operand, by_id, variables)
        when *NUMERIC_OPERATORS then numeric_errors(op.to_s, operand, by_id, variables)
        when "all", "any"
          return ["condition '#{op}' operand must be a non-empty array"] unless operand.is_a?(Array) && operand.any?
          operand.flat_map { |sub| errors(sub, by_id, variables: variables) }
        when "not" then errors(operand, by_id, variables: variables)
        else ["unknown condition operator '#{op}'"]
        end
      end

      def step_id_collision_errors(steps)
        Array(steps).filter_map do |s|
          "step id '#{s["id"]}' collides with a condition operator" if OPERATORS.include?(s["id"].to_s)
        end
      end

      def operator_node?(node)
        node.size == 1 && OPERATORS.include?(node.keys.first.to_s)
      end

      def reserved?(name) = Protocols::ConditionContext.reserved?(name)

      def eq_in_errors(op, operand, by_id, variables)
        return ["condition '#{op}' operand must be [step_id, value]"] unless operand.is_a?(Array) && operand.size == 2
        step_id, value = operand
        values = op == "in" ? Array(value) : [value]
        return variable_value_errors(op, step_id.to_s, values, variables) if reserved?(step_id)

        step = by_id[step_id.to_s]
        return ["condition '#{op}' references unknown step #{step_id}"] if step.nil?
        allowed = Answers.for(step)
        return [] if allowed.nil?
        values.map(&:to_s).reject { |v| allowed.include?(v) }
              .map { |v| "condition '#{op}' invalid answer '#{v}' for step #{step_id}" }
      end

      def numeric_errors(op, operand, by_id, variables)
        return ["condition '#{op}' operand must be [step_id, number]"] unless operand.is_a?(Array) && operand.size == 2
        name, threshold = operand
        errs = []
        if reserved?(name)
          errs.concat(variable_errors(name.to_s, variables))
          errs << "condition '#{op}' requires a numeric variable, got #{name}" if errs.empty? && VARIABLES[name.to_s] != :number
        else
          step = by_id[name.to_s]
          return ["condition '#{op}' references unknown step #{name}"] if step.nil?
          errs << "condition '#{op}' requires an integer step, got #{step["answer_type"]} for #{name}" unless step["answer_type"] == "integer"
        end
        errs << "condition '#{op}' threshold must be numeric" unless numeric?(threshold)
        errs
      end

      def variable_errors(name, variables)
        variables.include?(name) && VARIABLES.key?(name) ? [] : ["condition variable '#{name}' is not allowed here"]
      end

      def variable_value_errors(op, name, values, variables)
        errs = variable_errors(name, variables)
        return errs if errs.any?

        valid = case VARIABLES[name]
                when :sex then ->(v) { SEXES.include?(v.to_s) }
                when :uuid then ->(v) { v.to_s.match?(UUID) }
                when :number then ->(v) { numeric?(v) }
                else ->(_v) { true }
                end
        values.reject { |v| valid.call(v) }.map { |v| "condition '#{op}' invalid value '#{v}' for #{name}" }
      end

      def numeric?(value)
        value.is_a?(Numeric) || !Float(value.to_s, exception: false).nil?
      end

      def legacy_errors(map, by_id, variables)
        map.flat_map do |step_id, answer|
          next variable_value_errors("eq", step_id.to_s, [answer], variables) if reserved?(step_id)

          step = by_id[step_id.to_s]
          next ["decision_table rule references unknown step #{step_id}"] if step.nil?
          allowed = Answers.for(step)
          next [] if allowed.nil?
          allowed.include?(answer.to_s) ? [] : ["decision_table invalid answer '#{answer}' for step #{step_id}"]
        end
      end
    end
  end
end
```

- [ ] **Step 6: Ponha o contexto no núcleo puro**

Em `spec/invariants/protocol_engine_purity_spec.rb`, no método `core`, acrescente `condition_context.rb` à lista:

```ruby
    def core
      %w[protocol.rb step.rb outcome.rb condition.rb condition_context.rb priority_rules.rb scoring.rb
         scoring/weighted.rb scoring/decision_table.rb]
    end
```

- [ ] **Step 7: Rode as specs da task e as do motor**

Run: `docker compose exec -T -w /rails/.claude/mod15 api bundle exec rspec spec/protocols spec/invariants/protocol_engine_purity_spec.rb spec/invariants/clinical_regression_spec.rb`
Expected: PASS (as specs antigas de `Validation::Condition`, `Scoring` e `PriorityWhen` continuam verdes: o padrão `variables: []` não muda nada para passos).

- [ ] **Step 8: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod15 add app/protocols/condition.rb app/protocols/condition_context.rb app/protocols/validation/condition.rb spec/protocols/condition_spec.rb spec/protocols/condition_context_spec.rb spec/protocols/validation/condition_spec.rb spec/invariants/protocol_engine_purity_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod15 commit -m "feat: add gte/lte and reserved condition variables to the protocol language

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 2: Schema `protocols-v1.4.0` e gate de `offer`/`suggestions`

Depende da Task 1: depois desta cópia, `gte`/`lte` passam no schema em qualquer condição, e o avaliador e o validador já os conhecem.

**Files:**
- Modify: `config/protocols/schema.json`
- Create: `app/protocols/validation/offer.rb`
- Modify: `app/protocols/validation/condition.rb` (prefixo reservado em id de passo)
- Modify: `app/protocols/gate.rb`
- Test: `spec/protocols/schema_offer_spec.rb`, `spec/protocols/validation/offer_spec.rb`, `spec/protocols/gate_spec.rb`

**Interfaces:**
- Consumes: `Protocols::Validation::Condition.errors(node, by_id, variables:)`, `Protocols::ConditionContext.reserved?` (Task 1).
- Produces:
  - `Protocols::Validation::Offer::ELIGIBILITY == %w[profile.age profile.sex]`, `::SUGGESTION == %w[profile.age profile.sex outcome.tier outcome.score outcome.priority]`, `::RESTRICTION == %w[profile.age profile.sex citizen.neighborhood_id]`.
  - `Protocols::Validation::Offer.call(definition) -> Array<String>` (total para qualquer entrada).
  - `Protocols::Validation::Condition.reserved_prefix_errors(steps) -> Array<String>`.
  - `Protocols::Gate.call(definition)` recusa: variável fora do lugar, passo inexistente no `when`, sugestão para o próprio `name`, `retake_after_days` < 1, id de passo com prefixo reservado.

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/protocols/schema_offer_spec.rb
require "rails_helper"
require "json_schemer"

# protocols-v1.4.0 (ADR 0027; contratos §1): offer e suggestions opcionais;
# gte/lte em toda condição. A cópia em config/protocols/schema.json é idêntica
# à do contracts.
RSpec.describe "protocols schema.json offer/suggestions contract (v1.4.0)" do
  let(:schema) { JSONSchemer.schema(JSON.parse(File.read(Rails.root.join("config/protocols/schema.json")))) }

  def base(extra = {})
    {
      "name" => "saude-do-idoso", "version" => 1, "start_step_id" => "s1",
      "steps" => [ { "id" => "s1", "prompt" => "?", "answer_type" => "integer", "branches" => {}, "weights" => {} } ],
      "scoring" => { "type" => "weighted", "thresholds" => { "baixa" => 0 }, "priority_map" => { "baixa" => 9 } }
    }.merge(extra)
  end

  it "aceita protocolo sem offer nem suggestions (1.3.0 continua válido)" do
    expect(schema.valid?(base)).to be(true)
  end

  it "aceita offer completo e suggestions com gte/lte" do
    definition = base(
      "offer" => { "title" => "Saúde do idoso", "summary" => "Avaliação anual.",
                   "eligibility" => { "gte" => ["profile.age", 60] }, "retake_after_days" => 365 },
      "suggestions" => [ { "protocol" => "saude-mental-aprofundada", "when" => { "lte" => ["outcome.score", 3] } } ]
    )
    expect(schema.valid?(definition)).to be(true)
  end

  it "aceita gte/lte também em priority_when" do
    definition = base("priority_when" => [ { "when" => { "gte" => ["s1", 3] }, "priority" => 2 } ])
    expect(schema.valid?(definition)).to be(true)
  end

  it "recusa campo desconhecido, título longo, intervalo zero e nome de sugestão fora do padrão do name" do
    expect(schema.valid?(base("offer" => { "audience" => "todos" }))).to be(false)
    expect(schema.valid?(base("offer" => { "title" => "x" * 61 }))).to be(false)
    expect(schema.valid?(base("offer" => { "summary" => "x" * 201 }))).to be(false)
    expect(schema.valid?(base("offer" => { "retake_after_days" => 0 }))).to be(false)
    expect(schema.valid?(base("offer" => { "retake_after_days" => 3651 }))).to be(false)
    expect(schema.valid?(base("suggestions" => [ { "protocol" => "9-comeca-com-digito", "when" => { "eq" => ["s1", "1"] } } ]))).to be(false)
    expect(schema.valid?(base("suggestions" => [ { "protocol" => "Maiuscula", "when" => { "eq" => ["s1", "1"] } } ]))).to be(false)
    expect(schema.valid?(base("suggestions" => [ { "protocol" => "x" } ]))).to be(false)
    expect(schema.valid?(base("suggestions" => Array.new(11) { { "protocol" => "outro", "when" => { "eq" => ["s1", "1"] } } }))).to be(false)
  end
end
```

```ruby
# spec/protocols/validation/offer_spec.rb
require "rails_helper"

# ADR 0027 (spec 2026-10-05 §4.3; contratos §1): cada lugar aceita só as suas
# variáveis; sugestão nunca aponta para o próprio protocolo.
RSpec.describe Protocols::Validation::Offer do
  def definition(offer: nil, suggestions: nil)
    {
      "name" => "saude-mental", "version" => 1, "start_step_id" => "humor",
      "steps" => [ { "id" => "humor", "prompt" => "?", "answer_type" => "integer" },
                   { "id" => "sono", "prompt" => "?", "answer_type" => "boolean" } ],
      "offer" => offer, "suggestions" => suggestions
    }.compact
  end

  it "aceita elegibilidade de perfil e sugestão sobre perfil, resultado e passos" do
    errors = described_class.call(definition(
      offer: { "eligibility" => { "all" => [ { "gte" => ["profile.age", 18] }, { "eq" => ["profile.sex", "female"] } ] } },
      suggestions: [ { "protocol" => "saude-mental-aprofundada",
                       "when" => { "any" => [ { "gte" => ["outcome.score", 15] }, { "eq" => ["sono", "false"] },
                                              { "gte" => ["humor", 7] }, { "eq" => ["outcome.tier", "alta"] } ] } } ]
    ))
    expect(errors).to eq([])
  end

  it "recusa na elegibilidade resultado, bairro e passo" do
    errors = described_class.call(definition(offer: { "eligibility" => { "any" => [
      { "gte" => ["outcome.score", 1] }, { "in" => ["citizen.neighborhood_id", []] }, { "eq" => ["sono", "true"] }
    ] } }))
    expect(errors).to eq([
      "offer.eligibility: condition variable 'outcome.score' is not allowed here",
      "offer.eligibility: condition variable 'citizen.neighborhood_id' is not allowed here",
      "offer.eligibility: condition 'eq' references unknown step sono"
    ])
  end

  it "recusa na sugestão bairro, passo inexistente e o próprio protocolo" do
    errors = described_class.call(definition(suggestions: [
      { "protocol" => "saude-mental", "when" => { "gte" => ["outcome.score", 1] } },
      { "protocol" => "outro", "when" => { "in" => ["citizen.neighborhood_id", []] } },
      { "protocol" => "outro", "when" => { "eq" => ["fantasma", "1"] } }
    ]))
    expect(errors).to eq([
      "suggestions[0]: suggestion points to the protocol itself",
      "suggestions[1].when: condition variable 'citizen.neighborhood_id' is not allowed here",
      "suggestions[2].when: condition 'eq' references unknown step fantasma"
    ])
  end

  it "recusa intervalo de repetição que não é inteiro positivo" do
    expect(described_class.call(definition(offer: { "retake_after_days" => 0 })))
      .to eq(["offer.retake_after_days must be a positive integer"])
    expect(described_class.call(definition(offer: { "retake_after_days" => "365" })))
      .to eq(["offer.retake_after_days must be a positive integer"])
  end

  it "é total para qualquer entrada" do
    [ nil, {}, { "offer" => "x" }, { "suggestions" => "x" }, { "suggestions" => [ "x", nil ] },
      { "steps" => "x", "suggestions" => [ { "protocol" => "a", "when" => "x" } ] } ].each do |input|
      expect { described_class.call(input) }.not_to raise_error, input.inspect
    end
  end
end
```

Acrescente ao fim de `spec/protocols/gate_spec.rb` (antes do `end` final; o bloco traz o próprio protocolo de base):

```ruby
  describe "offer and suggestions (ADR 0027)" do
    let(:base) do
      {
        "name" => "saude-do-idoso", "version" => 1, "start_step_id" => "quedas",
        "steps" => [ { "id" => "quedas", "prompt" => "Caiu no último ano?", "answer_type" => "boolean",
                       "branches" => { "true" => nil, "false" => nil }, "weights" => { "true" => 3, "false" => 0 } } ],
        "scoring" => { "type" => "weighted", "thresholds" => { "baixa" => 0, "media" => 3 },
                       "priority_map" => { "baixa" => 9, "media" => 5 } }
      }
    end

    it "aceita oferta e sugestão válidas" do
      definition = base.merge(
        "offer" => { "title" => "Saúde do idoso", "eligibility" => { "gte" => ["profile.age", 60] }, "retake_after_days" => 365 },
        "suggestions" => [ { "protocol" => "saude-mental", "when" => { "eq" => ["quedas", "true"] } } ]
      )
      expect(Protocols::Gate.call(definition)).to be_valid
    end

    it "recusa variável fora do lugar, sugestão para si e intervalo zero" do
      result = Protocols::Gate.call(base.merge(
        "offer" => { "eligibility" => { "gte" => ["outcome.score", 1] } },
        "suggestions" => [ { "protocol" => "saude-do-idoso", "when" => { "eq" => ["quedas", "true"] } } ]
      ))
      expect(result.errors).to include(
        "offer.eligibility: condition variable 'outcome.score' is not allowed here",
        "suggestions[0]: suggestion points to the protocol itself"
      )
      expect(Protocols::Gate.call(base.merge("offer" => { "retake_after_days" => 0 }))).not_to be_valid
    end

    it "recusa id de passo com prefixo reservado e variável reservada em priority_when" do
      steps = [ base["steps"].first.merge("id" => "profile.age") ]
      result = Protocols::Gate.call(base.merge("steps" => steps, "start_step_id" => "profile.age"))
      expect(result.errors).to include("step id 'profile.age' uses a reserved prefix (profile., outcome., citizen.)")
      result = Protocols::Gate.call(base.merge("priority_when" => [ { "when" => { "gte" => ["profile.age", 60] }, "priority" => 2 } ]))
      expect(result.errors).to include("condition variable 'profile.age' is not allowed here")
    end
  end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod15 api bundle exec rspec spec/protocols/schema_offer_spec.rb spec/protocols/validation/offer_spec.rb spec/protocols/gate_spec.rb`
Expected: FAIL (`additionalProperties` em `offer`; `uninitialized constant Protocols::Validation::Offer`).

- [ ] **Step 3: Copie o schema do `contracts`**

Se a tag `protocols-v1.4.0` já existe no `contracts`, copie o arquivo:

```bash
/opt/homebrew/bin/git -C contracts fetch --tags
/opt/homebrew/bin/git -C contracts show protocols-v1.4.0:protocols/schema.json > apps/api/.claude/mod15/config/protocols/schema.json
```

Se a tag ainda não existe, aplique à mão o acréscimo do contrato §1 e **confira com o `contracts` antes do merge** (Task 21). Em `config/protocols/schema.json`, depois do bloco `"priority_when": { ... }` (dentro de `"properties"`, com vírgula depois do `}` dele), acrescente:

```json
    "offer": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "title":             { "type": "string", "minLength": 1, "maxLength": 60 },
        "summary":           { "type": "string", "minLength": 1, "maxLength": 200 },
        "eligibility":       { "$ref": "#/$defs/condition" },
        "retake_after_days": { "type": "integer", "minimum": 1, "maximum": 3650 }
      }
    },
    "suggestions": {
      "type": "array",
      "maxItems": 10,
      "items": {
        "type": "object",
        "additionalProperties": false,
        "required": ["protocol", "when"],
        "properties": {
          "protocol": { "type": "string", "pattern": "^[a-z][a-z0-9-]+$" },
          "when":     { "$ref": "#/$defs/condition" }
        }
      }
    }
```

e em `$defs.condition.oneOf`, depois da linha do `lt`:

```json
        { "required": ["gte"], "additionalProperties": false, "properties": { "gte": { "$ref": "#/$defs/condition_operand" } } },
        { "required": ["lte"], "additionalProperties": false, "properties": { "lte": { "$ref": "#/$defs/condition_operand" } } },
```

Run: `ruby -rjson -e 'JSON.parse(File.read("apps/api/.claude/mod15/config/protocols/schema.json")); puts "ok"'`
Expected: `ok`.

- [ ] **Step 4: Escreva `Validation::Offer`**

```ruby
# app/protocols/validation/offer.rb
# Gate de offer e suggestions (ADR 0027; spec 2026-10-05 §4.2–§4.3; contratos
# §1). Cada lugar aceita só as suas variáveis reservadas; o `when` da sugestão
# também aceita os passos do próprio protocolo. Total para qualquer entrada: o
# simulador do dashboard chama isto com definição ainda em edição.
module Protocols
  module Validation
    module Offer
      ELIGIBILITY = %w[profile.age profile.sex].freeze
      SUGGESTION = %w[profile.age profile.sex outcome.tier outcome.score outcome.priority].freeze
      RESTRICTION = %w[profile.age profile.sex citizen.neighborhood_id].freeze

      module_function

      def call(definition)
        definition = {} unless definition.is_a?(Hash)
        offer = definition["offer"].is_a?(Hash) ? definition["offer"] : {}
        errors = []
        if offer.key?("eligibility")
          errors.concat(prefixed("offer.eligibility", Condition.errors(offer["eligibility"], {}, variables: ELIGIBILITY)))
        end
        days = offer["retake_after_days"]
        unless days.nil? || (days.is_a?(Integer) && days.positive?)
          errors << "offer.retake_after_days must be a positive integer"
        end
        errors.concat(suggestion_errors(definition))
      end

      def suggestion_errors(definition)
        suggestions = definition["suggestions"]
        return [] if suggestions.nil?
        return ["suggestions must be an array"] unless suggestions.is_a?(Array)

        by_id = Array(definition["steps"]).select { |s| s.is_a?(Hash) }.to_h { |s| [s["id"].to_s, s] }
        suggestions.each_with_index.flat_map do |suggestion, index|
          next ["suggestions[#{index}] must be an object"] unless suggestion.is_a?(Hash)

          errs = []
          errs << "suggestions[#{index}]: suggestion points to the protocol itself" if suggestion["protocol"].to_s == definition["name"].to_s
          errs + prefixed("suggestions[#{index}].when", Condition.errors(suggestion["when"], by_id, variables: SUGGESTION))
        end
      end

      def prefixed(place, errors) = errors.map { |error| "#{place}: #{error}" }
    end
  end
end
```

- [ ] **Step 5: Prefixo reservado em id de passo e o gate**

Em `app/protocols/validation/condition.rb`, logo depois de `step_id_collision_errors`, acrescente:

```ruby
      # ADR 0027: nenhum passo pode ter id que comece por prefixo reservado — a
      # variável sempre venceria a resposta no contexto.
      def reserved_prefix_errors(steps)
        Array(steps).filter_map do |s|
          next unless s.is_a?(Hash) && reserved?(s["id"])

          "step id '#{s["id"]}' uses a reserved prefix (profile., outcome., citizen.)"
        end
      end
```

Em `app/protocols/gate.rb`, depois da linha `errors.concat(Validation::Condition.step_id_collision_errors(definition["steps"] || []))`, acrescente:

```ruby
      errors.concat(Validation::Condition.reserved_prefix_errors(definition["steps"] || []))
      errors.concat(Validation::Offer.call(definition)) # ADR 0027: offer.eligibility, suggestions
```

- [ ] **Step 6: Rode as specs de protocolo**

Run: `docker compose exec -T -w /rails/.claude/mod15 api bundle exec rspec spec/protocols spec/requests/protocols_gate_spec.rb spec/requests/authoring spec/commands/protocols_publish_gate_spec.rb`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod15 add config/protocols/schema.json app/protocols/validation/offer.rb app/protocols/validation/condition.rb app/protocols/gate.rb spec/protocols/schema_offer_spec.rb spec/protocols/validation/offer_spec.rb spec/protocols/gate_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod15 commit -m "feat: gate protocol offer and suggestions against schema 1.4.0

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 3: Aviso de protocolo sugerido inexistente

**Files:**
- Create: `app/services/protocols/suggestion_targets.rb`
- Modify: `app/controllers/authoring/protocols_controller.rb`
- Test: `spec/services/protocols/suggestion_targets_spec.rb`, `spec/requests/authoring/protocols_gate_spec.rb`

**Interfaces:**
- Produces: `Protocols::SuggestionTargets.warnings(definition) -> Array<String>` (`"suggestion protocol '<name>' does not exist in this city"`; total). `POST /authoring/protocols/gate` ganha `warnings: [...]` só quando há aviso.

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/services/protocols/suggestion_targets_spec.rb
require "rails_helper"

# ADR 0027 (spec 2026-10-05 §4.3): sugestão para protocolo que não existe hoje
# na cidade é aviso, não erro — a cidade pode criar o sugerido depois.
RSpec.describe Protocols::SuggestionTargets do
  before { create_default_protocol! }
  after { Rails.cache.clear }

  def definition(*names)
    { "name" => "saude-mental", "suggestions" => names.map { |n| { "protocol" => n, "when" => { "eq" => ["q1", "true"] } } } }
  end

  it "avisa só os nomes sem nenhuma versão na cidade, uma vez cada" do
    expect(described_class.warnings(definition(StartTriage::DEFAULT_PROTOCOL_NAME, "fantasma", "fantasma")))
      .to eq(["suggestion protocol 'fantasma' does not exist in this city"])
  end

  it "não avisa sem sugestões e é total" do
    expect(described_class.warnings(definition)).to eq([])
    [ nil, "x", { "suggestions" => "x" }, { "suggestions" => [ nil, "x", {} ] } ].each do |input|
      expect(described_class.warnings(input)).to eq([]), input.inspect
    end
  end
end
```

Acrescente ao fim de `spec/requests/authoring/protocols_gate_spec.rb` (antes do `end` final):

```ruby
  it "valid:true com warnings quando a sugestão aponta protocolo que não existe na cidade" do
    sign_in_as(author)
    definition = valid_def.merge("suggestions" => [ { "protocol" => "fantasma", "when" => { "eq" => ["tosse", "true"] } } ])
    post "/authoring/protocols/gate", params: { definition: definition }, as: :json
    expect(response).to have_http_status(:ok)
    expect(JSON.parse(response.body)).to eq(
      "valid" => true, "warnings" => ["suggestion protocol 'fantasma' does not exist in this city"]
    )
  end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod15 api bundle exec rspec spec/services/protocols/suggestion_targets_spec.rb spec/requests/authoring/protocols_gate_spec.rb`
Expected: FAIL (`uninitialized constant Protocols::SuggestionTargets`; resposta sem `warnings`).

- [ ] **Step 3: Implemente**

```ruby
# app/services/protocols/suggestion_targets.rb
# Aviso do gate que precisa de banco (ADR 0027; spec 2026-10-05 §4.3): fica
# fora de app/protocols/, que é o motor puro. "Não existe" = nenhuma versão com
# esse name na cidade; o próprio protocolo já é erro do gate.
module Protocols
  module SuggestionTargets
    module_function

    def warnings(definition)
      return [] unless definition.is_a?(Hash) && definition["suggestions"].is_a?(Array)

      names = definition["suggestions"].filter_map { |s| s["protocol"].to_s.presence if s.is_a?(Hash) }.uniq
      names -= [definition["name"].to_s]
      return [] if names.empty?

      existing = ProtocolDefinition.where(name: names).distinct.pluck(:name)
      (names - existing).map { |name| "suggestion protocol '#{name}' does not exist in this city" }
    end
  end
end
```

Em `app/controllers/authoring/protocols_controller.rb`:
- `gate` passa a `render_gate(Protocols::Gate.call(definition_param), warnings: Protocols::SuggestionTargets.warnings(definition_param))`;
- `render_gate` passa a:

```ruby
    def render_gate(result, warnings: [])
      extra = warnings.any? ? { warnings: warnings } : {}
      if result.valid?
        render json: { valid: true }.merge(extra)
      else
        render json: { valid: false, errors: result.errors }.merge(extra), status: :unprocessable_entity
      end
    end
```

(`preview` continua chamando `render_gate(result)` sem avisos.)

- [ ] **Step 4: Rode**

Run: `docker compose exec -T -w /rails/.claude/mod15 api bundle exec rspec spec/services/protocols/suggestion_targets_spec.rb spec/requests/authoring`
Expected: PASS (o exemplo antigo `eq("valid" => true)` continua: sem aviso, sem a chave).

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod15 add app/services/protocols/suggestion_targets.rb app/controllers/authoring/protocols_controller.rb spec/services/protocols/suggestion_targets_spec.rb spec/requests/authoring/protocols_gate_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod15 commit -m "feat: warn when a protocol suggests one that does not exist in the city

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Fatia 2 — Dados e perfil (F-15.1)

### Task 4: Migração de cidade, modelos do catálogo, trigger de transição e guarda de ADR

**Files:**
- Create: `db/city_migrate/20261005100001_add_triage_catalog.rb`
- Modify: `db/city_schema.rb`, `db/city_triggers.sql`
- Create: `app/models/triage_offer.rb`, `app/models/triage_suggestion.rb`, `app/models/triage_offer_daily_count.rb`
- Create: `spec/support/triage_catalog_helpers.rb`; Modify: `spec/rails_helper.rb`
- Modify: `spec/adr_pointers_spec.rb`
- Test: `spec/models/triage_catalog_tables_guard_spec.rb`, `spec/models/add_triage_catalog_migration_spec.rb`

**Interfaces:**
- Produces:
  - `citizens`: `birth_date` text, `sex` text, `gender_identity` text, `profile_source` string (todos nulos); `CHECK`s `ck_citizens_profile_source` e `ck_citizens_profile_complete` (sem perfil ⇔ `profile_source`, `birth_date` e `sex` nulos juntos).
  - `triage_offers` (uuid; `protocol_name` único, `enabled` default true, `position` 1..10000 default 1, `restriction` jsonb, `available_from`/`available_until` date com `until ≥ from`, `updated_by_user_id` NOT NULL, timestamps).
  - `triage_suggestions` (uuid; `citizen_id`, `source_triage_id`, `protocol_name`, `status` `pending|taken|expired`, `taken_triage_id`, `created_at`, `resolved_at`); índice único parcial `idx_triage_suggestions_one_pending (citizen_id, protocol_name) WHERE status = 'pending'`; trigger `triage_suggestions_transition` (só `pending → taken|expired`; DELETE passa).
  - `triage_offer_daily_counts` (bigint; `day`, `protocol_name`, `offered ≥ 1`; único `idx_triage_offer_daily_counts_cell`).
  - `TriageOffer`, `TriageSuggestion` (`enum :status`, prefixo `status_`: `TriageSuggestion.status_pending`...), `TriageOfferDailyCount.increment!(names, day:)`.
  - Helpers de spec: `birth_date_for(age, on: Time.zone.today) -> String`, `catalog_definition(name, offer: nil, suggestions: nil, priority_when: nil) -> Hash`, `active_protocol!(name, version: 1, **opts) -> ProtocolDefinition`, `profiled_citizen!(age:, sex: "female", phone: "+5541998765432", cpf: nil, source: "declared", neighborhood: nil) -> Citizen`, `completed_triage!(citizen, protocol_name, at: Time.current) -> Triage`, `start_for!(citizen, protocol_name) -> Result`.

- [ ] **Step 1: Confira o número da migração**

Run: `ls apps/api/.claude/mod15/db/city_migrate | tail -2`
Expected: a última é `20261004100001_add_unit_drains.rb`. Se houver outra mais nova, use um número maior (e ajuste o nome nos Steps 4, 8, 9 e o `define(version:)` no Step 7).

- [ ] **Step 2: Escreva os helpers de spec**

```ruby
# spec/support/triage_catalog_helpers.rb
# Módulo 15 (ADR 0027): protocolos com offer/suggestions, pares com perfil e
# triagens concluídas no passado (para o intervalo de repetição).
module TriageCatalogHelpers
  # Data de nascimento de quem faz `age` anos exatamente em `on`.
  def birth_date_for(age, on: Time.zone.today) = (on - age.years).iso8601

  # Um passo boolean "q1" (sim = 4 pontos → tier media, prioridade 5, não
  # urgente). priority_when pode tornar o "sim" urgente.
  def catalog_definition(name, offer: nil, suggestions: nil, priority_when: nil)
    {
      "name" => name, "version" => 1, "start_step_id" => "q1",
      "steps" => [ { "id" => "q1", "prompt" => "Tudo bem?", "answer_type" => "boolean",
                     "branches" => { "true" => nil, "false" => nil }, "weights" => { "true" => 4, "false" => 0 } } ],
      "scoring" => { "type" => "weighted", "thresholds" => { "baixa" => 0, "media" => 3 },
                     "priority_map" => { "baixa" => 9, "media" => 5 } },
      "offer" => offer, "suggestions" => suggestions, "priority_when" => priority_when
    }.compact
  end

  def active_protocol!(name, version: 1, **opts)
    definition = catalog_definition(name, **opts).merge("version" => version)
    ProtocolDefinition.create!(name: name, version: version, status: "active", definition: definition)
  end

  def profiled_citizen!(age:, sex: "female", phone: "+5541998765432", cpf: nil, source: "declared", neighborhood: nil)
    Citizen.create!(cpf: cpf || CampaignHistory.cpf_for("#{phone}:#{age}:#{sex}"), phone: phone,
                    birth_date: birth_date_for(age), sex: sex, profile_source: source, neighborhood: neighborhood)
  end

  # Triagem web concluída em `at`, sem passar pelo fluxo (para datas no passado).
  def completed_triage!(citizen, protocol_name, at: Time.current)
    definition = ProtocolDefinition.find_by!(name: protocol_name, status: "active")
    conversation = Conversation.create!(channel: "web", citizen: citizen, phone: citizen.phone, state: "completed")
    Triage.create!(conversation: conversation, protocol_definition: definition, protocol_name: protocol_name,
                   status: "completed", tier: "baixa", priority: 9, answers: { "q1" => "false" },
                   created_at: at, completed_at: at)
  end

  # Início pelo comando da web, com o consentimento vigente.
  def start_for!(citizen, protocol_name)
    Citizens::StartConversation.call(citizen: citizen, consent_version: Consents.current_version,
                                     session_id: "spec", protocol_name: protocol_name)
  end
end

RSpec.configure { |c| c.include TriageCatalogHelpers }
```

Em `spec/rails_helper.rb`, depois de `require_relative "support/analytics_helpers"`, acrescente `require_relative "support/triage_catalog_helpers"`.

(`start_for!` só funciona depois da Task 10 — antes dela `StartConversation` não aceita `protocol_name`; nenhuma spec das Tasks 4–9 chama esse helper. `profiled_citizen!` grava `birth_date` em claro até a Task 5 declarar o `encrypts`; nenhuma spec desta task o usa.)

- [ ] **Step 3: Escreva a spec de guarda (falha: tabelas não existem)**

```ruby
# spec/models/triage_catalog_tables_guard_spec.rb
require "rails_helper"

# Módulo 15 (ADR 0027; spec 2026-10-05 §3): o banco garante o que o modelo não
# vê — perfil completo ou ausente, um pendente por protocolo por par, só
# pending → taken|expired, período e posição do catálogo, contagem ≥ 1.
RSpec.describe "Guardas das tabelas do catálogo de triagens" do
  before { Current.city = TEST_CITY_A }
  after { Current.reset; Rails.cache.clear }

  let(:admin) { staff_with("catalogo-#{SecureRandom.hex(3)}@cidade.gov.br") }
  let!(:protocol) { active_protocol!("saude-do-idoso") }
  let(:citizen) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }
  let(:triage) { completed_triage!(citizen, "saude-do-idoso") }

  def suggestion!(**attrs)
    ApplicationRecord.transaction(requires_new: true) do
      TriageSuggestion.create!({ citizen: citizen, source_triage: triage, protocol_name: "saude-mental" }.merge(attrs))
    end
  end

  it "perfil: tudo ou nada, e profile_source só declared/verified" do
    expect { sql_in_savepoint("UPDATE citizens SET profile_source = 'declared' WHERE id = '#{citizen.id}'") }
      .to raise_error(ActiveRecord::StatementInvalid, /ck_citizens_profile_complete/)
    expect { sql_in_savepoint("UPDATE citizens SET birth_date = 'x', sex = 'y', profile_source = 'cadsus' WHERE id = '#{citizen.id}'") }
      .to raise_error(ActiveRecord::StatementInvalid, /ck_citizens_profile_source/)
    expect { sql_in_savepoint("UPDATE citizens SET birth_date = 'x', sex = 'y', profile_source = 'declared' WHERE id = '#{citizen.id}'") }
      .not_to raise_error
  end

  it "uma sugestão pendente por protocolo por par; resolvida não conta" do
    first = suggestion!
    expect { suggestion! }.to raise_error(ActiveRecord::RecordNotUnique)
    first.update!(status: "expired", resolved_at: Time.current)
    expect { suggestion! }.not_to raise_error
  end

  it "transições: só pending → taken | expired; resolvida congela; DELETE passa" do
    taken = suggestion!
    expect { taken.update!(status: "taken", taken_triage_id: triage.id, resolved_at: Time.current) }.not_to raise_error
    {
      "status = 'pending', resolved_at = NULL, taken_triage_id = NULL" => /pending refused|ck_triage_suggestions/,
      "status = 'expired', taken_triage_id = NULL" => /refused|ck_triage_suggestions/,
      "resolved_at = now() - interval '1 day'" => /frozen/,
      "protocol_name = 'outro'" => /only the status columns/
    }.each do |assignment, error|
      expect { sql_in_savepoint("UPDATE triage_suggestions SET #{assignment} WHERE id = '#{taken.id}'") }
        .to raise_error(ActiveRecord::StatementInvalid, error), assignment
    end
    expect { sql_in_savepoint("DELETE FROM triage_suggestions WHERE id = '#{taken.id}'") }.not_to raise_error
  end

  # Por SQL: o enum do modelo levanta ArgumentError antes de chegar ao banco.
  def insert_suggestion(status:, resolved_at: "NULL", taken: "NULL")
    sql_in_savepoint("INSERT INTO triage_suggestions (id, citizen_id, source_triage_id, protocol_name, status, " \
                     "taken_triage_id, created_at, resolved_at) VALUES (gen_random_uuid(), '#{citizen.id}', " \
                     "'#{triage.id}', 'x', '#{status}', #{taken}, now(), #{resolved_at})")
  end

  it "CHECKs de sugestão: status conhecido, resolved_at ⇔ resolvida, taken ⇔ taken_triage_id" do
    expect { insert_suggestion(status: "lixo", resolved_at: "now()") }
      .to raise_error(ActiveRecord::StatementInvalid, /ck_triage_suggestions_status/)
    expect { insert_suggestion(status: "expired") }
      .to raise_error(ActiveRecord::StatementInvalid, /ck_triage_suggestions_resolved/)
    expect { insert_suggestion(status: "expired", resolved_at: "now()", taken: "'#{triage.id}'") }
      .to raise_error(ActiveRecord::StatementInvalid, /ck_triage_suggestions_taken/)
  end

  it "catálogo: nome único, período em ordem, posição 1..10000" do
    TriageOffer.create!(protocol_name: "saude-do-idoso", updated_by_user: admin)
    expect { TriageOffer.create!(protocol_name: "saude-do-idoso", updated_by_user: admin) }
      .to raise_error(ActiveRecord::RecordInvalid)
    expect { sql_in_savepoint("UPDATE triage_offers SET available_from = '2026-10-10', available_until = '2026-10-09'") }
      .to raise_error(ActiveRecord::StatementInvalid, /ck_triage_offers_period/)
    expect { sql_in_savepoint("UPDATE triage_offers SET position = 0") }
      .to raise_error(ActiveRecord::StatementInvalid, /ck_triage_offers_position/)
  end

  it "contagem diária soma por (dia, protocolo) e nunca grava zero" do
    day = Time.zone.today
    TriageOfferDailyCount.increment!(%w[a b], day: day)
    TriageOfferDailyCount.increment!(%w[a], day: day)
    TriageOfferDailyCount.increment!([], day: day)
    expect(TriageOfferDailyCount.where(day: day).order(:protocol_name).pluck(:protocol_name, :offered))
      .to eq([ [ "a", 2 ], [ "b", 1 ] ])
    expect { sql_in_savepoint("UPDATE triage_offer_daily_counts SET offered = 0") }
      .to raise_error(ActiveRecord::StatementInvalid, /ck_triage_offer_daily_counts_offered/)
  end
end
```

- [ ] **Step 4: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod15 api bundle exec rspec spec/models/triage_catalog_tables_guard_spec.rb`
Expected: FAIL com `uninitialized constant TriageSuggestion` (ou coluna inexistente).

- [ ] **Step 5: Escreva a migração**

```ruby
# db/city_migrate/20261005100001_add_triage_catalog.rb
# Catálogo de triagens por perfil (ADR 0027; spec 2026-10-05 §3): o perfil
# cifrado do par em citizens, o catálogo da cidade (triage_offers), as
# sugestões (triage_suggestions, transição por trigger) e a contagem agregada
# de "oferecida" (triage_offer_daily_counts, sem coluna de pessoa). Só
# expansão. Os CHECKs vão na forma que o dump reproduz.
class AddTriageCatalog < ActiveRecord::Migration[8.1]
  def up
    add_column :citizens, :birth_date, :text
    add_column :citizens, :sex, :text
    add_column :citizens, :gender_identity, :text
    add_column :citizens, :profile_source, :string
    add_check_constraint :citizens,
                         "profile_source IS NULL OR profile_source::text = ANY (ARRAY['declared', 'verified']::text[])",
                         name: "ck_citizens_profile_source"
    add_check_constraint :citizens,
                         "(profile_source IS NULL AND birth_date IS NULL AND sex IS NULL) OR " \
                         "(profile_source IS NOT NULL AND birth_date IS NOT NULL AND sex IS NOT NULL)",
                         name: "ck_citizens_profile_complete"

    create_table :triage_offers, id: :uuid do |t|
      t.string :protocol_name, null: false
      t.boolean :enabled, null: false, default: true
      t.integer :position, null: false, default: 1
      t.jsonb :restriction
      t.date :available_from
      t.date :available_until
      t.references :updated_by_user, type: :uuid, null: false, foreign_key: { to_table: :users }
      t.timestamps
    end
    add_index :triage_offers, :protocol_name, unique: true
    add_check_constraint :triage_offers,
                         "available_from IS NULL OR available_until IS NULL OR available_until >= available_from",
                         name: "ck_triage_offers_period"
    add_check_constraint :triage_offers, "position >= 1 AND position <= 10000", name: "ck_triage_offers_position"

    create_table :triage_suggestions, id: :uuid do |t|
      t.references :citizen, type: :uuid, null: false, foreign_key: true, index: false
      t.references :source_triage, type: :uuid, null: false, foreign_key: { to_table: :triages }
      t.string :protocol_name, null: false
      t.string :status, null: false, default: "pending"
      t.references :taken_triage, type: :uuid, foreign_key: { to_table: :triages }
      t.timestamptz :created_at, null: false
      t.timestamptz :resolved_at
    end
    add_index :triage_suggestions, %i[citizen_id protocol_name], unique: true,
              where: "((status)::text = 'pending'::text)", name: "idx_triage_suggestions_one_pending"
    add_index :triage_suggestions, %i[citizen_id status], name: "idx_triage_suggestions_citizen_status"
    add_check_constraint :triage_suggestions, "status::text = ANY (ARRAY['pending', 'taken', 'expired']::text[])",
                         name: "ck_triage_suggestions_status"
    add_check_constraint :triage_suggestions, "(status::text = 'pending'::text) = (resolved_at IS NULL)",
                         name: "ck_triage_suggestions_resolved"
    add_check_constraint :triage_suggestions, "(status::text = 'taken'::text) = (taken_triage_id IS NOT NULL)",
                         name: "ck_triage_suggestions_taken"

    create_table :triage_offer_daily_counts do |t|
      t.date :day, null: false
      t.string :protocol_name, null: false
      t.integer :offered, null: false
    end
    add_index :triage_offer_daily_counts, %i[day protocol_name], unique: true,
              name: "idx_triage_offer_daily_counts_cell"
    add_check_constraint :triage_offer_daily_counts, "offered >= 1", name: "ck_triage_offer_daily_counts_offered"

    execute File.read(Rails.root.join("db/city_triggers.sql"))
  end

  def down
    drop_table :triage_offer_daily_counts
    drop_table :triage_suggestions
    drop_table :triage_offers
    remove_check_constraint :citizens, name: "ck_citizens_profile_complete"
    remove_check_constraint :citizens, name: "ck_citizens_profile_source"
    remove_column :citizens, :profile_source
    remove_column :citizens, :gender_identity
    remove_column :citizens, :sex
    remove_column :citizens, :birth_date
  end
end
```

(`drop_table :triage_suggestions` leva o trigger junto; a função `rota_triage_suggestion_guard` fica órfã e inofensiva, como as das outras tabelas.)

- [ ] **Step 6: Trigger e modelos**

Acrescente ao fim de `db/city_triggers.sql`:

```sql
-- triage_suggestions (ADR 0027; spec 2026-10-05 §3.3): a sugestão nasce
-- pending e só sai dali uma vez, para taken (a triagem iniciada a partir dela)
-- ou expired (o protocolo deixou de estar em oferta para o par). Resolvida,
-- congela. As colunas de identidade nunca mudam. DELETE passa de propósito: a
-- exclusão do cadastro e a revogação apagam as sugestões (ADR 0026, §5.5).
CREATE OR REPLACE FUNCTION rota_triage_suggestion_guard() RETURNS trigger AS $fn$
BEGIN
  IF NEW.id IS DISTINCT FROM OLD.id
     OR NEW.citizen_id IS DISTINCT FROM OLD.citizen_id
     OR NEW.source_triage_id IS DISTINCT FROM OLD.source_triage_id
     OR NEW.protocol_name IS DISTINCT FROM OLD.protocol_name
     OR NEW.created_at IS DISTINCT FROM OLD.created_at THEN
    RAISE EXCEPTION 'triage_suggestions: only the status columns change';
  END IF;
  IF NEW.status IS DISTINCT FROM OLD.status THEN
    IF OLD.status <> 'pending' OR NEW.status NOT IN ('taken', 'expired') THEN
      RAISE EXCEPTION 'triage_suggestions: % -> % refused (only pending -> taken | expired)', OLD.status, NEW.status;
    END IF;
  ELSIF OLD.status <> 'pending'
        AND (NEW.taken_triage_id IS DISTINCT FROM OLD.taken_triage_id
             OR NEW.resolved_at IS DISTINCT FROM OLD.resolved_at) THEN
    RAISE EXCEPTION 'triage_suggestions: a resolved suggestion is frozen';
  END IF;
  RETURN NEW;
END;
$fn$ LANGUAGE plpgsql;

DO $do$
BEGIN
  IF to_regclass('public.triage_suggestions') IS NOT NULL THEN
    EXECUTE 'DROP TRIGGER IF EXISTS triage_suggestions_transition ON triage_suggestions';
    EXECUTE 'CREATE TRIGGER triage_suggestions_transition
      BEFORE UPDATE ON triage_suggestions
      FOR EACH ROW EXECUTE FUNCTION rota_triage_suggestion_guard()';
  END IF;
END
$do$;
```

```ruby
# app/models/triage_offer.rb
# Linha do catálogo da cidade (ADR 0027; spec 2026-10-05 §3.2). Só restringe a
# elegibilidade assinada (soma com E). Escrita só por Triages::SetOffer.
class TriageOffer < ApplicationRecord
  belongs_to :updated_by_user, class_name: "User"

  validates :protocol_name, presence: true, uniqueness: true
end
```

```ruby
# app/models/triage_suggestion.rb
# Sugestão nascida da conclusão de uma triagem (ADR 0027; spec 2026-10-05
# §3.3, §5.3). pending → taken | expired, garantido por trigger; no máximo uma
# pendente por protocolo por par (índice único parcial).
class TriageSuggestion < ApplicationRecord
  belongs_to :citizen
  belongs_to :source_triage, class_name: "Triage"
  belongs_to :taken_triage, class_name: "Triage", optional: true

  enum :status, { pending: "pending", taken: "taken", expired: "expired" }, prefix: true

  validates :protocol_name, presence: true
end
```

```ruby
# app/models/triage_offer_daily_count.rb
# Quantas vezes cada protocolo foi mostrado em oferta num dia (desvio 1 do
# plano do módulo 15). Agregado, sem coluna de pessoa.
class TriageOfferDailyCount < ApplicationRecord
  def self.increment!(names, day:)
    rows = names.uniq.map { |name| { day: day, protocol_name: name, offered: 1 } }
    return if rows.empty?

    upsert_all(rows, unique_by: :idx_triage_offer_daily_counts_cell, record_timestamps: false,
                     on_duplicate: Arel.sql("offered = triage_offer_daily_counts.offered + 1"))
  end
end
```

- [ ] **Step 7: Migre os bancos de dev e faça o dump à mão**

Run:
```bash
docker compose exec -T -w /rails/.claude/mod15 api bin/rails "city:migrate[curitiba]"
docker compose exec -T -w /rails/.claude/mod15 api bin/rails "city:migrate[maringa]"
```
Expected: `[city:migrate] curitiba → 20261005100001` (e maringa).

Em `db/city_schema.rb`:
- `define(version: 2026_10_04_100001)` passa a `define(version: 2026_10_05_100001)`;
- `create_table "citizens"` passa a:

```ruby
  create_table "citizens", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.text "birth_date"
    t.string "cpf", null: false
    t.datetime "created_at", null: false
    t.timestamptz "erased_at"
    t.text "gender_identity"
    t.uuid "neighborhood_id"
    t.string "phone", null: false
    t.string "profile_source"
    t.text "sex"
    t.datetime "updated_at", null: false
    t.string "verification_level", default: "declared", null: false
    t.index ["cpf", "phone"], name: "index_citizens_on_cpf_and_phone", unique: true
    t.index ["neighborhood_id"], name: "index_citizens_on_neighborhood_id"
    t.index ["phone"], name: "index_citizens_on_phone"
    t.check_constraint "(verification_level)::text = ANY (ARRAY['declared'::text, 'verified'::text])", name: "ck_citizens_verification_level"
    t.check_constraint "profile_source IS NULL OR profile_source::text = ANY (ARRAY['declared', 'verified']::text[])", name: "ck_citizens_profile_source"
    t.check_constraint "(profile_source IS NULL AND birth_date IS NULL AND sex IS NULL) OR (profile_source IS NOT NULL AND birth_date IS NOT NULL AND sex IS NOT NULL)", name: "ck_citizens_profile_complete"
  end
```

- entre `create_table "triages"` e `create_table "users"` (ordem alfabética: `triage_offer_daily_counts`, `triage_offers`, `triage_suggestions` vêm **antes** de `triages`), as três tabelas:

```ruby
  create_table "triage_offer_daily_counts", force: :cascade do |t|
    t.date "day", null: false
    t.integer "offered", null: false
    t.string "protocol_name", null: false
    t.index ["day", "protocol_name"], name: "idx_triage_offer_daily_counts_cell", unique: true
    t.check_constraint "offered >= 1", name: "ck_triage_offer_daily_counts_offered"
  end

  create_table "triage_offers", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.date "available_from"
    t.date "available_until"
    t.datetime "created_at", null: false
    t.boolean "enabled", default: true, null: false
    t.integer "position", default: 1, null: false
    t.string "protocol_name", null: false
    t.jsonb "restriction"
    t.datetime "updated_at", null: false
    t.uuid "updated_by_user_id", null: false
    t.index ["protocol_name"], name: "index_triage_offers_on_protocol_name", unique: true
    t.index ["updated_by_user_id"], name: "index_triage_offers_on_updated_by_user_id"
    t.check_constraint "available_from IS NULL OR available_until IS NULL OR available_until >= available_from", name: "ck_triage_offers_period"
    t.check_constraint "position >= 1 AND position <= 10000", name: "ck_triage_offers_position"
  end

  create_table "triage_suggestions", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.uuid "citizen_id", null: false
    t.timestamptz "created_at", null: false
    t.string "protocol_name", null: false
    t.timestamptz "resolved_at"
    t.uuid "source_triage_id", null: false
    t.string "status", default: "pending", null: false
    t.uuid "taken_triage_id"
    t.index ["citizen_id", "protocol_name"], name: "idx_triage_suggestions_one_pending", unique: true, where: "((status)::text = 'pending'::text)"
    t.index ["citizen_id", "status"], name: "idx_triage_suggestions_citizen_status"
    t.index ["source_triage_id"], name: "index_triage_suggestions_on_source_triage_id"
    t.index ["taken_triage_id"], name: "index_triage_suggestions_on_taken_triage_id"
    t.check_constraint "status::text = ANY (ARRAY['pending', 'taken', 'expired']::text[])", name: "ck_triage_suggestions_status"
    t.check_constraint "(status::text = 'pending'::text) = (resolved_at IS NULL)", name: "ck_triage_suggestions_resolved"
    t.check_constraint "(status::text = 'taken'::text) = (taken_triage_id IS NOT NULL)", name: "ck_triage_suggestions_taken"
  end
```

- na lista de `add_foreign_key` (ordem alfabética, antes de `add_foreign_key "triages", "conversations"`):

```ruby
  add_foreign_key "triage_offers", "users", column: "updated_by_user_id"
  add_foreign_key "triage_suggestions", "citizens"
  add_foreign_key "triage_suggestions", "triages", column: "source_triage_id"
  add_foreign_key "triage_suggestions", "triages", column: "taken_triage_id"
```

O juiz é a paridade (Step 9). Se ela acusar diferença, compare com o banco migrado —

```bash
docker compose exec -T db psql -U postgres -d <banco_de_curitiba> -c "\d+ citizens" -c "\d+ triage_offers" -c "\d+ triage_suggestions" -c "\d+ triage_offer_daily_counts"
```

(o nome do banco sai de `bin/rails runner 'puts URI(City.find_by(slug: "curitiba").database_url).path.delete_prefix("/")'`; não cole a URL em lugar nenhum) — e ajuste o dump, nunca a migração, salvo erro de verdade nela.

- [ ] **Step 8: Spec de down e up e guarda de ADR**

```ruby
# spec/models/add_triage_catalog_migration_spec.rb
require "rails_helper"
require Rails.root.join("db/city_migrate/20261005100001_add_triage_catalog.rb").to_s

# F-15.1/F-15.4: o down() da migração de cidade 20261005100001 desfaz o que o
# up() cria, e o up() seguinte restaura o schema idêntico. DDL do Postgres é
# transacional: down e up rodam num savepoint desfeito no fim.
RSpec.describe "Migração de cidade 20261005100001 (AddTriageCatalog): down e up" do
  let(:new_tables) { %w[triage_offers triage_suggestions triage_offer_daily_counts] }
  let(:models) { [ Citizen, TriageOffer, TriageSuggestion, TriageOfferDailyCount ] }

  def conn = ApplicationRecord.connection

  def migrate(direction)
    ActiveRecord::Migration.suppress_messages { AddTriageCatalog.new.exec_migration(conn, direction) }
    models.each(&:reset_column_information)
  end

  def fingerprint
    {
      columns: conn.select_rows(<<~SQL),
        SELECT table_name, column_name, data_type, is_nullable, column_default FROM information_schema.columns
        WHERE table_schema = 'public' AND table_name NOT IN ('schema_migrations', 'ar_internal_metadata') ORDER BY 1, 2
      SQL
      indexes: conn.select_rows("SELECT tablename, indexname, indexdef FROM pg_indexes WHERE schemaname = 'public' ORDER BY 1, 2"),
      constraints: conn.select_rows(<<~SQL)
        SELECT rel.relname, con.conname, pg_get_constraintdef(con.oid) FROM pg_constraint con
        JOIN pg_class rel ON rel.oid = con.conrelid JOIN pg_namespace ns ON ns.oid = rel.relnamespace
        WHERE ns.nspname = 'public' ORDER BY 1, 2
      SQL
    }
  end

  it "down remove tabelas e colunas; up seguinte restaura o schema idêntico" do
    ApplicationRecord.transaction(requires_new: true) do
      before = fingerprint
      migrate(:down)
      expect(conn.tables & new_tables).to be_empty
      expect(conn.columns(:citizens).map(&:name)).not_to include("birth_date", "sex", "gender_identity", "profile_source")
      migrate(:up)
      expect(fingerprint).to eq(before)
      raise ActiveRecord::Rollback
    end
  ensure
    models.each(&:reset_column_information)
  end
end
```

Em `spec/adr_pointers_spec.rb`:
- no comentário do topo, `de 0001 a 0026` passa a `de 0001 a 0027`, e a lista que termina em `0026 exclusão do cadastro e revogação)` passa a terminar em `0026 exclusão do cadastro e revogação, 0027 catálogo de triagens por perfil)`;
- `VALID_RANGE = (1..26).freeze` passa a `VALID_RANGE = (1..27).freeze`;
- `it "only points at ADRs that exist in the v2 corpus (0001..0026)"` passa a `(0001..0027)`.

- [ ] **Step 9: Recarregue os bancos de teste e rode**

Run:
```bash
docker compose exec -T db psql -U postgres -c "DROP DATABASE IF EXISTS rota_saude_test_city_a" -c "DROP DATABASE IF EXISTS rota_saude_test_city_b"
docker compose exec -T -w /rails/.claude/mod15 api bin/rails city:test_databases
docker compose exec -T -w /rails/.claude/mod15 api bundle exec rspec spec/services/city_schema_spec.rb spec/adr_pointers_spec.rb spec/models/triage_catalog_tables_guard_spec.rb spec/models/add_triage_catalog_migration_spec.rb spec/architecture/city_encrypted_attributes_guard_spec.rb
```
Expected: PASS.

- [ ] **Step 10: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod15 add db/city_migrate/20261005100001_add_triage_catalog.rb db/city_schema.rb db/city_triggers.sql app/models/triage_offer.rb app/models/triage_suggestion.rb app/models/triage_offer_daily_count.rb spec/support/triage_catalog_helpers.rb spec/rails_helper.rb spec/adr_pointers_spec.rb spec/models/triage_catalog_tables_guard_spec.rb spec/models/add_triage_catalog_migration_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod15 commit -m "feat: add citizen profile columns and triage catalog tables

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 5: Perfil no modelo `Citizen` (cifra, idade, filtro de log)

**Files:**
- Modify: `app/models/citizen.rb`
- Modify: `app/services/city_encryption.rb` (`CITY_KEYED_TARGETS`)
- Modify: `config/initializers/filter_parameter_logging.rb`
- Create: `app/commands/citizens/profile_json.rb`
- Test: `spec/models/citizen_profile_spec.rb`

**Interfaces:**
- Produces:
  - `Citizen::SEXES`, `Citizen::GENDER_IDENTITIES`, `Citizen::PROFILE_SOURCES` (valores das Global Constraints).
  - `Citizen.age_between(born_on, on) -> Integer`; `Citizen#age(on: Time.zone.today) -> Integer | nil`; `Citizen#profile? -> Boolean`; `Citizen#profile_context(on: Time.zone.today) -> { age:, sex: }`.
  - `Citizens::ProfileJson.call(citizen) -> Hash | nil` (`{ birth_date:, sex:, gender_identity:, profile_source: }`, `nil` sem perfil — contratos §3.1).

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/models/citizen_profile_spec.rb
require "rails_helper"

# ADR 0027 (spec 2026-10-05 §3.1): perfil cifrado por cidade, idade calculada
# na leitura, nunca gravada; nada do perfil nos logs de parâmetros.
RSpec.describe Citizen, "perfil" do
  let(:citizen) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }

  it "idade no aniversário, na véspera e em 29/02" do
    expect(described_class.age_between(Date.new(1966, 10, 5), Date.new(2026, 10, 5))).to eq(60)
    expect(described_class.age_between(Date.new(1966, 10, 5), Date.new(2026, 10, 4))).to eq(59)
    expect(described_class.age_between(Date.new(2000, 2, 29), Date.new(2026, 2, 28))).to eq(25)
    expect(described_class.age_between(Date.new(2000, 2, 29), Date.new(2026, 3, 1))).to eq(26)
    expect(described_class.age_between(Date.new(2000, 2, 29), Date.new(2028, 2, 29))).to eq(28)
  end

  it "age e profile? leem o perfil; sem perfil, nil e false" do
    expect(citizen.age).to be_nil
    expect(citizen).not_to be_profile
    citizen.update!(birth_date: birth_date_for(62), sex: "female", profile_source: "declared")
    expect(citizen.reload.age).to eq(62)
    expect(citizen).to be_profile
    expect(citizen.profile_context).to eq(age: 62, sex: "female")
    expect(Citizens::ProfileJson.call(citizen))
      .to eq(birth_date: birth_date_for(62), sex: "female", gender_identity: nil, profile_source: "declared")
    expect(Citizens::ProfileJson.call(Citizen.new)).to be_nil
  end

  it "grava birth_date, sex e gender_identity cifrados (não determinístico) e nunca a idade" do
    citizen.update!(birth_date: "1963-04-02", sex: "female", gender_identity: "cis_woman", profile_source: "declared")
    raw = ApplicationRecord.connection.select_one(
      ApplicationRecord.sanitize_sql([ "SELECT birth_date, sex, gender_identity, profile_source FROM citizens WHERE id = ?", citizen.id ])
    )
    expect(raw["birth_date"]).not_to include("1963")
    expect(raw["sex"]).not_to include("female")
    expect(raw["gender_identity"]).not_to include("cis_woman")
    expect(raw["profile_source"]).to eq("declared")
    expect(described_class.column_names).not_to include("age")
    expect(CityEncryption::CITY_KEYED_TARGETS).to include([ Citizen, :birth_date ], [ Citizen, :sex ], [ Citizen, :gender_identity ])
  end

  it "recusa valores fora da lista no modelo" do
    expect(citizen.update(sex: "x", birth_date: "1963-04-02", profile_source: "declared")).to be(false)
    expect(citizen.update(sex: "male", birth_date: "02/04/1963", profile_source: "declared")).to be(false)
    expect(citizen.update(sex: "male", birth_date: "1963-04-02", gender_identity: "y", profile_source: "declared")).to be(false)
    expect(citizen.update(sex: "male", birth_date: "1963-04-02", profile_source: "cadsus")).to be(false)
  end

  it "filtra os três campos do log de parâmetros" do
    filter = ActiveSupport::ParameterFilter.new(Rails.application.config.filter_parameters)
    expect(filter.filter("birth_date" => "1963-04-02", "sex" => "female", "gender_identity" => "cis_woman"))
      .to eq("birth_date" => "[FILTERED]", "sex" => "[FILTERED]", "gender_identity" => "[FILTERED]")
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod15 api bundle exec rspec spec/models/citizen_profile_spec.rb`
Expected: FAIL (`undefined method 'age_between'`).

- [ ] **Step 3: Implemente**

Em `app/models/citizen.rb`, depois das duas linhas `encrypts :cpf`/`encrypts :phone`, acrescente:

```ruby
  # ADR 0027: perfil do par. Cifrado com a chave da cidade, NÃO determinístico
  # (nenhum é chave de busca). A idade nunca é gravada: Citizen#age calcula.
  SEXES = %w[female male].freeze
  GENDER_IDENTITIES = %w[cis_woman cis_man trans_woman trans_man travesti non_binary other].freeze
  PROFILE_SOURCES = %w[declared verified].freeze
  BIRTH_DATE_FORMAT = /\A\d{4}-\d{2}-\d{2}\z/

  encrypts :birth_date
  encrypts :sex
  encrypts :gender_identity
```

depois de `validates :cpf, :phone, presence: true`, acrescente:

```ruby
  # O banco só vê texto cifrado: os valores são conferidos aqui (o CHECK do
  # banco cobre profile_source e o "tudo ou nada" do perfil).
  validates :sex, inclusion: { in: SEXES }, allow_nil: true
  validates :gender_identity, inclusion: { in: GENDER_IDENTITIES }, allow_nil: true
  validates :profile_source, inclusion: { in: PROFILE_SOURCES }, allow_nil: true
  validates :birth_date, format: { with: BIRTH_DATE_FORMAT }, allow_nil: true
```

e, antes de `def cpf_masked`:

```ruby
  # Anos completos em `on`. 29/02 faz aniversário em 01/03 nos anos comuns.
  def self.age_between(born_on, on)
    years = on.year - born_on.year
    years -= 1 if on.month < born_on.month || (on.month == born_on.month && on.day < born_on.day)
    years
  end

  def profile? = birth_date.present? && sex.present?

  # Fuso da cidade: dentro de CityConnection.with, Time.zone é o da cidade.
  def age(on: Time.zone.today)
    return nil if birth_date.blank?

    self.class.age_between(Date.iso8601(birth_date), on)
  rescue Date::Error
    nil
  end

  def profile_context(on: Time.zone.today) = { age: age(on: on), sex: sex }
```

Em `app/services/city_encryption.rb`, em `CITY_KEYED_TARGETS`, depois de `[ Citizen,        :phone ],`:

```ruby
    [ Citizen,        :birth_date ],
    [ Citizen,        :sex ],
    [ Citizen,        :gender_identity ],
```

Em `config/initializers/filter_parameter_logging.rb`, acrescente ao fim da lista (antes do `]`), com o comentário:

```ruby
  # ADR 0027: perfil do par — dado de saúde sensível, nunca em log.
  :birth_date, :sex, :gender_identity,
```

(Cuidado com a vírgula: a linha `:variables, :query` passa a `:variables, :query,`.)

```ruby
# app/commands/citizens/profile_json.rb
# O perfil do par como as rotas do cidadão e o balcão o mostram (contratos
# §3.1, §4.4). nil sem perfil (sem birth_date ou sex). Nunca a idade.
module Citizens
  module ProfileJson
    module_function

    def call(citizen)
      return nil unless citizen.profile?

      { birth_date: citizen.birth_date, sex: citizen.sex, gender_identity: citizen.gender_identity,
        profile_source: citizen.profile_source }
    end
  end
end
```

- [ ] **Step 4: Rode**

Run: `docker compose exec -T -w /rails/.claude/mod15 api bundle exec rspec spec/models/citizen_profile_spec.rb spec/architecture/city_encrypted_attributes_guard_spec.rb spec/commands/city_rekey_spec.rb spec/models/citizen_spec.rb`
Expected: PASS (se `spec/models/citizen_spec.rb` não existir, rode sem ele).

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod15 add app/models/citizen.rb app/services/city_encryption.rb config/initializers/filter_parameter_logging.rb app/commands/citizens/profile_json.rb spec/models/citizen_profile_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod15 commit -m "feat: encrypt the citizen profile and compute age on read

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 6: `Citizens::ProfileValues` e `Citizens::SetProfile`

**Files:**
- Create: `app/commands/citizens/profile_values.rb`, `app/commands/citizens/set_profile.rb`
- Modify: `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`
- Test: `spec/commands/citizens/profile_values_spec.rb`, `spec/commands/citizens/set_profile_spec.rb`

**Interfaces:**
- Consumes: `Citizen::SEXES`, `Citizen::GENDER_IDENTITIES`, `Citizen.age_between` (Task 5).
- Produces:
  - `Citizens::ProfileValues.call(birth_date:, sex:, gender_identity:, today: Time.zone.today) -> Result` — ok com `{ birth_date: "YYYY-MM-DD", sex:, gender_identity: String | nil }`; falhas `:invalid_birth_date`, `:invalid_sex`, `:invalid_gender_identity` (nessa ordem). `""` e `nil` em `gender_identity` = não informado.
  - `Citizens::SetProfile.call(citizen:, birth_date:, sex:, gender_identity:) -> Result` — ok `{ citizen: }`; falhas as de `ProfileValues` e `:profile_verified`. Grava `profile_source: "declared"`; evento `citizen.profile_changed` `{ citizen_id }` só quando muda.

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/commands/citizens/profile_values_spec.rb
require "rails_helper"

# Contratos §2: birth_date YYYY-MM-DD, não futura, idade ≤ 130; sex
# female|male; gender_identity da lista do e-SUS ou nulo.
RSpec.describe Citizens::ProfileValues do
  let(:today) { Date.new(2026, 10, 5) }

  def call(birth_date: "1963-04-02", sex: "female", gender_identity: nil)
    described_class.call(birth_date: birth_date, sex: sex, gender_identity: gender_identity, today: today)
  end

  it "normaliza um perfil válido" do
    expect(call.payload).to eq(birth_date: "1963-04-02", sex: "female", gender_identity: nil)
    expect(call(gender_identity: "travesti").payload[:gender_identity]).to eq("travesti")
    expect(call(gender_identity: "").payload[:gender_identity]).to be_nil
    expect(call(birth_date: "2026-10-05").payload[:birth_date]).to eq("2026-10-05")
    expect(call(birth_date: "1896-10-05")).to be_ok
  end

  it "recusa data inválida, futura ou de mais de 130 anos" do
    [ "2026-10-06", "1896-10-04", "1963-02-30", "02/04/1963", "1963-4-2", 19630402, nil, "" ].each do |value|
      expect(call(birth_date: value).reason).to eq(:invalid_birth_date), value.inspect
    end
  end

  it "recusa sexo e identidade de gênero fora da lista" do
    [ "F", "feminino", "other", nil, 1 ].each { |value| expect(call(sex: value).reason).to eq(:invalid_sex), value.inspect }
    [ "mulher", 1, [ "cis_woman" ] ].each do |value|
      expect(call(gender_identity: value).reason).to eq(:invalid_gender_identity), value.inspect
    end
  end
end
```

```ruby
# spec/commands/citizens/set_profile_spec.rb
require "rails_helper"

# ADR 0027 (spec 2026-10-05 §5.4): no padrão de SetNeighborhood — lock, evento
# só com o id, só quando muda; perfil verified não muda pelo cidadão.
RSpec.describe Citizens::SetProfile do
  before { Current.city = TEST_CITY_A }
  after { Current.reset }

  let(:citizen) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }
  def payloads = DomainEvent.where(name: "citizen.profile_changed").map(&:payload)

  def set(birth_date: "1963-04-02", sex: "female", gender_identity: nil, target: citizen)
    described_class.call(citizen: target, birth_date: birth_date, sex: sex, gender_identity: gender_identity)
  end

  it "grava declared e publica só o id" do
    expect(set(gender_identity: "cis_woman")).to be_ok
    expect(citizen.reload).to have_attributes(birth_date: "1963-04-02", sex: "female", gender_identity: "cis_woman",
                                              profile_source: "declared")
    expect(payloads).to eq([ { "citizen_id" => citizen.id } ])
  end

  it "corrige enquanto declared; repetir o mesmo valor não publica de novo" do
    set
    set
    set(sex: "male")
    expect(citizen.reload.sex).to eq("male")
    expect(payloads.size).to eq(2)
  end

  it "verified não muda pelo cidadão: :profile_verified e nada gravado" do
    citizen.update!(birth_date: "1963-04-02", sex: "female", profile_source: "verified")
    expect(set(birth_date: "1970-01-01").reason).to eq(:profile_verified)
    expect(citizen.reload).to have_attributes(birth_date: "1963-04-02", profile_source: "verified")
    expect(payloads).to be_empty
  end

  it "valor inválido: motivo de ProfileValues e nada gravado" do
    expect(set(sex: "x").reason).to eq(:invalid_sex)
    expect(citizen.reload.profile_source).to be_nil
  end
end
```

Em `spec/initializers/domain_events_bindings_spec.rb`, acrescente ao fim:

```ruby
# Módulo 15 (ADR 0027): eventos do catálogo de triagens, só ids e sem consumidor.
RSpec.describe "triage catalog event bindings (ADR 0027)" do
  it "declares every triage catalog event with no consumer" do
    names = %w[citizen.profile_changed]
    expect(DomainEvents.registry.keys).to include(*names)
    expect(names.flat_map { |n| DomainEvents.registry[n] }).to be_empty
  end
end
```

(As Tasks 11 e 14 acrescentam `triage.suggested` e `triage_offer.changed` a esse `names`.)

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod15 api bundle exec rspec spec/commands/citizens/profile_values_spec.rb spec/commands/citizens/set_profile_spec.rb spec/initializers/domain_events_bindings_spec.rb`
Expected: FAIL (`uninitialized constant Citizens::ProfileValues`; evento não declarado).

- [ ] **Step 3: Implemente**

```ruby
# app/commands/citizens/profile_values.rb
# Confere e normaliza o perfil do par (contratos §2). Usado pelo cidadão
# (SetProfile, POST /citizen/people) e pelo balcão (Verify). Puro: `today`
# vem de quem chama (o fuso da cidade). Reasons: :invalid_birth_date,
# :invalid_sex, :invalid_gender_identity.
module Citizens
  module ProfileValues
    MAX_AGE = 130

    module_function

    def call(birth_date:, sex:, gender_identity:, today: Time.zone.today)
      date = parse(birth_date)
      if date.nil? || date > today || Citizen.age_between(date, today) > MAX_AGE
        return Result.fail(:invalid_birth_date)
      end
      return Result.fail(:invalid_sex) unless Citizen::SEXES.include?(sex)

      identity = gender_identity == "" ? nil : gender_identity
      unless identity.nil? || Citizen::GENDER_IDENTITIES.include?(identity)
        return Result.fail(:invalid_gender_identity)
      end

      Result.ok(birth_date: date.iso8601, sex: sex, gender_identity: identity)
    end

    def parse(raw)
      return nil unless raw.is_a?(String) && raw.match?(Citizen::BIRTH_DATE_FORMAT)

      Date.iso8601(raw)
    rescue Date::Error
      nil
    end
  end
end
```

```ruby
# app/commands/citizens/set_profile.rb
# Perfil declarado pelo cidadão (ADR 0027; spec 2026-10-05 §5.4), no padrão de
# SetNeighborhood: lock no par, evento só com o id e só quando muda. Perfil
# conferido no posto (verified) só muda no posto. Reasons: :profile_verified e
# as de ProfileValues.
module Citizens
  class SetProfile
    def self.call(citizen:, birth_date:, sex:, gender_identity:)
      values = ProfileValues.call(birth_date: birth_date, sex: sex, gender_identity: gender_identity)
      return values if values.failure?

      result = nil
      ApplicationRecord.transaction do
        citizen.lock!
        next result = Result.fail(:profile_verified) if citizen.profile_source == "verified"

        citizen.assign_attributes(values.payload.merge(profile_source: "declared"))
        if citizen.changed?
          citizen.save!
          DomainEvents.publish("citizen.profile_changed", citizen_id: citizen.id)
        end
        result = Result.ok(citizen: citizen)
      end
      result
    end
  end
end
```

Em `config/initializers/domain_events.rb`, antes do `end` final do bloco:

```ruby

  # Catálogo de triagens (ADR 0027; spec 2026-10-05 §3–§5): trilha, só ids e
  # nome de protocolo; nunca data de nascimento, idade, sexo ou identidade de
  # gênero. Sem consumidor, de propósito.
  DomainEvents.bind "citizen.profile_changed", to: []
```

- [ ] **Step 4: Rode**

Run: `docker compose exec -T -w /rails/.claude/mod15 api bundle exec rspec spec/commands/citizens/profile_values_spec.rb spec/commands/citizens/set_profile_spec.rb spec/initializers/domain_events_bindings_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod15 add app/commands/citizens/profile_values.rb app/commands/citizens/set_profile.rb config/initializers/domain_events.rb spec/commands/citizens/profile_values_spec.rb spec/commands/citizens/set_profile_spec.rb spec/initializers/domain_events_bindings_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod15 commit -m "feat: let the citizen declare and correct the pair profile

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 7: API do perfil (`GET/POST /citizen/people`, `POST /citizen/people/:id/profile`)

**Files:**
- Modify: `app/controllers/citizen_api/people_controller.rb`
- Modify: `app/controllers/citizen_api/base_controller.rb` (recebe `requested_neighborhood_id`)
- Modify: `app/controllers/citizen_api/conversations_controller.rb` (perde `requested_neighborhood_id`, que sobe para a base)
- Modify: `app/commands/citizens/register_person.rb`
- Modify: `config/routes.rb`
- Test: `spec/requests/citizen_api/profile_spec.rb`, `spec/commands/citizens/register_person_spec.rb`

**Interfaces:**
- Consumes: `Citizens::ProfileValues`, `Citizens::SetProfile`, `Citizens::ProfileJson` (Tasks 5–6).
- Produces:
  - `Citizens::RegisterPerson.call(phone:, cpf:, profile: nil) -> Result` — payload `{ citizen:, created: Boolean }`; com `profile` (saída de `ProfileValues`), o par novo nasce `declared` e publica `citizen.profile_changed`; par existente não muda.
  - `CitizenApi::BaseController#requested_neighborhood_id` (privado; mesmo comportamento de hoje).
  - Rotas: `POST /citizen/people` → `people#create`; `POST /citizen/people/:id/profile` → `people#profile`; `GET /citizen/people` ganha `profile`.

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/requests/citizen_api/profile_spec.rb
require "rails_helper"

# Contratos §3.1–§3.3 (ADR 0027): o par nasce com perfil; o perfil declarado se
# corrige; o verificado não muda pelo canal do cidadão.
RSpec.describe "Perfil do par", type: :request do
  before do
    create_default_protocol!
    ConsentTerm.create!(version: "1", body: "Termo de teste", published_at: Time.current)
    sign_in_citizen("+5541998765432")
  end
  after { Rails.cache.clear }

  def body = JSON.parse(response.body)
  def status_and_error = [ response.status, body["error"] ]
  let!(:centro) { Neighborhood.create!(name: "Centro", source: "seed") }

  def create_person(**params)
    json_post "/citizen/people", { cpf: "529.982.247-25", consent_version: "1", birth_date: "1963-04-02",
                                   sex: "female" }.merge(params)
  end

  describe "POST /citizen/people" do
    it "cria o par com perfil e bairro: 201 e a pessoa no formato de GET /citizen/people" do
      create_person(gender_identity: "cis_woman", neighborhood_id: centro.id)
      expect(response).to have_http_status(:created)
      expect(body["person"]).to include(
        "cpf_masked" => "***.982.247-**", "verification_level" => "declared",
        "neighborhood" => { "id" => centro.id, "name" => "Centro" },
        "profile" => { "birth_date" => "1963-04-02", "sex" => "female", "gender_identity" => "cis_woman",
                       "profile_source" => "declared" }
      )
      expect(DomainEvent.where(name: "citizen.profile_changed").sole.payload).to eq("citizen_id" => body.dig("person", "id"))

      get "/citizen/people"
      expect(body["people"].sole["profile"]).to include("birth_date" => "1963-04-02")
    end

    it "par já existente: 200 e o perfil não é sobrescrito" do
      create_person
      create_person(birth_date: "1990-01-01", sex: "male")
      expect(response).to have_http_status(:ok)
      expect(body.dig("person", "profile")).to include("birth_date" => "1963-04-02", "sex" => "female")
      expect(Citizen.count).to eq(1)
    end

    it "termo desatualizado: 409 antes de gravar qualquer coisa" do
      expect { create_person(consent_version: "0") }.not_to change(Citizen, :count)
      expect(status_and_error).to eq([ 409, "consent_outdated" ])
    end

    it "valores inválidos: 422 com o motivo e nenhum CPF gravado" do
      {
        { cpf: "111.111.111-11" } => "invalid_cpf",
        { birth_date: "2999-01-01" } => "invalid_birth_date",
        { sex: "x" } => "invalid_sex",
        { gender_identity: "x" } => "invalid_gender_identity",
        { neighborhood_id: SecureRandom.uuid } => "invalid_neighborhood"
      }.each do |params, error|
        expect { create_person(**params) }.not_to change(Citizen, :count)
        expect(status_and_error).to eq([ 422, error ]), params.inspect
      end
    end

    it "teto de pessoas por celular: 422 too_many_people" do
      Citizen::MAX_PER_PHONE.times { |i| Citizen.create!(cpf: CampaignHistory.cpf_for("teto-#{i}"), phone: "+5541998765432") }
      create_person
      expect(status_and_error).to eq([ 422, "too_many_people" ])
    end
  end

  describe "POST /citizen/people/:id/profile" do
    let!(:person) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }

    it "grava o perfil declarado; gender_identity null apaga" do
      json_post "/citizen/people/#{person.id}/profile", birth_date: "1963-04-02", sex: "female", gender_identity: "cis_woman"
      expect(response).to have_http_status(:ok)
      json_post "/citizen/people/#{person.id}/profile", birth_date: "1963-04-02", sex: "female", gender_identity: nil
      expect(body.dig("person", "profile")).to eq("birth_date" => "1963-04-02", "sex" => "female",
                                                  "gender_identity" => nil, "profile_source" => "declared")
    end

    it "verificado: 409 profile_verified" do
      person.update!(birth_date: "1963-04-02", sex: "female", profile_source: "verified")
      json_post "/citizen/people/#{person.id}/profile", birth_date: "1970-01-01", sex: "female", gender_identity: nil
      expect(status_and_error).to eq([ 409, "profile_verified" ])
    end

    it "valor inválido: 422; par de outro celular: 404" do
      json_post "/citizen/people/#{person.id}/profile", birth_date: "1963-04-02", sex: "outro", gender_identity: nil
      expect(status_and_error).to eq([ 422, "invalid_sex" ])
      other = Citizen.create!(cpf: "52998224725", phone: "+5541900000000")
      json_post "/citizen/people/#{other.id}/profile", birth_date: "1963-04-02", sex: "female", gender_identity: nil
      expect(status_and_error).to eq([ 404, "not_found" ])
    end
  end

  it "GET /citizen/people: profile null sem perfil" do
    Citizen.create!(cpf: "52998224725", phone: "+5541998765432")
    get "/citizen/people"
    expect(body["people"].sole["profile"]).to be_nil
  end
end
```

Acrescente a `spec/commands/citizens/register_person_spec.rb` (antes do `end` final; se o arquivo não tiver `Current.city`, acrescente `before { Current.city = TEST_CITY_A }` e `after { Current.reset }` neste bloco):

```ruby
  describe "com perfil (ADR 0027)" do
    before { Current.city = TEST_CITY_A }
    after { Current.reset }

    let(:profile) { { birth_date: "1963-04-02", sex: "female", gender_identity: nil } }

    it "par novo nasce declared, com created: true e o evento" do
      result = described_class.call(phone: "+5541998765432", cpf: "529.982.247-25", profile: profile)
      expect(result.payload[:created]).to be(true)
      expect(result.payload[:citizen].reload).to have_attributes(birth_date: "1963-04-02", profile_source: "declared")
      expect(DomainEvent.where(name: "citizen.profile_changed").count).to eq(1)
    end

    it "par existente: created: false e perfil intacto" do
      Citizen.create!(cpf: "52998224725", phone: "+5541998765432")
      result = described_class.call(phone: "+5541998765432", cpf: "529.982.247-25", profile: profile)
      expect(result.payload[:created]).to be(false)
      expect(result.payload[:citizen].reload.profile_source).to be_nil
    end
  end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod15 api bundle exec rspec spec/requests/citizen_api/profile_spec.rb spec/commands/citizens/register_person_spec.rb`
Expected: FAIL (rota inexistente; `unknown keyword: :profile`).

- [ ] **Step 3: `RegisterPerson` com perfil**

Substitua `app/commands/citizens/register_person.rb` por:

```ruby
# "Para quem é esta triagem?" com CPF novo: cria (ou acha) o par CPF + telefone.
# ADR 0027: com `profile` (saída de Citizens::ProfileValues), o par NOVO nasce
# com o perfil declarado, num INSERT só; o par existente não muda (a correção
# é pela rota própria). `created` diz qual dos dois aconteceu.
# Reasons: :invalid_cpf, :too_many_people.
module Citizens
  class RegisterPerson
    def self.call(phone:, cpf:, profile: nil)
      digits = CitizenIdentity::Cpf.normalize(cpf)
      return Result.fail(:invalid_cpf) unless digits

      existing = Citizen.find_by(cpf: digits, phone: phone)
      return Result.ok(citizen: existing, created: false) if existing
      return Result.fail(:too_many_people) if Citizen.where(phone: phone).count >= Citizen::MAX_PER_PHONE

      attributes = { cpf: digits, phone: phone }
      attributes.merge!(profile.slice(:birth_date, :sex, :gender_identity), profile_source: "declared") if profile
      citizen = Citizen.create!(attributes)
      DomainEvents.publish("citizen.profile_changed", citizen_id: citizen.id) if profile
      Result.ok(citizen: citizen, created: true)
    rescue ActiveRecord::RecordNotUnique
      Result.ok(citizen: Citizen.find_by!(cpf: digits, phone: phone), created: false)
    end
  end
end
```

- [ ] **Step 4: Bairro pedido sobe para a base**

Em `app/controllers/citizen_api/conversations_controller.rb`, **remova** o método `requested_neighborhood_id` (e o comentário acima dele). Em `app/controllers/citizen_api/base_controller.rb`, dentro de `private`, depois de `render_error`, acrescente o mesmo método:

```ruby
    # nil quando não veio (ou veio vazio/null: "prefiro não informar"); o id
    # quando é um bairro ativo; senão responde 422 e devolve nil (ADR 0023).
    def requested_neighborhood_id
      raw = params[:neighborhood_id]
      return nil if raw.nil? || raw == ""
      return raw if raw.is_a?(String) && Neighborhood.active_neighborhoods.exists?(id: raw)

      render_error("invalid_neighborhood", :unprocessable_entity)
      nil
    end
```

- [ ] **Step 5: Controller e rotas**

Substitua `app/controllers/citizen_api/people_controller.rb` por:

```ruby
# GET  /citizen/people — "para quem é esta triagem?": os CPFs ligados ao
#   telefone da sessão, mascarados, com o bairro declarado (ADR 0023) e o
#   perfil (ADR 0027; contratos §3.1).
# POST /citizen/people { cpf, consent_version, birth_date, sex, gender_identity?, neighborhood_id? }
#   — o par nasce COM o perfil (contratos §3.2). LGPD: nenhum CPF gravado sem o
#   termo vigente, nem com valor inválido. Par existente: 200 sem sobrescrever.
# POST /citizen/people/:id/profile { birth_date, sex, gender_identity } — corrige
#   o perfil declarado; o verificado só muda no posto (409 profile_verified).
# POST /citizen/people/:id/neighborhood { neighborhood_id | null } — troca o
#   bairro ("Trocar bairro" no wpda). Não muda triagem antiga: a cópia de cada
#   uma é imutável. Sem a chave: 422 (só null explícito apaga).
module CitizenApi
  class PeopleController < BaseController
    def index
      people = current_citizen_session.citizens.includes(:neighborhood).order(:created_at)
      render json: { people: people.map { |c| person_json(c) } }
    end

    def create
      return render_error("consent_outdated", :conflict) unless params[:consent_version].to_s == Consents.current_version

      values = Citizens::ProfileValues.call(birth_date: params[:birth_date], sex: params[:sex],
                                            gender_identity: params[:gender_identity])
      return render_error(values.reason, :unprocessable_entity) if values.failure?

      neighborhood_id = requested_neighborhood_id
      return if performed?

      result = Citizens::RegisterPerson.call(phone: current_citizen_session.phone, cpf: params[:cpf], profile: values.payload)
      return render_error(result.reason, :unprocessable_entity) if result.failure?

      citizen = result.payload[:citizen]
      created = result.payload[:created]
      if created && neighborhood_id
        set = Citizens::SetNeighborhood.call(citizen: citizen, neighborhood_id: neighborhood_id)
        return render_error(set.reason, :unprocessable_entity) if set.failure?
      end

      render json: { person: person_json(citizen.reload) }, status: created ? :created : :ok
    end

    def profile
      citizen = current_citizen_session.citizens.find_by(id: params[:id])
      return render_error("not_found", :not_found) unless citizen

      result = Citizens::SetProfile.call(citizen: citizen, birth_date: params[:birth_date], sex: params[:sex],
                                         gender_identity: params[:gender_identity])
      if result.failure?
        return render_error(result.reason, result.reason == :profile_verified ? :conflict : :unprocessable_entity)
      end

      render json: { person: person_json(citizen.reload) }
    end

    def neighborhood
      citizen = current_citizen_session.citizens.find_by(id: params[:id])
      return render_error("not_found", :not_found) unless citizen

      raw = params[:neighborhood_id]
      unless params.key?(:neighborhood_id) && (raw.nil? || raw.is_a?(String))
        return render_error("invalid_neighborhood", :unprocessable_entity)
      end

      result = Citizens::SetNeighborhood.call(citizen: citizen, neighborhood_id: raw)
      return render_error(result.reason, :unprocessable_entity) if result.failure?

      render json: { person: person_json(citizen.reload) }
    end

    private

    def person_json(citizen)
      {
        id: citizen.id, cpf_masked: citizen.cpf_masked, verification_level: citizen.verification_level,
        neighborhood: citizen.neighborhood && { id: citizen.neighborhood.id, name: citizen.neighborhood.name },
        profile: Citizens::ProfileJson.call(citizen)
      }
    end
  end
end
```

Em `config/routes.rb`, no bloco `scope "/citizen"`, depois de `get  "people",                     to: "people#index"`:

```ruby
    post "people",                     to: "people#create"
    post "people/:id/profile",         to: "people#profile"
```

- [ ] **Step 6: Rode**

Run: `docker compose exec -T -w /rails/.claude/mod15 api bundle exec rspec spec/requests/citizen_api spec/commands/citizens/register_person_spec.rb spec/requests/cookie_write_requires_json_spec.rb`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod15 add app/controllers/citizen_api/people_controller.rb app/controllers/citizen_api/base_controller.rb app/controllers/citizen_api/conversations_controller.rb app/commands/citizens/register_person.rb config/routes.rb spec/requests/citizen_api/profile_spec.rb spec/commands/citizens/register_person_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod15 commit -m "feat: register citizen pairs with a profile and expose it on people

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 8: Perfil conferido na validação presencial

**Files:**
- Modify: `app/commands/citizens/verify.rb`
- Modify: `app/controllers/attendance_controller.rb`
- Modify (chamadas existentes): `spec/requests/attendance_spec.rb`, `spec/requests/citizen_api/verified_history_spec.rb`, `spec/commands/citizens/revoke_verification_spec.rb`
- Test: `spec/commands/citizens/verify_spec.rb`, `spec/requests/attendance_spec.rb`

**Interfaces:**
- Consumes: `Citizens::ProfileValues`, `Citizens::ProfileJson`.
- Produces: `Citizens::Verify.call(cpf:, code:, document_checked:, by:, birth_date:, sex:, gender_identity: Citizens::Verify::UNCHANGED) -> Result` (falhas novas `:invalid_birth_date`, `:invalid_sex`, `:invalid_gender_identity`, conferidas antes de consumir o código); grava o perfil no par validado com `profile_source: "verified"` e publica `citizen.profile_changed`. `POST /attendance/lookup` devolve `citizen.profile` (formato 3.1). `Citizens::Verify.record!` não muda.

- [ ] **Step 1: Escreva as specs que falham e ajuste as chamadas existentes**

Em `spec/commands/citizens/verify_spec.rb`, troque o helper `verify` por:

```ruby
  def verify(code, checked: true, **profile)
    described_class.call(cpf: "529.982.247-25", code: code, document_checked: checked, by: verifier,
                         birth_date: "1963-04-02", sex: "female", **profile)
  end
```

e acrescente antes do `end` final:

```ruby
  describe "perfil conferido no documento (ADR 0027)" do
    it "grava o perfil verified no par validado e publica só o id" do
      citizen.update!(birth_date: "1963-04-03", sex: "male", gender_identity: "cis_man", profile_source: "declared")
      expect(verify(issue_code_for(citizen))).to be_ok
      expect(citizen.reload).to have_attributes(birth_date: "1963-04-02", sex: "female", gender_identity: "cis_man",
                                                profile_source: "verified")
      expect(DomainEvent.where(name: "citizen.profile_changed").sole.payload).to eq("citizen_id" => citizen.id)
    end

    it "gender_identity presente (inclusive nil) substitui o declarado" do
      citizen.update!(birth_date: "1963-04-02", sex: "female", gender_identity: "cis_woman", profile_source: "declared")
      verify(issue_code_for(citizen), gender_identity: nil)
      expect(citizen.reload.gender_identity).to be_nil
    end

    it "valor inválido: motivo e o código continua usável" do
      code = issue_code_for(citizen)
      expect(verify(code, birth_date: "2999-01-01").reason).to eq(:invalid_birth_date)
      expect(verify(code, sex: "x").reason).to eq(:invalid_sex)
      expect(citizen.reload).to be_verification_level_declared
      expect(CitizenVerificationCode.usable.where(citizen: citizen)).to exist
    end
  end
```

Nas chamadas diretas a `Citizens::Verify.call(...)` de `spec/requests/citizen_api/verified_history_spec.rb` (linha 24) e `spec/commands/citizens/revoke_verification_spec.rb` (linhas 11 e 48), acrescente `birth_date: "1963-04-02", sex: "female"` aos argumentos.

Em `spec/requests/attendance_spec.rb`, acrescente `birth_date: "1963-04-02", sex: "female"` a cada `json_post "/attendance/verifications", ...` (hoje nas linhas 28, 55, 62, 78 e 98) e, antes do `end` final, os exemplos:

```ruby
  it "validação sem perfil conferido: 422 com o motivo" do
    sign_in_as(staff_with("perfil-#{SecureRandom.hex(3)}@cidade.gov.br", "citizen_verifier"))
    citizen = Citizen.create!(cpf: "52998224725", phone: "+5541998765432")
    json_post "/attendance/verifications", cpf: citizen.cpf, code: issue_code_for(citizen), document_checked: true
    expect(response).to have_http_status(:unprocessable_entity)
    expect(JSON.parse(response.body)["error"]).to eq("invalid_birth_date")
  end

  it "lookup mostra o perfil declarado para o atendente conferir" do
    sign_in_as(staff_with("perfil2-#{SecureRandom.hex(3)}@cidade.gov.br", "citizen_verifier"))
    citizen = Citizen.create!(cpf: "52998224725", phone: "+5541998765432", birth_date: "1963-04-02", sex: "female",
                              profile_source: "declared")
    json_post "/attendance/lookup", cpf: citizen.cpf, code: issue_code_for(citizen)
    expect(JSON.parse(response.body).dig("citizen", "profile"))
      .to eq("birth_date" => "1963-04-02", "sex" => "female", "gender_identity" => nil, "profile_source" => "declared")
  end
```

(Se o arquivo já define um `let` de atendente com `citizen_verifier`, use-o no lugar de `staff_with(...)`.)

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod15 api bundle exec rspec spec/commands/citizens/verify_spec.rb spec/requests/attendance_spec.rb`
Expected: FAIL (`unknown keywords: :birth_date, :sex`).

- [ ] **Step 3: Implemente**

Substitua `app/commands/citizens/verify.rb` por:

```ruby
# Valida o par no balcão (spec 2026-09-24 §2, §4). Confere e consome o código
# sob lock: dois atendentes com o mesmo código → só um valida.
# ADR 0027 (spec 2026-10-05 §5.4): o atendente confere no documento a data de
# nascimento e o sexo (e, se quiser, a identidade de gênero); o perfil do par
# validado passa a `verified` e só muda no posto. gender_identity ausente
# (UNCHANGED) mantém o declarado. Os valores são conferidos ANTES do código:
# um erro de digitação não gasta o código.
module Citizens
  class Verify
    UNCHANGED = Object.new.freeze

    def self.call(cpf:, code:, document_checked:, by:, birth_date:, sex:, gender_identity: UNCHANGED)
      return Result.fail(:document_check_required) unless document_checked == true

      keep_identity = gender_identity.equal?(UNCHANGED)
      values = ProfileValues.call(birth_date: birth_date, sex: sex, gender_identity: keep_identity ? nil : gender_identity)
      return values if values.failure?

      result = nil
      ApplicationRecord.transaction do
        match = VerificationCodeMatch.call(cpf: cpf, code: code, lock: true)
        next result = match if match.failure?

        citizen = match.payload[:citizen]
        if (active = citizen.active_verification)
          next result = Result.fail(:already_verified, details: { verified_at: active.verified_at })
        end

        match.payload[:verification_code].update!(consumed_at: Time.current)
        verification = record!(citizen: citizen, by: by)
        apply_profile!(citizen, values.payload, keep_identity: keep_identity)
        result = Result.ok(verification: verification)
      end
      result
    rescue ActiveRecord::RecordNotUnique
      Result.fail(:already_verified)
    end

    # Cria a validação dentro de uma transação que o chamador já abriu (usado
    # também pelo check-in, spec 2026-09-24-citizen-attendance-check-in §2.6).
    # Não mexe no perfil: o check-in não confere documento.
    def self.record!(citizen:, by:)
      verification = CitizenVerification.create!(citizen: citizen, verified_by_user: by, verified_at: Time.current)
      citizen.update!(verification_level: "verified")
      DomainEvents.publish("citizen.verified", citizen_id: citizen.id, verification_id: verification.id,
                                               verified_by_user_id: by.id)
      verification
    end

    def self.apply_profile!(citizen, values, keep_identity:)
      attributes = values.merge(profile_source: "verified")
      attributes.delete(:gender_identity) if keep_identity
      citizen.update!(attributes)
      DomainEvents.publish("citizen.profile_changed", citizen_id: citizen.id)
    end
  end
end
```

Em `app/controllers/attendance_controller.rb`:
- no comentário do topo, a linha de `POST /attendance/verifications` passa a `{cpf, code, document_checked, birth_date, sex, gender_identity?}  citizen_verifier`;
- `verify` passa a:

```ruby
  def verify
    result = Citizens::Verify.call(
      cpf: params[:cpf], code: params[:code], document_checked: params[:document_checked] == true, by: Current.user,
      birth_date: params[:birth_date], sex: params[:sex],
      gender_identity: params.key?(:gender_identity) ? params[:gender_identity] : Citizens::Verify::UNCHANGED
    )
    return render_failure(result, ERROR_STATUS) if result.failure?

    v = result.payload[:verification]
    render json: { verification: { id: v.id, citizen_id: v.citizen_id, verified_at: v.verified_at.iso8601 } },
           status: :created
  end
```

- em `ERROR_STATUS`, acrescente `invalid_birth_date: :unprocessable_entity, invalid_sex: :unprocessable_entity, invalid_gender_identity: :unprocessable_entity`;
- em `lookup`, o hash `citizen:` ganha `profile: Citizens::ProfileJson.call(citizen)` depois de `verification_level: citizen.verification_level`.

- [ ] **Step 4: Rode**

Run: `docker compose exec -T -w /rails/.claude/mod15 api bundle exec rspec spec/commands/citizens spec/requests/attendance_spec.rb spec/requests/attendance_contract_spec.rb spec/requests/citizen_api spec/commands/attendances/check_in_spec.rb`
Expected: PASS. Se `spec/requests/attendance_contract_spec.rb` fixar o formato exato de `citizen` no lookup, acrescente `"profile" => nil` (ou o perfil do cidadão do exemplo) ao hash esperado.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod15 add app/commands/citizens/verify.rb app/controllers/attendance_controller.rb spec/commands/citizens/verify_spec.rb spec/commands/citizens/revoke_verification_spec.rb spec/requests/attendance_spec.rb spec/requests/attendance_contract_spec.rb spec/requests/citizen_api/verified_history_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod15 commit -m "feat: confirm the citizen profile at the counter verification

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Fatia 3 — Oferta, início e sugestões (F-15.3, F-15.5, F-15.6)

### Task 9: `Triages::Offer` — a regra de oferta

**Files:**
- Create: `app/services/triages/offer.rb`
- Test: `spec/services/triages/offer_spec.rb` (função pura, tabela de casos), `spec/services/triages/offer_for_spec.rb` (carga do banco)

**Interfaces:**
- Consumes: `Protocols::Condition.eval`, `Protocols::ConditionContext.build` (Task 1); `Citizen#profile_context` (Task 5); `TriageOffer` (Task 4).
- Produces:
  - `Triages::Offer::Row = Data.define(:enabled, :position, :restriction, :available_from, :available_until)`.
  - `Triages::Offer::Item = Data.define(:protocol_name, :title, :summary, :state, :position, :last_completed_on, :next_available_on)` com `#available?` e `#recent?`; `state` ∈ `"available"`, `"recent"`.
  - `Triages::Offer.evaluate(protocols:, rows:, context:, last_completed:, on:) -> Array<Item>` (puro; `protocols` = `[{ name:, offer: }]`, `rows` = `{ name => Row }`, `last_completed` = `{ name => Date }`), ordenado por `position` (sem linha por último), depois `title`.
  - `Triages::Offer.for(citizen:, on: Time.zone.today) -> Array<Item>`; `Triages::Offer.available?(citizen:, protocol_name:, on: Time.zone.today) -> Boolean`; `Triages::Offer.title_for(definition, name) -> String`.

- [ ] **Step 1: Escreva a tabela de casos (falha)**

```ruby
# spec/services/triages/offer_spec.rb
require "rails_helper"

# Regra de oferta do ADR 0027 (spec 2026-10-05 §5.1), como função pura: data
# fixa de propósito (on: é argumento, não relógio).
RSpec.describe Triages::Offer, ".evaluate" do
  let(:on) { Date.new(2026, 10, 5) }
  let(:idoso) { { "title" => "Saúde do idoso", "summary" => "Anual.", "eligibility" => { "gte" => ["profile.age", 60] }, "retake_after_days" => 365 } }

  def protocol(name, offer = nil) = { name: name, offer: offer }
  def row(**attrs) = described_class::Row.new(**{ enabled: true, position: 0, restriction: nil, available_from: nil, available_until: nil }.merge(attrs))
  def ctx(age: 62, sex: "female", neighborhood_id: nil)
    Protocols::ConditionContext.build(profile: { age: age, sex: sex }, citizen: { neighborhood_id: neighborhood_id })
  end

  def evaluate(protocols, rows: {}, context: ctx, last_completed: {})
    described_class.evaluate(protocols: protocols, rows: rows, context: context, last_completed: last_completed, on: on)
  end

  def names(...) = evaluate(...).map(&:protocol_name)

  it "sem linha: em oferta só sem elegibilidade (catálogo vazio = hoje)" do
    expect(names([ protocol("respiratoria"), protocol("idoso", idoso) ])).to eq([ "respiratoria" ])
  end

  it "com linha: enabled e dentro do período; pausado some" do
    expect(names([ protocol("idoso", idoso) ], rows: { "idoso" => row })).to eq([ "idoso" ])
    expect(names([ protocol("idoso", idoso) ], rows: { "idoso" => row(enabled: false) })).to eq([])
  end

  it "período: bordas inclusivas" do
    {
      { available_from: on } => [ "x" ], { available_until: on } => [ "x" ],
      { available_from: on + 1 } => [], { available_until: on - 1 } => [],
      { available_from: on - 10, available_until: on + 10 } => [ "x" ]
    }.each do |period, expected|
      expect(names([ protocol("x") ], rows: { "x" => row(**period) })).to eq(expected), period.inspect
    end
  end

  it "idade nas bordas: 59 fora, 60 dentro; sexo pela elegibilidade" do
    rows = { "idoso" => row }
    expect(names([ protocol("idoso", idoso) ], rows: rows, context: ctx(age: 59))).to eq([])
    expect(names([ protocol("idoso", idoso) ], rows: rows, context: ctx(age: 60))).to eq([ "idoso" ])
    mulher = { "eligibility" => { "eq" => ["profile.sex", "female"] } }
    expect(names([ protocol("mulher", mulher) ], rows: { "mulher" => row }, context: ctx(sex: "male"))).to eq([])
  end

  it "restrição soma com E e nunca amplia" do
    centro = "0b6f6c1e-9f1a-4d8b-9a4c-1f2e3d4c5b6a"
    no_centro = row(restriction: { "in" => ["citizen.neighborhood_id", [centro]] })
    expect(names([ protocol("idoso", idoso) ], rows: { "idoso" => no_centro }, context: ctx(neighborhood_id: centro))).to eq([ "idoso" ])
    expect(names([ protocol("idoso", idoso) ], rows: { "idoso" => no_centro }, context: ctx)).to eq([])
    tudo = row(restriction: { "gte" => ["profile.age", 0] })
    expect(names([ protocol("idoso", idoso) ], rows: { "idoso" => tudo }, context: ctx(age: 30))).to eq([])
  end

  it "restrição quebrada (gravada por fora) deixa o protocolo fora, sem levantar" do
    [ "lixo", { "xyz" => 1 }, { "all" => "x" }, { "gte" => ["profile.age"] } ].each do |broken|
      expect(names([ protocol("x") ], rows: { "x" => row(restriction: broken) })).to eq([]), broken.inspect
    end
  end

  it "intervalo: recent com next_available_on; no dia exato volta a available" do
    item = evaluate([ protocol("idoso", idoso) ], rows: { "idoso" => row }, last_completed: { "idoso" => Date.new(2026, 3, 10) }).sole
    expect(item).to have_attributes(state: "recent", last_completed_on: Date.new(2026, 3, 10),
                                    next_available_on: Date.new(2027, 3, 10))
    item = evaluate([ protocol("idoso", idoso) ], rows: { "idoso" => row }, last_completed: { "idoso" => on - 365 }).sole
    expect(item).to have_attributes(state: "available", next_available_on: nil)
    item = evaluate([ protocol("sem-intervalo") ], last_completed: { "sem-intervalo" => on }).sole
    expect(item.state).to eq("available")
  end

  it "perfil ausente: elegibilidade falsa, protocolo sem elegibilidade continua" do
    empty = Protocols::ConditionContext.build
    expect(names([ protocol("idoso", idoso), protocol("x") ], rows: { "idoso" => row }, context: empty)).to eq([ "x" ])
  end

  it "ordem: position do catálogo, depois título; sem linha por último; título cai para o name" do
    items = evaluate([ protocol("c-sem-linha"), protocol("b", { "title" => "Bê" }), protocol("a", { "title" => "Á" }) ],
                     rows: { "a" => row(position: 2), "b" => row(position: 1) })
    expect(items.map(&:protocol_name)).to eq(%w[b a c-sem-linha])
    expect(items.map(&:title)).to eq([ "Bê", "Á", "c-sem-linha" ])
    expect(items.last.summary).to be_nil
  end

  it "offer malformado no banco é tratado como ausente" do
    expect(names([ protocol("x", "lixo") ])).to eq([ "x" ])
  end
end
```

```ruby
# spec/services/triages/offer_for_spec.rb
require "rails_helper"

# Triages::Offer.for carrega do banco: protocolos ativos, linhas do catálogo,
# perfil do par e a última conclusão por protocolo (data local da cidade).
RSpec.describe Triages::Offer, ".for" do
  include ActiveSupport::Testing::TimeHelpers
  before { Current.city = TEST_CITY_A }
  after { Current.reset; Rails.cache.clear }

  let(:admin) { staff_with("oferta-#{SecureRandom.hex(3)}@cidade.gov.br") }
  let!(:idoso) do
    active_protocol!("saude-do-idoso", offer: { "title" => "Saúde do idoso", "eligibility" => { "gte" => ["profile.age", 60] },
                                                "retake_after_days" => 365 })
  end
  let!(:respiratoria) { create_default_protocol! }

  before { TriageOffer.create!(protocol_name: "saude-do-idoso", updated_by_user: admin, position: 1) }

  it "avó vê os dois; neto só o de todos; nome inexistente nunca está em oferta" do
    avo = profiled_citizen!(age: 62)
    neto = profiled_citizen!(age: 8, sex: "male", cpf: CampaignHistory.cpf_for("neto"))
    expect(described_class.for(citizen: avo).map(&:protocol_name)).to eq(%w[saude-do-idoso triage-respiratoria])
    expect(described_class.for(citizen: neto).map(&:protocol_name)).to eq(%w[triage-respiratoria])
    expect(described_class.available?(citizen: neto, protocol_name: "saude-do-idoso")).to be(false)
    expect(described_class.available?(citizen: avo, protocol_name: "fantasma")).to be(false)
  end

  it "conclusão há 100 dias deixa recent; triagem de outro par não conta" do
    avo = profiled_citizen!(age: 62)
    completed_triage!(avo, "saude-do-idoso", at: 100.days.ago)
    outro = profiled_citizen!(age: 70, phone: "+5541900000001")
    completed_triage!(outro, "saude-do-idoso", at: 1.day.ago)
    item = described_class.for(citizen: avo).find { |i| i.protocol_name == "saude-do-idoso" }
    expect(item).to have_attributes(state: "recent", next_available_on: (100.days.ago.to_date + 365))
    expect(described_class.for(citizen: outro).find { |i| i.protocol_name == "saude-do-idoso" }.state).to eq("recent")
  end

  it "faz 60 hoje no fuso de Manaus, não no de São Paulo" do
    manaus = TEST_CITY_A.dup.tap { |c| c.time_zone = "America/Manaus" }
    CityConnection.with(manaus) do
      travel_to(Time.utc(2026, 10, 6, 3, 30)) do # 23h30 de 05/10 em Manaus; 00h30 de 06/10 em SP
        avo = Citizen.create!(cpf: "52998224725", phone: "+5541998765432", birth_date: "1966-10-06", sex: "female",
                              profile_source: "declared")
        expect(described_class.available?(citizen: avo, protocol_name: "saude-do-idoso")).to be(false)
      end
      travel_to(Time.utc(2026, 10, 6, 4, 1)) do # 00h01 de 06/10 em Manaus
        avo = Citizen.find_by!(cpf: "52998224725", phone: "+5541998765432")
        expect(described_class.available?(citizen: avo, protocol_name: "saude-do-idoso")).to be(true)
      end
    end
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod15 api bundle exec rspec spec/services/triages/offer_spec.rb spec/services/triages/offer_for_spec.rb`
Expected: FAIL (`uninitialized constant Triages::Offer`).

- [ ] **Step 3: Implemente**

```ruby
# app/services/triages/offer.rb
# Regra de oferta (ADR 0027; spec 2026-10-05 §5.1). Um protocolo com versão
# active está em oferta para o par quando:
#   1. sem linha no catálogo: só se não tiver offer.eligibility (o de hoje);
#      com linha: enabled e dentro de available_from..available_until;
#   2. offer.eligibility verdadeira no contexto do par;
#   3. restriction verdadeira (ausente = verdadeira) — soma com E, nunca amplia;
#   4. intervalo: recent enquanto hoje < última conclusão + retake_after_days.
# `evaluate` é PURA (dados já carregados, data por argumento) e testável em
# tabela; `for` só carrega do banco da cidade e chama.
module Triages
  module Offer
    Row = Data.define(:enabled, :position, :restriction, :available_from, :available_until)
    Item = Data.define(:protocol_name, :title, :summary, :state, :position, :last_completed_on, :next_available_on) do
      def available? = state == "available"
      def recent? = state == "recent"
    end

    module_function

    def evaluate(protocols:, rows:, context:, last_completed:, on:)
      items = protocols.filter_map do |protocol|
        name = protocol.fetch(:name)
        offer = protocol[:offer].is_a?(Hash) ? protocol[:offer] : {}
        row = rows[name]
        next unless listed?(row, offer, on)
        next unless offer["eligibility"].nil? || Protocols::Condition.eval(offer["eligibility"], context)
        next unless row.nil? || row.restriction.nil? || Protocols::Condition.eval(row.restriction, context)

        item(name, offer, row, last_completed[name], on)
      end
      items.sort_by { |i| [ i.position.nil? ? 1 : 0, i.position || 0, i.title ] }
    end

    def listed?(row, offer, on)
      return offer["eligibility"].nil? if row.nil?

      row.enabled && (row.available_from.nil? || row.available_from <= on) &&
        (row.available_until.nil? || on <= row.available_until)
    end

    def item(name, offer, row, last_on, on)
      days = offer["retake_after_days"]
      next_on = last_on && days.is_a?(Integer) && days.positive? ? last_on + days : nil
      recent = next_on.present? && on < next_on
      Item.new(protocol_name: name, title: offer["title"].presence || name, summary: offer["summary"].presence,
               state: recent ? "recent" : "available", position: row&.position, last_completed_on: last_on,
               next_available_on: recent ? next_on : nil)
    end

    def title_for(definition, name)
      offer = definition.is_a?(Hash) && definition["offer"].is_a?(Hash) ? definition["offer"] : {}
      offer["title"].presence || name
    end

    def for(citizen:, on: Time.zone.today)
      protocols = ProtocolDefinition.active.pluck(:name, :definition).map do |name, definition|
        { name: name, offer: definition.is_a?(Hash) ? definition["offer"] : nil }
      end
      rows = TriageOffer.all.to_h do |r|
        [ r.protocol_name, Row.new(enabled: r.enabled, position: r.position, restriction: r.restriction,
                                   available_from: r.available_from, available_until: r.available_until) ]
      end
      context = Protocols::ConditionContext.build(profile: citizen.profile_context(on: on),
                                                  citizen: { neighborhood_id: citizen.neighborhood_id })
      evaluate(protocols: protocols, rows: rows, context: context, last_completed: last_completed(citizen), on: on)
    end

    def available?(citizen:, protocol_name:, on: Time.zone.today)
      self.for(citizen: citizen, on: on).any? { |i| i.protocol_name == protocol_name.to_s && i.available? }
    end

    # Data local (fuso da cidade) do created_at da última conclusão do PAR.
    def last_completed(citizen)
      Triage.joins(:conversation).where(conversations: { citizen_id: citizen.id }).status_completed
            .group(:protocol_name).maximum(:created_at)
            .transform_values { |at| at.in_time_zone.to_date }
    end
  end
end
```

- [ ] **Step 4: Rode**

Run: `docker compose exec -T -w /rails/.claude/mod15 api bundle exec rspec spec/services/triages`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod15 add app/services/triages/offer.rb spec/services/triages/offer_spec.rb spec/services/triages/offer_for_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod15 commit -m "feat: add the triage offer rule as a pure function over loaded data

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 10: Início por nome — `StartTriage`, `StartConversation` e `POST /citizen/conversations`

**Files:**
- Modify: `app/commands/start_triage.rb`, `app/commands/citizens/start_conversation.rb`, `app/controllers/citizen_api/conversations_controller.rb`
- Modify: `spec/support/citizen_request_helpers.rb`
- Modify (specs existentes que começavam por CPF): `spec/requests/citizen_api/triage_flow_spec.rb`, `spec/requests/citizen_api/neighborhood_spec.rb`, `spec/requests/citizen_api/isolation_spec.rb`, `spec/requests/citizen_api/reference_units_spec.rb`, `spec/invariants/territory_invariants_spec.rb`
- Test: `spec/commands/start_triage_by_name_spec.rb`, `spec/commands/citizens/start_conversation_spec.rb`, `spec/commands/citizens/start_conversation_concurrency_spec.rb`, `spec/requests/citizen_api/start_by_name_spec.rb`

**Interfaces:**
- Consumes: `Triages::Offer.available?` (Task 9), `TriageSuggestion` (Task 4), `Citizen#profile?` (Task 5).
- Produces:
  - `StartTriage.call(conversation:, protocol_name: StartTriage::DEFAULT_PROTOCOL_NAME) -> Result` — reasons `:no_protocol` (conversa sem cidadão), `:not_offered` (com cidadão). Com cidadão: `citizen.lock!`, confere a oferta, cria a triagem e marca `taken` a sugestão pendente daquele protocolo (`taken_triage_id`, `resolved_at`), tudo numa transação.
  - `Citizens::StartConversation.call(citizen:, consent_version:, session_id:, protocol_name: StartTriage::DEFAULT_PROTOCOL_NAME) -> Result` — reasons novas `:triage_in_progress`, `:not_offered`; mesmo protocolo em andamento → `resumed: true`.
  - `POST /citizen/conversations`: 422 `protocol_name_required`; 409 `profile_required`, `not_offered`, `triage_in_progress`.
  - Helper de request: `start_citizen_triage(cpf: "529.982.247-25", protocol_name: StartTriage::DEFAULT_PROTOCOL_NAME, birth_date: "1980-05-10", sex: "female", **people_params) -> Hash` (corpo da última resposta).

- [ ] **Step 1: Escreva as specs novas (falham)**

```ruby
# spec/commands/start_triage_by_name_spec.rb
require "rails_helper"

# ADR 0027 (spec 2026-10-05 §5.2): a triagem começa pelo nome escolhido, se
# estiver em oferta para o par; a sugestão pendente daquele protocolo vira
# taken na mesma transação; a triagem aponta a versão ativa exata (ADR 0010).
RSpec.describe StartTriage, "por nome" do
  before { Current.city = TEST_CITY_A }
  after { Current.reset; Rails.cache.clear }

  let(:admin) { staff_with("inicio-#{SecureRandom.hex(3)}@cidade.gov.br") }
  let!(:idoso) do
    active_protocol!("saude-do-idoso", offer: { "eligibility" => { "gte" => ["profile.age", 60] }, "retake_after_days" => 365 })
  end
  let!(:offer_row) { TriageOffer.create!(protocol_name: "saude-do-idoso", updated_by_user: admin) }
  let(:avo) { profiled_citizen!(age: 62) }

  # Uma conversa web ativa por par (índice único): reaproveita a que existe.
  def conversation_for(citizen)
    Conversation.channel_web.find_by(citizen: citizen, state: Conversation::ACTIVE_STATES) ||
      Conversation.create!(channel: "web", citizen: citizen, phone: citizen.phone, state: :consented)
  end
  def start(citizen, name = "saude-do-idoso") = described_class.call(conversation: conversation_for(citizen), protocol_name: name)

  it "abre a triagem do protocolo escolhido na versão ativa" do
    triage = start(avo).payload[:triage]
    expect(triage).to have_attributes(protocol_name: "saude-do-idoso", protocol_definition_id: idoso.id, current_step: "q1")
  end

  it "fora de oferta: :not_offered e nenhuma triagem (idade, pausa, intervalo, inexistente)" do
    neto = profiled_citizen!(age: 8, sex: "male", cpf: CampaignHistory.cpf_for("neto"))
    expect(start(neto).reason).to eq(:not_offered)
    offer_row.update!(enabled: false)
    expect(start(avo).reason).to eq(:not_offered)
    offer_row.update!(enabled: true)
    completed_triage!(avo, "saude-do-idoso", at: 10.days.ago)
    expect(start(avo).reason).to eq(:not_offered)
    expect(start(avo, "fantasma").reason).to eq(:not_offered)
    expect(Triage.where(status: "in_progress")).to be_empty
  end

  it "sugestão pendente do protocolo vira taken com a triagem nova" do
    create_default_protocol!
    source = completed_triage!(avo, StartTriage::DEFAULT_PROTOCOL_NAME)
    suggestion = TriageSuggestion.create!(citizen: avo, source_triage: source, protocol_name: "saude-do-idoso")
    triage = start(avo).payload[:triage]
    expect(suggestion.reload).to have_attributes(status: "taken", taken_triage_id: triage.id, resolved_at: be_present)
  end
end
```

Acrescente a `spec/commands/citizens/start_conversation_spec.rb` (antes do `end` final):

```ruby
  describe "por nome (ADR 0027)" do
    before do
      create_default_protocol! unless ProtocolDefinition.exists?(name: StartTriage::DEFAULT_PROTOCOL_NAME, status: "active")
      active_protocol!("saude-mental")
    end
    after { Rails.cache.clear }

    let(:par) { profiled_citizen!(age: 30, phone: "+5541977770001") }

    def start(name) = described_class.call(citizen: par, consent_version: Consents.current_version, session_id: "s", protocol_name: name)

    it "mesmo protocolo em andamento retoma; outro protocolo: :triage_in_progress" do
      first = start("saude-mental")
      expect(first.payload[:resumed]).to be(false)
      expect(start("saude-mental").payload).to include(resumed: true, triage: first.payload[:triage])
      expect(start(StartTriage::DEFAULT_PROTOCOL_NAME).reason).to eq(:triage_in_progress)
    end

    it "não oferecido: :not_offered, sem triagem" do
      expect(start("fantasma").reason).to eq(:not_offered)
      expect(Triage.joins(:conversation).where(conversations: { citizen_id: par.id })).to be_empty
    end
  end
```

```ruby
# spec/commands/citizens/start_conversation_concurrency_spec.rb
require "rails_helper"

# Review Focus 4 (ADR 0027 §5.2): dois toques ou duas abas ao mesmo tempo.
# Threads reais contra TEST_CITY_A (sem fixture transacional), como
# spec/commands/attendances/call_next_concurrency_spec.rb; o after apaga o que
# commitou.
RSpec.describe Citizens::StartConversation, "concorrência" do
  self.use_transactional_tests = false

  let(:ids) { {} }

  def in_city(&block) = CityConnection.with(TEST_CITY_A) { Current.set(city: TEST_CITY_A, &block) }

  before do
    in_city do
      ids[:started_at] = Time.current
      tag = SecureRandom.hex(4)
      ids[:names] = [ "corrida-a-#{tag}", "corrida-b-#{tag}" ]
      ids[:protocols] = ids[:names].map { |name| active_protocol!(name).id }
      citizen = Citizen.create!(cpf: CampaignHistory.cpf_for("corrida-#{tag}"),
                                phone: "+55419#{format('%08d', 30_000_000 + SecureRandom.random_number(1_000_000))}",
                                birth_date: birth_date_for(40), sex: "female", profile_source: "declared")
      ids[:citizen] = citizen.id
    end
  end

  after do
    in_city do
      ApplicationRecord.transaction do
        ApplicationRecord.connection.execute("SET LOCAL session_replication_role = replica")
        conversation_ids = Conversation.where(citizen_id: ids[:citizen]).pluck(:id)
        Triage.where(conversation_id: conversation_ids).delete_all
        Consent.where(conversation_id: conversation_ids).delete_all
        Conversation.where(id: conversation_ids).delete_all
        Citizen.where(id: ids[:citizen]).delete_all
        ProtocolDefinition.where(id: ids[:protocols]).delete_all
        DomainEvent.where("occurred_at >= ?", ids[:started_at]).delete_all
      end
    end
    Rails.cache.clear
  end

  def race(*names)
    go = Queue.new
    threads = names.map do |name|
      Thread.new do
        in_city do
          go.pop
          described_class.call(citizen: Citizen.find(ids[:citizen]), consent_version: Consents.current_version,
                               session_id: "corrida", protocol_name: name)
        end
      end
    end
    names.size.times { go << true }
    threads.map { |t| t.join(10) ? t.value : raise("thread presa") }
  end

  def triages = in_city { Triage.joins(:conversation).where(conversations: { citizen_id: ids[:citizen] }).count }

  it "o mesmo protocolo duas vezes: uma triagem, a outra retoma" do
    results = race(ids[:names].first, ids[:names].first)
    expect(results.map { |r| r.payload[:resumed] }).to contain_exactly(false, true)
    expect(triages).to eq(1)
  end

  it "protocolos diferentes: uma triagem e um :triage_in_progress" do
    results = race(*ids[:names])
    expect(results.map(&:reason)).to contain_exactly(nil, :triage_in_progress)
    expect(triages).to eq(1)
  end
end
```

```ruby
# spec/requests/citizen_api/start_by_name_spec.rb
require "rails_helper"

# Contratos §3.5 (ADR 0027): POST /citizen/conversations exige protocol_name e
# perfil, e confere a oferta.
RSpec.describe "Início de triagem por nome", type: :request do
  before do
    create_default_protocol!
    active_protocol!("saude-do-idoso", offer: { "eligibility" => { "gte" => ["profile.age", 60] } })
    ConsentTerm.create!(version: "1", body: "Termo de teste", published_at: Time.current)
    sign_in_citizen("+5541998765432")
  end
  after { Rails.cache.clear }

  def body = JSON.parse(response.body)
  def status_and_error = [ response.status, body["error"] ]
  def start(params) = json_post("/citizen/conversations", { consent_version: "1" }.merge(params))

  let(:admin) { staff_with("catalogo-#{SecureRandom.hex(3)}@cidade.gov.br") }

  it "sem protocol_name: 422 protocol_name_required, antes de gravar o CPF" do
    expect { start(cpf: "529.982.247-25") }.not_to change(Citizen, :count)
    expect(status_and_error).to eq([ 422, "protocol_name_required" ])
    [ "", 1, [ "x" ] ].each do |value|
      start(cpf: "529.982.247-25", protocol_name: value)
      expect(status_and_error).to eq([ 422, "protocol_name_required" ]), value.inspect
    end
  end

  it "par sem perfil: 409 profile_required" do
    citizen = Citizen.create!(cpf: "52998224725", phone: "+5541998765432")
    start(citizen_id: citizen.id, protocol_name: StartTriage::DEFAULT_PROTOCOL_NAME)
    expect(status_and_error).to eq([ 409, "profile_required" ])
  end

  it "não oferecido e triagem em andamento: 409" do
    TriageOffer.create!(protocol_name: "saude-do-idoso", updated_by_user: admin)
    neto = profiled_citizen!(age: 8, sex: "male")
    start(citizen_id: neto.id, protocol_name: "saude-do-idoso")
    expect(status_and_error).to eq([ 409, "not_offered" ])

    start(citizen_id: neto.id, protocol_name: StartTriage::DEFAULT_PROTOCOL_NAME)
    expect(response).to have_http_status(:created)
    start(citizen_id: neto.id, protocol_name: "outro-qualquer")
    expect(status_and_error).to eq([ 409, "triage_in_progress" ])
  end

  it "protocolo pausado: not_offered" do
    TriageOffer.create!(protocol_name: StartTriage::DEFAULT_PROTOCOL_NAME, enabled: false, updated_by_user: admin)
    par = profiled_citizen!(age: 30)
    start(citizen_id: par.id, protocol_name: StartTriage::DEFAULT_PROTOCOL_NAME)
    expect(status_and_error).to eq([ 409, "not_offered" ])
  end
end
```

(No exemplo "não oferecido e triagem em andamento", o terceiro pedido usa um nome qualquer: com triagem em andamento, a recusa `triage_in_progress` vem antes da oferta — o nome nem é conferido.)

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod15 api bundle exec rspec spec/commands/start_triage_by_name_spec.rb spec/commands/citizens/start_conversation_spec.rb spec/commands/citizens/start_conversation_concurrency_spec.rb spec/requests/citizen_api/start_by_name_spec.rb`
Expected: FAIL (`unknown keyword: :protocol_name`).

- [ ] **Step 3: `StartTriage` por nome**

Substitua `app/commands/start_triage.rb` por:

```ruby
# Abre a triagem de um protocolo numa conversa já consentida. Usado pela web
# (Citizens::StartConversation, com o nome escolhido no catálogo) e pelo
# WhatsApp descontinuado (ConversationAdvance, com o nome padrão). Ver ADR-0009.
#
# ADR 0027 (spec 2026-10-05 §5.2): com cidadão, trava o par e confere a regra
# de oferta (Triages::Offer) antes de criar — a mesma triagem pedida duas vezes
# em paralelo não fura o intervalo de repetição; a sugestão pendente daquele
# protocolo vira `taken` na mesma transação. Sem cidadão (WhatsApp), como
# antes. Reasons: :no_protocol, :not_offered.
#
# Copia o bairro atual do cidadão na criação (ADR 0023): é a única escrita de
# triages.neighborhood_id — depois, o trigger triages_neighborhood_immutable
# recusa qualquer mudança. Conversa do WhatsApp sem cidadão: sem bairro.
class StartTriage
  DEFAULT_PROTOCOL_NAME = "triage-respiratoria"

  def self.call(conversation:, protocol_name: DEFAULT_PROTOCOL_NAME)
    name = protocol_name.to_s
    citizen = conversation.citizen
    ApplicationRecord.transaction do
      if citizen
        citizen.lock!
        return Result.fail(:not_offered) unless Triages::Offer.available?(citizen: citizen, protocol_name: name)
      end

      record = ProtocolDefinition.find_by(name: name, status: "active")
      return Result.fail(:no_protocol) unless record

      engine = Protocols.current(name: name)
      triage = conversation.triages.create!(
        protocol_definition: record,
        protocol_name: record.name,
        answers: {},
        current_step: engine.start_step_id.to_s,
        status: :in_progress,
        neighborhood_id: citizen&.neighborhood_id
      )
      take_suggestion!(citizen, triage) if citizen
      Result.ok(triage: triage)
    end
  rescue Protocols::NotFound
    Result.fail(:no_protocol)
  end

  def self.take_suggestion!(citizen, triage)
    TriageSuggestion.status_pending.find_by(citizen_id: citizen.id, protocol_name: triage.protocol_name)
                    &.update!(status: "taken", taken_triage_id: triage.id, resolved_at: Time.current)
  end
end
```

- [ ] **Step 4: `StartConversation` por nome**

Em `app/commands/citizens/start_conversation.rb`:
- no comentário do topo, a linha de reasons passa a `# Reasons: :consent_outdated, :no_protocol, :not_offered, :triage_in_progress (e as de GiveConsent).` e acrescente antes dela:

```ruby
# ADR 0027: o cidadão escolhe o protocolo. Uma triagem em andamento por par:
# pedir o MESMO protocolo retoma; pedir outro é :triage_in_progress. O lock é
# cidadão → conversa, a mesma ordem de Citizens::Erase (desvio 5 do plano).
```

- `self.call` e `initialize` passam a:

```ruby
    def self.call(citizen:, consent_version:, session_id:, protocol_name: StartTriage::DEFAULT_PROTOCOL_NAME)
      new(citizen, consent_version.to_s, session_id, protocol_name.to_s).call
    end

    def initialize(citizen, consent_version, session_id, protocol_name)
      @citizen = citizen
      @consent_version = consent_version
      @session_id = session_id
      @protocol_name = protocol_name
    end
```

- em `call`, a linha `conversation.with_lock { result = locked_call(conversation) }` passa a:

```ruby
      ApplicationRecord.transaction do
        @citizen.lock!
        conversation.lock!
        result = locked_call(conversation)
      end
```

- em `locked_call`, as duas linhas que retomam e iniciam passam a:

```ruby
      triage = conversation.triages.status_in_progress.order(created_at: :desc).first
      if triage
        return Result.fail(:triage_in_progress) unless triage.protocol_name == @protocol_name

        return Result.ok(conversation: conversation, triage: triage, resumed: true)
      end

      started = StartTriage.call(conversation: conversation, protocol_name: @protocol_name)
```

- [ ] **Step 5: O controller**

Em `app/controllers/citizen_api/conversations_controller.rb`:
- comentário do topo: `POST /citizen/conversations { citizen_id | cpf, consent_version, protocol_name, neighborhood_id? }`;
- `START_ERRORS` passa a:

```ruby
    START_ERRORS = {
      consent_outdated: :conflict, wrong_state: :conflict, version_mismatch: :conflict,
      not_offered: :conflict, triage_in_progress: :conflict, no_protocol: :service_unavailable
    }.freeze
```

- em `create`, logo depois do bloco do `consent_outdated`, acrescente:

```ruby
      # ADR 0027: o cidadão escolhe o protocolo no catálogo. Antes de
      # resolve_citizen, como o consentimento: um pedido recusado não grava CPF.
      protocol_name = params[:protocol_name]
      unless protocol_name.is_a?(String) && protocol_name.present?
        return render_error("protocol_name_required", :unprocessable_entity)
      end
```

- logo depois de `citizen = resolve_citizen` / `return if performed?`, acrescente:

```ruby
      # O catálogo é por perfil (ADR 0027): sem perfil, o wpda pede antes.
      return render_error("profile_required", :conflict) unless citizen.profile?
```

- a chamada a `Citizens::StartConversation.call` ganha `protocol_name: protocol_name`.

- [ ] **Step 6: Helper de request e specs existentes**

Em `spec/support/citizen_request_helpers.rb`, dentro do módulo, acrescente:

```ruby
  # Módulo 15 (ADR 0027): o par nasce com perfil por POST /citizen/people e a
  # triagem começa pelo nome do protocolo. Devolve o corpo da última resposta
  # (a de /people, se ela falhou).
  def start_citizen_triage(cpf: "529.982.247-25", protocol_name: StartTriage::DEFAULT_PROTOCOL_NAME,
                           birth_date: "1980-05-10", sex: "female", **people_params)
    json_post "/citizen/people", { cpf: cpf, consent_version: "1", birth_date: birth_date, sex: sex }.merge(people_params)
    return JSON.parse(response.body) unless response.successful?

    citizen_id = JSON.parse(response.body).dig("person", "id")
    json_post "/citizen/conversations", citizen_id: citizen_id, consent_version: "1", protocol_name: protocol_name
    JSON.parse(response.body)
  end
```

Ajustes nas specs existentes (o comportamento esperado não muda; só o caminho de início):

1. `spec/requests/citizen_api/triage_flow_spec.rb`:
   - `start_new` passa a:

     ```ruby
       def start_new(cpf = "529.982.247-25")
         start_citizen_triage(cpf: cpf)
       end
     ```

   - no exemplo "retoma a conversa em andamento com 200", a chamada passa a `json_post "/citizen/conversations", citizen_id: started["citizen_id"], consent_version: "1", protocol_name: StartTriage::DEFAULT_PROTOCOL_NAME`;
   - os três exemplos de termo desatualizado/ausente (que postam `cpf:` direto) ficam como estão: o 409 `consent_outdated` vem antes de tudo.
2. `spec/requests/citizen_api/neighborhood_spec.rb`:
   - `start` e `own_citizen` passam a:

     ```ruby
       def start(params)
         return start_citizen_triage(**params.slice(:cpf, :neighborhood_id)) if params.key?(:cpf)

         json_post("/citizen/conversations",
                   { consent_version: "1", protocol_name: StartTriage::DEFAULT_PROTOCOL_NAME }.merge(params))
       end

       def own_citizen(**attrs)
         Citizen.create!({ cpf: "52998224725", phone: "+5541998765432", birth_date: "1980-05-10", sex: "female",
                           profile_source: "declared" }.merge(attrs))
       end
     ```

   - o exemplo "termo desatualizado com bairro" fica como está.
3. `spec/requests/citizen_api/isolation_spec.rb`: a chamada `json_post "/citizen/conversations", citizen_id: other[:citizen].id, consent_version: "1"` ganha `protocol_name: StartTriage::DEFAULT_PROTOCOL_NAME` (continua 404); no exemplo "o mesmo CPF digitado neste telefone…", as duas primeiras linhas passam a `mine = start_citizen_triage(cpf: "529.982.247-25")["citizen_id"]`.
4. `spec/requests/citizen_api/reference_units_spec.rb`: em `start_triage`, a linha do `json_post` passa a `start_citizen_triage(neighborhood_id: neighborhood_id)`.
5. `spec/invariants/territory_invariants_spec.rb` (exemplo "o relatório público nunca traz reference_units nem bairro"): `json_post "/citizen/conversations", cpf: ..., neighborhood_id: centro.id` e a linha seguinte passam a `triage_id = start_citizen_triage(neighborhood_id: centro.id).dig("step", "triage_id")`.

- [ ] **Step 7: Rode**

Run: `docker compose exec -T -w /rails/.claude/mod15 api bundle exec rspec spec/commands/start_triage_spec.rb spec/commands/start_triage_by_name_spec.rb spec/commands/start_triage_neighborhood_spec.rb spec/commands/citizens spec/commands/conversation_advance_spec.rb spec/requests/citizen_api spec/invariants/territory_invariants_spec.rb spec/integration`
Expected: PASS.

Run também a busca de quem ainda posta conversa sem nome:
`grep -rn '"/citizen/conversations"' apps/api/.claude/mod15/spec | grep -v protocol_name`
Expected: só as linhas de termo desatualizado/ausente (409 antes do nome) e as de `spec/requests/citizen_api/start_by_name_spec.rb` que testam a ausência.

- [ ] **Step 8: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod15 add app/commands/start_triage.rb app/commands/citizens/start_conversation.rb app/controllers/citizen_api/conversations_controller.rb spec/support/citizen_request_helpers.rb spec/commands/start_triage_by_name_spec.rb spec/commands/citizens/start_conversation_spec.rb spec/commands/citizens/start_conversation_concurrency_spec.rb spec/requests/citizen_api spec/invariants/territory_invariants_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod15 commit -m "feat: start triages by protocol name under the offer rule

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 11: Sugestões na conclusão (`Triages::Suggest`)

**Files:**
- Create: `app/commands/triages/suggest.rb`
- Modify: `app/commands/complete_triage.rb`
- Modify: `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`
- Test: `spec/commands/triages/suggest_spec.rb`

**Interfaces:**
- Consumes: `Triages::Offer.for` (Task 9), `Protocols::ConditionContext.build`, `Protocols::Urgency.urgent?`, `TriageSuggestion`.
- Produces: `Triages::Suggest.call(triage:, outcome:, on: Time.zone.today) -> Array<TriageSuggestion>` (as criadas agora). Chamado por `CompleteTriage` dentro da mesma transação, depois de `complete!`. Evento `triage.suggested` `{ triage_id, suggestion_id, protocol_name }`.

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/commands/triages/suggest_spec.rb
require "rails_helper"

# ADR 0027 (spec 2026-10-05 §5.3): na conclusão, cada suggestions[] cujo when
# é verdadeiro e cujo protocolo está available para o par vira uma sugestão
# pending (uma por protocolo por par). Urgente nunca sugere. Nunca para si.
RSpec.describe Triages::Suggest do
  before { Current.city = TEST_CITY_A }
  after { Current.reset; Rails.cache.clear }

  let(:to_deep) { [ { "protocol" => "saude-mental-aprofundada", "when" => { "gte" => ["outcome.score", 4] } } ] }
  let!(:deep) { active_protocol!("saude-mental-aprofundada", offer: { "title" => "Aprofundamento" }) }
  let(:par) { profiled_citizen!(age: 30) }

  def complete(name, answer)
    started = start_for!(par, name).payload
    Citizens::SubmitAnswer.call(conversation: started[:conversation], answer: answer, idempotency_key: SecureRandom.uuid)
    started[:triage].reload
  end

  def pending = TriageSuggestion.status_pending.where(citizen_id: par.id)

  it "pontuação alta sugere, com evento só de ids" do
    active_protocol!("saude-mental", suggestions: to_deep)
    triage = complete("saude-mental", "true")
    suggestion = pending.sole
    expect(suggestion).to have_attributes(protocol_name: "saude-mental-aprofundada", source_triage_id: triage.id)
    expect(DomainEvent.where(name: "triage.suggested").sole.payload).to eq(
      "triage_id" => triage.id, "suggestion_id" => suggestion.id, "protocol_name" => "saude-mental-aprofundada"
    )
  end

  it "when falso não sugere" do
    active_protocol!("saude-mental", suggestions: to_deep)
    complete("saude-mental", "false")
    expect(pending).to be_empty
  end

  it "resultado urgente nunca sugere" do
    active_protocol!("saude-mental", suggestions: to_deep,
                                     priority_when: [ { "when" => { "eq" => ["q1", "true"] }, "priority" => 1 } ])
    triage = complete("saude-mental", "true")
    expect(triage.priority).to eq(1)
    expect(pending).to be_empty
    expect(DomainEvent.where(name: "triage.suggested")).to be_empty
  end

  it "protocolo sugerido fora de oferta (pausado) ou inexistente não sugere" do
    TriageOffer.create!(protocol_name: "saude-mental-aprofundada", enabled: false,
                        updated_by_user: staff_with("pausa-#{SecureRandom.hex(3)}@cidade.gov.br"))
    active_protocol!("saude-mental", suggestions: to_deep + [ { "protocol" => "fantasma", "when" => { "eq" => ["q1", "true"] } } ])
    complete("saude-mental", "true")
    expect(pending).to be_empty
  end

  it "uma pendente por protocolo: a segunda conclusão não duplica nem falha" do
    active_protocol!("saude-mental", suggestions: to_deep + to_deep)
    complete("saude-mental", "true")
    complete("saude-mental", "true")
    expect(pending.count).to eq(1)
  end

  it "sugestão para o próprio protocolo gravada por fora é ignorada" do
    active_protocol!("saude-mental", suggestions: [ { "protocol" => "saude-mental", "when" => { "eq" => ["q1", "true"] } } ])
    complete("saude-mental", "true")
    expect(pending).to be_empty
  end

  it "conversa sem cidadão (WhatsApp) não sugere" do
    definition = active_protocol!("saude-mental", suggestions: to_deep)
    conversation = Conversation.create!(phone: "+5541911112222", state: :consented)
    triage = Triage.create!(conversation: conversation, protocol_definition: definition, protocol_name: "saude-mental",
                            status: "completed", answers: { "q1" => "true" }, completed_at: Time.current)
    outcome = Protocols::Outcome.terminal(trail: [], tier: "media", priority: 5, score: 4)
    expect(described_class.call(triage: triage, outcome: outcome)).to eq([])
  end
end
```

No `spec/initializers/domain_events_bindings_spec.rb`, o `names` do bloco do módulo 15 passa a `%w[citizen.profile_changed triage.suggested]`.

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod15 api bundle exec rspec spec/commands/triages/suggest_spec.rb spec/initializers/domain_events_bindings_spec.rb`
Expected: FAIL (nenhuma sugestão criada; evento não declarado).

- [ ] **Step 3: Implemente**

```ruby
# app/commands/triages/suggest.rb
# Sugestões na conclusão (ADR 0027; spec 2026-10-05 §5.3). Roda dentro da
# transação do CompleteTriage, depois de complete!. Nada se o resultado é
# urgente (Protocols::Urgency) ou se a conversa não tem cidadão (WhatsApp).
# Cada suggestions[] da versão EXATA da triagem cujo `when` é verdadeiro
# (perfil + respostas + resultado) e cujo protocolo está `available` para o par
# vira uma linha pending. Já havendo pendente daquele protocolo para o par, o
# índice único parcial recusa e a sugestão é ignorada (savepoint: a transação
# da conclusão segue). Não trava o cidadão (desvio 5 do plano).
module Triages
  module Suggest
    module_function

    def call(triage:, outcome:, on: Time.zone.today)
      return [] if Protocols::Urgency.urgent?(outcome)

      citizen = triage.conversation.citizen
      rules = triage.protocol_definition.definition["suggestions"]
      return [] unless citizen && rules.is_a?(Array) && rules.any?

      context = Protocols::ConditionContext.build(
        answers: triage.answers, profile: citizen.profile_context(on: on),
        outcome: { tier: outcome.tier, score: outcome.score, priority: outcome.priority }
      )
      available = Offer.for(citizen: citizen, on: on).select(&:available?).map(&:protocol_name)
      rules.filter_map do |rule|
        next unless rule.is_a?(Hash)

        name = rule["protocol"].to_s
        next if name == triage.protocol_name || !available.include?(name)
        next unless Protocols::Condition.eval(rule["when"], context)

        create_pending(citizen, triage, name)
      end
    end

    def create_pending(citizen, triage, name)
      suggestion = ApplicationRecord.transaction(requires_new: true) do
        TriageSuggestion.create!(citizen: citizen, source_triage: triage, protocol_name: name)
      end
      DomainEvents.publish("triage.suggested", triage_id: triage.id, suggestion_id: suggestion.id, protocol_name: name)
      suggestion
    rescue ActiveRecord::RecordNotUnique
      nil
    end
  end
end
```

Em `app/commands/complete_triage.rb`, dentro do `if outcome.terminal?`, depois da linha do `triage.urgent`, acrescente:

```ruby
          Triages::Suggest.call(triage: @triage, outcome: outcome) # ADR 0027: nunca em urgente
```

Em `config/initializers/domain_events.rb`, no bloco do módulo 15, depois de `citizen.profile_changed`:

```ruby
  DomainEvents.bind "triage.suggested", to: []
```

- [ ] **Step 4: Rode**

Run: `docker compose exec -T -w /rails/.claude/mod15 api bundle exec rspec spec/commands/triages spec/commands/complete_triage_spec.rb spec/commands/complete_triage_payload_spec.rb spec/commands/citizens/submit_answer_spec.rb spec/commands/conversation_advance_spec.rb spec/initializers/domain_events_bindings_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod15 add app/commands/triages/suggest.rb app/commands/complete_triage.rb config/initializers/domain_events.rb spec/commands/triages/suggest_spec.rb spec/initializers/domain_events_bindings_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod15 commit -m "feat: create pending suggestions when a non-urgent triage completes

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 12: Catálogo do cidadão (`GET /citizen/people/:id/catalog`)

**Files:**
- Create: `app/services/triages/catalog.rb`
- Modify: `app/controllers/citizen_api/people_controller.rb`, `config/routes.rb`
- Test: `spec/services/triages/catalog_spec.rb`, `spec/requests/citizen_api/catalog_spec.rb`

**Interfaces:**
- Consumes: `Triages::Offer.for`, `Triages::Offer.title_for` (Task 9); `TriageOfferDailyCount.increment!` (Task 4); `Territory::ReferenceUnits.for`/`.as_json_list` (módulo 11).
- Produces:
  - `Triages::Catalog.for(citizen:, on: Time.zone.today) -> Hash` no formato do contrato §3.4 (`in_progress`, `suggested`, `available`, `recent`, `reference_units`; datas ISO; cada `suggested[]` com `suggestion_id`, `source_triage_id`, `source_title`, `suggested_on`). Efeitos: expira (`expired`, `resolved_at`) toda pendente do par cujo protocolo não está `available`; soma 1 em `triage_offer_daily_counts` para cada protocolo em `suggested` ou `available`.
  - `Triages::Catalog.in_progress_triage(citizen) -> Triage | nil`.
  - Rota `GET /citizen/people/:id/catalog` → `people#catalog` (404 `not_found`, 409 `profile_required`).

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/services/triages/catalog_spec.rb
require "rails_helper"

# Contratos §3.4 (ADR 0027; spec §5.3): suggested OU available, nunca os dois;
# expiração preguiçosa; contagem agregada de "oferecida".
RSpec.describe Triages::Catalog do
  before { Current.city = TEST_CITY_A }
  after { Current.reset; Rails.cache.clear }

  let(:admin) { staff_with("catalogo-#{SecureRandom.hex(3)}@cidade.gov.br") }
  let!(:respiratoria) { create_default_protocol! }
  let!(:mental) { active_protocol!("saude-mental", offer: { "title" => "Saúde mental" }) }
  let!(:deep) { active_protocol!("saude-mental-aprofundada", offer: { "title" => "Aprofundamento", "summary" => "Mais perguntas." }) }
  let!(:idoso) do
    active_protocol!("saude-do-idoso", offer: { "title" => "Saúde do idoso", "eligibility" => { "gte" => ["profile.age", 60] },
                                                "retake_after_days" => 365 })
  end
  let(:avo) { profiled_citizen!(age: 62) }

  before do
    TriageOffer.create!(protocol_name: "saude-do-idoso", updated_by_user: admin, position: 1)
    TriageOffer.create!(protocol_name: "saude-mental-aprofundada", updated_by_user: admin, position: 2)
  end

  def suggest!(citizen, name, from: "saude-mental")
    TriageSuggestion.create!(citizen: citizen, source_triage: completed_triage!(citizen, from, at: 2.days.ago), protocol_name: name)
  end

  it "monta as três seções na ordem do catálogo, sugestão com a origem" do
    suggestion = suggest!(avo, "saude-mental-aprofundada")
    catalog = described_class.for(citizen: avo)
    expect(catalog[:suggested]).to eq([ {
      protocol_name: "saude-mental-aprofundada", title: "Aprofundamento", summary: "Mais perguntas.",
      suggestion_id: suggestion.id, source_triage_id: suggestion.source_triage_id, source_title: "Saúde mental",
      suggested_on: suggestion.created_at.in_time_zone.to_date.iso8601
    } ])
    expect(catalog[:available].map { |i| i[:protocol_name] }).to eq(%w[saude-do-idoso saude-mental triage-respiratoria])
    expect(catalog[:available].last).to eq(protocol_name: "triage-respiratoria", title: "triage-respiratoria", summary: nil)
    expect(catalog[:recent]).to eq([])
    expect(catalog[:in_progress]).to be_nil
    expect(catalog[:reference_units]).to eq([])
  end

  it "recent traz a última conclusão e a próxima data" do
    completed_triage!(avo, "saude-do-idoso", at: 10.days.ago)
    recent = described_class.for(citizen: avo)[:recent].sole
    last_on = 10.days.ago.to_date
    expect(recent).to eq(protocol_name: "saude-do-idoso", title: "Saúde do idoso", summary: nil,
                         last_completed_on: last_on.iso8601, next_available_on: (last_on + 365).iso8601)
  end

  it "pausa expira a sugestão na leitura" do
    suggestion = suggest!(avo, "saude-mental-aprofundada")
    TriageOffer.find_by!(protocol_name: "saude-mental-aprofundada").update!(enabled: false)
    catalog = described_class.for(citizen: avo)
    expect(catalog[:suggested]).to eq([])
    expect(suggestion.reload).to have_attributes(status: "expired", resolved_at: be_present)
  end

  it "triagem em andamento sai das listas e vem em in_progress" do
    started = start_for!(avo, "saude-mental").payload
    catalog = described_class.for(citizen: avo)
    expect(catalog[:in_progress]).to eq(conversation_id: started[:conversation].id, protocol_name: "saude-mental",
                                        title: "Saúde mental")
    expect(catalog[:available].map { |i| i[:protocol_name] }).not_to include("saude-mental")
  end

  it "conta oferecida por protocolo mostrado, a cada leitura" do
    suggest!(avo, "saude-mental-aprofundada")
    2.times { described_class.for(citizen: avo) }
    expect(TriageOfferDailyCount.where(day: Time.zone.today).order(:protocol_name).pluck(:protocol_name, :offered))
      .to eq([ [ "saude-do-idoso", 2 ], [ "saude-mental", 2 ], [ "saude-mental-aprofundada", 2 ], [ "triage-respiratoria", 2 ] ])
  end

  it "unidades de referência do bairro ATUAL do par" do
    centro = Neighborhood.create!(name: "Centro", source: "seed")
    ubs = create_unit("UBS Centro")
    NeighborhoodCoverage.create!(neighborhood: centro, health_unit: ubs)
    avo.update!(neighborhood: centro)
    expect(described_class.for(citizen: avo)[:reference_units]).to eq(Territory::ReferenceUnits.as_json_list([ ubs ]))
  end
end
```

```ruby
# spec/requests/citizen_api/catalog_spec.rb
require "rails_helper"

# Contratos §3.4 e Review Focus 2: pares que dividem celular ou CPF não se
# enxergam; sem perfil, 409.
RSpec.describe "Catálogo do cidadão", type: :request do
  before do
    create_default_protocol!
    active_protocol!("saude-do-idoso", offer: { "title" => "Saúde do idoso", "eligibility" => { "gte" => ["profile.age", 60] } })
    active_protocol!("saude-mental", offer: { "title" => "Saúde mental" })
    # Protocolo com elegibilidade só entra em oferta com linha no catálogo.
    TriageOffer.create!(protocol_name: "saude-do-idoso", updated_by_user: staff_with("cat-#{SecureRandom.hex(3)}@cidade.gov.br"))
    sign_in_citizen("+5541998765432")
  end
  after { Rails.cache.clear }

  def body = JSON.parse(response.body)
  def status_and_error = [ response.status, body["error"] ]
  def catalog_of(citizen) = get("/citizen/people/#{citizen.id}/catalog")
  def names(section) = body[section].map { |i| i["protocol_name"] }

  it "dois pares no mesmo celular: avó e neto com catálogos diferentes" do
    avo = profiled_citizen!(age: 62)
    neto = profiled_citizen!(age: 8, sex: "male", cpf: CampaignHistory.cpf_for("neto"))
    catalog_of(avo)
    expect(names("available")).to eq(%w[saude-do-idoso saude-mental triage-respiratoria])
    expect(body).to include("in_progress" => nil, "suggested" => [], "recent" => [], "reference_units" => [])
    catalog_of(neto)
    expect(names("available")).to eq(%w[saude-mental triage-respiratoria])
  end

  it "mesmo CPF em dois celulares: perfil e sugestão não cruzam" do
    mine = profiled_citizen!(age: 30, cpf: "52998224725")
    other = profiled_citizen!(age: 62, cpf: "52998224725", phone: "+5541900000000")
    TriageSuggestion.create!(citizen: other, source_triage: completed_triage!(other, "saude-mental"), protocol_name: "saude-do-idoso")
    catalog_of(mine)
    expect(body["suggested"]).to eq([])
    expect(names("available")).not_to include("saude-do-idoso")
    catalog_of(other)
    expect(status_and_error).to eq([ 404, "not_found" ])
  end

  it "sem perfil: 409 profile_required" do
    citizen = Citizen.create!(cpf: "52998224725", phone: "+5541998765432")
    catalog_of(citizen)
    expect(status_and_error).to eq([ 409, "profile_required" ])
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod15 api bundle exec rspec spec/services/triages/catalog_spec.rb spec/requests/citizen_api/catalog_spec.rb`
Expected: FAIL (`uninitialized constant Triages::Catalog`; rota inexistente).

- [ ] **Step 3: Implemente**

```ruby
# app/services/triages/catalog.rb
# Catálogo do cidadão (ADR 0027; spec 2026-10-05 §5.3, §6.1; contratos §3.4).
# Um protocolo aparece em `suggested` OU em `available`, nunca nos dois; o que
# está em andamento só aparece em `in_progress` (desvio 6 do plano). A leitura
# EXPIRA toda sugestão pendente do par cujo protocolo deixou de estar
# `available` (expiração preguiçosa) e soma 1 na contagem diária de cada
# protocolo mostrado em oferta (desvio 1). `reference_units` vem do bairro
# ATUAL do par (módulo 11), [] sem bairro.
module Triages
  module Catalog
    module_function

    def for(citizen:, on: Time.zone.today)
      items = Offer.for(citizen: citizen, on: on)
      available = items.select(&:available?)
      in_progress = in_progress_triage(citizen)
      expire_stale!(citizen, available.map(&:protocol_name))

      pending = TriageSuggestion.status_pending.where(citizen_id: citizen.id)
                                .includes(source_triage: :protocol_definition).index_by(&:protocol_name)
      shown = available.reject { |item| item.protocol_name == in_progress&.protocol_name }
      suggested, offered = shown.partition { |item| pending.key?(item.protocol_name) }
      TriageOfferDailyCount.increment!(shown.map(&:protocol_name), day: on)

      {
        in_progress: in_progress && in_progress_json(in_progress),
        suggested: suggested.map { |item| suggested_json(item, pending.fetch(item.protocol_name)) },
        available: offered.map { |item| item_json(item) },
        recent: items.select(&:recent?).map { |item| recent_json(item) },
        reference_units: Territory::ReferenceUnits.as_json_list(Territory::ReferenceUnits.for(citizen.neighborhood_id))
      }
    end

    def in_progress_triage(citizen)
      Triage.status_in_progress.joins(:conversation)
            .where(conversations: { channel: "web", citizen_id: citizen.id, state: Conversation::ACTIVE_STATES })
            .includes(:protocol_definition).order(created_at: :desc).first
    end

    def expire_stale!(citizen, available_names)
      TriageSuggestion.status_pending.where(citizen_id: citizen.id).where.not(protocol_name: available_names)
                      .update_all(status: "expired", resolved_at: Time.current)
    end

    def item_json(item) = { protocol_name: item.protocol_name, title: item.title, summary: item.summary }

    def recent_json(item)
      item_json(item).merge(last_completed_on: item.last_completed_on.iso8601,
                            next_available_on: item.next_available_on.iso8601)
    end

    def suggested_json(item, suggestion)
      source = suggestion.source_triage
      item_json(item).merge(
        suggestion_id: suggestion.id, source_triage_id: suggestion.source_triage_id,
        source_title: Offer.title_for(source.protocol_definition.definition, source.protocol_name),
        suggested_on: suggestion.created_at.in_time_zone.to_date.iso8601
      )
    end

    def in_progress_json(triage)
      { conversation_id: triage.conversation_id, protocol_name: triage.protocol_name,
        title: Offer.title_for(triage.protocol_definition.definition, triage.protocol_name) }
    end
  end
end
```

Em `app/controllers/citizen_api/people_controller.rb`, depois de `def profile ... end`, acrescente (e a linha `# GET  /citizen/people/:id/catalog — catálogo do par (contratos §3.4); sem perfil, 409.` no comentário do topo):

```ruby
    def catalog
      citizen = current_citizen_session.citizens.find_by(id: params[:id])
      return render_error("not_found", :not_found) unless citizen
      return render_error("profile_required", :conflict) unless citizen.profile?

      render json: Triages::Catalog.for(citizen: citizen)
    end
```

Em `config/routes.rb`, depois de `post "people/:id/profile", ...`:

```ruby
    get  "people/:id/catalog",         to: "people#catalog"
```

- [ ] **Step 4: Rode**

Run: `docker compose exec -T -w /rails/.claude/mod15 api bundle exec rspec spec/services/triages spec/requests/citizen_api`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod15 add app/services/triages/catalog.rb app/controllers/citizen_api/people_controller.rb config/routes.rb spec/services/triages/catalog_spec.rb spec/requests/citizen_api/catalog_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod15 commit -m "feat: serve the citizen triage catalog with lazy suggestion expiry

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 13: Sugestões no resultado (`GET /citizen/triages/:id`)

**Files:**
- Modify: `app/services/triages/catalog.rb` (acrescenta `suggestions_for`)
- Modify: `app/controllers/citizen_api/triages_controller.rb`
- Test: `spec/requests/citizen_api/triage_suggestions_spec.rb`

**Interfaces:**
- Consumes: `Triages::Offer.for`, `TriageSuggestion`.
- Produces: `Triages::Catalog.suggestions_for(triage, on: Time.zone.today) -> Array<Hash>` (`{ suggestion_id, protocol_name, title, summary }` das pendentes nascidas da triagem e ainda `available` para o par). `GET /citizen/triages/:id` ganha `suggestions` (`[]` para triagem de outro par).

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/requests/citizen_api/triage_suggestions_spec.rb
require "rails_helper"

# Contratos §3.6 (ADR 0027): "Recomendamos também" — só as pendentes nascidas
# desta triagem e ainda em oferta; [] em urgente; nunca para outro par.
RSpec.describe "Sugestões no resultado", type: :request do
  before do
    create_default_protocol!
    active_protocol!("saude-mental-aprofundada", offer: { "title" => "Aprofundamento", "summary" => "Mais perguntas." })
    ConsentTerm.create!(version: "1", body: "Termo de teste", published_at: Time.current)
    sign_in_citizen("+5541998765432")
  end
  after { Rails.cache.clear }

  def body = JSON.parse(response.body)
  let(:to_deep) { [ { "protocol" => "saude-mental-aprofundada", "when" => { "gte" => ["outcome.score", 4] } } ] }
  let(:admin) { staff_with("catalogo-#{SecureRandom.hex(3)}@cidade.gov.br") }

  def finish_mental(answer)
    started = start_citizen_triage(protocol_name: "saude-mental")
    json_post "/citizen/conversations/#{started['conversation_id']}/answers", answer: answer, idempotency_key: SecureRandom.uuid
    body["triage_id"]
  end

  it "traz a sugestão nascida desta triagem" do
    active_protocol!("saude-mental", suggestions: to_deep)
    triage_id = finish_mental("true")
    get "/citizen/triages/#{triage_id}"
    expect(body["suggestions"]).to eq([ {
      "suggestion_id" => TriageSuggestion.sole.id, "protocol_name" => "saude-mental-aprofundada",
      "title" => "Aprofundamento", "summary" => "Mais perguntas."
    } ])
  end

  it "protocolo pausado some do resultado" do
    active_protocol!("saude-mental", suggestions: to_deep)
    triage_id = finish_mental("true")
    TriageOffer.create!(protocol_name: "saude-mental-aprofundada", enabled: false, updated_by_user: admin)
    get "/citizen/triages/#{triage_id}"
    expect(body["suggestions"]).to eq([])
  end

  it "urgente: []" do
    active_protocol!("saude-mental", suggestions: to_deep,
                                     priority_when: [ { "when" => { "eq" => ["q1", "true"] }, "priority" => 1 } ])
    triage_id = finish_mental("true")
    get "/citizen/triages/#{triage_id}"
    expect(body["suggestions"]).to eq([])
  end

  it "triagem de outro par do CPF verificado: suggestions []" do
    active_protocol!("saude-mental", suggestions: to_deep)
    mine = profiled_citizen!(age: 30, cpf: "52998224725")
    mine.update!(verification_level: "verified")
    other = profiled_citizen!(age: 30, cpf: "52998224725", phone: "+5541900000000")
    source = completed_triage!(other, "saude-mental")
    TriageSuggestion.create!(citizen: other, source_triage: source, protocol_name: "saude-mental-aprofundada")
    get "/citizen/triages/#{source.id}"
    expect(response).to have_http_status(:ok)
    expect(body["suggestions"]).to eq([])
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod15 api bundle exec rspec spec/requests/citizen_api/triage_suggestions_spec.rb`
Expected: FAIL (`suggestions` ausente: `nil`).

- [ ] **Step 3: Implemente**

Em `app/services/triages/catalog.rb`, antes de `def item_json`, acrescente:

```ruby
    # "Recomendamos também" (contratos §3.6): as pendentes nascidas DESTA
    # triagem e ainda available para o par dela. Urgente nunca gerou nenhuma.
    def suggestions_for(triage, on: Time.zone.today)
      citizen = triage.conversation.citizen
      return [] unless citizen

      pending = TriageSuggestion.status_pending.where(source_triage_id: triage.id).order(:created_at).to_a
      return [] if pending.empty?

      available = Offer.for(citizen: citizen, on: on).select(&:available?).index_by(&:protocol_name)
      pending.filter_map do |suggestion|
        item = available[suggestion.protocol_name]
        item && { suggestion_id: suggestion.id, protocol_name: item.protocol_name, title: item.title, summary: item.summary }
      end
    end
```

Em `app/controllers/citizen_api/triages_controller.rb`, `show` passa a:

```ruby
    def show
      triage = visible_triages.find_by(id: params[:id])
      return render_error("not_found", :not_found) unless triage

      # ADR 0023: do bairro COPIADO na triagem (o mesmo do atendimento), não do
      # atual do cidadão. ADR 0027: sugestões só para o próprio par.
      units = Territory::ReferenceUnits.for(triage.neighborhood_id)
      suggestions = own_triage?(triage) ? Triages::Catalog.suggestions_for(triage) : []
      render json: summary(triage).merge(reference_units: Territory::ReferenceUnits.as_json_list(units),
                                         suggestions: suggestions)
    end
```

e o comentário do topo ganha `#   GET  /citizen/triages/:id   (com reference_units, ADR 0023, e suggestions, ADR 0027)`.

- [ ] **Step 4: Rode**

Run: `docker compose exec -T -w /rails/.claude/mod15 api bundle exec rspec spec/requests/citizen_api`
Expected: PASS (se algum exemplo antigo de `GET /citizen/triages/:id` comparar o corpo inteiro com `eq`, acrescente `"suggestions" => []` ao esperado).

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod15 add app/services/triages/catalog.rb app/controllers/citizen_api/triages_controller.rb spec/requests/citizen_api/triage_suggestions_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod15 commit -m "feat: show pending suggestions on the citizen triage result

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Fatia 4 — Catálogo da cidade, contadores e simulador (F-15.4, F-15.7, F-15.8)

### Task 14: `Triages::SetOffer` — a linha do catálogo da cidade

**Files:**
- Create: `app/commands/triages/set_offer.rb`
- Modify: `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`
- Test: `spec/commands/triages/set_offer_spec.rb`

**Interfaces:**
- Consumes: `Protocols::Validation::Condition.errors`, `Protocols::Validation::Offer::RESTRICTION` (Tasks 1–2); `TriageOffer`.
- Produces: `Triages::SetOffer.call(protocol_name:, attributes:, by:) -> Result` — `attributes` é o corpo JSON cru (chaves string `enabled`, `position`, `restriction`, `available_from`, `available_until`); reasons `:unknown_protocol`, `:invalid_enabled`, `:invalid_position`, `:invalid_period`, `:invalid_restriction`; ok `{ offer: TriageOffer }`. Grava a linha inteira (cria se não existe), `updated_by_user`, e publica `triage_offer.changed` `{ protocol_name, user_id }` só quando muda. `Triages::SetOffer::MAX_RESTRICTION_BYTES == 4096`.

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/commands/triages/set_offer_spec.rb
require "rails_helper"

# ADR 0027 (spec 2026-10-05 §3.2, §6.2; contratos §4.2): a cidade pausa,
# ordena, restringe (só profile.age, profile.sex, citizen.neighborhood_id) e
# limita o período; nunca amplia. Evento só com nome e usuário.
RSpec.describe Triages::SetOffer do
  before { Current.city = TEST_CITY_A }
  after { Current.reset; Rails.cache.clear }

  let(:admin) { staff_with("catalogo-#{SecureRandom.hex(3)}@cidade.gov.br", "municipal_admin") }
  let(:centro) { "0b6f6c1e-9f1a-4d8b-9a4c-1f2e3d4c5b6a" }
  let(:valid) do
    { "enabled" => true, "position" => 2, "restriction" => { "in" => ["citizen.neighborhood_id", [centro]] },
      "available_from" => nil, "available_until" => "2026-12-31" }
  end

  before { active_protocol!("saude-do-idoso") }

  def set(attrs = {}, name: "saude-do-idoso") = described_class.call(protocol_name: name, attributes: valid.merge(attrs), by: admin)
  def events = DomainEvent.where(name: "triage_offer.changed").map(&:payload)

  it "cria a linha, depois atualiza; evento só quando muda" do
    expect(set).to be_ok
    expect(TriageOffer.sole).to have_attributes(enabled: true, position: 2, available_until: Date.new(2026, 12, 31),
                                                restriction: valid["restriction"], updated_by_user_id: admin.id)
    set
    set("enabled" => false)
    expect(TriageOffer.sole.enabled).to be(false)
    expect(events).to eq([ { "protocol_name" => "saude-do-idoso", "user_id" => admin.id } ] * 2)
  end

  it "protocolo sem nenhuma versão: :unknown_protocol" do
    expect(set(name: "fantasma").reason).to eq(:unknown_protocol)
    expect(TriageOffer.count).to eq(0)
  end

  it "versão não ativa também configura (a cidade prepara antes de ativar)" do
    ProtocolDefinition.create!(name: "rascunho", version: 1, status: "draft", definition: catalog_definition("rascunho"))
    expect(set(name: "rascunho")).to be_ok
  end

  it "recusa enabled, posição e período inválidos" do
    [ nil, "sim", 1 ].each { |v| expect(set("enabled" => v).reason).to eq(:invalid_enabled), v.inspect }
    [ nil, 0, -1, 10_001, 1.5, "2" ].each { |v| expect(set("position" => v).reason).to eq(:invalid_position), v.inspect }
    [
      { "available_from" => "2026-12-31", "available_until" => "2026-01-01" },
      { "available_from" => "31/12/2026" }, { "available_until" => "2026-02-30" }, { "available_from" => 20261231 }
    ].each { |v| expect(set(v).reason).to eq(:invalid_period), v.inspect }
    expect(TriageOffer.count).to eq(0)
  end

  it "recusa restrição inválida: tabela (Review Focus 5)" do
    [
      { "xyz" => [ "profile.age", 1 ] },
      { "gte" => [ "outcome.score", 1 ] },
      { "eq" => [ "q1", "true" ] },
      { "eq" => [ "profile.sex", "outro" ] },
      { "in" => [ "citizen.neighborhood_id", [ "nao-uuid" ] ] },
      { "any" => [] },
      "lixo", [ 1 ], 42,
      { "in" => [ "citizen.neighborhood_id", Array.new(120) { SecureRandom.uuid } ] } # > 4096 bytes
    ].each do |restriction|
      expect(set("restriction" => restriction).reason).to eq(:invalid_restriction), restriction.inspect[0, 80]
    end
    expect(TriageOffer.count).to eq(0)
  end

  it "restrição nula = sem restrição" do
    expect(set("restriction" => nil)).to be_ok
    expect(TriageOffer.sole.restriction).to be_nil
  end
end
```

No `spec/initializers/domain_events_bindings_spec.rb`, o `names` do bloco do módulo 15 passa a `%w[citizen.profile_changed triage.suggested triage_offer.changed]`.

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod15 api bundle exec rspec spec/commands/triages/set_offer_spec.rb spec/initializers/domain_events_bindings_spec.rb`
Expected: FAIL (`uninitialized constant Triages::SetOffer`).

- [ ] **Step 3: Implemente**

```ruby
# app/commands/triages/set_offer.rb
# A linha do catálogo da cidade para um protocolo (ADR 0027; spec 2026-10-05
# §3.2, §6.2; contratos §4.2). O corpo é a linha INTEIRA. A restrição só usa
# profile.age, profile.sex e citizen.neighborhood_id e soma com E à
# elegibilidade assinada (Triages::Offer): a cidade restringe, nunca amplia.
# Quem pode e o step-up ficam no controller. Evento só quando muda.
# Reasons: :unknown_protocol, :invalid_enabled, :invalid_position,
# :invalid_period, :invalid_restriction.
module Triages
  module SetOffer
    POSITIONS = (1..10_000)
    MAX_RESTRICTION_BYTES = 4096
    DATE = /\A\d{4}-\d{2}-\d{2}\z/

    module_function

    def call(protocol_name:, attributes:, by:)
      name = protocol_name.to_s
      return Result.fail(:unknown_protocol) unless ProtocolDefinition.exists?(name: name)

      changes = validate(attributes)
      return changes if changes.is_a?(Result)

      save(name, changes, by)
    end

    def validate(attributes)
      enabled = attributes["enabled"]
      return Result.fail(:invalid_enabled) unless [ true, false ].include?(enabled)

      position = attributes["position"]
      return Result.fail(:invalid_position) unless position.is_a?(Integer) && POSITIONS.cover?(position)

      from = date(attributes["available_from"])
      until_on = date(attributes["available_until"])
      return Result.fail(:invalid_period) if [ from, until_on ].include?(:invalid)
      return Result.fail(:invalid_period) if from && until_on && until_on < from

      restriction = attributes["restriction"]
      return Result.fail(:invalid_restriction) unless restriction_valid?(restriction)

      { enabled: enabled, position: position, restriction: restriction, available_from: from, available_until: until_on }
    end

    def date(raw)
      return nil if raw.nil?
      return :invalid unless raw.is_a?(String) && raw.match?(DATE)

      Date.iso8601(raw)
    rescue Date::Error
      :invalid
    end

    def restriction_valid?(restriction)
      return true if restriction.nil?
      return false unless restriction.is_a?(Hash) && restriction.to_json.bytesize <= MAX_RESTRICTION_BYTES

      Protocols::Validation::Condition.errors(restriction, {}, variables: Protocols::Validation::Offer::RESTRICTION).empty?
    end

    def save(name, changes, by, attempt: 1)
      ApplicationRecord.transaction do
        row = TriageOffer.lock.find_by(protocol_name: name) || TriageOffer.new(protocol_name: name)
        row.assign_attributes(changes)
        if row.new_record? || row.changed?
          row.updated_by_user = by
          row.save!
          DomainEvents.publish("triage_offer.changed", protocol_name: name, user_id: by.id)
        end
      end
      Result.ok(offer: TriageOffer.find_by!(protocol_name: name))
    rescue ActiveRecord::RecordNotUnique
      # Duas criações ao mesmo tempo: a segunda acha a linha da primeira.
      raise if attempt > 1

      save(name, changes, by, attempt: attempt + 1)
    end
  end
end
```

Em `config/initializers/domain_events.rb`, no bloco do módulo 15:

```ruby
  DomainEvents.bind "triage_offer.changed", to: []
```

- [ ] **Step 4: Rode**

Run: `docker compose exec -T -w /rails/.claude/mod15 api bundle exec rspec spec/commands/triages spec/initializers/domain_events_bindings_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod15 add app/commands/triages/set_offer.rb config/initializers/domain_events.rb spec/commands/triages/set_offer_spec.rb spec/initializers/domain_events_bindings_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod15 commit -m "feat: let the city pause, order, restrict and schedule a triage offer

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 15: Contadores por protocolo com supressão

**Files:**
- Create: `app/services/triages/counters.rb`
- Test: `spec/services/triages/counters_spec.rb`

**Interfaces:**
- Consumes: `TriageOfferDailyCount`, `TriageSuggestion`, `Triage.counted_completed`, `Admin::SmallCount.small?`.
- Produces: `Triages::Counters::WINDOW_DAYS == 30`; `Triages::Counters.for(names, on: Time.zone.today) -> { name => { offered:, started:, completed:, from_suggestion: } }` — cada valor `Integer` (0 inclusive) ou `nil` quando entre 1 e 4.

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/services/triages/counters_spec.rb
require "rails_helper"

# Spec 2026-10-05 §7 e contratos §4.1 (desvio 10 do plano): últimos 30 dias
# locais, agregados, 1 a 4 viram nil (ADR 0025); 0 continua 0.
RSpec.describe Triages::Counters do
  before { Current.city = TEST_CITY_A }
  after { Current.reset; Rails.cache.clear }

  let!(:mental) { active_protocol!("saude-mental") }
  let!(:deep) { active_protocol!("saude-mental-aprofundada") }

  def pair(index) = profiled_citizen!(age: 30, phone: format("+55419%08d", 40_000_000 + index))

  it "conta e suprime 1 a 4, na janela de 30 dias" do
    today = Time.zone.today
    TriageOfferDailyCount.create!(day: today, protocol_name: "saude-mental", offered: 7)
    TriageOfferDailyCount.create!(day: today - 29, protocol_name: "saude-mental", offered: 3)
    TriageOfferDailyCount.create!(day: today - 30, protocol_name: "saude-mental", offered: 100) # fora
    5.times { |i| completed_triage!(pair(i), "saude-mental") }
    completed_triage!(pair(10), "saude-mental", at: 31.days.ago) # fora
    6.times do |i|
      citizen = pair(20 + i)
      source = completed_triage!(citizen, "saude-mental")
      taken = completed_triage!(citizen, "saude-mental-aprofundada")
      TriageSuggestion.create!(citizen: citizen, source_triage: source, protocol_name: "saude-mental-aprofundada",
                               status: "taken", taken_triage_id: taken.id, resolved_at: Time.current)
    end

    counters = described_class.for(%w[saude-mental saude-mental-aprofundada fantasma])
    expect(counters["saude-mental"]).to eq(offered: 10, started: 11, completed: 11, from_suggestion: 0)
    expect(counters["saude-mental-aprofundada"]).to eq(offered: 0, started: 6, completed: 6, from_suggestion: 6)
    expect(counters["fantasma"]).to eq(offered: 0, started: 0, completed: 0, from_suggestion: 0)

    TriageOfferDailyCount.where(protocol_name: "saude-mental").delete_all
    TriageOfferDailyCount.create!(day: today, protocol_name: "saude-mental", offered: 4)
    expect(described_class.for(%w[saude-mental])["saude-mental"][:offered]).to be_nil
  end

  it "concluída revogada não conta como concluída" do
    5.times do |i|
      triage = completed_triage!(pair(50 + i), "saude-mental")
      triage.update_columns(status: "aborted_by_revocation")
    end
    expect(described_class.for(%w[saude-mental])["saude-mental"]).to include(started: 5, completed: 0)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod15 api bundle exec rspec spec/services/triages/counters_spec.rb`
Expected: FAIL (`uninitialized constant Triages::Counters`).

- [ ] **Step 3: Implemente**

```ruby
# app/services/triages/counters.rb
# Contadores da aba "Catálogo de triagens" (spec 2026-10-05 §7; contratos §4.1;
# desvio 10 do plano): hoje e os 29 dias anteriores, no fuso da cidade.
# Agregados; cada valor de 1 a 4 vira nil (ADR 0025, Admin::SmallCount).
#   offered         — soma de triage_offer_daily_counts
#   started         — triagens criadas com o protocol_name (todos os canais)
#   completed       — Triage.counted_completed (exclui revogada)
#   from_suggestion — sugestões taken resolvidas na janela
module Triages
  module Counters
    WINDOW_DAYS = 30

    module_function

    def for(names, on: Time.zone.today)
      first_day = on - (WINDOW_DAYS - 1)
      since = first_day.in_time_zone.beginning_of_day
      offered = TriageOfferDailyCount.where(protocol_name: names, day: first_day..on).group(:protocol_name).sum(:offered)
      started = Triage.where(protocol_name: names, created_at: since..).group(:protocol_name).count
      completed = Triage.counted_completed.where(protocol_name: names, completed_at: since..).group(:protocol_name).count
      from_suggestion = TriageSuggestion.status_taken.where(protocol_name: names, resolved_at: since..)
                                        .group(:protocol_name).count
      names.to_h do |name|
        [ name, { offered: hide(offered[name]), started: hide(started[name]), completed: hide(completed[name]),
                  from_suggestion: hide(from_suggestion[name]) } ]
      end
    end

    def hide(value)
      count = value.to_i
      Admin::SmallCount.small?(count) ? nil : count
    end
  end
end
```

- [ ] **Step 4: Rode**

Run: `docker compose exec -T -w /rails/.claude/mod15 api bundle exec rspec spec/services/triages/counters_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod15 add app/services/triages/counters.rb spec/services/triages/counters_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod15 commit -m "feat: count offered, started, completed and suggested triages with suppression

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 16: Aba do dashboard — `GET /triage_catalog` e `PUT /triage_catalog/:protocol_name`

**Files:**
- Create: `app/services/triages/catalog_admin.rb`, `app/controllers/triage_catalog_controller.rb`
- Modify: `app/policies/protocol_policy.rb`, `config/routes.rb`
- Test: `spec/requests/triage_catalog_spec.rb`

**Interfaces:**
- Consumes: `Triages::SetOffer` (Task 14), `Triages::Counters` (Task 15), `Triages::Offer.title_for` (Task 9), `MfaStepUp`.
- Produces:
  - `Triages::CatalogAdmin.index -> Array<Hash>` (um item por protocolo com versão `active`, formato contratos §4.1; configurados por `position` e título, depois os sem linha por título); `Triages::CatalogAdmin.item_for(protocol_name) -> Hash`.
  - `ProtocolPolicy#read_catalog?` (`protocol_author`, `protocol_reviewer`, `municipal_admin`), `#manage_catalog?` (`municipal_admin`), `#simulate?` (`protocol_author`, `protocol_reviewer`).
  - Rotas `GET /triage_catalog`, `PUT /triage_catalog/:protocol_name`.

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/requests/triage_catalog_spec.rb
require "rails_helper"

# Contratos §4.1–§4.2 (ADR 0027): autores, revisores e admin leem; só o
# municipal_admin muda, com step-up.
RSpec.describe "Catálogo de triagens da cidade", type: :request do
  def body = JSON.parse(response.body)
  def status_and_error = [ response.status, body["error"] ]

  let(:admin) do
    staff_with("admin-cat@cidade.gov.br", "municipal_admin").tap do |u|
      Mfa::Enroll.call(u)
      u.update!(otp_enabled: true)
    end
  end

  def sign_in_admin!(stepped_up: true)
    session = sign_in_as(admin)
    session.update!(mfa_verified_at: Time.current) if stepped_up
  end

  before do
    create_default_protocol!
    active_protocol!("saude-do-idoso", offer: { "title" => "Saúde do idoso", "eligibility" => { "gte" => ["profile.age", 60] },
                                                "retake_after_days" => 365 })
  end
  after { Rails.cache.clear }

  let(:body_for_put) do
    { enabled: true, position: 1, restriction: { "gte" => ["profile.age", 65] }, available_from: nil,
      available_until: "2026-12-31" }
  end

  it "autor, revisor e admin leem; os demais, 403" do
    %w[protocol_author protocol_reviewer municipal_admin].each do |role|
      sign_in_as(staff_with("#{role}-cat@cidade.gov.br", role))
      get "/triage_catalog"
      expect(response).to have_http_status(:ok), role
    end
    sign_in_as(staff_with("viewer-cat@cidade.gov.br", "viewer"))
    get "/triage_catalog"
    expect(status_and_error).to eq([ 403, "missing_role" ])
  end

  it "lista os ativos: sem linha = configured false; contadores suprimidos" do
    sign_in_admin!
    get "/triage_catalog"
    idoso = body["offers"].find { |o| o["protocol_name"] == "saude-do-idoso" }
    expect(idoso).to eq(
      "protocol_name" => "saude-do-idoso", "title" => "Saúde do idoso", "active_version" => 1,
      "eligibility" => { "gte" => ["profile.age", 60] }, "retake_after_days" => 365, "configured" => false,
      "enabled" => nil, "position" => nil, "restriction" => nil, "available_from" => nil, "available_until" => nil,
      "counters" => { "offered" => 0, "started" => 0, "completed" => 0, "from_suggestion" => 0 }
    )
  end

  it "admin com step-up grava e recebe o item; evento só com nome e usuário" do
    sign_in_admin!
    put "/triage_catalog/saude-do-idoso", params: body_for_put, as: :json
    expect(response).to have_http_status(:ok)
    expect(body["offer"]).to include("configured" => true, "enabled" => true, "position" => 1,
                                     "restriction" => { "gte" => ["profile.age", 65] }, "available_until" => "2026-12-31")
    expect(DomainEvent.where(name: "triage_offer.changed").sole.payload)
      .to eq("protocol_name" => "saude-do-idoso", "user_id" => admin.id)

    get "/triage_catalog"
    expect(body["offers"].map { |o| o["protocol_name"] }).to eq(%w[saude-do-idoso triage-respiratoria])
  end

  it "sem step-up: 401 mfa_required; autor não muda: 403" do
    sign_in_admin!(stepped_up: false)
    put "/triage_catalog/saude-do-idoso", params: body_for_put, as: :json
    expect(status_and_error).to eq([ 401, "mfa_required" ])
    sign_in_as(staff_with("autor-cat@cidade.gov.br", "protocol_author")).update!(mfa_verified_at: Time.current)
    put "/triage_catalog/saude-do-idoso", params: body_for_put, as: :json
    expect(status_and_error).to eq([ 403, "missing_role" ])
    expect(TriageOffer.count).to eq(0)
  end

  it "erros: 404 unknown_protocol; 422 com o motivo" do
    sign_in_admin!
    put "/triage_catalog/fantasma", params: body_for_put, as: :json
    expect(status_and_error).to eq([ 404, "unknown_protocol" ])
    put "/triage_catalog/saude-do-idoso", params: body_for_put.merge(restriction: { "gte" => ["outcome.score", 1] }), as: :json
    expect(status_and_error).to eq([ 422, "invalid_restriction" ])
    put "/triage_catalog/saude-do-idoso", params: body_for_put.merge(available_from: "2027-01-01"), as: :json
    expect(status_and_error).to eq([ 422, "invalid_period" ])
    put "/triage_catalog/saude-do-idoso", params: body_for_put.merge(position: -1), as: :json
    expect(status_and_error).to eq([ 422, "invalid_position" ])
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod15 api bundle exec rspec spec/requests/triage_catalog_spec.rb`
Expected: FAIL (rota inexistente).

- [ ] **Step 3: Implemente**

Em `app/policies/protocol_policy.rb`, antes do `end` final:

```ruby
  # Catálogo de triagens (ADR 0027; contratos §4): quem lê protocolos lê o
  # catálogo; só o municipal_admin muda (com step-up, no controller); o
  # simulador é do editor (autor e revisor).
  def read_catalog?
    author? || review? || role?(:municipal_admin)
  end

  def manage_catalog?
    role?(:municipal_admin)
  end

  def simulate?
    author? || review?
  end
```

```ruby
# app/services/triages/catalog_admin.rb
# O catálogo como a aba do dashboard o mostra (contratos §4.1): um item por
# protocolo com versão active; `configured: false` = sem linha em
# triage_offers. Elegibilidade e intervalo vêm da versão ativa (assinada).
module Triages
  module CatalogAdmin
    module_function

    def index
      actives = ProtocolDefinition.active.to_a
      names = actives.map(&:name)
      rows = TriageOffer.where(protocol_name: names).index_by(&:protocol_name)
      counters = Counters.for(names)
      items = actives.map { |definition| item(definition.name, definition, rows[definition.name], counters[definition.name]) }
      items.sort_by { |i| [ i[:configured] ? 0 : 1, i[:position] || 0, i[:title] ] }
    end

    def item_for(protocol_name)
      name = protocol_name.to_s
      item(name, ProtocolDefinition.active.find_by(name: name), TriageOffer.find_by(protocol_name: name),
           Counters.for([ name ]).fetch(name))
    end

    def item(name, definition, row, counters)
      offer = definition && definition.definition["offer"].is_a?(Hash) ? definition.definition["offer"] : {}
      {
        protocol_name: name, title: Offer.title_for(definition&.definition, name), active_version: definition&.version,
        eligibility: offer["eligibility"], retake_after_days: offer["retake_after_days"], configured: !row.nil?,
        enabled: row&.enabled, position: row&.position, restriction: row&.restriction,
        available_from: row&.available_from&.iso8601, available_until: row&.available_until&.iso8601,
        counters: counters
      }
    end
  end
end
```

```ruby
# app/controllers/triage_catalog_controller.rb
# Aba "Catálogo de triagens" (ADR 0027; spec 2026-10-05 §6.2; contratos §4.1–
# §4.2). Prefixo próprio: /protocols/:name já captura qualquer segmento.
#   GET /triage_catalog                  — autor, revisor, municipal_admin
#   PUT /triage_catalog/:protocol_name   — municipal_admin + step-up
class TriageCatalogController < ApplicationController
  include Authentication
  include MfaStepUp

  wrap_parameters false

  def index
    return forbid unless policy.read_catalog?

    render json: { offers: Triages::CatalogAdmin.index }
  end

  def update
    return forbid unless policy.manage_catalog?
    return require_step_up! unless reauthenticated_recently?

    result = Triages::SetOffer.call(protocol_name: params[:protocol_name], attributes: request.request_parameters,
                                    by: Current.user)
    if result.failure?
      status = result.reason == :unknown_protocol ? :not_found : :unprocessable_entity
      return render(json: { error: result.reason.to_s }, status: status)
    end

    render json: { offer: Triages::CatalogAdmin.item_for(params[:protocol_name]) }
  end

  private

  def policy = ProtocolPolicy.new(Current.user, nil)

  def forbid
    render json: { error: "missing_role" }, status: :forbidden
  end
end
```

Em `config/routes.rb`, depois do bloco `scope "/campaigns" do ... end`:

```ruby
  # Catálogo de triagens da cidade (ADR 0027; contratos §4). Prefixo próprio:
  # /protocols/:name já captura qualquer segmento.
  get "/triage_catalog",                to: "triage_catalog#index"
  put "/triage_catalog/:protocol_name", to: "triage_catalog#update"
```

- [ ] **Step 4: Rode**

Run: `docker compose exec -T -w /rails/.claude/mod15 api bundle exec rspec spec/requests/triage_catalog_spec.rb spec/policies spec/requests/citizen_session_on_staff_endpoints_spec.rb spec/architecture/operator_grant_access_spec.rb`
Expected: PASS. Se `operator_grant_access_spec.rb` exigir que todo controller novo declare o acesso do operador com grant, declare no `TriageCatalogController` o mesmo que `CampaignSmsSettingsController` declara (nenhuma ação para grant) e registre o controller na lista da spec, no mesmo formato das entradas vizinhas.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod15 add app/services/triages/catalog_admin.rb app/controllers/triage_catalog_controller.rb app/policies/protocol_policy.rb config/routes.rb spec/requests/triage_catalog_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod15 commit -m "feat: expose the city triage catalog to the dashboard with step-up edits

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 17: Simulador — `POST /authoring/protocols/simulate_offer`

**Files:**
- Create: `app/services/protocols/condition_text.rb`, `app/services/protocols/simulate_offer.rb`
- Modify: `app/controllers/authoring/protocols_controller.rb`, `config/routes.rb`
- Test: `spec/services/protocols/condition_text_spec.rb`, `spec/requests/authoring/simulate_offer_spec.rb`

**Interfaces:**
- Consumes: `Protocols::Condition`, `Protocols::ConditionContext`, `Protocols::Validation::Schema`, `Protocols::Validation::Offer`, `Protocols::SuggestionTargets`, `ProtocolPolicy#simulate?` (Task 16).
- Produces:
  - `Protocols::ConditionText.call(node) -> String` (frase em português só para conferência: `nil` → `"todos"`; `{"gte":["profile.age",60]}` → `"idade ≥ 60"`; inválido → `"regra inválida"`).
  - `Protocols::SimulateOffer.call(definition:, profile: {}, answers: {}, outcome: {}) -> { eligible:, eligibility_text:, suggestions: [{ protocol:, matches: }], errors: [String], warnings: [String] }`. Com `errors` não vazio (schema de `/offer`/`/suggestions` ou `Validation::Offer`): `eligible: false`, `suggestions: []` (contratos §4.3). Nunca grava.
  - Rota `POST /authoring/protocols/simulate_offer` (autor e revisor; demais 403; **sempre 200** com corpo válido).

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/services/protocols/condition_text_spec.rb
require "rails_helper"

# Contratos §4.3: frase gerada no api só para conferência (a da tela é do
# construtor do dashboard). Total: árvore inválida vira "regra inválida".
RSpec.describe Protocols::ConditionText do
  it "escreve as condições do catálogo em português" do
    {
      nil => "todos",
      { "gte" => ["profile.age", 60] } => "idade ≥ 60",
      { "all" => [ { "gte" => ["profile.age", 60] }, { "eq" => ["profile.sex", "female"] } ] } => "idade ≥ 60 e sexo = feminino",
      { "any" => [ { "all" => [ { "gte" => ["profile.age", 60] }, { "lte" => ["profile.age", 79] } ] },
                   { "eq" => ["profile.sex", "male"] } ] } => "(idade ≥ 60 e idade ≤ 79) ou sexo = masculino",
      { "not" => { "lt" => ["profile.age", 18] } } => "não (idade < 18)",
      { "in" => ["citizen.neighborhood_id", %w[a b]] } => "bairro em a, b",
      { "gte" => ["outcome.score", 15] } => "pontuação ≥ 15",
      { "eq" => ["humor", "true"] } => "resposta de humor = true",
      { "profile.sex" => "female" } => "sexo = feminino"
    }.each { |node, text| expect(described_class.call(node)).to eq(text), node.inspect }
  end

  it "árvore inválida vira 'regra inválida', sem levantar" do
    [ "x", {}, { "xyz" => 1 }.merge("abc" => 2).then { |h| { "all" => h } }, { "gte" => "x" }, { "any" => [] } ].each do |node|
      expect(described_class.call(node)).to eq("regra inválida"), node.inspect
    end
  end
end
```

```ruby
# spec/requests/authoring/simulate_offer_spec.rb
require "rails_helper"

# Contratos §4.3 (ADR 0027): o editor simula o perfil sem gravar nada; definição
# inválida responde 200 com eligible false, suggestions [] e os errors.
RSpec.describe "Simulador de oferta", type: :request do
  def body = JSON.parse(response.body)
  after { Rails.cache.clear }

  def definition(offer: { "eligibility" => { "gte" => ["profile.age", 60] } }, suggestions: nil)
    catalog_definition("saude-do-idoso", offer: offer, suggestions: suggestions)
  end

  def simulate(params) = post("/authoring/protocols/simulate_offer", params: params, as: :json)

  it "autor e revisor simulam; os demais, 403" do
    %w[protocol_author protocol_reviewer].each do |role|
      sign_in_as(staff_with("#{role}-sim@cidade.gov.br", role))
      simulate(definition: definition, profile: { age: 62, sex: "female" })
      expect(response).to have_http_status(:ok), role
    end
    sign_in_as(staff_with("admin-sim@cidade.gov.br", "municipal_admin"))
    simulate(definition: definition, profile: { age: 62, sex: "female" })
    expect(response).to have_http_status(:forbidden)
  end

  it "avalia elegibilidade e sugestões sobre perfil, respostas e resultado" do
    sign_in_as(staff_with("autor-sim@cidade.gov.br", "protocol_author"))
    create_default_protocol!
    suggestions = [ { "protocol" => StartTriage::DEFAULT_PROTOCOL_NAME, "when" => { "gte" => ["outcome.score", 4] } },
                    { "protocol" => "fantasma", "when" => { "eq" => ["q1", "true"] } } ]
    expect do
      simulate(definition: definition(suggestions: suggestions), profile: { age: 62, sex: "female", neighborhood_id: nil },
               answers: { q1: "false" }, outcome: { tier: "media", score: 4, priority: 5 })
    end.not_to change { [ ProtocolDefinition.count, TriageSuggestion.count, DomainEvent.count ] }
    expect(body).to eq(
      "eligible" => true, "eligibility_text" => "idade ≥ 60",
      "suggestions" => [ { "protocol" => StartTriage::DEFAULT_PROTOCOL_NAME, "matches" => true },
                         { "protocol" => "fantasma", "matches" => false } ],
      "errors" => [], "warnings" => [ "suggestion protocol 'fantasma' does not exist in this city" ]
    )

    simulate(definition: definition, profile: { age: 59, sex: "female" })
    expect(body).to include("eligible" => false, "suggestions" => [])
  end

  it "definição inválida: 200, eligible false, suggestions [] e os errors do gate" do
    sign_in_as(staff_with("autor2-sim@cidade.gov.br", "protocol_author"))
    bad = definition(offer: { "eligibility" => { "gte" => ["outcome.score", 1] } },
                     suggestions: [ { "protocol" => "saude-do-idoso", "when" => { "eq" => ["q1", "true"] } } ])
    simulate(definition: bad, profile: { age: 62, sex: "female" }, outcome: { score: 10 })
    expect(response).to have_http_status(:ok)
    expect(body).to include("eligible" => false, "suggestions" => [])
    expect(body["errors"]).to include("offer.eligibility: condition variable 'outcome.score' is not allowed here",
                                      "suggestions[0]: suggestion points to the protocol itself")

    simulate(definition: definition(offer: { "title" => "x" * 61 }), profile: { age: 62, sex: "female" })
    expect(response).to have_http_status(:ok)
    expect(body["errors"]).to include(a_string_starting_with("schema: /offer/title"))
    expect(body["eligible"]).to be(false)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod15 api bundle exec rspec spec/services/protocols/condition_text_spec.rb spec/requests/authoring/simulate_offer_spec.rb`
Expected: FAIL (`uninitialized constant Protocols::ConditionText`; rota inexistente).

- [ ] **Step 3: Implemente**

```ruby
# app/services/protocols/condition_text.rb
# Frase em português de uma condição (contratos §4.3), só para conferência no
# simulador: a frase da tela é do construtor do dashboard. Total.
module Protocols
  module ConditionText
    LABELS = {
      "profile.age" => "idade", "profile.sex" => "sexo", "outcome.tier" => "faixa", "outcome.score" => "pontuação",
      "outcome.priority" => "prioridade", "citizen.neighborhood_id" => "bairro"
    }.freeze
    VALUES = { "female" => "feminino", "male" => "masculino" }.freeze
    SYMBOLS = { "eq" => "=", "gt" => ">", "lt" => "<", "gte" => "≥", "lte" => "≤" }.freeze
    INVALID = "regra inválida".freeze

    module_function

    def call(node) = node.nil? ? "todos" : phrase(node, top: true)

    def phrase(node, top: false)
      return INVALID unless node.is_a?(Hash) && node.any?
      return group(node.map { |key, val| "#{label(key)} = #{value(val)}" }, " e ", top) unless Condition.operator_node?(node)

      op, operand = node.first
      case op.to_s
      when *SYMBOLS.keys
        pair?(operand) ? "#{label(operand[0])} #{SYMBOLS[op.to_s]} #{value(operand[1])}" : INVALID
      when "in"
        pair?(operand) ? "#{label(operand[0])} em #{Array(operand[1]).map { |v| value(v) }.join(', ')}" : INVALID
      when "all", "any"
        return INVALID unless operand.is_a?(Array) && operand.any?

        group(operand.map { |child| phrase(child) }, op.to_s == "all" ? " e " : " ou ", top)
      when "not" then "não (#{phrase(operand, top: true)})"
      else INVALID
      end
    end

    def pair?(operand) = operand.is_a?(Array) && operand.size == 2
    def group(parts, separator, top) = parts.size > 1 && !top ? "(#{parts.join(separator)})" : parts.join(separator)
    def label(name) = LABELS.fetch(name.to_s) { "resposta de #{name}" }
    def value(raw) = VALUES.fetch(raw.to_s, raw.to_s)
  end
end
```

(No exemplo `{ "all" => { "xyz" => 1, "abc" => 2 } }` da spec, o operando de `all` é um Hash e não um Array: "regra inválida".)

```ruby
# app/services/protocols/simulate_offer.rb
# Simulador do editor (ADR 0027; contratos §4.3). Avalia offer.eligibility e
# suggestions[].when de uma definição em edição contra um perfil, respostas e
# resultado de exemplo. Não grava nada. Definição inválida para offer/
# suggestions (schema ou gate) responde eligible false e suggestions [] com os
# errors — nunca 422 (o editor mostra o erro ao lado do construtor).
module Protocols
  module SimulateOffer
    module_function

    def call(definition:, profile: {}, answers: {}, outcome: {})
      definition = {} unless definition.is_a?(Hash)
      offer = definition["offer"].is_a?(Hash) ? definition["offer"] : {}
      eligibility = offer["eligibility"]
      errors = errors(definition)
      result = { eligible: false, eligibility_text: ConditionText.call(eligibility), suggestions: [], errors: errors,
                 warnings: SuggestionTargets.warnings(definition) }
      return result if errors.any?

      context = context(profile, answers, outcome)
      result.merge(
        eligible: eligibility.nil? || Condition.eval(eligibility, context),
        suggestions: Array(definition["suggestions"]).select { |s| s.is_a?(Hash) }
                                                    .map { |s| { protocol: s["protocol"], matches: Condition.eval(s["when"], context) } }
      )
    end

    def context(profile, answers, outcome)
      profile = ConditionContext.symbolize(profile)
      ConditionContext.build(
        answers: answers, outcome: outcome,
        profile: { age: Integer(profile[:age], exception: false), sex: profile[:sex] },
        citizen: { neighborhood_id: profile[:neighborhood_id] }
      )
    end

    def errors(definition)
      schema = Validation::Schema.call(definition).select { |e| e.start_with?("schema: /offer", "schema: /suggestions") }
      schema.any? ? schema : Validation::Offer.call(definition)
    end
  end
end
```

Em `app/controllers/authoring/protocols_controller.rb`:
- `before_action :require_author!` passa a `before_action :require_author!, except: :simulate_offer` e acrescente `before_action :require_simulator!, only: :simulate_offer`;
- nova ação, depois de `preview`:

```ruby
    # ADR 0027 (contratos §4.3): autor e revisor; sempre 200, nunca grava.
    def simulate_offer
      render json: Protocols::SimulateOffer.call(definition: definition_param, profile: hash_param(:profile),
                                                 answers: hash_param(:answers), outcome: hash_param(:outcome))
    end
```

- em `private`:

```ruby
    def hash_param(key)
      value = params[key]
      value.respond_to?(:to_unsafe_h) ? value.to_unsafe_h : {}
    end

    def require_simulator!
      head :forbidden unless ProtocolPolicy.new(Current.user, ProtocolDefinition.new).simulate?
    end
```

Em `config/routes.rb`, no `scope "/authoring/protocols"`, depois de `post "draft", ...`:

```ruby
    post "simulate_offer", to: "authoring/protocols#simulate_offer"
```

- [ ] **Step 4: Rode**

Run: `docker compose exec -T -w /rails/.claude/mod15 api bundle exec rspec spec/services/protocols spec/requests/authoring`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod15 add app/services/protocols/condition_text.rb app/services/protocols/simulate_offer.rb app/controllers/authoring/protocols_controller.rb config/routes.rb spec/services/protocols/condition_text_spec.rb spec/requests/authoring/simulate_offer_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod15 commit -m "feat: simulate protocol eligibility and suggestions for the editor

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Fatia 5 — LGPD, invariantes e semente

### Task 18: Exclusão e revogação limpam perfil e sugestões

**Files:**
- Modify: `app/commands/citizens/erase.rb`, `app/commands/revoke_consent.rb`
- Test: `spec/commands/citizens/erase_spec.rb`, `spec/commands/revoke_consent_spec.rb`

**Interfaces:**
- Produces: `Citizens::Erase.erase_pair` zera `birth_date`, `sex`, `gender_identity`, `profile_source` e apaga todas as `triage_suggestions` do par; `RevokeConsent.call` apaga, na mesma transação, as `triage_suggestions` cuja `source_triage_id` é triagem da conversa revogada (o perfil fica).

- [ ] **Step 1: Escreva as specs que falham**

Acrescente a `spec/commands/citizens/erase_spec.rb` (antes do `end` final):

```ruby
  it "ADR 0027: zera o perfil e apaga todas as sugestões do par" do
    pair.update!(birth_date: "1963-04-02", sex: "female", gender_identity: "cis_woman", profile_source: "declared")
    other_source = completed_triage!(pair, triage.protocol_name)
    TriageSuggestion.create!(citizen: pair, source_triage: triage, protocol_name: "saude-mental")
    TriageSuggestion.create!(citizen: pair, source_triage: other_source, protocol_name: "saude-do-idoso",
                             status: "expired", resolved_at: Time.current)

    expect(described_class.call(request: request, by: admin)).to be_ok
    expect(pair.reload).to have_attributes(birth_date: nil, sex: nil, gender_identity: nil, profile_source: nil)
    expect(TriageSuggestion.where(citizen_id: pair.id)).to be_empty
  end
```

Acrescente a `spec/commands/revoke_consent_spec.rb` (antes do `end` final):

```ruby
  it "ADR 0027: apaga as sugestões nascidas das triagens da conversa revogada; o perfil e as outras ficam" do
    citizen = profiled_citizen!(age: 40, phone: "+5541998761099")
    revoked = completed_web_triage_for(citizen)
    kept_source = completed_triage!(citizen, revoked.protocol_name)
    TriageSuggestion.create!(citizen: citizen, source_triage: revoked, protocol_name: "saude-mental")
    kept = TriageSuggestion.create!(citizen: citizen, source_triage: kept_source, protocol_name: "saude-do-idoso")

    expect(described_class.call(conversation: revoked.conversation, origin: "web")).to be_ok
    expect(TriageSuggestion.where(citizen_id: citizen.id)).to eq([ kept ])
    expect(citizen.reload).to have_attributes(profile_source: "declared", sex: "female")
  end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod15 api bundle exec rspec spec/commands/citizens/erase_spec.rb spec/commands/revoke_consent_spec.rb`
Expected: FAIL (perfil continua; sugestões continuam).

- [ ] **Step 3: Implemente**

Em `app/commands/citizens/erase.rb`, em `erase_pair`, depois de `CampaignRecipient.where(citizen_id: citizen.id).delete_all`:

```ruby
      # ADR 0027 (spec 2026-10-05 §5.5): as sugestões do par (o trigger deixa o
      # DELETE passar de propósito).
      TriageSuggestion.where(citizen_id: citizen.id).delete_all
```

e o `update_columns` do fim passa a:

```ruby
      citizen.update_columns(cpf: tombstone, phone: tombstone, neighborhood_id: nil, erased_at: Time.current,
                             birth_date: nil, sex: nil, gender_identity: nil, profile_source: nil,
                             updated_at: Time.current)
```

Em `app/commands/revoke_consent.rb`, dentro da transação, depois do `update!` da triagem em andamento:

```ruby
      # ADR 0027 (spec 2026-10-05 §5.5): somem as sugestões nascidas das
      # triagens desta conversa; o perfil, como o bairro, é cadastro e fica.
      TriageSuggestion.where(source_triage_id: @conversation.triages.select(:id)).delete_all
```

- [ ] **Step 4: Rode**

Run: `docker compose exec -T -w /rails/.claude/mod15 api bundle exec rspec spec/commands/citizens spec/commands/revoke_consent_spec.rb spec/jobs spec/requests/erasure_requests_spec.rb spec/requests/citizen_api/triage_flow_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod15 add app/commands/citizens/erase.rb app/commands/revoke_consent.rb spec/commands/citizens/erase_spec.rb spec/commands/revoke_consent_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod15 commit -m "fix: clear the citizen profile and suggestions on erasure and revocation

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 19: Suíte de invariantes do ADR 0027

**Files:**
- Create: `spec/invariants/triage_catalog_invariants_spec.rb`

**Interfaces:**
- Consumes: tudo das Tasks 1–18.

- [ ] **Step 1: Escreva a suíte**

```ruby
# spec/invariants/triage_catalog_invariants_spec.rb
require "rails_helper"

# Invariantes do ADR 0027 (spec 2026-10-05 §9.2). Cada bloco diz a mutação que
# ele pega; rode a mutação à mão uma vez (Step 3) antes de confiar no verde.
RSpec.describe "Invariantes do catálogo de triagens (ADR 0027)" do
  before { Current.city = TEST_CITY_A }
  after { Current.reset; Rails.cache.clear }

  let(:admin) { staff_with("inv-#{SecureRandom.hex(3)}@cidade.gov.br", "municipal_admin") }

  # 1. A restrição da cidade nunca amplia a elegibilidade assinada.
  # Mutação: em Triages::Offer.evaluate, trocar o E por OU entre elegibilidade
  # e restrição, ou pular a elegibilidade quando há linha.
  it "1. a restrição nunca amplia" do
    idoso = { name: "idoso", offer: { "eligibility" => { "gte" => ["profile.age", 60] } } }
    restrictions = [ nil, { "gte" => ["profile.age", 0] }, { "any" => [ { "gte" => ["profile.age", 0] }, { "eq" => ["profile.sex", "male"] } ] },
                     { "not" => { "lt" => ["profile.age", 0] } } ]
    (0..100).step(7).to_a.product(%w[female male]).each do |age, sex|
      context = Protocols::ConditionContext.build(profile: { age: age, sex: sex })
      signed = age >= 60 ? [ "idoso" ] : [] # o que a elegibilidade assinada permite
      restrictions.each do |restriction|
        row = Triages::Offer::Row.new(enabled: true, position: 1, restriction: restriction, available_from: nil, available_until: nil)
        with_row = Triages::Offer.evaluate(protocols: [ idoso ], rows: { "idoso" => row }, context: context,
                                           last_completed: {}, on: Date.new(2026, 10, 5)).map(&:protocol_name)
        expect(with_row - signed).to be_empty, "#{age}/#{sex}/#{restriction.inspect}"
      end
    end
  end

  # 2. Perfil verified não muda pelo canal do cidadão.
  # Mutação: tirar o `profile_verified` de Citizens::SetProfile, ou deixar
  # RegisterPerson sobrescrever o perfil de par existente.
  it "2. perfil verified não muda pelo canal do cidadão" do
    citizen = Citizen.create!(cpf: "52998224725", phone: "+5541998765432", birth_date: "1963-04-02", sex: "female",
                              profile_source: "verified")
    expect(Citizens::SetProfile.call(citizen: citizen, birth_date: "1990-01-01", sex: "male", gender_identity: nil).reason)
      .to eq(:profile_verified)
    Citizens::RegisterPerson.call(phone: citizen.phone, cpf: citizen.cpf,
                                  profile: { birth_date: "1990-01-01", sex: "male", gender_identity: nil })
    expect(citizen.reload).to have_attributes(birth_date: "1963-04-02", sex: "female", profile_source: "verified")
  end

  # 3. Nenhum payload de evento, URL ou log carrega data de nascimento, idade,
  # sexo ou identidade de gênero.
  # Mutação: pôr `sex:` ou `birth_date:` em qualquer DomainEvents.publish do
  # módulo; tirar :sex do filter_parameters; criar rota com o perfil no path.
  it "3. nenhum evento, rota ou log carrega o perfil" do
    active_protocol!("saude-mental-aprofundada")
    active_protocol!("saude-mental", suggestions: [ { "protocol" => "saude-mental-aprofundada", "when" => { "gte" => ["outcome.score", 4] } } ])
    reg = Citizens::RegisterPerson.call(phone: "+5541998765432", cpf: "52998224725",
                                        profile: { birth_date: "1963-04-02", sex: "female", gender_identity: "cis_woman" })
    citizen = reg.payload[:citizen]
    Citizens::SetProfile.call(citizen: citizen, birth_date: "1963-04-03", sex: "female", gender_identity: "travesti")
    started = start_for!(citizen, "saude-mental").payload
    Citizens::SubmitAnswer.call(conversation: started[:conversation], answer: "true", idempotency_key: SecureRandom.uuid)
    Triages::SetOffer.call(protocol_name: "saude-mental", attributes: { "enabled" => true, "position" => 1,
                                                                         "restriction" => { "gte" => ["profile.age", 18] } }, by: admin)
    other = Citizen.create!(cpf: CampaignHistory.cpf_for("inv-verify"), phone: "+5541900000001")
    Citizens::Verify.call(cpf: other.cpf, code: issue_code_for(other), document_checked: true, by: admin,
                          birth_date: "1950-07-08", sex: "male")

    names = %w[citizen.profile_changed triage.suggested triage_offer.changed]
    expect(DomainEvent.where(name: names).distinct.pluck(:name)).to match_array(names)
    forbidden_keys = %w[birth_date sex gender_identity age profile]
    forbidden_values = [ "1963-04-02", "1963-04-03", "1950-07-08", "female", "male", "cis_woman", "travesti",
                         citizen.age.to_s ]
    DomainEvent.find_each do |event|
      keys = deep_keys(event.payload)
      expect(keys & forbidden_keys).to be_empty, "#{event.name}: #{keys.inspect}"
      values = deep_values(event.payload).map(&:to_s)
      expect(values & forbidden_values).to be_empty, "#{event.name}: #{values.inspect}"
    end

    filter = ActiveSupport::ParameterFilter.new(Rails.application.config.filter_parameters)
    expect(filter.filter("birth_date" => "x", "sex" => "x", "gender_identity" => "x").values.uniq).to eq([ "[FILTERED]" ])
    paths = Rails.application.routes.routes.map { |r| r.path.spec.to_s }
    expect(paths.grep(/birth|sex|gender|age\b/)).to be_empty
  end

  # 4. Resultado urgente nunca gera sugestão.
  # Mutação: tirar o `return [] if Protocols::Urgency.urgent?(outcome)`.
  it "4. urgente nunca sugere" do
    active_protocol!("saude-mental-aprofundada")
    definition = active_protocol!("saude-mental", suggestions: [ { "protocol" => "saude-mental-aprofundada", "when" => { "eq" => ["q1", "true"] } } ])
    citizen = profiled_citizen!(age: 30)
    triage = completed_triage!(citizen, "saude-mental")
    urgent = Protocols::Outcome.terminal(trail: [], tier: "alta", priority: 1, score: 99)
    expect(Protocols::Urgency.urgent?(urgent)).to be(true)
    expect(Triages::Suggest.call(triage: triage, outcome: urgent)).to eq([])
    expect(definition).to be_persisted
  end

  # 5. Sugestão nunca aponta para o próprio protocolo.
  # Mutação: tirar o teste de nome igual do gate (Validation::Offer) ou do
  # Triages::Suggest.
  it "5. nunca sugere a si mesmo, nem pelo gate nem em execução" do
    definition = catalog_definition("saude-mental", suggestions: [ { "protocol" => "saude-mental", "when" => { "eq" => ["q1", "true"] } } ])
    expect(Protocols::Gate.call(definition).errors).to include("suggestions[0]: suggestion points to the protocol itself")
    ProtocolDefinition.create!(name: "saude-mental", version: 1, status: "active", definition: definition)
    citizen = profiled_citizen!(age: 30)
    triage = completed_triage!(citizen, "saude-mental")
    outcome = Protocols::Outcome.terminal(trail: [], tier: "media", priority: 5, score: 4)
    expect(Triages::Suggest.call(triage: triage, outcome: outcome)).to eq([])
  end

  # 6. A triagem continua apontando a versão exata do protocolo usada (ADR 0010).
  # Mutação: StartTriage gravar a versão errada (ex.: a primeira, não a ativa).
  it "6. a triagem aponta a versão ativa exata" do
    ProtocolDefinition.create!(name: "saude-mental", version: 1, status: "retired", definition: catalog_definition("saude-mental"))
    v2 = active_protocol!("saude-mental", version: 2)
    citizen = profiled_citizen!(age: 30)
    triage = start_for!(citizen, "saude-mental").payload[:triage]
    expect(triage.protocol_definition_id).to eq(v2.id)
  end

  def deep_keys(value)
    case value
    when Hash then value.keys.map(&:to_s) + value.values.flat_map { |v| deep_keys(v) }
    when Array then value.flat_map { |v| deep_keys(v) }
    else []
    end
  end

  def deep_values(value)
    case value
    when Hash then value.values.flat_map { |v| deep_values(v) }
    when Array then value.flat_map { |v| deep_values(v) }
    else [ value ]
    end
  end
end

# 7. Nenhum dado de perfil ou sugestão de um par aparece para outro par.
# Mutação: buscar o par por id sem o escopo da sessão em people#catalog/profile,
# ou devolver suggestions de triagem de outro par em triages#show.
RSpec.describe "Invariante 7 do ADR 0027: pares não se enxergam", type: :request do
  before do
    create_default_protocol!
    active_protocol!("saude-mental-aprofundada")
    sign_in_citizen("+5541998765432")
  end
  after { Rails.cache.clear }

  it "perfil, catálogo e sugestões de outro par ficam fora da sessão" do
    mine = profiled_citizen!(age: 30, cpf: "52998224725")
    mine.update!(verification_level: "verified")
    other = profiled_citizen!(age: 62, cpf: "52998224725", phone: "+5541900000000")
    source = completed_triage!(other, StartTriage::DEFAULT_PROTOCOL_NAME)
    TriageSuggestion.create!(citizen: other, source_triage: source, protocol_name: "saude-mental-aprofundada")

    get "/citizen/people"
    expect(JSON.parse(response.body)["people"].map { |p| p["id"] }).to eq([ mine.id ])
    get "/citizen/people/#{other.id}/catalog"
    expect(response).to have_http_status(:not_found)
    json_post "/citizen/people/#{other.id}/profile", birth_date: "1990-01-01", sex: "male", gender_identity: nil
    expect(response).to have_http_status(:not_found)
    get "/citizen/people/#{mine.id}/catalog"
    expect(JSON.parse(response.body)["suggested"]).to eq([])
    get "/citizen/triages/#{source.id}"
    expect(JSON.parse(response.body)["suggestions"]).to eq([])
    expect(response.body).not_to include(other.birth_date)
  end
end
```

- [ ] **Step 2: Rode**

Run: `docker compose exec -T -w /rails/.claude/mod15 api bundle exec rspec spec/invariants/triage_catalog_invariants_spec.rb`
Expected: PASS.

- [ ] **Step 3: Prove que a suíte pega as mutações**

Uma de cada vez, aplique a mutação descrita no comentário do invariante, rode a suíte e confira que **o invariante correspondente falha**; desfaça (`/opt/homebrew/bin/git -C apps/api/.claude/mod15 checkout -- <arquivo>`) antes da próxima. No mínimo: (1) em `Triages::Offer.evaluate`, `next unless offer["eligibility"].nil? || ...` → `next unless row || offer["eligibility"].nil? || ...`; (3) em `Citizens::SetProfile`, publicar `citizen.profile_changed` com `sex: citizen.sex` a mais; (4) apagar a linha `return [] if Protocols::Urgency.urgent?(outcome)`; (7) em `CitizenApi::PeopleController#catalog`, trocar `current_citizen_session.citizens.find_by` por `Citizen.find_by`. Anote o resultado (qual exemplo falhou) para o relatório da Task 21.

- [ ] **Step 4: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod15 add spec/invariants/triage_catalog_invariants_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod15 commit -m "test: pin the ADR 0027 triage catalog invariants

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 20: Semente de dev (spec §10)

**Files:**
- Create: `lib/triage_catalog_crew.rb`
- Modify: `db/seeds.rb`
- Test: `spec/lib/triage_catalog_crew_spec.rb`

**Interfaces:**
- Consumes: `Protocols::SaveDraft`, `SubmitForReview`, `Sign`, `Publish`, `Activate` (ciclo assinado, como `lib/analytics_crew.rb`), `SignatureCrew`, `Citizens::RegisterPerson`, `Citizens::SetNeighborhood`, `Triages::SetOffer`.
- Produces: `TriageCatalogCrew.seed_current_city(slug:, ddd:) -> { protocols: [String], restricted_neighborhoods: [String], family: [{ cpf_masked:, age:, sex: }] }`. Idempotente.

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/lib/triage_catalog_crew_spec.rb
require "rails_helper"
require Rails.root.join("lib/signature_crew")
require Rails.root.join("lib/triage_catalog_crew")

# Semente do módulo 15 (spec 2026-10-05 §10): três protocolos pelo ciclo
# assinado, avó e neto no mesmo celular, idoso restrito a dois bairros em
# Curitiba. Idempotente.
RSpec.describe TriageCatalogCrew do
  let!(:city_record) { register_test_city! }

  before do
    create_default_protocol!
    %w[Centro Batel Portão].each { |name| Neighborhood.create!(name: name, source: "seed") }
    staff_with("admin@curitiba.demo", "municipal_admin")
    SignatureCrew.seed_current_city(slug: "curitiba", password: "dev-password")
  end
  after { Rails.cache.clear }

  def seed = described_class.seed_current_city(slug: "curitiba", ddd: "41")

  it "ativa os três protocolos assinados, restringe o idoso e cria a família" do
    result = seed
    expect(result[:protocols]).to eq(%w[saude-mental-aprofundada saude-do-idoso saude-mental])
    %w[saude-mental-aprofundada saude-do-idoso saude-mental].each do |name|
      record = ProtocolDefinition.find_by!(name: name, version: 1)
      expect(record.status).to eq("active")
      expect(ProtocolSignature.where(protocol_definition: record).pluck(:purpose).tally)
        .to eq("publication" => 2, "activation" => 2)
      expect(Protocols::Gate.call(record.definition)).to be_valid
    end
    expect(result[:restricted_neighborhoods]).to eq(%w[Batel Centro])
    expect(TriageOffer.find_by!(protocol_name: "saude-do-idoso").restriction)
      .to eq("in" => [ "citizen.neighborhood_id", Neighborhood.where(name: %w[Batel Centro]).order(:name).pluck(:id) ])

    family = Citizen.where(phone: "+5541944440001").to_a
    avo = family.find { |c| c.age.to_i >= 60 }
    neto = family.find { |c| c.age.to_i < 18 }
    expect([ avo.age, neto.age ]).to eq([ 62, 8 ])
    expect(Triages::Offer.for(citizen: avo).map(&:protocol_name)).to include("saude-do-idoso")
    expect(Triages::Offer.for(citizen: neto).map(&:protocol_name)).not_to include("saude-do-idoso", "saude-mental")
  end

  it "rodar de novo não duplica nada" do
    seed
    expect { seed }.not_to change { [ ProtocolDefinition.count, Citizen.count, TriageOffer.count ] }
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod15 api bundle exec rspec spec/lib/triage_catalog_crew_spec.rb`
Expected: FAIL (`cannot load such file -- .../lib/triage_catalog_crew`).

- [ ] **Step 3: Implemente**

```ruby
# lib/triage_catalog_crew.rb
require_relative "signature_crew"
require_relative "campaign_history"

# Semente de dev do módulo 15 (spec 2026-10-05 §10). Dev é fictício mas imita
# o real: três protocolos passam pelo ciclo assinado de verdade (SaveDraft →
# SubmitForReview → 2 assinaturas → Publish → 2 assinaturas → Activate), o
# aprofundamento primeiro (é o alvo da sugestão da saúde mental); avó (62) e
# neto (8) no mesmo celular, CPF com dígito válido; em Curitiba, o idoso fica
# restrito a dois bairros e a avó mora num deles. Idempotente.
class TriageCatalogCrew
  PHONE_PREFIX = "94444"
  DEEP = "saude-mental-aprofundada"
  ELDERLY = "saude-do-idoso"
  MENTAL = "saude-mental"

  def self.boolean_step(id, prompt, next_id, weight)
    { "id" => id, "prompt" => prompt, "answer_type" => "boolean",
      "branches" => { "true" => next_id, "false" => next_id }, "weights" => { "true" => weight, "false" => 0 } }
  end

  SCORING = { "type" => "weighted", "thresholds" => { "baixa" => 0, "media" => 3, "alta" => 6 },
              "priority_map" => { "baixa" => 9, "media" => 5, "alta" => 3 } }.freeze
  RECOMMENDATIONS = {
    "alta" => { "title" => "Procure sua unidade de saúde", "body" => "Agende uma consulta na sua UBS nos próximos dias." },
    "media" => { "title" => "Converse com a equipe", "body" => "Na próxima ida à UBS, conte o que respondeu aqui." },
    "baixa" => { "title" => "Continue se cuidando", "body" => "Mantenha as consultas de rotina em dia." }
  }.freeze

  PROTOCOLS = {
    DEEP => {
      "name" => DEEP, "version" => 1, "start_step_id" => "desesperanca",
      "steps" => [ boolean_step("desesperanca", "Nas últimas duas semanas, sentiu-se sem esperança na maior parte dos dias?", "isolamento", 3),
                   boolean_step("isolamento", "Tem evitado ver amigos e família?", "rotina", 2),
                   boolean_step("rotina", "Tem deixado de fazer tarefas do dia a dia?", nil, 2) ],
      "scoring" => SCORING, "recommendations" => RECOMMENDATIONS,
      "offer" => { "title" => "Saúde mental — aprofundamento",
                   "summary" => "Perguntas a mais para a equipe entender melhor como você está.",
                   "eligibility" => { "gte" => ["profile.age", 18] } }
    },
    ELDERLY => {
      "name" => ELDERLY, "version" => 1, "start_step_id" => "quedas",
      "steps" => [ boolean_step("quedas", "Caiu alguma vez nos últimos 12 meses?", "memoria", 3),
                   boolean_step("memoria", "Tem esquecido compromissos ou recados com frequência?", "remedios", 2),
                   boolean_step("remedios", "Usa cinco ou mais remédios todos os dias?", nil, 2) ],
      "scoring" => SCORING, "recommendations" => RECOMMENDATIONS,
      "offer" => { "title" => "Saúde do idoso", "summary" => "Avaliação anual de quedas, memória e medicamentos.",
                   "eligibility" => { "gte" => ["profile.age", 60] }, "retake_after_days" => 365 }
    },
    MENTAL => {
      "name" => MENTAL, "version" => 1, "start_step_id" => "humor",
      "steps" => [ boolean_step("humor", "Nas últimas duas semanas, sentiu-se para baixo ou deprimido?", "interesse", 3),
                   boolean_step("interesse", "Perdeu o interesse por coisas de que gostava?", "sono", 3),
                   boolean_step("sono", "Tem dormido mal quase todas as noites?", nil, 2) ],
      "scoring" => SCORING, "recommendations" => RECOMMENDATIONS,
      "offer" => { "title" => "Saúde mental", "summary" => "Três perguntas sobre humor, interesse e sono.",
                   "eligibility" => { "gte" => ["profile.age", 18] }, "retake_after_days" => 30 },
      "suggestions" => [ { "protocol" => DEEP, "when" => { "gte" => ["outcome.score", 6] } } ]
    }
  }.freeze

  FAMILY = [ { key: "avo", age: 62, extra_days: 40, sex: "female" }, { key: "neto", age: 8, extra_days: 100, sex: "male" } ].freeze

  class << self
    def seed_current_city(slug:, ddd:)
      PROTOCOLS.each_key { |name| ensure_protocol!(slug, name) }
      restricted = slug == "curitiba" ? restrict_elderly!(slug) : []
      family = ensure_family!(slug, ddd, restricted.first)
      { protocols: PROTOCOLS.keys, restricted_neighborhoods: restricted.map(&:name), family: family }
    end

    private

    # Retoma de onde parou: cada passo só roda se a versão estiver no estado dele.
    def ensure_protocol!(slug, name)
      record = ProtocolDefinition.find_by(name: name, version: 1)
      return record if record&.status == "active"

      author, first, second, publisher = %w[autor revisora1 revisora2 publisher]
                                         .map { |prefix| User.find_by!(email_address: "#{prefix}@#{slug}.demo") }
      status = -> { ProtocolDefinition.find_by!(name: name, version: 1).status }
      check!(Protocols::SaveDraft.call(definition: PROTOCOLS.fetch(name).deep_dup, by: author), name, "rascunho") if record.nil?
      check!(Protocols::SubmitForReview.call(name: name, version: 1, by: author), name, "revisão") if status.call == "draft"
      if status.call == "in_review"
        [ first, second ].each { |reviewer| sign!(reviewer, name, "publication") }
        check!(Protocols::Publish.call(name: name, version: 1, by: publisher), name, "publicação")
      end
      if status.call == "published"
        [ first, second ].each { |reviewer| sign!(reviewer, name, "activation") }
        check!(Protocols::Activate.call(name: name, version: 1, by: publisher), name, "ativação")
      end
      ProtocolDefinition.find_by!(name: name, version: 1)
    end

    def sign!(reviewer, name, purpose)
      result = Protocols::Sign.call(name: name, version: 1, purpose: purpose, by: reviewer)
      check!(result, name, "assinatura de #{purpose}") unless result.reason == :already_signed
    end

    def check!(result, name, step)
      return result if result.ok?

      raise "semente do catálogo: #{step} de #{name} falhou — #{result.reason}: #{result.message}"
    end

    # Curitiba: o idoso só nos dois primeiros bairros ativos (por nome).
    def restrict_elderly!(slug)
      neighborhoods = Neighborhood.where(active: true).order(:name).first(2)
      raise "semente do catálogo: nenhum bairro (rode a Territory::Seed antes)" if neighborhoods.empty?

      admin = User.find_by!(email_address: "admin@#{slug}.demo")
      attributes = { "enabled" => true, "position" => 1, "available_from" => nil, "available_until" => nil,
                     "restriction" => { "in" => [ "citizen.neighborhood_id", neighborhoods.map(&:id) ] } }
      result = Triages::SetOffer.call(protocol_name: ELDERLY, attributes: attributes, by: admin)
      check!(result, ELDERLY, "linha do catálogo")
      neighborhoods
    end

    def ensure_family!(slug, ddd, neighborhood)
      phone = format("+55%s#{PHONE_PREFIX}0001", ddd)
      FAMILY.map do |member|
        born = Time.zone.today - member[:age].years - member[:extra_days].days
        result = Citizens::RegisterPerson.call(
          phone: phone, cpf: CampaignHistory.cpf_for("#{slug}:catalogo:#{member[:key]}"),
          profile: { birth_date: born.iso8601, sex: member[:sex], gender_identity: nil }
        )
        citizen = check!(result, "família", member[:key]).payload[:citizen]
        if neighborhood && citizen.neighborhood_id.nil?
          Citizens::SetNeighborhood.call(citizen: citizen, neighborhood_id: neighborhood.id)
        end
        { cpf_masked: citizen.cpf_masked, age: citizen.age, sex: citizen.sex }
      end
    end
  end
end
```

(`boolean_step` é método de classe chamado dentro da própria classe na montagem de `PROTOCOLS`; por isso ele vem antes da constante.)

Em `db/seeds.rb`:
- na lista de `require`, depois de `lib/analytics_crew`: `require Rails.root.join("lib/triage_catalog_crew").to_s`;
- no comentário do topo, depois da linha do analyst: `#     Os protocolos do catálogo (saúde do idoso, saúde mental e aprofundamento), a família avó+neto no mesmo celular e o idoso restrito a dois bairros em Curitiba vêm de `lib/triage_catalog_crew.rb` (módulo 15).`;
- depois do bloco do Analytics (antes de `puts "[seeds] cidade ......"`):

```ruby
        # ── Catálogo de triagens (módulo 15, spec 2026-10-05 §10) ─────────────
        # Depois do elenco do ciclo assinado e do território: usa autor,
        # revisoras, publisher, admin e bairros.
        catalog = TriageCatalogCrew.seed_current_city(slug: slug, ddd: ddd)
        puts "[seeds] catálogo .... #{catalog[:protocols].join(', ')} ativos" \
             "#{catalog[:restricted_neighborhoods].any? ? "; idoso só em #{catalog[:restricted_neighborhoods].join(' e ')}" : ''}"
        catalog[:family].each { |p| puts "[seeds] família ..... #{p[:cpf_masked]} (#{p[:age]} anos, #{p[:sex]})" }
```

- [ ] **Step 4: Rode a spec e a semente de dev**

Run:
```bash
docker compose exec -T -w /rails/.claude/mod15 api bundle exec rspec spec/lib/triage_catalog_crew_spec.rb spec/lib/analytics_crew_spec.rb
docker compose exec -T -w /rails/.claude/mod15 api bin/rails db:seed
```
Expected: PASS; a semente imprime as linhas `[seeds] catálogo ....` e `[seeds] família .....` para curitiba e maringa, e rodar `db:seed` de novo não falha.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod15 add lib/triage_catalog_crew.rb db/seeds.rb spec/lib/triage_catalog_crew_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod15 commit -m "chore: seed the dev triage catalog with signed protocols and a family

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 21: Revisão final, suíte completa e api na porta 3032

- [ ] **Step 1:** Confira a cópia do schema contra a tag do `contracts`:

  ```bash
  /opt/homebrew/bin/git -C contracts fetch --tags
  diff <(/opt/homebrew/bin/git -C contracts show protocols-v1.4.0:protocols/schema.json) apps/api/.claude/mod15/config/protocols/schema.json
  ```
  Expected: sem diferença. Se a tag ainda não existe, **pare** e avise: o api não mergeia antes do `contracts`.
- [ ] **Step 2:** Um subagente revisor lê `origin/main..HEAD` do api contra a spec, o ADR 0027, os contratos e as seções "Desvios" e "Divergências" deste plano, com atenção a:
  - nenhum `DomainEvents.publish` com perfil (`grep -rn -A3 "DomainEvents.publish(" app lib` — as chamadas são multilinha; compare call sites × nomes distintos);
  - `Current.city` nunca atribuído em `app/`/`lib/`; `encrypts` sem `deterministic` nos três campos e presentes em `CITY_KEYED_TARGETS`;
  - toda leitura de par pelo cidadão passa por `current_citizen_session.citizens` (nunca `Citizen.find_by(id:)` cru);
  - `app/protocols/` continua puro (`spec/invariants/protocol_engine_purity_spec.rb`);
  - N+1 no catálogo do cidadão e na aba do dashboard (uma consulta por tabela, `includes` nas sugestões).
- [ ] **Step 3:** Corrija os achados e rode a suíte completa com o worker parado:

  ```bash
  docker compose stop worker
  docker compose exec -T -w /rails/.claude/mod15 api bundle exec rspec
  docker compose start worker
  ```
  Expected: 0 falhas. Relatório: commits, contagem e tempo (s/exemplo; > 0,15 s/exemplo com host calmo é regressão), evidência de mutação da Task 19, specs existentes alteradas e por quê (`adr_pointers_spec`, `protocol_engine_purity_spec`, `domain_events_bindings_spec`, as cinco da Task 10, as de validação da Task 8).
- [ ] **Step 4:** Para a prova no navegador (spec §9.5, com o usuário) e para os planos do wpda e do dashboard, suba o api do worktree no container, na porta **3032**, sem derrubar o servidor principal:

  ```bash
  docker compose exec -d -w /rails/.claude/mod15 api bin/rails server -b 0.0.0.0 -p 3032 -P tmp/pids/server-mod15.pid
  ```

  O Vite do worktree de cada front aponta o proxy para `http://api:3032` (`VITE_API_PROXY_TARGET`). Roteiro: no wpda, o celular `+55 41 94444-0001` com a avó (62) e o neto (8) mostra catálogos diferentes; a saúde mental respondida com "sim" nas três perguntas sugere o aprofundamento; o `admin@curitiba.demo` pausa o aprofundamento na aba "Catálogo de triagens" (step-up) e a sugestão expira na próxima leitura do catálogo. Login, OTP e TOTP são do usuário (senhas e TOTP da semente de dev podem ser mostrados no chat se ele pedir).
- [ ] **Step 5:** **Pare.** Merge, push, board e docs (página de status do módulo) só com autorização explícita do usuário, uma etapa de cada vez. Ordem: `contracts` → api → wpda e dashboard. **O api novo exige `protocol_name` em `POST /citizen/conversations`: o wpda antigo deixa de iniciar triagem até o wpda novo entrar** — publique os dois juntos. Rollout por cidade: publicar a imagem nova e rodar `city:migrate:all` dela antes de cortar tráfego; antes de configurar protocolo com elegibilidade numa cidade, publicar versão nova do termo cobrindo o perfil (`city:consent_term:publish`). Ao voltar o checkout para a main: `DROP DATABASE` dos dois bancos de teste de cidade e `city:test_databases` de novo; derrube o servidor da 3032 (`kill $(cat tmp/pids/server-mod15.pid)` no container).

---

## Self-review (feito ao escrever o plano)

**Cobertura da spec:**

| Spec / contrato | Task |
|---|---|
| §3.1 perfil em `citizens` (cifra, valores, `profile_source`, `age`, `profile?`) | 4, 5 |
| §3.2 `triage_offers` (único, período, posição ≥ 1, `updated_by_user_id`, evento) | 4, 14 |
| §3.3 `triage_suggestions` (estados, trigger, índice parcial) | 4 |
| §4.1 `gte`/`lte`, contexto com variáveis reservadas, chamada antiga, total | 1 |
| §4.2 schema 1.4.0 (`offer`, `suggestions`, padrão do `protocol` = do `name`) | 2 |
| §4.3 gate (variáveis por lugar, passo inexistente, autossugestão, `retake_after_days`, prefixo reservado) e aviso de inexistente | 1, 2, 3 |
| §5.1 regra de oferta (pura, tabela de casos) | 9 |
| §5.2 `StartTriage` por nome, lock, `not_offered`, sugestão `taken`, WhatsApp | 10 |
| §5.3 sugestões na conclusão; expiração preguiçosa | 11, 12, 13 |
| §5.4 `SetProfile`; `Verify` com perfil | 6, 8 |
| §5.5 LGPD (Erase, revogação, `filter_parameters`, eventos só com ids) | 5, 18, 19 |
| §6.1 rotas do cidadão (people, profile, catalog com `reference_units`/`source_title`, conversations, triages) | 7, 10, 12, 13 |
| §6.2 `GET/PUT /triage_catalog` com step-up; `simulate_offer` (200 sempre) | 16, 17 |
| §7 contadores com supressão | 1 (desvio), 12, 15, 16 |
| §9.1 testes do api; §9.2 invariantes | todas; 19 |
| §10 semente | 20 |
| §11 rollout (`VALID_RANGE`, `city:migrate:all`, termo) | 4, 21 |
| Contratos §0.1 (`POST /citizen/people`), §0.2 (uma em andamento, `triage_in_progress`) | 7, 10 |

**Placeholders:** nenhum "TBD"/"implementar depois"; os ajustes em specs existentes (Tasks 8, 10, 13, 16) dizem a linha e o texto exato, e os únicos "se" condicionais são sobre specs cujo conteúdo exato não foi lido ao planejar (`attendance_contract_spec`, `operator_grant_access_spec`, exemplos antigos de `GET /citizen/triages/:id`), com a regra do ajuste escrita.

**Tipos e nomes:** `Triages::Offer::Item`/`Row` (Task 9) são os usados em 11, 12, 13, 19; `Triages::Offer.title_for` (9) em 12, 16; `Protocols::Validation::Offer::RESTRICTION` (2) em 14; `Citizens::ProfileValues` (6) em 7, 8; `Citizens::ProfileJson` (5) em 7, 8; `start_for!`/`profiled_citizen!`/`completed_triage!`/`active_protocol!` (4) nas specs de 9 em diante; `start_citizen_triage` (10) em 13.

**Ordem e suíte verde:** a Task 1 (avaliação e validação de `gte`/`lte`) vem antes da cópia do schema (Task 2); `protocol_name` continua opcional nos comandos (padrão = nome de hoje), só a rota o exige (Task 10, que ajusta as specs de request); catálogo vazio = comportamento de hoje (Task 9, "sem linha").

**Review Focus:** cada linha tem teste na task dona — 1: Tasks 5 e 9; 2: Tasks 12, 13, 19; 3: Tasks 10, 12, 13; 4: Task 10 (threads); 5: Tasks 9 e 14.

---

## Divergências propostas ao contrato

1. **`warnings` acrescentado** (aditivo): `POST /authoring/protocols/gate` ganha `warnings: [String]` **só quando há aviso** (sugestão para protocolo sem nenhuma versão na cidade, spec §4.3 "avisa sem bloquear"); `POST /authoring/protocols/simulate_offer` devolve `warnings` sempre (lista, vazia sem aviso). O contrato não diz onde o aviso aparece; o dashboard pode ignorar a chave.
2. **`PUT /triage_catalog/:protocol_name` com `enabled` que não é booleano** → 422 `invalid_enabled` (código novo; o contrato lista só `invalid_restriction`, `invalid_period`, `invalid_position`).
3. **Restrição limitada a 4096 bytes de JSON** → `invalid_restriction` (precisão do motivo existente).
4. **Protocolo inexistente/inativo pedido em `POST /citizen/conversations` por um par** → 409 `not_offered` (o contrato não distingue; `no_protocol`/503 fica só para o caminho sem cidadão).
5. **Catálogo do cidadão:** o protocolo em andamento aparece só em `in_progress`, fora de `suggested`/`available` (o contrato não diz).
6. **`POST /citizen/people` com par já existente** também ignora `neighborhood_id` (como ignora o perfil: "não sobrescreve"; a troca é por `POST /citizen/people/:id/neighborhood`).
7. **Validação presencial:** chave `gender_identity` ausente mantém o valor declarado; presente (inclusive `null`) grava o que veio.
8. **Contadores:** "oferecida" soma uma por leitura do catálogo (não por pessoa distinta) e `started` conta triagens de todos os canais, inclusive revogadas (como o Analytics); a janela é hoje + 29 dias anteriores, no fuso da cidade.
