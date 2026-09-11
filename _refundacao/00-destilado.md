# Etapa 0 — Destilado dos ADRs (estado final, sem emenda)

- **Movimento:** 1 (refundação documental) · **Etapa:** 0 (extração)
- **Fonte:** `docs/adr/` (25 ADRs numerados + `README.md` + `nota-revisao-operacional.md`) — **read-only**
- **Saída:** este arquivo. Alimenta a Etapa 1 (mapa de consolidação 26→N).
- **Método:** para cada ADR, registra-se **o que ficou decidido em estado final**,
  já dobrando as emendas que o atingem. Onde o texto original ficou desatualizado
  pela emenda, o estado final é o da emenda. Onde há contradição ou ambiguidade
  real, **sinaliza-se — não se resolve** (princípio 5 do plano).

> **Como ler "estado final":** os ADRs 0001–0017 foram escritos numa fase
> *single-tenant*. Os ADRs 0019–0024 fizeram o *retrofit multi-tenant* (RLS) e
> emendaram vários dos primeiros. Os ADRs 0018 e 0025 mexeram na topologia física
> (monorepo → multi-repo). Por isso o estado final de muitos ADRs antigos **só se
> lê cruzando com a emenda**. Cada verbete abaixo já faz esse cruzamento.

---

## 1. Mapa de emendas (grafo dobrado)

Posterior **emenda/supersede** anterior:

| ADR emendado | É emendado por | Natureza da emenda |
|---|---|---|
| 0002 (estrutura de apps) | **0018** depois **0025** | layout físico: monorepo 4 apps → multi-repo por app. Papéis web/worker + imagem única **permanecem** |
| 0003 (DomainEvents.publish) | **0020**, **0023** | `publish` carimba `municipality_id` e faz o insert de auditoria no mesmo átomo (0020 §1.3 é a forma autoritativa); `municipality_id` vira nullable p/ platform-scope (0023) |
| 0005 (IdempotentConsumer) | **0020** | `TenantScopedJob`/`with_tenant`, `SET LOCAL`; `processed_events` ganha `municipality_id`; índice único permanece `(event_id, consumer)` |
| 0009 (auditoria/replay) | **0023** | `domain_events.municipality_id` nullable (eventos platform-scope sob bypass) |
| 0010 (webhook/ingestão) | **0014**, **0021** | HTTP de saída sai do lock (0014); ingestão resolve e carimba tenant via `municipality_channels` (0021) |
| 0011 (secrets/cripto) | **0021**, **0024** | `inbound_messages` ganha tenant+RLS (0021); custódia/rotação da chave AR Encryption definida (0024) |
| 0012 (conversa/consentimento) | **0021** | `Conversation.for` recebe `municipality_id`; índice ativo vira `(municipality_id, phone)` |
| 0013 (motor de protocolos) | **0016** | storage + façade `Protocols.current` + JSON Schema fecham de onde vem a definição |
| 0016 (storage/RBAC protocolo) | (fechado por **0017** scoring; roles por **0023**) | parcial: UI de autoria fica em aberto |
| 0018 (monorepo 4 apps) | **0025** | layout de diretório superseded (multi-repo); papéis/fronteiras dos apps permanecem |

Bases transversais que não emendam mas governam todos os que tocam o banco:
**0019 (RLS)** é base de 0020/0021/0023/0024; **0022 (identidade)** é base de 0023.

---

## 2. Destilado por ADR

### 0001 — Fila de background: Solid Queue + supervisor próprio
- **Status final:** Aceita, não emendada.
- **Decisão:** Active Job → **Solid Queue** sobre o mesmo Postgres; sem Redis/Sidekiq.
  Worker é processo separado via `bin/jobs` (mesma imagem, papel diferente). Filas
  nomeadas em `config/queue.yml` (0008).
- **Invariante:** sem dependência de fila fora do Postgres. Worker exige Postgres vivo.
- **Toca em multi-tenant:** a fila (`solid_queue_*`) guarda jobs de **todas** as
  cidades e **não recebe RLS** (0019) — o tenant é setado no *corpo* do job (0020).

