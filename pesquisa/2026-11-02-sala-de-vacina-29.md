# Pesquisa: módulo 29 (Sala de vacina), 2026-11-02

Legenda: **[C]** confirmado em fonte oficial lida; **[P]** parcial, ou fonte secundária/estadual; **[N]** não confirmado.

## 1. RNDS e RIA

- **[C] Portaria GM/MS nº 5.663, de 31/10/2024** (DOU 04/11/2024). Cria o art. 312-A na PRC 1/2017 e entrou em vigor 120 dias após a publicação, por volta de 03/2025. O que ela fixa:
  - todo sistema de vacinação envia as doses aplicadas **somente à RNDS**, no modelo RIA vigente;
  - o sistema guarda o retorno da integração (id do registro e status de sucesso ou erro);
  - salas com conectividade enviam **em até 24 h**; salas sem conectividade, em até **15 dias**;
  - depois de uma nova orientação técnica de vacinação, o desenvolvedor tem **15 dias para se adequar**. É isso que significa a "adequação em 15 dias": não se trata de prazo de retificação.
  - O art. 2 **desativa a integração LEDI (Thrift/XML) de sistemas próprios ou de terceiros para vacinas**.
  - Fontes: https://bvsms.saude.gov.br/bvs/saudelegis/gm/2024/prt5663_04_11_2024.html e https://www.cosemssp.org.br/wp-content/uploads/2024/11/PORTARIA-No-5.663.pdf
- **[C] NT 115/2024-DPNI** acrescenta requisitos:
  - validar o CPF/CNS do cidadão (PIX/PDQ) antes do envio;
  - validar o CNS do vacinador no CNES;
  - manter o log de integração.
  - Fonte: https://www.cosemssp.org.br/wp-content/uploads/2025/02/RNDS-Nota-Tecnica-115.pdf
- **[C] O modelo vigente é o RIA-R 2.0, que vale também para campanha.** Comunicado DPNI de 23/02/2026 e Ofício-Circular 120/2026/SVSA de 09/03/2026:
  - quem enviar para outros profiles "poderá ser prejudicado nos seus indicadores";
  - haverá um cronograma de desativação do **RIA Carga, do RIA-R 1.0 e do RIA-C 2.0**, ainda não publicado ([N] data).
  - Fontes: https://prefeitura.poa.br/sites/default/files/usu_doc/hotsites/sms/vigilancia-em-saude/SEI_0053610638_Comunicado%20(1).pdf e https://www.prefeitura.poa.br/sites/default/files/usu_doc/hotsites/sms/vigilancia-em-saude/SEI_0053936746_Oficio_Circular_120%20(3).pdf
- **[C] Modelo de informação (Portaria Conjunta SAES/SVSA/SEIDIGI nº 25/2023, anexo, quadro Rotina).** Campos 1..1 (obrigatórios):
  - CNS ou CPF do cidadão;
  - **condição maternal**;
  - CNES;
  - **CNS do profissional** (o CBO só aparece como *especialidade do prescritor*, obrigatória quando a estratégia é "Especial", junto com o CID do motivo);
  - grupo de atendimento;
  - estratégia;
  - data da administração (não pode ser futura);
  - data e hora do registro na origem;
  - imunobiológico;
  - fabricante e lote (**opcionais quando é transcrição**);
  - dose;
  - via;
  - local de aplicação;
  - **flag transcrição de caderneta**.
  - Opcionais: país de origem (na transcrição), comunicante de hanseníase (condicional, para BCG) e pesquisa clínica.
  - **A "transcrição de caderneta" (registro anterior) é o próprio RIA-R com o flag ligado.**
  - Fonte: https://www.conass.org.br/conass-informa-n-180-2023-publicada-a-portaria-conjunta-saes-svsa-seidigi-n-25-que-institui-o-modelo-de-informacao-do-registro-de-imunobiologico-administrado/
