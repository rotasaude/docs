# Módulo 25 — Contratos entre apps

**Spec:** `superpowers/specs/2026-10-27-module-25-disease-surveillance-design.md` · **ADR:** `adr/0043.md`

Fonte única dos formatos entre `api`, `dashboard` e `maintenance` (o wpda não muda).
Convenções herdadas (contratos do 21a §15, 21b §14, 21c §11, 22 §12–§13, 23 §11–§12 e
24a §14): erros `{ "error" }`; step-up; interruptor 403 `feature_disabled`; JSON em toda
escrita por cookie; listas com envelope e `next_before`; registro de outro escopo por
id → 404, lista com escopo alheio → 403; leitura nominal com trilha e sem polling;
contadores sem trilha; texto longo recusado; dado sensível só no corpo, nunca na URL
nem no log; todo Data/Struct com dado pessoal sem dado em `inspect`/`to_s`/
`pretty_print`/`as_json`; paciente sem CPF como no 24a (`has_cpf`, `cns_masked`).

## 1. Nomes no código

| Spec | Código |
|---|---|
| interruptor | `surveillance` (sem `requires`; a sugestão na consulta exige também `clinical_record`) |
| papel | `surveillance` (privilegiado, step-up para conceder) |
| prazo da doença | `immediate` (24 h) \| `weekly` (7 dias) |
| origem | `consultation` \| `manual` \| `safety_net` |
| situação | `possible` (rede de segurança) \| `notified` \| `investigating` \| `closed` \| `entered` (digitada no sistema oficial) \| `dismissed` (possível descartada) |
| classificação final | `confirmed` \| `discarded` \| `inconclusive` |
| semana epidemiológica | `YYYY-WW` pela regra do Sinan (domingo a sábado; semana 1 = a que contém o primeiro sábado do ano — confirmar na Task 0) |

## 2. Formas

`<disease>`: `{ "code", "name", "deadline": "immediate"|"weekly", "cid10": [..], "ciap2": [..], "repeat_after_days", "sinan_form"|null, "local": bool, "list_version" }`.

`<notification>` (nominal; trilha): `{ "id", "patient": { "id", "display_name", "has_cpf", "cns_masked"|null }, "disease": { "code", "name", "deadline" }, "origin", "status", "classification"|null, "suspected_on", "symptoms_on"|null, "unit": { "id", "cnes", "name" }, "notifier": { "name" }|null, "due_at", "overdue": bool, "official_number"|null, "resident_elsewhere": bool, "residence_ibge"|null, "neighborhood": { "id", "name" }|null, "form": {…}, "version", "events": [ { "status", "at", "by": { "name" }|null, "reason"|null } ] }`.

`form` traz os campos da ficha de notificação individual do Sinan que não vêm do paciente; as chaves são fixadas pela Task 0 do api e publicadas como §14 deste contrato antes das tasks de formulário do dashboard.

## 3. Lista de doenças

- Plataforma (maintenance, GraphQL): `notifiableDiseases` (versão vigente e histórico) e `publishNotifiableDiseaseList(input)` → `{ ok, errors }`; publicar cria versão nova; evento de plataforma `notifiable_disease_list.published` (R18).
- Cidade: `GET /surveillance/diseases` → `{ "diseases": [<disease>] }` (plataforma vigente + locais; `surveillance` e `health_professional`).
- Doenças locais: `GET/POST/PATCH /surveillance/admin/local_diseases[/:id]` (`surveillance`, escrita com step-up); 422 `invalid_disease`, 409 `duplicate_code`.

## 4. Na consulta (`health_professional` na consulta, `clinical_record` e `surveillance` ligados)

- `GET /consultations/:id/notification_suggestions` → `{ "suggestions": [ { "disease": <disease>, "matched_code" } ] }` (problemas `add` da consulta cujo código está na lista, sem notificação aberta ou recente do paciente para a doença).
- `POST /surveillance/notifications` — `{ "consultation_id", "disease_code", "symptoms_on"? }` → 201 `<notification>`; 409 `already_notified` (com `notification_id` se visível); 403 `cbo_not_allowed` (só médico `225*` e enfermeiro `2235*`). Nunca bloqueia a finalização.

## 5. Fila da vigilância (`surveillance`)

