# Identidade da cidade e login do painel — design

**Data:** 2026-10-19
**Status:** aprovado (2026-10-19)
**Card:** rotasaude/api#73. Esta spec amplia o escopo dele, que antes cobria só o wpda e agora cobre também o login do dashboard.
**Afeta:**
- `apps/api` (plataforma):
  - Active Storage no banco da plataforma.
  - Brasão e nome oficial do órgão de saúde em `cities`.
  - Rota pública de identidade por host da cidade.
  - Rota do console para gravar a identidade.
  - MinIO como serviço de arquivos.
- `apps/admin`: campos de identidade no provisionamento e na tela da cidade.
- `apps/dashboard`: login novo (e telas de entrada irmãs, ver §5.4).
- `apps/wpda`: alimentar o `CityHeader`, que já existe.
- `docker-compose.yml`: serviço MinIO de dev.

**Base:**
- Decisão 6 do ciclo de interface (`superpowers/specs/2026-10-12-interface-cycle-foundation-design.md`): topo por cidade no wpda.
- Decisões 10 a 12 do mesmo ciclo: temas dos painéis.

## 1. Intenção

A equipe da prefeitura começa o turno pela tela de login do painel: atendentes, enfermeiros, médicos e administradores. Essa tela precisa dizer **em que cidade a pessoa está entrando** e passar a mensagem **"este é o sistema da sua prefeitura"**, com a marca Rota Saúde junto, sem disputar o primeiro plano.

O cidadão tem a mesma necessidade no wpda, e por isso existe o api#73. As duas telas passam a ler a mesma fonte de dado.

## 2. Objetivos, não objetivos e antiobjetivos

**Objetivos**
- Antes de qualquer sessão, o login do dashboard mostra o brasão da cidade, "Prefeitura de <nome> · <UF>" e o nome oficial do órgão de saúde.
- O wpda mostra brasão e nome no `CityHeader` desde a primeira tela. Isso fecha o critério do api#73.
- O operador da plataforma cadastra o brasão e o nome do órgão no `admin`, no provisionamento, e pode trocá-los depois.
- O dado tem uma fonte só, no api.

**Não objetivos**
- Marca gov.br por cidade. Continua pendente de confirmação legal, como já registrado no api#73.
- Cores da prefeitura no painel. A faixa usa o destaque do tema ativo.
- Mudar autenticação: sem MFA para usuário municipal, gov.br como está.
- Tema por cidade.

**Antiobjetivos** (falhas mesmo que "funcione")
- Uma segunda fonte de verdade sobre a cidade, como um mapa estático em algum front.
- Login quebrado ou vazio quando a cidade não tem brasão, quando a rota de identidade falha ou quando o MinIO está fora. O login nunca pode depender da identidade para funcionar.
- A rota pública vazar qualquer coisa além de identidade institucional. Ficam de fora `database_url`, chaves, status interno, `pec_url`, `record_mode` e contagens.
- O login carregar recurso de host de terceiro.

## 3. Decisões tomadas pelo usuário

1. O login atende à equipe da prefeitura começando o turno.
2. A tela mostra a cidade (nome, UF e brasão) ao lado da marca.
3. **Layout B com a saudação do C** (esboços em `.superpowers/brainstorm/`, `login-layout.html` e `login-layout-v2.html`): tela dividida, com a metade da prefeitura em primeiro plano, e saudação de turno no formulário.
4. O login do dashboard entra no escopo do api#73.
5. Quem cadastra o brasão é o **operador da plataforma, no `admin`, no provisionamento**.
6. Armazenamento por **Active Storage**.
7. Serviço de arquivos em staging e produção: **MinIO próprio**.
8. O nome do órgão de saúde é **cadastrado pelo operador, com o nome oficial de cada cidade** (por exemplo "Secretaria Municipal de Saúde" ou "Fundação Municipal de Saúde").

## 4. api

### 4.1 Dado

- **`cities`** (plataforma) ganha:
  - `health_department_name`: string, nulo permitido, até 120 caracteres.
  - O anexo `coat_of_arms` (`has_one_attached`).
- **Nome, UF e fuso** já estão em `cities` (`name`, `uf`, `time_zone`). Não se duplica nada em `city_profile`.
- **Tabelas do Active Storage** (`active_storage_blobs`, `_attachments`, `_variant_records`): ficam no **banco da plataforma**, via `platform_migrate`. *Escolha do autor:* o brasão é cadastro da plataforma, não dado do município. A instalação do Active Storage tem de respeitar a topologia multibanco: o modelo base do Active Storage conecta na plataforma, nunca no banco da cidade corrente.
- **Validação do brasão:**
  - Formatos: PNG ou WebP. SVG fica de fora, porque pode carregar script.
  - Tamanho máximo: 256 KB.
  - Dimensões: mínimo de 128 px no lado menor, máximo de 1024 px.
  - O `content_type` é conferido pelos bytes do arquivo, não pelo que o navegador declara.
  - *Escolha do autor:* não há processamento de imagem (sem `image_processing`/vips). O arquivo é servido como enviado. As dimensões são lidas do cabeçalho PNG (IHDR) e WebP (VP8/VP8L/VP8X) por código próprio, porque o analisador do Active Storage só extrai dimensões com vips.
