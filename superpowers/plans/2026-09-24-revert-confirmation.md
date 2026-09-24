# Confirmar a versão efetivada na reversão — plano

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** depois de uma reversão de emergência, as duas telas afirmam qual versão a cidade está usando — e, quando ela difere da que o painel previu, dizem as duas coisas.

**Architecture:** o número já existe no domínio (`Protocols::RevertActivation.call` devolve `Result.ok(protocol_definition:)`). O painel da cidade já o recebe pela API REST e o descarta no cliente. A API de manutenção não tem por onde devolvê-lo — o wrapper `audited` descarta o `Result` —, então `in_city` ganha um mapa opcional de `Result` para chaves extras do payload, e a mutation da reversão expõe `revertedToVersion`. A comparação com o previsto acontece no cliente, que é quem tinha a previsão na mão.

**Tech Stack:** Rails 8.1 + graphql-ruby 2.6 (`apps/api`); Vite + React 18 + React Query 5 + Vitest (`apps/dashboard`, `apps/maintenance`).

**Spec:** `docs/superpowers/specs/2026-09-24-revert-confirmation-design.md` (commit `e3c84cd`).

## Global Constraints

- Commits: Conventional Commits **em inglês**, **um** tipo por assunto, terminando com `Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>`.
- Git: `/opt/homebrew/bin/git` (o `git` do PATH está quebrado neste Mac). Branch `feat/revert-confirmation` em **cada** repo tocado. Nunca em `main`, nunca push, nunca merge.
- Staging explícito por arquivo; nunca `git add -A` (`apps/dashboard` e `apps/maintenance` têm `.superpowers/` fora do git).
- Suíte completa do api: na raiz do monorepo, `docker compose stop worker`, depois `docker compose exec -T api bundle exec rspec`, depois `docker compose start worker` (religar SEMPRE, mesmo se a suíte falhar). Acima de ~3,5 min é regressão.
- **`Protocols::RevertActivation.call` fica intocado.** Ele trava as linhas e reresolve o par sob lock; é ele quem decide.
- **Nada de consultar o banco de novo depois do ato** para descobrir o que valeu. O que a tela afirma é o que o command devolveu.
- **Só o número da versão** chega às telas. Nada de definição ou conteúdo de protocolo.
- `revertedToVersion` é **Int** no tipo de manutenção (como `version` já é ali), nulo quando não houve reversão.
- **Nunca `undefined` na tela.** Sem número, a frase sai sem número.
- Specs de request exigem `type: :request` (o projeto desliga `infer_spec_type_from_file_location!`); consultas GraphQL em specs ficam em métodos ou `let`, nunca em constante de topo.
- Nunca rode `start.sh`.

## Review Focus

Coisas que o spec implica e que nenhuma tarefa exercitaria sem um teste dedicado. Cada linha tem o teste apontado na tarefa que a possui.

1. **Uma mutation de cidade que NÃO usa o mecanismo novo continua respondendo exatamente `{ok, errors}`** — abrir o caminho não pode mudar publish/activate/retire. Teste na Tarefa 1.
2. **Recusa não chama o mapa** — numa recusa não existe `Result` de sucesso; chamar o mapa ali levantaria erro dentro do wrapper de auditoria. Teste na Tarefa 1.
3. **A tela sem número não imprime `undefined`** — foi o modo de falha que a revisão final do spec anterior pegou. Teste nas Tarefas 2 e 3.
4. **O teste da divergência precisa de previsão e resultado independentes** — se o exemplo usar o mesmo número nos dois, passa sem provar nada. Testes nas Tarefas 2 e 3.
5. **As outras ações do ciclo continuam com a frase de hoje** — publicar/ativar/aposentar nomeiam a versão sobre a qual se agiu, e isso está certo. Testes nas Tarefas 2 e 3.

## File Structure

**apps/api**
- Modify: `app/graphql/maintenance/mutations/city_mutation.rb` — o parâmetro `payload:` no `in_city`.
- Modify: `app/graphql/maintenance/mutations/revert_protocol_activation.rb` — o campo e o mapa.
- Modify: `spec/requests/maintenance/protocol_mutations_spec.rb` — o campo na reversão, e a garantia de que as outras mutations não mudaram.

**apps/dashboard**
- Modify: `src/lib/api.ts` — `revertProtocol` devolve o protocolo em vez de descartá-lo.
- Modify: `src/modules/Protocols.tsx` — a frase de sucesso da reversão.
- Modify: `src/modules/Protocols.test.tsx`.

