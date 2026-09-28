# Módulo 11 — Território (api, fatias 1 a 5) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Bairros por cidade, cobertura (bairro → unidades), bairro declarado pelo cidadão e copiado na triagem, unidade de referência para o cidadão e para o desfecho, endereço da unidade e filtro por bairro com supressão de 1 a 4 nos cinco painéis com cidadão (F-11.1 a F-11.7, ADR 0023).

**Architecture:** Uma migração de cidade só de expansão cria `neighborhoods` e `neighborhood_coverages` e acrescenta colunas em `health_units`, `citizens` e `triages`; um trigger em `db/city_triggers.sql` torna imutável o bairro copiado na triagem. Comandos em `app/commands/territory/` e `Citizens::SetNeighborhood` devolvem `Result` e publicam eventos só com ids. `Territory::ReferenceUnits` é a fonte única da unidade de referência. `Admin::NeighborhoodFilter` + `Admin::SmallCount` recortam e suprimem as cinco queries de `/admin/api`, sem mudar nada quando o parâmetro não vem.

**Tech Stack:** Rails 8.1 (API), PostgreSQL, RSpec, Active Record Encryption (chave da cidade), Rake.

**Spec:** `docs/superpowers/specs/2026-09-28-module-11-territory-design.md` e `docs/adr/0023.md` (leia os dois antes de começar; este plano argumenta a partir deles).

## Desvios da spec (e precisões de contrato)

Os planos do dashboard e do wpda leem os mesmos contratos. Onde a spec é omissa ou o código real obrigou a escolher, a escolha está aqui, com o motivo:

1. **Relatório público sem unidade de referência (decisão do usuário, 2026-09-28).** A spec §4.1 dizia que `GET /citizen/triages/:id` **e o relatório** ganhariam `reference_units`. O relatório é `GET /r/:token` (`ReportsController`), público pelo token, sem login. Decisão: `reference_units` sai **só** em `GET /citizen/triages/:id` (sessão do cidadão) e, no dashboard, como `reference_unit_ids` do atendimento. O JSON de `/r/:token` não ganha `reference_units` nem nada de bairro — há teste que prova isso (Task 9).
2. **Formato de `reference_units` (alinhado com o wpda):** `[{ id, name, kind, address: { street, number, complement, zip } }]`, cada campo de `address` podendo ser `null`; `zip` são 8 dígitos sem hífen (`"80020310"`). O objeto `address` sempre existe, mesmo com os quatro campos nulos.
3. **Envelopes de lista:** `GET /citizen/neighborhoods` → `{ neighborhoods: [{ id, name }] }` (padrão de `{ people: [...] }`, alinhado com o wpda); `GET /territory/neighborhoods` → `{ neighborhoods: [{ id, name, source, active, units: [{ id, name, active }] }] }`. Escritas de `/territory` → `{ neighborhood: {...} }` (201 no create). `POST /citizen/people/:id/neighborhood` → `{ person: { id, cpf_masked, verification_level, neighborhood } }`.
4. **Leitura do atendimento usada no desfecho:** o formulário de desfecho (`ClosePanel` em `apps/dashboard/src/modules/attendance/UnitQueue.tsx`) usa as linhas de `GET /attendance/units/:id/queue`; não existe leitura de um atendimento avulso. `reference_unit_ids` entra em **cada linha** da fila (`waiting` e `in_care`) e **nunca inclui a própria unidade do atendimento** (`health_unit_id` da linha) — decisão do usuário, 2026-09-28; ordem por nome.
5. **Rota nova `GET /admin/api/neighborhoods`** (fora da spec; decisão do usuário, 2026-09-28): o filtro vale para **todos** os papéis que leem `/admin/api`, e `/territory/neighborhoods` é só `municipal_admin`. Só leitura, mesma autorização dos demais painéis da cidade (`require_city_membership`, operador por grant incluído). Formato **sem o envelope `data`**, como combinado com o dashboard: `{ neighborhoods: [{ id, name, active }] }`, todos (inclusive inativos), por nome.
6. **`filter` fica dentro de `data`** do envelope, ao lado de `scope`: `data.filter = { neighborhood: { id, name } | "none" | null }`.
7. **Listas de amostra suprimidas vêm `null`** (a chave continua; contrato confirmado com o dashboard): `sampleTriages` (Classificação) e `reports` (Relatórios). KPI com valor suprimido tem `delta: null` (hoje todo `delta` já é `null`); o KPI "Taxa de conclusão" suprimido vem com `tone: "neutral"`.
8. **Bairro usado na referência do cidadão:** `GET /citizen/triages/:id` usa o bairro **copiado na triagem** (mesma regra do atendimento), não o atual do cidadão.
9. **`blank_name`** cobre nome vazio **e** nome acima de 120 caracteres (a spec não dá código para o segundo caso).
10. **Cobertura:** `health_unit_ids` ausente ou que não seja lista de textos → 422 `inactive_unit` (a mesma recusa de unidade inválida; a spec não prevê outro código).
11. **Endereço da unidade:** só as chaves presentes no corpo mudam (o dashboard anterior ao módulo 11 salva nome e tipo sem apagar o endereço); o CEP aceita `80020-310` e grava só os dígitos; valor não escalar em `address_zip` → `invalid_zip`, em `neighborhood_id` → `invalid_neighborhood`, nos outros → `invalid_unit`. As leituras de `/attendance/units` (index e all) devolvem `address_street`, `address_number`, `address_complement`, `address_zip`, `neighborhood_id`.
12. **`POST /citizen/people/:id/neighborhood` sem a chave `neighborhood_id`** → 422 `invalid_neighborhood` (só `null` explícito apaga o bairro).
13. **Chave de semente (decisão do usuário, 2026-09-28; a spec §3.1/§3.4 comparava por `lower(name)`):** `neighborhoods` ganha `seed_key` (texto opcional, único quando presente — índice parcial `idx_neighborhoods_seed_key`). Cada item do YAML ganha `key` estável (slug do nome original, ex.: `santa-felicidade`). A semente casa **só** por `seed_key`: bairro criado por ela grava a chave; bairro manual fica com `seed_key` nulo; renomear no dashboard mantém a chave, e rodar a semente de novo não recria nem renomeia. Se já existe bairro com o mesmo nome (`lower`) e outra chave (ou nenhuma), a semente **não** cria duplicado nem adota a chave: registra aviso e pula. `seed_key` não sai em nenhum JSON.
14. **Exceção única ao bairro imutável da triagem (decisão do usuário, 2026-09-28; o ADR 0023 diz "imutável depois" e precisa de emenda no docs):** a anonimização por revogação (`AnonymizeRevokedTriageJob`, consumidor de `consent.revoked`) zera `triages.neighborhood_id`. O trigger aceita mudar o bairro **para NULL** somente quando a linha fica (ou já está) com `status = 'aborted_by_revocation'` na mesma UPDATE; qualquer outra mudança continua recusada. Consequência: triagem revogada cai em "Sem bairro" nos painéis.

## Global Constraints

