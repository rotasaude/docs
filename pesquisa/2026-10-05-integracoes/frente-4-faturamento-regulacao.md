# Frente 4: faturamento e regulação de média e alta complexidade (MAC)

Pesquisa feita em 2026-10-05 para os módulos PROCEDIMENTOS e REGULAÇÃO do Rota Saúde (e, só como referência, para a futura linha hospitalar).

Convenção: **[CONFIRMADO: url]** = lido na fonte nesta pesquisa. **[INFERÊNCIA]** = dedução minha, ou dado vindo de fonte secundária que não consegui conferir na oficial. Não inventei número de portaria, versão nem prazo. Quando a fonte é uma secretaria estadual, o COSEMS, o CONASS ou o CONASEMS (e não o Ministério), isso aparece na URL.

---

## Resumo executivo

1. **Faturamento ambulatorial ainda passa pelo SIA/SUS e por programas desktop do DATASUS.** O estabelecimento registra a produção em BPA-C, BPA-I, APAC ou RAAS. O gestor (município ou estado) importa, consiste contra CNES, SIGTAP e FPO e transmite ao Ministério por competência mensal. Um sistema de terceiro **pode gerar o arquivo texto do BPA** no layout publicado pelo DATASUS, que é importado no BPA Magnético. Assim fazem os sistemas privados, o SISREG e o e-SUS APS. Gerar BPA-I é, portanto, uma integração viável e de baixo risco.
2. **O Ministério está trocando o caminho:** a RNDS é declarada a "fonte oficial" do Agora Tem Especialistas. O CMD (estabelecimento sem prontuário) e o RAC (estabelecimento com prontuário) vão substituir SIA e SIH, ainda **sem data limite** (Portaria GM/MS 7.495/2025; modelo do CMD na Portaria GM/MS 11.952/2026). O Rota Saúde deve tratar o BPA como ponte e a RNDS como destino.
3. **Na regulação, a novidade é de 2025.** A Portaria GM/MS 6.656/2025 obriga **todo registro de solicitação ou encaminhamento à atenção especializada** a ir para a RNDS no modelo MIRA. Sistema próprio ou de terceiro envia **diariamente**. Se a prefeitura usar o Rota Saúde como sistema de regulação, o envio à RNDS (FHIR R4, certificado ICP-Brasil) vira requisito de produto. Sem ele, há risco de a cidade ficar impedida de aderir a programas de cirurgias eletivas.
4. **O SISREG III não aceita novas adesões** e está sendo substituído pelo **e-SUS Regulação**, que é gratuito, por enquanto só tem o módulo ambulatorial e já integra com a RNDS. Não existe API de escrita documentada nos dois. A API do SISREG é de leitura, restrita a secretarias. Conviver com eles hoje quase sempre significa dupla digitação ou importação de arquivo.
5. **Recomendação:** construir em camadas.
   - (1) Catálogo SIGTAP mensal, registro do procedimento com as críticas da tabela e exportação de BPA-I/BPA-C.
   - (2) Regulação municipal própria (fila, classificação de risco, cotas, agendamento, confirmação), com envio MIRA à RNDS. Até a camada 2 ficar pronta, operar em convivência com SISREG ou e-SUS Regulação.
   - (3) APAC/OCI, FPO e conciliação do faturamento.
   - (4) RAC/CMD na RNDS.
   - AIH e SISAIH01 só na linha hospitalar.

---

## 1. Definições: atenção básica, média e alta complexidade, atenção especializada

