# Módulo 21 — Exames e laboratório

- **Estado:** 21a Entregue (F-21.1 a F-21.13 provadas no navegador e `Verified` em 2026-10-14; `Fechado` depois dos gates de go-live: primeiro laboratório real na entrada FHIR e botão no PEC real, rotasaude/api#64); 21b Entregue (F-21.14 a F-21.24 provadas no navegador em 2026-10-16; `Fechado` depois dos gates: primeiro middleware real, dossiê do art. 106 da RDC 978, certificados ICP dos liberadores médicos); 21c Planejado (ADR 0038)
- **Tipo:** Ciclo 2

## Escopo

Dois subprojetos:

- **21a — rastreamento do exame** (ADR 0036): pedido (consulta, guia da
  recepção ou botão no PEC) → autorização (fila de regulação simples) →
  agendamento → coleta → resultado (unidade, portal do prestador ou entrada
  FHIR) → ciência do profissional, com prazo por etapa (prestador × exame,
  catálogo, padrão) e alerta de atraso; resultado externo; "Meus exames" no
  wpda com laudo por política do exame; relatório de conferência por
  prestador; exames na linha do tempo.
- **21b — laboratório municipal, fluxo analítico** (ADR 0037): o 21a é a fonte
  única do exame; catálogo-base da plataforma com versões por laboratório;
  requisição, etiqueta Code 128, coleta na UBS (com remessa) ou no laboratório,
  recebimento com rejeição e recoleta; mapa de trabalho; digitação e entrada HL7
  v2.5.1 do middleware do fornecedor do reagente; liberação por `lab_releaser`
  com ICP ou assinatura avançada (médico só ICP); laudo da RDC 978; valores
  críticos; retificação; produção; registro do GAL.
- **21c — qualidade e insumos** (ADR 0038): CIQ com lotes, níveis e referência
  (bula, depois laboratório), valores digitados ou pela HL7, Westgard avaliado
  na gravação com bloqueio da liberação, exceção do RT e revisão; registro do
  CEQ; estoque de reagentes e insumos com lotes, lote em uso guardado em cada
  resultado e controle (rastreio por lote) e consumo estimado sem baixa
  automática. Importação CSV de controles em rotasaude/api#87.

**Fora, por enquanto:** regulação completa (módulo 26); BPA-I; HL7 v2; REL/RELC
na RNDS; GAL; DICOM; aviso por SMS ou e-mail.

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0038 | Qualidade e insumos: CIQ, Westgard, bloqueio, CEQ, estoque com lotes e rastreio |
| 0037 | Laboratório municipal: catálogo com versões, amostras, HL7, liberação e laudo |
| 0036 | Exame com ciclo de vida, regulação simples, prestador, FHIR na borda, prazos, ciência, política de liberação |
| 0031 | Regras de leitura e trilha |
| 0035 | Linha do tempo e ponte com o PEC |
| 0017, 0028 | Par validado; interruptores |

## Funcionalidades planejadas

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-21.1 | Interruptores e configuração dos exames | api, maintenance | 0036, 0028 |
| F-21.2 | Catálogo de exames e prestadores da cidade | api, dashboard | 0036 |
| F-21.3 | Exame com ciclo de vida (consulta e guia da recepção) | api, dashboard | 0036, 0031 |
| F-21.4 | Fila de regulação de exames | api, dashboard | 0036 |
| F-21.5 | Prazos e painel de exames atrasados | api, dashboard | 0036 |
| F-21.6 | Portal do prestador | api, dashboard | 0036 |
| F-21.7 | Entrada FHIR de resultados e laboratório simulado | api, contracts | 0036 |
| F-21.8 | Resultado, ciência e fila da equipe | api, dashboard | 0036 |
| F-21.9 | Resultado externo | api, dashboard | 0036 |
| F-21.10 | Exames do cidadão no wpda | api, wpda | 0036, 0017 |
| F-21.11 | Solicitar exames pelo PEC | api, dashboard | 0036, 0035 |
| F-21.12 | Relatório de conferência por prestador | api, dashboard | 0036 |
| F-21.13 | Exames na linha do tempo (fecha api#71 para exames) | api | 0036, 0035 |
| F-21.14 | Interruptores e catálogo-base do laboratório | api, maintenance | 0037, 0028 |
| F-21.15 | Catálogo do laboratório com versões | api, dashboard | 0037 |
| F-21.16 | Requisição, etiqueta e coleta | api, dashboard | 0037, 0036 |
| F-21.17 | Remessa da UBS e recebimento com rejeição | api, dashboard | 0037 |
| F-21.18 | Mapa de trabalho e digitação | api, dashboard | 0037 |
| F-21.19 | Entrada HL7 e middleware simulado | api, contracts | 0037 |
| F-21.20 | Liberação com assinatura e laudo | api, dashboard | 0037, 0032, 0033 |
| F-21.21 | Valores críticos | api, dashboard | 0037 |
| F-21.22 | Retificação de laudo do laboratório municipal (parte de api#85) | api, dashboard | 0037, 0036 |
| F-21.23 | Relatórios de produção, rejeição e prazo | api, dashboard | 0037 |
| F-21.24 | Registro do GAL | api, dashboard | 0037, 0036 |
| F-21.25 | Interruptores e aparelhos do laboratório | api, dashboard, maintenance | 0038, 0028 |
| F-21.26 | Materiais, lotes e referências de controle (bula e laboratório) | api, dashboard | 0038 |
| F-21.27 | Corridas de CIQ e regras de Westgard | api, dashboard | 0038 |
| F-21.28 | Controle pela HL7 | api, contracts | 0038, 0037 |
| F-21.29 | Bloqueio da liberação, exceção do RT e revisão | api, dashboard | 0038, 0037 |
| F-21.30 | Gráfico de Levey-Jennings | dashboard | 0038 |
| F-21.31 | Registro do CEQ | api, dashboard | 0038 |
| F-21.32 | Estoque de reagentes e insumos com lotes | api, dashboard | 0038 |
| F-21.33 | Lote em uso e rastreabilidade do reagente | api, dashboard | 0038 |
| F-21.34 | Relatório de consumo estimado | api, dashboard | 0038 |

## Critério de fechamento (21a)

- F-21.1 a F-21.13 verificadas (F-21.11 só se a prova técnica no PEC local passar).
- Prova no navegador com laboratório simulado: pedido → autorização → agendamento pelo prestador → resultado FHIR → ciência; atraso e escalonamento; cidadão nas três políticas.
- Gates de go-live: primeiro laboratório real na entrada FHIR; botão no PEC real.

## Entrega do 21a (2026-10-14)

| Repo | Commit | Observação |
|---|---|---|
| `contracts` | `2598335` | tag `exams-v1.0.0` (`exams/exam-order-v1.json`, `exams/fhir-diagnostic-report-inbound.json`) |
| `api` | `15facaf` | 56 commits sobre `4ffe7c6`; suíte 5686/0 |
| `dashboard` | `c56e16e` | 40 commits sobre `94e3a48`; 260 arquivos / 2732 testes |
| `wpda` | `1b5012a` | 9 commits sobre `7336dd6`; 55 arquivos / 660 testes; e2e 15/0 |
| `maintenance` | `068b4a1` | 3 commits sobre `cc91351`; 36 arquivos / 358 testes |
| `docs` | `f94a417`, `0867062` | provas técnicas e runbook |
| `catalog` | rotasaude/catalog#27 | F-21.7 e F-21.11 "em validação" |

Migração de cidade `20261013900001` (tabelas do exame, trilha só de acréscimo
com trigger, `priority` no pedido do 19a). Contrato com acréscimos A1–A48
(`superpowers/plans/2026-10-13-module-21a-exams-contracts.md` §15).
Runbook [`operacao/rollout-rastreamento-de-exames.md`](../operacao/rollout-rastreamento-de-exames.md).

**Provas técnicas** ([`pesquisa/2026-10-13-exames-provas-tecnicas.md`](../pesquisa/2026-10-13-exames-provas-tecnicas.md)):
envio de formulário dentro da tela de sistemas externos — passou por reprodução
fiel do iframe e da CSP do PEC 5.5.28 (o PEC local não tinha unidade para abrir
atendimento); perfil do `DiagnosticReport` conferido contra o FHIR R4 e os
NamingSystem da RNDS.

**Provas no navegador (2026-10-14)**, com o laboratório simulado (`fake-lab`):
dashboard nos 13 itens (pedido, regulação, portal do prestador, resultado FHIR,
ciência, fila da equipe, atrasos, guia da recepção, resultado externo,
relatório); wpda (três políticas, `never_citizen` só após a ciência, negado sem
motivo, crítico invisível, PDF pela linha do tempo, par declarado, interruptor
desligado, alto contraste, 375 px); maintenance (quatro interruptores, aviso do
laboratório simulado); F-21.11 pelo simulador do PEC (Maringá, `integrated`:
CSP, pedido com origem `pec`, deduplicação, chamada expirada). Dois defeitos
achados e corrigidos no api (`15facaf`: equipe de referência na semente de dev,
log do fake-lab).

**Decisões tomadas na execução** (contrato §15): `exam_tracking` nos dois modos
de prontuário; equipe de referência = equipes ativas da unidade do exame; botão
no PEC dentro da página do módulo 20; `fake_lab` em dev e staging; funcionário
do prestador na trilha; regulador vê o CID; `never_citizen` só após a ciência;
laudo de correção (`amended`) recusado; contador de ciência sem trilha.

**Gates de go-live:** primeiro laboratório real na entrada FHIR (contrato,
SIGTAP↔LOINC, identificação do paciente); botão no PEC real (rotasaude/api#64).
LGPD — exames na exclusão (ADR 0026, rotasaude/api#84): resolvido em api
`0e81173` (exame retém a exclusão; reconferência na confirmação com a mesma regra).

## Entrega do 21b (2026-10-16)

| Repo | Commit | Observação |
|---|---|---|
| `contracts` | `a595d5d` | tags `lab-v1.0.0` (perfil HL7, `lab-report-v1`) e `exams-v1.1.0` (`exam-order-v1` com campos opcionais novos) |
| `api` | `2deea81` | 55 commits sobre `0e81173`; suíte 5982/0 |
| `dashboard` | `31e24f6` | 48 commits sobre `c56e16e`; 290 arquivos / 3042 testes |
| `maintenance` | `c3a534d` | 8 commits sobre `068b4a1` |
| `docs` | `b99c840`, `b43c92d` | provas técnicas e runbook |

Migrações `20261015900001` (plataforma: catálogo-base e valor SIGTAP) e
`20261015900002` (cidade: tabelas do laboratório, irreversível). Gem nova
`barby` (imagem do api reconstruída no deploy). Contrato com acréscimos B1–B64
(`superpowers/plans/2026-10-15-module-21b-lab-contracts.md` §14).
Runbook [`operacao/rollout-laboratorio-municipal.md`](../operacao/rollout-laboratorio-municipal.md).

**Provas técnicas** ([`pesquisa/2026-10-15-laboratorio-provas-tecnicas.md`](../pesquisa/2026-10-15-laboratorio-provas-tecnicas.md)):
perfil HL7 v2.5.1 (`QBP^Q11`, `RSP^K11`, `OUL^R22`, `ACK`) conferido contra a
norma; etiqueta Code 128 de 50 × 25 mm lida pelo `zbarimg` a 203 e 300 dpi.

**Provas no navegador (2026-10-16)**, com o middleware e o PSC simulados:
requisição e etiqueta; coleta por leitura; remessa com temperatura; recebimento
com rejeição e recoleta automática; resultados pela HL7 (com crítico e
repetição) e digitação do hemograma com faixas por sexo; liberação parcial com
assinatura avançada e com ICP (patologista); crítico sem aviso e registro do
aviso; retificação; laudo no 21a e no wpda nas três políticas (PDF entregue =
laudo vigente); conferência pública sem valores; relatórios e CSV; entrega;
GAL só na visão profissional e administrativa; catálogo-base e interruptores no
maintenance. Defeitos achados e corrigidos na prova: leitor deslocando botões,
decimal com ponto, componentes sem nome visível, "Pendentes de assinatura"
inalcançável para o liberador, lista de pendentes sem atualizar, motivo do
escalonamento deduzido em vez de gravado.

**Decisões tomadas na execução** (contrato §14): código do espécime local
derivado do material; todas as recusas HL7 de uma vez, estrutura sempre `AR`;
limite crítico inclusivo; valor efetivo por linha; marca do prestador vale
independentemente do interruptor; candidatos entre unidades só por paciente e
com trilha; crítico liberado nunca sai da fila sem aviso; laudo ICP congelado
até assinar; liberador encerrado não assina; retificação substitui o laudo
inteiro; mensagem HL7 bruta guardada 5 anos (rotasaude/api#86); ponto recusado
em número digitado.

**Gates de go-live:** primeiro middleware real ligado à HL7; dossiê de
validação do sistema (art. 106 da RDC 978) e aceite da assinatura avançada pela
VISA; certificados ICP dos liberadores médicos; REL/RELC à RNDS quando o Rota
Saúde for o LIS (fora do módulo).

## Integrações externas

v1 (21a) sem integração obrigatória: portal do prestador e entrada FHIR
(`DiagnosticReport` + `Observation` LOINC) com token por prestador, provada com
laboratório simulado. v2 integração por laboratório real; v3 REL/RELC na RNDS
(especificação hoje só COVID/Mpox). GAL sem API. Detalhe e fontes em
[`pesquisa/2026-10-05-integracoes/frente-3-exames.md`](../pesquisa/2026-10-05-integracoes/frente-3-exames.md)
e [`pesquisa/2026-10-05-mapa-de-integracoes.md`](../pesquisa/2026-10-05-mapa-de-integracoes.md).

## Depende de

Módulos 19 (pedido de exame) e 20 (linha do tempo, ponte com o PEC).

## Em aberto

- Arranjo de exames e regulação de cada cidade-piloto e LIS usado.
- Escrita pela tela de "sistemas externos" do PEC (prova técnica).
- Mapeamento SIGTAP↔LOINC do primeiro laboratório real.
- Escopo do 21b.

## Histórico

- 2026-10-05 — Registrado como `Stub` no portfólio do Ciclo 2 (brainstorm de
  novos módulos; decisão de suíte completa).
- 2026-10-13 — Brainstorm: dividido em 21a (rastreamento) e 21b (laboratório
  municipal); 21a desenhado e `Planejado` (spec
  `superpowers/specs/2026-10-13-module-21a-exam-tracking-design.md`, ADR 0036).
- 2026-10-14 — **21a implementado, publicado e Entregue** (13 cards `Verified`;
  ver "Entrega do 21a"). `Fechado` depois dos gates de go-live.
- 2026-10-14 — rotasaude/api#84 resolvido (api `0e81173`): exame conta como
  documento clínico na exclusão LGPD; a reconferência também passou a olhar
  documentos do 19c.
- 2026-10-15 — Brainstorm do 21b (pesquisas `pesquisa/2026-10-15-laboratorio-municipal-21b.md`
  e `pesquisa/2026-10-15-laboratorio-analisadores.md`): objetivo prova de conceito
  dos editais; dividido em 21b (fluxo analítico, `Planejado`, ADR 0037, spec
  `superpowers/specs/2026-10-15-module-21b-municipal-lab-design.md`) e 21c
  (controle de qualidade e estoque, `Stub`).
- 2026-10-16 — **21b implementado, publicado e Entregue** (ver "Entrega do
  21b"). `Fechado` depois dos gates de go-live. A retificação de laudo do
  laboratório municipal saiu no 21b; a de prestador externo segue em
  rotasaude/api#85.
- 2026-10-18 — Brainstorm do 21c: CIQ completo com CEQ, estoque com lotes e
  consumo estimado; `Planejado` (ADR 0038, spec
  `superpowers/specs/2026-10-18-module-21c-lab-quality-supplies-design.md`).
  Importação CSV de controles com colunas configuráveis em rotasaude/api#87.
