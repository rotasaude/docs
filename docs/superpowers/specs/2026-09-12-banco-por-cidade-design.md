# Banco por cidade (database-per-tenant)

**Date:** 2026-09-12
**Status:** Approved (brainstorming → 5 seções aprovadas → spike validado → plan pendente)
**Substitui:** ADR-0003 (Multi-tenant isolation via RLS) integralmente
**Revisa:** ADR-0011 (identidade), ADR-0012 (memberships), ADR-0013 (provisionamento e custódia de chaves), ADR-0015 (contratos — MAJOR em `events`)
**Touches:** `apps/api` (schema, conexão, auth, jobs, provisionamento, ~metade da suíte), `apps/admin`, `apps/dashboard`, `apps/wpda`, `contracts/events`, `docker-compose.yml`, `apps/api/deploy/`
**Não toca:** `contracts/protocols` (o schema de protocolo não menciona tenant)

## Problem

O Rota Saúde é hoje uma instalação única multi-tenant: um banco, um schema, e a
coluna `municipality_id` distinguindo a cidade, com isolamento imposto por
Row-Level Security (ADR-0003). Quatro exigências tornam esse modelo insuficiente:

1. **Contratual/regulatória.** Prefeituras exigem evidência de separação física
   dos dados da cidade. "Existe uma policy no Postgres" não é uma resposta que se
   apresenta em licitação.
2. **Risco de vazamento.** O namespace `/admin/api` roda inteiro sob `rota_admin`
   (BYPASSRLS) por decisão documentada em
   `apps/api/app/controllers/admin/api/base_controller.rb:12-21`. Ali o
   isolamento é 100% código — e tem furo (ver *Current state*).
3. **Ciclo de vida por cidade.** Backup, restore, purga LGPD, exportação e
   offboarding hoje são `DELETE ... WHERE municipality_id = ?` em 16 tabelas,
   torcendo para não sobrar nada.
4. **Soberania / on-premise.** A venda de instalação própria por prefeitura não
   é viável enquanto o app pressupõe uma base compartilhada.

## Current state (verified)

### O que está isolado, e por quê

- **9 tabelas** têm política `tenant_isolation` com `USING` e `WITH CHECK` em
  `municipality_id = current_setting('app.municipality_id')::uuid`, `FORCE ROW
  LEVEL SECURITY`, e dono `rota_admin` — de modo que `rota_app` nunca escapa por
  ownership: `consents`, `conversations`, `dashboard_metrics`, `domain_events`,
  `inbound_messages`, `outbound_messages`, `protocol_definitions`,
  `report_snapshots`, `triages`.
- **16 tabelas** carregam `municipality_id`. As 7 restantes — `alert_recipients`,
  `authors`, `consent_terms`, `invitations`, `memberships`,
  `municipality_channels`, `processed_events` — **não têm política nenhuma**.
  `lib/migration_helpers/rls.rb` existe exatamente para isso e não é usado por
  migration alguma.
- Quatro entradas setam o tenant, todas fail-closed:
  `app/controllers/concerns/tenant_scoped_request.rb:21`,
  `app/jobs/concerns/tenant_scoped_job.rb:12`,
  `app/services/whatsapp/ingest.rb:28`, e os commands de setup.
- Dois desvios deliberados para `rota_admin`: `AdminRoleJob` (via `prepend`) e
  todo o `/admin/api`.

### O furo

`Admin::Api::BaseController#resolve_municipality` cai em
`first_member_municipality` (`base_controller.rb:70-78`), que devolve **`nil`**
quando o usuário não tem membership ativo não-operador. `Admin::Scoped` trata
`nil` como `:all` (`app/queries/admin/scoped.rb:21-24`), devolvendo
`Model.all` — sob conexão BYPASSRLS, isso são **todas as cidades**.

O caminho de chegada existe: `RevokeMembership` só marca `revoked_at`
(`app/commands/revoke_membership.rb:7`) e **não destrói as sessions**, diferente
de `DeactivateUser`, que chama `user.deactivate!` → `sessions.destroy_all`
(`app/models/user.rb:35`). Um `municipal_admin` recém-revogado mantém cookie
válido e passa a enxergar a plataforma inteira. Nenhum spec cobre o caso.

### Escala e dados

- 3 municípios, 49 conversas, 29 triagens, 3 usuários, 161 eventos no banco de
  desenvolvimento — tudo semente e demo.
