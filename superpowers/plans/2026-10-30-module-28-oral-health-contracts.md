# Módulo 28 — Contratos entre apps

**Spec:** `superpowers/specs/2026-10-30-module-28-oral-health-design.md` · **ADR:** `adr/0046.md` · **Pesquisa:** `pesquisa/2026-10-30-odontologia-28.md`

Fonte única dos formatos entre `api`, `dashboard` e `maintenance`. O wpda não muda:
na linha do tempo, a consulta odontológica entra como consulta.

Convenções herdadas dos contratos do 21a ao 27a (o 27a §13 é o mais recente):
- erros `{ "error" }`, com `problems[]` trazendo todas as recusas de uma vez;
- step-up (TOTP);
- interruptor desligado: 403 `feature_disabled`;
- JSON em toda escrita por cookie;
- listas com envelope e `next_before`;
- registro de outro escopo pedido por id → 404; lista com escopo alheio → 403;
- leitura nominal com trilha `clinical_record.viewed` e sem polling; contadores sem trilha;
- dado sensível só no corpo, nunca na URL nem no log;
- nenhum dado pessoal em `inspect`/`to_s`/`pretty_print`/`as_json`;
- paciente sem CPF como no 24a (`has_cpf`, `cns_masked`);
- imutabilidade por trigger em `db/city_triggers.sql`;
- todo `Platform.audit` novo entra no R18.

## 1. Nomes no código

| Spec | Código |
|---|---|
| interruptor | `dental_care` (`requires: clinical_record`) |
| tipo da consulta | `consultations.kind`: `general` (padrão; o que existe hoje) \| `dental` |
| registro inicial | `consultations.baseline` (bool; só `dental`) |
| quem registra `dental` | dentista: família `2232*` (a lista do LEDI); TSB `322405`, `322425`; ASB `322415`, `322430`. A consulta `general` continua excluindo `2232` (`Consultations::Cbos`) |
| papel na consulta | `dentist` \| `auxiliary` (TSB/ASB), derivado do CBO do vínculo |
| unidade | `health_units.kind` ganha `ceo` |
| perfil do CEO | `ceo_type`: `I` \| `II` \| `III`; `specialties`: `endodontics`, `periodontics`, `oral_surgery`, `stomatology`, `special_needs`, `prosthesis` |
| equipe de saúde bucal | `health_teams.kind` ganha os tipos de eSB do CNES; o plano confirma o código na tabela de tipos de equipe do CNES. Tem `associated_team_id` (a eSF/eAP) |
| dente | FDI: permanente `11–18, 21–28, 31–38, 41–48`; decíduo `51–55, 61–65, 71–75, 81–85` |
| face | `M` \| `D` \| `O` (oclusal/incisal) \| `V` (vestibular/bucal) \| `L` (lingual/palatina) |
| local do procedimento | `tooth` \| `face` \| `sextant` (`S1`–`S6`) \| `arch` (`upper`/`lower`) \| `mouth` |
| situação do dente/face | lista do odontograma do PEC; o plano do api fixa os códigos em YAML versionado (ex.: `sound`, `caries`, `restored`, `missing`, `extraction_indicated`, `prosthesis`, …) |
| plano | `open` \| `completed` \| `cancelled`; item `planned` \| `done` \| `cancelled` |
| prótese | etapa `impression` \| `sent` \| `received` \| `installed` \| `cancelled` |
| campos da FAO | códigos do LEDI 8.7.0 sem tradução: `care_type` (2/4/5/6), `dental_consultation_type` (1/2/4), `oral_health_surveillance` (1–6, 99), `supplies` (1–3), `dental_outcomes` (1–12, 14–18) |

## 2. Formas

`<location>`: `{ "kind", "tooth"|null, "face"|null, "sextant"|null, "arch"|null }`.

`<odontogram>` (nominal; trilha): `{ "patient": { "id", "display_name", "has_cpf", "cns_masked"|null }, "dentition": "permanent"|"deciduous"|"mixed", "teeth": [ { "tooth", "situation", "faces": { "M": "<situação>"|null, … }, "last_event_at", "notes_count" } ], "open_plans": [ { "id", "health_unit": { "id", "name", "kind" } } ] }`.

`<chart_event>`: `{ "id", "tooth", "face"|null, "situation"|null, "procedure": { "sigtap", "name" }|null, "consultation_id", "addendum_id"|null, "by": { "name", "cbo" }, "at" }`.

