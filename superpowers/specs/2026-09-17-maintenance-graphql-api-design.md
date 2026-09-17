# API GraphQL de manutenção — design

- **Status:** Proposto (aguardando revisão)
- **Data:** 2026-09-17
- **App:** `api`
- **Relacionado:** ADR 0001 (stack e papéis de processo), ADR 0004 (eventos de domínio),
  ADR 0010 (projeções e snapshots), ADR 0011 (identidade e MFA), ADR 0012 (RBAC e
  quatro-olhos), ADR 0013 (segredos e custódia de chaves), ADR 0014 (auditoria e LGPD),
  ADR 0015 (contratos); [mascaramento adiado](2026-09-17-mascaramento-dados-sensiveis-decisao.md)

## 1. Objetivo

Criar uma API GraphQL, servida pelo `api`, para que um **frontend separado** consulte e
altere configuração e dados operacionais de **todas as cidades** de um ambiente. O
acesso é por pessoas (mantenedores) e por automação (tokens de serviço).

É uma irmã da tela `/maintenance`, mas com outro modelo de segurança. A `/maintenance`
não autentica porque só existe em development. Esta API existe num ambiente
compartilhado e acessível pela internet (staging), por isso a autenticação é
obrigatória e carrega todo o risco.

## 2. Decisões

| # | Tema | Decisão |
|---|---|---|
| D1 | Ambientes | `development` (máquina de cada dev) e `staging` (compartilhado, ensaio de produção). **Produção fica fora** e será decisão própria no futuro |
| D2 | Escopo de dados | Configuração + dados operacionais. **Sem conteúdo de cidadão** enquanto o mascaramento não existir |
| D3 | Consumidores | Pessoas (frontend) e máquinas (automação) |
| D4 | Identidade | Conta própria `Maintainer`, separada de `Operator` |
| D5 | Autorização | Papel único e total (superusuário), inclusive para conceder e revogar acesso de outros mantenedores |
| D6 | Limite do superusuário | Pula a **autorização** (papéis e memberships), **não** as regras de domínio dos commands (ex.: quatro-olhos) |
| D7 | Tokens | Poderes totais **restringíveis** (cidades, só leitura), validade obrigatória, nunca gerenciam mantenedores nem tokens |
| D8 | Rede | Internet pública. Rate limit, bloqueio de conta e limites de query são obrigatórios |
| D9 | Transporte humano | Cookie no mesmo site (opção A). Sem bearer para pessoas |
| D10 | Onde mora | Dentro do `api`, no mesmo processo (abordagem 1) |
| D11 | Ambiente de staging | `RAILS_ENV=staging` explícito, herdando de produção |
| D12 | Isolamento | Total entre ambientes (§4) |

**Alternativas descartadas**

- **Processo separado na mesma imagem (papel `maintenance`):** isola a execução, mas
  cria uma topologia que só existiria em staging, o que tira fidelidade do ensaio. Fica
  como evolução natural caso produção entre.
- **Aplicação própria:** duplicaria models ou escreveria nos bancos por baixo dos
  commands, furando a invariante de que eventos de domínio são a fonte da verdade
  (ADR 0004/0014). Também exigiria uma segunda custódia da chave de cifra da
  plataforma (ADR 0013).
- **Bearer token para pessoas (opções B/C):** development roda na máquina do dev, com
  frontend e API no mesmo site, então o cookie cobre todos os casos. Adicionar bearer
  depois é acrescentar um ramo no autenticador, sem mudar o resto.

## 3. Hosts

| Ambiente | Frontend | API |
|---|---|---|
| development | `https://maintenance.development.rotasaude.com.br` | `https://maintenance-api.development.rotasaude.com.br` |
| staging | `https://maintenance.staging.rotasaude.com.br` | `https://maintenance-api.staging.rotasaude.com.br` |
| production | `https://maintenance.rotasaude.com.br` (não usado) | — |

- **Development** aponta os dois hosts para `127.0.0.1`, com TLS local (ex.: `mkcert`).
- **Rota restrita ao host `maintenance-api.*`**, como o console é restrito a `admin.*`.
  O slug `maintenance-api` é **reservado**, e nenhuma cidade pode usá-lo.
- **Os três ambientes estão no mesmo site (`rotasaude.com.br`)**, então `SameSite` não
  os isola entre si. Quem isola é o cookie sem `Domain=` e a checagem exata de `Origin`
  (§6).

## 4. Ambiente staging e isolamento

**`RAILS_ENV=staging`**

