# Frontend da manutenção — design

**Data:** 2026-09-18
**Status:** aprovado em conversa, aguardando revisão do documento
**Consome:** a API GraphQL de manutenção (`2026-09-17-maintenance-graphql-api-design.md`), no estado dos Planos 1–4 (identidade, tokens, administração, leitura de cidades)
**Hosts:** `maintenance.<env>.rotasaude.com.br` (frontend) e `maintenance-api.<env>.rotasaude.com.br` (API), só em development e staging

## 1. Objetivo

Dar ao mantenedor uma interface para o que a API de manutenção já oferece: entrar com senha e TOTP, aceitar convite, ler o catálogo e o detalhe de todas as cidades, administrar mantenedores e tokens de serviço, e consultar a auditoria. A escrita em cidade entra depois, junto com o Plano 5 reescrito.

## 2. Decisões

| # | Decisão | Escolha |
|---|---|---|
| F1 | Onde o app mora | Repo próprio `rotasaude/maintenance`, clonado em `apps/maintenance`. Identidade, host e cookie próprios não se misturam com `admin` ou `dashboard` |
| F2 | Cliente GraphQL | `fetch` próprio dentro do React Query, com tipos do `graphql-codegen` (`client-preset`). Sem Apollo nem urql |
| F3 | Escopo da primeira versão | Tudo o que a API já oferece (§6). Escrita em cidade fica para depois do Plano 5 |
| F4 | Transporte em dev | Proxy do Vite, com o frontend em `http://maintenance.localhost:5177` e o `Host` trocado para `maintenance-api.localhost` no proxy. Mesma origem para o navegador, sem CORS em dev |
| F5 | Base visual | Tokens e `global.css` próprios, no padrão do `dashboard`, com os componentes escritos no app. Sem biblioteca de componentes |
| F6 | Faixa de ambiente | Fixa, sempre visível, "DEVELOPMENT" ou "STAGING", em cores diferentes |
| F7 | Testes | Vitest com Testing Library para o app, e uma suíte pequena de Playwright contra o stack de dev real para os fluxos que só se provam de ponta a ponta |
| F8 | Navegação | Por estado (`useState` no `App`), como no `dashboard`. O convite é a exceção: lido do fragmento da URL ao carregar |

## 3. Stack e repositório

- Vite, React 18, TypeScript, React Query, Vitest com Testing Library e jsdom, Playwright para o e2e — as mesmas versões-base do `dashboard`.
- Dependências novas em relação aos outros apps: `graphql`, `@graphql-codegen/cli` com `client-preset` (dev), uma biblioteca de QR que gera no navegador, e uma biblioteca de TOTP só para o e2e (dev).
- Repo `rotasaude/maintenance`; commits em inglês, Conventional Commits, como nos demais.

## 4. Como o app chega à API

### Dev

- **Container** novo no compose da raiz: serviço `maintenance`, `node:22-alpine`, porta `${MAINTENANCE_PORT:-5177}:5173`, `VITE_API_PROXY_TARGET=http://api:3000`.
- **Proxy do Vite** (`vite.config.ts`) repassa `/graphql`, `/session` (inclui `/session/challenge`) e `/invitations/*` ao api, e **troca o `Host` para `maintenance-api.localhost`**: a rota do api só casa com o rótulo `maintenance-api`. É o único lugar onde o host muda. O `Origin` que o navegador mandou passa intacto. `allowedHosts: [".localhost"]`.
- **Ambiente do api em dev** (compose da raiz, que está fora do git):
  - `MAINTENANCE_API_ENABLED=true` — hoje a variável não está definida, e a API de manutenção está desligada em dev;
  - `MAINTENANCE_FRONTEND_ORIGIN` com padrão `http://maintenance.localhost:5177`.

  Variável nova não chega a container que já está rodando: o passo inclui recriar o api e conferir com `printenv`. O README do repo novo repete essas duas linhas para quem monta o ambiente do zero.

### Staging

- O bundle chama `https://maintenance-api.staging.rotasaude.com.br` direto: outro host, mesmo site (`rotasaude.com.br`), com o CORS que o api já tem por host.
- Uma única função, `apiUrl(path)`, decide entre o proxy (dev) e a URL absoluta (staging).
- `VITE_MAINTENANCE_ENV` (`development` | `staging`) alimenta a faixa de ambiente; `VITE_MAINTENANCE_API_URL` dá a URL absoluta em staging.

### Tipos

- `graphql-codegen` lê o schema por introspecção do api em dev — a introspecção só existe em development, e é o que se quer.
- O schema lido é commitado no repo (`schema.graphql`), para o typecheck e a CI não dependerem do api rodando.
- Quando o Plano 6 publicar o SDL em `contracts`, o codegen passa a ler de lá.

## 5. Identidade e sessão

**Cliente único** (`lib/api.ts`). Toda chamada:

- manda `X-Rota-Maintenance: 1` e `credentials: "include"`;
- nunca põe token nem segredo na URL;
- devolve erro tipado — `AuthRequired` (401), `RateLimited` (429), `GraphQLRefusal` (com `code`: `CITY_UNREACHABLE`, `CITY_ARCHIVED`, `CITY_READ_FAILED`, `CITY_OUT_OF_SCOPE`, `CITY_BUDGET_EXCEEDED`, `TOKEN_SCOPE_REFUSED`), `NetworkError`. Nenhuma tela lê status HTTP.

**Entrar**

1. E-mail e senha → `POST /session` → `{ mfa_required, session_id }`.
2. Código TOTP → `POST /session/challenge` com o `session_id`, que vive só em memória (nunca em `localStorage`).
3. Sessão aberta → carrega `me` e mostra o shell.

