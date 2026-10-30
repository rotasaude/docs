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

## 16. Acréscimos (revisão dos planos, 2026-10-30)

Reconciliação dos planos do api (D1–D12 do api), do dashboard (D1–D24 do dashboard) e do maintenance (M1–M3). Onde os dois planos divergem, vale o que está aqui.

- **O1 — Prefixos (dashboard D1):**
  - As rotas da consulta que já existem continuam sob `/attendance`: `consultation_options`, `attendances/:id/consultation`, `consultations/:id…`.
  - Tudo o que é novo fica sob `/dental`, inclusive as leituras do §5, que passam a `/dental/patients/:id/dental_chart`, `/dental/patients/:id/dental_chart/events`, `/dental/patients/:id/dental_plans` e `/dental/patients/:id/endodontic_treatments`.
  - O dashboard ganha um proxy `/dental`.
- **O2 — Erros (dashboard D2; api):** valem `{ "error", "path" }` e `problems: [ { "path", "error" } ]`. O caminho segue o corpo, por exemplo `dental.procedures[2].plan_item_id`. A finalização devolve todas as recusas; o rascunho e a administração devolvem a primeira.
- **O3 — Coroa e raiz (api D1):** valem as situações do PEC.
  - **Coroa:** 21 situações; **raiz:** 6; **arcada:** prótese total.
  - Entram `teeth[].root` em `<odontogram>`, `part: "crown"|"root"` em `<chart_event>` e em `findings[]`, e achado por arcada (`complete_denture`).
  - Só um subconjunto das situações vale para face.
  - Recusas: `situation_not_allowed_for_tooth`, `invalid_situation`.
- **O4 — Opções (api D8, dashboard D3/D4):** `GET /consultation_options` traz `dental_role` (`dentist`, `auxiliary` ou `null`) e o bloco `dental`, com `care_types`, `consultation_types`, `surveillance`, `supplies`, `outcomes` e `situations: { crown, root, arch, face }`. Cada item é `{ code, label }`, com código em string. O dashboard usa essa forma e nunca traduz código por conta própria.
- **O5 — Local e procedimento (api D3, dashboard D5):**
  - O local precisa casar com a lista do LEDI (dente, sextante ou arcada); procedimento fora da lista só aceita `mouth` e quantidade 1. Recusas: `invalid_location`, `invalid_procedure`.
  - Rota nova `POST /dental/procedures/search { "q" }` → `{ "procedures": [ { "sigtap", "name", "locations", "max_quantity", "auxiliary_allowed", "prosthesis" } ] }`, sem dado pessoal e sem trilha. A busca vai no corpo; o api acrescenta a rota na Task 5.
- **O6 — Plano (api D4, dashboard D11):**
  - `plan_changes.cancel_plan { reason }` cancela o plano dentro da consulta, sem rota própria.
  - Na UBS, item novo sem plano aberto exige primeira consulta programática: `plan_requires_first_consultation`.
- **O7 — Recusas da finalização (api D4, dashboard D8):** além das do §4:
  - `care_type_required`;
  - `dental_outcome_required` (a FAO exige ao menos um desfecho);
  - `baseline_requires_findings`.
- **O8 — Regras do 19a na consulta odontológica (dashboard D12):**
  - Na consulta odontológica não se exige conduta nem o tipo de atendimento do 19a.
  - Continua valendo "A ou P" e o desfecho do atendimento.
  - O registro inicial (`baseline`) não exige problema, conduta nem A/P, mas exige achados.
  - Fora do registro inicial, `problem_required` continua valendo, porque a FAO exige CID-10 ou CIAP-2.
- **O9 — Desfecho e especialidade (api D5):** os desfechos 1–9 da FAO exigem pedido de regulação da especialidade correspondente, pelo mapeamento abaixo, quando `care_regulation` está ligado. Recusa: `dental_referral_request_required`. Os desfechos 10 e 11 não exigem pedido.

  | Desfecho | CBO da especialidade |
  |---|---|
  | 1 | `223288` |
  | 2 | `223268` |
  | 3 | `223212` |
  | 4 | `223220` |
  | 5 | `223224` |
  | 6 | `223236` |
  | 7 | `223240` |
  | 8 | `223248` |
  | 9 | `223256`, `223252` ou `223264` |
