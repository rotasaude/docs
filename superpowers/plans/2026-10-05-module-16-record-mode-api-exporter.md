# Módulo 16 — Exportador LEDI (api) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Provar que o Rota Saúde serializa fichas LEDI APS 8.7.0 em Ruby e as entrega ao PEC da cidade, e então construir o exportador contínuo — layout versionado, fila `ledi_outbox` cifrada, envio por job da cidade com novas tentativas e pausa por credencial, prazo da competência, painel "Produção e-SUS" na cidade e no console — o lado api de F-16.6 e F-16.7 (ADR 0028).

**Architecture:** A Task 1 é a prova técnica: um PEC de verdade em container de dev, as classes Thrift geradas dos IDLs oficiais (`bin/ledi-generate` → `vendor/ledi/8.7.0/`), uma ficha sintética enviada pelo `Ledi::PecClient` e as respostas do PEC gravadas em `config/ledi/pec_observations.yml`, que o código lê (duplicidade, reenvio, sessão expirada). Se a serialização falhar, o plano **para**. Depois: `Ledi::Version`/`Ledi::Transport` montam o `DadoTransporteThrift`; uma ficha é qualquer objeto que cumpre `Ledi::Ficha` (no 16 só `Ledi::Fichas::Synthetic`); `Ledi::Enqueue` grava a ficha já serializada e cifrada em `ledi_outbox` (migração de cidade + trigger); `Ledi::DeliverJob` (por cidade, a cada minuto e ao enfileirar) reivindica um lote com `FOR UPDATE SKIP LOCKED`, marca `sending`, sai da transação e envia com `Ledi::Delivery` (login com cookie em cache por cidade, relogin uma vez, pausa na segunda recusa). `Ledi::Deadline` calcula o 10º dia útil; `Ledi::ProductionSummary` e `Ledi::Alert` alimentam `GET /production`; um job da cidade publica o resumo numa tabela da plataforma, que o console lê em `GET /city_production` sem abrir banco de cidade (padrão do Analytics, ADR 0025).

**Tech Stack:** Rails 8.1 (API), PostgreSQL 16 (banco por cidade + plataforma), Solid Queue, RSpec, gem `thrift` (TBinaryProtocol), Active Record Encryption (chave por cidade, ADR 0007), e-SUS APS PEC ≥ 5.5.26 (Linux, Java 17) em Docker só para dev.

**Spec:** `docs/superpowers/specs/2026-10-05-module-16-record-mode-and-export-design.md` (§6, §8, §9, §10) e `docs/adr/0028.md` — leia os dois antes de começar. Contratos entre apps (fonte única de formatos): `docs/superpowers/plans/2026-10-05-module-16-record-mode-contracts.md` (§4.3, §5.3, §6). Pesquisa: `docs/pesquisa/2026-10-05-integracoes/frente-1-aps-sisab.md` (§2 LEDI, §3 prazo). Neste worktree de planejamento os caminhos ficam sob `docs/.claude/ciclo2/`.

**Depende de:** plano `2026-10-05-module-16-record-mode-api-foundation.md` **mergeado** na `origin/main` do api antes da Task 2 (a Task 1 só precisa do `Ledi::PecClient#login` dele).

## Desvios da spec (e precisões)

1. **`codIbge` vem de `city_profile.ibge_code`** (decisão do coordenador): não existe `cities.ibge_code`. `Ledi::Transport` lê `CityProfile.current.ibge_code` no banco da cidade; sem ele, `Ledi::Transport::IbgeMissing` (o pré-requisito `ibge_code_missing` da fundação já impede o envio antes disso).
2. **`payload` é `text`, não `bytea`.** O Active Record Encryption grava texto; o binário Thrift entra em Base64 e é cifrado com a chave da cidade (`encrypts :payload`, sem `deterministic`). A garantia da spec (cifrado; apagado em `accepted`) fica igual, com `CHECK (status <> 'accepted' OR payload IS NULL)`.
3. **Coluna a mais em `ledi_outbox`: `first_attempt_at`.** "Nova tentativa até 24 h, depois `failed`" conta 24 h desde a **primeira tentativa**, não desde o enfileiramento: uma cidade com o envio pausado (credencial recusada, interruptor desligado) não perde fichas que nunca foram tentadas.
4. **`sending` é reivindicação, não trava longa.** O lote é marcado `sending` numa transação curta (`SKIP LOCKED`) e o HTTP acontece fora dela; `sending` parado há mais de 10 min volta a `pending` (processo que caiu). O reenvio do mesmo `uuidDadoSerializado` depois de um 200 perdido é tratado pelo que a prova observar (`duplicate_after_accept` em `pec_observations.yml`).
5. **Login recusado é pausa direta.** O PEC responde 400 no login com credencial inválida ou inativa (documentação da API); o exportador trata como a "segunda 401": credencial `unauthorized`, lote devolvido a `pending` sem contar tentativa.
6. **`numLote` não é enviado** (campo opcional do transporte; o exemplo oficial manda `0` sem significado).
7. **Resumo do console na plataforma.** O console nunca abre banco de cidade (padrão do ADR 0025): nasce `city_production_summaries` (plataforma), escrita por `Ledi::PublishProductionJob` (cidade, a cada 10 min); `business_days_left` e `alert` são recalculados na leitura.
8. **Alerta `attention` conta `failed` junto com pendente e recusada** — falha definitiva também é produção que não chegou.
9. **Feriados:** fixos nacionais (01/01, 21/04, 01/05, 07/09, 12/10, 02/11, 15/11, 20/11, 25/12) e móveis (segunda e terça de Carnaval, Sexta-feira Santa, Corpus Christi — os dois primeiros e o último são ponto facultativo federal, contados como não úteis: o prazo estimado sai igual ou **antes** do oficial, nunca depois). Conferido contra o calendário SIAPS 2026: 11 das 12 competências batem; maio/2026 dá 15/06 (o calendário publicado diz 16/06).

## Global Constraints

- Tudo de domínio no banco de cada cidade: migração em `db/city_migrate`, dump à mão em `db/city_schema.rb`, triggers em `db/city_triggers.sql` (o dump não representa trigger; a migração termina com `execute File.read(Rails.root.join("db/city_triggers.sql"))`). A tabela de plataforma vai em `db/platform_migrate` com `db/platform_schema.rb` regravado pelo `db:migrate`. Migrações só de expansão. Rollout roda `city:migrate:all`; **nunca** migrar cidade fora do rake (cidade trava em 503).
- Valores, exatamente: `ledi_outbox.status` ∈ `pending`, `sending`, `accepted`, `rejected`, `failed`; `ficha_type` do sintético = `procedimento` (`tipoDadoSerializado` 7); `competence` `AAAAMM`; `uuid` = `<CNES>-<UUID v4>` (≤ 44 caracteres); `ledi_version` = `Ledi::Version::ACTIVE` = `"8.7.0"`; `VersaoThrift(8, 7, 0)`; `contraChave` = `"Rota Saúde - <versão do software>"`; `alert` ∈ `none`, `attention`, `critical`.
- Credencial e `payload` nunca em resposta, log, evento ou argumento de job. `Ledi::DeliverJob` e `Ledi::PublishProductionJob` não recebem argumentos. `last_error` nunca guarda dado de cidadão: passa por `Ledi::ErrorText.sanitize` (sequências de 11+ dígitos viram `[número]`, máximo 500 caracteres).
- `payload` cifrado com a chave da cidade (`encrypts :payload`) e listado em `CityEncryption::CITY_KEYED_TARGETS` (`spec/architecture/city_encrypted_attributes_guard_spec.rb`).
- Eventos de domínio novos, exatamente e só com ids: `ledi.ficha_accepted` e `ledi.ficha_rejected` `{ outbox_id, ficha_type, competence }`; declarados em `config/initializers/domain_events.rb` com `to: []` e em `spec/initializers/domain_events_bindings_spec.rb`. Nenhum `Platform.audit` novo.
- Interruptor: nada sai sem `Platform::Features.usable?(city, :ledi_export)`; nada entra na fila sem `Platform::Features.enabled?(city, :ledi_export)` e `city.record_mode != "off"`; rota da funcionalidade desligada → `403 { "error": "feature_disabled", "feature": "ledi_export" }`.
- Jobs por cidade só com `prepend EachCityJob` (recorrentes, sem argumento) ou `include CityScopedJob`; `Current.city` **nunca** é atribuído em `app/` ou `lib/` (`spec/architecture/current_city_assignment_spec.rb`).
- Erros sempre `{ "error": "<reason>" }`; step-up pelo `MfaStepUp` (401 `mfa_required`); papel ausente 403 `missing_role`.
- `Ledi::Fichas::Synthetic` só existe em development e test (`Rails.env.local?`); levanta `Ledi::Fichas::Synthetic::NotAllowed` fora deles.
- A suíte `:pec` (contra PEC real) fica fora da CI: `config.filter_run_excluding :pec unless ENV["LEDI_PEC_URL"].present?`.
- Specs de request precisam de `type: :request` (inferência desligada). Arquivo novo em `spec/support/` precisa de `require_relative` em `spec/rails_helper.rb`. Datas de banco derivadas de `Time.zone.today`; funções puras (`Ledi::Deadline`, `Ledi::Alert`) recebem `today:` e podem usar data fixa.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com a linha `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Nunca `git add -A`: caminhos explícitos.

## Interfaces consumidas da fundação (`api-foundation`)

Os nomes abaixo são os combinados para a fundação. A Task 2, Step 1, confere cada um no código mergeado; se algum nome real for diferente, ajuste **só** `spec/support/ledi_helpers.rb` e as chamadas desta lista (registre no relatório da task), nunca o código da fundação — salvo o acréscimo explícito da Task 1 no `Ledi::PecClient`.

- `Platform::Features.enabled?(city, key) -> Boolean` e `.usable?(city, key) -> Boolean` (`city` = `City`; `key` símbolo ou texto — o catálogo compara `key.to_s`; `usable?` é falso com `credential_unauthorized:ledi`, `ibge_code_missing` etc.); `Platform::Features.set!(city:, key:, enabled:, maintainer:)` (só no helper de spec).
- `City#record_mode` (`"off"`|`"integrated"`|`"record"`) e `City#pec_url` (HTTPS ou `nil`), plataforma.
- `CityProfile.current.ibge_code` (já existe; fonte única do código IBGE — decisão do coordenador).
- `CityFeature` (plataforma): `city_id`, `key`, `enabled`, `changed_at`, `changed_by_maintainer_id` (NOT NULL, FK para `maintainers`).
- `IntegrationCredential` (cidade): `kind` (`"ledi"`|`"cadsus"`, único), `secret` (hash cifrado `{ "username", "password" }`), `#username`, `#password`, `set_by_user`, `set_at`, `last_check_at`, `last_check_status` (`ok`|`unauthorized`|`unreachable`|`error`), `last_check_message`.
- `Ledi::PecClient.new(base_url:, username:, password:, timeout: 10)`; `#login -> Ledi::PecClient::Session` (`Data` com `cookie`, ex. `"JSESSIONID=abc"`); erros `Ledi::PecClient::Error` e subclasses `Unauthorized` (login 401/403), `Unreachable` (rede/timeout/TLS), `Failed` (`#status`); `LOGIN_PATH`; `#post(path, body, content_type, headers = {})` privado, que levanta `Unreachable`. O login da fundação manda JSON `{ usuario, senha }`; o exemplo oficial manda `multipart/form-data` — a Task 1 decide pela prova.
- `FeatureGate#require_feature!` (concern de controller, 403 `feature_disabled`): se existir com essa assinatura, a Task 12 o usa no lugar do check local.
- `Terminology::SigtapStatus.call(today:) -> { sigtap_current_competence:, sigtap_imported:, alert: }` (plataforma; usado na Task 13).
- Não consumidos aqui (fichas reais, módulos 18+): `health_units.cnes`, `health_teams.ine/kind`, `Terminology::Sigtap.compatible?` — a ficha sintética recebe CNES, INE, CNS e CBO explícitos.

## Ambiente de execução

- Antes de tocar `apps/api`, avise a sessão dona do api (sessão "API"). Ordem: `contracts` → `api-foundation` → **este plano** → dashboard (Produção) e admin (visão geral).
- Worktrees (a partir da raiz do monorepo, `/Users/eduardovrocha/Development/ioit.solutions/rota-saude`):

  ```bash
  /opt/homebrew/bin/git -C apps/api fetch origin
  /opt/homebrew/bin/git -C apps/api worktree add .claude/mod16-exporter -b feat/mod-16-ledi-exporter origin/main
  cp apps/api/config/master.key apps/api/.claude/mod16-exporter/config/master.key
  /opt/homebrew/bin/git -C docs fetch origin
  /opt/homebrew/bin/git -C docs worktree add .claude/mod16-exporter -b feat/mod-16-ledi-exporter origin/main
  ```

  Se a fundação ainda estiver só no branch `feat/mod-16-record-mode`, crie a partir dele e faça rebase em `origin/main` quando ela entrar.
- `./apps/api` é montado em `/rails` no container `api`; o worktree é `/rails/.claude/mod16-exporter`. Todo comando Rails/RSpec:

  ```bash
  docker compose exec -T -w /rails/.claude/mod16-exporter api bundle exec rspec <arquivos>
  ```

- Depois da migração de cidade (Task 5): `DROP DATABASE` dos dois bancos de teste de cidade e `city:test_databases`:

  ```bash
  psql -h localhost -U rota_saude -d postgres -c "DROP DATABASE IF EXISTS rota_saude_test_city_a" -c "DROP DATABASE IF EXISTS rota_saude_test_city_b"
  docker compose exec -T -w /rails/.claude/mod16-exporter -e RAILS_ENV=test api bin/rails city:test_databases
  ```

  Ao voltar para a main, repita (o banco de teste fica à frente).
- Suíte completa só com o worker parado:

  ```bash
  docker compose stop worker
  docker compose exec -T -w /rails/.claude/mod16-exporter api bundle exec rspec
  docker compose start worker
  ```

- PEC local: `apps/api/.claude/mod16-exporter/deploy/development/pec/` (Task 1), alcançado do container `api` como `https://pec.rota.test:8443`.

## Review Focus

1. **O 200 que se perdeu:** o processo cai depois do POST e antes do `UPDATE`; a ficha volta de `sending` para `pending` e é reenviada com o mesmo `uuidDadoSerializado`. Ela não pode virar `rejected` por "já recebida" nem ser contada duas vezes. Testes: Task 7 ("duplicidade observada classifica como aceita") e Task 9 ("sending parado volta a pending e o reenvio duplicado vira accepted").
2. **A cidade troca a credencial no meio:** o cookie em cache da credencial antiga não pode ser reaproveitado, e a pausa por credencial recusada termina assim que a credencial nova é testada como `ok`. Testes: Task 8 ("credencial trocada não reaproveita o cookie" e "pausada não envia; voltou a ok, envia").
3. **Mensagem de recusa do PEC com CPF/CNS do cidadão:** `last_error` e o agrupamento de motivos no painel mostram `[número]`, nunca os dígitos. Testes: Task 7 (tabela do `sanitize`) e Task 12 ("rejections agrupa a mensagem já mascarada").
4. **Virada do mês e do fuso:** ficha atendida às 23h30 do último dia do mês em Manaus é da competência daquele mês (não do seguinte, como seria em UTC); no dia do prazo restam 1 dia útil e o alerta vale, no dia seguinte 0 e o alerta some; Carnaval de 2027 empurra o prazo de janeiro. Testes: Task 4 ("competência no fuso da cidade"), Task 10 (tabela 2026 + Carnaval 2027 + `business_days_left`) e Task 11 ("prazo vencido: none").
5. **PEC fora do ar por dias, ou interruptor desligado no meio:** fichas ficam `pending` com espera crescente e só viram `failed` 24 h depois da **primeira** tentativa; desligar o interruptor (ou `record_mode=off`) com fila cheia faz o próximo minuto não enviar nada. Testes: Task 7 (backoff), Task 8 ("24 h depois da primeira tentativa: failed" e "interruptor desligado: nada sai").

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| `deploy/development/pec/{compose.yml,Dockerfile,entrypoint.sh}`, `.gitignore`, `bin/ledi-generate`, `vendor/ledi/8.7.0/**`, `Gemfile`, `Gemfile.lock`, `app/services/ledi/pec_client.rb` (acréscimo), `config/ledi/pec_observations.yml`, `spec/integration/ledi_pec_spec.rb`, `spec/spec_helper.rb`, `spec/services/ledi/pec_client_deliver_spec.rb`; docs: `operacao/pec-local-dev.md` | prova técnica | 1 |
| `app/services/ledi/version.rb`, `app/services/ledi/observations.rb`, `spec/services/ledi/contract_spec.rb`, `spec/support/ledi_helpers.rb`, `spec/rails_helper.rb` | layout versionado e contrato contra os IDLs | 2 |
| `config/ledi.yml`, `app/services/ledi/sender.rb`, `app/services/ledi/transport.rb`, `app/services/ledi/ficha_types.rb` | transporte | 3 |
| `app/services/ledi/ficha.rb`, `app/services/ledi/fichas/synthetic.rb` | interface de ficha e ficha sintética | 4 |
| `db/city_migrate/20261007100001_create_ledi_outbox.rb`, `db/city_schema.rb`, `db/city_triggers.sql`, `app/models/ledi_outbox_entry.rb`, `app/services/city_encryption.rb` | fila | 5 |
| `app/commands/ledi/enqueue.rb` | enfileiramento | 6 |
| `app/services/ledi/outcome.rb`, `app/services/ledi/error_text.rb`, `app/services/ledi/backoff.rb` | classificação, saneamento, espera | 7 |
| `app/services/ledi/session_cache.rb`, `app/services/ledi/delivery.rb`, `app/jobs/ledi/deliver_job.rb`, `config/recurring.yml`, `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb` | envio | 8 |
| `spec/jobs/ledi/deliver_job_concurrency_spec.rb` | `SKIP LOCKED` e `sending` parado | 9 |
| `app/services/ledi/business_calendar.rb`, `app/services/ledi/deadline.rb` | prazo | 10 |
| `app/services/ledi/alert.rb`, `app/queries/ledi/production_summary.rb` | resumo e alerta | 11 |
| `app/controllers/production_controller.rb`, `app/policies/production_policy.rb`, `app/commands/ledi/resend.rb`, `config/routes.rb` | painel da cidade | 12 |
| `db/platform_migrate/20261007100002_create_city_production_summaries.rb`, `db/platform_schema.rb`, `app/models/city_production_summary.rb`, `app/jobs/ledi/publish_production_job.rb`, `app/queries/ledi/city_production_query.rb`, `app/controllers/operators/city_production_controller.rb` | console | 13 |
| `spec/invariants/record_mode_invariants_spec.rb` | invariantes do exportador (ADR 0028) | 14 |
| `lib/ledi_crew.rb`, `db/seeds.rb`, `lib/tasks/ledi.rake` | semente e prova no navegador | 15 |

---

## Fatia 1 — Prova técnica (F-16.6)

### Task 1: PEC local, classes Thrift 8.7.0 e uma ficha sintética aceita

Esta task decide se o resto do plano existe. **Critério de saída:** a ficha sintética serializada em Ruby recebe 2xx do `POST /api/v1/recebimento/ficha` **e** aparece na instalação. **Se a serialização falhar** (o PEC responder erro de desserialização/formato — 400 ou 500 cuja mensagem fale de leitura do arquivo, Thrift, protocolo ou formato — para a ficha montada como o exemplo oficial), **pare o plano**, não comece a Task 2, e reporte ao usuário: versão do PEC, commit dos IDLs, bytes enviados (tamanho e os 64 primeiros em hex), status e corpo da resposta. A decisão volta para o ADR 0028 (alternativa: serviço Java). Recusa de **validação** (CNES, CNS, CBO, IBGE) não é falha de serialização: corrija o dado e tente de novo.

**Files:**
- Create: `deploy/development/pec/compose.yml`, `deploy/development/pec/Dockerfile`, `deploy/development/pec/entrypoint.sh`
- Modify: `.gitignore` (acrescenta `deploy/development/pec/local/`)
- Create: `bin/ledi-generate`, `vendor/ledi/8.7.0/` (saída do gerador)
- Modify: `Gemfile`, `Gemfile.lock` (gem `thrift`)
- Modify: `app/services/ledi/pec_client.rb` (acréscimo `#deliver`, `Reply`, CA local; formato do login conforme a prova)
- Create: `config/ledi/pec_observations.yml`
- Modify: `spec/spec_helper.rb` (exclusão `:pec`)
- Test: `spec/services/ledi/pec_client_deliver_spec.rb`, `spec/integration/ledi_pec_spec.rb`
- Create (docs): `operacao/pec-local-dev.md`

**Interfaces:**
- Consumes: `Ledi::PecClient.new(base_url:, username:, password:)`, `#login -> Session`, `#post` privado (fundação).
- Produces:
  - `Ledi::PecClient::Reply = Data.define(:status, :body)`; `Ledi::PecClient::DELIVER_PATH`; `#deliver(cookie:, filename:, bytes:) -> Reply` (multipart, campo `ficha`), levantando `Ledi::PecClient::Unreachable` em timeout/conexão/TLS; `#post` passa a aceitar a CA local (`LEDI_PEC_CA_FILE`, só development/test); o formato do login (JSON ou multipart) e o status do login recusado ficam os que a prova observar.
  - `vendor/ledi/8.7.0/gen-rb/*.rb` (módulos `Br::Gov::Saude::Esusab::Dadotransp`, `Br::Gov::Saude::Esusab::Ras::Atendprocedimentos`, `Br::Gov::Saude::Esusab::Ras::Common`), `vendor/ledi/8.7.0/idl/**/*.thrift`, `vendor/ledi/8.7.0/SOURCE`.
  - `config/ledi/pec_observations.yml` com as chaves `observed_on`, `pec_version`, `ledi_version`, `idl_commit`, `success_statuses`, `compression`, `rejection.status`, `rejection.example`, `duplicate_after_accept.status`, `duplicate_after_accept.marker`, `resend_after_rejection.same_uuid`, `session_expired_statuses`.

- [ ] **Step 1: Baixe o instalador do PEC (com autorização)**

O instalador Linux fica em `https://sisaps.saude.gov.br/esus/` (versão atual 5.5.28; a LEDI 8.7.0 exige PEC ≥ 5.5.26). **Peça autorização ao usuário antes de baixar**, dizendo nome do arquivo (`eSUS-AB-PEC-5.5.28-Linux64.jar` ou o nome exato da página), origem e tamanho. Salve em `apps/api/.claude/mod16-exporter/deploy/development/pec/local/` e acrescente ao `.gitignore` do api:

```gitignore
# PEC local da prova técnica (módulo 16): instalador, certificados e CA. Nunca no git.
/deploy/development/pec/local/
```

- [ ] **Step 2: Crie o container do PEC**

```dockerfile
# deploy/development/pec/Dockerfile
# PEC e-SUS APS SÓ para desenvolvimento (prova técnica do módulo 16, ADR 0028).
# O instalador oficial é Linux x86-64 e espera systemd; no container não há init,
# então `systemctl`, `ps` e `file` são simulados só nas consultas que o
# instalador faz. Nunca use esta imagem fora de dev.
FROM --platform=linux/amd64 ubuntu:22.04

ENV DEBIAN_FRONTEND=noninteractive TZ=America/Sao_Paulo LANG=pt_BR.UTF-8

RUN apt-get update && apt-get install -y --no-install-recommends \
      ca-certificates wget gnupg locales tzdata postgresql-client file procps fontconfig \
    && locale-gen pt_BR.UTF-8 \
    && rm -rf /var/lib/apt/lists/*

RUN wget -qO- https://apt.corretto.aws/corretto.key | gpg --dearmor -o /usr/share/keyrings/corretto.gpg \
    && echo "deb [signed-by=/usr/share/keyrings/corretto.gpg] https://apt.corretto.aws stable main" \
       > /etc/apt/sources.list.d/corretto.list \
    && apt-get update && apt-get install -y --no-install-recommends java-17-amazon-corretto-jdk \
    && rm -rf /var/lib/apt/lists/*

RUN printf '#!/bin/sh\nexit 0\n' > /bin/systemctl && chmod +x /bin/systemctl \
    && mv /bin/ps /bin/ps.original \
    && printf '#!/bin/sh\nif [ "$1" = "--no-headers" ] && [ "$4" = "1" ]; then echo systemd; else exec /bin/ps.original "$@"; fi\n' > /bin/ps \
    && chmod +x /bin/ps \
    && mv /usr/bin/file /usr/bin/file.original \
    && printf '#!/bin/sh\nif [ "$1" = "-L" ] && [ "$2" = "/sbin/init" ]; then echo "/sbin/init: ELF 64-bit LSB executable, x86-64, systemd"; else exec /usr/bin/file.original "$@"; fi\n' > /usr/bin/file \
    && chmod +x /usr/bin/file

COPY entrypoint.sh /entrypoint.sh
RUN chmod +x /entrypoint.sh
ENTRYPOINT ["/entrypoint.sh"]
```

```sh
#!/bin/sh
# deploy/development/pec/entrypoint.sh
# Instala o PEC na primeira subida (modo treinamento, banco no serviço pec-db) e
# liga o HTTPS manual do manual oficial (APOIO/Certificado_Https_Linux) com o
# keystore de local/esusaps.p12. Depois só inicia.
set -eu

if [ ! -x /opt/e-SUS/webserver/standalone.sh ]; then
  java -jar "/pec/local/${PEC_JAR}" -console \
    -url="jdbc:postgresql://pec-db:5432/esus" -username=esus -password=esus -continue
  PGPASSWORD=esus psql -h pec-db -U esus -d esus -v ON_ERROR_STOP=1 \
    -c "update tb_config_sistema set ds_texto = null, ds_inteiro = 1 where co_config_sistema = 'TREINAMENTO';"
fi

CONF=/opt/e-SUS/webserver/config/application.properties
if [ -f /pec/local/esusaps.p12 ] && ! grep -q '^server.ssl.key-store=' "$CONF"; then
  cp /pec/local/esusaps.p12 /opt/e-SUS/webserver/config/esusaps.p12
  cat >> "$CONF" <<EOF
server.port=8443
server.ssl.key-store=config/esusaps.p12
server.ssl.key-store-password=${PEC_KEYSTORE_PASSWORD}
server.ssl.key-store-type=PKCS12
server.ssl.key-alias=esusaps
security.require-ssl=true
EOF
fi

exec /opt/e-SUS/webserver/standalone.sh
```

