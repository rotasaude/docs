# Pesquisa 21b — laboratório municipal (LIS leve para prova de conceito)

Data: 2026-10-15. Pesquisa só de leitura, nenhum repo editado. Complementa
`2026-10-05-integracoes/frente-3-exames.md` (arranjos de exame, RNDS/REL, GAL) e
`2026-10-13-exames-provas-tecnicas.md` (21a) sem repetir o que já está lá.

Legenda: **[CONFIRMADO: url]** = lido na fonte. **[INFERÊNCIA]** = conclusão nossa, ou dado visto só em
resumo de mecanismo de busca (quando for o caso, a URL vem junto com a observação "só resumo de busca").

Fontes primárias lidas por inteiro:
- RDC Anvisa 978/2025, texto do DOU: https://www.in.gov.br/en/web/dou/-/resolucao-anvisa-n-978-de-6-de-junho-de-2025-635044217
- Perguntas & Respostas da RDC 978, 1ª edição (Anvisa/GGTES, 27/08/2025, cópia da SBAC):
  https://www.sbac.org.br/wp-content/uploads/2025/08/PR__978_2025_1a-versao-revisada_Portal_27ago2025.pdf
- Resoluções CFF 296/1996, CFM 2.235/2019 e CFM 2.299/2021 (texto integral) e CFBM 78/2002 (por extrato).
- Nove editais ou TRs municipais (lista na seção 3), o manual do GAL do LACEN-ES e o POP de envio ao LACEN-MS do HU-UFGD.

---

## Resumo executivo

1. **A norma vigente é a RDC 978, de 06/06/2025**, publicada no DOU de 10/06/2025 e em vigor na publicação
   (art. 194), com 90 dias de adequação (art. 191). Ela **revogou a RDC 786/2023** (art. 193), e foi a 786 que
   substituiu a RDC 302/2005. Foi alterada pela RDC 986, de 15/08/2025 (ajustes de redação).
   [CONFIRMADO: DOU e P&R acima]
2. **O que obriga o sistema** está nos arts. 105, 115–116, 128–139, 154 e 165–170:
   - cadastro do paciente e do exame com campos mínimos;
   - identificação da amostra com 4 campos;
   - quem coletou e quem recebeu;
   - histórico de alterações com usuário, data, hora e local, preservando o dado original;
   - laudo com 14 itens mínimos e "assinatura legalmente válida";
   - retificação só por **novo laudo**, com registro de quem foi avisado;
   - valores críticos definidos e com fluxo de comunicação;
   - guarda de **5 anos**.
3. **Assinatura do laudo:** a RDC aceita assinatura física ou digital **qualificada ou avançada** (Lei
   14.063) — ICP não é exigida pela Anvisa. Mas o **CFM 2.299/2021 exige ICP-Brasil NGS2** em laudo médico
   eletrônico. A "rubrica eletrônica com senha" do CFF 296/1996, que ainda aparece em editais, provavelmente não
   é "avançada" [INFERÊNCIA].
4. **Nos 9 editais lidos**, aparecem em 7 ou mais:
   - etiqueta com código de barras;
   - mapa de trabalho;
   - digitação com valores de referência;
   - liberação restrita ao profissional;
   - laudo na internet ou no app.

   Aparece em 6: **interfaceamento com equipamentos**, que é o maior risco para o 21b sem aparelhos. Em
   **2 ou menos**: GAL, controle de qualidade, SMS de resultado, estoque de reagentes, valores críticos e
   retificação. Nenhum pede coleta domiciliar.
5. **Os editais são cópias de modelos de fornecedor.**
   - Pato Branco e Contenda têm o mesmo texto; Mata de São João e Dionísio Cerqueira também.
   - A prova de conceito cobra de 90% a 100% dos itens **por módulo**. Faltar no laboratório pode derrubar a
     proposta inteira.
6. **GAL:** o laboratório municipal é **usuário** do GAL, não integra com ele. Ele faz quatro coisas:
   - cadastra a requisição;
   - imprime a requisição e a etiqueta;
   - faz a triagem e encaminha à rede;
   - emite a lista de remessa e imprime o laudo do LACEN.

   Não há API pública. Para o Rota Saúde, o mínimo é o número da requisição GAL, o registro da remessa, o PDF
   do laudo e o alerta de atraso. Alguns estados exigem etiqueta com código de barras no envio (RS, CIB
   634/2024).

---

## 1. Regra sanitária: RDC Anvisa 978/2025

### 1.1 Número, data, vigência, cadeia normativa

