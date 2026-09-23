# Mostrar a versão-alvo da reversão — plano

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** quem confirma uma reversão de emergência — no painel da cidade ou na API de manutenção — lê **qual** versão deve voltar a valer, em vez de só a regra.

**Architecture:** um método de domínio, `Protocols::RevertActivation.revert_target`, passa a ser a única fonte: `revertible?` delega a ele. As duas leituras (`Admin::ProtocolsQuery` e `Maintenance::Types::ProtocolVersionType`) expõem a versão-alvo, e cada painel de confirmação nomeia o número.

**Tech Stack:** Rails 8.1 + graphql-ruby (apps/api); Vite + React 18 + Vitest (apps/dashboard, apps/maintenance).

**Spec:** `docs/superpowers/specs/2026-09-23-revert-target-design.md` (commit `38f4b1b`). Este plano implementa §3 a §7.

## Global Constraints

- Commits: Conventional Commits **em inglês**, **um** tipo por assunto, terminando com `Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>`.
- Git: `/opt/homebrew/bin/git`. Branch `feat/revert-target` em **cada** repo tocado (`apps/api`, `apps/dashboard`, `apps/maintenance`). Nunca em `main`, nunca push, nunca merge.
- Staging explícito por arquivo; nunca `git add -A` (`apps/dashboard` e `apps/maintenance` têm `.superpowers/` fora do git).
- Suíte completa do api: na raiz do monorepo, `docker compose stop worker`, depois `docker compose exec -T api bundle exec rspec`, depois `docker compose start worker` (sempre religar). Acima de ~3,5 min é regressão.
- **Não mudar o comportamento da reversão.** `Protocols::RevertActivation.call` fica intocado: ele trava as linhas e reconfere sob lock, e é ele quem decide. `revert_target` é leitura pura.
- **Uma fonte só:** depois desta mudança, `revertible?` não pode ter regra própria — deve ser `revert_target(protocol).present?`.
- A leitura é **previsão**: sem lock, o alvo pode mudar entre a leitura e o clique. O texto das telas diz "deve voltar", nunca "vai voltar".
- Só o **número** da versão chega às telas. Nada de definição de protocolo.
- Specs de request exigem `type: :request`; consultas GraphQL em specs ficam em métodos ou `let`, nunca em constante de topo.
- Nunca rode `start.sh`.

## File Structure

**apps/api**
- Modify: `app/commands/protocols/revert_activation.rb` — `revert_target`, e `revertible?` delegando.
- Modify: `app/queries/admin/protocols_query.rb` — `revertTargetVersion` no `signature_state`.
- Modify: `app/graphql/maintenance/types/protocol_version_type.rb` e `app/graphql/maintenance/types/city_type.rb` — o campo e o cálculo dentro do `inside`.
- Modify/Create: specs de `spec/commands/`, `spec/requests/admin/protocols_signatures_spec.rb` e `spec/requests/maintenance/city_protocol_versions_spec.rb`.

**apps/dashboard**
- Modify: `src/lib/types.ts` (o campo no `SignatureState`), `src/modules/Protocols.tsx` (a frase), `src/modules/Protocols.test.tsx`.

**apps/maintenance**
- Modify: `schema.graphql` + `src/gql/*` (via `npm run schema:pull && npm run codegen`), `src/screens/ProtocolsTab.tsx`, `src/screens/ProtocolsTab.test.tsx`.

---

### Task 1: `revert_target` e as duas leituras

**Files:**
- Modify: `apps/api/app/commands/protocols/revert_activation.rb`, `apps/api/app/queries/admin/protocols_query.rb`, `apps/api/app/graphql/maintenance/types/protocol_version_type.rb`, `apps/api/app/graphql/maintenance/types/city_type.rb`
- Test: `apps/api/spec/commands/protocols_revert_target_spec.rb` (novo), e acréscimos em `apps/api/spec/requests/admin/protocols_signatures_spec.rb` e `apps/api/spec/requests/maintenance/city_protocol_versions_spec.rb`

