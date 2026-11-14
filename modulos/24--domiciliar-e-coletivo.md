# Módulo 24 — Atendimento domiciliar e coletivo

- **Estado:** 24a Entregue (2026-10-27; F-24.1–F-24.6 e F-24.8 `Verified`; F-24.7 `Done` até o SIAPS aceitar fichas reais); 24b Entregue (2026-11-14; F-24.9, .10, .12, .14, .15, .16 e .17 `Verified`; F-24.11 e .13 `Done` até o PEC real e o SIAPS, api#165); 24c `Planejado` (2026-11-13, ADR 0053)
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
| 0053 | App de campo (24c): um tablet por profissional, tudo offline numa entrega, mescla campo a campo, mapa OpenStreetMap por município servido pelo Rota Saúde, revogação automática guarda o trabalho |
| 0052 | Visita domiciliar e atividade coletiva (24b): folha do dia como ficha LEDI tipo 8, atividade coletiva tipo 6 sem PSE, visita nas linhas de cuidado e na linha do tempo, "quem visitar", contrato de sincronização do app de campo |
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

## Funcionalidades planejadas (24b)

Spec: [`superpowers/specs/2026-11-12-module-24b-home-visit-collective-activity-design.md`](../superpowers/specs/2026-11-12-module-24b-home-visit-collective-activity-design.md) · pesquisa: [`pesquisa/2026-11-12-visita-e-coletiva-24b.md`](../pesquisa/2026-11-12-visita-e-coletiva-24b.md)

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-24.9 | Interruptores e quem registra (Tabelas 7 e 5 do LEDI) | api, dashboard, maintenance | 0052 |
| F-24.10 | Folha do dia e visita domiciliar pelo dashboard | api, dashboard | 0052, 0042 |
| F-24.11 | Ficha LEDI de visita domiciliar e territorial (tipo 8) | api | 0052, 0028 |
| F-24.12 | Atividade coletiva, sem PSE | api, dashboard | 0052 |
| F-24.13 | Ficha LEDI de atividade coletiva (tipo 6) | api | 0052, 0028 |
| F-24.14 | Visita nas linhas de cuidado e em "Meu acompanhamento" | api, dashboard, wpda | 0052, 0040 |
| F-24.15 | Visita na linha do tempo do profissional e do cidadão | api, dashboard, wpda | 0052, 0035 |
| F-24.16 | "Quem visitar" por microárea | api, dashboard | 0052, 0040, 0042 |
| F-24.17 | Contrato de sincronização do app de campo e lado do api (aparelho, envio idempotente, download por microárea) | contracts, api, dashboard | 0052, 0041 |

**Gates de go-live (24b):** prova no PEC real das fichas 8 e 6 (inclusive mais de uma ficha de visita do mesmo profissional no mesmo dia); SIAPS contando visita e atividade no Acompanhamento; regra da C5 conferida contra o relatório do SIAPS.

## Funcionalidades planejadas (24c)

Spec: [`superpowers/specs/2026-11-13-module-24c-field-app-design.md`](../superpowers/specs/2026-11-13-module-24c-field-app-design.md) · pesquisa: [`pesquisa/2026-11-13-app-de-campo-24c.md`](../pesquisa/2026-11-13-app-de-campo-24c.md)

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-24.18 | App de campo: cadastro do aparelho por QR, PIN, banco cifrado e proteção do tablet | campo, dashboard | 0053, 0041 |
| F-24.19 | Download do território e sincronização (fila, lotes, reset, remoções, 30 dias, revogação) | contracts, api, campo, dashboard | 0053, 0052 |
| F-24.20 | Mapa offline por município (geração, entrega, mapa no app e no dashboard) | contracts, api, campo, dashboard, maintenance | 0053 |
| F-24.21 | Cadastro domiciliar, família e individual offline com duplicado no tablet | contracts, api, campo | 0053, 0042 |
| F-24.22 | Conflito com mescla campo a campo | campo | 0053 |
| F-24.23 | Visita offline, "quem visitar" e atividade coletiva no app | campo | 0053, 0052 |
| F-24.24 | Coordenada do imóvel e da visita nas fichas LEDI 3 e 8 | contracts, api, campo | 0053, 0028 |
| F-24.25 | Build assinado, distribuição pelo Google Play gerenciado e runbook | campo, docs | 0053 |

**Gates de go-live (24c):** prova no PEC real das fichas 3 e 8 com coordenada; conta de desenvolvedor Google verificada; app privado publicado no Google Play gerenciado de uma cidade piloto; um dia de campo real sem sinal.

## Critério de fechamento (24a)

- F-24.1 a F-24.8 verificadas, com prova no navegador.

## Entrega do 24a (2026-10-27)

| Repo | Commit | Observação |
|---|---|---|
| `api` | `10c2ab9` | 63 commits; suíte completa 7245/0; migração de cidade `20261023900001` irreversível |
| `dashboard` | `8d388b8` | 45 commits; 4151/0 |
| `maintenance` | `eb0eeb0` | interruptor `territory_registry` (requer `clinical_record`, IBGE e UF) |
| `catalog` | PR rotasaude/catalog#31 | módulo 24 com F-24.1..F-24.8 (F-24.7 em validação) |
| `docs` | contrato §14 L1–L31 e §15 | runbook [`operacao/rollout-cadastro-territorial.md`](../operacao/rollout-cadastro-territorial.md) |

**Prova técnica:** regras e dicionários LEDI 8.7 das fichas de cadastro individual (tipo 2) e domiciliar (tipo 3) vendorizados com SHA-256 no api (`vendor/ledi/8.7.0/docs`) e publicados como §15 do contrato; traduções de raça/cor, etnia, UF e saída.

**Prova no navegador (2026-10-26/27):** microáreas; ACS sem conselho; território do ACS; domicílio novo e troca de tipo; edição simultânea; recém-nascido sem CPF; CPF de outro paciente; recusa; saída por óbito; duplicado com cadastro abre o existente; troca do responsável falecido; óbito encerra inscrições e dispensa sugestões das linhas de cuidado; validade por microárea e equipe; sem polling e sem rascunho; celular; interruptor no maintenance (Curitiba e Maringá).

**Decisões tomadas na execução** (contrato §14): paciente sem CPF e sem CNS grava e a ficha fica "não gerada"; nomes no conjunto oficial só na ficha; interruptor exige UF e IBGE; validade conta da última versão e 24 meses já vence; inscrição ativa mantém a equipe na própria linha; validação no posto só marca "revisar e salvar"; guarda de duplicado no api; troca de responsável por família nova; óbito do cadastro encerra inscrições.

**Pendências:** api#98 (situação de rua no BPA-I) e tipo de logradouro no BPA-I (faturamento); maintenance#1.

**Gates de go-live:** prova no PEC real das fichas 2 e 3; SIAPS aceitando cadastro sem CPF identificado por CNS.

Página de status: https://claude.ai/artifact/BWNV4wLVver5fWM4wfpCo6

## Entrega do 24b (2026-11-14)

| Repo | Commit | Observação |
|---|---|---|
| `contracts` | `673bb143` | tag `field-sync-v1.1.0` (v1.0.0 = `5493651`) |
| `api` | `7a7343bc` | suíte completa 12084/0 |
| `dashboard` | `ceb53aff` | 24b em `4f13f0e` + correção do `<form>` aninhado na busca de pessoa e no registro rápido |
| `wpda` | `cfa6675` | visita em "Meu histórico" (só o dia) e em "Meu acompanhamento" (`by_team`); frase de atraso da Q79 |
| `maintenance` | `39a729b3` | interruptores; `min_gap_days` e rótulo `acs_visit` no editor do catálogo-base |
| `catalog` | PR rotasaude/catalog#40 (`dfe4601`) | F-24.9..17; .11, .13 e .17 em validação |
| `docs` | contrato §16 Q1–Q79 | |

**Prova no navegador (2026-11-13/14):** ACS, enfermeira, TACS e admin no dashboard (folha do dia, visita, atividade coletiva, aparelhos de campo, Produção); catálogo-base C2–C6 publicado pelo maintenance e adotado em Curitiba; cidadã no wpda (histórico e acompanhamento, visita retirada oculta). A prova achou o `<form>` aninhado que o jsdom não reproduz (busca por documento e registro rápido sem efeito no Chromium), corrigido antes do Verified.

**Pendências (board #2):** api#162–#170, maintenance#2–#3, dashboard#29–#30, wpda#3, contracts#2 (entra na MINOR do 24c). Mais api#171 (CBOs 352210 e 515124 da Tabela 1 fora do `cbo_saude.yml`).

**Gates de go-live:** api#165 (PEC real das fichas 8 e 6, SIAPS contando, regra da C5 contra o relatório do SIAPS) e api#164 (Q67).

Página de status: https://claude.ai/artifact/K9ThEFh3xLtnUrTDaUsksj

## Integrações externas

Fichas LEDI de visita, cadastro e atividade coletiva (componente de Vínculo do cofinanciamento). Detalhe e fontes em
[`pesquisa/2026-10-05-mapa-de-integracoes.md`](../pesquisa/2026-10-05-mapa-de-integracoes.md).

## Depende de

Módulos 10, 11, 16, 20, 22 e 23.

## Histórico

- 2026-10-05 — Registrado como `Stub` no portfólio do Ciclo 2 (brainstorm de
  novos módulos; decisão de suíte completa).
- 2026-10-23 — Dividido em núcleo e ferramenta em repositório próprio (ADR 0041).
- 2026-10-23 — Dividido em 24a (agora), 24b e 24c (no fim do Ciclo 2).
- 2026-10-23 — Desenho do 24a aprovado e `Planejado` (ADR 0042, spec
  `superpowers/specs/2026-10-23-module-24a-territory-registry-design.md`).
- 2026-10-27 — 24a **implementado, publicado e Entregue** (ver "Entrega do 24a"). `Fechado` depois do gate do SIAPS.
- 2026-11-12 — Desenho do 24b aprovado e `Planejado` (ADR 0052, spec
  `superpowers/specs/2026-11-12-module-24b-home-visit-collective-activity-design.md`;
  decisões do usuário 1A–7A).
- 2026-11-13 — Desenho do 24c aprovado e `Planejado` (ADR 0053, spec
  `superpowers/specs/2026-11-13-module-24c-field-app-design.md`;
  decisões do usuário 1A, 2A, 3B, 4A, 5A, 6A).
- 2026-11-14 — 24b **implementado, publicado e Entregue** (ver "Entrega do 24b"). `Fechado` depois do api#165.
