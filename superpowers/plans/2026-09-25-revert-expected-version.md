# Versão esperada na reversão — plano

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** a reversão de emergência passa a recusar quando a versão vigente mudou entre o que a tela mostrou e o clique chegar — em vez de reverter outra coisa com sucesso.

**Architecture:** o cliente manda a versão vigente que estava na tela; `Protocols::RevertActivation.call` a compara com a vigente real na leitura rápida e recusa com um motivo próprio. Uma conferência só: a que viria depois, sob lock, seria código morto (ver §3.3 do spec). As duas superfícies do api ganham o argumento como **opcional** neste passo, e as duas telas passam a mandá-lo.

**Tech Stack:** Rails 8.1 + graphql-ruby 2.6 (`apps/api`); Vite + React 18 + React Query 5 + Vitest (`apps/dashboard`, `apps/maintenance`).

**Spec:** `docs/superpowers/specs/2026-09-25-revert-expected-version-design.md` (commit `db31e9a`).

## Global Constraints

- Commits: Conventional Commits **em inglês**, **um** tipo por assunto, terminando com `Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>`.
- Git: `/opt/homebrew/bin/git`. Branch `feat/revert-expected-version` em **cada** repo tocado. Nunca em `main`, nunca push, nunca merge.
- Staging explícito por arquivo; nunca `git add -A`.
- Suíte completa do api: na raiz do monorepo, `docker compose stop worker`, depois `docker compose exec -T api bundle exec rspec`, depois `docker compose start worker` (religar SEMPRE). Acima de ~3,5 min é regressão.
- **Se o `apps/api` tiver acabado de sair de outra branch, rode `bin/rails city:test_databases` antes da suíte** — os bancos de teste de cidade são compartilhados entre branches, e um schema à frente quebra `spec/services/city_schema_spec.rb` sem relação com o que você mudou.
- **Sem token, o comportamento é byte por byte o de hoje.** É o que permite os três passos do §5 do spec.
- **Uma conferência só**, na leitura rápida. Não acrescente uma segunda dentro da transação: `current` é a mesma linha travada, e a checagem `unless current.status == "active"` que já existe ali pega tudo o que a segunda pegaria (§3.3 do spec).
- Só o **número** da versão viaja. Nada de definição ou conteúdo de protocolo.
- **Nunca `undefined` na tela.** Sem número na recusa, a frase sai sem número.
- Specs de request exigem `type: :request`; consultas GraphQL em specs ficam em métodos ou `let`, nunca em constante de topo.
- Nunca rode `start.sh`.

## Review Focus

1. **Sem token, nada muda** — é o que sustenta o rollout em três passos. Teste na Tarefa 1.
2. **O token é conferido contra a vigente REAL, não contra a que o cliente mandou** — um teste que monte os dois números iguais não prova nada. Teste na Tarefa 1.
3. **A recusa por divergência é distinguível das outras recusas** — a tela precisa saber que deve reler a lista, e não repetir a frase genérica. Testes nas Tarefas 2, 3 e 4.
4. **A tela não imprime `undefined`** quando a recusa não traz número. Testes nas Tarefas 3 e 4.
5. **O argumento do GraphQL é anulável neste passo** — se nascer não-anulável, o console antigo quebra na validação no instante do deploy do api. Teste na Tarefa 2.

## File Structure

**apps/api**
- Modify: `app/commands/protocols/revert_activation.rb` — o argumento, a conferência e a recusa.
- Modify: `app/controllers/protocol_lifecycle_controller.rb` — lê `expected_version` do corpo.
- Modify: `app/controllers/concerns/protocol_result_rendering.rb` — `current_version_changed: :conflict` no `STATUS_FOR`.
- Modify: `app/graphql/maintenance/mutations/revert_protocol_activation.rb` — `expectedVersion: Int`, anulável.
- Test: `spec/commands/protocols_revert_activation_spec.rb`, `spec/requests/protocol_lifecycle_spec.rb`, `spec/requests/maintenance/protocol_mutations_spec.rb`.

