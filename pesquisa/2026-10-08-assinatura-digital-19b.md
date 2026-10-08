# Assinatura digital do prontuário (subprojeto 19b): pesquisa técnica e regulatória

Data: 2026-10-08. Somente leitura na web. Legenda: **[CONFIRMADO]** = lido na fonte primária citada; **[SECUNDÁRIO]** = fonte secundária, conferir o texto oficial; **[INFERÊNCIA]** = conclusão nossa.

Não repete o que já estava levantado (CFM 1.821/2007, selo SBIS voluntário, Lei 14.063/2020, CFM 2.299/2021, Lei 13.787/2018, gov.br via órgão público).

---

## 1. NGS2: o que vai além da assinatura

Fonte lida: *Requisitos para Certificação de S-RES, categoria Segurança da Informação, v5.1 (29/03/2021)*, seção 3.2: https://sbis.org.br/certificacao/Requisitos_Certificacao_SBIS_Seguranca_V5.1.pdf. Os certificados emitidos em 2025–2026 citam o "Manual de Certificação 2021 – versão 5.2" (ex.: https://sbis.org.br/wp-content/uploads/2026/08/Certificado_SBIS_SRES_202_InterSystems.pdf), e há uma v5.3 anunciada. **[INFERÊNCIA]** O bloco NGS2 é o mesmo entre categorias, mas convém baixar a 5.2/5.3 antes do ADR.

Requisitos NGS2 [CONFIRMADO, v5.1]. A coluna "estágio" é o estágio de maturidade (1–3); "todos" significa obrigatório já no estágio 1.

| ID | Requisito | Estágio |
|---|---|---|
| 01.01 | Usar certificado ICP-Brasil para assinar documentos do prontuário | todos |
| 01.02 | Exigir que o CPF do certificado seja igual ao CPF do cadastro do usuário, a cada uso | todos |
| 01.03 | Validar cadeia e revogação **no servidor**, contra raízes configuradas no servidor | todos |
| 01.04 | Tela restrita para incluir e excluir raízes confiáveis | 3 |
| 01.05 | Assinar com certificados de **≥ 2 ACs de 1º nível distintas por tipo de mídia suportada** (cartão, token, HSM, software, **PSC**) | todos |
| 02.01 | Gerar **CAdES, XAdES ou PAdES, no mínimo na política AD-RB** | todos |
| 02.02 | Conferir keyUsage (digitalSignature + nonRepudiation) e que o certificado é do tipo A1–A4 | todos |
| 02.03 | Gravar o instante da assinatura (`signingTime` no CAdES; entrada `M` no PAdES) | todos |
| 02.04 | Mostrar antes de assinar **exatamente** o conteúdo que será assinado | todos |
| 02.05/06 | Criar **pendência de assinatura**; avisar ao sair da tela e no logoff; listar as pendências do profissional após o login | 2–3 / todos |
| 02.08 | Se OCSP, LCR ou carimbo estiverem fora do ar: deixar a assinatura pendente ou registrar a pendência e completar depois | 3 |
| 02.09 | Indicar "assinado" e mostrar quem assinou e quando | todos |
| 02.10/11 | **Encadeamento**: garantir a ordem temporal e a presença de todos os registros assinados de cada paciente, com verificação disponível ao usuário | 3 |
| 03.01–04 | Validar a assinatura antes de gravar, logo após assinar, ao imprimir, ao importar e exportar, e quando o usuário pedir; validar **todas as co-assinaturas**; resultado Válida/Inválida/Indeterminada com motivo; o estado da assinatura sai na impressão | todos |
| 03.02/03 | Revogação conferida no instante do `signingTime` (sem carimbo) ou do carimbo (com carimbo) | todos / 2–3 |
| 04.01 | **Política mínima AD-RT** (com carimbo), embutindo os objetos de validação ou guardando-os localmente com garantia de recuperação | 2–3 |
| 04.02 | Carimbo de **ACT credenciada ICP-Brasil**, aplicado logo após a assinatura e revalidado | 2–3 |
| 04.03/04 | Parametrizar se há carimbo e para quais tipos de documento (no mínimo receita e atestado) | 2–3 / 3 |
| 05.x | Certificado de atributo (DOC-ICP-16): opcional, condicional | 2–3 |
| 06.03 | Exportar os registros assinados de forma validável fora do sistema (ex.: verificador do ITI) | todos |
| 06.04 | Receita, atestado, solicitação de exame e laudo seguem a especificação SBIS "Exportação de Documentos Assinados Digitalmente" | 2–3 |
| 06.05–07 | Impressão com **rodapé** padronizado (signatário, hora em UTC, estado) ou com **relatório de assinaturas** | todos |
| 07.01 | Login por certificado (opcional): validar vigência, cadeia, revogação, CPF e EKU clientAuth | todos |

