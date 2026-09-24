# Decisões de Arquitetura — Rota Saúde

Corpus consolidado (v2). Cada ADR documenta uma decisão vigente, completa, sem cadeia
de emendas. Numeração linear. O histórico pré-consolidação vive na pasta de arquivo
(ver [DE-PARA.md](DE-PARA.md)).

## Índice

| # | Item | Tema |
|---|------|------|
| 0001 | [Platform & application stack](0001.md) | Rails 8 + Solid Queue no Postgres; imagem única, papéis web/worker; identificadores em inglês |
| 0002 | [Repository topology (multi-repo)](0002.md) | Um repo por app + `contracts` + `docs`; compartilhar contrato, não código |
| 0003 | [Multi-tenant isolation (RLS)](0003.md) | Isolamento pelo Postgres; `SET LOCAL` por transação; `rota_app`/`rota_admin`; control plane vs data plane |
| 0004 | [Domain events, commands & write atomicity](0004.md) | Command muta + publica evento carimbado; `domain_events` imutável; enqueue após COMMIT |
| 0005 | [Consumer idempotency & side-effect isolation](0005.md) | `processed_events` exactly-once; corpo transacional; HTTP fora do lock |
| 0006 | [Queues & clinical priority](0006.md) | Filas por SLA; trabalho pesado nunca atrasa alerta urgente |
| 0007 | [WhatsApp ingestion & channel routing](0007.md) | Borda HMAC → rota → persiste sob tenant; dedup na borda |
| 0008 | [Conversation state & consent](0008.md) | Máquina de estados da conversa; `Consent` append-only, versionado, revogável |
| 0009 | [Protocol engine](0009.md) | Motor puro; `Outcome`; storage + façade; scoring weighted/decision_table |
| 0010 | [CQRS: projections & snapshots](0010.md) | `ReportSnapshot` imutável (prova); `DashboardMetric` reconstrutível |
| 0011 | [Identity, session & MFA](0011.md) | Auth nativa Rails 8; identidade global RLS-exempt; MFA operador/publisher |
| 0012 | [Authorization: RBAC & memberships](0012.md) | `memberships (user, city, role)`; operador = cidade nula; append-only; quatro-olhos de protocolo (ADR 0016) |
| 0013 | [Provisioning, secrets & key custody](0013.md) | Secrets via Kamal; AR Encryption; uma chave no env do `api`; `ProvisionMunicipality` |
| 0014 | [Audit log, retention & LGPD](0014.md) | `domain_events` como evidência imutável; replay; purga por retenção |
| 0015 | [Contracts & versioning](0015.md) | SemVer por domínio; expand/contract para MAJOR; tolerância do consumidor |
| 0016 | [Protocol signatures: two city reviewers per publication and per activation](0016.md) | `protocol_reviewer`; duas assinaturas para publicar e para ativar; substitui o quatro-olhos do ADR 0012 |
| 0017 | [Web citizen channel: declared identity and SMS-confirmed session](0017.md) | Cidadão entra pelo `wpda` com CPF declarado + celular por SMS; mesmo contrato de dados; WhatsApp desligável por cidade |
| 0018 | [Health units and attendances: from remote triage to in-person care](0018.md) | `health_units` mínima; check-in na unidade abre `attendances` a partir da triagem; desfecho e encaminhamento descritivo; código do balcão com finalidade |

## Itens em aberto

Reconhecidos como **não decididos** e ainda válidos no estado final do corpus. Cada um
vira ADR próprio quando entrar no ciclo.

### Arquitetura

- **UI de autoria de protocolos** — formato de edição, fluxo de revisão, preview lado
  a lado (ADR 0009).
- **Estado `:aborted` no `Outcome`** — a triagem revogada vive como
  `aborted_by_revocation` na `Triage`, sem estado correspondente no `Outcome`
  (ADR 0008/0009).
- **Mesclar as duas estratégias de scoring** no mesmo fluxo (ADR 0009).
- **Granularidade de consentimento por finalidade** (`Consent.scope`) e **canal de
  entrada por telefone (voz)** (ADR 0008; a web foi resolvida pelo ADR 0017).
- **Versionamento de schema de payload** de evento e **particionamento de
  `domain_events`** por volume (ADR 0004).
- **Seleção de canal no outbound** quando a cidade tiver mais de um número (ADR 0007).
- **Fila por município** (isolamento de carga) (ADR 0006).
- **Granularidade de `viewer`** e **delegação de `municipal_admin`** (ADR 0012).
- **Fatias do dashboard** por médico/posto e **payload para object storage** se o
  snapshot crescer (ADR 0010).
- **Reavaliação de Solid Queue / tamanho da imagem** sob carga (ADR 0001); **métricas
  agregadas do operador** (ADR 0003).
- **Mecanismo de distribuição de `contracts`** por domínio, **validação automática de
  categoria** e **compatibilidade de eventos persistidos** (ADR 0015).
- **Nome da organização**, **preservação de histórico do `api`** na extração e
  **release coordenado cross-repo** (ADR 0002).

### Confiabilidade e segurança clínica

- **Mecanismo de alerta urgente** — SLA + escalonamento + monitoramento +
  dead-letter-para-humano + plantão. O destino do alerta é config (ADR 0013); o
  mecanismo fim-a-fim é aberto (ADR 0006).
- **Idempotência HTTP** dos consumers de efeito externo (`NotifyCitizenJob`,
  `AlertMunicipalityJob`) (ADR 0005).
- **Corrida de primeiro contato** na ingestão (índice + tratamento explícito do
  conflito) (ADR 0007).

### Operação e custódia

- **Chave por tenant** (envelope encryption via KMS) — endurecimento pós-piloto
  (ADR 0013).
- **Rotação sem janela de break** de `WHATSAPP_APP_SECRET` e `REPORT_SIGNING_KEY`
  (ADR 0013).
- **Inventário de secrets por repo** (ADR 0013).
- **Suspensão/desprovisionamento de cidade** — retenção pós-saída, base legal
  (ADR 0013).
- **Biblioteca de templates de protocolo** (herança municipal/estadual) (ADR 0013/0009).
- **Retenção / TTL por município** (ADR 0003).

### Identidade e LGPD

- **Integração gov.br OIDC** e **recovery de MFA** além dos recovery codes (ADR 0011).
- **Imutabilidade × Art. 18 LGPD** — o que é apagável vs retido por base legal
  (ADR 0014). Inclui a **exclusão do cadastro do cidadão** do canal web (ADR 0017):
  `citizens` guarda CPF e celular, além de `citizen_sessions` e `otp_challenges`;
  decidido fora do subprojeto 2 (validação presencial) para análise própria.
- **Art. 20 LGPD** — revisão humana de decisão automatizada (ADR 0014).
- **Assinante real de `consent.revoked`** e base legal de retenção pós-revogação
  (ADR 0014).
- **Reconciliação de identidade do cidadão com CNS/SUS** (ADR 0014).
- **Mascaramento de dados pessoais na API de manutenção** — adiado; até existir, a
  API não expõe dado de cidadão
  ([registro](../superpowers/specs/2026-09-17-mascaramento-dados-sensiveis-decisao.md))
  (ADR 0014).

## Proveniência

Este corpus sucede o anterior (26 ADRs com cadeia de emendas + uma nota de custódia). O
mapeamento completo está em [DE-PARA.md](DE-PARA.md). O corpus antigo é preservado como
arquivo histórico, read-only, em [`_v1/adr/`](../_v1/README.md). Como o corpus
v2 foi construído está em [`../../_refundacao/`](../../_refundacao/PROVENIENCIA.md).