- **[P] Modelo computacional FHIR R4.**
  - Um Bundle com Composition `BRRegistroImunobiologicoAdministrado` referencia Immunization `BRImunobiologicoAdministrado`.
  - `vaccineCode` tem binding *required* para o CodeSystem `BRImunobiologico` (canônico `http://www.saude.gov.br/fhir/r4/CodeSystem/BRImunobiologico`).
  - Há também BRDose, BREstrategiaVacinacao (códigos 1 Rotina, 2 Especial, 3 Bloqueio… 13 Reexposição), BRViaAdministracao, BRLocalAplicacao, BRCondicaoMaternal e BRFabricante.
  - O guia oficial (https://rnds-guia.saude.gov.br) é uma SPA: os links de RIA-R e RIA-C aparecem como "#" e não consegui ler o perfil 2.0 lá.
  - Fontes secundárias: https://kyriosdata.github.io/rnds-ig/StructureDefinition-BRRegistroImunobiologicoAdministrado.html e https://fhir.saude.go.gov.br/r4/imunizacao/StructureDefinition-imunobiologico-administrado.html (SES-GO)
  - **Antes de codificar, baixar os StructureDefinitions do RIA-R 2.0 no Portal de Serviços DATASUS.**
- **[P] Autenticação.**
  - Certificado digital **A1 ICP-Brasil** (.pfx) do estabelecimento ou integrador, trocado por um access token no serviço `ehr-auth` (homologação: `ehr-auth-hmg`).
  - O acesso é solicitado pelo gestor (conta gov.br) no Portal de Serviços DATASUS.
  - Fontes: https://www.cosemssc.org.br/wp-content/uploads/2022/09/Integracao-PEC-e-SUS-a-RNDS.pdf e https://www.npmjs.com/package/rnds
  - **[N]** e-CNPJ versus e-CPF, validade do token (30 min) e cabeçalhos exatos.
- **[N] Retificação ou cancelamento.** Nenhum normativo lido define o fluxo. Pelo padrão RNDS, deve ser a substituição do documento (relatesTo/replaces), mas confirmar no guia.

## 2. e-SUS APS e LEDI

- **[C]** O LEDI APS está na **8.7.0** (13/08/2026) e ainda lista o *Modelo de Informação de Vacinação (FV)*. Fonte: https://integracao.esusaps.bridge.ufsc.tech/ledi/documentacao/estrutura_arquivos/index.html
- **[C]** Mesmo assim, pela Portaria 5.663 (art. 2), **sistema próprio não envia vacina pelo LEDI**: o envio é **só pela RNDS**. O LEDI segue valendo para as demais fichas, o que inclui o atendimento em que a vacina é dada.
- **[C] Financiamento da APS (Portaria 3.493/2024, componente qualidade).**
  - O indicador **C2, Cuidado no desenvolvimento infantil** (crianças até 2 anos; nota assinada em 19/06/2026), tem como boa prática (E) as vacinas **penta, VIP, SCR/SCRV e pneumocócica com todas as doses**.
  - Intervalo mínimo de 30 dias entre doses; SCR só conta a partir dos 12 meses.
  - Fontes de dados: **SCNES e RNDS**.
  - "Todos que submeterem o registro ao SIAPS ou à RNDS" contam: o MIV pelo PEC ou o **RIA (vacina ou transcrição)**.
  - Os códigos de vacina são do BRImunobiologico: 09, 17, 22, 24, 26, 29, 39, 42, 43, 46, 47, 56, 58, 59, 106 e 107.
  - Extração no 20º dia útil, avaliação quadrimestral.
  - Fonte: https://www.gov.br/saude/pt-br/composicao/saps/publicacoes/fichas-tecnicas/equipe-de-atencao-primaria-e-saude-da-familia/nota-metodologica-c2-cuidado-no-desenvolvimento-infantil
  - **[N]** Vacina em outros indicadores (gestante, dTpa etc.) não foi verificada.

## 3. SI-PNI e estoque

- **[C] NT 77/2025-CGGI/DPNI** (15/04/2025):
  - **perda física** = frasco fechado (quebra, validade, falha de equipamento, transporte, energia etc.);
  - **perda técnica** = depois da abertura do frasco.
  - Perdas físicas e a **abertura de frasco** ("Saída por Consumo") são lançadas no **SIES** (NFM "Sem Pedido", com 11 motivos e observação obrigatória).
  - A movimentação nas **salas de vacina** é registrada no **SIPNI, funcionalidade "Movimentação dos Imunobiológicos"** (NI Conjunta 4/2023).
  - Fonte: https://www.gov.br/saude/pt-br/vacinacao/notas-tecnicas/nota-tecnica-no-77-2025-cggi-dpni-svsa-ms
- **[C] NT Conjunta 236/2026** (05/08/2026), item 3.3.8: a perda técnica nas salas vai para o **SIPNI ou SIES**; a perda física nas Centrais de Rede de Frio vai para o SIES.
  - Também proíbe condicionar a abertura de multidose a um número mínimo de pessoas ou a agendamento.
  - Fonte: https://www.gov.br/saude/pt-br/centrais-de-conteudo/publicacoes/notas-tecnicas/2026/nota-tecnica-conjunta-no-236.pdf
- **[C]** O SI-PNI **não foi descontinuado**: segue como registrador manual (alimenta a RNDS) e como módulo de movimentação.
- **[N]** Não há API pública documentada para sistemas próprios lançarem movimentação no SIPNI ou SIES. Hoje isso parece ser trabalho manual da sala.
- **[P]** Alguns estados trocaram o SIES por sistemas próprios para pedidos (ES: "Vacina e Confia", 04/2025).

## 4. Calendário

- **[C] Calendário Nacional de Vacinação 2026** é publicado em PDF por ciclo de vida. O PDF da criança cobre 0 a 9a11m29d.
  - Colunas: vacina, proteção, composição, via e volume por laboratório, histórico vacinal, esquema básico, reforços, periódicas, idade/grupo, **intervalo recomendado e mínimo**, mais notas de rodapé com regras de exceção.
  - Exemplo: penta sem HepB ao nascer exige 4 meses entre a D1 e a D3.
  - Fonte: https://www.gov.br/saude/pt-br/composicao/svsa/pni/calendario-tecnico/calendario-tecnico-nacional-de-vacinacao-crianca
  - Os PDFs de adolescente, adulto, idoso e gestante seguem o mesmo padrão ([P]: URLs irmãs não verificadas).
- **[N] Não existe tabela oficial estruturada (CSV/JSON/ImmunizationRecommendation) do calendário.** O que há em FHIR são os CodeSystems de **terminologia** (BRImunobiologico, BRDose, BREstrategiaVacinacao, BRViaAdministracao, BRLocalAplicacao, BRFabricante, BRGrupoAtendimento, BRCondicaoMaternal), que dão para importar.
  - As **regras** de idade e intervalo teriam de ser curadas manualmente a partir do PDF.
  - A SES-GO tem um perfil de "recomendação vacinal" próprio, não nacional: https://fhir-homolog.saude.go.gov.br/r4/imunizacao/StructureDefinition-recomendacao-vacinal-definitions.html

## 5. Regras da sala

- **[C] Multidose: validade após abertura por produto** (NT 236/2026, quadro 1):
  - BCG 6 h (Serum) ou 8 h (Fiocruz);
  - tríplice viral 6 h;
  - febre amarela 6 h;
  - hepatite B 10 dias;
  - VIP, dT e DTP (10 doses) 28 dias;
  - influenza 7 dias;
  - VPC10 (4 doses) 28 dias;
  - Covid Pfizer 12 h;
  - unidoses: uso imediato.
  - "Consultar a bula vigente": o prazo varia por fabricante e apresentação. **O modelo deve guardar o prazo por produto e lote, não por vacina.**
- **[P] Cadeia de frio: +2 °C a +8 °C (ideal +5 °C)**, com registro no mapa de temperatura **duas vezes ao dia** (início e fim do expediente), usando termômetro de máxima e mínima. Fonte é um POP municipal que cita o Manual de Rede de Frio, 5ª ed.: https://prefeitura.poa.br/sites/default/files/usu_doc/hotsites/sms/vigilancia-em-saude/POP%20-%20%20Armazenamento%20de%20Imunobiol%C3%B3gicos%20e%20Controle%20de%20Temperatura%202026.docx.pdf
- **[P] ESAVI** é notificado no **módulo ESAVI do e-SUS Notifica** (acesso gov.br):
  - grave: notificação imediata, **até 24 h**, e investigação em até 48 h (NT 88/2025);
  - a NT 6/2026 pede **todos** os ESAVI e erros de imunização.
  - Não há integração de sistema terceiro documentada ([N]). O caminho é um link ou atalho para o e-SUS Notifica e um registro local da ocorrência.
  - Fontes: https://www.gov.br/saude/pt-br/vacinacao/esavi/notas-tecnicas/nota-tecnica-no-88-2025-cgfam-dpni-svsa-ms e https://www.gov.br/saude/pt-br/vacinacao/notas-tecnicas/nota-tecnica-no-6-2026-cgfam-dpni-svsa-ms
- **[P] Extramuros e campanha:** pelo RIA-R 2.0 a campanha é uma *estratégia* (BREstrategiaVacinacao 5/6), e o CNES é o do estabelecimento responsável. Não achei regra oficial específica de extramuros ([N]); o prazo de 15 dias "sem conectividade" é a âncora para operação offline.

## 6. Caderneta digital

- **[C]** O **Meu SUS Digital** mostra as vacinas da RNDS ("Minha Saúde > Vacinas") e emite certificado. Desde 04/2025 traz a **Caderneta Digital da Criança** (miniapp).
  - Fontes: https://www.gov.br/saude/caderneta-digital-da-crianca e https://www.gov.br/pt-br/servicos/certificado-nacional-de-vacinacao-covid-19
- **[P]** "Até 10 dias após o envio" para aparecer no app, segundo matéria de imprensa (jb.com.br), não fonte oficial.
- **Implicações para o wpda:**
  1. O que for enviado via RIA aparece no Meu SUS. Uma caderneta municipal diferente gera divergência visível, então é melhor mostrar o que é local com o status RNDS (enviado, pendente ou erro).
  2. Doses aplicadas em outro município ou na rede privada só chegam lendo a RNDS (consulta de imunizações por CPF/CNS). Ler a RNDS exige perfil de consulta e finalidade LGPD; não verifiquei o endpoint ([N]).
  3. Não é preciso emitir certificado próprio: basta apontar para o Meu SUS Digital.

## Pontos críticos para o desenho

1. Envio **RNDS-only** (RIA-R 2.0) até 24 h, com fila e retry, log por registro e prazo de 15 dias offline.
2. Validar CPF/CNS e CNS/CNES do vacinador **antes** do envio.
3. A transcrição é um RIA-R com flag, e entra no indicador C2.
4. O estoque continua no SIPNI/SIES, sem API conhecida: controle local mais lançamento manual.
5. Calendário: importar as terminologias e curar as regras à mão a partir do PDF anual.
