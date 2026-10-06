# Rollout — Catálogo de triagens (módulo 15, ADR 0027)

As migrações de cidade `20261005100001_add_triage_catalog` e
`20261005300001_add_suggestion_only_to_triage_offers` são **só de expansão**:
colunas opcionais de perfil em `citizens` (cifradas), tabelas `triage_offers`,
`triage_suggestions` e `triage_offer_daily_counts`, e o trigger de transição
das sugestões. Cidadão antigo fica sem perfil e o wpda pede o perfil antes do
catálogo.

O que **não** é opcional é a ordem do api e do wpda: o api novo exige
`protocol_name` em `POST /citizen/conversations`, então o wpda antigo deixa de
iniciar triagem assim que o api novo entra.

## Ordem

1. **contracts** — tag `protocols-v1.4.0` (já publicada). O api valida contra a
   cópia em `config/protocols/schema.json`, idêntica à tag.
2. **api** (`a582dbe` ou posterior) — publicar a imagem e rodar
   `bin/rails city:migrate:all` **da imagem nova**, antes de cortar o tráfego.
   Nunca migrar fora do rake (a cidade trava em 503).
3. **wpda** (`6a5d55c` ou posterior) — **na mesma janela do api**. Entre o
   passo 2 e este, o cidadão não consegue iniciar triagem.
4. **dashboard** (`66c0d45` ou posterior) — depois do api: a aba "Catálogo de
   triagens" e o simulador leem rotas novas (`/triage_catalog`,
   `simulate_offer`).

## Antes de oferecer protocolo com elegibilidade numa cidade

O perfil (nascimento, sexo, identidade de gênero) é dado novo, e o termo de
consentimento vigente da cidade precisa cobri-lo. Publique a versão nova do
termo **antes** de configurar qualquer protocolo com `offer.eligibility`:

```bash
bin/rails "city:consent_term:publish[curitiba,3,/caminho/termo-v3.txt]"
```

O sistema não confere se o texto cobre o perfil: isso é revisão humana. Depois
da publicação, cada cidadão aceita o termo novo na próxima entrada no wpda.

Protocolo com elegibilidade só é oferecido se a cidade tiver a linha dele no
catálogo (`triage_offers`, criada pela tela). Para um protocolo que só deve
chegar pela sugestão de outro (um aprofundamento, por exemplo), marque "Só por
sugestão" na linha: pausar a linha tiraria também a sugestão. Protocolo sem bloco `offer`
continua "para todos" e aparece ao cidadão pelo nome técnico até ganhar título.

## Conferência depois do deploy

- `GET /citizen/people` traz `profile` em cada pessoa.
- No wpda, uma pessoa sem perfil vê "Sobre esta pessoa" antes do catálogo, e o
  catálogo mostra só as triagens que se aplicam a ela.
- Tela **Governança → Protocolos → Catálogo de triagens** (`municipal_admin`)
  lista os protocolos ativos; salvar pede step-up e gera `triage_offer.changed`.
- Contadores de 1 a 4 aparecem como "< 5".

## Reversão

`down` remove as tabelas e as colunas de perfil (e com elas o perfil já
declarado e as sugestões); a função `rota_triage_suggestion_guard()` fica órfã
no banco, sem efeito. Só em emergência, e depois de voltar as imagens do wpda e
do dashboard — e o wpda antigo só funciona com o api antigo.
