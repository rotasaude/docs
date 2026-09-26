# Módulo 07 — LGPD/Auditoria

- **Estado:** Planejado
- **Tipo:** MVP

## Escopo

Trilha de auditoria via `domain_events`, retenção, purge, criptografia em
repouso, isolamento entre cidades (um banco por cidade), tratamento de
revogação de consentimento. Atravessa o sistema inteiro mas tem responsabilidades
próprias.

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0014 | `domain_events` append-only, escopo (todos os eventos), TTL 12 meses, purge |
| 0013 | Cifragem do `raw` (AR Encryption), TTL curto, purge backstop |
| 0020 | Um banco e um role por cidade, conexão escolhida pelo Host, `platform_events` no banco de plataforma, chave derivada por cidade (substitui o RLS do ADR 0003) |

## Superfícies

| Superfície | O que aparece |
|---|---|
| `api` | `DomainEvent` e `domain_events` no banco de cada cidade, purge recorrentes, `Platform.audit` gravando em `platform_events` no banco de plataforma |
| `admin` | Nenhuma visão entre cidades; entrada numa cidade por grant, auditada nos dois bancos; configuração de retenção (futuro) |
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
| F-07.6 | Um banco e um role Postgres por cidade (`CONNECT` revogado de `PUBLIC` + `CONNECTION LIMIT`) | api | 0020 |
| F-07.7 | Roles Postgres: `rota_city_<slug>` por cidade + `rota_platform` + `rota_provisioner` (só no worker) | api (config) | 0020 |
| F-07.8 | Cidade resolvida pelo Host antes da autenticação (`CityCatalog` + `CityConnection.with`) | api | 0020 |
| F-07.9 | `CityScopedJob` e `EachCityJob`: jobs na fila do banco da cidade | api | 0020, 0006 |
| F-07.10 | `DomainEvents.publish` grava no `domain_events` do banco da cidade (sem `municipality_id`) | api | 0004, 0020 |
| F-07.11 | `platform_events` no banco de plataforma (sem dado pessoal; referencia `city_id`) | api | 0014, 0020 |
| F-07.12 | `Platform.audit` para eventos de identidade | api | 0014 |
| F-07.13 | Painel de eventos no dashboard da cidade | dashboard | brief |
| F-07.14 | Console de plataforma sem visão entre cidades; entrada por grant auditada nos dois bancos | api, admin | 0014, 0020 |
| F-07.15 | Tratamento de revogação de consentimento (assinante de `consent.revoked`) | api | 0008 |

## Dependências

- Todos os outros módulos — o banco por cidade isola o dado de todos;
  auditoria cobre eventos de todos.

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
  worker. Worker parado = LGPD violada por purge não executado. Com um
  supervisor por cidade, uma cidade atrasada ou fora do ar para só as
  tarefas dela.
- **(ADR 0020):** a proteção entre cidades está inteira no resolver de
  conexão. Errar a conexão entrega uma cidade inteira; o resolver precisa do
  mesmo peso de teste que o RLS tinha.

## Critério de fechamento do módulo

- F-07.1 a F-07.15 verificadas.
- Suíte de invariante (a mais importante do MVP): cidade A não vê dado de
  cidade B (host A não lê o banco B; sessão, job e grant de A não valem em B —
  `spec/cities/city_isolation_spec.rb`); `domain_events` é insert-only;
  purge respeita TTL; `raw` decifrável apenas com a chave derivada da própria
  cidade.
- Documentação operacional do "como atender uma requisição LGPD" registrada
  em `docs/operacao/`.

## Histórico