**Interfaces:**
- Consumes: `ProtocolActivation` (com `kind` `"signed"`/`"baseline"`), `ProtocolDefinition` (status `draft|in_review|published|active|retired`), e o `activation_history(name)` privado que já existe no command.
- Produces:
  ```ruby
  Protocols::RevertActivation.revert_target(protocol) # -> ProtocolDefinition | nil
  Protocols::RevertActivation.revertible?(protocol)   # -> revert_target(protocol).present?
  ```
  ```json
  // /admin/api/protocols e /admin/api/protocols/:id, por versão
  "revertTargetVersion": "1"   // string como `version` ali; null sem alvo
  ```
  ```graphql
  # Maintenance::Types::ProtocolVersionType
  revertTargetVersion: Int   # nulo sem alvo
  ```

- [ ] **Step 1: Branch**

```bash
cd apps/api && /opt/homebrew/bin/git checkout -b feat/revert-target
```

- [ ] **Step 2: Escrever a spec do domínio (falha)**

`spec/commands/protocols_revert_target_spec.rb`. O arranjo de ativações assinadas está em `spec/requests/admin/protocols_signatures_spec.rb` (o exemplo de `revertible`) e em `spec/commands/` — **leia um deles e copie o caminho**, que usa `Protocols::Activate` com assinaturas de verdade, em vez de criar `ProtocolActivation` na mão.

```ruby
require "rails_helper"

# A versão-alvo da reversão (spec 2026-09-23-revert-target §3): as MESMAS três
# condições que `call` exige, numa leitura pura. `revertible?` passa a derivar
# daqui — se as duas tivessem regra própria, a tela habilitaria o botão
# prometendo uma versão e a API reverteria para outra.
RSpec.describe Protocols::RevertActivation, ".revert_target" do
  # ... arranjo copiado do spec de assinaturas: autor salva v1 e v2, dois
  #     revisores assinam publicação e ativação de cada uma, e Protocols::Activate
  #     ativa v1 e depois v2 ...

  it "devolve a versão anterior quando a ativação corrente é assinada e a anterior está published" do
    expect(described_class.revert_target(v2.reload)).to eq(v1)
    expect(described_class.revertible?(v2.reload)).to be(true)
  end

  it "não devolve alvo para uma versão que não está em uso" do
    expect(described_class.revert_target(v1.reload)).to be_nil
    expect(described_class.revertible?(v1.reload)).to be(false)
  end

  it "não devolve alvo quando a ativação corrente é a linha-base" do
    # Uma versão ativa cuja única ativação é `kind: "baseline"` (o protocolo que
    # já estava em uso antes das assinaturas) não reverte: não há passo anterior.
    expect(described_class.revert_target(baseline_version.reload)).to be_nil
    expect(described_class.revertible?(baseline_version.reload)).to be(false)
  end

  it "não devolve alvo quando a versão anterior não está mais published" do
    v1.update!(status: "retired")

    expect(described_class.revert_target(v2.reload)).to be_nil
    expect(described_class.revertible?(v2.reload)).to be(false)
  end

  it "revertible? concorda com revert_target em todos os casos" do
    [ v1, v2, baseline_version ].each do |protocol|
      protocol.reload
      expect(described_class.revertible?(protocol)).to eq(described_class.revert_target(protocol).present?)
    end
  end
end
```

Preencha o arranjo com o caminho real (assinaturas + `Protocols::Activate`), sem criar ativação na mão: é o que garante que `kind: "signed"` venha do domínio e não de um `create!` otimista.

- [ ] **Step 3: Rodar e ver falhar**

Da raiz do monorepo:
```bash
docker compose exec -T api bundle exec rspec spec/commands/protocols_revert_target_spec.rb
```
Esperado: FAIL — `undefined method 'revert_target'`.

- [ ] **Step 4: Implementar o domínio**

Em `app/commands/protocols/revert_activation.rb`, troque o corpo de `revertible?` e acrescente o método novo, mantendo o comentário que já explica a ausência de lock:

```ruby
    # A versão que voltaria a valer, ou nil. Leitura pura das MESMAS três
    # condições que `call` exige (spec 2026-09-23-revert-target §3):
    # a versão tem de estar em uso, a ativação corrente tem de ser dela e
    # assinada (linha-base não reverte: não há passo anterior), e a ativação
    # anterior tem de apontar para uma versão ainda `published`.
    #
    # Sem lock: a resposta pode ficar obsoleta assim que outra escrita comita —
    # quem chama sabe disso, e as telas dizem "deve voltar", não "vai voltar".
    def self.revert_target(protocol)
      return nil unless protocol.status == "active"

      latest, previous = activation_history(protocol.name)
      return nil unless latest&.protocol_definition_id == protocol.id && latest.kind == "signed"
      return nil if previous.nil?

      ProtocolDefinition.find_by(id: previous.protocol_definition_id, status: "published")
    end

    # Uma fonte só: quem pergunta "reverteria?" recebe a resposta derivada do
    # ALVO, e não de uma consulta paralela que poderia divergir dele.
    def self.revertible?(protocol)
      revert_target(protocol).present?
    end
```

