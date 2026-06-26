# Módulo 03 — Triagem

- **Estado:** Planejado
- **Tipo:** MVP

## Escopo

Motor de protocolos (fluxo, classificação, scoring), cadastro de protocolos
com workflow de quatro olhos, snapshot da triagem. Inclui a explicabilidade
da decisão. O motor é puro e determinístico; toda lógica clínica vive aqui.

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0009 | Protocolo como dado declarativo, interpretador puro, linguagem de condição, gate (JSON Schema + linter de grafo); classificação: `priority_when`, `Outcome`, trail de explicabilidade; cadastro: storage **per-cidade**, **dois eixos** no campo `status` da definição — lifecycle de autoria (`draft → in_review → published`) e vigência (`published → active`, `published` ≠ `active`), uma `active` por `(municipality_id, protocol_key)`; comandos `PublishProtocol` / `ActivateProtocolVersion` / `RetireProtocolVersion`; RBAC, preview; modelos de scoring (`weighted` e `decision_table`), tiers do sistema (low/medium/high) |

## Superfícies

| Superfície | O que aparece |
|---|---|
| `api` | `Protocol`, `step`, `evaluate`, `Protocols.current(municipality_id)`, `CompleteTriage`, gate de validação, eventos `protocol.*` |
| `admin` | — (autoria por cidade é da cidade; admin pode revisar como `platform_operator` backstop, ver módulo 06) |
| `dashboard` | Lista de protocolos por cidade, autoria (`author`/`publisher`), preview ao vivo, publicação com step-up MFA, lista de triagens (referências), inspeção de trail por triagem |
| `wpda` | Resultado da triagem para o paciente (tier, recomendação, sem dado clínico de terceiros) |

## Funcionalidades planejadas

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-03.1 | Interpretador puro (`step`) | api | 0009 |
| F-03.2 | Linguagem de condição (`eq`, `in`, `gt`, `lt`, `all`, `any`, `not`) | api | 0009 |
| F-03.3 | Mapeamento tipo de pergunta → elemento WhatsApp | api | 0009 |
| F-03.4 | `evaluate` modo `weighted` | api | 0009 |
| F-03.5 | `evaluate` modo `decision_table` | api | 0009 |
| F-03.6 | `priority_when` (independente do modo) | api | 0009 |
| F-03.7 | Trail estruturado (`scored`, `rule_matched`, `priority_rule`, `tier_assigned`) | api | 0009 |
| F-03.8 | Storage `protocols` + workflow draft/in_review/published/retired | api | 0009 |
| F-03.9 | Gate server-side (schema + linter de grafo + validações de scoring) | api | 0009 |
| F-03.10 | RBAC `protocol_author` / `protocol_publisher` | api, dashboard | 0009, 0012 |
| F-03.11 | Step-up MFA na publicação | api, dashboard | 0009, 0011 |
| F-03.12 | Editor JSON + preview ao vivo na cidade | dashboard | 0009 |
| F-03.13 | Publicação + auditoria (`protocol.published`) | api, dashboard | 0009, 0014 |
| F-03.14 | Aposentadoria (retired, não deleta) | api, dashboard | 0009 |
| F-03.15 | Triagens completam com `protocol_version` congelado | api | 0010, 0009 |
| F-03.16 | Inspeção de trail (referências, sem clínica crua) | dashboard | 0009, brief |
| F-03.17 | Resultado da triagem renderizado no wpda | wpda | 0009 |
| F-03.18 | Ativação de versão por cidade (`ActivateProtocolVersion`; `status` da definição per-cidade `published → active`; só `published`) | api, dashboard | 0009 |
| F-03.19 | Aposentadoria com guarda R4 (`RetireProtocolVersion` falha se alguma cidade ativa) | api, dashboard | 0009 |

## Dependências

- Módulo 02 (Conversação) — `handle_answer` consome `step`.
- Módulo 04 (Relatórios) — snapshot do `Outcome` é a entrada do relatório.
- Módulo 06 (Identidade/Acesso) — roles, MFA, backstop do `platform_operator`.
- Módulo 07 (LGPD/Auditoria) — eventos `protocol.*` via `DomainEvents`.

## Riscos herdados

- **Clínica:** linter de grafo ≠ correção clínica. Protocolo
  estruturalmente válido pode classificar errado.
- **Clínica:** miscalibração de scoring é inobservável — não há teste
  de regressão de casos clínicos.
- **Operacional:** quatro olhos colapsa em município pequeno (ADR 0012). Fallback
  via `platform_operator` backstop está desenhado mas exige operador
  disponível.
- **Operacional:** editor JSON pode reintroduzir "mudança depende de dev"
  (ADR 0009). Construtor gráfico fora de escopo do MVP.

## Critério de fechamento do módulo

- F-03.1 a F-03.19 verificadas.
- Suíte de invariante: pureza (mesmas respostas + mesma versão → mesmo
  `Outcome`); imutabilidade por versão; gate barra publicação inválida;
  step-up MFA obrigatório na publicação. Mais as 4 invariantes de lifecycle
  (ADR 0009): **INV-protocol-1** (ponteiro só aponta `published`),
  **INV-protocol-2** (uma ativa por `(municipality_id, protocol_key)`),
  **INV-protocol-3** (triagem termina na versão em que começou),
  **INV-protocol-4** (`retired` sem cidade ativa).
- Suíte de regressão clínica: bateria de casos conhecidos por protocolo,
  com `Outcome` esperado por versão.

## Histórico
