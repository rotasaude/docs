# Módulo 31 — Atenção psicossocial (CAPS) — design

**Data:** 2026-11-04
**Status:** aprovado (2026-11-04)
**Afeta:**
- `apps/api`:
  - **cidade:** CAPS como tipo de unidade, equipe do CAPS, matrícula (atendimento RAAS), PTS, ações individuais, coletivas e no território, matriciamento, acolhimento noturno (leitos e permanências), RAAS mensal com crítica, remessa e ficha espelho, BPA-C das ações para a rede (módulo 22), fonte interna `psychosocial` da linha do tempo, vínculo visível à UBS, painel;
  - **plataforma:** interruptor `psychosocial_care`.
- `apps/dashboard`: CAPS (matrícula, PTS, ações, grupos e oficinas, leitos, matriciamento, fechamento da RAAS do mês), vínculo na tela da UBS, painel.
- `apps/maintenance`: interruptor, pelo mecanismo genérico.
- `apps/wpda`: **nada**. O CAPS não aparece para o cidadão (decisão 4).

**ADR:** `docs/adr/0049.md` · **Módulo:** `docs/modulos/31--caps.md` · **Pesquisa:** `docs/pesquisa/2026-11-04-atencao-psicossocial-31.md`
- **ADRs relacionados:** 0020 (banco por cidade), 0026 (exclusão LGPD), 0028 (LEDI), 0029 (agenda), 0031 (leitura do prontuário), 0034 (receita controlada), 0035 (linha do tempo).
- **Módulos relacionados:** 16 (prontuário e LEDI), 17 (agenda), 19 (consulta e documentos), 22 (BPA).

**Fora desta entrega:**
- residência terapêutica como serviço próprio (só o BPA-C do acompanhamento pelo CAPS entra);
- unidade de acolhimento (UA e UAI);
- comunidade terapêutica e internação psiquiátrica em hospital;
- RAAS da atenção domiciliar;
- prontuário do cidadão no wpda para o CAPS;
- integração com o aplicativo RAAS além da remessa e da ficha.

## 1. Intenção

A equipe do CAPS acompanha cada usuário num prontuário próprio, desde o acolhimento até a alta. O PTS fica vivo, os grupos e oficinas registram a presença de todos de uma vez, e no fim do mês a RAAS sai pronta, sem digitação no aplicativo do DATASUS. A UBS sabe que o usuário está no CAPS e com quem falar, mas não lê o que é do CAPS.

**Por quê:**
- A atenção psicossocial com RAAS está em 14 dos 15 editais.
- A RAAS é mensal, repete a identificação e as ações, e hoje é redigitada.
- O CAPS III e o AD III funcionam 24 horas, com leito.

**Sucesso:**
- Todo usuário em acompanhamento tem matrícula e PTS com revisão em dia.
- A RAAS do mês sai sem inconsistência no SIA.
- Nenhuma ação vai à RAAS com código, CBO ou data fora da regra.
- Nenhum dado do CAPS aparece fora da equipe do CAPS, exceto o vínculo e o matriciamento.

**Antiobjetivos:**
- CID ou evolução do CAPS visível na UBS, no wpda, no SMS ou no PEC.
- Leito com dois ocupantes.
- Permanência noturna acima do limite sem reavaliação registrada.
- Ação digitada duas vezes, no sistema e no aplicativo RAAS.

## 2. Decisões

