# Módulo 15 — Triagem direcionada por perfil — design

**Data:** 2026-10-05
**Status:** aprovado em conversa (2026-10-05), aguardando revisão do texto
**Afeta:**
- `contracts`: schema de protocolo `protocols-v1.4.0` (`offer`, `suggestions`, `gte`/`lte`).
- `apps/api`:
  - banco de cada cidade: colunas de perfil em `citizens`; tabelas `triage_offers` e `triage_suggestions`;
  - linguagem de condição (`gte`, `lte`, variáveis `profile.*` e `outcome.*`), gate e validador;
  - `Triages::Offer` (regra de oferta), `StartTriage` por nome, criação de sugestões na conclusão;
  - rotas novas em `/citizen` (perfil, catálogo, sugestões) e em `/protocols` (catálogo da cidade, simulador).
- `apps/wpda`: perfil obrigatório, catálogo, "Recomendamos também", correção do perfil.
- `apps/dashboard`: construtor visual de condições, simulador, pré-visualização no visual do wpda, aba "Catálogo de triagens", contadores.
- `apps/admin`, `apps/maintenance`: nada.

**ADR:** `docs/adr/0027.md` · **Módulo:** `docs/modulos/15--triagem-por-perfil.md` (a criar)

**Fora desta entrega:** editor visual completo do protocolo (subprojeto seguinte), condição de saúde declarada (módulo 23), perfil vindo do CADSUS (módulo 16), aviso de triagem vencida (módulo 23), recorte por idade/sexo no Analytics, estimativa de público.

## 1. Ponto de partida

- **Um protocolo só para o cidadão.** `StartTriage` abre sempre `DEFAULT_PROTOCOL_NAME = "triage-respiratoria"` (`app/commands/start_triage.rb`), usado pela web (`Citizens::StartConversation`) e pelo WhatsApp descontinuado (`ConversationAdvance`). A cidade cria e assina quantos protocolos quiser, mas o cidadão não escolhe.
- **Cidadão = par (CPF, celular)** (ADR 0017). `citizens`: `cpf`, `phone` (cifrados determinísticos), `neighborhood_id`, `verification_level` (`declared`/`verified`), `erased_at` (ADR 0026). Sem idade nem sexo. A sessão é por celular; `GET /citizen/people` lista os pares do celular (teto de 10).
- **Bairro declarado** já segue o padrão que o perfil vai copiar: `Citizens::SetNeighborhood` (lock no cidadão, evento só com ids, `nil` = prefiro não informar).
- **Linguagem de condição** (`app/protocols/condition.rb`, ADR 0009): árvore JSON com `eq`, `in`, `gt`, `lt`, `all`, `any`, `not`; total (nunca levanta; operando ausente dá falso); avalia sobre `answers` (hash por id de passo). Há o mapa legado `{step_id => valor}`.
- **Schema de protocolo** (`contracts/protocols/schema.json`, `additionalProperties: false`): `name`, `version`, `schema_version`, `start_step_id`, `steps`, `scoring`, `recommendations`, `priority_when`. Última tag: `protocols-v1.3.0`. O `api` valida contra uma cópia em `config/protocols/schema.json`.
- **Resultado** (`Protocols::Outcome`): `tier`, `priority`, `score`, `explanation`. Urgência em `Protocols::Urgency.urgent?(outcome)`.
- **Editor de protocolo** no dashboard: JSON cru com preview (`src/modules/ProtocolEditor.tsx`).
- **Termo de consentimento** é por cidade, publicado pelo operador (`city:consent_term:publish`), texto fora do repositório.
- **Guarda de ADR:** `spec/adr_pointers_spec.rb` tem `VALID_RANGE = (1..26)`.

## 2. Decisões desta conversa

| # | Decisão |
|---|---|
| 1 | Oferta por perfil **e** encadeamento por sugestão. |
| 2 | Perfil declarado agora (`declared`/`verified`), CADSUS depois. |
| 3 | Guardar sexo e identidade de gênero (opcional), como o LEDI; elegibilidade olha só o sexo. |
| 4 | Elegibilidade assinada no protocolo; catálogo da cidade só restringe (soma com E). |
| 5 | Sugestão no resultado, pendente no catálogo; nunca continuação automática. |
| 6 | Intervalo de repetição declarado no protocolo; aviso de vencimento fica para o módulo 23. |
| 7 | Elegibilidade e sugestão escritas na linguagem de condição, com construtor visual. |
| 8 | Perfil do par (CPF, celular), sem mudar o modelo do ADR 0017. |
| 9 | Perfil **obrigatório** no wpda depois de "para quem é?". |
| 10 | Editor visual completo do protocolo é subprojeto próprio, logo em seguida. A pré-visualização deste módulo já usa o visual do wpda. |

## 3. Dados

### 3.1 `citizens` (migração de cidade)

