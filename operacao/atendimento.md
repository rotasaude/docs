# Atendimento na unidade (módulo 13, ADRs 0018, 0019 e 0021)

Procedimentos do dia a dia do módulo Atendimento: check-in, fila, chamada e
desfecho. O **porquê** está nos ADRs 0018 (check-in e desfecho), 0019 (estados
e chamada) e 0021 (chamada só por profissional vinculado).

## Quando aplicar

- Um atendimento ficou esquecido na fila (não expira sozinho — risco do ADR
  0019).
- O cidadão chegou sem celular e precisa de check-in por exceção.
- O balcão recebeu "já fez check-in" ou "tentativas esgotadas".
- O profissional não consegue chamar ou encerrar.
- Auditoria: quem consultou um CPF pela exceção.

## Pré-requisitos

- Recepção: papel `citizen_verifier` na cidade.
- Chamada e desfecho clínico: papel `health_professional` **e** vínculo ativo
  com a unidade do atendimento (módulo 10). O turno nunca bloqueia.
- Acesso ao console da cidade (`bin/rails console` da imagem em produção) só
  para as consultas de auditoria; nenhum passo abaixo altera dado por console.

## Passos

### 1. Atendimento esquecido na fila

`attendances` só aceita as transições previstas; não há expiração automática
e nada se apaga.

1. Se está em **Aguardando** (`waiting`): qualquer atendente ou profissional
   da unidade usa **"Saiu sem atendimento"** na fila. O desfecho fica `left`.
2. Se está em **Em atendimento** (`in_care`): só um profissional com vínculo
   ativo na unidade encerra, com o desfecho clínico real (atendido,
   encaminhado ou retorno). Se o profissional que chamou saiu, outro
   vinculado encerra.
3. Não existe desfazer: atendimento encerrado não muda mais (trigger
   `rota_attendance_guard`). Erro de desfecho vira nota no próximo
   atendimento, não correção.

### 2. Cidadão sem celular (exceção por CPF)

1. Em Atendimento → Check-in → "Cidadão sem o código", digitar o CPF do
   documento.
2. Escolher a triagem (ou o horário de hoje) na lista; só aparecem as
   elegíveis (web, concluída há até 3 dias, sem atendimento).
3. Escrever o motivo (10 caracteres ou mais). O motivo fica gravado no
   atendimento e nunca sai em evento.
4. A exceção **não** valida o cadastro; para validar, use o balcão de
   validação com o código do cidadão.

### 3. Mensagens do balcão

| Mensagem na tela | Causa | O que fazer |
|---|---|---|
| código não confere | código errado ou de outro CPF | conferir o CPF do documento; o cidadão gera outro código |
| código vencido ou já usado | passou de 10 min ou já foi usado | o cidadão gera outro em "Cheguei na unidade" |
| tentativas esgotadas | 5 erros no mesmo código | o cidadão gera outro; se persistir, exceção por CPF com motivo |
| triagem com mais de 3 dias | concluída há mais de 3 dias | não há check-in por essa triagem; o cidadão faz nova triagem |
| já fez check-in (com unidade e hora) | a triagem já tem atendimento | procurar o cidadão na fila da unidade indicada |
| muitas tentativas — aguarde alguns minutos | 30 ações de check-in em 10 min pelo mesmo servidor | aguardar |

### 4. Profissional não consegue chamar ou encerrar

- "Você não tem vínculo com esta unidade" / `missing_link`: o
  `municipal_admin` confere o vínculo no módulo Profissionais (vínculo
  encerrado ou em outra unidade).
- `missing_role`: falta o papel `health_professional` (Equipe, com step-up).
- "Chamar próximo" com a fila cheia pega o próximo livre; dois profissionais
  nunca levam o mesmo cidadão. Se todos os que aguardam estão sendo chamados
  por outros naquele instante, o botão só recarrega a fila; basta clicar de
  novo.

### 5. Auditoria da exceção (LGPD)

Cada busca por CPF na exceção grava `attendance.exception_searched` em
`domain_events` com quem buscou (`by_user_id`), a unidade
(`health_unit_id`), os cidadãos encontrados (`citizen_ids`) e quantos
resultados vieram — nunca o CPF. Para responder "quem consultou este
cidadão", filtre esses eventos pelo `citizen_id` dele; o check-in consumado
fica em `attendances` com método `cpf_exception`, quem fez e o motivo.

## Validação

- A fila da unidade mostra Aguardando pela prioridade da triagem (depois pela
  chegada) e Em atendimento pela hora da chamada.
- Um atendimento encerrado some da fila e aparece no histórico do cidadão no
  `wpda`.

## Rollback

Não se aplica aos procedimentos: nenhum passo altera dado fora das telas.
Para o deploy da garantia "nasce aguardando" no banco, ver a seção abaixo.

## Deploy da verificação (2026-09-30)

A migração de cidade da verificação do módulo 13 só acrescenta uma guarda
de INSERT em `attendances` (nasce `waiting`, sem chamada nem desfecho).
Publicar a imagem do api e rodar `bin/rails city:migrate:all` **da imagem
nova** antes de cortar o tráfego; nunca migrar fora do rake. O dashboard
pode subir depois do api.

## Fechar uma unidade (esvaziar e desativar)

Quando uma unidade precisa fechar (reforma, mudança de endereço), o
`municipal_admin` a esvazia antes de desativar (api#29, F-09.3):

1. Em **Atendimento → Unidades**, a unidade com pedidos ou horários mostra
   **Esvaziar**. Escolha a unidade de destino (ativa) e escreva o motivo — ele
   não pode ser alterado depois e não deve ter dado pessoal.
2. Todos os pedidos abertos e horários marcados vão para o destino. Os
   horários mantêm data e hora; com 48h ou mais, o cidadão precisa confirmar
   de novo (e recebe o lembrete, quando houver SMS). No `wpda`, ele vê "Local
   alterado".
3. Atendimentos abertos não são movidos: encerre-os na fila.
4. Clique em **Desativar**. Se um pedido novo tiver chegado entre o
   esvaziamento e a desativação, ela é recusada; esvazie de novo.

## ADRs relacionados

0017, 0018, 0019, 0021.
