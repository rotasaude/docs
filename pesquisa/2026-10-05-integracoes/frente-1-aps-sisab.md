# Frente 1 — Produção e financiamento da APS (e-SUS APS, LEDI, SISAB/SIAPS, cofinanciamento)

Pesquisa feita em 2026-10-05. Legenda: **[CONFIRMADO: url]** = lido na fonte citada; **[INFERÊNCIA]** = dedução minha, não escrita na fonte.
Cópias locais das notas técnicas e metodológicas usadas: `_fontes-frente-1/` (ao lado deste arquivo).

> Aviso de contexto: o portal sisaps.saude.gov.br exibe, em outubro de 2026, o aviso de que "alguns conteúdos encontram-se temporariamente indisponíveis" por causa do defeso eleitoral, e algumas notícias do gov.br voltaram "Conteúdo Restrito". Parte da documentação pode estar fora do ar até o fim do período eleitoral. [CONFIRMADO: https://sisaps.saude.gov.br/sistemas/esusaps/]

---

## Resumo executivo

1. **A premissa está correta, mas falta um detalhe.** Um prontuário de terceiro **não envia ao SISAB/SIAPS diretamente**. Ele gera registros no layout **LEDI APS** e os entrega a uma instalação do **PEC e-SUS APS** do município (de unidade ou centralizadora). O PEC transmite ao **Centralizador Nacional**. Desde o PEC 5.3.19 essa entrega pode ser feita **por API HTTPS**, com credencial gerada pelo administrador da instalação; antes, só por arquivo Thrift/XML. **A versão vigente é o LEDI 8.7.0** (13/08/2026), compatível com o PEC 5.5.26 ou superior. [CONFIRMADO: https://integracao.esusaps.bridge.ufsc.tech/ledi/index.html]
2. **O SISAB foi renovado como SIAPS** pela Portaria GM/MS nº 7.639/2025, que é agora o sistema vigente para o financiamento. [CONFIRMADO: https://bvsms.saude.gov.br/bvs/saudelegis/gm/2025/prt7639_22_07_2025.html] Desde 1º/01/2026, dado enviado por versão com mais de 12 meses de publicação é **invalidado**, e isso vale também para terceiros que usam o LEDI. [CONFIRMADO: NT 12/2025, https://sisaps.saude.gov.br/sistemas/esusaps/assets/files/NT_12-2025_criterio_validacao_dados_siaps-0394bed57dc6efcddaa83dab337f9533.pdf]
3. **Vacinação saiu do LEDI.** A Portaria GM/MS 5.663/2024 obriga o envio **exclusivo à RNDS** (hoje RIA-R 2.0) e desativa o LEDI Thrift/XML para vacinas. [CONFIRMADO: https://bvsms.saude.gov.br/bvs/saudelegis/gm/2024/prt5663_04_11_2024.html]
4. **O PEC não tem API de leitura clínica para terceiros.** Existem três mecanismos oficiais: (a) LEDI, só de escrita; (b) o **DW PEC**, estrutura documentada do banco para quem consulta via SQL; (c) **"Sistemas externos"**, um iframe dentro do atendimento do PEC que recebe CPF/CNS do cidadão e do profissional e o CNES, com assinatura HMAC. [CONFIRMADO: https://integracao.esusaps.bridge.ufsc.tech/]
5. **Não achei homologação nem custo do Ministério** para integrar via LEDI. O controle acontece na validação do SIAPS (layout e versão). A RNDS, ao contrário, **tem credenciamento** com homologação e depois produção, certificado ICP-Brasil e gov.br. [CONFIRMADO: https://rnds-guia.saude.gov.br/docs/passo-a-passo/]
6. **Prazo:** competência mensal, enviada até o **10º dia útil do mês seguinte**. Exemplo: a competência set/2026 vai até 15/10/2026. Três competências seguidas sem envio suspendem 100% do componente fixo. [CONFIRMADO: https://sisaps.saude.gov.br/sistemas/siaps/docs/manual/calendario-siaps/ ; https://bvsms.saude.gov.br/bvs/saudelegis/gm/2024/prt3493_11_04_2024.html]
7. **Financiamento (Portaria 3.493/2024 e alterações):** o componente II (Vínculo/Acompanhamento) depende de Cadastro Individual e Cadastro Domiciliar mais contatos assistenciais. O componente III (Qualidade, indicadores C1–C7 de eSF/eAP) depende quase só de **Atendimento Individual (com CID-10/CIAP-2, PA, peso/altura, tipo de demanda), Procedimentos (SIGTAP), Visita Domiciliar do ACS, Vacinação (RNDS) e Odonto**. Se o Rota Saúde for o sistema de registro, **o repasse federal da cidade passa a depender da qualidade do LEDI que ele gerar**.

---

## 1. e-SUS APS — PEC e CDS

| Item | Achado |
|---|---|
| **O que é** | Estratégia da SAPS/MS com software gratuito: **PEC** (Prontuário Eletrônico do Cidadão), **CDS** (Coleta de Dados Simplificada) e aplicativos (Território, Vacinação, Atividade Coletiva, AD, Gestão). [CONFIRMADO: https://sisaps.saude.gov.br/sistemas/esusaps/] A versão atual de download é o PEC **5.5.28**. [CONFIRMADO: mesma URL] |
| **Quem opera** | Desenvolvido pelo Laboratório Bridge/UFSC para a SAPS/MS (a documentação de integração fica em `integracao.esusaps.bridge.ufsc.tech`) [CONFIRMADO: https://integracao.esusaps.bridge.ufsc.tech/]. Quem instala e administra é o município. [CONFIRMADO: https://sisaps.saude.gov.br/sistemas/esusaps/docs/manual/PEC/PEC_01_implantacao/] |
| **Arquiteturas** | Centralizada (uma instalação para várias UBS, a recomendada), descentralizada (uma por unidade) e multimunicipal. Há dois modos de instalação: **Prontuário** e **Centralizador**; o Centralizador só agrega dados de outras instalações. O fluxo é unidade → centralizador municipal (opcional) → **Centralizador Nacional**. [CONFIRMADO: https://sisaps.saude.gov.br/sistemas/esusaps/docs/manual/PEC/PEC_01_implantacao/] |
| **Obrigatório?** | O município precisa enviar ao SIAPS **pelo e-SUS APS ou por sistema próprio/terceiro "de forma compatível ao Siaps"** (art. 308, §1º). Usar o PEC como prontuário não é obrigatório; ter o dado no SIAPS é. [CONFIRMADO: https://bvsms.saude.gov.br/bvs/saudelegis/gm/2025/prt7639_22_07_2025.html] Na prática, um terceiro precisa de **pelo menos uma instalação PEC** (pode ser a centralizadora) para receber o LEDI. [INFERÊNCIA a partir da documentação do LEDI e da API] |
| **Padrão técnico** | LEDI APS (seção 2). |
| **Custo** | O software é gratuito para download. [CONFIRMADO: https://sisaps.saude.gov.br/sistemas/esusaps/] O município paga a infraestrutura (servidor, HTTPS). [INFERÊNCIA] |
| **Atualização** | O suporte e-SUS vale **6 meses** após cada versão. A aceitação de dados no SIAPS vale **12 meses**. [CONFIRMADO: NI 13/2025, https://sisaps.saude.gov.br/sistemas/esusaps/assets/files/NI_13-2025_cenario_versoes_incompativeis-90647909abe17697641f1a44b859e48a.pdf] |
| **Módulos do Rota Saúde que dependem** | Todos os do modo "convive". No modo "substitui", o PEC continua necessário como **porta de entrada do LEDI**. |

## 2. LEDI APS — Layout e-SUS APS de Dados e Interface

**O que é:** "camada abstrata que especifica as informações, e seus formatos, que são aceitos no envio de dados de sistemas próprios para o PEC e-SUS APS". Pode ser implementada em **XML ou Apache Thrift** e tem versionamento independente do PEC. [CONFIRMADO: https://integracao.esusaps.bridge.ufsc.tech/ledi/index.html]

**Versão vigente:** **LEDI 8.7.0**, liberada em 13/08/2026 e compatível com o PEC ≥ 5.5.26. As anteriores recentes são 8.5.0 (16/07/2026, PEC ≥ 5.5.23), 8.4.2 (02/07/2026) e 7.4.2 (14/05/2026). [CONFIRMADO: https://integracao.esusaps.bridge.ufsc.tech/ledi/index.html] O ritmo é de **uma versão a cada 2–6 semanas**. Há mudanças relevantes recentes: **CPF como identificador principal** e campos de dispensa de CPF (8.4.0/8.5.0/8.7.0), CNPJ alfanumérico (8.5.0), tabela SIGTAP atualizada mês a mês e procedimentos "AB" inativados em favor de códigos SIGTAP (8.4.0). [CONFIRMADO: https://integracao.esusaps.bridge.ufsc.tech/ledi/documentacao/principais_alteracoes.html]

**Formas de entrega ao PEC:**
- **API de transmissão** (PEC ≥ 5.3.19). A instalação precisa ter HTTPS. O administrador gera a credencial em "Integração > Credenciais de integração", com dados de pessoa física ou jurídica do integrador. O fluxo é `POST [url]/api/recebimento/login` (usuario/senha → cookie `JSESSIONID`) e depois `POST [url]/api/v1/recebimento/ficha` (arquivo binário `<uuidDadoSerializado>.esus`). A resposta é 200 ou 400/500 com mensagem de validação. A credencial não pode ser recuperada; se for perdida, é preciso gerar outra. Os exemplos oficiais estão em Java, no GitHub. [CONFIRMADO: https://integracao.esusaps.bridge.ufsc.tech/ledi/documentacao/api_transmissao/api_transmissao_registro_LEDI.html ; https://sisaps.saude.gov.br/sistemas/esusaps/docs/manual/APOIO/API_transmissao/]
- **Arquivo/lote** Thrift (`.esus`, serializado em TBinaryProtocol e compactado) ou XML, importado na instalação. [CONFIRMADO: https://integracao.esusaps.bridge.ufsc.tech/ledi/documentacao/estrutura_arquivos/index.html]

**Camada de transporte (DadoTransporte), campos que afetam o modelo de dados do Rota Saúde** [CONFIRMADO: https://integracao.esusaps.bridge.ufsc.tech/ledi/documentacao/estrutura_arquivos/camada-transporte.html]:
- `uuidDadoSerializado` (recomenda-se CNES + "-" + UUID, até 44 caracteres), `tipoDadoSerializado`, `cnesDadoSerializado` (só CNES do município), `codIbge` (precisa estar ativo na instalação), `ineDadoSerializado` (opcional no transporte; só INE do município), `numLote`.
- `remetente` e `originadora` (DadoInstalacao): `contraChave` = "<Nome do software> - <versão>", `uuidInstalacao`, `cpfOuCnpj`, `nomeOuRazaoSocial`, e-mail e telefone. **O Rota Saúde se identifica aqui como software de terceiro.**

**Cabeçalho do profissional (headerTransport)** [CONFIRMADO: https://integracao.esusaps.bridge.ufsc.tech/ledi/documentacao/estrutura_arquivos/header-transport.html]: `profissionalCNS` (15 dígitos, algoritmo válido, **profissional com vínculo no município**), `cboCodigo_2002` (só os CBOs aceitos em cada modelo), `cnes`, `ine`, `dataAtendimento` (epoch em ms, nunca futura) e `codigoIbgeMunicipio`. Para atendimento compartilhado há `VariasLotacoesHeader`, com lotação principal e lotação de atendimento compartilhado.

**Modelos de informação (fichas) existentes no LEDI 8.7.0** [CONFIRMADO: https://integracao.esusaps.bridge.ufsc.tech/ledi/documentacao/estrutura_arquivos/index.html ; https://integracao.esusaps.bridge.ufsc.tech/ledi/documentacao/thrift-xsd.html]:

| Ficha (sigla nas notas MS) | Conteúdo-chave | Usada no financiamento? |
|---|---|---|
| Cadastro Individual (MICI) | Identificação, sociodemográfico, condições de saúde, situação de rua, saída do cadastro | **Sim**: componente II, dimensão Cadastro (NT 30/2025) |
| Cadastro Domiciliar e Territorial (MICDT) | Domicílio, família, condições de moradia, instituição de permanência | **Sim**: o fator 1,5 vale se MICI + MICDT estiverem atualizados em 24 meses (NT 30/2025) |
| Atendimento Individual (MIAI) | CNS/CPF do cidadão, tipo de atendimento (demanda programada/espontânea), medições (PA, peso, altura), DUM/IG, problemas/condições (**CIAP-2/CID-10**, situação ativo/resolvido), exames solicitados/avaliados (SIGTAP), resultados de exames, **medicamentos (CATMAT, posologia)**, encaminhamentos (especialidade, CID/CIAP, risco), IVCF-20, solicitações de OCI, condutas, horário de início e fim | **Sim**: núcleo de C1–C7 e do acompanhamento |
| Atendimento Odontológico Individual (MIAOI) | Procedimentos, medicamentos, encaminhamentos, problemas | **Sim**: B1–B6 (eSB) e C3 (K) |
| Atividade Coletiva (MIAC) | Tema, público, participantes, profissionais (CBO) | **Sim**: C3, C4 e indicadores de eSB (escovação) e eMulti |
| Procedimentos (MIP) | Procedimentos SIGTAP por cidadão | **Sim**: C2–C7 (aferição de PA, antropometria, pé diabético, coleta citopatológico etc.) |
| Visita Domiciliar e Territorial (MIVDT) | Visita de ACS/TACS com "motivo da visita" | **Sim**: C2–C6 e acompanhamento |
| Marcadores de Consumo Alimentar (MIMCA) | Questionários por faixa etária | Conta como contato no acompanhamento (NT 30/2025) |
| Avaliação de Elegibilidade (Atenção Domiciliar) | Elegibilidade para AD, campos de dispensa de CPF (8.7.0) | Não aparece em C1–C7 [INFERÊNCIA] |
| Atendimento Domiciliar (AD) | Atendimento das equipes EMAD/EMAP | Não aparece em C1–C7 [INFERÊNCIA] |
| Complementar Zika/Microcefalia | Ficha complementar | Não aparece em C1–C7 [INFERÊNCIA] |
| Vacinação (MIV) | Imunobiológico, dose, lote, fabricante (código RNDS) | Usada em C2, C3, C6, C7 e no acompanhamento, mas **o envio de terceiros deve ir para a RNDS (RIA)**, ver seção 4 |
| Cuidado Compartilhado | Evolução de compartilhamento de cuidado (solicitante/executante) | Provável uso em eMulti [INFERÊNCIA] |

## 3. SISAB → SIAPS (base nacional)

| Item | Achado |
|---|---|
| **O que é** | O **SIAPS** (Sistema de Informação para a Atenção Primária à Saúde) foi instituído pela **Portaria GM/MS nº 7.639, de 18/07/2025**, como "sistema vigente para financiamento e adesão aos programas da PNAB" (art. 305). É operado pela Estratégia e-SUS APS (art. 306). [CONFIRMADO: https://bvsms.saude.gov.br/bvs/saudelegis/gm/2025/prt7639_22_07_2025.html] Os relatórios públicos e o relatório de validação ainda ficam no domínio sisab.saude.gov.br. [CONFIRMADO: NT 12/2025] |
| **Terceiros** | "No caso do DF e dos Municípios que utilizam sistemas de informação próprios ou de terceiros, as informações deverão ser enviadas de forma compatível ao Siaps" (art. 308, §1º). [CONFIRMADO: idem] Não achei endpoint público para terceiros enviarem direto ao SIAPS; o canal documentado é LEDI → PEC → Centralizador Nacional. [INFERÊNCIA a partir da ausência na documentação de integração] |
| **Validação** | Só dado **validado (aprovado)** aparece no SIAPS. Há um Relatório de Validação (público e restrito). Desde **1º/01/2026**, dado gerado por versão com mais de 12 meses é reprovado com o motivo "versão do sistema disponibilizada por período superior a 12 meses". Isso vale também para quem integra via LEDI. [CONFIRMADO: NT 12/2025 (link no resumo) e NI 13/2025] |
| **Prazo** | Competência mensal, enviada **até o 10º dia útil do mês seguinte**. Calendário 2026: jan → 13/02, fev → 13/03, mar → 15/04, abr → 15/05, mai → 16/06, jun → 14/07, jul → 14/08, ago → 15/09, **set → 15/10/2026**, out → 16/11, nov → 14/12, dez → 15/01/2027. [CONFIRMADO: https://sisaps.saude.gov.br/sistemas/siaps/docs/manual/calendario-siaps/] A NT 8/2026 repete que só conta para os indicadores o que for enviado até o 10º dia do mês seguinte. [CONFIRMADO: https://www.gov.br/saude/pt-br/centrais-de-conteudo/publicacoes/notas-tecnicas/2026/nota-tecnica-no-8-2026-deaps-saps-ms.pdf/@@download/file] |
| **Extração para indicadores** | No 20º dia útil de cada mês, a partir do SIAPS e da última competência válida do SCNES. [CONFIRMADO: Nota Metodológica C4, https://www.gov.br/saude/pt-br/composicao/saps/publicacoes/fichas-tecnicas/equipe-de-atencao-primaria-e-saude-da-familia/nota-metodologica-c4-cuidado-da-pessoa-com-diabetes] |
| **Credenciamento** | Para o município: o gestor e-SUS é cadastrado no e-Gestor AB. [CONFIRMADO via resultado de busca que cita o manual de instalação; INFERÊNCIA quanto à vigência] Para o software de terceiro: **não achei homologação nem certificação federal**. O controle é a conformidade de layout e versão. [INFERÊNCIA; não achei texto oficial que diga "não homologa"] |
| **Custo** | Não achei tarifa. [INFERÊNCIA] |
| **Módulos do Rota Saúde que dependem** | Analytics (indicadores C1–C7 e painéis de vínculo), cadastro do cidadão, território, agendamento (tipo de demanda), consulta e prontuário, procedimentos, visita domiciliar, atividade coletiva e campanhas. |

## 4. RNDS para vacinação (interface com esta frente)

- A **Portaria GM/MS 5.663/2024** estabelece que a vacinação só pode ser registrada no PEC, no CDS ou em sistema próprio/terceiro **integrado à RNDS** no modelo RIA vigente. Prazo de **24 h** onde há conectividade e de **15 dias** sem conectividade. O **LEDI Thrift/XML fica desativado para vacinas**. [CONFIRMADO: https://bvsms.saude.gov.br/bvs/saudelegis/gm/2024/prt5663_04_11_2024.html]
- Comunicado DPNI de 23/02/2026: o RIA vigente é o **RIA-R 2.0**, que deve ser usado inclusive para campanha. RIA Carga, RIA-R 1.0 e RIA-C 2.0 serão desativados por cronograma; quem enviar a outros profiles "poderá ser prejudicado nos seus indicadores". [CONFIRMADO: https://sbim.org.br/images/SEI_MS-0053610638-Comunicado.pdf_2026-02-26.pdf (cópia de documento SEI/MS)]
- As notas C2, C3, C6 e C7 contam vacinação pelo **MIV** (e-SUS) **e pelo RIA (RNDS)**. [CONFIRMADO: notas metodológicas C2/C3/C6/C7, em `_fontes-frente-1/`]
- O credenciamento na RNDS tem duas fases (homologação e produção), exige certificado ICP-Brasil (e-CNPJ/e-CPF) e conta gov.br, e é solicitado pelo **estabelecimento/gestor**; a software house desenvolve e testa. Não achei custo. [CONFIRMADO: https://rnds-guia.saude.gov.br/docs/passo-a-passo/]
- Contradição a acompanhar: o LEDI 8.4.x ainda publica "regras de vacina" e "fabricante via código RNDS" no MIV. [CONFIRMADO: https://integracao.esusaps.bridge.ufsc.tech/ledi/documentacao/principais_alteracoes.html] Pode ser que o MIV siga servindo ao PEC e aos apps oficiais, ou a algum caso residual. [INFERÊNCIA; pergunta em aberto]
- Para o Rota Saúde: o módulo **Campanhas** e uma futura sala de vacina precisam de **integração RNDS**, não de LEDI. [INFERÊNCIA]

## 5. Cofinanciamento federal da APS (modelo que substituiu o Previne Brasil)

**Base legal:** a **Portaria GM/MS nº 3.493, de 10/04/2024** instituiu a nova metodologia do Piso da APS. [CONFIRMADO: https://bvsms.saude.gov.br/bvs/saudelegis/gm/2024/prt3493_11_04_2024.html] Ela foi alterada pelas **Portarias GM/MS 6.907/2025 e 7.799/2025** e regulamentada pela **Portaria SAPS/MS 161/2024** (metodologia do componente II), pela **NT 30/2025-CGESCO/DESCO/SAPS/MS** (vínculo) e pela **NT 8/2026-DEAPS/SAPS/MS** (cálculo quadrimestral de II e III, que revoga a NT 6/2025). [CONFIRMADO: lista de referências da NT 8/2026, link acima]

**Componentes** (resumo da 3.493 consolidada) [CONFIRMADO: https://bvsms.saude.gov.br/bvs/saudelegis/gm/2024/prt3493_11_04_2024.html]: fixo (eSF/eAP por estrato), **vínculo e acompanhamento territorial**, **qualidade**, implantação, programas específicos (eSB, eMulti, ACS etc.) e per capita de base populacional (IBGE). Os componentes II e III são recalculados a cada quadrimestre e pagos nos quatro meses seguintes. [CONFIRMADO: NT 8/2026] Os valores em reais vêm do texto consolidado e podem ter mudado com as portarias de 2025; não os usar sem conferir. [INFERÊNCIA]

**Suspensão por falta de dado:** suspensão **total** depois de **3 competências consecutivas sem envio de produção** ao SISAB/SIAPS. Há suspensões proporcionais por ausência de categorias profissionais no SCNES. [CONFIRMADO: Portaria 3.493 consolidada, art. 12-K]

**Componente II — Vínculo e Acompanhamento Territorial** [CONFIRMADO: NT 30/2025, https://www.gov.br/saude/pt-br/centrais-de-conteudo/publicacoes/notas-tecnicas/2025/nota-tecnica-no-30-2025-cgesco-desco-saps-ms.pdf]:
- **Dimensão Cadastro (30%)**: só conta o **Cadastro Individual (MICI)** que passa na validação do SIAPS, incluído ou atualizado nos últimos **24 meses**. O cadastro rápido do módulo Cidadão **não conta**. O fator é 0,75 só com MICI e **1,5 com MICI + MICDT** atualizados.
- **Dimensão Acompanhamento (70%)**: pessoa com **mais de um contato** em 12 meses, sendo ao menos um uma prática de cuidado (MIAI, MIAOI, MIV, MIMCA, MIVDT etc.). Pesos: 1,2 para idoso/criança e 1,3 para beneficiário de PBF/BPC.
- **Bônus de satisfação**: avaliação do atendimento pelo cidadão no app **Meu SUS Digital**.

**Componente III — Qualidade (eSF/eAP)**: indicadores C1–C7 com pesos 1/2/2/1/1/1/2 (total 10), conceitos Ótimo/Bom/Suficiente/Regular, granularidade por **INE**. [CONFIRMADO: NT 8/2026] Indicadores próprios para eSB (B1–B6) e eMulti (M1–M2). [CONFIRMADO: NT 8/2026]

## 6. Indicadores que dependem de dado que o Rota Saúde produziria

Fonte das linhas abaixo: notas metodológicas C1–C7 (SAPS/MS, 2025), índice em https://www.gov.br/saude/pt-br/composicao/saps/publicacoes/fichas-tecnicas/equipe-de-atencao-primaria-e-saude-da-familia (cópias em `_fontes-frente-1/`). [CONFIRMADO]

| Indicador | Boas práticas (resumo) | Dados que o Rota Saúde precisa gerar | Módulo do Rota Saúde |
|---|---|---|---|
| **C1 Mais acesso** | % de atendimentos de demanda **programada** (consulta agendada programada, cuidado continuado, consulta agendada) sobre o total (programada + espontânea: escuta inicial/orientação, consulta no dia, urgência). Só médicos e enfermeiros (CBOs listados) | MIAI com `tipoAtendimento` correto | **Agendamento e fila/triagem**: a distinção agendado × espontâneo nasce aqui. Triagem/escuta inicial conta como demanda espontânea |
| **C2 Desenvolvimento infantil** (peso 2) | 1ª consulta até o 30º dia; ≥9 consultas e ≥9 registros de peso+altura até 2 anos; 2 visitas do ACS (até 30 dias e até 6 meses); vacinas (penta, VIP, SCR, pneumo) | MIAI, MIP (antropometria e puericultura), MIVDT com motivo "recém-nascido/criança", MIV/RIA | Consulta, linhas de cuidado, visita domiciliar, vacina via RNDS |
| **C3 Gestação e puerpério** (peso 2) | 1ª consulta até 12 semanas; ≥7 consultas, ≥7 PA, ≥7 peso+altura; 3 visitas do ACS; dTpa a partir de 20 semanas; testes de sífilis/HIV/hepatites no 1º e 3º trimestres; consulta e visita no puerpério; atividade de saúde bucal | MIAI com DUM/IG e CID/CIAP de gestação, MIP (PA, testes rápidos, consulta pré-natal), MIVDT, MIV/RIA, MIAOI, MIAC | Consulta (SOAP com CIAP/CID), exames, linha de cuidado da gestante |
| **C4 Diabetes** | Consulta em 6 meses; PA em 6 meses; peso+altura em 12 meses; 2 visitas do ACS em 12 meses; HbA1c solicitada ou avaliada em 12 meses; avaliação dos pés em 12 meses. Elegibilidade: CIAP T89/T90, CID E10/E11/E14 **ativos** | MIAI (problema ativo, exame solicitado/avaliado), MIP (03.01.04.009-5 pé diabético, 02.02.01.050-3, ABEX008) | Consulta, exames, linha de cuidado |
| **C5 Hipertensão** | Consulta e PA em 6 meses; peso+altura em 12 meses; 2 visitas do ACS em 12 meses (25 pontos cada). Elegibilidade: CIAP K86/K87, CID I10–I15 | MIAI, MIP, MIVDT | Idem |
| **C6 Pessoa idosa** | Consulta em 12 meses; peso+altura; 2 visitas do ACS; influenza em 12 meses | MIAI, MIP, MIVDT, MIV/RIA | Consulta, visita, campanhas (RNDS) |
| **C7 Mulher / prevenção de câncer** (peso 2) | Rastreamento de colo (25–64 anos, 36 meses; HPV molecular 60 meses); HPV em meninas de 9–14 anos; atendimento de saúde sexual e reprodutiva (14–69 anos) em 12 meses; mamografia solicitada ou avaliada (50–69 anos, 24 meses) | MIAI (exames solicitados/avaliados, CIAP/CID), MIP, RIA | Consulta, exames |

Regras comuns a todos [CONFIRMADO: notas C1–C7]: o cidadão é identificado por **nome, data de nascimento e CPF ou CNS válido conforme o CadSUS**; só contam equipes **eSF (tipo 70) e eAP (tipo 76)**; o registro precisa de **CNS do profissional** e de **CBO dentro dos grupos listados**; a vinculação da pessoa segue a NT 30/2025, com desempate pela Portaria SAPS 161/2024; o acompanhamento é interrompido por "saída do cadastro: mudança de território" ou por óbito no CadSUS. **Problema marcado como "resolvido" tira a pessoa do denominador** (C4/C5).

## 7. Identificadores exigidos

| Identificador | Onde é exigido | Fonte |
|---|---|---|
| **CNES** (7 dígitos) | Transporte e cabeçalho; precisa ser do município | [CONFIRMADO: camada-transporte e header-transport] |
| **INE** (10 dígitos) | Opcional no transporte, mas é a **granularidade de todos os indicadores** e o critério de vínculo; precisa ser do município | [CONFIRMADO: idem; NT 8/2026; notas C1–C7] |
| **Código IBGE** (7) | Transporte e cabeçalho; precisa estar ativo na instalação PEC | [CONFIRMADO: camada-transporte] |
| **CNS do profissional** (15) | Obrigatório no cabeçalho; algoritmo válido; vínculo no município (SCNES) | [CONFIRMADO: header-transport] |
| **CPF do profissional** | Obrigatório em `LotacaoThrift` (cuidado compartilhado); precisa existir na base | [CONFIRMADO: header-transport] |
| **CBO 2002** | Obrigatório; só os CBOs aceitos por modelo. Os indicadores usam grupos (2235, 2231/2251/2252/2253, 5151-05 etc.) | [CONFIRMADO: header-transport; notas C1–C7] |
| **CPF/CNS do cidadão** | MIAI tem `cnsCidadao` e `cpfCidadao`; desde o LEDI 8.4.0 o **CPF é o identificador primário**, com campos `stCidadaoNaoPossuiCpf` e justificativa | [CONFIRMADO: dicionário FAI; principais_alteracoes] |
| **Tipo de equipe** | 70 (eSF) e 76 (eAP) para C1–C7 | [CONFIRMADO: notas C1–C7] |
| **Software/instalação** | `contraChave` ("Nome – versão"), `uuidInstalacao`, CPF/CNPJ do responsável | [CONFIRMADO: camada-transporte] |

Terminologias: **CIAP-2, CID-10, SIGTAP** (atualizado mês a mês), **CATMAT** (medicamentos no MIAI) e código de fabricante RNDS (vacina). [CONFIRMADO: dicionário FAI; principais_alteracoes]

## 8. Respostas diretas às 6 perguntas

1. **Como o terceiro envia ao SISAB/SIAPS?** Gera LEDI e entrega a uma instalação PEC do município, por **API** (PEC ≥ 5.3.19) ou por arquivo. O PEC repassa ao Centralizador Nacional. **A premissa está confirmada**, com um ajuste: o destino pode ser **qualquer** instalação PEC (de unidade ou centralizadora), não necessariamente "o centralizador". **Não achei envio direto ao SIAPS** documentado para terceiros. **Versão vigente: LEDI 8.7.0.**
2. **Fichas:** ver tabela da seção 2. Para o financiamento são essenciais: **Cadastro Individual, Cadastro Domiciliar, Atendimento Individual, Procedimentos, Visita Domiciliar, Atendimento Odontológico, Atividade Coletiva**, e **Vacinação via RNDS**. Consumo Alimentar conta no acompanhamento.
3. **API de leitura no PEC?** **Não há API de leitura clínica.** As opções oficiais são: **DW PEC** (esquema documentado de fatos, dimensões e visualizações, como `tb_fat_cidadao_pec`, `tb_fat_rel_op_gestante`, `tb_fat_rel_op_risco_cardio` e `tb_acomp_cidadaos_vinculados`), voltado para "profissionais com conhecimento em banco de dados", ou seja, **acesso SQL ao banco da instalação** [CONFIRMADO: https://integracao.esusaps.bridge.ufsc.tech/dw/index.html]; e **Sistemas externos** (iframe na aba do atendimento ou no menu, com `documentoCidadao` (CPF ou CNS), `documentoProfissional` e `CNES`, assinado com HMAC-SHA256 e `pec_timestamp`) [CONFIRMADO: https://integracao.esusaps.bridge.ufsc.tech/sistemas_externos/parametros_dinamicos.html ; .../chave_assinatura.html]. **Importar dado de terceiro no PEC** = LEDI. O dado vira registro de produção, mas não sei se aparece no "Histórico" clínico do cidadão do mesmo jeito que um atendimento do PEC. [pergunta em aberto]
4. **Homologação/custo/prazo:** não achei homologação nem custo federal para LEDI [INFERÊNCIA]. A exigência é conformidade com a versão (12 meses). Prazo: 10º dia útil do mês seguinte à competência. A RNDS (vacina) **exige** credenciamento em duas fases.
5. **Indicadores:** C1–C7 (seção 6), as dimensões Cadastro e Acompanhamento do componente II, e B1–B6/M1–M2 se o Rota Saúde registrar odonto e eMulti.
6. **Identificadores:** INE, CNES, IBGE, CNS e CPF do profissional, CBO, CPF/CNS do cidadão validado no CadSUS, tipo de equipe e identificação do software (seção 7).

---

## Implicações para o "modo de prontuário por cidade"

### Modo REGISTRO (Rota Saúde substitui o PEC)
- O Rota Saúde vira **produtor de LEDI**: precisa gerar MICI, MICDT, MIAI, MIAOI, MIP, MIVDT, MIAC e MIMCA com UUID estável (CNES-UUID), e reenviar quando o registro é corrigido. [INFERÊNCIA]
- **Ainda é preciso um PEC** na cidade (pode ser só o centralizador) com HTTPS e credencial de integração. O Rota Saúde depende do município manter esse PEC atualizado: se a versão do PEC ficar velha, a cidade perde dados no SIAPS mesmo com o LEDI certo. [INFERÊNCIA a partir da NT 12/2025 e da tabela de compatibilidade]
- **Esteira de versões:** o LEDI muda a cada 2–6 semanas e a aceitação dura 12 meses. Isso exige um módulo de conformidade versionado (Thrift gerado dos IDLs oficiais), uma suíte de contrato por versão e um painel de rejeições (resposta 400 da API + Relatório de Validação do SIAPS). [INFERÊNCIA]
- **O modelo clínico tem de nascer compatível:** problemas/condições com CIAP-2/CID-10 e **situação ativo/resolvido** (resolver um problema tira a pessoa do C4/C5); medições estruturadas (PA, peso, altura no mesmo dia); DUM/IG/desfecho da gestação; exames **solicitados × avaliados** com código SIGTAP; medicamentos em CATMAT; tipo de atendimento (programada × espontânea) amarrado ao agendamento. [CONFIRMADO para os campos; INFERÊNCIA para o desenho]
- **Cadastro:** o cadastro rápido não conta para o componente II. O Rota Saúde precisa do **Cadastro Individual completo** e do **Domiciliar/Territorial**, com atualização a cada 24 meses (alerta de vencimento). Isso liga direto ao módulo **Território**. [CONFIRMADO: NT 30/2025]
- **Vacina** não passa pelo LEDI: o Rota Saúde precisa de integração **RNDS (RIA-R 2.0)** com credenciamento do estabelecimento. [CONFIRMADO]
- **Risco financeiro transferido:** um erro de geração pode derrubar o conceito de C1–C7 ou zerar o envio, e três competências sem envio suspendem o componente. Isso pede contrato/SLA com a prefeitura e monitoramento até o 10º dia útil. [INFERÊNCIA]

### Modo INTEGRADO (convive com o PEC)
- A consulta acontece no PEC, que já envia ao SIAPS. O Rota Saúde **não deve enviar MIAI do mesmo atendimento** (risco de duplicidade). [INFERÊNCIA]
- O Rota Saúde pode enviar via LEDI **o que o PEC não registra no fluxo dele**, como a escuta inicial feita na triagem do Rota Saúde, se for registrada como atendimento individual. Isso pede uma regra clara por cidade para evitar dupla contagem. [INFERÊNCIA; pergunta em aberto]
- **Leitura do PEC**: não há API. As opções são (a) **SQL no DW PEC** (o município precisa liberar rede e usuário de banco; o esquema muda com as versões); (b) **iframe "Sistemas externos"**, em que o profissional abre a tela do Rota Saúde dentro do atendimento do PEC já com cidadão e profissional identificados. Isso resolve o contexto, mas não traz dado de volta. (c) Ler da **RNDS** (se o PEC da cidade estiver integrado), assunto de outra frente. [CONFIRMADO para os mecanismos; INFERÊNCIA para a avaliação]
- **Fila e agenda × C1:** se a agenda for do Rota Saúde e a consulta for no PEC, o "tipo de demanda" é marcado no PEC pelo profissional. A informação de agendamento do Rota Saúde não chega ao C1 sozinha. Vale estudar se o iframe ou o DW permitem reconciliar. [INFERÊNCIA]
- **Analytics** do Rota Saúde nesse modo só reproduz C1–C7 se ler o DW PEC (ou o SIAPS, onde o gestor tem acesso restrito). [INFERÊNCIA]

## Riscos

1. **Cadência de versões do LEDI** (8 versões entre jun/2025 e ago/2026) somada à invalidação após 12 meses gera custo contínuo de manutenção. [CONFIRMADO: tabela de compatibilidade; NT 12/2025]
2. **Dependência do PEC do município** como porta de entrada, mesmo no modo registro. Se a instalação estiver sem HTTPS ou desatualizada, nada chega ao SIAPS. [INFERÊNCIA]
3. **Dupla contagem ou omissão** no modo integrado. [INFERÊNCIA]
4. **Dado clínico mal estruturado = perda de repasse:** CIAP/CID ausente, problema "resolvido" por engano, PA e peso fora do campo estruturado, CBO fora do grupo, profissional sem CNS ou sem vínculo no SCNES. [CONFIRMADO: regras das notas C1–C7]
5. **Vacina:** sem RNDS o Rota Saúde não pode registrar dose, e a cidade perde pontos em C2/C3/C6/C7. [CONFIRMADO: Portaria 5.663/2024; notas]
6. **Acesso SQL ao DW PEC** é frágil (o esquema muda; há implicações de LGPD e segurança no acesso direto ao banco municipal). [INFERÊNCIA]
7. **Normas em movimento:** a NT 8/2026 já revogou a NT 6/2025, e as portarias de 2025 alteraram a 3.493. Os indicadores podem mudar de um quadrimestre para o outro. [CONFIRMADO]
8. **Defeso eleitoral:** parte da documentação oficial está fora do ar em out/2026. [CONFIRMADO]

## Perguntas em aberto

1. Existe algum canal **direto ao Centralizador Nacional/SIAPS** para terceiros, sem PEC? Não achei. Vale confirmar no Webatendimento SAPS (https://webatendimento.saude.gov.br/faq/saps).
2. O Ministério **homologa ou certifica** prontuários de terceiros (fora da RNDS)? Não achei norma afirmando nem negando. A Portaria 1.355/2022 (piloto UBS Digital) exige "padrões de interoperabilidade do DATASUS, CadSUS e SCNES, RNDS e gov.br", mas só para o piloto.
3. Registros recebidos via LEDI aparecem no **prontuário/histórico clínico** do cidadão no PEC, ou só na produção e nos relatórios? Isso importa para a continuidade do cuidado nas cidades que trocam de modo.
4. O **MIV (vacinação) no LEDI** ainda é aceito de terceiros em algum caso, dada a desativação pela Portaria 5.663/2024 e as atualizações de vacina no LEDI 8.4.x?
5. Qual o **número de série e a semântica** de reenvio/correção (mesmo `uuidDadoSerializado` substitui? como cancelar uma ficha enviada por engano)? Ver em `/ledi/documentacao/regras/`, que não li.
6. No modo integrado, a **escuta inicial** feita no Rota Saúde deve ser enviada como MIAI pelo Rota Saúde ou digitada no PEC? Qual o efeito no C1?
7. O módulo 10 do Rota Saúde (profissionais/CBO) já guarda **CNS e CPF do profissional, INE da equipe e tipo de equipe (70/76)**? Sem isso não dá para gerar o LEDI nem calcular indicadores por INE.
8. O cadastro de cidadão do Rota Saúde valida **CPF/CNS contra o CadSUS**? As notas exigem "válido conforme CadSUS". Pode ser assunto da frente CadSUS/RNDS.
9. Quais **valores em R$** vigentes por componente depois das Portarias 6.907/2025 e 7.799/2025? Não conferi o texto consolidado atual.
10. O bônus de **satisfação via Meu SUS Digital** depende de o atendimento estar na RNDS ou no SIAPS? Se depender da RNDS, o modo registro também precisa enviar RAC à RNDS.
11. O DW PEC é acessível em instalações **centralizadas/multimunicipais** geridas por terceiros (consórcios), e com qual governança e LGPD?

## Fontes

- Integração e-SUS APS (portal): https://integracao.esusaps.bridge.ufsc.tech/
- LEDI — compatibilidade e versões: https://integracao.esusaps.bridge.ufsc.tech/ledi/index.html
- LEDI — principais alterações: https://integracao.esusaps.bridge.ufsc.tech/ledi/documentacao/principais_alteracoes.html
- LEDI — estrutura dos arquivos: https://integracao.esusaps.bridge.ufsc.tech/ledi/documentacao/estrutura_arquivos/index.html
- LEDI — camada de transporte: https://integracao.esusaps.bridge.ufsc.tech/ledi/documentacao/estrutura_arquivos/camada-transporte.html
- LEDI — cabeçalho: https://integracao.esusaps.bridge.ufsc.tech/ledi/documentacao/estrutura_arquivos/header-transport.html
- LEDI — Thrift/XSD por modelo: https://integracao.esusaps.bridge.ufsc.tech/ledi/documentacao/thrift-xsd.html
- LEDI — dicionário FAI: https://integracao.esusaps.bridge.ufsc.tech/ledi/documentacao/estrutura_arquivos/dicionario-fai.html
- LEDI — API de transmissão: https://integracao.esusaps.bridge.ufsc.tech/ledi/documentacao/api_transmissao/api_transmissao_registro_LEDI.html e https://sisaps.saude.gov.br/sistemas/esusaps/docs/manual/APOIO/API_transmissao/
- DW PEC: https://integracao.esusaps.bridge.ufsc.tech/dw/index.html
- Sistemas externos (iframe, HMAC, parâmetros): https://integracao.esusaps.bridge.ufsc.tech/sistemas_externos/index.html
- Manual PEC — implantação: https://sisaps.saude.gov.br/sistemas/esusaps/docs/manual/PEC/PEC_01_implantacao/
- e-SUS APS (portal, versão 5.5.28): https://sisaps.saude.gov.br/sistemas/esusaps/
- SIAPS: https://sisaps.saude.gov.br/sistemas/siaps/ ; calendário: https://sisaps.saude.gov.br/sistemas/siaps/docs/manual/calendario-siaps/
- Portaria GM/MS 7.639/2025 (SIAPS): https://bvsms.saude.gov.br/bvs/saudelegis/gm/2025/prt7639_22_07_2025.html
- NT 12/2025-CGIAD/DEAPS/SAPS/MS: https://sisaps.saude.gov.br/sistemas/esusaps/assets/files/NT_12-2025_criterio_validacao_dados_siaps-0394bed57dc6efcddaa83dab337f9533.pdf
- NI 13/2025-CGIAD/DEAPS/SAPS/MS: https://sisaps.saude.gov.br/sistemas/esusaps/assets/files/NI_13-2025_cenario_versoes_incompativeis-90647909abe17697641f1a44b859e48a.pdf
- Portaria GM/MS 3.493/2024: https://bvsms.saude.gov.br/bvs/saudelegis/gm/2024/prt3493_11_04_2024.html
- NT 8/2026-DEAPS/SAPS/MS: https://www.gov.br/saude/pt-br/centrais-de-conteudo/publicacoes/notas-tecnicas/2026/nota-tecnica-no-8-2026-deaps-saps-ms.pdf/@@download/file
- NT 30/2025-CGESCO/DESCO/SAPS/MS: https://www.gov.br/saude/pt-br/centrais-de-conteudo/publicacoes/notas-tecnicas/2025/nota-tecnica-no-30-2025-cgesco-desco-saps-ms.pdf
- Notas metodológicas C1–C7: https://www.gov.br/saude/pt-br/composicao/saps/publicacoes/fichas-tecnicas/equipe-de-atencao-primaria-e-saude-da-familia
- Portaria GM/MS 5.663/2024 (vacinação → RNDS): https://bvsms.saude.gov.br/bvs/saudelegis/gm/2024/prt5663_04_11_2024.html
- Comunicado DPNI 23/02/2026 (RIA-R 2.0), cópia SBIm: https://sbim.org.br/images/SEI_MS-0053610638-Comunicado.pdf_2026-02-26.pdf
- RNDS — passo a passo de credenciamento: https://rnds-guia.saude.gov.br/docs/passo-a-passo/
- Portaria GM/MS 1.355/2022 (piloto UBS Digital): https://bvsms.saude.gov.br/bvs/saudelegis/gm/2022/prt1355_06_06_2022.html
