# Módulo 12 — Campanhas (F-12.1 a F-12.7) — design

**Data:** 2026-09-29
**Status:** aprovado em conversa (2026-09-29), aguardando revisão do texto
**Afeta:**
- `apps/api`:
  - banco de cada cidade: `campaigns`, `campaign_recipients` e `citizen_contact_preferences` novas; coluna `campaigns_sms_enabled` em `city_profile`; triggers de imutabilidade;
  - papel `campaign_manager` (privilegiado);
  - `Campaigns::Audience` e um critério por tipo; `Campaigns::DispatchJob`, `Campaigns::SmsBatchJob` e o recorrente `Campaigns::DueJob`;
  - `SmsGateway` (backends `log`, `test`, `unconfigured`);
  - rotas sob o prefixo único `/campaigns` (dashboard) e novas rotas em `/citizen` (wpda).
- `apps/dashboard`: módulo "Campanhas" (lista, editor com construtor de público, envio, painel da campanha), chave de SMS da cidade para o `municipal_admin`, papel novo em Equipe.
- `apps/wpda`: caixa de avisos, selo de não lidos, preferências (opt-in de SMS e silêncio).
- `apps/admin`: nada.

**ADR:** `docs/adr/0024.md` · **Módulo:** `docs/modulos/12--campanhas.md`

**Fora desta entrega:** grupos "OU" no público, provedor de SMS real e sua tela no maintenance, reenvio de SMS `unavailable`/`failed`, expiração de aviso, anexos e imagens, texto livre no SMS.

## 1. Ponto de partida

- **O stub supunha WhatsApp.** O módulo 12 foi escrito sobre templates aprovados pelo Meta. O WhatsApp foi descontinuado em 2026-09-27; o wpda é o único canal do cidadão.
- **SMS só existe para o código de acesso.** `OtpSender` escolhe o backend por `config.x.otp_sender` (`:log`, `:test`, senão `Unconfigured`, que levanta `Unavailable` e a API responde 503). O provedor real é pendência de go-live.
- **Cidadão:** `citizens` tem `cpf` e `phone` (cifrados, determinísticos), `neighborhood_id` (ADR 0023) e `verification_level`. Não há idade, sexo nem condição de saúde.
- **Sessão do cidadão é por telefone** (`citizen_sessions.phone`); `GET /citizen/people` lista os CPFs daquele telefone. Um telefone pode ter vários cidadãos.
- **Consentimento é por conversa** (`consents.conversation_id`, um ativo por conversa). A revogação aborta a triagem (`aborted_by_revocation`) e o `AnonymizeRevokedTriageJob` apaga o conteúdo clínico.
- **Ligação cidadão → clínico:** `conversations.citizen_id` → `triages.conversation_id`; `attendances.citizen_id` e `attendances.triage_id`; `appointments.citizen_id`; `appointment_requests.citizen_id`.
- **Estados que os critérios usam:**
  - `triages.status`: `in_progress`, `completed`, `aborted_by_revocation`, `aborted_by_timeout`, `aborted_by_cancellation`; `tier` é texto definido pelo protocolo, não enum.
  - `attendances.outcome`: `discharged`, `referred`, `return`, `left`.
  - `appointments.status`: `scheduled`, `confirmed`, `checked_in`, `cancelled_by_citizen`, `expired`, `no_show`. `MarkNoShowAppointmentsJob` só marca `no_show` a partir de `confirmed`.
  - `appointment_requests.status`: `open`, `scheduled`, `closed`; `kind`: `return`, `referral`.
