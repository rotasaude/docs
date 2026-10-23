# Módulo 24 — Atendimento domiciliar e coletivo

- **Estado:** Planejado (24a); 24b e 24c no fim do Ciclo 2
- **Tipo:** Ciclo 2

## Escopo (intenção)

Visita do ACS/eSF, cadastro individual e domiciliar, atividade coletiva; app offline para tablet (11 de 15 editais).

**Divisão (ADR 0041), decidida em 2026-10-23:**

| Entrega | Onde | O que entra | Quando |
|---|---|---|---|
| 24a — Cadastro individual e domiciliar | api, dashboard | Cadastro domiciliar (imóvel, moradia, família, responsável) e individual pela web, aviso de cadastro com mais de 24 meses, fichas LEDI de cadastro (componente de Vínculo) | Agora |
| 24b — Visita e atividade coletiva | api, dashboard, wpda | Ficha de visita domiciliar e de atividade coletiva, LEDI, visita como fonte das linhas de cuidado (23) e da linha do tempo (20), contrato de sincronização do app | No fim do Ciclo 2 |
| 24c — App de campo | `rotasaude/campo` | App offline para tablet (React Native + Expo): cadastro, visita e atividade coletiva no campo, sincronização com conflito | Depois do 24b |

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0042 | Cadastro territorial (24a): microárea, domicílio, família, cadastro individual, paciente sem CPF, fichas LEDI, equipe pelo cadastro |
| 0041 | Divisão em núcleo e app de campo |
| 0023 | Território (bairros, cobertura) |
| 0028 | Modo de prontuário e LEDI |

## Funcionalidades planejadas (24a)

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-24.1 | Interruptor e microáreas | api, dashboard, maintenance | 0042 |
| F-24.2 | ACS e agente de endemias sem conselho | api, dashboard | 0042, 0021 |
| F-24.3 | Paciente sem CPF e possíveis duplicados | api, dashboard | 0042 |
| F-24.4 | Cadastro domiciliar e família | api, dashboard | 0042 |
| F-24.5 | Cadastro individual, recusa e saída | api, dashboard | 0042 |
| F-24.6 | Validade de 24 meses e quem falta cadastrar | api, dashboard | 0042 |
| F-24.7 | Fichas LEDI de cadastro | api | 0042, 0028 |
| F-24.8 | Equipe e óbito pelo cadastro | api | 0042, 0040 |

## Critério de fechamento (24a)

- F-24.1 a F-24.8 verificadas, com prova no navegador.

## Integrações externas

Fichas LEDI de visita, cadastro e atividade coletiva (componente de Vínculo do cofinanciamento). Detalhe e fontes em
[`pesquisa/2026-10-05-mapa-de-integracoes.md`](../pesquisa/2026-10-05-mapa-de-integracoes.md).

## Depende de

Módulos 10, 11, 16, 22 e 23.

## Histórico

- 2026-10-05 — Registrado como `Stub` no portfólio do Ciclo 2 (brainstorm de
  novos módulos; decisão de suíte completa).
- 2026-10-23 — Dividido em núcleo e ferramenta em repositório próprio (ADR 0041).
- 2026-10-23 — Dividido em 24a (agora), 24b e 24c (no fim do Ciclo 2).
- 2026-10-23 — Desenho do 24a aprovado e `Planejado` (ADR 0042, spec
  `superpowers/specs/2026-10-23-module-24a-territory-registry-design.md`).
