# Módulo 23 — Linhas de cuidado

- **Estado:** Planejado
- **Tipo:** Ciclo 2

## Escopo

Linhas de cuidado (ADR 0040), só em cidades em modo registro: seis linhas da
plataforma com as boas práticas dos indicadores do componente de qualidade da APS
(C2 criança, C3 gestante, C4 diabetes, C5 hipertensão, C6 pessoa idosa, C7
mulher) e linhas próprias da cidade; sugestão de inscrição por problema ativo,
idade e sexo e condição declarada pelo cidadão — inclusive cidadão validado que
nunca consultou —, sempre confirmada por médico ou enfermeiro; boas práticas
avaliadas na leitura sobre consulta, escuta, exames e procedimentos; painel de
busca ativa por equipe; prévia de C2–C7 sobre a população elegível, sempre
rotulada como estimativa; "Meu acompanhamento" e condição declarada no wpda;
aviso de triagem periódica vencida (pendência do módulo 15).

**Fora, por enquanto:** visita do ACS (módulo 24) e vacinas (módulo 29) como
fontes; modo integrado; mensagem automática de busca ativa.

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0040 | Linhas, inscrição, acompanhamento, busca ativa, prévia |
| 0031 | Lista de problemas e regras de leitura |
| 0027 | Ofertas com intervalo de repetição (triagem vencida) |

## Funcionalidades planejadas

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-23.1 | Interruptor e catálogo-base das linhas | api, maintenance | 0040, 0028 |
| F-23.2 | Linhas da cidade (adoção e linhas próprias) | api, dashboard | 0040 |
| F-23.3 | Sugestão e inscrição confirmada | api, dashboard | 0040 |
| F-23.4 | Acompanhamento das boas práticas | api | 0040 |
| F-23.5 | Painel da equipe e busca ativa | api, dashboard | 0040, 0029 |
| F-23.6 | Prévia dos indicadores C2–C7 | api, dashboard | 0040 |
| F-23.7 | Condição declarada e "Meu acompanhamento" no wpda | api, wpda | 0040, 0017 |
| F-23.8 | Aviso de triagem periódica vencida | api, wpda | 0040, 0027 |

## Critério de fechamento

- F-23.1 a F-23.8 verificadas, com prova no navegador.

## Integrações externas

Indicadores C2–C7 do cofinanciamento da APS (notas metodológicas) definem as boas
práticas; o valor oficial é do SIAPS. Detalhe em
[`pesquisa/2026-10-05-integracoes/frente-1-aps-sisab.md`](../pesquisa/2026-10-05-integracoes/frente-1-aps-sisab.md).

## Depende de

Módulos 15, 17, 18, 19, 21 e 22.

## Histórico

- 2026-10-05 — Registrado como `Stub` no portfólio do Ciclo 2 (brainstorm de
  novos módulos; decisão de suíte completa).
- 2026-10-22 — Desenho aprovado e `Planejado` (ADR 0040, spec
  `superpowers/specs/2026-10-22-module-23-care-lines-design.md`).