### 0002 — Imagem Docker única, papéis web e worker (Kamal 2)
- **Status final:** Aceita; **layout superseded por 0018→0025**, **papéis permanecem**.
- **Decisão que sobrevive:** uma imagem Docker; papel selecionado por `cmd` do Kamal
  (`web` = `bin/rails server`, `worker` = `bin/jobs`); rollback atômico mesma SHA.
  Mono-processo descartado (isolamento de falha).
- **O que mudou:** os caminhos root-relative deste ADR foram re-parented para
  `apps/api/` (0018) e depois o backend virou **repositório próprio `api`** no
  multi-repo (0025). O literal do 0002 está desatualizado; a topologia válida é a do 0025.

### 0003 — Eventos de domínio: outbox-leve, auditoria, bindings declarativos
- **Status final:** Aceita; **emendada por 0020 e 0023**.
- **Decisão:** tabela `domain_events` (imutável, auditoria+replay), publisher
  `Events.publish(name, aggregate:, payload:)`, bindings declarativos em initializer.
  `published_at` separa "registrado" de "consumidores notificados".
- **Estado final (dobrado):**
  - `publish` **carimba `municipality_id`** (lido do `Current`, levanta `TenantMissing` se ausente) e **insere a linha de `domain_events` no mesmo `publish`** — esta é a forma autoritativa (0020 §1.3). *O trecho do 0003 que não inseria a linha está obsoleto.*
  - `municipality_id` é **nullable**: eventos platform-scope (login/MFA/provisionamento) gravam com tenant nulo via `Platform.audit` sob bypass (0023). `publish` em si segue **tenant-only**.
- **Invariante:** `domain_events` é imutável; único UPDATE legítimo é `published_at`.

### 0004 — `enqueue_after_transaction_commit = :always`
- **Status final:** Aceita, não emendada.
- **Decisão:** todo `perform_later` dentro de transação espera o COMMIT; rollback ⇒
  job nunca enfileira. Exceções só explícitas no call-site.
- **Invariante:** nenhum efeito externo dispara para uma transação que sofreu rollback.

### 0005 — Idempotência de consumidores via `processed_events`
- **Status final:** Aceita; **emendada por 0020**.
- **Decisão:** exactly-once por consumidor; tabela `processed_events` com único
  `(consumer, event_id)`; concern `IdempotentConsumer` (INSERT-ou-no-op, `consume` transacional).
- **Estado final (dobrado):** o concern agora inclui `TenantScopedJob` e abre
  `with_tenant(municipality_id)` (transação + `SET LOCAL app.municipality_id`) antes do
  `create!`+`handle`. `processed_events` ganha coluna `municipality_id` (escopo/auditoria),
  **mas o índice único permanece `(event_id, consumer)`** — `event_id` é UUID global.
- **Mudança de semântica sinalizada (§1.2):** envolver `create!`+`handle` numa transação
  torna efeitos **em banco** at-least-once correto (rollback re-executa); efeitos **HTTP**
  continuam sem rollback ⇒ caso HTTP **permanece aberto** (idempotency key no provedor).

### 0006 — Commands com `Result`
- **Status final:** Aceita, não emendada.
- **Decisão:** Command Object `.call(**) -> Result`; `Result.ok/fail`; comando não chama
  HTTP, roda em transação, não conhece controller. Commands iniciais: `CompleteTriagem`,
  `GiveConsent`, `RevokeConsent`. `IngestInboundMessage` **não** é command (webhook faz INSERT direto).
- **Nota de drift:** 0021 grafa `CompleteTriage`/`CompleteTriage.call` (sem "m") e
  `result.triage_args`; 0006 usa `CompleteTriagem`. **Inconsistência de nomenclatura
  PT/EN** (`Triagem` vs `Triage`) — sinalizada, não resolvida.