- Alvos de deploy apontam para `dev.rota-saude.example` e
  `web1.rota-saude.example`: **não há produção**.
- 81 arquivos de spec. Destes, **41 mencionam `municipality_id`**, 17 usam
  `SET LOCAL`, 15 abrem `connected_to(role: :admin)`.
- Rails 8.1.3, Solid Queue 1.4.0, Solid Cache 1.0.10.

## Spike: registro de shard em runtime (executado 2026-09-12)

**Pergunta:** o ActiveRecord 8.1 registra um banco novo em runtime e o atende sem
restart, sem derrubar pools ativos e sem cruzar threads?

| Verificação | Resultado |
|---|---|
| Ler shard declarado no boot | ok |
| Shard desconhecido | levanta `ConnectionNotDefined` — falha fechada |
| Registrar cidade em runtime via `connects_to` | funciona parado; **sob tráfego, 171.620 interrupções** em cidades já ativas |
| Registrar via `connection_handler.establish_connection` | **0,11 ms, zero interrupções**, 2.046 leituras concluídas na mesma janela |
| 1.200 leituras concorrentes em 2 shards | zero cruzamento |
| Pools | preguiçosos — shard registrado e não usado fica com 0 conexões |

**Achado estruturante:** `connects_to` reconstrói o mapa inteiro de shards.
Chamá-lo de novo para adicionar uma cidade derruba as vizinhas durante a janela.
`establish_connection` direto no handler adiciona um pool isolado.

Isso vira invariante de arquitetura, não detalhe de implementação. E força o
registro a ser **preguiçoso por processo**: com Puma multi-worker e réplicas,
provisionar não pode "avisar" os processos — cada um descobre a cidade na
primeira requisição e se auto-corrige.

## Spike 2: worker por cidade (executado 2026-09-12)

**Pergunta:** um processo pai consegue forkar um supervisor Solid Queue por
cidade, cada um ligado ao banco da sua cidade, sem cruzamento — e ganhar uma
cidade nova em runtime, sem restart nem deploy?

| Verificação | Resultado |
|---|---|
| 2 supervisores forkados, cada um no seu banco | ok — sonda grava o banco real de execução; zero cruzamento |
| Cidade nova (C) sobe com A e B rodando | ok — sem restart de A/B |
| A e B seguem processando depois de C entrar | ok |
| `SIGTERM` no pai | todos encerram; zero órfãos |
| Pidfile | `SolidQueue.supervisor_pidfile` é **`nil`** por default neste app — não há colisão hoje. Se alguém setar (padrão comum em healthcheck de Kamal), tem de ser por cidade |
| Cidade com banco inacessível | filho morre em **0,4 s** com `ActiveRecord::NoDatabaseError` e `exit=1`; **cidade saudável segue processando** |

**Custo de processo medido:** 8 processos ruby/solid para 3 cidades com *um*
worker cada (~2,7 por cidade). Com os 2–3 workers propostos em §4, esperar 4–5
por cidade. Confirma que o `config/queue.yml` atual (5 workers → ~7 processos por
cidade) não escala.

**Consequência de desenho:** a falha de uma cidade é contida por construção
(processos separados) e é **rápida e ruidosa**, não silenciosa. O gerente de
processos precisa de política de *restart com backoff*, não de timeout — um filho
cujo banco sumiu morre em menos de um segundo e reiniciaria em loop apertado sem
backoff.

**Não coberto:** escala real (o spike rodou 3 cidades e poucos jobs), o mecanismo
de observação do catálogo, e o rollout de deploy sobre N supervisores.

## Decision

### 1. Topologia

Um código, um deploy, **N bancos**. Migrations universais; consumo por cidade.
O app deixa de ser multi-tenant em runtime: a cidade passa a ser o banco, não um
valor de coluna.

O papel web é compartilhado e resolve a conexão por subdomínio. O worker é **por
cidade**, porque um supervisor do Solid Queue liga-se a um único banco de fila, e
a fila precisa morar no banco da cidade — o payload de `ProcessInboundMessageJob`
e `SendWhatsappJob` carrega telefone e conteúdo de mensagem.

### 2. Fundação: catálogo e resolução

**Banco de plataforma** (`rota_saude_platform`), via classe abstrata
`PlatformRecord` com conexão fixa. Guarda apenas o que precisa existir antes de
saber a cidade — **nunca dado de cidadão**:

| Tabela | Papel | Origem |
|---|---|---|
| `cities` | catálogo: `slug` (= subdomínio), nome, uf, status, `database_url` cifrada, chave de cifra | `municipalities` |
| `city_channels` | `phone_number_id → city_id`, credencial WhatsApp | `municipality_channels` |
| `unknown_channels` | `phone_number_id` sem dono | já é órfã hoje |
| `operators`, `operator_sessions` | contas de plataforma | o `platform_operator` de hoje |
| `platform_events` | auditoria platform-scope | `Platform.audit` |

**Banco da cidade:** todo o resto — as 9 tabelas do plano de dados, mais `users`,
`sessions`, `identities`, `memberships`, `invitations`, `authors`,
`consent_terms`, `alert_recipients`, `processed_events`, mais Solid Queue e Solid
Cache. Schema idêntico em todas, **sem `municipality_id` em lugar nenhum**.

**Resolução**, invertendo a ordem de hoje:

1. Middleware lê `request.host`. Subdomínios reservados (`admin`, `api`, `auth`,
   `www`) → conexão de plataforma. Qualquer outro → catálogo (em cache);
   404 se não existir, 403 se `suspended`, 503 se o schema estiver atrasado.
2. Se este processo ainda não tem pool para a cidade, `establish_connection` na
   hora. `connects_to` **só no boot, uma vez**.
3. `ApplicationController` embrulha a ação em `connected_to` daquele shard.
4. **Só então** a autenticação roda — `Session.find_by` passa a consultar o banco
   da cidade. Hoje é o contrário (`tenant_scoped_request.rb:36`). A inversão é o
   que torna o cookie de uma cidade inútil na outra sem verificação de aplicação.

O webhook do WhatsApp continua chegando sem cidade no host e resolvendo por
`phone_number_id`, agora contra o catálogo.

**Pools:** um por cidade, criado sob demanda, com `idle_timeout` devolvendo
ociosas. O teto é cidades *simultaneamente ativas* × `RAILS_MAX_THREADS` contra o
`max_connections` do Postgres — não o total de cidades cadastradas.

### 3. Desmontagem do multi-tenant

| O que morre | Onde aparece hoje |
|---|---|
| 9 políticas `tenant_isolation`, `FORCE RLS`, split de ownership | 9 tabelas, `lib/migration_helpers/rls.rb` |
| Roles `rota_app` / `rota_admin` | `connected_to(role: :admin)` em 20 arquivos + 15 specs |
| `Current.municipality_id` | 13 arquivos + 12 specs |
| `TenantScopedRequest`, `TenantScopedJob`, `with_tenant` | 12 + 8 arquivos |
| `AdminRoleJob` | 10 arquivos |
| `Admin::Scoped` (16 métodos) | 12 arquivos |
| `resolve_municipality`, `cross_tenant?`, `first_member_municipality`, `municipality_filter` | `Admin::Api::BaseController` |
| coluna `municipality_id`, FKs, `conversation_partial_index_by_tenant` | 16 tabelas, 56 arquivos, 41 specs |

Substituem: `CityConnection` (resolver + registry) e `Current.city` — que guarda
slug e nome para log e envelope, e **nunca entra em `WHERE`**.

Três peças mudam de forma:

- `DomainEvents.publish` perde `municipality_id` e a exceção `TenantMissing`. O
  payload enfileirado também perde o `municipality_id:` que hoje viaja em toda
  subscrição (`app/events/domain_events.rb:40`).
- `Platform.audit` sai do `domain_events` da cidade e vai para `platform_events`.
  Hoje grava com `municipality_id: nil` (`app/events/platform.rb:11`) — linha que
  a própria policy torna invisível para `rota_app`. O efeito colateral some.
- `Conversation.for(phone, municipality_id:)` → `Conversation.for(phone)`; a chave
  única `(municipality_id, phone)` vira `phone`.

**Identidade da cidade no domínio:** uma linha singleton `city_profile` dentro do
banco da cidade (nome, uf, settings), escrita no provisionamento — em vez de
leitura cross-banco do catálogo. É o que torna um dump restaurado sozinho
utilizável, e o que os dois jobs que hoje tocam `Municipality`
(`dispatch_municipality_alert_job`, `reconcile_consents_job`) passam a ler.

**O furo do `/admin/api` deixa de existir por construção**, não por guarda: não há
`.all` cross-tenant possível quando não há dado alheio no banco.

