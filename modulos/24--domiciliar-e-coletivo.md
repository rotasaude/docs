# Módulo 24 — Atendimento domiciliar e coletivo

- **Estado:** Stub
- **Tipo:** Ciclo 2

## Escopo (intenção)

Visita do ACS/eSF, cadastro individual e domiciliar, atividade coletiva; app offline para tablet (11 de 15 editais).

**Divisão (ADR 0041):**
- **Núcleo, no Rota Saúde:** visita, cadastro individual e domiciliar, atividade coletiva, fichas LEDI e as fontes das linhas de cuidado (23), da linha do tempo (20) e do território (11).
- **Ferramenta, em repositório próprio:** app de campo offline para tablet (`rotasaude/campo`), com sincronização e resolução de conflitos, integrado por contrato; nunca é fonte de verdade.

## Integrações externas

Fichas LEDI de visita, cadastro e atividade coletiva (componente de Vínculo do cofinanciamento). Detalhe e fontes em
[`pesquisa/2026-10-05-mapa-de-integracoes.md`](../pesquisa/2026-10-05-mapa-de-integracoes.md).

## Depende de

Módulos 11, 16.

## Histórico

- 2026-10-05 — Registrado como `Stub` no portfólio do Ciclo 2 (brainstorm de
  novos módulos; decisão de suíte completa).
- 2026-10-23 — Dividido em núcleo e ferramenta em repositório próprio (ADR 0041).
