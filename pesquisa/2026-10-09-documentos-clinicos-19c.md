# Documentos clínicos da consulta — pesquisa para o 19c

Data: 2026-10-09. Escopo: atestado, declaração de comparecimento, receita comum (e o caso do
antimicrobiano), pedido de exame e base estruturada de medicamentos. Controlados (SNCR) ficam no 19d.

Marcadores: **[CONFIRMADO]** = li a fonte primária (lei, resolução, manual oficial ou API);
**[SECUNDÁRIO]** = imprensa, blog ou resumo de terceiro; **[INFERÊNCIA]** = conclusão minha a partir das fontes.

---

## 0. Conclusões que mais pesam no desenho

1. **O PEC e-SUS APS é o padrão de mercado a imitar, e ele já faz quase tudo o que o 19c propõe.** No SOAP/Plano, o PEC tem atestado (modelos "Padrão", "Em branco" e "Licença maternidade", CID opcional), declaração de comparecimento (período ou horário, com acompanhante opcional), exames (comum, alto custo com CID obrigatório, e OCI com laudo de autorização), prescrição com lista **CATMAT** mais "registro manual", orientações e **prescrição digital**. A prescrição digital aceita gov.br para receitas sem retenção e ICP em nuvem para as demais. O cidadão recebe e-mail com QR code e código, e a farmácia confere e registra a dispensação num portal. [CONFIRMADO, manual PEC cap. 6]
2. **O nível de assinatura muda conforme o documento, e a regra mais dura vem do CFM, não da lei.**
   - Lei 14.063: só atestado médico e controlado exigem assinatura qualificada (art. 13). Os demais documentos de saúde valem com assinatura avançada (art. 14).
   - Anvisa (P&R RDC 1.000): receita comum e receita de antimicrobiano aceitam assinatura avançada, gov.br incluído.
   - CFM 2.299/2021: todo documento **médico** eletrônico (prescrição, atestado, pedido de exame) exige ICP-Brasil.
   - Desenho: a política de assinatura é decidida por **categoria profissional × tipo de documento**. Médico usa sempre ICP. Enfermeiro e dentista em documento não controlado podem usar gov.br avançada. [CONFIRMADO + INFERÊNCIA]
3. **A receita de antimicrobiano em meio eletrônico vai depender do SNCR.** A numeração SNCR "será obrigatória quando a ferramenta correspondente do SNCR estiver disponível". Até lá, a 2ª via é o PDF impresso e anotado pela farmácia. **Recomendação:** no 19c, o antimicrobiano sai **impresso em 2 vias com assinatura manual**, ou fica fora do digital, e vai para o 19d junto com o SNCR. [CONFIRMADO P&R Anvisa 25, 26, 49, 50]
4. **Não existe registro central de dispensação para receita comum.**
   - O "uso" da receita é registrado em cada plataforma: dispensador da Memed, portal do PEC, plataforma do CFM/CFF.
   - O registro nacional que está chegando é o **REPM/REDFM na RNDS** (Portaria GM/MS 6.100/2024), obrigatório por lei mas com prazo ainda pendente de plano tripartite. O REPM exige o medicamento codificado em **OBM**, motivo em CID/CIAP e assinatura eletrônica.
   - Desenho: guardar desde já os campos do REPM. [CONFIRMADO]
5. **Base de medicamentos:**
   - **CATMAT agora**, porque é o que o PEC usa e porque a API do Compras.gov é aberta, em JSON e sem autenticação; a classe 6505 tem 6.778 itens ativos.
   - **OBM como alvo**, porque é obrigatória para a RNDS e tem estrutura dm+d com vínculo a CATMAT e RENAME. O portal da OBM, porém, exige login gov.br e não tem download aberto confirmado.
   - Guardar o **código BR CATMAT** e a **descrição livre**, com espaço para o código OBM. [CONFIRMADO + INFERÊNCIA]
