# Mascaramento de dados pessoais na API de manutenção — decisão de adiamento

- **Status:** Adiado
- **Data:** 2026-09-17
- **Contexto:** desenho da API GraphQL de manutenção (em andamento)
- **Relacionado:** ADR 0011 (identidade e MFA), ADR 0013 (segredos e cifra), ADR 0014 (auditoria, retenção e LGPD)

## Contexto

Está em desenho uma API GraphQL de manutenção, irmã da tela `/maintenance`, para
consultar e alterar dados dos módulos de cada cidade a partir de um frontend separado.
As decisões já tomadas que importam aqui:

- **Ambientes:** development e staging. Nunca produção.
- **Quem acessa:** pessoas (pelo frontend) e automação (tokens de serviço).
- **Identidade:** conta própria de mantenedor, separada de `Operator`.
- **Autorização:** papel único e total (superusuário). O mantenedor pode tudo,
  inclusive conceder e revogar o acesso de outros mantenedores.
- **Auditoria:** toda alteração é registrada com data, hora, login e módulo.

Durante o desenho, surgiu a proposta de que o superusuário também visse dados de
cidadão, **desde que mascarados**. Este documento registra essa necessidade e a
decisão de não implementá-la agora.

## A necessidade levantada

Regra proposta: ofuscar **85%** do conteúdo de:

- documentos (CPF, RG) e qualquer outro dado pessoal sensível;
- números de telefone;
- número e complemento de endereço, quando registrados.

## O que a análise encontrou

**Estado do schema em 2026-09-17.** Hoje não existe coluna de CPF, RG ou endereço.
Os dados pessoais de cidadão estão em dois formatos:

| Formato | Onde | Dificuldade |
|---|---|---|
| Campo estruturado | `conversations.phone`, destinatários em `alert_recipients` | Baixa: mascarar a coluna resolve |
| Texto livre | `messages.body`, `inbound_messages.raw` (payload do webhook), `consents.evidence`, `triages.context` / `triages.response` | Alta: o cidadão pode digitar CPF ou telefone em qualquer posição do texto |

**Cuidado com o termo "dado sensível".** Pela LGPD (art. 5º, II), dado de saúde é
dado pessoal **sensível**. As respostas de triagem já são isso hoje, não só quando os
módulos futuros entrarem. Por isso o gatilho de reabertura abaixo é definido pelo que
a API **expõe**, e não só pela chegada de módulos novos.

**Requisitos já identificados para quando o tema voltar:**

1. **Mascarar no servidor, sem forma de desligar.** Nem o superusuário recebe o valor
   original. Se algum parâmetro revelasse o valor, a máscara seria só cosmética.
2. **Impedir que a máscara volte ao banco.** Se o frontend lê `***.***.*89-00` e
   devolve o registro ao salvar, o dado real é sobrescrito pela máscara. Campos
   mascarados precisam rejeitar valores mascarados ou só aceitar alteração por
   mutation própria.
3. **A auditoria registra o nome do campo, nunca o valor.** Guardar antes e depois de
   um CPF no log transformaria o próprio log no vazamento.
4. **Decidir o que fazer com o texto livre.** Opções levantadas, sem escolha feita:
   - **A)** detectar padrões (CPF, RG, telefone, CEP) e mascarar dentro do texto. Não
     cobre formatos inesperados, como números por extenso ou com espaços;
   - **B)** não expor texto livre, só metadados (tamanho, direção, data, estado);
   - **C)** combinar A e B, com o texto completo (ainda mascarado) disponível só em
     query dedicada, com step-up de TOTP e evento de auditoria próprio.
5. **Definir o arredondamento dos 85%.** Um CPF tem 11 dígitos: 85% dá 9,35 dígitos
   ocultos, então sobram 1 ou 2 visíveis? A regra precisa ser determinística e igual
   para todos os tipos de dado.
6. **Garantir por spec.** Uma lista explícita de campos mascarados e um teste que
   falhe se um campo de dado pessoal entrar no schema fora dela, como já faz o spec
   "sem conteúdo de cidadão" da `/maintenance`.

## Decisão

**O mascaramento de dados pessoais fica adiado.** Não será desenhado nem implementado
nesta etapa da API de manutenção.

**Regra provisória, válida até o mascaramento existir:** a API GraphQL de manutenção
**não expõe dado pessoal de cidadão**, nem em leitura nem em escrita. Conversas,
mensagens, payloads de webhook, consentimentos e triagens aparecem, no máximo, como
contagens e metadados sem conteúdo. Vale a mesma regra da `/maintenance`, protegida
por spec.

Sem essa regra, adiar o mascaramento equivaleria a expor os dados sem máscara.

## Quando reabrir

Qualquer um destes eventos reabre o tema, **antes** de a mudança ser mergeada:

- surgir a necessidade de a API de manutenção ler ou alterar conteúdo de cidadão
  (texto de conversa, resposta de triagem, evidência de consentimento, telefone);
- entrar no ciclo um módulo que armazene documento (CPF, RG, CNS) ou endereço de
  cidadão, por exemplo agendamento (08), território (11) ou acompanhamento (13);
- staging passar a receber qualquer cópia de dados de produção.

Ao reabrir, os requisitos 1 a 6 acima são o ponto de partida, e a decisão vira um ADR
próprio (ver "Itens em aberto" no índice de ADRs).