- [ ] **Step 5: Rodar e ver passar**

```bash
docker compose exec -T api bundle exec rspec spec/commands/protocols_revert_target_spec.rb spec/commands spec/requests/protocol_lifecycle_spec.rb
```
Esperado: PASS. Os specs de reversão que já existem cobrem o `call`; se algum falhar, a delegação mudou comportamento.

- [ ] **Step 6: Expor no painel da cidade**

Em `app/queries/admin/protocols_query.rb`, no `signature_state` (onde `revertible:` é montado hoje), calcule o alvo **uma vez** e derive os dois campos dele — duas consultas por versão custariam o dobro e poderiam divergir:

```ruby
    target = Protocols::RevertActivation.revert_target(d)
```

e, no Hash devolvido, no lugar da linha atual de `revertible:`:

```ruby
      revertible: target.present?,
      # `version` é string em toda esta query (`d.version.to_s`); o alvo segue a
      # mesma forma, para o dashboard não ter dois tipos para o mesmo conceito.
      revertTargetVersion: target&.version&.to_s
```

Acrescente ao `spec/requests/admin/protocols_signatures_spec.rb`, junto do exemplo de `revertible` que já existe: a versão ativa responde `revertTargetVersion` igual à versão anterior (na lista **e** no detalhe), e `null` quando não há alvo.

- [ ] **Step 7: Expor na API de manutenção**

Em `app/graphql/maintenance/types/protocol_version_type.rb`:

```ruby
      field :revert_target_version, Integer, null: true,
            description: "Versão que voltaria a valer numa reversão de emergência; nula quando não há."
```

Em `app/graphql/maintenance/types/city_type.rb`, no Hash montado dentro do `inside` de `protocol_versions`, calcule o alvo uma vez e use nos dois campos:

```ruby
            target = Protocols::RevertActivation.revert_target(d)
            {
              # ... campos existentes ...
              revertible: target.present?,
              revert_target_version: target&.version
            }
```

Inteiro aqui, porque `version` nesse tipo é `Integer`. Acrescente ao `spec/requests/maintenance/city_protocol_versions_spec.rb`: a versão ativa responde `revertTargetVersion` com o número da anterior, e `null` sem alvo.

- [ ] **Step 8: Suíte completa** (worker parado; ver Global Constraints). Esperado: 0 falhas.

- [ ] **Step 9: Commit**

