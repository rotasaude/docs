# Rollout — receita controlada (módulo 19d)

Runbook do operador para publicar o módulo 19d (ADR 0034), ligar a receita de
controle especial (RCE, listas C1/C5) e a de antimicrobiano (RET) digitais com
número do SNCR numa cidade, e acompanhar o estoque de números. Bases:
[`rollout-assinatura.md`](rollout-assinatura.md) (19b) e
[`rollout-documentos-clinicos.md`](rollout-documentos-clinicos.md) (19c).

O 19d cobre a RCE e a RET digitais, o estoque de números do SNCR por
prescritor, o registro da Notificação A/B/B2 de papel na consulta e o painel do
SNCR do admin. Tudo fica atrás do interruptor `controlled_prescriptions`.
Desligado, vale o comportamento do 19c: controlado bloqueado e antimicrobiano
só em papel.

**O SNCR real ainda não foi provado.** O 19d foi entregue e verificado contra o
SNCR simulado. Nenhuma cidade real liga o SNCR antes dos gates do §11
(rotasaude/api#63).

## 1. Publicação

Ordem obrigatória. O api vai **antes** dos frontends.

1. `contracts` com a tag `clinical-v1.2.0`: a receita do esquema
   `rotasaude.clinical_document.v1` ganha `category`, `sncr`,
   `patient_identification` e `prescriber_contact`. É MINOR: os vetores da
   1.1.0 seguem iguais byte a byte.
2. `api` (`123b8b7` ou posterior), com **imagem nova**. Por cidade, antes de
   cortar o tráfego:
   1. publique a imagem;
   2. faça backup dos bancos (plataforma e cidades) e pare o worker;
   3. rode `bin/rails db:migrate` na plataforma. A migração `20261010700001`
      cria a marca de anticonvulsivante no catálogo, o kind `anticonvulsant`
      nas listas Anvisa e a tabela `sncr_checks`. **Não existe**
      `db:migrate:platform`;
   4. rode `bin/rails city:migrate:all` **da imagem nova**. A
      `20261010700001` de cidade cria o estoque de números
      (`sncr_number_batches`, `sncr_numbers`), o `prescriber_address` cifrado
      no perfil do profissional, o documento `controlled_notification_record`
      e a coluna `paper_reason`. É **irreversível**;
   5. religue o worker.
3. `bin/rails medications:import_anvisa`, na imagem publicada, para gravar a
   lista da Portaria 344 de cada item do catálogo e a marca de
   anticonvulsivante. Sem isso, nenhum item vira RCE.
4. Credenciais do SNCR nas credenciais cifradas do api, em
   `production.yml.enc`: `sncr.{base_url, auth_url, maintainer_cnpj}`. A
   Anvisa não pede `client_id` nem `client_secret`.
5. `dashboard` (`daafeee` ou posterior), depois do api.
6. `maintenance` (`7db3311` ou posterior), depois do dashboard.

O retorno do gov.br cai em `https://<host do dashboard da cidade>/dashboard/sncr/callback`.

Publicar não muda nenhuma cidade: `controlled_prescriptions` nasce desligado.

## 2. Ligar numa cidade

1. Pré-requisito: `clinical_documents` ligado (19c). Sem ele o interruptor
   mostra `clinical_documents_disabled`.
2. Ligue `controlled_prescriptions` na cidade:
   - em dev e staging, pelo `maintenance` (aba Funcionalidades);
   - em produção, que não tem maintenance, pelo console do Rails:
     `Platform::Features.set!(city:, key: "controlled_prescriptions", enabled: true, maintainer:)`.
     A troca vai para a trilha da plataforma (`city.feature_changed`).
3. Em dev e staging, ligue também `sncr_mock` (exige `controlled_prescriptions`;
   sem ele mostra `controlled_prescriptions_disabled`). Os números vêm do SNCR
   simulado e toda tela, PDF e página pública mostra a faixa **"numeração
   simulada — sem validade"**. Em produção `sncr_mock` não aparece no catálogo e
   a mutation responde `unknown_feature`.
4. Cada prescritor preenche o **contato** (endereço e telefone) em **Meu
   perfil**. O contato sai impresso na RCE e na RET digital. Sem endereço, a RCE
   dá 422 `prescriber_address_missing`, e a RET também, quando sairia digital.
   O telefone é o do cadastro de profissional (módulo 10).

## 3. Números do SNCR

- Cada médico ou dentista com CRM ou CRO pede os números pelo próprio gov.br,
  em **Conta → SNCR** ("Obter números do SNCR"). Sem perfil de profissional, ou
  com conselho fora de CRM/CRO, a tela responde 403 `cbo_not_allowed`.
- Cada pedido traz um bloco de **1.000 números** do tipo pedido (RCE ou RET).
- Até **3 pedidos por mês por tipo**. O quarto dá 409 `monthly_limit_reached`.
- A cota e o saldo são **separados por modo**: números simulados e reais nunca
  se misturam, nem na cota do mês nem no saldo.
- A tela avisa quando sobram **menos de 50** livres de um tipo. Saldo zero não
  impede emitir: a receita sai em papel (`no_sncr_number`).
- Erros do retorno do gov.br: `authorization_expired` (passou dos 30 s),
  `authorization_denied`, `registration_mismatch` (o gov.br é de outra pessoa),
  `sncr_unavailable` e `sncr_exhausted`.

## 4. Regras de emissão

| Receita | Itens aceitos | Limites | Vias e validade |
|---|---|---|---|
| RCE (controle especial) | Só itens C1 ou C5 do catálogo, todos com duração | Até 3 substâncias C1; até 60 dias por item (180 para anticonvulsivante) | 2 vias; 30 dias |
| RET (antimicrobiano) | Só itens antimicrobianos do catálogo | — | 2 vias; 10 dias |

- **Mistura:** item comum ou texto livre junto de item da RCE ou da RET dá 422
  `mixed_categories`, com o `index` do primeiro item que não serve. O
  profissional separa em duas receitas (por exemplo, amoxicilina numa RET e
  paracetamol numa receita comum).
- **Texto livre antimicrobiano:** receita só com itens em texto livre (sem
  nenhum item do catálogo) sai em papel, com `paper_reason: free_text_items`,
  como a RET em papel do 19c.
- **RCE:** exige o endereço do paciente, digitado ou reaproveitado da última
  receita. O CPF vem do cadastro do paciente.
- **Notificação A, B e B2:** não é impressa. O profissional registra na
  consulta ("Registrar Notificação de papel") o número do talão, a UF da
  numeração e o item (uma substância das listas A ou B; até 30 dias na A e 60
  na B/B2). O registro entra na lista de medicamentos em uso e não aparece na
  página pública. Item A ou B numa receita dá 422 `requires_notification`.
- **Controlado C2, C3 e C4:** não suportado (422 `not_supported`).
- **Enfermagem:** não emite RCE (403 `cbo_not_allowed`). A RET dela sai sempre
  em papel (`nurse_antimicrobial`).
- **Dentista:** não emite enquanto a consulta do 19a recusar o CBO 2232
  (odontologia é o módulo 28).

## 5. Modo e `paper_reason`

O modo é decidido na emissão e **nunca** é convertido. Digital exige
certificado ativo da autora, `digital_signature` utilizável e um número livre
do tipo. Em papel, o documento grava o motivo em `paper_reason` (só no papel,
imutável, devolvido em toda leitura):

| `paper_reason` | Significado |
|---|---|
| `feature_disabled` | Antimicrobiano com `controlled_prescriptions` desligado: papel do 19c |
| `nurse_antimicrobial` | RET da enfermagem, sempre em papel |
| `free_text_items` | Receita só com itens em texto livre antimicrobianos |
| `signature_unavailable` | `digital_signature` desligado ou inutilizável na cidade |
| `no_certificate` | A autora não tem certificado ativo |
| `no_sncr_number` | A autora não tem número livre do tipo (o saldo acabou) |

O motivo é gravado em todo documento da consulta emitido em papel, não só na
receita. O documento que voltou ao papel depois (19b) fica com `paper_reason`
nulo. O registro da Notificação é sempre papel, sem motivo.

## 6. Assinatura

Igual ao 19b e ao 19c: o pedido de assinatura nasce na emissão, na mesma
transação que tira o número, e sai na fila `signatures`. Receita pendente
aparece como "aguardando assinatura" e não imprime. Sem nenhuma escrita com
step-up no 19d.

## 7. PDF nos leiautes da Anvisa

- **RCE:** "RECEITA DE CONTROLE ESPECIAL", número do SNCR em destaque, 1ª via
  da farmácia e 2ª via do paciente.
- **RET:** "Receita de Antimicrobiano", 2 vias, validade de 10 dias. A RET
  digital sem identificação do paciente não imprime a linha de endereço.
- **Rodapé Anvisa:** CNPJ da plataforma (gravado no documento na emissão), a
  URL de conferência e sncr.anvisa.gov.br, mais o QR code e o rodapé do 19c.
- **Simulado:** faixa "NUMERAÇÃO SIMULADA — SEM VALIDADE".
- Papel: como no 19c.

## 8. Cancelamento e volta ao papel

- O cancelamento (19c) e a volta ao papel (19b) **anulam** o número
  (`voided`). Ele nunca volta a `free`. Uma emissão que falha devolve o número
  a `free` por rollback.
- O SNCR não tem cancelamento na API. A página pública `/v` da receita
  cancelada mostra o aviso de **não dispensar**; a farmácia só sabe do
  cancelamento por ela.

## 9. Acompanhar

- **Admin municipal:** painel do SNCR no dashboard, só leitura, com o saldo de
  RCE e RET de cada prescritor e a marca de saldo baixo (menos de 50 livres em
  um dos tipos). Saldo e último pedido são do modo corrente.
- **Maintenance:** quadro do SNCR ao lado do interruptor (`sncrStatus`, só
  token humano): configurado, alcançável, última checagem e se o simulado está
  disponível. O aviso de simulado aparece junto do interruptor `sncr_mock`.

## 10. Validação

- `bin/rails medications:import_anvisa` imprime as contagens, inclusive a dos
  anticonvulsivantes.
- Em staging, com `sncr_mock`: pedir um bloco de RCE e um de RET, emitir e
  assinar uma de cada, conferir a faixa de simulado no PDF e em `/v`, cancelar
  uma e ver o número `voided` e o aviso de não dispensar.
- O painel do admin mostra o saldo, e o maintenance mostra o SNCR configurado.
- Nenhum número do SNCR, CPF ou endereço em log, evento, Analytics ou erro.

## 11. Gates de go-live

| Item | Bloqueia? |
|---|---|
| rotasaude/api#63 — troca do token do SNCR no servidor provada na homologação, com login gov.br do usuário: o servidor troca o `session_id` com o `Origin` do `client_url` e o pedido do bloco responde 201 em 30 s. A sonda **não foi feita**; decisão e plano B em [`pesquisa/2026-10-10-sncr-token-no-servidor.md`](../pesquisa/2026-10-10-sncr-token-no-servidor.md) | **Sim** |
| `SEND_ORIGIN` e host do `/auth` em `Sncr::Client`, e as expressões das mensagens de erro do SNCR (limite, `/esgot/`, inscrição), conferidas contra o SNCR real | **Sim** |
| Domínio `.br` de cada cidade aceito como `client_url` | **Sim** |
| CNPJ da mantenedora confirmado com a Anvisa (SaaS municipal) | **Sim** |
| Credenciais `sncr.*` em `production.yml.enc` | **Sim** |
| Lista de anticonvulsivantes (ATC N03A, 52 linhas, com canabidiol, gabapentina e pregabalina) conferida por farmacêutico contra a RENAME, junto com rotasaude/api#56 | **Sim** |
| rotasaude/api#62 — `log_statement` do Postgres de produção (e staging) em `'none'` ou `'ddl'`, conferido com `SHOW log_statement;`. Com `'all'` ou `'mod'`, INSERT/UPDATE gravam em claro os números do SNCR e outros valores | **Sim** |
| rotasaude/api#62 — proxy de borda de cada cidade a partir de `apps/api/config/edge-proxy/nginx-city.conf.example` (access log sem o token de `/v` e `/r`), com a evidência colhida no proxy real de staging | **Sim** |
| rotasaude/api#62 — `TRUSTED_PROXIES` no deploy do api com os CIDRs dos proxies de borda (sem ele o api avisa no log e o limite por IP vê só o IP do proxy; valor inválido derruba o boot) | **Sim** |
| Regra da VISA local para a Notificação de papel | **Sim** |
| Contato (endereço e telefone) preenchido por cada prescritor antes de ligar | **Sim** |
| Gates do 19b e do 19c (api#53, api#56, api#57, api#62) | **Sim** |

## ADRs relacionados

0028 (interruptores), 0032 (assinatura), 0033 (documentos clínicos), 0034
(receita controlada e SNCR).