**apps/maintenance**
- Modify: `src/screens/ProtocolsTab.tsx` — o campo na query da mutation e a frase de sucesso.
- Modify: `src/screens/ProtocolsTab.test.tsx`.
- Gerados por `npm run schema:pull && npm run codegen`: `schema.graphql`, `src/gql/*`.

---

### Task 1: `payload:` no `in_city` e `revertedToVersion` na mutation

**Files:**
- Modify: `apps/api/app/graphql/maintenance/mutations/city_mutation.rb`
- Modify: `apps/api/app/graphql/maintenance/mutations/revert_protocol_activation.rb`
- Test: `apps/api/spec/requests/maintenance/protocol_mutations_spec.rb`

**Interfaces:**
- Consumes: `Protocols::RevertActivation.call(name:, by:, reason:, correlation_id:)`, que devolve `Result.ok(protocol_definition:)` — um `ProtocolDefinition` com `version` (Integer).
- Produces:
  ```graphql
  revertProtocolActivation(...): RevertProtocolActivationPayload
  # o payload ganha, além de ok e errors:
  revertedToVersion: Int
  ```
  ```ruby
  # Maintenance::CityMutation#in_city ganha o argumento nomeado opcional:
  payload: ->(result) { { reverted_to_version: result.payload[:protocol_definition].version } }
  ```

- [ ] **Step 1: Escreva os testes que falham**

Em `apps/api/spec/requests/maintenance/protocol_mutations_spec.rb`, dentro do `describe "revertProtocolActivation"` que já existe. O arquivo já tem tudo de que você precisa: `legacy_active_version!` (cria a v1 ativa por `baseline`), `publish_and_activate_v2!` (leva a v2 a ativa pelas mutations, com os dois step-up), `with_fresh_totp`, `revert!`, `revert_payload`, `publish_payload` e `version_row`.

Primeiro, peça o campo novo na consulta compartilhada `revert_mutation` (por volta da linha 507):

```ruby
    def revert_mutation
      <<~GQL
        mutation($citySlug: String!, $name: String!, $reason: String!, $code: String!) {
          revertProtocolActivation(citySlug: $citySlug, name: $name, reason: $reason, code: $code) {
            ok
            revertedToVersion
            errors { path message }
          }
        }
      GQL
    end
```

**Consequência que não é opcional:** dois exemplos que já existem afirmam o payload inteiro com `eq`, assim:

```ruby
        expect(revert_payload).to eq("ok" => true, "errors" => [])
```

Com o campo novo na consulta, o payload passa a ter três chaves e esses `eq` quebram. Atualize os dois para incluir `"revertedToVersion" => 1` — não troque `eq` por `include`, porque é justamente o `eq` que garante que nenhuma chave a mais apareceu sem alguém decidir.

Os três exemplos novos:

```ruby
      it "responds with the version that took effect" do
        legacy_active_version!
        publish_and_activate_v2!

        with_fresh_totp { |code| revert!(reason: "motivo qualquer", code: code) }

        expect(revert_payload).to eq("ok" => true, "errors" => [], "revertedToVersion" => 1)
        expect(version_row(1).status).to eq("active")
      end

      # Recusa não tem Result de sucesso para ler: o mapa não pode ser chamado,
      # e o campo não pode inventar número (Review Focus 2).
      it "does not invent a version when the revert is refused" do
        legacy_active_version!

        with_fresh_totp { |code| revert!(reason: "motivo qualquer", code: code) }

        expect(revert_payload["ok"]).to be(false)
        expect(revert_payload["revertedToVersion"]).to be_nil
      end

      # Abrir o caminho não pode mudar quem não o usa (Review Focus 1).
      it "leaves the payload of a mutation that does not map its result untouched" do
        legacy_active_version!
        save_draft!(protocol_definition_hash(version: 2))
        submit!(version: 2)
        sign_two_reviewers!(version_row(2), purpose: "publication")
        with_fresh_totp { |code| publish!(version: 2, code: code) }

        expect(publish_payload.keys).to contain_exactly("ok", "errors")
      end
```

No segundo exemplo a cidade está ativa apenas pela `baseline` da v1, e `RevertActivation` exige que a ativação corrente seja `signed` — então ele exercita a recusa sem precisar montar um estado artificial.