| # | Decisão |
|---|---|
| 1 | **Escopo:** um módulo, com o CAPS ambulatorial completo e o **acolhimento noturno** do CAPS III e do CAPS AD III (escolha do usuário, opção B). |
| 2 | **Saída da RAAS:** remessa no layout oficial do SIA, com a homologação do layout como gate, e **ficha espelho** por usuário e mês para digitação no aplicativo RAAS enquanto isso (escolha do usuário, opção A). |
| 3 | **Prontuário fechado:** só a equipe do CAPS lê. A UBS e os outros profissionais veem apenas o vínculo genérico, "em acompanhamento no CAPS desde …", sem o nome nem a modalidade da unidade (escolha do usuário em 2026-11-04, porque o nome de um CAPS AD revela uso de substância), com o técnico de referência e o telefone de contato, além do que a equipe do CAPS registrar como **matriciamento compartilhado** com aquela equipe. Não veem CID, evolução nem PTS (escolha do usuário, opção A). |
| 4 | **Cidadão:** nada do CAPS no wpda. Isso vale para consultas, avisos, linha do tempo e documentos. Também não há SMS de agendamento no CAPS (escolha do usuário, opção B). |
| 5 | **CAPS:** `health_units.kind` ganha `caps`, com modalidade (`caps_i`, `caps_ii`, `caps_iii`, `caps_i_child`, `caps_ad`, `caps_ad_iii`, `caps_ad_iv`) e classificação do serviço 115 conforme o CNES. Só `caps_iii`, `caps_ad_iii` e `caps_ad_iv` têm leitos. |
| 6 | **Equipe:** papel `psychosocial_staff` (privilegiado, com segundo fator). A lotação vem do vínculo CNES que já existe (`professional_links` com `health_unit_id` do CAPS), que também dá o CBO da crítica. `psychosocial_lead` (coordenação) fecha a RAAS do mês e corrige ações. A leitura vale só para os usuários dos CAPS em que a pessoa está lotada. A recepção do CAPS é `psychosocial_staff`. |
| 7 | **Matrícula** (o atendimento RAAS): começa pelo **acolhimento inicial**, por demanda espontânea ou encaminhamento. Registra: data de início; origem do usuário (lista da RAAS); CID principal e secundários; uso de álcool ou drogas e quais substâncias; cobertura de ESF com o CNES da USF (preenchido do território do 11 quando houver); técnico de referência. Fecha com o **destino do usuário** e a data de conclusão. Um paciente tem no máximo uma matrícula aberta por CAPS. |
| 8 | **Acolhimento sem matrícula:** quem é acolhido e não fica no CAPS (encaminhado à UBS ou a outro serviço) gera só o acolhimento, com o instrumento que o SIGTAP indicar, sem abrir matrícula. |
| 9 | **PTS:** problemas e necessidades, objetivos, ações e responsáveis, frequência proposta (intensivo, semi-intensivo, não intensivo) e data de revisão. Tem versões imutáveis; revisar cria uma versão nova. A revisão vencida avisa o técnico de referência e a coordenação. |
| 10 | **Ação:** código SIGTAP com instrumento RAAS-PSI ou BPA-C, data, hora, profissional (CNS e CBO), local (`caps` ou `territory`), quantidade, evolução em texto cifrado e participantes. **Ação coletiva** (grupo, oficina, assembleia) tem vários participantes, e cada um recebe a ação na própria RAAS. **Atendimento familiar** pode ter familiar sem cadastro, só com o nome. Os códigos permitidos vêm do SIGTAP pelo instrumento, nunca de lista fixa no código. |
| 11 | **Crítica da ação, na hora** (as regras de quantidade máxima, tempo de permanência e atributo 036 exigem ampliar o importador do SIGTAP da plataforma, que entra neste módulo): instrumento compatível; CBO compatível; quantidade dentro do máximo do SIGTAP; data não futura e dentro do mês aberto; matrícula aberta para ação RAAS; número de autorização quando o atributo 036 exigir. A ação é imutável. Corrigir é cancelar com motivo e registrar de novo, e depois do fechamento do mês só a coordenação corrige. |
| 12 | **RAAS do mês:** `psychosocial_monthly_records`, um por matrícula e mês de atendimento. Guarda a foto da identificação, a situação de rua do mês (perguntada a cada mês), CIDs, substâncias, ESF, destino e as ações do mês. A continuidade nasce sozinha no 1º dia do mês para toda matrícula em "permanência", com a tela de ações vazia. |
| 13 | **Fechamento e competência:** a coordenação fecha o mês do CAPS. A crítica completa lista as pendências: CNS do usuário ausente, telefone ausente, situação de rua não informada, CBO ou CNS do profissional sem vínculo no CNES local do 16, ação fora da regra. Com o mês fechado, sai a remessa da competência de apresentação, que é a competência aberta do calendário de faturamento do 22 (`billing_periods`), sem calendário próprio. Baixam a remessa e as fichas `psychosocial_lead` e `billing_clerk`, com trilha. Reabrir exige motivo e trilha. |
| 14 | **Remessa e ficha espelho:** a remessa segue o layout oficial do SIA, com gerador isolado, como o `BpaFile` do 22. A ficha espelho é um PDF por usuário e mês, na ordem das telas do aplicativo RAAS. Ela sai mesmo antes da homologação do layout e não leva dados que o aplicativo não pede. |
| 15 | **BPA-C:** as ações com instrumento BPA-C vão ao módulo 22 como `performed_procedures` com fonte nova `psychosocial`. Isso inclui o matriciamento da UBS e da urgência, a articulação de rede, a redução de danos, o protagonismo e a residência terapêutica. Como o BPA-C é consolidado, a linha da ação de equipe não tem paciente. `performed_procedures.patient_id` passa a aceitar nulo só quando `source = psychosocial` e o instrumento é `bpa_c` (CHECK), e a crítica do 22 aceita linha sem paciente nesse caso. Sai **uma linha por ação**, com a quantidade da regra do SIGTAP, nunca uma por participante. Ações com instrumento BPA-I (acolhimento, se o SIGTAP indicar) levam o paciente. As ações RAAS não vão ao BPA. |
| 16 | **Matriciamento:** registro da equipe do CAPS com uma equipe de saúde (`health_teams`) de uma UBS (data, profissionais, tema). A discussão de um caso cita o paciente e tem um texto **compartilhado**. Lê esse texto quem tem vínculo CNES na unidade da equipe matriciada, com trilha. Se o paciente muda de território, a equipe nova não herda o texto e pede outro matriciamento. O **vínculo** (decisão 3) é visto por quem já pode abrir o paciente pela leitura contextual do ADR 0031. O resto do prontuário do CAPS continua fechado. |
| 17 | **Acolhimento noturno:** leitos do CAPS (número, ativo) e permanência por noite (entrada, saída prevista, saída, motivo). O banco recusa duas permanências que se cruzam no mesmo leito. O limite por cidade e modalidade é configurável: padrão de 14 noites em 30 dias no AD III (Portaria 130/2012) e de 7 corridas ou 10 intercaladas em 30 no CAPS III (a confirmar, gate). Passar do limite exige reavaliação registrada pela coordenação. A noite é registrada como ação com o código de acolhimento noturno do SIGTAP. |
| 18 | **Agenda própria, sem o 17:** o CAPS não usa a agenda do 17, porque o agendamento do 17 nasce de pedido do cidadão e alimenta o wpda e o SMS. Grupos e oficinas têm uma grade semanal (dia, hora, responsável, vagas). Os atendimentos individuais têm "próximo atendimento" (data e hora) registrado na matrícula ou na ação. A tela "Hoje no CAPS" junta os dois. Nada disso gera aviso ao cidadão. |
| 19 | **Receita:** psicotrópicos pela receita do 19 e do 19d, emitida a partir de uma ação de atendimento individual por profissional prescritor (CBO de médico). `clinical_documents` ganha `psychosocial_action_id` como pai alternativo de `attendance_id`, com CHECK de exatamente um pai. Não se cria consulta nem atendimento do 19. O documento vale na verificação pública como qualquer outro. Fica fora dos documentos do wpda, da linha do tempo geral e do LEDI. |
| 20 | **Encaminhamento:** a UBS encaminha ao CAPS por um pedido simples (motivo em texto cifrado, prioridade), que cai na fila de acolhimento do CAPS. O CAPS é porta aberta e também recebe demanda espontânea. |
| 21 | **Linha do tempo:** fonte `psychosocial`, mostrada só à equipe do CAPS. Fica fora da visão do cidadão, da visão profissional geral e da página do PEC (ADR 0035). Na visão geral, um marcador mostra só o vínculo da decisão 3. |
| 22 | **LEDI e PEC:** nada do CAPS vai ao LEDI (ADR 0028), porque o CAPS não é atenção básica. |
| 23 | **Painel:** matrículas ativas por CAPS e modalidade, novas e encerradas por destino, PTS vencidos, ações por tipo, ocupação dos leitos e situação da RAAS do mês. São só contagens, com "menos de 5". |
| 24 | **Interruptor:** `psychosocial_care`, que requer `feature:clinical_record`. Desligado, a resposta é 403 `feature_disabled`. |

