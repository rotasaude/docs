# Módulo 03 — Triagem

- **Estado:** Fechado
- **Tipo:** MVP

## Escopo

Motor de protocolos (fluxo, classificação, scoring), cadastro de protocolos
por cidade com ciclo de vida assinado e explicabilidade da decisão. O motor é
puro e determinístico: toda a lógica clínica vive aqui, como dado declarativo
interpretado por `Protocols::Protocol` (`step` e `evaluate`), sem banco,
relógio, acaso ou ambiente.

**Cadastro por cidade:** cada cidade guarda os próprios protocolos no próprio
banco (ADR 0020), em `protocol_definitions`, com unicidade por `(name,
version)` e uma única versão `active` por `name`. O `status` tem cinco valores:
`draft → in_review → published → active`, e `retired`. Publicar e ativar
exigem **duas assinaturas** de revisores da cidade sobre o mesmo conteúdo
(ADR 0016). As assinaturas, contribuições e ativações ficam em três tabelas
só de acréscimo. O mantenedor da API de manutenção pode executar publicar e
ativar quando as assinaturas da cidade já existem, mas nunca assina.

**Canal do cidadão:** desde 2026-09-27 o WhatsApp está descontinuado, e o
wpda (canal web, ADR 0017) conduz a triagem. `Citizens::SubmitAnswer` chama
`CompleteTriage`, que avança a triagem no motor. A F-03.3 (elemento de
WhatsApp por tipo de pergunta) continua no código, verificada, mas é do canal
descontinuado.

**Fora por enquanto:** construtor gráfico de protocolo (a autoria é o editor
JSON com preview) e suíte de regressão clínica além do protocolo semeado.

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0009 | Protocolo como dado declarativo, interpretador puro, linguagem de condição, gate (JSON Schema + linter de grafo), `priority_when`, `Outcome` com `explanation`, modelos de scoring `weighted` e `decision_table`, invariantes INV-protocol-1 a 4, versão publicada imutável |
| 0016 | Duas assinaturas de revisores por publicação e por ativação; papel `protocol_reviewer`; mantenedor nunca assina; reversão de emergência |
| 0010 | A triagem e o relatório apontam para a versão exata usada (`protocol_definition_id`) |
| 0011 | Step-up MFA nos atos do ciclo de vida |
| 0020 | Banco por cidade: o cadastro não tem coluna de município |

## Superfícies

| Superfície | O que aparece |
|---|---|
| `api` | `Protocols::Protocol` (`step`, `evaluate`), `Outcome#explanation`, `Protocols::Gate`, `Protocols.current(name:)` e `Protocols.fetch`, `StartTriage` e `CompleteTriage`, comandos `Protocols::SaveDraft`, `SubmitForReview`, `Publish`, `Activate`, `Retire` e `RevertActivation`, `ProtocolPolicy`, trigger `protocol_definitions_guard`, eventos `protocol.*` |
| `admin` | Lista de protocolos por cidade, só leitura (KPI de publicados) |
| `dashboard` | Editor JSON com preview ao vivo, ciclo assinado (assinar, publicar, ativar, aposentar, reverter) com step-up, painel de Classificação com o drawer de trail por triagem |
| `wpda` | Triagem em etapas e resultado (tier, prioridade, recomendação), sem nenhuma resposta no link público |
| `maintenance` | Publicar, ativar e reverter pela API GraphQL de manutenção, com as assinaturas da cidade |

## Funcionalidades planejadas

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-03.1 | Interpretador puro (`step`) | api | 0009 |
| F-03.2 | Linguagem de condição (`eq`, `in`, `gt`, `lt`, `all`, `any`, `not`) | api | 0009 |
| F-03.3 | Mapeamento tipo de pergunta → elemento WhatsApp (canal descontinuado) | api | 0009 |
| F-03.4 | `evaluate` modo `weighted` | api | 0009 |
| F-03.5 | `evaluate` modo `decision_table` | api | 0009 |
| F-03.6 | `priority_when` (independente do modo) | api | 0009 |
| F-03.7 | Trail estruturado (`scored`, `rule_matched`, `priority_rule`, `tier_assigned`) em `Outcome#explanation` | api | 0009 |
| F-03.8 | Storage `protocol_definitions` + workflow draft/in_review/published/active/retired | api | 0009, 0016 |
| F-03.9 | Gate server-side (schema + linter de grafo + validações de scoring) | api | 0009 |
| F-03.10 | RBAC `protocol_author` / `protocol_reviewer` / `protocol_publisher` | api, dashboard | 0009, 0012, 0016 |
| F-03.11 | Step-up MFA na publicação e nos demais atos do ciclo | api, dashboard | 0009, 0011 |
| F-03.12 | Editor JSON + preview ao vivo na cidade | dashboard | 0009 |
| F-03.13 | Publicação com duas assinaturas + auditoria (`protocol.published`) | api, dashboard | 0009, 0014, 0016 |
| F-03.14 | Aposentadoria (retired, não deleta; trigger no banco da cidade) | api, dashboard | 0009 |
| F-03.15 | Triagens completam na versão congelada (`protocol_definition_id`) | api | 0010, 0009 |
| F-03.16 | Inspeção de trail (referências, sem clínica crua) | api, dashboard | 0009 |
| F-03.17 | Resultado da triagem renderizado no wpda | api, wpda | 0009, 0010 |
| F-03.18 | Ativação de versão na cidade (`Protocols::Activate`; só `published`; duas assinaturas) | api, dashboard | 0009, 0016 |
| F-03.19 | Aposentadoria com guarda R4 (`Protocols::Retire` recusa a versão `active`) | api, dashboard | 0009 |

