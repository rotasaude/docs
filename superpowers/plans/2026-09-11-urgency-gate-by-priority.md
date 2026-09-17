# Urgency gate by priority Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Trocar o gate do alerta municipal de `outcome.tier == "alta"` para um limiar sobre `outcome.priority`, corrigindo tanto o acoplamento a vocabulário em português quanto o `priority_when` hoje ignorado.

**Architecture:** Um módulo puro `Protocols::Urgency` (padrão de `Protocols::PriorityRules`: total, nunca levanta) decide `urgent?(outcome)` por `priority <= limiar`; `CompleteTriage` passa a consultá-lo; o seed de demo tem suas prioridades ilegais (0) corrigidas para a faixa 1..9 do contrato.

**Tech Stack:** Ruby (módulos puros), RSpec, Docker Compose (`api-dev`), Vite/Vitest no `apps/dashboard`.

**Spec:** `docs/superpowers/specs/2026-09-11-urgency-gate-by-priority-design.md`

## Global Constraints

- **Commits em inglês**, em todas as apps.
- **`apps/api` está na branch `fix/migrations-owner-ddl-as-admin`** (11 commits à frente de `main`, sem PR). Commite **nessa branch**, não crie outra — o seed de demo que esta entrega conserta só existe nela. Use `git -C apps/api ...`.
- **As specs do api rodam DENTRO do container:** `docker exec api-dev bundle exec rspec <path>`. O Ruby do host é 3.4.4 e o `Gemfile` pede 3.3.6 — rodar no host falha.
- **O Docker pode estar parado.** Antes da Task 1, suba com `./start.sh` na raiz e confirme `docker ps` listando `api-dev`.
- **Semântica de priority:** inteiro 1..9, **menor = mais urgente**, garantido pelo contrato nos dois modos de scoring. `priority_when` só escala (`min`).
- **NÃO alterar `schema.json`** (nem em `contracts/protocols/`, nem em `apps/api/config/protocols/`, nem em `packages/protocols/`). A decisão escolhida não mexe no contrato — se você sentir vontade de mexer, pare e releia a spec.
- **NÃO alterar** `AlertMunicipalityJob`, `DispatchMunicipalityAlertJob`, `AlertMailer`, `ResendPendingAlertsJob`.
- **Deferido (NÃO implemente):** trocar `where(priority: true)` por `where(priority: 1)` em `Admin::ClassificationQuery`; qualquer campo novo de contrato para declarar urgência.

---

### Task 1: `Protocols::Urgency`

**Files:**
- Create: `apps/api/app/protocols/urgency.rb`
- Test: `apps/api/spec/protocols/urgency_spec.rb` (create)

**Interfaces:**
- Produces: `Protocols::Urgency.urgent?(outcome) -> true | false` — true quando `outcome` é terminal e sua `priority` é um inteiro `<= Protocols::Urgency.max_priority`. Total: nunca levanta.
- Produces: `Protocols::Urgency.max_priority -> Integer` — `ENV["URGENT_MAX_PRIORITY"]` se for inteiro válido, senão `DEFAULT_MAX_PRIORITY` (1).
- Consumes: `Protocols::Outcome` (já existe) — `#terminal?`, `#priority`.

- [ ] **Step 1: Write the failing test**

Crie `apps/api/spec/protocols/urgency_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe Protocols::Urgency do
  def terminal(priority:, tier: "high")
    Protocols::Outcome.terminal(trail: [], tier: tier, priority: priority)
  end

  it "is urgent at the threshold" do
    expect(described_class.urgent?(terminal(priority: 1))).to be(true)
  end

  it "is not urgent above the threshold" do
    expect(described_class.urgent?(terminal(priority: 2))).to be(false)
    expect(described_class.urgent?(terminal(priority: 9))).to be(false)
  end

  it "is not urgent when priority is absent" do
    expect(described_class.urgent?(terminal(priority: nil))).to be(false)
  end

  it "is not urgent for a pending outcome" do
    pending_outcome = Protocols::Outcome.pending(trail: [], awaiting: :febre)
    expect(described_class.urgent?(pending_outcome)).to be(false)
  end

  it "is tier-agnostic" do
    %w[alta high urgente vermelho].each do |tier|
      expect(described_class.urgent?(terminal(priority: 1, tier: tier))).to be(true)
    end
  end

  it "never raises on garbage" do
    expect(described_class.urgent?(nil)).to be(false)
    expect(described_class.urgent?("nope")).to be(false)
  end

  it "honours URGENT_MAX_PRIORITY" do
    allow(ENV).to receive(:fetch).and_call_original
    allow(ENV).to receive(:fetch).with("URGENT_MAX_PRIORITY", 1).and_return("3")
    expect(described_class.urgent?(terminal(priority: 3))).to be(true)
    expect(described_class.urgent?(terminal(priority: 4))).to be(false)
  end

  it "falls back to the default when the override is garbage" do
    allow(ENV).to receive(:fetch).and_call_original
    allow(ENV).to receive(:fetch).with("URGENT_MAX_PRIORITY", 1).and_return("banana")
    expect(described_class.max_priority).to eq(1)
  end
end
```

