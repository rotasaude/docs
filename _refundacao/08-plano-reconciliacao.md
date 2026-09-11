# Plano de reconciliação — 2026-06-23 · CHECKPOINT

- **Movimento:** 2 · **Etapa:** 8 · **Fase A** (C2/C3 no v2): **aplicada, gate passado**.
- **Fase B:** este plano. **Não executa nada.** A execução é das Etapas 9–12, após sua aprovação.
- **Decisões fixas incorporadas:** C1 (rename PT→EN como etapa pós-Phase 5), C2 (v2 dropou `aggregate_*`), C3 (protocolo per-cidade, `active` é status, sem `active_protocol_versions`).

## Fase A — registro do que foi corrigido no v2

- **C3** no ADR 0009 + módulo 03: `protocol_definitions` volta a ser **per-cidade**
  (`municipality_id`); `active` é valor do campo `status`
  (`draft|in_review|published|active|retired`); unicidade por índice **parcial**
  `(municipality_id, protocol_key) WHERE status='active'`; tabela `active_protocol_versions`
  **descartada**; `ActivateProtocolVersion`/`RetireProtocolVersion` operam sobre o
  `status`; R1–R4 e INV-protocol-1..4 preservados (só mudou onde `active` vive).
- **C2** no ADR 0004: `domain_events` **sem** `aggregate_type`/`aggregate_id`; o id do
  agregado viaja no `payload` (ex.: `triage_id`); `municipality_id` (nullable) é o
  discriminador.

---

## Ordem dos repos

1. **`api`** — onde quase tudo vive; a Phase em-voo (4/5) está aqui; C2/C3 de código e o
   rename C1 acontecem aqui. É o gargalo: nada estável a jusante sem ele.
2. **`contracts`** — materializa events/protocols/types/design-tokens; depende do `api`
   estável (e, para `events/` em EN, do rename C1) para extrair os schemas reais.
3. **Frontends** (`admin-console`, `dashboard`, `wpda`) — consomem `contracts` e a API;
   vêm por último, na versão fixada do contrato.

Justificativa: a dependência é estritamente a jusante (frontends → contracts → api).
Inverter arriscaria materializar contratos sobre um `api` que ainda muda de nome (C1).

---

## api (o mais denso)

| # | Passo | Depende de | Risco | Validação |
|---|---|---|---|---|
| 1 | **Fix do filter-ordering (fecha Phase 4)** — tirar `include TenantScopedRequest` do `ApplicationController`; incluir explicitamente em cada controller de domínio **depois** de `Authentication`, para `current_municipality` ler `Current.user` real | — | **alto** | request specs reais (não só `prepend_before_action` de teste) verdes; produção desbloqueada |
| 2 | **Fechar Phase 5 (webhook real)** — `Whatsapp::Ingest` direto; restaurar corpo do `SendWhatsappJob` (`perform_original`) com `MunicipalityChannel`; remover bind stale `inbound_message.received`; corrigir signature `NotifyCitizenJob → SendWhatsappJob` | 1 | **alto** | webhook processa fim-a-fim sem `TenantMissing`; outbound entrega; dedup na borda |
| 3 | **C2 no código** — confirmar que `aggregate_*` segue ausente (Phase 2.2 removeu) e que nada novo os reintroduziu; reconciliar queries admin que ainda referenciam `aggregate_*` (`triage_trail_query`, `events_query`, `protocols_query`) | 1 | médio | grep zero `aggregate_*`; queries admin verdes |
| 4 | **C3 no código** — `protocol_definitions` já é per-cidade; **adicionar** o estado `active` ao enum de `status` e o índice parcial `WHERE status='active'`; refatorar `Protocols::Publish` (hoje seta `status='active'` direto) para `in_review → published`; **implementar** `ActivateProtocolVersion` (`published → active`, R1) e a guarda **R4** em `RetireProtocolVersion`; emitir `protocol.activated`/`protocol.retired`; revisar `protocol.created` (existe no código, não no v2) | 1 | médio | comandos de lifecycle verdes; INV-protocol-1..4 em teste; `published ≠ active` exercido |
| 5 | **Rename PT→EN (C1) — SÓ após Phase 5 verde** — etapa **atômica**: `triagens→triages`, `triagem_id→triage_id` (FK em `report_snapshots`), `complete_triagem→complete_triage`, eventos `triagem.completed/urgent → triage.*`, `protocol_name` e a tabela `authors` se confirmado domínio; migration de rename + refactor de referências + specs | **2** | **ALTO (o maior)** | suíte inteira verde pós-rename; grep zero identificador de domínio PT (`triagem`/`triagens`) |
| 6 | **Cobrir invariantes do v2 em teste** — as 4 de protocolo (INV-protocol-1..4), RLS (falha-fechada, isolamento), atomicidade (write+enqueue), idempotência (`(event_id,consumer)`) | 3,4,5 | baixo | camada de invariante do ciclo completa |

> **Nota de flexibilidade:** os passos **3 e 4 (C2/C3)** não dependem da Phase 5 (webhook) —
> tocam eventos/protocolo, não ingestão. Podem rodar logo após o passo 1. Mantive-os
> antes do rename porque o rename (5) deve ser o último ato sobre base estável. Só o
> **rename (5) depende do passo 2 (Phase 5)**.

