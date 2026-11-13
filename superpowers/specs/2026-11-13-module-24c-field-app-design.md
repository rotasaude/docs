# Módulo 24c — App de campo offline — design

**Data:** 2026-11-13
**Status:** aprovado pelo usuário (2026-11-13)
**Afeta:**
- `rotasaude/campo` (**repositório novo**): o app de tablet offline para o ACS e os demais agentes de campo da Tabela 7 do LEDI.
- `rotasaude/contracts`: `field-sync` v1.2.0 (MINOR): item `family`, coordenadas do imóvel e da visita, manifesto do mapa e referência offline, `version` no `accepted`, `device_suspended`, `cpf_or_reason_required` (Q71 do 24b).
- `apps/api`:
  - **cidade:** coordenadas do domicílio (24a) e da visita (24b), cifradas; latitude e longitude nas fichas LEDI de cadastro domiciliar (tipo 3) e de visita (tipo 8); item `family` no envio; rota do mapa para o aparelho e para a sessão do dashboard;
  - **plataforma:** arquivo de mapa por cidade (geração, versão, resumo SHA-256), a partir de uma cópia própria do mapa base do Brasil.
- `apps/dashboard`: mapa da microárea com os domicílios e a correção do ponto do imóvel.
- `apps/maintenance`: situação do mapa de cada cidade e "gerar de novo".

**ADR:** `docs/adr/0053.md` · **Módulo:** `docs/modulos/24--domiciliar-e-coletivo.md` · **Pesquisa:** `docs/pesquisa/2026-11-13-app-de-campo-24c.md`
- **ADRs relacionados:** 0020 (banco por cidade), 0026 (exclusão LGPD), 0028 (LEDI), 0041 (núcleo e ferramenta; stack do app), 0042 (cadastro territorial), 0052 (visita, atividade coletiva e contrato de sincronização v1).
- **Módulos relacionados:** 24a (cadastro), 24b (visita, atividade, "quem visitar", aparelho de campo).

**Fora desta entrega:**
- iPad e iOS (a stack permite depois, sem reescrever);
- tablet compartilhado por vários profissionais;
- sincronização em segundo plano (só com o app aberto);
- fechar o dia pelo app (o job de madrugada do 24b fecha; o ACS também pode fechar pelo dashboard);
- agendamento de visita, PSE, IVCF-20 (como no 24b);
- rota até o domicílio (navegação curva a curva);
- trabalho de campo do agente de endemias além da ficha de visita (módulo 35).

## 1. Intenção

O ACS sai para a microárea com o tablet, sem depender de sinal: vê quem visitar e onde fica no mapa, cadastra domicílios, famílias e moradores, registra visitas e atividades coletivas, e tudo chega ao Rota Saúde quando houver rede. O núcleo continua a única fonte de verdade (ADR 0041); o app é a ferramenta.

**Por quê:**
- 11 de 15 editais pedem app offline para tablet.
- O 24b deixou o contrato de sincronização v1 e o lado do api prontos e testados sem app (decisão 4A).
- O e-SUS Território, gratuito e oficial, já faz cadastro e visita offline com mapa. O diferencial do nosso é o resto do Rota Saúde no bolso do ACS: "quem visitar" pelas linhas de cuidado, a visita contando nas linhas e na linha do tempo, e o cadastro do 24a com a guarda de duplicado.

**Sucesso:**
- Um dia de campo inteiro sem sinal: o ACS cadastra, visita e registra atividade, e ao voltar sincroniza sem perder nada e sem duplicar.
- O conflito com o que o posto mudou no meio do caminho se resolve sozinho quando os campos não se cruzam, e pergunta só o que se cruza.
- O mapa da microárea abre offline, com os domicílios e quem visitar.
- Visita e domicílio chegam ao SIAPS com a coordenada.

**Antiobjetivos:**
- Perder trabalho do ACS (item pendente apagado sem ele saber).
- Dado de saúde ou de pessoa legível fora do app (banco em claro, log, captura de tela, backup do Android).
- Posição do ACS, do imóvel ou do morador saindo para serviço de terceiro (mapa, geocodificação, telemetria, relatório de falhas).
- O app virar fonte de verdade ou ter regra de negócio diferente do api.
- Tablet perdido com o território legível.

