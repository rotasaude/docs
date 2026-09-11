# Reconciliação dos ponteiros de ADR no código (v1 → v2)

Registro de decisão, uma linha por ocorrência. A reescrita dos comentários em
`rotasaude/api` foi **derivada desta tabela** — se um ponteiro do código não
bate com a linha aqui, a tabela é a fonte de verdade e o código está errado.

Método e mapa v1→v2: [`DE-PARA.md`](DE-PARA.md). Texto v1: [`_v1/adr/`](../../_v1/README.md).

Legenda de `Decisão`: `v1→v2` (era v1, reescrito) · `já v2` (não mexido) ·
`DÚVIDA` (não mexido, pendente de revisão humana).

## app/protocols, app/events, app/mailers, app/auth, app/policies

| Ocorrência | Antes | Depois | Decisão | Justificativa |
|---|---|---|---|---|
| `app/auth/authenticator.rb:1` | ADR-0022 | ADR-0011 | v1→v2 | v1 0022 = autenticação/sessão/MFA; o arquivo é o ponto único de autenticação |
| `app/auth/authenticator/gov_br.rb:1` | ADR-0022 | ADR-0011 | v1→v2 | idem — strategy gov.br OIDC |
| `app/auth/authenticator/gov_br.rb:26` | ADR-0022 | ADR-0011 | v1→v2 | mapeamento assurance→role é parte do ato de autenticar |
| `app/events/domain_events.rb:1` | ADR-0003 + ADR-0020 | ADR-0004 | v1→v2 | v1 0003 (eventos de domínio) e v1 0020 (publish carimbado com tenant) colapsam ambos no v2 0004 — deduplicado |
| `app/events/domain_events.rb:3` | ADR-0004 | ADR-0004 | v1→v2 | v1 0004 (`enqueue_after_transaction_commit`) → v2 0004; número coincide, sentido confirmado |
| `app/events/events.rb:2` | ADR-0020 | ADR-0004 | v1→v2 | alias `Events = DomainEvents`; v1 0020 aqui é a face publish → v2 0004 |
| `app/events/platform.rb:1` | ADR-0023 | ADR-0012 | v1→v2 | v1 0023 = RBAC/tier de plataforma; o canal é platform-scope |
| `app/events/platform.rb:3` | ADR-0020 | ADR-0004 | v1→v2 | "publish exige tenant" é a regra do publish → v2 0004, não o mecanismo (0003) |
| `app/mailers/alert_mailer.rb:1` | ADR-0007 | ADR-0010 | v1→v2 | v1 0007 §3 ("Notificação ao cidadão e alerta à secretaria") nomeia o `AlertMunicipalityJob`; DE-PARA manda 0007→0010. **Ver nota de lossiness abaixo.** |
| `app/mailers/password_mailer.rb:1` | ADR-0022 | ADR-0011 | v1→v2 | reset de senha é sessão/credencial |
| `app/policies/protocol_policy.rb:10` | ADR-0009 | ADR-0009 | já v2 | "vigência por cidade" é o motor/storage de protocolos = v2 0009; v1 0009 era replay de eventos, não casa |
| `app/protocols/condition.rb:1` | ADR-0013 | ADR-0009 | v1→v2 | v1 0013 = "Motor de protocolos como módulo puro" |
| `app/protocols/definitions.rb:1` | ADR-0013 | ADR-0009 | v1→v2 | idem |
| `app/protocols/definitions.rb:2` | ADR-0016 | ADR-0009 | v1→v2 | v1 0016 = storage + façade `Protocols.current`, consolidado no v2 0009 |
| `app/protocols/outcome.rb:1` | ADR-0015 | ADR-0009 | v1→v2 | v1 0015 = "`Outcome`: tier, priority, trail"; o arquivo **é** o Outcome |
| `app/protocols/priority_rules.rb:1` | ADR-0017 | ADR-0009 | v1→v2 | v1 0017 = estratégias de scoring |
| `app/protocols/protocol.rb:1` | ADR-0013 | ADR-0009 | v1→v2 | agregado raiz do motor |
| `app/protocols/protocol.rb:31` | ADR-0015 | ADR-0009 | v1→v2 | devolve o Outcome |
| `app/protocols/protocol.rb:32` | ADR-0017 | ADR-0009 | v1→v2 | scoring decide tier/priority |
| `app/protocols/scoring.rb:1` | ADR-0017 | ADR-0009 | v1→v2 | fábrica de scoring |
| `app/protocols/scoring/decision_table.rb:1` | ADR-0017 | ADR-0009 | v1→v2 | estratégia decision_table |
| `app/protocols/scoring/weighted.rb:1` | ADR-0017 | ADR-0009 | v1→v2 | estratégia weighted |
| `app/protocols/step.rb:1` | ADR-0013 | ADR-0009 | v1→v2 | passo do motor |
| `app/protocols/urgency.rb:2` | ADR-0006 | ADR-0006 | já v2 | escrito depois da refundação; v2 0006 = filas e prioridade clínica |
| `app/protocols/urgency.rb:3` | ADR-0009 | ADR-0009 | já v2 | idem; v2 0009 = motor de protocolos |
| `app/protocols/validation/condition.rb:4` | ADR-0013/0017 | ADR-0009 | v1→v2 | os dois v1 colapsam no mesmo v2 — deduplicado |
| `app/protocols/validation/priority_when.rb:2` | ADR-0017 | ADR-0009 | v1→v2 | validação da camada de prioridade do motor |
| `app/protocols/validation/schema.rb:2` | ADR-0016 | ADR-0009 | v1→v2 | schema da definição de protocolo |
| `app/protocols/validator.rb:1` | ADR-0013 e ADR-0016 | ADR-0009 | v1→v2 | ambos colapsam — deduplicado |
| `app/protocols/validator.rb:31` | ADR-0016 | ADR-0009 | v1→v2 | idem |

