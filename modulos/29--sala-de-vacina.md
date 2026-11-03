# Módulo 29 — Sala de vacina

- **Estado:** Entregue
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

## Entrega (2026-11-03)

| Repo | Commit | Observação |
|---|---|---|
| `api` | `5a32ced0` | 51 commits; suíte completa 9538 exemplos, 1 falha fora do módulo (layout do PDF do 19c, tarefa à parte); migração de cidade `20261102900001`; fecha api#71 |
| `dashboard` | `022ccd3` | 30 commits; suíte 5176/0 |
| `wpda` | `44b2533` | 8 commits; suíte 807/0 + e2e em celulares |
| `maintenance` | `6b788fc` | interruptores `vaccination`, `rnds_immunization`, `rnds_mock`; suíte 602/0 |
| `docs` | contrato §15 V1–V29 | |

**Prova no navegador (2026-11-03, RNDS falsa em dev):** busca por CPF sem dado na URL; caderneta do bebê de 2 meses com aprazamento e atraso; dose com lote por FEFO e abertura de frasco; envio aceito com `rnds_id`; transcrição recusada pela RNDS com CPF mascarado e listada em Pendências; estoque por frasco; excursão de temperatura que bloqueia lotes; conduta lote a lote com step-up e descarte acima do limite aguardando aprovação (a aprovação pela 2ª pessoa não foi exercitada); resumo mensal fechando a conta; painel do gestor só com contagens; "Minhas vacinas" no wpda sem lote nem vacinador; interruptores e 403 `feature_disabled`. Achados corrigidos antes do merge (V29): caderneta do cidadão sem linhas calculada sob demanda; "Transcrita por"; Pendências relida ao abrir; lote bloqueado identificado; plural e validade do frasco em dias.

**Decisões tomadas na execução** (contrato §15 V17–V29): ordem das listas; `manufacturer_key` com código resolvido no envio; 108 grupos de atendimento; duplicados por POST; recusas da transcrição e cancelamento antes de tentativa nunca enviado; excursão fecha só com conduta em todo lote; colunas do resumo mensal; `go_live_on` limitado e catálogo em revisão como requisito; condição materna só de 10 a 49 anos; motivos de perda no catálogo; situação do cidadão sob demanda.

Página de status: https://claude.ai/artifact/MojHsDxjz2U4SGZhe3G1Mm

**Gates de go-live (a abrir no board #2):** perfil RIA-R 2.0 oficial e homologação na RNDS (extensões, BRDose, BRTipoDocumento, NamingSystems, REG-IMUNO, forma do cancelamento, default "1" da condição materna, transcrição sem via/local/grupo, aceite sem `Location`, executante da transcrição); autenticação (certificado A1 legado no OpenSSL 3, e-CNPJ/e-CPF, token por CNES, alerta de certificado TLS); códigos de fabricante nulos e hep B "9"×"09"; revisão de enfermagem do calendário (fronteiras, influenza do idoso, resgate, dose após `not_applicable`, D1 repetido, 2ª dose de rotavírus sem a 1ª); custo do `RefreshStatusesJob` numa base de 300 mil pacientes.

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
- 2026-11-03 — Executado nos quatro apps, provado no navegador, mergeado e publicado; estado `Entregue`.