```yaml
# deploy/development/pec/compose.yml
# PEC local da prova técnica (módulo 16). Projeto separado do compose da raiz;
# entra na rede dele (rota-saude_default) com o apelido pec.rota.test, que é o
# pec_url da cidade de dev. Subir: ver docs/operacao/pec-local-dev.md.
name: rota-saude-pec

services:
  pec-db:
    image: postgres:14
    environment:
      POSTGRES_USER: esus
      POSTGRES_PASSWORD: esus
      POSTGRES_DB: esus
    volumes:
      - pec-db:/var/lib/postgresql/data

  pec:
    build: .
    platform: linux/amd64
    environment:
      PEC_JAR: ${PEC_JAR:?defina PEC_JAR com o nome do instalador em local/}
      PEC_KEYSTORE_PASSWORD: ${PEC_KEYSTORE_PASSWORD:?defina PEC_KEYSTORE_PASSWORD}
    volumes:
      - ./local:/pec/local:ro
      - pec-opt:/opt/e-SUS
    ports:
      - "8443:8443"
    depends_on:
      - pec-db
    networks:
      default: {}
      rota:
        aliases:
          - pec.rota.test

networks:
  rota:
    external: true
    name: rota-saude_default

volumes:
  pec-db:
  pec-opt:
```

- [ ] **Step 3: Gere a CA local e o certificado de `pec.rota.test`**

Run (no diretório `deploy/development/pec/local/` do worktree):
```bash
openssl req -x509 -newkey rsa:2048 -nodes -days 365 -subj "/CN=Rota Saude Dev CA" -keyout ca.key -out ca.pem
openssl req -newkey rsa:2048 -nodes -subj "/CN=pec.rota.test" -keyout pec.key -out pec.csr
printf "subjectAltName=DNS:pec.rota.test,DNS:localhost\n" > san.ext
openssl x509 -req -in pec.csr -CA ca.pem -CAkey ca.key -CAcreateserial -days 365 -extfile san.ext -out pec.crt
openssl pkcs12 -export -in pec.crt -inkey pec.key -name esusaps -out esusaps.p12 -passout "pass:$PEC_KEYSTORE_PASSWORD"
```
Expected: `ca.pem`, `esusaps.p12` em `local/` (fora do git).

- [ ] **Step 4: Suba o PEC e configure a instalação**

Run:
```bash
cd apps/api/.claude/mod16-exporter/deploy/development/pec
PEC_JAR=<nome do .jar> PEC_KEYSTORE_PASSWORD=<senha local> docker compose up -d --build
docker compose logs -f pec
```
Expected: o instalador termina e o log do `standalone.sh` mostra o servidor ouvindo em 8443 (a primeira subida em Mac arm64 roda emulada e leva vários minutos; o PEC pede ≥ 4 GB de memória para o Docker).

No navegador, `https://localhost:8443` (aceite o certificado da CA local):
1. Cadastre o administrador da instalação.
2. Configure o município da instalação como **Maringá (4115200)** — é o `city_profile.ibge_code` de `maringa` na semente de dev.
3. Importe o CNES do município (arquivo XML do CNES para o e-SUS APS, obtido no portal do CNES) e anote **fora do repositório**: um CNES de UBS, o INE de uma equipe eSF (tipo 70) dessa UBS e o CNS + CBO de um profissional dessa equipe (CBO 225142 ou 223505).
4. Em "Transmissão de dados" → "Credenciais para API", gere uma credencial de pessoa jurídica ("Rota Saúde dev", CNPJ fictício 11.222.333/0001-81). Usuário e senha aparecem **uma vez**: guarde só em variáveis do shell (`LEDI_PEC_USERNAME`, `LEDI_PEC_PASSWORD`), nunca em arquivo do repositório.

**Parada antecipada:** se a instalação de treinamento não oferecer "Credenciais para API" (ou recusar gerar por falta de HTTPS reconhecido), **pare** e reporte ao usuário com print/texto da tela — uma instalação de produção exige contra-chave do Ministério e sai do escopo de dev.

- [ ] **Step 5: Escreva `bin/ledi-generate`**

```bash
#!/usr/bin/env bash
# bin/ledi-generate <versão-ledi> <commit-do-esusaps-integracao>
# Gera vendor/ledi/<versão>/ a partir dos IDLs oficiais (github.com/laboratoriobridge/
# esusaps-integracao, pasta thrift/), conferindo cada .thrift contra o sha256 que o
# próprio repositório publica em thrift/hash/. Roda no HOST (precisa de docker,
# curl, tar e shasum). O compilador vem do pacote thrift-compiler do Debian, num
# container descartável. Nunca edite vendor/ledi/ à mão: rode de novo.
set -euo pipefail

VERSION="${1:?uso: bin/ledi-generate <versão> <commit>}"
COMMIT="${2:?uso: bin/ledi-generate <versão> <commit>}"
ROOT="$(cd "$(dirname "$0")/.." && pwd)"
OUT="$ROOT/vendor/ledi/$VERSION"
WORK="$(mktemp -d)"
trap 'rm -rf "$WORK"' EXIT

curl -fsSL "https://codeload.github.com/laboratoriobridge/esusaps-integracao/tar.gz/$COMMIT" | tar -xz -C "$WORK"
SRC="$(find "$WORK" -maxdepth 1 -mindepth 1 -type d)/thrift"

for idl in "$SRC"/layout-ras/thrift/*.thrift "$SRC"/layout-camada-transport/thrift/*.thrift; do
  name="$(basename "$idl")"
  expected="$(tr -d '[:space:]' < "$SRC/hash/$name.sha256")"
  actual="$(shasum -a 256 "$idl" | cut -d' ' -f1)"
  if [ "$expected" != "$actual" ]; then
    echo "hash divergente para $name: esperado $expected, obtido $actual" >&2
    exit 1
  fi
done

rm -rf "$OUT"
mkdir -p "$OUT/idl/ras" "$OUT/idl/transport" "$OUT/gen-rb"
cp "$SRC"/layout-ras/thrift/*.thrift "$OUT/idl/ras/"
cp "$SRC"/layout-camada-transport/thrift/*.thrift "$OUT/idl/transport/"

docker run --rm -v "$OUT:/ledi" debian:bookworm-slim sh -c '
  apt-get update -qq >/dev/null && apt-get install -y -qq thrift-compiler >/dev/null &&
  for f in /ledi/idl/ras/*.thrift /ledi/idl/transport/*.thrift; do thrift --gen rb -out /ledi/gen-rb "$f"; done &&
  thrift --version' > "$WORK/compiler.txt"

{
  echo "ledi_version: $VERSION"
  echo "source: https://github.com/laboratoriobridge/esusaps-integracao/tree/$COMMIT/thrift"
  echo "generated_on: $(date +%F)"
  echo "compiler: $(tail -1 "$WORK/compiler.txt")"
  echo "idl_sha256:"
  (cd "$OUT/idl" && find . -name '*.thrift' | sort | while read -r f; do echo "  $f: $(shasum -a 256 "$f" | cut -d' ' -f1)"; done)
} > "$OUT/SOURCE"

echo "vendor/ledi/$VERSION gerado de $COMMIT"
```

Run: `chmod +x bin/ledi-generate`.

- [ ] **Step 6: Escolha o commit dos IDLs da 8.7.0 e gere**

O repositório oficial não tem tags. A LEDI 8.7.0 saiu em 13/08/2026; o `master` em 2026-10-05 é `9316f280f5` (11/09/2026). Compare os campos de `thrift/layout-ras/thrift/ficha_atendimento_procedimento.thrift` e `thrift/layout-camada-transport/thrift/dado_transporte.thrift` desse commit com o dicionário da 8.7.0 em `https://integracao.esusaps.bridge.ufsc.tech/ledi/documentacao/estrutura_arquivos/` (procedimentos e camada de transporte). Se baterem, use o commit; se não, ande para trás no histórico (`git log -- thrift/`) até bater e anote a escolha.

Run (no host, raiz do worktree do api):
```bash
bin/ledi-generate 8.7.0 9316f280f5
ls vendor/ledi/8.7.0/gen-rb | head
```
Expected: `common_types.rb`, `dado_transporte_types.rb`, `ficha_atendimento_procedimento_types.rb` etc. Hash divergente → **pare** e reporte (o gerador não grava nada).

- [ ] **Step 7: Acrescente a gem `thrift`**

Em `Gemfile`, depois de `gem "graphql", "~> 2.3"`:

```ruby
gem "thrift", "~> 0.22"   # LEDI APS (ADR 0028): TBinaryProtocol das fichas para o PEC
```

Run: `docker compose exec -T -w /rails/.claude/mod16-exporter api bundle install`
Expected: `Gemfile.lock` com `thrift (0.22.x)`. Se a extensão nativa não compilar, reporte antes de seguir.

- [ ] **Step 8: Escreva a spec do `#deliver` (falha)**

```ruby
# spec/services/ledi/pec_client_deliver_spec.rb
require "rails_helper"

# ADR 0028 / API de transmissão do PEC: POST /api/v1/recebimento/ficha,
# multipart com o campo `ficha` (arquivo <uuid>.esus) e o cookie da sessão,
# como no exemplo oficial ExemploEnvioApi.java. Stuba Net::HTTP.start (como
# spec/services/whatsapp/outbound_spec.rb).
RSpec.describe Ledi::PecClient, "#deliver" do
  let(:client) { described_class.new(base_url: "https://pec.cidade.gov.br", username: "rota", password: "senha-secreta") }
  let(:http) { instance_double(Net::HTTP) }
  let(:sent) { [] }

  before { allow(Net::HTTP).to receive(:start).and_yield(http) }

  it "envia o binário como multipart `ficha` com o nome do arquivo e o cookie" do
    allow(http).to receive(:request) { |req| sent << req; instance_double(Net::HTTPCreated, code: "201", body: "") }
    bytes = "\x0B\x00\x01\xFF".b

    reply = client.deliver(cookie: "JSESSIONID=abc", filename: "1234567-u.esus", bytes: bytes)

    expect(reply).to eq(Ledi::PecClient::Reply.new(status: 201, body: ""))
    req = sent.sole
    expect(req.path).to eq("/api/v1/recebimento/ficha")
    expect(req["Cookie"]).to eq("JSESSIONID=abc")
    boundary = req["Content-Type"][/\Amultipart\/form-data; boundary=(\S+)\z/, 1]
    expect(boundary).to be_present
    expect(req.body.b).to include(%(name="ficha"; filename="1234567-u.esus").b, bytes, "--#{boundary}--".b)
    expect(req.body).not_to include("senha-secreta")
  end

  it "devolve o status e o corpo de uma recusa" do
    allow(http).to receive(:request).and_return(instance_double(Net::HTTPBadRequest, code: "400", body: "CNES inválido"))
    expect(client.deliver(cookie: "c", filename: "f.esus", bytes: "x").to_h).to eq(status: 400, body: "CNES inválido")
  end

  it "timeout, conexão recusada e TLS viram Unreachable" do
    [ Net::ReadTimeout, Net::OpenTimeout, Errno::ECONNREFUSED, OpenSSL::SSL::SSLError, SocketError ].each do |error|
      allow(http).to receive(:request).and_raise(error)
      expect { client.deliver(cookie: "c", filename: "f.esus", bytes: "x") }
        .to raise_error(Ledi::PecClient::Unreachable), error.name
    end
  end

  it "CA local só em development/test" do
    allow(http).to receive(:request).and_return(instance_double(Net::HTTPCreated, code: "201", body: ""))
    allow(ENV).to receive(:[]).and_call_original
    allow(ENV).to receive(:[]).with("LEDI_PEC_CA_FILE").and_return("/tmp/ca.pem")
    client.deliver(cookie: "c", filename: "f.esus", bytes: "x")
    expect(Net::HTTP).to have_received(:start).with("pec.cidade.gov.br", 443, hash_including(ca_file: "/tmp/ca.pem"))
  end
end
```

Run: `docker compose exec -T -w /rails/.claude/mod16-exporter api bundle exec rspec spec/services/ledi/pec_client_deliver_spec.rb`
Expected: FAIL com `undefined method 'deliver'`.

- [ ] **Step 9: Acrescente `#deliver` ao `Ledi::PecClient` da fundação**

Dentro de `module Ledi; class PecClient`, depois de `LOGIN_PATH`:

```ruby
    # Resposta crua do recebimento (o que significa fica com Ledi::Outcome).
    Reply = Data.define(:status, :body)

    DELIVER_PATH = "/api/v1/recebimento/ficha".freeze
```

Depois de `def login ... end`:

```ruby
    # POST do binário .esus (DadoTransporteThrift em TBinaryProtocol), como no
    # exemplo oficial ExemploEnvioApi.java: multipart, campo `ficha`, nome do
    # arquivo = uuidDadoSerializado + ".esus". Nunca loga corpo nem cookie.
    def deliver(cookie:, filename:, bytes:)
      boundary = "rotasaude#{SecureRandom.hex(12)}"
      body = "--#{boundary}\r\n" \
             "Content-Disposition: form-data; name=\"ficha\"; filename=\"#{filename}\"\r\n" \
             "Content-Type: application/octet-stream\r\n\r\n".b + bytes.b + "\r\n--#{boundary}--\r\n".b
      response = post(DELIVER_PATH, body, "multipart/form-data; boundary=#{boundary}", "Cookie" => cookie)
      Reply.new(status: response.code.to_i, body: response.body.to_s)
    end
```

No `#post` privado da fundação, troque o `Net::HTTP.start(...)` por:

```ruby
      Net::HTTP.start(uri.host, uri.port, **http_options(uri)) do |http|
        http.request(request)
      end
```

e acrescente, entre os privados:

```ruby
    # CA local só em development/test (PEC da prova técnica, certificado da CA de
    # deploy/development/pec/local/ca.pem; docs/operacao/pec-local-dev.md).
    def http_options(uri)
      options = { use_ssl: uri.scheme == "https", open_timeout: @timeout, read_timeout: @timeout, write_timeout: @timeout }
      ca_file = ENV["LEDI_PEC_CA_FILE"]
      options[:ca_file] = ca_file if ca_file.present? && Rails.env.local?
      options
    end
```

Run: `docker compose exec -T -w /rails/.claude/mod16-exporter api bundle exec rspec spec/services/ledi/pec_client_deliver_spec.rb spec/services/ledi/pec_client_spec.rb`
Expected: PASS (a spec do login da fundação continua verde).

- [ ] **Step 10: Exclua a suíte `:pec` por padrão**

Em `spec/spec_helper.rb`, dentro do `RSpec.configure`, logo depois de `config.filter_run_when_matching :focus`:

```ruby
  # Prova técnica do módulo 16: specs `:pec` falam com um PEC real (dev) e só
  # rodam com LEDI_PEC_URL definida — nunca na CI.
  config.filter_run_excluding :pec unless ENV["LEDI_PEC_URL"].to_s.strip != ""
```

- [ ] **Step 11: Escreva a prova `:pec`**

```ruby
# spec/integration/ledi_pec_spec.rb
require "rails_helper"

# Prova técnica do módulo 16 (spec §6.1). Fala com o PEC local de
# deploy/development/pec (docs/operacao/pec-local-dev.md). Variáveis (só no shell):
#   LEDI_PEC_URL, LEDI_PEC_CA_FILE, LEDI_PEC_USERNAME, LEDI_PEC_PASSWORD,
#   LEDI_PROOF_CNES, LEDI_PROOF_INE, LEDI_PROOF_CNS, LEDI_PROOF_CBO, LEDI_PROOF_IBGE.
# Cada exemplo imprime o que o PEC respondeu: é daí que sai pec_observations.yml.
RSpec.describe "Prova técnica LEDI contra o PEC local", :pec do
  around do |example|
    WebMock.disable! if defined?(WebMock)
    example.run
  ensure
    WebMock.enable! if defined?(WebMock)
  end

  GEN = Rails.root.join("vendor/ledi/8.7.0/gen-rb").to_s
  before(:all) do
    $LOAD_PATH.unshift(GEN) unless $LOAD_PATH.include?(GEN)
    require "dado_transporte_types"
    require "ficha_atendimento_procedimento_types"
  end

  let(:client) do
    Ledi::PecClient.new(base_url: ENV.fetch("LEDI_PEC_URL"), username: ENV.fetch("LEDI_PEC_USERNAME"),
                        password: ENV.fetch("LEDI_PEC_PASSWORD"))
  end
  let(:cookie) { client.login.cookie }
  let(:serializer) { Thrift::Serializer.new(Thrift::BinaryProtocolFactory.new) }
  let(:ras) { Br::Gov::Saude::Esusab::Ras }
  let(:transp) { Br::Gov::Saude::Esusab::Dadotransp }

  def ms(time) = (time.to_f * 1000).to_i

  def ficha(uuid, cnes: ENV.fetch("LEDI_PROOF_CNES"))
    now = Time.current.change(min: 0) - 1.hour
    child = ras::Atendprocedimentos::FichaProcedimentoChildThrift.new(
      dtNascimento: ms(Time.zone.local(1980, 5, 10)), sexo: 1, localAtendimento: 1, turno: 1,
      cpfCidadao: "12345678909", stCidadaoNaoPossuiCpf: false, procedimentos: [ "0301100039" ],
      dataHoraInicialAtendimento: ms(now), dataHoraFinalAtendimento: ms(now + 10.minutes)
    )
    header = ras::Common::UnicaLotacaoHeaderThrift.new(
      profissionalCNS: ENV.fetch("LEDI_PROOF_CNS"), cboCodigo_2002: ENV.fetch("LEDI_PROOF_CBO"), cnes: cnes,
      ine: ENV.fetch("LEDI_PROOF_INE"), dataAtendimento: ms(now), codigoIbgeMunicipio: ENV.fetch("LEDI_PROOF_IBGE")
    )
    ras::Atendprocedimentos::FichaProcedimentoMasterThrift.new(
      uuidFicha: uuid, tpCdsOrigem: 3, headerTransport: header, atendProcedimentos: [ child ]
    )
  end

  def transport(uuid, cnes: ENV.fetch("LEDI_PROOF_CNES"))
    installation = transp::DadoInstalacaoThrift.new(
      contraChave: "Rota Saúde - prova", uuidInstalacao: "rota-saude-dev-proof", cpfOuCnpj: "11222333000181",
      nomeOuRazaoSocial: "Rota Saúde dev", email: "dev@rotasaude.app"
    )
    transp::DadoTransporteThrift.new(
      uuidDadoSerializado: uuid, tipoDadoSerializado: 7, cnesDadoSerializado: cnes,
      codIbge: ENV.fetch("LEDI_PROOF_IBGE"), ineDadoSerializado: ENV.fetch("LEDI_PROOF_INE"),
      dadoSerializado: serializer.serialize(ficha(uuid, cnes: cnes)), remetente: installation,
      originadora: installation, versao: transp::VersaoThrift.new(major: 8, minor: 7, revision: 0)
    )
  end

  def send!(uuid, cookie_value: cookie, cnes: ENV.fetch("LEDI_PROOF_CNES"))
    bytes = serializer.serialize(transport(uuid, cnes: cnes))
    reply = client.deliver(cookie: cookie_value, filename: "#{uuid}.esus", bytes: bytes)
    puts "[pec] #{uuid} → #{reply.status} #{reply.body.to_s.truncate(300).inspect}"
    reply
  end

  def new_uuid = "#{ENV.fetch('LEDI_PROOF_CNES')}-#{SecureRandom.uuid}"

  it "aceita a ficha sintética (critério de saída) e responde ao reenvio do mesmo uuid" do
    uuid = new_uuid
    expect(send!(uuid).status).to be_between(200, 299)
    send!(uuid) # duplicate_after_accept: registre status e corpo
  end

  it "recusa ficha com CNES fora do município (formato do 400) e responde ao reenvio do mesmo uuid" do
    uuid = new_uuid
    first = send!(uuid, cnes: "0000000")
    expect(first.status).to eq(400)
    send!(uuid) # resend_after_rejection: mesmo uuid, agora com o CNES certo
  end

  it "sessão inválida: registre o status" do
    send!(new_uuid, cookie_value: "JSESSIONID=invalida")
  end

  it "login com senha errada: registre a exceção (e o status, se Failed)" do
    wrong = Ledi::PecClient.new(base_url: ENV.fetch("LEDI_PEC_URL"), username: ENV.fetch("LEDI_PEC_USERNAME"),
                                password: "senha-errada")
    wrong.login
  rescue Ledi::PecClient::Error => e
    puts "[pec] login recusado → #{e.class.name} #{e.respond_to?(:status) ? e.status : ''}"
    expect(e).to be_a(Ledi::PecClient::Unauthorized)
  end
end
```

- [ ] **Step 12: Rode a prova**

Run (shell com as variáveis do Step 4 exportadas; o CA visto de dentro do container):
```bash
docker compose exec -T -w /rails/.claude/mod16-exporter \
  -e LEDI_PEC_URL=https://pec.rota.test:8443 \
  -e LEDI_PEC_CA_FILE=/rails/.claude/mod16-exporter/deploy/development/pec/local/ca.pem \
  -e LEDI_PEC_USERNAME -e LEDI_PEC_PASSWORD \
  -e LEDI_PROOF_CNES -e LEDI_PROOF_INE -e LEDI_PROOF_CNS -e LEDI_PROOF_CBO -e LEDI_PROOF_IBGE=4115200 \
  api bundle exec rspec spec/integration/ledi_pec_spec.rb --format documentation
```
Expected: o primeiro exemplo PASSA com 2xx.

**Formato do login.** O login da fundação manda JSON (`{ usuario, senha }`); o exemplo oficial manda `multipart/form-data`. Se o `client.login` falhar com `Failed` (400/415) e a credencial estiver certa, troque o corpo do `#login` da fundação para multipart, no mesmo `#post` (campos `usuario` e `senha`, `boundary` como no `#deliver`), e rode de novo. Se o login com senha errada sair como `Failed(400)` (a documentação lista 400 para credencial inválida/inativa), acrescente `400` ao `when 401, 403` do `#login`, para que vire `Unauthorized`; o último exemplo da prova fica verde. Registre as duas decisões no runbook e avise a sessão da fundação. Confira no PEC (menu de transmissão/recebimento de dados e o relatório de procedimentos da UBS no dia) que a ficha aparece e anote o caminho exato da tela. Erro de formato/desserialização → aplique a regra de parada do topo desta task.

Se o PEC exigir o arquivo compactado (resposta de formato ao binário cru, mas aceitando um ZIP com uma entrada `<uuid>.esus`, como na importação por arquivo), **pare e reporte**: o exemplo oficial da API envia cru, e mudar isso muda o `Ledi::PecClient` — decisão do usuário.

- [ ] **Step 13: Grave as observações**

Crie `config/ledi/pec_observations.yml` com o que a prova imprimiu (os valores abaixo são o **formato**; troque cada um pelo observado — `marker` é um trecho curto e estável do corpo do 400 de duplicidade, ou `null` se o reenvio respondeu 2xx; `same_uuid` é `accepted` ou `rejected`; o `example` do 400 vai sem CNS, CPF ou nome):

```yaml
# Prova técnica do módulo 16 (Task 1 do plano api-exporter). Valores OBSERVADOS
# contra o PEC local; o exportador lê este arquivo (Ledi::Observations). Só mude
# com uma nova prova contra um PEC de verdade, registrando a data.
observed_on: "2026-10-07"
pec_version: "5.5.28"
ledi_version: "8.7.0"
idl_commit: "9316f280f5"
success_statuses: [201]
compression: none
rejection:
  status: 400
  example: "Texto do 400 observado, sem dado pessoal"
duplicate_after_accept:
  status: 400
  marker: "trecho estável do corpo"
resend_after_rejection:
  same_uuid: accepted
session_expired_statuses: [401]
```

- [ ] **Step 14: Escreva o runbook no docs**

Crie `operacao/pec-local-dev.md` no worktree do docs com: pré-requisitos (Docker com ≥ 4 GB, emulação amd64 em Mac arm64), os Steps 1–4 como foram executados (com o caminho exato das telas do PEC e do download do CNES que você usou), a geração das classes (Step 6, com o commit escolhido e por quê), o comando da prova (Step 12), onde ver a ficha no PEC, a tabela das respostas observadas (espelho de `pec_observations.yml`) e "o que nunca entra no git" (instalador, chaves, CA, usuário/senha da credencial, CNES/CNS/INE reais). Cite as fontes: `https://integracao.esusaps.bridge.ufsc.tech/ledi/documentacao/api_transmissao/api_transmissao_registro_LEDI.html`, `https://sisaps.saude.gov.br/sistemas/esusaps/docs/manual/APOIO/API_transmissao/`, `https://sisaps.saude.gov.br/sistemas/esusaps/docs/manual/APOIO/Certificado_Https_Linux/`, `https://github.com/laboratoriobridge/esusaps-integracao`.

- [ ] **Step 15: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod16-exporter add .gitignore deploy/development/pec/compose.yml deploy/development/pec/Dockerfile deploy/development/pec/entrypoint.sh bin/ledi-generate vendor/ledi/8.7.0 Gemfile Gemfile.lock app/services/ledi/pec_client.rb config/ledi/pec_observations.yml spec/spec_helper.rb spec/services/ledi/pec_client_deliver_spec.rb spec/services/ledi/pec_client_spec.rb spec/integration/ledi_pec_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod16-exporter commit -m "feat: prove LEDI 8.7.0 Thrift delivery to a local PEC

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
/opt/homebrew/bin/git -C docs/.claude/mod16-exporter add operacao/pec-local-dev.md
/opt/homebrew/bin/git -C docs/.claude/mod16-exporter commit -m "docs: add local PEC runbook for the LEDI technical proof

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Fatia 2 — Layout versionado (F-16.6)

