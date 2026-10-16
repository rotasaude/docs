# Módulo 21b — Contratos entre apps

**Spec:** `superpowers/specs/2026-10-15-module-21b-municipal-lab-design.md` · **ADR:** `adr/0037.md` · **Base:** contrato do 21a (`superpowers/plans/2026-10-13-module-21a-exams-contracts.md`, inclusive §15).

Fonte única dos formatos entre `api`, `dashboard`, `maintenance`, `contracts` e o
middleware simulado. Convenções do projeto: `{ "error": "<reason>" }`; step-up
401 `mfa_required`; interruptor 403 `{ "error": "feature_disabled", "feature": "<key>" }`;
Content-Type JSON em toda escrita por cookie; upload em JSON base64 (A3 do 21a);
leitura clínica com trilha (`ClinicalRecord::Trail`); polling nunca em rota que
grava trilha (A46 do 21a).

## 1. Nomes no código

| Spec | Código |
|---|---|
| papéis | `Membership` `lab_staff` e `lab_releaser` (os dois em `PRIVILEGED_ROLES`); RT/substituto = `lab_technical_lead` / `lab_technical_deputy` na ligação do liberador com o laboratório |
| origem nova do exame | `exam_orders.origin` + `lab_reception` |
| fonte nova do resultado | `exam_results.source` + `lab` |
| local de coleta | `ubs` \| `lab` |
| situação da requisição | `open` \| `in_analysis` \| `typed` \| `released` \| `delivered` |
| situação da amostra | `awaiting` \| `in_transit` \| `received` \| `rejected` \| `accepted_with_restriction` \| `not_received` |
| situação da remessa | `open` \| `sent` \| `received` |
| situação do laudo | `pending_signature` \| `released` \| `rectified` |
| marca do valor | `normal` \| `low` \| `high` \| `critical_low` \| `critical_high` |
| tipo de assinatura | `icp` \| `advanced` |
| conselho | `CRF` \| `CRBM` \| `CRM` |

## 2. Formas comuns

`<lab_requisition>`:
```json
{ "id", "number", "patient": { "id", "display_name", "birth_date", "biological_sex" },
  "origin_unit": { "id", "name" }, "collection_place", "urgent", "status",
  "expected_delivery_on", "items": [<lab_item>], "samples": [<lab_sample>],
  "reports": [<lab_report_summary>], "delivered_to"|null, "delivered_at"|null }
```
`<lab_item>`: `{ "id", "exam_order_id", "sigtap_code", "label", "version": <n>, "sample_id"|null, "status" }`.
`<lab_sample>`: `{ "id", "barcode", "material", "container", "status", "collected_at"|null, "collected_by"|null, "received_at"|null, "rejection_reason"|null, "restriction_note"|null, "recollection_of_id"|null, "shipment_id"|null }`.
`<lab_result_row>`: `{ "id", "item_id", "analyte_key", "name", "value", "unit", "flag", "source", "entered_at", "entered_by", "chosen" }` — valores só para `lab_staff`/`lab_releaser` do laboratório.
`<lab_report_summary>`: `{ "id", "version", "partial", "status", "signature_kind", "released_at"|null, "released_by"|null, "short_code"|null, "pdf_url"|null, "replaces_report_id"|null }`.

## 3. Admin do laboratório (`/lab/admin`, `municipal_admin`; escrita com step-up)

- `GET /lab/admin/templates` (catálogo-base da plataforma, só leitura).
- `GET/POST /lab/admin/exams`, `GET /lab/admin/exams/:id` — adotar: `{ "provider_id", "template_id", "sector", "bench", "delivery_days" }`.
- `POST /lab/admin/exams/:id/versions` — `{ "method", "material", "container", "instrument_test_code"|null, "analytes": [ { "key", "name", "loinc"|null, "unit"|null, "value_type", "choices"?, "decimals", "reference_ranges": [ { "sex", "age_min_days", "age_max_days", "low"|null, "high"|null, "text"|null } ], "entry_min"|null, "entry_max"|null, "critical_low"|null, "critical_high"|null, "formula"|null } ] }` → nova versão (nunca edita). 422 `invalid_analytes`, `invalid_reference_ranges`, `invalid_formula`, `duplicate_instrument_code`.
- `GET/POST/PATCH /lab/admin/rejection_reasons[/:id]`.
- `GET/POST/DELETE /lab/admin/releasers[/:id]` — `{ "user_id", "provider_id", "technical_lead": bool, "technical_deputy": bool }`; 422 `council_missing` (sem conselho/UF/registro no perfil), `lead_taken`.
- `POST/DELETE /lab/admin/providers/:id/middleware_token` → `{ "token" }` uma vez.
- `PATCH /exams/admin/providers/:id` aceita `uses_municipal_lab`; `PATCH` da unidade aceita `collects_samples` (rota existente de unidades).
- Perfil profissional: `council`, `council_uf`, `council_number` (rota existente de profissionais).