Qualquer falha das duas etapas mostra a mesma mensagem ("e-mail, senha ou código inválidos"), como a API, que não diferencia os casos para não revelar se uma conta existe ou está bloqueada. O 429 vira "muitas tentativas, aguarde".

**Convite e matrícula**

1. O link chega como `…/#token=…`. Ao carregar, o app lê o fragmento, guarda o token em memória e **limpa a URL** com `history.replaceState`, para que ele não fique no histórico.
2. `POST /invitations/enroll` com o token → `otpauth_uri`. O QR é **gerado no navegador**; o segredo nunca vai a um serviço externo. A chave aparece também em texto.
3. Senha, confirmação e o primeiro código → `POST /invitations/accept`. Sucesso leva à tela de entrar: a matrícula não abre sessão, e o primeiro login já exige o TOTP.

**Sessão viva**

- Ao abrir o app, `GET /session` decide entre o shell e a tela de entrar.
- Qualquer 401 no meio do uso (30 min parado, 8 h no total) limpa o cache do React Query e volta para a tela de entrar com "sessão expirada". Nada do que estava na tela sobrevive.
- Sair → `DELETE /session` e cache limpo.

**Step-up**

- Convidar mantenedor e criar token pedem o TOTP do momento num campo da própria ação.
- A API consome cada código uma vez: o campo avisa "se acabou de entrar, espere o próximo código".
- Falha no step-up conta para o bloqueio da conta, e a mensagem diz isso.

## 6. Telas

**Shell.** Faixa de ambiente no topo, fora de qualquer rolagem. Cabeçalho com o e-mail do mantenedor, "sair" e a navegação: Cidades, Mantenedores, Tokens, Auditoria. A tela ativa e a cidade aberta vivem em estado no `App`.

**Cidades.** Tabela do catálogo (`cities`): slug, nome, UF, status, schema atrasado, com filtro por status. Uma consulta, nenhuma conexão com cidade. Clicar abre o detalhe.

**Detalhe de cidade.**

- Topo: campos de plataforma e canal de WhatsApp, numa consulta leve.
- Abas, cada uma com a **sua** consulta, carregada só quando aberta: Perfil, Protocolos, Destinatários de alerta, Contas, Contagens, Operação. Cada campo do banco da cidade abre uma conexão; por aba, abre-se só o que se olha.
- Erro isolado por aba: cidade inalcançável, arquivada ou com falha de leitura mostra o código e a mensagem da API naquela aba; o resto da página segue.
- Botão "atualizar" por aba. Sem atualização automática.

**Mantenedores.**

- Lista: e-mail, estado, matrícula.
- Convidar (e-mail e TOTP). O link do convite não volta pela API: a tela diz que a entrega é pelo `rake maintainer:invite` no servidor, enquanto não houver mailer.
- Desativar, com confirmação. As recusas da API (a si mesmo, o último ativo) aparecem como erro do formulário.

**Tokens de serviço.**

- Lista: nome, acesso, cidades, validade, revogado.
- Criar: nome, acesso (`read` | `read_write`), cidades (vazio = todas, com o aviso escrito), validade de até 90 dias, TOTP.
- O segredo aparece **uma única vez**, num painel com "copiar" e o aviso "não será mostrado de novo". Ao fechar, é apagado da memória e do cache do React Query, e nunca entra em consulta posterior.
- Revogar, com confirmação.

**Auditoria.** Lista de `auditEvents` com os filtros e a paginação que a API oferece (data, módulo, resultado, mantenedor). O id do mantenedor é resolvido para e-mail pela lista de mantenedores. Somente leitura.

**Regra transversal.** O frontend mostra só o que a API devolve, e não guarda nada em `localStorage` além de preferência que não seja dado.

## 7. Estratégia de teste

**Vitest com Testing Library** (API simulada)

- Cliente: header e `credentials` em toda chamada; 401 → `AuthRequired` e cache limpo; 429 → `RateLimited`; cada código GraphQL → erro tipado; nenhuma URL com token.
- Convite: token lido do fragmento, URL limpa.
- Segredo do token: aparece uma vez; depois de fechar o painel, não existe no cache nem no DOM.
- Detalhe de cidade: uma aba com erro não derruba as outras.
- Faixa de ambiente: texto e cor por `VITE_MAINTENANCE_ENV`.

**Playwright** contra o stack de dev real (`npm run e2e`)

- Preparação: o teste cria o próprio mantenedor com `rake maintainer:invite` no container do api (e-mail único por execução) e calcula o TOTP a partir do `otpauth_uri`.
- Fluxos: convite → matrícula → aceite; entrar; catálogo e detalhe de `curitiba` com uma aba; criar token (segredo uma vez), recarregar e ver que não volta, revogar; sair e a consulta seguinte dar 401.
- Custo aceito: cada execução deixa um mantenedor e linhas de auditoria no banco de dev (a auditoria só aceita acréscimo). Documentado no README.
- Só local nesta versão: na CI, exigiria subir api, Postgres e cidades.

**CI do repo** — `typecheck`, `vitest`, `build`, e uma checagem de que os tipos gerados batem com o `schema.graphql` commitado.

## 8. Fora de escopo

- Telas de escrita em cidade (Plano 5 reescrito).
- Hospedagem em staging (servidor do build estático, TLS, DNS): spec própria quando a infraestrutura de staging existir. O build sai pronto para ela.
- Mailer do convite.
- Produção.
- SDL lido de `contracts` (Plano 6).
- Outros idiomas: só português.
- Navegação por URL (link direto para uma cidade): descartada na escolha F8; reabre se o uso em incidente pedir.
