# Módulo 21b: analisadores de laboratório clínico (interface com aparelhos)

Pesquisa para o módulo 21b, o "laboratório municipal" do Rota Saúde. O módulo entrega a parte do LIS que os editais cobram, mas **sem interface com aparelhos nesta versão**. O objetivo aqui é mapear quais analisadores os laboratórios municipais e os LACENs usam, como esses aparelhos se comunicam e quem costuma fornecer a interface, para decidirmos depois se vale integrar e de que jeito.

Data da pesquisa: 2026-10-15. Legenda:
- **[CONFIRMADO: url]**: li na própria fonte (página ou PDF com o texto extraído).
- **[INFERÊNCIA]**: dedução minha ou conhecimento que não verifiquei nesta sessão.
- **[INFERÊNCIA, trecho de buscador: url]**: vi só o resumo do buscador, não o documento inteiro.

---

## Resumo executivo

1. **O modelo dominante é comodato do aparelho mais compra de reagente**, com o preço do equipamento embutido no teste. Às vezes aparece como "locação paga por teste reportado". O mesmo edital costuma juntar bioquímica, hematologia, imuno/hormônios, urinálise, coagulação e gasometria em lotes. Exemplos lidos: Sorriso/MT (TR de 15/09/2025), Joinville/SC (PE 124/2025, hematologia e coagulação; PE 012/2024, imunologia com o Abbott Alinity i), Parnamirim/RN (TR de 01/02/2024), Prefeitura de São Paulo (hemograma em comodato para laboratórios municipais) e Hospital Municipal de Salvador (gasometria em comodato).
2. **Quem paga a interface quase sempre é o fornecedor do reagente**, e é ele quem entrega o "interfaceamento bidirecional". Muitas vezes entrega também o middleware, o servidor e as estações de trabalho, e em alguns casos **até o próprio sistema de gestão laboratorial** (Sorriso exige um "Sistema de Gerenciamento da Produção e Liberação Laboratorial" com banco Oracle licenciado; o edital de João Dourado/BA de 2026 põe o LIS dentro do objeto do comodato). **Isso torna o fornecedor de reagente um concorrente direto do LIS do Rota Saúde no edital**, e é o principal achado estratégico desta pesquisa.
3. **Protocolos.** Os grandes imunoensaios e analisadores de bioquímica e hematologia (Beckman, Abbott, Roche, Siemens, Sysmex) falam **ASTM E1381/E1394**, hoje chamados CLSI LIS01/LIS02. A camada física pode ser serial RS-232 ou TCP/IP. Para receber pedidos, o aparelho manda um *host query* (registro `Q`). A Mindray e os POCT mais novos tendem a usar **HL7 v2** (ORU/OUL) sobre TCP/MLLP. Os POCT de HbA1c e gasometria usam **POCT1-A2**, além de ASTM ou HL7, quase sempre por meio de um *data manager* do fabricante. Poucos manuais de interface são públicos: Beckman DxI/Access e Roche cobas liat publicam; Sysmex, Abbott, Mindray e Labtest/Wiener entregam o manual ao cliente ou ao fornecedor do LIS.
4. **Na prática, o LIS fala com um middleware ou interfaceador, e esse middleware fala com os aparelhos.** Os middlewares globais são o Data Innovations Instrument Manager (mais de 1.200 drivers), o Siemens Atellica Data Manager (mais de 700 drivers, inclui aparelhos Abbott, Roche e Sysmex), o Roche cobas infinity e o Sysmex WAM/Caresphere. No Brasil, o próprio LIS ou um interfaceador contratado à parte cumpre esse papel (o Hospital Municipal de Salvador licitou um "sistema de interfaceamento" separado para ligar os aparelhos ao MV Soul).
5. **Recomendação: opção B.** O 21b segue sem integração. O contrato do Rota Saúde passa a exigir que o fornecedor do reagente entregue a interface, e o Rota Saúde expõe **uma única porta de entrada documentada** (HL7 v2 OUL/ORU ou um JSON próprio, mais o host query) para o middleware do fornecedor. ASTM direto (opção C) só faz sentido mais adiante e com um agente local, porque um SaaS não alcança uma porta serial dentro do laboratório.

---

## 1. Analisadores que aparecem em editais, por setor

### 1.1 Exemplos de editais, atas e pareceres

