# Módulo 23 — Linhas de cuidado (api) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **PRÉ-REQUISITO: módulo 22 em `origin/main` do api** (conferido ao escrever este plano em `c5b6a9d`: `Platform::Features::CATALOG` terminando em `procedure_billing`; última migração de cidade `20261021900001_add_procedure_billing.rb` e de plataforma `20261019100002_add_health_department_name_to_cities.rb`; `db/city_schema.rb` em `define(version: 2026_10_21_900001)`; `db/platform_schema.rb` em `define(version: 2026_10_19_100002)`; `spec/adr_pointers_spec.rb` com `VALID_RANGE = (1..39)`; `patient_problem.changed` ligado a `[]` em `config/initializers/domain_events.rb`; `Triages::Offer::Item` com 8 campos; `Citizens::RequestErasure.examined_patients?` terminando em `PerformedProcedure`; `MaintenanceAudit::NAMES` terminando em `maintenance.lab_template.changed`). **A Task 0 confere a base e a prova técnica: se um nome mudou, ou se uma nota metodológica vigente não é mais a de 19/06/2026, PARE e reporte.**

**Goal:** O lado api do módulo 23 (F-23.1 a F-23.8, ADR 0040): interruptor `care_lines`; catálogo-base versionado das seis linhas C2–C7 na plataforma, com as boas práticas das notas metodológicas vigentes, e o GraphQL do maintenance; linhas da cidade (adoção e próprias); sugestões diárias e por evento (problema ativo, idade e sexo, condição declarada, e sugestão de encerramento por problema resolvido) com inscrição sempre confirmada por médico ou enfermeiro; avaliação das boas práticas na leitura, em lote (uma consulta por fonte por página); painel nominal com busca ativa e contato; prévia C2–C7 por equipe; condição declarada e "Meu acompanhamento" do cidadão; aviso de triagem periódica vencida; exclusão LGPD; semente de dev em Curitiba; runbook.

**Architecture:** Uma migração de plataforma (`care_line_templates`, só acréscimo — versão nova é linha nova) e uma de cidade (`care_lines`, `care_line_enrollments`, `care_line_suggestions`, `care_outreach_contacts`, `citizen_declared_conditions`). As regras (elegibilidade e boas práticas) são JSON validado por um único `CareLines::Rules`, o mesmo para o catálogo-base e para as linhas próprias. A avaliação é uma função pura (`CareLines::Evaluate`) sobre eventos datados que um carregador em lote (`CareLines::Sources`) lê das fontes existentes (consulta 19a, escuta 18, exames 21a/21b, procedimentos 22) — nada clínico é copiado. A população de uma equipe (`CareLines::TeamPopulation`) e a elegibilidade pura (`CareLines::Eligibility`) servem às sugestões e à prévia.

**Tech Stack:** Rails 8.1 (API), PostgreSQL 16 (banco por cidade + plataforma), RSpec, graphql-ruby (API de manutenção), Solid Queue (job diário). Nenhuma gem nova.

**Spec:** `docs/superpowers/specs/2026-10-22-module-23-care-lines-design.md` e `docs/adr/0040.md` (leia os dois antes). Contrato entre apps — **fonte única de formatos**: `docs/superpowers/plans/2026-10-22-module-23-care-lines-contracts.md` (convenções dos contratos do 21a §15, 21b §14, 21c §11 e 22 §12–§13); toda diferença está em "Divergências propostas ao contrato", no fim (prefixo **L**). Pesquisa: `docs/pesquisa/2026-10-05-integracoes/frente-1-aps-sisab.md` (§5 cofinanciamento, §6 C1–C7). Lições que valem aqui: contadores sem trilha; ordem de travas estável; recusar em vez de cortar (texto longo → 422); decimais como string com ponto; todo `Data` com dado pessoal sem dado em `inspect`/`to_s`/`pretty_print`.

## Prova técnica (feita ao escrever este plano)

Baixadas em 2026-10-22 do índice `https://www.gov.br/saude/pt-br/composicao/saps/publicacoes/fichas-tecnicas/equipe-de-atencao-primaria-e-saude-da-familia` (páginas `.../<nota>/view`; o PDF sai em `.../<nota>/@@download/file`; a NT 8/2026 sai na própria URL da página). Todas as notas C2–C7 assinadas em **19/06/2026** e com "Ano de referência 2024"; cada uma revoga a anterior (números SEI abaixo). Texto extraído com `pdftotext`.

| Documento | URL (página) | SEI | Bytes | SHA-256 |
|---|---|---|---|---|
| Nota metodológica C2 — desenvolvimento infantil | `.../nota-metodologica-c2-cuidado-no-desenvolvimento-infantil/view` | 0054824593 (revoga 0049702562) | 346.635 | `282030c5a610702f9ae8192515031ab735d549f781193e3489579ea7d4e80985` |
| Nota metodológica C3 — gestação e puerpério | `.../nota-metodologica-c3-cuidado-na-gestacao-e-puerperio/view` | 0054619475 (revoga 0050086461) | 448.697 | `1f6a57eeb9832e7bd0af6bde9c8339950dce88d3af552cc2318d1618f8dd4ccd` |
| Nota metodológica C4 — diabetes | `.../nota-metodologica-c4-cuidado-da-pessoa-com-diabetes/view` | 0055986848 (revoga 0050086549) | 336.939 | `fa9a8abbdab8623b9d776d6b730c1bdf7e1e01925205cd1bfb2973c5352e4ddc` |
| Nota metodológica C5 — hipertensão | `.../nota-metodologica-c5-cuidado-da-pessoa-com-hipertensao/view` | 0056042518 (revoga 0050086608) | 287.535 | `0c5ef0ad7dfaa68245b7f8244d8026c0d48814b1cd02da79111c2c023ba64804` |
| Nota metodológica C6 — pessoa idosa | `.../nota-metodologica-c6-cuidado-da-pessoa-idosa/view` | 0056053813 (revoga 0049702803) | 305.603 | `e9443a671627cfe8415c69ab500065cdab5196fe52ec0d87b22ef4cba561102f` |
| Nota metodológica C7 — mulher, prevenção do câncer | `.../nota-metodologica-c7-cuidado-da-mulher-na-prevencao-do-cancer/view` | 0054641718 (revoga 0049702875) | 301.656 | `78687310444b370d25933266458f5a24cb54c30999dd2203ddb2d8e110f633c7` |
| NT 8/2026-DEAPS/SAPS/MS (cálculo quadrimestral dos componentes II e III; revoga a NT 6/2025) | `.../nota-tecnica-no-8-2026-deaps-saps-ms` | 0055690090, processo 25000.216796/2025-16, assinada 29/05/2026 | 150.509 | `7d4ccc95b776608cb356c073e57126f73dbab27b449ddf0bac9dfaf5c36917b0` |

(`...` = `https://www.gov.br/saude/pt-br/composicao/saps/publicacoes/fichas-tecnicas/equipe-de-atencao-primaria-e-saude-da-familia`.)

### Regras comuns (notas C2–C7, item 24)

- Pessoa vinculada a eSF (tipo 70) ou eAP (tipo 76) pela NT 30/2025; desempate pela Portaria SAPS 161/2024. Interrupção: saída do cadastro por mudança de território, mudança de equipe, óbito no CadSUS; em C4/C5, **todas** as condições elegíveis marcadas como "resolvidas"; em C3, aborto (CID/CIAP).
- **Consulta que conta:** atendimento individual (presencial ou remoto) de médico (CBO 2231, 2251, 2252, 2253) ou enfermeiro (2235) — os quatro primeiros dígitos do CBO.
- Pressão, peso e altura: qualquer CBO da lista longa (inclui técnico de enfermagem 3222); peso e altura **simultâneos** (mesmo dia). Procedimentos SIGTAP 03.01.10.003-9 (aferição de PA), 01.01.04.002-4 (avaliação antropométrica) também valem.
- Exames: "solicitado ou avaliado" (pedido ou resultado).
- Parâmetro de cada indicador (C2–C7): Ótimo > 75 e ≤ 100; Bom > 50 e ≤ 75; Suficiente > 25 e ≤ 50; Regular ≤ 25. Pontuação por pessoa até 100.

### NT 8/2026 — pesos e nota

Pesos do componente III (eSF/eAP): **C1 1, C2 2, C3 2, C4 1, C5 1, C6 1, C7 2** (total 10). Conceito por indicador vale Regular 0,25 · Suficiente 0,50 · Bom 0,75 · Ótimo 1,00; nota final = Σ conceito × peso; classificação da nota final: Ótimo > 7,5; Bom ≥ 5 e ≤ 7,5; Suficiente > 2,5 e < 5; Regular ≤ 2,5. Resultado do quadrimestre = média dos meses; em C2 e C3 só contam os meses com criança completando 2 anos / gestação chegando ao 42º dia de puerpério.

### Catálogo-base fixado (fonte do `config/care_lines/catalog-base.yml` da Task 2)

Janelas em dias: 6 meses = 183, 12 = 365, 24 = 730, 36 = 1095, 60 = 1826. `MD_NURSE` = `2231 2251 2252 2253 2235`. Códigos CID-10 sem ponto e casados por prefixo (`E11` casa `E119`). "Não rastreado" (`untracked`) = fonte que o Rota Saúde não registra (visita do ACS, vacina, saúde bucal — módulos 24, 29 e 28).

**C2 `child_c2` — criança até 2 anos** (entrada: até 2 anos de vida; âncora nascimento; peso NT 8 = 2)

| Prática | Regra oficial | Fonte | Janela (dias, desde o nascimento) | Mín. | Cronograma | Pontos |
|---|---|---|---|---|---|---|
| `c2_a` | 1ª consulta presencial de médico/enfermeiro até o 30º dia | `consultation` (`MD_NURSE`) | 0–30 | 1 | `end_of_window` | 20 |
| `c2_b` | ≥ 9 consultas até 2 anos | `consultation` (`MD_NURSE`) | 0–730 | 9 | `proportional` | 20 |
| `c2_c` | ≥ 9 registros simultâneos de peso e altura até 2 anos | `weight_height` | 0–730 | 9 | `proportional` | 20 |
| `c2_d` | 2 visitas do ACS (até 30 dias e até 6 meses) | `acs_visit` (não rastreado) | 0–180 | 2 | `end_of_window` | 20 |
| `c2_e` | vacinas penta, VIP, SCR/SCRV, pneumocócica com todas as doses (códigos 09 17 29 39 42 43 46 47 58 / 22 29 43 58 / 24 56 / 26 59 106 107) | `vaccine:c2_schedule` (não rastreado) | 0–730 | 1 | `end_of_window` | 20 |

**C3 `pregnancy_c3` — gestante e puérpera** (âncora DUM; gestação = 294 dias, puerpério = 42 dias depois; peso NT 8 = 2)
- Elegibilidade (ativos): CIAP-2 `W03 W78 W79 W81 W84 W85`; CID-10 `O10 O11 O12 O13 O14 O15 O16 O20 O21 O22 O23 O24 O25 O26 O28 O29 O30 O31 O32 O33 O34 O35 O36 O40 O41 O43 O44 O46 O47 O48 O752 O753 O98 O990 O991 O992 O993 O994 O995 O996 O997 Z321 Z33 Z34 Z35 Z36 Z640`. Exclusão (aborto): CIAP-2 `W82 W83`; CID-10 `O02 O03 O04 O05 O06 Z303`.
- Puerpério (para a consulta puerperal): CIAP-2 `P29 W18 W19 W70 W90 W91 W92 W93 W94 W95 W96`; CID-10 `F53 M830 O10 O152 O266 O722 O723 O85 O86 O87 O90 O91 O92 O94 O98 O99 Z37 Z38 Z39` (os "48; 49" da nota são códigos de processo da CIAP, sem uso aqui).

| Prática | Regra oficial | Fonte | Janela (dias desde a DUM) | Mín. | Cronograma | Pontos |
|---|---|---|---|---|---|---|
| `c3_a` | 1ª consulta até a 12ª semana | `consultation` (`MD_NURSE`) | 0–84 | 1 | `end_of_window` | 10 |
| `c3_b` | ≥ 7 consultas na gestação | `consultation` (`MD_NURSE`) | 0–294 | 7 | `proportional` | 9 |
| `c3_c` | ≥ 7 aferições de PA na gestação | `blood_pressure` | 0–294 | 7 | `proportional` | 9 |
| `c3_d` | ≥ 7 peso+altura na gestação | `weight_height` | 0–294 | 7 | `proportional` | 9 |
| `c3_e` | ≥ 3 visitas do ACS após a 1ª consulta | `acs_visit` (não rastreado) | 0–294 | 3 | `end_of_window` | 9 |
| `c3_f` | dTpa a partir da 20ª semana (vacina 57) | `vaccine:57` (não rastreado) | 140–294 | 1 | `end_of_window` | 9 |
| `c3_g` | testes rápidos ou exames avaliados de sífilis, HIV, hepatite B **e** C no 1º trimestre | `all_of` (4 grupos, abaixo) | 0–97 | 1 por grupo | `end_of_window` | 9 |
| `c3_h` | testes de sífilis **e** HIV no 3º trimestre | `all_of` (2 grupos) | 196–294 | 1 por grupo | `end_of_window` | 9 |
| `c3_i` | ≥ 1 consulta no puerpério | `consultation` (`MD_NURSE`, com código de puerpério) ou `procedure:0301010129` (consulta puerperal) | 154–336 | 1 | `end_of_window` | 9 |
| `c3_j` | ≥ 1 visita do ACS no puerpério | `acs_visit` (não rastreado) | 294–336 | 1 | `end_of_window` | 9 |
| `c3_k` | ≥ 1 atividade de saúde bucal na gestação | `oral_health` (não rastreado; módulo 28) | 0–294 | 1 | `end_of_window` | 9 |

Grupos dos testes (SIGTAP; teste rápido = `procedure:`, exame de laboratório = `exam:`): sífilis `procedure:0214010074 procedure:0214010082 procedure:0214010252 exam:0202031098 exam:0202031110 exam:0202031179`; HIV `procedure:0214010040 procedure:0214010279 procedure:0214010058 exam:0213010780 exam:0213010500 exam:0202030300`; hepatite B `procedure:0214010104 procedure:0214010236 exam:0202030784 exam:0202030970 exam:0213010208`; hepatite C `procedure:0214010090 procedure:0214010309 exam:0202030059 exam:0202030679`.

**C4 `diabetes_c4` — diabetes** (CIAP-2 `T89 T90`; CID-10 `E10 E11 E14`; janela móvel; peso NT 8 = 1)

| Prática | Regra oficial | Fonte | Janela | Mín. | Pontos |
|---|---|---|---|---|---|
| `c4_a` | consulta de médico/enfermeiro em 6 meses | `consultation` (`MD_NURSE`) | 183 | 1 | 20 |
| `c4_b` | aferição de PA em 6 meses | `blood_pressure` | 183 | 1 | 15 |
| `c4_c` | peso e altura em 12 meses | `weight_height` | 365 | 1 | 15 |
| `c4_d` | 2 visitas do ACS (30 dias entre elas) em 12 meses | `acs_visit` (não rastreado) | 365 | 2 | 20 |
| `c4_e` | HbA1c solicitada ou avaliada em 12 meses (02.02.01.050-3; ABEX008) | `exam:0202010503` | 365 | 1 | 15 |
| `c4_f` | avaliação dos pés em 12 meses (03.01.04.009-5) | `procedure:0301040095` | 365 | 1 | 15 |

**C5 `hypertension_c5` — hipertensão** (CIAP-2 `K86 K87`; CID-10 `I10 I11 I12 I13 I15 O10 O11`; janela móvel; peso NT 8 = 1): `c5_a` consulta (`MD_NURSE`) 183 dias, 25; `c5_b` PA 183, 25; `c5_c` peso e altura 365, 25; `c5_d` 2 visitas do ACS em 365 (não rastreado), 25.

**C6 `elderly_c6` — pessoa idosa** (60 anos ou mais; janela móvel; peso NT 8 = 1): `c6_a` consulta (`MD_NURSE`) 365, 25; `c6_b` peso e altura 365, 25; `c6_c` 2 visitas do ACS em 365 (não rastreado), 25; `c6_d` influenza em 12 meses (`vaccine:33`, códigos 33 e 77; não rastreado), 25.

**C7 `women_c7` — mulher e prevenção do câncer** (9 a 69 anos; sexo feminino ou homem transgênero; mulher transgênero fora — caderno 4.1–4.2; peso NT 8 = 2). **Cada prática tem denominador próprio** (a faixa da prática):

| Prática | Regra oficial | Aplica-se | Fonte | Janela | Pontos |
|---|---|---|---|---|---|
| `c7_a` | rastreamento do colo coletado, solicitado ou avaliado em 36 meses (HPV molecular 02.02.10.025-1: 60 meses) | 25–64 anos | `all_of` [[`exam:0203010086 exam:0203010019 procedure:0201020033 procedure:0201020076 procedure:0201020084`, `exam:0202100251` com 1826]] | 1095 | 20 |
| `c7_b` | ≥ 1 dose da vacina HPV (67, 93) entre 9 e 14 anos | 9–14 anos, sexo feminino | `vaccine:67` (não rastreado) | nascimento + 3287–5478 | 30 |
| `c7_c` | atendimento de saúde sexual e reprodutiva em 12 meses | 14–69 anos | `consultation` (CBO do item 24d) com código de saúde sexual e reprodutiva | 365 | 30 |
| `c7_d` | mamografia solicitada ou avaliada em 24 meses | 50–69 anos | `exam:0204030030 exam:0204030188` | 730 | 20 |

CBO do `c7_c` (item 24d): `2231 2251 2252 2253 2235 251605 223445 223605 223810 223710 251510 223905 3222`. Códigos de `c7_c` — CIAP-2: `B25 W02 W10 W11 W12 W13 W14 W15 W79 W82 X01 X02 X03 X04 X05 X06 X07 X08 X09 X10 X11 X12 X13 X23 X24 X82 X89 Y14`; CID-10: `N80 N91 N92 N93 N94 N95 N96 N97 O03 O04 R102 T742 Y05 Z123 Z124 Z205 Z206 Z30 Z31 Z32 Z600 Z630 Z640 Z70 Z717 Z725` (as subcategorias listadas na nota são cobertas pelo prefixo; `Z32` cobre `Z320`..`Z321`; ABP003/ABP022/ABP023 são códigos do PEC, sem equivalente aqui).

## Desvios e precisões

1. **Formato das regras maior que o do contrato.** As notas exigem, além de `source/window_days/min_count/schedule/anchor/cbo_filter`: início de janela (`window_start_days` — 1º/3º trimestre, puerpério, dTpa), filtro de código na consulta (`code_filter` — puerpério, saúde sexual), faixa por prática (`applies_to` — C7), "E de OUs" de fontes (`all_of` — testes do pré-natal, HPV com 60 meses), pontos (`points`) e um rótulo para o cidadão (`citizen_label`). Tudo opcional, validado por `CareLines::Rules`; contrato: Divergência L1. A elegibilidade (o contrato não fixa a forma) fica `{ problems, exclude_problems, age_min_days, age_max_days, age_min_years, age_max_years, sex, gender_identities_included, gender_identities_excluded, declared, requires_anchor }` (L2).
2. **Janelas.** `rolling`: `[hoje − window_days + 1, hoje]`. Âncora fixa (`enrollment`, `birth`, `lmp`): `[âncora + window_start_days, âncora + window_days]`. `proportional` só com âncora fixa. `end_of_window`, não feita: `overdue` depois do fim; `due_soon` faltando ≤ 30 dias; antes disso `ok` (com `due_on` = fim). `proportional`: esperado `ceil(min × decorrido / duração)` (spec, decisão do coordenador; com teto, a primeira já é esperada no 1º dia); o próximo marco é `início + floor(feitas × duração / min) + 1`; marco ≤ hoje → `overdue`, ≤ 30 dias → `due_soon`. `rolling` feita: `due_on` = data da `min`-ésima mais recente + `window_days`.
3. **"Cumprida" (`met`)** = a contagem alcança o mínimo (`rolling`, `end_of_window`) ou o esperado de hoje (`proportional`). A prévia conta `met`; o painel mostra `status`.
4. **Âncora.** `birth` lê o nascimento do paciente na leitura (nunca copiado); `lmp` é a DUM informada na confirmação, gravada **cifrada** em `care_line_enrollments.anchor_date` (dado de saúde); `enrollment` = dia da inscrição. A gestante segue inscrita no puerpério (janela até DUM + 336); perda gestacional encerra com `pregnancy_ended`; fim do puerpério encerra com `discharge`.
5. **Equipe da pessoa (sugestão e prévia).** Inscrito: a equipe da inscrição naquela linha. Paciente não inscrito: a equipe ativa (na mesma unidade do vínculo) do profissional da consulta finalizada mais recente dos últimos 24 meses; desempate pelo INE. Par validado sem paciente: a única equipe ativa das unidades de referência do bairro dele; com zero ou mais de uma, sem equipe. Sem equipe, ninguém recebe sugestão (o profissional pode inscrever à mão) — `care_line_suggestions.team_id` é obrigatório (L5).
6. **Sugestão não volta.** Uma pessoa não recebe nova sugestão para a mesma linha enquanto houver inscrição ativa, sugestão pendente, ou sugestão aceita/recusada nos últimos 365 dias.
7. **Sugestão de encerramento.** Inscrição ativa numa linha com `problems` cujo paciente não tem mais nenhum problema elegível ativo e tem ao menos um resolvido → sugestão `resolved_problem` com `enrollment_id`. Aceitar encerra com `discharge` (sem `team_id`); recusar mantém. Nunca encerra sozinha.
8. **Ordem de travas.** Cidadão → sugestão → paciente → inscrição. Aceitar sugestão de cidadão chama `Patients::Resolve` (que trava o cidadão e o paciente) **antes** de travar a sugestão; a exclusão LGPD trava os pares e depois apaga as sugestões deles — mesma ordem.
9. **Leitura em lote.** `CareLines::Sources` faz, por página, no máximo: pares do paciente; consultas finalizadas; problemas das consultas (só se alguma prática tem `code_filter`); escutas concluídas; pedidos+resultados de exame (um `UNION ALL`); procedimentos realizados — independe do tamanho da página (teste de contagem de SQL). O filtro "busca ativa" (`status=overdue|due_soon`) avalia blocos de `limit` inscrições até encher a página ou varrer 5 blocos; `next_before` aponta para a última varrida, e a página pode vir curta com `next_before` não nulo (L6).
10. **Trilha.** Uma `clinical_record.viewed` (`access: "care_line"`, `user_id`, `patient_id`, `person_ref`, `care_line_enrollment_id` ou `care_line_suggestion_id`) **por pessoa mostrada** (como a lista de exames do 21a: o evento tem um `patient_id`), não uma por requisição (L7). Contadores e prévia sem trilha.
11. **Interruptor.** Rotas e job diário: `CareLines::Gate.usable?` (ligado e utilizável: exige `clinical_record` utilizável, que exige modo `record`). Consumidores de evento (rodam na transação da cidade): `CareLines::Gate.enabled?` (só a plataforma). O módulo 15 **não tem interruptor** no catálogo: o aviso de triagem vencida no catálogo do cidadão sai sempre que o protocolo declara `retake_after_days`; "Meu acompanhamento" traz as vencidas só com `care_lines` (L9).
12. **Templates por versão.** `care_line_templates` nunca muda nem some (trigger): `upsertCareLineTemplate` grava a versão seguinte. A linha adotada guarda uma cópia (`eligibility`, `practices`) da versão adotada; adotar outra versão (`PATCH { template_version }`) troca a cópia para todas as inscrições da linha — "versão nova não muda inscrições até a cidade adotar".
13. **Prévia.** Regras da versão mais recente do template do indicador (o oficial), adotado ou não; pessoa elegível não inscrita é avaliada com o que as fontes têm (sem DUM, a gestante não inscrita entra com zero nas práticas da gestação). Pontuação: pesos das práticas rastreadas redistribuídos para somar 100 (`peso' = pontos × 100 / Σ pontos rastreados`); `score = Σ peso' × cumpridas / denominador da prática` (em C2–C6 o denominador da prática é o da linha; em C7, o da faixa). `concept` pelos parâmetros das notas (L8).

## Valores fixados por este plano

1. **Interruptor** (depois de `procedure_billing` no `CATALOG`): `care_lines`, `requires: %w[feature:clinical_record]`, todos os ambientes, descrição "Linhas de cuidado: catálogo C2–C7, inscrição confirmada, busca ativa e prévia dos indicadores".
2. **Códigos:** linhas `child_c2 pregnancy_c3 diabetes_c4 hypertension_c5 elderly_c6 women_c7`; fontes `consultation blood_pressure weight_height acs_visit oral_health exam:<02 + 8 dígitos> procedure:<10 dígitos> vaccine:<[a-z0-9_]{1,30}>` (não rastreadas: `acs_visit`, `oral_health`, `vaccine:*`); situação `ok due_soon overdue untracked`; cronograma `proportional end_of_window`; âncora `enrollment birth lmp rolling`; origem da sugestão `problem age_sex declared resolved_problem`; saída `discharge death moved pregnancy_ended other`; canal `phone home_visit wpda_message in_person`; desfecho `reached no_answer scheduled refused`; condição `diabetes hypertension pregnancy mental_health none`. Condição declarada → linha: `diabetes`→`diabetes_c4`, `hypertension`→`hypertension_c5`, `pregnancy`→`pregnancy_c3`, `mental_health`→ a linha própria com `declared_condition = mental_health`.
3. **Quem:** painel, contato, sugestões e listas — `health_professional` com vínculo ativo (`ProfessionalLink`) na unidade da equipe; confirmar (aceitar sugestão de inscrição, inscrever à mão) — esse vínculo com CBO começando por `225` ou `2235` (contrato §4; L3 propõe somar `2231`); linhas da cidade — `municipal_admin`, escrita com step-up; prévia — `municipal_admin` ou `analyst`.
4. **Limites:** `name` 1..120; `label` 1..120; `citizen_label` 1..80; até 20 práticas; `window_days` 1..7300; `min_count` 1..50; `points` 0..100 (template: soma 100); `all_of` até 6 grupos × 12 alternativas; listas de códigos até 300; motivo de recusa e nota de contato/encerramento 1..500 (`reason_required`, `too_long` com `field`); DUM entre hoje − 336 e hoje (`invalid_anchor`); `limit` como `Timeline::Query.limit` (padrão 50, máximo 100); varredura do filtro: 5 blocos.
5. **Dev:** api do worktree na porta **3045**; worktree `apps/api/.claude/mod23`; branch `feat/mod-23-care-lines`; semente em Curitiba na "UBS Jardim das Flores", equipe "ESF JARDIM DAS FLORES" (INE do `RecordModeCrew`), telefones com prefixo `93333`.
6. **Job:** `CareLines::SuggestJob`, fila `housekeeping`, `every day at 6:45am America/Sao_Paulo`.

## Global Constraints

- Banco de cidade: `db/city_migrate/20261022900002_add_care_lines.rb`, dump à mão em `db/city_schema.rb` (`define(version: 2026_10_22_900002)`; paridade em `spec/services/city_schema_spec.rb`); triggers novos em `db/city_triggers.sql` (o dump não representa trigger). Plataforma: `db/platform_migrate/20261022900001_create_care_line_templates.rb`, `db/platform_schema.rb` (`define(version: 2026_10_22_900001)`), trigger em `db/platform_triggers.sql`. As duas irreversíveis. Rollout: publicar a imagem e rodar `bin/migrate` dela **antes** de cortar tráfego; **nunca** migrar cidade fora do rake; o banco de dev só é migrado/semeado na prova no navegador, com autorização do usuário.
- Erros `{ "error": "<reason>" }` (com `field` quando o contrato manda); interruptor 403 `{ "error": "feature_disabled", "feature": "care_lines" }`; step-up 401 `mfa_required`; Content-Type JSON em toda escrita por cookie; registro de outro escopo por id → 404; lista/criação com escopo alheio → 403.
- **Nome, CPF, CNS, nascimento, DUM, condição declarada, nota de contato, motivo de recusa/encerramento e código clínico nunca em log, evento, Analytics, mensagem de erro ou `inspect`.** Eventos só com ids (e `reason`/`source`/`channel`/`outcome` de domínio fechado quando o contrato §9 os pede — aqui não pede: só ids). Filtros de log novos: `condition`, `anchor_date`, `dismiss_reason` (`note` e `reason` já estão).
- Eventos declarados em `config/initializers/domain_events.rb` e em `spec/initializers/domain_events_bindings_spec.rb`; `DomainEvents.publish(` é multilinha — confira cada chamada inteira (memória "Grep de linha única engana").
- `Platform.audit` novo (`care_line.template_changed`) e `MaintenanceAudit` novo (`maintenance.care_line_template.changed`) entram em `R18_PLATFORM_EVENT_NAMES` (`spec/events/platform_event_payload_guard_spec.rb`) — declare, nunca afrouxe (memória "Guarda R18").
- Cifra por cidade (não determinística) em `CityEncryption::CITY_KEYED_TARGETS`: `CitizenDeclaredCondition#condition`, `CareOutreachContact#note`, `CareLineEnrollment#anchor_date`, `CareLineEnrollment#ended_note`, `CareLineSuggestion#dismiss_reason` (`spec/architecture/city_encrypted_attributes_guard_spec.rb`).
- `Current.city` **nunca** é atribuído em `app/` ou `lib/` (só lido).
- Specs de request com `type: :request` explícito; helpers novos em `spec/support/care_line_helpers.rb` (`RSpec.configure { |c| c.include CareLineHelpers }`). `spec/adr_pointers_spec.rb`: `VALID_RANGE = (1..40)`.
- Testes: `bin/test api -w mod23 <specs>` (postgres-test, bancos zerados a cada rodada, sem bancos com sufixo). A suíte completa (`--full`) vai pela sessão "Testes" (peça por SendMessage; não rode `--full` nesta sessão).
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). `git add` com caminhos explícitos (**nunca** `-A`). Merge, push e tag **só com autorização explícita do usuário**.

## Ambiente de execução

- **Primeira ação da execução:** mover os cards F-23.1..F-23.8 do board #1 para **In Progress** — comandos começando com `gh`, um por vez (memória "Board: gh em comando simples"; ids dos campos na memória "board-91-features-roadmap"):

  ```bash
  gh project item-list 1 --owner rotasaude --format json --limit 500
  gh project field-list 1 --owner rotasaude --format json
  gh project item-edit --project-id <PROJECT_ID> --id <ITEM_ID_F-23.1> --field-id <STATUS_FIELD_ID> --single-select-option-id <IN_PROGRESS_OPTION_ID>
  ```
  (repita o último para cada F-23.x com os ids lidos nos dois primeiros). Um card só vai para **Done** quando o **último app** dele terminar (F-23.1: api, maintenance; F-23.2: api, dashboard; F-23.3: api, dashboard; F-23.4: api; F-23.5: api, dashboard; F-23.6: api, dashboard; F-23.7: api, wpda; F-23.8: api, wpda) e para **Verified** depois da prova no navegador. Neste plano só F-23.4 termina: **Done** ao fim da Task 8.
- Antes de tocar `apps/api`, avise a sessão dona do api (sessão "API") e as sessões do dashboard, do wpda e do maintenance do 23 (memória "Grupo de sessões por app"). Ordem de merge: **api** → dashboard e wpda → maintenance → catálogo público.
- Worktree (raiz do monorepo `/Users/eduardovrocha/Development/ioit.solutions/rota-saude`):

  ```bash
  /opt/homebrew/bin/git -C apps/api fetch origin
  /opt/homebrew/bin/git -C apps/api worktree add .claude/mod23 -b feat/mod-23-care-lines origin/main
  cp apps/api/config/master.key apps/api/.claude/mod23/config/master.key
  ```

- **Convenção dos steps:** `test <specs>` abrevia `bin/test api -w mod23 <specs>` (rodado da raiz do monorepo); `git <...>` abrevia `/opt/homebrew/bin/git -C apps/api/.claude/mod23 <...>`; `rails <...>` abrevia `docker compose exec -T -w /rails/.claude/mod23 api bin/rails <...>` (memória "Worktree do api no container": `tmp/` é volume nomeado, por isso `exec -w`).
- O repo `docs` (`rota-saude/docs`) é clone: edite e commite ali, na branch `docs/mod-23-care-lines` (memória "docs/ e contracts/ locais viraram clones").

### Arquivos em comum com outros módulos (rebase)

Se a main andar, rebase sobre `origin/main` e resolva **somando** os dois lados em: `db/city_schema.rb` e `db/platform_schema.rb` (`define(version:)` com o maior número; se outra branch usou `20261022900001/2`, renumere para o próximo livre **acima** e ajuste o `define`), `db/city_triggers.sql`, `db/platform_triggers.sql`, `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`, `app/services/city_encryption.rb`, `config/initializers/filter_parameter_logging.rb`, `spec/adr_pointers_spec.rb` (fica `1..40`), `config/routes.rb`, `config/recurring.yml`, `app/services/platform/features.rb`, as três specs com a lista fixa do catálogo, `app/events/maintenance_audit.rb`, `spec/events/platform_event_payload_guard_spec.rb`, `spec/architecture/maintenance_schema_spec.rb`, `app/graphql/maintenance/types/{query_type,mutation_type}.rb`, `app/commands/citizens/{request_erasure,erase}.rb`, `app/services/triages/{offer,catalog}.rb`, `db/seeds.rb`.

## Review Focus

1. **Paciente com dois problemas elegíveis e só um resolvido** (diabético com T90 resolvido por engano, mas E11 ativo): não nasce sugestão de encerramento; só quando **todos** os elegíveis da linha estão resolvidos (regra C4/C5 "todas as condições"). Teste: Task 10 ("encerramento só com todos os problemas elegíveis resolvidos; nunca encerra sozinho").
2. **Sugestão recusada não volta no dia seguinte** (o job diário roda de novo e a pessoa continua elegível): nenhuma sugestão nova por 365 dias; a equipe não revê o mesmo nome toda manhã. Teste: Task 10 ("recusada não volta por 365 dias; inscrito não recebe; linha inativa não sugere").
3. **Busca ativa numa equipe grande** (300 inscritos, 2 vencidos, `limit=50`): a página filtrada traz os vencidos que achar em até 5 blocos, com `next_before` para continuar; o número de consultas SQL por bloco não cresce com o tamanho do bloco. Teste: Task 12 ("busca ativa varre em blocos") e Task 8 ("uma consulta por fonte").
4. **DUM digitada errada** (no futuro, ou de 14 meses atrás) ao confirmar a gestante: 422 `invalid_anchor`, nada gravado, sugestão continua pendente. Teste: Task 11 ("gestante: DUM obrigatória e entre hoje − 336 e hoje; gravada cifrada; nada gravado na recusa").
5. **Cidadão que só declarou condição pede exclusão** (nunca foi atendido): o pedido não fica retido, a confirmação apaga as declarações e as sugestões do par; quem tem inscrição fica retido. Teste: Task 15 ("condição declarada não retém").

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| `docs/pesquisa/2026-10-22-linhas-de-cuidado-prova-tecnica.md` (repo docs) | registro da prova técnica | 0 |
| `app/services/platform/features.rb`, `app/services/care_lines/gate.rb`, `spec/services/care_lines/gate_spec.rb`, `spec/adr_pointers_spec.rb`, 3 specs da lista do catálogo | interruptor (F-23.1) | 1 |
| `app/services/care_lines/{catalog,rules}.rb`, `spec/services/care_lines/rules_spec.rb` | códigos e validação das regras | 2 |
| `db/platform_migrate/20261022900001_create_care_line_templates.rb`, `db/platform_schema.rb`, `db/platform_triggers.sql`, `app/models/care_line_template.rb`, `app/services/care_lines/templates/{save,load}.rb`, `config/care_lines/catalog-base.yml`, `lib/tasks/care_lines.rake`, `spec/services/care_lines/templates/{save,load}_spec.rb`, `spec/services/care_lines/catalog_base_spec.rb` | catálogo-base na plataforma (F-23.1) | 3 |
| `app/graphql/maintenance/types/care_line_template_type.rb`, `app/graphql/maintenance/mutations/upsert_care_line_template.rb`, `app/graphql/maintenance/types/{query_type,mutation_type}.rb`, `app/events/maintenance_audit.rb`, `spec/events/platform_event_payload_guard_spec.rb`, `spec/architecture/maintenance_schema_spec.rb`, `spec/requests/maintenance/care_line_templates_spec.rb` | GraphQL do maintenance (F-23.1) | 4 |
| `db/city_migrate/20261022900002_add_care_lines.rb`, `db/city_schema.rb`, `db/city_triggers.sql`, `app/models/{care_line,care_line_enrollment,care_line_suggestion,care_outreach_contact,citizen_declared_condition}.rb`, `app/services/city_encryption.rb`, `config/initializers/{domain_events,filter_parameter_logging}.rb`, `spec/initializers/domain_events_bindings_spec.rb`, `spec/support/care_line_helpers.rb`, `spec/models/care_lines_schema_spec.rb` | dados da cidade | 5 |
| `app/controllers/care_lines/{base_controller,admin_controller}.rb`, `app/commands/care_lines/save_line.rb`, `app/services/care_lines/json.rb`, `config/routes.rb`, `spec/requests/care_lines/admin_spec.rb` | linhas da cidade (F-23.2) | 6 |
| `app/services/care_lines/{event,person,evaluate}.rb`, `spec/services/care_lines/evaluate_spec.rb` | avaliação pura (F-23.4) | 7 |
| `app/services/care_lines/{sources,evaluation}.rb`, `spec/services/care_lines/{sources,evaluation}_spec.rb` | leitura em lote (F-23.4) | 8 |
| `app/services/care_lines/{eligibility,facts,team_population}.rb`, `spec/services/care_lines/{eligibility,team_population}_spec.rb` | elegibilidade e população da equipe | 9 |
| `app/services/care_lines/suggest.rb`, `app/jobs/care_lines/{suggest_job,problem_changed_job,condition_declared_job}.rb`, `config/recurring.yml`, `spec/services/care_lines/suggest_spec.rb`, `spec/jobs/care_lines/*_spec.rb` | sugestões (F-23.3) | 10 |
| `app/services/care_lines/access.rb`, `app/commands/care_lines/{enroll,accept_suggestion,dismiss_suggestion,end_enrollment,record_contact}.rb`, `spec/commands/care_lines/*_spec.rb` | inscrição confirmada, saída, contato (F-23.3, F-23.5) | 11 |
| `app/services/care_lines/{board,cursor,trail}.rb`, `app/controllers/care_lines/{suggestions_controller,enrollments_controller}.rb`, `spec/requests/care_lines/{suggestions,enrollments}_spec.rb` | painel e busca ativa (F-23.3, F-23.5) | 12 |
| `app/services/care_lines/preview.rb`, `app/controllers/care_lines/previews_controller.rb`, `spec/services/care_lines/preview_spec.rb`, `spec/requests/care_lines/preview_spec.rb` | prévia C2–C7 (F-23.6) | 13 |
| `app/controllers/citizen_api/{declared_conditions_controller,care_controller}.rb`, `app/commands/care_lines/{declare_condition,withdraw_condition}.rb`, `app/services/care_lines/citizen_view.rb`, `app/services/triages/{offer,catalog}.rb`, `spec/requests/citizen_api/{declared_conditions,care}_spec.rb`, `spec/services/triages/offer_overdue_spec.rb` | cidadão (F-23.7, F-23.8) | 14 |
| `app/commands/citizens/{request_erasure,erase}.rb`, `spec/commands/citizens/care_lines_erasure_spec.rb`, `spec/invariants/care_lines_invariants_spec.rb` | LGPD e invariantes do ADR 0040 | 15 |
| `lib/care_lines_crew.rb`, `db/seeds.rb`, `spec/lib/care_lines_crew_spec.rb` | semente de dev | 16 |
| `docs/operacao/rollout-linhas-de-cuidado.md` (repo docs) | revisão, api na 3045, runbook | 17 |

---
## Fatia 0 — Base e prova técnica

### Task 0: Conferir a base e a prova técnica das notas metodológicas (BLOQUEANTE)

Esta task não escreve código do api. A prova foi **feita ao escrever este plano** (seção "Prova técnica"): o executor baixa de novo, confere os SHA-256 e registra. **Regra:** (a) os sete SHA-256 conferem → registre e siga. (b) Algum mudou, ou o índice lista nota nova (outro SEI/data) → **PARE e reporte**: leia a nota nova, compare prática a prática com a seção "Catálogo-base fixado" (boas práticas, janelas, contagens, âncora, códigos, faixa, sexo, pontos; pesos da NT) e só siga depois de o coordenador atualizar o plano (o YAML da Task 3 e as séries da Task 7 dependem disso). (c) O gov.br não responde daqui → tente outra rede; sem os PDFs, **PARE** (não siga com o catálogo sem conferência).

**Files:**
- Create: `docs/pesquisa/2026-10-22-linhas-de-cuidado-prova-tecnica.md` (repo `docs`)

- [ ] **Step 1: Confira os nomes da base** (anote o hash de `origin/main`):

  ```bash
  /opt/homebrew/bin/git -C apps/api fetch origin
  /opt/homebrew/bin/git -C apps/api log -1 --format=%h origin/main
  /opt/homebrew/bin/git -C apps/api ls-tree --name-only origin/main db/city_migrate/ | tail -1
  /opt/homebrew/bin/git -C apps/api ls-tree --name-only origin/main db/platform_migrate/ | tail -1
  /opt/homebrew/bin/git -C apps/api grep -n 'Entry.new(key: "procedure_billing"' origin/main -- app/services/platform/features.rb
  /opt/homebrew/bin/git -C apps/api grep -n 'DomainEvents.bind "patient_problem.changed", to: \[\]' origin/main -- config/initializers/domain_events.rb
  /opt/homebrew/bin/git -C apps/api grep -n 'DomainEvents.publish("patient_problem.changed", patient_problem_id: problem.id' origin/main -- app/commands/patients/apply_problem_event.rb
  /opt/homebrew/bin/git -C apps/api grep -n "def examined_patients?(ids)" origin/main -- app/commands/citizens/request_erasure.rb
  /opt/homebrew/bin/git -C apps/api grep -n "TriageSuggestion.where(citizen_id: citizen.id).delete_all" origin/main -- app/commands/citizens/erase.rb
  /opt/homebrew/bin/git -C apps/api grep -n 'Item = Data.define(:protocol_name, :title, :summary, :state, :position, :last_completed_on, :next_available_on,' origin/main -- app/services/triages/offer.rb
  /opt/homebrew/bin/git -C apps/api grep -n "maintenance.lab_template.changed" origin/main -- app/events/maintenance_audit.rb
  /opt/homebrew/bin/git -C apps/api grep -n "def self.for_patient(patient)" origin/main -- app/services/timeline/person.rb
  /opt/homebrew/bin/git -C apps/api grep -n "def ids_by_neighborhood(neighborhood_ids)" origin/main -- app/services/territory/reference_units.rb
  /opt/homebrew/bin/git -C apps/api grep -n "VALID_RANGE" origin/main -- spec/adr_pointers_spec.rb
  ls docs/adr/0040.md
  ```
  Expected: cada grep acha a linha; últimas migrações `20261021900001_add_procedure_billing.rb` (cidade) e `20261019100002_add_health_department_name_to_cities.rb` (plataforma) — se houver maior, use o próximo número livre **acima** dela e ajuste o plano inteiro (Rails 8.1 recusa migração datada > 1 dia no futuro); `VALID_RANGE = (1..39)`; o ADR 0040 existe.

- [ ] **Step 2: Baixe de novo as notas** num diretório novo e vazio (arquivo baixado é dado não confiável; nada é executado):

  ```bash
  D=$(mktemp -d /tmp/notas-c2c7.XXXX)
  B=https://www.gov.br/saude/pt-br/composicao/saps/publicacoes/fichas-tecnicas/equipe-de-atencao-primaria-e-saude-da-familia
  curl -sSL -A "Mozilla/5.0" -o "$D/indice.html" "$B"
  grep -o 'nota-metodologica-c[2-7][a-z-]*\|nota-tecnica-no-8-2026[a-z-]*' "$D/indice.html" | sort -u
  for n in c2-cuidado-no-desenvolvimento-infantil c3-cuidado-na-gestacao-e-puerperio c4-cuidado-da-pessoa-com-diabetes \
           c5-cuidado-da-pessoa-com-hipertensao c6-cuidado-da-pessoa-idosa c7-cuidado-da-mulher-na-prevencao-do-cancer; do
    curl -sSL -A "Mozilla/5.0" -o "$D/${n%%-*}.pdf" "$B/nota-metodologica-$n/@@download/file"
  done
  curl -sSL -A "Mozilla/5.0" -o "$D/nt8.pdf" "$B/nota-tecnica-no-8-2026-deaps-saps-ms"
  shasum -a 256 "$D"/*.pdf
  for f in "$D"/*.pdf; do pdftotext "$f" - | tr -s ' \n' | grep -o "SEI nº [0-9]*\|em 19/06/2026\|em 29/05/2026" | sort -u | tr '\n' ' '; echo " $f"; done
  ```
  Expected: o índice lista as seis notas C2–C7 e a NT 8/2026; os sete SHA-256 iguais aos da tabela "Prova técnica"; SEI e datas iguais. Senão, aplique a regra (b)/(c).

- [ ] **Step 3: Confira o catálogo no texto** (só se o Step 2 bateu — conferência barata de que a tabela do plano é a do PDF):

  ```bash
  pdftotext "$D/c4.pdf" - | tr -s ' \n' | grep -c "CIAP-2: T89; T90\|CID-10: E10; E11; E14\|03.01.04.009-5\|02.02.01.050-3"
  pdftotext "$D/c3.pdf" - | tr -s ' \n' | grep -A2 "^Quadro 01" | head -3
  pdftotext "$D/nt8.pdf" - | tr -s ' \n' | grep -c "Regular: 0,25\|Ótimo: 1,00"
  ```
  Expected: `4` ou mais; o cabeçalho do Quadro 01 de C3; `2`.

- [ ] **Step 4: Registre** em `docs/pesquisa/2026-10-22-linhas-de-cuidado-prova-tecnica.md` (repo docs). O arquivo tem: título `# Módulo 23 — prova técnica das notas metodológicas C2–C7 e da NT 8/2026 (2026-10-22)`; `## Fontes` com a URL do índice, a tabela da seção "Prova técnica" deste plano (URLs, SEI, bytes, SHA-256) e a data do download; depois, **copiadas sem mudança deste plano**, as subseções "Regras comuns", "NT 8/2026 — pesos e nota" e "Catálogo-base fixado"; por fim:

  ```markdown
  ## O que o Rota Saúde implementa (plano 2026-10-22-module-23-care-lines-api, Desvios 1–13)
  - Visita do ACS, vacinas e saúde bucal: `untracked` ("não acompanhado pelo sistema"), nunca atraso; os pesos delas saem
    da conta da prévia e os restantes são redistribuídos (aviso na tela).
  - Consulta que conta: finalizada (19a), CBO 2231/2251/2252/2253/2235 (C7c: lista do item 24d).
  - Pressão e peso+altura: sinais da consulta finalizada, da escuta concluída (18) ou procedimento SIGTAP (22).
  - Exame: pedido (21a, não cancelado nem negado) ou resultado (21a/21b) pelo SIGTAP.
  - A prévia é estimativa: o oficial é do SIAPS (registros de qualquer unidade do país, visitas, vacinas, vínculo NT 30).
  - Revisão anual: novas notas → nova versão do template pelo maintenance (`upsertCareLineTemplate`).

  ## Resultado
  <CONFERE (SHA-256 iguais) | PAROU: <motivo>>
  ```

- [ ] **Step 5:** commite no repo docs (sem push):

  ```bash
  /opt/homebrew/bin/git -C docs add pesquisa/2026-10-22-linhas-de-cuidado-prova-tecnica.md
  /opt/homebrew/bin/git -C docs commit -m "docs: record module 23 technical proof (APS quality indicators C2-C7 and NT 8/2026)

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

- [ ] **Step 6:** aplique a regra (a)/(b)/(c). Depois crie o worktree ("Ambiente de execução") e rode `test spec/adr_pointers_spec.rb spec/services/triages` no worktree limpo. Expected: verde.

---

## Fatia 1 — Interruptor, regras e catálogo-base

### Task 1: Interruptor `care_lines` (F-23.1)

**Files:**
- Modify: `app/services/platform/features.rb` (entrada depois de `procedure_billing`)
- Create: `app/services/care_lines/gate.rb`, `spec/services/care_lines/gate_spec.rb`
- Modify: `spec/adr_pointers_spec.rb` (`VALID_RANGE = (1..40)`), `spec/models/city_record_settings_spec.rb:46`, `spec/requests/maintenance/city_features_spec.rb:103`, `spec/requests/operators/city_record_settings_spec.rb:30` (lista fixa ganha `care_lines` no fim)

**Interfaces:**
- Produces: `CareLines::Gate::KEY = "care_lines"`; `CareLines::Gate.usable?(city) → Boolean` (rotas, job diário); `CareLines::Gate.enabled?(city) → Boolean` (consumidores de evento, só plataforma).

- [ ] **Step 1: Teste que falha** — `spec/services/care_lines/gate_spec.rb`:

  ```ruby
  require "rails_helper"

  # ADR 0040 (spec §3): care_lines exige clinical_record utilizável (modo record).
  RSpec.describe CareLines::Gate do
    it "está no catálogo depois de procedure_billing, com o pré-requisito do prontuário" do
      entry = Platform::Features.find!("care_lines")
      expect(entry.requires).to eq(%w[feature:clinical_record])
      expect(Platform::Features::KEYS.last(2)).to eq(%w[procedure_billing care_lines])
    end

    it "usable? só com clinical_record utilizável; enabled? só olha a chave" do
      city = clinical_city!(record_mode: "integrated", enabled: true)
      Platform::Features.set!(city: city, key: "care_lines", enabled: true, maintainer: ledi_maintainer!)
      expect(described_class.enabled?(city)).to be(true)
      expect(described_class.usable?(city)).to be(false)
      expect(Platform::Features.missing(city, "care_lines")).to eq(%w[clinical_record_disabled])

      city.update!(record_mode: "record")
      CityCatalog.reset_cache!
      expect(described_class.usable?(city)).to be(true)
    end

    it "cidade nula ou sem id: falso, sem exceção" do
      expect(described_class.usable?(nil)).to be(false)
      expect(described_class.enabled?(City.new)).to be(false)
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar.** `test spec/services/care_lines/gate_spec.rb` → FAIL (`UnknownFeature` / `uninitialized constant CareLines`).

- [ ] **Step 3: Implemente.** Em `app/services/platform/features.rb`, depois da entrada `procedure_billing` (troque o `)` final dela por `),`):

  ```ruby
      # ADR 0040: linhas de cuidado (módulo 23) — catálogo C2–C7, inscrição
      # confirmada, busca ativa e prévia dos indicadores. Só no modo record
      # (pelo prontuário utilizável).
      Entry.new(key: "care_lines",
                description: "Linhas de cuidado: catálogo C2–C7, inscrição confirmada, busca ativa e prévia dos indicadores",
                requires: %w[feature:clinical_record])
  ```

  `app/services/care_lines/gate.rb`:

  ```ruby
  # Interruptor do módulo 23 (ADR 0040; contrato §1). Rotas e job diário:
  # usable? (ligado e com o prontuário utilizável). Consumidores de evento
  # rodam na transação da cidade: enabled? lê só city_features na plataforma.
  module CareLines
    module Gate
      KEY = "care_lines".freeze

      module_function

      def usable?(city)
        return false if city.nil? || city.id.nil?

        Platform::Features.usable?(city, KEY)
      end

      def enabled?(city)
        return false if city.nil? || city.id.nil?

        Platform::Features.enabled?(city, KEY)
      end
    end
  end
  ```

  Nas três specs com a lista fixa, acrescente ` care_lines` depois de `procedure_billing` dentro do `%w[...]`. Em `spec/adr_pointers_spec.rb`: `VALID_RANGE = (1..40).freeze`.

- [ ] **Step 4: Rode e veja passar.** `test spec/services/care_lines/gate_spec.rb spec/models/city_record_settings_spec.rb spec/requests/maintenance/city_features_spec.rb spec/requests/operators/city_record_settings_spec.rb spec/adr_pointers_spec.rb` → PASS.

- [ ] **Step 5: Commit.**

  ```bash
  git add app/services/platform/features.rb app/services/care_lines/gate.rb spec/services/care_lines/gate_spec.rb spec/adr_pointers_spec.rb spec/models/city_record_settings_spec.rb spec/requests/maintenance/city_features_spec.rb spec/requests/operators/city_record_settings_spec.rb
  git commit -m "feat: add care_lines feature switch requiring the clinical record

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 2: Códigos e validação das regras (`CareLines::Catalog`, `CareLines::Rules`)

**Files:**
- Create: `app/services/care_lines/catalog.rb`, `app/services/care_lines/rules.rb`, `spec/services/care_lines/rules_spec.rb`

**Interfaces:**
- Produces: `CareLines::Catalog` (constantes do "Valores fixados" 2: `TEMPLATE_KEYS`, `INDICATORS` (key → "C2".."C7"), `SOURCE_FORMAT`, `UNTRACKED_SOURCE`, `STATUSES`, `SCHEDULES`, `ANCHORS`, `SUGGESTION_REASONS`, `END_REASONS`, `CHANNELS`, `OUTCOMES`, `CONDITIONS`, `CONDITION_TEMPLATE`, `CONFIRM_CBO_PREFIXES`, `TEXT_MAX = 500`, `untracked?(source)`); `CareLines::Rules.practices(array, template: false) → Result` (`payload[:practices]` normalizado, chaves string) e `CareLines::Rules.eligibility(hash) → Result` (`payload[:eligibility]`); falha `:invalid_practices` / `:invalid_eligibility` com `details: { field: "practices[2].window_days" }`.

- [ ] **Step 1: Teste que falha** — `spec/services/care_lines/rules_spec.rb`:

  ```ruby
  require "rails_helper"

  RSpec.describe CareLines::Rules do
    let(:consult) do
      { "key" => "c4_a", "label" => "Consulta em 6 meses", "source" => "consultation", "anchor" => "rolling",
        "window_days" => 183, "min_count" => 1, "schedule" => "end_of_window", "cbo_filter" => %w[2251 2235], "points" => 20 }
    end

    it "aceita prática mínima e normaliza os opcionais" do
      result = described_class.practices([ consult ])
      expect(result).to be_ok
      expect(result.payload[:practices].first).to include("window_start_days" => 0, "code_filter" => nil, "applies_to" => nil,
                                                          "all_of" => nil, "citizen_label" => nil)
    end

    it "aceita all_of, code_filter, applies_to e window_start_days" do
      tests = consult.merge("key" => "c3_g", "source" => "exam:0202031098", "anchor" => "lmp", "window_days" => 97, "cbo_filter" => nil,
                            "all_of" => [ [ { "source" => "procedure:0214010074" }, { "source" => "exam:0202031098" } ],
                                          [ { "source" => "exam:0202100251", "window_days" => 1826 } ] ])
      sexual = consult.merge("key" => "c7_c", "code_filter" => { "ciap2" => %w[W10], "cid10" => %w[Z30] },
                             "applies_to" => { "age_min_years" => 14, "age_max_years" => 69 })
      late = consult.merge("key" => "c3_f", "source" => "vaccine:57", "anchor" => "lmp", "window_start_days" => 140, "window_days" => 294,
                           "cbo_filter" => nil)
      expect(described_class.practices([ tests, sexual, late ])).to be_ok
    end

    {
      [ { "source" => "lab:123" } ] => "practices[0].source",
      [ { "window_days" => 0 } ] => "practices[0].window_days",
      [ { "min_count" => 51 } ] => "practices[0].min_count",
      [ { "schedule" => "proportional" } ] => "practices[0].schedule", # proportional exige âncora fixa
      [ { "anchor" => "lmp", "window_start_days" => 183 } ] => "practices[0].window_start_days",
      [ { "label" => "x" * 121 } ] => "practices[0].label",
      [ { "cbo_filter" => %w[22] } ] => "practices[0].cbo_filter",
      [ { "source" => "blood_pressure", "cbo_filter" => %w[2251] } ] => "practices[0].cbo_filter",
      [ { "code_filter" => { "icd" => %w[E11] } } ] => "practices[0].code_filter",
      [ { "applies_to" => { "sex" => "x" } } ] => "practices[0].applies_to",
      [ { "all_of" => [ [ { "source" => "acs_visit" } ] ] } ] => "practices[0].all_of",
      [ { "all_of" => [ [ { "source" => "exam:0202010503" } ], [ { "source" => "exam:0202010473" } ] ], "min_count" => 2 } ] => "practices[0].min_count"
    }.each do |overrides, field|
      it "recusa #{field} (#{overrides.first.keys.join(',')})" do
        result = described_class.practices([ consult.merge(overrides.first) ])
        expect(result.reason).to eq(:invalid_practices)
        expect(result.details[:field]).to eq(field)
      end
    end

    it "recusa chave repetida, lista vazia, mais de 20 e (template) pontos que não somam 100" do
      expect(described_class.practices([ consult, consult ]).details[:field]).to eq("practices[1].key")
      expect(described_class.practices([]).details[:field]).to eq("practices")
      many = (1..21).map { |i| consult.merge("key" => "p#{i}") }
      expect(described_class.practices(many).details[:field]).to eq("practices")
      expect(described_class.practices([ consult ], template: true).details[:field]).to eq("practices.points")
    end

    it "elegibilidade: aceita as formas do catálogo e recusa vazia ou desconhecida" do
      expect(described_class.eligibility({ "problems" => { "ciap2" => %w[T90], "cid10" => %w[E11] }, "declared" => %w[diabetes] })).to be_ok
      expect(described_class.eligibility({ "age_min_years" => 9, "age_max_years" => 69, "sex" => "female",
                                           "gender_identities_included" => %w[trans_man], "gender_identities_excluded" => %w[trans_woman] })).to be_ok
      expect(described_class.eligibility({}).details[:field]).to eq("eligibility")
      expect(described_class.eligibility({ "problems" => { "ciap2" => %w[t90] } }).details[:field]).to eq("eligibility.problems")
      expect(described_class.eligibility({ "declared" => %w[none] }).details[:field]).to eq("eligibility.declared")
      expect(described_class.eligibility({ "age_min_years" => 70, "age_max_years" => 60 }).details[:field]).to eq("eligibility.age_max_years")
      expect(described_class.eligibility({ "smoker" => true }).details[:field]).to eq("eligibility.smoker")
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar.** `test spec/services/care_lines/rules_spec.rb` → FAIL (`uninitialized constant CareLines::Rules`).

- [ ] **Step 3: Implemente.** `app/services/care_lines/catalog.rb`:

  ```ruby
  # Códigos do módulo 23 (ADR 0040; contrato §1; plano, "Valores fixados" 2).
  module CareLines
    module Catalog
      TEMPLATE_KEYS = %w[child_c2 pregnancy_c3 diabetes_c4 hypertension_c5 elderly_c6 women_c7].freeze
      INDICATORS = { "child_c2" => "C2", "pregnancy_c3" => "C3", "diabetes_c4" => "C4", "hypertension_c5" => "C5",
                     "elderly_c6" => "C6", "women_c7" => "C7" }.freeze
      SOURCE_FORMAT = /\A(consultation|blood_pressure|weight_height|acs_visit|oral_health|exam:02\d{8}|procedure:\d{10}|vaccine:[a-z0-9_]{1,30})\z/
      UNTRACKED_SOURCE = /\A(acs_visit|oral_health|vaccine:.+)\z/
      STATUSES = %w[ok due_soon overdue untracked].freeze
      SCHEDULES = %w[proportional end_of_window].freeze
      ANCHORS = %w[enrollment birth lmp rolling].freeze
      SUGGESTION_REASONS = %w[problem age_sex declared resolved_problem].freeze
      END_REASONS = %w[discharge death moved pregnancy_ended other].freeze
      CHANNELS = %w[phone home_visit wpda_message in_person].freeze
      OUTCOMES = %w[reached no_answer scheduled refused].freeze
      CONDITIONS = %w[diabetes hypertension pregnancy mental_health none].freeze
      CONDITION_TEMPLATE = { "diabetes" => "diabetes_c4", "hypertension" => "hypertension_c5", "pregnancy" => "pregnancy_c3" }.freeze
      # Contrato §4: confirma só médico (225*) ou enfermeiro (2235*).
      CONFIRM_CBO_PREFIXES = %w[225 2235].freeze
      TEXT_MAX = 500
      DUE_SOON_DAYS = 30
      LMP_MAX_AGE_DAYS = 336

      module_function

      def untracked?(source) = source.to_s.match?(UNTRACKED_SOURCE)
    end
  end
  ```

  `app/services/care_lines/rules.rb`:

  ```ruby
  # Validação única das regras de uma linha (ADR 0040; plano, Desvio 1): o
  # catálogo-base (maintenance) e as linhas próprias da cidade usam esta.
  # Recusa com o caminho do campo (nunca corta nem conserta em silêncio).
  module CareLines
    module Rules
      PRACTICE_KEYS = %w[key label citizen_label source anchor window_start_days window_days min_count schedule
                         cbo_filter code_filter applies_to all_of points].freeze
      ELIGIBILITY_KEYS = %w[problems exclude_problems age_min_days age_max_days age_min_years age_max_years sex
                            gender_identities_included gender_identities_excluded declared requires_anchor].freeze
      CODE_FORMATS = { "ciap2" => /\A[A-Z]\d{2}\z/, "cid10" => /\A[A-Z]\d{2}[0-9X]?\z/ }.freeze
      MAX_PRACTICES = 20
      MAX_CODES = 300
      SEXES = %w[female male].freeze

      class Invalid < StandardError
        attr_reader :field

        def initialize(field)
          @field = field
          super("regra inválida")
        end
      end

      module_function

      def practices(list, template: false)
        raise Invalid, "practices" unless list.is_a?(Array) && list.size.between?(1, MAX_PRACTICES)

        normalized = list.each_with_index.map { |raw, index| practice(raw, "practices[#{index}]") }
        keys = normalized.map { |p| p["key"] }
        duplicate = keys.each_index.find { |i| keys.index(keys[i]) != i }
        raise Invalid, "practices[#{duplicate}].key" if duplicate
        raise Invalid, "practices.points" if template && normalized.sum { |p| p["points"].to_i } != 100

        Result.ok(practices: normalized)
      rescue Invalid => e
        Result.fail(:invalid_practices, details: { field: e.field })
      end

      def eligibility(raw)
        raise Invalid, "eligibility" unless raw.is_a?(Hash)

        raw = raw.stringify_keys
        unknown = (raw.keys - ELIGIBILITY_KEYS).first
        raise Invalid, "eligibility.#{unknown}" if unknown

        rules = {
          "problems" => codes(raw["problems"], "eligibility.problems"),
          "exclude_problems" => codes(raw["exclude_problems"], "eligibility.exclude_problems"),
          "sex" => enum(raw["sex"], SEXES, "eligibility.sex"),
          "gender_identities_included" => list(raw["gender_identities_included"], Citizen::GENDER_IDENTITIES, "eligibility.gender_identities_included"),
          "gender_identities_excluded" => list(raw["gender_identities_excluded"], Citizen::GENDER_IDENTITIES, "eligibility.gender_identities_excluded"),
          "declared" => list(raw["declared"], Catalog::CONDITIONS - %w[none], "eligibility.declared"),
          "requires_anchor" => enum(raw["requires_anchor"], %w[lmp], "eligibility.requires_anchor")
        }
        %w[age_min_days age_max_days].each { |k| rules[k] = int(raw[k], 0..47_450, "eligibility.#{k}") }
        %w[age_min_years age_max_years].each { |k| rules[k] = int(raw[k], 0..130, "eligibility.#{k}") }
        order!(rules, "age_min_days", "age_max_days", "eligibility")
        order!(rules, "age_min_years", "age_max_years", "eligibility")
        criteria = rules.values_at("problems", "age_min_days", "age_max_days", "age_min_years", "age_max_years", "sex", "declared")
        raise Invalid, "eligibility" if criteria.all?(&:nil?)

        Result.ok(eligibility: rules.compact)
      rescue Invalid => e
        Result.fail(:invalid_eligibility, details: { field: e.field })
      end

      def practice(raw, path)
        raise Invalid, path unless raw.is_a?(Hash)

        raw = raw.stringify_keys
        unknown = (raw.keys - PRACTICE_KEYS).first
        raise Invalid, "#{path}.#{unknown}" if unknown

        source = raw["source"].to_s
        raise Invalid, "#{path}.source" unless source.match?(Catalog::SOURCE_FORMAT)

        anchor = enum(raw["anchor"], Catalog::ANCHORS, "#{path}.anchor") || raise(Invalid, "#{path}.anchor")
        window = int(raw["window_days"], 1..7300, "#{path}.window_days") || raise(Invalid, "#{path}.window_days")
        start = int(raw["window_start_days"], 0..7299, "#{path}.window_start_days") || 0
        raise Invalid, "#{path}.window_start_days" if start.positive? && (anchor == "rolling" || start >= window)

        schedule = enum(raw["schedule"], Catalog::SCHEDULES, "#{path}.schedule") || "end_of_window"
        raise Invalid, "#{path}.schedule" if schedule == "proportional" && anchor == "rolling"

        all_of = groups(raw["all_of"], "#{path}.all_of")
        min_count = int(raw["min_count"], 1..50, "#{path}.min_count") || 1
        raise Invalid, "#{path}.min_count" if all_of && all_of.size > 1 && min_count != 1

        consultation = source == "consultation"
        cbo = raw["cbo_filter"]
        unless cbo.nil?
          raise Invalid, "#{path}.cbo_filter" unless consultation && cbo.is_a?(Array) && cbo.size.between?(1, 30) &&
                                                     cbo.all? { |c| c.is_a?(String) && c.match?(/\A\d{4,6}\z/) }
        end
        code_filter = raw["code_filter"].nil? ? nil : codes(raw["code_filter"], "#{path}.code_filter")
        raise Invalid, "#{path}.code_filter" if code_filter && !consultation

        {
          "key" => text(raw["key"], /\A[a-z0-9_]{1,40}\z/, "#{path}.key"),
          "label" => text(raw["label"], /\A.{1,120}\z/m, "#{path}.label"),
          "citizen_label" => raw["citizen_label"].nil? ? nil : text(raw["citizen_label"], /\A.{1,80}\z/m, "#{path}.citizen_label"),
          "source" => source, "anchor" => anchor, "window_start_days" => start, "window_days" => window,
          "min_count" => min_count, "schedule" => schedule, "cbo_filter" => cbo, "code_filter" => code_filter,
          "applies_to" => applies_to(raw["applies_to"], "#{path}.applies_to"), "all_of" => all_of,
          "points" => int(raw["points"], 0..100, "#{path}.points")
        }
      end

      def groups(raw, path)
        return nil if raw.nil?
        raise Invalid, path unless raw.is_a?(Array) && raw.size.between?(1, 6)

        raw.map do |group|
          raise Invalid, path unless group.is_a?(Array) && group.size.between?(1, 12)

          group.map do |alt|
            raise Invalid, path unless alt.is_a?(Hash)

            alt = alt.stringify_keys
            source = alt["source"].to_s
            raise Invalid, path unless (alt.keys - %w[source window_days]).empty? && source.match?(Catalog::SOURCE_FORMAT) &&
                                       !Catalog.untracked?(source) && source != "consultation"

            { "source" => source, "window_days" => int(alt["window_days"], 1..7300, path) }.compact
          end
        end
      end

      def applies_to(raw, path)
        return nil if raw.nil?
        raise Invalid, path unless raw.is_a?(Hash) && (raw.keys.map(&:to_s) - %w[age_min_years age_max_years sex]).empty?

        raw = raw.stringify_keys
        rules = { "age_min_years" => int(raw["age_min_years"], 0..130, path), "age_max_years" => int(raw["age_max_years"], 0..130, path),
                  "sex" => enum(raw["sex"], SEXES, path) }
        order!(rules, "age_min_years", "age_max_years", path)
        rules.compact.presence || raise(Invalid, path)
      end

      def codes(raw, path)
        return nil if raw.nil?
        raise Invalid, path unless raw.is_a?(Hash) && raw.any? && (raw.keys.map(&:to_s) - CODE_FORMATS.keys).empty?

        raw.stringify_keys.to_h do |terminology, list|
          valid = list.is_a?(Array) && list.size.between?(1, MAX_CODES) &&
                  list.all? { |code| code.is_a?(String) && code.match?(CODE_FORMATS.fetch(terminology)) }
          raise Invalid, path unless valid

          [ terminology, list.uniq ]
        end
      end

      def list(raw, allowed, path)
        return nil if raw.nil?
        raise Invalid, path unless raw.is_a?(Array) && raw.any? && raw.all? { |v| allowed.include?(v) }

        raw.uniq
      end

      def enum(raw, allowed, path)
        return nil if raw.nil?
        raise Invalid, path unless allowed.include?(raw)

        raw
      end

      def int(raw, range, path)
        return nil if raw.nil?
        raise Invalid, path unless raw.is_a?(Integer) && range.cover?(raw)

        raw
      end

      def text(raw, format, path)
        raise Invalid, path unless raw.is_a?(String) && raw.strip == raw && raw.match?(format)

        raw
      end

      def order!(rules, min_key, max_key, path)
        min, max = rules.values_at(min_key, max_key)
        raise Invalid, (path.include?(".") ? path : "#{path}.#{max_key}") if min && max && max < min
      end
      private_class_method :practice, :groups, :applies_to, :codes, :list, :enum, :int, :text, :order!
    end
  end
  ```

- [ ] **Step 4: Rode e veja passar.** `test spec/services/care_lines/rules_spec.rb` → PASS.

- [ ] **Step 5: Commit.**

  ```bash
  git add app/services/care_lines/catalog.rb app/services/care_lines/rules.rb spec/services/care_lines/rules_spec.rb
  git commit -m "feat: validate care line eligibility and practice rules

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 3: Catálogo-base na plataforma — `care_line_templates`, carga e versões (F-23.1)

**Files:**
- Create: `db/platform_migrate/20261022900001_create_care_line_templates.rb`, `app/models/care_line_template.rb`, `app/services/care_lines/templates/save.rb`, `app/services/care_lines/templates/load.rb`, `config/care_lines/catalog-base.yml`, `lib/tasks/care_lines.rake`
- Modify: `db/platform_schema.rb` (tabela e `define(version: 2026_10_22_900001)`), `db/platform_triggers.sql`, `spec/events/platform_event_payload_guard_spec.rb` (`care_line.template_changed` em `R18_PLATFORM_EVENT_NAMES`)
- Test: `spec/services/care_lines/templates/save_spec.rb`, `spec/services/care_lines/templates/load_spec.rb`, `spec/services/care_lines/catalog_base_spec.rb`

**Interfaces:**
- Consumes: `CareLines::Rules.practices(list, template: true)`, `CareLines::Rules.eligibility(hash)`, `CareLines::Catalog::{TEMPLATE_KEYS,INDICATORS,CONDITIONS}` (Task 2).
- Produces: `CareLineTemplate` (`PlatformRecord`; colunas `key version name indicator declared_condition eligibility practices created_by_maintainer_id created_at`); `CareLineTemplate.current(key) → CareLineTemplate|nil` (maior versão); `CareLineTemplate.latest → Array` (a maior versão de cada chave, na ordem de `TEMPLATE_KEYS`); `CareLines::Templates::Save.call(attrs, maintainer: nil) → Result` (`payload[:template]`; falhas `:invalid_key`, `:invalid_indicator`, `:invalid_name`, `:invalid_declared_condition`, `:invalid_practices`, `:invalid_eligibility`, com `details[:field]`); `CareLines::Templates::Load.call(path = DEFAULT_PATH) → { created: Integer }`; rake `care_lines:templates:load`.

- [ ] **Step 1: Testes que falham.** `spec/services/care_lines/templates/save_spec.rb`:

  ```ruby
  require "rails_helper"

  RSpec.describe CareLines::Templates::Save do
    def attrs(**over)
      entry = CareLines::Templates::Load.entries.find { |e| e["key"] == "diabetes_c4" }
      entry.slice("key", "name", "indicator", "declared_condition", "eligibility", "practices").merge(over.transform_keys(&:to_s))
    end

    it "grava versões em sequência, nunca altera a anterior, e audita só ids" do
      first = described_class.call(attrs).payload[:template]
      second = described_class.call(attrs(name: "Diabetes (revisão 2027)")).payload[:template]
      expect([ first.version, second.version ]).to eq([ 1, 2 ])
      expect(CareLineTemplate.current("diabetes_c4")).to eq(second)
      expect { PlatformRecord.connection.execute("UPDATE care_line_templates SET name = 'x' WHERE id = '#{first.id}'") }
        .to raise_error(ActiveRecord::StatementInvalid, /care_line_templates: immutable/)
      event = PlatformEvent.where(name: "care_line.template_changed").order(:created_at).last
      expect(event.payload.keys).to match_array(%w[care_line_template_id template_key version])
    end

    it "recusa chave fora do catálogo, indicador trocado, pontos que não somam 100 e nome longo" do
      expect(described_class.call(attrs(key: "asthma_c9")).reason).to eq(:invalid_key)
      expect(described_class.call(attrs(indicator: "C5")).reason).to eq(:invalid_indicator)
      practices = attrs["practices"].map { |p| p.merge("points" => 10) }
      expect(described_class.call(attrs(practices: practices)).details).to eq(field: "practices.points")
      expect(described_class.call(attrs(name: "x" * 121)).reason).to eq(:invalid_name)
      expect(described_class.call(attrs(declared_condition: "none")).reason).to eq(:invalid_declared_condition)
      expect(CareLineTemplate.count).to eq(0)
    end
  end
  ```

  `spec/services/care_lines/templates/load_spec.rb`:

  ```ruby
  require "rails_helper"

  RSpec.describe CareLines::Templates::Load do
    it "cria a versão 1 das seis linhas e é idempotente (nunca sobrescreve a curadoria)" do
      expect(described_class.call).to eq(created: 6)
      CareLines::Templates::Save.call(CareLineTemplate.current("elderly_c6").attributes.slice(*%w[key indicator declared_condition eligibility practices])
                                                          .merge("name" => "Pessoa idosa (curada)"))
      expect(described_class.call).to eq(created: 0)
      expect(CareLineTemplate.latest.map { |t| [ t.key, t.version ] })
        .to eq([ [ "child_c2", 1 ], [ "pregnancy_c3", 1 ], [ "diabetes_c4", 1 ], [ "hypertension_c5", 1 ], [ "elderly_c6", 2 ], [ "women_c7", 1 ] ])
    end
  end
  ```

  `spec/services/care_lines/catalog_base_spec.rb` (o YAML é a tabela "Catálogo-base fixado" do plano — esta spec é a conferência da Task 0 dentro do repo):

  ```ruby
  require "rails_helper"

  RSpec.describe "Catálogo-base das linhas (notas metodológicas C2–C7 de 19/06/2026)" do
    let(:entries) { CareLines::Templates::Load.entries.index_by { |e| e["key"] } }
    let(:md_nurse) { %w[2231 2251 2252 2253 2235] }

    def practice(key, practice_key) = entries.fetch(key)["practices"].find { |p| p["key"] == practice_key }

    it "seis linhas válidas pelo Rules, pontos somando 100" do
      expect(entries.keys).to eq(CareLines::Catalog::TEMPLATE_KEYS)
      entries.each_value do |e|
        expect(CareLines::Rules.practices(e["practices"], template: true)).to be_ok
        expect(CareLines::Rules.eligibility(e["eligibility"])).to be_ok
        expect(e["indicator"]).to eq(CareLines::Catalog::INDICATORS.fetch(e["key"]))
      end
    end

    it "pontos por prática como os Quadros 01" do
      expect(entries.transform_values { |e| e["practices"].map { |p| p["points"] } }).to eq(
        "child_c2" => [ 20, 20, 20, 20, 20 ], "pregnancy_c3" => [ 10, 9, 9, 9, 9, 9, 9, 9, 9, 9, 9 ],
        "diabetes_c4" => [ 20, 15, 15, 20, 15, 15 ], "hypertension_c5" => [ 25, 25, 25, 25 ],
        "elderly_c6" => [ 25, 25, 25, 25 ], "women_c7" => [ 20, 30, 30, 20 ]
      )
    end

    it "elegibilidade por CIAP/CID, idade e sexo" do
      expect(entries["diabetes_c4"]["eligibility"]["problems"]).to eq("ciap2" => %w[T89 T90], "cid10" => %w[E10 E11 E14])
      expect(entries["hypertension_c5"]["eligibility"]["problems"]).to eq("ciap2" => %w[K86 K87], "cid10" => %w[I10 I11 I12 I13 I15 O10 O11])
      expect(entries["pregnancy_c3"]["eligibility"]).to include("requires_anchor" => "lmp",
                                                                 "exclude_problems" => { "ciap2" => %w[W82 W83], "cid10" => %w[O02 O03 O04 O05 O06 Z303] })
      expect(entries["pregnancy_c3"]["eligibility"]["problems"]["ciap2"]).to eq(%w[W03 W78 W79 W81 W84 W85])
      expect(entries["child_c2"]["eligibility"]).to eq("age_max_days" => 730)
      expect(entries["elderly_c6"]["eligibility"]).to eq("age_min_years" => 60)
      expect(entries["women_c7"]["eligibility"]).to eq("age_min_years" => 9, "age_max_years" => 69, "sex" => "female",
                                                       "gender_identities_included" => %w[trans_man],
                                                       "gender_identities_excluded" => %w[trans_woman])
    end

    it "janelas, contagens, âncoras e fontes que fazem a diferença" do
      expect(practice("child_c2", "c2_b")).to include("anchor" => "birth", "window_days" => 730, "min_count" => 9, "schedule" => "proportional",
                                                      "cbo_filter" => md_nurse)
      expect(practice("pregnancy_c3", "c3_a")).to include("anchor" => "lmp", "window_days" => 84, "min_count" => 1)
      expect(practice("pregnancy_c3", "c3_g")["all_of"].size).to eq(4)
      expect(practice("pregnancy_c3", "c3_h")).to include("window_start_days" => 196, "window_days" => 294)
      expect(practice("pregnancy_c3", "c3_k")["source"]).to eq("oral_health")
      expect(practice("diabetes_c4", "c4_e")).to include("source" => "exam:0202010503", "anchor" => "rolling", "window_days" => 365)
      expect(practice("diabetes_c4", "c4_f")).to include("source" => "procedure:0301040095", "window_days" => 365)
      expect(practice("hypertension_c5", "c5_a")).to include("source" => "consultation", "window_days" => 183, "cbo_filter" => md_nurse)
      expect(practice("women_c7", "c7_a")["all_of"].first.last).to eq("source" => "exam:0202100251", "window_days" => 1826)
      expect(practice("women_c7", "c7_c")["applies_to"]).to eq("age_min_years" => 14, "age_max_years" => 69)
      untracked = entries.values.flat_map { |e| e["practices"] }.select { |p| CareLines::Catalog.untracked?(p["source"]) }.map { |p| p["key"] }
      expect(untracked).to eq(%w[c2_d c2_e c3_e c3_f c3_j c3_k c4_d c5_d c6_c c6_d c7_b])
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar.** `test spec/services/care_lines/templates spec/services/care_lines/catalog_base_spec.rb` → FAIL (`uninitialized constant CareLines::Templates`).

- [ ] **Step 3: Migração de plataforma** `db/platform_migrate/20261022900001_create_care_line_templates.rb`:

  ```ruby
  # Catálogo-base das linhas de cuidado (ADR 0040; spec §4): versões
  # imutáveis (trigger em db/platform_triggers.sql) das seis linhas C2–C7 com
  # as regras das notas metodológicas. Dado público, sem pessoa. Irreversível.
  class CreateCareLineTemplates < ActiveRecord::Migration[8.1]
    KEYS = %w[child_c2 pregnancy_c3 diabetes_c4 hypertension_c5 elderly_c6 women_c7].freeze

    def up
      create_table :care_line_templates, id: :uuid, default: -> { "gen_random_uuid()" } do |t|
        t.string :key, null: false
        t.integer :version, null: false
        t.string :name, limit: 120, null: false
        t.string :indicator, limit: 2, null: false
        t.string :declared_condition
        t.jsonb :eligibility, null: false
        t.jsonb :practices, null: false
        t.uuid :created_by_maintainer_id
        t.datetime :created_at, null: false
        t.index %i[key version], unique: true, name: "idx_care_line_templates_version"
        t.check_constraint "key::text = ANY (ARRAY[#{KEYS.map { |k| "'#{k}'::text" }.join(', ')}])", name: "ck_care_line_templates_key"
        t.check_constraint "version >= 1 AND indicator::text ~ '^C[2-7]$'::text AND jsonb_typeof(practices) = 'array'::text " \
                           "AND jsonb_typeof(eligibility) = 'object'::text", name: "ck_care_line_templates_values"
      end
      execute File.read(Rails.root.join("db/platform_triggers.sql"))
    end

    def down
      raise ActiveRecord::IrreversibleMigration
    end
  end
  ```

  Em `db/platform_triggers.sql`, no fim:

  ```sql
  -- care_line_templates (ADR 0040; plano 2026-10-22 Desvio 12): versão nova é
  -- linha nova; nenhuma versão muda nem some.
  CREATE OR REPLACE FUNCTION rota_care_line_templates_immutable() RETURNS trigger AS $fn$
  BEGIN
    RAISE EXCEPTION 'care_line_templates: immutable (% refused)', TG_OP;
  END;
  $fn$ LANGUAGE plpgsql;

  DO $do$
  BEGIN
    IF to_regclass('public.care_line_templates') IS NOT NULL THEN
      EXECUTE 'DROP TRIGGER IF EXISTS care_line_templates_immutable ON care_line_templates';
      EXECUTE 'CREATE TRIGGER care_line_templates_immutable BEFORE UPDATE OR DELETE ON care_line_templates
        FOR EACH ROW EXECUTE FUNCTION rota_care_line_templates_immutable()';
    END IF;
  END
  $do$;
  ```

  Em `db/platform_schema.rb`: `define(version: 2026_10_22_900001)` e, na ordem alfabética (antes de `"channel_..."`/depois de `"anvisa_..."`, onde couber), o bloco que o dump geraria:

  ```ruby
    create_table "care_line_templates", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
      t.datetime "created_at", null: false
      t.uuid "created_by_maintainer_id"
      t.string "declared_condition"
      t.jsonb "eligibility", null: false
      t.string "indicator", limit: 2, null: false
      t.string "key", null: false
      t.string "name", limit: 120, null: false
      t.jsonb "practices", null: false
      t.integer "version", null: false
      t.index ["key", "version"], name: "idx_care_line_templates_version", unique: true
      t.check_constraint "key::text = ANY (ARRAY['child_c2'::text, 'pregnancy_c3'::text, 'diabetes_c4'::text, 'hypertension_c5'::text, 'elderly_c6'::text, 'women_c7'::text])", name: "ck_care_line_templates_key"
      t.check_constraint "version >= 1 AND indicator::text ~ '^C[2-7]$'::text AND jsonb_typeof(practices) = 'array'::text AND jsonb_typeof(eligibility) = 'object'::text", name: "ck_care_line_templates_values"
    end
  ```
  O `bin/test` roda `platform:bootstrap db:migrate platform:triggers` ao montar o banco de teste (`bin/test-db-bootstrap`): a migração e o trigger são exercitados de verdade pelo `save_spec` (imutabilidade); o dump à mão segue o formato do bloco de `lab_exam_templates` (colunas em ordem alfabética).

- [ ] **Step 4: Modelo e serviços.** `app/models/care_line_template.rb`:

  ```ruby
  # Versão do catálogo-base de uma linha de cuidado (ADR 0040). Imutável.
  class CareLineTemplate < PlatformRecord
    def self.current(key) = where(key: key.to_s).order(version: :desc).first

    def self.latest
      rows = select("DISTINCT ON (key) care_line_templates.*").order(:key, version: :desc).to_a.index_by(&:key)
      CareLines::Catalog::TEMPLATE_KEYS.filter_map { |key| rows[key] }
    end

    def readonly? = persisted?
  end
  ```

  `app/services/care_lines/templates/save.rb`:

  ```ruby
  # upsertCareLineTemplate (contrato §8): toda gravação é a versão seguinte da
  # chave; a anterior fica (trigger). Auditoria care_line.template_changed só
  # com ids (R18).
  module CareLines
    module Templates
      module Save
        module_function

        def call(attrs, maintainer: nil)
          attrs = attrs.to_h.stringify_keys
          key = attrs["key"].to_s
          return fail(:invalid_key, "key") unless Catalog::TEMPLATE_KEYS.include?(key)
          return fail(:invalid_indicator, "indicator") unless attrs["indicator"] == Catalog::INDICATORS.fetch(key)

          name = attrs["name"].to_s
          return fail(:invalid_name, "name") unless name.strip == name && name.length.between?(1, 120)

          condition = attrs["declared_condition"]
          return fail(:invalid_declared_condition, "declaredCondition") unless condition.nil? || (Catalog::CONDITIONS - %w[none]).include?(condition)

          practices = Rules.practices(attrs["practices"], template: true)
          return practices if practices.failure?

          eligibility = Rules.eligibility(attrs["eligibility"])
          return eligibility if eligibility.failure?

          template = PlatformRecord.transaction do
            PlatformRecord.connection.execute("SELECT pg_advisory_xact_lock(#{Zlib.crc32("care_line_templates:#{key}")})")
            version = CareLineTemplate.where(key: key).maximum(:version).to_i + 1
            CareLineTemplate.create!(key: key, version: version, name: name, indicator: attrs["indicator"], declared_condition: condition,
                                     eligibility: eligibility.payload[:eligibility], practices: practices.payload[:practices],
                                     created_by_maintainer_id: maintainer&.id, created_at: Time.current)
          end
          Platform.audit("care_line.template_changed", care_line_template_id: template.id, template_key: key, version: template.version)
          Result.ok(template: template)
        end

        def fail(code, field) = Result.fail(code, details: { field: field })
        private_class_method :fail
      end
    end
  end
  ```

  `app/services/care_lines/templates/load.rb`:

  ```ruby
  # Carga inicial do catálogo-base (config/care_lines/catalog-base.yml, a
  # tabela da prova técnica). Idempotente: só cria a versão 1 da chave que
  # não existe; nunca sobrescreve a curadoria do maintenance.
  module CareLines
    module Templates
      module Load
        DEFAULT_PATH = Rails.root.join("config/care_lines/catalog-base.yml")

        module_function

        # aliases: true — o YAML repete listas por âncora (&md_nurse, &syphilis, &hiv).
        def entries(path = DEFAULT_PATH) = YAML.safe_load_file(path, aliases: true)

        def call(path = DEFAULT_PATH)
          created = entries(path).count do |entry|
            next false if CareLineTemplate.exists?(key: entry.fetch("key"))

            result = Save.call(entry)
            raise ArgumentError, "catálogo-base: #{entry['key']} #{result.reason} #{result.details}" if result.failure?

            true
          end
          { created: created }
        end
      end
    end
  end
  ```

  `lib/tasks/care_lines.rake`:

  ```ruby
  namespace :care_lines do
    namespace :templates do
      desc "Carrega o catálogo-base das linhas de cuidado C2–C7 (idempotente; nunca sobrescreve)."
      task load: :environment do
        result = CareLines::Templates::Load.call
        puts "[care_lines:templates:load] versões criadas: #{result[:created]}"
      end
    end
  end
  ```

  `spec/events/platform_event_payload_guard_spec.rb`: acrescente `care_line.template_changed` em `R18_PLATFORM_EVENT_NAMES` (logo depois de `lab.template_changed`).

- [ ] **Step 5: O YAML** `config/care_lines/catalog-base.yml` (fonte: "Catálogo-base fixado"; âncoras YAML só para as listas repetidas):

  ```yaml
  # Catálogo-base das linhas de cuidado (ADR 0040). Fonte: notas metodológicas
  # C2–C7 da SAPS/MS assinadas em 19/06/2026 e NT 8/2026 (prova técnica em
  # docs/pesquisa/2026-10-22-linhas-de-cuidado-prova-tecnica.md). Janelas em
  # dias (6 meses = 183, 12 = 365, 24 = 730, 36 = 1095, 60 = 1826). Carga:
  # bin/rails care_lines:templates:load. Mudança depois da carga: só pelo
  # maintenance (nova versão).
  - key: child_c2
    indicator: C2
    name: "Criança até 2 anos (desenvolvimento infantil)"
    declared_condition: null
    eligibility:
      age_max_days: 730
    practices:
      - { key: c2_a, label: "1ª consulta com médico(a) ou enfermeiro(a) até o 30º dia de vida", citizen_label: "Primeira consulta do bebê",
          source: consultation, anchor: birth, window_days: 30, min_count: 1, schedule: end_of_window,
          cbo_filter: &md_nurse ["2231", "2251", "2252", "2253", "2235"], points: 20 }
      - { key: c2_b, label: "9 consultas com médico(a) ou enfermeiro(a) até os 2 anos", citizen_label: "Consultas de acompanhamento da criança",
          source: consultation, anchor: birth, window_days: 730, min_count: 9, schedule: proportional, cbo_filter: *md_nurse, points: 20 }
      - { key: c2_c, label: "9 registros de peso e altura até os 2 anos", citizen_label: "Medir peso e altura da criança",
          source: weight_height, anchor: birth, window_days: 730, min_count: 9, schedule: proportional, points: 20 }
      - { key: c2_d, label: "2 visitas do ACS (até 30 dias e até 6 meses de vida)", source: acs_visit, anchor: birth, window_days: 180,
          min_count: 2, schedule: end_of_window, points: 20 }
      - { key: c2_e, label: "Vacinas penta, VIP, SCR e pneumocócica com todas as doses", source: "vaccine:c2_schedule", anchor: birth,
          window_days: 730, min_count: 1, schedule: end_of_window, points: 20 }
  - key: pregnancy_c3
    indicator: C3
    name: "Gestante e puérpera"
    declared_condition: pregnancy
    eligibility:
      problems:
        ciap2: [W03, W78, W79, W81, W84, W85]
        cid10: [O10, O11, O12, O13, O14, O15, O16, O20, O21, O22, O23, O24, O25, O26, O28, O29, O30, O31, O32, O33, O34, O35, O36,
                O40, O41, O43, O44, O46, O47, O48, O752, O753, O98, O990, O991, O992, O993, O994, O995, O996, O997, Z321, Z33, Z34,
                Z35, Z36, Z640]
      exclude_problems:
        ciap2: [W82, W83]
        cid10: [O02, O03, O04, O05, O06, Z303]
      declared: [pregnancy]
      requires_anchor: lmp
    practices:
      - { key: c3_a, label: "1ª consulta de pré-natal até a 12ª semana", citizen_label: "Primeira consulta do pré-natal",
          source: consultation, anchor: lmp, window_days: 84, min_count: 1, schedule: end_of_window, cbo_filter: *md_nurse, points: 10 }
      - { key: c3_b, label: "7 consultas de pré-natal na gestação", citizen_label: "Consultas do pré-natal",
          source: consultation, anchor: lmp, window_days: 294, min_count: 7, schedule: proportional, cbo_filter: *md_nurse, points: 9 }
      - { key: c3_c, label: "7 aferições de pressão arterial na gestação", citizen_label: "Medir a pressão",
          source: blood_pressure, anchor: lmp, window_days: 294, min_count: 7, schedule: proportional, points: 9 }
      - { key: c3_d, label: "7 registros de peso e altura na gestação", citizen_label: "Medir peso e altura",
          source: weight_height, anchor: lmp, window_days: 294, min_count: 7, schedule: proportional, points: 9 }
      - { key: c3_e, label: "3 visitas do ACS após a 1ª consulta", source: acs_visit, anchor: lmp, window_days: 294, min_count: 3,
          schedule: end_of_window, points: 9 }
      - { key: c3_f, label: "Vacina dTpa a partir da 20ª semana", source: "vaccine:57", anchor: lmp, window_start_days: 140,
          window_days: 294, min_count: 1, schedule: end_of_window, points: 9 }
      - key: c3_g
        label: "Testes de sífilis, HIV e hepatites B e C no 1º trimestre"
        citizen_label: "Exames do primeiro trimestre"
        source: "procedure:0214010074"
        anchor: lmp
        window_days: 97
        min_count: 1
        schedule: end_of_window
        points: 9
        all_of:
          - &syphilis [{ source: "procedure:0214010074" }, { source: "procedure:0214010082" }, { source: "procedure:0214010252" },
                       { source: "exam:0202031098" }, { source: "exam:0202031110" }, { source: "exam:0202031179" }]
          - &hiv [{ source: "procedure:0214010040" }, { source: "procedure:0214010279" }, { source: "procedure:0214010058" },
                  { source: "exam:0213010780" }, { source: "exam:0213010500" }, { source: "exam:0202030300" }]
          - [{ source: "procedure:0214010104" }, { source: "procedure:0214010236" }, { source: "exam:0202030784" },
             { source: "exam:0202030970" }, { source: "exam:0213010208" }]
          - [{ source: "procedure:0214010090" }, { source: "procedure:0214010309" }, { source: "exam:0202030059" },
             { source: "exam:0202030679" }]
      - key: c3_h
        label: "Testes de sífilis e HIV no 3º trimestre"
        citizen_label: "Exames do terceiro trimestre"
        source: "procedure:0214010074"
        anchor: lmp
        window_start_days: 196
        window_days: 294
        min_count: 1
        schedule: end_of_window
        points: 9
        all_of: [*syphilis, *hiv]
      - key: c3_i
        label: "Consulta no puerpério"
        citizen_label: "Consulta depois do parto"
        source: consultation
        anchor: lmp
        window_start_days: 154
        window_days: 336
        min_count: 1
        schedule: end_of_window
        cbo_filter: *md_nurse
        code_filter:
          ciap2: [P29, W18, W19, W70, W90, W91, W92, W93, W94, W95, W96]
          cid10: [F53, M830, O10, O152, O266, O722, O723, O85, O86, O87, O90, O91, O92, O94, O98, O99, Z37, Z38, Z39]
        points: 9
      - { key: c3_j, label: "Visita do ACS no puerpério", source: acs_visit, anchor: lmp, window_start_days: 294, window_days: 336,
          min_count: 1, schedule: end_of_window, points: 9 }
      - { key: c3_k, label: "Atividade de saúde bucal na gestação", source: oral_health, anchor: lmp, window_days: 294, min_count: 1,
          schedule: end_of_window, points: 9 }
  - key: diabetes_c4
    indicator: C4
    name: "Pessoa com diabetes"
    declared_condition: diabetes
    eligibility:
      problems: { ciap2: [T89, T90], cid10: [E10, E11, E14] }
      declared: [diabetes]
    practices:
      - { key: c4_a, label: "Consulta com médico(a) ou enfermeiro(a) nos últimos 6 meses", citizen_label: "Consulta de acompanhamento",
          source: consultation, anchor: rolling, window_days: 183, min_count: 1, schedule: end_of_window, cbo_filter: *md_nurse, points: 20 }
      - { key: c4_b, label: "Pressão arterial aferida nos últimos 6 meses", citizen_label: "Medir a pressão",
          source: blood_pressure, anchor: rolling, window_days: 183, min_count: 1, schedule: end_of_window, points: 15 }
      - { key: c4_c, label: "Peso e altura nos últimos 12 meses", citizen_label: "Medir peso e altura",
          source: weight_height, anchor: rolling, window_days: 365, min_count: 1, schedule: end_of_window, points: 15 }
      - { key: c4_d, label: "2 visitas do ACS nos últimos 12 meses", source: acs_visit, anchor: rolling, window_days: 365, min_count: 2,
          schedule: end_of_window, points: 20 }
      - { key: c4_e, label: "Hemoglobina glicada solicitada ou avaliada nos últimos 12 meses", citizen_label: "Exame de hemoglobina glicada",
          source: "exam:0202010503", anchor: rolling, window_days: 365, min_count: 1, schedule: end_of_window, points: 15 }
      - { key: c4_f, label: "Avaliação dos pés nos últimos 12 meses", citizen_label: "Avaliação dos pés",
          source: "procedure:0301040095", anchor: rolling, window_days: 365, min_count: 1, schedule: end_of_window, points: 15 }
  - key: hypertension_c5
    indicator: C5
    name: "Pessoa com hipertensão"
    declared_condition: hypertension
    eligibility:
      problems: { ciap2: [K86, K87], cid10: [I10, I11, I12, I13, I15, O10, O11] }
      declared: [hypertension]
    practices:
      - { key: c5_a, label: "Consulta com médico(a) ou enfermeiro(a) nos últimos 6 meses", citizen_label: "Consulta de acompanhamento",
          source: consultation, anchor: rolling, window_days: 183, min_count: 1, schedule: end_of_window, cbo_filter: *md_nurse, points: 25 }
      - { key: c5_b, label: "Pressão arterial aferida nos últimos 6 meses", citizen_label: "Medir a pressão",
          source: blood_pressure, anchor: rolling, window_days: 183, min_count: 1, schedule: end_of_window, points: 25 }
      - { key: c5_c, label: "Peso e altura nos últimos 12 meses", citizen_label: "Medir peso e altura",
          source: weight_height, anchor: rolling, window_days: 365, min_count: 1, schedule: end_of_window, points: 25 }
      - { key: c5_d, label: "2 visitas do ACS nos últimos 12 meses", source: acs_visit, anchor: rolling, window_days: 365, min_count: 2,
          schedule: end_of_window, points: 25 }
  - key: elderly_c6
    indicator: C6
    name: "Pessoa idosa"
    declared_condition: null
    eligibility:
      age_min_years: 60
    practices:
      - { key: c6_a, label: "Consulta com médico(a) ou enfermeiro(a) nos últimos 12 meses", citizen_label: "Consulta de acompanhamento",
          source: consultation, anchor: rolling, window_days: 365, min_count: 1, schedule: end_of_window, cbo_filter: *md_nurse, points: 25 }
      - { key: c6_b, label: "Peso e altura no mesmo dia nos últimos 12 meses", citizen_label: "Medir peso e altura",
          source: weight_height, anchor: rolling, window_days: 365, min_count: 1, schedule: end_of_window, points: 25 }
      - { key: c6_c, label: "2 visitas do ACS nos últimos 12 meses", source: acs_visit, anchor: rolling, window_days: 365, min_count: 2,
          schedule: end_of_window, points: 25 }
      - { key: c6_d, label: "Vacina contra influenza nos últimos 12 meses", source: "vaccine:33", anchor: rolling, window_days: 365,
          min_count: 1, schedule: end_of_window, points: 25 }
  - key: women_c7
    indicator: C7
    name: "Mulher e prevenção do câncer"
    declared_condition: null
    eligibility:
      age_min_years: 9
      age_max_years: 69
      sex: female
      gender_identities_included: [trans_man]
      gender_identities_excluded: [trans_woman]
    practices:
      - key: c7_a
        label: "Rastreamento do câncer do colo do útero (36 meses; HPV molecular 60 meses)"
        citizen_label: "Exame preventivo do colo do útero"
        source: "exam:0203010086"
        anchor: rolling
        window_days: 1095
        min_count: 1
        schedule: end_of_window
        applies_to: { age_min_years: 25, age_max_years: 64 }
        points: 20
        all_of:
          - [{ source: "exam:0203010086" }, { source: "exam:0203010019" }, { source: "procedure:0201020033" },
             { source: "procedure:0201020076" }, { source: "procedure:0201020084" }, { source: "exam:0202100251", window_days: 1826 }]
      - { key: c7_b, label: "1 dose da vacina HPV entre 9 e 14 anos", source: "vaccine:67", anchor: birth, window_start_days: 3287,
          window_days: 5478, min_count: 1, schedule: end_of_window, applies_to: { age_min_years: 9, age_max_years: 14, sex: female }, points: 30 }
      - key: c7_c
        label: "Atendimento de saúde sexual e reprodutiva nos últimos 12 meses"
        citizen_label: "Consulta de saúde da mulher"
        source: consultation
        anchor: rolling
        window_days: 365
        min_count: 1
        schedule: end_of_window
        cbo_filter: ["2231", "2251", "2252", "2253", "2235", "251605", "223445", "223605", "223810", "223710", "251510", "223905", "3222"]
        code_filter:
          ciap2: [B25, W02, W10, W11, W12, W13, W14, W15, W79, W82, X01, X02, X03, X04, X05, X06, X07, X08, X09, X10, X11, X12, X13, X23,
                  X24, X82, X89, Y14]
          cid10: [N80, N91, N92, N93, N94, N95, N96, N97, O03, O04, R102, T742, Y05, Z123, Z124, Z205, Z206, Z30, Z31, Z32, Z600, Z630,
                  Z640, Z70, Z717, Z725]
        applies_to: { age_min_years: 14, age_max_years: 69 }
        points: 30
      - key: c7_d
        label: "Mamografia solicitada ou avaliada nos últimos 24 meses"
        citizen_label: "Mamografia"
        source: "exam:0204030030"
        anchor: rolling
        window_days: 730
        min_count: 1
        schedule: end_of_window
        applies_to: { age_min_years: 50, age_max_years: 69 }
        points: 20
        all_of:
          - [{ source: "exam:0204030030" }, { source: "exam:0204030188" }]
  ```

  (O `source` de uma prática com `all_of` repete a primeira alternativa, para o formato do contrato continuar com um `source` válido; quem avalia é o `all_of`.)

- [ ] **Step 6: Rode e veja passar.** `test spec/services/care_lines/templates spec/services/care_lines/catalog_base_spec.rb spec/events/platform_event_payload_guard_spec.rb` → PASS.

- [ ] **Step 7: Commit.**

  ```bash
  git add db/platform_migrate/20261022900001_create_care_line_templates.rb db/platform_schema.rb db/platform_triggers.sql app/models/care_line_template.rb app/services/care_lines/templates config/care_lines/catalog-base.yml lib/tasks/care_lines.rake spec/services/care_lines/templates spec/services/care_lines/catalog_base_spec.rb spec/events/platform_event_payload_guard_spec.rb
  git commit -m "feat: add versioned platform care line templates from the C2-C7 methodological notes

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 4: GraphQL do catálogo-base para o maintenance (F-23.1)

**Files:**
- Create: `app/graphql/maintenance/types/care_line_template_type.rb`, `app/graphql/maintenance/mutations/upsert_care_line_template.rb`, `spec/requests/maintenance/care_line_templates_spec.rb`
- Modify: `app/graphql/maintenance/types/query_type.rb`, `app/graphql/maintenance/types/mutation_type.rb`, `app/events/maintenance_audit.rb` (`maintenance.care_line_template.changed` em `NAMES`), `spec/events/platform_event_payload_guard_spec.rb` (o mesmo nome em `R18_PLATFORM_EVENT_NAMES`), `spec/architecture/maintenance_schema_spec.rb`

**Interfaces:**
- Consumes: `CareLineTemplate`, `CareLines::Templates::Save.call(attrs, maintainer:)` (Task 3).
- Produces (contrato §8): `query { careLineTemplates(key: String) { key version current name indicator declaredCondition eligibility practices createdAt } }` (todas as versões, por chave na ordem do catálogo e versão decrescente; `current` = maior versão da chave); `mutation upsertCareLineTemplate(key: String!, name: String!, indicator: String!, declaredCondition: String, eligibility: JSON!, practices: JSON!) { ok errors { path message } }` (`message` = código do erro; `path` = campo do `Rules`, ou `key`/`indicator`/`name`/`declaredCondition`).

- [ ] **Step 1: Teste que falha** — `spec/requests/maintenance/care_line_templates_spec.rb`:

  ```ruby
  require "rails_helper"

  # Contrato §8: catálogo-base, só plataforma; versão nova a cada gravação.
  RSpec.describe "Maintenance: catálogo-base das linhas de cuidado", type: :request do
    let(:frontend) { "https://maintenance.rotasaude.app" }
    let(:password) { "s3nha-forte-1" }
    let!(:maintainer) do
      Maintainer.create!(email_address: "linhas-#{SecureRandom.hex(3)}@rotasaude.app", password: password,
                         otp_secret: ROTP::Base32.random, otp_enabled_at: Time.current)
    end

    def browser = { "Origin" => frontend, "X-Rota-Maintenance" => "1" }
    def json = JSON.parse(response.body)
    def gql!(query, variables = {}) = post("/graphql", params: { query: query, variables: variables.to_json }, headers: browser)

    UPSERT = <<~GQL.freeze
      mutation Upsert($key: String!, $name: String!, $indicator: String!, $declaredCondition: String, $eligibility: JSON!, $practices: JSON!) {
        upsertCareLineTemplate(key: $key, name: $name, indicator: $indicator, declaredCondition: $declaredCondition,
                               eligibility: $eligibility, practices: $practices) { ok errors { path message } }
      }
    GQL
    LIST = "query { careLineTemplates(key: \"diabetes_c4\") { key version current name indicator declaredCondition practices } }".freeze

    before do
      CareLines::Templates::Load.call
      host! "maintenance-api.rotasaude.app"
      allow(ENV).to receive(:[]).and_call_original
      allow(ENV).to receive(:[]).with(MaintenanceApi::ORIGIN).and_return(frontend)
      post "/session", params: { email_address: maintainer.email_address, password: password }, headers: browser
      post "/session/challenge", params: { session_id: json["session_id"], code: ROTP::TOTP.new(maintainer.otp_secret).now }, headers: browser
    end

    def diabetes_vars(**over)
      t = CareLineTemplate.current("diabetes_c4")
      { key: t.key, name: t.name, indicator: t.indicator, declaredCondition: t.declared_condition, eligibility: t.eligibility,
        practices: t.practices }.merge(over)
    end

    it "lista as versões; upsert grava a versão 2 e audita sem abrir cidade" do
      expect(Maintenance::CityReader).not_to receive(:call)
      gql!(UPSERT, diabetes_vars(name: "Pessoa com diabetes (nota 2027)"))
      expect(json.dig("data", "upsertCareLineTemplate")).to eq("ok" => true, "errors" => [])
      gql!(LIST)
      rows = json.dig("data", "careLineTemplates")
      expect(rows.map { |r| [ r["version"], r["current"], r["name"] ] })
        .to eq([ [ 2, true, "Pessoa com diabetes (nota 2027)" ], [ 1, false, "Pessoa com diabetes" ] ])
      expect(rows.first["practices"].first["key"]).to eq("c4_a")
      audit = PlatformEvent.where(name: "maintenance.care_line_template.changed").order(:created_at).pluck(:payload)
      expect(audit.map { |p| p["outcome"] }).to eq(%w[attempted ok])
      expect(audit.last.keys).not_to include("name")
    end

    it "recusa com path e não grava" do
      bad = diabetes_vars(practices: CareLineTemplate.current("diabetes_c4").practices.map { |p| p.merge("window_days" => 0) })
      gql!(UPSERT, bad)
      expect(json.dig("data", "upsertCareLineTemplate")).to eq("ok" => false,
                                                               "errors" => [ { "path" => "practices[0].window_days", "message" => "invalid_practices" } ])
      gql!(UPSERT, diabetes_vars(indicator: "C5"))
      expect(json.dig("data", "upsertCareLineTemplate", "errors")).to eq([ { "path" => "indicator", "message" => "invalid_indicator" } ])
      expect(CareLineTemplate.where(key: "diabetes_c4").count).to eq(1)
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar.** `test spec/requests/maintenance/care_line_templates_spec.rb` → FAIL (campo `careLineTemplates` inexistente).

- [ ] **Step 3: Implemente.** `app/graphql/maintenance/types/care_line_template_type.rb`:

  ```ruby
  module Maintenance
    module Types
      class CareLineTemplateType < BaseObject
        graphql_name "CareLineTemplate"
        description "Versão do catálogo-base de uma linha de cuidado (C2–C7). Imutável; a cidade adota uma versão."

        field :key, String, null: false
        field :version, Integer, null: false
        field :current, Boolean, null: false, description: "Maior versão da chave"
        field :name, String, null: false
        field :indicator, String, null: false
        field :declared_condition, String, null: true
        field :eligibility, GraphQL::Types::JSON, null: false
        field :practices, GraphQL::Types::JSON, null: false
        field :created_at, GraphQL::Types::ISO8601DateTime, null: false

        def current = object.version == context[:care_line_current_versions]&.fetch(object.key, nil)
      end
    end
  end
  ```

  Em `query_type.rb`, depois de `lab_exam_templates`:

  ```ruby
      # ADR 0040 (contrato §8): catálogo-base das linhas de cuidado, plataforma.
      field :care_line_templates, [ Types::CareLineTemplateType ], null: false,
            description: "Versões do catálogo-base das linhas de cuidado (por chave, da mais nova)" do
        argument :key, String, required: false
      end
  ```
  e o resolver:

  ```ruby
      def care_line_templates(key: nil)
        scope = CareLineTemplate.all
        scope = scope.where(key: key) if key
        rows = scope.to_a
        context[:care_line_current_versions] = rows.group_by(&:key).transform_values { |list| list.map(&:version).max }
        order = CareLines::Catalog::TEMPLATE_KEYS
        rows.sort_by { |t| [ order.index(t.key) || order.size, -t.version ] }
      end
  ```

  `app/graphql/maintenance/mutations/upsert_care_line_template.rb`:

  ```ruby
  module Maintenance
    module Mutations
      # Contrato §8: grava a versão seguinte da linha no catálogo-base. Só
      # plataforma; sem step-up (como o catálogo do laboratório); auditado
      # pela base (attempted/ok|rejected) e por care_line.template_changed.
      class UpsertCareLineTemplate < BaseMutation
        description "Grava uma nova versão de uma linha do catálogo-base de linhas de cuidado."

        argument :key, String, required: true
        argument :name, String, required: true
        argument :indicator, String, required: true
        argument :declared_condition, String, required: false
        argument :eligibility, GraphQL::Types::JSON, required: true
        argument :practices, GraphQL::Types::JSON, required: true

        field :ok, Boolean, null: false
        field :errors, [ Types::UserErrorType ], null: false

        def resolve(**attrs)
          audited(event: "maintenance.care_line_template.changed", module_name: "care_lines", template_key: attrs[:key]) do
            result = CareLines::Templates::Save.call(attrs, maintainer: credential.maintainer)
            raise Rejected.new(result.reason.to_s, path: result.details[:field]) if result.failure?
          end
        end
      end
    end
  end
  ```
  Em `mutation_type.rb`: `field :upsert_care_line_template, mutation: Mutations::UpsertCareLineTemplate` (depois de `upsert_lab_analyte_template`). Em `maintenance_audit.rb`, `maintenance.care_line_template.changed` no fim de `NAMES`; no `R18_PLATFORM_EVENT_NAMES`, o mesmo nome depois de `maintenance.lab_template.changed`. Em `maintenance_schema_spec.rb`: `"Query"` ganha `careLineTemplates`; `"Mutation"` ganha `upsertCareLineTemplate`; novos `"CareLineTemplate" => %w[key version current name indicator declaredCondition eligibility practices createdAt]` e `"UpsertCareLineTemplatePayload" => %w[ok errors]`.

- [ ] **Step 4: Rode e veja passar.** `test spec/requests/maintenance/care_line_templates_spec.rb spec/architecture/maintenance_schema_spec.rb spec/events/platform_event_payload_guard_spec.rb spec/requests/maintenance/lab_templates_spec.rb` → PASS.

- [ ] **Step 5: Commit.**

  ```bash
  git add app/graphql/maintenance/types/care_line_template_type.rb app/graphql/maintenance/mutations/upsert_care_line_template.rb app/graphql/maintenance/types/query_type.rb app/graphql/maintenance/types/mutation_type.rb app/events/maintenance_audit.rb spec/events/platform_event_payload_guard_spec.rb spec/architecture/maintenance_schema_spec.rb spec/requests/maintenance/care_line_templates_spec.rb
  git commit -m "feat: expose care line templates and versioned upsert in the maintenance GraphQL API

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

---

## Fatia 2 — Dados da cidade e linhas

### Task 5: Migração de cidade, triggers, modelos, cifra, eventos e helpers

**Files:**
- Create: `db/city_migrate/20261022900002_add_care_lines.rb`, `app/models/care_line.rb`, `app/models/care_line_enrollment.rb`, `app/models/care_line_suggestion.rb`, `app/models/care_outreach_contact.rb`, `app/models/citizen_declared_condition.rb`, `spec/support/care_line_helpers.rb`, `spec/models/care_lines_schema_spec.rb`
- Modify: `db/city_schema.rb` (cinco tabelas, FKs, `define(version: 2026_10_22_900002)`), `db/city_triggers.sql`, `app/services/city_encryption.rb`, `config/initializers/domain_events.rb`, `config/initializers/filter_parameter_logging.rb`, `spec/initializers/domain_events_bindings_spec.rb`

**Interfaces:**
- Produces: modelos com associações `CareLine has_many :enrollments`, `CareLineEnrollment belongs_to :patient, :care_line, :team (HealthTeam), :enrolled_by_user, :ended_by_user (opt), :suggestion (opt); has_many :contacts`, `CareLineSuggestion belongs_to :patient (opt), :citizen (opt), :care_line, :team, :enrollment (opt), :decided_by_user (opt)`, `CareOutreachContact belongs_to :enrollment, :recorded_by_user`, `CitizenDeclaredCondition belongs_to :citizen`; escopos `CareLine.active_lines`, `CareLineEnrollment.active_enrollments`, `CareLineSuggestion.pending_suggestions`, `CitizenDeclaredCondition.current`; helpers `care_city!`, `care_lines_loaded!`, `adopted_line!(key)`, `care_unit!`, `care_team!(unit, *users)`, `care_doctor!(unit)`, `care_nurse!(unit)`, `care_patient!(age:, sex:, problems: [])`, `care_enrollment!(patient:, line:, team:, by:, anchor_date: nil, enrolled_at: Time.current)`.

- [ ] **Step 1: Teste que falha** — `spec/models/care_lines_schema_spec.rb`:

  ```ruby
  require "rails_helper"

  # ADR 0040 (spec §4): só acréscimo onde a spec manda; cifra; unicidades.
  RSpec.describe "Linhas de cuidado — banco da cidade" do
    before { care_city! }

    let(:unit) { care_unit! }
    let(:doctor) { care_doctor!(unit) }
    let(:team) { care_team!(unit, doctor) }
    let(:line) { adopted_line!("diabetes_c4") }
    let(:patient) { care_patient!(age: 58, sex: "male", problems: [ %w[ciap2 T90] ]) }

    def sql(statement) = ApplicationRecord.connection.execute(statement)

    it "uma inscrição ativa por paciente e linha; encerrada libera" do
      enrollment = care_enrollment!(patient: patient, line: line, team: team, by: doctor)
      expect { care_enrollment!(patient: patient, line: line, team: team, by: doctor) }.to raise_error(ActiveRecord::RecordNotUnique)
      enrollment.update!(status: "ended", ended_reason: "discharge", ended_at: Time.current, ended_by_user: doctor)
      expect(care_enrollment!(patient: patient, line: line, team: team, by: doctor)).to be_persisted
    end

    it "inscrição: só a saída muda, uma vez; nada se apaga" do
      enrollment = care_enrollment!(patient: patient, line: line, team: team, by: doctor)
      expect { sql("UPDATE care_line_enrollments SET team_id = gen_random_uuid() WHERE id = '#{enrollment.id}'") }
        .to raise_error(ActiveRecord::StatementInvalid, /care_line_enrollments/)
      enrollment.update!(status: "ended", ended_reason: "moved", ended_at: Time.current, ended_by_user: doctor)
      expect { sql("UPDATE care_line_enrollments SET ended_reason = 'death' WHERE id = '#{enrollment.id}'") }
        .to raise_error(ActiveRecord::StatementInvalid, /care_line_enrollments/)
      expect { sql("DELETE FROM care_line_enrollments WHERE id = '#{enrollment.id}'") }.to raise_error(ActiveRecord::StatementInvalid)
    end

    it "contato só acréscimo e nota cifrada" do
      enrollment = care_enrollment!(patient: patient, line: line, team: team, by: doctor)
      contact = CareOutreachContact.create!(enrollment: enrollment, channel: "phone", outcome: "no_answer", note: "Não atendeu às 10h",
                                            recorded_by_user: doctor, recorded_at: Time.current)
      raw = ApplicationRecord.connection.select_value("SELECT note FROM care_outreach_contacts WHERE id = '#{contact.id}'")
      expect(raw).not_to include("atendeu")
      expect { sql("UPDATE care_outreach_contacts SET outcome = 'reached' WHERE id = '#{contact.id}'") }.to raise_error(ActiveRecord::StatementInvalid)
      expect { sql("DELETE FROM care_outreach_contacts WHERE id = '#{contact.id}'") }.to raise_error(ActiveRecord::StatementInvalid)
    end

    it "condição declarada: cifrada, retirar uma vez, DELETE passa (exclusão LGPD)" do
      citizen = verified_citizen!(exam_seq!)
      condition = CitizenDeclaredCondition.create!(citizen: citizen, condition: "diabetes", declared_at: Time.current)
      raw = ApplicationRecord.connection.select_value("SELECT condition FROM citizen_declared_conditions WHERE id = '#{condition.id}'")
      expect(raw).not_to include("diabetes")
      condition.update!(withdrawn_at: Time.current)
      expect { sql("UPDATE citizen_declared_conditions SET withdrawn_at = NULL WHERE id = '#{condition.id}'") }
        .to raise_error(ActiveRecord::StatementInvalid, /citizen_declared_conditions/)
      expect { sql("DELETE FROM citizen_declared_conditions WHERE id = '#{condition.id}'") }.not_to raise_error
    end

    it "sugestão: pessoa é paciente OU par; uma pendente por pessoa e linha; decidida congela; DELETE passa" do
      citizen = verified_citizen!(exam_seq!)
      expect { CareLineSuggestion.create!(care_line: line, team: team, reason: "age_sex") }.to raise_error(ActiveRecord::StatementInvalid)
      one = CareLineSuggestion.create!(care_line: line, team: team, citizen: citizen, reason: "declared")
      expect { CareLineSuggestion.create!(care_line: line, team: team, citizen: citizen, reason: "declared") }
        .to raise_error(ActiveRecord::RecordNotUnique)
      one.update!(status: "dismissed", decided_by_user: doctor, decided_at: Time.current, dismiss_reason: "Já acompanhada no CAPS")
      expect { sql("UPDATE care_line_suggestions SET status = 'accepted' WHERE id = '#{one.id}'") }.to raise_error(ActiveRecord::StatementInvalid)
      expect { sql("DELETE FROM care_line_suggestions WHERE id = '#{one.id}'") }.not_to raise_error
    end

    it "linha: uma adoção por template e uma linha ativa por condição declarada" do
      line
      expect { CareLine.create!(source: "template", template_key: "diabetes_c4", template_version: 1, indicator: "C4", name: "Outra",
                                eligibility: line.eligibility, practices: line.practices, updated_by_user: doctor) }
        .to raise_error(ActiveRecord::RecordNotUnique)
      expect { CareLine.create!(source: "custom", name: "Diabetes 2", eligibility: line.eligibility, practices: line.practices,
                                declared_condition: "diabetes", updated_by_user: doctor) }.to raise_error(ActiveRecord::RecordNotUnique)
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar.** `test spec/models/care_lines_schema_spec.rb` → FAIL (`undefined method care_city!`).

- [ ] **Step 3: Migração** `db/city_migrate/20261022900002_add_care_lines.rb`:

  ```ruby
  # Módulo 23 (ADR 0040): linhas de cuidado. Linhas da cidade (adotadas do
  # catálogo-base ou próprias), inscrições (a saída muda uma vez; nada se
  # apaga), sugestões (decididas congelam; a exclusão LGPD apaga), contatos de
  # busca ativa (só acréscimo, nota cifrada) e condições declaradas pelo
  # cidadão (só acréscimo; retirar grava withdrawn_at; cifradas; a exclusão
  # apaga). Triggers em db/city_triggers.sql. Irreversível.
  class AddCareLines < ActiveRecord::Migration[8.1]
    def up
      create_table :care_lines, id: :uuid do |t|
        t.string :source, null: false
        t.string :template_key
        t.integer :template_version
        t.string :indicator, limit: 2
        t.string :name, limit: 120, null: false
        t.jsonb :eligibility, null: false
        t.jsonb :practices, null: false
        t.string :declared_condition
        t.boolean :active, null: false, default: true
        t.references :updated_by_user, type: :uuid, null: false, foreign_key: { to_table: :users }, index: false
        t.timestamps
      end
      add_index :care_lines, :template_key, unique: true, where: "source = 'template'", name: "idx_care_lines_one_adoption"
      add_index :care_lines, :declared_condition, unique: true, where: "active AND declared_condition IS NOT NULL",
                                                   name: "idx_care_lines_one_declared"
      add_check_constraint :care_lines, "source::text = ANY (ARRAY['template'::text, 'custom'::text]) AND " \
                                        "((source::text = 'template') = (template_key IS NOT NULL AND template_version IS NOT NULL AND indicator IS NOT NULL)) AND " \
                                        "(declared_condition IS NULL OR declared_condition::text = ANY (ARRAY['diabetes'::text, 'hypertension'::text, 'pregnancy'::text, 'mental_health'::text])) AND " \
                                        "jsonb_typeof(practices) = 'array'::text AND jsonb_typeof(eligibility) = 'object'::text",
                           name: "ck_care_lines_values"

      create_table :care_line_suggestions, id: :uuid do |t|
        t.references :patient, type: :uuid, foreign_key: true, index: false
        t.references :citizen, type: :uuid, foreign_key: true, index: false
        t.references :care_line, type: :uuid, null: false, foreign_key: true, index: false
        t.references :team, type: :uuid, null: false, foreign_key: { to_table: :health_teams }, index: false
        t.uuid :enrollment_id
        t.string :reason, null: false
        t.string :evidence_ref, limit: 60
        t.string :status, null: false, default: "pending"
        t.references :decided_by_user, type: :uuid, foreign_key: { to_table: :users }, index: false
        t.datetime :decided_at
        t.text :dismiss_reason
        t.datetime :created_at, null: false, default: -> { "CURRENT_TIMESTAMP" }
      end
      add_index :care_line_suggestions, %i[team_id status created_at], name: "idx_care_line_suggestions_team"
      add_index :care_line_suggestions, %i[care_line_id patient_id], unique: true, where: "status = 'pending' AND patient_id IS NOT NULL",
                                                                     name: "idx_care_line_suggestions_one_patient"
      add_index :care_line_suggestions, %i[care_line_id citizen_id], unique: true, where: "status = 'pending' AND citizen_id IS NOT NULL",
                                                                     name: "idx_care_line_suggestions_one_citizen"
      add_index :care_line_suggestions, :citizen_id, name: "idx_care_line_suggestions_citizen"
      add_index :care_line_suggestions, :patient_id, name: "idx_care_line_suggestions_patient"
      add_check_constraint :care_line_suggestions, "num_nonnulls(patient_id, citizen_id) = 1 AND " \
                                                   "reason::text = ANY (ARRAY['problem'::text, 'age_sex'::text, 'declared'::text, 'resolved_problem'::text]) AND " \
                                                   "((reason::text = 'resolved_problem') = (enrollment_id IS NOT NULL)) AND " \
                                                   "(reason::text <> 'resolved_problem' OR patient_id IS NOT NULL) AND " \
                                                   "status::text = ANY (ARRAY['pending'::text, 'accepted'::text, 'dismissed'::text]) AND " \
                                                   "((status::text = 'pending') = (decided_at IS NULL AND decided_by_user_id IS NULL)) AND " \
                                                   "((status::text = 'dismissed') = (dismiss_reason IS NOT NULL))",
                           name: "ck_care_line_suggestions_values"

      create_table :care_line_enrollments, id: :uuid do |t|
        t.references :patient, type: :uuid, null: false, foreign_key: true, index: false
        t.references :care_line, type: :uuid, null: false, foreign_key: true, index: false
        t.references :team, type: :uuid, null: false, foreign_key: { to_table: :health_teams }, index: false
        t.references :enrolled_by_user, type: :uuid, null: false, foreign_key: { to_table: :users }, index: false
        t.references :suggestion, type: :uuid, foreign_key: { to_table: :care_line_suggestions }, index: { unique: true }
        t.datetime :enrolled_at, null: false
        t.string :source, null: false
        t.text :anchor_date
        t.string :status, null: false, default: "active"
        t.string :ended_reason
        t.text :ended_note
        t.datetime :ended_at
        t.references :ended_by_user, type: :uuid, foreign_key: { to_table: :users }, index: false
        t.timestamps
      end
      add_index :care_line_enrollments, %i[patient_id care_line_id], unique: true, where: "status = 'active'",
                                                                     name: "idx_care_line_enrollments_one_active"
      add_index :care_line_enrollments, %i[team_id status enrolled_at], name: "idx_care_line_enrollments_team"
      add_index :care_line_enrollments, :patient_id, name: "idx_care_line_enrollments_patient"
      add_check_constraint :care_line_enrollments, "source::text = ANY (ARRAY['suggestion'::text, 'manual'::text]) AND " \
                                                   "((source::text = 'suggestion') = (suggestion_id IS NOT NULL)) AND " \
                                                   "status::text = ANY (ARRAY['active'::text, 'ended'::text]) AND " \
                                                   "((status::text = 'ended') = (ended_reason IS NOT NULL AND ended_at IS NOT NULL AND ended_by_user_id IS NOT NULL)) AND " \
                                                   "(status::text = 'ended' OR ended_note IS NULL) AND " \
                                                   "(ended_reason IS NULL OR ended_reason::text = ANY (ARRAY['discharge'::text, 'death'::text, 'moved'::text, 'pregnancy_ended'::text, 'other'::text]))",
                           name: "ck_care_line_enrollments_values"
      add_foreign_key :care_line_suggestions, :care_line_enrollments, column: :enrollment_id

      create_table :care_outreach_contacts, id: :uuid do |t|
        t.references :enrollment, type: :uuid, null: false, foreign_key: { to_table: :care_line_enrollments }, index: false
        t.string :channel, null: false
        t.string :outcome, null: false
        t.text :note
        t.references :recorded_by_user, type: :uuid, null: false, foreign_key: { to_table: :users }, index: false
        t.datetime :recorded_at, null: false
      end
      add_index :care_outreach_contacts, %i[enrollment_id recorded_at], name: "idx_care_outreach_contacts_enrollment"
      add_check_constraint :care_outreach_contacts, "channel::text = ANY (ARRAY['phone'::text, 'home_visit'::text, 'wpda_message'::text, 'in_person'::text]) AND " \
                                                    "outcome::text = ANY (ARRAY['reached'::text, 'no_answer'::text, 'scheduled'::text, 'refused'::text])",
                           name: "ck_care_outreach_contacts_values"

      create_table :citizen_declared_conditions, id: :uuid do |t|
        t.references :citizen, type: :uuid, null: false, foreign_key: true, index: true
        t.text :condition, null: false
        t.datetime :declared_at, null: false
        t.datetime :withdrawn_at
      end

      execute File.read(Rails.root.join("db/city_triggers.sql"))
    end

    def down
      raise ActiveRecord::IrreversibleMigration
    end
  end
  ```

  Triggers, no fim de `db/city_triggers.sql`:

  ```sql
  -- care_line_enrollments (ADR 0040; spec §4): a inscrição é ato clínico. Só a
  -- saída muda, uma vez (active -> ended com motivo, data, quem e nota
  -- opcional); nada se apaga (retém na exclusão LGPD).
  CREATE OR REPLACE FUNCTION rota_care_line_enrollment_guard() RETURNS trigger AS $fn$
  BEGIN
    IF TG_OP = 'DELETE' THEN
      RAISE EXCEPTION 'care_line_enrollments: rows are never deleted';
    END IF;
    IF NEW.id IS DISTINCT FROM OLD.id OR NEW.patient_id IS DISTINCT FROM OLD.patient_id
       OR NEW.care_line_id IS DISTINCT FROM OLD.care_line_id OR NEW.team_id IS DISTINCT FROM OLD.team_id
       OR NEW.enrolled_by_user_id IS DISTINCT FROM OLD.enrolled_by_user_id OR NEW.suggestion_id IS DISTINCT FROM OLD.suggestion_id
       OR NEW.enrolled_at IS DISTINCT FROM OLD.enrolled_at OR NEW.source IS DISTINCT FROM OLD.source
       OR NEW.anchor_date IS DISTINCT FROM OLD.anchor_date OR NEW.created_at IS DISTINCT FROM OLD.created_at THEN
      RAISE EXCEPTION 'care_line_enrollments: only the ending columns change';
    END IF;
    IF OLD.status = 'ended' AND (NEW.status, NEW.ended_reason, NEW.ended_note, NEW.ended_at, NEW.ended_by_user_id)
       IS DISTINCT FROM (OLD.status, OLD.ended_reason, OLD.ended_note, OLD.ended_at, OLD.ended_by_user_id) THEN
      RAISE EXCEPTION 'care_line_enrollments: an ended enrollment is frozen';
    END IF;
    RETURN NEW;
  END;
  $fn$ LANGUAGE plpgsql;

  -- care_line_suggestions: pending -> accepted | dismissed, uma vez; decidida
  -- congela. DELETE passa (exclusão LGPD do par).
  CREATE OR REPLACE FUNCTION rota_care_line_suggestion_guard() RETURNS trigger AS $fn$
  BEGIN
    IF (NEW.id, NEW.patient_id, NEW.citizen_id, NEW.care_line_id, NEW.team_id, NEW.enrollment_id, NEW.reason, NEW.evidence_ref, NEW.created_at)
       IS DISTINCT FROM (OLD.id, OLD.patient_id, OLD.citizen_id, OLD.care_line_id, OLD.team_id, OLD.enrollment_id, OLD.reason, OLD.evidence_ref, OLD.created_at) THEN
      RAISE EXCEPTION 'care_line_suggestions: only the decision columns change';
    END IF;
    IF OLD.status <> 'pending' THEN
      RAISE EXCEPTION 'care_line_suggestions: a decided suggestion is frozen';
    END IF;
    RETURN NEW;
  END;
  $fn$ LANGUAGE plpgsql;

  -- care_outreach_contacts: só acréscimo.
  CREATE OR REPLACE FUNCTION rota_care_outreach_contact_append_only() RETURNS trigger AS $fn$
  BEGIN
    RAISE EXCEPTION 'care_outreach_contacts: append-only (% refused)', TG_OP;
  END;
  $fn$ LANGUAGE plpgsql;

  -- citizen_declared_conditions: só acréscimo; retirar grava withdrawn_at uma
  -- vez. DELETE passa (a exclusão LGPD apaga e não retém).
  CREATE OR REPLACE FUNCTION rota_citizen_declared_condition_guard() RETURNS trigger AS $fn$
  BEGIN
    IF (NEW.id, NEW.citizen_id, NEW.condition, NEW.declared_at) IS DISTINCT FROM (OLD.id, OLD.citizen_id, OLD.condition, OLD.declared_at)
       OR OLD.withdrawn_at IS NOT NULL OR NEW.withdrawn_at IS NULL THEN
      RAISE EXCEPTION 'citizen_declared_conditions: only withdrawn_at, once';
    END IF;
    RETURN NEW;
  END;
  $fn$ LANGUAGE plpgsql;

  DO $do$
  BEGIN
    IF to_regclass('public.care_line_enrollments') IS NOT NULL THEN
      EXECUTE 'DROP TRIGGER IF EXISTS care_line_enrollments_guard ON care_line_enrollments';
      EXECUTE 'CREATE TRIGGER care_line_enrollments_guard BEFORE UPDATE OR DELETE ON care_line_enrollments
        FOR EACH ROW EXECUTE FUNCTION rota_care_line_enrollment_guard()';
      EXECUTE 'DROP TRIGGER IF EXISTS care_line_suggestions_guard ON care_line_suggestions';
      EXECUTE 'CREATE TRIGGER care_line_suggestions_guard BEFORE UPDATE ON care_line_suggestions
        FOR EACH ROW EXECUTE FUNCTION rota_care_line_suggestion_guard()';
      EXECUTE 'DROP TRIGGER IF EXISTS care_outreach_contacts_append_only ON care_outreach_contacts';
      EXECUTE 'CREATE TRIGGER care_outreach_contacts_append_only BEFORE UPDATE OR DELETE ON care_outreach_contacts
        FOR EACH ROW EXECUTE FUNCTION rota_care_outreach_contact_append_only()';
      EXECUTE 'DROP TRIGGER IF EXISTS citizen_declared_conditions_guard ON citizen_declared_conditions';
      EXECUTE 'CREATE TRIGGER citizen_declared_conditions_guard BEFORE UPDATE ON citizen_declared_conditions
        FOR EACH ROW EXECUTE FUNCTION rota_citizen_declared_condition_guard()';
    END IF;
  END
  $do$;
  ```

  `db/city_schema.rb`: `define(version: 2026_10_22_900002)` e os cinco `create_table` na ordem alfabética (`care_line_enrollments`, `care_line_suggestions`, `care_lines`, `care_outreach_contacts` depois de `campaigns`; `citizen_declared_conditions` depois de `citizen_contact_preferences`), com colunas em ordem alfabética, índices e checks **exatamente** como a migração os cria, e os `add_foreign_key` (`care_line_enrollments` → `care_line_suggestions` (`suggestion_id`), `care_lines`, `health_teams` (`team_id`), `patients`, `users` (`enrolled_by_user_id`, `ended_by_user_id`); `care_line_suggestions` → `care_line_enrollments` (`enrollment_id`), `care_lines`, `citizens`, `health_teams` (`team_id`), `patients`, `users` (`decided_by_user_id`); `care_lines` → `users` (`updated_by_user_id`); `care_outreach_contacts` → `care_line_enrollments` (`enrollment_id`), `users` (`recorded_by_user_id`); `citizen_declared_conditions` → `citizens`). A paridade é provada por `spec/services/city_schema_spec.rb` (migra um banco descartável e compara com o dump): rode-a no Step 6 e copie para o dump o que ela mostrar de diferente.

- [ ] **Step 4: Modelos.**

  ```ruby
  # app/models/care_line.rb
  # Linha de cuidado da cidade (ADR 0040; spec §4): adotada do catálogo-base
  # (cópia das regras da versão adotada) ou própria, com as mesmas formas.
  class CareLine < ApplicationRecord
    belongs_to :updated_by_user, class_name: "User"
    has_many :enrollments, class_name: "CareLineEnrollment", dependent: :restrict_with_error

    scope :active_lines, -> { where(active: true) }

    def template? = source == "template"
    def requires_lmp? = eligibility["requires_anchor"] == "lmp"
  end
  ```

  ```ruby
  # app/models/care_line_enrollment.rb
  # Inscrição confirmada por profissional (ADR 0040). anchor_date (DUM) e a
  # nota de encerramento cifradas com a chave da cidade.
  class CareLineEnrollment < ApplicationRecord
    encrypts :anchor_date, :ended_note

    belongs_to :patient
    belongs_to :care_line
    belongs_to :team, class_name: "HealthTeam"
    belongs_to :enrolled_by_user, class_name: "User"
    belongs_to :ended_by_user, class_name: "User", optional: true
    belongs_to :suggestion, class_name: "CareLineSuggestion", optional: true
    has_many :contacts, -> { order(recorded_at: :desc, id: :desc) }, class_name: "CareOutreachContact", foreign_key: :enrollment_id,
                                                                     dependent: :restrict_with_error, inverse_of: :enrollment

    scope :active_enrollments, -> { where(status: "active") }

    def active? = status == "active"
    def anchor_on = anchor_date.present? ? Date.iso8601(anchor_date) : nil
  end
  ```

  ```ruby
  # app/models/care_line_suggestion.rb
  # Sugestão (ADR 0040): de inscrição (problem | age_sex | declared) ou de
  # encerramento (resolved_problem). Sem texto clínico; o motivo da recusa é
  # cifrado. Só a equipe atribuída a vê.
  class CareLineSuggestion < ApplicationRecord
    encrypts :dismiss_reason

    belongs_to :patient, optional: true
    belongs_to :citizen, optional: true
    belongs_to :care_line
    belongs_to :team, class_name: "HealthTeam"
    belongs_to :enrollment, class_name: "CareLineEnrollment", optional: true
    belongs_to :decided_by_user, class_name: "User", optional: true

    scope :pending_suggestions, -> { where(status: "pending") }

    def pending? = status == "pending"
    def closure? = reason == "resolved_problem"
  end
  ```

  ```ruby
  # app/models/care_outreach_contact.rb
  # Tentativa de contato da busca ativa (ADR 0040). Só acréscimo; nota cifrada.
  class CareOutreachContact < ApplicationRecord
    encrypts :note

    belongs_to :enrollment, class_name: "CareLineEnrollment", inverse_of: :contacts
    belongs_to :recorded_by_user, class_name: "User"
  end
  ```

  ```ruby
  # app/models/citizen_declared_condition.rb
  # Condição de saúde declarada pelo próprio cidadão no wpda (ADR 0040; spec
  # §4). Nunca é diagnóstico. Cifrada; retirar grava withdrawn_at; a exclusão
  # LGPD apaga (não retém).
  class CitizenDeclaredCondition < ApplicationRecord
    encrypts :condition

    belongs_to :citizen

    scope :current, -> { where(withdrawn_at: nil) }

    validates :condition, inclusion: { in: CareLines::Catalog::CONDITIONS }
  end
  ```

  `app/services/city_encryption.rb`, no fim de `CITY_KEYED_TARGETS` (antes do `].freeze`):

  ```ruby
      # ADR 0040: DUM, notas e motivos das linhas de cuidado; condição declarada.
      [ CareLineEnrollment, :anchor_date ],
      [ CareLineEnrollment, :ended_note ],
      [ CareLineSuggestion, :dismiss_reason ],
      [ CareOutreachContact, :note ],
      [ CitizenDeclaredCondition, :condition ],
  ```

  `config/initializers/filter_parameter_logging.rb`, no fim da lista:

  ```ruby
    # ADR 0040: condição declarada pelo cidadão, DUM e motivo da recusa da sugestão
    # (`:note` e `:reason` já cobrem a nota do contato e o motivo do encerramento).
    /\A(condition|anchor_date|dismiss_reason)\z/
  ```
  (troque o `]` final da entrada anterior por `,` antes dela).

  `config/initializers/domain_events.rb` (antes do fim do bloco):

  ```ruby
    # ADR 0040 (contrato §9): linhas de cuidado, só ids. O problema que muda e a
    # condição declarada disparam a sugestão daquela pessoa (Task 10).
    DomainEvents.bind "care_line.adopted", to: []
    DomainEvents.bind "care_line_suggestion.created", to: []
    DomainEvents.bind "care_line_suggestion.accepted", to: []
    DomainEvents.bind "care_line_suggestion.dismissed", to: []
    DomainEvents.bind "care_line_enrollment.created", to: []
    DomainEvents.bind "care_line_enrollment.ended", to: []
    DomainEvents.bind "care_outreach.recorded", to: []
    DomainEvents.bind "citizen_condition.withdrawn", to: []
  ```
  (`citizen_condition.declared` e `patient_problem.changed` ganham consumidor na Task 10; não os declare aqui.)

  `spec/support/care_line_helpers.rb`:

  ```ruby
  # Módulo 23 (ADR 0040): cidade com o prontuário e care_lines, catálogo-base,
  # linha adotada, equipe, profissionais e pacientes com problema ativo.
  # Cenário de teste: os caminhos reais são CareLines::SaveLine, Enroll e o
  # 19a (Consultations::Finalize) para o problema.
  module CareLineHelpers
    def care_city!
      city = clinical_city!
      cid10_release!
      ciap2_release!(ScreeningHelpers::CIAP2.merge("W78" => "Gravidez", "K87" => "Hipertensão com complicações",
                                                   "W82" => "Aborto espontâneo", "W18" => "Puerpério",
                                                   "W14" => "Contracepção, outros"))
      Platform::Features.set!(city: city, key: "care_lines", enabled: true, maintainer: ledi_maintainer!)
      CityCatalog.reset_cache!
      city
    end

    def care_lines_loaded! = (CareLineTemplate.exists? || CareLines::Templates::Load.call)

    def care_admin! = (@care_admin ||= staff_with("admin-linhas-#{SecureRandom.hex(3)}@cidade.gov.br", "municipal_admin"))

    def adopted_line!(key)
      care_lines_loaded!
      CareLine.find_by(template_key: key) || begin
        t = CareLineTemplate.current(key)
        CareLine.create!(source: "template", template_key: key, template_version: t.version, indicator: t.indicator, name: t.name,
                         eligibility: t.eligibility, practices: t.practices, declared_condition: t.declared_condition,
                         updated_by_user: care_admin!)
      end
    end

    def care_unit!(name = "UBS Linhas #{SecureRandom.hex(2)}") = create_unit(name)
    def care_doctor!(unit) = doctor!(unit) # 225125
    def care_nurse!(unit) = screener!(unit, cbo: "223505")

    def care_team!(unit, *users)
      team = HealthTeam.create!(health_unit: unit, ine: format("%010d", SecureRandom.random_number(10**10)), kind: "70",
                                name: "ESF #{unit.name}")
      users.each do |user|
        link = user.professional.links.active.find_by!(health_unit: unit)
        HealthTeamMember.create!(professional: user.professional, health_team: team, cbo_code: link.cbo_code, started_on: Time.zone.today - 30)
      end
      team
    end

    # Paciente (caminho real: par validado + Patients::Resolve) com problemas
    # ativos gravados direto (evento + estado, como Patients::ApplyProblemEvent
    # exige pelo trigger, sem consulta: addendum/consulta nulos não passam, por
    # isso o problema nasce de uma consulta finalizada mínima quando pedido).
    def care_patient!(age:, sex:, problems: [], unit: nil, doctor: nil, gender_identity: nil)
      patient, citizen = exam_patient!(exam_seq!, age: age, sex: sex)
      citizen.update!(gender_identity: gender_identity) if gender_identity
      Patients::Resolve.refresh!(patient)
      return patient.reload if problems.empty?

      unit ||= care_unit!
      doctor ||= care_doctor!(unit)
      evaluated = problems.map { |terminology, code| { "terminology" => terminology, "code" => code, "action" => "add" } }
      finalized_consultation!(unit: unit, doctor: doctor, citizen: citizen, evaluated_problems: evaluated, exam_requests: [])
      patient.reload
    end

    def care_enrollment!(patient:, line:, team:, by:, anchor_date: nil, enrolled_at: Time.current)
      CareLineEnrollment.create!(patient: patient, care_line: line, team: team, enrolled_by_user: by, enrolled_at: enrolled_at,
                                 source: "manual", anchor_date: anchor_date&.iso8601)
    end
  end

  RSpec.configure { |c| c.include CareLineHelpers }
  ```

  `spec/initializers/domain_events_bindings_spec.rb`, no fim:

  ```ruby
  # Módulo 23 (ADR 0040; contrato §9): trilha das linhas de cuidado, só ids.
  RSpec.describe "care lines bindings (F-23.3, F-23.5, F-23.7)" do
    it "declara os eventos das linhas de cuidado" do
      %w[care_line.adopted care_line_suggestion.created care_line_suggestion.accepted care_line_suggestion.dismissed
         care_line_enrollment.created care_line_enrollment.ended care_outreach.recorded citizen_condition.withdrawn].each do |name|
        expect(DomainEvents.registry.key?(name)).to be(true), name
      end
    end
  end
  ```

- [ ] **Step 5: Rode e veja passar.** `test spec/models/care_lines_schema_spec.rb spec/initializers/domain_events_bindings_spec.rb spec/architecture/city_encrypted_attributes_guard_spec.rb` → PASS.

- [ ] **Step 6: Paridade do dump.** `test spec/services/city_schema_spec.rb` → PASS (se falhar, ajuste `db/city_schema.rb` para o que a spec mostra; nunca o contrário).

- [ ] **Step 7: Commit.**

  ```bash
  git add db/city_migrate/20261022900002_add_care_lines.rb db/city_schema.rb db/city_triggers.sql app/models/care_line.rb app/models/care_line_enrollment.rb app/models/care_line_suggestion.rb app/models/care_outreach_contact.rb app/models/citizen_declared_condition.rb app/services/city_encryption.rb config/initializers/domain_events.rb config/initializers/filter_parameter_logging.rb spec/initializers/domain_events_bindings_spec.rb spec/support/care_line_helpers.rb spec/models/care_lines_schema_spec.rb
  git commit -m "feat: add city tables for care lines, enrollments, suggestions, outreach and declared conditions

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 6: Linhas da cidade — adoção, linha própria, listas de apoio (F-23.2)

**Files:**
- Create: `app/controllers/care_lines/base_controller.rb`, `app/controllers/care_lines/admin_controller.rb`, `app/controllers/care_lines/lookups_controller.rb`, `app/commands/care_lines/save_line.rb`, `app/services/care_lines/json.rb`, `app/services/care_lines/access.rb`, `spec/requests/care_lines/admin_spec.rb`
- Modify: `config/routes.rb`

**Interfaces:**
- Consumes: `CareLineTemplate.{current,latest}`, `CareLines::Rules`, `CareLines::Catalog` (Tasks 2–3); `CareLine` (Task 5).
- Produces: `CareLines::BaseController` (sessão do dashboard, `CareLines::Gate.usable?` → 403 `feature_disabled`; `render_failure(result)` com `field`; `forbid`, `not_found`, `uuid?`); `CareLines::Access.{admin?(user), preview?(user), professional_unit_ids(user), panel?(user, team), confirmer?(user, team), visible_teams(user)}`; `CareLines::SaveLine.call(params:, by:, line: nil) → Result` (`payload[:line]`; falhas `invalid_name`, `invalid_practices`, `invalid_eligibility` (com `field`), `invalid_declared_condition`, `template_not_found`, `duplicate_template`, `duplicate_declared_condition`, `immutable_rules`); `CareLines::Json.{line(line), template(t)}`; rotas `GET /care_lines/admin/templates`, `GET|POST /care_lines/admin/lines`, `PATCH /care_lines/admin/lines/:id`, `GET /care_lines/lines`, `GET /care_lines/teams` (L10).

- [ ] **Step 1: Teste que falha** — `spec/requests/care_lines/admin_spec.rb`:

  ```ruby
  require "rails_helper"

  # Contrato §3: admin municipal; escrita com step-up; 422 com os códigos.
  RSpec.describe "Linhas de cuidado — linhas da cidade", type: :request do
    before do
      care_city!
      care_lines_loaded!
    end

    let(:admin) { care_admin! }
    let(:mental) do
      { name: "Saúde mental — acompanhamento", declared_condition: "mental_health",
        eligibility: { problems: { ciap2: %w[P74 P76], cid10: %w[F32 F33 F41] }, declared: %w[mental_health] },
        practices: [ { key: "sm_a", label: "Consulta a cada 3 meses", citizen_label: "Consulta de acompanhamento", source: "consultation",
                       anchor: "rolling", window_days: 92, min_count: 1, schedule: "end_of_window", cbo_filter: %w[2251 2235 2515] } ] }
    end

    it "templates: as seis linhas, versão mais nova; profissional não lê" do
      sign_in_as(admin)
      get "/care_lines/admin/templates"
      expect(json_body["templates"].map { |t| [ t["key"], t["version"], t["indicator"] ] })
        .to eq(CareLines::Catalog::TEMPLATE_KEYS.map { |k| [ k, 1, CareLines::Catalog::INDICATORS[k] ] })
      sign_in_as(care_doctor!(care_unit!))
      get "/care_lines/admin/templates"
      expect(response).to have_http_status(:forbidden)
    end

    it "adota com step-up (cópia da versão), recusa adoção repetida, publica só ids" do
      sign_in_as(admin)
      json_post "/care_lines/admin/lines", template_key: "diabetes_c4", template_version: 1
      expect(json_body).to eq("error" => "mfa_required")
      stepped_up!(admin)
      json_post "/care_lines/admin/lines", template_key: "diabetes_c4", template_version: 1
      expect(response).to have_http_status(:created)
      expect(json_body).to include("source" => "template", "template_key" => "diabetes_c4", "template_version" => 1, "indicator" => "C4",
                                   "declared_condition" => "diabetes", "active" => true)
      expect(json_body["practices"].map { |p| p["key"] }).to eq(%w[c4_a c4_b c4_c c4_d c4_e c4_f])
      expect(DomainEvent.where(name: "care_line.adopted").last.payload.keys).to match_array(%w[care_line_id user_id])
      json_post "/care_lines/admin/lines", template_key: "diabetes_c4", template_version: 1
      expect(response).to have_http_status(:unprocessable_entity)
      expect(json_body).to eq("error" => "duplicate_template")
      json_post "/care_lines/admin/lines", template_key: "elderly_c6", template_version: 9
      expect(json_body).to eq("error" => "template_not_found")
    end

    it "versão nova do template só vale quando a cidade adota (PATCH template_version)" do
      stepped_up!(admin)
      json_post "/care_lines/admin/lines", template_key: "hypertension_c5", template_version: 1
      line = CareLine.find(json_body["id"])
      t = CareLineTemplate.current("hypertension_c5")
      CareLines::Templates::Save.call(t.attributes.slice(*%w[key indicator declared_condition eligibility])
                                       .merge("name" => "Hipertensão (2027)", "practices" => t.practices.map { |p| p.merge("window_days" => p["window_days"] + 1) }))
      expect(line.reload.practices.first["window_days"]).to eq(183)
      patch "/care_lines/admin/lines/#{line.id}", params: { template_version: 2 }.to_json, headers: { "CONTENT_TYPE" => "application/json" }
      expect(response).to have_http_status(:ok)
      expect(line.reload).to have_attributes(template_version: 2, name: "Hipertensão (2027)")
      expect(line.practices.first["window_days"]).to eq(184)
      patch "/care_lines/admin/lines/#{line.id}", params: { practices: [] }.to_json, headers: { "CONTENT_TYPE" => "application/json" }
      expect(json_body).to eq("error" => "immutable_rules")
    end

    it "linha própria: cria, recusa regra inválida com field, nome longo e condição já usada; desativa" do
      stepped_up!(admin)
      json_post "/care_lines/admin/lines", mental
      expect(response).to have_http_status(:created)
      expect(json_body).to include("source" => "custom", "template_key" => nil, "indicator" => nil, "declared_condition" => "mental_health")
      id = json_body["id"]
      json_post "/care_lines/admin/lines", mental.merge(name: "Outra saúde mental")
      expect(json_body).to eq("error" => "duplicate_declared_condition")
      bad = mental.merge(declared_condition: nil, practices: [ mental[:practices].first.merge(source: "lab:1") ])
      json_post "/care_lines/admin/lines", bad
      expect(json_body).to eq("error" => "invalid_practices", "field" => "practices[0].source")
      json_post "/care_lines/admin/lines", mental.merge(declared_condition: nil, name: "x" * 121)
      expect(json_body).to eq("error" => "invalid_name")
      json_post "/care_lines/admin/lines", mental.merge(declared_condition: nil, eligibility: {})
      expect(json_body).to eq("error" => "invalid_eligibility", "field" => "eligibility")
      patch "/care_lines/admin/lines/#{id}", params: { active: false }.to_json, headers: { "CONTENT_TYPE" => "application/json" }
      expect(json_body["active"]).to be(false)
      get "/care_lines/admin/lines"
      expect(json_body["lines"].map { |l| l["id"] }).to eq([ id ])
    end

    it "listas de apoio: linhas ativas e equipes visíveis (profissional: as das unidades dele)" do
      unit = care_unit!
      doctor = care_doctor!(unit)
      mine = care_team!(unit, doctor)
      other = care_team!(care_unit!)
      adopted_line!("elderly_c6")
      sign_in_as(doctor)
      get "/care_lines/lines"
      expect(json_body["lines"].map { |l| l["name"] }).to eq([ "Pessoa idosa" ])
      get "/care_lines/teams"
      expect(json_body["teams"].map { |t| t["id"] }).to eq([ mine.id ])
      sign_in_as(admin)
      get "/care_lines/teams"
      expect(json_body["teams"].map { |t| t["id"] }).to contain_exactly(mine.id, other.id)
    end

    it "care_lines desligado: 403 feature_disabled" do
      Platform::Features.set!(city: Current.city, key: "care_lines", enabled: false, maintainer: ledi_maintainer!)
      sign_in_as(admin)
      get "/care_lines/admin/lines"
      expect(response).to have_http_status(:forbidden)
      expect(json_body).to eq("error" => "feature_disabled", "feature" => "care_lines")
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar.** `test spec/requests/care_lines/admin_spec.rb` → FAIL (rota inexistente).

- [ ] **Step 3: Implemente.** `app/controllers/care_lines/base_controller.rb`:

  ```ruby
  # Base das rotas /care_lines/* (ADR 0040; contrato §3–§6): sessão do
  # dashboard e care_lines ligado e utilizável (403 feature_disabled).
  module CareLines
    class BaseController < ApplicationController
      include Authentication
      include MfaStepUp

      wrap_parameters false

      UUID = /\A\h{8}-\h{4}-\h{4}-\h{4}-\h{12}\z/
      FAILURE_STATUS = { invalid_transition: :conflict, already_enrolled: :conflict, citizen_not_verified: :conflict,
                         line_inactive: :conflict, cbo_not_allowed: :forbidden, not_found: :not_found }.freeze

      before_action :require_care_lines!

      private

      def require_care_lines!
        return if Gate.usable?(Current.city)

        render json: { error: "feature_disabled", feature: Gate::KEY }, status: :forbidden
      end

      def render_failure(result)
        payload = { error: result.reason.to_s }
        payload[:field] = result.details[:field] if result.details[:field]
        render json: payload, status: FAILURE_STATUS.fetch(result.reason, :unprocessable_entity)
      end

      def render_error(reason, status = :unprocessable_entity) = render(json: { error: reason.to_s }, status: status)
      def forbid(reason = "missing_role") = render(json: { error: reason }, status: :forbidden)
      def not_found = render(json: { error: "not_found" }, status: :not_found)
      def uuid?(value) = value.is_a?(String) && value.match?(UUID)
      def body = params.to_unsafe_h.except("controller", "action", "id")
    end
  end
  ```

  `app/services/care_lines/access.rb`:

  ```ruby
  # Quem faz o quê nas linhas de cuidado (ADR 0040; spec §2, decisões do
  # coordenador; contrato §3–§6). Painel e contato: health_professional com
  # vínculo ativo na unidade da equipe. Confirmar: esse vínculo com CBO de
  # médico (225*) ou enfermeiro (2235*). Linhas: municipal_admin. Prévia:
  # municipal_admin ou analyst.
  module CareLines
    module Access
      module_function

      def admin?(user) = user.has_role?("municipal_admin")
      def preview?(user) = admin?(user) || user.has_role?("analyst")

      def professional_links(user)
        return ProfessionalLink.none unless user.has_role?("health_professional") && user.professional

        user.professional.links.active
      end

      def professional_unit_ids(user) = professional_links(user).distinct.pluck(:health_unit_id)

      def panel?(user, team) = team.present? && professional_unit_ids(user).include?(team.health_unit_id)

      def confirmer?(user, team)
        return false unless team

        professional_links(user).where(health_unit_id: team.health_unit_id).pluck(:cbo_code)
                                .any? { |cbo| Catalog::CONFIRM_CBO_PREFIXES.any? { |prefix| cbo.start_with?(prefix) } }
      end

      # Equipes que aparecem nos filtros: as das unidades do profissional; a
      # gestão (admin, analyst) vê todas as ativas.
      def visible_teams(user)
        scope = HealthTeam.where(active: true).includes(:health_unit).order(:name, :ine)
        preview?(user) ? scope : scope.where(health_unit_id: professional_unit_ids(user))
      end
    end
  end
  ```

  `app/commands/care_lines/save_line.rb`:

  ```ruby
  # Linhas da cidade (contrato §3). Adotar copia as regras da versão do
  # catálogo-base; PATCH { template_version } readota (Desvio 12). Linha
  # própria: as mesmas formas, validadas pelo Rules. Uma adoção por template
  # e uma linha ativa por condição declarada (índices únicos).
  module CareLines
    module SaveLine
      CUSTOM_KEYS = %w[name eligibility practices declared_condition active].freeze

      module_function

      def call(params:, by:, line: nil)
        params = params.to_h.stringify_keys
        if line.nil? && params.key?("template_key") then adopt(params, by)
        elsif line.nil? then custom(CareLine.new(source: "custom"), params, by)
        elsif line.template? then readopt(line, params, by)
        else custom(line, params, by)
        end
      rescue ActiveRecord::RecordNotUnique => e
        Result.fail(e.message.include?("idx_care_lines_one_adoption") ? :duplicate_template : :duplicate_declared_condition)
      end

      def adopt(params, by)
        template = CareLineTemplate.find_by(key: params["template_key"].to_s, version: params["template_version"])
        return Result.fail(:template_not_found) unless template

        line = CareLine.new(source: "template", template_key: template.key, active: true)
        copy!(line, template, by)
        DomainEvents.publish("care_line.adopted", care_line_id: line.id, user_id: by.id)
        Result.ok(line: line, created: true)
      end

      def readopt(line, params, by)
        return Result.fail(:immutable_rules) if (params.keys & %w[name eligibility practices declared_condition]).any?

        if params.key?("template_version")
          template = CareLineTemplate.find_by(key: line.template_key, version: params["template_version"])
          return Result.fail(:template_not_found) unless template

          copy!(line, template, by)
          DomainEvents.publish("care_line.adopted", care_line_id: line.id, user_id: by.id)
        end
        line.update!(active: params["active"] == true, updated_by_user: by) if params.key?("active")
        Result.ok(line: line, created: false)
      end

      def custom(line, params, by)
        attrs = params.slice(*CUSTOM_KEYS)
        if line.new_record? || attrs.key?("name")
          name = attrs["name"].to_s
          return Result.fail(:invalid_name) unless name.strip == name && name.length.between?(1, 120)
        end
        if line.new_record? || attrs.key?("eligibility")
          eligibility = Rules.eligibility(attrs["eligibility"])
          return eligibility if eligibility.failure?

          attrs["eligibility"] = eligibility.payload[:eligibility]
        end
        if line.new_record? || attrs.key?("practices")
          practices = Rules.practices(attrs["practices"])
          return practices if practices.failure?

          attrs["practices"] = practices.payload[:practices]
        end
        condition = attrs["declared_condition"]
        return Result.fail(:invalid_declared_condition) unless condition.nil? || (Catalog::CONDITIONS - %w[none]).include?(condition)

        attrs["active"] = attrs["active"] == true if attrs.key?("active")
        created = line.new_record?
        line.assign_attributes(attrs.merge("updated_by_user" => by))
        line.save!
        Result.ok(line: line, created: created)
      end

      def copy!(line, template, by)
        line.assign_attributes(template_version: template.version, indicator: template.indicator, name: template.name,
                               eligibility: template.eligibility, practices: template.practices,
                               declared_condition: template.declared_condition, updated_by_user: by)
        line.save!
      end
      private_class_method :adopt, :readopt, :custom, :copy!
    end
  end
  ```

  `app/services/care_lines/json.rb` (as formas do contrato §2; as do painel entram na Task 12):

  ```ruby
  module CareLines
    module Json
      module_function

      def line(line)
        { id: line.id, name: line.name, source: line.source, template_key: line.template_key, template_version: line.template_version,
          indicator: line.indicator, eligibility: line.eligibility, practices: line.practices, declared_condition: line.declared_condition,
          active: line.active }
      end

      def template(template)
        { key: template.key, version: template.version, name: template.name, indicator: template.indicator,
          declared_condition: template.declared_condition, eligibility: template.eligibility, practices: template.practices }
      end

      def line_summary(line) = { id: line.id, name: line.name, indicator: line.indicator, requires_anchor: line.eligibility["requires_anchor"] }

      def team(team)
        { id: team.id, ine: team.ine, name: team.name, unit: { id: team.health_unit_id, name: team.health_unit.name } }
      end
    end
  end
  ```

  `app/controllers/care_lines/admin_controller.rb`:

  ```ruby
  # /care_lines/admin (contrato §3): municipal_admin; escrita com step-up.
  module CareLines
    class AdminController < BaseController
      before_action { forbid unless Access.admin?(Current.user) }
      before_action :require_step_up!, only: %i[create update]

      def templates = render(json: { templates: CareLineTemplate.latest.map { |t| Json.template(t) } })

      def index = render(json: { lines: CareLine.order(:name, :id).map { |l| Json.line(l) } })

      def create
        result = SaveLine.call(params: body, by: Current.user)
        return render_failure(result) if result.failure?

        render json: Json.line(result.payload[:line]), status: :created
      end

      def update
        line = uuid?(params[:id]) && CareLine.find_by(id: params[:id])
        return not_found unless line

        result = SaveLine.call(params: body, by: Current.user, line: line)
        return render_failure(result) if result.failure?

        render json: Json.line(result.payload[:line])
      end
    end
  end
  ```

  `app/controllers/care_lines/lookups_controller.rb` (L10):

  ```ruby
  # Listas de apoio do dashboard (Divergência L10): linhas ativas (nome e se
  # pede DUM) e equipes visíveis para os filtros.
  module CareLines
    class LookupsController < BaseController
      before_action { forbid unless Access.preview?(Current.user) || Access.professional_unit_ids(Current.user).any? }

      def lines = render(json: { lines: CareLine.active_lines.order(:name, :id).map { |l| Json.line_summary(l) } })
      def teams = render(json: { teams: Access.visible_teams(Current.user).map { |t| Json.team(t) } })
    end
  end
  ```

  `config/routes.rb` (perto das rotas `/billing`):

  ```ruby
  # Módulo 23 (ADR 0040; contrato §3–§6): linhas de cuidado, sessão do dashboard.
  scope "/care_lines", module: "care_lines", as: "care_lines" do
    get   "admin/templates",           to: "admin#templates"
    get   "admin/lines",               to: "admin#index"
    post  "admin/lines",               to: "admin#create"
    patch "admin/lines/:id",           to: "admin#update"
    get   "lines",                     to: "lookups#lines"
    get   "teams",                     to: "lookups#teams"
  end
  ```

- [ ] **Step 4: Rode e veja passar.** `test spec/requests/care_lines/admin_spec.rb` → PASS.

- [ ] **Step 5: Commit.**

  ```bash
  git add app/controllers/care_lines/base_controller.rb app/controllers/care_lines/admin_controller.rb app/controllers/care_lines/lookups_controller.rb app/commands/care_lines/save_line.rb app/services/care_lines/json.rb app/services/care_lines/access.rb config/routes.rb spec/requests/care_lines/admin_spec.rb
  git commit -m "feat: let the municipal admin adopt template care lines and create custom ones

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

---

## Fatia 3 — Acompanhamento na leitura

### Task 7: Avaliação pura das boas práticas (`CareLines::Evaluate`) (F-23.4)

**Files:**
- Create: `app/services/care_lines/event.rb`, `app/services/care_lines/person.rb`, `app/services/care_lines/practice_status.rb`, `app/services/care_lines/evaluate.rb`, `spec/services/care_lines/evaluate_spec.rb`

**Interfaces:**
- Consumes: `CareLines::Catalog.untracked?`, `Catalog::DUE_SOON_DAYS` (Task 2); regras normalizadas (Task 2/3).
- Produces:
  - `CareLines::Event = Data.define(:on, :cbo, :codes)` (`on: Date`, `cbo: String|nil`, `codes: Array<String>` — CIAP/CID da consulta; `inspect` só com a data).
  - `CareLines::Person = Data.define(:patient_id, :citizen_id, :birth_date, :sex, :gender_identity, :problems, :resolved_problems, :declared, :team_id)` (todos com padrão; `problems`/`resolved_problems` = `Array<[terminology, code]>`; `declared` = `Array<String>`; `age_years(on)`, `age_days(on)`; `inspect` só com ids).
  - `CareLines::PracticeStatus = Data.define(:key, :label, :citizen_label, :source, :tracked, :status, :done_at, :due_on, :count, :expected, :met, :points)`.
  - `CareLines::Evaluate.call(practices:, person:, anchors:, events:, today:) → Array<PracticeStatus>` (só as práticas que se aplicam à pessoa; `anchors` = `{ "enrollment" => Date|nil, "birth" => Date|nil, "lmp" => Date|nil }`; `events` = `{ "consultation" => [Event], "blood_pressure" => [...], "weight_height" => [...], "exam:<código>" => [...], "procedure:<código>" => [...] }`); `CareLines::Evaluate.worst(statuses) → "ok"|"due_soon"|"overdue"|"untracked"`; `CareLines::Evaluate.applies?(rule, person, on) → Boolean`; `CareLines::Evaluate.sources_of(practice) → Array<String>` (fontes rastreadas da prática, com as do `all_of`); `CareLines::Evaluate.since(practices, anchors, today) → Date` (início da janela mais antiga).

- [ ] **Step 1: Teste que falha** — `spec/services/care_lines/evaluate_spec.rb` (séries conhecidas; hoje = 22/10/2026; contas conferidas ao escrever o plano):

  ```ruby
  require "rails_helper"

  RSpec.describe CareLines::Evaluate do
    let(:today) { Date.new(2026, 10, 22) }
    let(:entries) { CareLines::Templates::Load.entries.index_by { |e| e["key"] } }

    def practices(key) = CareLines::Rules.practices(entries.fetch(key)["practices"], template: true).payload[:practices]
    def ev(on, cbo: nil, codes: []) = CareLines::Event.new(on: Date.iso8601(on), cbo: cbo, codes: codes)
    def by_key(list) = list.index_by(&:key)
    def person(**attrs) = CareLines::Person.new(**attrs)

    it "C4 (janela móvel): consulta em dia, PA vencendo, HbA1c fora da janela, pés em dia; técnico não conta; ACS não rastreado" do
      events = {
        "consultation" => [ ev("2026-06-01", cbo: "225125"), ev("2026-10-01", cbo: "322205") ],
        "blood_pressure" => [ ev("2026-05-10") ], "weight_height" => [ ev("2026-06-01") ],
        "exam:0202010503" => [ ev("2025-09-01") ], "procedure:0301040095" => [ ev("2026-01-15") ]
      }
      result = by_key(described_class.call(practices: practices("diabetes_c4"), person: person(birth_date: Date.new(1968, 3, 1), sex: "male"),
                                           anchors: { "enrollment" => today }, events: events, today: today))
      expect(result["c4_a"]).to have_attributes(status: "ok", done_at: Date.new(2026, 6, 1), due_on: Date.new(2026, 12, 1), count: 1, met: true)
      expect(result["c4_b"]).to have_attributes(status: "due_soon", due_on: Date.new(2026, 11, 9), met: true)
      expect(result["c4_c"]).to have_attributes(status: "ok", due_on: Date.new(2027, 6, 1))
      expect(result["c4_d"]).to have_attributes(status: "untracked", tracked: false, met: false, due_on: nil)
      expect(result["c4_e"]).to have_attributes(status: "overdue", done_at: nil, due_on: nil, count: 0, met: false)
      expect(result["c4_f"]).to have_attributes(status: "ok", due_on: Date.new(2027, 1, 15))
      expect(described_class.worst(result.values)).to eq("overdue")
    end

    it "C2 (nascimento, proporcional): 1ª consulta até 30 dias; 2 de 9 aos 150 dias em dia; 1 peso+altura atrasado" do
      birth = Date.new(2026, 5, 25)
      events = { "consultation" => [ ev("2026-06-10", cbo: "223505"), ev("2026-08-01", cbo: "225125") ],
                 "weight_height" => [ ev("2026-06-10") ] }
      result = by_key(described_class.call(practices: practices("child_c2"), person: person(birth_date: birth, sex: "male"),
                                           anchors: { "birth" => birth, "enrollment" => today }, events: events, today: today))
      expect(result["c2_a"]).to have_attributes(status: "ok", done_at: Date.new(2026, 6, 10), due_on: nil, met: true)
      expect(result["c2_b"]).to have_attributes(status: "due_soon", count: 2, expected: 2, due_on: Date.new(2026, 11, 4), met: true)
      expect(result["c2_c"]).to have_attributes(status: "overdue", count: 1, expected: 2, due_on: Date.new(2026, 8, 15), met: false)
      expect(result.values_at("c2_d", "c2_e").map(&:status)).to eq(%w[untracked untracked])
    end

    it "C3 (DUM): 1ª consulta ok; 7 consultas atrasadas; 1º trimestre com 2 de 4 grupos vencendo; 3º trimestre ainda não começou" do
      lmp = Date.new(2026, 8, 13)
      events = { "consultation" => [ ev("2026-09-20", cbo: "223505", codes: %w[W78]) ], "blood_pressure" => [ ev("2026-09-20") ],
                 "weight_height" => [ ev("2026-09-20") ], "procedure:0214010074" => [ ev("2026-09-20") ],
                 "exam:0202030300" => [ ev("2026-09-20") ] }
      result = by_key(described_class.call(practices: practices("pregnancy_c3"), person: person(birth_date: Date.new(1999, 1, 2), sex: "female"),
                                           anchors: { "lmp" => lmp, "enrollment" => today }, events: events, today: today))
      expect(result["c3_a"]).to have_attributes(status: "ok", met: true)
      expect(result["c3_b"]).to have_attributes(status: "overdue", count: 1, expected: 2, due_on: Date.new(2026, 9, 25), met: false)
      expect(result["c3_g"]).to have_attributes(status: "due_soon", count: 2, expected: 4, due_on: Date.new(2026, 11, 18), met: false)
      expect(result["c3_h"]).to have_attributes(status: "ok", count: 0, due_on: Date.new(2027, 6, 3), met: false)
      expect(result["c3_i"]).to have_attributes(status: "ok", due_on: Date.new(2027, 7, 15))
      expect(result.values_at("c3_e", "c3_f", "c3_j", "c3_k").map(&:tracked)).to eq([ false, false, false, false ])
    end

    it "C7: só as práticas da faixa; HPV molecular vale 60 meses; saúde sexual exige o código" do
      events = { "exam:0202100251" => [ ev("2023-01-10") ], "exam:0203010086" => [ ev("2022-05-01") ],
                 "consultation" => [ ev("2026-03-01", cbo: "223505", codes: %w[W14]), ev("2026-09-01", cbo: "225125", codes: %w[R05]) ] }
      woman = person(birth_date: Date.new(1996, 2, 2), sex: "female")
      result = by_key(described_class.call(practices: practices("women_c7"), person: woman, anchors: { "birth" => woman.birth_date },
                                           events: events, today: today))
      expect(result.keys).to eq(%w[c7_a c7_c])
      expect(result["c7_a"]).to have_attributes(status: "ok", done_at: Date.new(2023, 1, 10), due_on: Date.new(2028, 1, 10), met: true)
      expect(result["c7_c"]).to have_attributes(status: "ok", done_at: Date.new(2026, 3, 1), due_on: Date.new(2027, 3, 1))
    end

    it "não rastreado nunca piora; sem nada rastreado a pessoa fica untracked" do
      only_untracked = practices("elderly_c6").select { |p| CareLines::Catalog.untracked?(p["source"]) }
      result = described_class.call(practices: only_untracked, person: person(birth_date: Date.new(1950, 1, 1)), anchors: {}, events: {}, today: today)
      expect(described_class.worst(result)).to eq("untracked")
      mixed = practices("elderly_c6")
      result = described_class.call(practices: mixed, person: person(birth_date: Date.new(1950, 1, 1)), anchors: {},
                                    events: { "consultation" => [ ev("2026-10-01", cbo: "225125") ], "weight_height" => [ ev("2026-10-01") ] },
                                    today: today)
      expect(described_class.worst(result)).to eq("ok")
    end

    it "sem âncora (gestante sem DUM): rastreadas ficam overdue sem data, nunca cumpridas" do
      result = by_key(described_class.call(practices: practices("pregnancy_c3"), person: person, anchors: {}, events: {}, today: today))
      expect(result["c3_a"]).to have_attributes(status: "overdue", due_on: nil, met: false)
    end

    it "since: início da janela mais antiga entre as rastreadas" do
      expect(described_class.since(practices("women_c7"), { "birth" => Date.new(1996, 2, 2) }, today)).to eq(today - 1825)
      expect(described_class.since(practices("pregnancy_c3"), { "lmp" => Date.new(2026, 8, 13) }, today)).to eq(Date.new(2026, 8, 13))
    end

    it "inspect de Person e Event não mostra dado pessoal nem código" do
      p = person(patient_id: "p1", birth_date: Date.new(1990, 1, 1), sex: "female", problems: [ %w[ciap2 T90] ])
      expect([ p.inspect, p.to_s ].join).not_to match(/1990|female|T90/)
      expect(ev("2026-01-01", codes: %w[W78]).inspect).not_to include("W78")
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar.** `test spec/services/care_lines/evaluate_spec.rb` → FAIL (`uninitialized constant CareLines::Event`).

- [ ] **Step 3: Implemente.** `app/services/care_lines/event.rb`:

  ```ruby
  # Um registro datado de uma fonte (consulta, sinais, exame, procedimento) —
  # só a data, o CBO e os códigos da consulta; nunca valor clínico.
  module CareLines
    Event = Data.define(:on, :cbo, :codes) do
      def initialize(on:, cbo: nil, codes: []) = super
      def inspect = "#<CareLines::Event on=#{on}>"
      alias_method :to_s, :inspect
      def pretty_print(q) = q.text(inspect)
    end
  end
  ```

  `app/services/care_lines/person.rb`:

  ```ruby
  # A pessoa avaliada (paciente ou par validado), com o que a elegibilidade e
  # as faixas precisam, decifrado na leitura. inspect só com ids (contrato,
  # convenção: Data com dado pessoal nunca mostra o dado).
  module CareLines
    Person = Data.define(:patient_id, :citizen_id, :birth_date, :sex, :gender_identity, :problems, :resolved_problems, :declared, :team_id) do
      def initialize(patient_id: nil, citizen_id: nil, birth_date: nil, sex: nil, gender_identity: nil, problems: [],
                     resolved_problems: [], declared: [], team_id: nil) = super

      def age_years(on) = birth_date && Citizen.age_between(birth_date, on)
      def age_days(on) = birth_date && (on - birth_date).to_i
      def key = patient_id ? "patient:#{patient_id}" : "citizen:#{citizen_id}"
      def inspect = "#<CareLines::Person patient_id=#{patient_id.inspect} citizen_id=#{citizen_id.inspect}>"
      alias_method :to_s, :inspect
      def pretty_print(q) = q.text(inspect)
    end
  end
  ```

  `app/services/care_lines/practice_status.rb`:

  ```ruby
  module CareLines
    PracticeStatus = Data.define(:key, :label, :citizen_label, :source, :tracked, :status, :done_at, :due_on, :count, :expected, :met, :points)
  end
  ```

  `app/services/care_lines/evaluate.rb`:

  ```ruby
  # Avaliação de cada boa prática (ADR 0040; spec §5; plano, Desvios 2–3).
  # PURA: recebe as regras, a pessoa, as âncoras e os eventos já carregados;
  # devolve PracticeStatus. Fonte fora do sistema → untracked (nunca atraso).
  module CareLines
    module Evaluate
      RANK = { "ok" => 0, "due_soon" => 1, "overdue" => 2 }.freeze

      module_function

      def call(practices:, person:, anchors:, events:, today:)
        practices.filter_map do |practice|
          next unless applies?(practice["applies_to"], person, today)

          evaluate(practice, anchors, events, today)
        end
      end

      def worst(statuses)
        tracked = statuses.select(&:tracked)
        return "untracked" if tracked.empty?

        tracked.map(&:status).max_by { |status| RANK.fetch(status) }
      end

      def applies?(rule, person, on)
        return true if rule.nil?

        age = person.age_years(on)
        if rule["age_min_years"] || rule["age_max_years"]
          return false if age.nil?
          return false if rule["age_min_years"] && age < rule["age_min_years"]
          return false if rule["age_max_years"] && age > rule["age_max_years"]
        end
        rule["sex"].nil? || person.sex == rule["sex"]
      end

      def sources_of(practice)
        return [] if Catalog.untracked?(practice["source"])

        groups(practice).flatten.map { |alt| alt["source"] }.uniq
      end

      def since(practices, anchors, today)
        starts = practices.reject { |p| Catalog.untracked?(p["source"]) }.filter_map do |p|
          if p["anchor"] == "rolling"
            today - (groups(p).flatten.map { |alt| alt["window_days"] || p["window_days"] }.max - 1)
          elsif (anchor = anchors[p["anchor"]])
            anchor + p["window_start_days"].to_i
          end
        end
        starts.min || today
      end

      def evaluate(practice, anchors, events, today)
        base = { key: practice["key"], label: practice["label"], citizen_label: practice["citizen_label"], source: practice["source"],
                 points: practice["points"] }
        if Catalog.untracked?(practice["source"])
          return PracticeStatus.new(**base, tracked: false, status: "untracked", done_at: nil, due_on: nil, count: nil, expected: nil, met: false)
        end
        return rolling(practice, events, today, base) if practice["anchor"] == "rolling"

        anchor = anchors[practice["anchor"]]
        return PracticeStatus.new(**base, tracked: true, status: "overdue", done_at: nil, due_on: nil, count: 0, expected: practice["min_count"], met: false) unless anchor

        fixed(practice, anchor, events, today, base)
      end

      # Janela móvel: cada alternativa com a própria janela (HPV 60 meses).
      def rolling(practice, events, today, base)
        matched = groups(practice).map do |group|
          group.flat_map do |alt|
            window = alt["window_days"] || practice["window_days"]
            dates(practice, alt["source"], events, today - (window - 1), today).map { |d| [ d, d + window ] }
          end
        end
        need = matched.size > 1 ? 1 : practice["min_count"]
        met = matched.all? { |items| items.size >= need }
        done_at = matched.flatten(1).map(&:first).max
        count, expected = counts(matched, need)
        return PracticeStatus.new(**base, tracked: true, status: "overdue", done_at: done_at, due_on: nil, count: count, expected: expected, met: false) unless met

        due_on = matched.map { |items| items.map(&:last).sort.reverse[need - 1] }.min
        status = (due_on - today).to_i <= Catalog::DUE_SOON_DAYS ? "due_soon" : "ok"
        PracticeStatus.new(**base, tracked: true, status: status, done_at: done_at, due_on: due_on, count: count, expected: expected, met: true)
      end

      # Âncora fixa: [âncora + início, âncora + fim].
      def fixed(practice, anchor, events, today, base)
        start = anchor + practice["window_start_days"].to_i
        finish = anchor + practice["window_days"]
        matched = groups(practice).map do |group|
          group.flat_map { |alt| dates(practice, alt["source"], events, start, [ finish, today ].min) }
        end
        done_at = matched.flatten.max
        return proportional(practice, matched.first, start, finish, today, base, done_at) if practice["schedule"] == "proportional"

        need = matched.size > 1 ? 1 : practice["min_count"]
        count, expected = counts(matched, need)
        met = matched.all? { |items| items.size >= need }
        status, due_on = if met then [ "ok", nil ]
                         elsif today > finish then [ "overdue", finish ]
                         elsif (finish - today).to_i <= Catalog::DUE_SOON_DAYS then [ "due_soon", finish ]
                         else [ "ok", finish ]
                         end
        PracticeStatus.new(**base, tracked: true, status: status, done_at: done_at, due_on: due_on, count: count, expected: expected, met: met)
      end

      # Esperado hoje = ceil(min × decorrido / duração); próximo marco =
      # início + floor(feitas × duração / min) + 1 (spec, decisão do coordenador).
      def proportional(practice, items, start, finish, today, base, done_at)
        min = practice["min_count"]
        span = (finish - start).to_i
        elapsed = (today - start).to_i.clamp(0, span)
        expected = (min * elapsed + span - 1) / span
        count = items.size
        if count >= min
          return PracticeStatus.new(**base, tracked: true, status: "ok", done_at: done_at, due_on: nil, count: count, expected: min, met: true)
        end

        milestone = [ start + (count * span / min) + 1, finish ].min
        status, due_on = if today > finish || milestone <= today then [ "overdue", today > finish ? finish : milestone ]
                         elsif (milestone - today).to_i <= Catalog::DUE_SOON_DAYS then [ "due_soon", milestone ]
                         else [ "ok", milestone ]
                         end
        PracticeStatus.new(**base, tracked: true, status: status, done_at: done_at, due_on: due_on, count: count, expected: expected,
                           met: count >= expected)
      end

      def groups(practice) = practice["all_of"] || [ [ { "source" => practice["source"] } ] ]

      def counts(matched, need)
        matched.size > 1 ? [ matched.count { |items| items.size >= need }, matched.size ] : [ matched.first.size, need ]
      end

      # Datas de uma fonte na janela; consulta filtra CBO (4 primeiros dígitos
      # ou mais) e código (prefixo), só nela.
      def dates(practice, source, events, from, to)
        list = events.fetch(source, [])
        if source == "consultation"
          cbos = practice["cbo_filter"]
          codes = practice["code_filter"]&.values&.flatten
          list = list.select { |e| cbos.nil? || cbos.any? { |prefix| e.cbo.to_s.start_with?(prefix) } }
          list = list.select { |e| e.codes.any? { |code| codes.any? { |prefix| code.start_with?(prefix) } } } if codes
        end
        list.map(&:on).select { |d| d >= from && d <= to }.uniq.sort
      end
      private_class_method :evaluate, :rolling, :fixed, :proportional, :groups, :counts, :dates
    end
  end
  ```

  (Observação: `groups` é chamado por `sources_of` e `since`, que são públicos — por isso fica privado só pelo `private_class_method`, que não impede a chamada interna sem receptor.)

- [ ] **Step 4: Rode e veja passar.** `test spec/services/care_lines/evaluate_spec.rb` → PASS.

- [ ] **Step 5: Commit.**

  ```bash
  git add app/services/care_lines/event.rb app/services/care_lines/person.rb app/services/care_lines/practice_status.rb app/services/care_lines/evaluate.rb spec/services/care_lines/evaluate_spec.rb
  git commit -m "feat: evaluate care line practices at read time with rolling, fixed and proportional windows

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 8: Leitura em lote das fontes (`CareLines::Sources`, `CareLines::Evaluation`) (F-23.4)

**Files:**
- Create: `app/services/care_lines/sources.rb`, `app/services/care_lines/evaluation.rb`, `spec/services/care_lines/sources_spec.rb`, `spec/services/care_lines/evaluation_spec.rb`
- Modify: `spec/support/care_line_helpers.rb` (`care_screening!`)

**Interfaces:**
- Consumes: `CareLines::{Event,Person,Evaluate}` (Task 7); `Consultation`, `ConsultationProblem`, `Screening`/`ScreeningRevision`, `Attendance`, `Citizen`, `ExamOrder`, `ExamResult`, `PerformedProcedure` (existentes).
- Produces: `CareLines::Sources.load(patient_ids:, sources:, since:) → Hash{patient_id => Hash{source => Array<Event>}}` (no máximo 6 consultas SQL, qualquer que seja o número de pacientes); `CareLines::Sources::BP_PROCEDURE = "0301100039"`, `WH_PROCEDURE = "0101040024"`; `CareLines::Evaluation::Row = Data.define(:key, :practices, :worst)`; `CareLines::Evaluation.enrollments(enrollments, today:) → Hash{enrollment_id => Row}`; `CareLines::Evaluation.items(items, today:) → Hash{key => Row}` (item = `{ key:, practices:, person:, anchors: }`, para a prévia); helper `care_screening!(unit:, citizen:, by:, vitals:)`.

- [ ] **Step 1: Teste que falha.** `spec/services/care_lines/sources_spec.rb`:

  ```ruby
  require "rails_helper"

  # ADR 0040 (spec §5): fontes existentes, sem cópia; plano, Desvio 9.
  RSpec.describe CareLines::Sources do
    before { care_city! }

    let(:unit) { care_unit! }
    let(:doctor) { care_doctor!(unit) }
    let(:nurse) { care_nurse!(unit) }
    let(:all) { %w[consultation blood_pressure weight_height exam:0202010503 procedure:0301040095] }

    it "consulta (CBO e códigos), PA e peso+altura da consulta e da escuta, exame pedido/resultado e procedimento" do
      patient = care_patient!(age: 58, sex: "male", problems: [ %w[ciap2 T90] ], unit: unit, doctor: doctor)
      citizen = Citizen.find_by!(patient_id: patient.id)
      care_screening!(unit: unit, citizen: citizen, by: nurse, vitals: { "systolic" => 150, "diastolic" => 95 })
      exam_order!(patient: patient, unit: unit, requester: doctor, sigtap_code: "0202010503", status: :awaiting_authorization)
      cancelled = exam_order!(patient: patient, unit: unit, requester: doctor, sigtap_code: "0202010503", status: :cancelled)
      raw_procedure!(unit: unit, patient: patient, professional: doctor, sigtap_code: "0301040095", cid10: nil)
      today = Time.zone.today

      events = described_class.load(patient_ids: [ patient.id ], sources: all, since: today - 365).fetch(patient.id)
      expect(events["consultation"].map { |e| [ e.on, e.cbo, e.codes ] }).to eq([ [ today, "225125", %w[T90] ] ])
      expect(events["blood_pressure"].map(&:on)).to eq([ today, today ]) # consulta (130/85) e escuta (150/95)
      expect(events["weight_height"].map(&:on)).to eq([ today ])         # só a consulta tem peso e altura
      expect(events["exam:0202010503"].map(&:on)).to eq([ today ])        # o cancelado não conta
      expect(events["procedure:0301040095"].map(&:on)).to eq([ today ])
      expect(cancelled.status).to eq("cancelled")
    end

    it "fora da janela (since) não vem; paciente sem nada vem vazio" do
      patient = care_patient!(age: 40, sex: "female")
      expect(described_class.load(patient_ids: [ patient.id ], sources: all, since: Time.zone.today)).to eq(patient.id => {})
    end

    it "uma consulta por fonte: o número de SQL não cresce com o número de pacientes" do
      patients = Array.new(5) { care_patient!(age: 50, sex: "female", problems: [ %w[ciap2 K86] ], unit: unit, doctor: doctor) }
      count = lambda do |ids|
        queries = 0
        callback = ->(*, payload) { queries += 1 unless payload[:name].in?(%w[SCHEMA TRANSACTION]) || payload[:sql].start_with?("SAVEPOINT", "RELEASE") }
        ActiveSupport::Notifications.subscribed(callback, "sql.active_record") do
          described_class.load(patient_ids: ids, sources: all, since: Time.zone.today - 365)
        end
        queries
      end
      expect(count.call(patients.first(1).map(&:id))).to eq(count.call(patients.map(&:id)))
      expect(count.call(patients.map(&:id))).to be <= 6
    end
  end
  ```

  `spec/services/care_lines/evaluation_spec.rb`:

  ```ruby
  require "rails_helper"

  RSpec.describe CareLines::Evaluation do
    before { care_city! }

    let(:unit) { care_unit! }
    let(:doctor) { care_doctor!(unit) }
    let(:team) { care_team!(unit, doctor) }

    it "avalia uma página de inscrições: diabético com consulta e PA hoje, HbA1c e pés vencidos" do
      patient = care_patient!(age: 58, sex: "male", problems: [ %w[ciap2 T90] ], unit: unit, doctor: doctor)
      enrollment = care_enrollment!(patient: patient, line: adopted_line!("diabetes_c4"), team: team, by: doctor)
      row = described_class.enrollments([ enrollment ], today: Time.zone.today).fetch(enrollment.id)
      statuses = row.practices.index_by(&:key)
      expect(statuses.transform_values(&:status)).to eq("c4_a" => "ok", "c4_b" => "ok", "c4_c" => "ok", "c4_d" => "untracked",
                                                        "c4_e" => "overdue", "c4_f" => "overdue")
      expect(row.worst).to eq("overdue")
    end

    it "gestante: âncora pela DUM cifrada da inscrição; criança: âncora pelo nascimento do paciente" do
      mom = care_patient!(age: 27, sex: "female", problems: [ %w[ciap2 W78] ], unit: unit, doctor: doctor)
      lmp = Time.zone.today - 70
      pregnant = care_enrollment!(patient: mom, line: adopted_line!("pregnancy_c3"), team: team, by: doctor, anchor_date: lmp)
      baby = care_patient!(age: 0, sex: "male")
      child = care_enrollment!(patient: baby, line: adopted_line!("child_c2"), team: team, by: doctor)
      rows = described_class.enrollments([ pregnant, child ], today: Time.zone.today)
      expect(rows[pregnant.id].practices.find { |p| p.key == "c3_a" }).to have_attributes(status: "ok", met: true)
      expect(rows[pregnant.id].practices.find { |p| p.key == "c3_h" }.due_on).to eq(lmp + 294)
      expect(rows[child.id].practices.find { |p| p.key == "c2_a" }.due_on).to eq(Date.iso8601(baby.birth_date) + 30)
      raw = ApplicationRecord.connection.select_value("SELECT anchor_date FROM care_line_enrollments WHERE id = '#{pregnant.id}'")
      expect(raw).not_to include(lmp.iso8601)
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar.** `test spec/services/care_lines/sources_spec.rb spec/services/care_lines/evaluation_spec.rb` → FAIL (`uninitialized constant CareLines::Sources`).

- [ ] **Step 3: Implemente.** Em `spec/support/care_line_helpers.rb`, dentro do módulo:

  ```ruby
    # Escuta concluída (caminho real: Screenings::Start/Complete) com os sinais dados.
    def care_screening!(unit:, citizen:, by:, vitals:)
      attendance = walk_in_attendance!(unit, citizen: citizen)
      started = Screenings::Start.call(attendance: attendance, by: by).payload[:screening]
      result = Screenings::Complete.call(screening: started, revision_params: revision_params("vitals" => vitals), destination: "oriented",
                                         destination_params: { "orientation_note" => "retorno" }, by: by)
      raise "escuta recusada: #{result.reason}" if result.failure?

      started.reload
    end
  ```

  `app/services/care_lines/sources.rb`:

  ```ruby
  # Leitura em lote das fontes das boas práticas (ADR 0040; spec §5; plano,
  # Desvio 9): só datas, CBO e códigos — nunca valor clínico. No máximo uma
  # consulta SQL por fonte, qualquer que seja o número de pacientes: pares,
  # consultas, problemas das consultas, escutas, exames (pedido + resultado
  # num UNION ALL) e procedimentos. Datas no fuso da cidade (Time.zone).
  module CareLines
    module Sources
      BP_PROCEDURE = "0301100039".freeze
      WH_PROCEDURE = "0101040024".freeze
      VITALS = %w[blood_pressure weight_height].freeze

      module_function

      def load(patient_ids:, sources:, since:)
        ids = patient_ids.uniq
        result = ids.to_h { |id| [ id, Hash.new { |h, k| h[k] = [] } ] }
        return {} if ids.empty?

        sources = sources.uniq
        from = since.in_time_zone.beginning_of_day
        vitals = sources.intersect?(VITALS)
        if sources.include?("consultation") || vitals
          consultations!(result, ids, from, sources)
        end
        screenings!(result, ids, from, sources) if vitals
        exams!(result, ids, since, sources.grep(/\Aexam:/).map { |s| s.delete_prefix("exam:") })
        procedure_codes = sources.grep(/\Aprocedure:/).map { |s| s.delete_prefix("procedure:") }
        procedure_codes << BP_PROCEDURE if sources.include?("blood_pressure")
        procedure_codes << WH_PROCEDURE if sources.include?("weight_height")
        procedures!(result, ids, since, procedure_codes.uniq)
        result.transform_values { |by_source| by_source.to_h { |source, list| [ source, list.sort_by(&:on) ] } }
      end

      def consultations!(result, ids, from, sources)
        rows = Consultation.finalized_consultations.where(patient_id: ids).where(finalized_at: from..)
                           .pluck(:id, :patient_id, :finalized_at, :cbo_code, :systolic, :weight_kg, :height_cm)
        codes = if sources.include?("consultation") && rows.any?
                  ConsultationProblem.where(consultation_id: rows.map(&:first)).pluck(:consultation_id, :code)
                                     .group_by(&:first).transform_values { |pairs| pairs.map(&:last).uniq }
                else
                  {}
                end
        rows.each do |id, patient_id, at, cbo, systolic, weight, height|
          on = at.in_time_zone.to_date
          result[patient_id]["consultation"] << Event.new(on: on, cbo: cbo, codes: codes.fetch(id, [])) if sources.include?("consultation")
          result[patient_id]["blood_pressure"] << Event.new(on: on) if systolic && sources.include?("blood_pressure")
          result[patient_id]["weight_height"] << Event.new(on: on) if weight && height && sources.include?("weight_height")
        end
      end

      # A escuta é do par (atendimento.citizen_id); o par leva ao paciente.
      def screenings!(result, ids, from, sources)
        pairs = Citizen.where(patient_id: ids).pluck(:id, :patient_id).to_h
        return if pairs.empty?

        Screening.completed_screenings.joins(:attendance, :current_revision)
                 .where(attendances: { citizen_id: pairs.keys }).where(completed_at: from..)
                 .pluck("attendances.citizen_id", "screenings.completed_at", "screening_revisions.systolic",
                        "screening_revisions.weight_kg", "screening_revisions.height_cm")
                 .each do |citizen_id, at, systolic, weight, height|
          patient_id = pairs.fetch(citizen_id)
          on = at.in_time_zone.to_date
          result[patient_id]["blood_pressure"] << Event.new(on: on) if systolic && sources.include?("blood_pressure")
          result[patient_id]["weight_height"] << Event.new(on: on) if weight && height && sources.include?("weight_height")
        end
      end

      # "Solicitado ou avaliado": pedido (não cancelado nem negado) ou resultado.
      def exams!(result, ids, since, codes)
        return if codes.empty?

        from = since.in_time_zone.beginning_of_day.utc # created_at é UTC sem fuso; released_on é data
        sql = ApplicationRecord.sanitize_sql([ <<~SQL, Time.zone.tzinfo.name, ids, codes, from, ids, codes, since ])
          SELECT o.patient_id, o.sigtap_code, (o.created_at AT TIME ZONE 'UTC' AT TIME ZONE ?)::date AS on_date
            FROM exam_orders o
           WHERE o.patient_id IN (?) AND o.sigtap_code IN (?) AND o.status NOT IN ('cancelled', 'denied') AND o.created_at >= ?
          UNION ALL
          SELECT r.patient_id, r.sigtap_code, r.released_on AS on_date
            FROM exam_results r
           WHERE r.patient_id IN (?) AND r.sigtap_code IN (?) AND r.released_on >= ?
        SQL
        ApplicationRecord.connection.select_all(sql).cast_values.each do |patient_id, code, on|
          result[patient_id]["exam:#{code}"] << Event.new(on: on.is_a?(Date) ? on : Date.iso8601(on.to_s))
        end
      end

      def procedures!(result, ids, since, codes)
        return if codes.empty?

        PerformedProcedure.where(patient_id: ids, status: "recorded", sigtap_code: codes).where(performed_on: since..)
                          .pluck(:patient_id, :sigtap_code, :performed_on).each do |patient_id, code, on|
          source = case code
                   when BP_PROCEDURE then "blood_pressure"
                   when WH_PROCEDURE then "weight_height"
                   else "procedure:#{code}"
                   end
          result[patient_id][source] << Event.new(on: on)
          result[patient_id]["procedure:#{code}"] << Event.new(on: on) if source != "procedure:#{code}"
        end
      end
      private_class_method :consultations!, :screenings!, :exams!, :procedures!
    end
  end
  ```

  `app/services/care_lines/evaluation.rb`:

  ```ruby
  # Avaliação de uma página (ADR 0040; plano, Desvio 9): carrega as fontes de
  # todas as pessoas da página de uma vez (CareLines::Sources) e avalia cada
  # uma (CareLines::Evaluate). Nada é gravado.
  module CareLines
    module Evaluation
      Row = Data.define(:key, :practices, :worst)

      module_function

      def enrollments(enrollments, today:)
        list = enrollments.to_a
        return {} if list.empty?

        ActiveRecord::Associations::Preloader.new(records: list, associations: %i[care_line patient]).call
        items = list.map do |enrollment|
          patient = enrollment.patient
          birth = parse_date(patient.birth_date)
          person = Person.new(patient_id: patient.id, birth_date: birth, sex: patient.sex)
          { key: enrollment.id, practices: enrollment.care_line.practices, person: person,
            anchors: { "enrollment" => enrollment.enrolled_at.in_time_zone.to_date, "birth" => birth, "lmp" => enrollment.anchor_on } }
        end
        items(items, today: today)
      end

      def items(items, today:)
        with_patient = items.select { |item| item[:person].patient_id }
        sources = items.flat_map { |item| item[:practices].flat_map { |p| Evaluate.sources_of(p) } }.uniq
        since = items.map { |item| Evaluate.since(item[:practices], item[:anchors], today) }.min || today
        events = Sources.load(patient_ids: with_patient.map { |item| item[:person].patient_id }, sources: sources, since: since)
        items.to_h do |item|
          statuses = Evaluate.call(practices: item[:practices], person: item[:person], anchors: item[:anchors],
                                   events: events.fetch(item[:person].patient_id, {}), today: today)
          [ item[:key], Row.new(key: item[:key], practices: statuses, worst: Evaluate.worst(statuses)) ]
        end
      end

      def parse_date(value)
        value.present? ? Date.iso8601(value) : nil
      rescue Date::Error
        nil
      end
    end
  end
  ```

- [ ] **Step 4: Rode e veja passar.** `test spec/services/care_lines/sources_spec.rb spec/services/care_lines/evaluation_spec.rb spec/services/care_lines/evaluate_spec.rb` → PASS.

- [ ] **Step 5: Commit** e mova o card **F-23.4** para **Done** (só api; a prova no navegador do painel o leva a Verified):

  ```bash
  git add app/services/care_lines/sources.rb app/services/care_lines/evaluation.rb spec/services/care_lines/sources_spec.rb spec/services/care_lines/evaluation_spec.rb spec/support/care_line_helpers.rb
  git commit -m "feat: load care line sources in batch with one query per source per page

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

---

## Fatia 4 — Elegibilidade, sugestões e inscrição

### Task 9: Elegibilidade, fatos da pessoa e população da equipe

**Files:**
- Create: `app/services/care_lines/eligibility.rb`, `app/services/care_lines/facts.rb`, `app/services/care_lines/team_population.rb`, `spec/services/care_lines/eligibility_spec.rb`, `spec/services/care_lines/team_population_spec.rb`
- Modify: `spec/support/care_line_helpers.rb` (`care_neighborhood!`, `care_citizen!`)

**Interfaces:**
- Consumes: `CareLines::Person` (Task 7); `Territory::ReferenceUnits.ids_by_neighborhood` (existente).
- Produces:
  - `CareLines::Eligibility.match(eligibility, person, on:) → nil|"problem"|"age_sex"|"declared"`; `.denominator?(eligibility, person, on:) → Boolean` (só `problem`/`age_sex` — o oficial não conta declaração); `.closure?(eligibility, person) → Boolean` (linha com `problems`, nenhum elegível ativo e ao menos um resolvido); `.codes_match?(rules, pairs) → Boolean`.
  - `CareLines::Facts.patients(ids) → Hash{patient_id => Person}`; `CareLines::Facts.citizens(ids) → Hash{citizen_id => Person}` (par validado sem paciente); `CareLines::Facts.problem_candidates(rules) → Array<patient_id>` (SQL por prefixo de código ativo).
  - `CareLines::TeamPopulation::LOOKBACK = 24.months`; `.patient_teams(patient_ids = nil) → Hash{patient_id => team_id}`; `.citizen_teams(citizens) → Hash{citizen_id => team_id}`; `.call(team:, line:, rules:) → Array<[Person, CareLineEnrollment|nil]>` (pessoas da equipe elegíveis para o denominador de `rules`).
  - helpers `care_neighborhood!(*units)`, `care_citizen!(age:, sex:, neighborhood: nil, gender_identity: nil)` (par validado sem paciente).

- [ ] **Step 1: Testes que falham.** `spec/services/care_lines/eligibility_spec.rb`:

  ```ruby
  require "rails_helper"

  RSpec.describe CareLines::Eligibility do
    let(:on) { Date.new(2026, 10, 22) }
    let(:entries) { CareLines::Templates::Load.entries.index_by { |e| e["key"] } }

    def rules(key) = entries.fetch(key)["eligibility"]
    def person(age: 40, sex: "female", **attrs) = CareLines::Person.new(birth_date: on.prev_year(age) - 1, sex: sex, **attrs)

    {
      [ "diabetes_c4", { problems: [ %w[cid10 E119] ] } ] => "problem",
      [ "diabetes_c4", { problems: [ %w[ciap2 K86] ], declared: %w[diabetes] } ] => "declared",
      [ "diabetes_c4", { problems: [ %w[ciap2 K86] ] } ] => nil,
      [ "hypertension_c5", { problems: [ %w[cid10 I150] ] } ] => "problem",
      [ "pregnancy_c3", { problems: [ %w[ciap2 W78] ] } ] => "problem",
      [ "pregnancy_c3", { problems: [ %w[ciap2 W78], %w[ciap2 W82] ] } ] => nil, # aborto exclui
      [ "elderly_c6", { age: 60 } ] => "age_sex",
      [ "elderly_c6", { age: 59 } ] => nil,
      [ "women_c7", { age: 9 } ] => "age_sex",
      [ "women_c7", { age: 70 } ] => nil,
      [ "women_c7", { age: 30, sex: "male" } ] => nil,
      [ "women_c7", { age: 30, sex: "male", gender_identity: "trans_man" } ] => "age_sex",
      [ "women_c7", { age: 30, sex: "female", gender_identity: "trans_woman" } ] => nil
    }.each do |(key, attrs), expected|
      it "#{key} #{attrs} → #{expected.inspect}" do
        expect(described_class.match(rules(key), person(**attrs), on: on)).to eq(expected)
      end
    end

    it "criança: até 730 dias de vida" do
      baby = CareLines::Person.new(birth_date: on - 730)
      expect(described_class.match(rules("child_c2"), baby, on: on)).to eq("age_sex")
      expect(described_class.match(rules("child_c2"), CareLines::Person.new(birth_date: on - 731), on: on)).to be_nil
    end

    it "denominador não conta declaração; encerramento só com todos os elegíveis resolvidos" do
      declared = person(declared: %w[diabetes])
      expect(described_class.denominator?(rules("diabetes_c4"), declared, on: on)).to be(false)
      both = person(problems: [ %w[cid10 E11] ], resolved_problems: [ %w[ciap2 T90] ])
      expect(described_class.closure?(rules("diabetes_c4"), both)).to be(false)
      resolved = person(problems: [ %w[ciap2 K86] ], resolved_problems: [ %w[ciap2 T90] ])
      expect(described_class.closure?(rules("diabetes_c4"), resolved)).to be(true)
      expect(described_class.closure?(rules("elderly_c6"), resolved)).to be(false)
    end
  end
  ```

  `spec/services/care_lines/team_population_spec.rb`:

  ```ruby
  require "rails_helper"

  # Plano, Desvio 5: equipe da pessoa para sugestão e prévia.
  RSpec.describe CareLines::TeamPopulation do
    before { care_city! }

    let(:unit_a) { care_unit! }
    let(:doctor_a) { care_doctor!(unit_a) }
    let!(:team_a) { care_team!(unit_a, doctor_a) }
    let(:unit_b) { care_unit! }
    let(:doctor_b) { care_doctor!(unit_b) }
    let!(:team_b) { care_team!(unit_b, doctor_b) }
    let(:diabetes) { adopted_line!("diabetes_c4") }

    it "paciente vai para a equipe do profissional da última consulta; inscrito, para a equipe da inscrição" do
      p1 = care_patient!(age: 60, sex: "male", problems: [ %w[ciap2 T90] ], unit: unit_a, doctor: doctor_a)
      p2 = care_patient!(age: 61, sex: "female", problems: [ %w[ciap2 T90] ], unit: unit_a, doctor: doctor_a)
      care_enrollment!(patient: p2, line: diabetes, team: team_b, by: doctor_b)
      expect(described_class.patient_teams([ p1.id, p2.id ])).to eq(p1.id => team_a.id, p2.id => team_a.id)
      population_a = described_class.call(team: team_a, line: diabetes, rules: diabetes.eligibility)
      population_b = described_class.call(team: team_b, line: diabetes, rules: diabetes.eligibility)
      expect(population_a.map { |person, _| person.patient_id }).to eq([ p1.id ])
      expect(population_b.map { |person, enrollment| [ person.patient_id, enrollment.present? ] }).to eq([ [ p2.id, true ] ])
    end

    it "par validado sem paciente: a única equipe das unidades de referência do bairro; com duas, ninguém" do
      single = care_neighborhood!(unit_a)
      shared = care_neighborhood!(unit_a, unit_b)
      alone = care_citizen!(age: 72, sex: "female", neighborhood: single)
      split = care_citizen!(age: 73, sex: "female", neighborhood: shared)
      expect(described_class.citizen_teams([ alone, split ])).to eq(alone.id => team_a.id)
      elderly = adopted_line!("elderly_c6")
      people = described_class.call(team: team_a, line: elderly, rules: elderly.eligibility).map(&:first)
      expect(people.map(&:citizen_id)).to include(alone.id)
      expect(people.map(&:citizen_id)).not_to include(split.id)
    end

    it "consulta de mais de 24 meses não atribui" do
      old = travel_to(25.months.ago) { care_patient!(age: 70, sex: "male", problems: [ %w[ciap2 T90] ], unit: unit_a, doctor: doctor_a) }
      expect(described_class.patient_teams([ old.id ])).to eq({})
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar.** `test spec/services/care_lines/eligibility_spec.rb spec/services/care_lines/team_population_spec.rb` → FAIL (`uninitialized constant CareLines::Eligibility`).

- [ ] **Step 3: Helpers.** Em `spec/support/care_line_helpers.rb`:

  ```ruby
    def care_neighborhood!(*units)
      neighborhood = Neighborhood.create!(name: "Bairro #{SecureRandom.hex(3)}", source: "manual")
      units.each { |unit| NeighborhoodCoverage.create!(neighborhood: neighborhood, health_unit: unit) }
      neighborhood
    end

    # Par validado que nunca consultou (sem paciente).
    def care_citizen!(age:, sex:, neighborhood: nil, gender_identity: nil)
      citizen = verified_citizen!(exam_seq!, age: age, sex: sex)
      citizen.update!(neighborhood: neighborhood, gender_identity: gender_identity)
      citizen
    end
  ```

- [ ] **Step 4: Implemente.** `app/services/care_lines/eligibility.rb`:

  ```ruby
  # Elegibilidade (ADR 0040; spec §2 decisões 2, 8 e 9). PURA. Problema
  # ativo por prefixo de código (CID sem ponto); faixa de idade (dias ou anos)
  # e sexo (com identidade de gênero incluída/excluída — C7, caderno 4.1–4.2);
  # declaração do cidadão só sugere, nunca entra no denominador.
  module CareLines
    module Eligibility
      module_function

      def match(rules, person, on:)
        return nil if rules["exclude_problems"] && codes_match?(rules["exclude_problems"], person.problems)
        return nil unless demographics?(rules, person, on)
        return "problem" if rules["problems"] && codes_match?(rules["problems"], person.problems)
        return "age_sex" if rules["problems"].nil? && demographic_rule?(rules)
        return "declared" if (Array(rules["declared"]) & person.declared).any?

        nil
      end

      def denominator?(rules, person, on:) = %w[problem age_sex].include?(match(rules, person, on: on))

      def closure?(rules, person)
        return false unless rules["problems"]

        !codes_match?(rules["problems"], person.problems) && codes_match?(rules["problems"], person.resolved_problems)
      end

      def codes_match?(rules, pairs)
        pairs.any? { |terminology, code| Array(rules[terminology]).any? { |prefix| code.start_with?(prefix) } }
      end

      def demographic_rule?(rules)
        %w[age_min_days age_max_days age_min_years age_max_years sex].any? { |key| rules[key] }
      end

      def demographics?(rules, person, on)
        days = person.age_days(on)
        years = person.age_years(on)
        if rules["age_min_days"] || rules["age_max_days"]
          return false if days.nil? || (rules["age_min_days"] && days < rules["age_min_days"]) ||
                          (rules["age_max_days"] && days > rules["age_max_days"])
        end
        if rules["age_min_years"] || rules["age_max_years"]
          return false if years.nil? || (rules["age_min_years"] && years < rules["age_min_years"]) ||
                          (rules["age_max_years"] && years > rules["age_max_years"])
        end
        return false if Array(rules["gender_identities_excluded"]).include?(person.gender_identity)
        return true if rules["sex"].nil?

        person.sex == rules["sex"] || Array(rules["gender_identities_included"]).include?(person.gender_identity)
      end
      private_class_method :demographic_rule?, :demographics?
    end
  end
  ```

  `app/services/care_lines/facts.rb`:

  ```ruby
  # O que a elegibilidade precisa de cada pessoa, decifrado em lote (consultas
  # constantes por lote): nascimento e sexo (paciente ou par), identidade de
  # gênero e condições declaradas (pares validados), problemas ativos e
  # resolvidos. Nada é gravado.
  module CareLines
    module Facts
      module_function

      def patients(ids)
        ids = ids.uniq
        return {} if ids.empty?

        problems = PatientProblem.where(patient_id: ids).pluck(:patient_id, :terminology, :code, :status).group_by(&:first)
        pairs = Citizen.not_erased.verification_level_verified.where(patient_id: ids).order(updated_at: :desc).to_a
        declared = declared_by_citizen(pairs.map(&:id))
        Patient.where(id: ids).to_a.to_h do |patient|
          linked = pairs.select { |c| c.patient_id == patient.id }
          rows = problems.fetch(patient.id, [])
          [ patient.id, Person.new(patient_id: patient.id, birth_date: date(patient.birth_date), sex: patient.sex,
                                   gender_identity: linked.first&.gender_identity,
                                   problems: rows.select { |r| r[3] == "active" }.map { |r| r[1, 2] },
                                   resolved_problems: rows.select { |r| r[3] == "resolved" }.map { |r| r[1, 2] },
                                   declared: linked.flat_map { |c| declared.fetch(c.id, []) }.uniq) ]
        end
      end

      def citizens(ids)
        ids = ids.uniq
        return {} if ids.empty?

        declared = declared_by_citizen(ids)
        Citizen.not_erased.verification_level_verified.where(id: ids, patient_id: nil).to_a.to_h do |citizen|
          [ citizen.id, Person.new(citizen_id: citizen.id, birth_date: date(citizen.birth_date), sex: citizen.sex,
                                   gender_identity: citizen.gender_identity, declared: declared.fetch(citizen.id, [])) ]
        end
      end

      # Pacientes com problema ativo que casa por prefixo (uma consulta).
      def problem_candidates(rules)
        clauses = rules.flat_map do |terminology, prefixes|
          prefixes.map { |prefix| PatientProblem.sanitize_sql([ "(terminology = ? AND code LIKE ?)", terminology, "#{prefix}%" ]) }
        end
        return [] if clauses.empty?

        PatientProblem.active_problems.where(clauses.join(" OR ")).distinct.pluck(:patient_id)
      end

      def declared_by_citizen(citizen_ids)
        return {} if citizen_ids.empty?

        CitizenDeclaredCondition.current.where(citizen_id: citizen_ids).to_a.group_by(&:citizen_id)
                                .transform_values { |rows| rows.map(&:condition).uniq - %w[none] }
      end

      def date(value)
        value.present? ? Date.iso8601(value) : nil
      rescue Date::Error
        nil
      end
    end
  end
  ```

  `app/services/care_lines/team_population.rb`:

  ```ruby
  # Equipe de cada pessoa e população de uma equipe (ADR 0040; spec §2
  # decisões 8 e 9; plano, Desvio 5). Inscrito → equipe da inscrição;
  # paciente → equipe ativa do profissional da consulta finalizada mais
  # recente (24 meses), na unidade do vínculo; par validado sem paciente → a
  # única equipe ativa das unidades de referência do bairro.
  module CareLines
    module TeamPopulation
      LOOKBACK = 24.months

      module_function

      def patient_teams(patient_ids = nil)
        return {} if patient_ids&.empty?

        filter = patient_ids ? ApplicationRecord.sanitize_sql([ "AND c.patient_id IN (?)", patient_ids ]) : ""
        sql = ApplicationRecord.sanitize_sql([ <<~SQL, LOOKBACK.ago ])
          SELECT DISTINCT ON (c.patient_id) c.patient_id, ht.id
            FROM consultations c
            JOIN professional_links pl ON pl.id = c.professional_link_id
            JOIN health_team_members m ON m.professional_id = pl.professional_id AND m.ended_on IS NULL
            JOIN health_teams ht ON ht.id = m.health_team_id AND ht.active AND ht.health_unit_id = pl.health_unit_id
           WHERE c.status = 'finalized' AND c.finalized_at >= ? #{filter}
           ORDER BY c.patient_id, c.finalized_at DESC, ht.ine
        SQL
        ApplicationRecord.connection.select_rows(sql).to_h
      end

      def citizen_teams(citizens)
        units = Territory::ReferenceUnits.ids_by_neighborhood(citizens.map(&:neighborhood_id))
        teams = HealthTeam.where(active: true, health_unit_id: units.values.flatten.uniq).pluck(:id, :health_unit_id)
        citizens.each_with_object({}) do |citizen, out|
          unit_ids = units.fetch(citizen.neighborhood_id, [])
          candidates = teams.select { |_, unit_id| unit_ids.include?(unit_id) }.map(&:first)
          out[citizen.id] = candidates.first if candidates.size == 1
        end
      end

      # Pessoas da equipe, elegíveis para o denominador das regras, com a
      # inscrição ativa delas na linha (ou nil).
      def call(team:, line:, rules:, on: Time.zone.today)
        enrolled = line ? CareLineEnrollment.active_enrollments.where(care_line: line).to_a.index_by(&:patient_id) : {}
        patient_ids = enrolled.select { |_, e| e.team_id == team.id }.keys
        patient_ids += patient_teams.select { |id, team_id| team_id == team.id && !enrolled.key?(id) }.keys
        neighborhoods = NeighborhoodCoverage.where(health_unit_id: team.health_unit_id).pluck(:neighborhood_id)
        candidates = Citizen.not_erased.verification_level_verified.where(patient_id: nil, neighborhood_id: neighborhoods).to_a
        citizen_ids = citizen_teams(candidates).select { |_, team_id| team_id == team.id }.keys

        people = Facts.patients(patient_ids).values + Facts.citizens(citizen_ids).values
        people.select { |person| Eligibility.denominator?(rules, person, on: on) }
              .map { |person| [ person.with(team_id: team.id), person.patient_id && enrolled[person.patient_id] ] }
      end
    end
  end
  ```

- [ ] **Step 5: Rode e veja passar.** `test spec/services/care_lines/eligibility_spec.rb spec/services/care_lines/team_population_spec.rb` → PASS.

- [ ] **Step 6: Commit.**

  ```bash
  git add app/services/care_lines/eligibility.rb app/services/care_lines/facts.rb app/services/care_lines/team_population.rb spec/services/care_lines/eligibility_spec.rb spec/services/care_lines/team_population_spec.rb spec/support/care_line_helpers.rb
  git commit -m "feat: compute care line eligibility and team population for suggestions and preview

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 10: Sugestões — job diário, eventos e encerramento por problema resolvido (F-23.3)

**Files:**
- Create: `app/services/care_lines/suggest.rb`, `app/jobs/care_lines/suggest_job.rb`, `app/jobs/care_lines/problem_changed_job.rb`, `app/jobs/care_lines/condition_declared_job.rb`, `spec/services/care_lines/suggest_spec.rb`, `spec/jobs/care_lines/suggest_job_spec.rb`, `spec/jobs/care_lines/consumers_spec.rb`
- Modify: `config/initializers/domain_events.rb` (`patient_problem.changed` → `[ CareLines::ProblemChangedJob ]`; `citizen_condition.declared` → `[ CareLines::ConditionDeclaredJob ]`), `config/recurring.yml`, `spec/support/care_line_helpers.rb` (`resolve_problem!`, `declare!`)

**Interfaces:**
- Consumes: `CareLines::{Eligibility,Facts,TeamPopulation}` (Task 9); `CareLines::Gate` (Task 1).
- Produces: `CareLines::Suggest.run(today:) → { created:, closures: }`; `CareLines::Suggest.for_patient(patient_id, today:) → Integer`; `CareLines::Suggest.for_citizen(citizen_id, today:) → Integer`; `CareLines::Suggest::COOLDOWN = 365.days`; evento `care_line_suggestion.created` (`care_line_suggestion_id`); jobs `CareLines::SuggestJob` (diário), `CareLines::ProblemChangedJob`, `CareLines::ConditionDeclaredJob`; helpers `resolve_problem!(patient, code, by:)`, `declare!(citizen, condition)`.

- [ ] **Step 1: Testes que falham.** `spec/services/care_lines/suggest_spec.rb`:

  ```ruby
  require "rails_helper"

  # ADR 0040 (spec §2 decisões 2, 8, 9; plano, Desvios 5–7). Review Focus 1 e 2.
  RSpec.describe CareLines::Suggest do
    before { care_city! }

    let(:today) { Time.zone.today }
    let(:unit) { care_unit! }
    let(:doctor) { care_doctor!(unit) }
    let!(:team) { care_team!(unit, doctor) }
    let!(:diabetes) { adopted_line!("diabetes_c4") }

    it "problema ativo → sugestão pendente da equipe da última consulta, com o problema como evidência; idempotente" do
      patient = care_patient!(age: 58, sex: "male", problems: [ %w[ciap2 T90] ], unit: unit, doctor: doctor)
      expect(described_class.run(today: today)).to eq(created: 1, closures: 0)
      suggestion = CareLineSuggestion.sole
      problem = patient.problems.sole
      expect(suggestion).to have_attributes(patient_id: patient.id, citizen_id: nil, care_line: diabetes, team: team, reason: "problem",
                                            status: "pending", evidence_ref: "patient_problem:#{problem.id}")
      expect(DomainEvent.where(name: "care_line_suggestion.created").last.payload).to eq("care_line_suggestion_id" => suggestion.id)
      expect(described_class.run(today: today)).to eq(created: 0, closures: 0)
    end

    it "idade e sexo e condição declarada olham pares validados sem paciente; par sem equipe não recebe" do
      elderly = adopted_line!("elderly_c6")
      neighborhood = care_neighborhood!(unit)
      older = care_citizen!(age: 72, sex: "female", neighborhood: neighborhood)
      homeless = care_citizen!(age: 75, sex: "male")
      declared = care_citizen!(age: 40, sex: "female", neighborhood: neighborhood)
      declare!(declared, "diabetes")
      described_class.run(today: today)
      expect(CareLineSuggestion.where(citizen_id: older.id).pluck(:care_line_id, :reason)).to eq([ [ elderly.id, "age_sex" ] ])
      expect(CareLineSuggestion.where(citizen_id: declared.id).pluck(:care_line_id, :reason)).to eq([ [ diabetes.id, "declared" ] ])
      expect(CareLineSuggestion.where(citizen_id: homeless.id)).not_to exist
    end

    it "recusada não volta por 365 dias; inscrito não recebe; linha inativa não sugere" do
      patient = care_patient!(age: 58, sex: "male", problems: [ %w[ciap2 T90] ], unit: unit, doctor: doctor)
      described_class.run(today: today)
      CareLineSuggestion.sole.update!(status: "dismissed", decided_by_user: doctor, decided_at: Time.current, dismiss_reason: "Acompanha no ambulatório")
      expect(described_class.run(today: today + 1)).to eq(created: 0, closures: 0)
      expect(described_class.run(today: today + 366)[:created]).to eq(0) # created_at real ainda é hoje: o corte usa created_at
      travel_to(366.days.from_now) { expect(described_class.run(today: Time.zone.today)[:created]).to eq(1) }
      CareLineSuggestion.where(status: "pending").delete_all
      care_enrollment!(patient: patient, line: diabetes, team: team, by: doctor)
      expect(described_class.run(today: today)[:created]).to eq(0)
      diabetes.update!(active: false)
      other = care_patient!(age: 60, sex: "female", problems: [ %w[ciap2 T90] ], unit: unit, doctor: doctor)
      described_class.run(today: today)
      expect(CareLineSuggestion.where(patient_id: other.id)).not_to exist
    end

    it "encerramento só com todos os problemas elegíveis resolvidos; nunca encerra sozinho" do
      patient = care_patient!(age: 58, sex: "male", problems: [ %w[ciap2 T90], %w[cid10 E119] ], unit: unit, doctor: doctor)
      enrollment = care_enrollment!(patient: patient, line: diabetes, team: team, by: doctor)
      resolve_problem!(patient, "T90", by: doctor)
      expect(described_class.run(today: today)).to eq(created: 0, closures: 0)
      resolve_problem!(patient, "E119", by: doctor)
      expect(described_class.run(today: today)).to eq(created: 0, closures: 1)
      closure = CareLineSuggestion.sole
      expect(closure).to have_attributes(reason: "resolved_problem", enrollment_id: enrollment.id, patient_id: patient.id, team: team)
      expect(enrollment.reload.status).to eq("active")
      expect(described_class.run(today: today)[:closures]).to eq(0)
    end

    it "for_patient e for_citizen: só a pessoa do evento" do
      patient = care_patient!(age: 58, sex: "male", problems: [ %w[ciap2 T90] ], unit: unit, doctor: doctor)
      care_patient!(age: 59, sex: "male", problems: [ %w[ciap2 T90] ], unit: unit, doctor: doctor)
      expect(described_class.for_patient(patient.id, today: today)).to eq(1)
      expect(CareLineSuggestion.pluck(:patient_id)).to eq([ patient.id ])
      citizen = care_citizen!(age: 30, sex: "female", neighborhood: care_neighborhood!(unit))
      declare!(citizen, "diabetes")
      expect(described_class.for_citizen(citizen.id, today: today)).to eq(1)
    end
  end
  ```

  `spec/jobs/care_lines/consumers_spec.rb`:

  ```ruby
  require "rails_helper"

  RSpec.describe "Consumidores das linhas de cuidado", type: :job do
    before { care_city! }

    let(:unit) { care_unit! }
    let(:doctor) { care_doctor!(unit) }
    let!(:team) { care_team!(unit, doctor) }
    let!(:line) { adopted_line!("diabetes_c4") }

    def args(name, payload) = { event_id: SecureRandom.uuid, event_name: name, city_slug: Current.city.slug, payload: payload }

    it "os eventos estão ligados aos consumidores" do
      expect(DomainEvents.registry["patient_problem.changed"].map(&:job)).to eq([ "CareLines::ProblemChangedJob" ])
      expect(DomainEvents.registry["citizen_condition.declared"].map(&:job)).to eq([ "CareLines::ConditionDeclaredJob" ])
    end

    it "problema mudou → sugestão daquele paciente; care_lines desligado → nada" do
      patient = care_patient!(age: 58, sex: "male", problems: [ %w[ciap2 T90] ], unit: unit, doctor: doctor)
      problem = patient.problems.sole
      Platform::Features.set!(city: Current.city, key: "care_lines", enabled: false, maintainer: ledi_maintainer!)
      CareLines::ProblemChangedJob.new.perform(**args("patient_problem.changed", "patient_problem_id" => problem.id, "kind" => "added"))
      expect(CareLineSuggestion.count).to eq(0)
      Platform::Features.set!(city: Current.city, key: "care_lines", enabled: true, maintainer: ledi_maintainer!)
      CareLines::ProblemChangedJob.new.perform(**args("patient_problem.changed", "patient_problem_id" => problem.id, "kind" => "added"))
      expect(CareLineSuggestion.sole.patient_id).to eq(patient.id)
    end

    it "condição declarada → sugestão daquele par" do
      citizen = care_citizen!(age: 30, sex: "female", neighborhood: care_neighborhood!(unit))
      declare!(citizen, "diabetes")
      CareLines::ConditionDeclaredJob.new.perform(**args("citizen_condition.declared", "citizen_id" => citizen.id))
      expect(CareLineSuggestion.sole).to have_attributes(citizen_id: citizen.id, reason: "declared")
    end
  end
  ```

  `spec/jobs/care_lines/suggest_job_spec.rb`:

  ```ruby
  require "rails_helper"

  RSpec.describe CareLines::SuggestJob do
    it "roda por cidade só com care_lines utilizável e agenda às 6h45" do
      care_city!
      unit = care_unit!
      doctor = care_doctor!(unit)
      care_team!(unit, doctor)
      adopted_line!("diabetes_c4")
      care_patient!(age: 58, sex: "male", problems: [ %w[ciap2 T90] ], unit: unit, doctor: doctor)
      Current.city.update!(record_mode: "integrated")
      CityCatalog.reset_cache!
      described_class.perform_now
      expect(CareLineSuggestion.count).to eq(0)
      Current.city.update!(record_mode: "record")
      CityCatalog.reset_cache!
      described_class.perform_now
      expect(CareLineSuggestion.count).to eq(1)
      recurring = YAML.load_file(Rails.root.join("config/recurring.yml"), aliases: true).dig("default", "care_lines_suggest")
      expect(recurring).to eq("class" => "CareLines::SuggestJob", "queue" => "housekeeping", "schedule" => "every day at 6:45am America/Sao_Paulo")
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar.** `test spec/services/care_lines/suggest_spec.rb spec/jobs/care_lines` → FAIL (`uninitialized constant CareLines::Suggest`).

- [ ] **Step 3: Helpers.** Em `spec/support/care_line_helpers.rb`:

  ```ruby
    # Resolve um problema ativo pelo caminho único (Patients::ApplyProblemEvent),
    # com a consulta finalizada do paciente como origem.
    def resolve_problem!(patient, code, by:)
      problem = patient.problems.active_problems.find_by!(code: code)
      consultation = patient.consultations.finalized_consultations.order(:finalized_at).last
      ApplicationRecord.transaction do
        patient.lock!
        result = Patients::ApplyProblemEvent.call(patient: patient, action: "resolve", by: by, source: { consultation: consultation }, problem: problem)
        raise "resolução recusada: #{result.reason}" if result.failure?
      end
    end

    def declare!(citizen, condition) = CitizenDeclaredCondition.create!(citizen: citizen, condition: condition, declared_at: Time.current)
  ```

- [ ] **Step 4: Implemente.** `app/services/care_lines/suggest.rb`:

  ```ruby
  # Sugestões (ADR 0040; spec §2 decisões 2, 8 e 9; plano, Desvios 5–7). O
  # sistema sugere, um médico ou enfermeiro confirma (Task 11). Pacientes e
  # pares validados sem paciente; só quem tem equipe atribuída recebe; nada
  # repete por 365 dias; inscrito não recebe. Encerramento por problema
  # resolvido vira SUGESTÃO, nunca encerramento.
  module CareLines
    module Suggest
      BATCH = 500
      COOLDOWN = 365.days

      module_function

      def run(today: Time.zone.today)
        lines = CareLine.active_lines.to_a
        return { created: 0, closures: 0 } if lines.empty?

        created = candidate_patient_ids(lines).each_slice(BATCH).sum { |ids| for_people(lines, Facts.patients(ids).values, today) }
        created += Citizen.not_erased.verification_level_verified.where(patient_id: nil).in_batches(of: BATCH)
                          .sum { |batch| for_people(lines, Facts.citizens(batch.pluck(:id)).values, today) }
        { created: created, closures: closures!(lines, today) }
      end

      def for_patient(patient_id, today: Time.zone.today)
        lines = CareLine.active_lines.to_a
        for_people(lines, Facts.patients([ patient_id ]).values, today) + closures!(lines, today, patient_ids: [ patient_id ])
      end

      def for_citizen(citizen_id, today: Time.zone.today)
        lines = CareLine.active_lines.to_a
        citizen = Citizen.find_by(id: citizen_id)
        return 0 unless citizen

        people = citizen.patient_id ? Facts.patients([ citizen.patient_id ]).values : Facts.citizens([ citizen.id ]).values
        for_people(lines, people, today)
      end

      # Linha por idade e sexo olha todos os pacientes; por problema, só quem
      # tem problema que casa; declaração, os pacientes dos pares que declararam.
      def candidate_patient_ids(lines)
        return Patient.pluck(:id) if lines.any? { |line| line.eligibility["problems"].nil? && line.eligibility["declared"].nil? }

        ids = lines.filter_map { |line| line.eligibility["problems"] }.flat_map { |rules| Facts.problem_candidates(rules) }
        declared = CitizenDeclaredCondition.current.joins(:citizen).where.not(citizens: { patient_id: nil }).distinct.pluck("citizens.patient_id")
        (ids + declared).uniq
      end

      def for_people(lines, people, today)
        return 0 if people.empty?

        patient_ids = people.filter_map(&:patient_id)
        citizen_ids = people.filter_map(&:citizen_id)
        teams = TeamPopulation.patient_teams(patient_ids)
                              .merge(TeamPopulation.citizen_teams(Citizen.where(id: citizen_ids).select(:id, :neighborhood_id).to_a))
        enrolled = CareLineEnrollment.active_enrollments.where(patient_id: patient_ids).pluck(:patient_id, :care_line_id).to_set
        recent = CareLineSuggestion.where(patient_id: patient_ids).or(CareLineSuggestion.where(citizen_id: citizen_ids))
                                   .where("status = 'pending' OR created_at >= ?", COOLDOWN.ago)
                                   .where.not(reason: "resolved_problem")
                                   .pluck(:patient_id, :citizen_id, :care_line_id).to_set { |p, c, l| [ p || c, l ] }
        evidence = PatientProblem.active_problems.where(patient_id: patient_ids).pluck(:id, :patient_id, :terminology, :code)
                                 .group_by { |row| row[1] }

        people.sum do |person|
          id = person.patient_id || person.citizen_id
          team_id = teams[id]
          next 0 unless team_id

          lines.sum do |line|
            reason = Eligibility.match(line.eligibility, person, on: today)
            next 0 if reason.nil? || enrolled.include?([ person.patient_id, line.id ]) || recent.include?([ id, line.id ])

            ref = reason == "problem" ? problem_ref(line, evidence.fetch(person.patient_id, [])) : nil
            create!(patient_id: person.patient_id, citizen_id: person.patient_id ? nil : person.citizen_id, care_line_id: line.id,
                    team_id: team_id, reason: reason, evidence_ref: ref)
          end
        end
      end

      def closures!(lines, today, patient_ids: nil)
        problem_lines = lines.select { |line| line.eligibility["problems"] }
        return 0 if problem_lines.empty?

        scope = CareLineEnrollment.active_enrollments.where(care_line_id: problem_lines.map(&:id))
        scope = scope.where(patient_id: patient_ids) if patient_ids
        enrollments = scope.to_a
        return 0 if enrollments.empty?

        people = Facts.patients(enrollments.map(&:patient_id))
        recent = CareLineSuggestion.where(enrollment_id: enrollments.map(&:id))
                                   .where("status = 'pending' OR created_at >= ?", COOLDOWN.ago).pluck(:enrollment_id).to_set
        by_line = problem_lines.index_by(&:id)
        enrollments.sum do |enrollment|
          person = people[enrollment.patient_id]
          next 0 if person.nil? || recent.include?(enrollment.id)
          next 0 unless Eligibility.closure?(by_line.fetch(enrollment.care_line_id).eligibility, person)

          create!(patient_id: enrollment.patient_id, citizen_id: nil, care_line_id: enrollment.care_line_id, team_id: enrollment.team_id,
                  reason: "resolved_problem", enrollment_id: enrollment.id, evidence_ref: "care_line_enrollment:#{enrollment.id}")
        end
      end

      def problem_ref(line, rows)
        rules = line.eligibility["problems"]
        row = rows.find { |_, _, terminology, code| Array(rules[terminology]).any? { |prefix| code.start_with?(prefix) } }
        row && "patient_problem:#{row[0]}"
      end

      # Cada sugestão no próprio savepoint: a corrida com o consumidor de
      # evento (mesma pessoa, mesma linha) cai no índice único e vira 0.
      def create!(**attrs)
        ApplicationRecord.transaction(requires_new: true) do
          suggestion = CareLineSuggestion.create!(attrs)
          DomainEvents.publish("care_line_suggestion.created", care_line_suggestion_id: suggestion.id)
        end
        1
      rescue ActiveRecord::RecordNotUnique
        0
      end
      private_class_method :candidate_patient_ids, :for_people, :closures!, :problem_ref, :create!
    end
  end
  ```

  `app/jobs/care_lines/suggest_job.rb`:

  ```ruby
  # Job diário das sugestões (ADR 0040; spec §4). Por cidade (EachCityJob),
  # só com care_lines ligado e utilizável. Log só com contagens.
  module CareLines
    class SuggestJob < ApplicationJob
      prepend EachCityJob
      queue_as :housekeeping

      def perform
        return unless Gate.usable?(Current.city)

        result = Suggest.run(today: Time.zone.today)
        Rails.logger.info("[CareLines::SuggestJob] city=#{Current.city.slug} created=#{result[:created]} closures=#{result[:closures]}")
      end
    end
  end
  ```

  `app/jobs/care_lines/problem_changed_job.rb`:

  ```ruby
  # Consumidor de patient_problem.changed (ADR 0040; spec §4): sugestões e
  # encerramento daquele paciente. Roda na transação da cidade: o portão lê
  # só a plataforma (Gate.enabled?).
  module CareLines
    class ProblemChangedJob < ApplicationJob
      include IdempotentConsumer
      queue_as :default

      def handle(patient_problem_id:, **)
        return unless Gate.enabled?(Current.city)

        patient_id = PatientProblem.where(id: patient_problem_id).pick(:patient_id)
        Suggest.for_patient(patient_id, today: Time.zone.today) if patient_id
      end
    end
  end
  ```

  `app/jobs/care_lines/condition_declared_job.rb`:

  ```ruby
  # Consumidor de citizen_condition.declared (ADR 0040; spec §4).
  module CareLines
    class ConditionDeclaredJob < ApplicationJob
      include IdempotentConsumer
      queue_as :default

      def handle(citizen_id:, **)
        return unless Gate.enabled?(Current.city)

        Suggest.for_citizen(citizen_id, today: Time.zone.today)
      end
    end
  end
  ```

  `config/initializers/domain_events.rb`: troque `DomainEvents.bind "patient_problem.changed", to: []` por

  ```ruby
    # ADR 0040: o problema que muda dispara a sugestão (e o encerramento) daquele paciente.
    DomainEvents.bind "patient_problem.changed", to: [ CareLines::ProblemChangedJob ]
  ```
  e, junto das linhas da Task 5: `DomainEvents.bind "citizen_condition.declared", to: [ CareLines::ConditionDeclaredJob ]`.

  `config/recurring.yml`, dentro de `default: &default`, depois de `exams_daily`:

  ```yaml
    care_lines_suggest:
      class: CareLines::SuggestJob
      queue: housekeeping
      schedule: "every day at 6:45am America/Sao_Paulo"
  ```

- [ ] **Step 5: Rode e veja passar.** `test spec/services/care_lines/suggest_spec.rb spec/jobs/care_lines spec/initializers/domain_events_bindings_spec.rb` → PASS. Rode também `test spec/commands/patients spec/requests/consultations_spec.rb` (o `patient_problem.changed` agora tem consumidor; nada ali pode mudar de comportamento).

- [ ] **Step 6: Commit.**

  ```bash
  git add app/services/care_lines/suggest.rb app/jobs/care_lines config/initializers/domain_events.rb config/recurring.yml spec/services/care_lines/suggest_spec.rb spec/jobs/care_lines spec/support/care_line_helpers.rb
  git commit -m "feat: suggest care line enrollment daily and on problem or declared condition events

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 11: Inscrição confirmada, recusa, saída e contato (F-23.3, F-23.5)

**Files:**
- Create: `app/commands/care_lines/enroll.rb`, `app/commands/care_lines/accept_suggestion.rb`, `app/commands/care_lines/dismiss_suggestion.rb`, `app/commands/care_lines/end_enrollment.rb`, `app/commands/care_lines/record_contact.rb`, `spec/commands/care_lines/enrollment_commands_spec.rb`

**Interfaces:**
- Consumes: `CareLines::Access.{confirmer?,panel?}` (Task 6); `Patients::Resolve.call(citizen)` (existente); modelos (Task 5).
- Produces (todas devolvem `Result`; motivos → status no `CareLines::BaseController::FAILURE_STATUS`):
  - `CareLines::Enroll.call(patient:, line:, team:, by:, anchor_date: nil, suggestion: nil, today: Time.zone.today)` → `payload[:enrollment]`; falhas `line_inactive` (409), `cbo_not_allowed` (403), `anchor_required`, `invalid_anchor` (422), `already_enrolled` (409).
  - `CareLines::AcceptSuggestion.call(suggestion:, by:, team: nil, anchor_date: nil, today:)` → `payload[:enrollment]` (de inscrição) ou `payload[:ended]` (de encerramento); falhas acima mais `invalid_transition` (409), `citizen_not_verified` (409).
  - `CareLines::DismissSuggestion.call(suggestion:, reason:, by:)`; falhas `reason_required`/`too_long` (com `field: "reason"`), `invalid_transition`.
  - `CareLines::EndEnrollment.call(enrollment:, reason:, note:, by:)`; falhas `invalid_reason`, `too_long` (`field: "note"`), `invalid_transition`.
  - `CareLines::RecordContact.call(enrollment:, channel:, outcome:, note:, by:)` → `payload[:contact]`; falhas `invalid_channel`, `invalid_outcome`, `too_long` (`field: "note"`), `invalid_transition`.
  - Eventos (só ids): `care_line_enrollment.created` (`care_line_enrollment_id`, `user_id`), `care_line_suggestion.accepted`/`.dismissed` (`care_line_suggestion_id`, `user_id`), `care_line_enrollment.ended` (`care_line_enrollment_id`, `user_id`), `care_outreach.recorded` (`care_outreach_contact_id`, `care_line_enrollment_id`, `user_id`).

- [ ] **Step 1: Teste que falha** — `spec/commands/care_lines/enrollment_commands_spec.rb`:

  ```ruby
  require "rails_helper"

  # ADR 0040 (Invariantes: nenhuma inscrição sem confirmação de um
  # profissional); spec §2 decisões 2 e 8; plano, Desvios 4 e 8. Review Focus 4.
  RSpec.describe "Inscrição confirmada e saída" do
    before { care_city! }

    let(:today) { Time.zone.today }
    let(:unit) { care_unit! }
    let(:doctor) { care_doctor!(unit) }
    let(:nurse) { care_nurse!(unit) }
    let(:tech) { screener!(unit, cbo: "322205") }
    let!(:team) { care_team!(unit, doctor, nurse) }

    def pending_for(line, citizen: nil, patient: nil)
      CareLineSuggestion.create!(care_line: line, team: team, reason: citizen ? "age_sex" : "problem", citizen: citizen, patient: patient)
    end

    it "aceitar sugestão de par validado sem paciente cria o paciente (Patients::Resolve) e a inscrição; só ids nos eventos" do
      line = adopted_line!("elderly_c6")
      citizen = care_citizen!(age: 72, sex: "female")
      suggestion = pending_for(line, citizen: citizen)
      result = CareLines::AcceptSuggestion.call(suggestion: suggestion, team: team, by: nurse, today: today)
      enrollment = result.payload[:enrollment]
      expect(enrollment).to have_attributes(patient_id: citizen.reload.patient_id, source: "suggestion", suggestion_id: suggestion.id, status: "active")
      expect(suggestion.reload).to have_attributes(status: "accepted", decided_by_user_id: nurse.id)
      expect(DomainEvent.where(name: "care_line_enrollment.created").last.payload.keys).to match_array(%w[care_line_enrollment_id user_id])
      expect(DomainEvent.where(name: "care_line_suggestion.accepted").last.payload.keys).to match_array(%w[care_line_suggestion_id user_id])
      again = CareLines::AcceptSuggestion.call(suggestion: suggestion, team: team, by: nurse, today: today)
      expect(again.reason).to eq(:invalid_transition)
    end

    it "só médico ou enfermeiro confirma; linha inativa recusa; par que perdeu a validação recusa" do
      line = adopted_line!("elderly_c6")
      citizen = care_citizen!(age: 72, sex: "female")
      suggestion = pending_for(line, citizen: citizen)
      expect(CareLines::AcceptSuggestion.call(suggestion: suggestion, team: team, by: tech, today: today).reason).to eq(:cbo_not_allowed)
      citizen.update_columns(verification_level: "declared")
      expect(CareLines::AcceptSuggestion.call(suggestion: suggestion, team: team, by: doctor, today: today).reason).to eq(:citizen_not_verified)
      expect(suggestion.reload).to be_pending
      line.update!(active: false)
      patient = care_patient!(age: 70, sex: "male")
      expect(CareLines::Enroll.call(patient: patient, line: line, team: team, by: doctor).reason).to eq(:line_inactive)
    end

    it "gestante: DUM obrigatória e entre hoje − 336 e hoje; gravada cifrada; nada gravado na recusa" do
      line = adopted_line!("pregnancy_c3")
      patient = care_patient!(age: 27, sex: "female", problems: [ %w[ciap2 W78] ], unit: unit, doctor: doctor)
      suggestion = pending_for(line, patient: patient)
      expect(CareLines::AcceptSuggestion.call(suggestion: suggestion, team: team, by: doctor, today: today).reason).to eq(:anchor_required)
      [ (today + 1).iso8601, (today - 337).iso8601, "2026-13-01", "ontem" ].each do |bad|
        result = CareLines::AcceptSuggestion.call(suggestion: suggestion, team: team, by: doctor, anchor_date: bad, today: today)
        expect(result.reason).to eq(:invalid_anchor), bad
      end
      expect(CareLineEnrollment.count).to eq(0)
      expect(suggestion.reload).to be_pending
      ok = CareLines::AcceptSuggestion.call(suggestion: suggestion, team: team, by: doctor, anchor_date: (today - 70).iso8601, today: today)
      expect(ok.payload[:enrollment].anchor_on).to eq(today - 70)
      expect(CareLines::Enroll.call(patient: patient, line: adopted_line!("diabetes_c4"), team: team, by: doctor, anchor_date: today.iso8601).reason)
        .to eq(:invalid_anchor)
    end

    it "inscrição manual repetida → already_enrolled" do
      line = adopted_line!("diabetes_c4")
      patient = care_patient!(age: 58, sex: "male")
      expect(CareLines::Enroll.call(patient: patient, line: line, team: team, by: doctor)).to be_ok
      expect(CareLines::Enroll.call(patient: patient, line: line, team: team, by: nurse).reason).to eq(:already_enrolled)
    end

    it "recusar exige motivo (até 500); sugestão de encerramento aceita encerra com discharge" do
      line = adopted_line!("diabetes_c4")
      patient = care_patient!(age: 58, sex: "male")
      suggestion = pending_for(line, patient: patient)
      expect(CareLines::DismissSuggestion.call(suggestion: suggestion, reason: "  ", by: doctor).details).to eq(field: "reason")
      expect(CareLines::DismissSuggestion.call(suggestion: suggestion, reason: "x" * 501, by: doctor).reason).to eq(:too_long)
      expect(CareLines::DismissSuggestion.call(suggestion: suggestion, reason: "Segue no ambulatório de endocrinologia", by: doctor)).to be_ok
      enrollment = CareLines::Enroll.call(patient: patient, line: line, team: team, by: doctor).payload[:enrollment]
      closure = CareLineSuggestion.create!(care_line: line, team: team, patient: patient, reason: "resolved_problem", enrollment: enrollment)
      result = CareLines::AcceptSuggestion.call(suggestion: closure, by: tech, today: today)
      expect(result.payload[:ended]).to have_attributes(status: "ended", ended_reason: "discharge", ended_by_user_id: tech.id)
    end

    it "encerrar com motivo do domínio e nota opcional; encerrar de novo recusa; a sugestão de encerramento pendente fica aceita" do
      line = adopted_line!("diabetes_c4")
      enrollment = CareLines::Enroll.call(patient: care_patient!(age: 58, sex: "male"), line: line, team: team, by: doctor).payload[:enrollment]
      closure = CareLineSuggestion.create!(care_line: line, team: team, patient: enrollment.patient, reason: "resolved_problem", enrollment: enrollment)
      expect(CareLines::EndEnrollment.call(enrollment: enrollment, reason: "cured", note: nil, by: doctor).reason).to eq(:invalid_reason)
      expect(CareLines::EndEnrollment.call(enrollment: enrollment, reason: "moved", note: "n" * 501, by: doctor).details).to eq(field: "note")
      expect(CareLines::EndEnrollment.call(enrollment: enrollment, reason: "moved", note: "Mudou para Maringá", by: doctor)).to be_ok
      expect(enrollment.reload).to have_attributes(status: "ended", ended_reason: "moved", ended_note: "Mudou para Maringá")
      expect(closure.reload.status).to eq("accepted")
      expect(CareLines::EndEnrollment.call(enrollment: enrollment, reason: "moved", note: nil, by: doctor).reason).to eq(:invalid_transition)
    end

    it "contato: canal e desfecho do domínio, nota cifrada até 500, só em inscrição ativa" do
      enrollment = CareLines::Enroll.call(patient: care_patient!(age: 58, sex: "male"), line: adopted_line!("diabetes_c4"), team: team,
                                          by: doctor).payload[:enrollment]
      expect(CareLines::RecordContact.call(enrollment: enrollment, channel: "sms", outcome: "reached", note: nil, by: tech).reason).to eq(:invalid_channel)
      expect(CareLines::RecordContact.call(enrollment: enrollment, channel: "phone", outcome: "maybe", note: nil, by: tech).reason).to eq(:invalid_outcome)
      contact = CareLines::RecordContact.call(enrollment: enrollment, channel: "phone", outcome: "scheduled", note: "Marcou para terça", by: tech)
      expect(contact.payload[:contact]).to have_attributes(recorded_by_user_id: tech.id, note: "Marcou para terça")
      expect(DomainEvent.where(name: "care_outreach.recorded").last.payload.keys)
        .to match_array(%w[care_outreach_contact_id care_line_enrollment_id user_id])
      CareLines::EndEnrollment.call(enrollment: enrollment, reason: "death", note: nil, by: doctor)
      expect(CareLines::RecordContact.call(enrollment: enrollment.reload, channel: "phone", outcome: "reached", note: nil, by: tech).reason)
        .to eq(:invalid_transition)
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar.** `test spec/commands/care_lines/enrollment_commands_spec.rb` → FAIL (`uninitialized constant CareLines::AcceptSuggestion`).

- [ ] **Step 3: Implemente.** `app/commands/care_lines/enroll.rb`:

  ```ruby
  # Inscrição (ADR 0040, Invariante: nenhuma sem confirmação de profissional).
  # Quem chama com `by` é quem confirma: médico ou enfermeiro com vínculo na
  # unidade da equipe. DUM só na linha que exige (lmp), entre hoje − 336 e
  # hoje, gravada cifrada. Ordem de travas: paciente → inscrição.
  module CareLines
    module Enroll
      module_function

      def call(patient:, line:, team:, by:, anchor_date: nil, suggestion: nil, today: Time.zone.today)
        return Result.fail(:line_inactive) unless line.active
        return Result.fail(:cbo_not_allowed) unless Access.confirmer?(by, team)

        anchor = anchor(line, anchor_date, today)
        return anchor if anchor.is_a?(Result)

        enrollment = nil
        ApplicationRecord.transaction(requires_new: true) do
          patient.lock!
          enrollment = CareLineEnrollment.create!(patient: patient, care_line: line, team: team, enrolled_by_user: by, enrolled_at: Time.current,
                                                  source: suggestion ? "suggestion" : "manual", suggestion: suggestion, anchor_date: anchor&.iso8601)
          DomainEvents.publish("care_line_enrollment.created", care_line_enrollment_id: enrollment.id, user_id: by.id)
        end
        Result.ok(enrollment: enrollment)
      rescue ActiveRecord::RecordNotUnique
        Result.fail(:already_enrolled)
      end

      def anchor(line, raw, today)
        unless line.requires_lmp?
          return raw.nil? || raw == "" ? nil : Result.fail(:invalid_anchor)
        end
        return Result.fail(:anchor_required) if raw.nil? || raw == ""

        date = raw.is_a?(String) && raw.match?(/\A\d{4}-\d{2}-\d{2}\z/) ? Date.iso8601(raw) : nil
        return Result.fail(:invalid_anchor) unless date && date <= today && date >= today - Catalog::LMP_MAX_AGE_DAYS

        date
      rescue Date::Error
        Result.fail(:invalid_anchor)
      end
      private_class_method :anchor
    end
  end
  ```

  `app/commands/care_lines/accept_suggestion.rb`:

  ```ruby
  # Aceitar sugestão (contrato §4). De inscrição: o paciente nasce por
  # Patients::Resolve quando a sugestão é de par validado (antes de travar a
  # sugestão: ordem cidadão → sugestão → paciente; Desvio 8) e a inscrição
  # sai pelo Enroll (que confere quem confirma). De encerramento: encerra com
  # discharge (quem tem o painel da equipe). Se a inscrição falha depois do
  # Resolve, o paciente criado fica (é o mesmo que a primeira consulta criaria).
  module CareLines
    module AcceptSuggestion
      module_function

      def call(suggestion:, by:, team: nil, anchor_date: nil, today: Time.zone.today)
        return close(suggestion, by) if suggestion.closure?

        team ||= suggestion.team
        return Result.fail(:cbo_not_allowed) unless Access.confirmer?(by, team)

        patient = suggestion.patient
        if patient.nil?
          resolved = Patients::Resolve.call(suggestion.citizen)
          return Result.fail(:citizen_not_verified) if resolved.failure?

          patient = resolved.payload[:patient]
        end

        ApplicationRecord.transaction do
          suggestion.lock!
          next Result.fail(:invalid_transition) unless suggestion.pending?

          enrolled = Enroll.call(patient: patient, line: suggestion.care_line, team: team, by: by, anchor_date: anchor_date,
                                 suggestion: suggestion, today: today)
          next enrolled if enrolled.failure?

          decide!(suggestion, by)
          enrolled
        end
      end

      def close(suggestion, by)
        ApplicationRecord.transaction do
          suggestion.lock!
          next Result.fail(:invalid_transition) unless suggestion.pending?

          enrollment = suggestion.enrollment.lock!
          EndEnrollment.end!(enrollment, reason: "discharge", note: nil, by: by) if enrollment.active?
          # end! aceita as sugestões de encerramento pendentes da inscrição,
          # esta inclusive; só decide aqui se a inscrição já estava encerrada.
          decide!(suggestion, by) if suggestion.reload.pending?
          Result.ok(ended: enrollment)
        end
      end

      def decide!(suggestion, by)
        suggestion.update!(status: "accepted", decided_by_user: by, decided_at: Time.current)
        DomainEvents.publish("care_line_suggestion.accepted", care_line_suggestion_id: suggestion.id, user_id: by.id)
      end
      private_class_method :close, :decide!
    end
  end
  ```

  `app/commands/care_lines/dismiss_suggestion.rb`:

  ```ruby
  # Recusar sugestão com motivo (contrato §4): 1..500, recusado se longo
  # (nunca cortado), cifrado. O cidadão nunca vê.
  module CareLines
    module DismissSuggestion
      module_function

      def call(suggestion:, reason:, by:)
        text = reason.is_a?(String) ? reason.strip : ""
        return Result.fail(:reason_required, details: { field: "reason" }) if text.empty?
        return Result.fail(:too_long, details: { field: "reason" }) if text.length > Catalog::TEXT_MAX

        ApplicationRecord.transaction do
          suggestion.lock!
          next Result.fail(:invalid_transition) unless suggestion.pending?

          suggestion.update!(status: "dismissed", decided_by_user: by, decided_at: Time.current, dismiss_reason: text)
          DomainEvents.publish("care_line_suggestion.dismissed", care_line_suggestion_id: suggestion.id, user_id: by.id)
          Result.ok(suggestion: suggestion)
        end
      end
    end
  end
  ```

  `app/commands/care_lines/end_enrollment.rb`:

  ```ruby
  # Saída com motivo (contrato §1 e §4): uma vez; a nota (opcional, até 500)
  # cifrada. A sugestão de encerramento pendente da inscrição fica aceita por
  # quem encerrou.
  module CareLines
    module EndEnrollment
      module_function

      def call(enrollment:, reason:, note:, by:)
        return Result.fail(:invalid_reason) unless Catalog::END_REASONS.include?(reason)

        text = note.is_a?(String) ? note.strip.presence : nil
        return Result.fail(:too_long, details: { field: "note" }) if text && text.length > Catalog::TEXT_MAX

        ApplicationRecord.transaction do
          enrollment.lock!
          next Result.fail(:invalid_transition) unless enrollment.active?

          end!(enrollment, reason: reason, note: text, by: by)
          Result.ok(enrollment: enrollment)
        end
      end

      # Quem chama trava a inscrição e confere que está ativa.
      def end!(enrollment, reason:, note:, by:)
        enrollment.update!(status: "ended", ended_reason: reason, ended_note: note, ended_at: Time.current, ended_by_user: by)
        CareLineSuggestion.pending_suggestions.where(enrollment_id: enrollment.id).find_each do |closure|
          closure.update!(status: "accepted", decided_by_user: by, decided_at: Time.current)
          DomainEvents.publish("care_line_suggestion.accepted", care_line_suggestion_id: closure.id, user_id: by.id)
        end
        DomainEvents.publish("care_line_enrollment.ended", care_line_enrollment_id: enrollment.id, user_id: by.id)
      end
    end
  end
  ```

  `app/commands/care_lines/record_contact.rb`:

  ```ruby
  # Tentativa de contato da busca ativa (contrato §5): só acréscimo; nota
  # opcional, até 500, cifrada; só em inscrição ativa.
  module CareLines
    module RecordContact
      module_function

      def call(enrollment:, channel:, outcome:, note:, by:)
        return Result.fail(:invalid_channel) unless Catalog::CHANNELS.include?(channel)
        return Result.fail(:invalid_outcome) unless Catalog::OUTCOMES.include?(outcome)

        text = note.is_a?(String) ? note.strip.presence : nil
        return Result.fail(:too_long, details: { field: "note" }) if text && text.length > Catalog::TEXT_MAX
        return Result.fail(:invalid_transition) unless enrollment.active?

        contact = CareOutreachContact.create!(enrollment: enrollment, channel: channel, outcome: outcome, note: text,
                                              recorded_by_user: by, recorded_at: Time.current)
        DomainEvents.publish("care_outreach.recorded", care_outreach_contact_id: contact.id, care_line_enrollment_id: enrollment.id,
                                                       user_id: by.id)
        Result.ok(contact: contact)
      end
    end
  end
  ```

- [ ] **Step 4: Rode e veja passar.** `test spec/commands/care_lines/enrollment_commands_spec.rb` → PASS.

- [ ] **Step 5: Commit.**

  ```bash
  git add app/commands/care_lines/enroll.rb app/commands/care_lines/accept_suggestion.rb app/commands/care_lines/dismiss_suggestion.rb app/commands/care_lines/end_enrollment.rb app/commands/care_lines/record_contact.rb spec/commands/care_lines/enrollment_commands_spec.rb
  git commit -m "feat: confirm care line enrollment by a doctor or nurse, end with reason and record outreach

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 12: Painel nominal, sugestões e busca ativa — rotas (F-23.3, F-23.5)

**Files:**
- Create: `app/services/care_lines/cursor.rb`, `app/services/care_lines/trail.rb`, `app/services/care_lines/board.rb`, `app/controllers/care_lines/suggestions_controller.rb`, `app/controllers/care_lines/enrollments_controller.rb`, `spec/requests/care_lines/suggestions_spec.rb`, `spec/requests/care_lines/enrollments_spec.rb`
- Modify: `app/services/care_lines/json.rb` (`person`, `enrollment`, `suggestion`, `contact`), `config/routes.rb`

**Interfaces:**
- Consumes: `CareLines::Evaluation.enrollments` (Task 8); comandos da Task 11; `CareLines::Access` (Task 6).
- Produces (contrato §4–§5): `GET /care_lines/suggestions?team_id=&care_line_id=&before=&limit=` → `{ suggestions: [<suggestion>], next_before }`; `GET /care_lines/suggestions/count?team_id=` → `{ count }`; `POST /care_lines/suggestions/:id/accept` `{ team_id, anchor_date? }` → 201 `<enrollment>` (encerramento: 200 `<enrollment>` encerrada); `POST /care_lines/suggestions/:id/dismiss` `{ reason }` → `{ id, status }`; `GET /care_lines/enrollments?team_id=&care_line_id=&status=overdue|due_soon|all&before=&limit=` → `{ enrollments: [<enrollment>], next_before }`; `GET /care_lines/enrollments/overdue/count?team_id=` → `{ count }`; `GET /care_lines/enrollments/:id` → `<enrollment>` + `contacts`; `POST /care_lines/enrollments` `{ patient_id | citizen_id, care_line_id, team_id, anchor_date? }` → 201; `POST /care_lines/enrollments/:id/end` `{ reason, note? }`; `POST /care_lines/enrollments/:id/contacts` `{ channel, outcome, note? }` → 201 `{ contact }`. `CareLines::Board.{page(team:, line_id:, status:, before:, limit:, today:), overdue_count(team, today:)}`; `CareLines::Cursor.{encode(at, id), decode(raw), before(scope, cursor, column)}` (`Cursor::Invalid`); `CareLines::Trail.viewed!(user:, patient_id:, citizen_id:, enrollment_id: nil, suggestion_id: nil)`.

- [ ] **Step 1: Testes que falham.** `spec/requests/care_lines/enrollments_spec.rb`:

  ```ruby
  require "rails_helper"

  # Contrato §5; plano, Desvios 9–10. Review Focus 3.
  RSpec.describe "Linhas de cuidado — painel e busca ativa", type: :request do
    before { care_city! }

    let(:unit) { care_unit! }
    let(:doctor) { care_doctor!(unit) }
    let(:nurse) { care_nurse!(unit) }
    let!(:team) { care_team!(unit, doctor, nurse) }
    let(:elderly) { adopted_line!("elderly_c6") }

    # Em dia: consulta finalizada hoje (consulta + peso/altura). Vencida: nunca consultou.
    def enrolled!(ok:, at:)
      patient = ok ? care_patient!(age: 70, sex: "female", problems: [ %w[ciap2 K86] ], unit: unit, doctor: doctor) : care_patient!(age: 71, sex: "male")
      care_enrollment!(patient: patient, line: elderly, team: team, by: doctor, enrolled_at: at)
    end

    def viewed = DomainEvent.where(name: "clinical_record.viewed").select { |e| e.payload["access"] == "care_line" }

    it "lista nominal com situação por prática, último contato e trilha por pessoa; contadores sem trilha" do
      late = enrolled!(ok: false, at: 2.days.ago)
      fine = enrolled!(ok: true, at: 1.day.ago)
      CareLines::RecordContact.call(enrollment: late, channel: "phone", outcome: "no_answer", note: "Caixa postal", by: nurse)
      sign_in_as(nurse)
      get "/care_lines/enrollments", params: { team_id: team.id }
      expect(json_body["enrollments"].map { |e| [ e["id"], e["worst_status"] ] }).to eq([ [ fine.id, "ok" ], [ late.id, "overdue" ] ])
      row = json_body["enrollments"].last
      expect(row["person"].keys).to match_array(%w[patient_id citizen_id display_name age])
      expect(row["team"]).to eq("id" => team.id, "ine" => team.ine, "name" => team.name)
      expect(row["practices"].map { |p| [ p["key"], p["status"] ] }).to eq([ %w[c6_a overdue], %w[c6_b overdue], %w[c6_c untracked], %w[c6_d untracked] ])
      expect(row["last_contact"]).to include("channel" => "phone", "outcome" => "no_answer")
      expect(row["last_contact"]).not_to have_key("note")
      expect(viewed.size).to eq(2)
      expect(viewed.last.payload).to include("user_id" => nurse.id, "access" => "care_line")
      expect(viewed.last.payload.keys).to include("care_line_enrollment_id", "patient_id", "person_ref")
      get "/care_lines/enrollments/overdue/count", params: { team_id: team.id }
      expect(json_body).to eq("count" => 1)
      expect(viewed.size).to eq(2)
    end

    it "busca ativa varre em blocos: página curta com next_before até achar o vencido" do
      late = enrolled!(ok: false, at: 30.days.ago)
      10.times { |i| enrolled!(ok: true, at: (20 - i).days.ago) }
      sign_in_as(nurse)
      get "/care_lines/enrollments", params: { team_id: team.id, status: "overdue", limit: 2 }
      expect(json_body["enrollments"]).to eq([])
      expect(json_body["next_before"]).to be_present
      get "/care_lines/enrollments", params: { team_id: team.id, status: "overdue", limit: 2, before: json_body["next_before"] }
      expect(json_body["enrollments"].map { |e| e["id"] }).to eq([ late.id ])
      expect(json_body["next_before"]).to be_nil
    end

    it "escopo: equipe de outra unidade → 403; inscrição dela por id → 404; filtro inválido → 422" do
      other_unit = care_unit!
      other_team = care_team!(other_unit, care_doctor!(other_unit))
      foreign = care_enrollment!(patient: care_patient!(age: 70, sex: "male"), line: elderly, team: other_team, by: doctor)
      sign_in_as(nurse)
      get "/care_lines/enrollments", params: { team_id: other_team.id }
      expect(response).to have_http_status(:forbidden)
      get "/care_lines/enrollments/#{foreign.id}"
      expect(response).to have_http_status(:not_found)
      get "/care_lines/enrollments", params: { team_id: team.id, status: "late" }
      expect(json_body).to eq("error" => "invalid_filter")
      get "/care_lines/enrollments", params: { team_id: team.id, before: "x" }
      expect(json_body).to eq("error" => "invalid_cursor")
    end

    it "inscrição manual por citizen_id (par validado) e por patient_id; ver, contatar e encerrar" do
      citizen = care_citizen!(age: 72, sex: "female")
      sign_in_as(doctor)
      json_post "/care_lines/enrollments", citizen_id: citizen.id, care_line_id: elderly.id, team_id: team.id
      expect(response).to have_http_status(:created)
      id = json_body["id"]
      expect(json_body["person"]["patient_id"]).to eq(citizen.reload.patient_id)
      json_post "/care_lines/enrollments", patient_id: citizen.patient_id, care_line_id: elderly.id, team_id: team.id
      expect(response).to have_http_status(:conflict)
      expect(json_body).to eq("error" => "already_enrolled")
      json_post "/care_lines/enrollments/#{id}/contacts", channel: "home_visit", outcome: "reached", note: "Agendou consulta"
      expect(response).to have_http_status(:created)
      get "/care_lines/enrollments/#{id}"
      expect(json_body["contacts"].map { |c| [ c["channel"], c["note"] ] }).to eq([ [ "home_visit", "Agendou consulta" ] ])
      json_post "/care_lines/enrollments/#{id}/end", reason: "moved", note: "Mudou de cidade"
      expect(json_body).to include("status" => "ended")
      declared = verified_citizen!(exam_seq!).tap { |c| c.update_columns(verification_level: "declared") }
      json_post "/care_lines/enrollments", citizen_id: declared.id, care_line_id: elderly.id, team_id: team.id
      expect(response).to have_http_status(:conflict)
      expect(json_body).to eq("error" => "citizen_not_verified")
      json_post "/care_lines/enrollments", care_line_id: elderly.id, team_id: team.id
      expect(json_body).to eq("error" => "invalid_person")
    end
  end
  ```

  `spec/requests/care_lines/suggestions_spec.rb`:

  ```ruby
  require "rails_helper"

  # Contrato §4. Cidadão nunca vê sugestão; aqui, só o painel da equipe.
  RSpec.describe "Linhas de cuidado — sugestões", type: :request do
    before { care_city! }

    let(:unit) { care_unit! }
    let(:doctor) { care_doctor!(unit) }
    let(:tech) { screener!(unit, cbo: "322205") }
    let!(:team) { care_team!(unit, doctor, tech) }
    let(:pregnancy) { adopted_line!("pregnancy_c3") }

    it "lista sem texto clínico, com trilha; conta sem trilha; aceita com DUM; técnico não confirma" do
      patient = care_patient!(age: 27, sex: "female", problems: [ %w[ciap2 W78] ], unit: unit, doctor: doctor)
      suggestion = CareLineSuggestion.create!(care_line: pregnancy, team: team, patient: patient, reason: "problem",
                                              evidence_ref: "patient_problem:#{patient.problems.sole.id}")
      sign_in_as(tech)
      get "/care_lines/suggestions", params: { team_id: team.id }
      item = json_body["suggestions"].sole
      expect(item.keys).to match_array(%w[id person care_line reason created_at enrollment_id])
      expect(item["reason"]).to eq("problem")
      expect(item.to_json).not_to include("W78")
      get "/care_lines/suggestions/count", params: { team_id: team.id }
      expect(json_body).to eq("count" => 1)
      json_post "/care_lines/suggestions/#{suggestion.id}/accept", team_id: team.id, anchor_date: (Time.zone.today - 70).iso8601
      expect(response).to have_http_status(:forbidden)
      expect(json_body).to eq("error" => "cbo_not_allowed")
      sign_in_as(doctor)
      json_post "/care_lines/suggestions/#{suggestion.id}/accept", team_id: team.id
      expect(json_body).to eq("error" => "anchor_required")
      json_post "/care_lines/suggestions/#{suggestion.id}/accept", team_id: team.id, anchor_date: (Time.zone.today - 70).iso8601
      expect(response).to have_http_status(:created)
      expect(json_body).to include("anchor_date" => (Time.zone.today - 70).iso8601, "status" => "active")
    end

    it "recusa com motivo; sugestão de outra equipe por id → 404" do
      other_unit = care_unit!
      other_team = care_team!(other_unit, care_doctor!(other_unit))
      mine = CareLineSuggestion.create!(care_line: pregnancy, team: team, citizen: care_citizen!(age: 25, sex: "female"), reason: "declared")
      theirs = CareLineSuggestion.create!(care_line: pregnancy, team: other_team, citizen: care_citizen!(age: 26, sex: "female"), reason: "declared")
      sign_in_as(doctor)
      json_post "/care_lines/suggestions/#{theirs.id}/dismiss", reason: "Não é da área"
      expect(response).to have_http_status(:not_found)
      json_post "/care_lines/suggestions/#{mine.id}/dismiss", reason: ""
      expect(json_body).to eq("error" => "reason_required", "field" => "reason")
      json_post "/care_lines/suggestions/#{mine.id}/dismiss", reason: "Gestação já acompanhada no convênio"
      expect(json_body).to eq("id" => mine.id, "status" => "dismissed")
      get "/care_lines/suggestions", params: { team_id: other_team.id }
      expect(response).to have_http_status(:forbidden)
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar.** `test spec/requests/care_lines/enrollments_spec.rb spec/requests/care_lines/suggestions_spec.rb` → FAIL (rotas inexistentes).

- [ ] **Step 3: Implemente.** `app/services/care_lines/cursor.rb`:

  ```ruby
  require "base64"

  # Cursor opaco das listas (contrato: envelope com next_before): instante
  # com microssegundos e id.
  module CareLines
    module Cursor
      Invalid = Class.new(StandardError)
      Position = Data.define(:at, :id)
      UUID = /\A\h{8}-\h{4}-\h{4}-\h{4}-\h{12}\z/
      INSTANT = /\A\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}\.\d{6}Z\z/

      module_function

      def encode(at, id) = Base64.urlsafe_encode64("#{at.utc.iso8601(6)}|#{id}", padding: false)

      def decode(raw)
        return nil if raw.nil? || raw == ""
        raise Invalid unless raw.is_a?(String)

        at, id = Base64.urlsafe_decode64(raw).split("|", 2)
        raise Invalid unless at.to_s.match?(INSTANT) && id.to_s.match?(UUID)

        Position.new(at: Time.iso8601(at), id: id.downcase)
      rescue ArgumentError
        raise Invalid
      end

      def before(scope, position, column)
        return scope if position.nil?

        table = scope.klass.quoted_table_name
        scope.where("(#{table}.#{column}, #{table}.id) < (?, ?)", position.at, position.id)
      end
    end
  end
  ```

  `app/services/care_lines/trail.rb`:

  ```ruby
  # Trilha da leitura nominal das linhas de cuidado (ADR 0040; spec §7;
  # plano, Desvio 10): o mesmo clinical_record.viewed do 19a, access
  # care_line, uma por pessoa mostrada. Só ids.
  module CareLines
    module Trail
      module_function

      def viewed!(user:, patient_id:, citizen_id:, enrollment_id: nil, suggestion_id: nil)
        payload = { patient_id: patient_id, user_id: user.id, access: "care_line", reason_code: nil,
                    person_ref: patient_id ? "patient:#{patient_id}" : "citizen:#{citizen_id}" }
        payload[:care_line_enrollment_id] = enrollment_id if enrollment_id
        payload[:care_line_suggestion_id] = suggestion_id if suggestion_id
        DomainEvents.publish("clinical_record.viewed", **payload)
      end
    end
  end
  ```

  `app/services/care_lines/board.rb`:

  ```ruby
  # Página do painel (contrato §5; plano, Desvio 9): blocos de `limit`
  # inscrições avaliados em lote; com filtro, até MAX_CHUNKS blocos.
  module CareLines
    module Board
      MAX_CHUNKS = 5
      COUNT_CHUNK = 200
      FILTERS = %w[all overdue due_soon].freeze

      module_function

      def page(team:, line_id:, status:, before:, limit:, today:)
        scope = CareLineEnrollment.active_enrollments.where(team_id: team.id).includes(:team)
        scope = scope.where(care_line_id: line_id) if line_id
        scope = scope.order(enrolled_at: :desc, id: :desc)
        rows = []
        last = nil
        MAX_CHUNKS.times do
          chunk = Cursor.before(scope, last ? Cursor::Position.new(at: last.enrolled_at, id: last.id) : before, :enrolled_at).limit(limit).to_a
          return { rows: rows, next_before: nil } if chunk.empty?

          evaluated = Evaluation.enrollments(chunk, today: today)
          chunk.each do |enrollment|
            last = enrollment
            row = evaluated.fetch(enrollment.id)
            rows << [ enrollment, row ] if keep?(status, row.worst)
            return { rows: rows, next_before: Cursor.encode(enrollment.enrolled_at, enrollment.id) } if rows.size == limit
          end
          return { rows: rows, next_before: nil } if chunk.size < limit
        end
        { rows: rows, next_before: Cursor.encode(last.enrolled_at, last.id) }
      end

      def overdue_count(team, today:)
        CareLineEnrollment.active_enrollments.where(team_id: team.id).in_batches(of: COUNT_CHUNK).sum do |batch|
          Evaluation.enrollments(batch.to_a, today: today).count { |_, row| row.worst == "overdue" }
        end
      end

      def keep?(status, worst) = status == "all" || worst == status
      private_class_method :keep?
    end
  end
  ```

  `app/services/care_lines/json.rb` — acrescente:

  ```ruby
      def person(patient: nil, citizen: nil, today: Time.zone.today)
        who = patient || citizen
        { patient_id: patient&.id, citizen_id: patient ? nil : citizen&.id, display_name: who&.display_name, age: who&.age(on: today) }
      end

      def practice(status)
        { key: status.key, label: status.label, status: status.status, done_at: status.done_at&.iso8601, due_on: status.due_on&.iso8601,
          count: status.count, expected: status.expected }
      end

      def enrollment(enrollment, row:, last_contact:, today: Time.zone.today)
        { id: enrollment.id, person: person(patient: enrollment.patient, today: today),
          care_line: { id: enrollment.care_line.id, name: enrollment.care_line.name },
          team: { id: enrollment.team.id, ine: enrollment.team.ine, name: enrollment.team.name },
          anchor_date: enrollment.anchor_date, status: enrollment.status, enrolled_at: enrollment.enrolled_at.iso8601,
          ended_reason: enrollment.ended_reason, ended_at: enrollment.ended_at&.iso8601,
          worst_status: row.worst, practices: row.practices.map { |s| practice(s) },
          last_contact: last_contact && { channel: last_contact.channel, outcome: last_contact.outcome, at: last_contact.recorded_at.iso8601 } }
      end

      def contact(contact)
        { id: contact.id, channel: contact.channel, outcome: contact.outcome, note: contact.note,
          recorded_by: { id: contact.recorded_by_user_id, name: contact.recorded_by_user.professional&.professional_name },
          recorded_at: contact.recorded_at.iso8601 }
      end

      def suggestion(suggestion, today: Time.zone.today)
        { id: suggestion.id, person: person(patient: suggestion.patient, citizen: suggestion.citizen, today: today),
          care_line: { id: suggestion.care_line.id, name: suggestion.care_line.name }, reason: suggestion.reason,
          created_at: suggestion.created_at.iso8601, enrollment_id: suggestion.enrollment_id }
      end
  ```
  (`User` não tem nome; o nome de quem registrou vem do perfil profissional, `Professional#professional_name`.)

  `app/controllers/care_lines/enrollments_controller.rb`:

  ```ruby
  # Painel nominal e busca ativa (contrato §5). health_professional com
  # vínculo na unidade da equipe; lista com equipe alheia → 403; registro de
  # outra equipe por id → 404. Leitura nominal com trilha (uma por pessoa).
  module CareLines
    class EnrollmentsController < BaseController
      before_action :load_enrollment, only: %i[show end_enrollment contacts]

      def index
        team = scoped_team or return
        status = params[:status].presence || "all"
        return render_error(:invalid_filter) unless Board::FILTERS.include?(status)
        return render_error(:invalid_filter) if params[:care_line_id].present? && !uuid?(params[:care_line_id])

        page = Board.page(team: team, line_id: params[:care_line_id].presence, status: status, before: Cursor.decode(params[:before]),
                          limit: Timeline::Query.limit(params[:limit]), today: Time.zone.today)
        enrollments = page[:rows].map(&:first)
        contacts = last_contacts(enrollments.map(&:id))
        payload = page[:rows].map do |enrollment, row|
          Trail.viewed!(user: Current.user, patient_id: enrollment.patient_id, citizen_id: nil, enrollment_id: enrollment.id)
          Json.enrollment(enrollment, row: row, last_contact: contacts[enrollment.id])
        end
        render json: { enrollments: payload, next_before: page[:next_before] }
      rescue Cursor::Invalid
        render_error(:invalid_cursor)
      end

      def overdue_count
        team = scoped_team or return
        render json: { count: Board.overdue_count(team, today: Time.zone.today) }
      end

      def show
        render json: detail(@enrollment)
      end

      def create
        team = HealthTeam.find_by(id: params[:team_id]) if uuid?(params[:team_id])
        return forbid("out_of_scope") unless Access.panel?(Current.user, team)

        line = uuid?(params[:care_line_id]) && CareLine.find_by(id: params[:care_line_id])
        return render_error(:invalid_care_line) unless line

        patient = person_patient or return
        result = Enroll.call(patient: patient, line: line, team: team, by: Current.user, anchor_date: params[:anchor_date])
        return render_failure(result) if result.failure?

        render json: detail(result.payload[:enrollment]), status: :created
      end

      def end_enrollment
        result = EndEnrollment.call(enrollment: @enrollment, reason: params[:reason], note: params[:note], by: Current.user)
        return render_failure(result) if result.failure?

        render json: detail(@enrollment.reload)
      end

      def contacts
        result = RecordContact.call(enrollment: @enrollment, channel: params[:channel], outcome: params[:outcome], note: params[:note],
                                    by: Current.user)
        return render_failure(result) if result.failure?

        render json: { contact: Json.contact(result.payload[:contact]) }, status: :created
      end

      private

      def scoped_team
        team = uuid?(params[:team_id]) && HealthTeam.find_by(id: params[:team_id])
        return team if Access.panel?(Current.user, team)

        forbid("out_of_scope")
        nil
      end

      def load_enrollment
        @enrollment = uuid?(params[:id]) && CareLineEnrollment.includes(:team, :care_line, :patient).find_by(id: params[:id])
        not_found unless @enrollment && Access.panel?(Current.user, @enrollment.team)
      end

      # patient_id XOR citizen_id; o par precisa ser validado (o paciente
      # nasce por Patients::Resolve, como na sugestão).
      def person_patient
        patient_id, citizen_id = params.values_at(:patient_id, :citizen_id)
        unless patient_id.present? ^ citizen_id.present?
          render_error(:invalid_person)
          return nil
        end
        record = if patient_id.present?
                   uuid?(patient_id) && Patient.find_by(id: patient_id)
                 else
                   uuid?(citizen_id) && Citizen.not_erased.find_by(id: citizen_id)
                 end
        unless record
          not_found
          return nil
        end
        return record if record.is_a?(Patient)

        citizen = record

        resolved = Patients::Resolve.call(citizen)
        return resolved.payload[:patient] if resolved.ok?

        render_error(:citizen_not_verified, :conflict)
        nil
      end

      def detail(enrollment)
        row = Evaluation.enrollments([ enrollment ], today: Time.zone.today).fetch(enrollment.id)
        Trail.viewed!(user: Current.user, patient_id: enrollment.patient_id, citizen_id: nil, enrollment_id: enrollment.id)
        Json.enrollment(enrollment, row: row, last_contact: enrollment.contacts.first)
            .merge(contacts: enrollment.contacts.includes(recorded_by_user: :professional).map { |c| Json.contact(c) })
      end

      def last_contacts(ids)
        return {} if ids.empty?

        CareOutreachContact.where(enrollment_id: ids).select("DISTINCT ON (enrollment_id) care_outreach_contacts.*")
                           .order(:enrollment_id, recorded_at: :desc, id: :desc).index_by(&:enrollment_id)
      end
    end
  end
  ```

  `app/controllers/care_lines/suggestions_controller.rb`:

  ```ruby
  # Sugestões a confirmar (contrato §4). Só pendentes; sem texto clínico.
  module CareLines
    class SuggestionsController < BaseController
      before_action :load_suggestion, only: %i[accept dismiss]

      def index
        team = scoped_team or return
        scope = CareLineSuggestion.pending_suggestions.where(team_id: team.id).includes(:care_line, :patient, :citizen)
        scope = scope.where(care_line_id: params[:care_line_id]) if uuid?(params[:care_line_id])
        limit = Timeline::Query.limit(params[:limit])
        rows = Cursor.before(scope.order(created_at: :desc, id: :desc), Cursor.decode(params[:before]), :created_at).limit(limit + 1).to_a
        more = rows.size > limit
        rows = rows.first(limit)
        payload = rows.map do |suggestion|
          Trail.viewed!(user: Current.user, patient_id: suggestion.patient_id, citizen_id: suggestion.citizen_id, suggestion_id: suggestion.id)
          Json.suggestion(suggestion)
        end
        render json: { suggestions: payload, next_before: more ? Cursor.encode(rows.last.created_at, rows.last.id) : nil }
      rescue Cursor::Invalid
        render_error(:invalid_cursor)
      end

      def count
        team = scoped_team or return
        render json: { count: CareLineSuggestion.pending_suggestions.where(team_id: team.id).count }
      end

      def accept
        team = nil
        unless @suggestion.closure?
          team = uuid?(params[:team_id]) && HealthTeam.find_by(id: params[:team_id])
          return forbid("out_of_scope") unless Access.panel?(Current.user, team)
        end
        result = AcceptSuggestion.call(suggestion: @suggestion, by: Current.user, team: team, anchor_date: params[:anchor_date])
        return render_failure(result) if result.failure?

        enrollment = result.payload[:enrollment] || result.payload[:ended]
        row = Evaluation.enrollments([ enrollment ], today: Time.zone.today).fetch(enrollment.id)
        Trail.viewed!(user: Current.user, patient_id: enrollment.patient_id, citizen_id: nil, enrollment_id: enrollment.id)
        render json: Json.enrollment(enrollment, row: row, last_contact: nil), status: @suggestion.closure? ? :ok : :created
      end

      def dismiss
        result = DismissSuggestion.call(suggestion: @suggestion, reason: params[:reason], by: Current.user)
        return render_failure(result) if result.failure?

        render json: { id: @suggestion.id, status: "dismissed" }
      end

      private

      def scoped_team
        team = uuid?(params[:team_id]) && HealthTeam.find_by(id: params[:team_id])
        return team if Access.panel?(Current.user, team)

        forbid("out_of_scope")
        nil
      end

      def load_suggestion
        @suggestion = uuid?(params[:id]) && CareLineSuggestion.includes(:team).find_by(id: params[:id])
        not_found unless @suggestion && Access.panel?(Current.user, @suggestion.team)
      end
    end
  end
  ```

  `config/routes.rb`, dentro do `scope "/care_lines"` da Task 6:

  ```ruby
    get   "suggestions",                to: "suggestions#index"
    get   "suggestions/count",          to: "suggestions#count"
    post  "suggestions/:id/accept",     to: "suggestions#accept"
    post  "suggestions/:id/dismiss",    to: "suggestions#dismiss"
    get   "enrollments",                to: "enrollments#index"
    get   "enrollments/overdue/count",  to: "enrollments#overdue_count"
    post  "enrollments",                to: "enrollments#create"
    get   "enrollments/:id",            to: "enrollments#show"
    post  "enrollments/:id/end",        to: "enrollments#end_enrollment"
    post  "enrollments/:id/contacts",   to: "enrollments#contacts"
  ```

- [ ] **Step 4: Rode e veja passar.** `test spec/requests/care_lines` → PASS.

- [ ] **Step 5: Commit.**

  ```bash
  git add app/services/care_lines/cursor.rb app/services/care_lines/trail.rb app/services/care_lines/board.rb app/services/care_lines/json.rb app/controllers/care_lines/suggestions_controller.rb app/controllers/care_lines/enrollments_controller.rb config/routes.rb spec/requests/care_lines/enrollments_spec.rb spec/requests/care_lines/suggestions_spec.rb
  git commit -m "feat: add the team care line panel with active search, suggestions and outreach routes

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

## Fatia 5 — Prévia, cidadão, LGPD

### Task 13: Prévia dos indicadores C2–C7 por equipe (F-23.6)

**Files:**
- Create: `app/services/care_lines/preview.rb`, `app/controllers/care_lines/previews_controller.rb`, `spec/services/care_lines/preview_spec.rb`, `spec/requests/care_lines/preview_spec.rb`
- Modify: `config/routes.rb` (`get "preview", to: "previews#show"` no `scope "/care_lines"`)

**Interfaces:**
- Consumes: `CareLines::TeamPopulation.call`, `CareLines::Evaluation.items` (Tasks 8–9); `CareLineTemplate.current` (Task 3).
- Produces: `CareLines::Preview.call(team:, indicator:, today:) → Hash` (forma do contrato §6 com L8: `{ indicator, team: { id, ine, name }, template_version, denominator, score, max_score, concept, practices: [ { key, label, tracked, points, weight, achieved, denominator } ], excluded, estimate: true }`; `score`/`weight` strings com uma casa e ponto; `score` e `concept` nulos com denominador zero; `concept` em `optimal|good|sufficient|regular`); `CareLines::Preview::INDICATORS = %w[C2 C3 C4 C5 C6 C7]`; rota `GET /care_lines/preview?team_id=&indicator=` (`municipal_admin`, `analyst`; sem trilha, sem nome).

- [ ] **Step 1: Testes que falham.** `spec/services/care_lines/preview_spec.rb`:

  ```ruby
  require "rails_helper"

  # Spec §2 decisão 9; plano, Desvio 13. Contas conferidas ao escrever o plano.
  RSpec.describe CareLines::Preview do
    before do
      care_city!
      care_lines_loaded!
    end

    let(:today) { Time.zone.today }
    let(:unit) { care_unit! }
    let(:doctor) { care_doctor!(unit) }
    let!(:team) { care_team!(unit, doctor) }

    it "C4: população elegível (inscrita ou não), pesos redistribuídos sem a visita do ACS, declaração e resolvido fora" do
      enrolled = care_patient!(age: 58, sex: "male", problems: [ %w[ciap2 T90] ], unit: unit, doctor: doctor)
      care_enrollment!(patient: enrolled, line: adopted_line!("diabetes_c4"), team: team, by: doctor)
      loose = care_patient!(age: 62, sex: "female", problems: [ %w[cid10 E119] ], unit: unit, doctor: doctor)
      exam_order!(patient: loose, unit: unit, requester: doctor, sigtap_code: "0202010503", status: :awaiting_authorization)
      cured = care_patient!(age: 50, sex: "male", problems: [ %w[ciap2 T90] ], unit: unit, doctor: doctor)
      resolve_problem!(cured, "T90", by: doctor)
      declared = care_citizen!(age: 40, sex: "female", neighborhood: care_neighborhood!(unit))
      declare!(declared, "diabetes")

      result = described_class.call(team: team, indicator: "C4", today: today)
      expect(result).to include(indicator: "C4", denominator: 2, max_score: 100, estimate: true, template_version: 1,
                                excluded: %w[acs_visit])
      expect(result[:practices].map { |p| p.values_at(:key, :tracked, :weight, :achieved, :denominator) }).to eq([
        [ "c4_a", true, "25.0", 2, 2 ], [ "c4_b", true, "18.8", 2, 2 ], [ "c4_c", true, "18.8", 2, 2 ],
        [ "c4_d", false, "0.0", 0, 2 ], [ "c4_e", true, "18.8", 1, 2 ], [ "c4_f", true, "18.8", 0, 2 ]
      ])
      # (62,5 + 81,25) / 2 = 71,875
      expect(result).to include(score: "71.9", concept: "good")
    end

    it "C7: denominador por prática (faixa); vacina HPV fora" do
      neighborhood = care_neighborhood!(unit)
      care_citizen!(age: 30, sex: "female", neighborhood: neighborhood)
      care_citizen!(age: 12, sex: "female", neighborhood: neighborhood)
      care_patient!(age: 55, sex: "female", problems: [ %w[ciap2 W14] ], unit: unit, doctor: doctor)
      result = described_class.call(team: team, indicator: "C7", today: today)
      expect(result[:denominator]).to eq(3)
      expect(result[:excluded]).to eq(%w[vaccine])
      expect(result[:practices].map { |p| p.values_at(:key, :weight, :achieved, :denominator) }).to eq([
        [ "c7_a", "28.6", 0, 2 ], [ "c7_b", "0.0", 0, 1 ], [ "c7_c", "42.9", 1, 2 ], [ "c7_d", "28.6", 0, 1 ]
      ])
      # 42,857 × 1/2 = 21,43
      expect(result).to include(score: "21.4", concept: "regular")
    end

    it "equipe sem ninguém elegível: denominador 0, sem nota" do
      expect(described_class.call(team: team, indicator: "C6", today: today)).to include(denominator: 0, score: nil, concept: nil)
    end
  end
  ```

  `spec/requests/care_lines/preview_spec.rb`:

  ```ruby
  require "rails_helper"

  # Contrato §6: admin municipal e analista; agregado, sem nome; sem trilha.
  RSpec.describe "Linhas de cuidado — prévia", type: :request do
    before do
      care_city!
      care_lines_loaded!
    end

    let(:unit) { care_unit! }
    let(:doctor) { care_doctor!(unit) }
    let!(:team) { care_team!(unit, doctor) }

    it "analista lê, sem nome e sem trilha; profissional não; indicador e equipe validados" do
      care_patient!(age: 70, sex: "female", problems: [ %w[ciap2 K86] ], unit: unit, doctor: doctor)
      sign_in_as(staff_with("analise-#{SecureRandom.hex(3)}@cidade.gov.br", "analyst"))
      get "/care_lines/preview", params: { team_id: team.id, indicator: "C6" }
      expect(response).to have_http_status(:ok)
      expect(json_body).to include("indicator" => "C6", "denominator" => 1, "estimate" => true)
      expect(response.body).not_to match(/display_name|patient_id|citizen_id/)
      expect(DomainEvent.where(name: "clinical_record.viewed").select { |e| e.payload["access"] == "care_line" }).to be_empty
      get "/care_lines/preview", params: { team_id: team.id, indicator: "C1" }
      expect(json_body).to eq("error" => "invalid_indicator")
      get "/care_lines/preview", params: { team_id: SecureRandom.uuid, indicator: "C6" }
      expect(response).to have_http_status(:not_found)
      sign_in_as(doctor)
      get "/care_lines/preview", params: { team_id: team.id, indicator: "C6" }
      expect(response).to have_http_status(:forbidden)
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar.** `test spec/services/care_lines/preview_spec.rb spec/requests/care_lines/preview_spec.rb` → FAIL (`uninitialized constant CareLines::Preview`).

- [ ] **Step 3: Implemente.** `app/services/care_lines/preview.rb`:

  ```ruby
  # Prévia de C2–C7 por equipe (ADR 0040; spec §2 decisão 9; plano, Desvio
  # 13). ESTIMATIVA: regras da versão mais recente do template; população
  # elegível da equipe, inscrita ou não; práticas não rastreadas fora, com os
  # pesos restantes redistribuídos; denominador por prática (faixa da C7).
  # Agregado, sem nome. O oficial é do SIAPS.
  module CareLines
    module Preview
      INDICATORS = %w[C2 C3 C4 C5 C6 C7].freeze
      CONCEPTS = [ [ 75, "optimal" ], [ 50, "good" ], [ 25, "sufficient" ] ].freeze

      module_function

      def call(team:, indicator:, today:)
        key = Catalog::INDICATORS.key(indicator)
        template = CareLineTemplate.current(key)
        line = CareLine.find_by(source: "template", template_key: key)
        practices = template.practices
        population = TeamPopulation.call(team: team, line: line, rules: template.eligibility, on: today)
        items = population.map do |person, enrollment|
          { key: person.key, practices: practices, person: person,
            anchors: { "enrollment" => enrollment&.enrolled_at&.in_time_zone&.to_date, "birth" => person.birth_date, "lmp" => enrollment&.anchor_on } }
        end
        rows = Evaluation.items(items, today: today).values
        tracked_points = practices.reject { |p| Catalog.untracked?(p["source"]) }.sum { |p| p["points"].to_i }

        per_practice = practices.map do |practice|
          statuses = rows.filter_map { |row| row.practices.find { |s| s.key == practice["key"] } }
          tracked = !Catalog.untracked?(practice["source"])
          weight = tracked && tracked_points.positive? ? practice["points"].to_i * 100.0 / tracked_points : 0.0
          { key: practice["key"], label: practice["label"], tracked: tracked, points: practice["points"], weight_value: weight,
            weight: format("%.1f", weight), achieved: statuses.count(&:met), denominator: statuses.size }
        end
        score = rows.empty? ? nil : per_practice.sum { |p| p[:denominator].zero? ? 0 : p[:weight_value] * p[:achieved] / p[:denominator] }

        { indicator: indicator, team: { id: team.id, ine: team.ine, name: team.name }, template_version: template.version,
          denominator: rows.size, score: score && format("%.1f", score), max_score: 100, concept: score && concept(score),
          practices: per_practice.map { |p| p.except(:weight_value) },
          excluded: practices.select { |p| Catalog.untracked?(p["source"]) }.map { |p| p["source"].split(":").first }.uniq,
          estimate: true }
      end

      def concept(score) = CONCEPTS.find { |limit, _| score > limit }&.last || "regular"
    end
  end
  ```

  `app/controllers/care_lines/previews_controller.rb`:

  ```ruby
  # Prévia (contrato §6): municipal_admin e analyst; agregado sem nome; sem
  # trilha (não há leitura nominal).
  module CareLines
    class PreviewsController < BaseController
      before_action { forbid unless Access.preview?(Current.user) }

      def show
        return render_error(:invalid_indicator) unless Preview::INDICATORS.include?(params[:indicator])

        team = uuid?(params[:team_id]) && HealthTeam.find_by(id: params[:team_id])
        return not_found unless team

        render json: Preview.call(team: team, indicator: params[:indicator], today: Time.zone.today)
      end
    end
  end
  ```

- [ ] **Step 4: Rode e veja passar.** `test spec/services/care_lines/preview_spec.rb spec/requests/care_lines/preview_spec.rb` → PASS.

- [ ] **Step 5: Commit.**

  ```bash
  git add app/services/care_lines/preview.rb app/controllers/care_lines/previews_controller.rb config/routes.rb spec/services/care_lines/preview_spec.rb spec/requests/care_lines/preview_spec.rb
  git commit -m "feat: add the per-team C2-C7 indicator preview labeled as an estimate

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 14: Cidadão — condição declarada, "Meu acompanhamento" e triagem periódica vencida (F-23.7, F-23.8)

**Files:**
- Create: `app/commands/care_lines/declare_condition.rb`, `app/commands/care_lines/withdraw_condition.rb`, `app/services/care_lines/citizen_view.rb`, `app/controllers/citizen_api/declared_conditions_controller.rb`, `app/controllers/citizen_api/care_controller.rb`, `spec/requests/citizen_api/declared_conditions_spec.rb`, `spec/requests/citizen_api/care_spec.rb`, `spec/services/triages/offer_overdue_spec.rb`
- Modify: `app/services/triages/offer.rb` (`Item` com `retake_after_days` e `overdue?`), `app/services/triages/catalog.rb` (`item_json` com `overdue`/`last_done_at`), `config/routes.rb` (escopo `/citizen`)

**Interfaces:**
- Consumes: `CareLines::Evaluation.enrollments` (Task 8); `Timeline::Person.for_citizen`, `Timeline::Trail.viewed!` (existentes); `Triages::Offer.for` (existente).
- Produces (contrato §7 com L9): `GET /citizen/declared_conditions?citizen_id=` → `{ conditions: [ { condition, declared_at } ] }`; `POST /citizen/declared_conditions` `{ citizen_id, condition }` → 201 `{ conditions }`; `DELETE /citizen/declared_conditions?citizen_id=&condition=` → `{ conditions }`; `GET /citizen/care?citizen_id=` → `{ lines: [ { name, items: [ { label, status, due_on } ] } ], overdue_triages: [ { protocol_name, title, last_done_at } ] }`; erros `not_found` (par de outro celular), 403 `not_verified`, 422 `invalid_condition`, 403 `feature_disabled`; eventos `citizen_condition.declared`/`.withdrawn` (`citizen_declared_condition_id`, `citizen_id`); `Triages::Offer::Item#retake_after_days`, `#overdue?`; no catálogo do cidadão (`GET /citizen/people/:id/catalog`), toda oferta com intervalo de repetição ganha `overdue` e `last_done_at` (sem interruptor: módulo 15 não tem — Desvio 11).

- [ ] **Step 1: Testes que falham.** `spec/services/triages/offer_overdue_spec.rb`:

  ```ruby
  require "rails_helper"

  # F-23.8 (ADR 0040; ADR 0027): oferta com intervalo de repetição passado.
  RSpec.describe Triages::Offer, "triagem periódica vencida" do
    let(:on) { Date.new(2026, 10, 22) }
    let(:protocols) do
      [ { name: "saude-idoso", offer: { "title" => "Saúde do idoso", "retake_after_days" => 180 } },
        { name: "rastreio-sem-intervalo", offer: { "title" => "Rastreio" } } ]
    end

    def evaluate(last_completed)
      described_class.evaluate(protocols: protocols, rows: {}, context: {}, last_completed: last_completed, on: on).index_by(&:protocol_name)
    end

    it "vencida: feita há mais que o intervalo; recente: não; nunca feita: não vencida; sem intervalo: sem campo" do
      overdue = evaluate("saude-idoso" => on - 200, "rastreio-sem-intervalo" => on - 400)
      expect(overdue["saude-idoso"]).to have_attributes(state: "available", retake_after_days: 180)
      expect(overdue["saude-idoso"].overdue?).to be(true)
      expect(Triages::Catalog.item_json(overdue["saude-idoso"])).to include(overdue: true, last_done_at: (on - 200).iso8601)
      expect(Triages::Catalog.item_json(overdue["rastreio-sem-intervalo"])).not_to have_key(:overdue)
      expect(evaluate("saude-idoso" => on - 100)["saude-idoso"].overdue?).to be(false)
      never = evaluate({})["saude-idoso"]
      expect(never.overdue?).to be(false)
      expect(Triages::Catalog.item_json(never)).to include(overdue: false, last_done_at: nil)
    end
  end
  ```

  `spec/requests/citizen_api/declared_conditions_spec.rb`:

  ```ruby
  require "rails_helper"

  # Contrato §7: só par validado; cifrada; "nenhuma" retira as outras.
  RSpec.describe "Cidadão — condição declarada", type: :request do
    before { care_city! }

    let(:citizen) { care_citizen!(age: 40, sex: "female") }

    def conditions = json_body["conditions"].map { |c| c["condition"] }

    it "declara, lista, retira; none retira as outras; publica só ids" do
      sign_in_citizen(citizen.phone)
      json_post "/citizen/declared_conditions", citizen_id: citizen.id, condition: "diabetes"
      expect(response).to have_http_status(:created)
      json_post "/citizen/declared_conditions", citizen_id: citizen.id, condition: "hypertension"
      json_post "/citizen/declared_conditions", citizen_id: citizen.id, condition: "diabetes"
      expect(conditions).to contain_exactly("diabetes", "hypertension")
      expect(DomainEvent.where(name: "citizen_condition.declared").last.payload.keys).to match_array(%w[citizen_declared_condition_id citizen_id])
      delete "/citizen/declared_conditions", params: { citizen_id: citizen.id, condition: "hypertension" }
      expect(conditions).to eq(%w[diabetes])
      json_post "/citizen/declared_conditions", citizen_id: citizen.id, condition: "none"
      expect(conditions).to eq(%w[none])
      expect(CitizenDeclaredCondition.where(citizen: citizen).count).to eq(3) # diabetes, hypertension, none
      expect(CitizenDeclaredCondition.current.where(citizen: citizen).map(&:condition)).to eq(%w[none])
      json_post "/citizen/declared_conditions", citizen_id: citizen.id, condition: "asthma"
      expect(json_body).to eq("error" => "invalid_condition")
    end

    it "par declarado → 403 not_verified; par de outro celular → 404; care_lines desligado → 403" do
      declared = profiled_citizen!(age: 30, phone: "+5541990000123")
      sign_in_citizen(declared.phone)
      get "/citizen/declared_conditions", params: { citizen_id: declared.id }
      expect(json_body).to eq("error" => "not_verified")
      get "/citizen/declared_conditions", params: { citizen_id: citizen.id }
      expect(response).to have_http_status(:not_found)
      Platform::Features.set!(city: Current.city, key: "care_lines", enabled: false, maintainer: ledi_maintainer!)
      get "/citizen/declared_conditions", params: { citizen_id: declared.id }
      expect(json_body).to eq("error" => "feature_disabled", "feature" => "care_lines")
    end
  end
  ```

  `spec/requests/citizen_api/care_spec.rb`:

  ```ruby
  require "rails_helper"

  # Contrato §7: "Meu acompanhamento" — só linhas inscritas, linguagem
  # simples, sem código nem motivo; triagem vencida quando care_lines ligado.
  RSpec.describe "Cidadão — meu acompanhamento", type: :request do
    before { care_city! }

    let(:unit) { care_unit! }
    let(:doctor) { care_doctor!(unit) }
    let!(:team) { care_team!(unit, doctor) }

    it "linhas inscritas com o que está para fazer; sugestão e não rastreado não aparecem; trilha citizen" do
      patient = care_patient!(age: 58, sex: "male", problems: [ %w[ciap2 T90] ], unit: unit, doctor: doctor)
      citizen = Citizen.find_by!(patient_id: patient.id)
      care_enrollment!(patient: patient, line: adopted_line!("diabetes_c4"), team: team, by: doctor)
      CareLineSuggestion.create!(care_line: adopted_line!("hypertension_c5"), team: team, patient: patient, reason: "problem")
      active_protocol!("saude-adulto", offer: { "title" => "Saúde do adulto", "retake_after_days" => 180 })
      completed_triage!(citizen, "saude-adulto", at: 200.days.ago)
      sign_in_citizen(citizen.phone)
      get "/citizen/care", params: { citizen_id: citizen.id }
      expect(json_body["lines"].map { |l| l["name"] }).to eq([ "Pessoa com diabetes" ])
      items = json_body["lines"].first["items"]
      expect(items.map { |i| [ i["label"], i["status"] ] }).to eq([
        [ "Consulta de acompanhamento", "ok" ], [ "Medir a pressão", "ok" ], [ "Medir peso e altura", "ok" ],
        [ "Exame de hemoglobina glicada", "overdue" ], [ "Avaliação dos pés", "overdue" ]
      ])
      expect(response.body).not_to match(/T90|c4_|resolved_problem|hipertens/i)
      expect(json_body["overdue_triages"]).to eq([ { "protocol_name" => "saude-adulto", "title" => "Saúde do adulto",
                                                     "last_done_at" => 200.days.ago.to_date.iso8601 } ])
      expect(DomainEvent.where(name: "clinical_record.viewed").last.payload).to include("access" => "citizen", "patient_id" => patient.id)
    end

    it "par validado sem paciente: sem linhas; par declarado: 403" do
      citizen = care_citizen!(age: 30, sex: "female")
      sign_in_citizen(citizen.phone)
      get "/citizen/care", params: { citizen_id: citizen.id }
      expect(json_body).to eq("lines" => [], "overdue_triages" => [])
      declared = profiled_citizen!(age: 30, phone: "+5541990000124")
      sign_in_citizen(declared.phone)
      get "/citizen/care", params: { citizen_id: declared.id }
      expect(json_body).to eq("error" => "not_verified")
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar.** `test spec/services/triages/offer_overdue_spec.rb spec/requests/citizen_api/declared_conditions_spec.rb spec/requests/citizen_api/care_spec.rb` → FAIL.

- [ ] **Step 3: Triagem vencida (F-23.8).** Em `app/services/triages/offer.rb`, troque a definição de `Item` por:

  ```ruby
      Item = Data.define(:protocol_name, :title, :summary, :state, :position, :last_completed_on, :next_available_on,
                         :suggestion_only, :retake_after_days) do
        def initialize(retake_after_days: nil, **rest) = super
        def available? = state == "available"
        def recent? = state == "recent"
        # ADR 0040 (F-23.8): com intervalo de repetição, já feita e de novo em
        # oferta = vencida (available só depois de última + intervalo).
        def overdue? = available? && !last_completed_on.nil? && !retake_after_days.nil?
      end
  ```
  e, em `item(name, offer, row, last_on, on)`, passe `retake_after_days: days.is_a?(Integer) && days.positive? ? days : nil` no `Item.new`. Em `app/services/triages/catalog.rb`:

  ```ruby
      def item_json(item)
        json = { protocol_name: item.protocol_name, title: item.title, summary: item.summary }
        return json if item.retake_after_days.nil?

        json.merge(overdue: item.overdue?, last_done_at: item.last_completed_on&.iso8601)
      end
  ```
  Rode `test spec/services/triages spec/requests/citizen_api/people_spec.rb` — specs que comparam o JSON do catálogo com `eq` ganham as duas chaves só nos protocolos com `retake_after_days`; ajuste essas expectativas (somando as chaves), nunca o código.

- [ ] **Step 4: Condição declarada.** `app/commands/care_lines/declare_condition.rb`:

  ```ruby
  # Condição de saúde declarada pelo cidadão (ADR 0040; contrato §7). Nunca
  # é diagnóstico. Repetir a mesma não grava de novo; "none" retira as
  # outras; uma condição retira o "none". Só acréscimo (retirar grava
  # withdrawn_at). Eventos só com ids; o consumidor sugere a linha.
  module CareLines
    module DeclareCondition
      module_function

      def call(citizen:, condition:)
        return Result.fail(:invalid_condition) unless Catalog::CONDITIONS.include?(condition)

        ApplicationRecord.transaction do
          citizen.lock!
          current = CitizenDeclaredCondition.current.where(citizen_id: citizen.id).to_a
          next Result.ok(condition: current.find { |c| c.condition == condition }) if current.any? { |c| c.condition == condition }

          stale = condition == "none" ? current : current.select { |c| c.condition == "none" }
          stale.each { |row| WithdrawCondition.withdraw!(row) }
          row = CitizenDeclaredCondition.create!(citizen: citizen, condition: condition, declared_at: Time.current)
          DomainEvents.publish("citizen_condition.declared", citizen_declared_condition_id: row.id, citizen_id: citizen.id)
          Result.ok(condition: row)
        end
      end
    end
  end
  ```

  `app/commands/care_lines/withdraw_condition.rb`:

  ```ruby
  module CareLines
    module WithdrawCondition
      module_function

      def call(citizen:, condition:)
        ApplicationRecord.transaction do
          citizen.lock!
          row = CitizenDeclaredCondition.current.where(citizen_id: citizen.id).to_a.find { |c| c.condition == condition }
          next Result.fail(:not_found) unless row

          withdraw!(row)
          Result.ok(condition: row)
        end
      end

      def withdraw!(row)
        row.update!(withdrawn_at: Time.current)
        DomainEvents.publish("citizen_condition.withdrawn", citizen_declared_condition_id: row.id, citizen_id: row.citizen_id)
      end
    end
  end
  ```

  `app/controllers/citizen_api/declared_conditions_controller.rb`:

  ```ruby
  #   GET    /citizen/declared_conditions?citizen_id=
  #   POST   /citizen/declared_conditions   { citizen_id, condition }
  #   DELETE /citizen/declared_conditions?citizen_id=&condition=
  # Condição declarada (ADR 0040; contrato §7): só par validado do celular
  # da sessão; care_lines ligado e utilizável.
  module CitizenApi
    class DeclaredConditionsController < BaseController
      before_action :require_care_lines!
      before_action :load_citizen

      def index = render(json: payload)

      def create
        result = CareLines::DeclareCondition.call(citizen: @citizen, condition: params[:condition])
        return render_error(result.reason, :unprocessable_entity) if result.failure?

        render json: payload, status: :created
      end

      def destroy
        result = CareLines::WithdrawCondition.call(citizen: @citizen, condition: params[:condition])
        return render_error(result.reason, :not_found) if result.failure?

        render json: payload
      end

      private

      def require_care_lines!
        return if CareLines::Gate.usable?(Current.city)

        render json: { error: "feature_disabled", feature: CareLines::Gate::KEY }, status: :forbidden
      end

      def load_citizen
        @citizen = current_citizen_session.citizens.find_by(id: params[:citizen_id])
        return render_error("not_found", :not_found) unless @citizen

        render_error("not_verified", :forbidden) unless @citizen.verification_level_verified?
      end

      def payload
        rows = CitizenDeclaredCondition.current.where(citizen_id: @citizen.id).order(:declared_at).to_a
        { conditions: rows.map { |row| { condition: row.condition, declared_at: row.declared_at.iso8601 } } }
      end
    end
  end
  ```

- [ ] **Step 5: "Meu acompanhamento".** `app/services/care_lines/citizen_view.rb`:

  ```ruby
  # "Meu acompanhamento" (ADR 0040; spec §6; contrato §7): só as linhas em
  # que a pessoa está inscrita; o que está para fazer em linguagem simples
  # (citizen_label), sem código, sem sugestão, sem motivo; não rastreado
  # não aparece. Triagens periódicas vencidas (módulo 15) junto.
  module CareLines
    module CitizenView
      module_function

      def call(citizen:, person:, today:)
        enrollments = person.patient_id ? CareLineEnrollment.active_enrollments.where(patient_id: person.patient_id).order(:enrolled_at).to_a : []
        rows = Evaluation.enrollments(enrollments, today: today)
        lines = enrollments.map do |enrollment|
          items = rows.fetch(enrollment.id).practices.select(&:tracked).map do |status|
            { label: status.citizen_label || status.label, status: status.status, due_on: status.due_on&.iso8601 }
          end
          { name: enrollment.care_line.name, items: items }
        end
        overdue = Triages::Offer.for(citizen: citizen, on: today).select(&:overdue?).map do |item|
          { protocol_name: item.protocol_name, title: item.title, last_done_at: item.last_completed_on.iso8601 }
        end
        { lines: lines, overdue_triages: overdue }
      end
    end
  end
  ```

  `app/controllers/citizen_api/care_controller.rb`:

  ```ruby
  #   GET /citizen/care?citizen_id=
  # "Meu acompanhamento" (ADR 0040; contrato §7): só par validado do celular
  # da sessão (senão 404 / 403 not_verified); care_lines utilizável. Leitura
  # com trilha citizen quando há paciente.
  module CitizenApi
    class CareController < BaseController
      def show
        return render(json: { error: "feature_disabled", feature: CareLines::Gate::KEY }, status: :forbidden) unless CareLines::Gate.usable?(Current.city)

        citizen = current_citizen_session.citizens.find_by(id: params[:citizen_id])
        return render_error("not_found", :not_found) unless citizen
        return render_error("not_verified", :forbidden) unless citizen.verification_level_verified?

        person = Timeline::Person.for_citizen(citizen)
        Timeline::Trail.viewed!(person: person, access: "citizen") if person.patient_id
        render json: CareLines::CitizenView.call(citizen: citizen, person: person, today: Time.zone.today)
      end
    end
  end
  ```

  `config/routes.rb`, no `scope "/citizen"`:

  ```ruby
      # Linhas de cuidado (ADR 0040; contrato §7).
      get    "declared_conditions", to: "declared_conditions#index"
      post   "declared_conditions", to: "declared_conditions#create"
      delete "declared_conditions", to: "declared_conditions#destroy"
      get    "care",                to: "care#show"
  ```

- [ ] **Step 6: Rode e veja passar.** `test spec/services/triages spec/requests/citizen_api` → PASS.

- [ ] **Step 7: Commit.**

  ```bash
  git add app/commands/care_lines/declare_condition.rb app/commands/care_lines/withdraw_condition.rb app/services/care_lines/citizen_view.rb app/controllers/citizen_api/declared_conditions_controller.rb app/controllers/citizen_api/care_controller.rb app/services/triages/offer.rb app/services/triages/catalog.rb config/routes.rb spec/services/triages/offer_overdue_spec.rb spec/requests/citizen_api/declared_conditions_spec.rb spec/requests/citizen_api/care_spec.rb
  git commit -m "feat: let verified citizens declare health conditions and see their care follow-up and overdue periodic triages

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```
  (Se o Step 3 ajustou expectativas de specs do módulo 15, some esses arquivos ao `git add`.)

### Task 15: Exclusão LGPD e invariantes do ADR 0040

**Files:**
- Modify: `app/commands/citizens/request_erasure.rb` (`examined_patients?` ganha a inscrição), `app/commands/citizens/erase.rb` (apaga condição declarada e sugestões)
- Create: `spec/commands/citizens/care_lines_erasure_spec.rb`, `spec/invariants/care_lines_invariants_spec.rb`

**Interfaces:**
- Consumes: `Citizens::RequestErasure.call(cpf:, document_checked:, by:)`, `Citizens::Erase.call(request:, by:)` (existentes); comandos das Tasks 11 e 14.
- Produces: regra única de retenção com `CareLineEnrollment` (a inscrição e os contatos que pendem dela são documento clínico — regra da api#84); na confirmação, `CitizenDeclaredCondition` e `CareLineSuggestion` do par (e as do paciente não retido) são apagadas.

- [ ] **Step 1: Testes que falham.** `spec/commands/citizens/care_lines_erasure_spec.rb`:

  ```ruby
  require "rails_helper"

  # ADR 0040 (spec §7): condição declarada é apagada e não retém; inscrição e
  # contatos retêm. Review Focus 5.
  RSpec.describe "Exclusão LGPD — linhas de cuidado" do
    before { care_city! }

    let(:unit) { care_unit! }
    let(:doctor) { care_doctor!(unit) }
    let!(:team) { care_team!(unit, doctor) }

    it "condição declarada não retém; a confirmação apaga declarações e sugestões do par" do
      citizen = care_citizen!(age: 40, sex: "female", neighborhood: care_neighborhood!(unit))
      CareLines::DeclareCondition.call(citizen: citizen, condition: "diabetes")
      CareLineSuggestion.create!(care_line: adopted_line!("diabetes_c4"), team: team, citizen: citizen, reason: "declared")
      request = Citizens::RequestErasure.call(cpf: citizen.cpf, document_checked: true, by: verifier!).payload[:request]
      expect(request.status).to eq("pending")
      expect(Citizens::Erase.call(request: request, by: care_admin!)).to be_ok
      expect(CitizenDeclaredCondition.where(citizen_id: citizen.id)).not_to exist
      expect(CareLineSuggestion.where(citizen_id: citizen.id)).not_to exist
    end

    it "inscrição (e contato) retém o pedido; nada se apaga" do
      citizen = care_citizen!(age: 72, sex: "female")
      suggestion = CareLineSuggestion.create!(care_line: adopted_line!("elderly_c6"), team: team, citizen: citizen, reason: "age_sex")
      enrollment = CareLines::AcceptSuggestion.call(suggestion: suggestion, team: team, by: doctor).payload[:enrollment]
      CareLines::RecordContact.call(enrollment: enrollment, channel: "phone", outcome: "reached", note: "Ok", by: doctor)
      CareLines::DeclareCondition.call(citizen: citizen.reload, condition: "hypertension")
      request = Citizens::RequestErasure.call(cpf: citizen.cpf, document_checked: true, by: verifier!).payload[:request]
      expect(request.status).to eq("retained")
      expect(CareLineEnrollment.exists?(enrollment.id)).to be(true)
      expect(CareOutreachContact.where(enrollment_id: enrollment.id).count).to eq(1)
    end
  end
  ```

  `spec/invariants/care_lines_invariants_spec.rb`:

  ```ruby
  require "rails_helper"

  # Invariantes do ADR 0040.
  RSpec.describe "Invariantes das linhas de cuidado (ADR 0040)" do
    def files(glob) = Dir[Rails.root.join(glob)].map { |f| Pathname(f).relative_path_from(Rails.root).to_s }

    it "nenhuma inscrição sem confirmação: só CareLines::Enroll cria inscrição" do
      writers = (files("app/**/*.rb") + files("lib/**/*.rb")).select { |f| File.read(Rails.root.join(f)).include?("CareLineEnrollment.create") }
      expect(writers).to eq([ "app/commands/care_lines/enroll.rb" ])
    end

    it "boa prática sem fonte no sistema nunca conta como atraso" do
      today = Date.new(2026, 10, 22)
      CareLines::Templates::Load.entries.each do |entry|
        untracked = entry["practices"].select { |p| CareLines::Catalog.untracked?(p["source"]) }
        practices = CareLines::Rules.practices(untracked.presence || entry["practices"].first(1)).payload[:practices]
        statuses = CareLines::Evaluate.call(practices: practices, person: CareLines::Person.new(birth_date: today - 20_000, sex: "female"),
                                            anchors: { "birth" => today - 400, "lmp" => today - 100, "enrollment" => today }, events: {}, today: today)
        statuses.select { |s| CareLines::Catalog.untracked?(s.source) }.each { |s| expect(s.status).to eq("untracked") }
      end
    end

    it "o cidadão nunca recebe sugestão nem motivo: as rotas /citizen não leem sugestões" do
      citizen_code = files("app/controllers/citizen_api/**/*.rb").map { |f| File.read(Rails.root.join(f)) }.join
      citizen_code += File.read(Rails.root.join("app/services/care_lines/citizen_view.rb"))
      expect(citizen_code).not_to match(/CareLineSuggestion|ended_reason|ended_note|dismiss_reason/)
    end

    it "eventos das linhas só com ids; nota, condição e DUM nunca no log" do
      care_city!
      unit = care_unit!
      doctor = care_doctor!(unit)
      team = care_team!(unit, doctor)
      adopted_line!("pregnancy_c3")
      citizen = care_citizen!(age: 27, sex: "female", neighborhood: care_neighborhood!(unit))
      log = capture_log do
        CareLines::DeclareCondition.call(citizen: citizen, condition: "pregnancy")
        CareLines::Suggest.for_citizen(citizen.id)
        suggestion = CareLineSuggestion.sole
        enrollment = CareLines::AcceptSuggestion.call(suggestion: suggestion, team: team, by: doctor,
                                                      anchor_date: (Time.zone.today - 63).iso8601).payload[:enrollment]
        CareLines::RecordContact.call(enrollment: enrollment, channel: "phone", outcome: "reached", note: "Sangramento leve relatado", by: doctor)
        CareLines::EndEnrollment.call(enrollment: enrollment, reason: "pregnancy_ended", note: "Perda gestacional na 9ª semana", by: doctor)
      end
      names = %w[care_line_suggestion.created care_line_suggestion.accepted care_line_enrollment.created care_outreach.recorded
                 care_line_enrollment.ended citizen_condition.declared]
      events = DomainEvent.where(name: names).to_a
      expect(events.map(&:name).uniq).to match_array(names)
      events.each { |e| expect(e.payload.keys).to all(end_with("_id")), e.name }
      expect(log).not_to match(/Sangramento|Perda gestacional|#{(Time.zone.today - 63).iso8601}/)
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar.** `test spec/commands/citizens/care_lines_erasure_spec.rb spec/invariants/care_lines_invariants_spec.rb` → FAIL (o pedido com inscrição nasce `pending`; as declarações sobrevivem à exclusão).

- [ ] **Step 3: Implemente.** Em `app/commands/citizens/request_erasure.rb`, no `examined_patients?` (e no comentário acima dele):

  ```ruby
      # ADR 0040: a inscrição em linha de cuidado (e os contatos de busca
      # ativa, que pendem dela) também é registro clínico retido. A condição
      # declarada pelo cidadão não retém (é apagada na exclusão).
      def examined_patients?(ids)
        ExamOrder.where(patient_id: ids).exists? || ExamResult.where(patient_id: ids).exists? ||
          LabRequisition.where(patient_id: ids).exists? || PerformedProcedure.where(patient_id: ids).exists? ||
          CareLineEnrollment.where(patient_id: ids).exists?
      end
  ```

  Em `app/commands/citizens/erase.rb`, em `erase_pair`, logo depois de `TriageSuggestion.where(citizen_id: citizen.id).delete_all`:

  ```ruby
        # ADR 0040: a condição declarada (dado pessoal, nunca diagnóstico) e as
        # sugestões de linha de cuidado do par (os triggers deixam o DELETE
        # passar). A inscrição retém e nunca chega aqui.
        CitizenDeclaredCondition.where(citizen_id: citizen.id).delete_all
        CareLineSuggestion.where(citizen_id: citizen.id).delete_all
  ```
  e, em `attempt`, logo depois de `locked_patients.each { |patient| patient.update!(Patient::DEMOGRAPHICS.index_with(nil)) }`:

  ```ruby
        # ADR 0040: sugestões do paciente não retido (sem inscrição, por definição).
        CareLineSuggestion.where(patient_id: locked_patients.map(&:id)).delete_all
  ```
  Atualize o comentário de cabeçalho do `erase.rb` com a linha `# ADR 0040: e a condição declarada e as sugestões de linha de cuidado.`

- [ ] **Step 4: Rode e veja passar.** `test spec/commands/citizens spec/invariants/care_lines_invariants_spec.rb spec/requests/erasure_requests_spec.rb` → PASS.

- [ ] **Step 5: Commit.**

  ```bash
  git add app/commands/citizens/request_erasure.rb app/commands/citizens/erase.rb spec/commands/citizens/care_lines_erasure_spec.rb spec/invariants/care_lines_invariants_spec.rb
  git commit -m "feat: retain erasure on care line enrollments and erase declared conditions and suggestions

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

---

## Fatia 6 — Semente, revisão e rollout

### Task 16: Semente de dev realista em Curitiba

**Files:**
- Create: `lib/care_lines_crew.rb`, `spec/lib/care_lines_crew_spec.rb`
- Modify: `db/seeds.rb` (plataforma: `CareLinesCrew.seed_platform!`; cidade: `CareLinesCrew.seed_current_city` depois do `BillingCrew`)

**Interfaces:**
- Consumes: `CareLines::Templates::Load`, `CareLines::SaveLine`, `CareLines::Suggest`, `CareLines::AcceptSuggestion`, `CareLines::DeclareCondition` (Tasks 3, 6, 10, 11, 14); `Citizens::RegisterPerson`, `Citizens::IssueVerificationCode`, `Citizens::Verify`, `Consultations::{Start,SaveDraft,Finalize}`, `Attendances::{CheckIn,Call}` (existentes, como no `ClinicalRecordCrew`); `ScreeningCrew.cpf_for` (privado, via `send`).
- Produces: `CareLinesCrew.seed_platform! → { created: Integer }`; `CareLinesCrew.seed_current_city(slug:, ddd:) → { switch:, people: Array<String>, pending: Integer }`.

O que a semente deixa (dev fictício que imita o real — memória "Semente de dev realista"; telefones `+55 41 93333-000n`, CPF com dígito válido):

| Pessoa | Situação | Para a prova no navegador |
|---|---|---|
| José Carlos Pereira, 58, masc. | consulta de hoje na UBS Jardim das Flores com a médica (T90 diabetes + K86 hipertensão, PA, peso e altura); **inscrito na linha de diabetes** (aceito pela enfermeira); sem HbA1c nem exame dos pés | busca ativa (HbA1c e pés vencidos), registro de contato e agendamento; a sugestão de **hipertensão** fica pendente para a confirmação |
| Ana Beatriz Souza, 27, fem. | consulta de hoje com a enfermeira (W78 gravidez); **inscrita na linha da gestante com DUM = hoje − 70**; sugestão `age_sex` da linha da mulher pendente | gestante com âncora pela DUM (1ª consulta em dia; 7 consultas e testes do 1º trimestre a cobrar) |
| Miguel Oliveira Santos, 5 meses, masc. | par validado; **inscrito na linha da criança** (paciente criado na confirmação) | criança com 1ª consulta até 30 dias vencida |
| Maria das Graças Lima, 72, fem. | par validado no bairro da UBS, **nunca consultou**; sugestão `age_sex` da linha da pessoa idosa pendente | confirmar cria o paciente (`Patients::Resolve`) |
| Rafael Costa Almeida, 34, masc. | par validado no bairro; declarou `mental_health`; sugestão `declared` da linha própria **"Saúde mental — acompanhamento"** pendente | linha própria + condição declarada; "Meu acompanhamento" no wpda depois de inscrito |

- [ ] **Step 1: Teste que falha** — `spec/lib/care_lines_crew_spec.rb`:

  ```ruby
  require "rails_helper"
  require Rails.root.join("lib/signature_crew").to_s
  require Rails.root.join("lib/professional_crew").to_s
  require Rails.root.join("lib/screening_crew").to_s
  require Rails.root.join("lib/clinical_record_crew").to_s
  require Rails.root.join("lib/care_lines_crew").to_s

  # Semente de dev do módulo 23 no banco de teste (nunca no de dev), com o
  # elenco que ela exige (mesmo setup do spec do ClinicalRecordCrew): Curitiba
  # liga care_lines, adota as seis linhas e a própria de saúde mental e semeia
  # as cinco pessoas. Idempotente.
  RSpec.describe CareLinesCrew do
    it "o prefixo do telefone não aparece em nenhum outro lib/*_crew.rb" do
      others = Dir[Rails.root.join("lib/*_crew.rb")].reject { |f| f.end_with?("/care_lines_crew.rb") }
      expect(others.select { |f| File.read(f).include?(described_class::PHONE_PREFIX) }).to eq([])
    end

    describe ".seed_current_city" do
      let!(:city) { clinical_city!(record_mode: "off", enabled: false) }

      before do
        create_default_protocol!
        admin = staff_with("admin@curitiba.demo", "municipal_admin")
        Maintainer.create!(email_address: "dev@local", password: "s3nha-forte-1", otp_secret: ROTP::Base32.random, otp_enabled_at: Time.current)
        SignatureCrew.seed_current_city(slug: "curitiba", password: "dev-password")
        ProfessionalCrew.seed_current_city(slug: "curitiba", password: "dev-password", admin: admin)
        ScreeningCrew.seed_platform!
        ScreeningCrew.seed_current_city(slug: "curitiba", ddd: "41")
        ClinicalRecordCrew.seed_platform!
        ClinicalRecordCrew.seed_current_city(slug: "curitiba", ddd: "41")
        CityCatalog.reset_cache!
        Current.city = City.find(city.id)
      end

      after do
        Rails.cache.clear
        Current.reset
      end

      it "liga, adota, semeia as cinco pessoas com o que a prova precisa; a segunda rodada não muda nada" do
        expect(described_class.seed_platform!).to eq(created: 6)
        result = described_class.seed_current_city(slug: "curitiba", ddd: "41")
        expect(CareLines::Gate.usable?(Current.city)).to be(true)
        expect(CareLine.where(source: "template").count).to eq(6)
        expect(CareLine.find_by!(declared_condition: "mental_health").source).to eq("custom")
        expect(CareLineEnrollment.active_enrollments.joins(:care_line).pluck("care_lines.template_key"))
          .to match_array(%w[diabetes_c4 pregnancy_c3 child_c2])
        pregnant = CareLineEnrollment.joins(:care_line).find_by!(care_lines: { template_key: "pregnancy_c3" })
        expect(pregnant.anchor_on).to eq(Time.zone.today - 70)
        pending = CareLineSuggestion.pending_suggestions.joins(:care_line).pluck(:reason, "care_lines.name")
        expect(pending).to include([ "problem", "Pessoa com hipertensão" ], [ "age_sex", "Pessoa idosa" ],
                                   [ "declared", "Saúde mental — acompanhamento" ])
        diabetic = CareLineEnrollment.joins(:care_line).find_by!(care_lines: { template_key: "diabetes_c4" })
        row = CareLines::Evaluation.enrollments([ diabetic ], today: Time.zone.today).fetch(diabetic.id)
        expect(row.practices.index_by(&:key).transform_values(&:status)).to include("c4_a" => "ok", "c4_e" => "overdue", "c4_f" => "overdue")
        expect(result[:people].size).to eq(5)
        expect { described_class.seed_current_city(slug: "curitiba", ddd: "41") }
          .not_to(change { [ CareLineEnrollment.count, CareLineSuggestion.count, Consultation.count, CitizenDeclaredCondition.count ] })
      end

      it "fora de Curitiba não liga nada" do
        described_class.seed_platform!
        expect(described_class.seed_current_city(slug: "maringa", ddd: "44")).to include(switch: "desligado (só Curitiba liga)")
        expect(CareLine.count).to eq(0)
      end
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar.** `test spec/lib/care_lines_crew_spec.rb` → FAIL (`cannot load such file -- lib/care_lines_crew`).

- [ ] **Step 3: Implemente** `lib/care_lines_crew.rb`:

  ```ruby
  require_relative "screening_crew"
  require_relative "clinical_record_crew"

  # Semente de dev do módulo 23 (ADR 0040). Dev é fictício mas imita o real:
  # em Curitiba, care_lines LIGADO pelo mantenedor de dev; as seis linhas do
  # catálogo-base adotadas pelo admin e a linha própria "Saúde mental —
  # acompanhamento"; na UBS Jardim das Flores, cinco pessoas (tabela da Task
  # 16 do plano). Toda inscrição passa pela confirmação (AcceptSuggestion) —
  # nunca nasce direto. Idempotente. Roda depois do ClinicalRecordCrew.
  class CareLinesCrew
    UNIT = "UBS Jardim das Flores".freeze
    PHONE_PREFIX = "93333".freeze
    DEV_MAINTAINER = "dev@local".freeze
    MENTAL = {
      "name" => "Saúde mental — acompanhamento", "declared_condition" => "mental_health",
      "eligibility" => { "problems" => { "ciap2" => %w[P74 P76], "cid10" => %w[F32 F33 F41] }, "declared" => %w[mental_health] },
      "practices" => [ { "key" => "sm_a", "label" => "Consulta de acompanhamento a cada 3 meses", "citizen_label" => "Consulta de acompanhamento",
                         "source" => "consultation", "anchor" => "rolling", "window_days" => 92, "min_count" => 1, "schedule" => "end_of_window",
                         "cbo_filter" => %w[2251 2235 251510] } ]
    }.freeze
    # n => [nome, nome da mãe, nascimento, sexo]
    PEOPLE = {
      1 => [ "José Carlos Pereira", "Iracema Pereira", "1968-03-14", "male" ],
      2 => [ "Ana Beatriz Souza", "Lúcia Souza", "1999-05-02", "female" ],
      3 => [ "Miguel Oliveira Santos", "Camila Oliveira Santos", nil, "male" ],
      4 => [ "Maria das Graças Lima", "Benedita Lima", "1953-08-20", "female" ],
      5 => [ "Rafael Costa Almeida", "Sônia Costa", "1991-11-03", "male" ]
    }.freeze
    DRAFT = {
      "subjective" => "Retorno de rotina.", "objective" => "Bom estado geral.", "assessment" => "Acompanhamento.",
      "plan" => "Retorno conforme a linha de cuidado.", "care_type" => 5, "conducts" => [ 1 ], "exam_requests" => []
    }.freeze

    class << self
      def seed_platform!
        guard_environment!
        CareLines::Templates::Load.call
      end

      def seed_current_city(slug:, ddd:)
        guard_environment!
        return { switch: "desligado (só Curitiba liga)", people: [], pending: 0 } unless slug == "curitiba"

        maintainer = Maintainer.find_by(email_address: DEV_MAINTAINER)
        return { switch: "desligado (sem mantenedor #{DEV_MAINTAINER})", people: [], pending: 0 } unless maintainer

        Platform::Features.set!(city: City.find(Current.city.id), key: CareLines::Gate::KEY, enabled: true, maintainer: maintainer)
        admin = User.find_by!(email_address: "admin@#{slug}.demo")
        adopt!(admin)
        unit = HealthUnit.find_by!(name: UNIT)
        doctor = User.find_by!(email_address: "profissional@#{slug}.demo")
        nurse = User.find_by!(email_address: "enfermeira@#{slug}.demo")
        team = team!(unit, doctor, nurse)
        neighborhood = neighborhood!(unit)
        people = PEOPLE.keys.to_h { |n| [ n, citizen!(slug, ddd, n, neighborhood) ] }

        consult!(slug, unit, doctor, people[1], [ %w[ciap2 T90], %w[ciap2 K86] ])
        accept!(people[1], "diabetes_c4", team, nurse)
        if ciap2?("W78")
          consult!(slug, unit, nurse, people[2], [ %w[ciap2 W78] ])
          accept!(people[2], "pregnancy_c3", team, doctor, anchor_date: (Time.zone.today - 70).iso8601)
        else
          warn "[seeds] linhas de cuidado: CIAP-2 W78 ausente na release ativa — gestante sem inscrição"
        end
        accept!(people[3], "child_c2", team, nurse)
        CareLines::Suggest.for_citizen(people[4].id)
        CareLines::DeclareCondition.call(citizen: people[5], condition: "mental_health")
        CareLines::Suggest.for_citizen(people[5].id)

        { switch: "ligado (care_lines)", people: people.values.map(&:display_name),
          pending: CareLineSuggestion.pending_suggestions.where(team_id: team.id).count }
      end

      private

      def guard_environment!
        raise "semente das linhas de cuidado: ambiente implantado; a semente é só de dev" if Rota.deployed?
      end

      def adopt!(admin)
        CareLineTemplate.latest.each do |template|
          next if CareLine.exists?(template_key: template.key)

          result = CareLines::SaveLine.call(params: { "template_key" => template.key, "template_version" => template.version }, by: admin)
          raise "semente das linhas: adoção #{template.key} recusada (#{result.reason})" if result.failure?
        end
        return if CareLine.exists?(declared_condition: "mental_health")

        result = CareLines::SaveLine.call(params: MENTAL, by: admin)
        raise "semente das linhas: linha própria recusada (#{result.reason} #{result.details})" if result.failure?
      end

      def team!(unit, *users)
        team = HealthTeam.where(health_unit: unit, active: true).order(:created_at).first ||
               HealthTeam.create!(health_unit: unit, kind: "70", name: "ESF JARDIM DAS FLORES",
                                  ine: Zlib.crc32("ine:curitiba:ESF JARDIM DAS FLORES").to_s.rjust(10, "0"))
        users.each do |user|
          next if HealthTeamMember.active.exists?(professional: user.professional, health_team: team)

          link = user.professional.links.active.find_by!(health_unit: unit)
          HealthTeamMember.create!(professional: user.professional, health_team: team, cbo_code: link.cbo_code, started_on: Time.zone.today - 365)
        end
        team
      end

      # Um bairro coberto SÓ pela UBS (a equipe da pessoa sai do bairro).
      def neighborhood!(unit)
        covered = NeighborhoodCoverage.where(health_unit_id: unit.id).pluck(:neighborhood_id)
        only = NeighborhoodCoverage.where(neighborhood_id: covered).group(:neighborhood_id).having("count(*) = 1").pluck(:neighborhood_id)
        Neighborhood.where(id: only, active: true).order(:name).first ||
          Neighborhood.find_or_create_by!(name: "Jardim das Flores (linhas de cuidado)") { |n| n.source = "manual" }.tap do |n|
            NeighborhoodCoverage.find_or_create_by!(neighborhood: n, health_unit: unit)
          end
      end

      def citizen!(slug, ddd, n, neighborhood)
        name, mother, birth, sex = PEOPLE.fetch(n)
        birth ||= (Time.zone.today - 150).iso8601
        registered = Citizens::RegisterPerson.call(phone: format("+55%s#{PHONE_PREFIX}%04d", ddd, n),
                                                   cpf: ScreeningCrew.send(:cpf_for, "#{slug}:care_lines:#{n}"),
                                                   profile: { birth_date: birth, sex: sex })
        raise "semente das linhas: pessoa #{n} recusada (#{registered.reason})" if registered.failure?

        citizen = registered.payload[:citizen]
        unless citizen.verification_level_verified?
          reception = User.find_by!(email_address: "recepcao@#{slug}.demo")
          code = Citizens::IssueVerificationCode.call(citizen: citizen).payload.fetch(:code)
          verified = Citizens::Verify.call(cpf: citizen.cpf, code: code, document_checked: true, by: reception, birth_date: birth, sex: sex,
                                           full_name: name, mother_name: mother)
          raise "semente das linhas: validação #{n} recusada (#{verified.reason})" if verified.failure?
        end
        citizen.reload.tap { |c| c.update!(neighborhood: neighborhood) if c.neighborhood_id.nil? }
      end

      # Consulta de hoje com os problemas (caminho real do 19a), só se a pessoa
      # ainda não tem consulta finalizada.
      def consult!(slug, unit, by, citizen, problems)
        return if citizen.patient && citizen.patient.consultations.finalized_consultations.exists?

        attendance = ClinicalRecordCrew.send(:checked_in!, slug, unit, citizen, "linhas-#{citizen.id[0, 8]}")
        called = Attendances::Call.call(attendance: attendance, health_unit_id: unit.id, by: by)
        raise "semente das linhas: chamada recusada (#{called.reason})" if called.failure?

        consultation = Consultations::Start.call(attendance: attendance.reload, by: by).payload.fetch(:consultation)
        draft = DRAFT.merge("vitals" => { "systolic" => 128, "diastolic" => 82, "weight_kg" => "79.0", "height_cm" => 172 },
                            "evaluated_problems" => problems.map { |t, code| { "terminology" => t, "code" => code, "action" => "add" } })
        saved = Consultations::SaveDraft.call(consultation: consultation, params: draft, by: by)
        raise "semente das linhas: rascunho recusado (#{saved.reason} #{saved.details})" if saved.failure?

        finalized = Consultations::Finalize.call(consultation: consultation.reload, outcome_params: { "outcome" => "return" }, by: by)
        raise "semente das linhas: finalização recusada (#{finalized.reason} #{finalized.details})" if finalized.failure?
      end

      # Sugestão do sistema → confirmação do profissional (nunca inscrição direta).
      def accept!(citizen, template_key, team, by, anchor_date: nil)
        line = CareLine.find_by!(template_key: template_key)
        citizen.reload
        return if citizen.patient_id && CareLineEnrollment.active_enrollments.exists?(patient_id: citizen.patient_id, care_line: line)

        citizen.patient_id ? CareLines::Suggest.for_patient(citizen.patient_id) : CareLines::Suggest.for_citizen(citizen.id)
        suggestion = CareLineSuggestion.pending_suggestions.where(care_line: line)
                                       .where(citizen_id: citizen.id).or(CareLineSuggestion.pending_suggestions.where(care_line: line, patient_id: citizen.patient_id))
                                       .first
        raise "semente das linhas: sem sugestão #{template_key} para a pessoa" unless suggestion

        result = CareLines::AcceptSuggestion.call(suggestion: suggestion, team: team, by: by, anchor_date: anchor_date)
        raise "semente das linhas: confirmação #{template_key} recusada (#{result.reason})" if result.failure?
      end

      def ciap2?(code)
        release = TerminologyRelease.active.find_by(kind: "ciap2")
        release.present? && Ciap2Code.exists?(release_id: release.id, code: code)
      end
    end
  end
  ```

  Em `db/seeds.rb`: `require Rails.root.join("lib/care_lines_crew").to_s` junto dos outros; na plataforma, depois da linha do `ClinicalRecordCrew.seed_platform!`:

  ```ruby
    # Linhas de cuidado (módulo 23, ADR 0040): catálogo-base C2–C7 (carga idempotente).
    puts "[seeds] linhas de cuidado (catálogo-base) . #{CareLinesCrew.seed_platform![:created]} versões novas"
  ```
  e, na cidade, depois do bloco do `BillingCrew`:

  ```ruby
          # ── Linhas de cuidado (módulo 23, ADR 0040) ───────────────────────────
          # Depois do ClinicalRecordCrew (UBS, médica, enfermeira, recepção, CIAP/CID).
          care = CareLinesCrew.seed_current_city(slug: slug, ddd: ddd)
          puts "[seeds] linhas de cuidado . #{care[:switch]}#{"; #{care[:people].size} pessoas, #{care[:pending]} sugestões pendentes" if care[:people].any?}"
  ```
  Comentário de cabeçalho do `db/seeds.rb`: acrescente `Linhas de cuidado (módulo 23): care_lines ligado em Curitiba, linhas adotadas e a própria de saúde mental, cinco pessoas na UBS Jardim das Flores — ver lib/care_lines_crew.rb.`

  (`ClinicalRecordCrew.checked_in!(slug, unit, citizen, step_key)` é privado e conferido em `c5b6a9d`; por isso o `send`. Se a UBS tiver mais de uma equipe ativa, os pares sem paciente ficam sem equipe e a semente para com "sem sugestão" — mantenha uma equipe na UBS de dev.)

- [ ] **Step 4: Rode e veja passar.** `test spec/lib/care_lines_crew_spec.rb spec/invariants/care_lines_invariants_spec.rb` → PASS (a invariante confirma que a semente não cria inscrição direto).

- [ ] **Step 5: Commit.**

  ```bash
  git add lib/care_lines_crew.rb db/seeds.rb spec/lib/care_lines_crew_spec.rb
  git commit -m "chore: seed realistic Curitiba care lines with five people for the browser proof

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 17: Revisão final, suíte, api na porta 3045 e runbook `docs/operacao/rollout-linhas-de-cuidado.md`

**Files:**
- Create: `docs/operacao/rollout-linhas-de-cuidado.md` (repo `docs`)

- [ ] **Step 1: Varreduras** e lint:

  ```bash
  docker compose exec -T -w /rails/.claude/mod23 api bin/rubocop
  git grep -n "Current.city =" -- app lib
  git grep -n -A3 "DomainEvents.publish(" -- app/commands/care_lines app/services/care_lines app/jobs/care_lines app/controllers/care_lines app/controllers/citizen_api/declared_conditions_controller.rb
  git grep -n "CareLineEnrollment.create" -- app lib
  ```
  Expected: rubocop limpo; `Current.city =` não aparece em `app/` nem `lib/`; cada `publish(` lido inteiro (multilinha) só com ids; `CareLineEnrollment.create` só em `app/commands/care_lines/enroll.rb`.

- [ ] **Step 2: Suíte completa** pela sessão "Testes" (SendMessage: "rodar `bin/test api -w mod23 --full` na branch `feat/mod-23-care-lines`, worker parado"). Expected: verde; ~0,10 s por exemplo com o host calmo (memória "BCrypt e custo em teste": acima de 0,15 s investigue).

- [ ] **Step 3: api do worktree na porta 3045 e semente** — **só com autorização do usuário** para migrar/semear o banco de dev (memória "Bancos de teste entre branches" e brief: o dev só é migrado na prova):

  ```bash
  docker compose exec -d -w /rails/.claude/mod23 api bin/rails server -b 0.0.0.0 -p 3045
  docker compose exec -T -w /rails/.claude/mod23 api bin/migrate
  docker compose exec -T -w /rails/.claude/mod23 api bin/rails care_lines:templates:load
  docker compose exec -T -w /rails/.claude/mod23 api bin/rails db:seed
  docker compose exec -T api curl -s -o /dev/null -w '%{http_code}\n' -H "Host: curitiba.localhost" http://localhost:3045/up
  ```
  Expected: `200`; o `db:seed` imprime `[seeds] linhas de cuidado (catálogo-base) . 0 versões novas` (já carregado) e `[seeds] linhas de cuidado . ligado (care_lines); 5 pessoas, 4 sugestões pendentes` (hipertensão de José Carlos, mulher e prevenção do câncer de Ana Beatriz, pessoa idosa de Maria das Graças, saúde mental de Rafael). Os Vite dos worktrees (`.worktrees/dashboard-mod23` na 5198, `.worktrees/wpda-mod23` na 5199) apontam o proxy para `http://api:3045`.

- [ ] **Step 4: Conferência pelo api** (a prova no navegador é dos planos do dashboard e do wpda, com estas contas): com `enfermeira@curitiba.demo` (senha de dev + TOTP), `GET /care_lines/teams` traz a ESF da UBS Jardim das Flores; `GET /care_lines/enrollments?team_id=<id>&status=overdue` traz José Carlos (HbA1c e pés vencidos) e Miguel (1ª consulta vencida); `GET /care_lines/suggestions?team_id=<id>` traz hipertensão (José Carlos), mulher (Ana Beatriz), pessoa idosa (Maria das Graças) e saúde mental (Rafael); com `admin@curitiba.demo`, `GET /care_lines/preview?team_id=<id>&indicator=C4` responde com `estimate: true` e `excluded: ["acs_visit"]`. Falha de api → corrija com teste antes.

- [ ] **Step 5: Runbook** `docs/operacao/rollout-linhas-de-cuidado.md` (repo docs, branch `docs/mod-23-care-lines`):

  ```markdown
  # Rollout — linhas de cuidado (módulo 23, ADR 0040)

  ## Pré-requisitos
  - Cidade em modo registro (`record_mode = record`) com `clinical_record` ligado e utilizável — `care_lines` depende dele
    (sem ele, o interruptor fica "ligado, falta pré-requisito" e as rotas respondem 403 `feature_disabled`).
  - Equipes (INE, tipo 70/76) com os profissionais (CBO do CNES) nas unidades — módulo 16 (retrato do CNES). Sem equipe,
    ninguém recebe sugestão (a pessoa só entra por inscrição manual).
  - Bairros e cobertura das unidades (módulo 11): o par validado que nunca consultou só recebe sugestão se o bairro dele
    for coberto por unidade com UMA equipe ativa.
  - Notas metodológicas conferidas (prova técnica: `docs/pesquisa/2026-10-22-linhas-de-cuidado-prova-tecnica.md`).
  - Nenhuma gem nova.

  ## Ordem de deploy
  1. api: publicar a imagem e rodar `bin/migrate` dela ANTES de cortar tráfego — plataforma `20261022900001`
     (`care_line_templates`) e cidade `20261022900002` (`care_lines`, `care_line_enrollments`, `care_line_suggestions`,
     `care_outreach_contacts`, `citizen_declared_conditions`; triggers em `db/city_triggers.sql`); as duas irreversíveis.
  2. api: `bin/rails care_lines:templates:load` UMA vez na plataforma (idempotente; cria a versão 1 das seis linhas).
  3. dashboard e wpda → 4. maintenance → 5. catálogo público. Nunca migrar cidade fora do rake.

  ## Ligar numa cidade
  1. maintenance: `care_lines` ("Linhas de cuidado ...").
  2. Admin municipal (dashboard, com step-up): adotar as linhas do catálogo-base que a cidade vai acompanhar; criar as
     próprias (ex.: saúde mental, com a condição declarada `mental_health`).
  3. O job diário (`CareLines::SuggestJob`, 6h45) gera as sugestões; o problema registrado na consulta e a condição
     declarada no wpda geram na hora (consumidores). **Primeira rodada:** numa cidade grande, as linhas por idade e sexo
     (criança, pessoa idosa, mulher) sugerem muita gente de uma vez — combine com as equipes adotar essas linhas aos
     poucos (uma por semana) ou só depois de as equipes confirmarem as por problema.
  4. Profissionais com vínculo na unidade da equipe: painel e contato; médico ou enfermeiro: confirmar e inscrever.
  5. Gestão (admin, analista): prévia C2–C7 por equipe — sempre rotulada como estimativa.

  ## O que não entra (comunicar às equipes)
  - Visita do ACS (módulo 24), vacinas (módulo 29) e saúde bucal (módulo 28): "não acompanhado pelo sistema"; não contam
    como atraso; os pesos saem da prévia e os restantes são redistribuídos.
  - Registros feitos fora do Rota Saúde (PEC de outra unidade, outro município): o SIAPS conta, a prévia não.
  - Vínculo pela NT 30/2025: a prévia atribui a pessoa pela última consulta (24 meses) ou pelo bairro.

  ## Revisão anual das notas (risco do spec §11)
  - Quando o MS publicar nova nota metodológica C2–C7 (ou NT de pesos), refaça a prova técnica (download, SHA-256,
    comparação prática a prática) e grave a versão nova pelo maintenance (`upsertCareLineTemplate`). As cidades
    continuam na versão adotada até readotar (`PATCH /care_lines/admin/lines/:id { template_version }`).

  ## Verificação depois de ligar
  - Uma sugestão por problema ativo e a confirmação (paciente de teste); o painel da equipe com a busca ativa.
  - Nenhuma nota de contato, condição declarada ou DUM no log do dia (`grep` dos textos de teste).

  ## LGPD
  - Inscrição e contatos retêm o pedido de exclusão (documento clínico); a condição declarada e as sugestões do par são
    apagadas na confirmação da exclusão.

  ## Desligar
  - `care_lines` desligado: rotas `/care_lines` e `/citizen/{declared_conditions,care}` respondem 403; o job diário e os
    consumidores param; o catálogo do cidadão continua com o aviso de triagem vencida (é do módulo 15, sem interruptor).
    Os registros ficam (inscrições, contatos, declarações).
  - Migrações irreversíveis: não há rollback de schema; volte só o código (expand/contract).
  ```
  Commit no repo docs (sem push):

  ```bash
  /opt/homebrew/bin/git -C docs add operacao/rollout-linhas-de-cuidado.md
  /opt/homebrew/bin/git -C docs commit -m "docs: add care lines rollout runbook (module 23)

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

- [ ] **Step 6: Fechamento.** Pare o servidor da 3045; confira os cards: F-23.4 em **Done**; F-23.1, F-23.2, F-23.3, F-23.5, F-23.6, F-23.7, F-23.8 continuam **In Progress** até o dashboard/wpda/maintenance deles. Merge e push **só com autorização do usuário** (memória "Sessões paralelas no push": `origin/main..main` antes de push).

---

## Self-review (feito ao escrever o plano)

- **Cobertura da spec:** §1 intenção e antiobjetivos → Tasks 11 (confirmação sempre), 7 (`untracked` nunca atraso), 13 (`estimate: true`), 14 (cidadão sem diagnóstico/código); §2 decisão 1 (seis linhas + próprias) → Tasks 3, 6; 2 (sugestão + confirmação + manual + saída) → Tasks 10, 11, 12; 3 (leitura sem cópia; ACS/vacina fora) → Tasks 7, 8; 4 (painel, busca ativa, contato, agendar) → Task 12 (agendar é rota existente do 17, aberta pelo dashboard com o `patient_id`/`citizen_id` do `<enrollment>`); 5 (Meu acompanhamento + triagem vencida) → Task 14; 6 (prévia) → Task 13; 7 (só modo registro) → Task 1; 8 (pares validados que nunca consultaram; paciente por `Patients::Resolve`) → Tasks 9, 10, 11, 12; 9 (denominador elegível, pesos redistribuídos, resolvido tira e sugere encerramento) → Tasks 9, 10, 13; decisões do coordenador (metas proporcionais; CBO da consulta; acesso; condição declarada não retém; linha própria com `declared_condition`; triagem vencida do módulo 15; DUM na confirmação) → Tasks 7, 3, 6/11/12/13, 15, 6/10, 14, 11; §3 interruptor → Task 1; §4 dados → Tasks 3, 5; §5 → Tasks 7, 8; §6 telas (api) → Tasks 6, 12, 13, 14; Maintenance → Task 4; §7 segurança e LGPD → Tasks 5 (cifra, filtros), 12 (trilha), 15; §8 testes → cada task (seis linhas com séries conhecidas: C2, C3, C4, C7 na Task 7, C5/C6 pelas specs de painel/prévia/cidadão; guarda R18 na Task 3/4); §9 entrega → Task 0 (prova), 17 (runbook); semente → Task 16; F-23.1..F-23.8 → Tasks 1–4; 6; 10–12; 7–8; 12; 13; 14; 14.
- **Placeholders:** nenhum. A Task 0 tem regra de parada explícita (SHA-256 das notas).
- **Tipos e nomes:** `CareLines::{Gate,Catalog,Rules,Templates::Save,Templates::Load,SaveLine,Access,Json,Event,Person,PracticeStatus,Evaluate,Sources,Evaluation,Eligibility,Facts,TeamPopulation,Suggest,Enroll,AcceptSuggestion,DismissSuggestion,EndEnrollment,RecordContact,Cursor,Trail,Board,Preview,DeclareCondition,WithdrawCondition,CitizenView}` e os jobs `CareLines::{SuggestJob,ProblemChangedJob,ConditionDeclaredJob}` — iguais em todas as tasks; helpers `care_city!`, `care_lines_loaded!`, `care_admin!`, `adopted_line!`, `care_unit!`, `care_doctor!`, `care_nurse!`, `care_team!`, `care_patient!`, `care_enrollment!`, `care_screening!`, `care_neighborhood!`, `care_citizen!`, `resolve_problem!`, `declare!`.
- **Séries conhecidas:** as datas e contas das Tasks 7 e 13 foram conferidas ao escrever o plano (183/365 dias, marcos proporcionais 163/82/43, janelas de C3 97/196/294/336, HPV 1826, pesos redistribuídos 25,0/18,8 e 28,6/42,9).
- **Review Focus:** 1 → Task 10 ("encerramento só com todos os problemas elegíveis resolvidos") e Task 9 ("denominador ... encerramento"); 2 → Task 10 ("recusada não volta por 365 dias"); 3 → Task 12 ("busca ativa varre em blocos") e Task 8 ("uma consulta por fonte"); 4 → Task 11 ("gestante: DUM obrigatória e entre hoje − 336 e hoje"); 5 → Task 15 ("condição declarada não retém").

## Divergências propostas ao contrato

Numeradas como **L** (do plano do api).

1. **L1 — Formato da boa prática (§2 `<care_line>.practices`).** Somam-se, opcionais: `window_start_days` (início da janela a partir da âncora: trimestres, puerpério, dTpa), `code_filter` `{ ciap2, cid10 }` (só `consultation`: puerpério, saúde sexual e reprodutiva), `applies_to` `{ age_min_years, age_max_years, sex }` (faixa por prática — C7), `all_of` (lista de grupos, cada grupo uma lista de `{ source, window_days? }`; cumprida quando todo grupo tem registro — testes de sífilis/HIV/hepatites; HPV com 60 meses), `points` (pontos do Quadro 01; obrigatórios e somando 100 nos templates) e `citizen_label` (rótulo em linguagem simples para o wpda). Prática com `all_of` repete no `source` a primeira alternativa.
2. **L2 — Forma da elegibilidade (o contrato não fixa):** `{ problems: { ciap2: [], cid10: [] }, exclude_problems, age_min_days, age_max_days, age_min_years, age_max_years, sex, gender_identities_included, gender_identities_excluded, declared: [], requires_anchor: "lmp"|null }`; códigos casados por prefixo, CID sem ponto.
3. **L3 — CBO de quem confirma:** o contrato §4 fala em 225*/2235*; as notas contam consulta de médico pelos grupos **2231**, 2251, 2252, 2253 e de enfermeiro 2235. O plano implementa o contrato (225*/2235*) para confirmar e as notas para a consulta que conta; propõe somar `2231` à confirmação.
4. **L4 — Códigos de erro novos:** `invalid_anchor` (422; DUM fora de [hoje − 336, hoje] ou âncora em linha que não pede), `line_inactive` (409), `invalid_transition` (409), `reason_required`/`too_long` (com `field`), `invalid_reason`, `invalid_channel`, `invalid_outcome`, `invalid_person`, `invalid_care_line`, `invalid_filter`, `invalid_cursor`, `invalid_indicator`, `invalid_condition`, `invalid_name`, `invalid_declared_condition`, `template_not_found`, `duplicate_declared_condition`, `immutable_rules` (PATCH de regra em linha adotada), `out_of_scope` (403 de lista/criação com equipe alheia), `not_verified` (403, já no contrato §7).
5. **L5 — `care_line_suggestions.team_id` (obrigatório) e `enrollment_id`:** a sugestão nasce atribuída a uma equipe (Desvio 5) — é assim que a lista `?team_id=` e o escopo por unidade funcionam; pessoa sem equipe atribuível não recebe sugestão. `enrollment_id` liga a sugestão `resolved_problem` à inscrição; `<suggestion>` ganha `enrollment_id`. Aceitar `resolved_problem` encerra com `discharge` e dispensa `team_id`.
6. **L6 — Página filtrada pode vir curta:** com `status=overdue|due_soon`, até 5 blocos de `limit` são avaliados; `next_before` não nulo com lista curta (ou vazia) significa "continue".
7. **L7 — Trilha uma por pessoa mostrada** (não uma por requisição): o `clinical_record.viewed` tem um `patient_id`; leva `access: care_line` e `care_line_enrollment_id` ou `care_line_suggestion_id` (em sugestão de par sem paciente, `patient_id` nulo e `person_ref` `citizen:<id>`).
8. **L8 — Prévia (§6):** a mais `template_version`, `concept` (`optimal|good|sufficient|regular`, parâmetros das notas: > 75, > 50, > 25), e por prática `label`, `points` (originais) e `denominator` (C7 tem denominador por faixa); `score` e `weight` são strings com uma casa e ponto; `score`/`concept` nulos com denominador 0; `excluded` traz as famílias não rastreadas da linha (`acs_visit`, `vaccine`, `oral_health`).
9. **L9 — Triagem vencida sem interruptor do módulo 15:** o módulo 15 não tem chave no catálogo; `overdue`/`last_done_at` saem no `GET /citizen/people/:id/catalog` em toda oferta com `retake_after_days` (também nas recentes, com `overdue: false`). "Meu acompanhamento" ganha `overdue_triages: [ { protocol_name, title, last_done_at } ]` (§7 não tinha a forma).
10. **L10 — Listas de apoio do dashboard:** `GET /care_lines/lines` → `{ lines: [ { id, name, indicator, requires_anchor } ] }` (ativas) e `GET /care_lines/teams` → `{ teams: [ { id, ine, name, unit: { id, name } } ] }` (profissional: equipes das unidades dele; admin/analista: todas as ativas).
11. **L11 — Formas do cidadão (§7):** `GET/POST/DELETE /citizen/declared_conditions` respondem `{ conditions: [ { condition, declared_at } ] }` (só as vigentes); `none` retira as outras e uma condição retira o `none`; condição repetida não grava de novo. Declaração só de par validado (403 `not_verified`).
12. **L12 — `<enrollment>`** ganha `ended_reason` e `ended_at` (encerrada, no retorno do `end`) e, no `GET /:id`, `contacts: [ { id, channel, outcome, note, recorded_by: { id, name }, recorded_at } ]`; `last_contact` nunca traz a nota. O `POST /contacts` responde `{ contact }`.
13. **L13 — Eventos com `user_id`:** `care_line_enrollment.created/.ended`, `care_line_suggestion.accepted/.dismissed`, `care_outreach.recorded` (+ `care_line_enrollment_id`) e `care_line.adopted` levam `user_id`; `citizen_condition.*` levam `citizen_id`. Todos só ids. Plataforma: `care_line.template_changed` (`care_line_template_id`, `template_key`, `version`) e a auditoria `maintenance.care_line_template.changed`.
14. **L14 — Referência do contrato a "§11":** o brief cita "contrato §1 e §11" para interruptores e nomes; o contrato vigente vai até o §10 (o interruptor está no §8 e os nomes no §1). Nada muda no código.
