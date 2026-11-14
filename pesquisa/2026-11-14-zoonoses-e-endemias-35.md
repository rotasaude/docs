# Pesquisa — Módulo 35 (Zoonoses e endemias)

Data: 2026-11-14. Convenção: **[FATO]** vem com a URL da fonte lida; **[INFERÊNCIA]** é dedução minha ou algo que não consegui confirmar em fonte oficial. Vários PDFs oficiais foram lidos por extração de texto. Quando a afirmação vem só de um trecho de busca, e não do documento lido, isso está dito.

Fontes-base mais citadas:
- **DN-2025**: *Diretrizes Nacionais para Prevenção e Controle das Arboviroses Urbanas: Vigilância Entomológica e Controle Vetorial*, MS/SVSA, 2025: https://www.gov.br/saude/pt-br/centrais-de-conteudo/publicacoes/svsa/dengue/diretrizes-nacionais-para-prevencao-e-controle-das-arboviroses-urbanas-vigilancia-entomologica-e-controle-vetorial.pdf
- **NT3-2025**: Nota Técnica nº 3/2025-CGARB/DEDT/SVSA/MS (ovitrampas): https://www.gov.br/saude/pt-br/centrais-de-conteudo/publicacoes/notas-tecnicas/2025/nota-tecnica-no-3-2025-cgarb-dedt-svsa-ms.pdf
- **NT41-2026**: Nota Técnica nº 41/2026-CGARB/DEDT/SVSA/MS (integração das tecnologias): https://www.gov.br/saude/pt-br/centrais-de-conteudo/publicacoes/notas-tecnicas/2026/nota-tecnica-no-41-2026-cgarb-dedt-svsa-ms.pdf
- **LIRAa-2013**: Manual LIRAa, MS/SVS, 2013: https://bvsms.saude.gov.br/bvs/publicacoes/manual_liraa_2013.pdf
- **MAN-2001**: *Dengue: instruções para pessoal de combate ao vetor*, FUNASA, 2001: https://bvsms.saude.gov.br/bvs/publicacoes/funasa/man_dengue.pdf
- **DN-2009**: Diretrizes Nacionais para Prevenção e Controle de Epidemias de Dengue, MS, 2009 (cópia NESCON/UFMG): https://www.nescon.medicina.ufmg.br/biblioteca/imagem/diretrizes-nacio-prevencao-controle.pdf
- **ZOO-2016**: *Manual de Vigilância, Prevenção e Controle de Zoonoses: Normas Técnicas e Operacionais*, MS, 2016: https://www.gov.br/saude/pt-br/centrais-de-conteudo/publicacoes/svsa/zoonose/manual-zoonoses-normas-2v-7julho16-site.pdf
- **LV-2026**: *Manual de Vigilância e Controle da Leishmaniose Visceral*, MS, 2026: https://www.gov.br/saude/pt-br/centrais-de-conteudo/publicacoes/guias-e-manuais/2026/manual-de-vigilancia-e-controle-da-leishmaniose-visceral.pdf

---

## 1. Sistemas nacionais de informação para o trabalho do ACE (2025–2026)

### 1.1 Quadro geral

- **[INFERÊNCIA]** Hoje não existe um sistema nacional único e vigente para o trabalho de campo do ACE com dado por imóvel. O registro está espalhado entre sistemas estaduais, programas desktop do MS, planilhas e, para ovitrampas, o app Conta Ovos. Os itens abaixo sustentam essa leitura.
- **[FATO]** A DN-2025 não indica um sistema nacional de vetores. Sobre o boletim de contagem de ovos, ela diz que os dados "podem ser incluídas no sistema de informações local ou no sistema nacional (o que estiver disponível)" (Apêndice F, item 4.5). Uma busca no texto extraído não encontrou nenhuma menção a SISPNCD, SisLoc ou e-SUS Vetores. — DN-2025
- **[FATO]** Nas DN-2009, cabia ao estado "gerenciar o sistema de informação no âmbito estadual, consolidar e enviar os dados regularmente à esfera federal". O ACE devia registrar as ações "nos formulários apropriados [...] com o objetivo de alimentar o sistema de informações vetoriais". — DN-2009

### 1.2 Sistema por sistema