### 0007 — Relatórios congelados e dashboard reconstrutível
- **Status final:** Aceita, não emendada (mas dado vira tenant-scoped via 0019).
- **Decisão:** `ReportSnapshot` (imutável, `token`+`signature` HMAC, `expires_at`,
  congela `outcome.to_h`+`definition_id`); `DashboardMetric` (pré-agregado, reconstrutível
  por `rebuild_dashboard_metrics.rb`); `NotifyCitizenJob`/`AlertMunicipalityJob` consomem eventos.
  `recurring.yml` agenda rebuild/purges/reconcile.
- **Estado final (dobrado):** `report_snapshots`, `dashboard_metrics` recebem RLS (0019);
  rebuild cross-tenant roda sob `rota_admin` com `GROUP BY municipality_id`.
- **Invariante:** relatório individual imutável = prova auditável; métricas convergem para a verdade dos fatos via rebuild.
- **Aberto:** rotação de `REPORT_SIGNING_KEY` sem janela de break (não implementada).

### 0008 — Filas nomeadas e separação de SLA
- **Status final:** Aceita, não emendada.
- **Decisão:** filas `urgent`/`realtime`/`default`/`reports`/`housekeeping` por SLA;
  pool dedicado por fila; prioridade numérica como segundo eixo.
- **Invariante clínico:** relatório pesado **nunca** atrasa alerta urgente. `urgent` deve ficar em 0; >0 por 30s ⇒ investigar.
- **Aberto:** SLA do caminho de priorização clínica precisa de input do município; mecanismo de alerta urgente (escalonamento/plantão) é ADR à parte.

### 0009 — Replay de `domain_events`
- **Status final:** Aceita; **emendada por 0023** (`municipality_id` nullable).
- **Decisão:** `scripts/replay_domain_events.rb` (pendentes / seletivo por nome+since /
  `--dry-run` obrigatório); garante at-least-once + exactly-once-por-consumidor (via 0005);
  não garante ordem global nem re-rodar jobs já `discard_on`/excedidos.
- **Invariante:** `domain_events` é a fonte da verdade; o resto se reconstrói dela.

### 0010 — Webhook do WhatsApp: ack rápido, HMAC, dedup na borda
- **Status final:** Aceita; **emendada por 0014 e 0021**.
- **Decisão:** três camadas — (1) **HMAC antes de qualquer escrita** (`secure_compare`,
  401 na falha); (2) dedup na borda via `processed_events` com `consumer="whatsapp_webhook"`;
  (3) persistir cru (encriptado) + publicar + ack ~50ms. GET challenge de verificação.
- **Estado final (dobrado por 0021):** a borda agora é **HMAC → rota (qual cidade?) → persiste**.
  `Whatsapp::Ingest` resolve `phone_number_id → municipality_id` em `municipality_channels`
  (RLS-exempt), faz `SET LOCAL`, carimba `municipality_id` no `InboundMessage` e no enqueue.
  Índice único do inbound passa a ser `provider_message_id` (wamid). Número desconhecido ⇒
  ack 200 + parking em `unknown_channels` + alerta operador (novo modo de falha).
- **Invariantes:** atacante não injeta (HMAC trava antes do 1º INSERT); reentrega não polui banco; `raw` encriptado e fora do log.
- **Aberto:** rotação de `WHATSAPP_APP_SECRET` com janela dupla **não implementada** — rotação é break.

### 0011 — Gestão de secrets e criptografia em repouso
- **Status final:** Aceita; **emendada por 0021 e 0024**.
- **Decisão:** três camadas — secrets de runtime via Kamal (`deploy/<env>/secrets`, só
  referências a cofre); `master.key` nunca commitada (via `RAILS_MASTER_KEY`); **AR Encryption**
  em `InboundMessage#raw`, `Conversation`, `Consent`.
- **Estado final (dobrado):** `inbound_messages` ganha `municipality_id`+RLS (0021); a chave
  AR Encryption agora tem **custódia definida** (env por ambiente, rotação por lista de chaves,
  dono = operador — 0024). `municipality_channels.access_token` (token WhatsApp por cidade) também `encrypts`.