## 2. Decisões

### Decisões do usuário

| # | Decisão |
|---|---|
| 1A | **Uma entrega só:** cadastro domiciliar e individual (com família), visita, atividade coletiva e conflito, tudo offline. |
| 2A | **Mapa offline aberto:** base do OpenStreetMap, um arquivo por município servido pelo próprio Rota Saúde; o ACS vê domicílios e quem visitar no mapa e marca o ponto do imóvel. Nenhum serviço de mapa de terceiros. |
| 3B | **Conflito por mescla campo a campo:** o que só o ACS mudou entra sozinho; o campo que o posto e o ACS mudaram pergunta qual valor fica. |
| 4A | **Um tablet por profissional:** um usuário por aparelho, cadastrado pelo QR do 24b, com PIN próprio do app. |
| 5A | **Mapa como um arquivo por município** (PMTiles) servido pelo Rota Saúde; a primeira task é um teste rápido de que o app abre o arquivo local. Se falhar, a decisão volta ao usuário (alternativa: servidor de mapa próprio com pacotes por região). |
| 6A | **Revogação automática guarda o trabalho:** o api distingue revogação manual de automática; na manual o app apaga tudo, na automática guarda a fila cifrada e apaga só o território (decisão 11). |

### Decisões do coordenador (desenho)