**apps/dashboard**
- Modify: `src/lib/api.ts` — `revertProtocol` manda o token e distingue a recusa.
- Modify: `src/modules/Protocols.tsx` — a frase da recusa e a releitura.
- Test: `src/lib/api.test.ts`, `src/modules/Protocols.test.tsx`.

**apps/maintenance**
- Modify: `src/screens/ProtocolsTab.tsx` — o argumento na mutation e a frase da recusa.
- Test: `src/screens/ProtocolsTab.test.tsx`.
- Gerados: `schema.graphql`, `src/gql/*`.

---

### Task 1: `expected_version:` no command

**Files:**
- Modify: `apps/api/app/commands/protocols/revert_activation.rb`
- Test: `apps/api/spec/commands/protocols_revert_activation_spec.rb`

**Interfaces:**
- Produces:
  ```ruby
  Protocols::RevertActivation.call(name:, by:, reason:, expected_version: nil, correlation_id: nil)
  # divergência -> Result.fail(:current_version_changed, message: "a versão em uso agora é a N")
  ```

- [ ] **Step 1: Escreva os testes que falham**

Em `apps/api/spec/commands/protocols_revert_activation_spec.rb`. O arquivo já tem os helpers de que você precisa: `activate_signed!(v)` (cria a versão se faltar, coleta as duas assinaturas de ativação e ativa pelo caminho real, avançando 1 segundo), `version(v)` e `revert(reason:, by:)`. Use-os; montar `ProtocolActivation` na mão produziria um estado que o domínio não produz.

`revert` não aceita o token, então os exemplos novos chamam `described_class.call` direto — ou, se preferir, acrescente um parâmetro opcional ao helper. O plano usa a chamada direta para não mexer nos exemplos que já existem.

```ruby
  describe "expected_version" do
    it "reverts when the token matches the active version" do
      activate_signed!(1)
      activate_signed!(2)

      result = described_class.call(name: "dengue", by: publisher, reason: "motivo", expected_version: 2)

      expect(result.ok?).to be(true)
      expect(version(1).status).to eq("active")
    end

    # A corrida que o token existe para pegar: a tela mostrava a v2 como
    # vigente, alguém ativou a v3, e só então o clique chegou.
    it "refuses when the active version changed since the screen read it" do
      activate_signed!(1)
      activate_signed!(2)
      activate_signed!(3)

      result = described_class.call(name: "dengue", by: publisher, reason: "motivo", expected_version: 2)

      expect(result.failure?).to be(true)
      expect(result.reason).to eq(:current_version_changed)
      expect(result.message).to include("3")
      expect(version(3).status).to eq("active")
    end

    # Review Focus 1: é o que permite o rollout em três passos.
    it "reverts exactly as before when no token is sent" do
      activate_signed!(1)
      activate_signed!(2)

      expect(revert.ok?).to be(true)
      expect(version(1).status).to eq("active")
    end
  end
```

- [ ] **Step 2: Rode e confirme que falham**

```bash
docker compose exec -T api bundle exec rspec spec/commands/protocols_revert_activation_spec.rb -e "expected_version"
```

Esperado: os dois primeiros falham com `ArgumentError: unknown keyword: :expected_version` — o argumento ainda não existe. O terceiro passa, porque não manda token: ele existe para prender o comportamento que **não** pode mudar. Confirme que a falha é o `ArgumentError`, e não outra coisa.

- [ ] **Step 3: Implemente**

Em `apps/api/app/commands/protocols/revert_activation.rb`, a assinatura e a conferência. A conferência vai **logo depois** de `current` ser encontrado e **antes** da política, para que um token errado não vire `:forbidden` por acaso:

