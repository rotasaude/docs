# Módulo 28 — Saúde bucal (APS, CEO e prótese) — design

**Data:** 2026-10-30
**Status:** aprovado (2026-10-30)
**Afeta:**
- `apps/api`: cidade — atendimento odontológico (tipo da consulta do 19a), odontograma por eventos, plano de tratamento, registros de especialidade (periograma, registro periodontal simplificado, endodontia), pedidos de prótese ao laboratório, unidade CEO com especialidades e tipo, metas do CEO, indicadores B1/B2; ficha LEDI de Atendimento Odontológico (FAO); documentos do dentista (19c/19d); marco `c3_k` da linha de pré-natal (23) passa a ser medido; equipes de saúde bucal importadas do CNES; interruptor `dental_care`.
- `apps/dashboard`: atendimento odontológico com odontograma, plano, especialidades e prótese; painel de metas do CEO e de indicadores da equipe de saúde bucal; cadastro da unidade CEO.
- `apps/maintenance`: interruptor (mecanismo genérico).

**ADR:** `docs/adr/0046.md` · **Módulo:** `docs/modulos/28--odontologia.md` · **Pesquisa:** `docs/pesquisa/2026-10-30-odontologia-28.md` · ADR 0031 (consulta), 0033/0034 (documentos e controlados), 0035 (linha do tempo), 0029 (agenda), 0044 (regulação), 0040 (linhas de cuidado) · card api#75

**Fora desta entrega:** escovação supervisionada e demais ações coletivas (ficha de Atividade Coletiva, transversal; indicador B4); imagem radiográfica; ortodontia e implante; Sesb (Portaria 751/2023); BPA do CEO ao SIA (ver §9); indicadores B3, B5 e B6 até a leitura das notas metodológicas.

## 1. Intenção

O dentista da unidade e o especialista do CEO registram o atendimento no Rota Saúde com o odontograma do paciente na tela, seguem um plano de tratamento até a conclusão, emitem atestado e receita, e a produção chega ao SIAPS pela ficha LEDI, sem PEC paralelo nem BPA.

**Por quê:** odontologia com odontograma está em 15 de 15 editais; desde 05/08/2026 a produção do CEO e do laboratório de prótese só conta para o financiamento federal se chegar ao SIAPS (Portaria GM/MS 10.192/2026, NT 32/2026-CGSB), e sistema próprio integrado via LEDI é aceito.

**Sucesso:** o dentista abre o paciente e vê a boca inteira com o que já foi feito e o que falta; a coordenação de saúde bucal vê no meio do mês se o CEO vai bater a meta de cada especialidade; a gestante aparece com o marco de saúde bucal cumprido na linha de pré-natal.

**Antiobjetivos:** odontograma editado à mão, sem atendimento que o explique; produção do CEO que não chega ao SIAPS; dentista fora do sistema para atestado e receita; ASB ou TSB registrando encaminhamento.

## 2. Decisões