- **Ligação do Active Storage com a topologia multibanco** (achados da revisão no código):
  - `ActiveStorage::Record` precisa de `connects_to` explícito para a plataforma. Sem isso, cai no `primary` (`rota_saude_no_city_selected`) e quebra com `UndefinedTable`. `CityConnection.with` só troca `CityRecord` e `SolidQueue::Record`.
  - `config.active_storage.draw_routes = false`. As rotas padrão `/rails/active_storage/*` redirecionariam para o MinIO.
  - Sem `AnalyzeJob` (`analyze: false` no anexo) e com purge **síncrono** (`purge`, não `purge_later`). Os jobs do Active Storage não herdam `ApplicationJob` e escapariam da guarda `PlatformQueue.check!`.
- **`cities.uf` aceita nulo.** Sem UF, todo texto "<nome> · <UF>" vira só "<nome>".

### 4.2 Rota pública de identidade

Ela fica no host da cidade, sem sessão, fora de `/citizen`, para servir o wpda e o dashboard.

- **`GET /city_identity`** responde com:
  ```json
  { "name": "Curitiba", "uf": "PR", "time_zone": "America/Sao_Paulo",
    "health_department_name": "Secretaria Municipal da Saúde",
    "coat_of_arms_url": "/city_identity/coat_of_arms?v=<checksum>" }
  ```
  - `health_department_name` e `coat_of_arms_url` vêm `null` quando ausentes.
  - O caminho tem de vir **antes** de qualquer rota que case por prefixo, e precisa ser declarado nos proxies do Vite do dashboard e do wpda (e no proxy de produção).
- **`GET /city_identity/coat_of_arms`** devolve os bytes do brasão. O api faz proxy do MinIO e nunca redireciona para o MinIO. Com isso:
  - o MinIO não fica exposto;
  - não há URL assinada que expira;
  - o login não carrega recurso de outro host.
- **Cabeçalhos da imagem:**
  - `Cache-Control: public, max-age=31536000, immutable`. O `?v=<checksum>` invalida o cache quando o brasão troca.
  - `X-Content-Type-Options: nosniff`.
  - `404` quando a cidade não tem brasão.
- **Cabeçalhos do JSON:** cache curto (`max-age=300`).
- **Cidade inexistente ou não ativa:** `404` sem corpo distintivo. Cidade `suspended` ou `archived` não expõe identidade. *Escolha do autor:* `provisioning` também devolve 404.
- **Só a plataforma.** O controller usa `skip_city_resolution` e resolve a cidade pelo host via `CityCatalog`, lendo só `cities`. Ele não passa por `CityResolution#within_city`, que responde 403 `city_suspended` e 503 `city_schema_behind` e abre o banco da cidade. Assim a identidade não cai junto com o banco da cidade, e a suspensa não vaza corpo distintivo.
- **Limite de taxa** (*escolha do autor*). O limite de `/passwords` (10 em 3 min por IP) estouraria numa UBS atrás de um único IP na troca de turno, ou num CGNAT de operadora.
  - `/city_identity` (JSON): 120 por minuto por IP, no `RateLimitStore` existente.
  - A imagem fica fora do limite, porque é imutável e cacheável.

### 4.3 Escrita pelo console

- **`PATCH /cities/:id/identity`**: multipart, no bloco `PlatformConsoleHost`, seguindo o padrão de `record_settings`. Recebe:
  - `health_department_name`;
  - `coat_of_arms` (arquivo);
  - `remove_coat_of_arms` (booleano).
- **Auditoria:** `Platform.audit` com o nome de evento novo `city.identity_updated`, declarado em `R18_PLATFORM_EVENT_NAMES`. O payload leva só o que mudou (nome do órgão, checksum novo ou remoção), nunca os bytes.
- **Provisionamento:** `POST /cities` não muda. O formulário do `admin` chama `identity` logo depois da criação (§6). *Escolha do autor:* isso evita transformar o provisionamento em multipart e mantém a criação em duas fases intacta.

### 4.4 MinIO

