# Maintenance frontend — Plano 2: telas de escrita de protocolo

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** o mantenedor opera o ciclo de vida de protocolo de uma cidade pela aba Protocolos: enviar para revisão, publicar, ativar, aposentar e reverter. Cada versão mostra o estado das assinaturas, e toda ação que a API recusaria de antemão aparece desabilitada, com o motivo.

**Architecture:** a API de manutenção ganha um campo de leitura, `city.protocolVersions`, que lista todas as versões com o estado de assinatura. O estado é calculado pelas funções de domínio que já existem (`Protocols::Signatures`, `Protocols::RevertActivation.revertible?`). O frontend troca a tabela da aba Protocolos por um componente próprio, `ProtocolsTab`. A regra "quais ações cabem nesta versão, e por que não" vira uma função pura, e o formulário de confirmação usa as mutations do Plano 5 da API.

**Tech Stack:** Rails 8.1 + graphql-ruby 2.6.10 (apps/api); Vite + React 18 + React Query 5 + graphql-codegen (client preset) + Vitest + Playwright (apps/maintenance).

**Spec:**
- `docs/superpowers/specs/2026-09-18-maintenance-frontend-design.md`. Este plano estende o §8, que deixou as telas de escrita de fora da v1.
- `docs/superpowers/specs/2026-09-18-protocol-signatures-design.md`, para as regras de assinatura e reversão.
- As decisões de escopo foram tomadas em conversa em 2026-09-22 (abaixo, em "Decisões").

## Decisões (tomadas com o usuário, 2026-09-22)

1. **Escopo A: só o ciclo de vida.** Não há editor de rascunho: `saveProtocolDraft` continua existindo na API, mas não ganha tela. A autoria é das equipes clínicas, no editor do dashboard. O campo `definition` **não** é exposto na leitura.
2. **Transições por status**, conferidas nos commands em `apps/api/app/commands/protocols/`:

   | status | ações oferecidas |
   |---|---|
   | `draft` | Enviar para revisão; Aposentar |
   | `in_review` | Publicar; Aposentar |
   | `published` | Ativar; Aposentar |
   | `active` | Reverter, só quando `revertible` (nunca Aposentar: R4) |
   | `retired` | nenhuma |

3. **Step-up** (o TOTP do momento) em Publicar, Ativar, Aposentar e Reverter. Enviar para revisão não pede. Reverter pede também um **motivo**.
4. **Ação desabilitada com motivo** quando a API recusaria com certeza pelo estado lido:
   - Publicar/Ativar com `eligibleReviewers < 2`: `a cidade tem N revisor(es) elegível(is); são necessários 2`;
   - senão, com assinaturas faltando: `falta 1 assinatura` ou `faltam N assinaturas`, a mesma frase de `Protocols::Signatures.shortfall_message`.

   O estado pode ficar obsoleto entre a leitura e o clique. A API continua sendo quem decide, e a recusa dela aparece inline.
5. **Sem e-mail de revisor na API de manutenção.** Só contagens. O mantenedor não precisa saber quem assinou para operar, e é um dado a menos numa superfície com alcance em todas as cidades.
6. **`city.protocols` fica intacto** (contrato já consumido): é um campo novo, não uma mudança de semântica.
7. **O estado de assinatura NÃO é extraído de `Admin::ProtocolsQuery`**, porque `Protocols::Signatures.missing` / `eligible_reviewer_count` e `RevertActivation.revertible?` já são a regra compartilhada. O tipo novo só as chama, sem refatorar o painel da cidade.

## Global Constraints

- Commits: Conventional Commits **em inglês**, tipo por extenso (`feat`, `fix`, `refactor`, `test`, `docs`, `chore`...), terminando com `Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>`.
- Git: use `/opt/homebrew/bin/git` (o `git` do PATH aborta). `apps/api` e `apps/maintenance` são repositórios separados; a raiz do monorepo não é git.
- Trabalho em branch: `feat/maintenance-protocol-writes` em **cada** repo tocado (api e maintenance). Nunca em `main`, nunca push.
- Suíte completa do api: `docker compose stop worker` antes (na raiz do monorepo) e `docker compose start worker` depois. Rode dentro do container: `docker compose exec -T api bundle exec rspec`. Suíte acima de ~3 min é regressão.
- Specs de request exigem `type: :request` (a inferência por diretório está desligada). Consultas GraphQL em specs ficam em **métodos ou `let`, nunca em constante** de topo.
- A API de manutenção **não expõe e-mail de usuário de cidade** fora de `accounts`: o tipo novo não tem campo de e-mail nem de id de usuário.
- Toda leitura de dentro da cidade passa por `CityReader` (via o helper `inside` de `CityType`), com erro de campo `CITY_ARCHIVED` / `CITY_UNREACHABLE` / `CITY_READ_FAILED`.
- Frontend: toda consulta de aba usa `staleTime: Infinity` e só dispara com a aba aberta. Nenhuma tela lê status HTTP (erros vêm de `src/lib/errors.ts`). Mensagens ao usuário em português.
- O código TOTP é limpo do estado **a cada tentativa** (vale uma vez só). Nunca ligue React Query Devtools nem assine o MutationCache (ver o comentário em `src/screens/Tokens.tsx`).
- Nunca rode `start.sh`. Nunca desligue o trigger de imutabilidade. Nunca imprima segredos.

## File Structure

**apps/api**
- Create: `app/graphql/maintenance/types/protocol_version_type.rb`, uma versão mais o estado de assinatura (só contagens e booleanos).
- Modify: `app/graphql/maintenance/types/city_type.rb`, com o campo `protocol_versions` e o resolver `inside`.
- Create: `spec/requests/maintenance/city_protocol_versions_spec.rb`.

**apps/maintenance**
- Modify: `schema.graphql` (via `npm run schema:pull`) e `src/gql/*` (via `npm run codegen`).
- Create: `src/lib/protocolActions.ts`, com a regra pura "que ações cabem nesta versão".
- Create: `src/lib/protocolActions.test.ts`.
- Create: `src/screens/ProtocolsTab.tsx`, com a tabela, o formulário de confirmação e as mutations.
- Create: `src/screens/ProtocolsTab.test.tsx`.
- Modify: `src/screens/CityDetail.tsx`: a aba Protocolos passa a renderizar `ProtocolsTab`, e a `CityProtocolsQuery` sai.
- Modify: `src/screens/CityDetail.test.tsx`: o nome da operação `CityProtocols` passa a `CityProtocolVersions`.
- Modify: `e2e/maintenance.spec.ts` e `e2e/support.ts`, com o teste "protocolos".
- Modify: `README.md`, com uma seção curta sobre a aba Protocolos.

---

### Task 1: API — `city.protocolVersions` com estado de assinatura

**Files:**
- Create: `apps/api/app/graphql/maintenance/types/protocol_version_type.rb`
- Modify: `apps/api/app/graphql/maintenance/types/city_type.rb`
- Test: `apps/api/spec/requests/maintenance/city_protocol_versions_spec.rb`

**Interfaces:**
- Consumes:
  - `Protocols::Signatures.missing(protocol, purpose:)` e `valid_signer_ids(protocol, purpose:)`, com `purpose` `"publication"` ou `"activation"`;
  - `Protocols::Signatures.eligible_reviewer_count(protocol)`;
  - `Protocols::RevertActivation.revertible?(protocol)`;
  - os helpers de spec `protocol_definition_hash`, `make_reviewer!` e `sign!(protocol, purpose:, by:)` (`spec/support/protocol_signatures.rb`).
