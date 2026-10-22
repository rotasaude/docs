# Módulo 23 — prova técnica das notas metodológicas C2–C7 e da NT 8/2026 (2026-10-22)

## Fontes

Índice: `https://www.gov.br/saude/pt-br/composicao/saps/publicacoes/fichas-tecnicas/equipe-de-atencao-primaria-e-saude-da-familia`. Baixadas de novo em 2026-10-22 na execução do plano do api (Task 0); o índice lista as seis notas C2–C7 e a NT 8/2026; os sete SHA-256, os tamanhos, os números SEI e as datas de assinatura conferem com a tabela abaixo, que veio da escrita do plano.

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

## Regras comuns (notas C2–C7, item 24)

- Pessoa vinculada a eSF (tipo 70) ou eAP (tipo 76) pela NT 30/2025; desempate pela Portaria SAPS 161/2024. Interrupção: saída do cadastro por mudança de território, mudança de equipe, óbito no CadSUS; em C4/C5, **todas** as condições elegíveis marcadas como "resolvidas"; em C3, aborto (CID/CIAP).
- **Consulta que conta:** atendimento individual (presencial ou remoto) de médico (CBO 2231, 2251, 2252, 2253) ou enfermeiro (2235) — os quatro primeiros dígitos do CBO.
- Pressão, peso e altura: qualquer CBO da lista longa (inclui técnico de enfermagem 3222); peso e altura **simultâneos** (mesmo dia). Procedimentos SIGTAP 03.01.10.003-9 (aferição de PA), 01.01.04.002-4 (avaliação antropométrica) também valem.
- Exames: "solicitado ou avaliado" (pedido ou resultado).
- Parâmetro de cada indicador (C2–C7): Ótimo > 75 e ≤ 100; Bom > 50 e ≤ 75; Suficiente > 25 e ≤ 50; Regular ≤ 25. Pontuação por pessoa até 100.

## NT 8/2026 — pesos e nota

Pesos do componente III (eSF/eAP): **C1 1, C2 2, C3 2, C4 1, C5 1, C6 1, C7 2** (total 10). Conceito por indicador vale Regular 0,25 · Suficiente 0,50 · Bom 0,75 · Ótimo 1,00; nota final = Σ conceito × peso; classificação da nota final: Ótimo > 7,5; Bom ≥ 5 e ≤ 7,5; Suficiente > 2,5 e < 5; Regular ≤ 2,5. Resultado do quadrimestre = média dos meses; em C2 e C3 só contam os meses com criança completando 2 anos / gestação chegando ao 42º dia de puerpério.

## Catálogo-base fixado (fonte do `config/care_lines/catalog-base.yml` da Task 2)

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

## O que o Rota Saúde implementa (plano 2026-10-22-module-23-care-lines-api, Desvios 1–13)
- Visita do ACS, vacinas e saúde bucal: `untracked` ("não acompanhado pelo sistema"), nunca atraso; os pesos delas saem
  da conta da prévia e os restantes são redistribuídos (aviso na tela).
- Consulta que conta: finalizada (19a), CBO 2231/2251/2252/2253/2235 (C7c: lista do item 24d).
- Pressão e peso+altura: sinais da consulta finalizada, da escuta concluída (18) ou procedimento SIGTAP (22).
- Exame: pedido (21a, não cancelado nem negado) ou resultado (21a/21b) pelo SIGTAP.
- A prévia é estimativa: o oficial é do SIAPS (registros de qualquer unidade do país, visitas, vacinas, vínculo NT 30).
- Revisão anual: novas notas → nova versão do template pelo maintenance (`upsertCareLineTemplate`).

## Resultado
CONFERE (SHA-256 iguais). Conferência no texto: C4 com 6 ocorrências dos códigos de elegibilidade e SIGTAP (esperado ≥ 4); cabeçalho do Quadro 01 de C3 ("Boas práticas de cuidado na gestação e puerpério"); NT 8/2026 com os conceitos Regular 0,25 e Ótimo 1,00.