- **Dev:** serviço `minio` no `docker-compose.yml`, com volume nomeado e um bucket criado no boot (`rota-saude-dev`). Credenciais de dev no `.env` de exemplo. O Active Storage usa o serviço `S3` com `endpoint` e `force_path_style: true`.
- **Teste:** serviço `Disk` (`tmp/storage`). Nenhum spec depende do MinIO.
- **Staging e produção:** MinIO próprio, configurado por variáveis de ambiente (`MINIO_ENDPOINT`, `MINIO_BUCKET`, `MINIO_ACCESS_KEY_ID`, `MINIO_SECRET_ACCESS_KEY`), como as senhas de banco. **Gate de go-live:**
  - bucket criado;
  - variáveis de ambiente definidas;
  - backup do bucket;
  - o boot de produção falha se o serviço não estiver configurado. A checagem é de **presença** da configuração, não de conectividade. O job `production-boot` da CI recebe uma configuração fictícia do MinIO para continuar verde (*escolha do autor*).
- **Gem:** `aws-sdk-s3`.

## 5. dashboard: login

### 5.1 Layout (desktop, a partir de 1024 px)

**Metade esquerda: a prefeitura.**
- Fundo no destaque do tema ativo.
- Brasão grande sobre um quadro claro.
- "Prefeitura de", seguido de "<nome> · <UF>" em destaque.
- Embaixo, `health_department_name`.
- No rodapé, "operado com Rota Saúde", com o ícone da marca (`Brand.tsx`).

**Metade direita: o formulário.**
- Saudação de turno ("Bom dia.", "Boa tarde." ou "Boa noite.").
- A frase "Entre para começar o turno em **<nome> · <UF>**."
- E-mail, senha, **Entrar**, **Entrar com gov.br** e **Esqueci minha senha**.

### 5.2 Tela estreita (abaixo de 1024 px, inclusive 768 px)

- A metade da prefeitura vira uma faixa no topo, com brasão pequeno e "Prefeitura de <nome> · <UF>".
- O formulário vem logo abaixo.
- O "operado com Rota Saúde" desce para o rodapé.

### 5.3 Estados

- **Sem brasão:** no quadro do brasão entra o ícone da marca Rota Saúde. Nome e UF aparecem normalmente.
- **Sem `health_department_name`:** a linha não aparece. Não se usa texto genérico.
- **Rota de identidade falhou ou está carregando:**
  - O formulário aparece e funciona **imediatamente**. A identidade nunca bloqueia o login.
  - A metade esquerda mostra só a marca Rota Saúde.
  - A frase do formulário vira "Entre para começar o turno."
  - Não há spinner bloqueante nem mensagem de erro.
- **Saudação:** é calculada no fuso da cidade (`time_zone` da identidade).
  - 05h00–11h59: "Bom dia."
  - 12h00–17h59: "Boa tarde."
  - 18h00–04h59: "Boa noite."
  - Sem identidade, usa o fuso do navegador.
- **Ficam como estão hoje, só com o visual novo:**
  - erros ("E-mail ou senha inválidos.", "Não foi possível entrar…");
  - grant expirado;
  - gov.br indisponível;
  - recuperação de senha sem enumeração.

### 5.4 Telas irmãs

*Escolha do autor:* "Esqueci minha senha", `ResetPassword` e `AcceptInvitation` usam a mesma moldura dividida, trocando só o conteúdo da metade direita. São a mesma porta de entrada.

### 5.5 Tema e acessibilidade

- **Tema:** o login lê o tema salvo no navegador (`govbr` por padrão, `rota` ou alto contraste), com a mesma store do shell. A faixa usa `--accent` e os tokens do tema, nunca cor literal. Os guardas de literais e de contraste do `src/theme` têm de passar.
- **Acessibilidade:** WCAG 2.2 AA + eMAG, como os painéis.
  - O brasão tem `alt="Brasão de <nome>"`.
  - O ícone de reserva é decorativo.
  - A saudação não é `h1`. O `h1` acessível é "Entrar no painel de <nome>", ou "Entrar no painel" sem identidade, podendo ficar só para leitor de tela.
  - O foco inicial vai para o e-mail.
- **Estilo:** o estilo inline atual dá lugar às classes e tokens do ciclo de interface.

## 6. admin

- **Formulário de provisionamento:** ganha, opcionais, "Nome oficial do órgão de saúde" e "Brasão" (arquivo, com prévia). Depois que `POST /cities` dá certo, o front envia `PATCH /cities/:id/identity`.
  - Se só a identidade falhar, a cidade continua criada e a tela da cidade mostra o aviso "identidade não salva", com opção de tentar de novo. *Escolha do autor:* o aviso é só estado do front. Depois de recarregar, a seção "Identidade" mostra "sem brasão" e "sem nome do órgão", o que já basta para o operador perceber.