- `config/environments/staging.rb` faz `require_relative "production"` e sobrescreve
  só o necessário.
- **Predicado único para ambiente publicado:** `Rota.deployed?`, verdadeiro em
  `production` ou `staging`.
- **Os pontos que hoje usam `Rails.env.production?` com esse sentido passam a usar
  `Rota.deployed?`.** Sem isso, staging perderia segurança sem nenhum aviso:
  - cookie `Secure`: `concerns/authentication.rb`, `concerns/operator_authentication.rb`,
    `config/application.rb`;
  - `sslmode=require` na conexão da cidade: `services/city_database.rb`;
  - falha de boot por `CITY_DATABASE_HOST` / `PROVISIONER_DATABASE_URL` ausentes:
    `services/city_database.rb`;
  - porta padrão do banco da cidade: `services/city_database.rb`.
- `config/database.yml` e `config/credentials/staging.yml.enc` ganham entradas próprias.
- **Spec de guarda:** `Rails.env.production?` fora de `config/environments` quebra a
  suíte.

**Isolamento total entre ambientes**

- **Bancos e roles do Postgres:** separados, sem nenhuma credencial em comum.
- **Chaves:** master key, chave de plataforma e chaves por cidade próprias de cada
  ambiente. Nada de produção é reaproveitado.
- **Segredos externos:** app secret do WhatsApp, chave de assinatura de relatório e SMTP
  próprios de cada ambiente, ou sandbox.
- **Mantenedores, sessões, tokens e auditoria:** existem por ambiente, sem sincronização.
- **Dados:** staging **nunca** recebe dump de produção. Fazer isso reabre o mascaramento.
- **Rotas de development:** `/maintenance` e `/dev/impersonate*` continuam só em
  development, e staging não as recebe.

## 5. Trava de ativação

- **Função pura** `MaintenanceApi.enabled?(env:, flag:)`, verdadeira quando:
  - `env == "test"`; ou
  - `env ∈ {development, staging}` **e** `flag == "true"`.
- **A rota GraphQL e as rotas de sessão só são criadas** quando `enabled?` é verdadeiro.
  Fora disso não existem, e o Rails responde 404 no roteamento.
- **Trava de boot:** em `production` com `MAINTENANCE_API_ENABLED=true`, um initializer
  levanta erro e **o processo não sobe**.
- **Por que a rota existe em `test`:** cookie, CSRF, bloqueio de conta e escopo de token
  precisam de request spec. `test` não é ambiente publicado. A ausência em produção é
  provada pelos testes da função pura, cobrindo as combinações de env × flag.

## 6. Identidade e sessão humana

**Tabelas** (banco de plataforma)

- **`maintainers`:** `email_address` (único, sem diferenciar maiúsculas),
  `password_digest`, `otp_secret` (cifrado), `otp_enabled_at`, `deactivated_at`,
  `failed_attempts`, `locked_until`, `invited_by_id`.
- **`maintainer_sessions`:** `maintainer_id`, `mfa_verified_at`, `last_seen_at`,
  `totp_attempts`, `ip_address`, `user_agent`.
- **`maintainer_invitations`:** `maintainer_id`, `token_digest`, `expires_at` (24h),
  `used_at`.

**Login**

1. `POST /session` com e-mail e senha cria uma sessão **pendente**, válida por 10 min,
   e grava o cookie.
2. `POST /session/challenge` com o código TOTP torna a sessão **verificada**. Após 5
   códigos errados na sessão, ela é apagada.
3. Só sessão verificada autentica.
4. `DELETE /session` encerra a sessão.

**Validade, checada no servidor a cada requisição**

- **absoluta:** 8h a partir de `mfa_verified_at`;
- **por inatividade:** 30 min sem requisição (`last_seen_at`).

**Cookie `maintainer_session_id`:** assinado, `HttpOnly`, `SameSite=Strict`, **sem
`Domain=`**, com `Secure` quando `Rota.deployed?`.

**CSRF.** Toda requisição autenticada por cookie exige:

- `Origin` exatamente igual à origem do frontend do ambiente
  (`MAINTENANCE_FRONTEND_ORIGIN`);
- header `X-Rota-Maintenance: 1`, que força o preflight de CORS.

Faltando qualquer um, a resposta é **403**. O CORS permite só essa origem, com
credenciais.

**Força bruta**

- `rate_limit` por IP em `POST /session` e `POST /session/challenge`.
- Após 5 falhas seguidas na conta (senha ou TOTP), `locked_until = now + 15 min`, com
  evento de auditoria. O bloqueio vale mesmo com a senha certa.

