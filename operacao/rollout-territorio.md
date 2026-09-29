# Rollout — Território (módulo 11, ADR 0023)

A migração é **só de expansão** (tabelas `neighborhoods` e
`neighborhood_coverages`; colunas opcionais em `health_units`, `citizens` e
`triages`; trigger de imutabilidade do bairro da triagem). Não há ordem rígida
de imagens: sem semente, nada quebra — o cidadão fica sem bairro e o bloco de
unidade de referência não aparece.

## Ordem

1. **api** — publicar a imagem (a partir de `2995f95` ou posterior) e rodar
   `bin/rails city:migrate:all` **da imagem nova**, antes de cortar o tráfego.
   Nunca migrar fora do rake (a cidade trava em 503).
2. **Semente de bairros** — em cada cidade que tem arquivo em
   `db/seeds/territory/<slug>.yml`:

   ```bash
   bin/rails "city:territory:seed[curitiba]"
   bin/rails city:territory:seed:all
   ```

   A carga casa por `seed_key`, só cria o que falta e nunca altera nem desativa
   o que existe. A cobertura da semente aponta para unidades **pelo nome**:
   unidade que não existe na cidade vira aviso na saída, não erro. Em staging e
   produção, confira os avisos e ajuste a cobertura pela tela Território.
3. **dashboard** (`cbbb75b` ou posterior) e **wpda** (`d1b552f` ou posterior).
   Os dois leem campos novos do api; publicar antes do passo 1 mostra erro nas
   telas novas.

## Conferência depois do deploy

- `GET /admin/api/neighborhoods` responde 200 com a lista (qualquer papel que
  lê os painéis).
- Tela **Cidade → Território** (só `municipal_admin`) lista os bairros da
  semente.
- No wpda, a escolha da pessoa pergunta o bairro quando a cidade tem bairros.
- O relatório público (`/r/:token`) **não** mostra unidade de referência nem
  bairro; mostra "Voltar ao início".

## CSP

Hoje não há CSP no deploy do dashboard. Se passar a existir, precisa de
`connect-src https://viacep.com.br` (a consulta de CEP é feita pelo navegador).
O api nunca chama serviço de CEP.

## Reversão

`down` da migração `20260928100001_create_territory` foi exercitado num banco
descartável (remove e recria tudo). Reverter apaga bairros, cobertura e o bairro
copiado nas triagens — só em emergência, e depois de voltar as imagens do
dashboard e do wpda.
