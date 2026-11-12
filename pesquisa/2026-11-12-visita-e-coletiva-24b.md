# Pesquisa — Visita domiciliar e territorial e atividade coletiva (módulo 24b)

Data da pesquisa: 2026-11-12 (todas as fontes acessadas nesta data). [FATO] = com fonte; [INFERÊNCIA] = leitura nossa.
Numeração das fontes: ver seção 8 (ex.: [F3]). Caminhos de `vendor/` são do `apps/api`.

## 1. Resumo

- [FATO] Códigos de `tipoDadoSerializado`: **Visita Domiciliar e Territorial (MIVDT) = 8**, **Atividade Coletiva (MIAC) = 6**. O 13 é a ficha complementar Zika/Microcefalia, não a visita. [F1]
- [FATO] O IDL e o Ruby gerado já estão vendorizados: `idl/ras/ficha_visita_domiciliar.thrift`, `idl/ras/ficha_atividade_coletiva.thrift` e os `_types.rb` em `gen-rb/` (fonte `laboratoriobridge/esusaps-integracao@9316f280f5`, Thrift 0.17.0, com SHA-256 no `SOURCE`). Os módulos Ruby são `Br::Gov::Saude::Esusab::Ras::Visitadomiciliar` e `...::Atividadecoletiva` (conferido no `gen-rb` em 2026-11-12). [F2]
- [FATO] Os dicionários de campo das duas fichas **não estão vendorizados**: em `vendor/ledi/8.7.0/docs/` só há fai, fao, fcd e fci. Faltam `dicionario-fvd.html` ("Alterado em 11/09/2026") e `dicionario-fac.html`. Eles ficam em `.../ledi/documentacao/estrutura_arquivos/`; sem esse subcaminho a URL dá 404, embora o `SOURCE` registre só `.../ledi/documentacao/`. [F3][F4]
- [FATO] As enumerações (MotivoVisita, Desfecho, tipoDeImovel, TipoAtividadeColetiva, TemasParaReuniao, TemasParaSaude, PraticasEmSaude, PublicoAlvo, Turno, Sexo, tipoGlicemiaCapilar, JustificativaNaoPossuiCpf) já estão no `docs/dicionario.html` vendorizado. [F1]
- [FATO] Quem pode registrar: `docs/cbo.html` vendorizado. MIVDT = Tabela 7; MIAC = Tabela 5. [F5]
- [FATO] **MIVDT:** uma ficha = um profissional + uma data + até 99 visitas. Desfecho, turno e tipo de imóvel obrigatórios; motivo e medidas proibidos em visita recusada ou de ausente; imóvel não residencial sem dado de cidadão. [F3]
- [FATO] **MIAC:** uma ficha = uma atividade, com regras cruzadas entre tipo, temas, público e práticas; participantes obrigatórios nos tipos 5 e 6; CPF é a identificação primária do participante; PSE exige INEP. [F4]
- [FATO] MIVDT e MIAC são **práticas de cuidado** na dimensão de Acompanhamento (70%) do componente de Vínculo e Acompanhamento Territorial. [F6]
- [FATO] A C5 (hipertensão) pede 2 visitas de ACS/TACS em 12 meses, com 30 dias de intervalo e o motivo preenchido. [F7]

## 2. Ficha de visita domiciliar e territorial (MIVDT, tipo 8)

**Estrutura** [F2][F3]
- [FATO] `FichaVisitaDomiciliarMasterThrift`: `uuidFicha` (36–44, recomendado `CNES-uuid`), `tpCdsOrigem` = 3 (sistemas terceiros), `headerTransport` (`UnicaLotacaoHeader`, CBO da Tabela 7) e `visitasDomiciliares` (0 a **99**).
- [FATO] No `FichaVisitaDomiciliarChildThrift` todos os campos são `optional` na IDL; a obrigatoriedade vem do dicionário.
- [INFERÊNCIA] Uma ficha por visita também é válida pela IDL. Se o PEC aceita várias fichas do mesmo profissional no mesmo dia não foi testado.

**Campos da visita** [F3]