- Tudo no banco de cada cidade (`db/city_migrate`, `db/city_schema.rb` à mão, `db/city_triggers.sql`); nada no banco de plataforma. A migração é só de expansão e reversível (`down`).
- `db/city_triggers.sql` é re-executado por migrações antigas (replay do zero): todo trigger novo fica dentro de `DO $do$ ... END $do$` guardado por existência — aqui, da **coluna** `triages.neighborhood_id` (a tabela `triages` existe antes dela).
- O dump `db/city_schema.rb` é feito à mão; o juiz é `spec/services/city_schema_spec.rb`. Texto exato de CHECK vem de `pg_get_constraintdef` de um banco migrado.
- Migração de cidade roda pela rake de cidade: `bin/rails city:migrate[curitiba]`, `city:migrate[maringa]`; bancos de teste por `bin/rails city:test_databases`. **Branch com migração deixa os bancos de teste à frente da main:** ao voltar para a main, `DROP DATABASE rota_saude_test_city_a` e `rota_saude_test_city_b` e rode `city:test_databases` de novo (não encoste em `rota_saude_no_city_selected` nem nos bancos de dev).
- Eventos (`DomainEvents.publish`) carregam **só ids** (e `source`). Nunca nome de bairro junto com dado do cidadão, CPF, telefone ou nome de unidade. Todo evento novo é declarado em `config/initializers/domain_events.rb` com `to: []`.
- `DomainEvents.publish(` e `Platform.audit(` costumam ser **multilinha**: grep de uma linha devolve lista incompleta sem erro. Para conferir call sites, use `grep -rn -A3 "DomainEvents.publish(" app | grep -oE '"[a-z_]+\.[a-z_]+"' | sort -u`.
- `Current.city` **nunca** é atribuído em `app/` ou `lib/` (guarda `spec/architecture/current_city_assignment_spec.rb`); rake e semente usam `CityConnection.with(city)`. Em spec, o harness já põe `TEST_CITY_A` (`spec/support/city_test_databases.rb:171`).
- Todo comando devolve `Result` (`app/commands/result.rb`).
- Recusa de papel em `/territory`: HTTP 403 `{"error":"missing_role"}`.
- Nenhum arquivo em `app/`, `lib/`, `config/` ou `db/` menciona `viacep` (nem em comentário): a invariante varre o fonte. O `api` nunca chama serviço de CEP.
- Specs de request precisam de `type: :request` (a inferência por pasta está desligada). Arquivos novos em `spec/support/` precisam de `require_relative` em `spec/rails_helper.rb` (eles são carregados um a um).
- Constante dentro de bloco `describe` vaza para o topo: em spec use variável local, não `CONSTANTE = ...`.
- Nenhum teste com data fixa contra o relógio real: use `travel_to` ou datas relativas a `Time.current`.
- Commits em inglês, Conventional Commits com o tipo por extenso (`feat`, `fix`, `refactor`, `test`, `chore`, `docs`...), terminando com a linha `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta).

## Ambiente de execução

- Worktree do api já criado: `apps/api/.claude/mod11`, branch `feat/mod-11-territory`, a partir de `origin/main` (`a2d20c8`). Ele **não** tem `config/master.key` (arquivo ignorado): copie antes do primeiro comando.

  ```bash
  cp apps/api/config/master.key apps/api/.claude/mod11/config/master.key
  ```

- No compose, `./apps/api` é montado em `/rails` e `/rails/tmp` é o volume nomeado `rails-tmp`; por isso o worktree mora em `apps/api/.claude/mod11` (dentro do bind mount), visto no container como `/rails/.claude/mod11`. Todo comando roda dentro do container, nesse diretório:

  ```bash
  docker compose exec -T -w /rails/.claude/mod11 api bundle exec rspec <arquivos>
  ```

- Depois da migração (Task 1), recarregue os bancos de teste:

  ```bash
  docker compose exec -T -w /rails/.claude/mod11 api bin/rails city:test_databases
  ```

- Suíte completa só com o worker parado e sem outra sessão rodando suíte completa ao mesmo tempo (evita "too many clients"):

  ```bash
  docker compose stop worker
  docker compose exec -T -w /rails/.claude/mod11 api bundle exec rspec
  docker compose start worker
  ```

- Antes de tocar `apps/api`, avise a sessão dona do api (sessão "API"). Os planos do dashboard e do wpda dependem dos contratos acima; ordem de merge: api antes de dashboard e wpda.

## Review Focus

1. **`neighborhood_id` malformado** (texto que não é UUID, lista, objeto) em `POST /citizen/conversations`, `POST /citizen/people/:id/neighborhood` e nos painéis de `/admin/api`: 422 `invalid_neighborhood`, nunca 500, e nenhum `Citizen` criado. Testes: Task 8 ("bairro inválido com CPF novo") e Task 14 ("parâmetro inválido").
2. **Pessoa que já tem bairro manda outro `neighborhood_id` ao começar a triagem:** o valor é ignorado (a troca é pela rota própria) e a triagem copia o bairro que ela já tinha. Teste: Task 8 ("pessoa que já tem bairro").
3. **Unidade desativada depois de entrar na cobertura, ou bairro coberto só pela própria unidade do atendimento:** some de `reference_units` e de `reference_unit_ids` (esta nunca traz a unidade da linha), mas continua em `/territory/neighborhoods` com `active: false`. Testes: Task 3, Task 9, Task 10.
4. **Salvar a unidade sem as chaves de endereço** (dashboard ainda sem o módulo 11, durante o rollout): o endereço gravado continua lá. Teste: Task 6 ("update sem as chaves de endereço").
5. **Painel filtrado com bairro de 5 ou mais casos em que uma categoria tem 1 caso:** o total aparece, a categoria e o `share` dela saem `{ suppressed: true }`. Testes: Task 12 ("bairro grande com uma categoria de 1 caso") e Task 13.

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| `db/city_migrate/20260928100001_create_territory.rb` | tabelas, colunas, trigger | 1 |
| `db/city_triggers.sql` | `triages_neighborhood_immutable` (exceção: NULL na revogação) | 1 |
| `app/jobs/anonymize_revoked_triage_job.rb` | revogação apaga o bairro | 7 |
| `db/city_schema.rb` | dump à mão | 1 |
| `app/models/neighborhood.rb`, `neighborhood_coverage.rb` | modelos | 1, 2 |
| `app/models/health_unit.rb`, `citizen.rb`, `triage.rb` | associações; endereço da unidade | 1, 6 |
| `spec/support/territory_helpers.rb` | helpers de spec | 1 |
| `spec/adr_pointers_spec.rb` | `VALID_RANGE` 1..23 | 1 |
| `app/commands/territory/*.rb` | `CreateNeighborhood`, `RenameNeighborhood`, `SetNeighborhoodActive`, `ReplaceCoverage` | 2 |
| `config/initializers/domain_events.rb` | eventos do território | 2 |
| `app/controllers/territory_controller.rb` + `config/routes.rb` | `/territory` | 3 |
| `app/services/territory/seed.rb`, `db/seeds/territory/*.yml`, `lib/tasks/territory.rake` | semente e rake | 4 |
| `app/controllers/health_units_controller.rb` | endereço em `/attendance/units` | 6 |
| `app/commands/citizens/set_neighborhood.rb`, `app/commands/start_triage.rb` | bairro do cidadão; cópia na triagem | 7 |
| `app/controllers/citizen_api/neighborhoods_controller.rb`, `people_controller.rb`, `conversations_controller.rb` | `/citizen` | 8 |
| `app/services/territory/reference_units.rb`, `app/controllers/citizen_api/triages_controller.rb` | unidade de referência | 9 |
| `app/models/attendance.rb`, `app/controllers/attendances_controller.rb` | `reference_unit_ids` na fila | 10 |
| `app/queries/admin/small_count.rb`, `neighborhood_filter.rb` | supressão e filtro | 11 |
| `app/queries/admin/{overview,classification,triages,reports,conversations}_query.rb` | queries filtradas | 12, 13 |
| `app/controllers/admin/api/*` | parâmetro, `filter`, `GET /admin/api/neighborhoods` | 14 |
| `lib/territory_crew.rb`, `db/seeds.rb` | semente de dev | 15 |
| `spec/invariants/territory_invariants_spec.rb` | invariantes + mutação | 16 |

---

## Fatia 1 — F-11.1 + F-11.2 (bairros, cobertura, semente)

### Task 1: Migração, trigger, modelos mínimos, helpers de spec e guarda de ADR

**Files:**
- Create: `db/city_migrate/20260928100001_create_territory.rb`
- Modify: `db/city_triggers.sql` (acrescentar ao fim), `db/city_schema.rb`
- Create: `app/models/neighborhood.rb`, `app/models/neighborhood_coverage.rb`
- Modify: `app/models/health_unit.rb`, `app/models/citizen.rb`, `app/models/triage.rb`
- Create: `spec/support/territory_helpers.rb`; Modify: `spec/rails_helper.rb`
- Modify: `spec/adr_pointers_spec.rb`
- Test: `spec/models/territory_tables_guard_spec.rb`

**Interfaces:**
- Produces: tabelas `neighborhoods` (`name`, `active`, `source`, `seed_key`), `neighborhood_coverages` (`neighborhood_id`, `health_unit_id`, `created_at`); colunas `health_units.address_street/address_number/address_complement/address_zip/neighborhood_id`, `citizens.neighborhood_id`, `triages.neighborhood_id`; `Neighborhood#coverages`, `#health_units`; `NeighborhoodCoverage` (`belongs_to :neighborhood, :health_unit`); `HealthUnit#neighborhood`, `Citizen#neighborhood`, `Triage#neighborhood`.
- Helpers de spec: `territory_triage!(neighborhood, status: "completed", priority: 1, tier: "alta", created_at: 2.hours.ago) → Triage` (cidadão web novo, conversa própria, bairro copiado); `territory_report!(triage) → ReportSnapshot` (assinatura falsa, só para painéis).

- [ ] **Step 1: Confira o número da migração**

Run: `ls apps/api/.claude/mod11/db/city_migrate | tail -2`
Expected: a última é `20260928000001_create_professionals.rb`. Se houver outra mais nova, use um número maior que ela.

- [ ] **Step 2: Escreva os helpers de spec e registre-os**

```ruby
# spec/support/territory_helpers.rb
# Módulo 11 (ADR 0023): casos com bairro para os painéis e para o trigger.
module TerritoryHelpers
  # Triagem de um cidadão web NOVO (conversa própria), com o bairro copiado no
  # INSERT — como StartTriage faz. nil = cidadão sem bairro.
  def territory_triage!(neighborhood, status: "completed", priority: 1, tier: "alta", created_at: 2.hours.ago)
    protocol = ProtocolDefinition.find_by(name: StartTriage::DEFAULT_PROTOCOL_NAME, status: "active") ||
               create_default_protocol!
    phone = "+55419#{SecureRandom.random_number(10**8).to_s.rjust(8, '0')}"
    citizen = Citizen.create!(cpf: "52998224725", phone: phone, neighborhood: neighborhood)
    completed = status == "completed"
    conversation = Conversation.create!(channel: "web", citizen: citizen, phone: phone,
                                        state: completed ? "completed" : "consented", created_at: created_at)
    Triage.create!(conversation: conversation, protocol_definition: protocol, protocol_name: protocol.name,
                   status: status, tier: completed ? tier : nil, priority: completed ? priority : nil,
                   answers: {}, created_at: created_at, completed_at: completed ? created_at + 5.minutes : nil,
                   neighborhood_id: neighborhood&.id)
  end

  # Relatório só para os painéis (assinatura falsa: /r/:token não o acha).
  def territory_report!(triage)
    token = "tok-#{SecureRandom.hex(8)}"
    ReportSnapshot.create!(triage: triage, protocol_definition: triage.protocol_definition, token: token,
                           signature: "sig-#{token}", payload: { "tier" => triage.tier },
                           outcome: { "tier" => triage.tier }, expires_at: 10.days.from_now)
  end
end

RSpec.configure { |c| c.include TerritoryHelpers }
```

Em `spec/rails_helper.rb`, logo depois de `require_relative "support/appointment_helpers"`:

```ruby
require_relative "support/territory_helpers"
```

- [ ] **Step 3: Escreva a spec de guarda (falha: tabelas e colunas não existem)**

```ruby
# spec/models/territory_tables_guard_spec.rb
require "rails_helper"

# Módulo 11 (ADR 0023): o bairro copiado na triagem nunca muda; o banco recusa
# por SQL direto, sem passar pelo modelo. Nome, origem, par de cobertura e CEP
# também são garantidos pelo banco.
RSpec.describe "Guardas das tabelas de território" do
  def sql(statement) = ApplicationRecord.connection.execute(statement)

  def insert_neighborhood(name_sql, source_sql)
    sql("INSERT INTO neighborhoods (id, name, source, active, created_at, updated_at) " \
        "VALUES (gen_random_uuid(), #{name_sql}, #{source_sql}, true, now(), now())")
  end

  let(:centro) { Neighborhood.create!(name: "Centro", source: "seed") }
  let(:batel) { Neighborhood.create!(name: "Batel", source: "manual") }

  it "triagem: o bairro gravado no INSERT não muda por UPDATE, nem para nulo" do
    triage = territory_triage!(centro)
    expect { sql("UPDATE triages SET neighborhood_id = '#{batel.id}' WHERE id = '#{triage.id}'") }
      .to raise_error(ActiveRecord::StatementInvalid, /neighborhood_id never changes/)
    expect { sql("UPDATE triages SET neighborhood_id = NULL WHERE id = '#{triage.id}'") }
      .to raise_error(ActiveRecord::StatementInvalid, /neighborhood_id never changes/)
  end

  it "revogação: ir para NULL passa quando a linha fica (ou está) em aborted_by_revocation; outro bairro, não" do
    triage = territory_triage!(centro, status: "in_progress")
    expect { sql("UPDATE triages SET neighborhood_id = NULL, status = 'aborted_by_revocation' WHERE id = '#{triage.id}'") }
      .not_to raise_error
    expect(triage.reload.neighborhood_id).to be_nil

    revoked = territory_triage!(centro, status: "in_progress")
    sql("UPDATE triages SET status = 'aborted_by_revocation' WHERE id = '#{revoked.id}'")
    expect { sql("UPDATE triages SET neighborhood_id = '#{batel.id}' WHERE id = '#{revoked.id}'") }
      .to raise_error(ActiveRecord::StatementInvalid, /neighborhood_id never changes/)
    expect { sql("UPDATE triages SET neighborhood_id = NULL WHERE id = '#{revoked.id}'") }.not_to raise_error
  end

  it "triagem sem bairro não ganha bairro depois; as outras colunas continuam mudando" do
    triage = territory_triage!(nil, status: "in_progress")
    expect { sql("UPDATE triages SET neighborhood_id = '#{centro.id}' WHERE id = '#{triage.id}'") }
      .to raise_error(ActiveRecord::StatementInvalid, /neighborhood_id never changes/)
    expect { triage.update!(current_step: "febre") }.not_to raise_error
  end

  it "bairro: nome único sem diferenciar maiúsculas, pelo índice" do
    centro
    expect { insert_neighborhood("'CENTRO'", "'manual'") }.to raise_error(ActiveRecord::RecordNotUnique)
  end

  it "bairro: seed_key única quando presente; várias nulas convivem" do
    Neighborhood.create!(name: "Centro", source: "seed", seed_key: "centro")
    expect { Neighborhood.create!(name: "Centro Novo", source: "seed", seed_key: "centro") }
      .to raise_error(ActiveRecord::RecordNotUnique)
    Neighborhood.create!(name: "Batel", source: "manual")
    expect { Neighborhood.create!(name: "Ahu", source: "manual") }.not_to raise_error
  end

  it "bairro: nome vazio, com espaço nas pontas ou origem desconhecida é recusado pelo CHECK" do
    expect { insert_neighborhood("'  '", "'manual'") }.to raise_error(ActiveRecord::StatementInvalid, /ck_neighborhoods_name/)
    expect { insert_neighborhood("' Batel'", "'manual'") }.to raise_error(ActiveRecord::StatementInvalid, /ck_neighborhoods_name/)
    expect { insert_neighborhood("'Ahu'", "'import'") }.to raise_error(ActiveRecord::StatementInvalid, /ck_neighborhoods_source/)
  end

  it "cobertura: o par (bairro, unidade) é único" do
    unit = create_unit
    NeighborhoodCoverage.create!(neighborhood: centro, health_unit: unit)
    expect { NeighborhoodCoverage.create!(neighborhood: centro, health_unit: unit) }
      .to raise_error(ActiveRecord::RecordNotUnique)
  end

  it "unidade: CEP só com 8 dígitos" do
    unit = create_unit
    expect { sql("UPDATE health_units SET address_zip = '8002031' WHERE id = '#{unit.id}'") }
      .to raise_error(ActiveRecord::StatementInvalid, /ck_health_units_address_zip/)
    expect { sql("UPDATE health_units SET address_zip = '80020310' WHERE id = '#{unit.id}'") }.not_to raise_error
  end
end
```

- [ ] **Step 4: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod11 api bundle exec rspec spec/models/territory_tables_guard_spec.rb`
Expected: FAIL com `uninitialized constant Neighborhood` (ou `unknown attribute 'neighborhood'`).

- [ ] **Step 5: Escreva a migração**

```ruby
# db/city_migrate/20260928100001_create_territory.rb
# Território (ADR 0023; spec 2026-09-28-module-11-territory §3): bairros da
# cidade, cobertura (bairro → unidades), endereço da unidade, bairro declarado
# pelo cidadão e bairro COPIADO na triagem, imutável depois — trigger em
# db/city_triggers.sql, a mesma fonte que load_city_schema executa depois de
# carregar o dump. Só expansão: nada existente muda de forma.
class CreateTerritory < ActiveRecord::Migration[8.1]
  def up
    create_table :neighborhoods, id: :uuid do |t|
      t.string :name, limit: 120, null: false
      t.boolean :active, null: false, default: true
      t.string :source, null: false
      # Chave estável da semente (desvio 13): casa item do YAML com a linha
      # mesmo depois de renomeada; nula em bairro manual.
      t.string :seed_key
      t.timestamps
    end
    add_index :neighborhoods, "lower((name)::text)", unique: true, name: "idx_neighborhoods_name_ci"
    add_index :neighborhoods, :seed_key, unique: true, where: "(seed_key IS NOT NULL)", name: "idx_neighborhoods_seed_key"
    add_check_constraint :neighborhoods, "length(btrim(name::text)) > 0 AND name::text = btrim(name::text)",
                         name: "ck_neighborhoods_name"
    add_check_constraint :neighborhoods, "source::text = ANY (ARRAY['seed'::text, 'manual'::text])",
                         name: "ck_neighborhoods_source"

    create_table :neighborhood_coverages, id: :uuid do |t|
      t.references :neighborhood, type: :uuid, null: false, foreign_key: true, index: false
      t.references :health_unit, type: :uuid, null: false, foreign_key: true, index: true
      t.datetime :created_at, null: false
    end
    add_index :neighborhood_coverages, %i[neighborhood_id health_unit_id], unique: true,
              name: "idx_neighborhood_coverages_pair"

    add_column :health_units, :address_street, :string, limit: 160
    add_column :health_units, :address_number, :string, limit: 20
    add_column :health_units, :address_complement, :string, limit: 80
    add_column :health_units, :address_zip, :string, limit: 8
    add_check_constraint :health_units, "address_zip IS NULL OR address_zip::text ~ '^[0-9]{8}$'::text",
                         name: "ck_health_units_address_zip"
    add_reference :health_units, :neighborhood, type: :uuid, foreign_key: true, index: true

    add_reference :citizens, :neighborhood, type: :uuid, foreign_key: true, index: true

    add_reference :triages, :neighborhood, type: :uuid, foreign_key: true, index: false
    add_index :triages, %i[neighborhood_id created_at], name: "idx_triages_neighborhood_created"

    execute File.read(Rails.root.join("db/city_triggers.sql"))
  end

  def down
    execute "DROP TRIGGER IF EXISTS triages_neighborhood_immutable ON triages"
    execute "DROP FUNCTION IF EXISTS rota_triage_neighborhood_guard()"
    remove_reference :triages, :neighborhood, foreign_key: true, index: false
    remove_reference :citizens, :neighborhood, foreign_key: true, index: true
    remove_reference :health_units, :neighborhood, foreign_key: true, index: true
    remove_check_constraint :health_units, name: "ck_health_units_address_zip"
    remove_column :health_units, :address_zip
    remove_column :health_units, :address_complement
    remove_column :health_units, :address_number
    remove_column :health_units, :address_street
    drop_table :neighborhood_coverages
    drop_table :neighborhoods
  end
end
```

Nota: `remove_reference :triages` derruba também `idx_triages_neighborhood_created` (o índice cai junto com a coluna).

- [ ] **Step 6: Acrescente o trigger ao fim de `db/city_triggers.sql`**

```sql
-- triages.neighborhood_id (ADR 0023; spec 2026-09-28-module-11-territory §3.3):
-- o bairro do cidadão é COPIADO na criação da triagem (StartTriage) e nunca
-- muda depois — nem para outro bairro, nem de nulo para um bairro. É o que faz
-- o painel contar cada caso no bairro onde a pessoa morava quando ele
-- aconteceu. UMA exceção (decisão de 2026-09-28): ir para NULL quando a linha
-- fica em aborted_by_revocation — a anonimização da revogação
-- (AnonymizeRevokedTriageJob) apaga o bairro junto com o conteúdo clínico. As
-- outras colunas continuam mudando. A guarda é pela COLUNA, não pela tabela:
-- triages existe desde a primeira migração, e um replay do zero executa este
-- arquivo antes de a coluna existir.
CREATE OR REPLACE FUNCTION rota_triage_neighborhood_guard() RETURNS trigger AS $fn$
BEGIN
  IF NEW.neighborhood_id IS DISTINCT FROM OLD.neighborhood_id THEN
    IF NEW.neighborhood_id IS NULL AND NEW.status = 'aborted_by_revocation' THEN
      RETURN NEW;
    END IF;
    RAISE EXCEPTION 'triages: neighborhood_id never changes after insert (only to NULL on revocation)';
  END IF;
  RETURN NEW;
END;
$fn$ LANGUAGE plpgsql;

DO $do$
BEGIN
  IF EXISTS (SELECT 1 FROM information_schema.columns
             WHERE table_schema = 'public' AND table_name = 'triages' AND column_name = 'neighborhood_id') THEN
    EXECUTE 'DROP TRIGGER IF EXISTS triages_neighborhood_immutable ON triages';
    EXECUTE 'CREATE TRIGGER triages_neighborhood_immutable
      BEFORE UPDATE ON triages
      FOR EACH ROW EXECUTE FUNCTION rota_triage_neighborhood_guard()';
  END IF;
END
$do$;
```

- [ ] **Step 7: Escreva os modelos mínimos e as associações**

```ruby
# app/models/neighborhood.rb
# Bairro da cidade (ADR 0023). Nunca se apaga: desativa. Validações e escrita
# pelos comandos de app/commands/territory.
class Neighborhood < ApplicationRecord
  has_many :coverages, class_name: "NeighborhoodCoverage"
  has_many :health_units, through: :coverages
end
```

```ruby
# app/models/neighborhood_coverage.rb
# Par (bairro, unidade) da cobertura (ADR 0023): ligar cria a linha, desligar
# apaga. A trilha é o evento neighborhood.coverage_changed.
class NeighborhoodCoverage < ApplicationRecord
  belongs_to :neighborhood
  belongs_to :health_unit
end
```

Em `app/models/health_unit.rb`, logo depois de `has_many :attendances, dependent: :restrict_with_error`:

```ruby
  # ADR 0023: onde a unidade FICA (pode não estar entre os bairros que atende).
  belongs_to :neighborhood, optional: true
```

Em `app/models/citizen.rb`, logo depois de `has_many :verification_codes, ...`:

```ruby
  # ADR 0023: bairro declarado; nil = "prefiro não informar".
  belongs_to :neighborhood, optional: true
```

Em `app/models/triage.rb`, logo depois de `belongs_to :protocol_definition`:

```ruby
  # ADR 0023: cópia do bairro do cidadão na criação; imutável (trigger).
  belongs_to :neighborhood, optional: true
```

- [ ] **Step 8: Suba a guarda de ADR para 0023**

Em `spec/adr_pointers_spec.rb`:
- no comentário do topo, `O v2 vai de 0001 a 0022 (0020 banco por cidade, 0021 profissionais, 0022 painéis ao vivo)` passa a `O v2 vai de 0001 a 0023 (0020 banco por cidade, 0021 profissionais, 0022 painéis ao vivo, 0023 território)`;
- `VALID_RANGE = (1..22).freeze` passa a `VALID_RANGE = (1..23).freeze`;
- `it "only points at ADRs that exist in the v2 corpus (0001..0022)"` passa a `(0001..0023)`.

- [ ] **Step 9: Migre os bancos de dev e faça o dump à mão**

Run:
```bash
docker compose exec -T -w /rails/.claude/mod11 api bin/rails "city:migrate[curitiba]"
docker compose exec -T -w /rails/.claude/mod11 api bin/rails "city:migrate[maringa]"
```
Expected: `[city:migrate] curitiba → 20260928100001` (e maringa).

Em `db/city_schema.rb`:
- `define(version: 2026_09_28_000001)` passa a `define(version: 2026_09_28_100001)`;
- em `create_table "citizens"`, a coluna `t.uuid "neighborhood_id"` entre `created_at` e `phone`, e `t.index ["neighborhood_id"], name: "index_citizens_on_neighborhood_id"` entre os índices (ordem alfabética do nome do índice);
- em `create_table "health_units"`, as colunas em ordem alfabética (`active`, `address_complement` com `limit: 80`, `address_number` com `limit: 20`, `address_street` com `limit: 160`, `address_zip` com `limit: 8`, `created_at`, `kind`, `name`, `neighborhood_id`, `updated_at`), o índice `index_health_units_on_neighborhood_id` e o `t.check_constraint` de `ck_health_units_address_zip`;
- em `create_table "triages"`, `t.uuid "neighborhood_id"` entre `current_step` e `outcome`, e `t.index ["neighborhood_id", "created_at"], name: "idx_triages_neighborhood_created"`;
- as duas tabelas novas entre `memberships` e `otp_challenges` (`neighborhood_coverages` antes de `neighborhoods`):

```ruby
  create_table "neighborhood_coverages", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.datetime "created_at", null: false
    t.uuid "health_unit_id", null: false
    t.uuid "neighborhood_id", null: false
    t.index ["health_unit_id"], name: "index_neighborhood_coverages_on_health_unit_id"
    t.index ["neighborhood_id", "health_unit_id"], name: "idx_neighborhood_coverages_pair", unique: true
  end

  create_table "neighborhoods", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.boolean "active", default: true, null: false
    t.datetime "created_at", null: false
    t.string "name", limit: 120, null: false
    t.string "seed_key"
    t.string "source", null: false
    t.datetime "updated_at", null: false
    t.index "lower((name)::text)", name: "idx_neighborhoods_name_ci", unique: true
    t.index ["seed_key"], name: "idx_neighborhoods_seed_key", unique: true, where: "(seed_key IS NOT NULL)"
    t.check_constraint "length(btrim(name::text)) > 0 AND name::text = btrim(name::text)", name: "ck_neighborhoods_name"
    t.check_constraint "source::text = ANY (ARRAY['seed'::text, 'manual'::text])", name: "ck_neighborhoods_source"
  end
```

- os `add_foreign_key` novos na lista final, em ordem alfabética: `add_foreign_key "citizens", "neighborhoods"`, `add_foreign_key "health_units", "neighborhoods"`, `add_foreign_key "neighborhood_coverages", "health_units"`, `add_foreign_key "neighborhood_coverages", "neighborhoods"`, `add_foreign_key "triages", "neighborhoods"`.

O texto exato de cada CHECK vem do banco (substitua o que estiver acima se diferir):

```bash
docker compose exec -T db psql -U postgres -d <banco_de_curitiba> -c "select conname, pg_get_constraintdef(oid) from pg_constraint where conname in ('ck_neighborhoods_name','ck_neighborhoods_source','ck_health_units_address_zip')"
```

(O nome do banco de Curitiba está em `bin/rails runner 'puts City.find_by(slug: "curitiba").database_url'` — só o nome do banco, não cole a URL em lugar nenhum.)

- [ ] **Step 10: Recarregue os bancos de teste e rode a paridade, a guarda de ADR e a spec de guarda**

Run:
```bash
docker compose exec -T -w /rails/.claude/mod11 api bin/rails city:test_databases
docker compose exec -T -w /rails/.claude/mod11 api bundle exec rspec spec/services/city_schema_spec.rb spec/adr_pointers_spec.rb spec/models/territory_tables_guard_spec.rb
```
Expected: PASS.

- [ ] **Step 11: Commit**

```bash
/opt/homebrew/bin/git add db/city_migrate/20260928100001_create_territory.rb db/city_triggers.sql db/city_schema.rb app/models/neighborhood.rb app/models/neighborhood_coverage.rb app/models/health_unit.rb app/models/citizen.rb app/models/triage.rb spec/support/territory_helpers.rb spec/rails_helper.rb spec/adr_pointers_spec.rb spec/models/territory_tables_guard_spec.rb
/opt/homebrew/bin/git commit -m "feat: add neighborhoods, coverage and immutable triage neighborhood copy

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```


### Task 2: Modelo do bairro, comandos do território e eventos

**Files:**
- Modify: `app/models/neighborhood.rb`
- Create: `app/commands/territory/create_neighborhood.rb`, `rename_neighborhood.rb`, `set_neighborhood_active.rb`, `replace_coverage.rb`
- Modify: `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`
- Test: `spec/models/neighborhood_spec.rb`, `spec/commands/territory/neighborhood_commands_spec.rb`, `spec/commands/territory/replace_coverage_spec.rb`

**Interfaces:**
- Consumes: modelos da Task 1.
- Produces:
  - `Neighborhood::SOURCES`, `Neighborhood::NAME_MAX` (120), `Neighborhood.active_neighborhoods`, `Neighborhood.named(name)` (compara por `lower`, com `squish`);
  - `Territory::CreateNeighborhood.call(name:, by:, source: "manual", seed_key: nil) → Result` (ok: `neighborhood:`; fail: `:blank_name`, `:name_taken`; `seed_key` só a semente passa);
  - `Territory::RenameNeighborhood.call(neighborhood:, name:, by:) → Result` (fail: `:blank_name`, `:name_taken`);
  - `Territory::SetNeighborhoodActive.call(neighborhood:, active:, by:) → Result`;
  - `Territory::ReplaceCoverage.call(neighborhood:, health_unit_ids:, by:) → Result` (ok: `neighborhood:`, `added:`, `removed:`; fail: `:inactive_neighborhood`, `:inactive_unit`);
  - `by:` pode ser `nil` (semente); o evento leva `by_user_id: nil`.
  - Eventos: `neighborhood.created` `{neighborhood_id, source, by_user_id}`, `neighborhood.renamed` / `.deactivated` / `.activated` `{neighborhood_id, by_user_id}`, `neighborhood.coverage_changed` `{neighborhood_id, added_unit_ids, removed_unit_ids, by_user_id}`, `citizen.neighborhood_changed` `{citizen_id, from_id, to_id}` (publicado na Task 7, declarado aqui).

- [ ] **Step 1: Escreva a spec do modelo**

```ruby
# spec/models/neighborhood_spec.rb
require "rails_helper"

RSpec.describe Neighborhood do
  it "normaliza o nome (pontas e espaços repetidos)" do
    expect(described_class.new(name: "  Santa   Felicidade ", source: "seed").name).to eq("Santa Felicidade")
  end

  it "recusa nome vazio, acima de 120 e origem desconhecida" do
    expect(described_class.new(name: "  ", source: "seed")).not_to be_valid
    expect(described_class.new(name: "x" * 121, source: "seed")).not_to be_valid
    expect(described_class.new(name: "Batel", source: "import")).not_to be_valid
  end

  it "named compara sem diferenciar maiúsculas e sem espaços extras" do
    batel = described_class.create!(name: "Batel", source: "seed")
    expect(described_class.named("  BATEL ")).to eq([ batel ])
  end

  it "active_neighborhoods deixa os inativos de fora" do
    ativo = described_class.create!(name: "Centro", source: "seed")
    described_class.create!(name: "Ahu", source: "seed", active: false)
    expect(described_class.active_neighborhoods).to eq([ ativo ])
  end
end
```

- [ ] **Step 2: Escreva as specs dos comandos de bairro**

```ruby
# spec/commands/territory/neighborhood_commands_spec.rb
require "rails_helper"

RSpec.describe "Comandos de bairro" do
  let(:admin) { staff_with("admin@cidade.gov.br", "municipal_admin") }

  def payloads(name) = DomainEvent.where(name: name).map(&:payload)

  describe Territory::CreateNeighborhood do
    it "cria com source manual, normaliza o nome e publica só ids" do
      result = described_class.call(name: "  Santa   Felicidade ", by: admin)
      expect(result).to be_ok
      neighborhood = result.payload[:neighborhood]
      expect(neighborhood).to have_attributes(name: "Santa Felicidade", source: "manual", active: true)
      expect(payloads("neighborhood.created"))
        .to eq([ { "neighborhood_id" => neighborhood.id, "source" => "manual", "by_user_id" => admin.id } ])
    end

    it "semente: source seed, chave da semente e sem autor" do
      neighborhood = described_class.call(name: "Batel", by: nil, source: "seed", seed_key: "batel").payload[:neighborhood]
      expect(neighborhood).to have_attributes(source: "seed", seed_key: "batel")
      expect(payloads("neighborhood.created").sole["by_user_id"]).to be_nil
    end

    it "nome vazio, nulo ou acima de 120: blank_name, nada criado" do
      [ "   ", nil, "x" * 121 ].each do |name|
        expect(described_class.call(name: name, by: admin).reason).to eq(:blank_name), name.inspect
      end
      expect(Neighborhood.count).to eq(0)
    end

    it "nome repetido em outra caixa: name_taken" do
      described_class.call(name: "Batel", by: admin)
      expect(described_class.call(name: "  batel", by: admin).reason).to eq(:name_taken)
      expect(Neighborhood.count).to eq(1)
    end
  end

  describe Territory::RenameNeighborhood do
    let!(:batel) { Territory::CreateNeighborhood.call(name: "Batel", by: admin).payload[:neighborhood] }

    it "renomeia e publica neighborhood.renamed sem o nome" do
      expect(described_class.call(neighborhood: batel, name: "Batel Soho", by: admin)).to be_ok
      expect(batel.reload.name).to eq("Batel Soho")
      expect(payloads("neighborhood.renamed")).to eq([ { "neighborhood_id" => batel.id, "by_user_id" => admin.id } ])
    end

    it "renomear mantém a chave da semente" do
      seeded = Territory::CreateNeighborhood.call(name: "Ahu", by: nil, source: "seed", seed_key: "ahu").payload[:neighborhood]
      described_class.call(neighborhood: seeded, name: "Ahú de Baixo", by: admin)
      expect(seeded.reload.seed_key).to eq("ahu")
    end

    it "só a caixa muda: aceito (não é name_taken)" do
      expect(described_class.call(neighborhood: batel, name: "BATEL", by: admin)).to be_ok
      expect(batel.reload.name).to eq("BATEL")
    end

    it "mesmo nome: ok, sem evento" do
      described_class.call(neighborhood: batel, name: " Batel ", by: admin)
      expect(payloads("neighborhood.renamed")).to be_empty
    end

    it "nome de outro bairro: name_taken e nada muda; vazio: blank_name" do
      Territory::CreateNeighborhood.call(name: "Centro", by: admin)
      expect(described_class.call(neighborhood: batel, name: "centro", by: admin).reason).to eq(:name_taken)
      expect(described_class.call(neighborhood: batel, name: "", by: admin).reason).to eq(:blank_name)
      expect(batel.reload.name).to eq("Batel")
    end
  end

  describe Territory::SetNeighborhoodActive do
    let!(:batel) { Territory::CreateNeighborhood.call(name: "Batel", by: admin).payload[:neighborhood] }

    it "desativa e reativa, com um evento cada; repetir não publica" do
      described_class.call(neighborhood: batel, active: false, by: admin)
      described_class.call(neighborhood: batel, active: false, by: admin)
      expect(batel.reload.active).to be(false)
      described_class.call(neighborhood: batel, active: true, by: admin)
      expect(batel.reload.active).to be(true)
      expect(payloads("neighborhood.deactivated")).to eq([ { "neighborhood_id" => batel.id, "by_user_id" => admin.id } ])
      expect(payloads("neighborhood.activated")).to eq([ { "neighborhood_id" => batel.id, "by_user_id" => admin.id } ])
    end
  end
end
```

```ruby
# spec/commands/territory/replace_coverage_spec.rb
require "rails_helper"

RSpec.describe Territory::ReplaceCoverage do
  let(:admin) { staff_with("admin@cidade.gov.br", "municipal_admin") }
  let(:centro) { Neighborhood.create!(name: "Centro", source: "manual") }
  let(:ubs) { create_unit("UBS Centro") }
  let(:upa) { create_unit("UPA Norte", kind: "upa") }
  let(:ubs_sul) { create_unit("UBS Sul") }

  def payloads = DomainEvent.where(name: "neighborhood.coverage_changed").map(&:payload)

  it "substitui o conjunto e publica adicionados e removidos, só ids" do
    described_class.call(neighborhood: centro, health_unit_ids: [ ubs.id, upa.id ], by: admin)
    result = described_class.call(neighborhood: centro, health_unit_ids: [ upa.id, ubs_sul.id ], by: admin)

    expect(result).to be_ok
    expect(result.payload).to include(added: [ ubs_sul.id ], removed: [ ubs.id ])
    expect(centro.coverages.pluck(:health_unit_id)).to contain_exactly(upa.id, ubs_sul.id)
    expect(payloads).to include(
      "neighborhood_id" => centro.id, "added_unit_ids" => [ ubs_sul.id ], "removed_unit_ids" => [ ubs.id ],
      "by_user_id" => admin.id
    )
  end

  it "mesmo conjunto (em outra ordem, com repetição): ok, sem segundo evento" do
    described_class.call(neighborhood: centro, health_unit_ids: [ ubs.id, upa.id ], by: admin)
    described_class.call(neighborhood: centro, health_unit_ids: [ upa.id, ubs.id, upa.id ], by: admin)
    expect(payloads.size).to eq(1)
  end

  it "lista vazia remove tudo" do
    described_class.call(neighborhood: centro, health_unit_ids: [ ubs.id ], by: admin)
    expect(described_class.call(neighborhood: centro, health_unit_ids: [], by: admin)).to be_ok
    expect(centro.coverages).to be_empty
  end

  it "unidade inativa, inexistente ou id que não é UUID: inactive_unit e nada muda" do
    described_class.call(neighborhood: centro, health_unit_ids: [ ubs.id ], by: admin)
    inativa = create_unit("UBS Fechada", active: false)
    [ [ inativa.id ], [ SecureRandom.uuid ], [ "nao-e-uuid" ], [ ubs.id, inativa.id ] ].each do |ids|
      expect(described_class.call(neighborhood: centro, health_unit_ids: ids, by: admin).reason)
        .to eq(:inactive_unit), ids.inspect
    end
    expect(centro.coverages.pluck(:health_unit_id)).to eq([ ubs.id ])
  end

  it "bairro inativo: inactive_neighborhood" do
    centro.update!(active: false)
    expect(described_class.call(neighborhood: centro, health_unit_ids: [ ubs.id ], by: admin).reason)
      .to eq(:inactive_neighborhood)
    expect(centro.coverages).to be_empty
  end
end
```

- [ ] **Step 3: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod11 api bundle exec rspec spec/models/neighborhood_spec.rb spec/commands/territory`
Expected: FAIL (`uninitialized constant Territory::CreateNeighborhood`, validações ausentes).

- [ ] **Step 4: Complete o modelo**

```ruby
# app/models/neighborhood.rb
# Bairro da cidade (ADR 0023): lista curada por cidade (semente) e editada
# pelo municipal_admin. Nunca se apaga: desativa. Inativo não entra em escolha
# nova (cidadão, cobertura), mas continua no histórico e no filtro dos
# painéis. Unicidade sem diferenciar maiúsculas: índice idx_neighborhoods_name_ci.
class Neighborhood < ApplicationRecord
  SOURCES = %w[seed manual].freeze
  NAME_MAX = 120

  has_many :coverages, class_name: "NeighborhoodCoverage"
  has_many :health_units, through: :coverages

  normalizes :name, with: ->(v) { v.to_s.squish }

  validates :name, presence: true, length: { maximum: NAME_MAX }
  validates :source, inclusion: { in: SOURCES }

  scope :active_neighborhoods, -> { where(active: true) }

  # Mesmo critério do índice único.
  def self.named(name)
    where("lower(name) = lower(?)", name.to_s.squish)
  end
end
```

- [ ] **Step 5: Implemente os comandos**

```ruby
# app/commands/territory/create_neighborhood.rb
# Cria um bairro (ADR 0023; spec 2026-09-28-module-11-territory §4.2). O
# municipal_admin cria com source manual e sem seed_key; a semente
# (Territory::Seed), com seed, a chave estável do YAML e sem autor.
# Reasons: :blank_name (vazio ou acima de 120), :name_taken.
module Territory
  class CreateNeighborhood
    def self.call(name:, by:, source: "manual", seed_key: nil)
      neighborhood = Neighborhood.new(name: name, source: source, seed_key: seed_key)
      return Result.fail(:blank_name) if neighborhood.name.blank? || neighborhood.name.length > Neighborhood::NAME_MAX
      return Result.fail(:name_taken) if Neighborhood.named(neighborhood.name).exists?

      ApplicationRecord.transaction do
        neighborhood.save!
        DomainEvents.publish("neighborhood.created", neighborhood_id: neighborhood.id, source: neighborhood.source,
                                                     by_user_id: by&.id)
      end
      Result.ok(neighborhood: neighborhood)
    rescue ActiveRecord::RecordNotUnique
      Result.fail(:name_taken)
    end
  end
end
```

```ruby
# app/commands/territory/rename_neighborhood.rb
# Renomeia um bairro (ADR 0023). Só mudar a caixa é aceito; o evento não leva
# o nome (só ids). Reasons: :blank_name, :name_taken.
module Territory
  class RenameNeighborhood
    def self.call(neighborhood:, name:, by:)
      new_name = name.to_s.squish
      return Result.fail(:blank_name) if new_name.empty? || new_name.length > Neighborhood::NAME_MAX
      return Result.ok(neighborhood: neighborhood) if new_name == neighborhood.name
      return Result.fail(:name_taken) if Neighborhood.named(new_name).where.not(id: neighborhood.id).exists?

      ApplicationRecord.transaction do
        neighborhood.update!(name: new_name)
        DomainEvents.publish("neighborhood.renamed", neighborhood_id: neighborhood.id, by_user_id: by&.id)
      end
      Result.ok(neighborhood: neighborhood)
    rescue ActiveRecord::RecordNotUnique
      neighborhood.restore_attributes
      Result.fail(:name_taken)
    end
  end
end
```

```ruby
# app/commands/territory/set_neighborhood_active.rb
# Desativa ou reativa um bairro (ADR 0023). Desativar não apaga cobertura nem
# o bairro de quem já o declarou: só tira o bairro das escolhas novas.
module Territory
  class SetNeighborhoodActive
    def self.call(neighborhood:, active:, by:)
      return Result.ok(neighborhood: neighborhood) if neighborhood.active == active

      ApplicationRecord.transaction do
        neighborhood.update!(active: active)
        DomainEvents.publish(active ? "neighborhood.activated" : "neighborhood.deactivated",
                             neighborhood_id: neighborhood.id, by_user_id: by&.id)
      end
      Result.ok(neighborhood: neighborhood)
    end
  end
end
```

```ruby
# app/commands/territory/replace_coverage.rb
# Substitui o conjunto de unidades que cobrem o bairro (ADR 0023; spec §4.2).
# Trava o bairro (FOR UPDATE): duas edições simultâneas não calculam
# adicionados/removidos sobre o mesmo estado velho. Toda unidade do conjunto
# novo precisa existir e estar ativa. Reasons: :inactive_neighborhood,
# :inactive_unit.
module Territory
  class ReplaceCoverage
    def self.call(neighborhood:, health_unit_ids:, by:)
      ids = Array(health_unit_ids).map(&:to_s).uniq
      result = nil
      ApplicationRecord.transaction do
        locked = Neighborhood.lock("FOR UPDATE").find(neighborhood.id)
        result = replace(locked, ids, by)
        raise ActiveRecord::Rollback if result.failure?
      end
      result
    end

    def self.replace(neighborhood, ids, by)
      return Result.fail(:inactive_neighborhood) unless neighborhood.active?
      return Result.fail(:inactive_unit) unless HealthUnit.where(id: ids, active: true).count == ids.size

      current = neighborhood.coverages.pluck(:health_unit_id)
      added = (ids - current).sort
      removed = (current - ids).sort
      neighborhood.coverages.where(health_unit_id: removed).delete_all if removed.any?
      added.each { |id| neighborhood.coverages.create!(health_unit_id: id) }
      if added.any? || removed.any?
        DomainEvents.publish("neighborhood.coverage_changed", neighborhood_id: neighborhood.id,
                                                              added_unit_ids: added, removed_unit_ids: removed,
                                                              by_user_id: by&.id)
      end
      Result.ok(neighborhood: neighborhood, added: added, removed: removed)
    end
    private_class_method :replace
  end
end
```

Nota: `HealthUnit.where(id: ["nao-e-uuid"])` converte o texto inválido para `NULL` (tipo uuid do Rails) e conta 0 — vira `inactive_unit`, sem 500.

- [ ] **Step 6: Declare os eventos do módulo e prove a declaração**

Ao fim do bloco de binds em `config/initializers/domain_events.rb`, antes do `end`:

```ruby
  # Território (ADR 0023; spec 2026-09-28 §3.5): trilha, só ids; sem
  # consumidor, de propósito.
  DomainEvents.bind "neighborhood.created", to: []
  DomainEvents.bind "neighborhood.renamed", to: []
  DomainEvents.bind "neighborhood.deactivated", to: []
  DomainEvents.bind "neighborhood.activated", to: []
  DomainEvents.bind "neighborhood.coverage_changed", to: []
  DomainEvents.bind "citizen.neighborhood_changed", to: []
```

Ao fim de `spec/initializers/domain_events_bindings_spec.rb`:

```ruby
# Módulo 11 (ADR 0023): eventos do território declarados, só trilha.
RSpec.describe "territory event bindings (ADR 0023)" do
  it "declares every territory event with no consumer" do
    names = %w[neighborhood.created neighborhood.renamed neighborhood.deactivated neighborhood.activated
               neighborhood.coverage_changed citizen.neighborhood_changed]
    expect(DomainEvents.registry.keys).to include(*names)
    expect(names.flat_map { |n| DomainEvents.registry[n] }).to be_empty
  end
end
```

- [ ] **Step 7: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod11 api bundle exec rspec spec/models/neighborhood_spec.rb spec/commands/territory spec/initializers/domain_events_bindings_spec.rb`
Expected: PASS.

- [ ] **Step 8: Commit**

```bash
/opt/homebrew/bin/git add app/models/neighborhood.rb app/commands/territory config/initializers/domain_events.rb spec/models/neighborhood_spec.rb spec/commands/territory spec/initializers/domain_events_bindings_spec.rb
/opt/homebrew/bin/git commit -m "feat: create, rename, toggle and cover neighborhoods with id-only events

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 3: Rotas `/territory`

**Files:**
- Create: `app/controllers/territory_controller.rb`
- Modify: `config/routes.rb`
- Test: `spec/requests/territory_spec.rb`

**Interfaces:**
- Consumes: comandos da Task 2; `AttendanceAccess#forbid`, `#render_failure`.
- Produces: `GET /territory/neighborhoods` → `{ neighborhoods: [...] }`; `POST /territory/neighborhoods` (201), `POST /territory/neighborhoods/:id`, `/:id/deactivate`, `/:id/activate`, `/:id/coverage` → `{ neighborhood: { id, name, source, active, units: [{ id, name, active }] } }` (unidades por nome). Recusas: 403 `missing_role`; 422 `blank_name`, `name_taken`, `inactive_unit`, `inactive_neighborhood`; 404 `not_found`.

- [ ] **Step 1: Escreva a spec de request**

```ruby
# spec/requests/territory_spec.rb
require "rails_helper"

RSpec.describe "Território", type: :request do
  def body = JSON.parse(response.body)
  def status_and_error = [ response.status, body["error"] ]

  let(:admin) { staff_with("admin@cidade.gov.br", "municipal_admin") }
  let!(:ubs) { create_unit("UBS Centro") }
  let!(:upa) { create_unit("UPA Norte", kind: "upa") }

  describe "com municipal_admin" do
    before { sign_in_as(admin) }

    it "cria, renomeia, cobre, desativa, reativa e lista com origem, estado e unidades" do
      json_post "/territory/neighborhoods", name: "  Santa Felicidade "
      expect(response).to have_http_status(:created)
      id = body.dig("neighborhood", "id")
      expect(body["neighborhood"])
        .to eq("id" => id, "name" => "Santa Felicidade", "source" => "manual", "active" => true, "units" => [])

      json_post "/territory/neighborhoods/#{id}", name: "Santa Felicidade Velha"
      expect(response).to have_http_status(:ok)
      expect(body.dig("neighborhood", "name")).to eq("Santa Felicidade Velha")

      json_post "/territory/neighborhoods/#{id}/coverage", health_unit_ids: [ upa.id, ubs.id ]
      expect(body.dig("neighborhood", "units")).to eq([
        { "id" => ubs.id, "name" => "UBS Centro", "active" => true },
        { "id" => upa.id, "name" => "UPA Norte", "active" => true }
      ])

      json_post "/territory/neighborhoods/#{id}/deactivate"
      expect(body.dig("neighborhood", "active")).to be(false)
      json_post "/territory/neighborhoods/#{id}/activate"
      expect(body.dig("neighborhood", "active")).to be(true)

      Neighborhood.create!(name: "Batel", source: "seed", active: false)
      get "/territory/neighborhoods"
      expect(body["neighborhoods"].map { |n| [ n["name"], n["source"], n["active"], n["units"].size ] })
        .to eq([ [ "Batel", "seed", false, 0 ], [ "Santa Felicidade Velha", "manual", true, 2 ] ])
    end

    it "unidade desativada depois de coberta continua listada, com active false" do
      centro = Neighborhood.create!(name: "Centro", source: "manual")
      NeighborhoodCoverage.create!(neighborhood: centro, health_unit: ubs)
      ubs.update!(active: false)
      get "/territory/neighborhoods"
      expect(body["neighborhoods"].sole["units"]).to eq([ { "id" => ubs.id, "name" => "UBS Centro", "active" => false } ])
    end

    describe "recusas" do
      let!(:centro) { Neighborhood.create!(name: "Centro", source: "manual") }

      it "nome vazio ou que não é texto: 422 blank_name" do
        json_post "/territory/neighborhoods", name: "  "
        expect(status_and_error).to eq([ 422, "blank_name" ])
        json_post "/territory/neighborhoods", name: { "x" => 1 }
        expect(status_and_error).to eq([ 422, "blank_name" ])
        json_post "/territory/neighborhoods/#{centro.id}", name: ""
        expect(status_and_error).to eq([ 422, "blank_name" ])
      end

      it "nome repetido em outra caixa: 422 name_taken" do
        json_post "/territory/neighborhoods", name: "CENTRO"
        expect(status_and_error).to eq([ 422, "name_taken" ])
      end

      it "cobertura com unidade inativa, inexistente, ou corpo sem lista: 422 inactive_unit e nada muda" do
        ubs.update!(active: false)
        [ { health_unit_ids: [ ubs.id ] }, { health_unit_ids: [ SecureRandom.uuid ] },
          { health_unit_ids: "x" }, { health_unit_ids: [ { "id" => upa.id } ] }, {} ].each do |payload|
          json_post "/territory/neighborhoods/#{centro.id}/coverage", payload
          expect(status_and_error).to eq([ 422, "inactive_unit" ]), payload.inspect
        end
        expect(centro.coverages).to be_empty
      end

      it "cobertura em bairro inativo: 422 inactive_neighborhood" do
        centro.update!(active: false)
        json_post "/territory/neighborhoods/#{centro.id}/coverage", health_unit_ids: [ upa.id ]
        expect(status_and_error).to eq([ 422, "inactive_neighborhood" ])
      end

      it "bairro inexistente ou id que não é UUID: 404 not_found" do
        [ SecureRandom.uuid, "nao-e-uuid" ].each do |id|
          json_post "/territory/neighborhoods/#{id}/deactivate"
          expect(status_and_error).to eq([ 404, "not_found" ])
        end
      end
    end
  end

  describe "só o municipal_admin" do
    (Membership::ROLES - %w[municipal_admin]).each do |role|
      it "#{role}: 403 missing_role em todas, nada muda" do
        centro = Neighborhood.create!(name: "Centro", source: "manual")
        sign_in_as(staff_with("#{role}@cidade.gov.br", role))

        get "/territory/neighborhoods"
        expect(status_and_error).to eq([ 403, "missing_role" ])
        [
          [ "/territory/neighborhoods", { name: "Novo" } ],
          [ "/territory/neighborhoods/#{centro.id}", { name: "Outro" } ],
          [ "/territory/neighborhoods/#{centro.id}/deactivate", {} ],
          [ "/territory/neighborhoods/#{centro.id}/activate", {} ],
          [ "/territory/neighborhoods/#{centro.id}/coverage", { health_unit_ids: [ ubs.id ] } ]
        ].each do |path, params|
          json_post path, params
          expect(status_and_error).to eq([ 403, "missing_role" ]), path
        end
        expect(centro.reload).to have_attributes(name: "Centro", active: true)
        expect(Neighborhood.count).to eq(1)
        expect(NeighborhoodCoverage.count).to eq(0)
      end
    end
  end

  it "sem sessão: 401" do
    get "/territory/neighborhoods"
    expect(response).to have_http_status(:unauthorized)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod11 api bundle exec rspec spec/requests/territory_spec.rb`
Expected: FAIL (rota inexistente: 404 em vez dos códigos esperados).

- [ ] **Step 3: Implemente o controller**

```ruby
# app/controllers/territory_controller.rb
# Território da cidade (ADR 0023; spec 2026-09-28-module-11-territory §4.1):
# bairros e cobertura, só municipal_admin, sem step-up (nada disso autoriza
# ato clínico nem expõe dado de cidadão). Prefixo único /territory: uma
# entrada só no proxy de dev do dashboard.
class TerritoryController < ApplicationController
  include Authentication
  include AttendanceAccess

  wrap_parameters false

  ERROR_STATUS = {
    blank_name: :unprocessable_entity, name_taken: :unprocessable_entity, inactive_unit: :unprocessable_entity,
    inactive_neighborhood: :unprocessable_entity, not_found: :not_found
  }.freeze

  before_action :require_territory_admin
  before_action :set_neighborhood, except: %i[index create]

  def index
    neighborhoods = Neighborhood.includes(:health_units).order(:name)
    render json: { neighborhoods: neighborhoods.map { |n| neighborhood_json(n) } }
  end

  def create
    return refuse(:blank_name) unless params[:name].is_a?(String)

    respond(Territory::CreateNeighborhood.call(name: params[:name], by: Current.user), status: :created)
  end

  def update
    return refuse(:blank_name) unless params[:name].is_a?(String)

    respond(Territory::RenameNeighborhood.call(neighborhood: @neighborhood, name: params[:name], by: Current.user))
  end

  def deactivate
    respond(Territory::SetNeighborhoodActive.call(neighborhood: @neighborhood, active: false, by: Current.user))
  end

  def activate
    respond(Territory::SetNeighborhoodActive.call(neighborhood: @neighborhood, active: true, by: Current.user))
  end

  def coverage
    ids = params[:health_unit_ids]
    return refuse(:inactive_unit) unless ids.is_a?(Array) && ids.all?(String)

    respond(Territory::ReplaceCoverage.call(neighborhood: @neighborhood, health_unit_ids: ids, by: Current.user))
  end

  private

  def require_territory_admin
    forbid("missing_role") unless CitizenVerificationPolicy.new(Current.user, nil).manage?
  end

  def set_neighborhood
    @neighborhood = Neighborhood.find_by(id: params[:id])
    refuse(:not_found) unless @neighborhood
  end

  def refuse(reason)
    render json: { error: reason.to_s }, status: ERROR_STATUS.fetch(reason)
  end

  def respond(result, status: :ok)
    return render_failure(result, ERROR_STATUS) if result.failure?

    neighborhood = Neighborhood.includes(:health_units).find(result.payload[:neighborhood].id)
    render json: { neighborhood: neighborhood_json(neighborhood) }, status: status
  end

  def neighborhood_json(neighborhood)
    {
      id: neighborhood.id, name: neighborhood.name, source: neighborhood.source, active: neighborhood.active,
      units: neighborhood.health_units.sort_by(&:name).map { |u| { id: u.id, name: u.name, active: u.active } }
    }
  end
end
```

Nota: `render_failure` só usa `result.reason` e `result.details` (vazio nos comandos do território); `added`/`removed` do payload não saem no JSON.

- [ ] **Step 4: Rotas**

Em `config/routes.rb`, logo depois do `scope "/professionals" do ... end`:

```ruby
  # Território (ADR 0023; spec 2026-09-28-module-11-territory §4.1). Prefixo
  # único: uma entrada só no proxy de dev do dashboard.
  scope "/territory" do
    get  "neighborhoods",                to: "territory#index"
    post "neighborhoods",                to: "territory#create"
    post "neighborhoods/:id",            to: "territory#update"
    post "neighborhoods/:id/deactivate", to: "territory#deactivate"
    post "neighborhoods/:id/activate",   to: "territory#activate"
    post "neighborhoods/:id/coverage",   to: "territory#coverage"
  end
```

- [ ] **Step 5: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod11 api bundle exec rspec spec/requests/territory_spec.rb`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git add app/controllers/territory_controller.rb config/routes.rb spec/requests/territory_spec.rb
/opt/homebrew/bin/git commit -m "feat: add territory routes for municipal admins

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 4: Semente por cidade e rake `city:territory:seed`

**Files:**
- Create: `db/seeds/territory/curitiba.yml`, `db/seeds/territory/maringa.yml`
- Create: `app/services/territory/seed.rb`, `lib/tasks/territory.rake`
- Test: `spec/services/territory/seed_spec.rb`, `spec/services/territory/seed_files_spec.rb`, `spec/tasks/territory_rake_spec.rb`

**Interfaces:**
- Consumes: `Territory::CreateNeighborhood`, `Territory::ReplaceCoverage` (Task 2).
- Produces: `Territory::Seed.path_for(slug) → Pathname`; `Territory::Seed.call(path:) → Territory::Seed::Report` (`created`, `existing` Integer; `warnings` Array<String>); rakes `city:territory:seed[slug]` e `city:territory:seed:all`. Cada item do YAML: `name`, `key` (slug estável do nome original), `units` opcional.

- [ ] **Step 1: Escreva a spec do serviço**

```ruby
# spec/services/territory/seed_spec.rb
require "rails_helper"

RSpec.describe Territory::Seed do
  let(:dir) { Pathname(Dir.mktmpdir) }
  after { FileUtils.remove_entry(dir) }

  def run(yaml)
    path = dir.join("cidade.yml")
    path.write(yaml)
    described_class.call(path: path)
  end

  let!(:ubs) { create_unit("UBS Santa Felicidade") }
  let!(:fechada) { create_unit("UBS Fechada", active: false) }

  let(:yaml) do
    <<~YAML
      neighborhoods:
        - name: Santa Felicidade
          key: santa-felicidade
          units: ["ubs santa felicidade", "UBS Fechada", "UBS Inexistente"]
        - name: Batel
          key: batel
          units: []
        - name: Ahu
          key: ahu
    YAML
  end

  it "cria os bairros com source seed, a chave e a cobertura das unidades ativas achadas pelo nome" do
    report = run(yaml)

    expect([ report.created, report.existing ]).to eq([ 3, 0 ])
    expect(report.warnings).to contain_exactly(match(/UBS Fechada/), match(/UBS Inexistente/))
    santa = Neighborhood.named("Santa Felicidade").sole
    expect(santa).to have_attributes(source: "seed", active: true, seed_key: "santa-felicidade")
    expect(santa.health_units).to eq([ ubs ])
    expect(Neighborhood.named("Ahu").sole.coverages).to be_empty
  end

  it "rodar duas vezes não muda nada" do
    run(yaml)
    snapshot = -> { [ Neighborhood.order(:name).pluck(:name, :active, :source, :seed_key), NeighborhoodCoverage.count, DomainEvent.count ] }
    before = snapshot.call
    report = run(yaml)
    expect(snapshot.call).to eq(before)
    expect([ report.created, report.existing ]).to eq([ 0, 3 ])
  end

  it "não desfaz edição: renomeado não é recriado nem renomeado; desativado e cobertura editada ficam" do
    run(yaml)
    santa = Neighborhood.find_by!(seed_key: "santa-felicidade")
    Territory::RenameNeighborhood.call(neighborhood: santa, name: "Santa Felicidade Velha", by: nil)
    Territory::ReplaceCoverage.call(neighborhood: santa, health_unit_ids: [], by: nil)
    batel = Neighborhood.find_by!(seed_key: "batel")
    Territory::SetNeighborhoodActive.call(neighborhood: batel, active: false, by: nil)

    report = run(yaml)
    expect([ report.created, report.existing ]).to eq([ 0, 3 ])
    expect(Neighborhood.count).to eq(3)
    expect(santa.reload).to have_attributes(name: "Santa Felicidade Velha", seed_key: "santa-felicidade")
    expect(santa.coverages).to be_empty
    expect(batel.reload.active).to be(false)
  end

  it "bairro manual com o mesmo nome: aviso, sem duplicado, sem adotar a chave, sem cobertura" do
    manual = Neighborhood.create!(name: "SANTA FELICIDADE", source: "manual")
    report = run(yaml)
    expect(report.created).to eq(2)
    expect(report.warnings).to include(match(/Santa Felicidade.*já existe/))
    expect(Neighborhood.named("Santa Felicidade").count).to eq(1)
    expect(manual.reload.seed_key).to be_nil
    expect(manual.coverages).to be_empty
  end

  it "entrada sem nome ou sem key vira aviso; arquivo sem a chave neighborhoods não cria nada" do
    expect(run("neighborhoods:\n  - key: x\n  - name: Batel\n").warnings)
      .to contain_exactly(match(/sem nome/), match(/sem key/))
    expect(run("outra: 1\n").created).to eq(0)
    expect(Neighborhood.count).to eq(0)
  end
end
```

- [ ] **Step 2: Escreva a spec dos arquivos de semente**

```ruby
# spec/services/territory/seed_files_spec.rb
require "rails_helper"

# As sementes versionadas (ADR 0023): bairros reais, nomes que o banco aceita,
# e cobertura ligando as unidades da semente de dev (2 a 4 bairros cada).
RSpec.describe "Sementes de bairros (db/seeds/territory)" do
  files = Dir[Rails.root.join("db/seeds/territory/*.yml").to_s].sort

  it "existem para curitiba e maringa" do
    expect(files.map { |f| File.basename(f) }).to eq(%w[curitiba.yml maringa.yml])
  end

  files.each do |path|
    describe File.basename(path) do
      let(:entries) { YAML.safe_load_file(path).fetch("neighborhoods") }
      let(:names) { entries.map { |e| e.fetch("name") } }

      it "nomes são texto, únicos sem diferenciar maiúsculas, sem espaço extra, até 120" do
        expect(names).to all(be_a(String))
        expect(names.map(&:downcase).uniq.size).to eq(names.size)
        expect(names).to all(satisfy { |n| n == n.squish && n.length.between?(1, 120) })
      end

      it "key é slug estável, única" do
        keys = entries.map { |e| e.fetch("key") }
        expect(keys).to all(match(/\A[a-z0-9]+(-[a-z0-9]+)*\z/))
        expect(keys.uniq.size).to eq(keys.size)
      end

      it "units é lista de nomes, e cada unidade cobre de 2 a 4 bairros" do
        units = entries.map { |e| e.fetch("units", []) }
        expect(units).to all(be_an(Array))
        expect(units.flatten.tally.values).to all(be_between(2, 4))
      end
    end
  end

  it "curitiba tem os 75 bairros oficiais" do
    expect(YAML.safe_load_file(files.first).fetch("neighborhoods").size).to eq(75)
  end
end
```

- [ ] **Step 3: Escreva a spec da rake**

```ruby
# spec/tasks/territory_rake_spec.rb
require "rails_helper"
require "rake"

RSpec.describe "city:territory:seed rake task" do
  before(:all) do
    Rails.application.load_tasks unless Rake::Task.task_defined?("city:territory:seed")
  end

  before do
    Rake::Task["city:territory:seed"].reenable
    Rake::Task["city:territory:seed:all"].reenable
  end

  after { CityCatalog.reset_cache! }

  # A mesma linha de catálogo que os request specs usam (city_request_auth.rb):
  # CityConnection.with(city) cai na conexão de TEST_CITY_A da transação.
  let!(:city) do
    City.find_by(slug: TEST_CITY_A.slug) ||
      City.create!(slug: TEST_CITY_A.slug, name: TEST_CITY_A.name, status: "active",
                   database_url: TEST_CITY_A.database_url, encryption_key: TEST_CITY_A.encryption_key,
                   schema_version: CitySchema.expected_version.to_s)
  end
  let(:dir) { Pathname(Dir.mktmpdir) }
  after { FileUtils.remove_entry(dir) }

  def run(task, *args)
    out = StringIO.new
    original_stdout, $stdout = $stdout, out
    original_stderr, $stderr = $stderr, StringIO.new
    Rake::Task[task].invoke(*args)
    out.string
  ensure
    $stdout = original_stdout
    $stderr = original_stderr
  end

  it "carrega a semente da cidade, relata, e a segunda vez não cria nada" do
    path = dir.join("#{city.slug}.yml")
    path.write("neighborhoods:\n  - name: Centro\n    key: centro\n  - name: Batel\n    key: batel\n")
    allow(Territory::Seed).to receive(:path_for).with(city.slug).and_return(path)

    expect(run("city:territory:seed", city.slug)).to include("2 criados, 0 já existentes, 0 avisos")
    expect(Neighborhood.pluck(:name)).to contain_exactly("Centro", "Batel")

    Rake::Task["city:territory:seed"].reenable
    expect(run("city:territory:seed", city.slug)).to include("0 criados, 2 já existentes, 0 avisos")
  end

  it "cidade sem arquivo: mensagem e saída sem erro" do
    allow(Territory::Seed).to receive(:path_for).and_return(dir.join("nao-existe.yml"))
    expect(run("city:territory:seed", city.slug)).to include("sem semente")
  end

  it ":all passa por toda cidade active/suspended" do
    path = dir.join("todas.yml")
    path.write("neighborhoods:\n  - name: Centro\n    key: centro\n")
    allow(Territory::Seed).to receive(:path_for).and_return(path)
    allow(City).to receive(:where).and_call_original
    allow(City).to receive(:where).with(status: %w[active suspended]).and_return(City.where(id: city.id))

    expect(run("city:territory:seed:all")).to include("#{city.slug}: 1 criados")
  end

  it "slug inexistente: aborta" do
    expect { run("city:territory:seed", "nao-existe") }.to raise_error(SystemExit)
  end
end
```

- [ ] **Step 4: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod11 api bundle exec rspec spec/services/territory spec/tasks/territory_rake_spec.rb`
Expected: FAIL (`uninitialized constant Territory::Seed`, arquivos ausentes, task inexistente).

- [ ] **Step 5: Implemente o serviço**

```ruby
# app/services/territory/seed.rb
require "yaml"

# Carga dos bairros de uma cidade (ADR 0023; spec 2026-09-28 §3.4, com a
# chave de semente decidida em 2026-09-28). Casa cada item do YAML com a linha
# pela `key` (neighborhoods.seed_key), NUNCA pelo nome: um bairro renomeado no
# dashboard não é recriado nem renomeado. Cria só o que falta; nunca altera
# nem desativa o que existe, e só liga cobertura em bairro criado NESTA carga
# — cobertura editada no dashboard prevalece. Nome já usado por outro bairro
# (manual, sem chave) vira aviso: sem duplicado e sem adotar a chave. Unidade
# é achada pelo nome, sem diferenciar maiúsculas, entre as ativas; a que não
# for achada vira aviso. Roda na cidade da conexão corrente: quem chama
# escolhe (city:territory:seed usa CityConnection.with).
module Territory
  class Seed
    Report = Struct.new(:created, :existing, :warnings, keyword_init: true)

    def self.path_for(slug)
      Rails.root.join("db/seeds/territory/#{slug}.yml")
    end

    def self.call(path:)
      data = YAML.safe_load_file(path)
      entries = data.is_a?(Hash) ? Array(data["neighborhoods"]) : []
      report = Report.new(created: 0, existing: 0, warnings: [])
      entries.each { |entry| load_entry(entry, report) }
      report
    end

    def self.load_entry(entry, report)
      entry = {} unless entry.is_a?(Hash)
      name = entry["name"].to_s.squish
      key = entry["key"].to_s.strip
      return report.warnings << "entrada sem nome: #{entry.inspect}" if name.empty?
      return report.warnings << "bairro #{name}: entrada sem key" if key.empty?
      return report.existing += 1 if Neighborhood.exists?(seed_key: key)
      if Neighborhood.named(name).exists?
        return report.warnings << "bairro #{name}: já existe um bairro com esse nome fora da semente (key #{key}) — pulado"
      end

      created = CreateNeighborhood.call(name: name, by: nil, source: "seed", seed_key: key)
      return report.warnings << "bairro #{name}: #{created.reason}" if created.failure?

      report.created += 1
      unit_ids = Array(entry["units"]).filter_map { |unit_name| active_unit_id(unit_name, name, report) }
      return if unit_ids.empty?

      covered = ReplaceCoverage.call(neighborhood: created.payload[:neighborhood], health_unit_ids: unit_ids, by: nil)
      report.warnings << "cobertura de #{name}: #{covered.reason}" if covered.failure?
    end

    def self.active_unit_id(unit_name, neighborhood_name, report)
      unit = HealthUnit.where(active: true).find_by("lower(name) = lower(?)", unit_name.to_s.strip)
      report.warnings << "unidade \"#{unit_name}\" não encontrada ou inativa (bairro #{neighborhood_name})" unless unit
      unit&.id
    end
    private_class_method :load_entry, :active_unit_id
  end
end
```

- [ ] **Step 6: Implemente a rake**

```ruby
# lib/tasks/territory.rake
# Semente de bairros por cidade (ADR 0023; spec 2026-09-28 §3.4). Roda na
# cidade pela conexão dela (CityConnection.with — nunca Current.city solto).
# Idempotente: cria só o que falta. Sem arquivo para a cidade: mensagem e
# saída sem erro. Rollout: depois de city:migrate:all.
namespace :city do
  namespace :territory do
    # Lambda (não método): um `def` dentro de `namespace` vaza para o escopo
    # top-level do processo Rake (mesma razão de lib/tasks/city.rake).
    territory_seed = lambda do |city|
      path = Territory::Seed.path_for(city.slug)
      unless File.file?(path)
        puts "[city:territory:seed] #{city.slug}: sem semente (db/seeds/territory/#{city.slug}.yml) — nada a fazer"
        next
      end

      report = CityConnection.with(city) { Territory::Seed.call(path: path) }
      report.warnings.each { |warning| puts "[city:territory:seed] #{city.slug}: aviso — #{warning}" }
      puts "[city:territory:seed] #{city.slug}: #{report.created} criados, #{report.existing} já existentes, " \
           "#{report.warnings.size} avisos"
    end

    desc "Carrega a semente de bairros de uma cidade (idempotente). Uso: city:territory:seed[slug]"
    task :seed, %i[slug] => :environment do |_t, args|
      abort "uso: rails 'city:territory:seed[slug]'" if args[:slug].blank?
      city = City.find_by(slug: args[:slug]) || abort("[city:territory:seed] cidade #{args[:slug]} não existe")
      abort "[city:territory:seed] cidade #{city.slug} está archived — não tem banco" if city.status == "archived"

      territory_seed.call(city)
    end

    namespace :seed do
      desc "Carrega a semente de bairros de toda cidade active/suspended (idempotente)."
      task all: :environment do
        failed = []
        City.where(status: %w[active suspended]).order(:slug).each do |city|
          territory_seed.call(city)
        rescue StandardError => e
          failed << city.slug
          warn "[city:territory:seed:all] #{city.slug} falhou — #{e.class}"
        end
        abort "[city:territory:seed:all] falharam: #{failed.join(', ')}" if failed.any?
      end
    end
  end
end
```

- [ ] **Step 7: Escreva as sementes**

Cada item leva `key`: o slug do nome original (sem acento, minúsculo, hífen), que nunca muda depois de publicado — é por ela que a semente reconhece o bairro mesmo renomeado.

Curitiba: os 75 bairros oficiais da prefeitura. A cobertura liga as três unidades da semente de dev (`lib/professional_crew.rb`: "UBS Jardim das Flores", "UBS Vila Esperança", "UPA 24h Centro"), de 2 a 4 bairros cada; os outros ficam sem cobertura (em staging/produção essas unidades não existem e viram aviso da carga).

```yaml
# db/seeds/territory/curitiba.yml
# Bairros oficiais de Curitiba (75, divisão da prefeitura). A cobertura liga
# as unidades da semente de dev; em outro ambiente, unidade não encontrada é
# aviso da carga, não erro. Editar pelo dashboard depois da carga.
neighborhoods:
  - name: Abranches
    key: abranches
  - name: Água Verde
    key: agua-verde
  - name: Ahú
    key: ahu
  - name: Alto Boqueirão
    key: alto-boqueirao
    units: ["UBS Vila Esperança"]
  - name: Alto da Glória
    key: alto-da-gloria
  - name: Alto da Rua XV
    key: alto-da-rua-xv
  - name: Atuba
    key: atuba
  - name: Augusta
    key: augusta
  - name: Bacacheri
    key: bacacheri
  - name: Bairro Alto
    key: bairro-alto
  - name: Barreirinha
    key: barreirinha
  - name: Batel
    key: batel
  - name: Bigorrilho
    key: bigorrilho
  - name: Boa Vista
    key: boa-vista
  - name: Bom Retiro
    key: bom-retiro
  - name: Boqueirão
    key: boqueirao
    units: ["UBS Vila Esperança"]
  - name: Butiatuvinha
    key: butiatuvinha
    units: ["UBS Jardim das Flores"]
  - name: Cabral
    key: cabral
  - name: Cachoeira
    key: cachoeira
  - name: Cajuru
    key: cajuru
  - name: Campina do Siqueira
    key: campina-do-siqueira
  - name: Campo Comprido
    key: campo-comprido
  - name: Campo de Santana
    key: campo-de-santana
  - name: Capão da Imbuia
    key: capao-da-imbuia
  - name: Capão Raso
    key: capao-raso
  - name: Cascatinha
    key: cascatinha
    units: ["UBS Jardim das Flores"]
  - name: Caximba
    key: caximba
  - name: Centro
    key: centro
    units: ["UPA 24h Centro"]
  - name: Centro Cívico
    key: centro-civico
  - name: Cidade Industrial
    key: cidade-industrial
  - name: Cristo Rei
    key: cristo-rei
  - name: Fanny
    key: fanny
  - name: Fazendinha
    key: fazendinha
  - name: Ganchinho
    key: ganchinho
  - name: Guabirotuba
    key: guabirotuba
  - name: Guaíra
    key: guaira
  - name: Hauer
    key: hauer
    units: ["UBS Vila Esperança"]
  - name: Hugo Lange
    key: hugo-lange
  - name: Jardim Botânico
    key: jardim-botanico
  - name: Jardim das Américas
    key: jardim-das-americas
  - name: Jardim Social
    key: jardim-social
  - name: Juvevê
    key: juveve
  - name: Lamenha Pequena
    key: lamenha-pequena
  - name: Lindóia
    key: lindoia
  - name: Mercês
    key: merces
  - name: Mossunguê
    key: mossungue
  - name: Novo Mundo
    key: novo-mundo
  - name: Orleans
    key: orleans
  - name: Parolin
    key: parolin
  - name: Pilarzinho
    key: pilarzinho
  - name: Pinheirinho
    key: pinheirinho
  - name: Portão
    key: portao
  - name: Prado Velho
    key: prado-velho
  - name: Rebouças
    key: reboucas
  - name: Riviera
    key: riviera
  - name: Santa Cândida
    key: santa-candida
  - name: Santa Felicidade
    key: santa-felicidade
    units: ["UBS Jardim das Flores"]
  - name: Santa Quitéria
    key: santa-quiteria
  - name: Santo Inácio
    key: santo-inacio
  - name: São Braz
    key: sao-braz
  - name: São Francisco
    key: sao-francisco
    units: ["UPA 24h Centro"]
  - name: São João
    key: sao-joao
    units: ["UBS Jardim das Flores"]
  - name: São Lourenço
    key: sao-lourenco
  - name: São Miguel
    key: sao-miguel
  - name: Seminário
    key: seminario
  - name: Sítio Cercado
    key: sitio-cercado
  - name: Taboão
    key: taboao
  - name: Tarumã
    key: taruma
  - name: Tatuquara
    key: tatuquara
  - name: Tingui
    key: tingui
  - name: Uberaba
    key: uberaba
  - name: Umbará
    key: umbara
  - name: Vila Izabel
    key: vila-izabel
  - name: Vista Alegre
    key: vista-alegre
  - name: Xaxim
    key: xaxim
```

Maringá: a cidade tem centenas de loteamentos; a semente traz um subconjunto de nomes reais conferidos na base de CEP (grafia dos Correios, ex.: "Zona 01"). A lista completa é decisão da prefeitura (editar no dashboard).

```yaml
# db/seeds/territory/maringa.yml
# Bairros de Maringá: subconjunto de nomes reais, na grafia da base de CEP
# (a cidade tem centenas de loteamentos; a prefeitura completa pelo
# dashboard). A cobertura liga as unidades da semente de dev.
neighborhoods:
  - name: Zona 01
    key: zona-01
    units: ["UPA 24h Centro"]
  - name: Zona 02
    key: zona-02
    units: ["UPA 24h Centro"]
  - name: Zona 03
    key: zona-03
    units: ["UPA 24h Centro"]
  - name: Zona 04
    key: zona-04
  - name: Zona 05
    key: zona-05
  - name: Zona 06
    key: zona-06
    units: ["UBS Jardim das Flores"]
  - name: Zona 07
    key: zona-07
    units: ["UBS Jardim das Flores"]
  - name: Zona 08
    key: zona-08
  - name: Zona Armazém
    key: zona-armazem
  - name: Vila Morangueira
    key: vila-morangueira
  - name: Vila Ipiranga
    key: vila-ipiranga
  - name: Vila Marumby
    key: vila-marumby
  - name: Vila Emília
    key: vila-emilia
  - name: Jardim Alvorada
    key: jardim-alvorada
    units: ["UBS Vila Esperança"]
  - name: Jardim Novo Alvorada
    key: jardim-novo-alvorada
  - name: Parque Residencial Cidade Nova
    key: parque-residencial-cidade-nova
    units: ["UBS Vila Esperança"]
  - name: Parque Residencial Eldorado
    key: parque-residencial-eldorado
    units: ["UBS Vila Esperança"]
  - name: Parque Avenida
    key: parque-avenida
    units: ["UBS Vila Esperança"]
  - name: Parque da Gávea
    key: parque-da-gavea
  - name: Jardim Novo Horizonte
    key: jardim-novo-horizonte
  - name: Jardim Ipanema
    key: jardim-ipanema
  - name: Parque Industrial
    key: parque-industrial
  - name: Parque Industrial Bandeirantes
    key: parque-industrial-bandeirantes
  - name: Loteamento Sumaré
    key: loteamento-sumare
  - name: Jardim Tupinambá
    key: jardim-tupinamba
  - name: Jardim Santa Clara
    key: jardim-santa-clara
  - name: Jardim Real
    key: jardim-real
  - name: Jardim Pinheiros
    key: jardim-pinheiros
  - name: Jardim Oásis
    key: jardim-oasis
  - name: Jardim Higienópolis
    key: jardim-higienopolis
  - name: Jardim Dourados
    key: jardim-dourados
  - name: Bom Jardim
    key: bom-jardim
  - name: Alto das Grevíleas
    key: alto-das-grevileas
  - name: Parque das Grevíleas
    key: parque-das-grevileas
  - name: Parque das Bandeiras
    key: parque-das-bandeiras
  - name: Portal das Torres
    key: portal-das-torres
  - name: Jardim Vitória
    key: jardim-vitoria
  - name: Jardim Paris
    key: jardim-paris
  - name: Jardim Brasil
    key: jardim-brasil
  - name: Jardim Carolina
    key: jardim-carolina
  - name: Jardim Tropical
    key: jardim-tropical
  - name: Conjunto Habitacional Hermann Moraes Barros
    key: conjunto-habitacional-hermann-moraes-barros
  - name: Parque Residencial Tuiuti
    key: parque-residencial-tuiuti
  - name: Jardim Santa Alice
    key: jardim-santa-alice
```

- [ ] **Step 8: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod11 api bundle exec rspec spec/services/territory spec/tasks/territory_rake_spec.rb`
Expected: PASS.

- [ ] **Step 9: Rode a rake nos bancos de dev**

Run:
```bash
docker compose exec -T -w /rails/.claude/mod11 api bin/rails "city:territory:seed[curitiba]"
docker compose exec -T -w /rails/.claude/mod11 api bin/rails city:territory:seed:all
```
Expected: a primeira cria 75 em Curitiba (com cobertura se as unidades da semente de dev existirem; senão, avisos); a segunda relata `0 criados` em Curitiba e cria os 44 de Maringá.

- [ ] **Step 10: Commit**

```bash
/opt/homebrew/bin/git add db/seeds/territory app/services/territory/seed.rb lib/tasks/territory.rake spec/services/territory/seed_spec.rb spec/services/territory/seed_files_spec.rb spec/tasks/territory_rake_spec.rb
/opt/homebrew/bin/git commit -m "feat: seed city neighborhoods from versioned files with a rake task

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 5: Revisão da fatia 1

- [ ] **Step 1:** Rode tudo do módulo e a regressão direta:

```bash
docker compose exec -T -w /rails/.claude/mod11 api bundle exec rspec spec/models/territory_tables_guard_spec.rb spec/models/neighborhood_spec.rb spec/commands/territory spec/requests/territory_spec.rb spec/services/territory spec/tasks/territory_rake_spec.rb spec/services/city_schema_spec.rb spec/initializers/domain_events_bindings_spec.rb spec/adr_pointers_spec.rb spec/architecture/current_city_assignment_spec.rb
```
Expected: PASS.

- [ ] **Step 2:** Revisor (subagente) lê o diff `origin/main..HEAD` contra a spec §3, §4.1 (território), §4.2 (comandos) e o ADR 0023. Corrija o que ele achar antes da fatia 2.

---

## Fatia 2 — F-11.3 (endereço da unidade)

### Task 6: Endereço em `/attendance/units`

**Files:**
- Modify: `app/models/health_unit.rb`, `app/controllers/health_units_controller.rb`
- Test: `spec/models/health_unit_address_spec.rb`, `spec/requests/health_units_address_spec.rb`

**Interfaces:**
- Consumes: `Neighborhood` (Task 1).
- Produces: `HealthUnit#address_street/#address_number/#address_complement/#address_zip` normalizados (texto com `squish`, vazio → `nil`; CEP só dígitos); `create`/`update` de `/attendance/units` aceitam os cinco campos (só as chaves presentes mudam); leituras devolvem `address_street`, `address_number`, `address_complement`, `address_zip`, `neighborhood_id`. Recusas: 422 `invalid_zip`, `invalid_neighborhood`, `invalid_unit` (texto longo demais).

- [ ] **Step 1: Escreva a spec do modelo**

```ruby
# spec/models/health_unit_address_spec.rb
require "rails_helper"

RSpec.describe HealthUnit, "endereço (ADR 0023)" do
  def unit(**attrs) = described_class.new({ name: "UBS Centro", kind: "ubs" }.merge(attrs))

  it "normaliza texto e CEP; vazio vira nil" do
    u = unit(address_street: "  Rua XV  de Novembro ", address_number: " 500 ", address_complement: "",
             address_zip: "80020-310")
    expect(u).to be_valid
    expect(u).to have_attributes(address_street: "Rua XV de Novembro", address_number: "500",
                                 address_complement: nil, address_zip: "80020310")
    expect(unit(address_zip: "").address_zip).to be_nil
  end

  it "recusa CEP fora de 8 dígitos e texto acima do limite" do
    { address_zip: "8002031", address_street: "x" * 161, address_number: "1" * 21, address_complement: "c" * 81 }
      .each do |field, value|
        u = unit(field => value)
        expect(u).not_to be_valid, field.to_s
        expect(u.errors.attribute_names).to include(field)
      end
    expect(unit(address_zip: "8002031a")).not_to be_valid
  end
end
```

- [ ] **Step 2: Escreva a spec de request**

```ruby
# spec/requests/health_units_address_spec.rb
require "rails_helper"

RSpec.describe "Endereço da unidade", type: :request do
  def body = JSON.parse(response.body)
  def status_and_error = [ response.status, body["error"] ]

  let(:admin) { staff_with("admin@cidade.gov.br", "municipal_admin") }
  let!(:centro) { Neighborhood.create!(name: "Centro", source: "seed") }
  let(:address) do
    { address_street: "Rua XV de Novembro", address_number: "500", address_complement: "sala 2",
      address_zip: "80020-310", neighborhood_id: centro.id }
  end

  before { sign_in_as(admin) }

  it "cria com endereço e as leituras devolvem os campos" do
    json_post "/attendance/units", { name: "UPA 24h Centro", kind: "upa" }.merge(address)
    expect(response).to have_http_status(:created)
    expected = { "address_street" => "Rua XV de Novembro", "address_number" => "500",
                 "address_complement" => "sala 2", "address_zip" => "80020310", "neighborhood_id" => centro.id }
    expect(body["unit"]).to include(expected)

    get "/attendance/units/all"
    expect(body["units"].sole).to include(expected)

    sign_in_as(staff_with("atendente@cidade.gov.br", "citizen_verifier"))
    get "/attendance/units"
    expect(body["units"].sole).to include(expected)
  end

  it "update sem as chaves de endereço preserva o endereço (dashboard anterior ao módulo 11)" do
    unit = create_unit("UBS Centro")
    unit.update!(address_street: "Rua XV de Novembro", address_zip: "80020310", neighborhood: centro)
    json_post "/attendance/units/#{unit.id}", name: "UBS Centro Renomeada", kind: "ubs"
    expect(response).to have_http_status(:ok)
    expect(unit.reload).to have_attributes(name: "UBS Centro Renomeada", address_street: "Rua XV de Novembro",
                                           address_zip: "80020310", neighborhood_id: centro.id)
  end

  it "null apaga o campo" do
    unit = create_unit("UBS Centro")
    unit.update!(address_complement: "sala 2", neighborhood: centro)
    json_post "/attendance/units/#{unit.id}", name: "UBS Centro", kind: "ubs", address_complement: nil, neighborhood_id: nil
    expect(unit.reload).to have_attributes(address_complement: nil, neighborhood_id: nil)
  end

  it "bairro inativo é aceito como localização (a spec só recusa inexistente)" do
    centro.update!(active: false)
    json_post "/attendance/units", name: "UBS Centro", kind: "ubs", neighborhood_id: centro.id
    expect(response).to have_http_status(:created)
  end

  describe "recusas (nada gravado)" do
    it "CEP fora de 8 dígitos ou não texto: 422 invalid_zip" do
      [ "1234", "8002031a", [ "80020310" ] ].each do |zip|
        json_post "/attendance/units", name: "UBS Nova", kind: "ubs", address_zip: zip
        expect(status_and_error).to eq([ 422, "invalid_zip" ]), zip.inspect
      end
      expect(HealthUnit.count).to eq(0)
    end

    it "bairro inexistente, id que não é UUID, ou não texto: 422 invalid_neighborhood" do
      unit = create_unit("UBS Centro")
      [ SecureRandom.uuid, "nao-e-uuid", { "id" => centro.id } ].each do |value|
        json_post "/attendance/units/#{unit.id}", name: "UBS Centro", kind: "ubs", neighborhood_id: value
        expect(status_and_error).to eq([ 422, "invalid_neighborhood" ]), value.inspect
      end
      expect(unit.reload.neighborhood_id).to be_nil
    end

    it "logradouro acima de 160: 422 invalid_unit" do
      json_post "/attendance/units", name: "UBS Nova", kind: "ubs", address_street: "x" * 161
      expect(status_and_error).to eq([ 422, "invalid_unit" ])
    end
  end
end
```

- [ ] **Step 3: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod11 api bundle exec rspec spec/models/health_unit_address_spec.rb spec/requests/health_units_address_spec.rb`
Expected: FAIL (sem normalização; leituras sem os campos).

- [ ] **Step 4: Complete o modelo**

Em `app/models/health_unit.rb`, depois do `before_validation { self.name = name&.strip }`:

```ruby
  # Endereço em texto (ADR 0023; módulo 11 entrega o que o ADR 0018 deixou em
  # aberto). Tudo opcional. O CEP vem do navegador do dashboard; o api só
  # confere o formato (8 dígitos) e nunca consulta serviço de CEP.
  normalizes :address_street, :address_number, :address_complement,
             with: ->(v) { v.to_s.squish.presence }, apply_to_nil: true
  normalizes :address_zip, with: ->(v) { v.to_s.gsub(/[\s.-]/, "").presence }, apply_to_nil: true

  validates :address_street, length: { maximum: 160 }
  validates :address_number, length: { maximum: 20 }
  validates :address_complement, length: { maximum: 80 }
  validates :address_zip, format: { with: /\A\d{8}\z/ }, allow_nil: true
```

- [ ] **Step 5: Atualize o controller**

Em `app/controllers/health_units_controller.rb`:

1. Logo depois de `include AttendanceAccess`:

```ruby
  # Endereço da unidade (ADR 0023; spec 2026-09-28-module-11-territory §4.1).
  ADDRESS_FIELDS = %w[address_street address_number address_complement address_zip neighborhood_id].freeze
```

2. Substitua `create` e `update` por:

```ruby
  def create
    unit = HealthUnit.new(name: params[:name], kind: params[:kind])
    error = assign_address(unit)
    return render(json: { error: error }, status: :unprocessable_entity) if error
    return render_invalid(unit) unless unit.save

    render json: { unit: unit_json(unit, include_active: true) }, status: :created
  rescue ActiveRecord::RecordNotUnique
    render json: { error: "unit_name_taken" }, status: :unprocessable_entity
  end

  def update
    return render json: { error: "not_found" }, status: :not_found unless @unit

    @unit.assign_attributes(name: params[:name], kind: params[:kind])
    error = assign_address(@unit)
    return render(json: { error: error }, status: :unprocessable_entity) if error
    return render_invalid(@unit) unless @unit.save

    render json: { unit: unit_json(@unit, include_active: true) }
  rescue ActiveRecord::RecordNotUnique
    render json: { error: "unit_name_taken" }, status: :unprocessable_entity
  end
```

3. Em `private`, acrescente:

```ruby
  # Só as chaves presentes no corpo mudam: um cliente que ainda não manda o
  # endereço (dashboard anterior ao módulo 11) não o apaga ao salvar nome e
  # tipo. Valor não escalar é recusado com o código do campo. Bairro precisa
  # existir (inativo é aceito: é onde a unidade fica, não uma escolha nova).
  def assign_address(unit)
    ADDRESS_FIELDS.each do |field|
      next unless params.key?(field)

      value = params[field]
      return address_error(field) unless value.nil? || value.is_a?(String)
      if field == "neighborhood_id" && value.present? && !Neighborhood.exists?(id: value)
        return "invalid_neighborhood"
      end

      unit.public_send("#{field}=", value.presence)
    end
    nil
  end

  def address_error(field)
    { "address_zip" => "invalid_zip", "neighborhood_id" => "invalid_neighborhood" }.fetch(field, "invalid_unit")
  end
```

4. Em `render_invalid`, antes do `else` final:

```ruby
    elsif unit.errors.details[:address_zip].present?
      render json: { error: "invalid_zip" }, status: :unprocessable_entity
```

5. Substitua `unit_json` por:

```ruby
  def unit_json(unit, include_active: false)
    json = { id: unit.id, name: unit.name, kind: unit.kind }.merge(unit.slice(*ADDRESS_FIELDS).symbolize_keys)
    json[:active] = unit.active if include_active
    json
  end
```

- [ ] **Step 6: Rode e veja passar, com a regressão das unidades**

Run: `docker compose exec -T -w /rails/.claude/mod11 api bundle exec rspec spec/models/health_unit_address_spec.rb spec/requests/health_units_address_spec.rb spec/requests/health_units_spec.rb spec/invariants/health_unit_invariants_spec.rb`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
/opt/homebrew/bin/git add app/models/health_unit.rb app/controllers/health_units_controller.rb spec/models/health_unit_address_spec.rb spec/requests/health_units_address_spec.rb
/opt/homebrew/bin/git commit -m "feat: store health unit address and neighborhood

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Fatia 3 — F-11.4 + F-11.5 (bairro do cidadão, cópia na triagem, unidade de referência)

### Task 7: `Citizens::SetNeighborhood`, cópia no `StartTriage` e bairro apagado na revogação

**Files:**
- Create: `app/commands/citizens/set_neighborhood.rb`
- Modify: `app/commands/start_triage.rb`, `app/jobs/anonymize_revoked_triage_job.rb`
- Test: `spec/commands/citizens/set_neighborhood_spec.rb`, `spec/commands/start_triage_neighborhood_spec.rb`, `spec/jobs/anonymize_revoked_triage_job_spec.rb`

**Interfaces:**
- Consumes: `Neighborhood.active_neighborhoods` (Task 2); evento `citizen.neighborhood_changed` já declarado (Task 2).
- Produces: `Citizens::SetNeighborhood.call(citizen:, neighborhood_id:) → Result` (ok: `citizen:`; fail: `:invalid_neighborhood`); `neighborhood_id` `nil` ou `""` apaga; evento só quando muda. `StartTriage` grava `triages.neighborhood_id = conversation.citizen&.neighborhood_id`. `AnonymizeRevokedTriageJob` zera `neighborhood_id` das triagens `aborted_by_revocation` (desvio 14; o trigger da Task 1 aceita só esse caso).

- [ ] **Step 1: Escreva as specs**

```ruby
# spec/commands/citizens/set_neighborhood_spec.rb
require "rails_helper"

RSpec.describe Citizens::SetNeighborhood do
  let(:citizen) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }
  let!(:centro) { Neighborhood.create!(name: "Centro", source: "seed") }
  let!(:batel) { Neighborhood.create!(name: "Batel", source: "seed") }

  def payloads = DomainEvent.where(name: "citizen.neighborhood_changed").map(&:payload)

  it "grava, troca e apaga, com um evento por mudança, só ids" do
    described_class.call(citizen: citizen, neighborhood_id: centro.id)
    described_class.call(citizen: citizen, neighborhood_id: batel.id.upcase)
    described_class.call(citizen: citizen, neighborhood_id: nil)

    expect(citizen.reload.neighborhood_id).to be_nil
    expect(payloads).to eq([
      { "citizen_id" => citizen.id, "from_id" => nil, "to_id" => centro.id },
      { "citizen_id" => citizen.id, "from_id" => centro.id, "to_id" => batel.id },
      { "citizen_id" => citizen.id, "from_id" => batel.id, "to_id" => nil }
    ])
  end

  it "mesmo bairro, ou nil sem bairro: ok, sem evento" do
    described_class.call(citizen: citizen, neighborhood_id: "")
    described_class.call(citizen: citizen, neighborhood_id: centro.id)
    described_class.call(citizen: citizen, neighborhood_id: centro.id)
    expect(payloads.size).to eq(1)
  end

  it "bairro inativo, inexistente ou id que não é UUID: invalid_neighborhood e nada muda" do
    centro.update!(active: false)
    [ centro.id, SecureRandom.uuid, "nao-e-uuid" ].each do |id|
      expect(described_class.call(citizen: citizen, neighborhood_id: id).reason).to eq(:invalid_neighborhood)
    end
    expect(citizen.reload.neighborhood_id).to be_nil
    expect(payloads).to be_empty
  end
end
```

```ruby
# spec/commands/start_triage_neighborhood_spec.rb
require "rails_helper"

RSpec.describe StartTriage, "cópia do bairro (ADR 0023)" do
  before { create_default_protocol! }

  let!(:centro) { Neighborhood.create!(name: "Centro", source: "seed") }
  let!(:batel) { Neighborhood.create!(name: "Batel", source: "seed") }

  def web_conversation(citizen)
    Conversation.create!(channel: "web", citizen: citizen, phone: citizen.phone, state: "consented")
  end

  it "a triagem nasce com o bairro atual do cidadão; trocar depois não muda a triagem" do
    citizen = Citizen.create!(cpf: "52998224725", phone: "+5541998765432", neighborhood: centro)
    triage = described_class.call(conversation: web_conversation(citizen)).payload[:triage]
    expect(triage.neighborhood_id).to eq(centro.id)

    Citizens::SetNeighborhood.call(citizen: citizen, neighborhood_id: batel.id)
    expect(triage.reload.neighborhood_id).to eq(centro.id)
  end

  it "cidadão sem bairro, ou conversa do WhatsApp sem cidadão: triagem sem bairro" do
    citizen = Citizen.create!(cpf: "52998224725", phone: "+5541998765432")
    expect(described_class.call(conversation: web_conversation(citizen)).payload[:triage].neighborhood_id).to be_nil

    whatsapp = Conversation.create!(phone: "+5541911112222", state: "consented")
    expect(described_class.call(conversation: whatsapp).payload[:triage].neighborhood_id).to be_nil
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod11 api bundle exec rspec spec/commands/citizens/set_neighborhood_spec.rb spec/commands/start_triage_neighborhood_spec.rb`
Expected: FAIL (`uninitialized constant Citizens::SetNeighborhood`; triagem com `neighborhood_id` nil).

- [ ] **Step 3: Implemente o comando**

```ruby
# app/commands/citizens/set_neighborhood.rb
# Bairro declarado pelo cidadão (ADR 0023; spec 2026-09-28 §4.2). nil (ou "")
# = "prefiro não informar", ou apagar o que tinha. Recusa bairro inativo ou
# inexistente. Trocar o bairro não muda triagem antiga: a cópia de cada uma é
# imutável (trigger triages_neighborhood_immutable). O evento só sai quando
# muda, e leva só ids. Reasons: :invalid_neighborhood.
module Citizens
  class SetNeighborhood
    def self.call(citizen:, neighborhood_id:)
      target = neighborhood_id.to_s.downcase.presence
      return Result.fail(:invalid_neighborhood) if target && !Neighborhood.active_neighborhoods.exists?(id: target)

      ApplicationRecord.transaction do
        citizen.lock!
        from = citizen.neighborhood_id
        next if from == target

        citizen.update!(neighborhood_id: target)
        DomainEvents.publish("citizen.neighborhood_changed", citizen_id: citizen.id, from_id: from, to_id: target)
      end
      Result.ok(citizen: citizen)
    end
  end
end
```

- [ ] **Step 4: Copie o bairro no `StartTriage`**

Em `app/commands/start_triage.rb`, acrescente ao comentário do topo:

```ruby
# Copia o bairro atual do cidadão na criação (ADR 0023): é a única escrita de
# triages.neighborhood_id — depois, o trigger triages_neighborhood_immutable
# recusa qualquer mudança. Conversa do WhatsApp sem cidadão: sem bairro.
```

e, no `create!`, depois de `status: :in_progress`:

```ruby
      status: :in_progress,
      neighborhood_id: conversation.citizen&.neighborhood_id
```

- [ ] **Step 5: Anonimização da revogação apaga o bairro (spec do job primeiro)**

Ao fim do `RSpec.describe` de `spec/jobs/anonymize_revoked_triage_job_spec.rb`, antes do `end` final:

```ruby
  # ADR 0023, decisão de 2026-09-28: revogar apaga também o bairro copiado.
  it "zera o bairro da triagem revogada e não toca a de outra conversa" do
    pd = ProtocolDefinition.create!(name: "rev-demo", version: 1, status: "active", definition: definition_hash)
    centro = Neighborhood.create!(name: "Centro", source: "seed")
    convo = Conversation.create!(phone: "+551133", state: "revoked")
    revoked = Triage.create!(conversation: convo, protocol_definition: pd, protocol_name: "rev-demo",
                             status: "aborted_by_revocation", answers: {}, neighborhood_id: centro.id,
                             completed_at: Time.current)
    other = Triage.create!(conversation: Conversation.create!(phone: "+551144", state: "completed"),
                           protocol_definition: pd, protocol_name: "rev-demo", status: "completed",
                           answers: {}, neighborhood_id: centro.id, completed_at: Time.current)

    described_class.new.perform(**event_args(convo.id))

    expect(revoked.reload.neighborhood_id).to be_nil
    expect(other.reload.neighborhood_id).to eq(centro.id)
  end
```

Run: `docker compose exec -T -w /rails/.claude/mod11 api bundle exec rspec spec/jobs/anonymize_revoked_triage_job_spec.rb`
Expected: FAIL no exemplo novo (`neighborhood_id` continua preenchido).

Em `app/jobs/anonymize_revoked_triage_job.rb`, acrescente ao comentário do topo `# ADR 0023: apaga também o bairro copiado (única exceção do trigger triages_neighborhood_immutable).` e troque o `update_columns` por:

```ruby
      t.update_columns(
        answers: {}, outcome: nil, tier: nil, priority: nil,
        current_step: nil, neighborhood_id: nil, updated_at: Time.current
      )
```

(o `where(status: :aborted_by_revocation)` já garante a condição que o trigger exige.)

- [ ] **Step 6: Rode e veja passar, com a regressão da triagem e da revogação**

Run: `docker compose exec -T -w /rails/.claude/mod11 api bundle exec rspec spec/commands/citizens/set_neighborhood_spec.rb spec/commands/start_triage_neighborhood_spec.rb spec/jobs/anonymize_revoked_triage_job_spec.rb spec/requests/citizen_api/triage_flow_spec.rb spec/commands`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
/opt/homebrew/bin/git add app/commands/citizens/set_neighborhood.rb app/commands/start_triage.rb app/jobs/anonymize_revoked_triage_job.rb spec/commands/citizens/set_neighborhood_spec.rb spec/commands/start_triage_neighborhood_spec.rb spec/jobs/anonymize_revoked_triage_job_spec.rb
/opt/homebrew/bin/git commit -m "feat: let citizens declare a neighborhood and copy it into new triages

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 8: Rotas do cidadão: lista de bairros, bairro da pessoa e início da conversa

**Files:**
- Create: `app/controllers/citizen_api/neighborhoods_controller.rb`
- Modify: `app/controllers/citizen_api/people_controller.rb`, `app/controllers/citizen_api/conversations_controller.rb`, `config/routes.rb`
- Test: `spec/requests/citizen_api/neighborhood_spec.rb`

**Interfaces:**
- Consumes: `Citizens::SetNeighborhood` (Task 7).
- Produces: `GET /citizen/neighborhoods` → `{ neighborhoods: [{ id, name }] }` (ativos, por nome); `GET /citizen/people` → cada pessoa com `neighborhood: { id, name } | null`; `POST /citizen/people/:id/neighborhood` `{ neighborhood_id | null }` → `{ person: {...} }` (404 `not_found` para CPF fora da sessão; 422 `invalid_neighborhood`); `POST /citizen/conversations` aceita `neighborhood_id` opcional.

- [ ] **Step 1: Escreva a spec de request**

```ruby
# spec/requests/citizen_api/neighborhood_spec.rb
require "rails_helper"

RSpec.describe "Bairro do cidadão", type: :request do
  before do
    create_default_protocol!
    ConsentTerm.create!(version: "1", body: "Termo de teste", published_at: Time.current)
    sign_in_citizen("+5541998765432")
  end
  after { Rails.cache.clear }

  def body = JSON.parse(response.body)
  def status_and_error = [ response.status, body["error"] ]

  let!(:centro) { Neighborhood.create!(name: "Centro", source: "seed") }
  let!(:batel) { Neighborhood.create!(name: "Batel", source: "seed") }
  let!(:fechado) { Neighborhood.create!(name: "Ahu", source: "seed", active: false) }

  def start(params) = json_post("/citizen/conversations", { consent_version: "1" }.merge(params))
  def own_citizen(**attrs) = Citizen.create!({ cpf: "52998224725", phone: "+5541998765432" }.merge(attrs))

  it "lista só os bairros ativos, por nome" do
    get "/citizen/neighborhoods"
    expect(body).to eq("neighborhoods" => [ { "id" => batel.id, "name" => "Batel" }, { "id" => centro.id, "name" => "Centro" } ])
  end

  it "CPF novo com bairro: grava, a triagem copia, e people mostra" do
    start(cpf: "529.982.247-25", neighborhood_id: centro.id)
    expect(response).to have_http_status(:created)
    expect(Citizen.find(body["citizen_id"]).neighborhood).to eq(centro)
    expect(Triage.find(body.dig("step", "triage_id")).neighborhood_id).to eq(centro.id)

    get "/citizen/people"
    expect(body["people"].sole["neighborhood"]).to eq("id" => centro.id, "name" => "Centro")
  end

  it "prefiro não informar (null ou ausente): pessoa e triagem sem bairro" do
    start(cpf: "529.982.247-25", neighborhood_id: nil)
    expect(response).to have_http_status(:created)
    expect(Triage.find(body.dig("step", "triage_id")).neighborhood_id).to be_nil
    get "/citizen/people"
    expect(body["people"].sole["neighborhood"]).to be_nil
  end

  it "bairro inválido com CPF novo (inativo, inexistente, não UUID, lista): 422 e nenhum Citizen criado" do
    [ fechado.id, SecureRandom.uuid, "nao-e-uuid", [ centro.id ] ].each do |value|
      start(cpf: "529.982.247-25", neighborhood_id: value)
      expect(status_and_error).to eq([ 422, "invalid_neighborhood" ]), value.inspect
    end
    expect(Citizen.count).to eq(0)
  end

  it "termo desatualizado com bairro: 409 e nada gravado" do
    json_post "/citizen/conversations", cpf: "529.982.247-25", consent_version: "0", neighborhood_id: centro.id
    expect(response).to have_http_status(:conflict)
    expect(Citizen.count).to eq(0)
    expect(DomainEvent.where(name: "citizen.neighborhood_changed")).to be_empty
  end

  it "pessoa existente sem bairro: o bairro vem no início e é gravado" do
    citizen = own_citizen
    start(citizen_id: citizen.id, neighborhood_id: batel.id)
    expect(citizen.reload.neighborhood).to eq(batel)
    expect(Triage.find(body.dig("step", "triage_id")).neighborhood_id).to eq(batel.id)
  end

  it "pessoa que já tem bairro: outro bairro no início é ignorado; a triagem copia o que ela tinha" do
    citizen = own_citizen(neighborhood: centro)
    start(citizen_id: citizen.id, neighborhood_id: batel.id)
    expect(response).to have_http_status(:created)
    expect(citizen.reload.neighborhood).to eq(centro)
    expect(Triage.find(body.dig("step", "triage_id")).neighborhood_id).to eq(centro.id)
  end

  describe "POST /citizen/people/:id/neighborhood" do
    it "troca, publica o evento, e não muda a triagem antiga" do
      start(cpf: "529.982.247-25", neighborhood_id: centro.id)
      triage_id = body.dig("step", "triage_id")
      citizen_id = body["citizen_id"]

      json_post "/citizen/people/#{citizen_id}/neighborhood", neighborhood_id: batel.id
      expect(response).to have_http_status(:ok)
      expect(body["person"]).to include("id" => citizen_id, "neighborhood" => { "id" => batel.id, "name" => "Batel" })
      expect(Triage.find(triage_id).neighborhood_id).to eq(centro.id)
      expect(DomainEvent.where(name: "citizen.neighborhood_changed").last.payload)
        .to eq("citizen_id" => citizen_id, "from_id" => centro.id, "to_id" => batel.id)
    end

    it "null apaga (prefiro não informar)" do
      citizen = own_citizen(neighborhood: centro)
      json_post "/citizen/people/#{citizen.id}/neighborhood", neighborhood_id: nil
      expect(body.dig("person", "neighborhood")).to be_nil
      expect(citizen.reload.neighborhood_id).to be_nil
    end

    it "bairro inativo, inexistente, não texto, ou sem a chave: 422 invalid_neighborhood" do
      citizen = own_citizen(neighborhood: centro)
      [ { neighborhood_id: fechado.id }, { neighborhood_id: SecureRandom.uuid }, { neighborhood_id: [ batel.id ] }, {} ]
        .each do |payload|
          json_post "/citizen/people/#{citizen.id}/neighborhood", payload
          expect(status_and_error).to eq([ 422, "invalid_neighborhood" ]), payload.inspect
        end
      expect(citizen.reload.neighborhood).to eq(centro)
    end

    it "CPF de outro telefone, ou id inexistente: 404" do
      other = Citizen.create!(cpf: "11144477735", phone: "+5541911112222")
      [ other.id, SecureRandom.uuid ].each do |id|
        json_post "/citizen/people/#{id}/neighborhood", neighborhood_id: batel.id
        expect(status_and_error).to eq([ 404, "not_found" ])
      end
      expect(other.reload.neighborhood_id).to be_nil
    end
  end

  it "sem sessão de cidadão: 401" do
    cookies.delete(:citizen_session)
    get "/citizen/neighborhoods"
    expect(response).to have_http_status(:unauthorized)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod11 api bundle exec rspec spec/requests/citizen_api/neighborhood_spec.rb`
Expected: FAIL (rotas inexistentes; bairro não gravado).

- [ ] **Step 3: Controller da lista**

```ruby
# app/controllers/citizen_api/neighborhoods_controller.rb
# GET /citizen/neighborhoods — bairros ATIVOS da cidade, por nome, para a
# escolha do cidadão (ADR 0023): inativo não entra em escolha nova. Envelope
# { neighborhoods: [...] }, como { people: [...] }.
module CitizenApi
  class NeighborhoodsController < BaseController
    def index
      rows = Neighborhood.active_neighborhoods.order(:name).map { |n| { id: n.id, name: n.name } }
      render json: { neighborhoods: rows }
    end
  end
end
```

- [ ] **Step 4: Pessoas com bairro e troca**

Substitua `app/controllers/citizen_api/people_controller.rb` por:

```ruby
# GET  /citizen/people — "para quem é esta triagem?": os CPFs ligados ao
#   telefone da sessão, mascarados, com o bairro declarado (ADR 0023).
# POST /citizen/people/:id/neighborhood { neighborhood_id | null } — troca o
#   bairro ("Trocar bairro" no wpda). Não muda triagem antiga: a cópia de cada
#   uma é imutável. Sem a chave: 422 (só null explícito apaga).
module CitizenApi
  class PeopleController < BaseController
    def index
      people = current_citizen_session.citizens.includes(:neighborhood).order(:created_at)
      render json: { people: people.map { |c| person_json(c) } }
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
        neighborhood: citizen.neighborhood && { id: citizen.neighborhood.id, name: citizen.neighborhood.name }
      }
    end
  end
end
```

- [ ] **Step 5: Bairro no início da conversa**

Em `app/controllers/citizen_api/conversations_controller.rb`:

1. Na primeira linha do comentário do topo, `POST /citizen/conversations { citizen_id | cpf, consent_version }` passa a `{ citizen_id | cpf, consent_version, neighborhood_id? }`.

2. Em `create`, entre o bloco do `consent_outdated` e `citizen = resolve_citizen`, e logo depois do `return if performed?` que segue `resolve_citizen`:

```ruby
      # ADR 0023: bairro inválido é recusado ANTES de resolve_citizen, que cria
      # o Citizen (com o CPF) para um CPF novo — um pedido recusado não grava
      # nada. Depois do consentimento, como o CPF.
      neighborhood_id = requested_neighborhood_id
      return if performed?

      citizen = resolve_citizen
      return if performed?

      # Grava só quando a pessoa ainda não tem bairro; a troca é pela rota
      # própria. Antes do StartConversation: a triagem nova copia o bairro.
      if neighborhood_id && citizen.neighborhood_id.nil?
        set = Citizens::SetNeighborhood.call(citizen: citizen, neighborhood_id: neighborhood_id)
        return render_error(set.reason, :unprocessable_entity) if set.failure?
      end
```

(o `citizen = resolve_citizen` e o `return if performed?` originais ficam substituídos por estes.)

3. Em `private`:

```ruby
    # nil quando não veio (ou veio vazio/null: "prefiro não informar"); o id
    # quando é um bairro ativo; senão responde 422 e devolve nil.
    def requested_neighborhood_id
      raw = params[:neighborhood_id]
      return nil if raw.nil? || raw == ""
      return raw if raw.is_a?(String) && Neighborhood.active_neighborhoods.exists?(id: raw)

      render_error("invalid_neighborhood", :unprocessable_entity)
      nil
    end
```

- [ ] **Step 6: Rotas**

Em `config/routes.rb`, no `scope "/citizen"`, logo depois de `get "people", to: "people#index"`:

```ruby
    post "people/:id/neighborhood",    to: "people#neighborhood"
    get  "neighborhoods",              to: "neighborhoods#index"
```

- [ ] **Step 7: Rode e veja passar, com a regressão do canal web**

Run: `docker compose exec -T -w /rails/.claude/mod11 api bundle exec rspec spec/requests/citizen_api`
Expected: PASS.

- [ ] **Step 8: Commit**

```bash
/opt/homebrew/bin/git add app/controllers/citizen_api/neighborhoods_controller.rb app/controllers/citizen_api/people_controller.rb app/controllers/citizen_api/conversations_controller.rb config/routes.rb spec/requests/citizen_api/neighborhood_spec.rb
/opt/homebrew/bin/git commit -m "feat: let web citizens pick and change their neighborhood

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 9: `Territory::ReferenceUnits` e `reference_units` na triagem do cidadão

**Files:**
- Create: `app/services/territory/reference_units.rb`
- Modify: `app/controllers/citizen_api/triages_controller.rb`
- Test: `spec/services/territory/reference_units_spec.rb`, `spec/requests/citizen_api/reference_units_spec.rb`

**Interfaces:**
- Consumes: `NeighborhoodCoverage`, endereço da unidade (Task 6).
- Produces:
  - `Territory::ReferenceUnits.for(neighborhood_id) → Array<HealthUnit>` (ativas, com cobertura no bairro, por nome; `nil` → `[]`);
  - `Territory::ReferenceUnits.ids_by_neighborhood(neighborhood_ids) → Hash{String => Array<String>}` (ids de unidade ativa por nome; bairro sem nenhuma fica fora do hash);
  - `Territory::ReferenceUnits.as_json_list(units) → [{ id:, name:, kind:, address: { street:, number:, complement:, zip: } }]`;
  - `GET /citizen/triages/:id` ganha `reference_units` (do bairro copiado na triagem). `GET /r/:token` **não** muda.

- [ ] **Step 1: Escreva a spec do serviço**

```ruby
# spec/services/territory/reference_units_spec.rb
require "rails_helper"

RSpec.describe Territory::ReferenceUnits do
  let!(:centro) { Neighborhood.create!(name: "Centro", source: "seed") }
  let!(:batel) { Neighborhood.create!(name: "Batel", source: "seed") }
  let!(:upa) { create_unit("UPA 24h", kind: "upa") }
  let!(:ubs) { create_unit("UBS Centro") }
  let!(:fechada) { create_unit("UBS Antiga", active: false) }

  before do
    [ upa, ubs, fechada ].each { |u| NeighborhoodCoverage.create!(neighborhood: centro, health_unit: u) }
    NeighborhoodCoverage.create!(neighborhood: batel, health_unit: fechada)
  end

  it "for: só as ativas que cobrem o bairro, por nome; nil = lista vazia" do
    expect(described_class.for(centro.id)).to eq([ ubs, upa ])
    expect(described_class.for(batel.id)).to eq([])
    expect(described_class.for(nil)).to eq([])
  end

  it "ids_by_neighborhood: várias de uma vez, mesmas regras" do
    expect(described_class.ids_by_neighborhood([ centro.id, batel.id, nil, centro.id ]))
      .to eq(centro.id => [ ubs.id, upa.id ])
    expect(described_class.ids_by_neighborhood([ nil ])).to eq({})
  end

  it "as_json_list: endereço como objeto, campos podendo ser nulos" do
    ubs.update!(address_street: "Rua XV de Novembro", address_number: "500", address_zip: "80020310")
    expect(described_class.as_json_list(described_class.for(centro.id))).to eq([
      { id: ubs.id, name: "UBS Centro", kind: "ubs",
        address: { street: "Rua XV de Novembro", number: "500", complement: nil, zip: "80020310" } },
      { id: upa.id, name: "UPA 24h", kind: "upa", address: { street: nil, number: nil, complement: nil, zip: nil } }
    ])
  end
end
```

- [ ] **Step 2: Escreva a spec de request**

```ruby
# spec/requests/citizen_api/reference_units_spec.rb
require "rails_helper"

RSpec.describe "Unidade de referência do cidadão", type: :request do
  before do
    create_default_protocol!
    ConsentTerm.create!(version: "1", body: "Termo de teste", published_at: Time.current)
    sign_in_citizen("+5541998765432")
    [ ubs, upa, fechada ].each { |u| NeighborhoodCoverage.create!(neighborhood: centro, health_unit: u) }
  end
  after { Rails.cache.clear }

  def body = JSON.parse(response.body)

  let!(:centro) { Neighborhood.create!(name: "Centro", source: "seed") }
  let!(:batel) { Neighborhood.create!(name: "Batel", source: "seed") }
  let!(:ubs) do
    create_unit("UBS Centro").tap do |u|
      u.update!(address_street: "Rua XV de Novembro", address_number: "500", address_zip: "80020310", neighborhood: centro)
    end
  end
  let!(:upa) { create_unit("UPA 24h", kind: "upa") }
  let!(:fechada) { create_unit("UBS Antiga", active: false) }

  def start_triage(neighborhood_id)
    json_post "/citizen/conversations", cpf: "529.982.247-25", consent_version: "1", neighborhood_id: neighborhood_id
    { triage_id: body.dig("step", "triage_id"), citizen_id: body["citizen_id"] }
  end

  it "GET /citizen/triages/:id traz as unidades ativas que cobrem o bairro da triagem, por nome" do
    ids = start_triage(centro.id)
    get "/citizen/triages/#{ids[:triage_id]}"
    expect(body["reference_units"]).to eq([
      { "id" => ubs.id, "name" => "UBS Centro", "kind" => "ubs",
        "address" => { "street" => "Rua XV de Novembro", "number" => "500", "complement" => nil, "zip" => "80020310" } },
      { "id" => upa.id, "name" => "UPA 24h", "kind" => "upa",
        "address" => { "street" => nil, "number" => nil, "complement" => nil, "zip" => nil } }
    ])
  end

  it "sem bairro: lista vazia" do
    ids = start_triage(nil)
    get "/citizen/triages/#{ids[:triage_id]}"
    expect(body["reference_units"]).to eq([])
  end

  it "trocar o bairro depois não muda a referência da triagem antiga" do
    ids = start_triage(centro.id)
    json_post "/citizen/people/#{ids[:citizen_id]}/neighborhood", neighborhood_id: batel.id
    get "/citizen/triages/#{ids[:triage_id]}"
    expect(body["reference_units"].map { |u| u["id"] }).to eq([ ubs.id, upa.id ])
  end

  it "unidade desativada depois: some da referência" do
    ids = start_triage(centro.id)
    upa.update!(active: false)
    get "/citizen/triages/#{ids[:triage_id]}"
    expect(body["reference_units"].map { |u| u["id"] }).to eq([ ubs.id ])
  end

  # Decisão do usuário (2026-09-28): o relatório público (link sem login) não
  # mostra a unidade de referência nem nada de bairro.
  it "o relatório público GET /r/:token não traz reference_units nem bairro" do
    ids = start_triage(centro.id)
    triage = Triage.find(ids[:triage_id])
    token = ReportSnapshot.mint_token
    ReportSnapshot.create!(triage: triage, protocol_definition: triage.protocol_definition,
                           outcome: { "tier" => "alta" }, payload: { "tier" => "alta", "priority" => 1 },
                           token: token, signature: ReportSnapshot.sign(token), expires_at: 30.days.from_now)

    get "/r/#{token}"
    expect(response).to have_http_status(:ok)
    expect(body.keys).not_to include("reference_units")
    expect(response.body).not_to match(/reference_units|neighborhood/)
    expect(response.body).not_to include(centro.id, "Centro", ubs.id, "UBS Centro")
  end
end
```

- [ ] **Step 3: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod11 api bundle exec rspec spec/services/territory/reference_units_spec.rb spec/requests/citizen_api/reference_units_spec.rb`
Expected: FAIL (`uninitialized constant Territory::ReferenceUnits`; `reference_units` ausente). O exemplo do relatório público já passa — é a guarda de regressão.

- [ ] **Step 4: Implemente o serviço**

```ruby
# app/services/territory/reference_units.rb
# Unidade de referência (ADR 0023; spec 2026-09-28 §4.2): as unidades ATIVAS
# que cobrem o bairro, por nome. Calculada na hora, nunca gravada; informa e
# sugere, nunca restringe. Fonte única do cidadão (GET /citizen/triages/:id)
# e do atendimento (reference_unit_ids na fila da unidade). Não vai para o
# relatório público (/r/:token): decisão de 2026-09-28.
module Territory
  module ReferenceUnits
    module_function

    def for(neighborhood_id)
      return [] if neighborhood_id.blank?

      HealthUnit.where(active: true)
                .where(id: NeighborhoodCoverage.where(neighborhood_id: neighborhood_id).select(:health_unit_id))
                .order(:name).to_a
    end

    # Várias de uma vez (fila da unidade, sem N+1).
    def ids_by_neighborhood(neighborhood_ids)
      ids = neighborhood_ids.compact.uniq
      return {} if ids.empty?

      NeighborhoodCoverage.joins(:health_unit)
                          .where(neighborhood_id: ids, health_units: { active: true })
                          .order("health_units.name")
                          .pluck(:neighborhood_id, :health_unit_id)
                          .group_by(&:first)
                          .transform_values { |pairs| pairs.map(&:last) }
    end

    def as_json_list(units)
      units.map do |u|
        {
          id: u.id, name: u.name, kind: u.kind,
          address: { street: u.address_street, number: u.address_number, complement: u.address_complement,
                     zip: u.address_zip }
        }
      end
    end
  end
end
```

- [ ] **Step 5: `reference_units` na leitura da triagem**

Em `app/controllers/citizen_api/triages_controller.rb`:
- no comentário do topo, `GET  /citizen/triages/:id` passa a `GET  /citizen/triages/:id   (com reference_units, ADR 0023)`;
- em `show`, `render json: summary(triage)` passa a:

```ruby
      # ADR 0023: do bairro COPIADO na triagem (o mesmo do atendimento), não do
      # atual do cidadão.
      units = Territory::ReferenceUnits.for(triage.neighborhood_id)
      render json: summary(triage).merge(reference_units: Territory::ReferenceUnits.as_json_list(units))
```

`ReportsController` (`/r/:token`) **não** muda.

- [ ] **Step 6: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod11 api bundle exec rspec spec/services/territory/reference_units_spec.rb spec/requests/citizen_api spec/requests/reports_spec.rb`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
/opt/homebrew/bin/git add app/services/territory/reference_units.rb app/controllers/citizen_api/triages_controller.rb spec/services/territory/reference_units_spec.rb spec/requests/citizen_api/reference_units_spec.rb
/opt/homebrew/bin/git commit -m "feat: show reference health units on the citizen triage

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Fatia 4 — F-11.6 (referência no desfecho)

### Task 10: `reference_unit_ids` na fila da unidade

**Files:**
- Modify: `app/models/attendance.rb`, `app/controllers/attendances_controller.rb`
- Test: `spec/models/attendance_territory_spec.rb`, `spec/requests/attendance_reference_units_spec.rb`

**Interfaces:**
- Consumes: `Territory::ReferenceUnits.ids_by_neighborhood` (Task 9); cópia no `StartTriage` (Task 7).
- Produces: `Attendance#territory_neighborhood_id → String | nil` (bairro da triagem raiz; sem triagem raiz, o atual do cidadão); cada linha de `GET /attendance/units/:id/queue` (`waiting` e `in_care`) ganha `reference_unit_ids: [String]` (ativas, por nome, **sem a própria unidade da linha**; `[]` sem bairro, sem cobertura ou quando só a própria unidade cobre).

- [ ] **Step 1: Escreva a spec do modelo**

```ruby
# spec/models/attendance_territory_spec.rb
require "rails_helper"

# ADR 0023: o bairro do caso é o copiado na triagem raiz (a própria, ou a do
# pedido do horário); sem triagem nenhuma, o bairro atual do cidadão.
RSpec.describe Attendance, "#territory_neighborhood_id" do
  let(:centro) { Neighborhood.create!(name: "Centro", source: "seed") }
  let(:batel) { Neighborhood.create!(name: "Batel", source: "seed") }
  let(:citizen) { Citizen.new(neighborhood: batel) }

  it "atendimento por triagem: o bairro da triagem" do
    triage = Triage.new(neighborhood: centro)
    expect(described_class.new(citizen: citizen, triage: triage).territory_neighborhood_id).to eq(centro.id)
  end

  it "triagem sem bairro: nil (não cai para o bairro atual)" do
    expect(described_class.new(citizen: citizen, triage: Triage.new).territory_neighborhood_id).to be_nil
  end

  it "atendimento por horário: o bairro da triagem raiz do pedido" do
    request = AppointmentRequest.new(root_triage: Triage.new(neighborhood: centro))
    attendance = described_class.new(citizen: citizen, appointment: Appointment.new(request: request))
    expect(attendance.territory_neighborhood_id).to eq(centro.id)
  end

  it "sem triagem nenhuma: o bairro atual do cidadão" do
    expect(described_class.new(citizen: citizen).territory_neighborhood_id).to eq(batel.id)
  end
end
```

- [ ] **Step 2: Escreva a spec de request**

```ruby
# spec/requests/attendance_reference_units_spec.rb
require "rails_helper"

RSpec.describe "Unidade de referência no desfecho", type: :request do
  def body = JSON.parse(response.body)

  let(:verifier) { staff_with("atendente@cidade.gov.br", "citizen_verifier") }
  let(:doctor) { staff_with("medica@cidade.gov.br", "health_professional") }
  let!(:unit) { create_unit("UBS Centro") }
  let!(:upa) { create_unit("UPA Norte", kind: "upa") }
  let!(:ubs_sul) { create_unit("UBS Sul") }
  let!(:fechada) { create_unit("UBS Antiga", active: false) }
  let!(:centro) { Neighborhood.create!(name: "Centro", source: "seed") }
  let!(:batel) { Neighborhood.create!(name: "Batel", source: "seed") }

  # "UBS Centro" (a própria unidade da fila) também cobre o Centro: nunca volta.
  before do
    [ upa, unit, ubs_sul, fechada ].each { |u| NeighborhoodCoverage.create!(neighborhood: centro, health_unit: u) }
    NeighborhoodCoverage.create!(neighborhood: batel, health_unit: upa)
  end

  it "cada linha traz as unidades ativas do bairro copiado na triagem, por nome, sem a própria unidade" do
    citizen = Citizen.create!(cpf: "52998224725", phone: "+5541998765432", neighborhood: centro)
    waiting = waiting_attendance(citizen, unit: unit, by: verifier)
    Citizens::SetNeighborhood.call(citizen: citizen, neighborhood_id: batel.id) # troca depois: não muda o caso

    other = Citizen.create!(cpf: "11144477735", phone: "+5541911112222", neighborhood: centro)
    in_care!(waiting_attendance(other, unit: unit, by: verifier), by: doctor)

    sign_in_as(verifier)
    get "/attendance/units/#{unit.id}/queue"
    expect(body["waiting"].sole).to include("id" => waiting.id, "reference_unit_ids" => [ upa.id, ubs_sul.id ])
    expect(body["in_care"].sole["reference_unit_ids"]).to eq([ upa.id, ubs_sul.id ])
  end

  it "bairro coberto só pela própria unidade do atendimento: lista vazia" do
    so_propria = Neighborhood.create!(name: "Ahu", source: "seed")
    NeighborhoodCoverage.create!(neighborhood: so_propria, health_unit: unit)
    citizen = Citizen.create!(cpf: "52998224725", phone: "+5541998765432", neighborhood: so_propria)
    waiting_attendance(citizen, unit: unit, by: verifier)
    sign_in_as(verifier)
    get "/attendance/units/#{unit.id}/queue"
    expect(body["waiting"].sole["reference_unit_ids"]).to eq([])
  end

  it "triagem sem bairro: lista vazia" do
    citizen = Citizen.create!(cpf: "52998224725", phone: "+5541998765432")
    waiting_attendance(citizen, unit: unit, by: verifier)
    sign_in_as(verifier)
    get "/attendance/units/#{unit.id}/queue"
    expect(body["waiting"].sole["reference_unit_ids"]).to eq([])
  end

  it "unidade desativada depois: some de reference_unit_ids" do
    citizen = Citizen.create!(cpf: "52998224725", phone: "+5541998765432", neighborhood: centro)
    waiting_attendance(citizen, unit: unit, by: verifier)
    upa.update!(active: false)
    sign_in_as(verifier)
    get "/attendance/units/#{unit.id}/queue"
    expect(body["waiting"].sole["reference_unit_ids"]).to eq([ ubs_sul.id ])
  end
end
```

- [ ] **Step 3: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod11 api bundle exec rspec spec/models/attendance_territory_spec.rb spec/requests/attendance_reference_units_spec.rb`
Expected: FAIL (`undefined method 'territory_neighborhood_id'`; `reference_unit_ids` ausente).

- [ ] **Step 4: Implemente**

Em `app/models/attendance.rb`, depois de `def priority ... end`:

```ruby
  # Bairro do caso para a unidade de referência (ADR 0023): o copiado na
  # triagem raiz; sem triagem raiz (nem pela cadeia do horário), o bairro
  # atual do cidadão. Triagem sem bairro continua sem bairro.
  def territory_neighborhood_id
    root = root_triage
    root ? root.neighborhood_id : citizen&.neighborhood_id
  end
```

Em `app/controllers/attendances_controller.rb`, substitua `queue` por:

```ruby
  def queue
    unit = HealthUnit.find_by(id: params[:id])
    return render json: { error: "not_found" }, status: :not_found unless unit

    waiting = Attendances::UnitQueue.waiting(unit.id)
    in_care = Attendances::UnitQueue.in_care(unit.id)
    # ADR 0023: o formulário de desfecho (dashboard) pré-seleciona a primeira
    # por nome; informa e sugere, nunca restringe. A própria unidade da linha
    # nunca entra (decisão de 2026-09-28): encaminhar para si mesma não é
    # encaminhamento.
    refs = Territory::ReferenceUnits.ids_by_neighborhood((waiting + in_care).map(&:territory_neighborhood_id))
    render json: { waiting: waiting.map { |a| queue_json(a, refs) }, in_care: in_care.map { |a| queue_json(a, refs) } }
  end
```

e `queue_json` passa a receber `refs`:

```ruby
  def queue_json(a, refs)
    {
      id: a.id, cpf_masked: a.citizen.cpf_masked, checked_in_at: a.checked_in_at&.iso8601,
      protocol_name: a.root_triage&.protocol_name, priority: a.priority,
      source: a.appointment_id ? "appointment" : "triage", appointment_time: a.appointment&.scheduled_at&.iso8601,
      called_at: a.called_at&.iso8601, called_by_name: staff_name(a.called_by_user),
      reference_unit_ids: refs.fetch(a.territory_neighborhood_id, []) - [ a.health_unit_id ]
    }
  end
```

`Attendances::UnitQueue::INCLUDES` já carrega `:citizen`, `:triage` e `appointment: { request: :root_triage }`: sem N+1. Confira que `queue_json` não é chamado em outro lugar (`grep -n "queue_json" app`); se for, passe `refs` também.

- [ ] **Step 5: Rode e veja passar, com a regressão do atendimento**

Run: `docker compose exec -T -w /rails/.claude/mod11 api bundle exec rspec spec/models/attendance_territory_spec.rb spec/requests/attendance_reference_units_spec.rb spec/requests/attendances_spec.rb spec/requests/attendance_contract_spec.rb spec/invariants/appointment_invariants_spec.rb`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git add app/models/attendance.rb app/controllers/attendances_controller.rb spec/models/attendance_territory_spec.rb spec/requests/attendance_reference_units_spec.rb
/opt/homebrew/bin/git commit -m "feat: expose reference unit ids on the unit queue for the referral outcome

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Fatia 5 — F-11.7 (filtro por bairro e supressão nos painéis)

### Task 11: `Admin::SmallCount` e `Admin::NeighborhoodFilter`

**Files:**
- Create: `app/queries/admin/small_count.rb`, `app/queries/admin/neighborhood_filter.rb`
- Test: `spec/queries/admin/neighborhood_filter_spec.rb`

**Interfaces:**
- Produces:
  - `Admin::SmallCount::SUPPRESSED` (`{ suppressed: true }`, congelado), `.small?(n)` (inteiro de 1 a 4), `.wrap(n)`;
  - `Admin::NeighborhoodFilter.off`, `.parse(raw)` (levanta `Admin::NeighborhoodFilter::Invalid`), `#active?`, `#descriptor` (`nil` | `"none"` | `{ id:, name: }`), `#triages(rel)`, `#conversations(rel)`, `#report_snapshots(rel)`, `#count(n)`, `#series(values)`, `#over(total, value)`, `#share(count, total, value)`, `#list(total, rows)` (`nil` quando suprimida). Com o filtro desligado, todos devolvem a entrada intacta.

- [ ] **Step 1: Escreva a spec**

```ruby
# spec/queries/admin/neighborhood_filter_spec.rb
require "rails_helper"

RSpec.describe Admin::NeighborhoodFilter do
  let(:suppressed) { Admin::SmallCount::SUPPRESSED }
  let!(:centro) { Neighborhood.create!(name: "Centro", source: "seed") }
  let!(:antigo) { Neighborhood.create!(name: "Ahu", source: "seed", active: false) }

  describe ".parse" do
    it "ausente ou vazio: desligado, descritor nil" do
      [ nil, "" ].each do |raw|
        filter = described_class.parse(raw)
        expect(filter.active?).to be(false)
        expect(filter.descriptor).to be_nil
      end
    end

    it "none e bairro (ativo ou inativo) ligam o filtro" do
      expect(described_class.parse("none").descriptor).to eq("none")
      expect(described_class.parse(centro.id).descriptor).to eq(id: centro.id, name: "Centro")
      expect(described_class.parse(antigo.id).active?).to be(true)
    end

    it "texto que não é UUID, UUID inexistente, lista ou objeto: Invalid" do
      [ "abc", SecureRandom.uuid, [ centro.id ], { "id" => centro.id } ].each do |raw|
        expect { described_class.parse(raw) }.to raise_error(described_class::Invalid), raw.inspect
      end
    end
  end

  describe "recortes" do
    before do
      territory_triage!(centro)
      territory_triage!(nil)
      Conversation.create!(phone: "+5541911110000", state: "consented") # WhatsApp, sem cidadão
    end

    it "triagens pelo bairro copiado; none = sem bairro" do
      expect(described_class.parse(centro.id).triages(Triage.all).count).to eq(1)
      expect(described_class.parse("none").triages(Triage.all).count).to eq(1)
      expect(described_class.off.triages(Triage.all).count).to eq(2)
    end

    it "conversas pelo bairro atual do cidadão; none inclui conversa sem cidadão" do
      expect(described_class.parse(centro.id).conversations(Conversation.all).count).to eq(1)
      expect(described_class.parse("none").conversations(Conversation.all).count).to eq(2)
    end

    it "relatórios pelo bairro da triagem" do
      Triage.find_each { |t| territory_report!(t) }
      expect(described_class.parse(centro.id).report_snapshots(ReportSnapshot.all).count).to eq(1)
    end
  end

  describe "supressão" do
    let(:on) { described_class.parse(centro.id) }
    let(:off) { described_class.off }

    it "count: 1 a 4 suprimido, 0 e 5+ aparecem; desligado não mexe" do
      expect([ 0, 1, 4, 5 ].map { |n| on.count(n) }).to eq([ 0, suppressed, suppressed, 5 ])
      expect(off.count(3)).to eq(3)
    end

    it "series: ponto a ponto" do
      expect(on.series([ 0, 2, 7 ])).to eq([ 0, suppressed, 7 ])
      expect(off.series([ 0, 2, 7 ])).to eq([ 0, 2, 7 ])
    end

    it "over: taxa ou média sobre total de 1 a 4 some" do
      expect(on.over(3, 66.7)).to eq(suppressed)
      expect(on.over(0, 0.0)).to eq(0.0)
      expect(on.over(6, 50.0)).to eq(50.0)
      expect(off.over(3, 66.7)).to eq(66.7)
    end

    it "share: some se a contagem OU o total for de 1 a 4" do
      expect(on.share(1, 10, 10)).to eq(suppressed)
      expect(on.share(5, 3, 100)).to eq(suppressed)
      expect(on.share(5, 10, 50)).to eq(50)
    end

    it "list: null quando o total é de 1 a 4" do
      expect(on.list(3, [ :a ])).to be_nil
      expect(on.list(5, [ :a ])).to eq([ :a ])
      expect(off.list(3, [ :a ])).to eq([ :a ])
    end
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod11 api bundle exec rspec spec/queries/admin/neighborhood_filter_spec.rb`
Expected: FAIL (`uninitialized constant Admin::NeighborhoodFilter`).

- [ ] **Step 3: Implemente**

```ruby
# app/queries/admin/small_count.rb
# Supressão de contagem pequena nos painéis filtrados por bairro (ADR 0023;
# spec 2026-09-28 §4.3): contagem por bairro de 1 a 4, somada à urgência, pode
# identificar uma pessoa. Aplicada DEPOIS de agregar. 0 continua 0.
module Admin::SmallCount
  SUPPRESSED = { suppressed: true }.freeze
  RANGE = (1..4)

  module_function

  def small?(value)
    value.is_a?(Integer) && RANGE.cover?(value)
  end

  def wrap(value)
    small?(value) ? SUPPRESSED : value
  end
end
```

```ruby
# app/queries/admin/neighborhood_filter.rb
# Filtro de bairro dos cinco painéis com cidadão (ADR 0023; spec 2026-09-28
# §4.3). Objeto único: recorta as relações e suprime os números. Desligado
# (parâmetro ausente — o console admin nunca o manda), devolve tudo intacto.
#
#   neighborhood_id=<uuid> → um bairro (ativo ou inativo: o filtro vale para o histórico)
#   neighborhood_id=none   → sem bairro (inclui conversa sem cidadão)
#
# Triagem e relatório: bairro COPIADO na triagem. Conversa: bairro ATUAL do
# cidadão (a conversa não tem cópia).
class Admin::NeighborhoodFilter
  class Invalid < StandardError; end

  NONE = "none"
  UUID = /\A\h{8}-\h{4}-\h{4}-\h{4}-\h{12}\z/

  def self.off
    new(mode: :off)
  end

  def self.parse(raw)
    return off if raw.nil? || raw == ""
    raise Invalid unless raw.is_a?(String)
    return new(mode: :none) if raw == NONE
    raise Invalid unless raw.match?(UUID)

    neighborhood = Neighborhood.find_by(id: raw) || raise(Invalid)
    new(mode: :one, neighborhood: neighborhood)
  end

  def initialize(mode:, neighborhood: nil)
    @mode = mode
    @neighborhood = neighborhood
  end

  def active?
    @mode != :off
  end

  def descriptor
    case @mode
    when :off then nil
    when :none then NONE
    else { id: @neighborhood.id, name: @neighborhood.name }
    end
  end

  def triages(relation)
    active? ? relation.where(triages: { neighborhood_id: value }) : relation
  end

  def conversations(relation)
    active? ? relation.left_joins(:citizen).where(citizens: { neighborhood_id: value }) : relation
  end

  def report_snapshots(relation)
    active? ? relation.joins(:triage).where(triages: { neighborhood_id: value }) : relation
  end

  def count(n)
    active? ? Admin::SmallCount.wrap(n) : n
  end

  def series(values)
    active? ? values.map { |v| Admin::SmallCount.wrap(v) } : values
  end

  # Taxa ou média calculada sobre `total`.
  def over(total, value)
    active? && Admin::SmallCount.small?(total) ? Admin::SmallCount::SUPPRESSED : value
  end

  # Fatia de uma categoria: some se a categoria ou o total for pequeno (com
  # os dois à mostra, a fatia devolveria a contagem suprimida).
  def share(count, total, value)
    return value unless active?
    return Admin::SmallCount::SUPPRESSED if Admin::SmallCount.small?(count) || Admin::SmallCount.small?(total)

    value
  end

  # Lista de amostra: null quando o total filtrado é pequeno (a chave fica).
  def list(total, rows)
    active? && Admin::SmallCount.small?(total) ? nil : rows
  end

  private

  def value
    @mode == :none ? nil : @neighborhood.id
  end
end
```

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod11 api bundle exec rspec spec/queries/admin/neighborhood_filter_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add app/queries/admin/small_count.rb app/queries/admin/neighborhood_filter.rb spec/queries/admin/neighborhood_filter_spec.rb
/opt/homebrew/bin/git commit -m "feat: add neighborhood filter and small count suppression for panels

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 12: Visão geral e Classificação com filtro

**Files:**
- Modify: `app/queries/admin/overview_query.rb`, `app/queries/admin/classification_query.rb`
- Test: `spec/queries/admin/overview_classification_filter_spec.rb`

**Interfaces:**
- Consumes: `Admin::NeighborhoodFilter` (Task 11).
- Produces: `Admin::OverviewQuery.call(period:, filter: Admin::NeighborhoodFilter.off)`, `Admin::ClassificationQuery.call(period:, filter: Admin::NeighborhoodFilter.off)` — sem `filter`, saída idêntica à de hoje.

- [ ] **Step 1: Escreva a spec**

```ruby
# spec/queries/admin/overview_classification_filter_spec.rb
require "rails_helper"

RSpec.describe "Visão geral e Classificação filtradas por bairro (ADR 0023)" do
  def period = Admin::Api::Period.parse(key: "7d", from: nil, to: nil, tz: ActiveSupport::TimeZone["America/Sao_Paulo"])
  def filter(raw) = Admin::NeighborhoodFilter.parse(raw)

  let(:suppressed) { Admin::SmallCount::SUPPRESSED }
  let!(:small) { Neighborhood.create!(name: "Batel", source: "seed") }
  let!(:big) { Neighborhood.create!(name: "Centro", source: "seed") }

  before do
    3.times { territory_triage!(small) }
    6.times { territory_triage!(big) }
    2.times { territory_triage!(nil) }
    Conversation.create!(phone: "+5541911110000", state: "consented") # WhatsApp, sem cidadão
  end

  describe Admin::OverviewQuery do
    def kpis(raw) = described_class.call(period: period, filter: filter(raw))[:kpis].index_by { |k| k[:id] }

    it "sem filtro: a cidade inteira, sem supressão" do
      expect(kpis(nil).transform_values { |k| k[:value] }).to include("done" => 11, "urgent" => 11, "completion" => 100.0)
      expect(described_class.call(period: period)[:kpis]).to eq(described_class.call(period: period, filter: filter(nil))[:kpis])
    end

    it "bairro com 1 a 4: KPI, pontos da série e taxa suprimidos; jobs com falha não" do
      k = kpis(small.id)
      expect(k["done"][:value]).to eq(suppressed)
      expect(k["urgent"][:value]).to eq(suppressed)
      expect(k["done"][:spark]).to all(satisfy { |v| v == 0 || v == suppressed })
      expect(k["done"][:spark]).to include(suppressed)
      expect(k["completion"]).to include(value: suppressed, tone: "neutral")
      expect(k["failed"][:value]).to eq(SolidQueue::FailedExecution.count)
    end

    it "bairro com 5 ou mais: o número aparece" do
      k = kpis(big.id)
      expect(k["done"][:value]).to eq(6)
      expect(k["completion"][:value]).to eq(100.0)
    end

    it "none: triagens sem bairro; conversas ativas sem cidadão ou sem bairro" do
      k = kpis("none")
      expect(k["done"][:value]).to eq(suppressed)
      expect(k["active"][:value]).to eq(suppressed)
    end
  end

  describe Admin::ClassificationQuery do
    def out(raw) = described_class.call(period: period, filter: filter(raw))

    it "bairro com 1 a 4: contagens, apelidos, share e série suprimidos; amostra null" do
      o = out(small.id)
      expect(o[:tiers].map { |t| t[:count] }).to all(eq(suppressed))
      expect([ o[:urgent], o[:priorityTrue] ]).to eq([ suppressed, suppressed ])
      expect(o[:urgentTrend]).to include(suppressed)
      expect(o[:byMode].map { |m| [ m[:count], m[:share] ] }).to all(eq([ suppressed, suppressed ]))
      expect(o[:byProtocol].flat_map { |r| r[:counts].values }).to all(eq(suppressed))
      expect(o[:sampleTriages]).to be_nil
    end

    it "bairro grande com uma categoria de 1 caso: a categoria e o share dela saem suppressed; o resto aparece" do
      territory_triage!(big, tier: "baixa", priority: 9)
      o = out(big.id)
      counts = o[:tiers].to_h { |t| [ t[:key], t[:count] ] }
      expect(counts).to eq("alta" => 6, "baixa" => suppressed)
      expect(o[:byProtocol].sole[:counts]).to eq("alta" => 6, "baixa" => suppressed)
      expect(o[:byMode].sole).to include(count: 7, share: 100)
      expect(o[:sampleTriages].size).to eq(7)
    end

    it "sem filtro: igual ao de antes" do
      expect(out(nil)).to eq(described_class.call(period: period))
      expect(out(nil)[:urgent]).to eq(11)
    end
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod11 api bundle exec rspec spec/queries/admin/overview_classification_filter_spec.rb`
Expected: FAIL (`unknown keyword: :filter`).

- [ ] **Step 3: Reescreva a Visão geral**

```ruby
# app/queries/admin/overview_query.rb
# GET /admin/api/overview — KPIs operacionais (§4.0).
#
# Todos os KPIs agregam ao vivo no banco da cidade (ADR 0022; source: live).
# O campo source continua no contrato (§7) para quando um KPI passar a ler
# projeção por ADR novo. Urgência = régua do alerta (Protocols::Urgency).
#
# Filtro de bairro (ADR 0023): triagens pelo bairro copiado, conversas pelo
# bairro atual do cidadão; com o filtro ligado, 1 a 4 sai suprimido. Jobs com
# falha não são do cidadão: ignoram o filtro.
class Admin::OverviewQuery
  def self.call(period:, filter: Admin::NeighborhoodFilter.off)
    new(period, filter).call
  end

  def initialize(period, filter)
    @period = period
    @filter = filter
  end

  def call
    {
      kpis: [
        kpi_done,
        kpi_active,
        kpi_urgent,
        kpi_completion,
        kpi_failed_jobs
      ]
    }
  end

  private

  def triages = @filter.triages(Triage.all)
  def conversations = @filter.conversations(Conversation.all)

  def kpi_done
    completed = triages
                  .where(status: "completed", completed_at: @period.from..@period.to)
                  .count
    {
      id: "done",
      label: "Triagens concluídas",
      value: @filter.count(completed),
      unit: "",
      delta: nil,
      tone: completed.positive? ? "ok" : "neutral",
      spark: @filter.series(@period.series(triages.where(status: "completed"), :completed_at)),
      source: "live"
    }
  end

  def kpi_active
    active = conversations
               .where(state: %w[awaiting_consent consented])
               .where(updated_at: 1.hour.ago..)
               .count
    {
      id: "active",
      label: "Conversas ativas agora",
      value: @filter.count(active),
      unit: "",
      delta: nil,
      tone: "info",
      spark: @filter.series(@period.series(conversations, :updated_at)),
      source: "live"
    }
  end

  def kpi_urgent
    urgent = triages.where(status: "completed", priority: ..Protocols::Urgency.max_priority)
    count = urgent.where(completed_at: @period.from..@period.to).count
    {
      id: "urgent",
      label: "Casos urgentes",
      value: @filter.count(count),
      unit: "",
      delta: nil,
      tone: count.positive? ? "warn" : "ok",
      spark: @filter.series(@period.series(urgent, :completed_at)),
      source: "live"
    }
  end

  def kpi_completion
    base = triages.where(created_at: @period.from..@period.to)
    started = base.count
    completed = base.where(status: "completed").count
    rate = started.zero? ? 0.0 : (completed.to_f / started * 100).round(1)
    value = @filter.over(started, rate)
    {
      id: "completion",
      label: "Taxa de conclusão",
      value: value,
      unit: "%",
      delta: nil,
      tone: value.is_a?(Hash) ? "neutral" : (rate >= 70 ? "ok" : (rate >= 40 ? "warn" : "down")),
      spark: [],
      source: "live"
    }
  end

  # Infraestrutura — o Solid Queue mora no banco da cidade desde o Plano 5:
  # este número é só desta cidade (city_connection_queue_spec). Não é do
  # cidadão: ignora o filtro de bairro.
  def kpi_failed_jobs
    failed = SolidQueue::FailedExecution.count
    {
      id: "failed",
      label: "Jobs falhados abertos",
      value: failed,
      unit: "",
      delta: nil,
      tone: failed.zero? ? "ok" : (failed < 5 ? "warn" : "down"),
      spark: [],
      source: "live"
    }
  end
end
```

- [ ] **Step 4: Reescreva a Classificação**

Mantenha o comentário do topo e acrescente ao fim dele:

```ruby
#
# Filtro de bairro (ADR 0023): triagens pelo bairro copiado; com o filtro
# ligado, contagens de 1 a 4 (e o share delas) saem suprimidas, e a amostra
# vem null quando o total filtrado é de 1 a 4.
```

e troque o corpo da classe por:

```ruby
class Admin::ClassificationQuery
  LEGACY_TIERS = %w[low medium high].freeze
  MODE_SQL = "protocol_definitions.definition -> 'scoring' ->> 'type'".freeze

  def self.call(period:, filter: Admin::NeighborhoodFilter.off)
    new(period, filter).call
  end

  def initialize(period, filter)
    @period = period
    @filter = filter
  end

  def call
    triages = @filter.triages(Triage.all)
    base = triages.where(status: "completed", completed_at: @period.from..@period.to)
    total = base.count
    urgent_max = Protocols::Urgency.max_priority
    tiers = tier_counts(base, urgent_max)
    urgent = @filter.count(base.where(priority: ..urgent_max).count)
    urgent_trend = @filter.series(@period.series(triages.where(status: "completed", priority: ..urgent_max), :completed_at))

    {
      tiers: tiers,
      tierKeys: tiers.map { |t| t[:key] },
      urgent: urgent,
      urgentMaxPriority: urgent_max,
      urgentTrend: urgent_trend,
      priorityTrue: urgent,        # apelido (apps/admin)
      priorityTrend: urgent_trend, # apelido (apps/admin)
      byProtocol: by_protocol(base),
      byMode: by_mode(base),
      sampleTriages: @filter.list(total, sample(base.limit(8), urgent_max))
    }
  end

  private

  # Ordena pela prioridade mais urgente que o tier recebeu no período (menor
  # primeiro); tier sem priority vai para o fim.
  def tier_counts(scope, urgent_max)
    rows = scope.group(:tier).pluck(:tier, Arel.sql("COUNT(*)"), Arel.sql("MIN(priority)"))
    rows.sort_by { |tier, _count, min_priority| [ min_priority || Float::INFINITY, tier.to_s ] }.map do |tier, count, min_priority|
      key = tier || "sem tier"
      { key: key, label: key, count: @filter.count(count), tone: tone(min_priority, urgent_max) }
    end
  end

  def tone(min_priority, urgent_max)
    return "neutral" if min_priority.nil?
    min_priority <= urgent_max ? "down" : "info"
  end

  def by_protocol(scope)
    rows = scope
             .joins(:protocol_definition)
             .group("protocol_definitions.name", "protocol_definitions.version", :tier)
             .count
    rows.each_with_object({}) do |((name, version, tier), count), pivot|
      key = "#{name} · #{version}"
      pivot[key] ||= { protocol: key, counts: {} }
      pivot[key][:counts][tier || "sem tier"] = count
    end.values.map do |row|
      legacy = LEGACY_TIERS.to_h { |t| [ t.to_sym, @filter.count(row[:counts][t] || 0) ] } # apelidos (apps/admin)
      row.merge(counts: row[:counts].transform_values { |c| @filter.count(c) }).merge(legacy)
    end
  end

  def by_mode(scope)
    rows = scope.joins(:protocol_definition).group(Arel.sql(MODE_SQL)).count
    total = rows.values.sum
    rows.map do |mode, count|
      share = total.zero? ? 0 : (count.to_f / total * 100).round
      {
        mode: mode,
        label: mode,
        count: @filter.count(count),
        share: @filter.share(count, total, share)
      }
    end
  end

  # Amostra: só referências. NUNCA payload de answers.
  def sample(scope, urgent_max)
    scope.includes(:protocol_definition).order(completed_at: :desc).map do |t|
      {
        id: t.id,
        tier: t.tier,
        priority: t.priority,
        urgent: !t.priority.nil? && t.priority <= urgent_max,
        mode: t.protocol_definition&.definition&.dig("scoring", "type"),
        protocol: "#{t.protocol_name} · #{t.protocol_definition&.version}",
        at: t.completed_at&.strftime("%H:%M")
      }
    end
  end
end
```

- [ ] **Step 5: Rode e veja passar, com as specs antigas das duas queries**

Run: `docker compose exec -T -w /rails/.claude/mod11 api bundle exec rspec spec/queries/admin/overview_classification_filter_spec.rb spec/queries/admin/overview_query_spec.rb spec/queries/admin/classification_query_spec.rb spec/lib/dashboard_demo_spec.rb`
Expected: PASS. Se `sampleTriages` vier com 8 no caso "bairro grande" (limite da amostra), ajuste a expectativa para `8` — `base.limit(8)` é o teto.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git add app/queries/admin/overview_query.rb app/queries/admin/classification_query.rb spec/queries/admin/overview_classification_filter_spec.rb
/opt/homebrew/bin/git commit -m "feat: filter overview and classification panels by neighborhood with suppression

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 13: Triagens, Relatórios e Conversas com filtro

**Files:**
- Modify: `app/queries/admin/triages_query.rb`, `app/queries/admin/reports_query.rb`, `app/queries/admin/conversations_query.rb`
- Test: `spec/queries/admin/triages_reports_conversations_filter_spec.rb`

**Interfaces:**
- Consumes: `Admin::NeighborhoodFilter` (Task 11).
- Produces: `Admin::TriagesQuery.call(period:, filter: ...)`, `Admin::ReportsQuery.call(period:, filter: ...)`, `Admin::ConversationsQuery.call(period:, filter: ...)` — sem `filter`, saída idêntica à de hoje.

- [ ] **Step 1: Escreva a spec**

```ruby
# spec/queries/admin/triages_reports_conversations_filter_spec.rb
require "rails_helper"

RSpec.describe "Triagens, Relatórios e Conversas filtrados por bairro (ADR 0023)" do
  def period = Admin::Api::Period.parse(key: "7d", from: nil, to: nil, tz: ActiveSupport::TimeZone["America/Sao_Paulo"])
  def filter(raw) = Admin::NeighborhoodFilter.parse(raw)

  let(:suppressed) { Admin::SmallCount::SUPPRESSED }
  let!(:small) { Neighborhood.create!(name: "Batel", source: "seed") }
  let!(:big) { Neighborhood.create!(name: "Centro", source: "seed") }

  before do
    3.times { territory_report!(territory_triage!(small)) }
    6.times { territory_report!(territory_triage!(big)) }
    2.times { territory_report!(territory_triage!(nil)) }
    Conversation.create!(phone: "+5541911110000", state: "consented") # WhatsApp, sem cidadão
  end

  describe Admin::TriagesQuery do
    def out(raw) = described_class.call(period: period, filter: filter(raw))

    it "bairro com 1 a 4: tudo suprimido" do
      o = out(small.id)
      expect(o.slice(:started, :completed, :completionRate).values).to all(eq(suppressed))
      expect(o[:series]).to include(suppressed)
      expect(o[:series]).to all(satisfy { |v| v == 0 || v == suppressed })
      expect(o[:byProtocol].sole.slice(:count, :share).values).to all(eq(suppressed))
    end

    it "bairro com 5 ou mais: números aparecem" do
      o = out(big.id)
      expect(o).to include(started: 6, completed: 6, completionRate: 100.0)
      expect(o[:byProtocol].sole).to include(count: 6, share: 100)
    end

    it "sem filtro: igual ao de antes" do
      expect(out(nil)).to eq(described_class.call(period: period))
      expect(out(nil)[:started]).to eq(11)
    end
  end

  describe Admin::ReportsQuery do
    def out(raw) = described_class.call(period: period, filter: filter(raw))

    it "bairro com 1 a 4: total suprimido e lista null; 5 ou mais: lista e total" do
      expect(out(small.id)).to eq(reports: nil, total: suppressed)
      expect(out(big.id)[:total]).to eq(6)
      expect(out(big.id)[:reports].size).to eq(6)
      expect(out("none")).to eq(reports: nil, total: suppressed)
    end

    it "sem filtro: igual ao de antes" do
      expect(out(nil)[:total]).to eq(11)
    end
  end

  describe Admin::ConversationsQuery do
    def out(raw) = described_class.call(period: period, filter: filter(raw))

    it "bairro com 1 a 4: saídas, taxa e tempo médio suprimidos; zeros continuam zero" do
      o = out(small.id)
      exits = o[:exits].to_h { |e| [ e[:key], e[:count] ] }
      expect(exits["completed"]).to eq(suppressed)
      expect(exits["abandoned"]).to eq(0)
      expect([ o[:abandonRate], o[:avgToCompleteMin] ]).to eq([ suppressed, suppressed ])
      expect(o[:live]).to eq(0)
    end

    it "bairro com 5 ou mais: números aparecem" do
      o = out(big.id)
      expect(o[:exits].find { |e| e[:key] == "completed" }[:count]).to eq(6)
      expect(o[:avgToCompleteMin]).to eq(5.0)
      expect(o[:abandonRate]).to eq(0.0)
    end

    it "none: a conversa do WhatsApp sem cidadão entra, suprimida" do
      o = out("none")
      expect(o[:live]).to eq(suppressed)
      expect(o[:liveActive][:inProgress]).to eq(suppressed)
    end

    it "sem filtro: igual ao de antes" do
      expect(out(nil)).to eq(described_class.call(period: period))
    end
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod11 api bundle exec rspec spec/queries/admin/triages_reports_conversations_filter_spec.rb`
Expected: FAIL (`unknown keyword: :filter`).

- [ ] **Step 3: Reescreva as três queries**

```ruby
# app/queries/admin/triages_query.rb
# GET /admin/api/triages — visão de triages (§4.4).
#
# Reduzimos a referências e contagens. NUNCA expomos `answers` (LGPD).
# Versão do protocolo vem de protocol_definitions.
#
# Filtro de bairro (ADR 0023): pelo bairro copiado na triagem; com o filtro
# ligado, 1 a 4 sai suprimido (e a taxa/share calculados sobre eles).
class Admin::TriagesQuery
  def self.call(period:, filter: Admin::NeighborhoodFilter.off)
    new(period, filter).call
  end

  def initialize(period, filter)
    @period = period
    @filter = filter
  end

  def call
    triages = @filter.triages(Triage.all)
    base = triages.where(created_at: @period.from..@period.to)
    started = base.count
    completed = base.where(status: "completed").count
    rate = started.zero? ? 0.0 : (completed.to_f / started * 100).round(1)

    {
      series: @filter.series(@period.series(triages, :created_at)),
      started: @filter.count(started),
      completed: @filter.count(completed),
      completionRate: @filter.over(started, rate),
      byProtocol: by_protocol(base, started)
    }
  end

  private

  def by_protocol(scope, total)
    rows = scope
             .joins(:protocol_definition)
             .group("protocol_definitions.name", "protocol_definitions.version", "protocol_definitions.status")
             .count
    rows.sort_by { |_key, count| -count }.map do |(name, version, status), count|
      share = total.zero? ? 0 : (count.to_f / total * 100).round
      {
        version: "#{name} · #{version}",
        count: @filter.count(count),
        share: @filter.share(count, total, share),
        status: status
      }
    end
  end
end
```

Nota: a ordenação passou para antes de embrulhar (`sort_by` no número cru); a ordem resultante é a mesma de antes.

```ruby
# app/queries/admin/reports_query.rb
# GET /admin/api/reports — lista de relatórios (F-04.6). Metadados APENAS.
# NUNCA expõe token/url/payload/signature (LGPD, como o painel de triages).
#
# Filtro de bairro (ADR 0023): pelo bairro copiado na triagem do relatório;
# com o filtro ligado e total de 1 a 4, o total sai suprimido e a lista vem null.
class Admin::ReportsQuery
  def self.call(period:, filter: Admin::NeighborhoodFilter.off)
    new(period, filter).call
  end

  def initialize(period, filter)
    @period = period
    @filter = filter
  end

  def call
    rows = @filter.report_snapshots(ReportSnapshot.all)
             .where(created_at: @period.from..@period.to)
             .includes(:protocol_definition)
             .order(created_at: :desc)
             .to_a
    { reports: @filter.list(rows.size, rows.map { |r| serialize(r) }), total: @filter.count(rows.size) }
  end

  private

  def serialize(report)
    {
      id: report.id,
      createdAt: report.created_at.iso8601,
      tier: report.outcome["tier"],
      protocol: "#{report.protocol_definition.name} · #{report.protocol_definition.version}",
      expiresAt: report.expires_at&.iso8601,
      live: report.expires_at.nil? || report.expires_at > Time.current
    }
  end
end
```

```ruby
# app/queries/admin/conversations_query.rb
# GET /admin/api/conversations — FSM e funil (§4.2; F-02.9).
#
# Conta as conversas da cidade, de qualquer canal (a web é o canal do cidadão,
# ADR 0017), iniciadas no período: o funil mostra os estados ativos e as
# saídas mostram cada desfecho terminal. abandonRate = abandonadas ÷ iniciadas
# no período, em %.
#
# Filtro de bairro (ADR 0023): conversa pelo bairro ATUAL do cidadão (none
# inclui conversa sem cidadão); tempo médio pelo bairro copiado na triagem.
# Com o filtro ligado, 1 a 4 sai suprimido (e taxa/média sobre eles).
class Admin::ConversationsQuery
  EXITS = { "completed" => "ok", "abandoned" => "warn", "declined" => "neutral",
            "cancelled" => "neutral", "revoked" => "warn" }.freeze

  def self.call(period:, filter: Admin::NeighborhoodFilter.off)
    new(period, filter).call
  end

  def initialize(period, filter)
    @period = period
    @filter = filter
  end

  def call
    base = @filter.conversations(Conversation.all)
    in_period = base.where(created_at: @period.from..@period.to)

    state_counts = in_period.group(:state).count
    {
      live: @filter.count(base.where(state: %w[awaiting_consent consented]).count),
      funnel: [
        { key: "greeting",         label: "greeting",         count: @filter.count(state_counts["greeting"]         || 0), tone: "neutral" },
        { key: "awaiting_consent", label: "awaiting_consent", count: @filter.count(state_counts["awaiting_consent"] || 0), tone: "info" },
        { key: "consented",        label: "consented",        count: @filter.count(state_counts["consented"]        || 0), tone: "ok" }
      ],
      exits: EXITS.map { |key, tone| { key: key, label: key, count: @filter.count(state_counts[key] || 0), tone: tone } },
      abandonRate: abandon_rate(state_counts),
      avgToCompleteMin: avg_complete_minutes,
      liveActive: {
        awaiting: @filter.count(base.where(state: "awaiting_consent").count),
        inProgress: @filter.count(base.where(state: "consented").count)
      }
    }
  end

  private

  def abandon_rate(state_counts)
    started = state_counts.values.sum
    return nil if started.zero?
    @filter.over(started, ((state_counts["abandoned"] || 0).to_f / started * 100).round(1))
  end

  def avg_complete_minutes
    completed = @filter.triages(Triage.all)
                  .where(status: "completed", completed_at: @period.from..@period.to)
    total = completed.count
    return nil if total.zero?
    seconds = completed
                .where.not(completed_at: nil)
                .pluck(Arel.sql("EXTRACT(EPOCH FROM (triages.completed_at - triages.created_at))"))
                .compact
    return nil if seconds.empty?
    @filter.over(total, (seconds.sum / seconds.size / 60.0).round(1))
  end
end
```

- [ ] **Step 4: Rode e veja passar, com as specs antigas das três queries**

Run: `docker compose exec -T -w /rails/.claude/mod11 api bundle exec rspec spec/queries/admin/triages_reports_conversations_filter_spec.rb spec/queries/admin/triages_query_spec.rb spec/queries/admin/conversations_query_spec.rb spec/requests/admin/api/reports_spec.rb spec/requests/admin/api/conversations_spec.rb spec/lib/dashboard_demo_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add app/queries/admin/triages_query.rb app/queries/admin/reports_query.rb app/queries/admin/conversations_query.rb spec/queries/admin/triages_reports_conversations_filter_spec.rb
/opt/homebrew/bin/git commit -m "feat: filter triages, reports and conversations panels by neighborhood

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 14: Parâmetro nos controllers de `/admin/api` e lista de bairros dos painéis

**Files:**
- Modify: `app/controllers/admin/api/base_controller.rb`, `overview_controller.rb`, `classification_controller.rb`, `triages_controller.rb`, `reports_controller.rb`, `conversations_controller.rb`
- Create: `app/controllers/admin/api/neighborhoods_controller.rb`
- Modify: `config/routes.rb`
- Test: `spec/requests/admin/api/neighborhood_filter_spec.rb`

**Interfaces:**
- Consumes: queries das Tasks 12 e 13.
- Produces: `?neighborhood_id=<uuid>|none` nos cinco painéis; `data.filter = { neighborhood: { id, name } | "none" | null }` nos cinco; 422 `invalid_neighborhood`; `GET /admin/api/neighborhoods` → `{ neighborhoods: [{ id, name, active }] }`, sem envelope (desvio 5).

- [ ] **Step 1: Escreva a spec de request**

```ruby
# spec/requests/admin/api/neighborhood_filter_spec.rb
require "rails_helper"

RSpec.describe "Filtro de bairro em /admin/api", type: :request do
  def body = JSON.parse(response.body)

  panels = %w[overview classification triages reports conversations]

  let(:viewer) { staff_with("viewer@cidade.gov.br", "viewer") }
  let!(:antigo) { Neighborhood.create!(name: "Ahu", source: "seed", active: false) }
  let!(:centro) { Neighborhood.create!(name: "Centro", source: "seed") }

  before { sign_in_as(viewer) }

  panels.each do |panel|
    describe "/admin/api/#{panel}" do
      it "sem parâmetro: filter null; com bairro (mesmo inativo): {id, name}; none: \"none\"" do
        get "/admin/api/#{panel}", params: { period: "7d" }
        expect(response).to have_http_status(:ok)
        expect(body.dig("data", "filter")).to eq("neighborhood" => nil)
        expect(body.dig("data", "scope")).to be_present

        get "/admin/api/#{panel}", params: { period: "7d", neighborhood_id: antigo.id }
        expect(body.dig("data", "filter")).to eq("neighborhood" => { "id" => antigo.id, "name" => "Ahu" })

        get "/admin/api/#{panel}", params: { period: "7d", neighborhood_id: "none" }
        expect(body.dig("data", "filter")).to eq("neighborhood" => "none")
      end

      it "parâmetro inválido (não UUID, UUID inexistente, lista): 422 invalid_neighborhood" do
        [ "abc", SecureRandom.uuid, [ centro.id ] ].each do |raw|
          get "/admin/api/#{panel}", params: { period: "7d", neighborhood_id: raw }
          expect(response).to have_http_status(:unprocessable_entity), raw.inspect
          expect(body).to eq("error" => "invalid_neighborhood")
        end
      end
    end
  end

  it "o resto de /admin/api ignora o parâmetro" do
    get "/admin/api/events", params: { period: "7d", neighborhood_id: "abc" }
    expect(response).to have_http_status(:ok)
    expect(body["data"]).not_to have_key("filter")
  end

  describe "GET /admin/api/neighborhoods" do
    let(:expected) do
      { "neighborhoods" => [ { "id" => antigo.id, "name" => "Ahu", "active" => false },
                             { "id" => centro.id, "name" => "Centro", "active" => true } ] }
    end

    Membership::ROLES.each do |role|
      it "#{role}: todos, inativos marcados, por nome, sem envelope" do
        sign_in_as(staff_with("#{role}-nb@cidade.gov.br", role))
        get "/admin/api/neighborhoods"
        expect(response).to have_http_status(:ok)
        expect(body).to eq(expected)
      end
    end

    it "usuário sem vínculo ativo na cidade: 403 no_city_membership (mesma regra dos painéis)" do
      sign_in_as(User.create!(email_address: "sem-vinculo@cidade.gov.br", password: "senha-segura-123"))
      get "/admin/api/neighborhoods"
      expect(response).to have_http_status(:forbidden)
      expect(body).to eq("error" => "no_city_membership")
    end
  end

  it "GET /admin/api/neighborhoods sem sessão: 401" do
    cookies.delete(:session_id)
    get "/admin/api/neighborhoods"
    expect(response).to have_http_status(:unauthorized)
  end
end
```

Antes de rodar, confira que `/admin/api/events` responde 200 a um `viewer` sem dados (`spec/requests/admin/api/events_spec.rb`); se exigir outro papel, troque por um painel sem cidadão que o `viewer` leia (`consent` ou `health`).

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod11 api bundle exec rspec spec/requests/admin/api/neighborhood_filter_spec.rb`
Expected: FAIL (`filter` ausente; rota `neighborhoods` inexistente).

- [ ] **Step 3: Base e controllers**

Em `app/controllers/admin/api/base_controller.rb`:

1. Logo depois de `rescue_from Admin::Api::InvalidScope, with: :render_invalid_scope`:

```ruby
  rescue_from Admin::NeighborhoodFilter::Invalid, with: :render_invalid_neighborhood
```

2. Em `private`, depois de `resolve_scope`:

```ruby
  # Filtro de bairro (ADR 0023; spec 2026-09-28 §4.3): só os cinco painéis com
  # cidadão o leem (Visão geral, Classificação, Triagens, Relatórios,
  # Conversas). Ausente = cidade inteira, sem supressão — o console admin
  # nunca o manda.
  def neighborhood_filter
    @neighborhood_filter ||= Admin::NeighborhoodFilter.parse(params[:neighborhood_id])
  end
```

3. Substitua `render_envelope` por:

```ruby
  # as_of = instante da leitura: os painéis agregam ao vivo (ADR 0022), então
  # é também o horário de origem do dado. `filter` só nos painéis filtráveis.
  def render_envelope(data, as_of: Time.current, filter: nil)
    data = data.merge(filter: { neighborhood: filter.descriptor }) if filter
    render json: {
      data: data.deep_merge(scope_block),
      as_of: as_of.iso8601
    }
  end
```

4. Depois de `render_invalid_scope`:

```ruby
  def render_invalid_neighborhood(_error)
    render json: { error: "invalid_neighborhood" }, status: :unprocessable_entity
  end
```

Os cinco controllers:

```ruby
class Admin::Api::OverviewController < Admin::Api::BaseController
  def show
    data = Admin::OverviewQuery.call(period: period, filter: neighborhood_filter)
    render_envelope(data, filter: neighborhood_filter)
  end
end
```

```ruby
class Admin::Api::ClassificationController < Admin::Api::BaseController
  def show
    data = Admin::ClassificationQuery.call(period: period, filter: neighborhood_filter)
    render_envelope(data, filter: neighborhood_filter)
  end
end
```

Em `Admin::Api::TriagesController#show` (o `trail` não muda):

```ruby
  def show
    data = Admin::TriagesQuery.call(period: period, filter: neighborhood_filter)
    render_envelope(data, filter: neighborhood_filter)
  end
```

Em `Admin::Api::ReportsController#show`:

```ruby
  def show
    render_envelope(Admin::ReportsQuery.call(period: period, filter: neighborhood_filter), filter: neighborhood_filter)
  end
```

```ruby
class Admin::Api::ConversationsController < Admin::Api::BaseController
  def show
    data = Admin::ConversationsQuery.call(period: period, filter: neighborhood_filter)
    render_envelope(data, filter: neighborhood_filter)
  end
end
```

- [ ] **Step 4: Lista de bairros dos painéis**

```ruby
# app/controllers/admin/api/neighborhoods_controller.rb
# GET /admin/api/neighborhoods — opções do seletor de bairro dos painéis (ADR
# 0023; decisão de 2026-09-28: o filtro vale para todos os papéis que leem
# /admin/api). Só leitura, mesma autorização dos painéis (BaseController:
# sessão + vínculo ativo, ou operador por grant); /territory é só do
# municipal_admin. Inclui inativos, marcados — o filtro vale para o
# histórico. Sem o envelope `data` (contrato combinado com o dashboard).
class Admin::Api::NeighborhoodsController < Admin::Api::BaseController
  def index
    rows = Neighborhood.order(:name).map { |n| { id: n.id, name: n.name, active: n.active } }
    render json: { neighborhoods: rows }
  end
end
```

Em `config/routes.rb`, no `namespace :api` de `namespace :admin`, depois de `get "municipalities", ...`:

```ruby
      get "neighborhoods",   to: "neighborhoods#index"
```

- [ ] **Step 5: Rode e veja passar, com a regressão de `/admin/api`**

Run: `docker compose exec -T -w /rails/.claude/mod11 api bundle exec rspec spec/requests/admin spec/queries/admin spec/requests/citizen_session_on_staff_endpoints_spec.rb`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git add app/controllers/admin/api config/routes.rb spec/requests/admin/api/neighborhood_filter_spec.rb
/opt/homebrew/bin/git commit -m "feat: accept a neighborhood filter on citizen panels and list neighborhoods for it

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Fechamento — semente de dev, invariantes e revisão

### Task 15: Semente de dev realista

**Files:**
- Create: `lib/territory_crew.rb`
- Modify: `db/seeds.rb`
- Test: `spec/lib/territory_crew_spec.rb`

**Interfaces:**
- Consumes: `Territory::Seed` (Task 4), `Citizens::RegisterPerson`, `Citizens::SetNeighborhood` (Task 7), `Citizens::StartConversation`, `Citizens::SubmitAnswer`, `CitizenIdentity::Cpf.check_digit`.
- Produces: `TerritoryCrew.seed_current_city(slug:, ddd:) → { units_with_address: Integer, citizens: Integer, new_triages: Integer }`.

- [ ] **Step 1: Escreva a spec**

```ruby
# spec/lib/territory_crew_spec.rb
require "rails_helper"
require Rails.root.join("lib/territory_crew")

RSpec.describe TerritoryCrew do
  let(:dir) { Pathname(Dir.mktmpdir) }
  after { FileUtils.remove_entry(dir) }

  before do
    create_default_protocol!
    ConsentTerm.create!(version: "1", body: "Termo de teste", published_at: Time.current)
    [ [ "UBS Jardim das Flores", "ubs" ], [ "UBS Vila Esperança", "ubs" ], [ "UPA 24h Centro", "upa" ] ]
      .each { |name, kind| create_unit(name, kind: kind) }
    path = dir.join("curitiba.yml")
    path.write(<<~YAML)
      neighborhoods:
        - name: Santa Felicidade
          key: santa-felicidade
          units: ["UBS Jardim das Flores"]
        - name: Boqueirão
          key: boqueirao
          units: ["UBS Vila Esperança"]
        - name: Centro
          key: centro
          units: ["UPA 24h Centro"]
        - name: Batel
          key: batel
    YAML
    Territory::Seed.call(path: path)
  end

  def seed = described_class.seed_current_city(slug: "curitiba", ddd: "41")

  it "dá endereço de CEP real às unidades e cria cidadãos com bairros variados, alguns sem" do
    result = seed

    expect(result).to eq(units_with_address: 3, citizens: 15, new_triages: 15)
    expect(HealthUnit.find_by!(name: "UBS Jardim das Flores")).to have_attributes(
      address_street: "Avenida Manoel Ribas", address_number: "5000", address_zip: "82400000",
      neighborhood: Neighborhood.named("Santa Felicidade").sole
    )
    by_neighborhood = Triage.where(status: "completed").group(:neighborhood_id).count
                            .transform_keys { |id| id && Neighborhood.find(id).name }
    expect(by_neighborhood).to eq("Santa Felicidade" => 6, "Boqueirão" => 3, "Centro" => 1, "Batel" => 2, nil => 3)
    expect(Citizen.all.map(&:cpf)).to all(satisfy { |cpf| CitizenIdentity::Cpf.normalize(cpf) == cpf })
    expect(Triage.distinct.pluck(:tier)).to contain_exactly("alta", "baixa")
  end

  it "é idempotente e não sobrescreve endereço editado" do
    seed
    HealthUnit.find_by!(name: "UPA 24h Centro").update!(address_street: "Rua Editada")
    counts = -> { [ Citizen.count, Conversation.count, Triage.count, NeighborhoodCoverage.count ] }
    before = counts.call

    expect(seed[:new_triages]).to eq(0)
    expect(counts.call).to eq(before)
    expect(HealthUnit.find_by!(name: "UPA 24h Centro").address_street).to eq("Rua Editada")
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod11 api bundle exec rspec spec/lib/territory_crew_spec.rb`
Expected: FAIL (`cannot load such file -- .../lib/territory_crew`).

- [ ] **Step 3: Implemente**

```ruby
# lib/territory_crew.rb
require "digest"

# Semente de dev do módulo 11 (spec 2026-09-28-module-11-territory §8). Dev é
# fictício, mas imita o real: os bairros são reais (vêm de db/seeds/territory
# pelo Territory::Seed, que roda ANTES), as unidades da semente de dev ganham
# endereço com CEP real do bairro onde ficam, e os cidadãos de demonstração
# têm bairros variados — um bairro com 5 ou mais casos (o painel mostra o
# número), outros com 1 a 4 (o painel mostra "< 5"), um bairro sem cobertura
# (sem unidade de referência) e alguns sem bairro ("Sem bairro"). Tudo pelos
# comandos do domínio: a triagem copia o bairro no StartTriage, como em
# produção. Idempotente; não sobrescreve endereço já preenchido.
class TerritoryCrew
  UNIT_ADDRESSES = {
    "curitiba" => {
      "UBS Jardim das Flores" => { street: "Avenida Manoel Ribas", number: "5000", zip: "82400000",
                                   neighborhood: "Santa Felicidade" },
      "UBS Vila Esperança" => { street: "Avenida Marechal Floriano Peixoto", number: "6601", zip: "81650000",
                                neighborhood: "Boqueirão" },
      "UPA 24h Centro" => { street: "Rua XV de Novembro", number: "500", zip: "80020310", neighborhood: "Centro" }
    },
    "maringa" => {
      "UPA 24h Centro" => { street: "Avenida Brasil", number: "3000", zip: "87013000", neighborhood: "Zona 01" },
      "UBS Jardim das Flores" => { street: "Avenida Pedro Taques", number: "800", zip: "87030000",
                                   neighborhood: "Zona 07" },
      "UBS Vila Esperança" => { street: "Avenida Morangueira", number: "1200", zip: "87033070",
                                neighborhood: "Jardim Alvorada" }
    }
  }.freeze

  # [bairro (nil = prefere não informar), quantos cidadãos]
  CITIZENS = {
    "curitiba" => [ [ "Santa Felicidade", 6 ], [ "Boqueirão", 3 ], [ "Centro", 1 ], [ "Batel", 2 ], [ nil, 3 ] ],
    "maringa" => [ [ "Zona 07", 5 ], [ "Jardim Alvorada", 2 ], [ "Zona 01", 1 ], [ nil, 2 ] ]
  }.freeze

  # Alterna triagem alta (tosse e febre) e baixa (sem tosse).
  ANSWERS = [ %w[true true], %w[false] ].freeze

  class << self
    def seed_current_city(slug:, ddd:)
      addressed = UNIT_ADDRESSES.fetch(slug, {}).count { |unit_name, address| ensure_address(unit_name, address) }
      people = CITIZENS.fetch(slug, []).flat_map { |name, n| Array.new(n, name) }
      new_triages = people.each_with_index.count { |neighborhood_name, i| ensure_citizen(slug, ddd, i, neighborhood_name) }
      { units_with_address: addressed, citizens: people.size, new_triages: new_triages }
    end

    private

    # true quando a unidade existe (com endereço novo ou já preenchido).
    def ensure_address(unit_name, address)
      unit = HealthUnit.find_by(name: unit_name)
      return false unless unit
      return true if unit.address_street.present?

      neighborhood = Neighborhood.named(address[:neighborhood]).first
      unit.update!(address_street: address[:street], address_number: address[:number], address_zip: address[:zip],
                   neighborhood: neighborhood)
      true
    end

    # true quando criou a triagem agora.
    def ensure_citizen(slug, ddd, index, neighborhood_name)
      registered = Citizens::RegisterPerson.call(phone: format("+55%s97777%04d", ddd, index + 1),
                                                 cpf: cpf_for("#{slug}:territory:#{index}"))
      raise "semente: cidadão recusado (#{registered.reason})" if registered.failure?

      citizen = registered.payload[:citizen]
      return false if Triage.joins(:conversation).where(conversations: { citizen_id: citizen.id }).exists?

      if neighborhood_name
        neighborhood = Neighborhood.named(neighborhood_name).first ||
                       raise("semente: bairro #{neighborhood_name} ausente (rode Territory::Seed antes)")
        Citizens::SetNeighborhood.call(citizen: citizen, neighborhood_id: neighborhood.id)
      end

      started = Citizens::StartConversation.call(citizen: citizen, consent_version: Consents.current_version,
                                                 session_id: "seed-territory")
      raise "semente: conversa recusada (#{started.reason})" if started.failure?

      ANSWERS[index % ANSWERS.size].each_with_index do |answer, step|
        Citizens::SubmitAnswer.call(conversation: started.payload[:conversation], answer: answer,
                                    idempotency_key: "seed-territory-#{slug}-#{index}-#{step}")
      end
      true
    end

    # CPF com dígitos verificadores válidos, determinístico pela semente.
    def cpf_for(seed)
      base = Digest::SHA256.hexdigest(seed).scan(/\d/).join[0, 9].ljust(9, "7")
      nums = base.chars.map(&:to_i)
      first = CitizenIdentity::Cpf.check_digit(nums)
      second = CitizenIdentity::Cpf.check_digit(nums + [ first ])
      "#{base}#{first}#{second}"
    end
  end
end
```

Em `db/seeds.rb`, junto dos outros `require` do topo:

```ruby
  require Rails.root.join("lib/territory_crew").to_s
```

e, logo depois do bloco que imprime `pros[:professionals]` (módulo 10) e antes do `puts "[seeds] cidade ...`:

```ruby
        # ── Território (módulo 11, spec 2026-09-28 §8) ────────────────────────
        # Depois do ProfessionalCrew: a cobertura da semente liga as unidades dele.
        territory = Territory::Seed.call(path: Territory::Seed.path_for(slug))
        puts "[seeds] território .. #{territory.created} bairros criados, #{territory.existing} já existentes"
        territory.warnings.each { |warning| puts "[seeds] território .. aviso: #{warning}" }
        demo = TerritoryCrew.seed_current_city(slug: slug, ddd: ddd)
        puts "[seeds] território .. #{demo[:units_with_address]} unidades com endereço, " \
             "#{demo[:citizens]} cidadãos de demonstração (#{demo[:new_triages]} triagens novas)"
```

- [ ] **Step 4: Rode a spec e a semente de dev duas vezes**

Run:
```bash
docker compose exec -T -w /rails/.claude/mod11 api bundle exec rspec spec/lib/territory_crew_spec.rb spec/lib/professional_crew_spec.rb
docker compose exec -T -w /rails/.claude/mod11 api bin/rails db:seed
docker compose exec -T -w /rails/.claude/mod11 api bin/rails db:seed
```
Expected: PASS; a primeira semente imprime `15 cidadãos de demonstração (15 triagens novas)` em Curitiba e `10 ... (10 ...)` em Maringá; a segunda, `(0 triagens novas)` e `0 bairros criados`.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add lib/territory_crew.rb db/seeds.rb spec/lib/territory_crew_spec.rb
/opt/homebrew/bin/git commit -m "feat: seed dev unit addresses and demo citizens across neighborhoods

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 16: Suíte de invariantes com teste de mutação, suíte completa e custo

**Files:**
- Create: `spec/invariants/territory_invariants_spec.rb`

**Interfaces:**
- Consumes: tudo das Tasks 1 a 15.

- [ ] **Step 1: Escreva a suíte**

```ruby
# spec/invariants/territory_invariants_spec.rb
require "rails_helper"

# Módulo 11, critério de fechamento (ADR 0023; spec 2026-09-28 §7.1). Cada
# bloco tem a mutação que precisa deixá-lo vermelho (registrada no relatório
# da entrega).
RSpec.describe "Invariantes do território (ADR 0023)", type: :request do
  def sql(statement) = ApplicationRecord.connection.execute(statement)

  let(:admin) { staff_with("admin@cidade.gov.br", "municipal_admin") }
  let!(:centro) { Neighborhood.create!(name: "Centro", source: "seed") }
  let!(:batel) { Neighborhood.create!(name: "Batel", source: "seed") }

  # Mutação: apagar o bloco DO $do$ de triages_neighborhood_immutable em
  # db/city_triggers.sql, ou afrouxar a exceção (ex.: tirar a condição de
  # status), e recarregar os bancos de teste.
  it "o bairro copiado na triagem só muda para NULL na revogação" do
    triage = territory_triage!(centro)
    expect { sql("UPDATE triages SET neighborhood_id = '#{batel.id}' WHERE id = '#{triage.id}'") }
      .to raise_error(ActiveRecord::StatementInvalid)
    expect { triage.update_columns(neighborhood_id: nil) }.to raise_error(ActiveRecord::StatementInvalid)
    expect(triage.reload.neighborhood_id).to eq(centro.id)

    in_progress = territory_triage!(centro, status: "in_progress")
    expect { in_progress.update_columns(status: "aborted_by_revocation", neighborhood_id: nil) }.not_to raise_error
    expect(in_progress.reload.neighborhood_id).to be_nil
  end

  # Mutação: tirar `neighborhood_id: nil` de AnonymizeRevokedTriageJob.
  it "revogar o consentimento apaga o bairro da triagem" do
    create_default_protocol!
    ConsentTerm.create!(version: "1", body: "Termo", published_at: Time.current)
    citizen = Citizen.create!(cpf: "52998224725", phone: "+5541998765432", neighborhood: centro)
    started = Citizens::StartConversation.call(citizen: citizen, consent_version: "1", session_id: "s").payload
    RevokeConsent.call(conversation: started[:conversation], reason: "citizen_web")
    AnonymizeRevokedTriageJob.new.handle(conversation_id: started[:conversation].id)
    expect(started[:triage].reload).to have_attributes(status: "aborted_by_revocation", neighborhood_id: nil)
  end

  # Mutação: tirar a checagem de ativo de Territory::ReplaceCoverage, ou o
  # `active_neighborhoods` de Citizens::SetNeighborhood.
  it "bairro inativo não entra em cobertura nem no cidadão" do
    inactive = Neighborhood.create!(name: "Ahu", source: "seed", active: false)
    unit = create_unit
    citizen = Citizen.create!(cpf: "52998224725", phone: "+5541998765432")

    expect(Territory::ReplaceCoverage.call(neighborhood: inactive, health_unit_ids: [ unit.id ], by: admin).reason)
      .to eq(:inactive_neighborhood)
    expect(Citizens::SetNeighborhood.call(citizen: citizen, neighborhood_id: inactive.id).reason)
      .to eq(:invalid_neighborhood)
    expect(inactive.coverages).to be_empty
    expect(citizen.reload.neighborhood_id).to be_nil
  end

  # Mutação: tirar `where(active: true)` de Territory::ReferenceUnits.for, ou
  # `health_units: { active: true }` de ids_by_neighborhood.
  it "a unidade de referência nunca inclui unidade inativa" do
    ativa = create_unit("UBS Centro")
    inativa = create_unit("UBS Antiga", active: false)
    [ ativa, inativa ].each { |u| NeighborhoodCoverage.create!(neighborhood: centro, health_unit: u) }

    expect(Territory::ReferenceUnits.for(centro.id)).to eq([ ativa ])
    expect(Territory::ReferenceUnits.ids_by_neighborhood([ centro.id ])).to eq(centro.id => [ ativa.id ])
  end

  # Mutação: tirar qualquer @filter.count / series / over / share / list de
  # uma das cinco queries (uma por vez).
  describe "com filtro, nenhum dos cinco painéis devolve número de 1 a 4" do
    panels = %w[overview classification triages reports conversations]
    # Não são contagens: régua do protocolo e prioridade clínica da amostra.
    non_count_keys = %w[urgentMaxPriority priority]

    small_numbers = lambda do |node, path = "$"|
      case node
      when Hash
        node.flat_map { |k, v| non_count_keys.include?(k) ? [] : small_numbers.call(v, "#{path}.#{k}") }
      when Array
        node.each_with_index.flat_map { |v, i| small_numbers.call(v, "#{path}[#{i}]") }
      when Numeric
        (1..4).cover?(node) ? [ "#{path}=#{node}" ] : []
      else
        []
      end
    end

    before do
      3.times { territory_report!(territory_triage!(batel)) }
      6.times { territory_report!(territory_triage!(centro)) }
      territory_report!(territory_triage!(centro, tier: "baixa", priority: 9))
      2.times { territory_report!(territory_triage!(nil)) }
      Conversation.create!(phone: "+5541911110000", state: "consented") # WhatsApp, sem cidadão
      sign_in_as(staff_with("viewer@cidade.gov.br", "viewer"))
    end

    it "a varredura acha número pequeno sem filtro (prova que a fixture o exercita)" do
      get "/admin/api/classification", params: { period: "7d" }
      expect(small_numbers.call(JSON.parse(response.body)["data"])).not_to be_empty
    end

    { "bairro com 3 casos" => :batel, "bairro com 7 casos, um deles único na categoria" => :centro,
      "sem bairro" => nil }.each do |label, which|
      it "#{label}: nenhum número de 1 a 4" do
        param = which ? public_send(which).id : "none"
        offenders = panels.flat_map do |panel|
          get "/admin/api/#{panel}", params: { period: "7d", neighborhood_id: param }
          expect(response).to have_http_status(:ok), panel
          small_numbers.call(JSON.parse(response.body)["data"], panel)
        end
        expect(offenders).to eq([])
      end
    end
  end

  # Mutação: fazer Territory::Seed casar pelo nome em vez da seed_key, atualizar
  # o bairro existente (ex.: `active: true`) ou recriar a cobertura.
  it "a semente é idempotente e não desfaz edição (nem renomear)" do
    unit = create_unit("UBS Centro")
    dir = Pathname(Dir.mktmpdir)
    path = dir.join("cidade.yml")
    path.write("neighborhoods:\n  - name: Portão\n    key: portao\n    units: [\"UBS Centro\"]\n  - name: Ahú\n    key: ahu\n    units: [\"UBS Centro\"]\n")

    Territory::Seed.call(path: path)
    snapshot = -> { [ Neighborhood.order(:name).pluck(:name, :active, :source), NeighborhoodCoverage.count ] }
    before = snapshot.call
    Territory::Seed.call(path: path)
    expect(snapshot.call).to eq(before)

    portao = Neighborhood.named("Portão").sole
    ahu = Neighborhood.named("Ahú").sole
    Territory::SetNeighborhoodActive.call(neighborhood: portao, active: false, by: admin)
    Territory::RenameNeighborhood.call(neighborhood: portao, name: "Portão Velho", by: admin)
    Territory::ReplaceCoverage.call(neighborhood: ahu, health_unit_ids: [], by: admin)
    Territory::Seed.call(path: path)

    expect(Neighborhood.count).to eq(before.first.size)
    expect(portao.reload).to have_attributes(name: "Portão Velho", active: false)
    expect(ahu.reload.coverages).to be_empty
    expect(unit.reload).to be_active
  ensure
    FileUtils.remove_entry(dir) if dir
  end

  # Mutação: pôr uma URL do serviço de CEP em qualquer arquivo de app/.
  it "o api nunca chama serviço de CEP" do
    offenders = Dir[Rails.root.join("{app,lib,config,db}/**/*").to_s].select do |path|
      File.file?(path) && File.binread(path).match?(/viacep/i)
    end
    expect(offenders).to eq([])
  end

  # Mutação: pôr `name: neighborhood.name` no payload de neighborhood.created,
  # ou `cpf: citizen.cpf` no de citizen.neighborhood_changed.
  it "eventos do território carregam só ids" do
    unit = create_unit("UBS Centro")
    citizen = Citizen.create!(cpf: "52998224725", phone: "+5541998765432")
    created = Territory::CreateNeighborhood.call(name: "Santa Felicidade", by: admin).payload[:neighborhood]
    Territory::RenameNeighborhood.call(neighborhood: created, name: "Santa Felicidade Velha", by: admin)
    Territory::ReplaceCoverage.call(neighborhood: created, health_unit_ids: [ unit.id ], by: admin)
    Territory::SetNeighborhoodActive.call(neighborhood: batel, active: false, by: admin)
    Territory::SetNeighborhoodActive.call(neighborhood: batel, active: true, by: admin)
    Citizens::SetNeighborhood.call(citizen: citizen, neighborhood_id: created.id)

    payloads = DomainEvent.where("name LIKE 'neighborhood.%' OR name = 'citizen.neighborhood_changed'")
                          .map { |e| e.payload.to_json }
    expect(payloads.size).to eq(6)
    [ "Santa Felicidade", "Batel", "UBS Centro", citizen.cpf, citizen.phone ].each do |secret|
      expect(payloads).to all(satisfy { |p| !p.include?(secret) }), "vazou #{secret}"
    end
  end