**O que fica pior, registrado:** hoje um `where` esquecido não vaza porque o
Postgres filtra. Depois também não vaza, mas só porque não há dado alheio na
base. A proteção inteira se desloca para o resolver: superfície **menor** (um
ponto) e **mais crítica** (errar a conexão vaza tudo de uma vez). O resolver
precisa carregar o mesmo peso de teste que o `tenant_isolation_spec` carrega hoje.

Efeito colateral bom: `processed_events` (exactly-once, ADR-0005) passa a ser por
banco — o mesmo `event_id` em cidades diferentes deixa de colidir.

### 4. Ciclo de vida da cidade

**Provisionamento em duas fases.** Criar banco exige `CREATEDB`, e o processo web
não pode ter esse privilégio — mesmo princípio de menor privilégio do ADR-0019.
`POST /setup/cities` grava no catálogo com `status: provisioning` e devolve `202`.
Um job assume, sob um role `rota_provisioner` usado só aí: cria banco e role →
migra → grava `city_profile` → cria o primeiro `municipal_admin` e o convite
**dentro do banco da cidade** → semeia `consent_terms`, `alert_recipients` e o
protocolo template → flipa para `active`.

Muda o contrato do endpoint: hoje devolve o token do convite na resposta
(`app/controllers/setup_controller.rb:36`); passa a devolver só o id.

Esse job precisa de fila, e a da cidade ainda não existe → **Solid Queue no banco
de plataforma**, para operações de ciclo de vida. Não fura o isolamento (trafega
slug, nome e e-mail de servidor público) e se paga: é onde rodam também o rollout
de migrations e o offboarding.

**Migrations universais.**

- O entrypoint **deixa de migrar** (hoje `bin/docker-entrypoint:28` migra no boot
  do web) — com N bancos, réplicas correriam entre si e o boot ficaria
  proporcional ao número de cidades.
- Migração vira passo explícito de deploy: `city:migrate:all`, iterando o
  catálogo com lock por cidade, falhando alto se alguma ficar para trás.
- **Guarda de runtime:** o resolver devolve 503 **para a cidade atrasada**, não
  para todas.
- **Deploy deixa de ser atômico.** Sempre existe janela em que código novo
  encontra schema velho → **expand/contract obrigatório em toda migração
  destrutiva**, a mesma disciplina que o ADR-0015 já exige dos contratos.

**Worker por cidade.** O `config/queue.yml` atual declara 5 workers com
`processes: 1` cada, mais dispatcher e supervisor: ~7 processos por cidade. Dez
cidades são 70 processos — não fecha. O `queue.yml` por cidade é enxugado para
2–3 workers, preservando `urgent` isolado, que é o compromisso clínico do
ADR-0006 e a única separação inegociável. As 9 tarefas de `config/recurring.yml`
passam a ser por cidade automaticamente, por viverem em
`solid_queue_recurring_tasks` no banco da cidade.

Um processo gerente forka um supervisor por cidade, cada um com
`SolidQueue::Record.establish_connection` para o banco dela — mecanismo validado
pelo spike 2, incluindo entrada de cidade nova em runtime, falha contida e
desligamento limpo no `SIGTERM`. O gerente precisa de **restart com backoff**:
um filho cujo banco esteja inacessível morre em 0,4 s e, sem backoff, reiniciaria
em loop apertado.

**Offboarding e backup.** Catálogo com quatro estados: `provisioning` → `active`
→ `suspended` → `archived`. Backup é `pg_dump` por cidade. **Não é restaurável
sozinho**: desde a chave de cifra por cidade (Plano 7), o dump carrega
ciphertext, e restaurá-lo exige as chaves de plataforma **e** o
`cities.encryption_key` daquela cidade da mesma época — restaurar num catálogo
cujo material mudou devolve dado ilegível sem erro nenhum. Procedimento e
avisos em `apps/api/README.md`, seção "Chave de cifra por cidade (Plano 7)",
bloco "Restaurar um dump".
Offboarding é suspender → dump final entregue → `DROP DATABASE`.

### 5. Identidade e frontends

`users`, `sessions`, `identities`, `memberships`, `invitations` → banco da cidade.
`operators`, `operator_sessions` → plataforma.

Simplifica o RBAC: hoje `memberships.municipality_id` é nullable **só** para
acomodar o operador global, com CHECK `ck_memberships_operator_global`. Com o
operador fora da tabela, somem a coluna e o CHECK. Sobram quatro papéis locais:
`municipal_admin`, `protocol_author`, `protocol_publisher`, `viewer`.