| # | campo | obrig. | regras principais |
|---|---|---|---|
| 1 | turno | Sim | 1 Manhã, 2 Tarde, 3 Noite |
| 2 | numProntuario | Não (≤30) | proibido em imóvel não residencial |
| 3 | cnsCidadao | Não (15) | algoritmo do CNS; exclusivo com o CPF; em visita periódica e controle vetorial, o CNS do **responsável familiar** |
| 4 | dtNascimento | Condicional | ≤ data do atendimento e ≤130 anos; obrigatório em imóvel residencial se houver CNS, sexo, motivo de BUSCA_ATIVA ou ACOMPANHAMENTO, motivo 25 ou 31, ou peso/altura |
| 5 | sexo | Condicional | mesma regra do nascimento (0 M, 1 F, 4 Ignorado, 5 Indeterminado) |
| 6 | statusVisitaCompartilhadaOutroProfissional | Não | booleano |
| 7 | motivosVisita | Condicional (0..36) | proibido com desfecho 2 ou 3; em não residencial só 1, 27, 28, 31, 34, 35, 36, 37; 5 e 6 proibidos para sexo 0 |
| 8 | desfecho | **Sim** | 1 Realizada, 2 Recusada, 3 Ausente |
| 9 | microArea | Condicional (2) | proibido se fora de área; senão obrigatório |
| 10 | stForaArea | Não | booleano |
| 11 | tipoDeImovel | **Sim** | ver domínio |
| 12/13 | peso / altura | Não | 0,5–500 kg (3 casas) / 20–250 cm (1 casa); proibidos em não residencial e com desfecho 2/3 |
| 14 | cpfCidadao | Não (11) | CPF válido; exclusivo com o CNS; responsável familiar na visita periódica e vetorial |
| 15/16 | pressão sistólica / diastólica | Não | 0–999, uma exige a outra; mesmas proibições |
| 17 | temperatura | Não | 20–45 (1 casa); mesmas proibições |
| 18/19 | tipoGlicemia / glicemia | Condicional | um exige o outro; 0–800 mg/dL |
| 20/21 | latitude / longitude | Condicional | um exige o outro |
| 22 | uuidOrigemCadastroDomiciliar | Não | uuid de origem do cadastro domiciliar |
| 23 | ivcf | Não | só com 60 anos ou mais; resultado 0–40, 13 booleanos e data |
| 24/25 | stCidadaoNaoPossuiCpf / justificativa | Condicional | justificativa obrigatória com o marcador, proibida sem; códigos 2–7 |

- [FATO] O dicionário chama o #25 de `justificativaNaoPossuiCpf` e a IDL de `justificativaCidadaoNaoPossuiCpf`; o dicionário também inverte a numeração de tipoGlicemia e glicemia. Vale a IDL, que é o que serializa.
- [FATO] **Imóvel não residencial:** 2 Comércio, 3 Terreno baldio, 4 Ponto estratégico, 5 Escola, 6 Creche, 12 Estabelecimento religioso, 13 CASAI. Neles são proibidos todos os dados do cidadão (prontuário, CNS, CPF, nascimento, sexo, medidas). **Residencial:** 1 Domicílio, 7 Abrigo, 8 ILPI, 9 Unidade prisional, 10 Medida socioeducativa, 11 Delegacia, 99 Outros.

**Domínios** [F1]
- [FATO] **MotivoVisita**, por grupo:
  - TIPO_VISITA: 1 Cadastramento/Atualização, 29 Visita periódica.
  - BUSCA_ATIVA: 2 Consulta, 3 Exame, 4 Vacina, 30 Condicionalidades do Bolsa Família.
  - ACOMPANHAMENTO: 5 Gestante, 6 Puérpera, 7 Recém-nascido, 8 Criança, 9 Desnutrição, 10 Reabilitação/deficiência, 11 Hipertensão, 12 Diabetes, 13 Asma, 14 DPOC/enfisema, 15 Câncer, 16 Outras doenças crônicas, 17 Hanseníase, 18 Tuberculose, 32 Sintomáticos respiratórios, 33 Tabagista, 19 Domiciliados/acamados, 20 Vulnerabilidade social, 21 Condicionalidades do Bolsa Família, 22 Saúde mental, 23 Álcool, 24 Outras drogas, 38 Pessoa idosa.
  - CONTROLE_AMBIENTAL_E_VETORIAL: 34 Ação educativa, 35 Imóvel com foco, 36 Ação mecânica, 37 Tratamento focal.
  - OUTROS: 25 Egresso de internação, 27 Convite para atividades coletivas/campanha, 31 Orientação/prevenção, 28 Outros.
- [FATO] "Condicionalidades do Bolsa Família" aparece duas vezes (30 e 21) no próprio dicionário.
- [FATO] **tipoDeImovel:** 1 Domicílio, 2 Comércio, 3 Terreno baldio, 4 Ponto estratégico, 5 Escola, 6 Creche, 7 Abrigo, 8 ILPI, 9 Unidade prisional, 10 Unidade de medida socioeducativa, 11 Delegacia, 12 Estabelecimento religioso, 13 CASAI, 99 Outros.
- [FATO] **Tabela 7 (quem registra):** 515105 ACS, 515310 Agente de ação social, 515140 ACE, 515130 Agente indígena de saneamento, 515125 Agente indígena de saúde, 322255 Técnico em ACS, 515120 Visitador sanitário. **Enfermeiro e médico não registram MIVDT**, diferente do cadastro do 24a. [F5]