end
```

Nota: `public_send(which)` chama o `let` (`batel`/`centro`) pelo nome; é o jeito de iterar exemplos sobre `let` sem constante.

- [ ] **Step 2: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod11 api bundle exec rspec spec/invariants/territory_invariants_spec.rb`
Expected: PASS.

- [ ] **Step 3: Teste de mutação**

Para cada comentário `# Mutação:` da suíte: aplique a mutação, rode só aquele bloco, confirme que fica **vermelho**, desfaça (`/opt/homebrew/bin/git checkout -- <arquivo>`; para o trigger, recarregue com `bin/rails city:test_databases` antes e depois). No bloco dos painéis, faça pelo menos três mutações separadas: tirar o `@filter.count` de `kpi_done` (Visão geral), o `@filter.share` de `by_mode` (Classificação) e o `@filter.over` de `avg_complete_minutes` (Conversas). Registre no relatório uma linha por mutação: arquivo, mudança, exemplo que falhou, mensagem. Mutação que passar verde = a spec não prova a invariante: conserte a spec antes de seguir.

- [ ] **Step 4: Regressão, suíte completa e custo por exemplo**

```bash
docker compose exec -T -w /rails/.claude/mod11 api bundle exec rspec spec/invariants spec/architecture
docker compose stop worker
docker compose exec -T -w /rails/.claude/mod11 api bundle exec rspec --profile 10
docker compose start worker
```
Expected: `spec/invariants` e `spec/architecture` verdes (inclusive `current_city_assignment_spec.rb`, `appointment_invariants_spec.rb`, `health_unit_invariants_spec.rb`, `professional_invariants_spec.rb`); suíte completa com 0 falhas. Custo: divida o "Finished in N seconds" pelo número de exemplos — com o host calmo, perto de **0,10 s/exemplo**. Acima de ~0,12 s, ou mais de ~3 min no total, investigue pelo `--profile` antes do merge (candidatos: `waiting_attendance` e `territory_triage!` em laço).