| # | Decisão |
|---|---|
| 1 | **Stack (ADR 0041, versões da pesquisa):** Expo SDK 58 (React Native 0.88), TypeScript, nova arquitetura; `expo-sqlite` com `useSQLCipher`; `expo-secure-store`; `@maplibre/maplibre-react-native` 11.x com config plugin; `expo-location`; `expo-camera` para o QR. Nada de Expo Go: build de desenvolvimento e prebuild. Tipos gerados dos schemas do `contracts` (`field-sync`), presos por versão. |
| 2 | **Build e distribuição:** build local (Gradle), nunca o serviço de build na nuvem do Expo; APK/AAB assinado com chave guardada fora do `android/` gerado. Distribuição como app privado no Google Play gerenciado da cidade, por causa da verificação de desenvolvedor Android obrigatória no Brasil desde 2026-09-30. Um app só para todas as cidades: a cidade vem do QR. |
| 3 | **Cadastro do aparelho:** o app lê o QR do 24b (`rotasaude-field:v1:` + host e token), chama `GET /field/v1/session` e pede o PIN novo (6 dígitos, duas vezes). O token fica no SecureStore; nunca em log, tela ou arquivo. |
| 4 | **Banco cifrado:** SQLCipher com chave aleatória de 32 bytes (chave bruta, sem PBKDF2), gerada no cadastro e guardada no SecureStore sem `requireAuthentication`. Na abertura, o app confere `PRAGMA cipher_version`; sem resposta, recusa abrir (o banco em claro nunca é usado). Perder a chave (desinstalar) = perder o banco local; o app avisa quando há pendente e a sincronização é o backup. |
| 5 | **PIN e bloqueio:** PIN pedido ao abrir e depois de 5 minutos em segundo plano. O PIN não cifra o banco (a chave do item 4 cifra); ele destrava o app. Guardado só como resumo com sal. 5 erros seguidos travam o app até um QR novo do **mesmo** usuário (gerado no dashboard com segundo fator, o que revoga o token antigo); o pendente fica. QR de outro usuário com pendente → recusado até a sincronização do pendente; sem pendente, apaga tudo e cadastra o novo. |
| 6 | **Proteção da tela e do aparelho:** `FLAG_SECURE` (sem captura de tela nem miniatura nas recentes); `allowBackup=false`; sem relatório de falhas e sem telemetria de terceiros; logs do app sem dado de pessoa (só ids e códigos). |
| 7 | **Modelo local:** o tablet guarda o que o download entrega, como entrega: `micro_area`, `household` (com famílias e condições de moradia), `registration`, `who_to_visit`, cada um com `id`, `version` e o corpo. Cada cadastro guarda também a **versão-base** (o corpo e a versão baixados por último), usada na mescla (3B). Índices locais para busca por nome, nascimento, mãe, rua e número (CPF e CNS não descem; decisão 14). Regra de negócio fica no api; o app espelha só as regras de tela que o api expõe (`domains-v1.json` e a referência da decisão 24). |
| 8 | **Fila de envio:** toda gravação local vira um item da fila com `client_uuid` (UUID v4 gerado no tablet), tipo e corpo, na ordem em que foi feita. O `accepted` de `household`, `family` e `individual_registration` passa a devolver `version` (v1.2.0); o app atualiza a base local com ela e os itens seguintes do mesmo domicílio vão com a versão nova. Envio em lotes de até 50, na ordem; o item só sai da fila com `accepted` ou `duplicate`. `rejected` fica na fila como "precisa de atenção", com a frase do código; o ACS corrige e reenvia com o mesmo `client_uuid` (o api reprocessa recusados). Itens que citam outro (`patient_client_uuid`, `household_client_uuid`, `family_client_uuid`) seguem depois dele; se ele foi recusado, os dependentes esperam. |
| 9 | **Sincronizar:** botão "Sincronizar" e sincronização automática quando o app está aberto e há rede (ao abrir, ao voltar ao primeiro plano e a cada 15 minutos). Ordem: envio primeiro, depois download. Carga inicial pede Wi-Fi (o ACS pode autorizar dados móveis). Página com `reset: true` apaga o território baixado (nunca a fila) e recomeça. Remoções (`moved`) apagam o domicílio ou cadastro local. `who_to_visit` da última página substitui o anterior. O `next_cursor` fica guardado. |
| 10 | **Prazo sem sincronizar:** 30 dias sem download bem-sucedido → o app apaga o território baixado (nunca a fila) e pede sincronização para voltar a mostrá-lo. Aviso a partir do 25º dia. |
| 11 | **Aparelho revogado (decisão 6A):** o api passa a separar dois códigos no `401` (muda a Q7 do 24b, só nesse ponto): `device_revoked` (revogação manual, pelo dono ou pelo administrador) e `device_suspended` (revogação automática por `membership_ended` ou `user_disabled`, sem dizer qual). `device_revoked` → o app apaga tudo, inclusive a fila, e volta ao cadastro; a tela de revogação do dashboard avisa "o que não foi sincronizado neste aparelho será perdido", com a data do último uso. `device_suspended` → o app apaga o território baixado, guarda a fila cifrada e mostra "acesso suspenso: procure o enfermeiro da equipe"; um QR novo do mesmo usuário (decisão 25) envia a fila. `invalid_token` → como `device_suspended` (pode ser host ou configuração errados). |
| 12 | **Relógio:** o app guarda a diferença entre o relógio do tablet e o `server_time` da última sessão; diferença acima de 5 minutos mostra aviso e corrige o `occurred_at` pela diferença conhecida. `occurred_at` sempre com fuso. A data mínima da visita é o `min_date` da sessão (prazo do SIAPS, decisão 8 do 24b). |
| 13 | **Família offline (contrato v1.2.0):** item `family` no envio, com `client_uuid`, `household_id` ou `household_client_uuid`, `version` do domicílio (como o `SaveFamily` do 24a exige) e o corpo da família do 24a; o responsável vai por `responsible_patient_id` ou `responsible_patient_client_uuid` e precisa de cadastro ativo no domicílio (o item do cadastro dele vem antes na fila). Cadastro individual e visita aceitam `family_client_uuid` no lugar de `family_id`. Só do ACS, como o cadastro (`cbo_not_allowed` para os outros). `stale_version` de `family` traz o domicílio atual (com as famílias) em `current`, e a mescla da decisão 15 vale para os campos da família. |
| 14 | **Duplicado no tablet:** CPF e CNS não descem ao tablet (o download traz só `has_cpf` e `cns_masked`), e o 24c não muda isso: nenhum resumo de documento no aparelho. Antes de salvar paciente novo, o app procura nos moradores baixados por nome, nascimento e mãe parecidos (avisa e oferece abrir o existente) e, por CPF ou CNS, só entre os pacientes que o próprio tablet criou e ainda não sincronizou (bloqueia). A busca no tablet só cobre as microáreas do aparelho; o api segue a última palavra (`duplicate_suspected` com `candidates`). Com `candidates`, o app mostra os candidatos que tem localmente; o ACS escolhe um deles (o item vira cadastro do existente) ou confirma que é outra pessoa (`duplicates_reviewed`). Sem rede, não há busca por CPF no servidor. |
| 15 | **Mescla campo a campo (3B):** `stale_version` traz `current_version` e `current`. O app compara três corpos: a base (o que baixou), o local (o que o ACS mudou) e o atual. A unidade de campo é cada chave de primeiro nível do corpo e cada chave dentro das cinco seções de dicionário do 24a (`living_conditions`, `identification`, `sociodemographic`, `health_conditions`, `street_situation`); listas (membros da família, deficiências) contam como um campo. Mudado só no local → fica o local; mudado só no atual → fica o atual; mudado nos dois com valores diferentes → conflito. Sem conflito, o app reenvia sozinho com `current_version`; com conflito, a tela "Conflitos" mostra cada campo com os dois valores (o do posto e o do ACS) e ele escolhe. Depois da escolha, reenvia com o mesmo `client_uuid`. Novo `stale_version` repete o ciclo. |
| 16 | **Coordenadas:** domicílio e visita ganham `location: { latitude, longitude, accuracy_m, source: "gps" \| "map" }` opcional. Latitude e longitude com até 8 casas, sempre em par. **Visita:** o app captura o GPS ao salvar (`Accuracy.High`, tempo limite de 15 s); sem posição precisa (permissão só aproximada, sem sinal de GPS, precisão pior que 50 m), a visita grava sem coordenada, nunca com uma ruim. **Domicílio:** GPS na frente do imóvel ou ponto marcado no mapa; o ACS pode corrigir. Sem localização em segundo plano. |
| 17 | **Coordenada no api e no LEDI:** colunas cifradas (`CityEncryption::CITY_KEYED_TARGETS`) no domicílio e na visita, porque localizam a casa de uma pessoa. A ficha tipo 3 leva `latitude`/`longitude` do domicílio (IDL 17/18) e a tipo 8 leva as da visita (IDL 20/21), arredondadas a 8 casas; sem coordenada, sem os campos. A visita feita pelo dashboard (24b) segue sem coordenada. Mudar só o ponto do domicílio é uma atualização do 24a (nova versão, nova ficha de atualização). |
| 18 | **Mapa por cidade:** a plataforma guarda uma cópia própria do mapa base do Brasil (build do Protomaps, PMTiles) e recorta um arquivo por cidade pela caixa do município (IBGE) com margem de 2 km, até o zoom 15. Cada arquivo tem versão (data da base), tamanho e SHA-256. Geração por job, disparada no maintenance ("gerar de novo") e a cada trimestre. O arquivo não tem dado de pessoa: domicílios e quem visitar ficam numa camada à parte, montada no app a partir do banco cifrado. |
| 19 | **Entrega do mapa:** `GET /field/v1/session` ganha `map: null \| { version, size_bytes, sha256 }`; `GET /field/v1/map` entrega o arquivo da cidade (com o token do aparelho, aceitando continuar download interrompido por `Range`). O app baixa só pelo Wi-Fi (salvo autorização), confere o SHA-256, guarda no armazenamento do próprio app e troca o antigo só depois de conferido. Estilo, fontes e ícones do mapa vão dentro do app, em português. A atribuição "© OpenStreetMap" fica sempre visível no canto do mapa. |
| 20 | **Mapa no app:** a microárea com os domicílios (agrupados quando próximos), cor por situação (quem visitar, visitado hoje, sem ponto), "minha posição" só quando o ACS toca no botão, e "marcar o ponto do imóvel". Lista continua sendo o caminho principal; o mapa é outra vista dos mesmos dados. |
| 21 | **Mapa no dashboard:** a microárea com os domicílios que têm ponto, para o ACS, o enfermeiro e o médico da equipe, com "corrigir o ponto" para quem pode editar o domicílio. Lê o mesmo arquivo da cidade pela rota da sessão (`GET /territory/map`, com `Range`), com o `pmtiles` do MapLibre GL JS. Leitura dos domicílios com ponto é nominal: trilha. |
| 22 | **O que o app oferece por CBO:** ACS → tudo; os outros CBOs da Tabela 7 → visita e atividade coletiva, só nas microáreas atribuídas, sem cadastro (regra do 24a e Q11 do 24b). O app lê o escopo e os interruptores da sessão: `home_visits` desligado → o app só mostra a fila e "Sincronizar" (o envio volta `feature_disabled` e o item fica na fila); `collective_activities` desligado → some a atividade. |
| 23 | **Primeira task do campo é prova técnica:** num build de desenvolvimento no emulador, abrir um `.pmtiles` local por `pmtiles://file://` com fontes e ícones locais, e abrir o SQLCipher conferindo `cipher_version` e que o arquivo não abre sem a chave. Resultado publicado no contrato antes das demais tasks. |
| 24 | **Referência offline:** rota nova `GET /field/v1/reference` (token do aparelho, sem trilha, sem dado de cidadão): dicionários e regras de tela do cadastro do 24a (as cinco seções, `tipo_logradouro`, `uf` e os demais do `registration_dictionaries.yml`), bairros ativos do módulo 11, unidades de saúde da cidade (para o local da atividade), profissionais das equipes do aparelho com CBO (para a atividade) e os domínios do 24b. A `session` ganha `reference_version` (resumo do conteúdo); o app baixa de novo só quando muda. Schema `reference-response-v1.json` no `field-sync` v1.2.0. |
| 25 | **Substituir aparelho:** `POST /field_devices` aceita `replaces_device_id` (do próprio usuário, step-up): cria o novo e revoga o antigo na mesma transação, sem ocupar vaga a mais do limite de 3. É o caminho depois do bloqueio do PIN (decisão 5) e da troca de tablet; o app, ao ler o QR do mesmo usuário, troca o token e mantém banco e fila. |
| 26 | **Mapa no armazenamento de objetos:** a base do Brasil e os arquivos por cidade ficam no MinIO/S3 que o api já usa. O job copia a build do Protomaps para o MinIO quando o operador pede no maintenance (e lembra a cada trimestre), recorta as cidades a partir da cópia e grava cada arquivo no MinIO. O api entrega o arquivo lendo do MinIO por intervalo (`Range`), sem URL assinada para o app. O CLI `pmtiles` entra na imagem do api (worker). |
| 27 | **Visitas de hoje e retenção:** sem rota nova de folha. "Minhas visitas de hoje" mostra as visitas que o tablet registrou no dia da cidade; "retirar" vale até o fim do dia e, se o servidor já fechou a folha, volta recusado com a frase de dia fechado. Visitas e atividades aceitas saem do tablet no dia seguinte (ADR 0041: apagado depois de sincronizado e confirmado); as do dashboard não descem. |
| 28 | **Testes no `bin/test`:** o `bin/test` ganha o alvo `campo` (Jest no container, como os frontends; worktree em `.worktrees/`). O Maestro fica fora do `bin/test`: roda no emulador local contra o api de dev e, na CI do `campo`, contra um servidor falso montado dos exemplos do `contracts` (sem api nem banco na CI). |