### Nota de lossiness — v1 0007 §3

O v1 0007 ("Relatórios congelados e dashboard reconstrutível") tinha uma seção §3
sobre **notificação ao cidadão e alerta à secretaria**. O DE-PARA manda o ADR
inteiro para o v2 0010 (CQRS: projections & snapshots), que não fala de alerta.
Os ponteiros de `AlertMailer` e `AlertMunicipalityJob` ficaram formalmente
corretos mas **magros**: quem seguir cai num ADR de projeções.

Não inventamos mapeamento para consertar isso — seria decidir arquitetura numa
tabela de reconciliação. Fica registrado como candidato à lista de itens em
aberto do corpus: o alerta à secretaria não tem ADR próprio no v2.

## app/commands, app/services

| Ocorrência | Antes | Depois | Decisão | Justificativa |
|---|---|---|---|---|
| `app/commands/complete_triage.rb:1` | ADR-0006 e ADR-0017 | ADR-0004 e ADR-0009 | v1→v2 | v1 0006 = commands com `Result`; v1 0017 = scoring |
| `app/commands/conversation_advance.rb:2` | ADR-0012, 0013, 0010 | ADR-0008, 0009, 0007 | v1→v2 | consent → 0008; motor → 0009; ingestão WhatsApp → 0007 |
| `app/commands/conversation_advance.rb:9` | ADR-0021 emenda 0012 | ADR-0008 | v1→v2 | estados da conversa = v2 0008; o v2 não tem cadeia de emendas, então a referência à emenda sai |
| `app/commands/conversation_advance.rb:13` | ADR-0013 | ADR-0009 | v1→v2 | prompt vem do step do motor |
| `app/commands/give_consent.rb:1` | ADR-0006 e ADR-0012 | ADR-0004 e ADR-0008 | v1→v2 | command + consentimento |
| `app/commands/municipality_channels/rotate_token.rb:4` | ADR-0011/0023 | ADR-0012/0013 | v1→v2 | **split de v1 0011**: "SEM o valor do token" é custódia de secret → 0013 (não 0014/retenção). v1 0023 = RBAC → 0012 |
| `app/commands/protocols/activate.rb:2` | ADR-0009 | ADR-0009 | já v2 | "lifecycle de dois eixos" é o motor (v2 0009); v1 0009 era replay de eventos |
| `app/commands/protocols/publish.rb:2` | ADR-0009 | ADR-0009 | já v2 | idem |
| `app/commands/protocols/publish.rb:5` | ADR-0003 | ADR-0003 | já v2 | "within_tenant do request" é RLS (v2 0003); v1 0003 era pub/sub de eventos |
| `app/commands/protocols/publish.rb:6` | ADR-0011 | ADR-0011 | já v2 | step-up MFA = v2 0011; v1 0011 era cripto do payload bruto |
| `app/commands/protocols/retire.rb:2` | ADR-0009 | ADR-0009 | já v2 | idem activate |
| `app/commands/provision_municipality.rb:2` | ADR-0024 | ADR-0013 | v1→v2 | v1 0024 = provisionamento de município |
| `app/commands/provision_municipality.rb:26` | ADR-0020 | ADR-0003 | v1→v2 | **split de v1 0020**: `Current.municipality_id` + `SET LOCAL` é o mecanismo → 0003 |
| `app/commands/result.rb:1` | ADR-0006 | ADR-0004 | v1→v2 | o `Result` é da camada de commands |
| `app/commands/revoke_consent.rb:1` | ADR-0006 e ADR-0012 | ADR-0004 e ADR-0008 | v1→v2 | command + consentimento |
| `app/commands/seed_protocol.rb:2` | ADR-0024 | ADR-0013 | v1→v2 | seed de protocolo faz parte do provisionamento |
| `app/services/consents.rb:2` | ADR-0012 | ADR-0008 | v1→v2 | consentimento versionado |
| `app/services/whatsapp/ingest.rb:1` | ADR-0021 | ADR-0007 | v1→v2 | v1 0021 = roteamento de canal e ingestão multi-tenant |
| `app/services/whatsapp/ingest.rb:3` | ADR-0010 | ADR-0007 | v1→v2 | v1 0010 = webhook WhatsApp, HMAC na borda |
| `app/services/whatsapp/ingest/parser.rb:1` | ADR-0010 | ADR-0007 | v1→v2 | idem |
| `app/services/whatsapp/outbound.rb:1` | ADR-0021/0024 | ADR-0007/0013 | v1→v2 | canal → 0007; config por cidade → 0013 |