```bash
/opt/homebrew/bin/git add app/commands/protocols/revert_activation.rb app/queries/admin/protocols_query.rb app/graphql/maintenance/types/protocol_version_type.rb app/graphql/maintenance/types/city_type.rb spec/commands/protocols_revert_target_spec.rb spec/requests/admin/protocols_signatures_spec.rb spec/requests/maintenance/city_protocol_versions_spec.rb
/opt/homebrew/bin/git commit -m "feat: expose the revert target version in both read surfaces" -m "Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 2: o painel do dashboard nomeia a versão

**Files:**
- Modify: `apps/dashboard/src/lib/types.ts`, `apps/dashboard/src/modules/Protocols.tsx`
- Test: `apps/dashboard/src/modules/Protocols.test.tsx`

**Interfaces:**
- Consumes: `revertTargetVersion: string | null` em cada versão de `/admin/api/protocols` e `/admin/api/protocols/:id` (Task 1).
- Produces: a `description` do `SensitiveAction` de reverter nomeia a versão.

**Texto:** substitui o `REVERT_DESCRIPTION` atual por uma função do alvo:

> "A cidade deve voltar para a versão N, que estava em uso antes desta. A versão atual sai de uso. Não encadeia: reverter de novo exige uma nova ativação assinada, não outra reversão."

O botão de reverter só existe quando `revertible` é verdadeiro, e depois da Task 1 `revertible` é exatamente `revertTargetVersion != null` — então não há estado "reverter sem alvo" para tratar. Ainda assim, **não** use `!` de não-nulidade: se o campo vier nulo, caia numa frase sem número (a antiga), em vez de imprimir "versão null".

- [ ] **Step 1: Branch**

```bash
cd apps/dashboard && /opt/homebrew/bin/git checkout -b feat/revert-target
```

- [ ] **Step 2: Escrever o teste que falha**

Em `src/modules/Protocols.test.tsx`, o arquivo já tem os helpers `row`, `versionRow` e `stubReads` e um exemplo de reverter — acrescente `revertTargetVersion` aos helpers (default `null`) e um exemplo:

```tsx
  it("o painel de reverter nomeia a versão que deve voltar", async () => {
    stubReads(
      [ row({ status: "active", revertible: true, version: "3", revertTargetVersion: "2" }) ],
      [ versionRow({ status: "active", revertible: true, version: "3", revertTargetVersion: "2" }) ]
    );
    renderProtocols("protocol_publisher");

    fireEvent.click(await screen.findByText("dengue"));
    fireEvent.click(await screen.findByRole("button", { name: "Reverter" }));

    expect(await screen.findByText(/deve voltar para a versão 2/)).not.toBeNull();
    expect(screen.getByText(/Não encadeia/)).not.toBeNull();
  });

  it("sem alvo na leitura, o painel cai na frase sem número", async () => {
    stubReads(
      [ row({ status: "active", revertible: true, version: "3", revertTargetVersion: null }) ],
      [ versionRow({ status: "active", revertible: true, version: "3", revertTargetVersion: null }) ]
    );
    renderProtocols("protocol_publisher");

    fireEvent.click(await screen.findByText("dengue"));
    fireEvent.click(await screen.findByRole("button", { name: "Reverter" }));

    expect(await screen.findByText(/versão ativada antes desta/)).not.toBeNull();
    expect(screen.queryByText(/versão null/)).toBeNull();
  });
```

Confira no arquivo os nomes reais dos helpers e do papel usado em `renderProtocols` antes de escrever; ajuste a chamada, não o que o teste prova.

- [ ] **Step 3: Rodar e ver falhar**

Run: `npx vitest run src/modules/Protocols.test.tsx`
Esperado: FAIL — a frase com o número não existe.

- [ ] **Step 4: Implementar**

Em `src/lib/types.ts`, `SignatureState` ganha:

```ts
  revertTargetVersion: string | null;
```

Em `src/modules/Protocols.tsx`, troque a constante por uma função:

```tsx
// A frase nomeia a versão-alvo (spec 2026-09-23-revert-target §5). "deve
// voltar", não "vai voltar": a leitura é sem lock, e a API decide no clique.
// Sem alvo na leitura, cai na frase sem número em vez de imprimir "null".
function revertDescription(targetVersion: string | null): string {
  const destino = targetVersion
    ? `a versão ${targetVersion}, que estava em uso antes desta`
    : "a versão ativada antes desta";
  return `A cidade deve voltar para ${destino}. A versão atual sai de uso. ` +
    "Não encadeia: reverter de novo exige uma nova ativação assinada, não outra reversão.";
}
```

O `pending` hoje é `{ version: string; action: LifecycleAction }` e o `onPick` do `VersionActions` o monta a partir da versão (`setPending({ version: v.version, action })`). Acrescente o alvo ali:

```tsx
  const [ pending, setPending ] = useState<{ version: string; revertTargetVersion: string | null; action: LifecycleAction } | null>(null);
```

```tsx
                    onPick={(action) => {
                      setPending({ version: v.version, revertTargetVersion: v.revertTargetVersion, action });
                      setDone(null);
                    }}
```

e no `SensitiveAction`:

```tsx
            description={pending.action.kind === "revert" ? revertDescription(pending.revertTargetVersion) : undefined}