**Vínculo com o cadastro do 24a** [F3][F8]
- [FATO] Cidadão por CPF **ou** CNS, nunca os dois; sem nenhum, o marcador "não possui CPF" com justificativa. Na visita periódica e no controle vetorial, o documento é o do responsável familiar.
- [FATO] `uuidOrigemCadastroDomiciliar` é o uuid de origem da ficha de cadastro domiciliar. No 24a, é o que `Ledi::RegistrationFicha#originating_uuid` devolve para o domicílio: a ficha mais antiga dele que não está `rejected`, `failed` nem `correction_pending`. O campo é opcional.
- [FATO] Não há referência ao uuid do cadastro individual; a pessoa se liga só pelo documento.

## 3. Ficha de atividade coletiva (MIAC, tipo 6)

**Estrutura** [F2][F4]
- [FATO] Não há "Master" com lista: uma ficha = uma atividade, com o cabeçalho **dentro** da ficha (campo 24, CBO da Tabela 5).

| # IDL | campo | obrig. | regras |
|---|---|---|---|
| 1 | uuidFicha | Sim | idem MIVDT |
| 4 | outraLocalidade | Não (≤250) | só um entre outraLocalidade, inep e cnesLocalAtividade; proibido com PSE |
| 7 | inep | Condicional (≤8) | obrigatório com PSE (educação ou saúde) |
| 11 | numParticipantes | Sim | 1–999, ≥ tamanho de `participantes` |
| 12 | numAvaliacoesAlteradas | Não | = participantes com avaliação alterada |
| 13 | profissionais | Não | CNS (algoritmo) e CBO de cada um; proibido se só PSE Educação |
| 14 | atividadeTipo | Sim | TipoAtividadeColetiva |
| 15 | temasParaReuniao | Condicional | obrigatório nos tipos 1–3; proibido nos tipos 4–7 |
| 16 | publicoAlvo | Condicional | obrigatório nos tipos 4–7; proibido nos tipos 1–3 |
| 18 | participantes | Condicional | **obrigatório nos tipos 5 e 6** |
| 19 | tbCdsOrigem | Sim | 3 (o nome é `tb`, não `tp`) |
| 21 | cnesLocalAtividade | Não | só um dos 3 locais; proibido com PSE |
| 22 | procedimento | Não | SIGTAP, só com a prática 30 |
| 23 | turno | Sim | Turno |
| 25 | temasParaSaude | Condicional | obrigatório nos tipos 4, 5 e 7; proibido nos tipos 1–3, salvo reunião de Educação permanente (6); tema 18 exige PSE |
| 26 | praticasEmSaude | Condicional | obrigatório no tipo 6; proibido nos tipos 1, 2, 3, 4 e 7; 2, 9 e 33 só no tipo 6 |
| 27/28 | pseEducacao / pseSaude | Sim | `optional` na IDL, mas o dicionário diz "Sim": enviar sempre os dois |

- [FATO] **Participante:** CPF obrigatório nos tipos 5 e 6 sem o marcador "não possui CPF"; o CNS **só** com o marcador. Nascimento e sexo obrigatórios. Avaliação alterada, peso e altura opcionais. "Cessou hábito de fumar" e "abandonou o grupo" só com as práticas 25–28 (PNCT).
- [FATO] Na MIAC o CPF é a identificação primária; na MIVDT, CPF e CNS são só mutuamente exclusivos.