Custo aceito: quem atua em duas prefeituras tem duas contas e **dois enrollments
de MFA** (`otp_secret` e `otp_recovery_codes` moram em `users`).

**Cookie:** segue `httponly`, `same_site: :lax`, assinado — com uma regra nova e
inegociável: **nunca setar `domain:`**. Host-only, o cookie não viaja para outro
subdomínio. Spec de guarda obrigatório.

**Operador entrando numa cidade:** grant assinado de curta duração. O operador
autentica na plataforma, pede acesso à cidade X, e é redirecionado para
`x.rotasaude.app` com token assinado pela chave da plataforma. A cidade valida,
cria `Session` local marcada como origem-plataforma, e registra o acesso **nos
dois lados** — `domain_events` da cidade e `platform_events`. Nenhuma conta
replicada, e a prefeitura fica com trilha de toda entrada da plataforma.

**gov.br:** o OIDC valida `redirect_uri` contra o registrado, e registrar N URIs
junto ao gov.br não escala com provisionamento automático. Callback único em
`auth.rotasaude.app`, resolvendo a cidade pelo `state` e entregando com **o mesmo
mecanismo de grant** — uma peça serve os dois casos.

**Frontends:**

| App | Host | O que muda |
|---|---|---|
| `admin` | `admin.rotasaude.app` | perde painéis cross-tenant; vira catálogo, provisionamento, saúde da plataforma e "entrar na cidade X" |
| `dashboard` | `<cidade>.rotasaude.app/dashboard/` | só o host; `base` fica. **Entra aqui o `/authoring` faltando no proxy** |
| `wpda` | `<cidade>.rotasaude.app/wpda/` | idem |

- `ReportSnapshot#url` monta o link do cidadão a partir de `WPDA_PUBLIC_BASE`,
  env var única (`app/models/report_snapshot.rb:29`). Passa a derivar do slug. O
  link vai por WhatsApp — host errado quebra o relatório sem erro no servidor.
- **CORS:** a lista estática `ALLOWED_ORIGINS` não enumera N cidades.
  `Rack::Cors` aceita bloco em `origins`, validando `<slug>` contra o catálogo.
- Em dev, `*.localhost` resolve para 127.0.0.1 no Chrome e Firefox sem
  `/etc/hosts`. Safari historicamente não — nesse caso, uma linha por cidade.

### 6. Chaves por cidade

Hoje `report_signing_key` vem das credentials e `ACTIVE_RECORD_ENCRYPTION_*` do
env: **uma chave para todas as cidades**. Um dump de A seria decifrável com a
chave de B, o que enfraquece justamente o argumento de separação física.

**Decisão: chave por cidade.** A chave de cifra passa a ser atributo do catálogo,
carregada junto com a conexão. Custa custódia e rotação por cidade.

Para o token de relatório não havia risco prático (a busca é `find_by(token:)`
dentro do banco da cidade, então token de A não existe em B), mas a chave também
passa a ser por cidade, por consistência de custódia.

## Ordem de execução

Cada etapa termina com a suíte verde:

1. Banco de plataforma, `PlatformRecord` e catálogo. O app segue como está.
2. Resolver e registry de conexão (`establish_connection`), com `city:create` de
   dev. Convive com o banco velho.
3. **Schema inicial limpo.** São 33 migrations acumuladas, várias puramente sobre
   tenancy. Sem produção, `db:drop` e recriação limpa — decisão do usuário,
   2026-09-12. Migrations de remoção seriam documentação de um caminho que
   ninguém percorre de novo.
4. Desmontagem (§3).
5. Identidade (§5) — users na cidade, grant assinado, cookie host-only.
6. Ciclo de vida (§4) — `ProvisionCity`, `city:migrate:all`, guarda de 503.
7. Worker por cidade — gerente de supervisores, `queue.yml` enxuto, restart com
   backoff (mecanismo validado pelo spike 2).
8. Frontends — hosts, CORS dinâmico, `/authoring` no proxy, `ReportSnapshot#url`.
9. Chaves por cidade.

## Alternativas descartadas

- **Fechar só o furo do `/admin/api`** (guarda de `nil` + `RevokeMembership`
  destruindo sessions). Resolve o driver 2 e nenhum dos outros três. As duas
  correções continuam válidas se este trabalho for adiado.
