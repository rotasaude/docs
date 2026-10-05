# Módulo 15 — Contratos entre apps

**Spec:** `superpowers/specs/2026-10-05-module-15-triage-catalog-design.md` · **ADR:** `adr/0027.md`

Fonte única dos formatos que mais de um app usa. Os planos de `contracts`,
`api`, `wpda` e `dashboard` citam este arquivo; se um plano precisar mudar um
formato, muda aqui primeiro.

## 0. Decisões de plano (não estavam na spec)

1. **Pessoa nova com perfil.** Hoje o par só nasce dentro de
   `POST /citizen/conversations` (via `Citizens::RegisterPerson`). Como o
   perfil precisa existir antes do catálogo, nasce `POST /citizen/people`, que
   registra o par **com** o perfil, conferindo antes o consentimento vigente
   (mesma regra LGPD do `conversations#create`: nenhum CPF gravado sem termo).
   `conversations#create` continua aceitando `cpf` (compatibilidade), mas o
   wpda novo sempre manda `citizen_id`.
2. **Uma triagem em andamento por par.** A regra de hoje (uma conversa web
   ativa por cidadão) continua. O catálogo devolve `in_progress`; pedir outro
   protocolo enquanto há um em andamento responde 409 `triage_in_progress`;
   pedir o mesmo protocolo retoma (`resumed: true`, como hoje).

## 1. Schema de protocolo `protocols-v1.4.0`

Acréscimos ao `contracts/protocols/schema.json` (tudo opcional, MINOR):

```json
"offer": {
  "type": "object", "additionalProperties": false,
  "properties": {
    "title":             { "type": "string", "minLength": 1, "maxLength": 60 },
    "summary":           { "type": "string", "minLength": 1, "maxLength": 200 },
    "eligibility":       { "$ref": "#/$defs/condition" },
    "retake_after_days": { "type": "integer", "minimum": 1, "maximum": 3650 }
  }
},
"suggestions": {
  "type": "array", "maxItems": 10,
  "items": {
    "type": "object", "additionalProperties": false, "required": ["protocol", "when"],
    "properties": {
      "protocol": { "type": "string", "pattern": "^[a-z][a-z0-9-]+$" },
      "when":     { "$ref": "#/$defs/condition" }
    }
  }
}
```

`$defs/condition` ganha `gte` e `lte` com o mesmo formato de `gt`/`lt`
(`[operando, valor]`, valor número ou string, como em `gt`/`lt`). Operando de variável reservada é string com prefixo
`profile.`, `outcome.` ou `citizen.`.

Variáveis permitidas por lugar (o schema não distingue; quem confere é o gate
do `api`):

| Lugar | Variáveis |
|---|---|
| `offer.eligibility` | `profile.age`, `profile.sex` |
| `suggestions[].when` | `profile.age`, `profile.sex`, `outcome.tier`, `outcome.score`, `outcome.priority`, ids de passo do protocolo |
| restrição do catálogo (`triage_offers.restriction`) | `profile.age`, `profile.sex`, `citizen.neighborhood_id` |

Valores: `profile.sex` ∈ `female` \| `male`; `profile.age` inteiro;
`citizen.neighborhood_id` uuid (comparado com `in`).

## 2. Valores de perfil

- `sex`: `female` \| `male`.
- `gender_identity` (opcional, Cadastro Individual e-SUS APS):
  `cis_woman` \| `cis_man` \| `trans_woman` \| `trans_man` \| `travesti` \|
  `non_binary` \| `other`; `null` = não informado.
- `birth_date`: `YYYY-MM-DD`, não futura, idade ≤ 130.
- `profile_source`: `declared` \| `verified`.

Rótulos em português (wpda e dashboard): Mulher cis, Homem cis, Mulher trans,
Homem trans, Travesti, Não binária, Outra; "Prefiro não informar" = `null`.

## 3. Cidadão (`/citizen`, cookie `citizen_session`)

Erro sempre `{ "error": "<reason>" }`.

### 3.1 `GET /citizen/people` (existente, ganha `profile`)

```json
{ "people": [ {
  "id": "uuid", "cpf_masked": "***.456.789-**", "verification_level": "declared",
  "neighborhood": { "id": "uuid", "name": "Centro" },
  "profile": { "birth_date": "1963-04-02", "sex": "female",
               "gender_identity": null, "profile_source": "declared" }
} ] }
```
`profile` é `null` quando o par não tem `birth_date` e `sex`.

### 3.2 `POST /citizen/people` (novo)

