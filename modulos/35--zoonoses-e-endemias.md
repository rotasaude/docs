# Módulo 35 — Zoonoses e endemias

- **Estado:** `Entregue` (35a em 2026-11-23; 35b em 2026-11-25). ADR 0054: 35a (campo de endemias) primeiro, 35b (centro de zoonoses) depois
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

## Entrega do 35a (2026-11-23)

Contrato entre apps: [`superpowers/plans/2026-11-16-module-35a-vector-control-contracts.md`](../superpowers/plans/2026-11-16-module-35a-vector-control-contracts.md) (§1–§19, decisões Q1–Q63; Q41 retirada). Planos por repo no mesmo diretório (`-contracts-repo`, `-api`, `-dashboard`, `-maintenance`, `-campo`). Execução na sessão "Módulo 35a", uma tarefa por subagente.

| Repo | Commit | Observação |
|---|---|---|
| `contracts` | `c022e80`, `6ce25c2` | tag `field-sync-v1.3.0` (entidades de controle vetorial, contracts#2 em parte); `6ce25c2` só acrescenta a linha da Q33 no README |
| `api` | `ab32775c`, `85dc80ee` | migrações de cidade `20261116900001`–`3` (irreversíveis); suíte completa 12731/0 em `ead7f5a5`; correções pós-merge Q31–Q40 e o revert da Q41 em `85dc80ee`, suíte 12781/1 (falha conhecida de outro módulo, api#180, aceita pelo usuário) |
| `dashboard` | `2e3a2a9` | 47 commits; 592 arquivos / 7458 testes |
| `maintenance` | `482e53b` | interruptor de vigilância ambiental por cidade; 697 testes |
| `campo` | `dbc54b3` | 37 commits sobre `ef51ad5`; 1457 testes em 96 suítes; Maestro 15/15 mais os 11 do 24c |
| `docs` | contrato Q1–Q63 | |

**Prova no navegador (2026-11-19, só desenvolvimento):** dashboard e maintenance contra o api de dev de Curitiba, com os consertos achados na prova (diálogo com Esc e foco, mapa centrado na área; `2e3a2a9`). A coleta da ovitrampa, que só abria em 21/11, foi provada depois, pelo app e pela contagem no dashboard.

**Prova no emulador (2026-11-23, só desenvolvimento):** a ordem da Q6(4) (interruptor desligado, pareamento do app do 35a, interruptor religado e `reset: true`), a folha do agente igual à do api, e os casos sem sinal: boletim com amostra, LIRAa num levantamento fictício do dev (Q60), consumo com falta de estoque, coleta e reinstalação de ovitrampa com contagem no dashboard, instalação, ponto estratégico cadastrado sem sinal (Q59) e fila guardada com o interruptor desligado. Achado da prova corrigido antes do merge: o imóvel do LIRAa voltava à folha até a rodada seguinte (Q61, `e8bb483`, coberto por spec). A revisão final da branch do campo trouxe a Q62 (`dbc54b3`).

**Board #1:** F-35.1 e F-35.3..10 `Verified`; F-35.2 `Done` até a ficha 8 do agente de endemias ser aceita no PEC real.

**Decisões tomadas na execução** (contrato): LIRAa pelo manual de 2013, com o modo de meia amostra (Q16, Q17, decisão do usuário); estoque de endemias sem a farmácia (Q21); concorrência otimista em parâmetros, cadências, correções e levantamento (Q34–Q37); recomeço sem cursor como reset parcial (Q42); visita de imóvel sem morador pelo detalhe (Q59); horizonte do download no app (Q61); boletim à espera de etiquetas conta como na fila (Q62).

**Pendências (board #2):** api#178 (desempenho do 35a em cidade grande: LIRAa, bloqueios e mapa; Média, antes do go-live), api#179 (achar imóveis com quarteirão de outro bairro; Baixa), campo#13 (o reset do servidor apaga "Minhas visitas de hoje" e, com a Q63, solta o domicílio nascido no tablet; Baixa, do 24c). Mais api#180 (falso alarme no spec de sigilo da urgência, achado na suíte do 35a) e o comentário no contracts#2 (exemplos adiados para a próxima versão do `field-sync`).

**Gates de go-live:** diretriz vigente conferida com a vigilância ambiental de uma cidade piloto; ficha 8 do agente de endemias aceita no PEC real; todos os tablets da cidade no app do 35a antes de ligar a vigilância ambiental (Q6); os gates do app de campo do 24c.

Página de status: https://claude.ai/artifact/UBwAiqrD2NzsCiH7ny2epx

## Entrega do 35b (2026-11-25)

Contrato entre apps: [`superpowers/plans/2026-11-23-module-35b-zoonoses-contracts.md`](../superpowers/plans/2026-11-23-module-35b-zoonoses-contracts.md) (§1–§19, decisões Q1–Q43). Planos por repo no mesmo diretório (`-contracts-repo`, `-api`, `-dashboard`, `-wpda`, `-maintenance`, `-campo`). Execução na sessão "MVP Modulo 35b", uma tarefa por subagente.

| Repo | Commit | Observação |
|---|---|---|
| `contracts` | `a199ff0` | tag `field-sync-v1.4.0` (objeto `062bc57`): animal, vacinação e inquérito de leishmaniose no app; 267 exemplos |
| `api` | `42349541`, `ef0effb1`, `ca2a92f0` | 57 commits sobre `85dc80ee`; migrações de cidade a partir de `20261123900001` (irreversíveis); suíte completa 13262/1 em `a58b9e24`, e a falha (lista de alvos do `reencryption_job_spec`) foi corrigida só no spec em `fe8b4acd`; Q41 e Q43 depois, com os specs do escopo |
| `dashboard` | `51b1871` | 41 commits; 609 arquivos / 7750 testes; barra superior cabendo em 375 px (decisão do usuário) |
| `wpda` | `05a450f` | 8 commits: demanda de zoonoses na ouvidoria e a correção do anexo do 33 |
| `maintenance` | `14406bd` | 4 commits: interruptor de zoonoses por cidade, cabeçalho a 375 px e rótulos de todos os papéis |
| `campo` | `9b2a583` | 19 commits sobre `dbc54b3`; 1751 testes em 105 suítes; Maestro 19/19; CI verde |
| `docs` | contrato Q1–Q43 | |

**Prova no navegador (2026-11-24, só desenvolvimento):** dashboard, wpda e maintenance contra o api da branch (`api-mod35b`, porta 3061), com o banco de dev migrado e semeado por autorização do usuário. No wpda, a denúncia anônima e a solicitação identificada chegaram como demanda de zoonoses, e a lista do cidadão não mostra o encaminhamento nem o endereço (Q13). No maintenance, o interruptor muda o `public_config` do wpda.

**Prova no emulador (2026-11-24, só desenvolvimento):** a ordem da Q24 (zoonoses desligado, pareamento, religado e `reset: true`), e os casos sem sinal:
- tutor novo no mesmo lote (Q29);
- vacina, com a recusa `already_vaccinated_in_campaign` para o animal já vacinado na campanha;
- coleta de leishmaniose com a etiqueta;
- coleira com falta de estoque registrada;
- coleiras a trocar;
- fila guardada com o interruptor desligado.

A revisão final do campo corrigiu o tutor recusado no mesmo lote, o "Reenviar" que sobrescreveria a estação e a leitura de "Coleiras a trocar".

**Board #1:** F-35.11..18 `Verified`.

**Decisões tomadas na execução** (contrato): `zoonoses_team` em PROFILE_ROLES e LINK_ROLES (Q30); transições do animal, com eutanásia e saída por óbito só de `active` ou `missing` (Q31); prevalência da leishmaniose por cão (Q36); etiqueta LV com 4 a 6 dígitos (Q36); tutor entre lotes (Q37); peças do 35a abertas por `supplies_usable?` (Q39); unidades e animal nas estadas (Q41); unidades de vacinação em `/zoonoses/domains` (Q43).

**Pendências (board #2):** dashboard#31 (a tela Animais busca só pelo CPF ou CNS do tutor ou de um morador, e o animal de domicílio sem morador documentado não é achado; Média, antes do go-live).

**Gates de go-live:** diretriz vigente conferida com a vigilância ambiental de uma cidade piloto; formato da planilha estadual da campanha antirrábica da UF piloto; todos os tablets da cidade no app do 35b antes de ligar as zoonoses (Q24); os gates do 35a e do app de campo do 24c.

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
- 2026-11-23 — 35a implementado, provado e publicado nos cinco repos (ver "Entrega do 35a"); módulo `Em andamento` até o 35b.
- 2026-11-25 — 35b implementado, provado e publicado nos seis repos (ver "Entrega do 35b"); módulo `Entregue`.