### Task 2: `Ledi::Version`, `Ledi::Observations` e o contrato contra os IDLs

**Files:**
- Create: `app/services/ledi/version.rb`, `app/services/ledi/observations.rb`
- Create: `spec/support/ledi_helpers.rb`; Modify: `spec/rails_helper.rb`
- Test: `spec/services/ledi/contract_spec.rb`, `spec/services/ledi/observations_spec.rb`

**Interfaces:**
- Consumes: `vendor/ledi/8.7.0/` e `config/ledi/pec_observations.yml` (Task 1); nomes da fundação (lista do topo).
- Produces:
  - `Ledi::Version::ACTIVE == "8.7.0"`; `Ledi::Version.load!` (põe `vendor/ledi/<ACTIVE>/gen-rb` no `$LOAD_PATH` e carrega os tipos usados; idempotente); `Ledi::Version.thrift -> Br::Gov::Saude::Esusab::Dadotransp::VersaoThrift`; `Ledi::Version.root -> Pathname`; `Ledi::Version.serialize(struct) -> String (binário)`; `Ledi::Version.deserialize(klass, bytes) -> struct`.
  - `Ledi::Observations.duplicate_marker -> String|nil`; `.resend_uuid_policy -> :same|:new`; `.session_expired_statuses -> Array<Integer>`; `.reload!`.
  - Helpers de spec: `ledi_ready!(city, pec_url:, username: "rota", password: "segredo-ledi", record_mode: "integrated", ibge_code: "4115200")`, `ledi_off!(city)`, `FakePec` (`.for(base_url)`, `.reset!`, `#with_credentials`, `#logins`, `#deliveries`, `#login_replies`, `#delivery_replies`), `stub_pec!` (stuba `Ledi::PecClient.new`), `ledi_maintainer!`.

- [ ] **Step 1: Confira os nomes da fundação**

Run:
```bash
grep -n "def self.enabled?\|def self.usable?\|def enabled?\|def usable?" -r apps/api/.claude/mod16-exporter/app/services/platform
grep -n "record_mode\|pec_url" apps/api/.claude/mod16-exporter/db/platform_schema.rb
grep -n "integration_credentials" -A12 apps/api/.claude/mod16-exporter/db/city_schema.rb
grep -n "city_features" -A8 apps/api/.claude/mod16-exporter/db/platform_schema.rb
grep -n "def login\|class .* < \|Unauthorized\|Unreachable" apps/api/.claude/mod16-exporter/app/services/ledi/pec_client.rb
```
Expected: os nomes da seção "Interfaces consumidas da fundação". Divergência → ajuste só `spec/support/ledi_helpers.rb` (Step 3) e as chamadas afetadas nas tasks seguintes, e registre no relatório.

- [ ] **Step 2: Escreva as specs (falham)**

```ruby
# spec/services/ledi/contract_spec.rb
require "rails_helper"

# Spec §9 "Contrato LEDI": as classes geradas em vendor/ledi/8.7.0 batem campo a
# campo (id, required/optional, tipo, nome) com os IDLs oficiais copiados ao lado
# delas, e os IDLs batem com o sha256 gravado em SOURCE. Um vendor editado à mão
# ou gerado de outro commit fica vermelho aqui.
RSpec.describe "Contrato LEDI 8.7.0" do
  before(:all) { Ledi::Version.load! }

  THRIFT_TYPES = { "string" => :STRING, "binary" => :STRING, "i32" => :I32, "i64" => :I64, "bool" => :BOOL,
                   "double" => :DOUBLE }.freeze

  def idl_fields(file, struct)
    text = Ledi::Version.root.join("idl", file).read
    body = text[/struct\s+#{struct}\s*\{(.*?)\n\}/m, 1] or raise "#{struct} não está em #{file}"
    body.scan(/^\s*(\d+):\s*(required|optional)\s+([\w.<>]+)\s+(\w+)/).to_h do |id, _req, type, name|
      [ id.to_i, [ name, type ] ]
    end
  end

  def generated_fields(klass)
    klass::FIELDS.to_h { |id, spec| [ id, spec[:name] ] }
  end

  {
    [ "transport/dado_transporte.thrift", "DadoTransporteThrift" ] => "Br::Gov::Saude::Esusab::Dadotransp::DadoTransporteThrift",
    [ "transport/dado_transporte.thrift", "DadoInstalacaoThrift" ] => "Br::Gov::Saude::Esusab::Dadotransp::DadoInstalacaoThrift",
    [ "transport/dado_transporte.thrift", "VersaoThrift" ] => "Br::Gov::Saude::Esusab::Dadotransp::VersaoThrift",
    [ "ras/ficha_atendimento_procedimento.thrift", "FichaProcedimentoMasterThrift" ] =>
      "Br::Gov::Saude::Esusab::Ras::Atendprocedimentos::FichaProcedimentoMasterThrift",
    [ "ras/ficha_atendimento_procedimento.thrift", "FichaProcedimentoChildThrift" ] =>
      "Br::Gov::Saude::Esusab::Ras::Atendprocedimentos::FichaProcedimentoChildThrift",
    [ "ras/common.thrift", "UnicaLotacaoHeaderThrift" ] => "Br::Gov::Saude::Esusab::Ras::Common::UnicaLotacaoHeaderThrift"
  }.each do |(file, struct), class_name|
    it "#{class_name.demodulize} tem os mesmos ids e nomes de campo do IDL" do
      expected = idl_fields(file, struct)
      expect(generated_fields(class_name.constantize)).to eq(expected.transform_values(&:first))
      expected.each do |id, (name, type)|
        next unless THRIFT_TYPES.key?(type)
        expect(class_name.constantize::FIELDS[id][:type]).to eq(Thrift::Types.const_get(THRIFT_TYPES[type])), name
      end
    end
  end

  it "os IDLs batem com o sha256 de SOURCE" do
    recorded = Ledi::Version.root.join("SOURCE").read.scan(%r{^\s+\./(\S+\.thrift): (\h{64})$}).to_h
    expect(recorded).not_to be_empty
    recorded.each do |path, sha|
      expect(Digest::SHA256.file(Ledi::Version.root.join("idl", path)).hexdigest).to eq(sha), path
    end
  end

  it "serializa e lê de volta em TBinaryProtocol, com a versão 8.7.0" do
    versao = Ledi::Version.thrift
    expect([ versao.major, versao.minor, versao.revision ]).to eq([ 8, 7, 0 ])
    bytes = Ledi::Version.serialize(versao)
    expect(bytes.encoding).to eq(Encoding::BINARY)
    expect(Ledi::Version.deserialize(versao.class, bytes)).to eq(versao)
  end
end
```

```ruby
# spec/services/ledi/observations_spec.rb
require "rails_helper"

RSpec.describe Ledi::Observations do
  after { described_class.reload! }

  def with_file(yaml)
    allow(File).to receive(:read).and_call_original
    allow(File).to receive(:read).with(described_class::PATH).and_return(yaml)
    described_class.reload!
  end

  it "lê o arquivo da prova técnica" do
    expect(described_class::PATH).to eq(Rails.root.join("config/ledi/pec_observations.yml"))
    expect(described_class.session_expired_statuses).to all(be_an(Integer))
    expect(%i[same new]).to include(described_class.resend_uuid_policy)
  end

  it "duplicidade e política de reenvio vêm do que foi observado" do
    with_file(<<~YAML)
      duplicate_after_accept: { status: 400, marker: "já recebida" }
      resend_after_rejection: { same_uuid: rejected }
      session_expired_statuses: [401, 302]
    YAML
    expect(described_class.duplicate_marker).to eq("já recebida")
    expect(described_class.resend_uuid_policy).to eq(:new)
    expect(described_class.session_expired_statuses).to eq([ 401, 302 ])
  end

  it "reenvio após 2xx sem marcador e mesmo uuid aceito" do
    with_file(<<~YAML)
      duplicate_after_accept: { status: 201, marker: null }
      resend_after_rejection: { same_uuid: accepted }
      session_expired_statuses: [401]
    YAML
    expect(described_class.duplicate_marker).to be_nil
    expect(described_class.resend_uuid_policy).to eq(:same)
  end
end
```

Run: `docker compose exec -T -w /rails/.claude/mod16-exporter api bundle exec rspec spec/services/ledi/contract_spec.rb spec/services/ledi/observations_spec.rb`
Expected: FAIL com `uninitialized constant Ledi::Version`.

- [ ] **Step 3: Implemente**

```ruby
# app/services/ledi/version.rb
# Layout LEDI APS ativo (ADR 0028). As classes Thrift moram em
# vendor/ledi/<versão>/gen-rb, geradas por bin/ledi-generate dos IDLs oficiais;
# o vendor não é autoload do Zeitwerk (o código gerado usa `require` por nome de
# arquivo), então entra no $LOAD_PATH aqui, uma vez. Trocar de versão = gerar a
# pasta nova, rodar spec/services/ledi/contract_spec.rb e mudar ACTIVE.
module Ledi
  module Version
    ACTIVE = "8.7.0"
    TYPES = %w[dado_transporte_types ficha_atendimento_procedimento_types].freeze
    LOCK = Mutex.new

    module_function

    def root = Rails.root.join("vendor/ledi", ACTIVE)

    def load!
      LOCK.synchronize do
        return if @loaded
        gen = root.join("gen-rb").to_s
        $LOAD_PATH.unshift(gen) unless $LOAD_PATH.include?(gen)
        require "thrift"
        TYPES.each { |file| require file }
        @loaded = true
      end
    end

    def thrift
      load!
      major, minor, revision = ACTIVE.split(".").map(&:to_i)
      Br::Gov::Saude::Esusab::Dadotransp::VersaoThrift.new(major: major, minor: minor, revision: revision)
    end

    def serialize(struct)
      load!
      Thrift::Serializer.new(Thrift::BinaryProtocolFactory.new).serialize(struct).b
    end

    def deserialize(klass, bytes)
      load!
      Thrift::Deserializer.new(Thrift::BinaryProtocolFactory.new).deserialize(klass.new, bytes)
    end
  end
end
```

```ruby
# app/services/ledi/observations.rb
# O que a prova técnica observou no PEC (config/ledi/pec_observations.yml, Task 1
# do plano api-exporter). O exportador decide duplicidade, reenvio e sessão
# expirada por aqui, não por suposição.
module Ledi
  module Observations
    PATH = Rails.root.join("config/ledi/pec_observations.yml")

    module_function

    def data = @data ||= YAML.safe_load(File.read(PATH)) || {}

    def reload! = @data = nil

    def duplicate_marker = data.dig("duplicate_after_accept", "marker").presence

    def resend_uuid_policy = data.dig("resend_after_rejection", "same_uuid") == "accepted" ? :same : :new

    def session_expired_statuses = Array(data["session_expired_statuses"]).map(&:to_i)
  end
end
```

```ruby
# spec/support/ledi_helpers.rb
# Exportador LEDI (módulo 16): deixa uma cidade pronta para exportar com os
# nomes da fundação (interruptor, modo, endereço do PEC, IBGE, credencial) e
# troca o PEC por um falso que registra o que recebeu. Se a fundação mudar um
# nome, o ajuste é AQUI (Task 2, Step 1).
class FakePec
  attr_reader :base_url, :logins, :deliveries
  attr_accessor :login_replies, :delivery_replies

  def self.registry = (@registry ||= {})
  def self.for(base_url) = registry[base_url] ||= new(base_url)
  def self.reset! = registry.clear

  def initialize(base_url)
    @base_url = base_url
    @logins = []
    @deliveries = []
    @login_replies = []
    @delivery_replies = []
  end

  # O stub de Ledi::PecClient.new devolve este objeto com as credenciais do
  # cliente construído (o falso é um por endereço; a credencial é a da vez).
  def with_credentials(username, password)
    @username, @password = username, password
    self
  end

  # login_replies: :ok (padrão) ou uma classe de erro do Ledi::PecClient.
  def login
    @logins << { username: @username, password: @password }
    reply = @login_replies.shift || :ok
    raise reply, "fake" unless reply == :ok

    Ledi::PecClient::Session.new(cookie: "JSESSIONID=fake-#{@logins.size}")
  end

  # delivery_replies: [status, corpo] (padrão [201, ""]) ou uma classe de erro.
  def deliver(cookie:, filename:, bytes:)
    @deliveries << { cookie: cookie, filename: filename, bytes: bytes }
    reply = @delivery_replies.shift || [ 201, "" ]
    raise reply, "fake" if reply.is_a?(Class)

    Ledi::PecClient::Reply.new(status: reply[0], body: reply[1])
  end
end

module LediHelpers
  def stub_pec!
    allow(Ledi::PecClient).to receive(:new) do |base_url:, username:, password:, **|
      FakePec.for(base_url).with_credentials(username, password)
    end
  end

  def ledi_maintainer!
    Maintainer.find_by(email_address: "ledi-mantenedor@rotasaude.app") ||
      Maintainer.create!(email_address: "ledi-mantenedor@rotasaude.app", password: "s3nha-forte-1",
                         otp_secret: ROTP::Base32.random, otp_enabled: true)
  end

  def ledi_admin!
    User.find_by(email_address: "ledi-admin@cidade.gov.br") || staff_with("ledi-admin@cidade.gov.br", "municipal_admin")
  end

  # Liga ledi_export para `city` (City da plataforma) e prepara o banco DELA.
  def ledi_ready!(city, pec_url:, username: "rota", password: "segredo-ledi", record_mode: "integrated",
                  ibge_code: "4115200")
    city.update!(record_mode: record_mode, pec_url: pec_url)
    Platform::Features.set!(city: city, key: "ledi_export", enabled: true, maintainer: ledi_maintainer!)
    CityConnection.with(city) do
      profile = CityProfile.current || CityProfile.create!(name: city.name)
      profile.update!(ibge_code: ibge_code)
      credential = IntegrationCredential.find_or_initialize_by(kind: "ledi")
      credential.update!(secret: { "username" => username, "password" => password }, set_by_user: ledi_admin!,
                         set_at: Time.current, last_check_status: "ok", last_check_at: Time.current)
    end
    city
  end

  def ledi_off!(city)
    Platform::Features.set!(city: city, key: "ledi_export", enabled: false, maintainer: ledi_maintainer!)
  end
end

RSpec.configure do |config|
  config.include LediHelpers
  config.before do
    FakePec.reset!
    Ledi::SessionCache.clear! if defined?(Ledi::SessionCache)
  end
end
```

Em `spec/rails_helper.rb`, depois de `require_relative "support/lock_wait"`:

```ruby
require_relative "support/ledi_helpers"
```

Run: `docker compose exec -T -w /rails/.claude/mod16-exporter api bundle exec rspec spec/services/ledi/contract_spec.rb spec/services/ledi/observations_spec.rb`
Expected: PASS. Se um campo divergir, o vendor não é o dos IDLs: rode `bin/ledi-generate` de novo, nunca edite o vendor.

- [ ] **Step 4: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod16-exporter add app/services/ledi/version.rb app/services/ledi/observations.rb spec/support/ledi_helpers.rb spec/rails_helper.rb spec/services/ledi/contract_spec.rb spec/services/ledi/observations_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod16-exporter commit -m "feat: load the versioned LEDI layout and check it against the official IDLs

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 3: `Ledi::Transport.wrap` (camada de transporte)

**Files:**
- Create: `config/ledi.yml`, `app/services/ledi/sender.rb`, `app/services/ledi/ficha_types.rb`, `app/services/ledi/transport.rb`
- Test: `spec/services/ledi/transport_spec.rb`

**Interfaces:**
- Consumes: `Ledi::Version.load!/.thrift/.serialize/.deserialize` (Task 2); `CityProfile.current.ibge_code`.
- Produces:
  - `Ledi::FichaTypes.code(type) -> Integer` (`"procedimento"` → 7); `.klass(type) -> Class`; `.uuid_field(type) -> Symbol`; `Ledi::FichaTypes::UnknownType`.
  - `Ledi::Sender.installation(city) -> DadoInstalacaoThrift`; `Ledi::Sender.software_version -> String`; `Ledi::Sender::Missing`.
  - `Ledi::Transport.wrap(ficha, city:, uuid:) -> String` (binário do `DadoTransporteThrift`); `Ledi::Transport.rewrap(bytes, uuid:) -> String` (troca `uuidDadoSerializado` e o uuid da ficha interna); `Ledi::Transport.read(bytes) -> DadoTransporteThrift`; `Ledi::Transport::IbgeMissing`.
  - Uma ficha passada a `wrap` responde a `type`, `cnes`, `ine` e `to_thrift(uuid:)` (interface completa na Task 4).

- [ ] **Step 1: Escreva a spec (falha)**

```ruby
# spec/services/ledi/transport_spec.rb
require "rails_helper"

# Spec §6.2: DadoTransporteThrift com uuid CNES-UUID, tipo, CNES, codIbge (do
# city_profile da cidade), INE, remetente/originadora com contraChave
# "Rota Saúde - <versão>", uuidInstalacao estável por cidade, CNPJ do
# responsável e a versão 8.7.0.
RSpec.describe Ledi::Transport do
  let(:city) { register_test_city! }
  let(:uuid) { "1234567-#{SecureRandom.uuid}" }
  let(:ficha) do
    Ledi::Fichas::Synthetic.new(cnes: "1234567", ine: "0000123456", professional_cns: "700000000000005",
                                cbo: "225142", attended_at: Time.zone.parse("2026-10-06 10:00"),
                                source_id: SecureRandom.uuid)
  end

  before { CityProfile.create!(name: "Maringá", ibge_code: "4115200") }

  it "monta o transporte com a ficha serializada dentro" do
    transport = described_class.read(described_class.wrap(ficha, city: city, uuid: uuid))

    expect(transport.uuidDadoSerializado).to eq(uuid)
    expect(transport.tipoDadoSerializado).to eq(7)
    expect(transport.cnesDadoSerializado).to eq("1234567")
    expect(transport.codIbge).to eq("4115200")
    expect(transport.ineDadoSerializado).to eq("0000123456")
    expect(transport.numLote).to be_nil
    expect([ transport.versao.major, transport.versao.minor, transport.versao.revision ]).to eq([ 8, 7, 0 ])
    inner = Ledi::Version.deserialize(Ledi::FichaTypes.klass("procedimento"), transport.dadoSerializado)
    expect(inner.uuidFicha).to eq(uuid)
  end

  it "remetente e originadora identificam o Rota Saúde com uuid estável por cidade" do
    transport = described_class.read(described_class.wrap(ficha, city: city, uuid: uuid))
    again = described_class.read(described_class.wrap(ficha, city: city, uuid: "1234567-#{SecureRandom.uuid}"))
    other = described_class.read(described_class.wrap(ficha, city: build(:city, id: SecureRandom.uuid), uuid: uuid))

    expect(transport.remetente).to eq(transport.originadora)
    expect(transport.remetente.contraChave).to eq("Rota Saúde - #{Ledi::Sender.software_version}")
    expect(transport.remetente.cpfOuCnpj).to eq("11222333000181")
    expect(transport.remetente.uuidInstalacao).to eq(again.remetente.uuidInstalacao)
    expect(transport.remetente.uuidInstalacao).not_to eq(other.remetente.uuidInstalacao)
  end

  it "rewrap troca o uuid do transporte e da ficha, mantendo o resto" do
    bytes = described_class.wrap(ficha, city: city, uuid: uuid)
    fresh = "1234567-#{SecureRandom.uuid}"
    transport = described_class.read(described_class.rewrap(bytes, uuid: fresh))

    expect(transport.uuidDadoSerializado).to eq(fresh)
    expect(Ledi::Version.deserialize(Ledi::FichaTypes.klass("procedimento"), transport.dadoSerializado).uuidFicha)
      .to eq(fresh)
    expect(transport.cnesDadoSerializado).to eq("1234567")
  end

  it "sem IBGE no city_profile: IbgeMissing" do
    CityProfile.current.update!(ibge_code: nil)
    expect { described_class.wrap(ficha, city: city, uuid: uuid) }.to raise_error(Ledi::Transport::IbgeMissing)
  end

  it "remetente sem CNPJ configurado: Sender::Missing" do
    allow(Rails.application).to receive(:config_for).and_call_original
    allow(Rails.application).to receive(:config_for).with(:ledi).and_return({ sender_cnpj: "", sender_name: "" })
    expect { described_class.wrap(ficha, city: city, uuid: uuid) }.to raise_error(Ledi::Sender::Missing)
  end

  it "tipo desconhecido: UnknownType" do
    expect { Ledi::FichaTypes.code("vacinacao") }.to raise_error(Ledi::FichaTypes::UnknownType)
  end
end
```

Run: `docker compose exec -T -w /rails/.claude/mod16-exporter api bundle exec rspec spec/services/ledi/transport_spec.rb`
Expected: FAIL com `uninitialized constant Ledi::Transport` (a Task 4 cria `Ledi::Fichas::Synthetic`; até lá o erro pode ser esse).

- [ ] **Step 2: Implemente**

```yaml
# config/ledi.yml
# Quem envia as fichas LEDI (DadoInstalacaoThrift, ADR 0028): o CNPJ e o nome
# do responsável pelo software. development/test usam um CNPJ fictício com dígito
# válido; publicados leem do ambiente (vazio → Ledi::Sender::Missing, o envio
# falha visível no job e nada sai com remetente em branco).
shared:
  sender_email: <%= ENV.fetch("LEDI_SENDER_EMAIL", "") %>
  sender_phone: <%= ENV.fetch("LEDI_SENDER_PHONE", "") %>

development:
  sender_cnpj: "11222333000181"
  sender_name: "Rota Saúde (desenvolvimento)"

test:
  sender_cnpj: "11222333000181"
  sender_name: "Rota Saúde (teste)"

staging:
  sender_cnpj: <%= ENV.fetch("LEDI_SENDER_CNPJ", "") %>
  sender_name: <%= ENV.fetch("LEDI_SENDER_NAME", "") %>

production:
  sender_cnpj: <%= ENV.fetch("LEDI_SENDER_CNPJ", "") %>
  sender_name: <%= ENV.fetch("LEDI_SENDER_NAME", "") %>
```

```ruby
# app/services/ledi/ficha_types.rb
# Tipos de ficha que o exportador conhece: tipoDadoSerializado (camada de
# transporte), a classe Thrift mestre e o campo do uuid da ficha. Ficha nova
# (módulos 18, 19, 22, 24) entra aqui.
module Ledi
  module FichaTypes
    class UnknownType < StandardError; end

    REGISTRY = {
      "procedimento" => { code: 7, klass: "Br::Gov::Saude::Esusab::Ras::Atendprocedimentos::FichaProcedimentoMasterThrift",
                          uuid_field: :uuidFicha }
    }.freeze

    module_function

    def entry(type) = REGISTRY.fetch(type.to_s) { raise UnknownType, type.to_s }
    def code(type) = entry(type)[:code]
    def uuid_field(type) = entry(type)[:uuid_field]

    def klass(type)
      Ledi::Version.load!
      entry(type)[:klass].constantize
    end

    def type_for_code(code)
      REGISTRY.find { |_type, e| e[:code] == code }&.first or raise UnknownType, code.to_s
    end
  end
end
```

```ruby
# app/services/ledi/sender.rb
# Remetente e originadora do transporte LEDI: o software Rota Saúde, numa
# "instalação" por cidade (uuidInstalacao estável, derivado do id da cidade).
module Ledi
  module Sender
    class Missing < StandardError; end

    NAMESPACE = "6f1c3d1e-2b7a-5c4e-9a8d-0e5f4b3a2c10"
    SOFTWARE = "Rota Saúde"

    module_function

    def software_version = ENV.fetch("APP_VERSION", "dev")

    def installation(city)
      Ledi::Version.load!
      config = Rails.application.config_for(:ledi)
      cnpj, name = config[:sender_cnpj].to_s, config[:sender_name].to_s
      raise Missing, "LEDI_SENDER_CNPJ/LEDI_SENDER_NAME ausentes" if cnpj.blank? || name.blank?

      Br::Gov::Saude::Esusab::Dadotransp::DadoInstalacaoThrift.new(
        contraChave: "#{SOFTWARE} - #{software_version}",
        uuidInstalacao: Digest::UUID.uuid_v5(NAMESPACE, city.id.to_s),
        cpfOuCnpj: cnpj, nomeOuRazaoSocial: name,
        email: config[:sender_email].presence, fone: config[:sender_phone].presence,
        versaoSistema: software_version
      )
    end
  end
end
```

```ruby
# app/services/ledi/transport.rb
# Camada de transporte LEDI (spec §6.2): embrulha a ficha serializada num
# DadoTransporteThrift e devolve o binário que vai para o PEC. O codIbge vem do
# city_profile da cidade (fonte única; decisão do coordenador do módulo 16).
# Precisa rodar na conexão da cidade (CityConnection.with / request / job).
module Ledi
  module Transport
    class IbgeMissing < StandardError; end

    module_function

    def wrap(ficha, city:, uuid:)
      Ledi::Version.load!
      ibge = CityProfile.current&.ibge_code
      raise IbgeMissing, "city_profile sem ibge_code" if ibge.blank?

      sender = Ledi::Sender.installation(city)
      transport = Br::Gov::Saude::Esusab::Dadotransp::DadoTransporteThrift.new(
        uuidDadoSerializado: uuid,
        tipoDadoSerializado: Ledi::FichaTypes.code(ficha.type),
        cnesDadoSerializado: ficha.cnes,
        codIbge: ibge,
        ineDadoSerializado: ficha.ine.presence,
        dadoSerializado: Ledi::Version.serialize(ficha.to_thrift(uuid: uuid)),
        remetente: sender,
        originadora: sender,
        versao: Ledi::Version.thrift
      )
      Ledi::Version.serialize(transport)
    end

    def read(bytes) = Ledi::Version.deserialize(Br::Gov::Saude::Esusab::Dadotransp::DadoTransporteThrift, bytes)

    # Reenvio com uuid novo (Ledi::Resend, quando a prova mandar): o uuid vive
    # no transporte E dentro da ficha; os dois mudam juntos.
    def rewrap(bytes, uuid:)
      transport = read(bytes)
      type = Ledi::FichaTypes.type_for_code(transport.tipoDadoSerializado)
      inner = Ledi::Version.deserialize(Ledi::FichaTypes.klass(type), transport.dadoSerializado)
      inner.public_send(:"#{Ledi::FichaTypes.uuid_field(type)}=", uuid)
      transport.uuidDadoSerializado = uuid
      transport.dadoSerializado = Ledi::Version.serialize(inner)
      Ledi::Version.serialize(transport)
    end
  end
end
```

