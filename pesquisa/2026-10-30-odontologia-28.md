# Pesquisa — Módulo 28 (odontologia) — 2026-10-30

Marcas: [CONFIRMADO] lido na fonte oficial · [SECUNDÁRIO] fonte não oficial (COSEMS, artigo) · [INFERÊNCIA] conclusão minha.

---

## 1. LEDI — Ficha de Atendimento Odontológico Individual (FAO / MIAOI)

Fonte: https://integracao.esusaps.bridge.ufsc.tech/ledi/documentacao/estrutura_arquivos/dicionario-fao.html (o domínio antigo integracao.esusab.ufsc.br redireciona para cá). Tabelas de domínio: https://integracao.esusaps.bridge.ufsc.tech/ledi/documentacao/referencias/dicionario.html

- Versão do layout na página: **LEDI APS 8.7.0**, SIGTAP competência 06/2026. [CONFIRMADO]
- **Dente/face/sextante: NÃO existem no LEDI.** O procedimento é `ProcedimentoQuantidade { coMsProcedimento (SIGTAP sem pontuação), quantidade }`. Há uma tabela "procedimentos por local e quantidade máxima" que só limita a quantidade: Arcada ≤ 2, Dente ≤ 32, Sextante ≤ 6. Procedimento fora dessa lista: quantidade máxima 1. [CONFIRMADO]
  - No PEC, o odontograma existe (a NT 32/2026 manda selecionar o elemento ou a arcada), mas isso é dado do prontuário. O que vai ao SIAPS é código + quantidade. [CONFIRMADO para o PEC / INFERÊNCIA para o envio]

**Master** (`FichaAtendimentoOdontologicoMaster`): uuidFicha (CNES + "-" + UUID recomendado), headerTransport (`VariasLotacoesHeader`), atendimentosOdontologicos (1–99), tpCdsOrigem = 3. [CONFIRMADO]

**Child** (`FichaAtendimentoOdontologicoChild`), campos principais [CONFIRMADO]:

| Campo | Obrig. | Observação |
|---|---|---|
| dtNascimento, sexo, turno | sim | |
| cpfCidadao / cnsCidadao | um ou outro (mutuamente exclusivos) | + `stCidadaoNaoPossuiCpf` (obrig.) e `justificativaNaoPossuiCpf` |
| numProntuario | não | |
| **gestante** | sim (boolean) | proibido se sexo masculino |
| **necessidadesEspeciais** | não (boolean) | |
| localAtendimento | sim | 1–10 |
| **tipoAtendimento** | sim | 2 consulta agendada · 4 escuta inicial/orientação · 5 consulta no dia · 6 urgência |
| **tiposConsultaOdonto** | condicional (0–1) | 1 primeira consulta programática · 2 retorno · 4 manutenção. Obrigatório se tipoAtendimento=2, proibido se 4, opcional se 5; com 6, não pode ser retorno |
| **tiposVigilanciaSaudeBucal** | sim (1–7) | 1 abscesso dentoalveolar · 2 alteração em tecidos moles · 3 dor de dente · 4 fendas/fissuras labiopalatais · 5 fluorose moderada/severa · 6 traumatismo dentoalveolar · 99 não identificado |
| **tiposFornecimOdonto** | não (0–3) | 1 escova · 2 creme dental · 3 fio dental |
| **tiposEncamOdonto** (conduta/desfecho) | sim (1–17) | Encaminhamentos 1–11 (1 PNE · 2 cirurgia BMF · 3 endodontia · 4 estomatologia · 5 implantodontia · 6 odontopediatria · 7 ortodontia/ortopedia · 8 periodontia · 9 prótese · 10 radiologia · 11 outros) + 12 agendamento p/ outros profissionais AB · 14 grupos · 15 **tratamento concluído** (só se tipo de consulta = primeira consulta ou retorno) · 16 retorno p/ consulta agendada · 17 alta do episódio (vedado com primeira consulta ou retorno) · 18 agendamento eMulti |
| procedimentosRealizados | não (0–20) | `ProcedimentoQuantidade`. Grupos SIGTAP 01–04, telessaúde (08), lista do grupo 07. Proibido 03.01.04.007-9 (usar tipoAtendimento=4) |
| exame | não (0–100) | codigoExame (SIGTAP grupo 02 ou código AB) + solicitadoAvaliado |
| resultadosExames | não (0–10) | |
| **Medicamentos** | não (0–15) | codigoCatmat, viaAdministracao, dose, doseUnica/usoContinuo, frequência, duração, quantidadeReceitada, doses por turno |
| **Encaminhamentos** | não (0–10) | especialidade (exige o código correspondente em tiposEncamOdonto), hipoteseDiagnosticoCID10, classificacaoRisco. Vedado para os CBOs 322405, 322415, 322425 e 322430 (TSB/ASB) |
| **problemasCondicoes** (CID-10/CIAP-2) | sim (mín. 1) | ciap ou cid10, situacao, datas, isAvaliado, uuids de evolução |
| medicoes | não | PA, FC, glicemia, peso, altura etc. |
| ivcf | não | só para ≥ 60 anos |
| solicitacoesOci | não | SIGTAP grupo 09 (OCI) |
| tipoParticipacaoCidadao / ProfissionalConvidado | não | teleatendimento |
| dataHoraInicial/FinalAtendimento | sim | |

