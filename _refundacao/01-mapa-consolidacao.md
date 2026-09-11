# Mapa de consolidação — APROVADO (2026-06-23)

- **Movimento:** 1 · **Etapa:** 1 · **Status: APROVADO pelo autor** (libera Etapa 2)
- **Entradas:** `etapa-0-destilado.md` (24 ADRs + 0025) · Anexo A (ADR 0026 contracts) · Anexo B (custódia multi-repo)
- **Decisões fixas:** corte limpo · re-numeração do zero · **idioma TUDO EN** · chave AR Encryption ancorada no `api`.
- **Resultado:** **N=15** ADRs lineares no corpus v2.

> Este arquivo é a **espinha autoritativa** da refundação. Etapas 2 (scaffold) e 3
> (reescrita) consomem o mapa abaixo. O corpus v1 permaneceu intocado (read-only).

---

## Decisões do autor aplicadas (2026-06-23)

| # | Pergunta | Decisão | Efeito no mapa |
|---|---|---|---|
| **(a)** | N=16 ou N=15? | **N=15 — fundir** secrets/cripto/custódia + provisionamento num só ADR | ADR 0013 mesclado; **0024 e Anexo B deixam de ser split** (caem inteiros nele) |
| **(b)** | Confirma D1–D5? | **Confirmado** | 0011 fora do WhatsApp · 0014 no consumo · commands na escrita · 0020 em 3 · reordenação |
| **(c)** | §2.4 reconstruir ou perder? | **Aceitar a perda** (julguei não-crítico — ver abaixo) | §2.4 não portado; invariantes viram alvo de teste no v2 |
| **(d)** | `domain_events` / roles em dois ADRs? | **"mesma tabela cumprindo dois papéis que pertencem a dois ADRs"** | princípio aplicado a `domain_events` (0004+0014) **e** aos roles de protocolo (0009+0012) |

