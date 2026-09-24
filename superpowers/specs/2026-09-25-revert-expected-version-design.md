# Recusar a reversão quando o mundo mudou desde a leitura — design

**Data:** 2026-09-25
**Status:** aprovado em conversa (2026-09-25)
**Afeta:** `apps/api` (um argumento no command e nas duas superfícies), `apps/dashboard` e `apps/maintenance` (mandar o token e tratar a recusa).

**Origem:** pendência deixada de propósito por `2026-09-24-revert-confirmation-design.md` §2. Aquele trabalho fechou o ciclo do lado da informação — as telas passaram a dizer qual versão de fato passou a valer. Ficou aberta a decisão: quando o mundo muda entre a leitura da tela e o clique, o servidor reverte o que estiver lá e conta depois. Este spec troca "contar depois" por "recusar antes", **só** nesse caso.

## 1. O que já está fechado, e o que não está

`Protocols::RevertActivation.call` já reconfere tudo sob lock e já recusa quando o alvo deixou de ser o que ele mesmo havia lido — *"a versão anterior mudou desde a leitura"* (`revert_activation.rb`, dentro da transação). **A janela entre a leitura do command e o lock está fechada.**

O que não está fechado é a janela anterior: entre o que a **tela** mostrou e o clique chegar ao servidor. `current` e `target` são resolvidos do zero dentro do `call`, a partir do nome — nada do que a pessoa viu viaja junto. Se outra pessoa ativou uma versão nesse meio tempo, a reversão acerta um par diferente do que estava na tela, com sucesso.

## 2. Escopo

**Dentro:**
- `expected_version:` opcional no command, conferido na leitura rápida — uma vez só, pelo motivo do §3.3;
- o argumento nas duas superfícies, e uma recusa própria;
- as duas telas mandando o token e tratando a recusa relendo a lista.

**Fora:**
- estender o token a Ativar e Aposentar;
- qualquer mudança no que a reversão faz **quando** ela acontece;
- reconfirmação de um clique ("reverter assim mesmo") — ver §3.1.

## 3. Decisões

### 3.1 Recusar, não reconfirmar

Na recusa a tela relê a lista e mostra o estado novo; não existe botão de "reverter assim mesmo".

O ato que se está protegendo troca o protocolo clínico em uso **sem assinatura nova**. Se o mundo mudou desde a leitura, a decisão foi tomada com informação errada, e reapresentar o estado custa segundos. Um botão de "assim mesmo" é o botão que as pessoas aprendem a clicar sem ler — transformaria a proteção em ritual exatamente na situação em que ela importa.

### 3.2 Um token só: a versão vigente

O cliente manda a **versão vigente que a tela mostrava** — aquela de onde se reverte, já visível no título do painel de confirmação. Não manda o alvo previsto.

Um token basta porque as condições se amarram: se a vigente ainda é a mesma **e** a ativação mais recente ainda é a `signed` dela, então `previous` também não mudou — qualquer ativação nova troca o `latest`, e aí o `call` já recusa por *"só se reverte uma ativação assinada"*. Mandar os dois números seria duas chances de divergir e nenhuma garantia a mais.

Tipo: **string** no painel da cidade e **Int** no GraphQL de manutenção, como `version` já é em cada superfície (mesmo argumento de `revertTargetVersion`).

### 3.3 Uma conferência só, e por quê

O token é conferido **uma vez**, na leitura rápida — que é onde ele tem valor: é ali que se compara o que a **tela** viu com o que o banco tem agora.

Uma segunda conferência sob lock seria código morto, e a razão precisa ficar escrita porque o instinto (o meu inclusive) diz o contrário. `current` é resolvido uma vez por `find_by(name:, status: "active")` e depois **travado**, não re-resolvido: é a mesma linha, e a `version` dela não muda. Se outra versão tiver sido ativada no meio do caminho, `Protocols::Activate` demove esta para `published` antes de ativar a outra — o índice único parcial em `status = 'active'` não admite duas —, e aí a checagem que **já existe** dentro da transação, `unless current.status == "active"`, dispara antes de qualquer coisa que o token pudesse pegar.

