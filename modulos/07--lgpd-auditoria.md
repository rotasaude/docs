# Módulo 07 — LGPD/Auditoria

- **Estado:** Planejado
- **Tipo:** MVP

## Escopo

Trilha de auditoria via `domain_events`, retenção, purge, criptografia em
repouso, isolamento multi-tenant (RLS), tratamento de revogação de
consentimento. Atravessa o sistema inteiro mas tem responsabilidades
próprias.

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0014 | `domain_events` append-only, escopo (todos os eventos), TTL 12 meses, purge |
| 0013 | Cifragem do `raw` (AR Encryption), TTL curto, purge backstop |
| 0003 | RLS, dois papéis (`rota_app` / `rota_admin`), políticas, fail-closed |

## Superfícies

| Superfície | O que aparece |
|---|---|
| `api` | `DomainEvent`, `domain_events` table, RLS em todas as tabelas de tenant, purge recorrentes, `Platform.audit` (caminho irmão para platform-scope) |
| `admin` | Visão cross-tenant de eventos de plataforma, configuração de retenção (futuro), saúde do RLS |
| `dashboard` | Eventos de domínio da cidade (referências), inspeção, busca por janela de tempo / nome |
| `wpda` | — |

## Funcionalidades planejadas

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-07.1 | Tabela `domain_events` append-only | api | 0014 |
| F-07.2 | Insert na mesma transação do `publish` | api | 0014, 0004 |
| F-07.3 | Purge de `domain_events` (12 meses) recurring | api | 0014 |
| F-07.4 | Cifragem do `raw` (AR Encryption) | api | 0013 |
| F-07.5 | Purge do `raw` pós-processamento (recurring) | api | 0014 |
| F-07.6 | RLS habilitado + FORCE em todas as tabelas de tenant | api | 0003 |
| F-07.7 | Dois papéis Postgres (`rota_app` / `rota_admin`) | api (config) | 0003 |
| F-07.8 | `around_action` `within_tenant` na web | api | 0003 |
| F-07.9 | `TenantScopedJob` concern para jobs | api | 0003 |
| F-07.10 | Carimbo `municipality_id` no `publish` | api | 0004 |
| F-07.11 | `domain_events.municipality_id` nullable (platform-scope) | api | 0014 |
| F-07.12 | `Platform.audit` para eventos de identidade | api | 0014 |
| F-07.13 | Painel de eventos no dashboard da cidade | dashboard | brief |
| F-07.14 | Painel cross-tenant no admin | admin | brief |
| F-07.15 | Tratamento de revogação de consentimento (assinante de `consent.revoked`) | api | 0008 |

## Dependências

- Todos os outros módulos — RLS impõe isolamento universal; auditoria
  cobre eventos de todos.

## Riscos herdados

- **(ADR 0014) (LGPD vs. imutabilidade):** Art. 18 (eliminação) bate de
  frente com append-only de `consents`, `domain_events`, `report_snapshots`.
  Nenhum ADR resolveu. Decisão de "o que é apagável vs. retido por base
  legal" continua em aberto.
- **(ADR 0014):** `consent.revoked` é publicado mas não desfaz nada;
  efeitos colaterais já disparados permanecem. Falta consumer real e base
  legal declarada de retenção pós-revogação.
- **(ADR 0014) (Art. 20):** revisão humana de decisão automatizada não
  tem dono. Trail (módulo 03) dá insumo mas não fluxo.
- Todas as recurring tasks concentram fragilidade no
  worker. Worker parado = LGPD violada por purge não executado.

## Critério de fechamento do módulo

- F-07.1 a F-07.15 verificadas.
- Suíte de invariante (a mais importante do MVP): cidade A não vê dado de
  cidade B em nenhuma tabela de domínio; `rota_app` sem
  `app.municipality_id` setado levanta; `domain_events` é insert-only;
  purge respeita TTL; `raw` decifrável apenas com a chave do env.
- Documentação operacional do "como atender uma requisição LGPD" registrada
  em `docs/operacao/`.

## Histórico
