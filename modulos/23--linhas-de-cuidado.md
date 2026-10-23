# Módulo 23 — Linhas de cuidado

- **Estado:** Fechado (2026-10-23; F-23.1–F-23.8 `Verified`; comparação da prévia F-23.6 com o SIAPS virou pendência de go-live, api#97)
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

## Entrega (2026-10-23)

| Repo | Commit | Observação |
|---|---|---|
| `api` | `9d35b6e` | 48 commits; suíte completa 6851/0 |
| `dashboard` | `a9b2356` | 29 commits; 3619/3619 |
| `wpda` | `7c46249` | 16 commits; 747/747 e e2e de celular |
| `maintenance` | `2df03d3` | 8 commits; interruptor `care_lines` e catálogo-base versionado |
| `catalog` | `7b9400c` | F-23.6 "em validação" (rotasaude/catalog#30) |
| `docs` | `caa254c`, `b85008f` | prova técnica e runbook |

Migração de plataforma (catálogo-base) e de cidade `20261022900002`/`20261022900003`
(a segunda em `appointment_requests` do módulo 17, irreversível; as duas no mesmo
deploy). Contrato com acréscimos §11 (L1–L13, T1–T7) e §12 (P1–P22). Runbook
[`operacao/rollout-linhas-de-cuidado.md`](../operacao/rollout-linhas-de-cuidado.md).

**Prova técnica:** notas metodológicas C2–C7 e NT 8/2026 baixadas e conferidas por
SHA-256 ([`pesquisa/2026-10-22-linhas-de-cuidado-prova-tecnica.md`](../pesquisa/2026-10-22-linhas-de-cuidado-prova-tecnica.md));
C4 (71,9, Bom) e C7 (21,4, Regular) conferidas à mão.

**Prova no navegador (2026-10-23):** sugestão por problema aceita pela médica;
técnica sem "Aceitar"; busca ativa com HbA1c e pés vencidos e visita do ACS "não
acompanhada"; contato e "Agendar" (pedido à recepção, horário marcado pela técnica,
segundo "Agendar" recusado); gestante com DUM (vazia e futura recusadas); idosa
validada que nunca consultou inscrita; linha própria de saúde mental; prévia C4
71,9 rotulada como estimativa; cidadão declara condição (nenhuma URL a leva) e vê
"Meu acompanhamento"; triagem vencida com "Começar"; interruptor no maintenance;
versão nova da C4 publicada (permanente no dev). Fora da tela, cobertos por teste:
`duplicate_declared_condition` (step-up) e o check-in da consulta da linha.

**Decisões tomadas na execução** (contrato §12): aborto só exclui na gestação
atual; trilha por registro mostrado; "Agendar" reaproveita pedido aberto de
qualquer tipo e a saída por óbito ou mudança fecha o pedido da linha; prévia só com
coortes de janela fechada, peso de prática sem elegível redistribuído e óbito fora
do denominador; paciente sem consulta com equipe em 24 meses vai pela equipe do
território; sugestão bloqueada 365 dias após encerrar e para sempre após óbito;
DUM, notas, motivos e CPF só no corpo; painel só nas equipes do vínculo.

**Pendências abertas:** api#94 (status "cancelado pela unidade" no 17), api#95
(cifrar motivos em texto livre do 17, antes do go-live), api#96 (rubocop),
maintenance#1 (`env.test.ts`), api#97 (comparação com o SIAPS, antes do go-live).

**Gate de go-live:** comparar a prévia F-23.6 com uma extração oficial do SIAPS de
uma equipe piloto (api#97, board de pendências, antes do go-live).

Página de status: https://claude.ai/artifact/De7hkPWP8ZwAxTMVzbqnXe

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
- 2026-10-23 — **Implementado, publicado e Entregue** (ver "Entrega"). `Fechado` depois do gate do SIAPS.
- 2026-10-23 — **Verificado e Fechado** por decisão do usuário; a comparação com o SIAPS ficou como pendência de go-live (api#97).