```ruby
    def self.call(name:, by:, reason:, expected_version: nil, correlation_id: nil)
      return Result.fail(:city_missing) if Current.city.nil?
      return Result.fail(:reason_required, message: "a reversão de emergência exige um motivo") if reason.to_s.strip.empty?

      current = ProtocolDefinition.find_by(name: name, status: "active")
      return Result.fail(:not_found) if current.nil?

      # O que a TELA via quando a pessoa decidiu. `current` é o que vale agora.
      # Divergiram: outra ativação comitou entre a leitura e o clique, e a
      # decisão foi tomada sobre informação que não vale mais.
      #
      # Uma conferência só, e de propósito: repetir isto dentro da transação
      # seria código morto. `current` é esta mesma linha, travada e não
      # re-resolvida — a `version` dela não muda —, e se outra versão tiver
      # sido ativada no meio do caminho, `Protocols::Activate` demove esta
      # para `published` antes (o índice único parcial em status='active' não
      # admite duas), de modo que o `unless current.status == "active"` que já
      # existe lá embaixo dispara primeiro.
      #
      # `expected_version` é opcional porque o rollout tem três passos (spec
      # 2026-09-25 §5): o api aceita, os clientes passam a mandar, e só então
      # a ausência vira recusa. O GraphQL não deixa o frontend ir na frente —
      # argumento não declarado é erro de validação —, por isso o api vem
      # primeiro. Enquanto o terceiro passo não vier, existe um caminho sem
      # guarda, e ele é deliberado.
      if expected_version.present? && Integer(expected_version) != current.version
        return Result.fail(:current_version_changed,
                           message: "a versão em uso agora é a #{current.version}")
      end

      return Result.fail(:forbidden) unless ProtocolPolicy.new(by, current).activate?
```

O resto do método fica intocado.

- [ ] **Step 4: Rode os testes e confirme que passam**

```bash
docker compose exec -T api bundle exec rspec spec/commands/protocols_revert_activation_spec.rb
```

Esperado: PASS, incluindo os exemplos que já existiam (que não mandam token).

- [ ] **Step 5: Suíte completa e commit**

```bash
docker compose stop worker
docker compose exec -T api bundle exec rspec
docker compose start worker
```

Esperado: 0 falhas. Depois:

```bash
cd apps/api
/opt/homebrew/bin/git add app/commands/protocols/revert_activation.rb spec/commands/protocols_revert_activation_spec.rb
/opt/homebrew/bin/git commit
```

Assunto sugerido: `feat: refuse a revert whose expected version is no longer the active one`. Corpo: a corrida que isso pega, e por que a conferência é uma só.

---

### Task 2: as duas superfícies do api

**Files:**
- Modify: `apps/api/app/controllers/protocol_lifecycle_controller.rb`
- Modify: `apps/api/app/controllers/concerns/protocol_result_rendering.rb`
- Modify: `apps/api/app/graphql/maintenance/mutations/revert_protocol_activation.rb`
- Test: `apps/api/spec/requests/protocol_lifecycle_spec.rb`, `apps/api/spec/requests/maintenance/protocol_mutations_spec.rb`, `apps/api/spec/architecture/maintenance_schema_spec.rb`

**Interfaces:**
- Consumes: `Protocols::RevertActivation.call(..., expected_version:)` e a recusa `:current_version_changed`, da Tarefa 1.
- Produces:
  ```
  POST /admin/api/protocols/revert  { name, reason, expected_version }  -> 409 { "error": "current_version_changed", "message": "..." }
  ```
  ```graphql
  revertProtocolActivation(citySlug: String!, name: String!, reason: String!, code: String!, expectedVersion: Int): RevertProtocolActivationPayload
  ```

- [ ] **Step 1: Escreva os testes que falham**

Em `apps/api/spec/requests/protocol_lifecycle_spec.rb`, junto dos exemplos de reversão que já existem:

```ruby
    it "responds 409 when the expected version is no longer the active one" do
      # Mesmo arranjo do exemplo de reversão feliz deste arquivo (o que faz
      # `post "/protocols/revert"` terminar com `statuses` voltando a
      # `%w[active published ...]`), com o token apontando para uma versão que
      # não é a vigente. As rotas aqui são montadas sem o prefixo /admin/api —
      # siga o que o arquivo já usa.
      sign_in_stepped_up!(admin)
      post "/protocols/revert",
           params: { name: "dengue", reason: "v2 erra a prioridade", expected_version: "99" }, as: :json

      expect(response).to have_http_status(:conflict)
      expect(json["error"]).to eq("current_version_changed")
    end
```

