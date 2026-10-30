# Módulo 28 — Odontologia

- **Estado:** Planejado
- **Tipo:** Ciclo 2

## Escopo (intenção)

Atendimento odontológico (odontograma, procedimentos). Lacuna presente nos 15 editais.

**Escopo (ADR 0046), decidido em 2026-10-30:** um módulo só, com a saúde bucal da APS (equipe de saúde bucal), o CEO e a prótese; odontograma clínico completo por eventos e plano de tratamento; produção da APS e do CEO pela ficha FAO do LEDI (Portaria GM/MS 10.192/2026, NT 32/2026-CGSB). Fora: ações coletivas (B4), imagem, ortodontia, implante, Sesb, BPA do CEO.

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0046 | Saúde bucal (consulta odontológica, odontograma, CEO, prótese, FAO) |
| 0031 | Consulta da APS (tipo `dental`) |
| 0033, 0034 | Documentos e controlados do dentista (fecha o api#75) |
| 0044 | Encaminhamento ao CEO pela regulação |
| 0040 | Marco `c3_k` da linha de pré-natal |

## Funcionalidades planejadas

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-28.1 | Interruptor, profissionais de saúde bucal e unidade CEO | api, dashboard, maintenance | 0046 |
| F-28.2 | Consulta odontológica e ficha FAO | api, dashboard | 0046, 0031 |
| F-28.3 | Odontograma | api, dashboard | 0046 |
| F-28.4 | Plano de tratamento e tratamento concluído | api, dashboard | 0046 |
| F-28.5 | Especialidades: periograma, registro periodontal, endodontia | api, dashboard | 0046 |
| F-28.6 | Prótese e laboratório | api, dashboard | 0046 |
| F-28.7 | Documentos do dentista e encaminhamento ao CEO | api, dashboard | 0046, 0033, 0034, 0044 |
| F-28.8 | Metas do CEO, indicadores B1/B2 e marco da gestante | api, dashboard | 0046, 0040 |

## Critério de fechamento

- F-28.1 a F-28.8 verificadas, com prova no navegador e a FAO aceita no PEC local.

## Integrações externas

Ficha LEDI de Atendimento Odontológico (FAO, layout 8.7.0) ao SIAPS, inclusive a produção do CEO e da prótese; SIGTAP. Pesquisa em
[`pesquisa/2026-10-30-odontologia-28.md`](../pesquisa/2026-10-30-odontologia-28.md); visão geral em
[`pesquisa/2026-10-05-mapa-de-integracoes.md`](../pesquisa/2026-10-05-mapa-de-integracoes.md).

## Depende de

Módulos 16, 17, 19, 23 e 26a.

## Histórico

- 2026-10-05 — Registrado como `Stub` no portfólio do Ciclo 2 (brainstorm de
  novos módulos; decisão de suíte completa).
- 2026-10-30 — Desenho aprovado e `Planejado` (ADR 0046, spec
  `superpowers/specs/2026-10-30-module-28-oral-health-design.md`).