- [ ] **Step 2: Run test to verify it fails**

Run: `docker exec api-dev bundle exec rspec spec/protocols/urgency_spec.rb`
Expected: FAIL com `uninitialized constant Protocols::Urgency`.

- [ ] **Step 3: Write minimal implementation**

Crie `apps/api/app/protocols/urgency.rb`:

```ruby
# Decide se um Outcome terminal merece o alerta urgente à secretaria (F-01/F-03).
# Módulo puro e TOTAL — nunca levanta. Ver ADR 0006 (prioridade clínica nas
# filas) e ADR 0009 (motor de protocolos).
#
# A urgência é lida da `priority`, não do `tier`. `priority` é a única escala
# que o contrato garante nos DOIS modos de scoring (inteiro 1..9, menor = mais
# urgente) e o único valor que `priority_when` consegue escalar. `tier` é
# vocabulário livre do autor (schema.json: "tier": {"type":"string"}, sem enum),
# e usá-lo como gate fazia o alerta silenciar em qualquer cidade cujo protocolo
# não dissesse literalmente "alta".
module Protocols
  module Urgency
    # Política de plataforma: 1 = só a prioridade máxima alerta.
    DEFAULT_MAX_PRIORITY = 1

    module_function

    def max_priority
      Integer(ENV.fetch("URGENT_MAX_PRIORITY", DEFAULT_MAX_PRIORITY), exception: false) ||
        DEFAULT_MAX_PRIORITY
    end

    def urgent?(outcome)
      return false unless outcome.respond_to?(:terminal?) && outcome.terminal?

      priority = Integer(outcome.priority, exception: false)
      return false if priority.nil?

      priority <= max_priority
    end
  end
end
```

- [ ] **Step 4: Run test to verify it passes**

Run: `docker exec api-dev bundle exec rspec spec/protocols/urgency_spec.rb`
Expected: PASS (8 examples).

- [ ] **Step 5: Commit**

```bash
git -C apps/api add app/protocols/urgency.rb spec/protocols/urgency_spec.rb
git -C apps/api commit -m "Add Protocols::Urgency, a tier-agnostic urgency predicate"
```

---

### Task 2: `CompleteTriage` passa a usar o gate por priority

**Files:**
- Modify: `apps/api/app/commands/complete_triage.rb:28`
- Test: `apps/api/spec/commands/complete_triage_spec.rb` (create — o command não tem spec nenhuma hoje)

**Interfaces:**
- Consumes: `Protocols::Urgency.urgent?(outcome)` da Task 1.
- Produces: nenhuma interface nova. Muda o critério de emissão de `triage.urgent`.

- [ ] **Step 1: Write the failing test**

Crie `apps/api/spec/commands/complete_triage_spec.rb`:

```ruby
require "rails_helper"

RSpec.describe CompleteTriage do
  let(:muni) { create(:municipality) }

  # Vocabulário de tier em INGLÊS de propósito: é o que db/seeds/dashboard_demo.rb
  # já usa (TIER_CYCLE = %w[low medium high]) e o que o gate antigo silenciava.
  def definition_hash
    {
      "name" => "triagem-urgencia",
      "version" => 1,
      "start_step_id" => "febre",
      "steps" => [
        { "id" => "febre", "prompt" => "Febre?", "answer_type" => "boolean",
          "branches" => { "true" => nil, "false" => nil },
          "weights" => { "true" => 5, "false" => 0 } }
      ],
      "scoring" => { "type" => "weighted",
                     "thresholds" => { "low" => 0, "high" => 5 },
                     "priority_map" => { "low" => 9, "high" => 1 } }
    }
  end

  def build_triage(definition = definition_hash)
    pd = ProtocolDefinition.create!(
      name: definition["name"], version: 1, status: "active",
      definition: definition, municipality_id: muni.id
    )
    convo = Conversation.create!(
      municipality_id: muni.id, phone: "+5511977770000", state: :consented
    )
    convo.consents.create!(
      version: Consents.current_version(muni.id),
      policy_text_sha: Consents.policy_text_sha(Consents.current_version(muni.id)),
      given_at: 1.minute.ago, channel: "whatsapp", evidence: { text: "sim" }
    )
    Triage.create!(
      conversation: convo, protocol_definition: pd, protocol_name: definition["name"],
      municipality_id: muni.id, status: :in_progress,
      current_step: "febre", answers: {}
    )
  end

  def urgent_events = DomainEvent.where(name: "triage.urgent")

  around do |ex|
    ApplicationRecord.transaction do
      Current.municipality_id = muni.id
      ApplicationRecord.connection.execute(
        ApplicationRecord.sanitize_sql(["SET LOCAL app.municipality_id = ?", muni.id])
      )
      ex.run
      raise ActiveRecord::Rollback
    end
  end

  after { Current.reset }

  it "publishes triage.urgent for a priority-1 outcome whose tier is not 'alta'" do
    triage = build_triage

    expect { described_class.call(triage: triage, answer: "true") }
      .to change { urgent_events.count }.by(1)

    expect(triage.reload.tier).to eq("high")
    expect(triage.reload.priority).to eq(1)
  end

  it "does not publish triage.urgent for a low-priority outcome" do
    triage = build_triage

    expect { described_class.call(triage: triage, answer: "false") }
      .not_to change { urgent_events.count }

    expect(triage.reload.priority).to eq(9)
  end

  it "publishes triage.urgent when priority_when escalates a non-urgent tier" do
    definition = definition_hash.merge(
      "priority_when" => [{ "when" => { "eq" => ["febre", "false"] }, "priority" => 1 }]
    )
    triage = build_triage(definition)

    expect { described_class.call(triage: triage, answer: "false") }
      .to change { urgent_events.count }.by(1)

    expect(triage.reload.tier).to eq("low")
    expect(triage.reload.priority).to eq(1)
  end

  it "always publishes triage.completed on a terminal answer" do
    triage = build_triage

    expect { described_class.call(triage: triage, answer: "false") }
      .to change { DomainEvent.where(name: "triage.completed").count }.by(1)
  end
end
```

- [ ] **Step 2: Run test to verify it fails**

Run: `docker exec api-dev bundle exec rspec spec/commands/complete_triage_spec.rb`
Expected: FAIL nos exemplos 1 e 3 (`expected count to change by 1, but was changed by 0`) — o gate antigo exige `tier == "alta"`. Os exemplos 2 e 4 já passam.

- [ ] **Step 3: Write minimal implementation**

Em `apps/api/app/commands/complete_triage.rb`, troque a linha 28:

```ruby
          DomainEvents.publish("triage.urgent",    triage_id: @triage.id, **outcome.to_h) if outcome.tier == "alta"
```

por:

```ruby
          DomainEvents.publish("triage.urgent",    triage_id: @triage.id, **outcome.to_h) if Protocols::Urgency.urgent?(outcome)
```

- [ ] **Step 4: Run test to verify it passes**

Run: `docker exec api-dev bundle exec rspec spec/commands/complete_triage_spec.rb`
Expected: PASS (4 examples).

Depois rode a suíte inteira para garantir que nenhuma spec dependia do gate por tier:

Run: `docker exec api-dev bundle exec rspec`
Expected: PASS. Se alguma spec quebrar, ela estava codificando o defeito — corrija a spec, não o gate, e registre no commit.

- [ ] **Step 5: Commit**

```bash
git -C apps/api add app/commands/complete_triage.rb spec/commands/complete_triage_spec.rb
git -C apps/api commit -m "Gate the urgent municipal alert on priority, not on the tier label"
```

---

### Task 3: Corrigir as prioridades ilegais do seed de demo

**Files:**
- Modify: `apps/api/db/seeds/dashboard_demo.rb` (constante nova perto de `TIER_CYCLE`, linha ~149; uso na linha ~168)
- Test: verificação pela rake task existente (não há spec de seed no repo)

