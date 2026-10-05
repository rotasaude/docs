# Módulo 16 — Modo de prontuário: fundação do api Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** O lado api da fundação do módulo 16 (F-16.1, F-16.2, F-16.3, F-16.4, F-16.5, F-16.8): interruptores por cidade escritos só pelo `maintenance`, modo de prontuário e endereço do PEC no console, IBGE editado no `city_profile`, credenciais de integração cifradas com teste de conexão, terminologias CID-10/CIAP-2/SIGTAP versionadas na plataforma, retratos do CNES com casamento confirmado pela cidade, e a consulta ao CADSUS na validação presencial — tudo nascendo desligado.

**Architecture:** Duas migrações de plataforma de expansão (`cities.record_mode`/`pec_url` + `city_features`; terminologias + retratos do CNES, com trigger de imutabilidade em `db/platform_triggers.sql`) e uma de cidade (`integration_credentials`, `health_units.cnes`, `health_teams`, `health_team_members`, `professionals.cpf`, `citizens.cns`/`cadsus_*`). `Platform::Features` é o catálogo em código e a regra de "ligado / utilizável / o que falta"; ele lê a plataforma para `enabled` e o banco da cidade (credenciais e `city_profile.ibge_code`) para `missing`, degradando para `["city_unreachable"]`. Clientes externos ficam atrás de objetos pequenos (`Ledi::PecClient#login`, `Cadsus::Client.for(city)` com `simulated`/`soap_pdq`) e nunca carregam segredo em mensagem, log, evento ou argumento de job. Importadores (`terminology:import`, `cnes:import`) leem o arquivo oficial (ZIP ou pasta) por `OfficialArchive`, gravam em transação e ativam no fim. `Cnes::Proposal` é cálculo em memória sobre o retrato mais recente; `Cnes::Apply` recalcula sob trava e só aplica o que a cidade confirmou.

**Tech Stack:** Rails 8.1 (API), PostgreSQL 16 (plataforma + banco por cidade), RSpec, graphql-ruby 2.x, Active Record Encryption (chave por cidade, ADR 0007), Net::HTTP, Nokogiri, WebMock, rubyzip (novo).

**Spec:** `docs/.claude/ciclo2/superpowers/specs/2026-10-05-module-16-record-mode-and-export-design.md` e `docs/.claude/ciclo2/adr/0028.md` (leia os dois antes de começar). Contratos entre apps (fonte única de formatos, já com as decisões da sessão principal sobre IBGE, `setCityFeature` e console): `docs/.claude/ciclo2/superpowers/plans/2026-10-05-module-16-record-mode-contracts.md` — **não mude nomes, formatos nem códigos de erro**; o que o código real obrigou a precisar está no fim, em "Divergências propostas ao contrato". Pesquisa: `docs/.claude/ciclo2/pesquisa/2026-10-05-integracoes/` (frente 1 LEDI/PEC, frente 2 CADSUS/CNES, frente 4 SIGTAP). O exportador LEDI (prova técnica, fila, envio, painel de produção, `GET /city_production`, `GET /production`) é **outro plano** (`api-exporter`) e depende deste.

## Desvios da spec (e precisões de contrato)

Onde a spec é omissa, foi atualizada depois, ou o código real obrigou a escolher:

1. **IBGE fica no banco da cidade** (decisão da sessão principal; contratos §3/§4): **não nasce** `cities.ibge_code`. A fonte única é o `city_profile.ibge_code` existente, gravado pelo provisionamento (`ProvisionCityJob`) e exposto em `City.profile.ibgeCode` na API de manutenção. O `PATCH /cities/:id/record_settings` grava nele (503 `city_unreachable` se o banco não responde); `cnes:import` descobre os municípios lendo o `city_profile` de cada cidade ativa; `Platform::Features` lê o IBGE junto com as credenciais, numa conexão só.
2. **Auditoria do console é um evento só** (contratos §4.1): `city.record_settings_changed` (`city_id`, `fields`), não os três nomes da spec §3.2.
3. **`setCityFeature` escreve só na plataforma, mas a guarda exige `CityMutation`**: toda mutation com `citySlug` precisa herdar de `CityMutation` (`spec/graphql/maintenance/analyzers_spec.rb`), e `in_city` abre o banco da cidade (`CityWriter`). Como o contrato exige que o liga/desliga funcione com a cidade inalcançável, `CityMutation` ganha `on_platform` — mesma ordem de `in_city` (escopo → campos auditáveis → tentativa gravada → cidade existe e está ativa → bloco) sem abrir a cidade; a guarda de escopo passa a conferir os dois métodos. O par tentativa/resultado é `maintenance.city.feature_changed` (`MaintenanceAudit`), e o fato é `city.feature_changed` (`Platform.audit`, payload do contrato §6). `unknown_feature` sai pela regra de campo auditado (`AUDITED_FIELD_RULES[:feature_key]`), depois da checagem de escopo; cidade inexistente sai como a de toda mutation de cidade (`path: "citySlug"`, mensagem existente "cidade inexistente").
4. **`record_mode` e `pec_url` vêm sempre da plataforma, sem o cache do catálogo**: `Current.city` sai de `CityCatalog` (cache de 30 s). Quem decide (`Platform::Features`, `GET /integrations`) relê `cities` por id (`City.where(id:).pick`).
5. **Segredo da credencial é `text`, não `jsonb`**: Active Record Encryption grava um envelope; o Hash `{ "username", "password" }` é serializado em JSON (`serialize :secret, coder: JSON`) e depois cifrado com a chave da cidade.
6. **CADSUS pendente até a confirmação** (contratos §5.4 vence a spec §7): a consulta grava em `citizens.cadsus_pending_cns` (cifrado), `cadsus_pending_session_id` e `cadsus_pending_at`; `Citizens::Verify` com `cadsus_confirmed: true` exige consulta da **mesma sessão** há no máximo 10 min, copia para `citizens.cns`/`cadsus_checked_at` e limpa o pendente. Sem consulta válida: 409 `cadsus_lookup_missing` e nada é validado. Nascimento e sexo do CADSUS nunca são gravados — só comparados em memória. O Rails cache não serve: o Solid Cache mora no banco de **plataforma**, que não guarda dado de cidadão.
7. **`POST /attendance/cadsus_lookup` recebe `{ cpf, code }`** (contratos §5.4), e o par sai de `Citizens::VerificationCodeMatch.call(cpf:, code:)` sem consumir o código. Falha do par segue exatamente os erros de `POST /attendance/lookup` (422 `invalid_cpf`, `invalid_code`, `code_expired`, `code_exhausted`) — o `404 not_found` do contrato não tem caso que o produza; "CADSUS não achou" é 200 com `found: false`. Ver Divergências.
8. **`birth_date_matches`/`sex_matches` são `null` na main**: o perfil declarado nasce no módulo 15 (em paralelo). `Cadsus::Lookup` compara com `citizen.try(:birth_date)`/`citizen.try(:sex)`; sem perfil, `null`. Quando o 15 entrar, a comparação passa a valer sem mudança aqui.
9. **Interruptor desligado × não utilizável** (contratos §5.4): desligado → 403 `{ "error": "feature_disabled", "feature": "cadsus_lookup" }`; ligado mas sem credencial, com credencial recusada, cidade sem estado legível ou serviço fora → 503 `cadsus_unavailable`. A guarda genérica (`FeatureGate#require_feature!`) só cuida do 403; o 503 é da rota. `cadsus_confirmed` no `POST /attendance/verifications` é opcional (ausente = `false`). Respostas 200 de escrita na cidade são o objeto sem envelope (o item de credencial, `{ applied, skipped }`, `{ found, ... }`).
10. **Formatos oficiais não conferidos na fonte** (pesquisa marcou [INFERÊNCIA]): o leitor da SIGTAP lê o **layout que vem no próprio ZIP** (`<arquivo>_layout.txt`), então tamanho de campo mudado não quebra; os nomes de arquivo `tb_procedimento`, `rl_procedimento_ocupacao`, `rl_procedimento_cid`, `rl_procedimento_registro`, `tb_registro` e a idade em **meses** (9999 = sem limite) vêm de projetos abertos. CID-10: os CSV do DATASUS (`CID-10-CATEGORIAS.CSV`, `CID-10-SUBCATEGORIAS.CSV`, `;`, ISO-8859-1). CIAP-2: CSV `codigo;titulo`. CNES: CSVs da base mensal (`tbEstabelecimento`, `tbEquipe`, `rlEstabEquipeProf`, `tbCargaHorariaSus`, `tbDadosProfissionalSus`, `;`, ISO-8859-1), lidos **por nome de coluna** num lugar só (`Cnes::BaseLayout`); o município no CNES tem 6 dígitos (IBGE sem o verificador). Na primeira importação real, o operador confere; se um nome divergir, muda só a constante.
11. **Login no PEC em JSON** `{ "usuario", "senha" }` → cookie `JSESSIONID` (pesquisa frente 1). A prova técnica do exportador confirma o formato; `Ledi::PecClient` é o único lugar que o conhece.
12. **CNES do profissional na plataforma é cifrado**: `cnes_professional_bonds.cpf`/`cns` com a chave da plataforma (`PlatformKeyProvider`, como `City#database_url`); o casamento decifra em memória (retrato de um município cabe em memória). Nenhum nome de profissional é gravado na plataforma.
13. **Propostas de equipe só aparecem para unidade já casada** (CNES gravado na unidade): confirmar a unidade primeiro, e as equipes dela aparecem na próxima leitura. Criar unidade só é proposto para estabelecimento com equipe ativa (a base do município lista clínicas privadas e hospitais que não são da APS).
14. **`POST /cnes/apply` pode pular por `conflict`**, além de `stale`: proposta de criar unidade cujo nome já existe (`unit_name_taken`), por exemplo. Ver Divergências.
15. **Profissional e unidade ganham campos nas respostas existentes**: `profile_json` ganha `cpf_masked` (e `cpf` na ficha completa); `unit_json` ganha `cnes`; `POST /professionals[/:id]` aceita `cpf` (o admin; o próprio profissional não) e `POST /attendance/units[/:id]` aceita `cnes`, com 422 `invalid_cnes`/`cnes_taken` na unidade e 409 `cpf_taken` no profissional (como `cns_taken`). Ver Divergências.
16. **Alerta de SIGTAP** (spec §4): `Terminology::SigtapStatus.call(today:)` → `{ sigtap_current_competence:, sigtap_imported:, alert: }` (`alert` = dia ≥ 5 sem release ativa da competência corrente). Quem o publica é `GET /city_production`, do plano do exportador.

## Global Constraints

