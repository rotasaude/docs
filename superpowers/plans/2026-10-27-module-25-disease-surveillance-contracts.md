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

## 14. Ficha de notificação individual

Fixado pela Task 0 do api (`config/surveillance/sinan-form.yml`, versão "Sinan NET v5 — notificação
individual (campos 1–30)"; fontes em `apps/api/vendor/surveillance/2026-10-27/`). `form` é um objeto
com estas chaves (todas opcionais; ausente ou `null` = não preenchido); valor fora do domínio ou do
limite → 422 `invalid_form` com `path: "form.<chave>"`. O campo é o número no bloco de notificação
individual da Ficha de Notificação/Conclusão e das fichas de investigação do Sinan (a mesma numeração
do dicionário de dados). Nome, nascimento, idade, sexo, Cartão SUS e nome da mãe (8–11, 15, 16) vêm do
paciente; agravo + CID, datas, unidade e município de residência (2–7, 18) vêm da notificação — só
leitura, fora de `form`.

| chave | campo | tipo | domínio (código: descrição) | limite | pré-preenchido de |
|---|---|---|---|---|---|
| `pregnancy` | 12 Gestante | código | 1: 1º trimestre · 2: 2º trimestre · 3: 3º trimestre · 4: Idade gestacional ignorada · 5: Não · 6: Não se aplica · 9: Ignorado | — | vazio |
| `race_color` | 13 Raça/Cor | código | 1: Branca · 2: Preta · 3: Amarela · 4: Parda · 5: Indígena · 9: Ignorado | — | raça/cor do paciente convertida (BPA `01→1`, `02→2`, `03→4`, `04→3`, `05→5`; sem raça → `9`) |
| `schooling` | 14 Escolaridade | código | 0: Analfabeto · 1: 1ª a 4ª série incompleta do EF · 2: 4ª série completa do EF · 3: 5ª à 8ª série incompleta do EF · 4: Ensino fundamental completo · 5: Ensino médio incompleto · 6: Ensino médio completo · 7: Educação superior incompleta · 8: Educação superior completa · 9: Ignorado · 10: Não se aplica | — | vazio |
| `residence_uf` | 17 UF | UF (sigla) | 27 UFs | 2 | endereço do paciente |
| `residence_district` | 19 Distrito | texto | — | 60 | vazio |
| `residence_neighborhood` | 20 Bairro | texto | — | 60 | endereço do paciente |
| `residence_street` | 21 Logradouro | texto | — | 60 | endereço do paciente |
| `residence_number` | 22 Número | texto | — | 6 | endereço do paciente |
| `residence_complement` | 23 Complemento | texto | — | 30 | endereço do paciente |
| `residence_geo1` | 24 Geo campo 1 | dígitos | código da tabela georreferenciada do município | 1–8 | vazio |
| `residence_geo2` | 25 Geo campo 2 | dígitos | código da tabela georreferenciada do município | 1–8 | vazio |
| `residence_reference` | 26 Ponto de referência | texto | — | 70 | vazio |
| `residence_zip` | 27 CEP | dígitos | — | 8 exatos | endereço do paciente |
| `phone` | 28 (DDD) Telefone | dígitos | DDD + número | 10–11 | telefone do par validado |
| `residence_zone` | 29 Zona | código | 1: Urbana · 2: Rural · 3: Periurbana · 9: Ignorado | — | vazio |
| `residence_country` | 30 País (se residente fora do Brasil) | texto | — | 40 | vazio |
| `occupation_cbo` | — ("Ocupação" das fichas de investigação, número variável 31–34; não existe na Notificação/Conclusão nem na NINDIV) | CBO | código CBO | — (tamanho fora do dicionário baixado) | ocupação da última versão do cadastro individual (24a) |

**Achados da Task 0**

- **Semana epidemiológica (V1/K1):** conferida nos calendários do Sinan de 2025 e 2026 — `2025-12-28..2026-01-03` = `2025-53`; `2026-01-04` = `2026-01`; `2026-10-27` = `2026-43`; `2027-01-02` = `2026-52` (2026 não tem semana 53). Regra: domingo a sábado; semana 1 = a que tem mais dias de janeiro (= a que contém 4 de janeiro).
- **Numeração da ficha:** a NINDIV v5 genérica (negativa/individual/surto) numera a residência 20–33; a numeração acima (17–30) é a do bloco individual da Ficha de Notificação/Conclusão, das fichas de investigação e do dicionário de dados. Ocupação só existe nas fichas de investigação, com número variável (31–34; ausente em tuberculose e PFA) e sem tamanho no dicionário da notificação individual — por isso sem número de campo.
- **Limites do dicionário de dados do Sinan:** ponto de referência 70; geocampos são códigos numéricos (até 8 dígitos) da tabela do município; distrito e país são códigos de tabela no Sinan — a ficha guarda o nome (60 e 40 são proposta).
- **Caxumba:** semanal (anexo), não imediata (art. 2º); a doença carrega a nota.
- **CIAP-2:** só rubricas que correspondem a uma doença da lista (`A71`, `A72`, `A73`, `B90`, `D71`, `D72`, `R71`, `A84`, `Z25`); ambíguas fora (`A70` e `N70` incluem efeitos tardios/sequelas, `A74`, `A76`, `A77`, `A78`, `X70`, `Y70`, `N71`, `N72`, `D70`, `A86`, `S13`).
- **Formas imediatas sem CID própria** (óbito por dengue, Zika em gestante e óbito, chikungunya em área sem transmissão e óbito, Oropouche óbito/gestante/anomalia, violência sexual e tentativa de suicídio): ficam em `notes` da doença; a vigilância notifica à mão. **15 doenças sem CID** (`cid10: []`) só entram por notificação manual.
- **`X22` (escorpião) é da lista nacional** (acidente por animal peçonhento): não serve para doença local.

## 15. Acréscimos (revisão dos planos, 2026-10-27)

Reconciliação dos planos do api (V1–V17), do dashboard (D1–D16) e do maintenance (M1–M6). O §14 é a ficha de notificação individual, fixada pela Task 0 do api.

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
- **K13 — Prova técnica (Task 0 do api, 2026-10-27):** (a) SRAG só por notificação manual (o J11 da ficha é o da gripe sazonal); (b) malária com prazo imediato (região extra-amazônica), com nota para a região amazônica; (c) códigos genéricos impressos nas fichas — F99, L98.9, C80 e Z20.9 — não disparam sugestão (doença só manual, com nota); W64, Y09, O98.6, A92.8 e Z57.9 continuam; validar com a vigilância piloto (runbook); (d) a semente usa um agravo de notificação estadual do PR com fonte, ou uma doença local fictícia rotulada de dev; (e) tuberculose ganha os códigos extrapulmonares do GVS que não colidem. Lacunas registradas: P35.4 fora da tabela DATASUS 2008; Oropouche apoiado na NT 117 (cópia da SES-MG); resposta do Fala.BR sobre a caxumba não baixada.
