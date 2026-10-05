# Mapa de integrações — Ciclo 2 (atenção primária)

- **Data:** 2026-10-05
- **Origem:** brainstorm do Ciclo 2 (módulos 15–26) e pesquisa em seis frentes
- **Status:** levantamento; nenhuma decisão de arquitetura tomada aqui
- **Anexos:** os seis relatórios completos, com URL e marcação
  `[CONFIRMADO]`/`[INFERÊNCIA]` em cada afirmação, estão em
  [`2026-10-05-integracoes/`](2026-10-05-integracoes/).

> Este documento é um resumo. Toda afirmação dele vem de um anexo; em caso de
> dúvida, vale o anexo e a fonte oficial citada lá. Normas e versões mudam
> rápido neste domínio (a lista de notificação mudou três vezes em 14 meses; o
> LEDI publica versão a cada 2–6 semanas).

## 1. Contexto

O Rota Saúde fechou o MVP (14 módulos) como plataforma de triagem do cidadão e
coordenação do acesso ao atendimento. O Ciclo 2 quer torná-lo plataforma da
atenção primária, com **modo de prontuário por cidade**:

- **Registro:** o Rota Saúde é o prontuário e substitui o e-SUS PEC.
- **Integrado:** a consulta é feita no PEC; o Rota Saúde faz triagem,
  recepção, fila, agenda, linhas de cuidado.

Ficaram como **linhas de produto separadas**: hospitalar (prontuário
hospitalar e de maternidade, gestão hospitalar) e vigilância sanitária.

Roteiro proposto: 15 triagem direcionada por perfil · 16 modo de prontuário +
exportação · 17 agenda · 18 acolhimento · 19 consulta · 20 linha do tempo ·
21 exames · 22 procedimentos · 23 linhas de cuidado · 24 domiciliar/coletivo ·
25 vigilância epidemiológica · 26 regulação.

## 2. O que a pesquisa mudou

1. **Nenhum caminho evita o PEC.** O sistema de terceiro entrega as fichas
   LEDI a uma instalação do PEC do município (API HTTPS desde o PEC 5.3.19, ou
   arquivo Thrift/XML), e o PEC envia ao SIAPS (o antigo SISAB). Mesmo no modo
   registro, a cidade mantém um PEC atualizado como porta de entrada. (F1)
2. **O exportador LEDI é manutenção contínua.** Versão vigente 8.7.0
   (13/08/2026, exige PEC ≥ 5.5.26); dado enviado por versão com mais de 12
   meses é invalidado desde 01/01/2026. (F1)
3. **No modo registro, o repasse federal da cidade depende do Rota Saúde.**
   Os componentes de Vínculo e Qualidade (indicadores C1–C7 por equipe) usam
   Cadastro Individual/Domiciliar, Atendimento Individual com CIAP-2/CID-10,
   PA e peso/altura estruturados, tipo de demanda, procedimentos SIGTAP, visita
   do ACS, vacina via RNDS e odontologia. Três competências seguidas sem envio
   suspendem o componente fixo. (F1)
4. **A RNDS já tem obrigações vigentes**, todas por credenciamento por
   estabelecimento (CNES + gov.br do gestor + certificado ICP-Brasil, em duas
   fases: homologação e produção):
   - vacina aplicada → modelo RIA em até 24 h (Portaria 5.663/2024); o Thrift
     deixou de ser aceito em set/2025; (F2, F5)
   - solicitação/encaminhamento à atenção especializada → modelo MIRA, envio
     diário (Portaria 6.656/2025); (F2, F4)
   - resultado de exame → modelo REL (Portaria 8.276/2025), mas a
     especificação publicada só cobre SARS-CoV-2 e Mpox; (F3)
   - produção especializada → RAC/CMD substituindo SIA/SIH, sem data-limite
     (Portarias 7.495/2025 e 11.952/2026). (F4)