O NGS2 só vale **em cima do NGS1 inteiro** [CONFIRMADO]. O NGS1 cobre controle de versão do software, identificação e autenticação (bloqueio de conta e de sessão por inatividade, com aviso), autorização e controle de acesso, disponibilidade (**backup completo, restauração e integridade do backup**), comunicação entre componentes, segurança de dados, **auditoria contínua e inalterável** com eventos mínimos, documentação, **tempo** (sincronismo), privacidade e integridade.

**Para eliminar o papel sem buscar o selo** [INFERÊNCIA]: CFM 1.821 (art. 3º–5º), CFO 91/2009 e COFEN 754/2024 exigem "atender integralmente" ao NGS2. O selo é só a prova de conformidade. Sem o selo, o ônus de comprovar fica com a instituição e o diretor técnico. O CRM-MG (Parecer 140/2018: https://sistemas.cfm.org.br/normas/arquivos/pareceres/MG/2018/140_2018.pdf) nega a eliminação do papel sem atendimento integral. Na prática o piso é NGS1 inteiro mais a tabela acima no estágio pretendido. O estágio 1 dispensa AD-RT, carimbo e encadeamento.

## 2. Assinatura em nuvem (PSC ICP-Brasil)

**Padrão** [CONFIRMADO]: o ITI **não usa o CSC (Cloud Signature Consortium)**. Ele define uma API própria, a "v0", no **DOC-ICP-17.01**. A versão vigente é a 3.0, aprovada pela IN ITI 20/2020 (https://www.gov.br/iti/pt-br/assuntos/legislacao/instrucoes-normativas/IN_20_2020_DOC_17.01_assinada.pdf). Também lida a v2.3, com o mesmo texto da API: https://www.gov.br/iti/pt-br/assuntos/legislacao/documentos-principais/DOCICP17.01verso2.3PROCEDIMENTOSOPERACIONAISMNIMOSPARAOSPRESTADORESDESERVIODECONFIANADAICPBRASIL.pdf.

- **Fluxo**: OAuth 2.0 com **PKCE S256 obrigatório**.
  1. `GET <base>/v0/oauth/authorize` com `client_id`, `redirect_uri`, `state`, `scope`, `lifetime` (em segundos) e `login_hint` (CPF).
  2. O titular aprova no app do PSC (VIDaaS, BirdID etc.). A aplicação **não pode coletar os fatores de autenticação**; a exceção é o serviço opcional `pwd_authorize`, com OTP.
  3. `POST /oauth/token`.
  4. `POST /oauth/signature`.
- **Escopos**: `single_signature` (1 hash), `multi_signature` (**N hashes numa requisição**, um único uso), `signature_session` (várias chamadas enquanto o token valer) e `authentication_session`.
- **Duração**: `lifetime`/`expires_in` têm **teto de 7 dias para pessoa física** (30 dias para PJ). O PSC pode limitar abaixo disso. **[INFERÊNCIA]** O "até 12 h" é configuração de cada PSC ou app, não regra do ITI.
- **Assinatura de hash** [CONFIRMADO]: o sistema envia `hashes[]` com `id`, `alias`, `hash` em base64, `hash_algorithm` (OID, ex.: SHA-256 = 2.16.840.1.101.3.4.2.1) e `signature_format`:
  - `RAW`: devolve a assinatura PKCS#1 em base64;
  - `CMS`: devolve um CMS destacado (detached) com **apenas** `contentType`, `signingTime` (hora do PSC), `messageDigest` e `signingCertificateV2`.
- **Consequência** [INFERÊNCIA forte]: o CMS do PSC **não traz o identificador de política (`SignaturePolicyIdentifier`)**. Logo, não é AD-RB ICP-Brasil, e no PAdES o `signingTime` dentro dos atributos assinados conflita com o perfil. O caminho correto é:
  1. buscar o certificado pelo serviço obrigatório "Recuperação de Certificado";
  2. montar localmente os atributos assinados (política, signingCertificateV2, messageDigest);
  3. pedir assinatura `RAW` do hash desses atributos;
  4. montar o CMS ou o PAdES no servidor.

  É o que fazem as bibliotecas ICP.
- **Serviços obrigatórios de todo PSC** [CONFIRMADO]: autorização, token, assinatura, **cadastro de aplicação com certificado SSL ICP-Brasil**, listagem de certificados do titular, **localização de titular por CPF** (`/oauth/user-discovery`) e recuperação de certificado.
- **Vários PSCs** [INFERÊNCIA]: como o protocolo é o mesmo, um único cliente fala com todos. Para cada PSC é preciso registrar a aplicação e guardar credenciais, e há variações de comportamento. Para descobrir o PSC do profissional, consulta-se o `user-discovery` de cada um.
- **Agregadores comerciais**:
  - **Certillion** [CONFIRMADO, https://certillion.com/en/api/online-api/]: uma API só para VIDaaS, BirdID, SafeID, NeoID, RemoteID e VaultID. Usa os mesmos escopos e `lifetime`, assina em lote por `hashes[]` com uma única autenticação, gera `PADES_ICP_BR`, CAdES e XAdES com as políticas AD-RB, AD-RT, AD-RV, AD-RC e AD-RA, e também valida. Preço não publicado.
  - **Lacuna** tem conector de PSC no PKI SDK [SECUNDÁRIO].
- **Custo por profissional** [SECUNDÁRIO]:
  - e-CPF A3 em nuvem NeoID/Serpro: **R$ 89,90 por 3 anos**; e-CPF A3 em token: R$ 391–413 (TR do IFPI: https://www.ifpi.edu.br/acesso-a-informacao/licitacoes-e-contratos/contratos-de-ti/pasta-contratos-de-ti/certificado-digital-tr).
  - BirdID: modelo por créditos de uso.
  - Mercado em geral: R$ 160–600.
  - **Médicos adimplentes têm certificado em nuvem ICP gratuito do CFM, via VIDaaS** (Res. CFM 2.296/2021: https://portal.cfm.org.br/noticias/cfm-lanca-oficialmente-certificado-digital-gratuito-para-todos-os-medicos-brasileiros/).
  - Órgão público pode contratar o Serpro por dispensa (Lei 14.133, art. 75, IX).

## 3. Certificado A3 (token ou cartão) num app web

Opções:
- **Lacuna Web PKI**: extensão de navegador + componente nativo, com licença por domínio passada no construtor (https://npmjs.com/package/web-pki). Preço só sob consulta; contrato do TCE-SP para a suíte Lacuna PKI: R$ 50 mil por 48 meses [SECUNDÁRIO].
- **Assinador SERPRO**: desktop, gratuito, com assinatura em lote; o navegador fala com ele em `assinador-desktop.serpro.gov.br:65166` (https://www.serpro.gov.br/links-fixos-superiores/assinador-digital/assinador-serpro). Relatos indicam atrito entre navegadores [SECUNDÁRIO].
- **Extensões próprias de fornecedores**: ex.: "Crescer", da Vivver, em Sorocaba.

**Viabilidade na APS** [INFERÊNCIA]: baixa. Exige estações compartilhadas com drivers e leitores instalados, e tokens se perdem. O próprio mercado público foge disso: o TR da FUABC **proíbe** exigir token USB, smartcard ou leitor, e pede API, assinatura em sessão e autenticação pelo celular (ver §8). Sorocaba aceita A3, mas o custo fica com o profissional. Recomendação: **deixar o A3 fora do 19b** e atender só PSC em nuvem.

## 4. Formatos, políticas e carimbo do tempo

- **Formatos**: CAdES, XAdES e PAdES pelo **DOC-ICP-15.03**. A versão 8.0 veio pela IN 03/2021 (https://www.gov.br/iti/pt-br/assuntos/legislacao/instrucoes-normativas/IN032021_DOC_15.03_assinada.pdf), seguida da compilação v9.0 pela IN 33/2025. A **IN 34/2025** atualizou as políticas PAdES (AD-RB v1.3, AD-RA v1.4) [SECUNDÁRIO: https://okai.com.br/documento/2025-07-21/instrucao-normativa-iti-nº-34-de-17-de-julho-de-2025]. Os OIDs e hashes das políticas vêm da **LPA** do ITI: nunca fixar no código, ler da LPA [INFERÊNCIA].
- **Políticas**:
  - AD-RB: básica;
  - AD-RT: com carimbo do tempo;
  - AD-RV: com referências para validação, **só CAdES e XAdES**;
  - AD-RC: referências completas;
  - AD-RA: arquivamento, com recarimbo periódico.
  
  O **PAdES não tem AD-RV**, mas admite carimbo sobre o documento inteiro (IN 05/2015) [SECUNDÁRIO].
- **Carimbo do tempo**: na ICP-Brasil ele é **facultativo**; a assinatura vale sem ele [SECUNDÁRIO, requisitos de ACT]. O NGS2 o exige a partir do estágio 2 (AD-RT). Há ACTs credenciadas com API (Serpro: https://www.serpro.gov.br/links-fixos-superiores/pss-serpro/actserpro/; Prodesp). Não há preço público; é contrato por volume.
- **validar.iti.gov.br**: aceita CAdES, XAdES e PAdES, embutidos ou destacados [SECUNDÁRIO, Portaria ITI 22/2023]. O CFM 2.299 exige que o documento possa ser validado no ITI.
- **20 anos de guarda** [INFERÊNCIA]: o certificado vence em 1 a 5 anos e as cadeias e LCRs somem. Recomendações:
  - assinar com AD-RB e aplicar o carimbo ACT logo em seguida (AD-RT);
  - **guardar** cadeias, LCRs e respostas OCSP no momento da assinatura (permitido pela Nota 1 do NGS2.04.01);
  - para PDFs, usar LTV/DSS com carimbo de documento;
  - ter um job de **recarimbo** (AD-RA) antes de vencer o certificado da ACT.

  PAdES serve para o documento que vai ao paciente. CAdES destacado serve para o registro estruturado.

## 5. Bibliotecas e serviços

| Opção | Licença e custo | ICP-Brasil | Observação |
|---|---|---|---|
| **Ruby**: OpenSSL::ASN1/PKCS7 | stdlib | manual | Dá para montar CMS com atributos próprios; trabalho artesanal e arriscado |
| **HexaPDF** (Ruby) | AGPL ou comercial (obrigatória em SaaS fechado; preço sob consulta) | não nativo | PAdES B-B a B-LTA, assinatura externa e assíncrona, carimbo de documento (https://hexapdf.gettalong.org/documentation/digital-signatures/index.html). Política ICP não documentada |
| **rustpdf** (gem) | MIT | diz suportar AD-RB (`SignaturePolicy`) | Primeira versão em 2026, cerca de 8 mil downloads: **imaturo** |
| **Demoiselle Signer** (Java, Serpro) | LGPL-3, grátis | **sim**, nativo (políticas e cadeia ICP) | v4.6.x no Maven Central; github.com/demoiselle/signer |
| **EU DSS** (Java) | LGPL-2.1 | configurável | Referência em PAdES/CAdES/XAdES e validação LTV; políticas ICP exigem configuração |
| **iText** (Java/.NET) | AGPL ou comercial (caro) | não nativo | Evitar pela licença |
| **pyHanko** (Python) | MIT | não nativo | Forte em PAdES, assinatura interrompida/externa, LTV e validação. Política ICP exige atributo customizado [INFERÊNCIA] |
| **Lacuna Rest PKI / PKI SDK** | comercial | **sim** | Cliente Ruby existe [SECUNDÁRIO, TCE-RN]; conector PSC |
| **Certillion** | comercial (planos) | **sim** (PADES_ICP_BR) | Agrega PSCs, faz lote e valida |
| **BRy** | comercial | sim | Plataforma SaaS e API; docs públicas escassas |

**Recomendação para o Rails** [INFERÊNCIA]. Trade-offs:
- **(a) Comprar**: Certillion ou Lacuna. Mais rápido, já cobre PSCs, políticas e validação. Contra: custo por assinatura, dependência do fornecedor, e o PDF e o hash saem da nossa infraestrutura (no Certillion o documento é enviado).
- **(b) Montar**: o Rails conversa direto com os PSCs (OAuth "v0", que é simples) e um **sidecar de assinatura** em container monta o CMS/PAdES ICP e valida.
  - Demoiselle Signer, ou DSS com políticas ICP, em Java;
  - ou pyHanko com atributo de política, em Python.

  A favor: nada sai da cidade a não ser o hash, custo zero de licença, e combina com o modelo de um banco por cidade. Contra: esforço de conformidade (testar no Verificador de Conformidade do ITI) e manter a LPA e as cadeias.

A preferência é **(b) com Demoiselle**, por ser a opção mais simples que ainda dá conformidade. Fica a ressalva de prototipar antes **um** PSC (VIDaaS, por causa do CFM gratuito) e validar no ITI. Ruby puro não tem nada maduro para PAdES ICP.

## 6. Conselhos

- **COFEN 754/2024** [CONFIRMADO, texto integral: https://cofen.gov.br/wp-content/uploads/2024/05/Resolucao-Cofen-no-754-2024-Normatiza-o-uso-do-prontuario-eletronico-e-plataformas-digitais-no-ambito-da-Enfermagem.pdf]:
  - art. 2º §1º: o prontuário é *paperless* só com **assinatura qualificada ICP-Brasil do profissional responsável**;
  - §2º: admite login e senha;
  - §3º: sem a assinatura qualificada, **imprime e assina**;
  - §4º: S-RES em NGS2 dispensa a impressão;
  - art. 3º: os profissionais de enfermagem podem usar qualquer certificado ICP;
  - **art. 4º: quando a instituição adota prontuário eletrônico, *não cabe aos profissionais de Enfermagem adquirir os certificados*.**
  - **Técnico e auxiliar** [INFERÊNCIA]: o texto fala em "profissionais de Enfermagem" e "profissional responsável", sem excluir o técnico. Para eliminar o papel das anotações do técnico, **ele também precisa de ICP**, e o município paga.
- **COFEN 801/2026** (prescrição por enfermeiro): segundo resumo, o art. 4º admitiria prontuário digital com assinatura **avançada ou qualificada** [SECUNDÁRIO, conferir: https://www.portalcofen.gov.br/wp-content/uploads/2026/01/Resolucao-Cofen-no-801-2026-1.pdf]. Isso cria tensão com a 754 e precisa de leitura jurídica.
- **CFO 91/2009** [SECUNDÁRIO: https://sistemas.cfo.org.br/visualizar/atos/RESOLU%C3%87%C3%83O/SEC/2009/91]: mesma lógica do CFM. Papel só sai com NGS2 e ICP; NGS1 não basta.
- **COFFITO 414/2012, CFN 594/2017, CFP 001/2009** [SECUNDÁRIO]: aceitam registro eletrônico, exigem nome e número do conselho, e **não há regra própria de ICP**. Para eMulti valem a Lei 13.787 e o NGS2 se a meta for eliminar papel [INFERÊNCIA].

## 7. Adendo, contra-assinatura e o que se assina

[INFERÊNCIA, apoiada nos requisitos NGS2.02.10/11 e 03.01c]
- **Adendo**: é documento novo, assinado por quem o escreveu, contendo o **hash do registro original assinado**, ou do último elo da cadeia do paciente. A cadeia por paciente (hash do anterior + carimbo) atende ao "encadeamento" do NGS2.02.10 sem reabrir o original. O 19a já tem consulta imutável com adendos, e encaixa direto.
- **Contra-assinatura** (ex.: técnico registra, enfermeiro confere): mais de um `SignerInfo` no mesmo CAdES, ou várias assinaturas incrementais no PAdES. Todas precisam ser validadas (03.01c), e o rodapé lista todos os signatários.
- **Estruturado vs PDF**:
  - assinar o **registro canônico**, um JSON em JCS/RFC 8785 com versão do esquema, em **CAdES destacado AD-RB/RT**. É a fonte de verdade, sobrevive a mudanças de layout e é exportável e validável no ITI (enviando o .p7s e o JSON);
  - gerar **PAdES** só para o que **sai para o paciente ou para terceiros** (atestado, receita, encaminhamento: 19c/19d e CFM 2.299);
  - com `multi_signature` ou `signature_session`, **uma única aprovação no celular assina o JSON e o PDF juntos** (2 hashes).

## 8. Quem paga e o que os editais pedem

- **Enfermagem**: por norma, paga a instituição (COFEN 754, art. 4º) [CONFIRMADO].
- **Médicos**: o certificado em nuvem do CFM é gratuito; os municípios se apoiam nele [CONFIRMADO, Sorocaba].
- **Sorocaba (SES, SISWEB/Vivver, 2025–2026)** [CONFIRMADO: https://estante-ses.sorocaba.sp.gov.br/books/manuais-de-utilizacao-do-sisweb/page/diretrizes-institucionais-sobre-assinatura-digital-no-sisweb/export/pdf]:
  - desde 01/12/2025 oferece duas vias gratuitas: **gov.br Prata/Ouro**, integrado pelo município com o fornecedor, e **certificado CFM via VIDaaS**;
  - A1 em .pfx e A3 são aceitos, **por conta do profissional**;
  - a assinatura passa pela extensão de navegador "Crescer".
  
  É a prova de que o município consegue integrar o gov.br para um SaaS de saúde.
- **FUABC / Complexo Hospitalar Municipal de São Caetano do Sul** (TR 2023: https://fuabc.org.br/wp-content/uploads/2023/06/TR-CERTIFICADO-DIGITAL.pdf) [CONFIRMADO]:
  - a instituição contrata **cerca de 2.200 e-CPF A3 em HSM na nuvem por ano**, válidos por 5 anos;
  - **sem token USB, smartcard ou leitor**; autenticação por celular, web ou OTP;
  - **API e assinatura em sessão**, integradas ao PEP (MV);
  - justificativa: sem certificado, tudo é impresso.
- **Órgãos públicos em geral**: TRE-MG, IFPI e Fiocruz/AGHU compram e-CPF em nuvem para servidores (3 anos), muitas vezes do Serpro por dispensa [SECUNDÁRIO].
- **TRs de PEP** costumam pedir "assinatura qualificada ICP-Brasil" e portabilidade dos dados. A exigência de selo SBIS é contestável em impugnação, porque não há lei que a imponha [SECUNDÁRIO: https://agenciasus.org.br/shared-files/31565/].
- **Receitas (19c/19d)** [SECUNDÁRIO: https://prescricaoeletronica.cfm.org.br/rdc-1000.html]: pela **RDC Anvisa 1.000/2025**:
  - controlados (Portaria 344) exigem **qualificada ICP**;
  - antimicrobianos e GLP-1 aceitam **avançada** (gov.br);
  - a receita de controlado tem de ser **nativamente digital, emitida em sistema integrado ao SNCR por API**;
  - prazo de integração prorrogado para 30/09–30/10/2026 (datas conflitantes; conferir na Anvisa).

## Fontes principais
- SBIS, Requisitos Segurança v5.1: https://sbis.org.br/certificacao/Requisitos_Certificacao_SBIS_Seguranca_V5.1.pdf
- DOC-ICP-17.01 v3.0 (IN ITI 20/2020): https://www.gov.br/iti/pt-br/assuntos/legislacao/instrucoes-normativas/IN_20_2020_DOC_17.01_assinada.pdf
- COFEN 754/2024: https://cofen.gov.br/wp-content/uploads/2024/05/Resolucao-Cofen-no-754-2024-Normatiza-o-uso-do-prontuario-eletronico-e-plataformas-digitais-no-ambito-da-Enfermagem.pdf
- Certillion API: https://certillion.com/en/api/online-api/
- HexaPDF signatures: https://hexapdf.gettalong.org/documentation/digital-signatures/index.html
- rustpdf: https://rubygems.org/gems/rustpdf
- Demoiselle Signer: https://central.sonatype.com/artifact/org.demoiselle.signer/chain-icp-brasil
- DOC-ICP-15.03 (IN 03/2021): https://www.gov.br/iti/pt-br/assuntos/legislacao/instrucoes-normativas/IN032021_DOC_15.03_assinada.pdf
- Sorocaba SISWEB, FUABC TR, IFPI TR, CFM certificado gratuito, RDC 1.000 (links no texto)