- **O10 — Prótese (api D6, dashboard D6/D14/D15):**
  - `procedures[].prosthesis_order_id` entra no §2. A rota de etapas recusa `installed` e transição fora de ordem. Recusas na consulta: `prosthesis_order_not_received`, `prosthesis_order_mismatch`.
  - A forma `<prosthesis_order>` é a da Task 11 do api, com `stages[]`.
  - A lista aceita o filtro `patient_id` (nominal; trilha).
  - Rota nova `GET /dental/laboratories`, legível pelo dentista: os `care_providers` ativos do 26a; o laboratório terceiro continua aceito por nome.
- **O11 — Equipes de saúde bucal (api D2/D7, dashboard D23):**
  - O tipo da eSB é `71`, sem distinguir modalidade. Está confirmado só em fonte secundária, o que vira **gate de go-live**.
  - A associação da eSB à eSF/eAP é automática quando a unidade tem uma só eSF/eAP ativa. Nos outros casos, `GET/PUT /dental/admin/oral_health_teams` (admin, step-up), com o evento `dental.team_association_changed`.
  - Rota nova `GET /dental/teams`: as eSB visíveis, para o seletor do B1/B2.
  - As linhas de cuidado e o território continuam só com `70`/`76` (Review Focus do api).
- **O12 — Linha do tempo (api D9, dashboard D24):** a entrada continua `kind: consultation`, com `detail_ref.kind = "dental"` e o título pronto vindo do api ("Consulta odontológica" ou "Exame odontológico inicial"). Dashboard e wpda não mudam.
- **O13 — Histórico do dente sem dado na URL (dashboard D9):** sai o filtro `?tooth=`. `/dental/patients/:id/dental_chart/events` pagina com `before` e `limit`, e o dashboard filtra o dente na tela.
- **O14 — Formas que faltavam (dashboard D7, D10, D13, D16–D21):**
  - `name` vem na leitura dos procedimentos e dos itens do plano; o api ignora o campo na escrita.
  - `limit` em todas as listas.
  - Adendo: `changes.dental = { findings, plan_changes: { cancel } }`.
  - Endodontia: histórico com `sessions[]`.
  - Sextante excluído no registro periodontal: `code: null`.
  - Faixas do periograma: sondagem de 0 a 20 mm, recessão de −10 a 20 mm, mobilidade e furca de 0 a 3.
  - Tamanhos: motivo com 10 a 500 caracteres; nome do laboratório terceiro com 3 a 120.
  - `GET /dental/admin/ceo_profiles` em lista, legível pelo admin e pelo analista.
  - Lista do TSB/ASB no envelope `{ "procedures": [ { "sigtap", "name" } ] }`.
- **O15 — Painéis (api D11/D12, dashboard D22/D23):**
  - `ceo_goals` ganha `history: [ { "month", "specialties": [ { "key", "done", "met" } ] } ]` com os últimos 12 meses.
  - As chaves dos grupos e das regras de composição são as do YAML do api.
  - `period=YYYY-MM` nos indicadores = os 12 meses que terminam nesse mês; `denominator: null` quando a eSB não tem equipe associada.
  - Os códigos sucessores de restauração da SIGTAP de 06/2026 entraram nas metas por inferência e só valem depois da confirmação da coordenação de saúde bucal (**gate de go-live**).
- **O16 — Faturamento e interruptor (decisões do plano do dashboard):**
  - O painel de BPA do 22 não aparece na consulta odontológica, nem no CEO.
  - Com `dental_care` desligado, a consulta odontológica já finalizada continua legível, como o registro retido da farmácia no 27a.
