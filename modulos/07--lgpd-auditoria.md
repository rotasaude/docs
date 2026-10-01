# Módulo 07 — LGPD/Auditoria

- **Estado:** Em andamento
- **Tipo:** MVP

## Escopo

Trilha de auditoria via `domain_events`, retenção, purge, criptografia em
repouso, isolamento entre cidades (um banco por cidade), tratamento de
revogação de consentimento. Atravessa o sistema inteiro mas tem responsabilidades
próprias.

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0014 | `domain_events` append-only, escopo (todos os eventos), TTL 12 meses, purge |
| 0013 | Cifragem do `raw` (AR Encryption), TTL curto, purge backstop |
| 0020 | Um banco e um role por cidade, conexão escolhida pelo Host, `platform_events` no banco de plataforma, chave derivada por cidade (substitui o RLS do ADR 0003) |

## Superfícies

| Superfície | O que aparece |
|---|---|
| `api` | `DomainEvent` e `domain_events` no banco de cada cidade, purge recorrentes, `Platform.audit` gravando em `platform_events` no banco de plataforma |
| `admin` | Nenhuma visão entre cidades; entrada numa cidade por grant, auditada nos dois bancos; configuração de retenção (futuro) |
| `dashboard` | Eventos de domínio da cidade (referências), inspeção, busca por janela de tempo / nome |
| `wpda` | — |

## Funcionalidades planejadas

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-07.1 | Tabela `domain_events` append-only (trigger; DELETE só além de 12 meses) | api | 0014 |
| F-07.2 | Insert na mesma transação do `publish` | api | 0014, 0004 |
| F-07.3 | Purge de `domain_events` (12 meses) recurring | api | 0014 |
| F-07.4 | Cifragem do `raw` (AR Encryption) | api | 0013 |
| F-07.5 | Purge do `raw` após 90 dias, processado ou não (recurring) | api | 0014 |
| F-07.6 | Um banco e um role Postgres por cidade (`CONNECT` revogado de `PUBLIC` + `CONNECTION LIMIT`) | api | 0020 |
| F-07.7 | Roles Postgres: `rota_city_<slug>` por cidade + `rota_platform` + `rota_provisioner` (só no worker) | api (config) | 0020 |
| F-07.8 | Cidade resolvida pelo Host antes da autenticação (`CityCatalog` + `CityConnection.with`) | api | 0020 |
| F-07.9 | `CityScopedJob` e `EachCityJob`: jobs na fila do banco da cidade | api | 0020, 0006 |
| F-07.10 | `DomainEvents.publish` grava no `domain_events` do banco da cidade (sem `municipality_id`) | api | 0004, 0020 |
| F-07.11 | `platform_events` no banco de plataforma (sem dado pessoal; referencia `city_id`; imutável por trigger) | api | 0014, 0020 |
| F-07.12 | `Platform.audit` para eventos de identidade | api | 0014 |
| F-07.13 | Painel de eventos no dashboard da cidade | dashboard | brief |
| F-07.14 | Console de plataforma sem visão entre cidades; entrada por grant auditada nos dois bancos | api, admin | 0014, 0020 |
| F-07.15 | Tratamento de revogação de consentimento (assinante de `consent.revoked`; anonimiza a triagem sem atendimento, concluída ou não) | api | 0008, 0026 |
| F-07.16 | Exclusão do cadastro do cidadão no posto (Art. 18): verificador pede, admin confirma com step-up; casca no lugar do CPF e do celular; quem foi atendido fica retido | api, dashboard | 0026 |

## Dependências

- Todos os outros módulos — o banco por cidade isola o dado de todos;
  auditoria cobre eventos de todos.

## Riscos herdados

- **(ADR 0014) (LGPD vs. imutabilidade):** resolvido pelo ADR 0026 — a
  exclusão deixa uma casca no lugar do CPF e do celular, e o que só aceita
  acréscimo (`consents`, `domain_events`, `citizen_verifications`) fica, sem
  ninguém identificável por trás.
- **(ADR 0014):** `consent.revoked` tem assinantes (F-07.15): anonimizam a
  triagem sem atendimento (interrompida ou concluída) e contam a métrica. A
  triagem que virou atendimento fica retida por tutela da saúde (ADR 0026).
- **(ADR 0014) (Art. 20):** revisão humana de decisão automatizada não
  tem dono. Trail (módulo 03) dá insumo mas não fluxo.
- Todas as recurring tasks concentram fragilidade no
  worker. Worker parado = LGPD violada por purge não executado. Com um
  supervisor por cidade, uma cidade atrasada ou fora do ar para só as
  tarefas dela.
- **(ADR 0020):** a proteção entre cidades está inteira no resolver de
  conexão. Errar a conexão entrega uma cidade inteira; o resolver precisa do
  mesmo peso de teste que o RLS tinha.

## Riscos que continuam

- **Exclusão (Art. 18, VI):** decidida no ADR 0026 e entregue (F-07.16).
  Ficam de fora: purga de `citizen_sessions`, `otp_challenges` e
  `outbound_messages`; texto livre congelado em atendimento, agendamento e
  pedido; revogação pelo WhatsApp depois de a conversa terminar. Os argumentos
  de job em `solid_queue_jobs` (ex.: `SendWhatsappJob(to:)`) guardam o
  telefone até a limpeza da fila.
- **Painéis ao vivo e triagem anonimizada:** a concluída anonimizada continua
  contando como concluída nos painéis do módulo 05 ("sem tier"); o Analytics
  já a trata como revogada.
- **Admin sem autenticador:** a tela de pedidos de exclusão não leva o
  `municipal_admin` sem TOTP até a Segurança; ele precisa cadastrar antes.