## 4. Requisição, amostra e remessa (`/lab`, sessão do dashboard)

- `GET /lab/requisitions?status[]=&provider_id=&origin_unit_id=&number=&barcode=&before=&limit=` → `{ "requisitions": [<lab_requisition sem items/samples>], "next_before" }`.
- `GET /lab/requisitions/:id` → `<lab_requisition>`.
- `GET /lab/candidate_orders?patient_id=&provider_id=` → exames do 21a do paciente para o laboratório que podem entrar numa requisição (`awaiting_schedule`/`scheduled`, sem item).
- `POST /lab/requisitions` — `{ "patient_id", "provider_id", "origin_unit_id", "collection_place", "biological_sex", "mother_name", "urgent", "clinical_info"?, "exam_order_ids": [...], "paper_orders"?: [ { "sigtap_code", "requester": { "user_id" } | { "name", "cns" } } ] }` → 201 `<lab_requisition>` (cria `exam_orders` `lab_reception` para `paper_orders`). Quem: enfermagem da UBS com `collects_samples` (`health_professional` da unidade) ou `lab_staff`. 422 `order_not_eligible`, `exam_not_in_lab_catalog`, `mother_name_required`, `biological_sex_required`.
- `GET /lab/requisitions/:id/receipt.pdf` (comprovante) e `GET /lab/requisitions/:id/labels.pdf?sample_ids[]=` (etiquetas; reimpressão grava trilha).
- `POST /lab/samples/:id/collect` — `{ "collected_at"? }` (por id ou `POST /lab/samples/collect { "barcode" }`).
- `POST /lab/shipments` — `{ "origin_unit_id", "provider_id", "barcodes": [...], "temperature_celsius" }`; `POST /lab/shipments/:id/send`; `POST /lab/shipments/:id/receive { "barcodes": [...] }` (lidos; os da lista não lidos viram `not_received`).
- `POST /lab/samples/receive` — `{ "barcode", "decision": "accept" | "reject" | "accept_with_restriction", "reason_id"?, "restriction_note"? }`. Rejeição cria a recoleta (amostra nova `awaiting` na UBS de origem, `recollection_of_id`). 409 `invalid_transition`; 404 `not_found` (código desconhecido).
- `GET /lab/recollections?origin_unit_id=` → amostras de recoleta da UBS.
- `POST /lab/requisitions/:id/deliver { "delivered_to" }`.

## 5. Mapa, digitação e liberação

- `GET /lab/worklists?provider_id=&sector=&bench=&urgent=&date=&pending=true` → itens com amostra recebida, agrupados por setor/bancada; `GET …/worklists.pdf` (mesmos filtros).
- `POST /lab/results` — `{ "item_id", "values": [ { "analyte_key", "value" } ] }` (ou por `barcode`); calcula `formula`, marca `flag`; 422 `out_of_entry_range` (com `analyte_key`), `unknown_analyte`, `sample_not_received`.
- `POST /lab/results/:id/choose` — escolhe a repetição que vale (linha nova).
- `GET /lab/releases?provider_id=` → fila de liberação (itens `typed`); `GET /lab/criticals/unnotified`.
- `POST /lab/criticals` — `{ "result_id", "notified_person", "notified_role", "channel", "notified_at" }`.
- `POST /lab/requisitions/:id/release` — `{ "item_ids": [...], "signature": "icp" | "advanced", "totp"? }` → `<lab_report_summary>`. Regras: só `lab_releaser` do laboratório; CRM → só `icp` (422 `icp_required`); `advanced` exige TOTP válido (401 `mfa_required`); `icp` sem certificado ativo → 422 `no_active_certificate`; crítico sem aviso → libera e entra em `criticals/unnotified`. `icp` nasce `pending_signature` (pedido do 19b na mesma transação) e o 21a só recebe o resultado quando assinado; `advanced` nasce `released`.
- `POST /lab/reports/:id/rectify` — `{ "item_ids", "reason", "contacts": [ { "person", "channel", "at" } ], "signature", "totp"? }` → novo laudo; anterior `rectified`.
- `GET /lab/reports/:id/pdf` (URL curta assinada, A4 do 21a; trilha).
- Conferência pública: a página `/v` do 19c aceita o `short_code` do laudo (categoria "laudo laboratorial", nunca resultado nem CPF).

