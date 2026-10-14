# Módulo 21 — Exames e laboratório

- **Estado:** 21a Entregue (F-21.1 a F-21.13 provadas no navegador e `Verified` em 2026-10-14; `Fechado` depois dos gates de go-live: primeiro laboratório real na entrada FHIR e botão no PEC real, rotasaude/api#64); 21b Stub
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
- **21b — laboratório municipal** (coleta com etiqueta, mapa de trabalho,
  digitação e liberação de laudo), lacuna dos editais. Brainstorm próprio depois
  do 21a.

**Fora, por enquanto:** regulação completa (módulo 26); BPA-I; HL7 v2; REL/RELC
na RNDS; GAL; DICOM; aviso por SMS ou e-mail.

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
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
SIGTAP↔LOINC, identificação do paciente); botão no PEC real (rotasaude/api#64);
LGPD — exames na exclusão (ADR 0026) antes de ligar `exam_tracking` numa cidade
real.

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
