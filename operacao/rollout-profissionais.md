# Rollout — profissionais (módulo 10)

Deploy do módulo 10 (perfil, vínculo com CBO, turnos e a regra da chamada;
ADR 0021).

## Quando aplicar

No primeiro deploy que leva a migração de cidade
`20260928000001_create_professionals` e o api com a regra da chamada
(`Professionals::ClinicalAuthorization`, merge `66ab5fc` em diante) a qualquer
ambiente com cidades reais (staging ou produção).

## Pré-requisitos

- O Postgres de cada cidade precisa da extensão **`btree_gist`** (pacote
  `postgresql-contrib`). O papel dono do banco da cidade consegue criá-la sem ser
  superusuário (provado no PG 15.19), mas a imagem do banco tem que trazer o
  pacote.
- A migração é só expansão (três tabelas novas) e é reversível.
- A partir do api com a regra da chamada, **um `health_professional` sem perfil e
  sem vínculo ativo com a unidade não chama nem registra desfecho clínico**
  (403 `missing_link`). Sem cadastro, ninguém chama.
- A fila de atendimento passa a ler as tabelas novas (nome do profissional). Uma
  cidade sem a migração quebra a fila existente, não só as telas novas.

## Passos

1. Publicar o api em **`8ee7da4`** (fim da fatia 3: tabelas, perfil, vínculos e
   turnos, ainda **sem** a regra da chamada).
2. Rodar `city:migrate:all` **a partir dessa imagem** (nunca migrar fora do
   rake: a cidade trava em 503).
3. Reiniciar ou trocar todos os processos de api e de worker.
4. Publicar o dashboard com as telas do módulo 10 (merge `f750c41` em diante).
5. Em cada cidade, o `municipal_admin` cadastra no dashboard (Equipe →
   Profissionais):
   - o perfil de cada profissional listado em "Com papel, sem cadastro
     completo";
   - os vínculos com as unidades onde cada um atende (pede step-up).
6. Conferir que o painel "Com papel, sem cadastro completo" está vazio, ou que
   quem sobrou realmente não atende.
7. Publicar o api final (merge `a2d20c8` em diante, que inclui a regra da
   chamada e a leitura das unidades pelo profissional) e reiniciar api e worker.

## Validação

- Um profissional com vínculo vê "Chamar próximo" na fila da unidade dele e
  chama.
- Na unidade sem vínculo, a fila mostra "Você não tem vínculo com esta unidade".
- A recepção continua fazendo check-in e marcando "Saiu sem atendimento".
- A fila mostra o nome profissional de quem chamou.
- Nenhum 403 `missing_link` inesperado nos logs de quem deveria atender.

## Rollback

- Do passo 7 para trás: republicar o api `8ee7da4`. A regra da chamada sai; as
  tabelas e os cadastros ficam.
- A migração tem `down`, mas só faz sentido antes de haver cadastro: vínculos e
  turnos só aceitam acréscimo, e apagar as tabelas perde o histórico de lotação.

## ADRs relacionados

- ADR 0021 — profissionais, vínculo, turnos e a regra da chamada.
- ADR 0019 — chamada e desfecho.
- ADR 0020 — banco por cidade; migração pelo rake.