**Domínios** [F1]
- [FATO] **TipoAtividadeColetiva:** 1 Reunião de equipe, 2 Reunião com outras equipes de saúde, 3 Reunião intersetorial/conselho local/controle social, 4 Educação em saúde, 5 Atendimento em grupo, 6 Avaliação/procedimento coletivo, 7 Mobilização social.
- [FATO] **TemasParaReuniao:** 1 Questões administrativas, 2 Processo de trabalho, 3 Diagnóstico do território, 4 Planejamento das ações da equipe, 5 Discussão de caso/PTS, 6 Educação permanente, 7 Outros, 8 Vigilância em saúde.
- [FATO] **TemasParaSaude:** 29 Combate ao Aedes, 19 Agravos negligenciados, 1 Alimentação saudável, 4 Autocuidado em doenças crônicas, 5 Cidadania e direitos humanos, 7 Prevenção de álcool, tabaco e drogas, 8 Envelhecimento/climatério/andropausa, 10 Plantas medicinais, 13 Prevenção da violência, 14 Saúde ambiental, 15 Saúde bucal, 6 Saúde do trabalhador, 16 Saúde mental, 17 Saúde sexual e reprodutiva, 18 Semana saúde na escola, 31 Amamentação, 32 Alimentação complementar, 21 Outros.
- [FATO] **PraticasEmSaude:** 20 Antropometria, 2 Flúor tópico, 23 Desenvolvimento da linguagem, 9 Escovação supervisionada, 11 Práticas corporais, 25–28 PNCT sessões 1–4, 34 PNCT manutenção, 22 Saúde auditiva, 3 Saúde ocular, 24 Situação vacinal, 12 Outras, 30 Outro procedimento coletivo (exige SIGTAP), 33 Kit de saúde bucal.
- [FATO] **PublicoAlvo:** 1 Comunidade, 2 Criança 0–3, 3 Criança 4–5, 4 Criança 6–11, 5 Adolescente, 6 Mulher, 7 Gestante, 8 Homem, 9 Familiares, 10 Pessoa idosa, 12 Doenças crônicas, 13 Tabaco, 14 Álcool, 15 Outras drogas, 16 Sofrimento ou transtorno mental, 17 Profissional da educação, 18 Outros. Na extração do HTML, o 18 saiu colado a um "74 Outro"; o código que vale não foi confirmado.
- [FATO] **Tabela 5 (quem registra):** lista ampla (ACS, ACE, agentes indígenas, agente de saúde pública 352210, assistente social, auxiliares de enfermagem e de saúde bucal, cirurgiões-dentistas, enfermeiros, farmacêuticos e outros). A lista completa está em `docs/cbo.html`, seção "Modelo de Informação de Atividade Coletiva". [F5]

## 4. Cabeçalho

- [FATO] `UnicaLotacaoHeader`: `profissionalCNS` (algoritmo, vínculo no município), `cboCodigo_2002` (na tabela da ficha), `cnes` (7), `ine` (10, opcional), `dataAtendimento` (epoch ms, não futura), `codigoIbgeMunicipio` (7). [F1]
- [INFERÊNCIA] O `Ledi::Fichas::RegistrationHeader` do 24a já monta isso e pode ser reaproveitado. Na visita, o instante é o da visita, não o da gravação.

## 5. Cofinanciamento federal

**Componente de Vínculo e Acompanhamento Territorial** (Portaria GM/MS 3.493/2024; Portaria SAPS/MS 161/2024; NT 30/2025, assinada em 23/09/2025) [F6]
- [FATO] **Cadastro** (30%, até 3 pontos): só o cadastro individual incluído ou atualizado em 24 meses, sem "fora de área" e "mudança de território"; fator 0,75 (só individual) ou 1,5 (individual + domiciliar). É do 24a; o 24b não muda.
- [FATO] **Acompanhamento** (70%, até 7 pontos): pessoa com mais de um contato assistencial em 12 meses, com pelo menos uma **prática de cuidado** (prática + procedimento, ou duas práticas), e com cadastro individual. Práticas de cuidado incluem MIAC e MIVDT, ao lado de MIAI, MIAOI e MIMCA; procedimentos são MIV e MIP. Peso por vulnerabilidade: 1,0, 1,2 (idoso ou criança <5), 1,3 (PBF ou BPC), 2,5 (ambos).
- [FATO] **Identificação** (rodapé 1 da NT): só contam pessoas com CNS ou CPF e data de nascimento idênticos ao CADSUS, com CNS e CBO do profissional válidos e CNES e INE ativos.
- [INFERÊNCIA] Visita com cidadão identificado conta como prática de cuidado; sem identificação ou em imóvel não residencial, não alimenta o acompanhamento individual.
- [INFERÊNCIA] Na atividade coletiva, só os participantes identificados contam por pessoa; `numParticipantes` sozinho não identifica ninguém. Vale registrar a lista sempre que possível, não só nos tipos 5 e 6.

**Componente de qualidade** [F7]
- [FATO] **C5 (hipertensão)**, boa prática D: pelo menos 2 visitas domiciliares de ACS/TACS, com intervalo mínimo de 30 dias, nos últimos 12 meses (25 pontos). Conta a MIVDT com o motivo preenchido, por CBO 515105 ou 322255. PA e peso/altura informados nos campos da MIVDT também contam.
- [INFERÊNCIA] Como o motivo é proibido com desfecho 2 ou 3, só a visita realizada com motivo conta para a C5. Não está escrito na nota.
- [FATO] Segundo a busca, C2 (desenvolvimento infantil), C3 (gestação e puerpério) e R3 também pedem visitas de ACS/TACS. As notas não foram abertas. C6 e C7 provavelmente seguem o padrão da C5, sem verificação.
- [FATO] O IVCF-20 na visita (60 anos ou mais) aparece na Nota Informativa 1/2026 (idosos), não lida. [F9]