## 3. Dados

**Tablet (SQLCipher):** `session` (usuário, profissional, equipes, microáreas, interruptores, `min_date`, diferença de relógio, cursor, último download); `micro_areas`; `households`; `registrations` (cada um com `version`, `body`, `base_version`, `base_body`, `local_body`); `who_to_visit`; `outbox` (`client_uuid`, tipo, corpo, estado `pending` \| `sending` \| `needs_attention` \| `conflict`, erro, tentativas, criado em); `local_visits` e `local_activities` (o que o ACS registrou, para a folha do dia local); `map` (versão, caminho, SHA-256).

**api — cidade:**
- `households` e `home_visits` ganham `location` cifrada (latitude, longitude, `accuracy_m`, `source`).
- `FieldAuthentication` responde `device_suspended` na revogação automática (6A).
- `field_sync_receipts` aceita o tipo `family`; `accepted` devolve `version` nos tipos de cadastro.
- Rota `GET /field/v1/reference` (decisão 24) e `replaces_device_id` no cadastro do aparelho (decisão 25).
- Fichas tipo 3 e tipo 8 com latitude e longitude.

**api — plataforma:** `field_maps` (cidade, versão, tamanho, SHA-256, chave no MinIO, gerado em, estado `building` \| `ready` \| `failed`); a cópia da base do Brasil no MinIO (decisão 26); o CLI `pmtiles` na imagem do api.

