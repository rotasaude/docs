# Agenda dos profissionais — rollout e recepção (módulo 17)

Runbook do operador e da recepção para o módulo 17 (ADR 0029): publicar a
agenda, preparar a cidade e conduzir a marcação no balcão.

## 1. Publicação

Ordem obrigatória:

1. `contracts` com a tag `protocols-v1.5.0` (bloco `scheduling` no protocolo).
2. `api`. Antes de cortar o tráfego, rode `bin/rails city:migrate:all` com a
   imagem nova. A migração `20261006210001`:
   - cria `appointment_types` (com a base da plataforma copiada), `schedule_templates`,
     `appointment_request_triages` e `appointment_notices`;
   - transforma todo horário existente em `legacy` (sem profissional);
   - dá prazo `created_at + 30 dias` aos pedidos existentes;
   - põe a trava de sobreposição (`EXCLUDE`) nos horários `slot`.

   Se encontrar horários `slot` sobrepostos (rollout interrompido ou SQL
   manual), ela aborta com `InheritedOverlap` listando os pares de ids. Decida
   cada par com a cidade antes de rodar de novo. Nunca migre uma cidade fora do
   rake: ela fica em 503.
3. `dashboard`, logo depois do `api`. O dashboard antigo funciona, mas mostra
   pedidos da triagem como "Encaminhado de null" e a prioridade como texto.
4. `wpda`. Contra um `api` antigo, o resultado da triagem trata o pedido como
   aberto.

Depois do deploy do dashboard novo, remova do `api` a chave antiga
`appointments` da agenda da unidade (ninguém mais a lê).

A migração é irreversível. O rollback é publicar a imagem anterior **sem**
reverter o banco: o módulo 08 continua funcionando sobre os horários `legacy`.

## 2. Preparar a cidade (`municipal_admin`, em Profissionais)

1. **Tipos de atendimento:** a base (`consulta_medica`, `consulta_enfermagem`,
   `consulta_odontologica`, `retorno`) já vem copiada. Ajuste a duração, desative
   os que a cidade não usa e crie os próprios, sempre com os CBOs que atendem.
2. **Modelos de agenda:** cada modelo divide o turno em faixas — demanda do dia,
   agendável (com tipo e duração da vaga) e bloqueada — e define o limite de
   encaixes por turno. Use a pré-visualização com uma ocupação de exemplo antes
   de salvar.
3. **Turnos:** ligue o modelo a cada turno. Turno sem modelo vira uma faixa
   agendável do tipo padrão do vínculo; sem tipo padrão, só aceita encaixe.
4. **Limite padrão de encaixes** (turno sem modelo): `city_profile.default_fit_in_limit`,
   padrão 2.

Mudar o modelo ou cancelar um turno nunca move um horário já marcado. Turno
cancelado deixa os horários dele como "precisa remarcar" na fila.

## 3. Transição

Um dia passa a usar a agenda quando a unidade tem turno nele. Em dia sem turno,
a recepção continua marcando como no módulo 08 ("marcação livre", horário
`legacy`). Os horários antigos não ganham profissional.

## 4. Balcão (recepção, em Atendimento)

**Fila de pedidos da unidade.** Ordem: atrasados primeiro, depois prazo e
prioridade. Marcas: `atrasado`, `faltou`, `sem confirmação`, remarcação pedida
pelo cidadão (com motivo, período preferido e a nota dele no detalhe) e
`precisa remarcar`.

**Marcar em vaga.** "Marcar horário" mostra os dias com vagas do tipo do pedido;
escolha o dia e a vaga. Duas recepções na mesma vaga: a segunda recebe
"vaga já ocupada" e a lista recarrega. O horário nasce confirmado.

**Encaixe.** Quando não há vaga, "Encaixe": escolha o turno e a hora de início e
escreva a justificativa (sem nome, telefone ou CPF; o texto não pode ser
alterado). O limite é por turno; ao atingir, o encaixe é recusado.

**Remarcar.** "Marcar horário" num pedido já marcado encerra o horário vivo como
movido e cria o novo.

**Pedidos sem unidade.** Pedidos da triagem de cidadão cujo bairro não tem
unidade de referência caem em "Pedidos sem unidade" (balcão de qualquer
unidade). "Atribuir unidade" leva o pedido para a fila da unidade escolhida.
Atribuição é definitiva.

**Não posso nesse horário.** Quando o cidadão pede outro horário no wpda, o
horário é cancelado e o pedido volta à fila com o mesmo prazo, o motivo e o
período preferido.

## 5. Lembrete da véspera

`Appointments::RemindJob` roda a cada 15 minutos e só age das 17h às 20h no fuso
da cidade, uma vez por horário (`reminded_at`). Sempre cria o aviso na caixa do
cidadão no wpda. SMS só com a chave de SMS da cidade ligada, provedor
configurado, opt-in do cidadão e lembretes não silenciados; falha do provedor
não repete.

Antes do go-live do SMS, configure timeout curto no provedor: o envio acontece
com o horário travado.

## 6. Problemas conhecidos

- Pedido "precisa remarcar" numa unidade sem turno não tem marcação livre;
  crie um turno ou encaminhe.
- Pedido com tipo que não existe na cidade não encontra vagas: crie o tipo ou
  ajuste o protocolo (o gate só avisa).
- Lembrete continua na caixa depois de cancelamento (rotasaude/api#46).
- Remarcação pedida em unidade desativada reabre o pedido nela (rotasaude/api#45);
  esvazie a unidade antes de desativar.
