# Reconciliação dos ponteiros de ADR no código (v1 → v2)

Registro de decisão, uma linha por ocorrência. A reescrita dos comentários em
`rotasaude/api` foi **derivada desta tabela** — se um ponteiro do código não
bate com a linha aqui, a tabela é a fonte de verdade e o código está errado.

Método e mapa v1→v2: [`DE-PARA.md`](DE-PARA.md). Texto v1: [`_v1/adr/`](../_v1/README.md).

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

## db (migrations e seeds)

Só comentários foram tocados — nenhuma DDL mudou, e o `db:migrate:status` segue
com todas as migrations `up`.

| Ocorrência | Antes | Depois | Decisão | Justificativa |
|---|---|---|---|---|
| `20260618100003_create_conversations.rb:1` | ADR-0012 | ADR-0008 | v1→v2 | conversa/consentimento |
| `20260618100004_create_consents.rb:1` | ADR-0012 | ADR-0008 | v1→v2 | idem |
| `20260618100005_create_protocol_definitions.rb:1` | ADR-0016 | ADR-0009 | v1→v2 | storage de definições |
| `20260618100006_create_triagens.rb:1` | ADR-0006 e ADR-0013 | ADR-0004 e ADR-0009 | v1→v2 | commands + motor |
| `20260618100007_create_inbound_messages.rb:1` | ADR-0010 e ADR-0011 | ADR-0007 e ADR-0013 | v1→v2 | webhook + **split**: cripto do payload bruto |
| `20260618100008_create_outbound_messages.rb:1` | ADR-0014 | ADR-0005 | v1→v2 | envio fora do lock |
| `20260618100009_create_domain_events.rb:1` | ADR-0003 e ADR-0009 | ADR-0004 e ADR-0014 | v1→v2 | eventos + auditoria |
| `20260618100010_create_processed_events.rb:1` | ADR-0005 | ADR-0005 | v1→v2 | número coincide |
| `20260618100011_create_report_snapshots.rb:1` | ADR-0007 | ADR-0010 | v1→v2 | snapshot congelado |
| `20260618100012_create_dashboard_metrics.rb:1` | ADR-0007 | ADR-0010 | v1→v2 | projeção reconstrutível |
| `20260619210000_create_users_and_sessions.rb:2` | ADR-0022 | ADR-0011 | v1→v2 | sessão assinada |
| `20260620000001_create_database_roles.rb:1` | ADR-0019 | ADR-0003 | v1→v2 | `rota_app`/`rota_admin` |
| `20260620000010_add_municipality_id_to_data_plane.rb:3` | ADR-0019 | ADR-0003 | v1→v2 | coluna de tenant |
| `20260620000020_enable_rls_on_data_plane.rb:1` | ADR-0019 | ADR-0003 | v1→v2 | política `tenant_isolation` |
| `20260620000020_enable_rls_on_data_plane.rb:2` | ADR-0016 | ADR-0009 | v1→v2 | escopo de `protocol_definitions` |
| `20260620000030_processed_events_tenant…:2` | ADR-0020 | ADR-0003 | v1→v2 | **split**: `municipality_id` + RLS = mecanismo |
| `20260620000030_processed_events_tenant…:4` | ADR-0020 | ADR-0004 | v1→v2 | **split**: forma do payload publicado |
| `20260620000040_identity_for_multitenant.rb:1` | ADR-0022 | ADR-0011 | v1→v2 | identidade global |
| `20260620000040_identity_for_multitenant.rb:2` | ADR-0023 | ADR-0012 | v1→v2 | memberships |
| `20260620000060_domain_events_municipality_nullable.rb:1` | Emenda ADR-0023 ao ADR-0009 | ADR-0012 e ADR-0014 | v1→v2 | platform-scope + auditoria; a palavra "emenda" sai |
| `20260620000070_create_memberships.rb:2` | ADR-0023 | ADR-0012 | v1→v2 | memberships RLS-exempt |
| `20260620000090_create_municipality_channels.rb:1` | ADR-0021 | ADR-0007 | v1→v2 | roteamento de canal |
| `20260620000100_create_unknown_channels.rb:1` | ADR-0021 | ADR-0007 | v1→v2 | parking de canal desconhecido |
| `20260620000110_conversation_partial_index_by_tenant.rb:2` | ADR-0021 emenda 0012 | ADR-0008 | v1→v2 | índice sobre estados da conversa |
| `20260620000120_municipality_status_and_consent_terms.rb:1` | ADR-0024 | ADR-0013 | v1→v2 | provisionamento/config por cidade |
| `20260623000010_protocol_lifecycle_states.rb:6` | ADR-0019 | ADR-0003 | v1→v2 | dono da tabela sob RLS |
| `20260623000020_rename_triagens_to_triages.rb:12` | ADR-0019 | ADR-0003 | v1→v2 | `FORCE ROW LEVEL SECURITY` |
| `db/seeds.rb:15` | ADR-0022 | ADR-0011 | v1→v2 | TOTP a cada login |
| `db/seeds/dashboard_demo.rb:197` | ADR-0007 | ADR-0010 | v1→v2 | **ponteiro escrito por engano em numeração v1 em 2026-09-11**, no plano do gate de urgência; "prova imutável" é o v2 0010 |