| Coluna | Tipo | Regra |
|---|---|---|
| `birth_date` | text cifrado (`encrypts`, não determinístico) | data ISO; não futura; idade ≤ 130 |
| `sex` | text cifrado | `female` \| `male` (validação no modelo, não `CHECK`, porque o valor no banco é cifrado) |
| `gender_identity` | text cifrado, nulo | valores do Cadastro Individual do e-SUS APS; nulo = não informado |
| `profile_source` | text, nulo | `declared` \| `verified` (`CHECK`); nulo = sem perfil |

`Citizen#age(on: Date.current)` calcula a idade no fuso da aplicação. `Citizen#profile?` = `birth_date` e `sex` presentes.

### 3.2 `triage_offers`

| Coluna | Tipo | Regra |
|---|---|---|
| `protocol_name` | text, único | `name` de protocolo |
| `enabled` | boolean | default `true` |
| `position` | integer | ordem no catálogo |
| `restriction` | jsonb, nulo | condição sobre `profile.*` e `citizen.neighborhood_id` |
| `available_from`, `available_until` | date, nulo | período; `until ≥ from` |
| `updated_by_user_id` | uuid | quem mudou por último |

Mudanças publicam `triage_offer.changed` (só `protocol_name` e ids).

### 3.3 `triage_suggestions`

| Coluna | Tipo | Regra |
|---|---|---|
| `citizen_id` | uuid | o par |
| `source_triage_id` | uuid | triagem que gerou |
| `protocol_name` | text | sugerido |
| `status` | text | `pending` \| `taken` \| `expired` (`CHECK`) |
| `taken_triage_id` | uuid, nulo | triagem iniciada a partir da sugestão |
| `created_at`, `resolved_at` | timestamptz | |

Índice único parcial `(citizen_id, protocol_name) WHERE status = 'pending'`: no máximo uma sugestão pendente por protocolo por par. Transições só `pending → taken` e `pending → expired` (trigger).

## 4. Linguagem de condição e schema

### 4.1 Linguagem (`Protocols::Condition`)

- Operadores novos `gte` e `lte`, com o mesmo guarda de tipo de `gt`/`lt`.
- `eval(node, context)`: o segundo argumento passa a ser um contexto plano que junta `answers` com as variáveis reservadas: `profile.age` (inteiro), `profile.sex`, `outcome.tier`, `outcome.score`, `outcome.priority`, `citizen.neighborhood_id`. A chamada atual com `answers` continua funcionando.
- Prefixos reservados: `profile.`, `outcome.`, `citizen.`. O validador recusa passo com id que comece por eles.
- Continua total: variável ausente → falso.

### 4.2 Schema `protocols-v1.4.0` (MINOR)

```json
{
  "offer": {
    "title": "Saúde do idoso",
    "summary": "Avaliação anual de quedas, memória e medicamentos.",
    "eligibility": { "gte": ["profile.age", 60] },
    "retake_after_days": 365
  },
  "suggestions": [
    { "protocol": "saude-mental-aprofundada",
      "when": { "gte": ["outcome.score", 15] } }
  ]
}
```

- `offer` opcional; dentro dele tudo opcional. `title` ≤ 60 caracteres, `summary` ≤ 200.
- `offer.eligibility` só pode usar `profile.*`. `suggestions[].when` pode usar `profile.*`, `outcome.*` e ids de passo do próprio protocolo.
- `$defs/condition` ganha `gte` e `lte`.
- CHANGELOG no contracts; o `api` atualiza a cópia no mesmo ciclo.

### 4.3 Gate (`Protocols::Gate`)

- Recusa: variável fora do permitido em cada lugar; id de passo inexistente no `when`; sugestão para o próprio `name`; `retake_after_days` ≤ 0.
- Avisa sem bloquear: sugestão para `name` que não existe hoje na cidade.

## 5. Regras

### 5.1 Oferta (`Triages::Offer`, puro)

`Triages::Offer.for(citizen:, on: Date.current)` → lista ordenada de `{ protocol_name, title, summary, state, next_available_on }`, com `state` em `available` ou `recent`. Para cada protocolo `active`:

1. linha do catálogo: sem linha → em oferta só se não houver `offer.eligibility`; com linha → `enabled` e dentro de `available_from..available_until`;
2. `offer.eligibility` verdadeira no contexto do par;
3. `restriction` verdadeira (ausente = verdadeira);
4. intervalo: última triagem `completed` do par naquele `protocol_name` com `created_at` mais antigo que `retake_after_days`; senão `recent`, com `next_available_on`.

A lógica é uma função sobre dados já carregados (protocolos ativos, linhas do catálogo, perfil, datas das últimas triagens), testável em tabela de casos.

### 5.2 Início (`StartTriage`)