Em `apps/api/spec/requests/maintenance/protocol_mutations_spec.rb`, dentro do `describe "revertProtocolActivation"`. Acrescente `$expectedVersion: Int` à consulta compartilhada `revert_mutation` e um parâmetro opcional ao helper `revert!`, para os exemplos existentes seguirem passando sem mudança:

```ruby
      it "refuses when the expected version is no longer the active one, pointing at the argument" do
        legacy_active_version!
        publish_and_activate_v2!

        with_fresh_totp { |code| revert!(reason: "motivo qualquer", code: code, expected_version: 99) }

        expect(revert_payload["ok"]).to be(false)
        expect(revert_payload["errors"].map { |e| e["path"] }).to eq([ "expectedVersion" ])
      end

      # Review Focus 5: não-anulável quebraria o console antigo no deploy.
      it "accepts the mutation without the argument, which is what keeps the old console working" do
        legacy_active_version!
        publish_and_activate_v2!

        with_fresh_totp { |code| revert!(reason: "motivo qualquer", code: code) }

        expect(revert_payload["ok"]).to be(true)
      end
```

- [ ] **Step 2: Rode e confirme que falham**

```bash
docker compose exec -T api bundle exec rspec spec/requests/protocol_lifecycle_spec.rb spec/requests/maintenance/protocol_mutations_spec.rb
```

Esperado: o de 409 falha com 422 (o motivo novo ainda cai no default do `STATUS_FOR`); o do GraphQL falha na validação, porque o argumento não existe.

- [ ] **Step 3: O corpo REST e o status**

Em `app/controllers/protocol_lifecycle_controller.rb`:

```ruby
  def revert
    render_protocol_result(Protocols::RevertActivation.call(name: protocol_name,
                                                            reason: optional_scalar_param(:reason).to_s,
                                                            expected_version: optional_scalar_param(:expected_version),
                                                            by: Current.user))
  end
```

`optional_scalar_param` é o idioma deste controller para escalar opcional: ele recusa array e hash, que é o que impede um corpo hostil de virar comparação estranha.

Em `app/controllers/concerns/protocol_result_rendering.rb`:

```ruby
  # 409 e não 422: o corpo estava bem formado; o mundo é que mudou entre a
  # leitura da tela e o clique. A tela distingue os dois para saber quando
  # deve reler a lista em vez de repetir a frase genérica.
  STATUS_FOR = { forbidden: :forbidden, not_found: :not_found, current_version_changed: :conflict }.freeze
```

- [ ] **Step 4: O argumento do GraphQL**

Em `app/graphql/maintenance/mutations/revert_protocol_activation.rb`:

```ruby
      # Anulável NESTE passo, de propósito: um argumento não-anulável faria o
      # console em produção — que ainda não manda nada — quebrar na validação
      # no instante em que este api subisse. Vira obrigatório no terceiro
      # passo do rollout (spec 2026-09-25 §5), com issue própria.
      argument :expected_version, Integer, required: false,
               description: "A versão que a tela via como vigente. Divergiu, a reversão é recusada."
```

E no `resolve`, repasse ao command e mande a recusa para o caminho do argumento:

```ruby
      def resolve(city_slug:, name:, reason:, code:, expected_version: nil)
        in_city(city_slug: city_slug, step_up_code: code, event: "maintenance.protocol.reverted", module_name: "protocol",
                rejection_path: lambda { |result|
                  case result.reason
                  when :reason_required then "reason"
                  when :current_version_changed then "expectedVersion"
                  else "name"
                  end
                },
                protocol_key: name, reason_given: !reason.to_s.strip.empty?,
                changed_fields: [ "status" ],
                payload_from_result: ->(r) { { reverted_to_version: r.payload[:protocol_definition].version } }) do |actor, correlation_id|
          Protocols::RevertActivation.call(name: name, reason: reason, by: actor,
                                           expected_version: expected_version, correlation_id: correlation_id)
        end
      end
```

- [ ] **Step 5: Guarda de schema**