**contracts:** `field-sync` v1.2.0 (MINOR, compatível com o v1.1.0): `family` em `upload-request`; `family_client_uuid` e `responsible_patient_client_uuid`; `version` no `accepted` dos cadastros; `location` no domicílio e na visita; `map` e `reference_version` na sessão; `reference-response-v1.json`; `device_suspended` e `cpf_or_reason_required` em `domains-v1.json`.

## 4. Telas do app

- **Cadastro do aparelho:** ler QR, criar PIN, carga inicial (com progresso e aviso de Wi-Fi), baixar o mapa.
- **Início:** microáreas, quantos a visitar, pendentes de envio, "precisa de atenção", conflitos, última sincronização, aviso dos 30 dias, botão "Sincronizar".
- **Quem visitar:** por microárea, com o motivo de cada pessoa (24b), atalho para a visita.
- **Mapa:** item 20.
- **Domicílio:** endereço, ponto, famílias, moradores, histórico local de visitas; novo domicílio, nova família, novo morador, editar (formulários do 24a, com as mesmas regras de tela).
- **Visita:** a partir do domicílio, da família, do morador ou de imóvel sem cadastro; os campos do 24b; "minhas visitas de hoje" (folha local) com retirar enquanto o dia está aberto no servidor.
- **Atividade coletiva:** formulário do 24b com participantes buscados entre os moradores baixados ou registro rápido.
- **Envio:** a fila, com o estado de cada item e a frase do erro; corrigir e reenviar.
- **Conflitos:** campo a campo, item 15.
- **Sobre:** versão, cidade, usuário, atribuição do OpenStreetMap, sair (apaga tudo, com aviso se houver pendente).