5. **Ler dados de fora é quase impossível.** O PEC não tem API de leitura
   clínica (só o DW PEC em SQL e a tela de "sistemas externos" aberta dentro
   do atendimento do PEC). A RNDS só deixa ler num contexto de atendimento,
   com o profissional identificado; não há API para puxar o histórico. (F1, F2)
6. **Prontuário sem papel exige NGS2 com ICP-Brasil** (CFM 1.821/2007). A
   certificação SBIS é voluntária; o requisito é o NGS2. Atestado e receita de
   controlado exigem ICP-Brasil (Lei 14.063/2020; CFM 2.299/2021); receita de
   controlado exige integração com o SNCR da Anvisa (RDC 1.000/2025). Guarda
   mínima de 20 anos (Lei 13.787/2018). (F2)
7. **gov.br só para órgão público em domínio .gov.br** (Portaria SGD/MGI
   7.076/2024). O Rota Saúde, como empresa e no domínio atual, não pode ter
   credencial própria; cada prefeitura precisa ser a titular. **Isso afeta o
   que já existe** (login gov.br do Plano 3B e gates de go-live do Plano 6).
   (F2)
8. **Exames não têm padrão nacional de pedido.** O pedido é guia; o resultado
   volta como PDF ou papel, em quatro arranjos (laboratório municipal,
   credenciado por chamamento, consórcio, LACEN/GAL). (F3)
9. **Notificação de agravo não tem API.** A vigilância municipal digita no
   Sinan Net/Online, e-SUS Sinan, e-SUS Notifica, SIVEP-Gripe. A RNDS não tem
   modelo de notificação. (F5)
10. **Faturamento ambulatorial ainda é arquivo.** Um terceiro pode gerar o
    arquivo texto do BPA (layout DATASUS) para o BPA Magnético; SISREG e e-SUS
    APS já fazem isso. SIGTAP é tabela mensal. SISREG III não aceita novas
    adesões; o e-SUS Regulação (gratuito, só ambulatorial) o substitui; nenhum
    dos dois tem API de escrita. (F4)
11. **O mercado compra suíte, e o e-SUS cresce.** Pregão de menor preço global
    com prova de conceito eliminatória (Mafra/SC: 438 itens obrigatórios);
    R$ 12–16 mil/mês em cidade pequena. O PEC passou de 21% para 60% das UBS
    com prontuário (2017→2022). O Ministério também lançou agendamento pelo
    Meu SUS Digital, e-SUS AF (farmácia) e AGHUse (hospitalar). (F6)
12. **O diferencial existe, mas não pontua em edital.** Nenhum concorrente faz
    triagem do cidadão por protocolo escrito e assinado pela cidade. (F6)

## 3. Matriz: sistema externo × módulo

Legenda: **Obr.** = obrigatório para o município quando o Rota Saúde faz
aquela função; **Opc.** = opcional; **—** = não se aplica.

