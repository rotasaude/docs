# Etapa 10 — `contracts` (log)

- **Movimento:** 2 · **Etapa:** 10 · Segue o plano `08-plano-reconciliacao.md` §contracts.
- **Local:** `rota-saude/contracts/` (sibling de `apps/`/`packages/`). A raiz não é repo git
  — `contracts/` vira repositório próprio no split de topologia (ADR 0025). Sem build
  wired → nada a testar; é artefato de contrato.
- **Pré-condição satisfeita:** `contracts/events` esperava o rename C1 (feito na Etapa 9).

## Materializado (do material real)

| Domínio | Estado | Origem |
|---|---|---|
| `events/EVENTS.md` + `CHANGELOG.md` | **materializado** | extraído dos call sites reais (`DomainEvents.publish`/`Platform.audit`), EN pós-rename |
| `protocols/schema.json` + README + CHANGELOG | **materializado** | copiado de `packages/protocols/schema.json` (real) |
| `README.md` | **materializado** | política de versionamento por domínio (ADR 0015): SemVer, expand/contract, tolerância do consumidor, changelog obrigatório |

**Eventos documentados** (13): tenant-scoped (`inbound_message.received`, `triage.completed`,
`triage.urgent`, `consent.given/revoked`, `protocol.published/activated/retired`,
`membership.granted/revoked`) + platform-scope (`user.invited/deactivated`,
`municipality.provisioned`). Payloads reais; id do agregado no payload (C2).

## Scaffold (precisa de input — NÃO fabricado)

| Domínio | Por quê |
|---|---|
| `types/` | extração do contrato real da API (shapes request/response). A API ainda estabiliza (admin/setup/gov.br). Tarefa: extrair quando estável. |
| `design-tokens/` | `packages/ui` dissolvido aqui (ADR 0025), mas o conteúdo (paleta da marca, tipografia, espaçamento) é **decisão de design** — não inventado. Materializar quando o design existir. |

## Flags / decisões para o autor

1. **`protocol.created`** reaparece no contrato de eventos: emitido e lido
   (`admin/protocols_query`, `created_by`), não modelado no v2. Mesma situação de `authors`.
   Modelar (vira evento de 1ª classe) ou remover a feature? (pendência da Etapa 9 #1.)
2. **`types/` e `design-tokens/`** — materialização pendente de extração/design.
3. **`packages/` antigo** permanece até o split de topologia removê-lo (não-importado por
   ninguém; seguro manter ou remover depois).

>>> **ETAPA 10: events + protocols materializados; types + design-tokens scaffolded (pendem input). Versionamento (ADR 0015) documentado.** <<<