- **Rollout do ADR 0026:** web e worker trocam juntos (worker antigo não
  monta o relatório a partir do evento só com `triage_id`); `city:migrate:all`
  antes do tráfego (o check-in lê `anonymized_at`); publicar em cada cidade o
  termo de consentimento novo (`city:consent_term:publish`).
- **Revisão de decisão automatizada (Art. 20) sem dono:** a trilha existe,
  mas o fluxo de revisão, não.
- **Dado pessoal em `domain_events`:** `user.invited` grava o e-mail do servidor
  convidado no payload, por 12 meses. Não há guarda de payload do lado da cidade,
  como a de `platform_events`. O painel só mostra referências, mas o dado fica
  na trilha.
- **Telefone das mensagens sem prazo:** `inbound_messages.from` e
  `outbound_messages.to` agora são cifrados, mas não expiram.
- **Rollout da cifra dos telefones:** depois do deploy, e com web e worker
  antigos já drenados, rode `city:encrypt_message_phones:all`. Até lá, linha
  antiga em claro levanta ao ser lida (produção não aceita dado sem cifra), e
  `ReencryptionJob`/`city:rotate_key` falham na cidade. Restaurar um dump
  anterior à cifra traz o texto claro de volta; rode a task de novo depois do
  `city:restore`.
- **Retenção fixa no banco:** o TTL de 12 meses de `domain_events` e
  `platform_events` está no trigger. Encurtar exige migração.
- **Worker parado = purga parada:** com um supervisor por cidade, só a cidade
  afetada atrasa.

## Critério de fechamento do módulo

- ✓ F-07.1 a F-07.15 verificadas. F-07.16 entregue em 2026-10-01 (ADR 0026), aguardando verificação.
- ✓ Suíte de invariante, no `api`:
  - cidade A não vê dado de B:
    - por conexão: `cities/city_isolation_spec.rb`;
    - pelo Host (host desconhecido responde 404 antes de autenticar):
      `cities/city_isolation_requests_spec.rb`;
    - por role de banco: `services/city_database_spec.rb`;
  - sessão de A não vale em B: `requests/city_session_isolation_spec.rb`,
    `requests/operator_city_session_spec.rb`;
  - job de A não grava em B: `models/city_connection_queue_spec.rb`,
    `jobs/concerns/city_scoped_job_spec.rb`,
    `architecture/current_city_assignment_spec.rb`;
  - grant de A não vale em B: `requests/session_grant_spec.rb`,
    `services/city_grants_spec.rb`;
  - `domain_events` só recebe acréscimos: `models/domain_event_append_only_spec.rb`;
  - `platform_events` só recebe acréscimos: `events/maintenance_audit_spec.rb`;
  - evento some com o rollback da transação: `events/domain_events_spec.rb`;
  - purga respeita o TTL: `jobs/purge_domain_events_job_spec.rb`,
    `jobs/purge_inbound_raw_job_spec.rb`;
  - `raw` e telefones só se decifram com a chave da própria cidade:
    `models/city_connection_encryption_spec.rb`,
    `models/message_phone_encryption_spec.rb`;
  - credencial do provisioner só no worker:
    `config/provisioner_credential_spec.rb`.
- ✓ Requisição LGPD em `operacao/atender-requisicao-lgpd.md`.

## Histórico

- 2026-09-27: módulo fechado, com 15/15 `Verified`. A verificação por F-ID
  achou lacunas reais, consertadas com TDD:
  - F-07.1: `domain_events` não era só acréscimo. Agora um trigger só aceita
    marcar `published_at` uma vez e recusa DELETE dentro de 12 meses. O modelo
    trata o evento gravado como somente leitura. A purga usa o mesmo corte em
    SQL e recusa janela menor que 12 meses.
  - F-07.11: a imutabilidade de `platform_events` cobria só `maintenance.*`.
    Agora cobre toda a trilha: auditoria de manutenção nunca sai; o resto só
    sai além de 12 meses.
  - F-07.13: a tela ganhou janela de tempo própria e busca por nome, e o
    endpoint ganhou specs e escape de curinga.
  - O telefone das mensagens (`inbound_messages.from`,
    `outbound_messages.to`) passou a ser cifrado com a chave da cidade.
  - Specs novas:
    - F-07.2: rollback do evento junto com a transação.
    - F-07.5: corte exato, idempotência, mensagem nunca processada.
    - F-07.7: credencial do provisioner só no worker; atributos dos roles.
    - Isolamento pelo Host e guarda estática de `Current.city`.
  - Migrações: de cidade `20260927300001`; de plataforma `20260927300002`.
  - ADR 0014 ganhou nota apontando o mecanismo do ADR 0020.
  - Commits: api `433f7c0` (2167 exemplos, 0 falhas), dashboard
    `a4316d2` (339 testes).

- 2026-10-01: ADR 0026 (resolve rotasaude/api#30 e rotasaude/docs#2).
  - F-07.15: a revogação passa a anonimizar também a triagem concluída sem
    atendimento (`triages.anonymized_at`); a que virou atendimento fica retida;
    o check-in recusa triagem anonimizada ou de conversa revogada, com trava na
    linha.
  - Trilha só com referência: `triage.completed`/`urgent` levam só
    `triage_id`; `consent.revoked` leva a origem (`web`, `whatsapp`, `erasure`).
    Os assinantes pulam a triagem anonimizada.
  - F-07.16 (novo): exclusão do cadastro no posto, por duas pessoas, com casca
    no lugar do CPF e do celular; `citizen_erasure_requests` só aceita
    acréscimos.
  - Migrações de cidade `20261001000001` e `20261001000002`.
  - Commits: api `eafe586` (3059 exemplos, 0 falhas), dashboard `723ccf1`
    (792 testes).