## app/jobs

| Ocorrência | Antes | Depois | Decisão | Justificativa |
|---|---|---|---|---|
| `app/jobs/alert_municipality_job.rb:1` | ADR-0007 e ADR-0008 | ADR-0010 e ADR-0006 | v1→v2 | v1 0007 §3 (alerta à secretaria) e v1 0008 (filas/SLA). Ver nota de lossiness |
| `app/jobs/alert_municipality_job.rb:10` | ADR-0014 | ADR-0005 | v1→v2 | v1 0014 = HTTP fora do lock |
| `app/jobs/anonymize_revoked_triage_job.rb:2` | ADR-0005/0020 | ADR-0005/0003 | v1→v2 | consumidor idempotente + **split de v1 0020** na face mecanismo de tenant |
| `app/jobs/concerns/admin_role_job.rb:1` | ADR-0019 | ADR-0003 | v1→v2 | v1 0019 = RLS |
| `app/jobs/concerns/admin_role_job.rb:3` | ADR-0020 | ADR-0003 | v1→v2 | **split**: `TenantScopedJob` é o mecanismo |
| `app/jobs/concerns/idempotent_consumer.rb:1` | ADR-0005 + emenda ADR-0020 | ADR-0005 | v1→v2 | **split**: v1 0020 §1.2 é idempotência → colapsa no mesmo 0005, deduplicado |
| `app/jobs/concerns/idempotent_consumer.rb:3` | ADR-0014 | ADR-0005 | v1→v2 | efeito colateral fora do lock é parte do v2 0005 |
| `app/jobs/concerns/tenant_scoped_job.rb:2` | ADR-0020 | ADR-0003 | v1→v2 | **split**: mecanismo de tenant |
| `app/jobs/dispatch_municipality_alert_job.rb:1` | ADR-0014 | ADR-0005 | v1→v2 | HTTP/SMTP fora do lock |
| `app/jobs/generate_report_job.rb:1` | ADR-0007 | ADR-0010 | v1→v2 | cria o `ReportSnapshot` — casa exatamente com o v2 0010 |
| `app/jobs/notify_citizen_job.rb:1` | ADR-0007 e ADR-0014 | ADR-0010 e ADR-0005 | v1→v2 | link do snapshot + envio fora do lock |
| `app/jobs/process_inbound_message_job.rb:1` | ADR-0010/0014/0020/0021 | ADR-0007/0005/0003 | v1→v2 | 0010 e 0021 colapsam ambos em 0007 — deduplicado |
| `app/jobs/purge_domain_events_job.rb:2` | ADR-0005/0014 | ADR-0005/0014 | já v2 | escrito depois da refundação; v2 0014 = retenção/LGPD descreve o job, o v1 0014 (HTTP fora do lock) não |
| `app/jobs/purge_expired_reports_job.rb:1` | ADR-0007 | ADR-0010 | v1→v2 | apaga `ReportSnapshot` |
| `app/jobs/purge_inbound_raw_job.rb:2` | ADR-0011 | ADR-0014 | v1→v2 | **split de v1 0011**: o comentário diz "(retenção)" explicitamente → 0014, não 0013 |
| `app/jobs/purge_processed_events_job.rb:1` | ADR-0005 | ADR-0005 | v1→v2 | `processed_events` é do v1 0005; número coincide |
| `app/jobs/purge_processed_events_job.rb:2` | ADR-0009 | ADR-0014 | v1→v2 | v1 0009 = replay de `domain_events`, consolidado no v2 0014 |
| `app/jobs/rebuild_dashboard_metrics_job.rb:1` | ADR-0007 | ADR-0010 | v1→v2 | `DashboardMetric` reconstrutível |
| `app/jobs/reconcile_consents_job.rb:3` | ADR-0012 | ADR-0008 | v1→v2 | consentimento |
| `app/jobs/reencryption_job.rb:2` | ADR-0024 | ADR-0013 | v1→v2 | rotação de chave = custódia de secrets |
| `app/jobs/reencryption_job.rb:8` | ADR-0019 | ADR-0003 | v1→v2 | BYPASSRLS |
| `app/jobs/resend_pending_alerts_job.rb:3` | ADR-0005 | ADR-0005 | v1→v2 | idempotência por consumidor; número coincide |
| `app/jobs/send_whatsapp_job.rb:1` | ADR-0014/0021 | ADR-0005/0007 | v1→v2 | fora-de-banda + canal WhatsApp |
| `app/jobs/sweep_abandoned_conversations_job.rb:1` | ADR-0019 | ADR-0003 | v1→v2 | varredura cross-tenant sob RLS |
| `app/jobs/update_dashboard_job.rb:1` | ADR-0007 + 0020 | ADR-0010 + 0003 | v1→v2 | projeção do dashboard + **split** na face mecanismo |

