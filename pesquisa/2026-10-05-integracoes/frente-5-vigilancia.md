# Frente 5 — Vigilância (epidemiológica e sanitária)

Pesquisa para o Rota Saúde · data da pesquisa: 2026-10-05
Convenção: **[CONFIRMADO: url]** = lido na fonte citada; **[INFERÊNCIA]** = conclusão minha, não está escrita em fonte oficial lida nesta pesquisa.

---

## Resumo executivo

1. A Lista Nacional de Notificação Compulsória fica no Anexo 1 do Anexo V da Portaria de Consolidação GM/MS nº 4/2017. A redação vigente é a da **Portaria GM/MS nº 11.211, de 13/05/2026**: são 66 itens, com caxumba e Oropouche incluídas e a covid-19 retirada como item próprio. Ela veio depois da **Portaria GM/MS nº 10.175/2026**, que incluiu as anomalias congênitas. O texto da 11.211 tem uma contradição sobre a caxumba, e o MS respondeu que vale o anexo (notificação semanal). A obrigação de notificar vale para médicos, demais profissionais de saúde e responsáveis por estabelecimentos públicos e privados.
2. **A notificação oficial hoje é digitada em sistemas do MS.** Sinan Net trabalha por lote semanal, Sinan Online cuida de dengue e chikungunya, e e-SUS Sinan, e-SUS Notifica e SIVEP-Gripe são online. Não achei nenhuma API pública de envio ao Sinan/e-SUS Sinan para sistemas de terceiros. A única API documentada de envio (o "Robô Notifica", do e-SUS Notifica) é restrita a covid/síndrome gripal e a estados e municípios integrados.
3. O **e-SUS Sinan** vai substituir Sinan Net, Sinan Online, e-SUS Notifica e SIVEP-Gripe, mas ainda cobre poucos agravos (Mpox, Oropouche, HTLV, hepatite B, doença falciforme, entre outros). Integrar com ele está entre as metas, mas não é um caminho que exista hoje.
4. **Vacinação é o único ponto com integração obrigatória e já definida.** A Portaria GM/MS 5.663/2024 exige que todo sistema que registra dose aplicada envie à **RNDS no modelo RIA (FHIR R4)** em até 24h e guarde o retorno de cada envio. O caminho antigo por Thrift/LEDI foi desligado para vacina a partir de set/2025. Se o Rota Saúde registrar vacina, a RNDS deixa de ser opcional.
5. Recomendação para a vigilância epidemiológica: **v1 = notificação interna** disparada pelo desfecho do atendimento (CID), com fila da vigilância municipal, alerta de 24h para os agravos imediatos, ficha no formato Sinan para digitar e um painel de casos por bairro e semana epidemiológica. **v2 = integração oficial** (RNDS-RIA, se houver vacina; e-SUS Sinan quando o MS abrir a interface). Até lá, não registrar dose aplicada.
6. **VISA** é um produto de gestão municipal sem sistema nacional obrigatório. O SINAVISA da Anvisa existiu e, segundo relato de 2024, saiu do ar em 2021. O mercado é de ERPs municipais (IPM, Inovadora e outros). O que é obrigatório é **regulatório**, e não uma integração específica: classificação de risco em 3 níveis (RDC 153/2017 alterada pela RDC 418/2020, IN 66/2020, CGSIM 62/2020 alterada pela CGSIM 66/2021), Lei de Liberdade Econômica (licença dispensada no baixo risco, aprovação tácita) e adesão à Redesim pelo integrador estadual, que muda de estado para estado. A Anvisa revisa a classificação de risco (CP 1.249/2024) e um modelo de risco para inspeções (consulta de 2026). As duas coisas estão em movimento.

---

# PARTE A — Vigilância epidemiológica

## A1. Lista Nacional de Notificação Compulsória (LNNC)

