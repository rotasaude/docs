# Confirmar qual versão passou a valer depois da reversão — design

**Data:** 2026-09-24
**Status:** aprovado em conversa (2026-09-24)
**Afeta:** `apps/api` (um parâmetro no wrapper das mutations de cidade e um campo na mutation da reversão), `apps/dashboard` e `apps/maintenance` (a frase de sucesso de cada um).

**Origem:** pendência registrada na revisão final de `2026-09-23-revert-target-design.md`. Aquele trabalho fez as duas telas nomearem a versão que a reversão **deve** restaurar. Ficou aberto o outro lado do ato: depois que ele acontece, nenhuma superfície diz qual versão a cidade **está** usando.

Não é só silêncio. Hoje as duas telas nomeiam o número **errado**:

```
Reverter concluído: triage-respiratoria v2
```

O `v2` é a versão de onde se reverteu — a que acabou de sair de uso. A que passou a valer é a anterior, e não aparece. `apps/dashboard/src/modules/Protocols.tsx:196` e `apps/maintenance/src/screens/ProtocolsTab.tsx:146` interpolam a linha sobre a qual a pessoa agiu, o que é correto para publicar, ativar e aposentar, e é o oposto do certo para reverter.

## 1. Por que isso importa mais do que parece

A reversão é chamada **por nome**, sem versão esperada, e `Protocols::RevertActivation.call` trava as duas linhas e **reresolve** o par sob lock — de propósito: é o que torna a operação segura contra uma ativação concorrente. A consequência é que o par revertido pode não ser o que o painel previu, e hoje isso termina em sucesso silencioso.

O spec anterior aceitou essa janela de propósito (§3 dele: "a leitura é previsão, não promessa", e por isso o texto diz "deve voltar"). O que faltou foi fechar o ciclo do outro lado. Sem isso, a única maneira de saber o que está em uso depois de uma reversão de emergência é recarregar a tela e reler a lista — numa urgência clínica, que é exatamente quando esse botão é usado.

## 2. Escopo

**Dentro:**
- um caminho para uma mutation de cidade da API de manutenção devolver dado do command;
- `revertedToVersion` na mutation da reversão;
- as duas frases de sucesso nomeando a versão efetivada;
- quando ela difere da prevista, as duas telas dizem as duas coisas.

**Fora:**
- aceitar versão esperada em `call` e recusar quando o mundo mudou entre a leitura e o clique. Seria o conserto "de verdade" da corrida, mas muda o contrato do ato mais perigoso do ciclo, e numa urgência recusar pode ser pior do que reverter para outro alvo e dizer qual foi. Merece spec próprio;
- estender a confirmação a Ativar e Aposentar, onde a versão sobre a qual se agiu já é a versão certa;
- qualquer mudança no comportamento da reversão. `call` fica intocado.

## 3. Decisões

1. **O dado já existe no domínio.** `call` devolve `Result.ok(protocol_definition:)`. Nada de consultar de novo depois do ato: o que a tela afirma é o que o command decidiu sob lock.
2. **O painel da cidade não precisa de backend.** `render_protocol_result` (`app/controllers/concerns/protocol_result_rendering.rb:12`) já responde `{ok: true, protocol: {name, version, status}}`. Quem descarta é o cliente: `revertProtocol` é `Promise<void>`.
3. **A API de manutenção precisa.** `Maintenance::BaseMutation#audited` (`base_mutation.rb:81`) descarta o `Result` e devolve o literal `{ ok: true, errors: [] }` — hoje **nenhuma** mutation de cidade tem por onde devolver nada.
4. **O caminho de volta abre no `in_city`, não no `audited`.** `audited` é por onde passa a auditoria de toda mutation de manutenção, e "o command devolveu dado" não é assunto dele. `in_city` já conhece `Result` (inspeciona `result.failure?` e `result.message`), então é onde a tradução Result → payload naturalmente mora. Mecanismo geral o bastante para a próxima mutation de cidade que precisar, estreito o bastante para não tocar o wrapper de auditoria.
5. **A comparação com o previsto é no cliente.** O servidor nunca soube o que o painel previu; a tela sabe, porque tinha o `revertTargetVersion` na mão quando abriu o painel de confirmação. Nada de mandar a previsão ao servidor só para ele devolvê-la.
6. **Divergência é fato, não alarme.** Quando o número efetivado difere do previsto, a frase diz os dois, com o mesmo peso visual do sucesso comum. Divergir é raro e legítimo — significa que o sistema funcionou como devia diante de uma ativação concorrente. Transformar isso em alerta destacado treina a pessoa a ignorar alerta.
7. **Sem número, sem frase inventada.** Se a resposta não trouxer a versão, a mensagem sai sem número. Nunca `undefined` na tela: foi exatamente o modo de falha que a revisão final do spec anterior pegou em `ProtocolsTab`.

