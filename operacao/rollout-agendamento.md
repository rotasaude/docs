# Rollout — chamada, retorno e agendamento

Deploy e rollback do subprojeto 4 (chamada, retorno e agendamento; ADR 0019).

## Quando aplicar

No primeiro deploy que leva a migração de cidade
`20260926000001_create_appointments` a qualquer ambiente com cidades reais.

## Pré-requisitos

- A migração é **irreversível**: o `down` levanta erro.
- Código antigo **não funciona** no banco migrado:
  - processos antigos têm em cache o default `'open'` do status do
    atendimento, então o check-in bate numa violação de CHECK;
  - o `close` antigo não considera `waiting` como aberto e devolve 409.
- O inverso também falha: código novo num processo cujo cache de colunas é
  anterior à migração devolve 500 `unknown attribute appointment_id` (visto em
  dev).
- Conclusão: entre a migração e a troca de processos, check-in e fechamento de
  atendimento falham. Essa janela tem que ser a menor possível.

## Passos

1. Publicar a imagem nova do api.
2. Rodar `city:migrate:all` **a partir dessa imagem** (nunca migrar fora do
   rake: a cidade trava em 503).
3. Logo em seguida, reiniciar ou trocar **todos** os processos de api e de
   worker, sem exceção.
4. Só então publicar dashboard e wpda:
   - dashboard novo contra api antigo recebe 404 em `/queue`;
   - wpda novo contra api antigo mostra erro no topo do histórico.

## Validação

- Check-in por código e por exceção abrem atendimento `waiting`.
- Fechar um atendimento com desfecho `return` cria o pedido de retorno.
- Dashboard carrega a fila (`/queue`) e o wpda mostra o histórico sem erro.
- Nenhum 500 `unknown attribute appointment_id` nos logs do api.

## Rollback

Rollback é **correção para frente**. Depois de migrar, nunca reverter o código
do api nem do worker: código antigo quebra no banco novo, e o `down` da
migração não existe.

## Cuidado no console

Nunca rodar um `EachCityJob` inline com `perform_now` dentro de um contexto de
cidade, em console ou runner: ele vaza `Current.city`, e as escritas saem
cifradas com a chave de outra cidade. Até isso ser corrigido (há tarefa
separada), expirar horários manualmente com `Appointments::Lapse.call`, dentro
do contexto da cidade certa.

## ADRs relacionados

- 0019 — chamada, retorno e agendamento.
- 0020 — banco por cidade.