- **Invariante:** master key é o pivot — perda = dados criptografados inacessíveis; backup offline obrigatório.
- **Blast radius assumido (0024):** uma chave protege todas as cidades; endurecimento (chave por tenant/KMS) é pós-piloto.

### 0012 — Consentimento: entidade versionada e revogável
- **Status final:** Aceita; **emendada por 0021**.
- **Decisão:** `Conversation` (máquina de estados greeting→awaiting_consent→consented→revoked);
  `Consent` imutável append-only (linha por evento, `revoked_at` end-date, `policy_text_sha`,
  `evidence` encriptado); commands `GiveConsent`/`RevokeConsent`; service `Consents.interpret`/`current_version`.
- **Estado final (dobrado):** `Conversation.for(phone, municipality_id:)`; índice ativo
  `(municipality_id, phone)` — mesmo telefone pode ter conversa viva em duas cidades.
  `Consents.current_version` e `Protocols.current` passam a receber `municipality_id`.
  Termos de consentimento por cidade vêm de `consent_terms` (0024).
- **Invariantes clínicos/LGPD:** cidadão fora de `consented` **não** alimenta triagem;
  revogação aborta triagem em curso (`:aborted_by_revocation`, não conta como "completada");
  prova auditável (texto+versão+canal+timestamp).

### 0013 — Motor de protocolos como módulo puro
- **Status final:** Aceita; **complementada por 0016 (storage) e 0017 (scoring)**.
- **Decisão:** motor Ruby puro em `app/protocols/` (sem AR/AJ/controllers); `Protocol`,
  `Step`, `Validator`, `Definitions`; `evaluate(answers) -> Outcome`; erra cedo; serializável;
  JSON Schema único compartilhado em `packages/protocols/`.
- **Nota de topologia:** `packages/protocols` (contrato/schema) sobrevive ao multi-repo
  como `contracts/protocols/` (0025). O **dado de runtime** vive no banco (0016).

### 0014 — HTTP fora do lock
- **Status final:** Aceita, não emendada.
- **Decisão:** todo efeito externo (HTTP/e-mail/push) mora em job próprio, separado do
  consumidor transacional. `SendWhatsappJob` (fila `realtime`, recebe **dados literais**,
  idempotency key na request à Meta, `retry_on`, grava `OutboundMessage` em transação curta).
- **Estado final (dobrado por 0021):** `SendWhatsappJob` recebe `municipality_id`, abre
  `with_tenant`, e busca o canal em `municipality_channels` **com escopo manual** (tabela RLS-exempt).
- **Invariante:** consumidor não chama HTTP dentro do lock; falha de HTTP não rebobina decisão de domínio.

### 0015 — `Outcome`: tier, priority, trail
- **Status final:** Aceita — **"reconstruído por referência" (não havia ADR original)**. Sinalizado.
- **Decisão:** `Protocols::Outcome` Value Object imutável; `status` (:pending/:terminal),
  `tier` (String, não enum — deliberado), `priority`, `trail`, `awaiting`. Construtores
  `Outcome.pending/terminal`. `to_h` JSON-safe (congelado no snapshot).
- **Convenção/invariante:** nunca inspecionar `tier` sem checar `status` primeiro.
- **Aberto (relevante a 0012):** prever estado `:aborted` (consentimento revogado) — hoje a triagem usa `:aborted_by_revocation`, não há estado de `Outcome` correspondente. Sinalizado.

### 0016 — Storage de definições, façade `Protocols.current`, UI de autoria
- **Status final:** **Aceita (parcial — UI de autoria em aberto)**. Sinalizado.
- **Decisão:** tabela `protocol_definitions` (`name`, `version`, `municipality_id` null=global,
  `definition` jsonb, `status` draft/active/retired; único parcial `(name, municipality_id) WHERE active`);
  façade `Protocols.current/fetch` (único ponto que junta motor+banco, com cache e precedência
  município>global); JSON Schema único valida em `before_save`.