- [ ] **Step 3: Commit (o teste passa na Task 4)**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod16-exporter add config/ledi.yml app/services/ledi/sender.rb app/services/ledi/ficha_types.rb app/services/ledi/transport.rb spec/services/ledi/transport_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod16-exporter commit -m "feat: wrap LEDI fichas in the transport layer with the city's IBGE code

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 4: Interface `Ledi::Ficha` e `Ledi::Fichas::Synthetic`

**Files:**
- Create: `app/services/ledi/ficha.rb`, `app/services/ledi/fichas/synthetic.rb`
- Test: `spec/services/ledi/fichas/synthetic_spec.rb` (e a `transport_spec.rb` da Task 3 passa)

**Interfaces:**
- Consumes: `Ledi::FichaTypes`, `Ledi::Version` (Tasks 2–3).
- Produces:
  - `Ledi::Ficha::METHODS == %i[type competence cnes ine to_thrift source]`; `Ledi::Ficha.assert!(ficha)` (levanta `Ledi::Ficha::Invalid` se faltar método, `competence` não for `AAAAMM`, `cnes` não tiver 7 dígitos, `ine` não for `nil`/10 dígitos ou `source` não for `{ type: String, id: String }`).
  - `Ledi::Fichas::Synthetic.new(cnes:, ine:, professional_cns:, cbo:, attended_at:, source_id: SecureRandom.uuid)`; `#type == "procedimento"`; `#competence` (`attended_at` no fuso corrente, `%Y%m`); `#source == { type: "synthetic", id: source_id }`; `#to_thrift(uuid:)`; `Ledi::Fichas::Synthetic::NotAllowed`; `Ledi::Fichas::Synthetic::PROCEDURE == "0301100039"`; `CITIZEN_CPF == "12345678909"`.

- [ ] **Step 1: Escreva a spec (falha)**

```ruby
# spec/services/ledi/fichas/synthetic_spec.rb
require "rails_helper"

# Spec §6.2: a interface Ledi::Ficha e a ficha sintética (só dev/test), uma
# Ficha de Procedimentos com aferição de PA (SIGTAP 0301100039) de um cidadão
# fictício. Nenhuma ficha real nasce no módulo 16.
RSpec.describe Ledi::Fichas::Synthetic do
  let(:attended_at) { Time.zone.parse("2026-10-06 10:00") }
  let(:ficha) do
    described_class.new(cnes: "1234567", ine: "0000123456", professional_cns: "700000000000005", cbo: "225142",
                        attended_at: attended_at, source_id: "f0f0f0f0-0000-4000-8000-000000000001")
  end

  it "cumpre a interface Ledi::Ficha" do
    expect { Ledi::Ficha.assert!(ficha) }.not_to raise_error
    expect(ficha.type).to eq("procedimento")
    expect(ficha.competence).to eq("202610")
    expect(ficha.source).to eq(type: "synthetic", id: "f0f0f0f0-0000-4000-8000-000000000001")
  end

  it "monta a Ficha de Procedimentos com cabeçalho e um atendimento" do
    master = ficha.to_thrift(uuid: "1234567-u")
    expect(master.uuidFicha).to eq("1234567-u")
    expect(master.tpCdsOrigem).to eq(3)
    header = master.headerTransport
    expect([ header.profissionalCNS, header.cboCodigo_2002, header.cnes, header.ine ])
      .to eq(%w[700000000000005 225142 1234567 0000123456])
    expect(header.dataAtendimento).to eq((attended_at.to_f * 1000).to_i)
    child = master.atendProcedimentos.sole
    expect(child.procedimentos).to eq([ "0301100039" ])
    expect(child.cpfCidadao).to eq("12345678909")
    expect(CitizenIdentity::Cpf.valid?(child.cpfCidadao)).to be(true)
    expect { master.validate }.not_to raise_error
  end

  # Review Focus 4: a competência é a do fuso da cidade, não a de UTC.
  it "competência no fuso da cidade: 23h30 de 31/10 em Manaus é 202610" do
    Time.use_zone("America/Manaus") do
      late = Time.zone.parse("2026-10-31 23:30")
      expect(late.utc.month).to eq(11)
      expect(described_class.new(cnes: "1234567", ine: nil, professional_cns: "700000000000005", cbo: "225142",
                                 attended_at: late).competence).to eq("202610")
    end
  end

  it "fora de development/test: NotAllowed" do
    allow(Rails.env).to receive(:local?).and_return(false)
    expect { ficha }.to raise_error(described_class::NotAllowed)
  end

  it "Ledi::Ficha.assert! recusa objeto incompleto ou com identificador malformado" do
    expect { Ledi::Ficha.assert!(Object.new) }.to raise_error(Ledi::Ficha::Invalid, /type/)
    bad = described_class.new(cnes: "123", ine: nil, professional_cns: "700000000000005", cbo: "225142",
                              attended_at: attended_at)
    expect { Ledi::Ficha.assert!(bad) }.to raise_error(Ledi::Ficha::Invalid, /cnes/)
    bad_ine = described_class.new(cnes: "1234567", ine: "12", professional_cns: "700000000000005", cbo: "225142",
                                  attended_at: attended_at)
    expect { Ledi::Ficha.assert!(bad_ine) }.to raise_error(Ledi::Ficha::Invalid, /ine/)
  end
end
```

Run: `docker compose exec -T -w /rails/.claude/mod16-exporter api bundle exec rspec spec/services/ledi/fichas/synthetic_spec.rb`
Expected: FAIL com `uninitialized constant Ledi::Fichas`.

Se `CitizenIdentity::Cpf.valid?` não existir com esse nome, use o validador de CPF que `app/services/citizen_identity.rb` expõe (confira com `grep -n "def self" app/services/citizen_identity.rb`).

- [ ] **Step 2: Implemente**

```ruby
# app/services/ledi/ficha.rb
# Interface de ficha LEDI (spec §6.2; ADR 0028, "Consequências"): os módulos 18,
# 19, 22 e 24 produzem fichas que respondem a estes métodos. Ledi::Enqueue chama
# assert! antes de gravar.
#   type        -> String   chave de Ledi::FichaTypes ("procedimento")
#   competence  -> String   "AAAAMM" do atendimento, no fuso da cidade
#   cnes        -> String   7 dígitos
#   ine         -> String|nil 10 dígitos
#   to_thrift(uuid:) -> struct Thrift mestre da ficha, com o uuid dado
#   source      -> { type: String, id: String } (o registro que gerou a ficha)
module Ledi
  module Ficha
    class Invalid < StandardError; end

    METHODS = %i[type competence cnes ine to_thrift source].freeze

    module_function

    def assert!(ficha)
      missing = METHODS.reject { |m| ficha.respond_to?(m) }
      raise Invalid, "faltam: #{missing.join(', ')}" if missing.any?
      raise Invalid, "competence" unless ficha.competence.to_s.match?(/\A\d{4}(0[1-9]|1[0-2])\z/)
      raise Invalid, "cnes" unless ficha.cnes.to_s.match?(/\A\d{7}\z/)
      raise Invalid, "ine" unless ficha.ine.nil? || ficha.ine.to_s.match?(/\A\d{10}\z/)

      source = ficha.source
      unless source.is_a?(Hash) && source[:type].is_a?(String) && source[:id].is_a?(String)
        raise Invalid, "source"
      end
      Ledi::FichaTypes.code(ficha.type)
      ficha
    end
  end
end
```

```ruby
# app/services/ledi/fichas/synthetic.rb
# Ficha sintética (spec §6.1/§6.2): Ficha de Procedimentos com uma aferição de
# PA de um cidadão fictício (CPF de teste com dígito válido). Serve à prova
# técnica, às specs e à semente de dev; NUNCA existe fora de development/test.
module Ledi
  module Fichas
    class Synthetic
      class NotAllowed < StandardError; end

      PROCEDURE = "0301100039"          # SIGTAP 03.01.10.003-9 aferição de pressão arterial
      CITIZEN_CPF = "12345678909"
      CITIZEN_BIRTH = Date.new(1980, 5, 10)
      ORIGIN_THIRD_PARTY = 3            # tpCdsOrigem: sistema de terceiro

      attr_reader :cnes, :ine, :professional_cns, :cbo, :attended_at, :source_id

      def initialize(cnes:, ine:, professional_cns:, cbo:, attended_at:, source_id: SecureRandom.uuid)
        raise NotAllowed, "ficha sintética só em development/test" unless Rails.env.local?

        @cnes, @ine, @professional_cns, @cbo = cnes, ine, professional_cns, cbo
        @attended_at = attended_at.in_time_zone
        @source_id = source_id
      end

      def type = "procedimento"
      def competence = attended_at.strftime("%Y%m")
      def source = { type: "synthetic", id: source_id }

      def to_thrift(uuid:)
        Ledi::Version.load!
        ras = Br::Gov::Saude::Esusab::Ras
        header = ras::Common::UnicaLotacaoHeaderThrift.new(
          profissionalCNS: professional_cns, cboCodigo_2002: cbo, cnes: cnes, ine: ine,
          dataAtendimento: ms(attended_at), codigoIbgeMunicipio: CityProfile.current&.ibge_code
        )
        child = ras::Atendprocedimentos::FichaProcedimentoChildThrift.new(
          dtNascimento: ms(CITIZEN_BIRTH.in_time_zone), sexo: 1, localAtendimento: 1, turno: 1,
          cpfCidadao: CITIZEN_CPF, stCidadaoNaoPossuiCpf: false, procedimentos: [ PROCEDURE ],
          dataHoraInicialAtendimento: ms(attended_at), dataHoraFinalAtendimento: ms(attended_at + 10.minutes)
        )
        ras::Atendprocedimentos::FichaProcedimentoMasterThrift.new(
          uuidFicha: uuid, tpCdsOrigem: ORIGIN_THIRD_PARTY, headerTransport: header, atendProcedimentos: [ child ]
        )
      end

      private

      def ms(time) = (time.to_f * 1000).to_i
    end
  end
end
```

Run: `docker compose exec -T -w /rails/.claude/mod16-exporter api bundle exec rspec spec/services/ledi/fichas/synthetic_spec.rb spec/services/ledi/transport_spec.rb`
Expected: PASS.

- [ ] **Step 3: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod16-exporter add app/services/ledi/ficha.rb app/services/ledi/fichas/synthetic.rb spec/services/ledi/fichas/synthetic_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod16-exporter commit -m "feat: add the Ledi::Ficha interface and the dev-only synthetic ficha

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Fatia 3 — Fila e envio (F-16.6)

### Task 5: `ledi_outbox` (migração de cidade, modelo, trigger, cifra)

**Files:**
- Create: `db/city_migrate/20261007100001_create_ledi_outbox.rb`
- Modify: `db/city_schema.rb`, `db/city_triggers.sql`, `app/services/city_encryption.rb`
- Create: `app/models/ledi_outbox_entry.rb`
- Test: `spec/models/ledi_outbox_entry_spec.rb`, `spec/models/create_ledi_outbox_migration_spec.rb`

**Interfaces:**
- Produces:
  - Tabela `ledi_outbox` (colunas no Step 3); `LediOutboxEntry` (`self.table_name = "ledi_outbox"`), `STATUSES`, `encrypts :payload`; `#bytes -> String|nil` e `#bytes=(String)` (Base64 por dentro); escopo `for_competence(competence)`.
  - `LediOutboxEntry.claim!(limit:) -> Array<LediOutboxEntry>` (pending vencidas → `sending`, `SKIP LOCKED`); `.mark_sending(ids)`; `.release!(ids)` (sending → pending, sem tentativa); `.release_stale!(before:) -> Integer`.
  - `#accept!`, `#reject!(message)`, `#retry_later!(error:, now: Time.current)` (Task 7 define a espera; aqui recebe `wait:`). Assinatura final: `#retry_later!(error:, wait:, give_up_after:, now: Time.current)`.

- [ ] **Step 1: Confira o número da migração**

Run: `ls apps/api/.claude/mod16-exporter/db/city_migrate | tail -3`
Expected: a última é da fundação (data 20261005/20261006). Se houver uma igual ou maior que `20261007100001`, use um número maior e ajuste-o nos Steps 3, 5 e 6 e no `define(version:)`.

- [ ] **Step 2: Escreva as specs (falham)**

```ruby
# spec/models/ledi_outbox_entry_spec.rb
require "rails_helper"

# Spec §6.3 e ADR 0028 ("O conteúdo serializado de ficha aceita não existe mais
# na fila"): accepted é imutável e sem payload; payload só vai a nulo; a
# identidade da ficha nunca muda; o uuid só muda no reenvio de uma recusada.
RSpec.describe LediOutboxEntry do
  def entry!(**attrs)
    described_class.create!({ uuid: "1234567-#{SecureRandom.uuid}", ficha_type: "procedimento", competence: "202610",
                              source_type: "synthetic", source_id: SecureRandom.uuid, ledi_version: "8.7.0",
                              next_attempt_at: Time.current, bytes: "\x0B\x01".b }.merge(attrs))
  end

  it "guarda o binário cifrado (Base64 por dentro) e lê de volta" do
    entry = entry!
    raw = described_class.connection.select_value("SELECT payload FROM ledi_outbox WHERE id = #{described_class.connection.quote(entry.id)}")
    expect(raw).not_to include(Base64.strict_encode64("\x0B\x01".b))
    expect(entry.reload.bytes).to eq("\x0B\x01".b)
  end

  it "accepted exige payload nulo e não muda mais; DELETE de accepted é recusado" do
    entry = entry!
    expect { entry.update_columns(status: "accepted", accepted_at: Time.current) }
      .to raise_error(ActiveRecord::StatementInvalid, /ck_ledi_outbox_accepted_payload/)
    entry.accept!
    expect(entry.reload.payload).to be_nil
    expect { entry.update_columns(last_error: "x") }.to raise_error(ActiveRecord::StatementInvalid, /accepted is immutable/)
    expect { entry.delete }.to raise_error(ActiveRecord::StatementInvalid, /accepted is immutable/)
  end

  it "payload nulo não volta; identidade não muda; uuid só muda de rejected para pending" do
    entry = entry!
    entry.update_columns(payload: nil)
    expect { entry.update_columns(payload: "x") }.to raise_error(ActiveRecord::StatementInvalid, /payload only goes to null/)

    other = entry!
    expect { other.update_columns(competence: "202611") }.to raise_error(ActiveRecord::StatementInvalid, /identity/)
    expect { other.update_columns(uuid: "1234567-x") }.to raise_error(ActiveRecord::StatementInvalid, /uuid/)
    other.reject!("CNES inválido")
    other.update!(status: "pending", uuid: "1234567-#{SecureRandom.uuid}")
    expect(other.reload.status).to eq("pending")
  end

  it "CHECKs: status, competência, accepted_at e uuid únicos por fonte" do
    expect { entry!(status: "lost") }.to raise_error(ActiveRecord::StatementInvalid, /ck_ledi_outbox_status/)
    expect { entry!(competence: "202613") }.to raise_error(ActiveRecord::StatementInvalid, /ck_ledi_outbox_competence/)
    source = SecureRandom.uuid
    entry!(source_id: source)
    expect { entry!(source_id: source) }.to raise_error(ActiveRecord::RecordNotUnique)
  end

  it "claim! marca sending e grava a primeira tentativa; release! devolve sem tocar em attempts" do
    due = entry!(next_attempt_at: 1.minute.ago)
    later = entry!(next_attempt_at: 1.hour.from_now)
    claimed = described_class.claim!(limit: 10)
    expect(claimed.map(&:id)).to eq([ due.id ])
    expect(due.reload.status).to eq("sending")
    expect(due.first_attempt_at).to be_present
    expect(later.reload.status).to eq("pending")

    described_class.release!([ due.id ])
    expect(due.reload.slice(:status, :attempts)).to eq("status" => "pending", "attempts" => 0)
  end

  it "release_stale! devolve só o sending parado há mais do limite" do
    stale = entry!.tap { |e| e.update_columns(status: "sending", updated_at: 11.minutes.ago) }
    fresh = entry!.tap { |e| e.update_columns(status: "sending", updated_at: 1.minute.ago) }
    expect(described_class.release_stale!(before: 10.minutes.ago)).to eq(1)
    expect([ stale.reload.status, fresh.reload.status ]).to eq(%w[pending sending])
  end
end
```

```ruby
# spec/models/create_ledi_outbox_migration_spec.rb
require "rails_helper"
require Rails.root.join("db/city_migrate/20261007100001_create_ledi_outbox.rb").to_s

RSpec.describe "Migração de cidade 20261007100001 (CreateLediOutbox): down e up" do
  def conn = ApplicationRecord.connection

  def migrate(direction)
    ActiveRecord::Migration.suppress_messages { CreateLediOutbox.new.exec_migration(conn, direction) }
    LediOutboxEntry.reset_column_information
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

  it "down remove a tabela; up seguinte restaura o schema idêntico" do
    ApplicationRecord.transaction(requires_new: true) do
      before = fingerprint
      migrate(:down)
      expect(conn.table_exists?(:ledi_outbox)).to be(false)
      migrate(:up)
      expect(fingerprint).to eq(before)
      raise ActiveRecord::Rollback
    end
  ensure
    LediOutboxEntry.reset_column_information
  end
end
```

Run: `docker compose exec -T -w /rails/.claude/mod16-exporter api bundle exec rspec spec/models/ledi_outbox_entry_spec.rb`
Expected: FAIL com `uninitialized constant LediOutboxEntry`.

- [ ] **Step 3: Escreva a migração**

```ruby
# db/city_migrate/20261007100001_create_ledi_outbox.rb
# Fila de saída LEDI da cidade (ADR 0028; spec 2026-10-05 §6.3). Cada ficha
# entra já serializada (DadoTransporteThrift) e cifrada com a chave da cidade;
# o conteúdo é apagado quando o PEC aceita. Trigger em db/city_triggers.sql.
class CreateLediOutbox < ActiveRecord::Migration[8.1]
  def up
    create_table :ledi_outbox, id: :uuid do |t|
      t.string :uuid, limit: 44, null: false
      t.string :ficha_type, null: false
      t.string :competence, limit: 6, null: false
      t.string :source_type, null: false
      t.uuid :source_id, null: false
      t.string :status, null: false, default: "pending"
      t.integer :attempts, null: false, default: 0
      t.datetime :next_attempt_at, null: false
      t.datetime :first_attempt_at
      t.string :last_error, limit: 500
      t.string :ledi_version, null: false
      t.text :payload
      t.datetime :accepted_at
      t.timestamps
    end
    add_index :ledi_outbox, :uuid, unique: true, name: "idx_ledi_outbox_uuid"
    add_index :ledi_outbox, %i[source_type source_id ficha_type], unique: true, name: "idx_ledi_outbox_source"
    add_index :ledi_outbox, %i[status next_attempt_at], name: "idx_ledi_outbox_due"
    add_index :ledi_outbox, %i[competence status], name: "idx_ledi_outbox_competence"
    add_check_constraint :ledi_outbox,
                         "status::text = ANY (ARRAY['pending'::text, 'sending'::text, 'accepted'::text, " \
                         "'rejected'::text, 'failed'::text])",
                         name: "ck_ledi_outbox_status"
    add_check_constraint :ledi_outbox, "competence::text ~ '^[0-9]{4}(0[1-9]|1[0-2])$'::text",
                         name: "ck_ledi_outbox_competence"
    add_check_constraint :ledi_outbox, "ficha_type::text ~ '^[a-z_]+$'::text", name: "ck_ledi_outbox_ficha_type"
    add_check_constraint :ledi_outbox, "(status::text = 'accepted'::text) = (accepted_at IS NOT NULL)",
                         name: "ck_ledi_outbox_accepted_at"
    add_check_constraint :ledi_outbox, "status::text <> 'accepted'::text OR payload IS NULL",
                         name: "ck_ledi_outbox_accepted_payload"
    add_check_constraint :ledi_outbox, "status::text <> 'rejected'::text OR last_error IS NOT NULL",
                         name: "ck_ledi_outbox_rejected_error"

    execute File.read(Rails.root.join("db/city_triggers.sql"))
  end

  def down
    drop_table :ledi_outbox
    execute "DROP FUNCTION IF EXISTS rota_ledi_outbox_guard()"
  end
end
```

Ao fim de `db/city_triggers.sql`:

```sql
-- ledi_outbox (ADR 0028; spec 2026-10-05-module-16 §6.3): a ficha aceita é prova
-- do que foi entregue ao PEC — não muda nem some, e o conteúdo dela já foi
-- apagado (CHECK ck_ledi_outbox_accepted_payload). A identidade da ficha nunca
-- muda; o uuid só muda no reenvio de uma recusada (rejected → pending); o
-- payload pode ir a nulo, nunca voltar. A re-cifra (ReencryptionJob) regrava o
-- payload não nulo de linhas não aceitas, o que continua permitido.
CREATE OR REPLACE FUNCTION rota_ledi_outbox_guard() RETURNS trigger AS $fn$
BEGIN
  IF TG_OP = 'DELETE' THEN
    IF OLD.status = 'accepted' THEN
      RAISE EXCEPTION 'ledi_outbox: accepted is immutable';
    END IF;
    RETURN OLD;
  END IF;
  IF OLD.status = 'accepted' THEN
    RAISE EXCEPTION 'ledi_outbox: accepted is immutable';
  END IF;
  IF NEW.id IS DISTINCT FROM OLD.id
     OR NEW.ficha_type IS DISTINCT FROM OLD.ficha_type
     OR NEW.competence IS DISTINCT FROM OLD.competence
     OR NEW.source_type IS DISTINCT FROM OLD.source_type
     OR NEW.source_id IS DISTINCT FROM OLD.source_id
     OR NEW.ledi_version IS DISTINCT FROM OLD.ledi_version
     OR NEW.created_at IS DISTINCT FROM OLD.created_at THEN
    RAISE EXCEPTION 'ledi_outbox: identity columns never change';
  END IF;
  IF NEW.uuid IS DISTINCT FROM OLD.uuid AND NOT (OLD.status = 'rejected' AND NEW.status = 'pending') THEN
    RAISE EXCEPTION 'ledi_outbox: uuid changes only when a rejected ficha is resent';
  END IF;
  IF OLD.payload IS NULL AND NEW.payload IS NOT NULL THEN
    RAISE EXCEPTION 'ledi_outbox: payload only goes to null';
  END IF;
  RETURN NEW;
END;
$fn$ LANGUAGE plpgsql;

DO $do$
BEGIN
  IF to_regclass('public.ledi_outbox') IS NOT NULL THEN
    EXECUTE 'DROP TRIGGER IF EXISTS ledi_outbox_guard ON ledi_outbox';
    EXECUTE 'CREATE TRIGGER ledi_outbox_guard
      BEFORE UPDATE OR DELETE ON ledi_outbox
      FOR EACH ROW EXECUTE FUNCTION rota_ledi_outbox_guard()';
  END IF;
END
$do$;
```

- [ ] **Step 4: Modelo e cifra**

```ruby
# app/models/ledi_outbox_entry.rb
# Uma ficha na fila de saída LEDI da cidade (ADR 0028; spec §6.3). O payload é o
# DadoTransporteThrift serializado, em Base64, cifrado com a chave da cidade;
# vai a nulo no aceite (CHECK + trigger). Transições de envio: Ledi::Delivery.
class LediOutboxEntry < ApplicationRecord
  self.table_name = "ledi_outbox"

  STATUSES = %w[pending sending accepted rejected failed].freeze

  encrypts :payload

  scope :for_competence, ->(competence) { where(competence: competence) }

  def bytes = payload && Base64.strict_decode64(payload)

  def bytes=(raw)
    self.payload = raw && Base64.strict_encode64(raw)
  end

  # Reivindica um lote vencido: transação curta, SKIP LOCKED (duas execuções
  # nunca pegam a mesma linha), e a linha sai como `sending` — o HTTP acontece
  # FORA desta transação.
  def self.claim!(limit:)
    ids = transaction do
      picked = where(status: "pending").where(next_attempt_at: ..Time.current)
               .order(:next_attempt_at, :id).limit(limit).lock("FOR UPDATE SKIP LOCKED").pluck(:id)
      mark_sending(picked)
      picked
    end
    where(id: ids).order(:next_attempt_at, :id).to_a
  end

  def self.mark_sending(ids)
    return if ids.empty?

    where(id: ids).update_all([ "status = 'sending', updated_at = :now, first_attempt_at = COALESCE(first_attempt_at, :now)",
                                { now: Time.current } ])
  end

  def self.release!(ids)
    where(id: ids, status: "sending").update_all(status: "pending", updated_at: Time.current)
  end

  def self.release_stale!(before:)
    where(status: "sending").where(updated_at: ...before).update_all(status: "pending", updated_at: Time.current)
  end

  def accept!
    transaction do
      update!(status: "accepted", accepted_at: Time.current, payload: nil, last_error: nil, attempts: attempts + 1)
      DomainEvents.publish("ledi.ficha_accepted", outbox_id: id, ficha_type: ficha_type, competence: competence)
    end
  end

  def reject!(message)
    transaction do
      update!(status: "rejected", last_error: Ledi::ErrorText.sanitize(message), attempts: attempts + 1)
      DomainEvents.publish("ledi.ficha_rejected", outbox_id: id, ficha_type: ficha_type, competence: competence)
    end
  end

  # Falha transitória: nova tentativa depois de `wait`, ou failed quando a
  # primeira tentativa já passou de `give_up_after`.
  def retry_later!(error:, wait:, give_up_after:, now: Time.current)
    started = first_attempt_at || now
    status = started <= now - give_up_after ? "failed" : "pending"
    update!(status: status, attempts: attempts + 1, last_error: Ledi::ErrorText.sanitize(error),
            next_attempt_at: now + wait)
  end
end
```