### 1.1 Atenção básica (APS)
- A PNAB (Portaria nº 2.436, de 21/09/2017) define a atenção básica como o conjunto de ações individuais, familiares e coletivas de promoção, prevenção, diagnóstico, tratamento etc., feitas por equipe multiprofissional em território definido. Ela é a "principal porta de entrada" e ordenadora da Rede de Atenção à Saúde. A norma trata AB e APS como termos equivalentes. [CONFIRMADO: https://bvsms.saude.gov.br/bvs/saudelegis/gm/2017/prt2436_22_09_2017.html]

### 1.2 Média e alta complexidade (definição clássica do MS/SAS)
- **Média complexidade ambulatorial:** "ações e serviços que visam atender aos principais problemas e agravos de saúde da população, cuja complexidade da assistência na prática clínica demande a disponibilidade de profissionais especializados e a utilização de recursos tecnológicos, para o apoio diagnóstico e tratamento". Inclui procedimentos especializados, cirurgias ambulatoriais, traumato-ortopedia, odontologia especializada, patologia clínica, radiodiagnóstico, ultrassonografia, fisioterapia, órteses e próteses e anestesia. [CONFIRMADO: https://www.conass.org.br/bibliotecav3/pdfs/colecao2011/livro_4.pdf (CONASS, citando a SAS/MS e "O SUS de A a Z")]
- **Alta complexidade:** "conjunto de procedimentos que, no contexto do SUS, envolve alta tecnologia e alto custo", organizado em redes: terapia renal substitutiva, oncologia, cirurgia cardiovascular, neurocirurgia, traumato-ortopedia, implante coclear e outras. [CONFIRMADO: mesma fonte]
- **No SIGTAP, o nível de complexidade é um atributo de cada procedimento.** A complexidade "indica a especialização e os recursos tecnológicos exigidos". [CONFIRMADO: https://wiki.saude.gov.br/sigtap/index.php/Gerais]
- [INFERÊNCIA] Os valores do atributo "complexidade" (atenção básica / média / alta / não se aplica) estão no arquivo da tabela. Não confirmei a lista exata de códigos porque a página da wiki está vazia.

### 1.3 Atenção especializada: PNAES (2023)
- A **Portaria GM/MS nº 1.604, de 18/10/2023**, institui a Política Nacional de Atenção Especializada em Saúde (PNAES). Ela define atenção especializada como o conjunto de "conhecimentos, práticas assistenciais, ações, técnicas e serviços [...] marcados, caracteristicamente, por uma maior densidade tecnológica" (art. 1º, §1º). A política cobre urgência, reabilitação, atenção domiciliar, rede hospitalar, materno-infantil, transplantes, atenção psicossocial, sangue e **atenção ambulatorial especializada**. [CONFIRMADO: https://bvsms.saude.gov.br/bvs/saudelegis/gm/2023/prt1604_20_10_2023.html]
- A PNAES **não usa** os termos "média/alta complexidade". O recorte é por densidade tecnológica e linha de cuidado. [CONFIRMADO: idem]
- Regulação (arts. 21–30): o acesso regulado deve garantir equidade e transparência, ser organizado por linhas de cuidado e priorizado por critério clínico. O gestor estadual ou municipal regula com protocolos, e os serviços disponibilizam sua oferta às centrais de regulação. [CONFIRMADO: idem]
- Financiamento (arts. 45–46): tripartite, com "substituição gradativa" do pagamento por procedimento por remuneração centrada no cuidado integral e contratualização com metas. [CONFIRMADO: idem]

### 1.4 PMAE e "Agora Tem Especialistas"; OCI
- O **PMAE** (Programa Mais Acesso a Especialistas) foi instituído pela Portaria GM/MS nº 3.492/2024 e operacionalizado pela Portaria SAES/MS nº 1.640, de 07/05/2024, que criou a habilitação 38.01 no CNES. [CONFIRMADO: https://www.gov.br/saude/pt-br/centrais-de-conteudo/publicacoes/guias-e-manuais/2024/manual-pmae-registro-da-producao-controle-e-avaliacao.pdf ; https://www.cosemssp.org.br/wp-content/uploads/2026/05/Katia-Vital-Navarro-Watanabe.pdf]
- O **Agora Tem Especialistas** foi instituído por MP em 2025 e convertido na **Lei nº 15.233, de 07/10/2025** (vigência até 31/12/2030, com prestação também por privados e operadoras de planos). Foi regulamentado pela Portaria GM/MS nº 7.266, de 18/06/2025. [CONFIRMADO: https://www.lexml.gov.br/urn/urn:lex:br:federal:lei:2025-10-07;15233 ; https://www.gov.br/planalto/pt-br/acompanhe-o-planalto/noticias/2025/10/lula-sanciona-lei-do-agora-tem-especialistas-e-regulamenta-lei-de-pesquisa-clinica-para-fortalecer-a-saude-publica ; Portaria 7.266 citada em https://www.cosemssp.org.br/wp-content/uploads/2026/05/Katia-Vital-Navarro-Watanabe.pdf]
- **OCI (Oferta de Cuidados Integrados)** é um procedimento principal do **Grupo 09 do SIGTAP**. Agrupa consulta, exames e tecnologias de cuidado para fechar uma etapa (diagnóstico ou tratamento) dentro de um prazo.
  - Registro em **APAC única**, sem APAC de continuidade, com **quinto dígito "7"** no número da autorização.
  - O início da validade é a data do 1º procedimento. Há atributo "APAC com validade fixa de 2 competências".
  - Os procedimentos secundários vêm com valor zerado e financiamento FAEC (regra condicionada 0012, Portaria SAES/MS nº 2.630/2025), e precisam ser programados na aba FAEC da FPO.
  - Se a OCI não for concluída, os procedimentos realizados podem ir em BPA-I.

  [CONFIRMADO: manual PMAE acima]
- O controle do programa cruza a **fila nominal informada** (SISREG, e-SUS Captação de Filas, e-SUS Regulação **ou sistemas terceiros**) com APAC, BPA, AIH, CNES e SIGTAP. Uma das métricas é a data de início da APAC contra a data do agendamento regulado. [CONFIRMADO: manual PMAE]
- No Agora Tem Especialistas:
  - a RNDS é a "fonte oficial de dados do Programa" (modelos de informação no art. 5º da Portaria GM/MS nº 7.495/2025);
  - a identificação por **CPF é obrigatória** (indígenas podem usar CNS);
  - é preciso **número de autorização emitido pelo gestor**, que "também executa a regulação conforme fluxos locais".

  [CONFIRMADO: https://www.cosemssp.org.br/wp-content/uploads/2026/05/Katia-Vital-Navarro-Watanabe.pdf (apresentação COSEMS-SP, mai/2026)]
- Quinto dígito da APAC por componente: 7 = OCI; 6 = cirurgias eletivas; 9 = créditos financeiros e complementar. Na AIH, o quinto dígito "8" aparece em algumas modalidades, junto com o campo "Fonte Orçamentária". [CONFIRMADO: mesma apresentação COSEMS-SP] [INFERÊNCIA: o mapeamento exato modalidade→dígito deve ser conferido nas portarias SAES antes de implementar]
- Normas recentes do componente ambulatorial: a Portaria SAES/MS nº 4.107, de 11/05/2026, incluiu o atributo complementar "063 – Exige procedimento de diagnóstico em oftalmologia". [CONFIRMADO: https://www.conass.org.br/que-estabelece-no-ambito-do-programa-agora-tem-especialistas-criterios-operacionais-referentes-ao-componente-ambulatorial-e-inclui-atributo-complementar-na-tabela-de-procedimentos-medicamentos-or/]

### 1.5 Financiamento: bloco MAC, teto MAC e FAEC
- **Histórico:** o Pacto pela Saúde criou seis blocos. Um deles era o de MAC, com dois componentes: **Limite Financeiro MAC** ("teto MAC") e **FAEC** (Fundo de Ações Estratégicas e Compensação). Os procedimentos do FAEC iriam sendo incorporados gradualmente ao teto. [CONFIRMADO: https://www.conass.org.br/bibliotecav3/pdfs/colecao2011/livro_4.pdf]
- A **Portaria GM/MS nº 3.992, de 28/12/2017**, reduziu os blocos a dois, **Custeio** e **Investimento**. Referências ao antigo "Bloco de MAC" passaram a valer como Bloco de Custeio. [CONFIRMADO: https://bvsms.saude.gov.br/bvs/saudelegis/gm/2017/prt3992_28_12_2017.html]
- A **Portaria GM/MS nº 828, de 17/04/2020**, renomeou os blocos para **Manutenção** e **Estruturação**. Dentro da Manutenção ficam grupos de identificação: Atenção Primária, **Atenção Especializada**, Assistência Farmacêutica, Vigilância e Gestão. Portarias de repasse recentes ainda falam em incorporar recursos "ao Limite Financeiro de MAC" a partir do "Bloco de Manutenção – Grupo de Atenção Especializada". [CONFIRMADO: https://bvsms.saude.gov.br/bvs/saudelegis/gm/2020/prt0828_24_04_2020.html (via busca) ; https://www.conass.org.br/conass-informa-n-440-2020-publicada-a-portaria-gm-n-3-822-que-estabelece-recurso-financeiro-do-bloco-de-manutencao-das-acoes-e-servicos-publicos-de-saude-grupo-de-atencao-especializada/]
- No SIGTAP, cada procedimento tem **tipo de financiamento**: PAB, **MAC**, **FAEC**, Incentivo MAC, Assistência Farmacêutica, Vigilância e Gestão. [CONFIRMADO: https://wiki.saude.gov.br/sigtap/index.php/Gerais]
- A **FPO Magnética** é onde o gestor programa, por estabelecimento, a quantidade física e o orçamento (por grupo, subgrupo, forma de organização ou procedimento) e define limites por tipo de financiamento (PAB, MAC, FAEC). O SIA consiste a produção contra a FPO. [CONFIRMADO: http://w3.datasus.gov.br/sia/index.php?area=2&id=18790&assunto=11581 (via busca) ; https://wiki.saude.gov.br/sia/index.php/P%C3%A1gina_principal]
- [INFERÊNCIA] Para o produto, isto significa que produção MAC acima do programado na FPO é glosada ou bloqueada no SIA. Um painel de "produção × FPO × teto" é a funcionalidade que o secretário de saúde mais valoriza no faturamento.

---

## 2. Sistema por sistema

### 2.1 SIGTAP: Tabela de Procedimentos, Medicamentos e OPM do SUS

| Item | Achado |
|---|---|
| O que é | É a tabela que define descrição, grupo, subgrupo, forma de organização, complexidade, financiamento, valores e regras de compatibilidade de cada procedimento. Substituiu as tabelas separadas do SIA e do SIH em jan/2008. [CONFIRMADO: https://wiki.datasus.gov.br/sigtap/index.php/Download] |
| Código | 10 dígitos, no formato **GR.SB.FO.PPP-D**: grupo, subgrupo, forma de organização, sequencial e dígito verificador (módulo 11). [CONFIRMADO: https://wiki.saude.gov.br/sigtap/index.php/Gerais] |
| Grupos | 01 Ações de promoção e prevenção · 02 Finalidade diagnóstica · 03 Clínicos · 04 Cirúrgicos · 05 Transplantes · 06 Medicamentos · 07 OPM · 08 Ações complementares · **09 Ofertas de Cuidados Integrados**. Em mai/2025 eram 4.866 procedimentos no total, 29 deles no grupo 09. [CONFIRMADO: https://wiki.saude.gov.br/sigtap/index.php/Grupo] |
| Atributos relevantes | **Instrumento de registro** (BPA-C, BPA-I, APAC principal/secundário, AIH principal/secundário, RAAS); **CBO** que pode executar; **idade** mínima e máxima (0–130); **sexo**; **CID** principal e secundário compatíveis; **complexidade**; **financiamento**; habilitação ou serviço/classificação exigidos; compatibilidades entre procedimentos; atributos complementares. [CONFIRMADO: https://wiki.saude.gov.br/sigtap/index.php/Gerais ; manual PMAE] |
| Quem opera | DATASUS e SAES/MS. Atualização por portaria, com Nota Técnica a cada competência. [CONFIRMADO: https://wiki.datasus.gov.br/sigtap/index.php/Download] |
| Periodicidade | **Mensal, por competência.** A competência pode ser republicada (versão com carimbo de data e hora). Hoje a página de download lista `TabelaUnificada_202609_v2610050950.zip`, `TabelaUnificada_202608_v2608141139.zip` e `TabelaUnificada_202607_v2607101010.zip`. Um espelho no GitHub tem a mesma 202609 com outra versão (`v2609171117`), o que mostra que houve republicação. [CONFIRMADO: http://sigtap.datasus.gov.br/tabela-unificada/app/download.jsp (lido via curl) ; https://github.com/RenatoKR/SIGTAP] |
| Formato | ZIP com arquivos **.txt de largura fixa, sem separador**, cada um com seu arquivo de layout, mais a Nota Técnica em PDF da competência. Há também a planilha de mapeamento SIGTAP↔TUSS da ANS. [CONFIRMADO: https://wiki.datasus.gov.br/sigtap/index.php/Download] Um projeto aberto fala em "pouco mais de 80 arquivos" por competência. [CONFIRMADO: https://github.com/RicardoHerrero/SIGTAP-dataSUS] |
| Onde baixar | Os links da página apontam para **FTP** (`ftp://ftp2.datasus.gov.br/pub/sistemas/tup/downloads/`). O próprio DATASUS avisa que o Chrome não suporta FTP. [CONFIRMADO: download.jsp ; wiki Download] Daqui, o FTP abriu a sessão de controle, mas a transferência em modo passivo expirou (timeout). [CONFIRMADO: teste local com curl] |
| Obrigatório? | Na prática, sim, para qualquer registro de produção: o SIA consiste o BPA/APAC contra o SIGTAP da competência. [CONFIRMADO: https://wiki.saude.gov.br/sia/index.php/P%C3%A1gina_principal] |
| Credenciamento / custo | Nenhum. É dado público e gratuito. [CONFIRMADO: páginas de download públicas] |
| Módulos do Rota Saúde | **Procedimentos** (catálogo e validações), **Regulação** (o que se solicita é procedimento SIGTAP), **Profissionais** (o CBO já está cadastrado, módulo 10, e serve para cruzar com o CBO compatível), Analytics. |
| Detalhes de implementação | [INFERÊNCIA] Nomes de arquivo como `tb_procedimento`, `rl_procedimento_cid`, `rl_procedimento_ocupacao` e `rl_procedimento_registro` vêm de projetos abertos. Não consegui abrir o ZIP oficial pelo FTP para confirmar. Os layouts mudam de vez em quando: um projeto aberto diz que o tamanho do campo de valor mudou em 2026-09. **Isso não está confirmado** e precisa ser conferido no layout oficial da competência 202609. |

### 2.2 SIA/SUS: BPA-C, BPA-I, APAC, RAAS e FPO

**O que é.** O SIA é o sistema nacional da produção ambulatorial financiada pelo SUS. O ciclo é: atendimento → instrumento de registro → envio ao gestor → críticas → aprovação ou rejeição → disseminação. Só a produção aprovada vira dado público. [CONFIRMADO: https://rfsaldanha.github.io/sis/sia.html (livro aberto); https://wiki.saude.gov.br/sia/index.php/P%C3%A1gina_principal]

**Instrumentos**

| Instrumento | Uso | Fonte |
|---|---|---|
| BPA-C | Produção agregada (procedimento × quantidade × CBO × idade), sem identificar o usuário e sem autorização | [CONFIRMADO: https://rfsaldanha.github.io/sis/sia.html] |
| BPA-I | Produção individualizada: paciente (CNS/CPF), profissional (CNS), CBO, data, CID, município de residência etc. Desde 2008 | [CONFIRMADO: idem] |
| APAC | Procedimentos de maior complexidade e custo, que **exigem autorização prévia (laudo) do gestor** e numeração em faixa. Quimio, radio, TRS, medicamentos especializados, OCI | [CONFIRMADO: idem; manual PMAE] |
| RAAS | Atenção psicossocial e atenção domiciliar | [CONFIRMADO: idem] |

**Quem envia.** O estabelecimento (público, conveniado ou contratado) registra a produção no **BPA Magnético** (ou APAC Magnético / RAAS) e exporta uma "remessa" para o gestor. O **gestor** (secretaria municipal ou estadual, conforme quem gere o prestador) importa no **SIA**, que consiste contra CNES, FPO e SIGTAP, e envia a base ao Ministério pelo **Transmissor**. [CONFIRMADO: https://wiki.saude.gov.br/sia/index.php/P%C3%A1gina_principal ; https://www.fehosp.com.br/files/manuais/5a6995cb091895b75c83d0ed39a0bb45.pdf (manual BPA, via busca)]

**Programas DATASUS (versões vigentes na página oficial em 05/10/2026)**
- BPA: `BPAMAG0500.exe`, `Layout_Exportacao_BPA.pdf` e `BPA_LEIAME.txt`, todos de **09-Jul-2026**. [CONFIRMADO: https://sia.datasus.gov.br/versao/listar_ftp_bpa.php]
- APAC: `APACMAG_0402.exe`, `layout_Exportacao_APAC.pdf`. [CONFIRMADO: https://sia.datasus.gov.br/versao/listar_ftp_apac.php]
- SIA: `SIA0605.exe` e bases de competência `BDSIA202609a.exe`. [CONFIRMADO: https://sia.datasus.gov.br/versao/listar_ftp_sia.php]
- FPO: `FPOMAG_Atualiza_0303.exe`. [CONFIRMADO: https://sia.datasus.gov.br/versao/listar_ftp_fpo.php]
- RAAS: `RAAS_0235.exe`, `Layout_Exportacao_RAAS.pdf`. RAAS 3.0 anunciado para 20/08/2026, e o SIA "só aceitará a versão igual ou superior a 3.0". [CONFIRMADO: https://sia.datasus.gov.br/versao/listar_ftp_raas.php ; http://sia.datasus.gov.br/principal/index.php]
- São todos executáveis **Windows**, distribuídos por FTP. [CONFIRMADO: páginas acima]

**Prazos e competência**
- O Transmissor do SIA abre por janela de competência. Na página hoje: competência 08/2026 de 14/09/2026 a 05/10/2026; 07/2026 de 14/08/2026 a 05/10/2026. [CONFIRMADO: http://sia.datasus.gov.br/principal/index.php]
- A SAES publica um cronograma anual (SIGTAP → CNES → remessa SIA/SIH → TabNet). Exemplo para jan/2026: SIGTAP 05/01, CNES Desktop 14/01, remessa CNES 06/02, **remessa SIA/SIH 27/02**. [CONFIRMADO só em fonte secundária: https://p2saude.com.br/ministerio-da-saude-divulga-cronograma-2026-para-sistemas-cnes-sia-e-sih/] [INFERÊNCIA: o envio do prestador ao gestor é anterior e definido localmente pelo município, normalmente nos primeiros dias úteis do mês seguinte]
- A **competência de apresentação** precisa coincidir com a competência de processamento do SIA. [CONFIRMADO: https://info.saude.df.gov.br/wp-content/uploads/2023/11/Manual_Operacional_APAC_v_1_1.pdf (via busca)]

**Identificação do paciente.** A partir da competência 01/2025, **não se pode informar CPF e CNS no mesmo atendimento**: usa-se um só, de preferência o **CPF**. [CONFIRMADO: http://sia.datasus.gov.br/principal/index.php (aviso de 12/12/2024)]

**Um terceiro pode gerar o arquivo BPA? Sim.**
- O DATASUS publica o **layout da interface texto do BPA** para gerar arquivo importável no BPA Magnético. O BPA Mag tem um menu de **importação** que aceita "arquivos gerados por outros sistemas próprios", porque só ele gera a remessa final para o gestor. [CONFIRMADO: https://sia.datasus.gov.br/versao/listar_ftp_bpa.php ; https://www.fehosp.com.br/files/circulares/3c765d77164dc09521d225a9d2386844.pdf ; resumo do manual do SISREG executante em https://saude.sc.gov.br/index.php/pt/regulacao/manuais/manual-sisreg-executante-ambulatorial-28-06-2016/download (via busca)]
- **Precedentes:** o **SISREG** gera "Geração de Arquivo (TXT)" de BPA para o executante importar [CONFIRMADO (via busca): https://wiki.saude.gov.br/SISREG/index.php/Categoria:GERANDO_BPA, que estava fora do ar quando acessei]. O **SISAB/e-SUS APS** gera arquivo BPA-I para importar no BPA Mag/SIA (Nota Técnica nº 10/2024-CGIAD/SAPS/MS). [CONFIRMADO: https://sisab.saude.gov.br/resource/file/nota_tecnica_arquivo_bpa_i_241025.pdf]
- **Estrutura do layout** (versão reproduzida pela FEHOSP em 09/2020; os campos são estáveis, mas a versão vigente é a de 09-Jul-2026 e **precisa ser conferida**):
  - Texto ASCII de largura fixa, linhas terminadas em CR+LF.
  - **Header** "01" + `#BPA#` + competência AAAAMM + nº de linhas + nº de folhas + **campo de controle** (soma módulo 1111, +1111) + órgão de origem, CNPJ/CPF, órgão de destino e indicador M/E (municipal/estadual) + versão.
  - **Linha "02" (BPA-C):** CNES, competência, CBO, folha/sequência (até 20 linhas por folha), procedimento, idade, quantidade, origem.
  - **Linha "03" (BPA-I):** CNES, competência, CNS do profissional, CBO, data do atendimento, folha/sequência, procedimento, CNS do paciente, sexo, IBGE de residência, CID-10, idade, quantidade, caráter do atendimento, nº de autorização, origem, nome, data de nascimento, raça/cor, etnia, nacionalidade, serviço/classificação, equipe (seq/área/INE), CNPJ, CEP e endereço, telefone, e-mail.

  [CONFIRMADO: https://www.fehosp.com.br/files/circulares/3c765d77164dc09521d225a9d2386844.pdf] [INFERÊNCIA: o layout de 2026 deve ter campo de CPF do paciente ou trocar o CNS por CPF, por causa da regra de 01/2025, e talvez o campo "Pessoa sem CPF/Registro Civil" citado em fonte secundária. Não consegui baixar o PDF oficial: o FTP do DATASUS não completou a transferência daqui]
- **Como os sistemas privados fazem** [INFERÊNCIA, baseada no padrão acima e em manuais de fornecedores como TOTVS e Argus encontrados na busca]: o prontuário ou ERP gera o TXT no layout de interface, o faturista importa no BPA Mag, corrige as críticas e exporta a remessa para a secretaria. Na secretaria, o SIA processa. Nenhum fornecedor "pula" o BPA Mag e o SIA. A secretaria sempre opera o SIA.

**APAC: pontos que pesam no produto**
- Numeração `UFAAX0000000/Y`: UF IBGE, ano, quinto dígito de tipo, sequencial de 7 dígitos e dígito verificador. **O gestor gera e distribui as faixas.** [CONFIRMADO: manual PMAE ; apresentação COSEMS-SP]
- Exige laudo, autorização, validade em competências, procedimento principal com secundários compatíveis e motivo de saída. [CONFIRMADO: manual PMAE]
- Portaria SAS/MS nº 1.011/2014 trata de **laudo digital e certificação digital** no SIA e no SIH. [CONFIRMADO só como menção: http://sia.datasus.gov.br/principal/index.php (aviso de 03/11/2015)] [INFERÊNCIA: precisa ser lida antes de desenhar a emissão eletrônica de laudo de APAC]

**FPO.** O gestor registra a programação físico-orçamentária por estabelecimento, com limite por financiamento. Toda produção apresentada, inclusive os secundários de OCI, precisa estar programada. [CONFIRMADO: manual PMAE ; http://w3.datasus.gov.br/sia/index.php?area=2&id=18790&assunto=11581 (via busca)]

**Resumo do SIA para o Rota Saúde**

| Aspecto | Resposta |
|---|---|
| Quem opera | Estabelecimento (BPA/APAC Mag); gestor municipal ou estadual (SIA, FPO, autorização de APAC); DATASUS (base nacional) |
| Obrigatório para o município? | Sim, enquanto houver produção SUS a faturar. Sem SIA não há registro de produção MAC nem prestação de contas por produção. [INFERÊNCIA] |
| Padrão técnico | TXT de largura fixa (layouts DATASUS); programas Windows; FTP |
| Credenciamento | Nenhum para gerar o TXT. O estabelecimento precisa estar no CNES com habilitação e serviço/classificação compatíveis. [INFERÊNCIA a partir do papel do CNES na consistência] |
| Custo | Programas gratuitos [CONFIRMADO: páginas de versão públicas] |
| Documentação | sia.datasus.gov.br (Versões, Documentos), wiki.saude.gov.br/sia |
| Módulos do Rota Saúde | **Procedimentos** (gera BPA-I/BPA-C), **Regulação** (número de autorização, APAC), Agendamento, Desfecho, Profissionais (CNS/CPF e CBO), Unidades (CNES) |

### 2.3 SIH/SUS: AIH e SISAIH01 (só referência para a linha hospitalar)
- O **SISAIH01** é o programa gratuito do DATASUS onde o prestador digita as AIH autorizadas e exporta a produção para o gestor, que processa no **SIHD** (SIH Descentralizado). Pode-se usar outro sistema, mas "a informação final deve ser entregue por meio do SISAIH01" para validação. O SISAIH01 tem layout de interface texto para importação. [CONFIRMADO (via busca): http://w3.datasus.gov.br/sihd/Manuais/LAYOUT_SISAIH01_201010.pdf ; http://w3.datasus.gov.br/sihd/Manuais/MANUAL_SISAIH01_SIH_MODULO_II_VERSAO_AGOSTO_2008.pdf ; https://www.gov.br/hubrasil/pt-br/hospitais-universitarios/regiao-nordeste/hujb-ufcg/acesso-a-informacao/gestao-documental/superintendencia/POP.STCOR.014DigitaodeAIHnoSISAIH01.pdf]
- A AIH também usa o quinto dígito para identificar programas (por exemplo, "8" em modalidades do Agora Tem Especialistas). [CONFIRMADO: apresentação COSEMS-SP]
- [INFERÊNCIA] Para a linha hospitalar valem os mesmos passos do BPA: gerar o TXT de interface do SISAIH01, deixar a autorização da AIH com o gestor e caminhar para o Sumário de Alta na RNDS (Portaria GM/MS nº 8.026/2025, citada em https://www.conass.org.br/conass-informa-n-151-2025-publicada-a-portaria-gm-n-8-026-que-institui-o-modelo-de-informacao-do-sumario-de-alta-sa-no-ambito-da-rede-nacional-de-dados-em-saude-rnds/). Não pesquisei a fundo.

### 2.4 Para onde o faturamento está indo: RNDS, CMD e RAC
- A **Portaria GM/MS nº 7.495, de 04/08/2025**, institui o Componente SUS Digital do Agora Tem Especialistas.
  - Art. 5º: estabelecimento **com prontuário eletrônico** envia RAC (Registro de Atendimento Clínico), Sumário de Alta e Sumário de Alta Obstétrico.
  - Estabelecimento **sem prontuário** envia o **CMD** (Conjunto Mínimo de Dados), com o software "CMD Coleta".
  - Quem já usa SIA/SIH "poderá manter o envio [...] até que a transição para o CMD [...] ou prontuário eletrônico devidamente integrado à RNDS estejam concluídos". **Não há data limite.**

  [CONFIRMADO: https://bvsms.saude.gov.br/bvs/saudelegis/gm/2025/prt7495_05_08_2025.html]
- A **Portaria GM/MS nº 11.952, de 13/07/2026**, institui o Modelo de Informação do CMD, com efeito operacional na competência seguinte. As especificações técnicas ficam com o DATASUS. [CONFIRMADO: https://www.conass.org.br/altera-a-portaria-de-consolidacao-no-1-de-28-de-setembro-de-2017-para-instituir-o-modelo-de-informacao-do-conjunto-minimo-de-dados-cmd/]
- **Integração com a RNDS:** o pedido é feito no Portal de Serviços do DATASUS, com **certificado digital ICP-Brasil (e-CNPJ)**, homologação antes da produção e envio em **FHIR R4** por HTTPS. [CONFIRMADO: https://rnds-guia.saude.gov.br/docs/introducao/] Quem integra é o gestor do ente que usa o sistema, "por meio de ação própria ou do desenvolvedor do sistema próprio ou terceiro". [CONFIRMADO: Portaria GM/MS nº 6.656/2025, via https://bvsms.saude.gov.br/bvs/saudelegis/gm/2025/prt6656_10_03_2025.html (busca)]
- [INFERÊNCIA] O Rota Saúde já faz o atendimento (check-in, chamada, desfecho). Ele está mais perto de um **RAC/prontuário integrado à RNDS** do que de um faturador. Para o médio prazo, a aposta mais segura é a RNDS, e não o BPA.

### 2.5 Regulação: SISREG III

| Item | Achado |
|---|---|
| O que é | Software web gratuito do Ministério para gerir o Complexo Regulador (APS → atenção especializada), com módulos **ambulatorial** (consultas e exames) e **hospitalar** (internações). [CONFIRMADO: https://www.gov.br/saude/pt-br/composicao/saes/drac/regulacao-do-acesso/sistemas-de-informacao] |
| Situação | "por conta de limitações estruturais, **não há possibilidade de novas adesões**". Está "sendo gradativamente substituído pelo e-SUS Regulação". [CONFIRMADO: mesma URL ; https://webatendimento.saude.gov.br/faq/sisreg] |
| Quem opera | Centrais de regulação **municipais ou estaduais**, conforme pactuação na CIB. Perfis: administrador municipal, coordenador de unidade, **solicitante**, **executante**, **regulador/autorizador**, videofonista. A configuração depende de CNES, CNS e **PPI**. [CONFIRMADO (via busca): manuais SES-MT/SES-SC, por exemplo https://www.saude.mt.gov.br/storage/old/files/plano-de-implantacao-sisreg-iii-[178-250811-SES-MT].pdf] |
| Fluxo | Solicitante (UBS) insere na fila → regulador classifica o risco e autoriza, devolve ou nega → agenda (cota) → executante confirma com chave gerada pelo sistema → executante gera o TXT de BPA. [CONFIRMADO (via busca): manuais SES-SC de solicitante e executante] |
| API | Existe uma "API" para dar, "em tempo real, os dados de regulação do acesso, de forma estruturada e controlada". O acesso é **restrito às Secretarias de Saúde**, mediante ofício do gestor e Termo de Responsabilidade e Confidencialidade. [CONFIRMADO: https://www.gov.br/saude/pt-br/composicao/saes/drac/regulacao-do-acesso/sistemas-de-informacao ; https://webatendimento.saude.gov.br/faq/sisreg] Uma fonte secundária indica que o acesso é por Elasticsearch (`elasticsearch-saps.saude.gov.br`), ou seja, **somente leitura**. [INFERÊNCIA: não confirmei a URL em página oficial e não achei operação de escrita documentada] Também existe uma ferramenta de BI com dados de D-1 por arquivo. [CONFIRMADO: FAQ SISREG] |
| RNDS | A integração já é feita pelo próprio Ministério. A secretaria não precisa fazer nada. [CONFIRMADO: https://portal.conasems.org.br/orientacoes-tecnicas/noticias/6362_conasems-em-foco-saude-digital-orientacoes-sobre-sistemas-de-informacao-para-a-regulacao-assistencial-no-contexto-do-pmae] |
| Custo / credenciamento | Gratuito. Fechado para centrais novas. |
| Módulos do Rota Saúde | **Regulação** (convivência ou importação), **Agendamento** (encaminhamento interno vs. vaga regulada), Procedimentos (BPA do executante). |

### 2.6 Regulação: e-SUS Regulação (substituto federal)
- Desenvolvido pelo MS para substituir o SISREG III. Faz gestão de recursos, escalas e "administração e priorização de agendamentos". Login gov.br. Integra com **RNDS** e **CNES**. Módulos Consultas, Exames e Laboratório. Perfis: administrador, solicitante, regulador, executante e coordenador. [CONFIRMADO: https://wiki.saude.gov.br/e-SUSREGULACAO/index.php/P%C3%A1gina_principal]
- Por enquanto tem **só o módulo ambulatorial**. [CONFIRMADO: https://www.gov.br/saude/pt-br/composicao/saes/drac/regulacao-do-acesso/sistemas-de-informacao]
- **Adesão:** chamado em webatendimento.saude.gov.br, com ofício do gestor, **CNES válido da Central de Regulação** e dados do operador. Produção em https://regulacao.saude.gov.br/ e treinamento em https://regulacao-treina.saude.gov.br/. [CONFIRMADO (via busca): resultado agregando gov.br e wiki ; ambientes também em https://www.gov.br/saude/pt-br/composicao/saes/drac/regulacao-do-acesso/sistemas-de-informacao]
- Também serve a quem tem serviço de regulação sem sistema ou "deseja substituir sistemas próprios ou terceiros". [CONFIRMADO (via busca): Portaria GM/MS nº 6.656/2025 e nota CONASEMS]
- [INFERÊNCIA] Não achei API pública de escrita (webservice para receber solicitações de sistema externo). Ou seja: **é o concorrente gratuito do módulo de Regulação do Rota Saúde**, e não um parceiro de integração.

### 2.7 e-SUS Captação de Filas
- Módulo simplificado para quem não tem sistema de regulação informatizado. Junta as filas de espera do país em https://fila-regulacao.saude.gov.br/. Envio diário de preferência, mensal aceito (até o 5º dia útil). [CONFIRMADO: https://www.gov.br/saude/pt-br/composicao/saes/drac/regulacao-do-acesso/sistemas-de-informacao ; Portaria 6.656/2025 via https://www.conass.org.br/conass-informa-n-34-2025-publicada-a-portaria-gm-n-6-656-que-estabelece-a-obrigatoriedade-e-periodicidade-de-envio-de-dados-de-regulacao-assistencial-no-ambito-do-sistema-unico-de-saude/]

### 2.8 Obrigação de enviar dados de regulação à RNDS (MIRA)
- A **Portaria Conjunta SAES/SEIDIGI nº 3, de 18/04/2023**, institui o **MIRA** (Modelo de Informação da Regulação Assistencial), padrão nacional dos dados de regulação e conjunto mínimo para envio à RNDS. [CONFIRMADO: https://bvs.saude.gov.br/bvs/saudelegis/saes/2023/poc0003_20_04_2023.html (via busca) ; https://www.gov.br/saude/pt-br/composicao/saes/drac/regulacao-do-acesso/sistemas-de-informacao]
- A **Portaria GM/MS nº 6.656, de 07/03/2025**, estabelece que "todos os registros de solicitação de procedimentos ou encaminhamento a serviços de atenção especializada devem ser enviados para a RNDS" (art. 2º).
  - Vale para estados, municípios, DF e prestadores.
  - SISREG e e-SUS Regulação enviam de forma automática.
  - **Sistemas próprios ou de terceiros enviam diariamente, com dados até o dia anterior.**
  - Prazos ficam num plano operativo tripartite.
  - Sanção: "impedimento para a adesão a programas de cirurgias eletivas".

  [CONFIRMADO: https://www.conass.org.br/conass-informa-n-34-2025-publicada-a-portaria-gm-n-6-656-que-estabelece-a-obrigatoriedade-e-periodicidade-de-envio-de-dados-de-regulacao-assistencial-no-ambito-do-sistema-unico-de-saude/ ; https://www.gov.br/saude/pt-br/composicao/saes/drac/regulacao-do-acesso/compartilhamento-de-dados-com-a-rnds/compartilhamento-de-dados-com-a-rnds]
- **Perfis FHIR R4 publicados no Simplifier:** `BRRegulacaoAssistencial` (perfil de Composition), `BRRequisicaoRegulacaoAssistencial` (perfil de ServiceRequest) e o ValueSet `BRModalidadeAssistencialMIRA`. [CONFIRMADO (via busca): https://simplifier.net/redenacionaldedadosemsaude/brregulacaoassistencial ; https://simplifier.net/redenacionaldedadosemsaude/brrequisicaoregulacaoassistencial]
- No PMAE, a "fila de espera de forma individualizada" é **requisito para receber recursos**. [CONFIRMADO: nota CONASEMS acima]

### 2.9 Sistemas estaduais de regulação

| UF | Sistema | Achado |
|---|---|---|
| SP | **CROSS** (Central de Regulação de Ofertas de Serviços de Saúde) | Criada pelo Decreto estadual nº 56.061/2010. A lei estadual criou a **CROSS-U**, que deve interligar os sistemas municipais e pode fazer convênios com municípios para integração. [CONFIRMADO (via busca): https://www.al.sp.gov.br/repositorio/legislacao/lei/2018/lei-16657-12.01.2018.html ; https://www.al.sp.gov.br/noticia/?id=388365] [INFERÊNCIA: não achei especificação técnica pública de integração] |
| PR | **CARE Paraná** | Sistema estadual online com regulação ambulatorial, de internação e eletiva, **faturamento AIH e APAC** e SAMU. O **Sistema MV** legado fica só para consulta. Usuários: gestores, **municípios e consórcios intermunicipais**, nos perfis solicitante, executante, regulador e médico regulador. Acesso pela Central de Segurança do portal GSUS, com autorização da Regional de Saúde. [CONFIRMADO: https://www.saude.pr.gov.br/Pagina/Sistema-Estadual-de-Regulacao] [INFERÊNCIA: as cidades atuais do Rota Saúde (Curitiba, Maringá) referenciam MAC intermunicipal via CARE Paraná e consórcios. Não achei API pública] |
| SC | **SISREG III** | A SES-SC publica manuais de SISREG para solicitante, executante e administrador; o COSEMS-SC relata dificuldades com o sistema. [CONFIRMADO (via busca): https://saude.sc.gov.br/index.php/pt/regulacao/manuais/manual-sisreg-executante-ambulatorial-28-06-2016/download ; https://www.cosemssc.org.br/dificuldades-no-sistema-regulacao-sisreg/] |
| MT | **SISREG III** | Plano de implantação e manuais da SES-MT. [CONFIRMADO (via busca): https://www.saude.mt.gov.br/unidade/superintendencia-de-regulacao-controle-e-avaliacao/139/manuais-sisreg-3] |
| MG | (não confirmado) | [INFERÊNCIA] Historicamente usou o "SUSfácilMG" para regulação. Não confirmei a situação atual. A SES-MG regulamenta o envio de produção ao SIA via consórcios (Resolução SES/MG nº 5.819/2017, citada em busca). |

- **Base normativa da regulação:** a Portaria nº 1.559, de 01/08/2008, institui a Política Nacional de Regulação do SUS, com três dimensões (sistemas, atenção e acesso) e complexos reguladores com centrais de consultas e exames, de internações e de urgências. [CONFIRMADO: https://bvsms.saude.gov.br/bvs/saudelegis/gm/2008/prt1559_01_08_2008.html (via busca)]
- **Responsabilidade:** nas referências intermunicipais (parte da média e quase toda a alta complexidade), a regulação historicamente cabe ao **gestor estadual**, com operação pactuada na CIB. [CONFIRMADO: https://www.conass.org.br/bibliotecav3/pdfs/colecao2011/livro_4.pdf]

**Convivência e dupla digitação** [INFERÊNCIA, porque nenhuma fonte oficial descreve isso como prática]:
- Não há escrita via API no SISREG, no e-SUS Regulação nem no CARE Paraná, pelo que encontrei.
- Por isso, um sistema municipal convive com eles de três jeitos possíveis:
  - (a) **dupla digitação**: a UBS registra no sistema municipal e alguém transcreve no estadual/federal;
  - (b) **pré-regulação municipal**: o município regula a sua cota e só o que sai do município vai ao sistema estadual;
  - (c) **leitura de retorno** pela API de leitura do SISREG, para atualizar o status no sistema municipal.
- O COSEMS-SC relatar "dificuldades no sistema de regulação SISREG" é indício de atrito operacional, não prova de dupla digitação.

### 2.10 PPI, programação regional e consórcios
- A **PPI** (Programação Pactuada e Integrada) estabelece e quantifica as ações para a população de um território e direciona os recursos financeiros entre gestores. Na prática, define **quanto cada município "compra" de referência em outro**, ou seja, as cotas. Continua em uso. Por exemplo, a "Nova PPI Capixaba 2024-2025" foi aprovada pela Resolução CIB-ES nº 193/2024. [CONFIRMADO: https://bvsms.saude.gov.br/bvs/publicacoes/DiretrizesProgPactuadaIntegAssistSaude.pdf (via busca) ; https://saude.es.gov.br/Media/CIB/Resolu%C3%A7%C3%A3o%20CIB%20ES%20N%C2%BA193-%202024%20-%20Aprovar%20a%20PPI%20denominada%20Nova%20PPI%20Capixaba%202024%20-%202025.pdf]
- A **PGASS** e o **PRI** (Planejamento Regional Integrado) aparecem como evolução regional da programação. A atenção ambulatorial especializada se organiza regionalmente pelo PRI. O MS publicou em 2025 um módulo de "Programação da Atenção Especializada". [CONFIRMADO (via busca): https://bvsms.saude.gov.br/bvs/publicacoes/modulo2_programacao_atencao_especializada_sus.pdf] [INFERÊNCIA: não achei portaria que revogue a PPI. As duas convivem e o formato varia por estado (MG e TO têm sistemas próprios de PPI: http://ppiassistencial.saude.mg.gov.br/ , https://sistemas.saude.to.gov.br/sisppi/)]
- **Consórcios intermunicipais de saúde:** regidos pela Lei nº 11.107/2005 (consórcios públicos). No PR, **24 consórcios reúnem 96,7% dos municípios** e mantêm Ambulatórios Médicos de Especialidades (programa QualiCIS). O acesso deve ser agendado pelo sistema de regulação. [CONFIRMADO: https://www.saude.pr.gov.br/Pagina/QualiCIS (via busca) ; https://www.conass.org.br/guiainformacao/consorcio-publico/] Em MG, existe resolução específica sobre como os consórcios alimentam o SIA. [CONFIRMADO (via busca): https://www.saude.mg.gov.br/consorcios/]
- [INFERÊNCIA] Para o produto, a cota tem três origens que o módulo precisa modelar com origem e vigência: (1) oferta própria do município; (2) cota PPI ou programação regional em prestador de outro município ou do estado; (3) compra via consórcio, por contrato e com valor que pode passar da tabela SUS.

---

## 3. Recomendação em camadas

> Princípio: o Rota Saúde **não substitui** os sistemas oficiais de faturamento (SIA, SIH), que são do gestor. Ele **gera insumos** para eles e, cada vez mais, **envia à RNDS**. Na regulação, ele **pode ser** o sistema municipal, mas assume a obrigação de mandar o MIRA à RNDS todo dia.

### Camada 1: Procedimentos e faturamento ambulatorial básico (MVP do ciclo)
- **Importador SIGTAP mensal.** Baixar ZIP por competência, guardar a versão (`vAAMMDDhhmm`), carregar procedimento, CBO, CID, instrumento de registro, idade, sexo, complexidade, financiamento e compatibilidades. Tabela única e compartilhada entre cidades: é dado público, não de cidadão. [INFERÊNCIA: o banco por cidade não precisa replicar o SIGTAP, basta referenciar a competência]
- **Registro do procedimento realizado**, ligado ao atendimento ou desfecho que já existe. Valida CBO do profissional × procedimento, idade e sexo do paciente, CID exigido e instrumento.
- **Exportar BPA-I e BPA-C** no layout de interface texto do BPA Magnético, para o faturista importar. Usar CPF como identificador (regra 01/2025).
- Painel "produção do mês por estabelecimento × instrumento × financiamento".
- **Sem credenciamento e sem dependência externa**, além de acompanhar layouts e o SIGTAP.

### Camada 2: Regulação municipal
- Solicitação (UBS) com procedimento SIGTAP, CID, justificativa e classificação de risco → fila regulada → autorização, devolução ou negativa pelo regulador → agendamento em **cota** (própria, PPI ou consórcio) → confirmação da execução → desfecho e BPA.
- O fluxo de "encaminhado" que já existe vira solicitação regulada.
- **Envio diário à RNDS no MIRA** (perfis FHIR `BRRegulacaoAssistencial` e `BRRequisicaoRegulacaoAssistencial`), com certificado e-CNPJ. Sem isso, o Rota Saúde só pode ser **pré-regulação/gestão interna**, e a cidade continua registrando no SISREG, no e-SUS Regulação ou no sistema estadual. Isso tem de ser dito com clareza à prefeitura.
- **Modo convivência** enquanto não houver RNDS: fila interna, exportação de lista para digitar no sistema oficial, campo para o número da solicitação externa (SISREG, CARE etc.) e conciliação manual.

### Camada 3: Alta complexidade e programas federais
- **APAC:** laudo, controle de faixa numérica (com quinto dígito por programa: 7 = OCI etc.), validade em competências, secundários compatíveis, motivo de saída, exportação no layout de interface da APAC.
- **OCI do Agora Tem Especialistas:** fila nominal, prazo de conclusão e alerta de OCI que vai estourar a validade, que se perdida volta a BPA-I.
- **FPO e teto:** importar ou digitar a programação e mostrar produção × programado × teto MAC/FAEC.
- **Conciliação:** produção apresentada × aprovada/glosada (retorno do SIA). [INFERÊNCIA: precisa descobrir em que formato o município recebe o retorno do processamento]

### Camada 4: RNDS para produção (RAC/CMD)
- Mandar o atendimento como RAC (o Rota Saúde já registra o contato) ou CMD. Isso prepara a saída do BPA quando o Ministério fixar o prazo de transição.

### Fora do escopo do ciclo: linha hospitalar
- AIH via interface do SISAIH01 e Sumário de Alta na RNDS. Produto separado.

### Estratégia de teste (para quando virar spec)
- Fixtures com recortes reais do SIGTAP de uma competência (alguns procedimentos de cada instrumento), com specs de importação e versionamento.
- Specs de validação: CBO incompatível, idade ou sexo fora da faixa, CID ausente quando exigido, instrumento errado.
- Teste "golden file" do TXT de BPA: comparar byte a byte com um arquivo de referência (largura fixa, CR+LF, campo de controle). Validar uma vez de verdade **importando no BPA Magnético** numa VM Windows, porque é o único oráculo confiável.
- RNDS: só em homologação, com payload validado contra os perfis do Simplifier (validador FHIR) antes de pedir credencial.

---

## 4. Riscos

1. **Layouts mudam sem aviso amplo.** O BPA foi atualizado em 09-Jul-2026 e o RAAS 3.0 tornou-se obrigatório em ago/2026. Um BPA gerado com layout velho é rejeitado no BPA Mag. É preciso alguém acompanhando as Notas Técnicas e o "Leia-me" a cada competência. [CONFIRMADO: páginas de versão do SIA]
2. **Distribuição por FTP e Windows.** O FTP do DATASUS não completou a transferência daqui. A automação de download do SIGTAP e dos layouts pode precisar de rota alternativa: espelho, download manual ou outro IP. [CONFIRMADO: teste local]
3. **Obrigação RNDS na regulação** (Portaria 6.656/2025): se o Rota Saúde for o sistema de regulação e não enviar o MIRA todo dia, a cidade fica exposta a sanção (impedimento em programas de eletivas). É risco contratual e reputacional.
4. **Concorrente gratuito federal** (e-SUS Regulação) e **sistemas estaduais obrigatórios** para a referência intermunicipal (CARE Paraná, CROSS). O valor do Rota Saúde tende a ficar na regulação **intramunicipal**, na experiência do cidadão (web) e na conciliação, não em substituir o estadual. [INFERÊNCIA]
5. **Transição para CMD/RAC sem data.** Investir pesado em BPA pode virar dívida, e esperar a RNDS deixa a prefeitura sem faturamento. Por isso a recomendação é BPA simples (camada 1) e RNDS como norte.
6. **Identificação do paciente:** CPF obrigatório no Agora Tem Especialistas e CPF ou CNS (nunca os dois) no SIA. O cadastro do cidadão no Rota Saúde já usa CPF declarado, mas o CNS do profissional continua necessário no BPA-I. [INFERÊNCIA a partir do layout 2020]
7. **Responsabilidade por glosa:** se o arquivo gerado pelo Rota Saúde levar à rejeição da produção, o prejuízo financeiro é da prefeitura. Exige termo de responsabilidade e validação prévia rigorosa.
8. **LGPD:** o BPA-I leva nome, endereço, telefone e e-mail do paciente. O arquivo exportado precisa de trilha de auditoria e de controle de quem baixa.
9. **Fonte secundária:** parte dos detalhes do Agora Tem Especialistas (dígitos de APAC por modalidade, cronograma 2026) veio do COSEMS-SP e de blogs. Antes de implementar, confirmar nas portarias SAES.

---

## 5. Perguntas em aberto

1. Qual é o **layout vigente do BPA** (PDF de 09-Jul-2026)? O que mudou em relação a 2020: campo de CPF do paciente, "pessoa sem CPF", tamanho de campos?
2. Qual é a **lista exata de arquivos e layouts** do ZIP do SIGTAP 202609, e o campo de valor mudou mesmo de tamanho nesta competência?
3. A RNDS **aceita credencial de software house** que envia em nome de várias secretarias, ou cada município precisa da própria credencial e do próprio e-CNPJ? Como fica isso no banco por cidade?
4. O plano operativo tripartite da Portaria 6.656/2025 já definiu **prazos por UF** para sistemas próprios? Curitiba e Maringá já enviam MIRA hoje, e por qual sistema?
5. Em Curitiba e Maringá, quem regula a MAC **intramunicipal**: a central municipal em SISREG, sistema próprio ou CARE Paraná? E a **intermunicipal** (CARE Paraná ou consórcio)?
6. O **CARE Paraná** tem API ou importação de arquivo para solicitações vindas de sistema municipal?
7. O **e-SUS Regulação** tem ou terá API de escrita para receber solicitações de sistemas externos?
8. Em que formato o município recebe o **retorno do processamento do SIA** (aprovado/glosado), para a conciliação?
9. O que diz a Portaria SAS/MS nº 1.011/2014 (laudo e certificação digital) para **emissão eletrônica de laudo de APAC** por sistema de terceiro?
10. Qual é o **cronograma 2026 oficial** da SAES (fonte primária) para remessas SIA/SIH?
11. Quando sai a especificação técnica do **CMD** (Portaria 11.952/2026) e existe previsão de **desligamento do BPA**?
12. Qual **valor de cota** a prefeitura quer controlar no produto: só quantidade física ou também valor financeiro (tabela SUS × contrato de consórcio)?

---

## 6. Fontes

**Oficiais (MS, DATASUS, BVS)**
- https://bvsms.saude.gov.br/bvs/saudelegis/gm/2023/prt1604_20_10_2023.html (PNAES)
- https://bvsms.saude.gov.br/bvs/saudelegis/gm/2017/prt2436_22_09_2017.html (PNAB)
- https://bvsms.saude.gov.br/bvs/saudelegis/gm/2017/prt3992_28_12_2017.html (blocos de financiamento)
- https://bvsms.saude.gov.br/bvs/saudelegis/gm/2020/prt0828_24_04_2020.html (Bloco de Manutenção, via busca)
- https://bvsms.saude.gov.br/bvs/saudelegis/gm/2008/prt1559_01_08_2008.html (Política Nacional de Regulação, via busca)
- https://bvsms.saude.gov.br/bvs/saudelegis/gm/2025/prt7495_05_08_2025.html (SUS Digital / Agora Tem Especialistas)
- https://bvsms.saude.gov.br/bvs/saudelegis/gm/2025/prt6656_10_03_2025.html (envio de dados de regulação, via busca)
- https://www.gov.br/saude/pt-br/composicao/saes/drac/regulacao-do-acesso/sistemas-de-informacao
- https://www.gov.br/saude/pt-br/composicao/saes/drac/regulacao-do-acesso/compartilhamento-de-dados-com-a-rnds/compartilhamento-de-dados-com-a-rnds
- https://www.gov.br/saude/pt-br/centrais-de-conteudo/publicacoes/guias-e-manuais/2024/manual-pmae-registro-da-producao-controle-e-avaliacao.pdf
- https://webatendimento.saude.gov.br/faq/sisreg
- https://wiki.saude.gov.br/e-SUSREGULACAO/index.php/P%C3%A1gina_principal
- https://wiki.saude.gov.br/sigtap/index.php/Gerais
- https://wiki.saude.gov.br/sigtap/index.php/Grupo
- https://wiki.datasus.gov.br/sigtap/index.php/Download
- http://sigtap.datasus.gov.br/tabela-unificada/app/download.jsp
- https://wiki.saude.gov.br/sia/index.php/P%C3%A1gina_principal
- http://sia.datasus.gov.br/principal/index.php
- https://sia.datasus.gov.br/versao/listar_ftp_bpa.php · listar_ftp_apac.php · listar_ftp_sia.php · listar_ftp_fpo.php · listar_ftp_raas.php
- https://sisab.saude.gov.br/resource/file/nota_tecnica_arquivo_bpa_i_241025.pdf
- https://rnds-guia.saude.gov.br/docs/introducao/
- https://simplifier.net/redenacionaldedadosemsaude/brregulacaoassistencial · https://simplifier.net/redenacionaldedadosemsaude/brrequisicaoregulacaoassistencial
- http://w3.datasus.gov.br/sihd/Manuais/LAYOUT_SISAIH01_201010.pdf (via busca)
- https://www.lexml.gov.br/urn/urn:lex:br:federal:lei:2025-10-07;15233 · https://www.gov.br/planalto/pt-br/acompanhe-o-planalto/noticias/2025/10/lula-sanciona-lei-do-agora-tem-especialistas-e-regulamenta-lei-de-pesquisa-clinica-para-fortalecer-a-saude-publica

**Estaduais e entidades de gestores (CONASS, CONASEMS, COSEMS)**
- https://www.conass.org.br/bibliotecav3/pdfs/colecao2011/livro_4.pdf
- https://www.conass.org.br/conass-informa-n-34-2025-publicada-a-portaria-gm-n-6-656-que-estabelece-a-obrigatoriedade-e-periodicidade-de-envio-de-dados-de-regulacao-assistencial-no-ambito-do-sistema-unico-de-saude/
- https://www.conass.org.br/altera-a-portaria-de-consolidacao-no-1-de-28-de-setembro-de-2017-para-instituir-o-modelo-de-informacao-do-conjunto-minimo-de-dados-cmd/
- https://www.conass.org.br/que-estabelece-no-ambito-do-programa-agora-tem-especialistas-criterios-operacionais-referentes-ao-componente-ambulatorial-e-inclui-atributo-complementar-na-tabela-de-procedimentos-medicamentos-or/
- https://portal.conasems.org.br/orientacoes-tecnicas/noticias/6362_conasems-em-foco-saude-digital-orientacoes-sobre-sistemas-de-informacao-para-a-regulacao-assistencial-no-contexto-do-pmae
- https://www.cosemssp.org.br/wp-content/uploads/2026/05/Katia-Vital-Navarro-Watanabe.pdf
- https://www.saude.pr.gov.br/Pagina/Sistema-Estadual-de-Regulacao
- https://www.saude.pr.gov.br/Pagina/QualiCIS
- https://saude.sc.gov.br/index.php/pt/regulacao/manuais/manual-sisreg-executante-ambulatorial-28-06-2016/download
- https://www.saude.mt.gov.br/storage/old/files/plano-de-implantacao-sisreg-iii-[178-250811-SES-MT].pdf
- https://www.al.sp.gov.br/repositorio/legislacao/lei/2018/lei-16657-12.01.2018.html

**Secundárias (usadas só como pista, marcadas no texto)**
- https://www.fehosp.com.br/files/circulares/3c765d77164dc09521d225a9d2386844.pdf (layout BPA reproduzido, 2020)
- https://rfsaldanha.github.io/sis/sia.html
- https://p2saude.com.br/ministerio-da-saude-divulga-cronograma-2026-para-sistemas-cnes-sia-e-sih/
- https://github.com/RenatoKR/SIGTAP · https://github.com/RicardoHerrero/SIGTAP-dataSUS