- **Estado final (dobrado):** roles `protocol_author`/`protocol_publisher` ganham dono e
  ciclo de vida em 0023; publicação exige **step-up MFA** (0022) e emite `protocol.published`.
  Scoring fechado por 0017. Protocolo semente no provisionamento (0024).
- **Aberto:** UI de autoria (formato de edição, fluxo de revisão, preview) — deliberadamente adiada; autoria por `scripts/import_protocol_definition.rb` por ora.

### 0017 — Scoring: weighted e decision_table
- **Status final:** Aceita, não emendada.
- **Decisão:** duas estratégias em `app/protocols/scoring/`, escolhidas por `scoring.type`;
  interface `call(trail) -> Outcome` + `to_h`; `Scoring.build(hash)`. Trail é a entrada canônica.
  Não se mistura as duas no mesmo fluxo (seria terceira estratégia).
- **Invariante:** decision_table = primeira regra que casa ganha; estável sob ordenação.

### 0018 — Topologia física do monorepo: 4 apps
- **Status final:** Aceita; **layout superseded por 0025**, **papéis/fronteiras permanecem**.
- **Decisão que sobrevive:** quatro apps com papéis e **fronteiras de segurança intransponíveis**:
  - `api` (Rails, web+worker);
  - `admin-console` — plataforma, cross-tenant, opera sob `rota_admin` (BYPASSRLS);
  - `dashboard` — cidade, tenant-scoped, sob `rota_app` (RLS);
  - `wpda` — paciente, público (token assinado), tenant-scoped, sob `rota_app`.
  Três frontends separados (não um console) por fronteira de privacidade/blast radius distinto.
- **O que mudou (0025):** o layout monorepo `apps/`+`packages/` foi invertido para multi-repo;
  `packages/ui` **abandonado**.

### 0019 — Isolamento multi-tenant: Row-Level Security
- **Status final:** Aceita. **Base transversal** (não emendada; governa 0020/0021/0023/0024).
- **Decisão:** isolamento imposto pelo **Postgres via RLS**, não só por escopo de app.
  Variável transação-local `app.municipality_id` via `SET LOCAL` (nunca `SET` puro);
  `FORCE ROW LEVEL SECURITY`; `WITH CHECK` impede gravar em outro tenant. **Dois papéis de banco:**
  `rota_app` (sujeito a RLS) e `rota_admin` (BYPASSRLS) — bypass por papel, não por variável.
  RLS em todas as tabelas com `municipality_id`+dado de tenant. **Exceções:** `municipality_channels`
  (estabelece o tenant) e `solid_queue_*` (quebraria o despacho).
- **Invariantes (críticos, falha fechada):** nenhuma query de domínio roda sem tenant setado
  (`current_setting` levanta ⇒ query falha barulhento, **nega não libera**); cross-tenant é
  **explícito** via `rota_admin`, sem caminho acidental.

### 0020 — Tenant em eventos e jobs (emenda a 0003 e 0005)
- **Status final:** Aceita.
- **Decisão:** `municipality_id` viaja como **dado literal** com evento/job (não se consulta —
  consulta já estaria sob RLS). `publish` carimba do `Current`; `TenantScopedJob#with_tenant`
  seta `Current`+`SET LOCAL` em transação; `IdempotentConsumer` reescrito sobre ele. Jobs de
  ingestão (`ProcessInboundMessageJob`, `SendWhatsappJob`) também escopam.
- **Invariante:** job que esqueça o tenant **falha fechado** (query levanta), não vaza.
- **Aberto:** §1.2 para efeito HTTP (idempotency key no provedor) — ADR à parte.

### 0021 — Roteamento de canal e ingestão multi-tenant (emenda a 0010, 0011, 0012)
- **Status final:** Aceita.
- **Decisão:** `municipality_channels` (`phone_number_id` único → `municipality_id`, `access_token`
  encriptado; RLS-exempt). Borda = HMAC → rota → persiste. `route` lê o canal sem tenant
  (necessário e seguro). `inbound_messages` ganha tenant+RLS. `Conversation.for` recebe município;
  índice `(municipality_id, phone)`. `SendWhatsappJob` escopa o canal **à mão** (custo da exceção RLS).