- [ ] **Step 5: Busca por data fixa contra o relógio real**

```bash
/opt/homebrew/bin/git diff origin/main --name-only -- spec | xargs grep -nE "Time\.zone\.parse\(\"20|Date\.new\(20|\"20[0-9]{2}-[0-9]{2}-[0-9]{2}"
```
Expected: nenhuma ocorrência nos arquivos novos.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git add spec/invariants/territory_invariants_spec.rb
/opt/homebrew/bin/git commit -m "test: add territory invariants suite with mutation evidence

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 17: Revisão final do api

- [ ] **Step 1:** Um subagente revisor lê `origin/main..HEAD` (código de produção, migração, trigger, sementes) contra a spec inteira, o ADR 0023 e a seção "Desvios da spec", com atenção a:
  - todo `DomainEvents.publish` novo declarado no initializer e sem nome/CPF/telefone — conferir por multilinha: `grep -rn -A3 "DomainEvents.publish(" app | grep -oE '"[a-z_]+\.[a-z_]+"' | sort -u` contra os binds;
  - nenhuma atribuição de `Current.city` em `app/` ou `lib/`;
  - `POST /citizen/conversations`: consentimento → validação do bairro → `RegisterPerson` → `SetNeighborhood` → `StartConversation`, nessa ordem;
  - N+1 em `GET /territory/neighborhoods`, na fila da unidade e nos painéis;
  - nenhuma menção a serviço de CEP em `app/`, `lib/`, `config/`, `db/`;
  - `/r/:token` sem `reference_units` nem bairro;
  - rotas de `/admin/api` continuam só `GET`.