## 6. Relatórios (`municipal_admin`, `lab_releaser`)

- `GET /lab/reports/production?provider_id=&month=` (+ `.csv`) → linhas por unidade de origem, exame, setor e liberador com quantidade e valor SIGTAP.
- `GET /lab/reports/rejections?provider_id=&month=`; `GET /lab/reports/turnaround?provider_id=&month=` (no prazo × atrasado).

## 7. Entrada HL7 (`lab_hl7_inbound`, host da cidade, sem cookie)

- `POST /hl7/v2`, `Authorization: Bearer <token do middleware>`, `Content-Type: application/hl7-v2` (ER7, `\r` como separador de segmento). Resposta `200` com `ACK` (ou `RSP^K11`) em `application/hl7-v2`; `MSA-1` `AA`/`AE`/`AR`; `ERR-3` com código, nunca dado de paciente. HTTP 401 só sem token; token revogado → `AR`. 429 por token (120/min).
- `QBP^Q11` (`QPD-3` = código de barras) → `RSP^K11` com `ORC`/`OBR` por exame da amostra, `OBR-4` = `instrument_test_code`.
- `OUL^R22`: `SPM-2` = código de barras; `OBR-4` = `instrument_test_code`; `OBX-3` = `instrument_test_code^analyte_key`; `OBX-5` valor; `OBX-6` unidade. `AE` com `ERR-3`: `unknown_barcode`, `sample_not_received`, `exam_not_in_version`, `unknown_analyte`, `out_of_entry_range`, `unit_mismatch`. `MSH-10` repetido do mesmo token → `AA` sem efeito.
- Mensagem bruta cifrada em `lab_results.raw`; resultado entra como `source: hl7`, nunca liberado sozinho.
- `contracts/lab/hl7-v2-profile.md` (perfil) + exemplos em `contracts/lab/examples/*.hl7`.

## 8. Laudo (`contracts/lab/lab-report-v1.json`)

JSON canônico assinado/com hash: laboratório (`name`, `cnes`, `address`, `phone`), `technical_lead` e `released_by` (`name`, `council`, `council_uf`, `council_number`), paciente (`name`, `birth_date`, `biological_sex`, `record_number`), `requisition_number`, por exame: `sigtap_code`, `name`, `material`, `method`, `version`, `collected_at`, `restriction_note`, componentes (`name`, `value`, `unit`, `flag`, `reference_text`, `loinc`); `partial`, `issued_at`, `signature_kind`, `replaces_report_id`, `rectification_reason`. PDF gerado deste JSON.

## 9. Volta ao 21a

- Liberação (`released`) cria, por `exam_order` liberado, `exam_result` com `source: lab`, `values` com LOINC, `critical` (algum `critical_*`), PDF do laudo; evento `exam_order.resulted`; a regra de escalonamento e de ciência do 21a vale.
- Retificação: `exam_result` novo com `replaces_result_id`; o anterior fica `superseded` (só profissional vê, com marca); o exame volta a `resulted` e perde a ciência; evento `exam_order.result_rectified`. O cidadão vê só o vigente.
- O portal do prestador e a entrada FHIR do 21a recusam resultado de prestador `uses_municipal_lab` (422 `provider_uses_municipal_lab`).

## 10. Maintenance (GraphQL)

- Interruptores `municipal_lab` (`["feature:exam_tracking"]`), `lab_hl7_inbound` (`["feature:municipal_lab"]`), `fake_middleware` (`["feature:lab_hl7_inbound"]`, `environments: SIMULATION_ENVS`).
- Catálogo-base: `labExamTemplates` (lista) e mutations `upsertLabExamTemplate` / `upsertLabAnalyteTemplate` → `{ ok, errors }`; evento de plataforma `lab.template_changed` (em R18).

## 11. Eventos (só ids)

`lab_requisition.opened`, `lab_sample.collected`, `lab_sample.received`, `lab_sample.rejected`, `lab_shipment.sent`, `lab_shipment.received`, `lab_result.entered` (`source`), `lab_report.released` (`signature_kind`, `partial`), `lab_report.rectified`, `lab_critical.notified`, `exam_order.result_rectified`; `clinical_record.viewed` + `access: lab_report`.

