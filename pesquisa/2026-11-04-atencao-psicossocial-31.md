# Pesquisa — Atenção psicossocial (CAPS) e RAAS (módulo 31)

**Data:** 2026-11-04 · Fontes consultadas nessa data.

## Instrumento de registro — Portaria SAS/MS nº 854, de 22/08/2012

Fontes:
- https://www.conass.org.br/?p=1153 (íntegra da portaria)
- https://prefeitura.sp.gov.br/documents/d/saude/nt_caps_instrucao_e_alinhamento_de_registro_caps_final-pdf (SMS-SP, NT 01/2019, tabela de códigos e instrumentos)

A portaria altera os procedimentos dos CAPS no SIGTAP a partir da competência 10/2012, com base nas modalidades da Portaria GM 336/2002, no CAPS AD III da Portaria GM 130/2012 e na Portaria 3.089/2011. Segundo a NT da SMS-SP:

| Código | Procedimento | Instrumento |
|---|---|---|
| 03.01.08.020-8 | Atendimento individual de paciente em CAPS | RAAS – Atenção Psicossocial |
| 03.01.08.021-6 | Atendimento em grupo de paciente em CAPS | RAAS-PSI |
| 03.01.08.022-4 | Atendimento familiar em CAPS | RAAS-PSI |
| 03.01.08.024-0 | Atendimento domiciliar para paciente de CAPS e/ou familiares | RAAS-PSI |
| 03.01.08.028-3 | Práticas expressivas e comunicativas em CAPS | RAAS-PSI |
| 03.01.08.029-1 | Atenção às situações de crise | RAAS-PSI |
| 03.01.08.034-8 | Ações de reabilitação psicossocial | RAAS-PSI |
| 03.01.08.025-9 | Ações de articulação de redes intra e intersetoriais | BPA-C |
| 03.01.08.026-7 | Fortalecimento do protagonismo de usuários e familiares | BPA-C |
| 03.01.08.030-5 | Matriciamento de equipes da atenção básica | BPA-C |
| 03.01.08.031-3 | Ações de redução de danos | BPA-C |
| 03.01.08.032-1 | Acompanhamento de serviço residencial terapêutico | BPA-C |
| 03.01.08.039-9 | Matriciamento de equipes da urgência e dos hospitais de referência | BPA-C |

- Não verificados: os códigos de acolhimento inicial, de acolhimento diurno e de acolhimento noturno, e se eles vão em RAAS ou em BPA-I. **Conferir no SIGTAP oficial**, pelo atributo "instrumento de registro" (09 = RAAS Atenção Psicossocial).

## RAAS — Portaria SAS 276/2012 e Manual Operacional RAAS v1.5

Fontes:
- https://www.conass.org.br/?p=554
- https://info.saude.df.gov.br/wp-content/uploads/2023/11/Manual-Operacional-RAAS-V.1.5.pdf (aplicativo RAAS 01.40)

**Remessa:**
- A remessa do aplicativo RAAS é importada no SIA da mesma competência de apresentação.
- A portaria diz que o sistema, o manual, os formulários e o **layout da base de dados** ficam em sia.datasus.gov.br. O layout não foi obtido nesta pesquisa.
- O aplicativo também importa remessas de outras instalações para centralizar a produção antes de exportar ao gestor.

**Atendimento:**
- Um registro por usuário, CNES e mês, com data de início.
- O início e o mês definem se o atendimento é inicial ou de continuidade. A continuidade repete a identificação e os dados do atendimento, com a tela de ações vazia.

**Identificação (todos os campos obrigatórios):**
- CNS e dados do usuário.
- Raça/cor, com etnia só se a raça for indígena.
- Celular ou telefone de contato.
- Situação de rua, informada **a cada mês**.

**Dados do atendimento:**
- Uso de álcool e drogas (obrigatório), com quais substâncias.
- Origem do usuário, só no inicial.
- CID principal (obrigatório) e CIDs secundários.
- Cobertura de ESF, com o CNES da USF quando houver.
- Destino do usuário. "00 – Permanência" continua no mês seguinte. Qualquer outro destino exige a data de conclusão e encerra o atendimento.

**Ações:**
- Só procedimentos com instrumento 09 no SIGTAP.
- Data da ação dentro do mês.
- Quantidade dentro do limite do SIGTAP.
- Serviço fixo 115 (Atenção Psicossocial), com a classificação cadastrada no CNES.
- CNS e CBO do profissional, criticados no SIA contra o CNES.
- Local de realização: CAPS ou território.
- Número de autorização (faixa de APAC) quando o atributo 036 exige.
- Tempo máximo de permanência por procedimento, no SIGTAP.

**Calendário:** a competência de apresentação segue o processamento do SIA do gestor, e o cronograma é definido pelo gestor.

## Modalidades e acolhimento noturno

Fontes:
- https://bvsms.saude.gov.br/bvs/saudelegis/gm/2012/prt0130_26_01_2012.html (Portaria GM 130/2012)
- https://www.prefeitura.sp.gov.br/cidade/secretarias/upload/saude/Protocolo%20Saude%20Mental%20Centro%20de%20Aten%C3%A7%C3%A3o%20Psicossocial%20(CAPS).pdf

Modalidades:
- **Portaria 336/2002:** CAPS I, II, III, i e AD.
- **Portaria 130/2012:** CAPS AD III, 24 h, com acolhimento noturno e até 14 dias de permanência em 30 dias. Mais que isso exige reavaliação.
- **CAPS III:** fontes secundárias citam no máximo 7 dias corridos ou 10 intercalados em 30 dias. **Não verificado na norma.**

## Edital e mercado

- Atenção psicossocial com RAAS aparece em 14 de 15 editais (`pesquisa/2026-10-05-integracoes/frente-6-mercado.md`).
- O PTS é construído pela equipe de referência, com o usuário e a família, desde o primeiro atendimento. Fonte: https://moodle.unasus.gov.br/vitrine29/mod/resource/view.php?id=5314
- A Prefeitura do Rio lançou o Prontuário Carioca de Saúde Mental em 2026: https://saude.prefeitura.rio/wp-content/uploads/sites/47/2026/01/Manual-PCSM-2026.pdf

## LGPD

- Saúde mental é dado sensível (LGPD art. 11), e o estigma agrava o dano de um vazamento.
- Decisões do usuário (2026-11-04):
  - fora do CAPS, só o vínculo "em acompanhamento no CAPS X" e o técnico de referência;
  - nada do CAPS no wpda.