| Sistema | Situação | O que recebe | Forma de entrada | Quem digita |
|---|---|---|---|---|
| **SISPNCD** (Sistema do Programa Nacional de Controle da Dengue) | **[INFERÊNCIA]** legado. Não aparece na DN-2025 nem nas notas de 2025–2026. Material de apoio de Goiás o citava como fonte do indicador de visitas do PQA-VS ("% de imóveis visitados", visto em trecho de busca: https://goias.gov.br/saude/wp-content/uploads/sites/34/2017/10/vigilancia-em-saude-dengue-vd-05.2015-e91.pdf) | Visitas e tratamento por imóvel/quarteirão, consolidados por ciclo **[INFERÊNCIA]** | Não confirmei se é desktop ou web | Município/regional **[INFERÊNCIA]** |
| Indicador de ciclos do PQA-VS | **[FATO]** As metas do PQA-VS a partir de 2025 (14 indicadores) não incluem mais "ciclos com ≥80% de imóveis visitados". O único indicador de arbovirose é o de óbitos suspeitos de dengue/chikungunya encerrados em até 60 dias. — Portaria GM/MS 6.878/2025: https://www.saude.mg.gov.br/wp-content/uploads/2025/07/12-PORTARIA-GMMS-No-6.878-DE-17-DE-ABRIL-DE-2025-Estabelece-as-metas-e-os-indicadores-do-Programa-de-Qualificacao-das-Acoes-de-Vigilancia-em-Saude-PQA-VS.pdf | — | — | — |
| **SisLoc** (cadastro de localidades) | **[FATO]** O DATASUS tem um "Sistema de Cadastro de Localidade" para localidades municipais, com base em DBF, citado num manual de operação do Sinan hospedado pela SES-GO (visto em trecho de busca: https://goias.gov.br/saude/wp-content/uploads/sites/34/2013/08/manual_de_operacao-d7c.pdf). **[INFERÊNCIA]** Era a tabela de localidades usada pelo Sinan/SISPNCD; não achei versão atual. | Localidades/bairros | Desktop/DBF | Município |
| **LIRAa/LIA** (programa do MS) | **[FATO]** Vigente. Em Goiás, o município instala o "LIRAa-LIA município" (desktop), digita o plano amostral (estratos, imóveis, quarteirões) e o consolidado de campo, gera o arquivo "dados para SVS" e o **envia por e-mail à Regional**. O estado importa no "LIRAa-LIA estado", consolida e envia ao MS. — POP 09 SES-GO: https://goias.gov.br/saude/wp-content/uploads/sites/34/2025/12/POP-Sistemas-de-Informacao-de-Vigilancia-de-Vetores.pdf | Plano amostral e consolidado por estrato (não o dado por imóvel) | Desktop + **arquivo exportado** | Município digita; estado consolida |
| **Conta Ovos** (ovitrampas) | **[FATO]** É o canal indicado pelo MS: "Os estados e municípios que implementarem o monitoramento por ovitrampas terão acesso ao aplicativo 'Conta-ovos'", que permite inserir dados, calcular índices e gerar mapas. — NT3-2025. **[FATO]** Foi criado por Fiocruz, CEFET-RJ e Mosqlimate (FGV); o MS supervisiona a implantação municipal "with the intention of absorbing its structure in the future". — https://contaovos.com/ | Ovitrampas, contagens de ovos, visitas em quarteirões, quarteirões, EDL, manutenção de EDL, pontos estratégicos, imóveis | Web/app + **API REST** (ver 5.4) | Município (gestor gera a chave de API) |
| **SIES** (Sistema de Informação de Insumos Estratégicos) | **[FATO]** Vigente: http://sies.saude.gov.br. Pedido, autorização, entrada por nota fiscal de movimento e estoque de **praguicidas**. Também cobre **vacina antirrábica animal e seringas** (MT 2025) e soro antirrábico (NT Conjunta 35/2026). — POP 09 SES-GO; NT MT 02/2025: https://www.saude.mt.gov.br/storage/files/60kiWpkl4f1XLQ6eHCkVrD7p8yvQWNd5rd9d6iV8.pdf | Movimentação de insumos | Web (digitação) | Município, regional, estado |
| **Sistemas estaduais** | **[FATO]** SP: **Sisaweb**, "Sistema de Vigilância e Controle do Aedes" (https://vigent.saude.sp.gov.br/sisaweb/), integrado ao app **SisaMob**, segundo o BEPA (visto em trecho de busca: https://periodicos.saude.sp.gov.br/BEPA182/article/download/37802/35773/39977). GO: **SIMAZ** (lançamento de visitas a PE, bloqueio de casos notificados e "quadras visitadas" com boletim de visita) e **GISSAUDE** (bairros e quadras), ambos na intranet da SES-GO. — POP 09 SES-GO | Visita, PE, bloqueio | Web (digitação) | Município |
| **e-SUS APS** | **[FATO]** O ACE é cadastrado no SCNES com CBO 5151-40 (Portaria GM/MS 535/2016). — Diretriz Nacional ACE/ACS 2025: https://www.gov.br/saude/pt-br/centrais-de-conteudo/publicacoes/svsa/vigilancia-ambiental/diretriz-nacional-para-atuacao-dos-ace-e-acs-no-territorio.pdf. **[FATO]** O app e-SUS Território é destinado a "agentes comunitários, de endemias e de ação social". — Manual e-SUS AB Território v2.2 (cópia municipal): https://www.salgueiro.pe.gov.br/saude/doc/esus/Manual_ABTerritorio.pdf | Ficha de Visita Domiciliar e Territorial (motivos 34–37) | CDS/app/**LEDI** | Município |
| **"e-SUS Vetores" / "Sistema de Informação de Vetores"** | **[INFERÊNCIA]** Não achei nenhum sistema oficial com esse nome em 2025–2026. **[FATO]** O "SisVetor" é um software registrado pela UnB (vitrine tecnológica), não um sistema do MS. — https://cdt.unb.br/images/CITT/VITRINE_TECNOLOGICA/11_Saude/PDFs/BR_51_2022_003020_9_-_Demanda_20-0622_-_SOFTWARE_-_SisVetor.pdf | — | — | — |

### 1.3 Boletins diários de campo

- **[FATO]** O boletim diário de campo existe em papel ou no app e é digitado depois. SP: "Boletim diário utilizado para registro das atividades de vigilância e controle [...] O objetivo desse preenchimento é a coleta de dados no campo e digitação das informações em sistema próprio". — Instrução SES-SP (atualizada em 14/04/2025): https://portal.saude.sp.gov.br/resources/cve-centro-de-vigilancia-epidemiologica/areas-de-vigilancia/doencas-de-transmissao-por-vetores-e-zoonoses/vetores/outros/instrucaaopreenchimento-boletimvigilaenciaecontrole-aedes.pdf
- **[FATO]** No LIRAa, o fluxo é: o agente entrega o **BCL** (Boletim de Campo e Laboratório) ao supervisor no fim do dia; o supervisor faz o Resumo Parcial; o supervisor-geral envia ao laboratório; depois o BCL e o RP vão para o "setor de digitação". — LIRAa-2013, cap. 4 e 6

---

## 2. Fluxo de trabalho do ACE (PNCD → DN-2025)

### 2.1 Organização territorial e ciclos

- **[FATO]** Reconhecimento geográfico: quarteirões numerados em sequência (por bairro ou setor), com sinais nas esquinas, croquis por zona de trabalho e a relação de imóveis por quarteirão. A zona de trabalho fixa tem **800 a 1.000 imóveis**, visitados em **ciclos bimensais** nos municípios infestados. O rendimento de referência é de 20 a 25 imóveis por agente por dia em LI e tratamento focal, e 15 por dia em PE. — MAN-2001
- **[FATO]** DN-2009: "Visita domiciliar bimestral em 100% dos imóveis" e pesquisa larvária amostral bimestral ou **quatro LIRAa** por ano. — DN-2009
- **[FATO]** A DN-2025 rompe esse modelo. A estratificação de risco orienta o planejamento "para além da tradicional visita casa a casa a cada 60 dias". Nas áreas prioritárias (*hotspots*), mantém-se a visita em 100% dos domicílios, "a intervalos mais curtos do que o preconizado atualmente (60 dias)". Nas não prioritárias (*coldspots*), a visita casa a casa passa a partir "do raio de positividade das ovitrampas". — DN-2025, cap. 2. **[FATO]** A NT41-2026 reafirma o mesmo critério.
- **[FATO]** A DN-2025 classifica o imóvel domiciliar das áreas prioritárias por risco: **verde** (baixo), **amarelo** (médio), **vermelho** (alto), **cinza** (difícil acesso, com agendamento) e **roxo** (difícil acesso, exige intervenção legal ou VISA). A frequência de visita muda conforme essa classe, que deve ser reavaliada periodicamente. — DN-2025, 5.3

### 2.2 Tipos de imóvel

- **[FATO]** O e-SUS APS (`tipoDeImovel`) usa: 1 Domicílio, 2 Comércio, 3 Terreno baldio, 4 **Ponto Estratégico**, 5 Escola, 6 Creche, 7 Abrigo, 8 ILPI, 9 Unidade prisional, 10 Unidade socioeducativa, 11 Delegacia, 12 Estabelecimento religioso, 13 CASAI, 99 Outros. — Dicionário LEDI: https://integracao.esusaps.bridge.ufsc.tech/ledi/documentacao/referencias/dicionario.html
- **[FATO]** Os editais e boletins municipais usam o agrupamento clássico **residência, comércio, terreno baldio (TB), outros**, mais o PE. — Contagem RFZ16–19 e BH RF30 (seção 4). O BCL do LIRAa tem uma coluna específica "TB". — LIRAa-2013, 6.1.1. **[INFERÊNCIA]** As siglas R/C/TB/PE/O do boletim PNCD são prática consolidada, mas não achei a tabela oficial de códigos.
- **[FATO]** **Ponto estratégico (PE)** é local com concentração de depósitos preferenciais ou vulnerável à introdução do vetor: cemitérios, borracharias, ferros-velhos, sucata, materiais de construção, garagens de veículos grandes, residências de acumuladores. A vigilância é **quinzenal** e a aplicação residual/focal é **bimestral** ou quando houver foco. — DN-2025, 7.1.3 e glossário
- **[FATO]** **Imóvel especial (IE)** é local de grande fluxo ou permanência (UBS, ILPI, escolas, rodoviárias, universidades) e é alvo da BRI-Aedes. — DN-2025; NT41-2026

### 2.3 Situação da visita

- **[FATO]** O BCL do LIRAa registra **Fechado (F), Recusado (R), Trabalhado (T)**, com código do motivo do fechado no rodapé. No LIRAa, um imóvel sorteado que está fechado ou recusado é **substituído** pelo imediatamente posterior (ou anterior). — LIRAa-2013
- **[FATO]** O boletim de SP é mais fino: Trabalhado, Fechado, Desocupado, Temporada, Parcial, Recusa. O tipo de trabalho é **Rotina** ou **Pendência**, esta última para o retorno a imóvel fechado, que é a "recuperação". Há também a casela "Programar retorno". — Instrução SES-SP 2025
- **[INFERÊNCIA]** "Recuperado" não é uma situação no boletim. É o resultado de uma visita de *pendência* que conseguiu trabalhar um imóvel antes fechado. O indicador típico é a % de pendência por ciclo.

### 2.4 Depósitos

- **[FATO]** São cinco grupos: **A1** (depósito de água elevado ligado à rede, poço ou cisterna: caixa d'água, tambores, alvenaria), **A2** (armazenamento ao nível do solo: tonel, barril, tina, potes, cisternas), **B** (móveis: vasos, pratos, garrafas, pingadeiras, bebedouros), **C** (fixos: calhas, lajes, ralos, piscinas não tratadas, tanques de obra), **D1** (pneus), **D2** (lixo e recipientes descartáveis), **E** (naturais: bromélias, ocos de árvore). — LIRAa-2013, cap. 5; DN-2025, 5.2.2
- **[FATO]** O boletim de SP registra, por tipo de recipiente: **existentes**, **com água**, **com larva**. — Instrução SES-SP 2025

### 2.5 Amostras e laboratório

- **[FATO]** Coleta-se **um tubito por depósito positivo**, com no máximo 10 larvas. Não se faz *pool* entre criadouros, e a numeração das amostras é sequencial por imóvel (ex.: 1 a 10, 11 a 18). Quando o IIP sai igual ao IB, isso indica coleta só do primeiro depósito. — LIRAa-2013
- **[FATO]** O laboratório preenche no próprio boletim: tubitos examinados, tubitos com *Ae. aegypti*, tubitos com *Ae. albopictus*, recipientes positivos para cada espécie. — LIRAa-2013, 6.1.2. No boletim de SP: larvas examinadas, *Ae. aegypti* e *Ae. albopictus* por amostra. Em PE, IE e ADL, cada recipiente com larva ocupa uma linha. — Instrução SES-SP 2025
- **[INFERÊNCIA]** A identificação larvária costuma ficar no laboratório municipal ou regional de entomologia, fora do GAL. Só a entomologia molecular ou especial passa pelo GAL (ver 5.3).

### 2.6 Tratamento focal, perifocal e espacial

- **[FATO]** O controle mecânico é a primeira opção. O larvicida só entra em depósito que não pode ser eliminado, nunca em descartáveis, utensílios, aquários, pratos de vaso, bebedouros de animais ou tipo E. É obrigatória a **cubagem** do depósito para dosar o larvicida. — DN-2025, 7.1.1–7.1.2
- **[FATO]** No boletim de SP, as medidas de controle são: Mecânico, Produto alternativo, **Focal** (nº de cargas abertas), **Perifocal** (cargas), **Nebulização** (cargas), **BR-Aedes** (cargas). O rodapé traz o código do inseticida, o agente, o supervisor e o laboratório. Produtos e cargas atuais em SP: Sumilarv (código 1A, carga 20 g), BTI Vectobac WG (1B, 500 g), Bendiocarb (2A, 80 ml), BTI perifocal (2B, 125 g), Fludora Fusion (2C, 100 g), Komvektor (3A, 1000 ml), Cielo (3B, 2000 ml). — Instrução SES-SP 2025
- **[FATO]** Histórico: temephós G 1% a 1 ppm, com consumo por imóvel por ano e por região. — MAN-2001, anexo. **[INFERÊNCIA]** Os produtos mudam por nota técnica, então o catálogo de produtos e cargas precisa ser parametrizável.

### 2.7 LIRAa / LIA

- **[FATO]** Os estratos vão de **8.100 a 12.000 imóveis**. Em caráter excepcional, estratos de 2.000 a 8.100 imóveis inspecionam 50%. A amostra parte de 450 imóveis corrigidos pela fórmula n = 450 / (1 + 450/N). Os quarteirões são sorteados por intervalo, e em cada quarteirão sorteado inspecionam-se **20% dos imóveis**. — LIRAa-2013
- **[FATO]** Índices: **IIP** = imóveis positivos/pesquisados × 100; **IB** = recipientes positivos/imóveis pesquisados × 100; **ITR** = recipientes positivos do tipo X/total de positivos × 100. — LIRAa-2013; DN-2025
- **[INFERÊNCIA]** "IBP" não aparece nos documentos nacionais lidos. Talvez seja um índice de Breteau para pupas, que a DN-2025 só cita como método complementar ("contagem de pupas").
- **[INFERÊNCIA]** Classificação usual do IIP: <1% satisfatório, 1–3,9% alerta, ≥4% risco. Não confirmei no texto extraído; deve estar em figura do manual.
- **[FATO]** Periodicidade nova: "deve ser realizado em todo o município, com periodicidade **mínima de uma vez ao ano**, no período pré-epidêmico". O período nacional sai por nota técnica. — DN-2025, 5.2.2; NT3-2025, 4.26

### 2.8 Ovitrampas

- **[FATO]** São preconizadas para **todos** os municípios, infestados ou não. São pré-requisito, com no mínimo 3 meses de uso contínuo, para EDL, Wolbachia e TIE. O espaçamento é de **300 ou 400 m**. A leitura é semanal ou quinzenal nas áreas prioritárias e mensal nas demais. A palheta é recolhida **5 dias** após a instalação, sem coincidir com fim de semana ou feriado. Os índices são **IPO** (% positivas), **IDO** (ovos por armadilha positiva) e **IDV** (ovos por armadilha examinada). — DN-2025, 5.2.1 e Apêndice F
- **[FATO]** Quando os indicadores sobem, intensifica-se o controle num raio de **200 m** ao redor da ovitrampa. — NT3-2025, 4.29
- **[FATO]** O **boletim de campo da ovitrampa** registra endereço, nº do quarteirão, nº da armadilha, nome do responsável pelo imóvel e data de instalação, com **TCLE assinado pelo morador**. Avarias, perdas e armadilhas sem água também vão para o boletim. A **etiqueta da palheta** leva o nº da ovitrampa, o **código IBGE** do município e a data de instalação. Há um "Boletim de Contagem de Ovos" por lote, inclusive para palhetas negativas. — NT3-2025, 4.9, 4.16, 4.21; DN-2025, Apêndice F

### 2.9 Bloqueio de transmissão

- **[FATO]** É a ação imediata diante de notificação de caso suspeito. Faz-se vistoria e controle larvário num **raio mínimo de 150 m** da residência do caso (cerca de 9 quarteirões), junto com UBV costal intra e peridomiciliar. Agrupam-se os casos do mesmo período (**até 7 dias**) e da mesma área. São **até 3 ciclos** de nebulização, cada um com 3 a 5 dias consecutivos e intervalo de 5 dias. A efetividade "depende da oportunidade", o que exige um "fluxo oportuno de informações das notificações". — DN-2025, 7.1.4
- **[FATO]** As liberações de Wolbachia e de TIE são suspensas durante o bloqueio com UBV e retomadas depois dos ciclos. As EDL não ficam no mesmo imóvel que uma ovitrampa. — NT41-2026
- **[FATO]** Em Goiás, o bloqueio é lançado no SIMAZ **a partir da consulta de casos notificados** (município, agravo, data de notificação), com boletim de campo por ponto no mapa. — POP 09 SES-GO

### 2.10 Campos do boletim diário (síntese)

Composição a partir do boletim de SP 2025 e do BCL do LIRAa-2013 (**[FATO]** para cada campo, com as fontes citadas):

- **Cabeçalho:** atividade (LI, LI+T, PE, IE, DF, bloqueio, nebulização, ADL…), folha n/N, data, município, área ou estrato, setor censitário, bairro, quarteirão, órgão executor, tipo de trabalho (rotina/pendência), nº de quarteirões e de imóveis trabalhados no dia.
- **Linha por imóvel:** nº de cadastro ou ordem do imóvel, endereço (logradouro, nº, complemento), tipo de imóvel (ou só a marca TB no LIRAa), situação (T/F/R ou a escala de SP), medidas de controle (mecânico, alternativo, focal com cargas, perifocal, nebulização, BR-Aedes), recipientes por tipo (existentes, com água, com larva), nº das amostras e de tubitos, retorno programado.
- **Campos do laboratório:** tubitos/larvas examinados, *Ae. aegypti*, *Ae. albopictus*, recipientes positivos por espécie.
- **Rodapé:** código e consumo do inseticida (cargas), agente, supervisor, laboratorista.
- **[INFERÊNCIA]** No app, os campos "folha n/N" e "Total" do boletim de papel deixam de existir. O GPS e o horário de entrada e saída da visita, exigidos em editais (BH RF31), entram no lugar.

---

## 3. Zoonoses e UVZ

### 3.1 Base normativa

- **[FATO]** A Portaria GM/MS 1.138/2014 define as ações e os serviços de vigilância, prevenção e controle de zoonoses e de acidentes com animais peçonhentos e venenosos de relevância para a saúde pública. Os estabelecimentos que executam essas ações devem ser inscritos no SCNES. Ela foi consolidada na **Portaria de Consolidação nº 5/2017**, cujo art. 233 foi alterado pela Portaria GM 2.087/2018, que aprovou o Manual de Normas Técnicas para Estruturas Físicas de UVZ. — https://bvsms.saude.gov.br/bvs/saudelegis/gm/2018/prt2087_18_07_2018.html; https://www.conass.org.br/?p=14276
- **[FATO]** A Portaria SAS 758/2014 incluiu no SCNES o **Tipo 50 (Unidade de Vigilância em Saúde), subtipo 01 (Unidade de Vigilância de Zoonoses)**. — https://www.conass.org.br/?p=3644. **[FATO]** A Portaria SAES 465/2023 (serviço 141, classificação 005, Vigilância de Zoonoses) aparece em trecho de busca; não li o texto integral. — https://ses.sp.bvs.br/wp-content/uploads/2023/05/U_PT-MS-SAES-465_250523.pdf
- **[FATO]** Capítulos do ZOO-2016: vigilância, prevenção e controle (raiva, vacinação animal, LV, LTA); atividades da UVZ (recolhimento, recepção e remoção de animais, transporte, alojamento, necropsia); laboratório (diagnóstico de zoonoses, entomologia); controle de populações (roedores, escorpiões, vetores); inspeção zoossanitária; educação em saúde. — ZOO-2016

### 3.2 Raiva: vacinação animal

- **[FATO]** A vacina antirrábica de cães e gatos é a **única** vacina animal preconizada pelo MS no serviço público. A aplicação é **em massa**, por campanha casa a casa, posto fixo ou as duas formas, ou **por bloqueio de foco**, casa a casa, **em até 72 h** após cão ou gato positivo com AgV1/AgV2. Pode haver campanha de intensificação (duas por ano). Em área de risco AgV1/AgV2, a vacina deve ficar disponível **o ano todo** fora da campanha (rotina). — ZOO-2016, 1.3
- **[FATO]** O evento adverso pós-vacinal (**EATAV**) é notificado "por meio de formulário específico, ao Ministério da Saúde". — ZOO-2016. Em MT 2025, esse formulário é um **REDCap do MS** (`redcap.saude.gov.br`). — NT MT 02/2025
- **[FATO]** O resultado da campanha é reportado como **quantitativo agregado**: em MT 2025, o município atualiza **semanalmente** uma planilha Google estadual ("Resultados da Campanha Estadual de Vacinação Antirrábica Animal - 2025"), acompanhada por município, regional e estado. As doses e seringas usadas são lançadas como saída no **SIES**. Dois municípios "não registraram o número de cães e gatos vacinados" em 2024, e a meta de referência é cobertura canina ≥80%. — NT MT 02/2025
- **[FATO]** Um estudo de MT cita o **SI-PNI** como fonte da cobertura canina entre 2012 e 2017 (visto em trecho de busca). **[INFERÊNCIA]** Hoje o SI-PNI não recebe o dado animal; o canal varia por estado (planilha ou formulário). **Não existe registro nacional individual de animal vacinado.**

### 3.3 Raiva: agressão, observação e atendimento humano

- **[FATO]** Todo cão ou gato agressor (que mordeu, lambeu ou arranhou) ou com sinais neurológicos é **observado por 10 dias** em local adequado. Se morrer ou passar por eutanásia, envia-se o SNC ao laboratório. Animal positivo leva a investigação, bloqueio de foco e avaliação de risco. Morcego que entra em casa é recolhido para exame. Cão ou gato que teve contato com morcego positivo segue um protocolo próprio (isolamento por 180 dias e doses nos dias 0, 7 e 30, ou 0 e 30). — ZOO-2016, 1.2
- **[FATO]** Pela lista nacional vigente, "**Acidente por animal potencialmente transmissor da raiva**" é de notificação **imediata (até 24 h) à SMS**, e "Raiva humana" é imediata ao MS, à SES e à SMS. Li a tabela pela posição das colunas no PDF. — Portaria GM/MS 11.211/2026: https://www.crmpr.org.br/uploadAddress/PORTARIA-GM-MS-NA-11.211--DE-13-DE-MAIO-DE-2026-PORTARIA-GM-MS-NA-11.211--DE-13-DE-MAIO-DE-2026-DOU-Imprensa-Nacional[6907].pdf
- **[FATO]** A ficha Sinan de **Atendimento Antirrábico Humano** tem espécie do agressor (canina, felina, quiróptera, primata, raposa, herbívoro doméstico, outra), condição do animal para conduta (sadio, suspeito, raivoso, morto/desaparecido), "animal passível de observação?" (só cão ou gato), tratamento indicado e **condição final do animal após a observação**. — protocolo USP 2022 e ficha SES-GO, vistos em trecho de busca: https://sites.usp.br/guiacseb/wp-content/uploads/sites/943/2023/05/PROTOCOLO-AARH-2022.pdf; https://goias.gov.br/saude/wp-content/uploads/sites/34/2013/08/atendimento-anti-rabico-00f.pdf
- **[FATO]** As doses de soro e vacina antirrábicos aplicadas em pessoas "deverão ser registradas nos sistemas de informação **SI-PNI e SINAN**". — NT Conjunta 35/2026: https://www.gov.br/saude/pt-br/composicao/svsa/pni/notas-tecnicas/2026/nota-tecnica-conjunta-no-35-2026-cggi-cgici-dpni-cgzha-dedt-svsa-ms-e-cgzha-dedt-svsa-ms
- **[INFERÊNCIA]** O caso nasce na UBS (ficha Sinan) e segue para a UVZ, que observa o animal por 10 dias. O desfecho volta para fechar a ficha e suspender ou completar o esquema. Hoje isso costuma ser telefone ou papel.

### 3.4 Epizootias (notificação de evento animal)

- **[FATO]** São de notificação compulsória **imediata**, com base na vigilância animal: febre amarela, raiva, Nilo Ocidental, outras arboviroses (encefalites equinas, Oropouche, Mayaro), peste, influenza. Os eventos de saúde pública incluem morte de PNH, **morte ou adoecimento de cães e gatos com sintomatologia neurológica**, morte de aves silvestres, equídeos com quadro neurológico, morte de canídeos silvestres, **morte de quirópteros em área urbana**, roedores silvestres em área de peste e animais silvestres sem causa conhecida. Qualquer cidadão pode comunicar. — Portaria GM/MS 782/2017 (DOU 16/03/2017): https://www.biodiversidade.ciss.fiocruz.br/sites/www.biodiversidade.ciss.fiocruz.br/files/DUO%20Portaria%20782.pdf

### 3.5 Leishmaniose visceral canina (manual de 2026)

- **[FATO]** Diagnóstico: **TR-DPP** (Bio-Manguinhos) na triagem e **ELISA** (Bio-Manguinhos) na confirmação. Os dois são fornecidos pelo MS para inquéritos. — LV-2026, cap. 3; também ZOO-2016
- **[FATO]** A **investigação de foco**, feita no 1º caso canino suspeito ou no 1º caso humano, cobre um raio com no mínimo **100 cães** examinados, com inquérito sorológico amostral ou censitário e biópsia, aspirado ou PCR para identificar *L. infantum*. — LV-2026, 4.6.3.1
- **[FATO]** O **inquérito sorológico amostral** é feito por ATL. A amostra sai de uma tabela de população canina × prevalência esperada (2% por padrão; ex.: 600 cães → 334). A amostragem é aleatória simples ou por conglomerado (quarteirão), assumindo cerca de 16 cães por quarteirão. O manual exige que o município tenha **"algum tipo de estratégia de identificação individual de todos os cães da ATL, por exemplo, número do registro geral, número do microchip"**. — LV-2026, 4.6.4.1
- **[FATO]** As **coleiras impregnadas com deltametrina 4%** foram incorporadas pelo MS para cães a partir de 3 meses e são **trocadas a cada 6 meses**. A estimativa de cães pode vir de "censo canino ou campanha antirrábica". — LV-2026, 3.4 e 6.3.1
- **[FATO]** A eutanásia de cão reagente é respaldada pela Lei 14.228/2021 e feita conforme a Resolução CFMV 1.000/2012, sob responsabilidade de médico veterinário. O tutor pode optar por tratamento com droga registrada no MAPA (miltefosina), conforme portaria interministerial. — LV-2026, 3.3 e 6.3.2
- **[FATO]** Casos humanos vão para o Sinan, com notificação semanal na lista de 2026. — Portaria 11.211/2026; LV-2026
- **[FATO]** O **SisLeish** (Sistema Regional de Informação sobre Leishmanioses) é usado na estratificação. — LV-2026, lista de siglas e cap. 4. **[INFERÊNCIA]** É o sistema regional da OPAS, que não recebe dado municipal operacional.

### 3.6 Outros agravos e ações típicas

- **[FATO]** Esporotricose **humana** entrou na lista nacional pela Portaria GM/MS 6.734/2025, com notificação semanal, que a Portaria 11.211/2026 mantém. — https://app.wiki.saude.es.gov.br/vigilancia/sistemas/portaria_gm_ms_n%C2%BA_6.734_de_18_de_mar%C3%A7o_de_2025_-_esporotricose_humana_na_lista_nacional.pdf; Portaria 11.211/2026. **[INFERÊNCIA]** A esporotricose **animal** não está na lista nacional de epizootias de 2017. Quando é notificada, é por norma estadual ou municipal.
- **[FATO]** **Febre maculosa e outras riquetsioses** é de notificação imediata (MS, SES, SMS). **Leptospirose** é imediata à SMS. **Acidente por animal peçonhento** é imediato à SMS. — Portaria 11.211/2026
- **[FATO]** O ZOO-2016 traz controle de roedores (vigilância, ações diretas, monitoramento da infestação), manejo de escorpiões, controle de vetores (carrapatos, flebotomíneos, triatomíneos e outros) e coleta e envio de espécimes ao laboratório de entomologia. — ZOO-2016, cap. 3–4
- **[FATO]** O recolhimento de animais (recepção, remoção/apreensão, transporte, alojamento, eutanásia) é registrado em "formulário próprio" com assinatura do avaliador. Animais invertebrados levam "ficha" própria. — ZOO-2016, cap. 2
- **[FATO]** Em BH, as solicitações da população (roedores, peçonhentos, vetores, morcegos) entram pelo **156** e pelo portal e viram vistoria. — Projeto Básico SMSA-BH 001/24 (seção 4)

### 3.7 Existe sistema nacional para zoonoses?

- **[INFERÊNCIA]** Não. A UVZ trabalha com peças soltas: **Sinan** (agravos humanos e epizootias), **GAL** (amostras animais), **SIES** (vacina e insumos), **REDCap do MS** (EATAV), **planilha ou formulário estadual** (resultado da campanha antirrábica) e formulários próprios (recolhimento, observação, canil, eutanásia). BH justificou a compra de um sistema próprio porque suas bases "não são integradas e tratam os dados de forma quantitativa", com exceção do sistema de leishmanioses. — Projeto Básico SMSA-BH 001/24

---

## 4. O que os editais municipais pedem

**[FATO]** para os itens com URL. São quatro fontes:

1. **Belo Horizonte/MG — Projeto Básico SMSA/DTIS 001/24, "Implantação da Solução Tecnológica de Zoonoses" (SIGZOO, SaaS, nov./2024)** — https://prefeitura.pbh.gov.br/sites/default/files/estrutura-de-governo/saude/projeto_basico_001.24.pdf
   - O app do ACE deve cadastrar imóveis e registrar visitas em 100% do território (domicílios, praças, TB, comércios, córregos), com depósitos encontrados, tratados e eliminados, coleta de amostras e resultado laboratorial.
   - App **offline** com sincronização (RF28–29). Endereços por quarteirão com tipo de unidade (residência, comércio, TB, outros) e classificação de risco, em mapa e lista (RF30). **GPS automático** com horários de entrada e saída (RF31).
   - Planejamento: ciclos, estratos/amostragem, calendário epidemiológico (RF17–18, RF23–24). Ovitrampas: planejamento, instalação, retirada e contagem (RF21, RF49, RF71). **Raio de UBV em quarteirões** (RF38). Atributo "ponto estratégico" no imóvel (RF55).
   - Amostras: registro de palhetas, larvas e sangue canino, **etiqueta QR Code**, remessa ao laboratório, módulo de laboratório com recebimento, qualidade e laudo (RF57–59, RF70, RF105, RF107–113).
   - Consumo de químicos e insumos por atividade (RF54). Boletim de Operação com Inseticida (RF81). **Censo animal** (RF35, RF117). Histórico do animal (RF50). Supervisão com sorteio aleatório de ACE (RF72–79).
   - **Notificação de caso animal "para exportação para o SINAN"** (RF103). Demandas do portal BH Digital/156 com ciclo de status e retorno (RF82–93, RF123). Integração com bases geográficas e cadastro imobiliário municipal (RF124–126).
   - Volumes que BH declara: 85.006 ovitrampas, cerca de 260 mil vacinações antirrábicas por ano, 51.600 cães examinados e 36.462 encoleirados por ano para LV, 27 mil castrações por ano.

2. **Contagem/MG — Termo de Referência Simplificado, SIGESC (módulo Gestão de Vigilância em Saúde, subitem Zoonoses)** — https://consultapublica.contagem.mg.gov.br/storage/anexos/AnEWIXyIXrCjkgsNY9iTPKSCIlQL8kSLnTwarYuQ.pdf
   - Ovitrampas: número, distrito, endereço, supervisor, histórico de alterações, datas de instalação e recolhimento ligadas ao **ciclo de contagem de ovos**, mapa estratificado por contagem (RFZ3–12).
   - Imóvel com tipo, endereço, quarteirão e distrito, **vinculado a um ACE** (RFZ13–14, obrigatório).
   - Relatórios de visitas por mês, **ciclo focal**, distrito e município, com abertos, fechados e recusados por tipo (residência, comércio, TB, outros). Produção do ACE. PE existentes, abertos, fechados e recusas por ciclo. **Inseticida recebido e utilizado** por tipo. % de imóveis vistoriados no focal (RFZ15–29).
   - **Borrifação** com dupla responsável, categoria (censitário, raio humano), locais (peri/intra), pendência e consumo em cargas (RFZ30).
   - **Demandas espontâneas** (roedores, peçonhentos, **observação de animal agressor**, carrapatos, primatas, morcegos, **esporotricose**) com solicitante, telefone, histórico e permissões (RFZ31–32).
   - **Cadastro de animais** (espécie, raça, sexo, pelagem, **microchip**, procedência), canis e gatis, serviços no canil, prontuário clínico do animal, eutanásia (RFZ33–40+).
   - Cadastro de imunobiológicos e esquemas conforme o PNI (RFZ1).

3. **Pedra Preta/MT — Termo de Referência, licenças de software de gestão de saúde pública com módulo "Gestão em Vigilância Ambiental"** — https://www.pedrapreta.mt.gov.br/fotos_licitacao/6452.pdf. O PDF passa de 10 MB e não abriu; o que segue vem de trecho de busca, com OCR corrompido.
   - Cadastro de locais com coordenadas e proprietário, histórico de visitas, acompanhamento de solicitações de ponto estratégico com encaminhamento e mudança de status, mapeamento georreferenciado. O módulo é voltado a endemias e zoonoses.

4. **Camalaú/PB — Pregão Eletrônico 00029/2023** — https://camalau.pb.gov.br/images/arquivos/documentos/PE_0007_Solicita_o_anex0000.PDF. O documento é escaneado; o que segue vem de trecho de busca.
   - Formulários de controle de endemias: **registro diário do serviço antivetorial**, relatórios do LIRAa e formulários de **controle de Chagas**. Mapas e painéis para o gestor. Integração com o MS por exportação (o alvo não aparece no trecho).

- **[INFERÊNCIA]** Os requisitos que se repetem nos editais: app offline com GPS; imóvel e quarteirão vinculados ao ACE; ciclos e relatórios de abertos, fechados e recusados por tipo de imóvel; PE; ovitrampas com mapa; LIRAa; consumo de inseticida; borrifação e UBV por raio; demandas da população com status; cadastro de animais com microchip; canil e eutanásia; observação de animal agressor; laboratório com etiqueta e laudo; exportação para os sistemas do MS (Sinan, LIRAa).

---

## 5. Integrações relevantes

### 5.1 e-SUS APS / LEDI

- **[FATO]** Na Ficha de Visita Domiciliar e Territorial, o controle vetorial se resume aos motivos **34 ação educativa, 35 imóvel com foco, 36 ação mecânica, 37 tratamento focal**, mais o `tipoDeImovel` (inclui 4, Ponto Estratégico). Não há campo de depósito, amostra, larvicida ou situação F/R/T. — Dicionário LEDI
- **[FATO]** O cadastro domiciliar já tem `stAnimaisNoDomicilio`, `animaisNoDomicilio` (gato, cachorro, pássaro, macaco, galinha, porco, répteis, outros) e `quantosAnimaisNoDomicilio`, e o Rota Saúde já valida esses campos. — Dicionário LEDI; `/Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/api/app/services/territory_registry/rules.rb` (linhas 23, 244–258)
- **[INFERÊNCIA]** O número de animais por domicílio serve de estimativa para a meta da campanha antirrábica e para o denominador das coleiras de LV. Não substitui o cadastro individual do animal.
- **[INFERÊNCIA]** As ações educativas e mobilizações coletivas do ACE poderiam ir na Ficha de Atividade Coletiva. Falta confirmar se o CBO 515140 é aceito nessa ficha pelas regras LEDI.
- **[FATO]** As doses antirrábicas humanas (pós-exposição) vão para o SI-PNI e o Sinan. — NT Conjunta 35/2026. **[INFERÊNCIA]** No e-SUS APS isso cai na ficha ou no registro de vacinação com estratégia pós-exposição; não confirmei o código da estratégia.

### 5.2 Sinan

- **[FATO]** A efetividade do bloqueio depende do fluxo oportuno da notificação para o controle vetorial. — DN-2025, 7.1.4. **[INFERÊNCIA]** A fila de notificação do módulo 25 deve disparar a ordem de bloqueio com o raio e o agrupamento de 7 dias.
- **[FATO]** Atendimento antirrábico humano: notificação imediata à SMS, com campos da condição do animal e do resultado da observação (ver 3.3).
- **[FATO]** Epizootias (Portaria 782/2017) têm ficha própria no Sinan **[INFERÊNCIA quanto ao instrumento; a portaria prevê "formulário" e sistemas definidos pela SVS]**. BH pede notificação de caso animal "para exportação para o SINAN" (RF103).
- **[FATO]** A esporotricose humana é notificada pelo **e-SUS Sinan**, segundo material da SES-ES. **[INFERÊNCIA]** Parte dos agravos já migrou do Sinan Net para o e-SUS Sinan. O módulo 25 precisa saber, por agravo, qual sistema recebe a digitação.

### 5.3 GAL

- **[FATO]** O GAL cobre amostras de origem humana, animal e ambiental, nos módulos Ambiental, Biologia Médica, Animal e Controle de Qualidade. O módulo Animal se divide em **invertebrados** (protocolo de campo para entomologia e malacologia) e **vertebrados** (investigação animal completa ou simplificada). — LACEN-ES (vistos em trecho de busca): https://saude.es.gov.br/Media/sesa/LACEN/GAL/REFERENCIAS%20%20GAL%2021-22-23-11-2022.pdf
- **[FATO]** Em SP, o módulo animal vertebrado passou a ser usado na **vigilância da LV canina** em junho de 2024, com dificuldades de cadastro municipal e encaminhamento (visto em trecho de busca). — https://periodicos.saude.sp.gov.br/RIAL/article/download/41154/38355/49974. **[FATO]** No LACEN-MG, a análise molecular de vetores exige cadastro do vetor no GAL (visto em trecho de busca). — https://www.mg.gov.br/servico/solicitar-identificacao-e-analise-de-vetores-de-doencas
- **[FATO]** Guias do módulo animal: SC https://lacen.saude.sc.gov.br/arquivos/Modulo_Animal_Guia_Rapido.pdf e PR https://lacen.saude.pr.gov.br/sites/lacen/arquivos_restritos/files/documento/2021-09/gal_animal_executor_2021.pdf
- **[INFERÊNCIA]** Não achei API pública do GAL para o município. O caminho realista é digitar no GAL e, no Rota Saúde, guardar o nº da requisição e o resultado (manual ou importado), como já acontece com exames humanos. O ELISA confirmatório e o SNC da raiva vão ao LACEN pelo GAL; o TR-DPP e a larvária costumam ficar no nível local.

### 5.4 Conta Ovos (API)

- **[FATO]** A API tem parte pública (sem chave: ovitrampas, últimas contagens, visitas em quarteirões, quarteirões, EDL, PE) e parte privada, com chave de 45 caracteres via `Authorization: Bearer`. A chave é gerada pelo gestor municipal e tem escopo geográfico definido pelo plano. A parte privada tem **POST para inserir, editar e apagar** leituras, ovitrampas, visitas, quarteirões, EDL, manutenção de EDL e imóveis, mas é "only recommended for applications that work in partnership with Conta Ovos". — https://contaovos.com/en-us/api/
- **[INFERÊNCIA]** Integrar pede acordo de parceria. Sem ele, o Rota Saúde só **lê** pela API pública para cruzar a positividade com as visitas.

### 5.5 Outros

- **[FATO]** O LIRAa municipal exporta um arquivo para a Regional (5.2/1.2). **[INFERÊNCIA]** O Rota Saúde pode gerar o plano amostral e o consolidado no formato do programa, mas o layout do arquivo não é público.
- **[FATO]** O SIES (insumos) é só digitação web (POP 09 SES-GO). **[INFERÊNCIA]** Não tem API conhecida.

---

## Achados que mudam o desenho

1. **O Rota Saúde é o sistema de registro municipal; ninguém espera dado por imóvel no nível nacional.** Não há sistema nacional vigente para a visita do ACE: o SISPNCD virou legado, a DN-2025 fala em "sistema local ou nacional, o que estiver disponível" e o PQA-VS deixou de medir ciclos em 2025. O dado detalhado fica dentro do produto. Para fora saem a ficha LEDI (4 motivos), o arquivo do LIRAa e adaptadores **por UF** (SP Sisaweb, GO SIMAZ), que entram como integração opcional e não como núcleo.
2. **Uma visita do ACE gera dois registros.** O boletim rico (situação, depósitos A1–E por existentes, com água e com larva, amostras, larvicida em cargas) e a ficha LEDI derivada (motivo 34–37 + tipo de imóvel). O app de campo em construção precisa capturar o boletim inteiro, porque a ficha LEDI não comporta o detalhe.
3. **O ciclo fixo de 60 dias em 100% dos imóveis deixou de ser a norma.** A DN-2025 e a NT41-2026 põem a cadência por área (hotspot/coldspot, a partir da estratificação e da positividade das ovitrampas) e por classe de risco do imóvel (verde, amarelo, vermelho, cinza, roxo). O LIRAa passa a ser ao menos anual. O planejamento de ciclos tem de ser parametrizável por área e imóvel; ciclo bimestral codificado de forma fixa é erro.
4. **O MS distribui o Conta Ovos para ovitrampas, e ovitrampa é pré-requisito das tecnologias novas.** O Conta Ovos tem API de escrita para parceiros e cobre também quarteirões, PE, EDL e visitas. A decisão é integrar ou só ler, e não construir um módulo concorrente. Se for módulo próprio, precisa da etiqueta com código IBGE, do TCLE do morador e do raio de 200 m.
5. **O bloqueio de transmissão sai da fila de notificação do módulo 25.** O caso notificado gera uma ordem com raio de 150 m, agrupamento de casos em 7 dias e até 3 ciclos de UBV. Ele suspende Wolbachia e TIE na área. O tempo entre notificação e bloqueio vira indicador. O mesmo gatilho vale para raiva (bloqueio de foco em 72 h) e LV (investigação de foco com ≥100 cães).
6. **O módulo precisa de uma entidade "animal" individual (cão/gato) ligada ao domicílio do cadastro territorial.** Ela leva identificação (microchip ou RG animal), vacinação antirrábica, sorologias TR-DPP e ELISA, coleira (troca a cada 6 meses), observação de 10 dias e destino (eutanásia, canil). Quem exige: o manual de LV de 2026 e os editais de BH e Contagem. A contagem `quantosAnimaisNoDomicilio` que já existe serve só de denominador. Cuidado com LGPD: o tutor é cidadão identificado.
7. **O atendimento antirrábico une UBS, UVZ e Sinan.** É notificação imediata à SMS, e a ficha Sinan tem os campos "passível de observação" e "condição final do animal". O fluxo desenhado: UBS notifica, a UVZ observa por 10 dias, o desfecho volta para fechar a ficha e ajustar o esquema (SI-PNI e Sinan). O resultado da campanha antirrábica é agregado e semanal por planilha estadual, sem registro individual nacional. Nenhum sistema federal recebe essa informação, então o produto pode gerar o consolidado no formato da UF.
8. **Demandas da população e insumos são requisitos de edital que se cruzam com módulos já existentes.** O 156 e a ouvidoria (roedor, escorpião, morcego, agressor, esporotricose) pedem fila com status e retorno ao cidadão: reaproveitar o módulo 33. Larvicida, inseticida e vacina antirrábica circulam pelo SIES e o consumo é medido em cargas no boletim: reaproveitar estoque e farmácia (27a), com produto e carga parametrizáveis, porque os produtos mudam por nota técnica.