O comentário no código nomeia essa checagem, para que quem vier depois não "conserte" a ausência da segunda conferência.

### 3.4 Ausência do token é permitida — por enquanto

Sem token, o comportamento é byte por byte o de hoje. Isso é transitório, e §5 diz como deixa de ser.

## 4. O que muda, superfície por superfície

**Command:** `call(name:, by:, reason:, expected_version: nil, correlation_id: nil)`. Divergência devolve `Result.fail(:current_version_changed, message: ...)`, com a mensagem nomeando a versão que está em uso agora.

**Painel da cidade:** `POST /admin/api/protocols/revert` aceita `expected_version` no corpo. A recusa sai como **409**, o que exige acrescentar `current_version_changed: :conflict` ao `STATUS_FOR` de `ProtocolResultRendering` — hoje o mapa só traduz `forbidden` e `not_found`, e todo o resto cai em 422. Um conflito de concorrência não é um corpo inválido.

**API de manutenção:** a mutation ganha `expectedVersion: Int` (anulável neste passo), e a recusa aponta para esse argumento.

**Telas:** mandam o token; na recusa, relêem a lista e mostram *"a versão em uso mudou enquanto você lia: agora é a vN. Confira e decida de novo."* Sem número disponível na recusa, a frase sai sem número — nunca `undefined`.

## 5. Os três passos, e por que não são dois

1. o api aceita o token e recusa divergência; a ausência continua valendo;
2. dashboard e console passam a mandá-lo;
3. o api passa a recusar a ausência.

**Por que o frontend não pode ir primeiro.** No REST poderia: o api ignora campo desconhecido no corpo. No GraphQL, não — mandar um argumento que o schema não declara é erro de **validação**, e a consulta inteira é recusada. Quem tem de ir na frente ali é o api. Fazer o REST em dois passos e o GraphQL em três deixaria duas histórias convivendo, e daqui a seis meses alguém pergunta por que num lugar é obrigatório e no outro não. As duas superfícies seguem os mesmos três passos.

**O risco assumido:** entre o passo 1 e o passo 3 existe um caminho sem guarda. O passo 3 vira **issue no Project #1 no mesmo dia em que o passo 1 for mergeado** — não memória de quem estava na sala. O comentário no `call` e no argumento do GraphQL registra que esse passo existe, por que não pôde ser o primeiro, e que enquanto ele não vier a ausência do token é aceita.

## 6. Privacidade

Nada novo: número de versão de protocolo, que já circula nas duas telas.

## 7. Estratégia de teste

- **Domínio:** token igual à vigente passa; token diferente recusa com `:current_version_changed`; **sem token, os exemplos de hoje seguem inalterados** (é o que prova que o caminho antigo não mudou).
- **Domínio, o exemplo que importa:** monta o estado que a tela viu (vigente v2), ativa uma v3 **depois** dessa leitura, e então reverte mandando `expected_version` = 2. Sem o token isso reverteria alguma coisa com sucesso; com ele, recusa. É a corrida que o token existe para pegar, e é a única que ele pega sozinho — a que acontece depois, entre a leitura do command e o lock, já é recusada pelas checagens existentes (§3.3).
- **Painel da cidade (request):** divergência responde **409** com o código da recusa; token correto reverte; sem token, o comportamento de hoje.
- **Manutenção (request GraphQL):** divergência recusa apontando para `expectedVersion`; token correto reverte.
- **Telas (Vitest):** a recusa mostra a frase com o número novo e dispara a releitura da lista; uma asserção de que a mensagem nunca contém `undefined`.

## 8. Entrega

Um plano, quatro tarefas:

1. `expected_version:` no command, com a conferência e a recusa nova;
2. as duas superfícies do api (corpo REST com 409, argumento GraphQL);
3. o dashboard mandando o token e tratando a recusa;
4. o console de manutenção, idem.

O passo 3 do §5 é trabalho próprio, com issue própria — não entra neste plano.

## 9. Fora de escopo

- Token em Ativar, Aposentar, Publicar ou Assinar.
- Reconfirmação de um clique.
- Mostrar quem ativou a versão que apareceu no meio do caminho.
- Tornar a ausência do token uma recusa (é o passo 3, com issue própria).