- Produces (o SDL que o frontend consome):
  ```graphql
  type ProtocolVersion {
    name: String!
    version: Int!
    status: String!            # draft | in_review | published | active | retired
    publicationSignatures: Int!
    publicationMissing: Int!
    activationSignatures: Int!
    activationMissing: Int!
    eligibleReviewers: Int!
    revertible: Boolean!
  }
  # em City:
  protocolVersions: [ProtocolVersion!]!   # todas as versões, name asc, version desc, até 100
  ```

- [ ] **Step 1: Branch**

```bash
cd apps/api && /opt/homebrew/bin/git checkout -b feat/maintenance-protocol-writes
```

- [ ] **Step 2: Escrever a spec que falha**

Use `spec/requests/maintenance/city_spec.rb` como molde. Copie de lá, sem inventar: `let!(:maintainer)`, `browser`, `json`, `totp`, `login!`, `gql!`, `register_city!`, o arranjo de `city` e o `before` que faz login e registra `TEST_CITY_A`. O arquivo novo:

```ruby
require "rails_helper"

# Leitura do ciclo de vida para as telas de escrita (Plano 2 do frontend de
# manutenção): TODAS as versões, com o estado de assinatura calculado pelas
# mesmas funções de domínio que os commands usam. Só contagens — nenhum
# e-mail nem id de revisor sai por aqui.
RSpec.describe "Maintenance city protocolVersions", type: :request do
  # ... (let!(:maintainer), browser, json, totp, login!, gql!, register_city!,
  #      city e before — copiados de city_spec.rb) ...

  def versions_query
    <<~GQL
      query($slug: String!) {
        city(slug: $slug) {
          protocolVersions {
            name version status
            publicationSignatures publicationMissing
            activationSignatures activationMissing
            eligibleReviewers revertible
          }
        }
      }
    GQL
  end

  def versions = json.dig("data", "city", "protocolVersions")
  def row(name, version) = versions.find { |v| v["name"] == name && v["version"] == version }

  def create_version!(status:, version: 1, name: "dengue")
    ProtocolDefinition.create!(name: name, version: version, status: status,
                               definition: protocol_definition_hash(name: name, version: version))
  end

  it "lista versões em todo status, não só a ativa" do
    %w[draft in_review published retired].each_with_index do |status, i|
      create_version!(status: status, version: i + 1)
    end

    gql!(versions_query, slug: city.slug)

    expect(json["errors"]).to be_nil
    expect(versions.map { |v| [ v["version"], v["status"] ] })
      .to eq([ [ 4, "retired" ], [ 3, "published" ], [ 2, "in_review" ], [ 1, "draft" ] ])
  end

  it "conta assinaturas válidas e o que falta por finalidade" do
    protocol = create_version!(status: "in_review")
    reviewers = Array.new(3) { make_reviewer! }
    sign!(protocol, purpose: "publication", by: reviewers.first)

    gql!(versions_query, slug: city.slug)

    expect(row("dengue", 1)).to include(
      "publicationSignatures" => 1, "publicationMissing" => 1,
      "activationSignatures" => 0, "activationMissing" => 2,
      "eligibleReviewers" => 3, "revertible" => false
    )
  end

  it "responde o mesmo que o domínio para faltantes e revisores elegíveis" do
    protocol = create_version!(status: "in_review")
    2.times { make_reviewer! }

    gql!(versions_query, slug: city.slug)

    expect(row("dengue", 1)["publicationMissing"])
      .to eq(Protocols::Signatures.missing(protocol, purpose: "publication"))
    expect(row("dengue", 1)["eligibleReviewers"])
      .to eq(Protocols::Signatures.eligible_reviewer_count(protocol))
  end

  it "não expõe e-mail nem id de revisor" do
    protocol = create_version!(status: "in_review")
    reviewer = make_reviewer!
    sign!(protocol, purpose: "publication", by: reviewer)

    gql!(versions_query, slug: city.slug)

    expect(response.body).not_to include(reviewer.email_address)
    expect(response.body).not_to include(reviewer.id.to_s)
  end

  it "não tem campo de definição (escopo A: sem editor)" do
    gql!('query($slug: String!) { city(slug: $slug) { protocolVersions { definition } } }', slug: city.slug)

    expect(json["errors"].first["message"]).to include("definition")
  end

  it "responde lista vazia numa cidade sem protocolo" do
    gql!(versions_query, slug: city.slug)

    expect(versions).to eq([])
  end
end
```

Acrescente um exemplo de cidade inalcançável, **copiando** o arranjo que `city_spec.rb` já usa para `CITY_UNREACHABLE`; não aponte para uma URL real (ver a memória "Specs não inferem tipo"). Ele espera `errors.first.extensions.code == "CITY_UNREACHABLE"` com path `["city", "protocolVersions"]`.

Para o caso "revertible", reuse o arranjo de `spec/requests/admin/protocols_signatures_spec.rb:68-92`, que ativa duas versões assinadas. O exemplo espera que só a versão ativa, depois da segunda ativação, responda `revertible: true`.

- [ ] **Step 3: Rodar e ver falhar**

```bash
docker compose exec -T api bundle exec rspec spec/requests/maintenance/city_protocol_versions_spec.rb
```
Esperado: FAIL com `Field 'protocolVersions' doesn't exist on type 'City'`.

- [ ] **Step 4: Implementar o tipo**

`app/graphql/maintenance/types/protocol_version_type.rb`:

```ruby
module Maintenance
  module Types
    # Uma versão de protocolo e o estado de assinatura dela (spec de
    # assinaturas §5, ADR-0016), para o mantenedor saber ANTES de gastar um
    # TOTP se publicar/ativar vai passar. Só contagens e booleanos: quem
    # assinou (e-mail, id) não sai pela API de manutenção. Tudo vem das
    # funções que os commands usam — este tipo não reimplementa regra.
    #
    # Lido sem lock: pode ficar obsoleto assim que outra escrita comita. A
    # decisão de verdade é a do command, que trava a linha e reconfere.
    #
    # Os valores chegam PRONTOS num Hash montado por CityType#protocol_versions:
    # a conexão da cidade (CityReader) fecha ao sair do `inside`, então nada
    # aqui pode consultar o banco — mesmo desenho de CityCountsType.
    class ProtocolVersionType < BaseObject
      description "Versão de protocolo com estado de assinatura (contagens, sem identidade de revisor)."

      field :name, String, null: false
      field :version, Integer, null: false
      field :status, String, null: false
      field :publication_signatures, Integer, null: false
      field :publication_missing, Integer, null: false
      field :activation_signatures, Integer, null: false
      field :activation_missing, Integer, null: false
      field :eligible_reviewers, Integer, null: false
      field :revertible, Boolean, null: false
    end
  end
end
```

- [ ] **Step 5: Implementar o campo em `CityType`**

Em `app/graphql/maintenance/types/city_type.rb`, depois de `field :protocols`:

```ruby
      # Plano 2 do frontend (telas de escrita): TODAS as versões, com o estado
      # de assinatura — `protocols` acima segue só com as ativas (contrato já
      # consumido). Teto de 100 como as listas de `operations`: cada versão
      # custa algumas consultas de assinatura.
      field :protocol_versions, [ Types::ProtocolVersionType ], null: false
```

e, junto dos outros resolvers:

```ruby
      # Calculado DENTRO de `inside`: a conexão da cidade fecha ao sair do
      # bloco, então ProtocolVersionType só lê as chaves já prontas.
      def protocol_versions
        inside do
          ProtocolDefinition.order(:name, version: :desc).limit(100).map do |d|
            {
              name: d.name, version: d.version, status: d.status,
              publication_signatures: Protocols::Signatures.valid_signer_ids(d, purpose: "publication").size,
              publication_missing: Protocols::Signatures.missing(d, purpose: "publication"),
              activation_signatures: Protocols::Signatures.valid_signer_ids(d, purpose: "activation").size,
              activation_missing: Protocols::Signatures.missing(d, purpose: "activation"),
              eligible_reviewers: Protocols::Signatures.eligible_reviewer_count(d),
              revertible: Protocols::RevertActivation.revertible?(d)
            }
          end
        end
      end
```

Confira como `CityCountsType` resolve a partir de um Hash com chaves símbolo. Se o `BaseObject` não resolver `hash[:campo]` sozinho, use o mesmo mecanismo que `CityCountsType` usa, sem inventar outro.

- [ ] **Step 6: Rodar e ver passar**

```bash
docker compose exec -T api bundle exec rspec spec/requests/maintenance/city_protocol_versions_spec.rb spec/requests/maintenance/city_spec.rb
```
Esperado: PASS.

- [ ] **Step 7: Suíte completa** (com o worker parado, ver Global Constraints). Esperado: 0 falhas, duração na casa de ~2,5 min.

- [ ] **Step 8: Commit**