- [ ] **Step 2: Rode e confirme que falham**

Na raiz do monorepo:

```bash
docker compose exec -T api bundle exec rspec spec/requests/maintenance/protocol_mutations_spec.rb
```

Esperado: falha em `revertedToVersion` — o campo não existe no schema, então o graphql-ruby recusa a consulta inteira.

- [ ] **Step 3: O parâmetro no `in_city`**

Em `apps/api/app/graphql/maintenance/mutations/city_mutation.rb`, o método `in_city` hoje termina assim (o `audited` devolve `{ ok: true, errors: [] }` e o `result` do bloco é descartado):

```ruby
      def in_city(city_slug:, event:, module_name:, rejection_path: "version", field_paths: {}, step_up_code: nil,
                  **fields)
```

Acrescente `payload: nil` à assinatura, guarde o `Result` num local como o `correlation_id` já é guardado, e mescle as chaves depois que o `audited` voltar:

```ruby
      # `payload:` (opcional) traduz o Result do command em chaves extras do
      # payload da mutation. Existe porque `audited` devolve `{ ok:, errors: }`
      # literal e descarta o Result — sem isto, nenhuma mutation de cidade
      # consegue devolver dado nenhum. Fica AQUI, e não no `audited`, porque
      # auditoria não tem nada a ver com "o command devolveu dado", e este
      # método já conhece Result (inspeciona failure? e message logo abaixo).
      #
      # Só é chamado no caminho de sucesso: numa recusa não existe Result de
      # sucesso para ler.
      def in_city(city_slug:, event:, module_name:, rejection_path: "version", field_paths: {}, step_up_code: nil,
                  payload: nil, **fields)
        refuse_out_of_scope!(city_slug)
        refusal = unauditable_input(city_slug, fields, DEFAULT_FIELD_PATHS.merge(field_paths))
        return refusal if refusal

        correlation_id = nil
        command_result = nil
        outcome = audited(event: event, module_name: module_name, city_slug: city_slug, **fields) do |attempt_id|
          correlation_id = attempt_id
          city = City.find_by(slug: city_slug)
          raise Rejected.new("cidade inexistente", path: "citySlug") if city.nil?

          begin
            CityWriter.ensure_writable!(city)
            step_up!(step_up_code) unless step_up_code.nil?
            result = CityWriter.call(city) { yield(MaintainerActor.new(credential.maintainer), correlation_id) }
          rescue CityWriter::NotWritable => e
            raise Rejected.new(e.message, path: "citySlug")
          end
          if result.failure?
            path = rejection_path.respond_to?(:call) ? rejection_path.call(result) : rejection_path
            raise Rejected.new(result.message.presence || result.reason.to_s, path: path)
          end

          command_result = result
          result
        end

        return outcome if payload.nil? || !outcome[:ok]

        outcome.merge(payload.call(command_result))
      end
```

Não mude mais nada no método: os `rescue` do fim continuam onde estão.

- [ ] **Step 4: O campo na mutation**

Em `apps/api/app/graphql/maintenance/mutations/revert_protocol_activation.rb`, declare o campo na classe e passe o mapa ao `in_city`. O `field` vai logo depois da linha `description` da classe, e o `payload:` entra na chamada do `in_city` junto dos outros argumentos nomeados:

```ruby
      # O número que passou a valer. Vem do Result do command — o que a tela
      # afirma é o que ele decidiu sob lock, nunca uma releitura depois do ato.
      # Int como `version` já é nos tipos de manutenção; nulo quando a mutation
      # não chegou a reverter.
      field :reverted_to_version, Integer, null: true

      def resolve(city_slug:, name:, reason:, code:)
        in_city(city_slug: city_slug, step_up_code: code, event: "maintenance.protocol.reverted", module_name: "protocol",
                rejection_path: ->(result) { result.reason == :reason_required ? "reason" : "name" },
                protocol_key: name, reason_given: !reason.to_s.strip.empty?,
                changed_fields: [ "status" ],
                payload: ->(result) { { reverted_to_version: result.payload[:protocol_definition].version } }) do |actor, correlation_id|
          Protocols::RevertActivation.call(name: name, reason: reason, by: actor, correlation_id: correlation_id)
        end
      end
```

- [ ] **Step 5: Rode os testes e confirme que passam**

```bash
docker compose exec -T api bundle exec rspec spec/requests/maintenance/protocol_mutations_spec.rb
```