- Plataforma: migração em `db/platform_migrate`, `db/platform_schema.rb` regravado pelo `db:migrate` (em development **e** em test; `db:migrate:platform` não existe), trigger em `db/platform_triggers.sql` (fonte única, idempotente; a migração que o muda termina com `execute File.read(Rails.root.join("db/platform_triggers.sql"))`; o dump não representa trigger). O banco de plataforma de dev/test é **compartilhado** com a main: só expansão.
- Cidade: migração em `db/city_migrate`, dump à mão em `db/city_schema.rb` (o juiz é `spec/services/city_schema_spec.rb`), triggers em `db/city_triggers.sql`. Depois da migração de cidade: `DROP DATABASE` de `rota_saude_test_city_a` e `rota_saude_test_city_b` + `bin/rails city:test_databases`. Rollout roda `city:migrate:all`; **nunca** migrar cidade fora do rake (cidade trava em 503).
- CHECKs na forma que o dump reproduz (`col::text = ANY (ARRAY['a'::text, ...])` na plataforma, api#38; `ARRAY['a', 'b']::text[]` na cidade, como as migrações de cidade existentes).
- Valores, exatamente: `record_mode` ∈ `off`, `integrated`, `record` (default `off`); `pec_url` só `https://`, sem usuário/senha, sem query/fragmento, até 255; IBGE 7 dígitos; chaves de interruptor `ledi_export`, `cadsus_lookup`; `kind` de credencial `ledi`, `cadsus`; `last_check_status` ∈ `ok`, `unauthorized`, `unreachable`, `error`; `kind` de terminologia `cid10`, `ciap2`, `sigtap` (SIGTAP `version` = `AAAAMM`); `status` de release `importing`, `active`, `superseded`, `failed`; `health_teams.kind` ∈ `70`, `76`; INE 10 dígitos; CNES 7 dígitos; CBO 6 dígitos.
- Códigos de `missing`, exatamente (contratos §2): `record_mode_off`, `pec_url_missing`, `ibge_code_missing`, `credential_missing:ledi`, `credential_unauthorized:ledi`, `credential_missing:cadsus`, `credential_unauthorized:cadsus`; cidade inalcançável → `["city_unreachable"]` (e `usable: false`).
- Credencial, `password`, `secret` e CPF/CNS nunca em resposta, log, evento, auditoria, mensagem de exceção ou argumento de job. `filter_parameters` ganha `:cns` (`:passw`, `:secret`, `:cpf` já estão). Segredo, `professionals.cpf`, `citizens.cns` e `citizens.cadsus_pending_cns` cifrados com a chave da cidade e listados em `CityEncryption::CITY_KEYED_TARGETS` (`spec/architecture/city_encrypted_attributes_guard_spec.rb`).
- `Platform.audit` com nome novo exige o nome em `R18_PLATFORM_EVENT_NAMES` (`spec/events/platform_event_payload_guard_spec.rb`); nome de `MaintenanceAudit` novo exige `NAMES` + branch do `case`. Nomes deste plano: `city.feature_changed`, `city.record_settings_changed`, `terminology.release_activated`, `cnes.snapshot_imported`, `maintenance.city.feature_changed`. Payload só com ids e nomes de campo; nenhuma chave com `name`, `email`, `cpf`, `phone`, `body`.
- Eventos de domínio novos (cidade), declarados em `config/initializers/domain_events.rb` com `to: []` e conferidos em `spec/initializers/domain_events_bindings_spec.rb`: `integration_credential.changed` `{ kind, user_id }`, `cnes.proposals_applied` `{ user_id, count }`, `citizen.cadsus_looked_up` `{ citizen_id, user_id, found }`.
- A API de manutenção só existe em development, staging e test (`MaintenanceApi.enabled?`); em produção todo interruptor fica desligado. Campo de raiz novo entra em `HumanOnly::RESTRICTED`; tipo e campo novos entram em `spec/architecture/maintenance_schema_spec.rb`.
- `Current.city` **nunca** é atribuído em `app/` ou `lib/` (`spec/architecture/current_city_assignment_spec.rb`); job, rake e semente usam `CityConnection.with`.
- Erros sempre `{ "error": "<reason>" }`; step-up pelo `MfaStepUp` (401 `mfa_required`); papel ausente 403 `missing_role`.
- Specs de request precisam de `type: :request` (inferência desligada). Arquivo novo em `spec/support/` precisa de `require_relative` em `spec/rails_helper.rb`. Spec que usa `stub_request` faz `require "webmock/rspec"` (o WebMock já bloqueia rede real na suíte).
- `spec/adr_pointers_spec.rb`: `VALID_RANGE = (1..28)` (Task 1, antes de qualquer comentário citar o ADR 0028).
- Tudo nasce desligado: `record_mode = off`, nenhuma linha em `city_features`, nenhuma credencial.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Nunca `git add -A`: caminhos explícitos.
- O módulo 15 corre em paralelo (`feat/mod-15-triage-catalog`) e mexe em `app/commands/citizens/verify.rb`, `app/commands/citizens/erase.rb`, `db/city_schema.rb` (versão) e `spec/adr_pointers_spec.rb` (`1..27`). Este plano parte de `origin/main`; no merge, as duas mudanças se somam (a faixa fica `1..28`, a versão do dump fica a maior).

## Ambiente de execução

- Antes de tocar `apps/api`, avise a sessão dona do api (sessão "API"). Ordem: `contracts` (`session-v1.1.0`) → este plano → `api-exporter`; `maintenance`, `admin` e as telas de Integrações/CNES/CADSUS do `dashboard` podem começar depois deste.
- Worktree (da raiz do monorepo, `/Users/eduardovrocha/Development/ioit.solutions/rota-saude`):

  ```bash
  /opt/homebrew/bin/git -C apps/api fetch origin
  /opt/homebrew/bin/git -C apps/api worktree add .claude/mod16 -b feat/mod-16-record-mode origin/main
  cp apps/api/config/master.key apps/api/.claude/mod16/config/master.key
  ```

- `./apps/api` é montado em `/rails` no container `api`; o worktree é `/rails/.claude/mod16`. Todo comando roda no container, da raiz do monorepo:

  ```bash
  docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec <arquivos>
  ```

- Todo `git add`/`git commit` usa `-C apps/api/.claude/mod16`.
- Gem nova (Task 11): `docker compose exec -T -w /rails/.claude/mod16 api bundle install` instala no container em execução; depois do merge, `docker compose build api` para a imagem de dev.
- Suíte completa só com o worker parado e sem outra sessão rodando suíte completa:

  ```bash
  docker compose stop worker
  docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec
  docker compose start worker
  ```

- Ao voltar para a main depois da migração de cidade: repita o DROP + `city:test_databases` (o banco de teste fica à frente e a paridade da main quebra sem culpa).

## Review Focus

1. **Operador cola um endereço do PEC "quase certo"** — `http://`, `https://admin:senha@pec...`, com `?x=1`, só espaço, ou `""` para limpar — e o IBGE com 6 dígitos ou com pontuação: 422 com o código certo e nada gravado em lugar nenhum (nem a parte que era válida); `""`/`null` limpam. Teste: Task 6 ("tabela de corpos inválidos não grava nada").
2. **PEC da cidade fora do ar, lento, ou atrás de um proxy que responde 200 em HTML sem cookie**: o teste de conexão termina em segundos com `unreachable`/`error`, a mensagem não traz senha nem corpo da resposta, e a credencial continua lá. Teste: Task 7 (timeout, 200 sem cookie) e Task 9 ("PEC lento e 200 sem cookie").
3. **A mesma base do CNES importada duas vezes, ou uma 14ª competência**: a segunda importação da mesma competência substitui (não duplica), e a mais antiga além de 13 some; uma cidade com IBGE ainda vazio ou banco fora do ar não derruba a importação das outras. Teste: Task 15 ("reimportar substitui; 14ª competência apaga a mais antiga; cidade inalcançável é pulada").
4. **Dois administradores confirmam as mesmas propostas ao mesmo tempo, ou o cadastro mudou depois da leitura**: um aplica, o outro recebe `skipped: stale`, e nunca nasce membro de equipe duplicado. Teste: Task 18 (duas threads; proposta de unidade editada à mão depois da leitura).
5. **Atendente consulta o CADSUS, o cidadão volta mais de 10 minutos depois com um código novo, ou a confirmação sai de outra aba/sessão**: 409 `cadsus_lookup_missing`, a validação não acontece e o código não é consumido. Teste: Task 19 ("consulta vencida ou de outra sessão").

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| `db/platform_migrate/20261005200001_add_record_mode_to_cities.rb`, `db/platform_schema.rb`, `app/models/city.rb`, `app/models/city_feature.rb`, `app/services/platform/features.rb` (catálogo), `spec/adr_pointers_spec.rb` | modo, PEC e tabela de interruptores | 1 |
| `db/city_migrate/20261005200001_add_record_mode_foundation.rb`, `db/city_schema.rb`, `app/models/integration_credential.rb`, `app/models/health_team.rb`, `app/models/health_team_member.rb`, `app/models/health_unit.rb`, `app/models/professional.rb`, `app/models/citizen.rb`, `app/services/city_encryption.rb`, `config/initializers/filter_parameter_logging.rb` | dados da cidade | 2 |
| `app/services/platform/features.rb`, `spec/events/platform_event_payload_guard_spec.rb` | ligado / utilizável / o que falta | 3 |
| `app/graphql/maintenance/mutations/set_city_feature.rb`, `.../city_mutation.rb`, `.../types/city_feature_type.rb`, `.../types/city_type.rb`, `.../types/mutation_type.rb`, `.../analyzers/human_only.rb`, `app/events/maintenance_audit.rb` | interruptor no maintenance | 4 |
| `app/controllers/sessions_controller.rb` | `features` na sessão | 5 |
| `app/commands/update_city_record_settings.rb`, `app/controllers/operators/cities_controller.rb`, `config/routes.rb` | console | 6 |
| `app/services/ledi/pec_client.rb` | login no PEC | 7 |
| `app/services/cadsus/client.rb`, `app/services/cadsus/simulated.rb`, `app/services/cadsus/soap_pdq.rb`, `config/environments/{development,test}.rb` | cliente CADSUS | 8 |
| `app/commands/integrations/set_credential.rb`, `app/commands/integrations/check_connection.rb`, `app/controllers/integrations_controller.rb`, `app/policies/integration_policy.rb` | Integrações | 9 |
| `db/platform_migrate/20261005200002_create_terminologies.rb`, `db/platform_triggers.sql`, `app/models/terminology_release.rb` e tabelas de código | terminologias (dados) | 10 |
| `Gemfile`, `app/services/official_archive.rb`, `app/services/terminology/import.rb`, `.../cid10_reader.rb`, `.../ciap2_reader.rb`, `lib/tasks/terminology.rake` | importação CID-10/CIAP-2 | 11 |
| `app/services/terminology/sigtap_reader.rb`, `lib/sigtap_sample.rb` | importação SIGTAP | 12 |
| `app/services/terminology/sigtap.rb`, `app/services/terminology/sigtap_status.rb` | compatibilidade e alerta | 13 |
| `db/platform_migrate/20261005200003_create_cnes_snapshots.rb`, modelos `cnes_*`, `app/services/cnes/snapshot_writer.rb` | retratos (dados) | 14 |
| `app/services/cnes/base_layout.rb`, `app/services/cnes/base_reader.rb`, `app/services/cnes/import.rb`, `lib/tasks/cnes.rake`, `spec/fixtures/cnes/` | importação do CNES | 15 |
| `app/controllers/health_units_controller.rb`, `app/controllers/concerns/professional_rendering.rb`, `app/commands/professionals/create.rb` | edição manual de CNES e CPF | 16 |
| `app/services/cnes/proposal.rb` | propostas e divergências | 17 |
| `app/commands/cnes/apply.rb`, `app/controllers/cnes_controller.rb` | confirmar | 18 |
| `app/commands/cadsus/lookup.rb`, `app/controllers/concerns/feature_gate.rb`, `app/controllers/attendance_controller.rb`, `app/commands/citizens/verify.rb` | CADSUS no balcão | 19 |
| `app/commands/citizens/erase.rb`, `spec/invariants/record_mode_invariants_spec.rb` | LGPD e invariantes | 20 |
| `lib/record_mode_crew.rb`, `db/seeds.rb` | semente de dev | 21 |
| — | verificação final | 22 |

---

## Fatia 1 — Interruptores e modo (F-16.1, F-16.2)

### Task 1: `cities.record_mode`/`pec_url`, `city_features` e a faixa de ADR

**Files:**
- Create: `db/platform_migrate/20261005200001_add_record_mode_to_cities.rb`, `app/models/city_feature.rb`, `app/services/platform/features.rb`
- Modify: `db/platform_schema.rb` (gerado), `app/models/city.rb`, `spec/adr_pointers_spec.rb`
- Test: `spec/models/city_record_settings_spec.rb`, `spec/models/add_record_mode_to_cities_migration_spec.rb`

**Interfaces:**
- Produces: `City::RECORD_MODES == %w[off integrated record]`; `City.valid_pec_url?(value) -> Boolean`; `CityFeature` (`city_id`, `key`, `enabled`, `changed_by_maintainer_id`, `changed_at`; `belongs_to :city`, `belongs_to :changed_by_maintainer`); `Platform::Features::Entry` (`key`, `description`, `requires`), `Platform::Features::CATALOG`, `Platform::Features::KEYS == %w[ledi_export cadsus_lookup]`, `Platform::Features.find(key) -> Entry|nil`.

- [ ] **Step 1: Suba a faixa de ADR**

Em `spec/adr_pointers_spec.rb`: no comentário do topo, `O v2 vai de 0001 a 0026 (` passa a `O v2 vai de 0001 a 0028 (` e a lista termina em `0026 exclusão do cadastro e revogação, 0027 catálogo de triagens por perfil, 0028 modo de prontuário e exportação LEDI)`; `VALID_RANGE = (1..26).freeze` passa a `VALID_RANGE = (1..28).freeze`; o título `(0001..0026)` passa a `(0001..0028)`.

- [ ] **Step 2: Escreva a spec que falha**

```ruby
# spec/models/city_record_settings_spec.rb
require "rails_helper"

# ADR 0028 (spec 2026-10-05 §3): modo de prontuário e endereço do PEC na
# plataforma; interruptores por cidade, um por chave do catálogo.
RSpec.describe "Modo de prontuário e interruptores da cidade" do
  let(:city) { create(:city) }
  let(:maintainer) do
    Maintainer.create!(email_address: "m-#{SecureRandom.hex(3)}@rotasaude.app", password: "s3nha-forte-1",
                       otp_secret: ROTP::Base32.random, otp_enabled_at: Time.current)
  end

  it "nasce off e sem PEC" do
    expect(city.record_mode).to eq("off")
    expect(city.pec_url).to be_nil
  end

  it "recusa modo fora da lista, no modelo e no banco" do
    city.record_mode = "parcial"
    expect(city).not_to be_valid
    expect { City.where(id: city.id).update_all(record_mode: "parcial") }.to raise_error(ActiveRecord::StatementInvalid)
  end

  it "aceita só https sem credencial, query ou fragmento" do
    ok = %w[https://pec.cidade.gov.br https://pec.cidade.gov.br:8443/esus]
    bad = [ "http://pec.cidade.gov.br", "https://admin:senha@pec.cidade.gov.br", "https://pec.cidade.gov.br/?a=1",
            "https://pec.cidade.gov.br/#x", "https://", "pec.cidade.gov.br", " ", "https://#{'a' * 250}.br" ]
    ok.each { |url| expect(City.valid_pec_url?(url)).to be(true), url }
    bad.each { |url| expect(City.valid_pec_url?(url)).to be(false), url }
    city.pec_url = "http://pec.cidade.gov.br"
    expect(city).not_to be_valid
    expect { City.where(id: city.id).update_all(pec_url: "http://x") }.to raise_error(ActiveRecord::StatementInvalid)
  end

  it "um interruptor por cidade e chave; chave fora do catálogo é inválida" do
    CityFeature.create!(city: city, key: "cadsus_lookup", enabled: true, changed_by_maintainer: maintainer,
                        changed_at: Time.current)
    expect {
      CityFeature.new(city: city, key: "cadsus_lookup", changed_by_maintainer: maintainer, changed_at: Time.current)
                 .save!(validate: false)
    }.to raise_error(ActiveRecord::RecordNotUnique)
    expect(CityFeature.new(city: city, key: "rnds", changed_by_maintainer: maintainer, changed_at: Time.current))
      .not_to be_valid
  end

  it "catálogo em código, com as duas chaves" do
    expect(Platform::Features::KEYS).to eq(%w[ledi_export cadsus_lookup])
    expect(Platform::Features.find("cadsus_lookup").requires).to eq(%w[credential:cadsus])
    expect(Platform::Features.find("nada")).to be_nil
  end
end
```

- [ ] **Step 3: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec spec/models/city_record_settings_spec.rb`
Expected: FAIL (`record_mode` indefinido / `uninitialized constant CityFeature`).

- [ ] **Step 4: Migração, modelos e catálogo**

```ruby
# db/platform_migrate/20261005200001_add_record_mode_to_cities.rb
# ADR 0028 (spec 2026-10-05 §3): modo de prontuário e endereço do PEC por
# cidade, e os interruptores por cidade. O IBGE NÃO nasce aqui: a fonte única é
# city_profile.ibge_code, no banco da cidade (contratos §3). Só expansão. A
# chave do interruptor é conferida contra o catálogo em código
# (Platform::Features::CATALOG), não por CHECK: chave nova não pede migração.
class AddRecordModeToCities < ActiveRecord::Migration[8.1]
  RECORD_MODES = %w[off integrated record].freeze

  # Forma que o Postgres devolve igual ao reler o próprio texto (api#38).
  def self.record_mode_check
    "record_mode::text = ANY (ARRAY[#{RECORD_MODES.map { |m| "'#{m}'::text" }.join(', ')}])"
  end

  def change
    add_column :cities, :record_mode, :string, null: false, default: "off"
    add_column :cities, :pec_url, :string, limit: 255
    add_check_constraint :cities, self.class.record_mode_check, name: "ck_cities_record_mode"
    add_check_constraint :cities, "pec_url IS NULL OR pec_url::text ~ '^https://'::text", name: "ck_cities_pec_url_https"

    create_table :city_features, id: :uuid, default: -> { "gen_random_uuid()" } do |t|
      t.uuid :city_id, null: false
      t.string :key, null: false
      t.boolean :enabled, null: false, default: false
      t.uuid :changed_by_maintainer_id, null: false
      t.datetime :changed_at, null: false
      t.timestamps
      t.index %i[city_id key], unique: true, name: "idx_city_features_city_key"
    end
    add_foreign_key :city_features, :cities
    add_foreign_key :city_features, :maintainers, column: :changed_by_maintainer_id
  end
end
```

Em `app/models/city.rb`, depois de `TIME_ZONES`:

```ruby
  # ADR 0028 (spec 2026-10-05 §3.2): o modo é decisão de contrato, do operador
  # no console. Toda cidade nasce off.
  RECORD_MODES = %w[off integrated record].freeze
  PEC_URL_MAX = 255
```

depois de `validates :time_zone, ...`:

```ruby
  validates :record_mode, inclusion: { in: RECORD_MODES }
  validate { errors.add(:pec_url, :invalid) unless pec_url.nil? || self.class.valid_pec_url?(pec_url) }

  has_many :features, class_name: "CityFeature", dependent: :restrict_with_error

  # Endereço do PEC da cidade (ADR 0028): só HTTPS, sem usuário/senha na URL
  # (credencial é da cidade, cifrada no banco dela), sem query nem fragmento.
  def self.valid_pec_url?(value)
    return false unless value.is_a?(String) && value.length <= PEC_URL_MAX

    uri = URI.parse(value)
    uri.is_a?(URI::HTTPS) && uri.host.present? && uri.userinfo.nil? && uri.query.nil? && uri.fragment.nil?
  rescue URI::InvalidURIError
    false
  end
```

```ruby
# app/models/city_feature.rb
# Interruptor de funcionalidade por cidade (ADR 0028; spec 2026-10-05 §3.1), no
# banco de PLATAFORMA. Só o maintenance escreve (Platform::Features.set!).
class CityFeature < PlatformRecord
  belongs_to :city
  belongs_to :changed_by_maintainer, class_name: "Maintainer"

  validates :key, inclusion: { in: ->(_) { Platform::Features::KEYS } }
  validates :changed_at, presence: true
end
```

```ruby
# app/services/platform/features.rb
# Interruptores de funcionalidade por cidade (ADR 0028; spec 2026-10-05 §3.1;
# contratos §2). O catálogo mora no código; o estado por cidade, em
# city_features (plataforma). `requires` diz o que precisa existir para a
# funcionalidade ser UTILIZÁVEL; ligado e utilizável são coisas diferentes.
module Platform
  module Features
    Entry = Data.define(:key, :description, :requires)

    CATALOG = [
      Entry.new(key: "ledi_export",
                description: "Exportação contínua da produção (LEDI APS) para o PEC da cidade",
                requires: %w[record_mode pec_url ibge_code credential:ledi]),
      Entry.new(key: "cadsus_lookup",
                description: "Consulta ao CADSUS na validação presencial",
                requires: %w[credential:cadsus])
    ].freeze

    KEYS = CATALOG.map(&:key).freeze

    module_function

    def find(key) = CATALOG.find { |entry| entry.key == key.to_s }
  end
end
```

`app/events/platform.rb` já define `module Platform`; `app/services/platform/` é o mesmo namespace em outra raiz (o Zeitwerk aceita). A Task 22 roda `zeitwerk:check`.

- [ ] **Step 5: Migre a plataforma em development e em test**

```bash
docker compose exec -T -w /rails/.claude/mod16 api bin/rails db:migrate
docker compose exec -T -w /rails/.claude/mod16 -e RAILS_ENV=test -e POSTGRES_PASSWORD=postgres api bin/rails db:migrate
/opt/homebrew/bin/git -C apps/api/.claude/mod16 diff --stat db/platform_schema.rb
```
Expected: `define(version: 2026_10_05_200001)`, em `cities` as colunas `pec_url` (limit 255) e `record_mode` (default "off") e os CHECKs `ck_cities_pec_url_https`/`ck_cities_record_mode`; `create_table "city_features"` com o índice único; `add_foreign_key "city_features", "cities"` e `"city_features", "maintainers", column: "changed_by_maintainer_id"`. Diff com outra coisa: **pare** e descubra de onde veio.

- [ ] **Step 6: Spec de down e up**

```ruby
# spec/models/add_record_mode_to_cities_migration_spec.rb
require "rails_helper"
require Rails.root.join("db/platform_migrate/20261005200001_add_record_mode_to_cities.rb").to_s

# ADR 0028: a migração de plataforma é reversível — down remove colunas, CHECKs
# e a tabela; up restaura o schema idêntico. Num savepoint da plataforma.
RSpec.describe "Migração de plataforma 20261005200001 (AddRecordModeToCities): down e up" do
  def conn = PlatformRecord.connection

  def migrate(direction)
    ActiveRecord::Migration.suppress_messages { AddRecordModeToCities.new.exec_migration(conn, direction) }
    [ City, CityFeature ].each(&:reset_column_information)
  end

  def fingerprint
    ignored = "('schema_migrations', 'ar_internal_metadata')"
    {
      columns: conn.select_rows(<<~SQL),
        SELECT table_name, column_name, data_type, is_nullable, column_default FROM information_schema.columns
        WHERE table_schema = 'public' AND table_name NOT IN #{ignored} ORDER BY 1, 2
      SQL
      indexes: conn.select_rows("SELECT tablename, indexname, indexdef FROM pg_indexes WHERE schemaname = 'public' ORDER BY 1, 2"),
      constraints: conn.select_rows(<<~SQL)
        SELECT rel.relname, con.conname, pg_get_constraintdef(con.oid) FROM pg_constraint con
        JOIN pg_class rel ON rel.oid = con.conrelid JOIN pg_namespace ns ON ns.oid = rel.relnamespace
        WHERE ns.nspname = 'public' ORDER BY 1, 2
      SQL
    }
  end

  it "down remove o que up cria; up seguinte restaura idêntico" do
    PlatformRecord.transaction(requires_new: true) do
      before = fingerprint
      migrate(:down)
      expect(conn.table_exists?(:city_features)).to be(false)
      expect(conn.columns(:cities).map(&:name)).not_to include("record_mode", "pec_url")
      migrate(:up)
      expect(fingerprint).to eq(before)
      raise ActiveRecord::Rollback
    end
  ensure
    [ City, CityFeature ].each(&:reset_column_information)
  end
end
```

- [ ] **Step 7: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec spec/models/city_record_settings_spec.rb spec/models/add_record_mode_to_cities_migration_spec.rb spec/adr_pointers_spec.rb spec/requests/operators/cities_spec.rb`
Expected: PASS.

- [ ] **Step 8: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod16 add db/platform_migrate/20261005200001_add_record_mode_to_cities.rb db/platform_schema.rb app/models/city.rb app/models/city_feature.rb app/services/platform/features.rb spec/adr_pointers_spec.rb spec/models/city_record_settings_spec.rb spec/models/add_record_mode_to_cities_migration_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod16 commit -m "feat: add record mode, PEC address and per-city feature switches to the platform

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 2: Migração de cidade — credenciais, CNES da unidade, equipes, CPF do profissional, CNS do cidadão

**Files:**
- Create: `db/city_migrate/20261005200001_add_record_mode_foundation.rb`, `app/models/integration_credential.rb`, `app/models/health_team.rb`, `app/models/health_team_member.rb`
- Modify: `db/city_schema.rb`, `app/models/health_unit.rb`, `app/models/professional.rb`, `app/models/citizen.rb`, `app/services/city_encryption.rb`, `config/initializers/filter_parameter_logging.rb`
- Test: `spec/models/record_mode_city_tables_spec.rb`, `spec/models/add_record_mode_foundation_migration_spec.rb`

**Interfaces:**
- Produces:
  - `IntegrationCredential` (`kind`, `secret` Hash `{ "username", "password" }` cifrado, `set_by_user`, `set_at`, `last_check_at`, `last_check_status`, `last_check_message`); `KINDS == %w[ledi cadsus]`, `STATUSES == %w[ok unauthorized unreachable error]`; `#username`, `#password`; `inspect` sem segredo.
  - `HealthTeam` (`ine`, `kind` ∈ `HealthTeam::KINDS == %w[70 76]`, `name`, `health_unit`, `active`, `has_many :members`); `HealthTeamMember` (`professional`, `health_team`, `cbo_code`, `started_on`, `ended_on`; `scope :active`).
  - `HealthUnit#cnes` (7 dígitos, único, normalizado para dígitos, `nil` permitido).
  - `Professional#cpf` (cifrado determinístico, único, dígito verificador), `Professional#cpf_masked`, `"cpf"` em `Professional::FIELDS`, `Professionals::Create::UNIQUE_REASONS["idx_professionals_cpf"] == :cpf_taken`.
  - `Citizen#cns`, `#cadsus_checked_at`, `#cadsus_pending_cns`, `#cadsus_pending_session_id`, `#cadsus_pending_at`; `Citizen#cns_masked`.

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/models/record_mode_city_tables_spec.rb
require "rails_helper"

# ADR 0028 (spec 2026-10-05 §3.3, §5, §7): o banco da cidade guarda credencial
# cifrada, CNES/INE/CPF/CNS — e garante unicidade e formato onde o modelo não vê.
RSpec.describe "Tabelas da cidade do módulo 16" do
  let(:admin) { staff_with("admin@cidade.gov.br", "municipal_admin") }
  let(:unit) { create_unit }
  def sql(statement) = ApplicationRecord.transaction(requires_new: true) { ApplicationRecord.connection.execute(statement) }

  describe IntegrationCredential do
    let!(:credential) do
      described_class.create!(kind: "ledi", secret: { "username" => "rota", "password" => "segredo-123" },
                              set_by_user: admin, set_at: Time.current)
    end

    it "cifra o segredo e nunca o mostra no inspect" do
      raw = ApplicationRecord.connection.select_value("SELECT secret FROM integration_credentials WHERE id = '#{credential.id}'")
      expect(raw).not_to include("segredo-123")
      expect(credential.reload.password).to eq("segredo-123")
      expect(credential.inspect).not_to include("segredo-123")
    end

    it "uma por kind; kind e status fora da lista são recusados" do
      expect { sql("INSERT INTO integration_credentials (id, kind, secret, set_by_user_id, set_at, created_at, updated_at) " \
                   "VALUES (gen_random_uuid(), 'ledi', 'x', '#{admin.id}', now(), now(), now())") }
        .to raise_error(ActiveRecord::RecordNotUnique)
      expect { sql("UPDATE integration_credentials SET kind = 'rnds'") }.to raise_error(ActiveRecord::StatementInvalid)
      expect { sql("UPDATE integration_credentials SET last_check_status = 'talvez'") }.to raise_error(ActiveRecord::StatementInvalid)
    end

    it "segredo precisa de usuário e senha não vazios" do
      credential.secret = { "username" => "rota", "password" => "" }
      expect(credential).not_to be_valid
    end
  end

  it "CNES da unidade: 7 dígitos, único, normalizado" do
    unit.update!(cnes: "123.456-7")
    expect(unit.reload.cnes).to eq("1234567")
    expect(HealthUnit.new(name: "Outra", kind: "ubs", cnes: "1234567")).not_to be_valid
    expect(HealthUnit.new(name: "Outra", kind: "ubs", cnes: "123")).not_to be_valid
    expect { sql("UPDATE health_units SET cnes = '12'") }.to raise_error(ActiveRecord::StatementInvalid)
  end

  it "equipe: INE único de 10 dígitos e tipo 70 ou 76; um membro ativo por profissional e equipe" do
    team = HealthTeam.create!(ine: "0001234567", kind: "70", name: "ESF 1", health_unit: unit)
    expect(HealthTeam.new(ine: "0001234567", kind: "76", health_unit: unit)).not_to be_valid
    expect(HealthTeam.new(ine: "1", kind: "70", health_unit: unit)).not_to be_valid
    expect { sql("UPDATE health_teams SET kind = '71'") }.to raise_error(ActiveRecord::StatementInvalid)

    doctor = staff_with("medica@cidade.gov.br", "health_professional")
    link_professional!(doctor, unit)
    professional = doctor.reload.professional
    HealthTeamMember.create!(professional: professional, health_team: team, cbo_code: "225125", started_on: Date.current)
    expect {
      HealthTeamMember.new(professional: professional, health_team: team, cbo_code: "225125", started_on: Date.current)
                      .save!(validate: false)
    }.to raise_error(ActiveRecord::RecordNotUnique)
    expect { sql("UPDATE health_team_members SET ended_on = started_on - 1") }.to raise_error(ActiveRecord::StatementInvalid)
  end

  it "CPF do profissional: dígito verificador, cifrado, único, mascarado" do
    doctor = staff_with("medica@cidade.gov.br", "health_professional")
    link_professional!(doctor, unit)
    professional = doctor.reload.professional
    professional.update!(cpf: "529.982.247-25")
    expect(professional.reload.cpf).to eq("52998224725")
    expect(professional.cpf_masked).to eq("***.982.247-**")
    raw = ApplicationRecord.connection.select_value("SELECT cpf FROM professionals WHERE id = '#{professional.id}'")
    expect(raw).not_to include("52998224725")
    professional.cpf = "52998224724"
    expect(professional).not_to be_valid
    expect(Professional::FIELDS).to include("cpf")
    expect(Professional::SELF_EDITABLE).not_to include("cpf")
  end

  it "CNS do cidadão e o pendente do CADSUS cifrados" do
    citizen = Citizen.create!(cpf: "52998224725", phone: "+5541998765432")
    citizen.update!(cns: "700000000000005", cadsus_pending_cns: "700000000000005")
    row = ApplicationRecord.connection.select_one("SELECT cns, cadsus_pending_cns FROM citizens WHERE id = '#{citizen.id}'")
    expect(row.values.join).not_to include("700000000000005")
    expect(citizen.reload.cns_masked).to eq("*** **** **** 0005")
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec spec/models/record_mode_city_tables_spec.rb`
Expected: FAIL (`uninitialized constant IntegrationCredential`).

- [ ] **Step 3: Migração de cidade**

```ruby
# db/city_migrate/20261005200001_add_record_mode_foundation.rb
# ADR 0028 (spec 2026-10-05 §3.3, §5, §7): credenciais de integração (segredo
# cifrado com a chave da cidade, em text — o envelope da cifra não é jsonb),
# CNES da unidade, equipes (INE) e seus membros, CPF do profissional (cifrado
# determinístico, para unicidade e casamento) e o CNS do cidadão com a marca da
# conferência e o pendente da consulta (desvio 6). Só expansão; os CHECKs vão
# na forma que o dump reproduz.
class AddRecordModeFoundation < ActiveRecord::Migration[8.1]
  def change
    create_table :integration_credentials, id: :uuid do |t|
      t.string :kind, null: false
      t.text :secret, null: false
      t.references :set_by_user, type: :uuid, null: false, foreign_key: { to_table: :users }
      t.datetime :set_at, null: false
      t.datetime :last_check_at
      t.string :last_check_status
      t.string :last_check_message, limit: 200
      t.timestamps
      t.index :kind, unique: true
    end
    add_check_constraint :integration_credentials, "kind::text = ANY (ARRAY['ledi', 'cadsus']::text[])",
                         name: "ck_integration_credentials_kind"
    add_check_constraint :integration_credentials,
                         "last_check_status IS NULL OR last_check_status::text = ANY (ARRAY['ok', 'unauthorized', " \
                         "'unreachable', 'error']::text[])",
                         name: "ck_integration_credentials_status"

    add_column :health_units, :cnes, :string, limit: 7
    add_index :health_units, :cnes, unique: true, where: "cnes IS NOT NULL", name: "idx_health_units_cnes"
    add_check_constraint :health_units, "cnes IS NULL OR cnes::text ~ '^[0-9]{7}$'::text", name: "ck_health_units_cnes"

    create_table :health_teams, id: :uuid do |t|
      t.string :ine, null: false, limit: 10
      t.string :kind, null: false
      t.string :name, limit: 120
      t.references :health_unit, type: :uuid, null: false, foreign_key: true
      t.boolean :active, null: false, default: true
      t.timestamps
      t.index :ine, unique: true
    end
    add_check_constraint :health_teams, "ine::text ~ '^[0-9]{10}$'::text", name: "ck_health_teams_ine"
    add_check_constraint :health_teams, "kind::text = ANY (ARRAY['70', '76']::text[])", name: "ck_health_teams_kind"

    create_table :health_team_members, id: :uuid do |t|
      t.references :professional, type: :uuid, null: false, foreign_key: true
      t.references :health_team, type: :uuid, null: false, foreign_key: true
      t.string :cbo_code, null: false
      t.date :started_on, null: false
      t.date :ended_on
      t.timestamps
      t.index %i[professional_id health_team_id], unique: true, where: "ended_on IS NULL",
                                                   name: "idx_health_team_members_one_active"
    end
    add_check_constraint :health_team_members, "cbo_code::text ~ '^[0-9]{6}$'::text", name: "ck_health_team_members_cbo_code"
    add_check_constraint :health_team_members, "ended_on IS NULL OR ended_on >= started_on",
                         name: "ck_health_team_members_order"

    add_column :professionals, :cpf, :string
    add_index :professionals, :cpf, unique: true, where: "cpf IS NOT NULL", name: "idx_professionals_cpf"

    add_column :citizens, :cns, :string
    add_column :citizens, :cadsus_checked_at, :timestamptz
    add_column :citizens, :cadsus_pending_cns, :string
    add_column :citizens, :cadsus_pending_session_id, :uuid
    add_column :citizens, :cadsus_pending_at, :timestamptz
  end
end
```

- [ ] **Step 4: Modelos, cifra e filtro de log**

```ruby
# app/models/integration_credential.rb
# Credencial de integração da cidade (ADR 0028; spec 2026-10-05 §3.3). Só o
# municipal_admin escreve (Integrations::SetCredential). O segredo é um Hash
# { "username", "password" } serializado em JSON e cifrado com a chave da
# cidade — `serialize` vem ANTES de `encrypts` (o tipo cifrado embrulha o
# serializado). Nunca aparece em resposta, log, evento ou argumento de job.
class IntegrationCredential < ApplicationRecord
  KINDS = %w[ledi cadsus].freeze
  STATUSES = %w[ok unauthorized unreachable error].freeze
  FIELD_MAX = 200

  serialize :secret, coder: JSON
  encrypts :secret

  belongs_to :set_by_user, class_name: "User"

  validates :kind, inclusion: { in: KINDS }, uniqueness: true
  validates :set_at, presence: true
  validates :last_check_status, inclusion: { in: STATUSES }, allow_nil: true
  validates :last_check_message, length: { maximum: FIELD_MAX }
  validate :secret_shape

  def username = secret.is_a?(Hash) ? secret["username"] : nil
  def password = secret.is_a?(Hash) ? secret["password"] : nil

  def inspect = "#<IntegrationCredential id=#{id.inspect} kind=#{kind.inspect} last_check_status=#{last_check_status.inspect}>"

  private

  def secret_shape
    ok = secret.is_a?(Hash) && secret.keys.sort == %w[password username] &&
         secret.values.all? { |v| v.is_a?(String) && v.strip.present? && v.length <= FIELD_MAX }
    errors.add(:secret, :invalid) unless ok
  end
end
```

```ruby
# app/models/health_team.rb
# Equipe da APS (ADR 0028; spec 2026-10-05 §5): INE do CNES, tipo 70 (eSF) ou
# 76 (eAP), unidade onde atua. Nasce só pela confirmação de proposta do CNES
# (Cnes::Apply); desativar grava active=false, nada se apaga.
class HealthTeam < ApplicationRecord
  KINDS = %w[70 76].freeze

  belongs_to :health_unit
  has_many :members, class_name: "HealthTeamMember", dependent: :restrict_with_error

  validates :ine, format: { with: /\A\d{10}\z/ }, uniqueness: true
  validates :kind, inclusion: { in: KINDS }
  validates :name, length: { maximum: 120 }
end
```

```ruby
# app/models/health_team_member.rb
# Profissional numa equipe, com o CBO do CNES (ADR 0028; spec 2026-10-05 §5).
# Encerrar grava ended_on; um vínculo ativo por profissional e equipe (índice
# único parcial).
class HealthTeamMember < ApplicationRecord
  belongs_to :professional
  belongs_to :health_team

  scope :active, -> { where(ended_on: nil) }

  validates :cbo_code, format: { with: /\A\d{6}\z/ }
  validates :started_on, presence: true
end
```

Em `app/models/health_unit.rb`, depois de `normalizes :address_zip ...`:

```ruby
  # CNES da unidade (ADR 0028): confirmado pela cidade a partir do retrato do
  # CNES, ou editado pelo admin. Só dígitos; único.
  normalizes :cnes, with: ->(v) { v.to_s.gsub(/\D/, "").presence }, apply_to_nil: true
  validates :cnes, format: { with: /\A\d{7}\z/ }, uniqueness: true, allow_nil: true
```

Em `app/models/professional.rb`:
- `FIELDS` passa a `%w[professional_name council council_state registration_number cns cpf phone contact_email].freeze`;
- depois de `encrypts :cns, ...`: `encrypts :cpf, deterministic: true, key_provider: CityDeterministicKeyProvider.new`;
- depois de `normalizes :registration_number, :cns, ...`: `normalizes :cpf, with: ->(v) { v.to_s.gsub(/\D/, "").presence }, apply_to_nil: true`;
- depois do `validate` do CNS: `validate { errors.add(:cpf, :invalid) unless cpf.nil? || CitizenIdentity::Cpf.normalize(cpf) }`;
- depois de `def cns_masked`: `def cpf_masked = cpf && CitizenIdentity::Cpf.mask(cpf)`.

Em `app/commands/professionals/create.rb`, `UNIQUE_REASONS` ganha `"idx_professionals_cpf" => :cpf_taken`.

Em `app/models/citizen.rb`, depois de `encrypts :phone, ...`:

```ruby
  # ADR 0028 (spec 2026-10-05 §7): do CADSUS só ficam o CNS e a marca da
  # conferência. O pendente guarda o CNS da última consulta até a validação
  # confirmar (contratos §5.4), e é limpo por ela.
  encrypts :cns
  encrypts :cadsus_pending_cns
```

e depois de `def cpf_masked ... end`:

```ruby
  def cns_masked = Professionals::Cns.mask(cns)
```

Em `app/services/city_encryption.rb`, `CITY_KEYED_TARGETS` ganha, antes de `[ Professional, :phone ]`:

```ruby
    [ Professional,   :cpf ],
    [ Citizen,        :cns ],
    [ Citizen,        :cadsus_pending_cns ],
    [ IntegrationCredential, :secret ],
```

Em `config/initializers/filter_parameter_logging.rb`, a primeira lista ganha `:cns` depois de `:cpf`.

- [ ] **Step 5: Migre os bancos de dev e faça o dump à mão**

```bash
docker compose exec -T -w /rails/.claude/mod16 api bin/rails "city:migrate[curitiba]"
docker compose exec -T -w /rails/.claude/mod16 api bin/rails "city:migrate[maringa]"
```
Expected: `[city:migrate] curitiba → 20261005200001` (e maringa).

Em `db/city_schema.rb`:
- `define(version: 2026_10_04_100001)` passa a `define(version: 2026_10_05_200001)`;
- em `create_table "citizens"`, antes de `t.string "cpf", null: false`:

```ruby
    t.timestamptz "cadsus_checked_at"
    t.timestamptz "cadsus_pending_at"
    t.string "cadsus_pending_cns"
    t.uuid "cadsus_pending_session_id"
    t.string "cns"
```

- em `create_table "health_units"`, depois de `t.string "address_zip", limit: 8`: `t.string "cnes", limit: 7`; antes do índice `lower((name)::text)`: `t.index ["cnes"], name: "idx_health_units_cnes", unique: true, where: "(cnes IS NOT NULL)"`; entre os CHECKs `address_zip` e `kind`: `t.check_constraint "cnes IS NULL OR cnes::text ~ '^[0-9]{7}$'::text", name: "ck_health_units_cnes"`;
- em `create_table "professionals"`, depois de `t.string "council_state", null: false`: `t.string "cpf"`; entre os índices `idx_professionals_cns` e `idx_professionals_registration`: `t.index ["cpf"], name: "idx_professionals_cpf", unique: true, where: "(cpf IS NOT NULL)"`;
- antes de `create_table "health_unit_drains"` (ordem alfabética):

```ruby
  create_table "health_team_members", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.string "cbo_code", null: false
    t.datetime "created_at", null: false
    t.date "ended_on"
    t.uuid "health_team_id", null: false
    t.uuid "professional_id", null: false
    t.date "started_on", null: false
    t.datetime "updated_at", null: false
    t.index ["health_team_id"], name: "index_health_team_members_on_health_team_id"
    t.index ["professional_id", "health_team_id"], name: "idx_health_team_members_one_active", unique: true, where: "(ended_on IS NULL)"
    t.index ["professional_id"], name: "index_health_team_members_on_professional_id"
    t.check_constraint "cbo_code::text ~ '^[0-9]{6}$'::text", name: "ck_health_team_members_cbo_code"
    t.check_constraint "ended_on IS NULL OR ended_on >= started_on", name: "ck_health_team_members_order"
  end

  create_table "health_teams", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.boolean "active", default: true, null: false
    t.datetime "created_at", null: false
    t.uuid "health_unit_id", null: false
    t.string "ine", limit: 10, null: false
    t.string "kind", null: false
    t.string "name", limit: 120
    t.datetime "updated_at", null: false
    t.index ["health_unit_id"], name: "index_health_teams_on_health_unit_id"
    t.index ["ine"], name: "index_health_teams_on_ine", unique: true
    t.check_constraint "ine::text ~ '^[0-9]{10}$'::text", name: "ck_health_teams_ine"
    t.check_constraint "kind::text = ANY (ARRAY['70', '76']::text[])", name: "ck_health_teams_kind"
  end
```

- na posição alfabética de `integration_credentials` (depois de `inbound_messages`):

```ruby
  create_table "integration_credentials", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.datetime "created_at", null: false
    t.string "kind", null: false
    t.datetime "last_check_at"
    t.string "last_check_message", limit: 200
    t.string "last_check_status"
    t.text "secret", null: false
    t.datetime "set_at", null: false
    t.uuid "set_by_user_id", null: false
    t.datetime "updated_at", null: false
    t.index ["kind"], name: "index_integration_credentials_on_kind", unique: true
    t.index ["set_by_user_id"], name: "index_integration_credentials_on_set_by_user_id"
    t.check_constraint "kind::text = ANY (ARRAY['ledi', 'cadsus']::text[])", name: "ck_integration_credentials_kind"
    t.check_constraint "last_check_status IS NULL OR last_check_status::text = ANY (ARRAY['ok', 'unauthorized', 'unreachable', 'error']::text[])", name: "ck_integration_credentials_status"
  end
```

- na lista de `add_foreign_key`, em ordem alfabética:

```ruby
  add_foreign_key "health_team_members", "health_teams"
  add_foreign_key "health_team_members", "professionals"
  add_foreign_key "health_teams", "health_units"
  add_foreign_key "integration_credentials", "users", column: "set_by_user_id"
```

O juiz é a paridade (Step 7). Se ela acusar diferença, compare com o banco migrado (`docker compose exec -T db psql -U postgres -d <banco_de_curitiba> -c "\d+ integration_credentials"` …; o nome do banco sai de `bin/rails runner 'puts URI(City.find_by(slug: "curitiba").database_url).path.delete_prefix("/")'` — não cole a URL em lugar nenhum) e ajuste o dump, nunca a migração.

- [ ] **Step 6: Spec de down e up**

```ruby
# spec/models/add_record_mode_foundation_migration_spec.rb
require "rails_helper"
require Rails.root.join("db/city_migrate/20261005200001_add_record_mode_foundation.rb").to_s

# ADR 0028: down desfaz o que up cria; up seguinte restaura idêntico. Savepoint.
RSpec.describe "Migração de cidade 20261005200001 (AddRecordModeFoundation): down e up" do
  let(:models) { [ IntegrationCredential, HealthTeam, HealthTeamMember, HealthUnit, Professional, Citizen ] }

  def conn = ApplicationRecord.connection

  def migrate(direction)
    ActiveRecord::Migration.suppress_messages { AddRecordModeFoundation.new.exec_migration(conn, direction) }
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

  it "down remove tabelas e colunas; up restaura idêntico" do
    ApplicationRecord.transaction(requires_new: true) do
      before = fingerprint
      migrate(:down)
      expect(conn.tables & %w[integration_credentials health_teams health_team_members]).to be_empty
      expect(conn.columns(:citizens).map(&:name)).not_to include("cns", "cadsus_checked_at", "cadsus_pending_cns")
      migrate(:up)
      expect(fingerprint).to eq(before)
      raise ActiveRecord::Rollback
    end
  ensure
    models.each(&:reset_column_information)
  end
end
```

- [ ] **Step 7: Recarregue os bancos de teste e rode**

```bash
docker compose exec -T db psql -U postgres -c "DROP DATABASE IF EXISTS rota_saude_test_city_a" -c "DROP DATABASE IF EXISTS rota_saude_test_city_b"
docker compose exec -T -w /rails/.claude/mod16 api bin/rails city:test_databases
docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec spec/models/record_mode_city_tables_spec.rb spec/models/add_record_mode_foundation_migration_spec.rb spec/services/city_schema_spec.rb spec/architecture/city_encrypted_attributes_guard_spec.rb spec/requests/professionals_spec.rb spec/requests/health_units_spec.rb
```
Expected: PASS.

- [ ] **Step 8: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod16 add db/city_migrate/20261005200001_add_record_mode_foundation.rb db/city_schema.rb app/models/integration_credential.rb app/models/health_team.rb app/models/health_team_member.rb app/models/health_unit.rb app/models/professional.rb app/models/citizen.rb app/commands/professionals/create.rb app/services/city_encryption.rb config/initializers/filter_parameter_logging.rb spec/models/record_mode_city_tables_spec.rb spec/models/add_record_mode_foundation_migration_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod16 commit -m "feat: add integration credentials, health teams and CNES/CPF/CNS columns to the city database

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 3: `Platform::Features` — ligado, utilizável e o que falta

**Files:**
- Modify: `app/services/platform/features.rb`, `spec/events/platform_event_payload_guard_spec.rb`
- Test: `spec/services/platform/features_spec.rb`

**Interfaces:**
- Consumes: `CityFeature`, `City#record_mode`/`#pec_url` (Task 1); `IntegrationCredential`, `CityProfile#ibge_code` (Task 2).
- Produces (todos `module_function` em `Platform::Features`):
  - `UnknownFeature < StandardError`; `find!(key) -> Entry` (levanta `UnknownFeature`).
  - `city_state(city) -> { credentials: Hash{kind => last_check_status|nil}, ibge_code: String|nil } | nil` — `nil` quando a cidade não está `active` ou não responde (`Maintenance::CityConnectionErrors::CLASSES`).
  - `enabled?(city, key) -> Boolean`; `enabled_keys(city) -> Array<String>` (ordenadas, únicas).
  - `missing(city, key, state: :load) -> Array<String>` (códigos do contrato §2; `["city_unreachable"]` com `state` nil).
  - `usable?(city, key, state: :load) -> Boolean`.
  - `summary(city, state: :load) -> Array<Hash>` com `{ key:, description:, enabled:, usable:, missing:, changed_at:, changed_by: }` (`changed_by` = e-mail do mantenedor ou nil), um por entrada do catálogo, na ordem do catálogo.
  - `set!(city:, key:, enabled:, maintainer:) -> CityFeature` — grava e audita `city.feature_changed` só quando o valor muda.
  - `settings(city) -> { record_mode:, pec_url: }` lido da plataforma, sem cache.

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/services/platform/features_spec.rb
require "rails_helper"

# ADR 0028 (spec 2026-10-05 §3.1; contratos §2): ligado vem da plataforma; o que
# falta lê a plataforma (modo, PEC) e o banco da cidade (IBGE, credenciais). Com
# a cidade fora do ar, o liga/desliga continua e o resto degrada.
RSpec.describe Platform::Features do
  # A mesma linha de catálogo dos request specs: CityConnection.with(city) cai na
  # sessão de TEST_CITY_A da transação, onde as linhas abaixo são criadas.
  let!(:city) do
    City.find_by(slug: TEST_CITY_A.slug) ||
      City.create!(slug: TEST_CITY_A.slug, name: TEST_CITY_A.name, status: "active",
                   database_url: TEST_CITY_A.database_url, encryption_key: TEST_CITY_A.encryption_key,
                   schema_version: CitySchema.expected_version.to_s)
  end
  let(:admin) { staff_with("admin@cidade.gov.br", "municipal_admin") }
  let(:maintainer) do
    Maintainer.create!(email_address: "m-#{SecureRandom.hex(3)}@rotasaude.app", password: "s3nha-forte-1",
                       otp_secret: ROTP::Base32.random, otp_enabled_at: Time.current)
  end
  let(:unreachable) do
    create(:city, database_url: "postgres://rota_city:x@127.0.0.1:1/nenhum")
  end

  after { CityCatalog.reset_cache! }

  def credential!(kind, status: nil)
    IntegrationCredential.create!(kind: kind, secret: { "username" => "u", "password" => "p" }, set_by_user: admin,
                                  set_at: Time.current, last_check_status: status)
  end

  it "ledi_export sem nada: falta tudo, na ordem do contrato" do
    expect(described_class.missing(city, "ledi_export"))
      .to eq(%w[record_mode_off pec_url_missing ibge_code_missing credential_missing:ledi])
  end

  it "ledi_export completo não falta nada; credencial recusada volta a faltar" do
    city.update!(record_mode: "integrated", pec_url: "https://pec.cidade.gov.br")
    CityProfile.create!(name: "Cidade", uf: "PR", ibge_code: "4106902")
    credential = credential!("ledi", status: "ok")
    expect(described_class.missing(city, "ledi_export")).to eq([])
    credential.update!(last_check_status: "unauthorized")
    expect(described_class.missing(city, "ledi_export")).to eq(%w[credential_unauthorized:ledi])
  end

  it "credencial cadastrada e nunca testada conta como presente" do
    credential!("cadsus")
    expect(described_class.missing(city, "cadsus_lookup")).to eq([])
  end

  it "utilizável exige ligado E nada faltando" do
    credential!("cadsus", status: "ok")
    expect(described_class.usable?(city, "cadsus_lookup")).to be(false)
    described_class.set!(city: city, key: "cadsus_lookup", enabled: true, maintainer: maintainer)
    expect(described_class.enabled?(city, "cadsus_lookup")).to be(true)
    expect(described_class.usable?(city, "cadsus_lookup")).to be(true)
    expect(described_class.enabled_keys(city)).to eq(%w[cadsus_lookup])
  end

  it "set! audita só quando muda, só com ids" do
    expect {
      described_class.set!(city: city, key: "ledi_export", enabled: true, maintainer: maintainer)
      described_class.set!(city: city, key: "ledi_export", enabled: true, maintainer: maintainer)
    }.to change { PlatformEvent.where(name: "city.feature_changed").count }.by(1)
    expect(PlatformEvent.where(name: "city.feature_changed").last.payload)
      .to eq("city_id" => city.id, "key" => "ledi_export", "enabled" => true, "maintainer_id" => maintainer.id)
    described_class.set!(city: city, key: "ledi_export", enabled: false, maintainer: maintainer)
    expect(described_class.enabled?(city, "ledi_export")).to be(false)
  end

  it "chave fora do catálogo levanta" do
    expect { described_class.set!(city: city, key: "rnds", enabled: true, maintainer: maintainer) }
      .to raise_error(described_class::UnknownFeature)
    expect { described_class.enabled?(city, "rnds") }.to raise_error(described_class::UnknownFeature)
  end

  it "cidade inalcançável: liga, e o resumo degrada sem levantar" do
    described_class.set!(city: unreachable, key: "cadsus_lookup", enabled: true, maintainer: maintainer)
    row = described_class.summary(unreachable).find { |f| f[:key] == "cadsus_lookup" }
    expect(row).to include(enabled: true, usable: false, missing: [ "city_unreachable" ],
                           changed_by: maintainer.email_address)
    expect(row[:changed_at]).to be_present
  end

  it "cidade não ativa não é discada" do
    archived = create(:city, status: "archived")
    expect(CityConnection).not_to receive(:with)
    expect(described_class.city_state(archived)).to be_nil
  end

  it "modo e PEC sem o cache do catálogo" do
    stale = City.find(city.id)
    City.where(id: city.id).update_all(record_mode: "record")
    expect(described_class.settings(stale)).to eq(record_mode: "record", pec_url: nil)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec spec/services/platform/features_spec.rb`
Expected: FAIL (`undefined method 'missing'`).

- [ ] **Step 3: Implemente**

Substitua o corpo de `module_function` em `app/services/platform/features.rb` (mantendo `Entry`, `CATALOG`, `KEYS`):

```ruby
    class UnknownFeature < StandardError; end

    UNREACHABLE = [ "city_unreachable" ].freeze

    module_function

    def find(key) = CATALOG.find { |entry| entry.key == key.to_s }

    def find!(key) = find(key) || raise(UnknownFeature, "interruptor fora do catálogo: #{key}")

    def enabled?(city, key)
      find!(key)
      CityFeature.exists?(city_id: city.id, key: key.to_s, enabled: true)
    end

    # Para a sessão (contratos §1): só a plataforma, sem abrir a cidade.
    def enabled_keys(city)
      CityFeature.where(city_id: city.id, enabled: true, key: KEYS).order(:key).distinct.pluck(:key)
    end

    # record_mode e pec_url relidos por id: Current.city sai do CityCatalog, com
    # cache de 30 s (desvio 4).
    def settings(city)
      record_mode, pec_url = City.where(id: city.id).pick(:record_mode, :pec_url)
      { record_mode: record_mode, pec_url: pec_url }
    end

    # O que mora no banco da cidade, numa conexão só. nil = não deu para ler
    # (cidade não ativa não é discada; inalcançável vira nil, nunca exceção).
    def city_state(city)
      return nil unless city.servable?

      CityConnection.with(city) do
        { credentials: IntegrationCredential.pluck(:kind, :last_check_status).to_h,
          ibge_code: CityProfile.current&.ibge_code }
      end
    rescue *Maintenance::CityConnectionErrors::CLASSES
      nil
    end

    def missing(city, key, state: :load)
      entry = find!(key)
      state = city_state(city) if state == :load
      return UNREACHABLE.dup if state.nil?

      platform = settings(city)
      entry.requires.filter_map { |requirement| missing_for(requirement, platform, state) }
    end

    def usable?(city, key, state: :load) = enabled?(city, key) && missing(city, key, state: state).empty?

    def summary(city, state: :load)
      state = city_state(city) if state == :load
      rows = CityFeature.where(city_id: city.id).index_by(&:key)
      emails = Maintainer.where(id: rows.values.map(&:changed_by_maintainer_id)).pluck(:id, :email_address).to_h

      CATALOG.map do |entry|
        row = rows[entry.key]
        lacking = missing(city, entry.key, state: state)
        enabled = row&.enabled || false
        { key: entry.key, description: entry.description, enabled: enabled, usable: enabled && lacking.empty?,
          missing: lacking, changed_at: row&.changed_at, changed_by: row && emails[row.changed_by_maintainer_id] }
      end
    end

    def set!(city:, key:, enabled:, maintainer:)
      find!(key)
      PlatformRecord.transaction do
        feature = CityFeature.lock.find_or_initialize_by(city_id: city.id, key: key.to_s)
        next feature if feature.persisted? && feature.enabled == enabled

        feature.update!(enabled: enabled, changed_by_maintainer: maintainer, changed_at: Time.current)
        Platform.audit("city.feature_changed", city_id: city.id, key: key.to_s, enabled: enabled,
                                               maintainer_id: maintainer.id)
        feature
      end
    rescue ActiveRecord::RecordNotUnique
      retry
    end

    def missing_for(requirement, platform, state)
      case requirement
      when "record_mode" then "record_mode_off" if platform[:record_mode] == "off"
      when "pec_url" then "pec_url_missing" if platform[:pec_url].blank?
      when "ibge_code" then "ibge_code_missing" if state[:ibge_code].blank?
      when /\Acredential:(\w+)\z/
        kind = Regexp.last_match(1)
        if !state[:credentials].key?(kind) then "credential_missing:#{kind}"
        elsif state[:credentials][kind] == "unauthorized" then "credential_unauthorized:#{kind}"
        end
      else
        raise ArgumentError, "pré-requisito desconhecido no catálogo: #{requirement}"
      end
    end
```

Em `spec/events/platform_event_payload_guard_spec.rb`, `R18_PLATFORM_EVENT_NAMES` ganha, depois de `city.consent_term_published`, a linha:

```ruby
                                city.feature_changed city.record_settings_changed
                                terminology.release_activated cnes.snapshot_imported
```

e, depois de `maintenance.protocol.retired maintenance.protocol.reverted`, `maintenance.city.feature_changed` (usado na Task 4).

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec spec/services/platform/features_spec.rb spec/events/platform_event_payload_guard_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod16 add app/services/platform/features.rb spec/services/platform/features_spec.rb spec/events/platform_event_payload_guard_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod16 commit -m "feat: compute enabled, usable and missing prerequisites for city feature switches

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 4: `setCityFeature` e os interruptores na API de manutenção

**Files:**
- Create: `app/graphql/maintenance/mutations/set_city_feature.rb`, `app/graphql/maintenance/types/city_feature_type.rb`
- Modify: `app/graphql/maintenance/mutations/city_mutation.rb`, `app/graphql/maintenance/types/city_type.rb`, `app/graphql/maintenance/types/mutation_type.rb`, `app/graphql/maintenance/analyzers/human_only.rb`, `app/events/maintenance_audit.rb`, `spec/architecture/maintenance_schema_spec.rb`, `spec/graphql/maintenance/analyzers_spec.rb`
- Test: `spec/requests/maintenance/city_features_spec.rb`

**Interfaces:**
- Consumes: `Platform::Features.summary`, `.set!`, `.find`, `.city_state` (Task 3).
- Produces: GraphQL `CityFeature { key description enabled usable missing changedAt changedBy }`; `City.features`, `City.recordMode`; `setCityFeature(citySlug, key, enabled) { ok errors { path message } feature }`; `CityMutation#on_platform(city_slug:, event:, module_name:, field_paths: {}, payload_from_result: nil, **fields) { |city, actor, correlation_id| Result }`.

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/requests/maintenance/city_features_spec.rb
require "rails_helper"

# ADR 0028 (contratos §3): só o mantenedor liga e desliga, auditado; o
# liga/desliga funciona com a cidade fora do ar; usable/missing degradam.
RSpec.describe "Interruptores na API de manutenção", type: :request do
  let(:frontend) { "https://maintenance.rotasaude.app" }
  let(:password) { "s3nha-forte-1" }
  let!(:maintainer) do
    Maintainer.create!(email_address: "cf-#{SecureRandom.hex(3)}@rotasaude.app", password: password,
                       otp_secret: ROTP::Base32.random, otp_enabled_at: Time.current)
  end
  let(:city) { City.find_by!(slug: TEST_CITY_A.slug) }

  def browser = { "Origin" => frontend, "X-Rota-Maintenance" => "1" }
  def json = JSON.parse(response.body)
  def gql!(query, **variables) = post("/graphql", params: { query: query, variables: variables.to_json }, headers: browser)

  def login!
    post "/session", params: { email_address: maintainer.email_address, password: password }, headers: browser
    post "/session/challenge", params: { session_id: json["session_id"], code: ROTP::TOTP.new(maintainer.otp_secret).now },
                               headers: browser
    expect(response).to have_http_status(:ok)
  end

  def set_feature(slug, key, enabled)
    gql!(<<~GQL, citySlug: slug, key: key, enabled: enabled)
      mutation($citySlug: String!, $key: String!, $enabled: Boolean!) {
        setCityFeature(citySlug: $citySlug, key: $key, enabled: $enabled) {
          ok errors { path message } feature { key enabled usable missing changedAt changedBy }
        }
      }
    GQL
    json.dig("data", "setCityFeature")
  end

  before do
    use_test_city_host!
    host! "maintenance-api.rotasaude.app"
    login!
  end

  it "liga, devolve o interruptor e audita o par e o fato" do
    payload = set_feature(city.slug, "cadsus_lookup", true)

    expect(payload["ok"]).to be(true)
    expect(payload["feature"]).to include("key" => "cadsus_lookup", "enabled" => true, "usable" => false,
                                          "missing" => [ "credential_missing:cadsus" ],
                                          "changedBy" => maintainer.email_address)
    expect(CityFeature.find_by!(city: city, key: "cadsus_lookup").enabled).to be(true)
    expect(PlatformEvent.where(name: "maintenance.city.feature_changed").order(:created_at).pluck(Arel.sql("payload->>'outcome'")))
      .to eq(%w[attempted ok])
    expect(PlatformEvent.find_by!(name: "city.feature_changed").payload)
      .to include("city_id" => city.id, "key" => "cadsus_lookup", "enabled" => true, "maintainer_id" => maintainer.id)
  end

  it "chave fora do catálogo: erro em key, nada gravado nem auditado" do
    payload = set_feature(city.slug, "rnds", true)
    expect(payload).to include("ok" => false, "errors" => [ { "path" => "key", "message" => "unknown_feature" } ])
    expect(CityFeature.count).to eq(0)
    expect(PlatformEvent.where("name LIKE ?", "%feature_changed")).to be_empty
  end

  it "cidade inexistente: erro em citySlug" do
    payload = set_feature("nao-existe", "cadsus_lookup", true)
    expect(payload["ok"]).to be(false)
    expect(payload["errors"].first["path"]).to eq("citySlug")
  end

  it "cidade ativa mas inalcançável: liga mesmo assim e degrada" do
    down = create(:city, database_url: "postgres://rota_city:x@127.0.0.1:1/nenhum")
    payload = set_feature(down.slug, "ledi_export", true)
    expect(payload["ok"]).to be(true)
    expect(payload["feature"]).to include("enabled" => true, "usable" => false, "missing" => [ "city_unreachable" ])
  end

  it "city { features recordMode } lê plataforma e cidade" do
    city.update!(record_mode: "integrated")
    gql!(<<~GQL, slug: city.slug)
      query($slug: String!) { city(slug: $slug) { recordMode features { key description enabled usable missing } } }
    GQL
    data = json.dig("data", "city")
    expect(data["recordMode"]).to eq("integrated")
    expect(data["features"].map { |f| f["key"] }).to eq(%w[ledi_export cadsus_lookup])
    expect(data["features"].first["missing"]).to eq(%w[pec_url_missing ibge_code_missing credential_missing:ledi])
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec spec/requests/maintenance/city_features_spec.rb`
Expected: FAIL (`Field 'setCityFeature' doesn't exist on type 'Mutation'`).

- [ ] **Step 3: Base `on_platform`, regras de campo e auditoria**

Em `app/graphql/maintenance/mutations/city_mutation.rb`, em `AUDITED_FIELD_RULES`, depois de `reason_given:`:

```ruby
        # ADR 0028: chave do interruptor — o valor só entra na auditoria se for
        # do catálogo, e a recusa é a do contrato (§3).
        feature_key: lambda do |value|
          "unknown_feature" unless value.is_a?(String) && Platform::Features.find(value)
        end,
        enabled: lambda do |value|
          "enabled inválido" unless value == true || value == false
        end
```

e, depois de `def outcome_unknown?(error) ...`:

```ruby
      # Escrita de PLATAFORMA sobre uma cidade (ADR 0028: interruptores). Mesma
      # ordem de in_city — escopo, campos auditáveis, tentativa gravada, cidade
      # existe e está ativa — mas SEM abrir o banco da cidade: o contrato (§3)
      # exige que o liga/desliga funcione com a cidade fora do ar. O bloco
      # recebe (city, actor, correlation_id) e devolve um Result.
      def on_platform(city_slug:, event:, module_name:, field_paths: {}, payload_from_result: nil, **fields)
        refuse_out_of_scope!(city_slug)
        refusal = unauditable_input(city_slug, fields, DEFAULT_FIELD_PATHS.merge(field_paths))
        return refusal if refusal

        command_result = nil
        outcome = audited(event: event, module_name: module_name, city_slug: city_slug, **fields) do |correlation_id|
          city = City.find_by(slug: city_slug)
          raise Rejected.new("cidade inexistente", path: "citySlug") if city.nil?
          raise Rejected.new("cidade não está ativa (#{city.status})", path: "citySlug") unless city.servable?

          result = yield(city, MaintainerActor.new(credential.maintainer), correlation_id)
          raise Rejected.new(result.message.presence || result.reason.to_s, path: "citySlug") if result.failure?

          command_result = result
        end
        return outcome if payload_from_result.nil? || !outcome[:ok]

        outcome.merge(payload_from_result.call(command_result))
      end
```

Em `app/events/maintenance_audit.rb`: `NAMES` ganha `maintenance.city.feature_changed` (depois de `maintenance.protocol.reverted`) e o `case` ganha, antes do `else`:

```ruby
    when "maintenance.city.feature_changed" then Platform.audit("maintenance.city.feature_changed", **payload)
```

- [ ] **Step 4: Mutation, tipo e campos de City**

```ruby
# app/graphql/maintenance/mutations/set_city_feature.rb
module Maintenance
  module Mutations
    # ADR 0028 (contratos §3): o maintenance é a única porta que liga e desliga
    # interruptores. Escreve na plataforma (on_platform), sem abrir a cidade.
    class SetCityFeature < CityMutation
      description "Liga ou desliga um interruptor de funcionalidade da cidade."

      argument :key, String, required: true
      argument :enabled, Boolean, required: true

      field :feature, Types::CityFeatureType, null: true

      def resolve(city_slug:, key:, enabled:)
        on_platform(city_slug: city_slug, event: "maintenance.city.feature_changed", module_name: "city",
                    field_paths: { feature_key: "key", enabled: "enabled" }, feature_key: key, enabled: enabled,
                    payload_from_result: ->(result) { { feature: result.payload[:feature] } }) do |city, actor, _id|
          Platform::Features.set!(city: city, key: key, enabled: enabled, maintainer: actor.maintainer)
          Result.ok(feature: Platform::Features.summary(city).find { |row| row[:key] == key })
        end
      end
    end
  end
end
```

```ruby
# app/graphql/maintenance/types/city_feature_type.rb
module Maintenance
  module Types
    # ADR 0028 (contratos §3). enabled/changedAt/changedBy vêm da plataforma;
    # usable/missing leem a cidade e degradam para false/["city_unreachable"].
    class CityFeatureType < BaseObject
      graphql_name "CityFeature"
      description "Interruptor de funcionalidade de uma cidade"

      field :key, String, null: false
      field :description, String, null: false
      field :enabled, Boolean, null: false
      field :usable, Boolean, null: false
      field :missing, [ String ], null: false
      field :changed_at, GraphQL::Types::ISO8601DateTime, null: true
      field :changed_by, String, null: true
    end
  end
end
```

Em `app/graphql/maintenance/types/city_type.rb`: troque o comentário `P7 (fix round 1)` por

```ruby
      # O código IBGE mora em `city.profile.ibgeCode` (banco da cidade), fonte
      # única (ADR 0028; contratos §3): `cities` na plataforma não tem a coluna.
      # Modo de prontuário (só leitura aqui; quem escreve é o console) e
      # interruptores (escritos por setCityFeature).
      field :record_mode, String, null: false
      field :features, [ Types::CityFeatureType ], null: false
```

e acrescente, depois de `def schema_behind ...`:

```ruby
      # Nunca levanta: cidade inalcançável degrada só usable/missing.
      def features = Platform::Features.summary(object)
```

Em `app/graphql/maintenance/types/mutation_type.rb`: `field :set_city_feature, mutation: Mutations::SetCityFeature`.
Em `app/graphql/maintenance/analyzers/human_only.rb`, `RESTRICTED` ganha `setCityFeature` (escrita por token de serviço é decisão própria, como as de protocolo).

- [ ] **Step 5: Guardas do schema e dos analisadores**

Em `spec/architecture/maintenance_schema_spec.rb`:
- `"Mutation"` ganha `setCityFeature`; acrescente `"SetCityFeaturePayload" => %w[ok errors feature]` e `"CityFeature" => %w[key description enabled usable missing changedAt changedBy]`;
- `"City"` ganha `recordMode features`;
- `forbidden_name_exempt_fields` passa a:

```ruby
  # ADR 0028: `key` do interruptor é o identificador do catálogo em código
  # ("ledi_export"), não uma chave criptográfica — o par exato, nunca o nome.
  def forbidden_name_exempt_fields
    %w[CityChannel.phoneNumberId CityChannel.displayPhoneNumber CityFeature.key Mutation.setCityFeature.key]
  end
```

Em `spec/graphql/maintenance/analyzers_spec.rb`:
- `def step_up_exempt = %w[saveProtocolDraft submitProtocolForReview setCityFeature]`;
- no exemplo `"routes every city-scoped root field through the credential's city scope"`, depois da linha do `in_city`, acrescente:

```ruby
    expect(scope_checked_before_audit?(method_body(base, "on_platform"))).to be(true),
                                                                          "on_platform não consulta o escopo antes de auditar"
```

Se alguma outra guarda das specs de manutenção exigir `in_city(` no `resolve` de toda subclasse de `CityMutation`, aceite `on_platform(` como alternativa na mesma regra (com um auto-teste que prove que um resolver sem nenhum dos dois é pego) — nunca afrouxe a guarda para "qualquer coisa".

- [ ] **Step 6: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec spec/requests/maintenance spec/graphql spec/architecture/maintenance_schema_spec.rb spec/events`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod16 add app/graphql/maintenance/mutations/set_city_feature.rb app/graphql/maintenance/types/city_feature_type.rb app/graphql/maintenance/mutations/city_mutation.rb app/graphql/maintenance/types/city_type.rb app/graphql/maintenance/types/mutation_type.rb app/graphql/maintenance/analyzers/human_only.rb app/events/maintenance_audit.rb spec/architecture/maintenance_schema_spec.rb spec/graphql/maintenance/analyzers_spec.rb spec/requests/maintenance/city_features_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod16 commit -m "feat: let maintainers switch city features through the maintenance API

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 5: `features` na sessão da cidade

**Files:**
- Modify: `app/controllers/sessions_controller.rb`
- Test: `spec/requests/session_features_spec.rb`

**Interfaces:**
- Consumes: `Platform::Features.enabled_keys(city)` (Task 3).
- Produces: `GET /session` (usuário e operador por grant) com `features: Array<String>` — chaves ligadas, únicas, ordenadas (contratos §1, `session-v1.1.0`).

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/requests/session_features_spec.rb
require "rails_helper"

# Contratos §1 (session-v1.1.0): a sessão da cidade traz as chaves LIGADAS —
# não necessariamente utilizáveis —, únicas; o console não.
RSpec.describe "features na sessão", type: :request do
  let(:city) { City.find_by!(slug: TEST_CITY_A.slug) }
  let(:maintainer) do
    Maintainer.create!(email_address: "sf-#{SecureRandom.hex(3)}@rotasaude.app", password: "s3nha-forte-1",
                       otp_secret: ROTP::Base32.random, otp_enabled_at: Time.current)
  end
  def json = JSON.parse(response.body)

  it "sem interruptor ligado: lista vazia" do
    sign_in_as(staff_with("admin@cidade.gov.br", "municipal_admin"))
    get "/session"
    expect(json["features"]).to eq([])
  end

  it "ligado aparece mesmo sem credencial; desligado não" do
    Platform::Features.set!(city: city, key: "cadsus_lookup", enabled: true, maintainer: maintainer)
    Platform::Features.set!(city: city, key: "ledi_export", enabled: false, maintainer: maintainer)
    sign_in_as(staff_with("admin@cidade.gov.br", "municipal_admin"))
    get "/session"
    expect(json["features"]).to eq(%w[cadsus_lookup])
    expect(json["features"]).to all(match(/\A[a-z][a-z0-9_]*\z/))
  end

  it "operador vindo do console também recebe" do
    Platform::Features.set!(city: city, key: "ledi_export", enabled: true, maintainer: maintainer)
    operator = Operator.create!(email_address: "op-#{SecureRandom.hex(3)}@rotasaude.app", password: "s3nha-forte-1",
                                otp_secret: ROTP::Base32.random, otp_enabled: true)
    sign_in_operator_grant(operator)
    get "/session"
    expect(json["features"]).to eq(%w[ledi_export])
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec spec/requests/session_features_spec.rb`
Expected: FAIL (`features` nil).

- [ ] **Step 3: Implemente**

Em `app/controllers/sessions_controller.rb`, nos dois hashes (`serialize_operator_grant` e `serialize`), depois de `time_zone: Current.city.time_zone`:

```ruby
      # Interruptores LIGADOS na cidade do host (ADR 0028; contratos §1,
      # session-v1.1.0). O que falta para usar, a tela pergunta à rota da
      # funcionalidade (GET /integrations).
      features: Platform::Features.enabled_keys(Current.city)
```

(com vírgula depois de `time_zone: Current.city.time_zone`).

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec spec/requests/session_features_spec.rb spec/requests/sessions_spec.rb spec/requests/session_grant_spec.rb spec/requests/city_time_zone_exposure_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod16 add app/controllers/sessions_controller.rb spec/requests/session_features_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod16 commit -m "feat: expose enabled city features in the session payload

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 6: Console — `PATCH /cities/:id/record_settings` e os campos novos da cidade

**Files:**
- Create: `app/commands/update_city_record_settings.rb`
- Modify: `app/controllers/operators/cities_controller.rb`, `config/routes.rb`
- Test: `spec/requests/operators/city_record_settings_spec.rb`; ajuste `spec/requests/operators/cities_spec.rb` se algum exemplo comparar o hash inteiro de `GET /cities/:id`

**Interfaces:**
- Consumes: `City::RECORD_MODES`, `City.valid_pec_url?` (Task 1); `Platform::Features.city_state`, `.summary` (Task 3); `CityProfile`.
- Produces: `UpdateCityRecordSettings.call(city:, attrs:) -> Result` (`ok(city:, changed:)`; `fail` com `:invalid_record_mode`, `:invalid_ibge_code`, `:invalid_pec_url`, `:city_unreachable`); `Operators::CitiesController#city_json(city)` — o item de `GET /cities`, de `GET /cities/:id` e de `PATCH` (`{ city: ... }`): `id, slug, name, uf, status, schema_version, time_zone, created_at, record_mode, pec_url, ibge_code, city_reachable, features: [{ key, enabled, usable, missing }]`.

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/requests/operators/city_record_settings_spec.rb
require "rails_helper"

# ADR 0028 (spec 2026-10-05 §3.2; contratos §4.1/§4.2): o operador define modo,
# endereço do PEC e IBGE. O IBGE grava no city_profile do banco da cidade; um
# corpo inválido não grava nada; um evento de auditoria só.
RSpec.describe "Console: modo de prontuário da cidade", type: :request do
  let(:password) { "s3nha-forte-1" }
  let!(:operator) do
    Operator.create!(email_address: "op-#{SecureRandom.hex(3)}@rotasaude.app", password: password,
                     otp_secret: ROTP::Base32.random, otp_enabled: true)
  end
  let!(:city) { City.find_by!(slug: TEST_CITY_A.slug) }

  def json = JSON.parse(response.body)
  def patch_settings(body, id: city.id) = patch("/cities/#{id}/record_settings", params: body, as: :json)
  def audits = PlatformEvent.where(name: "city.record_settings_changed")

  before do
    host! "admin.rotasaude.app"
    post "/session", params: { email_address: operator.email_address, password: password }
    post "/session/challenge", params: { session_id: json["session_id"], code: ROTP::TOTP.new(operator.otp_secret).now }
  end

  it "grava os três, IBGE no city_profile, e audita uma vez com os nomes dos campos" do
    patch_settings(record_mode: "integrated", pec_url: "https://pec.cidade.gov.br", ibge_code: "4106902")

    expect(response).to have_http_status(:ok)
    expect(json["city"]).to include("record_mode" => "integrated", "pec_url" => "https://pec.cidade.gov.br",
                                    "ibge_code" => "4106902", "city_reachable" => true)
    expect(json["city"]["features"].map { |f| f["key"] }).to eq(%w[ledi_export cadsus_lookup])
    expect(json["city"]["features"].first).to include("enabled" => false, "usable" => false,
                                                      "missing" => [ "credential_missing:ledi" ])
    expect(CityProfile.current.ibge_code).to eq("4106902")
    expect(audits.map(&:payload)).to eq([ { "city_id" => city.id, "fields" => %w[ibge_code pec_url record_mode] } ])
    expect(audits.first.payload.to_s).not_to include("pec.cidade")
  end

  it "null e \"\" limpam PEC e IBGE; nada mudou não audita" do
    patch_settings(pec_url: "https://pec.cidade.gov.br", ibge_code: "4106902")
    patch_settings(pec_url: "", ibge_code: nil)
    expect(json["city"]).to include("pec_url" => nil, "ibge_code" => nil)
    expect { patch_settings(pec_url: nil) }.not_to change(audits, :count)
  end

  # Review Focus 1: tabela de corpos inválidos não grava nada.
  [
    [ { record_mode: "parcial" }, "invalid_record_mode" ],
    [ { record_mode: nil }, "invalid_record_mode" ],
    [ { ibge_code: "410690" }, "invalid_ibge_code" ],
    [ { ibge_code: "41.069-02" }, "invalid_ibge_code" ],
    [ { ibge_code: 4_106_902 }, "invalid_ibge_code" ],
    [ { pec_url: "http://pec.cidade.gov.br" }, "invalid_pec_url" ],
    [ { pec_url: "https://admin:senha@pec.cidade.gov.br" }, "invalid_pec_url" ],
    [ { pec_url: "https://pec.cidade.gov.br/?x=1" }, "invalid_pec_url" ],
    [ { pec_url: " " }, "invalid_pec_url" ],
    [ { ibge_code: "4106902", pec_url: "http://pec" }, "invalid_pec_url" ]
  ].each do |body, error|
    it "#{body.inspect} → 422 #{error}, nada gravado" do
      patch_settings(body)
      expect(response).to have_http_status(:unprocessable_entity)
      expect(json).to eq("error" => error)
      expect(city.reload).to have_attributes(record_mode: "off", pec_url: nil)
      expect(CityProfile.current&.ibge_code).to be_nil
      expect(audits).to be_empty
    end
  end

  it "cidade inexistente: 404 not_found" do
    patch_settings({ record_mode: "record" }, id: SecureRandom.uuid)
    expect(response).to have_http_status(:not_found)
    expect(json).to eq("error" => "not_found")
  end

  it "IBGE com a cidade fora do ar: 503 city_unreachable e a plataforma também não muda" do
    down = create(:city, database_url: "postgres://rota_city:x@127.0.0.1:1/nenhum")
    patch_settings({ ibge_code: "4106902", record_mode: "record" }, id: down.id)
    expect(response).to have_http_status(:service_unavailable)
    expect(json).to eq("error" => "city_unreachable")
    expect(down.reload.record_mode).to eq("off")
  end

  it "lista e ficha trazem os mesmos campos; cidade inalcançável responde 200 com city_reachable false" do
    down = create(:city, name: "Fora do Ar", database_url: "postgres://rota_city:x@127.0.0.1:1/nenhum")
    get "/cities"
    row = json["data"].find { |c| c["id"] == down.id }
    expect(row).to include("name" => "Fora do Ar", "ibge_code" => nil, "city_reachable" => false, "record_mode" => "off")
    expect(row["features"].first["missing"]).to eq([ "city_unreachable" ])

    get "/cities/#{down.id}"
    expect(response).to have_http_status(:ok)
    expect(json.keys).to match_array(row.keys)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec spec/requests/operators/city_record_settings_spec.rb`
Expected: FAIL (`No route matches [PATCH]`).

- [ ] **Step 3: Command**

```ruby
# app/commands/update_city_record_settings.rb
# Modo de prontuário, endereço do PEC e código IBGE de uma cidade, pelo operador
# no console (ADR 0028; spec 2026-10-05 §3.2; contratos §4.1). Valida TUDO antes
# de escrever: um corpo com qualquer campo inválido não grava nada. O IBGE mora
# no city_profile do banco da cidade (fonte única); modo e PEC, na plataforma.
# A cidade é escrita primeiro: se ela não responde, a plataforma não muda.
# null e "" limpam PEC e IBGE; record_mode nunca é nulo.
# Reasons: :invalid_record_mode, :invalid_ibge_code, :invalid_pec_url, :city_unreachable.
class UpdateCityRecordSettings
  FIELDS = %w[record_mode ibge_code pec_url].freeze
  IBGE_CODE = /\A\d{7}\z/

  def self.call(city:, attrs:)
    attrs = attrs.to_h.stringify_keys.slice(*FIELDS).transform_values { |v| v == "" ? nil : v }
    error = invalid(attrs)
    return Result.fail(error) if error

    changed = []
    if attrs.key?("ibge_code")
      return Result.fail(:city_unreachable) unless city.servable?

      changed << "ibge_code" if write_ibge_code(city, attrs["ibge_code"])
    end

    city.assign_attributes(attrs.slice("record_mode", "pec_url"))
    changed.concat(city.changed & %w[record_mode pec_url])
    city.save!
    changed.sort!
    Platform.audit("city.record_settings_changed", city_id: city.id, fields: changed) if changed.any?
    Result.ok(city: city, changed: changed)
  rescue *Maintenance::CityConnectionErrors::CLASSES
    Result.fail(:city_unreachable)
  end

  def self.invalid(attrs)
    return :invalid_record_mode if attrs.key?("record_mode") && !City::RECORD_MODES.include?(attrs["record_mode"])
    if attrs.key?("ibge_code") && !(attrs["ibge_code"].nil? || (attrs["ibge_code"].is_a?(String) && attrs["ibge_code"].match?(IBGE_CODE)))
      return :invalid_ibge_code
    end

    :invalid_pec_url if attrs.key?("pec_url") && !(attrs["pec_url"].nil? || City.valid_pec_url?(attrs["pec_url"]))
  end

  # true se mudou. A linha única do city_profile nasce no provisionamento; se
  # faltar (cidade antiga), nasce aqui com o nome e a UF do catálogo.
  def self.write_ibge_code(city, value)
    CityConnection.with(city) do
      profile = CityProfile.current || CityProfile.new(name: city.name, uf: city.uf&.upcase)
      profile.ibge_code = value
      next false unless profile.new_record? || profile.changed?

      profile.save!
      true
    end
  end

  private_class_method :invalid, :write_ibge_code
end
```

- [ ] **Step 4: Controller e rota**

Em `config/routes.rb`, no bloco `constraints(PlatformConsoleHost) ... scope module: :operators`, depois de `resources :cities, only: %i[index create show]`:

```ruby
      # Modo de prontuário, PEC e IBGE (ADR 0028; contratos §4.1).
      patch "/cities/:id/record_settings", to: "cities#record_settings"
```

Em `app/controllers/operators/cities_controller.rb`:
- no cabeçalho, acrescente `#   PATCH /cities/:id/record_settings { record_mode?, ibge_code?, pec_url? } → 200 { city }` e troque o comentário de `index` por: `# GET /cities — catálogo para o console (Plano 6). ADR 0028: cada cidade ativa abre a conexão dela uma vez (IBGE e credenciais, para ibge_code e features); cidade que não responde sai com city_reachable false, nunca derruba a lista.`;
- `index` passa a `render json: { data: City.order(created_at: :desc).map { |city| city_json(city) } }`;
- `show` passa a:

```ruby
    def show
      city = City.find_by(id: params[:id].to_s)
      return head(:not_found) unless city

      render json: city_json(city)
    end

    def record_settings
      city = City.find_by(id: params[:id].to_s)
      return render(json: { error: "not_found" }, status: :not_found) unless city

      result = UpdateCityRecordSettings.call(city: city, attrs: request.request_parameters)
      if result.failure?
        status = result.reason == :city_unreachable ? :service_unavailable : :unprocessable_entity
        return render(json: { error: result.reason.to_s }, status: status)
      end

      render json: { city: city_json(city.reload) }
    end

    private

    # Mesmo formato na lista, na ficha e no PATCH (contratos §4.2).
    def city_json(city)
      state = Platform::Features.city_state(city)
      {
        id: city.id, slug: city.slug, name: city.name, uf: city.uf, status: city.status,
        schema_version: city.schema_version, time_zone: city.time_zone, created_at: city.created_at.iso8601,
        record_mode: city.record_mode, pec_url: city.pec_url, ibge_code: state&.dig(:ibge_code),
        city_reachable: !state.nil?,
        features: Platform::Features.summary(city, state: state).map { |f| f.slice(:key, :enabled, :usable, :missing) }
      }
    end
```

- [ ] **Step 5: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec spec/requests/operators spec/requests/city_time_zone_exposure_spec.rb`
Expected: PASS (se `cities_spec.rb` comparava o hash inteiro da ficha, inclua as chaves novas no esperado).

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod16 add app/commands/update_city_record_settings.rb app/controllers/operators/cities_controller.rb config/routes.rb spec/requests/operators/city_record_settings_spec.rb spec/requests/operators/cities_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod16 commit -m "feat: let operators set record mode, PEC address and IBGE code per city

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Fatia 2 — Credenciais e clientes externos (F-16.3)

### Task 7: `Ledi::PecClient#login`

**Files:**
- Create: `app/services/ledi/pec_client.rb`
- Test: `spec/services/ledi/pec_client_spec.rb`

**Interfaces:**
- Produces: `Ledi::PecClient.new(base_url:, username:, password:, timeout: 10)`; `#login -> Ledi::PecClient::Session` (`Data` com `cookie`, ex. `"JSESSIONID=abc"`); erros `Ledi::PecClient::Error` e as subclasses `Unauthorized`, `Unreachable`, `Failed` (`#status`). `LOGIN_PATH = "/api/recebimento/login"`. O plano do exportador acrescenta o envio da ficha a esta classe e reaproveita `#login`, os erros e `#post` (privado).

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/services/ledi/pec_client_spec.rb
require "rails_helper"
require "webmock/rspec"

# ADR 0028 (spec 2026-10-05 §1, §3.3): login na API de recebimento do PEC
# (usuário/senha → cookie JSESSIONID). Erros nunca carregam senha nem corpo.
RSpec.describe Ledi::PecClient do
  let(:url) { "https://pec.cidade.gov.br/api/recebimento/login" }
  let(:client) { described_class.new(base_url: "https://pec.cidade.gov.br/", username: "rota", password: "senha-secreta") }

  it "200 com JSESSIONID: devolve a sessão; manda usuario/senha em JSON" do
    stub_request(:post, url).with(body: { usuario: "rota", senha: "senha-secreta" }.to_json,
                                  headers: { "Content-Type" => "application/json" })
                            .to_return(status: 200, headers: { "Set-Cookie" => "JSESSIONID=abc123; Path=/; Secure; HttpOnly" })
    expect(client.login.cookie).to eq("JSESSIONID=abc123")
  end

  it "401/403: Unauthorized" do
    stub_request(:post, url).to_return(status: 401, body: "usuário senha-secreta inválido")
    expect { client.login }.to raise_error(described_class::Unauthorized) { |e| expect(e.message).not_to include("senha-secreta") }
  end

  it "timeout e conexão recusada: Unreachable" do
    stub_request(:post, url).to_timeout
    expect { client.login }.to raise_error(described_class::Unreachable)
    stub_request(:post, url).to_raise(Errno::ECONNREFUSED)
    expect { client.login }.to raise_error(described_class::Unreachable)
  end

  # Review Focus 2: proxy que responde 200 em HTML, sem cookie.
  it "200 sem cookie e 500: Failed com o status, sem o corpo" do
    stub_request(:post, url).to_return(status: 200, body: "<html>login</html>")
    expect { client.login }.to raise_error(described_class::Failed) { |e| expect(e.message).not_to include("html") }
    stub_request(:post, url).to_return(status: 500, body: "stack trace com senha-secreta")
    expect { client.login }.to raise_error(described_class::Failed) { |e|
      expect(e.status).to eq(500)
      expect(e.message).not_to include("senha-secreta")
    }
  end

  it "inspect nunca mostra a senha" do
    expect(client.inspect).not_to include("senha-secreta")
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec spec/services/ledi/pec_client_spec.rb`
Expected: FAIL (`uninitialized constant Ledi`).

- [ ] **Step 3: Implemente**

```ruby
# app/services/ledi/pec_client.rb
require "net/http"

# Cliente da API de recebimento do PEC da cidade (ADR 0028; spec 2026-10-05 §1;
# pesquisa frente 1): POST /api/recebimento/login (usuario/senha em JSON →
# cookie JSESSIONID). Único lugar que conhece o formato do login — a prova
# técnica do exportador confirma, e o envio da ficha entra aqui no plano dele.
# Nenhuma mensagem de erro carrega senha, cookie ou corpo da resposta.
module Ledi
  class PecClient
    class Error < StandardError; end
    class Unauthorized < Error; end
    class Unreachable < Error; end

    class Failed < Error
      attr_reader :status

      def initialize(status)
        @status = status
        super("o PEC respondeu #{status}")
      end
    end

    Session = Data.define(:cookie)

    LOGIN_PATH = "/api/recebimento/login".freeze
    TIMEOUT = 10
    NETWORK_ERRORS = [ SocketError, IOError, EOFError, Errno::ECONNREFUSED, Errno::ECONNRESET, Errno::EHOSTUNREACH,
                       Errno::ETIMEDOUT, Net::OpenTimeout, Net::ReadTimeout, OpenSSL::SSL::SSLError ].freeze

    def initialize(base_url:, username:, password:, timeout: TIMEOUT)
      @base_url = base_url.to_s.chomp("/")
      @username = username
      @password = password
      @timeout = timeout
    end

    def login
      response = post(LOGIN_PATH, { usuario: @username, senha: @password }.to_json, "application/json")
      case response.code.to_i
      when 200..299
        cookie = session_cookie(response)
        raise Failed.new(response.code.to_i) unless cookie

        Session.new(cookie: cookie)
      when 401, 403 then raise Unauthorized, "o PEC recusou usuário ou senha"
      else raise Failed.new(response.code.to_i)
      end
    end

    def inspect = "#<Ledi::PecClient #{@base_url}>"

    private

    def post(path, body, content_type, headers = {})
      uri = URI.parse("#{@base_url}#{path}")
      request = Net::HTTP::Post.new(uri)
      request["Content-Type"] = content_type
      headers.each { |name, value| request[name] = value }
      request.body = body
      Net::HTTP.start(uri.host, uri.port, use_ssl: uri.scheme == "https",
                                          open_timeout: @timeout, read_timeout: @timeout, write_timeout: @timeout) do |http|
        http.request(request)
      end
    rescue *NETWORK_ERRORS => e
      raise Unreachable, "PEC inalcançável (#{e.class.name})"
    end

    def session_cookie(response)
      Array(response.get_fields("set-cookie")).map { |c| c.split(";").first.to_s.strip }
                                              .find { |c| c.start_with?("JSESSIONID=") && c.length > 11 }
    end
  end
end
```

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec spec/services/ledi/pec_client_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod16 add app/services/ledi/pec_client.rb spec/services/ledi/pec_client_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod16 commit -m "feat: add PEC reception API login client

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 8: `Cadsus::Client` — backends `simulated` e `soap_pdq`

**Files:**
- Create: `app/services/cadsus.rb`, `app/services/cadsus/client.rb`, `app/services/cadsus/simulated.rb`, `app/services/cadsus/soap_pdq.rb`, `spec/fixtures/cadsus/pdq_found.xml`, `spec/fixtures/cadsus/pdq_not_found.xml`
- Modify: `config/environments/development.rb`, `config/environments/test.rb`
- Test: `spec/services/cadsus/client_spec.rb`

**Interfaces:**
- Consumes: `IntegrationCredential` (Task 2).
- Produces: `Cadsus::Error`, `Cadsus::Unauthorized`, `Cadsus::Unavailable` (`< Cadsus::Error`); `Cadsus::Record = Data.define(:cns, :birth_date, :sex)` (`sex` ∈ `"female"`, `"male"`, `nil`); `Cadsus::Client.for(city) -> backend` (levanta `Unavailable` sem credencial `cadsus`); backend responde `#lookup(cpf) -> Record|nil` e `#health_check -> :ok` (ou levanta). `Cadsus::Simulated::NOT_FOUND_CPF == "11144477735"`, `UNAVAILABLE_CPF == "39053344705"`, `REFUSED_USERNAME == "recusado"`. `Cadsus::SoapPdq.new(url:, username:, password:, timeout: 10)`, `SoapPdq::HOMOLOGATION_URL`, `SoapPdq::PRODUCTION_URL`.

- [ ] **Step 1: Fixtures da resposta PDQ**

```xml
<!-- spec/fixtures/cadsus/pdq_found.xml — recorte de PRPA_IN201306UV02 (spec PDQ v5 do DATASUS); valores fictícios -->
<soap:Envelope xmlns:soap="http://www.w3.org/2003/05/soap-envelope">
  <soap:Body>
    <PRPA_IN201306UV02 xmlns="urn:hl7-org:v3" ITSVersion="XML_1.0">
      <controlActProcess classCode="CACT" moodCode="EVN">
        <subject typeCode="SUBJ">
          <registrationEvent classCode="REG" moodCode="EVN">
            <subject1 typeCode="SBJ">
              <patient classCode="PAT">
                <id root="2.16.840.1.113883.13.236" extension="700000000000005"/>
                <patientPerson classCode="PSN" determinerCode="INSTANCE">
                  <name use="L"><given>NOME QUE NUNCA SAI</given></name>
                  <administrativeGenderCode code="F" codeSystem="2.16.840.1.113883.5.1"/>
                  <birthTime value="19800517000000"/>
                </patientPerson>
              </patient>
            </subject1>
          </registrationEvent>
        </subject>
      </controlActProcess>
    </PRPA_IN201306UV02>
  </soap:Body>
</soap:Envelope>
```

```xml
<!-- spec/fixtures/cadsus/pdq_not_found.xml -->
<soap:Envelope xmlns:soap="http://www.w3.org/2003/05/soap-envelope">
  <soap:Body>
    <PRPA_IN201306UV02 xmlns="urn:hl7-org:v3" ITSVersion="XML_1.0">
      <controlActProcess classCode="CACT" moodCode="EVN">
        <queryAck><queryResponseCode code="NF"/></queryAck>
      </controlActProcess>
    </PRPA_IN201306UV02>
  </soap:Body>
</soap:Envelope>
```

- [ ] **Step 2: Escreva a spec que falha**

```ruby
# spec/services/cadsus/client_spec.rb
require "rails_helper"
require "webmock/rspec"

# ADR 0028 (spec 2026-10-05 §7): CADSUS por PDQv3 (SOAP, WS-Security) ou
# simulado em dev/test. Do retorno só saem CNS, nascimento e sexo — em memória.
RSpec.describe Cadsus::Client do
  let(:admin) { staff_with("admin@cidade.gov.br", "municipal_admin") }
  let(:city) do
    City.find_by(slug: TEST_CITY_A.slug) ||
      City.create!(slug: TEST_CITY_A.slug, name: TEST_CITY_A.name, status: "active",
                   database_url: TEST_CITY_A.database_url, encryption_key: TEST_CITY_A.encryption_key,
                   schema_version: CitySchema.expected_version.to_s)
  end

  def credential!(username = "rota")
    IntegrationCredential.create!(kind: "cadsus", secret: { "username" => username, "password" => "senha-pdq" },
                                  set_by_user: admin, set_at: Time.current)
  end

  it "sem credencial: Unavailable" do
    expect { described_class.for(city) }.to raise_error(Cadsus::Unavailable)
  end

  describe Cadsus::Simulated do
    it "casos fixos: achado determinístico, não achado, fora do ar, credencial recusada" do
      credential!
      client = described_class.for(city)
      record = client.lookup("52998224725")
      expect(record.cns).to satisfy { |cns| Professionals::Cns.valid?(cns) }
      expect(client.lookup("52998224725")).to eq(record)
      expect(%w[female male]).to include(record.sex)
      expect(client.lookup(Cadsus::Simulated::NOT_FOUND_CPF)).to be_nil
      expect { client.lookup(Cadsus::Simulated::UNAVAILABLE_CPF) }.to raise_error(Cadsus::Unavailable)
      expect(client.health_check).to eq(:ok)
      expect { Cadsus::Simulated.new(username: "recusado").health_check }.to raise_error(Cadsus::Unauthorized)
    end
  end

  describe Cadsus::SoapPdq do
    let(:url) { Cadsus::SoapPdq::HOMOLOGATION_URL }
    let(:client) { described_class.new(url: url, username: "rota", password: "senha-pdq") }
    def fixture(name) = File.read(Rails.root.join("spec/fixtures/cadsus/#{name}"))

    it "consulta por CPF com WS-Security e lê CNS, nascimento e sexo" do
      stub = stub_request(:post, url).with { |req|
        req.body.include?('root="2.16.840.1.113883.13.237" extension="52998224725"') &&
          req.body.include?("<wsse:Username>rota</wsse:Username>")
      }.to_return(status: 200, body: fixture("pdq_found.xml"))

      record = client.lookup("52998224725")

      expect(stub).to have_been_requested
      expect(record).to eq(Cadsus::Record.new(cns: "700000000000005", birth_date: Date.new(1980, 5, 17), sex: "female"))
    end

    it "sem paciente: nil" do
      stub_request(:post, url).to_return(status: 200, body: fixture("pdq_not_found.xml"))
      expect(client.lookup("52998224725")).to be_nil
    end

    it "401/403 ou Fault de segurança: Unauthorized; 5xx e timeout: Unavailable; nenhuma mensagem traz senha" do
      stub_request(:post, url).to_return(status: 401, body: "senha-pdq")
      expect { client.lookup("52998224725") }.to raise_error(Cadsus::Unauthorized) { |e| expect(e.message).not_to include("senha-pdq") }
      fault = '<soap:Envelope xmlns:soap="http://www.w3.org/2003/05/soap-envelope"><soap:Body><soap:Fault>' \
              "<soap:Reason><soap:Text>FailedAuthentication: The security token could not be authenticated</soap:Text>" \
              "</soap:Reason></soap:Fault></soap:Body></soap:Envelope>"
      stub_request(:post, url).to_return(status: 500, body: fault)
      expect { client.lookup("52998224725") }.to raise_error(Cadsus::Unauthorized)
      stub_request(:post, url).to_return(status: 503, body: "fora")
      expect { client.lookup("52998224725") }.to raise_error(Cadsus::Unavailable)
      stub_request(:post, url).to_timeout
      expect { client.lookup("52998224725") }.to raise_error(Cadsus::Unavailable)
      expect(client.inspect).not_to include("senha-pdq")
    end
  end
end
```

- [ ] **Step 3: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec spec/services/cadsus/client_spec.rb`
Expected: FAIL (`uninitialized constant Cadsus`).

- [ ] **Step 4: Implemente**

```ruby
# app/services/cadsus.rb
# CADSUS (ADR 0028; spec 2026-10-05 §7; pesquisa frente 2). Do retorno só
# interessam CNS, nascimento e sexo, e só o CNS chega a ser gravado (depois da
# confirmação do atendente). Nome, mãe e endereço nunca saem do cliente.
module Cadsus
  class Error < StandardError; end
  class Unauthorized < Error; end
  class Unavailable < Error; end

  Record = Data.define(:cns, :birth_date, :sex)
end
```

```ruby
# app/services/cadsus/client.rb
# Escolhe o backend (ADR 0028): `config.x.cadsus_backend = :simulated` em
# development e test; ausente (staging, production) → PDQv3 SOAP com a
# credencial `cadsus` da cidade. Precisa da credencial nos dois casos: o
# simulado também espelha "sem credencial = indisponível".
module Cadsus
  module Client
    module_function

    def for(city)
      credential = CityConnection.with(city) { IntegrationCredential.find_by(kind: "cadsus") }
      raise Unavailable, "credencial do CADSUS não cadastrada" unless credential

      if Rails.configuration.x.cadsus_backend == :simulated
        Simulated.new(username: credential.username)
      else
        SoapPdq.new(url: Rails.configuration.x.cadsus_pdq_url || SoapPdq.default_url,
                    username: credential.username, password: credential.password)
      end
    end
  end
end
```

```ruby
# app/services/cadsus/simulated.rb
require "digest"

# CADSUS de mentira para development e test (ADR 0028): casos fixos, resposta
# determinística por CPF, sem rede. CPFs com dígito verificador válido.
module Cadsus
  class Simulated
    NOT_FOUND_CPF = "11144477735".freeze
    UNAVAILABLE_CPF = "39053344705".freeze
    REFUSED_USERNAME = "recusado".freeze

    def initialize(username:)
      @username = username
    end

    def health_check
      raise Unauthorized, "o CADSUS recusou a credencial" if @username == REFUSED_USERNAME

      :ok
    end

    def lookup(cpf)
      health_check
      raise Unavailable, "CADSUS simulado fora do ar" if cpf == UNAVAILABLE_CPF
      return nil if cpf == NOT_FOUND_CPF

      seed = Digest::SHA256.hexdigest("cadsus:#{cpf}").to_i(16)
      Record.new(cns: Professionals::Cns.generate("cadsus:#{cpf}"), birth_date: Date.new(1950, 1, 1) + (seed % 25_000),
                 sex: seed.even? ? "female" : "male")
    end
  end
end
```

```ruby
# app/services/cadsus/soap_pdq.rb
require "net/http"
require "erb"

# PDQv3 do CADSUS (spec PDQ v5 do DATASUS; pesquisa frente 2): SOAP 1.2,
# usuário e senha no cabeçalho WS-Security, consulta por CPF
# (2.16.840.1.113883.13.237); o CNS volta como id 2.16.840.1.113883.13.236. A
# credencial vem da cidade; o acesso é autorizado pelo DATASUS. O formato é
# conferido na homologação antes de ligar o interruptor em qualquer cidade.
module Cadsus
  class SoapPdq
    HOMOLOGATION_URL = "https://servicoshm.saude.gov.br/cadsus/PDQSupplier".freeze
    PRODUCTION_URL = "https://servicos.saude.gov.br/cadsus/PDQSupplier".freeze
    CPF_ROOT = "2.16.840.1.113883.13.237".freeze
    CNS_ROOT = "2.16.840.1.113883.13.236".freeze
    # CPF válido de teste, usado só para o teste de conexão (achar ou não achar
    # prova que a credencial foi aceita).
    HEALTH_CHECK_CPF = "11144477735".freeze
    TIMEOUT = 10
    NETWORK_ERRORS = Ledi::PecClient::NETWORK_ERRORS
    SEX = { "F" => "female", "M" => "male" }.freeze

    def self.default_url = Rails.env.production? ? PRODUCTION_URL : HOMOLOGATION_URL

    def initialize(url:, username:, password:, timeout: TIMEOUT)
      @url = url
      @username = username
      @password = password
      @timeout = timeout
    end

    def health_check
      lookup(HEALTH_CHECK_CPF)
      :ok
    end

    def lookup(cpf)
      response = post(envelope(cpf))
      doc = Nokogiri::XML(response.body.to_s)
      fault = doc.at_xpath("//*[local-name()='Fault']")
      code = response.code.to_i
      raise Unauthorized, "o CADSUS recusou a credencial" if [ 401, 403 ].include?(code) || security_fault?(fault)
      raise Unavailable, "o CADSUS respondeu #{code}" if fault || code >= 300

      patient = doc.at_xpath("//*[local-name()='patient']")
      return nil unless patient

      cns = patient.at_xpath(".//*[local-name()='id'][@root='#{CNS_ROOT}']")&.[]("extension")
      birth = patient.at_xpath(".//*[local-name()='birthTime']")&.[]("value").to_s[0, 8]
      sex = patient.at_xpath(".//*[local-name()='administrativeGenderCode']")&.[]("code")
      Record.new(cns: cns.presence, birth_date: birth.match?(/\A\d{8}\z/) ? Date.strptime(birth, "%Y%m%d") : nil,
                 sex: SEX[sex])
    rescue Date::Error
      raise Unavailable, "resposta do CADSUS com data inválida"
    end

    def inspect = "#<Cadsus::SoapPdq #{@url}>"

    private

    def security_fault?(fault)
      fault && fault.text.match?(/auth|secur|credencia|senha|password/i)
    end

    def post(body)
      uri = URI.parse(@url)
      request = Net::HTTP::Post.new(uri)
      request["Content-Type"] = "application/soap+xml; charset=utf-8"
      request.body = body
      Net::HTTP.start(uri.host, uri.port, use_ssl: uri.scheme == "https",
                                          open_timeout: @timeout, read_timeout: @timeout, write_timeout: @timeout) do |http|
        http.request(request)
      end
    rescue *NETWORK_ERRORS => e
      raise Unavailable, "CADSUS inalcançável (#{e.class.name})"
    end

    def envelope(cpf)
      h = ->(value) { ERB::Util.html_escape(value.to_s) }
      now = Time.current.utc.strftime("%Y%m%d%H%M%S")
      <<~XML
        <soap:Envelope xmlns:soap="http://www.w3.org/2003/05/soap-envelope" xmlns:urn="urn:hl7-org:v3">
          <soap:Header>
            <wsse:Security xmlns:wsse="http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-wssecurity-secext-1.0.xsd">
              <wsse:UsernameToken>
                <wsse:Username>#{h.call(@username)}</wsse:Username>
                <wsse:Password Type="http://docs.oasis-open.org/wss/2004/01/oasis-200401-wss-username-token-profile-1.0#PasswordText">#{h.call(@password)}</wsse:Password>
              </wsse:UsernameToken>
            </wsse:Security>
          </soap:Header>
          <soap:Body>
            <urn:PRPA_IN201305UV02 ITSVersion="XML_1.0">
              <urn:id root="2.16.840.1.113883.4.714" extension="#{SecureRandom.uuid}"/>
              <urn:creationTime value="#{now}"/>
              <urn:interactionId root="2.16.840.1.113883.1.6" extension="PRPA_IN201305UV02"/>
              <urn:processingCode code="P"/>
              <urn:processingModeCode code="T"/>
              <urn:acceptAckCode code="AL"/>
              <urn:receiver typeCode="RCV"><urn:device classCode="DEV" determinerCode="INSTANCE"><urn:id root="2.16.840.1.113883.3.72.6.5.100.85"/></urn:device></urn:receiver>
              <urn:sender typeCode="SND"><urn:device classCode="DEV" determinerCode="INSTANCE"><urn:id root="2.16.840.1.113883.3.72.6.2"/><urn:name>ROTASAUDE</urn:name></urn:device></urn:sender>
              <urn:controlActProcess classCode="CACT" moodCode="EVN">
                <urn:code code="PRPA_TE201305UV02" codeSystem="2.16.840.1.113883.1.6"/>
                <urn:queryByParameter>
                  <urn:queryId root="1.2.840.114350.1.13.28.1.18.5.999" extension="#{SecureRandom.uuid}"/>
                  <urn:statusCode code="new"/>
                  <urn:responseModalityCode code="R"/>
                  <urn:responsePriorityCode code="I"/>
                  <urn:parameterList>
                    <urn:livingSubjectId>
                      <urn:value root="#{CPF_ROOT}" extension="#{h.call(cpf)}"/>
                      <urn:semanticsText>LivingSubject.id</urn:semanticsText>
                    </urn:livingSubjectId>
                  </urn:parameterList>
                </urn:queryByParameter>
              </urn:controlActProcess>
            </urn:PRPA_IN201305UV02>
          </soap:Body>
        </soap:Envelope>
      XML
    end
  end
end
```

Em `config/environments/development.rb` e `config/environments/test.rb`, junto de `config.x.sms_gateway`:

```ruby
  # CADSUS (ADR 0028): simulado, sem rede. Ausente (staging, production) → PDQv3.
  config.x.cadsus_backend = :simulated
```

- [ ] **Step 5: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec spec/services/cadsus/client_spec.rb`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod16 add app/services/cadsus.rb app/services/cadsus/client.rb app/services/cadsus/simulated.rb app/services/cadsus/soap_pdq.rb config/environments/development.rb config/environments/test.rb spec/fixtures/cadsus/pdq_found.xml spec/fixtures/cadsus/pdq_not_found.xml spec/services/cadsus/client_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod16 commit -m "feat: add CADSUS client with simulated and PDQv3 SOAP backends

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 9: Integrações — cadastrar, trocar e testar credenciais

**Files:**
- Create: `app/commands/integrations/set_credential.rb`, `app/commands/integrations/check_connection.rb`, `app/controllers/integrations_controller.rb`, `app/policies/integration_policy.rb`
- Modify: `config/routes.rb`, `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`
- Test: `spec/requests/integrations_spec.rb`

**Interfaces:**
- Consumes: `IntegrationCredential` (Task 2), `Platform::Features.settings`, `.city_state`, `.summary` (Task 3), `Ledi::PecClient` (Task 7), `Cadsus::Client` (Task 8).
- Produces: `IntegrationPolicy#manage?` (municipal_admin; reaproveitada pelo CNES na Task 18); `Integrations::SetCredential.call(kind:, username:, password:, by:) -> Result` (`:unknown_kind`, `:invalid_credential`); `Integrations::CheckConnection.call(kind:, city:) -> Result` (`:unknown_kind`, `:credential_missing`); rotas `GET /integrations`, `PUT /integrations/credentials/:kind`, `POST /integrations/credentials/:kind/check` (contratos §5.1).

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/requests/integrations_spec.rb
require "rails_helper"
require "webmock/rspec"

# ADR 0028 (spec 2026-10-05 §3.3; contratos §5.1): o municipal_admin cadastra,
# troca (step-up) e testa credenciais. Nenhuma resposta, evento ou log traz o
# segredo.
RSpec.describe "Integrações da cidade", type: :request do
  let(:city) { City.find_by!(slug: TEST_CITY_A.slug) }
  let(:admin) do
    staff_with("admin@cidade.gov.br", "municipal_admin").tap do |u|
      Mfa::Enroll.call(u)
      u.update!(otp_enabled: true)
    end
  end
  let(:login_url) { "https://pec.cidade.gov.br/api/recebimento/login" }
  def json = JSON.parse(response.body)

  def sign_in_admin!(stepped_up: true)
    session = sign_in_as(admin)
    session.update!(mfa_verified_at: Time.current) if stepped_up
  end

  def put_credential(kind, body) = put("/integrations/credentials/#{kind}", params: body, as: :json)

  it "GET mostra estado sem segredo, com o que falta por interruptor" do
    city.update!(record_mode: "integrated")
    CityProfile.create!(name: "Cidade", uf: "PR", ibge_code: "4106902")
    sign_in_admin!
    get "/integrations"
    expect(json).to include("record_mode" => "integrated", "pec_url_set" => false, "ibge_code_set" => true)
    expect(json["credentials"]).to eq([
      { "kind" => "ledi", "set" => false, "set_at" => nil, "set_by" => nil, "last_check_at" => nil,
        "last_check_status" => nil, "last_check_message" => nil },
      { "kind" => "cadsus", "set" => false, "set_at" => nil, "set_by" => nil, "last_check_at" => nil,
        "last_check_status" => nil, "last_check_message" => nil }
    ])
    expect(json["features"].first).to eq("key" => "ledi_export", "enabled" => false, "usable" => false,
                                         "missing" => %w[pec_url_missing credential_missing:ledi])
  end

  it "PUT exige step-up, grava cifrado, publica evento só com ids e devolve o item sem segredo" do
    sign_in_admin!(stepped_up: false)
    put_credential("ledi", username: "rota", password: "senha-secreta")
    expect([ response.status, json["error"] ]).to eq([ 401, "mfa_required" ])

    sign_in_admin!
    logs = StringIO.new
    sink = ActiveSupport::Logger.new(logs)
    Rails.logger.broadcast_to(sink)
    put_credential("ledi", username: "rota", password: "senha-secreta")
    expect(response).to have_http_status(:ok)
    expect(json).to include("kind" => "ledi", "set" => true, "set_by" => "admin@cidade.gov.br", "last_check_status" => nil)
    expect(response.body).not_to include("senha-secreta")
    expect(logs.string).not_to include("senha-secreta")
    expect(IntegrationCredential.find_by!(kind: "ledi").password).to eq("senha-secreta")
    expect(DomainEvent.where(name: "integration_credential.changed").map(&:payload))
      .to eq([ { "kind" => "ledi", "user_id" => admin.id } ])
  ensure
    Rails.logger.stop_broadcasting_to(sink) if sink
  end

  it "PUT recusa campo vazio e kind desconhecido; só municipal_admin" do
    sign_in_admin!
    put_credential("ledi", username: "rota", password: " ")
    expect([ response.status, json ]).to eq([ 422, { "error" => "invalid_credential" } ])
    put_credential("rnds", username: "rota", password: "x")
    expect([ response.status, json ]).to eq([ 422, { "error" => "unknown_kind" } ])
    sign_in_as(staff_with("recepcao@cidade.gov.br", "citizen_verifier")).update!(mfa_verified_at: Time.current)
    get "/integrations"
    expect([ response.status, json ]).to eq([ 403, { "error" => "missing_role" } ])
  end

  it "check sem credencial: 409 credential_missing" do
    sign_in_admin!
    post "/integrations/credentials/ledi/check"
    expect([ response.status, json ]).to eq([ 409, { "error" => "credential_missing" } ])
  end

  it "check LEDI: login aceito → ok; recusado → unauthorized e o interruptor passa a faltar" do
    city.update!(pec_url: "https://pec.cidade.gov.br")
    sign_in_admin!
    put_credential("ledi", username: "rota", password: "senha-secreta")
    stub_request(:post, login_url).to_return(status: 200, headers: { "Set-Cookie" => "JSESSIONID=x1; Path=/" })
    post "/integrations/credentials/ledi/check"
    expect(json).to include("last_check_status" => "ok", "last_check_message" => "Login no PEC aceito")

    stub_request(:post, login_url).to_return(status: 401)
    post "/integrations/credentials/ledi/check"
    expect(json["last_check_status"]).to eq("unauthorized")
    get "/integrations"
    expect(json["features"].first["missing"]).to include("credential_unauthorized:ledi")
  end

  # Review Focus 2: PEC lento e 200 sem cookie.
  it "check LEDI: PEC lento → unreachable; 200 sem cookie → error; mensagens sem segredo" do
    city.update!(pec_url: "https://pec.cidade.gov.br")
    sign_in_admin!
    put_credential("ledi", username: "rota", password: "senha-secreta")
    stub_request(:post, login_url).to_timeout
    post "/integrations/credentials/ledi/check"
    expect(json).to include("last_check_status" => "unreachable", "last_check_message" => "PEC inalcançável")
    stub_request(:post, login_url).to_return(status: 200, body: "<html>senha-secreta</html>")
    post "/integrations/credentials/ledi/check"
    expect(json).to include("last_check_status" => "error", "last_check_message" => "O PEC respondeu 200 sem sessão")
    expect(response.body).not_to include("senha-secreta")
    expect(IntegrationCredential.find_by!(kind: "ledi").password).to eq("senha-secreta")
  end

  it "check LEDI sem endereço do PEC: error com a explicação" do
    sign_in_admin!
    put_credential("ledi", username: "rota", password: "senha-secreta")
    post "/integrations/credentials/ledi/check"
    expect(json).to include("last_check_status" => "error",
                            "last_check_message" => "Endereço do PEC não definido pelo operador")
  end

  it "check CADSUS (simulado): ok; usuário recusado → unauthorized" do
    sign_in_admin!
    put_credential("cadsus", username: "rota", password: "x")
    post "/integrations/credentials/cadsus/check"
    expect(json["last_check_status"]).to eq("ok")
    put_credential("cadsus", username: Cadsus::Simulated::REFUSED_USERNAME, password: "x")
    post "/integrations/credentials/cadsus/check"
    expect(json["last_check_status"]).to eq("unauthorized")
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec spec/requests/integrations_spec.rb`
Expected: FAIL (`No route matches [GET] "/integrations"`).

- [ ] **Step 3: Policy e commands**

```ruby
# app/policies/integration_policy.rb
# Integrações e CNES da cidade (ADR 0028): só o municipal_admin.
class IntegrationPolicy < ApplicationPolicy
  def manage? = role?(:municipal_admin)
end
```

```ruby
# app/commands/integrations/set_credential.rb
# Cadastra ou troca a credencial de integração da cidade (ADR 0028; spec
# 2026-10-05 §3.3). Escrita só: nada devolve o segredo. Trocar zera o último
# teste (a credencial nova nunca foi testada). O evento leva só kind e user_id.
# Reasons: :unknown_kind, :invalid_credential.
module Integrations
  module SetCredential
    module_function

    def call(kind:, username:, password:, by:)
      return Result.fail(:unknown_kind) unless IntegrationCredential::KINDS.include?(kind)
      return Result.fail(:invalid_credential) unless [ username, password ].all? { |v| filled?(v) }

      credential = nil
      ApplicationRecord.transaction do
        credential = IntegrationCredential.lock.find_or_initialize_by(kind: kind)
        credential.update!(secret: { "username" => username.strip, "password" => password }, set_by_user: by,
                           set_at: Time.current, last_check_at: nil, last_check_status: nil, last_check_message: nil)
        DomainEvents.publish("integration_credential.changed", kind: kind, user_id: by.id)
      end
      Result.ok(credential: credential)
    rescue ActiveRecord::RecordNotUnique
      retry
    end

    def filled?(value) = value.is_a?(String) && value.strip.present? && value.length <= IntegrationCredential::FIELD_MAX
  end
end
```

```ruby
# app/commands/integrations/check_connection.rb
# Testa a credencial da cidade (ADR 0028; spec 2026-10-05 §3.3): LEDI = login
# no PEC da cidade; CADSUS = consulta de saúde do serviço. Grava o resultado com
# uma mensagem NOSSA — nunca o corpo da resposta nem a mensagem da exceção.
# Reasons: :unknown_kind, :credential_missing.
module Integrations
  module CheckConnection
    module_function

    def call(kind:, city:)
      return Result.fail(:unknown_kind) unless IntegrationCredential::KINDS.include?(kind)

      credential = IntegrationCredential.find_by(kind: kind)
      return Result.fail(:credential_missing) unless credential

      status, message = kind == "ledi" ? check_ledi(city, credential) : check_cadsus(city)
      credential.update!(last_check_at: Time.current, last_check_status: status, last_check_message: message)
      Result.ok(credential: credential)
    end

    def check_ledi(city, credential)
      pec_url = Platform::Features.settings(city)[:pec_url]
      return [ "error", "Endereço do PEC não definido pelo operador" ] if pec_url.blank?

      Ledi::PecClient.new(base_url: pec_url, username: credential.username, password: credential.password).login
      [ "ok", "Login no PEC aceito" ]
    rescue Ledi::PecClient::Unauthorized
      [ "unauthorized", "O PEC recusou usuário ou senha" ]
    rescue Ledi::PecClient::Unreachable
      [ "unreachable", "PEC inalcançável" ]
    rescue Ledi::PecClient::Failed => e
      [ "error", e.status.between?(200, 299) ? "O PEC respondeu #{e.status} sem sessão" : "O PEC respondeu #{e.status}" ]
    end

    def check_cadsus(city)
      Cadsus::Client.for(city).health_check
      [ "ok", "CADSUS respondeu" ]
    rescue Cadsus::Unauthorized
      [ "unauthorized", "O CADSUS recusou a credencial" ]
    rescue Cadsus::Unavailable
      [ "unreachable", "CADSUS inalcançável" ]
    end
  end
end
```

- [ ] **Step 4: Controller, rotas e eventos**

```ruby
# app/controllers/integrations_controller.rb
# Integrações da cidade (ADR 0028; contratos §5.1): estado, cadastro/troca de
# credencial (step-up) e teste de conexão. Só municipal_admin. Nenhuma
# resposta devolve segredo.
class IntegrationsController < ApplicationController
  include Authentication
  include MfaStepUp

  wrap_parameters false

  ERROR_STATUS = { unknown_kind: :unprocessable_entity, invalid_credential: :unprocessable_entity,
                   credential_missing: :conflict }.freeze

  before_action :require_admin

  def show
    city = Current.city
    settings = Platform::Features.settings(city)
    state = Platform::Features.city_state(city)
    credentials = IntegrationCredential.includes(:set_by_user).index_by(&:kind)
    render json: {
      record_mode: settings[:record_mode], pec_url_set: settings[:pec_url].present?,
      ibge_code_set: state&.dig(:ibge_code).present?,
      credentials: IntegrationCredential::KINDS.map { |kind| credential_json(kind, credentials[kind]) },
      features: Platform::Features.summary(city, state: state).map { |f| f.slice(:key, :enabled, :usable, :missing) }
    }
  end

  def update
    return render_error(:unknown_kind) unless IntegrationCredential::KINDS.include?(params[:kind])
    return require_step_up! unless reauthenticated_recently?

    body = request.request_parameters
    result = Integrations::SetCredential.call(kind: params[:kind], username: body["username"],
                                              password: body["password"], by: Current.user)
    return render_error(result.reason) if result.failure?

    render json: credential_json(params[:kind], result.payload[:credential])
  end

  def check
    result = Integrations::CheckConnection.call(kind: params[:kind], city: Current.city)
    return render_error(result.reason) if result.failure?

    render json: credential_json(params[:kind], result.payload[:credential])
  end

  private

  def require_admin
    render json: { error: "missing_role" }, status: :forbidden unless IntegrationPolicy.new(Current.user, nil).manage?
  end

  def render_error(reason)
    render json: { error: reason.to_s }, status: ERROR_STATUS.fetch(reason, :unprocessable_entity)
  end

  def credential_json(kind, credential)
    {
      kind: kind, set: !credential.nil?, set_at: credential&.set_at&.iso8601,
      set_by: credential&.set_by_user&.email_address, last_check_at: credential&.last_check_at&.iso8601,
      last_check_status: credential&.last_check_status, last_check_message: credential&.last_check_message
    }
  end
end
```

Em `config/routes.rb`, depois do bloco `scope "/campaigns"`:

```ruby
  # Integrações da cidade (ADR 0028; contratos §5.1). Prefixo único no proxy
  # de dev do dashboard.
  get  "/integrations", to: "integrations#show"
  put  "/integrations/credentials/:kind", to: "integrations#update"
  post "/integrations/credentials/:kind/check", to: "integrations#check"
```

Em `config/initializers/domain_events.rb`, no fim do bloco:

```ruby
  # Módulo 16 (ADR 0028): trilha; payload só com ids. Sem consumidor.
  DomainEvents.bind "integration_credential.changed", to: []
  DomainEvents.bind "cnes.proposals_applied", to: []
  DomainEvents.bind "citizen.cadsus_looked_up", to: []
```

Em `spec/initializers/domain_events_bindings_spec.rb`, no fim:

```ruby
# Módulo 16 (ADR 0028): eventos declarados, só trilha.
RSpec.describe "record mode event bindings (ADR 0028)" do
  it "declares every module 16 city event with no consumer" do
    names = %w[integration_credential.changed cnes.proposals_applied citizen.cadsus_looked_up]
    expect(DomainEvents.registry.keys).to include(*names)
    expect(names.flat_map { |n| DomainEvents.registry[n] }).to be_empty
  end
end
```

- [ ] **Step 5: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec spec/requests/integrations_spec.rb spec/initializers/domain_events_bindings_spec.rb`
Expected: PASS. (Se `Rails.logger.broadcast_to` não existir na versão em uso, troque a captura por `allow(Rails.logger).to receive(:info).and_call_original` + `expect(Rails.logger).not_to have_received(:info).with(a_string_including("senha-secreta"))` — o que importa é provar que o "Parameters:" filtrado não leva a senha.)

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod16 add app/policies/integration_policy.rb app/commands/integrations/set_credential.rb app/commands/integrations/check_connection.rb app/controllers/integrations_controller.rb config/routes.rb config/initializers/domain_events.rb spec/initializers/domain_events_bindings_spec.rb spec/requests/integrations_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod16 commit -m "feat: let municipal admins set and test integration credentials

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Fatia 3 — Terminologias (F-16.4)

### Task 10: Tabelas de terminologia na plataforma e o trigger de imutabilidade

**Files:**
- Create: `db/platform_migrate/20261005200002_create_terminologies.rb`, `app/models/terminology_release.rb`, `app/models/cid10_code.rb`, `app/models/ciap2_code.rb`, `app/models/sigtap_procedure.rb`, `app/models/sigtap_procedure_cbo.rb`, `app/models/sigtap_procedure_cid.rb`, `app/models/sigtap_procedure_instrument.rb`
- Modify: `db/platform_triggers.sql`, `db/platform_schema.rb` (gerado)
- Test: `spec/models/terminology_tables_guard_spec.rb`

**Interfaces:**
- Produces: `TerminologyRelease` (`kind`, `version`, `source_sha256`, `imported_by`, `imported_at`, `status`, `activated_at`; `KINDS == %w[cid10 ciap2 sigtap]`, `STATUSES == %w[importing active superseded failed]`, `scope :active`); `Cid10Code` (`release_id`, `code`, `description`, `sex_restriction`), `Ciap2Code` (`release_id`, `code`, `description`), `SigtapProcedure` (`release_id`, `code`, `name`, `sex`, `age_min_months`, `age_max_months`, `complexity`), `SigtapProcedureCbo` (`release_id`, `procedure_code`, `cbo_code`), `SigtapProcedureCid` (`release_id`, `procedure_code`, `cid_code`, `principal`), `SigtapProcedureInstrument` (`release_id`, `procedure_code`, `instrument_code`, `instrument_name`). Todos com `belongs_to :release, class_name: "TerminologyRelease"`.

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/models/terminology_tables_guard_spec.rb
require "rails_helper"

# ADR 0028 (spec 2026-10-05 §4): release ativa nunca muda; release com falha
# nunca fica ativa; códigos só entram numa release em importação e não mudam
# depois de ativa. Garantido pelo banco (db/platform_triggers.sql).
RSpec.describe "Terminologias: guarda do banco" do
  def sql(statement) = PlatformRecord.transaction(requires_new: true) { PlatformRecord.connection.execute(statement) }

  def release!(status: "importing", version: "202610", kind: "sigtap")
    TerminologyRelease.create!(kind: kind, version: version, source_sha256: "a" * 64, imported_by: "rspec",
                               imported_at: Time.current, status: "importing").tap do |r|
      r.update!(status: status, activated_at: (Time.current if status == "active")) unless status == "importing"
    end
  end

  it "transições permitidas: importing → active|failed, active → superseded; o resto é recusado" do
    active = release!(status: "active")
    expect { sql("UPDATE terminology_releases SET status = 'importing' WHERE id = '#{active.id}'") }
      .to raise_error(ActiveRecord::StatementInvalid, /transition/)
    failed = release!(status: "failed", version: "202609")
    expect { sql("UPDATE terminology_releases SET status = 'active' WHERE id = '#{failed.id}'") }
      .to raise_error(ActiveRecord::StatementInvalid, /transition/)
    expect { active.update!(status: "superseded") }.not_to raise_error
  end

  it "release ativa ou substituída: nenhum outro campo muda e não se apaga" do
    active = release!(status: "active")
    expect { sql("UPDATE terminology_releases SET version = '202611' WHERE id = '#{active.id}'") }
      .to raise_error(ActiveRecord::StatementInvalid, /immutable/)
    expect { sql("DELETE FROM terminology_releases WHERE id = '#{active.id}'") }
      .to raise_error(ActiveRecord::StatementInvalid, /immutable/)
  end

  it "códigos: entram só em release importing; não mudam nem saem de release ativa" do
    importing = release!(kind: "ciap2", version: "2")
    Ciap2Code.create!(release: importing, code: "K86", description: "Hipertensão sem complicações")
    importing.update!(status: "active", activated_at: Time.current)
    expect { sql("INSERT INTO ciap2_codes (release_id, code, description) VALUES ('#{importing.id}', 'T90', 'x')") }
      .to raise_error(ActiveRecord::StatementInvalid, /being imported/)
    expect { sql("UPDATE ciap2_codes SET description = 'y'") }.to raise_error(ActiveRecord::StatementInvalid, /immutable/)
    expect { sql("DELETE FROM ciap2_codes") }.to raise_error(ActiveRecord::StatementInvalid, /immutable/)
  end

  it "no máximo uma ativa por kind e versão; versão da SIGTAP é AAAAMM" do
    release!(status: "active")
    expect { release!(status: "active") }.to raise_error(ActiveRecord::RecordNotUnique)
    expect { release!(version: "2026-10") }.to raise_error(ActiveRecord::StatementInvalid)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec spec/models/terminology_tables_guard_spec.rb`
Expected: FAIL (`uninitialized constant TerminologyRelease`).

- [ ] **Step 3: Migração e trigger**

```ruby
# db/platform_migrate/20261005200002_create_terminologies.rb
# Terminologias nacionais na plataforma (ADR 0028; spec 2026-10-05 §4):
# CID-10, CIAP-2 e SIGTAP versionadas (a SIGTAP por competência AAAAMM),
# importadas pelo operador, ativadas só no fim e nunca apagadas. Dado público,
# sem dado de pessoa. Os códigos referenciam o procedimento pelo CÓDIGO (não
# por id), como o arquivo oficial, para a carga em lote. Imutabilidade por
# trigger: db/platform_triggers.sql.
class CreateTerminologies < ActiveRecord::Migration[8.1]
  def self.text_in(column, values)
    "#{column}::text = ANY (ARRAY[#{values.map { |v| "'#{v}'::text" }.join(', ')}])"
  end

  def up
    create_table :terminology_releases, id: :uuid, default: -> { "gen_random_uuid()" } do |t|
      t.string :kind, null: false
      t.string :version, null: false, limit: 20
      t.string :source_sha256, null: false, limit: 64
      t.string :imported_by, null: false, limit: 80
      t.datetime :imported_at, null: false
      t.string :status, null: false, default: "importing"
      t.datetime :activated_at
      t.timestamps
      t.index %i[kind version], unique: true, where: "((status)::text = 'active'::text)",
                                name: "idx_terminology_releases_one_active"
      t.index %i[kind status version], name: "idx_terminology_releases_lookup"
      t.check_constraint text_in("kind", %w[cid10 ciap2 sigtap]), name: "ck_terminology_releases_kind"
      t.check_constraint text_in("status", %w[importing active superseded failed]), name: "ck_terminology_releases_status"
      t.check_constraint "kind::text <> 'sigtap'::text OR version::text ~ '^[0-9]{6}$'::text",
                         name: "ck_terminology_releases_sigtap_version"
    end

    create_table :cid10_codes do |t|
      t.uuid :release_id, null: false
      t.string :code, null: false, limit: 4
      t.text :description, null: false
      t.string :sex_restriction, limit: 1
      t.index %i[release_id code], unique: true
      t.check_constraint "sex_restriction IS NULL OR #{text_in('sex_restriction', %w[F M])}", name: "ck_cid10_codes_sex"
    end

    create_table :ciap2_codes do |t|
      t.uuid :release_id, null: false
      t.string :code, null: false, limit: 3
      t.text :description, null: false
      t.index %i[release_id code], unique: true
    end

    create_table :sigtap_procedures do |t|
      t.uuid :release_id, null: false
      t.string :code, null: false, limit: 10
      t.text :name, null: false
      t.string :sex, limit: 1
      t.integer :age_min_months
      t.integer :age_max_months
      t.string :complexity, limit: 1
      t.index %i[release_id code], unique: true
    end

    create_table :sigtap_procedure_cbos do |t|
      t.uuid :release_id, null: false
      t.string :procedure_code, null: false, limit: 10
      t.string :cbo_code, null: false, limit: 6
      t.index %i[release_id procedure_code cbo_code], unique: true, name: "idx_sigtap_procedure_cbos_unique"
    end

    create_table :sigtap_procedure_cids do |t|
      t.uuid :release_id, null: false
      t.string :procedure_code, null: false, limit: 10
      t.string :cid_code, null: false, limit: 4
      t.boolean :principal, null: false, default: false
      t.index %i[release_id procedure_code cid_code], unique: true, name: "idx_sigtap_procedure_cids_unique"
    end

    create_table :sigtap_procedure_instruments do |t|
      t.uuid :release_id, null: false
      t.string :procedure_code, null: false, limit: 10
      t.string :instrument_code, null: false, limit: 2
      t.string :instrument_name, null: false
      t.index %i[release_id procedure_code instrument_code], unique: true, name: "idx_sigtap_procedure_instruments_unique"
    end

    %i[cid10_codes ciap2_codes sigtap_procedures sigtap_procedure_cbos sigtap_procedure_cids
       sigtap_procedure_instruments].each do |table|
      add_foreign_key table, :terminology_releases, column: :release_id
    end

    execute File.read(Rails.root.join("db/platform_triggers.sql"))
  end

  def down
    execute <<~SQL
      DROP TRIGGER IF EXISTS terminology_releases_guard ON terminology_releases;
      DROP FUNCTION IF EXISTS terminology_release_guard() CASCADE;
      DROP FUNCTION IF EXISTS terminology_codes_guard() CASCADE;
    SQL
    %i[sigtap_procedure_instruments sigtap_procedure_cids sigtap_procedure_cbos sigtap_procedures ciap2_codes
       cid10_codes terminology_releases].each { |table| drop_table table }
  end
end
```

Acrescente ao fim de `db/platform_triggers.sql`:

```sql
-- Terminologias (ADR 0028; spec 2026-10-05 §4): release ativa nunca muda,
-- release com falha nunca fica ativa, nada ativo ou substituído se apaga. Só
-- status (pelas transições abaixo), activated_at (na ativação) e updated_at
-- mudam.
CREATE OR REPLACE FUNCTION terminology_release_guard() RETURNS trigger AS $fn$
BEGIN
  IF TG_OP = 'DELETE' THEN
    IF OLD.status IN ('active', 'superseded') THEN
      RAISE EXCEPTION 'terminology release % % is immutable: DELETE refused', OLD.kind, OLD.version;
    END IF;
    RETURN OLD;
  END IF;

  IF NEW.id IS DISTINCT FROM OLD.id OR NEW.kind IS DISTINCT FROM OLD.kind
     OR NEW.version IS DISTINCT FROM OLD.version OR NEW.source_sha256 IS DISTINCT FROM OLD.source_sha256
     OR NEW.imported_by IS DISTINCT FROM OLD.imported_by OR NEW.imported_at IS DISTINCT FROM OLD.imported_at
     OR NEW.created_at IS DISTINCT FROM OLD.created_at
     OR (OLD.status <> 'importing' AND NEW.activated_at IS DISTINCT FROM OLD.activated_at) THEN
    RAISE EXCEPTION 'terminology release % % is immutable: only status may change', OLD.kind, OLD.version;
  END IF;

  IF NEW.status IS DISTINCT FROM OLD.status AND NOT (
       (OLD.status = 'importing' AND NEW.status IN ('active', 'failed'))
       OR (OLD.status = 'active' AND NEW.status = 'superseded')) THEN
    RAISE EXCEPTION 'terminology release % %: transition % -> % refused', OLD.kind, OLD.version, OLD.status, NEW.status;
  END IF;

  RETURN NEW;
END;
$fn$ LANGUAGE plpgsql;

-- O arquivo inteiro roda também em bancos ainda sem estas tabelas (migrações
-- antigas que o executam, platform:triggers): o trigger só é instalado onde a
-- tabela existe.
DO $$
BEGIN
  IF to_regclass('public.terminology_releases') IS NOT NULL THEN
    DROP TRIGGER IF EXISTS terminology_releases_guard ON terminology_releases;
    CREATE TRIGGER terminology_releases_guard
      BEFORE UPDATE OR DELETE ON terminology_releases
      FOR EACH ROW EXECUTE FUNCTION terminology_release_guard();
  END IF;
END $$;

-- Códigos: entram só numa release em importação; não mudam nem saem de
-- release ativa ou substituída.
CREATE OR REPLACE FUNCTION terminology_codes_guard() RETURNS trigger AS $fn$
DECLARE
  release_status text;
BEGIN
  SELECT status INTO release_status FROM terminology_releases
   WHERE id = CASE WHEN TG_OP = 'DELETE' THEN OLD.release_id ELSE NEW.release_id END;

  IF TG_OP = 'INSERT' THEN
    IF release_status IS DISTINCT FROM 'importing' THEN
      RAISE EXCEPTION '% accepts rows only for a release being imported', TG_TABLE_NAME;
    END IF;
    RETURN NEW;
  END IF;

  IF release_status IN ('active', 'superseded') THEN
    RAISE EXCEPTION '% rows of an active or superseded release are immutable: % refused', TG_TABLE_NAME, TG_OP;
  END IF;

  IF TG_OP = 'DELETE' THEN
    RETURN OLD;
  END IF;
  RETURN NEW;
END;
$fn$ LANGUAGE plpgsql;

DO $$
DECLARE
  t text;
BEGIN
  FOREACH t IN ARRAY ARRAY['cid10_codes', 'ciap2_codes', 'sigtap_procedures', 'sigtap_procedure_cbos',
                           'sigtap_procedure_cids', 'sigtap_procedure_instruments'] LOOP
    CONTINUE WHEN to_regclass('public.' || t) IS NULL;
    EXECUTE format('DROP TRIGGER IF EXISTS %I ON %I', t || '_guard', t);
    EXECUTE format('CREATE TRIGGER %I BEFORE INSERT OR UPDATE OR DELETE ON %I FOR EACH ROW ' ||
                   'EXECUTE FUNCTION terminology_codes_guard()', t || '_guard', t);
  END LOOP;
END $$;
```

- [ ] **Step 4: Modelos**

```ruby
# app/models/terminology_release.rb
# Uma versão importada de terminologia nacional (ADR 0028; spec 2026-10-05
# §4). Escrita só por Terminology::Import; imutabilidade por trigger.
class TerminologyRelease < PlatformRecord
  KINDS = %w[cid10 ciap2 sigtap].freeze
  STATUSES = %w[importing active superseded failed].freeze

  validates :kind, inclusion: { in: KINDS }
  validates :status, inclusion: { in: STATUSES }
  validates :version, :source_sha256, :imported_by, :imported_at, presence: true

  scope :active, -> { where(status: "active") }
end
```

```ruby
# app/models/cid10_code.rb
class Cid10Code < PlatformRecord
  belongs_to :release, class_name: "TerminologyRelease"
end
```

```ruby
# app/models/ciap2_code.rb
class Ciap2Code < PlatformRecord
  belongs_to :release, class_name: "TerminologyRelease"
end
```

```ruby
# app/models/sigtap_procedure.rb
class SigtapProcedure < PlatformRecord
  belongs_to :release, class_name: "TerminologyRelease"
end
```

```ruby
# app/models/sigtap_procedure_cbo.rb
class SigtapProcedureCbo < PlatformRecord
  belongs_to :release, class_name: "TerminologyRelease"
end
```

```ruby
# app/models/sigtap_procedure_cid.rb
class SigtapProcedureCid < PlatformRecord
  belongs_to :release, class_name: "TerminologyRelease"
end
```

```ruby
# app/models/sigtap_procedure_instrument.rb
class SigtapProcedureInstrument < PlatformRecord
  belongs_to :release, class_name: "TerminologyRelease"
end
```

- [ ] **Step 5: Migre a plataforma (dev e test) e confira o dump**

```bash
docker compose exec -T -w /rails/.claude/mod16 api bin/rails db:migrate
docker compose exec -T -w /rails/.claude/mod16 -e RAILS_ENV=test -e POSTGRES_PASSWORD=postgres api bin/rails db:migrate
docker compose exec -T -w /rails/.claude/mod16 api bin/rails platform:triggers
/opt/homebrew/bin/git -C apps/api/.claude/mod16 diff --stat db/platform_schema.rb
```
Expected: `define(version: 2026_10_05_200002)`, as sete tabelas novas e seis `add_foreign_key ... column: "release_id"`; `platform:triggers` termina sem erro (prova de idempotência).

- [ ] **Step 6: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec spec/models/terminology_tables_guard_spec.rb spec/architecture`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod16 add db/platform_migrate/20261005200002_create_terminologies.rb db/platform_triggers.sql db/platform_schema.rb app/models/terminology_release.rb app/models/cid10_code.rb app/models/ciap2_code.rb app/models/sigtap_procedure.rb app/models/sigtap_procedure_cbo.rb app/models/sigtap_procedure_cid.rb app/models/sigtap_procedure_instrument.rb spec/models/terminology_tables_guard_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod16 commit -m "feat: add versioned terminology tables with immutability triggers

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 11: `OfficialArchive`, importação de CID-10 e CIAP-2 e `terminology:import`

**Files:**
- Create: `app/services/official_archive.rb`, `app/services/terminology/import.rb`, `app/services/terminology/cid10_reader.rb`, `app/services/terminology/ciap2_reader.rb`, `lib/tasks/terminology.rake`, `spec/fixtures/terminology/cid10/CID-10-CATEGORIAS.CSV`, `spec/fixtures/terminology/cid10/CID-10-SUBCATEGORIAS.CSV`, `spec/fixtures/terminology/ciap2/ciap2.csv`
- Modify: `Gemfile`, `Gemfile.lock`
- Test: `spec/services/official_archive_spec.rb`, `spec/services/terminology/import_spec.rb`, `spec/tasks/terminology_rake_spec.rb`

**Interfaces:**
- Consumes: modelos da Task 10.
- Produces:
  - `OfficialArchive.open(path) { |archive| }` (pasta ou ZIP; `OfficialArchive::NotFound`); `#sha256 -> String`; `#each_row(pattern, encoding: "ISO-8859-1", col_sep: ";") { |Hash{"COLUNA" => valor}| }` (cabeçalho em maiúsculas); `#each_line(pattern, encoding: "ISO-8859-1") { |String| }`; `pattern` é `Regexp` casado contra o nome do arquivo (sem pasta), sem diferenciar maiúsculas.
  - `Terminology::Import.call(kind:, version:, path:, by: "terminology:import") -> Result` (`ok(release:, counts:)`; `fail` com `:unknown_kind`, `:invalid_version`, `:file_not_found`, `:invalid_file`); `Terminology::Import::Invalid < StandardError`; `Terminology::Import::BATCH = 1_000`; `Terminology::Import.readers` (Hash kind → classe do leitor).
  - Contrato do leitor: `Reader.new(archive).write(release) -> Hash{tabela => contagem}`, levanta `Import::Invalid` com mensagem nossa.

- [ ] **Step 1: Gem**

Em `Gemfile`, depois de `gem "json_schemer"`:

```ruby
# Arquivos oficiais do DATASUS chegam em ZIP (SIGTAP, CNES) — ADR 0028.
gem "rubyzip", "~> 2.4", require: "zip"
```

Run: `docker compose exec -T -w /rails/.claude/mod16 api bundle install`
Expected: `Gemfile.lock` com `rubyzip (2.4.x)`.

- [ ] **Step 2: Fixtures (recorte do formato oficial, só ASCII)**

`spec/fixtures/terminology/cid10/CID-10-CATEGORIAS.CSV`:

```
CAT;CLASSIF;DESCRICAO;DESCRABREV;REFOUTRA;REFERENCIA
E11;;Diabetes mellitus nao-insulino-dependente;E11 Diabetes mellitus nao-insulino-depend;;
I10;;Hipertensao essencial (primaria);I10 Hipertensao essencial;;
O80;;Parto unico espontaneo;O80 Parto unico espontaneo;;
Z01;;Outros exames e investigacoes especiais;Z01 Outr exames e investig espec;;
```

`spec/fixtures/terminology/cid10/CID-10-SUBCATEGORIAS.CSV`:

```
SUBCAT;CLASSIF;RESTRSEXO;CAUSAOBITO;DESCRICAO;DESCRABREV;REFOUTRA;REFERENCIA;EXCLUIDOS
E119;;;;Diabetes mellitus nao-insulino-dependente - sem complicacoes;E11.9 Diab mell n-insul-dep s/compl;;;
O800;;F;N;Parto espontaneo cefalico;O80.0 Parto espontaneo cefalico;;;
Z014;;F;N;Exame ginecologico (geral) (de rotina);Z01.4 Exame ginecologico;;;
```

`spec/fixtures/terminology/ciap2/ciap2.csv`:

```
codigo;titulo
A01;Dor generalizada/multipla
K86;Hipertensao sem complicacoes
T90;Diabetes nao insulino-dependente
W78;Gravidez
```

- [ ] **Step 3: Escreva as specs que falham**

```ruby
# spec/services/official_archive_spec.rb
require "rails_helper"

# ADR 0028: o arquivo oficial chega em ZIP ou já extraído; o leitor é o mesmo.
RSpec.describe OfficialArchive do
  let(:dir) { Rails.root.join("spec/fixtures/terminology/cid10") }
  let(:tmp) { Pathname(Dir.mktmpdir) }
  after { FileUtils.remove_entry(tmp) }

  def zip_of(source)
    path = tmp.join("oficial.zip")
    Zip::File.open(path.to_s, create: true) do |zip|
      source.children.each { |file| zip.add("pasta/#{file.basename}", file.to_s) }
    end
    path
  end

  it "lê linhas por cabeçalho, da pasta e do ZIP, sem diferenciar maiúsculas no nome" do
    from_dir = []
    described_class.open(dir) { |a| a.each_row(/\Acid-10-categorias\.csv\z/i) { |row| from_dir << row } }
    from_zip = []
    described_class.open(zip_of(dir)) { |a| a.each_row(/\Acid-10-categorias\.csv\z/i) { |row| from_zip << row } }
    expect(from_dir.first).to include("CAT" => "E11", "DESCRICAO" => "Diabetes mellitus nao-insulino-dependente")
    expect(from_zip).to eq(from_dir)
  end

  it "converte ISO-8859-1 para UTF-8" do
    tmp.join("x.csv").binwrite("codigo;titulo\nK86;Hipertens\xE3o\n".b)
    rows = []
    described_class.open(tmp) { |a| a.each_row(/\Ax\.csv\z/) { |r| rows << r } }
    expect(rows).to eq([ { "CODIGO" => "K86", "TITULO" => "Hipertensão" } ])
  end

  it "sha256 estável; caminho inexistente, ZIP quebrado e arquivo ausente levantam NotFound" do
    a = described_class.open(dir, &:sha256)
    expect(described_class.open(dir, &:sha256)).to eq(a)
    expect { described_class.open(tmp.join("nada")) { nil } }.to raise_error(described_class::NotFound)
    tmp.join("ruim.zip").write("não é zip")
    expect { described_class.open(tmp.join("ruim.zip")) { nil } }.to raise_error(described_class::NotFound)
    expect { described_class.open(dir) { |x| x.each_row(/\Aoutro\.csv\z/) { nil } } }.to raise_error(described_class::NotFound)
  end
end
```

```ruby
# spec/services/terminology/import_spec.rb
require "rails_helper"

# ADR 0028 (spec 2026-10-05 §4): importa em transação com status importing,
# ativa no fim (a anterior vira superseded); falha → failed e nada ativo muda.
RSpec.describe Terminology::Import do
  let(:cid10) { Rails.root.join("spec/fixtures/terminology/cid10") }
  let(:ciap2) { Rails.root.join("spec/fixtures/terminology/ciap2") }
  let(:tmp) { Pathname(Dir.mktmpdir) }
  after { FileUtils.remove_entry(tmp) }

  it "CID-10: categorias e subcategorias, com restrição de sexo; audita a ativação" do
    result = described_class.call(kind: "cid10", version: "2008", path: cid10)

    expect(result).to be_ok
    release = result.payload[:release]
    expect(release).to have_attributes(status: "active", kind: "cid10", version: "2008")
    expect(Cid10Code.where(release: release).pluck(:code)).to contain_exactly("E11", "I10", "O80", "Z01", "E119", "O800", "Z014")
    expect(Cid10Code.find_by!(release: release, code: "O800").sex_restriction).to eq("F")
    expect(result.payload[:counts]).to eq(cid10_codes: 7)
    expect(PlatformEvent.where(name: "terminology.release_activated").map(&:payload))
      .to eq([ { "kind" => "cid10", "version" => "2008" } ])
  end

  it "nova versão substitui a anterior do mesmo kind" do
    first = described_class.call(kind: "ciap2", version: "2", path: ciap2).payload[:release]
    second = described_class.call(kind: "ciap2", version: "2.1", path: ciap2).payload[:release]
    expect(first.reload.status).to eq("superseded")
    expect(second.status).to eq("active")
    expect(TerminologyRelease.active.where(kind: "ciap2").count).to eq(1)
  end

  it "arquivo com código inválido: failed, nenhum código gravado, a ativa continua" do
    active = described_class.call(kind: "ciap2", version: "2", path: ciap2).payload[:release]
    tmp.join("ciap2.csv").write("codigo;titulo\nK86;Hipertensao\nXYZ1;quebrado\n")

    result = described_class.call(kind: "ciap2", version: "3", path: tmp)

    expect(result.reason).to eq(:invalid_file)
    expect(result.message).to include("XYZ1")
    failed = TerminologyRelease.find_by!(kind: "ciap2", version: "3")
    expect(failed.status).to eq("failed")
    expect(Ciap2Code.where(release: failed)).to be_empty
    expect(active.reload.status).to eq("active")
  end

  it "recusa kind, versão e caminho inválidos sem criar release" do
    expect(described_class.call(kind: "cbo", version: "1", path: ciap2).reason).to eq(:unknown_kind)
    expect(described_class.call(kind: "sigtap", version: "2026-10", path: ciap2).reason).to eq(:invalid_version)
    expect(described_class.call(kind: "sigtap", version: "202613", path: ciap2).reason).to eq(:invalid_version)
    expect(described_class.call(kind: "ciap2", version: "2", path: tmp.join("nada")).reason).to eq(:file_not_found)
    expect(TerminologyRelease.count).to eq(0)
  end
end
```

```ruby
# spec/tasks/terminology_rake_spec.rb
require "rails_helper"
require "rake"

RSpec.describe "terminology:import rake task" do
  before(:all) { Rails.application.load_tasks unless Rake::Task.task_defined?("terminology:import") }
  before { Rake::Task["terminology:import"].reenable }

  def run(*args)
    out = StringIO.new
    original, $stdout = $stdout, out
    Rake::Task["terminology:import"].invoke(*args)
    out.string
  ensure
    $stdout = original
  end

  it "importa e relata" do
    expect(run("ciap2", "2", Rails.root.join("spec/fixtures/terminology/ciap2").to_s)).to include("ciap2 2 ativa (ciap2_codes: 4)")
  end

  it "falha sai com abort e o motivo" do
    expect { run("ciap2", "2", "/nao/existe") }.to raise_error(SystemExit)
  end
end
```

- [ ] **Step 4: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec spec/services/official_archive_spec.rb spec/services/terminology/import_spec.rb spec/tasks/terminology_rake_spec.rb`
Expected: FAIL (`uninitialized constant OfficialArchive`).

- [ ] **Step 5: Implemente**

```ruby
# app/services/official_archive.rb
require "csv"
require "digest"

# Arquivo oficial do DATASUS (ADR 0028): ZIP ou pasta já extraída, com o mesmo
# leitor. Lê em fluxo (a base do CNES tem milhões de linhas), linha a linha,
# convertendo a codificação para UTF-8. Acha o arquivo por nome, sem
# diferenciar maiúsculas, em qualquer subpasta.
class OfficialArchive
  class NotFound < StandardError; end

  def self.open(path)
    path = Pathname(path.to_s)
    raise NotFound, "arquivo não encontrado: #{path}" unless path.exist?
    return yield(new(path, nil)) if path.directory?

    Zip::File.open(path.to_s) { |zip| return yield(new(path, zip)) }
  rescue Zip::Error
    raise NotFound, "não é um ZIP legível: #{path.basename}"
  end

  def initialize(path, zip)
    @path = path
    @zip = zip
  end

  def sha256
    return Digest::SHA256.file(@path.to_s).hexdigest if @zip

    files = Dir.glob(@path.join("**/*").to_s).select { |f| File.file?(f) }.sort
    Digest::SHA256.hexdigest(files.map { |f| "#{Pathname(f).relative_path_from(@path)}:#{Digest::SHA256.file(f).hexdigest}" }.join("\n"))
  end

  def each_line(pattern, encoding: "ISO-8859-1")
    open_entry(pattern) do |io|
      io.each_line do |raw|
        line = raw.dup.force_encoding(encoding).encode("UTF-8", invalid: :replace, undef: :replace).delete_prefix("﻿").chomp
        yield line unless line.strip.empty?
      end
    end
  end

  def each_row(pattern, encoding: "ISO-8859-1", col_sep: ";")
    headers = nil
    each_line(pattern, encoding: encoding) do |line|
      fields = CSV.parse_line(line, col_sep: col_sep) || []
      if headers.nil?
        headers = fields.map { |h| h.to_s.strip.upcase }
        next
      end
      yield headers.zip(fields.map { |v| v&.strip }).to_h
    end
  end

  private

  def open_entry(pattern, &block)
    name = entry_names.find { |n| File.basename(n).match?(pattern) }
    raise NotFound, "arquivo #{pattern.source} ausente em #{@path.basename}" unless name
    return @zip.get_entry(name).get_input_stream(&block) if @zip

    File.open(@path.join(name), "rb", &block)
  end

  def entry_names
    return @zip.entries.reject(&:directory?).map(&:name) if @zip

    Dir.glob(@path.join("**/*").to_s).select { |f| File.file?(f) }.map { |f| Pathname(f).relative_path_from(@path).to_s }
  end
end
```

```ruby
# app/services/terminology/import.rb
# Importação de terminologia nacional (ADR 0028; spec 2026-10-05 §4). A release
# nasce `importing` FORA da transação (para a falha ficar registrada); os
# códigos e a ativação entram numa transação só (savepoint), com a auditoria
# — tudo no banco de plataforma. Falha → `failed`, e nada ativo muda.
# Reasons: :unknown_kind, :invalid_version, :file_not_found, :invalid_file.
module Terminology
  module Import
    class Invalid < StandardError; end

    BATCH = 1_000
    VERSION = { "sigtap" => /\A\d{4}(0[1-9]|1[0-2])\z/ }.freeze
    DEFAULT_VERSION = /\A[0-9A-Za-z][0-9A-Za-z.-]{0,19}\z/

    module_function

    def readers = { "cid10" => Cid10Reader, "ciap2" => Ciap2Reader, "sigtap" => SigtapReader }

    def call(kind:, version:, path:, by: "terminology:import")
      reader = readers[kind.to_s]
      return Result.fail(:unknown_kind) unless reader
      return Result.fail(:invalid_version) unless version.to_s.match?(VERSION.fetch(kind.to_s, DEFAULT_VERSION))

      OfficialArchive.open(path) { |archive| import(reader, archive, kind.to_s, version.to_s, by) }
    rescue OfficialArchive::NotFound => e
      Result.fail(:file_not_found, message: e.message)
    end

    def import(reader, archive, kind, version, by)
      release = TerminologyRelease.create!(kind: kind, version: version, source_sha256: archive.sha256,
                                           imported_by: by, imported_at: Time.current, status: "importing")
      counts = PlatformRecord.transaction(requires_new: true) do
        written = reader.new(archive).write(release)
        activate!(release)
        Platform.audit("terminology.release_activated", kind: kind, version: version)
        written
      end
      Result.ok(release: release.reload, counts: counts)
    rescue Invalid, OfficialArchive::NotFound, ActiveRecord::ActiveRecordError, CSV::MalformedCSVError => e
      release&.update!(status: "failed")
      Result.fail(:invalid_file, message: e.message.truncate(300))
    end

    # CID-10 e CIAP-2: uma ativa por kind. SIGTAP: uma ativa por competência (a
    # republicação da mesma competência substitui a anterior).
    def activate!(release)
      previous = TerminologyRelease.active.where(kind: release.kind).where.not(id: release.id)
      previous = previous.where(version: release.version) if release.kind == "sigtap"
      previous.lock.each { |old| old.update!(status: "superseded") }
      release.update!(status: "active", activated_at: Time.current)
    end

    def insert(model, rows)
      rows.each_slice(BATCH) { |slice| model.insert_all!(slice) }
      rows.size
    end
  end
end
```

```ruby
# app/services/terminology/cid10_reader.rb
# CID-10 do DATASUS (ADR 0028; desvio 10): CID-10-CATEGORIAS.CSV (3 caracteres)
# e CID-10-SUBCATEGORIAS.CSV (4 caracteres, com RESTRSEXO), ";" e ISO-8859-1.
module Terminology
  class Cid10Reader
    CODE = /\A[A-Z]\d{2}[0-9X]?\z/

    def initialize(archive)
      @archive = archive
    end

    def write(release)
      rows = []
      @archive.each_row(/\ACID-10-CATEGORIAS\.CSV\z/i) { |r| rows << row(release, r["CAT"], r["DESCRICAO"], nil) }
      @archive.each_row(/\ACID-10-SUBCATEGORIAS\.CSV\z/i) { |r| rows << row(release, r["SUBCAT"], r["DESCRICAO"], r["RESTRSEXO"]) }
      raise Import::Invalid, "nenhum código CID-10 no arquivo" if rows.empty?

      { cid10_codes: Import.insert(Cid10Code, rows) }
    end

    private

    def row(release, code, description, sex)
      code = code.to_s.strip.upcase
      raise Import::Invalid, "código CID-10 inválido: #{code.inspect}" unless code.match?(CODE)
      raise Import::Invalid, "descrição vazia em #{code}" if description.to_s.strip.empty?

      sex = sex.to_s.strip.upcase.presence
      raise Import::Invalid, "restrição de sexo inválida em #{code}: #{sex}" unless sex.nil? || %w[F M].include?(sex)

      { release_id: release.id, code: code, description: description.strip, sex_restriction: sex }
    end
  end
end
```

```ruby
# app/services/terminology/ciap2_reader.rb
# CIAP-2 (ADR 0028; desvio 10): CSV `codigo;titulo`, UTF-8.
module Terminology
  class Ciap2Reader
    CODE = /\A[A-Z]\d{2}\z/

    def initialize(archive)
      @archive = archive
    end

    def write(release)
      rows = []
      @archive.each_row(/\Aciap2\.csv\z/i, encoding: "UTF-8") do |r|
        code = r["CODIGO"].to_s.strip.upcase
        raise Import::Invalid, "código CIAP-2 inválido: #{code.inspect}" unless code.match?(CODE)
        raise Import::Invalid, "título vazio em #{code}" if r["TITULO"].to_s.strip.empty?

        rows << { release_id: release.id, code: code, description: r["TITULO"].strip }
      end
      raise Import::Invalid, "nenhum código CIAP-2 no arquivo" if rows.empty?

      { ciap2_codes: Import.insert(Ciap2Code, rows) }
    end
  end
end
```

`Terminology::Import.readers` cita `SigtapReader`, que nasce na Task 12: até lá, crie `app/services/terminology/sigtap_reader.rb` com o esqueleto abaixo (a Task 12 o substitui):

```ruby
# app/services/terminology/sigtap_reader.rb
module Terminology
  class SigtapReader
    def initialize(archive)
      @archive = archive
    end

    def write(_release) = raise(Import::Invalid, "leitor da SIGTAP ainda não implementado")
  end
end
```

```ruby
# lib/tasks/terminology.rake
# Importação de terminologia nacional pelo operador (ADR 0028; spec
# 2026-10-05 §4). path: o ZIP oficial ou a pasta extraída.
namespace :terminology do
  desc "Importa uma terminologia. Uso: terminology:import[kind,version,path] (kind: cid10|ciap2|sigtap; SIGTAP: version AAAAMM)"
  task :import, %i[kind version path] => :environment do |_t, args|
    abort "uso: rails 'terminology:import[kind,version,path]'" if %i[kind version path].any? { |k| args[k].blank? }

    result = Terminology::Import.call(kind: args[:kind], version: args[:version], path: args[:path])
    abort "[terminology:import] #{result.reason}: #{result.message}" if result.failure?

    counts = result.payload[:counts].map { |table, n| "#{table}: #{n}" }.join(", ")
    puts "[terminology:import] #{args[:kind]} #{args[:version]} ativa (#{counts})"
  end
end
```

- [ ] **Step 6: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec spec/services/official_archive_spec.rb spec/services/terminology/import_spec.rb spec/tasks/terminology_rake_spec.rb spec/events/platform_event_payload_guard_spec.rb`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod16 add Gemfile Gemfile.lock app/services/official_archive.rb app/services/terminology/import.rb app/services/terminology/cid10_reader.rb app/services/terminology/ciap2_reader.rb app/services/terminology/sigtap_reader.rb lib/tasks/terminology.rake spec/fixtures/terminology spec/services/official_archive_spec.rb spec/services/terminology/import_spec.rb spec/tasks/terminology_rake_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod16 commit -m "feat: import CID-10 and CIAP-2 releases from official files

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 12: Leitor da SIGTAP (largura fixa, layout do próprio ZIP) e o recorte de amostra

**Files:**
- Create: `lib/sigtap_sample.rb`
- Modify: `app/services/terminology/sigtap_reader.rb` (substitui o esqueleto da Task 11)
- Test: `spec/services/terminology/sigtap_reader_spec.rb`

**Interfaces:**
- Consumes: `OfficialArchive#each_row`/`#each_line`, `Terminology::Import.insert`, `Import::Invalid` (Task 11).
- Produces: `Terminology::SigtapReader#write(release) -> { sigtap_procedures:, sigtap_procedure_cbos:, sigtap_procedure_cids:, sigtap_procedure_instruments: }`; `Terminology::SigtapReader::NO_LIMIT = 9999`; `SigtapSample.write_to(dir, competence:) -> Pathname` (escreve os cinco `.txt` e os cinco `_layout.txt` no formato do DATASUS, ISO-8859-1, CRLF), `SigtapSample::PROCEDURES`, `::CBOS`, `::CIDS`, `::INSTRUMENTS`, `::REGISTRIES`, `::LAYOUTS` — usados pela semente (Task 21) e pelas specs das Tasks 12, 13 e 20.

- [ ] **Step 1: O recorte de amostra**

```ruby
# lib/sigtap_sample.rb
require "fileutils"

# Recorte da Tabela Unificada SIGTAP (ADR 0028; spec 2026-10-05 §4, §9, §10):
# códigos reais de procedimentos da APS, no formato do ZIP oficial — cada
# arquivo de largura fixa com o seu `<arquivo>_layout.txt` (Coluna, Tamanho,
# Inicio, Fim, Tipo). Idade em meses (9999 = sem limite). Os atributos do
# recorte (CBOs, CIDs, idades) servem para teste e para a semente de dev; a
# verdade é a competência importada do DATASUS.
module SigtapSample
  LAYOUTS = {
    "tb_procedimento" => [ [ "CO_PROCEDIMENTO", 10, "VARCHAR2" ], [ "NO_PROCEDIMENTO", 250, "VARCHAR2" ],
                           [ "TP_COMPLEXIDADE", 1, "VARCHAR2" ], [ "TP_SEXO", 1, "VARCHAR2" ],
                           [ "QT_MAXIMA_EXECUCAO", 4, "NUMBER" ], [ "QT_DIAS_PERMANENCIA", 4, "NUMBER" ],
                           [ "QT_PONTOS", 4, "NUMBER" ], [ "VL_IDADE_MINIMA", 4, "NUMBER" ],
                           [ "VL_IDADE_MAXIMA", 4, "NUMBER" ], [ "DT_COMPETENCIA", 6, "VARCHAR2" ] ],
    "rl_procedimento_ocupacao" => [ [ "CO_PROCEDIMENTO", 10, "VARCHAR2" ], [ "CO_OCUPACAO", 6, "VARCHAR2" ],
                                    [ "DT_COMPETENCIA", 6, "VARCHAR2" ] ],
    "rl_procedimento_cid" => [ [ "CO_PROCEDIMENTO", 10, "VARCHAR2" ], [ "CO_CID", 4, "VARCHAR2" ],
                               [ "ST_PRINCIPAL", 1, "VARCHAR2" ], [ "DT_COMPETENCIA", 6, "VARCHAR2" ] ],
    "rl_procedimento_registro" => [ [ "CO_PROCEDIMENTO", 10, "VARCHAR2" ], [ "CO_REGISTRO", 2, "VARCHAR2" ],
                                    [ "DT_COMPETENCIA", 6, "VARCHAR2" ] ],
    "tb_registro" => [ [ "CO_REGISTRO", 2, "VARCHAR2" ], [ "NO_REGISTRO", 50, "VARCHAR2" ],
                       [ "DT_COMPETENCIA", 6, "VARCHAR2" ] ]
  }.freeze

  # código, nome, complexidade, sexo, idade mínima, idade máxima (meses)
  PROCEDURES = [
    [ "0301010064", "CONSULTA MEDICA EM ATENCAO PRIMARIA", "1", "I", 0, 9999 ],
    [ "0301010030", "CONSULTA DE PROFISSIONAIS DE NIVEL SUPERIOR NA ATENCAO PRIMARIA (EXCETO MEDICO)", "1", "I", 0, 9999 ],
    [ "0201020033", "COLETA DE MATERIAL P/ EXAME CITOPATOLOGICO DE COLO UTERINO", "1", "F", 120, 1560 ],
    [ "0301100039", "AFERICAO DE PRESSAO ARTERIAL", "1", "I", 0, 9999 ],
    [ "0301040079", "ESCUTA INICIAL / ORIENTACAO (ACOLHIMENTO A DEMANDA ESPONTANEA)", "1", "I", 0, 9999 ]
  ].freeze

  CBOS = {
    "0301010064" => %w[225125 225142 225130 225170],
    "0301010030" => %w[223505 251510],
    "0201020033" => %w[223505 225125],
    "0301100039" => %w[223505 322205 225125],
    "0301040079" => %w[223505 322205 225125]
  }.freeze

  CIDS = { "0201020033" => [ %w[Z014 S] ] }.freeze

  INSTRUMENTS = {
    "0301010064" => %w[02], "0301010030" => %w[02], "0201020033" => %w[01 02], "0301100039" => %w[01],
    "0301040079" => %w[01]
  }.freeze

  REGISTRIES = { "01" => "BPA (CONSOLIDADO)", "02" => "BPA (INDIVIDUALIZADO)" }.freeze

  module_function

  def write_to(dir, competence:)
    dir = Pathname(dir)
    FileUtils.mkdir_p(dir)
    LAYOUTS.each do |file, columns|
      dir.join("#{file}_layout.txt").write(layout_lines(columns).join("\r\n") + "\r\n")
      data = rows_for(file, competence).map { |values| fixed(columns, values) }
      dir.join("#{file}.txt").binwrite((data.join("\r\n") + "\r\n").encode("ISO-8859-1"))
    end
    dir
  end

  def layout_lines(columns)
    start = 1
    [ "Coluna,Tamanho,Inicio,Fim,Tipo" ] + columns.map do |name, size, type|
      line = "#{name},#{size},#{start},#{start + size - 1},#{type}"
      start += size
      line
    end
  end

  def fixed(columns, values)
    columns.zip(values).map { |(_, size, type), v| type == "NUMBER" ? v.to_s.rjust(size, "0") : v.to_s.ljust(size) }.join
  end

  def rows_for(file, competence)
    case file
    when "tb_procedimento"
      PROCEDURES.map { |code, name, cx, sex, min, max| [ code, name, cx, sex, 9999, 0, 0, min, max, competence ] }
    when "rl_procedimento_ocupacao" then CBOS.flat_map { |code, cbos| cbos.map { |cbo| [ code, cbo, competence ] } }
    when "rl_procedimento_cid" then CIDS.flat_map { |code, cids| cids.map { |cid, principal| [ code, cid, principal, competence ] } }
    when "rl_procedimento_registro" then INSTRUMENTS.flat_map { |code, regs| regs.map { |reg| [ code, reg, competence ] } }
    when "tb_registro" then REGISTRIES.map { |code, name| [ code, name, competence ] }
    end
  end
end
```

- [ ] **Step 2: Escreva a spec que falha**

```ruby
# spec/services/terminology/sigtap_reader_spec.rb
require "rails_helper"
require Rails.root.join("lib/sigtap_sample").to_s

# ADR 0028 (spec 2026-10-05 §4): a SIGTAP chega por competência, em largura
# fixa com o layout no próprio ZIP — o leitor obedece ao layout, então campo de
# tamanho mudado não quebra a importação.
RSpec.describe Terminology::SigtapReader do
  let(:tmp) { Pathname(Dir.mktmpdir) }
  after { FileUtils.remove_entry(tmp) }

  def import(version, dir = SigtapSample.write_to(tmp.join(version), competence: version))
    Terminology::Import.call(kind: "sigtap", version: version, path: dir)
  end

  it "grava procedimentos, CBOs, CIDs e instrumentos; 9999 vira sem limite" do
    result = import("202610")

    expect(result).to be_ok
    expect(result.payload[:counts]).to eq(sigtap_procedures: 5, sigtap_procedure_cbos: 14, sigtap_procedure_cids: 1,
                                          sigtap_procedure_instruments: 6)
    release = result.payload[:release]
    coleta = SigtapProcedure.find_by!(release: release, code: "0201020033")
    expect(coleta).to have_attributes(sex: "F", age_min_months: 120, age_max_months: 1560, complexity: "1",
                                      name: "COLETA DE MATERIAL P/ EXAME CITOPATOLOGICO DE COLO UTERINO")
    expect(SigtapProcedure.find_by!(release: release, code: "0301010064").age_max_months).to be_nil
    expect(SigtapProcedureCid.find_by!(release: release, procedure_code: "0201020033")).to have_attributes(cid_code: "Z014", principal: true)
    expect(SigtapProcedureInstrument.where(release: release, procedure_code: "0201020033").pluck(:instrument_name))
      .to contain_exactly("BPA (CONSOLIDADO)", "BPA (INDIVIDUALIZADO)")
  end

  it "obedece ao layout: nome com largura diferente continua lendo certo, do ZIP" do
    layouts = SigtapSample::LAYOUTS.merge(
      "tb_procedimento" => SigtapSample::LAYOUTS["tb_procedimento"].map { |c| c.first == "NO_PROCEDIMENTO" ? [ c[0], 300, c[2] ] : c }
    )
    stub_const("SigtapSample::LAYOUTS", layouts)
    dir = SigtapSample.write_to(tmp.join("larga"), competence: "202610")
    zip = tmp.join("TabelaUnificada_202610.zip")
    Zip::File.open(zip.to_s, create: true) { |z| dir.children.each { |f| z.add(f.basename.to_s, f.to_s) } }

    expect(import("202610", zip)).to be_ok
    expect(SigtapProcedure.where(code: "0301100039").pick(:name)).to eq("AFERICAO DE PRESSAO ARTERIAL")
  end

  it "republicação da mesma competência substitui; outra competência continua ativa" do
    first = import("202609").payload[:release]
    old = import("202610").payload[:release]
    again = import("202610", SigtapSample.write_to(tmp.join("rep"), competence: "202610")).payload[:release]
    expect([ first.reload.status, old.reload.status, again.status ]).to eq(%w[active superseded active])
  end

  it "procedimento com código inválido ou instrumento sem registro: failed" do
    dir = SigtapSample.write_to(tmp.join("ruim"), competence: "202610")
    text = dir.join("tb_procedimento.txt").binread
    dir.join("tb_procedimento.txt").binwrite(text.sub("0301010064", "03010100XX"))
    result = import("202610", dir)
    expect(result.reason).to eq(:invalid_file)
    expect(TerminologyRelease.find_by!(kind: "sigtap", version: "202610").status).to eq("failed")
    expect(SigtapProcedure.count).to eq(0)
  end
end
```

- [ ] **Step 3: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec spec/services/terminology/sigtap_reader_spec.rb`
Expected: FAIL (`leitor da SIGTAP ainda não implementado`).

- [ ] **Step 4: Implemente**

```ruby
# app/services/terminology/sigtap_reader.rb
# Tabela Unificada SIGTAP (ADR 0028; spec 2026-10-05 §4; desvio 10): arquivos
# de largura fixa, ISO-8859-1, cada um descrito pelo `<arquivo>_layout.txt` que
# vem no mesmo ZIP. Lê só o que o módulo usa: procedimento (sexo, idades em
# meses, complexidade), CBO, CID e instrumento de registro.
module Terminology
  class SigtapReader
    NO_LIMIT = 9999
    CODE = /\A\d{10}\z/
    CBO = /\A\d{6}\z/
    CID = /\A[A-Z]\d{2}[0-9X]?\z/
    SEXES = %w[M F I N].freeze

    def initialize(archive)
      @archive = archive
    end

    def write(release)
      registries = fixed("tb_registro").to_h { |r| [ r["CO_REGISTRO"], r["NO_REGISTRO"] ] }
      procedures = fixed("tb_procedimento").map { |r| procedure_row(release, r) }
      raise Import::Invalid, "nenhum procedimento no arquivo" if procedures.empty?

      known = procedures.to_set { |p| p[:code] }
      cbos = fixed("rl_procedimento_ocupacao").map do |r|
        { release_id: release.id, procedure_code: known_code(known, r), cbo_code: matching(r["CO_OCUPACAO"], CBO, "CBO") }
      end
      cids = fixed("rl_procedimento_cid").map do |r|
        { release_id: release.id, procedure_code: known_code(known, r), cid_code: matching(r["CO_CID"], CID, "CID"),
          principal: r["ST_PRINCIPAL"] == "S" }
      end
      instruments = fixed("rl_procedimento_registro").map do |r|
        code = r["CO_REGISTRO"]
        name = registries.fetch(code) { raise Import::Invalid, "instrumento de registro desconhecido: #{code.inspect}" }
        { release_id: release.id, procedure_code: known_code(known, r), instrument_code: code, instrument_name: name }
      end

      {
        sigtap_procedures: Import.insert(SigtapProcedure, procedures),
        sigtap_procedure_cbos: Import.insert(SigtapProcedureCbo, cbos),
        sigtap_procedure_cids: Import.insert(SigtapProcedureCid, cids),
        sigtap_procedure_instruments: Import.insert(SigtapProcedureInstrument, instruments)
      }
    end

    private

    def fixed(base)
      columns = []
      @archive.each_row(/\A#{base}_layout\.txt\z/i, col_sep: ",") do |r|
        columns << [ r["COLUNA"], Integer(r["INICIO"], 10), Integer(r["TAMANHO"], 10) ]
      end
      raise Import::Invalid, "layout vazio: #{base}" if columns.empty?

      rows = []
      @archive.each_line(/\A#{base}\.txt\z/i) do |line|
        rows << columns.to_h { |name, start, size| [ name, line[start - 1, size].to_s.strip ] }
      end
      rows
    rescue ArgumentError, TypeError
      raise Import::Invalid, "layout ilegível: #{base}"
    end

    def procedure_row(release, r)
      code = matching(r["CO_PROCEDIMENTO"], CODE, "procedimento")
      raise Import::Invalid, "procedimento #{code} sem nome" if r["NO_PROCEDIMENTO"].blank?

      sex = r["TP_SEXO"].presence
      raise Import::Invalid, "sexo inválido em #{code}: #{sex}" unless sex.nil? || SEXES.include?(sex)

      { release_id: release.id, code: code, name: r["NO_PROCEDIMENTO"], sex: sex,
        age_min_months: age(r["VL_IDADE_MINIMA"], code), age_max_months: age(r["VL_IDADE_MAXIMA"], code),
        complexity: r["TP_COMPLEXIDADE"].presence }
    end

    def age(value, code)
      months = Integer(value.to_s, 10)
      months == NO_LIMIT ? nil : months
    rescue ArgumentError
      raise Import::Invalid, "idade inválida em #{code}: #{value.inspect}"
    end

    def known_code(known, row)
      code = row["CO_PROCEDIMENTO"]
      raise Import::Invalid, "relação aponta para procedimento ausente: #{code.inspect}" unless known.include?(code)

      code
    end

    def matching(value, pattern, label)
      raise Import::Invalid, "#{label} inválido: #{value.inspect}" unless value.to_s.match?(pattern)

      value
    end
  end
end
```

- [ ] **Step 5: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec spec/services/terminology`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod16 add lib/sigtap_sample.rb app/services/terminology/sigtap_reader.rb spec/services/terminology/sigtap_reader_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod16 commit -m "feat: import SIGTAP competences from the official fixed-width layout

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 13: `Terminology::Sigtap.compatible?` e o alerta de SIGTAP não importada

**Files:**
- Create: `app/services/terminology/sigtap.rb`, `app/services/terminology/sigtap_status.rb`
- Test: `spec/services/terminology/sigtap_spec.rb`

**Interfaces:**
- Consumes: modelos e importação das Tasks 10–12; `SigtapSample`.
- Produces: `Terminology::Sigtap.release_for(competence) -> TerminologyRelease|nil` (a ativa da competência, ou a última ativa anterior; `nil` → a mais recente); `Terminology::Sigtap.compatible?(code, competence:, cbo: nil, age_months: nil, sex: nil, cid: nil) -> { ok: Boolean, reasons: Array<String>, release_version: String|nil }` (motivos: `no_release`, `unknown_procedure`, `sex_incompatible`, `age_below_minimum`, `age_above_maximum`, `cbo_incompatible`, `cid_incompatible`; `sex` aceita `"female"`/`"male"`/`"F"`/`"M"`; argumento `nil` não é conferido); `Terminology::SigtapStatus.call(today: Time.zone.today) -> { sigtap_current_competence:, sigtap_imported:, alert: }` (o plano do exportador publica as duas primeiras chaves em `GET /city_production`).

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/services/terminology/sigtap_spec.rb
require "rails_helper"
require Rails.root.join("lib/sigtap_sample").to_s

# ADR 0028 (spec 2026-10-05 §4): compatibilidade de procedimento com CBO,
# idade, sexo e CID na competência; competência sem release cai na última
# ativa anterior. Alerta no dia 5 sem a SIGTAP da competência corrente.
RSpec.describe Terminology::Sigtap do
  let(:tmp) { Pathname(Dir.mktmpdir) }
  after { FileUtils.remove_entry(tmp) }

  def import(version)
    Terminology::Import.call(kind: "sigtap", version: version, path: SigtapSample.write_to(tmp.join(version), competence: version))
  end

  before { import("202610") }

  def check(**overrides)
    described_class.compatible?("0201020033", **{ competence: "202610", cbo: "223505", age_months: 360, sex: "female",
                                                  cid: "Z01.4" }.merge(overrides))
  end

  it "tudo compatível" do
    expect(check).to eq(ok: true, reasons: [], release_version: "202610")
  end

  it "cada regra quebrada dá o seu motivo" do
    expect(check(sex: "male")[:reasons]).to eq(%w[sex_incompatible])
    expect(check(age_months: 100)[:reasons]).to eq(%w[age_below_minimum])
    expect(check(age_months: 1600)[:reasons]).to eq(%w[age_above_maximum])
    expect(check(cbo: "322205")[:reasons]).to eq(%w[cbo_incompatible])
    expect(check(cid: "I10")[:reasons]).to eq(%w[cid_incompatible])
    expect(check(sex: "M", cbo: "322205")).to include(ok: false, reasons: %w[sex_incompatible cbo_incompatible])
  end

  it "sem limite de idade, sem sexo e sem CID exigido não recusam; argumento nil não é conferido" do
    expect(described_class.compatible?("0301010064", competence: "202610", cbo: "225125", age_months: 1700,
                                                     sex: "male", cid: "I10")[:ok]).to be(true)
    expect(check(cbo: nil, age_months: nil, sex: nil, cid: nil)[:ok]).to be(true)
  end

  it "competência sem release cai na anterior; antes da primeira não há release; código desconhecido" do
    expect(check(competence: "202612")[:release_version]).to eq("202610")
    expect(check(competence: "202608")).to eq(ok: false, reasons: %w[no_release], release_version: nil)
    expect(described_class.compatible?("0000000000", competence: "202610")[:reasons]).to eq(%w[unknown_procedure])
    expect { check(competence: "2026-10") }.to raise_error(ArgumentError)
  end

  describe Terminology::SigtapStatus do
    it "alerta a partir do dia 5 só quando a competência corrente não foi importada" do
      expect(described_class.call(today: Date.new(2026, 10, 9)))
        .to eq(sigtap_current_competence: "202610", sigtap_imported: true, alert: false)
      expect(described_class.call(today: Date.new(2026, 11, 4))).to include(sigtap_imported: false, alert: false)
      expect(described_class.call(today: Date.new(2026, 11, 5))).to include(sigtap_current_competence: "202611", alert: true)
    end
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec spec/services/terminology/sigtap_spec.rb`
Expected: FAIL (`uninitialized constant Terminology::Sigtap`).

- [ ] **Step 3: Implemente**

```ruby
# app/services/terminology/sigtap.rb
# Compatibilidade de procedimento SIGTAP (ADR 0028; spec 2026-10-05 §4). Lê a
# release ativa da competência — ou a última ativa anterior a ela. Relação
# vazia (procedimento sem CBO/CID listado) não restringe. Os módulos que
# registram procedimento (18, 19…) chamam isto antes de gerar a ficha.
module Terminology
  module Sigtap
    COMPETENCE = /\A\d{4}(0[1-9]|1[0-2])\z/
    SEX = { "female" => "F", "male" => "M", "F" => "F", "M" => "M" }.freeze

    module_function

    def release_for(competence)
      scope = TerminologyRelease.active.where(kind: "sigtap")
      scope = scope.where("version <= ?", competence) if competence
      scope.order(version: :desc).first
    end

    def compatible?(code, competence:, cbo: nil, age_months: nil, sex: nil, cid: nil)
      raise ArgumentError, "competência inválida: #{competence.inspect}" unless competence.nil? || competence.to_s.match?(COMPETENCE)

      release = release_for(competence&.to_s)
      return { ok: false, reasons: %w[no_release], release_version: nil } unless release

      procedure = SigtapProcedure.find_by(release_id: release.id, code: code.to_s)
      return { ok: false, reasons: %w[unknown_procedure], release_version: release.version } unless procedure

      reasons = []
      reasons << "sex_incompatible" if sex_incompatible?(procedure.sex, sex)
      if age_months
        reasons << "age_below_minimum" if procedure.age_min_months && age_months < procedure.age_min_months
        reasons << "age_above_maximum" if procedure.age_max_months && age_months > procedure.age_max_months
      end
      reasons << "cbo_incompatible" if cbo && !listed?(SigtapProcedureCbo, release, code, cbo_code: cbo.to_s)
      if cid && !listed?(SigtapProcedureCid, release, code, cid_code: cid.to_s.delete(".").upcase)
        reasons << "cid_incompatible"
      end
      { ok: reasons.empty?, reasons: reasons, release_version: release.version }
    end

    def sex_incompatible?(required, sex)
      return false if sex.nil? || !%w[F M].include?(required)

      SEX[sex.to_s] != required
    end

    # Sem relação nenhuma para o procedimento = sem restrição.
    def listed?(model, release, code, **value)
      scope = model.where(release_id: release.id, procedure_code: code.to_s)
      !scope.exists? || scope.exists?(value)
    end
  end
end
```

```ruby
# app/services/terminology/sigtap_status.rb
# Alerta do console (ADR 0028; spec 2026-10-05 §4): a partir do dia 5, sem a
# SIGTAP da competência corrente ativa. `GET /city_production` (plano do
# exportador) publica sigtap_current_competence e sigtap_imported.
module Terminology
  module SigtapStatus
    ALERT_DAY = 5

    def self.call(today: Time.zone.today)
      competence = today.strftime("%Y%m")
      imported = TerminologyRelease.active.exists?(kind: "sigtap", version: competence)
      { sigtap_current_competence: competence, sigtap_imported: imported, alert: !imported && today.day >= ALERT_DAY }
    end
  end
end
```

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec spec/services/terminology`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod16 add app/services/terminology/sigtap.rb app/services/terminology/sigtap_status.rb spec/services/terminology/sigtap_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod16 commit -m "feat: check SIGTAP procedure compatibility and flag a missing current competence

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Fatia 4 — CNES (F-16.5)

### Task 14: Retratos do CNES na plataforma e o gravador com retenção

**Files:**
- Create: `db/platform_migrate/20261005200003_create_cnes_snapshots.rb`, `app/models/cnes_snapshot.rb`, `app/models/cnes_establishment.rb`, `app/models/cnes_team.rb`, `app/models/cnes_professional_bond.rb`, `app/services/cnes/snapshot_writer.rb`
- Modify: `db/platform_schema.rb` (gerado)
- Test: `spec/services/cnes/snapshot_writer_spec.rb`

**Interfaces:**
- Produces: `CnesSnapshot` (`competence`, `ibge_code`, `imported_at`; `RETENTION = 13`; `has_many :establishments, :teams, :bonds`); `CnesEstablishment` (`snapshot_id`, `cnes`, `name`, `unit_type`); `CnesTeam` (`snapshot_id`, `ine`, `kind`, `cnes`, `name`, `active`); `CnesProfessionalBond` (`snapshot_id`, `cnes`, `ine`, `cbo_code`, `cpf`, `cns` — os dois cifrados com a chave da plataforma; `#cpf_masked`, `#cns_masked`); `Cnes::SnapshotWriter.write!(competence:, ibge_code:, establishments:, teams:, bonds:) -> CnesSnapshot` (Hashes com chaves símbolo: estabelecimento `{ cnes:, name:, unit_type: }`, equipe `{ ine:, kind:, cnes:, name:, active: }`, vínculo `{ cnes:, ine:, cbo_code:, cpf:, cns: }`).

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/services/cnes/snapshot_writer_spec.rb
require "rails_helper"

# ADR 0028 (spec 2026-10-05 §5, §8): um retrato por município e competência
# (reimportar substitui), últimas 13 competências por município, CPF/CNS do
# profissional cifrados na plataforma.
RSpec.describe Cnes::SnapshotWriter do
  def write(competence, ibge_code: "4106902", bonds: [])
    described_class.write!(competence: competence, ibge_code: ibge_code,
                           establishments: [ { cnes: "0000001", name: "UBS JARDIM DAS FLORES", unit_type: "02" } ],
                           teams: [ { ine: "0000123456", kind: "70", cnes: "0000001", name: "ESF 1", active: true } ],
                           bonds: bonds)
  end

  let(:bond) { { cnes: "0000001", ine: "0000123456", cbo_code: "225125", cpf: "52998224725", cns: "700000000000005" } }

  it "grava o retrato e cifra CPF e CNS" do
    snapshot = write("202609", bonds: [ bond ])
    expect(snapshot.establishments.pluck(:name)).to eq([ "UBS JARDIM DAS FLORES" ])
    expect(snapshot.teams.pluck(:ine, :active)).to eq([ [ "0000123456", true ] ])
    stored = snapshot.bonds.first
    expect([ stored.cpf, stored.cns ]).to eq(%w[52998224725 700000000000005])
    expect(stored.cpf_masked).to eq("***.982.247-**")
    raw = PlatformRecord.connection.select_one("SELECT cpf, cns FROM cnes_professional_bonds WHERE id = #{stored.id}")
    expect(raw.values.join).not_to include("52998224725")
    expect(raw.values.join).not_to include("700000000000005")
  end

  # Review Focus 3 (metade do gravador).
  it "reimportar a mesma competência substitui; a 14ª competência apaga a mais antiga, com os filhos" do
    write("202609", bonds: [ bond ])
    write("202609", bonds: [ bond, bond.merge(cbo_code: "223505") ])
    expect(CnesSnapshot.where(ibge_code: "4106902").count).to eq(1)
    expect(CnesProfessionalBond.count).to eq(2)

    competences = (0..13).map { |i| (Date.new(2025, 9, 1) >> i).strftime("%Y%m") }
    competences.each { |c| write(c) }
    write("202609", ibge_code: "4115200")
    expect(CnesSnapshot.where(ibge_code: "4106902").order(:competence).pluck(:competence)).to eq(competences.last(13))
    expect(CnesSnapshot.where(ibge_code: "4115200").count).to eq(1)
    expect(CnesEstablishment.where.not(snapshot_id: CnesSnapshot.select(:id))).to be_empty
  end

  it "competência e IBGE fora do formato são recusados pelo banco" do
    expect { write("2026-09") }.to raise_error(ActiveRecord::StatementInvalid)
    expect { write("202609", ibge_code: "410690") }.to raise_error(ActiveRecord::StatementInvalid)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec spec/services/cnes/snapshot_writer_spec.rb`
Expected: FAIL (`uninitialized constant Cnes`).

- [ ] **Step 3: Migração e modelos**

```ruby
# db/platform_migrate/20261005200003_create_cnes_snapshots.rb
# Retratos mensais do CNES (ADR 0028; spec 2026-10-05 §5, §8): só municípios
# de cidades ativas, últimas 13 competências por município (retenção no
# gravador). CPF/CNS do profissional cifrados com a chave da plataforma (desvio
# 12); nenhum nome de profissional. Os filhos saem com o retrato (cascade).
class CreateCnesSnapshots < ActiveRecord::Migration[8.1]
  def change
    create_table :cnes_snapshots, id: :uuid, default: -> { "gen_random_uuid()" } do |t|
      t.string :competence, null: false, limit: 6
      t.string :ibge_code, null: false, limit: 7
      t.datetime :imported_at, null: false
      t.timestamps
      t.index %i[ibge_code competence], unique: true, name: "idx_cnes_snapshots_municipality_competence"
      t.check_constraint "competence::text ~ '^[0-9]{4}(0[1-9]|1[0-2])$'::text", name: "ck_cnes_snapshots_competence"
      t.check_constraint "ibge_code::text ~ '^[0-9]{7}$'::text", name: "ck_cnes_snapshots_ibge_code"
    end

    create_table :cnes_establishments do |t|
      t.uuid :snapshot_id, null: false
      t.string :cnes, null: false, limit: 7
      t.string :name, null: false
      t.string :unit_type, limit: 4
      t.index %i[snapshot_id cnes], unique: true
    end

    create_table :cnes_teams do |t|
      t.uuid :snapshot_id, null: false
      t.string :ine, null: false, limit: 10
      t.string :kind, null: false, limit: 4
      t.string :cnes, null: false, limit: 7
      t.string :name
      t.boolean :active, null: false
      t.index %i[snapshot_id ine], unique: true
    end

    create_table :cnes_professional_bonds do |t|
      t.uuid :snapshot_id, null: false
      t.string :cnes, null: false, limit: 7
      t.string :ine, limit: 10
      t.string :cbo_code, null: false, limit: 6
      t.text :cpf
      t.text :cns
      t.index :snapshot_id
    end

    %i[cnes_establishments cnes_teams cnes_professional_bonds].each do |table|
      add_foreign_key table, :cnes_snapshots, column: :snapshot_id, on_delete: :cascade
    end
  end
end
```

```ruby
# app/models/cnes_snapshot.rb
# Retrato do CNES de um município numa competência (ADR 0028; spec 2026-10-05
# §5). Escrito só por Cnes::SnapshotWriter.
class CnesSnapshot < PlatformRecord
  RETENTION = 13

  has_many :establishments, class_name: "CnesEstablishment", foreign_key: :snapshot_id, inverse_of: false
  has_many :teams, class_name: "CnesTeam", foreign_key: :snapshot_id, inverse_of: false
  has_many :bonds, class_name: "CnesProfessionalBond", foreign_key: :snapshot_id, inverse_of: false
end
```

```ruby
# app/models/cnes_establishment.rb
class CnesEstablishment < PlatformRecord
  belongs_to :snapshot, class_name: "CnesSnapshot"
end
```

```ruby
# app/models/cnes_team.rb
class CnesTeam < PlatformRecord
  belongs_to :snapshot, class_name: "CnesSnapshot"
end
```

```ruby
# app/models/cnes_professional_bond.rb
# Vínculo de profissional no retrato do CNES (ADR 0028). CPF e CNS cifrados com
# a chave da PLATAFORMA (key_provider fixo, como City#database_url) — dado
# pessoal de profissional, não de cidadão; sai mascarado nas telas.
class CnesProfessionalBond < PlatformRecord
  belongs_to :snapshot, class_name: "CnesSnapshot"

  encrypts :cpf, key_provider: PlatformKeyProvider.new
  encrypts :cns, key_provider: PlatformKeyProvider.new

  def cpf_masked = cpf && CitizenIdentity::Cpf.mask(cpf)
  def cns_masked = Professionals::Cns.mask(cns)
end
```

- [ ] **Step 4: Gravador**

```ruby
# app/services/cnes/snapshot_writer.rb
# Grava o retrato de um município numa competência (ADR 0028; spec 2026-10-05
# §5): substitui o da mesma competência, insere em lote e poda além das 13
# competências mais recentes. Numa transação (savepoint) da plataforma. O
# insert_all não passa pelo modelo: CPF e CNS são cifrados aqui pelo próprio
# tipo do atributo, com a chave da plataforma.
module Cnes
  module SnapshotWriter
    BATCH = 1_000

    module_function

    def write!(competence:, ibge_code:, establishments:, teams:, bonds:)
      PlatformRecord.transaction(requires_new: true) do
        CnesSnapshot.where(ibge_code: ibge_code, competence: competence).delete_all
        snapshot = CnesSnapshot.create!(ibge_code: ibge_code, competence: competence, imported_at: Time.current)
        insert(CnesEstablishment, establishments.map { |e| e.slice(:cnes, :name, :unit_type).merge(snapshot_id: snapshot.id) })
        insert(CnesTeam, teams.map { |t| t.slice(:ine, :kind, :cnes, :name, :active).merge(snapshot_id: snapshot.id) })
        insert(CnesProfessionalBond, bonds.map do |b|
          { snapshot_id: snapshot.id, cnes: b[:cnes], ine: b[:ine], cbo_code: b[:cbo_code],
            cpf: cipher(:cpf, b[:cpf]), cns: cipher(:cns, b[:cns]) }
        end)
        prune!(ibge_code)
        snapshot
      end
    end

    def insert(model, rows)
      rows.each_slice(BATCH) { |slice| model.insert_all!(slice) }
    end

    def cipher(attribute, value)
      value.nil? ? nil : CnesProfessionalBond.type_for_attribute(attribute).serialize(value)
    end

    def prune!(ibge_code)
      stale = CnesSnapshot.where(ibge_code: ibge_code).order(competence: :desc).offset(CnesSnapshot::RETENTION).pluck(:id)
      CnesSnapshot.where(id: stale).delete_all
    end
  end
end
```

- [ ] **Step 5: Migre a plataforma (dev e test)**

```bash
docker compose exec -T -w /rails/.claude/mod16 api bin/rails db:migrate
docker compose exec -T -w /rails/.claude/mod16 -e RAILS_ENV=test -e POSTGRES_PASSWORD=postgres api bin/rails db:migrate
/opt/homebrew/bin/git -C apps/api/.claude/mod16 diff --stat db/platform_schema.rb
```
Expected: `define(version: 2026_10_05_200003)`, as quatro tabelas e três `add_foreign_key ... on_delete: :cascade`.

- [ ] **Step 6: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec spec/services/cnes/snapshot_writer_spec.rb`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod16 add db/platform_migrate/20261005200003_create_cnes_snapshots.rb db/platform_schema.rb app/models/cnes_snapshot.rb app/models/cnes_establishment.rb app/models/cnes_team.rb app/models/cnes_professional_bond.rb app/services/cnes/snapshot_writer.rb spec/services/cnes/snapshot_writer_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod16 commit -m "feat: store monthly CNES snapshots per municipality with 13-competence retention

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 15: `cnes:import` — leitura da base mensal, filtrada pelas cidades ativas

**Files:**
- Create: `app/services/cnes/base_layout.rb`, `app/services/cnes/base_reader.rb`, `app/services/cnes/import.rb`, `lib/tasks/cnes.rake`, `spec/fixtures/cnes/202609/tbEstabelecimento202609.csv`, `spec/fixtures/cnes/202609/tbEquipe202609.csv`, `spec/fixtures/cnes/202609/rlEstabEquipeProf202609.csv`, `spec/fixtures/cnes/202609/tbCargaHorariaSus202609.csv`, `spec/fixtures/cnes/202609/tbDadosProfissionalSus202609.csv`
- Test: `spec/services/cnes/import_spec.rb`, `spec/tasks/cnes_rake_spec.rb`

**Interfaces:**
- Consumes: `OfficialArchive` (Task 11), `Cnes::SnapshotWriter` (Task 14), `CityProfile#ibge_code`.
- Produces: `Cnes::BaseLayout` (arquivos e colunas, desvio 10); `Cnes::BaseReader.read(archive, municipalities:) -> Hash{ibge_code => { establishments:, teams:, bonds: }}` (`municipalities`: Hash código de 6 dígitos → IBGE de 7); `Cnes::Import.call(competence:, path:) -> Result` (`ok(imported: Hash{ibge => { establishments:, teams:, bonds: }}, skipped: [{ slug:, reason: "no_ibge_code"|"city_unreachable"|"not_in_file" }])`; `fail` com `:invalid_competence`, `:file_not_found`, `:no_city`); `Cnes::Import.municipalities -> [Hash, skipped]`.

- [ ] **Step 1: Fixtures (layout presumido da base mensal, ASCII)**

`spec/fixtures/cnes/202609/tbEstabelecimento202609.csv`:

```
"CO_UNIDADE";"CO_CNES";"NU_CNPJ_MANTENEDORA";"NO_FANTASIA";"TP_UNIDADE";"CO_MUNICIPIO_GESTOR"
"4106902000001";"0000001";"";"UBS JARDIM DAS FLORES";"02";"410690"
"4106902000002";"0000002";"";"UBS VILA ESPERANCA";"02";"410690"
"4106902000003";"0000003";"";"UPA 24H CENTRO";"73";"410690"
"4106902000004";"0000004";"";"HOSPITAL PARTICULAR";"05";"410690"
"3550308000001";"9999999";"";"UBS DE OUTRA CIDADE";"02";"355030"
```

`spec/fixtures/cnes/202609/tbEquipe202609.csv`:

```
"CO_MUNICIPIO";"CO_AREA";"SEQ_EQUIPE";"TP_EQUIPE";"CO_UNIDADE";"CO_EQUIPE";"NO_REFERENCIA";"DT_ATIVACAO";"DT_DESATIVACAO"
"410690";"0001";"1";"70";"4106902000001";"123456";"ESF JARDIM 1";"01/01/2020";""
"410690";"0002";"2";"76";"4106902000002";"0000123457";"EAP VILA";"01/01/2021";""
"410690";"0003";"3";"70";"4106902000001";"0000123458";"ESF JARDIM 2";"01/01/2019";"01/06/2026"
"355030";"0001";"1";"70";"3550308000001";"0000999999";"ESF OUTRA";"01/01/2020";""
```

`spec/fixtures/cnes/202609/rlEstabEquipeProf202609.csv`:

```
"CO_MUNICIPIO";"CO_AREA";"SEQ_EQUIPE";"CO_UNIDADE";"CO_PROFISSIONAL_SUS";"CO_CBO";"DT_ENTRADA";"DT_DESLIGAMENTO"
"410690";"0001";"1";"4106902000001";"P001";"225125";"01/01/2020";""
"410690";"0001";"1";"4106902000001";"P002";"223505";"01/01/2020";""
"410690";"0002";"2";"4106902000002";"P003";"223505";"01/01/2021";"01/03/2026"
"355030";"0001";"1";"3550308000001";"P999";"225125";"01/01/2020";""
```

`spec/fixtures/cnes/202609/tbCargaHorariaSus202609.csv`:

```
"CO_UNIDADE";"CO_PROFISSIONAL_SUS";"CO_CBO";"TP_SUS_NAO_SUS"
"4106902000001";"P001";"225125";"S"
"4106902000003";"P001";"225124";"S"
"4106902000003";"P004";"322205";"S"
"3550308000001";"P999";"225125";"S"
```

`spec/fixtures/cnes/202609/tbDadosProfissionalSus202609.csv`:

```
"CO_PROFISSIONAL_SUS";"NO_PROFISSIONAL";"CO_CPF";"CO_CNS"
"P001";"NOME QUE NAO SAI";"52998224725";"700000000000005"
"P002";"NOME QUE NAO SAI";"11144477735";""
"P003";"NOME QUE NAO SAI";"39053344705";""
"P004";"NOME QUE NAO SAI";"12345678909";"123"
"P999";"NOME QUE NAO SAI";"98765432100";""
```

- [ ] **Step 2: Escreva as specs que falham**

```ruby
# spec/services/cnes/import_spec.rb
require "rails_helper"

# ADR 0028 (spec 2026-10-05 §5, §8): a base mensal grava só os municípios das
# cidades ativas (IBGE lido do city_profile de cada uma); cidade sem IBGE ou
# fora do ar é pulada sem derrubar as outras; reimportar substitui.
RSpec.describe Cnes::Import do
  let(:path) { Rails.root.join("spec/fixtures/cnes/202609") }
  let!(:city) do
    City.find_by(slug: TEST_CITY_A.slug) ||
      City.create!(slug: TEST_CITY_A.slug, name: TEST_CITY_A.name, status: "active",
                   database_url: TEST_CITY_A.database_url, encryption_key: TEST_CITY_A.encryption_key,
                   schema_version: CitySchema.expected_version.to_s)
  end
  before { CityProfile.create!(name: "Curitiba", uf: "PR", ibge_code: "4106902") }
  after { CityCatalog.reset_cache! }

  it "grava o retrato de Curitiba: estabelecimentos, equipes (INE com zeros), vínculos ativos; nada de outro município" do
    result = described_class.call(competence: "202609", path: path)

    expect(result).to be_ok
    expect(result.payload[:imported]).to eq("4106902" => { establishments: 4, teams: 3, bonds: 5 })
    snapshot = CnesSnapshot.find_by!(ibge_code: "4106902", competence: "202609")
    expect(snapshot.teams.order(:ine).pluck(:ine, :kind, :cnes, :active))
      .to eq([ %w[0000123456 70 0000001] + [ true ], %w[0000123457 76 0000002] + [ true ], %w[0000123458 70 0000001] + [ false ] ])
    bonds = snapshot.bonds.map { |b| [ b.cnes, b.ine, b.cbo_code, b.cpf, b.cns ] }
    expect(bonds).to contain_exactly(
      [ "0000001", "0000123456", "225125", "52998224725", "700000000000005" ],
      [ "0000001", "0000123456", "223505", "11144477735", nil ],
      [ "0000001", nil, "225125", "52998224725", "700000000000005" ],
      [ "0000003", nil, "225124", "52998224725", "700000000000005" ],
      [ "0000003", nil, "322205", "12345678909", nil ]
    )
    expect(CnesEstablishment.where(cnes: "9999999")).to be_empty
    expect(PlatformEvent.where(name: "cnes.snapshot_imported").map(&:payload))
      .to eq([ { "competence" => "202609", "ibge_codes_count" => 1 } ])
  end

  # Review Focus 3.
  it "reimportar substitui; cidade sem IBGE e cidade fora do ar são puladas" do
    create(:city, slug: "sem-ibge")
    create(:city, slug: "fora-do-ar", database_url: "postgres://rota_city:x@127.0.0.1:1/nenhum")
    described_class.call(competence: "202609", path: path)
    result = described_class.call(competence: "202609", path: path)

    expect(CnesSnapshot.where(ibge_code: "4106902").count).to eq(1)
    expect(result.payload[:skipped]).to include({ slug: "sem-ibge", reason: "no_ibge_code" },
                                                { slug: "fora-do-ar", reason: "city_unreachable" })
  end

  it "competência inválida, caminho ausente e nenhuma cidade com IBGE" do
    expect(described_class.call(competence: "2026-09", path: path).reason).to eq(:invalid_competence)
    expect(described_class.call(competence: "202609", path: "/nao/existe").reason).to eq(:file_not_found)
    CityProfile.current.update!(ibge_code: nil)
    expect(described_class.call(competence: "202609", path: path).reason).to eq(:no_city)
  end

  it "município sem linha no arquivo vira not_in_file" do
    CityProfile.current.update!(ibge_code: "4115200")
    result = described_class.call(competence: "202609", path: path)
    expect(result.payload[:imported]).to eq({})
    expect(result.payload[:skipped]).to include(slug: city.slug, reason: "not_in_file")
  end
end
```

```ruby
# spec/tasks/cnes_rake_spec.rb
require "rails_helper"
require "rake"

RSpec.describe "cnes:import rake task" do
  before(:all) { Rails.application.load_tasks unless Rake::Task.task_defined?("cnes:import") }
  before { Rake::Task["cnes:import"].reenable }

  it "relata por município sem CPF nem CNS na saída" do
    City.find_by(slug: TEST_CITY_A.slug) ||
      City.create!(slug: TEST_CITY_A.slug, name: TEST_CITY_A.name, status: "active",
                   database_url: TEST_CITY_A.database_url, encryption_key: TEST_CITY_A.encryption_key,
                   schema_version: CitySchema.expected_version.to_s)
    CityProfile.create!(name: "Curitiba", uf: "PR", ibge_code: "4106902")
    out = StringIO.new
    original, $stdout = $stdout, out
    Rake::Task["cnes:import"].invoke("202609", Rails.root.join("spec/fixtures/cnes/202609").to_s)
    $stdout = original
    expect(out.string).to include("4106902: 4 estabelecimentos, 3 equipes, 5 vínculos")
    expect(out.string).not_to include("52998224725")
  ensure
    $stdout = original if original
    CityCatalog.reset_cache!
  end
end
```

- [ ] **Step 3: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec spec/services/cnes/import_spec.rb spec/tasks/cnes_rake_spec.rb`
Expected: FAIL (`uninitialized constant Cnes::Import`).

- [ ] **Step 4: Implemente**

```ruby
# app/services/cnes/base_layout.rb
# Layout da base mensal do CNES (BASE_DE_DADOS_CNES_AAAAMM.ZIP; pesquisa frente
# 2; desvio 10): CSV com ";" e ISO-8859-1, lido por NOME de coluna. Se o
# dicionário oficial nomear diferente, muda só aqui. O município no CNES tem
# 6 dígitos (IBGE sem o verificador).
module Cnes
  module BaseLayout
    ENCODING = "ISO-8859-1".freeze
    FILES = {
      establishments: /\AtbEstabelecimento\d{6}\.csv\z/i,
      teams: /\AtbEquipe\d{6}\.csv\z/i,
      team_bonds: /\ArlEstabEquipeProf\d{6}\.csv\z/i,
      unit_bonds: /\AtbCargaHorariaSus\d{6}\.csv\z/i,
      professionals: /\AtbDadosProfissionalSus\d{6}\.csv\z/i
    }.freeze
    ESTABLISHMENT = { unit_id: "CO_UNIDADE", cnes: "CO_CNES", name: "NO_FANTASIA", unit_type: "TP_UNIDADE",
                      municipality: "CO_MUNICIPIO_GESTOR" }.freeze
    TEAM = { municipality: "CO_MUNICIPIO", area: "CO_AREA", seq: "SEQ_EQUIPE", kind: "TP_EQUIPE", unit_id: "CO_UNIDADE",
             ine: "CO_EQUIPE", name: "NO_REFERENCIA", deactivated_on: "DT_DESATIVACAO" }.freeze
    TEAM_BOND = { municipality: "CO_MUNICIPIO", area: "CO_AREA", seq: "SEQ_EQUIPE", unit_id: "CO_UNIDADE",
                  professional_id: "CO_PROFISSIONAL_SUS", cbo: "CO_CBO", left_on: "DT_DESLIGAMENTO" }.freeze
    UNIT_BOND = { unit_id: "CO_UNIDADE", professional_id: "CO_PROFISSIONAL_SUS", cbo: "CO_CBO" }.freeze
    PROFESSIONAL = { professional_id: "CO_PROFISSIONAL_SUS", cpf: "CO_CPF", cns: "CO_CNS" }.freeze
  end
end
```

```ruby
# app/services/cnes/base_reader.rb
# Lê a base mensal em fluxo e devolve, por município de interesse, o retrato
# pronto para Cnes::SnapshotWriter (ADR 0028; spec 2026-10-05 §5). Cinco
# passadas, cada uma filtrando pelo que a anterior achou; só os profissionais
# com vínculo num estabelecimento de interesse têm CPF/CNS lidos. Vínculo
# desligado não entra; equipe desativada entra com active: false.
module Cnes
  module BaseReader
    module_function

    def read(archive, municipalities:)
      layout = BaseLayout
      units = {}
      rows(archive, :establishments) do |r|
        ibge = municipalities[r[layout::ESTABLISHMENT[:municipality]].to_s[0, 6]]
        cnes = r[layout::ESTABLISHMENT[:cnes]].to_s.rjust(7, "0")
        next unless ibge && cnes.match?(/\A\d{7}\z/)

        units[r[layout::ESTABLISHMENT[:unit_id]]] = { ibge: ibge, cnes: cnes, name: r[layout::ESTABLISHMENT[:name]].to_s,
                                                      unit_type: r[layout::ESTABLISHMENT[:unit_type]] }
      end

      teams = []
      team_keys = {}
      rows(archive, :teams) do |r|
        unit = units[r[layout::TEAM[:unit_id]]] or next
        ine = r[layout::TEAM[:ine]].to_s.rjust(10, "0")
        next unless ine.match?(/\A\d{10}\z/)

        team_keys[r.values_at(*layout::TEAM.values_at(:municipality, :area, :seq))] = ine
        teams << { ibge: unit[:ibge], ine: ine, kind: r[layout::TEAM[:kind]].to_s, cnes: unit[:cnes],
                   name: r[layout::TEAM[:name]], active: r[layout::TEAM[:deactivated_on]].blank? }
      end

      bonds = []
      rows(archive, :team_bonds) do |r|
        unit = units[r[layout::TEAM_BOND[:unit_id]]] or next
        next if r[layout::TEAM_BOND[:left_on]].present?

        ine = team_keys[r.values_at(*layout::TEAM_BOND.values_at(:municipality, :area, :seq))] or next
        bonds << { ibge: unit[:ibge], cnes: unit[:cnes], ine: ine, cbo_code: r[layout::TEAM_BOND[:cbo]],
                   professional_id: r[layout::TEAM_BOND[:professional_id]] }
      end
      rows(archive, :unit_bonds) do |r|
        unit = units[r[layout::UNIT_BOND[:unit_id]]] or next
        bonds << { ibge: unit[:ibge], cnes: unit[:cnes], ine: nil, cbo_code: r[layout::UNIT_BOND[:cbo]],
                   professional_id: r[layout::UNIT_BOND[:professional_id]] }
      end
      bonds = bonds.select { |b| b[:cbo_code].to_s.match?(/\A\d{6}\z/) }.uniq

      wanted = bonds.to_set { |b| b[:professional_id] }
      people = {}
      rows(archive, :professionals) do |r|
        id = r[layout::PROFESSIONAL[:professional_id]]
        next unless wanted.include?(id)

        cns = r[layout::PROFESSIONAL[:cns]].to_s
        people[id] = { cpf: CitizenIdentity::Cpf.normalize(r[layout::PROFESSIONAL[:cpf]]),
                       cns: Professionals::Cns.valid?(cns) ? cns : nil }
      end

      municipalities.values.uniq.to_h do |ibge|
        establishments = units.values.select { |u| u[:ibge] == ibge }.map { |u| u.slice(:cnes, :name, :unit_type) }
        [ ibge, {
          establishments: establishments,
          teams: teams.select { |t| t[:ibge] == ibge }.map { |t| t.except(:ibge) },
          bonds: bonds.select { |b| b[:ibge] == ibge }.map do |b|
            person = people.fetch(b[:professional_id], {})
            { cnes: b[:cnes], ine: b[:ine], cbo_code: b[:cbo_code], cpf: person[:cpf], cns: person[:cns] }
          end
        } ]
      end
    end

    def rows(archive, file, &block)
      archive.each_row(BaseLayout::FILES.fetch(file), encoding: BaseLayout::ENCODING, &block)
    end
  end
end
```

```ruby
# app/services/cnes/import.rb
# Importação da base mensal do CNES pelo operador (ADR 0028; spec 2026-10-05
# §5, §8). Os municípios saem do city_profile de cada cidade ATIVA (fonte única
# do IBGE, contratos §3); cidade sem IBGE ou fora do ar é pulada e relatada,
# sem derrubar as outras. Um retrato por município encontrado no arquivo.
# Reasons: :invalid_competence, :file_not_found, :no_city.
module Cnes
  module Import
    COMPETENCE = /\A\d{4}(0[1-9]|1[0-2])\z/

    module_function

    def call(competence:, path:)
      return Result.fail(:invalid_competence) unless competence.to_s.match?(COMPETENCE)

      wanted, skipped = municipalities
      return Result.fail(:no_city, details: { skipped: skipped }) if wanted.empty?

      imported = {}
      OfficialArchive.open(path) do |archive|
        BaseReader.read(archive, municipalities: wanted.transform_keys { |ibge| ibge[0, 6] }.to_h { |k, v| [ k, v.first ] })
                  .each do |ibge, data|
          if data[:establishments].empty?
            wanted[ibge].last.each { |slug| skipped << { slug: slug, reason: "not_in_file" } }
            next
          end

          SnapshotWriter.write!(competence: competence.to_s, ibge_code: ibge, **data)
          imported[ibge] = data.transform_values(&:size)
        end
      end
      Platform.audit("cnes.snapshot_imported", competence: competence.to_s, ibge_codes_count: imported.size) if imported.any?
      Result.ok(imported: imported, skipped: skipped)
    rescue OfficialArchive::NotFound => e
      Result.fail(:file_not_found, message: e.message)
    end

    # { "4106902" => ["4106902", ["curitiba"]] } e os pulados.
    def municipalities
      wanted = {}
      skipped = []
      City.active.order(:slug).each do |city|
        ibge = CityConnection.with(city) { CityProfile.current&.ibge_code }
        next skipped << { slug: city.slug, reason: "no_ibge_code" } if ibge.blank?

        (wanted[ibge] ||= [ ibge, [] ]).last << city.slug
      rescue *Maintenance::CityConnectionErrors::CLASSES
        skipped << { slug: city.slug, reason: "city_unreachable" }
      end
      [ wanted, skipped ]
    end
  end
end
```

```ruby
# lib/tasks/cnes.rake
# Importação da base mensal do CNES pelo operador (ADR 0028; spec 2026-10-05
# §5). path: BASE_DE_DADOS_CNES_AAAAMM.ZIP ou a pasta extraída. A saída nunca
# leva CPF nem CNS.
namespace :cnes do
  desc "Importa a base mensal do CNES para as cidades ativas. Uso: cnes:import[AAAAMM,path]"
  task :import, %i[competence path] => :environment do |_t, args|
    abort "uso: rails 'cnes:import[AAAAMM,path]'" if args[:competence].blank? || args[:path].blank?

    result = Cnes::Import.call(competence: args[:competence], path: args[:path])
    abort "[cnes:import] #{result.reason} #{result.message} #{result.details}".strip if result.failure?

    result.payload[:imported].each do |ibge, counts|
      puts "[cnes:import] #{ibge}: #{counts[:establishments]} estabelecimentos, #{counts[:teams]} equipes, " \
           "#{counts[:bonds]} vínculos"
    end
    result.payload[:skipped].each { |s| puts "[cnes:import] #{s[:slug]} pulada (#{s[:reason]})" }
  end
end
```

- [ ] **Step 5: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec spec/services/cnes spec/tasks/cnes_rake_spec.rb`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod16 add app/services/cnes/base_layout.rb app/services/cnes/base_reader.rb app/services/cnes/import.rb lib/tasks/cnes.rake spec/fixtures/cnes spec/services/cnes/import_spec.rb spec/tasks/cnes_rake_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod16 commit -m "feat: import the monthly CNES base for active cities' municipalities

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 16: Edição manual pontual — CNES da unidade e CPF do profissional

**Files:**
- Modify: `app/controllers/health_units_controller.rb`, `app/controllers/concerns/professional_rendering.rb`, `app/controllers/professionals_controller.rb`
- Test: `spec/requests/record_mode_manual_edits_spec.rb`

**Interfaces:**
- Consumes: `HealthUnit#cnes`, `Professional#cpf`/`#cpf_masked`, `Professional::FIELDS` com `cpf`, `UNIQUE_REASONS[:cpf_taken]` (Task 2).
- Produces: `POST /attendance/units` e `POST /attendance/units/:id` aceitam `cnes` (só quando a chave vem; `null`/`""` limpam; 422 `invalid_cnes`, `cnes_taken`); `unit_json` com `cnes`; `POST /professionals` e `POST /professionals/:id` aceitam `cpf` (admin; 422 `invalid` com `fields: ["cpf"]`, 409 `cpf_taken`); `profile_json` com `cpf_masked` (e `cpf` na ficha completa).

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/requests/record_mode_manual_edits_spec.rb
require "rails_helper"

# ADR 0028 (spec 2026-10-05 §5): além de confirmar o CNES, o admin corrige à
# mão o CNES da unidade e o CPF do profissional. CPF sai mascarado na lista.
RSpec.describe "Edição manual de CNES e CPF", type: :request do
  let(:admin) { staff_with("admin@cidade.gov.br", "municipal_admin") }
  let(:doctor) { staff_with("medica@cidade.gov.br", "health_professional") }
  def json = JSON.parse(response.body)

  before { sign_in_as(admin) }

  it "CNES da unidade: grava normalizado, limpa com null, recusa formato e repetição" do
    json_post "/attendance/units", name: "UBS Centro", kind: "ubs", cnes: "000.000-1"
    expect(json["unit"]["cnes"]).to eq("0000001")
    id = json["unit"]["id"]
    json_post "/attendance/units", name: "UBS Norte", kind: "ubs", cnes: "0000001"
    expect([ response.status, json ]).to eq([ 422, { "error" => "cnes_taken" } ])
    json_post "/attendance/units/#{id}", name: "UBS Centro", kind: "ubs", cnes: "12"
    expect([ response.status, json ]).to eq([ 422, { "error" => "invalid_cnes" } ])
    json_post "/attendance/units/#{id}", name: "UBS Centro", kind: "ubs", cnes: nil
    expect(json["unit"]["cnes"]).to be_nil
    json_post "/attendance/units/#{id}", name: "UBS Centro", kind: "ubs"
    expect(HealthUnit.find(id).cnes).to be_nil
  end

  it "CPF do profissional: o admin grava; a lista mascara; repetição é 409; inválido é 422" do
    json_post "/professionals", user_id: doctor.id, professional_name: "Helena", council: "CRM", council_state: "PR",
                                registration_number: "12345", cns: "700000000000005", cpf: "529.982.247-25"
    expect(response).to have_http_status(:created)
    expect(json["professional"]).to include("cpf_masked" => "***.982.247-**", "cpf" => "52998224725")
    get "/professionals"
    expect(response.body).not_to include("52998224725")

    other = staff_with("enfermeira@cidade.gov.br", "health_professional")
    json_post "/professionals", user_id: other.id, professional_name: "Carla", council: "COREN", council_state: "PR",
                                registration_number: "54321", cns: Professionals::Cns.generate("x"), cpf: "52998224725"
    expect([ response.status, json["error"] ]).to eq([ 409, "cpf_taken" ])
    json_post "/professionals/#{Professional.first.id}", cpf: "52998224724"
    expect([ response.status, json ]).to eq([ 422, { "error" => "invalid", "fields" => [ "cpf" ] } ])
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec spec/requests/record_mode_manual_edits_spec.rb`
Expected: FAIL (`unit.cnes` ausente na resposta).

- [ ] **Step 3: Implemente**

Em `app/controllers/health_units_controller.rb`:
- em `create`, troque `error = assign_address(unit)` por `error = assign_address(unit) || assign_cnes(unit)`; em `update`, o mesmo com `@unit`;
- nos dois `rescue ActiveRecord::RecordNotUnique`, troque o corpo por `render json: { error: $!.message.include?("idx_health_units_cnes") ? "cnes_taken" : "unit_name_taken" }, status: :unprocessable_entity` (troque `rescue ActiveRecord::RecordNotUnique` por `rescue ActiveRecord::RecordNotUnique => e` e use `e.message`);
- em `render_invalid`, antes do `elsif unit.errors.details[:kind].present?`:

```ruby
    elsif unit.errors.details[:cnes]&.any? { |e| e[:error] == :taken }
      render json: { error: "cnes_taken" }, status: :unprocessable_entity
    elsif unit.errors.details[:cnes].present?
      render json: { error: "invalid_cnes" }, status: :unprocessable_entity
```

- em `private`, depois de `assign_address`:

```ruby
  # CNES da unidade (ADR 0028): só muda quando a chave vem no corpo; null e ""
  # limpam. Formato e unicidade ficam com o modelo (render_invalid).
  def assign_cnes(unit)
    return nil unless params.key?("cnes")

    value = params["cnes"]
    return "invalid_cnes" unless value.nil? || value.is_a?(String)

    unit.cnes = value.presence
    nil
  end
```

- `unit_json` passa a `json = { id: unit.id, name: unit.name, kind: unit.kind, cnes: unit.cnes }.merge(...)`.

Em `app/controllers/concerns/professional_rendering.rb`, `profile_json`: o hash base ganha `cpf_masked: p.cpf_masked` depois de `cns_masked:`, e o `merge!` da ficha completa ganha `cpf: p.cpf`.

Em `app/controllers/professionals_controller.rb`, `ERROR_STATUS` ganha `cpf_taken: :conflict`.

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec spec/requests/record_mode_manual_edits_spec.rb spec/requests/health_units_spec.rb spec/requests/health_units_address_spec.rb spec/requests/professionals_spec.rb spec/commands/professionals`
Expected: PASS (exemplos existentes que comparam o hash inteiro da unidade ou do perfil ganham `cnes`/`cpf_masked` no esperado).

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod16 add app/controllers/health_units_controller.rb app/controllers/concerns/professional_rendering.rb app/controllers/professionals_controller.rb spec/requests/record_mode_manual_edits_spec.rb spec/requests/health_units_spec.rb spec/requests/professionals_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod16 commit -m "feat: let admins edit a unit's CNES and a professional's CPF

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 17: `Cnes::Proposal` — propostas e divergências contra o retrato mais recente

**Files:**
- Create: `app/services/cnes/proposal.rb`, `spec/support/cnes_helpers.rb`
- Modify: `spec/rails_helper.rb` (`require_relative "support/cnes_helpers"`)
- Test: `spec/services/cnes/proposal_spec.rb`

**Interfaces:**
- Consumes: `Cnes::SnapshotWriter` (Task 14), `HealthUnit`, `HealthTeam`, `HealthTeamMember`, `Professional`, `ProfessionalLink`, `CityProfile`.
- Produces: `Cnes::Proposal.for(city) -> { snapshot: CnesSnapshot|nil, proposals: Array<Hash>, divergences: Array<Hash> }`, calculado dentro de `CityConnection.with(city)`. Proposta: `{ id:, kind:, action:, local:, cnes:, confidence:, target: }` — `target` é interno (o que `Cnes::Apply` grava), nunca vai para a resposta; `local`/`cnes` no formato do contrato (`{ name, cnes?, ine?, cbo?, cpf_masked?, cns_masked? }`). Divergência: `{ kind:, subject: { type:, id:, label: }, detail: String|nil }`. `id` = 24 hex estáveis para o mesmo retrato e o mesmo alvo. Helper de spec `cnes_snapshot!(ibge_code: "4106902", competence:, establishments:, teams:, bonds:)`.

Regras (desvio 13):
- **unidade/link** (`probable`): estabelecimento sem unidade local com o mesmo CNES, e uma unidade local sem CNES com o mesmo nome normalizado (sem acento, maiúsculas, só letras e dígitos). **unidade/create** (`exact`): estabelecimento sem par local e com equipe ativa no retrato.
- **equipe/create** (`exact`): equipe ativa, tipo 70/76, INE ausente localmente, estabelecimento já casado com uma unidade local. **equipe/end**: equipe local ativa cujo INE está inativo no retrato (`exact`) ou ausente dele (`probable`).
- **membro/create**: vínculo com INE de equipe local ativa, profissional local achado por CPF (`exact`) ou só por CNS (`probable`), sem membro ativo naquela equipe. **membro/end** (`exact`): membro ativo cujo profissional não tem vínculo com aquele INE no retrato.
- **divergências**: `unit_without_cnes` (unidade ativa sem CNES e sem proposta de link); `team_inactive_in_cnes` (a mesma condição da equipe/end); `no_bond_in_cnes` (profissional com vínculo ativo numa unidade com CNES e nenhum vínculo no retrato naquele CNES); `cbo_mismatch` (vínculo do retrato no CNES da unidade com CBO que não está entre os CBOs dos vínculos ativos do profissional ali).

- [ ] **Step 1: Helper de spec**

```ruby
# spec/support/cnes_helpers.rb
# Módulo 16 (ADR 0028): retrato do CNES na plataforma e cadastro local
# coerentes, para Proposal/Apply.
module CnesHelpers
  def cnes_snapshot!(ibge_code: "4106902", competence: "202609", establishments: [], teams: [], bonds: [])
    Cnes::SnapshotWriter.write!(competence: competence, ibge_code: ibge_code, establishments: establishments,
                                teams: teams, bonds: bonds)
  end

  def cnes_city!
    city = City.find_by(slug: TEST_CITY_A.slug) ||
           City.create!(slug: TEST_CITY_A.slug, name: TEST_CITY_A.name, status: "active",
                        database_url: TEST_CITY_A.database_url, encryption_key: TEST_CITY_A.encryption_key,
                        schema_version: CitySchema.expected_version.to_s)
    (CityProfile.current || CityProfile.new(name: "Curitiba", uf: "PR")).update!(ibge_code: "4106902")
    city
  end

  def professional_with!(email, unit:, cbo:, cpf: nil, cns: nil)
    user = staff_with(email, "health_professional")
    link_professional!(user, unit, cbo: cbo)
    user.reload.professional.tap { |p| p.update!(cpf: cpf, cns: cns || p.cns) }
  end
end

RSpec.configure { |c| c.include CnesHelpers }
```

- [ ] **Step 2: Escreva a spec que falha**

```ruby
# spec/services/cnes/proposal_spec.rb
require "rails_helper"

# ADR 0028 (spec 2026-10-05 §5; contratos §5.2): o retrato mais recente contra
# o cadastro da cidade. Nada é aplicado aqui — só proposto.
RSpec.describe Cnes::Proposal do
  let!(:city) { cnes_city! }
  let!(:jardim) { create_unit("UBS Jardim das Flores") }
  let!(:upa) { create_unit("UPA 24h Centro", kind: "upa") }

  def proposals = described_class.for(city)[:proposals]
  def divergences = described_class.for(city)[:divergences]
  def find(kind, action) = proposals.find { |p| p[:kind] == kind && p[:action] == action }

  it "sem retrato: snapshot nil e listas vazias" do
    CityProfile.current.update!(ibge_code: "4115200")
    expect(described_class.for(city)).to eq(snapshot: nil, proposals: [], divergences: [])
  end

  it "unidade: liga por nome (provável), cria a que tem equipe (exata), aponta a unidade sem CNES" do
    cnes_snapshot!(establishments: [ { cnes: "0000001", name: "UBS JARDIM DAS FLORES", unit_type: "02" },
                                     { cnes: "0000002", name: "UBS VILA ESPERANCA", unit_type: "02" },
                                     { cnes: "0000004", name: "HOSPITAL PARTICULAR", unit_type: "05" } ],
                   teams: [ { ine: "0000123457", kind: "76", cnes: "0000002", name: "EAP VILA", active: true } ])

    link = find("unit", "link")
    expect(link).to include(confidence: "probable", local: { name: "UBS Jardim das Flores", cnes: nil },
                            cnes: { name: "UBS JARDIM DAS FLORES", cnes: "0000001" })
    expect(link[:target]).to eq(health_unit_id: jardim.id, cnes: "0000001")
    create = find("unit", "create")
    expect(create).to include(confidence: "exact", local: nil, cnes: { name: "UBS VILA ESPERANCA", cnes: "0000002" })
    expect(proposals.map { |p| p[:cnes][:cnes] }).not_to include("0000004")
    expect(divergences).to contain_exactly(
      { kind: "unit_without_cnes", subject: { type: "health_unit", id: upa.id, label: "UPA 24h Centro" }, detail: nil }
    )
  end

  it "equipe e membro: cria equipe da unidade casada; membro por CPF exato; encerra o que saiu do CNES" do
    jardim.update!(cnes: "0000001")
    helena = professional_with!("medica@cidade.gov.br", unit: jardim, cbo: "225125", cpf: "52998224725")
    carla = professional_with!("enfermeira@cidade.gov.br", unit: jardim, cbo: "223505", cns: "700000000000005")
    old_team = HealthTeam.create!(ine: "0000123458", kind: "70", name: "ESF JARDIM 2", health_unit: jardim)
    gone = HealthTeamMember.create!(professional: carla, health_team: old_team, cbo_code: "223505", started_on: Date.current)
    cnes_snapshot!(establishments: [ { cnes: "0000001", name: "UBS JARDIM DAS FLORES", unit_type: "02" } ],
                   teams: [ { ine: "0000123456", kind: "70", cnes: "0000001", name: "ESF JARDIM 1", active: true },
                            { ine: "0000123458", kind: "70", cnes: "0000001", name: "ESF JARDIM 2", active: false } ],
                   bonds: [ { cnes: "0000001", ine: "0000123456", cbo_code: "225125", cpf: "52998224725", cns: nil },
                            { cnes: "0000001", ine: nil, cbo_code: "225142", cpf: "52998224725", cns: nil } ])

    expect(find("team", "create")).to include(confidence: "exact", cnes: { name: "ESF JARDIM 1", cnes: "0000001", ine: "0000123456" })
    expect(find("team", "end")).to include(confidence: "exact", local: { name: "ESF JARDIM 2", ine: "0000123458" })
    expect(find("member", "end")[:target]).to eq(health_team_member_id: gone.id)
    expect(divergences.map { |d| d[:kind] }).to include("team_inactive_in_cnes", "no_bond_in_cnes", "cbo_mismatch")
    expect(divergences.find { |d| d[:kind] == "no_bond_in_cnes" }[:subject]).to include(type: "professional", id: carla.id)
    expect(divergences.find { |d| d[:kind] == "cbo_mismatch" }[:detail]).to eq("CNES informa CBO 225142; cadastro: 225125")

    HealthTeam.create!(ine: "0000123456", kind: "70", name: "ESF JARDIM 1", health_unit: jardim)
    member = find("member", "create")
    expect(member).to include(confidence: "exact",
                              local: { name: helena.professional_name, cpf_masked: "***.982.247-**", cns_masked: helena.cns_masked },
                              cnes: { name: "ESF JARDIM 1", ine: "0000123456", cbo: "225125", cpf_masked: "***.982.247-**",
                                      cns_masked: nil })
  end

  it "membro só por CNS é provável; id estável para o mesmo retrato" do
    jardim.update!(cnes: "0000001")
    HealthTeam.create!(ine: "0000123456", kind: "70", name: "ESF 1", health_unit: jardim)
    professional_with!("enfermeira@cidade.gov.br", unit: jardim, cbo: "223505", cns: "700000000000005")
    cnes_snapshot!(establishments: [ { cnes: "0000001", name: "UBS JARDIM DAS FLORES", unit_type: "02" } ],
                   teams: [ { ine: "0000123456", kind: "70", cnes: "0000001", name: "ESF 1", active: true } ],
                   bonds: [ { cnes: "0000001", ine: "0000123456", cbo_code: "223505", cpf: nil, cns: "700000000000005" } ])
    expect(find("member", "create")[:confidence]).to eq("probable")
    expect(proposals.map { |p| p[:id] }).to eq(proposals.map { |p| p[:id] })
    expect(find("member", "create")[:id]).to match(/\A\h{24}\z/)
  end

  it "o retrato mais recente vence" do
    cnes_snapshot!(competence: "202608", establishments: [ { cnes: "0000009", name: "VELHA", unit_type: "02" } ])
    cnes_snapshot!(competence: "202609", establishments: [])
    expect(described_class.for(city)[:snapshot].competence).to eq("202609")
  end
end
```

- [ ] **Step 3: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec spec/services/cnes/proposal_spec.rb`
Expected: FAIL (`uninitialized constant Cnes::Proposal`).

- [ ] **Step 4: Implemente**

```ruby
# app/services/cnes/proposal.rb
require "digest"

# Propostas de casamento entre o retrato mais recente do CNES e o cadastro da
# cidade (ADR 0028; spec 2026-10-05 §5; contratos §5.2; desvio 13). Cálculo em
# memória, sem escrita: nada é aplicado sem confirmação (Cnes::Apply). O `id`
# deriva do retrato e do alvo — mudou o cadastro ou chegou retrato novo, a
# proposta antiga deixa de existir (Apply a pula como `stale`). CPF/CNS só
# saem mascarados.
module Cnes
  class Proposal
    def self.for(city) = CityConnection.with(city) { new.call }

    def call
      ibge = CityProfile.current&.ibge_code
      @snapshot = ibge && CnesSnapshot.where(ibge_code: ibge).order(competence: :desc).first
      return { snapshot: nil, proposals: [], divergences: [] } unless @snapshot

      load_data
      @proposals = []
      @divergences = []
      units
      teams
      members
      bond_divergences
      { snapshot: @snapshot, proposals: @proposals, divergences: @divergences.uniq }
    end

    private

    def load_data
      @establishments = @snapshot.establishments.to_a
      @cnes_teams = @snapshot.teams.to_a
      @bonds = @snapshot.bonds.to_a
      @units = HealthUnit.order(:name).to_a
      @unit_by_cnes = @units.select(&:cnes).index_by(&:cnes)
      @teams = HealthTeam.all.to_a
      @team_by_ine = @teams.index_by(&:ine)
      @professionals = Professional.all.to_a
      @by_cpf = @professionals.select(&:cpf).index_by(&:cpf)
      @by_cns = @professionals.index_by(&:cns)
      @members = HealthTeamMember.active.includes(:health_team, :professional).to_a
    end

    def units
      teamed = @cnes_teams.select(&:active).map(&:cnes).to_set
      linked = []
      @establishments.each do |est|
        next if @unit_by_cnes.key?(est.cnes)

        candidate = @units.find { |u| u.cnes.nil? && !linked.include?(u.id) && normalize(u.name) == normalize(est.name) }
        cnes_side = { name: est.name, cnes: est.cnes }
        if candidate
          linked << candidate.id
          add("unit", "link", "probable", { name: candidate.name, cnes: nil }, cnes_side,
              health_unit_id: candidate.id, cnes: est.cnes)
        elsif teamed.include?(est.cnes)
          add("unit", "create", "exact", nil, cnes_side, cnes: est.cnes, name: est.name)
        end
      end
      @units.select { |u| u.active && u.cnes.nil? && !linked.include?(u.id) }.each do |u|
        diverge("unit_without_cnes", "health_unit", u.id, u.name, nil)
      end
    end

    def teams
      snapshot_by_ine = @cnes_teams.index_by(&:ine)
      @cnes_teams.each do |team|
        unit = @unit_by_cnes[team.cnes]
        next unless team.active && HealthTeam::KINDS.include?(team.kind) && unit && !@team_by_ine.key?(team.ine)

        add("team", "create", "exact", nil, { name: team.name.to_s, cnes: team.cnes, ine: team.ine },
            ine: team.ine, kind: team.kind, name: team.name, health_unit_id: unit.id)
      end
      @teams.select(&:active).each do |team|
        remote = snapshot_by_ine[team.ine]
        next if remote&.active

        detail = remote ? "Equipe inativa no CNES" : "INE ausente do CNES"
        add("team", "end", remote ? "exact" : "probable", { name: team.name.to_s, ine: team.ine },
            { name: remote&.name.to_s, ine: team.ine }, health_team_id: team.id)
        diverge("team_inactive_in_cnes", "health_team", team.id, team.name.presence || team.ine, detail)
      end
    end

    def members
      @bonds.select(&:ine).each do |bond|
        team = @team_by_ine[bond.ine]
        next unless team&.active

        professional, confidence = match(bond)
        next unless professional
        next if @members.any? { |m| m.professional_id == professional.id && m.health_team_id == team.id }

        add("member", "create", confidence, person(professional),
            { name: team.name.to_s, ine: team.ine, cbo: bond.cbo_code, cpf_masked: bond.cpf_masked, cns_masked: bond.cns_masked },
            professional_id: professional.id, health_team_id: team.id, cbo_code: bond.cbo_code)
      end
      @members.each do |member|
        next if @bonds.any? { |b| b.ine == member.health_team.ine && same_person?(b, member.professional) }

        add("member", "end", "exact", person(member.professional),
            { name: member.health_team.name.to_s, ine: member.health_team.ine, cbo: member.cbo_code },
            health_team_member_id: member.id)
      end
    end

    def bond_divergences
      ProfessionalLink.active.includes(:health_unit, :professional).group_by { |l| [ l.professional, l.health_unit ] }
                      .each do |(professional, unit), links|
        next unless unit.cnes

        here = @bonds.select { |b| b.cnes == unit.cnes && same_person?(b, professional) }
        label = professional.professional_name
        if here.empty?
          diverge("no_bond_in_cnes", "professional", professional.id, label, "Sem vínculo no CNES em #{unit.name}")
          next
        end
        local = links.map(&:cbo_code).uniq.sort
        here.map(&:cbo_code).uniq.reject { |cbo| local.include?(cbo) }.each do |cbo|
          diverge("cbo_mismatch", "professional", professional.id, label, "CNES informa CBO #{cbo}; cadastro: #{local.join(', ')}")
        end
      end
    end

    def match(bond)
      return [ @by_cpf[bond.cpf], "exact" ] if bond.cpf && @by_cpf[bond.cpf]
      return [ @by_cns[bond.cns], "probable" ] if bond.cns && @by_cns[bond.cns]

      [ nil, nil ]
    end

    def same_person?(bond, professional)
      (bond.cpf && bond.cpf == professional.cpf) || (bond.cns && bond.cns == professional.cns)
    end

    def person(p) = { name: p.professional_name, cpf_masked: p.cpf_masked, cns_masked: p.cns_masked }

    def add(kind, action, confidence, local, cnes, **target)
      id = Digest::SHA256.hexdigest([ @snapshot.id, kind, action, target.sort.to_s ].join("|"))[0, 24]
      @proposals << { id: id, kind: kind, action: action, local: local, cnes: cnes, confidence: confidence, target: target }
    end

    def diverge(kind, type, id, label, detail)
      @divergences << { kind: kind, subject: { type: type, id: id, label: label }, detail: detail }
    end

    def normalize(name) = I18n.transliterate(name.to_s).upcase.gsub(/[^A-Z0-9]+/, " ").squish
  end
end
```

- [ ] **Step 5: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec spec/services/cnes/proposal_spec.rb`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod16 add app/services/cnes/proposal.rb spec/support/cnes_helpers.rb spec/rails_helper.rb spec/services/cnes/proposal_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod16 commit -m "feat: propose CNES matches and flag divergences against the city's registry

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 18: `Cnes::Apply`, `GET /cnes` e `POST /cnes/apply`

**Files:**
- Create: `app/commands/cnes/apply.rb`, `app/controllers/cnes_controller.rb`
- Modify: `config/routes.rb`
- Test: `spec/requests/cnes_spec.rb`, `spec/commands/cnes/apply_lock_spec.rb`

**Interfaces:**
- Consumes: `Cnes::Proposal.for` (Task 17), `IntegrationPolicy#manage?` (Task 9), `MfaStepUp`.
- Produces: `Cnes::Apply.call(city:, proposal_ids:, by:) -> Result` (`ok(applied: Integer, skipped: [{ id:, reason: "stale"|"conflict" }])`; `fail(:invalid_proposals)` — lista vazia, mais de 500, ou id fora do formato); `GET /cnes`, `POST /cnes/apply` (contratos §5.2; resposta 200 sem envelope).

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/requests/cnes_spec.rb
require "rails_helper"

# ADR 0028 (spec 2026-10-05 §5; contratos §5.2): o municipal_admin vê propostas
# e divergências e confirma com step-up; proposta que mudou desde a leitura é
# pulada; nada é aplicado sem confirmação.
RSpec.describe "CNES da cidade", type: :request do
  let(:admin) do
    staff_with("admin@cidade.gov.br", "municipal_admin").tap do |u|
      Mfa::Enroll.call(u)
      u.update!(otp_enabled: true)
    end
  end
  let!(:jardim) { create_unit("UBS Jardim das Flores") }
  def json = JSON.parse(response.body)
  def apply(ids) = post("/cnes/apply", params: { proposal_ids: ids }, as: :json)

  before do
    cnes_city!
    cnes_snapshot!(establishments: [ { cnes: "0000001", name: "UBS JARDIM DAS FLORES", unit_type: "02" } ],
                   bonds: [ { cnes: "0000001", ine: nil, cbo_code: "225125", cpf: "52998224725", cns: "700000000000005" } ])
    sign_in_as(admin).update!(mfa_verified_at: Time.current)
  end

  it "sem retrato: snapshot null e listas vazias" do
    CityProfile.current.update!(ibge_code: "4115200")
    get "/cnes"
    expect(json).to eq("snapshot" => nil, "proposals" => [], "divergences" => [])
  end

  it "GET mostra a proposta sem o alvo interno e sem CPF/CNS em claro; nada aplicado" do
    get "/cnes"
    expect(json["snapshot"]).to include("competence" => "202609")
    expect(json["proposals"].first.keys).to match_array(%w[id kind action local cnes confidence])
    expect(response.body).not_to include("52998224725")
    expect(response.body).not_to include("700000000000005")
    expect(jardim.reload.cnes).to be_nil
  end

  it "confirma com step-up, aplica, publica evento só com ids; a mesma proposta de novo é stale" do
    get "/cnes"
    id = json["proposals"].first["id"]
    apply([ id ])
    expect(response).to have_http_status(:ok)
    expect(json).to eq("applied" => 1, "skipped" => [])
    expect(jardim.reload.cnes).to eq("0000001")
    expect(DomainEvent.where(name: "cnes.proposals_applied").map(&:payload)).to eq([ { "user_id" => admin.id, "count" => 1 } ])
    apply([ id ])
    expect(json).to eq("applied" => 0, "skipped" => [ { "id" => id, "reason" => "stale" } ])
  end

  # Review Focus 4: o cadastro mudou depois da leitura.
  it "unidade editada à mão depois da leitura: stale, nada sobrescrito" do
    get "/cnes"
    id = json["proposals"].first["id"]
    jardim.update!(name: "UBS Jardim Renomeada")
    apply([ id ])
    expect(json["skipped"]).to eq([ { "id" => id, "reason" => "stale" } ])
    expect(jardim.reload.cnes).to be_nil
  end

  it "sem step-up: 401 mfa_required; lista inválida: 422; papel errado: 403" do
    get "/cnes"
    id = json["proposals"].first["id"]
    Session.find_by!(user: admin).update!(mfa_verified_at: 10.minutes.ago)
    apply([ id ])
    expect([ response.status, json ]).to eq([ 401, { "error" => "mfa_required" } ])
    Session.find_by!(user: admin).update!(mfa_verified_at: Time.current)
    apply([])
    expect([ response.status, json ]).to eq([ 422, { "error" => "invalid_proposals" } ])
    apply([ "../etc" ])
    expect(response).to have_http_status(:unprocessable_entity)
    sign_in_as(staff_with("recepcao@cidade.gov.br", "citizen_verifier"))
    get "/cnes"
    expect([ response.status, json ]).to eq([ 403, { "error" => "missing_role" } ])
  end
end
```

```ruby
# spec/commands/cnes/apply_lock_spec.rb
require "rails_helper"

# Review Focus 4: dois administradores confirmam a mesma proposta ao mesmo
# tempo. O segundo espera a trava (pg_advisory_xact_lock), recalcula sobre o
# estado já commitado e recebe stale — sem escrita dupla. Threads reais, sem
# fixture transacional (padrão de spec/commands/territory/replace_coverage_lock_spec.rb).
RSpec.describe "Cnes::Apply sob concorrência" do
  self.use_transactional_tests = false

  let(:release) { Queue.new }
  let(:threads) { [] }
  let(:ids) { {} }
  let(:by) { Data.define(:id).new(id: SecureRandom.uuid) }

  before do
    CityConnection.with(TEST_CITY_A) do
      tag = SecureRandom.hex(4)
      ids[:unit] = HealthUnit.create!(name: "UBS Trava CNES #{tag}", kind: "ubs").id
      ids[:name] = "UBS TRAVA CNES #{tag.upcase}"
      ids[:profile_created] = CityProfile.current.nil?
      profile = CityProfile.current || CityProfile.new(name: "Cidade", uf: "PR")
      ids[:previous_ibge] = profile.ibge_code
      profile.update!(ibge_code: "4106902")
    end
    ids[:snapshot] = Cnes::SnapshotWriter.write!(competence: "209912", ibge_code: "4106902",
                                                 establishments: [ { cnes: "7777777", name: ids[:name], unit_type: "02" } ],
                                                 teams: [], bonds: []).id
  end

  after do
    2.times { release << true }
    threads.each { |t| t.join(5) || t.kill }
    CnesSnapshot.where(id: ids[:snapshot]).delete_all
    CityConnection.with(TEST_CITY_A) do
      ApplicationRecord.transaction do
        ApplicationRecord.connection.execute("SET LOCAL session_replication_role = replica")
        DomainEvent.where(name: "cnes.proposals_applied").where("payload->>'user_id' = ?", by.id).delete_all
        HealthUnit.where(id: ids[:unit]).delete_all
        ids[:profile_created] ? CityProfile.delete_all : CityProfile.update_all(ibge_code: ids[:previous_ibge])
      end
    end
  end

  def in_city(&) = CityConnection.with(TEST_CITY_A) { ApplicationRecord.transaction(&) }

  it "o segundo espera e recebe stale" do
    id = Cnes::Proposal.for(TEST_CITY_A)[:proposals].find { |p| p[:target][:health_unit_id] == ids[:unit] }[:id]
    first = Queue.new
    locked = Queue.new
    threads << Thread.new do
      in_city do
        first << Cnes::Apply.call(city: TEST_CITY_A, proposal_ids: [ id ], by: by)
        locked << true
        release.pop
      end
    end
    locked.pop(timeout: 5) or raise "a primeira chamada não pegou a trava"

    second = Queue.new
    threads << Thread.new do
      CityConnection.with(TEST_CITY_A) { second << Cnes::Apply.call(city: TEST_CITY_A, proposal_ids: [ id ], by: by) }
    rescue StandardError => e
      second << e
    end
    expect(second.pop(timeout: 0.5)).to be_nil

    release << true
    threads.first.join(5)
    outcome = second.pop(timeout: 5)
    expect(first.pop.payload).to eq(applied: 1, skipped: [])
    expect(outcome.payload).to eq(applied: 0, skipped: [ { id: id, reason: "stale" } ])
    expect(CityConnection.with(TEST_CITY_A) { HealthUnit.find(ids[:unit]).cnes }).to eq("7777777")
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec spec/requests/cnes_spec.rb spec/commands/cnes/apply_lock_spec.rb`
Expected: FAIL (`No route matches [GET] "/cnes"`).

- [ ] **Step 3: Implemente**

```ruby
# app/commands/cnes/apply.rb
# Confirma propostas do CNES (ADR 0028; spec 2026-10-05 §5; contratos §5.2).
# Sob trava da cidade (pg_advisory_xact_lock), RECALCULA as propostas e aplica
# só as que ainda existem com o mesmo id: proposta que mudou desde a leitura é
# `stale`; a que bate em regra do banco (nome de unidade repetido, membro já
# ativo) é `conflict`, num savepoint, sem derrubar as outras. Unidades antes de
# equipes antes de membros.
module Cnes
  module Apply
    ORDER = { "unit" => 0, "team" => 1, "member" => 2 }.freeze
    MAX = 500
    ID = /\A\h{24}\z/

    module_function

    def call(city:, proposal_ids:, by:)
      ids = Array(proposal_ids)
      valid = ids.any? && ids.size <= MAX && ids.all? { |id| id.is_a?(String) && id.match?(ID) }
      return Result.fail(:invalid_proposals) unless valid

      applied = 0
      skipped = []
      ApplicationRecord.transaction do
        ApplicationRecord.connection.execute("SELECT pg_advisory_xact_lock(hashtext('cnes_apply'))")
        current = Proposal.for(city)[:proposals].index_by { |p| p[:id] }
        chosen = ids.uniq.filter_map do |id|
          next current[id] if current.key?(id)

          skipped << { id: id, reason: "stale" }
          nil
        end
        chosen.sort_by { |p| ORDER.fetch(p[:kind]) }.each do |proposal|
          ApplicationRecord.transaction(requires_new: true) { apply!(proposal) }
          applied += 1
        rescue ActiveRecord::RecordInvalid, ActiveRecord::RecordNotUnique
          skipped << { id: proposal[:id], reason: "conflict" }
        end
        DomainEvents.publish("cnes.proposals_applied", user_id: by.id, count: applied) if applied.positive?
      end
      Result.ok(applied: applied, skipped: skipped)
    end

    def apply!(proposal)
      t = proposal[:target]
      today = Time.zone.today
      case [ proposal[:kind], proposal[:action] ]
      when %w[unit link] then HealthUnit.lock.find(t[:health_unit_id]).update!(cnes: t[:cnes])
      when %w[unit create] then HealthUnit.create!(name: t[:name], kind: "ubs", cnes: t[:cnes])
      when %w[team create]
        HealthTeam.create!(ine: t[:ine], kind: t[:kind], name: t[:name], health_unit_id: t[:health_unit_id])
      when %w[team end] then HealthTeam.lock.find(t[:health_team_id]).update!(active: false)
      when %w[member create]
        HealthTeamMember.create!(professional_id: t[:professional_id], health_team_id: t[:health_team_id],
                                 cbo_code: t[:cbo_code], started_on: today)
      when %w[member end]
        member = HealthTeamMember.lock.find(t[:health_team_member_id])
        member.update!(ended_on: [ today, member.started_on ].max)
      else
        raise ArgumentError, "proposta desconhecida: #{proposal[:kind]}/#{proposal[:action]}"
      end
    end
  end
end
```

```ruby
# app/controllers/cnes_controller.rb
# CNES da cidade (ADR 0028; contratos §5.2): propostas e divergências contra o
# retrato mais recente; confirmar exige step-up. Só municipal_admin.
class CnesController < ApplicationController
  include Authentication
  include MfaStepUp

  wrap_parameters false

  before_action :require_admin

  def show
    data = Cnes::Proposal.for(Current.city)
    snapshot = data[:snapshot]
    render json: {
      snapshot: snapshot && { competence: snapshot.competence, imported_at: snapshot.imported_at.iso8601 },
      proposals: data[:proposals].map { |p| p.except(:target) },
      divergences: data[:divergences]
    }
  end

  def apply
    return require_step_up! unless reauthenticated_recently?

    result = Cnes::Apply.call(city: Current.city, proposal_ids: request.request_parameters["proposal_ids"], by: Current.user)
    return render(json: { error: result.reason.to_s }, status: :unprocessable_entity) if result.failure?

    render json: result.payload
  end

  private

  def require_admin
    render json: { error: "missing_role" }, status: :forbidden unless IntegrationPolicy.new(Current.user, nil).manage?
  end
end
```

Em `config/routes.rb`, depois das rotas de Integrações:

```ruby
  # CNES da cidade (ADR 0028; contratos §5.2).
  get  "/cnes", to: "cnes#show"
  post "/cnes/apply", to: "cnes#apply"
```

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec spec/requests/cnes_spec.rb spec/commands/cnes/apply_lock_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod16 add app/commands/cnes/apply.rb app/controllers/cnes_controller.rb config/routes.rb spec/requests/cnes_spec.rb spec/commands/cnes/apply_lock_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod16 commit -m "feat: let municipal admins confirm CNES proposals under a city lock

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Fatia 5 — CADSUS (F-16.8)

### Task 19: Consulta ao CADSUS no balcão e `cadsus_confirmed` na validação

**Files:**
- Create: `app/commands/cadsus/lookup.rb`, `app/controllers/concerns/feature_gate.rb`
- Modify: `app/controllers/attendance_controller.rb`, `app/commands/citizens/verify.rb`, `config/routes.rb`
- Test: `spec/requests/attendance_cadsus_spec.rb`

**Interfaces:**
- Consumes: `Platform::Features.enabled?`, `.usable?` (Task 3), `Cadsus::Client` (Task 8), `Citizens::VerificationCodeMatch`, `Citizen#cadsus_pending_*` (Task 2).
- Produces: `FeatureGate` (concern; `require_feature(key, **before_action_options)` na classe; 403 `{ error: "feature_disabled", feature: key }`) — o plano do exportador usa nas rotas de Produção; `Cadsus::Lookup.call(citizen:, by:, session:, city:) -> Result` (`ok(found:, cns_masked:, birth_date_matches:, sex_matches:)`, `fail(:cadsus_unavailable)`); `Cadsus::Lookup::WINDOW = 10.minutes`; `POST /attendance/cadsus_lookup { cpf, code }`; `Citizens::Verify.call(..., cadsus_confirmed: false, session_id: nil)` com o motivo novo `:cadsus_lookup_missing` (409).

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/requests/attendance_cadsus_spec.rb
require "rails_helper"

# ADR 0028 (spec 2026-10-05 §7; contratos §5.4): consulta ao CADSUS no balcão,
# atrás do interruptor. Do CADSUS só ficam o CNS (depois de o atendente
# confirmar a validação) e a marca da conferência.
RSpec.describe "CADSUS na validação presencial", type: :request do
  include ActiveSupport::Testing::TimeHelpers

  before { Current.city = TEST_CITY_A; Rails.cache.clear }
  after { Current.reset }

  let(:city) { City.find_by!(slug: TEST_CITY_A.slug) }
  let(:verifier) { staff_with("atendente@cidade.gov.br", "citizen_verifier") }
  let(:admin) { staff_with("admin@cidade.gov.br", "municipal_admin") }
  let!(:citizen) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }
  let(:maintainer) do
    Maintainer.create!(email_address: "cad-#{SecureRandom.hex(3)}@rotasaude.app", password: "s3nha-forte-1",
                       otp_secret: ROTP::Base32.random, otp_enabled_at: Time.current)
  end
  def json = JSON.parse(response.body)

  def switch_on! = Platform::Features.set!(city: city, key: "cadsus_lookup", enabled: true, maintainer: maintainer)

  def credential!(username = "rota")
    IntegrationCredential.create!(kind: "cadsus", secret: { "username" => username, "password" => "x" },
                                  set_by_user: admin, set_at: Time.current)
  end

  def lookup(person = citizen, code = issue_code_for(person))
    json_post "/attendance/cadsus_lookup", cpf: person.cpf, code: code
    code
  end

  it "interruptor desligado: 403 feature_disabled e o CADSUS não é chamado" do
    credential!
    sign_in_as(verifier)
    expect(Cadsus::Client).not_to receive(:for)
    lookup
    expect([ response.status, json ]).to eq([ 403, { "error" => "feature_disabled", "feature" => "cadsus_lookup" } ])
  end

  it "ligado sem credencial ou com credencial recusada: 503 cadsus_unavailable; a recusa marca a credencial" do
    switch_on!
    sign_in_as(verifier)
    lookup
    expect([ response.status, json ]).to eq([ 503, { "error" => "cadsus_unavailable" } ])
    credential!(Cadsus::Simulated::REFUSED_USERNAME)
    lookup
    expect(response).to have_http_status(:service_unavailable)
    expect(IntegrationCredential.find_by!(kind: "cadsus").last_check_status).to eq("unauthorized")
  end

  it "achado: CNS mascarado, comparação nula sem perfil declarado, nada de CPF; pendente gravado, CNS ainda não" do
    switch_on!
    credential!
    sign_in_as(verifier)
    lookup
    expect(response).to have_http_status(:ok)
    expect(json.keys).to match_array(%w[found cns_masked birth_date_matches sex_matches])
    expect(json).to include("found" => true, "birth_date_matches" => nil, "sex_matches" => nil)
    expect(json["cns_masked"]).to match(/\A\*\*\* \*\*\*\* \*\*\*\* \d{4}\z/)
    expect(response.body).not_to include("52998224725")
    citizen.reload
    expect(citizen.cns).to be_nil
    expect(citizen.cadsus_pending_cns).to be_present
    expect(DomainEvent.where(name: "citizen.cadsus_looked_up").map(&:payload))
      .to eq([ { "citizen_id" => citizen.id, "user_id" => verifier.id, "found" => true } ])
  end

  it "não achado e fora do ar; código errado segue os erros do lookup" do
    switch_on!
    credential!
    sign_in_as(verifier)
    lookup(Citizen.create!(cpf: Cadsus::Simulated::NOT_FOUND_CPF, phone: "+5541998765433"))
    expect(json).to eq("found" => false, "cns_masked" => nil, "birth_date_matches" => nil, "sex_matches" => nil)
    lookup(Citizen.create!(cpf: Cadsus::Simulated::UNAVAILABLE_CPF, phone: "+5541998765434"))
    expect([ response.status, json ]).to eq([ 503, { "error" => "cadsus_unavailable" } ])
    issue_code_for(citizen)
    json_post "/attendance/cadsus_lookup", cpf: citizen.cpf, code: "000000"
    expect([ response.status, json ]).to eq([ 422, { "error" => "invalid_code" } ])
  end

  it "validação com cadsus_confirmed grava CNS e a marca, e limpa o pendente" do
    switch_on!
    credential!
    sign_in_as(verifier)
    code = lookup
    pending = citizen.reload.cadsus_pending_cns
    json_post "/attendance/verifications", cpf: citizen.cpf, code: code, document_checked: true, cadsus_confirmed: true
    expect(response).to have_http_status(:created)
    citizen.reload
    expect(citizen.cns).to eq(pending)
    expect(citizen.cadsus_checked_at).to be_present
    expect([ citizen.cadsus_pending_cns, citizen.cadsus_pending_session_id, citizen.cadsus_pending_at ]).to eq([ nil, nil, nil ])
  end

  # Review Focus 5.
  it "consulta vencida ou de outra sessão: 409, nada validado, código intacto" do
    switch_on!
    credential!
    sign_in_as(verifier)
    lookup
    travel 12.minutes do
      # O código da consulta venceu (TTL de 10 min); o cidadão gera outro e
      # volta ao balcão — a consulta ao CADSUS já não vale.
      fresh = issue_code_for(citizen)
      json_post "/attendance/verifications", cpf: citizen.cpf, code: fresh, document_checked: true, cadsus_confirmed: true
      expect([ response.status, json ]).to eq([ 409, { "error" => "cadsus_lookup_missing" } ])
    end
    expect(citizen.reload).to be_verification_level_declared

    code = issue_code_for(citizen)

    lookup(citizen, code)
    sign_in_as(verifier)
    json_post "/attendance/verifications", cpf: citizen.cpf, code: code, document_checked: true, cadsus_confirmed: true
    expect([ response.status, json ]).to eq([ 409, { "error" => "cadsus_lookup_missing" } ])

    json_post "/attendance/verifications", cpf: citizen.cpf, code: code, document_checked: true
    expect(response).to have_http_status(:created)
    expect(citizen.reload.cns).to be_nil
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec spec/requests/attendance_cadsus_spec.rb`
Expected: FAIL (`No route matches [POST] "/attendance/cadsus_lookup"`).

- [ ] **Step 3: Guarda de interruptor e command**

```ruby
# app/controllers/concerns/feature_gate.rb
# Rota de funcionalidade com interruptor (ADR 0028; contratos §1): desligado na
# cidade do host → 403 feature_disabled. "Ligado mas sem o que precisa" é da
# rota (o CADSUS responde 503; a Produção do exportador decide a sua).
module FeatureGate
  extend ActiveSupport::Concern

  class_methods do
    def require_feature(key, **options)
      before_action(-> { require_feature!(key) }, **options)
    end
  end

  private

  def require_feature!(key)
    return if Platform::Features.enabled?(Current.city, key)

    render json: { error: "feature_disabled", feature: key.to_s }, status: :forbidden
  end
end
```

```ruby
# app/commands/cadsus/lookup.rb
# Consulta ao CADSUS no balcão (ADR 0028; spec 2026-10-05 §7; contratos §5.4;
# desvios 6 e 8). Exige o interruptor utilizável. Compara nascimento e sexo em
# memória com o perfil declarado (null sem perfil) e guarda só o CNS como
# PENDENTE desta sessão — quem o efetiva é Citizens::Verify com
# cadsus_confirmed. Credencial recusada marca a credencial (o interruptor deixa
# de ser utilizável e a tela de Integrações mostra). Reasons: :cadsus_unavailable.
module Cadsus
  module Lookup
    WINDOW = 10.minutes

    module_function

    def call(citizen:, by:, session:, city:)
      return Result.fail(:cadsus_unavailable) unless Platform::Features.usable?(city, "cadsus_lookup")

      record = Client.for(city).lookup(citizen.cpf)
      ApplicationRecord.transaction do
        citizen.update!(cadsus_pending_cns: record&.cns, cadsus_pending_session_id: record&.cns && session.id,
                        cadsus_pending_at: record&.cns && Time.current)
        DomainEvents.publish("citizen.cadsus_looked_up", citizen_id: citizen.id, user_id: by.id, found: !record.nil?)
      end
      Result.ok(found: !record.nil?, cns_masked: Professionals::Cns.mask(record&.cns),
                birth_date_matches: compare(citizen.try(:birth_date), record&.birth_date),
                sex_matches: compare(citizen.try(:sex), record&.sex))
    rescue Cadsus::Unauthorized
      IntegrationCredential.where(kind: "cadsus").update_all(last_check_at: Time.current, last_check_status: "unauthorized",
                                                             last_check_message: "O CADSUS recusou a credencial")
      Result.fail(:cadsus_unavailable)
    rescue Cadsus::Error
      Result.fail(:cadsus_unavailable)
    end

    def compare(declared, found)
      return nil if declared.nil? || found.nil?

      declared.to_s == found.to_s
    end
  end
end
```

- [ ] **Step 4: Rota, controller e `Citizens::Verify`**

Em `config/routes.rb`, no `scope "/attendance"`, depois de `post "lookup", ...`:

```ruby
    # CADSUS no balcão (ADR 0028; contratos §5.4), atrás do interruptor.
    post "cadsus_lookup",            to: "attendance#cadsus_lookup"
```

Em `app/controllers/attendance_controller.rb`:
- no cabeçalho: `#   POST /attendance/cadsus_lookup              {cpf, code}                    citizen_verifier + cadsus_lookup`;
- `include FeatureGate` depois de `include AttendanceAccess`;
- `ERROR_STATUS` ganha `cadsus_lookup_missing: :conflict`;
- `before_action :require_verifier, only: %i[lookup verify cadsus_lookup]` e, logo abaixo, `require_feature "cadsus_lookup", only: :cadsus_lookup`;
- o `rate_limit` passa a `only: %i[lookup verify cadsus_lookup]`;
- `verify` passa a:

```ruby
  def verify
    result = Citizens::Verify.call(cpf: params[:cpf], code: params[:code],
                                   document_checked: params[:document_checked] == true, by: Current.user,
                                   cadsus_confirmed: params[:cadsus_confirmed] == true, session_id: Current.session.id)
    return render_failure(result, ERROR_STATUS) if result.failure?

    v = result.payload[:verification]
    render json: { verification: { id: v.id, citizen_id: v.citizen_id, verified_at: v.verified_at.iso8601 } },
           status: :created
  end

  # ADR 0028 (contratos §5.4): o par sai do mesmo CPF + código do lookup, sem
  # consumir o código; a resposta nunca traz nome, mãe, endereço nem CPF.
  def cadsus_lookup
    match = Citizens::VerificationCodeMatch.call(cpf: params[:cpf], code: params[:code])
    return render_failure(match, ERROR_STATUS) if match.failure?

    result = Cadsus::Lookup.call(citizen: match.payload[:citizen], by: Current.user, session: Current.session,
                                 city: Current.city)
    return render(json: { error: "cadsus_unavailable" }, status: :service_unavailable) if result.failure?

    render json: result.payload
  end
```

Em `app/commands/citizens/verify.rb`:
- a assinatura passa a `def self.call(cpf:, code:, document_checked:, by:, cadsus_confirmed: false, session_id: nil)`;
- depois do bloco `if (active = citizen.active_verification) ... end`:

```ruby
        # ADR 0028 (contratos §5.4): confirmar o CADSUS exige consulta desta
        # sessão há no máximo 10 min — antes de consumir o código.
        if cadsus_confirmed && !cadsus_pending?(citizen, session_id)
          next result = Result.fail(:cadsus_lookup_missing)
        end
```

- depois de `verification = record!(citizen: citizen, by: by)`: `confirm_cadsus!(citizen) if cadsus_confirmed`;
- novos métodos de classe, depois de `record!`:

```ruby
    def self.cadsus_pending?(citizen, session_id)
      citizen.cadsus_pending_cns.present? && session_id.present? && citizen.cadsus_pending_session_id == session_id &&
        citizen.cadsus_pending_at.present? && citizen.cadsus_pending_at >= Cadsus::Lookup::WINDOW.ago
    end

    # Do CADSUS só ficam o CNS e a marca da conferência (ADR 0028).
    def self.confirm_cadsus!(citizen)
      citizen.update!(cns: citizen.cadsus_pending_cns, cadsus_checked_at: Time.current, cadsus_pending_cns: nil,
                      cadsus_pending_session_id: nil, cadsus_pending_at: nil)
    end
```

- no comentário do topo, acrescente: `# ADR 0028: com cadsus_confirmed, efetiva o CNS da consulta ao CADSUS desta sessão (Reasons: :cadsus_lookup_missing).`

- [ ] **Step 5: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec spec/requests/attendance_cadsus_spec.rb spec/requests/attendance_spec.rb spec/requests/attendance_contract_spec.rb spec/commands/citizens`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod16 add app/controllers/concerns/feature_gate.rb app/commands/cadsus/lookup.rb app/controllers/attendance_controller.rb app/commands/citizens/verify.rb config/routes.rb spec/requests/attendance_cadsus_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod16 commit -m "feat: look up CADSUS at the counter and confirm the CNS on verification

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Fatia 6 — LGPD, invariantes, semente e fechamento

### Task 20: Exclusão limpa o CADSUS e as invariantes do ADR 0028

**Files:**
- Modify: `app/commands/citizens/erase.rb`, `spec/commands/citizens/erase_spec.rb`
- Create: `spec/invariants/record_mode_invariants_spec.rb`

**Interfaces:**
- Consumes: tudo das Tasks 1–19.
- Produces: `spec/invariants/record_mode_invariants_spec.rb` com um bloco por invariante do ADR 0028 que não depende do exportador; o plano do exportador acrescenta os dele (ficha não sai com interruptor desligado ou `record_mode = off`; `payload` de ficha aceita não existe mais) **no mesmo arquivo**.

- [ ] **Step 1: Escreva as specs que falham**

No fim de `spec/commands/citizens/erase_spec.rb` (antes do `end` final):

```ruby
  # ADR 0028 (spec 2026-10-05 §8): a casca não guarda o CNS do CADSUS, a marca
  # da conferência nem a consulta pendente.
  it "apaga CNS, marca do CADSUS e pendente" do
    pair.update!(cns: "700000000000005", cadsus_checked_at: Time.current, cadsus_pending_cns: "700000000000005",
                 cadsus_pending_session_id: SecureRandom.uuid, cadsus_pending_at: Time.current)

    described_class.call(request: request, by: admin)

    row = ApplicationRecord.connection.select_one(
      ApplicationRecord.sanitize_sql([ "SELECT cns, cadsus_checked_at, cadsus_pending_cns, cadsus_pending_session_id, " \
                                       "cadsus_pending_at FROM citizens WHERE id = ?", pair.id ])
    )
    expect(row.values).to all(be_nil)
  end
```

```ruby
# spec/invariants/record_mode_invariants_spec.rb
require "rails_helper"
require "webmock/rspec"
require Rails.root.join("lib/sigtap_sample").to_s

# Módulo 16, critério de fechamento (ADR 0028, "Invariantes"). Cada bloco tem a
# mutação que precisa deixá-lo vermelho. Os invariantes da fila e do envio
# (ficha não sai desligada; payload aceito apagado) entram aqui pelo plano do
# exportador.
RSpec.describe "Invariantes do modo de prontuário (ADR 0028)", type: :request do
  before { Current.city = TEST_CITY_A; Rails.cache.clear }
  after { Current.reset }

  let(:city) { City.find_by!(slug: TEST_CITY_A.slug) }
  let(:admin) do
    staff_with("admin@cidade.gov.br", "municipal_admin").tap do |u|
      Mfa::Enroll.call(u)
      u.update!(otp_enabled: true)
    end
  end
  let(:verifier) { staff_with("atendente@cidade.gov.br", "citizen_verifier") }
  let(:maintainer) do
    Maintainer.create!(email_address: "inv-#{SecureRandom.hex(3)}@rotasaude.app", password: "s3nha-forte-1",
                       otp_secret: ROTP::Base32.random, otp_enabled_at: Time.current)
  end
  let(:marker) { "senha-marcador-#{SecureRandom.hex(4)}" }
  def json = JSON.parse(response.body)
  def sql(statement) = PlatformRecord.transaction(requires_new: true) { PlatformRecord.connection.execute(statement) }

  # Mutação: tirar `require_feature "cadsus_lookup"` do AttendanceController.
  it "interruptor desligado: a rota da funcionalidade responde 403 e o serviço não é chamado" do
    IntegrationCredential.create!(kind: "cadsus", secret: { "username" => "u", "password" => "p" }, set_by_user: admin,
                                  set_at: Time.current)
    Platform::Features.set!(city: city, key: "cadsus_lookup", enabled: true, maintainer: maintainer)
    Platform::Features.set!(city: city, key: "cadsus_lookup", enabled: false, maintainer: maintainer)
    citizen = Citizen.create!(cpf: "52998224725", phone: "+5541998765432")
    expect(Cadsus::Client).not_to receive(:for)
    sign_in_as(verifier)
    json_post "/attendance/cadsus_lookup", cpf: citizen.cpf, code: issue_code_for(citizen)
    expect([ response.status, json["error"] ]).to eq([ 403, "feature_disabled" ])
  end

  # Mutação: devolver `secret` em credential_json, ou tirar :passw do filtro.
  it "nenhuma resposta, evento, auditoria, fila ou log carrega a credencial" do
    city.update!(pec_url: "https://pec.cidade.gov.br")
    stub_request(:post, "https://pec.cidade.gov.br/api/recebimento/login").to_return(status: 500, body: marker)
    sign_in_as(admin).update!(mfa_verified_at: Time.current)
    bodies = []
    put "/integrations/credentials/ledi", params: { username: "rota", password: marker }, as: :json
    bodies << response.body
    post "/integrations/credentials/ledi/check"
    bodies << response.body
    get "/integrations"
    bodies << response.body

    expect(bodies.join).not_to include(marker)
    expect(DomainEvent.pluck(:payload).to_json).not_to include(marker)
    expect(PlatformEvent.pluck(:payload).to_json).not_to include(marker)
    expect(ActiveJob::Base.queue_adapter.enqueued_jobs.to_json).not_to include(marker)
    expect(IntegrationCredential.find_by!(kind: "ledi").last_check_message).not_to include(marker)
    filtered = ActiveSupport::ParameterFilter.new(Rails.application.config.filter_parameters)
                                             .filter("password" => marker, "secret" => marker, "cpf" => marker, "cns" => marker)
    expect(filtered.values).to all(eq("[FILTERED]"))
  end

  # Mutação: CheckConnection ler a credencial ou o PEC de outra cidade (ex.: sem
  # CityConnection, ou Platform::Features.settings de outra City).
  it "a cidade A nunca usa a credencial nem o PEC da cidade B" do
    city.update!(pec_url: "https://pec-a.cidade.gov.br")
    city_b = City.find_by(slug: TEST_CITY_B.slug) ||
             City.create!(slug: TEST_CITY_B.slug, name: TEST_CITY_B.name, status: "active",
                          database_url: TEST_CITY_B.database_url, encryption_key: TEST_CITY_B.encryption_key,
                          schema_version: CitySchema.expected_version.to_s)
    city_b.update!(pec_url: "https://pec-b.cidade.gov.br")
    within_city(city_b) do
      user_b = User.create!(email_address: "admin-b@cidade.gov.br", password: "senha-segura-123")
      IntegrationCredential.create!(kind: "ledi", secret: { "username" => "b", "password" => "senha-de-b" },
                                    set_by_user: user_b, set_at: Time.current)
      IntegrationCredential.create!(kind: "cadsus", secret: { "username" => "b", "password" => "senha-de-b" },
                                    set_by_user: user_b, set_at: Time.current)
    end
    stub_request(:post, %r{\Ahttps://pec-[ab]\.cidade\.gov\.br/}).to_return(status: 200, headers: { "Set-Cookie" => "JSESSIONID=z" })

    expect(Integrations::CheckConnection.call(kind: "ledi", city: city).reason).to eq(:credential_missing)
    expect { Cadsus::Client.for(city) }.to raise_error(Cadsus::Unavailable)

    IntegrationCredential.create!(kind: "ledi", secret: { "username" => "a", "password" => "senha-de-a" }, set_by_user: admin,
                                  set_at: Time.current)
    Integrations::CheckConnection.call(kind: "ledi", city: city)
    within_city(city_b) { Integrations::CheckConnection.call(kind: "ledi", city: city_b) }

    expect(a_request(:post, "https://pec-a.cidade.gov.br/api/recebimento/login").with(body: /senha-de-a/)).to have_been_made.once
    expect(a_request(:post, "https://pec-b.cidade.gov.br/api/recebimento/login").with(body: /senha-de-b/)).to have_been_made.once
    expect(a_request(:post, %r{pec-a}).with(body: /senha-de-b/)).not_to have_been_made
    expect(a_request(:post, %r{pec-b}).with(body: /senha-de-a/)).not_to have_been_made
  end

  # Mutação: tirar terminology_releases_guard de db/platform_triggers.sql.
  it "release de terminologia ativa nunca muda; release com falha nunca fica ativa" do
    dir = SigtapSample.write_to(Dir.mktmpdir, competence: "202610")
    active = Terminology::Import.call(kind: "sigtap", version: "202610", path: dir).payload[:release]
    expect { sql("UPDATE terminology_releases SET source_sha256 = 'x' WHERE id = '#{active.id}'") }
      .to raise_error(ActiveRecord::StatementInvalid)
    expect { sql("UPDATE sigtap_procedures SET name = 'x' WHERE release_id = '#{active.id}'") }
      .to raise_error(ActiveRecord::StatementInvalid)
    dir.join("tb_procedimento.txt").binwrite("lixo\r\n")
    Terminology::Import.call(kind: "sigtap", version: "202611", path: dir)
    failed = TerminologyRelease.find_by!(version: "202611")
    expect(failed.status).to eq("failed")
    expect { sql("UPDATE terminology_releases SET status = 'active' WHERE id = '#{failed.id}'") }
      .to raise_error(ActiveRecord::StatementInvalid)
  ensure
    FileUtils.remove_entry(dir) if dir
  end

  # Mutação: Cnes::Import ou GET /cnes chamar Cnes::Apply.
  it "nenhum casamento do CNES é aplicado sem confirmação da cidade" do
    cnes_city!
    unit = create_unit("UBS Jardim das Flores")
    Cnes::Import.call(competence: "202609", path: Rails.root.join("spec/fixtures/cnes/202609"))
    sign_in_as(admin)
    get "/cnes"
    expect(json["proposals"]).not_to be_empty
    expect(unit.reload.cnes).to be_nil
    expect([ HealthTeam.count, HealthTeamMember.count ]).to eq([ 0, 0 ])
  end

  # Mutação: gravar birth_date/sex do CADSUS, ou o CNS antes da confirmação.
  it "do CADSUS só ficam o CNS e a marca da conferência" do
    IntegrationCredential.create!(kind: "cadsus", secret: { "username" => "u", "password" => "p" }, set_by_user: admin,
                                  set_at: Time.current)
    Platform::Features.set!(city: city, key: "cadsus_lookup", enabled: true, maintainer: maintainer)
    citizen = Citizen.create!(cpf: "52998224725", phone: "+5541998765432")
    found = Cadsus::Simulated.new(username: "u").lookup(citizen.cpf)
    sign_in_as(verifier)
    code = issue_code_for(citizen)
    json_post "/attendance/cadsus_lookup", cpf: citizen.cpf, code: code
    json_post "/attendance/verifications", cpf: citizen.cpf, code: code, document_checked: true, cadsus_confirmed: true

    citizen.reload
    expect(citizen.cns).to eq(found.cns)
    expect(citizen.cadsus_checked_at).to be_present
    expect(Citizen.column_names.grep(/cadsus|cns/)).to match_array(%w[cns cadsus_checked_at cadsus_pending_cns
                                                                      cadsus_pending_session_id cadsus_pending_at])
    raw = ApplicationRecord.connection.select_one(ApplicationRecord.sanitize_sql([ "SELECT * FROM citizens WHERE id = ?", citizen.id ]))
    expect(raw.values.compact.map(&:to_s).join("|")).not_to include(found.birth_date.iso8601, found.birth_date.strftime("%Y%m%d"))
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec spec/commands/citizens/erase_spec.rb spec/invariants/record_mode_invariants_spec.rb`
Expected: FAIL só no exemplo novo do Erase (os invariantes já valem pelas Tasks 1–19; se algum falhar, a falha é de uma task anterior — volte a ela).

- [ ] **Step 3: Implemente**

Em `app/commands/citizens/erase.rb`, `erase_pair`: o `citizen.update_columns(...)` final passa a

```ruby
      # ADR 0028: o CNS do CADSUS, a marca e a consulta pendente saem junto.
      citizen.update_columns(cpf: tombstone, phone: tombstone, neighborhood_id: nil, erased_at: Time.current,
                             cns: nil, cadsus_checked_at: nil, cadsus_pending_cns: nil, cadsus_pending_session_id: nil,
                             cadsus_pending_at: nil, updated_at: Time.current)
```

e o comentário do topo ganha `# ADR 0028: também o CNS do CADSUS e a consulta pendente.`

- [ ] **Step 4: Prove as mutações**

Para cada bloco de `record_mode_invariants_spec.rb`, aplique a mutação do comentário, rode o arquivo e veja o bloco **ficar vermelho**; desfaça. Registre no relatório da task as seis mutações e o exemplo que caiu em cada.

Run: `docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec spec/invariants/record_mode_invariants_spec.rb spec/commands/citizens/erase_spec.rb`
Expected: PASS (sem mutação).

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod16 add app/commands/citizens/erase.rb spec/commands/citizens/erase_spec.rb spec/invariants/record_mode_invariants_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod16 commit -m "test: pin ADR 0028 invariants and clear CADSUS data on erasure

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 21: Semente de dev (spec §10, sem o exportador)

**Files:**
- Create: `lib/record_mode_crew.rb`
- Modify: `db/seeds.rb`
- Test: `spec/lib/record_mode_crew_spec.rb`

**Interfaces:**
- Consumes: `SigtapSample` (Task 12), `Terminology::Import` (Task 11), `Cnes::SnapshotWriter` (Task 14), `ProfessionalCrew` (existente), `IntegrationCredential` (Task 2).
- Produces: `RecordModeCrew.seed_platform! -> { sigtap:, imported: }`; `RecordModeCrew.seed_current_city(slug:, admin:) -> { ibge_code:, cnes_competence:, establishments:, teams:, bonds:, proposals: }` (dentro de `CityConnection.with`, depois do `ProfessionalCrew`); `RecordModeCrew.cpf_for(seed) -> String` (CPF de 11 dígitos com verificador válido).

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/lib/record_mode_crew_spec.rb
require "rails_helper"
require Rails.root.join("lib/signature_crew")
require Rails.root.join("lib/professional_crew")
require Rails.root.join("lib/record_mode_crew")

# Spec 2026-10-05 §10: IBGE real no city_profile, modo off, SIGTAP reduzida
# ativa, retrato do CNES fictício coerente com unidades e profissionais da
# semente, credencial cadsus simulada. Idempotente.
RSpec.describe RecordModeCrew do
  before do
    Current.city = TEST_CITY_A
    (CityProfile.current || CityProfile.new).update!(name: "Cidade Teste", uf: "PR", ibge_code: "4106902")
    SignatureCrew.seed_current_city(slug: "teste", password: "dev-password")
    ProfessionalCrew.seed_current_city(slug: "teste", password: "dev-password", admin: admin)
  end
  after { Current.reset }

  let(:admin) { staff_with("admin@teste.demo", "municipal_admin") }

  it "semeia SIGTAP da competência corrente, CPFs, credencial simulada e um retrato que gera propostas" do
    2.times do
      described_class.seed_platform!
      described_class.seed_current_city(slug: "teste", admin: admin)
    end

    competence = Time.zone.today.strftime("%Y%m")
    expect(TerminologyRelease.active.where(kind: "sigtap", version: competence).count).to eq(1)
    expect(Professional.all).to all(satisfy { |p| CitizenIdentity::Cpf.normalize(p.cpf) == p.cpf })
    expect(IntegrationCredential.find_by!(kind: "cadsus").username).to eq("simulado")
    expect(CnesSnapshot.where(ibge_code: "4106902").count).to eq(1)

    proposals = Cnes::Proposal.for(TEST_CITY_A)[:proposals]
    expect(proposals.select { |p| p[:kind] == "unit" && p[:action] == "link" }.map { |p| p[:local][:name] })
      .to contain_exactly("UBS Jardim das Flores", "UBS Vila Esperança", "UPA 24h Centro")
    expect(HealthUnit.where.not(cnes: nil)).to be_empty
    expect(City.find_by(slug: TEST_CITY_A.slug)&.record_mode).to be_in([ nil, "off" ])
  end

  it "cpf_for gera CPF válido e estável" do
    expect(described_class.cpf_for("x")).to eq(described_class.cpf_for("x"))
    expect(CitizenIdentity::Cpf.normalize(described_class.cpf_for("x"))).to eq(described_class.cpf_for("x"))
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec spec/lib/record_mode_crew_spec.rb`
Expected: FAIL (`cannot load such file -- .../lib/record_mode_crew`).

- [ ] **Step 3: Implemente**

```ruby
# lib/record_mode_crew.rb
require "digest"
require "zlib"
require "tmpdir"
require Rails.root.join("lib/sigtap_sample").to_s

# Semente de dev do módulo 16 (spec 2026-10-05 §10), sem o exportador. Dev é
# fictício mas imita o real: IBGE real (já no city_profile), modo off, SIGTAP
# reduzida ativa na competência corrente, CPF com verificador válido nos
# profissionais, credencial cadsus "simulado" (o backend de dev é simulado) e
# um retrato do CNES da competência anterior, coerente com as unidades e
# profissionais do ProfessionalCrew: as unidades ainda SEM CNES, para a tela
# mostrar propostas a confirmar; uma divergência de CBO de propósito.
# Interruptores ficam desligados: quem liga é o mantenedor. Idempotente.
class RecordModeCrew
  class << self
    def seed_platform!
      competence = Time.zone.today.strftime("%Y%m")
      return { sigtap: competence, imported: false } if TerminologyRelease.active.exists?(kind: "sigtap", version: competence)

      Dir.mktmpdir do |dir|
        result = Terminology::Import.call(kind: "sigtap", version: competence,
                                          path: SigtapSample.write_to(dir, competence: competence), by: "db:seed")
        raise "semente: SIGTAP recusada (#{result.reason} #{result.message})" if result.failure?
      end
      { sigtap: competence, imported: true }
    end

    def seed_current_city(slug:, admin:)
      ibge = CityProfile.current&.ibge_code
      raise "semente: cidade sem IBGE no city_profile" if ibge.blank?

      pros = ProfessionalCrew::PROFILES.keys.to_h do |prefix|
        [ prefix, User.find_by!(email_address: "#{prefix}@#{slug}.demo").professional ]
      end
      pros.each { |prefix, p| p.update!(cpf: cpf_for("#{slug}:#{prefix}")) if p.cpf.nil? }

      IntegrationCredential.find_or_create_by!(kind: "cadsus") do |c|
        c.secret = { "username" => "simulado", "password" => "simulado" }
        c.set_by_user = admin
        c.set_at = Time.current
      end

      competence = Time.zone.today.prev_month.strftime("%Y%m")
      cnes = ProfessionalCrew::UNITS.to_h { |u| [ u[:name], cnes_for(slug, u[:name]) ] }
      esf = ine_for(slug, "ESF")
      eap = ine_for(slug, "EAP")
      snapshot = Cnes::SnapshotWriter.write!(
        competence: competence, ibge_code: ibge,
        establishments: ProfessionalCrew::UNITS.map { |u| { cnes: cnes[u[:name]], name: u[:name].upcase, unit_type: u[:kind] == "upa" ? "73" : "02" } } +
                        [ { cnes: cnes_for(slug, "HOSPITAL"), name: "HOSPITAL MUNICIPAL", unit_type: "05" } ],
        teams: [ { ine: esf, kind: "70", cnes: cnes["UBS Jardim das Flores"], name: "ESF JARDIM DAS FLORES", active: true },
                 { ine: eap, kind: "76", cnes: cnes["UBS Vila Esperança"], name: "EAP VILA ESPERANCA", active: true } ],
        bonds: [
          bond(pros["profissional"], cnes["UBS Jardim das Flores"], esf, "225125"),
          bond(pros["profissional"], cnes["UPA 24h Centro"], nil, "225124"),
          bond(pros["enfermeira"], cnes["UBS Jardim das Flores"], esf, "223505"),
          # Divergência de propósito: o CNES diz outro CBO para o técnico.
          bond(pros["tecnico"], cnes["UPA 24h Centro"], nil, "322245")
        ]
      )
      { ibge_code: ibge, cnes_competence: competence, establishments: snapshot.establishments.count,
        teams: snapshot.teams.count, bonds: snapshot.bonds.count }
    end

    # CPF fictício com dígito verificador válido, estável pela semente.
    def cpf_for(seed)
      (0..).each do |attempt|
        base = Digest::SHA256.hexdigest("#{seed}:#{attempt}").scan(/\d/).join[0, 9].ljust(9, "1")
        next if base.chars.uniq.size == 1

        nums = base.chars.map(&:to_i)
        first = CitizenIdentity::Cpf.check_digit(nums)
        return base + first.to_s + CitizenIdentity::Cpf.check_digit(nums + [ first ]).to_s
      end
    end

    private

    def bond(professional, cnes, ine, cbo)
      { cnes: cnes, ine: ine, cbo_code: cbo, cpf: professional.cpf, cns: professional.cns }
    end

    def cnes_for(slug, name) = (Zlib.crc32("cnes:#{slug}:#{name}") % 9_000_000 + 1_000_000).to_s
    def ine_for(slug, name) = Zlib.crc32("ine:#{slug}:#{name}").to_s.rjust(10, "0")
  end
end
```

Em `db/seeds.rb`:
- no cabeçalho, depois da linha do Analytics: `#     Modo de prontuário (módulo 16): SIGTAP reduzida, CPF dos profissionais, credencial cadsus simulada e um retrato do CNES — ver lib/record_mode_crew.rb. Interruptores e record_mode ficam desligados.`;
- junto dos `require`: `require Rails.root.join("lib/record_mode_crew").to_s`;
- antes de `protocol_defn = ...`:

```ruby
  # ── Terminologias (módulo 16, spec 2026-10-05 §10) ──────────────────────────
  sigtap = RecordModeCrew.seed_platform!
  puts "[seeds] SIGTAP ....... #{sigtap[:sigtap]} #{sigtap[:imported] ? 'importada (recorte de dev)' : 'já ativa'}"
```

- depois do bloco do Analytics (`puts "[seeds] analytics .. ..."`):

```ruby
        # ── Modo de prontuário (módulo 16, spec 2026-10-05 §10) ───────────────
        # Depois do ProfessionalCrew: o retrato do CNES usa as unidades e os
        # profissionais dele. record_mode fica off e nenhum interruptor ligado.
        record = RecordModeCrew.seed_current_city(slug: slug, admin: muni_admin)
        puts "[seeds] CNES ......... #{record[:ibge_code]} #{record[:cnes_competence]}: #{record[:establishments]} " \
             "estabelecimentos, #{record[:teams]} equipes, #{record[:bonds]} vínculos; credencial cadsus simulada"
```

- [ ] **Step 4: Rode e veja passar; rode a semente no dev**

Run: `docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec spec/lib/record_mode_crew_spec.rb spec/lib/professional_crew_spec.rb`
Expected: PASS.

Run: `docker compose exec -T -w /rails/.claude/mod16 api bin/rails db:seed`
Expected: as linhas `[seeds] SIGTAP ...` e `[seeds] CNES ......... 4106902 ...` (e 4115200 para Maringá); rodar de novo não duplica nada.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod16 add lib/record_mode_crew.rb db/seeds.rb spec/lib/record_mode_crew_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod16 commit -m "chore: seed dev SIGTAP sample, professional CPFs, CADSUS credential and a CNES snapshot

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 22: Verificação final

**Files:** nenhum novo.

- [ ] **Step 1: Carregamento e estilo**

```bash
docker compose exec -T -w /rails/.claude/mod16 api bin/rails zeitwerk:check
docker compose exec -T -w /rails/.claude/mod16 api bundle exec rubocop app lib config db/platform_migrate db/city_migrate spec
```
Expected: `All is good!` e nenhuma ofensa nova (corrija as que vierem dos arquivos deste plano).

- [ ] **Step 2: Suíte completa com o worker parado**

```bash
docker compose stop worker
docker compose exec -T -w /rails/.claude/mod16 api bundle exec rspec
docker compose start worker
```
Expected: 0 falhas. Uma falha em `spec/services/city_schema_spec.rb` na primeira rodada depois de voltar de outra branch é o banco de teste à frente/atrás: DROP + `city:test_databases` e rode de novo.

- [ ] **Step 3: Conferências manuais curtas**

- `grep -rn "Current.city\s*=" app lib` não traz arquivo novo.
- `grep -rn "password\|secret" app/controllers/integrations_controller.rb` só mostra a leitura do corpo, nunca a escrita na resposta.
- `bin/rails routes | grep -E "integrations|cnes|cadsus_lookup|record_settings"` lista as seis rotas novas (e `record_settings` só com host de console).

- [ ] **Step 4: Fechamento**

Use superpowers:finishing-a-development-branch. Push só com autorização do usuário; antes, `origin/main..main` (sessões paralelas). Ordem de rollout (spec §11): `contracts` `session-v1.1.0` → este api (`bin/migrate`: plataforma, `platform:triggers`, `city:migrate:all`; tudo nasce `off`) → `maintenance`/`admin`/`dashboard`. Imagem de dev: `docker compose build api` (rubyzip).

---

## Self-review (feito)

- **Cobertura da spec (escopo deste plano):** §3.1 → Tasks 1, 3, 4, 5, 19 (`FeatureGate`); §3.2 → Tasks 1, 6 (IBGE no `city_profile`, decisão da sessão principal); §3.3 → Tasks 2, 7, 8, 9; §4 → Tasks 10–13 (alerta: Task 13; publicação no console é do exportador); §5 → Tasks 2, 14–18; §7 → Tasks 8, 19; §8 → Tasks 2 (cifra, filtro), 9, 14, 19, 20; invariantes do ADR sem exportador → Task 20; §10 sem exportador → Task 21; `VALID_RANGE` → Task 1; R18/`MaintenanceAudit`/`DomainEvents` → Tasks 3, 4, 9. Fora: §6 inteiro e `GET /city_production`/`GET /production` (plano do exportador).
- **Placeholders:** nenhum "TBD"; o único esqueleto (leitor da SIGTAP na Task 11) é substituído na Task 12 e está escrito por inteiro.
- **Consistência de nomes:** `Platform::Features.{find,find!,enabled?,enabled_keys,settings,city_state,missing,usable?,summary,set!}`, `Cnes::{SnapshotWriter,BaseLayout,BaseReader,Import,Proposal,Apply}`, `Cadsus::{Client,Simulated,SoapPdq,Lookup,Record}`, `Ledi::PecClient#login`, `Integrations::{SetCredential,CheckConnection}`, `UpdateCityRecordSettings`, `IntegrationPolicy#manage?`, `FeatureGate#require_feature` — conferidos entre as tasks que os produzem e consomem.
- **Review Focus:** as cinco linhas têm teste na task dona (6, 7/9, 14/15, 18, 19).

---

## Divergências propostas ao contrato

1. **§5.4 — erros de código no `POST /attendance/cadsus_lookup`**: o par sai de `VerificationCodeMatch` (como `/attendance/lookup`), então as falhas são as mesmas: 422 `invalid_cpf`, `invalid_code`, `code_expired`, `code_exhausted`. O `404 not_found` listado não tem caso que o produza; proposta: trocar por esses quatro códigos. "CADSUS não achou" continua 200 `found: false`.
2. **§5.4 — `cadsus_confirmed: true` sem consulta válida**: 409 `{ "error": "cadsus_lookup_missing" }` (consulta ausente, de outra sessão, ou há mais de 10 min); nada é validado e o código não é consumido. Proposta: acrescentar ao contrato.
3. **§3 — erros de `setCityFeature`**: `unknown_feature` sai como `UserError { path: "key", message: "unknown_feature" }`; cidade inexistente sai como em toda mutation de cidade, `{ path: "citySlug", message: "cidade inexistente" }` (e cidade não ativa, `"cidade não está ativa (<status>)"`). Proposta: o contrato citar `path` em vez de um código `unknown_city`.
4. **§3 — auditoria do maintenance**: além de `city.feature_changed`, a mutation grava o par tentativa/resultado `maintenance.city.feature_changed` (padrão de toda mutation da API de manutenção). Proposta: acrescentar à tabela §6.
5. **§5.2 — `POST /cnes/apply` pula também por `conflict`** (`skipped: [{ id, reason: "conflict" }]`): proposta ainda válida que esbarra numa regra do banco (nome de unidade repetido ao criar, membro já ativo). Proposta: `reason` ∈ `stale`, `conflict`. E `422 { "error": "invalid_proposals" }` para lista vazia, com mais de 500 ids ou id fora do formato (24 hex).
6. **Respostas existentes ganham campos** (não estão no contrato e as telas do dashboard vão usar): `unit_json` de `/attendance/units*` ganha `cnes`; `POST /attendance/units[/:id]` aceita `cnes` (422 `invalid_cnes`, `cnes_taken`); `profile_json` de `/professionals*` ganha `cpf_masked` (e `cpf` na ficha completa); `POST /professionals[/:id]` aceita `cpf` (409 `cpf_taken`, 422 `invalid` com `fields: ["cpf"]`).
7. **§5.1 — `last_check_message`**: texto nosso, em português, até 200 caracteres (ex.: "Login no PEC aceito", "O PEC recusou usuário ou senha", "PEC inalcançável", "O PEC respondeu 500", "O PEC respondeu 200 sem sessão", "Endereço do PEC não definido pelo operador", "CADSUS respondeu", "O CADSUS recusou a credencial", "CADSUS inalcançável"). Proposta: o contrato dizer que a tela mostra o texto como veio.
8. **§4.2 — `features` no console para cidade não `active`**: o api não disca cidade suspensa/arquivada/em provisionamento; ela sai com `city_reachable: false`, `ibge_code: null` e `missing: ["city_unreachable"]`. Proposta: o contrato registrar que "inalcançável" inclui "não ativa".
