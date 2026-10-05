# Frente 3 — Exames e laboratórios

Pesquisa para o módulo de EXAMES do Rota Saúde (pedido → regulação → agendamento → coleta → resultado → ciência do profissional, com alertas, linha do tempo e canal web do cidadão).
Data da pesquisa: 2026-10-05. Legenda: **[CONFIRMADO: url]** = lido na fonte citada; **[INFERÊNCIA]** = dedução minha ou conhecimento não verificado nesta sessão.

---

## Resumo executivo

1. Os municípios fazem exames por quatro arranjos que convivem: laboratório municipal próprio, laboratório privado credenciado (chamamento público, pago pela tabela SUS ou tabela própria, produção lançada em BPA-I), consórcio intermunicipal (sistema web próprio do consórcio emite guias) e LACEN/GAL (só exames de vigilância). Em nenhum desses arranjos existe padrão nacional de pedido eletrônico: na prática a "integração" é guia em papel ou guia de um sistema web + laudo em PDF/papel.
2. A obrigação regulatória nova é do **laboratório**, não do município: a Portaria GM/MS 8.276/2025 (vigente desde 29/09/2025) manda que todo resultado laboratorial do país siga o modelo REL e seja enviado à RNDS; a Portaria GM/MS 10.341/2026 criou o RELC (laudo clínico, cobre imagem/ECG). Mas a especificação técnica publicada do REL ainda cobre só SARS-CoV-2 e Orthopoxvírus, e **não há serviço público documentado para um sistema municipal LER resultados de terceiros na RNDS** — a API documentada é de envio.
3. Terminologias: **SIGTAP (grupo 02)** é o código natural do PEDIDO (é o que o e-SUS PEC e o faturamento usam); **LOINC** é o código do RESULTADO exigido pelo REL; TUSS só importa se houver saúde suplementar (irrelevante para o MVP SUS).
4. Recomendação: **v1 = pedido estruturado (SIGTAP) + registro manual/upload de PDF do laudo pela unidade ou pelo próprio laboratório num portal de prestador do Rota Saúde**, com prazos/alertas — isso resolve o problema real (rastrear o que atrasou) sem depender de terceiros. **v2 = integração por laboratório (FHIR ServiceRequest/DiagnosticReport preferido; HL7 v2 OML/ORU e webservice proprietário como adaptadores)**, começando pelo laboratório municipal/credenciado de maior volume. **v3 = RNDS** (envio de REL quando o Rota Saúde for o LIS/prontuário, e leitura quando o MS abrir consulta para integradores). GAL fica fora: é sistema web do LACEN sem API pública documentada.

---

## 1. Arranjos típicos de realização de exames

