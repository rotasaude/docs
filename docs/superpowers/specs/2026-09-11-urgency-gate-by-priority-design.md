# Urgency gate by priority (não por tier)

**Date:** 2026-09-11
**Status:** Approved (análise → decisão do usuário → plan written)
**Module:** mod-03 (Triagem) / mod-01 (WhatsApp, borda do alerta)
**Touches:** `apps/api` (command + motor + seed + specs), `apps/dashboard` (tom de exibição do tier — cosmético e parcial: a API de relatórios não devolve `priority`)
**Não toca:** `contracts/` — a decisão escolhida NÃO muda o schema.

## Problem

`CompleteTriage` decide se a secretaria municipal é alertada comparando o `tier`
com um literal em português:

```ruby
# apps/api/app/commands/complete_triage.rb:28
DomainEvents.publish("triage.urgent", triage_id: @triage.id, **outcome.to_h) if outcome.tier == "alta"
```

`triage.urgent` é o **único** caminho de alerta: o initializer liga
`triage.urgent → AlertMunicipalityJob → DispatchMunicipalityAlertJob → AlertMailer`.
Se o evento não sai, ninguém é avisado, e nada falha visivelmente.

Mas o contrato (`contracts/protocols/schema.json`) declara `tier` como
`{"type": "string"}` **sem enum** — vocabulário livre do autor, em três lugares
(`scoring_decision_table.rules[].tier`, `.fallback.tier`, chaves de
`recommendations`). Uma cidade que publique um protocolo com tier `"high"`,
`"urgente"` ou `"vermelho"` passa no gate de publicação e **nunca alerta**.

### Dois defeitos, não um

1. **Vocabulário.** O gate exige `"alta"`. O próprio seed de demonstração do
   repo já usa inglês — `db/seeds/dashboard_demo.rb:149` define
   `TIER_CYCLE = %w[low medium high ...]` — logo os dados de demo nunca disparam
   alerta hoje.
2. **`priority_when` ignorado.** F-03.6 adicionou escalação de prioridade
   independente do modo de scoring (`Protocol#apply_priority_when`, escala-só,
   `min`). Ela mexe **só na priority**, nunca no tier. Um protocolo que declara
   "idade > 80 → priority 1" com tier resultante `"media"` **não alerta hoje**,
   mesmo o autor tendo dito explicitamente que aquilo é o caso mais urgente.

### Por que sobreviveu

Não há cobertura: `CompleteTriage` não tem spec (`spec/commands/` não contém
`complete_triage_spec.rb`), e a string `triage.urgent` não aparece em spec
nenhuma das 78.

## Current state (verified)

- `Outcome` é imutável e congelado; `terminal?`/`pending?`; `tier`, `priority`,
  `score`, `trail`, `awaiting`.
- **`priority` nunca é nil num outcome com scoring:**
  - `Scoring::Weighted#call` → `priority: priority_map.fetch(tier, 5)` — sempre Integer.
  - `Scoring::DecisionTable#call` → priority da regra casada ou do `fallback`
    (default `{tier: "indefinido", priority: 9}`) — sempre Integer.
  - Só um protocolo **sem** `scoring` produz `Outcome.terminal(trail:)` com
    `priority` nil — e esse hoje também não alerta (tier nil ≠ `"alta"`).
- **O contrato limita priority a `integer, minimum 1, maximum 9`** nos quatro
  pontos onde ela aparece (`priority_when[].priority`, `scoring_weighted.priority_map.*`,
  `scoring_decision_table.rules[].priority`, `.fallback.priority`).
  `PriorityRules.valid_priority` descarta `< 1` — "nunca escala para 0".
- Semântica consolidada: **menor = mais urgente**; `apply_priority_when` usa `min`.
- `schema.json` tem duas cópias byte-idênticas em uso (`contracts/protocols/` e
  `apps/api/config/protocols/`); verificado idênticas. (`packages/protocols/` é
  a cópia legada que a migração multi-repo removerá.)
- `DomainEvents.publish` levanta `TenantMissing` se `Current.municipality_id`
  for nil — specs precisam do `around` com `SET LOCAL app.municipality_id`.

## Decision

**Gate por `priority`, com limiar de plataforma.** (escolha do usuário)

`priority` é a única escala que o contrato garante nos dois modos de scoring,
é numérica e ordenada, e é o valor que `priority_when` escala. Trocar o gate
de `tier` para `priority`:

- corrige o defeito de vocabulário sem tocar o contrato (zero mudança de
  schema, zero CHANGELOG, zero tag, zero migração de protocolo existente);
- corrige de quebra o `priority_when` ignorado;
- deixa o limiar como **política de plataforma**, explícita e num só lugar.

### Forma

Módulo puro novo `Protocols::Urgency`, no padrão de `Protocols::PriorityRules`
(total — nunca levanta):

```ruby
Protocols::Urgency.urgent?(outcome) # => true/false
```

- `DEFAULT_MAX_PRIORITY = 1` — preserva a semântica de hoje para os protocolos
  canônicos, onde `priority_map` mapeia o tier mais alto para 1
  (`{"baixa" => 9, "alta" => 1}` no seed e nas specs).
- Override por ambiente: `URGENT_MAX_PRIORITY`. Valor inválido cai no default.
- Total: outcome não-terminal → false; `priority` nil → false (preserva o
  comportamento atual dos protocolos sem scoring).

### Consequência que precisa de conserto junto

`db/seeds/dashboard_demo.rb:168` faz `priority = tier == "high" ? 1 : 0`.
Priority **0 é ilegal pelo contrato** (mínimo 1) e, com um gate `priority <= 1`,
faria **toda triagem não-high do seed parecer urgente**. Tem que virar
`{"high" => 1, "medium" => 5, "low" => 9}` na mesma entrega. `priorityTrue` do
`Admin::ClassificationQuery` é `where(priority: true)` → `priority = 1` no
Postgres, então a contagem de demo não muda (só `high` continua sendo 1).

## Alternativas descartadas

- **Campo novo no contrato (autor declara os tiers urgentes).** MINOR aditivo,
  mas exige CHANGELOG + tag + as duas cópias do schema, e a cidade que esquecer
  de declarar continua sem alerta — a mesma falha silenciosa, só que declarada.
- **Enum fixo de tier (`baixa|media|alta`).** Faria o código atual ficar
  correto, mas é MAJOR com expand/contract (ADR 0015), remove o vocabulário
  livre do autor e quebraria o próprio seed de demo.

## Non-goals

- Não mudar `schema.json` nem versionar `contracts/`.
- Não mexer em `AlertMunicipalityJob`/`DispatchMunicipalityAlertJob`/`AlertMailer`
  — eles já recebem `tier` e `priority` e são só informativos.
- Não mexer em `ResendPendingAlertsJob` — ele reprocessa `triage.urgent`
  pendentes e é indiferente ao gate.
- Não trocar `where(priority: true)` por `where(priority: 1)` no
  `Admin::ClassificationQuery`. É um cheiro real (coluna integer comparada com
  booleano), mas é neutro em comportamento — **follow-up**, não este plano.

## Verification

- `Protocols::Urgency` coberto em unidade, incluindo o caso tier-agnóstico.
- `CompleteTriage` ganha sua primeira spec, com três casos: urgente por
  scoring, não-urgente, e urgente **só** por `priority_when` sobre tier
  não-urgente (o segundo defeito).
- Demo reseedado e verificado pela rake task existente.