`<plan>`: `{ "id", "health_unit": {…}, "status", "opened_in_consultation_id", "closed_in_consultation_id"|null, "items": [ { "id", "sigtap", "name", "location": <location>, "status", "done_in_consultation_id"|null, "cancel_reason"|null } ] }`.

`<dental_fields>` (rascunho e consulta finalizada): `{ "care_type", "dental_consultation_type"|null, "oral_health_surveillance": [..], "supplies": [..], "dental_outcomes": [..], "pregnant": bool|null, "special_needs": bool, "procedures": [ { "id", "sigtap", "location": <location>, "quantity", "plan_item_id"|null } ], "findings": [ { "tooth", "face"|null, "situation" } ], "plan_changes": { "add": [ { "sigtap", "location" } ], "cancel": [ { "item_id", "reason" } ] }, "periodontal_screening"|null, "periodontal_chart"|null, "endodontics": [..] }`.

## 3. Administração (`municipal_admin`; escrita com step-up)

- `GET/PUT /dental/admin/ceo_profiles/:health_unit_id` — `{ "ceo_type", "specialties": [..] }`; só unidade `kind: ceo`.
- `GET/PUT /dental/admin/auxiliary_procedures` — `[ { "sigtap" } ]`: o que TSB/ASB registram e podem baixar no plano. Semente com os preventivos do LEDI.
- O cadastro da unidade (`units`) aceita `kind: ceo`.

## 4. Consulta odontológica

- `GET /consultation_options` ganha `kinds: ["general"]`, `["dental"]` ou os dois, conforme o CBO do vínculo.
- `POST /attendances/:id/consultation` — `{ "kind": "dental", "baseline"?: bool }`. Recusas:
  - 403 `cbo_not_allowed`;
  - 403 `feature_disabled` (`dental_care`);
  - 422 `baseline_requires_dentist`.
- `PATCH /consultations/:id` (rascunho) aceita o bloco `dental` (`<dental_fields>`) além do que já aceita.
- `POST /consultations/:id/finalize` — recusas em `problems[]`:
  - `oral_health_surveillance_required`;
  - `dental_consultation_type_required` (com `care_type` 2);
  - `dental_consultation_type_forbidden` (com 4);
  - `return_forbidden_on_urgency` (6 + retorno);
  - `completed_requires_first_or_return` (15);
  - `discharge_forbidden_with_first_or_return` (17);
  - `pregnant_required`;
  - `pregnant_forbidden_male`;
  - `problem_required` (pelo menos um CID-10 ou CIAP-2);
  - `auxiliary_cannot_refer`;
  - `auxiliary_procedure_not_allowed`;
  - `auxiliary_cannot_plan`;
  - `quantity_exceeds_location_limit`;
  - `baseline_only_findings` (o registro inicial não aceita procedimento, desfecho, tipo de consulta nem documento);
  - `plan_item_not_open`.
- Ao finalizar, numa transação, os achados e procedimentos viram `dental_chart_events`; os procedimentos mudam a situação pela tabela SIGTAP → situação; o plano recebe as mudanças, os itens ligados passam a `done`, e o desfecho 15 fecha o plano da unidade.
- `pregnant` vem pré-preenchido com `true` quando o paciente tem inscrição ativa na linha de pré-natal (23); o profissional confirma.
- **Adendo:** aceita `findings` e `plan_changes.cancel` (com motivo). Não aceita procedimento novo.

## 5. Odontograma e plano (leitura nominal; trilha)

- `GET /patients/:id/dental_chart` → `<odontogram>`.
- `GET /patients/:id/dental_chart/events?tooth=&before=` → `{ "events": [<chart_event>], "next_before" }`.
- `GET /patients/:id/dental_plans?health_unit_id=&status=` → `{ "plans": [<plan>] }`.
- A sugestão de "tratamento concluído" vem em `GET /consultations/:id` como `dental.suggest_completed: bool` (plano da unidade sem item `planned`, depois das mudanças do rascunho).

## 6. Especialidades (dentro do bloco `dental`)

- `periodontal_screening`: `{ "sextants": [ { "sextant", "code": 0–4, "star": bool } ] }` (registro periodontal simplificado).
- `periodontal_chart`: `{ "teeth": [ { "tooth", "mobility": 0–3|null, "furcation": 0–3|null, "sites": [ { "site": "MV"|"V"|"DV"|"ML"|"L"|"DL", "probing_depth_mm", "recession_mm", "bleeding": bool } ] } ] }`.
- `endodontics`: `[ { "tooth", "canals": 1–5, "status": "in_progress"|"completed" } ]`; o histórico por dente sai em `GET /patients/:id/endodontic_treatments`.
- Finalizados, todos imutáveis. Leitura com trilha.