Observação: `problemasCondicoes` é obrigatório (mín. 1). Toda FAO precisa de pelo menos um CID-10 ou CIAP-2. [CONFIRMADO]

## 2. CBOs aceitos (Tabela 4 de regras/cbo.html)

Fonte: https://integracao.esusaps.bridge.ufsc.tech/ledi/documentacao/regras/cbo.html [CONFIRMADO]

- Toda a família de cirurgiões-dentistas: 223204, 223208, 223212, 223216, 223220, 223224, 223228, 223232, 223236, 223240, 223244, 223248, 223252, 223256, 223260, 223264, 223268, 223272, 223276, 223280, 223284, 223288, 223293.
- TSB: 322405 e 322425 (ESF). ASB: 322415 e 322430 (ESF).
- **Não estão** na tabela: 322410 (protético dentário) e 322420 (auxiliar de prótese). [CONFIRMADO, por ausência]
- ASB e TSB podem registrar a ficha, mas não podem registrar `Encaminhamentos`. [CONFIRMADO]
- Os especialistas (endodontista, periodontista, PNE, BMF…) já são aceitos, o que serve ao CEO na mesma ficha. [CONFIRMADO]

## 3. Registro da produção do CEO e do LRPD

**Mudança grande de 2026: a produção do CEO e do LRPD saiu do SIA (BPA) e foi para o SIAPS (e-SUS APS).**

Fonte: Nota Técnica nº 32/2026-CGSB/DESF/SAPS/MS, assinada em 18/03/2026, https://www.gov.br/saude/pt-br/composicao/saps/publicacoes/notas-tecnicas-e-informativas/nota-tecnica-no-32-2026-cgsb-desf-saps-ms, referente à **Portaria GM/MS nº 10.192, de 05/02/2026**. [CONFIRMADO na NT; o texto da portaria em si não foi lido]

- O SIAPS passa a ser o sistema oficial de monitoramento do CEO e do LRPD. O registro é feito no PEC e-SUS APS, no CDS (só de forma transitória) ou em **sistema próprio integrado via LEDI** (API LEDI ao PEC, a partir da versão 5.3.19, com HTTPS e credenciais geradas pelo administrador da instalação). [CONFIRMADO]
- Quem adota o e-SUS APS fica dispensado do BPA ao SIA (item 6.5). [CONFIRMADO]
- Transição de 180 dias a partir de 06/02/2026, **encerrada em 05/08/2026**. Durante a transição era proibido enviar a mesma produção aos dois sistemas. Depois dela, **produção enviada por BPA não conta para monitoramento nem financiamento** dos serviços habilitados (item 8.3). [CONFIRMADO]
- Prazo de envio: mensal, até o 10º dia útil do mês seguinte (PC SAPS nº 1/2021). [CONFIRMADO]
- **CEO:** cadastro no SCNES sem mudança. Procedimentos e metas do **Anexo XL da PC GM/MS nº 6/2017** (origem: Portaria 1.464/2011) continuam monitorados, agora pelo SIAPS. [CONFIRMADO]
- **LRPD:** a produção é registrada **na entrega da prótese, pelo cirurgião-dentista, no CNES da UBS ou CEO que entrega** (não no CNES do laboratório). No PEC, a prótese fixa é lançada no odontograma por elemento (07.01.07.014-5) e as removíveis por arcada. Para LRPD privado de outro município, a unidade que entrega registra o serviço como "Terceiro" no SCNES. [CONFIRMADO]
- Códigos de prótese monitorados (Quadro 3 da NT é imagem e não foi lido; lista tirada da Portaria 1.825/2012 via COSEMS-SC): 07.01.07.012-9 PT mandibular · 07.01.07.013-7 PT maxilar · 07.01.07.009-9 PPR mandibular · 07.01.07.010-2 PPR maxilar · 07.01.07.014-5 coronária/intrarradicular fixa/adesiva por elemento. [SECUNDÁRIO]
- Faixas do LRPD (repasse mensal): 20–50 próteses/mês R$ 7.500 · 51–80 R$ 12.000 · 81–120 R$ 18.000 · >120 R$ 22.500. As etapas clínicas não contam, só as 5 próteses. Fonte: Nota Informativa COSEMS-SC 023/2025, https://www.cosemssc.org.br/wp-content/uploads/2025/08/1-Nota-Informativa-COSEMS-SC-no-023_2025-Saude-Bucal-e-os-LRPD.pdf [SECUNDÁRIO]
- Antes de 2026 (histórico): o LRPD registrava BPA-I no próprio CNES (exige CNS) e, se o LRPD fosse privado de outro município, BPA-C no CNES da unidade. Não havia APAC para prótese dentária. [SECUNDÁRIO, mesma nota COSEMS-SC]
- **BPA-I no CEO:** não encontrei lista oficial atual de procedimentos do CEO que exijam BPA-I. Depois de 05/08/2026, isso deixa de valer para o monitoramento federal. [INFERÊNCIA]
- O LRPD precisa de pelo menos 1 profissional CBO 3224-10 (protético) e/ou 2232 com carga SUS. [CONFIRMADO, NT 32]