Corpo: `{ "cpf", "consent_version", "birth_date", "sex", "gender_identity"?, "neighborhood_id"? }`.
- 409 `consent_outdated` (antes de gravar qualquer coisa);
- 422 `invalid_cpf`, `too_many_people`, `invalid_birth_date`, `invalid_sex`,
  `invalid_gender_identity`, `invalid_neighborhood` (motivos de
  `RegisterPerson` passam como estão);
- 200 `{ "person": <pessoa 3.1> }` se o par já existe (perfil não é
  sobrescrito; o wpda usa 3.3 para corrigir); 201 se criou.

### 3.3 `POST /citizen/people/:id/profile` (novo)

Corpo: `{ "birth_date", "sex", "gender_identity" }` (a chave
`gender_identity` pode vir `null`).
- 404 `not_found`; 409 `profile_verified`; 422 `invalid_birth_date`,
  `invalid_sex`, `invalid_gender_identity`;
- 200 `{ "person": <pessoa 3.1> }`.

### 3.4 `GET /citizen/people/:id/catalog` (novo)

- 404 `not_found`; 409 `profile_required`.
- 200:

```json
{
  "in_progress": { "conversation_id": "uuid", "protocol_name": "triage-respiratoria", "title": "Sintomas respiratórios" },
  "suggested": [ { "protocol_name": "saude-mental-aprofundada", "title": "…", "summary": "…",
                   "suggestion_id": "uuid", "source_triage_id": "uuid", "source_title": "Saúde mental",
                   "suggested_on": "2026-10-02" } ],
  "available": [ { "protocol_name": "saude-do-idoso", "title": "…", "summary": "…" } ],
  "recent":    [ { "protocol_name": "saude-do-idoso", "title": "…", "summary": "…",
                   "last_completed_on": "2026-03-10", "next_available_on": "2027-03-10" } ],
  "reference_units": [ "<mesmo formato de reference_units em GET /citizen/triages/:id>" ]
}
```
`reference_units` vem de `Territory::ReferenceUnits.for(citizen.neighborhood_id)`
(bairro **atual** do par) e usa `Territory::ReferenceUnits.as_json_list`; `[]`
sem bairro. `source_title` é o título (regra de `title` abaixo) do protocolo da
triagem de origem.
`in_progress` é `null` sem triagem em andamento. Um protocolo aparece em
`suggested` **ou** em `available`, nunca nos dois. Ordem: `position` do
catálogo, depois `title`. `title` cai para o `name` quando o protocolo não tem
`offer.title`; `summary` pode ser `null`.

### 3.5 `POST /citizen/conversations` (existente, muda)

Corpo ganha `protocol_name` (obrigatório). Erros novos:
422 `protocol_name_required`; 409 `profile_required`, `not_offered`,
`triage_in_progress`. Resposta igual à de hoje.

### 3.6 `GET /citizen/triages/:id` (existente, ganha `suggestions`)

```json
"suggestions": [ { "suggestion_id": "uuid", "protocol_name": "…", "title": "…", "summary": "…" } ]
```
Só as `pending` nascidas desta triagem e ainda em oferta; `[]` em resultado
urgente.

## 4. Dashboard (sessão municipal, banco da cidade)

### 4.1 `GET /triage_catalog`

Papéis: quem lê protocolos hoje (`protocol_author`, `protocol_reviewer`,
`municipal_admin`).

```json
{ "offers": [ {
  "protocol_name": "saude-do-idoso", "title": "Saúde do idoso", "active_version": 3,
  "eligibility": { "gte": ["profile.age", 60] },
  "retake_after_days": 365,
  "configured": true,
  "enabled": true, "position": 2,
  "restriction": { "in": ["citizen.neighborhood_id", ["uuid", "uuid"]] },
  "available_from": null, "available_until": "2026-12-31",
  "counters": { "offered": 120, "started": 40, "completed": 31, "from_suggestion": 6 }
} ] }
```
Um item por protocolo com versão `active`. `configured: false` = sem linha em
`triage_offers` (então `enabled`/`position`/`restriction`/período vêm `null`).
Contadores: últimos 30 dias, cada valor `null` quando entre 1 e 4 (supressão do
ADR 0025).

### 4.2 `PUT /triage_catalog/:protocol_name`

Só `municipal_admin`, com step-up (`MfaStepUp#require_step_up!`, 401
`{ "error": "mfa_required" }` como nas outras rotas). Corpo:
`{ "enabled", "position", "restriction", "available_from", "available_until" }`.
- 404 `unknown_protocol` (nenhuma versão com esse `name`);
- 422 `invalid_restriction` (variável fora de 1, árvore inválida),
  `invalid_period`, `invalid_position` (`position` inteiro ≥ 1);