## Dependências

- Módulo 02 (Conversação) — `Citizens::SubmitAnswer` e `CompleteTriage`
  consomem o motor; a triagem só avança com a conversa consentida.
- Módulo 04 (Relatórios) — o snapshot do relatório congela tier, prioridade e
  recomendação da versão usada.
- Módulo 06 (Identidade/Acesso) — papéis, MFA e step-up.
- Módulo 07 (LGPD/Auditoria) — eventos `protocol.*` via `DomainEvents`.

## Riscos que continuam

- **Clínica:** linter de grafo ≠ correção clínica. A regressão clínica cobre
  só o protocolo semeado (`triage-respiratoria` v1); protocolo novo de cidade
  só tem casos se quem publica os escrever.
- **Clínica:** os tiers são texto livre por protocolo (`baixa`/`alta` no
  semeado). O painel de Classificação conta `low`/`medium`/`high` e compara
  `priority` com `true`, então com dados reais ele fica zerado. O card
  `rotasaude/api#21` (módulo 05) cuida disso.
- **LGPD:** o evento `triage.completed` ainda carrega o `trail` com as
  respostas em `domain_events` (payload do `Outcome#to_h`). O link público
  deixou de expor as respostas, mas o evento interno continua com elas.
- **Operacional:** duas assinaturas colapsam em município pequeno (ADR 0016);
  o mantenedor executa, mas não substitui revisor.
- **Produto:** o menu "Editor de protocolo" aparece para qualquer papel; só a
  API recusa (403).
- **Legado:** triagens concluídas antes da `explanation` mostram o trail
  vazio no drawer; snapshots antigos ainda guardam `summary` no banco, mas o
  `/r/:token` não o devolve mais.
- **Rollout:** o trigger `protocol_definitions_guard` chega às cidades
  existentes pela migração de cidade `20260927100001`; rode
  `city:migrate:all` com a imagem nova antes de cortar o tráfego. Deploy do
  api antes do wpda.

## Critério de fechamento do módulo

Cumprido em 2026-09-27.

- F-03.1 a F-03.19 verificadas no board.
- Suíte de invariante do motor (`api`
  `spec/invariants/protocol_engine_purity_spec.rb`): o núcleo não referencia
  banco, relógio, acaso, IO nem ambiente; mesmas respostas + mesma versão →
  mesmo `Outcome`, em builds novos e em qualquer data; `step` e `evaluate` não
  tocam o banco; valores congelados.
- Imutabilidade por versão (`spec/models/protocol_definition_guard_spec.rb`):
  nenhuma versão é apagada; conteúdo congelado depois de `published`; status
  só anda para frente.
- Gate barra publicação inválida (`spec/commands/protocols_publish_gate_spec.rb`)
  e step-up obrigatório nos atos (`spec/requests/protocol_lifecycle_spec.rb`,
  `spec/controllers/publications_controller_spec.rb`).
- INV-protocol-1 a 4 (`spec/commands/protocols_lifecycle_spec.rb`), com a
  INV-3 provando o `Outcome` da v1 numa triagem que termina depois de ativar a v2.
- Regressão clínica por versão (`spec/invariants/clinical_regression_spec.rb`)
  para o protocolo semeado em toda cidade nova.

## Histórico

_(funcionalidades concluídas aparecem aqui em ordem cronológica reversa)_

- **2026-09-27** — Módulo **Fechado**. F-03.1 a F-03.19 `Verified` no board.
  O dossiê achou cinco `Done` que não se sustentavam; todos consertados (api
  63f8568, merge de 42f497e, 13367c2, 9c47e95 e 52abdab; dashboard ffb9061;
  wpda 512a90b e a105b14):
  - F-03.7 e F-03.16: o trail estruturado nunca era gravado (só o seed de
    demo tinha os eventos). O motor passa a congelar `Outcome#explanation`,
    e o drawer lê dali.
  - F-03.17: `/r/:token`, público por 30 dias, devolvia as respostas cruas em
    `summary`. O job parou de gravá-las e o endpoint parou de servi-las.
  - F-03.14: `protocol_definitions` não tinha proteção contra `DELETE` nem
    contra edição depois de publicada. Ganhou o trigger
    `protocol_definitions_guard`.
  - F-03.1: sem spec de `step` nem de pureza. Ganhou a suíte de invariante do
    motor e a regressão clínica.
  O doc passou a refletir os ADRs 0016 e 0020 (`protocol_reviewer`, duas
  assinaturas, `name` no lugar de `municipality_id`/`protocol_key`, status
  `active`).
- **2026-09-26** — Dossiê de verificação: 9 F-IDs sobem para `Verified`
  (F-03.4, .5, .8, .10, .11, .13, .15, .18, .19); 287 exemplos de protocolo
  verdes.