- `GET /surveillance/notifications?status=&disease=&overdue=&unit_id=&before=&limit=` → `{ "notifications": [<notification> sem `form` e `events`], "next_before" }` (trilha uma por consulta; imediatas e vencidas primeiro).
- `GET /surveillance/notifications/:id` → `<notification>` (trilha).
- `POST /surveillance/notifications` (manual) — `{ "patient_id" | "citizen_id" | "new_patient": {…forma do 24a}, "disease_code", "suspected_on", "symptoms_on"?, "unit_id", "resident_elsewhere"?, "residence_ibge"? }`.
- `PATCH /surveillance/notifications/:id` — `{ "form", "version" }`; 409 `stale_version`; 422 `invalid_form` (com `path`).
- `POST /surveillance/notifications/:id/transition` — `{ "to", "classification"?, "official_number"?, "reason"?, "version" }`; 422 `invalid_transition`, `classification_required`, `official_number_required`.
- `POST /surveillance/notifications/:id/dismiss` — `{ "reason", "version" }` (só `possible`).
- `GET /surveillance/notifications/count` → `{ "possible", "notified", "investigating", "overdue", "immediate_open", "without_official_number" }` (sem trilha).

## 6. Notificação negativa

- `health_units.notifying` (bool; padrão verdadeiro para unidades de APS ativas), editável pelo `municipal_admin`.
- `GET /surveillance/weekly_negatives?week=YYYY-WW` → `{ "week", "units": [ { "unit": {…}, "status": "with_cases"|"negative"|"pending" } ] }`.
- `POST /surveillance/weekly_negatives` — `{ "unit_id", "week" }`; 409 `unit_has_notifications`, 409 `already_recorded`, 422 `future_week`.

## 7. Saídas (`surveillance`; trilha; sem cache)

- `GET /surveillance/notifications/:id/form.pdf` — ficha de notificação individual no formato do Sinan.
- `GET /surveillance/exports/week.csv?week=YYYY-WW` — uma linha por notificação da semana (sem CPF; CNS mascarado).
- Relatório sem número oficial = lista com `status=closed` e sem `official_number`.

## 8. Painel (`surveillance` e `analyst`; agregado sem nome)

`GET /surveillance/panel?from_week=&to_week=&disease=` → `{ "weeks": ["YYYY-WW"…], "rows": [ { "neighborhood": { "id"|null, "name" }, "counts": [n|null], "signal": [bool] } ], "suppressed": true }`. Contagens 1–4 viram `null` (regra do analytics); sinal = semana acima do dobro da média das 4 anteriores com pelo menos 5 casos; sempre rotulado como sinal para avaliar.

## 9. Alertas e jobs

- Notificação imediata criada → e-mail para todos os membros ativos `surveillance`: "nova notificação imediata: <doença>, <unidade>" (sem dado do paciente).
- Job diário: vencidas → e-mail resumo (só contagens).
- Job semanal (segunda): unidades notificadoras sem negativa nem caso na semana anterior → e-mail resumo.
- Rede de segurança: consumidor de `consultation.finalized` (e `consultation.addendum_added`) cria `possible` idempotente.

## 10. Bairro e residência

Bairro: domicílio do cadastro territorial (24a) → `citizens.neighborhood_id` → null ("não informado"). `resident_elsewhere` com `residence_ibge` (7 dígitos) para quem mora em outro município.

## 11. Maintenance

- Interruptor `surveillance`.
- Lista de doenças da plataforma: listar, editar e publicar versão (mantenedor com step-up).

## 12. Eventos (só ids)

`disease_notification.created`, `.updated`, `.transitioned`, `.dismissed`, `weekly_negative.recorded`, `local_disease.changed`; plataforma `notifiable_disease_list.published`.

## 13. Ordem de entrega

0. Prova técnica (plano do api, Task 0): Portaria 11.211/2026 (lista e prazos), CID-10 de cada doença nas fichas do Sinan e CIAP-2 equivalentes, ficha de notificação individual do Sinan (campos), regra da semana epidemiológica; carga inicial com fonte por linha; tabela da ficha publicada como §14.
1. `api` (porta 3047; migrações de cidade e plataforma a partir de 20261027900001; worktree `.claude/mod25`, branch `feat/mod-25-disease-surveillance`; testes com `bin/test api -w mod25`).
2. `dashboard` (Vite 5202), worktree em `rota-saude/.worktrees/dashboard-mod25`.
3. `maintenance` (prova na 5177 com o serviço do compose parado), worktree em `rota-saude/.worktrees/maintenance-mod25`.
4. Catálogo público.

## 15. Acréscimos (revisão dos planos, 2026-10-27)

Reconciliação dos planos do api (V1–V17), do dashboard (D1–D16) e do maintenance (M1–M6). O §14 continua reservado para a ficha (Task 0 do api).