## spec e `.md` da raiz do api

| Ocorrência | Antes | Depois | Decisão | Justificativa |
|---|---|---|---|---|
| `README.md:5` | ADR-0019 | ADR-0003 | v1→v2 | RLS/ownership não cabem no `schema.rb` |
| `RECONCILE_admin_console.md:13` | ADR-0022 | ADR-0011 | v1→v2 | cookie de sessão |
| `RECONCILE_admin_console.md:30` | ADR 0007 | ADR 0010 | v1→v2 | projeção `ingestion_metrics` |
| `RECONCILE_admin_console.md:31` | ADR 0011 | ADR 0014 | v1→v2 | **split**: backlog de purga é retenção/LGPD |
| `RECONCILE_admin_console.md:37` | ADR 0016 | ADR 0009 | v1→v2 | colunas de `protocol_definitions` |
| `RECONCILE_admin_console.md:41` | ADR-0022 | ADR-0011 | v1→v2 | RBAC/sessão real |
| `RECONCILE_admin_console.md:43` | ADR-0020 | ADR-0003 | v1→v2 | **split**: flag cross-tenant é o mecanismo de tenant |
| `RECONCILE_admin_console.md:99` | ADR-0022 + link `0022.md` | ADR-0011 + link `0011.md` | v1→v2 | **o link do arquivo também mudou**, não só o número |
| `RECONCILE_admin_console.md:107` | ADR 0007 | ADR 0010 | v1→v2 | ADR de read-side |
| `spec/commands/protocols_lifecycle_spec.rb:3` | ADR-0009 | ADR-0009 | já v2 | lifecycle de protocolo = v2 0009 |
| `spec/controllers/sessions_controller_govbr_spec.rb:3` | ADR-0022 | ADR-0011 | v1→v2 | seam gov.br (dentro da string do `describe`) |
| `spec/integration/event_dispatch_with_tenant_spec.rb:3` | ADR-0020 | ADR-0004 e ADR-0003 | v1→v2 | **split que não colapsa**: a spec exercita as duas faces — o publish carimbado (0004) e o consumer rodando sob o tenant (0003). Citar só uma seria mentira |
| `spec/rls/tenant_isolation_spec.rb:4` | ADR-0019 | ADR-0003 | v1→v2 | as quatro invariantes de RLS |
| `spec/support/admin_rls.rb:1` | ADR-0019 | ADR-0003 | v1→v2 | helper cross-tenant |

## Fecho

**Cobertura.** 228 ocorrências inventariadas em 140 arquivos de `rotasaude/api`,
distribuídas em **196 linhas distintas**. A tabela tem **196 linhas — uma por
linha de código, não por ocorrência**, porque a linha é a unidade de edição:
um comentário que cita dois ADRs é uma decisão só. Conferido: 196 = 196, sem
buraco. Zero casos marcados `DÚVIDA`.

**Resultado.** Inventário final: 235 ocorrências, **todas dentro de 0001..0015**.
O número subiu em relação às 228 iniciais porque o matcher passou a enxergar as
formas compactas (ver abaixo), não porque ponteiros foram acrescentados.

**Verificação.** Cada uma das 196 linhas foi conferida programaticamente contra
o arquivo real: o ponteiro que está no código é o que a coluna `Depois` manda.

### Duas correções ao próprio método, achadas durante a execução

1. **Forma compacta.** `ADR-0012/0013` carrega dois ponteiros, mas o regex
   original (`ADR[-\s]?(\d{4})`) só lia o primeiro — o segundo passava sem
   conferência, na guarda e no inventário. Nove ocorrências usam essa forma.
   Nenhuma escondia número fora da faixa, mas o furo era real: um
   `ADR-0007/0024` teria passado batido. Regex corrigido nos dois lugares.
2. **Forma sem prefixo.** `update_dashboard_job.rb` ficou, por um deslize desta
   própria reconciliação, com `(ADR-0010 + 0003)` — o `0003` sem `ADR-` não é
   reconhecível por ninguém: nem pela guarda, nem por quem faz grep. Reescrito
   como `(ADR-0010 e ADR-0003)`. **Ponteiro só conta se for procurável.**

### Fora do escopo

- `schema.json` (três cópias byte-idênticas) — ver seção própria acima.
- `rota-saude/scripts/*.rb` — não estão em repositório nenhum.
- A cópia local não-versionada de `rota-saude/docs/`.