6. **Atestado Atesta CFM (Res. 2.382/2024):** a plataforma oficial e obrigatória está **suspensa por liminar do TRF-1** desde 2024-11-04. Não achei julgamento de mérito até 2026-10. Não integrar agora, mas deixar o atestado com código único, QR code e validador próprio. [SECUNDÁRIO]
7. **A declaração de comparecimento pode ser feita pela recepção.** Quando é só presença, qualquer colaborador emite, se houver norma interna (Parecer Cofen 44/2025). Quando decorre de procedimento de enfermagem, só o enfermeiro emite. Basta assinatura simples ou institucional. [CONFIRMADO]

---

## 1. Atestado

**Conteúdo obrigatório.**
- **CFM 1.658/2002, art. 3º (redação da 1.851/2008)** [CONFIRMADO via texto reproduzido]. O atestado deve:
  - especificar o tempo de afastamento;
  - trazer diagnóstico **só com autorização expressa do paciente**, e essa concordância deve constar no próprio atestado (parágrafo único);
  - ser legível;
  - identificar o emissor com assinatura e carimbo ou número do CRM.
  - O médico deve exigir documento de identidade do paciente.
  - Fontes: https://sistemas.cfm.org.br/normas/arquivos/resolucoes/BR/2008/1851_2008.pdf · https://www.legisweb.com.br/legislacao/?id=108475
- **CFM 2.299/2021** (documentos médicos eletrônicos) [SECUNDÁRIO, resumos de escritório]. Exige:
  - nome, CRM e endereço do médico;
  - RQE, se o documento for vinculado à especialidade;
  - nome e número de documento do paciente;
  - **data e hora**;
  - assinatura ICP-Brasil NGS2;
  - validação possível pelo validador do ITI ou do CFM.
  - Fonte: https://www.mattosfilho.com.br/unico/cfm-emissao-documentos-eletronicos/
- **CFM 2.382/2024** (Atesta CFM), arts. 6–9 [SECUNDÁRIO, texto via normaslegais]. Repete o conteúdo acima e acrescenta: CPF do paciente quando houver, contatos e endereço profissional, e **registro da autorização do CID** com aviso ao paciente sobre o risco de uso indevido. O art. 9 diz que só médico e dentista (no âmbito odontológico) emitem atestado de **afastamento do trabalho**. A obrigatoriedade da plataforma e da integração (arts. 1, 12, 13 e 15) está **suspensa** (TRF-1, 3ª Vara Federal Cível/DF, 2024-11-04). Fontes:
  - https://www.normaslegais.com.br/legislacao/resolucao-cfm-2382-2024.htm
  - https://canaltech.com.br/saude/justica-federal-suspende-norma-dos-atestados-digitais/
  - https://med.estrategia.com/portal/noticias/trf-1-suspende-resolucao-que-obriga-uso-do-atesta-cfm-para-atestados-medicos/
  - [INFERÊNCIA] O conteúdo mínimo da 2.382 vale como boa prática mesmo com a suspensão. Usar como checklist.

**Quem emite.**
- **Dentista:** emite atestado de afastamento no âmbito odontológico, com base na Lei 5.081/1966 e na CFO 87/2009 (que trata do assistente versus o perito). O manual do PEC cita a Lei 605/49 combinada com a Lei 5.081/66. [CONFIRMADO no manual PEC; CFO 87/2009 SECUNDÁRIO] https://sistemas.cfo.org.br/visualizar/atos/RESOLU%C3%87%C3%83O/SEC/2009/87
- **Enfermeiro:** **não** emite atestado de afastamento para o trabalho. Emite declaração ou atestado de **comparecimento** quando atende ou faz procedimento de enfermagem (Parecer Cofen 44/2025). [CONFIRMADO] https://www.cofen.gov.br/wp-content/uploads/2025/09/Parecer-no-44-2025-Camaras-Tecnicas-de-Enfermagem.pdf
- **eMulti** (fisioterapeuta, psicólogo, nutricionista): o manual do PEC registra que cada conselho admite documentos próprios, como atestado e relatório (COFFITO, CFP). O valor para abonar falta trabalhista é limitado. [CONFIRMADO no manual PEC; valor trabalhista: INFERÊNCIA]