## app/models, app/queries

| Ocorrência | Antes | Depois | Decisão | Justificativa |
|---|---|---|---|---|
| `app/models/alert_recipient.rb:1` | ADR-0024 | ADR-0013 | v1→v2 | destinatário por município é config por cidade |
| `app/models/consent.rb:1` | ADR-0012 | ADR-0008 | v1→v2 | consentimento versionado |
| `app/models/consent.rb:6` | ADR-0011 | ADR-0013 | v1→v2 | **split de v1 0011**: `encrypts :evidence` é criptografia → 0013 |
| `app/models/consent_term.rb:1` | ADR-0024 | ADR-0013 | v1→v2 | termo por município é config por cidade |
| `app/models/conversation.rb:1` | ADR-0012 + emenda ADR-0021 | ADR-0008 e ADR-0007 | v1→v2 | consent + roteamento de canal; a palavra "emenda" sai (o v2 não tem cadeia) |
| `app/models/conversation.rb:30` | ADR-0021 | ADR-0007 | v1→v2 | `Conversation.for(phone, municipality_id:)` é roteamento de canal |
| `app/models/current.rb:1` | ADR-0019, ADR-0020 | ADR-0003 | v1→v2 | RLS + **split** na face mecanismo — os dois colapsam, deduplicado |
| `app/models/dashboard_metric.rb:1` | ADR-0007 | ADR-0010 | v1→v2 | projeção reconstrutível |
| `app/models/domain_event.rb:1` | ADR-0009, ADR-0020 | ADR-0014, ADR-0004 | v1→v2 | auditoria imutável → 0014; **split** na face publish → 0004 |
| `app/models/domain_event.rb:2` | ADR-0023 | ADR-0012 | v1→v2 | eventos platform-scope |
| `app/models/identity.rb:1` | ADR-0022 | ADR-0011 | v1→v2 | seam de provedores de auth |
| `app/models/inbound_message.rb:2` | ADR-0010, ADR-0011 | ADR-0007, ADR-0013 | v1→v2 | webhook → 0007; **split**: o comentário diz "(encryption)" → 0013 |
| `app/models/membership.rb:2` | ADR-0023 | ADR-0012 | v1→v2 | memberships/RBAC |
| `app/models/municipality_channel.rb:1` | ADR-0021, ADR-0011/0024 | ADR-0007, ADR-0013 | v1→v2 | canal → 0007; AR Encryption e config por cidade colapsam em 0013, deduplicado |
| `app/models/outbound_message.rb:1` | ADR-0014 | ADR-0005 | v1→v2 | envio fora do lock |
| `app/models/processed_event.rb:2` | ADR-0005 | ADR-0005 | v1→v2 | `processed_events`; número coincide |
| `app/models/protocol_definition.rb:1` | ADR-0009 | ADR-0009 | já v2 | o comentário já diz "(Protocol engine)", título do v2 |
| `app/models/report_snapshot.rb:1` | ADR-0007 | ADR-0010 | v1→v2 | snapshot congelado |
| `app/models/triage.rb:2` | ADR-0006, ADR-0013 | ADR-0004, ADR-0009 | v1→v2 | o próprio comentário nomeia os temas: "commands" e "motor de protocolos" |
| `app/models/user.rb:1` | ADR-0022 | ADR-0011 | v1→v2 | identidade global |
| `app/models/user.rb:2` | ADR-0023 | ADR-0012 | v1→v2 | desativação por end-dating é RBAC append-only |
| `app/queries/admin/classification_query.rb:3` | ADR 0007 | ADR 0010 | v1→v2 | lê do snapshot congelado |
| `app/queries/admin/events_query.rb:3` | ADR 0003/0009 | ADR 0004/0014 | v1→v2 | domain events → 0004; auditoria/replay → 0014 |
| `app/queries/admin/protocols_query.rb:78` | ADR-0020 | ADR-0004 | v1→v2 | **split**: "IDs viajam no payload" é o publish carimbado |
| `app/queries/admin/triage_trail_query.rb:3` | ADR 0015 | ADR 0009 | v1→v2 | v1 0015 = `Outcome`/trail, que é o que a query lê |
| `app/queries/admin/triage_trail_query.rb:22` | ADR-0020 | ADR-0004 | v1→v2 | **split**: payload do evento |