```bash
/opt/homebrew/bin/git add app/graphql/maintenance/types/protocol_version_type.rb app/graphql/maintenance/types/city_type.rb spec/requests/maintenance/city_protocol_versions_spec.rb
/opt/homebrew/bin/git commit -m "feat: expose protocol versions with signature state to maintenance" -m "Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 2: frontend — schema e regra pura das ações

**Files:**
- Modify: `apps/maintenance/schema.graphql`, `apps/maintenance/src/gql/*` (gerados)
- Create: `apps/maintenance/src/lib/protocolActions.ts`
- Test: `apps/maintenance/src/lib/protocolActions.test.ts`

**Interfaces:**
- Consumes: o SDL de `ProtocolVersion` da Task 1.
- Produces:
  ```ts
  export type ProtocolVersionState = {
    name: string; version: number; status: string;
    publicationMissing: number; activationMissing: number;
    eligibleReviewers: number; revertible: boolean;
  };
  export type ProtocolActionKind = "submit" | "publish" | "activate" | "retire" | "revert";
  export type ProtocolAction = {
    kind: ProtocolActionKind;
    label: string;              // rótulo do botão
    stepUp: boolean;            // pede o TOTP do momento
    needsReason: boolean;       // só revert
    disabledReason: string | null;
  };
  export const REQUIRED_SIGNATURES = 2;
  export function shortfallMessage(missing: number): string;
  export function actionsFor(v: ProtocolVersionState): ProtocolAction[];
  ```

- [ ] **Step 1: Branch, schema e codegen**

```bash
cd apps/maintenance && /opt/homebrew/bin/git checkout -b feat/maintenance-protocol-writes
```

O `schema:pull` lê o schema do container `api` em execução, que monta `apps/api` como volume. O api precisa estar **na branch da Task 1**, então confira `cd ../api && /opt/homebrew/bin/git branch --show-current` antes. Depois:

```bash
npm run schema:pull && npm run codegen
grep -n "type ProtocolVersion" schema.graphql
```
Esperado: `type ProtocolVersion {` presente e `src/gql/graphql.ts` regenerado.

- [ ] **Step 2: Escrever o teste que falha**

`src/lib/protocolActions.test.ts`:

```ts
import { describe, expect, it } from "vitest";
import { actionsFor, shortfallMessage, type ProtocolVersionState } from "./protocolActions";

function version(overrides: Partial<ProtocolVersionState>): ProtocolVersionState {
  return {
    name: "dengue", version: 1, status: "draft",
    publicationMissing: 2, activationMissing: 2, eligibleReviewers: 3, revertible: false,
    ...overrides
  };
}

const kinds = (v: ProtocolVersionState) => actionsFor(v).map((a) => a.kind);

describe("actionsFor", () => {
  it("oferece por status só as transições que o domínio aceita", () => {
    expect(kinds(version({ status: "draft" }))).toEqual([ "submit", "retire" ]);
    expect(kinds(version({ status: "in_review" }))).toEqual([ "publish", "retire" ]);
    expect(kinds(version({ status: "published" }))).toEqual([ "activate", "retire" ]);
    expect(kinds(version({ status: "active", revertible: true }))).toEqual([ "revert" ]);
    expect(kinds(version({ status: "active", revertible: false }))).toEqual([]);
    expect(kinds(version({ status: "retired" }))).toEqual([]);
  });

  it("step-up em tudo menos enviar para revisão; motivo só na reversão", () => {
    const all = [
      ...actionsFor(version({ status: "draft" })),
      ...actionsFor(version({ status: "in_review" })),
      ...actionsFor(version({ status: "published" })),
      ...actionsFor(version({ status: "active", revertible: true }))
    ];
    for (const action of all) {
      expect(action.stepUp).toBe(action.kind !== "submit");
      expect(action.needsReason).toBe(action.kind === "revert");
    }
  });

  it("publicar desabilita com o que falta; habilita com as assinaturas", () => {
    const [ missingTwo ] = actionsFor(version({ status: "in_review", publicationMissing: 2 }));
    expect(missingTwo.disabledReason).toBe("faltam 2 assinaturas");

    const [ missingOne ] = actionsFor(version({ status: "in_review", publicationMissing: 1 }));
    expect(missingOne.disabledReason).toBe("falta 1 assinatura");

    const [ ready ] = actionsFor(version({ status: "in_review", publicationMissing: 0 }));
    expect(ready.disabledReason).toBeNull();
  });

  it("ativar olha as assinaturas de ativação, não as de publicação", () => {
    const [ activate ] = actionsFor(version({ status: "published", publicationMissing: 0, activationMissing: 1 }));
    expect(activate.disabledReason).toBe("falta 1 assinatura");
  });

  it("revisores insuficientes vencem o faltante — a cidade está bloqueada", () => {
    const [ publish ] = actionsFor(version({ status: "in_review", eligibleReviewers: 1, publicationMissing: 2 }));
    expect(publish.disabledReason).toBe("a cidade tem 1 revisor(es) elegível(is); são necessários 2");
  });

  it("aposentar e enviar nunca dependem de assinatura", () => {
    for (const action of actionsFor(version({ status: "draft", eligibleReviewers: 0 }))) {
      expect(action.disabledReason).toBeNull();
    }
  });
});

describe("shortfallMessage", () => {
  it("singular e plural como Protocols::Signatures.shortfall_message", () => {
    expect(shortfallMessage(1)).toBe("falta 1 assinatura");
    expect(shortfallMessage(2)).toBe("faltam 2 assinaturas");
  });
});
```

- [ ] **Step 3: Rodar e ver falhar**

Run: `npx vitest run src/lib/protocolActions.test.ts`
Esperado: FAIL, `Failed to resolve import "./protocolActions"`.

- [ ] **Step 4: Implementar**

`src/lib/protocolActions.ts`:

```ts
// Que ações de ciclo de vida cabem numa versão, e por que uma estaria
// desabilitada (Plano 2, Decisões 2-4). Espelha as pré-condições dos
// commands em apps/api/app/commands/protocols/ — mas é só uma PREVISÃO a
// partir do estado lido: a API continua decidindo (trava a linha e
// reconfere), e a recusa dela aparece na tela.
export type ProtocolVersionState = {
  name: string; version: number; status: string;
  publicationMissing: number; activationMissing: number;
  eligibleReviewers: number; revertible: boolean;
};

export type ProtocolActionKind = "submit" | "publish" | "activate" | "retire" | "revert";

export type ProtocolAction = {
  kind: ProtocolActionKind;
  label: string;
  stepUp: boolean;
  needsReason: boolean;
  disabledReason: string | null;
};

// Protocols::Signatures::REQUIRED na API.
export const REQUIRED_SIGNATURES = 2;

// Mesma frase de Protocols::Signatures.shortfall_message.
export function shortfallMessage(missing: number): string {
  return missing === 1 ? "falta 1 assinatura" : `faltam ${missing} assinaturas`;
}

function signatureBlock(eligibleReviewers: number, missing: number): string | null {
  if (eligibleReviewers < REQUIRED_SIGNATURES) {
    return `a cidade tem ${eligibleReviewers} revisor(es) elegível(is); são necessários ${REQUIRED_SIGNATURES}`;
  }
  return missing > 0 ? shortfallMessage(missing) : null;
}

const RETIRE: ProtocolAction = {
  kind: "retire", label: "Aposentar", stepUp: true, needsReason: false, disabledReason: null
};

export function actionsFor(v: ProtocolVersionState): ProtocolAction[] {
  switch (v.status) {
    case "draft":
      return [ { kind: "submit", label: "Enviar para revisão", stepUp: false, needsReason: false, disabledReason: null }, RETIRE ];
    case "in_review":
      return [ {
        kind: "publish", label: "Publicar", stepUp: true, needsReason: false,
        disabledReason: signatureBlock(v.eligibleReviewers, v.publicationMissing)
      }, RETIRE ];
    case "published":
      return [ {
        kind: "activate", label: "Ativar", stepUp: true, needsReason: false,
        disabledReason: signatureBlock(v.eligibleReviewers, v.activationMissing)
      }, RETIRE ];
    case "active":
      // R4: a versão ativa nunca é aposentada; só a reversão de emergência.
      return v.revertible
        ? [ { kind: "revert", label: "Reverter", stepUp: true, needsReason: true, disabledReason: null } ]
        : [];
    default:
      return [];
  }
}
```

- [ ] **Step 5: Rodar e ver passar**

Run: `npx vitest run src/lib/protocolActions.test.ts && npm run typecheck`
Esperado: PASS; typecheck limpo.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git add schema.graphql src/gql src/lib/protocolActions.ts src/lib/protocolActions.test.ts
/opt/homebrew/bin/git commit -m "feat: derive allowed protocol lifecycle actions from signature state" -m "Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 3: frontend — `ProtocolsTab` com confirmação, step-up e mutations

**Files:**
- Create: `apps/maintenance/src/screens/ProtocolsTab.tsx`
- Test: `apps/maintenance/src/screens/ProtocolsTab.test.tsx`
- Modify: `apps/maintenance/src/screens/CityDetail.tsx`, `apps/maintenance/src/screens/CityDetail.test.tsx`

**Interfaces:**
- Consumes:
  - `actionsFor`, `ProtocolAction` e `ProtocolVersionState` (Task 2);
  - `gql` (`src/lib/api.ts`), que devolve `{ data, fieldErrors }` e **levanta** `GraphQLRefusal` quando `data` vem nulo, `RequestRejected`/`NetworkError`/`AuthRequired` em falha HTTP;
  - `Field`, `Button`, `Panel`, `DataTable`, `EmptyState`, `ErrorState`, `Tag`;
  - `STEP_UP_CODE_HELP_TEXT` e `StepUpCodeError` (`src/components/StepUpCode.tsx`).
- As mutations da API (Plano 5), todas com payload `{ ok errors { path message } }`:
  - `submitProtocolForReview(citySlug, name, version)`;
  - `publishProtocol` / `activateProtocol` / `retireProtocol(citySlug, name, version, code)`;
  - `revertProtocolActivation(citySlug, name, reason, code)`.

  O erro de step-up vem com `path: "code"`; o de motivo, com `path: "reason"`.
- Produces: `export function ProtocolsTab({ slug }: { slug: string })`. Faz a própria consulta `CityProtocolVersions`, com a chave `["city", slug, "protocols"]` e `staleTime: Infinity`.

**Comportamento:**
- **Aba:** o botão "atualizar", a mesma lista de erros de campo de `CityDetail` (`CITY_UNREACHABLE` etc., path `["city","protocolVersions"]` ou só `["city"]`) e `EmptyState` "nenhum protocolo".
- **Tabela**, colunas:
  - Nome;
  - Versão;
  - Status: `Tag` com os rótulos `rascunho`, `em revisão`, `publicada`, `ativa` e `aposentada`;
  - Publicação: `N/2`;
  - Ativação: `N/2`;
  - Revisores: `eligibleReviewers`;
  - Ações: um `Button` por ação de `actionsFor`, desabilitado quando há `disabledReason`, com o motivo num `<small>` logo abaixo.

  Para `N/2`, use `publicationSignatures`/`activationSignatures` da consulta, **não** `2 - missing`: pode haver mais de duas válidas.
- **Clique numa ação habilitada:** abre um `Panel` "Confirmar: <label> <name> v<version>" abaixo da tabela, sempre um só por vez. Ele traz:
  - o campo "Código do autenticador", quando `stepUp`, com `helpText` = `STEP_UP_CODE_HELP_TEXT`;
  - o campo "Motivo", quando `needsReason`;
  - os botões "Confirmar" e "Cancelar".

  Na reversão, o painel diz também: `volta para a versão ativada antes desta, sem assinatura nova`.
- **Enviar:**
  - A mutation daquela ação vai com `citySlug: slug` e `name`/`version` (ou `name`/`reason`/`code` na reversão).
  - O código é limpo do estado **em toda resposta**, sucesso ou erro.
  - `ok: true`: fecha o painel, mostra `role="status"` "<label> concluído: <name> v<version>" e `invalidateQueries({ queryKey: ["city", slug, "protocols"] })`.
  - `ok: false`: erro com `path === "code"` → `StepUpCodeError` debaixo do campo de código; `path === "reason"` → `ErrorState` debaixo do motivo; qualquer outro → `ErrorState` no topo do painel (ex.: "falta 1 assinatura" vindo da própria API). O painel continua aberto.
  - Exceção levantada:
    - `GraphQLRefusal` → `ErrorState` com `${code} — ${message}`. Para `CITY_UNREACHABLE` depois de o command começar, a mensagem da API já diz que o resultado é desconhecido e traz o correlation id. Não reescreva a mensagem.
    - Qualquer outra → `ErrorState` com `err.message`.

    Nos dois casos, a lista também é invalidada, porque o resultado pode ter comitado.
- **Sem mutationKey/gcTime especial:** nenhuma resposta carrega segredo. Mas o `code` vai nas `variables` da mutation, então use `gcTime: 0` na `useMutation`, como Tokens, para ela não ficar no cache.

- [ ] **Step 1: Escrever os testes que falham**

`src/screens/ProtocolsTab.test.tsx`. Use o molde de fetch de `Tokens.test.tsx` (`reply`, `bodyOf`, `operationCalls` filtrando por nome de operação, `vi.stubGlobal("fetch", ...)`) e um `QueryClient` com `retry: false`. Os casos:

```tsx
import { afterEach, beforeEach, describe, expect, it, vi } from "vitest";
import { cleanup, render, screen, waitFor, within } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import type { ReactNode } from "react";
import { ProtocolsTab } from "./ProtocolsTab";

afterEach(cleanup);

function reply(status: number, body: unknown) {
  return new Response(JSON.stringify(body), { status, headers: { "Content-Type": "application/json" } });
}
function bodyOf(call: unknown[]): { query: string; variables: Record<string, unknown> } {
  return JSON.parse((call[1] as RequestInit).body as string);
}
function calls(fetchMock: ReturnType<typeof vi.fn>, operation: string) {
  return fetchMock.mock.calls.filter((call) => bodyOf(call).query.includes(operation));
}

function v(overrides: Record<string, unknown>) {
  return {
    name: "dengue", version: 1, status: "draft",
    publicationSignatures: 0, publicationMissing: 2, activationSignatures: 0, activationMissing: 2,
    eligibleReviewers: 3, revertible: false, ...overrides
  };
}
function versionsReply(rows: unknown[]) {
  return reply(200, { data: { city: { slug: "sp", protocolVersions: rows } } });
}
function mutationReply(field: string, ok: boolean, errors: unknown[] = []) {
  return reply(200, { data: { [field]: { ok, errors } } });
}

function renderTab() {
  const client = new QueryClient({ defaultOptions: { queries: { retry: false } } });
  function wrapper({ children }: { children: ReactNode }) {
    return <QueryClientProvider client={client}>{children}</QueryClientProvider>;
  }
  return render(<ProtocolsTab slug="sp" />, { wrapper });
}

describe("ProtocolsTab", () => {
  let fetchMock: ReturnType<typeof vi.fn>;
  beforeEach(() => { fetchMock = vi.fn(); vi.stubGlobal("fetch", fetchMock); });
  afterEach(() => vi.unstubAllGlobals());

  // `route` responde a consulta de versões com `rows` e cada mutation pelo
  // mapa `mutations` (nome do campo → Response).
  function route(rows: unknown[], mutations: Record<string, () => Response> = {}) {
    fetchMock.mockImplementation((url: string, init?: RequestInit) => {
      const body = bodyOf([ url, init ]);
      if (body.query.includes("query CityProtocolVersions")) return Promise.resolve(versionsReply(rows));
      const hit = Object.keys(mutations).find((field) => body.query.includes(field));
      return Promise.resolve(hit ? mutations[hit]() : reply(500, {}));
    });
  }

  it("mostra status, assinaturas N/2 e revisores; só as ações do status", async () => {
    route([ v({ status: "in_review", version: 2, publicationSignatures: 1, publicationMissing: 1 }) ]);
    renderTab();

    const table = within(await screen.findByRole("table"));
    expect(table.getByText("em revisão")).not.toBeNull();
    expect(table.getByText("1/2")).not.toBeNull();
    expect(table.getByRole("button", { name: "Publicar" })).not.toBeNull();
    expect(table.getByRole("button", { name: "Aposentar" })).not.toBeNull();
    expect(table.queryByRole("button", { name: "Ativar" })).toBeNull();
  });

  it("publicar fica desabilitado com o motivo quando falta assinatura", async () => {
    route([ v({ status: "in_review", publicationMissing: 1 }) ]);
    renderTab();

    const publish = await screen.findByRole("button", { name: "Publicar" });
    expect((publish as HTMLButtonElement).disabled).toBe(true);
    expect(screen.getByText("falta 1 assinatura")).not.toBeNull();
  });

  it("enviar para revisão não pede código e manda name/version/citySlug", async () => {
    const user = userEvent.setup();
    route([ v({ status: "draft" }) ], { submitProtocolForReview: () => mutationReply("submitProtocolForReview", true) });
    renderTab();

    await user.click(await screen.findByRole("button", { name: "Enviar para revisão" }));
    expect(screen.queryByLabelText("Código do autenticador")).toBeNull();
    await user.click(screen.getByRole("button", { name: "Confirmar" }));

    await waitFor(() => expect(calls(fetchMock, "submitProtocolForReview")).toHaveLength(1));
    expect(bodyOf(calls(fetchMock, "submitProtocolForReview")[0]).variables)
      .toEqual({ citySlug: "sp", name: "dengue", version: 1 });
    expect(await screen.findByRole("status")).toHaveProperty("textContent", "Enviar para revisão concluído: dengue v1");
  });

  it("sucesso invalida a lista (nova consulta de versões)", async () => {
    const user = userEvent.setup();
    route([ v({ status: "draft" }) ], { submitProtocolForReview: () => mutationReply("submitProtocolForReview", true) });
    renderTab();

    await user.click(await screen.findByRole("button", { name: "Enviar para revisão" }));
    await user.click(screen.getByRole("button", { name: "Confirmar" }));

    await waitFor(() => expect(calls(fetchMock, "query CityProtocolVersions")).toHaveLength(2));
  });

  it("aposentar pede código, envia e limpa o código mesmo na recusa", async () => {
    const user = userEvent.setup();
    route([ v({ status: "published", activationMissing: 0 }) ], {
      retireProtocol: () => mutationReply("retireProtocol", false, [ { path: "code", message: "código inválido" } ])
    });
    renderTab();

    await user.click(await screen.findByRole("button", { name: "Aposentar" }));
    await user.type(screen.getByLabelText("Código do autenticador"), "123456");
    await user.click(screen.getByRole("button", { name: "Confirmar" }));

    await waitFor(() => expect(calls(fetchMock, "retireProtocol")).toHaveLength(1));
    expect(bodyOf(calls(fetchMock, "retireProtocol")[0]).variables)
      .toEqual({ citySlug: "sp", name: "dengue", version: 1, code: "123456" });
    expect(await screen.findByText("código inválido")).not.toBeNull();
    expect(screen.getByText("Tentativas erradas contam para o bloqueio da conta.")).not.toBeNull();
    expect((screen.getByLabelText("Código do autenticador") as HTMLInputElement).value).toBe("");
  });

  it("recusa de domínio fora de code/reason aparece no painel, que continua aberto", async () => {
    const user = userEvent.setup();
    route([ v({ status: "in_review", publicationMissing: 0 }) ], {
      publishProtocol: () => mutationReply("publishProtocol", false, [ { path: "version", message: "falta 1 assinatura" } ])
    });
    renderTab();

    await user.click(await screen.findByRole("button", { name: "Publicar" }));
    await user.type(screen.getByLabelText("Código do autenticador"), "123456");
    await user.click(screen.getByRole("button", { name: "Confirmar" }));

    expect(await screen.findByText("falta 1 assinatura")).not.toBeNull();
    expect(screen.getByRole("button", { name: "Confirmar" })).not.toBeNull();
  });

  it("reverter pede motivo e código e manda name/reason/code (sem version)", async () => {
    const user = userEvent.setup();
    route([ v({ status: "active", version: 3, revertible: true }) ], {
      revertProtocolActivation: () => mutationReply("revertProtocolActivation", true)
    });
    renderTab();

    await user.click(await screen.findByRole("button", { name: "Reverter" }));
    expect(screen.getByText("volta para a versão ativada antes desta, sem assinatura nova")).not.toBeNull();
    await user.type(screen.getByLabelText("Motivo"), "regra errada em produção");
    await user.type(screen.getByLabelText("Código do autenticador"), "654321");
    await user.click(screen.getByRole("button", { name: "Confirmar" }));

    await waitFor(() => expect(calls(fetchMock, "revertProtocolActivation")).toHaveLength(1));
    expect(bodyOf(calls(fetchMock, "revertProtocolActivation")[0]).variables)
      .toEqual({ citySlug: "sp", name: "dengue", reason: "regra errada em produção", code: "654321" });
  });

  it("erro GraphQL de cidade (data nula) aparece com código e a lista é recarregada", async () => {
    const user = userEvent.setup();
    route([ v({ status: "draft" }) ], {
      submitProtocolForReview: () => reply(200, {
        data: null,
        errors: [ { message: "cidade não respondeu; resultado desconhecido (correlation abc)", extensions: { code: "CITY_UNREACHABLE" } } ]
      })
    });
    renderTab();

    await user.click(await screen.findByRole("button", { name: "Enviar para revisão" }));
    await user.click(screen.getByRole("button", { name: "Confirmar" }));

    expect(await screen.findByText("CITY_UNREACHABLE — cidade não respondeu; resultado desconhecido (correlation abc)")).not.toBeNull();
    await waitFor(() => expect(calls(fetchMock, "query CityProtocolVersions")).toHaveLength(2));
  });

  it("cancelar fecha o painel sem chamar mutation", async () => {
    const user = userEvent.setup();
    route([ v({ status: "draft" }) ]);
    renderTab();

    await user.click(await screen.findByRole("button", { name: "Enviar para revisão" }));
    await user.click(screen.getByRole("button", { name: "Cancelar" }));

    expect(screen.queryByRole("button", { name: "Confirmar" })).toBeNull();
    expect(calls(fetchMock, "submitProtocolForReview")).toHaveLength(0);
  });

  it("cidade inalcançável na leitura mostra o erro do campo", async () => {
    fetchMock.mockImplementation(() => Promise.resolve(reply(200, {
      // protocolVersions é não nulo: o erro do campo nulifica `city` inteira.
      data: { city: null },
      errors: [ { message: "cidade indisponível", path: [ "city", "protocolVersions" ], extensions: { code: "CITY_UNREACHABLE" } } ]
    })));
    renderTab();

    expect(await screen.findByText("CITY_UNREACHABLE — cidade indisponível")).not.toBeNull();
  });

  it("sem versão nenhuma, estado vazio", async () => {
    route([]);
    renderTab();
    expect(await screen.findByText("nenhum protocolo")).not.toBeNull();
  });
});
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `npx vitest run src/screens/ProtocolsTab.test.tsx`
Esperado: FAIL, `Failed to resolve import "./ProtocolsTab"`.

- [ ] **Step 3: Implementar `ProtocolsTab.tsx`**

```tsx
import { useState, type FormEvent } from "react";
import { useMutation, useQuery, useQueryClient } from "@tanstack/react-query";
import { gql } from "../lib/api";
import { graphql } from "../gql";
import { GraphQLRefusal } from "../lib/errors";
import { actionsFor, type ProtocolAction } from "../lib/protocolActions";
import { Panel } from "../components/Panel";
import { Field } from "../components/Field";
import { Button } from "../components/Button";
import { Tag } from "../components/Tag";
import { DataTable } from "../components/DataTable";
import { EmptyState } from "../components/EmptyState";
import { ErrorState } from "../components/ErrorState";
import { STEP_UP_CODE_HELP_TEXT, StepUpCodeError } from "../components/StepUpCode";

// Aba Protocolos (Plano 2 do frontend): ciclo de vida das versões de uma
// cidade. O mantenedor NUNCA assina (ADR-0016) — aqui ele só executa o ato
// quando as assinaturas dos revisores da cidade já existem. O que está
// habilitado é previsão (src/lib/protocolActions.ts); quem decide é a API.
const CityProtocolVersionsQuery = graphql(`
  query CityProtocolVersions($slug: String!) {
    city(slug: $slug) {
      slug
      protocolVersions {
        name version status
        publicationSignatures publicationMissing
        activationSignatures activationMissing
        eligibleReviewers revertible
      }
    }
  }
`);

const SubmitMutation = graphql(`
  mutation SubmitProtocolForReview($citySlug: String!, $name: String!, $version: Int!) {
    submitProtocolForReview(citySlug: $citySlug, name: $name, version: $version) { ok errors { path message } }
  }
`);
const PublishMutation = graphql(`
  mutation PublishProtocol($citySlug: String!, $name: String!, $version: Int!, $code: String!) {
    publishProtocol(citySlug: $citySlug, name: $name, version: $version, code: $code) { ok errors { path message } }
  }
`);
const ActivateMutation = graphql(`
  mutation ActivateProtocol($citySlug: String!, $name: String!, $version: Int!, $code: String!) {
    activateProtocol(citySlug: $citySlug, name: $name, version: $version, code: $code) { ok errors { path message } }
  }
`);
const RetireMutation = graphql(`
  mutation RetireProtocol($citySlug: String!, $name: String!, $version: Int!, $code: String!) {
    retireProtocol(citySlug: $citySlug, name: $name, version: $version, code: $code) { ok errors { path message } }
  }
`);
const RevertMutation = graphql(`
  mutation RevertProtocolActivation($citySlug: String!, $name: String!, $reason: String!, $code: String!) {
    revertProtocolActivation(citySlug: $citySlug, name: $name, reason: $reason, code: $code) { ok errors { path message } }
  }
`);

const STATUS_LABELS: Record<string, string> = {
  draft: "rascunho", in_review: "em revisão", published: "publicada", active: "ativa", retired: "aposentada"
};
const REVERT_NOTICE = "volta para a versão ativada antes desta, sem assinatura nova";
const GENERIC_ERROR = "não foi possível concluir — tente de novo";

type Row = {
  name: string; version: number; status: string;
  publicationSignatures: number; publicationMissing: number;
  activationSignatures: number; activationMissing: number;
  eligibleReviewers: number; revertible: boolean;
};
type Pending = { row: Row; action: ProtocolAction };
type FieldError = { path?: string | null; message: string };
type Payload = { ok: boolean; errors: FieldError[] };

// Uma função por ação: devolve o payload `{ ok, errors }` da mutation.
async function run(slug: string, { row, action }: Pending, code: string, reason: string): Promise<Payload | null> {
  const base = { citySlug: slug, name: row.name, version: row.version };
  switch (action.kind) {
    case "submit": return (await gql(SubmitMutation, base)).data?.submitProtocolForReview ?? null;
    case "publish": return (await gql(PublishMutation, { ...base, code })).data?.publishProtocol ?? null;
    case "activate": return (await gql(ActivateMutation, { ...base, code })).data?.activateProtocol ?? null;
    case "retire": return (await gql(RetireMutation, { ...base, code })).data?.retireProtocol ?? null;
    case "revert":
      return (await gql(RevertMutation, { citySlug: slug, name: row.name, reason, code })).data?.revertProtocolActivation ?? null;
  }
}

export function ProtocolsTab({ slug }: { slug: string }) {
  const queryClient = useQueryClient();
  const queryKey = [ "city", slug, "protocols" ];
  const query = useQuery({ queryKey, queryFn: () => gql(CityProtocolVersionsQuery, { slug }), staleTime: Infinity });

  const [ pending, setPending ] = useState<Pending | null>(null);
  const [ code, setCode ] = useState("");
  const [ reason, setReason ] = useState("");
  const [ errors, setErrors ] = useState<FieldError[]>([]);
  const [ formError, setFormError ] = useState<string | null>(null);
  const [ done, setDone ] = useState<string | null>(null);

  function open(row: Row, action: ProtocolAction) {
    setPending({ row, action });
    setCode(""); setReason(""); setErrors([]); setFormError(null); setDone(null);
  }
  function close() { setPending(null); setCode(""); setReason(""); setErrors([]); setFormError(null); }

  // gcTime 0: o código TOTP vai nas variables — a mutation não fica no cache.
  const mutation = useMutation({
    gcTime: 0,
    mutationFn: (vars: { pending: Pending; code: string; reason: string }) => run(slug, vars.pending, vars.code, vars.reason),
    onSuccess: (payload, { pending: p }) => {
      setCode("");
      if (payload?.ok) {
        close();
        setDone(`${p.action.label} concluído: ${p.row.name} v${p.row.version}`);
        void queryClient.invalidateQueries({ queryKey });
        return;
      }
      const list = payload?.errors ?? [];
      setErrors(list);
      const general = list.filter((e) => e.path !== "code" && e.path !== "reason");
      setFormError(payload ? (general.map((e) => e.message).join(" ") || null) : GENERIC_ERROR);
    },
    onError: (err) => {
      setCode("");
      setErrors([]);
      setFormError(err instanceof GraphQLRefusal ? `${err.code} — ${err.message}` : err instanceof Error ? err.message : GENERIC_ERROR);
      // Pode ter comitado (CITY_UNREACHABLE depois do command): relê.
      void queryClient.invalidateQueries({ queryKey });
    }
  });

  function submit(event: FormEvent) {
    event.preventDefault();
    if (pending) mutation.mutate({ pending, code, reason });
  }

  const codeError = errors.find((e) => e.path === "code");
  const reasonError = errors.find((e) => e.path === "reason");
  const fieldError = query.data?.fieldErrors.find((r) => {
    const path = r.path ?? [];
    return path[0] === "city" && (path.length === 1 || path[1] === "protocolVersions");
  });
  const rows: Row[] = query.data?.data?.city?.protocolVersions ?? [];

  return (
    <div style={{ display: "flex", flexDirection: "column", gap: 12 }}>
      <div><Button onClick={() => void query.refetch()} busy={query.isFetching}>atualizar</Button></div>
      {query.isPending && <p>carregando…</p>}
      {query.isError && <ErrorState message={query.error instanceof Error ? query.error.message : "erro inesperado"} />}
      {fieldError && <ErrorState message={`${fieldError.code} — ${fieldError.message}`} />}
      {done && <p role="status" style={{ margin: 0, fontSize: 12.5 }}>{done}</p>}

      {query.isSuccess && !fieldError && rows.length === 0 && <EmptyState message="nenhum protocolo" />}
      {rows.length > 0 && (
        <DataTable
          rowKey={(row) => `${row.name}-${row.version}`}
          columns={[
            { key: "name", label: "Nome" },
            { key: "version", label: "Versão" },
            { key: "status", label: "Status", render: (row: Row) => <Tag>{STATUS_LABELS[row.status] ?? row.status}</Tag> },
            { key: "publication", label: "Publicação", render: (row: Row) => `${row.publicationSignatures}/2` },
            { key: "activation", label: "Ativação", render: (row: Row) => `${row.activationSignatures}/2` },
            { key: "eligibleReviewers", label: "Revisores" },
            {
              key: "actions", label: "Ações",
              render: (row: Row) => (
                <div style={{ display: "flex", gap: 6, flexWrap: "wrap" }}>
                  {actionsFor(row).map((action) => (
                    <div key={action.kind} style={{ display: "flex", flexDirection: "column", gap: 2 }}>
                      <Button onClick={() => open(row, action)} disabled={action.disabledReason !== null}>{action.label}</Button>
                      {action.disabledReason && <small style={{ fontSize: 11, color: "var(--ink3)" }}>{action.disabledReason}</small>}
                    </div>
                  ))}
                </div>
              )
            }
          ]}
          rows={rows}
        />
      )}

      {pending && (
        <Panel title={`Confirmar: ${pending.action.label} ${pending.row.name} v${pending.row.version}`}>
          <form onSubmit={submit} style={{ display: "flex", flexDirection: "column", gap: 10 }}>
            {pending.action.kind === "revert" && <p style={{ margin: 0, fontSize: 12.5 }}>{REVERT_NOTICE}</p>}
            {formError && <ErrorState message={formError} />}
            {pending.action.needsReason && (
              <>
                <Field label="Motivo" value={reason} onChange={setReason} />
                {reasonError && <ErrorState message={reasonError.message} />}
              </>
            )}
            {pending.action.stepUp && (
              <>
                <Field label="Código do autenticador" value={code} onChange={setCode}
                       autoComplete="one-time-code" helpText={STEP_UP_CODE_HELP_TEXT} />
                {codeError && <StepUpCodeError message={codeError.message} />}
              </>
            )}
            <div style={{ display: "flex", gap: 8 }}>
              <Button type="submit" busy={mutation.isPending}>Confirmar</Button>
              <Button onClick={close}>Cancelar</Button>
            </div>
          </form>
        </Panel>
      )}
    </div>
  );
}
```

Se o `DataTable` não aceitar o genérico `Row` nas funções `render`, ajuste só as anotações de tipo: a lógica não muda.

- [ ] **Step 4: Ligar em `CityDetail`**

Em `src/screens/CityDetail.tsx`:
- remova `CityProtocolsQuery` e o `useQuery` de `protocols`;
- `import { ProtocolsTab } from "./ProtocolsTab";`;
- troque o bloco `{activeTab === "protocols" && renderTab(protocols, ...)}` por `{activeTab === "protocols" && <ProtocolsTab slug={slug} />}`;
- atualize o comentário de topo, que lista as consultas das abas: a aba Protocolos consulta pelo próprio componente, e só quando aberta, porque o componente só monta com a aba ativa.

Em `src/screens/CityDetail.test.tsx`, troque `"CityProtocols"` por `"CityProtocolVersions"` na lista de operações que não devem disparar ao abrir. Se algum teste de lá abre a aba Protocolos, ajuste a resposta para o formato `protocolVersions`.

Depois, `npm run codegen`: a operação removida some de `src/gql`.

- [ ] **Step 5: Rodar e ver passar**

```bash
npm run codegen && npm run typecheck && npm test && npm run build
```
Esperado: tudo verde. O `npm test` inclui os testes antigos de `CityDetail`.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git add src/screens/ProtocolsTab.tsx src/screens/ProtocolsTab.test.tsx src/screens/CityDetail.tsx src/screens/CityDetail.test.tsx src/gql
/opt/homebrew/bin/git commit -m "feat: add protocol lifecycle actions to the city protocols tab" -m "Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

### Task 4: e2e "protocolos" e README

**Files:**
- Modify: `apps/maintenance/e2e/support.ts`, `apps/maintenance/e2e/maintenance.spec.ts`, `apps/maintenance/README.md`

**Interfaces:**
- Consumes: `freshCode` e `inviteMaintainer` (`e2e/support.ts`), a `sharedPage` serial de `maintenance.spec.ts`, e a UI da Task 3.
- Produces: `export function draftDefinition(name: string)` em `e2e/support.ts` (o conteúdo mínimo de rascunho que passa no portão).

**Por que o e2e cobre pouco (Decisão da conversa):** publicar ou ativar de ponta a ponta exige dois revisores assinando numa cidade de dev, e isso é caro de semear e deixa linhas append-only a cada execução. O e2e prova então o caminho real que não pede assinatura (enviar para revisão) e a ação bloqueada que aparece em seguida ("Publicar" desabilitado, com o motivo). O resto fica nos testes da Task 3.

- [ ] **Step 1: Semear um rascunho pelo caminho de domínio**

O rascunho é criado **como mantenedor**, pela mutation `saveProtocolDraft` da própria API, feita de dentro da página logada. Assim não entra nenhum atalho de banco:

```ts
// e2e/support.ts
// Rascunho de protocolo para o teste "protocolos": o conteúdo mínimo que passa
// no portão (o mesmo de apps/api/spec/support/protocol_signatures.rb), com
// nome único por execução. Criado pela mutation saveProtocolDraft na sessão
// do próprio mantenedor, dentro da página — deixa uma linha de contribuição
// append-only no banco de dev a cada execução (custo aceito, como o
// mantenedor e a auditoria; ver README).
export function draftDefinition(name: string) {
  return {
    name, version: 1, start_step_id: "s1",
    steps: [ { id: "s1", prompt: "?", answer_type: "boolean",
               branches: { true: null, false: null }, weights: { true: 1, false: 0 } } ],
    scoring: { type: "weighted", thresholds: { baixa: 0 }, priority_map: { baixa: 9 } }
  };
}
```

Em `maintenance.spec.ts`, um `page.evaluate` chama `fetch("/graphql", …)` com os mesmos headers de `src/lib/api.ts` (`Content-Type: application/json`, `Accept: application/json`, `X-Rota-Maintenance: 1`) e `credentials: "include"`.

- [ ] **Step 2: Escrever o teste**

Troque o import do topo por `import { inviteMaintainer, freshCode, draftDefinition } from "./support";` e, entre "cidades" e "token de serviço", no `describe.serial`:

```ts
  test("protocolos", async ({ sharedPage: page }) => {
    const name = `e2e-${Date.now().toString(36)}`;

    const saved = await page.evaluate(async ({ definition }) => {
      const response = await fetch("/graphql", {
        method: "POST", credentials: "include",
        headers: { "Content-Type": "application/json", "Accept": "application/json", "X-Rota-Maintenance": "1" },
        body: JSON.stringify({
          query: "mutation($citySlug: String!, $definition: JSON!) { saveProtocolDraft(citySlug: $citySlug, definition: $definition) { ok errors { path message } } }",
          variables: { citySlug: "curitiba", definition }
        })
      });
      return (await response.json()).data?.saveProtocolDraft;
    }, { definition: draftDefinition(name) });
    expect(saved?.ok).toBe(true);

    await page.getByRole("button", { name: "Cidades", exact: true }).click();
    await page.locator("tr", { hasText: "curitiba" }).click();
    await page.getByRole("button", { name: "Protocolos", exact: true }).click();

    const row = page.locator("tr", { hasText: name });
    await expect(row.getByText("rascunho")).toBeVisible();

    await row.getByRole("button", { name: "Enviar para revisão" }).click();
    await page.getByRole("button", { name: "Confirmar" }).click();
    await expect(page.getByRole("status").filter({ hasText: `Enviar para revisão concluído: ${name} v1` })).toBeVisible();

    // O mantenedor editou esta versão e nunca assina: sem assinaturas, Publicar
    // fica bloqueado com o motivo (qual dos dois depende dos revisores de dev).
    const refreshed = page.locator("tr", { hasText: name });
    await expect(refreshed.getByText("em revisão")).toBeVisible();
    await expect(refreshed.getByRole("button", { name: "Publicar" })).toBeDisabled();
    await expect(refreshed.getByText(/falta|faltam|revisor\(es\) elegível\(is\)/)).toBeVisible();

    await page.getByRole("button", { name: "voltar" }).click();
  });
```

Confira o slug `curitiba` no teste "cidades" (já usado lá). Atualize o comentário de topo do arquivo, que conta os testes ("Os cinco testes abaixo..."), para seis.

- [ ] **Step 3: Rodar**

Com o stack de dev de pé: `docker compose up -d api worker maintenance` na raiz. O container `api` tem de estar com a branch da Task 1 (o código é montado como volume; confira com `docker compose exec -T api bin/rails runner 'puts Maintenance::Types::ProtocolVersionType'`). Depois:

```bash
npm run e2e
```
Esperado: 6 passed.

- [ ] **Step 4: README**

Em `README.md`, uma seção nova antes de "Schema e codegen":

```markdown
## Protocolos

A aba **Protocolos** do detalhe de cidade lista todas as versões com o estado
das assinaturas (publicação e ativação, N/2) e os revisores elegíveis da
cidade, e oferece só as ações que o status permite: enviar para revisão,
publicar, ativar, aposentar e reverter. Publicar, ativar, aposentar e
reverter pedem o código do autenticador; reverter pede também um motivo.

O mantenedor **nunca assina** (ADR-0016): publicar e ativar só passam quando
dois revisores da cidade — que não editaram a versão — já assinaram pelo
dashboard. Sem isso o botão aparece desabilitado com o motivo. Não há editor
de rascunho aqui; a autoria é do editor do dashboard.
```

E, na seção de E2E, acrescente que cada execução deixa também **um protocolo de rascunho, em revisão, com nome `e2e-…`, em curitiba**, e as linhas append-only de contribuição dele.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git add e2e/support.ts e2e/maintenance.spec.ts README.md
/opt/homebrew/bin/git commit -m "test: cover protocol submission and a blocked publish end to end" -m "Co-Authored-By: Claude Opus 5 <noreply@anthropic.com>"
```

---

## Fora deste plano

- O editor de rascunho no maintenance (escopo B/C). `saveProtocolDraft` segue disponível na API.
- As telas de assinatura do dashboard (os revisores assinam lá). O spec ainda não foi escrito, e o proxy do dashboard ainda não tem `/protocols` nem `/mfa`.
- A mensagem não traduzida da reversão com motivo só de U+00A0 (conserto no command `RevertActivation`).