Esperado: PASS, incluindo os três exemplos novos.

- [ ] **Step 6: Guarda de schema**

A API de manutenção tem uma guarda de allowlist de campos: `spec/architecture/maintenance_schema_spec.rb`. Campo novo no schema quebra essa guarda até ser declarado ali. Rode:

```bash
docker compose exec -T api bundle exec rspec spec/architecture/maintenance_schema_spec.rb
```

Se falhar, acrescente `revertedToVersion` à lista do payload da reversão nesse arquivo — declare, nunca afrouxe a guarda. Rode de novo até passar.

- [ ] **Step 7: Suíte completa**

Na raiz do monorepo:

```bash
docker compose stop worker
docker compose exec -T api bundle exec rspec
docker compose start worker
```

Esperado: 0 falhas. Religue o worker mesmo se falhar.

- [ ] **Step 8: Commit**

```bash
cd apps/api
/opt/homebrew/bin/git add app/graphql/maintenance/mutations/city_mutation.rb \
  app/graphql/maintenance/mutations/revert_protocol_activation.rb \
  spec/requests/maintenance/protocol_mutations_spec.rb
/opt/homebrew/bin/git commit
```

Assunto sugerido: `feat: return the version a revert restored from the maintenance API`. Corpo: por que o caminho abriu no `in_city` e não no `audited`. Trailer obrigatório. Se o Step 6 tiver mudado `spec/architecture/maintenance_schema_spec.rb`, inclua esse arquivo no mesmo `git add`.

---

### Task 2: a frase do dashboard

**Files:**
- Modify: `apps/dashboard/src/lib/api.ts`
- Modify: `apps/dashboard/src/modules/Protocols.tsx`
- Test: `apps/dashboard/src/modules/Protocols.test.tsx`

**Interfaces:**
- Consumes: a API REST da cidade, que **já** responde `{ok: true, protocol: {name, version, status}}` em `POST /admin/api/protocols/revert` (ver `app/controllers/concerns/protocol_result_rendering.rb`). Nenhuma mudança de backend é necessária para esta tarefa — ela não depende da Tarefa 1.
- Produces: nada que outra tarefa consuma.

- [ ] **Step 1: Escreva os testes que falham**

Em `apps/dashboard/src/modules/Protocols.test.tsx`. O arquivo mocka o módulo `../lib/api` inteiro (topo do arquivo), então a resposta da API se ajusta com `mocked(api.revertProtocol).mockResolvedValue(...)`, não com `fetch`. Os helpers `stubReads`, `row`, `versionRow` e `renderProtocols` já existem; `row`/`versionRow` já aceitam `revertTargetVersion`.

```tsx
  it("a frase de sucesso nomeia a versão que passou a valer, não a que saiu de uso", async () => {
    stubReads(
      [ row({ status: "active", revertible: true, version: "3", revertTargetVersion: "2" }) ],
      [ versionRow({ status: "active", revertible: true, version: "3", revertTargetVersion: "2" }) ]
    );
    mocked(api.revertProtocol).mockResolvedValue({ version: "2" });
    renderProtocols("protocol_publisher");

    fireEvent.click(await screen.findByText("dengue"));
    fireEvent.click(await screen.findByRole("button", { name: "Reverter" }));
    fireEvent.change(screen.getByLabelText("Motivo"), { target: { value: "regra errada" } });
    fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));

    const status = await screen.findByRole("status");
    expect(status.textContent).toContain("a cidade está com dengue v2");
    expect(status.textContent).not.toContain("v3");
  });

  // Previsão e resultado vêm de fontes DIFERENTES do arranjo — a previsão do
  // `revertTargetVersion` da linha, o resultado do retorno da API. Com o mesmo
  // número nos dois, o exemplo passaria sem provar nada.
  it("quando a versão efetivada difere da prevista, a frase diz as duas", async () => {
    stubReads(
      [ row({ status: "active", revertible: true, version: "3", revertTargetVersion: "2" }) ],
      [ versionRow({ status: "active", revertible: true, version: "3", revertTargetVersion: "2" }) ]
    );
    mocked(api.revertProtocol).mockResolvedValue({ version: "5" });
    renderProtocols("protocol_publisher");

    fireEvent.click(await screen.findByText("dengue"));
    fireEvent.click(await screen.findByRole("button", { name: "Reverter" }));
    fireEvent.change(screen.getByLabelText("Motivo"), { target: { value: "regra errada" } });
    fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));

    const status = await screen.findByRole("status");
    expect(status.textContent).toContain("estava previsto v2");
    expect(status.textContent).toContain("a cidade está com dengue v5");
  });

  it("sem número na resposta, a frase sai sem número e nunca com undefined", async () => {
    stubReads(
      [ row({ status: "active", revertible: true, version: "3", revertTargetVersion: "2" }) ],
      [ versionRow({ status: "active", revertible: true, version: "3", revertTargetVersion: "2" }) ]
    );
    mocked(api.revertProtocol).mockResolvedValue(null);
    renderProtocols("protocol_publisher");

    fireEvent.click(await screen.findByText("dengue"));
    fireEvent.click(await screen.findByRole("button", { name: "Reverter" }));
    fireEvent.change(screen.getByLabelText("Motivo"), { target: { value: "regra errada" } });
    fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));

    const status = await screen.findByRole("status");
    expect(status.textContent).toContain("Reverter concluído");
    expect(status.textContent).not.toContain("undefined");
  });

  it("publicar continua nomeando a versão sobre a qual se agiu", async () => {
    stubReads(
      [ row({ status: "in_review", version: "4" }) ],
      [ versionRow({ status: "in_review", version: "4" }) ]
    );
    mocked(api.publishProtocolVersion).mockResolvedValue(undefined);
    renderProtocols("protocol_publisher");

    fireEvent.click(await screen.findByText("dengue"));
    fireEvent.click(await screen.findByRole("button", { name: "Publicar" }));
    fireEvent.click(screen.getByRole("button", { name: "Confirmar" }));

    const status = await screen.findByRole("status");
    expect(status.textContent).toContain("v4");
  });
```