**Atestado de acompanhante.**
- Não há figura própria no CFM. Na prática é uma declaração de comparecimento com o nome do acompanhante, como faz o PEC.
- A CLT, art. 473, garante abono sem prejuízo do salário em dois casos: **X**, até 2 dias para acompanhar consultas e exames da esposa ou companheira gestante; **XI**, 1 dia por ano para acompanhar filho de até 6 anos em consulta.
- **XII**: até 3 dias a cada 12 meses para exames preventivos de câncer.
- Fora disso, o abono depende do empregador ou da convenção coletiva.
- [CONFIRMADO CLT] https://www.planalto.gov.br/ccivil_03/decreto-lei/del5452.htm

**Validade digital para empregador e INSS.**
- Lei 14.063, art. 13: o atestado médico eletrônico só vale com **assinatura qualificada**. [CONFIRMADO] https://www.planalto.gov.br/ccivil_03/_ato2019-2022/2020/lei/l14063.htm
- **Atestmed/INSS:** exige nome, CID ou diagnóstico por extenso, data de início e prazo de afastamento, assinatura e registro no conselho (CRM, CRO ou RMS), sem rasura, emitido há menos de 90 dias. A assinatura pode ser eletrônica "conforme legislação". [SECUNDÁRIO] https://cbic.org.br/radar-trabalhista-atestados-medicos-de-ate-90-dias-nao-tem-mais-pericia-presencial/
- [INFERÊNCIA] Para o INSS, o CID é na prática necessário. A tela deve deixar claro, quando o paciente pede CID, que a autorização fica registrada no documento e no prontuário.
- **QR code:** não é exigido por lei para documento eletrônico. O ITI só valida a versão **impressa** se houver QR code compatível. A 2.382 previa QR code por página no atestado físico. [CONFIRMADO ITI/SECUNDÁRIO] https://h-validar.iti.gov.br/Docs/cartilha-de-uso.pdf
- **Desenho:** PDF/A assinado em PAdES (o 19b já faz isso), mais QR code apontando para uma página pública de verificação do Rota Saúde com código curto, mais o hash. Assim a versão impressa continua verificável.

## 2. Declaração de comparecimento

- **Quem emite:** em comparecimento administrativo, **qualquer colaborador**, se houver norma interna do serviço. Em comparecimento a procedimento ou consulta de enfermagem, **só o enfermeiro**; técnico e auxiliar não emitem. [CONFIRMADO, Parecer Cofen 44/2025, item 13]
- **Assinatura:** não está no art. 13 da Lei 14.063, porque não é atestado médico. Sendo administrativa, [INFERÊNCIA] basta assinatura simples ou institucional, como login com selo da unidade e código de verificação. Se emitida por profissional sobre ato clínico, use a regra do art. 14 (avançada).
- **Conteúdo** [CONFIRMADO, modelo PEC]: identificação do cidadão; data; período (matutino, vespertino, noturno ou horário de entrada e saída); unidade (CNES); nome do acompanhante, se houver; emissor.
- **Valor trabalhista:** a CLT não prevê abono por declaração de comparecimento, salvo os incisos do art. 473. [SECUNDÁRIO, parecer Coren-RS]
- **Desenho:** pode sair na recepção, fora da consulta imutável, a partir do registro de presença do módulo de agenda e check-in. Em vez de assinatura, usar QR code e código de verificação.

## 3. Receita comum (e antimicrobiano)

**Requisitos legais** — Lei 5.991/73, art. 35, redação da Lei 14.063, art. 15 [CONFIRMADO]. A receita deve:
- ser escrita em vernáculo, sem abreviações, legível, com nomenclatura e pesos oficiais;
- trazer nome e **endereço residencial** do paciente e o modo de usar;
- ter data, assinatura, endereço do consultório e número no conselho;
- **§1º:** vale em todo o território nacional;
- **§2º:** receita eletrônica só vale com assinatura **avançada ou qualificada** e atendendo a ato da Anvisa ou do MS.

Fonte: https://www.planalto.gov.br/ccivil_03/leis/l5991.htm