| Sistema | Quem opera | Padrão | Obrigatoriedade | Módulos | Leitura possível? |
|---|---|---|---|---|---|
| e-SUS APS PEC + LEDI → SIAPS | MS/SAPS | LEDI (API HTTPS, Thrift, XML) | Obr. no modo registro | 16, 18, 19, 22, 24 | Não (só DW PEC, SQL) |
| RNDS — RIA (vacina) | MS/DATASUS | FHIR R4 | Obr. se registrar dose | 12, 22 | Só em contexto de atendimento |
| RNDS — MIRA (regulação) | MS/DATASUS | FHIR R4 | Obr. se regular | 26 | idem |
| RNDS — REL/RELC (exame) | MS/DATASUS | FHIR R4, LOINC | Obr. por norma; especificação parcial | 21 | Não documentado |
| RNDS — RAC/CMD | MS/DATASUS | FHIR R4 | Sem data-limite | 19, 26 | idem |
| CADSUS | MS/DATASUS | SOAP PDQv3/PIXv3 ou FHIR via RNDS | Opc. (validação) | 15, 16 | Sim, com autorização |
| CNES | MS/DATASUS | SOAP público, base mensal, API aberta | Opc. (cadastro) | 09, 10, 16 | Sim (INE só na base) |
| gov.br (login/assinatura) | MGI | OIDC | Opc.; só órgão público | 06, 19 | — |
| ICP-Brasil | ITI | certificado A1/A3 | Obr. p/ prontuário sem papel, atestado, receita | 19 | — |
| SNCR Anvisa | Anvisa | — | Obr. p/ receita de controlado | 19 | — |
| SIGTAP | MS/DATASUS | tabela mensal (ZIP) | Obr. p/ procedimento e faturamento | 21, 22, 26 | Sim (download) |
| SIA — BPA/APAC | MS/DATASUS | arquivo texto | Obr. p/ faturar MAC | 21, 22, 26 | Retorno a conferir |
| SISREG III / e-SUS Regulação / estaduais (CARE-PR, CROSS) | MS ou estado | web; SISREG com API só de leitura | Convivência | 26 | Parcial |
| Sinan / e-SUS Sinan / e-SUS Notifica | MS/SVSA | web (digitação) | Notificar é obrigação; envio é digitado | 25 | Não |
| GAL | LACEN | web | — | 21, 25 | Não |
| LIS de laboratório | laboratório | HL7 v2, FHIR, proprietário | Opc. | 21 | Por contrato |

## 4. Implicações por módulo

| # | Módulo | O que a pesquisa diz | Pode começar? |
|---|---|---|---|
| 15 | Triagem direcionada por perfil | Sem integração externa. Perfil do cidadão deve nascer compatível com CADSUS e com o Cadastro Individual do LEDI (nascimento, sexo, CPF primário). | **Sim** |
| 16 | Modo de prontuário + exportação | Exportador LEDI para o PEC da cidade; manutenção a cada versão; CNS/CPF do profissional, INE, tipo de equipe (70/76), CNES. Precisa decidir se cada cidade tem credencial própria. | Depois das decisões da §5 |
| 17 | Agenda | Sem obrigação externa. Concorre com o agendamento do Meu SUS Digital (500+ municípios); não há API para um terceiro ler a agenda do PEC. | **Sim** |
| 18 | Acolhimento | Gera indicador C1 (acesso); risco de dupla contagem no modo integrado. | Depois do 16 |
| 19 | Consulta | NGS2 + ICP-Brasil; atestado/receita com ICP-Brasil; controlado via SNCR. Maior pacote regulatório do roteiro. | Depois do 16 |
| 20 | Linha do tempo | No modo integrado fica incompleta (sem leitura do PEC). A tela de "sistemas externos" do PEC é ponte possível. | Depois do 19 |
| 21 | Exames | v1 sem integração (pedido SIGTAP, upload de laudo, prazos, ciência, portal do prestador); v2 FHIR/HL7 por laboratório; v3 RNDS. | v1 sim, depois do 19 |
| 22 | Procedimentos | SIGTAP mensal com regras (CBO, idade, sexo, CID, instrumento); exporta BPA. | Depois do 16 |
| 23 | Linhas de cuidado | Indicadores C2–C7 definem o que acompanhar. | Depois do 19/21 |
| 24 | Domiciliar/coletivo | Fichas LEDI de visita e atividade coletiva; app offline aparece em 11 de 15 editais. | Depois do 16 |
| 25 | Vigilância epidemiológica | v1 = ficha interna, catálogo versionado, alerta 24 h, painel por bairro; sem API oficial. | v1 sim, depois do 19 |
| 26 | Regulação | MIRA diário à RNDS se a cidade regular no Rota Saúde; convivência com SISREG/e-SUS Regulação/estadual. | Por último |

## 5. Decisões que a pesquisa abriu