| Município ou órgão | Instrumento | Setores | O que foi lido | Fonte |
|---|---|---|---|---|
| **Sorriso/MT** | TR de comodato e reagentes, 15/09/2025 | hematologia, bioquímica, imunologia, hormônios, uroanálise, coagulação, gasometria POC | Exige "Interfaceamento obrigatoriamente bidirecional", "computador servidor próprio para gerenciamento de sua interface" e estações de validação, e põe as alterações de rede por conta da vencedora. Exige ainda um sistema de gerenciamento da produção e liberação laboratorial, com licença Oracle, rastreabilidade da amostra, CQ, TAT e reagentes com código de barras da mesma marca do equipamento. | [CONFIRMADO: https://site.sorriso.mt.gov.br/dl/638108] |
| **Joinville/SC**, Laboratório Municipal | PE 124/2025, locação paga por teste reportado | hematologia e coagulação | Validação prática antes do contrato: 300 hemogramas com reticulócitos, 100 TAP, 100 TTPA etc. Empresa convocada: J R Ehlke e Cia Ltda. | [CONFIRMADO: https://www.joinville.sc.gov.br/public/edital/anexo/a7dbc7abd6f7f351a8138c3464d8efeb.pdf] |
| **Joinville/SC** | PE 012/2024, parecer técnico de 09/05/2024 | imunologia (infecciosas, hormônios, vitaminas, marcadores tumorais) | Aparelho avaliado: Abbott Alinity i. O checklist exige "Interface bidirecional", reteste automático com "envio automático de resultados para a interface" e "identificar tubos por códigos de barras no padrão de impressão do sistema de interfaceamento". A repetição no laboratório de apoio usou Alinity i, Architect, Beckman DxI e Roche cobas e 411. | [CONFIRMADO: https://joinville.sc.gov.br/public/edital/anexo/48292724c035182a42b1deb37ae0b7dd.pdf] |
| **Parnamirim/RN**, Laboratório Central | TR de 01/02/2024 | hematologia, imunologia, hormônios, bioquímica, urinálise, gasometria, coagulação, marcadores cardíacos portáteis | "equipamentos automatizados e computadores em SISTEMA DE COMODATO". O TR não traz cláusula explícita de interface; no texto extraído não há "interfac". | [CONFIRMADO: https://antigo.parnamirim.rn.gov.br/pdf/pregao_licitacao/anexo_licitacao_16_02_2024_16_10_14_1903.pdf] |
| **Prefeitura de São Paulo**, SMS | TR de hematologia (2020/21, fora da janela de 3 anos, mantido como referência de modelo) | hemograma e reticulócitos | Comodato por dispensa para os laboratórios Sudeste, Lapa e outros. A empresa "deverá permitir o interfaceamento bidirecional" e fornecer computador, impressora de código de barras e impressora de laudos. | [CONFIRMADO: https://drive.prefeitura.sp.gov.br/cidade/secretarias/upload/controladoria_geral/TERMOREFERENCIA_SMS_HEMATOLOGIA_publ_24_12_2020.pdf] |
| **Hospital Municipal de Salvador** (Santa Casa da Bahia, gestora em contrato de gestão) | TR 029/2023, gasometria em comodato | gasometria | Leitor de código de barras "com recurso de interfaceamento bidirecional". Os custos da interface são "acordados na negociação". Exige CQ com Levey-Jennings. | [CONFIRMADO: https://www.santacasaba.org.br/conteudo/dow/002/dowarq/arq/000454.pdf] |
| **Hospital Municipal de Salvador** | TR 026/2024, **sistema de interfaceamento** contratado à parte | bioquímica, hematologia, coagulação, gasometria | Interfaceador bidirecional entre os aparelhos e o LIS MV Soul, em nuvem privada no Brasil. Prazos: 30 dias para os aparelhos atuais e 20 dias para cada aparelho novo depois de receber a especificação do fabricante. O desenvolvimento de drivers corre por conta da contratada. | [CONFIRMADO: https://www.santacasaba.org.br/conteudo/dow/002/dowarq/arq/000619.pdf] |
| **João Dourado/BA** | PE JDPE014/2026, R$ 3.256.553,00 | reagentes do laboratório municipal | O objeto inclui "cessão de equipamentos em regime de comodato e sistema informatizado de gestão laboratorial (LIS)". | [CONFIRMADO: https://alertalicitacao.com.br/!licitacao/PNCP-13891510000148-1-000103-2026] (resumo do agregador; o edital completo não foi lido) |
| **Balneário Camboriú/SC** | PE 00210/2023, comodato com reagentes | laboratório municipal | Ata citaria Sysmex XN-350 e Mindray BC-5150. | [INFERÊNCIA, trecho de buscador: https://www.bc.sc.gov.br/arquivos/licitacao/RP8NS2EF.pdf] (PDF não abriu) |
| **Pouso Alegre/MG** | PE 2025, locação de analisador de bioquímica Wiener | bioquímica | Marca justificada por padronização com o equipamento Wiener que o laboratório já tinha. | [INFERÊNCIA, trecho de buscador: https://pousoalegre.mg.gov.br/licitacoes/ff544f32f433e826b4327e2719aa3c75.pdf] (403) |
| **Lages/SC** | DFD 31/2024, compra de analisador de bioquímica | bioquímica | 240 testes/h fotométricos, 400 com ISE, "sistema aberto". O perfil bate com a classe Mindray BS-240, mas o documento não cita marca. | [INFERÊNCIA, trecho de buscador: https://licitacoes.lages.sc.gov.br/assets/licitacao/a74b4250b5efe6b9bffb24e75a5bcddc.pdf] |
| **Recife/PE** | PE 95/2026, hemostasia | coagulação | Cinco coagulômetros automatizados em comodato para três hospitais municipais. | [INFERÊNCIA, trecho de buscador: https://alertalicitacao.com.br/!licitacao/CEPE-PE-5-373135] |
| Outros de 2026 (São Caetano/PE, Salgueiro/PE, Camaragibe/PE, Sidrolândia/MS, Fortaleza/CE em hormônios) | pregões de comodato | bioquímica, hematologia, imuno-hormônio, hemostasia, uroanálise | Só objeto e valor; marcas e cláusulas de interface não foram lidas. | [INFERÊNCIA, trecho de buscador: https://alertalicitacao.com.br/!licitacao/BNC-2603454-751e-0032026 ; https://alertalicitacao.com.br/!licitacao/PNCP-11361243000171-1-000196-2026] |
| **LACEN-RO** | PE 90088/2024/SUPEL, biologia molecular | PCR multiplex | Kits com "todo sistema/solução de automação laboratorial" em comodato. O parecer trata kit, hardware e software como "cadeia analítica indivisível". | [INFERÊNCIA, trecho de buscador: https://rondonia.ro.gov.br/wp-content/uploads/2025/05/SEI_0060507675_Analise_4.pdf] |
| **LACEN-MG** (Funed) | pregão de 11/12/2023, LIS Korus (Pixeon) | todos | Em 13/09/2024: "A fase atual é de interfaciamento dos equipamentos e de integração do GAL". O LIS importa cadastros do GAL e exporta resultados para ele. | [CONFIRMADO: https://www.agenciaminas.mg.gov.br/noticia/novo-sistema-laboratorial-agiliza-entrega-de-exames-no-laboratorio-de-saude-publica/] |

### 1.2 Famílias de analisadores por setor

O que eu vi nas fontes aparece com o link. O resto é conhecimento de mercado, sem verificação nesta sessão.

| Setor | Famílias recorrentes no setor público brasileiro | Observações |
|---|---|---|
| **Bioquímica** | Roche cobas c (c 111/c 311/c 501/pure); Abbott Architect c / Alinity c; Siemens Atellica CH / Dimension; Beckman AU (480/680); **Mindray BS (120/200/240/380/480)**; **Wiener (CM 200/250)**; **Labtest Labmax (240/Plenno/Premium)**; **Bioclin (reagentes nacionais; analisadores parceiros)** [INFERÊNCIA] | Municípios pequenos e médios ficam com Mindray, Labtest e Wiener, a classe de 200 a 400 testes/h, muitas vezes "sistema aberto" (Lages). Os grandes, como Sorriso, pedem plataforma modular bioquímica + imuno com ISE e um único software [CONFIRMADO: https://site.sorriso.mt.gov.br/dl/638108]. |
| **Hematologia** | **Sysmex XN (XN-350/450/550/1000)** e XS/XP; **Mindray BC (5000/5150/5390/6000)**; Horiba Yumizen H500/H2500, ABX Pentra/Micros; Beckman DxH [INFERÊNCIA] | Os TRs pedem diferencial de 5 partes, 23 ou mais parâmetros, reticulócitos e preparador/corador de lâmina [CONFIRMADO: TR SMS-SP; TR Sorriso]. |
| **Imunologia e hormônios** | Abbott Architect i / **Alinity i**; Roche cobas e 411/e 601/e 801; Beckman Access 2 / DxI; Siemens Atellica IM / Centaur; Snibe Maglumi; DiaSorin Liaison; bioMérieux Vidas (marcadores e infecciosas) [Alinity, Architect, DxI e cobas e 411 CONFIRMADOS no parecer de Joinville; o resto é INFERÊNCIA] | Joinville paga por teste reportado. |
| **Urinálise** | Roche cobas u 411/u 601/u 701; Sysmex UC/UF/UN; Arkray Aution; Mindray UA/EU; leitores nacionais de tira [INFERÊNCIA] | Sorriso pede leitor de tiras, sedimentoscopia automatizada (100/h), conexão entre os dois e "interfaceamento bidirecional completo" [CONFIRMADO: https://site.sorriso.mt.gov.br/dl/638108]. |
| **Coagulação** | Stago STA (Compact Max/Satellite); Werfen ACL TOP / ACL Elite; Sysmex CS/CN; analisadores semiautomáticos nacionais [INFERÊNCIA] | Joinville coloca coagulação no mesmo pregão de hematologia (Lote 2: TAP, TTPA, fibrinogênio, D-dímero) [CONFIRMADO]. |
| **Gasometria** | Radiometer ABL (ABL90 FLEX, ABL800); Werfen GEM Premier; Siemens RAPIDPoint/RAPIDLab; Roche cobas b 121/b 221; Abbott i-STAT [INFERÊNCIA] | Quase sempre comodato com cartucho. Sorriso pede "Analisador de Gasometria, Point of Care" [CONFIRMADO]; Hospital Municipal de Salvador, TR 029/2023 [CONFIRMADO]. |
| **POCT** (HbA1c, testes rápidos) | Siemens DCA Vantage; Abbott Afinion; Quo-Lab; leitores de imunocromatografia (Eco F, Wama, Abbott/Alere); glicosímetros [INFERÊNCIA] | Não achei edital municipal de HbA1c POC em comodato. A Conitec analisou HbA1c POC com recomendação preliminar desfavorável por impacto orçamentário [INFERÊNCIA, trecho de buscador: https://www.gov.br/conitec/pt-br/midias/reuniao_conitec/2023/ata-da-119a-reuniao-ordinaria-da-conitec_produtos-e-procedimentos.pdf]. Os testes rápidos da APS (HIV, sífilis, hepatites) são do MS e, em geral, **não têm interface**: a leitura é visual [INFERÊNCIA]. |

### 1.3 O modelo comodato mais reagente: quem entrega o software

- **O padrão nos TRs é que o fornecedor do reagente paga tudo o que liga o aparelho ao LIS.** Isso inclui o interfaceamento bidirecional, o servidor da interface, as estações de validação, as impressoras de etiqueta e as mudanças de rede [CONFIRMADO: TR Sorriso; TR SMS-SP]. Há uma variante em que os custos ficam "acordados na negociação" [CONFIRMADO: TR 029/2023 HMS].
- **O fornecedor do reagente muitas vezes entrega o middleware dele**, o que é coerente com o portfólio: Roche com cobas infinity, Siemens com Atellica Data Manager, Sysmex com WAM e Caresphere. Também pode subcontratar um interfaceador nacional, ou o próprio fornecedor do LIS, para escrever o driver [INFERÊNCIA].
- **Há editais em que o fornecedor do reagente entrega o LIS inteiro.** Sorriso exige "sistema de gerenciamento de produção laboratorial", com liberação de resultados, soroteca, estoque e licença Oracle [CONFIRMADO]. João Dourado/BA põe o LIS no objeto do comodato [CONFIRMADO via agregador]. **Consequência para o Rota Saúde:** quando o município monta o edital assim, o LIS do 21b concorre com um LIS "de graça" embutido no preço do reagente. Daí a importância do argumento de que o Rota Saúde é o prontuário e a regulação da cidade, e que o laboratório é só um módulo dele [INFERÊNCIA].
- **Outra cláusula comum:** se o laboratório trocar de LIS, o fornecedor do equipamento refaz a interface [INFERÊNCIA, trecho de buscador sobre TR UFRGS/FAURGS]. Isso joga a nosso favor: o município pode exigir que o fornecedor do reagente conecte o aparelho ao Rota Saúde.

---

## 2. Protocolos de comunicação por família

### 2.1 Padrões

- **ASTM E1381** (camada baixa: enquadramento, checksum, ENQ/ACK/EOT) e **ASTM E1394** (registros H, P, O, R, C, Q, L). Os dois foram redesignados como **CLSI LIS1/LIS2**, hoje **LIS01-A2/LIS02-A2** [CONFIRMADO, só a menção ASTM: https://www.beckmancoulter.com/download/file/wsr-228138/C03112AE?type=pdf; a redesignação para CLSI veio de trecho de buscador da ANSI webstore, então é INFERÊNCIA].
- **HL7 v2**: ORU^R01 para resultado, OUL^R22 para resultado de laboratório, OML^O21/O33 para pedido, QBP/ORM para consulta. O transporte é TCP/IP com MLLP. Exemplo: o cobas liat usa **ORU^R30** e **MLLP**, e "always acts as the TCP/IP client" [CONFIRMADO: https://assets.cwp.roche.com/f/94122/x/e6e97fa41f/cobas-liat-system_hostinterfacemanual_hl7_sw3-4-3-5.pdf].
- **CLSI POCT1-A2**: o padrão de dispositivos point-of-care. No DCA Vantage, a Ethernet aceita ASTM ou POCT1-A2. O POCT1-A2 é bidirecional (bloqueio de teste e lista de operadores); o ASTM roda por Ethernet ou RS-232 [CONFIRMADO: https://www.siemens-healthineers.com/diabetes/diabetes/dca-vantage-analyzer]. O cobas liat também documenta POCT1-A [CONFIRMADO: manual liat].
- **IHE LAW** (Laboratory Analytical Workflow, perfil HL7 v2.5.1 para aparelho-middleware) e **IHE LTW** (Laboratory Testing Workflow). O cobas infinity cita o IHE LTW [CONFIRMADO: https://diagnostics.roche.com/content/dam/diagnostics/ch/de/cd-centralized-diagnostics/ch-en-cd-cobas-infinity-lab-solution-spezifikationen.pdf]. A adoção do LAW no Brasil não foi verificada [INFERÊNCIA].

### 2.2 Tabela por família

| Família | Protocolo | Física | Bidirecional | Manual de interface |
|---|---|---|---|---|
| **Beckman UniCel DxI / Access 2** | ASTM E1394-97 + E1381-95 | EIA-232 (DB-9, 9600 bps padrão, 8N1), com seção de rede RJ-45 | Sim, por host query: registro `Q` ("Request Information"), com o exemplo `Q|1|^Samp45||ALL...` | **Público**, "LIS Vendor Information", feito para ser entregue ao fornecedor do LIS [CONFIRMADO: https://www.beckmancoulter.com/download/file/wsr-228138/C03112AE?type=pdf] |
| **Roche cobas liat** (POC PCR) | HL7 v2 (ORU^R30) + POCT1-A | TCP/IP, MLLP, TLS | Sim | **Público** (Host Interface Manual) [CONFIRMADO: manual liat] |
| **Roche cobas c/e/u** (c 311, u 411...) | ASTM ("ASTM plus" da Roche); o c 311 em RS-232 | RS-232 e TCP/IP | Sim (host query) | Host Interface Manual distribuído ao cliente; há cópias não oficiais em Scribd e SlideShare [INFERÊNCIA, trecho de buscador] |
| **Roche cobas infinity** (middleware) | Para o LIS: HL7 v2.x/v3.x, XML nativo e SOAP. Para terceiros: ASTM e HL7 v2/v3 | TCP/IP | Order Query, resultados de paciente e resultados de CQ | Ficha pública [CONFIRMADO: https://diagnostics.roche.com/content/dam/diagnostics/ch/de/cd-centralized-diagnostics/ch-en-cd-cobas-infinity-lab-solution-spezifikationen.pdf] |
| **Abbott Architect / Alinity** | ASTM E1381/E1394 ("Standard Interface") [INFERÊNCIA] | RS-232 e TCP/IP [INFERÊNCIA] | Sim: Joinville confirma "Interface bidirecional: SIM" no Alinity i [CONFIRMADO] | Só ao cliente ou ao fornecedor do LIS [INFERÊNCIA] |
| **Sysmex XN** | ASTM E1394 [INFERÊNCIA] | TCP/IP e serial [INFERÊNCIA] | Sim (host query) [INFERÊNCIA] | Apêndice "Interface Protocol" no Administrator's Guide; site de terceiro deu 403 [INFERÊNCIA, trecho de buscador]. Houve recall de transmissão incorreta ao host nas versões 16 a 19 [INFERÊNCIA, trecho de buscador: https://medicaldevices.icij.org/events/aus-sysmex-xn-and-xn-l-series-all-sysmex-xn-analysers-versions-16-to-19] |
| **Mindray BS / BC** | **HL7 v2.3.1** sobre TCP. Há relato de desenvolvedor para BS-200/220/120/130 e a página do BC-3600 diz "integrated HL7 communication protocol" [INFERÊNCIA, trecho de buscador]. Alguns modelos também fazem ASTM [INFERÊNCIA] | TCP/IP | Sim (pedido do host) | Distribuído pelo representante; o "Patient Data Share Protocol Programmer's Guide" aparece em cópia não oficial [INFERÊNCIA] |
| **Horiba ABX / Yumizen** | ASTM E1394 [INFERÊNCIA] | RS-232 e TCP/IP | Sim | Ao cliente [INFERÊNCIA] |
| **Siemens Atellica / Dimension** | ASTM e HL7 [INFERÊNCIA] | TCP/IP | Sim | Ao cliente [INFERÊNCIA] |
| **Labtest Labmax / Wiener / bioquímica nacional** | ASTM ou protocolo próprio [INFERÊNCIA; não achei nada publicado] | RS-232 nos modelos antigos, TCP/IP nos novos | Varia: há modelos só unidirecionais [INFERÊNCIA] | Não público |
| **Radiometer ABL90 FLEX** | **HL7 v2.5** para HIS/LIS; middleware AQURE | TCP/IP | Busca automática de dados do paciente [INFERÊNCIA, trecho de buscador do IFU] | Ao cliente. Aviso de segurança de 2012: com software anterior à V2.7 MR4, depois de queda de conexão com 2 ou mais resultados na fila, "les résultats suivant seront transmis avec l'ID du premier patient" [CONFIRMADO: https://archive.ansm.sante.fr/content/download/42382/550490/version/1/file/mes-120711-Automategazom%C3%A9trieABL90FLEX-+Radiometer.pdf] |
| **Siemens DCA Vantage** (HbA1c POC) | ASTM ou POCT1-A2; data manager RAPIDComm | Ethernet e RS-232 | Sim no POCT1-A2 | Ficha pública [CONFIRMADO: https://www.siemens-healthineers.com/diabetes/diabetes/dca-vantage-analyzer] |

**Padrão geral** [INFERÊNCIA]: aparelhos grandes e antigos usam ASTM, em RS-232 ou TCP. Chineses e POC novos usam HL7 v2 sobre TCP. POCT usa POCT1-A2 por meio de um data manager. **Em todos, a documentação detalhada sai do fabricante para o cliente ou para o integrador**, e cada modelo e versão de firmware tem particularidades. Uma tese mexicana citada pelo buscador resume que cada instrumento exige uma variante própria do E1394.

---

## 3. Middleware no mercado brasileiro

| Produto | Tipo | O que foi visto |
|---|---|---|
| **Data Innovations Instrument Manager** | Middleware independente | Biblioteca de mais de 1.200 drivers, autoverificação de até 95%, CQ integrado com médias móveis [CONFIRMADO: https://intuitionlabs.ai/software/laboratory-automation-middleware/middleware-lis-analyzer-interface/data-innovations-instrument-manager]. Escritório no Brasil e site em português (pt.datainnovations.com) [INFERÊNCIA, trecho de buscador]. A Siemens revende o IM em vários países [INFERÊNCIA]. |
| **Siemens Atellica Data Manager** | Middleware do fabricante, multimarca | Página no Brasil: mais de 700 drivers, cerca de 1.500 modelos, inclui Abbott, Roche e Sysmex. Autovalidação por faixa, delta check, CQ e flags [CONFIRMADO: https://www.siemens-healthineers.com/br/diagnostics-it/atellica-diagnostics-it/atellica-data-manager]. |
| **Roche cobas infinity** (lab e POC) | Middleware ou LIS do fabricante | HL7, XML, SOAP, IHE LTW e conexão com terceiros por ASTM ou HL7 [CONFIRMADO: ficha Roche]. |
| **Sysmex WAM / Caresphere WS** | Middleware de hematologia | O WAM liga a série X e as linhas de automação e diz ter experiência com mais de 30 LIS. O Caresphere WS é o sucessor em nuvem. Fontes divergem sobre o status do WAM [INFERÊNCIA, trecho de buscador; a página Sysmex LA deu loop de redirecionamento]. |
| **Radiometer AQURE / Siemens RAPIDComm** | Data managers de POC | Gerenciam aparelhos, operadores e CQ e enviam resultados ao LIS [RAPIDComm: CONFIRMADO na página do DCA; AQURE: INFERÊNCIA]. |
| **LIS nacionais com interfaceamento próprio** (Shift, Matrix, Autolac, Pixeon Korus, MV Soul + interfaceador) | O LIS tem drivers próprios ou terceiriza | O Shift declara "instrument interfacing and management" [INFERÊNCIA, trecho de buscador LIMSWiki]. Para o Matrix não achei lista de interfaces. O HMS contratou um interfaceador separado para o MV Soul [CONFIRMADO: TR 026/2024]. Worklab e Labmaster não foram verificados [INFERÊNCIA]. |

**O padrão de fato é "LIS → middleware → aparelhos"?** Sim nos laboratórios médios e grandes [INFERÊNCIA bem fundamentada]. O próprio edital do HMS separa LIS (MV Soul) de interfaceador, e Sorriso exige servidor próprio da interface [CONFIRMADO]. Em laboratórios pequenos, com 1 a 3 aparelhos, o LIS nacional costuma ter driver direto para o aparelho, sem middleware [INFERÊNCIA]. O middleware fica com as regras de autovalidação, as repetições, a diluição e o CQ do aparelho. O LIS fica com o pedido, a liberação, o laudo e o faturamento.

---

## 4. O que um LIS precisa para integrar

1. **Identificação da amostra por código de barras.** É o requisito central: o aparelho lê o tubo e pergunta ao host "o que faço com a amostra X" (host query), ou recebe a lista de trabalho antes (download).
   - **Simbologia:** Code 128 é a mais comum; Codabar/NW7, ITF e Code 39 também são lidos pelos analisadores [INFERÊNCIA]. Joinville exige que o aparelho leia "no padrão de impressão do sistema de interfaceamento" [CONFIRMADO], ou seja, **quem define a etiqueta é o LIS**.
   - **Formato:** o CLSI AUTO12 padroniza conteúdo e posição da etiqueta, com tamanho de 2 × 1 polegadas [INFERÊNCIA, trecho de buscador: https://clsi.org/shop/standards/auto12/]. Não confirmei se ele obriga Code 128.
   - **Na prática:** número de amostra curto e numérico (8 a 12 dígitos), com sufixo por tubo ou material, porque um pedido gera vários tubos [INFERÊNCIA].
2. **Mapeamento exame ↔ código do aparelho.** Cada aparelho tem códigos próprios (ex.: "GLU", "TSH", "WBC"). O LIS precisa de uma tabela por aparelho que ligue exame ou analito (SIGTAP/LOINC no Rota Saúde) ao código do aparelho, com unidade e casas decimais. O manual do liat alerta que os códigos de teste do LIS devem ser únicos por teste [INFERÊNCIA, trecho de buscador]. Hematologia devolve 20 a 30 parâmetros por exame: o "hemograma" vira muitos resultados.
3. **Repetição, reteste e diluição.** O aparelho pode mandar o mesmo resultado mais de uma vez (reteste automático, diluição). O LIS precisa versionar o resultado bruto e deixar o profissional escolher qual liberar, sem nunca sobrescrever em silêncio. O caso Radiometer mostra que **fila de reenvio é ponto de risco de troca de paciente** [CONFIRMADO: aviso ANSM]. Por isso o LIS deve conferir ID da amostra mais ID do pedido mais exame esperado e rejeitar resultado de exame não pedido.
4. **Bidirecionalidade.** Na unidirecional, o aparelho só envia resultados e o operador digita os pedidos no aparelho. Na bidirecional, há host query por código de barras ou download de lista de trabalho. Os editais exigem bidirecional [CONFIRMADO: Sorriso, SMS-SP, Joinville, HMS].
5. **Controle de qualidade.** Resultados de controle (níveis 1, 2 e 3) também chegam pela interface (registros de CQ no ASTM, ou ORU com flag). O LIS ou o middleware precisa de Levey-Jennings e regras de Westgard [TR HMS 029/2023 exige Levey-Jennings: CONFIRMADO]. A RDC Anvisa 786/2023 aparece em Sorriso para controle externo [CONFIRMADO].
6. **Flags e autovalidação.** Flags do aparelho (H/L, alarmes de célula imatura, coágulo), delta check e faixa de referência. Geralmente o middleware cuida disso (seção 3).
7. **Infraestrutura física.** ASTM em RS-232 exige um PC ou conversor serial-IP no laboratório. **Um SaaS não fala com uma porta serial**: é preciso um agente local ou o middleware do fornecedor, que dentro da rede do laboratório converse com o Rota Saúde por HTTPS [INFERÊNCIA; é a restrição de arquitetura mais importante].

---

## 5. Recomendação para o Rota Saúde

| Opção | O que é | Custo [INFERÊNCIA] | Risco |
|---|---|---|---|
| **A. Não integrar** (o 21b como está) | Pedido, etiqueta, mapa de trabalho, digitação ou importação de resultado, liberação e laudo | Zero adicional | **Comercial alto**: todo edital de laboratório municipal lido exige interfaceamento bidirecional. Sem um caminho para isso, o 21b perde para o "LIS de graça" do fornecedor de reagente (Sorriso, João Dourado). Digitação manual de hemograma, com 20 ou mais parâmetros, é inviável e arriscada. |
| **B. Integrar via middleware do fornecedor do reagente** (recomendada como próximo passo) | O Rota Saúde publica **uma** interface documentada: HL7 v2.5.1 OUL^R22/ORU^R01 para resultado, OML^O33 ou query para pedido, ou um JSON/HTTPS próprio. Publica também a especificação de etiqueta (Code 128 + ID de amostra). O contrato municipal obriga o fornecedor do reagente a ligar o middleware dele (cobas infinity, Atellica DM, IM, WAM ou interfaceador nacional) a essa porta, como já fazem Sorriso, SMS-SP e HMS | Baixo a médio: cerca de 4 a 8 semanas para a porta HL7/JSON, o mapeamento por aparelho, o versionamento de resultado e o CQ básico. O custo de cada aparelho fica com o fornecedor | Depende da boa vontade e da competência de cada fornecedor (prazo, qualidade). Precisa de um "kit de homologação" (especificação, ambiente de teste, massa de dados). Risco clínico (troca de paciente) mitigado por validação de ID de amostra × pedido × exame. |
| **C. Integrar ASTM direto** | O Rota Saúde mantém drivers por aparelho (ASTM E1381/E1394, HL7 Mindray, POCT1-A2) e um **agente local** (serviço Windows/Linux no laboratório com porta serial/TCP, fila offline e envio por HTTPS) | Alto: cerca de 2 a 6 semanas por família de aparelho, mais o agente local, suporte em campo e homologação. Manuais só sob NDA ou via cliente | Alto: firmware muda e quebra o driver, há recall de transmissão (Sysmex XN, ABL90), o suporte é presencial e o produto deixa de ser SaaS puro (instalação local). É o que faz o interfaceador do HMS ou do Shift, e é um negócio à parte. |

**Sugestão** [INFERÊNCIA]: no 21b, entregar a opção A com o **modelo de dados já preparado para B**. Isso significa amostra com código de barras próprio (Code 128), tabela exame ↔ código de aparelho, resultado bruto versionado separado do resultado liberado, e resultados de CQ. Abrir um módulo 21c para a opção B (porta HL7/JSON + kit de homologação), validada com **um** fornecedor real de um laboratório municipal piloto. A opção C fica fora até haver demanda que pague por ela, e mesmo então provavelmente via parceria com um interfaceador nacional, não com drivers próprios.

---

## 6. Riscos

1. **Concorrência com o LIS embutido no reagente.** Editais que exigem "sistema de gestão laboratorial" no lote do reagente tiram espaço do 21b. Mitigação: orientar o município a separar o LIS do lote de reagente e a exigir que o fornecedor se interfaceie com "o sistema em uso pelo município" [CONFIRMADO o modelo: Sorriso diz "compatível com o Sistema... atualmente utilizado, ou outro Sistema que porventura venha substituí-lo"].
2. **Troca de paciente na interface.** Há aviso de segurança real (ABL90 FLEX HL7 v2.5) [CONFIRMADO]. Exige validação forte de ID e trilha de auditoria do resultado bruto.
3. **Manuais fechados.** A integração depende do fabricante ou do representante local. Não dá para planejar drivers sem acesso a cada laboratório cliente.
4. **Serial e rede local.** Muitos aparelhos de municípios pequenos ainda são RS-232. O SaaS precisa de um ponto local de alguém (do fornecedor ou nosso).
5. **Regulatório.** Software que faz autovalidação ou cálculo de resultado pode cair em enquadramento de software como dispositivo médico (Anvisa RDC 657/2022) [INFERÊNCIA; não verificado nesta sessão]. Limitar o Rota Saúde a receber, armazenar e liberar, e deixar regras de autovalidação no middleware, reduz esse risco.
6. **Esta pesquisa tem lacunas.** Vários editais (Balneário Camboriú, Pouso Alegre, editais de PE de 2026) foram vistos só por resumo de buscador. As marcas das atas ainda precisam ser conferidas no PNCP.

---

## 7. Perguntas em aberto

1. Nos municípios piloto do Rota Saúde, que aparelhos o laboratório municipal tem hoje, em que contrato (comodato, locação, próprio) e até quando? Isso define com quais fornecedores negociar.
2. Esses contratos têm cláusula de "interface com o LIS em uso ou que venha a substituí-lo"? Se tiverem, o fornecedor já é obrigado a conectar com o Rota Saúde sem custo para o município.
3. Que formato de porta o Rota Saúde oferece na opção B: HL7 v2.5.1 (IHE LAW/LTW), que os middlewares falam nativamente, ou JSON/FHIR (`ServiceRequest`/`Observation`), mais fácil para nós e menos para eles? Recomendo HL7 v2, porque é o que cobas infinity, Atellica DM e IM já exportam [INFERÊNCIA].
4. O laudo final continua no Rota Saúde (assinatura, módulo 19b) e o envio REL/RNDS (frente 3, 2026-10-05) parte dele? Isso reforça que o resultado liberado tem de nascer no Rota Saúde, não no middleware.
5. Vale uma parceria com um interfaceador nacional (Shift, o fornecedor do MV, outros) em vez da opção C?
6. Os testes rápidos da APS (HIV, sífilis, hepatites) e a HbA1c POC entram no 21b como registro manual na unidade, já que não há leitor com interface na rede?

---

## Fontes

Lidas (CONFIRMADO):
- Sorriso/MT, TR de comodato e reagentes (15/09/2025): https://site.sorriso.mt.gov.br/dl/638108
- Joinville/SC, convocação do PE 124/2025: https://www.joinville.sc.gov.br/public/edital/anexo/a7dbc7abd6f7f351a8138c3464d8efeb.pdf
- Joinville/SC, parecer técnico do PE 012/2024 (Alinity i): https://joinville.sc.gov.br/public/edital/anexo/48292724c035182a42b1deb37ae0b7dd.pdf
- Parnamirim/RN, TR do Laboratório Central (02/2024): https://antigo.parnamirim.rn.gov.br/pdf/pregao_licitacao/anexo_licitacao_16_02_2024_16_10_14_1903.pdf
- SMS São Paulo, TR de hematologia: https://drive.prefeitura.sp.gov.br/cidade/secretarias/upload/controladoria_geral/TERMOREFERENCIA_SMS_HEMATOLOGIA_publ_24_12_2020.pdf
- Hospital Municipal de Salvador, TR 029/2023 (gasometria): https://www.santacasaba.org.br/conteudo/dow/002/dowarq/arq/000454.pdf
- Hospital Municipal de Salvador, TR 026/2024 (interfaceamento): https://www.santacasaba.org.br/conteudo/dow/002/dowarq/arq/000619.pdf
- João Dourado/BA, JDPE014/2026 (agregador): https://alertalicitacao.com.br/!licitacao/PNCP-13891510000148-1-000103-2026
- Agência Minas, LIS do LACEN-MG: https://www.agenciaminas.mg.gov.br/noticia/novo-sistema-laboratorial-agiliza-entrega-de-exames-no-laboratorio-de-saude-publica/
- Beckman UniCel DxI/Access 2, LIS Vendor Information: https://www.beckmancoulter.com/download/file/wsr-228138/C03112AE?type=pdf
- Roche cobas liat, Host Interface Manual HL7: https://assets.cwp.roche.com/f/94122/x/e6e97fa41f/cobas-liat-system_hostinterfacemanual_hl7_sw3-4-3-5.pdf
- Roche cobas infinity, especificações: https://diagnostics.roche.com/content/dam/diagnostics/ch/de/cd-centralized-diagnostics/ch-en-cd-cobas-infinity-lab-solution-spezifikationen.pdf
- Siemens DCA Vantage: https://www.siemens-healthineers.com/diabetes/diabetes/dca-vantage-analyzer
- Siemens Atellica Data Manager (BR): https://www.siemens-healthineers.com/br/diagnostics-it/atellica-diagnostics-it/atellica-data-manager
- Data Innovations IM (perfil): https://intuitionlabs.ai/software/laboratory-automation-middleware/middleware-lis-analyzer-interface/data-innovations-instrument-manager
- ANSM, aviso de segurança Radiometer ABL90 FLEX (2012): https://archive.ansm.sante.fr/content/download/42382/550490/version/1/file/mes-120711-Automategazom%C3%A9trieABL90FLEX-+Radiometer.pdf

Vistas só por trecho de buscador (INFERÊNCIA):
- Balneário Camboriú, PE 210/2023: https://www.bc.sc.gov.br/arquivos/licitacao/RP8NS2EF.pdf
- Pouso Alegre/MG, Wiener: https://pousoalegre.mg.gov.br/licitacoes/ff544f32f433e826b4327e2719aa3c75.pdf
- Lages/SC, DFD 31/2024: https://licitacoes.lages.sc.gov.br/assets/licitacao/a74b4250b5efe6b9bffb24e75a5bcddc.pdf
- Recife/PE, PE 95/2026: https://alertalicitacao.com.br/!licitacao/CEPE-PE-5-373135
- Camaragibe/PE, PE 003/2026: https://alertalicitacao.com.br/!licitacao/BNC-2603454-751e-0032026
- Salgueiro/PE, Edital 056/2026: https://alertalicitacao.com.br/!licitacao/PNCP-11361243000171-1-000196-2026
- LACEN-RO, análise do PE 90088/2024: https://rondonia.ro.gov.br/wp-content/uploads/2025/05/SEI_0060507675_Analise_4.pdf
- Recall do Sysmex XN: https://medicaldevices.icij.org/events/aus-sysmex-xn-and-xn-l-series-all-sysmex-xn-analysers-versions-16-to-19
- CLSI AUTO12: https://clsi.org/shop/standards/auto12/
- Shift no LIMSWiki: https://limswiki.org/index.php/Vendor:Shift_Consultoria_e_Sistemas_Ltda.
- Sysmex WAM (LA): https://sysmex.com/la/en/Products/InformationSystems/Pages/WAM.aspx
- Conitec, HbA1c POC: https://www.gov.br/conitec/pt-br/midias/reuniao_conitec/2023/ata-da-119a-reuniao-ordinaria-da-conitec_produtos-e-procedimentos.pdf
