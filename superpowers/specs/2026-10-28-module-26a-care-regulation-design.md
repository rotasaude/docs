# Módulo 26a — Regulação municipal — design

**Data:** 2026-10-28
**Status:** aprovado (2026-10-28)
**Afeta:**
- `apps/api`: cidade — pedidos de regulação, andamento, prestadores de consulta e procedimento, ofertas e cotas, prazos por cor, lista de procedimentos que o enfermeiro pode pedir; guia de encaminhamento em PDF; papel `care_regulator`; interruptor `care_regulation`.
- `apps/dashboard`: pedido na consulta; fila regulada; autorização com escolha de oferta; devolução, negativa, cancelamento; prestadores e cotas; agendamento externo e registro de realizado/falta; exportação para o sistema do estado.
- `apps/wpda`: situação do pedido para o cidadão.
- `apps/maintenance`: interruptor (mecanismo genérico).

**ADR:** `docs/adr/0044.md` · **Módulo:** `docs/modulos/26--regulacao.md` · **Pesquisa:** `docs/pesquisa/2026-10-05-integracoes/frente-4-faturamento-regulacao.md` (camada 2, SISREG, sistemas estaduais, PPI e consórcios)

**Fora desta entrega:** APAC, OCI (PMAE) e financiamento MAC/FAEC (26b); envio diário à RNDS no padrão MIRA (26c); teto em dinheiro; posição na fila para o cidadão e lista pública; unificação com a regulação de exames do 21a; transporte e TFD (30); escrita em SISREG, e-SUS Regulação ou sistemas estaduais (não há interface).

## 1. Intenção

Todo pedido de especialista ou procedimento sai da consulta com o que o regulador precisa para decidir (o quê, por quê, com que urgência), entra numa fila única por especialidade que respeita risco e tempo de espera, e só é autorizado quando há vaga numa oferta real. A secretaria sabe quem espera, há quanto tempo, e quanto da cota de cada prestador já foi usado.

**Sucesso:** o médico faz o pedido em um minuto na consulta; o regulador decide a fila do dia sem telefone; o paciente recebe data e local; a vigilância da fila mostra quem passou do prazo da cor.

**Antiobjetivos:** autorizar sem vaga; furar a fila sem motivo registrado; mostrar ao cidadão uma posição que muda com o risco dos outros; prometer que o pedido "foi para o SISREG".

## 2. Decisões

| # | Decisão |
|---|---|
| 1 | O 26 se divide em 26a (regulação municipal, agora), 26b (APAC, OCI, MAC/FAEC) e 26c (MIRA à RNDS) (escolha do usuário). |
| 2 | **Pedido de regulação** na consulta substitui o "encaminhado" de hoje: consulta por especialidade (CBO) ou procedimento SIGTAP; CID-10; justificativa; cor de risco sugerida; guia de encaminhamento em PDF. |
| 3 | **Quem pede:** médico (qualquer especialidade ou procedimento); enfermeiro só o que a lista municipal permitir; dentista só especialidades odontológicas. |
| 4 | **Cor de risco:** quem pede sugere; o regulador confirma ou muda, com motivo. Prazos padrão: vermelho 7 dias, amarelo 30, verde 90, azul 180; editáveis por cidade. |
| 5 | **Fila regulada** por especialidade ou procedimento, ordenada por cor e tempo de espera; papel `care_regulator` (privilegiado, segundo fator). Ações: autorizar, devolver para complementar, negar com motivo, cancelar. |
| 6 | **Oferta:** rede própria pela agenda do 17; prestadores externos (contratado, consórcio, PPI) com CNES e cota em quantidade por especialidade ou procedimento e por mês. Autorizar consome cota; cancelar devolve. |
| 7 | **Agendamento:** na rede própria, a autorização vira horário na agenda; no externo, data, hora e local registrados; depois, realizado ou falta. Falta devolve o pedido à fila uma vez; a segunda falta encerra. |
| 8 | **Convivência com o estado:** número externo (SISREG, CARE, CROSS), marca "regulado pelo estado" e lista para digitar no sistema oficial. |
| 9 | **Cidadão no wpda** vê só a situação (aguardando, agendado com data e local, devolvido, negado, encerrado), nunca a posição (escolha do usuário, opção A). |
| 10 | Prestadores de consulta e procedimento em tabela própria; a de exames (21a) não muda. |
| 11 | Exclusão LGPD: pedido retido, como o prontuário. |
| 12 | Interruptor `care_regulation`, que requer `clinical_record` (o pedido nasce na consulta). |

Decisões 1 e 9 são do usuário; as demais são do coordenador, apresentadas e aprovadas.

