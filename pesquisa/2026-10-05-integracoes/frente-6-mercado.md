# Frente 6 — Mercado e concorrentes (software de saúde municipal / SUS)

Data da pesquisa: 2026-10-05. Convenção: **[CONFIRMADO: url]** = lido na fonte; **[INFERÊNCIA]** = dedução minha a partir das fontes. Números de clientes são os **declarados pelo próprio fornecedor**, sem auditoria independente.

---

## 1. Resumo executivo

1. O mercado municipal é dominado por **suítes integradas** (recepção → agenda → prontuário → farmácia → regulação → faturamento BPA → vigilância), vendidas por licitação (pregão, menor preço global, com **prova de conceito** eliminatória) em modelo de **licença mensal + implantação + horas técnicas**, hospedado em nuvem. Faixa observada em município pequeno: R$ 11,8 mil a R$ 15,7 mil/mês + R$ 25 mil a R$ 157 mil de implantação [CONFIRMADO: editais Contenda/PR e Sangão/SC, ver §4].
2. O e-SUS APS PEC (gratuito) ganhou espaço: 59,93% das UBS com prontuário eletrônico usavam o PEC em 2022 (eram 20,82% em 2017), e sistemas próprios/terceiros caíram de 27,86% para 22,34% [CONFIRMADO: SciELO/RSP 2024]. Os privados sobrevivem **onde o PEC não chega**: integração com especializada/hospital, farmácia/estoque, regulação, TFD/transporte, faturamento e relatórios gerenciais.
3. Em **15 editais/TRs** lidos, aparecem em 100% deles: prontuário, agenda, acolhimento/classificação de risco, cadastro CNS/CADSUS, exportação e-SUS/SISAB, **farmácia/dispensação, odontologia, laboratório, regulação, faturamento BPA/SIGTAP, vacinas e transporte/viagens de pacientes**. Logo atrás, em 14 dos 15: CAPS/saúde mental, hospitalar (AIH/internação), vigilância epidemiológica e sanitária, BI e assinatura digital.
4. **Lacunas críticas do roteiro do Rota Saúde** frente aos editais: farmácia/dispensação + almoxarifado, odontologia, faturamento BPA/SIA (e APAC/AIH), laboratório, TFD/transporte sanitário, imunização (com envio à RNDS), CAPS/RAAS e vigilância sanitária. Sem elas, o produto **não passa da prova de conceito** de um pregão típico (Mafra/SC exigiu 438 itens obrigatórios de 666; Pato Branco/PR exigiu 95%).
5. A concorrência pública está crescendo: **Meu SUS Digital** já agenda consultas da UBS via gov.br em mais de 500 municípios (nov/2025); o **e-SUS AF** (gratuito, substitui o Hórus) foi lançado em abril/2026; o **AGHUse** (hospitalar, gratuito) atende 8 estados.
6. Só **um** fornecedor de APS municipal aparece na lista vigente de certificados SBIS consultada: **G-MUS v25 (Inovadora)**. SOUL MV, Tasy, AGHU e outros sistemas hospitalares/de clínica também constam. SBIS é exigida ou pontuada em 6 dos 15 editais.
7. Onde o Rota Saúde se diferencia: triagem digital do cidadão **antes** da UBS, com protocolos que a própria cidade assina; isolamento com um banco por cidade; analytics com privacidade. Nenhum edital lido pede "triagem do cidadão por protocolo". Isso diferencia, mas **não pontua** na planilha da licitação: precisa ser vendido como item adicional ou por inexigibilidade/inovação.

---

## 2. Tabela de fornecedores

