# Mostrar a versão-alvo da reversão — design

**Data:** 2026-09-23
**Status:** aprovado em conversa (2026-09-23)
**Afeta:** `apps/api` (um método de domínio e duas leituras), `apps/dashboard` e `apps/maintenance` (uma frase em cada painel de confirmação).

**Origem:** pendência registrada nas fatias das telas de assinatura. A reversão de emergência (ADR 0016, S10) devolve a cidade para a versão ativada imediatamente antes da vigente, mas **nenhuma** das duas superfícies diz **qual** é essa versão: o painel explica a regra e o servidor confirma no escuro. É o ato mais arriscado do ciclo — o único que muda o protocolo em uso sem assinatura nova — e é feito numa urgência clínica, quando ninguém tem tempo de ir ao histórico conferir.

## 1. Escopo

**Dentro:**
- um método de domínio que devolve a versão-alvo da reversão;
- `revertible?` passa a derivar desse método, para não existirem duas respostas;
- `/admin/api/protocols` e `/admin/api/protocols/:id` (painel da cidade) e `city.protocolVersions` (API de manutenção) expõem a versão-alvo;
- o painel de confirmação da reversão, nas duas telas, nomeia a versão que passará a valer.

**Fora:**
- mudar o comportamento da reversão. Quem decide continua sendo `Protocols::RevertActivation.call`, que trava as linhas e reconfere tudo sob lock;
- histórico de ativações na tela (quem quiser a linha do tempo tem o painel de eventos);
- reversão encadeada, que o ADR 0016 já proíbe.

## 2. Decisões

1. **Uma fonte só.** `Protocols::RevertActivation.revert_target(protocol)` devolve o `ProtocolDefinition` que voltaria a valer, ou `nil`. `revertible?` passa a ser `revert_target(protocol).present?`. Hoje são duas consultas parecidas; se divergirem, a tela habilita o botão prometendo uma versão e a API reverte para outra.
2. **As duas superfícies.** O mantenedor executa o mesmo ato, com alcance em todas as cidades e menos conhecimento local — é quem mais precisa do número. O método é escrito uma vez; cada leitura ganha um campo e cada tela uma frase.
3. **A leitura é previsão, não promessa.** Sem lock, o alvo pode mudar entre a leitura e o clique (outra ativação comita no meio). O texto da tela diz "deve voltar", e a recusa da API continua aparecendo inline quando o mundo mudou. Mesma regra que já vale para as contagens de assinatura.
4. **Só o número da versão** viaja para as telas. Nada de conteúdo do protocolo, nada de definição.

## 3. Domínio (apps/api)

`Protocols::RevertActivation`:

```
revert_target(protocol) -> ProtocolDefinition | nil
```

Devolve a versão-alvo **exatamente** quando as três condições que `call` exige estão satisfeitas:
- `protocol.status == "active"`;
- a ativação mais recente do nome é desta versão e tem `kind == "signed"`;
- existe uma ativação anterior, e a versão dela está `published`.

Reusa `activation_history`, como `revertible?` já faz. `revertible?` fica no lugar (há chamadores) e passa a delegar.

## 4. Leitura

**Painel da cidade** (`Admin::ProtocolsQuery`): cada versão, na lista e no detalhe, ganha

```json
"revertTargetVersion": "1"
```

— string, como `version` já é ali, e `null` quando não há alvo. `revertible` continua, porque o dashboard já o consome.

**API de manutenção** (`Maintenance::Types::ProtocolVersionType`): o tipo ganha

```graphql
revertTargetVersion: Int
```

— inteiro, como `version` é nesse tipo, e nulo quando não há alvo. Calculado dentro do mesmo `inside` que já monta o estado de assinatura, porque a conexão da cidade fecha ao sair do bloco.

## 5. Telas

**Dashboard** (`src/modules/Protocols.tsx`): a `description` do `SensitiveAction` de reverter passa a nomear a versão:

> "A cidade deve voltar para a versão N, que estava em uso antes desta. A versão atual sai de uso. Não encadeia: só volta um passo."

Sem alvo na leitura, o botão não aparece (é o que `revertible` já decide), então a frase não precisa de variante para "alvo desconhecido".

**Manutenção** (`src/screens/ProtocolsTab.tsx`): o mesmo, no lugar do aviso atual que só explica a regra.

## 6. Privacidade

Nada novo: número de versão de protocolo, que já circula nas duas telas. Nenhum dado de cidadão, nenhum e-mail.

## 7. Estratégia de teste

- **Domínio:** `revert_target` devolve a versão anterior quando as três condições valem; devolve `nil` em cada uma das três violações (versão não ativa; ativação corrente não assinada, isto é a linha-base; anterior não `published`); e `revertible?` concorda com `revert_target.present?` em todos esses casos — é o que garante a fonte única.
- **Painel da cidade (request):** uma versão ativa com ativação anterior assinada responde `revertTargetVersion` igual à versão anterior, na lista e no detalhe; sem alvo, responde `null`.
- **Manutenção (request GraphQL):** o mesmo pelo `city.protocolVersions`, com o campo inteiro.
- **Dashboard (Vitest):** o painel de reverter mostra o número que veio da leitura, e não um texto fixo.
- **Manutenção (Vitest):** idem.

## 8. Entrega

Um plano, três tarefas:

1. `revert_target` no domínio, com `revertible?` delegando — e as duas leituras (painel da cidade e manutenção) expondo o campo;
2. o painel do dashboard nomeando a versão;
3. o painel da manutenção nomeando a versão.

## 9. Fora de escopo

- Mostrar quem ativou a versão-alvo, ou quando.
- Linha do tempo de ativações na tela.
- Avisar por e-mail que houve reversão (os dois avisos de segurança existentes são de segundo fator; reversão é ato de protocolo, com evento de domínio próprio).