Tablet em pé e deitado; letras e alvos de toque grandes (uso ao sol e em pé); tema claro e escuro; tokens visuais do ciclo de interface.

## 5. Segurança e LGPD

- **No tablet:** banco cifrado (item 4), PIN (item 5), sem captura de tela, sem backup, sem log de pessoa, sem terceiros (item 6). Pendente só se perde na revogação, no `invalid_token` ou em "sair" confirmado.
- **Pendente na revogação:** só a manual apaga a fila (6A).
- **Fora do sistema:** nada de nome, CPF, CNS, motivo, medida, observação ou coordenada em URL, log, evento ou SMS. O token só no cabeçalho.
- **Trilha:** o download já grava `clinical_record.viewed` por página (24b); o mapa do dashboard com os pontos também. O arquivo de mapa não é nominal: sem trilha.
- **Exclusão LGPD (ADR 0026):** como no 24b, a exclusão não apaga cadastro nem domicílio (quem tem cadastro é retido); a remoção `erased` segue reservada. O pendente no tablet de quem pediu exclusão chega pelo envio e segue a regra de retenção do 24a.
- **Coordenada:** cifrada no banco da cidade; no tablet, dentro do banco cifrado; nunca no arquivo de mapa.

## 6. Testes

**campo:**
- Jest com React Native Testing Library: fila (ordem, dependentes, recusado fica, `duplicate` sai, reenvio com o mesmo `client_uuid`); mescla de três corpos (só local, só atual, os dois, listas, seções de dicionário); `reset`, remoções, `who_to_visit`; prazo de 30 dias; revogação apaga tudo; PIN e bloqueio; relógio; duplicado local; regras de tela lidas de `domains-v1.json`.
- Tudo pelo `bin/test campo` (decisão 28).
- Contrato: requisições e respostas dos testes validadas contra os schemas do `field-sync` v1.2.0 e os exemplos do `contracts`.
- Banco: teste que abre o arquivo sem a chave e falha, e que `cipher_version` responde.
- Maestro sobre APK release no emulador (API 34), contra o api de dev: cadastro por QR, carga, visita offline (modo avião), atividade, cadastro com família nova, sincronização, conflito com mescla e com escolha, revogação.
- CI no GitHub Actions com emulador (repositório público).