**Ciclo de vida da conta**

- **Primeiro mantenedor de cada ambiente:** `rake maintainer:invite EMAIL=…`, rodado no
  servidor. É o único caminho fora da API.
- **Convite:** link de uso único válido por 24h. A pessoa define a senha, cadastra o
  TOTP (QR code) e confirma com um código. Sem TOTP confirmado, a conta não loga.
- **Recuperação:** outro mantenedor **reenvia o convite**, o que zera senha, TOTP e
  sessões. Não há recovery codes.
- **Desativação:** encerra na hora todas as sessões e **todos os tokens** da conta.
- **Travas:** ninguém desativa a si mesmo, e o último mantenedor ativo não pode ser
  desativado.
- **Gerenciar mantenedores** (convidar, reenviar convite, desativar) só é aceito por
  **sessão humana**.

**Código:** concern `MaintainerAuthentication` no mesmo formato de
`OperatorAuthentication`, reaproveitando a verificação TOTP existente. O console não é
refatorado.

## 7. Tokens de serviço

**Tabela `maintenance_tokens`** (banco de plataforma)

- **Dono e nome:** `maintainer_id` e `name`.
- **Segredo:** `token_digest` (HMAC-SHA256) e `token_prefix`. O token em claro nunca é
  gravado.
- **Escopo:** `access` (`read` | `read_write`) e `city_slugs` (array; vazio = todas).
- **Controle:** `expires_at` (**obrigatório**, no máximo 90 dias), `revoked_at`,
  `last_used_at`, `last_used_ip`.

**Formato:** `rsm_dev_<aleatório>` / `rsm_stg_<aleatório>`.

- O token em claro aparece **uma única vez**, na resposta da criação.
- **Prefixo de outro ambiente é recusado antes de consultar o banco.**
- Os repositórios são públicos, então o padrão do prefixo é cadastrado no secret
  scanning do GitHub.

**Criação e revogação**

- **Criação:** mutation `createMaintenanceToken`, só por **sessão humana** e com
  **código TOTP no próprio input** (step-up).
- **Revogação:** `revokeMaintenanceToken`, por qualquer mantenedor via sessão humana, com
  efeito imediato.
- **Sem refresh:** para renovar, cria-se um token novo e revoga-se o antigo.

**Autenticação por token, a cada requisição**

- **Transporte:** só `Authorization: Bearer …`. Requisição com cookie **e** bearer ao
  mesmo tempo recebe **401**.
- **Validade:** o token é inválido se estiver expirado, revogado, se o dono estiver
  desativado ou se for de outro ambiente.
- `rate_limit` por token.

**Escopo aplicado em ponto único**

- **`read`:** operações `mutation` são recusadas na análise da query, antes da execução.
- **`city_slugs`:** a checagem fica em `city(slug:)`, e a listagem `cities` é filtrada.
  Não existe outro caminho para dados de cidade (§8).
- **Nunca permitido a tokens:** gerenciar mantenedores, criar ou revogar tokens,
  consultar `auditEvents`.

## 8. Schema GraphQL

**Endpoint:** só `POST /graphql` no host `maintenance-api.*`, uma operação por
requisição. Não aceita GET nem lotes. Gem: `graphql-ruby`, schema `Maintenance::Schema`.

```graphql
type Query {
  me: Maintainer!
  maintainers: [Maintainer!]!                 # só sessão humana
  maintenanceTokens: [MaintenanceToken!]!
  auditEvents(filter: AuditFilter!): AuditEventConnection!   # só sessão humana
  cities(status: CityStatus): [CitySummary!]! # só banco de plataforma
  city(slug: String!): City                   # único caminho para dados da cidade
}

type City {
  slug: String!
  name: String!
  uf: String!
  status: CityStatus!
  schemaVersion: String
  schemaBehind: Boolean!
  channel: CityChannel            # sem access_token
  profile: CityProfile
  consentTerm: ConsentTerm
  protocols: [ProtocolDefinition!]!
  alertRecipients: [AlertRecipient!]!
  memberships: [Membership!]!
  operations: CityOperations!     # domain events, snapshots, métricas, jobs
  counts: CityCounts!             # conversas, mensagens, consentimentos: só números
}
```

**Acesso às cidades**

- **Só `city(slug:)` abre conexão** com o banco da cidade, via `CityConnection.with`.
  `cities` lê apenas o banco de plataforma.