Relacionado: a Portaria GM/MS nº 751/2023 **não trata de metas do CEO**. Ela cria o **Sesb** (Serviço de Especialidades em Saúde Bucal, para municípios com até 20 mil habitantes e sem CEO) e já exigia PEC e-SUS APS ou sistema terceiro e envio ao SISAB. Suspende o custeio após 3 competências sem envio. Fonte: https://www.in.gov.br/en/web/dou/-/portaria-gm/ms-n-751-de-15-de-junho-de-2023-490746192 (cópia em cosemssc.org.br) [CONFIRMADO]

## 4. Metas mínimas mensais do CEO

Fonte: Portaria GM/MS 1.464/2011, https://bvsms.saude.gov.br/bvs/saudelegis/gm/2011/prt1464_24_06_2011.html, hoje no Anexo XL da PC 6/2017. A NT 32/2026 confirma que as metas continuam valendo. [CONFIRMADO]

| Grupo | CEO I | CEO II | CEO III |
|---|---|---|---|
| Procedimentos básicos | 80 | 110 | 190 |
| Periodontia | 60 | 90 | 150 |
| Endodontia | 35 | 60 | 95 |
| Cirurgia oral | 80 | 90 | 170 |

- Básicos: 0101020058/66/74/82/90, 0307010015/23/31/40, 0307020070, 0307030016/24, 0414020120/138. **Contam só para paciente com necessidades especiais.** Pelo menos 50% devem ser restaurações (0307010023/31/40).
- Periodontia: 0307030032, 0414020081/154/162/375.
- Endodontia: 0307020037/45/53/61/88/96, 0307020100/118. Pelo menos 20% devem ser 0307020053 ou 0307020096 (dentes com 3+ raízes).
- Cirurgia: lista longa (0201010232, 0404020xxx, 0414010xxx, 0414020xxx…), na portaria.
- **Consequência:** suspensão total do incentivo de custeio se a meta de qualquer especialidade não for atingida por 2 meses consecutivos ou 3 alternados em 12 meses. A suspensão dura até a produção voltar ao mínimo. [CONFIRMADO]
- A portaria também diz que procedimento de CBO 2232 feito em CEO habilitado (regra contratual 7107) não gera crédito no SIA. [CONFIRMADO; com o SIAPS, INFERÊNCIA de que isso perde efeito]
- Não achei portaria posterior que mude esses números. [CONFIRMADO por ausência na busca; ver perguntas em aberto]

## 5. Indicadores da eSB (Portaria GM/MS 3.493/2024, Anexo V; notas metodológicas de 2026)

Fontes: notas metodológicas B1 e B2 (SEI 25000.053348/2026-78, maio/2026), https://www.gov.br/saude/pt-br/composicao/saps/publicacoes/fichas-tecnicas/equipe-de-saude-bucal/nota-metodologica-b1-primeira-consulta-programada e .../nota-metodologica-b2-tratamento-concluido [CONFIRMADO]. Resumo das mudanças de 2026 (B1–B6, C3): COSEMS-SP, https://www.cosemssp.org.br/wp-content/uploads/2026/06/notas-metodologicas-indicadores-APS.pdf [SECUNDÁRIO]. As alterações da 3.493 vieram pelas portarias 7.799/2025, 9.591/2025 (eSB 20h/30h) e 10.994/2026. [SECUNDÁRIO]