| Fornecedor / produto | Módulos declarados | Envio ao SISAB (via LEDI) | Integrações declaradas | SBIS/CFM | Porte (declarado) | Contratação |
|---|---|---|---|---|---|---|
| **Celk Saúde** (Celk Sistemas, Florianópolis/SC) | APS Cloud Care, Cuidado Integrado (especializada), Acesso Regulador, Urgência Móvel (SAMU), Vigilância Sanitária, app Celk Saúde Cidadão [CONFIRMADO: https://celk.com.br/ ; proposta de Ilhota https://ilhota.sc.gov.br/wp-content/uploads/2023/07/2520857_Proposta_Comercial__.pdf] | Sim. A proposta fala em "+60 milhões de registros enviados ao MS" [CONFIRMADO: proposta Ilhota] | RNDS (tabela OBM-VMP de medicamentos, regras de vacina) [CONFIRMADO: https://ajuda.celk.com.br/news/noticias-e-comunicados/2025/10-07-2025-or-tela-de-consulta-obm-vmp-para-viabilizacao-da-interoperabilidade-com-a-rnds] | Não aparece na lista SBIS consultada | +250 municípios no site; "+280" em busca [CONFIRMADO: celk.com.br]. Casos citados: Florianópolis, Goiânia, Criciúma [CONFIRMADO: proposta Ilhota] | SaaS na AWS. O app do cidadão sozinho custava **R$ 600/mês** em Ilhota (2023) [CONFIRMADO: proposta Ilhota] |
| **IPM Saúde / Atende.Net** (IPM Sistemas, SC) | Prontuário multiprofissional (78 especialidades), agenda, vacinas, distribuição de medicamentos, laboratório, IA "Dara", geolocalização, reconhecimento de voz, cadastro único [CONFIRMADO: https://www.ipm.com.br/saude/] | Sim [INFERÊNCIA: "+90 sistemas do SUS e MS"] | "+90 sistemas do SUS/MS"; vacina COVID enviada em tempo real ao SI-PNI via RNDS [CONFIRMADO: https://ipm.com.br/noticias/saude/vacina-contra-covid-19-tecnologia-ipm-esta-integrada-ao-ministerio-da-saude] | Não aparece na lista SBIS | "+850 clientes municipais" em toda a gestão pública, não só saúde [CONFIRMADO: ipm.com.br/saude] | Nuvem própria desde 2008, Tier III. Vende dentro do ERP municipal Atende.Net [CONFIRMADO] |
| **G-MUS** (Inovadora Sistemas, MT) | RCOP/PEP, APS com mobilidade, agenda, autorização/APAC, regulação e fila, comunicação com o cidadão, laboratório, imagem, odontologia, imunização, epidemiologia/dengue, faturamento BPA/RAAS, BI [CONFIRMADO: https://www.inovadora.com.br/solucao/saude-municipal/] | Sim (e-SUS AB) [CONFIRMADO: idem] | DATASUS, SIGTAP, e-SUS AB, **RNDS, CADSUS, CNES, SISREG, SINAN**, assinatura digital, **login gov.br** [CONFIRMADO: idem] | **Sim: G-MUS v25, Clínica/Ambulatório, Estágio 2, válido até 12/03/2027** [CONFIRMADO: https://sbis.org.br/certificacoes/certificacao-software/sistemas-certificados/] | +200 municípios, 10 estados, 12 milhões de cidadãos [CONFIRMADO: inovadora.com.br] | Modular. Também é vendido via consórcio (CISAMURES) [CONFIRMADO: https://cisamures.g-cis.com.br/gmus.html] |
| **IDS Saúde** (IDS Software e Assessoria, PR) | 24+ módulos: PEP com IA (resumo da anamnese), APS, gestão hospitalar, app ACS, app endemias, VISA, central de agendamento e regulação, faturamento, BI (IDS Builder/Manager), portal do cidadão, **WhatsApp**, odontologia, saúde animal, teleconsulta [CONFIRMADO: https://ids.inf.br/solucoes/ids-saude/] | Sim (e-SUS) [CONFIRMADO: idem] | MS e laboratórios, ICP-Brasil [CONFIRMADO: idem] | Diz ser certificado SBIS [CONFIRMADO: https://medicinasa.com.br/ids-pep/], **mas não aparece na lista vigente consultada** [INFERÊNCIA: certificado vencido ou fora da lista] | Logos de ~10 municípios (Limeira/SP, Pato Branco/PR, Feira de Santana/BA) [CONFIRMADO: ids.inf.br]. Ganhou Pato Branco [CONFIRMADO: https://ids.inf.br/pato-branco-implanta-sistema-de-gestao-publica-da-saude-da-ids/] | Licitação. Não achei se o nome "Saúde Simples" foi dele. |
| **Vivver Saúde Pública** (Vivver, MG) | 30+ módulos: PEC, farmácia/estoque, hospitalar, vacinação, epidemiologia, faturamento, telemedicina, regulação/filas [CONFIRMADO: https://briefing-saude.vivver.com.br/] | Sim (e-SUS APS/SISAB) [CONFIRMADO: idem] | RNDS, SIGTAP/CNES/SCNES, SI-PNI, **BNAFAR/Hórus, GAL (laboratório)**, BPA/SIA, HL7/FHIR [CONFIRMADO: idem] | Não aparece na lista SBIS | 140+ municípios (SE/NE). Clientes: Contagem/MG, João Pessoa/PB, Volta Redonda/RJ [CONFIRMADO: idem; https://portal.contagem.mg.gov.br/vivver-sistema-contagem] | Nuvem Tier III, ISO 27001, SLA 99,7%, 120 dias de operação assistida [CONFIRMADO: briefing-saude] |
| **Olostech / SaúdeTech** (SC) | PEP, urgência, imunização, assinatura digital, ESF, regulação, farmácia/almoxarifado, **transporte de pacientes**, telemedicina [CONFIRMADO: https://www.olostech.com/post/municipios-destaque-em-gestao-da-saude-olostech] | [INFERÊNCIA: sim, pelos clientes de grande porte] | Laboratório municipal (Piracicaba) [CONFIRMADO: https://www.acate.com.br/noticias/implantada-a-integracao-sistema-olostech-com-laboratorio-municipal-de-piracicaba/] | Não aparece | "+100 municípios": Joinville, Jaraguá do Sul, Balneário Camboriú, Piracicaba, Praia Grande [CONFIRMADO: busca/olostech.com] | Não informado. Não confirmei que "Viver" é o nome dele. |
| **Fiorilli SIS** (SP) | Ambulatório, ACS, vacinas, **viagens**, farmácia, hospital, laboratório, banco de sangue, zoonoses, VISA, faturamento [CONFIRMADO: https://fiorilli.com.br/servicos/sis-sistema-integrado-de-saude/] | Sim (ESUS AB) [CONFIRMADO: idem] | BPA-MAG, SISAIH01, CIHA, RAAS, BNDASAF, TISS, **Hórus, Qualifar-SUS**, CNS [CONFIRMADO: idem] | Não aparece | Não informado | Licença dentro do ERP municipal [INFERÊNCIA] |
| **Betha Saúde / Fly Saúde** (SC) | Transporte, farmácia, faturamento, CAPS, ambulatório, prontuário, odontologia, **TFD**, AIH, APAC, regulação, teleatendimento [CONFIRMADO: busca sobre Betha + http://e-gov.betha.com.br/www-services/mediacenter/noticias/index.jsp?noticia=490&anoticia=2012&mes=2 ; https://www.acate.com.br/noticias/sistema-de-saude-betha-permite-que-municipios-facam-teleatendimento-durante-pandemia/] | [INFERÊNCIA: sim] | Hórus/BNDASAF [CONFIRMADO: https://test.betha.com.br/help/saude/sincronizacao_horus___bndasaf.htm]; manual de produção RNDS [CONFIRMADO: https://saude.ajuda.betha.cloud/novomanuais/RNDS/producaornds/]; plataforma de serviços municipais com gov.br (2025) [CONFIRMADO: https://inforchannel.com.br/2025/09/25/betha-sistemas-lanca-plataforma-que-unifica-servicos-municipais-com-gov-br/] | Não aparece | Não informado para saúde | Betha Store (marketplace), cloud [CONFIRMADO: https://betha.store/] |
| **SOUL MV Saúde Pública** (MV) | APS, gestão do cuidado e hospitalar, programas, regulação de alta complexidade, vigilância, assistência farmacêutica, diagnóstico, assistência social, transparência, relacionamento com o cidadão; atende UBS, CAPS, UPA, SAMU, laboratório [CONFIRMADO: https://mv.com.br/solucao/soul-mv-saude-publica] | [INFERÊNCIA: sim] | "Alinhado ao MS" (sem lista) | **Sim: SOUL MV v2020, várias categorias, Estágio 2, até 23/10/2026** [CONFIRMADO: lista SBIS] | Líder hospitalar na América Latina (KLAS 2025, citado em https://futurodasaude.com.br/philips-tasy-mercado/) | Atende capitais e estados [INFERÊNCIA] |
| **Tasy** (ex-Philips, vendido à Bionexo) | Hospitalar | n/a | n/a | **Sim: Tasy HTML5, Estágio 2, até 20/05/2028** (titular "Tucano do Brasil") [CONFIRMADO: lista SBIS] | ~500 hospitais [CONFIRMADO: futurodasaude]. Venda à Bionexo por cerca de €161 milhões (~R$ 1 bi) anunciada no fim de 2025 [CONFIRMADO: https://www.acate.com.br/noticias/philips-vende-software-desenvolvido-em-sc/ ; https://sysmiddle.com.br/tasy-philips/] | Privado/hospitais. Relevante só para a futura linha hospitalar. |
| **AGHU / AGHUse** (EBSERH / HCPA) | Gestão hospitalar | n/a | Padrões de interoperabilidade, software livre | **AGHU v11: certificado SBIS NGS1 em 05/06/2025, 198 requisitos** [CONFIRMADO: https://sbis.org.br/noticia/sbis-certifica-o-primeiro-sistema-publico-de-gestao-hospitalar-certificado-do-pais/] | AGHUse em 8 estados; na Bahia, rede de 417 municípios [CONFIRMADO: https://www.hcpa.edu.br/3911-software-de-gestao-desenvolvido-pelo-hcpa-ja-esta-presente-em-oito-estados-brasileiros] | **Gratuito** para gestores do SUS: concorrente direto da futura linha hospitalar |
| **Siss Saúde / SissOnline** (GIESPP, grupo Eicon) | Portal/app do cidadão (marcar consulta, exame, retirada de remédio, avaliação), UBS/UPA/especializada/laboratório, telemedicina [CONFIRMADO: https://piracicaba.sp.gov.br/noticias/prefeito-assina-contratacao-de-novo-software-para-a-saude/] | [INFERÊNCIA: sim] | — | Não aparece | Barueri, Guarulhos, Osasco, Santo André, Mauá (segundo terceiro, não verificado); Piracicaba: R$ 6,3 milhões por 24 meses (2023) [CONFIRMADO: piracicaba.sp.gov.br] | Licitação |
| **e-SUS APS PEC** (MS / Laboratório Bridge-UFSC) | PEC (RCOP/SOAP), CDS, apps ACS/Território, agenda; agendamento pelo Meu SUS Digital [CONFIRMADO: SciELO RSP 2024; https://www.conass.org.br/agora-e-possivel-marcar-consultas-nas-ubss-pelo-aplicativo-meu-sus-digital/] | É o próprio destino | SISAB/SIAPS, Meu SUS Digital, RNDS | Não aparece | 26.091 UBS em dez/2022 (59,93% das UBS informatizadas); "mais de 4.000 municípios" [CONFIRMADO: SciELO RSP 2024; https://www.jasb.com.br/2026/03/prontuario.html?m=1] | **Gratuito**. Arquiteturas centralizada, descentralizada ou multimunicipal |
| **e-SUS AF** (MS) | Dispensação, estoque, integração por API | — | BNAFAR, RNDS, CADSUS | — | Adesão aberta em abr/2026 | **Gratuito**, substitui o Hórus (adesão voluntária) [CONFIRMADO: https://site.cff.org.br/noticia/Noticias-gerais/13/04/2026/sus-lanca-novo-sistema-digital-para-modernizar-a-assistencia-farmaceutica-em-todo-o-pais] |

**Candidatos não confirmados:** "Govbr/Sistemas Públicos", "WPD/Vitae", "Pronto (e-SUS)" e "Saúde Simples". As buscas não retornaram produtos com esses nomes no segmento municipal [INFERÊNCIA: nome incorreto ou empresa pequena e regional]. Achei também: System Sistemas (systempro.com.br), Maestro Sistemas, Medicina Direta (PEP certificado SBIS Estágio 3, com oferta para o setor público: https://medicinadireta.com.br/otimizado/setor-publico-e-sus), Equiplano/Elotech [INFERÊNCIA: ERPs municipais com saúde, não verificados].

---

## 3. Dores e motivações de troca

### Por que o município sai do PEC (ou nunca entra) e contrata um privado
- **Integração com especializada e hospital.** O PEC "não foi desenvolvido com o objetivo de ser integrado a sistemas terceiros", e gestores "tendem a priorizar aparelhos que operem nos diferentes pontos de atenção" [CONFIRMADO: SciELO RSP 2024, https://www.scielo.br/j/rsp/a/7jZL8DrBTxtGjDBzCTRBCGH/?format=pdf&lang=pt]. Em 2024, só 25,3% das UBS compartilhavam o PEC com serviços especializados públicos (Censo 2024, citado em busca) [CONFIRMADO via busca; fonte primária não aberta].
- **Escopo que o PEC não cobre.** O ETP de Sangão/SC (2025) compara as alternativas. Sobre os sistemas federais gratuitos, lista "funcionalidades limitadas e pouca flexibilidade", "dependência da atualização e estabilidade do governo federal" e "pode não atender todas as necessidades específicas do município". Conclui que, pelo tempo, pela **migração do histórico** e pelas funcionalidades específicas, só resta contratar empresa especializada [CONFIRMADO: https://sangao.sc.gov.br/uploads/sites/342/2025/03/ETP.pdf].
- **Indicadores e repasse federal.** Com o Previne Brasil, municípios passaram a contratar empresas para extrair dados e gerar indicadores das equipes [CONFIRMADO: SciELO RSP 2024]. Desde 1/1/2026, dados enviados por versão com mais de 12 meses são invalidados no SIAPS, **inclusive os de sistemas próprios via LEDI** (Notas Informativas SAPS 12 e 13/2025) [CONFIRMADO: https://p2saude.com.br/risco-de-apagao-de-dados-versoes-desatualizadas-do-e-sus-aps-serao-bloqueadas-em-2026/ ; https://sisaps.saude.gov.br/sistemas/esusaps/assets/files/NI_13-2025_cenario_versoes_incompativeis-90647909abe17697641f1a44b859e48a.pdf]. Isso pesa nos dois sentidos: o município quer quem garanta a conformidade, e o fornecedor ganha uma obrigação recorrente.
- **Absenteísmo e acesso.** Piracicaba justificou o contrato de R$ 6,3 milhões com 1.782 faltas em 10.291 consultas especializadas agendadas (nov/2022) e com o app do cidadão [CONFIRMADO: piracicaba.sp.gov.br]. A Celk anuncia 50% menos faltas e 43% menos problemas na dispensação [CONFIRMADO: celk.com.br].
- **Offline e mobilidade.** 11 dos 15 editais exigem app (ACS/tablet) que funcione offline. Bom Jesus do Tocantins/PA pede "100% OFFLINE/ONLINE" [CONFIRMADO: https://bomjesusdotocantins.pa.gov.br/wp-content/uploads/2021/10/TERMO-DE-REFERENCIA-1-12.pdf].
- **Suporte e implantação.** O município pequeno não tem TI. O PEC exige instalação e servidor, e a literatura cita rotatividade de profissionais e falta de treinamento [CONFIRMADO: SciELO RSP 2024]. O privado vende implantação, treinamento e horas técnicas.

### Por que o município volta ao PEC (ou fica nele)
- **Custo zero e integração federal nativa.** Jaú/SP migrou para o PEC em maio/2022 para usar a plataforma gratuita e liberar recursos para outras áreas [CONFIRMADO: https://www.jau.sp.gov.br/noticia/11253/saude-passa-a-utilizar-novo-sistema-informatizado].
- **Menos retrabalho no envio ao SISAB.** O próprio Ministério recomenda não registrar a mesma informação em mais de um instrumento [CONFIRMADO: busca, sisaps CDS].
- **O PEC cresce em funcionalidades:** agendamento pelo Meu SUS Digital em mais de 500 municípios, com reserva de 3 vagas por profissional e turno [CONFIRMADO: https://www.conass.org.br/agora-e-possivel-marcar-consultas-nas-ubss-pelo-aplicativo-meu-sus-digital/ ; https://www.gov.br/secom/pt-br/acompanhe-a-secom/noticias/2026/03/populacao-brasileira-ja-pode-agendar-consultas-pelo-aplicativo-meu-sus-digital], e Painel e-SUS APS com a Fiocruz [CONFIRMADO: busca].
- **Infraestrutura deixou de ser barreira.** Manchete do MS (jun/2025): "Mais de 87% das UBS utilizam prontuário eletrônico e quase a totalidade tem acesso à internet" [CONFIRMADO, só o título: https://www.gov.br/saude/pt-br/assuntos/noticias/2025/junho/mais-de-87-das-unidades-basicas-de-saude-utilizam-prontuario-eletronico-e-quase-a-totalidade-tem-acesso-a-internet].

### Dores recorrentes (síntese)
Dupla digitação PEC × sistema de gestão; farmácia e estoque fora do PEC; regulação, filas e TFD em planilha; faturamento BPA manual; relatórios gerenciais e indicadores do financiamento; conectividade em área rural (offline); troca de versão do e-SUS (risco de perder dados); migração do histórico na troca de fornecedor [INFERÊNCIA consolidada das fontes acima].

---

## 4. Módulos recorrentes em editais (amostra de 15)

**Amostra (PDFs públicos baixados e lidos por busca textual):**
- Saúde exclusiva (11): Bom Jesus do Tocantins/PA 2021; Contagem/MG (SIGESC, módulos de gestão ambulatorial, farmacêutica, vigilância, regulação, urgência e logística); Contenda/PR; Mafra/SC 2025; Pato Branco/PR 2023; Pedra Preta/MT (saúde + hospitalar + vigilância ambiental); Planaltina/GO 2025; Sangão/SC 2025 (ETP); São Carlos/SP 2024; Sorriso/MT (adesão à ata do consórcio CIMCERO/RO); Xanxerê/SC 2022.
- ERP municipal com lote de saúde (4): Catanduvas/SC 2023; Rio das Antas/SC 2022; Monte Castelo/SC 2025; Salto/SP 2025.

Método: busca textual por termos-chave (sem acento, sem distinção de maiúsculas) e checagem manual de contexto em amostras. A contagem indica que o tema **aparece** no documento, não que seja lote separado [INFERÊNCIA metodológica]. Em "Transporte" usei "viagens/transporte de pacientes/transporte sanitário"; o contexto foi verificado (ex.: "Agendamento de Viagens" em Xanxerê).

| Módulo / requisito | Editais (de 15) | Está no roteiro do Rota Saúde? |
|---|---|---|
| Prontuário eletrônico | 15 | Sim (próximo ciclo) |
| Agendamento / agenda | 15 | Sim (já tem agendamento, parcial) |
| Acolhimento / classificação de risco | 15 | Sim (triagem + acolhimento) |
| Cadastro único CNS / CADSUS | 15 | Parcial (CPF/CNS; integração CADSUS a confirmar) |
| Exportação e-SUS / SISAB (LEDI/Thrift) | 15 | Necessário no modo "convive com PEC" e no modo "substitui PEC" |
| Procedimentos / SIGTAP | 15 | Implícito em procedimentos |
| **Farmácia / dispensação** | 15 | **NÃO** |
| **Odontologia** (odontograma) | 15 | **NÃO** |
| **Laboratório** (requisição, coleta, laudo) | 15 | Parcial ("exames") |
| Regulação (consultas, exames, fila) | 15 | Sim (regulação MAC) |
| **Faturamento BPA / SIA-SUS** | 15 | **NÃO** |
| **Vacinas / imunização** (com RNDS) | 15 | **NÃO explícito** (campanhas ≠ registro de vacina) |
| **Transporte / viagens de pacientes** | 15 | **NÃO** |
| Assinatura digital / ICP-Brasil | 14 | A definir (protocolos assinados, mas não é ICP) |
| **CAPS / saúde mental (RAAS)** | 14 | Linha de cuidado "saúde mental" (sem RAAS/CAPS) |
| **Hospitalar / AIH / internação** | 14 | Linha separada futura |
| Vigilância epidemiológica / SINAN / notificação | 14 | Sim (vigilância epidemiológica) |
| **Vigilância sanitária** | 14 | Linha separada futura |
| BI / indicadores / painel gerencial | 14 | Sim (analytics) |
| CNES | 14 | Parcial (unidades/profissionais) |
| Programas (hiperdia, pré-natal, puericultura) | 14 | Sim (linhas de cuidado) |
| Nuvem / datacenter | 14 | Sim |
| Painel de chamada / fila / senha | 14 | Sim (fila e chamada) |
| **Almoxarifado / estoque** | 13 | **NÃO** |
| Hórus / BNAFAR / BNDASAF | 13 | **NÃO** |
| Georreferenciamento | 13 | Sim (território) |
| WhatsApp / SMS ao paciente | 12 | Parcial (SMS desligado até go-live) |
| ACS / visita domiciliar | 12 | Sim (atendimento domiciliar) |
| PPI / cotas / consórcio | 12 | Parcial (regulação) |
| Triagem (termo literal) | 12 | Sim (diferencial) |
| **TFD** | 11 | **NÃO** |
| App offline / tablet | 11 | **NÃO** (web) |
| Migração / conversão de dados | 11 | Processo, não módulo |
| App / portal do cidadão | 10 | Sim (wpda) |
| LGPD | 10 | Sim |
| Urgência / UPA / pronto-socorro | 10 | **NÃO** |
| Zoonoses / endemias / vigilância ambiental | 8 | **NÃO** |
| Integração RNDS (quase sempre **vacinação**) | 8 | A planejar |
| Exames de imagem / radiologia | 7 | Parcial (exames) |
| Telessaúde / teleconsulta | 7 | **NÃO** |
| Ouvidoria | 7 | **NÃO** |
| Certificação SBIS (exigida ou pontuada) | 6 | **NÃO** |
| Classificação de Manchester (literal) | 6 | A avaliar |
| SISREG | 2 | A avaliar |
| HL7 / FHIR (literal) | 2 | A planejar |

Fontes da amostra: https://bomjesusdotocantins.pa.gov.br/wp-content/uploads/2021/10/TERMO-DE-REFERENCIA-1-12.pdf · https://consultapublica.contagem.mg.gov.br/storage/anexos/AnEWIXyIXrCjkgsNY9iTPKSCIlQL8kSLnTwarYuQ.pdf · https://www.contenda.pr.gov.br/prefeitura_app/storage/app/public/licitacoes/Edital-Sistema-de-Gestao-Saude.pdf · https://mafra.sc.gov.br/uploads/sites/372/2025/02/Edital-Pregao-Eletronico-n%C2%B0-010.2025-PR-022.2025-Contratacao-de-empresa-licenca-de-uso-de-sistema-informatizado-em-gestao-de-saude-Sec.-Saude.pdf · https://patobranco.pr.gov.br/wp-content/uploads/2023/04/41-SOFTWARE-GESTAO-DE-SAUDE-PUBLICA.pdf · https://www.pedrapreta.mt.gov.br/fotos_licitacao/6452.pdf · https://planaltina.go.gov.br/wp-content/uploads/2025/09/Edital_de_pre-qualificacao_processo_administrativo_no_31060-2025.pdf · https://sangao.sc.gov.br/uploads/sites/342/2025/03/ETP.pdf · https://servico.saocarlos.sp.gov.br/licitacao/arquivos-resumos/07-16-2024-04:46:59-pm-8345-6866--Resumo-.pdf · https://site.sorriso.mt.gov.br/dl/103199 · https://xanxere.sc.gov.br/uploads/sites/92/2022/06/Termo-de-Referencia-Sistema-de-Gestao-Saude.pdf · https://catanduvas.sc.gov.br/uploads/sites/270/2023/04/Termo-Referencia-Gestao-Publica-2023.pdf · https://riodasantas.sc.gov.br/uploads/sites/547/2022/11/2311058_TERMO_DE_REFERENCIA____SISTEMA_DE_GESTAO_PUBLICA.pdf · https://montecastelo.sc.gov.br/uploads/sites/449/2025/11/ANEXO-I-TERMO-DE-REFERENCIA-1.pdf · https://salto.sp.gov.br/wp-content/uploads/2026/06/Edital-Pregao-eletronico-no-63-2025-Sistemas-SAAS-Republicacao-3.pdf

**Padrões de contratação observados:**
- **Prova de conceito eliminatória.** Mafra: roteiro de 666 itens funcionais, 438 (66%) obrigatórios. Pato Branco: 95% dos requisitos. Contagem: 90% [CONFIRMADO: editais citados].
- **Preço.** Contenda/PR: implantação R$ 25.011,96; manutenção, suporte e hospedagem R$ 11.833,33/mês (12 meses); 250 h técnicas a R$ 139,87; total R$ 201.979,42. Sangão/SC: implantação R$ 156.843,33; R$ 15.711,25/mês; total R$ 406.188,33. Xanxerê: estimativa de R$ 274.253,92. Piracicaba: R$ 6,3 milhões em 24 meses [CONFIRMADO].
- **Modalidades.** Pregão por menor preço global (lote único), adesão a ata de consórcio (Sorriso → CIMCERO), "licença de uso perpétuo" (São Carlos), PaaS/SaaS (Pato Branco, Salto) [CONFIRMADO].
- **RNDS nos editais** aparece quase só como envio de **vacinação**, com tela de conferência de rejeições (Sangão, Pato Branco, Contenda) [CONFIRMADO].

---

## 5. Lacunas do roteiro do Rota Saúde (por prioridade de edital)

1. **Farmácia / dispensação + almoxarifado + BNAFAR** (15 e 13 de 15). Maior lacuna. Concorre agora com o **e-SUS AF gratuito**, então a alternativa "integrar com o e-SUS AF por API" precisa ser avaliada contra "construir" [INFERÊNCIA].
2. **Faturamento BPA/SIA (+ APAC, RAAS, AIH)** (15 de 15). É o que paga a conta da secretaria. Sem ele, o gestor mantém outro sistema [INFERÊNCIA].
3. **Odontologia com odontograma** (15 de 15). Exigida até em edital de cidade pequena.
4. **Imunização** com registro de vacina e envio à RNDS (15 de 15). "Campanhas" do roteiro não cobrem o registro de dose.
5. **Transporte / viagens de pacientes e TFD** (15 e 11 de 15). Típico de município pequeno e médio.
6. **Laboratório** (requisição, coleta, resultado, integração GAL/LACEN) (15 de 15).
7. **CAPS / RAAS** (14 de 15): a linha de cuidado de saúde mental não cobre o faturamento RAAS.
8. **Urgência/UPA** (10 de 15), **app offline para ACS** (11 de 15), **ouvidoria** (7 de 15), **telessaúde** (7 de 15), **zoonoses/endemias** (8 de 15).
9. **Conformidade**: certificação SBIS (6 de 15), assinatura ICP-Brasil (14 de 15), atualização LEDI dentro da janela de 12 meses.

[INFERÊNCIA] Para a primeira licitação, o mínimo competitivo é: farmácia, BPA, odontologia, vacinas e transporte/TFD, além do que já está no roteiro. Sem esse pacote, o caminho é vender o Rota Saúde como **complemento** (triagem, fila, regulação, analytics) ao lado do PEC ou de um ERP existente, por contrato menor ou dispensa.

---

## 6. Diferenciais possíveis

- **Triagem digital do cidadão antes da UBS, com protocolo que a cidade escreve e assina.** Nenhum concorrente pesquisado declara isso. Os apps de cidadão do mercado (Celk Cidadão, Siss, IPM, Meu SUS Digital) **agendam** e **consultam histórico**, mas não fazem triagem por protocolo municipal [CONFIRMADO para o escopo declarado; ausência = INFERÊNCIA]. Também se encaixa no Meu SUS Digital, que reserva só 3 vagas por profissional e turno: a triagem pode decidir quem precisa delas [INFERÊNCIA].
- **Governança clínica auditável.** Duas assinaturas de revisor por publicação de protocolo e trilha de versões atendem a demandas de controle interno e do Ministério Público que a concorrência não endereça [INFERÊNCIA].
- **Isolamento com um banco por cidade + LGPD (exclusão e anonimização).** Útil contra exigências de LGPD (10 de 15 editais) e em consórcios multimunicipais [INFERÊNCIA].
- **Modo de prontuário por cidade (convive ou substitui o PEC).** O mercado vende "substituir". Oferecer "conviver" (triagem, fila, regulação e analytics sobre o PEC, sem dupla digitação) reduz a barreira de entrada [INFERÊNCIA]. Exige ler e escrever dados do PEC, que não foi feito para integrar com terceiros [CONFIRMADO: SciELO].
- **Analytics com privacidade + indicadores do novo financiamento da APS,** prontos para o gestor (o mercado contrata isso à parte desde o Previne) [CONFIRMADO: SciELO; INFERÊNCIA sobre a oportunidade].
- **Stack web moderna.** O mercado declara J2EE (Celk) e sistemas de mais de 20 anos (Olostech 29 anos, Vivver desde 1999) [CONFIRMADO]. A vantagem real depende da usabilidade comprovada na PoC [INFERÊNCIA].

---

## 7. Riscos

- **Concorrência gratuita do MS se expandindo:** PEC, Meu SUS Digital (agendamento), e-SUS AF (farmácia), AGHUse (hospital, gratuito) e Painel e-SUS APS [CONFIRMADO].
- **Prova de conceito** com centenas de itens favorece suítes maduras. Um produto novo perde por cobertura, não por qualidade [CONFIRMADO: Mafra, Pato Branco].
- **Lote único / menor preço global** une saúde ao ERP municipal (4 dos 15 editais) e exclui quem só faz saúde [CONFIRMADO].
- **Regra de 12 meses do LEDI:** custo recorrente de conformidade. Atraso derruba o repasse do município [CONFIRMADO: NI 12 e 13/2025].
- **RNDS:** credenciamento em duas fases (homologação → produção), com gov.br, e-CNPJ ICP-Brasil e evidências aprovadas pelas áreas finalísticas. **Não há prazo publicado** [CONFIRMADO: https://rnds-guia.saude.gov.br/docs/passo-a-passo/].
- **SBIS:** processo com inscrição, taxa, auditoria (um ou mais ciclos) e conclusão [CONFIRMADO: https://www.sbis.org.br/certificacao/Manual_Certificacao_S-RES_SBIS_v5-0.pdf]. O AGHU foi avaliado em 198 requisitos e a SBIS observa que sistemas grandes têm mais dificuldade [CONFIRMADO]. **Prazo e custo não publicados.**
- **Migração do histórico** do fornecedor anterior é requisito em 11 dos 15 editais e pesou na decisão de Sangão [CONFIRMADO].
- **Incumbentes regionais fortes** (SC: Celk, IPM, Betha, Olostech; MT: G-MUS; MG/NE: Vivver) [CONFIRMADO, presença; INFERÊNCIA, força].

---

## 8. Perguntas em aberto

1. Qual a fatia atual (2025/2026) de UBS com PEC × sistema próprio? O dado mais recente que achei é de dez/2022 (59,93% × 22,34%). O Censo UBS 2024/2025 não abriu (página do MS com acesso restrito).
2. Quanto tempo e quanto custa, na prática, homologar na RNDS e certificar na SBIS? Não há número público. É preciso perguntar a fornecedores ou à própria SBIS.
3. O e-SUS AF expõe API de escrita para sistemas terceiros (dispensar a partir do Rota Saúde) ou só recebe dados? Isso decide entre construir farmácia ou integrar.
4. O PEC/Meu SUS Digital permite que terceiro **leia a agenda e reserve vagas**, para ligar a triagem do Rota Saúde ao agendamento federal? Sem isso, o modo "convive com PEC" vira dupla digitação.
5. Quantos municípios contratam por consórcio (CIMCERO, CISLESTE, CISAMURES)? O consórcio pode ser o canal de entrada mais barato.
6. Por qual caminho entrar com um produto não suíte: inexigibilidade, Marco Legal das Startups (CPSI) ou contrato complementar ao lado do ERP? Precisa de parecer jurídico.
7. IDS Saúde: a certificação SBIS que ela anuncia está vencida? Ela não aparece na lista vigente consultada.
8. Quem são "Govbr/Sistemas Públicos", "WPD/Vitae", "Pronto (e-SUS)" e "Saúde Simples" na lista original? Não os encontrei.
9. Os números de clientes (Celk 250+, IPM 850+ da gestão toda, G-MUS 200+, Vivver 140+, Olostech 100+) são autodeclarados. Dá para cruzar com contratos no PNCP?

---

## 9. Fontes principais

- Celuppi IC et al. Dez anos do PEC e-SUS APS. Rev Saúde Pública 2024 — https://www.scielo.br/j/rsp/a/7jZL8DrBTxtGjDBzCTRBCGH/?format=pdf&lang=pt
- SISAB FAQ / LEDI — https://sisab.saude.gov.br/paginas/acessoPublico/faq/IndexFaq.xhtml ; https://integracao.esusaps.bridge.ufsc.tech/ledi/index.html
- NI 13/2025 SAPS — https://sisaps.saude.gov.br/sistemas/esusaps/assets/files/NI_13-2025_cenario_versoes_incompativeis-90647909abe17697641f1a44b859e48a.pdf ; resumo: https://p2saude.com.br/risco-de-apagao-de-dados-versoes-desatualizadas-do-e-sus-aps-serao-bloqueadas-em-2026/
- RNDS passo a passo — https://rnds-guia.saude.gov.br/docs/passo-a-passo/
- SBIS lista de certificados — https://sbis.org.br/certificacoes/certificacao-software/sistemas-certificados/ ; manual v5.0 — https://www.sbis.org.br/certificacao/Manual_Certificacao_S-RES_SBIS_v5-0.pdf ; AGHU — https://sbis.org.br/noticia/sbis-certifica-o-primeiro-sistema-publico-de-gestao-hospitalar-certificado-do-pais/
- Meu SUS Digital agendamento — https://www.conass.org.br/agora-e-possivel-marcar-consultas-nas-ubss-pelo-aplicativo-meu-sus-digital/ ; https://www.gov.br/secom/pt-br/acompanhe-a-secom/noticias/2026/03/populacao-brasileira-ja-pode-agendar-consultas-pelo-aplicativo-meu-sus-digital
- e-SUS AF — https://site.cff.org.br/noticia/Noticias-gerais/13/04/2026/sus-lanca-novo-sistema-digital-para-modernizar-a-assistencia-farmaceutica-em-todo-o-pais
- SUS Digital — https://www.conass.org.br/estados-municipios-e-distrito-federal-ja-podem-participar-do-programa-sus-digital/
- Fornecedores: https://celk.com.br/ · https://www.ipm.com.br/saude/ · https://www.inovadora.com.br/solucao/saude-municipal/ · https://ids.inf.br/solucoes/ids-saude/ · https://briefing-saude.vivver.com.br/ · https://www.olostech.com/post/municipios-destaque-em-gestao-da-saude-olostech · https://fiorilli.com.br/servicos/sis-sistema-integrado-de-saude/ · https://betha.store/ · https://mv.com.br/solucao/soul-mv-saude-publica · https://www.hcpa.edu.br/3911-software-de-gestao-desenvolvido-pelo-hcpa-ja-esta-presente-em-oito-estados-brasileiros · https://futurodasaude.com.br/philips-tasy-mercado/
- Casos municipais: Jaú — https://www.jau.sp.gov.br/noticia/11253/saude-passa-a-utilizar-novo-sistema-informatizado ; Piracicaba — https://piracicaba.sp.gov.br/noticias/prefeito-assina-contratacao-de-novo-software-para-a-saude/
- Editais: os 15 listados no §4.
