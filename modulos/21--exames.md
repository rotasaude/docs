# Módulo 21 — Exames e laboratório

- **Estado:** Planejado (21a); 21b Stub
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