`accept!`/`reject!` dependem de `Ledi::ErrorText` e dos eventos (Tasks 7 e 8); para esta task passar, crie já o `Ledi::ErrorText` mínimo da Task 7 (a Task 7 só acrescenta a spec dele) e declare os dois eventos (Task 8, Step 5) — os dois arquivos estão transcritos lá; copie-os agora e mantenha-os iguais.

Em `app/services/city_encryption.rb`, no fim de `CITY_KEYED_TARGETS` (depois de `[ Professional,   :contact_email ]`, mais as entradas que a fundação acrescentou):

```ruby
    [ LediOutboxEntry, :payload ]
```

- [ ] **Step 5: Migre os bancos de dev e faça o dump à mão**

Run:
```bash
docker compose exec -T -w /rails/.claude/mod16-exporter api bin/rails "city:migrate[curitiba]"
docker compose exec -T -w /rails/.claude/mod16-exporter api bin/rails "city:migrate[maringa]"
```
Expected: `[city:migrate] curitiba → 20261007100001` (e maringa).

Em `db/city_schema.rb`: `define(version: <última da fundação>)` passa a `define(version: 2026_10_07_100001)` e, na ordem alfabética das tabelas (depois de `integration_credentials`, antes de `memberships`), entra:

```ruby
  create_table "ledi_outbox", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.datetime "accepted_at"
    t.integer "attempts", default: 0, null: false
    t.string "competence", limit: 6, null: false
    t.datetime "created_at", null: false
    t.string "ficha_type", null: false
    t.datetime "first_attempt_at"
    t.string "last_error", limit: 500
    t.string "ledi_version", null: false
    t.datetime "next_attempt_at", null: false
    t.text "payload"
    t.uuid "source_id", null: false
    t.string "source_type", null: false
    t.string "status", default: "pending", null: false
    t.datetime "updated_at", null: false
    t.string "uuid", limit: 44, null: false
    t.index ["competence", "status"], name: "idx_ledi_outbox_competence"
    t.index ["source_type", "source_id", "ficha_type"], name: "idx_ledi_outbox_source", unique: true
    t.index ["status", "next_attempt_at"], name: "idx_ledi_outbox_due"
    t.index ["uuid"], name: "idx_ledi_outbox_uuid", unique: true
    t.check_constraint "(status::text = 'accepted'::text) = (accepted_at IS NOT NULL)", name: "ck_ledi_outbox_accepted_at"
    t.check_constraint "competence::text ~ '^[0-9]{4}(0[1-9]|1[0-2])$'::text", name: "ck_ledi_outbox_competence"
    t.check_constraint "ficha_type::text ~ '^[a-z_]+$'::text", name: "ck_ledi_outbox_ficha_type"
    t.check_constraint "status::text <> 'accepted'::text OR payload IS NULL", name: "ck_ledi_outbox_accepted_payload"
    t.check_constraint "status::text <> 'rejected'::text OR last_error IS NOT NULL", name: "ck_ledi_outbox_rejected_error"
    t.check_constraint "status::text = ANY (ARRAY['pending'::text, 'sending'::text, 'accepted'::text, 'rejected'::text, 'failed'::text])", name: "ck_ledi_outbox_status"
  end
```

O juiz é a paridade (Step 6). Se ela acusar diferença, compare com `\d+ ledi_outbox` no banco de dev de `maringa` (nome do banco: `bin/rails runner 'puts URI(City.find_by(slug: "maringa").database_url).path.delete_prefix("/")'`; não cole a URL em lugar nenhum) e ajuste o dump, nunca a migração, salvo erro de verdade nela.

- [ ] **Step 6: Recarregue os bancos de teste e rode**

Run:
```bash
psql -h localhost -U rota_saude -d postgres -c "DROP DATABASE IF EXISTS rota_saude_test_city_a" -c "DROP DATABASE IF EXISTS rota_saude_test_city_b"
docker compose exec -T -w /rails/.claude/mod16-exporter -e RAILS_ENV=test api bin/rails city:test_databases
docker compose exec -T -w /rails/.claude/mod16-exporter api bundle exec rspec spec/services/city_schema_spec.rb spec/models/ledi_outbox_entry_spec.rb spec/models/create_ledi_outbox_migration_spec.rb spec/architecture/city_encrypted_attributes_guard_spec.rb
```
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod16-exporter add db/city_migrate/20261007100001_create_ledi_outbox.rb db/city_schema.rb db/city_triggers.sql app/models/ledi_outbox_entry.rb app/services/city_encryption.rb app/services/ledi/error_text.rb config/initializers/domain_events.rb spec/models/ledi_outbox_entry_spec.rb spec/models/create_ledi_outbox_migration_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod16-exporter commit -m "feat: add the encrypted LEDI outbox with its immutability guard

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 6: `Ledi::Enqueue` (enfileiramento)

**Files:**
- Create: `app/commands/ledi/enqueue.rb`
- Test: `spec/commands/ledi/enqueue_spec.rb`

**Interfaces:**
- Consumes: `Ledi::Ficha.assert!`, `Ledi::Transport.wrap`, `LediOutboxEntry`, `Platform::Features.enabled?`, `City#record_mode`.
- Produces: `Ledi::Enqueue.call(ficha, city:) -> LediOutboxEntry|nil` (`nil` = interruptor desligado ou `record_mode=off`; a mesma fonte devolve a linha existente); enfileira `Ledi::DeliverJob` depois do commit (a classe nasce na Task 8; até lá a spec stuba `perform_later`).

- [ ] **Step 1: Escreva a spec (falha)**

```ruby
# spec/commands/ledi/enqueue_spec.rb
require "rails_helper"

RSpec.describe Ledi::Enqueue do
  let(:city) { register_test_city! }
  let(:ficha) do
    Ledi::Fichas::Synthetic.new(cnes: "1234567", ine: "0000123456", professional_cns: "700000000000005",
                                cbo: "225142", attended_at: Time.current)
  end

  before do
    ledi_ready!(city, pec_url: "https://pec.a.test")
    allow(Ledi::DeliverJob).to receive(:perform_later)
  end

  it "grava a ficha serializada e cifrada, pendente para agora, e dispara o envio" do
    entry = described_class.call(ficha, city: city)

    expect(entry).to have_attributes(status: "pending", ficha_type: "procedimento", competence: ficha.competence,
                                     source_type: "synthetic", source_id: ficha.source_id, ledi_version: "8.7.0",
                                     attempts: 0)
    expect(entry.uuid).to match(/\A1234567-\h{8}-\h{4}-4\h{3}-\h{4}-\h{12}\z/)
    expect(entry.uuid.length).to eq(44)
    expect(entry.next_attempt_at).to be <= Time.current
    expect(Ledi::Transport.read(entry.bytes).uuidDadoSerializado).to eq(entry.uuid)
    expect(Ledi::DeliverJob).to have_received(:perform_later).with(no_args)
  end

  it "a mesma fonte não entra duas vezes" do
    first = described_class.call(ficha, city: city)
    expect(described_class.call(ficha, city: city).id).to eq(first.id)
    expect(LediOutboxEntry.count).to eq(1)
  end

  it "interruptor desligado ou record_mode off: nada entra" do
    ledi_off!(city)
    expect(described_class.call(ficha, city: city)).to be_nil
    ledi_ready!(city, pec_url: "https://pec.a.test", record_mode: "off")
    expect(described_class.call(ficha, city: city)).to be_nil
    expect(LediOutboxEntry.count).to eq(0)
    expect(Ledi::DeliverJob).not_to have_received(:perform_later)
  end

  it "ficha fora da interface: Ledi::Ficha::Invalid, nada gravado" do
    expect { described_class.call(Object.new, city: city) }.to raise_error(Ledi::Ficha::Invalid)
    expect(LediOutboxEntry.count).to eq(0)
  end
end
```

Run: `docker compose exec -T -w /rails/.claude/mod16-exporter api bundle exec rspec spec/commands/ledi/enqueue_spec.rb`
Expected: FAIL com `uninitialized constant Ledi::Enqueue` (ou `Ledi::DeliverJob`; crie nesta task o esqueleto `module Ledi; class DeliverJob < ApplicationJob; prepend EachCityJob; queue_as :default; def perform; end; end; end` em `app/jobs/ledi/deliver_job.rb` — a Task 8 o preenche).

- [ ] **Step 2: Implemente**

```ruby
# app/commands/ledi/enqueue.rb
# Entrada de uma ficha na fila LEDI da cidade (ADR 0028; spec §6.3/§6.4). Só
# entra com o interruptor ledi_export LIGADO e record_mode diferente de off —
# credencial recusada ou PEC fora do ar não impedem a entrada (a ficha espera).
# A mesma fonte (source_type, source_id, ficha_type) entra uma vez. Precisa
# rodar na conexão da cidade.
module Ledi
  module Enqueue
    module_function

    def call(ficha, city:)
      Ledi::Ficha.assert!(ficha)
      return nil unless Platform::Features.enabled?(city, :ledi_export) && city.record_mode != "off"

      source = ficha.source
      existing = LediOutboxEntry.find_by(source_type: source[:type], source_id: source[:id], ficha_type: ficha.type)
      return existing if existing

      uuid = "#{ficha.cnes}-#{SecureRandom.uuid}"
      entry = ApplicationRecord.transaction(requires_new: true) do
        LediOutboxEntry.create!(uuid: uuid, ficha_type: ficha.type, competence: ficha.competence,
                                source_type: source[:type], source_id: source[:id],
                                ledi_version: Ledi::Version::ACTIVE, next_attempt_at: Time.current,
                                bytes: Ledi::Transport.wrap(ficha, city: city, uuid: uuid))
      end
      Ledi::DeliverJob.perform_later
      entry
    rescue ActiveRecord::RecordNotUnique
      LediOutboxEntry.find_by!(source_type: source[:type], source_id: source[:id], ficha_type: ficha.type)
    end
  end
end
```

Run: `docker compose exec -T -w /rails/.claude/mod16-exporter api bundle exec rspec spec/commands/ledi/enqueue_spec.rb`
Expected: PASS.

- [ ] **Step 3: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod16-exporter add app/commands/ledi/enqueue.rb app/jobs/ledi/deliver_job.rb spec/commands/ledi/enqueue_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod16-exporter commit -m "feat: enqueue LEDI fichas only when the city's export is on

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 7: Classificação da resposta, saneamento do erro e espera crescente

**Files:**
- Create: `app/services/ledi/outcome.rb`, `app/services/ledi/backoff.rb`; `app/services/ledi/error_text.rb` (já criado na Task 5 — aqui entra a spec)
- Test: `spec/services/ledi/outcome_spec.rb`, `spec/services/ledi/error_text_spec.rb`, `spec/services/ledi/backoff_spec.rb`

**Interfaces:**
- Consumes: `Ledi::Observations` (Task 2).
- Produces: `Ledi::Outcome.classify(status, body) -> :accepted|:rejected|:unauthorized|:retry`; `Ledi::ErrorText.sanitize(text) -> String` (≤ 500); `Ledi::Backoff.wait(attempts) -> ActiveSupport::Duration`; `Ledi::Backoff::GIVE_UP_AFTER == 24.hours`.

- [ ] **Step 1: Escreva as specs (falham)**

```ruby
# spec/services/ledi/outcome_spec.rb
require "rails_helper"

# Spec §6.4: 2xx aceita; 400 recusa (não reenvia sozinho); sessão expirada
# (status observado na prova) pede relogin; 5xx e o resto tentam de novo.
RSpec.describe Ledi::Outcome do
  before do
    allow(Ledi::Observations).to receive(:duplicate_marker).and_return("já foi recebida")
    allow(Ledi::Observations).to receive(:session_expired_statuses).and_return([ 401, 302 ])
  end

  {
    [ 200, "" ] => :accepted, [ 201, "" ] => :accepted,
    [ 400, "CNES 1234567 não pertence ao município" ] => :rejected,
    [ 401, "" ] => :unauthorized, [ 302, "" ] => :unauthorized,
    [ 500, "erro" ] => :retry, [ 502, "" ] => :retry, [ 503, "" ] => :retry, [ 404, "" ] => :retry
  }.each do |(status, body), expected|
    it("#{status} → #{expected}") { expect(described_class.classify(status, body)).to eq(expected) }
  end

  # Review Focus 1: o reenvio do mesmo uuid depois de um 200 perdido.
  it "duplicidade observada classifica como aceita" do
    expect(described_class.classify(400, "A ficha 1234567-abc já foi recebida.")).to eq(:accepted)
  end

  it "sem marcador observado, todo 400 é recusa" do
    allow(Ledi::Observations).to receive(:duplicate_marker).and_return(nil)
    expect(described_class.classify(400, "A ficha já foi recebida.")).to eq(:rejected)
  end
end
```

```ruby
# spec/services/ledi/error_text_spec.rb
require "rails_helper"

# Review Focus 3: last_error nunca guarda dado de cidadão.
RSpec.describe Ledi::ErrorText do
  {
    "CPF 12345678909 inválido" => "CPF [número] inválido",
    "CNS do cidadão 898001160660761 não encontrado" => "CNS do cidadão [número] não encontrado",
    "CNES 1234567 não pertence ao município" => "CNES 1234567 não pertence ao município",
    "  INE   0000123456 inativo \n" => "INE 0000123456 inativo",
    "telefone 41999990000" => "telefone [número]"
  }.each do |input, expected|
    it("#{input.strip.inspect}") { expect(described_class.sanitize(input)).to eq(expected) }
  end

  it "corta em 500 caracteres e trata nil" do
    expect(described_class.sanitize("x" * 900).length).to eq(500)
    expect(described_class.sanitize(nil)).to eq("")
  end
end
```

```ruby
# spec/services/ledi/backoff_spec.rb
require "rails_helper"

# Review Focus 5: espera crescente, teto de 2 h, desistência 24 h depois da
# PRIMEIRA tentativa.
RSpec.describe Ledi::Backoff do
  it "dobra a partir de 1 minuto até o teto de 2 horas" do
    expect((1..9).map { |n| described_class.wait(n).to_i / 60 }).to eq([ 1, 2, 4, 8, 16, 32, 64, 120, 120 ])
  end

  it "desiste 24 h depois da primeira tentativa" do
    expect(described_class::GIVE_UP_AFTER).to eq(24.hours)
  end
end
```

Run: `docker compose exec -T -w /rails/.claude/mod16-exporter api bundle exec rspec spec/services/ledi/outcome_spec.rb spec/services/ledi/error_text_spec.rb spec/services/ledi/backoff_spec.rb`
Expected: FAIL com `uninitialized constant Ledi::Outcome`.

- [ ] **Step 2: Implemente**

```ruby
# app/services/ledi/error_text.rb
# Texto de erro que pode ir para last_error e para o painel: sem sequência de
# 11 ou mais dígitos (CPF, CNS, telefone), espaços normalizados, até 500.
# CNES (7) e INE (10) continuam legíveis — são da unidade e da equipe.
module Ledi
  module ErrorText
    MAX = 500

    module_function

    def sanitize(text)
      text.to_s.gsub(/\d{11,}/, "[número]").squish.truncate(MAX, omission: "")
    end
  end
end
```

```ruby
# app/services/ledi/outcome.rb
# O que uma resposta do recebimento do PEC significa para a fila (spec §6.4).
# Duplicidade e sessão expirada seguem o que a prova técnica observou
# (config/ledi/pec_observations.yml).
module Ledi
  module Outcome
    module_function

    # Só 400 recusa; sessão expirada pede relogin; todo o resto tenta de novo.
    def classify(status, body)
      return :accepted if (200..299).cover?(status)
      return :unauthorized if Ledi::Observations.session_expired_statuses.include?(status)
      return :retry unless status == 400

      marker = Ledi::Observations.duplicate_marker
      marker && body.to_s.include?(marker) ? :accepted : :rejected
    end
  end
end
```

```ruby
# app/services/ledi/backoff.rb
# Espera entre tentativas de uma ficha com falha transitória (5xx, timeout):
# 1, 2, 4... minutos, até 2 h; desistência (failed) 24 h depois da primeira
# tentativa (spec §6.4; desvio 3 do plano).
module Ledi
  module Backoff
    CAP = 2.hours
    GIVE_UP_AFTER = 24.hours

    module_function

    def wait(attempts)
      [ 1.minute * (2**(attempts - 1)), CAP ].min
    end
  end
end
```

Run: `docker compose exec -T -w /rails/.claude/mod16-exporter api bundle exec rspec spec/services/ledi/outcome_spec.rb spec/services/ledi/error_text_spec.rb spec/services/ledi/backoff_spec.rb`
Expected: PASS.

- [ ] **Step 3: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod16-exporter add app/services/ledi/outcome.rb app/services/ledi/error_text.rb app/services/ledi/backoff.rb spec/services/ledi/outcome_spec.rb spec/services/ledi/error_text_spec.rb spec/services/ledi/backoff_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod16-exporter commit -m "feat: classify PEC replies, mask citizen numbers and back off retries

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 8: `Ledi::DeliverJob` e `Ledi::Delivery` (tabela de casos)

**Files:**
- Create: `app/services/ledi/session_cache.rb`, `app/services/ledi/delivery.rb`
- Modify: `app/jobs/ledi/deliver_job.rb`, `config/recurring.yml`, `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`
- Test: `spec/jobs/ledi/deliver_job_spec.rb`

**Interfaces:**
- Consumes: `LediOutboxEntry.claim!/release!/release_stale!/#accept!/#reject!/#retry_later!` (Task 5), `Ledi::Outcome`, `Ledi::Backoff` (Task 7), `Ledi::PecClient.new/#login/#deliver` (Task 1 + fundação), `IntegrationCredential`, `Platform::Features.usable?`.
- Produces:
  - `Ledi::DeliverJob` (`prepend EachCityJob`, sem argumentos; `BATCH_SIZE = 20`; `STALE_SENDING = 10.minutes`).
  - `Ledi::Delivery.new(city).run(entries) -> :done|:paused`; `Ledi::Delivery::PAUSE_MESSAGE`.
  - `Ledi::SessionCache.fetch(key) { cookie }`, `.forget(key)`, `.clear!`; chave `[city.id, credential.set_at.to_f]`.
  - Recorrente `ledi_deliver` (a cada minuto, fila `default`).

- [ ] **Step 1: Escreva a spec (falha)**

```ruby
# spec/jobs/ledi/deliver_job_spec.rb
require "rails_helper"

# Spec §6.4 / §9 "Envio": tabela de casos. O PEC é o FakePec
# (spec/support/ledi_helpers.rb).
RSpec.describe Ledi::DeliverJob do
  include ActiveSupport::Testing::TimeHelpers

  let!(:city) { ledi_ready!(register_test_city!, pec_url: "https://pec.a.test") }
  let(:pec) { FakePec.for("https://pec.a.test") }

  before do
    stub_pec!
    allow(Ledi::Observations).to receive(:duplicate_marker).and_return(nil)
    allow(Ledi::Observations).to receive(:session_expired_statuses).and_return([ 401 ])
  end

  def enqueue!(n = 1)
    allow(Ledi::DeliverJob).to receive(:perform_later)
    Array.new(n) do
      Ledi::Enqueue.call(Ledi::Fichas::Synthetic.new(cnes: "1234567", ine: "0000123456",
                                                     professional_cns: "700000000000005", cbo: "225142",
                                                     attended_at: Time.current), city: city)
    end
  end

  def run! = described_class.perform_now
  def events(name) = DomainEvent.where(name: name).map(&:payload)

  it "200/201: accepted, payload apagado, evento só com ids, um login para o lote" do
    entries = enqueue!(2)
    pec.delivery_replies = [ [ 200, "" ], [ 201, "" ] ]
    run!
    entries.each(&:reload)
    expect(entries.map(&:status)).to eq(%w[accepted accepted])
    expect(entries.map(&:payload)).to eq([ nil, nil ])
    expect(pec.logins.size).to eq(1)
    expect(pec.deliveries.map { |d| d[:filename] }).to eq(entries.map { |e| "#{e.uuid}.esus" })
    expect(events("ledi.ficha_accepted")).to contain_exactly(
      *entries.map { |e| { "outbox_id" => e.id, "ficha_type" => "procedimento", "competence" => e.competence } }
    )
  end

  it "400: rejected com a mensagem mascarada, sem nova tentativa sozinha" do
    entry = enqueue!.first
    pec.delivery_replies = [ [ 400, "CPF 12345678909 inválido" ] ]
    run!
    expect(entry.reload.slice(:status, :last_error, :attempts))
      .to eq("status" => "rejected", "last_error" => "CPF [número] inválido", "attempts" => 1)
    expect(events("ledi.ficha_rejected").sole).to eq("outbox_id" => entry.id, "ficha_type" => "procedimento",
                                                      "competence" => entry.competence)
    run!
    expect(pec.deliveries.size).to eq(1)
  end

  it "5xx e timeout: pending com espera crescente; payload continua" do
    first, second = enqueue!(2)
    pec.delivery_replies = [ [ 503, "" ], Ledi::PecClient::Unreachable ]
    freeze_time do
      run!
      expect(first.reload.slice(:status, :attempts, :last_error)).to eq("status" => "pending", "attempts" => 1,
                                                                        "last_error" => "HTTP 503")
      expect(first.next_attempt_at).to eq(1.minute.from_now)
      expect(second.reload.last_error).to eq("PEC inacessível")
      expect(second.bytes).to be_present
    end
  end

  it "24 h depois da primeira tentativa: failed" do
    entry = enqueue!.first
    pec.delivery_replies = [ [ 500, "" ] ]
    run!
    entry.reload.update_columns(next_attempt_at: 1.minute.ago, first_attempt_at: 25.hours.ago)
    pec.delivery_replies = [ [ 500, "" ] ]
    run!
    expect(entry.reload.slice(:status, :attempts)).to eq("status" => "failed", "attempts" => 2)
  end

  it "401: novo login uma vez e a mesma ficha é aceita" do
    entry = enqueue!.first
    pec.delivery_replies = [ [ 401, "" ], [ 201, "" ] ]
    run!
    expect(entry.reload.status).to eq("accepted")
    expect(pec.logins.size).to eq(2)
  end

  it "401 duas vezes: credencial unauthorized, lote volta a pending sem tentativa, envio pausado" do
    entries = enqueue!(2)
    pec.delivery_replies = [ [ 401, "" ], [ 401, "" ] ]
    run!
    expect(entries.map { |e| e.reload.slice(:status, :attempts) }).to all(eq("status" => "pending", "attempts" => 0))
    credential = IntegrationCredential.find_by!(kind: "ledi")
    expect(credential.last_check_status).to eq("unauthorized")
    expect(credential.last_check_message).to eq(Ledi::Delivery::PAUSE_MESSAGE)
    expect(credential.last_check_message).not_to include("segredo-ledi")

    pec.delivery_replies = []
    allow(Platform::Features).to receive(:usable?).and_call_original
    run!
    expect(pec.deliveries.size).to eq(2) # pausado: usable? é falso com credential_unauthorized
  end

  it "login recusado: pausa direto" do
    enqueue!
    pec.login_replies = [ Ledi::PecClient::Unauthorized ]
    run!
    expect(IntegrationCredential.find_by!(kind: "ledi").last_check_status).to eq("unauthorized")
    expect(pec.deliveries).to be_empty
    expect(LediOutboxEntry.pluck(:status, :attempts)).to eq([ [ "pending", 0 ] ])
  end

  # Review Focus 2.
  it "pausada não envia; voltou a ok (credencial nova), envia — sem reaproveitar o cookie antigo" do
    entry = enqueue!.first
    pec.delivery_replies = [ [ 401, "" ], [ 401, "" ] ]
    run!
    expect(entry.reload.status).to eq("pending")

    travel 1.second
    IntegrationCredential.find_by!(kind: "ledi").update!(secret: { "username" => "rota2", "password" => "nova" },
                                                         set_at: Time.current, last_check_status: "ok")
    pec.delivery_replies = [ [ 201, "" ] ]
    run!
    expect(entry.reload.status).to eq("accepted")
    expect(pec.logins.last[:username]).to eq("rota2")
    expect(pec.deliveries.last[:cookie]).to eq("JSESSIONID=fake-#{pec.logins.size}")
  end

  it "o cookie fica em cache entre execuções da mesma credencial" do
    enqueue!
    run!
    enqueue!
    run!
    expect(pec.logins.size).to eq(1)
  end

  it "interruptor desligado ou record_mode off: nada sai" do
    enqueue!(2)
    ledi_off!(city)
    run!
    ledi_ready!(city, pec_url: "https://pec.a.test", record_mode: "off")
    run!
    expect(pec.deliveries).to be_empty
    expect(LediOutboxEntry.distinct.pluck(:status)).to eq([ "pending" ])
  end

  it "está no recurring.yml, a cada minuto, na fila default" do
    task = YAML.load_file(Rails.root.join("config/recurring.yml"), aliases: true).dig("production", "ledi_deliver")
    expect(task).to eq("class" => "Ledi::DeliverJob", "queue" => "default", "schedule" => "every minute")
  end
end
```

O exemplo "401 duas vezes" depende do `Platform::Features.usable?` da fundação devolver falso com `last_check_status = "unauthorized"` (pré-requisito `credential_unauthorized:ledi` do contrato §2). Se a fundação ainda não ler esse campo, **pare** e avise a sessão da fundação: é o contrato.

Run: `docker compose exec -T -w /rails/.claude/mod16-exporter api bundle exec rspec spec/jobs/ledi/deliver_job_spec.rb`
Expected: FAIL (o `perform` vazio da Task 6 não envia nada).

