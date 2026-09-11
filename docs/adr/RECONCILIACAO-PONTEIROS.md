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
