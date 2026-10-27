# Módulo 24 — Atendimento domiciliar e coletivo

- **Estado:** 24a Entregue (2026-10-27; F-24.1–F-24.6 e F-24.8 `Verified`; F-24.7 `Done` até o SIAPS aceitar fichas reais); 24b e 24c no fim do Ciclo 2
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
- 2026-10-27 — 24a **implementado, publicado e Entregue** (ver "Entrega do 24a"). `Fechado` depois do gate do SIAPS.
