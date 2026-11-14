# Módulo 35 — Zoonoses e endemias

- **Estado:** `Planejado` (2026-11-14, ADR 0054): 35a (campo de endemias) primeiro, 35b (centro de zoonoses) depois
- **Tipo:** Ciclo 2

## Escopo (intenção)

Vigilância ambiental do município para a prova de conceito dos editais (8 de 15 pedem): o trabalho de campo do agente de combate às endemias contra arboviroses (35a) e o centro de zoonoses (35b). Entra só o que os editais repetem.

**Fora:** integração com sistemas estaduais e exportação para o programa do LIRAa; Conta Ovos (depois); módulo de laboratório e GAL; sorteio de supervisão; cadastro imobiliário da prefeitura; prontuário veterinário completo; Wolbachia, EDL e TIE; envio automático ao Sinan; território próprio de endemias sem microárea (card para depois).

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| [0054](../adr/0054.md) | Boletim na visita, ciclos por risco, ovitrampas, LIRAa, bloqueio em quarteirões, estoque, centro de zoonoses |
| [0042](../adr/0042.md) | Cadastro territorial (imóvel, tutor) |
| [0052](../adr/0052.md) | Visita, folha do dia e sincronização |
| [0053](../adr/0053.md) | App de campo |
| [0043](../adr/0043.md) | Notificação (bloqueio, animal agressor) |
| [0051](../adr/0051.md) | Ouvidoria (demandas de zoonoses) |

## Funcionalidades planejadas (35a)

Spec: [`superpowers/specs/2026-11-14-module-35-zoonoses-endemics-design.md`](../superpowers/specs/2026-11-14-module-35-zoonoses-endemics-design.md) · pesquisa: [`pesquisa/2026-11-14-zoonoses-e-endemias-35.md`](../pesquisa/2026-11-14-zoonoses-e-endemias-35.md)

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-35.1 | Interruptor de vigilância ambiental, coordenação de endemias, quarteirão e risco do imóvel | api, dashboard, maintenance | 0054 |
| F-35.2 | Boletim de campo na visita do agente de endemias, com a ficha LEDI derivada | api, dashboard | 0054 |
| F-35.3 | Produtos, cargas e consumo pelo estoque | api, dashboard | 0054 |
| F-35.4 | Ciclos por risco, folha do agente e produção | api, dashboard | 0054 |
| F-35.5 | Ovitrampas: instalação, retirada, contagem, índices e mapa | api, dashboard | 0054 |
| F-35.6 | LIRAa: estratos, sorteio, índices e relatório | api, dashboard | 0054 |
| F-35.7 | Bloqueio de transmissão a partir da notificação | api, dashboard | 0054 |
| F-35.8 | Amostras com etiqueta e resultado lançado | api, dashboard | 0054 |
| F-35.9 | Relatórios de endemias | api, dashboard | 0054 |
| F-35.10 | App de campo: boletim, ovitrampa, LIRAa e bloqueio offline | contracts, api, campo | 0054 |

## Funcionalidades planejadas (35b)

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-35.11 | Interruptor de zoonoses, veterinário e cadastro do animal com tutor | api, dashboard, maintenance | 0054 |
| F-35.12 | Vacinação antirrábica: campanha, posto fixo, rotina e resultado semanal | api, dashboard | 0054 |
| F-35.13 | Demandas de zoonoses pela ouvidoria | api, dashboard, wpda | 0054 |
| F-35.14 | Observação do animal agressor ligada à notificação de raiva | api, dashboard | 0054 |
| F-35.15 | Leishmaniose canina: inquérito, sorologia e coleira | api, dashboard | 0054 |
| F-35.16 | Canil e gatil: entrada, saída, castração e eutanásia | api, dashboard | 0054 |
| F-35.17 | Relatórios de zoonoses | api, dashboard | 0054 |
| F-35.18 | App de campo: animal, vacinação e inquérito offline | contracts, api, campo | 0054 |

**Gates de go-live:** diretriz vigente conferida com a vigilância ambiental de uma cidade piloto; formato da planilha estadual da campanha antirrábica da UF piloto; ficha 8 do agente de endemias aceita no PEC real.

## Integrações externas

Nenhuma automática nesta versão: o Rota Saúde é o sistema municipal. Sinan por digitação (como no 25); Conta Ovos, LIRAa estadual e sistemas estaduais ficam para depois. Detalhe em [`pesquisa/2026-11-14-zoonoses-e-endemias-35.md`](../pesquisa/2026-11-14-zoonoses-e-endemias-35.md).

## Depende de

Módulos 11, 24 (24a, 24b; 24c para a parte do app), 25, 27a e 33.

## Histórico

- 2026-10-05 — Registrado como `Stub` no portfólio do Ciclo 2 (brainstorm de
  novos módulos; decisão de suíte completa).
- 2026-11-14 — Desenho aprovado e `Planejado` (ADR 0054, spec
  `superpowers/specs/2026-11-14-module-35-zoonoses-endemics-design.md`;
  decisões do usuário 1–7, com o B do cadastro).