- **Topologia do Meta:** um App, um App Secret, vários `phone_number_id`; blast radius = suspensão do App afeta todas as cidades (aceito no piloto).
- **Aberto:** corrida de primeiro contato (índice+retry); seleção de canal no outbound com >1 número/cidade.

### 0022 — Autenticação, sessão e MFA
- **Status final:** Aceita. Fecha §2.1 da nota.
- **Decisão:** auth nativa do Rails 8 (`has_secure_password`, gerador `Session`, reset por token);
  seam para gov.br (OIDC) via `Authenticator`; **MFA (TOTP) em dois pontos** — operador no login
  (obrigatório), publisher step-up no ato de publicar. **Identidade é global / RLS-exempt**
  (`users`, `sessions`, `identities`): login roda antes de haver tenant.
- **Invariante:** identidade (global) e autorização (por-tenant) em camadas separadas; gestão de
  usuários sob `users` RLS-exempt vira responsabilidade da autorização (0023) — handoff à prova de vazamento.
- **Aberto:** integração gov.br; recovery de MFA; canal de auditoria platform-scope (formalizado no 0023).

### 0023 — Autorização: memberships, RBAC, tier de plataforma (emenda a 0009 e 0020)
- **Status final:** Aceita. Fecha §7.1 da nota + o canal de auditoria platform-scope do 0022.
- **Decisão:** autorização por `memberships (user, municipality, role)`; **operador de plataforma
  = membership com `municipality_id` nulo**; append-only (end-dating). Roles: `platform_operator`,
  `municipal_admin`, `protocol_author`, `protocol_publisher`, `viewer` (compõem). Enforcement por
  policy objects. **Princípio control plane vs data plane:** `municipality_channels`/`users`/`memberships`
  são control plane (RLS-exempt, protegidas pelo RBAC); domínio é data plane (RLS).
- **Emendas:** `domain_events.municipality_id` vira **nullable**; eventos platform-scope via
  `Platform.audit` (bypass, tenant nulo) — caminho irmão do `publish` tenant-only.
- **Four-eyes (§7.1):** colapso author+publisher permitido mas com step-up MFA + evento de auditoria;
  `platform_operator` é backstop de publisher.
- **Invariante LGPD:** staff não é titular de dado clínico; "quem publicou o quê" sobrevive à saída da pessoa (base de prestação de contas).

### 0024 — Provisionamento de município, custódia de secrets, config por cidade (emenda a 0011)
- **Status final:** Aceita. Fecha §5.4 da nota (custódia, no nível do piloto).
- **Decisão:** `ProvisionMunicipality` (único command que atravessa control plane sob bypass +
  data plane sob o tenant novo); `municipalities` (RLS-exempt, `ibge_code` como gancho SUS);
  **custódia da chave AR Encryption** (env por ambiente, rotação por lista, dono = operador);
  `consent_terms` por cidade (append-only); `alert_recipients` por cidade (config, não mecanismo).
- **Limite explícito:** dar destino ao alerta **não é** SLA, plantão, escalonamento nem dead-letter — tudo isso segue aberto.
- **Aberto:** chave por tenant (KMS); suspensão/desprovisionamento; mecanismo de alerta urgente; biblioteca de templates de protocolo.

### 0025 — Topologia multi-repo: um repositório por aplicação (emenda a 0018)
- **Status final:** Aceita (a topologia física vigente).
- **Decisão:** multi-repo sob org `rotasaude/`: `api`, `admin-console`, `dashboard`, `wpda`,
  `contracts` (events/protocols/types/design-tokens), `docs` (governança), `.github`. Board =
  Project v2 da org. **Princípio: compartilhar contrato, não código.** `packages/ui` **abandonado**;
  vira `contracts/design-tokens/` (cores/espaçamento/tipografia como dado). ADRs por **URL**, não path relativo.