As decisões 1 a 4 são do usuário. As demais são do coordenador, apresentadas no desenho.

**Decisões do coordenador (revisão da spec):** as decisões 6, 11, 13, 15, 16, 18 e 19 foram ajustadas depois da leitura do construtor.
- lotação pelo vínculo CNES;
- importador do SIGTAP ampliado;
- competência do calendário do 22;
- BPA-C sem paciente, uma linha por ação;
- leitores do matriciamento;
- agenda própria do CAPS, sem o 17;
- receita com pai `psychosocial_action_id`.

Essas mudanças tocam o 22 (`performed_procedures`), o 19 (`clinical_documents`) e o SIGTAP da plataforma. Isso é mudança transversal, e a suíte completa do api é obrigatória.

## 3. Dados

**Cidade:**
- `health_units` ganha `kind = caps`, `caps_modality` e `service_classification`. A lotação vem de `professional_links`.
- `psychosocial_groups`: grade semanal de grupos e oficinas.
- `psychosocial_referrals`: encaminhamento da UBS.
- `psychosocial_receptions`: acolhimento inicial, com ou sem matrícula.
- `psychosocial_enrollments`: matrícula, com CIDs, substâncias, origem, ESF, técnico de referência, destino e conclusão; os campos clínicos cifrados.
- `psychosocial_care_plans`: PTS, com versões imutáveis.
- `psychosocial_actions` e `psychosocial_action_participants`: imutáveis; cancelamento com motivo.
- `psychosocial_monthly_records`: RAAS do mês, com a foto e o estado `open`, `closed` ou `exported`.
- `psychosocial_month_closures`, por CAPS e mês, e `psychosocial_raas_exports`, a remessa gerada.
- `psychosocial_matrix_support` e `psychosocial_matrix_cases`: o caso tem o texto compartilhado.
- `psychosocial_beds` e `psychosocial_night_stays`, com exclusion constraint por leito.
- `performed_procedures.source` ganha `psychosocial`. `patient_id` aceita nulo só para `psychosocial` com `bpa_c`.
- `clinical_documents` ganha `psychosocial_action_id`, como pai alternativo.
- **Plataforma:** as tabelas do SIGTAP ganham quantidade máxima, tempo de permanência e atributos complementares (009, 035, 036), com o importador ampliado.
- `memberships.role` ganha `psychosocial_staff` e `psychosocial_lead`.
- Configuração por cidade: limites do acolhimento noturno e prazo de revisão do PTS (padrão de 90 dias).