**Um ajuste obrigatório no arquivo, que não é opcional:** o exemplo que já existe, `"reverter pede motivo e manda nome e motivo"`, faz `mocked(api.revertProtocol).mockResolvedValue(undefined)`. Depois do Step 3 o tipo de retorno passa a ser `{ version: string } | null`, e `undefined` não satisfaz esse tipo — o `npm run typecheck` quebra. Troque aquele `undefined` por `null`. O exemplo continua provando o que provava (os argumentos da chamada) e passa a exercitar o ramo sem número.

O segundo cobre o Review Focus 4, o terceiro o 3, o quarto o 5.

- [ ] **Step 2: Rode e confirme que falham**

```bash
cd apps/dashboard && npx vitest run src/modules/Protocols.test.tsx
```

Esperado: falham os três primeiros (a frase de hoje nomeia `pending.version`); o quarto já passa, e é isso mesmo — ele existe para travar o comportamento correto que não pode regredir.

- [ ] **Step 3: `revertProtocol` devolve o protocolo**

Em `apps/dashboard/src/lib/api.ts`, a função hoje descarta o corpo:

```ts
// Reversão de emergência: a rota não leva versão — a API acha a ativa pelo nome.
export async function revertProtocol(name: string, reason: string): Promise<void> {
  await jsonFetch<unknown>(`${PROTOCOLS_BASE}/revert`, {
    method: "POST", body: JSON.stringify({ name, reason })
  });
}
```

Troque por:

```ts
// Reversão de emergência: a rota não leva versão — a API acha a ativa pelo nome.
//
// Devolve a versão que PASSOU A VALER, que a resposta já trazia e esta função
// descartava. Não é a versão sobre a qual se agiu: a reversão sai da ativa e
// volta para a anterior. `protocol` é opcional de propósito — uma resposta sem
// ele faz a tela dizer a frase sem número, nunca "undefined".
export async function revertProtocol(
  name: string, reason: string
): Promise<{ version: string } | null> {
  const body = await jsonFetch<{ protocol?: { version?: string } }>(`${PROTOCOLS_BASE}/revert`, {
    method: "POST", body: JSON.stringify({ name, reason })
  });
  const version = body?.protocol?.version;
  return version == null ? null : { version: String(version) };
}
```

- [ ] **Step 4: A frase**

Em `apps/dashboard/src/modules/Protocols.tsx`.

O `SensitiveAction` tem `run(values): Promise<void>` e `onDone(): void` — o valor que o `run` resolve não chega ao `onDone`. **Não** alargue o contrato do `SensitiveAction`: ele é compartilhado com outros módulos, e mudar a assinatura por causa de uma ação custa mais do que compra. Use um `ref`, que é o padrão já usado nesta casa para o mesmo problema (ver `apps/maintenance/src/screens/Tokens.tsx`).