## 12. Middleware simulado (`fake-middleware`, compose de dev)

`FAKE_MIDDLEWARE_API_URL`, `FAKE_MIDDLEWARE_CITY_HOST`, `FAKE_MIDDLEWARE_TOKEN`, `FAKE_MIDDLEWARE_INTERVAL`; a cada ciclo lista amostras recebidas pelo `QBP^Q11` das que conhece (lista vinda de `GET /lab/hl7/pending_barcodes` com o mesmo token) e envia `OUL^R22` de exemplo, incluindo um crítico e uma repetição. Com `fake_middleware` desligado, o api responde `AR`.

## 13. Ordem de entrega

0. Provas técnicas (plano do api, Task 0): perfil HL7 2.5.1 das quatro mensagens; etiqueta Code 128 em PDF para impressora térmica, lida por leitor.
1. `contracts` — `lab/hl7-v2-profile.md`, `lab/lab-report-v1.json`, exemplos; tag `lab-v1.0.0`.
2. `api` (porta de dev 3042; migrações de cidade e plataforma a partir de 20261015900001; worktree `.claude/mod21b`, branch `feat/mod-21b-lab`, bancos `_mod21b`).
3. `dashboard` (Vite 5194; worktree em `rota-saude/.worktrees/dashboard-mod21b`). 4. `maintenance`. 5. catálogo público. O wpda não muda.

## 14. Acréscimos (revisão dos planos, 2026-10-15)

Aceitos pelo coordenador; prevalecem sobre as seções acima. Detalhe nas seções finais de cada plano (`2026-10-15-module-21b-lab-{api,dashboard,maintenance}.md`).

**maintenance (M1–M7)**
- **B1 — SDL do catálogo-base:** tipos `LabExamTemplate`, `LabAnalyteTemplate`, `LabReferenceRange`, `LabReferenceRangeInput`; `upsertLabExamTemplate` / `upsertLabAnalyteTemplate` com `id` nulo para criar; payload `{ ok, errors }`; sem step-up; sem exclusão (`active: false`). O texto exato está no plano do maintenance e o api o reproduz (teste de comparação).
- **B2 — Códigos:** `missing` `municipal_lab_disabled`, `lab_hl7_inbound_disabled`; setores `biochemistry` | `hematology` | `immunology` | `urinalysis` | `parasitology` | `microbiology`; `UserError` `sigtap_unknown`, `duplicate_sigtap`, `invalid_sector`, `blank`, `duplicate_analyte_key`, `invalid_analyte_key`, `invalid_value_type`, `choices_required`, `invalid_reference_ranges`, `invalid_loinc`, `invalid_decimals`, `not_found`.
- **B3 — Faixas:** `age_min_days`/`age_max_days` obrigatórios (0 a 43800 = sem limite); `low`/`high` nulos = sem limite daquele lado. Vale também para as versões da cidade (o dashboard não aceita idade nula).
- **B4 — Aviso de middleware simulado** na aba de interruptores (como A25 do 21a).

**dashboard (D1–D20) — adotadas pelo api**
- **B5 — Rotas de apoio:** `GET /lab/context`; `GET /lab/exams?provider_id=`; `GET /lab/shipments` e `<lab_shipment>`; `GET /lab/entry?item_id=|barcode=&sigtap_code=` (com faixa resolvida; grava trilha); `GET /lab/releases/:id`; `GET /lab/criticals/unnotified/count` (sem trilha, molde da A46); `PATCH /lab/admin/exams/:id`; CSV nos três relatórios; `GET /lab/rejection_reasons` (com `provider_id`) para `lab_staff`; `POST /exams/orders/:id/gal_requisition` (409 `gal_already_registered`; `lab_staff` também registra).
- **B6 — Formas e regras:** `candidate_orders`, `{ sample, recollection }` no recebimento, `recollections` + `not_received`, mapa em `groups`, fila de liberação e críticos (sem valor no contador), relatórios (`sigtap_value_cents` = quantidade × valor unitário; 0 sem valor na SIGTAP) como no plano do dashboard; `patient_lookup` aceita `lab_staff`; adotar cria a v1 copiada do template; `POST /lab/results` aceita item já liberado (valor para retificação); erros de remessa com `barcode`; `reason_required`, `restriction_note_required`, `unit_does_not_collect`.
- **B7 — Conselho do módulo 10:** usa `council`, `council_state`, `registration_number` do perfil profissional (no lugar de `council_uf`/`council_number`); `lab_releaser` pode ter perfil sem `health_professional`.
- **B8 — Substituição no 21a:** `exam_result.replaces_result_id`; vigente = o não substituído; `superseded_results`/`superseded_at` só na visão clínica; evento `exam_order.result_rectified`. Sem UPDATE.
- **B9 — Prestadores:** `uses_municipal_lab` e `has_middleware_token` em `/exams/admin/providers`; `collects_samples` na rota real de unidades (`POST`).