1. **Suíte completa ou complemento?** Concorrer em edital exige cobrir as
   lacunas da §6; vender como camada sobre o PEC ou ERP da prefeitura exige
   outro caminho jurídico (CPSI do Marco Legal das Startups, inexigibilidade
   ou contrato complementar). Decide a ordem do portfólio inteiro.
   **Decidido em 2026-10-05: suíte completa.** O roteiro passa a incluir as
   lacunas da §6, ordenadas pelo que a prova de conceito dos editais elimina.
2. **Credencial por cidade ou da plataforma?** RNDS, LEDI e gov.br parecem
   exigir titularidade da secretaria (CNES, e-CNPJ, domínio .gov.br). Encaixa
   no banco por cidade (ADR 0020), mas o fluxo de onboarding muda.
3. **gov.br no produto atual.** Reavaliar o login gov.br do Plano 3B e os
   gates do Plano 6 à luz da Portaria SGD/MGI 7.076/2024.
4. **O Rota Saúde registra dose de vacina?** Se sim, RNDS-RIA vira requisito
   (24 h, guarda de retorno, adequação em 15 dias).
5. **Assinatura.** Serviço misto ICP-Brasil + gov.br; assinatura em lote ou
   certificado em nuvem para viabilizar a rotina da APS.

## 6. Lacunas do roteiro reveladas pelos editais (F6)

Presentes nos 15 editais analisados e fora do roteiro: **farmácia e
dispensação, odontologia, laboratório, faturamento BPA, vacinas (RNDS),
transporte/TFD**. Logo depois: almoxarifado e Hórus/BNAFAR (13), CAPS/RAAS
(14), app offline para tablet (11), urgência/UPA (10), zoonoses/endemias (8),
telessaúde e ouvidoria (7). Certificação SBIS é exigida ou pontuada em 6.

## 7. Perguntas em aberto, por destinatário

**Ministério da Saúde / DATASUS** (Fala.BR, Webatendimento SAPS, Portal de
Serviços):
- Há envio direto ao SIAPS sem passar pelo PEC? Registros LEDI aparecem no
  histórico clínico do PEC ou só na produção?
- Uma empresa pode operar RNDS/CADSUS com o e-CNPJ da secretaria? Uma
  solicitação cobre todos os CNES do município? Prazo real de homologação?
- Haverá API de leitura da RNDS para sistemas municipais (REL, RAC)?
- Especificação e prazo do REL amplo, do RAC para a APS, do CMD e do plano
  operativo do MIRA; previsão de desligar o BPA.
- Previsão de interface para terceiros no e-SUS Sinan e de API de escrita no
  e-SUS Regulação; o e-SUS AF aceita gravação de terceiros?

**Cidades-piloto (Curitiba, Maringá e próximas):**
- Usam PEC, sistema próprio ou de terceiro? Instalação centralizadora?
- Arranjo de cada tipo de exame e qual LIS cada laboratório usa.
- Quem regula a MAC (SISREG, CARE Paraná, consórcio, planilha)? Já enviam
  MIRA?
- Quem digita no Sinan; quem notifica direto no e-SUS Sinan.
- Política de liberação de resultado de exame ao cidadão; prazos-alvo por
  etapa.
- Controle de cota só em quantidade ou também em valor.

**Jurídico / regulatório:**
- Caminho de contratação para produto que não é suíte.
- Titularidade gov.br para SaaS (modelo de consórcio/estado como gestor?).
- COFEN e demais conselhos sobre assinatura eletrônica.
- Vigência do convênio CFM–SBIS, manual e custo de certificação.
- Consentimento da RNDS (opt-out?) frente ao ADR 0026.

## 8. Próximo passo sugerido

1. Seguir com os módulos sem dependência externa: **15** (triagem por perfil)
   e **17** (agenda).
2. Decidir a §5.1 (suíte × complemento) antes de especificar o 16.
3. Enviar as perguntas da §7 ao Ministério e às cidades-piloto.
