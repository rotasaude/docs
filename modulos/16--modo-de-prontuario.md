# Módulo 16 — Modo de prontuário e exportação

- **Estado:** Planejado
- **Tipo:** Ciclo 2

## Escopo

Prepara o Rota Saúde para ser o prontuário de uma cidade, ou conviver com o
e-SUS PEC dela, e entrega a produção da APS pelo caminho documentado para
terceiros: fichas LEDI enviadas à instalação do PEC da cidade, que repassa ao
SIAPS (ADR 0028).

**Planejado (F-16.1 a F-16.8):**
- interruptores de funcionalidade por cidade (`city_features`), escritos só
  pelo `maintenance` e lidos pela sessão;
- modo de prontuário (`off`/`integrated`/`record`), código IBGE e endereço do
  PEC, pelo operador no console;
- credenciais de integração por cidade, cifradas, cadastradas e testadas pelo
  `municipal_admin`;
- terminologias CID-10, CIAP-2 e SIGTAP versionadas no banco da plataforma;
- importação do CNES, equipes (INE) e casamento confirmado pela cidade;
- exportador LEDI: prova técnica contra PEC local, layout Thrift versionado,
  fila de saída e envio contínuo;
- painel da competência e alertas de prazo;
- consulta ao CADSUS na validação presencial.

**Fora, por enquanto:** fichas reais (módulos 18, 19, 22, 24), RNDS,
`maintenance` em produção, CBO nas terminologias, busca de terminologia nas
telas.

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0028 | Modo de prontuário, interruptores, credenciais por cidade, terminologias, CNES, exportação LEDI, CADSUS |
| 0007 | Chave de cifra por cidade (credenciais, fila, CNS) |
| 0020 | Banco por cidade e banco da plataforma |
| 0021 | Profissionais (CNS, vínculo, CBO) |
| 0027 | Perfil do par conferido pelo CADSUS |

## Superfícies

| Superfície | O que aparece |
|---|---|
| `contracts` | `session-v1.1.0` (`features`) |
| `api` | `city_features`, `cities.record_mode/pec_url/ibge_code`, `integration_credentials`, terminologias, retratos CNES, `health_teams`, `ledi_outbox`, `Ledi::DeliverJob`, `Cadsus::Client` |
| `admin` | Modo, IBGE e PEC por cidade; produção de todas as cidades; alerta de SIGTAP |
| `dashboard` | Integrações, CNES, Produção e-SUS, CADSUS no balcão |
| `maintenance` | Interruptores por cidade |
| `wpda` | — |

## Pré-requisitos

- Módulos 09 (unidades), 10 (profissionais) e 15 (perfil do par).
- Por cidade: instalação do PEC com HTTPS e credencial de integração; base do
  CNES e SIGTAP importadas; autorização do DATASUS para o CADSUS.

## Funcionalidades planejadas

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-16.1 | Interruptores de funcionalidade por cidade, escritos pelo maintenance | api, maintenance, contracts, dashboard, admin | 0028 |
| F-16.2 | Modo de prontuário, código IBGE e endereço do PEC por cidade | api, admin | 0028 |
| F-16.3 | Credenciais de integração por cidade, cifradas, com teste de conexão | api, dashboard | 0028, 0007 |
| F-16.4 | Terminologias CID-10, CIAP-2 e SIGTAP versionadas na plataforma | api | 0028 |
| F-16.5 | Importação do CNES, equipes e casamento confirmado pela cidade | api, dashboard | 0028, 0021 |
| F-16.6 | Exportador LEDI: prova técnica, layout versionado, fila e envio contínuo | api | 0028 |
| F-16.7 | Painel da competência e alertas de prazo | api, dashboard, admin | 0028 |
| F-16.8 | Consulta ao CADSUS na validação presencial | api, dashboard | 0028, 0027 |

## Riscos herdados

- Serialização Thrift em Ruby sem exemplo oficial (prova técnica primeiro).
- Esteira de versões do LEDI (2–6 semanas; versão > 12 meses invalidada).
- PEC da cidade como porta única para o SIAPS.
- `maintenance` só em dev/staging: em produção tudo desligado até ADR próprio.

## Critério de fechamento do módulo

- F-16.1 a F-16.8 verificadas.
- Prova técnica registrada (ficha sintética aceita pelo PEC local).
- Suíte de invariante (`spec/invariants/record_mode_invariants_spec.rb`) com os
  invariantes do ADR 0028.
- Runbook do operador: importação de SIGTAP e CNES, configuração de cidade.

## Histórico

- 2026-10-05 — Registrado como `Stub` no portfólio do Ciclo 2.
- 2026-10-05 — Escopo decidido com o usuário; ADR 0028 e spec
  `superpowers/specs/2026-10-05-module-16-record-mode-and-export-design.md`.
  F-16.1 a F-16.8 criados (board #1, `Not Started`). Módulo passa a
  `Planejado`.