### 1.1 Laboratório municipal próprio
- **O que é / quem opera:** laboratório da própria Secretaria Municipal de Saúde, com LIS comercial. Ex.: Laboratório Municipal de Campinas usa o LIS Matrix [CONFIRMADO: https://www.matrixsaude.com/laboratorio-da-prefeitura-de-campinas/]. Fornecedores citam funcionalidades para laboratório público como etiqueta, coleta, liberação de resultado e "faturamento SUS" [CONFIRMADO: https://www.matrixsaude.com/matrix-diagnosis-para-o-laboratorio-publico/].
- **Obrigatório/opcional:** opcional (decisão de gestão). Comum em municípios médios/grandes [INFERÊNCIA].
- **Padrão técnico:** depende do LIS; integração com prontuário municipal normalmente via HL7 v2 ou API do fornecedor [INFERÊNCIA]. A página do Matrix para laboratório público não cita HL7/FHIR/RNDS/e-SUS [CONFIRMADO: https://www.matrixsaude.com/matrix-diagnosis-para-o-laboratorio-publico/].
- **Pagamento:** custeio próprio; produção registrada no SIA (BPA) para compor teto MAC [INFERÊNCIA].
- **Dependência no Rota Saúde:** é o melhor candidato para a primeira integração automática (v2), porque o município controla o contrato do LIS.

### 1.2 Laboratório privado credenciado (contratado)
- **O que é:** pessoa jurídica credenciada por chamamento público para fazer exames a pacientes SUS. Edital real (Nonoai/RS, 2022): exame só com "guia emitida por Sistema de Autorização de Exames", assinatura do paciente na guia, relatório mensal até o 5º dia útil, e o laboratório deve "Lançar a produção (Exames Realizados), no Boletim de Produção Ambulatorial (BPA) individualizado" para a Secretaria exportar ao SIA [CONFIRMADO: https://www.nonoai.rs.gov.br/attachments/article/2462/Edital%20Chamamento%20P%C3%BAblico%20Credenciamento%20003-2022.pdf].
- **Pagamento:** valores da tabela anexa ao edital, pagos pelo município em até 15 dias úteis [CONFIRMADO: mesmo edital]. Se é tabela SUS pura ou com complementação municipal varia por edital [INFERÊNCIA].
- **Faturamento:** SIA/SUS com BPA-C (consolidado, sem identificação do paciente) e BPA-I (individualizado); a tabela SIGTAP define qual instrumento cada procedimento usa [CONFIRMADO: https://www.fhemig.mg.gov.br/frames/guia_faturamento/sia/ ; https://bvsms.saude.gov.br/bvs/publicacoes/auditoria_assitenciais_ambulatorial_hospitalar_sus_1_reimp.pdf].
- **Entrega do resultado:** o edital lido não exige integração com sistema da prefeitura nem formato de laudo [CONFIRMADO: mesmo edital — ausência]. Na prática é laudo impresso/PDF/portal do laboratório [INFERÊNCIA].
- **Dependência no Rota Saúde:** o Rota Saúde pode ser o "Sistema de Autorização de Exames" que emite a guia; o resultado volta por upload. Integração automática exige negociar com cada laboratório (e, se for o caso, o contrato pode passar a exigir).

### 1.3 Consórcio intermunicipal de saúde
- **O que é:** consórcio público que credencia clínicas/laboratórios para municípios consorciados. CONIMS (PR) credencia "exames clínicos/imagem, exames laboratoriais" por chamamento [CONFIRMADO: https://www.conims.pr.gov.br/arquivo_usu/documentoanexo/conims-20260908-095748.pdf].
- **Fluxo:** usuário "previamente agendado pelo município" e atendido mediante "guia de consulta/autorização gerada pelo município através do Sistema Web utilizado pelos municípios integrantes do CONIMS"; o profissional é obrigado a usar o prontuário eletrônico do sistema web do consórcio; contrato regido pela Lei 14.133/2021 [CONFIRMADO: mesmo documento, Termo de Referência itens 2.5, 2.6, 2.20 e cláusula 3.1].
- **Dependência no Rota Saúde:** **concorrência de sistema** — o consórcio já tem seu próprio sistema de guias. O Rota Saúde provavelmente terá de registrar "encaminhado ao consórcio, guia nº X" e receber o resultado por upload, sem integração [INFERÊNCIA].

### 1.4 LACEN / GAL (vigilância)
- **O que é:** GAL (Gerenciador de Ambiente Laboratorial) informatiza laboratórios de saúde pública da rede estadual (LACEN), com acompanhamento das etapas do exame e relatórios epidemiológicos [CONFIRMADO: https://saude.es.gov.br/gerenciador-de-ambiente-laboratorial-gal]. Sistema do DATASUS [CONFIRMADO: http://gal.datasus.gov.br/GALL/index.php ; https://medicinasa.com.br/lacen-gestao-laboratorial/].
- **Quem usa:** a unidade de saúde (municipal) cadastra a requisição no GAL; o laboratório **não entrega laudo ao paciente** — os resultados são disponibilizados pelas unidades que cadastraram a requisição [CONFIRMADO: https://saude.es.gov.br/gerenciador-de-ambiente-laboratorial-gal ; http://gal.datasus.gov.br/GAL/download/Manual_Operacao_Modulo_Usuario.pdf (citado nos resultados de busca)].
- **Acesso:** formulário de cadastro + termo de confidencialidade enviados à regional; o LACEN libera login [CONFIRMADO: https://saude.es.gov.br/gerenciador-de-ambiente-laboratorial-gal].
- **Integração:** há webservice da base nacional do GAL conectando sistemas públicos [CONFIRMADO (fonte secundária, resumo de busca do relato SciELO): http://scielo.iec.gov.br/scielo.php?script=sci_arttext&pid=S1679-49742013000300018 — página não abriu nesta sessão]. Em 2025 o Lacen-MG foi o 1º a integrar um LIS (Korus/Pixeon) ao GAL, via interface ponto a ponto com o DATASUS; padrão técnico não divulgado [CONFIRMADO: https://medicinasa.com.br/lacen-gestao-laboratorial/]. **Não encontrei API pública documentada para sistemas municipais requisitarem ou lerem resultados do GAL** [CONFIRMADO por ausência nas fontes acima; INFERÊNCIA quanto à inexistência].
- **Obrigatório/opcional:** obrigatório para exames de agravos de notificação encaminhados ao LACEN (o fluxo exige requisição GAL) [INFERÊNCIA a partir das fontes estaduais].
- **Dependência no Rota Saúde:** v1 = registrar "requisição GAL nº X" no pedido e anexar o laudo baixado do GAL. Integração só com acordo SES/DATASUS — não planejar.

### 1.5 Imagem em clínicas conveniadas
- **O que é:** RX, USG, TC, RM em clínica credenciada direto ou via consórcio; o CONIMS credencia radiologia e diagnóstico por imagem justamente por não ter radiologista próprio [CONFIRMADO: https://www.conims.pr.gov.br/arquivo_usu/documentoanexo/conims-20260908-095748.pdf].
- **Regulação:** exames especializados/SADT de média e alta complexidade passam pelo SISREG (módulo ambulatorial) ou sistema de regulação próprio do município/estado [CONFIRMADO: https://www.conass.org.br/guiainformacao/o-sisreg/ ; https://saude.sc.gov.br/index.php/pt/regulacao/manuais/manual-sisreg-executante-ambulatorial-28-06-2016/download (via resumo de busca)]. Não encontrei API pública do SISREG [INFERÊNCIA].
- Ver seção 6 (DICOM/PACS).

### 1.6 Quem paga e como — SIGTAP grupo 02
- Grupo 02 = "Procedimentos com finalidade diagnóstica"; subgrupo 02.02 = "Diagnóstico em laboratório clínico", com formas de organização como 01 Exames bioquímicos e 02 Hematológicos e hemostasia [CONFIRMADO: https://sigtap.com.br/grupo/02-procedimentos-com-finalidade-diagnostica (site não oficial); consulta oficial em http://sigtap.datasus.gov.br/tabela-unificada/app/sec/inicio.jsp]. O site não oficial informa 14 subgrupos e 1.105 procedimentos [CONFIRMADO no site não oficial; número a revalidar na competência vigente do SIGTAP oficial].
- Subgrupos (02.01 coleta de material; 02.03 anatomia patológica/citopatologia; 02.04 radiologia; 02.05 ultrassonografia; 02.06 tomografia; 02.07 ressonância; 02.08 medicina nuclear; 02.09 endoscopia; 02.11 métodos diagnósticos em especialidades; 02.13 vigilância; 02.14 testes rápidos …) [INFERÊNCIA — de memória; a existência de 02.14 é confirmada pelo exemplo `0214010015` no LEDI: https://integracao.esusaps.bridge.ufsc.tech/ledi/documentacao/estrutura_arquivos/dicionario-fai.html].
- Fluxo financeiro: produção ambulatorial em BPA (C ou I) → SIA/SUS [CONFIRMADO: https://www.fhemig.mg.gov.br/frames/guia_faturamento/sia/]. O Rota Saúde não precisa faturar no v1, mas guardar o código SIGTAP no pedido permite, mais tarde, exportar BPA-I ou conciliar a produção que o prestador alega [INFERÊNCIA].

---

## 2. Sistemas de laboratório (LIS) e formas de integração

### 2.1 LIS no mercado brasileiro
- Mais conhecidos: Shift Lab, Pixeon, Concent, Matrix, Bionexo, Soul MV, Pep Healthtech [CONFIRMADO (fonte de fornecedor concorrente): https://www.labix.com.br/perguntas/o-que-e-lis-laboratorio]. Outros citados: Worklab (Criasoft) [CONFIRMADO: https://criasoft.com.br/integracao-com-laboratorios-de-apoio/], Korus (Pixeon, no Lacen-MG) [CONFIRMADO: https://medicinasa.com.br/lacen-gestao-laboratorial/].
- Quais dominam o segmento SUS: **não encontrei dado público de participação de mercado** [INFERÊNCIA: Matrix e Shift aparecem com cases públicos; pergunta em aberto].
- Shift converte dados do LIS para os padrões da RNDS (FHIR, HL7, LOINC) com InterSystems IRIS [CONFIRMADO: https://shift.com.br/rnds/ ; https://abes.org.br/en/shift-cria-solucao-com-tecnologia-intersystems-para-laboratorios-cumprirem-portaria-do-ministerio-da-saude/]. Ou seja, LIS grandes **já têm** pipeline FHIR para a RNDS — dá para pedir que enviem o mesmo REL ao Rota Saúde [INFERÊNCIA].

### 2.2 Padrões técnicos
| Padrão | Uso | Relevância para o Rota Saúde |
|---|---|---|
| **HL7 v2 ORM^O01 / OML^O21 (pedido) e ORU^R01 (resultado)** | ORM é a mensagem genérica de pedido (pré-2.5); OML^O21 substitui ORM para laboratório a partir da v2.5 e tem segmento SPM (amostra); ORU^R01 devolve o resultado [CONFIRMADO: https://www.hl7.eu/refactored/msgOML_O21.html ; https://www.interfaceware.com/hl7-oru] | Adaptador v2 para LIS legados. Exige MLLP/TCP ou arquivo; não é "web friendly" [INFERÊNCIA]. |
| **FHIR R4 ServiceRequest → DiagnosticReport + Observation (+ Specimen)** | DiagnosticReport liga ao pedido por `basedOn`, aos resultados atômicos por `result`, aceita o laudo inteiro em `presentedForm` (PDF) e imagem por `imagingStudy`/`media`; maturidade 3, Trial Use [CONFIRMADO: https://hl7.org/fhir/R4/diagnosticreport.html] | **Modelo interno recomendado** — mesmo que a v1 seja manual, modelar como ServiceRequest/DiagnosticReport/Observation facilita v2 e v3. |
| **REL/RELC RNDS (FHIR R4)** | Bundle-documento com Composition, Condition, Observation e Specimen [CONFIRMADO: https://rnds-guia.saude.gov.br/docs/rel/mc-rel/] | Formato-alvo de saída (v3) e possível formato de entrada (pedir ao LIS o mesmo payload). |
| **ASTM (E1394 / LIS2-A2)** | Comunicação LIS ↔ equipamento analisador [INFERÊNCIA] | Irrelevante para o Rota Saúde (camada abaixo do LIS). |
| **Webservice/XML proprietário** | Laboratórios de apoio (ver 2.3) | Padrão de fato lab-to-lab; cada um com layout próprio. |
| **CSV/arquivo, portal com PDF** | Laboratórios pequenos/credenciados [INFERÊNCIA] | v1: upload. |

### 2.3 Laboratórios de apoio (lab-to-lab)
- DB Diagnósticos e Hermes Pardini: integração "XML ou WebService", sendo WebService a mais usada; Alvaro Apoio: XML [CONFIRMADO: https://migracao.github.io/pdfs/9/01_Configuracao_Apoio_Pastas.pdf (resumo de busca; manual de LIS terceiro)]. No modo WebService, a etiqueta do laboratório de apoio é impressa na triagem da amostra [CONFIRMADO: idem].
- Worklab envia e recebe dados de paciente e exames "via webservice" com os principais apoios, sem redigitação [CONFIRMADO (resumo de busca): https://criasoft.com.br/integracao-com-laboratorios-de-apoio/]. Hotsoft integra Pardini e DB por WebService; Wareline desenvolveu layout compatível com o Pardini [CONFIRMADO (resumo de busca): https://www.wareline.com.br/blog/case/case-laboratorio-hermes-pardini-2/]. TOTVS RM tem integração específica DB Diagnósticos e Hermes Pardini [CONFIRMADO (títulos): https://tdn.totvs.com/pages/viewpage.action?pageId=862605886 ; https://centraldeatendimento.totvs.com/hc/pt-br/articles/9762065257495-RM-SAU-Integra%C3%A7%C3%A3o-Hermes-Pardini] — documentação técnica fechada (403).
- Alvaro Apoio é o lab-to-lab da Dasa, com mais de 700 rotas [CONFIRMADO: https://dasa.com.br/blog/saude/alvaro-apoio-700-rotas-no-brasil/].
- **Conclusão:** apoio é integração **laboratório ↔ laboratório**, com layout proprietário e documentação liberada só a clientes. Para o Rota Saúde (que não é laboratório) isso só importa se o município tiver laboratório próprio que manda exames ao apoio — e aí quem integra é o LIS municipal [INFERÊNCIA].

---

## 3. Terminologias — qual usar para pedido e resultado

| Terminologia | Dono | Onde aparece | Uso recomendado |
|---|---|---|---|
| **SIGTAP** (Tabela de Procedimentos SUS, grupo 02) | MS/DATASUS | e-SUS APS aceita no campo exame da Ficha de Atendimento Individual código SIGTAP do grupo 02 sem pontuação (ex. `0214010015`) ou código AB (`ABEX022`) [CONFIRMADO: https://integracao.esusaps.bridge.ufsc.tech/ledi/documentacao/estrutura_arquivos/dicionario-fai.html, layout 8.7.0]; faturamento BPA [CONFIRMADO: https://www.fhemig.mg.gov.br/frames/guia_faturamento/sia/] | **Código do PEDIDO** (catálogo de exames da cidade). |
| **LOINC** | Regenstrief; tradução pt-BR via HL7 Brasil, com ANS, SMS-SP, ABRAMED, SBPC/ML [CONFIRMADO (resumo de busca): https://loinc.org/international/portuguese/] | REL usa LOINC para "nome do exame" [CONFIRMADO: https://rnds-guia.saude.gov.br/docs/rel/mi-rel/]; Portaria 8.276/2025 exige resultados "conforme Loinc" [CONFIRMADO: https://www.conass.org.br/conass-informa-n-175-2025-publicada-a-portaria-gm-n-8-276-que-institui-o-modelo-de-informacao-de-resultado-de-exame-laboratorial-rel-no-ambito-da-rede-nacional-de-dados-em-saude-r/] | **Código do RESULTADO** (cada analito/Observation). Opcional no v1; obrigatório para v3. |
| **TUSS** (Tabela 22) | ANS | Obrigatória na saúde suplementar (padrão TISS) [CONFIRMADO: https://www.gov.br/ans/pt-br/assuntos/prestadores/padrao-para-troca-de-informacao-de-saude-suplementar-2013-tiss/codigos-da-tuss] | **Não usar no MVP** (SUS). O guia de exames da SES-GO e o RELC aceitam Tabela SUS, TUSS ou CBHPM como classificação [CONFIRMADO: https://fhir.saude.go.gov.br/r4/exame/rels.html ; Conass Informa 47/2026 abaixo]. |
| **CID-10** | OMS/MS | Suspeita diagnóstica no REL [CONFIRMADO: https://rnds-guia.saude.gov.br/docs/rel/mi-rel/] | Indicação clínica do pedido (opcional). |
| **UCUM** | Regenstrief | Unidades no guia SES-GO [CONFIRMADO: https://fhir.saude.go.gov.br/r4/exame/rels.html] | Unidade de resultado numérico. |

- Mapeamento SIGTAP↔LOINC: o projeto LOINC-BR teria um sistema de mapeamento com TUSS/SUS/AMB [CONFIRMADO (resumo de busca, fonte secundária): https://labnetwork.com.br/noticias/especialistas-detalham-versao-brasileira-do-loinc-em-conferencia-realizada-em-sao-paulo/]. Não verifiquei se está disponível/atualizado [INFERÊNCIA — pergunta em aberto].
- Observação: um código SIGTAP (ex.: "hemograma completo") gera vários LOINC (um por analito). Portanto pedido = SIGTAP (1) → resultado = N Observations LOINC [INFERÊNCIA].

---

## 4. RNDS — REL e RELC

- **O que é a RNDS:** plataforma de interoperabilidade do SUS, regulamentada pelo Decreto 12.560, de 23/07/2025 [CONFIRMADO: https://www.planalto.gov.br/ccivil_03/_ato2023-2026/2025/decreto/d12560.htm ; https://www.conass.org.br/conass-informa-n-120-2025-publicado-decreto-n-12-560-que-dispoe-sobre-a-rede-nacional-de-dados-em-saude-e-sobre-as-plataformas-sus-digital-e-regulamenta-o-art-47-e-o-art-47-acaput/]. API FHIR R4 sobre HTTPS [CONFIRMADO: https://www.gov.br/conecta/catalogo/apis/rnds-rede-nacional-de-dados-em-saude].
- **Histórico de obrigatoriedade:** Portaria GM/MS 1.792, de 17/07/2020 — todos os laboratórios (públicos e privados) notificam todo resultado de COVID-19 à RNDS em até 24 h [CONFIRMADO: https://bvsms.saude.gov.br/bvs/saudelegis/gm/2020/prt1792_21_07_2020.html ; https://www.gov.br/saude/pt-br/assuntos/noticias/2020/julho/portaria-torna-obrigatoria-notificacao-de-resultados-de-testes-da-covid-19].
- **Situação atual (além da COVID):**
  - Portaria GM/MS 8.276/2025 (publicada 29/09/2025, vigente na publicação) institui o MI REL; diz que "os resultados de exame laboratorial de todo território nacional deverão seguir os padrões definidos nesta Portaria e ser enviados regularmente à RNDS"; mínimo de 21 elementos (id do resultado no sistema de origem, CPF ou CNS, CNES, responsável técnico, profissional liberador, datas de coleta/realização/emissão, agravo e agente etiológico quando couber, resultados em LOINC, valores de referência, assinatura eletrônica…); revoga a Portaria SAES/MS 883/2022; especificações técnicas a publicar no Portal de Serviços do DATASUS [CONFIRMADO: https://www.conass.org.br/conass-informa-n-175-2025-publicada-a-portaria-gm-n-8-276-que-institui-o-modelo-de-informacao-de-resultado-de-exame-laboratorial-rel-no-ambito-da-rede-nacional-de-dados-em-saude-r/]. A nota não traz prazo de adequação [CONFIRMADO por ausência].
  - Portaria GM/MS 10.341, de 12/03/2026, institui o RELC (Resultado de Exame com Laudo Clínico — inclui imagem, ECG, telediagnóstico; codificação LOINC/Tabela SUS/CBHPM/TUSS; 18 categorias mínimas, incluindo informação de acesso digital ao laudo e às imagens) e o RATC (teleconsultoria) [CONFIRMADO: https://www.conass.org.br/conass-informa-n-47-2026-publicada-a-portaria-gm-n-10-341-que-altera-a-portaria-de-consolidacao-no-1-17-para-instituir-o-modelo-de-informacao-para-registro-de-atendimento-via-telecon/].
  - **Porém, a especificação técnica publicada do MI REL ainda limita o escopo a SARS-CoV-2 e Orthopoxvírus/Monkeypox**; o resto está "fora do escopo" com expansão prevista [CONFIRMADO: https://rnds-guia.saude.gov.br/docs/rel/mi-rel/]. Existe um perfil "REL v2" em status *Informative* (CI build) [CONFIRMADO: https://rnds-fhir.saude.gov.br/rel-v2.html]. Ou seja: **norma ampla, especificação ainda estreita** [INFERÊNCIA a partir das duas fontes].
- **Modelo computacional REL:** Bundle tipo documento com Composition → Condition (suspeita) + Observation ("Diagnóstico em Laboratório Clínico") → Specimen (BRAmostraBiologica-1.0) [CONFIRMADO: https://rnds-guia.saude.gov.br/docs/rel/mc-rel/]. Não usa DiagnosticReport nem ServiceRequest no perfil nacional [CONFIRMADO: idem]. Guias estaduais (SES-GO) acrescentam ServiceRequest e distinguem resultado simples (RELs) e composto (RELc) [CONFIRMADO: https://fhir.saude.go.gov.br/r4/exame/rels.html ; https://fhir.saude.go.gov.br/r4/exame/relc.html].
- **Credenciamento para integrar:** estabelecimento com CNES válido; certificado digital ICP-Brasil; solicitação no Portal de Serviços do DATASUS (https://servicos-datasus.saude.gov.br/) com conta gov.br; homologação e depois produção [CONFIRMADO: https://webatendimento.saude.gov.br/faq/rnds ; https://rnds-guia.saude.gov.br/docs/publico-alvo/gestor/portal/]. Requisições levam o CNS do profissional lotado no CNES no header `Authorization`; token válido por 30 min [CONFIRMADO: https://rnds-guia.saude.gov.br/docs/publico-alvo/ti/conhecer/]. O "Conector" é software que cada estabelecimento/integrador desenvolve [CONFIRMADO: https://rnds-guia.saude.gov.br/docs/conector/].
- **O município pode LER resultados de terceiros?**
  - Serviços documentados: token, contexto-atendimento, GET Patient/Organization/Practitioner/PractitionerRole, POST Bundle (enviar e substituir resultado) [CONFIRMADO: https://rnds-guia.saude.gov.br/docs/rnds/servicos/]. O `contexto-atendimento` gera token "necessário para consultar documentos clínicos" no Conecte SUS Profissional [CONFIRMADO (resumo de busca): https://rnds-guia.saude.gov.br/docs/rnds/servicos/ ; https://www.saude.mg.gov.br/wp-content/uploads/2025/11/RNDS-Manual-Integracao-Barramento_vSite.pdf].
  - **Não encontrei endpoint público documentado para um sistema integrador consultar REL de outro estabelecimento** [CONFIRMADO por ausência nas páginas lidas]. A leitura existe para o **profissional** em contexto assistencial (Conecte SUS/Meu SUS Digital Profissional) e para o **cidadão** no Meu SUS Digital [CONFIRMADO (fonte secundária): https://rfsaldanha.github.io/sis/rnds.html ; FAQ oficial cita só COVID para o cidadão: https://webatendimento.saude.gov.br/faq/meususdigital].
  - Conclusão: **ler da RNDS é hoje pergunta em aberto para o MS**, não plano de engenharia [INFERÊNCIA].
- **Dependência no Rota Saúde:** se a cidade estiver em "modo prontuário" e o Rota Saúde registrar resultados (inclusive de teste rápido feito na UBS — SIGTAP 02.14), o Rota Saúde poderá ter de **enviar** REL à RNDS como estabelecimento/integrador [INFERÊNCIA: a Portaria 8.276 obriga "estabelecimentos que realizam exames"; teste rápido na UBS é exame realizado na UBS].

---

## 5. GAL — integração possível?

- Sistema web do DATASUS operado pelos LACEN; a unidade municipal requisita no próprio GAL e retira o laudo lá [CONFIRMADO: https://saude.es.gov.br/gerenciador-de-ambiente-laboratorial-gal].
- Interoperabilidade existente: webservice da base nacional entre sistemas públicos [CONFIRMADO (resumo de busca do relato SciELO): http://scielo.iec.gov.br/scielo.php?script=sci_arttext&pid=S1679-49742013000300018]; integração LIS↔GAL feita pela primeira vez em 2025 (Lacen-MG/Korus), ponto a ponto, padrão não divulgado [CONFIRMADO: https://medicinasa.com.br/lacen-gestao-laboratorial/].
- **Avaliação:** integração Rota Saúde↔GAL não é viável sem acordo formal com SES/DATASUS; nenhum serviço público documentado [INFERÊNCIA]. Com a Portaria 8.276/2025, resultados do LACEN deveriam chegar à RNDS, que seria o caminho indireto (v3) [INFERÊNCIA].
- **Mínimo viável:** campo "nº da requisição GAL" no pedido + upload do laudo + alerta de atraso pelo prazo do agravo.

---

## 6. Imagem — DICOM/PACS e laudo

- DICOM é o formato da imagem com metadados; PACS armazena e distribui; RIS gerencia o fluxo da radiologia; laudo a distância usa PACS/nuvem [CONFIRMADO: https://telepacs.com.br/como-funcionam-e-quais-as-diferencas-entre-pacs-ris-cis-lis-e-his/ ; https://portaltelemedicina.com.br/dicom].
- O RELC (Portaria 10.341/2026) prevê "informação de acesso digital ao laudo e às imagens" — ou seja, o modelo nacional referencia imagens por link, não as embute [CONFIRMADO: https://www.conass.org.br/conass-informa-n-47-2026-publicada-a-portaria-gm-n-10-341-que-altera-a-portaria-de-consolidacao-no-1-17-para-instituir-o-modelo-de-informacao-para-registro-de-atendimento-via-telecon/].
- O e-SUS APS já trata alguns exames de imagem com resultado estruturado mínimo (USG obstétrica em semanas/dias/DPP; TC/RM de crânio) [CONFIRMADO: https://integracao.esusaps.bridge.ufsc.tech/ledi/documentacao/referencias/exames_estruturados.html].
- **Mínimo viável:** sim — **só o laudo** (PDF + texto da conclusão + data + profissional laudador com CRM) e, opcionalmente, URL do visualizador do PACS da clínica. Armazenar DICOM não faz sentido para o Rota Saúde (volume, custo, responsabilidade) [INFERÊNCIA]. Equivale a DiagnosticReport com `presentedForm` e `conclusion` [CONFIRMADO o modelo: https://hl7.org/fhir/R4/diagnosticreport.html].

---

## 7. Relação com o e-SUS APS PEC ("modo de prontuário por cidade")

- No PEC, exames solicitados e/ou avaliados são registrados por código SIGTAP (grupo 02) ou código AB; resultado estruturado só para uma lista curta (HbA1c, colesterol total/HDL/LDL, triglicerídeos, creatinina, clearance, prova do laço, USG obstétrica, fundoscopia, triagens auditivas, TC/RM de crânio, USG transfontanela) [CONFIRMADO: https://integracao.esusaps.bridge.ufsc.tech/ledi/documentacao/estrutura_arquivos/dicionario-fai.html ; https://integracao.esusaps.bridge.ufsc.tech/ledi/documentacao/referencias/exames_estruturados.html]. O PEC tem botão "Adicionar resultados de exames" com resultado, data de solicitação e de realização (entrada manual) [CONFIRMADO (resumo de busca): http://aps.saude.gov.br/ape/esus/manual_3_2/capitulo6].
- O LEDI é layout de **envio** de fichas ao e-SUS (sistemas próprios → centralizador), não API de leitura do PEC [INFERÊNCIA a partir de https://integracao.esusaps.bridge.ufsc.tech/].
- **Consequência:** em cidades "modo PEC", o pedido nasce no PEC e o Rota Saúde não o vê automaticamente. Opções: (a) o profissional repete o pedido no Rota Saúde (retrabalho); (b) a regulação/marcação do exame é que entra no Rota Saúde (a recepção/central de marcação cadastra a guia); (c) ler o banco do PEC instalado na cidade (frágil, não suportado) [INFERÊNCIA]. Recomendo (b) para cidades PEC.

---

## 8. Estratégia recomendada em camadas

### v1 — Rastreamento com registro manual e upload (sem integração)
- **Escopo:** catálogo de exames da cidade (SIGTAP, com prazo esperado por exame e por prestador); pedido estruturado (ServiceRequest interno: exame SIGTAP, indicação/CID opcional, prioridade, solicitante, unidade); etapa de regulação opcional por exame (autorizado/negado/fila); agendamento da coleta/realização com prestador (municipal, credenciado, consórcio, LACEN); registro de coleta; resultado por **upload de PDF + campos mínimos** (data de liberação, laboratório, conclusão; valores estruturados opcionais para a lista do e-SUS); ciência do profissional (com quem e quando); alertas de SLA por etapa; linha do tempo; cidadão vê status e, se a cidade permitir, o PDF.
- **Portal do prestador (opcional dentro da v1):** login do laboratório credenciado no Rota Saúde para ver guias autorizadas e anexar laudos — tira a digitação da UBS e não exige integração técnica [INFERÊNCIA].
- **Prós:** funciona com todos os arranjos (inclusive consórcio e GAL); o valor está nos alertas de atraso e na ciência, não no dado estruturado; sem dependência externa; modelagem já em FHIR facilita v2/v3.
- **Contras:** digitação/upload; resultado pouco estruturado (não gera gráfico de série de HbA1c, por exemplo, sem digitar); risco de PDF anexado ao paciente errado (mitigar com conferência de CPF/CNS e trilha de auditoria); liberar PDF ao cidadão antes da ciência do profissional é decisão clínica/legal.

### v2 — Integração com LIS (por laboratório, sob demanda)
- **Escopo:** um endpoint de entrada **FHIR R4** (DiagnosticReport + Observation LOINC + `presentedForm` PDF, referenciando o ServiceRequest do Rota Saúde pelo identificador da guia) e, como adaptador, HL7 v2 OML^O21/ORU^R01; envio do pedido ao LIS quando ele aceitar.
- **Ordem:** 1º laboratório municipal próprio (município manda no contrato); 2º credenciado de maior volume (incluir cláusula de integração no próximo chamamento); consórcio só se o sistema dele expuser API.
- **Prós:** fim da digitação; dados estruturados (LOINC) para linha do tempo e indicadores; LIS que já enviam REL à RNDS reaproveitam o pipeline FHIR.
- **Contras:** um projeto por laboratório (mapeamento SIGTAP↔LOINC, identificação do paciente, homologação); custo cobrado pelo fornecedor do LIS; HL7 v2 exige infraestrutura de mensageria (MLLP/VPN) fora do padrão web do Rota Saúde.

### v3 — RNDS
- **Envio:** quando a cidade estiver em modo prontuário e houver exames realizados na unidade (testes rápidos 02.14) ou o Rota Saúde atuar como sistema do laboratório municipal, enviar REL/RELC como integrador credenciado (CNES + ICP-Brasil + homologação DATASUS).
- **Leitura:** só quando o MS documentar consulta por integrador; até lá, não planejar.
- **Prós:** fonte única nacional; cumpre Portaria 8.276/2025; cidadão vê no Meu SUS Digital.
- **Contras:** especificação publicada cobre só COVID/Mpox; leitura por integrador não documentada; credenciamento por estabelecimento/CNES (multiplica por cidade no modelo banco-por-cidade); certificado ICP-Brasil por cidade.

---

## 9. Riscos

1. **Regulatório em movimento:** Portaria 8.276/2025 e 10.341/2026 ampliam obrigação, mas a especificação técnica atrasa; investir cedo em REL pode exigir retrabalho [INFERÊNCIA].
2. **Responsabilidade do envio à RNDS:** se o Rota Saúde registrar resultado de exame feito na UBS, a cidade pode passar a ter obrigação de envio — definir quem é o "estabelecimento" (CNES da UBS) e se o Rota Saúde assume o papel de integrador [INFERÊNCIA].
3. **Concorrência de sistemas:** consórcio, SISREG e GAL já emitem guias; duplicar o pedido gera divergência. Mitigar tratando o Rota Saúde como rastreador com "referência externa" (nº da guia de outro sistema) [INFERÊNCIA].
4. **Identidade do paciente:** laudo de terceiro chega com nome/CPF/CNS digitado pelo laboratório; vinculação errada é incidente LGPD e clínico [INFERÊNCIA].
5. **Liberação ao cidadão:** mostrar resultado crítico (HIV, câncer) sem mediação profissional; precisa de política por tipo de exame e da ciência do profissional antes [INFERÊNCIA].
6. **Armazenamento:** PDFs/imagens aumentam volume e custo por banco de cidade; DICOM fora de escopo [INFERÊNCIA].
7. **Dependência de fornecedores de LIS:** documentação fechada (TOTVS/apoio retornaram 403) e cobrança por interface [CONFIRMADO o 403; custo = INFERÊNCIA].
8. **Cidades "modo PEC":** o pedido não chega ao Rota Saúde sem retrabalho; o módulo pode ficar vazio nessas cidades se depender do médico [INFERÊNCIA].

---

## 10. Perguntas em aberto

1. O MS/DATASUS vai (ou já permite) **consulta de REL por sistemas integradores** municipais (não só Conecte SUS Profissional)? Com qual perfil/escopo e base legal (Decreto 12.560/2025)?
2. Qual o **prazo** e a **especificação técnica** do REL amplo (Portaria 8.276/2025) para exames além de COVID/Mpox? O "REL v2" (status Informative) é a versão que vai valer?
3. Se a UBS faz teste rápido (02.14) registrado no Rota Saúde, **quem envia o REL**: a cidade (CNES da UBS) via Rota Saúde como integrador? Precisamos de um credenciamento RNDS por cidade?
4. Nas cidades-piloto: qual o arranjo real de cada tipo de exame (municipal, credenciado, consórcio, LACEN, clínica de imagem) e qual **LIS** cada laboratório usa?
5. Os contratos/chamamentos dos credenciados permitem exigir **integração ou uso de portal do prestador** do Rota Saúde? Há próximo chamamento onde incluir cláusula?
6. A regulação de exames na cidade é SISREG, sistema estadual, sistema do consórcio ou planilha? O Rota Saúde substitui ou só referencia?
7. Em cidades "modo PEC", **quem registra o pedido** no Rota Saúde (médico, recepção, central de marcação)?
8. Política de **liberação do resultado ao cidadão** no canal web: imediata, após ciência do profissional, ou por tipo de exame?
9. O mapeamento **SIGTAP↔LOINC** do projeto LOINC-BR está disponível, mantido e com licença utilizável?
10. Existe API pública do **GAL** para unidades requisitantes, ou só a interface web e integrações LIS↔GAL autorizadas caso a caso?
11. A cidade quer que o Rota Saúde gere **BPA-I** dos exames (substituindo o relatório do prestador) ou só concilie?
12. Prazos-alvo (SLA) por etapa: existem normas/protocolos municipais (ex.: rastreamento de câncer de colo, pré-natal) que definam prazos de resultado que devemos usar como padrão dos alertas?

---

## Fontes

Oficiais (MS/DATASUS/governo)
- https://rnds-guia.saude.gov.br/docs/rel/mi-rel/
- https://rnds-guia.saude.gov.br/docs/rel/mc-rel/
- https://rnds-fhir.saude.gov.br/rel-v2.html
- https://rnds-guia.saude.gov.br/docs/rnds/servicos/
- https://rnds-guia.saude.gov.br/docs/publico-alvo/ti/conhecer/
- https://rnds-guia.saude.gov.br/docs/publico-alvo/gestor/portal/
- https://rnds-guia.saude.gov.br/docs/conector/
- https://webatendimento.saude.gov.br/faq/rnds
- https://webatendimento.saude.gov.br/faq/meususdigital
- https://www.gov.br/conecta/catalogo/apis/rnds-rede-nacional-de-dados-em-saude
- https://bvsms.saude.gov.br/bvs/saudelegis/gm/2020/prt1792_21_07_2020.html
- https://www.gov.br/saude/pt-br/assuntos/noticias/2020/julho/portaria-torna-obrigatoria-notificacao-de-resultados-de-testes-da-covid-19
- https://www.planalto.gov.br/ccivil_03/_ato2023-2026/2025/decreto/d12560.htm
- https://www.conass.org.br/conass-informa-n-175-2025-publicada-a-portaria-gm-n-8-276-que-institui-o-modelo-de-informacao-de-resultado-de-exame-laboratorial-rel-no-ambito-da-rede-nacional-de-dados-em-saude-r/
- https://www.conass.org.br/conass-informa-n-47-2026-publicada-a-portaria-gm-n-10-341-que-altera-a-portaria-de-consolidacao-no-1-17-para-instituir-o-modelo-de-informacao-para-registro-de-atendimento-via-telecon/
- https://www.conass.org.br/conass-informa-n-120-2025-publicado-decreto-n-12-560-que-dispoe-sobre-a-rede-nacional-de-dados-em-saude-e-sobre-as-plataformas-sus-digital-e-regulamenta-o-art-47-e-o-art-47-acaput/
- https://www.conass.org.br/guiainformacao/o-sisreg/
- https://fhir.saude.go.gov.br/r4/exame/rels.html ; https://fhir.saude.go.gov.br/r4/exame/relc.html
- https://saude.es.gov.br/gerenciador-de-ambiente-laboratorial-gal
- http://gal.datasus.gov.br/GALL/index.php
- http://sigtap.datasus.gov.br/tabela-unificada/app/sec/inicio.jsp
- https://integracao.esusaps.bridge.ufsc.tech/ledi/documentacao/estrutura_arquivos/dicionario-fai.html
- https://integracao.esusaps.bridge.ufsc.tech/ledi/documentacao/referencias/exames_estruturados.html
- https://www.fhemig.mg.gov.br/frames/guia_faturamento/sia/
- https://www.gov.br/ans/pt-br/assuntos/prestadores/padrao-para-troca-de-informacao-de-saude-suplementar-2013-tiss/codigos-da-tuss
- https://www.nonoai.rs.gov.br/attachments/article/2462/Edital%20Chamamento%20P%C3%BAblico%20Credenciamento%20003-2022.pdf
- https://www.conims.pr.gov.br/arquivo_usu/documentoanexo/conims-20260908-095748.pdf

Padrões
- https://hl7.org/fhir/R4/diagnosticreport.html
- https://www.hl7.eu/refactored/msgOML_O21.html
- https://www.interfaceware.com/hl7-oru
- https://loinc.org/international/portuguese/ (403 nesta sessão; conteúdo via resumo de busca)

Fornecedores e secundárias
- https://shift.com.br/rnds/ ; https://abes.org.br/en/shift-cria-solucao-com-tecnologia-intersystems-para-laboratorios-cumprirem-portaria-do-ministerio-da-saude/
- https://www.matrixsaude.com/laboratorio-da-prefeitura-de-campinas/ ; https://www.matrixsaude.com/matrix-diagnosis-para-o-laboratorio-publico/
- https://www.labix.com.br/perguntas/o-que-e-lis-laboratorio
- https://medicinasa.com.br/lacen-gestao-laboratorial/
- https://criasoft.com.br/integracao-com-laboratorios-de-apoio/
- https://migracao.github.io/pdfs/9/01_Configuracao_Apoio_Pastas.pdf
- https://www.wareline.com.br/blog/case/case-laboratorio-hermes-pardini-2/
- https://tdn.totvs.com/pages/viewpage.action?pageId=862605886 (403)
- https://dasa.com.br/blog/saude/alvaro-apoio-700-rotas-no-brasil/
- https://sigtap.com.br/grupo/02-procedimentos-com-finalidade-diagnostica
- https://labnetwork.com.br/noticias/especialistas-detalham-versao-brasileira-do-loinc-em-conferencia-realizada-em-sao-paulo/
- https://rfsaldanha.github.io/sis/rnds.html
- https://telepacs.com.br/como-funcionam-e-quais-as-diferencas-entre-pacs-ris-cis-lis-e-his/ ; https://portaltelemedicina.com.br/dicom
- http://scielo.iec.gov.br/scielo.php?script=sci_arttext&pid=S1679-49742013000300018 (não abriu; via resumo de busca)
