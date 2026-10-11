# Prova técnica 19d — troca do token do SNCR no servidor

Data: 2026-10-10. Fontes: Manual da API SNCR 3ª ed. (set/2026), §2.2 e §2.3; Instruções de Integração v1.0 (jul/2026), §3 e §4.

## O que os documentos dizem [CONFIRMADO]
- Sem credencial de plataforma: "não solicita chaves fixas ou credenciais de parceiro" (Manual §2.2.2); credenciamento de plataforma "não será necessário" (Instruções §3).
- `GET /auth/login?client_url=…[&state=…]`: o backend da Anvisa gera nonce e state assinados (HMAC-SHA256, 5 min) e redireciona ao Keycloak com `kc_idp_hint=govbr`; o callback dele troca o `code` pelo token e redireciona ao `client_url` com `?session_id={token}` (uso único, 30 s).
- `GET /auth/token?session_id=` → 200 `{ "access_token", "token_type": "Bearer" }`; erro `{ "error": "Sessão inválida ou expirada" }`; o token sai da sessão após a leitura.
- Access token de 30 s, sem refresh token.
- `client_url` só em domínio `.br`; em dev, `localhost` e `127.0.0.1` em qualquer porta.
- "O session_id fica vinculado ao Origin que iniciou a autenticação, sendo exigido o mesmo Origin para a obtenção do access_token."
- O `state` do cliente é opcional; a volta ao `client_url` só menciona `session_id`.
- Os exemplos oficiais chamam `/auth/token` e a API pelo navegador (CORS liberado para `*.br`). Nenhum trecho proíbe ou confirma a chamada pelo servidor.
- Hosts citados: API em `https://api-gateway.{hmg,prd}.apps.anvisa.gov.br/api-sncr/api/v1/`; auth nos exemplos em `https://sncr-api.apps.anvisa.gov.br/api/v1/auth/...`.

## O que falta provar na homologação
1. O servidor consegue trocar o `session_id` (com o cabeçalho `Origin` igual ao do `client_url`)?
2. O pedido de bloco com o token, feito pelo servidor, responde 201 dentro dos 30 s?
3. O `state` volta no `client_url`?
4. `*.localhost` (ex.: `curitiba.localhost`) é aceito como `client_url` em dev?
5. Qual host serve `/auth/*` na homologação (o do gateway ou `sncr-api`)?

## Decisão (2026-10-10)
- **A sonda não foi feita.** Nenhuma chamada à homologação do SNCR foi montada nem executada; as perguntas 1–5 acima seguem **não confirmadas**.
- **Opção (a), escolhida pelo usuário:** o 19d segue com o SNCR simulado (`fake-sncr`, porta 8092 em dev, e o mesmo app nas specs). O desenho continua o do ADR 0034: o api troca o `session_id` pelo `access_token` no servidor (cabeçalho `Origin` do host da cidade) e pede o bloco na mesma requisição; as constantes ainda não provadas ficam em `Sncr::Client` (`SEND_ORIGIN`, host de `/auth`) e em `sncr.auth_url`.
- **Gate de go-live:** antes de ligar o SNCR real em qualquer cidade, provar na homologação, com login gov.br feito pelo usuário, que (1) o servidor troca o `session_id` com o `Origin` do `client_url`, (2) o pedido do bloco responde 201 dentro dos 30 s, e responder 3–5. Junto: confirmar a mensagem real de "faixa esgotada" (o cliente a reconhece por `/esgot/i`, sem texto documentado).
- **Risco do plano B:** se a troca no servidor não funcionar (o `session_id` é preso ao `Origin` do navegador e a Anvisa recusar a chamada do servidor), vale o plano B do ADR 0034 — o navegador troca o token, chama o SNCR e repassa os números ao api. Isso obriga a refazer `Sncr::Client`, o callback `POST /sncr/oauth/callback` e a tela Conta → SNCR do dashboard, e a rever o contrato §4/§12 (o api deixaria de ver o token e passaria a confiar em números entregues pelo navegador, com conferência de formato, faixa e unicidade).