- [ ] **Step 2:** Corrija os achados, rode de novo a suíte completa (worker parado) e escreva o relatório da entrega: commits, contagem e tempo da suíte (s/exemplo), evidência de mutação, specs existentes alteradas e por quê.
- [ ] **Step 3:** Para a prova no navegador (planos do dashboard e do wpda), suba o api do worktree dentro do container, na porta 3031, sem derrubar o servidor principal:

  ```bash
  docker compose exec -d -w /rails/.claude/mod11 api bin/rails server -b 0.0.0.0 -p 3031 -P tmp/pids/server-mod11.pid
  ```

  O Vite do worktree de cada front aponta o proxy para `http://api:3031` (`VITE_API_PROXY_TARGET`). Login e step-up são feitos pelo usuário.
- [ ] **Step 4:** **Pare.** Merge, push, board e docs só com autorização explícita do usuário, uma etapa de cada vez. Ordem: api antes do dashboard e do wpda. Rollout (runbook `operacao/rollout-territorio.md`, no docs): (1) imagem do api + `city:migrate:all`; (2) `city:territory:seed[slug]` nas cidades com semente; (3) dashboard e wpda. Ao voltar o checkout para a main: `DROP DATABASE` dos dois bancos de teste de cidade e `city:test_databases` de novo.