- **Fila central na plataforma com `city_id` no job.** Um worker só, menor custo
  operacional — mas o payload de `ProcessInboundMessageJob` contém dado de
  cidadão, que passaria a residir em banco compartilhado. Indefensável em
  contrato.
- **Célula completa por cidade** (web + worker + db por cidade, proxy na frente).
  Zero código multi-banco e caminho literal para on-premise, mas contraria o
  requisito de um deploy: N processos web a ~512 MB cada e rollout sobre N
  células. O design atual deixa o caminho aberto — migrar para célula vira
  mudança de infraestrutura, não de código.
- **Identidade central com login único.** Menor mudança no módulo 06, mas o banco
  da cidade não conteria "apenas os dados da cidade" — quem acessa ficaria fora.
- **Cidade escolhida no login** (em vez de subdomínio). Não muda URLs, mas exige
  índice global de e-mails no catálogo, que por si só revela em qual prefeitura
  cada pessoa trabalha.
- **Fan-out cross-tenant em tempo real no Admin.** Mantém a UX atual, mas exige
  que o processo do Admin tenha credencial de leitura de todas as cidades —
  exatamente o privilégio que a separação elimina.

## Non-goals

- Migração para célula por cidade.
- Réplica de leitura, multi-região, sharding dentro de uma cidade.
- Visão agregada cross-tenant no Admin Console (removida por decisão).
- Migração de dados: não há produção. O trabalho é reescrita de schema e
  re-semeadura.

## Riscos abertos

1. **Gerente de supervisores** (spike 2 resolveu o mecanismo, não a operação). Um
   pai forkando um supervisor por cidade funciona, com falha contida e
   desligamento limpo. Falta especificar: observação do catálogo para cidades
   novas, política de restart com backoff (um filho com banco inacessível morre
   em 0,4 s e reiniciaria em loop apertado), e rollout de deploy sobre N
   supervisores.
2. **Pool sob carga real:** o spike mediu o mecanismo, não N cidades ativas
   simultâneas contra `max_connections`. Verificação de operação, não de spec.
3. **`contracts/events` é MAJOR:** `EVENTS.md` referencia `municipality_id` em 4
   pontos, incluindo a seção platform-scope. Pelo ADR-0015, remoção de campo exige
   expand/contract com entrada no CHANGELOG e issue de migração coordenada.
4. **Custódia de chave por cidade:** rotação e recuperação passam a ser por
   cidade. Precisa de procedimento operacional antes da etapa 9.

## Verification

**Infraestrutura de teste:** três bancos — `rota_saude_test_platform`,
`..._test_city_a`, `..._test_city_b`. Dois de cidade porque isolamento não se
prova com um.

**Três camadas:**

- **Unidade e request** — rodam contra uma cidade só, sem saber que existe
  multi-banco. É a maioria da suíte, e fica *mais simples* que hoje: some o
  `SET LOCAL` manual e o `use_transactional_tests = false` que o
  `spec/rls/tenant_isolation_spec.rb` precisa carregar.
- **Isolamento** (`spec/cities/city_isolation_spec.rb`) — herdeiro direto do spec
  de RLS, provando os mesmos invariantes por outro mecanismo: host A não lê banco
  B; sessão de A não autentica em B; job de A não escreve em B; grant de A não
  vale em B.
- **Arquitetura** — guarda de regressão no estilo do `spec/adr_pointers_spec.rb`
  existente. Falha se: `connects_to` for chamado fora do boot; sobrar
  `municipality_id` no schema; `cookies.signed` receber `domain:`; qualquer
  controller de `Admin::` voltar a abrir conexão privilegiada.

**Casos nomeados por seção:**

- host conhecido conecta no banco certo; desconhecido → 404; `suspended` → 403;
  schema atrasado → 503 só naquela cidade
- duas requisições concorrentes em cidades diferentes não trocam de conexão
- provisionamento idempotente; cidade em `provisioning` não é servida
- `city:migrate:all` falha alto quando uma cidade quebra
- offboarding de A não altera nada em B
- grant expirado e grant de outra cidade são rejeitados
- acesso de operador aparece nos dois logs de auditoria
- origem CORS de cidade inexistente é negada
- `ReportSnapshot#url` aponta para o host da cidade certa

**Verificação de operação (fora de spec):** comportamento de pool com N cidades
ativas; rollout de `city:migrate:all` com falha no meio.