## 4. apps/api

**`Maintenance::CityMutation#in_city`** ganha um parâmetro opcional que mapeia o `Result` do command em chaves extras do payload. Mutation que não passa o parâmetro continua devolvendo `{ok:, errors:}`, byte por byte como hoje. O mapa só é chamado no caminho de sucesso — numa recusa não há `Result` de sucesso para ler.

**O payload de `Maintenance::Mutations::RevertProtocolActivation`** ganha:

```graphql
revertedToVersion: Int
```

— inteiro, como `version` é nesse tipo (mesmo argumento do `revertTargetVersion`), e nulo quando a mutation não chegou a reverter (recusa). Só o número: nada de definição ou conteúdo de protocolo.

**Painel da cidade:** nenhuma mudança.

## 5. apps/dashboard

`revertProtocol` (`src/lib/api.ts`) passa a devolver o protocolo que veio na resposta em vez de descartá-lo. A mensagem de sucesso da reversão nomeia a versão efetivada; as demais ações do ciclo seguem com a frase de hoje.

## 6. apps/maintenance

O mesmo, lendo `revertedToVersion` do payload da mutation.

## 7. O que as telas dizem

Coincidindo com o previsto:

> Reverter concluído: a cidade está com triage-respiratoria v1.

Divergindo:

> Reverter concluído: estava previsto v1; a cidade está com v3.

Sem número disponível:

> Reverter concluído.

## 8. Privacidade

Nada novo: número de versão de protocolo, que já circula nas duas telas. Nenhum dado de cidadão, nenhum e-mail, nenhum conteúdo de protocolo.

## 9. Estratégia de teste

- **Mutation (request GraphQL):** uma reversão bem-sucedida responde `revertedToVersion` igual à versão que passou a valer; uma recusa responde nulo e não inventa número.
- **Mecanismo do `in_city`:** uma mutation de cidade que NÃO passa o mapa continua respondendo exatamente `{ok, errors}` — é o que garante que abrir o caminho não mudou as outras; e o mapa não é chamado no caminho de recusa.
- **Dashboard e manutenção (Vitest), os três ramos em cada:** efetivada igual à prevista (frase com um número); efetivada diferente da prevista (frase com os dois, e o teste monta previsão e resultado como dados independentes, senão não prova nada); resposta sem o número (frase sem número). Em todos, uma asserção de que a mensagem não contém `undefined`.
- **Painel da cidade (request):** nenhuma mudança de contrato para cobrir — a resposta já traz o protocolo, e a spec dela já existe.

## 10. Ordem de deploy

**O `api` sobe antes do `maintenance`.** Campo anulável novo é aditivo: um cliente que não o conhece — o dashboard, ou um bundle antigo do maintenance — não quebra. O inverso quebra feio: um bundle novo do maintenance contra um api sem o campo faz o graphql-ruby recusar a consulta inteira na validação, e a reversão de emergência fica impossível até o api subir, com uma mensagem de erro que não ajuda ninguém. Nada chega a ser escrito (a validação acontece antes da execução), então é seguro — mas é indisponibilidade do ato mais urgente do ciclo. O dashboard é independente: ele não depende de nada do api nesta série.

## 11. Entrega

Um plano, três tarefas:

1. o parâmetro no `in_city` e o `revertedToVersion` na mutation, com os specs de requisição;
2. a frase do dashboard, incluindo o retorno de `revertProtocol`;
3. a frase da manutenção.

## 12. Fora de escopo

- Mostrar quem ativou a versão que passou a valer, ou quando.
- Reverter de novo a partir da mensagem de sucesso.
- Avisar por e-mail que houve reversão (segue fora, como no spec anterior).
- Estender o mecanismo do `in_city` a outras mutations agora, sem um segundo caso real que o justifique.