- [ ] **Step 2: Cache do cookie**

```ruby
# app/services/ledi/session_cache.rb
# Cookie JSESSIONID do PEC por cidade, em memória do processo (spec §6.4). A
# chave inclui o set_at da credencial: credencial trocada nunca reaproveita o
# cookie da antiga. Nunca persiste, nunca loga.
module Ledi
  module SessionCache
    module_function

    def store = @store ||= Concurrent::Map.new

    def fetch(key, &block) = store.compute_if_absent(key, &block)

    def forget(key) = store.delete(key)

    def clear! = store.clear
  end
end
```

- [ ] **Step 3: Envio**

```ruby
# app/services/ledi/delivery.rb
# Envio de um lote já reivindicado (sending) ao PEC da cidade (spec §6.4).
# 2xx → accepted; 400 → rejected; 5xx/timeout → nova tentativa com espera;
# sessão expirada → novo login uma vez; segunda recusa (ou login recusado) →
# credencial unauthorized, o resto do lote volta a pending sem contar
# tentativa e a cidade fica pausada (usable? falso) até a credencial voltar a ok.
# A credencial e o cookie nunca saem daqui: nem log, nem evento, nem last_error.
module Ledi
  class Delivery
    PAUSE_MESSAGE = "O PEC recusou a credencial durante o envio; o envio está pausado.".freeze

    def initialize(city)
      @city = city
      @credential = IntegrationCredential.find_by!(kind: "ledi")
      @client = Ledi::PecClient.new(base_url: city.pec_url, username: @credential.username,
                                    password: @credential.password)
      @cache_key = [ city.id, @credential.set_at.to_f ]
    end

    def run(entries)
      entries.each_with_index do |entry, index|
        next if deliver(entry) != :paused

        LediOutboxEntry.release!(entries[index..].map(&:id))
        return :paused
      end
      :done
    end

    private

    def deliver(entry)
      reply = post(entry)
      return pause! if reply == :unauthorized

      case Ledi::Outcome.classify(reply.status, reply.body)
      when :accepted then entry.accept!
      when :rejected then entry.reject!(reply.body.presence || "HTTP #{reply.status}")
      else retry_later(entry, "HTTP #{reply.status}")
      end
    rescue Ledi::PecClient::Unreachable
      retry_later(entry, "PEC inacessível")
    rescue Ledi::PecClient::Failed => e
      retry_later(entry, "login no PEC respondeu #{e.status}")
    end

    def post(entry, relogged: false)
      reply = @client.deliver(cookie: cookie, filename: "#{entry.uuid}.esus", bytes: entry.bytes)
      return reply unless Ledi::Outcome.classify(reply.status, reply.body) == :unauthorized

      Ledi::SessionCache.forget(@cache_key)
      relogged ? :unauthorized : post(entry, relogged: true)
    rescue Ledi::PecClient::Unauthorized
      Ledi::SessionCache.forget(@cache_key)
      :unauthorized
    end

    def cookie
      Ledi::SessionCache.fetch(@cache_key) { @client.login.cookie }
    end

    def retry_later(entry, error)
      entry.retry_later!(error: error, wait: Ledi::Backoff.wait(entry.attempts + 1),
                         give_up_after: Ledi::Backoff::GIVE_UP_AFTER)
    end

    def pause!
      @credential.update!(last_check_status: "unauthorized", last_check_at: Time.current,
                          last_check_message: PAUSE_MESSAGE)
      :paused
    end
  end
end
```

Login que responde outro status (`Failed`) é falha transitória da ficha da vez (espera crescente), não pausa.

- [ ] **Step 4: Job e recorrente**

```ruby
# app/jobs/ledi/deliver_job.rb
# Envio contínuo das fichas LEDI da cidade (ADR 0028; spec §6.4). Recorrente a
# cada minuto e disparado por Ledi::Enqueue; no worker de cada cidade roda só
# nela (EachCityJob). Sem argumentos: nada de credencial ou ficha na fila do
# Solid Queue. Só roda com ledi_export UTILIZÁVEL (ligado, record_mode != off,
# pec_url, IBGE e credencial ok).
module Ledi
  class DeliverJob < ApplicationJob
    prepend EachCityJob
    queue_as :default

    BATCH_SIZE = 20
    STALE_SENDING = 10.minutes

    def perform
      city = Current.city
      return unless Platform::Features.usable?(city, :ledi_export)

      LediOutboxEntry.release_stale!(before: STALE_SENDING.ago)
      entries = LediOutboxEntry.claim!(limit: BATCH_SIZE)
      return if entries.empty?

      Ledi::Delivery.new(city).run(entries)
    end
  end
end
```

Em `config/recurring.yml`, depois de `campaigns_due`:

```yaml
  ledi_deliver:
    class: Ledi::DeliverJob
    queue: default
    schedule: "every minute"
```

- [ ] **Step 5: Eventos**

Em `config/initializers/domain_events.rb`, depois de `DomainEvents.bind "city.campaigns_sms_toggled", to: []` (e das linhas da fundação):

```ruby
  # Exportador LEDI (ADR 0028; contratos §6): só trilha, só ids.
  DomainEvents.bind "ledi.ficha_accepted", to: []
  DomainEvents.bind "ledi.ficha_rejected", to: []
```

No fim de `spec/initializers/domain_events_bindings_spec.rb`:

```ruby
# Módulo 16 (ADR 0028): eventos do exportador LEDI, só trilha.
RSpec.describe "ledi event bindings (ADR 0028)" do
  it "declares every exporter event with no consumer" do
    names = %w[ledi.ficha_accepted ledi.ficha_rejected]
    expect(DomainEvents.registry.keys).to include(*names)
    expect(names.flat_map { |n| DomainEvents.registry[n] }).to be_empty
  end
end
```

Run: `docker compose exec -T -w /rails/.claude/mod16-exporter api bundle exec rspec spec/jobs/ledi/deliver_job_spec.rb spec/initializers/domain_events_bindings_spec.rb spec/config/solid_queue_configuration_spec.rb`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod16-exporter add app/services/ledi/session_cache.rb app/services/ledi/delivery.rb app/jobs/ledi/deliver_job.rb config/recurring.yml config/initializers/domain_events.rb spec/initializers/domain_events_bindings_spec.rb spec/jobs/ledi/deliver_job_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod16-exporter commit -m "feat: deliver queued LEDI fichas to the city's PEC with relogin and pause

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 9: Concorrência (`SKIP LOCKED` com threads) e `sending` parado

**Files:**
- Test: `spec/jobs/ledi/deliver_job_concurrency_spec.rb`

**Interfaces:**
- Consumes: `Ledi::DeliverJob`, `LediOutboxEntry.mark_sending` (gancho), `FakePec`, `ledi_ready!` (Tasks 2, 5, 8).
- Produces: nada novo — prova que duas execuções nunca enviam a mesma ficha e que o 200 perdido não vira recusa.

- [ ] **Step 1: Escreva a spec**

```ruby
# spec/jobs/ledi/deliver_job_concurrency_spec.rb
require "rails_helper"

# Spec §9 "SKIP LOCKED com duas threads". Threads reais contra TEST_CITY_A, sem
# fixture transacional (como spec/jobs/campaigns/due_job_concurrency_spec.rb):
# a primeira execução para DENTRO da transação de reivindicação (gancho em
# mark_sending); a segunda passa por cima das linhas travadas sem esperar e
# não envia nada. O after apaga o que commitou desligando o trigger só na
# transação da limpeza.
RSpec.describe Ledi::DeliverJob, "concorrência" do
  self.use_transactional_tests = false

  let(:release) { Queue.new }
  let(:threads) { [] }
  let!(:created_city) { City.find_by(slug: TEST_CITY_A.slug).nil? }
  let!(:city) { ledi_ready!(register_test_city!, pec_url: "https://pec.a.test") }
  let(:pec) { FakePec.for("https://pec.a.test") }
  let!(:entry_ids) do
    CityConnection.with(TEST_CITY_A) do
      allow(Ledi::DeliverJob).to receive(:perform_later)
      Array.new(3) do
        Ledi::Enqueue.call(Ledi::Fichas::Synthetic.new(cnes: "1234567", ine: "0000123456",
                                                       professional_cns: "700000000000005", cbo: "225142",
                                                       attended_at: Time.current), city: city).id
      end
    end
  end

  before do
    stub_pec!
    allow(Ledi::Observations).to receive(:duplicate_marker).and_return("já foi recebida")
    allow(Ledi::Observations).to receive(:session_expired_statuses).and_return([ 401 ])
  end

  after do
    3.times { release << true }
    threads.each { |t| t.join(5) || t.kill }
    begin
      CityConnection.with(TEST_CITY_A) do
        ApplicationRecord.transaction do
          connection = ApplicationRecord.connection
          connection.execute("SET LOCAL session_replication_role = replica")
          DomainEvent.where("payload ->> 'outbox_id' IN (?)", entry_ids).delete_all
          LediOutboxEntry.where(id: entry_ids).delete_all
          IntegrationCredential.where(kind: "ledi").delete_all
          user_ids = User.where(email_address: "ledi-admin@cidade.gov.br").pluck(:id)
          Membership.where(user_id: user_ids).delete_all
          User.where(id: user_ids).delete_all
        end
      end
    ensure
      CityFeature.where(city_id: city.id).delete_all
      city.destroy if created_city
    end
  end

  def statuses = CityConnection.with(TEST_CITY_A) { LediOutboxEntry.where(id: entry_ids).pluck(:status) }

  it "duas execuções concorrentes enviam cada ficha uma única vez" do
    locked = Queue.new
    holder = nil
    original = LediOutboxEntry.method(:mark_sending)
    allow(LediOutboxEntry).to receive(:mark_sending) do |ids|
      if Thread.current == holder
        locked << true
        release.pop(timeout: 10)
      end
      original.call(ids)
    end

    threads << (holder = Thread.new { described_class.perform_now })
    locked.pop(timeout: 5) or raise "a primeira execução não travou as linhas"

    threads << second = Thread.new { described_class.perform_now }
    expect(second.join(5)).to be(second) # SKIP LOCKED: não espera, não envia
    expect(pec.deliveries).to be_empty

    release << true
    expect(holder.join(5)).to be(holder)
    expect(statuses).to all(eq("accepted"))
    expect(pec.deliveries.map { |d| d[:filename] }.uniq.size).to eq(3)
    expect(pec.deliveries.size).to eq(3)
  end

  # Review Focus 1: o processo caiu depois do POST (o PEC tem a ficha) e antes
  # do UPDATE. A linha fica sending; passados 10 min, volta a pending e o
  # reenvio do mesmo uuid recebe a duplicidade observada — vira accepted.
  it "sending parado volta a pending e o reenvio duplicado vira accepted" do
    CityConnection.with(TEST_CITY_A) do
      LediOutboxEntry.where(id: entry_ids).update_all(status: "sending", updated_at: 11.minutes.ago,
                                                     first_attempt_at: 11.minutes.ago)
    end
    pec.delivery_replies = Array.new(3) { [ 400, "A ficha já foi recebida." ] }
    described_class.perform_now
    expect(statuses).to all(eq("accepted"))
  end
end
```

Run: `docker compose exec -T -w /rails/.claude/mod16-exporter api bundle exec rspec spec/jobs/ledi/deliver_job_concurrency_spec.rb`
Expected: PASS. (Se falhar por a segunda thread esperar: o `lock("FOR UPDATE SKIP LOCKED")` do `claim!` sumiu — é o defeito que esta spec existe para pegar.)

O `allow(...)` dentro de `let!` só funciona porque o `let!` roda dentro do exemplo; se o RSpec reclamar, mova o `allow(Ledi::DeliverJob).to receive(:perform_later)` para um `before` declarado **antes** dos `let!`.

- [ ] **Step 2: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod16-exporter add spec/jobs/ledi/deliver_job_concurrency_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod16-exporter commit -m "test: prove concurrent LEDI deliveries send each ficha once

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Fatia 4 — Prazo, painel e console (F-16.7)

### Task 10: Dias úteis e prazo da competência

**Files:**
- Create: `app/services/ledi/business_calendar.rb`, `app/services/ledi/deadline.rb`
- Test: `spec/services/ledi/deadline_spec.rb`

**Interfaces:**
- Produces:
  - `Ledi::BusinessCalendar.holidays(year) -> Set<Date>`; `.business_day?(date) -> Boolean`; `.easter(year) -> Date`.
  - `Ledi::Deadline::BUSINESS_DAY == 10`; `Ledi::Deadline.on(competence) -> Date`; `.business_days_left(competence, today:) -> Integer` (dias úteis de `today` até o prazo, os dois inclusive; 0 depois do prazo); `.current(today) -> "AAAAMM"`; `.previous(today) -> "AAAAMM"`; `.valid?(competence) -> Boolean`.

- [ ] **Step 1: Escreva a spec (falha)**

```ruby
# spec/services/ledi/deadline_spec.rb
require "rails_helper"

# Spec §6.5: prazo = 10º dia útil do mês seguinte, com feriados nacionais fixos
# e móveis. Conferido contra o calendário SIAPS 2026
# (sisaps.saude.gov.br/sistemas/siaps/docs/manual/calendario-siaps/).
RSpec.describe Ledi::Deadline do
  {
    "202601" => "2026-02-13", "202602" => "2026-03-13", "202603" => "2026-04-15", "202604" => "2026-05-15",
    # Calendário publicado: 16/06. Corpus Christi (04/06) contado como não útil
    # dá 15/06 — a estimativa nunca passa do prazo oficial (desvio 9 do plano).
    "202605" => "2026-06-15",
    "202606" => "2026-07-14", "202607" => "2026-08-14", "202608" => "2026-09-15", "202609" => "2026-10-15",
    "202610" => "2026-11-16", "202611" => "2026-12-14", "202612" => "2027-01-15"
  }.each do |competence, deadline|
    it("#{competence} → #{deadline}") { expect(described_class.on(competence)).to eq(Date.parse(deadline)) }
  end

  it "Carnaval de 2027 (08 e 09/02) empurra o prazo de janeiro" do
    expect(Ledi::BusinessCalendar.easter(2027)).to eq(Date.new(2027, 3, 28))
    expect(described_class.on("202701")).to eq(Date.new(2027, 2, 16))
  end

  it "feriados fixos, Sexta-feira Santa e Consciência Negra" do
    holidays = Ledi::BusinessCalendar.holidays(2026)
    expect(holidays).to include(Date.new(2026, 4, 3), Date.new(2026, 11, 20), Date.new(2026, 10, 12),
                                Date.new(2026, 2, 16), Date.new(2026, 2, 17), Date.new(2026, 6, 4))
    expect(Ledi::BusinessCalendar.business_day?(Date.new(2026, 10, 10))).to be(false) # sábado
  end

  # Review Focus 4.
  it "business_days_left conta hoje e o dia do prazo; depois do prazo é 0" do
    expect(described_class.business_days_left("202610", today: Date.new(2026, 11, 3))).to eq(10)
    expect(described_class.business_days_left("202610", today: Date.new(2026, 11, 16))).to eq(1)
    expect(described_class.business_days_left("202610", today: Date.new(2026, 11, 17))).to eq(0)
    expect(described_class.business_days_left("202610", today: Date.new(2026, 10, 20))).to eq(19)
  end

  it "competência corrente e anterior, e validação" do
    expect(described_class.current(Date.new(2026, 1, 5))).to eq("202601")
    expect(described_class.previous(Date.new(2026, 1, 5))).to eq("202512")
    expect(%w[202610 202613 2026-10 20261].map { |c| described_class.valid?(c) }).to eq([ true, false, false, false ])
  end
end
```

Run: `docker compose exec -T -w /rails/.claude/mod16-exporter api bundle exec rspec spec/services/ledi/deadline_spec.rb`
Expected: FAIL com `uninitialized constant Ledi::Deadline`.

Confira à mão o "19" do último exemplo antes de rodar (20/10/2026 a 16/11/2026: outubro 20–30 = 9 úteis; novembro 03–06 e 09–13 = 9, 16 = 1; total 19).

- [ ] **Step 2: Implemente**

```ruby
# app/services/ledi/business_calendar.rb
# Dias úteis nacionais para o prazo da competência (spec §6.5). Feriados fixos
# (Lei 662/1949, Lei 6.802/1980, Lei 14.759/2023) e móveis a partir da Páscoa;
# Carnaval e Corpus Christi são ponto facultativo federal e contam como não
# úteis — a estimativa sai igual ou antes do prazo oficial (desvio 9).
module Ledi
  module BusinessCalendar
    FIXED = [ [ 1, 1 ], [ 4, 21 ], [ 5, 1 ], [ 9, 7 ], [ 10, 12 ], [ 11, 2 ], [ 11, 15 ], [ 11, 20 ], [ 12, 25 ] ].freeze
    EASTER_OFFSETS = [ -48, -47, -2, 60 ].freeze # Carnaval (seg, ter), Sexta-feira Santa, Corpus Christi

    module_function

    def holidays(year)
      @holidays ||= {}
      @holidays[year] ||= begin
        easter_day = easter(year)
        (FIXED.map { |m, d| Date.new(year, m, d) } + EASTER_OFFSETS.map { |o| easter_day + o }).to_set.freeze
      end
    end

    def business_day?(date)
      !(date.saturday? || date.sunday?) && !holidays(date.year).include?(date)
    end

    # Algoritmo anônimo gregoriano (Meeus/Jones/Butcher).
    def easter(year)
      a = year % 19
      b, c = year.divmod(100)
      d, e = b.divmod(4)
      f = (b + 8) / 25
      g = (b - f + 1) / 3
      h = (19 * a + b - d - g + 15) % 30
      i, k = c.divmod(4)
      l = (32 + 2 * e + 2 * i - h - k) % 7
      m = (a + 11 * h + 22 * l) / 451
      month, day = (h + l - 7 * m + 114).divmod(31)
      Date.new(year, month, day + 1)
    end
  end
end
```

```ruby
# app/services/ledi/deadline.rb
# Prazo estimado da competência LEDI (spec §6.5; pesquisa frente 1 §3): 10º dia
# útil do mês seguinte. Funções puras: quem chama passa `today` já no fuso da
# cidade (Time.zone.today dentro da cidade).
module Ledi
  module Deadline
    BUSINESS_DAY = 10
    FORMAT = /\A\d{4}(0[1-9]|1[0-2])\z/

    module_function

    def valid?(competence) = competence.to_s.match?(FORMAT)

    def first_day(competence) = Date.new(competence[0, 4].to_i, competence[4, 2].to_i, 1)

    def on(competence)
      day = first_day(competence).next_month
      count = 0
      loop do
        count += 1 if Ledi::BusinessCalendar.business_day?(day)
        return day if count == BUSINESS_DAY

        day += 1
      end
    end

    def business_days_left(competence, today:)
      deadline = on(competence)
      return 0 if today > deadline

      (today..deadline).count { |day| Ledi::BusinessCalendar.business_day?(day) }
    end

    def current(today) = today.strftime("%Y%m")

    def previous(today) = today.prev_month.strftime("%Y%m")
  end
end
```

Run: `docker compose exec -T -w /rails/.claude/mod16-exporter api bundle exec rspec spec/services/ledi/deadline_spec.rb`
Expected: PASS.

- [ ] **Step 3: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod16-exporter add app/services/ledi/business_calendar.rb app/services/ledi/deadline.rb spec/services/ledi/deadline_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod16-exporter commit -m "feat: estimate the competence deadline on the 10th national business day

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 11: Alerta e resumo da competência

**Files:**
- Create: `app/services/ledi/alert.rb`, `app/queries/ledi/production_summary.rb`
- Test: `spec/services/ledi/alert_spec.rb`, `spec/queries/ledi/production_summary_spec.rb`

**Interfaces:**
- Consumes: `Ledi::Deadline` (Task 10), `LediOutboxEntry` (Task 5).
- Produces:
  - `Ledi::Alert.level(counts:, business_days_left:, record_mode:) -> "none"|"attention"|"critical"` (`counts` com chaves símbolo `accepted rejected pending sending failed`).
  - `Ledi::ProductionSummary.counts(competence) -> Hash{Symbol => Integer}`; `.rejections(competence) -> Array<{ message:, count: }>`; `.call(competence:, today:, record_mode:) -> Hash` com `competence`, `deadline_on` (Date), `business_days_left`, `alert`, `counts`, `rejections`.

- [ ] **Step 1: Escreva as specs (falham)**

```ruby
# spec/services/ledi/alert_spec.rb
require "rails_helper"

# Spec §6.5 / contratos §4.3: attention a ≤ 5 dias úteis com pendente,
# recusada ou falha; critical com record e zero aceitas a ≤ 3 dias úteis.
RSpec.describe Ledi::Alert do
  def counts(**overrides) = { accepted: 0, rejected: 0, pending: 0, sending: 0, failed: 0 }.merge(overrides)

  it "longe do prazo: none, mesmo com pendência" do
    expect(described_class.level(counts: counts(pending: 4), business_days_left: 6, record_mode: "record")).to eq("none")
  end

  it "a 5 dias úteis com pendente, recusada ou falha: attention" do
    %i[pending sending rejected failed].each do |key|
      expect(described_class.level(counts: counts(accepted: 1, key => 1), business_days_left: 5,
                                   record_mode: "integrated")).to eq("attention"), key.to_s
    end
    expect(described_class.level(counts: counts(accepted: 3), business_days_left: 5, record_mode: "integrated"))
      .to eq("none")
  end

  it "record e zero aceitas a 3 dias úteis: critical; integrated não" do
    expect(described_class.level(counts: counts, business_days_left: 3, record_mode: "record")).to eq("critical")
    expect(described_class.level(counts: counts(pending: 1), business_days_left: 3, record_mode: "integrated"))
      .to eq("attention")
    expect(described_class.level(counts: counts, business_days_left: 4, record_mode: "record")).to eq("none")
  end

  # Review Focus 4: depois do prazo não há o que alertar.
  it "prazo vencido: none" do
    expect(described_class.level(counts: counts(pending: 9), business_days_left: 0, record_mode: "record")).to eq("none")
  end
end
```

```ruby
# spec/queries/ledi/production_summary_spec.rb
require "rails_helper"

RSpec.describe Ledi::ProductionSummary do
  def entry!(status, competence: "202610", error: nil)
    attrs = { uuid: "1234567-#{SecureRandom.uuid}", ficha_type: "procedimento", competence: competence,
              source_type: "synthetic", source_id: SecureRandom.uuid, ledi_version: "8.7.0",
              next_attempt_at: Time.current, status: status, last_error: error }
    attrs[:accepted_at] = Time.current if status == "accepted"
    attrs[:bytes] = "x".b unless status == "accepted"
    LediOutboxEntry.create!(attrs)
  end

  before do
    3.times { entry!("accepted") }
    entry!("rejected", error: "CNES 1234567 não pertence ao município")
    entry!("rejected", error: "CNES 1234567 não pertence ao município")
    entry!("rejected", error: "CBO incompatível")
    entry!("pending")
    entry!("failed", error: "HTTP 500")
    entry!("accepted", competence: "202609")
  end

  it "conta por status só da competência pedida" do
    expect(described_class.counts("202610")).to eq(accepted: 3, rejected: 3, pending: 1, sending: 0, failed: 1)
  end

  it "agrupa recusas por mensagem, da mais frequente para a menos" do
    expect(described_class.rejections("202610")).to eq([
      { message: "CNES 1234567 não pertence ao município", count: 2 }, { message: "CBO incompatível", count: 1 }
    ])
  end

  it "monta o resumo com prazo, dias úteis e alerta" do
    summary = described_class.call(competence: "202610", today: Date.new(2026, 11, 10), record_mode: "record")
    expect(summary.slice(:competence, :deadline_on, :business_days_left, :alert))
      .to eq(competence: "202610", deadline_on: Date.new(2026, 11, 16), business_days_left: 5, alert: "attention")
  end
end
```

Run: `docker compose exec -T -w /rails/.claude/mod16-exporter api bundle exec rspec spec/services/ledi/alert_spec.rb spec/queries/ledi/production_summary_spec.rb`
Expected: FAIL com `uninitialized constant Ledi::Alert`.

- [ ] **Step 2: Implemente**

```ruby
# app/services/ledi/alert.rb
# Nível de alerta da competência (spec §6.5; contratos §4.3). Puro: o painel da
# cidade e o console usam a mesma regra.
module Ledi
  module Alert
    ATTENTION_DAYS = 5
    CRITICAL_DAYS = 3

    module_function

    def level(counts:, business_days_left:, record_mode:)
      return "none" if business_days_left <= 0
      return "critical" if record_mode == "record" && counts[:accepted].zero? && business_days_left <= CRITICAL_DAYS

      open = counts.values_at(:pending, :sending, :rejected, :failed).sum
      open.positive? && business_days_left <= ATTENTION_DAYS ? "attention" : "none"
    end
  end
end
```

```ruby
# app/queries/ledi/production_summary.rb
# Resumo de uma competência da fila LEDI da cidade corrente (spec §6.5).
# Recusas agrupadas pela mensagem JÁ saneada (Ledi::ErrorText) — nunca dado de
# cidadão.
module Ledi
  module ProductionSummary
    STATUSES = %i[accepted rejected pending sending failed].freeze

    module_function

    def counts(competence)
      grouped = LediOutboxEntry.for_competence(competence).group(:status).count
      STATUSES.to_h { |status| [ status, grouped.fetch(status.to_s, 0) ] }
    end

    def rejections(competence)
      LediOutboxEntry.for_competence(competence).where(status: "rejected").group(:last_error).count
                     .sort_by { |message, count| [ -count, message ] }
                     .map { |message, count| { message: message, count: count } }
    end

    def call(competence:, today:, record_mode:)
      totals = counts(competence)
      left = Ledi::Deadline.business_days_left(competence, today: today)
      { competence: competence, deadline_on: Ledi::Deadline.on(competence), business_days_left: left,
        alert: Ledi::Alert.level(counts: totals, business_days_left: left, record_mode: record_mode),
        counts: totals, rejections: rejections(competence) }
    end
  end
end
```