## 4. Telas

- **CAPS · Acolhimento:** fila de encaminhamentos e demanda espontânea; acolhimento com ou sem matrícula.
- **CAPS · Usuário:** matrícula, PTS, ações, receitas, leito e encerramento, com a trilha da leitura.
- **CAPS · Hoje no CAPS:** grade dos grupos e próximos atendimentos do dia.
- **CAPS · Grupos e oficinas:** grade semanal e ação coletiva com a lista de presença.
- **CAPS · Leitos:** mapa dos leitos, entrada, saída, limite e reavaliação.
- **CAPS · Matriciamento:** registro com a UBS e casos compartilhados.
- **CAPS · RAAS do mês:** pendências da crítica, fechamento, remessa e fichas espelho.
- **UBS · Paciente:** cartão do vínculo com o CAPS e o texto compartilhado do matriciamento.
- **Painel:** CAPS.

## 5. Segurança e LGPD

- **Leitura:** todo dado clínico do CAPS (CIDs, substâncias, evolução, PTS, encaminhamento) é cifrado. A leitura nominal fica restrita à equipe lotada no CAPS e grava a trilha `clinical_record.viewed`. A UBS lê só o vínculo e o texto compartilhado, também com trilha.
- **Fora do sistema:** nada do CAPS vai a wpda, SMS, avisos, LEDI, URL, log ou evento. Os eventos levam só ids.
- **Imutabilidade:** trigger em `db/city_triggers.sql` para ações, participantes, versões do PTS, RAAS exportada e permanência fechada. Os novos `Platform.audit` entram em R18.
- **Exclusão LGPD (ADR 0026):** o prontuário do CAPS é registro de saúde e fica retido como o da atenção básica. O encaminhamento sem acolhimento e o acolhimento sem matrícula seguem a regra da triagem sem atendimento.
- **Painéis:** agregados, com "menos de 5".