- **Cliente HTTP:** `apps/admin/src/lib/api.ts` força `Content-Type: application/json` sempre que há corpo. O PATCH multipart precisa de um caminho próprio que deixe o navegador definir o boundary.
- **Tela da cidade (`CityRecord`):** ganha a seção "Identidade", com prévia do brasão como o login vai mostrar, troca, remoção e edição do nome do órgão.
- **Validação no front:** espelha a do api (formato, tamanho, dimensões) e mostra a mensagem antes do envio. O api continua sendo quem decide.

## 7. wpda

- O `CityHeader` (`apps/wpda/src/modules/citizen/CityHeader.tsx`) passa a receber `name` e `coatOfArmsUrl` de `GET /city_identity`, carregado uma vez por sessão do navegador e compartilhado entre as telas que já usam o componente.
- A reserva "Rota Saúde" que já existe continua valendo para falha ou ausência de dado.
- `/city_identity` entra no proxy do Vite do wpda.

## 8. Estratégia de teste

**api (RSpec, `type: :request` explícito)**
- `GET /city_identity`:
  - cidade ativa com e sem brasão e com e sem nome do órgão;
  - cidade `suspended`, `archived` e `provisioning` dão 404;
  - host desconhecido dá 404;
  - o JSON tem **exatamente** as chaves da §4.2 (guarda contra vazamento).
- `GET /city_identity/coat_of_arms`:
  - bytes e `content_type` corretos;
  - cabeçalhos de cache e `nosniff`;
  - 404 sem brasão;
  - o checksum em `v` muda quando o brasão troca.
- `PATCH /cities/:id/identity`:
  - só operador autenticado;
  - rejeita SVG (inclusive SVG renomeado para `.png`), arquivo acima de 256 KB e imagem fora das dimensões;
  - remoção;
  - evento `city.identity_updated` auditado e declarado na R18.
- Active Storage na plataforma: spec que confirma que, com e sem `Current.city`, o blob grava no banco da plataforma. O risco real é o `primary` vazio, não o banco da cidade.
- `/rails/active_storage/*` não existe (roteamento dá 404).
- Cidade suspensa dá 404 em `/city_identity`, não 403.
- O limite de taxa vale para o JSON e não para a imagem.

**dashboard (Vitest + Testing Library)**
- Login com identidade completa, sem brasão, sem nome do órgão e com a rota falhando. Nos quatro casos, o formulário está utilizável desde o primeiro render.
- Saudação nas fronteiras (04h59, 05h00, 11h59, 12h00, 17h59, 18h00), no fuso da cidade, com relógio falso.
- Os fluxos atuais continuam verdes: erro de credencial, grant expirado, esqueci a senha, gov.br.
- Guardas de tema (literais, contraste, a11y) passam com o login novo.
- O polling e os timers de saudação são limpos ao desmontar (lição do `polling-vaza-entre-testes`).

**admin (Vitest)**
- O provisionamento envia a identidade depois da criação, e uma falha só da identidade mostra o aviso.
- A seção "Identidade" troca e remove o brasão.
- A validação do arquivo no front funciona.

**wpda**
- O `CityHeader` mostra brasão e nome com a rota respondendo, e a reserva com a rota falhando.
- O Playwright de topo continua verde nos perfis atuais.

**Manual em dev (com evidência)**
- Curitiba com brasão em `curitiba.localhost` (login do dashboard e wpda).
- Uma cidade sem brasão.
- O MinIO parado com o login ainda funcionando.

## 9. Fica para quem implementa

- Nome exato das classes de layout e a divisão em componentes da moldura.
- Proporção das metades (50/50 nos esboços).
- Texto do aviso de identidade não salva no admin.
- Como o front carrega e guarda a identidade (hook próprio por app). Não existe pacote compartilhado (ADR 0025).
- Ordem das entregas, desde que o api vá primeiro: dashboard, admin e wpda dependem dele.

## 10. Gates de go-live

Decisão do usuário: a entrega fica **pronta em dev**. Staging e produção dependem destes gates:

1. **Borda HTTP de staging e produção.** Encaminhar `/city_identity` e `/city_identity/coat_of_arms` ao `api` nos hosts da cidade (dashboard e wpda). A borda não está nestes repositórios: os `nginx.conf` do dashboard e do wpda servem só o próprio app, e `apps/wpda/deploy/production` está vazio.
2. **MinIO de staging e de produção**, que nascem juntos:
   - bucket;
   - variáveis de ambiente de cada ambiente;
   - backup do bucket.

## 11. Coordenação

- `api`: avisar a sessão "API" (dona do app) antes de começar.
- `wpda` e `admin`: avisar as sessões donas antes de mexer.
- Atualizar o api#73 com o escopo novo (dashboard), as decisões da §3 e o link desta spec.
- Os gates da §10 entram na lista de gates de go-live.