## app/controllers

| Ocorrência | Antes | Depois | Decisão | Justificativa |
|---|---|---|---|---|
| `app/controllers/admin/api/base_controller.rb:3` | ADR-0022 | ADR-0011 | v1→v2 | cookie de sessão |
| `app/controllers/admin/api/base_controller.rb:15` | ADR-0019 | ADR-0003 | v1→v2 | `TenantScopedRequest` = RLS |
| `app/controllers/admin/api/triages_controller.rb:7` | ADR 0015 | ADR 0009 | v1→v2 | trail é o `Outcome` do motor |
| `app/controllers/application_controller.rb:8` | ADR-0019 | ADR-0003 | v1→v2 | `around_action :within_tenant` |
| `app/controllers/authoring/protocols_controller.rb:2` | ADR-0022, ADR-0019 | ADR-0011, ADR-0003 | v1→v2 | sessão + RLS. Arquivo escrito DEPOIS da refundação e ainda em numeração v1 |
| `app/controllers/concerns/authentication.rb:8` | ADR-0022 | ADR-0011 | v1→v2 | autenticação |
| `app/controllers/concerns/mfa_step_up.rb:2` | ADR-0022, ADR-0016 | ADR-0011, ADR-0009 | v1→v2 | MFA → 0011; ato de publicação protocolo → 0009 |
| `app/controllers/concerns/tenant_scoped_request.rb:3` | ADR-0019 | ADR-0003 | v1→v2 | tenant antes de qualquer SQL |
| `app/controllers/passwords_controller.rb:1` | ADR-0022 | ADR-0011 | v1→v2 | reset de senha |
| `app/controllers/protocols_controller.rb:1` | ADR-0016 e ADR-0017 | ADR-0009 | v1→v2 | os dois colapsam no motor — deduplicado |
| `app/controllers/publications_controller.rb:1` | ADR-0022 + ADR-0016 | ADR-0011 + ADR-0009 | v1→v2 | step-up MFA + publicação |
| `app/controllers/reports_controller.rb:1` | ADR-0007 | ADR-0010 | v1→v2 | endpoint público do snapshot congelado |
| `app/controllers/sessions_controller.rb:1` | ADR-0022 | ADR-0011 | v1→v2 | sessões |
| `app/controllers/sessions_controller.rb:29` | ADR-0022 | ADR-0011 | v1→v2 | TOTP no login |
| `app/controllers/sessions_controller.rb:50` | ADR-0022 | ADR-0011 | v1→v2 | seam gov.br |
| `app/controllers/setup_controller.rb:2` | ADR-0023, ADR-0024 | ADR-0012, ADR-0013 | v1→v2 | o comentário nomeia os temas: memberships/authz e provisionamento |
| `app/controllers/webhooks/whatsapp_controller.rb:1` | ADR-0010 | ADR-0007 | v1→v2 | webhook do WhatsApp Cloud API |

