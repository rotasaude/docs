# Módulo 05 — Dashboard

- **Estado:** Fechado
- **Tipo:** MVP

## Escopo

O `dashboard` em si — painel operacional da cidade, tenant-scoped,
dono `municipal_admin` / `viewer`. Read-only na fase 1. Nove painéis
operacionais (overview + 8 módulos), seguindo o brief do dashboard.
Os painéis agregam ao vivo no banco da cidade (ADR 0022).

Este módulo é peculiar: é o **app inteiro**, não um corte de funcionalidade
backend. Implementa-se em paralelo com os demais módulos MVP — cada um
contribui com seu painel observacional aqui.

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0022 | Painéis agregam ao vivo por janela de período; `as_of` = instante da leitura, carimbado em todo agregado |
| 0010 | `dashboard_metrics` reconstrutível; hoje só manutenção e painel de Saúde a leem (ADR 0022) |
| 0020 | Cada cidade é um banco: o painel lê só a cidade do Host, sem filtro por `municipality_id` |
| brief do dashboard | Especificação completa dos 9 painéis e contrato de API |

## Superfícies

| Superfície | O que aparece |
|---|---|
| `api` | Namespace `Admin::` (read-only), `app/queries/admin/` (agregação ao vivo), projeção `dashboard_metrics` mantida por consumer |
| `admin` | O console do operador lê os mesmos painéis, só leitura, pelo grant de operador na cidade (Plano 3B); ainda no contrato antigo da Classificação (card admin#2) |
| `dashboard` | **É o app.** React, recharts, 9 painéis, navegação por módulo |
| `wpda` | — |

## Painéis (espelham os módulos operacionais)

| Painel | Módulo backend correspondente |
|---|---|
| Overview | (cross) |
| Ingestão WhatsApp (canal descontinuado) | 01 |
| Conversas | 02 |
| Consentimento | 02 + 07 |
| Triagens | 03 |
| Classificação/Scoring | 03 |
| Protocolos | 03 |
| Filas | (transversal — Solid Queue) |
| Eventos de domínio | 07 |

## Funcionalidades planejadas

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-05.1 | Esqueleto do app + navegação + textos PT-BR | dashboard | brief |
| F-05.2 | Auth boundary (consome módulo 06; não implementa aqui) | dashboard, api | 0011, 0012 |
| F-05.3 | Carimbo "dados de \<timestamp\>" universal | dashboard | brief |
| F-05.4 | Painel Overview | dashboard, api | brief |
| F-05.5 | Painel Ingestão (WhatsApp, canal descontinuado) | dashboard, api | brief |
| F-05.6 | Painel Conversas | dashboard, api | brief |
| F-05.7 | Painel Consentimento | dashboard, api | brief |
| F-05.8 | Painel Triagens | dashboard, api | brief |
| F-05.9 | Painel Classificação | dashboard, api | brief |
| F-05.10 | Painel Protocolos | dashboard, api | brief |
| F-05.11 | Painel Filas (Solid Queue) | dashboard, api | brief |
| F-05.12 | Painel Eventos de domínio | dashboard, api | brief |
| F-05.13 | Atualização de projeções (`UpdateDashboardJob`) | api | 0004, 0005, 0010 |
| F-05.14 | Job de reconciliação (rebuild de projeção) | api | 0010 |

## Dependências

- Todos os outros módulos MVP — cada um contribui com fonte de dado.
- Módulo 06 (Identidade/Acesso) — auth boundary é pré-requisito de produção.

## Riscos herdados

- **(ADR 0005):** painel de Filas não cobre a pior classe de falha (no-op
  silencioso do idempotency bug). Documentar como limitação.
- **(ADR 0006):** monitoramento de Solid Queue não tem ADR próprio; o
  painel é a primeira tentativa, mas alerta proativo (Slack/email quando
  fila cresce) fica fora do escopo do dashboard.
- **Operacional:** dashboard sem auth não vai para produção; bloqueio no
  módulo 06.

## Riscos que continuam

- **Brief do dashboard nunca migrado:** os painéis não têm critério de aceite
  formal além dos comentários `§4.x` no código e destas specs.
- **Custo da leitura ao vivo (ADR 0022):** cada leitura agrega o período. O
  gatilho de revisão é p95 acima de 1 s em qualquer `GET /admin/api/*`.
- ~~**Contrato duplo da Classificação:** `priorityTrue`, `priorityTrend` e o
  pivô `low/medium/high` ficam como apelidos até o `admin` migrar (card
  admin#2).~~ Resolvido em 2026-10-04: o `admin` migrou e os apelidos saíram
  do `Admin::ClassificationQuery` (ver Histórico).
- **Navegação sem URL:** o app troca de tela por estado, sem link direto nem
  botão voltar; "PT-BR" são textos fixos, sem biblioteca de i18n.
- **Ingestão:** mede só o WhatsApp, canal descontinuado; o canal web não tem
  painel de ingestão próprio (a entrada do cidadão aparece em Conversas).
- **Consentimento:** recusa não tem contagem — na web quem não aceita o termo
  não deixa registro.

## Critério de fechamento do módulo

- ✓ F-05.1 a F-05.14 verificadas.
- ✓ Suíte de invariante (`spec/requests/admin/api/module_05_invariants_spec.rb`):
  nenhum endpoint retorna dado de outra cidade; todo painel devolve `as_of`
  (instante da leitura); nenhum dado clínico cru nem identidade do cidadão;
  `/admin/api` só leitura; nenhum painel operacional lê `dashboard_metrics`.
  Cada query tem spec da janela de período.
- ~~Decisão registrada: quais KPIs vêm de `dashboard_metrics` (projeção) vs.
  agregação ao vivo.~~ Feito: todos ao vivo (ADR 0022).

## Histórico

- 2026-09-26 — Verificação do módulo (dossiê por F-ID). F-05.2 `Verified`.
  Bloqueios: F-05.3 (blocos de KPI e painel de Saúde do Overview sem carimbo),
  F-05.9 (tiers fixos `low/medium/high` contra os tiers do protocolo; prioridade
  tratada como booleano; divisão por modo sempre vazia; trilha só no seed de
  demonstração), F-05.14 (rebuild apaga `consents_revoked` e não recria). Os
  demais F-IDs ficam `Done` com ressalvas em cards próprios (testes de painel
  ausentes, estados de conversa ignorados, colunas de validação fixas em
  Protocolos). Decisão projeção vs. ao vivo registrada no ADR 0022.
- 2026-09-27 — módulo **Fechado**. F-05.1 a F-05.14 verificadas pelo usuário.
  F-05.6 subiu direto (consertado no fechamento do módulo 02). F-05.1 e F-05.5
  renomeadas para descrever o que existe. Consertado no api (merge bd706e5): o
  rebuild recria `consents_revoked` e `since:` não apaga o histórico anterior;
  Classificação e Overview com os tiers reais dos protocolos e a urgência pela
  régua do alerta (`Protocols::Urgency`), com apelidos do contrato antigo para
  o `admin`; Consentimento conta todo concedido no período; colunas
  schema/linter/gates de Protocolos vêm do `Protocols::Gate`; data inválida no
  período responde 422 (antes 500) e o painel Eventos lê só o período
  validado; specs de todas as queries, do `UpdateDashboardJob` e a suíte de
  invariante com duas cidades; os testes Minitest que a CI nunca rodava foram
  portados e removidos; guarda de ponteiros de ADR aceita até 0022. No
  dashboard (e80386b): carimbo em toda linha de KPI e no resumo de Saúde,
  Classificação no contrato novo com tolerância ao antigo (ADR 0015), testes de
  tela de 7 painéis e da fronteira de sessão. Ordem de deploy: api antes do
  dashboard. Com o WhatsApp descontinuado, a Ingestão fica como canal
  descontinuado, sem bloquear.
- 2026-10-04 — Contrato da Classificação contraído (ADR 0015, card admin#2).
  O console do operador (`apps/admin`) passou a ler o contrato novo, como o
  dashboard: `urgent`, `urgentTrend` e `urgentMaxPriority` (rótulo "Casos
  urgentes (priority ≤ N)"), pivô por `tierKeys` com `byProtocol[].counts` no
  lugar das colunas fixas `low/medium/high` (que mostravam sempre 0, porque os
  tiers reais são vocabulário livre do protocolo) e `priority` inteira na
  amostra, com `urgent` marcando as urgentes; amostra `null` aparece como
  oculta. O KPI de urgência do Overview já vinha genérico (o `id` só serve de
  chave), sem mudança. Depois disso, o api removeu os apelidos
  (`LEGACY_TIERS`, `priorityTrue`, `priorityTrend`) do
  `Admin::ClassificationQuery`, com spec fixando as chaves da resposta.
  **Ordem de deploy: `admin` antes do api** — api sem os apelidos com o
  console antigo no ar deixa o KPI de urgência e o pivô do console sem valor. O
  dashboard ainda tem a tolerância ao contrato antigo, que agora não tem mais
  quem a use.
