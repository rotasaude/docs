# Módulo 26a — Contratos entre apps

**Spec:** `superpowers/specs/2026-10-28-module-26a-care-regulation-design.md` · **ADR:** `adr/0044.md`

Fonte única dos formatos entre `api`, `dashboard`, `wpda` e `maintenance`.
Convenções herdadas (contratos do 21a §15, 21b §14, 21c §11, 22 §12–§13, 23 §11–§12,
24a §14 e 25 §15): erros `{ "error" }`; step-up; interruptor 403 `feature_disabled`;
JSON em toda escrita por cookie; listas com envelope e `next_before` (cursor opaco
quando a ordem não é por id); registro de outro escopo por id → 404, lista com escopo
alheio → 403; leitura nominal com trilha e sem polling; contadores sem trilha; texto
longo recusado; dado sensível (justificativa, motivos, CID) só no corpo, nunca na URL
nem no log; todo Data/Struct com dado pessoal sem dado em `inspect`/`to_s`/
`pretty_print`/`as_json`; paciente sem CPF como no 24a (`has_cpf`, `cns_masked`).

## 1. Nomes no código

| Spec | Código |
|---|---|
| interruptor | `care_regulation` (`requires: ["feature:clinical_record"]`) |
| papel | `care_regulator` (privilegiado, step-up para conceder) |
| tipo do pedido | `specialty` (CBO) \| `procedure` (SIGTAP) |
| cor de risco | `red` \| `yellow` \| `green` \| `blue` (prazos padrão 7/30/90/180 dias) |
| situação | `draft` (rascunho na consulta) \| `pending` \| `returned` \| `authorized` \| `scheduled` \| `completed` \| `missed` \| `denied` \| `cancelled` \| `closed` (segunda falta) \| `external` (regulado pelo estado) |
| tipo de prestador | `own` (unidade da rede) \| `contracted` \| `consortium` \| `ppi` |
| sistema externo | `sisreg` \| `care` \| `cross` \| `other` |
| quem pede | médico `225*`; enfermeiro `2235*` (só a lista municipal); dentista `2232*` (só especialidades `2232*`) |

## 2. Formas

`<regulation_request>` (nominal; trilha): `{ "id", "patient": { "id", "display_name", "has_cpf", "cns_masked"|null }, "type", "target": { "code", "name" }, "cid10", "justification", "suggested_color", "color"|null, "color_reason"|null, "status", "requested_at", "due_on", "overdue": bool, "requester": { "name", "cbo", "unit": { "id", "name" } }, "consultation_id", "authorization": { "offer_id", "provider": { "id", "name", "kind" }, "month", "skip_reason"|null, "at", "by": { "name" } }|null, "appointment_request_id"|null, "schedule": { "at", "place", "notes"|null }|null, "outcome"|null, "misses", "external": { "system", "number" }|null, "state_regulated": bool, "version", "events": [ { "status", "at", "by": { "name" }|null, "reason"|null } ] }`.

`<offer>`: `{ "id", "provider": { "id", "name", "kind" }, "target": { "type", "code", "name" }, "month": "YYYY-MM", "quantity", "used", "available" }`.

## 3. Na consulta (`health_professional` na consulta)

- `GET/POST /attendance/consultations/:id/regulation_requests`; `PATCH/DELETE .../regulation_requests/:rid` só em `draft`. Corpo: `{ "type", "target_code", "cid10", "justification", "suggested_color", "state_regulated"? }`. 403 `cbo_not_allowed` (§1); 422 `invalid_target`, `invalid_cid10`, `invalid_request` (com `path`).
- Finalizar a consulta (ou adendo) passa os `draft` para `pending` (ou `external` quando `state_regulated`), na mesma transação.
- `GET /regulation/requests/:id/guide.pdf` — guia de encaminhamento (solicitante e regulador; trilha; sem cache; `Content-Disposition` sem dado pessoal).

## 4. Fila do regulador (`care_regulator`)