## 6. Implicações para o desenho [INFERÊNCIA]

- Registry: `visita_domiciliar` (8, `FichaVisitaDomiciliarMasterThrift`, `uuidFicha`) e `atividade_coletiva` (6, `FichaAtividadeColetivaThrift`, `uuidFicha`) no `Ledi::FichaTypes`.
- Tabelas 7 e 5 próprias no `config/ledi/registration_cbos.yml`; a do cadastro não serve.
- A folha do dia é a forma nativa da MIVDT; uma ficha por visita é a alternativa se o PEC exigir.
- Vendorizar `dicionario-fvd.html` e `dicionario-fac.html` com SHA-256 e data no `SOURCE`, como foi feito com fai e fao, antes de basear specs neles.
- A boa prática de visita das linhas de cuidado precisa de motivo preenchido, CBO de ACS/TACS e intervalo mínimo.

## 7. Não confirmado

1. Se o PEC aceita várias fichas MIVDT do mesmo profissional no mesmo dia, com uma visita cada (a IDL permite; o PEC não foi testado).
2. ~~Nome exato dos módulos Ruby no `gen-rb`~~ — conferido em 2026-11-12: `Visitadomiciliar` e `Atividadecoletiva`.
3. Código do público-alvo "Outros" (18 ou 74).
4. Se o SIAPS conta, no Acompanhamento, visita recusada ou de ausente (a NT 30 não distingue; pela C5, só com motivo) e participantes de atividade coletiva dos tipos 4 e 7.
5. Se a visita periódica com o documento do responsável familiar conta só para o responsável.
6. Notas C2, C3, C6 e C7 não lidas; os requisitos de validação do SIAPS citados na NT 30 também não foram consultados. [F10]
7. Os dicionários fvd e fac baixados não estão vendorizados nem com hash.

## 8. Fontes (acesso em 2026-11-12)

- F1 Dicionário LEDI 8.7 (tabelas de domínio, TipoDadoSerializado, UnicaLotacaoHeader) — `vendor/ledi/8.7.0/docs/dicionario.html` e `header-transport.html`
- F2 IDL e Ruby gerado — `vendor/ledi/8.7.0/idl/ras/ficha_visita_domiciliar.thrift`, `ficha_atividade_coletiva.thrift`, `gen-rb/ficha_visita_domiciliar_types.rb`, `gen-rb/ficha_atividade_coletiva_types.rb` (origem `laboratoriobridge/esusaps-integracao@9316f280f5`)
- F3 Dicionário da ficha de visita domiciliar e territorial — https://integracao.esusaps.bridge.ufsc.tech/ledi/documentacao/estrutura_arquivos/dicionario-fvd.html
- F4 Dicionário da ficha de atividade coletiva — https://integracao.esusaps.bridge.ufsc.tech/ledi/documentacao/estrutura_arquivos/dicionario-fac.html
- F5 Tabelas de CBO por ficha — `vendor/ledi/8.7.0/docs/cbo.html`
- F6 Nota Técnica nº 30/2025-CGESCO/DESCO/SAPS/MS — https://www.gov.br/saude/pt-br/centrais-de-conteudo/publicacoes/notas-tecnicas/2025/nota-tecnica-no-30-2025-cgesco-desco-saps-ms.pdf
- F7 Nota Metodológica C5 (pessoa com hipertensão) — https://www.gov.br/saude/pt-br/composicao/saps/publicacoes/fichas-tecnicas/equipe-de-atencao-primaria-e-saude-da-familia/nota-metodologica-c5-cuidado-da-pessoa-com-hipertensao
- F8 Código do 24a — `app/services/ledi/registration_ficha.rb`, `app/services/ledi/fichas/household_registration.rb`
- F9 Nota Informativa nº 1/2026-COPID/DGCI/SAPS/MS (idosos) — https://www.gov.br/saude/pt-br/centrais-de-conteudo/publicacoes/estudos-e-notas-informativas/2026/nota-informativa-no-1-2026-copid-dgci-saps-ms.pdf (não lida)
- F10 Requisitos do SIAPS — https://sisaps.saude.gov.br/sistemas/siaps/ (não consultado)