Acrescente o ref ao lado dos estados do componente:

```tsx
  // O `run` do SensitiveAction resolve para void: o número que a reversão
  // efetivou viaja por aqui até o onDone. Limpo ao abrir cada painel, para
  // que uma ação nunca leia o resultado da anterior.
  const revertedToRef = useRef<string | null>(null);
```

Importe `useRef` junto dos outros imports do React.

Em `runAction`, o ramo da reversão passa a guardar o número:

```tsx
    case "revert": {
      const reverted = await revertProtocol(name, values.reason ?? "");
      revertedToRef.current = reverted?.version ?? null;
      return;
    }
```

No `onPick`, que já limpa o estado ao abrir o painel, acrescente a limpeza do ref:

```tsx
                    onPick={(action) => {
                      revertedToRef.current = null;
                      setPending({ version: v.version, revertTargetVersion: v.revertTargetVersion, action });
                      setDone(null);
                    }}
```

E o `onDone` passa a escolher a frase:

```tsx
            onDone={() => {
              setDone(doneMessage(pending, name, revertedToRef.current));
              setPending(null);
              void queryClient.invalidateQueries({ queryKey: [ "protocols" ] });
              void queryClient.invalidateQueries({ queryKey: [ "protocol-detail", id ] });
              void auth.reload();
            }}
```

Acrescente a função pura, ao lado de `revertDescription`:

```tsx
// A reversão é a única ação cuja versão de sucesso NÃO é a versão sobre a qual
// se agiu: ela sai da ativa e volta para a anterior. Nomear `pending.version`
// ali, como as outras ações fazem com razão, anuncia a versão que acabou de
// sair de uso (spec 2026-09-24-revert-confirmation §origem).
//
// Divergir do previsto é fato, não alarme: significa que outra ativação comitou
// entre a leitura e o clique e o servidor reresolveu sob lock, como deve. A
// frase conta, com o mesmo peso do sucesso comum.
function doneMessage(
  pending: { version: string; revertTargetVersion: string | null; action: LifecycleAction },
  name: string,
  revertedTo: string | null
): string {
  if (pending.action.kind !== "revert") {
    return `${pending.action.label} concluído: ${name} v${pending.version}`;
  }
  if (revertedTo == null) return `${pending.action.label} concluído.`;
  if (pending.revertTargetVersion != null && pending.revertTargetVersion !== revertedTo) {
    return `${pending.action.label} concluído: estava previsto v${pending.revertTargetVersion}; ` +
      `a cidade está com ${name} v${revertedTo}.`;
  }
  return `${pending.action.label} concluído: a cidade está com ${name} v${revertedTo}.`;
}
```

- [ ] **Step 5: Rode os testes e confirme que passam**

```bash
cd apps/dashboard && npx vitest run src/modules/Protocols.test.tsx
```

Esperado: PASS nos quatro.

- [ ] **Step 6: Verificação completa**

```bash
cd apps/dashboard && npm run typecheck && npm test && npm run build
```

Esperado: os três verdes.

- [ ] **Step 7: Commit**

```bash
cd apps/dashboard
/opt/homebrew/bin/git add src/lib/api.ts src/modules/Protocols.tsx src/modules/Protocols.test.tsx
/opt/homebrew/bin/git commit
```

Assunto sugerido: `fix: name the version a revert restored in the city panel`. Corpo: que a frase antiga nomeava a versão que saiu de uso. Trailer obrigatório.

---

### Task 3: a frase da manutenção

**Files:**
- Modify: `apps/maintenance/src/screens/ProtocolsTab.tsx`
- Test: `apps/maintenance/src/screens/ProtocolsTab.test.tsx`
- Gerados: `apps/maintenance/schema.graphql`, `apps/maintenance/src/gql/*`

**Interfaces:**
- Consumes: `revertedToVersion: Int` no payload de `revertProtocolActivation`, produzido pela Tarefa 1. **Esta tarefa depende da Tarefa 1 estar na branch do api**, porque o `schema:pull` lê o schema do api em execução.
- Produces: nada que outra tarefa consuma.

- [ ] **Step 1: Atualize schema e tipos gerados**

O `apps/api` precisa estar rodando no container **na branch da Tarefa 1**. Na pasta do maintenance:

```bash
cd apps/maintenance && npm run schema:pull && npm run codegen
```

Confirme que `revertedToVersion` apareceu:

```bash
grep -n "revertedToVersion" schema.graphql
```

Se não aparecer, o container do api pode precisar recarregar: `docker compose restart api` na raiz do monorepo, e tente de novo. Se ainda assim não aparecer, **pare e reporte** — não escreva o campo à mão em `schema.graphql`, que é arquivo gerado.

- [ ] **Step 2: Escreva os testes que falham**

Em `apps/maintenance/src/screens/ProtocolsTab.test.tsx`. Os helpers `route`, `v`, `renderTab`, `calls` e `bodyOf` já existem; `v` já aceita `revertTargetVersion`.

O helper `mutationReply` hoje só monta `{ ok, errors }`:

```tsx
function mutationReply(field: string, ok: boolean, errors: unknown[] = []) {
  return reply(200, { data: { [field]: { ok, errors } } });
}
```

Acrescente um quarto parâmetro para os campos extras, mantendo as chamadas existentes funcionando sem mudança:

```tsx
function mutationReply(field: string, ok: boolean, errors: unknown[] = [], extra: object = {}) {
  return reply(200, { data: { [field]: { ok, errors, ...extra } } });
}
```

Os quatro exemplos:

```tsx
  it("a frase de sucesso nomeia a versão que passou a valer, não a que saiu de uso", async () => {
    const user = userEvent.setup();
    route([ v({ status: "active", version: 3, revertible: true, revertTargetVersion: 2 }) ], {
      revertProtocolActivation: () => mutationReply("revertProtocolActivation", true, [], { revertedToVersion: 2 })
    });
    renderTab();

    await user.click(await screen.findByRole("button", { name: "Reverter" }));
    await user.type(screen.getByLabelText("Motivo"), "regra errada em produção");
    await user.type(screen.getByLabelText("Código do autenticador"), "654321");
    await user.click(screen.getByRole("button", { name: "Confirmar" }));

    const status = await screen.findByRole("status");
    expect(status.textContent).toContain("a cidade está com dengue v2");
    expect(status.textContent).not.toContain("v3");
  });

  // Previsão e resultado vêm de fontes DIFERENTES do arranjo — a previsão do
  // `revertTargetVersion` da linha, o resultado do payload da mutation. Com o
  // mesmo número nos dois, o exemplo passaria sem provar nada.
  it("quando a versão efetivada difere da prevista, a frase diz as duas", async () => {
    const user = userEvent.setup();
    route([ v({ status: "active", version: 3, revertible: true, revertTargetVersion: 2 }) ], {
      revertProtocolActivation: () => mutationReply("revertProtocolActivation", true, [], { revertedToVersion: 5 })
    });
    renderTab();

    await user.click(await screen.findByRole("button", { name: "Reverter" }));
    await user.type(screen.getByLabelText("Motivo"), "regra errada em produção");
    await user.type(screen.getByLabelText("Código do autenticador"), "654321");
    await user.click(screen.getByRole("button", { name: "Confirmar" }));

    const status = await screen.findByRole("status");
    expect(status.textContent).toContain("estava previsto v2");
    expect(status.textContent).toContain("a cidade está com dengue v5");
  });

  it("sem número no payload, a frase sai sem número e nunca com undefined", async () => {
    const user = userEvent.setup();
    route([ v({ status: "active", version: 3, revertible: true, revertTargetVersion: 2 }) ], {
      revertProtocolActivation: () => mutationReply("revertProtocolActivation", true)
    });
    renderTab();

    await user.click(await screen.findByRole("button", { name: "Reverter" }));
    await user.type(screen.getByLabelText("Motivo"), "regra errada em produção");
    await user.type(screen.getByLabelText("Código do autenticador"), "654321");
    await user.click(screen.getByRole("button", { name: "Confirmar" }));

    const status = await screen.findByRole("status");
    expect(status.textContent).toContain("concluído");
    expect(status.textContent).not.toContain("undefined");
  });

  it("publicar continua nomeando a versão sobre a qual se agiu", async () => {
    const user = userEvent.setup();
    route([ v({ status: "in_review", version: 4, publicationSignatures: 2, publicationMissing: 0 }) ], {
      publishProtocol: () => mutationReply("publishProtocol", true)
    });
    renderTab();

    await user.click(await screen.findByRole("button", { name: "Publicar" }));
    await user.type(screen.getByLabelText("Código do autenticador"), "654321");
    await user.click(screen.getByRole("button", { name: "Confirmar" }));

    const status = await screen.findByRole("status");
    expect(status.textContent).toContain("v4");
  });
```