## 6. Testes

**api:**
- equipe vê só o próprio CAPS;
- UBS vê só o vínculo e o caso compartilhado;
- trilha;
- matrícula única aberta;
- ação: instrumento, CBO, quantidade, data e matrícula;
- coletiva com participantes;
- continuidade no dia 1º;
- crítica e fechamento do mês;
- remessa e ficha espelho;
- BPA-C com fonte `psychosocial`;
- leito sem sobreposição;
- limite e reavaliação;
- CAPS sem aviso ao cidadão e receita do CAPS fora do wpda;
- BPA-C sem paciente, uma linha por ação;
- importador do SIGTAP com os campos novos;
- nada no LEDI nem na linha do tempo do cidadão;
- exclusão LGPD;
- interruptor.

**dashboard:** testes de componente; tela de presença do grupo.

**Suíte completa do api no fim.**

## 7. Entrega

- **Ordem:** api → dashboard → maintenance. O wpda só confere que nada aparece, sem mudança de tela.
- **Runbook:** `operacao/rollout-atencao-psicossocial.md`, com:
  - modalidade e classificação 115 no CNES;
  - equipe e CBOs;
  - faixa de APAC do gestor, se houver ação com autorização;
  - layout da remessa;
  - leitos do CNES.

## 8. F-IDs (propostos)

| ID | Funcionalidade | Superfície |
|---|---|---|
| F-31.1 | Interruptor, CAPS, modalidade e equipe | api, dashboard, maintenance |
| F-31.2 | Encaminhamento, acolhimento e matrícula | api, dashboard |
| F-31.3 | PTS com versões e revisão | api, dashboard |
| F-31.4 | Ações individuais, coletivas e no território | api, dashboard |
| F-31.5 | Acolhimento noturno: leitos e permanências | api, dashboard |
| F-31.6 | RAAS do mês: crítica, fechamento, remessa e ficha espelho | api, dashboard |
| F-31.7 | Matriciamento, vínculo na UBS e BPA-C | api, dashboard |
| F-31.8 | Sigilo: nada no wpda nem em avisos, linha do tempo restrita, fora do LEDI, e painel | api, dashboard, wpda |

## 9. Riscos e pendências

- **Layout da remessa RAAS:** está no site do SIA e não foi obtido. A remessa só vale depois de importada com sucesso no aplicativo RAAS ou no SIA (gate). Até lá, a ficha espelho é o caminho.
- **SIGTAP:** conferir os códigos de acolhimento inicial, diurno e noturno, o instrumento de cada um, os CBOs, as quantidades e o tempo de permanência.
- **Limite do CAPS III:** 7 noites corridas ou 10 intercaladas em 30 dias está em fonte secundária. Conferir na norma.
- **Tabelas da RAAS:** conferir as listas de origem, destino, substâncias e raça/etnia na versão vigente do aplicativo.