```bash
docker compose exec -T api bundle exec rspec spec/architecture/maintenance_schema_spec.rb
```

A guarda de allowlist cobre campos de payload e pode cobrir argumentos. Se falhar, declare `expectedVersion` onde ela pedir — **declare, nunca afrouxe**.

- [ ] **Step 6: Rode os testes, a suíte completa e commit**

```bash
docker compose exec -T api bundle exec rspec spec/requests/protocol_lifecycle_spec.rb spec/requests/maintenance/protocol_mutations_spec.rb
docker compose stop worker && docker compose exec -T api bundle exec rspec; docker compose start worker
```

```bash
cd apps/api
/opt/homebrew/bin/git add app/controllers/protocol_lifecycle_controller.rb \
  app/controllers/concerns/protocol_result_rendering.rb \
  app/graphql/maintenance/mutations/revert_protocol_activation.rb \
  spec/requests/protocol_lifecycle_spec.rb spec/requests/maintenance/protocol_mutations_spec.rb
/opt/homebrew/bin/git commit
```

Assunto sugerido: `feat: accept the expected version on both revert surfaces`. Inclua `spec/architecture/maintenance_schema_spec.rb` no `git add` se o Step 5 o tiver mudado.

---

### Task 3: o dashboard manda o token e trata a recusa

**Files:**
- Modify: `apps/dashboard/src/lib/api.ts`
- Modify: `apps/dashboard/src/modules/Protocols.tsx`
- Test: `apps/dashboard/src/lib/api.test.ts`, `apps/dashboard/src/modules/Protocols.test.tsx`

**Interfaces:**
- Consumes: `POST /admin/api/protocols/revert` aceitando `expected_version` e respondendo **409** `{"error": "current_version_changed", "message": "a versão em uso agora é a N"}`.

- [ ] **Step 1: Escreva os testes que falham**

Em `src/lib/api.test.ts`, que usa `mockFetch(status, body)`:

```ts
  it("revertProtocol manda a versão esperada no corpo", async () => {
    mockFetch(200, { ok: true, protocol: { name: "dengue", version: 1, status: "active" } });
    await revertProtocol("dengue", "x", "3");
    expect(lastCall().body).toEqual({ name: "dengue", reason: "x", expected_version: "3" });
  });
```

Em `src/modules/Protocols.test.tsx`, o comportamento de tela. O módulo mocka `../lib/api` inteiro, então a recusa se simula com `mockRejectedValue(new ApiError(...))` — veja o exemplo `"a recusa da API aparece e o painel continua aberto"`, que já usa `ApiError`:

```tsx
  it("recusa por versão mudada: mostra o estado novo e não repete a frase genérica", async () => {
    stubReads(
      [ row({ status: "active", revertible: true, version: "3", revertTargetVersion: "2" }) ],
      [ versionRow({ status: "active", revertible: true, version: "3", revertTargetVersion: "2" }) ]
    );
    mocked(api.revertProtocol).mockRejectedValue(
      new ApiError(409, { error: "current_version_changed", message: "a versão em uso agora é a 5" }, "409")
    );
    renderProtocols("protocol_publisher");

    fireEvent.click(await screen.findByText("dengue"));
    fireEvent.click(await screen.findByRole("button", { name: "Reverter" }));
    fireEvent.change(screen.getByLabelText("Motivo"), { target: { value: "regra errada" } });
    fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));

    const aviso = await screen.findByText(/a versão em uso mudou/);
    expect(aviso.textContent).toContain("5");
    expect(aviso.textContent).not.toContain("undefined");
  });
```

- [ ] **Step 2: Rode e confirme que falham**

```bash
cd apps/dashboard && npx vitest run src/lib/api.test.ts src/modules/Protocols.test.tsx
```

- [ ] **Step 3: O cliente manda o token**

Em `src/lib/api.ts`, `revertProtocol` ganha o terceiro parâmetro:

```ts
export async function revertProtocol(
  name: string, reason: string, expectedVersion?: string
): Promise<{ version: string } | null> {
  const body = await jsonFetch<{ protocol?: { version?: string | number } }>(`${PROTOCOLS_BASE}/revert`, {
    method: "POST",
    body: JSON.stringify({ name, reason, ...(expectedVersion == null ? {} : { expected_version: expectedVersion }) })
  });
  const version = body?.protocol?.version;
  return version == null ? null : { version: String(version) };
}
```

Opcional na assinatura porque as chamadas existentes (e os testes que já existem) não o mandam.

- [ ] **Step 4: A tela manda e trata**

Em `src/modules/Protocols.tsx`, `runAction` passa a versão vigente que estava na tela:

```tsx
    case "revert":   return (await revertProtocol(name, values.reason ?? "", version))?.version ?? null;
```

`version` é o parâmetro que `runAction` já recebe — a versão da linha sobre a qual se agiu, que na reversão é exatamente a vigente que a tela mostrava.

O `SensitiveAction` já mostra `describeActionError(err)` na recusa. Essa função mora em `src/lib/actionErrors.ts`, devolve uma união discriminada (`ActionError`) e **lê o código do corpo**, não do erro: `ApiError` tem só `status`, `body` e `message`, e o código sai de `bodyField(err.body, "error")`. Acrescente o caso junto dos outros, reaproveitando o `kind: "rejected"` que já carrega `code` e `message`:

```ts
  // 409 da reversão: não é "tente de novo", é "o mundo mudou entre a leitura
  // e o clique". A mensagem do servidor nomeia a versão que está em uso
  // agora, e sai verbatim — é ela que deixa a pessoa decidir com o dado
  // certo. O `code` viaja para a tela saber que deve reler a lista.
  if (err.status === 409 && code === "current_version_changed") {
    return { kind: "rejected", code, message: bodyField(err.body, "message") ?? GENERIC_STALE };
  }
```

com, no topo do arquivo, ao lado do `GENERIC` que já existe:

```ts
const GENERIC_STALE = "a versão em uso mudou enquanto você lia. Confira e decida de novo.";
```

Sem mensagem do servidor a frase sai **sem número** — nunca `undefined`.

E a releitura: no `onError` do `SensitiveAction` (ou no ponto equivalente do módulo), invalide `["protocols"]` e `["protocol-detail", id]` quando o erro for esse, para a lista mostrar o estado novo.

- [ ] **Step 5: Rode os testes e a verificação completa**

```bash
cd apps/dashboard && npx vitest run src/lib/api.test.ts src/modules/Protocols.test.tsx
npm run typecheck && npm test && npm run build
```

- [ ] **Step 6: Commit**

```bash
cd apps/dashboard
/opt/homebrew/bin/git add src/lib/api.ts src/lib/api.test.ts src/modules/Protocols.tsx src/modules/Protocols.test.tsx
/opt/homebrew/bin/git commit
```

Assunto sugerido: `feat: send the expected version when reverting from the city panel`.

---

### Task 4: o console de manutenção manda o token e trata a recusa

**Files:**
- Modify: `apps/maintenance/src/screens/ProtocolsTab.tsx`
- Test: `apps/maintenance/src/screens/ProtocolsTab.test.tsx`
- Gerados: `apps/maintenance/schema.graphql`, `apps/maintenance/src/gql/*`

**Interfaces:**
- Consumes: `expectedVersion: Int` (anulável) na mutation, e a recusa com `path: "expectedVersion"`.
- **Depende da Tarefa 2 estar na branch do api**, porque o `schema:pull` lê o schema do api em execução.

- [ ] **Step 1: Atualize schema e tipos**

Com o `apps/api` rodando na branch da Tarefa 2:

```bash
cd apps/maintenance && npm run schema:pull && npm run codegen
grep -n "expectedVersion" schema.graphql
```

Se não aparecer, `docker compose restart api` na raiz e tente de novo. Se ainda assim não aparecer, **pare e reporte** — não escreva à mão em arquivo gerado.

- [ ] **Step 2: Escreva o teste que falha**

Em `src/screens/ProtocolsTab.test.tsx`, usando `route`, `v`, `mutationReply` e `renderTab` que já existem:

```tsx
  it("recusa por versão mudada: mostra o estado novo em vez da frase genérica", async () => {
    const user = userEvent.setup();
    route([ v({ status: "active", version: 3, revertible: true, revertTargetVersion: 2 }) ], {
      revertProtocolActivation: () => mutationReply("revertProtocolActivation", false, [
        { path: "expectedVersion", message: "a versão em uso agora é a 5" }
      ])
    });
    renderTab();

    await user.click(await screen.findByRole("button", { name: "Reverter" }));
    await user.type(screen.getByLabelText("Motivo"), "regra errada em produção");
    await user.type(screen.getByLabelText("Código do autenticador"), "654321");
    await user.click(screen.getByRole("button", { name: "Confirmar" }));

    const erro = await screen.findByText(/a versão em uso agora é a 5/);
    expect(erro.textContent).not.toContain("undefined");
  });

  it("manda a versão vigente da linha como expectedVersion", async () => {
    const user = userEvent.setup();
    route([ v({ status: "active", version: 3, revertible: true, revertTargetVersion: 2 }) ], {
      revertProtocolActivation: () => mutationReply("revertProtocolActivation", true, [], { revertedToVersion: 2 })
    });
    renderTab();

    await user.click(await screen.findByRole("button", { name: "Reverter" }));
    await user.type(screen.getByLabelText("Motivo"), "regra errada em produção");
    await user.type(screen.getByLabelText("Código do autenticador"), "654321");
    await user.click(screen.getByRole("button", { name: "Confirmar" }));

    await waitFor(() => expect(calls(fetchMock, "revertProtocolActivation")).toHaveLength(1));
    expect(bodyOf(calls(fetchMock, "revertProtocolActivation")[0]).variables.expectedVersion).toBe(3);
  });
```

- [ ] **Step 3: Rode e confirme que falham**

```bash
cd apps/maintenance && npx vitest run src/screens/ProtocolsTab.test.tsx
```

- [ ] **Step 4: Implemente**

A consulta ganha o argumento:

```ts
const RevertMutation = graphql(`
  mutation RevertProtocolActivation($citySlug: String!, $name: String!, $reason: String!, $code: String!, $expectedVersion: Int) {
    revertProtocolActivation(citySlug: $citySlug, name: $name, reason: $reason, code: $code, expectedVersion: $expectedVersion) {
      ok
      revertedToVersion
      errors { path message }
    }
  }
`);
```

E o `run` manda a versão da linha:

```ts
    case "revert": {
      const r = await gql(RevertMutation, { citySlug: slug, name: row.name, reason, code, expectedVersion: row.version });
      return { payload: r.data?.revertProtocolActivation ?? null, fieldErrors: r.fieldErrors };
    }
```

Rode `npm run codegen` de novo para os tipos acompanharem.

O `onSuccess` já mostra `${refusal.code} — ${refusal.message}` quando `payload === null`; **esta recusa vem com `payload.ok === false` e o erro em `payload.errors`**, que é o caminho já tratado logo abaixo no mesmo `onSuccess`. Confirme lendo o arquivo qual ramo recebe a recusa e garanta que a mensagem do servidor aparece inteira — é ela que carrega o número novo.

- [ ] **Step 5: Verificação completa e commit**

```bash
cd apps/maintenance && npm run typecheck && npm test && npm run build
```

```bash
/opt/homebrew/bin/git add src/screens/ProtocolsTab.tsx src/screens/ProtocolsTab.test.tsx schema.graphql src/gql
/opt/homebrew/bin/git commit
```

Assunto sugerido: `feat: send the expected version when reverting from the maintenance tab`.

---

## Depois do plano

Abra a issue do **passo 3** do §5 do spec no Project #1 (rotasaude) no mesmo dia em que a Tarefa 2 for mergeada: *tornar `expected_version` obrigatório nas duas superfícies, depois que dashboard e console estiverem em produção mandando-o*. Enquanto ela não for feita, existe um caminho de reversão sem guarda — deliberado, e registrado.