Run: `docker compose exec -T -w /rails/.claude/mod16-exporter api bundle exec rspec spec/services/ledi/alert_spec.rb spec/queries/ledi/production_summary_spec.rb`
Expected: PASS.

- [ ] **Step 3: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod16-exporter add app/services/ledi/alert.rb app/queries/ledi/production_summary.rb spec/services/ledi/alert_spec.rb spec/queries/ledi/production_summary_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod16-exporter commit -m "feat: summarize a LEDI competence with deadline and alert level

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 12: `GET /production` e `POST /production/fichas/:id/resend`

**Files:**
- Create: `app/controllers/production_controller.rb`, `app/policies/production_policy.rb`, `app/commands/ledi/resend.rb`
- Modify: `config/routes.rb`
- Test: `spec/requests/production_spec.rb`, `spec/commands/ledi/resend_spec.rb`

**Interfaces:**
- Consumes: `Ledi::ProductionSummary`, `Ledi::Deadline`, `Ledi::Observations.resend_uuid_policy`, `Ledi::Transport.rewrap`, `Ledi::DeliverJob`, `Platform::Features.enabled?`, `MfaStepUp`.
- Produces:
  - `GET /production?competence=AAAAMM&page=N` (com `fichas_total`) e `POST /production/fichas/:id/resend` (200 com o objeto da ficha, sem envelope) exatamente como contratos §5.3.
  - `Ledi::Resend.call(entry:) -> LediOutboxEntry` (levanta `Ledi::Resend::NotRejected`).
  - `ProductionPolicy#read?` (`municipal_admin` ou `analyst`), `#resend?` (`municipal_admin`).

- [ ] **Step 1: Escreva as specs (falham)**

```ruby
# spec/commands/ledi/resend_spec.rb
require "rails_helper"

# Contratos §5.3: reenviar só ficha recusada; a regra do uuid vem da prova
# técnica (pec_observations.yml → resend_uuid_policy).
RSpec.describe Ledi::Resend do
  let(:city) { ledi_ready!(register_test_city!, pec_url: "https://pec.a.test") }
  let(:entry) do
    allow(Ledi::DeliverJob).to receive(:perform_later)
    Ledi::Enqueue.call(Ledi::Fichas::Synthetic.new(cnes: "1234567", ine: "0000123456",
                                                   professional_cns: "700000000000005", cbo: "225142",
                                                   attended_at: Time.current), city: city)
  end

  before { entry.update!(status: "rejected", last_error: "CNES inválido", attempts: 1) }

  it "mesmo uuid: volta a pending para agora, limpa o erro e dispara o envio" do
    allow(Ledi::Observations).to receive(:resend_uuid_policy).and_return(:same)
    uuid = entry.uuid
    freeze_time do
      described_class.call(entry: entry)
      expect(entry.reload.slice(:status, :uuid, :last_error, :next_attempt_at))
        .to eq("status" => "pending", "uuid" => uuid, "last_error" => nil, "next_attempt_at" => Time.current)
    end
    expect(Ledi::DeliverJob).to have_received(:perform_later).twice
  end

  it "uuid novo: troca no transporte e na ficha" do
    allow(Ledi::Observations).to receive(:resend_uuid_policy).and_return(:new)
    old = entry.uuid
    described_class.call(entry: entry)
    entry.reload
    expect(entry.uuid).not_to eq(old)
    expect(entry.uuid).to start_with("1234567-")
    expect(Ledi::Transport.read(entry.bytes).uuidDadoSerializado).to eq(entry.uuid)
  end

  it "ficha que não está recusada: NotRejected" do
    entry.update_columns(status: "pending")
    expect { described_class.call(entry: entry.reload) }.to raise_error(described_class::NotRejected)
  end
end
```

```ruby
# spec/requests/production_spec.rb
require "rails_helper"

# Contratos §5.3. Exige ledi_export LIGADO; municipal_admin e analyst leem; só
# municipal_admin reenvia, com step-up.
RSpec.describe "Produção e-SUS", type: :request do
  include ActiveSupport::Testing::TimeHelpers

  def body = JSON.parse(response.body)
  def status_and_error = [ response.status, body["error"] ]

  let(:city) { City.find_by!(slug: TEST_CITY_A.slug) }
  let(:admin) do
    staff_with("prod-admin@cidade.gov.br", "municipal_admin").tap do |u|
      Mfa::Enroll.call(u)
      u.update!(otp_enabled: true)
    end
  end

  before do
    ledi_ready!(city, pec_url: "https://pec.a.test", record_mode: "record")
    allow(Ledi::DeliverJob).to receive(:perform_later)
  end

  def entry!(status, competence: Ledi::Deadline.current(Time.zone.today), error: nil)
    attrs = { uuid: "1234567-#{SecureRandom.uuid}", ficha_type: "procedimento", competence: competence,
              source_type: "synthetic", source_id: SecureRandom.uuid, ledi_version: "8.7.0",
              next_attempt_at: Time.current, status: status, last_error: error }
    status == "accepted" ? attrs[:accepted_at] = Time.current : attrs[:bytes] = "x".b
    LediOutboxEntry.create!(attrs)
  end

  it "devolve o resumo da competência corrente no formato do contrato" do
    entry!("accepted")
    rejected = entry!("rejected", error: "CPF 12345678909 inválido".then { Ledi::ErrorText.sanitize(_1) })
    sign_in_as(admin)
    get "/production"

    expect(response).to have_http_status(:ok)
    competence = Ledi::Deadline.current(Time.zone.today)
    expect(body.keys).to match_array(%w[competence deadline_on business_days_left alert counts rejections fichas fichas_total])
    expect(body["competence"]).to eq(competence)
    expect(body["deadline_on"]).to eq(Ledi::Deadline.on(competence).iso8601)
    expect(body["counts"]).to eq("accepted" => 1, "rejected" => 1, "pending" => 0, "sending" => 0, "failed" => 0)
    expect(body["fichas"].map(&:keys).uniq).to eq([ %w[id ficha_type status attempts last_error created_at accepted_at] ])
    expect(body["fichas"].find { |f| f["id"] == rejected.id }["last_error"]).to eq("CPF [número] inválido")
  end

  # Review Focus 3.
  it "rejections agrupa a mensagem já mascarada e nenhum payload sai" do
    2.times { entry!("rejected", error: Ledi::ErrorText.sanitize("CNS 898001160660761 sem vínculo")) }
    sign_in_as(admin)
    get "/production"
    expect(body["rejections"]).to eq([ { "message" => "CNS [número] sem vínculo", "count" => 2 } ])
    expect(response.body).not_to include("898001160660761", "payload")
  end

  it "competência pedida, paginação de 50 e competência inválida" do
    51.times { entry!("pending", competence: "202609") }
    sign_in_as(admin)
    get "/production", params: { competence: "202609" }
    expect(body["fichas"].size).to eq(50)
    expect(body["fichas_total"]).to eq(51)
    get "/production", params: { competence: "202609", page: 2 }
    expect(body["fichas"].size).to eq(1)
    get "/production", params: { competence: "2026-09" }
    expect(status_and_error).to eq([ 422, "invalid_competence" ])
  end

  it "analyst lê; viewer não; sem sessão 401" do
    sign_in_as(staff_with("analista@cidade.gov.br", "analyst"))
    get "/production"
    expect(response).to have_http_status(:ok)
    sign_in_as(staff_with("viewer@cidade.gov.br", "viewer"))
    get "/production"
    expect(status_and_error).to eq([ 403, "missing_role" ])
  end

  it "interruptor desligado: 403 feature_disabled nas duas rotas" do
    entry = entry!("rejected", error: "x")
    ledi_off!(city)
    sign_in_as(admin).update!(mfa_verified_at: Time.current)
    get "/production"
    expect([ response.status, body ]).to eq([ 403, { "error" => "feature_disabled", "feature" => "ledi_export" } ])
    post "/production/fichas/#{entry.id}/resend"
    expect([ response.status, body ]).to eq([ 403, { "error" => "feature_disabled", "feature" => "ledi_export" } ])
  end

  it "reenvio: municipal_admin com step-up; 409 not_rejected; 404; analyst 403; sem step-up 401" do
    allow(Ledi::Observations).to receive(:resend_uuid_policy).and_return(:same)
    rejected = entry!("rejected", error: "CNES inválido")
    pending = entry!("pending")

    sign_in_as(admin)
    post "/production/fichas/#{rejected.id}/resend"
    expect(status_and_error).to eq([ 401, "mfa_required" ])

    sign_in_as(staff_with("analista2@cidade.gov.br", "analyst")).update!(mfa_verified_at: Time.current)
    post "/production/fichas/#{rejected.id}/resend"
    expect(status_and_error).to eq([ 403, "missing_role" ])

    sign_in_as(admin).update!(mfa_verified_at: Time.current)
    post "/production/fichas/#{rejected.id}/resend"
    expect(response).to have_http_status(:ok)
    expect(body.slice("id", "status", "last_error")).to eq("id" => rejected.id, "status" => "pending", "last_error" => nil)
    post "/production/fichas/#{pending.id}/resend"
    expect(status_and_error).to eq([ 409, "not_rejected" ])
    post "/production/fichas/#{SecureRandom.uuid}/resend"
    expect(status_and_error).to eq([ 404, "not_found" ])
  end
end
```

Run: `docker compose exec -T -w /rails/.claude/mod16-exporter api bundle exec rspec spec/commands/ledi/resend_spec.rb spec/requests/production_spec.rb`
Expected: FAIL com `uninitialized constant Ledi::Resend` / rota inexistente.

- [ ] **Step 2: Implemente**

```ruby
# app/commands/ledi/resend.rb
# "Reenviar" ficha recusada (spec §6.5; contratos §5.3). O uuid segue a prova
# técnica: mesmo uuid quando o PEC aceitou o mesmo uuid depois de um 400, uuid
# novo (no transporte e na ficha) quando não aceitou.
module Ledi
  module Resend
    class NotRejected < StandardError; end

    module_function

    def call(entry:)
      entry.with_lock do
        raise NotRejected unless entry.status == "rejected"

        attrs = { status: "pending", last_error: nil, next_attempt_at: Time.current }
        if Ledi::Observations.resend_uuid_policy == :new
          uuid = "#{entry.uuid.split('-').first}-#{SecureRandom.uuid}"
          attrs.merge!(uuid: uuid, bytes: Ledi::Transport.rewrap(entry.bytes, uuid: uuid))
        end
        entry.update!(attrs)
      end
      Ledi::DeliverJob.perform_later
      entry
    end
  end
end
```

O CHECK `ck_ledi_outbox_rejected_error` exige `last_error` só em `rejected`; voltar a `pending` com `last_error: nil` é permitido.

```ruby
# app/policies/production_policy.rb
# Produção e-SUS (spec §6.5; contratos §5.3): municipal_admin e analyst leem;
# só o municipal_admin reenvia.
class ProductionPolicy < ApplicationPolicy
  def read?
    role?(:municipal_admin) || role?(:analyst)
  end

  def resend?
    role?(:municipal_admin)
  end
end
```

```ruby
# app/controllers/production_controller.rb
# Painel "Produção e-SUS" da cidade (ADR 0028; spec §6.5; contratos §5.3).
# Exige ledi_export LIGADO (não precisa estar utilizável: com a credencial
# recusada a cidade ainda vê o que está parado). Nunca devolve payload.
class ProductionController < ApplicationController
  include Authentication
  include MfaStepUp
  include ScalarParams

  wrap_parameters false

  PER_PAGE = 50

  def show
    return forbid unless policy.read?
    return feature_disabled unless feature_enabled?

    competence = optional_scalar_param(:competence) || Ledi::Deadline.current(Time.zone.today)
    return render(json: { error: "invalid_competence" }, status: :unprocessable_entity) unless Ledi::Deadline.valid?(competence)

    summary = Ledi::ProductionSummary.call(competence: competence, today: Time.zone.today,
                                           record_mode: Current.city.record_mode)
    render json: {
      competence: summary[:competence], deadline_on: summary[:deadline_on].iso8601,
      business_days_left: summary[:business_days_left], alert: summary[:alert],
      counts: summary[:counts], rejections: summary[:rejections], fichas: fichas(competence),
      fichas_total: LediOutboxEntry.for_competence(competence).count
    }
  end

  def resend
    return forbid unless policy.resend?
    return feature_disabled unless feature_enabled?
    return require_step_up! unless reauthenticated_recently?

    entry = LediOutboxEntry.find_by(id: params[:id])
    return render(json: { error: "not_found" }, status: :not_found) unless entry

    render json: ficha_json(Ledi::Resend.call(entry: entry))
  rescue Ledi::Resend::NotRejected
    render json: { error: "not_rejected" }, status: :conflict
  end

  private

  def policy = ProductionPolicy.new(Current.user, nil)

  def forbid = render(json: { error: "missing_role" }, status: :forbidden)

  def feature_enabled? = Platform::Features.enabled?(Current.city, :ledi_export)

  def feature_disabled
    render json: { error: "feature_disabled", feature: "ledi_export" }, status: :forbidden
  end

  def fichas(competence)
    page = [ optional_scalar_param(:page).to_i, 1 ].max
    LediOutboxEntry.for_competence(competence).order(created_at: :desc, id: :desc)
                   .offset((page - 1) * PER_PAGE).limit(PER_PAGE).map { |entry| ficha_json(entry) }
  end

  def ficha_json(entry)
    { id: entry.id, ficha_type: entry.ficha_type, status: entry.status, attempts: entry.attempts,
      last_error: entry.last_error, created_at: entry.created_at.iso8601, accepted_at: entry.accepted_at&.iso8601 }
  end
end
```

Se a fundação já tiver um helper de recusa por interruptor (ex.: um concern `RequiresFeature`), use-o no lugar de `feature_enabled?`/`feature_disabled`, mantendo o corpo do contrato.

Em `config/routes.rb`, depois do bloco de campanhas:

```ruby
  # Produção e-SUS (ADR 0028; contratos §5.3).
  get  "/production", to: "production#show"
  post "/production/fichas/:id/resend", to: "production#resend"
```

Run: `docker compose exec -T -w /rails/.claude/mod16-exporter api bundle exec rspec spec/commands/ledi/resend_spec.rb spec/requests/production_spec.rb`
Expected: PASS.

- [ ] **Step 3: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod16-exporter add app/controllers/production_controller.rb app/policies/production_policy.rb app/commands/ledi/resend.rb config/routes.rb spec/commands/ledi/resend_spec.rb spec/requests/production_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod16-exporter commit -m "feat: add the city's e-SUS production panel and rejected ficha resend

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 13: Resumo na plataforma e `GET /city_production` no console

**Files:**
- Create: `db/platform_migrate/20261007100002_create_city_production_summaries.rb`, `app/models/city_production_summary.rb`, `app/jobs/ledi/publish_production_job.rb`, `app/queries/ledi/city_production_query.rb`, `app/controllers/operators/city_production_controller.rb`
- Modify: `db/platform_schema.rb`, `config/recurring.yml`, `config/routes.rb`
- Test: `spec/jobs/ledi/publish_production_job_spec.rb`, `spec/requests/operators/city_production_spec.rb`, `spec/models/create_city_production_summaries_migration_spec.rb`

**Interfaces:**
- Consumes: `Ledi::ProductionSummary.counts`, `Ledi::Deadline`, `Ledi::Alert`, `Terminology::SigtapStatus.call(today:)` e `TerminologyRelease` (fundação; o segundo só na spec), `City#record_mode`.
- Produces:
  - Tabela de plataforma `city_production_summaries` (`city_id`, `competence`, `accepted`, `rejected`, `pending`, `sending`, `failed`, `published_at`; único `(city_id, competence)`); `CityProductionSummary < PlatformRecord`.
  - `Ledi::PublishProductionJob` (`prepend EachCityJob`, a cada 10 min, `housekeeping`): grava a competência corrente e a anterior; mantém 13 competências.
  - `Ledi::CityProductionQuery.call(now: Time.current) -> Hash` no formato de `data` de contratos §4.3; `GET /city_production` → `{ "data": { "cities": [...], "terminology": {...} } }`.

- [ ] **Step 1: Escreva as specs (falham)**

```ruby
# spec/jobs/ledi/publish_production_job_spec.rb
require "rails_helper"

# Desvio 7: o console nunca abre banco de cidade; a cidade publica as contagens
# na plataforma (padrão de Analytics::Publish, ADR 0025).
RSpec.describe Ledi::PublishProductionJob do
  include ActiveSupport::Testing::TimeHelpers

  let!(:city) { register_test_city! }

  def entry!(status, competence)
    attrs = { uuid: "1234567-#{SecureRandom.uuid}", ficha_type: "procedimento", competence: competence,
              source_type: "synthetic", source_id: SecureRandom.uuid, ledi_version: "8.7.0",
              next_attempt_at: Time.current, status: status }
    status == "accepted" ? attrs[:accepted_at] = Time.current : attrs[:bytes] = "x".b
    attrs[:last_error] = "x" if status == "rejected"
    LediOutboxEntry.create!(attrs)
  end

  it "publica a competência corrente e a anterior, idempotente, e guarda 13" do
    travel_to Time.zone.local(2026, 11, 3, 10) do
      2.times { entry!("accepted", "202611") }
      entry!("rejected", "202610")
      CityProductionSummary.create!(city_id: city.id, competence: "202510", published_at: 1.year.ago)

      2.times { described_class.perform_now }

      rows = CityProductionSummary.where(city_id: city.id).order(:competence)
      expect(rows.pluck(:competence)).to eq(%w[202610 202611])
      expect(rows.last.slice(:accepted, :rejected, :pending)).to eq("accepted" => 2, "rejected" => 0, "pending" => 0)
      expect(rows.first.rejected).to eq(1)
    end
  end

  it "está no recurring.yml, a cada 10 minutos, na fila housekeeping" do
    task = YAML.load_file(Rails.root.join("config/recurring.yml"), aliases: true).dig("production", "ledi_publish_production")
    expect(task).to eq("class" => "Ledi::PublishProductionJob", "queue" => "housekeeping",
                       "schedule" => "every 10 minutes")
  end
end
```

```ruby
# spec/requests/operators/city_production_spec.rb
require "rails_helper"

# Contratos §4.3: resumo por cidade (corrente primeiro, depois a anterior) no
# envelope { data: ... }, do banco de PLATAFORMA; terminology com o alerta de
# SIGTAP do dia 5 (America/Sao_Paulo).
RSpec.describe "GET /city_production (console do operador)", type: :request do
  include ActiveSupport::Testing::TimeHelpers

  let(:password) { "s3nha-forte-1" }
  let!(:operator) do
    Operator.create!(email_address: "op-#{SecureRandom.hex(3)}@rotasaude.app", password: password,
                     otp_secret: ROTP::Base32.random, otp_enabled: true)
  end
  let!(:maringa) { create(:city, name: "Maringá", uf: "PR").tap { |c| c.update!(record_mode: "record") } }
  let!(:off_city) { create(:city, name: "Desligada", uf: "PR") }
  let!(:archived) { create(:city, name: "Arquivada", status: "archived").tap { |c| c.update!(record_mode: "record") } }

  def json = JSON.parse(response.body)

  def verified_login!
    host! "admin.rotasaude.app"
    post "/session", params: { email_address: operator.email_address, password: password }
    post "/session/challenge", params: { session_id: json["session_id"], code: ROTP::TOTP.new(operator.otp_secret).now }
    expect(response).to have_http_status(:ok)
  end

  it "lista as cidades ativas fora de off, competência corrente primeiro, com prazo e alerta recalculados" do
    travel_to Time.zone.local(2026, 11, 10, 12) do
      CityProductionSummary.create!(city_id: maringa.id, competence: "202610", accepted: 0, rejected: 2, pending: 1,
                                    sending: 0, failed: 0, published_at: Time.current)
      verified_login!
      get "/city_production"

      expect(response).to have_http_status(:ok)
      cities = json.dig("data", "cities")
      expect(cities.map { |c| c["slug"] }).to eq([ maringa.slug ])
      entry = cities.sole
      expect(entry.slice("name", "record_mode")).to eq("name" => "Maringá", "record_mode" => "record")
      expect(entry["competences"].map { |c| c["competence"] }).to eq(%w[202611 202610])
      expect(entry["competences"].last).to eq(
        "competence" => "202610", "deadline_on" => "2026-11-16", "business_days_left" => 5,
        "accepted" => 0, "rejected" => 2, "pending" => 1, "failed" => 0, "alert" => "attention"
      )
      expect(entry["competences"].first.slice("accepted", "rejected", "pending", "failed"))
        .to eq("accepted" => 0, "rejected" => 0, "pending" => 0, "failed" => 0)
    end
  end

  it "terminology: SIGTAP da competência corrente e o alerta do dia 5" do
    travel_to Time.zone.local(2026, 11, 5, 9) do
      verified_login!
      get "/city_production"
      expect(json.dig("data", "terminology")).to eq("sigtap_current_competence" => "202611", "sigtap_imported" => false,
                                                    "sigtap_alert" => true)
      TerminologyRelease.create!(kind: "sigtap", version: "202611", status: "active", source_sha256: "0" * 64,
                                 imported_at: Time.current)
      get "/city_production"
      expect(json.dig("data", "terminology")).to include("sigtap_imported" => true, "sigtap_alert" => false)
    end
    travel_to Time.zone.local(2026, 12, 4, 9) do
      verified_login!
      get "/city_production"
      expect(json.dig("data", "terminology")).to include("sigtap_imported" => false, "sigtap_alert" => false)
    end
  end

  it "sem sessão de operador: 401; host de cidade: 404" do
    host! "admin.rotasaude.app"
    get "/city_production"
    expect(response).to have_http_status(:unauthorized)
  end
end
```

Ajuste o `TerminologyRelease.create!` aos campos obrigatórios reais da fundação (Task 2, Step 1). Se o `host!` do console nos specs existentes for outro, copie o de `spec/requests/operators/city_analytics_spec.rb`.

Run: `docker compose exec -T -w /rails/.claude/mod16-exporter api bundle exec rspec spec/jobs/ledi/publish_production_job_spec.rb spec/requests/operators/city_production_spec.rb`
Expected: FAIL com `uninitialized constant CityProductionSummary`.

- [ ] **Step 2: Migração de plataforma e modelo**

```ruby
# db/platform_migrate/20261007100002_create_city_production_summaries.rb
# Contagens da fila LEDI por cidade e competência, publicadas pela cidade
# (Ledi::PublishProductionJob) para o console ler sem abrir banco de cidade
# (ADR 0028; padrão do ADR 0025). Só números; nada de ficha.
class CreateCityProductionSummaries < ActiveRecord::Migration[8.1]
  def change
    create_table :city_production_summaries, id: :uuid, default: -> { "gen_random_uuid()" } do |t|
      t.uuid :city_id, null: false
      t.string :competence, limit: 6, null: false
      t.integer :accepted, null: false, default: 0
      t.integer :rejected, null: false, default: 0
      t.integer :pending, null: false, default: 0
      t.integer :sending, null: false, default: 0
      t.integer :failed, null: false, default: 0
      t.datetime :published_at, null: false
      t.index %i[city_id competence], unique: true, name: "idx_city_production_summaries_cell"
      t.check_constraint "competence::text ~ '^[0-9]{4}(0[1-9]|1[0-2])$'::text",
                         name: "ck_city_production_summaries_competence"
    end
    add_foreign_key :city_production_summaries, :cities
  end
end
```

```ruby
# app/models/city_production_summary.rb
# Contagem da fila LEDI de uma cidade numa competência (desvio 7 do plano do
# exportador). Escrita só por Ledi::PublishProductionJob; lida pelo console.
class CityProductionSummary < PlatformRecord
  belongs_to :city
end
```

Run:
```bash
docker compose exec -T -w /rails/.claude/mod16-exporter api bin/rails db:migrate
docker compose exec -T -w /rails/.claude/mod16-exporter -e RAILS_ENV=test -e POSTGRES_PASSWORD=postgres api bin/rails db:migrate
/opt/homebrew/bin/git -C apps/api/.claude/mod16-exporter diff --stat db/platform_schema.rb
```
Expected: `db/platform_schema.rb` com `define(version: 2026_10_07_100002)`, o `create_table "city_production_summaries"` e o `add_foreign_key`. Diff com outra mudança → **pare** e descubra de onde veio.

```ruby
# spec/models/create_city_production_summaries_migration_spec.rb
require "rails_helper"
require Rails.root.join("db/platform_migrate/20261007100002_create_city_production_summaries.rb").to_s

RSpec.describe "Migração de plataforma 20261007100002 (CreateCityProductionSummaries): down e up" do
  def conn = PlatformRecord.connection

  def migrate(direction)
    ActiveRecord::Migration.suppress_messages { CreateCityProductionSummaries.new.exec_migration(conn, direction) }
    CityProductionSummary.reset_column_information
  end

  def fingerprint
    {
      columns: conn.select_rows("SELECT column_name, data_type, is_nullable, column_default FROM information_schema.columns " \
                                "WHERE table_name = 'city_production_summaries' ORDER BY 1"),
      indexes: conn.select_rows("SELECT indexname, indexdef FROM pg_indexes WHERE tablename = 'city_production_summaries' ORDER BY 1")
    }
  end

  it "down remove a tabela; up seguinte restaura idêntica" do
    PlatformRecord.transaction(requires_new: true) do
      before = fingerprint
      migrate(:down)
      expect(conn.table_exists?(:city_production_summaries)).to be(false)
      migrate(:up)
      expect(fingerprint).to eq(before)
      raise ActiveRecord::Rollback
    end
  ensure
    CityProductionSummary.reset_column_information
  end
end
```

- [ ] **Step 3: Job, consulta, controller e rota**