- **O17 — Maintenance (M1, M2):** o rótulo é "Saúde bucal", no maintenance e no grupo do dashboard. O `missing` do `dental_care` é `clinical_record_disabled`. O interruptor existe em todos os ambientes, sem simulado.
- **O18 — Catálogo público (M3):** fica com a sessão de execução no fechamento, como nos módulos anteriores (`id: "28"`, F-28.1 a F-28.8, sem prometer o que a spec deixa fora).
- **O19 — Prova no PEC (Task 18 do api):** três pontos são confirmados na prova e, se divergirem, corrigidos pelo caminho já escrito no plano:
  - o CID-10 registrado pelo dentista na FAO;
  - o campo `gestante` omitido para paciente do sexo masculino;
  - `localAtendimento = 1` também no CEO.

**Gates de go-live conhecidos:**
- código `71` da eSB (O11);
- códigos sucessores de restauração (O15);
- BPA do CEO para o pagamento MAC;
- Quadro 3 da NT 32 (lista oficial de próteses);
- notas metodológicas B3–B6.

### Acréscimos da execução (pré-voo, 2026-10-30)

- **O20 — Opções sempre com vocabulário (api R5):** `GET /attendance/consultation_options` traz `dental_role` assim:
  - `dentist` quando algum vínculo do usuário é `2232*`;
  - `auxiliary` quando é TSB/ASB;
  - `null` nos outros casos.

  O bloco `dental` (vocabulário fixo, sem dado pessoal) vai sempre, mesmo com o interruptor desligado, para a consulta retida continuar legível (O16). `kinds` e `dental_role` seguem o CBO e o interruptor.
- **O21 — Local do efeito fora da lista do LEDI (api R6):** procedimento fora da lista do LEDI continua aceitando `mouth` com quantidade total 1 (O5). A exceção são os códigos da tabela SIGTAP → situação, que aceitam também o local do efeito (dente, face ou arcada), sempre com quantidade total 1. A spec vence o O5 aqui; a FAO não muda.
- **O22 — Recusas e exigências da finalização (api R7/R8):**
  - `care_type_required` (fora do `baseline`) e `baseline_requires_findings` confirmados.
  - Na `dental`, fora do `baseline`, continuam exigidos "A ou P" e o `outcome` do atendimento.
  - O `baseline` exige achados e o `outcome`.
- **O23 — Faixas e listas (api R9, R10, R11):**
  - Recessão de −10 a 20 mm; sextante "X" = `code: null`; nome do laboratório com 3 a 120 caracteres.
  - `GET /dental/admin/ceo_profiles` → `{ "profiles": [ { "health_unit": { "id", "name", "kind" }, "ceo_type"|null, "specialties" } ] }`, legível por quem lê os painéis.
  - A lista de próteses exige `health_unit_id` ou `patient_id`; sem nenhum dos dois → 422 `invalid_request`. Com `patient_id`, vale o acesso do prontuário, com trilha.
  - `GET /dental/laboratories` → `{ "laboratories": [ { "id", "name" } ] }`, legível por quem tem vínculo odontológico.
  - `GET /dental/teams` → `{ "teams": [ { "id", "name", "ine", "health_unit": { "id", "name" } } ] }` (eSB `71` ativas), legível por quem lê os painéis.
- **O24 — Painel do CEO (api R12, dashboard R6):**
  - `history` traz os 12 meses fechados antes de `month`.
  - `composition.value` é string percentual com ponto (`"20.00"`); a vírgula só aparece na tela.
  - As chaves são `restorations_min_50` e `three_roots_min_20`.
  - `denominator: null` aparece como "—".
- **O25 — Códigos como string (api R13, dashboard R5/R7):**
  - No JSON odontológico, os códigos da FAO e o `tooth` saem como string; a entrada aceita inteiro ou string de dígitos.
  - Na consulta `dental`, o api ignora o `care_type` do topo do corpo (vale `dental.care_type`), e o JSON espelha o mesmo valor nos dois lugares.