---

## Self-review (feito ao escrever o plano)

**Cobertura da spec:**

| Spec | Task |
|---|---|
| §3.1 `neighborhoods`, §3.2 `neighborhood_coverages`, §3.3 colunas e trigger | 1 |
| §3.4 semente por cidade e rake (`:seed[slug]`, `:seed:all`, arquivo ausente, relatório) | 4 |
| §3.5 eventos (declarados; publicados nos comandos) | 2, 7 |
| §4.1 `/territory` (rotas, papel, recusas) | 3 |
| §4.1 `/citizen` (neighborhoods, people, conversations, people/:id/neighborhood) | 8 |
| §4.1 `reference_units` em `GET /citizen/triages/:id` (relatório público fora — desvio 1) | 9 |
| §4.1 `/attendance/units` com endereço, `invalid_zip`, `invalid_neighborhood` | 6 |
| §4.1 `reference_unit_ids` na leitura do desfecho | 10 |
| §4.1/§4.3 painéis com `neighborhood_id`, `none`, 422, `filter` | 11–14 |
| §4.2 comandos, `SetNeighborhood`, cópia no `StartTriage`, `ReferenceUnits` | 2, 7, 9 |
| §4.3 supressão (KPI, categoria, série, percentual/média, listas) | 11–13 |
| §7.1 invariantes (trigger, inativo, referência, varredura 1–4, semente, CEP) | 16 |
| §7.2 request specs, spec do rake, custo por exemplo | 3, 4, 6, 8–10, 14, 16 |
| §8 semente de dev | 4, 15 |
| §9 guarda `adr_pointers_spec` 1..23 | 1 |

