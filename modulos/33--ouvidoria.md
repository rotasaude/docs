# Módulo 33 — Ouvidoria

- **Estado:** Entregue (2026-11-12); `Fechado` depois dos gates de go-live
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

## Funcionalidades

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

## Entrega — 2026-11-12

- **Commits na main:**
  - api d36aef34;
  - dashboard 4524082;
  - wpda 7482397;
  - maintenance 9269b55.
- **Cards:** F-33.1 a F-33.9 em Verified, com prova no navegador no dashboard, no wpda e no maintenance.
- **Testes:**

  | App | Resultado |
  |---|---|
  | api | suíte completa 11578/0 |
  | dashboard | 6649/0 |
  | wpda | 1075/0 |
  | maintenance | 639/0 |

- **Decisões do usuário:** U1 (par declarado com CPF e nome, sem termo da triagem), U2 (a denúncia anônima abre também no computador em produção), U3 (`ombudsman_staff` é papel privilegiado) e U4 (sem step-up para abrir identidade).
- **Decisões de reconciliação e execução:** Q1–Q62 no contrato [`superpowers/plans/2026-11-11-module-33-ombudsman-contracts.md`](../superpowers/plans/2026-11-11-module-33-ombudsman-contracts.md). Na execução: Q46–Q62, com destaque para a supressão complementar do painel (Q53–Q55) e o rótulo "oculto" (Q61).
- **Página de status:** https://claude.ai/artifact/RusHDSoJY9VNFK8rHzn2vX
- **Gates de go-live:**
  - decreto ou portaria municipal de ouvidoria (prazos locais);
  - SMS ao cidadão;
  - IP fora do log no proxy de borda e `TRUSTED_PROXIES`;
  - `client_max_body_size 15m` no nginx (anexos);
  - `Rails.cache` compartilhado entre processos em produção (limites da denúncia anônima).
- **Pendências (board #2):**
  - api#148 (Fala.BR), api#149 (ficha do OuvidorSUS), api#150 (perguntas próprias e totem de satisfação): próximo ciclo;
  - api#151 (supressão complementar nos painéis dos outros módulos);
  - api#152 (aviso com o id da manifestação);
  - dashboard#28 (rótulo "SITUAÇÃO" vazio nos cartões da Caixa no celular) e comentário no dashboard#22 (semente sem nome).

## Integrações externas

Nenhuma nesta entrega. Fala.BR (API REST) e OuvidorSUS (sem API publicada) ficam para o próximo ciclo: api#148, api#149. Detalhe e fontes em
[`pesquisa/2026-11-11-ouvidoria-33.md`](../pesquisa/2026-11-11-ouvidoria-33.md) e
[`pesquisa/2026-10-05-mapa-de-integracoes.md`](../pesquisa/2026-10-05-mapa-de-integracoes.md).

## Depende de

Módulos 06, 10, 14.

## Documentos

- Spec: [`superpowers/specs/2026-11-11-module-33-ombudsman-design.md`](../superpowers/specs/2026-11-11-module-33-ombudsman-design.md)
- Contrato e planos: `superpowers/plans/2026-11-11-module-33-ombudsman-{contracts,api,dashboard,wpda,maintenance}.md`

## Histórico

- 2026-10-05 — Registrado como `Stub` no portfólio do Ciclo 2 (brainstorm de
  novos módulos; decisão de suíte completa).
- 2026-11-11 — Desenho aprovado (ADR 0051); estado `Planejado`. Próximo ciclo no board #2: api#148 (Fala.BR), api#149 (ficha do OuvidorSUS), api#150 (totem e perguntas próprias de satisfação).
- 2026-11-12 — Entregue (F-33.1 a F-33.9 Verified nos quatro apps).