```

Se `protocolLifecycle.ts` precisar do campo no tipo `LifecycleTarget`, ele já estende `SignatureState`, então nada muda lá além do typecheck passar.

- [ ] **Step 5: Rodar e ver passar**

```bash
npx vitest run src/modules/Protocols.test.tsx && npm run typecheck && npm test && npm run build
```
Esperado: verde.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git add src/lib/types.ts src/modules/Protocols.tsx src/modules/Protocols.test.tsx
/opt/homebrew/bin/git commit -m "feat: name the revert target in the city panel" -m "Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 3: o painel da manutenção nomeia a versão

**Files:**
- Modify: `apps/maintenance/schema.graphql` e `src/gql/*` (gerados), `apps/maintenance/src/screens/ProtocolsTab.tsx`
- Test: `apps/maintenance/src/screens/ProtocolsTab.test.tsx`

**Interfaces:**
- Consumes: `revertTargetVersion: Int` em `city.protocolVersions` (Task 1).
- Produces: o aviso do painel de reverter nomeia a versão.

**Texto:** substitui o `REVERT_NOTICE` atual ("volta para a versão ativada antes desta, sem assinatura nova") por:

> "deve voltar para a versão N, que estava em uso antes desta; sem assinatura nova, e não encadeia"

Sem alvo, mantém a frase de hoje.

- [ ] **Step 1: Branch e schema**

```bash
cd apps/maintenance && /opt/homebrew/bin/git checkout -b feat/revert-target
```

O `schema:pull` lê do container `api`, que monta `apps/api` — confirme que o api está na branch da Task 1 (`git -C ../api branch --show-current`) e então:

```bash
npm run schema:pull && npm run codegen
grep -n "revertTargetVersion" schema.graphql
```
Esperado: o campo presente em `type ProtocolVersion`.

- [ ] **Step 2: Escrever o teste que falha**

Em `src/screens/ProtocolsTab.test.tsx`, o arquivo já tem o helper `v()` e um exemplo de reverter. Acrescente `revertTargetVersion` ao helper (default `null`) e:

```tsx
  it("o painel de reverter nomeia a versão que deve voltar", async () => {
    route([ v({ status: "active", version: 3, revertible: true, revertTargetVersion: 2 }) ]);
    renderTab();

    await user.click(await screen.findByRole("button", { name: "Reverter" }));

    expect(screen.getByText(/deve voltar para a versão 2/)).not.toBeNull();
  });

  it("sem alvo, mantém o aviso sem número", async () => {
    route([ v({ status: "active", version: 3, revertible: true, revertTargetVersion: null }) ]);
    renderTab();

    await user.click(await screen.findByRole("button", { name: "Reverter" }));

    expect(screen.getByText(/versão ativada antes desta/)).not.toBeNull();
  });
```

Este arquivo usa `fireEvent` ou `user` conforme o padrão dele — copie o que já está lá.

- [ ] **Step 3: Rodar e ver falhar**

Run: `npx vitest run src/screens/ProtocolsTab.test.tsx`
Esperado: FAIL.

- [ ] **Step 4: Implementar**

- a consulta `CityProtocolVersions` pede `revertTargetVersion` junto de `revertible`;
- o tipo `Row` ganha `revertTargetVersion: number | null`;
- `REVERT_NOTICE` vira função, no mesmo desenho do dashboard (sem `!` de não-nulidade, com a frase antiga como queda);
- o `pending` guarda a linha (`row`), então o aviso lê o alvo de `pending.row.revertTargetVersion`.

Rode `npm run codegen` depois de mudar a consulta.

- [ ] **Step 5: Rodar e ver passar**

```bash
npm run codegen && npx vitest run src/screens/ProtocolsTab.test.tsx && npm run typecheck && npm test && npm run build
```
Esperado: verde.

- [ ] **Step 6: Verificação no navegador** (opcional, se o stack estiver de pé)

Com `docker compose up -d api worker dashboard maintenance`, abra a aba Protocolos de uma cidade que tenha versão ativa com ativação anterior assinada e confira que o painel de reverter mostra o número. Em curitiba de dev há histórico de ativações da verificação da fatia 3; se não houver alvo, diga isso em vez de forçar dado.

- [ ] **Step 7: Commit**

```bash
/opt/homebrew/bin/git add schema.graphql src/gql src/screens/ProtocolsTab.tsx src/screens/ProtocolsTab.test.tsx
/opt/homebrew/bin/git commit -m "feat: name the revert target in the maintenance tab" -m "Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

## Fora deste plano

- Mostrar quem ativou a versão-alvo, ou quando.
- Linha do tempo de ativações na tela.
- Qualquer mudança em `Protocols::RevertActivation.call`.
- Aviso por e-mail de reversão.