**DCB no SUS.**
- Lei 9.787/99, art. 3º: prescrições médicas e odontológicas no SUS adotam **obrigatoriamente a DCB** (na falta dela, a DCI). [CONFIRMADO] https://www.planalto.gov.br/ccivil_03/leis/l9787.htm
- Desenho: o nome impresso deve sair do princípio ativo, nunca de marca.

**Validade da receita comum.**
- Não achei prazo federal para receita não sujeita a retenção. [INFERÊNCIA] A validade fica a critério do prescritor ou de norma municipal da assistência farmacêutica; muitas REMUMEs fixam algo entre 30 dias e 6 meses para uso contínuo.
- Desenho: campo "validade" opcional, com padrão configurável por cidade.
- Os 10 dias valem só para antimicrobiano: IN 360/2025 e RDC 471/2021, alterada pela RDC 973/2025. GLP-1 tem 90 dias. [SECUNDÁRIO] https://www.legisweb.com.br/legislacao/?id=477175

**Enfermeiro prescritor — Cofen 801/2026, DOU 22/01/2026** [CONFIRMADO, texto do DOU]. Prescreve na consulta de enfermagem com base em protocolo aprovado pelo serviço ou de programa de saúde pública (art. 2º).

Conteúdo mínimo (art. 3º):
- **identificação do protocolo e ano de publicação**;
- **nome da instituição e CNPJ**;
- nome ou nome social do prescritor, número e categoria do Coren, assinatura física ou eletrônica;
- data;
- nome ou nome social do paciente e outro identificador (CPF ou data de nascimento);
- medicamento por **denominação genérica**, com via e posologia;
- modelos de receituário simples e sujeito a retenção nos Anexos I-A e I-B.

O art. 4º permite prontuário totalmente digital com assinatura avançada ou qualificada. O considerando cita o SNGPC aceitando antimicrobiano prescrito por enfermeiro. Fonte: https://www.portalcofen.gov.br/wp-content/uploads/2026/01/Resolucao-Cofen-no-801-2026-1.pdf

**→ Desenho:**
- o catálogo de **protocolos municipais** (nome e ano) é cadastro da cidade;
- a receita de enfermeiro precisa obrigatoriamente referenciar um protocolo;
- é preciso cadastrar o **CNPJ da instituição** (Fundo ou Secretaria Municipal).

**Dentista:** prescreve no âmbito odontológico, com base na Lei 5.081/66, art. 6º. A CFO 295/2026 foi citada, mas não li o texto integral. O CFO tem plataforma própria em https://prescricao.cfo.org.br. [SECUNDÁRIO]

**Antimicrobiano** — RDC 471/2021, IN 360/2025 e RDC 1.000/2025 com o P&R da Anvisa (1ª ed.) [CONFIRMADO no P&R]:
- Retenção da 2ª via; validade de 10 dias; dados do paciente e do emitente. Segundo a RDC 20/2011 e as notas dos CRFs, também idade e sexo do paciente. [SECUNDÁRIO]
- **Assinatura avançada, inclusive gov.br, é aceita** para receita sujeita a retenção (antimicrobiano, GLP-1) e para receita comum (P&R 38 e 40). Se assinada via gov.br, a validação é obrigatória no **Validar/ITI** (P&R 49). Outras assinaturas avançadas valem se comprovarem autoria e integridade (P&R 48).
- **SNCR:** a numeração SNCR aplica-se só a receituário eletrônico e "será obrigatória quando a ferramenta correspondente do SNCR estiver disponível" (P&R 25). Enquanto não estiver, o registro de uso é feito no **PDF impresso com anotação em papel**, que é o procedimento com amparo normativo (P&R 26 e 50).
- Fonte: https://www.gov.br/anvisa/pt-br/centraisdeconteudo/publicacoes/medicamentos/controlados/perguntas-e-respostas-rdc-1000-2025-1-ed.pdf
- [INFERÊNCIA] O antimicrobiano digital é emitível hoje, mas deixa de valer sem SNCR assim que a ferramenta abrir. Por isso cabe no 19d, com o SNCR. No 19c: tipo de receita "sujeita a retenção" impresso em 2 vias com assinatura manual, e o PDF digital marcado como "não usar para dispensação" até o 19d.