**api:** item `family` (idempotência, dependentes, `cbo_not_allowed`); `location` (par, faixa, casas, cifra, sem log); fichas 3 e 8 com e sem coordenada, conferidas campo a campo; `field_maps` (geração com um recorte pequeno de teste, versão, SHA-256, `Range`, sem token → 401); `map` na sessão; respostas contra a cópia dos schemas v1.2.0.

**dashboard:** componente do mapa da microárea (com o MapLibre simulado), correção do ponto, trilha.

**maintenance:** situação do mapa e "gerar de novo".

**Suíte completa do api no fim** (migração de cidade, `Ledi`, cifra, `field_sync`).

## 7. Entrega

- **Uma entrega só (1A).**
- **Ordem:** prova técnica do campo (item 23) → contracts v1.2.0 → api → campo → dashboard → maintenance → catálogo.
- **Repositório novo:** `rotasaude/campo` público, como os outros 7, com CI, os 31 labels, os templates do `.github`, branch `main` com proteção; card no board #1 e módulo `mod-24`.
- **Runbook:** `operacao/rollout-app-de-campo.md`: conta de desenvolvedor Google verificada; app privado no Google Play gerenciado de cada cidade; tablets por Android Enterprise (aparelho totalmente gerenciado, permissão de localização precisa); gerar o mapa da cidade antes de cadastrar aparelhos; ligar `home_visits` antes; cadastro dos aparelhos pelo dashboard; guardar a chave de assinatura fora do repositório.

## 8. F-IDs

| ID | Funcionalidade | Superfície |
|---|---|---|
| F-24.18 | App de campo: cadastro do aparelho por QR, PIN, banco cifrado e proteção do tablet | campo |
| F-24.19 | Download do território e sincronização (fila, lotes, reset, remoções, 30 dias, revogação) | campo |
| F-24.20 | Mapa offline por município (geração, entrega, mapa no app e no dashboard) | api, campo, dashboard, maintenance |
| F-24.21 | Cadastro domiciliar, família e individual offline com duplicado no tablet | contracts, api, campo |
| F-24.22 | Conflito com mescla campo a campo | campo |
| F-24.23 | Visita offline, "quem visitar" e atividade coletiva no app | campo |
| F-24.24 | Coordenada do imóvel e da visita nas fichas LEDI 3 e 8 | contracts, api, campo |
| F-24.25 | Build assinado, distribuição pelo Google Play gerenciado e runbook | campo, docs |

## 9. Riscos e pendências

- **`pmtiles://file://` no React Native:** não confirmado (prova técnica, item 23; decisão 5A).
- **Tamanho do mapa por município:** estimado em dezenas de MB; medir na prova com Curitiba e Maringá.
- **Verificação de desenvolvedor Android:** isenção para instalação por gestão de aparelhos não confirmada em documentação oficial; o caminho do Google Play gerenciado evita a dúvida.
- **Permissão de localização precisa pela gestão de aparelhos:** concessão automática em aparelho totalmente gerenciado não confirmada; o ACS pode conceder na primeira vez.
- **Maestro com Android 35/36:** suporte não confirmado; os testes usam API 34.
- **Coordenada no SIAPS:** não confirmado se o SIAPS usa a coordenada; a ficha a aceita como opcional.
- **Gates de go-live:** prova no PEC real das fichas 3 e 8 com coordenada; conta de desenvolvedor verificada; app privado publicado no Google Play gerenciado de uma cidade piloto; um dia de campo real sem sinal.
- **Próximo ciclo:** iOS/iPad; tablet compartilhado; sincronização em segundo plano; fechar o dia pelo app; rota até o domicílio.