- **Custos assumidos:** design global mais lento; contrato exige disciplina de versão; ADR e código separados; setup ×6 (mitigado por workflows reutilizáveis).
- **Aberto (importante):** política de versionamento de `contracts` (ADR próprio imediato);
  preservação de histórico do `api`; nome da org; custódia de secrets por repo (revisar 0024);
  release coordenado cross-repo.

---

## 3. Invariantes a preservar (entrada dura da Etapa 6)

Decisões de segurança/clínica/LGPD/isolamento que **têm de sobreviver ao corte**.
A Etapa 6 audita cada uma como bloqueio duro.

1. **Exactly-once por consumidor** (`processed_events`, único `(event_id, consumer)`) — 0005/0020.
2. **HMAC antes de qualquer INSERT** no webhook; `secure_compare` — 0010.
3. **Consentimento explícito antes de processar dado clínico**; revogação aborta triagem em curso — 0012.
4. **RLS falha fechada:** nenhuma query de domínio sem tenant; ausência de tenant **nega** — 0019.
5. **Cross-tenant só explícito** via `rota_admin`; sem caminho acidental — 0019.
6. **Criptografia em repouso** de PII/saúde (`raw`, `Conversation`, `Consent`, tokens) — 0011/0024.
7. **`domain_events` imutável** (só `published_at` muda); fonte da verdade para replay — 0003/0009.
8. **Relatório individual imutável** (prova auditável) — 0007.
9. **Control plane RLS-exempt protegido por RBAC** (não pelo banco) — 0023.
10. **MFA:** operador no login; publisher step-up no ato de publicar — 0022.
11. **Append-only** para `consents`, `memberships`, `users` (revogar = end-date, nunca DELETE) — 0012/0023.
12. **Publicação de protocolo = ato deliberado e rastreável** (step-up + `protocol.published`) — 0016/0022/0023.
13. **Isolamento de SLA clínico:** trabalho pesado nunca atrasa alerta urgente — 0008.
14. **HTTP fora do lock**; idempotency key na request à Meta — 0014.
15. **`solid_queue_*` e tabelas de roteamento/identidade NÃO recebem RLS** — exceção deliberada, não esquecimento — 0019/0021/0023.

---

## 4. Contradições, drift e ambiguidades — SINALIZADAS, não resolvidas

O agente reporta; o autor decide (princípio 5). Cada item abaixo precisa de
decisão na Etapa 1 ou de uma nota explícita no corpus v2.

- **C1 — Contagem 26→N.** O plano fala em "26 ADRs"; o corpus real tem **25 numerados**
  (0001–0025) + `README.md` + `nota-revisao-operacional.md`. Off-by-one a confirmar:
  o "26" incluía a nota? Um ADR planejado e não escrito? **Decisão do autor na Etapa 1.**
- **C2 — `packages/ui` vivo no README vs abandonado no 0025.** O `README.md` (Itens em
  aberto) ainda lista "Design system compartilhado (`packages/ui`) — ADR pendente" e
  "Contrato de tipos Ruby↔TS (`packages/types`)". O **0025 já decidiu**: `ui` dissolvido em
  `contracts/design-tokens/`, `types` → `contracts/types/`. **README desatualizado vs 0025.**
- **C3 — Insert de `domain_events` divergente entre 0003, 0009 e 0020.** O 0009 dizia que o
  insert acontece no `publish`; o código do 0003 nunca refletiu; o 0020 §1.3 é a forma
  autoritativa. O v2 deve consolidar **uma** descrição (a do 0020).
- **C4 — Nomenclatura `Triagem` vs `Triage` / `CompleteTriagem` vs `CompleteTriage`.** 0006/0007/0012
  usam PT (`Triagem`, `CompleteTriagem`); 0021 usa EN (`CompleteTriage`, `result.triage_args`,
  tabela `triages`); 0019 cita tabela `triages`. **Inconsistência de idioma no modelo** — o v2 precisa fixar um.