`StartTriage.call(conversation:, protocol_name:)`: confere `Triages::Offer` para o par sob `citizen.lock!` (a mesma triagem iniciada duas vezes em paralelo não fura o intervalo); recusa com `:not_offered`. Se veio de sugestão pendente daquele protocolo, marca `taken` e grava `taken_triage_id` na mesma transação. O WhatsApp descontinuado continua chamando com o nome antigo.

### 5.3 Sugestões na conclusão

Depois de `CompleteTriage` concluir: se `Urgency.urgent?(outcome)`, nada. Senão, para cada `suggestions[]` cujo `when` é verdadeiro e cujo protocolo está `available` para o par, cria `triage_suggestions` `pending` (ignora se já houver pendente). Evento `triage.suggested` só com ids.

**Expiração preguiçosa:** a leitura do catálogo marca `expired` toda sugestão pendente cujo protocolo deixou de estar `available` para o par.

### 5.4 Perfil

`Citizens::SetProfile.call(citizen:, birth_date:, sex:, gender_identity:)`, no padrão de `SetNeighborhood`: lock, recusa se `profile_source == "verified"` (`:profile_verified`), valida datas e valores, grava `declared`, evento `citizen.profile_changed` só com `citizen_id`. A validação presencial (`Citizens::Verify.call(cpf:, code:, document_checked:, by:)`) passa a receber também a data de nascimento e o sexo conferidos no documento; grava-os em todos os pares daquele CPF que ela valida, com `profile_source = verified`. A tela de validação do dashboard mostra o que foi declarado para o atendente confirmar ou corrigir.

### 5.5 LGPD

- Exclusão (ADR 0026, `Citizens::Erase`): zera perfil e apaga `triage_suggestions` do par.
- Revogação: apaga as sugestões cuja `source_triage_id` é a triagem revogada; o perfil fica.
- Nenhum evento, log ou URL carrega `birth_date`, idade, `sex` ou `gender_identity`; `filter_parameters` ganha os três campos.

## 6. API

### 6.1 Cidadão (`/citizen`, sessão do cidadão)

| Rota | Faz |
|---|---|
| `GET /citizen/people` (existente) | cada par ganha `profile` (`birth_date`, `sex`, `gender_identity`, `profile_source`) ou `null` |
| `POST /citizen/people/:id/profile` | `SetProfile`, no padrão de `people/:id/neighborhood`; 422 com motivo; 409 `profile_verified` |
| `GET /citizen/people/:id/catalog` | `{ suggested: [...], available: [...], recent: [...] }`; 409 `profile_required` se não houver perfil |
| `POST /citizen/conversations` (existente) | passa a exigir `protocol_name`; 409 `not_offered` |

O resultado da triagem (`GET` já existente) ganha `suggestions: [{ protocol_name, title, summary }]`.

### 6.2 Dashboard (sessão municipal, banco da cidade)

`/protocols/:name` já captura qualquer segmento, por isso o catálogo ganha prefixo próprio.

| Rota | Quem | Faz |
|---|---|---|
| `GET /triage_catalog` | leitores de protocolo | linhas do catálogo + protocolos ativos sem linha |
| `PUT /triage_catalog/:name` | `municipal_admin` + step-up | grava `enabled`, `position`, `restriction`, período |
| `POST /authoring/protocols/simulate_offer` | autores e revisores | `{ definition, profile, answers? }` → `{ eligible, reasons, suggestions }`, sem gravar nada |

## 7. Dashboard

- **Construtor de condições** (componente único): grupos E/OU, linhas campo · operador · valor, NÃO por linha ou grupo; campos por contexto (perfil; + respostas e resultado na sugestão; + bairro na restrição); frase em português sempre visível. Árvore fora do subconjunto do construtor aparece como "regra avançada" em leitura, com a frase.
- **Editor de protocolo:** painel "Oferta e sugestões" ao lado do JSON, que lê e escreve `offer` e `suggestions`. A pré-visualização passa a desenhar as perguntas no visual do wpda (tokens do `contracts`) e ganha o **simulador de perfil**.
- **Aba "Catálogo de triagens"** no módulo Protocolos: uma linha por protocolo ativo (título, elegibilidade em frase, restrição em frase, período, oferecida/pausada, ordem); edição com step-up para o `municipal_admin`.
- **Contadores** por protocolo: oferecida (catálogos montados com ele `available`), iniciada, concluída, vinda de sugestão. Agregados, com a supressão de 1 a 4 do ADR 0025.

## 8. wpda

1. Celular + código → "para quem é?" (como hoje).
2. **Perfil**, se `profile_required`: data de nascimento, sexo, identidade de gênero (opcional, "prefiro não informar"), uma linha de finalidade e link para o termo.
3. **Catálogo**: "Sugeridas para você" (com a origem), "Disponíveis", "Feitas recentemente" (com "próxima a partir de").
4. Triagem (como hoje).
5. Resultado + "Recomendamos também" (Fazer agora / Depois); nunca com resultado urgente.
6. **Meu perfil**: corrigir enquanto `declared`; `verified` mostra "conferido no posto" sem edição.
7. Catálogo vazio: mensagem neutra com o contato da unidade de referência (módulo 11).