### Veredito do §2.4 (lacuna de testes)
**Não-crítico → perda aceita.** O §2.4 era *observação* de cobertura faltante, não
*decisão* arquitetural. Só a dimensão *isolamento* foi recuperada (via 0019); as
demais dimensões não estão no corpus. Substância acionável ("quais invariantes
faltam testar") é **reconstruível dos 15 invariantes** do destilado — cada um é um
alvo de teste. Nenhuma decisão se perde. **Mitigação adotada:** o v2 declara os 15
invariantes como alvo explícito de cobertura (ADR 0014 + camada de invariante do
ciclo), o que subsume qualquer enumeração que o §2.4 tivesse.

### Aplicação de (d) — uma artefato, dois papéis, dois ADRs
- **`domain_events`** — mecanismo de publicação ao vivo → **0004**; ciclo de vida de
  evidência imutável (retenção, purga, replay) → **0014**. A tabela é uma; os dois
  papéis pertencem a ADRs diferentes. Confirmado.
- **roles `protocol_author`/`protocol_publisher`** — nomeados no workflow de
  publicação → **0009 (Protocol engine)**; operacionalizados como memberships →
  **0011 (RBAC)**. Mesmo princípio. Confirmado como cross-ref, não duplicação.

---

## Mapa final (N=15)

| Novo | Título (EN) | Consolida | Tema (uma frase) |
|---|---|---|---|
| **0001** | Platform & application stack | 0001, 0002 | Rails 8 + Solid Queue no Postgres; imagem única, papéis web/worker; **convenção: todos os identificadores em inglês** |
| **0002** | Repository topology (multi-repo) | 0018, 0025 | Um repo por app + `contracts` + `docs`; papéis e fronteiras de segurança dos quatro apps |
| **0003** | Multi-tenant isolation (RLS) | 0019, 0020(mecanismo) | Isolamento pelo Postgres; `app.municipality_id` via `SET LOCAL`; `rota_app`/`rota_admin`; control plane vs data plane |
| **0004** | Domain events, commands & write atomicity | 0003, 0006, 0004, 0020(publish) | Command muta + publica evento; `domain_events` carimba tenant + audita no mesmo átomo; enqueue após COMMIT |
| **0005** | Consumer idempotency & side-effect isolation | 0005, 0014, 0020(§1.2 xref) | `processed_events` exactly-once por consumer; corpo transacional; **HTTP fora do lock** em job próprio |
| **0006** | Queues & clinical priority | 0008 | Filas por SLA; trabalho pesado nunca atrasa alerta urgente |
| **0007** | WhatsApp ingestion & channel routing | 0010, 0021 | Borda HMAC → rota (`phone_number_id`→cidade) → persiste sob tenant; dedup na borda; número desconhecido → parking |
| **0008** | Conversation state & consent | 0012 | Máquina de estados da conversa; `Consent` append-only versionado/revogável; sem consent não há triagem |
| **0009** | Protocol engine | 0013, 0015, 0016, 0017 | Motor puro; `Outcome` (tier/priority/trail); storage no Postgres + façade `Protocols.current`; scoring weighted/decision_table |
| **0010** | CQRS: projections & snapshots | 0007 | `ReportSnapshot` imutável (prova); `DashboardMetric` reconstrutível |
| **0011** | Identity, session & MFA | 0022 | Auth nativa Rails 8; identidade global RLS-exempt; MFA no operador (login) e publisher (step-up); seam gov.br |
| **0012** | Authorization: RBAC & memberships | 0023 | `memberships (user, city, role)`; operador = membership de cidade nula; append-only; four-eyes |
| **0013** | Provisioning, secrets & key custody | **0011(secrets+enc), 0024, AnexoB** | Secrets via Kamal; `master.key`; **AR Encryption at rest**; `ProvisionMunicipality`; **uma chave protege todas as cidades, vive só no env do `api`**; config por cidade |
| **0014** | Audit log, retention & LGPD | 0009, 0011(retention) | `domain_events` como evidência imutável; replay de recuperação; purga por retenção; itens LGPD em aberto |
| **0015** | Contracts & versioning | Anexo A | SemVer por domínio em `contracts`; expand/contract para MAJOR; invariante de tolerância do consumidor |

---

## Cobertura (27 origens → 0 órfão)

| Origem | Vai para | Split? | Observação |
|---|---|---|---|
| 0001 | 0001 | não | — |
| 0002 | 0001 | não | papéis permanecem; layout superseded (nota) |
| 0003 | 0004 | não | publish em estado final (0020 §1.3) |
| 0004 | 0004 | não | — |
| 0005 | 0005 | não | — |
| 0006 | 0004 | não | **D3** — escrita |
| 0007 | 0010 | não | — |
| 0008 | 0006 | não | — |
| 0009 | 0014 | não | replay = recuperação do log de auditoria |
| 0010 | 0007 | não | — |
| **0011** | **0013 + 0014** | **SIM (3 partes)** | secrets-mgmt→0013; encryption-at-rest→0013; retention→0014 |
| 0012 | 0008 | não | — |
| 0013 | 0009 | não | — |
| **0014** | **0005** | não | **D2** — consumo, não WhatsApp |
| 0015 | 0009 | não | — |
| 0016 | 0009 | não | roles: cross-ref a 0012 (princípio (d)) |
| 0017 | 0009 | não | — |
| 0018 | 0002 | não | layout superseded por 0025 |
| 0019 | 0003 | não | — |
| **0020** | **0004 + 0003 + 0005** | **SIM (3 partes)** | publish→0004; mecanismo with_tenant→0003; §1.2→0005 (xref) |
| 0021 | 0007 | não | — |
| 0022 | 0011 | não | — |
| 0023 | 0012 | não | absorve operacionalização dos roles do 0016 |
| **0024** | **0013** | não | **(merge a)** custódia+provisionamento juntos → não split |
| 0025 | 0002 | não | topologia vigente |
| **Anexo A** | **0015** | não | decisão não-arquivada |
| **Anexo B** | **0013** | não | **(merge a)** custódia + rota provis. juntos → não split |

**Splits remanescentes (só 2, o merge eliminou os outros 2):**
1. **0011 (3 partes):** secrets-mgmt + encryption-at-rest → **0013**; retenção/purga → **0014**.
2. **0020 (3 partes):** publish-carimba → **0004**; mecanismo `with_tenant`/`SET LOCAL` → **0003**; interação-idempotência §1.2 → **0005** (cross-ref, decisão-mãe é o 0005).

---

## Flags resolvidos (estado final)

- **C2 (`packages/ui`):** RESOLVIDO. Não aparece como "em aberto" no v2; design-tokens em `contracts` (ADR 0015), governado pelo Anexo A. Entrada obsoleta do README v1 não portada.
- **C4 (idioma EN):** convenção fixada no **ADR 0001** (seção de naming) e reforçada no **0015** (contracts já nascem EN). Identificadores: `triages`, `consents`, `Triage`, `CompleteTriage`, `consent_terms`…
- **C9 (custódia):** Anexo B absorvido pelo **ADR 0013**. Invariante **"uma chave protege todas as cidades" PERMANECE** + "vive só no env do `api`; frontends nunca a recebem; `contracts` nunca carrega secret". Endurecimento (chave por tenant/KMS) = item em aberto pós-piloto.
- **C10 (§ órfãos):** 11 de 12 recuperados por referência (tabela na versão anterior deste arquivo / destilado). **§2.4 = perda aceita** (veredito acima).

---

## Ordem (justificativa)

Arco: **fundações → espinha de eventos → ciclo da mensagem → plataforma/admin → compliance/governança.**

| Bloco | ADRs v2 | Razão |
|---|---|---|
| Fundações | 0001 platform · 0002 topology · 0003 RLS | RLS é consumido por quase tudo → cedo |
| Espinha de eventos | 0004 escrita · 0005 consumo · 0006 filas | motor assíncrono usado pelo domínio inteiro |
| Ciclo da mensagem | 0007 ingest → 0008 conversa/consent → 0009 protocolo → 0010 CQRS | fluxo real: entra → conversa → tria → reporta |
| Plataforma/admin | 0011 identidade · 0012 RBAC · 0013 provisionamento/secrets | quem opera; como cidades nascem. **0013 vem após 0011/0012 porque provisionamento USA identidade+RBAC+eventos como blocos** (dependência pesada). Cripto-em-repouso é forward-ref de uma linha para os consumidores anteriores |
| Compliance/governança | 0014 audit/retenção/LGPD · 0015 contracts | atravessa tudo; fecha o corpus |

**Consequência aceita do merge (a):** o ADR 0013 é o mais carregado (custódia de
secrets + cripto + provisionamento + config por cidade). Internamente terá duas
partes claras (fundação de custódia | operação de provisionamento). A descoberta de
"como o dado é encriptado em repouso" mora num ADR cujo título lidera com
"Provisioning" — mitigado pelo subtítulo "secrets & key custody" e por cross-ref a
partir de 0003/0007/0008/0011.

---

## Pronto para Etapa 2

Mapa aprovado, 15 ADRs, cobertura completa, splits explícitos, 4 flags fechados.
A Etapa 2 (scaffold) cria a estrutura de pastas vazia do corpus v2 conforme este mapa.