- `GET /regulation/requests?status=&type=&target_code=&color=&overdue=&before=&limit=` → `{ "requests": [<regulation_request> sem `events`], "next_before" }` (ordem: cor, depois data do pedido; trilha uma por consulta).
- `GET /regulation/requests/:id` → `<regulation_request>`.
- `POST /regulation/requests/:id/authorize` — `{ "offer_id", "color"?, "color_reason"?, "skip_reason"?, "version" }`; 409 `quota_exhausted`, 422 `skip_reason_required` (há pedido elegível mais antigo de cor igual ou mais grave), 422 `color_reason_required`, 409 `stale_version`. Oferta própria cria pedido de agendamento do 17 (`kind: regulated`) na unidade.
- `POST /regulation/requests/:id/return` — `{ "reason", "version" }`; `deny` — `{ "reason", "version" }` (só antes de autorizar); `cancel` — `{ "reason", "version" }` (devolve a cota se ainda não atendido).
- `GET /regulation/requests/count` → `{ "pending", "overdue", "returned", "authorized", "external" }` (sem trilha).

## 5. Solicitante (unidade que pediu)

- `GET /regulation/returned?unit_id=` → devolvidos da unidade (trilha).
- `POST /regulation/requests/:id/complement` — `{ "justification", "cid10"?, "suggested_color"?, "version" }` → volta a `pending` mantendo a data do pedido.

## 6. Prestadores e ofertas (`municipal_admin`, escrita com step-up; leitura também `care_regulator`)

- `GET/POST/PATCH /regulation/admin/providers[/:id]` — `{ "name", "cnes", "kind", "health_unit_id"? (só `own`), "active" }`.
- `GET/POST/PATCH /regulation/admin/offers[/:id]` — `{ "provider_id", "target_type", "target_code", "month", "quantity" }`; `GET ?month=&target_code=` devolve `<offer>` com `used`/`available`. 422 `quantity_below_used`.
- `GET/PATCH /regulation/admin/deadlines` — `{ "red", "yellow", "green", "blue" }` (dias).
- `GET/POST/DELETE /regulation/admin/nurse_items` — `{ "target_type", "target_code" }`.

## 7. Agendamento e desfecho

- Rede própria: o pedido segue o pedido de agendamento do 17 (horário marcado → `scheduled`; atendimento → `completed`; falta/lapse → `missed`).
- Externo: `POST /regulation/requests/:id/schedule` — `{ "at", "place", "notes"?, "version" }`; `POST /regulation/requests/:id/outcome` — `{ "result": "completed"|"missed", "version" }`.
- Falta: primeira → volta a `pending` com `misses: 1` (cota não devolvida); segunda → `closed`.

## 8. Convivência com o estado (`care_regulator`)

- `POST /regulation/requests/:id/external` — `{ "system", "number", "version" }`.
- `external`: não entra na fila nem consome cota; o regulador lança `schedule` e `outcome`.
- `GET /regulation/exports/state.csv?status=` — CSV livre para digitar no sistema do estado (trilha; CNS só 4 últimos dígitos; sem CPF).

## 9. Cidadão (wpda, par validado)

`GET /citizen/regulation_requests?citizen_id=` → `{ "requests": [ { "id", "label", "status": "waiting"|"scheduled"|"returned"|"denied"|"closed"|"completed", "scheduled_at"|null, "place"|null, "updated_at" } ] }` — sem cor, posição, CID ou justificativa; 403 `not_verified`.

## 10. Maintenance

Interruptor `care_regulation` (rótulo e dependência, mecanismo genérico).

## 11. Eventos (só ids)

`regulation_request.created`, `.authorized`, `.returned`, `.denied`, `.cancelled`, `.scheduled`, `.completed`, `.missed`, `.closed`, `care_offer.changed`, `care_provider.changed`.

## 12. Ordem de entrega