| Ind. | Nome | De onde sai na FAO | Marca |
|---|---|---|---|
| B1 | Primeira consulta programada (cobertura) | `tiposConsultaOdonto = 1` (SIGTAP 03.01.01.015-3). Conta 1 vez por pessoa e por dentista a cada 12 meses. Denominador: vinculados à eSF/eAP (NT 30/2025). CBOs 223208, 223293, 223272. CPF ou CNS válido no CADSUS | CONFIRMADO |
| B2 | Tratamento concluído | `tiposEncamOdonto` contém 15 ("Tratamento concluído") ÷ pessoas com primeira consulta (B1) | CONFIRMADO |
| B3 | Taxa de exodontia | exodontias ÷ total de procedimentos (`procedimentosRealizados`). Desde 2026, TSB 322405/322425 entram no denominador | SECUNDÁRIO |
| B4 | Escovação supervisionada | não sai da FAO. Sai da **Ficha de Atividade Coletiva** (MIAC), prática de escovação dental supervisionada | SECUNDÁRIO / INFERÊNCIA do modelo |
| B5 | Procedimentos preventivos | preventivos ÷ total de `procedimentosRealizados`. Em 2026 saíram selamento provisório e amálgamas e entraram 01.01.02.012-0 e 03.07.03.004-0. Faixa "ótimo" a partir de 65% | SECUNDÁRIO |
| B6 | TRA | TRA ÷ total restaurador (amálgamas fora do denominador) | SECUNDÁRIO |
| C3 | Gestação e puerpério (saúde da mulher) | inclui atendimento odontológico da gestante: CBO 2232 e (desde 2026) 3224. Na FAO, sai de `gestante = true` + profissional de saúde bucal | SECUNDÁRIO / INFERÊNCIA no campo |

O indicador de "gestante com atendimento odontológico" não é um B. Está dentro do C3. [SECUNDÁRIO]

Não li as notas B3–B6 nem o texto da 3.493 consolidada. As listas exatas de SIGTAP de numerador e denominador estão nessas notas (https://www.gov.br/saude/pt-br/composicao/saps/publicacoes/fichas-tecnicas/equipe-de-saude-bucal/).

## Bônus — CFO: atestado e receita eletrônica

- **Não achei a "Resolução CFO 295/2026"**: nenhum resultado com esse número. Em 2026 aparecem as Resoluções CFO 283, 284, 285 e 286 (19–20/03/2026). A 283 revoga a Res. CFO 100/2010 e artigos 43–48 da Res. CFO 63/2005; não confirmei se trata de documento eletrônico. Fonte: https://abmes.org.br/legislacoes/detalhe/5508/resolucao-cfo-n-283 [SECUNDÁRIO]
- O CFO mantém o portal de prescrição eletrônica (https://prescricao.cfo.org.br) com assinatura ICP-Brasil. Pelo guia de 2020, a assinatura digital não é obrigatória para exercer, mas é necessária para validar o documento digital: https://website.cfo.org.br/wp-content/uploads/2020/11/Guia-de-Esclarecimento-Assinatura-Digital.pdf [CONFIRMADO no guia; vigência atual não verificada]
- Para controlados vale a regra sanitária da ANVISA (RDC de prescrição eletrônica de controlados, em vigor em 2026 segundo fonte secundária). Isso já é tratado no módulo 19d. [SECUNDÁRIO]

## Perguntas em aberto

1. **Faturamento MAC do CEO:** a NT 32 só diz que o BPA deixa de valer para monitoramento e financiamento **federal** de serviços habilitados. O pagamento por produção MAC (PPI/FPO, teto MAC) de procedimentos do CEO ainda exige BPA no SIA? A nota não responde. Isso define se o módulo 28 precisa gerar BPA para o CEO (reuso do módulo 22) ou só LEDI.
2. Ler o texto da **Portaria GM/MS 10.192/2026** no DOU, para ver se ela muda metas, faixas do LRPD ou a regra de suspensão.
3. **Quadro 3 da NT 32** (imagem): confirmar a lista oficial de SIGTAP de prótese monitorados.
4. Notas metodológicas B3–B6 e C3: listas exatas de SIGTAP e CBO por indicador.
5. Se o odontograma (dente/face) não vai ao SIAPS, o Rota Saúde guarda dente/face só no prontuário próprio e agrega para `quantidade` na exportação. Falta confirmar se o PEC soma vários dentes do mesmo procedimento numa linha só (código + quantidade = nº de dentes).
6. CBO 322410 (protético) não está na Tabela 4: o protético do LRPD não registra FAO. Isso bate com a regra de registrar na entrega, pelo dentista.
7. Resolução CFO sobre atestado e receita eletrônica: confirmar número e texto no site do CFO (sistemas.cfo.org.br).