- **C5 — Estado `:aborted` do `Outcome` ausente.** 0012 aborta triagem com `:aborted_by_revocation`,
  mas 0015 não tem estado de `Outcome` correspondente (listado como "reavaliar se"). Gap entre os dois.
- **C6 — Mecanismo de alerta urgente fragmentado e parcialmente fantasma.** 0007 diz
  `AlertMunicipalityJob` chama a secretaria (e-mail); a nota (ponto cego) diz que ele "enfileira
  `DispatchMunicipalityAlertJob` que **não existe ainda**"; 0024 dá `alert_recipients` (config) mas
  **explicitamente não** cria mecanismo/SLA/plantão. Três descrições parciais do mesmo caminho.
- **C7 — Rotação de chaves sem janela dupla.** `WHATSAPP_APP_SECRET` (0010) e `REPORT_SIGNING_KEY`
  (0007) — rotação é break, não implementada. A chave AR Encryption (0024) tem rotação por lista,
  mas "não exercitada". Estado real ≠ estado desejado; o v2 deve declarar isso como dívida nomeada.
- **C8 — `0015` e `0016` têm status não-canônico** ("reconstruído por referência"; "Aceita parcial").
  Ao reescrever linear, o v2 precisa decidir se herda essa ressalva ou a promove a ADR pleno + item em aberto.
- **C9 — Custódia de secrets em multi-repo (0024 vs 0025).** 0024 ancorou a chave em
  `deploy/<env>/secrets` do monorepo; 0025 quebrou em 6 repos e listou "revisar 0024 quanto a onde
  a chave vive". Decisão pendente que **afeta um invariante** (custódia da chave-pivot).
- **C10 — `§` da nota referenciados mas a nota não tem numeração de seção.** Vários ADRs (0019–0024)
  citam "§1.2", "§2.1", "§5.4", "§7.1" da nota de revisão, mas a `nota-revisao-operacional.md` atual
  **não tem essas seções numeradas** (tem "Inventário", "Runbooks", "Pontos cegos"). As referências
  apontam para uma versão da nota mais detalhada que não está no corpus. **Rastreabilidade quebrada** — sinalizada.

---

## 5. Observações de agrupamento (NÃO é a decisão da Etapa 1)

Apenas para alimentar o mapa de consolidação — **o agrupamento 26→N é decisão do autor**.
Famílias temáticas que emergem naturalmente do corpus:

- **Plataforma de eventos/jobs:** 0001, 0003, 0004, 0005, 0008, 0009, 0014, 0020.
- **Borda WhatsApp / ingestão:** 0010, 0021 (+ 0014 no envio).
- **Consentimento & LGPD:** 0012, 0009 (auditoria), partes de 0011/0024.
- **Motor de protocolos:** 0013, 0015, 0016, 0017.
- **Multi-tenant / isolamento:** 0019, 0020, 0021, 0023, 0024.
- **Identidade & autorização:** 0022, 0023.
- **Relatórios/projeções:** 0007.
- **Topologia & deploy:** 0002, 0018, 0025.
- **Secrets & cripto:** 0011, 0024.
- **Commands:** 0006.

Note que vários ADRs cruzam famílias (0020 é eventos+tenant; 0024 é provisionamento+secrets+config).
A consolidação tem de escolher se agrupa por **camada técnica** ou por **capacidade de produto** — essa é a primeira pergunta do checkpoint da Etapa 1.

---

## 6. Veredito da Etapa 0

- **Cobertura:** 25/25 ADRs destilados + README + nota lidos integralmente.
- **Emendas dobradas:** 0002, 0003, 0005, 0009, 0010, 0011, 0012, 0013, 0016, 0018 — todas com estado final registrado.
- **Invariantes extraídos:** 15 (seção 3), prontos para a auditoria da Etapa 6.
- **Itens sinalizados (não resolvidos):** 10 contradições/drifts (C1–C10).
- **Nada inventado:** onde o destilado era ambíguo, foi reportado, não resolvido.

**Pronto para a Etapa 1** (mapa de consolidação 26→N) — que é checkpoint humano.