**Interfaces:**
- Consumes: nada das tasks anteriores.
- Produces: `SEED_PRIORITY -> Hash{String => Integer}` — mapa tier→priority dentro do seed de demo.

**Por que isto é obrigatório nesta entrega:** hoje o seed faz
`priority = tier == "high" ? 1 : 0`. Priority 0 é ilegal pelo contrato (mínimo 1)
e, com o gate novo (`priority <= 1`), **toda** triagem não-`high` do seed passaria
a parecer urgente. Sem esta task, a Task 2 troca um alerta que nunca dispara por
um que dispara sempre.

- [ ] **Step 1: Localizar as duas linhas**

Run: `grep -n 'TIER_CYCLE\|priority = tier' apps/api/db/seeds/dashboard_demo.rb`
Expected: `TIER_CYCLE` por volta da linha 149 e `priority = tier == "high" ? 1 : 0` por volta da 168.

- [ ] **Step 2: Adicionar o mapa de prioridade**

Logo abaixo da linha de `TIER_CYCLE`, adicione:

```ruby
  # Prioridade por tier, na faixa que o contrato exige (1..9, menor = mais
  # urgente). Antes disto o seed usava 0 para tudo que não fosse "high" — valor
  # ilegal pelo schema e que faria o gate de urgência (priority <= 1) tratar
  # toda triagem não-urgente como urgente.
  SEED_PRIORITY = { "high" => 1, "medium" => 5, "low" => 9 }.freeze
```

- [ ] **Step 3: Trocar o cálculo**

Substitua:

```ruby
      priority = tier == "high" ? 1 : 0
```

por:

```ruby
      priority = SEED_PRIORITY.fetch(tier)
```

- [ ] **Step 4: Reseedar e verificar**

```bash
docker exec api-dev bin/rails runner 'load Rails.root.join("db/seeds/dashboard_demo.rb")'
docker exec api-dev bin/rails db:seed:demo:verify
```

Expected: a rake task passa. `priorityTrue` não muda — ela é
`where(priority: true)`, que no Postgres compara com `1`, e só `high` continua
sendo 1.

Confirme também que nenhuma triagem ficou fora da faixa do contrato:

```bash
docker exec api-dev bin/rails runner 'puts Triage.where.not(priority: nil).where("priority < 1 OR priority > 9").count'
```

Expected: `0`.

- [ ] **Step 5: Commit**

```bash
git -C apps/api add db/seeds/dashboard_demo.rb
git -C apps/api commit -m "Seed demo triages with contract-legal priorities (1..9)"
```

---

### Task 4: Tirar o literal "alta" do tom de exibição do dashboard

**Files:**
- Create: `apps/dashboard/src/lib/tier.ts`
- Create: `apps/dashboard/src/lib/tier.test.ts`
- Modify: `apps/dashboard/src/modules/Reports.tsx` (remove a função privada `tierTone`, linhas ~42-47, e importa a nova)

**Interfaces:**
- Consumes: nada das tasks anteriores. Independente — pode ser feita fora de ordem.
- Produces: `tierTone(tier: string | null | undefined) => Tone` exportado de `src/lib/tier.ts`.

**Escopo e limite:** mesmo defeito de classe (literal `"alta"` codificado contra
um vocabulário livre), mas **cosmético** — decide só a cor do badge, não dispara
nem suprime alerta. E a correção aqui é **parcial por construção**: o endpoint
`/admin/api/reports` não devolve `priority` (`ReportRow` tem só
`id`, `createdAt`, `tier`, `protocol`, `expiresAt`, `live`), então não dá para
colorir por prioridade como o backend passou a fazer sem antes estender a query
e o tipo — o que está fora deste plano. O que dá para fazer sem tocar na API é
reconhecer os dois vocabulários que existem de fato no projeto (PT das specs,
EN do seed de demo) e degradar para neutro no resto, em vez de degradar para
neutro em tudo que não seja PT. Se o tempo apertar, esta é a task que se corta.

- [ ] **Step 1: Write the failing test**

Crie `apps/dashboard/src/lib/tier.test.ts`:

```ts
import { describe, it, expect } from "vitest";
import { tierTone } from "./tier";

describe("tierTone", () => {
  it("maps the PT vocabulary", () => {
    expect(tierTone("alta")).toBe("down");
    expect(tierTone("media")).toBe("warn");
    expect(tierTone("baixa")).toBe("ok");
  });

  it("maps the EN vocabulary used by the demo seed", () => {
    expect(tierTone("high")).toBe("down");
    expect(tierTone("medium")).toBe("warn");
    expect(tierTone("low")).toBe("ok");
  });

  it("is case- and accent-insensitive", () => {
    expect(tierTone("ALTA")).toBe("down");
    expect(tierTone("Média")).toBe("warn");
  });

  it("degrades to neutral for an unknown author vocabulary", () => {
    expect(tierTone("vermelho")).toBe("neutral");
    expect(tierTone(null)).toBe("neutral");
    expect(tierTone(undefined)).toBe("neutral");
    expect(tierTone("")).toBe("neutral");
  });
});
```

- [ ] **Step 2: Run test to verify it fails**

```bash
cd apps/dashboard && npm run test
```

Expected: FAIL — `Failed to resolve import "./tier"`.

- [ ] **Step 3: Write minimal implementation**

Crie `apps/dashboard/src/lib/tier.ts`:

```ts
// Tom de exibição a partir do tier de uma triagem.
//
// `tier` é vocabulário LIVRE do autor do protocolo (contracts/protocols/schema.json
// declara "tier" como string sem enum), então este mapa é best-effort: cobre os
// dois vocabulários que existem no projeto hoje — o PT das specs do api e o EN
// do seed de demo — e cai em "neutral" para qualquer outro, em vez de fingir
// que conhece a escala da cidade.
//
// O backend NÃO decide urgência por tier (ver Protocols::Urgency no api, que
// usa priority). Isto aqui é só cor.
import type { Tone } from "../theme/tokens";

const TONES: Record<string, Tone> = {
  alta: "down",
  high: "down",
  media: "warn",
  medium: "warn",
  baixa: "ok",
  low: "ok"
};

export function tierTone(tier: string | null | undefined): Tone {
  if (!tier) return "neutral";
  const key = tier
    .trim()
    .toLowerCase()
    .normalize("NFD")
    .replace(/[\u0300-\u036f]/g, "");
  return TONES[key] ?? "neutral";
}
```

- [ ] **Step 4: Run test to verify it passes**

```bash
cd apps/dashboard && npm run test
```

Expected: PASS — 35 testes (os 31 de antes + 4 novos).

- [ ] **Step 5: Usar a função nova no Reports**

Em `apps/dashboard/src/modules/Reports.tsx`:

1. apague a função privada `tierTone` (as ~6 linhas a partir de `function tierTone(t: string | null): Tone {`);
2. apague o `import type { Tone } from "../theme/tokens";` se ele ficar sem uso;
3. acrescente `import { tierTone } from "../lib/tier";`.

A chamada na linha 29 (`<Tag tone={tierTone(r.tier)}>`) não muda.

- [ ] **Step 6: Typecheck e testes**

```bash
cd apps/dashboard && npm run typecheck && npm run test
```

Expected: typecheck limpo (se reclamar de `Tone` importado e não usado, apague o import), 35 testes passando.

- [ ] **Step 7: Commit**

```bash
git -C apps/dashboard add src/lib/tier.ts src/lib/tier.test.ts src/modules/Reports.tsx
git -C apps/dashboard commit -m "Extract and widen tierTone so non-PT tier vocabularies render"
```

---

## Verificação final

- [ ] `docker exec api-dev bundle exec rspec` — suíte inteira verde.
- [ ] `cd apps/dashboard && npm run typecheck && npm run test` — verde, 35 testes.
- [ ] `git -C apps/api log --oneline -3` mostra os três commits do api; `git -C apps/dashboard log --oneline -1` mostra o do dashboard.
- [ ] Fumaça de ponta a ponta: com o demo reseedado, uma triagem `high` produz
      um `domain_events` com `name = 'triage.urgent'`:

```bash
docker exec api-dev bin/rails runner 'puts DomainEvent.where(name: "triage.urgent").count'
```

Nota: o seed cria triagens já completas direto no banco, sem passar por
`CompleteTriage`, então esse contador só sobe se o seed também semear o evento.
Se vier `0`, confirme o gate pelo caminho real, que é o que a spec da Task 2 já
exercita — não force o seed a emitir evento.
