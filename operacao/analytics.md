# Analytics — rollout e operação (módulo 14, ADR 0025)

## Quando aplicar

- No primeiro deploy do módulo 14 (api, dashboard, admin, maintenance).
- Quando a consolidação atrasar (`analyticsStatus.stale` no maintenance) ou falhar.
- Para corrigir um bug de consolidação (rebuild).

## Pré-requisitos

- Imagem do api com a migração de cidade `20260930200001_create_analytics` e a de
  plataforma `20260930200002_create_city_analytics_indicators`. As duas são **só
  aditivas**: tabelas novas e o papel `analyst` no CHECK de `memberships`.
- O worker de cada cidade na mesma imagem (o `ConsolidateAnalyticsJob` é
  recorrente de cidade, fila `housekeeping`, todo dia às 2h30 de São Paulo).
- `contracts` em `protocols-v1.2.0` (`analytic` na pergunta).

## Passos

1. **Migrar antes do tráfego.** Publicar a imagem nova do api e rodar, dela,
   `bin/rails db:migrate` (plataforma) e `bin/rails city:migrate:all` (cidades)
   **antes** de cortar o tráfego. Nunca migrar fora do rake: a cidade trava em 503.
2. **Ordem de deploy:** api → dashboard → admin → maintenance. Os três frontends
   leem rotas e campos novos (`/admin/api/analytics/*`, `GET /city_analytics`,
   `City.analyticsIndicators`/`analyticsStatus`); um frontend novo contra api
   antigo mostra erro nas telas de Analytics.
3. **Rebuild inicial**, depois do deploy do api:
   `bin/rails "city:analytics:rebuild:all"` — consolida cada cidade desde o dado
   cru mais antigo, em blocos de 30 dias, e publica as semanas fechadas na plataforma (a semana corrente, incompleta,
   nunca é publicada). O
   que já foi purgado do cru (eventos com mais de 12 meses, conteúdo de triagem
   revogada) não volta.
4. **Conceder o papel** "Análise" (`analyst`) em Equipe a quem vai ler o
   Analytics. Não pede step-up. O `municipal_admin` já lê.
5. **Marcar perguntas** para a Epidemiologia: o autor marca "Usar em Analytics"
   (só `boolean`/`enum`) numa versão nova do protocolo, que passa pelo ciclo
   assinado. Versão já publicada não muda.

## Validação

- No maintenance, na ficha da cidade: `analyticsStatus.lastRunStatus =
  succeeded`, `stale = false` e `lastPublishedAt` preenchido depois da primeira
  execução agendada.
- No dashboard, como `analyst`: as quatro abas com o carimbo "dados até
  <ontem>"; filtrando por bairro, células de 1 a 4 aparecem como "oculto".
- No console do operador: "Analytics das cidades" com as últimas 12 semanas.
- `analytics_runs` da cidade: um `scheduled` por dia; `error` vazio.

## Operação

- **Atraso (`stale`)**: o último run `succeeded` tem mais de 36 h. Veja
  `lastError` no maintenance e as falhas do Solid Queue do worker da cidade
  (`ConsolidateAnalyticsJob` levanta `Analytics::Run::Failed` quando o run falha).
  Corrigida a causa, a próxima execução refaz os últimos 30 dias sozinha; para
  não esperar, `bin/rails "city:analytics:rebuild[<slug>,<hoje-30>]"`.
- **Publicação falhou** (`lastError` começa com `publish:`): os fatos da cidade
  estão certos; a próxima execução republica a própria janela. Se o bloco era de
  um rebuild antigo, rode o rebuild daquele período de novo.
- **"Outra consolidação em curso"**: o rake encontrou o job (ou outro rebuild)
  rodando na cidade. Espere terminar e rode de novo; não há fila.
- **Bug de consolidação**: corrigido o código, `city:analytics:rebuild[<slug>,<from>,<to>]`
  do período afetado. **Atenção:** o rebuild refaz a partir do cru que existe
  hoje; triagem anonimizada depois da consolidação original sai do número (a
  consolidação agendada nunca mexe fora dos últimos 30 dias — o rebuild é a
  exceção explícita).
- **Retenção**: fatos 5 anos, `analytics_runs` 90 dias (o último `succeeded`
  sempre fica), indicadores da plataforma 5 anos — purgados pelo próprio job.
- **Revogação**: dentro dos últimos 30 dias, a pessoa sai dos números na
  próxima execução (vira triagem abortada por revogação, sem bairro); fora
  deles, o agregado anônimo fica (LGPD art. 12).

## Rollback

- Frontends: voltar a imagem anterior; nada no api depende deles.
- api: a imagem anterior ignora as tabelas novas. O `down` da migração de cidade
  **falha** depois que alguém recebeu `analyst` (memberships não se apagam) —
  rollback do papel é revogar, não migrar. As tabelas novas podem ficar.

## ADRs relacionados

0010, 0014, 0016, 0020, 0022, 0023, 0025.
