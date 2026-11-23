# BNAFAR mede estoque, não dispensação: o 27b muda de forma

O 27b não é uma fila de dispensações. O envio federal obrigatório desde a Portaria GM/MS 5.713/2024 tem dois registros diários (D-1) ao SI-BNAFAR, uma API REST/JSON do DATASUS. O REPE informa a posição de estoque por estabelecimento e lote. O RESMPE informa as saídas por transferência ou perda. A entrega ao paciente não entra nesse canal: ela vai à RNDS pelo modelo REDFM, cujos artefatos ainda não estavam publicados em abril de 2025 ([FATO] [Portaria 5.713/2024](https://bvsms.saude.gov.br/bvs/saudelegis/gm/2024/prt5713_17_12_2024.html); [FATO] [FAQ SI-BNAFAR](https://webatendimento.saude.gov.br/faq/sibnafar)). Para os insumos, a norma nacional só traz parâmetros clínicos e deixa as cotas para os municípios. Por isso o catálogo precisa de programas configuráveis por cidade, com elegibilidade, cota, validade da autorização e comodato do glicosímetro, e não de constantes nacionais ([FATO] [Portaria GM/MS 2.583/2007](https://bvsms.saude.gov.br/bvs/saudelegis/gm/2007/prt2583_10_10_2007.html) [ANTIGO, vigente]; [FATO] [SMS-SP, Protocolo 02/2026](https://autodromodeinterlagos.prefeitura.sp.gov.br/documents/d/saude/diagramacao-pa-glicemico_v3-pdf)). O razão do 27a já sustenta as duas entregas: um único estoque de lotes e movimentos, só de acréscimo, com um molde testado para ganhar mais um tipo de item. Faltam, porém, quatro coisas exigidas pelo REPE: valor unitário, CNPJ do fabricante, registro ANVISA e programa de saúde. Falta também um CNES para o almoxarifado central, e o código OBM está bloqueado por um CHECK ([FATO] `apps/api/db/city_schema.rb:4276-4293`, `4256`; [FATO] `apps/api/db/platform_schema.rb:387`). O prazo de adequação ao SI-BNAFAR venceu em 17/06/2025 ([FATO] [Informe CIT 24/04/2025](https://conasems-ava-prod.s3.sa-east-1.amazonaws.com/institucional/noticias/2-7-apresentacao-informe-cit-bnafar-24-04-2025-1745608921.pdf)). O Hórus para de aceitar lançamentos por volta de meados de dezembro de 2026, o que abre uma janela de mercado de semanas, não de meses ([INFERÊNCIA], contando 180 dias desde a [Portaria GM/MS 11.585, de 16/06/2026](https://www.conass.org.br/conass-informa-n-114-2026-publicada-a-portaria-gm-n-11-585-que-altera-a-portaria-de-consolidacao-gm-ms-no-1-de-28-de-setembro-de-2017-para-dispor-sobre-a-substituicao-do-sistema-hor/)).

Convenções: [FATO] vem com URL, ou com `arquivo:linha` para código e docs locais, com caminhos relativos a `rota-saude/`. [INFERÊNCIA] é dedução. [ANTIGO] marca fonte anterior a 2024.

## Só diabetes e estomia têm regra nacional, e nenhuma fixa cota

### Diabetes: a norma federal define o público, os municípios definem as quantidades

A base federal dos insumos de diabetes é a Lei 11.347/2006. Ela manda distribuir de graça os medicamentos e os materiais de aplicação e de monitoramento da glicemia capilar às pessoas inscritas em programas de educação para diabéticos ([FATO] [Câmara, Legin](https://www2.camara.leg.br/legin/fed/lei/2006/lei-11347-27-setembro-2006-545572-norma-pl.html) [ANTIGO, vigente]). A Portaria GM/MS 2.583/2007 fixa o elenco em três itens: **seringa com agulha acoplada, tira reagente e lanceta**. O glicosímetro **fica fora** desse artigo. O público são os **insulinodependentes** cadastrados no Cartão SUS ou no Hiperdia, e as tiras dependem de haver glicosímetro disponível ([FATO] [BVS/MS, Portaria 2.583/2007](https://bvsms.saude.gov.br/bvs/saudelegis/gm/2007/prt2583_10_10_2007.html) [ANTIGO, vigente]). O Anexo **não fixa quantidade mensal**. Recomenda em média 3 a 4 testes por dia para quem usa insulina, desaconselha o automonitoramento de rotina para quem usa só antidiabético oral e deixa a frequência à equipe ([FATO] [mesma fonte](https://bvsms.saude.gov.br/bvs/saudelegis/gm/2007/prt2583_10_10_2007.html)). O NATJUS/TJMG confirma que a quantidade depende da avaliação individual; os valores citados em pareceres, como 100 e 180 tiras/mês, são casos isolados ([FATO] [NATJUS TJMG NT 727/2018](https://bd-login.tjmg.jus.br/jspui/bitstream/tjmg/9108/1/NT%20-%20727%20-2018%20-%20fitas%20reagentes%20-%20NATJUS%20TJMG.odt.pdf) [ANTIGO]). O financiamento é tripartite pelo Componente Básico ([FATO] [Portaria 2.583, art. 4º](https://bvsms.saude.gov.br/bvs/saudelegis/gm/2007/prt2583_10_10_2007.html)). Em 2026 as contrapartidas estadual e municipal de R$ 3,01 por habitante/ano cobrem explicitamente "os insumos para os usuários insulinodependentes" ([FATO] [NI 3/2026-CGAFB/DAF](https://www.gov.br/saude/pt-br/centrais-de-conteudo/publicacoes/estudos-e-notas-informativas/2026/nota-informativa-no-3-2026-cgafb-daf-sctie-ms.pdf/@@download/file)).

Os números concretos estão nos protocolos municipais. O Protocolo 02/2026 de São Paulo (jun/2026) usa quatro critérios de entrada:

- residir no município;
- ter cadastro com vínculo ativo na UBS;
- ter relatório médico;
- pertencer a um de três perfis: DM1 com insulina, DM gestacional, ou DM2 e outros tipos com insulina.

As cotas são de **no mínimo 4 medições/dia** para insulinodependentes e **até 4 medições/dia, 3 dias por semana** para gestantes sem insulina, com **1 seringa por aplicação e por tipo de insulina**. Quem usa caneta pré-enchida não recebe seringa. Quantidade maior exige justificativa arquivada no prontuário ([FATO] [SMS-SP, Protocolo 02/2026](https://autodromodeinterlagos.prefeitura.sp.gov.br/documents/d/saude/diagramacao-pa-glicemico_v3-pdf)).

O mesmo protocolo define um ciclo de vida que o software precisa reproduzir:

- retirada a cada 30 dias, levando o glicosímetro para baixar as leituras;
- cadastro atualizado a cada 3 meses;
- **inativação automática após 6 meses sem retirada**;
- exclusão da puérpera 45 dias após o parto;
- retirada por terceiro autorizado;
- glicosímetro e leitor de CGM em **comodato**, com termo e com boletim de ocorrência em caso de perda ([FATO] [mesma fonte](https://autodromodeinterlagos.prefeitura.sp.gov.br/documents/d/saude/diagramacao-pa-glicemico_v3-pdf)).

Campinas usa outra régua ([FATO] [SMS Campinas 2022](https://portal-adm.campinas.sp.gov.br/sites/default/files/secretarias/arquivos-avulsos/125/2022/10/06-152750/Orientacoes_para_rede_municipal_fornecimento_glicosimetro_insumos_pacientes_diabeticos.pdf) [ANTIGO]):

- **até 50 tiras/mês** sem aprovação; acima disso, o Distrito aprova;
- lancetas na mesma quantidade das tiras;
- agulha de caneta a 1 por dia por tipo de insulina;
- cadastro com quantitativo de 1 ano e receita renovada a cada 12 meses;
- retirada em qualquer centro de saúde depois do cadastro.

As duas cidades divergem até na unidade da cota: São Paulo conta medições por dia e Campinas conta tiras por mês ([INFERÊNCIA]).

### Estomia, fraldas e preservativos seguem lógicas diferentes

Para estomia, a Portaria SAS/MS 400/2009 está consolidada na PRC SAES/MS nº 1/2022 e estava em revisão em 2022 ([FATO] [Câmara, audiência 29/11/2022](https://www2.camara.leg.br/atividade-legislativa/comissoes/comissoes-permanentes/cssf/29-11-2022%20-%20Pessoas%20Ostomizadas%20Importam%20-%20Raelma%20Paz%20Silva.pdf) [ANTIGO]). As bolsas e os adjuvantes são **procedimentos OPM do SIGTAP faturados em BPA-I**, como 0701050012, 0701050020, 0701050047, 0701060026, 0701060034 e 0701060042. O registro exige CNS, CID, CBO e serviço/classificação no CNES ([FATO] [NT 3/2025-CGSPD/DAET/SAES/MS](https://www.gov.br/saude/pt-br/assuntos/saude-de-a-a-z/s/saude-da-pessoa-com-deficiencia/notas-tecnicas/nota-tecnica-no-3-2025-cgspd-daet-saes-ms)). O teto de "no máximo 30 por mês" para o 0701050020 vem de uma apresentação e não foi conferido no SIGTAP ([FATO] [Câmara CPD 2025](https://www2.camara.leg.br/atividade-legislativa/comissoes/comissoes-permanentes/cpd/apresentacoes-em-eventos/eventos-2025/ApresentaoGILBERTOKOEHLER.pdf)).

Em Piracicaba quem **fornece** as bolsas é a SES-SP. O município cadastra, exige laudo com CID e entrega mensalmente. No cateterismo intermitente, o protocolo fixa **lidocaína gel em 4 frascos/mês para mulheres e 6 para homens**, luvas na mesma quantidade dos cateteres e seringa de 20 ml limitada a 30/mês ([FATO] [Piracicaba, Protocolo 2024](https://piracicaba.sp.gov.br/wp-content/uploads/2024/08/12-%E2%80%93-Protocolo-Ostomia-e-Incontinencia-Urinaria-2024.pdf)). Algumas cotas dependem do sexo do paciente, e o estoque pode pertencer a outro ente ([INFERÊNCIA]).

A fralda geriátrica é sobretudo do Farmácia Popular, um canal federal fora do estoque municipal, 100% gratuito desde 14/02/2025 ([FATO] [gov.br Farmácia Popular](https://www.gov.br/saude/pt-br/assuntos/farmaciapopular); [FATO] [TJRJ parecer 1072/2025](https://portaltj.tjrj.jus.br/documents/d/guest/parecer-1072-2025)). Há municípios com programa próprio de fraldas, como Criciúma ([FATO] [CIGTES Criciúma](http://cigtes.criciuma.sc.gov.br/redes-de-atencao-saude/material-apoio/29), só trecho). Os preservativos descem em cascata, do MS para a SES e daí para o município ([FATO] [Agência Gov, ago/2025](https://agenciagov.ebc.com.br/noticias/202508/para-aumentar-uso-de-preservativos-no-pais-ministerio-da-saude-diversifica-oferta-de-camisinhas-no-sus)). Não se achou cota nem prescrição nacional para eles. O controle municipal desses itens é logístico, sem dispensação nominal ([INFERÊNCIA]).

## O CATMAT de material fica fora da carga atual do catálogo

Medicamentos e materiais estão no mesmo CATMAT, mas em classes e padrões descritivos diferentes:

- a classe **6505** reúne os medicamentos, descritos por princípio ativo e apresentação;
- a classe **6515** reúne instrumentos e suprimentos médico-cirúrgicos e a **6550** os produtos de diagnóstico in vitro. Ambas são descritas por um **PDM**, o padrão descritivo de material com características técnicas ([FATO] [MS/DESID, CATMAT](https://www.gov.br/saude/pt-br/se/desid/catmat); [FATO] [Compras.gov.br, Catalogação de PPSME](https://www.gov.br/compras/pt-br/acesso-a-informacao/cursos-e-capacitacoes/materiais-de-apoio/catalogo-catmat-catser/CatalogaodePPSME.pdf)).

A ANVISA também separa os dois grupos: **13 dígitos para medicamento e 11 para dispositivo médico** ([FATO] [MS, BPS](https://www.gov.br/saude/pt-br/acesso-a-informacao/banco-de-precos/publicacoes/como-cadastrar-uma-compra)). Na BNAFAR, **o medicamento é identificado pela OBM** (Portaria GM/MS 6.093/2024) **e o insumo pelo CATMAT** ([FATO] [Portaria 5.713/2024, art. 391-B e 392](https://bvsms.saude.gov.br/bvs/saudelegis/gm/2024/prt5713_17_12_2024.html); [FATO] [Conass Informa 214/2024](https://www.conass.org.br/conass-informa-n-214-2024-publicada-a-portaria-gm-n-6-093-que-institui-a-ontologia-brasileira-de-medicamentos/)). Os municípios mantêm códigos paralelos, como o SUPRI de São Paulo ([FATO] [SMS-SP](https://autodromodeinterlagos.prefeitura.sp.gov.br/documents/d/saude/diagramacao-pa-glicemico_v3-pdf)).

O catálogo atual não acomoda esses itens. `medication_catalog_items` é uma tabela da plataforma, com um item por CATMAT e colunas de medicamento: `active_ingredient`, `strength`, `dosage_form` e `unit`. A coluna `obm_code` está reservada por um CHECK `obm_code IS NULL` ([FATO] `apps/api/db/platform_schema.rb:359-392`, `387`). A carga vem de uma única fonte, `catmat_6505` ([FATO] `apps/api/db/platform_schema.rb:404`). A unidade de dispensação é derivada de `dosage_form` por um YAML ([FATO] `apps/api/app/services/pharmacy/dispensing_unit.rb:1-22`), e um insumo não tem forma farmacêutica. O gate api#105, que mede itens CATMAT sem forma farmacêutica, já toca esse ponto ([FATO] `.worktrees/docs-coord/modulos/27--farmacia-e-almoxarifado.md:60`). Os insumos precisam, portanto, de outra fonte de carga (classes 6515/6550), de atributos livres vindos do PDM, como calibre, volume e tamanho, e de unidade de fornecimento própria ([INFERÊNCIA]).

## O 27a já tem o razão; falta a camada de programa

O estoque do 27a é um razão único em `stock_lots` + `stock_movements`, só de acréscimo, em que o saldo é a soma dos movimentos. Toda escrita passa por `Pharmacy::Ledger` ([FATO] `apps/api/app/services/pharmacy/ledger.rb:1-19`). O CHECK `ck_stock_lots_item` aceita três chaves de item mutuamente exclusivas: `medication` (catálogo da plataforma), `immunobiological` (YAML) e `vector_control` (tabela da cidade) ([FATO] `apps/api/db/city_schema.rb:4293`; `apps/api/app/models/stock_lot.rb:1-21`). O 35a acrescentou a terceira chave numa migração de cidade **irreversível**, que mexeu em locais, lotes, mínimos, aberturas e movimentos e trocou os CHECKs ([FATO] `apps/api/db/city_migrate/20261116900002_add_vector_inspections.rb:19-26, 194-246`). Esse é o molde para uma quarta chave de insumo ([INFERÊNCIA]).

Há uma diferença em relação aos precedentes. A vacina e o controle vetorial ganharam **domínio e local próprios**, e `Stock::Domain.allows?` recusa um lote vindo de outro domínio ([FATO] `apps/api/app/services/stock/domain.rb:1-53`). O usuário decidiu que os insumos moram "sobre o mesmo estoque" dos medicamentos, na opção A ([FATO] `.worktrees/docs-coord/superpowers/specs/2026-10-29-module-27a-pharmacy-stock-design.md:29`). O insumo precisa, então, ser uma chave nova **dentro** do domínio de medicamentos, o que contraria a suposição atual de que um lote que não é imunobiológico nem vetorial é medicamento ([INFERÊNCIA]).

A dispensação também não serve como está. `dispensations` exige receita (`num_nonnulls(clinical_document_id, external_prescription_id) = 1`), e `dispensation_items.catalog_item_id` é NOT NULL e aponta para o catálogo de medicamentos ([FATO] `apps/api/db/city_schema.rb:1254-1289`). O insumo de diabetes sai contra uma **autorização de programa** (relatório médico com validade de 3 a 12 meses e cota periódica), não contra uma receita item a item. O preservativo sai sem paciente ([INFERÊNCIA], a partir de [SMS-SP](https://autodromodeinterlagos.prefeitura.sp.gov.br/documents/d/saude/diagramacao-pa-glicemico_v3-pdf) e [Campinas](https://portal-adm.campinas.sp.gov.br/sites/default/files/secretarias/arquivos-avulsos/125/2022/10/06-152750/Orientacoes_para_rede_municipal_fornecimento_glicosimetro_insumos_pacientes_diabeticos.pdf)).

Os editais pedem um almoxarifado de padrão ERP. Os itens citados são:

- sub-almoxarifados;
- entrada automática a partir da liquidação;
- transferência com nota;
- requisição com atendimento parcial;
- estoque mínimo e máximo;
- localização física;
- custo médio, curva ABC, balancete e inventário ([FATO] [CISTRI-MG PE 03/2022](https://cistri.saude.mg.gov.br/wp-content/uploads/2024/07/03-PE-EDITAL-SISTEMAS.pdf) [ANTIGO]);
- "solicitação, controle e dispensação de insumos" ([FATO] [SES-RJ 2022](https://www.saude.rj.gov.br/comum/code/MostrarArquivo.php?C=NDk4Njk%2C) [ANTIGO], só trecho);
- sugestão do lote com vencimento mais próximo ([FATO] [Alvinópolis PE 03/2022](https://www.alvinopolis.mg.gov.br/abrir_arquivo.aspx/Pregao_Eletronico_3_2022_Pregao_Eletronico_003_2022_Sistema_Integrado_de_Gestao_em_Saude?cdLocal=3&arquivo=%7B847A0E4D-5B81-5EC4-ECAC-4EA7A08AC0CA%7D.pdf&cdLicitacaoArquivo=10129) [ANTIGO], só trecho).

**Nenhum edital lido exigia cota por paciente nem comodato.** Esses requisitos aparecem nos protocolos e POPs ([INFERÊNCIA]; a cobertura é fraca, porque vários editais só foram vistos em trecho). Já os editais de **compra** de tiras exigem glicosímetro cedido pelo fornecedor: Niterói em 2021 e Itarana em 2025, com 250 aparelhos ([FATO] [Niterói PE 37/2021](http://www.niteroi.rj.gov.br/wp-content/uploads/licitacao/fms/2021/PREG%C3%83O%20ELETR%C3%94NICO%2037-2021%20-%20SRP%20AQUISI%C3%87%C3%83O%20DE%20TIRAS%20REAGENTES%20DE%20MEDIDA%20DE%20GLICEMIA%20PARA%20DIABETES.pdf) [ANTIGO]; [FATO] [Itarana 2025](https://www.itarana.es.gov.br/licitacoes/58e90c868f473a37f6a3870ff4793e4b.pdf), só trecho). Quando o pregão troca a marca da fita, o glicosímetro precisa ser trocado junto ([INFERÊNCIA]).

O que o 27a ainda não tem e diferenciaria o produto: o lado ERP básico do almoxarifado (custo médio, curva ABC, requisição parcial) e, sobretudo, a camada de saúde, com programa, cota, comodato, exceção aprovada e exportação BPA-I ([INFERÊNCIA]). O `BillingExport` do módulo 22 já gera BPA só de acréscimo e cifrado e pode receber as bolsas de estomia ([FATO] `apps/api/app/models/billing_export.rb:1-4`).

## O BNAFAR virou obrigação geral, diária e por lote

A BNAFAR nasceu com a Portaria 271/2013, que na prática só alcançava os municípios do Qualifar-SUS ([FATO] [CONASS](https://www.conass.org.br/?p=19232) [ANTIGO]; [FATO] [COSEMS-SC](https://www.cosemssc.org.br/bnafar-envio-de-informacoes-para-a-base-nacional-da-assistencia-farmaceutica/) [ANTIGO, superado]). A Portaria GM/MS 5.713, de 09/12/2024, generalizou a obrigação ([FATO] [BVS SaúdeLegis](https://bvsms.saude.gov.br/bvs/saudelegis/gm/2024/prt5713_17_12_2024.html)):

- abrange "medicamentos, vacinas, insumos e equipamentos" (art. 391);
- cria o REPE e o RESMPE (art. 391-A);
- exige envio **diário, sem reenvio** (art. 391-D);
- alcança RENAME, REMUME, RESME e também itens de fora das listas, como os de ordem judicial (§2º);
- **obriga quem tem sistema próprio a adequá-lo** (§5º);
- permite **retificação ou exclusão até o último dia do mês seguinte** (§7º);
- deixa a SES ou um consórcio enviar pelo município, se houver pactuação na CIB (§§8–9).

O plano operativo da CIT publicou os serviços em 15/02/2025 e deu **120 dias, até 17/06/2025**, para a implementação ([FATO] [Informe CIT 24/04/2025](https://conasems-ava-prod.s3.sa-east-1.amazonaws.com/institucional/noticias/2-7-apresentacao-informe-cit-bnafar-24-04-2025-1745608921.pdf)). Envio retroativo só por exceção justificada ([FATO] [FAQ SI-BNAFAR](https://webatendimento.saude.gov.br/faq/sibnafar)).

A sanção explícita está na Portaria GM/MS 11.585/2026: **suspensão dos incentivos da assistência farmacêutica vinculados ao envio à BNAFAR e à RNDS** (art. 391-I) ([FATO] [Conass Informa 114/2026](https://www.conass.org.br/conass-informa-n-114-2026-publicada-a-portaria-gm-n-11-585-que-altera-a-portaria-de-consolidacao-gm-ms-no-1-de-28-de-setembro-de-2017-para-dispor-sobre-a-substituicao-do-sistema-hor/)). No Qualifar, o envio é condição do incentivo ([FATO] [FAQ Qualifar-SUS](https://www.gov.br/saude/pt-br/composicao/sectics/qualifar-sus/faq)). A nota de repasse do CBAF de 2026 **não cita a BNAFAR como condição** do per capita ([FATO] [NI 3/2026](https://www.gov.br/saude/pt-br/centrais-de-conteudo/publicacoes/estudos-e-notas-informativas/2026/nota-informativa-no-3-2026-cgafb-daf-sctie-ms.pdf/@@download/file)). O risco financeiro recai, portanto, sobre os incentivos, e não sobre o repasse básico ([INFERÊNCIA]). A adesão em 23/04/2025 mostra que a obrigação pegou só em parte: **1.254 municípios enviavam pelo SI-BNAFAR, 2.587 pelo Hórus e 1.729 não tinham pedido acesso** ([FATO] [Informe CIT](https://conasems-ava-prod.s3.sa-east-1.amazonaws.com/institucional/noticias/2-7-apresentacao-informe-cit-bnafar-24-04-2025-1745608921.pdf)).

### Insumos entram no envio, e a dispensação sai dele

O escopo dos insumos tem dois níveis. A lista **obrigatória** de insumos e equipamentos com CATMAT é a das compras centralizadas do MS (art. 392). Ao mesmo tempo, o §2º do art. 391-D alcança tudo o que está na RENAME e na REMUME ([FATO] [Portaria 5.713](https://bvsms.saude.gov.br/bvs/saudelegis/gm/2024/prt5713_17_12_2024.html)). Tiras, lancetas e seringas de insulinodependentes são insumos da RENAME financiados pelo CBAF ([FATO] [NI 3/2026](https://www.gov.br/saude/pt-br/centrais-de-conteudo/publicacoes/estudos-e-notas-informativas/2026/nota-informativa-no-3-2026-cgafb-daf-sctie-ms.pdf/@@download/file)). Devem, portanto, entrar no REPE e no RESMPE com CATMAT ([INFERÊNCIA]). A posição de estoque alcança "qualquer local com medicamento ou insumo em estoque", **almoxarifados incluídos**. E a regra sobre a entrega ao paciente é direta: *"a dispensação não deve ser informada como uma Saída"* ([FATO] [COSEMS-SP, slide 9](https://www.cosemssp.org.br/wp-content/uploads/2025/05/WEB-BNAFAR-APRESENTACAO-MARCOS.pdf)). As **entradas** e a avaliação do CEAF foram descontinuadas ([FATO] [mesma fonte](https://www.cosemssp.org.br/wp-content/uploads/2025/05/WEB-BNAFAR-APRESENTACAO-MARCOS.pdf)). Mesmo assim, o sistema precisa guardar na entrada os dados que o REPE exige depois ([INFERÊNCIA]).

## O canal tem duas pontas: SI-BNAFAR para estoque, RNDS para dispensação

### SI-BNAFAR: API REST com homologação própria

O SI-BNAFAR substituiu o SOA BNAFAR. Trouxe novo modelo de dados, envio D-1 e a OBM como terminologia oficial ([FATO] [FAQ SI-BNAFAR](https://webatendimento.saude.gov.br/faq/sibnafar)). O **SOAP WSBNDAF foi desligado em 28/02/2025** ([FATO] [COSEMS-SP, slides 10-11](https://www.cosemssp.org.br/wp-content/uploads/2025/05/WEB-BNAFAR-APRESENTACAO-MARCOS.pdf)).

Os recursos para integração ([FATO] [COSEMS-SP, slides 5, 12-13](https://www.cosemssp.org.br/wp-content/uploads/2025/05/WEB-BNAFAR-APRESENTACAO-MARCOS.pdf)):

- página do serviço no Portal de Serviços do DATASUS (`servicos-datasus.saude.gov.br/detalhe/DxRPsAn2mh`), com manual de integração, catálogo CATMAT, massa de dados de homologação e coleção Postman;
- Swagger de homologação em `bnafar-api-hom.saude.gov.br/swagger-ui.html`;
- manual vigente **v1.20 (04/2025)**.

A credencial é pedida nesse portal, e quem tinha credencial do SOA a mantém. **O FAQ não fala em certificado digital nem em SCPA** ([FATO] [FAQ SI-BNAFAR](https://webatendimento.saude.gov.br/faq/sibnafar)). O repositório oficial do DAF descreve a API com OAuth 2, quantidades sempre na **menor unidade de fornecimento** e identificadores CNES, CNPJ, CNS, CPF e CRM ([FATO] [GitHub CGMPAF/ServicosBnafar](https://github.com/CGMPAF/ServicosBnafar) [ANTIGO, ~2020]). Os entes passam de homologação para produção um a um: em abril de 2025, SP, PR, SC, MS, AM e BA estavam em produção ([FATO] [Informe CIT](https://conasems-ava-prod.s3.sa-east-1.amazonaws.com/institucional/noticias/2-7-apresentacao-informe-cit-bnafar-24-04-2025-1745608921.pdf)).

### Campos exigidos contra o que o 27a já tem

A tabela cruza os campos mínimos dos três registros com o schema atual. Os campos do REPE e do RESMPE vêm dos arts. 391-B e 391-C da [Portaria 5.713/2024](https://bvsms.saude.gov.br/bvs/saudelegis/gm/2024/prt5713_17_12_2024.html) e os do REDFM do art. 3º da [Portaria 6.100/2024](https://bvsms.saude.gov.br/bvs/saudelegis/gm/2024/prt6100_18_12_2024.html) ([FATO]).

| Campo exigido | Registro | Onde está no 27a | Situação |
|---|---|---|---|
| CNES do estabelecimento | REPE, RESMPE, REDFM | `stock_locations.health_unit_id` → `health_units.cnes`, que é anulável; o `central` é proibido de ter unidade ([FATO] `apps/api/db/city_schema.rb:4256`, `1672-1691`) | **Lacuna no almoxarifado central e na rede de frio central** |
| OBM do medicamento | REPE, RESMPE, REDFM | `obm_code` reservado com CHECK IS NULL ([FATO] `apps/api/db/platform_schema.rb:387`) | **Bloqueado** |
| CATMAT do insumo | REPE, RESMPE | Só existe CATMAT 6505 ([FATO] `apps/api/db/platform_schema.rb:404`) | Precisa do catálogo de materiais |
| CNPJ do fabricante | REPE, RESMPE | `stock_lots.manufacturer`: texto livre opcional; `manufacturer_key` só no imunobiológico ([FATO] `apps/api/db/city_schema.rb:4286-4287, 4293`) | **Lacuna** |
| Registro ANVISA | REPE, RESMPE | Ausente ([FATO] `apps/api/db/city_schema.rb:4276-4293`) | **Lacuna** |
| Lote e validade | REPE, RESMPE, REDFM | `lot_number`, `expires_on` NOT NULL ([FATO] mesmo trecho) | Pronto |
| Valor unitário | REPE, RESMPE | Ausente em lote, entrada e movimento ([FATO] `apps/api/db/city_schema.rb:4219-4228, 4276-4330`) | **Lacuna** |
| Programa de saúde | REPE, RESMPE | Ausente ([FATO] mesmo trecho) | **Lacuna** |
| IUM (SNCM) | REPE, RESMPE | Ausente ([FATO] mesmo trecho) | Lacuna, aplicável só onde houver SNCM |
| Quantidade | todos | Saldo como soma; `dispensing_unit` local ([FATO] `apps/api/app/services/pharmacy/ledger.rb:1-19`) | Conferir se é a menor unidade de fornecimento |
| Motivo e CNES de destino | RESMPE | Tipos `transfer_out`, `loss`, `expiry`; `loss_code` da lista do SIES ([FATO] `apps/api/db/city_schema.rb:4344`; `apps/api/app/commands/pharmacy/request_adjustment.rb:48-50`) | Falta mapear para a lista federal, que não foi lida |
| Identificador de origem | REPE, RESMPE, REDFM | uuid do movimento, que nunca é apagado ([FATO] `apps/api/db/city_schema.rb:4312`) | Pronto |
| CPF ou CNS do paciente | REDFM | `patients.cns` / `cpf`; o paciente não identificado não tem nenhum dos dois ([FATO] `apps/api/db/city_schema.rb:3120-3143`) | Pronto, com bloqueio para o não identificado |
| Dispensador com CBO | REDFM | `dispensed_by_user_id` → `professional`; o perfil completo da equipe é o gate api#104 ([FATO] `apps/api/db/city_schema.rb:1276-1277`; `.worktrees/docs-coord/modulos/27--farmacia-e-almoxarifado.md:60`) | Depende do api#104 |
| CPF ou CNS de quem retira | REDFM | Ausente em `dispensations` ([FATO] `apps/api/db/city_schema.rb:1272-1289`) | **Lacuna** |
| Estorno; justificativa de atendimento parcial | REDFM | `completes_item` existe; não há estorno nem justificativa ([FATO] `apps/api/db/city_schema.rb:1254-1270`) | **Lacuna** |
| ID da prescrição na RNDS | REDFM | Receita digital do 19c; a externa não tem CNS/CPF do prescritor ([FATO] `apps/api/db/city_schema.rb:1543-1558`) | Parcial |
| Assinatura eletrônica | REDFM | Ausente na dispensação | **Lacuna** |

### REDFM: dispensação vai à RNDS, mas o modelo ainda não estava publicado

A dispensação **não tem método no SI-BNAFAR** e vai exclusivamente à RNDS pelo REDFM da Portaria 6.100/2024 ([FATO] [FAQ SI-BNAFAR](https://webatendimento.saude.gov.br/faq/sibnafar); [FATO] [Portaria 6.100/2024](https://bvsms.saude.gov.br/bvs/saudelegis/gm/2024/prt6100_18_12_2024.html)). O envio é "regular", sem periodicidade fixa. Os prazos dependem de um plano tripartite (arts. 4º e 6º) ([FATO] [mesma fonte](https://bvsms.saude.gov.br/bvs/saudelegis/gm/2024/prt6100_18_12_2024.html)). Em 24/04/2025 a publicação dos modelos computacionais ainda estava "aguardando DATASUS" ([FATO] [Informe CIT](https://conasems-ava-prod.s3.sa-east-1.amazonaws.com/institucional/noticias/2-7-apresentacao-informe-cit-bnafar-24-04-2025-1745608921.pdf)). Existe um perfil FHIR `RNDSDispensacaoMedicamento` v2.0.0, mas o próprio Simplifier avisa que não é o repositório oficial ([FATO] [Simplifier RNDS](https://simplifier.net/redenacionaldedadosemsaude/rndsdispensacaomedicamento/$downloadsnapshot?format=xml)).

O REDFM fala só de medicamento em OBM, então a dispensação nominal de fita, lanceta e seringa fica fora dele ([INFERÊNCIA]). O Rota Saúde já tem o encanamento da RNDS:

- `Rnds::Client` com mTLS e o PKCS#12 A1 da cidade;
- `Rnds::DeliverJob` recorrente por cidade;
- fila com `FOR UPDATE SKIP LOCKED`, payload cifrado anulado no aceite e backoff ([FATO] `apps/api/app/services/rnds/client.rb:1-8`; `apps/api/app/jobs/rnds/deliver_job.rb:1-7`; `apps/api/app/models/rnds_outbox_entry.rb:1-143`).

A tabela `rnds_outbox`, porém, está presa a `immunization_id` NOT NULL e aos motivos de bloqueio da vacina ([FATO] `apps/api/db/city_schema.rb:3790-3815`). O REDFM pediria uma fila irmã que reaproveite o padrão e a credencial `rnds` ([INFERÊNCIA]).

### Hórus em extinção e e-SUS AF voluntário

A Portaria GM/MS 11.585, de 16/06/2026, substitui o Hórus pelo eSUSAF ([FATO] [Conass Informa 114/2026](https://www.conass.org.br/conass-informa-n-114-2026-publicada-a-portaria-gm-n-11-585-que-altera-a-portaria-de-consolidacao-gm-ms-no-1-de-28-de-setembro-de-2017-para-dispor-sobre-a-substituicao-do-sistema-hor/)):

- **180 dias** para encerrar os registros de entrada, estoque, saída e dispensação no Hórus;
- depois disso, 5 anos só de consulta;
- base histórica entregue aos entes por **API ou arquivos tabulados**;
- o Hórus Especializado espera o piloto em Sergipe, e a APAC continua no Hórus;
- os entes assumem a infraestrutura do eSUSAF;
- o recurso não pode pagar sistema privado (art. 391-H §6º).

A minuta apresentada ao CONASS em maio previa a janela de junho a novembro de 2026. Em 27/05/2026, 1.655 dos 2.221 municípios aderidos (79,38%) já tinham migrado ([FATO] [CONASS, apresentação 05/2026](https://www.conass.org.br/wp-content/uploads/2026/05/2.4.-Portaria-que-dispoe-sobre-as-estrategias-eSUS-AF.pdf)). O MS afirma que **a adesão ao eSUSAF não é obrigatória**, que "o envio diário de dados às bases nacionais continua obrigatório" e que **os dados do Hórus não serão migrados automaticamente** ([FATO] [gov.br/saude, abr/2026](https://www.gov.br/saude/pt-br/assuntos/noticias/2026/abril/ministerio-da-saude-apresenta-novo-sistema-de-assistencia-farmaceutica-no-sus)). A decisão do ADR 0045 de construir em vez de integrar com o e-SUS AF continua válida ([FATO] `.worktrees/docs-coord/adr/0045.md:12-14, 52`). A pergunta "o e-SUS AF aceita gravação de terceiros?" ([FATO] `.worktrees/docs-coord/pesquisa/2026-10-05-mapa-de-integracoes.md:171`) perdeu importância, porque o próprio eSUSAF envia pelo mesmo SI-BNAFAR e pela mesma RNDS que um sistema próprio usaria ([INFERÊNCIA]).

## Conclusão

A pesquisa muda o eixo do 27b. O núcleo deixa de ser "cada dispensação vira uma mensagem" e passa a ser "cada estabelecimento com estoque publica, todo dia, uma fotografia por lote, e cada transferência ou perda vira um evento com motivo". O razão do 27a é bom ponto de partida porque nunca apaga movimentos e calcula o saldo por soma, o que dá identificadores estáveis e fotografias D-1 reproduzíveis. As lacunas estão no **cadastro**, não no razão: o dado de lote é magro, o central não tem CNES e a OBM não está carregada. Isso desloca trabalho para o 27a já entregue, na entrada (`RecordEntry`), no catálogo da plataforma e em uma migração de cidade, antes que qualquer job de envio faça sentido ([INFERÊNCIA]).

Nos insumos, o valor está na camada de programa, não no catálogo. Codificar o CATMAT 6515 é trabalho de carga. Quem compra o sistema, porém, convive com cota por perfil, renovação de laudo, inativação por ausência, retirada por terceiro e glicosímetro em comodato preso à marca do pregão, e os editais nem pedem isso explicitamente. Construir a autorização de programa como entidade de primeira classe atende os protocolos de São Paulo, Campinas e Piracicaba com a mesma estrutura parametrizada. Ela também serve de documento de origem para a saída de estoque que o REPE e o RESMPE vão contar ([INFERÊNCIA]).

## Achados que mudam o desenho

### Insumos

1. **O insumo é uma quarta chave dentro do domínio de medicamentos, não um domínio novo.** O molde do 35a (troca de 4 CHECKs, novo `item_kind`, migração irreversível) se aplica. Mas `Stock::Domain.of_lot` e `allows?` hoje tratam como medicamento todo lote que não é imunobiológico nem vetorial e separam domínios por local. Os insumos dividem `central` e `unit` com os medicamentos, então esse ramo precisa ser reescrito ([FATO] `apps/api/app/services/stock/domain.rb:1-53`; `.worktrees/docs-coord/superpowers/specs/2026-10-29-module-27a-pharmacy-stock-design.md:29`; [INFERÊNCIA] na consequência).
2. **O catálogo de materiais precisa de outra carga e de outro formato de item.** As classes CATMAT são 6515/6550, com PDM em vez de princípio ativo, concentração e forma. O registro ANVISA tem 11 dígitos e a unidade de fornecimento é própria. `catmat_6505` e `DispensingUnit.for(dosage_form)` não servem ([FATO] [MS/DESID](https://www.gov.br/saude/pt-br/se/desid/catmat); `apps/api/db/platform_schema.rb:404`; `apps/api/app/services/pharmacy/dispensing_unit.rb:1-22`). Como o CATMAT é nacional, o lugar natural é uma tabela da plataforma, com código interno municipal opcional na cidade ([INFERÊNCIA]).
3. **A dispensação de insumo não nasce de receita, e sim de uma autorização de programa.** A autorização reúne elegibilidade por perfil e CID, laudo, validade de 3 a 12 meses, cota por período derivada de parâmetros, exceção com justificativa e aprovador, inativação por ausência e retirada por terceiro. Hoje `dispensations` exige receita e `dispensation_items.catalog_item_id` é NOT NULL ([FATO] `apps/api/db/city_schema.rb:1254-1289`; [FATO] [SMS-SP](https://autodromodeinterlagos.prefeitura.sp.gov.br/documents/d/saude/diagramacao-pa-glicemico_v3-pdf); [INFERÊNCIA] no desenho).
4. **As cotas são parâmetros por cidade e não constantes.** Mudam a unidade (medições/dia em SP, tiras/mês em Campinas), o perfil, o sexo (lidocaína em Piracicaba) e a frequência ([FATO] [SMS-SP](https://autodromodeinterlagos.prefeitura.sp.gov.br/documents/d/saude/diagramacao-pa-glicemico_v3-pdf); [FATO] [Campinas](https://portal-adm.campinas.sp.gov.br/sites/default/files/secretarias/arquivos-avulsos/125/2022/10/06-152750/Orientacoes_para_rede_municipal_fornecimento_glicosimetro_insumos_pacientes_diabeticos.pdf) [ANTIGO]; [FATO] [Piracicaba](https://piracicaba.sp.gov.br/wp-content/uploads/2024/08/12-%E2%80%93-Protocolo-Ostomia-e-Incontinencia-Urinaria-2024.pdf)).
5. **O comodato é patrimônio por número de série, não consumo de estoque.** Glicosímetro e leitor de CGM ficam com o paciente, com termo, devolução, troca contra o aparelho defeituoso e boletim de ocorrência em caso de perda. O aparelho fica vinculado ao fornecedor e à tira compatível, e a troca de pregão força a troca do aparelho ([FATO] [SMS-SP](https://autodromodeinterlagos.prefeitura.sp.gov.br/documents/d/saude/diagramacao-pa-glicemico_v3-pdf); [FATO] [Niterói PE 37/2021](http://www.niteroi.rj.gov.br/wp-content/uploads/licitacao/fms/2021/PREG%C3%83O%20ELETR%C3%94NICO%2037-2021%20-%20SRP%20AQUISI%C3%87%C3%83O%20DE%20TIRAS%20REAGENTES%20DE%20MEDIDA%20DE%20GLICEMIA%20PARA%20DIABETES.pdf) [ANTIGO]; [INFERÊNCIA] no desenho).
6. **A bolsa de estomia precisa de código SIGTAP no item e de CID na autorização, para sair em BPA-I pelo `BillingExport` do 22.** O estoque pode pertencer à SES, como em Piracicaba ([FATO] [NT 3/2025-CGSPD](https://www.gov.br/saude/pt-br/assuntos/saude-de-a-a-z/s/saude-da-pessoa-com-deficiencia/notas-tecnicas/nota-tecnica-no-3-2025-cgspd-daet-saes-ms); `apps/api/app/models/billing_export.rb:1-4`; [FATO] [Piracicaba](https://piracicaba.sp.gov.br/wp-content/uploads/2024/08/12-%E2%80%93-Protocolo-Ostomia-e-Incontinencia-Urinaria-2024.pdf)).
7. **Fralda do Farmácia Popular fica fora do estoque municipal, e preservativo sai sem paciente.** O primeiro caso dispensa catálogo; o segundo pede uma saída logística por unidade sem dispensação nominal ([FATO] [gov.br Farmácia Popular](https://www.gov.br/saude/pt-br/assuntos/farmaciapopular); [INFERÊNCIA] no desenho).
8. **O catálogo público hoje tem um teste com regex que proíbe prometer "insumo|material".** Entregar insumos exige mudar esse teste no repo do catálogo ([FATO] `.worktrees/docs-coord/superpowers/plans/2026-10-29-module-27a-pharmacy-stock-maintenance.md:44, 382`).

### 27b

1. **O núcleo do 27b é uma fotografia diária de estoque (REPE) e um evento de saída (RESMPE), não uma fila de dispensações.** A dispensação está proibida como saída, e as entradas foram descontinuadas. Para o REPE serve melhor o molde de remessa periódica (`PsychosocialRaasExport`/`BillingExport`). O RESMPE pode usar fila por evento ou lote D-1 ([FATO] [COSEMS-SP, slide 9](https://www.cosemssp.org.br/wp-content/uploads/2025/05/WEB-BNAFAR-APRESENTACAO-MARCOS.pdf); `apps/api/app/models/psychosocial_raas_export.rb:1-6`; [INFERÊNCIA] na escolha do molde).
2. **O lote precisa de valor unitário, CNPJ do fabricante, registro ANVISA, programa de saúde e IUM.** A captura entra no `RecordEntry`, por migração de cidade. Os lotes já abertos ficam sem esses dados e exigem uma regra de complemento ([FATO] [Portaria 5.713, art. 391-B](https://bvsms.saude.gov.br/bvs/saudelegis/gm/2024/prt5713_17_12_2024.html); `apps/api/db/city_schema.rb:4276-4293`; [INFERÊNCIA] na consequência).
3. **O almoxarifado central e a rede de frio central precisam de CNES.** `ck_stock_locations_unit` proíbe `health_unit_id` nesses locais, e o REPE abrange almoxarifados. `health_units.cnes` também é anulável ([FATO] `apps/api/db/city_schema.rb:4256`, `1672-1691`; [FATO] [COSEMS-SP, slide 9](https://www.cosemssp.org.br/wp-content/uploads/2025/05/WEB-BNAFAR-APRESENTACAO-MARCOS.pdf)).
4. **A OBM deixa de ser reservada.** É preciso liberar o CHECK `ck_medication_catalog_items_obm_reserved` e carregar o mapeamento CATMAT → OBM do `portal-obm.saude.gov.br`, que tem API, no catálogo da plataforma ([FATO] `apps/api/db/platform_schema.rb:387`; [FATO] [FAQ SI-BNAFAR](https://webatendimento.saude.gov.br/faq/sibnafar)).
5. **Retificação até o fim do mês seguinte num razão só de acréscimo.** O uuid do movimento serve como identificador de origem, mas um ajuste posterior precisa ser classificado como correção de registro já enviado ou como novo evento. Isso exige registrar o que foi enviado e quando ([FATO] [Portaria 5.713, art. 391-D §7º](https://bvsms.saude.gov.br/bvs/saudelegis/gm/2024/prt5713_17_12_2024.html); [INFERÊNCIA] no desenho).
6. **O CHECK do banco limita `IntegrationCredential.KINDS` a `ledi cadsus rnds`.** A credencial nova `bnafar` exige migração de cidade e segredo próprio. Pelo repositório antigo a autenticação é OAuth 2, enquanto a RNDS usa mTLS com A1 ([FATO] `apps/api/app/models/integration_credential.rb:8`; `apps/api/db/city_schema.rb:1947`; [FATO] [GitHub CGMPAF](https://github.com/CGMPAF/ServicosBnafar) [ANTIGO]).
7. **O REDFM é uma integração separada, que fica à espera da publicação do modelo e depende do api#104.** Ele reaproveita o `Rnds::Client` e a credencial `rnds`, mas pede uma fila irmã da `rnds_outbox`. Também pede campos que a dispensação não tem: quem retira, estorno, justificativa de atendimento parcial e assinatura eletrônica ([FATO] [Portaria 6.100/2024](https://bvsms.saude.gov.br/bvs/saudelegis/gm/2024/prt6100_18_12_2024.html); `apps/api/db/city_schema.rb:1254-1289, 3790-3815`).
8. **Insumos entram no REPE e no RESMPE com CATMAT, mas ficam fora do REDFM.** O catálogo de materiais é, portanto, pré-requisito do 27b completo ([FATO] [Portaria 5.713, art. 391-B](https://bvsms.saude.gov.br/bvs/saudelegis/gm/2024/prt5713_17_12_2024.html); [INFERÊNCIA] na consequência).
9. **A quantidade vai na menor unidade de fornecimento.** É preciso conferir se a `dispensing_unit` local equivale a ela ou se a conversão de embalagem cobre a diferença ([FATO] [GitHub CGMPAF](https://github.com/CGMPAF/ServicosBnafar) [ANTIGO]; `apps/api/db/city_schema.rb:4280`).
10. **A janela do Hórus fecha por volta de dezembro de 2026, e a base histórica sai por API ou arquivos tabulados.** Importar o histórico do Hórus é um diferencial de entrada para municípios que migrarem ([FATO] [Conass Informa 114/2026](https://www.conass.org.br/conass-informa-n-114-2026-publicada-a-portaria-gm-n-11-585-que-altera-a-portaria-de-consolidacao-gm-ms-no-1-de-28-de-setembro-de-2017-para-dispor-sobre-a-substituicao-do-sistema-hor/); [INFERÊNCIA] no uso comercial).

## Lacunas

1. O manual SI-BNAFAR v1.20 e o Swagger não foram lidos. Ficam sem confirmação os endpoints, os nomes dos campos JSON, a URL de produção, o fluxo OAuth (client_credentials?), os códigos de erro, a lista de "motivo de saída" e a tabela de "tipo de produto".
2. Não foi publicada, ou não se achou, a lista do art. 392 de insumos e equipamentos CATMAT de envio obrigatório.
3. Faltam os códigos CATMAT concretos de tira, lanceta, seringa de insulina e curativo, e a definição oficial de "unidade de fornecimento".
4. Não se confirmou se o REDFM já está em produção em 2026, nem o plano operativo da Portaria 6.100. Também não se confirmou se a RNDS exige certificado A1 para esse modelo.
5. Não se confirmou se a dispensação nominal de insumos deve ir a algum sistema federal. Não se achou procedimento SIGTAP nem ficha e-SUS APS para insumos de diabetes.
6. Os tetos mensais do SIGTAP para os códigos 0701050xxx/0701060xxx não foram conferidos; o "30/mês" vem de apresentação.
7. Não há regra oficial de quantidade de fraldas do Farmácia Popular; o PDF do elenco deu 404.
8. Não se leu na íntegra nenhum TR municipal de sistema de saúde de 2023 a 2026, e o PNCP não foi pesquisado.
9. A data exata da 11.585/2026 no DOU, o plano de transição da CIT (art. 391-J) e a portaria do incentivo Qualifar (Eixo Informação) não foram localizados.
10. Não se achou ato que suspenda o per capita do CBAF por falta de envio.
11. Não há números de adesão ao SI-BNAFAR de 2026, lista oficial de sistemas homologados nem relatos públicos de erros do SI-BNAFAR.
12. O texto consolidado da PRC 5/2017, art. 712, não foi lido.
13. Não se verificou se `health_units.cnes` está preenchido nas unidades com `dispenses = true` das cidades-piloto, nem se os profissionais têm CBO cadastrado. Isso é dado, não código.
14. Os specs do 27a e o contrato §13 H1–H17 em `contracts/` não foram lidos e podem conter decisões finas sobre insumo.