| # | Decisão |
|---|---|
| 1 | **Um módulo só** cobre a saúde bucal da APS, o CEO e a prótese (escolha do usuário, opção B). |
| 2 | **Odontograma clínico completo:** situação por dente e por face, mais plano de tratamento (escolha do usuário, opção A). |
| 3 | O **atendimento odontológico é um tipo da consulta do 19a** (`kind: dental`): reaproveita rascunho, finalização imutável, adendo, documentos e linha do tempo; gera a FAO no lugar da ficha de Atendimento Individual. A consulta comum continua fechada ao dentista (decisão do usuário de 2026-10-12). |
| 4 | **Odontograma por eventos:** cada marcação é um evento imutável preso a uma consulta odontológica (ou ao adendo dela); a situação atual é calculada. A situação de quem já é paciente entra por um **exame inicial**, que é uma consulta odontológica como outra qualquer. |
| 5 | **Dentições:** permanente (11–48) e decídua (51–85); faces M, D, O/I, V/B, L/P; procedimentos por dente, face, sextante (S1–S6), arcada (superior/inferior) ou boca toda. |
| 6 | **Plano de tratamento** por paciente: itens SIGTAP com local (dente/face/sextante/arcada); cada item passa a feito na consulta que o realiza, ou é cancelado com motivo. Plano todo feito → o sistema sugere o desfecho "tratamento concluído" (FAO 15), que só vale com primeira consulta programática ou retorno. |
| 7 | **FAO via LEDI**, pelo mesmo exportador do módulo 16. O LEDI não leva dente nem face: a exportação agrega por SIGTAP e soma a quantidade, respeitando os limites (dente ≤ 32, sextante ≤ 6, arcada ≤ 2, demais ≤ 1). A produção do CEO vai pela mesma FAO, no CNES do CEO. |
| 8 | **Quem registra:** a família 2232 (todas as especialidades) registra tudo; TSB (322405, 322425) e ASB (322415, 322430) registram a consulta odontológica com procedimentos de uma lista da cidade, sem encaminhamento, sem documento e sem plano. |
| 9 | **Unidade CEO:** novo tipo de unidade `ceo`, com o tipo de CEO (I, II, III) e as especialidades que oferece (endodontia, periodontia, cirurgia oral menor, estomatologia, pacientes com necessidades especiais, prótese). |
| 10 | **Registros de especialidade:** periograma com 6 sítios por dente (profundidade de sondagem, sangramento, recessão, mobilidade e furca por dente); registro periodontal simplificado por sextante na APS; endodontia por dente (canais, sessões, situação). Todos imutáveis depois de finalizada a consulta, como o odontograma. |
| 11 | **Prótese:** pedido ao laboratório (próprio ou terceiro), com etapas moldagem → enviada → recebida → instalada; a produção conta **na instalação**, registrada pelo dentista na consulta, no CNES da unidade que entrega (NT 32). O protético não usa o sistema. |
| 12 | **Metas do CEO:** painel mensal por especialidade contra a meta do tipo de CEO (Portaria 1.464/2011, Anexo XL da PC 6/2017), com as regras de composição (básicos só de paciente com necessidades especiais, ≥ 50% restaurações; endodontia ≥ 20% em dentes de 3+ raízes) e alerta de suspensão (2 meses seguidos ou 3 alternados em 12). |
| 13 | **Indicadores da equipe de saúde bucal:** B1 (primeira consulta programática) e B2 (tratamento concluído), por equipe, com as regras das notas metodológicas. |
| 14 | **Encaminhamento ao CEO** é o desfecho da FAO (1–11) e, com `care_regulation` ligado, vira pedido de regulação do 26a para especialidade odontológica (regra que já existe). |
| 15 | **Documentos do dentista** (fecha o api#75): atestado, receita comum, receita controlada (19d) e pedido de exame, presos à consulta odontológica, no âmbito odontológico. |
| 16 | **Gestante:** a FAO exige o campo; vem da inscrição ativa na linha de pré-natal (23) e o dentista confirma; o marco `c3_k` passa a ser medido pela consulta odontológica na gestação. |
| 17 | Interruptor `dental_care` (requer `clinical_record`); exclusão LGPD retém tudo, como o prontuário (ADR 0026). |

Decisões 1 e 2 são do usuário; as demais são do coordenador, apresentadas e aprovadas no desenho.

**Decisões do coordenador (revisão da spec):**
- **Plano de tratamento por unidade e episódio:** no máximo um plano aberto por paciente em cada unidade (a UBS e o CEO têm planos separados). O plano nasce na primeira consulta programática (na UBS) ou na primeira consulta do paciente no CEO, e fecha com o desfecho "tratamento concluído" ou é cancelado com motivo. O próximo plano começa com uma nova primeira consulta. A sugestão do desfecho 15 olha só o plano da unidade da consulta.
- **Equipes de saúde bucal:** as eSB do CNES (com INE próprio) passam a ser importadas como equipes (`HealthTeam`, novos tipos), e o dentista, o TSB e o ASB se vinculam a elas como já acontece com a eSF e a eAP. B1 e B2 são calculados por eSB, com os CBOs e o denominador da nota metodológica (vinculados à eSF/eAP associada à eSB). O especialista do CEO não entra no B1.
- **CEO coberto:** o CEO operado pela cidade, inclusive o que atende moradores de outros municípios; o morador de fora vira paciente da cidade como em qualquer atendimento. O CEO de consórcio ou de outro município continua como prestador externo do 26a, sem prontuário, FAO nem metas aqui.
- **TSB e ASB no odontograma:** os procedimentos deles geram eventos no odontograma e podem dar baixa num item do plano cujo SIGTAP esteja na lista da cidade. Não criam nem cancelam item.
- **Registro inicial sem ficha:** o exame inicial de quem já está em tratamento é uma consulta odontológica marcada como `baseline`. Ela só registra a situação dos dentes, sem procedimento, desfecho nem tipo de consulta, e **não gera FAO**. Atendimento real continua sendo consulta normal.
- **Vocabulário do odontograma:** espelha a lista de situações do odontograma do PEC (hígido, cariado, restaurado, ausente, extração indicada, prótese, entre outras). Uma tabela versionada SIGTAP → situação muda a face ou o dente automaticamente quando o procedimento é registrado. O dentista também pode marcar a situação diretamente, como achado do exame.
- **Marco `c3_k`:** conta a consulta odontológica finalizada, com `gestante = true`, de CBO 2232 ou 3224, dentro da janela da gestação. O filtro de CBO fica no catálogo (`cbo_filter`), para acompanhar a nota metodológica.

## 3. Dados

Cidade:
- `consultations.kind` (`general` | `dental`) e campos da FAO na consulta odontológica: tipo de atendimento, tipo de consulta odontológica, vigilância em saúde bucal (1–7 códigos), fornecimento, desfechos (`tiposEncamOdonto`), gestante, necessidades especiais.
- `dental_chart_events` (imutável): paciente, consulta (ou adendo), dente, face (opcional), situação ou procedimento SIGTAP, autor, hora.
- `dental_treatment_plans` e `dental_treatment_plan_items`: SIGTAP, local, situação (`planned`, `done`, `cancelled`), consulta que realizou, motivo do cancelamento.
- `dental_procedures` da consulta: SIGTAP, local, quantidade (base da agregação da FAO).
- `periodontal_charts` (+ sítios), `periodontal_screenings` (por sextante), `endodontic_treatments` (+ sessões).
- `dental_prosthesis_orders`: paciente, tipo (SIGTAP de prótese), laboratório, etapas com data e autor, consulta da instalação.
- `health_units.kind` ganha `ceo`; `ceo_profiles` (tipo I/II/III, especialidades).
- `dental_auxiliary_procedures`: lista da cidade do que TSB e ASB registram.
- `health_teams` ganha os tipos de equipe de saúde bucal do CNES (importados pelo `Cnes::Apply`, com INE e associação à eSF/eAP).
- `consultations.baseline` (registro inicial, só situação dos dentes, sem FAO).
- Tabelas de configuração versionadas: listas SIGTAP das metas do CEO e dos indicadores B1/B2.

Plataforma: nenhuma.

## 4. Telas (dashboard)

- **Consulta odontológica:** odontograma (dentição permanente/decídua/mista), plano de tratamento ao lado, procedimentos do dia marcados no dente, campos da FAO, problemas (CID-10/CIAP-2), documentos, desfecho.
- **Especialidades (CEO):** periograma, endodontia, prótese (pedido e etapas).
- **Painéis:** metas do CEO (mês corrente e últimos 12 meses, risco de suspensão); B1/B2 por equipe.
- **Cadastro:** unidade CEO (tipo e especialidades); lista de procedimentos do TSB/ASB.

## 5. Segurança e LGPD

Leitura do odontograma, do plano e das especialidades segue a regra de leitura do prontuário (ADR 0031: em contexto ou abertura justificada) com trilha `clinical_record.viewed`; painéis de metas e indicadores só com contagens, sem trilha e sem dado pessoal. Nada pessoal em evento, log, URL, `inspect` ou `as_json`. Eventos imutáveis por trigger em `db/city_triggers.sql`. Novos `Platform.audit` em R18.

## 6. Testes

- api: odontograma calculado pelos eventos (inclusive adendo); imutabilidade; plano → sugestão de tratamento concluído e regras da FAO (15 só com primeira consulta ou retorno; 17 vedado com elas; tipo de consulta condicional ao tipo de atendimento); TSB/ASB sem encaminhamento, documento e plano; agregação da FAO por SIGTAP com limites; ficha validada contra o layout LEDI 8.7.0; metas do CEO com as regras de composição e de suspensão; B1 uma vez por pessoa e dentista em 12 meses; gestante proibida para sexo masculino; marco `c3_k`; documentos do dentista.
- dashboard: odontograma, plano, periograma e painéis com testes de componente.
- Prova no PEC local (módulo 16): FAO aceita.
- Suíte completa do api no fim.

## 7. Entrega

api → dashboard → maintenance; runbook `operacao/rollout-saude-bucal.md` (cadastro das unidades CEO, lista do TSB/ASB, exame inicial dos pacientes em tratamento, CNES do laboratório de prótese terceiro como "Terceiro").

## 8. F-IDs (propostos)

| ID | Funcionalidade | Superfície |
|---|---|---|
| F-28.1 | Interruptor, profissionais de saúde bucal e unidade CEO | api, dashboard, maintenance |
| F-28.2 | Consulta odontológica e ficha FAO | api, dashboard |
| F-28.3 | Odontograma | api, dashboard |
| F-28.4 | Plano de tratamento e tratamento concluído | api, dashboard |
| F-28.5 | Especialidades: periograma, registro periodontal, endodontia | api, dashboard |
| F-28.6 | Prótese e laboratório | api, dashboard |
| F-28.7 | Documentos do dentista e encaminhamento ao CEO | api, dashboard |
| F-28.8 | Metas do CEO, indicadores B1/B2 e marco da gestante | api, dashboard |

## 9. Riscos e pendências

- **BPA do CEO ao SIA:** a NT 32 diz que o BPA deixou de valer para o financiamento federal dos serviços habilitados, mas não diz se o pagamento por produção de média complexidade (PPI/FPO) ainda o exige. O 28 não gera BPA; se for exigido, reaproveita o módulo 22 depois.
- Listas SIGTAP de prótese (Quadro 3 da NT 32) e dos indicadores B3–B6 não lidas na fonte oficial.
- O PEC pode somar dentes de forma diferente da nossa agregação (prova no PEC local).
- Exame inicial de quem já está em tratamento é trabalho de implantação (runbook).