**api (P1–P21)**
- **B10 — HL7:** nosso código de erro em `ERR-5` (`ERR-3` é a tabela 0357); `OBX-3` = `analyte_key^nome^99RSL`, exame em `OBR-4`; `RSP^K11` sem `PID`, `QAK-2` `NF` sem pendência; `OML^O33` fora desta entrega. Bruto uma vez por mensagem aceita (`lab_hl7_receipts`, cifrado); mensagem entra inteira ou nada. HTTP 401 sem token ou desconhecido; `AR` token revogado; 403 `lab_hl7_inbound` desligado; `AR feature_disabled` token simulado com `fake_middleware` desligado; 413 acima de 1 MB.
- **B11 — Quem atua:** `lab_staff` vale para todo laboratório municipal da cidade; `lab_releaser` só no laboratório da ligação.
- **B12 — Recoleta e cancelamento:** o 21a aceita `scheduled` a partir de `collected` só pelo laboratório; pedido em papel de exame que exige autorização → `order_not_eligible`; cancelar exame cancela item pendente; item digitado → 409; unidade e portal recusam resultado de prestador `uses_municipal_lab`.
- **B13 — Laudo:** itens em `lab_report_items`; PDF em `exam_files` (cópia por `exam_result`); conteúdo congelado e cifrado; situação nova `withdrawn` (o sistema devolve pedido ICP quando `digital_signature` é desligado — o dashboard trata `withdrawn`); `lab-report-v1` com `key` e `support_laboratory`; endereço e telefone do prestador obrigatórios (art. 167, II).
- **B14 — Erros da liberação:** `invalid_signature`, `laboratory_incomplete`, `technical_lead_missing`, `signature_unavailable` (médico em cidade sem `digital_signature` utilizável).
- **B15 — Pendências de assinatura:** laudo ICP aparece em `GET /signature/requests` (rota real, não `/signature/pending`) como `lab_report`; `canSign` vale para `lab_releaser`; "voltar ao papel" → 422 `not_applicable`.
- **B16 — Números:** requisição com 8 dígitos; código de barras = número + 2 dígitos; digitação pela etiqueta exige `sigtap_code`; lista de requisições pela UBS exige `origin_unit_id` (422 `filter_required`).
- **B17 — Valor SIGTAP:** `sigtap_procedures.amount_sa_cents` (plataforma); significado de `VL_SA` a conferir no layout oficial (Task 3).
- **B18 — `/v` do laudo:** `kind: "lab_report"`, `pdf_sha256`, nenhum PDF público.
- **B19 — Eventos a mais:** `lab_sample.label_printed`, `lab_requisition.delivered`, `lab_report.withdrawn`, `lab_exam.version_created`, `lab_releaser.assigned`/`.ended`, `lab_middleware.token_rotated`/`.token_revoked`, `exam_order.gal_registered`; plataforma `lab.template_changed` e `maintenance.lab_template.changed` (R18).
- **B20 — `contracts`:** além de `lab-v1.0.0`, `exams/exam-order-v1.json` v1.1 (`lab_reception`, `lab`, substituição) com tag `exams-v1.1.0`.
- **B21 — Dev:** api `api-dev` com servidor do worktree na 3042 (`exec -d -w /rails/.claude/mod21b`); contas `tecnica-lab@curitiba.demo` (`lab_staff`), `biomedica@curitiba.demo` (RT, CRBM), `patologista@curitiba.demo` (CRM, substituto, só ICP — certificado do PSC simulado vinculado no navegador), senha `dev-password` e TOTP fixo; "Laboratório Municipal Central"; UBS Jardim das Flores coleta, UBS Vila Esperança não.

