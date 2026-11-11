# Módulo 33 — Ouvidoria

- **Estado:** Planejado
- **Tipo:** Ciclo 2

## Escopo (intenção)

Manifestações do cidadão (7 de 15 editais).

**Escopo (ADR 0051), decidido em 2026-11-11:** ouvidoria própria da secretaria de saúde, conforme a Lei 13.460/2017, numa entrega só.
- Reclamação, denúncia, elogio, sugestão e solicitação, com protocolo, prazo de 30 dias prorrogável por mais 30 (urgente em 15), complemento, respostas intermediária e conclusiva, arquivamento com motivo e reabertura.
- Identificado, com identidade preservada (só o ouvidor vê, com trilha) e anônimo (só denúncia).
- Canais: wpda com CPF e código; denúncia anônima sem login no wpda; presencial, telefone e casos do OuvidorSUS registrados pela equipe.
- Encaminhamento a setores da secretaria e unidades de saúde, sempre por extrato.
- Avisos e pesquisa de satisfação no wpda; painel e relatório de gestão anual.
- Sem integração com o Fala.BR nem com o OuvidorSUS nesta entrega.

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0051 | Ouvidoria |
| 0017 | Identidade do cidadão no wpda |
| 0026 | Retenção na exclusão |
| 0010 | Supressão e publicação de agregados |

## Funcionalidades planejadas

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-33.1 | Interruptores, papéis, setores, respondentes, assuntos e prazos | api, dashboard, maintenance | 0051 |
| F-33.2 | Manifestação pelo wpda (identificada ou com identidade preservada) com protocolo e anexos | api, wpda | 0051, 0017 |
| F-33.3 | Denúncia anônima sem login, com limites | api, wpda | 0051 |
| F-33.4 | Registro pela equipe (presencial, telefone, OuvidorSUS) e comprovante | api, dashboard | 0051 |
| F-33.5 | Classificação, encaminhamento com extrato e resposta do ponto de resposta | api, dashboard | 0051 |
| F-33.6 | Prazos, prorrogação, complemento, respostas, arquivamento e reabertura | api, dashboard | 0051 |
| F-33.7 | Identidade preservada com trilha | api, dashboard | 0051 |
| F-33.8 | "Minhas manifestações", avisos e pesquisa de satisfação | api, wpda | 0051 |
| F-33.9 | Painel e relatório de gestão anual | api, dashboard | 0051 |

## Critério de fechamento

- F-33.1 a F-33.9 verificadas, com prova no navegador.
- São gates de go-live:
  - decreto ou portaria municipal de ouvidoria (prazos locais);
  - SMS ao cidadão (o aviso fica na caixa do wpda até lá).

## Integrações externas

Nenhuma nesta entrega. Fala.BR (API REST) e OuvidorSUS (sem API publicada) ficam para o próximo ciclo: api#148, api#149. Detalhe e fontes em
[`pesquisa/2026-11-11-ouvidoria-33.md`](../pesquisa/2026-11-11-ouvidoria-33.md) e
[`pesquisa/2026-10-05-mapa-de-integracoes.md`](../pesquisa/2026-10-05-mapa-de-integracoes.md).

## Depende de

Módulos 06, 10, 14.

## Documentos

- Spec: [`superpowers/specs/2026-11-11-module-33-ombudsman-design.md`](../superpowers/specs/2026-11-11-module-33-ombudsman-design.md)

## Histórico

- 2026-10-05 — Registrado como `Stub` no portfólio do Ciclo 2 (brainstorm de
  novos módulos; decisão de suíte completa).
- 2026-11-11 — Desenho aprovado (ADR 0051); estado `Planejado`. Próximo ciclo no board #2: api#148 (Fala.BR), api#149 (ficha do OuvidorSUS), api#150 (totem e perguntas próprias de satisfação).
