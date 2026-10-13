# Conexões do Postgres — orçamento por número de cidades

**Para que serve:** dimensionar o `max_connections` do Postgres de produção e o teto de conexões de cada cidade (`CITY_ROLE_CONNECTION_LIMIT`) antes de ativar cidades. Se a conta não fechar, o Postgres recusa conexões, e isso derruba o atendimento, não só a assinatura.

**Origem:** card rotasaude/api#74 (rodada especial do módulo 19, 2026-10-13); ADR 0020 (banco por cidade) e ADR 0032 (worker `signatures` por cidade).

**Ferramentas no api:**
- `bin/rails 'ops:connection_budget[N]'` imprime a conta para N cidades;
- `spec/config/connection_budget_spec.rb` reprova a CI se a conta para **5 cidades** passar de 80% do `max_connections`, ou se o pico de uma cidade passar de 80% do `CITY_ROLE_CONNECTION_LIMIT`.

## Configuração atual (produção)

| O quê | Valor | Onde |
|---|---|---|
| `max_connections` | **1000** | acessório `postgres`, `cmd`, em `apps/api/deploy/production/deploy.yml` |
| `CITY_ROLE_CONNECTION_LIMIT` | **150** (também o padrão do código) | `env.clear` do mesmo arquivo; aplicado a cada `rota_city_<slug>` no provisionamento e pela rake `ops:align_city_role_limits` nas cidades que já existem |
| Pool do worker (`RAILS_MAX_THREADS`) | 12 | papel `worker` (maior worker de `config/queue.yml` = 10, mais 2 do Solid Queue) |
| Pool do web (`RAILS_MAX_THREADS`) | 5 | `env.clear` (2 hosts × `WEB_CONCURRENCY` 2) |
| `idle_timeout` do pool de cada cidade | 60 s | `CityConnection.database_config` |

Mudar o `max_connections` exige reboot do acessório. A chave `cmd:` do acessório ainda precisa ser confirmada na versão do Kamal que faz o deploy (nota BLOQUEADO no `deploy.yml`).

## A conta

É a conta de **pico**: cada pool conta o menor entre o seu tamanho e as threads que o usam.

**Web.** São `hosts × WEB_CONCURRENCY` processos Puma, cada um com `T` threads.
- Base: `processos × T × 3` (plataforma, cache e a fila de plataforma, que o web usa ao enfileirar o provisionamento e os e-mails de convite).
- Por cidade atendida: `processos × T × 2` (`CityRecord` e shard do Solid Queue, os dois no banco da cidade).
- O `idle_timeout` de 60 s devolve o pool de uma cidade que ficou sem uso. Isso não muda o pico, só a média.

**Worker por cidade.** Um supervisor Solid Queue com 7 processos: 4 workers, dispatcher, scheduler e supervisor. Para cada worker com `t` threads:
- no banco da cidade: `(t + 2) + t + t` (Solid Queue padrão, shard do Solid Queue e `CityRecord`);
- na plataforma: `t` (`PlatformRecord`).

Os 3 processos de controle somam 2 conexões no banco da cidade e 1 na plataforma, cada um.

**Plataforma.** O supervisor da plataforma: `(t + 2) + t` por worker, mais o controle, mais 1 conexão do gerente `bin/city_workers`, que relê o catálogo de cidades.

**Hosts de worker.** Cada host de worker roda um supervisor por cidade e um da plataforma. Com 2 hosts, o worker por cidade e a plataforma dobram, e o pico de uma cidade no banco dela passa de 117 para 194. A rake lê os hosts do `deploy.yml`.

**Reserva:** 20 conexões (console, migrações, backup, `kamal app exec`, as `superuser_reserved_connections` e as conexões avulsas do provisionamento de cidade).

**Com os arquivos de hoje** (workers de produção com 5, 10, 3 e 3 threads; plataforma com 2):

| Parte | Conexões |
|---|---|
| Web, base | 60 |
| Web, por cidade | 40 |
| Worker por cidade, no banco da cidade | 77 |
| Worker por cidade, na plataforma | 24 |
| Plataforma (com o gerente) | 13 |
| Reserva | 20 |
| **Pico de uma cidade no banco dela** (web 40 + worker 77) | **117**, por isso o `CITY_ROLE_CONNECTION_LIMIT` = 150 |
| **Total** | `93 + 141 × cidades` |

## Tabela por número de cidades

`max_connections` mínimo = total ÷ 0,8 (20% de folga).

| Cidades | Total de pico | `max_connections` mínimo |
|---|---|---|
| 1 | 234 | 293 |
| 2 | 375 | 469 |
| 3 | 516 | 645 |
| 4 | 657 | 822 |
| **5 (go-live)** | **798** | **998 → 1000** |
| 8 | 1221 | 1527 |
| 10 | 1503 | 1879 |
| 15 | 2208 | 2760 |
| 20 | 2913 | 3642 |

**Memória:** no pior caso, cada conexão do Postgres ocupa cerca de 10 MB (`work_mem` à parte). Com 1000 conexões, isso dá cerca de 10 GB: dimensione o host do Postgres junto com o `max_connections`.

## Quando revisar

Rode `bin/rails 'ops:connection_budget[N]'` e ajuste o `deploy.yml` sempre que:
- o número de cidades ativas previsto passar de 5. A spec usa 5: suba o número nela também;
- `config/queue.yml` ganhar worker ou threads, ou o web ganhar host, processo ou thread. A spec reprova a CI até a conta fechar de novo;
- o pico de uma cidade passar de 120 (80% de 150). Suba o `CITY_ROLE_CONNECTION_LIMIT` e depois rode, no host de worker, `kamal app exec --roles=worker "bin/rails ops:align_city_role_limits"`: o provisionamento só aplica o teto às cidades novas, e esta rake realinha as que já existem;
- entrar um segundo host de worker. A conta dobra o worker por cidade.

**Acima de cerca de 8 cidades**, ou quando a memória do host não fechar, avalie um pooler (PgBouncer em modo transação) na frente do Postgres, em vez de subir o `max_connections`. É o próximo passo, ainda sem receita: o modo transação exige cuidado com *advisory locks* e *prepared statements*.

## Gate de go-live

A conta é teórica. Antes de ativar a 2ª cidade em produção:
1. **Staging com carga:** 5 cidades ativas, web e worker com a configuração de produção, carga de atendimento e de assinatura em lote ao mesmo tempo.
2. **Medição no pico:**

   ```sql
   select count(*) from pg_stat_activity;
   select usename, count(*) from pg_stat_activity group by 1 order by 2 desc;
   ```

   O total precisa ficar abaixo de 80% do `max_connections`, e cada `rota_city_<slug>` abaixo de 80% do `CITY_ROLE_CONNECTION_LIMIT`.
3. **Registro** do resultado no card rotasaude/api#74.
