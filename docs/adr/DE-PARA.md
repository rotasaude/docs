# De-para — corpus v1 → v2

A ponte entre o corpus antigo (26 ADRs com emendas + uma nota de custódia) e o corpus
consolidado (15 ADRs lineares). Sem ela, toda referência externa a um número v1 fica
órfã. O corpus v1 é preservado read-only no histórico local do projeto (fora deste repositório).

## v1 → v2 (onde foi parar cada ADR antigo)

| ADR v1 | Título v1 | Foi para v2 | Observação |
|---|---|---|---|
| 0001 | Transporte: Solid Queue | 0001 | fundido com 0002 (estrutura/papéis) |
| 0002 | Estrutura de aplicações | 0001 | papéis web/worker preservados; layout físico → 0002 |
| 0003 | Pub/sub de domínio (DomainEvents) | 0004 | publish em estado final (com tenant) |
| 0004 | Atomicidade write + enqueue | 0004 | — |
| 0005 | Idempotência dos consumers | 0005 | — |
| 0006 | Camada de commands | 0004 | commands na espinha de escrita |
| 0007 | CQRS: projeções e snapshots | 0010 | — |
| 0008 | Filas com prioridade (segurança clínica) | 0006 | — |
| 0009 | Auditoria LGPD via domain_events | 0014 | replay + retenção + LGPD |
| 0010 | Borda de ingestão: ack rápido e idempotência | 0007 | fundido com 0021 |
| 0011 | Payload bruto: criptografia e retenção | **0013** (secrets+cripto) + **0014** (retenção) | **SPLIT** |
| 0012 | Máquina de estados da conversa e consentimento | 0008 | — |
| 0013 | Motor de protocolos: fluxo e ramificação | 0009 | fundido com 0015/0016/0017 |
| 0014 | Resposta ao cidadão fora do lock | 0005 | HTTP fora do lock na espinha de consumo |
| 0015 | Classificação: prioridade e explicabilidade | 0009 | é o `Outcome` do motor |
| 0016 | Cadastro de protocolos: storage, workflow e RBAC | 0009 | roles operacionalizados no 0012 (cross-ref) |
| 0017 | Modelos de scoring: weighted e decision_table | 0009 | — |
| 0018 | Topologia física do monorepo: 4 apps | 0002 | papéis/fronteiras permanecem; layout → multi-repo |
| 0019 | Isolamento RLS multi-tenant | 0003 | — |
| 0020 | Tenant em eventos e jobs | **0003** (mecanismo) + **0004** (publish) + **0005** (idempotência §1.2) | **SPLIT** |
| 0021 | Roteamento de canal e ingestão multi-tenant | 0007 | fundido com 0010 |
| 0022 | Autenticação, sessão e MFA | 0011 | — |
| 0023 | RBAC: memberships e tier de plataforma | 0012 | — |
| 0024 | Provisionamento, custódia de secrets e config por cidade | 0013 | — |
| 0025 | Topologia multi-repo: um repositório por aplicação | 0002 | fundido com 0018 |
| 0026 | Política de versionamento do repositório `contracts` (Anexo A) | 0015 | decisão não-arquivada no v1; entrou direto no v2 |
| Nota | Custódia de secrets em multi-repo (Anexo B) | 0013 | nota não-arquivada; absorvida pelo ADR de provisionamento |

Todo ADR v1 (0001–0026) e a nota de custódia têm destino. Sem buraco.

## v2 → v1 (o que cada ADR novo consolidou)

| ADR v2 | Título v2 | Consolidou v1 |
|---|---|---|
| 0001 | Platform & application stack | 0001, 0002 |
| 0002 | Repository topology (multi-repo) | 0018, 0025 |
| 0003 | Multi-tenant isolation (RLS) | 0019, 0020 (mecanismo `with_tenant`) |
| 0004 | Domain events, commands & write atomicity | 0003, 0004, 0006, 0020 (publish) |
| 0005 | Consumer idempotency & side-effect isolation | 0005, 0014, 0020 (§1.2) |
| 0006 | Queues & clinical priority | 0008 |
| 0007 | WhatsApp ingestion & channel routing | 0010, 0021 |
| 0008 | Conversation state & consent | 0012 |
| 0009 | Protocol engine | 0013, 0015, 0016, 0017 |
| 0010 | CQRS: projections & snapshots | 0007 |
| 0011 | Identity, session & MFA | 0022 |
| 0012 | Authorization: RBAC & memberships | 0023 |
| 0013 | Provisioning, secrets & key custody | 0011 (secrets+cripto), 0024, Anexo B |
| 0014 | Audit log, retention & LGPD | 0009, 0011 (retenção) |
| 0015 | Contracts & versioning | 0026 (Anexo A) |

## Splits (uma metade inteira para cada destino, sem sobreposição)

- **v1 0011** → **0013** (gestão de secrets + AR Encryption + custódia) · **0014**
  (retenção/purga do `raw`).
- **v1 0020** → **0003** (mecanismo `with_tenant` / `SET LOCAL`) · **0004**
  (`publish` carimba tenant) · **0005** (interação com a idempotência, §1.2).

## Referências externas a atualizar

Lugares confirmados que ainda citam números v1 e precisam migrar para v2. As citações
ficam na pasta de origem (v1, read-only); a migração acontece reescrevendo os
sucessores no corpus v2.

| Origem (v1) | Cita | Migra em |
|---|---|---|
| `docs/modulos/README.md` | ADRs v1 por número | **Etapa 5** (reconciliação de módulos) |
| `docs/ciclo-desenvolvimento.md` | ADRs v1 por número | **Etapa 5** (reconciliação do ciclo) |
| `docs/operacao/` + `nota-revisao-operacional.md` | ADRs v1 por número | **Etapa 5** (reconciliação de runbooks) |
| `docs/adr/prompts/prompt-relatorio-semanal.md` | ADRs v1 | **Etapa 5** (checar/migrar) |
| `docs/superpowers/plans/2026-06-20-setup-multitenant.md` | ADR 0019–0024 | **Movimento 2** (alinhamento de código / Phase 2) |
| `docs/superpowers/2026-06-20-handoff-backend-to-frontend.md` | ADR 0019–0024 | **Movimento 2** (handoff; histórico-vivo) |

Não migram (snapshots datados, valor histórico):
`docs/relatorios/2026-06-*.md`, `docs/dashboard/alinhamento-2026-06-22.md` e os PRs
antigos — permanecem com os números v1 como registro da época.