## 9. Testes

### 9.1 api (RSpec, TDD)
- `Protocols::Condition`: `gte`/`lte`, contexto com variáveis reservadas, ausente → falso, compatibilidade com a chamada só com `answers`.
- Gate/validador: prefixos reservados, variáveis por lugar, autossugestão, aviso de protocolo inexistente.
- `Triages::Offer`: tabela de casos — bordas de idade (59/60, aniversário hoje), sexo, sem linha com e sem elegibilidade, pausado, período, restrição somada com E, intervalo (`recent` e `next_available_on`), perfil ausente.
- `StartTriage`: `not_offered`, corrida no intervalo (duas threads), sugestão `taken`.
- Sugestões: urgente não sugere, protocolo fora de oferta não sugere, uma pendente por protocolo, expiração preguiçosa.
- `SetProfile`/`Verify`: `verified` não muda pelo cidadão; validações.
- Requests: `profile_required`; dois pares no mesmo celular com catálogos diferentes; mesmo CPF em dois celulares sem cruzar perfil nem sugestão; step-up no catálogo.
- `Erase` e revogação limpando perfil/sugestões.

### 9.2 Suíte de invariante (`spec/invariants/triage_catalog_invariants_spec.rb`)
Os invariantes do ADR 0027, inclusive varredura de payloads de evento e de `filter_parameters`.

### 9.3 dashboard (Vitest)
Construtor ↔ árvore (ida e volta), frase em português, "regra avançada", simulador, aba do catálogo com step-up.

### 9.4 wpda (Vitest, relógio fixo)
Perfil obrigatório, catálogo nas três seções, "Recomendamos também" ausente em urgente, correção do perfil e bloqueio quando `verified`.

### 9.5 Prova no navegador (com o usuário)
Avó (62) e neto (8) no mesmo celular com catálogos diferentes; saúde mental com pontuação alta sugerindo o aprofundamento; cidade pausa o protocolo e a sugestão expira.

## 10. Semente de dev

Dois protocolos novos, assinados pela `SignatureCrew`: "Saúde do idoso" (`gte profile.age 60`, 365 dias) e "Saúde mental" com sugestão para "Saúde mental — aprofundamento" por pontuação. Família no mesmo celular com avó e neto; catálogo de Curitiba com o idoso restrito a dois bairros.

## 11. Rollout

1. contracts `protocols-v1.4.0` (tag; push exige autorização).
2. api: migração de cidade (`city:migrate:all`); catálogo vazio reproduz o comportamento atual; `VALID_RANGE` → `(1..27)`.
3. wpda: perfil obrigatório + catálogo (depende do api).
4. dashboard: construtor, simulador, catálogo.
5. Por cidade, antes de configurar protocolos com elegibilidade: publicar versão nova do termo cobrindo o perfil.

## 12. F-IDs (propostos)

| ID | Funcionalidade | Superfície |
|---|---|---|
| F-15.1 | Perfil do par: nascimento, sexo, identidade de gênero; `declared`/`verified`; obrigatório no wpda | api, wpda, dashboard |
| F-15.2 | Linguagem de condição com `gte`/`lte` e variáveis `profile.*`/`outcome.*`/`citizen.*` | contracts, api |
| F-15.3 | Oferta assinada no protocolo: elegibilidade, título, resumo, intervalo de repetição | contracts, api |
| F-15.4 | Catálogo da cidade: pausar, ordenar, restringir (E), período | api, dashboard |
| F-15.5 | Catálogo do cidadão no wpda e início por protocolo escolhido | api, wpda |
| F-15.6 | Sugestões ao fim da triagem, pendentes no catálogo | contracts, api, wpda |
| F-15.7 | Construtor visual de condições, simulador e pré-visualização no visual do wpda | dashboard |
| F-15.8 | Contadores por protocolo: oferecida, iniciada, concluída, vinda de sugestão | api, dashboard |

## 13. Riscos

- **Barreira do perfil obrigatório** para quem só quer a triagem de sintomas: dois campos, uma vez por par; medir abandono nessa tela pelos contadores.
- **Perfil declarado errado** oferece a triagem errada; o efeito é limitado (o cidadão pode não fazer) e a validação presencial corrige.
- **Sexo como dado sensível de saúde**: cifrado, fora de evento/log/URL, fora do Analytics nesta entrega.
- **Termo de consentimento por cidade** fora do repositório: o sistema não confere se cobre o perfil; fica como passo de rollout.
- **Protocolo sugerido inexistente** na cidade: gate avisa; em execução a sugestão não nasce.