- 200 `{ "offer": <item 4.1> }`. Evento `triage_offer.changed`
  `{ protocol_name, user_id }`.

### 4.3 `POST /authoring/protocols/simulate_offer`

Papéis do editor (`protocol_author`, `protocol_reviewer`). Não grava nada.

Corpo: `{ "definition": <protocolo JSON>, "profile": { "age": 62, "sex": "female", "neighborhood_id": null }, "answers": { "q1": "sim" }, "outcome": { "tier": "high", "score": 17, "priority": 1 } }`
(`answers` e `outcome` opcionais).

```json
{ "eligible": true,
  "eligibility_text": "idade ≥ 60",
  "suggestions": [ { "protocol": "saude-mental-aprofundada", "matches": true } ],
  "errors": [] }
```
`errors` lista os erros do gate para `offer`/`suggestions`, no mesmo formato do
`POST /protocols/:name/gate` de hoje. Definição inválida responde **200** com
`eligible: false`, `suggestions: []` e os `errors` (nunca 422), para o editor
mostrar o erro ao lado do construtor. `eligibility_text` é gerado no `api` só
para conferência; a frase da tela é do construtor do dashboard.

### 4.4 Validação presencial (existente, muda)

A rota de validação do dashboard (a que chama `Citizens::Verify`) ganha
`birth_date` e `sex` obrigatórios e `gender_identity` opcional, conferidos no
documento. A busca que monta a tela de validação devolve, em `citizen.profile`
(formato 3.1), o perfil declarado do par do código, para o atendente confirmar
ou corrigir. Recusas novas: 422 `invalid_birth_date`, `invalid_sex`,
`invalid_gender_identity`. O perfil conferido é gravado em todos os pares do
CPF que a validação marca `verified`.

**"Idade igual a N"** no construtor vira `all` de `gte` + `lte` com o mesmo N:
`eq` compara texto e `profile.age` é inteiro.

## 5. Eventos de domínio (só ids)

| Nome | Payload |
|---|---|
| `citizen.profile_changed` | `{ citizen_id }` |
| `triage.suggested` | `{ triage_id, suggestion_id, protocol_name }` |
| `triage_offer.changed` | `{ protocol_name, user_id }` |

Nenhum carrega `birth_date`, idade, `sex` ou `gender_identity`.

## 6. Ordem de entrega

1. `contracts` — tag `protocols-v1.4.0` (push exige autorização).
2. `api` — cópia do schema, migração de cidade, rotas acima; `VALID_RANGE`
   `(1..27)`. Porta de dev isolada sugerida: 3032.
3. `wpda` e `dashboard` — depois do `api`, em paralelo.

## 7. Acréscimos da escrita dos planos (2026-10-05)

Incorporados depois de os quatro planos serem escritos; valem sobre as seções
acima.

- **Avisos do gate:** `POST /authoring/protocols/gate` passa a devolver
  `warnings` (só quando há aviso) e `simulate_offer` sempre devolve `warnings`
  (`[]` sem aviso). O único aviso deste módulo: sugestão para protocolo que não
  existe na cidade. O dashboard mostra `warnings` do simulador sem bloquear.
- **`PUT /triage_catalog/:name`:** `enabled` não booleano → 422
  `invalid_enabled`; `restriction` acima de 4096 bytes de JSON → 422
  `invalid_restriction`.
- **`POST /citizen/conversations`:** protocolo inexistente ou inativo pedido por
  um par → 409 `not_offered`; o 503 `no_protocol` fica só para a conversa sem
  cidadão (WhatsApp).
- **Catálogo:** o protocolo em andamento aparece só em `in_progress`, nunca em
  `suggested` ou `available`.
- **`POST /citizen/people` com par existente:** ignora perfil e
  `neighborhood_id`; o wpda usa as rotas próprias de perfil e bairro.
- **Validação presencial:** a validação marca só o par do código (como hoje);
  chave `gender_identity` ausente mantém o declarado, presente (mesmo `null`)
  grava o que veio.
- **Contadores:** "oferecida" soma uma por leitura do catálogo (não por pessoa),
  guardada em `triage_offer_daily_counts` (agregado por dia e protocolo, sem
  coluna de pessoa); `started` conta todos os canais, inclusive revogadas; janela
  = hoje e os 29 dias anteriores, no fuso da cidade.
- **Deploy casado:** o `api` passa a exigir `protocol_name`; o `wpda` novo e o
  `api` novo sobem juntos (nenhuma cidade em produção hoje).