- **Até 5 cidades por operação.** Acima disso, a query é recusada na análise.
- **Cidade inalcançável** gera erro no campo (`extensions.code: CITY_UNREACHABLE`), com a
  mensagem passando por `CitySchema.redact`. O resto da resposta sai normalmente.
- **Cidade arquivada** retorna só os campos de plataforma. Os campos de dentro do banco
  retornam `CITY_ARCHIVED`.

**Mutations**

- **Toda escrita passa por um command existente** (`InviteMember`, `RevokeMembership`,
  `DeactivateUser`, `CityLifecycle`, `municipality_channels`, protocolos etc.), nunca por
  `update!` no resolver. **Onde não houver command, ele é criado antes**, publicando
  seu evento de domínio.
- **Retorno:** o `Result` do command vira `{ ok: Boolean!, errors: [UserError!]! }`.
  Violação de regra de domínio é `UserError`, não erro de sistema.
- **O superusuário não contorna regras de domínio (D6).** Um command que recusaria um
  usuário com o papel certo também recusa o mantenedor. Exemplo: quatro-olhos na
  publicação de protocolo.

**Dados operacionais e suas invariantes**

- **`domain_events`:** só leitura, mais a ação de **republicar**. O evento nunca é
  alterado.
- **`ReportSnapshot`:** só leitura, porque é prova imutável.
- **`DashboardMetric`:** leitura e **reconstrução** da projeção.
- **Jobs do Solid Queue:** listar os que falharam, tentar de novo e descartar.

**Nunca entram no schema**

- **Segredos:** `database_url`, `encryption_key`, `access_token`, `password_digest`,
  `otp_secret`, `token_digest`, digests de convite.
- **Conteúdo de cidadão:** telefone, texto de mensagem, payload de webhook, evidência de
  consentimento, contexto e respostas de triagem. Aparecem no máximo como contagem ou
  metadado.

**Limites contra abuso**

- profundidade máxima 10;
- complexidade máxima 200;
- query de até 10 KB;
- tempo máximo de 10 s;
- introspecção **só em development**.

**Contrato:** o SDL é gerado a partir do schema e publicado no repo `contracts`, com
SemVer (ADR 0015). O frontend consome dali.

## 9. Auditoria

**Onde:** `platform_events`, via `Platform.audit`, no banco de plataforma do ambiente.

**Tentativa e resultado.** A escrita acontece no banco da cidade e a auditoria no banco
de plataforma, então não há transação única. Cada mutation segue:

1. **Antes**, grava `maintenance.<módulo>.<ação>` com `outcome: "attempted"`. **Se a
   gravação falhar, o command não roda.**
2. **Executa** o command. O evento de domínio da cidade leva `actor` e `correlation_id`.
3. **Depois**, grava o evento com `outcome` = `ok` | `rejected` | `error` e o mesmo
   `correlation_id`.

Se o processo morrer entre 2 e 3, fica uma tentativa sem resultado, que aparece como
**resultado desconhecido**. O `correlation_id` permite conferir no `domain_events` da
cidade se a alteração foi aplicada.

**Payload**

```json
{
  "occurred_at": "2026-09-17T14:03:11Z",
  "actor": { "maintainer_id": "…", "login": "…", "credential": "session" },
  "module": "city_profile",
  "operation": "updateCityProfile",
  "city_slug": "curitiba",
  "target": { "type": "CityProfile", "id": "…" },
  "changed_fields": ["display_name", "timezone"],
  "outcome": "ok",
  "correlation_id": "…",
  "request_id": "…",
  "ip": "…"
}
```

- Com token, `credential` é `{ "token_id": "…", "token_name": "…" }`.
- **`changed_fields` leva só nomes, nunca valores.**
- `occurred_at` é gravado em UTC e exibido em `America/Sao_Paulo`.

**Eventos de identidade e acesso:** login com sucesso e com falha, TOTP errado, bloqueio
de conta, fim de sessão, convite, reenvio de convite, desativação de mantenedor, e
criação, revogação e uso recusado de token (expirado, revogado, de outro ambiente, fora
do escopo).

**Leituras não são auditadas.** O schema não expõe segredo nem conteúdo de cidadão.

**Garantias**

- **Nomes `maintenance.*`** declarados em `R18_PLATFORM_EVENT_NAMES`, com o payload
  validado pelo guard existente.
- **Nenhuma mutation** altera ou apaga `platform_events`.
- **Trigger no Postgres** recusa `DELETE` e qualquer `UPDATE` que não seja de
  `published_at` em eventos `maintenance.%`.