**Como a farmácia confere e dá baixa.**
- **Validar/ITI** (https://validar.iti.gov.br): confere a assinatura ICP ou gov.br no PDF, ou via QR code no impresso. **Não registra o uso**, apenas valida a assinatura. [CONFIRMADO P&R 39; ITI]
- **PEC:** o cidadão recebe link com QR code, código e PDF. A farmácia entra com gov.br em `prescricaodigital.esusaps.ufsc.br` (provisório), digita o código e clica "registrar medicamento", que registra a dispensação. [CONFIRMADO manual PEC]
- **Plataformas privadas** (Memed com dispensador próprio, Mevo, ex-Nexodata): baixa por item e bloqueio de dispensação duplicada. A verificação usa "o validador indicado pela empresa" (P&R 39). [SECUNDÁRIO + CONFIRMADO P&R]
- **CFM e CFF** (prescricaoeletronica.cfm.org.br): plataforma gratuita com registro de dispensação assinado pelo farmacêutico. [SECUNDÁRIO]
- **Registro central para receita comum:** **não existe hoje**. O caminho é o REDFM na RNDS. [CONFIRMADO Portaria 6.100]
- **→ Desenho mínimo:**
  - página pública de verificação por código e QR code, que mostra o PDF e o estado;
  - a farmácia **municipal** (Hórus ou própria) registra o fornecimento autenticada;
  - farmácia privada só confere.
  - Marcar "usada" para receita comum é opcional, ao contrário do antimicrobiano. [INFERÊNCIA]

## 4. Pedido de exame

- **Modelo do PEC** [CONFIRMADO]:
  - **Exame comum**: busca por nome em lista SIGTAP, CID opcional, justificativa e observações. Há "grupos de exames" configuráveis pelo município (dengue, gestante por trimestre, risco cardiovascular).
  - **Exame de alto custo**: CID-10 **obrigatório** e justificativa.
  - **OCI** (Programa Mais Acesso a Especialistas, Portaria SAES 1.821/2024): procedimentos, CID e justificativa. Gera **"Laudo para solicitação/autorização de procedimento ambulatorial"** e é transmitido ao Centralizador Nacional.
  - Os exames solicitados vão para o "Objetivo" do SOAP.
- **Assinatura:** pedido de exame médico, pela CFM 2.299, exige ICP. Pela Lei 14.063, art. 14, a avançada basta para os demais profissionais. Enfermeiro solicita exames de rotina e complementares em protocolo (Lei 7.498 e PNAB). Dentista solicita no âmbito odontológico. [CONFIRMADO lei; profissões: INFERÊNCIA/SECUNDÁRIO]
- **Formato municipal:** em geral é uma requisição em PDF ou papel com cabeçalho da Secretaria, aceita pelo laboratório conveniado ou próprio. Alto custo e consulta especializada passam pela **regulação** (SISREG ou sistema estadual ou municipal) com laudo APAC ou OCI. [INFERÊNCIA a partir do PEC e de POPs municipais]
- **→ Relação com o módulo 21 (exames) e a regulação:**
  - modelar o pedido como **entidade estruturada**: código SIGTAP, CID, justificativa, prioridade e estado (solicitado → agendado → coletado → resultado);
  - o PDF é só uma *renderização*, e o módulo 21 consome a entidade;
  - alto custo e OCI viram "solicitação regulada" com estado próprio;
  - não tentar integrar SISREG no 19c.

## 5. Base de medicamentos

| Base | O que é | Formato / acesso | Uso no Rota Saúde |
|---|---|---|---|
| **CATMAT** (Compras.gov, classe 6505) | Catálogo de compras, mantido para medicamentos pela Unidade Catalogadora do MS. Estrutura grupo → classe → PDM → item (código BR) | **API aberta JSON sem autenticação**: `dadosabertos.compras.gov.br/modulo-material/4_consultarItemMaterial?codigoClasse=6505&statusItem=true` → **6.778 itens ativos** (testado em 2026-10-09). A descrição é texto semiestruturado ("CIMETIDINA, CONCENTRAÇÃO: 400 MG") e há itens incoerentes ou inativos. Dado público. [CONFIRMADO] | **É o que o PEC usa** ("lista de medicamentos do CATMAT, disponibilizada pelo DESID/SECTICS/MS"; concentração, forma e tipo de receita vêm preenchidos). [CONFIRMADO manual PEC] |
| **OBM** (Portaria GM/MS 6.093/2024) | Terminologia nacional **obrigatória** para identificar medicamentos em sistemas e na RNDS. Modelo dm+d (NHS): VTM, VMP etc., com extensões RENAME e CATMAT | Portal `portal-obm.saude.gov.br`. Declara-se dado público, livre de licença e versionado, mas a navegação exige login gov.br, para "usuários e sistemas autorizados". Não confirmei download em massa nem pacote FHIR. [CONFIRMADO portal; download: não confirmado] | Alvo para o REPM e a RNDS. Prazos dependem de plano tripartite. |
| **RENAME / REMUME** | Lista de essenciais nacional e municipal, por DCB | PDF do MS; REMUME varia por município | Filtro de "disponível na rede" por cidade, pois cada cidade tem a sua REMUME |
| **DCB (Anvisa)** | Nomenclatura oficial dos princípios ativos | Lista publicada pela Anvisa | Nome impresso na receita (Lei 9.787) |
| **CMED** | Preços e registro de apresentações comerciais | Planilha da Anvisa | Pouco útil na APS do SUS, que prescreve por DCB |

Conexos:
- **Hórus**: o PEC consulta a disponibilidade do medicamento nas unidades.
- **BNAFAR**: base nacional da assistência farmacêutica.
- O **REPM** exige o medicamento conforme OBM, o motivo em CID/CIAP, via, dose, duração, frequência, CNES, conselho, UF e número do prescritor, e assinatura eletrônica. O **REDFM** pede lote, validade, fabricante e o CRF do farmacêutico. Não existe ficha **LEDI/CDS** própria de prescrição: o PEC transmite as prescrições dentro do atendimento individual. [CONFIRMADO Portaria 6.100; ausência de ficha LEDI: INFERÊNCIA, verificar no layout LEDI do módulo 16]

Fontes:
- https://bvsms.saude.gov.br/bvs/saudelegis/gm/2024/prt6100_18_12_2024.html
- https://www.conass.org.br/conass-informa-n-214-2024-publicada-a-portaria-gm-n-6-093-que-institui-a-ontologia-brasileira-de-medicamentos/
- https://www.gov.br/saude/pt-br/se/desid/catmat

**Recomendação** [INFERÊNCIA]:
1. Importar o CATMAT classe 6505 (job periódico pela API) para uma tabela de catálogo **de plataforma**, compartilhada entre cidades. Guardar código BR, PDM, descrição, status e data de atualização.
2. Usar uma camada de curadoria (princípio ativo DCB, concentração, forma, via padrão, tipo de receita: comum, retenção ou controlado). O texto do CATMAT não é bom para prescrição.
3. A **REMUME** é um recorte por cidade, no banco da cidade, sobre o catálogo.
4. Manter o **registro manual** em texto livre, como o PEC.
5. Na prescrição, guardar o snapshot do texto, o código CATMAT e a coluna `obm_code` vazia.
6. Abrir pedido de acesso à OBM (obm@saude.gov.br) para a fase RNDS.

## 6. Assinatura e papel

| Documento | Lei 14.063 | Regra setorial | Recomendação Rota Saúde |
|---|---|---|---|
| Atestado médico | **Qualificada** (art. 13) | CFM 2.299: ICP | ICP ou papel |
| Atestado odontológico | Avançada (art. 14)* | CFO (não confirmado) | ICP preferível; avançada aceitável [INFERÊNCIA] |
| Declaração de comparecimento | Não é atestado; administrativa | Cofen 44/2025 | Simples ou institucional mais QR code |
| Receita comum | Avançada ou qualificada (art. 14; Lei 5.991 art. 35 §2º) | Anvisa P&R 38: avançada ok. **CFM 2.299: médico usa ICP** | Médico: ICP. Enfermeiro e dentista: gov.br ou ICP |
| Antimicrobiano | Avançada ou qualificada | Anvisa: avançada ok, SNCR futuro | 19c: papel em 2 vias; digital no 19d |
| Pedido de exame | Avançada (art. 14) | CFM 2.299: médico usa ICP | Igual à receita comum |

\* O art. 13 fala em "atestados médicos". [INFERÊNCIA] O atestado odontológico de afastamento tende a ser tratado como equivalente. Por cautela, usar ICP.

**Quem não tem certificado:**
- A RDC 1.000 e a Lei 5.991 **não tornam a prescrição eletrônica obrigatória**. A receita em papel com assinatura manual continua válida. [CONFIRMADO P&R item 2.3]
- O sistema pode gerar o PDF **sem assinatura digital** para impressão e assinatura a caneta, com carimbo ou nome e número do conselho, como faz o PEC. O documento válido é o papel assinado. [CONFIRMADO no PEC; INFERÊNCIA jurídica]
- Atenção: **PDF físico assinado depois digitalmente não é receita eletrônica válida** (Anvisa, RDC 1.000). Também não se deve "imprimir em PDF" um documento já assinado. [SECUNDÁRIO e Prefeitura SP]
- **→ Desenho:** cada documento tem `modo_emissao` = `digital_icp` | `digital_avancada` | `papel`. A escolha é feita no ato da emissão; nunca converter um modo em outro depois.
- **gov.br como assinatura avançada:** o PEC e a Anvisa aceitam (contas prata e ouro). Isso exigiria integrar a API de assinatura gov.br, que hoje está restrita a órgãos públicos. [INFERÊNCIA; avaliar viabilidade para SaaS no contrato com o município]

## 7. Sistemas e editais

- **PEC e-SUS APS:** referência funcional completa. Ver item 0.1 e as seções 1 a 4. Exemplos de provedores de nuvem citados: Vidaas, RemoteID, NeoID, BirdID, Serasa. Manual: https://sisaps.saude.gov.br/sistemas/esusaps/docs/manual/PEC/PEC_06_atendimentos/
- **Rio de Janeiro (receita do PEC):** a farmácia confere o CRM, a assinatura com certificado e o QR code ou código numérico num link da prefeitura. [SECUNDÁRIO] https://doweb.rio.rj.gov.br/apifront/portal/edicoes/imprimir_materia/668034/4639
- **Memed:** usa dispensador com baixa por item e bloqueio de duplicidade; o PDF ICP é aberto pelo QR code. **Mevo (ex-Nexodata):** modelo B2B, com compra pelo link da receita. [SECUNDÁRIO] https://certforum.iti.gov.br/wp-content/uploads/apresentacoes/21-09/05-GabrielCouto-Memed.pdf
- **Editais:** não achei termo de referência municipal de prontuário APS com cláusula detalhada nesta busca. O padrão observado em TRs de certificado é A3 em nuvem (HSM) integrado ao prontuário (ex. FUABC). [SECUNDÁRIO] https://fuabc.org.br/wp-content/uploads/2023/06/TR-CERTIFICADO-DIGITAL.pdf
  - [INFERÊNCIA] O que editais costumam pedir: emissão de receita, atestado, declaração e requisição pelo prontuário; assinatura ICP; QR code de validação; prescrição por DCB integrada à REMUME e ao estoque (Hórus); exportação ao e-SUS/RNDS. Vale pesquisar editais concretos (PNCP) em separado.

## Lacunas abertas

- Texto integral da CFO 295/2026 e regras de atestado odontológico eletrônico.
- Status do mérito da ação contra o Atesta CFM, a ser conferido no processo no TRF-1.
- Acesso em massa ou FHIR à OBM, e os modelos computacionais do REPM/REDFM (guia RNDS).
- Data efetiva da ferramenta SNCR para retenção, que define o fim do papel para antimicrobiano.
- Viabilidade da API de assinatura gov.br para um fornecedor SaaS que não é órgão público.