- **Território:** `neighborhoods`, `neighborhood_coverages` (bairro × unidade), `citizens.neighborhood_id`.
- **Papéis:** `Membership::ROLES` e `PRIVILEGED_ROLES`; conceder papel privilegiado pede step-up (`MfaStepUp#require_step_up!`).
- **Jobs:** `EachCityJob`, `CityScopedJob`, `IdempotentConsumer`; recorrentes em `config/recurring.yml`. Fuso: `config.time_zone = "America/Sao_Paulo"` (fixo; ver api#27).
- **`/admin/api`** é só leitura e compartilhado com o console `admin` (ADR 0022). A escrita de campanha fica fora dele.

## 2. Decisões desta conversa

| # | Decisão |
|---|---|
| D1 | Canal: aviso no wpda sempre; SMS opcional, ligado ou desligado pelo `municipal_admin` da cidade. Desenvolver e testar sem provedor. |
| D2 | Consentimento: aviso livre (o cidadão pode silenciar); SMS com opt-in explícito, desligado por padrão. |
| D3 | Público = recorte geográfico (cidade toda, unidade de referência ou bairros) × filtro clínico opcional. |
| D4 | Sete critérios clínicos: protocolo + período, faixa da triagem, triagem não concluída, desfecho de atendimento, triado e não atendido, falta em agendamento confirmado, pedido de agendamento aberto. |
| D5 | Papel novo `campaign_manager`; step-up em enviar, agendar, cancelar e conceder. |
| D6 | SMS sempre com texto fixo + link; o conteúdo só existe no wpda. |
| D7 | Público congelado no envio (`campaign_recipients`). |
| D8 | Mínimo de 5, sempre, com ou sem filtro clínico. |
| D9 | Envio agora ou agendado; SMS só entre 8h e 20h no fuso da cidade. |
| D10 | Construtor A (cartões guiados, critérios combinados por E); o JSON já comporta grupos "OU" depois. |
| D11 | (achado ao conferir o código) "Revogou" = a conversa mais recente do cidadão tem consentimento revogado. |
| D12 | (achado ao conferir o código) O mínimo conta **telefones distintos**; o SMS sai uma vez por telefone por campanha; a caixa do wpda mostra os avisos de todos os cidadãos do telefone da sessão. |

## 3. Dados

Uma migração em `db/city_migrate`, só de expansão e reversível.

### 3.1 `campaigns`

| Coluna | Tipo | Regra |
|---|---|---|
| `id` | uuid | |
| `title` | string | NOT NULL, sem espaços nas pontas, 3–120 |
| `body` | text | NOT NULL, 10–2000; texto simples, quebras de linha preservadas, sem HTML |
| `audience` | jsonb | NOT NULL; validado pelo schema da §4.1 |
| `status` | string | NOT NULL, default `draft`; check: `draft`, `scheduled`, `sending`, `sent`, `cancelled`, `failed` |
| `send_at` | datetime | obrigatório quando `scheduled` (check) |
| `failure_reason` | string | obrigatório quando `failed` (check); hoje só `below_minimum` |
| `sms_enabled` | boolean | NULL até o congelamento; depois, a chave da cidade naquele instante |
| `recipients_count` | integer | NULL até o congelamento |
| `phones_count` | integer | NULL até o congelamento |
| `created_by_user_id` | uuid | NOT NULL, FK `users` |
| `dispatched_by_user_id` | uuid | FK `users`; quem enviou ou agendou |
| `dispatched_at` | datetime | instante do congelamento |
| `cancelled_by_user_id`, `cancelled_at` | | juntos (check) |
| timestamps | | |

Índices: (`status`, `send_at`), (`created_at`).

**Trigger `campaigns_frozen_after_send`:** com `status` em `sent`, `cancelled` ou `failed`, nenhuma coluna muda; em `sending`, só a transição para `sent`/`failed` com os campos do congelamento.

### 3.2 `campaign_recipients`

| Coluna | Tipo | Regra |
|---|---|---|
| `id` | uuid | |
| `campaign_id` | uuid | NOT NULL, FK |
| `citizen_id` | uuid | NOT NULL, FK |
| `notice_read_at` | datetime | |
| `sms_status` | string | NOT NULL; check: `not_opted_in`, `duplicate_phone`, `pending`, `deferred`, `sent`, `failed`, `unavailable` |
| `sms_sent_at` | datetime | |
| `sms_error` | string | até 200; nunca contém o telefone |
| `created_at` | datetime | |

Índice único (`campaign_id`, `citizen_id`); índice (`citizen_id`); índice (`campaign_id`, `sms_status`).

**Trigger `campaign_recipients_append_only`:** `UPDATE` só em `notice_read_at` (de NULL para valor, nunca de volta), `sms_status`, `sms_sent_at`, `sms_error`; `DELETE` permitido (anonimização, §5.6).

### 3.3 `citizen_contact_preferences`

| Coluna | Tipo | Regra |
|---|---|---|
| `citizen_id` | uuid | PK, FK |
| `sms_opt_in` | boolean | NOT NULL, default false |
| `sms_opt_in_changed_at` | datetime | |
| `notices_muted` | boolean | NOT NULL, default false |
| timestamps | | |

Ausência de linha = `sms_opt_in: false`, `notices_muted: false`. Cada mudança publica evento (§5.7).

### 3.4 `city_profile`

`campaigns_sms_enabled` boolean NOT NULL default false.

### 3.5 Papel

`campaign_manager` entra em `Membership::ROLES` e `Membership::PRIVILEGED_ROLES`.

## 4. Público

### 4.1 Formato (`audience`, versão 1)

```json
{
  "version": 1,
  "geo": { "scope": "city" }
      | { "scope": "unit", "health_unit_id": "<uuid>" }
      | { "scope": "neighborhoods", "neighborhood_ids": ["<uuid>", "..."] },
  "clinical": { "all": [ <critério>, ... ] }
}
```

`clinical.all` pode ser vazio. Cada critério tem `kind` e os campos abaixo; períodos são datas (`from`, `to`, inclusivas, no fuso da cidade) com `from <= to` e `to` não no futuro. O schema recusa `kind` desconhecido, campo extra, período invertido, `neighborhood_ids` vazio ou com mais de 50, e mais de 7 critérios.

| `kind` | Campos | Entra quem… |
|---|---|---|
| `protocol_period` | `protocol_name`, `from`, `to` | tem triagem `completed` com esse protocolo, `completed_at` no período |
| `triage_tier` | `tiers` (1+), `from`, `to` | tem triagem `completed` com `tier` na lista, no período |
| `triage_incomplete` | `from`, `to` | tem triagem `aborted_by_timeout` ou `aborted_by_cancellation` criada no período **e** nenhuma triagem `completed` criada depois dela |
| `attendance_outcome` | `outcomes` (1+), `health_unit_id` (opcional), `from`, `to` | tem atendimento `closed` com `outcome` na lista (e na unidade, se dada), `closed_at` no período |
| `triaged_not_attended` | `from`, `to` | tem triagem `completed` no período sem nenhum atendimento com aquele `triage_id` |
| `appointment_no_show` | `from`, `to` | tem agendamento `no_show` com `scheduled_at` no período |
| `appointment_request_open` | `kinds` (opcional: `return`, `referral`), `target_unit_id` (opcional) | tem pedido de agendamento `open` agora |

A triagem chega ao cidadão por `triages.conversation_id → conversations.citizen_id`; triagem sem cidadão (conversa antiga de WhatsApp) não conta.

### 4.2 Recorte geográfico

- `city`: todos os cidadãos.
- `neighborhoods`: `citizens.neighborhood_id` na lista.
- `unit`: `citizens.neighborhood_id` entre os bairros cobertos pela unidade (`neighborhood_coverages`).

Cidadão sem bairro declarado só entra no recorte `city`.

### 4.3 Resolução

`Campaigns::Audience.new(audience).citizen_ids` devolve uma relação SQL (sem carregar cidadãos em Ruby): recorte geográfico ∩ cada critério ∖ revogados. **Revogado** = a conversa mais recente do cidadão (por `created_at`, desempate por `id`) tem consentimento com `revoked_at`.

Cada critério é uma classe em `Campaigns::Criteria::<Kind>` com `.relation(params) → relação de citizen_id`. Critério novo = classe nova + entrada no schema.

`Campaigns::Audience#summary` devolve `{ citizens:, phones: }`, contando telefones distintos pelo `phone` cifrado determinístico (igualdade funciona sem decifrar).

### 4.4 Mínimo

`phones < 5` → a prévia responde `{ below_minimum: true }` sem números; o envio é recusado (422 `below_minimum`); no congelamento, a campanha vai para `failed` com `failure_reason = below_minimum`.

## 5. Fluxo

### 5.1 Estados

```
draft ──enviar──▶ sending ──▶ sent
  │                  └──────▶ failed (below_minimum)
  ├──agendar──▶ scheduled ──(send_at chegou)──▶ sending
  │                 └──cancelar──▶ cancelled
  └──cancelar──▶ cancelled
```

Só `draft` é editável. `scheduled` volta a `draft` por "desagendar" (step-up), para editar.

### 5.2 Enviar e agendar

- **Enviar agora:** step-up; valida o público (`phones >= 5`); `draft → sending`; enfileira `Campaigns::DispatchJob`.
- **Agendar:** step-up; valida o público; `send_at` pelo menos 5 minutos à frente e no máximo 90 dias; `draft → scheduled`.
- **`Campaigns::DueJob`** (recorrente, `EachCityJob`, a cada minuto, fila `default`): pega `scheduled` com `send_at <= agora`, passa a `sending` com `FOR UPDATE SKIP LOCKED` e enfileira o `DispatchJob`.

### 5.3 `Campaigns::DispatchJob`

Numa transação:

1. `FOR UPDATE` na campanha; se não estiver em `sending`, sai (idempotente).
2. Resolve o público. `phones < 5` → `failed`/`below_minimum`, evento, fim.
3. Lê `city_profile.campaigns_sms_enabled` e grava em `sms_enabled`.
4. Insere `campaign_recipients` em lote. `sms_status` de cada um:
   - `sms_enabled = false` → `not_opted_in`;
   - sem opt-in → `not_opted_in`;
   - com opt-in, primeiro cidadão daquele telefone (menor `citizen_id`) → `pending`;
   - com opt-in, demais cidadãos do mesmo telefone → `duplicate_phone`.
5. Grava `recipients_count`, `phones_count`, `dispatched_at`, `status = sent`; publica `campaign.dispatched`.

Depois da transação, se houver `pending`, enfileira `Campaigns::SmsBatchJob`. O aviso no wpda já está visível desde o passo 5.

### 5.4 `Campaigns::SmsBatchJob`

- Pega até 100 `pending`/`deferred` da campanha.
- **Fora da janela** (antes de 8h ou a partir de 20h no fuso da cidade): marca `deferred` e se reagenda para as 8h seguintes.
- **`SmsGateway` não configurado:** todos os `pending`/`deferred` da campanha → `unavailable`; evento `campaign.sms_unavailable` (uma vez); fim.
- Para cada destinatário: confere o opt-in de novo (desligou → `not_opted_in`); envia; `sent` + `sms_sent_at`, ou `failed` + `sms_error` depois de 1 retentativa. Falha de um nunca interrompe o lote.
- Sobrou `pending` → reenfileira a si mesmo.

### 5.5 SMS

- `SmsGateway.deliver(phone:, body:)`, backend por `config.x.sms_gateway` (`:log` em development, `:test` em test, `nil` → `Unconfigured` em production). `Unconfigured.configured?` é `false`; o `deliver` levanta `SmsGateway::Unavailable`.
- `:log` escreve `[sms] <telefone mascarado> <texto>` (mesmo mascaramento do OTP).
- Texto: `Secretaria de Saúde de {city_profile.name}: você tem um aviso novo. Acesse {base do wpda da cidade}/avisos` (a mesma base de `CityPublicUrl`, `CITY_WPDA_BASE_TEMPLATE`). Nada de id no link.
- Argumentos de job nunca carregam telefone: o job recebe ids e decifra dentro.

### 5.6 Revogação

O consumidor de `consent.revoked` passa a apagar as linhas de `campaign_recipients` do cidadão quando a conversa revogada é a mais recente dele. Os contadores gravados na campanha (`recipients_count`, `phones_count`) não mudam; leitura e status de SMS são contados ao vivo, então encolhem.

### 5.7 Eventos de domínio

`campaign.created`, `campaign.scheduled`, `campaign.unscheduled`, `campaign.cancelled`, `campaign.dispatched` (com `recipients_count`, `phones_count`, `sms_enabled` e o `audience`), `campaign.failed`, `campaign.sms_unavailable`, `citizen.contact_preferences_changed` (só `citizen_id` e os booleanos), `city.campaigns_sms_toggled`. Nenhum carrega telefone, CPF ou lista de cidadãos.

## 6. API

### 6.1 Dashboard (`/campaigns`, sessão municipal, banco da cidade)

| Rota | Papel | Step-up | Resposta |
|---|---|---|---|
| `GET /campaigns` | `campaign_manager` | | lista: id, título, status, `send_at`, `dispatched_at`, `recipients_count` |
| `GET /campaigns/options` | `campaign_manager` | | protocolos e faixas já vistos em triagens concluídas, bairros ativos, unidades ativas |
| `POST /campaigns/preview` | `campaign_manager` | | `{ citizens, phones }` ou `{ below_minimum: true }` |
| `POST /campaigns` | `campaign_manager` | | cria `draft` |
| `PATCH /campaigns/:id` | `campaign_manager` | | edita `draft` (outro status → 422 `not_editable`) |
| `GET /campaigns/:id` | `campaign_manager` | | campanha + agregados (§6.3) |
| `POST /campaigns/:id/send` | `campaign_manager` | sim | `draft → sending` |
| `POST /campaigns/:id/schedule` | `campaign_manager` | sim | `draft → scheduled` |
| `POST /campaigns/:id/unschedule` | `campaign_manager` | sim | `scheduled → draft` |
| `POST /campaigns/:id/cancel` | `campaign_manager` | sim | `draft`/`scheduled → cancelled` |
| `GET /campaigns/sms_setting` | `campaign_manager`, `municipal_admin` | | `{ enabled, gateway_configured }` |
| `PUT /campaigns/sms_setting` | `municipal_admin` | sim | liga/desliga; ligar com gateway não configurado é permitido e a resposta traz `gateway_configured: false` |

Recusas: sem papel → 403 `missing_role`; step-up ausente → o 401 padrão do `MfaStepUp`; público inválido → 422 com o caminho do campo; mínimo → 422 `below_minimum`.

### 6.2 Cidadão (`/citizen`, sessão do cidadão)

| Rota | Resposta |
|---|---|
| `GET /citizen/notices` | avisos de todos os cidadãos do telefone da sessão, mais novo primeiro: id (do `campaign_recipient`), título, texto, `dispatched_at`, `read`, nome da pessoa quando o telefone tem mais de um cidadão; mais `unread_count` (0 se todos os cidadãos do telefone silenciaram) |
| `POST /citizen/notices/:id/read` | marca lido; id de outro telefone → 404 |
| `GET /citizen/contact_preferences` | por pessoa do telefone: `sms_opt_in`, `notices_muted`; mais `sms_available` (chave da cidade) |
| `PUT /citizen/contact_preferences/:citizen_id` | altera; cidadão de outro telefone → 404 |

### 6.3 Agregados do painel da campanha

`recipients_count`, `phones_count`, `read_count` (cidadãos com `notice_read_at`), contagem por `sms_status`, `sms_enabled`, `audience`. Nenhuma lista de destinatários, em nenhuma rota.

## 7. Dashboard

- **Menu "Campanhas"** visível para `campaign_manager` e `municipal_admin`. O `municipal_admin` sem o papel vê só a chave de SMS.
- **Lista:** título, status (etiqueta), envio (data ou "agendada para…"), destinatários. Botão "Nova campanha".
- **Editor** (só `draft`):
  - título, texto e pré-visualização "como aparece no wpda";
  - **Público — 1. Recorte:** cidade toda / unidade de referência (seletor de unidade) / bairros (seleção múltipla);
  - **Público — 2. Critérios clínicos:** botão "Adicionar critério" abre a lista dos 7; cada um vira cartão com os próprios campos e um "remover"; entre cartões, "e também";
  - **Contador** ao vivo (`POST /campaigns/preview`, debounce de 500 ms): "≈ N pessoas (M telefones)" ou "menos de 5 — ajuste o público";
  - linha informativa "SMS nesta cidade: ligado/desligado";
  - salvar rascunho.
- **Enviar:** modal com o público em frase ("moradores de Boqueirão e Xaxim que faltaram a um agendamento entre 01/07 e 30/09"), contagem, canais, "agora" ou "agendar para", step-up, confirmar. Recusa `below_minimum` volta ao editor com a mensagem.
- **Painel da campanha** (depois do envio): destinatários, telefones, lidos (%), SMS por status; alerta destacado quando houver `unavailable` ("SMS não enviado: a plataforma ainda não tem provedor de SMS") ou `failed`; o público congelado em frase; status `failed` mostra o motivo.
- **Chave de SMS** (`municipal_admin`): interruptor com step-up; com gateway não configurado, aviso "A plataforma ainda não tem provedor de SMS: os avisos saem, o SMS fica pendente como não enviado".
- **Equipe:** `campaign_manager` entre os papéis concedíveis, com step-up.

## 8. wpda

- **Início:** selo com `unread_count` num link "Avisos" (some quando 0).
- **`/avisos`:** lista (título, data, "novo"); tocar abre o texto e marca lido. Com mais de uma pessoa no telefone, o nome da pessoa aparece em cada aviso. Vazio: "Nenhum aviso da Secretaria por enquanto."
- **`/preferencias`:** por pessoa do telefone, "Receber avisos por SMS" (só quando `sms_available`) com a explicação "A Secretaria de Saúde pode enviar um SMS avisando que há um aviso novo aqui. Você pode desligar quando quiser.", e "Silenciar avisos" (tira o selo; os avisos continuam na lista).
- **Link do SMS:** `/wpda/avisos` sem sessão passa pelo login (CPF + código) e volta a `/avisos`.
- Regras da casa: texto ≥ 18px, alvos ≥ 48px, links com `import.meta.env.BASE_URL`.

## 9. Testes

### 9.1 api (RSpec, TDD)

- **Critérios:** um spec por `kind`, cada um com: entra, não entra, limite do período (bordas inclusivas no fuso da cidade), triagem sem cidadão ignorada.
- **Audience:** os 3 recortes; cidadão sem bairro só em `city`; interseção de 2+ critérios; revogado excluído (e não excluído quando a revogação é de uma conversa antiga); `phones` conta telefone compartilhado uma vez; 4 telefones → `below_minimum`, 5 → ok.
- **Schema:** cada recusa da §4.1.
- **DispatchJob:** idempotência (2 execuções = 1 congelamento); público que encolheu → `failed`; os 4 casos de `sms_status`; chave da cidade gravada no instante do congelamento.
- **SmsBatchJob:** relógio fixo antes das 8h, às 20h e dentro da janela; `unconfigured` → `unavailable` e 1 evento; falha isolada; opt-out tardio; reenfileiramento.
- **DueJob:** pega só o que venceu; duas execuções concorrentes não enviam em dobro.
- **Request (`type: :request`):** papel e step-up de cada rota da §6.1; transições inválidas; rotas do cidadão escopadas por telefone (404 cruzado); `unread_count` com silêncio.
- **Revogação:** consumidor apaga as linhas de destinatário.
- **Guardas:** `R18_PLATFORM_EVENT_NAMES` (se algum evento for de plataforma), `VALID_RANGE` 1..24 no `spec/adr_pointers_spec.rb`.

### 9.2 Suíte de invariante

`spec/invariants/campaign_invariants_spec.rb`, uma seção por invariante do ADR 0024 (mínimo, revogado, opt-in, texto fixo, nenhuma lista no dashboard, imutabilidade pelos triggers, anonimização, eventos sem PII).

### 9.3 dashboard (Vitest)

Construtor (adicionar e remover cartão, validação por tipo, contador com "menos de 5" travando o envio, debounce), frase do público, modal com step-up, painel com alerta `unavailable`, chave de SMS com aviso de gateway, papel em Equipe.

### 9.4 wpda (Vitest)

Selo e silêncio, marcar lido, várias pessoas no telefone, preferência de SMS escondida sem `sms_available`, retorno do login a `/avisos`. Relógio fixo em `beforeEach`.

### 9.5 Prova no navegador (com o usuário)

Criar campanha por bairro + falta → prévia → enviar com step-up; no wpda, entrar como cidadão da semente e ler o aviso; com a chave ligada e `:log`, ver `[sms]` no log do api; com `config.x.sms_gateway = nil`, ver `unavailable` no painel.

## 10. Semente de dev

`campanhas@curitiba.demo` com `campaign_manager` (TOTP fixo como os demais); cidadãos com opt-in de SMS; casos para cada critério (faltas, triagens abandonadas, pedidos abertos, atendidos com cada desfecho) em pelo menos dois bairros, coerentes com a semente realista; um telefone com dois CPFs.

## 11. Rollout

Runbook `operacao/rollout-campanhas.md`:

1. Migração de cidade só aditiva; `city:migrate:all` da imagem nova antes do tráfego.
2. `campaigns_sms_enabled` nasce `false` e `config.x.sms_gateway` nasce `nil` em produção: o deploy não envia SMS.
3. Ordem: api → dashboard → wpda.
4. Gate de go-live do SMS: contratar provedor, escrever o backend, testar em staging, só então as cidades ligam a chave.

## 12. F-IDs

| F-ID | Funcionalidade | Apps |
|---|---|---|
| F-12.1 | Papel `campaign_manager` e chave de SMS por cidade | api, dashboard |
| F-12.2 | Construtor de público: 3 recortes, 7 critérios, mínimo de 5, prévia | api, dashboard |
| F-12.3 | Criar, agendar, desagendar, cancelar e enviar com step-up | api, dashboard |
| F-12.4 | Público congelado e aviso no wpda (caixa, selo, lido) | api, wpda |
| F-12.5 | Preferências do cidadão (opt-in de SMS, silenciar) | api, wpda |
| F-12.6 | Envio de SMS (gateway plugável, janela 8h–20h, estado por destinatário) | api |
| F-12.7 | Painel da campanha (agregados, alertas de SMS) | api, dashboard |

## 13. Riscos

- **Subtração entre campanhas:** duas campanhas com públicos quase iguais podem, pela diferença das contagens, revelar um grupo menor que 5. Aceito no Ciclo 1 (quem monta já é papel privilegiado e cada envio fica na trilha); a rever com o módulo 14.
- **Fuso fixo** em `America/Sao_Paulo` (api#27): a janela do SMS herda o problema.
- **Chave ligada sem provedor:** tudo vira `unavailable`, sem reenvio no Ciclo 1; o painel avisa.
- **`tier` é texto do protocolo:** mudar o nome da faixa num protocolo novo separa as triagens antigas e novas no critério.