**Execução (provas técnicas, 2026-10-15)**
- **B22 — HL7, detalhes do perfil:** `SPM-4` (tipo do espécime, obrigatório na v2.5.1) vem do `material` da versão do exame (código e texto); `ERR` é repetível no `ACK` e no `RSP^K11` (um por recusa); o `RSP^K11` usa o grupo `ORC`/`OBR` definido no perfil `lab/hl7-v2-profile.md` (não `RDF`/`RDT`).
- **B23 — Etiqueta:** nome que não cabe é cortado com reticências; o código de barras e o número nunca são cortados.
- **B24 — `exam-order-v1` continua MINOR:** os campos novos da v1.1 (`gal_requisition` e os da substituição) entram **opcionais** no esquema (fora de `required`, aceitando `null`), o que mantém a tag `exams-v1.1.0` sem quebrar documento antigo; o api sempre envia as chaves. Mesmo critério do `clinical-v1.1.0` (19c).
- **B25 — `SPM-4` na leitura:** o Rota Saúde escreve `material_key^rótulo^99RSL` (código local; sem a tabela HL7 0487) e não recusa `OUL^R22` sem `SPM-4` (a amostra é identificada por `SPM-2`).
- **B26 — Todas as recusas:** a leitura de `OUL^R22` confere a mensagem inteira e devolve um `ERR` por recusa (não para na primeira); a mensagem continua entrando inteira ou nada (B10).
- **B27 — Limite crítico inclusivo:** valor igual a `critical_low` ou `critical_high` é crítico (`<=`/`>=`). O mesmo vale para a faixa de referência: valor igual a `low`/`high` é normal (limites da faixa pertencem a ela).
- **B28 — `material_key` derivado:** sem coluna nova; `material_key` = `material` da versão transliterado (sem acento), minúsculo, não alfanumérico → `_`, sem `_` nas pontas (ex.: "Sangue total (EDTA)" → `sangue_total_edta`); rótulo = o texto do `material` com escape HL7. A mesma regra no `RSP^K11` e no perfil; na leitura o `SPM-4` não é conferido (B25).
- **B29 — Opções de componente `choice`:** no máximo 20 opções, cada uma com até 60 caracteres, sem vazia nem repetida; fora disso `choices_required`. Vale no catálogo-base e nas versões da cidade.
- **B30 — Valor que vale:** por componente, o valor efetivo é a última linha (escolher uma repetição cria cópia nova, que vira a última). `/lab/entry` e `/lab/releases[/:id]` trazem `effective: true|false` por linha, calculado no api; telas, crítico e laudo usam só `effective`; `chosen` fica como história.
- **B31 — Endereço e telefone do laboratório:** o formulário de prestador do 21a no dashboard ganha "Endereço" e "Telefone", obrigatórios quando `uses_municipal_lab` (o `PATCH /exams/admin/providers/:id` já aceita); sem eles a liberação dá `laboratory_incomplete`.
- **B32 — Contexto vazio:** `GET /lab/context` com papel de laboratório e nenhum laboratório → 200 `{ "labs": [] }`; 403 só com interruptor desligado ou sem papel.
- **B33 — Perfil do liberador:** `GET /professionals/pending` lista também quem só tem `lab_releaser` (B7).
- **B34 — Erros do catálogo do laboratório:** 422 `invalid_version` (método, material, recipiente ou `instrument_test_code` inválidos na nova versão), `choices_required` (B29) também nas versões da cidade, `invalid_template` e `invalid_provider` na adoção. Também: `invalid_delivery_days` (1 a 90) e `invalid_bench` (até 60 caracteres) na adoção e no `PATCH` do exame; `invalid_name`, `invalid_active` e `duplicate_name` nos motivos de rejeição.
- **B35 — A marca do prestador manda:** prestador `uses_municipal_lab` recusa resultado pelo portal, pela FHIR e pela rota da unidade independentemente do interruptor `municipal_lab`; com o interruptor desligado, o laboratório não entrega por canal nenhum até a marca ser retirada. O formulário do prestador avisa isso ao marcar e ao desmarcar.
- **B36 — `instrument_test_code` único por prestador,** sem diferenciar maiúsculas de minúsculas, entre as versões vigentes de exames ativos (o `OBR-4` casa sem ambiguidade).
- **B37 — Exames candidatos entre unidades:** `GET /lab/candidate_orders` mostra a quem coleta (UBS com `collects_samples` ou `lab_staff`) os exames do 21a do paciente vindos de qualquer unidade da cidade, com o solicitante — o paciente pode coletar fora da sua UBS. Como o nome do exame é dado de saúde, a leitura exige `patient_id` (sem listagem aberta) e grava `clinical_record.viewed` com `access: lab_candidates`; a tela não faz polling dela.
- **B38 — Ordem única de travas:** requisição → itens → pedido do 21a, em cancelamento, liberação e retirada por adendo; corrida no cancelamento refaz uma vez e devolve 409 se ainda divergir. Retirada por adendo segue a B12 (item pendente cancela junto; item digitado ou liberado mantém o exame sem abortar o adendo).
- **B39 — Leitura da etiqueta sem exame:** `GET /lab/entry?barcode=` sem `sigtap_code` devolve só os itens da amostra (`item_id`, `sigtap_code`, nome, situação), com uma trilha; com `sigtap_code` (ou `item_id`), o formulário de digitação como antes. O `POST /lab/results` por etiqueta continua exigindo `sigtap_code` (B16). O dashboard deixa de passar pela requisição.
- **B40 — Etiqueta e coleta:** `labels.pdf` deduplica os ids e nunca imprime amostra `rejected` ou `not_received`; sem amostra imprimível → 422 `invalid_samples`. Coleta com `collected_at` ilegível ou no futuro → 422 `invalid_collected_at`. A etiqueta tem 4 linhas (nome; nascimento; material e recipiente; número) acima do código, com reticências no que não couber (B23).
- **B41 — Gem nova:** `barby` 0.7.0 no `Gemfile` do api; a imagem do api precisa ser reconstruída no deploy (runbook).
- **B42 — Itens do laudo:** `<lab_report_summary>` ganha `item_ids` (itens da requisição cobertos pelo laudo, de `lab_report_items`); a retificação usa esses itens como padrão.
- **B43 — Recoletas:** `/lab/recollections` só para a UBS de origem, `lab_staff` e admin (não para `lab_releaser`, pois traz o nome do paciente); lista só o que falta coletar; rejeitar amostra sem item pendente não cria recoleta.
- **B44 — Guarda da mensagem HL7 bruta (decisão do usuário):** `lab_hl7_receipts.raw` é guardado por 5 anos a partir da mensagem aceita e depois purgado para `NULL` por job diário (`EachCityJob`); o recibo permanece. O job é pendência própria (board #2), fora desta entrega.
- **B45 — Endereço e telefone exigidos ao marcar:** o update do prestador recusa marcar `uses_municipal_lab` sem endereço e telefone (422 `invalid_address` / `invalid_phone`); `laboratory_incomplete` na liberação continua como defesa. Outros códigos da edição: `invalid_uses_municipal_lab`, `municipal_lab_disabled` (prestador) e `invalid_collects_samples` (unidade).
- **B46 — Troca de RT/substituto:** sem rota própria nesta entrega; troca = encerrar a designação e designar de novo (a tela orienta a ordem: encerrar o RT atual e depois designar o novo). Enquanto não houver RT, a liberação dá `technical_lead_missing`.
- **B47 — Calculados e mapa:** escolher repetição de componente calculado → 422 `unknown_analyte`; entrada que deixa um calculado sem conta é recusada inteira (422 `invalid_values` com a chave do calculado); valores até 15 algarismos; o mapa de trabalho (JSON e PDF) é só para membros do laboratório (traz nome de paciente).
- **B48 — Endurecimento HL7:** `MSH-11` só `P`/`D`/`T`; CR/LF e caracteres de controle viram espaço; `NUL` ou segundo `MSH` recusados; `OBR`/`OBX` órfãos recusados; `QAK-2` `OK` só com amostra conhecida, senão `NF`.
- **B49 — Item já fechado:** resultado HL7 para item `released` ou `cancelled` → `MSA-1` `AE`, `ERR-3` 207, `ERR-5` `item_not_open`, texto "Exame ja liberado ou cancelado", localização `OBR^n^4`. Item liberado só recebe valor pela digitação manual, para a retificação (B6).
- **B50 — Estrutura sempre `AR`:** `invalid_message` (estrutura inválida: sem `SPM`, `OBR`/`OBX` órfão, segmento obrigatório faltando) responde sempre `AR`, com ou sem `SPM` presente; `AE` fica para recusas de conteúdo (amostra, exame, componente, valor, item fechado). O perfil `lab/hl7-v2-profile.md` acompanha.
- **B51 — Um aviso por resultado:** índice único em `lab_critical_notices.lab_result_id`; segundo `POST /lab/criticals` do mesmo resultado → 409 `conflict`. Valor retificado é resultado novo e pede aviso novo. Horário do aviso entre o recebimento da amostra e agora + 1 min.
- **B52 — Fila de críticos sem aviso:** entra o crítico que é a linha efetiva do laudo liberado mais recente do item e não tem aviso; linha digitada para retificação só conta quando o laudo retificado é liberado; repetição escolhida depois da liberação não esconde crítico já liberado sem aviso. **Sem corte por idade:** o crítico fica na fila até ter aviso (lista paginada, mais antigos primeiro), para nunca sumir sem comunicação.
- **B53 — Crítico liberado nunca some (precisa B52):** todo valor crítico que foi a linha efetiva de **qualquer** laudo liberado do item (não só do mais recente) fica na fila até ter aviso; retificar para valor normal não apaga a obrigação de avisar o que já foi liberado.
- **B54 — Laudo ICP congelado:** o conteúdo do laudo ICP congela na criação (`pending_signature`); a linha efetiva é calculada até esse momento; entrada e escolha de resultado são recusadas para item coberto por laudo `pending_signature` (o valor já está no documento a assinar).
- **B55 — Liberação, códigos e limites:** `item_frozen` (422) em `POST /lab/results` e na escolha de repetição para item coberto por laudo `pending_signature` (B54); liberar de novo item congelado → 409 `invalid_transition`; pela HL7, `AE` `item_not_open` (B49). `invalid_content` (422) quando o conteúdo congelado não passa no esquema (defesa). Liberação com mais de 50 itens ou com item incompleto → 409 `invalid_transition`. `/lab/releases` só para membros do laboratório (traz valores e nomes). Laudo `released` tem `released_at` e PDF; `withdrawn` não tem nenhum dos dois (trigger). Cada liberação guarda uma cópia do PDF no laudo e uma por resultado do 21a.
- **B56 — Falha permanente na finalização ICP:** quando a finalização do laudo falha de forma permanente na assinatura, o laudo vai a `withdrawn` (como em B13) e os itens descongelam para nova liberação; o pedido do 19b fica com o motivo terminal existente `document_cancelled` (sem código nem migração novos).
- **B57 — Escopo do liberador no 19b:** `lab_releaser` passa a assinar pelo 19b (certificado e sessão), mas lista, lote e "voltar ao papel" ficam limitados a pedidos de `LabReport` (escopo `SignatureRequest.signable_by`).
- **B58 — Código curto único:** o `short_code` do laudo é gerado sem colidir com os de `clinical_documents` (e vice-versa); a conferência `/v` ainda desempata por ano de nascimento como defesa e, persistindo o empate, responde não encontrado.
- **B59 — Liberador encerrado não assina:** encerrar a designação do liberador leva a `withdrawn` os laudos `pending_signature` dele naquele laboratório; ao assinar, liberador sem designação ativa cai na falha permanente da B56 (`withdrawn` + `document_cancelled`); a assinatura em lote e em segundo plano aplica `signable_by` (quem só tem `lab_releaser` nunca assina consulta ou receita pela sessão aberta).
- **B60 — Retificação substitui o laudo inteiro (completa B42):** o laudo retificador cobre todos os itens do laudo retificado; os itens não alterados entram com os mesmos valores, congelados de novo. `item_ids` em `POST /lab/reports/:id/rectify` são só os itens que mudaram: só eles ganham resultado novo no 21a e perdem a ciência; os demais mantêm resultado e ciência. `contacts[].at` entre o `released_at` do laudo original e agora + 1 min. Duas retificações simultâneas do mesmo laudo → 409.
- **B61 — GAL na visão administrativa:** `gal_requisition` aparece em todas as visões do exame no 21a (é dado administrativo); cidadão, portal do prestador e FHIR não o recebem.
- **B62 — Itens da retificação conferidos:** `item_ids` que não batem com o que mudou → 422 `rectify_items_mismatch` com `{ "error", "unchanged": [item_id], "unlisted": [item_id] }` (listas sempre presentes e ordenadas; `unchanged` = listado sem mudança, `unlisted` = mudou e não foi listado). Problemas de estado seguem 409 `invalid_transition` (laudo não `released`, item fora do laudo, `pending_signature`, componente incompleto). Retificação ICP que falha (B56): o novo laudo vai a `withdrawn`, o anterior continua `released` e o item descongela.