Fora deste plano (docs/dashboard/wpda): ADR e módulo no repo docs, runbook de rollout, cards do board, proxy do Vite, telas.

**Placeholders:** nenhum "TBD"/"implemente depois"; cada passo de código traz o código.

**Tipos e nomes:** `Territory::ReferenceUnits.for` / `.ids_by_neighborhood` / `.as_json_list` (Tasks 9, 10, 16); `Admin::NeighborhoodFilter.off` / `.parse` e os métodos `count`/`series`/`over`/`share`/`list` (Tasks 11–14); `Citizens::SetNeighborhood.call(citizen:, neighborhood_id:)` (Tasks 7, 8, 15, 16); `Territory::Seed.call(path:)` → `Report` (Tasks 4, 15, 16); helpers `territory_triage!` / `territory_report!` (Task 1, usados em 11–13, 16).

**Review Focus:** as cinco linhas têm teste na task dona (Tasks 3, 6, 8, 9, 10, 12, 13, 14).

**Decisões do usuário incorporadas (2026-09-28):**
- Semente casa por `seed_key` (desvio 13): Tasks 1, 2, 4, 15, 16.
- Revogação apaga o bairro copiado (desvio 14): Tasks 1, 7, 16. O ADR 0023 precisa de emenda no docs ("imutável, exceto a anonimização da revogação").

**Em aberto (não bloqueia):**
- Maringá: a semente traz 44 nomes reais conferidos na base de CEP, não a lista oficial completa (centenas de loteamentos). Curitiba traz os 75 oficiais.
