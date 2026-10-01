# Revogação depois da triagem concluída e exclusão do cadastro (Art. 18)

- **Data:** 2026-10-01
- **ADR:** [0026](../../adr/0026.md)
- **Resolve:** rotasaude/api#30, rotasaude/docs#2
- **Apps:** api, dashboard
- **Módulos:** 07 (LGPD/Auditoria), toca 03 (triagem), 06 (papéis), 13 (atendimento)

## 1. Objetivo

1. A revogação passa a anonimizar também a triagem **concluída** que não virou
   atendimento.
2. A trilha de eventos deixa de guardar respostas cruas.
3. O cidadão pode ter o cadastro excluído no posto, por dois servidores, com a
   casca no lugar do dado.

## 2. Estado de partida (origin/main do api)

- `RevokeConsent` aborta só a triagem `in_progress` e publica `consent.revoked`
  com `reason` = texto do cidadão ou `"citizen_web"`.
- `AnonymizeRevokedTriageJob` limpa só `status: aborted_by_revocation`, e o
  trigger `triages_neighborhood_immutable` só aceita bairro nulo nesse status.
- `triage.completed` e `triage.urgent` publicam `outcome.to_h`, com `trail`
  (cada resposta). `GenerateReportJob#handle(triage_id:, **outcome)` usa esse
  payload.
- `Attendances::CheckInEligibility` aceita triagem web `completed` há até 3
  dias, sem atendimento. Não olha consentimento.
- Não há caminho de exclusão. `consents`, `attendances`, `appointments`,
  `appointment_requests` e `citizen_verifications` recusam DELETE; as FKs são
  NO ACTION.

## 3. Revogação

### 3.1 Schema

Migração de cidade `<ts>_add_anonymized_at_to_triages`:

- `triages.anonymized_at timestamptz NULL`.
- `db/city_triggers.sql`, `rota_triage_neighborhood_guard`: bairro pode ir a
  nulo quando `NEW.status = 'aborted_by_revocation'` **ou**
  `NEW.anonymized_at IS NOT NULL`. Nada mais muda.

### 3.2 `AnonymizeRevokedTriageJob`

Para a conversa revogada, sob `Triage.lock` por linha:

- triagens `aborted_by_revocation`, como hoje;
- triagens `completed` **sem** `Attendance` com aquele `triage_id`.

Em cada uma: `answers: {}`, `outcome: nil`, `tier: nil`, `priority: nil`,
`current_step: nil`, `neighborhood_id: nil`, `anonymized_at: now`. Triagem com
atendimento não é tocada. Idempotente (já anonimizada é pulada). Continua
chamando `Campaigns::ForgetRevokedRecipients`.

### 3.3 Check-in

`CheckInEligibility.check` e `.eligible_for` recusam triagem com
`anonymized_at` preenchido **ou** cuja conversa está `revoked`
(`:triage_not_eligible`). Isso fecha a corrida: depois do COMMIT da revogação o
check-in já recusa, antes de o job rodar. O check-in trava a triagem
(`lock`) antes de criar o atendimento; o job trava a mesma linha antes de
anonimizar e reconfere o atendimento.

### 3.4 Payload dos eventos

- `CompleteTriage`: `DomainEvents.publish("triage.completed", triage_id:)` e o
  mesmo para `triage.urgent`.
- `GenerateReportJob#handle(triage_id:, **)`: monta o `outcome` do relatório a
  partir de `triage.outcome`.
- `RevokeConsent`: `reason` vira a origem (`"web"`, `"whatsapp"` ou
  `"erasure"`); o texto do cidadão decide a revogação mas não vai para a
  trilha.
- Eventos já gravados com `trail` ficam até a purga de 12 meses (não há
  produção; em dev, nada a migrar).

### 3.5 Relatório

Sem mudança: segue `expires_at` e a purga.

### 3.6 Termo de consentimento

Texto novo (parágrafo a acrescentar ao termo de cada cidade, publicado por
`city:consent_term:publish`):

> Se você revogar este consentimento, apagamos as respostas e a classificação
> das suas triagens. A triagem que já levou a um atendimento no posto de saúde
> fica guardada, porque passou a fazer parte do seu registro de atendimento.

## 4. Exclusão do cadastro

### 4.1 Schema

Migração de cidade `<ts>_create_citizen_erasure_requests`:

| Coluna | Tipo | Nota |
|---|---|---|
| `id` | uuid | |
| `cpf` | string, NOT NULL | cifrado determinístico (chave da cidade), para achar os pares |
| `presented_citizen_id` | uuid, FK citizens | o par mostrado no posto |
| `requested_by_user_id` | uuid, FK users, NOT NULL | o `citizen_verifier` |
| `document_checked` | boolean, NOT NULL | precisa ser `true` |
| `status` | string, NOT NULL | `pending`, `confirmed`, `rejected`, `retained` |
| `decided_by_user_id` | uuid, FK users | |
| `decided_at` | timestamptz | |
| `reject_reason` | text | obrigatório em `rejected`, ≥ 10 caracteres |
| `created_at`, `updated_at` | | |

- Check: `status = 'pending'` ⇔ `decided_at IS NULL`; `document_checked`.
- Trigger `citizen_erasure_requests_guard`: DELETE e TRUNCATE recusados;
  UPDATE só a partir de `pending`, e só nas colunas de decisão.
- Na confirmação, o `cpf` do pedido também vira o marcador (§4.4): o pedido
  confirmado não guarda o CPF de ninguém. O trigger aceita isso só na mesma
  mudança para `confirmed`.