## 7. Prótese

- `POST /dental/prosthesis_orders` (dentista) — `{ "patient_id", "consultation_id", "sigtap", "laboratory": { "care_provider_id" } | { "name" } }` → `impression`.
- `POST /dental/prosthesis_orders/:id/stages` — `{ "stage": "sent"|"received"|"cancelled", "reason"? }`.
- **Instalação:** só pela consulta. O procedimento com o SIGTAP da prótese e `prosthesis_order_id` passa o pedido a `installed`, e a produção entra na FAO da unidade que entrega.
- `GET /dental/prosthesis_orders?health_unit_id=&stage=&before=` (nominal; trilha).

## 8. Documentos, regulação e linha do tempo

- **Documentos (19c/19d):** as rotas de `consultations/:id/documents` aceitam a consulta `dental` do dentista. O auxiliar recebe 403 `auxiliary_cannot_issue`. Vale a política de assinatura do dentista que já existe (19b/19c). A receita controlada segue o 19d. Fecha o api#75.
- **Regulação (26a):** `consultations/:id/regulation_requests` aceita a consulta `dental` do dentista. O desfecho 1–11 da FAO exige um pedido de especialidade odontológica correspondente quando `care_regulation` está ligado; o mapeamento código → especialidade fica no plano.
- **Linha do tempo (20):** item `consultation` com `kind: "dental"` e os procedimentos pelo nome (sem dente).

## 9. Ficha FAO (LEDI)

- Tipo de ficha novo no exportador do módulo 16: `FichaAtendimentoOdontologicoMaster`, uma por consulta `dental` finalizada e não `baseline`, no CNES da unidade do atendimento (UBS ou CEO).
- Procedimentos agregados por SIGTAP (soma das quantidades), com os limites por local (dente ≤ 32, sextante ≤ 6, arcada ≤ 2, demais ≤ 1).
- Adendo que só traz achado não gera ficha nova.
- Prova: ficha aceita no PEC local.

## 10. Painéis (sem trilha; só contagens)

- `GET /dental/ceo_goals?health_unit_id=&month=YYYY-MM` → `{ "ceo_type", "specialties": [ { "key", "goal", "done", "composition": { "rule", "ok": bool, "value" }|null, "met": bool } ], "misses_last_12": { "<key>": { "consecutive", "total" } }, "suspension_risk": bool }`. As listas SIGTAP por grupo ficam em YAML versionado (Portaria 1.464/2011).
- `GET /dental/indicators?team_id=&period=` → `{ "b1": { "numerator", "denominator" }, "b2": { "numerator", "denominator" } }`, com as regras das notas metodológicas.
- Papéis: `municipal_admin` e o gestor que já lê os painéis do 14/23.

## 11. Linha de cuidado (23)

O source `oral_health` sai de `UNTRACKED_SOURCE` e passa a contar consultas `dental` finalizadas com `pregnant = true`, dentro da janela. O `cbo_filter` do `c3_k` passa a `2232*` e `3224*`.

## 12. CNES

O `Cnes::Apply` importa as eSB (INE, tipo, unidade, eSF/eAP associada) e os vínculos de dentista, TSB e ASB a elas, pela mesma rotina de proposta e confirmação da eSF/eAP.

## 13. Maintenance

Interruptor `dental_care` (rótulo e descrição; mecanismo genérico).

## 14. Eventos (só ids)

- `dental_consultation.finalized`
- `dental_plan.completed`
- `dental_plan.cancelled`
- `prosthesis_order.created`
- `prosthesis_order.stage_changed`
- `ceo_profile.changed`
- `dental.auxiliary_procedures_changed`

## 15. Ordem de entrega

1. **`api`:** porta 3050; migrações de cidade a partir de 20261030900001; worktree `.claude/mod28`, branch `feat/mod-28-oral-health`; testes com `bin/test api -w mod28`.
2. **`dashboard`:** Vite 5206; worktree em `rota-saude/.worktrees/dashboard-mod28`.
3. **`maintenance`:** prova na 5177, com o serviço do compose parado.
4. **Catálogo público.**