- **O26 — Sugestão de tratamento concluído (api R15):** `suggest_completed` só aparece com plano aberto da unidade que tenha pelo menos 1 item e nenhum item `planned` depois do rascunho.
- **O27 — Etapas e endodontia (api R18, dashboard R8):**
  - `POST /dental/prosthesis_orders/:id/stages` → 200 com `<prosthesis_order>`, sem `created_at` nem `stages[].reason` (opcionais no dashboard).
  - Transição fora de ordem → 409 `invalid_transition`.
  - `sessions[]` da endodontia ganha `by: { "name" }`.
  - Os textos de recusa do dashboard usam os códigos do api.
- **O28 — Abertura justificada pelo dentista (api T4):**
  - O dentista (`2232*`, papel odontológico) pode abrir o prontuário com justificativa, como a spec §5 manda ("em contexto ou abertura justificada"; ADR 0031).
  - TSB e ASB ficam só com a leitura em contexto.
  - `PecExternal::Authorize` não muda.
- **O29 — CBOs de saúde bucal no vínculo (api T4):**
  - `config/professionals/cbo_saude.yml` é lista nacional do api (ADR 0021): a prefeitura não edita, código nunca sai, título oficial da CBO e `council: CRO`.
  - Entram os CBOs que o CEO e a ESF precisam para operar: `223212` endodontista, `223220` estomatologista, `223236` odontopediatra, `223248` periodontista, `223256` protesista, `223268` traumatologista bucomaxilofacial, `223288` cirurgião-dentista para pacientes com necessidades especiais, `322425` TSB da ESF e `322430` ASB da ESF.
  - A revisão confere cada título na tabela oficial da CBO.
- **O30 — Cancelar plano e concluir tratamento na mesma consulta (api T6):** a combinação é recusada com 422 `cancel_plan_conflicts_with_completed`, `path: "dental.plan_changes.cancel_plan"`. Sem isso, a FAO diria "tratamento concluído" com o plano cancelado.
- **O31 — Itens de plano cancelado (api T6):** cancelar o plano passa todos os itens `planned` a `cancelled`, com o motivo do plano, na mesma transação.
- **O32 — Limite de procedimentos da FAO (api T6):** mais de 20 SIGTAP distintos (`procedimentosRealizados` do LEDI) é recusado na finalização com `too_many_procedure_codes`, `path: "dental.procedures"`. A consulta nunca fica finalizada sem poder gerar ficha.
- **O33 — Adendo com o interruptor desligado (api T8):**
  - Adendo com `changes.dental` e `dental_care` desligado → 403 `feature_disabled`. O adendo sem `dental` continua aceito.
  - Interruptor desligado significa nenhuma escrita odontológica nova; a leitura da consulta retida continua (O16).
  - Para corrigir o odontograma, religa-se o módulo.
- **O34 — Título da consulta odontológica para o cidadão (api T10):**
  - O título vem do api, nas duas pontas:
    - dashboard: "Consulta odontológica" ou "Exame odontológico inicial";
    - wpda: "Consulta odontológica com <profissional>" ou "Exame odontológico inicial".
  - O resumo do cidadão continua nulo: nenhum procedimento, dente ou achado chega ao wpda.
  - O código do wpda não muda; é nesse sentido que vale o "o wpda não muda" do cabeçalho.
  - A linha do tempo do cidadão já mostra o tipo do que aconteceu (agendamento, procedimento, documento), e a consulta é dele.
- **O35 — Primeiro mês de meta do CEO (api T15):** o primeiro mês avaliado é o seguinte ao cadastro do perfil do CEO, a não ser que o perfil tenha nascido no dia 1. Um mês parcial nunca conta como falta. O `history` traz só os meses avaliados, por isso pode vir com menos de 12.
- **O36 — Janela das faltas alternadas (api T15):** "3 alternados em 12 meses" (Portaria 1.464/2011) usa a janela estrita de 12: os 11 meses fechados mais o mês corrente. "2 consecutivos" não muda.