- **K1 — Semana epidemiológica (V1, D16; corrige o §1):** a semana 1 é a que contém 4 de janeiro (domingo a sábado); há anos com semana 53 (`2022-01-01` = `2021-52`; `2026-01-03` = `2025-53`). A Task 0 confere no calendário do Sinan.
- **K2 — Códigos (V2, V3, M3):** códigos sem ponto; CID-10 de 3 caracteres cobre a categoria; nenhum código em duas doenças; CIAP-2 só quando corresponde a uma única doença; doença local com prefixo `local_`; colisão de código ou de CID/CIAP com a plataforma → 409 `duplicate_code` com `path`. `<disease>` ganha `notes`; a local ganha `id` e `active` (D4).
- **K3 — Janela de repetição:** valores iniciais da carga (tuberculose 270, hanseníase 365, sífilis 90, HIV e hepatites 36500 = nunca repetir; demais 30) são editáveis na lista e devem ser validados com a vigilância da cidade piloto (runbook).
- **K4 — Rotas (V4, V5, D1, D2):** sugestão em `GET /attendance/consultations/:id/notification_suggestions`; vigilância tem rotas próprias `POST /surveillance/person_lookup` e `POST /surveillance/patients/duplicates` (as do 24a exigem `health_professional` e `territory_registry`); `GET /surveillance/units` → unidades ativas com `notifying`. `new_patient` sem `duplicates_reviewed` com suspeita de duplicado → 409 `duplicate_suspected` (D3, como o L28 do 24a).
- **K5 — `<notification>` (V5, V6, V8):** ganha `deadline`, `consultation_id`, `closed_at`, `entered_at`, `dismissed_at`; notificador nulo na `possible` até virar `notified`; vencida = prazo passado com situação `possible` ou `notified`, contado do instante da suspeita; o 409 `already_notified` só traz `notification_id` para quem tem `surveillance`. Depois de finalizada a consulta, a tela da consulta não oferece mais a notificação (a rede de segurança cobre) (D13).
- **K6 — Andamento (V9, D5, D6, D9, D10):** transições lineares `possible→notified|dismissed`, `notified→investigating`, `investigating→closed`, `closed→entered`; número oficial `[0-9A-Za-z./-]{1,30}`; ficha travada em `entered`/`dismissed`; `PATCH` manda a ficha inteira, recusa com `path: "form.<chave>"`; recusas novas `invalid_reason`, `invalid_filter`, `invalid_unit`, `invalid_residence`, `invalid_symptoms_on`, `invalid_week`, `invalid_range`, `invalid_disease`. `immediate_open` = imediatas em `possible`/`notified`/`investigating`.
- **K7 — Contagens e semanas (V7, V10, V12, V14, D7, D8, D11):** o painel conta pela semana dos primeiros sintomas (ou da suspeita, sem data de sintomas) e exclui `possible`, `dismissed` e classificação `discarded`; CSV e negativa usam a semana da suspeita; `future_week` inclui a semana corrente; GET da negativa sem semana usa a anterior; relatório sem número = fila com `without_official_number=true`; painel com 12 semanas de padrão e 53 no máximo.
- **K8 — Eventos e trilha (V11, M5):** payloads só com ids e booleanos (nunca `disease_code`); trilha `clinical_record.viewed` com `access: "surveillance"`; auditoria de plataforma `maintenance.notifiable_disease_list.published` (R18).
- **K9 — Maintenance (V13, M1, M2, M4, M6):** `publishNotifiableDiseaseList(input: { diseases, effectiveFrom, expectedVersion, code })` → `{ ok, errors, version }` (TOTP em `code`; `expectedVersion` diferente da vigente → erro `stale_version`); tipo `NotifiableDiseaseVersion` com `effectiveFrom`, `publishedBy`, `notes`, `sources`; vigência não começa antes de hoje nem antes da vigente (futura = "agendada"); versão nova pode retirar doença.
- **K10 — Saídas (V15, V16, D12):** colunas do CSV fixadas no plano do api, CNS só nos 4 últimos dígitos; pré-preenchimento da ficha traduz raça/cor do paciente para o Sinan (03 parda → 4, 04 amarela → 3); `Content-Disposition` sem dado pessoal (`notificacao-<id>.pdf`, `notificacoes-<YYYY-WW>.csv`).
- **K11 — Unidade notificadora (V17, D14, D15):** `notifying` no POST/PATCH e no JSON da unidade; padrão `true` para unidades do tipo UBS na criação.
- **K12 — Pré-voo da execução (2026-10-27):** (a) a mutação usa argumentos planos, convenção do `BaseMutation`: `publishNotifiableDiseaseList(diseases, effectiveFrom, expectedVersion, ordinance, notes?, code)` → `{ ok, errors, version }`; `notes` e `sources` ficam na versão; (b) a API aceita notificar consulta já finalizada (evita corrida com a rede de segurança); só a tela deixa de oferecer; (c) recusas acrescentadas: `invalid_limit`, `invalid_cursor` (fila com cursor opaco) e `invalid_notifying`; (d) se uma versão nova da plataforma passar a cobrir um CID/CIAP de doença local, a plataforma vence na correspondência e a doença local é recusada só na próxima edição dela.