- **Número e data:** Resolução Anvisa nº 978, de 6 de junho de 2025. Ementa: "Dispõe sobre o funcionamento de
  Serviços que executam as atividades relacionadas aos Exames de Análises Clínicas (EAC)".
  [CONFIRMADO: https://www.in.gov.br/en/web/dou/-/resolucao-anvisa-n-978-de-6-de-junho-de-2025-635044217]
- **Vigência:** "entra em vigor na data de sua publicação" (art. 194). A publicação foi em 10/06/2025.
  - Prazo de adequação: 90 dias (art. 191).
  - Infraestrutura física só se exige em reforma ou construção.
  - A P&R explica que, nos 90 dias, valiam a 786 para os serviços existentes e a 978 para os novos.
  - A "vigência em 10/09/2025" citada pela VISA de Joinville é o fim desse prazo de adequação.
  [CONFIRMADO: DOU; P&R item 106; https://www.joinville.sc.gov.br/wp-content/uploads/2025/09/Informe-sobre-a-RDC-no-978-2025-Servicos-que-executam-atividades-relacionadas-a-Exames-de-Analises-Clinicas-EAC.pdf]
- **Cadeia normativa:** a RDC 978 **revoga a RDC 786/2023** (art. 193). A RDC 302/2005 já tinha saído com a 786.
  Sites comerciais que dizem "a 978 revoga a 302" simplificam. [CONFIRMADO: DOU art. 193; P&R, Introdução]
- **Alteração:** a RDC 986, de 15/08/2025 (DOU de 19/08/2025), alterou definições do art. 5º e os incisos I dos
  arts. 10, 22 e 51 (acrescentou "coleta em cavidade oral"). O texto do DOU usado aqui é o original. A versão
  consolidada está no AnvisaLegis. [CONFIRMADO: P&R, itens 107–108]
- **Revisão em curso:**
  - O Voto ROP 12/2025 abriu um processo de alteração da 978, para "ajuste de redação, sem alteração no mérito".
    [INFERÊNCIA, só resumo de busca: https://www.gov.br/anvisa/pt-br/composicao/diretoria-colegiada/reunioes-da-diretoria/votos/2025/rop-12.2025/2-12.pdf]
  - Não achei RDC de 2026 alterando a 978.
- **Abrangência:** aplica-se a serviços públicos e privados (art. 2º). O art. 58 permite que um EAS da
  administração pública seja classificado como serviço de EAC. [CONFIRMADO: DOU]

### 1.2 Tipos de serviço que importam para o município

- **Tipo III:** é o laboratório clínico completo. Só ele pode ter metodologia própria (art. 157) e ser
  laboratório de apoio (art. 5º, XXV). O laboratório municipal é Tipo III. [CONFIRMADO: DOU]
- **Posto de coleta (Tipo II):**
  - O posto de coleta é Tipo II (art. 19). Em regra, se vincula a um único Tipo III (arts. 61–62).
  - **EAS público Tipo II** pode receber, armazenar, acondicionar, processar e transportar amostra da fase
    pré-analítica para outro Tipo II ou Tipo III público (art. 70).
  - Também pode fazer EAC *in loco* com equipamento de uso único ou microscopia (art. 71).
  - Ou seja, a **UBS que coleta para o laboratório municipal** é um posto Tipo II.
  [CONFIRMADO: DOU]
- **Itinerante (art. 72):** o EAC itinerante pode coletar para EAS público. Na política pública, deve estar
  vinculado a um EAS fixo. [CONFIRMADO: DOU]
- **Coleta domiciliar:** pode ser feita por técnico capacitado, sem o supervisor presente. O supervisor fica na
  sede. [CONFIRMADO: P&R item 90]

### 1.3 O que obriga o SISTEMA (por tema)

#### Requisição e cadastro

A RDC não define "requisição" como documento. Ela define dois cadastros feitos pelo serviço que coleta.

- **Cadastro do paciente (art. 134):** oito campos.
  - número de registro do paciente;
  - nome completo;
  - nome social;
  - data de nascimento;
  - **sexo biológico**;
  - **nome da mãe**;
  - telefone ou endereço;
  - responsável, quando menor ou incapaz.
  [CONFIRMADO: DOU]
- **Documento com foto (art. 132, II):** é exigido para o cadastro. Duas exceções:
  - urgência ou internação, em que vale o prontuário (§1º);
  - políticas públicas, em que vale "normativa vigente do Ministério da Saúde" (§3º).
  [CONFIRMADO: DOU]
- **Cadastro do EAC (art. 136):** doze campos.
  - nome do solicitante;
  - data e hora do atendimento;
  - **horário da coleta**;
  - nome e **CNES da unidade que coletou**;
  - exames e tipo de material;
  - informações adicionais;
  - **quem cadastrou**;
  - **quem coletou**;
  - **quem validou o recebimento**;
  - data prevista do laudo;
  - **urgência**;
  - medicamentos e condições relevantes.
  [CONFIRMADO: DOU]
- **Comprovante de atendimento (art. 137):** número de registro, nome, data, **data prevista de entrega**,
  exames e contato do serviço. É dispensado quando o resultado sai no ato. [CONFIRMADO: DOU]
- **Instruções de preparo (art. 132, I):** físicas, digitais ou verbais, "em linguagem acessível". [CONFIRMADO: DOU]
- **Implicação para o 21b [INFERÊNCIA]:**
  - O pedido do 21a cobre o solicitante e os exames.
  - O 21b precisa acrescentar sexo biológico, nome da mãe, horário da coleta, CNES do coletor, os três
    "quem", urgência e informações clínicas.
  - O comprovante pode sair em PDF ou no wpda.

#### Identificação inequívoca e rastreabilidade

- **Rastreabilidade (art. 128):** o serviço deve "garantir e evidenciar a rastreabilidade de todas as atividades
  relacionadas ao material biológico" nas três fases. [CONFIRMADO: DOU]
- **Identificação da amostra (art. 138):** feita no momento da coleta ou do recebimento.
  - Mínimo de quatro campos: **nome, data de nascimento ou idade, tipo de material e número de registro do
    paciente**.
  - A data e a hora da coleta devem ter rastreabilidade garantida (§2º).
  - Exame no ato com laudo imediato admite identificação simplificada (art. 129, parágrafo único).
  [CONFIRMADO: DOU; P&R item 93]
- **Código de barras:** a RDC **não exige**. É prática e requisito de edital, não norma federal. Mas a CIB/RS
  634/2024 exige código de barras nas amostras enviadas à rede de laboratórios de saúde pública no RS (ver §4).
  [CONFIRMADO: DOU (ausência); ETP de Porto Alegre]
- **Aceitação e rejeição (art. 139):** o serviço deve "definir critérios para aceitação, rejeição, identificação e
  realização de EAC". Amostra aceita "com restrição" precisa constar no laudo (art. 167, XII). [CONFIRMADO: DOU]
- **Condição da amostra:** não há campo obrigatório de "condição" além da restrição.
  - O transporte segue a RDC 504/2021 (arts. 140 e 143).
  - Equipamentos de temperatura registram máxima, mínima e momento (art. 96).
  [CONFIRMADO: DOU]
- **Laboratório de apoio:**
  - O serviço responde pelo envio de amostras ao laboratório de apoio (art. 141).
  - O laudo deve conter o nome e o CNES de quem fez a fase analítica (arts. 129 e 169).
  - Transcrever o laudo do apoio exige fidedignidade (art. 168).
  [CONFIRMADO: DOU]

#### Fases pré-analítica, analítica e pós-analítica

- **Definições:**
  - A **pré-analítica** vai "da solicitação da análise" até o início da análise (art. 5º, XXI).
  - A **analítica** é o conjunto de processos de análise (XIX).
  - A **pós-analítica** vai dos resultados válidos até a "emissão do laudo e providências de comunicação"
    (XX).
  [CONFIRMADO: DOU]
- **Instruções escritas por fase (arts. 132-III, 154-I e 165):** no pós-analítico, cobrem "liberação de resultados
  e assinatura legalmente válida de laudos", incluindo "rotina, plantões, urgências e emergências".
  [CONFIRMADO: DOU]
- **Sistema de informação (art. 105):** é o artigo mais "de software". Exige política de responsabilidades com:
  - acesso a dados;
  - **"controle de acesso a lançamento e alterações de dados e resultados dos exames dos pacientes, contendo
    históricos com data, horário, usuário e local"**;
  - "processo sistematizado para liberação de resultados... seja ela automática ou manual";
  - privacidade;
  - proteção contra acesso não autorizado.
  [CONFIRMADO: DOU]
- **Validação do software (art. 106):** o serviço deve "realizar a avaliação e verificação do sistema ou programa
  de informática antes de sua implementação, e documentá-las". Os pontos são disponibilidade, confiabilidade,
  integridade dos dados e responsabilidades. [CONFIRMADO: DOU]
  - **Oportunidade [INFERÊNCIA]:** entregar ao laboratório um "dossiê de validação do sistema" pronto. O RT
    precisa dele para a VISA.
- **LGPD (art. 77):** política de acesso a dados "de acordo com" a LGPD. [CONFIRMADO: DOU]

#### Conteúdo obrigatório do laudo (art. 167, mais arts. 129, 159, 166 e 169)

O art. 166 pede laudo "legível, sem rasuras de transcrição, escrito em língua portuguesa, datado e assinado por
profissional de nível superior legalmente habilitado". O art. 167 lista o mínimo: [CONFIRMADO: DOU]

| # | Item (art. 167) | Observação para o 21b [INFERÊNCIA] |
|---|---|---|
| I | Nome do serviço responsável pela análise e **CNES** | Do estabelecimento laboratório |
| II | Endereço e telefone desse serviço | Cabeçalho |
| III | Nome e registro no conselho do **Responsável Técnico** | O RT do laboratório, que pode ser diferente de quem assina |
| IV | Nome e registro no conselho de **quem assina o laudo** | Profissional liberador |
| V | Nome e registro de identificação do paciente | — |
| VI | Idade ou data de nascimento | — |
| VII | **Data da coleta** | — |
| VIII | Nome do exame, **tipo de material** e **método analítico** | O método é um campo do cadastro do exame |
| IX | Resultado e **unidade de medida** | — |
| X | **Valores de referência**, limitações do método e dados para interpretação | Faixas por sexo e idade |
| XI | Metodologia própria, quando houver | Não se aplica ao 21b |
| XII | Aceitação de material **com restrição** | Vem da recepção |
| XIII | **Data de emissão** | — |
| XIV | **Assinatura legalmente válida** | Ver abaixo |

Também entram no laudo:
- o laboratório de apoio, quando houver: nome e CNES (arts. 129 e 169);
- a indicação de metodologia própria (art. 159).
[CONFIRMADO: DOU]

- **Laudo parcial:** a RDC **não menciona** laudo parcial. O GAL usa "Laudos Parciais" e "Laudos finais" (§4). O
  edital do HSPM pede exibir laudos com resultados parciais (§3). [CONFIRMADO: DOU (ausência); manual GAL-ES;
  edital HSPM]

#### Assinatura do laudo

- **RDC 978, art. 5º, III:** "assinatura legalmente válida: assinatura física ou digital, **qualificada ou
  avançada**, nos termos da Lei nº 14.063". O art. 167, XIV e o art. 116 usam o termo. [CONFIRMADO: DOU]
- **Consequências [INFERÊNCIA]:**
  - Para a Anvisa basta a assinatura **avançada**, como a do gov.br. A ICP-Brasil é suficiente, mas não exigida.
  - Uma "assinatura" por login e senha no próprio sistema tende a ser **simples**, e não avançada.
  - Ainda assim, editais pedem exatamente isso: "liberação eletrônica controlada por senha (assinatura
    eletrônica)" em Santa Izabel, e "assinatura eletrônica... identificar o operador, data e horário" em Pato
    Branco.
- **Conselhos:** detalhes na seção 2. O CFM exige ICP NGS2 para médico.
- **Recomendação [INFERÊNCIA]:**
  - Assinar o PDF em **PAdES** com o certificado do liberador, reaproveitando o 19b.
  - Aceitar ICP (A1/A3/nuvem) e gov.br avançado.
  - Manter a liberação por senha só como "liberação", nunca como assinatura.

#### Guarda de registros e laudos

- **Prazo (art. 115):** "Toda a documentação e registros de que trata esta Resolução devem ser mantidos e
  arquivados pelo prazo mínimo de **5 (cinco) anos**". [CONFIRMADO: DOU]
- **Alteração de registros (art. 116):** "devem conter data, nome ou assinatura legalmente válida do responsável
  pela alteração, **preservando o dado original**". [CONFIRMADO: DOU]
- **Recuperação (art. 131):** o serviço deve garantir "a recuperação e a disponibilidade de todas as informações
  relativas ao EAC". [CONFIRMADO: DOU]
- **Material biológico:** "Não existe normativo sanitário federal que determine o tempo de guarda de qualquer tipo
  de material biológico". [CONFIRMADO: P&R item 12]
- **Prontuário [INFERÊNCIA]:** se o laudo também integra o prontuário do paciente, vale a regra de guarda de
  prontuário (Lei 13.787/2018, 20 anos). Prevalece o maior prazo. Não reconferido nesta pesquisa.

#### Controle de qualidade interno (CIQ) e externo (CEQ)

- **Obrigatoriedade:**
  - A GCQ é obrigatória, com no mínimo CIQ e CEQ (arts. 171–172).
  - É feita *in loco* (art. 173), em todos os equipamentos e analitos (arts. 179–180 e 187–188).
  - A participação é individual por serviço (arts. 178 e 186).
  [CONFIRMADO: DOU]
- **Registro obrigatório:**
  - "manter registros dos Controles da Qualidade" (art. 174);
  - a documentação da GCQ inclui "avaliação e registro dos resultados dos controles" e o relatório anual do
    CEQ (art. 176);
  - "registros de resultados dos programas de Ensaios de Proficiência e Controle Interno" (art. 117, VI).
  [CONFIRMADO: DOU]
- **Elo com a liberação:** o CIQ exige "liberação ou rejeição das análises após avaliação dos resultados das
  amostras controle" e o registro de inadequações e ações (art. 180, III–IV). [CONFIRMADO: DOU]
- **Frequência:**
  - No mínimo a cada corrida analítica e a cada manutenção (art. 183, §2º).
  - Em produto de uso único, a cada lote e a cada remessa.
  [CONFIRMADO: DOU]
- **Leitura para o sistema [INFERÊNCIA]:**
  - A RDC exige o **registro**, não que ele esteja no LIS. O laboratório pode manter o CIQ no software do provedor
    de controle ou no próprio equipamento.
  - Um 21b "leve" pode ficar sem CIQ, desde que o RT tenha onde registrar.
  - Vale oferecer um campo mínimo: "corrida aprovada pelo CIQ? sim/não, responsável". Esse campo bloqueia a
    liberação do mapa e já atende o art. 180, III, sem gráfico de Levey-Jennings.

#### Valores críticos

- **Definir valores críticos e fluxo de comunicação (art. 154, III):** "definir limites de risco, valores
  críticos ou de alerta, para os exames com resultados que necessitem tomada imediata de decisão, bem como devem
  definir fluxo de comunicação ao profissional de saúde responsável ou ao paciente". [CONFIRMADO: DOU]
- **Fluxo de comunicação na decisão imediata (art. 165, parágrafo único):** "definir o fluxo de comunicação ao
  paciente, ao médico ou profissional de saúde responsável pelo paciente, quando houver necessidade de decisão
  imediata". [CONFIRMADO: DOU]
- **Procedimento documentado (art. 119, II):** "procedimentos para resultados potencialmente críticos".
  [CONFIRMADO: DOU]
- **Registro de quem foi avisado:**
  - A RDC **não exige expressamente** esse registro no valor crítico. Exige no caso da retificação (art. 170).
  - A rastreabilidade geral (arts. 128 e 105) e a boa prática (POPs de HU-EBSERH) levam a registrar quem, quando
    e como.
  - O edital do HSPM pede "registro da comunicação com o prescritor responsável".
  [CONFIRMADO: DOU (ausência); edital HSPM 1.4.14.14] [INFERÊNCIA: registrar sempre]
- **Doença de notificação compulsória (art. 66):** resultados com suspeita de DNC "devem ser notificados" (PRC
  GM/MS 4/2017 e Portaria GM/MS 420/2022). [CONFIRMADO: DOU] O 21b pode sinalizar exames de DNC [INFERÊNCIA].

#### Retificação de laudo

- **Art. 170:** "Caso haja a necessidade de retificação em qualquer dado constante do laudo já emitido, a mesma
  deve ser feita em **um novo laudo** em que fique clara a retificação realizada". [CONFIRMADO: DOU]
- **Art. 170, parágrafo único:** em alteração relevante depois da entrega, o Tipo II e o Tipo III "devem assegurar
  que seja realizado contato com paciente ou seu responsável, informando o fato e **registrando o nome contatado e
  data** de tal comunicação". [CONFIRMADO: DOU]
- **Implicação [INFERÊNCIA]:**
  - O laudo é imutável.
  - A retificação gera uma versão N+1, com motivo, referência à anterior, texto "RETIFICADO" e nova assinatura.
  - A anterior fica preservada (art. 116).
  - Em alteração relevante, abre-se uma tarefa "contatar paciente" com nome do contatado e data.

---

## 2. Conselhos: quem assina e libera, e assinatura digital

| Conselho | Norma | O que diz | Fonte |
|---|---|---|---|
| CFF (farmacêutico-bioquímico) | Res. 296/1996 | O farmacêutico-bioquímico pode ser RT de laboratório de análises clínicas (art. 1º). O RT "deverá datar e assinar os laudos... constando obrigatoriamente o seu registro profissional" (art. 3º). Laudos podem ser assinados por "chefes dos setores, plantonistas ou substitutos", legalmente habilitados (art. 4º). "rubricas eletrônicas... sob proteção de senhas" valem como as de próprio punho, sempre com nome completo e registro (art. 5º). | [CONFIRMADO: https://www.cff.org.br/userfiles/file/resolucoes/296.pdf] |
| CFF | Res. 727/2022 (telefarmácia) | Exige assinatura qualificada ICP-Brasil na telefarmácia. Não trata de laudo. | [INFERÊNCIA, só resumo de busca: https://www.legisweb.com.br/legislacao/?id=434209] |
| CFBM (biomédico) | Res. 78/2002 | O biomédico habilitado pode "assumir a Responsabilidade Técnica e firmar os respectivos laudos" (art. 5º e §3º). A RT é do biomédico com estabelecimento inscrito no CRBM (art. 11). **Não fala de assinatura eletrônica.** | [CONFIRMADO (extrato): https://www.legisweb.com.br/legislacao/?id=98598] |
| CFBM | Res. 399/2025 | Trata da cédula profissional digital (ProID com QR), não de laudo. | [CONFIRMADO: https://www.normasbrasil.com.br/norma/resolucao-399-2025_481877.html] |
| CFM (patologista clínico) | Res. 2.235/2019 | Exames de "patologia clínica..." em serviço médico ficam "sob a responsabilidade de médico" (art. 1º), com laudo "emitido por médicos, com registro no CRM" (art. 2º). Quem executa pode ser diferente de quem lauda (art. 3º). Laudo de exame especializado exige RQE na área (art. 5º, parágrafo único). | [CONFIRMADO: https://sistemas.cfm.org.br/normas/arquivos/resolucoes/BR/2019/2235_2019.pdf] |
| CFM | Res. 2.299/2021 | Autoriza documentos médicos eletrônicos, incluindo "Laudo" (art. 1º, e). Exige nome, CRM, endereço, RQE, paciente, **data e hora** e assinatura digital (art. 2º). A assinatura é por "certificados e chaves emitidos pela... ICP-Brasil, com Nível de Garantia de Segurança 2 (NGS2)" (art. 4º). | [CONFIRMADO: https://abmes.org.br/public/arquivos/legislacoes/Resolucao-CFM-2299-2021-09-30.pdf] |
| Anvisa | RDC 978, arts. 5º XXXV, 75, 122 e 166 | "Profissional legalmente habilitado" é o de nível superior inscrito no conselho, "com suas competências atribuídas por Lei". O laudo é assinado por ele. Há RT obrigatório com substituto. Há supervisor presencial durante todo o funcionamento. | [CONFIRMADO: DOU] |

**Leitura [INFERÊNCIA]:**
- **Quem libera:**
  - O 21b deve ter um papel "liberador", ligado ao conselho (CRF/CRBM/CRM e UF) e ao número de registro.
  - Também precisa do RT do laboratório e do substituto.
  - Técnico de nível médio não libera nem assina. A Res. CFF 485/2008 veda isso, segundo resumo de busca.
- **Assinatura:** o piso é o da Anvisa (avançada).
  - Se o liberador for médico, a ICP NGS2 é obrigatória.
  - Para farmacêutico e biomédico, o conselho não exige ICP no laudo. A rubrica com senha do CFF/1996 é anterior à
    Lei 14.063, e a RDC de 2025 é mais restritiva.
  - Padrão único proposto: **PAdES com ICP ou gov.br avançado**.
- **Disputa histórica:** CFM × CFF/CFBM sobre laudo citopatológico/anatomopatológico. Ver CFM 2.074/2014, revogada
  pela 2.169/2017, e decisões judiciais. **Fica fora do 21b**, que é de análises clínicas. [INFERÊNCIA, só resumo de
  busca: https://portal.cfm.org.br/noticias/sentenca-da-justica-federal-proibe-biomedicos-de-assinarem-laudos-de-exames-citopatologicos/]

---

## 3. Editais: requisitos funcionais de laboratório

### 3.1 Editais lidos

| # | Município / órgão | Documento | Ano | Tipo | Regra da prova de conceito | Fonte |
|---|---|---|---|---|---|---|
| E1 | Pato Branco/PR | PE 41/2023 | 2023 | Sistema integrado, com módulo Laboratório | ≥ **95% dos requisitos de cada módulo** (Anexo V, 3.1.5) | [CONFIRMADO: https://patobranco.pr.gov.br/wp-content/uploads/2023/04/41-SOFTWARE-GESTAO-DE-SAUDE-PUBLICA.pdf] |
| E2 | Contenda/PR | PE 106/2023 | 2023 | Integrado, **mesmo modelo de texto do E1** | "atender todos os itens" (16.7) | [CONFIRMADO: https://www.contenda.pr.gov.br/prefeitura_app/storage/app/public/licitacoes/Edital-Sistema-de-Gestao-Saude.pdf] |
| E3 | CISLESTE/MG (consórcio) | PE 08/2025 – TR | 2025 | Integrado; cada item marcado **SIM/NÃO** (obrigatório) | 100% dos obrigatórios; o resto em 180 dias (14.4–14.5) | [CONFIRMADO: http://www.cisleste.mg.gov.br/portal/wp-content/uploads/2025/07/PREGAO-08-25-SISTEMA-SAUDE-TR.pdf] |
| E4 | Xanxerê/SC | TR | 2022 | Integrado; a justificativa cita "Laboratório Municipal" | Demonstração de todos os itens do TR | [CONFIRMADO: https://xanxere.sc.gov.br/uploads/sites/92/2022/06/Termo-de-Referencia-Sistema-de-Gestao-Saude.pdf] |
| E5 | São Carlos/SP | TR | 2024 | Integrado; o laboratório é **externo, por integração** | Obrigatórios no ato, não obrigatórios em 90 dias (3.1.7) | [CONFIRMADO: https://servico.saocarlos.sp.gov.br/licitacao/arquivos-resumos/07-16-2024-04:46:59-pm-8345-6866--Resumo-.pdf] |
| E6 | HSPM – São Paulo (municipal) | PE 90412/2024 | 2024 | **LIS dedicado** (SGL + interfaceamento + BI) | 100% da lista da prova (8.18) | [CONFIRMADO: https://capital.sp.gov.br/documents/d/hospital_do_servidor_publico_municipal/90412-2024-pdf] |
| E7 | Santa Izabel do Pará/PA | PE SRP 036/2018 | 2018 | **LIS + analisadores + reagentes** para o Laboratório Municipal | — | [CONFIRMADO: https://santaizabel.pa.gov.br/wp-content/uploads/2021/05/ARQUIVO_1538768543.pdf] |
| E8 | Mata de São João/BA | TR, 28/12/2023 | 2023 | Integrado, seção "LIS" | Desclassifica acima de 10% de itens não atendidos | [CONFIRMADO: https://www.matadesaojoao.ba.gov.br/Handler.ashx?f=f&query=f9776ff0-6257-466b-8bca-8c3eed23d973.pdf&name=8_TERMO_DE_REFERENCIA.pdf] |
| E9 | Dionísio Cerqueira/SC | TR, 01/02/2024 | 2024 | Integrado, **mesmo modelo de texto do E8** | 95% das funcionalidades (6.2) | [CONFIRMADO: https://dionisiocerqueira.sc.gov.br/uploads/sites/267/2024/03/SISTEMA-LICITACAO-ANEXO-II.pdf] |

Documentos de apoio (não contados):
- **ICISMEP/MG, Pregão 02/2025:** registro de preço de LIS para os "laboratórios municipais" dos consorciados.
  - Ata 210/2025 com a Pixeon (Korus LIS).
  - Licença de **R$ 440/mês**, implantação a R$ 230 e hora técnica a R$ 330.
  - Inclui dashboards, soroteca, BPA-C/BPA-I e mapa de trabalho configurável.
  [CONFIRMADO: https://icismep.mg.gov.br/wp-content/uploads/2025/01/Catalogo-PL-03.2025-Software-LIS.pdf]
- **Araranguá/SC, Edital 25/2025:** compra de reagentes do laboratório municipal. O vencedor "deverá ser
  responsável por instalar o interfaceamento", inclusive "INTERFACIAMENTO PARA O SISTEMA G-MUS".
  [CONFIRMADO: https://ararangua.sc.gov.br/uploads/sites/281/2025/09/Edital-de-Licitacao-n-252025-Insumos-e-Reagentes-lab-municipal.pdf]
- **Joinville/SC, SEI 21.0.180526-4:** guia "iPSO" de integração entre o sistema municipal (Saudetech/Olostech) e
  um LIS externo.
  - O LIS busca a autorização pelo número.
  - Ele notifica o laudo pronto.
  - O sistema municipal exibe o PDF por URL temporária de 2 h.
  - É o modelo do **21a** (prestador), não do 21b.
  [CONFIRMADO: https://www.joinville.sc.gov.br/public/edital/anexo/98e995990ba1f5fad232da1df1b70fb4.pdf]
- **Mafra/SC:** não localizei edital de sistema de saúde com prova de conceito. Os editais de Mafra achados são de
  outros objetos.

### 3.2 Contagem por requisito (9 editais)

Legenda:
- ✓ = consta na seção de laboratório.
- (o) = consta, mas como item **não obrigatório** (NÃO no E3) ou **fora da lista da prova** (E6).
- "—" = não consta.

| Requisito | E1 | E2 | E3 | E4 | E5 | E6 | E7 | E8 | E9 | **Total** |
|---|---|---|---|---|---|---|---|---|---|---|
| Digitação de resultado com valores de referência / consistência mín.–máx. / destaque fora da faixa | ✓ | ✓ | ✓ | ✓ | — | ✓ | ✓ | ✓ | ✓ | **8** |
| Relatório de produção (por unidade, exame, profissional, valores SIGTAP) | ✓ | ✓ | (o) | ✓ | — | ✓ | ✓ | ✓ | ✓ | **8** |
| **Etiqueta com código de barras** (por material/bancada, reimpressão, impressora térmica) | ✓ | ✓ | ✓ | — | — | ✓ | ✓ | ✓ | ✓ | **7** |
| **Mapa/lista de trabalho** (por bancada/setor, urgência, não emitidos) | ✓ | ✓ | ✓ | — | — | ✓ | ✓ | ✓ | ✓ | **7** |
| Liberação restrita ao profissional, com registro de quem/quando | ✓ | ✓ | ✓ | — | — | ✓ | ✓ | ✓ | ✓ | **7** |
| **Laudo na internet / portal / app do cidadão** | ✓ | ✓ | ✓ | — | — | ✓ | ✓ | ✓ | ✓ | **7** |
| **Interfaceamento com equipamentos** | ✓ | ✓ | (o) | — | — | ✓ | ✓ | ✓ | — | **6** |
| Restrição de laudos sensíveis (ex.: HIV) | ✓ | ✓ | (o) | — | — | ✓ | ✓ | — | ✓ | **6** |
| Resultado visível no prontuário / HIS | — | — | ✓ | ✓ | ✓ | ✓ | — | ✓ | ✓ | **6** |
| Acompanhamento de status (coleta → digitação → assinatura → entrega) | ✓ | ✓ | ✓ | — | — | ✓ | ✓ | ✓ | — | **6** |
| Pedido importado do prontuário / requisição eletrônica | ✓ | ✓ | — | — | — | ✓ | — | ✓ | ✓ | **5** |
| Controle de entrega do laudo (quem retirou) | ✓ | ✓ | — | — | — | — | ✓ | ✓ | ✓ | **5** |
| Comprovante com data prevista de entrega | ✓ | ✓ | ✓ | — | — | — | ✓ | — | — | **4** |
| Registro de ocorrência/não conformidade da amostra | — | ✓ | ✓ | — | — | ✓ | ✓ | — | — | **4** |
| **Assinatura digital** do laudo (certificado ICP / avançada) | — | — | ✓ ICP | ✓ | — | ✓ avançada | — | — | — | **3** |
| (Assinatura "eletrônica" por senha, sem certificado) | ✓ | ✓ | — | — | — | — | ✓ | — | — | (3) |
| Bloqueio de impressão antes da liberação/assinatura | — | — | ✓ | — | — | — | — | ✓ | ✓ | **3** |
| **Faturamento BPA** da produção do laboratório (CID/SIGTAP para BPA, arquivo SUS) | ✓ | ✓ | ✓ | — | — | — | — | — | — | **3** |
| Laboratório de apoio (lote de envio, retorno de resultado) | — | — | ✓ | — | — | ✓ | ✓ | — | — | **3** |
| Resultados anteriores / delta-check | — | — | (o) | — | — | ✓ | ✓ | — | — | **3** |
| **Valores críticos** com comunicação registrada | — | — | (o) | — | — | (o) | — | — | — | **2** |
| **Controle de qualidade** (Westgard, Levey-Jennings) | — | — | — | — | — | (o) | ✓ | — | — | **2** |
| **Integração GAL** (consulta de resultado COVID via API da SESA-PR) | ✓ | ✓ | — | — | — | — | — | — | — | **2** |
| **Resultado por SMS / e-mail** ("retirada de resultado", "exames liberados") | ✓ | — | — | — | — | — | ✓ | — | — | **2** |
| Soroteca (armazenamento de amostras) | — | — | ✓ | — | — | ✓ | — | — | — | **2** |
| Lote de transporte com temperatura e umidade | — | — | (o) | — | — | ✓ | — | — | — | **2** |
| Recepção por biometria | ✓ | — | — | — | — | — | ✓ | — | — | **2** |
| **Controle de estoque de reagentes** dentro do módulo | — | — | ✓ | — | — | — | — | — | — | **1** |
| **Retificação** de laudo com motivo | — | — | — | — | — | (o) | — | — | — | **1** |
| Laudo parcial | — | — | — | — | — | ✓ | — | — | — | **1** |
| **Coleta domiciliar** | — | — | — | — | — | — | — | — | — | **0** |
| **Painel de produção / BI** (dashboard, não relatório) | — | — | — | — | — | ✓ | — | — | — | **1** (+ ICISMEP) |

Notas sobre a tabela:
- **E3 (CISLESTE) é o melhor termômetro do "mínimo"**, porque marca o que é obrigatório. **SIM**:
  - etiqueta (903, 904, 906, 907);
  - protocolo com previsão de entrega, link e senha web (909);
  - configuração do laudo (898);
  - rastreio por número da requisição em todo o fluxo (918);
  - ocorrências (919);
  - laboratório de apoio por API **ou** portal para o prestador digitar e assinar (914);
  - assinatura "com certificação digital... liberada por profissional mediante senha" (949);
  - "Certificação Digital... ICP-Brasil nos laudos laboratoriais, conforme estabelecido na RDC" (688);
  - liberar a impressão só após assinatura (695);
  - soroteca (946–950).

  **NÃO** (pontuável):
  - todo o interfaceamento (929, 931–941). O item 929 diz que ele "deve ser de responsabilidade das empresas
    responsáveis pelos equipamentos" e que "a contratada deve fornecer a API de integração";
  - valores críticos (915);
  - restrição de laudo por grupo (917);
  - repasse a outros laboratórios (942–943);
  - definição de quem assina (911).
  [CONFIRMADO: E3]
- **E6 (HSPM):** a lista da prova cobra:
  - integração com o HIS;
  - laboratório de apoio;
  - filas de atendimento;
  - etiquetas;
  - lista de trabalho;
  - interfaceamento;
  - laudo web;
  - valores de referência por sexo e idade;
  - painel de liberação com delta-check;
  - soroteca;
  - BI;
  - **assinatura avançada (1.4.21.1)**.

  Fica **fora** da lista, embora esteja no TR:
  - o CIQ (Westgard e Levey-Jennings, 1.4.8);
  - os valores de pânico (1.4.14.14);
  - a retificação (1.4.15.6, "segundo RDC 786/2023, com inclusão de motivo e geração de mensagem automática no
    laudo").
  [CONFIRMADO: E6]
- **E1 e E2:** as duas cidades do PR pedem o "GAL" só como **consulta do cidadão a resultado de COVID-19 via API da
  SESA-PR**, não como integração do laboratório. [CONFIRMADO: E1, E2]
- **E5 (São Carlos):** o laboratório é um sistema externo. O TR pede "integração com sistema laboratorial" e
  visualização no prontuário. É o arranjo do 21a. [CONFIRMADO: E5]
- **E7 e Araranguá:** o laboratório municipal **compra o LIS junto com reagentes e analisadores**, em comodato, ou
  exige que o fornecedor de reagente faça o interfaceamento com o sistema municipal. [CONFIRMADO]

  **Consequência [INFERÊNCIA]:** "sem interface com aparelhos" é defensável se o 21b oferecer uma **API de
  resultados**. O fornecedor de reagente ou o middleware do equipamento entrega por ela.
  - É o que pede o E3, item 929.
  - Cumpre a redação "Possibilitar a comunicação com sistemas de interfaceamento" do E1/E2.

---

## 4. GAL e LACEN: o que o laboratório municipal faz

### 4.1 Papel do município no GAL

- **Quem cadastra:**
  - O GAL faz o "Cadastro de pacientes e exames realizado pelos usuários (Municípios)".
  - Tem "Liberação dos resultados 'on line' pelos analistas responsáveis".
  - Permite "Impressão de Resultados diretamente no município solicitante".
  [CONFIRMADO: https://saude.es.gov.br/Media/sesa/LACEN/MANUAL_GAL_LACEN_ES.pdf]
- **Fluxo do usuário municipal** [CONFIRMADO: manual GAL-ES]:
  1. Cadastra a requisição, com dados do paciente, dados clínicos e número da notificação no SINAN.
  2. "Imprimir sempre a requisição para encaminhar junto com a ficha de solicitação do exame especifica e a
     **etiqueta por requisição**".
  3. **Triagem:** "Enc. Rede" encaminha ao laboratório da rede. "Descartar" é usado quando há não conformidade.
  4. No dia do envio, "Consultar Encaminhados" e emitir a **lista de protocolo** por laboratório de destino.
  5. Imprime o laudo, com abas "Laudos Parciais" e "Laudos finais".
  6. Consulta o andamento por requisição, paciente ou município.
- **Controle de qualidade de lâminas:**
  - Malária, esquistossomose e tuberculose: os controles **não** são cadastrados no GAL no ES.
  - Hanseníase: o CQ é cadastrado, com o resultado da lâmina do município nos dados clínicos.
  - Ou seja, o laboratório municipal que faz baciloscopia participa do **CQ por revisão de lâminas** do LACEN.
  [CONFIRMADO: manual GAL-ES]
- **Na prática, num hospital** (POP HU-UFGD, envio ao LACEN-MS, v2 de 24/01/2024) [CONFIRMADO: https://www.gov.br/hubrasil/pt-br/hospitais-universitarios/regiao-centro-oeste/hu-ufgd/acesso-a-informacao/pops-protocolos-e-processos/gad/pop-uacap-065-fluxo-de-amostras-encaminhadas-ao-lacen-v-2.pdf]:
  - O laboratório recebe a amostra com o pedido médico, a ficha GAL e, quando exigida, a ficha SINAN. Uma tabela
    lista quais exames exigem cada ficha.
  - **Rejeita** amostra:
    - sem pedido ou fichas;
    - com fichas incompletas;
    - sem identificação ou com identificação errada;
    - com material diferente do exigido;
    - em frasco inadequado;
    - mal conservada.
  - Cadastra a amostra no **seu próprio sistema (SIL) "somente para controle de envio"** e no GAL.
  - Escaneia e arquiva as fichas.
  - Acondiciona conforme a orientação do LACEN, com caixa térmica e gelox.
  - Envia em dias fixos. "Amostras enviadas sem suas respectivas fichas não serão processadas no LACEN".
  - O GAL é obrigatório para todos os municípios do MS desde a Resolução 040/SES/MS/2009.
- **Código de barras exigido pelo estado:**
  - No RS, a **Resolução CIB/RS 634/2024** define que as amostras enviadas aos laboratórios do Sistema Nacional
    de Laboratórios de Saúde Pública "passarão a ser identificados através de código de barras".
  - Porto Alegre comprou leitor, impressora térmica e etiqueta 55 × 27 mm para o SAE/CTA, com cerca de 42
    amostras por semana ao LACEN.
  [CONFIRMADO: ETP de Porto Alegre, https://pregaobanrisul.com.br/anexos/825195-d28633f270a6f573938182feb1dad871b3e7a2f1fd151721f9aed45886276f0a]
  - Não li a resolução nem o padrão de código (simbologia e conteúdo). [INFERÊNCIA: confirmar com o LACEN/RS]

### 4.2 Integração

- **Não há API pública do GAL para sistema municipal.**
  - Lacen-MG/Funed integrou o LIS Pixeon ao GAL: importa o cadastro e exporta resultados. Isso exigiu alterar a API
    do GAL com o DATASUS, porque "não temos o domínio completo do código fonte".
  - Os municípios continuam cadastrando e consultando no GAL.
  [INFERÊNCIA, só resumo de busca: https://www.agenciaminas.mg.gov.br/noticia/novo-sistema-laboratorial-agiliza-entrega-de-exames-no-laboratorio-de-saude-publica/]
- **Exceção no Paraná:** a SESA-PR oferece uma API de consulta de resultado de COVID-19, cobrada em E1 e E2.
  [CONFIRMADO: E1, E2]
- **O que já dizia a frente 3:** registrar o número da requisição GAL, anexar o laudo e alertar atraso. Ver
  `frente-3-exames.md` §1.4 e §5.

### 4.3 O que vai além de registrar o número GAL [INFERÊNCIA]

1. **Triagem local e rejeição** antes do envio, com motivo. O laboratório municipal é quem rejeita. Os motivos
   estão no POP do HU-UFGD.
2. **Remessa** ao LACEN: data, destino, lista de amostras, temperatura e responsável. Equivale à "lista de
   protocolo" do GAL, mas no Rota Saúde serve para conferência e prazo.
3. **Etiqueta** da amostra no padrão exigido pelo estado (RS), ou reaproveitar a etiqueta impressa pelo GAL.
4. **Ficha SINAN** vinculada, quando o agravo exige. Ligação com o módulo de vigilância.
5. **Laudo do LACEN** anexado (PDF do GAL), com ciência pelo solicitante. O 21a já faz isso.
6. **CQ de lâminas** (TB e hanseníase): registrar que a lâmina foi enviada para revisão e o resultado da
   concordância. Pontuável, não mínimo.

---

## 5. Síntese

### 5.1 Mínimo para passar na prova de conceito [INFERÊNCIA, salvo onde indicado]

Critério de corte: o requisito aparece em 5 ou mais dos 9 editais, ou é obrigatório no E3 ou na lista da prova do
E6, **ou** é exigência da RDC 978 que o RT vai cobrar.

1. **Cadastro de exames:**
   - SIGTAP 02.02, setor/bancada, material, recipiente, método e unidade;
   - **valores de referência por sexo e faixa etária**, casas decimais e limites de digitação (mín./máx.);
   - prazo de entrega e instruções de preparo.
2. **Recepção:**
   - importa o pedido do 21a ou cadastra avulso;
   - completa os campos do art. 134 (sexo biológico, nome da mãe) e do art. 136 (urgência, informações clínicas,
     CNES do coletor);
   - emite **comprovante com data prevista de entrega** (art. 137).
3. **Etiqueta com código de barras:**
   - uma por material ou tubo;
   - os 4 campos do art. 138 (nome, nascimento/idade, material, número de registro);
   - impressora térmica e **reimpressão**.
4. **Coleta e recebimento por leitura da etiqueta:**
   - quem coletou, data e hora, quem recebeu;
   - **aceite, rejeição ou aceite com restrição**, com motivo de uma lista do próprio laboratório (art. 139).
   - A restrição vai ao laudo.
5. **Mapa de trabalho por setor/bancada**, filtrando urgência, data e "não emitidos", em PDF.
6. **Digitação de resultados:**
   - por mapa ou por leitura da etiqueta;
   - **destaque fora da referência**, bloqueio fora dos limites de digitação e campos calculados simples;
   - lista de pendências.
7. **Conferência e liberação** pelo profissional habilitado:
   - papel "liberador" com conselho, UF e registro;
   - RT e substituto;
   - **bloqueio de impressão e de publicação antes da liberação**.
8. **Assinatura do laudo:** PAdES com ICP ou gov.br avançado, reaproveitando o 19b.
   - Dá conta do E3 (ICP, obrigatório), do E6 (avançada) e da RDC (art. 5º, III).
   - A "assinatura por senha" dos modelos E1/E2/E7 fica atendida por excesso.
9. **Laudo em PDF** com os 14 itens do art. 167, mais o CNES da fase analítica (art. 129).
10. **Publicação:**
    - no prontuário (21a);
    - no **wpda** (laudo do cidadão);
    - com **restrição de laudos sensíveis** (HIV e similares), que só o laboratório imprime.
11. **Rastreio de status** por número de requisição (recebido → em análise → digitado → liberado → entregue).
    **Controle de entrega**: quem retirou.
12. **Trilha de auditoria** de lançamento e alteração com usuário, data, hora e local, preservando o original
    (arts. 105 e 116).
13. **Retificação** por novo laudo versionado, com motivo e registro de contato com o paciente (art. 170).
    - Poucos editais pedem, mas a RDC obriga e o RT pergunta.
14. **Valores críticos:** marcação por exame, alerta na liberação e registro de quem foi avisado, quando e como
    (arts. 154-III e 165).
    - Mesmo motivo do item 13.
15. **Relatório de produção** por unidade, exame, profissional e mapa, com quantidade e valor SIGTAP.
16. **API de resultados** documentada para receber resultado de equipamento via middleware ou fornecedor de
    reagente.
    - É a resposta ao "interfaceamento" (6/9 editais) sem construir drivers.

### 5.2 Pontuável, mas opcional

| Item | Por que é opcional | Editais |
|---|---|---|
| Interfaceamento direto (ASTM/HL7, bidirecional, *request mode*) | O E3 marca NÃO e põe a responsabilidade no fornecedor do equipamento; o reagente costuma trazê-lo em comodato | 6/9, obrigatório na prova só no E6 |
| CIQ com Westgard e Levey-Jennings | A RDC exige o registro, não que ele esteja no LIS; fora da prova no E6 | 2/9 |
| Estoque de reagentes e insumos no laboratório | Existe um módulo de estoque genérico; só o E3 pede no laboratório | 1/9 |
| Integração GAL | Não há API. O mínimo útil é o registro da remessa e do laudo (§4.3) | 2/9 (só consulta COVID-PR) |
| Faturamento BPA da produção | O Rota Saúde já planeja BPA-I/BPA-C (frente 4). O laboratório só precisa gerar a produção com SIGTAP e CID | 3/9 |
| SMS ou e-mail de "resultado disponível" | Vem de graça se o 21b reutilizar as notificações do wpda (o SMS está desligado até o gate de go-live) | 2/9 |
| Laboratório de apoio (lote de envio e retorno) | Só o E3 e o E6 detalham; o 21a já cobre o prestador | 3/9 |
| Soroteca | Útil em laboratório grande | 2/9 |
| Resultados anteriores e delta-check | Ganho clínico, raramente obrigatório | 3/9 |
| Painel de produção/BI (dashboard) | Os relatórios bastam na maioria; BI só no E6 e no ICISMEP | 1/9 |
| Lote de transporte com temperatura e umidade | A RDC 504 se cumpre em papel | 2/9 |
| Biometria na recepção, filas e senhas | Específico de modelo de edital | 2/9 |
| Coleta domiciliar | Nenhum edital pede; a RDC permite com técnico capacitado | 0/9 |

---

## Riscos

1. **Prova por módulo com corte de 95–100%.**
   - Os editais vêm de modelos de concorrentes, e a redação favorece quem escreveu o modelo: "impressão direta por
     porta de impressora", "biometria", "assinatura digitalizada do solicitante".
   - Um item exótico pode derrubar o módulo.
   - Mitigação: pedir esclarecimento ou impugnar itens direcionados, e manter uma matriz de aderência por modelo
     (E1/E2 e E8/E9). [INFERÊNCIA]
2. **Interfaceamento.** Aparece em 6 de 9 editais. Sem a API do item 16, o 21b perde pontos ou é desclassificado
   onde ele for obrigatório. [INFERÊNCIA]
3. **Assinatura por senha não é "avançada".** Se o 21b oferecer só liberação por senha, o laudo fica fora do art.
   5º, III da RDC. Se o liberador for médico, fica também fora do CFM 2.299. [INFERÊNCIA]
4. **Revisão da RDC 978.** O processo de alteração está aberto (Voto ROP 12/2025). Os números de artigo podem
   mudar. Conferir o texto consolidado no AnvisaLegis antes de citar artigos na interface ou no dossiê.
   [INFERÊNCIA]
5. **RNDS/REL.**
   - A Portaria GM/MS 8.276/2025 manda todo resultado laboratorial seguir o REL e ir à RNDS (frente 3).
   - Se o Rota Saúde virar o LIS do laboratório municipal, **o envio passa a ser responsabilidade do sistema**.
   - Hoje a especificação cobre só SARS-CoV-2 e Orthopoxvírus.
   - Pode virar exigência de edital ou de fiscalização. Planejar para o v2. [INFERÊNCIA]
6. **Fronteira entre 21a e 21b.** O 21a trata o laboratório como prestador externo. No 21b ele é executante interno.
   - Se o mesmo exame existir nos dois modelos, haverá duplicidade de laudo.
   - Precisa de regra: o executante é laboratório próprio (21b) **ou** prestador (21a), por exame e por cidade.
   [INFERÊNCIA]
7. **Guarda.** O art. 115 fixa 5 anos para os registros. O laudo no prontuário segue a guarda de prontuário (20
   anos, Lei 13.787). Não pode haver purga de laudo antes disso. [INFERÊNCIA]
8. **Art. 106 (validação do software pelo serviço).** Sem um dossiê do fornecedor, a VISA pode autuar o
   laboratório. É um custo de implantação. [INFERÊNCIA]

## Perguntas em aberto

1. Qual laboratório municipal será o piloto? Ele já tem LIS em comodato com o fornecedor de reagente? Se tiver, o
   21b concorre com um sistema "grátis".
2. Os liberadores do piloto têm certificado ICP? São farmacêuticos, biomédicos ou médicos? O RT aceita gov.br
   avançado?
3. Os editais-alvo (SC/PR) usam qual família de modelo? A matriz da seção 3 cobre duas famílias (E1/E2 e E8/E9) e
   o modelo CISLESTE.
4. Texto e padrão técnico da CIB/RS 634/2024 (simbologia e campos da etiqueta). Outros estados exigem algo
   parecido para envio ao LACEN?
5. Instrumento de registro (BPA-C ou BPA-I) dos procedimentos 02.02 no SIGTAP de 2026. O site do SIGTAP estava fora
   do ar nesta pesquisa; a frente 4 trata do layout.
6. Lista de valores críticos: quem define é o RT de cada laboratório (art. 154). O 21b deve trazer uma lista
   sugerida, como a da SBPC/ML, ou deixar vazia?
7. O CFBM tem norma sobre assinatura eletrônica de laudo? Não localizei.
8. A revisão da RDC 978 anunciada pela Anvisa já foi publicada em 2026? Não localizei.
9. O laudo do 21b entra no prontuário (20 anos) ou fica só no laboratório (5 anos)? Decisão de arquitetura e
   jurídica.

## Fontes

Norma sanitária
- RDC 978/2025 (DOU): https://www.in.gov.br/en/web/dou/-/resolucao-anvisa-n-978-de-6-de-junho-de-2025-635044217
- P&R RDC 978, 1ª ed. (Anvisa/GGTES): https://www.sbac.org.br/wp-content/uploads/2025/08/PR__978_2025_1a-versao-revisada_Portal_27ago2025.pdf
- VISA Joinville, informe sobre a 978: https://www.joinville.sc.gov.br/wp-content/uploads/2025/09/Informe-sobre-a-RDC-no-978-2025-Servicos-que-executam-atividades-relacionadas-a-Exames-de-Analises-Clinicas-EAC.pdf
- Voto ROP 12/2025 (só resumo de busca): https://www.gov.br/anvisa/pt-br/composicao/diretoria-colegiada/reunioes-da-diretoria/votos/2025/rop-12.2025/2-12.pdf

Conselhos
- CFF 296/1996: https://www.cff.org.br/userfiles/file/resolucoes/296.pdf
- CFBM 78/2002 (extrato): https://www.legisweb.com.br/legislacao/?id=98598
- CFBM 399/2025: https://www.normasbrasil.com.br/norma/resolucao-399-2025_481877.html
- CFM 2.235/2019: https://sistemas.cfm.org.br/normas/arquivos/resolucoes/BR/2019/2235_2019.pdf
- CFM 2.299/2021: https://abmes.org.br/public/arquivos/legislacoes/Resolucao-CFM-2299-2021-09-30.pdf

Editais
- E1 Pato Branco/PR: https://patobranco.pr.gov.br/wp-content/uploads/2023/04/41-SOFTWARE-GESTAO-DE-SAUDE-PUBLICA.pdf
- E2 Contenda/PR: https://www.contenda.pr.gov.br/prefeitura_app/storage/app/public/licitacoes/Edital-Sistema-de-Gestao-Saude.pdf
- E3 CISLESTE/MG: http://www.cisleste.mg.gov.br/portal/wp-content/uploads/2025/07/PREGAO-08-25-SISTEMA-SAUDE-TR.pdf
- E4 Xanxerê/SC: https://xanxere.sc.gov.br/uploads/sites/92/2022/06/Termo-de-Referencia-Sistema-de-Gestao-Saude.pdf
- E5 São Carlos/SP: https://servico.saocarlos.sp.gov.br/licitacao/arquivos-resumos/07-16-2024-04:46:59-pm-8345-6866--Resumo-.pdf
- E6 HSPM/SP: https://capital.sp.gov.br/documents/d/hospital_do_servidor_publico_municipal/90412-2024-pdf
- E7 Santa Izabel do Pará/PA: https://santaizabel.pa.gov.br/wp-content/uploads/2021/05/ARQUIVO_1538768543.pdf
- E8 Mata de São João/BA: https://www.matadesaojoao.ba.gov.br/Handler.ashx?f=f&query=f9776ff0-6257-466b-8bca-8c3eed23d973.pdf&name=8_TERMO_DE_REFERENCIA.pdf
- E9 Dionísio Cerqueira/SC: https://dionisiocerqueira.sc.gov.br/uploads/sites/267/2024/03/SISTEMA-LICITACAO-ANEXO-II.pdf
- ICISMEP/MG (ata LIS): https://icismep.mg.gov.br/wp-content/uploads/2025/01/Catalogo-PL-03.2025-Software-LIS.pdf
- Araranguá/SC (reagentes + interfaceamento): https://ararangua.sc.gov.br/uploads/sites/281/2025/09/Edital-de-Licitacao-n-252025-Insumos-e-Reagentes-lab-municipal.pdf
- Joinville/SC (guia iPSO de integração LIS): https://www.joinville.sc.gov.br/public/edital/anexo/98e995990ba1f5fad232da1df1b70fb4.pdf

GAL e LACEN
- Manual GAL, LACEN-ES: https://saude.es.gov.br/Media/sesa/LACEN/MANUAL_GAL_LACEN_ES.pdf
- POP HU-UFGD, envio ao LACEN-MS: https://www.gov.br/hubrasil/pt-br/hospitais-universitarios/regiao-centro-oeste/hu-ufgd/acesso-a-informacao/pops-protocolos-e-processos/gad/pop-uacap-065-fluxo-de-amostras-encaminhadas-ao-lacen-v-2.pdf
- ETP Porto Alegre (CIB/RS 634/2024): https://pregaobanrisul.com.br/anexos/825195-d28633f270a6f573938182feb1dad871b3e7a2f1fd151721f9aed45886276f0a
- Lacen-MG integra LIS ao GAL (só resumo de busca): https://www.agenciaminas.mg.gov.br/noticia/novo-sistema-laboratorial-agiliza-entrega-de-exames-no-laboratorio-de-saude-publica/
- Página GAL, SES-GO: https://goias.gov.br/saude/sistema-gerenciador-de-ambiente-laboratorial-gal/

Interno
- `docs/pesquisa/2026-10-05-integracoes/frente-3-exames.md` (arranjos, RNDS/REL, GAL)
- `docs/pesquisa/2026-10-05-integracoes/frente-4-faturamento-regulacao.md` (BPA)
- `docs/pesquisa/2026-10-08-assinatura-digital-19b.md` (PAdES, Lei 14.063)
