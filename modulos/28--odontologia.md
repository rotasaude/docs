# Módulo 28 — Odontologia

- **Estado:** Entregue
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

## Entrega (2026-11-02)

| Repo | Commit | Observação |
|---|---|---|
| `api` | `1a1836a1` | 37 commits; suíte completa 9077 exemplos no `a63a01cf`, 3 falhas fora do módulo (bomba de data no spec do 19c, corrigida no `1a1836a1`); migração de cidade `20261030900001`; fecha api#75 |
| `dashboard` | `f052574` | 29 commits; suíte 4978/0 |
| `maintenance` | `31d37da` | interruptor `dental_care`; suíte 592/0 |
| `catalog` | `8754c1f` | PR catalog#33 mergeado (conteúdo 5226d88); F-28.2 e F-28.8 "em validação" até o aceite da FAO com CNES real e a confirmação dos códigos |
| `docs` | contrato §16 O1–O40 | runbook [`operacao/rollout-saude-bucal.md`](../operacao/rollout-saude-bucal.md) |

**Prova no navegador (2026-11-01/02):** administração e unidade CEO; registro inicial; primeira consulta com plano e gestante; paciente masculino; auxiliar (ASB); endodontista no CEO com periograma, registro periodontal com sextante excluído e endodontia; limites por SIGTAP; desfecho 15 com plano concluído; atestado; encaminhamento ao CEO com pedido de regulação (CID-10 do dentista, O40); adendo com achado novo e cancelamento de item; etapas e instalação da prótese; linha do tempo; três minutos sem polling.

**Decisões tomadas na execução** (contrato §16 O20–O40): abertura justificada pelo dentista; CBOs de saúde bucal na tabela de profissionais; cancelamento de plano; até 20 códigos de procedimento; adendo e rascunho bloqueados com o interruptor desligado; primeiro mês da meta do CEO; janela de 12 meses das faltas alternadas; desfecho 15 recusado com itens pendentes; periograma incompleto aceito no rascunho e exigido na finalização (O39); CID-10 do dentista no pedido de regulação com `dental_care` ligado (O40).

Página de status: https://claude.ai/artifact/Rtx5pYsJRf8t3kFDEmxwWb

**Gates de go-live:** FAO aceita no PEC/SIAPS real com CNES real, inclusive `localAtendimento` do CEO (api#41, O19); eSB tipo 71 no CNES real e associação à eSF/eAP (api#111); códigos sucessores das restaurações nas metas do CEO (api#112); Quadro 3 da NT 32, lista de próteses (api#113); BPA do CEO para o pagamento MAC (api#114); indicadores B3–B6 (api#115); receita de controlado do dentista depende do SNCR real (api#63). Cards no board #2, "A triar".

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
- 2026-11-02 — Entregue: api 1a1836a1, dashboard f052574, maintenance 31d37da, catálogo 8754c1f (PR #33); F-28.1..8 Verified; api#75 fechado.
