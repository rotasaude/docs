# Módulo 29 — Sala de vacina

- **Estado:** Planejado
- **Tipo:** Ciclo 2

## Escopo (intenção)

Registro da dose aplicada, estoque de imunobiológicos, caderneta. Lacuna presente nos 15 editais.

**Escopo (ADR 0047), decidido em 2026-11-02:** um módulo só, com o registro da dose (e transcrição de caderneta), o envio RIA-R 2.0 à RNDS em até 24 h (Portaria GM/MS 5.663/2024), o calendário curado com caderneta conferida, o estoque de imunobiológicos no razão do 27a (rede de frio municipal e salas, frasco multidose, perdas) e a cadeia de frio. Liga-se às campanhas (vacina em atraso, caderneta não conferida), às linhas de cuidado (marcos de vacina), à linha do tempo (api#71) e à vigilância (ESAVI). Fora: histórico nacional da RNDS, vacinação offline, SIES/SIPNI por API, ESAVI ao e-SUS Notifica.

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0047 | Sala de vacina (dose, RNDS, calendário, estoque, cadeia de frio) |
| 0028 | Credencial `rnds` e interruptores |
| 0045 | Razão de estoque reaproveitado |
| 0024 | Critérios novos das campanhas |
| 0040 | Marcos de vacina das linhas de cuidado |
| 0035 | Fonte `vaccination` da linha do tempo |

## Funcionalidades planejadas

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-29.1 | Interruptores, credencial RNDS, sala de vacina e vacinadores | api, dashboard, maintenance | 0047, 0028 |
| F-29.2 | Calendário, catálogo de imunobiológicos e caderneta conferida | api, dashboard | 0047 |
| F-29.3 | Registro da dose, transcrição e cancelamento | api, dashboard | 0047 |
| F-29.4 | Envio à RNDS (RIA-R 2.0) com guarda do retorno | api, dashboard | 0047, 0028 |
| F-29.5 | Estoque de imunobiológicos, frascos e perdas, resumo mensal | api, dashboard | 0047, 0045 |
| F-29.6 | Cadeia de frio | api, dashboard | 0047 |
| F-29.7 | Caderneta do cidadão e linha do tempo | api, wpda, dashboard | 0047, 0035 |
| F-29.8 | Linhas de cuidado, campanhas, ESAVI e painel | api, dashboard | 0047, 0040, 0024, 0043 |

## Critério de fechamento

- F-29.1 a F-29.8 verificadas, com prova no navegador (RNDS falsa em dev); envio aceito na homologação da RNDS é gate de go-live.

## Integrações externas

RNDS-RIA-R 2.0 obrigatório em até 24 h (Portaria 5.663/2024), com guarda do retorno. Detalhe e fontes em
[`pesquisa/2026-11-02-sala-de-vacina-29.md`](../pesquisa/2026-11-02-sala-de-vacina-29.md) e
[`pesquisa/2026-10-05-mapa-de-integracoes.md`](../pesquisa/2026-10-05-mapa-de-integracoes.md).

## Depende de

Módulos 16, 27a (razão de estoque), 12, 23, 20, 25.

## Documentos

- Spec: [`superpowers/specs/2026-11-02-module-29-vaccination-room-design.md`](../superpowers/specs/2026-11-02-module-29-vaccination-room-design.md)

## Histórico

- 2026-10-05 — Registrado como `Stub` no portfólio do Ciclo 2.
- 2026-11-02 — Desenho aprovado (ADR 0047); estado `Planejado`.