```ruby
# app/jobs/ledi/publish_production_job.rb
# Publica na plataforma as contagens da fila LEDI da cidade (competência
# corrente e anterior, no fuso da cidade) para o console (desvio 7). Recorrente
# de cidade; idempotente (upsert); guarda as últimas 13 competências.
module Ledi
  class PublishProductionJob < ApplicationJob
    prepend EachCityJob
    queue_as :housekeeping

    KEEP = 13

    def perform
      today = Time.zone.today
      city_id = Current.city.id
      now = Time.current
      rows = [ Ledi::Deadline.current(today), Ledi::Deadline.previous(today) ].map do |competence|
        { city_id: city_id, competence: competence, published_at: now }.merge(Ledi::ProductionSummary.counts(competence))
      end
      PlatformRecord.transaction(requires_new: true) do
        CityProductionSummary.upsert_all(rows, unique_by: :idx_city_production_summaries_cell)
        keep = CityProductionSummary.where(city_id: city_id).order(competence: :desc).limit(KEEP).select(:id)
        CityProductionSummary.where(city_id: city_id).where.not(id: keep).delete_all
      end
    end
  end
end
```

Em `config/recurring.yml`, depois de `ledi_deliver`:

```yaml
  ledi_publish_production:
    class: Ledi::PublishProductionJob
    queue: housekeeping
    schedule: "every 10 minutes"
```

```ruby
# app/queries/ledi/city_production_query.rb
# GET /city_production (contratos §4.3): cidades ativas com record_mode fora de
# off, competência corrente e anterior (corrente primeiro) no fuso de cada
# cidade, contagens publicadas (zero quando ainda não há linha) e prazo/alerta
# recalculados agora. Só o banco de plataforma.
module Ledi
  module CityProductionQuery
    PLATFORM_ZONE = "America/Sao_Paulo"
    ZERO = { accepted: 0, rejected: 0, pending: 0, sending: 0, failed: 0 }.freeze

    module_function

    def call(now: Time.current)
      cities = City.where(status: "active").where.not(record_mode: "off").order(:name).to_a
      summaries = CityProductionSummary.where(city_id: cities.map(&:id)).group_by(&:city_id)
      { cities: cities.map { |city| city_entry(city, summaries.fetch(city.id, []), now) }, terminology: terminology(now) }
    end

    def city_entry(city, rows, now)
      today = now.in_time_zone(city.time_zone).to_date
      by_competence = rows.index_by(&:competence)
      competences = [ Ledi::Deadline.current(today), Ledi::Deadline.previous(today) ].map do |competence|
        row = by_competence[competence]
        counts = row ? ZERO.keys.to_h { |k| [ k, row.public_send(k) ] } : ZERO
        left = Ledi::Deadline.business_days_left(competence, today: today)
        { competence: competence, deadline_on: Ledi::Deadline.on(competence).iso8601, business_days_left: left,
          accepted: counts[:accepted], rejected: counts[:rejected], pending: counts[:pending] + counts[:sending],
          failed: counts[:failed],
          alert: Ledi::Alert.level(counts: counts, business_days_left: left, record_mode: city.record_mode) }
      end
      { slug: city.slug, name: city.name, record_mode: city.record_mode, competences: competences }
    end

    # Regra do alerta de SIGTAP é da fundação (Terminology::SigtapStatus: dia ≥ 5
    # em America/Sao_Paulo sem release ativa da competência corrente); aqui só
    # renomeia `alert` para o campo do contrato.
    def terminology(now)
      status = Terminology::SigtapStatus.call(today: now.in_time_zone(PLATFORM_ZONE).to_date)
      { sigtap_current_competence: status[:sigtap_current_competence], sigtap_imported: status[:sigtap_imported],
        sigtap_alert: status[:alert] }
    end
  end
end
```

```ruby
# app/controllers/operators/city_production_controller.rb
# GET /city_production (ADR 0028; contratos §4.3): produção LEDI de todas as
# cidades, no console do operador. Só plataforma; envelope { data: ... }.
module Operators
  class CityProductionController < BaseController
    def index
      render json: { data: Ledi::CityProductionQuery.call }
    end
  end
end
```

Em `config/routes.rb`, no bloco `constraints(PlatformConsoleHost)`, depois de `get "/city_analytics", to: "city_analytics#index"`:

```ruby
      # Produção LEDI das cidades (ADR 0028; contratos §4.3). Só plataforma.
      get "/city_production", to: "city_production#index"
```

`pending` no console soma `pending` e `sending` (o contrato §4.3 não tem `sending`); o alerta usa as contagens separadas.

Run: `docker compose exec -T -w /rails/.claude/mod16-exporter api bundle exec rspec spec/jobs/ledi/publish_production_job_spec.rb spec/requests/operators/city_production_spec.rb spec/models/create_city_production_summaries_migration_spec.rb spec/config/solid_queue_configuration_spec.rb spec/architecture/database_config_parity_spec.rb`
Expected: PASS.

- [ ] **Step 4: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod16-exporter add db/platform_migrate/20261007100002_create_city_production_summaries.rb db/platform_schema.rb app/models/city_production_summary.rb app/jobs/ledi/publish_production_job.rb app/queries/ledi/city_production_query.rb app/controllers/operators/city_production_controller.rb config/recurring.yml config/routes.rb spec/jobs/ledi/publish_production_job_spec.rb spec/requests/operators/city_production_spec.rb spec/models/create_city_production_summaries_migration_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod16-exporter commit -m "feat: publish LEDI production per city and show it in the console

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Fatia 5 — Invariantes, semente e prova de ponta a ponta

### Task 14: Invariantes do exportador (ADR 0028)

**Files:**
- Modify (ou Create, se a fundação ainda não o criou): `spec/invariants/record_mode_invariants_spec.rb`

**Interfaces:**
- Consumes: tudo das Tasks 5–13; `ledi_ready!`, `ledi_off!`, `FakePec`, `stub_pec!`.
- Produces: quatro blocos de invariante, cada um com a mutação que o deixa vermelho.

- [ ] **Step 1: Escreva os invariantes**

Se o arquivo existir (da fundação), acrescente um `RSpec.describe` novo no fim; se não, crie-o com o cabeçalho abaixo.

```ruby
# spec/invariants/record_mode_invariants_spec.rb (bloco do exportador)
require "rails_helper"

# Módulo 16, critério de fechamento (ADR 0028 "Invariantes"; spec §9). Cada
# bloco tem a mutação que precisa deixá-lo vermelho (registrada no relatório da
# entrega).
RSpec.describe "Invariantes do exportador LEDI (ADR 0028)", type: :request do
  let(:city) { City.find_by!(slug: TEST_CITY_A.slug) }

  before do
    stub_pec!
    allow(Ledi::Observations).to receive(:duplicate_marker).and_return(nil)
    allow(Ledi::Observations).to receive(:session_expired_statuses).and_return([ 401 ])
  end

  def synthetic = Ledi::Fichas::Synthetic.new(cnes: "1234567", ine: "0000123456", professional_cns: "700000000000005",
                                              cbo: "225142", attended_at: Time.current)

  def enqueue!(target = city)
    allow(Ledi::DeliverJob).to receive(:perform_later)
    Ledi::Enqueue.call(synthetic, city: target)
  end

  # Mutação: tirar o `return unless Platform::Features.usable?` de
  # Ledi::DeliverJob#perform, ou o `enabled?`/`record_mode` de Ledi::Enqueue.
  it "com o interruptor desligado ou record_mode off, nenhuma ficha sai e as rotas respondem 403" do
    ledi_ready!(city, pec_url: "https://pec.a.test")
    enqueue!
    ledi_off!(city)
    Ledi::DeliverJob.perform_now
    expect(enqueue!).to be_nil
    ledi_ready!(city, pec_url: "https://pec.a.test", record_mode: "off")
    Ledi::DeliverJob.perform_now
    expect(FakePec.for("https://pec.a.test").deliveries).to be_empty

    ledi_off!(city)
    sign_in_as(staff_with("inv-admin@cidade.gov.br", "municipal_admin"))
    get "/production"
    expect([ response.status, JSON.parse(response.body)["error"] ]).to eq([ 403, "feature_disabled" ])
  end

  # Mutação: logar `@credential.secret` em Ledi::Delivery, passar a credencial
  # para o perform_later, ou incluir `secret` no last_check_message/evento.
  it "nenhuma resposta, log, evento ou argumento de job contém a credencial" do
    ledi_ready!(city, pec_url: "https://pec.a.test", username: "usuario-secreto", password: "senha-secreta-123")
    log = StringIO.new
    logger = ActiveSupport::Logger.new(log)
    allow(Rails).to receive(:logger).and_return(logger)
    ActiveJob::Base.queue_adapter = :test
    entry = Ledi::Enqueue.call(synthetic, city: city)
    FakePec.for("https://pec.a.test").delivery_replies = [ [ 401, "" ], [ 401, "" ] ]
    Ledi::DeliverJob.perform_now
    entry.update_columns(status: "pending")
    FakePec.for("https://pec.a.test").delivery_replies = [ [ 400, "recusada" ] ]
    IntegrationCredential.find_by!(kind: "ledi").update!(last_check_status: "ok", set_at: Time.current)
    Ledi::DeliverJob.perform_now

    sign_in_as(staff_with("inv-admin2@cidade.gov.br", "municipal_admin"))
    get "/production"
    surfaces = [ log.string, response.body, DomainEvent.pluck(:payload).to_json,
                 ActiveJob::Base.queue_adapter.enqueued_jobs.to_json,
                 IntegrationCredential.find_by!(kind: "ledi").last_check_message.to_s,
                 LediOutboxEntry.pluck(:last_error).to_json ]
    surfaces.each { |text| expect(text).not_to include("usuario-secreto", "senha-secreta-123") }
  end

  # Mutação: tirar o `payload: nil` de LediOutboxEntry#accept!, ou o CHECK
  # ck_ledi_outbox_accepted_payload, ou o ramo de payload do trigger.
  it "o conteúdo serializado de ficha aceita não existe mais na fila" do
    ledi_ready!(city, pec_url: "https://pec.a.test")
    entry = enqueue!
    Ledi::DeliverJob.perform_now
    expect(entry.reload.status).to eq("accepted")
    expect(LediOutboxEntry.where(status: "accepted").where.not(payload: nil)).to be_empty
    expect { entry.update_columns(payload: "x") }.to raise_error(ActiveRecord::StatementInvalid)
  end

  # Mutação: guardar o cookie no SessionCache sem o city.id na chave, ou
  # construir o PecClient com outro pec_url que não o de Current.city.
  it "uma ficha da cidade A nunca usa a credencial nem o endereço do PEC da cidade B" do
    city_b = City.find_by(slug: TEST_CITY_B.slug) ||
             create(:city, slug: TEST_CITY_B.slug, database_url: TEST_CITY_B.database_url)
    ledi_ready!(city, pec_url: "https://pec.a.test", username: "usuario-a", password: "senha-a")
    ledi_ready!(city_b, pec_url: "https://pec.b.test", username: "usuario-b", password: "senha-b")
    entry_a = enqueue!(city)
    entry_b = CityConnection.with(city_b) { enqueue!(city_b) }

    Ledi::DeliverJob.perform_now # EachCityJob: as duas cidades ativas

    pec_a = FakePec.for("https://pec.a.test")
    pec_b = FakePec.for("https://pec.b.test")
    expect(pec_a.logins.map { |l| l[:username] }.uniq).to eq([ "usuario-a" ])
    expect(pec_b.logins.map { |l| l[:username] }.uniq).to eq([ "usuario-b" ])
    expect(pec_a.deliveries.map { |d| d[:filename] }).to eq([ "#{entry_a.uuid}.esus" ])
    expect(pec_b.deliveries.map { |d| d[:filename] }).to eq([ "#{entry_b.uuid}.esus" ])
  ensure
    if city_b
      CityConnection.with(city_b) do
        ApplicationRecord.transaction do
          ApplicationRecord.connection.execute("SET LOCAL session_replication_role = replica")
          DomainEvent.where("payload ->> 'outbox_id' = ?", entry_b&.id.to_s).delete_all
          LediOutboxEntry.delete_all
          IntegrationCredential.where(kind: "ledi").delete_all
        end
      end
    end
  end
end
```

Run: `docker compose exec -T -w /rails/.claude/mod16-exporter api bundle exec rspec spec/invariants/record_mode_invariants_spec.rb`
Expected: PASS. Depois aplique **cada** mutação do comentário, uma de cada vez, rode, veja o bloco correspondente ficar vermelho, desfaça e anote no relatório.

- [ ] **Step 2: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod16-exporter add spec/invariants/record_mode_invariants_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod16-exporter commit -m "test: pin the LEDI exporter invariants of ADR 0028

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

### Task 15: Semente de dev, `ledi:enqueue_synthetic` e prova de ponta a ponta

**Files:**
- Create: `lib/ledi_crew.rb`, `lib/tasks/ledi.rake`
- Modify: `db/seeds.rb`, `spec/integration/ledi_pec_spec.rb` (exemplo pelo caminho de produção)
- Test: `spec/lib/ledi_crew_spec.rb`

**Interfaces:**
- Consumes: `LediOutboxEntry`, `Ledi::Enqueue`, `Ledi::Fichas::Synthetic`, `Ledi::DeliverJob`.
- Produces: `LediCrew.seed_current_city(slug:) -> { created: Integer }` (idempotente); `bin/rails "ledi:enqueue_synthetic[slug]"` (dev; lê `LEDI_PROOF_*`).

- [ ] **Step 1: Escreva a spec (falha)**

```ruby
# spec/lib/ledi_crew_spec.rb
require "rails_helper"
require Rails.root.join("lib/ledi_crew").to_s

# Spec §10 (parte de produção): a semente de dev dá ao painel uma competência
# com todos os estados, sem dado real e sem enviar nada.
RSpec.describe LediCrew do
  before { CityProfile.create!(name: "Maringá", ibge_code: "4115200") }

  it "semeia a competência corrente com todos os estados, uma vez" do
    expect(described_class.seed_current_city(slug: "maringa")).to eq(created: 12)
    expect(described_class.seed_current_city(slug: "maringa")).to eq(created: 0)

    competence = Ledi::Deadline.current(Time.zone.today)
    expect(LediOutboxEntry.for_competence(competence).group(:status).count)
      .to eq("accepted" => 6, "rejected" => 3, "pending" => 2, "failed" => 1)
    expect(LediOutboxEntry.where(status: "accepted").where.not(payload: nil)).to be_empty
    expect(LediOutboxEntry.where(status: "rejected").distinct.pluck(:last_error).size).to eq(2)
    expect(LediOutboxEntry.pluck(:source_type).uniq).to eq([ "synthetic" ])
  end
end
```

Run: `docker compose exec -T -w /rails/.claude/mod16-exporter api bundle exec rspec spec/lib/ledi_crew_spec.rb`
Expected: FAIL com `cannot load such file -- lib/ledi_crew`.

- [ ] **Step 2: Implemente a semente e a task**

```ruby
# lib/ledi_crew.rb
# Semente de dev da Produção e-SUS (módulo 16; spec §10). Grava direto na fila
# (as linhas não são enviadas: nascem com next_attempt_at no futuro distante e o
# interruptor da semente está desligado) uma competência corrente com todos os
# estados, para o painel do dashboard e do console terem o que mostrar.
# Identificadores fictícios; payload só nas linhas não aceitas, com a ficha
# sintética de verdade. Idempotente: não semeia se a fila já tem linha
# sintética da competência.
module LediCrew
  CNES = "9999991"
  INE = "9999999991"
  PROFESSIONAL_CNS = "700000000000005"
  CBO = "225142"
  REJECTIONS = [ "CNES 9999991 não pertence ao município da instalação.",
                 "CBO 225142 não permitido para o procedimento informado." ].freeze
  PLAN = { "accepted" => 6, "rejected" => 3, "pending" => 2, "failed" => 1 }.freeze

  module_function

  def seed_current_city(slug:)
    competence = Ledi::Deadline.current(Time.zone.today)
    return { created: 0 } if LediOutboxEntry.for_competence(competence).where(source_type: "synthetic").exists?

    city = City.find_by(slug: slug)
    created = 0
    PLAN.each do |status, count|
      count.times do |index|
        create_entry(city, status, index)
        created += 1
      end
    end
    { created: created }
  end

  def create_entry(city, status, index)
    ficha = Ledi::Fichas::Synthetic.new(cnes: CNES, ine: INE, professional_cns: PROFESSIONAL_CNS, cbo: CBO,
                                        attended_at: Time.zone.today.beginning_of_month.in_time_zone + 9.hours)
    uuid = "#{CNES}-#{SecureRandom.uuid}"
    attrs = { uuid: uuid, ficha_type: ficha.type, competence: ficha.competence, source_type: "synthetic",
              source_id: ficha.source_id, ledi_version: Ledi::Version::ACTIVE, status: status,
              next_attempt_at: 10.years.from_now, attempts: status == "pending" ? 0 : 1 }
    if status == "accepted"
      attrs.merge!(accepted_at: Time.current, first_attempt_at: Time.current)
    else
      attrs[:bytes] = Ledi::Transport.wrap(ficha, city: city || Struct.new(:id).new(SecureRandom.uuid), uuid: uuid)
      attrs[:last_error] = REJECTIONS[index % REJECTIONS.size] if status == "rejected"
      attrs[:last_error] = "HTTP 503" if status == "failed"
      attrs[:first_attempt_at] = 2.days.ago if status == "failed"
    end
    LediOutboxEntry.create!(attrs)
  end
end
```

```ruby
# lib/tasks/ledi.rake
namespace :ledi do
  # Prova no navegador (spec §9): enfileira UMA ficha sintética na cidade, com os
  # identificadores do PEC local (LEDI_PROOF_CNES/INE/CNS/CBO, ver
  # docs/operacao/pec-local-dev.md). Só development; nada acontece com o
  # interruptor desligado.
  desc "Enfileira uma ficha sintética LEDI na cidade (só development)"
  task :enqueue_synthetic, [ :slug ] => :environment do |_t, args|
    abort "ledi:enqueue_synthetic só existe em development" if Rota.deployed? || !Rails.env.development?

    city = City.find_by!(slug: args.fetch(:slug))
    CityConnection.with(city) do
      ficha = Ledi::Fichas::Synthetic.new(cnes: ENV.fetch("LEDI_PROOF_CNES"), ine: ENV.fetch("LEDI_PROOF_INE"),
                                          professional_cns: ENV.fetch("LEDI_PROOF_CNS"),
                                          cbo: ENV.fetch("LEDI_PROOF_CBO"), attended_at: 1.hour.ago)
      entry = Ledi::Enqueue.call(ficha, city: city)
      puts(entry ? "[ledi] #{city.slug}: ficha #{entry.id} enfileirada (#{entry.competence})" :
                   "[ledi] #{city.slug}: interruptor ledi_export desligado ou record_mode=off — nada enfileirado")
    end
  end
end
```

Em `db/seeds.rb`, logo depois do bloco do Analytics (`analytics = AnalyticsCrew.seed_current_city(...)` e seus `puts`):

```ruby
        # ── Produção e-SUS (módulo 16, spec §10) ──────────────────────────────
        # Fila LEDI com todos os estados na competência corrente; nada é
        # enviado (record_mode=off e interruptor desligado na semente).
        ledi = LediCrew.seed_current_city(slug: slug)
        puts "[seeds] produção e-SUS .. #{ledi[:created]} fichas sintéticas na fila (não enviadas)"
```

e, no topo de `db/seeds.rb`, junto dos outros `require` de `lib/` (se houver; senão, antes do primeiro uso), `require Rails.root.join("lib/ledi_crew").to_s`.

Run: `docker compose exec -T -w /rails/.claude/mod16-exporter api bundle exec rspec spec/lib/ledi_crew_spec.rb`
Expected: PASS.

- [ ] **Step 3: Exemplo `:pec` pelo caminho de produção**

No fim de `spec/integration/ledi_pec_spec.rb`, antes do último `end`:

```ruby
  it "caminho de produção: Enqueue + DeliverJob entregam e a ficha vira accepted" do
    city = register_test_city!
    ledi_ready!(city, pec_url: ENV.fetch("LEDI_PEC_URL"), username: ENV.fetch("LEDI_PEC_USERNAME"),
                      password: ENV.fetch("LEDI_PEC_PASSWORD"), ibge_code: ENV.fetch("LEDI_PROOF_IBGE"))
    allow(Ledi::DeliverJob).to receive(:perform_later)
    entry = Ledi::Enqueue.call(
      Ledi::Fichas::Synthetic.new(cnes: ENV.fetch("LEDI_PROOF_CNES"), ine: ENV.fetch("LEDI_PROOF_INE"),
                                  professional_cns: ENV.fetch("LEDI_PROOF_CNS"), cbo: ENV.fetch("LEDI_PROOF_CBO"),
                                  attended_at: 1.hour.ago),
      city: city
    )
    Ledi::DeliverJob.perform_now
    expect(entry.reload.slice(:status, :payload)).to eq("status" => "accepted", "payload" => nil)
  end
```

Run: o comando do Step 12 da Task 1.
Expected: os quatro exemplos rodam; o novo PASSA.

- [ ] **Step 4: Prova no navegador (spec §9)**

Com o PEC local de pé, a fundação, o maintenance e o dashboard rodando contra o api do worktree:
1. Semeie: `docker compose exec -T -w /rails/.claude/mod16-exporter api bin/rails db:seed`.
2. Console (`admin`): em Maringá, `record_mode=integrated` e `pec_url=https://pec.rota.test:8443`.
3. Maintenance: ligue `ledi_export` em Maringá.
4. Dashboard (admin de Maringá, step-up): cadastre a credencial LEDI do PEC local e "Testar conexão" → `ok`.
5. `docker compose exec -T -w /rails/.claude/mod16-exporter -e LEDI_PEC_CA_FILE=/rails/.claude/mod16-exporter/deploy/development/pec/local/ca.pem -e LEDI_PROOF_CNES -e LEDI_PROOF_INE -e LEDI_PROOF_CNS -e LEDI_PROOF_CBO api bin/rails "ledi:enqueue_synthetic[maringa]"` (o worker precisa da mesma `LEDI_PEC_CA_FILE`; rode `bin/rails runner 'Ledi::DeliverJob.perform_now'` com as mesmas variáveis se o worker principal não a tiver).
6. Dashboard → Produção e-SUS: a ficha nova aparece `accepted`; no PEC, a ficha aparece na tela anotada na Task 1.

Registre o resultado (prints/texto) no relatório da entrega. A semente de dev volta Maringá a `record_mode=off` na próxima `db:seed`.

- [ ] **Step 5: Suíte completa**

```bash
docker compose stop worker
docker compose exec -T -w /rails/.claude/mod16-exporter api bundle exec rspec
docker compose start worker
```
Expected: 0 falhas (os `:pec` ficam excluídos sem `LEDI_PEC_URL`).

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod16-exporter add lib/ledi_crew.rb lib/tasks/ledi.rake db/seeds.rb spec/lib/ledi_crew_spec.rb spec/integration/ledi_pec_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod16-exporter commit -m "feat: seed the dev LEDI queue and add the synthetic enqueue task

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Self-review

1. **Cobertura da spec.** §6.1 prova técnica → Task 1 (PEC, credencial, classes, ficha, critério e parada, observações). §6.2 layout → Tasks 2–4 (`vendor/ledi/8.7.0`, `bin/ledi-generate`, `Ledi::Version::ACTIVE`, `Ledi::Transport.wrap`, `Ledi::Ficha`, `Ledi::Fichas::Synthetic`). §6.3 fila e trigger → Task 5. §6.4 envio (recorrente, ao enfileirar, `usable?`, `SKIP LOCKED`, cookie em cache, 200/400/5xx/timeout/401/401 duplo, eventos) → Tasks 6–9. §6.5 painel, prazo, alertas, console, reenvio → Tasks 10–13. §8 LGPD (cifra, `last_error` saneado, nada de segredo em log/evento/argumento) → Tasks 5, 7, 8, 14. §9 testes (prova `:pec`, contrato LEDI, tabela de casos, threads, invariantes) → Tasks 1, 2, 8, 9, 14, 15. §10 semente (parte de produção) → Task 15. §11 rollout: `city:migrate:all` e `db:migrate` de plataforma, tudo desligado → Global Constraints.
2. **Placeholders.** O único arquivo cujo conteúdo final depende de observação é `config/ledi/pec_observations.yml` (Task 1, Step 13), com formato fixo e regra explícita; o código lê as chaves por `Ledi::Observations`.
3. **Consistência de tipos.** `LediOutboxEntry#retry_later!(error:, wait:, give_up_after:, now:)` (Task 5) é a chamada de `Ledi::Delivery#retry_later` (Task 8); `Ledi::PecClient::Reply` (Task 1) é o que `FakePec#deliver` devolve (Task 2); `Ledi::ProductionSummary.counts` (Task 11) alimenta `PublishProductionJob` (Task 13); `Ledi::Transport.rewrap` (Task 3) é usado por `Ledi::Resend` (Task 12).
4. **Review Focus.** Cada uma das cinco linhas tem teste na task dona (citadas na seção).

## Divergências propostas ao contrato

Já incorporadas ao contrato pelo coordenador e seguidas aqui: `codIbge` de `city_profile.ibge_code`; `GET /city_production` no envelope `{ "data": ... }`, competência corrente primeiro e `terminology.sigtap_alert`; `fichas_total` em `GET /production`; resposta 200 do reenvio como objeto sem envelope; prazo de 202610 = 2026-11-16 (caso da Task 10).

1. **§4.3 — `pending` no console soma `sending`.** O item de competência do console não tem `sending`; uma ficha em envio conta como pendente. Proposta: acrescentar `sending` (opcional) ou registrar a soma no contrato.
2. **§5.3 — `competence` malformada.** O contrato não diz o erro; este plano responde `422 { "error": "invalid_competence" }`.
3. **§6 — trilha do reenvio.** `POST /production/fichas/:id/resend` é ação com step-up sem evento; proposta: `ledi.ficha_resent { outbox_id, user_id }` (não implementado aqui).
4. **§4.3/§5.3 — `attention` conta `failed`.** O contrato diz "pendente/recusada"; este plano inclui `failed` (desvio 8). Registrar no contrato.
