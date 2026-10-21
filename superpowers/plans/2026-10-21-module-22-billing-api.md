# Módulo 22 — Procedimentos e faturamento BPA (api e contracts) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **PRÉ-REQUISITO: módulo 21c em `origin/main` do api** (conferido ao escrever este plano em `4908364`: `Platform::Features::CATALOG` terminando em `lab_quality_control`, `lab_inventory`; última migração de cidade `20261018900001_add_lab_quality_supplies.rb` e de plataforma `20261019100002_add_health_department_name_to_cities.rb`; `db/city_schema.rb` em `define(version: 2026_10_18_900001)`; `VALID_RANGE = (1..38)`; `Terminology::Sigtap.release_for(competence)`; `SigtapProcedure` com `amount_sa_cents`; `SigtapProcedureCbo`, `SigtapProcedureCid`, `SigtapProcedureInstrument` com `procedure_code`; `Lab::FinalizeReport` publicando `lab_report.released` com `lab_report_id`; `Citizens::RequestErasure.examined_patients?`; `Timeline::Query::SOURCES` com 9 fornecedores; `lib/sigtap_sample.rb` com 23 procedimentos; `lib/lab_crew.rb` e `lib/record_mode_crew.rb`). **A Task 0 confere a base e a prova técnica do layout: se um nome mudou, ou se o PDF vigente não é mais o de 09/07/2026, PARE e reporte.**

**Goal:** O lado api do módulo 22 (F-22.1 a F-22.7, ADR 0039): interruptor `procedure_billing` e papel `billing_clerk`; procedimento realizado com crítica SIGTAP que marca e não bloqueia (no atendimento, na consulta, avulso e lançamento manual do faturista); produção do laboratório municipal vinda dos laudos liberados do 21b; competência por estabelecimento faturante com pendências, fechamento, reabertura com motivo e versões; o arquivo de interface do BPA Magnético (BPA-C e BPA-I, layout do DATASUS de 09/07/2026) cifrado e baixado com trilha; procedimentos na linha do tempo; o perfil do arquivo e os exemplos golden no repo `contracts`; semente de dev em Curitiba; runbook.

**Architecture:** Tudo na cidade, numa migração (`20261021900001`), lendo a SIGTAP da plataforma (já importada pelo módulo 16; sem migração de plataforma). Um serviço puro de crítica (`Billing::Critic`) é usado na gravação e de novo no fechamento, com o cadastro do momento. Todo procedimento de estabelecimento faturante (unidade não-APS ou laboratório municipal) cai numa competência (`billing_periods`, uma por estabelecimento × AAAAMM) travada FOR SHARE na gravação e FOR UPDATE no fechamento; competência fechada empurra o registro para a próxima aberta (`late`). A UBS nunca ganha competência. O arquivo é montado por um gerador puro (`Billing::BpaFile`), provado byte a byte contra exemplos golden gerados por um oráculo independente em Python que mora no repo `contracts`. O laudo liberado do 21b chega por um consumidor idempotente do evento `lab_report.released`.

**Tech Stack:** Rails 8.1 (API), PostgreSQL 16 (banco por cidade), RSpec, `ActiveSupport::Inflector.transliterate` (nomes em ASCII), Python 3 (stdlib, só o oráculo do `contracts`). Nenhuma gem nova.

**Spec:** `docs/superpowers/specs/2026-10-21-module-22-procedures-billing-design.md` e `docs/adr/0039.md` (leia os dois antes). Contrato entre apps — **fonte única de formatos**: `docs/superpowers/plans/2026-10-21-module-22-billing-contracts.md` (convenções dos contratos do 21a §15, 21b §14 e 21c §11); toda diferença está em "Divergências propostas ao contrato", no fim (prefixo **F**). Pesquisa: `docs/pesquisa/2026-10-05-integracoes/frente-4-faturamento-regulacao.md` (§2.1 SIGTAP, §2.2 SIA/BPA com o layout de 2020, §3 camada 1, §4 riscos). Lições que valem aqui: contadores sem trilha; ordem de travas estável; recusar em vez de cortar (na entrada); decimais como string com ponto.

## Layout vigente (prova técnica feita ao escrever este plano)

Baixados em 2026-10-21 da página `https://sia.datasus.gov.br/versao/listar_ftp_bpa.php` (links FTP `ftp://arpoador.datasus.gov.br/siasus/BPA/`; o FTP passivo completou a transferência daqui):

| Arquivo | Data na página | Tamanho | SHA-256 |
|---|---|---|---|
| `Layout_Exportacao_BPA.pdf` (6 páginas, "Layout da interface texto do BPA e do SIA — Layout INTERNO") | 09-Jul-2026 | 313.580 bytes | `f318bc0359b378e8455280f87dec0ad8c016bcf853d640f2f83353e5648f9f1f` |
| `BPA_LEIAME.txt` (Windows-1252, CR+LF) | 09-Jul-2026 | 68.223 bytes | `5d94ae75585a8f9dca2f8a253a8fd46a28fbbe1c267e499021ffa990e87933b9` |
| `BPAMAG0500.exe` (não baixado: executável Windows) | 09-Jul-2026 | 7,4 MB | — |

Versão vigente do BPA Magnético: **05.00 de 07/07/2026, obrigatória**. Texto ASCII de largura fixa; todo registro termina em CR (13) + LF (10). `NUM` = só dígitos, zeros à esquerda; `ALFA` = espaços à direita. "Obr" = coluna obrigatória do PDF.

### Cabeçalho `01` (130 posições + CR+LF = 132)

| Seq | Nome | Obr | Tam | Ini | Fim | Tipo | Preenchimento | O que o Rota Saúde grava |
|---|---|---|---|---|---|---|---|---|
| 1 | cbc-hdr | | 2 | 1 | 2 | NUM | `01` | `01` |
| 2 | cbc-hdr | | 5 | 3 | 7 | ALFA | `#BPA#` | `#BPA#` |
| 3 | cbc-mvm | | 6 | 8 | 13 | NUM | ano e mês de **processamento**, AAAAMM | competência do fechamento (`billing_periods.competence`) |
| 4 | cbc-lin | | 6 | 14 | 19 | NUM | número de linhas do BPA gravadas, zeros à esquerda | quantidade de linhas `02` + `03` (sem o cabeçalho) |
| 5 | cbc-flh | | 6 | 20 | 25 | NUM | quantidade de folhas gravadas | total de folhas (BPA-C e BPA-I numa série só) |
| 6 | cbc-smt-vrf | | 4 | 26 | 29 | NUM | campo de controle, domínio 1111..2221 | ver "Campo de controle" |
| 7 | cbc-rsp | | 30 | 30 | 59 | ALFA | nome do órgão de origem responsável | `billing_settings.origin_name` em maiúsculas ASCII |
| 8 | cbc-sgl | | 6 | 60 | 65 | ALFA | sigla do órgão de origem (padrão brancos) | `billing_settings.origin_acronym` em maiúsculas ASCII; brancos sem ela (B10 do dashboard) |
| 9 | cbc-cgccpf | | 14 | 66 | 79 | NUM | CNPJ/CPF do prestador ou órgão público | `billing_settings.origin_cnpj` (só numérico; Review Focus 5) |
| 10 | cbc-dst | | 40 | 80 | 119 | ALFA | nome do órgão de saúde destino | `billing_settings.destination_body` em maiúsculas ASCII |
| 11 | cbc-dst-in | | 1 | 120 | 120 | ALFA | `M` municipal, `E` estadual | `billing_settings.management` |
| 12 | cbc_versao | | 10 | 121 | 130 | ALFA | versão do sistema, livre | `ROTASAUDE` + 1 branco |
| 13 | cbc-fim | | 2 | 131 | 132 | | CR+LF | CR+LF |

### Linha `02` — BPA-C (48 posições + CR+LF = 50)

| Seq | Nome | Obr | Tam | Ini | Fim | Tipo | Preenchimento | O que o Rota Saúde grava |
|---|---|---|---|---|---|---|---|---|
| 1 | prd-ident | | 2 | 1 | 2 | NUM | `02` | `02` |
| 2 | prd-cnes | X | 7 | 3 | 9 | NUM | CNES (último dígito é DV) | CNES do estabelecimento |
| 3 | prd-cmp | X | 6 | 10 | 15 | NUM | competência de **realização**, AAAAMM | mês de `performed_on` |
| 4 | Prd_cbo | | 6 | 16 | 21 | ALFA | CBO do profissional | `cbo_code` |
| 5 | prd-flh | X | 3 | 22 | 24 | NUM | folha, 001..999 | folha da série |
| 6 | prd-seq | | 2 | 25 | 26 | NUM | linha na folha, **01..20** | 01..20 |
| 7 | prd-pa | | 10 | 27 | 36 | NUM | procedimento (último dígito é DV) | `sigtap_code` |
| 8 | prd-ldade | | 3 | 37 | 39 | NUM | idade, 0 a 130 anos | idade em anos completos na data |
| 9 | prd-qt | | 6 | 40 | 45 | NUM | quantidade | soma das quantidades do grupo |
| 10 | prd-org | | 3 | 46 | 48 | ALFA | origem: `BPA` SIA/SUS, `PNI`, `SIE`, `SIB`, `MIN`, `PAC`, `SCL`, `EXT` outros sistemas | `BPA` (Divergência F6) |
| 11 | prd-fim | | 2 | 49 | 50 | | CR+LF | CR+LF |

### Linha `03` — BPA-I (351 posições + CR+LF = 353)

| Seq | Nome | Obr | Tam | Ini | Fim | Tipo | Preenchimento | O que o Rota Saúde grava |
|---|---|---|---|---|---|---|---|---|
| 1 | prd-ident | | 2 | 1 | 2 | NUM | `03` | `03` |
| 2 | prd-cnes | X | 7 | 3 | 9 | NUM | CNES | CNES do estabelecimento |
| 3 | prd-cmp | X | 6 | 10 | 15 | NUM | competência de realização | mês de `performed_on` |
| 4 | Prd_cnsmed | X | 15 | 16 | 30 | NUM | CNS do profissional com DV válido | `professionals.cns` do momento do fechamento |
| 5 | Prd_cbo | X | 6 | 31 | 36 | ALFA | CBO | `cbo_code` |
| 6 | Prd_dtaten | | 8 | 37 | 44 | NUM | data de atendimento AAAAMMDD | `performed_on` |
| 7 | prd-flh | X | 3 | 45 | 47 | NUM | folha 001..999 | folha da série |
| 8 | prd-seq | | 2 | 48 | 49 | NUM | linha na folha, **01..99** | 01..20 (folha de 20, como o BPA-C) |
| 9 | prd-pa | | 10 | 50 | 59 | NUM | procedimento | `sigtap_code` |
| 10 | Prd-cnspac | | 15 | 60 | 74 | NUM | CNS do paciente (obrigatório quando o procedimento exige) | **brancos** — o paciente vai pelo CPF (nota 2 do PDF: um documento só) |
| 11 | Prd-sexo | | 1 | 75 | 75 | ALFA | `M` ou `F` (`I` aceito desde 01/2026, LEIAME 04.11) | `F`/`M` do paciente |
| 12 | Prd-ibge | | 6 | 76 | 81 | NUM | IBGE de residência (6 dígitos, sem o DV) | 6 primeiros dígitos de `city_profile.ibge_code` |
| 13 | Prd-cid | | 4 | 82 | 85 | ALFA | CID-10 | `cid10` sem ponto ou brancos |
| 14 | prd-ldade | | 3 | 86 | 88 | NUM | idade 0..130 | anos completos na data |
| 15 | prd-qt | | 6 | 89 | 94 | NUM | quantidade | `quantity` |
| 16 | Prd-caten | | 2 | 95 | 96 | NUM | caráter de atendimento | `01` eletivo, `02` urgência |
| 17 | Prd-naut | | 13 | 97 | 109 | NUM | número da autorização | brancos |
| 18 | prd-org | | 3 | 110 | 112 | ALFA | origem | `BPA` |
| 19 | prd-nmpac | | 30 | 113 | 142 | ALFA | nome completo do paciente | `patients.full_name`, maiúsculas ASCII, cortado em 30 (Desvio 6) |
| 20 | prd-dtnasc | | 8 | 143 | 150 | NUM | nascimento AAAAMMDD | `patients.birth_date` |
| 21 | prd-raca | x | 2 | 151 | 152 | NUM | `01`..`05`, `99` sem informação | `patients.race_color` (`01`..`05`; sem ele, pendência `missing_race_color` — D-U1) |
| 22 | prd-etnia | | 4 | 153 | 156 | NUM | só se raça = 05; senão brancos | `patients.ethnicity` quando `05`; senão brancos |
| 23 | prd-nac | | 3 | 157 | 159 | NUM | nacionalidade (quando preenchido) | brancos |
| 24 | prd_srv | | 3 | 160 | 162 | NUM | serviço (quando o procedimento exige) | brancos (spec §2: não criticado nesta entrega) |
| 25 | prd_clf | | 3 | 163 | 165 | NUM | classificação (idem) | brancos |
| 26 | prd_equipe_Seq | | 8 | 166 | 173 | NUM | sequência da equipe | brancos |
| 27 | prd_equipe_Area | | 4 | 174 | 177 | NUM | área da equipe | brancos |
| 28 | prd_cnpj | | 14 | 178 | 191 | NUM | CNPJ da empresa de manutenção/adaptação de OPM | brancos |
| 29 | prd_cep_pcnte | | 8 | 192 | 199 | NUM | CEP | `patients.address_zip` (sem endereço completo, pendência `missing_address` — D-U1) |
| 30 | prd_lograd_pcnte | | 3 | 200 | 202 | NUM | código do logradouro | brancos (o cadastro não tem a tabela DNE; risco no runbook) |
| 31 | prd_end_pcnte | | 30 | 203 | 232 | ALFA | endereço | `patients.address_street` (maiúsculas ASCII, letras e dígitos, até 30) |
| 32 | prd_compl_pcnte | | 10 | 233 | 242 | ALFA | complemento | `patients.address_complement` (até 10) |
| 33 | prd_num_pcnte | | 5 | 243 | 247 | ALFA | número (`SN`/`00` sem número) | `patients.address_number` (até 5; `SN` sem número) |
| 34 | prd_bairro_pcnte | | 30 | 248 | 277 | ALFA | bairro | `patients.address_district` (até 30); sem ele, o bairro do par do wpda |
| 35 | prd_ddtel_pcnte | | 11 | 278 | 288 | NUM | telefone; senão brancos | telefone do par sem o 55 (10–11 dígitos) quando houver |
| 36 | prd_email_pcnte | | 40 | 289 | 328 | ALFA | e-mail | brancos |
| 37 | prd_ine | X | 10 | 329 | 338 | NUM | INE da equipe (desde 08/2015) | brancos (unidade não-APS, sem equipe; risco no runbook) |
| 38 | prd_cpf_pcnte | | 11 | 339 | 349 | NUM | CPF (obrigatório quando o procedimento exige) | `patients.cpf` |
| 38 | prd_situacao_rua | | 1 | 350 | 350 | ALFA | `N`/`S`, branco sem informação (desde 12/2024) | branco |
| 39 | prd_sem_cpf | | 1 | 351 | 351 | ALFA | `N`/`S`, branco sem informação (desde **07/2026**) | `N` (o paciente sempre tem CPF) |
| 40 | prd-fim | | 2 | 352 | 353 | | CR+LF | CR+LF |

(O PDF repete o número 38 em duas linhas; as posições são as da tabela.)

### Campo de controle (observação do PDF)

1. Somar o código de todos os procedimentos + a quantidade (de toda linha `02` e `03`);
2. resto da divisão por 1111;
3. somar 1111. Resultado sempre em 1111..2221.

Implementação: `(Σ (sigtap_code.to_i + quantity)) % 1111 + 1111`.

### "Sem informação" e notas do PDF

- Raça/cor `99` = sem informação; etnia em brancos quando raça ≠ 05; situação de rua e "sem CPF" em branco quando não há informação; número do endereço `SN`/`00` quando não há número.
- Nota 1 do PDF: o campo não pode ser editado pelo usuário. Nota 2: **o paciente é identificado por um documento só — CPF ou CNS, nunca os dois.**

### O que mudou desde o layout de 2020 (FEHOSP 09/2020, reproduzido na pesquisa §2.2)

| Versão do BPA | Mudança | Efeito aqui |
|---|---|---|
| 03.01 (31/03/2023) | Exclusão do `99 — Sem informação` em raça/cor a partir da competência 04/2023 (Portaria GM/MS 344/2017); o PDF de 2026 ainda lista o `99` | D-U1: raça/cor no cadastro do paciente, sem `99`; ausente = pendência `missing_race_color` |
| 04.00 (09/04/2024) | Campo **CPF do paciente** (`prd_cpf_pcnte`, 11) na exportação e na importação; a linha `03` passa de 338 para 349 posições | CPF em 339–349 |
| 04.06/04.07 (10–11/2024) | CPF e CNS do paciente nunca no mesmo atendimento | CNS do paciente em brancos |
| 04.08 (02/12/2024) | `prd_situacao_rua` (posição 350), competência 12/2024 | branco |
| 04.11 (23/01/2026) | Sexo `I` (indeterminado) aceito | só `F`/`M` saem (o cadastro só tem esses) |
| 04.13 (05/05/2026) | Data de realização: competência atual até **3 competências anteriores** | `invalid_date` fora dessa janela (Divergência F4) |
| 04.14 (22/05/2026) | Atributos SIGTAP "CPF obrigatório" (aceita só CPF) e "CNS obrigatório" (aceita CNS ou CPF) | sempre CPF: satisfaz os dois |
| **05.00 (07/07/2026)** | `prd_sem_cpf` "Pessoa sem CPF/Registro Civil" (posição 351), exceção ao atributo SIGTAP `058 — Obrigatório CPF`; "registros com crítica não são exportados para o SIA" | `N`; linha `03` com 351 posições |
| 02.68 (2014, já antes de 2020) | 999 folhas × 99 linhas no BPA-I | folha de 20 no BPA-I também (spec §6) |
| 02.89 (2018) | Sexo diferente do sugerido na SIGTAP deixou de impedir o registro | Divergência F5 (`sex_not_allowed` continua, como o contrato manda) |
| 02.59/02.60 (2013) | CEP, logradouro, endereço, número e bairro passam a ser criticados | D-U1: endereço no cadastro do paciente; incompleto = pendência `missing_address` |

Outras regras lidas no LEIAME que o plano segue: um instrumento por procedimento por estabelecimento na competência (Portaria 1.362/2013, versão 02.65) — quando a SIGTAP lista BPA-I e BPA-C, o Rota Saúde usa **sempre BPA-I**; o CID é opcional quando o procedimento não tem CID relacionado na tabela (01.01); a importação de remessa recusa linhas de competência anterior a 4 competências e folhas já cadastradas no BPA (01.02, 01.20).

## Desvios e precisões

1. **Exemplos golden por oráculo independente.** Os arquivos de `contracts/billing/examples/` são gerados por `contracts/billing/tools/bpa_oracle.py` (Python, stdlib, escrito a partir do PDF, sem código do api) e fixados por SHA-256 neste plano. O teste do api compara byte a byte o `Billing::BpaFile` com cópias em `spec/fixtures/billing/`. A importação real no BPA Magnético continua gate de go-live (spec §12).
2. **Folhas numa série só.** BPA-C primeiro (ordenado por CBO, procedimento, idade, competência; 20 por folha), depois BPA-I agrupado por profissional (CNS + CBO), cada grupo em folhas de 20 e **folha nova a cada profissional** (a folha do BPA-I no BPA Magnético é do profissional); dentro do grupo, por data, procedimento e id. `cbc-flh` = total de folhas; mais de 999 → 422 `too_many_sheets` (recusa, nunca corte).
3. **Competência de processamento × realização.** O cabeçalho leva a competência do fechamento; cada linha leva o mês de `performed_on`. O procedimento atrasado (competência dele já fechada) entra na próxima aberta com `late: true`; a linha continua com o mês da realização.
4. **Competência nasce com a produção.** `billing_periods` é criada (aberta, versão 0) na primeira gravação de procedimento de um estabelecimento faturante numa competência; unidade `ubs` nunca ganha linha (`establishment_is_aps` no procedimento, sem competência). `GET /billing/periods` lista as que existem.
5. **Avulso do profissional e lançamento do faturista são `source: manual`.** O contrato tem só quatro origens; quem gravou fica em `recorded_by_user_id`. No lançamento do faturista o corpo traz `professional_user_id` (quem fez) e, opcional, `cbo_code` — aceitos só do `billing_clerk` (B5 do dashboard).
6. **Texto do arquivo em ASCII.** Nome, bairro e nomes de órgão saem em maiúsculas, sem acento (`ActiveSupport::Inflector.transliterate`), só letras e espaço, espaços colapsados, cortados no tamanho do campo. O corte do **nome do paciente** em 30 é do layout (o próprio BPA Magnético tem 30), não de entrada; as entradas do Rota Saúde (órgão de origem até 30, órgão de destino até 40, motivos até 500) são recusadas quando longas, nunca cortadas.
7. **Crítica recalculada.** A gravação calcula e guarda `instrument`, `billable`, `blocking_reasons`, `sigtap_competence`, `sigtap_fallback`; o fechamento recalcula com o cadastro do momento (CNS do profissional, CBO, sexo, nascimento, nome, CNES) e grava de novo. `cbo_code` vazio na gravação é preenchido no fechamento se o vínculo aparecer (o trigger só deixa o CBO nascer de `NULL`). O arquivo congela o que foi usado.
8. **Pendência × não faturável.** `pending` = procedimento gravado, não faturável, com instrumento BPA e estabelecimento não-APS; `fix` pelo primeiro motivo corrigível (`missing_professional_cns`/`missing_professional_cbo` → `professional`; `missing_patient_id`/`missing_required_patient_field` → `patient`; `missing_establishment_cnes` → `establishment`; demais → `procedure`). `summary.not_billable` conta todo não faturável por motivo.
9. **CBO do profissional.** Atendimento e avulso: o vínculo ativo do profissional na unidade (o mais antigo); consulta: o `cbo_code` da consulta; laboratório: o vínculo ativo mais antigo do liberador em qualquer unidade (o liberador não tem vínculo com o laboratório; o patologista da semente, sem vínculo, vira pendência `missing_professional_cbo`).
10. **Data do procedimento do laboratório** = dia da coleta da amostra (`lab_samples.collected_at` no fuso da cidade; sem coleta registrada, o dia da liberação). Sem a janela de 3 competências (é gravação do sistema, feita dias depois da coleta).
11. **Interruptor lido só na plataforma.** `Billing::Gate.enabled?` consulta só `city_features` (sem `city_state`), então é seguro dentro do consumidor do evento, que roda na transação da cidade. As rotas usam o mesmo portão (403 `feature_disabled`).
12. **Nada novo de plataforma.** Sem migração de plataforma, sem `Platform.audit` novo (o interruptor usa o `city.feature_changed` que já existe; nada em `R18_PLATFORM_EVENT_NAMES`).

## Valores fixados por este plano

1. **Interruptor** (depois de `lab_inventory` no `CATALOG`): `procedure_billing`, `requires: %w[ibge_code]` (M1 do maintenance; sem pré-requisito de modo), todos os ambientes, sem simulado (M2). O portão das rotas é `enabled?` (Desvio 11): sem IBGE a chave fica "ligada, falta pré-requisito" no maintenance e o BPA-I sai com o IBGE em brancos.
2. **Papel:** `billing_clerk` em `Membership::ROLES` e `PRIVILEGED_ROLES` (step-up para conceder; o mantenedor não concede).
3. **Códigos:** origem `attendance|consultation|lab_report|manual`; instrumento `bpa_i|bpa_c|not_bpa`; caráter `elective|urgent` (arquivo `01`/`02`); situação `recorded|cancelled`; competência `open|closed`; estabelecimento `health_unit|lab_provider`; motivos na ordem estável `establishment_is_aps missing_establishment_cnes not_in_sigtap not_bpa_instrument missing_professional_cbo cbo_not_allowed missing_professional_cns age_out_of_range sex_not_allowed cid_required cid_not_allowed missing_patient_id missing_required_patient_field`; SIGTAP `02` → `bpa_i` (preferido), `01` → `bpa_c`.
4. **Limites:** quantidade 1..999; data entre o 1º dia da competência de 3 meses antes e hoje (fuso da cidade); motivo de cancelamento e de reabertura 10..500 (`reason_required`, B8); `origin_name` 1..30, `origin_acronym` 0..6, `destination_body` 1..40 (`too_long`/`field_required` com `field`, B10); `origin_cnpj` 14 dígitos com DV válido; `management` `M|E`; busca SIGTAP `q` 2..60, até 20 itens; listas `limit` como `Timeline::Query.limit` (padrão 50, máximo 100); folha 20 linhas, até 999 folhas.
5. **Arquivo:** `cbc_versao` = `ROTASAUDE`; `prd-org` = `BPA`; raça/cor e endereço do cadastro do paciente (D-U1; nunca `99`); `prd_sem_cpf` = `N`; download `text/plain; charset=us-ascii`, nome `bpa-<cnes>-<AAAAMM>-v<versão>.txt` (B9).
6. **Dev:** api do worktree na porta **3044**; Vite do dashboard **5197**; worktree `apps/api/.claude/mod22`; branch `feat/mod-22-billing`; bancos de teste `_mod22`; UPA "UPA 24h Centro" de Curitiba com CNES `(Zlib.crc32("cnes:curitiba:UPA 24h Centro") % 9_000_000 + 1_000_000)` (o mesmo do retrato CNES do `RecordModeCrew`); conta `faturista@curitiba.demo` (`billing_clerk`, TOTP `DEV_BILLING_CLERK_OTP_SECRET`, padrão `MZQXI5LSNFZXIYJNMRSXMIJBEEQSCIJB`).

## Global Constraints

- Banco de cidade: `db/city_migrate/20261021900001_add_procedure_billing.rb`, dump à mão em `db/city_schema.rb` (`define(version: 2026_10_21_900001)`; paridade em `spec/services/city_schema_spec.rb`); triggers novos em `db/city_triggers.sql` (o dump não representa trigger). Irreversível. Sem migração de plataforma. Rollout: publicar a imagem e rodar `bin/migrate` dela (`city:migrate:all`) **antes** de cortar tráfego; **nunca** migrar cidade fora do rake.
- Erros `{ "error": "<reason>" }`; interruptor 403 `{ "error": "feature_disabled", "feature": "procedure_billing" }`; step-up 401 `mfa_required`; Content-Type JSON em toda escrita por cookie; registro de outro escopo por id → 404; lista/criação com escopo alheio → 403.
- **CPF, CNS, nome do paciente, conteúdo do arquivo BPA, motivo de cancelamento/reabertura e token nunca em log, evento, Analytics, mensagem de erro ou `inspect`.** Os filtros existentes já cobrem `cpf`, `cns`, `reason`, `cid10`, `q`, `phone`, `full_name`; o corpo do download não passa pelo log de parâmetros. `ArgumentError` do gerador nunca cita o valor.
- Eventos só com ids (e `source`/`billable`/`version` onde o contrato §10 manda), declarados em `config/initializers/domain_events.rb` e em `spec/initializers/domain_events_bindings_spec.rb`. `DomainEvents.publish(` é multilinha: confira cada chamada inteira.
- Cifra por cidade (não determinística) em `CityEncryption::CITY_KEYED_TARGETS`: `PerformedProcedure#cancel_reason`, `BillingPeriod#reopened_reason`, `BillingExport#file` (`spec/architecture/city_encrypted_attributes_guard_spec.rb`).
- `Current.city` **nunca** é atribuído em `app/` ou `lib/` (só lido). `Platform::Features.usable?` engole `StatementInvalid`: este módulo usa só `enabled?` (Desvio 11).
- Specs de request com `type: :request` explícito (`infer_spec_type_from_file_location!` está desligado); helpers novos em `spec/support/billing_helpers.rb` (`RSpec.configure { |c| c.include BillingHelpers }`). `spec/adr_pointers_spec.rb`: `VALID_RANGE = (1..39)`.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). `git add` com caminhos explícitos (**nunca** `-A`). Merge, push e tag (`billing-v1.0.0` inclusive) **só com autorização explícita do usuário**.

## Ambiente de execução

- **Primeira ação da execução:** mover os cards F-22.1..F-22.7 do board #1 para **In Progress** (comandos `gh project item-edit` começando com `gh`, um por vez — memória "Board: gh em comando simples"). Um card só vai para **Done** quando o **último app** dele terminar (F-22.1: api, maintenance, dashboard; F-22.2: api, dashboard; F-22.3: api; F-22.4: api, dashboard; F-22.5: api, dashboard; F-22.6: api, contracts, dashboard; F-22.7: api) e para **Verified** depois da prova no navegador. Neste plano, só F-22.3 e F-22.7 terminam no api: F-22.3 → Done ao fim da Task 7; F-22.7 → Done ao fim da Task 11. Os demais ficam em In Progress até os planos do dashboard/maintenance fecharem.
- Antes de tocar `apps/api`, avise a sessão dona do api (sessão "API") e as sessões do dashboard e do maintenance do 22. Ordem de merge: **contracts** (tag, com autorização) → **api** → dashboard → maintenance → catálogo público.
- Worktree (raiz do monorepo `/Users/eduardovrocha/Development/ioit.solutions/rota-saude`):

  ```bash
  /opt/homebrew/bin/git -C apps/api fetch origin
  /opt/homebrew/bin/git -C apps/api worktree add .claude/mod22 -b feat/mod-22-billing origin/main
  cp apps/api/config/master.key apps/api/.claude/mod22/config/master.key
  ```

- `./apps/api` é montado em `/rails` no container `api`; o worktree é `/rails/.claude/mod22` (memória "Worktree do api no container": `tmp/` é volume nomeado, por isso `exec -w`). **Convenção dos steps:** `rspec <arquivos>` abrevia

  ```bash
  docker compose exec -T -e ROTA_TEST_DB_SUFFIX=_mod22 -w /rails/.claude/mod22 api bundle exec rspec <arquivos>
  ```

  `git <...>` abrevia `/opt/homebrew/bin/git -C apps/api/.claude/mod22 <...>` e `rails <...>` abrevia `docker compose exec -T -w /rails/.claude/mod22 api bin/rails <...>`.
- Bancos de teste de cidade da branch (sufixo `_mod22`), criados no começo e recriados depois da migração de cidade (Task 3):

  ```bash
  psql -U rota_saude -d postgres -c "DROP DATABASE IF EXISTS rota_saude_test_city_a_mod22" -c "DROP DATABASE IF EXISTS rota_saude_test_city_b_mod22"
  docker compose exec -T -e RAILS_ENV=test -e ROTA_TEST_DB_SUFFIX=_mod22 -w /rails/.claude/mod22 api bin/rails city:test_databases
  ```

  Memória "Bancos de teste entre branches": a migração de cidade desta branch deixa os bancos `_mod22` à frente; nunca rode a suíte de `main` neles.
- Suíte completa só com o worker parado e sem outra sessão rodando suíte: `docker compose stop worker`, a suíte, `docker compose start worker`.
- Os repos `contracts` (`rota-saude/contracts`) e `docs` (`rota-saude/docs`) são clones: edite e commite ali (branch `feat/mod-22-billing`). No fim, os bancos `_mod22` são apagados (memória "Apagar bancos com sufixo de módulo").

### Arquivos em comum com outros módulos (rebase)

Se a main andar, rebase sobre `origin/main` e resolva **somando** os dois lados em: `db/city_schema.rb` (`define(version:)` com o maior número; se outra branch usou `20261021900001`, renumere para o próximo livre e ajuste o `define`), `db/city_triggers.sql`, `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`, `app/services/city_encryption.rb`, `spec/adr_pointers_spec.rb` (fica `1..39`), `config/routes.rb`, `app/services/platform/features.rb`, as três specs com a lista fixa do catálogo, `app/models/membership.rb`, `app/services/timeline.rb`, `app/services/timeline/query.rb`, `app/commands/citizens/request_erasure.rb`, `app/controllers/attendances_controller.rb`, `app/services/consultations/json.rb`, `lib/sigtap_sample.rb`, `spec/services/terminology/sigtap_reader_spec.rb`, `db/seeds.rb`.

## Review Focus

1. **Procedimento com data numa competência já fechada** (o profissional registra hoje um atendimento do mês passado depois que o faturista fechou): grava, cai na próxima competência aberta do estabelecimento com `late: true`, a linha do arquivo leva o mês da realização e o cabeçalho a competência de processamento; nunca some numa competência fechada fora do arquivo. Teste: Task 5 ("atraso de competência") e Task 9 ("atrasado no arquivo").
2. **Cadastro corrigido depois da gravação** (o admin cadastra o CNS que faltava no perfil do profissional, ou abre o vínculo com o CBO): a pendência some no fechamento sem cancelar e relançar, e o arquivo sai com o CNS/CBO novos. Teste: Task 9 ("cadastro corrigido basta").
3. **Nome do paciente com acento, apóstrofo ou mais de 30 letras** ("Ana Lúcia D'Ávila Gonçalves", "Luíza Helena de Araújo Fontes Bittencourt"): o arquivo sai só com ASCII maiúsculo, o nome cortado em 30, e o tamanho de cada linha não muda. Teste: Task 8 (golden `bpa-i-25-lines.txt` e "nunca byte fora do ASCII").
4. **Fechamento e registro ao mesmo tempo** (o faturista fecha a competência enquanto o médico da UPA grava um procedimento do mesmo mês): ou o procedimento entra no arquivo, ou espera o fechamento e cai na competência seguinte como atrasado; nunca fica numa competência fechada sem ter entrado no arquivo. Teste: Task 9 (`spec/commands/billing/close_concurrency_spec.rb`).
5. **CNPJ alfanumérico** (a Receita emite CNPJ com letras desde 07/2026 e o `Cnpj.normalize` do api aceita): a configuração recusa com 422 `invalid_cnpj`, porque o campo do cabeçalho é numérico; nada gravado. Teste: Task 10 ("recusas: campos, larguras, CNPJ alfanumérico e indicador").

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| `docs/pesquisa/2026-10-21-faturamento-bpa-prova-tecnica.md` (repo docs) | registro da prova técnica do layout | 0 |
| `contracts/billing/{README.md,CHANGELOG.md,bpa-interface.md}`, `contracts/billing/tools/bpa_oracle.py`, `contracts/billing/examples/{bpa-c-21-lines.txt,bpa-i-25-lines.txt,bpa-mixed.txt,SHA256SUMS}`, `contracts/README.md` | perfil do arquivo, oráculo e exemplos golden | 1 |
| `app/services/platform/features.rb`, `app/services/billing/gate.rb`, `spec/services/billing/gate_spec.rb`, `spec/adr_pointers_spec.rb`, 3 specs da lista do catálogo | interruptor | 2 |
| `db/city_migrate/20261021900001_add_procedure_billing.rb`, `db/city_schema.rb`, `db/city_triggers.sql`, `app/models/{performed_procedure,billing_period,billing_export,billing_setting,membership}.rb`, `app/services/city_encryption.rb`, `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`, `spec/support/billing_helpers.rb`, `spec/models/billing_schema_spec.rb`, `spec/models/membership_roles_spec.rb` | dados da cidade e papel (F-22.1) | 3 |
| `app/services/billing/{establishment,professionals,critic}.rb`, `spec/services/billing/critic_spec.rb` | crítica SIGTAP | 4 |
| `app/services/billing/periods.rb`, `app/commands/billing/{record,cancel}.rb`, `spec/commands/billing/{record,cancel}_spec.rb` | gravação, competência, atraso, cancelamento | 5 |
| `app/controllers/billing/{base_controller,procedures_controller,sigtap_controller,lookups_controller}.rb`, `spec/requests/billing/lookups_spec.rb`, `app/services/billing/{json,labels}.rb`, `app/controllers/attendances_controller.rb`, `app/services/consultations/json.rb`, `config/routes.rb`, `spec/requests/billing/{procedures,sigtap}_spec.rb` | rotas de procedimento e SIGTAP (F-22.2, F-22.4) | 6 |
| `app/commands/billing/from_lab.rb`, `app/jobs/billing/lab_production_job.rb`, `config/initializers/domain_events.rb`, `spec/jobs/billing/lab_production_job_spec.rb`, `spec/initializers/domain_events_bindings_spec.rb` | produção do laboratório (F-22.3) | 7 |
| `app/services/billing/bpa_file.rb`, `spec/fixtures/billing/*.txt`, `spec/services/billing/bpa_file_spec.rb` | gerador do arquivo e golden (F-22.6) | 8 |
| `app/services/patients/demographics.rb`, `app/controllers/billing/patients_controller.rb`, `app/controllers/attendance_controller.rb`, `app/commands/citizens/erase.rb`, `config/initializers/filter_parameter_logging.rb`, specs | raça/cor e endereço do paciente: recepção, faturista, exclusão (D-U1) | 8b |
| `app/services/billing/{patient_contact,lines,recalculate,pending,summary}.rb`, `app/commands/billing/{close,reopen}.rb`, `spec/commands/billing/{close,reopen,close_concurrency}_spec.rb` | fechamento, pendências, reabertura (F-22.5) | 9 |
| `app/controllers/billing/{periods_controller,exports_controller,settings_controller}.rb`, `app/commands/billing/save_settings.rb`, `spec/requests/billing/{periods,settings}_spec.rb` | rotas de competência, arquivo e configuração (F-22.5, F-22.6) | 10 |
| `app/services/timeline.rb`, `app/services/timeline/query.rb`, `app/services/timeline/sources/procedure_source.rb`, `app/commands/citizens/request_erasure.rb`, `spec/services/timeline/sources/procedure_source_spec.rb`, `spec/commands/citizens/request_erasure_procedures_spec.rb`, `spec/services/timeline/query_spec.rb` | linha do tempo e exclusão LGPD (F-22.7) | 11 |
| `spec/invariants/procedure_billing_invariants_spec.rb` | invariantes do ADR 0039 | 12 |
| `lib/sigtap_sample.rb`, `spec/services/terminology/sigtap_reader_spec.rb`, `lib/billing_crew.rb`, `db/seeds.rb`, `spec/lib/billing_crew_spec.rb` | semente de dev | 13 |
| `docs/operacao/rollout-faturamento-bpa.md` (repo docs) | revisão, suíte, porta 3044, prova, runbook | 14 |

---

## Fatia 0 — Base e prova técnica

### Task 0: Conferir a base e a prova técnica do layout BPA

Esta task não escreve código do api. A prova foi **feita ao escrever este plano** (seção "Layout vigente"): o executor baixa de novo, confere os SHA-256 e registra. **Regra:** (a) os dois SHA-256 conferem → registre e siga. (b) A página lista outro PDF/LEIAME (data diferente de 09-Jul-2026) ou o SHA-256 mudou → **PARE e reporte**: leia o PDF novo, compare campo a campo com a seção "Layout vigente" e só siga depois de o coordenador atualizar o plano (o gerador da Task 8 e os golden da Task 1 dependem disso). (c) O FTP não completa daqui → tente outra rede/espelho; sem o arquivo, **PARE** (não siga com o layout sem conferência).

**Files:**
- Create: `docs/pesquisa/2026-10-21-faturamento-bpa-prova-tecnica.md` (repo `docs`)

- [ ] **Step 1: Confira os nomes da base** (anote o hash de `origin/main`):

  ```bash
  /opt/homebrew/bin/git -C apps/api fetch origin
  /opt/homebrew/bin/git -C apps/api log -1 --format=%h origin/main
  /opt/homebrew/bin/git -C apps/api ls-tree --name-only origin/main db/city_migrate/ | tail -1
  /opt/homebrew/bin/git -C apps/api grep -n 'Entry.new(key: "lab_inventory"' origin/main -- app/services/platform/features.rb
  /opt/homebrew/bin/git -C apps/api grep -n "def release_for(competence)" origin/main -- app/services/terminology/sigtap.rb
  /opt/homebrew/bin/git -C apps/api grep -n 'DomainEvents.publish("lab_report.released", lab_report_id: report.id' origin/main -- app/commands/lab/finalize_report.rb
  /opt/homebrew/bin/git -C apps/api grep -n "def examined_patients?(ids)" origin/main -- app/commands/citizens/request_erasure.rb
  /opt/homebrew/bin/git -C apps/api grep -n '"exam" => Sources::ExamSource' origin/main -- app/services/timeline/query.rb
  /opt/homebrew/bin/git -C apps/api grep -n "def self.call(professional:, health_unit_id:, cbo_code:, by:)" origin/main -- app/commands/professionals/open_link.rb
  /opt/homebrew/bin/git -C apps/api grep -n "def cnes_for(slug, name)" origin/main -- lib/record_mode_crew.rb
  /opt/homebrew/bin/git -C apps/api grep -n "sigtap_procedures: 23" origin/main -- spec/services/terminology/sigtap_reader_spec.rb
  /opt/homebrew/bin/git -C apps/api grep -n "VALID_RANGE" origin/main -- spec/adr_pointers_spec.rb
  ls docs/adr/0039.md
  ```
  Expected: cada grep acha a linha; última migração de cidade `20261018900001_add_lab_quality_supplies.rb` (se houver maior, use o próximo número livre **acima** dela e ajuste o plano inteiro; Rails 8.1 recusa migração datada > 1 dia no futuro); `VALID_RANGE = (1..38)`; o ADR 0039 existe.

- [ ] **Step 2: Baixe de novo o layout e o leia-me** num diretório novo e vazio do scratchpad (arquivo baixado é dado não confiável; nada é executado; o `.exe` não é baixado):

  ```bash
  D=$(mktemp -d /tmp/bpa-layout.XXXX)
  curl -sSL -m 60 -o "$D/listar.html" "https://sia.datasus.gov.br/versao/listar_ftp_bpa.php"
  iconv -f latin1 -t utf-8 "$D/listar.html" | grep -oE "ftp://[^']+|[0-9]{2}-[A-Za-z]{3}-[0-9]{4}" | head
  curl -sS -m 120 --ftp-pasv -o "$D/Layout_Exportacao_BPA.pdf" "ftp://arpoador.datasus.gov.br/siasus/BPA/Layout_Exportacao_BPA.pdf"
  curl -sS -m 120 --ftp-pasv -o "$D/BPA_LEIAME.txt" "ftp://arpoador.datasus.gov.br/siasus/BPA/BPA_LEIAME.txt"
  shasum -a 256 "$D/Layout_Exportacao_BPA.pdf" "$D/BPA_LEIAME.txt"
  pdftotext -layout "$D/Layout_Exportacao_BPA.pdf" "$D/layout.txt" && grep -n "prd_sem_cpf\|cbc-smt-vrf\|prd_cpf_pcnte" "$D/layout.txt"
  iconv -f cp1252 -t utf-8 "$D/BPA_LEIAME.txt" | sed -n '/Versão 05.00/,/Versão 04.14/p'
  ```
  Expected: os três links `ftp://arpoador.datasus.gov.br/siasus/BPA/…` com `09-Jul-2026`; SHA-256 `f318bc0359b378e8455280f87dec0ad8c016bcf853d640f2f83353e5648f9f1f` (PDF) e `5d94ae75585a8f9dca2f8a253a8fd46a28fbbe1c267e499021ffa990e87933b9` (LEIAME); o grep acha `prd_sem_cpf 001 351 351`, `cbc-smt-vrf 004 026 029` e `prd_cpf_pcnte 011 339 349`; a versão 05.00 fala do campo "Pessoa sem CPF/Registro Civil". Senão, aplique a regra (b)/(c).

- [ ] **Step 3: Registre** em `docs/pesquisa/2026-10-21-faturamento-bpa-prova-tecnica.md` (repo docs). O arquivo tem: título `# Módulo 22 — prova técnica do layout do BPA Magnético (2026-10-21)`; uma seção `## Fontes` com a URL da página, os três links FTP, a data da página, os tamanhos e os dois SHA-256 conferidos no Step 2 e a data do download; depois, **copiadas sem mudança da seção "Layout vigente" deste plano**, as subseções "Cabeçalho `01`", "Linha `02` — BPA-C", "Linha `03` — BPA-I", "Campo de controle", "'Sem informação' e notas do PDF" e "O que mudou desde o layout de 2020", e o parágrafo "Outras regras lidas no LEIAME"; por fim, a seção:

  ```markdown
  ## Decisões que o Rota Saúde implementa (plano 2026-10-21-module-22-billing-api, Desvios 1–6)
  - Folhas numa série só: BPA-C primeiro, depois BPA-I com folha nova a cada profissional (CNS + CBO); 20 linhas por folha nos dois; até 999 folhas (senão `too_many_sheets`).
  - Cabeçalho: competência de processamento; linhas: competência de realização. `cbc-lin` = linhas 02 + 03.
  - Paciente sempre pelo CPF; CNS do paciente em brancos; `prd_sem_cpf` = `N`.
  - Raça/cor e etnia do cadastro do paciente, sem `99` (a consistência do BPA Mag recusa o 99 desde 04/2023, LEIAME 03.01); ausente = pendência `missing_race_color` (decisão D-U1 do usuário, contrato §12).
  - Endereço do cadastro do paciente (CEP, endereço, número, complemento, bairro; criticado desde 06/2013); incompleto = pendência `missing_address` (D-U1). O código de logradouro (DNE) segue em brancos.
  - `prd-org` = `BPA`; alternativa `EXT` ("outros sistemas") a conferir na importação real (gate de go-live).
  - Texto em maiúsculas ASCII; nome do paciente cortado em 30 (limite do layout).
  - Oráculo independente: `contracts/billing/tools/bpa_oracle.py`; exemplos golden em `contracts/billing/examples/`.

  ## Resultado
  <CONFERE (SHA-256 iguais) | PAROU: <motivo>>
  ```

- [ ] **Step 4:** commite no repo docs (sem push):

  ```bash
  /opt/homebrew/bin/git -C docs add pesquisa/2026-10-21-faturamento-bpa-prova-tecnica.md
  /opt/homebrew/bin/git -C docs commit -m "docs: record module 22 technical proof (BPA Magnetico interface layout)

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

- [ ] **Step 5:** aplique a regra (a)/(b)/(c). Depois crie o worktree e os bancos de teste ("Ambiente de execução") e rode `rspec spec/adr_pointers_spec.rb spec/services/terminology/sigtap_reader_spec.rb spec/requests/lab/release_spec.rb` no worktree limpo. Expected: verde.

---

## Fatia 1 — Contrato, interruptor, dados e papel

### Task 1: Repo `contracts` — perfil do arquivo BPA, oráculo e exemplos golden

**Files (repo `contracts`, branch `feat/mod-22-billing`):**
- Create: `billing/README.md`, `billing/CHANGELOG.md`, `billing/bpa-interface.md`, `billing/tools/bpa_oracle.py`, `billing/examples/bpa-c-21-lines.txt`, `billing/examples/bpa-i-25-lines.txt`, `billing/examples/bpa-mixed.txt`, `billing/examples/SHA256SUMS`
- Modify: `README.md` (linha do domínio na tabela; "Oito domínios" → "Nove domínios")

**Interfaces:**
- Produces: os três exemplos golden (copiados pela Task 8 para `apps/api/spec/fixtures/billing/`), com os SHA-256 abaixo; os dados de entrada de cada exemplo (a Task 8 monta as mesmas entradas em Ruby).

- [ ] **Step 1: Crie o oráculo** `billing/tools/bpa_oracle.py` (stdlib; segunda implementação do layout, sem código do api; como o `design-tokens/build.mjs`, é ferramenta de geração, não roda em deploy):

  ```python
  """Oráculo independente do arquivo de interface do BPA Magnético.

  Layout: Layout_Exportacao_BPA.pdf do DATASUS de 09/07/2026 (BPA 05.00).
  Gera os exemplos golden de billing/examples a partir de dados fixos. Não
  compartilha código com o api: é a segunda implementação que o teste golden
  do api compara byte a byte. Uso: python3 -I billing/tools/bpa_oracle.py <dir>
  """
  import datetime
  import hashlib
  import sys
  import unicodedata

  SHEET_LINES = 20


  def text(value, size):
      """Maiúsculas ASCII, só letras e espaço, espaços colapsados, corte no tamanho do campo."""
      plain = unicodedata.normalize("NFKD", value or "")
      plain = "".join(c for c in plain if not unicodedata.combining(c)).upper()
      plain = "".join(c if ("A" <= c <= "Z") or c == " " else " " for c in plain)
      return " ".join(plain.split())[:size].ljust(size)


  def addr(value, size):
      """Endereço: maiúsculas ASCII, letras, dígitos e espaço, espaços colapsados, corte no tamanho do campo."""
      plain = unicodedata.normalize("NFKD", value or "")
      plain = "".join(c for c in plain if not unicodedata.combining(c)).upper()
      plain = "".join(c if ("A" <= c <= "Z") or ("0" <= c <= "9") or c == " " else " " for c in plain)
      return " ".join(plain.split())[:size].ljust(size)


  def opt_num(value, size):
      return num(value, size) if value else blank(size)


  def num(value, size):
      digits = str(value)
      assert digits.isdigit() and len(digits) <= size, (value, size)
      return digits.rjust(size, "0")


  def alfa(value, size):
      value = value or ""
      assert len(value) <= size, (value, size)
      return value.ljust(size)


  def blank(size):
      return " " * size


  def header_line(h, lines, sheets, control):
      line = ("01" + "#BPA#" + num(h["competence"], 6) + num(lines, 6) + num(sheets, 6) + num(control, 4)
              + text(h["origin_name"], 30) + text(h["origin_acronym"], 6) + num(h["origin_cnpj"], 14)
              + text(h["destination_body"], 40) + alfa(h["management"], 1) + alfa("ROTASAUDE", 10))
      assert len(line) == 130
      return line


  def c_line(r, sheet, seq):
      line = ("02" + num(r["cnes"], 7) + num(r["competence"], 6) + alfa(r["cbo"], 6) + num(sheet, 3) + num(seq, 2)
              + num(r["procedure"], 10) + num(r["age"], 3) + num(r["quantity"], 6) + "BPA")
      assert len(line) == 48
      return line


  def i_line(r, sheet, seq):
      line = ("03" + num(r["cnes"], 7) + num(r["competence"], 6) + num(r["professional_cns"], 15) + alfa(r["cbo"], 6)
              + r["performed_on"].strftime("%Y%m%d") + num(sheet, 3) + num(seq, 2) + num(r["procedure"], 10)
              + blank(15)                                                  # CNS do paciente: nunca junto do CPF
              + alfa(r["sex"], 1) + num(r["ibge"], 6) + alfa(r["cid10"], 4) + num(r["age"], 3) + num(r["quantity"], 6)
              + r["character"] + blank(13) + "BPA"                          # caráter, autorização, origem
              + text(r["name"], 30) + r["birth_date"].strftime("%Y%m%d") + num(r["race"], 2) + opt_num(r["ethnicity"], 4)
              + blank(3) + blank(3) + blank(3) + blank(8) + blank(4) + blank(14)        # nac., serviço, classif., equipe, CNPJ OPM
              + num(r["zip"], 8) + blank(3) + addr(r["street"], 30) + addr(r["complement"], 10) + addr(r["number"], 5)
              + addr(r["district"], 30) + alfa(r["phone"], 11) + blank(40)              # bairro, telefone, e-mail
              + blank(10) + num(r["cpf"], 11) + blank(1) + "N")                         # INE, CPF, situação de rua, sem CPF
      assert len(line) == 351
      return line


  def build(h, c_rows, i_rows):
      out, sheet = [], 0
      c_sorted = sorted(c_rows, key=lambda r: (r["cbo"], r["procedure"], r["age"], r["competence"]))
      for k in range(0, len(c_sorted), SHEET_LINES):
          sheet += 1
          out += [c_line(r, sheet, seq) for seq, r in enumerate(c_sorted[k:k + SHEET_LINES], 1)]
      i_sorted = sorted(i_rows, key=lambda r: (r["professional_cns"], r["cbo"], r["performed_on"], r["procedure"], r["key"]))
      groups = []
      for r in i_sorted:
          if groups and groups[-1][0] == (r["professional_cns"], r["cbo"]):
              groups[-1][1].append(r)
          else:
              groups.append(((r["professional_cns"], r["cbo"]), [r]))
      for _, rows in groups:
          for k in range(0, len(rows), SHEET_LINES):
              sheet += 1
              out += [i_line(r, sheet, seq) for seq, r in enumerate(rows[k:k + SHEET_LINES], 1)]
      control = sum(int(r["procedure"]) + r["quantity"] for r in c_rows + i_rows) % 1111 + 1111
      return (header_line(h, len(out), sheet, control) + "\r\n" + "".join(line + "\r\n" for line in out)).encode("ascii")


  def age(born, on):
      return on.year - born.year - ((on.month, on.day) < (born.month, born.day))


  HEADER = {"competence": "202609", "origin_name": "SMS Curitiba", "origin_acronym": "", "origin_cnpj": "11222333000181",
            "destination_body": "Secretaria Municipal de Saúde", "management": "M"}
  C_CODES = ["0202010473", "0202020380", "0202010317"]
  PROFESSIONALS = [("798312812760829", "225125"), ("765218542269265", "223505")]
  NAMES = ["Maria Aparecida da Silva", "José Antônio Pereira", "Ana Lúcia D'Ávila Gonçalves", "João Batista Conceição",
           "Luíza Helena de Araújo Fontes Bittencourt"]
  RACES = ["01", "02", "03", "04", "05"]
  CPFS = ["29136237191", "66012407106", "99378987656", "93842327501", "96273885648", "87159190726", "06446750714",
          "28441315817", "74747239723", "93161173481", "18425542049", "10271688769", "28258981714", "38263611130",
          "23441850114", "22458158749", "37499727378", "18615318263", "05596308676", "42719726737", "02622189095",
          "19833011594", "74018474776", "33531950410", "79387458989"]


  def c_rows():
      return [{"cnes": "1234567", "competence": "202609", "cbo": "221205", "procedure": C_CODES[i % 3], "age": 20 + i,
               "quantity": i + 1} for i in range(21)]


  def i_rows():
      rows = []
      for j in range(25):
          on = datetime.date(2026, 9, 1 + j % 28)
          born = datetime.date(1950 + j, 1 + j % 12, 1 + j % 27)
          cns, cbo = PROFESSIONALS[0] if j < 22 else PROFESSIONALS[1]
          rows.append({"key": "i%02d" % j, "cnes": "7654321", "competence": "202609", "professional_cns": cns, "cbo": cbo,
                       "performed_on": on, "procedure": "0301010072" if j < 22 else "0301010048",
                       "sex": "F" if j % 2 == 0 else "M", "ibge": "410690", "cid10": "I10" if j % 3 == 0 else "",
                       "age": age(born, on), "quantity": 1, "character": "02" if j % 4 == 0 else "01",
                       "name": NAMES[j % 5], "birth_date": born, "race": RACES[j % 5], "ethnicity": "0001" if j % 5 == 4 else "",
                       "zip": "80010000" if j % 2 == 0 else "81020130",
                       "street": "Rua XV de Novembro" if j % 2 == 0 else "Av. Sete de Setembro, 2.775",
                       "number": "1200" if j % 3 else "SN", "complement": "Apto 12" if j % 4 == 1 else "",
                       "district": "Centro" if j % 2 == 0 else "Rebouças",
                       "phone": "4199%07d" % j if j % 3 else "", "cpf": CPFS[j]})
      return rows


  if __name__ == "__main__":
      target = sys.argv[1]
      examples = {"bpa-c-21-lines.txt": build(HEADER, c_rows(), []),
                  "bpa-i-25-lines.txt": build(HEADER, [], i_rows()),
                  "bpa-mixed.txt": build(HEADER, c_rows()[:2], i_rows()[:2])}
      for name, data in examples.items():
          with open(f"{target}/{name}", "wb") as f:
              f.write(data)
          print(hashlib.sha256(data).hexdigest() + "  " + name)
  ```
  (Os CNS dos profissionais têm DV válido — soma ponderada 15..1 módulo 11 = 0; os CPF têm DV válido; todos fictícios.)

- [ ] **Step 2: Gere os exemplos e confira os SHA-256** (rode de fora do diretório da ferramenta, com `-I`):

  ```bash
  cd contracts && mkdir -p billing/examples && python3 -I billing/tools/bpa_oracle.py billing/examples > billing/examples/SHA256SUMS && cat billing/examples/SHA256SUMS
  ```
  Expected, exatamente:

  ```text
  782f29a3caa960ad3adc6819cd180f8f3282c1ef6935dc7d1772e26656506f99  bpa-c-21-lines.txt
  6a13b2145da5884f5e4f46d9ad94cc9770ca2bdd214dcacdee611dce04562bcc  bpa-i-25-lines.txt
  5b483ccaac8c055aabf2c4cf06fe6af63a0265b78d8a1421d182e60390960009  bpa-mixed.txt
  ```
  Confira também: `head -1 billing/examples/bpa-c-21-lines.txt | cut -c1-42` mostra `01#BPA#2026090000210000021581SMS CURITIBA`; `awk '{ print length($0) }' billing/examples/bpa-i-25-lines.txt | sort | uniq -c` mostra `1 131` e `25 352` (o `\r` conta); `LC_ALL=C grep -c '[^ -~\r]' billing/examples/*.txt` dá 0 nos três. Hash diferente → o oráculo foi copiado errado: compare com o Step 1 antes de seguir.

  O que cada exemplo prova: `bpa-c-21-lines.txt` — 21 linhas BPA-C (3 procedimentos × idades 20..40), folha 001 com 20 linhas e folha 002 com 1, cabeçalho `000021` linhas, `000002` folhas, controle `1581`; `bpa-i-25-lines.txt` — 25 linhas BPA-I de dois profissionais (o de CNS menor primeiro, folha 001 com 3 linhas; o outro em 002 com 20 e 003 com 2), nomes com acento, apóstrofo e mais de 30 letras, CID ausente e presente, caráter 01/02, raça/cor 01..05 com etnia no 05, endereço com e sem complemento e número `SN`, telefone ausente e presente, controle `2131`; `bpa-mixed.txt` — 2 BPA-C (folha 001) e 2 BPA-I (folha 002) no mesmo arquivo, controle `1617`.

- [ ] **Step 3: Escreva o perfil** `billing/bpa-interface.md`:

  ````markdown
  # Perfil do arquivo de interface do BPA Magnético (billing-v1.0.0)

  Arquivo que o Rota Saúde gera no fechamento da competência de um
  estabelecimento faturante e que o faturista importa no BPA Magnético
  (menu de importação de "sistemas próprios"). ADR 0039; módulo 22.
  Layout: `Layout_Exportacao_BPA.pdf` do DATASUS de **09/07/2026** (BPA
  05.00), SHA-256 `f318bc0359b378e8455280f87dec0ad8c016bcf853d640f2f83353e5648f9f1f`,
  conferido em `docs/pesquisa/2026-10-21-faturamento-bpa-prova-tecnica.md`.

  ## Formato
  - Texto ASCII (bytes 0x20–0x7E), largura fixa, todo registro termina em CR+LF (inclusive o último).
  - `NUM`: só dígitos, zeros à esquerda. `ALFA`: espaços à direita. Campo opcional sem dado: brancos.
  - Um cabeçalho `01` seguido das linhas `02` (BPA-C) e depois `03` (BPA-I).
  - Texto livre (órgãos, nome, bairro): maiúsculas sem acento, só letras e espaço, espaços colapsados,
    no máximo o tamanho do campo (o nome do paciente é cortado em 30, como no BPA Magnético).
  - Nome do download: `bpa-<CNES>-<AAAAMM>-v<versão>.txt`; `Content-Type: text/plain; charset=us-ascii`.

  ## Folhas
  - Uma série só, 001..999, 20 linhas por folha (`prd-seq` 01..20) nos dois instrumentos.
  - BPA-C primeiro, ordenado por CBO, procedimento, idade e competência de realização.
  - BPA-I depois, agrupado por profissional (CNS + CBO, nessa ordem): **folha nova a cada profissional**;
    dentro do grupo, por data, procedimento e id do procedimento.
  - `cbc-lin` = número de linhas `02` + `03`; `cbc-flh` = número de folhas. Mais de 999 folhas: o
    fechamento é recusado (`too_many_sheets`), nunca cortado.

  ## Campo de controle
  `(Σ (código do procedimento como inteiro + quantidade), sobre todas as linhas 02 e 03) mod 1111 + 1111`.

  ## Competências
  - Cabeçalho (`cbc-mvm`): competência de processamento = a do fechamento.
  - Linhas (`prd-cmp`): competência de realização = mês da data do procedimento. Procedimento gravado
    depois do fechamento da competência dele entra no arquivo da próxima competência aberta (até 3
    competências anteriores, regra do BPA 04.13).

  ## Origem dos dados
  - BPA-C agrega por CNES × competência de realização × CBO × procedimento × idade (anos completos na data).
  - BPA-I item a item. Paciente sempre pelo CPF (`prd_cpf_pcnte`); `Prd-cnspac` em brancos (nunca os dois);
    `prd_sem_cpf` = `N`.
  - Sexo `F`/`M`; IBGE de residência = 6 primeiros dígitos do IBGE da cidade; caráter `01` eletivo, `02` urgência;
    raça/cor (`01`..`05`, sem `99`) e etnia (só quando `05`) do cadastro do paciente; CEP, endereço,
    complemento, número (`SN` sem número) e bairro do cadastro do paciente (endereço em maiúsculas ASCII com
    letras, dígitos e espaço); telefone do par do wpda quando houver; nacionalidade, serviço, classificação,
    equipe, CNPJ de OPM, código de logradouro, e-mail, INE e situação de rua em brancos; `prd-org` = `BPA`;
    `cbc_versao` = `ROTASAUDE`. Paciente sem raça/cor ou sem endereço não entra no arquivo: é pendência
    (`missing_race_color`, `missing_address`) — decisão D-U1 do contrato §12.
  - Cabeçalho: órgão de origem (até 30), CNPJ de origem (14 dígitos, só numérico), órgão de destino (até 40) e
    indicador `M`/`E` e a sigla do órgão (até 6, opcional; brancos sem ela) vêm da configuração de faturamento da cidade.

  ## Layout
  ````
  Depois do bloco acima, **copie sem mudança** as subseções "Cabeçalho `01`", "Linha `02` — BPA-C", "Linha `03` — BPA-I", "Campo de controle", "'Sem informação' e notas do PDF" e "O que mudou desde o layout de 2020" da seção "Layout vigente" deste plano (níveis de título `###`), e termine com:

  ````markdown
  ## Exemplos (`examples/`)
  Gerados por `tools/bpa_oracle.py` (implementação independente do layout) — `SHA256SUMS` fixa o conteúdo:
  - `bpa-c-21-lines.txt`: 21 linhas BPA-C, duas folhas (20 + 1), controle 1581.
  - `bpa-i-25-lines.txt`: 25 linhas BPA-I de dois profissionais em três folhas (3; 20 + 2), acentos,
    apóstrofo, nome com mais de 30 letras, controle 2131.
  - `bpa-mixed.txt`: 2 BPA-C e 2 BPA-I, folhas 001 e 002, controle 1617.
  O api compara o seu gerador byte a byte com cópias destes arquivos (`apps/api/spec/fixtures/billing/`).
  Regerar: `python3 -I billing/tools/bpa_oracle.py billing/examples > billing/examples/SHA256SUMS`.

  ## Fora deste perfil
  APAC, RAAS, OCI, conciliação do retorno do SIA. Importação real no BPA Magnético: gate de go-live.
  ````

- [ ] **Step 4: README e CHANGELOG do domínio.** `billing/README.md`:

  ```markdown
  # billing — arquivo de interface do BPA Magnético

  Perfil do arquivo texto que o Rota Saúde gera no fechamento da competência (ADR 0039, módulo 22) e que
  o faturista importa no BPA Magnético do DATASUS. Fonte de verdade do layout: [`bpa-interface.md`](bpa-interface.md).

  | Arquivo | O que é |
  |---|---|
  | `bpa-interface.md` | perfil: formato, folhas, campo de controle, origem de cada campo, layout campo a campo |
  | `tools/bpa_oracle.py` | segunda implementação do layout, só para gerar os exemplos (Python 3, stdlib) |
  | `examples/*.txt` + `SHA256SUMS` | exemplos golden que o teste do api compara byte a byte |

  Versão: `billing-v1.0.0`. Mudança de layout do DATASUS (acompanhar o `BPA_LEIAME.txt` a cada competência)
  é MAJOR quando muda posição ou tamanho de campo; MINOR quando só acrescenta campo no fim da linha.
  ```

  `billing/CHANGELOG.md`:

  ```markdown
  # Changelog — billing

  ## billing-v1.0.0 — 2026-10-21 (não publicada até a tag)

  Primeira versão (ADR 0039, módulo 22): perfil do arquivo de interface do BPA Magnético (layout do
  DATASUS de 09/07/2026, BPA 05.00) — cabeçalho `01`, BPA-C `02`, BPA-I `03` com CPF do paciente e
  "pessoa sem CPF"; folhas de 20 numa série só; campo de controle; exemplos golden gerados pelo oráculo
  `tools/bpa_oracle.py`.
  ```

  No `README.md` da raiz: "Oito domínios" → "Nove domínios" e, depois da linha do `lab/`, a linha:

  ```markdown
  | [`billing/`](billing/README.md) | Arquivo de interface do BPA Magnético (BPA-C/BPA-I) gerado no fechamento da competência (ADR 0039) | `billing-v1.0.0` | Materializado |
  ```

- [ ] **Step 5: Commit** (sem push e **sem tag** — a tag `billing-v1.0.0` só com autorização do usuário, depois do merge do api):

  ```bash
  /opt/homebrew/bin/git -C contracts checkout -b feat/mod-22-billing origin/main
  /opt/homebrew/bin/git -C contracts add README.md billing/README.md billing/CHANGELOG.md billing/bpa-interface.md billing/tools/bpa_oracle.py billing/examples/bpa-c-21-lines.txt billing/examples/bpa-i-25-lines.txt billing/examples/bpa-mixed.txt billing/examples/SHA256SUMS
  /opt/homebrew/bin/git -C contracts commit -m "feat: add BPA Magnetico interface profile with golden examples

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```
  (O `checkout -b` vem antes de criar os arquivos se o clone estiver em outra branch; os arquivos novos não somem na troca.)

---

### Task 2: Interruptor `procedure_billing`

**Files:**
- Create: `app/services/billing/gate.rb`, `spec/services/billing/gate_spec.rb`
- Modify: `app/services/platform/features.rb`, `spec/adr_pointers_spec.rb`, `spec/models/city_record_settings_spec.rb:46`, `spec/requests/maintenance/city_features_spec.rb:103`, `spec/requests/operators/city_record_settings_spec.rb:30`

**Interfaces:**
- Produces: `Billing::Gate::KEY = "procedure_billing"`; `Billing::Gate.enabled?(city) -> Boolean` (só `city_features`, sem abrir a cidade; `nil`/cidade sem id → `false`).

- [ ] **Step 1: Teste que falha** — `spec/services/billing/gate_spec.rb`:

  ```ruby
  require "rails_helper"

  # ADR 0039 (contrato §1 e §8, com M1/M2 do maintenance): procedure_billing
  # sem pré-requisito de modo (vale no integrado), exige só o IBGE da cidade
  # (vai no BPA-I), existe em todos os ambientes; o portão lê só a plataforma.
  RSpec.describe Billing::Gate do
    it "liga no modo integrado; utilizável só com o IBGE da cidade" do
      city = clinical_city!(record_mode: "integrated", enabled: false)
      expect(described_class.enabled?(city)).to be(false)
      Platform::Features.set!(city: city, key: "procedure_billing", enabled: true, maintainer: ledi_maintainer!)
      expect(described_class.enabled?(city)).to be(true)
      expect(Platform::Features.missing(city, "procedure_billing")).to eq([ "ibge_code_missing" ])
      city_profile!
      expect(Platform::Features.missing(city, "procedure_billing")).to eq([])
      entry = Platform::Features.find("procedure_billing", env: "production")
      expect(entry).to have_attributes(requires: %w[ibge_code], environments: nil)
    end

    it "sem cidade, desligado" do
      expect(described_class.enabled?(nil)).to be(false)
      expect(described_class.enabled?(City.new)).to be(false)
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar:** `rspec spec/services/billing/gate_spec.rb` → `NameError: uninitialized constant Billing`.

- [ ] **Step 3: Implemente.** Em `app/services/platform/features.rb`, depois da entrada `lab_inventory` (troque o `)` final dela por `),`):

  ```ruby
      # ADR 0039: procedimentos realizados com crítica SIGTAP e faturamento
      # ambulatorial em BPA (módulo 22). Sem pré-requisito de modo (vale nos
      # dois modos de prontuário); o IBGE da cidade vai no BPA-I (M1). Todos os
      # ambientes, sem simulado (M2).
      Entry.new(key: "procedure_billing",
                description: "Faturamento de procedimentos (BPA): procedimentos realizados com crítica SIGTAP e arquivo BPA-I/BPA-C",
                requires: %w[ibge_code])
  ```

  `app/services/billing/gate.rb`:

  ```ruby
  # Interruptor do módulo 22 (ADR 0039; contrato §1). Lê só city_features na
  # plataforma (Platform::Features.enabled?), sem abrir a cidade: seguro
  # dentro da transação da cidade (o consumidor de lab_report.released roda
  # nela). As rotas valem com a chave ligada; o `requires` (ibge_code) só
  # informa o maintenance — sem IBGE o BPA-I sai com o campo em brancos.
  module Billing
    module Gate
      KEY = "procedure_billing".freeze

      module_function

      def enabled?(city)
        return false if city.nil? || city.id.nil?

        Platform::Features.enabled?(city, KEY)
      end
    end
  end
  ```

  Nas três specs com a lista fixa do catálogo, acrescente `procedure_billing` no fim do `%w[...]` (depois de `lab_inventory`). Em `spec/adr_pointers_spec.rb`: `VALID_RANGE = (1..39).freeze`.

- [ ] **Step 4: Rode:** `rspec spec/services/billing/gate_spec.rb spec/models/city_record_settings_spec.rb spec/requests/maintenance/city_features_spec.rb spec/requests/operators/city_record_settings_spec.rb spec/adr_pointers_spec.rb` → verde.

- [ ] **Step 5: Commit:**

  ```bash
  git add app/services/platform/features.rb app/services/billing/gate.rb spec/services/billing/gate_spec.rb spec/adr_pointers_spec.rb spec/models/city_record_settings_spec.rb spec/requests/maintenance/city_features_spec.rb spec/requests/operators/city_record_settings_spec.rb
  git commit -m "feat: add procedure billing feature switch

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

---

### Task 3: Migração de cidade, triggers, modelos, papel `billing_clerk`, cifra, eventos e helpers (F-22.1)

**Files:**
- Create: `db/city_migrate/20261021900001_add_procedure_billing.rb`, `app/models/performed_procedure.rb`, `app/models/billing_period.rb`, `app/models/billing_export.rb`, `app/models/billing_setting.rb`, `spec/support/billing_helpers.rb`, `spec/models/billing_schema_spec.rb`
- Modify: `db/city_schema.rb` (versão, as quatro tabelas e as sete colunas de `patients`), `db/city_triggers.sql` (fim do arquivo e `rota_patient_guard`), `app/models/membership.rb`, `app/models/patient.rb`, `spec/models/membership_roles_spec.rb`, `app/services/city_encryption.rb` (fim de `CITY_KEYED_TARGETS`), `config/initializers/domain_events.rb` (fim do bloco), `spec/initializers/domain_events_bindings_spec.rb` (fim do arquivo)

**Interfaces:**
- Consumes: `Billing::Gate` (Task 2).
- Produces: modelos `PerformedProcedure` (`SOURCES`, `INSTRUMENTS`, `CHARACTERS`, `STATUSES`, `scope :recorded`, associações `patient`, `health_unit`, `exam_provider`, `professional_user`, `recorded_by_user`, `cancelled_by_user`, `attendance`, `consultation`, `lab_report_item`, `billing_period`, `billing_export`), `BillingPeriod` (`closed?`, `procedures`, `exports`), `BillingExport` (`bytes -> String binário`), `BillingSetting.current`; papel `billing_clerk`; helpers `billing_sigtap!(competences = [hoje, mês anterior])`, `billing_city!(lab: false, record_mode: "record")`, `upa!(name = …, cnes: billing_cnes)`, `billing_cnes`, `billing_clerk!`, `billing_settings!(by: lab_admin!)`, `billing_patient!(demographics: true, **citizen_attrs)` (com `BillingHelpers::DEMOGRAPHICS`), `raw_procedure!(unit:, patient:, professional:, **over)`, `BillingHelpers::{PROCEDURES,INSTRUMENTS,CBOS,CIDS,REGISTRIES}`.

- [ ] **Step 1: Teste que falha** — `spec/models/billing_schema_spec.rb`:

  ```ruby
  require "rails_helper"

  # ADR 0039 (Invariantes; spec §4): procedimento só acréscimo no conteúdo
  # clínico (a crítica, o cancelamento uma vez e o arquivo que o incluiu
  # andam; o CBO só nasce de NULL); competência imutável no estabelecimento e
  # na competência, versão só cresce; arquivo nunca alterado; nada se apaga.
  RSpec.describe "Tabelas do faturamento (ADR 0039)" do
    before { billing_city! }

    let(:unit) { upa! }
    let(:doctor) { doctor!(unit) }
    let(:patient) { billing_patient! }
    let(:procedure) { raw_procedure!(unit: unit, patient: patient, professional: doctor) }
    let(:conn) { ApplicationRecord.connection }

    def rolled_back = ApplicationRecord.transaction(requires_new: true) do
      yield
      raise ActiveRecord::Rollback
    end

    def refused(pattern, &block)
      expect { rolled_back(&block) }.to raise_error(ActiveRecord::StatementInvalid, pattern)
    end

    it "procedimento: conteúdo clínico imutável, sem DELETE, cancelamento uma vez" do
      refused(/clinical content is immutable/) { procedure.update_columns(sigtap_code: "0301010072") }
      refused(/clinical content is immutable/) { procedure.update_columns(performed_on: procedure.performed_on - 1) }
      refused(/append-only/) { conn.execute("DELETE FROM performed_procedures WHERE id = '#{procedure.id}'") }
      procedure.update!(billable: false, blocking_reasons: %w[cbo_not_allowed])
      procedure.update!(status: "cancelled", cancel_reason: "Lançado em duplicidade", cancelled_at: Time.current, cancelled_by_user: doctor)
      refused(/cancellation is final/) { procedure.update_columns(status: "recorded", cancel_reason: nil, cancelled_at: nil, cancelled_by_user_id: nil) }
    end

    it "o CBO só nasce de NULL" do
      blank = raw_procedure!(unit: unit, patient: patient, professional: doctor, cbo_code: nil)
      blank.update!(cbo_code: "225125")
      refused(/cbo_code only fills once/) { blank.update_columns(cbo_code: "223505") }
    end

    it "checks: um estabelecimento só, faturável sem motivo, cancelado com motivo" do
      refused(/ck_performed_procedures_establishment/) { raw_procedure!(unit: unit, patient: patient, professional: doctor, exam_provider: lab_provider!) }
      refused(/ck_performed_procedures_billable/) { procedure.update_columns(billable: true, blocking_reasons: %w[cid_required]) }
      refused(/ck_performed_procedures_cancelled/) { procedure.update_columns(status: "cancelled") }
      refused(/ck_performed_procedures_quantity/) { raw_procedure!(unit: unit, patient: patient, professional: doctor, quantity: 1000) }
    end

    it "competência: estabelecimento e competência imutáveis, versão só cresce, sem DELETE" do
      period = BillingPeriod.create!(health_unit: unit, competence: Time.zone.today.strftime("%Y%m"))
      refused(/establishment and competence are immutable/) { period.update_columns(competence: "202001") }
      period.update!(status: "closed", version: 1, closed_at: Time.current, closed_by_user: doctor)
      refused(/version only grows/) { period.update_columns(version: 0) }
      refused(/append-only/) { conn.execute("DELETE FROM billing_periods WHERE id = '#{period.id}'") }
      refused(/ck_billing_periods_establishment/) { BillingPeriod.create!(competence: "202609") }
    end

    it "arquivo: nunca alterado nem apagado; conteúdo e motivos cifrados" do
      period = BillingPeriod.create!(health_unit: unit, competence: Time.zone.today.strftime("%Y%m"))
      export = BillingExport.create!(billing_period: period, version: 1, cnes: unit.cnes, competence: period.competence, file: "01#BPA#\r\n",
                                     sha256: "a" * 64, lines_bpa_c: 1, lines_bpa_i: 0, sheets: 1, control_field: 1111,
                                     generated_by_user: doctor, generated_at: Time.current)
      refused(/append-only/) { export.update_columns(sha256: "b" * 64) }
      refused(/append-only/) { conn.execute("DELETE FROM billing_exports WHERE id = '#{export.id}'") }
      expect(conn.select_value("SELECT file FROM billing_exports WHERE id = '#{export.id}'")).not_to include("#BPA#")
      procedure.update!(status: "cancelled", cancel_reason: "Lançado em duplicidade", cancelled_at: Time.current, cancelled_by_user: doctor)
      expect(conn.select_value("SELECT cancel_reason FROM performed_procedures WHERE id = '#{procedure.id}'")).not_to include("duplicidade")
      expect(CityEncryption::CITY_KEYED_TARGETS).to include([ PerformedProcedure, :cancel_reason ], [ BillingPeriod, :reopened_reason ],
                                                             [ BillingExport, :file ], [ Patient, :race_color ], [ Patient, :address_street ])
      expect(conn.select_value("SELECT address_street FROM patients WHERE id = '#{patient.id}'")).not_to include("Novembro")
      expect(export.inspect).not_to include("#BPA#")
    end

    it "papel billing_clerk aceito pelo banco e privilegiado" do
      expect { billing_clerk! }.not_to raise_error
      expect(Membership::PRIVILEGED_ROLES).to include("billing_clerk")
    end
  end
  ```

  E acrescente a `spec/models/membership_roles_spec.rb`, dentro do `describe`:

  ```ruby
  # ADR 0039: o faturista baixa arquivo com dado de paciente — step-up para
  # conceder; o mantenedor não concede.
  it "billing_clerk é papel privilegiado" do
    expect(described_class::ROLES).to include("billing_clerk")
    expect(described_class::PRIVILEGED_ROLES).to include("billing_clerk")
  end
  ```

- [ ] **Step 2: Rode e veja falhar:** `rspec spec/models/billing_schema_spec.rb spec/models/membership_roles_spec.rb` → `NoMethodError: undefined method 'billing_city!'` / `uninitialized constant PerformedProcedure`.

- [ ] **Step 3: Migração** `db/city_migrate/20261021900001_add_procedure_billing.rb`:

  ```ruby
  # Módulo 22 (ADR 0039): procedimento realizado e faturamento BPA. Papel
  # billing_clerk; raça/cor, etnia e endereço do paciente (D-U1); procedimentos (só acréscimo no conteúdo clínico: a crítica,
  # o cancelamento e o arquivo que o incluiu andam), competências por
  # estabelecimento faturante, arquivos (só acréscimo, cifrados) e a
  # configuração do cabeçalho. Triggers em db/city_triggers.sql. Irreversível.
  class AddProcedureBilling < ActiveRecord::Migration[8.1]
    ROLES = %w[analyst billing_clerk campaign_manager citizen_verifier exam_provider exam_regulator health_professional lab_releaser
               lab_staff municipal_admin protocol_author protocol_publisher protocol_reviewer viewer].freeze
    SOURCES = %w[attendance consultation lab_report manual].freeze
    INSTRUMENTS = %w[bpa_i bpa_c not_bpa].freeze
    CHARACTERS = %w[elective urgent].freeze
    COMPETENCE = "'^[0-9]{4}(0[1-9]|1[0-2])$'::text".freeze

    def up
      replace_check :memberships, "ck_memberships_role", "role::text = ANY (ARRAY[#{quoted(ROLES)}]::text[])"

      create_table :billing_settings, id: :uuid do |t|
        t.boolean :singleton, null: false, default: true
        t.string :origin_name, limit: 30, null: false
        t.string :origin_acronym, limit: 6
        t.string :origin_cnpj, limit: 14, null: false
        t.string :destination_body, limit: 40, null: false
        t.string :management, limit: 1, null: false
        t.references :updated_by_user, type: :uuid, null: false, foreign_key: { to_table: :users }, index: false
        t.timestamps
      end
      add_index :billing_settings, :singleton, unique: true, name: "idx_billing_settings_singleton"
      add_check_constraint :billing_settings, "singleton", name: "ck_billing_settings_singleton"
      add_check_constraint :billing_settings, "origin_cnpj::text ~ '^[0-9]{14}$'::text AND management::text = ANY (ARRAY['M'::text, 'E'::text])",
                           name: "ck_billing_settings_values"

      create_table :billing_periods, id: :uuid do |t|
        t.references :health_unit, type: :uuid, foreign_key: true, index: false
        t.references :exam_provider, type: :uuid, foreign_key: true, index: false
        t.string :competence, limit: 6, null: false
        t.string :status, null: false, default: "open"
        t.integer :version, null: false, default: 0
        t.datetime :closed_at
        t.references :closed_by_user, type: :uuid, foreign_key: { to_table: :users }, index: false
        t.text :reopened_reason
        t.datetime :reopened_at
        t.references :reopened_by_user, type: :uuid, foreign_key: { to_table: :users }, index: false
        t.timestamps
      end
      add_index :billing_periods, %i[health_unit_id competence], unique: true, where: "health_unit_id IS NOT NULL", name: "idx_billing_periods_unit"
      add_index :billing_periods, %i[exam_provider_id competence], unique: true, where: "exam_provider_id IS NOT NULL", name: "idx_billing_periods_lab"
      add_index :billing_periods, :competence, name: "idx_billing_periods_competence"
      add_check_constraint :billing_periods, "num_nonnulls(health_unit_id, exam_provider_id) = 1", name: "ck_billing_periods_establishment"
      add_check_constraint :billing_periods, "competence::text ~ #{COMPETENCE}", name: "ck_billing_periods_competence"
      add_check_constraint :billing_periods, "status::text = ANY (ARRAY['open'::text, 'closed'::text]) AND version >= 0", name: "ck_billing_periods_status"
      add_check_constraint :billing_periods, "status::text <> 'closed'::text OR (closed_at IS NOT NULL AND closed_by_user_id IS NOT NULL AND version >= 1)",
                           name: "ck_billing_periods_closed"

      create_table :billing_exports, id: :uuid do |t|
        t.references :billing_period, type: :uuid, null: false, foreign_key: true, index: false
        t.integer :version, null: false
        t.string :cnes, limit: 7, null: false
        t.string :competence, limit: 6, null: false
        t.text :file, null: false
        t.string :sha256, limit: 64, null: false
        t.integer :lines_bpa_c, null: false
        t.integer :lines_bpa_i, null: false
        t.integer :sheets, null: false
        t.integer :control_field, null: false
        t.references :generated_by_user, type: :uuid, null: false, foreign_key: { to_table: :users }, index: false
        t.datetime :generated_at, null: false
        t.datetime :created_at, null: false
      end
      add_index :billing_exports, %i[billing_period_id version], unique: true, name: "idx_billing_exports_version"
      add_check_constraint :billing_exports, "version >= 1 AND lines_bpa_c >= 0 AND lines_bpa_i >= 0 AND (lines_bpa_c + lines_bpa_i) >= 1 " \
                                             "AND sheets >= 1 AND sheets <= 999 AND control_field >= 1111 AND control_field <= 2221",
                           name: "ck_billing_exports_numbers"
      add_check_constraint :billing_exports, "cnes::text ~ '^[0-9]{7}$'::text AND competence::text ~ #{COMPETENCE} AND sha256::text ~ '^[0-9a-f]{64}$'::text",
                           name: "ck_billing_exports_format"

      create_table :performed_procedures, id: :uuid do |t|
        t.references :patient, type: :uuid, null: false, foreign_key: true, index: false
        t.references :health_unit, type: :uuid, foreign_key: true, index: false
        t.references :exam_provider, type: :uuid, foreign_key: true, index: false
        t.references :professional_user, type: :uuid, null: false, foreign_key: { to_table: :users }, index: true
        t.string :cbo_code, limit: 6
        t.date :performed_on, null: false
        t.string :sigtap_code, limit: 10, null: false
        t.string :sigtap_competence, limit: 6
        t.boolean :sigtap_fallback, null: false, default: false
        t.integer :quantity, null: false
        t.string :cid10, limit: 4
        t.string :character, null: false, default: "elective"
        t.string :authorization_number, limit: 13
        t.string :source, null: false
        t.references :attendance, type: :uuid, foreign_key: true, index: true
        t.references :consultation, type: :uuid, foreign_key: true, index: true
        t.references :lab_report_item, type: :uuid, foreign_key: true,
                                       index: { unique: true, where: "lab_report_item_id IS NOT NULL", name: "idx_performed_procedures_lab_item" }
        t.references :recorded_by_user, type: :uuid, foreign_key: { to_table: :users }, index: false
        t.string :instrument, null: false
        t.boolean :billable, null: false
        t.string :blocking_reasons, array: true, null: false, default: []
        t.string :status, null: false, default: "recorded"
        t.text :cancel_reason
        t.datetime :cancelled_at
        t.references :cancelled_by_user, type: :uuid, foreign_key: { to_table: :users }, index: false
        t.references :billing_period, type: :uuid, foreign_key: true, index: false
        t.string :billing_competence, limit: 6
        t.boolean :late, null: false, default: false
        t.references :billing_export, type: :uuid, foreign_key: true, index: true
        t.timestamps
      end
      add_index :performed_procedures, %i[patient_id created_at], name: "idx_performed_procedures_patient"
      add_index :performed_procedures, %i[billing_period_id status], name: "idx_performed_procedures_period"
      add_check_constraint :performed_procedures, "num_nonnulls(health_unit_id, exam_provider_id) = 1", name: "ck_performed_procedures_establishment"
      add_check_constraint :performed_procedures, "quantity >= 1 AND quantity <= 999", name: "ck_performed_procedures_quantity"
      add_check_constraint :performed_procedures, "sigtap_code::text ~ '^[0-9]{10}$'::text " \
                                                  "AND (cid10 IS NULL OR cid10::text ~ '^[A-Z][0-9]{2}[0-9X]?$'::text) " \
                                                  "AND (cbo_code IS NULL OR cbo_code::text ~ '^[0-9]{6}$'::text)",
                           name: "ck_performed_procedures_codes"
      add_check_constraint :performed_procedures, "source::text = ANY (ARRAY[#{quoted(SOURCES)}]::text[]) " \
                                                  "AND instrument::text = ANY (ARRAY[#{quoted(INSTRUMENTS)}]::text[]) " \
                                                  "AND character::text = ANY (ARRAY[#{quoted(CHARACTERS)}]::text[]) " \
                                                  "AND status::text = ANY (ARRAY['recorded'::text, 'cancelled'::text])",
                           name: "ck_performed_procedures_domains"
      add_check_constraint :performed_procedures, "(status::text = 'cancelled'::text) = (cancelled_at IS NOT NULL AND cancel_reason IS NOT NULL)",
                           name: "ck_performed_procedures_cancelled"
      add_check_constraint :performed_procedures, "(source::text <> 'attendance'::text OR attendance_id IS NOT NULL) " \
                                                  "AND (source::text <> 'consultation'::text OR consultation_id IS NOT NULL) " \
                                                  "AND (source::text <> 'lab_report'::text OR (lab_report_item_id IS NOT NULL AND exam_provider_id IS NOT NULL))",
                           name: "ck_performed_procedures_source_ref"
      add_check_constraint :performed_procedures, "NOT billable OR (cardinality(blocking_reasons) = 0 AND instrument::text <> 'not_bpa'::text)",
                           name: "ck_performed_procedures_billable"
      add_check_constraint :performed_procedures, "((billing_period_id IS NULL) = (billing_competence IS NULL)) " \
                                                  "AND (billing_competence IS NULL OR billing_competence::text ~ #{COMPETENCE}) " \
                                                  "AND (sigtap_competence IS NULL OR sigtap_competence::text ~ #{COMPETENCE}) " \
                                                  "AND (billing_export_id IS NULL OR billing_period_id IS NOT NULL)",
                           name: "ck_performed_procedures_competences"

      # D-U1 (contrato §12): raça/cor, etnia e endereço do paciente para o BPA-I,
      # cifrados com a chave da cidade (texto; a validação é do Patients::Demographics).
      %i[race_color ethnicity address_zip address_street address_number address_complement address_district].each do |column|
        add_column :patients, column, :text
      end

      execute File.read(Rails.root.join("db/city_triggers.sql"))
    end

    def down
      raise ActiveRecord::IrreversibleMigration
    end

    private

    def quoted(list) = list.map { |value| "'#{value}'::text" }.join(", ")

    def replace_check(table, name, expression)
      remove_check_constraint table, name: name
      add_check_constraint table, expression, name: name
    end
  end
  ```

- [ ] **Step 4: Triggers** — acrescente ao **fim** de `db/city_triggers.sql`:

  ```sql
  -- Módulo 22 (ADR 0039): o procedimento realizado só anda na crítica
  -- (recalculada no fechamento), no cancelamento (uma vez) e no arquivo que o
  -- incluiu; o CBO só nasce de NULL; a re-cifra regrava só o motivo.
  CREATE OR REPLACE FUNCTION rota_performed_procedure_guard() RETURNS trigger AS $fn$
  DECLARE
    moving text[] := ARRAY['instrument', 'billable', 'blocking_reasons', 'sigtap_competence', 'sigtap_fallback', 'cbo_code',
                           'status', 'cancel_reason', 'cancelled_at', 'cancelled_by_user_id', 'billing_export_id', 'updated_at'];
  BEGIN
    IF TG_OP = 'DELETE' THEN
      RAISE EXCEPTION 'performed_procedures is append-only: DELETE refused';
    END IF;
    IF current_setting('rota.reencrypting', true) = 'on' AND (to_jsonb(NEW) - 'cancel_reason') = (to_jsonb(OLD) - 'cancel_reason') THEN
      RETURN NEW;
    END IF;
    IF (to_jsonb(NEW) - moving) IS DISTINCT FROM (to_jsonb(OLD) - moving) THEN
      RAISE EXCEPTION 'performed_procedures: clinical content is immutable';
    END IF;
    IF OLD.cbo_code IS NOT NULL AND NEW.cbo_code IS DISTINCT FROM OLD.cbo_code THEN
      RAISE EXCEPTION 'performed_procedures: cbo_code only fills once';
    END IF;
    IF OLD.status = 'cancelled' AND ROW(NEW.status, NEW.cancel_reason, NEW.cancelled_at, NEW.cancelled_by_user_id)
         IS DISTINCT FROM ROW(OLD.status, OLD.cancel_reason, OLD.cancelled_at, OLD.cancelled_by_user_id) THEN
      RAISE EXCEPTION 'performed_procedures: a cancellation is final';
    END IF;
    RETURN NEW;
  END;
  $fn$ LANGUAGE plpgsql;

  -- Competência: estabelecimento e competência imutáveis; a versão só cresce.
  CREATE OR REPLACE FUNCTION rota_billing_period_guard() RETURNS trigger AS $fn$
  DECLARE
    moving text[] := ARRAY['status', 'version', 'closed_at', 'closed_by_user_id', 'reopened_reason', 'reopened_at',
                           'reopened_by_user_id', 'updated_at'];
  BEGIN
    IF TG_OP = 'DELETE' THEN
      RAISE EXCEPTION 'billing_periods is append-only: DELETE refused';
    END IF;
    IF (to_jsonb(NEW) - moving) IS DISTINCT FROM (to_jsonb(OLD) - moving) THEN
      RAISE EXCEPTION 'billing_periods: establishment and competence are immutable';
    END IF;
    IF NEW.version < OLD.version THEN
      RAISE EXCEPTION 'billing_periods: version only grows';
    END IF;
    RETURN NEW;
  END;
  $fn$ LANGUAGE plpgsql;

  DO $do$
  DECLARE
    item text;
  BEGIN
    IF to_regclass('public.performed_procedures') IS NOT NULL THEN
      EXECUTE 'DROP TRIGGER IF EXISTS performed_procedures_guard ON performed_procedures';
      EXECUTE 'CREATE TRIGGER performed_procedures_guard BEFORE UPDATE OR DELETE ON performed_procedures
        FOR EACH ROW EXECUTE FUNCTION rota_performed_procedure_guard()';
      EXECUTE 'DROP TRIGGER IF EXISTS billing_periods_guard ON billing_periods';
      EXECUTE 'CREATE TRIGGER billing_periods_guard BEFORE UPDATE OR DELETE ON billing_periods
        FOR EACH ROW EXECUTE FUNCTION rota_billing_period_guard()';
      EXECUTE 'DROP TRIGGER IF EXISTS billing_exports_reencryption_only ON billing_exports';
      EXECUTE 'CREATE TRIGGER billing_exports_reencryption_only BEFORE UPDATE OR DELETE ON billing_exports
        FOR EACH ROW EXECUTE FUNCTION rota_reencryption_only(''file'')';
      FOREACH item IN ARRAY ARRAY['performed_procedures', 'billing_periods', 'billing_exports'] LOOP
        EXECUTE format('DROP TRIGGER IF EXISTS %I ON %I', item || '_append_only_truncate', item);
        EXECUTE format('CREATE TRIGGER %I BEFORE TRUNCATE ON %I FOR EACH STATEMENT EXECUTE FUNCTION rota_append_only()',
                       item || '_append_only_truncate', item);
      END LOOP;
    END IF;
  END
  $do$;
  ```
  (O cancelamento de `recorded` para `cancelled` passa pelo `moving`; a volta é recusada. `billing_settings` é editável e não tem guarda.)

  No **mesmo arquivo**, em `rota_patient_guard()` (a função dos `patients`, ADR 0031), troque a lista das colunas cifradas — a re-cifra precisa poder regravar as novas:

  ```sql
    encrypted text[] := ARRAY['cpf', 'full_name', 'social_name', 'mother_name', 'birth_date', 'sex', 'race_color', 'ethnicity',
                              'address_zip', 'address_street', 'address_number', 'address_complement', 'address_district'];
  ```

- [ ] **Step 5: Modelos.** `app/models/performed_procedure.rb`:

  ```ruby
  # Procedimento realizado (ADR 0039; spec §4): só acréscimo no conteúdo
  # clínico; a crítica (instrumento, faturável, motivos) é recalculada no
  # fechamento e o estado só vai de recorded a cancelled (trigger). Exatamente
  # um estabelecimento: unidade de saúde ou laboratório municipal. O motivo do
  # cancelamento é cifrado com a chave da cidade e nunca aparece no inspect.
  class PerformedProcedure < ApplicationRecord
    SOURCES = %w[attendance consultation lab_report manual].freeze
    INSTRUMENTS = %w[bpa_i bpa_c not_bpa].freeze
    CHARACTERS = %w[elective urgent].freeze
    STATUSES = %w[recorded cancelled].freeze

    encrypts :cancel_reason

    belongs_to :patient
    belongs_to :health_unit, optional: true
    belongs_to :exam_provider, optional: true
    belongs_to :professional_user, class_name: "User"
    belongs_to :recorded_by_user, class_name: "User", optional: true
    belongs_to :cancelled_by_user, class_name: "User", optional: true
    belongs_to :attendance, optional: true
    belongs_to :consultation, optional: true
    belongs_to :lab_report_item, optional: true
    belongs_to :billing_period, optional: true
    belongs_to :billing_export, optional: true

    scope :recorded, -> { where(status: "recorded") }

    def inspect = "#<PerformedProcedure id=#{id} source=#{source} status=#{status} billable=#{billable}>"
  end
  ```

  `app/models/billing_period.rb`:

  ```ruby
  # Competência de um estabelecimento faturante (ADR 0039; spec §4): nasce
  # aberta com a primeira produção; fechar gera a versão seguinte do arquivo;
  # reabrir exige motivo (cifrado). Estabelecimento e competência imutáveis.
  class BillingPeriod < ApplicationRecord
    STATUSES = %w[open closed].freeze

    encrypts :reopened_reason

    belongs_to :health_unit, optional: true
    belongs_to :exam_provider, optional: true
    belongs_to :closed_by_user, class_name: "User", optional: true
    belongs_to :reopened_by_user, class_name: "User", optional: true
    has_many :procedures, class_name: "PerformedProcedure", dependent: :restrict_with_error
    has_many :exports, class_name: "BillingExport", dependent: :restrict_with_error

    def closed? = status == "closed"
    def inspect = "#<BillingPeriod id=#{id} competence=#{competence} status=#{status} version=#{version}>"
  end
  ```

  `app/models/billing_export.rb`:

  ```ruby
  # Arquivo de interface do BPA Magnético gerado no fechamento (ADR 0039;
  # contracts/billing/bpa-interface.md): só acréscimo, cifrado com a chave da
  # cidade (tem nome, CPF, nascimento do paciente). Nunca no inspect nem em log.
  class BillingExport < ApplicationRecord
    encrypts :file

    belongs_to :billing_period
    belongs_to :generated_by_user, class_name: "User"

    def bytes = file.to_s.b
    def inspect = "#<BillingExport id=#{id} version=#{version} sha256=#{sha256}>"
  end
  ```

  Em `app/models/patient.rb`, depois de `encrypts :sex`:

  ```ruby
  # D-U1 (ADR 0039; contrato §12): raça/cor (01..05, sem 99), etnia (só
  # indígena) e endereço do paciente para o BPA-I. Escritos pela recepção e
  # pelo faturista (Patients::Demographics); apagados na exclusão do cadastro.
  RACE_COLORS = %w[01 02 03 04 05].freeze
  DEMOGRAPHICS = %w[race_color ethnicity address_zip address_street address_number address_complement address_district].freeze
  encrypts :race_color, :ethnicity, :address_zip, :address_street, :address_number, :address_complement, :address_district
  ```

  `app/models/billing_setting.rb`:

  ```ruby
  # Configuração do cabeçalho do arquivo BPA da cidade (contrato §6): linha
  # única, editada pelo admin municipal com step-up (Billing::SaveSettings).
  class BillingSetting < ApplicationRecord
    belongs_to :updated_by_user, class_name: "User"

    def self.current = find_by(singleton: true)
  end
  ```

  Em `app/models/membership.rb`, troque `ROLES` e `PRIVILEGED_ROLES` e acrescente o comentário:

  ```ruby
    ROLES = %w[analyst billing_clerk campaign_manager citizen_verifier exam_provider exam_regulator health_professional lab_releaser
               lab_staff municipal_admin protocol_author protocol_publisher protocol_reviewer viewer].freeze
  ```

  ```ruby
    # lab_staff e lab_releaser (ADR 0037): veem nome e amostra do paciente; o
    # liberador assina laudo — step-up; o mantenedor não concede.
    # billing_clerk (ADR 0039): fecha competência e baixa o arquivo BPA com
    # nome, CPF e nascimento do paciente — step-up; o mantenedor não concede.
    PRIVILEGED_ROLES = %w[municipal_admin protocol_reviewer citizen_verifier health_professional
                          campaign_manager exam_regulator exam_provider lab_staff lab_releaser billing_clerk].freeze
  ```

  Em `app/services/city_encryption.rb`, no fim de `CITY_KEYED_TARGETS` (depois de `[ LabQcReviewItem, :note ]`, trocando o fim da linha por `,`):

  ```ruby
      [ PerformedProcedure, :cancel_reason ],
      [ BillingPeriod, :reopened_reason ],
      [ BillingExport, :file ],
      [ Patient, :race_color ], [ Patient, :ethnicity ], [ Patient, :address_zip ], [ Patient, :address_street ],
      [ Patient, :address_number ], [ Patient, :address_complement ], [ Patient, :address_district ]
  ```

  Em `config/initializers/domain_events.rb`, antes do `end` final do bloco `to_prepare`:

  ```ruby
    # Módulo 22 (ADR 0039): procedimentos e faturamento, só trilha, só ids (e
    # source/billable/version). Nunca CPF, CNS, nome, motivo ou arquivo.
    %w[performed_procedure.recorded performed_procedure.cancelled billing_period.closed billing_period.reopened
       billing_period.pending_viewed billing_export.generated billing_export.downloaded billing_settings.changed
       billing_patient.looked_up].each { |name| DomainEvents.bind name, to: [] }
  ```

  E no fim de `spec/initializers/domain_events_bindings_spec.rb`:

  ```ruby
  # Módulo 22 (ADR 0039): eventos do faturamento declarados, só trilha.
  RSpec.describe "billing event bindings (ADR 0039)" do
    it "declares every billing event with no consumer" do
      names = %w[performed_procedure.recorded performed_procedure.cancelled billing_period.closed billing_period.reopened
                 billing_period.pending_viewed billing_export.generated billing_export.downloaded billing_settings.changed
                 billing_patient.looked_up]
      expect(DomainEvents.registry.keys).to include(*names)
      expect(names.flat_map { |n| DomainEvents.registry[n] }).to be_empty
    end
  end
  ```

- [ ] **Step 6: Helpers** `spec/support/billing_helpers.rb`:

  ```ruby
  # Módulo 22 (ADR 0039): SIGTAP de faturamento (instrumento, CBO, CID), a
  # cidade com procedure_billing, UPA com CNES, faturista, configuração e
  # procedimento cru (para as specs de banco). A SIGTAP é imutável depois de
  # ativa: billing_sigtap! roda ANTES de exam_city!/lab_city! (o
  # sigtap_release! deles reaproveita a competência que já existe).
  module BillingHelpers
    # código => [nome, sexo, idade mínima, idade máxima (meses)]; atributos de teste.
    PROCEDURES = {
      "0301060029" => [ "ATENDIMENTO DE URGENCIA C/ OBSERVACAO ATE 24 HORAS EM ATENCAO ESPECIALIZADA", nil, nil, nil ],
      "0301060096" => [ "ATENDIMENTO MEDICO EM UNIDADE DE PRONTO ATENDIMENTO", nil, nil, nil ],
      "0301010072" => [ "CONSULTA MEDICA EM ATENCAO ESPECIALIZADA", nil, nil, nil ],
      "0301100012" => [ "ADMINISTRACAO DE MEDICAMENTOS NA ATENCAO ESPECIALIZADA.", nil, nil, nil ],
      "0301060118" => [ "ACOLHIMENTO COM CLASSIFICACAO DE RISCO", nil, nil, nil ],
      "0214010015" => [ "GLICEMIA CAPILAR", nil, nil, nil ],
      "0201020033" => [ "COLETA DE MATERIAL P/ EXAME CITOPATOLOGICO DE COLO UTERINO", "F", 300, 780 ],
      "0604320124" => [ "MEDICAMENTO DE TESTE REGISTRADO EM APAC", nil, nil, nil ]
    }.freeze
    INSTRUMENTS = {
      "0301060029" => %w[02], "0301060096" => %w[02], "0301010072" => %w[02], "0301010064" => %w[02], "0301100012" => %w[01],
      "0301060118" => %w[01], "0214010015" => %w[01], "0201020033" => %w[01 02], "0604320124" => %w[06],
      "0202010473" => %w[01], "0202020380" => %w[01], "0202010503" => %w[01], "0202010317" => %w[01]
    }.freeze
    CBOS = {
      "0301060029" => %w[225125 225124], "0301060096" => %w[225125 225124 225142], "0301010072" => %w[225125],
      "0301100012" => %w[223505 322205], "0301060118" => %w[223505 225125], "0214010015" => %w[223505 322205],
      "0201020033" => %w[223505 225125], "0202010473" => %w[221205], "0202020380" => %w[221205], "0202010503" => %w[221205],
      "0202010317" => %w[221205]
    }.freeze
    CIDS = { "0301060029" => %w[I10 E119] }.freeze
    REGISTRIES = { "01" => "BPA (CONSOLIDADO)", "02" => "BPA (INDIVIDUALIZADO)", "06" => "APAC (PROCEDIMENTO PRINCIPAL)" }.freeze

    def billing_sigtap!(competences = [ Time.zone.today.strftime("%Y%m"), Time.zone.today.prev_month.strftime("%Y%m") ])
      Array(competences).map do |competence|
        existing = TerminologyRelease.active.find_by(kind: "sigtap", version: competence)
        if existing
          raise "billing_sigtap! depois de exam_city!/lab_city!: chame antes" unless SigtapProcedureInstrument.exists?(release_id: existing.id)

          next existing
        end
        release = TerminologyRelease.create!(kind: "sigtap", version: competence, source_sha256: "b" * 64, imported_by: "rspec",
                                             imported_at: Time.current, status: "importing")
        ClinicalRecordHelpers::SIGTAP.transform_values { |name| [ name, nil, nil, nil ] }.merge(PROCEDURES).each do |code, (name, sex, min, max)|
          SigtapProcedure.create!(release: release, code: code, name: name, sex: sex, age_min_months: min, age_max_months: max)
        end
        INSTRUMENTS.each do |code, list|
          list.each { |reg| SigtapProcedureInstrument.create!(release: release, procedure_code: code, instrument_code: reg, instrument_name: REGISTRIES.fetch(reg)) }
        end
        CBOS.each { |code, list| list.each { |cbo| SigtapProcedureCbo.create!(release: release, procedure_code: code, cbo_code: cbo) } }
        CIDS.each { |code, list| list.each { |cid| SigtapProcedureCid.create!(release: release, procedure_code: code, cid_code: cid, principal: true) } }
        release.update!(status: "active", activated_at: Time.current)
        release
      end
    end

    # Cidade com procedure_billing ligado, CID-10, perfil com IBGE (4106902) e,
    # com lab: true, o laboratório municipal do 21b.
    def billing_city!(lab: false, record_mode: "record")
      billing_sigtap!
      city = lab ? lab_city! : clinical_city!(record_mode: record_mode)
      cid10_release!
      Platform::Features.set!(city: city, key: "procedure_billing", enabled: true, maintainer: ledi_maintainer!)
      city_profile!
      CityCatalog.reset_cache!
      city
    end

    def billing_cnes = format("%07d", SecureRandom.random_number(9_000_000) + 1_000_000)
    def upa!(name = "UPA #{SecureRandom.hex(2)}", cnes: billing_cnes) = HealthUnit.create!(name: name, kind: "upa", cnes: cnes)
    def billing_clerk! = staff_with("faturista-#{SecureRandom.hex(3)}@cidade.gov.br", "billing_clerk")
    # D-U1: o paciente de teste nasce com raça/cor e endereço (faturável no BPA-I); demographics: false para a pendência.
    DEMOGRAPHICS = { race_color: "03", ethnicity: nil, address_zip: "80010000", address_street: "Rua XV de Novembro",
                     address_number: "1200", address_complement: nil, address_district: "Centro" }.freeze

    def billing_patient!(demographics: true, **citizen_attrs)
      patient = exam_patient!(exam_seq!, **citizen_attrs).first
      patient.update!(DEMOGRAPHICS) if demographics
      patient
    end

    def billing_settings!(by: lab_admin!)
      BillingSetting.current ||
        BillingSetting.create!(origin_name: "SMS Curitiba", origin_cnpj: "11222333000181", destination_body: "Secretaria Municipal de Saude",
                               management: "M", updated_by_user: by)
    end

    # Procedimento gravado direto (specs de banco); o caminho real é Billing::Record (Task 5).
    def raw_procedure!(unit:, patient:, professional:, **over)
      PerformedProcedure.create!({ patient: patient, health_unit: unit, professional_user: professional, cbo_code: "225125",
                                   performed_on: Time.zone.today, sigtap_code: "0301060029", quantity: 1, cid10: "I10",
                                   source: "manual", recorded_by_user: professional, instrument: "bpa_i", billable: true,
                                   blocking_reasons: [] }.merge(over))
    end
  end

  RSpec.configure { |c| c.include BillingHelpers }
  ```

- [ ] **Step 7: Rode a migração e escreva o dump.** Recrie os bancos de teste `_mod22` ("Ambiente") e migre o de dev do worktree **só pelo rake**: `rails city:migrate:all`. **Escreva à mão** `db/city_schema.rb`: `define(version: 2026_10_21_900001)`; o `ck_memberships_role` novo (com `billing_clerk`); os blocos `create_table` novos em ordem alfabética (`billing_exports`, `billing_periods`, `billing_settings` antes de `cbo…`/`citizen…`; `performed_procedures` depois de `pec_signature_uses`/antes de `platform…`, onde a ordem alfabética mandar), com colunas em ordem alfabética, índices e checks como o `pg_dump` os devolve — compare com

  ```bash
  docker compose exec -T -e RAILS_ENV=test -e ROTA_TEST_DB_SUFFIX=_mod22 -w /rails/.claude/mod22 api bin/rails runner 'CityConnection.with(TEST_CITY_A) { puts ActiveRecord::SchemaDumper.dump }' > /tmp/city_dump_22.rb
  ```
  (se `TEST_CITY_A` não existir fora do rspec, use o `CitySchema.db_config_for` do banco `rota_saude_test_city_a_mod22`) — e as `add_foreign_key` novas em ordem.

- [ ] **Step 8: Rode:** `rspec spec/models/billing_schema_spec.rb spec/models/membership_roles_spec.rb spec/services/city_schema_spec.rb spec/architecture spec/initializers/domain_events_bindings_spec.rb spec/models/membership_spec.rb` → verde.

- [ ] **Step 9: Commit:**

  ```bash
  git add db/city_migrate/20261021900001_add_procedure_billing.rb db/city_schema.rb db/city_triggers.sql app/models/patient.rb app/models/performed_procedure.rb app/models/billing_period.rb app/models/billing_export.rb app/models/billing_setting.rb app/models/membership.rb app/services/city_encryption.rb config/initializers/domain_events.rb spec/initializers/domain_events_bindings_spec.rb spec/support/billing_helpers.rb spec/models/billing_schema_spec.rb spec/models/membership_roles_spec.rb
  git commit -m "feat: add procedure billing city tables, billing clerk role and immutability triggers

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

---

## Fatia 2 — Crítica, gravação e rotas de procedimento

### Task 4: Crítica SIGTAP (`Billing::Critic`), estabelecimento e dados do profissional

**Files:**
- Create: `app/services/billing/establishment.rb`, `app/services/billing/staff.rb`, `app/services/billing/critic.rb`, `spec/services/billing/critic_spec.rb`

**Interfaces:**
- Consumes: `Terminology::Sigtap.release_for(competence)`; `SigtapProcedure`, `SigtapProcedureCbo`, `SigtapProcedureCid`, `SigtapProcedureInstrument` (plataforma); `billing_sigtap!` (Task 3).
- Produces:
  - `Billing::Establishment = Data.define(:type, :id, :cnes, :name, :kind)` com `#aps? -> Boolean`, `#json -> { type:, id:, cnes:, name: }`, `#key -> { health_unit_id: id } | { exam_provider_id: id }`, `.for_unit(HealthUnit)`, `.for_lab(ExamProvider)`, `.find(type, id) -> Establishment | nil` (unidade ativa ou laboratório municipal ativo), `.of(record_com_health_unit_ou_exam_provider)`.
  - `Billing::Staff.cbo_for(user, establishment) -> String | nil`, `.linked?(user, unit_id) -> Boolean`, `.cns(user) -> String | nil`, `.cbos_for(user, establishment) -> [String]`, `.eligible?(user, establishment) -> Boolean`, `.non_aps_units(user) -> Relation`.
  - `Billing::Critic::REASONS` (ordem estável), `Billing::Critic::SEX = { "female" => "F", "male" => "M" }`, `Billing::Critic::Facts = Data.define(:establishment, :sigtap_code, :performed_on, :cbo_code, :professional_cns, :cid10, :patient)`, `Billing::Critic::Outcome = Data.define(:instrument, :billable, :reasons, :sigtap_competence, :fallback, :label)`, `Billing::Critic.call(facts, closing: false) -> Outcome` (`closing: true` acrescenta `missing_race_color` e `missing_address` no BPA-I — D-U1).

- [ ] **Step 1: Teste que falha** — `spec/services/billing/critic_spec.rb`:

  ```ruby
  require "rails_helper"

  # ADR 0039 (spec §2 decisões 2, 4 e 6; spec §9): cada crítica SIGTAP com caso
  # que passa e que falha; marca e não bloqueia; UBS nunca gera BPA; CID e
  # identificação do paciente só no BPA-I; a SIGTAP da competência ou a última
  # ativa anterior (fallback); motivos na ordem estável do contrato.
  RSpec.describe Billing::Critic do
    before { billing_sigtap! }

    let(:today) { Time.zone.today }
    let(:upa) { Billing::Establishment.new(type: "health_unit", id: SecureRandom.uuid, cnes: "1234567", name: "UPA", kind: "upa") }
    let(:lab) { Billing::Establishment.new(type: "lab_provider", id: SecureRandom.uuid, cnes: "7654321", name: "Lab", kind: "lab") }
    let(:patient) { Patient.new(cpf: "29136237191", full_name: "Maria Aparecida da Silva", birth_date: (today - 40.years).iso8601, sex: "female") }

    def check(**over)
      described_class.call(Billing::Critic::Facts.new(**{ establishment: upa, sigtap_code: "0301060029", performed_on: today, cbo_code: "225125",
                                                          professional_cns: "798312812760829", cid10: "I10", patient: patient }.merge(over)))
    end

    it "faturável: BPA-I na UPA com CBO, CNS, CID e paciente completos" do
      expect(check).to have_attributes(instrument: "bpa_i", billable: true, reasons: [], sigtap_competence: today.strftime("%Y%m"),
                                       fallback: false, label: "ATENDIMENTO DE URGENCIA C/ OBSERVACAO ATE 24 HORAS EM ATENCAO ESPECIALIZADA")
    end

    it "UBS nunca gera BPA: o mesmo procedimento na unidade de atenção primária não é faturável" do
      expect(check(establishment: upa.with(kind: "ubs"))).to have_attributes(instrument: "bpa_i", billable: false, reasons: %w[establishment_is_aps])
      expect(upa.with(kind: "hospital").aps?).to be(false)
      expect(upa.with(kind: "other").aps?).to be(false)
      expect(lab.aps?).to be(false)
    end

    it "laboratório municipal: BPA-C com o CBO do liberador, sem CNS nem CID" do
      expect(check(establishment: lab, sigtap_code: "0202010473", cbo_code: "221205", professional_cns: nil, cid10: nil))
        .to have_attributes(instrument: "bpa_c", billable: true, reasons: [])
    end

    it "estabelecimento sem CNES" do
      expect(check(establishment: upa.with(cnes: nil)).reasons).to eq(%w[missing_establishment_cnes])
    end

    it "fora da SIGTAP da competência e instrumento que não é BPA" do
      expect(check(sigtap_code: "0399999999")).to have_attributes(instrument: "not_bpa", billable: false, reasons: %w[not_in_sigtap], label: nil)
      expect(check(sigtap_code: "0604320124", cid10: nil)).to have_attributes(instrument: "not_bpa", billable: false, reasons: %w[not_bpa_instrument])
    end

    it "BPA-I e BPA-C listados: vale o BPA-I" do
      expect(check(sigtap_code: "0201020033", cbo_code: "223505", cid10: nil)).to have_attributes(instrument: "bpa_i", billable: true)
    end

    it "CBO: ausente, não permitido e procedimento sem lista de CBO" do
      expect(check(cbo_code: nil).reasons).to eq(%w[missing_professional_cbo])
      expect(check(cbo_code: "322205").reasons).to eq(%w[cbo_not_allowed])
      expect(check(sigtap_code: "0301010064", cbo_code: "322205", cid10: nil)).to have_attributes(billable: true)
    end

    it "CNS do profissional só no BPA-I" do
      expect(check(professional_cns: nil).reasons).to eq(%w[missing_professional_cns])
      expect(check(sigtap_code: "0301100012", cbo_code: "322205", professional_cns: nil, cid10: nil).billable).to be(true)
    end

    it "idade em meses: limite inclusivo nas duas pontas" do
      young = patient.dup.tap { |p| p.birth_date = (today - 25.years).iso8601 }
      younger = patient.dup.tap { |p| p.birth_date = (today - 25.years + 1.day).iso8601 }
      old = patient.dup.tap { |p| p.birth_date = (today - 65.years).iso8601 }
      older = patient.dup.tap { |p| p.birth_date = (today - 65.years - 1.month).iso8601 }
      base = { sigtap_code: "0201020033", cbo_code: "223505", cid10: nil }
      expect(check(**base, patient: young).billable).to be(true)
      expect(check(**base, patient: younger).reasons).to eq(%w[age_out_of_range])
      expect(check(**base, patient: old).billable).to be(true)
      expect(check(**base, patient: older).reasons).to eq(%w[age_out_of_range])
    end

    it "sexo do procedimento" do
      male = patient.dup.tap { |p| p.sex = "male" }
      expect(check(sigtap_code: "0201020033", cbo_code: "223505", cid10: nil, patient: male).reasons).to eq(%w[sex_not_allowed])
    end

    it "CID: exigido, não permitido, permitido e livre quando a tabela não lista CID" do
      expect(check(cid10: nil).reasons).to eq(%w[cid_required])
      expect(check(cid10: "N390").reasons).to eq(%w[cid_not_allowed])
      expect(check(cid10: "E119").billable).to be(true)
      expect(check(sigtap_code: "0301010072", cid10: "N390").billable).to be(true)
    end

    it "CID não conta no BPA-C (o arquivo não tem a coluna)" do
      expect(check(establishment: lab, sigtap_code: "0202010473", cbo_code: "221205", cid10: "N390").billable).to be(true)
    end

    it "paciente: identificação e campos obrigatórios no BPA-I; no BPA-C só o nascimento (idade)" do
      expect(check(patient: patient.dup.tap { |p| p.cpf = "" }).reasons).to eq(%w[missing_patient_id])
      expect(check(patient: patient.dup.tap { |p| p.full_name = nil }).reasons).to eq(%w[missing_required_patient_field])
      expect(check(patient: patient.dup.tap { |p| p.sex = nil }).reasons).to eq(%w[missing_required_patient_field])
      no_birth = patient.dup.tap { |p| p.birth_date = nil }
      expect(check(patient: no_birth).reasons).to eq(%w[missing_required_patient_field])
      bpa_c = { establishment: lab, sigtap_code: "0202010473", cbo_code: "221205", professional_cns: nil, cid10: nil }
      expect(check(**bpa_c, patient: patient.dup.tap { |p| p.full_name = nil }).billable).to be(true)
      expect(check(**bpa_c, patient: no_birth).reasons).to eq(%w[missing_required_patient_field])
    end

    it "D-U1: raça/cor e endereço só contam no fechamento (closing: true), só no BPA-I" do
      expect(check.billable).to be(true)
      closing = ->(person, **over) { described_class.call(Billing::Critic::Facts.new(**{ establishment: upa, sigtap_code: "0301060029", performed_on: today,
                                                                                         cbo_code: "225125", professional_cns: "798312812760829", cid10: "I10",
                                                                                         patient: person }.merge(over)), closing: true) }
      expect(closing.call(patient).reasons).to eq(%w[missing_race_color missing_address])
      complete = patient.dup.tap { |p| p.assign_attributes(race_color: "03", address_zip: "80010000", address_street: "Rua XV", address_number: "SN", address_district: "Centro") }
      expect(closing.call(complete).billable).to be(true)
      expect(closing.call(complete.dup.tap { |p| p.race_color = "99" }).reasons).to eq(%w[missing_race_color])
      expect(closing.call(complete.dup.tap { |p| p.race_color = "05" }).reasons).to eq(%w[missing_race_color])
      expect(closing.call(complete.dup.tap { |p| p.assign_attributes(race_color: "05", ethnicity: "0001") }).billable).to be(true)
      expect(closing.call(complete.dup.tap { |p| p.address_number = nil }).reasons).to eq(%w[missing_address])
      expect(closing.call(patient, establishment: lab, sigtap_code: "0202010473", cbo_code: "221205", professional_cns: nil, cid10: nil).billable).to be(true)
    end

    it "vários motivos de uma vez, na ordem estável do contrato" do
      male = patient.dup.tap { |p| p.sex = "male" }
      outcome = check(establishment: upa.with(kind: "ubs", cnes: nil), sigtap_code: "0201020033", cbo_code: "322205", professional_cns: nil,
                      cid10: nil, patient: male)
      expect(outcome.reasons).to eq(%w[establishment_is_aps missing_establishment_cnes cbo_not_allowed missing_professional_cns sex_not_allowed])
      expect(described_class::REASONS.first(3)).to eq(%w[establishment_is_aps missing_establishment_cnes not_in_sigtap])
    end

    it "sem a release da competência vale a última ativa anterior, com aviso; sem nenhuma, fora da SIGTAP" do
      expect(check(performed_on: today.next_month)).to have_attributes(billable: true, sigtap_competence: today.strftime("%Y%m"), fallback: true)
      expect(check(performed_on: Date.new(1999, 1, 1))).to have_attributes(reasons: %w[not_in_sigtap], sigtap_competence: nil, fallback: false)
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar:** `rspec spec/services/billing/critic_spec.rb` → `NameError: uninitialized constant Billing::Establishment`.

- [ ] **Step 3: Implemente.** `app/services/billing/establishment.rb`:

  ```ruby
  # Estabelecimento do procedimento (contrato §1): unidade de saúde ou
  # laboratório municipal (exam_providers com uses_municipal_lab). A UBS é de
  # atenção primária e nunca gera BPA (ADR 0039 Invariantes; spec §2 decisão 2).
  module Billing
    Establishment = Data.define(:type, :id, :cnes, :name, :kind) do
      def aps? = type == "health_unit" && kind == "ubs"
      def json = { type: type, id: id, cnes: cnes, name: name }
      def key = type == "health_unit" ? { health_unit_id: id } : { exam_provider_id: id }

      def self.for_unit(unit) = new(type: "health_unit", id: unit.id, cnes: unit.cnes, name: unit.name, kind: unit.kind)
      def self.for_lab(provider) = new(type: "lab_provider", id: provider.id, cnes: provider.cnes, name: provider.name, kind: "lab")

      # Do corpo da requisição: unidade ativa ou laboratório municipal ativo.
      def self.find(type, id)
        return nil unless id.is_a?(String) && id.match?(Exams::BaseController::UUID)

        case type
        when "health_unit" then HealthUnit.find_by(id: id, active: true)&.then { |unit| for_unit(unit) }
        when "lab_provider" then ExamProvider.find_by(id: id, uses_municipal_lab: true, active: true)&.then { |lab| for_lab(lab) }
        end
      end

      # De um procedimento ou de uma competência, com o cadastro de agora.
      def self.of(record) = record.health_unit_id ? for_unit(record.health_unit) : for_lab(record.exam_provider)
    end
  end
  ```

  `app/services/billing/staff.rb`:

  ```ruby
  # O profissional no faturamento (plano do módulo 22, Desvio 9): CBO pelo
  # vínculo ativo mais antigo na unidade (no laboratório, em qualquer unidade:
  # o liberador não tem vínculo com o laboratório) e CNS do perfil (módulo 10).
  module Billing
    module Staff
      module_function

      def cbo_for(user, establishment)
        return nil unless user

        links = ProfessionalLink.active.joins(:professional).where(professionals: { user_id: user.id })
        links = links.where(health_unit_id: establishment.id) if establishment.type == "health_unit"
        links.order(:started_at, :id).pick(:cbo_code)
      end

      def linked?(user, unit_id)
        ProfessionalLink.active.joins(:professional).where(professionals: { user_id: user.id }, health_unit_id: unit_id).exists?
      end

      def cns(user) = user&.professional&.cns.presence

      # CBOs dos vínculos ativos que valem no estabelecimento (B5).
      def cbos_for(user, establishment)
        links = ProfessionalLink.active.joins(:professional).where(professionals: { user_id: user.id })
        links = links.where(health_unit_id: establishment.id) if establishment.type == "health_unit"
        links.order(:started_at, :id).pluck(:cbo_code).uniq
      end

      # Pode ser o "quem fez" do lançamento manual: vínculo ativo na unidade, ou liberador ativo do laboratório (B5).
      def eligible?(user, establishment)
        return linked?(user, establishment.id) if establishment.type == "health_unit"

        LabReleaser.active.exists?(user_id: user.id, exam_provider_id: establishment.id)
      end

      # Unidades não-APS em que o usuário tem vínculo ativo (busca de paciente, B3).
      def non_aps_units(user)
        HealthUnit.where(active: true).where.not(kind: "ubs")
                  .where(id: ProfessionalLink.active.joins(:professional).where(professionals: { user_id: user.id }).select(:health_unit_id))
      end
    end
  end
  ```

  `app/services/billing/critic.rb`:

  ```ruby
  # Crítica SIGTAP do procedimento realizado (ADR 0039; spec §2 decisões 2, 4 e
  # 6; contrato §1). Marca, nunca bloqueia: devolve o instrumento, se é
  # faturável e os motivos, na ordem estável REASONS. Lê a release da
  # competência do dia do procedimento — ou a última ativa anterior (fallback,
  # com aviso). Relação vazia na SIGTAP (procedimento sem CBO/CID listado) não
  # restringe. BPA-I e BPA-C listados: BPA-I. CID e identificação do paciente
  # só contam no BPA-I (o BPA-C não tem essas colunas); o nascimento conta nos
  # dois (idade). Usada na gravação e de novo no fechamento.
  module Billing
    module Critic
      REASONS = %w[establishment_is_aps missing_establishment_cnes not_in_sigtap not_bpa_instrument missing_professional_cbo
                   cbo_not_allowed missing_professional_cns age_out_of_range sex_not_allowed cid_required cid_not_allowed
                   missing_patient_id missing_required_patient_field missing_race_color missing_address].freeze
      SEX = { "female" => "F", "male" => "M" }.freeze

      Facts = Data.define(:establishment, :sigtap_code, :performed_on, :cbo_code, :professional_cns, :cid10, :patient)
      Outcome = Data.define(:instrument, :billable, :reasons, :sigtap_competence, :fallback, :label)

      module_function

      # closing: true no fechamento e nas pendências (Billing::Recalculate): só
      # ali entram raça/cor e endereço do paciente (D-U1, contrato §12).
      def call(facts, closing: false)
        competence = facts.performed_on.strftime("%Y%m")
        release = Terminology::Sigtap.release_for(competence)
        reasons = []
        reasons << "establishment_is_aps" if facts.establishment.aps?
        reasons << "missing_establishment_cnes" if facts.establishment.cnes.blank?
        procedure = release && SigtapProcedure.find_by(release_id: release.id, code: facts.sigtap_code)
        return outcome("not_bpa", reasons + %w[not_in_sigtap], release, competence, nil) unless procedure

        instrument = instrument(release, facts.sigtap_code)
        reasons << "not_bpa_instrument" if instrument == "not_bpa"
        reasons.concat(professional_reasons(facts, release, instrument))
        reasons.concat(patient_reasons(facts, procedure, release, instrument, closing))
        outcome(instrument, reasons, release, competence, procedure.name)
      end

      def instrument(release, code)
        codes = SigtapProcedureInstrument.where(release_id: release.id, procedure_code: code).pluck(:instrument_code)
        return "bpa_i" if codes.include?("02")

        codes.include?("01") ? "bpa_c" : "not_bpa"
      end

      def professional_reasons(facts, release, instrument)
        reasons = []
        if facts.cbo_code.blank?
          reasons << "missing_professional_cbo"
        elsif !listed?(SigtapProcedureCbo, release, facts.sigtap_code, cbo_code: facts.cbo_code)
          reasons << "cbo_not_allowed"
        end
        reasons << "missing_professional_cns" if instrument == "bpa_i" && facts.professional_cns.blank?
        reasons
      end

      def patient_reasons(facts, procedure, release, instrument, closing)
        patient = facts.patient
        reasons = []
        months = age_months(patient&.birth_date, facts.performed_on)
        if months && ((procedure.age_min_months && months < procedure.age_min_months) ||
                      (procedure.age_max_months && months > procedure.age_max_months))
          reasons << "age_out_of_range"
        end
        reasons << "sex_not_allowed" if %w[F M].include?(procedure.sex) && SEX[patient&.sex] && SEX[patient.sex] != procedure.sex
        return reasons if instrument == "not_bpa"

        reasons << "missing_required_patient_field" if patient.nil? || patient.birth_date.blank?
        return reasons unless instrument == "bpa_i"

        reasons.concat(cid_reasons(facts, release))
        reasons << "missing_patient_id" if patient.nil? || patient.cpf.blank?
        reasons << "missing_required_patient_field" if patient && (patient.full_name.blank? || !SEX.key?(patient.sex))
        reasons.concat(demographic_reasons(patient)) if closing && patient
        reasons
      end

      # D-U1: raça/cor 01..05 (sem 99) e etnia de 4 dígitos quando indígena (05);
      # endereço com CEP, logradouro, número (ou SN) e bairro.
      def demographic_reasons(patient)
        reasons = []
        race = patient.race_color
        ethnic_ok = race != "05" || patient.ethnicity.to_s.match?(/\A\d{4}\z/)
        reasons << "missing_race_color" unless Patient::RACE_COLORS.include?(race) && ethnic_ok
        address = [ patient.address_zip, patient.address_street, patient.address_number, patient.address_district ]
        reasons << "missing_address" if address.any?(&:blank?)
        reasons
      end

      def cid_reasons(facts, release)
        cids = SigtapProcedureCid.where(release_id: release.id, procedure_code: facts.sigtap_code)
        return [] unless cids.exists?
        return %w[cid_required] if facts.cid10.blank?

        cids.exists?(cid_code: facts.cid10) ? [] : %w[cid_not_allowed]
      end

      # Sem relação nenhuma para o procedimento = sem restrição (como Terminology::Sigtap).
      def listed?(model, release, code, **value)
        scope = model.where(release_id: release.id, procedure_code: code)
        !scope.exists? || scope.exists?(value)
      end

      def age_months(birth_date, on)
        born = Date.iso8601(birth_date.to_s)
        months = (on.year * 12 + on.month) - (born.year * 12 + born.month)
        on.day < born.day ? months - 1 : months
      rescue Date::Error
        nil
      end

      def outcome(instrument, reasons, release, competence, label)
        ordered = reasons.uniq.sort_by { |reason| REASONS.index(reason) }
        Outcome.new(instrument: instrument, billable: ordered.empty?, reasons: ordered, sigtap_competence: release&.version,
                    fallback: release.present? && release.version != competence, label: label)
      end
      private_class_method :instrument, :professional_reasons, :patient_reasons, :demographic_reasons, :cid_reasons, :listed?, :age_months, :outcome
    end
  end
  ```

- [ ] **Step 4: Rode:** `rspec spec/services/billing/critic_spec.rb` → verde. Se o caso "idade" falhar em `younger`/`older` por um dia (fuso), confira que `today` é `Time.zone.today` dos dois lados — o spec e o serviço usam a mesma data.

- [ ] **Step 5: Commit:**

  ```bash
  git add app/services/billing/establishment.rb app/services/billing/staff.rb app/services/billing/critic.rb spec/services/billing/critic_spec.rb
  git commit -m "feat: add SIGTAP critic for performed procedures

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

---

### Task 5: Gravação (`Billing::Record`), competência com atraso e cancelamento

**Files:**
- Create: `app/services/billing/periods.rb`, `app/commands/billing/record.rb`, `app/commands/billing/cancel.rb`, `spec/commands/billing/record_spec.rb`, `spec/commands/billing/cancel_spec.rb`

**Interfaces:**
- Consumes: `Billing::Critic.call`, `Billing::Critic::Facts`, `Billing::Establishment`, `Billing::Staff` (Task 4); modelos e helpers (Task 3).
- Produces:
  - `Billing::Periods::COMPETENCE` (regex AAAAMM); `Billing::Periods.assign!(establishment, competence) -> [BillingPeriod | nil, late Boolean]` (trava FOR SHARE; `nil` para UBS); `Billing::Periods.following(competence) -> String`.
  - `Billing::Record.call(patient:, establishment:, professional:, source:, params:, recorded_by:, cbo_code: nil, attendance: nil, consultation: nil, today: Time.zone.today) -> Result` (`ok(procedure:)`; falhas `invalid_sigtap_code`, `invalid_quantity`, `invalid_date`, `invalid_cid10`, `invalid_character`); `params` com chaves string `performed_on` (ISO), `sigtap_code`, `quantity` (Integer), `cid10`, `character`.
  - `Billing::Record.insert!(patient:, establishment:, professional:, cbo_code:, source:, performed_on:, sigtap_code:, quantity:, cid10:, character:, recorded_by:, attendance: nil, consultation: nil, lab_report_item: nil) -> PerformedProcedure` (sem validar entrada; usado pelo laboratório).
  - `Billing::Cancel.call(procedure:, reason:, by:, now: Time.current) -> Result` (`ok(procedure:)`; `reason_required` (motivo fora de 10..500), `invalid_transition` (409), `already_exported` (409)).
  - Helper `recorded_procedure!(unit:, professional:, patient: nil, sigtap_code: "0301060029", performed_on: Time.zone.today, cid10: "I10", quantity: 1, character: "elective")` em `spec/support/billing_helpers.rb`.

- [ ] **Step 1: Teste que falha** — `spec/commands/billing/record_spec.rb`:

  ```ruby
  require "rails_helper"

  # ADR 0039 (spec §5.1, §5.3, §5.4; contrato §3 e §5): grava sempre, com a
  # crítica nos campos; competência do estabelecimento faturante nasce aberta;
  # UBS sem competência; procedimento de competência fechada vai para a
  # próxima aberta, atrasado; entradas inválidas recusadas sem gravar nada.
  RSpec.describe Billing::Record do
    include ActiveSupport::Testing::TimeHelpers

    before do
      travel_to(Time.zone.local(Time.zone.today.year, Time.zone.today.month, 15, 12, 0, 0))
      billing_city!
    end

    after { travel_back }

    let(:unit) { upa! }
    let(:doctor) { doctor!(unit) }
    let(:patient) { billing_patient! }
    let(:today) { Time.zone.today }
    let(:establishment) { Billing::Establishment.for_unit(unit) }

    def record(over = {}, establishment: self.establishment, professional: doctor)
      described_class.call(patient: patient, establishment: establishment, professional: professional, source: "manual", recorded_by: professional,
                           params: { "performed_on" => today.iso8601, "sigtap_code" => "0301060029", "quantity" => 1, "cid10" => "I10" }.merge(over))
    end

    it "grava faturável na competência do mês, que nasce aberta, e publica só ids" do
      procedure = record.payload[:procedure]
      expect(procedure).to have_attributes(instrument: "bpa_i", billable: true, blocking_reasons: [], cbo_code: "225125", source: "manual",
                                           billing_competence: today.strftime("%Y%m"), late: false, sigtap_fallback: false,
                                           character: "elective", recorded_by_user_id: doctor.id)
      expect(procedure.billing_period).to have_attributes(status: "open", version: 0, health_unit_id: unit.id, competence: today.strftime("%Y%m"))
      event = DomainEvent.where(name: "performed_procedure.recorded").order(:occurred_at).last
      expect(event.payload).to eq("performed_procedure_id" => procedure.id, "source" => "manual", "billable" => true)
    end

    it "sempre grava: não faturável fica com o motivo" do
      procedure = record("cid10" => nil).payload[:procedure]
      expect(procedure).to have_attributes(billable: false, blocking_reasons: %w[cid_required], status: "recorded")
    end

    it "UBS nunca ganha competência" do
      ubs = create_unit("UBS #{SecureRandom.hex(2)}")
      link_professional!(doctor, ubs, cbo: "225125")
      procedure = record({}, establishment: Billing::Establishment.for_unit(ubs)).payload[:procedure]
      expect(procedure).to have_attributes(billable: false, blocking_reasons: %w[establishment_is_aps], billing_period_id: nil, billing_competence: nil)
      expect(BillingPeriod.where(health_unit_id: ubs.id)).to be_empty
    end

    it "recusa entrada inválida sem gravar nada" do
      {
        { "sigtap_code" => "030106002" } => :invalid_sigtap_code, { "sigtap_code" => 301_060_029 } => :invalid_sigtap_code,
        { "quantity" => 0 } => :invalid_quantity, { "quantity" => 1000 } => :invalid_quantity, { "quantity" => "1" } => :invalid_quantity,
        { "performed_on" => (today + 1).iso8601 } => :invalid_date,
        { "performed_on" => (today << 4).end_of_month.iso8601 } => :invalid_date, { "performed_on" => "15/10/2026" } => :invalid_date,
        { "cid10" => "I1" } => :invalid_cid10, { "cid10" => "Z999" } => :invalid_cid10, { "character" => "emergency" } => :invalid_character
      }.each do |over, reason|
        expect(record(over).reason).to eq(reason), over.inspect
      end
      expect(PerformedProcedure.count).to eq(0)
      expect(BillingPeriod.count).to eq(0)
    end

    it "aceita a data até o 1º dia da competência de 3 meses antes e CID com ponto" do
      result = record("performed_on" => (today << 3).beginning_of_month.iso8601, "cid10" => "e11.9")
      expect(result).to be_ok
      expect(result.payload[:procedure].cid10).to eq("E119")
    end

    it "atraso de competência: competência fechada empurra para a próxima aberta, com late" do
      previous = (today << 1).strftime("%Y%m")
      BillingPeriod.create!(health_unit: unit, competence: previous, status: "closed", version: 1, closed_at: Time.current, closed_by_user: doctor)
      procedure = record("performed_on" => (today << 1).iso8601).payload[:procedure]
      expect(procedure).to have_attributes(billing_competence: today.strftime("%Y%m"), late: true, performed_on: today << 1)
    end

    it "duas competências fechadas seguidas: pula as duas" do
      [ 2, 1 ].each do |back|
        BillingPeriod.create!(health_unit: unit, competence: (today << back).strftime("%Y%m"), status: "closed", version: 1,
                              closed_at: Time.current, closed_by_user: doctor)
      end
      procedure = record("performed_on" => (today << 2).iso8601).payload[:procedure]
      expect(procedure).to have_attributes(billing_competence: today.strftime("%Y%m"), late: true)
    end

    it "laboratório municipal tem competência própria" do
      lab = lab_provider!
      procedure = record({ "sigtap_code" => "0202010473", "cid10" => nil }, establishment: Billing::Establishment.for_lab(lab)).payload[:procedure]
      expect(procedure.billing_period).to have_attributes(exam_provider_id: lab.id, health_unit_id: nil)
    end
  end
  ```

  `spec/commands/billing/cancel_spec.rb`:

  ```ruby
  require "rails_helper"

  # Contrato §3 com B8: cancelar com motivo (10..500, cifrado); uma vez; 409 already_exported se o
  # procedimento entrou no arquivo de uma competência fechada; depois da
  # reabertura, pode.
  RSpec.describe Billing::Cancel do
    before { billing_city! }

    let(:unit) { upa! }
    let(:doctor) { doctor!(unit) }
    let(:procedure) { recorded_procedure!(unit: unit, professional: doctor) }

    it "cancela com motivo e publica só o id" do
      result = described_class.call(procedure: procedure, reason: "  Lançado em duplicidade ", by: doctor)
      expect(result.payload[:procedure]).to have_attributes(status: "cancelled", cancel_reason: "Lançado em duplicidade", cancelled_by_user_id: doctor.id)
      expect(DomainEvent.where(name: "performed_procedure.cancelled").last.payload).to eq("performed_procedure_id" => procedure.id)
    end

    it "motivo obrigatório e até 500; cancelado não cancela de novo" do
      expect(described_class.call(procedure: procedure, reason: " ", by: doctor).reason).to eq(:reason_required)
      expect(described_class.call(procedure: procedure, reason: "Curto    ", by: doctor).reason).to eq(:reason_required)
      expect(described_class.call(procedure: procedure, reason: "x" * 501, by: doctor).reason).to eq(:reason_required)
      expect(described_class.call(procedure: procedure, reason: 7, by: doctor).reason).to eq(:reason_required)
      described_class.call(procedure: procedure, reason: "Erro de digitação", by: doctor)
      expect(described_class.call(procedure: procedure.reload, reason: "Erro de digitação", by: doctor).reason).to eq(:invalid_transition)
    end

    it "409 se já entrou no arquivo de competência fechada; reaberta, cancela" do
      period = procedure.billing_period
      export = BillingExport.create!(billing_period: period, version: 1, cnes: unit.cnes, competence: period.competence, file: "x",
                                     sha256: "a" * 64, lines_bpa_c: 0, lines_bpa_i: 1, sheets: 1, control_field: 1111,
                                     generated_by_user: doctor, generated_at: Time.current)
      procedure.update!(billing_export: export)
      period.update!(status: "closed", version: 1, closed_at: Time.current, closed_by_user: doctor)
      expect(described_class.call(procedure: procedure, reason: "Erro de digitação", by: doctor).reason).to eq(:already_exported)
      period.update!(status: "open", reopened_reason: "Correção do CNS", reopened_at: Time.current, reopened_by_user: doctor)
      expect(described_class.call(procedure: procedure, reason: "Erro de digitação", by: doctor)).to be_ok
    end
  end
  ```

  Acrescente a `spec/support/billing_helpers.rb`, dentro do módulo:

  ```ruby
    # Procedimento pelo caminho real (avulso do profissional, Billing::Record).
    def recorded_procedure!(unit:, professional:, patient: nil, sigtap_code: "0301060029", performed_on: Time.zone.today, cid10: "I10",
                            quantity: 1, character: "elective")
      establishment = unit.is_a?(ExamProvider) ? Billing::Establishment.for_lab(unit) : Billing::Establishment.for_unit(unit)
      result = Billing::Record.call(patient: patient || billing_patient!, establishment: establishment, professional: professional,
                                    source: "manual", recorded_by: professional,
                                    params: { "performed_on" => performed_on.iso8601, "sigtap_code" => sigtap_code, "quantity" => quantity,
                                              "cid10" => cid10, "character" => character })
      raise "procedimento recusado: #{result.reason}" if result.failure?

      result.payload[:procedure]
    end
  ```

- [ ] **Step 2: Rode e veja falhar:** `rspec spec/commands/billing/record_spec.rb spec/commands/billing/cancel_spec.rb` → `NameError: uninitialized constant Billing::Record`.

- [ ] **Step 3: Implemente.** `app/services/billing/periods.rb`:

  ```ruby
  # Competência por estabelecimento faturante (ADR 0039; plano do módulo 22,
  # Desvios 3 e 4). Nasce aberta na primeira produção; a UBS nunca ganha uma.
  # Quem grava procedimento trava a competência FOR SHARE; o fechamento a trava
  # FOR UPDATE (Billing::Close): um espera o outro. Ordem de travas:
  # competência → procedimento. Fechada, o procedimento vai para a próxima
  # competência aberta do estabelecimento, atrasado.
  module Billing
    module Periods
      COMPETENCE = /\A\d{4}(0[1-9]|1[0-2])\z/
      MAX_AHEAD = 24

      module_function

      def assign!(establishment, competence)
        return [ nil, false ] if establishment.aps?

        current = competence
        MAX_AHEAD.times do
          period = BillingPeriod.lock("FOR SHARE").find(find_or_create!(establishment, current).id)
          return [ period, current != competence ] if period.status == "open"

          current = following(current)
        end
        raise ActiveRecord::RecordNotFound, "nenhuma competência aberta em #{MAX_AHEAD} meses"
      end

      def find_or_create!(establishment, competence)
        BillingPeriod.find_by(**establishment.key, competence: competence) ||
          BillingPeriod.transaction(requires_new: true) { BillingPeriod.create!(**establishment.key, competence: competence) }
      rescue ActiveRecord::RecordNotUnique
        BillingPeriod.find_by!(**establishment.key, competence: competence)
      end

      def following(competence) = Date.strptime(competence, "%Y%m").next_month.strftime("%Y%m")
      private_class_method :find_or_create!
    end
  end
  ```

  `app/commands/billing/record.rb`:

  ```ruby
  # Gravação do procedimento realizado (ADR 0039; spec §5.1–§5.3; contrato §3).
  # Sempre grava: a crítica vai nos campos (Billing::Critic, lida fora da
  # transação). A competência do estabelecimento é travada FOR SHARE antes do
  # INSERT (Billing::Periods). Data: hoje ou antes, até o 1º dia da
  # competência de 3 meses antes (BPA 04.13). Ordem das recusas: código,
  # quantidade, data, CID, caráter. `insert!` é o caminho sem validação de
  # entrada (laboratório, Billing::FromLab).
  module Billing
    module Record
      CODE = /\A\d{10}\z/
      CID = /\A[A-Z]\d{2}[0-9X]?\z/
      DATE = /\A\d{4}-\d{2}-\d{2}\z/
      WINDOW_MONTHS = 3

      module_function

      def call(patient:, establishment:, professional:, source:, params:, recorded_by:, cbo_code: nil, attendance: nil, consultation: nil,
               today: Time.zone.today)
        code = params["sigtap_code"]
        return Result.fail(:invalid_sigtap_code) unless code.is_a?(String) && code.match?(CODE)

        quantity = params["quantity"]
        return Result.fail(:invalid_quantity) unless quantity.is_a?(Integer) && quantity.between?(1, 999)

        on = date(params["performed_on"])
        return Result.fail(:invalid_date) unless on && on <= today && on >= (today << WINDOW_MONTHS).beginning_of_month

        cid = cid10(params["cid10"])
        return Result.fail(:invalid_cid10) if cid == :invalid

        character = params["character"].nil? ? "elective" : params["character"]
        return Result.fail(:invalid_character) unless PerformedProcedure::CHARACTERS.include?(character)

        procedure = insert!(patient: patient, establishment: establishment, professional: professional,
                            cbo_code: cbo_code || Staff.cbo_for(professional, establishment), source: source, performed_on: on,
                            sigtap_code: code, quantity: quantity, cid10: cid, character: character, recorded_by: recorded_by,
                            attendance: attendance, consultation: consultation)
        Result.ok(procedure: procedure)
      end

      def insert!(patient:, establishment:, professional:, cbo_code:, source:, performed_on:, sigtap_code:, quantity:, cid10:, character:,
                  recorded_by:, attendance: nil, consultation: nil, lab_report_item: nil)
        outcome = Critic.call(Critic::Facts.new(establishment: establishment, sigtap_code: sigtap_code, performed_on: performed_on,
                                                cbo_code: cbo_code, professional_cns: Staff.cns(professional), cid10: cid10, patient: patient))
        ApplicationRecord.transaction(requires_new: true) do
          period, late = Periods.assign!(establishment, performed_on.strftime("%Y%m"))
          procedure = PerformedProcedure.create!(
            **establishment.key, patient: patient, professional_user: professional, cbo_code: cbo_code, performed_on: performed_on,
            sigtap_code: sigtap_code, quantity: quantity, cid10: cid10, character: character, source: source, attendance: attendance,
            consultation: consultation, lab_report_item: lab_report_item, recorded_by_user: recorded_by, instrument: outcome.instrument,
            billable: outcome.billable, blocking_reasons: outcome.reasons, sigtap_competence: outcome.sigtap_competence,
            sigtap_fallback: outcome.fallback, billing_period: period, billing_competence: period&.competence, late: late
          )
          DomainEvents.publish("performed_procedure.recorded", performed_procedure_id: procedure.id, source: source,
                                                                billable: outcome.billable)
          procedure
        end
      end

      def date(value)
        value.is_a?(String) && value.match?(DATE) ? Date.iso8601(value) : nil
      rescue Date::Error
        nil
      end

      # nil (sem CID), o código sem ponto em maiúsculas, ou :invalid (fora do
      # formato, ou fora da CID-10 ativa quando há uma).
      def cid10(value)
        return nil if value.nil? || value == ""
        return :invalid unless value.is_a?(String)

        code = value.delete(".").strip.upcase
        return :invalid unless code.match?(CID)

        release = TerminologyRelease.active.where(kind: "cid10").order(activated_at: :desc).first
        release.nil? || Cid10Code.exists?(release_id: release.id, code: code) ? code : :invalid
      end
      private_class_method :date, :cid10
    end
  end
  ```

  `app/commands/billing/cancel.rb`:

  ```ruby
  # Cancelamento do procedimento (contrato §3; B8): motivo 10..500 (cifrado), uma
  # vez só (trigger). Procedimento que entrou no arquivo de competência
  # fechada: 409 already_exported (reabra a competência antes). Travas na ordem
  # do módulo: competência (FOR SHARE) → procedimento.
  module Billing
    module Cancel
      module_function

      def call(procedure:, reason:, by:, now: Time.current)
        text = reason.is_a?(String) ? reason.strip : ""
        return Result.fail(:reason_required) unless text.length.between?(10, 500)

        ApplicationRecord.transaction do
          period = procedure.billing_period_id && BillingPeriod.lock("FOR SHARE").find(procedure.billing_period_id)
          procedure.lock!
          next Result.fail(:invalid_transition, details: { status: procedure.status }) unless procedure.status == "recorded"
          next Result.fail(:already_exported) if period&.closed? && procedure.billing_export_id

          procedure.update!(status: "cancelled", cancel_reason: text, cancelled_at: now, cancelled_by_user: by)
          DomainEvents.publish("performed_procedure.cancelled", performed_procedure_id: procedure.id)
          Result.ok(procedure: procedure)
        end
      end
    end
  end
  ```

- [ ] **Step 4: Rode:** `rspec spec/commands/billing/record_spec.rb spec/commands/billing/cancel_spec.rb spec/models/billing_schema_spec.rb` → verde.

- [ ] **Step 5: Commit:**

  ```bash
  git add app/services/billing/periods.rb app/commands/billing/record.rb app/commands/billing/cancel.rb spec/commands/billing/record_spec.rb spec/commands/billing/cancel_spec.rb spec/support/billing_helpers.rb
  git commit -m "feat: record performed procedures with competence assignment and cancellation

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

---

### Task 6: Rotas de procedimento, buscas de apoio, SIGTAP e procedimentos no atendimento e na consulta (F-22.2, F-22.4)

**Files:**
- Create: `app/controllers/billing/base_controller.rb`, `app/controllers/billing/procedures_controller.rb`, `app/controllers/billing/sigtap_controller.rb`, `app/controllers/billing/lookups_controller.rb`, `spec/requests/billing/lookups_spec.rb`, `app/services/billing/access.rb`, `app/services/billing/json.rb`, `app/services/billing/labels.rb`, `spec/requests/billing/procedures_spec.rb`, `spec/requests/billing/sigtap_spec.rb`, `spec/services/billing/attached_json_spec.rb`
- Modify: `config/routes.rb` (antes de `# Profissionais (ADR 0021; …)`), `app/controllers/attendances_controller.rb` (`attendance_json`), `app/services/consultations/json.rb` (`consultation`)

**Interfaces:**
- Consumes: `Billing::Record.call`, `Billing::Cancel.call` (Task 5); `Billing::Establishment`, `Billing::Staff`, `Billing::Critic::REASONS` (Task 4); `Billing::Gate` (Task 2); `Exams::Cursor.{encode,decode}`, `Timeline::Query.limit`.
- Produces:
  - `Billing::BaseController` (gate 403, `render_failure`, `render_error`, `forbid`, `not_found`, `invalid_filter`, `invalid_establishment`, `uuid?`; `FAILURE_STATUS` com `invalid_transition`/`already_exported` → 409, `mfa_required` → 401).
  - `Billing::Access.{clerk?, admin?, reader?}(user)`.
  - `Billing::Json.procedure(p, labels: {})`, `.procedures(list)`, `.summary(p, labels: {})`, `.attached(scope)`; `Billing::Labels.procedure(code, competence) -> String`.
  - Rotas `POST/GET /billing/procedures` (com `attendance_id`, B2), `POST /billing/procedures/:id/cancel`, `GET /billing/sigtap/search`, `POST /billing/patient_lookup` (B3), `GET /billing/establishments` (B4), `GET /billing/professionals` (B5); `procedures` no JSON do atendimento e da consulta (só com o interruptor ligado).

- [ ] **Step 1: Teste que falha** — `spec/requests/billing/procedures_spec.rb`:

  ```ruby
  require "rails_helper"

  # Contrato §3 com B1, B2, B5, B8 e B12 do dashboard (F-22.2, F-22.4): grava
  # pelo atendimento e pela consulta (paciente derivado), avulso do
  # profissional na própria unidade e lançamento do faturista (quem fez e CBO
  # opcional, só do faturista); sempre 201 com a crítica nos campos; lista
  # com envelope e filtros, inclusive por atendimento; cancelamento: o
  # profissional os próprios, o faturista qualquer um. Outro escopo por id →
  # 404; papel/escopo alheio → 403.
  RSpec.describe "Faturamento — procedimentos", type: :request do
    before { billing_city! }

    let(:unit) { upa! }
    let(:doctor) { doctor!(unit) }
    let(:clerk) { billing_clerk! }
    let(:pair) { exam_patient!(exam_seq!) }
    let(:patient) { pair.first }
    let(:today) { Time.zone.today.iso8601 }

    def body(**over)
      { patient_id: patient.id, establishment: { type: "health_unit", id: unit.id }, performed_on: today, sigtap_code: "0301060029",
        quantity: 1, cid10: "I10" }.merge(over)
    end

    def post_procedure(user, **over)
      sign_in_as(user)
      json_post "/billing/procedures", body(**over)
    end

    it "desligado: 403 feature_disabled" do
      Platform::Features.set!(city: Current.city, key: "procedure_billing", enabled: false, maintainer: ledi_maintainer!)
      post_procedure(doctor)
      expect(response).to have_http_status(:forbidden)
      expect(json_body).to eq("error" => "feature_disabled", "feature" => "procedure_billing")
    end

    it "avulso do profissional na própria unidade: 201 na forma do contrato" do
      post_procedure(doctor)
      expect(response).to have_http_status(:created)
      expect(json_body.keys).to match_array(%w[id patient establishment professional performed_on sigtap_code label sigtap_competence quantity
                                               cid10 character source source_ref instrument billable blocking_reasons status cancel_reason
                                               billing_competence late sigtap_fallback])
      expect(json_body).to include("patient" => { "id" => patient.id, "display_name" => "Maria Aparecida da Silva" },
                                   "establishment" => { "type" => "health_unit", "id" => unit.id, "cnes" => unit.cnes, "name" => unit.name },
                                   "performed_on" => today, "sigtap_code" => "0301060029", "quantity" => 1, "cid10" => "I10",
                                   "character" => "elective", "source" => "manual", "source_ref" => nil, "instrument" => "bpa_i",
                                   "billable" => true, "blocking_reasons" => [], "status" => "recorded", "late" => false,
                                   "label" => "ATENDIMENTO DE URGENCIA C/ OBSERVACAO ATE 24 HORAS EM ATENCAO ESPECIALIZADA")
      expect(json_body["professional"]).to include("user_id" => doctor.id, "cbo_code" => "225125")
    end

    it "não faturável também é 201, com o motivo" do
      post_procedure(doctor, cid10: nil)
      expect(response).to have_http_status(:created)
      expect(json_body).to include("billable" => false, "blocking_reasons" => %w[cid_required])
    end

    it "profissional sem vínculo na unidade e recepção: 403; profissional mandando quem fez: 403" do
      other = doctor!(create_unit("UBS #{SecureRandom.hex(2)}"))
      post_procedure(other)
      expect(response).to have_http_status(:forbidden)
      post_procedure(verifier!)
      expect(response).to have_http_status(:forbidden)
      post_procedure(doctor, professional_user_id: doctor.id)
      expect(response).to have_http_status(:forbidden)
      expect(PerformedProcedure.count).to eq(0)
    end

    it "lançamento do faturista: quem fez com vínculo no estabelecimento; CBO opcional, entre os do vínculo" do
      post_procedure(clerk)
      expect(json_body).to eq("error" => "invalid_professional")
      post_procedure(clerk, professional_user_id: doctor!(create_unit("UPA #{SecureRandom.hex(2)}", kind: "upa")).id)
      expect(json_body).to eq("error" => "invalid_professional")
      post_procedure(clerk, professional_user_id: doctor.id, cbo_code: "223505")
      expect(json_body).to eq("error" => "invalid_professional")
      post_procedure(clerk, professional_user_id: doctor.id, cbo_code: "225125")
      expect(response).to have_http_status(:created)
      expect(PerformedProcedure.find(json_body["id"])).to have_attributes(professional_user_id: doctor.id, recorded_by_user_id: clerk.id,
                                                                         source: "manual", cbo_code: "225125")
    end

    it "lançamento do faturista no laboratório: o liberador do laboratório" do
      lab = lab_provider!
      releaser = lab_releaser!(lab)
      post_procedure(clerk, establishment: { type: "lab_provider", id: lab.id }, professional_user_id: releaser.id, sigtap_code: "0202010473",
                            cid10: nil)
      expect(response).to have_http_status(:created)
      expect(json_body).to include("establishment" => include("type" => "lab_provider"), "blocking_reasons" => %w[missing_professional_cbo])
    end

    it "estabelecimento inválido 422; paciente inexistente 404; recusa de entrada 422" do
      post_procedure(clerk, professional_user_id: doctor.id, establishment: { type: "health_unit", id: SecureRandom.uuid })
      expect(json_body).to eq("error" => "invalid_establishment")
      post_procedure(clerk, professional_user_id: doctor.id, establishment: { type: "lab_provider", id: exam_provider!(name: "Particular").id })
      expect(json_body).to eq("error" => "invalid_establishment")
      post_procedure(clerk, professional_user_id: doctor.id, establishment: "x")
      expect(json_body).to eq("error" => "invalid_establishment")
      post_procedure(doctor, patient_id: SecureRandom.uuid)
      expect(response).to have_http_status(:not_found)
      post_procedure(doctor, quantity: 0)
      expect(json_body).to eq("error" => "invalid_quantity")
    end

    it "pelo atendimento (B1): paciente derivado; patient_id que não bate 422; atendimento inexistente 404" do
      attendance = walk_in_attendance!(unit, citizen: pair.last)
      post_procedure(doctor, patient_id: nil, attendance_id: attendance.id)
      expect(response).to have_http_status(:created)
      expect(json_body).to include("patient" => include("id" => patient.id), "source" => "attendance",
                                   "source_ref" => { "type" => "attendance", "id" => attendance.id })
      post_procedure(doctor, patient_id: billing_patient!.id, attendance_id: attendance.id)
      expect(json_body).to eq("error" => "invalid_patient")
      post_procedure(doctor, attendance_id: attendance.id, establishment: { type: "health_unit", id: create_unit("UBS #{SecureRandom.hex(2)}").id })
      expect(json_body).to eq("error" => "invalid_establishment")
      post_procedure(doctor, attendance_id: SecureRandom.uuid)
      expect(response).to have_http_status(:not_found)
    end

    it "pelo atendimento de par ainda sem paciente: o api resolve o paciente do par validado" do
      citizen = verified_citizen!(exam_seq!, age: 50)
      attendance = walk_in_attendance!(unit, citizen: citizen)
      post_procedure(doctor, patient_id: nil, attendance_id: attendance.id)
      expect(response).to have_http_status(:created)
      expect(json_body["patient"]["id"]).to eq(citizen.reload.patient_id)
    end

    it "pela consulta: source consultation, CBO da consulta; outro profissional 403" do
      consultation = started_consultation!(unit: unit, doctor: doctor, citizen: pair.last)
      post_procedure(doctor, patient_id: nil, consultation_id: consultation.id, establishment: nil)
      expect(response).to have_http_status(:created)
      expect(json_body).to include("source" => "consultation", "source_ref" => { "type" => "consultation", "id" => consultation.id })
      expect(json_body["professional"]["cbo_code"]).to eq(consultation.cbo_code)
      post_procedure(doctor!(unit), consultation_id: consultation.id, establishment: nil)
      expect(response).to have_http_status(:forbidden)
    end

    it "lista: faturista vê tudo com filtros e envelope; profissional os próprios do paciente ou os do atendimento (B2)" do
      ok = recorded_procedure!(unit: unit, professional: doctor, patient: patient)
      bad = recorded_procedure!(unit: unit, professional: doctor, patient: patient, cid10: nil)
      sign_in_as(clerk)
      get "/billing/procedures", params: { establishment_type: "health_unit", establishment_id: unit.id, competence: Time.zone.today.strftime("%Y%m"),
                                           billable: "false", reason: "cid_required" }
      expect(json_body["procedures"].map { |p| p["id"] }).to eq([ bad.id ])
      get "/billing/procedures", params: { limit: 1 }
      expect(json_body["procedures"].size).to eq(1)
      get "/billing/procedures", params: { limit: 1, before: json_body["next_before"] }
      expect(json_body["procedures"].map { |p| p["id"] }).to eq([ ok.id ])
      get "/billing/procedures", params: { reason: "whatever" }
      expect(json_body).to eq("error" => "invalid_filter")
      get "/billing/procedures", params: { before: "@@" }
      expect(json_body).to eq("error" => "invalid_cursor")
      sign_in_as(doctor)
      get "/billing/procedures"
      expect(response).to have_http_status(:forbidden)
      get "/billing/procedures", params: { patient_id: patient.id }
      expect(json_body["procedures"].size).to eq(2)
      sign_in_as(doctor!(unit))
      get "/billing/procedures", params: { patient_id: patient.id }
      expect(json_body["procedures"]).to eq([])
    end

    it "lista por atendimento: quem tem vínculo na unidade vê todos os do atendimento; sem vínculo 403" do
      attendance = walk_in_attendance!(unit, citizen: pair.last)
      sign_in_as(doctor)
      json_post "/billing/procedures", body(patient_id: nil, attendance_id: attendance.id)
      sign_in_as(doctor!(unit))
      get "/billing/procedures", params: { attendance_id: attendance.id }
      expect(json_body["procedures"].map { |p| p["source"] }).to eq(%w[attendance])
      sign_in_as(doctor!(create_unit("UBS #{SecureRandom.hex(2)}")))
      get "/billing/procedures", params: { attendance_id: attendance.id }
      expect(response).to have_http_status(:forbidden)
    end

    it "cancelar (B8, B12): o próprio profissional e o faturista; outro profissional 404; admin lê mas não cancela" do
      procedure = recorded_procedure!(unit: unit, professional: doctor, patient: patient)
      sign_in_as(doctor!(unit))
      json_post "/billing/procedures/#{procedure.id}/cancel", reason: "Lançado em duplicidade"
      expect(response).to have_http_status(:not_found)
      sign_in_as(lab_admin!)
      json_post "/billing/procedures/#{procedure.id}/cancel", reason: "Lançado em duplicidade"
      expect(response).to have_http_status(:forbidden)
      sign_in_as(doctor)
      json_post "/billing/procedures/#{procedure.id}/cancel", reason: "curto"
      expect(json_body).to eq("error" => "reason_required")
      json_post "/billing/procedures/#{procedure.id}/cancel", reason: "Lançado em duplicidade"
      expect(json_body).to include("status" => "cancelled", "cancel_reason" => "Lançado em duplicidade")
      json_post "/billing/procedures/#{procedure.id}/cancel", reason: "De novo, por engano"
      expect(response).to have_http_status(:conflict)
      other = recorded_procedure!(unit: unit, professional: doctor, patient: patient)
      sign_in_as(clerk)
      json_post "/billing/procedures/#{other.id}/cancel", reason: "Procedimento não realizado"
      expect(json_body["status"]).to eq("cancelled")
    end
  end
  ```

  `spec/requests/billing/lookups_spec.rb`:

  ```ruby
  require "rails_helper"

  # B3, B4 e B5 do dashboard: paciente pelo CPF (corpo, com trilha), lista de
  # estabelecimentos faturantes e de profissionais do estabelecimento.
  RSpec.describe "Faturamento — buscas de apoio", type: :request do
    before { billing_city! }

    let(:unit) { upa! }
    let(:pair) { exam_patient!(exam_seq!) }

    it "paciente pelo CPF: faturista e profissional de unidade não-APS; trilha só com ids; 404 e 422" do
      sign_in_as(billing_clerk!)
      json_post "/billing/patient_lookup", cpf: pair.first.cpf
      expect(json_body).to eq("patient" => { "id" => pair.first.id, "display_name" => "Maria Aparecida da Silva",
                                             "cpf_masked" => pair.first.cpf_masked })
      expect(DomainEvent.where(name: "billing_patient.looked_up").last.payload.keys).to match_array(%w[patient_id user_id])
      json_post "/billing/patient_lookup", cpf: "52998224725"
      expect(response).to have_http_status(:not_found)
      expect(json_body).to eq("error" => "patient_not_found")
      json_post "/billing/patient_lookup", cpf: "123"
      expect(json_body).to eq("error" => "invalid_cpf")
      sign_in_as(doctor!(unit))
      json_post "/billing/patient_lookup", cpf: pair.first.cpf
      expect(response).to have_http_status(:ok)
      sign_in_as(doctor!(create_unit("UBS #{SecureRandom.hex(2)}")))
      json_post "/billing/patient_lookup", cpf: pair.first.cpf
      expect(response).to have_http_status(:forbidden)
    end

    it "estabelecimentos faturantes: unidades não-APS ativas e laboratórios municipais; faturista e admin" do
      create_unit("UBS #{SecureRandom.hex(2)}")
      lab = lab_provider!
      unit
      sign_in_as(lab_admin!)
      get "/billing/establishments"
      expect(json_body["establishments"]).to include({ "type" => "health_unit", "id" => unit.id, "cnes" => unit.cnes, "name" => unit.name },
                                                     { "type" => "lab_provider", "id" => lab.id, "cnes" => lab.cnes, "name" => lab.name })
      expect(json_body["establishments"].map { |e| e["id"] }).not_to include(*HealthUnit.where(kind: "ubs").pluck(:id))
      sign_in_as(doctor!(unit))
      get "/billing/establishments"
      expect(response).to have_http_status(:forbidden)
    end

    it "profissionais do estabelecimento: vínculos ativos na unidade; liberadores no laboratório; só faturista" do
      doctor = doctor!(unit)
      lab = lab_provider!
      releaser = lab_releaser!(lab)
      sign_in_as(billing_clerk!)
      get "/billing/professionals", params: { establishment_type: "health_unit", establishment_id: unit.id }
      expect(json_body["professionals"]).to eq([ { "user_id" => doctor.id, "name" => doctor.professional.professional_name, "cbo_codes" => %w[225125] } ])
      get "/billing/professionals", params: { establishment_type: "lab_provider", establishment_id: lab.id }
      expect(json_body["professionals"].map { |p| p["user_id"] }).to eq([ releaser.id ])
      get "/billing/professionals", params: { establishment_type: "health_unit", establishment_id: SecureRandom.uuid }
      expect(json_body).to eq("error" => "invalid_establishment")
      sign_in_as(lab_admin!)
      get "/billing/professionals", params: { establishment_type: "health_unit", establishment_id: unit.id }
      expect(response).to have_http_status(:forbidden)
    end
  end
  ```

  `spec/requests/billing/sigtap_spec.rb`:

  ```ruby
  require "rails_helper"

  # Contrato §3: busca na SIGTAP da competência (reaproveita a plataforma),
  # com instrumento, CID exigido, sexo e idades; termo 2..60; sem release 503.
  RSpec.describe "Faturamento — busca SIGTAP", type: :request do
    before { billing_city! }

    let(:unit) { upa! }

    it "por nome e por início de código, para profissional e faturista" do
      sign_in_as(doctor!(unit))
      get "/billing/sigtap/search", params: { q: "urgencia" }
      item = json_body["procedures"].find { |p| p["code"] == "0301060029" }
      expect(item).to eq("code" => "0301060029", "name" => "ATENDIMENTO DE URGENCIA C/ OBSERVACAO ATE 24 HORAS EM ATENCAO ESPECIALIZADA",
                         "instruments" => %w[bpa_i], "requires_cid" => true, "sex" => nil, "age_min_months" => nil, "age_max_months" => nil)
      sign_in_as(billing_clerk!)
      get "/billing/sigtap/search", params: { q: "020102" }
      expect(json_body["procedures"].find { |p| p["code"] == "0201020033" }).to include("instruments" => %w[bpa_c bpa_i], "sex" => "F",
                                                                                      "age_min_months" => 300)
    end

    it "termo curto 422, recepção 403, competência sem release 503" do
      sign_in_as(billing_clerk!)
      get "/billing/sigtap/search", params: { q: "a" }
      expect(json_body).to eq("error" => "invalid_query")
      get "/billing/sigtap/search", params: { q: "consulta", competence: "199901" }
      expect(response).to have_http_status(:service_unavailable)
      expect(json_body).to eq("error" => "terminology_unavailable")
      sign_in_as(verifier!)
      get "/billing/sigtap/search", params: { q: "consulta" }
      expect(response).to have_http_status(:forbidden)
    end
  end
  ```

  `spec/services/billing/attached_json_spec.rb`:

  ```ruby
  require "rails_helper"

  # Contrato §3: o JSON do atendimento e da consulta ganha `procedures`
  # (forma resumida) só com procedure_billing ligado.
  RSpec.describe "Procedimentos no atendimento e na consulta" do
    before { billing_city! }

    let(:unit) { upa! }
    let(:doctor) { doctor!(unit) }
    let(:pair) { exam_patient!(exam_seq!) }

    it "a consulta e o atendimento listam os procedimentos dela/dele" do
      consultation = started_consultation!(unit: unit, doctor: doctor, citizen: pair.last)
      procedure = Billing::Record.call(patient: pair.first, establishment: Billing::Establishment.for_unit(unit), professional: doctor,
                                       source: "consultation", consultation: consultation, cbo_code: consultation.cbo_code, recorded_by: doctor,
                                       params: { "performed_on" => Time.zone.today.iso8601, "sigtap_code" => "0301060029", "quantity" => 1,
                                                 "cid10" => "I10" }).payload[:procedure]
      summary = { id: procedure.id, sigtap_code: "0301060029", label: "ATENDIMENTO DE URGENCIA C/ OBSERVACAO ATE 24 HORAS EM ATENCAO ESPECIALIZADA",
                  quantity: 1, performed_on: Time.zone.today.iso8601, billable: true, blocking_reasons: [], status: "recorded" }
      expect(Consultations::Json.consultation(consultation)[:procedures]).to eq([ summary ])
      walk_in = walk_in_attendance!(unit, citizen: pair.last)
      Billing::Record.call(patient: pair.first, establishment: Billing::Establishment.for_unit(unit), professional: doctor, source: "attendance",
                           attendance: walk_in, recorded_by: doctor,
                           params: { "performed_on" => Time.zone.today.iso8601, "sigtap_code" => "0301100012", "quantity" => 2 })
      expect(AttendancesController.new.send(:attendance_json, walk_in)[:procedures].map { |p| p[:sigtap_code] }).to eq(%w[0301100012])

      Platform::Features.set!(city: Current.city, key: "procedure_billing", enabled: false, maintainer: ledi_maintainer!)
      expect(Consultations::Json.consultation(consultation)).not_to have_key(:procedures)
      expect(AttendancesController.new.send(:attendance_json, walk_in)).not_to have_key(:procedures)
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar:** `rspec spec/requests/billing/procedures_spec.rb spec/requests/billing/lookups_spec.rb spec/requests/billing/sigtap_spec.rb spec/services/billing/attached_json_spec.rb` → `ActionController::RoutingError` / `NoMethodError`.

- [ ] **Step 3: Implemente.** `app/services/billing/access.rb`:

  ```ruby
  # Papéis do faturamento (ADR 0039; spec §3): o faturista opera; o admin
  # municipal lê. O profissional grava no próprio contexto (vínculo/autoria).
  module Billing
    module Access
      module_function

      def clerk?(user) = user.has_role?("billing_clerk")
      def admin?(user) = user.has_role?("municipal_admin")
      def reader?(user) = clerk?(user) || admin?(user)
    end
  end
  ```

  `app/services/billing/labels.rb`:

  ```ruby
  # Nome SIGTAP do procedimento na release usada na crítica (ou na última
  # ativa até a competência; sem release, o próprio código).
  module Billing
    module Labels
      module_function

      def procedure(code, competence)
        release = Terminology::Sigtap.release_for(competence)
        (release && SigtapProcedure.where(release_id: release.id, code: code).pick(:name)) || code
      end
    end
  end
  ```

  `app/services/billing/json.rb`:

  ```ruby
  # Formas do contrato do módulo 22 (§2): <performed_procedure> e o resumo do
  # atendimento/consulta. Rótulos SIGTAP memorizados por chamada (listas).
  module Billing
    module Json
      module_function

      def procedures(list)
        labels = {}
        list.map { |p| procedure(p, labels: labels) }
      end

      def procedure(p, labels: {})
        { id: p.id, patient: { id: p.patient_id, display_name: p.patient.display_name }, establishment: Establishment.of(p).json,
          professional: { user_id: p.professional_user_id, name: Screenings::Json.staff_name(p.professional_user), cbo_code: p.cbo_code },
          performed_on: p.performed_on.iso8601, sigtap_code: p.sigtap_code, label: label(p, labels), sigtap_competence: p.sigtap_competence,
          quantity: p.quantity, cid10: p.cid10, character: p.character, source: p.source, source_ref: source_ref(p),
          instrument: p.instrument, billable: p.billable, blocking_reasons: p.blocking_reasons, status: p.status,
          cancel_reason: p.cancel_reason, billing_competence: p.billing_competence, late: p.late, sigtap_fallback: p.sigtap_fallback }
      end

      def summary(p, labels: {})
        { id: p.id, sigtap_code: p.sigtap_code, label: label(p, labels), quantity: p.quantity, performed_on: p.performed_on.iso8601,
          billable: p.billable, blocking_reasons: p.blocking_reasons, status: p.status }
      end

      # `procedures` do atendimento e da consulta (contrato §3), na ordem de gravação.
      def attached(scope)
        labels = {}
        scope.order(:created_at, :id).map { |p| summary(p, labels: labels) }
      end

      def source_ref(p)
        case p.source
        when "attendance" then { type: "attendance", id: p.attendance_id }
        when "consultation" then { type: "consultation", id: p.consultation_id }
        when "lab_report" then { type: "lab_report_item", id: p.lab_report_item_id }
        end
      end

      def label(p, labels) = labels[[ p.sigtap_code, p.sigtap_competence ]] ||= Labels.procedure(p.sigtap_code, p.sigtap_competence)
      private_class_method :source_ref, :label
    end
  end
  ```

  `app/controllers/billing/base_controller.rb`:

  ```ruby
  # Base das rotas /billing/* (ADR 0039; contrato §3–§6): sessão do dashboard
  # e procedure_billing ligado (403 feature_disabled). Recusas no formato
  # { error }; 409 para transição inválida e competência fechada.
  module Billing
    class BaseController < ApplicationController
      include Authentication
      include MfaStepUp

      wrap_parameters false

      UUID = Exams::BaseController::UUID
      FAILURE_STATUS = { invalid_transition: :conflict, already_exported: :conflict, not_found: :not_found, patient_not_found: :not_found,
                         mfa_required: :unauthorized }.freeze

      before_action :require_billing!

      private

      def require_billing!
        return if Gate.enabled?(Current.city)

        render json: { error: "feature_disabled", feature: Gate::KEY }, status: :forbidden
      end

      def render_failure(result)
        payload = { error: result.reason.to_s }
        payload[:status] = result.details[:status] if result.details[:status]
        payload[:field] = result.details[:field] if result.details[:field]
        render json: payload, status: FAILURE_STATUS.fetch(result.reason, :unprocessable_entity)
      end

      def render_error(reason, status = :unprocessable_entity) = render(json: { error: reason.to_s }, status: status)
      def forbid(reason = "missing_role") = render(json: { error: reason }, status: :forbidden)
      def not_found = render(json: { error: "not_found" }, status: :not_found)
      def invalid_filter = render_error(:invalid_filter)
      def invalid_establishment = render_error(:invalid_establishment)
      def uuid?(value) = value.is_a?(String) && value.match?(UUID)
    end
  end
  ```

  `app/controllers/billing/procedures_controller.rb`:

  ```ruby
  # /billing/procedures (ADR 0039; contrato §3 com B1, B2, B5 e B12 do
  # dashboard): gravar pelo atendimento ou pela consulta (paciente derivado;
  # patient_id, se vier, tem de bater), avulso do profissional na própria
  # unidade ou lançamento do faturista (professional_user_id e cbo_code só
  # dele); listar (faturista e admin tudo; profissional os próprios por
  # paciente, ou os do atendimento da unidade em que tem vínculo); cancelar
  # (o profissional os próprios; o faturista qualquer um).
  module Billing
    class ProceduresController < BaseController
      RECORD_KEYS = %w[performed_on sigtap_code quantity cid10 character].freeze

      def create
        clerk = Access.clerk?(Current.user)
        return forbid unless clerk || Current.user.has_role?("health_professional")
        return forbid if !clerk && (params.key?(:professional_user_id) || params.key?(:cbo_code))

        context = if params[:consultation_id].present? then consultation_context
                  elsif params[:attendance_id].present? then attendance_context
                  else standalone_context(clerk)
                  end
        return if performed?

        result = Record.call(params: RECORD_KEYS.to_h { |key| [ key, params[key] ] }, recorded_by: Current.user, **context)
        return render_failure(result) if result.failure?

        render json: Json.procedure(result.payload[:procedure]), status: :created
      end

      def index
        scope = PerformedProcedure.includes(:patient, :health_unit, :exam_provider, professional_user: :professional)
        unless Access.reader?(Current.user)
          professional = Current.user.has_role?("health_professional")
          if uuid?(params[:attendance_id])
            attendance = Attendance.find_by(id: params[:attendance_id])
            return not_found unless attendance
            return forbid("missing_link") unless professional && Staff.linked?(Current.user, attendance.health_unit_id)
          elsif professional && uuid?(params[:patient_id])
            scope = scope.where(professional_user_id: Current.user.id)
          else
            return forbid
          end
        end
        scope = scope.where(attendance_id: params[:attendance_id]) if uuid?(params[:attendance_id])
        scope = scope.where(patient_id: params[:patient_id]) if uuid?(params[:patient_id])
        scope = filtered(scope)
        return if performed?

        cursor = Exams::Cursor.decode(params[:before])
        scope = scope.where("(performed_procedures.created_at, performed_procedures.id) < (?, ?)", *cursor) if cursor
        limit = Timeline::Query.limit(params[:limit])
        rows = scope.order(created_at: :desc, id: :desc).limit(limit + 1).to_a
        page = rows.first(limit)
        render json: { procedures: Json.procedures(page), next_before: rows.size > limit ? Exams::Cursor.encode(page.last) : nil }
      rescue Exams::Cursor::Invalid
        render json: { error: "invalid_cursor" }, status: :unprocessable_entity
      end

      def cancel
        procedure = uuid?(params[:id]) ? PerformedProcedure.find_by(id: params[:id]) : nil
        own = procedure && procedure.professional_user_id == Current.user.id
        return not_found unless procedure && (own || Access.reader?(Current.user))
        return forbid unless own || Access.clerk?(Current.user)

        result = Cancel.call(procedure: procedure, reason: params[:reason], by: Current.user)
        return render_failure(result) if result.failure?

        render json: Json.procedure(result.payload[:procedure])
      end

      private

      def filtered(scope)
        if params[:establishment_id].present?
          key = { "health_unit" => :health_unit_id, "lab_provider" => :exam_provider_id }[params[:establishment_type]]
          return invalid_filter unless key && uuid?(params[:establishment_id])

          scope = scope.where(key => params[:establishment_id])
        end
        if params[:competence].present?
          return invalid_filter unless params[:competence].is_a?(String) && params[:competence].match?(Periods::COMPETENCE)

          scope = scope.where(billing_competence: params[:competence])
        end
        scope = scope.where(billable: params[:billable] == "true") if %w[true false].include?(params[:billable])
        if params[:reason].present?
          return invalid_filter unless Critic::REASONS.include?(params[:reason])

          scope = scope.where("? = ANY (performed_procedures.blocking_reasons)", params[:reason])
        end
        scope
      end

      def consultation_context
        consultation = uuid?(params[:consultation_id]) ? Consultation.find_by(id: params[:consultation_id]) : nil
        return not_found unless consultation
        return render_error(:invalid_patient) unless patient_matches?(consultation.patient_id)
        return forbid("not_author") unless consultation.author_user_id == Current.user.id

        unit = consultation.attendance.health_unit
        return invalid_establishment unless same_unit?(unit)

        { patient: consultation.patient, establishment: Establishment.for_unit(unit), professional: Current.user,
          cbo_code: consultation.cbo_code, source: "consultation", consultation: consultation }
      end

      # B1: o paciente vem do par do atendimento (Patients::Resolve, se o par
      # validado ainda não tem paciente).
      def attendance_context
        attendance = uuid?(params[:attendance_id]) ? Attendance.includes(:citizen, :health_unit).find_by(id: params[:attendance_id]) : nil
        return not_found unless attendance
        return forbid("missing_link") unless Current.user.has_role?("health_professional") && Staff.linked?(Current.user, attendance.health_unit_id)

        resolved = attendance.citizen.patient_id ? nil : Patients::Resolve.call(attendance.citizen)
        return render_error(resolved.reason) if resolved&.failure?

        patient = resolved ? resolved.payload[:patient] : Patient.find(attendance.citizen.patient_id)
        return render_error(:invalid_patient) unless patient_matches?(patient.id)
        return invalid_establishment unless same_unit?(attendance.health_unit)

        { patient: patient, establishment: Establishment.for_unit(attendance.health_unit), professional: Current.user, source: "attendance",
          attendance: attendance }
      end

      def standalone_context(clerk)
        patient = uuid?(params[:patient_id]) ? Patient.find_by(id: params[:patient_id]) : nil
        return not_found unless patient

        establishment = Establishment.find(*establishment_param)
        return invalid_establishment unless establishment
        return clerk_context(patient, establishment) if clerk
        return forbid unless establishment.type == "health_unit" && Staff.linked?(Current.user, establishment.id)

        { patient: patient, establishment: establishment, professional: Current.user, source: "manual" }
      end

      # B5: quem fez tem vínculo no estabelecimento (liberador ativo no
      # laboratório); o CBO, se vier, é um dos do vínculo.
      def clerk_context(patient, establishment)
        professional = uuid?(params[:professional_user_id]) ? User.find_by(id: params[:professional_user_id]) : nil
        return render_error(:invalid_professional) unless professional&.professional && Staff.eligible?(professional, establishment)

        cbo = params[:cbo_code].presence
        return render_error(:invalid_professional) if cbo && !Staff.cbos_for(professional, establishment).include?(cbo)

        { patient: patient, establishment: establishment, professional: professional, cbo_code: cbo, source: "manual" }
      end

      def patient_matches?(patient_id) = params[:patient_id].blank? || params[:patient_id] == patient_id

      # [type, id] do corpo; no atendimento e na consulta, se vier, tem de ser a unidade deles.
      def establishment_param
        value = params[:establishment]
        value.respond_to?(:key?) ? [ value[:type], value[:id] ] : [ nil, nil ]
      end

      def same_unit?(unit)
        type, id = establishment_param
        (type.nil? && id.nil?) || (type == "health_unit" && id == unit.id)
      end
    end
  end
  ```

  `app/controllers/billing/lookups_controller.rb`:

  ```ruby
  # Buscas de apoio do dashboard (B3, B4, B5): paciente pelo CPF (no corpo,
  # nunca na URL; trilha com ids), estabelecimentos faturantes (unidade
  # não-APS ativa e laboratório municipal ativo) e profissionais do
  # estabelecimento (vínculos ativos; liberadores no laboratório).
  module Billing
    class LookupsController < BaseController
      def patient
        allowed = Access.clerk?(Current.user) ||
                  (Current.user.has_role?("health_professional") && Staff.non_aps_units(Current.user).any?)
        return forbid unless allowed

        cpf = params[:cpf].is_a?(String) ? CitizenIdentity::Cpf.normalize(params[:cpf]) : nil
        return render_error(:invalid_cpf) unless cpf

        found = Patient.find_by(cpf: cpf)
        return render_error(:patient_not_found, :not_found) unless found

        DomainEvents.publish("billing_patient.looked_up", patient_id: found.id, user_id: Current.user.id)
        render json: { patient: { id: found.id, display_name: found.display_name, cpf_masked: found.cpf_masked } }
      end

      def establishments
        return forbid unless Access.reader?(Current.user)

        units = HealthUnit.where(active: true).where.not(kind: "ubs").order(:name).map { |unit| Establishment.for_unit(unit).json }
        labs = ExamProvider.where(uses_municipal_lab: true, active: true).order(:name).map { |lab| Establishment.for_lab(lab).json }
        render json: { establishments: units + labs }
      end

      def professionals
        return forbid unless Access.clerk?(Current.user)

        establishment = Establishment.find(params[:establishment_type], params[:establishment_id])
        return invalid_establishment unless establishment

        users = if establishment.type == "health_unit"
                  User.where(id: Professional.joins(:links).merge(ProfessionalLink.active.where(health_unit_id: establishment.id)).select(:user_id))
                else
                  User.where(id: LabReleaser.active.where(exam_provider_id: establishment.id).select(:user_id))
                end
        rows = users.includes(:professional).select(&:professional).sort_by { |user| user.professional.professional_name }
        render json: { professionals: rows.map { |user| { user_id: user.id, name: user.professional.professional_name,
                                                          cbo_codes: Staff.cbos_for(user, establishment) } } }
      end
    end
  end
  ```

  `app/controllers/billing/sigtap_controller.rb`:

  ```ruby
  # GET /billing/sigtap/search (contrato §3): SIGTAP da competência (padrão: a
  # do dia; a última ativa até ela), por início de código ou por nome, até 20.
  # Profissional ou faturista. `q` não vai a log (filtro /\Aq\z/).
  module Billing
    class SigtapController < BaseController
      LIMIT = 20
      INSTRUMENTS = { "02" => "bpa_i", "01" => "bpa_c" }.freeze

      def search
        return forbid unless Access.clerk?(Current.user) || Current.user.has_role?("health_professional")

        q = params[:q].is_a?(String) ? params[:q].strip : ""
        return render_error(:invalid_query) unless q.length.between?(2, 60)

        competence = params[:competence].presence || Time.zone.today.strftime("%Y%m")
        return invalid_filter unless competence.is_a?(String) && competence.match?(Periods::COMPETENCE)

        release = Terminology::Sigtap.release_for(competence)
        return render_error(:terminology_unavailable, :service_unavailable) unless release

        scope = SigtapProcedure.where(release_id: release.id)
        scope = q.match?(/\A\d+\z/) ? scope.where("code LIKE ?", "#{q}%") : scope.where("name ILIKE ?", "%#{SigtapProcedure.sanitize_sql_like(q)}%")
        rows = scope.order(:name, :code).limit(LIMIT).to_a
        render json: { procedures: rows.map { |row| item(row, release, rows) } }
      end

      private

      def item(row, release, rows)
        @instruments ||= SigtapProcedureInstrument.where(release_id: release.id, procedure_code: rows.map(&:code))
                                                  .pluck(:procedure_code, :instrument_code).group_by(&:first)
        @with_cid ||= SigtapProcedureCid.where(release_id: release.id, procedure_code: rows.map(&:code)).distinct.pluck(:procedure_code).to_set
        instruments = Array(@instruments[row.code]).map { |_, code| INSTRUMENTS.fetch(code, "not_bpa") }.uniq.sort
        { code: row.code, name: row.name, instruments: instruments, requires_cid: @with_cid.include?(row.code), sex: row.sex,
          age_min_months: row.age_min_months, age_max_months: row.age_max_months }
      end
    end
  end
  ```

  Em `config/routes.rb`, antes da linha `  # Profissionais (ADR 0021; spec 2026-09-27-module-10-professionals §4.1).`:

  ```ruby
    # Módulo 22 (ADR 0039; contrato §3–§6): procedimentos e faturamento BPA.
    # Prefixo único (uma entrada no proxy de dev do dashboard). Literais antes de `:id`.
    scope "/billing", module: :billing, as: :billing do
      get  "sigtap/search",         to: "sigtap#search"
      post "patient_lookup",        to: "lookups#patient"
      get  "establishments",        to: "lookups#establishments"
      get  "professionals",         to: "lookups#professionals"
      get  "procedures",            to: "procedures#index"
      post "procedures",            to: "procedures#create"
      post "procedures/:id/cancel", to: "procedures#cancel"
    end

  ```

  Em `app/controllers/attendances_controller.rb`, troque `attendance_json` por:

  ```ruby
    def attendance_json(a)
      json = {
        id: a.id, triage_id: a.triage_id, appointment_id: a.appointment_id, health_unit_id: a.health_unit_id,
        unit_name: a.health_unit.name, status: a.status, checked_in_at: a.checked_in_at&.iso8601,
        check_in_method: a.check_in_method, called_at: a.called_at&.iso8601, outcome: a.outcome,
        referral_unit_name: a.referral_unit&.name, referral_note: a.referral_note, closed_at: a.closed_at&.iso8601,
        called_by_name: Screenings::Json.staff_name(a.called_by_user), closed_by_name: Screenings::Json.staff_name(a.closed_by_user)
      }
      # ADR 0039 (contrato §3): procedimentos do atendimento, só com procedure_billing ligado.
      json[:procedures] = Billing::Json.attached(PerformedProcedure.where(attendance_id: a.id)) if Billing::Gate.enabled?(Current.city)
      json
    end
  ```

  Em `app/services/consultations/json.rb`, no método `consultation`, antes da linha `json[:signature] = blocks[[ "Consultation", c.id ]] if c.finalized?`:

  ```ruby
        # ADR 0039 (contrato §3): procedimentos da consulta, só com procedure_billing ligado.
        json[:procedures] = Billing::Json.attached(PerformedProcedure.where(consultation_id: c.id)) if Billing::Gate.enabled?(Current.city)
  ```

- [ ] **Step 4: Rode:** `rspec spec/requests/billing/procedures_spec.rb spec/requests/billing/lookups_spec.rb spec/requests/billing/sigtap_spec.rb spec/services/billing/attached_json_spec.rb spec/requests/consultations_spec.rb spec/requests/attendances_spec.rb` → verde. (Se `spec/requests/attendances_spec.rb` não existir com esse nome, rode `rspec spec/requests --pattern '*attendance*'`.)

- [ ] **Step 5: Commit:**

  ```bash
  git add app/controllers/billing/base_controller.rb app/controllers/billing/procedures_controller.rb app/controllers/billing/sigtap_controller.rb app/controllers/billing/lookups_controller.rb app/services/billing/access.rb app/services/billing/json.rb app/services/billing/labels.rb app/controllers/attendances_controller.rb app/services/consultations/json.rb config/routes.rb spec/requests/billing/procedures_spec.rb spec/requests/billing/lookups_spec.rb spec/requests/billing/sigtap_spec.rb spec/services/billing/attached_json_spec.rb
  git commit -m "feat: add performed procedure routes and SIGTAP search for billing

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

---

## Fatia 3 — Laboratório municipal

### Task 7: Produção do laboratório municipal no faturamento (F-22.3)

**Files:**
- Create: `app/commands/billing/from_lab.rb`, `app/jobs/billing/lab_production_job.rb`, `spec/commands/billing/from_lab_spec.rb`, `spec/jobs/billing/lab_production_job_spec.rb`
- Modify: `config/initializers/domain_events.rb` (tirar `lab_report.released` do laço do 21b e ligá-lo ao consumidor), `spec/initializers/domain_events_bindings_spec.rb` (lista do 21b e o teste novo)

**Interfaces:**
- Consumes: `Billing::Record.insert!` (Task 5); `Billing::Establishment.for_lab`, `Billing::Staff.cbo_for` (Task 4); `Billing::Gate.enabled?` (Task 2); `Lab::Release.call`, `lab_*` helpers do 21b; `IdempotentConsumer`.
- Produces: `Billing::FromLab.call(report) -> Integer` (procedimentos criados); `Billing::LabProductionJob` (consumidor de `lab_report.released`, `handle(lab_report_id:, **)`).

- [ ] **Step 1: Teste que falha** — `spec/commands/billing/from_lab_spec.rb`:

  ```ruby
  require "rails_helper"

  # Contrato §4 (F-22.3): cada item do PRIMEIRO laudo liberado do laboratório
  # municipal vira procedimento (source lab_report, estabelecimento =
  # laboratório, profissional = liberador com o CBO do vínculo dele, data =
  # coleta, quantidade 1). Retificação não cria; idempotente por item;
  # liberador sem vínculo = pendência de CBO.
  RSpec.describe Billing::FromLab do
    before { billing_city!(lab: true) }

    let(:provider) { lab_provider! }
    let(:lead) { lab_releaser!(provider, lead: true).tap { |user| link_professional!(user, create_unit("UBS #{SecureRandom.hex(2)}"), cbo: "221205") } }
    let(:glucose) { lab_exam!(provider: provider) }

    def released_report(req)
      item = type!(req.items.first, { glicose: "98" })
      result = Lab::Release.call(requisition: req, item_ids: [ item.id ], signature: "advanced", totp: totp_for(lead), by: lead)
      raise "liberação recusada: #{result.reason}" if result.failure?

      result.payload[:report]
    end

    it "cria um procedimento por item, faturável em BPA-C, com a data da coleta" do
      req = received_requisition!(provider: provider, exams: [ glucose ])
      report = released_report(req)
      expect(described_class.call(report)).to eq(1)
      procedure = PerformedProcedure.find_by!(lab_report_item_id: report.report_items.first.id)
      collected = req.samples.first.reload.collected_at.in_time_zone.to_date
      expect(procedure).to have_attributes(source: "lab_report", exam_provider_id: provider.id, health_unit_id: nil, patient_id: req.patient_id,
                                           professional_user_id: lead.id, cbo_code: "221205", sigtap_code: "0202010473", quantity: 1,
                                           performed_on: collected, instrument: "bpa_c", billable: true, recorded_by_user_id: nil)
      expect(procedure.billing_period).to have_attributes(exam_provider_id: provider.id, status: "open")
    end

    it "idempotente por item; laudo que retifica outro, retirado ou pendente de assinatura não cria" do
      req = received_requisition!(provider: provider, exams: [ glucose ])
      report = released_report(req)
      described_class.call(report)
      expect(described_class.call(report)).to eq(0)
      # O caminho real da retificação é o do 21b (rectification_spec); aqui basta a forma do laudo novo.
      expect(described_class.call(report.dup.tap { |r| r.replaces_report_id = report.id })).to eq(0)
      expect(described_class.call(report.dup.tap { |r| r.status = "withdrawn" })).to eq(0)
      expect(described_class.call(report.dup.tap { |r| r.status = "pending_signature" })).to eq(0)
      expect(PerformedProcedure.where(exam_provider_id: provider.id).count).to eq(1)
    end

    it "liberador sem vínculo: grava com a pendência de CBO" do
      deputy = lab_releaser!(provider, deputy: true)
      req = received_requisition!(provider: provider, exams: [ glucose ])
      item = type!(req.items.first, { glicose: "98" })
      lead # o RT precisa existir para liberar
      report = Lab::Release.call(requisition: req, item_ids: [ item.id ], signature: "advanced", totp: totp_for(deputy), by: deputy).payload[:report]
      described_class.call(report)
      expect(PerformedProcedure.find_by!(lab_report_item_id: report.report_items.first.id))
        .to have_attributes(cbo_code: nil, billable: false, blocking_reasons: %w[missing_professional_cbo])
    end
  end
  ```

  `spec/jobs/billing/lab_production_job_spec.rb`:

  ```ruby
  require "rails_helper"

  # A liberação publica lab_report.released; o consumidor (idempotente por
  # evento e por item) chama Billing::FromLab só com procedure_billing ligado.
  RSpec.describe Billing::LabProductionJob, type: :job do
    before { billing_city!(lab: true) }

    let(:provider) { lab_provider! }
    let(:lead) { lab_releaser!(provider, lead: true).tap { |user| link_professional!(user, create_unit("UBS #{SecureRandom.hex(2)}"), cbo: "221205") } }

    def report!
      req = received_requisition!(provider: provider, exams: [ lab_exam!(provider: provider) ])
      item = type!(req.items.first, { glicose: "98" })
      Lab::Release.call(requisition: req, item_ids: [ item.id ], signature: "advanced", totp: totp_for(lead), by: lead).payload[:report]
    end

    def args(report, event_id: SecureRandom.uuid)
      { event_id: event_id, event_name: "lab_report.released", city_slug: Current.city.slug, payload: { "lab_report_id" => report.id } }
    end

    it "o evento da liberação está ligado ao consumidor" do
      expect(DomainEvents.registry["lab_report.released"].map(&:job)).to eq([ "Billing::LabProductionJob" ])
    end

    it "cria a produção; o mesmo evento de novo não duplica" do
      report = report!
      payload = args(report)
      described_class.new.perform(**payload)
      described_class.new.perform(**payload)
      expect(PerformedProcedure.where(exam_provider_id: provider.id, source: "lab_report").count).to eq(1)
    end

    it "com o interruptor desligado, nada" do
      report = report!
      Platform::Features.set!(city: Current.city, key: "procedure_billing", enabled: false, maintainer: ledi_maintainer!)
      described_class.new.perform(**args(report))
      expect(PerformedProcedure.count).to eq(0)
    end
  end
  ```

  Em `spec/initializers/domain_events_bindings_spec.rb`, no `describe "municipal lab event bindings (ADR 0037)"`, tire `lab_report.released` da lista `names` e acrescente, logo depois do `it`:

  ```ruby
    # ADR 0039: a liberação do laudo vira produção do laboratório municipal.
    it "binds lab_report.released only to the billing consumer" do
      expect(DomainEvents.registry["lab_report.released"].map(&:job)).to eq([ "Billing::LabProductionJob" ])
    end
  ```

- [ ] **Step 2: Rode e veja falhar:** `rspec spec/commands/billing/from_lab_spec.rb spec/jobs/billing/lab_production_job_spec.rb spec/initializers/domain_events_bindings_spec.rb` → `NameError: uninitialized constant Billing::FromLab`.

- [ ] **Step 3: Implemente.** `app/commands/billing/from_lab.rb`:

  ```ruby
  # Produção do laboratório municipal (ADR 0039; contrato §4; plano do módulo
  # 22, Desvio 10): um procedimento por item do PRIMEIRO laudo liberado
  # (replaces_report_id nulo; retificação e laudo retirado não criam), com o
  # laboratório como estabelecimento, o liberador como profissional (CBO do
  # vínculo dele), a data da coleta e quantidade 1. Idempotente por item
  # (consulta + índice único em lab_report_item_id).
  module Billing
    module FromLab
      module_function

      def call(report)
        return 0 unless report.status == "released" && report.replaces_report_id.nil?

        provider = report.exam_provider
        return 0 unless provider.uses_municipal_lab

        establishment = Establishment.for_lab(provider)
        releaser = report.released_by_user
        cbo = Staff.cbo_for(releaser, establishment)
        items = report.report_items.includes(lab_requisition_item: [ :lab_sample, :lab_requisition, { lab_exam_version: :lab_exam } ]).order(:id)
        items.count do |report_item|
          next false if PerformedProcedure.exists?(lab_report_item_id: report_item.id)

          item = report_item.lab_requisition_item
          Record.insert!(patient: item.lab_requisition.patient, establishment: establishment, professional: releaser, cbo_code: cbo,
                         source: "lab_report", performed_on: (item.lab_sample.collected_at || report.released_at).in_time_zone.to_date,
                         sigtap_code: item.lab_exam_version.lab_exam.sigtap_code, quantity: 1, cid10: nil, character: "elective",
                         recorded_by: nil, lab_report_item: report_item)
          true
        end
      end
    end
  end
  ```

  `app/jobs/billing/lab_production_job.rb`:

  ```ruby
  # Consumidor de lab_report.released (ADR 0039; contrato §4). Roda na
  # transação da cidade (IdempotentConsumer); o portão lê só a plataforma
  # (Billing::Gate, sem abrir a cidade), por isso é seguro aqui. Desligado no
  # momento da liberação: nada (sem retroativo; runbook).
  module Billing
    class LabProductionJob < ApplicationJob
      include IdempotentConsumer
      queue_as :default

      def handle(lab_report_id:, **)
        return unless Gate.enabled?(Current.city)

        report = LabReport.find_by(id: lab_report_id)
        FromLab.call(report) if report
      end
    end
  end
  ```

  Em `config/initializers/domain_events.rb`, no laço do 21b (`%w[lab_requisition.opened … exam_order.gal_registered].each { |name| DomainEvents.bind name, to: [] }`), **tire** `lab_report.released` da lista e acrescente, logo depois do laço:

  ```ruby
    # ADR 0039 (contrato §4): a liberação do PRIMEIRO laudo vira produção do
    # laboratório municipal (Billing::FromLab; idempotente por item).
    DomainEvents.bind "lab_report.released", to: [ Billing::LabProductionJob ]
  ```

- [ ] **Step 4: Rode:** `rspec spec/commands/billing/from_lab_spec.rb spec/jobs/billing/lab_production_job_spec.rb spec/initializers/domain_events_bindings_spec.rb spec/requests/lab/release_spec.rb spec/requests/lab/rectification_spec.rb` → verde.

- [ ] **Step 5: Commit e card:**

  ```bash
  git add app/commands/billing/from_lab.rb app/jobs/billing/lab_production_job.rb config/initializers/domain_events.rb spec/commands/billing/from_lab_spec.rb spec/jobs/billing/lab_production_job_spec.rb spec/initializers/domain_events_bindings_spec.rb
  git commit -m "feat: turn municipal lab released reports into billable production

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```
  F-22.3 só tem o api: mova o card para **Done** (`gh project item-edit …`, comando começando com `gh`).

---

## Fatia 4 — Arquivo, fechamento e configuração

### Task 8: Gerador do arquivo BPA (`Billing::BpaFile`) e testes golden (F-22.6)

**Files:**
- Create: `app/services/billing/bpa_file.rb`, `spec/services/billing/bpa_file_spec.rb`, `spec/fixtures/billing/bpa-c-21-lines.txt`, `spec/fixtures/billing/bpa-i-25-lines.txt`, `spec/fixtures/billing/bpa-mixed.txt`, `spec/fixtures/billing/SHA256SUMS`

**Interfaces:**
- Consumes: os exemplos golden da Task 1 (cópia byte a byte).
- Produces:
  - `Billing::BpaFile::Header = Data.define(:competence, :origin_name, :origin_acronym, :origin_cnpj, :destination_body, :management)`;
  - `Billing::BpaFile::CLine = Data.define(:cnes, :competence, :cbo, :procedure, :age, :quantity)`;
  - `Billing::BpaFile::ILine = Data.define(:key, :cnes, :competence, :professional_cns, :cbo, :performed_on, :procedure, :sex, :ibge, :cid10, :age, :quantity, :character, :name, :birth_date, :race_color, :ethnicity, :zip, :street, :number, :complement, :district, :phone, :cpf)` (raça/cor `01`..`05`; etnia só no `05`; endereço do cadastro do paciente, D-U1) (`performed_on`/`birth_date` como `Date`; `character` já no código do arquivo `"01"`/`"02"`; opcionais `nil`);
  - `Billing::BpaFile::Built = Data.define(:bytes, :control, :lines_bpa_c, :lines_bpa_i, :sheets)`;
  - `Billing::BpaFile.build(header:, c_lines:, i_lines:) -> Built` (levanta `Billing::BpaFile::TooManySheets` acima de 999 folhas; `ArgumentError` sem valor na mensagem se um campo não cabe);
  - `Billing::BpaFile::CHARACTER = { "elective" => "01", "urgent" => "02" }`, `SHEET_LINES = 20`, `MAX_SHEETS = 999`.

- [ ] **Step 1: Copie os golden** do repo `contracts` (Task 1) e confira:

  ```bash
  mkdir -p apps/api/.claude/mod22/spec/fixtures/billing
  cp contracts/billing/examples/bpa-c-21-lines.txt contracts/billing/examples/bpa-i-25-lines.txt contracts/billing/examples/bpa-mixed.txt contracts/billing/examples/SHA256SUMS apps/api/.claude/mod22/spec/fixtures/billing/
  (cd apps/api/.claude/mod22/spec/fixtures/billing && shasum -a 256 -c SHA256SUMS)
  ```
  Expected: `bpa-c-21-lines.txt: OK`, `bpa-i-25-lines.txt: OK`, `bpa-mixed.txt: OK`. Confira no `git` do worktree que nenhum editor converteu `\r\n` (`git -c core.autocrlf=false add` no commit; `file spec/fixtures/billing/*.txt` diz "with CRLF line terminators").

- [ ] **Step 2: Teste que falha** — `spec/services/billing/bpa_file_spec.rb`:

  ```ruby
  require "rails_helper"

  # Layout do BPA Magnético de 09/07/2026 (contracts/billing/bpa-interface.md):
  # golden byte a byte contra os exemplos gerados pelo oráculo independente do
  # repo contracts (mesmas entradas, montadas aqui em Ruby); folhas de 20;
  # campo de controle; só ASCII; linhas de tamanho fixo; recusa acima de 999 folhas.
  RSpec.describe Billing::BpaFile do
    FIXTURES = Rails.root.join("spec/fixtures/billing")
    HEADER = Billing::BpaFile::Header.new(competence: "202609", origin_name: "SMS Curitiba", origin_acronym: nil, origin_cnpj: "11222333000181",
                                          destination_body: "Secretaria Municipal de Saúde", management: "M")
    C_CODES = %w[0202010473 0202020380 0202010317].freeze
    PROFESSIONALS = [ %w[798312812760829 225125], %w[765218542269265 223505] ].freeze
    NAMES = [ "Maria Aparecida da Silva", "José Antônio Pereira", "Ana Lúcia D'Ávila Gonçalves", "João Batista Conceição",
              "Luíza Helena de Araújo Fontes Bittencourt" ].freeze
    CPFS = %w[29136237191 66012407106 99378987656 93842327501 96273885648 87159190726 06446750714 28441315817 74747239723
              93161173481 18425542049 10271688769 28258981714 38263611130 23441850114 22458158749 37499727378 18615318263
              05596308676 42719726737 02622189095 19833011594 74018474776 33531950410 79387458989].freeze

    def c_lines
      (0...21).map do |i|
        described_class::CLine.new(cnes: "1234567", competence: "202609", cbo: "221205", procedure: C_CODES[i % 3], age: 20 + i, quantity: i + 1)
      end
    end

    def i_lines
      (0...25).map do |j|
        on = Date.new(2026, 9, 1 + (j % 28))
        born = Date.new(1950 + j, 1 + (j % 12), 1 + (j % 27))
        cns, cbo = j < 22 ? PROFESSIONALS[0] : PROFESSIONALS[1]
        described_class::ILine.new(key: format("i%02d", j), cnes: "7654321", competence: "202609", professional_cns: cns, cbo: cbo,
                                   performed_on: on, procedure: j < 22 ? "0301010072" : "0301010048", sex: j.even? ? "F" : "M",
                                   ibge: "410690", cid10: (j % 3).zero? ? "I10" : nil, age: Citizen.age_between(born, on), quantity: 1,
                                   character: (j % 4).zero? ? "02" : "01", name: NAMES[j % 5], birth_date: born,
                                   race_color: %w[01 02 03 04 05][j % 5], ethnicity: j % 5 == 4 ? "0001" : nil,
                                   zip: j.even? ? "80010000" : "81020130", street: j.even? ? "Rua XV de Novembro" : "Av. Sete de Setembro, 2.775",
                                   number: (j % 3).zero? ? "SN" : "1200", complement: j % 4 == 1 ? "Apto 12" : nil,
                                   district: j.even? ? "Centro" : "Rebouças", phone: (j % 3).zero? ? nil : format("4199%07d", j), cpf: CPFS[j])
      end
    end

    def golden(name) = FIXTURES.join(name).binread

    it "BPA-C: 21 linhas em duas folhas (20 + 1), igual ao golden byte a byte" do
      built = described_class.build(header: HEADER, c_lines: c_lines.shuffle(random: Random.new(7)), i_lines: [])
      expect(built.bytes).to eq(golden("bpa-c-21-lines.txt"))
      expect(built).to have_attributes(control: 1581, lines_bpa_c: 21, lines_bpa_i: 0, sheets: 2)
    end

    it "BPA-I: folha nova a cada profissional, 20 por folha; acentos, apóstrofo e nome longo; igual ao golden" do
      built = described_class.build(header: HEADER, c_lines: [], i_lines: i_lines.reverse)
      expect(built.bytes).to eq(golden("bpa-i-25-lines.txt"))
      expect(built).to have_attributes(control: 2131, lines_bpa_i: 25, sheets: 3)
      lines = built.bytes.split("\r\n")
      expect(lines[1][44, 5]).to eq("00101")   # folha 001, linha 01 (o profissional de CNS menor)
      expect(lines[4][44, 5]).to eq("00201")
      expect(lines[24][44, 5]).to eq("00301")
      expect(lines.map { |l| l[112, 30] }.uniq).to include("LUIZA HELENA DE ARAUJO FONTES ", "ANA LUCIA D AVILA GONCALVES   ")
    end

    it "BPA-C e BPA-I no mesmo arquivo: uma série de folhas; igual ao golden" do
      built = described_class.build(header: HEADER, c_lines: c_lines.first(2), i_lines: i_lines.first(2))
      expect(built.bytes).to eq(golden("bpa-mixed.txt"))
      expect(built).to have_attributes(control: 1617, sheets: 2)
    end

    it "só ASCII imprimível, CR+LF em todo registro e tamanho fixo por tipo" do
      bytes = described_class.build(header: HEADER, c_lines: c_lines, i_lines: i_lines).bytes
      expect(bytes.bytes.reject { |b| b == 13 || b == 10 }).to all(be_between(0x20, 0x7E))
      expect(bytes).to end_with("\r\n")
      lengths = bytes.split("\r\n").group_by { |l| l[0, 2] }.transform_values { |ls| ls.map(&:length).uniq }
      expect(lengths).to eq("01" => [ 130 ], "02" => [ 48 ], "03" => [ 351 ])
    end

    it "campo de controle sempre em 1111..2221 (resto zero dá 1111)" do
      line = described_class::CLine.new(cnes: "1234567", competence: "202609", cbo: "221205", procedure: "0000001110", age: 30, quantity: 1)
      expect(described_class.build(header: HEADER, c_lines: [ line ], i_lines: []).control).to eq(1111)
      line = line.with(quantity: 1110)
      expect(described_class.build(header: HEADER, c_lines: [ line ], i_lines: []).control).to eq(2220)
    end

    it "999 folhas cabem; 1000 recusam (nunca corta)" do
      many = ->(n) { Array.new(n) { |i| described_class::CLine.new(cnes: "1234567", competence: "202609", cbo: "221205", procedure: "0202010473", age: i % 131, quantity: 1) } }
      expect(described_class.build(header: HEADER, c_lines: many.call(999 * 20), i_lines: []).sheets).to eq(999)
      expect { described_class.build(header: HEADER, c_lines: many.call(999 * 20 + 1), i_lines: []) }.to raise_error(described_class::TooManySheets)
    end

    it "campo que não cabe levanta ArgumentError sem o valor na mensagem" do
      bad = c_lines.first.with(cnes: "12345678")
      expect { described_class.build(header: HEADER, c_lines: [ bad ], i_lines: []) }.to raise_error(ArgumentError) { |e| expect(e.message).not_to include("12345678") }
    end
  end
  ```
  (O teste do controle com quantidade 1110: `(1110 + 1110) % 1111 + 1111 = 1109 + 1111 = 2220`. O gerador desta task foi rodado contra os três golden ao escrever o plano: iguais byte a byte.)

- [ ] **Step 3: Rode e veja falhar:** `rspec spec/services/billing/bpa_file_spec.rb` → `NameError: uninitialized constant Billing::BpaFile`.

- [ ] **Step 4: Implemente** `app/services/billing/bpa_file.rb`:

  ```ruby
  # Arquivo de interface do BPA Magnético (ADR 0039; contracts/billing/bpa-interface.md;
  # Layout_Exportacao_BPA.pdf do DATASUS de 09/07/2026). Função pura: recebe o
  # cabeçalho e as linhas já montadas (Billing::Lines) e devolve os bytes ASCII
  # com CR+LF. Folhas numa série só: BPA-C primeiro (CBO, procedimento, idade,
  # competência), depois BPA-I com folha nova a cada profissional (CNS + CBO),
  # 20 linhas por folha, até 999. Campo de controle: Σ(procedimento +
  # quantidade) mod 1111 + 1111. Mensagem de erro nunca cita o valor do campo.
  module Billing
    module BpaFile
      SHEET_LINES = 20
      MAX_SHEETS = 999
      ORIGIN = "BPA".freeze
      VERSION = "ROTASAUDE".freeze
      WITHOUT_CPF = "N".freeze
      CRLF = "\r\n".freeze
      CHARACTER = { "elective" => "01", "urgent" => "02" }.freeze
      LENGTHS = { "01" => 130, "02" => 48, "03" => 351 }.freeze

      class TooManySheets < StandardError; end

      Header = Data.define(:competence, :origin_name, :origin_acronym, :origin_cnpj, :destination_body, :management)
      CLine = Data.define(:cnes, :competence, :cbo, :procedure, :age, :quantity)
      ILine = Data.define(:key, :cnes, :competence, :professional_cns, :cbo, :performed_on, :procedure, :sex, :ibge, :cid10, :age, :quantity,
                          :character, :name, :birth_date, :race_color, :ethnicity, :zip, :street, :number, :complement, :district, :phone,
                          :cpf)
      Built = Data.define(:bytes, :control, :lines_bpa_c, :lines_bpa_i, :sheets)

      module_function

      def build(header:, c_lines:, i_lines:)
        c_sheets = c_lines.sort_by { |l| [ l.cbo, l.procedure, l.age, l.competence ] }.each_slice(SHEET_LINES).to_a
        i_sheets = i_lines.sort_by { |l| [ l.professional_cns, l.cbo, l.performed_on, l.procedure, l.key ] }
                          .chunk_while { |a, b| [ a.professional_cns, a.cbo ] == [ b.professional_cns, b.cbo ] }
                          .flat_map { |group| group.each_slice(SHEET_LINES).to_a }
        sheet = c_sheets.size + i_sheets.size
        raise TooManySheets if sheet > MAX_SHEETS

        rows = []
        c_sheets.each.with_index(1) { |slice, number| slice.each.with_index(1) { |line, seq| rows << c_row(line, number, seq) } }
        i_sheets.each.with_index(c_sheets.size + 1) { |slice, number| slice.each.with_index(1) { |line, seq| rows << i_row(line, number, seq) } }
        control = (c_lines + i_lines).sum { |l| l.procedure.to_i + l.quantity } % 1111 + 1111
        text = header_row(header, rows.size, sheet, control) + CRLF + rows.map { |row| row + CRLF }.join
        Built.new(bytes: text.encode(Encoding::US_ASCII).b, control: control, lines_bpa_c: c_lines.size, lines_bpa_i: i_lines.size, sheets: sheet)
      end

      def header_row(h, lines, sheets, control)
        fixed("01", "01" + "#BPA#" + num(h.competence, 6) + num(lines, 6) + num(sheets, 6) + num(control, 4) + text(h.origin_name, 30) +
                    text(h.origin_acronym, 6) + num(h.origin_cnpj, 14) + text(h.destination_body, 40) + alfa(h.management, 1) + alfa(VERSION, 10))
      end

      def c_row(l, sheet, seq)
        fixed("02", "02" + num(l.cnes, 7) + num(l.competence, 6) + alfa(l.cbo, 6) + num(sheet, 3) + num(seq, 2) + num(l.procedure, 10) +
                    num(l.age, 3) + num(l.quantity, 6) + alfa(ORIGIN, 3))
      end

      def i_row(l, sheet, seq)
        fixed("03", "03" + num(l.cnes, 7) + num(l.competence, 6) + num(l.professional_cns, 15) + alfa(l.cbo, 6) +
                    l.performed_on.strftime("%Y%m%d") + num(sheet, 3) + num(seq, 2) + num(l.procedure, 10) +
                    blank(15) +                                              # CNS do paciente: nunca junto do CPF
                    alfa(l.sex, 1) + opt_num(l.ibge, 6) + alfa(l.cid10, 4) + num(l.age, 3) + num(l.quantity, 6) +
                    num(l.character, 2) + blank(13) + alfa(ORIGIN, 3) +       # caráter, autorização, origem
                    text(l.name, 30) + l.birth_date.strftime("%Y%m%d") + num(l.race_color, 2) + opt_num(l.ethnicity, 4) +
                    blank(3) + blank(3) + blank(3) + blank(8) + blank(4) + blank(14) +   # nac., serviço, classif., equipe, CNPJ OPM
                    num(l.zip, 8) + blank(3) + address(l.street, 30) + address(l.complement, 10) + address(l.number, 5) +
                    address(l.district, 30) + opt_alfa_digits(l.phone, 11) + blank(40) + # bairro, telefone, e-mail
                    blank(10) + num(l.cpf, 11) + blank(1) + WITHOUT_CPF)                  # INE, CPF, situação de rua, sem CPF
      end

      def num(value, size)
        digits = value.to_s
        raise ArgumentError, "campo numérico fora do layout (#{size} posições)" unless digits.match?(/\A\d{1,#{size}}\z/)

        digits.rjust(size, "0")
      end

      def opt_num(value, size) = value.blank? ? blank(size) : num(value, size)

      def alfa(value, size)
        chars = value.to_s
        raise ArgumentError, "campo alfanumérico fora do layout (#{size} posições)" if chars.length > size || !chars.match?(/\A[ -~]*\z/)

        chars.ljust(size)
      end

      def opt_alfa_digits(value, size)
        return blank(size) if value.blank?
        raise ArgumentError, "campo numérico fora do layout (#{size} posições)" unless value.to_s.match?(/\A\d{1,#{size}}\z/)

        value.to_s.ljust(size)
      end

      # Maiúsculas ASCII, só letras e espaço, espaços colapsados, cortado no tamanho do campo (Desvio 6).
      def text(value, size)
        plain = ActiveSupport::Inflector.transliterate(value.to_s, " ").upcase.gsub(/[^A-Z ]/, " ").squish
        plain[0, size].ljust(size)
      end

      # Endereço (D-U1): como `text`, mas com dígitos ("RUA 7 DE SETEMBRO", "APTO 12", "1200", "SN").
      def address(value, size)
        plain = ActiveSupport::Inflector.transliterate(value.to_s, " ").upcase.gsub(/[^A-Z0-9 ]/, " ").squish
        plain[0, size].ljust(size)
      end

      def blank(size) = " " * size

      def fixed(kind, row)
        raise ArgumentError, "registro #{kind} com #{row.length} posições" unless row.length == LENGTHS.fetch(kind)

        row
      end
      private_class_method :header_row, :c_row, :i_row, :num, :opt_num, :alfa, :opt_alfa_digits, :text, :address, :blank, :fixed
    end
  end
  ```

- [ ] **Step 5: Rode:** `rspec spec/services/billing/bpa_file_spec.rb` → verde. Diferença no golden: `cmp -l` entre o arquivo gerado (grave `built.bytes` num tmp) e o golden mostra a posição; corrija o **gerador**, nunca o golden (o golden é o oráculo da Task 1).

- [ ] **Step 6: Commit:**

  ```bash
  git -c core.autocrlf=false add app/services/billing/bpa_file.rb spec/services/billing/bpa_file_spec.rb spec/fixtures/billing/bpa-c-21-lines.txt spec/fixtures/billing/bpa-i-25-lines.txt spec/fixtures/billing/bpa-mixed.txt spec/fixtures/billing/SHA256SUMS
  git commit -m "feat: generate BPA Magnetico interface file with golden tests

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

---

### Task 8b: Raça/cor e endereço do paciente — recepção, faturista e exclusão (D-U1)

Decisão do usuário (contrato §12, D-U1, docs `fdb2c98`): o BPA-I leva raça/cor (sem `99`), etnia (indígena) e o endereço do **cadastro do paciente**; a recepção preenche no check-in/cadastro, o faturista corrige; a falta vira pendência no fechamento (`missing_race_color`, `missing_address`, `fix: "patient"` — Tasks 4 e 9). As colunas nascem na migração da Task 3. O município de residência continua o da cidade.

**Files:**
- Create: `app/services/patients/demographics.rb`, `app/controllers/billing/patients_controller.rb`, `spec/services/patients/demographics_spec.rb`, `spec/requests/billing/patient_demographics_spec.rb`, `spec/requests/attendance_demographics_spec.rb`, `spec/commands/citizens/erase_demographics_spec.rb`
- Modify: `app/controllers/attendance_controller.rb` (`verify`, `names`, `ERROR_STATUS`), `app/commands/citizens/erase.rb` (paciente não retido perde os campos), `config/routes.rb` (escopo `/billing`), `config/initializers/domain_events.rb` e `spec/initializers/domain_events_bindings_spec.rb` (dois eventos), `config/initializers/filter_parameter_logging.rb`

**Interfaces:**
- Consumes: `Patient::DEMOGRAPHICS`, `Patient::RACE_COLORS` (Task 3); `Patients::Resolve.call(citizen)`; `Billing::BaseController`, `Billing::Access` (Task 6).
- Produces:
  - `Patients::Demographics.parse(params) -> Result` (`ok(values: { "campo" => valor | nil })` só com as chaves presentes; falhas `invalid_race_color`, `invalid_ethnicity`, `invalid_zip`, `too_long`, `invalid_demographics`, com `details: { field: }`);
  - `Patients::Demographics.save!(patient, values) -> Result` (`ok(patient:)`; `ethnicity_required` quando a raça final é `05` sem etnia; `invalid_ethnicity` com etnia fora do `05`);
  - `Patients::Demographics.flags(patient) -> { race_color_set:, address_set: }` (o que a tela pode ver sem o dado);
  - `PATCH /billing/patients/:id/demographics` (só `billing_clerk`; evento `billing_patient.demographics_updated` com `patient_id`, `user_id`);
  - `POST /attendance/verifications` e `POST /attendance/verifications/:id/names` aceitam `race_color`, `ethnicity`, `address_zip`, `address_street`, `address_number`, `address_complement`, `address_district` (opcionais; evento `patient.demographics_updated` com `patient_id`, `user_id`).

**Valores fixados:** raça/cor `01` branca, `02` preta, `03` parda, `04` amarela, `05` indígena (sem `99`); etnia 4 dígitos (tabela da Portaria SAS 508/2010), só com `05` e obrigatória com `05`; CEP 8 dígitos (aceita `80010-000`); endereço até 30, número até 5 (`SN` sem número), complemento até 10, bairro até 30 — larguras do layout, recusadas (`too_long`), nunca cortadas; `""`/`null` apaga o campo.

- [ ] **Step 1: Teste que falha** — `spec/services/patients/demographics_spec.rb`:

  ```ruby
  require "rails_helper"

  # D-U1 (contrato §12): validação dos campos do BPA-I no cadastro do
  # paciente — códigos do layout sem 99, etnia só e sempre com indígena,
  # larguras do layout recusadas (nunca cortadas), só as chaves presentes.
  RSpec.describe Patients::Demographics do
    before { clinical_city! }

    let(:patient) { exam_patient!(exam_seq!).first }

    it "aceita os campos válidos, normaliza CEP e espaços, e só mexe nas chaves presentes" do
      parsed = described_class.parse("race_color" => "03", "address_zip" => "80010-000", "address_street" => "  Rua XV   de Novembro ",
                                     "address_number" => "SN")
      expect(parsed.payload[:values]).to eq("race_color" => "03", "address_zip" => "80010000", "address_street" => "Rua XV de Novembro",
                                            "address_number" => "SN")
      described_class.save!(patient, parsed.payload[:values])
      described_class.save!(patient, described_class.parse("address_district" => "Centro").payload[:values])
      expect(patient.reload).to have_attributes(race_color: "03", address_zip: "80010000", address_district: "Centro")
      expect(described_class.flags(patient)).to eq(race_color_set: true, address_set: true)
      described_class.save!(patient, described_class.parse("address_district" => "").payload[:values])
      expect(described_class.flags(patient.reload)).to eq(race_color_set: true, address_set: false)
    end

    it "recusa com o campo: 99, CEP, larguras, tipo; etnia × raça" do
      {
        { "race_color" => "99" } => [ :invalid_race_color, "race_color" ], { "address_zip" => "8001000" } => [ :invalid_zip, "address_zip" ],
        { "address_street" => "R" * 31 } => [ :too_long, "address_street" ], { "address_number" => "123456" } => [ :too_long, "address_number" ],
        { "address_complement" => "C" * 11 } => [ :too_long, "address_complement" ], { "ethnicity" => "12" } => [ :invalid_ethnicity, "ethnicity" ],
        { "race_color" => 3 } => [ :invalid_demographics, "race_color" ]
      }.each do |params, (reason, field)|
        result = described_class.parse(params)
        expect([ result.reason, result.details[:field] ]).to eq([ reason, field ]), params.inspect
      end
      expect(described_class.save!(patient, "race_color" => "05").reason).to eq(:ethnicity_required)
      expect(described_class.save!(patient, "race_color" => "03", "ethnicity" => "0001").reason).to eq(:invalid_ethnicity)
      expect(described_class.save!(patient, "race_color" => "05", "ethnicity" => "0001")).to be_ok
      expect(patient.reload.race_color).to eq("05")
    end
  end
  ```

  `spec/requests/billing/patient_demographics_spec.rb`:

  ```ruby
  require "rails_helper"

  # D-U1: o faturista corrige raça/cor e endereço do paciente pendente; trilha
  # só com ids; a pendência some no fechamento seguinte.
  RSpec.describe "Faturamento — dados do paciente para o BPA-I", type: :request do
    before do
      billing_city!
      billing_settings!
    end

    let(:unit) { upa! }
    let(:patient) { billing_patient!(demographics: false) }
    let(:clerk) { billing_clerk! }

    it "faturista corrige; a pendência some; outros papéis 403; inexistente 404; inválido 422" do
      procedure = recorded_procedure!(unit: unit, professional: doctor!(unit), patient: patient)
      expect(Billing::Pending.call(procedure.billing_period).sole).to include(reasons: %w[missing_race_color missing_address], fix: "patient")
      sign_in_as(lab_admin!)
      json_patch "/billing/patients/#{patient.id}/demographics", race_color: "03"
      expect(response).to have_http_status(:forbidden)
      sign_in_as(clerk)
      json_patch "/billing/patients/#{SecureRandom.uuid}/demographics", race_color: "03"
      expect(response).to have_http_status(:not_found)
      json_patch "/billing/patients/#{patient.id}/demographics", race_color: "99"
      expect(json_body).to eq("error" => "invalid_race_color", "field" => "race_color")
      json_patch "/billing/patients/#{patient.id}/demographics", race_color: "03", address_zip: "80010000", address_street: "Rua XV de Novembro",
                                                                   address_number: "1200", address_district: "Centro"
      expect(response).to have_http_status(:ok)
      expect(json_body).to eq("patient" => { "id" => patient.id, "display_name" => patient.display_name,
                                             "demographics" => { "race_color" => "03", "ethnicity" => nil, "address_zip" => "80010000",
                                                                 "address_street" => "Rua XV de Novembro", "address_number" => "1200",
                                                                 "address_complement" => nil, "address_district" => "Centro" } })
      expect(DomainEvent.where(name: "billing_patient.demographics_updated").last.payload).to eq("patient_id" => patient.id, "user_id" => clerk.id)
      expect(Billing::Pending.call(procedure.billing_period)).to eq([])
      export = Billing::Close.call(period: procedure.billing_period, by: clerk).payload[:export]
      line = export.bytes.split("\r\n").last
      expect([ line[150, 2], line[191, 8], line[202, 30].strip, line[242, 5].strip, line[247, 30].strip ])
        .to eq([ "03", "80010000", "RUA XV DE NOVEMBRO", "1200", "CENTRO" ])
    end
  end
  ```

  `spec/requests/attendance_demographics_spec.rb`:

  ```ruby
  require "rails_helper"

  # D-U1: a recepção grava raça/cor e endereço ao validar o par e ao completar
  # o cadastro de par já validado; opcionais e validados (recusa antes de
  # validar o par); a resposta nunca devolve o endereço.
  RSpec.describe "Recepção — raça/cor e endereço do paciente", type: :request do
    let(:citizen) { Citizen.create!(cpf: "52998224725", phone: "+5541998765432") }
    let(:fields) do
      { race_color: "05", ethnicity: "0001", address_zip: "80010-000", address_street: "Rua XV de Novembro", address_number: "1200",
        address_district: "Centro" }
    end

    before do
      clinical_city!
      sign_in_as(verifier!)
    end

    def verify!(**extra)
      json_post "/attendance/verifications", { cpf: citizen.cpf, code: issue_code_for(citizen), document_checked: true, birth_date: "1980-05-10",
                                               sex: "female", full_name: "Maria Aparecida da Silva" }.merge(extra)
    end

    it "na validação: grava no paciente do par (resolvido na hora), com trilha só de ids" do
      verify!(**fields)
      expect(response).to have_http_status(:created)
      patient = Patient.find(citizen.reload.patient_id)
      expect(patient).to have_attributes(race_color: "05", ethnicity: "0001", address_zip: "80010000", address_district: "Centro")
      expect(DomainEvent.where(name: "patient.demographics_updated").last.payload.keys).to match_array(%w[patient_id user_id])
      expect(response.body).not_to include("Novembro")
    end

    it "inválido recusa antes de validar o par; sem os campos, nada muda" do
      verify!(race_color: "99")
      expect(json_body).to eq("error" => "invalid_race_color", "field" => "race_color")
      expect(citizen.reload.verification_level).to eq("declared")
      verify!
      expect(response).to have_http_status(:created)
      expect(citizen.reload.patient_id).to be_nil
    end

    it "no cadastro de par já validado: só os campos (sem nomes) também valem" do
      verification = CitizenVerification.create!(citizen: citizen, verified_by_user: verifier!, verified_at: Time.current)
      citizen.update!(verification_level: "verified", profile_source: "verified", birth_date: "1980-05-10", sex: "female",
                      full_name: "Maria Aparecida da Silva")
      json_post "/attendance/verifications/#{verification.id}/names", race_color: "03", address_zip: "80010000", address_street: "Rua XV",
                                                                     address_number: "SN", address_district: "Centro"
      expect(response).to have_http_status(:ok)
      expect(json_body["citizen"]["demographics"]).to eq("race_color_set" => true, "address_set" => true)
      expect(Patient.find(citizen.reload.patient_id).address_number).to eq("SN")
    end
  end
  ```

  `spec/commands/citizens/erase_demographics_spec.rb`:

  ```ruby
  require "rails_helper"

  # D-U1 + ADR 0026: na exclusão confirmada (pessoa sem registro clínico), o
  # paciente perde raça/cor, etnia e endereço junto com o par.
  RSpec.describe Citizens::Erase do
    before { billing_city! }

    it "apaga os campos do paciente não retido" do
      patient, citizen = exam_patient!(exam_seq!)
      patient.update!(BillingHelpers::DEMOGRAPHICS)
      reception = staff_with("rec-#{SecureRandom.hex(2)}@cidade.gov.br", "citizen_verifier")
      request = Citizens::RequestErasure.call(cpf: citizen.cpf, document_checked: true, by: reception).payload[:request]
      expect(request.status).to eq("pending")
      result = described_class.call(request: request, by: staff_with("adm-#{SecureRandom.hex(2)}@cidade.gov.br", "municipal_admin"))
      expect(result.payload[:request].status).to eq("confirmed")
      expect(patient.reload.slice(*Patient::DEMOGRAPHICS).values.compact).to eq([])
    end
  end
  ```

  Em `spec/initializers/domain_events_bindings_spec.rb`, acrescente `billing_patient.demographics_updated patient.demographics_updated` à lista `names` do `describe "billing event bindings (ADR 0039)"`.

- [ ] **Step 2: Rode e veja falhar:** `rspec spec/services/patients/demographics_spec.rb spec/requests/billing/patient_demographics_spec.rb spec/requests/attendance_demographics_spec.rb spec/commands/citizens/erase_demographics_spec.rb` → `NameError: uninitialized constant Patients::Demographics`.

- [ ] **Step 3: Implemente.** `app/services/patients/demographics.rb`:

  ```ruby
  # Raça/cor, etnia e endereço do paciente para o BPA-I (ADR 0039; D-U1 do
  # contrato §12). `parse` valida só as chaves presentes ("" ou nil apaga);
  # `save!` confere etnia × raça no resultado final e grava. Larguras do
  # layout do BPA recusadas, nunca cortadas. Nada daqui vai a log ou evento.
  module Patients
    module Demographics
      LIMITS = { "address_street" => 30, "address_number" => 5, "address_complement" => 10, "address_district" => 30 }.freeze
      ADDRESS = %w[address_zip address_street address_number address_district].freeze

      module_function

      def parse(params)
        values = {}
        Patient::DEMOGRAPHICS.each do |key|
          next unless params.key?(key)

          raw = params[key]
          return Result.fail(:invalid_demographics, details: { field: key }) unless raw.nil? || raw.is_a?(String)

          values[key] = raw.to_s.squish.presence
        end
        if values["race_color"] && !Patient::RACE_COLORS.include?(values["race_color"])
          return Result.fail(:invalid_race_color, details: { field: "race_color" })
        end
        return Result.fail(:invalid_ethnicity, details: { field: "ethnicity" }) if values["ethnicity"] && !values["ethnicity"].match?(/\A\d{4}\z/)

        if values["address_zip"]
          values["address_zip"] = values["address_zip"].gsub(/[\s.-]/, "")
          return Result.fail(:invalid_zip, details: { field: "address_zip" }) unless values["address_zip"].match?(/\A\d{8}\z/)
        end
        LIMITS.each do |key, limit|
          return Result.fail(:too_long, details: { field: key }) if values[key] && values[key].length > limit
        end
        Result.ok(values: values)
      end

      def save!(patient, values)
        final = Patient::DEMOGRAPHICS.to_h { |key| [ key, patient.public_send(key) ] }.merge(values)
        return Result.fail(:ethnicity_required, details: { field: "ethnicity" }) if final["race_color"] == "05" && final["ethnicity"].nil?
        return Result.fail(:invalid_ethnicity, details: { field: "ethnicity" }) if final["ethnicity"] && final["race_color"] != "05"

        patient.update!(values) if values.any?
        Result.ok(patient: patient)
      end

      def flags(patient)
        { race_color_set: patient.race_color.present?, address_set: ADDRESS.all? { |key| patient.public_send(key).present? } }
      end

      def json(patient) = Patient::DEMOGRAPHICS.to_h { |key| [ key, patient.public_send(key) ] }
    end
  end
  ```

  `app/controllers/billing/patients_controller.rb`:

  ```ruby
  # PATCH /billing/patients/:id/demographics (D-U1): o faturista corrige
  # raça/cor e endereço do paciente pendente; trilha só com ids.
  module Billing
    class PatientsController < BaseController
      def demographics
        return forbid unless Access.clerk?(Current.user)

        patient = uuid?(params[:id]) ? Patient.find_by(id: params[:id]) : nil
        return not_found unless patient

        parsed = Patients::Demographics.parse(params.to_unsafe_h.slice(*Patient::DEMOGRAPHICS))
        return render_failure(parsed) if parsed.failure?

        result = ApplicationRecord.transaction do
          patient.lock!
          saved = Patients::Demographics.save!(patient, parsed.payload[:values])
          if saved.ok?
            DomainEvents.publish("billing_patient.demographics_updated", patient_id: patient.id, user_id: Current.user.id)
          end
          saved
        end
        return render_failure(result) if result.failure?

        render json: { patient: { id: patient.id, display_name: patient.display_name, demographics: Patients::Demographics.json(patient) } }
      end
    end
  end
  ```

  Em `config/routes.rb`, no escopo `/billing`: `patch "patients/:id/demographics", to: "patients#demographics"`.

  Em `app/controllers/attendance_controller.rb`:
  - em `ERROR_STATUS`, acrescente `invalid_race_color: :unprocessable_entity, invalid_ethnicity: :unprocessable_entity, ethnicity_required: :unprocessable_entity, invalid_zip: :unprocessable_entity, too_long: :unprocessable_entity, invalid_demographics: :unprocessable_entity`;
  - no início de `verify` e de `names`, antes do comando:

    ```ruby
    demographics = Patients::Demographics.parse(params.to_unsafe_h.slice(*Patient::DEMOGRAPHICS))
    return render_failure(demographics, ERROR_STATUS) if demographics.failure?
    ```
    (o `render_failure` do `AttendanceAccess` já devolve `details`, então o `field` sai junto);
  - em `verify`, depois do `return render_failure(result, ERROR_STATUS) if result.failure?`:

    ```ruby
    saved = save_demographics(v.citizen, demographics.payload[:values])
    return render_failure(saved, ERROR_STATUS) if saved&.failure?
    ```
  - troque o corpo de `names` (depois do `find_by` e do 404) por:

    ```ruby
    names_sent = %w[full_name social_name mother_name].any? { |key| params.key?(key) }
    if names_sent || demographics.payload[:values].empty?
      result = Citizens::CompleteNames.call(verification: verification, full_name: params[:full_name],
                                            social_name: params[:social_name], mother_name: params[:mother_name],
                                            by: Current.user)
      return render_failure(result, ERROR_STATUS) if result.failure?
    end
    return render(json: { error: "already_revoked" }, status: :conflict) unless verification.reload.active?

    citizen = verification.citizen.reload
    saved = save_demographics(citizen, demographics.payload[:values])
    return render_failure(saved, ERROR_STATUS) if saved&.failure?

    patient = citizen.reload.patient_id && Patient.find(citizen.patient_id)
    render json: { citizen: { id: citizen.id, cpf_masked: citizen.cpf_masked, verification_level: citizen.verification_level,
                              names: Citizens::NamesJson.call(citizen),
                              demographics: patient ? Patients::Demographics.flags(patient) : { race_color_set: false, address_set: false } } }
    ```
  - e o método privado:

    ```ruby
    # D-U1: grava no paciente do par validado (Patients::Resolve cria se faltar); trilha só com ids.
    def save_demographics(citizen, values)
      return nil if values.empty?

      resolved = Patients::Resolve.call(citizen)
      return resolved if resolved.failure?

      patient = resolved.payload[:patient]
      ApplicationRecord.transaction do
        patient.lock!
        saved = Patients::Demographics.save!(patient, values)
        DomainEvents.publish("patient.demographics_updated", patient_id: patient.id, user_id: Current.user.id) if saved.ok?
        saved
      end
    end
    ```

  Em `app/commands/citizens/erase.rb`:
  - antes de `retained = RequestErasure.clinical_on_pairs?(pair_scope)`: `locked_patients = []`;
  - troque `Patient.where(id: patient_ids).order(:id).lock.to_a` por `locked_patients = Patient.where(id: patient_ids).order(:id).lock.to_a`;
  - depois de `pairs.each { |citizen| erase_pair(citizen, live.select { |a| a.citizen_id == citizen.id }) }`:

    ```ruby
            # D-U1 (ADR 0039): o paciente não retido perde raça/cor, etnia e endereço junto com o par.
            locked_patients.each { |patient| patient.update!(Patient::DEMOGRAPHICS.index_with(nil)) }
    ```

  Em `config/initializers/domain_events.rb`, acrescente `billing_patient.demographics_updated patient.demographics_updated` à lista do módulo 22. Em `config/initializers/filter_parameter_logging.rb`, no fim da lista:

  ```ruby
    # ADR 0039 (D-U1): raça/cor, etnia e endereço do paciente para o BPA-I
    # (`zip`, `street` e `address` já estavam; aqui as chaves inteiras).
    /\A(race_color|ethnicity|address_zip|address_street|address_number|address_complement|address_district)\z/
  ```
  (troque o `]` final da lista pela vírgula depois do último item existente).

- [ ] **Step 4: Rode:** `rspec spec/services/patients/demographics_spec.rb spec/requests/billing/patient_demographics_spec.rb spec/requests/attendance_demographics_spec.rb spec/requests/attendance_names_spec.rb spec/requests/attendance_cadsus_spec.rb spec/commands/citizens spec/initializers/domain_events_bindings_spec.rb` → verde.

- [ ] **Step 5: Commit:**

  ```bash
  git add app/services/patients/demographics.rb app/controllers/billing/patients_controller.rb app/controllers/attendance_controller.rb app/commands/citizens/erase.rb config/routes.rb config/initializers/domain_events.rb config/initializers/filter_parameter_logging.rb spec/services/patients/demographics_spec.rb spec/requests/billing/patient_demographics_spec.rb spec/requests/attendance_demographics_spec.rb spec/commands/citizens/erase_demographics_spec.rb spec/initializers/domain_events_bindings_spec.rb
  git commit -m "feat: capture patient race and address for BPA-I at reception and billing

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

---

### Task 9: Fechamento, pendências e reabertura (F-22.5)

**Files:**
- Create: `app/services/billing/patient_contact.rb`, `app/services/billing/lines.rb`, `app/services/billing/recalculate.rb`, `app/services/billing/pending.rb`, `app/services/billing/summary.rb`, `app/commands/billing/close.rb`, `app/commands/billing/reopen.rb`, `spec/commands/billing/close_spec.rb`, `spec/commands/billing/reopen_spec.rb`, `spec/services/billing/pending_spec.rb`, `spec/commands/billing/close_concurrency_spec.rb`

**Interfaces:**
- Consumes: `Billing::BpaFile` (Task 8); `Billing::Critic`, `Billing::Establishment`, `Billing::Staff` (Task 4); `Billing::Periods`, `Billing::Record` (Task 5); `Billing::Labels` (Task 6); `BillingSetting.current` (Task 3).
- Produces:
  - `Billing::PatientContact.for(patient_ids) -> { patient_id => Contact(district, phone) }`;
  - `Billing::Lines.build(period:, establishment:, procedures:, settings:) -> BpaFile::Built`;
  - `Billing::Recalculate.call(procedure, establishment:) -> Critic::Outcome` (grava) e `.outcome(procedure, establishment:) -> Critic::Outcome` (só lê) — os dois com `closing: true` (raça/cor e endereço, D-U1);
  - `Billing::Pending::FIX`, `Billing::Pending.call(period) -> [Hash]` (B6: `procedure_id`, `reasons`, `fix`, `performed_on`, `sigtap_code`, `label`, `patient: { id, display_name }`, `professional: { user_id, name }`);
  - `Billing::Summary.call(period) -> { bpa_i_lines:, bpa_c_lines:, not_billable: { motivo => n }, pending: }`;
  - `Billing::Close.call(period:, by:, now: Time.current) -> Result` (`ok(period:, export:)`; `settings_missing`, `establishment_is_aps`, `invalid_transition` (409), `nothing_to_export`, `too_many_sheets`);
  - `Billing::Reopen.call(period:, reason:, by:, now: Time.current) -> Result` (`ok(period:)`; `reason_required`, `invalid_transition` (409)).

- [ ] **Step 1: Teste que falha** — `spec/commands/billing/close_spec.rb`:

  ```ruby
  require "rails_helper"

  # ADR 0039 (spec §5.4–§5.5, §6; contrato §5): fecha por estabelecimento e
  # competência com a crítica recalculada (cadastro do momento); BPA-I item a
  # item, BPA-C agregado; cabeçalho com a competência de processamento e
  # linha com a de realização; arquivo cifrado, só acréscimo; procedimentos
  # ligados à versão; UBS nunca; nada a exportar não fecha.
  RSpec.describe Billing::Close do
    include ActiveSupport::Testing::TimeHelpers

    before do
      travel_to(Time.zone.local(Time.zone.today.year, Time.zone.today.month, 15, 12, 0, 0))
      billing_city!
      billing_settings!
    end

    after { travel_back }

    let(:unit) { upa! }
    let(:doctor) { doctor!(unit) }
    let(:nurse) { nurse!(unit) }
    let(:clerk) { billing_clerk! }
    let(:patient) { billing_patient! }
    let(:today) { Time.zone.today }
    let(:competence) { today.strftime("%Y%m") }

    def period = BillingPeriod.find_by!(health_unit_id: unit.id, competence: competence)
    def lines(export) = export.bytes.split("\r\n")
    def age = format("%03d", Citizen.age_between(Date.iso8601(patient.birth_date), today))

    it "gera a versão 1: BPA-I item a item, BPA-C agregado, controle, cabeçalho e eventos" do
      i1 = recorded_procedure!(unit: unit, professional: doctor, patient: patient)
      i2 = recorded_procedure!(unit: unit, professional: doctor, patient: patient, cid10: "E119", character: "urgent")
      c1 = recorded_procedure!(unit: unit, professional: nurse, patient: patient, sigtap_code: "0301100012", cid10: nil, quantity: 2)
      c2 = recorded_procedure!(unit: unit, professional: nurse, patient: patient, sigtap_code: "0301100012", cid10: nil, quantity: 3)
      bad = recorded_procedure!(unit: unit, professional: doctor, patient: patient, cid10: nil)
      result = described_class.call(period: period, by: clerk)
      export = result.payload[:export]
      control = (301_100_012 + 5 + (301_060_029 + 1) * 2) % 1111 + 1111
      expect(export).to have_attributes(version: 1, cnes: unit.cnes, competence: competence, lines_bpa_i: 2, lines_bpa_c: 1, sheets: 2,
                                        control_field: control, sha256: Digest::SHA256.hexdigest(export.bytes), generated_by_user_id: clerk.id)
      header, c_line, *i_lines = lines(export)
      expect(header[0, 29]).to eq("01#BPA##{competence}000003000002#{control}")
      expect(header[29, 30]).to eq("SMS CURITIBA".ljust(30))
      expect(c_line).to eq("02#{unit.cnes}#{competence}223505001010301100012#{age}000005BPA")
      expect(i_lines.map { |l| l[49, 10] }).to eq(%w[0301060029 0301060029])
      expect(i_lines.map { |l| l[94, 2] }).to match_array(%w[01 02])
      expect(i_lines.map { |l| l[338, 11] }.uniq).to eq([ patient.cpf ])
      expect(i_lines.first[15, 15]).to eq(doctor.professional.cns)
      expect(i_lines.first[75, 6]).to eq("410690")
      expect([ i1, i2, c1, c2 ].map { |p| p.reload.billing_export_id }.uniq).to eq([ export.id ])
      expect(bad.reload.billing_export_id).to be_nil
      expect(period).to have_attributes(status: "closed", version: 1, closed_by_user_id: clerk.id)
      expect(DomainEvent.where(name: %w[billing_export.generated billing_period.closed]).pluck(:payload))
        .to contain_exactly({ "billing_export_id" => export.id, "billing_period_id" => period.id }, { "billing_period_id" => period.id, "version" => 1 })
    end

    it "atrasado no arquivo: linha com o mês da realização, cabeçalho com a competência fechada agora" do
      previous = (today << 1).strftime("%Y%m")
      BillingPeriod.create!(health_unit: unit, competence: previous, status: "closed", version: 1, closed_at: Time.current, closed_by_user: clerk)
      recorded_procedure!(unit: unit, professional: doctor, patient: patient, performed_on: today << 1)
      header, line = lines(described_class.call(period: period, by: clerk).payload[:export])
      expect(header[7, 6]).to eq(competence)
      expect(line[9, 6]).to eq(previous)
      expect(line[36, 8]).to eq((today << 1).strftime("%Y%m%d"))
    end

    it "cadastro corrigido basta: sem perfil nem vínculo na gravação, com perfil e vínculo no fechamento" do
      newcomer = staff_with("novato-#{SecureRandom.hex(3)}@cidade.gov.br", "health_professional")
      procedure = Billing::Record.call(patient: patient, establishment: Billing::Establishment.for_unit(unit), professional: newcomer, source: "manual",
                                       recorded_by: clerk, params: { "performed_on" => today.iso8601, "sigtap_code" => "0301060029",
                                                                     "quantity" => 1, "cid10" => "I10" }).payload[:procedure]
      expect(procedure.blocking_reasons).to eq(%w[missing_professional_cbo missing_professional_cns])
      expect(Billing::Pending.call(period).sole).to include(procedure_id: procedure.id, fix: "professional")
      link_professional!(newcomer, unit, cbo: "225125")
      export = described_class.call(period: period, by: clerk).payload[:export]
      expect(procedure.reload).to have_attributes(billable: true, blocking_reasons: [], cbo_code: "225125", billing_export_id: export.id)
      expect(lines(export).last[15, 15]).to eq(newcomer.reload.professional.cns)
    end

    it "nada a exportar: 422 e a competência segue aberta, com a crítica atualizada" do
      bad = recorded_procedure!(unit: unit, professional: doctor, patient: patient, cid10: nil)
      expect(described_class.call(period: period, by: clerk).reason).to eq(:nothing_to_export)
      expect(period).to have_attributes(status: "open", version: 0)
      expect(BillingExport.count).to eq(0)
      expect(bad.reload.blocking_reasons).to eq(%w[cid_required])
    end

    it "sem configuração, fechada ou virada UBS: recusa sem arquivo" do
      recorded_procedure!(unit: unit, professional: doctor, patient: patient)
      BillingSetting.delete_all
      expect(described_class.call(period: period, by: clerk).reason).to eq(:settings_missing)
      billing_settings!
      expect(described_class.call(period: period, by: clerk)).to be_ok
      expect(described_class.call(period: period, by: clerk).reason).to eq(:invalid_transition)
      other = upa!
      recorded_procedure!(unit: other, professional: doctor!(other), patient: patient)
      other.update!(kind: "ubs")
      result = described_class.call(period: BillingPeriod.find_by!(health_unit_id: other.id), by: clerk)
      expect(result.reason).to eq(:establishment_is_aps)
      expect(BillingExport.joins(:billing_period).where(billing_periods: { health_unit_id: other.id })).to be_empty
    end

    it "D-U1: paciente sem raça/cor ou endereço fica fora do arquivo até o cadastro ser completado" do
      bare = billing_patient!(demographics: false)
      pending = recorded_procedure!(unit: unit, professional: doctor, patient: bare)
      done = recorded_procedure!(unit: unit, professional: doctor, patient: patient)
      expect(Billing::Pending.call(period).map { |i| [ i[:procedure_id], i[:reasons], i[:fix] ] })
        .to eq([ [ pending.id, %w[missing_race_color missing_address], "patient" ] ])
      export = described_class.call(period: period, by: clerk).payload[:export]
      expect(pending.reload).to have_attributes(billable: false, blocking_reasons: %w[missing_race_color missing_address], billing_export_id: nil)
      expect(done.reload.billing_export_id).to eq(export.id)
    end

    it "a configuração entra no cabeçalho, com a sigla" do
      BillingSetting.current.update!(origin_acronym: "SMS", destination_body: "Secretaria Estadual", management: "E")
      recorded_procedure!(unit: unit, professional: doctor, patient: patient)
      header = lines(described_class.call(period: period, by: clerk).payload[:export]).first
      expect(header[59, 6]).to eq("SMS   ")
      expect(header[65, 14]).to eq("11222333000181")
      expect(header[79, 41]).to eq("SECRETARIA ESTADUAL".ljust(40) + "E")
      expect(header[120, 10]).to eq("ROTASAUDE ")
    end
  end
  ```

  `spec/commands/billing/reopen_spec.rb`:

  ```ruby
  require "rails_helper"

  # Spec §5.5 com B8: reabrir exige motivo (10..500, cifrado) e só a fechada;
  # o próximo fechamento é a versão 2; a versão 1 fica intacta no histórico.
  RSpec.describe Billing::Reopen do
    before do
      billing_city!
      billing_settings!
    end

    let(:unit) { upa! }
    let(:doctor) { doctor!(unit) }
    let(:clerk) { billing_clerk! }

    it "reabre com motivo e o próximo arquivo é a versão 2; a 1 não muda" do
      first = recorded_procedure!(unit: unit, professional: doctor)
      period = first.billing_period
      v1 = Billing::Close.call(period: period, by: clerk).payload[:export]
      sha = v1.sha256
      expect(described_class.call(period: period, reason: "curto", by: clerk).reason).to eq(:reason_required)
      expect(described_class.call(period: period, reason: "x" * 501, by: clerk).reason).to eq(:reason_required)
      result = described_class.call(period: period, reason: "CNS do médico corrigido no CNES", by: clerk)
      expect(result.payload[:period]).to have_attributes(status: "open", version: 1, reopened_reason: "CNS do médico corrigido no CNES",
                                                         reopened_by_user_id: clerk.id)
      expect(described_class.call(period: period, reason: "De novo, por engano", by: clerk).reason).to eq(:invalid_transition)
      expect(DomainEvent.where(name: "billing_period.reopened").last.payload).to eq("billing_period_id" => period.id, "version" => 1)
      second = recorded_procedure!(unit: unit, professional: doctor)
      expect(second.billing_period_id).to eq(period.id)
      v2 = Billing::Close.call(period: period.reload, by: clerk).payload[:export]
      expect(v2).to have_attributes(version: 2, lines_bpa_i: 2)
      expect(v1.reload.sha256).to eq(sha)
      expect([ first, second ].map { |p| p.reload.billing_export_id }.uniq).to eq([ v2.id ])
      expect(period.reload).to have_attributes(status: "closed", version: 2)
    end
  end
  ```

  `spec/services/billing/pending_spec.rb`:

  ```ruby
  require "rails_helper"

  # Contrato §5 com B6: pendências recalculadas na hora (sem gravar), só de
  # procedimento BPA não faturável; correção pelo primeiro motivo corrigível;
  # o item traz o que a tela mostra.
  RSpec.describe Billing::Pending do
    before { billing_city! }

    let(:unit) { upa! }
    let(:doctor) { doctor!(unit) }
    let(:patient) { billing_patient! }

    it "lista o corrigível, com a forma de B6; ignora o faturável e o que não é BPA" do
      recorded_procedure!(unit: unit, professional: doctor, patient: patient)
      apac = recorded_procedure!(unit: unit, professional: doctor, patient: patient, sigtap_code: "0604320124", cid10: nil)
      cid = recorded_procedure!(unit: unit, professional: doctor, patient: patient, cid10: nil)
      items = described_class.call(cid.billing_period)
      expect(items).to eq([ { procedure_id: cid.id, reasons: %w[cid_required], fix: "procedure", performed_on: cid.performed_on.iso8601,
                              sigtap_code: "0301060029", label: "ATENDIMENTO DE URGENCIA C/ OBSERVACAO ATE 24 HORAS EM ATENCAO ESPECIALIZADA",
                              patient: { id: patient.id, display_name: "Maria Aparecida da Silva" },
                              professional: { user_id: doctor.id, name: doctor.professional.professional_name } } ])
      expect(items.map { |i| i[:procedure_id] }).not_to include(apac.id)
      expect(described_class::FIX).to include("missing_professional_cns" => "professional", "missing_required_patient_field" => "patient",
                                              "missing_establishment_cnes" => "establishment")
    end

    it "recalcula sem gravar: o cadastro corrigido some da lista antes do fechamento" do
      unit.update!(cnes: nil)
      procedure = recorded_procedure!(unit: unit, professional: doctor, patient: patient)
      expect(described_class.call(procedure.billing_period).sole).to include(fix: "establishment", reasons: %w[missing_establishment_cnes])
      unit.update!(cnes: billing_cnes)
      expect(described_class.call(procedure.billing_period)).to eq([])
      expect(procedure.reload.blocking_reasons).to eq(%w[missing_establishment_cnes])
    end
  end
  ```

  `spec/commands/billing/close_concurrency_spec.rb` (threads reais, sem fixture transacional; a crítica é trocada por um resultado fixo — a prova é das travas, não da SIGTAP):

  ```ruby
  require "rails_helper"

  # Review Focus 4: o faturista fecha enquanto o profissional grava um
  # procedimento da mesma competência. A gravação espera a trava da
  # competência (FOR SHARE × FOR UPDATE) e, fechada, cai na competência
  # seguinte, atrasada — nunca fica numa competência fechada fora do arquivo.
  RSpec.describe "Fechamento × gravação ao mesmo tempo" do
    self.use_transactional_tests = false

    let(:ids) { {} }
    let(:threads) { [] }
    let(:release) { Queue.new }

    def in_city(&) = CityConnection.with(TEST_CITY_A) { Current.set(city: TEST_CITY_A, &) }

    def insert(at)
      unit = HealthUnit.find(ids[:unit])
      Billing::Record.insert!(patient: Patient.find(ids[:patient]), establishment: Billing::Establishment.for_unit(unit),
                              professional: User.find(ids[:doctor]), cbo_code: "225125", source: "manual", performed_on: at,
                              sigtap_code: "0301100012", quantity: 1, cid10: nil, character: "elective", recorded_by: User.find(ids[:doctor]))
    end

    before do
      allow(Billing::Critic).to receive(:call).and_return(
        Billing::Critic::Outcome.new(instrument: "bpa_c", billable: true, reasons: [], sigtap_competence: nil, fallback: false, label: nil)
      )
      in_city do
        tag = SecureRandom.hex(4)
        admin = User.create!(email_address: "adm-#{tag}@c.gov.br", password: "senha-segura-123")
        doctor = User.create!(email_address: "doc-#{tag}@c.gov.br", password: "senha-segura-123")
        unit = HealthUnit.create!(name: "UPA #{tag}", kind: "upa", cnes: format("%07d", SecureRandom.random_number(9_000_000) + 1_000_000))
        patient = Patient.create!(cpf: CampaignHistory.cpf_for("faturamento-#{tag}"), full_name: "Maria Aparecida da Silva",
                                  birth_date: "1980-05-10", sex: "female")
        ids[:settings_created] = BillingSetting.current.nil?
        if ids[:settings_created]
          BillingSetting.create!(origin_name: "SMS", origin_cnpj: "11222333000181", destination_body: "SMS", management: "M", updated_by_user: admin)
        end
        ids.merge!(admin: admin.id, doctor: doctor.id, unit: unit.id, patient: patient.id)
      end
    end

    after do
      3.times { release << true }
      threads.each do |t|
        t.join(5) || t.kill
      rescue StandardError
        nil
      end
      in_city do
        ApplicationRecord.transaction do
          ApplicationRecord.connection.execute("SET LOCAL session_replication_role = replica")
          procedures = PerformedProcedure.where(health_unit_id: ids[:unit])
          periods = BillingPeriod.where(health_unit_id: ids[:unit])
          exports = BillingExport.where(billing_period_id: periods.select(:id))
          refs = procedures.pluck(:id) + periods.pluck(:id) + exports.pluck(:id)
          %w[performed_procedure_id billing_period_id billing_export_id].each do |key|
            DomainEvent.where("payload->>'#{key}' IN (?)", refs.presence || [ "" ]).delete_all
          end
          procedures.delete_all
          exports.delete_all
          periods.delete_all
          BillingSetting.delete_all if ids[:settings_created]
          Patient.where(id: ids[:patient]).delete_all
        end
      end
      purge_committed_rows(ids.slice(:admin, :doctor, :unit))
    end

    it "a gravação espera o fechamento e cai na competência seguinte, atrasada" do
      today = Date.current
      first = in_city { insert(today) }
      holder = nil
      holding = Queue.new
      original = DomainEvents.method(:publish)
      allow(DomainEvents).to receive(:publish) do |*args, **kwargs, &blk|
        result = original.call(*args, **kwargs, &blk)
        if Thread.current == holder && args.first == "billing_period.closed"
          holding << true
          release.pop(timeout: 10) or raise "timeout esperando release"
        end
        result
      end

      closed = Queue.new
      threads << (holder = Thread.new do
        closed << in_city { Billing::Close.call(period: BillingPeriod.find(first.billing_period_id), by: User.find(ids[:admin])) }
      end)
      holding.pop(timeout: 5) or raise "o fechamento não segurou a trava"

      recorded = Queue.new
      threads << Thread.new { recorded << in_city { insert(today) } }
      expect(wait_for_lock_wait).to be(true)
      release << true

      expect(closed.pop(timeout: 10)).to be_ok
      late = recorded.pop(timeout: 10)
      expect(late).to have_attributes(late: true, billing_competence: today.next_month.strftime("%Y%m"))
      in_city do
        expect(first.reload.billing_export_id).to be_present
        expect(late.reload.billing_export_id).to be_nil
      end
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar:** `rspec spec/commands/billing/close_spec.rb spec/commands/billing/reopen_spec.rb spec/services/billing/pending_spec.rb spec/commands/billing/close_concurrency_spec.rb` → `NameError: uninitialized constant Billing::Close`.

- [ ] **Step 3: Implemente.** `app/services/billing/patient_contact.rb`:

  ```ruby
  # Bairro e telefone do par validado mais recente do paciente (spec §2
  # decisão 8: o que o wpda tem; o endereço do BPA-I é o do paciente, D-U1). Telefone sem o
  # 55, só com 10 ou 11 dígitos; senão, nada.
  module Billing
    module PatientContact
      Contact = Data.define(:district, :phone)

      module_function

      def for(patient_ids)
        Citizen.not_erased.verification_level_verified.where(patient_id: patient_ids).includes(:neighborhood)
               .order(:updated_at, :id).to_a.group_by(&:patient_id).transform_values { |list| contact(list.last) }
      end

      def contact(citizen)
        digits = citizen.phone.to_s.gsub(/\D/, "")
        digits = digits.delete_prefix("55") if digits.length > 11
        Contact.new(district: citizen.neighborhood&.name, phone: digits.match?(/\A\d{10,11}\z/) ? digits : nil)
      end
      private_class_method :contact
    end
  end
  ```

  `app/services/billing/lines.rb`:

  ```ruby
  # Linhas do arquivo a partir dos procedimentos faturáveis de uma competência,
  # com o cadastro do momento (congelado no arquivo). BPA-C agregado por
  # CBO × procedimento × idade × mês de realização; BPA-I item a item, com
  # raça/cor, etnia e endereço do cadastro do paciente (D-U1; o fechamento só
  # chega aqui com eles completos); IBGE de residência = os 6 primeiros
  # dígitos do IBGE da cidade (spec §6); telefone (e bairro, se o paciente não
  # tem) do par do wpda.
  module Billing
    module Lines
      module_function

      def build(period:, establishment:, procedures:, settings:)
        ibge = CityProfile.current&.ibge_code.to_s[0, 6].presence
        c_lines = procedures.select { |p| p.instrument == "bpa_c" }
                            .group_by { |p| [ p.cbo_code, p.sigtap_code, age(p), p.performed_on.strftime("%Y%m") ] }
                            .map do |(cbo, code, years, month), list|
          BpaFile::CLine.new(cnes: establishment.cnes, competence: month, cbo: cbo, procedure: code, age: years, quantity: list.sum(&:quantity))
        end
        individual = procedures.select { |p| p.instrument == "bpa_i" }
        contacts = PatientContact.for(individual.map(&:patient_id).uniq)
        i_lines = individual.map do |p|
          contact = contacts[p.patient_id]
          BpaFile::ILine.new(key: p.id, cnes: establishment.cnes, competence: p.performed_on.strftime("%Y%m"),
                             professional_cns: Staff.cns(p.professional_user), cbo: p.cbo_code, performed_on: p.performed_on,
                             procedure: p.sigtap_code, sex: Critic::SEX.fetch(p.patient.sex), ibge: ibge, cid10: p.cid10, age: age(p),
                             quantity: p.quantity, character: BpaFile::CHARACTER.fetch(p.character), name: p.patient.full_name,
                             birth_date: Date.iso8601(p.patient.birth_date), race_color: p.patient.race_color,
                             ethnicity: p.patient.race_color == "05" ? p.patient.ethnicity : nil, zip: p.patient.address_zip,
                             street: p.patient.address_street, number: p.patient.address_number, complement: p.patient.address_complement,
                             district: p.patient.address_district.presence || contact&.district, phone: contact&.phone, cpf: p.patient.cpf)
        end
        header = BpaFile::Header.new(competence: period.competence, origin_name: settings.origin_name, origin_acronym: settings.origin_acronym,
                                     origin_cnpj: settings.origin_cnpj, destination_body: settings.destination_body,
                                     management: settings.management)
        BpaFile.build(header: header, c_lines: c_lines, i_lines: i_lines)
      end

      def age(procedure) = Citizen.age_between(Date.iso8601(procedure.patient.birth_date), procedure.performed_on)
      private_class_method :age
    end
  end
  ```

  `app/services/billing/recalculate.rb`:

  ```ruby
  # Crítica recalculada com o cadastro do momento (plano do módulo 22, Desvio
  # 7): CBO vazio é procurado de novo no vínculo; CNS, sexo, nascimento, nome e
  # CNES relidos. `call` grava (fechamento); `outcome` só lê (pendências).
  module Billing
    module Recalculate
      module_function

      def call(procedure, establishment:)
        cbo = procedure.cbo_code || Staff.cbo_for(procedure.professional_user, establishment)
        result = outcome(procedure, establishment: establishment, cbo: cbo)
        changes = { instrument: result.instrument, billable: result.billable, blocking_reasons: result.reasons,
                    sigtap_competence: result.sigtap_competence, sigtap_fallback: result.fallback }
        changes[:cbo_code] = cbo if procedure.cbo_code.nil? && cbo
        procedure.update!(changes) if changes.any? { |key, value| procedure.public_send(key) != value }
        result
      end

      def outcome(procedure, establishment:, cbo: procedure.cbo_code || Staff.cbo_for(procedure.professional_user, establishment))
        Critic.call(Critic::Facts.new(establishment: establishment, sigtap_code: procedure.sigtap_code, performed_on: procedure.performed_on,
                                      cbo_code: cbo, professional_cns: Staff.cns(procedure.professional_user), cid10: procedure.cid10,
                                      patient: procedure.patient), closing: true)
      end
    end
  end
  ```

  `app/services/billing/pending.rb`:

  ```ruby
  # Pendências da competência (contrato §5 com B6; plano, Desvio 8): crítica
  # recalculada na hora (sem gravar); entra o procedimento gravado, BPA, não
  # faturável e de estabelecimento não-APS; `fix` pelo primeiro motivo
  # corrigível. A trilha (tem paciente) é do controlador.
  module Billing
    module Pending
      FIX = { "missing_professional_cns" => "professional", "missing_professional_cbo" => "professional", "missing_patient_id" => "patient",
              "missing_required_patient_field" => "patient", "missing_race_color" => "patient", "missing_address" => "patient",
              "missing_establishment_cnes" => "establishment" }.freeze

      module_function

      def call(period)
        establishment = Establishment.of(period)
        labels = {}
        PerformedProcedure.recorded.where(billing_period_id: period.id).includes(:patient, professional_user: :professional)
                          .order(:performed_on, :id).filter_map do |procedure|
          outcome = Recalculate.outcome(procedure, establishment: establishment)
          next if outcome.billable || outcome.instrument == "not_bpa" || outcome.reasons.include?("establishment_is_aps")

          label = labels[[ procedure.sigtap_code, outcome.sigtap_competence ]] ||= Labels.procedure(procedure.sigtap_code, outcome.sigtap_competence)
          { procedure_id: procedure.id, reasons: outcome.reasons, fix: outcome.reasons.filter_map { |r| FIX[r] }.first || "procedure",
            performed_on: procedure.performed_on.iso8601, sigtap_code: procedure.sigtap_code, label: label,
            patient: { id: procedure.patient_id, display_name: procedure.patient.display_name },
            professional: { user_id: procedure.professional_user_id, name: Screenings::Json.staff_name(procedure.professional_user) } }
        end
      end
    end
  end
  ```

  `app/services/billing/summary.rb`:

  ```ruby
  # Resumo da competência (contrato §2). Fechada: linhas do último arquivo.
  # Aberta: itens faturáveis por instrumento (o BPA-C ainda não agregado),
  # contagem por motivo e pendências — com a crítica gravada (sem recalcular;
  # o recálculo é das pendências e do fechamento).
  module Billing
    module Summary
      module_function

      def call(period)
        rows = PerformedProcedure.recorded.where(billing_period_id: period.id)
        not_billable = rows.where(billable: false).pluck(:blocking_reasons).flatten.tally
        pending = rows.where(billable: false).where.not(instrument: "not_bpa")
                      .where.not("'establishment_is_aps' = ANY (performed_procedures.blocking_reasons)").count
        export = period.closed? ? period.exports.order(version: :desc).first : nil
        { bpa_i_lines: export ? export.lines_bpa_i : rows.where(billable: true, instrument: "bpa_i").count,
          bpa_c_lines: export ? export.lines_bpa_c : rows.where(billable: true, instrument: "bpa_c").count,
          not_billable: not_billable, pending: pending }
      end
    end
  end
  ```

  `app/commands/billing/close.rb`:

  ```ruby
  # Fechamento da competência (ADR 0039; spec §5.4; contrato §5). Ordem:
  # configuração e estabelecimento (UBS nunca gera BPA) fora da transação;
  # trava a competência FOR UPDATE (quem grava espera, e depois cai na
  # competência seguinte); trava os procedimentos por id; recalcula a crítica
  # com o cadastro do momento (grava); sem faturável → nothing_to_export (a
  # crítica atualizada fica); gera o arquivo (cifrado), liga os procedimentos à
  # versão, fecha. Mais de 999 folhas: too_many_sheets, nada gravado.
  module Billing
    module Close
      module_function

      def call(period:, by:, now: Time.current)
        settings = BillingSetting.current
        return Result.fail(:settings_missing) unless settings

        establishment = Establishment.of(period)
        return Result.fail(:establishment_is_aps) if establishment.aps?

        ApplicationRecord.transaction do
          period.lock!
          next Result.fail(:invalid_transition, details: { status: period.status }) unless period.status == "open"

          procedures = PerformedProcedure.recorded.where(billing_period_id: period.id).order(:id).lock
                                         .includes(:patient, professional_user: :professional).to_a
          procedures.each { |procedure| Recalculate.call(procedure, establishment: establishment) }
          billable = procedures.select(&:billable)
          next Result.fail(:nothing_to_export) if billable.empty?

          built = Lines.build(period: period, establishment: establishment, procedures: billable, settings: settings)
          export = BillingExport.create!(billing_period: period, version: period.version + 1, cnes: establishment.cnes, competence: period.competence,
                                         file: built.bytes.dup.force_encoding(Encoding::UTF_8), sha256: Digest::SHA256.hexdigest(built.bytes),
                                         lines_bpa_c: built.lines_bpa_c, lines_bpa_i: built.lines_bpa_i, sheets: built.sheets,
                                         control_field: built.control, generated_by_user: by, generated_at: now)
          PerformedProcedure.where(id: billable.map(&:id)).update_all(billing_export_id: export.id, updated_at: now)
          period.update!(status: "closed", version: export.version, closed_at: now, closed_by_user: by)
          DomainEvents.publish("billing_export.generated", billing_export_id: export.id, billing_period_id: period.id)
          DomainEvents.publish("billing_period.closed", billing_period_id: period.id, version: export.version)
          Result.ok(period: period, export: export)
        end
      rescue BpaFile::TooManySheets
        Result.fail(:too_many_sheets)
      end
    end
  end
  ```

  `app/commands/billing/reopen.rb`:

  ```ruby
  # Reabertura (spec §5.5; contrato §5 com B8): motivo 10..500 (cifrado), só a
  # competência fechada; o arquivo anterior fica no histórico e o próximo
  # fechamento é a versão seguinte. O step-up é do controlador.
  module Billing
    module Reopen
      module_function

      def call(period:, reason:, by:, now: Time.current)
        text = reason.is_a?(String) ? reason.strip : ""
        return Result.fail(:reason_required) unless text.length.between?(10, 500)

        ApplicationRecord.transaction do
          period.lock!
          next Result.fail(:invalid_transition, details: { status: period.status }) unless period.closed?

          period.update!(status: "open", reopened_reason: text, reopened_at: now, reopened_by_user: by)
          DomainEvents.publish("billing_period.reopened", billing_period_id: period.id, version: period.version)
          Result.ok(period: period)
        end
      end
    end
  end
  ```

- [ ] **Step 4: Rode:** `rspec spec/commands/billing/close_spec.rb spec/commands/billing/reopen_spec.rb spec/services/billing/pending_spec.rb spec/commands/billing/close_concurrency_spec.rb spec/commands/billing/record_spec.rb` → verde. A spec de concorrência commita de verdade e limpa no `after`; rode-a sozinha também (`rspec spec/commands/billing/close_concurrency_spec.rb`) e confira que nada sobrou: `rails runner 'CityConnection.with(City.find_by!(slug: "curitiba")) { p PerformedProcedure.where("created_at > ?", 1.hour.ago).count }'` não serve aqui (é o banco de dev) — confira no banco de teste `_mod22` com `psql -d rota_saude_test_city_a_mod22 -c "SELECT count(*) FROM performed_procedures"` → 0.

- [ ] **Step 5: Commit:**

  ```bash
  git add app/services/billing/patient_contact.rb app/services/billing/lines.rb app/services/billing/recalculate.rb app/services/billing/pending.rb app/services/billing/summary.rb app/commands/billing/close.rb app/commands/billing/reopen.rb spec/commands/billing/close_spec.rb spec/commands/billing/reopen_spec.rb spec/services/billing/pending_spec.rb spec/commands/billing/close_concurrency_spec.rb
  git commit -m "feat: close and reopen billing competences with recalculated SIGTAP critic

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

---

### Task 10: Rotas de competência, versões, download com trilha e configuração (F-22.5, F-22.6)

**Files:**
- Create: `app/controllers/billing/periods_controller.rb`, `app/controllers/billing/exports_controller.rb`, `app/controllers/billing/settings_controller.rb`, `app/commands/billing/save_settings.rb`, `spec/requests/billing/periods_spec.rb`, `spec/requests/billing/settings_spec.rb`
- Modify: `app/services/billing/json.rb` (`period`, `export`, `settings`), `config/routes.rb` (escopo `/billing`)

**Interfaces:**
- Consumes: `Billing::Close`, `Billing::Reopen`, `Billing::Pending`, `Billing::Summary` (Task 9); `Billing::BaseController`, `Billing::Access` (Task 6); `MfaStepUp`; `Cnpj.normalize`.
- Produces:
  - `Billing::Json.period(period) -> { id, establishment, competence, status, version, closed_at, closed_by (nome | null), reopened_reason, summary }`; `.export(export) -> { id, version, sha256, lines_bpa_i, lines_bpa_c, generated_at, generated_by (nome) }`; `.settings(setting | nil) -> { origin_name, origin_acronym, origin_cnpj, destination_body, management, updated_at }` (nulls sem configuração).
  - `Billing::SaveSettings.call(params:, by:) -> Result` (`ok(settings:)`; `field_required`/`too_long` com `details: { field: }`, `invalid_cnpj`, `invalid_management`).
  - Rotas `GET /billing/periods`, `GET /billing/periods/:id/pending`, `POST /billing/periods/:id/close` (step-up), `POST /billing/periods/:id/reopen` (step-up), `GET /billing/periods/:id/exports`, `GET /billing/exports/:id/file`, `GET/PUT /billing/settings` (step-up só no `PUT`).

- [ ] **Step 1: Teste que falha** — `spec/requests/billing/periods_spec.rb`:

  ```ruby
  require "rails_helper"

  # Contrato §5 com B6, B8, B9 e B11: competências por estabelecimento com
  # produção (faturista e admin leem), pendências com trilha, fechar e reabrir
  # (só faturista, step-up), versões em envelope (faturista e admin) e o
  # arquivo só para o faturista, com trilha e nome sem dado pessoal.
  RSpec.describe "Faturamento — competências e arquivos", type: :request do
    before do
      billing_city!
      billing_settings!
    end

    let(:unit) { upa! }
    let(:doctor) { doctor!(unit) }
    let(:clerk) { billing_clerk! }
    let(:competence) { Time.zone.today.strftime("%Y%m") }
    let!(:procedure) { recorded_procedure!(unit: unit, professional: doctor) }
    let(:period) { procedure.billing_period }

    it "lista as competências com id, versão 0 antes do fechamento e resumo; faturista e admin; outros 403" do
      recorded_procedure!(unit: unit, professional: doctor, cid10: nil)
      sign_in_as(lab_admin!)
      get "/billing/periods", params: { competence: competence }
      expect(json_body["periods"].sole).to eq(
        "id" => period.id, "establishment" => { "type" => "health_unit", "id" => unit.id, "cnes" => unit.cnes, "name" => unit.name },
        "competence" => competence, "status" => "open", "version" => 0, "closed_at" => nil, "closed_by" => nil, "reopened_reason" => nil,
        "summary" => { "bpa_i_lines" => 1, "bpa_c_lines" => 0, "not_billable" => { "cid_required" => 1 }, "pending" => 1 }
      )
      get "/billing/periods", params: { competence: "2026-10" }
      expect(json_body).to eq("error" => "invalid_filter")
      sign_in_as(doctor)
      get "/billing/periods"
      expect(response).to have_http_status(:forbidden)
    end

    it "pendências: faturista e admin; trilha só com ids" do
      recorded_procedure!(unit: unit, professional: doctor, cid10: nil)
      sign_in_as(lab_admin!)
      get "/billing/periods/#{period.id}/pending"
      expect(json_body["items"].sole).to include("reasons" => %w[cid_required], "fix" => "procedure", "sigtap_code" => "0301060029")
      expect(DomainEvent.where(name: "billing_period.pending_viewed").last.payload.keys).to match_array(%w[billing_period_id user_id])
      get "/billing/periods/#{SecureRandom.uuid}/pending"
      expect(response).to have_http_status(:not_found)
    end

    it "fechar e reabrir: só faturista, com step-up; transição errada 409" do
      sign_in_as(lab_admin!)
      json_post "/billing/periods/#{period.id}/close"
      expect(response).to have_http_status(:forbidden)
      sign_in_as(clerk)
      json_post "/billing/periods/#{period.id}/close"
      expect(response).to have_http_status(:unauthorized)
      expect(json_body).to eq("error" => "mfa_required")
      stepped_up!(clerk)
      json_post "/billing/periods/#{period.id}/close"
      expect(response).to have_http_status(:ok)
      expect(json_body).to include("status" => "closed", "version" => 1, "closed_by" => clerk.email_address)
      json_post "/billing/periods/#{period.id}/close"
      expect(response).to have_http_status(:conflict)
      expect(json_body).to eq("error" => "invalid_transition", "status" => "closed")
      json_post "/billing/periods/#{period.id}/reopen", reason: "curto"
      expect(json_body).to eq("error" => "reason_required")
      json_post "/billing/periods/#{period.id}/reopen", reason: "CNS do médico corrigido no CNES"
      expect(json_body).to include("status" => "open", "version" => 1, "reopened_reason" => "CNS do médico corrigido no CNES")
      json_post "/billing/periods/#{period.id}/reopen", reason: "CNS do médico corrigido no CNES"
      expect(response).to have_http_status(:conflict)
    end

    it "versões em envelope (faturista e admin); arquivo só do faturista, com trilha e nome sem dado pessoal" do
      export = Billing::Close.call(period: period, by: clerk).payload[:export]
      sign_in_as(lab_admin!)
      get "/billing/periods/#{period.id}/exports"
      expect(json_body).to eq("exports" => [ { "id" => export.id, "version" => 1, "sha256" => export.sha256, "lines_bpa_i" => 1, "lines_bpa_c" => 0,
                                               "generated_at" => export.generated_at.iso8601, "generated_by" => clerk.email_address } ])
      get "/billing/exports/#{export.id}/file"
      expect(response).to have_http_status(:forbidden)
      sign_in_as(clerk)
      get "/billing/exports/#{export.id}/file"
      expect(response).to have_http_status(:ok)
      expect(response.body.b).to eq(export.bytes)
      expect(response.media_type).to eq("text/plain")
      expect(response.headers["Content-Disposition"]).to include("attachment", "bpa-#{unit.cnes}-#{competence}-v1.txt")
      expect(DomainEvent.where(name: "billing_export.downloaded").last.payload).to eq("billing_export_id" => export.id, "user_id" => clerk.id)
      get "/billing/exports/#{SecureRandom.uuid}/file"
      expect(response).to have_http_status(:not_found)
    end

    it "desligado: 403 em toda rota de competência" do
      Platform::Features.set!(city: Current.city, key: "procedure_billing", enabled: false, maintainer: ledi_maintainer!)
      sign_in_as(clerk)
      get "/billing/periods"
      expect(json_body).to eq("error" => "feature_disabled", "feature" => "procedure_billing")
    end
  end
  ```

  `spec/requests/billing/settings_spec.rb`:

  ```ruby
  require "rails_helper"

  # Contrato §6 com B10: admin municipal; GET 200 com nulls; PUT com
  # step-up; larguras do layout recusadas (nunca cortadas), com o campo;
  # CNPJ só numérico (o campo do cabeçalho é NUM) — Review Focus 5.
  RSpec.describe "Faturamento — configuração", type: :request do
    before { billing_city! }

    let(:admin) { lab_admin! }
    let(:valid) do
      { origin_name: "SMS Curitiba", origin_acronym: "SMS", origin_cnpj: "11.222.333/0001-81", destination_body: "Secretaria Municipal de Saúde",
        management: "M" }
    end

    it "GET sem configuração: 200 com nulls; faturista não lê" do
      sign_in_as(admin)
      get "/billing/settings"
      expect(json_body).to eq("origin_name" => nil, "origin_acronym" => nil, "origin_cnpj" => nil, "destination_body" => nil,
                              "management" => nil, "updated_at" => nil)
      sign_in_as(billing_clerk!)
      get "/billing/settings"
      expect(response).to have_http_status(:forbidden)
    end

    it "PUT com step-up grava e publica só ids" do
      sign_in_as(admin)
      json_put "/billing/settings", valid
      expect(json_body).to eq("error" => "mfa_required")
      stepped_up!(admin)
      json_put "/billing/settings", valid
      expect(response).to have_http_status(:ok)
      expect(json_body).to include("origin_name" => "SMS Curitiba", "origin_acronym" => "SMS", "origin_cnpj" => "11222333000181", "management" => "M")
      expect(DomainEvent.where(name: "billing_settings.changed").last.payload.keys).to match_array(%w[billing_setting_id user_id])
      json_put "/billing/settings", valid.merge(management: "E", origin_acronym: "")
      expect(BillingSetting.count).to eq(1)
      expect(BillingSetting.current).to have_attributes(management: "E", origin_acronym: nil)
    end

    it "recusas: campos, larguras, CNPJ alfanumérico e indicador; nada gravado" do
      stepped_up!(admin)
      {
        { origin_name: " " } => { "error" => "field_required", "field" => "origin_name" },
        { origin_name: "S" * 31 } => { "error" => "too_long", "field" => "origin_name" },
        { origin_acronym: "SMSCURI" } => { "error" => "too_long", "field" => "origin_acronym" },
        { destination_body: "" } => { "error" => "field_required", "field" => "destination_body" },
        { destination_body: "D" * 41 } => { "error" => "too_long", "field" => "destination_body" },
        { origin_cnpj: "11.222.333/0001-82" } => { "error" => "invalid_cnpj" },
        { origin_cnpj: "12.ABC.345/01DE-35" } => { "error" => "invalid_cnpj" },
        { management: "X" } => { "error" => "invalid_management" }
      }.each do |over, error|
        json_put "/billing/settings", valid.merge(over)
        expect(json_body).to eq(error), over.inspect
      end
      expect(BillingSetting.count).to eq(0)
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar:** `rspec spec/requests/billing/periods_spec.rb spec/requests/billing/settings_spec.rb` → `ActionController::RoutingError`.

- [ ] **Step 3: Implemente.** Acrescente a `app/services/billing/json.rb`, dentro do módulo (antes de `private_class_method`):

  ```ruby
      # <billing_period> (contrato §2 com B11): id sempre; versão 0 antes do primeiro fechamento.
      def period(period)
        { id: period.id, establishment: Establishment.of(period).json, competence: period.competence, status: period.status,
          version: period.version, closed_at: period.closed_at&.iso8601, closed_by: Screenings::Json.staff_name(period.closed_by_user),
          reopened_reason: period.reopened_reason, summary: Summary.call(period) }
      end

      # Versão do arquivo (B9): `generated_by` é o nome.
      def export(export)
        { id: export.id, version: export.version, sha256: export.sha256, lines_bpa_i: export.lines_bpa_i, lines_bpa_c: export.lines_bpa_c,
          generated_at: export.generated_at.iso8601, generated_by: Screenings::Json.staff_name(export.generated_by_user) }
      end

      # Configuração (contrato §6 com B10): sem linha, nulls.
      def settings(setting)
        { origin_name: setting&.origin_name, origin_acronym: setting&.origin_acronym, origin_cnpj: setting&.origin_cnpj,
          destination_body: setting&.destination_body, management: setting&.management, updated_at: setting&.updated_at&.iso8601 }
      end
  ```

  `app/commands/billing/save_settings.rb`:

  ```ruby
  # Configuração do cabeçalho do arquivo (contrato §6 com B10): admin municipal,
  # step-up no controlador. Larguras do layout (cbc-rsp 30, cbc-sgl 6, cbc-dst
  # 40) recusadas, nunca cortadas, com o campo; CNPJ com DV válido e só
  # numérico (cbc-cgccpf é NUM: o CNPJ alfanumérico da Receita não cabe);
  # indicador M/E. Linha única (singleton).
  module Billing
    module SaveSettings
      LIMITS = { "origin_name" => 30, "origin_acronym" => 6, "destination_body" => 40 }.freeze
      REQUIRED = %w[origin_name destination_body].freeze
      MANAGEMENT = %w[M E].freeze

      module_function

      def call(params:, by:)
        texts = LIMITS.keys.to_h { |field| [ field, params[field].is_a?(String) ? params[field].squish : "" ] }
        LIMITS.each do |field, limit|
          return Result.fail(:field_required, details: { field: field }) if REQUIRED.include?(field) && texts[field].empty?
          return Result.fail(:too_long, details: { field: field }) if texts[field].length > limit
        end
        cnpj = params["origin_cnpj"].is_a?(String) ? Cnpj.normalize(params["origin_cnpj"]) : nil
        return Result.fail(:invalid_cnpj) unless cnpj&.match?(/\A\d{14}\z/)
        return Result.fail(:invalid_management) unless MANAGEMENT.include?(params["management"])

        settings = nil
        ApplicationRecord.transaction do
          settings = BillingSetting.lock.find_by(singleton: true) || BillingSetting.new(singleton: true)
          settings.update!(origin_name: texts["origin_name"], origin_acronym: texts["origin_acronym"].presence, origin_cnpj: cnpj,
                           destination_body: texts["destination_body"], management: params["management"], updated_by_user: by)
          DomainEvents.publish("billing_settings.changed", billing_setting_id: settings.id, user_id: by.id)
        end
        Result.ok(settings: settings)
      rescue ActiveRecord::RecordNotUnique
        retry
      end
    end
  end
  ```

  `app/controllers/billing/periods_controller.rb`:

  ```ruby
  # /billing/periods (contrato §5 com B6, B8, B11): faturista e admin leem a
  # lista e as pendências (com trilha: tem paciente); só o faturista fecha e
  # reabre, com step-up. Ordem: papel → step-up → comando.
  module Billing
    class PeriodsController < BaseController
      before_action :require_reader
      before_action :load_period, except: :index

      def index
        competence = params[:competence].presence || Time.zone.today.strftime("%Y%m")
        return invalid_filter unless competence.is_a?(String) && competence.match?(Periods::COMPETENCE)

        periods = BillingPeriod.where(competence: competence).includes(:health_unit, :exam_provider, closed_by_user: :professional).to_a
        render json: { periods: periods.sort_by { |p| [ Establishment.of(p).name.to_s, p.id ] }.map { |p| Json.period(p) } }
      end

      def pending
        items = Pending.call(@period)
        DomainEvents.publish("billing_period.pending_viewed", billing_period_id: @period.id, user_id: Current.user.id)
        render json: { items: items }
      end

      def close
        return forbid unless Access.clerk?(Current.user)
        return require_step_up! unless reauthenticated_recently?

        result = Close.call(period: @period, by: Current.user)
        return render_failure(result) if result.failure?

        render json: Json.period(@period.reload)
      end

      def reopen
        return forbid unless Access.clerk?(Current.user)
        return require_step_up! unless reauthenticated_recently?

        result = Reopen.call(period: @period, reason: params[:reason], by: Current.user)
        return render_failure(result) if result.failure?

        render json: Json.period(@period.reload)
      end

      private

      def require_reader = (forbid unless Access.reader?(Current.user))

      def load_period
        @period = uuid?(params[:id]) ? BillingPeriod.find_by(id: params[:id]) : nil
        not_found unless @period
      end
    end
  end
  ```

  `app/controllers/billing/exports_controller.rb`:

  ```ruby
  # Versões do arquivo (contrato §5 com B9): a lista para faturista e admin;
  # o arquivo só para o faturista, com trilha (billing_export.downloaded) e
  # nome sem dado pessoal. O conteúdo nunca passa pelo log.
  module Billing
    class ExportsController < BaseController
      def index
        return forbid unless Access.reader?(Current.user)

        period = uuid?(params[:id]) ? BillingPeriod.find_by(id: params[:id]) : nil
        return not_found unless period

        exports = period.exports.includes(generated_by_user: :professional).order(version: :desc)
        render json: { exports: exports.map { |export| Json.export(export) } }
      end

      def file
        return forbid unless Access.clerk?(Current.user)

        export = uuid?(params[:id]) ? BillingExport.find_by(id: params[:id]) : nil
        return not_found unless export

        DomainEvents.publish("billing_export.downloaded", billing_export_id: export.id, user_id: Current.user.id)
        send_data export.bytes, type: "text/plain; charset=us-ascii", disposition: "attachment",
                                filename: "bpa-#{export.cnes}-#{export.competence}-v#{export.version}.txt"
      end
    end
  end
  ```

  `app/controllers/billing/settings_controller.rb`:

  ```ruby
  # /billing/settings (contrato §6 com B10): admin municipal; GET sem step-up
  # (nada sensível), PUT com step-up.
  module Billing
    class SettingsController < BaseController
      before_action { forbid unless Access.admin?(Current.user) }

      def show = render(json: Json.settings(BillingSetting.current))

      def update
        return require_step_up! unless reauthenticated_recently?

        result = SaveSettings.call(params: params.to_unsafe_h.slice(*%w[origin_name origin_acronym origin_cnpj destination_body management]),
                                   by: Current.user)
        return render_failure(result) if result.failure?

        render json: Json.settings(result.payload[:settings])
      end
    end
  end
  ```

  Em `config/routes.rb`, dentro do escopo `/billing` (depois de `post "procedures/:id/cancel"`):

  ```ruby
      get  "periods",              to: "periods#index"
      get  "periods/:id/pending",  to: "periods#pending"
      post "periods/:id/close",    to: "periods#close"
      post "periods/:id/reopen",   to: "periods#reopen"
      get  "periods/:id/exports",  to: "exports#index"
      get  "exports/:id/file",     to: "exports#file"
      get  "settings",             to: "settings#show"
      put  "settings",             to: "settings#update"
  ```

- [ ] **Step 4: Rode:** `rspec spec/requests/billing` → verde. Confira o log do teste: `grep -c "#BPA#" log/test.log` → 0 (o corpo do download não vai ao log).

- [ ] **Step 5: Commit:**

  ```bash
  git add app/controllers/billing/periods_controller.rb app/controllers/billing/exports_controller.rb app/controllers/billing/settings_controller.rb app/commands/billing/save_settings.rb app/services/billing/json.rb config/routes.rb spec/requests/billing/periods_spec.rb spec/requests/billing/settings_spec.rb
  git commit -m "feat: add billing competence, export download and settings routes

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

---

## Fatia 5 — Linha do tempo, LGPD, invariantes, semente e rollout

### Task 11: Procedimentos na linha do tempo e na exclusão LGPD (F-22.7)

**Files:**
- Create: `app/services/timeline/sources/procedure_source.rb`, `spec/services/timeline/sources/procedure_source_spec.rb`, `spec/commands/citizens/request_erasure_procedures_spec.rb`
- Modify: `app/services/timeline.rb` (`KINDS`), `app/services/timeline/query.rb` (`SOURCES`), `app/commands/citizens/request_erasure.rb` (`examined_patients?`)

**Interfaces:**
- Consumes: `Billing::Gate`, `Billing::Labels`, `Billing::Establishment` (Tasks 2, 4, 6); `Timeline::Sources::Window`, `Timeline::Entry`, `Timeline::Person`.
- Produces: `Timeline::KINDS` com `procedure` no fim; `Timeline::Sources::ProcedureSource.entries(person:, before:, limit:, from:, to:) -> [Timeline::Entry]` (kind `procedure`); `Citizens::RequestErasure.examined_patients?` conta `performed_procedures`.

- [ ] **Step 1: Teste que falha** — `spec/services/timeline/sources/procedure_source_spec.rb`:

  ```ruby
  require "rails_helper"

  # Contrato §9 (F-22.7; fecha api#71 para procedimentos): o profissional vê
  # nome, data, unidade e profissional; o cidadão vê nome e data, sem crítica
  # nem faturamento; cancelado não aparece; vazio sem procedure_billing ou
  # sem paciente; janela e cursor como os outros fornecedores.
  RSpec.describe Timeline::Sources::ProcedureSource do
    include ActiveSupport::Testing::TimeHelpers

    before do
      billing_city!
      Platform::Features.set!(city: Current.city, key: "patient_timeline", enabled: true, maintainer: ledi_maintainer!)
    end

    let(:unit) { upa! }
    let(:doctor) { doctor!(unit) }
    let(:patient) { billing_patient! }
    let(:person) { Timeline::Person.for_patient(patient) }

    def entries(**opts) = described_class.entries(person: person, before: nil, limit: 10, from: nil, to: nil, **opts)

    it "profissional e cidadão, do mais novo ao mais antigo, sem crítica nem faturamento" do
      older = travel_to(2.days.ago) { recorded_procedure!(unit: unit, professional: doctor, patient: patient, cid10: nil) }
      newer = recorded_procedure!(unit: unit, professional: doctor, patient: patient)
      list = entries
      expect(list.map(&:source_id)).to eq([ newer.id, older.id ])
      expect(list.first).to have_attributes(kind: "procedure", status: "recorded", unit: { id: unit.id, name: unit.name },
                                            professional: { name: doctor.professional.professional_name, cbo_code: "225125" },
                                            title: "ATENDIMENTO DE URGENCIA C/ OBSERVACAO ATE 24 HORAS EM ATENCAO ESPECIALIZADA",
                                            detail_ref: { type: "performed_procedure", id: newer.id }, document: nil)
      citizen = Timeline::Json.citizen(list.last)
      expect(citizen).to include(kind: "procedure", title: "ATENDIMENTO DE URGENCIA C/ OBSERVACAO ATE 24 HORAS EM ATENCAO ESPECIALIZADA",
                                 status: "recorded", status_label: "Realizado", professional_name: nil)
      expect(list.map { |e| [ e.to_h, Timeline::Json.citizen(e) ] }.to_s).not_to include("cid_required", "billable", "bpa_")
    end

    it "cancelado some; desligado ou sem paciente, vazio" do
      procedure = recorded_procedure!(unit: unit, professional: doctor, patient: patient)
      Billing::Cancel.call(procedure: procedure, reason: "Lançado em duplicidade", by: doctor)
      expect(entries).to eq([])
      recorded_procedure!(unit: unit, professional: doctor, patient: patient)
      expect(described_class.entries(person: Timeline::Person.new(citizen_ids: [], patient_id: nil, verified: true), before: nil, limit: 10,
                                     from: nil, to: nil)).to eq([])
      Platform::Features.set!(city: Current.city, key: "procedure_billing", enabled: false, maintainer: ledi_maintainer!)
      expect(entries).to eq([])
    end

    it "entra na consulta da linha do tempo e no laboratório mostra o laboratório como unidade" do
      lab = lab_provider!
      recorded_procedure!(unit: lab, professional: doctor, patient: patient, sigtap_code: "0202010473", cid10: nil)
      page = Timeline::Query.call(person: person, kinds: %w[procedure])
      expect(page.entries.sole.unit).to eq(id: lab.id, name: lab.name)
      expect(Timeline::KINDS.last).to eq("procedure")
    end
  end
  ```

  `spec/commands/citizens/request_erasure_procedures_spec.rb`:

  ```ruby
  require "rails_helper"

  # ADR 0039 (spec §8; regra única de api#84): procedimento realizado conta
  # como registro clínico na exclusão do cadastro.
  RSpec.describe Citizens::RequestErasure do
    before { billing_city! }

    it "quem tem procedimento realizado fica retido; sem ele, não" do
      unit = upa!
      patient, citizen = exam_patient!(exam_seq!)
      allow(described_class).to receive_messages(attended?: false, documented?: false)
      expect(described_class.retained?(Citizen.where(id: citizen.id), cpf: citizen.cpf)).to be(false)
      recorded_procedure!(unit: unit, professional: doctor!(unit), patient: patient)
      expect(described_class.retained?(Citizen.where(id: citizen.id), cpf: citizen.cpf)).to be(true)
    end
  end
  ```

  Em `spec/services/timeline/query_spec.rb` nada muda: `citizen_kinds(verified: true, documents: true)` continua igual a `Timeline::KINDS` (que ganha `procedure`).

- [ ] **Step 2: Rode e veja falhar:** `rspec spec/services/timeline/sources/procedure_source_spec.rb spec/commands/citizens/request_erasure_procedures_spec.rb` → `NameError: uninitialized constant Timeline::Sources::ProcedureSource`.

- [ ] **Step 3: Implemente.** `app/services/timeline/sources/procedure_source.rb`:

  ```ruby
  # Procedimentos realizados (ADR 0039; contrato §9; fecha api#71 para
  # procedimentos): um por procedimento gravado (performed_procedures.created_at),
  # cancelado fora. Profissional: nome SIGTAP, unidade (ou o laboratório) e
  # profissional com CBO; nunca crítica nem faturamento. Cidadão: nome e
  # situação "Realizado". Vazio sem procedure_billing ou sem paciente. A
  # trilha é do controlador (uma por leitura).
  module Timeline
    module Sources
      module ProcedureSource
        KIND = "procedure".freeze
        CITIZEN_STATUS = "Realizado".freeze

        module_function

        def entries(person:, before:, limit:, from:, to:)
          return [] unless person.patient_id && Billing::Gate.enabled?(Current.city)

          scope = ::PerformedProcedure.recorded.where(patient_id: person.patient_id)
                                      .includes(:health_unit, :exam_provider, professional_user: :professional)
          rows = Window.apply(scope, kind: KIND, column: "performed_procedures.created_at", id_column: "performed_procedures.id",
                                     before: before, from: from, to: to).limit(limit).to_a
          labels = {}
          rows.map { |procedure| entry(procedure, labels) }
        end

        def entry(procedure, labels)
          title = labels[[ procedure.sigtap_code, procedure.sigtap_competence ]] ||=
            Billing::Labels.procedure(procedure.sigtap_code, procedure.sigtap_competence)
          place = Billing::Establishment.of(procedure)
          Entry.new(kind: KIND, source_id: procedure.id, occurred_at: procedure.created_at, unit: { id: place.id, name: place.name },
                    professional: { name: Labels.professional_name(procedure.professional_user), cbo_code: procedure.cbo_code },
                    title: title, summary: "Realizado em #{I18n.l(procedure.performed_on, format: '%d/%m/%Y')}", status: "recorded",
                    detail_ref: { type: "performed_procedure", id: procedure.id },
                    citizen: { title: title, summary: nil, status: "recorded", status_label: CITIZEN_STATUS, professional_name: nil },
                    document: nil)
        end
        private_class_method :entry
      end
    end
  end
  ```

  Em `app/services/timeline.rb`: `KINDS = %w[triage attendance screening appointment consultation clinical_document controlled_notification campaign exam procedure].freeze`.

  Em `app/services/timeline/query.rb`, no fim de `SOURCES`: troque `"exam" => Sources::ExamSource` por `"exam" => Sources::ExamSource, "procedure" => Sources::ProcedureSource`.

  Em `app/commands/citizens/request_erasure.rb`, troque `examined_patients?` e o comentário de cima dele:

  ```ruby
      # ADR 0037: a requisição do laboratório municipal (com amostras, resultados
      # e laudos que pendem dela) também é registro clínico retido.
      # ADR 0039: o procedimento realizado (e o arquivo BPA que o leva) também.
      def examined_patients?(ids)
        ExamOrder.where(patient_id: ids).exists? || ExamResult.where(patient_id: ids).exists? ||
          LabRequisition.where(patient_id: ids).exists? || PerformedProcedure.where(patient_id: ids).exists?
      end
  ```

- [ ] **Step 4: Rode:** `rspec spec/services/timeline spec/requests/timeline_spec.rb spec/commands/citizens` → verde. (Se `I18n.l` reclamar do formato, use `procedure.performed_on.strftime("%d/%m/%Y")` — o resultado é o mesmo.)

- [ ] **Step 5: Commit e card:**

  ```bash
  git add app/services/timeline/sources/procedure_source.rb app/services/timeline.rb app/services/timeline/query.rb app/commands/citizens/request_erasure.rb spec/services/timeline/sources/procedure_source_spec.rb spec/commands/citizens/request_erasure_procedures_spec.rb
  git commit -m "feat: show performed procedures on the patient timeline and retain them on erasure

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```
  F-22.7 só tem o api: mova o card para **Done**.

---

### Task 12: Invariantes do ADR 0039 e varreduras de evento e log

**Files:**
- Create: `spec/invariants/procedure_billing_invariants_spec.rb`

**Interfaces:**
- Consumes: tudo das Tasks 3–11.
- Produces: nada novo (prova).

- [ ] **Step 1: Escreva a spec** `spec/invariants/procedure_billing_invariants_spec.rb`:

  ```ruby
  require "rails_helper"

  # ADR 0039, Invariantes, provados de ponta a ponta: UBS nunca gera BPA;
  # crítica nunca impede o registro; arquivo gerado nunca é alterado
  # (reabertura gera versão nova); arquivo e dado de paciente nunca em log nem
  # em evento; download sempre na trilha.
  RSpec.describe "Invariantes do faturamento (ADR 0039)", type: :request do
    before do
      billing_city!
      billing_settings!
    end

    let(:upa) { upa! }
    let(:ubs) { create_unit("UBS #{SecureRandom.hex(2)}").tap { |unit| unit.update!(cnes: billing_cnes) } }
    let(:doctor) { doctor!(upa).tap { |user| link_professional!(user, ubs, cbo: "225125") } }
    let(:clerk) { billing_clerk! }
    let(:patient) { billing_patient! }
    let(:app_files) { Dir[Rails.root.join("app/{commands,services,controllers,jobs}/**/billing/**/*.rb")] + Dir[Rails.root.join("app/services/timeline/sources/procedure_source.rb")] }

    # Corpo inteiro (multilinha) das chamadas `DomainEvents.publish(`...`)`, por balanceamento de parênteses.
    def publish_calls
      app_files.flat_map do |file|
        src = File.read(file)
        src.enum_for(:scan, "DomainEvents.publish(").map do
          i = start = Regexp.last_match.end(0)
          depth = 1
          while depth.positive? && i < src.length
            depth += 1 if src[i] == "("
            depth -= 1 if src[i] == ")"
            i += 1
          end
          [ file, src[start...(i - 1)] ]
        end
      end
    end

    it "UBS nunca gera BPA: grava (201) com establishment_is_aps, sem competência; o arquivo da UPA não leva a UBS" do
      sign_in_as(doctor)
      json_post "/billing/procedures", patient_id: patient.id, establishment: { type: "health_unit", id: ubs.id }, performed_on: Time.zone.today.iso8601,
                                       sigtap_code: "0301060029", quantity: 1, cid10: "I10"
      expect(response).to have_http_status(:created)
      expect(json_body).to include("billable" => false, "blocking_reasons" => %w[establishment_is_aps], "billing_competence" => nil)
      expect(BillingPeriod.where(health_unit_id: ubs.id)).to be_empty
      upa_procedure = recorded_procedure!(unit: upa, professional: doctor, patient: patient)
      export = Billing::Close.call(period: upa_procedure.billing_period, by: clerk).payload[:export]
      expect(export.bytes.split("\r\n").drop(1).map { |line| line[2, 7] }.uniq).to eq([ upa.cnes ])
      expect(export.bytes).not_to include(ubs.cnes)
    end

    it "crítica nunca impede: cada motivo grava 201" do
      sign_in_as(doctor)
      [ { cid10: nil }, { cid10: "N390" }, { sigtap_code: "0399999999" }, { sigtap_code: "0604320124", cid10: nil } ].each do |over|
        json_post "/billing/procedures", { patient_id: patient.id, establishment: { type: "health_unit", id: upa.id },
                                           performed_on: Time.zone.today.iso8601, sigtap_code: "0301060029", quantity: 1, cid10: "I10" }.merge(over)
        expect(response).to have_http_status(:created), over.inspect
        expect(json_body["billable"]).to be(false)
      end
    end

    it "arquivo nunca alterado: reabrir e fechar gera versão 2; a 1 continua igual e baixável" do
      procedure = recorded_procedure!(unit: upa, professional: doctor, patient: patient)
      period = procedure.billing_period
      v1 = Billing::Close.call(period: period, by: clerk).payload[:export]
      bytes = v1.bytes
      Billing::Reopen.call(period: period, reason: "Correção do CNS do médico", by: clerk)
      v2 = Billing::Close.call(period: period.reload, by: clerk).payload[:export]
      expect(v2.version).to eq(2)
      expect(v1.reload.bytes).to eq(bytes)
      expect { v1.update!(sha256: "f" * 64) }.to raise_error(ActiveRecord::StatementInvalid, /append-only/)
      sign_in_as(clerk)
      get "/billing/exports/#{v1.id}/file"
      expect(response.body.b).to eq(bytes)
      expect(DomainEvent.where(name: "billing_export.downloaded").count).to eq(1)
    end

    it "eventos do faturamento: só ids e os campos do contrato" do
      allowed = %w[performed_procedure_id billing_period_id billing_export_id billing_setting_id patient_id user_id source billable version]
      procedure = recorded_procedure!(unit: upa, professional: doctor, patient: patient)
      Billing::Close.call(period: procedure.billing_period, by: clerk)
      payload_keys = DomainEvent.where("name LIKE 'billing%' OR name LIKE 'performed_procedure%'").flat_map { |event| event.payload.keys }
      expect(payload_keys - allowed).to eq([])
      expect(publish_calls).not_to be_empty
      publish_calls.each do |file, call|
        expect(call).not_to match(/\b(cpf|cns|full_name|birth_date|file|bytes|reason|cancel_reason|reopened_reason|phone)\b/), "#{file}: #{call}"
      end
    end

    it "nenhum log no código do faturamento e nenhum inspect com conteúdo" do
      app_files.each { |file| expect(File.read(file)).not_to match(/Rails\.logger|puts |\bp\(/), file }
      procedure = recorded_procedure!(unit: upa, professional: doctor, patient: patient)
      export = Billing::Close.call(period: procedure.billing_period, by: clerk).payload[:export]
      expect([ export.inspect, procedure.inspect, procedure.billing_period.inspect ].join).not_to include(patient.cpf, "#BPA#")
    end
  end
  ```

- [ ] **Step 2: Rode:** `rspec spec/invariants/procedure_billing_invariants_spec.rb` → verde. Se a varredura de `publish_calls` acusar `reason`, confira que é o **nome do evento** e não um argumento (os eventos do faturamento não levam motivo).

- [ ] **Step 3: Commit:**

  ```bash
  git add spec/invariants/procedure_billing_invariants_spec.rb
  git commit -m "test: prove ADR 0039 billing invariants end to end

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

---

### Task 13: Semente de dev — UPA de Curitiba, produção, laboratório, configuração e faturista

**Files:**
- Create: `lib/billing_crew.rb`, `spec/lib/billing_crew_spec.rb`
- Modify: `lib/sigtap_sample.rb` (procedimentos da UPA, instrumentos e CBOs dos exames), `spec/services/terminology/sigtap_reader_spec.rb:20-21` (contagens), `db/seeds.rb` (require, fase de plataforma e fase da cidade)

**Interfaces:**
- Consumes: `Billing::Record`, `Billing::FromLab`, `Billing::SaveSettings`, `Billing::Establishment`, `Billing::Gate` (Tasks 2–10); `SignatureCrew.{ensure_totp,otpauth_uri}`; `Terminology::Import.call(kind:, version:, path:, by:)`; `SigtapSample.write_to(dir, competence:)`.
- Produces: `BillingCrew.seed_platform! -> { competences:, status: }`, `BillingCrew.seed_current_city(slug:, password:, admin:) -> { switch:, created: [String], users: [{ email:, role:, otpauth_uri: }] }`; `BillingCrew::{UPA,UBS,LAB,CLERK,BILLING_CODES,PRODUCTION,LAB_PRODUCTION}`.

**A semente (dev fictício que imita o real; só Curitiba; idempotente; depois do `LabCrew`):**

| O quê | Valor |
|---|---|
| Interruptor | `procedure_billing` ligado (mantenedor `dev@local`) |
| UPA | "UPA 24h Centro" (`kind: upa`, do `ProfessionalCrew`) com CNES `Zlib.crc32("cnes:curitiba:UPA 24h Centro") % 9_000_000 + 1_000_000` — o mesmo do retrato do CNES |
| Profissionais | `profissional@curitiba.demo` (Helena Duarte Moreira, CRM, CNS com DV válido; vínculo na UPA com CBO 225124 e na UBS com 225125); `tecnico@curitiba.demo` (Rafael Teixeira Lima, COREN; vínculo na UPA com 322205) — já criados pelo `ProfessionalCrew` |
| Laboratório | "Laboratório Municipal Central" (CNES do `ExamCrew`); `biomedica@curitiba.demo` (CRBM, vínculo 221205 → BPA-C faturável); `patologista@curitiba.demo` (CRM, sem vínculo → pendência `missing_professional_cbo`) — do `LabCrew` |
| Procedimentos faturáveis | UPA: `0301060029` com CID I10 e `0301060096` (BPA-I, médica); `0301100012` ×2 e `0214010015` ×3 (BPA-C, técnico) |
| Não faturáveis | UPA: `0301060029` sem CID (`cid_required`); `0301060096` pelo técnico (`cbo_not_allowed`); UBS Jardim das Flores: `0301010064` pela médica (`establishment_is_aps`, sem competência — a UBS nunca gera BPA) |
| Produção do laboratório | `0202020380`, `0202010473`, `0202010503` pela biomédica (BPA-C); `0202010317` pelo patologista (pendência); e, se houver laudos liberados no dev, os itens deles por `Billing::FromLab` |
| Datas | cada linha no dia 10 do mês anterior e hoje (duas competências: a anterior para fechar no navegador) |
| Pacientes | os três primeiros pacientes do banco (`Patient.order(:created_at)`), criados pelas sementes do 19/20; os dois primeiros ganham raça/cor e endereço (D-U1, só se vazios: parda/Rua XV de Novembro, 1200/Centro e indígena com etnia `0001`/Av. Sete de Setembro, SN/Rebouças); o terceiro fica sem — a prova mostra a pendência `missing_race_color`/`missing_address` e a correção pelo faturista |
| Configuração | `origin_name` "SMS Curitiba", `origin_acronym` "SMS", `origin_cnpj` o do `city_profile` se numérico, senão `11222333000181` (fictício), `destination_body` "SMS Curitiba - Controle e Avaliacao", `management` "M" |
| Faturista | `faturista@curitiba.demo` / senha de dev, papel `billing_clerk`, TOTP `DEV_BILLING_CLERK_OTP_SECRET` (padrão `MZQXI5LSNFZXIYJNMRSXMIJBEEQSCIJB`) |

- [ ] **Step 1: Recorte SIGTAP de dev.** Em `lib/sigtap_sample.rb`:
  - acrescente ao fim de `PROCEDURES` (antes do `].freeze`):

    ```ruby
      # Módulo 22 (ADR 0039): procedimentos da atenção especializada da UPA.
      [ "0301060029", "ATENDIMENTO DE URGENCIA C/ OBSERVACAO ATE 24 HORAS EM ATENCAO ESPECIALIZADA", "2", "I", 0, 9999 ],
      [ "0301060096", "ATENDIMENTO MEDICO EM UNIDADE DE PRONTO ATENDIMENTO", "2", "I", 0, 9999 ],
      [ "0301100012", "ADMINISTRACAO DE MEDICAMENTOS NA ATENCAO ESPECIALIZADA.", "2", "I", 0, 9999 ],
      [ "0301060118", "ACOLHIMENTO COM CLASSIFICACAO DE RISCO", "2", "I", 0, 9999 ],
      [ "0214010015", "GLICEMIA CAPILAR", "2", "I", 0, 9999 ]
    ```
  - depois de `AMOUNTS`, a constante `LAB_CBOS = %w[221205 223415].freeze` (biomédico e farmacêutico analista clínico);
  - troque `CBOS` por:

    ```ruby
    CBOS = {
      "0301010064" => %w[225125 225142 225130 225170],
      "0301010030" => %w[223505 251510],
      "0201020033" => %w[223505 225125],
      "0301100039" => %w[223505 322205 225125],
      "0301040079" => %w[223505 322205 225125],
      # Módulo 22: UPA e exames do laboratório municipal.
      "0301060029" => %w[225125 225124 225142],
      "0301060096" => %w[225125 225124 225142],
      "0301100012" => %w[223505 322205 225125],
      "0301060118" => %w[223505 225125],
      "0214010015" => %w[223505 322205]
    }.merge(AMOUNTS.keys.to_h { |code| [ code, LAB_CBOS ] }).freeze
    ```
  - troque `CIDS` por `CIDS = { "0201020033" => [ %w[Z014 S] ], "0301060029" => [ %w[I10 S], %w[E119 S], %w[J069 S] ] }.freeze`;
  - troque `INSTRUMENTS` por:

    ```ruby
    INSTRUMENTS = {
      "0301010064" => %w[02], "0301010030" => %w[02], "0201020033" => %w[01 02], "0301100039" => %w[01],
      "0301040079" => %w[01],
      # Módulo 22: BPA-I na consulta/urgência; BPA-C no consolidado e nos exames.
      "0301060029" => %w[02], "0301060096" => %w[02], "0301100012" => %w[01], "0301060118" => %w[01], "0214010015" => %w[01],
      "0203020030" => %w[02]
    }.merge(AMOUNTS.keys.to_h { |code| [ code, %w[01] ] }).freeze
    ```
  - acrescente ao comentário de topo: "Módulo 22: códigos e atributos de faturamento de dev (instrumento, CBO, CID) — a verdade é a competência importada do DATASUS."

  Em `spec/services/terminology/sigtap_reader_spec.rb:20-21`, as contagens passam a `sigtap_procedures: 28, sigtap_procedure_cbos: 59, sigtap_procedure_cids: 4, sigtap_procedure_instruments: 28` (5 procedimentos novos; CBOs 14 + 13 + 16×2; CIDs 1 + 3; instrumentos 6 + 5 + 1 + 16).

- [ ] **Step 2: Teste que falha** — `spec/lib/billing_crew_spec.rb`:

  ```ruby
  require "rails_helper"
  require Rails.root.join("lib/billing_crew").to_s

  # Semente de dev do módulo 22 (ADR 0039; dev imita o real): UPA de Curitiba
  # com CNES, produção faturável e não faturável na UPA, produção da UBS
  # (nunca BPA), do laboratório municipal (BPA-C; liberador sem vínculo =
  # pendência), configuração e faturista com TOTP. Idempotente; fora de
  # Curitiba não liga nada.
  RSpec.describe BillingCrew do
    include ActiveSupport::Testing::TimeHelpers

    before do
      travel_to(Time.zone.local(Time.zone.today.year, Time.zone.today.month, 15, 12, 0, 0))
      billing_city!(lab: true)
      Platform::Features.set!(city: Current.city, key: "procedure_billing", enabled: false, maintainer: ledi_maintainer!)
      Maintainer.find_by(email_address: "dev@local") ||
        Maintainer.create!(email_address: "dev@local", password: "dev-password-123", otp_secret: ROTP::Base32.random, otp_enabled_at: Time.current)
      upa = HealthUnit.create!(name: BillingCrew::UPA, kind: "upa")
      ubs = HealthUnit.create!(name: BillingCrew::UBS, kind: "ubs")
      doctor = staff_with("profissional@curitiba.demo", "health_professional")
      link_professional!(doctor, upa, cbo: "225124")
      link_professional!(doctor, ubs, cbo: "225125")
      link_professional!(staff_with("tecnico@curitiba.demo", "health_professional"), upa, cbo: "322205")
      lab = ExamProvider.create!(name: BillingCrew::LAB, kind: "municipal", uses_municipal_lab: true, cnes: "2385123")
      bio = lab_releaser!(lab, lead: true).tap { |u| u.update!(email_address: "biomedica@curitiba.demo") }
      link_professional!(bio, ubs, cbo: "221205")
      lab_releaser!(lab, council: "CRM", deputy: true).update!(email_address: "patologista@curitiba.demo")
      3.times { billing_patient!(demographics: false) }
    end

    after { travel_back }

    def seed = described_class.seed_current_city(slug: "curitiba", password: "dev-password", admin: lab_admin!)

    it "liga, dá CNES à UPA, semeia a produção e o faturista; a segunda rodada não muda nada" do
      result = seed
      upa = HealthUnit.find_by!(name: BillingCrew::UPA)
      expect(Billing::Gate.enabled?(Current.city)).to be(true)
      expect(upa.cnes).to eq((Zlib.crc32("cnes:curitiba:#{BillingCrew::UPA}") % 9_000_000 + 1_000_000).to_s)
      at_upa = PerformedProcedure.where(health_unit_id: upa.id)
      expect(at_upa.where(billable: true).distinct.pluck(:instrument)).to match_array(%w[bpa_i bpa_c])
      expect(at_upa.pluck(:blocking_reasons).flatten).to include("cid_required", "cbo_not_allowed")
      expect(at_upa.distinct.pluck(:billing_competence)).to match_array([ Time.zone.today.strftime("%Y%m"), (Time.zone.today << 1).strftime("%Y%m") ])
      ubs = PerformedProcedure.where(health_unit_id: HealthUnit.find_by!(name: BillingCrew::UBS).id)
      expect(ubs.pluck(:blocking_reasons).uniq).to eq([ %w[establishment_is_aps] ])
      expect(ubs.pluck(:billing_period_id).uniq).to eq([ nil ])
      lab = PerformedProcedure.where(exam_provider_id: ExamProvider.find_by!(name: BillingCrew::LAB).id)
      expect(lab.where(billable: true).pluck(:instrument).uniq).to eq(%w[bpa_c])
      expect(lab.pluck(:blocking_reasons).flatten).to include("missing_professional_cbo")
      clerk = User.find_by!(email_address: "faturista@curitiba.demo")
      expect(clerk.has_role?("billing_clerk")).to be(true)
      expect(clerk.reload.mfa_enrolled?).to be(true)
      expect(BillingSetting.current).to have_attributes(origin_name: "SMS Curitiba", origin_acronym: "SMS", management: "M")
      expect(result[:users].map { |u| u[:email] }).to eq([ "faturista@curitiba.demo" ])
      first, second, third = Patient.order(:created_at).limit(3).to_a
      expect([ first.race_color, second.race_color, second.ethnicity, third.race_color ]).to eq([ "03", "05", "0001", nil ])
      period = BillingPeriod.find_by!(health_unit_id: upa.id, competence: Time.zone.today.strftime("%Y%m"))
      expect(Billing::Pending.call(period).flat_map { |i| i[:reasons] }).to include("missing_race_color", "missing_address")
      expect { seed }.not_to(change { [ PerformedProcedure.count, BillingPeriod.count, Membership.count, BillingSetting.count ] })
    end

    it "fora de Curitiba não liga nada" do
      expect(described_class.seed_current_city(slug: "maringa", password: "x", admin: lab_admin!)[:switch]).to start_with("desligado")
      expect(PerformedProcedure.count).to eq(0)
    end
  end
  ```

- [ ] **Step 3: Rode e veja falhar:** `rspec spec/lib/billing_crew_spec.rb spec/services/terminology/sigtap_reader_spec.rb` → `LoadError` (crew) e contagens antigas (reader).

- [ ] **Step 4: Implemente** `lib/billing_crew.rb`:

  ```ruby
  require "tmpdir"
  require "zlib"
  require Rails.root.join("lib/sigtap_sample").to_s
  require_relative "signature_crew"

  # Semente de dev do módulo 22 (ADR 0039). Dev é fictício mas imita o real:
  # recorte SIGTAP de dev com instrumento, CBO e CID; a "UPA 24h Centro" ganha
  # o CNES do retrato do CNES (RecordModeCrew); produção da médica e do técnico
  # na UPA (faturável BPA-I e BPA-C, um sem CID, um com CBO não permitido), um
  # procedimento na UBS (nunca BPA), produção do laboratório municipal
  # (biomédica faturável; patologista sem vínculo = pendência de CBO), no dia
  # 10 do mês anterior e hoje; configuração do cabeçalho; faturista@ com
  # billing_clerk e TOTP. Só Curitiba. Idempotente. Roda depois do LabCrew.
  class BillingCrew
    DEV_MAINTAINER = "dev@local".freeze
    UPA = "UPA 24h Centro".freeze
    UBS = "UBS Jardim das Flores".freeze
    LAB = "Laboratório Municipal Central".freeze
    DEV_CNPJ = "11222333000181".freeze
    CLERK = { prefix: "faturista", role: "billing_clerk", secret_env: "DEV_BILLING_CLERK_OTP_SECRET",
              default_secret: "MZQXI5LSNFZXIYJNMRSXMIJBEEQSCIJB" }.freeze
    SETTINGS = { "origin_name" => "SMS Curitiba", "origin_acronym" => "SMS", "destination_body" => "SMS Curitiba - Controle e Avaliacao",
                 "management" => "M" }.freeze
    BILLING_CODES = %w[0301060029 0301060096 0301100012 0301060118 0214010015].freeze
    # [prefixo do e-mail de quem fez, estabelecimento, código SIGTAP, CID, quantidade]
    PRODUCTION = [
      [ "profissional", UPA, "0301060029", "I10", 1 ],   # BPA-I faturável
      [ "profissional", UPA, "0301060096", nil, 1 ],     # BPA-I faturável (sem CID na tabela)
      [ "profissional", UPA, "0301060029", nil, 1 ],     # cid_required
      [ "tecnico", UPA, "0301100012", nil, 2 ],          # BPA-C faturável
      [ "tecnico", UPA, "0214010015", nil, 3 ],          # BPA-C faturável
      [ "tecnico", UPA, "0301060096", nil, 1 ],          # cbo_not_allowed
      [ "profissional", UBS, "0301010064", nil, 1 ]      # establishment_is_aps: a UBS nunca gera BPA
    ].freeze
    LAB_PRODUCTION = [ %w[biomedica 0202020380], %w[biomedica 0202010473], %w[biomedica 0202010503], %w[patologista 0202010317] ].freeze
    # D-U1: raça/cor e endereço dos dois primeiros pacientes; o terceiro fica sem (pendência na prova).
    DEMOGRAPHICS = [
      { "race_color" => "03", "address_zip" => "80010000", "address_street" => "Rua XV de Novembro", "address_number" => "1200",
        "address_district" => "Centro" },
      { "race_color" => "05", "ethnicity" => "0001", "address_zip" => "80250210", "address_street" => "Av. Sete de Setembro",
        "address_number" => "SN", "address_complement" => "Bloco B", "address_district" => "Rebouças" }
    ].freeze

    class << self
      # Garante o recorte de dev com os códigos de faturamento na competência do
      # mês e na anterior. Uma SIGTAP real ativa nunca é tocada.
      def seed_platform!
        guard_environment!
        today = Time.zone.today
        competences = [ (today << 1).strftime("%Y%m"), today.strftime("%Y%m") ]
        return { competences: competences, status: :real_sigtap } if real_sigtap_active?

        imported = competences.select do |competence|
          release = TerminologyRelease.active.find_by(kind: "sigtap", version: competence)
          next false if release && SigtapProcedureInstrument.where(release_id: release.id, procedure_code: BILLING_CODES).count == BILLING_CODES.size

          Dir.mktmpdir do |dir|
            result = Terminology::Import.call(kind: "sigtap", version: competence, path: SigtapSample.write_to(dir, competence: competence), by: "db:seed")
            raise "semente do faturamento: SIGTAP recusada (#{result.reason})" if result.failure?
          end
          true
        end
        { competences: competences, status: imported.any? ? :republished : :complete }
      end

      def seed_current_city(slug:, password:, admin:)
        guard_environment!
        return { switch: "desligado (só Curitiba liga)", created: [], users: [] } unless slug == "curitiba"

        maintainer = Maintainer.find_by(email_address: DEV_MAINTAINER)
        return { switch: "desligado (sem mantenedor #{DEV_MAINTAINER})", created: [], users: [] } unless maintainer

        Platform::Features.set!(city: City.find(Current.city.id), key: Billing::Gate::KEY, enabled: true, maintainer: maintainer)
        upa = upa!(slug)
        clerk = clerk!(slug, password)
        settings!(admin)
        created = production!(slug, upa, clerk) + lab_production!(slug, clerk)
        { switch: "ligado (procedure_billing)", created: created, users: [ clerk.except(:user) ] }
      end

      private

      def guard_environment!
        raise "semente do faturamento: ambiente implantado; a semente é só de dev" if Rota.deployed?
      end

      def real_sigtap_active?
        TerminologyRelease.active.where(kind: "sigtap").pluck(:id).any? do |id|
          SigtapProcedure.where(release_id: id).count > SigtapSample::PROCEDURES.size
        end
      end

      def upa!(slug)
        unit = HealthUnit.find_by!(name: UPA)
        unit.update!(cnes: (Zlib.crc32("cnes:#{slug}:#{UPA}") % 9_000_000 + 1_000_000).to_s) if unit.cnes.nil?
        unit
      end

      def clerk!(slug, password)
        user = User.find_or_initialize_by(email_address: "#{CLERK[:prefix]}@#{slug}.demo")
        user.password = password if user.new_record?
        user.save!
        Membership.find_or_create_by!(user: user, role: CLERK[:role], revoked_at: nil) { |m| m.granted_at = Time.current }
        SignatureCrew.ensure_totp(user, secret_env: CLERK[:secret_env], default_secret: CLERK[:default_secret])
        { email: user.email_address, role: CLERK[:role], otpauth_uri: SignatureCrew.otpauth_uri(user), user: user }
      end

      def settings!(admin)
        return if BillingSetting.current

        cnpj = CityProfile.current&.cnpj.to_s
        params = SETTINGS.merge("origin_cnpj" => cnpj.match?(/\A\d{14}\z/) ? cnpj : DEV_CNPJ)
        result = Billing::SaveSettings.call(params: params, by: admin)
        raise "semente do faturamento: configuração recusada (#{result.reason})" if result.failure?
      end

      def dates = [ (Time.zone.today << 1).change(day: 10), Time.zone.today ]

      # Idempotente: só semeia a UPA/UBS se ainda não há procedimento nelas.
      def production!(slug, upa, clerk)
        ubs = HealthUnit.find_by!(name: UBS)
        return [] if PerformedProcedure.where(health_unit_id: [ upa.id, ubs.id ]).exists?

        patients = Patient.order(:created_at).limit(3).to_a
        return [ "sem paciente para a produção da UPA" ] if patients.empty?

        patients.first(2).zip(DEMOGRAPHICS).each do |patient, values|
          next if patient.race_color.present?

          saved = Patients::Demographics.save!(patient, values)
          raise "semente do faturamento: dados do paciente recusados (#{saved.reason})" if saved.failure?
        end
        units = { UPA => upa, UBS => ubs }
        count = 0
        dates.each do |on|
          PRODUCTION.each_with_index do |(prefix, unit_name, code, cid, quantity), index|
            professional = User.find_by!(email_address: "#{prefix}@#{slug}.demo")
            record!(patients[index % patients.size], Billing::Establishment.for_unit(units.fetch(unit_name)), professional, professional, code, cid,
                    quantity, on)
            count += 1
          end
        end
        [ "#{count} procedimentos na UPA e na UBS" ]
      end

      def lab_production!(slug, clerk)
        lab = ExamProvider.find_by!(name: LAB)
        released = LabReport.where(exam_provider_id: lab.id, status: "released", replaces_report_id: nil).sum { |report| Billing::FromLab.call(report) }
        return [ "#{released} itens de laudo do laboratório" ] if PerformedProcedure.where(exam_provider_id: lab.id, source: "manual").exists?

        patients = Patient.order(:created_at).limit(3).to_a
        return [ "sem paciente para a produção do laboratório" ] if patients.empty?

        establishment = Billing::Establishment.for_lab(lab)
        count = 0
        dates.each do |on|
          LAB_PRODUCTION.each_with_index do |(prefix, code), index|
            record!(patients[index % patients.size], establishment, User.find_by!(email_address: "#{prefix}@#{slug}.demo"), clerk[:user], code, nil, 1, on)
            count += 1
          end
        end
        [ "#{count} exames do laboratório", "#{released} itens de laudo do laboratório" ]
      end

      def record!(patient, establishment, professional, recorded_by, code, cid, quantity, on)
        result = Billing::Record.call(patient: patient, establishment: establishment, professional: professional, source: "manual",
                                      recorded_by: recorded_by, params: { "performed_on" => on.iso8601, "sigtap_code" => code,
                                                                          "quantity" => quantity, "cid10" => cid })
        raise "semente do faturamento: procedimento #{code} recusado (#{result.reason})" if result.failure?
      end
    end
  end
  ```

  (O retorno de `clerk!` leva `user:` só para uso interno; o resultado da semente não o expõe.)

  Em `db/seeds.rb`:
  - depois de `require Rails.root.join("lib/lab_qc_crew").to_s`: `require Rails.root.join("lib/billing_crew").to_s`;
  - depois do `puts "[seeds] catálogo-base laboratório …"`:

    ```ruby
      # Faturamento (módulo 22, ADR 0039): recorte SIGTAP de dev com instrumento,
      # CBO e CID no mês e no anterior; SIGTAP real ativa nunca é tocada.
      billing_sigtap = BillingCrew.seed_platform!
      puts "[seeds] SIGTAP faturamento . #{billing_sigtap[:competences].join(', ')} #{{ republished: 'republicada (recorte de dev)', real_sigtap: 'SIGTAP real ativa; não verificada', complete: 'já completa' }.fetch(billing_sigtap[:status])}"
    ```
  - depois do bloco do `LabQcCrew` (o `puts "[seeds] qualidade ..."`):

    ```ruby
            # ── Procedimentos e faturamento BPA (módulo 22, ADR 0039) ─────────────
            # Depois do LabCrew (laboratório e liberadores) e do ProfessionalCrew (UPA e vínculos).
            billing = BillingCrew.seed_current_city(slug: slug, password: password, admin: muni_admin)
            puts "[seeds] faturamento . #{billing[:switch]}#{"; #{billing[:created].join(', ')}" if billing[:created].any?}"
            Array(billing[:users]).each { |u| puts "[seeds]               #{u[:email]} (#{u[:role]}) / #{password} → #{u[:otpauth_uri]}" }
    ```

- [ ] **Step 5: Rode:** `rspec spec/lib/billing_crew_spec.rb spec/services/terminology/sigtap_reader_spec.rb spec/lib/lab_crew_spec.rb spec/lib/exam_crew_spec.rb spec/invariants/record_mode_invariants_spec.rb` → verde.

- [ ] **Step 6: Commit:**

  ```bash
  git add lib/billing_crew.rb lib/sigtap_sample.rb db/seeds.rb spec/lib/billing_crew_spec.rb spec/services/terminology/sigtap_reader_spec.rb
  git commit -m "chore: seed billing dev data for Curitiba with UPA production and billing clerk

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

---

### Task 14: Revisão final, suíte completa, api na porta 3044 e runbook de rollout

**Files:**
- Create: `docs/operacao/rollout-faturamento-bpa.md` (repo `docs`)

- [ ] **Step 1: Varreduras** e lint:

  ```bash
  docker compose exec -T -w /rails/.claude/mod22 api bin/rubocop
  git grep -n "Current.city =" -- app lib
  git grep -n "DomainEvents.publish(" -- app/commands/billing app/controllers/billing app/jobs/billing app/services/billing
  ```
  Expected: rubocop limpo; o `Current.city =` não aparece em `app/` nem `lib/`; cada `publish(` (leia a chamada inteira, é multilinha) só com ids, `source`, `billable`, `version`.

- [ ] **Step 2: Suíte completa** com o worker parado e sem outra sessão rodando suíte:

  ```bash
  docker compose stop worker
  docker compose exec -T -e ROTA_TEST_DB_SUFFIX=_mod22 -w /rails/.claude/mod22 api bundle exec rspec
  docker compose start worker
  ```
  Expected: verde; ~0,10 s por exemplo com o host calmo (memória "BCrypt e custo em teste": acima de 0,15 s investigue).

- [ ] **Step 3: api do worktree na porta 3044** e semente:

  ```bash
  docker compose exec -d -w /rails/.claude/mod22 api bin/rails server -b 0.0.0.0 -p 3044
  docker compose exec -T -w /rails/.claude/mod22 api bin/rails city:migrate:all
  docker compose exec -T -w /rails/.claude/mod22 api bin/rails db:seed
  docker compose exec -T api curl -s -o /dev/null -w '%{http_code}\n' -H "Host: curitiba.localhost" http://localhost:3044/up
  ```
  Expected: `200`; o `db:seed` imprime `[seeds] SIGTAP faturamento . <AAAAMM anterior>, <AAAAMM> …` e `[seeds] faturamento . ligado (procedure_billing); 14 procedimentos na UPA e na UBS, 8 exames do laboratório, 0 itens de laudo do laboratório` e a linha do `faturista@curitiba.demo (billing_clerk)` com o `otpauth://`. O dashboard do worktree (`.worktrees/dashboard-mod22`, Vite 5197) aponta o proxy para `http://api:3044` com a entrada `/billing`.

- [ ] **Step 4: Conferência pelo api** (a prova no navegador é a Task 14 do plano do dashboard, com estas contas): com `faturista@curitiba.demo` (senha de dev + TOTP), `GET /billing/periods?competence=<AAAAMM anterior>` lista a UPA (BPA-I e BPA-C, pendências `cid_required`/`cbo_not_allowed`) e o laboratório (pendência `missing_professional_cbo`), e **não** lista a UBS; fechar a UPA com step-up gera a versão 1; o download sai `bpa-<cnes>-<AAAAMM>-v1.txt` com linhas de 130/48/351 posições + CR+LF (`awk '{print length($0)}' arquivo | sort | uniq -c` mostra 131, 49 e 352). Guarde uma cópia do arquivo de dev para a importação real (gate de go-live, abaixo). Falha de api → corrija com teste antes.

- [ ] **Step 5: Runbook** `docs/operacao/rollout-faturamento-bpa.md` (repo docs):

  ```markdown
  # Rollout — procedimentos e faturamento BPA (módulo 22, ADR 0039)

  ## Pré-requisitos
  - SIGTAP da competência importada na plataforma (módulo 16; `terminology:import` com o ZIP do DATASUS) — sem ela,
    a crítica usa a última ativa (aviso `sigtap_fallback`) ou marca `not_in_sigtap`.
  - Cada unidade não-APS (UPA, hospital, outro) com CNES no cadastro (módulo 09); o laboratório municipal com CNES (21a/21b).
  - Profissionais com perfil (CNS com DV válido) e vínculo com CBO na unidade (módulo 10); liberadores do laboratório
    com vínculo em alguma unidade (o CBO do laudo vem dele).
  - IBGE da cidade no perfil (`city_profile.ibge_code`) — sem ele o interruptor fica "ligado, falta pré-requisito" e o
    BPA-I sai com o IBGE em brancos.
  - Tag do contracts publicada: `billing-v1.0.0` (perfil e exemplos golden, copiados em `apps/api/spec/fixtures/billing/`).
  - Nenhuma gem nova; nenhuma migração de plataforma.

  ## Ordem de deploy
  1. contracts (tag `billing-v1.0.0`) → 2. api: publicar a imagem e rodar `bin/migrate` dela ANTES de cortar tráfego
     (`city:migrate:all` — 20261021900001, irreversível; papel `billing_clerk`, `performed_procedures`, `billing_periods`,
     `billing_exports`, `billing_settings` e triggers em `db/city_triggers.sql`).
  3. dashboard → 4. maintenance → 5. catálogo público. Nunca migrar cidade fora do rake.

  ## Ligar numa cidade
  1. maintenance: `procedure_billing` ("Faturamento de procedimentos (BPA)"); vale nos dois modos de prontuário.
  2. Admin municipal: configuração do faturamento (órgão de origem até 30, sigla até 6, CNPJ só numérico, órgão de
     destino até 40, indicador M/E) — com step-up.
  3. Admin municipal: convidar/conceder `billing_clerk` (papel privilegiado: step-up; o mantenedor não concede).
  4. Recepção: ao validar o par ou completar o cadastro, preencher raça/cor (sem "sem informação"; etnia se indígena) e
     endereço (CEP, logradouro, número ou SN, complemento, bairro) — D-U1. Quem não tem fica pendente no fechamento
     (`missing_race_color`, `missing_address`) e o faturista corrige em "dados do paciente".
  5. A partir daí: procedimentos no atendimento, na consulta e avulsos; os laudos liberados do laboratório municipal
     viram produção automaticamente. **Sem retroativo:** laudo liberado com o interruptor desligado não entra; procedimento
     anterior ao interruptor entra só por lançamento manual do faturista.

  ## Rotina mensal do faturista
  1. Conferir o `BPA_LEIAME.txt` vigente em https://sia.datasus.gov.br/versao/listar_ftp_bpa.php antes de fechar: versão
     nova do BPA Magnético com mudança de layout (posição/tamanho de campo) **para o fechamento** até o api ser atualizado
     (o layout implementado é o de 09/07/2026, BPA 05.00 — SHA-256 do PDF no perfil do contracts).
  2. Pendências da competência (por CNES); corrigir cadastro (CNS, CBO, CNES, perfil do paciente) — o fechamento recalcula.
  3. Fechar (step-up); baixar o arquivo; importar no BPA Magnético ("importar arquivos de sistemas próprios"); consistir
     e exportar a remessa para a secretaria.
  4. Erro no arquivo: reabrir com motivo (gera nova versão; a anterior fica no histórico). Procedimento registrado
     depois do fechamento com data da competência fechada entra na próxima, como atrasado (até 3 competências anteriores).

  ## Gates de go-live (spec §12)
  - **Importação real no BPA Magnético** (VM Windows ou secretaria piloto) de um arquivo gerado em staging: sem rejeição
    de layout; registrar o resultado da consistência. Pontos a conferir ali: `prd-org` `BPA` × `EXT` (F6), código de
    logradouro em brancos, INE em brancos em unidade sem equipe. Raça/cor e endereço vêm do cadastro (D-U1): a
    recepção ou o faturista completam; quem não tem fica pendente (`missing_race_color`, `missing_address`).
  - CNS e CBO de todos os profissionais e CNES de todas as unidades faturantes (módulos 09 e 10).
  - Transição para RAC/CMD (RNDS) sem data: o BPA é ponte.
  - Volume: mais de 999 folhas (19.980 linhas) num estabelecimento recusa o fechamento (`too_many_sheets`); acompanhar
    UPAs grandes no primeiro mês (BPA-I aceita 99 linhas por folha no layout — mudança a decidir se acontecer).

  ## Verificação depois de ligar
  - Um procedimento de teste numa UPA (faturável) e numa UBS (`establishment_is_aps`, sem competência).
  - `GET /billing/periods` com o faturista; pendências; nenhum CPF/nome/arquivo no log do dia (`grep` do CPF de teste).

  ## Desligar
  - `procedure_billing` desligado: rotas `/billing` respondem 403; atendimento e consulta não mostram procedimentos; a
    linha do tempo para de mostrar o tipo `procedure`; laudos liberados deixam de virar produção. Os registros e arquivos
    ficam (só acréscimo).
  - Migração irreversível: não há rollback de schema; volte só o código (expand/contract).
  ```
  Commit no repo docs (sem push):

  ```bash
  /opt/homebrew/bin/git -C docs add operacao/rollout-faturamento-bpa.md
  /opt/homebrew/bin/git -C docs commit -m "docs: add BPA billing rollout runbook (module 22)

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

- [ ] **Step 6: Fechamento.** Pare o servidor da 3044; apague os bancos de teste `_mod22` (conferindo conexões e sessões ativas — memória "Apagar bancos com sufixo de módulo"); confira os cards: F-22.3 e F-22.7 em **Done**; F-22.1, F-22.2, F-22.4, F-22.5, F-22.6 continuam **In Progress** até o dashboard/maintenance/contracts deles. Merge, push e a tag `billing-v1.0.0` **só com autorização do usuário**, uma etapa de cada vez (memória "Sessões paralelas no push": `origin/main..main` antes de push).

---

## Self-review (feito ao escrever o plano)

- **Cobertura da spec:** §2 decisão 1 (escopo) → Tasks 4–10; decisão 2 (UBS nunca; instrumento BPA) → Task 4 (crítica), Task 5 (sem competência), Task 9 (fechamento recusa), Task 12 (ponta a ponta); decisão 3 (origens) → Tasks 5, 6, 7; decisão 4 (marca e não bloqueia) → Tasks 4, 6, 12; decisão 5 (faturista, reabrir, trilha) → Tasks 3, 9, 10; decisão 6 (CPF, nunca os dois) → Task 8; decisão 7 (layout da Task 0, golden) → Tasks 0, 1, 8; decisão 8 (sem informação, IBGE, contato do wpda, pendência) → Tasks 4, 8, 9; decisões do coordenador (crítica recalculada, estabelecimento por CNES, cabeçalho como configuração, SIGTAP ausente com aviso) → Tasks 4, 9, 10; §3 interruptor e papel → Tasks 2, 3; §4 dados → Task 3; §5.1–§5.6 → Tasks 5, 6, 7, 9, 10; §6 arquivo → Tasks 1, 8, 9; §7 linha do tempo → Task 11; §8 segurança e LGPD → Tasks 3, 10, 11, 12; §9 testes → cada task (golden byte a byte BPA-I e BPA-C com folhas de 20 e campo de controle na Task 8; UBS nunca gera BPA nas Tasks 4, 5, 9 e 12); §10 entrega → Tasks 0, 1, 14; semente → Task 13; F-22.1..F-22.7 → mapa em "Ambiente de execução".
- **Placeholders:** nenhum. A Task 0 tem regra de parada explícita (SHA-256 do PDF).
- **Tipos e nomes:** `Billing::Establishment.{for_unit,for_lab,find,of}`, `Billing::Staff.{cbo_for,linked?,cns,cbos_for,eligible?,non_aps_units}`, `Billing::Critic.call(Facts)` → `Outcome`, `Billing::Periods.assign!`, `Billing::Record.{call,insert!}`, `Billing::Cancel.call`, `Billing::FromLab.call`, `Billing::BpaFile.build` (`Header` com `origin_acronym`), `Billing::Lines.build`, `Billing::Recalculate.{call,outcome}`, `Billing::Pending.call`, `Billing::Summary.call`, `Billing::Close.call`, `Billing::Reopen.call`, `Billing::SaveSettings.call`, `Billing::Json.{procedure,procedures,summary,attached,period,export,settings}`, helpers `billing_sigtap!`, `billing_city!`, `upa!`, `billing_cnes`, `billing_clerk!`, `billing_settings!`, `billing_patient!`, `raw_procedure!`, `recorded_procedure!` — iguais em todas as tasks.
- **Prova do gerador:** o oráculo Python (Task 1) e o `Billing::BpaFile` (Task 8) foram rodados ao escrever o plano: os três exemplos saem com os SHA-256 fixados e o gerador Ruby bate byte a byte, inclusive a recusa de 1000 folhas.
- **D-U1 (raça/cor e endereço):** colunas na Task 3; crítica de fechamento na Task 4; escrita da recepção, do faturista e exclusão na Task 8b; gerador e golden (regerados com raça/cor e endereço) nas Tasks 1 e 8; fechamento na Task 9; semente com e sem os dados na Task 13.
- **Review Focus:** 1 → Task 5 ("atraso de competência") e Task 9 ("atrasado no arquivo"); 2 → Task 9 ("cadastro corrigido basta"); 3 → Task 8 (golden BPA-I e "só ASCII…"); 4 → Task 9 (`close_concurrency_spec.rb`); 5 → Task 10 ("recusas: campos, larguras, CNPJ alfanumérico e indicador").

## Divergências propostas ao contrato

Numeradas como **F** (do plano do api). As B do dashboard e as M do maintenance estão na tabela logo abaixo, com a decisão deste plano.

1. **F1 — Raça/cor: decidida (D-U1, contrato §12, docs fdb2c98).** `patients.race_color` (`01`..`05`, sem `99`) e `ethnicity` (obrigatória quando `05`), cifrados; preenchidos na recepção ou pelo faturista; ausente = pendência `missing_race_color` (`fix: patient`) no fechamento. Deixa de ser gate.
2. **F2 — Endereço: decidido (D-U1).** `patients.address_{zip,street,number,complement,district}`, cifrados; incompleto (sem CEP, endereço, número ou bairro) = pendência `missing_address` no fechamento. O município de residência continua o da cidade; o código de logradouro (DNE) segue em brancos. Deixa de ser gate.
3. **F3 — Origem `manual` para o avulso do profissional e para o lançamento do faturista**; `recorded_by_user_id` diz quem gravou (contrato §1 só tem quatro origens). Com B5: a presença de `professional_user_id` (só do faturista) marca o lançamento.
4. **F4 — Data:** `invalid_date` fora de [1º dia da competência de 3 meses antes, hoje] (regra do BPA 04.13), em vez de "antes da competência mais antiga aberta" (§3). O laboratório grava a data da coleta sem essa janela.
5. **F5 — `sex_not_allowed` mantido** como o contrato manda, embora o BPA Mag tenha deixado de impedir sexo diferente do sugerido desde 2018 (LEIAME 02.89). Proposta: rebaixar a aviso numa revisão.
6. **F6 — `prd-org` = `BPA`** (o domínio também tem `EXT`, "outros sistemas"); conferir na importação real.
7. **F7 — BPA-I e BPA-C listados na SIGTAP:** vale BPA-I (um instrumento por procedimento por estabelecimento na competência, Portaria 1.362/2013).
8. **F8 — Folha de 20 também no BPA-I** (o layout aceita 01..99); limite de 999 folhas → 422 `too_many_sheets` (recusa, nunca corte). Folhas numa série só, BPA-C primeiro, folha nova a cada profissional no BPA-I.
9. **F9 — Códigos de erro novos:** `invalid_character`, `invalid_patient` (B1), `invalid_professional` (B5), `reason_required` (B8), `already_exported` (409, B8), `too_many_sheets`, `establishment_is_aps` (422 no fechamento), `invalid_filter`, `invalid_cursor`, `invalid_query`, `invalid_cpf` e `patient_not_found` (404, B3), `terminology_unavailable` (503 na busca), `field_required` e `too_long` com `field` (B10), `citizen_not_verified` (atendimento de par não validado, vindo do `Patients::Resolve`).
10. **F10 — Eventos a mais (só ids):** `billing_period.pending_viewed` (`billing_period_id`, `user_id`; trilha das pendências, §5) e `billing_patient.looked_up` (`patient_id`, `user_id`; B3); `billing_export.downloaded` e `billing_settings.changed` levam `user_id`.
11. **F11 — Formas:** `<performed_procedure>` com `late` (B7) e `source_ref = { type, id }` (`attendance`, `consultation`, `lab_report_item`); `<billing_period>` com `closed_by` = nome; `summary.bpa_c_lines` na competência aberta conta itens faturáveis (ainda não agregados por idade), na fechada as linhas do arquivo; resumo `procedures` no atendimento/consulta = `{ id, sigtap_code, label, quantity, performed_on, billable, blocking_reasons, status }`.
12. **F12 — Busca SIGTAP:** `instruments` em `bpa_i|bpa_c|not_bpa`; `q` 2..60 (dígitos = início do código; texto = nome); até 20.
13. **F13 — Linha do tempo:** fornecedor `Timeline::Sources::ProcedureSource` (convenção do código; o §9 diz `Procedures`), `occurred_at` = gravação; o cidadão vê `status_label` "Realizado"; cancelado não aparece.
14. **F14 — Interruptor lido só na plataforma** (`enabled?`): com M1 o `requires` (IBGE) só informa o maintenance; as rotas valem ligadas.
15. **F15 — Nome do paciente cortado em 30** no arquivo (limite do layout; não é entrada do Rota Saúde). Texto do arquivo em maiúsculas ASCII só com letras e espaço.

### Formas do dashboard (B1–B13) e do maintenance (M1–M3)

| Item | Decisão | Onde |
|---|---|---|
| B1 paciente derivado do atendimento/consulta | adotada; `patient_id` que não bate → 422 `invalid_patient`; par validado sem paciente → `Patients::Resolve` (par não validado → 422 `citizen_not_verified`) | Task 6 |
| B2 `GET /billing/procedures?attendance_id=` | adotada (profissional com vínculo na unidade do atendimento; vê todos os do atendimento); o `procedures` embutido no atendimento/consulta fica | Task 6 |
| B3 `POST /billing/patient_lookup` | adotada (faturista ou profissional com vínculo em unidade não-APS; trilha `billing_patient.looked_up`; 404 `patient_not_found`; **a mais:** 422 `invalid_cpf`) | Task 6 |
| B4 `GET /billing/establishments` | adotada (faturista e admin) | Task 6 |
| B5 `professional_user_id`/`cbo_code` só do faturista; `GET /billing/professionals` | adotada; profissional mandando esses campos → 403; `cbo_code` precisa ser um dos CBOs do vínculo (senão `invalid_professional`); no laboratório "quem fez" é liberador ativo dele | Task 6 |
| B6 pendência com `performed_on`, `sigtap_code`, `label`, `patient`, `professional` | adotada | Task 9 |
| B7 `late` | adotada (já estava) | Task 6 |
| B8 `already_exported`, motivo 10..500 `reason_required`, 409 `invalid_transition` | adotada (troca o `period_closed` e o `invalid_reason` do rascunho do api) | Tasks 5, 9, 10 |
| B9 `{ exports }`, `generated_by` nome, `bpa-<cnes>-<AAAAMM>-v<n>.txt`, lista faturista+admin, arquivo só faturista | adotada (troca o `BPA_…` do rascunho do api) | Task 10 |
| B10 `origin_acronym`, `too_long`, GET 200 com nulls, step-up só no PUT | adotada: `origin_acronym` é o `cbc-sgl` (6, posições 60–65) do layout lido; **a mais:** `field_required` com `field` para nome e destino vazios | Tasks 3, 8, 10 |
| B11 `id` em toda competência, `version` 0 antes do 1º fechamento, admin lê competências e pendências | adotada | Tasks 5, 10 |
| B12 profissional cancela os próprios; faturista qualquer um | adotada ("próprio" = o profissional que fez) | Task 6 |
| B13 dashboard não oferece UBS | concordo; o api segue aceitando e marcando `establishment_is_aps` | Tasks 4, 12 |
| M1 `requires: ibge_code` | adotada (`ibge_code_missing`), sem pré-requisito de modo | Task 2 |
| M2 todos os ambientes, sem simulado | adotada (sem `environments`) | Task 2 |
| M3 rótulo "Faturamento de procedimentos (BPA)" | adotada no `description` do catálogo (o rótulo é dos fronts) | Task 2 |