- `citizens.erased_at timestamptz NULL` na mesma migração.
- `CityEncryption::CITY_KEYED_TARGETS` ganha `[CitizenErasureRequest, :cpf]`.

### 4.2 Papéis

- Pedir: `citizen_verifier`.
- Decidir: `municipal_admin`, com step-up (`require_step_up`, ADR 0011), e
  diferente de `requested_by_user_id`.

### 4.3 Endpoints (`/admin/api`, sessão da cidade)

| Método | Rota | Quem | O que faz |
|---|---|---|---|
| POST | `/attendance/erasure_requests` | `citizen_verifier` | `{cpf, document_checked: true}`. Acha os pares pelo CPF. Sem par: 404 `citizen_not_found`. Algum par com atendimento: cria o pedido já `retained` e responde 201 com `status: "retained"`. Senão, 201 `pending`. Pedido `pending` do mesmo CPF já aberto: 409 `already_pending`. |
| GET | `/erasure_requests?status=pending` | `municipal_admin` | Lista com data, quem pediu, nº de pares e celular mascarado do par apresentado. Nunca o CPF inteiro. |
| POST | `/erasure_requests/:id/confirm` | `municipal_admin` + step-up | Executa §4.4. |
| POST | `/erasure_requests/:id/reject` | `municipal_admin` | `{reason}`. |

Todos os atos publicam evento só com ids:
`citizen.erasure_requested`, `citizen.erasure_retained`,
`citizen.erasure_rejected`, `citizen.erased`.

### 4.4 Confirmação (`Citizens::Erase`)

Numa transação, com `lock` no pedido e nos cidadãos do CPF:

1. Reconfere: pedido `pending`, quem confirma ≠ quem pediu. Se algum par
   tem atendimento agora: pedido vira `retained`, nada mais muda.
2. Para cada par (conversas do par = `citizen_id` do par **ou** `phone` do
   par, que cobre as do WhatsApp, sem `citizen_id`):
   - revoga cada consentimento ativo das conversas dele (`RevokeConsent`, que
     publica `consent.revoked` com `reason: "erasure"`);
   - anonimiza **todas** as triagens das conversas (mesma limpeza de §3.2,
     síncrona; nenhuma tem atendimento);
   - apaga `citizen_sessions` e `otp_challenges` do telefone,
     `citizen_verification_codes`, `citizen_contact_preferences`,
     `campaign_recipients` do cidadão, e `inbound_messages`/`outbound_messages`
     do telefone;
   - `conversations.phone` vira o marcador;
   - `citizens.cpf` e `citizens.phone` viram o marcador, `neighborhood_id`
     nulo, `erased_at` preenchido.
3. Pedido: `confirmed`, `decided_*`, `cpf` = marcador.

**Marcador:** `"erased:" + SecureRandom.uuid`, cifrado como qualquer valor
(determinístico). É único, então respeita `index_citizens_on_cpf_and_phone`, e
não casa com CPF ou telefone de ninguém.

**Depois:** login com o mesmo CPF e celular cria cidadão novo (o par antigo não
casa mais). Lookups ignoram `erased_at`.

### 4.5 Retido por base legal

Ficam: `consents` (revogados), `citizen_verifications`, `attendances`,
`appointments` e `appointment_requests` (só existem com atendimento, então não
coexistem com exclusão), `domain_events`, agregados do Analytics e
`report_snapshots` até a validade.

## 5. Dashboard

- **Atendimento:** ação "Pedir exclusão do cadastro" com CPF, caixa "Conferi o
  documento com foto" e a resposta (pendente ou retido, com o motivo).
- **Pedidos de exclusão** (só `municipal_admin`): lista dos pendentes,
  "Confirmar exclusão" (abre o step-up e uma confirmação que diz que é
  irreversível) e "Recusar" com motivo.
- Vitest nas duas telas.

## 6. Testes

**Invariante (api):**

- revogação: concluída sem atendimento fica sem resposta, classificação,
  prioridade e bairro, com `anonymized_at`; com atendimento fica intacta;
- anonimizada ou de conversa revogada não recebe check-in;
- payload de `triage.completed`/`urgent` só tem `triage_id`;
- `consent.revoked` não guarda texto do cidadão;
- CPF com par atendido nunca é excluído, nem na reconferência da confirmação;
- confirmar exige step-up e pessoa diferente de quem pediu;
- depois da exclusão, nenhuma tabela da cidade decifra o CPF ou o telefone do
  cidadão (varredura por `CITY_KEYED_TARGETS` e pelas colunas de telefone);
- `citizen_erasure_requests` recusa DELETE, TRUNCATE e segunda decisão.

**Comportamento:** pedido → confirmação → casca; pedido → recusa; pedido já
pendente; CPF sem cadastro; novo login depois da exclusão.

**Regressão:** relatório gerado a partir de `triage.outcome`; Analytics segue
contando a concluída revogada como revogada.

## 7. Rollout

1. Deploy do api com as migrações (`city:migrate:all`).
2. Deploy do dashboard.
3. Publicar a versão nova do termo em cada cidade.
4. Avisar: as migrações de cidade novas deixam os bancos de teste à frente de
   outras branches.

## 8. Fora do escopo (pendências de ciclo no board #2)

- Purga de `citizen_sessions`, `otp_challenges` e `outbound_messages`.
- Texto livre congelado em atendimento, agendamento e pedido.
- Revogação pelo WhatsApp depois de a conversa terminar.