**Decisões do coordenador (revisão da spec):**
- **Rede própria:** também tem cota mensal (capacidade que a unidade oferece à regulação). Autorizar numa oferta própria cria um pedido de agendamento do 17 (`kind: regulated`) na unidade de destino, e a recepção marca o horário, como no ADR 0029; o 26a não escolhe horário. Ordem de travas: oferta → pedido de regulação → pedido de agendamento.
- **Encaminhamento de hoje:** com o interruptor ligado, encaminhamento a especialista ou procedimento é sempre pedido de regulação; o encaminhamento simples dentro da rede (outra UBS, sem especialidade) continua como `referral` do 17. O pedido nasce no rascunho da consulta e vale na finalização (como os pedidos de exame); pode haver vários por consulta e pode entrar por adendo.
- **Cota:** conta no mês da autorização; autorizar consome; cancelar antes do atendimento devolve; falta não devolve (a vaga foi usada); negar só antes de autorizar (depois, cancela); remarcar mantém o mês. Saldo derivado da soma das autorizações, com trava na oferta.
- **Ordem da fila:** a ordenação é de exibição; autorizar um pedido quando existe outro mais antigo de cor igual ou mais grave, na mesma especialidade ou procedimento e com oferta elegível, exige motivo (auditado).
- **Escopo do regulador:** a cidade toda (equipe central); escopo por especialidade fica para depois.
- **Regulado pelo estado:** não entra na fila municipal nem consome cota; fica registrado com o número externo (situação `external`) até o regulador lançar data, local e desfecho. A exportação é um CSV livre com os campos necessários para digitar no sistema do estado, sem layout oficial.
- **Devolvido para complementar:** qualquer profissional habilitado da unidade solicitante complementa; é um evento no mesmo pedido; o tempo de espera e o prazo da cor continuam contando da data do pedido original.
- **Quem pede, pelo CBO do vínculo da consulta:** médico `225*`; enfermeiro `2235*` (só itens da lista municipal, que guarda especialidades CBO e procedimentos SIGTAP); dentista `2232*` (só especialidades odontológicas `2232*`).

## 3. Dados

Cidade:
- `regulation_requests`: paciente, consulta, solicitante (CBO), tipo (`specialty` \| `procedure`), especialidade CBO ou SIGTAP, CID-10, justificativa (cifrada), cor sugerida e confirmada, situação, número externo, regulado pelo estado, prazo, oferta autorizada, agendamento, desfecho.
- `regulation_request_events` (imutável): situação, autor, motivo (cifrado).
- `care_providers`: nome, CNES, tipo (`own` \| `contracted` \| `consortium` \| `ppi`), ativo.
- `care_offers`: prestador ou unidade própria, especialidade ou SIGTAP, mês, quantidade, usada.
- `regulation_risk_deadlines` (prazo por cor, por cidade) e `nurse_requestable_items` (lista municipal).

## 4. Telas

- **Consulta:** "Pedir regulação" com especialidade/procedimento, CID, justificativa e cor; guia em PDF.
- **Fila do regulador:** por especialidade, com prazo da cor, tempo de espera e vencidos no topo; autorizar escolhendo a oferta com saldo; devolver, negar, cancelar.
- **Solicitante:** pedidos devolvidos para complementar.
- **Prestadores e cotas** (administrador): cadastro e cota mensal, com saldo.
- **Agendamento externo e desfecho:** data/local; realizado ou falta.
- **Convivência:** número externo e exportação CSV.
- **wpda:** "Meus encaminhamentos", com a situação.

## 5. Segurança e LGPD

Trilha `clinical_record.viewed` em leitura nominal (sem polling); contagens sem trilha. Justificativa e motivos cifrados. Nada pessoal em evento, log, URL, `inspect` ou `as_json`. Ordem de travas: oferta → pedido. Eventos novos de `Platform.audit` em R18.

## 6. Testes

- api: transições; consumo e devolução de cota sob concorrência (duas autorizações para a última vaga); prazos por cor no fuso da cidade; quem pode pedir; falta e reentrada; retenção LGPD; escopo do regulador.
- dashboard e wpda: telas com testes de componente.
- Suíte completa do api no fim.

## 7. Entrega

api → dashboard e wpda → maintenance; runbook `operacao/rollout-regulacao.md`.

## 8. F-IDs (propostos)

| ID | Funcionalidade | Superfície |
|---|---|---|
| F-26.1 | Interruptor, papel de regulador e prazos por cor | api, dashboard, maintenance |
| F-26.2 | Pedido de regulação na consulta e guia | api, dashboard |
| F-26.3 | Fila regulada e decisões do regulador | api, dashboard |
| F-26.4 | Prestadores, ofertas e cotas | api, dashboard |
| F-26.5 | Agendamento e desfecho (realizado, falta) | api, dashboard |
| F-26.6 | Convivência com o sistema do estado | api, dashboard |
| F-26.7 | Situação do pedido no wpda | api, wpda |

## 9. Riscos

- Cidade que só usa o sistema do estado (mitigação: número externo e marca "regulado pelo estado").
- Cota mal cadastrada trava a fila (mitigação: saldo visível e alerta de cota esgotada).
- Divergência com a regulação de exames do 21a (mitigação: registrada como evolução).
