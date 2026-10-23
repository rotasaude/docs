# Módulo 24 — Atendimento domiciliar e coletivo

- **Estado:** Stub
- **Tipo:** Ciclo 2

## Escopo (intenção)

Visita do ACS/eSF, cadastro individual e domiciliar, atividade coletiva; app offline para tablet (11 de 15 editais).

**Divisão (ADR 0041), decidida em 2026-10-23:**

| Entrega | Onde | O que entra | Quando |
|---|---|---|---|
| 24a — Cadastro individual e domiciliar | api, dashboard | Cadastro domiciliar (imóvel, moradia, família, responsável) e individual pela web, aviso de cadastro com mais de 24 meses, fichas LEDI de cadastro (componente de Vínculo) | Agora |
| 24b — Visita e atividade coletiva | api, dashboard, wpda | Ficha de visita domiciliar e de atividade coletiva, LEDI, visita como fonte das linhas de cuidado (23) e da linha do tempo (20), contrato de sincronização do app | No fim do Ciclo 2 |
| 24c — App de campo | `rotasaude/campo` | App offline para tablet (React Native + Expo): cadastro, visita e atividade coletiva no campo, sincronização com conflito | Depois do 24b |

## Integrações externas

Fichas LEDI de visita, cadastro e atividade coletiva (componente de Vínculo do cofinanciamento). Detalhe e fontes em
[`pesquisa/2026-10-05-mapa-de-integracoes.md`](../pesquisa/2026-10-05-mapa-de-integracoes.md).

## Depende de

Módulos 11, 16.

## Histórico

- 2026-10-05 — Registrado como `Stub` no portfólio do Ciclo 2 (brainstorm de
  novos módulos; decisão de suíte completa).
- 2026-10-23 — Dividido em núcleo e ferramenta em repositório próprio (ADR 0041).
- 2026-10-23 — Dividido em 24a (agora), 24b e 24c (no fim do Ciclo 2).