O segundo exemplo cobre o Review Focus 4, o terceiro o 3, o quarto o 5.

- [ ] **Step 3: Rode e confirme que falham**

```bash
cd apps/maintenance && npx vitest run src/screens/ProtocolsTab.test.tsx
```

Esperado: falham os três primeiros; o quarto já passa e existe para travar o que não pode regredir.

- [ ] **Step 4: Peça o campo na mutation**

Em `apps/maintenance/src/screens/ProtocolsTab.tsx`, a consulta da reversão hoje é:

```ts
const RevertMutation = graphql(`
  mutation RevertProtocolActivation($citySlug: String!, $name: String!, $reason: String!, $code: String!) {
    revertProtocolActivation(citySlug: $citySlug, name: $name, reason: $reason, code: $code) { ok errors { path message } }
  }
`);
```

Acrescente o campo:

```ts
const RevertMutation = graphql(`
  mutation RevertProtocolActivation($citySlug: String!, $name: String!, $reason: String!, $code: String!) {
    revertProtocolActivation(citySlug: $citySlug, name: $name, reason: $reason, code: $code) {
      ok
      revertedToVersion
      errors { path message }
    }
  }
`);
```

Rode `npm run codegen` de novo para os tipos acompanharem.

- [ ] **Step 5: Leve o número até a frase**

Ainda em `ProtocolsTab.tsx`. O tipo `Payload` é compartilhado pelas cinco ações:

```ts
type Payload = { ok: boolean; errors: FieldError[] };
```

Acrescente o campo como opcional, porque só a reversão o traz:

```ts
// `revertedToVersion` só existe no payload da reversão — opcional aqui, e é a
// opcionalidade que obriga a tela a tratar a ausência em vez de imprimir
// undefined.
type Payload = { ok: boolean; errors: FieldError[]; revertedToVersion?: number | null };
```

No `onSuccess` da mutation, a linha de hoje é:

```ts
        setDone(`${p.action.label} concluído: ${p.row.name} v${p.row.version}`);
```

Troque por:

```ts
        setDone(doneMessage(p, payload.revertedToVersion ?? null));
```

E acrescente a função pura, ao lado de `revertNotice`:

```ts
// A reversão é a única ação cuja versão de sucesso NÃO é a versão sobre a qual
// se agiu: ela sai da ativa e volta para a anterior. Nomear `row.version` ali,
// como as outras ações fazem com razão, anuncia a versão que acabou de sair de
// uso (spec 2026-09-24-revert-confirmation §origem).
//
// Divergir do previsto é fato, não alarme: significa que outra ativação comitou
// entre a leitura e o clique e o servidor reresolveu sob lock, como deve.
function doneMessage(pending: Pending, revertedTo: number | null): string {
  const { row, action } = pending;
  if (action.kind !== "revert") return `${action.label} concluído: ${row.name} v${row.version}`;
  if (revertedTo == null) return `${action.label} concluído.`;
  if (row.revertTargetVersion != null && row.revertTargetVersion !== revertedTo) {
    return `${action.label} concluído: estava previsto v${row.revertTargetVersion}; ` +
      `a cidade está com ${row.name} v${revertedTo}.`;
  }
  return `${action.label} concluído: a cidade está com ${row.name} v${revertedTo}.`;
}
```

- [ ] **Step 6: Rode os testes e confirme que passam**

```bash
cd apps/maintenance && npx vitest run src/screens/ProtocolsTab.test.tsx
```

Esperado: PASS nos quatro.

- [ ] **Step 7: Verificação completa**

```bash
cd apps/maintenance && npm run typecheck && npm test && npm run build
```

Esperado: os três verdes.

- [ ] **Step 8: Commit**

```bash
cd apps/maintenance
/opt/homebrew/bin/git add src/screens/ProtocolsTab.tsx src/screens/ProtocolsTab.test.tsx schema.graphql src/gql
/opt/homebrew/bin/git commit
```

Assunto sugerido: `fix: name the version a revert restored in the maintenance tab`. Corpo: que a frase antiga nomeava a versão que saiu de uso. Trailer obrigatório.