### Base normativa vigente
- A lista é o **Anexo 1 do Anexo V da Portaria de Consolidação GM/MS nº 4/2017** e vale para serviços de saúde públicos e privados em todo o território nacional. [CONFIRMADO: https://bvsms.saude.gov.br/bvs/saudelegis/gm/2026/prt11211_14_05_2026.html]
- **Portaria GM/MS nº 10.175, de 23/01/2026** (DOU 26/01/2026): incluiu "Anomalias congênitas" (item 4), com notificação **semanal**. [CONFIRMADO: https://bvsms.saude.gov.br/bvs/saudelegis/gm/2026/prt10175_26_01_2026.html]
- **Portaria GM/MS nº 11.211, de 13/05/2026**, é a redação vigente do anexo. Ela:
  - inclui a Parotidite (caxumba) e a Febre do Oropouche;
  - remove a "covid-19" como item próprio, que passa a constar como "Síndrome Gripal por covid-19 confirmada";
  - renomeia itens como ESAVI, Poliomielite/PFA, Varicela e "Srag hospitalizado ou óbito por Srag";
  - torna a difteria imediata também para o MS e muda a periodicidade do tétano;
  - altera a lista sentinela (Portaria de Consolidação nº 5, Anexo XLIII): inclui DIH/DPI e exclui SRAG.
  
  [CONFIRMADO: https://bvsms.saude.gov.br/bvs/saudelegis/gm/2026/prt11211_14_05_2026.html]
- **Contradição na caxumba.** O art. 2º da 11.211 diz "imediata", mas o Anexo I marca "semanal". Em resposta via Fala.BR (13/08/2026), o MS disse que vale o anexo (semanal) e que vai publicar um ato para retificar, ainda sem data. [CONFIRMADO: https://med.estrategia.com/portal/noticias/ministerio-da-saude-reconhece-erro-em-portaria-sobre-notificacao-de-caxumba/] *(fonte secundária, que cita a resposta oficial)*
- Antes disso, a Portaria GM/MS 6.734/2025 tinha incluído a esporotricose humana. [CONFIRMADO: https://bvsms.saude.gov.br/bvs/saudelegis/gm/2025/prt6734_31_03_2025.html]
- Estados e municípios podem acrescentar agravos de relevância local. [CONFIRMADO: https://www.gov.br/saude/pt-br/composicao/svsa/notificacao-compulsoria]
- **Implicação para o produto:** a lista mudou 3 vezes em ~14 meses (mar/2025, jan/2026, mai/2026) e ainda tem uma retificação pendente. Ela precisa ser **dado versionado** (tabela de referência com vigência), nunca código fixo. [INFERÊNCIA]

### Quem notifica
- "Médicos, profissionais de saúde ou responsáveis pelos estabelecimentos de saúde, públicos ou privados". [CONFIRMADO: https://www.gov.br/saude/pt-br/composicao/svsa/notificacao-compulsoria]
- A notificação parte da unidade notificante (pública ou privada) e segue para a Secretaria Municipal de Saúde, que digita e encaminha ao estado e ao MS. [CONFIRMADO: Manual de Gestão Sinan SES-RJ, mar/2025 — https://www.saude.rj.gov.br/comum/code/MostrarArquivo.php?C=NzI0OTc%2C]
- **Notificação negativa:** no Sinan Net, toda semana epidemiológica precisa ter um lote enviado, com notificações **ou com notificação negativa**. [CONFIRMADO: manual SES-RJ acima]

### Periodicidade (Anexo I da Portaria 11.211/2026)
"Imediata" quer dizer até 24 horas, comunicada à esfera indicada (MS, SES, SMS). "Semanal" quer dizer até 7 dias. [CONFIRMADO: https://bvsms.saude.gov.br/bvs/saudelegis/gm/2026/prt11211_14_05_2026.html]

**Imediata para MS, SES e SMS:**
- botulismo; cólera; dengue (óbitos); difteria;
- doenças com suspeita de disseminação intencional (antraz pneumônico, tularemia, varíola);
- febres hemorrágicas emergentes (arenavírus, Ebola, Marburg, Lassa, febre purpúrica brasileira);
- Zika (óbito suspeito); Evento de Saúde Pública (ESP); ESAVI; febre amarela;
- chikungunya (em área sem transmissão e óbito); febre do Nilo Ocidental e outras arboviroses;
- Oropouche (óbitos e óbitos fetais, gestante, anomalias congênitas);
- febre maculosa e outras riquetsioses; hantavirose; influenza por novo subtipo;
- malária extra-amazônica; Mpox; peste; poliomielite/PFA; raiva humana;
- síndrome da rubéola congênita; sarampo e rubéola; SIM-A e SIM-P associadas à covid-19;
- SRAG (SARS-CoV, MERS-CoV, Srag hospitalizado ou óbito); síndrome gripal por covid-19 confirmada;
- tétano neonatal.

**Imediata para SES e SMS:**
- coqueluche; Chagas aguda; doença invasiva por *H. influenzae*; doença meningocócica e outras meningites;
- Zika em gestante; febre tifoide; tétano acidental; varicela.

**Imediata para SMS:**
- acidente de trabalho (sem exposição biológica); acidente por animal peçonhento;
- acidente por animal potencialmente transmissor da raiva; leptospirose;
- violência sexual e tentativa de suicídio.

**Semanal:**
- acidente de trabalho com exposição a material biológico; anomalias congênitas; câncer relacionado ao trabalho;
- dengue (casos); dermatoses ocupacionais; distúrbio de voz relacionado ao trabalho; Chagas crônica;
- doença de Creutzfeldt-Jakob; doença falciforme; Zika (doença aguda e síndrome congênita);
- esporotricose humana; esquistossomose; chikungunya (casos); Oropouche (casos);
- hanseníase; hepatites virais; hepatite B, HIV e HTLV em gestante, parturiente, puérpera e criança exposta;
- HIV/AIDS; infecção por HIV; HTLV; intoxicação exógena; leishmaniose tegumentar; leishmaniose visceral;
- LER/DORT; malária na região amazônica; óbito infantil e materno; caxumba (conforme o anexo);
- perda auditiva relacionada ao trabalho; pneumoconioses; sífilis (adquirida, congênita, em gestante);
- toxoplasmose gestacional e congênita; transtornos mentais relacionados ao trabalho; tuberculose;
- violência doméstica e outras violências.

> Observação: o art. 5º, II, "b", da 11.211 tem uma redação truncada sobre o tétano. Usei o que diz o Anexo I. [INFERÊNCIA]

---

## A2. Sinan (Net / Online) e e-SUS Sinan

### Sinan Net
- **O que é:** sistema desktop. A SMS digita as fichas e gera **lote semanal** para a SES (pelo menos 1 por semana epidemiológica). O MS devolve à SMS um "Fluxo de Retorno" pelo Portal Sinan-net. [CONFIRMADO: manual SES-RJ 2025 — https://www.saude.rj.gov.br/comum/code/MostrarArquivo.php?C=NzI0OTc%2C]
- **Quem opera:** a vigilância epidemiológica municipal, ou o local que a SMS designar para digitar. A unidade preenche a ficha em papel ou PDF. [CONFIRMADO: idem]
- **Obrigatório?** Sim, para os agravos que ainda passam por ele, que são a maioria. [CONFIRMADO: idem; https://rfsaldanha.github.io/sis/sinan.html]
- **Padrão técnico / integração:** não existe API para terceiros. A entrada é por digitação e a troca entre esferas é por lote/arquivo. [INFERÊNCIA — não achei nenhuma documentação de interface externa no portalsinan.saude.gov.br nem nos manuais]
- **Documentação:** fichas e manuais em https://portalsinan.saude.gov.br/ [CONFIRMADO]

### Sinan Online
- Usado para dengue e chikungunya. A notificação vai direto para a base nacional, sem lote. [CONFIRMADO: manual SES-RJ; https://www.gov.br/saude/pt-br/assuntos/saude-de-a-a-z/a/arboviroses/informe-semanal/2025/informe-semanal-no-01.pdf]
- Também não tem API para terceiros. [INFERÊNCIA]

### e-SUS Sinan
- **O que é:** plataforma 100% online que vai substituir Sinan Net, Sinan Online, e-SUS Notifica, SIVEP-Gripe e outros. Ela muda o modelo: **primeiro o paciente, depois uma ou mais investigações** (no Sinan é "uma ficha por agravo"). Também descentraliza: a notificação e a investigação podem acontecer na APS, e a VE encerra. [CONFIRMADO: manual SES-RJ, mar/2025]
- **Agravos habilitados:**
  - Mpox (desde set/2022) e Oropouche [CONFIRMADO: manual SES-RJ];
  - no DF, também HTLV, hepatite B, doença falciforme, distúrbio de voz relacionado ao trabalho e esporotricose [CONFIRMADO: https://www.saude.df.gov.br/sistema-de-informacao-de-agravos-de-notificacao];
  - em Porto Alegre, doença falciforme, esporotricose, HTLV e Mpox [CONFIRMADO: https://prefeitura.poa.br/sms/vigilancia-em-saude/sistemas-de-informacao].
  - Na prática, os agravos que entraram na lista há pouco tendem a nascer no e-SUS Sinan. [INFERÊNCIA]
- **Acesso:** usuário via SCPA (gov.br/DATASUS). O perfil **Notificador** é liberado automaticamente, então qualquer profissional cadastrado pode notificar direto. [CONFIRMADO: Manual de instruções e-SUS Sinan 2022 — https://bvsms.saude.gov.br/bvs/publicacoes/esus_sinan_manual_instrucoes.pdf]
- **Integração:** "fortalecer a integração com a RNDS" aparece como objetivo no manual. [CONFIRMADO: idem] Não achei nenhum modelo de informação de notificação de agravo publicado na RNDS. Os modelos documentados no guia são RIA/RIA-R, REL (exames, nascido para covid), RAC, SA, RPM e RDM. [CONFIRMADO: https://rnds-guia.saude.gov.br/docs/rel/mi-rel/ e o índice do guia] **Hoje não há como integrar a notificação compulsória pela RNDS.** [INFERÊNCIA]
- Versão 2.4.0 de 27/04/2025 (DocumentDB 8, Kafka), segundo fonte secundária. [CONFIRMADO só em fonte secundária: https://rfsaldanha.github.io/sis/sinan.html — não verificado em fonte oficial]
- **Custo:** gratuito para o SUS. [INFERÊNCIA]
- **Módulos do Rota Saúde que dependem disso:** vigilância (notificação, investigação), desfecho do atendimento (gatilho por CID) e analytics.

### e-SUS Notifica (complemento)
- Criado em 27/03/2020 para covid/SG leve. Também é usado para Chagas crônica e ESAVI (em Porto Alegre, por exemplo). [CONFIRMADO: https://prefeitura.poa.br/sms/vigilancia-em-saude/sistemas-de-informacao; manual SES-RJ]
- **API documentada:**
  - O "Manual de Integração e-SUS Notifica — Módulo Covid-19" (DATASUS, v1.0, 26/08/2021) descreve uma API REST de **consumo** (proxy de Elasticsearch) só para estados, DF e municípios com mais de 300 mil habitantes. [CONFIRMADO: https://datasus.saude.gov.br/wp-content/uploads/2022/02/Manual-de-Utilizacao-da-API-e-Sus-Notifica.pdf]
  - Quem **envia** dados por API ("robô/sistemas próprios") fica fora da exigência de cadastro de IP. Quem consome precisa de formulário, IP fixo e termo de responsabilidade. [CONFIRMADO: https://datasus.saude.gov.br/cadastramento-de-ip-para-acesso-a-api-de-consumo-de-dados-no-e-sus-notifica/]
- Serve de precedente: o MS **já aceitou envio por sistema próprio** em um sistema de notificação. Mas o escopo é covid e não há documentação pública de onboarding para fornecedor privado. [INFERÊNCIA]

### Precedente municipal relevante
- Porto Alegre usa um sistema próprio, o **"Sentinela"** (Procempa), para todos os profissionais notificarem dengue, hanseníase, hepatites, HIV, sífilis, TB, agravos do trabalho, violências e outros. [CONFIRMADO: https://prefeitura.poa.br/sms/vigilancia-em-saude/sistemas-de-informacao] Esse é o modelo da v1 proposta abaixo: um sistema municipal que recebe as notificações e uma vigilância que alimenta o sistema oficial. [INFERÊNCIA]

---

## A3. Sistemas específicos

### SIVEP-Gripe
- **O que é:** registro de SRAG hospitalizada e óbitos por SRAG, além das unidades sentinela de síndrome gripal. [CONFIRMADO: https://portal-adm.campinas.sp.gov.br/sites/default/files/secretarias/arquivos-avulsos/125/2026/03/16-150120/NT_01_2026_DEVISA-SMS_VirusRespiratorios.pdf (NT municipal 01/2026)]
- **Quem opera:** hospitais públicos e privados, UPA, SAMU, SVO e SMS cadastrados, em até 24h. [CONFIRMADO: idem] Em Porto Alegre, só profissionais de hospital. [CONFIRMADO: prefeitura.poa.br acima]
- **Relevância para o Rota Saúde:** baixa no curto prazo, porque o foco é APS/UBS e não internação. A Portaria 11.211 também tirou a SRAG da estratégia sentinela. [CONFIRMADO: portaria 11.211] [INFERÊNCIA quanto à relevância]
- API: não achei. [INFERÊNCIA]

### GAL (Gerenciador de Ambiente Laboratorial)
- **O que é:** sistema dos LACEN e da rede de laboratórios de saúde pública para exames de interesse da vigilância. O município cadastra o paciente e a requisição, e o laboratório libera o laudo. [CONFIRMADO: http://scielo.iec.gov.br/scielo.php?script=sci_arttext&pid=S1679-49742013000300018; https://saude.es.gov.br/gerenciador-de-ambiente-laboratorial-gal]
- **Quem opera:** a unidade solicitante (municipal), o LACEN e a vigilância. [CONFIRMADO: idem]
- **Relevância:** a investigação de caso costuma depender de exame no GAL (dengue, sarampo e outros). No Rota Saúde, a v1 deve guardar só "nº da requisição GAL" e o resultado digitado. [INFERÊNCIA]
- Integração: os resultados de covid do GAL e de laboratórios privados chegavam ao e-SUS Notifica via RNDS (modelo REL). [CONFIRMADO: busca sobre e-SUS Notifica/RNDS; https://rnds-guia.saude.gov.br/docs/introducao/] Não há API pública do GAL para terceiros. [INFERÊNCIA]

### SIM / SINASC (óbito e nascido vivo)
- **SINASC:**
  - A DN (Declaração de Nascido Vivo) é preenchida pelos profissionais de saúde ou pelas parteiras tradicionais que assistiram o parto. A **SMS digita, processa e valida** no SINASC local e transmite pelo Sisnet. A DN tem 3 vias numeradas que o MS distribui. [CONFIRMADO: https://www.gov.br/saude/pt-br/composicao/svsa/sistemas-de-informacao/sinasc]
  - Existem iniciativas estaduais de DNV digital em FHIR, como a SES-GO. [CONFIRMADO: https://fhir.saude.go.gov.br/r4/core/declaracao-nascido-vivo.html]
  - Não achei um cronograma nacional de DN eletrônica. [INFERÊNCIA]
- **SIM:**
  - A DO é emitida exclusivamente por médico (Res. CFM 1.779/2005). O controle da distribuição dos formulários e da digitação é da vigilância. [CONFIRMADO: https://www.saude.df.gov.br/sistema-de-informacao-sobre-mortalidade]
  - Não achei um cronograma nacional de DO eletrônica. [INFERÊNCIA]
- **Relevância:** só para uma futura maternidade ou hospital. "Óbito infantil e materno" está na LNNC (semanal), e o Rota Saúde pode **disparar a investigação de óbito** a partir do desfecho "óbito" sem emitir DO. [CONFIRMADO quanto à LNNC; INFERÊNCIA quanto ao produto]

### Arboviroses (dengue, chikungunya, Zika, Oropouche)
- Dengue e chikungunya vão no **Sinan Online**. A Zika, historicamente, no Sinan Net. [CONFIRMADO: https://rfsaldanha.github.io/sis/sinan.html; busca sobre ES/e-SUS VS]
- O Oropouche vai no **e-SUS Sinan**. [CONFIRMADO: manual SES-RJ]
- Não achei nota oficial sobre migrar dengue e chikungunya para o e-SUS Sinan em 2026. [INFERÊNCIA — a NT 7/2026-CGARB citada em busca trata da investigação no contexto da vacinação ampliada, não do sistema]
- **Relevância:** é o caso de uso nº 1 para "alerta de surto por território" (casos de dengue por bairro e semana epidemiológica). [INFERÊNCIA]

---

## A4. Vacinação — SI-PNI e RNDS (RIA)

- **Portaria GM/MS nº 5.663, de 31/10/2024** (DOU 04/11/2024), que acrescenta o art. 312-A na Portaria de Consolidação nº 1/2017 e entra em vigor 120 dias depois da publicação:
  - os sistemas de vacinação devem enviar as doses aplicadas **exclusivamente à RNDS, no modelo RIA vigente**;
  - na APS, o registro só pode ser feito no **PEC**, no **CDS** ou em **sistema próprio ou de terceiro integrado à RNDS**;
  - o sistema terceiro deve **guardar o retorno de controle da integração (identificador do registro e status de sucesso ou erro)** e enviar **em até 24h** (salas com conectividade) ou **15 dias** (salas sem conectividade);
  - quando as orientações técnicas mudam, o fornecedor tem **15 dias** para se adequar;
  - estados e municípios escolhem o sistema, desde que ele siga as regras de interoperabilidade da RNDS;
  - a integração Thrift/XML (LEDI) foi desativada para vacinas.
  
  [CONFIRMADO: texto do DOU em https://www.cosemssp.org.br/wp-content/uploads/2025/02/RNDS-PORTARIA-5.663.pdf; original https://www.in.gov.br/en/web/dou/-/portaria-gm/ms-n-5.663-de-31-de-outubro-de-2024-593693777]
- Desde **setembro/2025** o envio de imunização por Thrift não é mais aceito. Os sistemas municipais e de terceiros precisam usar RIA/RNDS. [CONFIRMADO: https://www.cosemssp.org.br/noticias/sistemas-de-vacinacao-deverao-estar-integrados-a-rnds-ate-setembro/ (21/07/2025)]
- **Padrão técnico:**
  - HL7 **FHIR R4**. O RIA-R (rotina) é um Bundle com Composition (type "RIA") e Immunization, com perfis nacionais no Simplifier (https://simplifier.net/redenacionaldedadosemsaude).
  - Campos obrigatórios incluem: paciente, CNES do autor, vacina, data e hora, fabricante, lote, local e via de aplicação, profissional e protocolo (dose/estratégia).
  - Existe também o RIA-C para campanha.
  
  [CONFIRMADO: https://rnds-guia.saude.gov.br/docs/ria-rotina/mc-ria-r/; busca sobre RIA-R/RIA-C]
- **Credenciamento:**
  - O **gestor do estabelecimento** obtém certificado digital ICP-Brasil (e-CPF ou e-CNPJ, tipo A1), cria conta gov.br e pede acesso pelo **Portal de Serviços DATASUS**. Recebe um "identificador do solicitante", faz a **homologação** com evidências e depois pede acesso à **produção**.
  - A RNDS identifica as requisições do estabelecimento pelo certificado.
  
  [CONFIRMADO: https://rnds-guia.saude.gov.br/docs/passo-a-passo/; https://rnds-guia.saude.gov.br/docs/publico-alvo/gestor/certificado/]
  - Detalhes de token: o endpoint EHR-auth emite um token a partir do certificado. Os detalhes de header e validade não foram confirmados nesta pesquisa. [CONFIRMADO parcialmente: https://rnds-guia.saude.gov.br/docs/publico-alvo/ti/autenticacao/]
  - Num SaaS multi-município, cada município ou estabelecimento (CNES) solicitante precisa do seu credenciamento e certificado. Isso casa com o modelo "banco por cidade" do Rota Saúde. [INFERÊNCIA]
- **Custo:** a RNDS é gratuita. O custo é o certificado ICP-Brasil (Serpro, Correios e outras ACs) e o esforço de homologação. [INFERÊNCIA; emissores CONFIRMADOS no guia]
- **ESAVI** (evento supostamente atribuível à vacinação) é de notificação imediata e foi notificado no e-SUS Notifica. [CONFIRMADO: portaria 11.211; prefeitura.poa.br]
- **Módulos do Rota Saúde que dependem disso:** campanhas (módulo 12, se registrar a aplicação), agendamento e vigilância (ESAVI).
- **Regra de decisão:** convocar, agendar e acompanhar a cobertura vacinal **não** dispara a obrigação. **Registrar a dose aplicada** dispara: RNDS-RIA obrigatória, prazo de 24h e guarda do retorno. [INFERÊNCIA a partir do art. 312-A]

---

## A5. Recomendação em camadas — módulo de vigilância epidemiológica

### v1 — Notificação interna + apoio à vigilância municipal (sem integração oficial)
1. **Catálogo versionado da LNNC:**
   - tabela de agravos com vigência (portaria de origem), periodicidade por esfera (MS/SES/SMS) e mapeamento CID-10 sugerido;
   - agravos locais por cidade, o que encaixa no banco por cidade.
   
   [INFERÊNCIA]
2. **Gatilho no desfecho do atendimento:** quando o CID do desfecho bate com um agravo do catálogo, o sistema sugere notificar. Quem decide é o profissional. Para os agravos imediatos, mostra o prazo de 24h. [INFERÊNCIA]
3. **Ficha interna mínima:** identificação (CPF/CNS), endereço com bairro e área do território, data dos primeiros sintomas, agravo, unidade notificante, CNES e notificador. O modelo segue o e-SUS Sinan: o paciente tem 1..N suspeitas e investigações. [CONFIRMADO quanto ao modelo e-SUS Sinan; INFERÊNCIA quanto ao produto]
4. **Fila da vigilância municipal:**
   - status: notificado → em investigação → encerrado (confirmado/descartado/inconclusivo) → **digitado no sistema oficial**, com o nº da notificação Sinan/e-SUS Sinan como campo manual;
   - alerta imediato (painel + e-mail) para os agravos de 24h;
   - lembrete semanal de **notificação negativa**.
   
   [INFERÊNCIA]
5. **Saídas:**
   - ficha em PDF no layout da ficha Sinan do agravo, para digitar;
   - CSV por semana epidemiológica;
   - relatório de "notificações sem número oficial", para controle de subnotificação e duplicidade.
   
   [INFERÊNCIA]
6. **Painel territorial:**
   - casos por bairro/área e semana epidemiológica, com supressão de células pequenas (reaproveitando o analytics);
   - alerta interno de **aumento atípico** (limiar simples: média móvel + k·desvio ou diagrama de controle por bairro), apresentado como "sinal para a VE avaliar" e nunca como "surto declarado".
   
   [INFERÊNCIA]
7. **Não fazer na v1:**
   - registrar dose de vacina, que obrigaria a RNDS;
   - emitir DO/DN;
   - prometer "envio automático ao Sinan".
   
   [INFERÊNCIA]

### v2 — Integração oficial
- **RNDS-RIA:** só se o produto passar a registrar a aplicação de vacina. Escopo: conector FHIR, credenciamento por CNES, fila com retry, guarda do retorno e reenvio em até 24h. [CONFIRMADO quanto à obrigação; INFERÊNCIA quanto ao escopo]
- **e-SUS Sinan:** acompanhar a abertura de interface para terceiros, seja pela RNDS ou por API própria. Até isso existir, a integração seria por RPA, o que é frágil e provavelmente não autorizado. **Não recomendo.** [INFERÊNCIA]
- **e-SUS Notifica:** só faz sentido para SG por covid confirmada, que está em queda de relevância. Baixa prioridade. [INFERÊNCIA]
- **GAL:** importar resultados só se o estado oferecer exportação. [INFERÊNCIA]

### v3 — Inteligência territorial
- Detecção espaço-temporal (tipo varredura), cruzamento com campanhas e agenda (busca ativa) e integração com dados de arbovírus/LIRAa. [INFERÊNCIA]

---

# PARTE B — Vigilância sanitária (VISA)

## B1. Como funciona a VISA municipal

- **Sistema:** o SNVS foi criado pela Lei 9.782/1999 dentro do SUS (Lei 8.080/1990), com execução descentralizada entre União, estados e municípios. [CONFIRMADO: https://planalto.gov.br/ccivil_03/leis/l9782.htm (referência); https://www.scielo.br/j/physis/a/j4P6D5nBmfZ7Mcp4j3DNHpf/?lang=pt]
- **Pactuação:**
  - as ações de alto risco são pactuadas entre estado e municípios na **CIB**, e as de baixo risco ficam com o município;
  - os municípios entraram na pactuação de média e alta complexidade a partir de 2004 (Portaria Anvisa 2.473/2003).
  - Isso varia por estado (ex.: Resolução CIB-AM 414/2025; pactuação SC).
  
  [CONFIRMADO: busca — https://www.scielo.br/j/physis/a/j4P6D5nBmfZ7Mcp4j3DNHpf/?lang=pt; https://vigilanciasanitaria.saude.sc.gov.br/index.php/servicos/pactuacao.html; http://ses.saude.am.gov.br/uploads/storage/cib/docs/res/2025_414_31072025100757.pdf]
  - **Implicação:** o catálogo de "o que esta cidade licencia e inspeciona" depende do estado e da pactuação. É configuração por cidade. [INFERÊNCIA]
- **Classificação de risco:**
  - **RDC Anvisa 153/2017** (critérios de classificação de risco e simplificação do licenciamento) e IN 16/2017 (lista CNAE). A RDC 418/2020 alterou a 153 e criou o nível **médio**. A **IN 66/2020** é a lista CNAE vigente por grau de risco. [CONFIRMADO: https://anvisalegis.datalegis.net/action/ActionDatalegis.php?acao=abrirTextoAto&link=S&tipo=RDC&numeroAto=00000153&seqAto=000&valorAno=2017&orgao=RDC/DC/ANVISA/MS&cod_modulo=310&cod_menu=9431; https://www.contadores.cnt.br/legislacoes/instrucao-normativa-dc-anvisa-no-66-de-01-09-2020.html]
  - **Resolução CGSIM 62/2020**, alterada pela **CGSIM 66/2021**, define 3 níveis para o licenciamento sanitário:
    - **I (baixo risco):** funciona sem inspeção e sem licença, sujeito a fiscalização posterior;
    - **II (médio):** recebe licença provisória e é inspecionado depois;
    - **III (alto):** precisa de inspeção e licença prévias.
    
    [CONFIRMADO: https://www.gov.br/empresas-e-negocios/pt-br/drei/cgsim/resolucoes-cgsim/arquivos/resolucao-62-2020-revogado-pela-resolucao-66-v2.pdf; definição dos níveis em https://www.kvlaw.com.br/anvisa-abre-consulta-publica-visando-a-alteracao-das-diretrizes-de-classificacao-de-risco-para-atividades-economicas-sujeitas-a-vigilancia-sanitaria/]
  - **Em revisão:** a Anvisa abriu a **CP 1.249/2024** com uma minuta de RDC que substitui as regras de identificação e classificação do grau de risco. O resultado foi publicado (428 respostas; 54% concordaram). Não achei a RDC final publicada. [CONFIRMADO: https://anexosportal.datalegis.net/arquivos/1888063.pdf; https://regulacaoemnumeros-direitorio.fgv.br/post/anvisa-abre-cp-sobre-risco-sanitario-de-atividades-economicas] [INFERÊNCIA: a norma final pode sair durante o desenvolvimento]
- **Lei de Liberdade Econômica (Lei 13.874/2019):**
  - o art. 3º, I, garante o direito de exercer atividade de baixo risco **sem ato público de liberação**;
  - o **Decreto 10.178/2019** regulamenta a classificação federal e a **aprovação tácita** quando o órgão perde o prazo.
  
  [CONFIRMADO: https://normas.leg.br/?urn=urn%3Alex%3Abr%3Afederal%3Alei%3A2019-09-20%3B13874%21art3; https://www2.camara.leg.br/legin/fed/decret/2019/decreto-10178-18-dezembro-2019-789614-norma-pe.html]
  - A classificação federal (CGSIM) vale na falta de norma estadual ou municipal própria. [INFERÊNCIA — não consegui ler o §1º do art. 3º no Planalto; confirmar]
  - **Implicação:** o produto precisa de **prazo e SLA por pedido**, com alerta de aprovação tácita. [INFERÊNCIA]
- **Realidade municipal** (levantamento de 2019, 2.111 municípios):
  - 62% usam algum sistema informatizado para o licenciamento;
  - só 20,2% estão integrados à Redesim;
  - 78% classificam risco (45% por norma estadual, 43% pela RDC 153, 12% por norma municipal);
  - só **15,4%** aplicam de fato o rito simplificado do baixo risco;
  - 74% cobram taxa;
  - o prazo típico é de 5 a 30 dias.
  
  [CONFIRMADO: https://www.redalyc.org/journal/5705/570567431010/html/]
- **Processo administrativo sanitário:** a **Lei 6.437/1977** (infrações e penalidades) é usada nos municípios que não têm Código Sanitário próprio. [CONFIRMADO: https://repositorio.furg.br/items/129043f8-28f5-4341-a757-b3d27623fd81]
- **Licença sanitária × alvará:** a licença sanitária é ato da VISA. O alvará de funcionamento é da prefeitura (posturas/urbanismo). No integrador estadual, as duas podem compor um "licenciamento integrado". [CONFIRMADO para SP: https://www.facilitasp.sp.gov.br/wp-content/uploads/2024/06/ADESAO-A-REDESIM-final-compactado.pdf] [INFERÊNCIA quanto à generalização]
- **Taxas:** são instituídas por lei municipal (código tributário) e cobradas por guia municipal. Isso exige integração com o sistema tributário da prefeitura. [INFERÊNCIA]
- **Faturamento SUS das ações de VISA:** existem procedimentos de VISA na Tabela SUS (Portaria MS/SAS 323/2010), registrados no SIA/SUS com repasse vinculado. É uma saída de dados que o produto pode gerar (BPA). [CONFIRMADO: https://www.redalyc.org/journal/5705/570578445002/570578445002_2.pdf (Vigil Sanit Debate 2024;12:e02145)]

## B2. Sistemas existentes

| Sistema | O que é | Obrigatório para o município? | Integração |
|---|---|---|---|
| **Redesim / integrador estadual** (Lei 11.598/2007) | Entrada única para registro e licenciamento de empresas. O integrador estadual (ex.: VRE/Facilita SP, da JUCESP) liga Junta, Receita, prefeituras e órgãos licenciadores (VISA, Bombeiros etc.). | A adesão é **voluntária** por convênio (gratuito em SP). As regras de simplificação e classificação de risco são obrigatórias. [CONFIRMADO: facilitasp PDF acima; https://antigo.redesim.gov.br/servicos/constitua-sua-pj/passo-3-licencas/orientacoes] | O integrador "se conecta diretamente a sistemas desses órgãos (...) Vigilância Sanitária". O protocolo e a API são **por estado** e não há especificação nacional única pública. [CONFIRMADO em parte; INFERÊNCIA no resto] |
| **SINAVISA** (Anvisa) | Sistema de cadastro de estabelecimentos, inspeções, taxas e licenças criado em 2002 a partir do sistema de Goiás. Chegou a 1.400+ municípios. | Não | Segundo relato de 2024, a Anvisa tentou reformulá-lo em 2015 sem avanço, e "o sistema saiu do ar no 2º semestre de 2021". Goiás ainda descreve um SINAVISA próprio. [CONFIRMADO: https://www.redalyc.org/journal/5705/570578445002/570578445002_2.pdf; https://goias.gov.br/saude/sinavisa-vigilancia-sanitaria/] |
| **NOTIVISA** (Anvisa) | Notificação de eventos adversos e queixas técnicas de produtos (pós-mercado). | Usado pelas VISAs, sem integração exigida | Sem API pública conhecida. [CONFIRMADO quanto à existência: redalyc acima] [INFERÊNCIA quanto à API] |
| **SNGPC** (Anvisa) | Escrituração de medicamentos controlados (farmácias e drogarias). | Obrigatório **para o estabelecimento**, não para o sistema municipal | A VISA consulta. Fora do escopo v1. [CONFIRMADO: redalyc acima] |
| **SIA/SUS (BPA)** | Registro dos procedimentos de VISA para financiamento. | Sim, para registrar a produção | Exportação BPA. [CONFIRMADO: redalyc acima] |
| **Sistemas privados** | ERPs municipais com módulo de VISA: IPM Atende.Net (cadastro, inspeção, autos, licenças), Inovadora G-VIS, Tecnologia.link e outros. | — | Normalmente fazem parte da suíte tributária da prefeitura, o que é uma vantagem competitiva deles. [CONFIRMADO: https://www.ipm.com.br/vigilancia/; https://www.inovadora.com.br/solucao/vigilancia-em-saude/; https://tecnologia.link/nossos-produtos/software-gestao-vigilancia-sanitaria/] |

- **Inspeção — referência nacional em formação:** a Anvisa abriu em 12/06/2026 (até 07/08/2026) uma consulta às VISAs sobre o **MARP — Modelo de Avaliação de Risco Potencial**, baseado em **19 Roteiros Objetivos de Inspeção (ROI)** para serviços de saúde e de interesse à saúde. [CONFIRMADO: https://www.gov.br/anvisa/pt-br/assuntos/noticias-anvisa/2026/anvisa-lanca-consulta-nacional-para-avaliar-modelo-de-risco-em-inspecoes-de-servicos-de-saude-e-de-interesse-para-saude] Os roteiros do produto deveriam nascer compatíveis com os ROI. [INFERÊNCIA]

## B3. Escopo típico de um produto de VISA municipal
[INFERÊNCIA consolidada a partir de IPM, Inovadora, SINAVISA-GO e do levantamento Fiocruz/Redalyc citados acima]
1. **Cadastro de estabelecimentos:** CNPJ/CPF, CNAEs, responsável técnico, endereço com bairro e área (reaproveita o território), classificação de risco derivada do CNAE e de questionário, histórico.
2. **Licenciamento:**
   - pedido (balcão, portal ou integrador Redesim), análise documental, licença provisória (nível II), inspeção prévia (nível III), dispensa (nível I);
   - emissão e validade da licença, renovação;
   - contagem de prazo com **alerta de aprovação tácita**.
3. **Análise de projeto arquitetônico** (para os estabelecimentos que exigem).
4. **Inspeção:** agenda e planejamento por risco, roteiro (checklist ROI) em tablet/offline, evidências (fotos), termo de inspeção, plano de ação e prazos de adequação.
5. **Denúncias:** canal do cidadão (pode usar o wpda), triagem, vínculo com inspeção, retorno ao denunciante (com anonimato).
6. **Autos e PAS:** auto de infração, termos de intimação, interdição e apreensão, defesa e recurso, prazos, julgamento, penalidade (Lei 6.437/1977 ou código municipal) e trânsito em julgado.
7. **Taxas e multas:** cálculo, emissão de guia (integração com o tributário municipal), baixa.
8. **Indicadores e saídas:** produção em BPA/SIA, metas pactuadas (CIB), relatórios.
9. **Interfaces:** integrador estadual Redesim (por estado), assinatura digital (gov.br/ICP), publicação de atos.

**Módulos do Rota Saúde reaproveitáveis:** identidade e papéis (com dupla assinatura no julgamento de auto, como no ADR 0016), território (bairros/áreas), canal web do cidadão (denúncia), auditoria e banco por cidade. [INFERÊNCIA]

---

## Riscos

1. **Regulatório da vacina:** registrar dose sem integrar à RNDS descumpre o art. 312-A da PC 1/2017 (Portaria 5.663/2024). Além disso, o fornecedor tem só 15 dias para se adequar quando o PNI muda as regras. É um custo de manutenção contínuo. [CONFIRMADO]
2. **Lista instável:** 3 alterações em 14 meses e uma retificação pendente (caxumba). Um catálogo fixo no código vai produzir notificação com prazo errado. [CONFIRMADO]
3. **Alvo móvel do MS:** o e-SUS Sinan está migrando aos poucos. Qualquer exportação no layout Sinan Net pode ficar obsoleta agravo a agravo. [CONFIRMADO quanto à migração; INFERÊNCIA quanto ao impacto]
4. **Duplicidade e subnotificação:** se a unidade também notifica direto no e-SUS Sinan (perfil Notificador é automático), o mesmo caso aparece duas vezes. A VE precisa reconciliar. [INFERÊNCIA]
5. **Expectativa errada:** vender "envio ao Sinan" sem existir API. A comunicação comercial precisa dizer "prepara para a vigilância digitar". [INFERÊNCIA]
6. **LGPD e sigilo:**
   - agravos como HIV, sífilis, violência e tentativa de suicídio exigem acesso restrito por papel e trilha de auditoria;
   - o painel territorial precisa de supressão de células pequenas, porque bairro pequeno + agravo raro identifica a pessoa.
   
   [INFERÊNCIA — o sigilo legal da notificação (Lei 6.259/1975) não foi lido nesta pesquisa]
7. **Residência × atendimento:** a notificação segue o município de residência para a investigação. Com um banco por cidade, um caso de morador de outra cidade atendido aqui precisa de fluxo de transferência ou aviso. [INFERÊNCIA]
8. **"Surto" como diagnóstico:** um alerta automático tratado como declaração oficial expõe o município. O produto só sinaliza; quem declara é a VE. [INFERÊNCIA]
9. **VISA — fragmentação:** Redesim e integrador por estado, pactuação por CIB e taxas por lei municipal. Cada estado é um projeto de integração. [CONFIRMADO em parte]
10. **VISA — concorrência:** os ERPs municipais já trazem VISA junto com o tributário (guias e cadastro mobiliário). Um produto isolado sofre na emissão de guia e no cadastro econômico. [INFERÊNCIA]
11. **VISA — revisão normativa:** a RDC que vai substituir a 153/2017 e a IN 66/2020 (CP 1.249/2024) e o MARP/ROI (2026) podem mudar o modelo de risco durante o desenvolvimento. [CONFIRMADO quanto às consultas]

## Perguntas em aberto

1. O Rota Saúde vai **registrar dose aplicada** (campanhas/módulo 12) ou só convocar e agendar? Isso define se a RNDS-RIA é obrigatória.
2. O MS tem previsão pública de **interface de terceiros para o e-SUS Sinan** (RNDS ou API) e de migrar dengue e chikungunya? Vale perguntar formalmente (Fala.BR / CGIAE-SVSA).
3. O **ato de retificação da Portaria 11.211/2026** (caxumba) já saiu depois de 13/08/2026?
4. Nos municípios-alvo, **quem digita no Sinan** (VE central ou distrito) e quem é notificador no e-SUS Sinan? Muda o desenho da fila.
5. Os estados-alvo acrescentam **agravos estaduais** à lista (SES-PR/SC/SP)? Quais?
6. O §1º do art. 3º da Lei 13.874/2019 confirma que a classificação CGSIM só vale na falta de norma local? Falta ler o texto oficial.
7. A **RDC final da CP 1.249/2024** já foi publicada? Revoga a RDC 153/2017 e a IN 66/2020?
8. Em cada estado-alvo, o **integrador Redesim tem webservice para órgão licenciador municipal** com sistema próprio? Qual é a especificação?
9. Para VISA, o município aceita **guia de taxa emitida fora do ERP tributário**, ou é obrigatório integrar?
10. Token e headers da RNDS (validade, CPF do profissional no header) para o desenho do conector. Confirmar no guia de TI e no ambiente de homologação.
11. O SINAVISA da Anvisa está mesmo descontinuado em nível nacional? Só tenho o relato de 2024 de que saiu do ar em 2021.

## Fontes

**Lista de notificação e vigilância epidemiológica**
- Portaria GM/MS 11.211/2026 — https://bvsms.saude.gov.br/bvs/saudelegis/gm/2026/prt11211_14_05_2026.html
- Portaria GM/MS 10.175/2026 — https://bvsms.saude.gov.br/bvs/saudelegis/gm/2026/prt10175_26_01_2026.html
- Portaria GM/MS 6.734/2025 — https://bvsms.saude.gov.br/bvs/saudelegis/gm/2025/prt6734_31_03_2025.html
- MS — Notificação compulsória — https://www.gov.br/saude/pt-br/composicao/svsa/notificacao-compulsoria
- Caxumba / Fala.BR — https://med.estrategia.com/portal/noticias/ministerio-da-saude-reconhece-erro-em-portaria-sobre-notificacao-de-caxumba/

**Sinan, e-SUS Sinan e e-SUS Notifica**
- Manual de Gestão Sinan, SES-RJ (mar/2025) — https://www.saude.rj.gov.br/comum/code/MostrarArquivo.php?C=NzI0OTc%2C
- Manual e-SUS Sinan (2022) — https://bvsms.saude.gov.br/bvs/publicacoes/esus_sinan_manual_instrucoes.pdf
- SES-DF Sinan — https://www.saude.df.gov.br/sistema-de-informacao-de-agravos-de-notificacao
- Porto Alegre — sistemas de VE — https://prefeitura.poa.br/sms/vigilancia-em-saude/sistemas-de-informacao
- Portal Sinan — https://portalsinan.saude.gov.br/
- e-SUS Notifica — manual da API — https://datasus.saude.gov.br/wp-content/uploads/2022/02/Manual-de-Utilizacao-da-API-e-Sus-Notifica.pdf
- e-SUS Notifica — cadastro de IP — https://datasus.saude.gov.br/cadastramento-de-ip-para-acesso-a-api-de-consumo-de-dados-no-e-sus-notifica/
- Resumo SIS (secundária) — https://rfsaldanha.github.io/sis/sinan.html

**SIVEP-Gripe, GAL, SINASC e SIM**
- NT Campinas 01/2026 (SIVEP-Gripe) — https://portal-adm.campinas.sp.gov.br/sites/default/files/secretarias/arquivos-avulsos/125/2026/03/16-150120/NT_01_2026_DEVISA-SMS_VirusRespiratorios.pdf
- GAL — http://scielo.iec.gov.br/scielo.php?script=sci_arttext&pid=S1679-49742013000300018 ; https://saude.es.gov.br/gerenciador-de-ambiente-laboratorial-gal
- SINASC — https://www.gov.br/saude/pt-br/composicao/svsa/sistemas-de-informacao/sinasc
- SIM (DF) — https://www.saude.df.gov.br/sistema-de-informacao-sobre-mortalidade
- DNV FHIR (SES-GO) — https://fhir.saude.go.gov.br/r4/core/declaracao-nascido-vivo.html

**Vacinação e RNDS**
- Portaria GM/MS 5.663/2024 — https://www.cosemssp.org.br/wp-content/uploads/2025/02/RNDS-PORTARIA-5.663.pdf
- COSEMS-SP (Thrift até set/2025) — https://www.cosemssp.org.br/noticias/sistemas-de-vacinacao-deverao-estar-integrados-a-rnds-ate-setembro/
- RNDS — RIA-R — https://rnds-guia.saude.gov.br/docs/ria-rotina/mc-ria-r/
- RNDS — passo a passo — https://rnds-guia.saude.gov.br/docs/passo-a-passo/
- RNDS — certificado — https://rnds-guia.saude.gov.br/docs/publico-alvo/gestor/certificado/
- RNDS — autenticação — https://rnds-guia.saude.gov.br/docs/publico-alvo/ti/autenticacao/
- RNDS — introdução e REL — https://rnds-guia.saude.gov.br/docs/introducao/

**Vigilância sanitária — normas**
- RDC 153/2017 — https://anvisalegis.datalegis.net/action/ActionDatalegis.php?acao=abrirTextoAto&link=S&tipo=RDC&numeroAto=00000153&seqAto=000&valorAno=2017&orgao=RDC/DC/ANVISA/MS&cod_modulo=310&cod_menu=9431
- IN 66/2020 — https://www.contadores.cnt.br/legislacoes/instrucao-normativa-dc-anvisa-no-66-de-01-09-2020.html
- CGSIM 62/2020 e 66/2021 — https://www.gov.br/empresas-e-negocios/pt-br/drei/cgsim/resolucoes-cgsim/arquivos/resolucao-62-2020-revogado-pela-resolucao-66-v2.pdf
- CP Anvisa 1.249/2024 — https://anexosportal.datalegis.net/arquivos/1888063.pdf ; https://www.kvlaw.com.br/anvisa-abre-consulta-publica-visando-a-alteracao-das-diretrizes-de-classificacao-de-risco-para-atividades-economicas-sujeitas-a-vigilancia-sanitaria/
- Anvisa MARP/ROI (2026) — https://www.gov.br/anvisa/pt-br/assuntos/noticias-anvisa/2026/anvisa-lanca-consulta-nacional-para-avaliar-modelo-de-risco-em-inspecoes-de-servicos-de-saude-e-de-interesse-para-saude
- Lei 13.874/2019, art. 3º — https://normas.leg.br/?urn=urn%3Alex%3Abr%3Afederal%3Alei%3A2019-09-20%3B13874%21art3
- Decreto 10.178/2019 — https://www2.camara.leg.br/legin/fed/decret/2019/decreto-10178-18-dezembro-2019-789614-norma-pe.html
- Lei 6.437/1977 (PAS municipal) — https://repositorio.furg.br/items/129043f8-28f5-4341-a757-b3d27623fd81

**Vigilância sanitária — realidade municipal e sistemas**
- Levantamento de licenciamento sanitário municipal — https://www.redalyc.org/journal/5705/570567431010/html/
- SINAVISA/DIVISA-BA (Vigil Sanit Debate 2024) — https://www.redalyc.org/journal/5705/570578445002/570578445002_2.pdf
- SINAVISA-GO — https://goias.gov.br/saude/sinavisa-vigilancia-sanitaria/
- Facilita SP / Redesim — https://www.facilitasp.sp.gov.br/wp-content/uploads/2024/06/ADESAO-A-REDESIM-final-compactado.pdf
- Pactuação: Physis — https://www.scielo.br/j/physis/a/j4P6D5nBmfZ7Mcp4j3DNHpf/?lang=pt ; SC — https://vigilanciasanitaria.saude.sc.gov.br/index.php/servicos/pactuacao.html ; CIB-AM 414/2025 — http://ses.saude.am.gov.br/uploads/storage/cib/docs/res/2025_414_31072025100757.pdf

**Fornecedores privados de VISA**
- IPM — https://www.ipm.com.br/vigilancia/
- Inovadora — https://www.inovadora.com.br/solucao/vigilancia-em-saude/
- Tecnologia.link — https://tecnologia.link/nossos-produtos/software-gestao-vigilancia-sanitaria/
