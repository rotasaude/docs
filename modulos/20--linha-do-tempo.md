# Módulo 20 — Linha do tempo do paciente

- **Estado:** Planejado
- **Tipo:** Ciclo 2

## Escopo

Linha do tempo da pessoa (ADR 0035), montada na leitura a partir de fornecedores
por módulo (triagem, atendimento, escuta, agenda, consulta, documentos,
Notificação, campanha), com duas visões: o profissional no dashboard (regras de
leitura do 19a) e o cidadão no wpda (administrativo, documentos e resumo
clínico só para par validado). No modo integrado, aviso de histórico parcial e
página para a tela de "sistemas externos" do e-SUS PEC.

**Fora, por enquanto:** exames, vacinas e procedimentos (fornecedores dos
módulos que os criarem); cópia integral do prontuário a pedido.

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0035 | Linha do tempo, visões, a pessoa, página do PEC |
| 0031 | Regras de leitura e trilha |
| 0017, 0028 | Par validado; modos e interruptores |

## Funcionalidades planejadas

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-20.1 | Interruptores e configuração do PEC | api, maintenance | 0035, 0028 |
| F-20.2 | Fornecedores e linha do tempo da pessoa | api | 0035 |
| F-20.3 | Linha do tempo do profissional | api, dashboard | 0035, 0031 |
| F-20.4 | Histórico do cidadão (linha do tempo e documentos) | api, wpda | 0035, 0017 |
| F-20.5 | Resumo clínico do cidadão | api, wpda | 0035 |
| F-20.6 | Página para os "sistemas externos" do PEC | api | 0035 |

## Critério de fechamento

- F-20.1 a F-20.6 verificadas.
- Prova no navegador (profissional nos dois modos, cidadão validado e declarado, PEC simulado).
- Gate de go-live: chamada e iframe provados num PEC real.

## Integrações externas

Nenhuma API de leitura (PEC sem leitura clínica; RNDS só em contexto de atendimento). A tela de "sistemas externos" do PEC é ponte possível no modo integrado. Detalhe e fontes em
[`pesquisa/2026-10-05-mapa-de-integracoes.md`](../pesquisa/2026-10-05-mapa-de-integracoes.md).

## Depende de

Módulos 19 (a–d).

## Em aberto

- Linha do tempo incompleta no modo integrado.

## Histórico

- 2026-10-05 — Registrado como `Stub` no portfólio do Ciclo 2 (brainstorm de
  novos módulos; decisão de suíte completa).
- 2026-10-11 — Escopo decidido com o usuário; ADR 0035 e spec
  `superpowers/specs/2026-10-11-module-20-patient-timeline-design.md`.
  F-20.1 a F-20.6 criados (board #1, `Not Started`). Módulo passa a `Planejado`.