## config, lib, deploy

| Ocorrência | Antes | Depois | Decisão | Justificativa |
|---|---|---|---|---|
| `config/application.rb:9` | ADR-0013 | ADR-0009 | v1→v2 | namespace do motor (`Protocols::Validator`) |
| `config/application.rb:27` | ADR-0022 | ADR-0011 | v1→v2 | auth via cookie |
| `config/application.rb:36` | ADR-0004 | ADR-0004 | v1→v2 | `enqueue_after_transaction_commit`; número coincide |
| `config/application.rb:40` | ADR-0001 | ADR-0001 | v1→v2 | Solid Queue como adapter; número coincide |
| `config/database.yml:10` | ADR-0019 | ADR-0003 | v1→v2 | roles `rota_app`/`rota_admin` são do RLS |
| `config/initializers/active_record_encryption.rb:1` | ADR-0011 | ADR-0013 | v1→v2 | **split**: chaves do AR Encryption = custódia |
| `config/initializers/cors.rb:8` | ADR-0022 | ADR-0011 | v1→v2 | cookie de sessão |
| `config/initializers/domain_events.rb:1` | ADR-0003 + ADR-0020 | ADR-0004 | v1→v2 | colapsam — deduplicado |
| `config/initializers/filter_parameter_logging.rb:1` | ADR-0010, ADR-0011 | ADR-0007, ADR-0013 | v1→v2 | **split de v1 0011**: a lista é dominada por credenciais (`secret`, `token`, `_key`, `crypt`, `salt`, `otp`, `authorization`) → custódia (0013), não retenção (0014) |
| `config/initializers/protocols_facade.rb:1` | ADR-0013, ADR-0016 | ADR-0009 | v1→v2 | motor e storage colapsam — deduplicado |
| `config/initializers/protocols_facade.rb:25` | ADR-0007 / ADR-0016 | ADR-0010 / ADR-0009 | v1→v2 | relatórios históricos + versão exata do protocolo |
| `config/locales/conversation_advance.pt-BR.yml:1` | ADR-0012/0021 | ADR-0008/0007 | v1→v2 | consent + canal |
| `config/puma.rb:1` | ADR-0002 | ADR-0001 | v1→v2 | papéis web/worker da imagem única |
| `config/queue.yml:1` | ADR-0008 | ADR-0006 | v1→v2 | pools por fila/SLA |
| `config/recurring.yml:1` | ADR-0007, 0011, 0009 | ADR-0010 e ADR-0014 | v1→v2 | projeções + retenção/replay; 0011 e 0009 colapsam em 0014 — deduplicado |
| `config/routes.rb:8` | ADR-0022 | ADR-0011 | v1→v2 | sessão |
| `config/routes.rb:12` | ADR-0022 | ADR-0011 | v1→v2 | reset de senha |
| `config/routes.rb:15` | ADR-0022 | ADR-0011 | v1→v2 | MFA |
| `config/routes.rb:20` | ADR-0022 | ADR-0011 | v1→v2 | gov.br OIDC |
| `config/routes.rb:24` | ADR-0023/0024 | ADR-0012/0013 | v1→v2 | memberships + provisionamento |
| `config/routes.rb:35` | ADR-0002 | ADR-0001 | v1→v2 | healthcheck do Kamal |
| `config/routes.rb:38` | ADR-0010 | **ADR-0007** | v1→v2 | webhook WhatsApp. **Troca com a linha 44** |
| `config/routes.rb:44` | ADR-0007 | **ADR-0010** | v1→v2 | relatório público congelado. **Troca com a linha 38** |
| `config/routes.rb:47` | ADR-0016 | ADR-0009 | v1→v2 | autoria/preview de protocolo |
| `config/routes.rb:63` | ADR-0022 + ADR-0016 | ADR-0011 + ADR-0009 | v1→v2 | step-up MFA + publicação |
| `config/routes.rb:66` | ADR-0018 | ADR-0002 | v1→v2 | v1 0018 = topologia dos 4 apps |
| `deploy/SECRETS.md:1` | ADR-0024 | ADR-0013 | v1→v2 | custódia de secrets |
| `deploy/development/deploy.yml:2` | ADR-0002 | ADR-0001 | v1→v2 | imagem única, papéis web/worker |
| `deploy/development/deploy.yml:3` | ADR-0011 | ADR-0013 | v1→v2 | **split**: secrets |
| `deploy/production/deploy.yml:2` | ADR-0002 | ADR-0001 | v1→v2 | idem |
| `deploy/production/deploy.yml:3` | ADR-0011 | ADR-0013 | v1→v2 | **split**: secrets |
| `lib/migration_helpers/rls.rb:2` | ADR-0019 | ADR-0003 | v1→v2 | `FORCE ROW LEVEL SECURITY` |
| `lib/tasks/bootstrap.rake:1` | ADR-0019 | ADR-0003 | v1→v2 | bootstrap sob RLS |
| `lib/tasks/bootstrap.rake:55` | ADR-0019 | ADR-0003 | v1→v2 | least-privilege do `rota_app` |

### Fora do escopo — o ponteiro dentro do contrato

`schema.json:5` diz `"Contrato compartilhado por motor Ruby (ADR-0013) e preview
TS (ADR-0016)"` — numeração v1, que em v2 seria **0009** nos dois casos.

Não foi corrigido aqui de propósito. O arquivo existe em **três cópias
byte-idênticas** (`contracts/protocols/`, `packages/protocols/`,
`apps/api/config/protocols/`), e mexer só na do `api` quebraria a identidade que
o ADR 0015 exige. Corrigir direito é uma mudança no repo `contracts` com entrada
de CHANGELOG e tag — entrega própria, não um efeito colateral desta.

Por isso o inventário e a spec de guarda não varrem `.json`: varrer tornaria a
guarda permanentemente vermelha por causa de um arquivo que este repo não pode
consertar sozinho.