- **Retenção:** 12 meses (ADR 0014).
- **Consulta:** `auditEvents(since, until, maintainer, city, module, outcome)`, só
  leitura e só por sessão humana.

## 10. Estratégia de teste

**Specs de arquitetura (guardas permanentes)**

- `Rails.env.production?` fora de `config/environments` quebra a suíte.
- `MaintenanceApi.enabled?`: todas as combinações de env × flag. Em produção nunca é
  verdadeiro.
- A trava de boot levanta erro em `production` com a flag ligada.
- **Schema:** lista explícita de tipos e campos (campo novo quebra até ser declarado) e
  recusa de nomes proibidos (`phone`, `body`, `raw`, `evidence`, `response`, `context`,
  `digest`, `secret`, `token`, `key`, `url`).
- Eventos `maintenance.*` declarados em `R18_PLATFORM_EVENT_NAMES`.
- Slug `maintenance-api` recusado na criação de cidade.

**Request specs: sessão e transporte**

- **Fluxo:** pendente → verificada. Sessão pendente não autentica e expira em 10 min.
- **Validade:** absoluta (8h) e por inatividade (30 min).
- **Bloqueio:** 5 falhas bloqueiam a conta por 15 min, mesmo com a senha certa depois.
  `rate_limit` responde 429.
- **Cookie:** `HttpOnly`, `SameSite=Strict`, sem `Domain`, e `Secure` quando
  `Rota.deployed?`.
- **CSRF:** sem `Origin` exato → 403; sem `X-Rota-Maintenance` → 403.
- **Credenciais ambíguas:** cookie e bearer juntos → 401.

**Request specs: tokens**

- Recusado quando expirado, revogado, de dono desativado ou de outro ambiente.
- Token `read` com mutation → recusado antes da execução.
- `city(slug:)` fora de `city_slugs` → recusado, e `cities` filtrado.
- Gerenciar mantenedores, gerenciar tokens e `auditEvents` via token → recusados.
- Criar token sem TOTP válido no input → recusado.
- Token em claro aparece só na resposta da criação.

**Mantenedores**

- Desativar a si mesmo e desativar o último ativo são recusados.
- A desativação encerra sessões e tokens.
- Reenviar convite zera senha, TOTP e sessões.

**Schema e cidades**

- Cidade inalcançável gera erro parcial com mensagem redigida, e as demais respondem.
- Mais de 5 cidades por operação, profundidade, complexidade e tamanho acima do limite →
  recusados.
- Um mantenedor publicando protocolo que ele mesmo escreveu é recusado pelo quatro-olhos
  (a regra de domínio vale para o superusuário).
- Resposta sem nenhum campo de segredo ou de cidadão.

**Auditoria**

- A tentativa é gravada antes do command, e falha na gravação impede o command.
- O resultado sai como `ok`, `rejected` ou `error`, com o mesmo `correlation_id`.
- `changed_fields` nunca contém valores.
- O trigger recusa `UPDATE` de payload e `DELETE`.

**Paridade de staging (job de CI)**

- Com `RAILS_ENV=staging` e variáveis de mentira: o boot passa, os cookies saem `Secure`
  e `sslmode=require` está ativo.
- Com `RAILS_ENV=production` e `MAINTENANCE_API_ENABLED=true`: o boot **falha**.

## 11. Fatias de entrega

Cada fatia é mergeável sozinha, com a suíte verde.

1. **Ambiente staging:** `staging.rb`, `Rota.deployed?`, troca dos pontos listados em
   §4, guard spec e job de paridade no CI. Independe do GraphQL e corrige um risco real.
2. **Fundação:** `maintainers`, sessões, convites e `rake maintainer:invite`,
   `MaintenanceApi.enabled?` com trava de boot, host e slug reservado, CORS/CSRF,
   GraphQL mínimo (`me`) e infraestrutura de auditoria com trigger.
3. **Tokens de serviço.**
4. **Leitura:** `cities`, `city(slug:)`, limites e spec de schema.
5. **Mutations por módulo:** uma fatia por módulo, criando o command quando faltar.
6. **Contrato:** SDL publicado em `contracts`.

## 12. Fora de escopo

- **Frontend** `maintenance.<env>.rotasaude.com.br` (spec própria).
- **Produção:** exige decisão própria, com dados reais, LGPD e revisão da exposição
  pública.
- **Mascaramento e exposição de conteúdo de cidadão:** adiado, ver
  [registro](2026-09-17-mascaramento-dados-sensiveis-decisao.md).
- **Bearer token para pessoas** e **recovery codes**.
- **Processo separado** para a API de manutenção.