0. Catálogo de alvos reguláveis (plano do api, Task 0): especialidades médicas e odontológicas (CBO 2002, `225*` e `2232*`) e procedimentos SIGTAP reguláveis (grupos 02, 03 e 04, sem os de APAC/OCI), conferidos com as tabelas já vendorizadas.
1. `api` (porta 3048; migrações de cidade a partir de 20261028900001; worktree `.claude/mod26a`, branch `feat/mod-26a-care-regulation`; testes com `bin/test api -w mod26a`).
2. `dashboard` (Vite 5203) e `wpda` (Vite 5204), worktrees em `rota-saude/.worktrees/<app>-mod26a`.
3. `maintenance` (prova na 5177 com o serviço do compose parado).
4. Catálogo público.

## 13. Acréscimos (revisão dos planos, 2026-10-28)

Reconciliação dos planos do api (R1–R19), do dashboard (D1–D18), do wpda (W1–W7) e do maintenance (M1–M3).

- **G1 — Autorizações (R1, R4–R7, R10, R11):** tabela `regulation_authorizations` (uma por autorização; saldo da oferta = autorizações ativas do mês). Só a oferta do mês corrente no fuso da cidade é autorizável. Cancelar na rede própria só com o pedido de agendamento ainda `open` (422 `appointment_scheduled`); encerramentos do 17 fora da regulação viram `cancelled` com a cota liberada; horário expirado volta a `authorized` sem contar falta; `missed` nunca é situação atual (só andamento), `outcome` = `completed`\|`missed`\|null; externo agendado = `scheduled`, remarcar mantém o mês.
- **G2 — Ordem da fila (R8, D6):** vencidos primeiro, depois cor, depois data do pedido — no api, numa leitura só (a tela não faz duas leituras).
- **G3 — Alvos (R2, R9, R13, W3):** `GET /regulation/targets?type=&q=` (busca de especialidade CBO ou SIGTAP regulável); código do grupo 02 ativo no catálogo de exames do 21a é recusado (422 `invalid_target`, `hint: "exam_order"`); o pedido de agendamento `regulated` usa `consulta_medica` ou `consulta_odontologica`, prioridade alta para vermelho; `label` do cidadão = "Consulta em <especialidade>" ou o nome do procedimento.
- **G4 — Rascunho e finalização (R14, D3, D4):** até 10 rascunhos por consulta; rascunho que deixou de valer **não bloqueia** a finalização — fica em `draft`, a consulta finaliza, e o profissional vê "pedido não enviado" e pode corrigir por adendo; recusas de rascunho trazem `index`. `cid10` é opcional para quem não pode registrar CID-10 (enfermeiro), com `ciap2` opcional no lugar.
- **G5 — Recusas (R3, D10, D12, D18):** `invalid_offer`, `citizen_not_verified`, `appointment_scheduled`, `regulation_request_required`, `invalid_transition`, `invalid_month`, `duplicate_offer`, `cnes_taken`, `unit_taken`, `duplicate_item`, `invalid_query`, `invalid_limit`, `invalid_cursor`, `invalid_deadlines` (cor menos grave com prazo menor que a mais grave); nome do prestador com 3 a 120 caracteres; 422 `skip_reason_required` sem nenhum dado dos outros pedidos (nem contagem).
- **G6 — Leitura e eventos (R12, R15, R16, D8):** fila e detalhe só para o regulador; guia para quem pediu, a unidade solicitante e o regulador; devolvidos da unidade vêm com `events`. Eventos novos `regulation_request.complemented`, `regulation_request.external`, `care_regulation.settings_changed` (só ids). A sincronização com o 17 independe do interruptor.
- **G7 — Prestador próprio e textos (R17, R19):** prestador `own` exige unidade ativa, herda o CNES e não muda de tipo; motivos 10–500 caracteres, local 1–200, observação até 500; `due_on` só é recalculado quando a cor muda e conta da data original.
- **G8 — Saídas (R18, D17):** colunas do CSV fixadas no plano do api; `Content-Disposition` sem dado pessoal (`guia-regulacao-<id>.pdf`, `regulacao-estado-<data>.csv`).
- **G9 — wpda e maintenance (W5, M1–M3):** a sessão do cidadão lista `care_regulation` nas features; o interruptor chega com `missing: ["clinical_record_disabled"]`; o catálogo usa o id `"26a"`; o maintenance faz merge depois de api, dashboard e wpda.
- **G10 — CIAP-2 no pedido (detalha o G4):** o corpo do pedido (POST/PATCH do rascunho e `complement`) aceita `ciap2` (opcional, código sem ponto); `<regulation_request>` ganha `"ciap2"|null`. Médico e dentista: `cid10` obrigatório. Enfermeiro: ao menos um de `cid10` (se o vínculo puder registrar CID-10) ou `ciap2`; 422 `diagnosis_required`. Validação pela mesma terminologia da cidade usada nos problemas da consulta (CIAP-2 e CID-10 ativos; 422 `invalid_ciap2`/`invalid_cid10`). A guia em PDF e o CSV mostram o código presente.
- **G11 — Alvos reguláveis (Task 0 do api, SIGTAP 202610):** procedimento dos grupos 02–04 é regulável quando tem instrumento BPA (01/02), mesmo que também tenha APAC principal; sai só o que vai apenas por APAC, OCI ou AIH. A fila de cirurgia eletiva (AIH) fica para o 26b. Especialidades: patologista, citopatologista, patologista clínico, radiologia e medicina nuclear não são reguláveis como especialidade (o ato deles é procedimento); hemoterapia é.
- **G12 — Cruzamento do §13 com os planos (2026-10-28; corrige o G10):** (1) o dentista segue a regra do enfermeiro: `cid10` só se o CBO puder registrar CID-10 (no 19a só médico pode), senão `ciap2` obrigatório; só o médico tem `cid10` obrigatório; (2) `cid10` enviado por quem não pode → 422 `cid10_not_allowed_for_cbo` (`path: "cid10"`); no complemento, o CID-10 que já estava no pedido não é recusado; (3) `POST …/finalize` e `POST …/addenda` devolvem `"regulation_not_sent": [ { "index", "id", "error", "path"? } ]` (ordem `created_at, id`; sem dado pessoal; só com o interruptor utilizável); (4) "encaminhado" sem unidade conta os rascunhos presentes, enviados ou não; o atendimento fecha como encaminhado; (5) a mesma regra no adendo: rascunho inválido nunca trava o adendo; (6) cada alvo ganha `label` ("Cardiologia", "Endodontia"), usado no `label` do cidadão e no `target.name` da fila e da guia; (7) número externo só onde o api aceita (`pending`, `returned`, `external`).
- **G13 — Encaminhado por regulação (api Task 5):** a migração 20261028900001 acrescenta `attendances.referral_regulated boolean NOT NULL DEFAULT false` e reescreve `ck_attendances_referral`: `referred` exige unidade, nota **ou** `referral_regulated`. O `Close` marca a flag só com a regulação utilizável e pedido presente na finalização; sem regulação, vale a regra anterior. Linhas existentes continuam válidas.
- **G14 — Cor no `<regulation_request>` (corrige o §2):** `color` é a cor **confirmada** pelo regulador, `null` até a confirmação; a cor efetiva (ordem da fila, prazo, exibição) é `color ?? suggested_color`. A tela mostra a efetiva, com a marca "sugerida" enquanto `color` for nulo, e só pede `color_reason` quando a cor enviada difere da efetiva.
- **G15 — Achados da prova (2026-10-29):** (1) o pedido de agendamento `regulated` mantém o `due_on` da regulação para a fila da unidade, mas a rota do cidadão (`citizen_api/appointments`) **não devolve** `due_on` para `kind: regulated`, e o wpda não mostra "Prazo previsto" nesse tipo (o prazo revelaria a cor); (2) `POST /regulation/requests/:id/outcome` recusa `completed` e `missed` antes do dia do agendamento no fuso da cidade (422 `outcome_before_schedule`).
- **G16 — Situação "autorizado" para o cidadão (ajusta o §9):** o pedido `authorized` (ainda sem horário) aparece ao cidadão como `"status": "authorized"`, com o texto "Autorizado — aguardando o agendamento", em vez de `waiting`, para não contradizer "Seus agendamentos" do 17. Sem cor, posição, CID ou justificativa.