---

## contracts

Materialização nova (não há código a migrar; `packages/ui` tem 1 arquivo — dissolução trivial).

| # | Passo | Depende de | Risco | Validação |
|---|---|---|---|---|
| 1 | `events/` — extrair dos eventos reais do `api` (estado final, **EN após C1**) | api#5 (rename) | médio | todo evento emitido tem schema; nomes EN |
| 2 | `protocols/` — JSON Schema da definição (**per-cidade**, C3) | api#4 | baixo | schema valida as definições reais |
| 3 | `types/` — contrato de API (shapes de request/response) | api estável | baixo | tipos batem com a API |
| 4 | `design-tokens/` — materializar cores/espaçamento/tipografia como dado; dissolver `packages/ui` | — | baixo | tokens consumíveis; `ui` removido |
| 5 | CHANGELOG por domínio + tags iniciais (`events-v1.0.0`, etc.) | 1–4 | baixo | política do ADR 0015 honrada |

**Dependência — RESOLVIDA pelo autor (opção a):** `contracts/events` **espera o rename
C1** (api#5). Nasce já EN, sem janela de desalinhamento. `contracts/events` fica
bloqueado até o rename fechar; os demais domínios de `contracts` (protocols/types/
design-tokens) não dependem de C1 e podem ser materializados antes.

---

## frontends

Estado atual inventariado (read-only): `admin-console` = 267 arquivos (app real →
**alinhar**); `wpda` = 31 arquivos (parcial → **completar/alinhar**); `dashboard` = **0
arquivos** (→ **scaffold do zero**, não migração).

| Repo | # | Passo | Depende de | Risco |
|---|---|---|---|---|
| admin-console | 1 | Alinhar ao papel v2 (plataforma, cross-tenant, `rota_admin`); reconciliar com a API pós-Phase 4/5 | api estável + contracts | médio |
| admin-console | 2 | Consumir `contracts` na versão fixada; honrar tolerância (campo/enum/evento desconhecido não quebra) | contracts | baixo |
| dashboard | 1 | **Scaffold do zero** (papel cidade, tenant-scoped, `rota_app`); 9 painéis do brief | api + contracts + brief | médio |
| dashboard | 2 | Consumir `contracts`; tolerância | contracts | baixo |
| wpda | 1 | Completar/alinhar (paciente, token assinado, público); resultado de triagem | api + contracts | médio |
| wpda | 2 | Consumir `contracts`; tolerância | contracts | baixo |

> O **brief formal do dashboard** segue item em aberto (não migrado na Etapa 5) — é
> pré-requisito do scaffold do `dashboard`.

---

## Cadeia de dependências crítica

```
api#1 (fix filter-ordering)  →  api#2 (Phase 5 webhook)  →  api#5 (rename C1)
                                                              →  contracts/events (EN)
                                                                   →  frontends
```

- **api#1** é o primeiro nó — desbloqueia produção; tudo a jusante o assume.
- **api#3/#4 (C2/C3)** correm em paralelo após api#1 (independem de Phase 5).
- **api#5 (rename)** é o nó de maior risco e só roda após **api#2 (Phase 5)**.
- **contracts/events EN** espera o rename (caminho seguro (a)); os demais domínios de
  `contracts` (protocols/types/design-tokens) não dependem de C1.
- **frontends** assumem `contracts` materializado e a API estável.

---

## Riscos que exigem atenção do autor

1. **Rename C1 (api#5) é o passo de maior risco** — atômico, pós-Phase 5, base estável.
   Toca migration, model, comando, 2 contratos de evento, FK `report_snapshots.triage_id`,
   specs e queries. Um rename parcial deixa o sistema meio-PT-meio-EN — pior que PT
   consistente. Recomenda-se branch dedicada + suíte inteira como gate.
2. **Sequenciamento `contracts/events` EN vs rename** — decisão (a)/(b) acima. Recomendo
   (a) (events espera o rename); (b) introduz janela que quebra consumidores.
3. **Bug de filter-ordering (api#1) bloqueia produção** — é o primeiro nó; nada real
   funciona sem ele, mesmo com Phase 4 "completa" no ledger.
4. **`authors` (tabela extra) e `protocol.created` (evento extra)** — **decisão do autor:
   são extras legítimos.** Permanecem; documentar cada um como **decisão nova (ADR)** ao
   tocá-los durante C3/rename. Não remover, não renomear (salvo o rename PT→EN se forem
   de domínio).
5. **Phase 4/5 são trabalho do superpowers em-voo** — a reconciliação **não** deve forçar
   EN nem o modelo v2 no meio delas; espera fecharem (decisão C1).

>>> **CHECKPOINT: APROVADO pelo autor — 2026-06-23** <<<

Decisões registradas:
- **(a)** Ordem `api → contracts → frontends` e cadeia crítica — **confirmadas**.
- **(b)** `contracts/events` **espera o rename C1** (opção a, sem janela de desalinhamento).
- **(c)** `authors` e `protocol.created` são **extras legítimos** — permanecem, viram ADR
  próprio ao serem tocados.
- **(d)** Rename C1 = **passo atômico final do `api`, pós-Phase 5** — confirmado.

As Etapas 9–12 ficam **liberadas para execução** (modificam código), nesta sequência.
