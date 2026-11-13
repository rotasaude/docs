# Módulo 24c — Contratos entre apps

**Fonte única** do que os repositórios do 24c combinam entre si. As linhas do §13 prevalecem sobre o texto das seções anteriores. Os planos de cada repositório (`-contracts-repo.md`, `-api.md`, `-campo.md`, `-dashboard.md`, `-maintenance.md`) seguem este documento; divergência vira pergunta (Q-n, §13) e a decisão volta aqui.

- **Spec:** `docs/superpowers/specs/2026-11-13-module-24c-field-app-design.md` (decisões do usuário 1A, 2A, 3B, 4A, 5A, 6A; do coordenador 1–28).
- **ADR:** 0053. **Base:** contrato do 24b (`2026-11-12-module-24b-home-visit-contracts.md`, §9 e §16 até a Q75) e `field-sync-v1.1.0` (contracts 673bb14). O api parte da main com o 24b (7a7343bc).
- **Nada do 24b muda**, salvo o que este documento diz que muda (§3.1 muda a Q7 só no código do 401).

## 1. Nomes

| Coisa | Nome |
|---|---|
| Repositório do app | `rotasaude/campo` (público), checkout em `apps/campo`; worktrees em `rota-saude/.worktrees/campo-<wt>` |
| Pacote Android | `br.rotasaude.campo` |
| Contrato | `field-sync` v1.2.0 (tag `field-sync-v1.2.0`; MINOR, compatível com v1.1.0); `session.contract` continua `"field-sync-v1"` |
| Interruptores | os do 24b (`home_visits`, `collective_activities`, `territory_registry`); nenhum novo |
| Tabela de plataforma | `field_maps` |
| Tabelas da cidade que mudam | `household_versions` (ou onde o 24a guarda o retrato do domicílio) e `home_visits` ganham `location` cifrada |
| Código novo de 401 | `device_suspended` (motivos `replaced`, `membership_ended`, `user_disabled`; Q17) |
| Códigos novos de item | `invalid_location`, `family_not_found` (ver §10) |
| Rotas novas | `GET /field/v1/reference`, `GET /field/v1/map`, `GET /territory/map`, `GET /territory/map/households` |
| Eventos novos | `field_map.built`, `field_map.failed` (plataforma; R18) |
| Job | `FieldMaps::BuildJob` (plataforma, por cidade) e `FieldMaps::RefreshBaseJob` (copia a base) |
| Alvo de teste | `bin/test campo [-w <wt>] [args do jest]` |

## 2. `field-sync` v1.2.0 no `rotasaude/contracts`

Arquivos novos ou alterados em `field-sync/` (JSON Schema 2020-12, fechado, mesmas regras de `$id`/`title` da v1):

| Arquivo | Muda |
|---|---|
| `session-response-v1.json` | + `map` e `reference_version` (§3.2) |
| `upload-request-v1.json` | + item `family` (responsável aninhado, Q1); + `family_client_uuid`; + `location` em `household` e `home_visit` (§3.3) |
| `upload-response-v1.json` | `accepted`/`duplicate` ganham `version` opcional (obrigatório nos tipos de cadastro, §3.3) |
| `changes-response-v1.json` | `<household>` ganha `location` (§3.4) |
| `reference-response-v1.json` | **novo** (§3.5) |
| `domains-v1.json` | `errors.http` + `device_suspended`; `errors.item` + `cpf_or_reason_required` (Q71 do 24b), `invalid_location`, `family_not_found`; `location` (limites do §4) |
| `examples/` + `manifest.json` | um válido por novidade: item `family` (novo, com `household_client_uuid` e `responsible_patient_client_uuid`), `accepted` com `version`, `stale_version` de `family`, `household` com `location`, `home_visit` com `location`, sessão com `map` e sem (`null`), referência; inválidos: `location` só com latitude, latitude fora da faixa, 9 casas |
| `README.md`, `CHANGELOG.md` | as regras do §3–§5 em texto curto |

- O api guarda a cópia em `config/field_sync/` presa por SHA-256, como na v1; a sessão do contracts devolve os SHA-256 novos ao api antes da tag.
- O app gera os tipos TypeScript dos schemas (`json-schema-to-typescript`) presos à tag `field-sync-v1.2.0` (cópia em `campo/contracts/field-sync/` com SHA-256 num teste).

## 3. api — sincronização

### 3.1 Revogação (6A)

- `authenticate_device!` (`FieldAuthentication`): aparelho revogado com `revoked_reason` em `membership_ended` ou `user_disabled` → `401 device_suspended`; revogado com `user` ou `admin` → `401 device_revoked`. A revogação nova feita na borda (vínculo perdido agora) responde `device_suspended`. O motivo da automática segue escondido (Q7 vale para ele).
- `invalid_token` não muda (o app trata como `device_suspended`, decisão 11).
- Dashboard (§8): `GET /field_devices` e `.../admin` já dão `status` e `revoked_reason`; a tela de revogar mostra o aviso de perda com `last_used_at`.

### 3.2 Sessão

`GET /field/v1/session` ganha:

```json
{ "map": null | { "version": "2026-11-01", "size_bytes": 41234567, "sha256": "<64 hex>" },
  "reference_version": "<64 hex>" }
```

- `map` é o `field_maps` da cidade com estado `ready`; `null` sem mapa.
- `reference_version` = SHA-256 do JSON canônico da referência (§3.5), calculado na hora (cache de 5 min por cidade e escopo).

### 3.3 Envio

**Item `family`** (só ACS; outros CBOs → `rejected cbo_not_allowed`; exige `territory_registry` utilizável):

```json
{ "client_uuid", "type": "family",
  "body": { "household_id" | "household_client_uuid", "family_id"?, "version",
            "responsible": { "patient_id" | "patient_client_uuid" },
            "income_band", "members_count", "living_since", "moved_out"? } }
```

- Sem `family_id` = família nova (`TerritoryRegistry::SaveFamily` com `family: nil`); com `family_id` = atualização. `version` é a do **domicílio** esperada.
- O cadastro ativo do responsável no domicílio, se houver, entra na família (L28 do 24a; Q25); quando o responsável é novo, o app manda o cadastro dele antes na fila.
- `household_client_uuid` e `patient_client_uuid` resolvem pelo livro (`FieldSyncReceipt`), como na Q27; citação não aceita → `rejected not_found`. `family_id` desconhecido ou de outro domicílio → `rejected family_not_found`.
- `stale_version` de `family` → `current` = `<household>` atual (com `families`), `current_version` = versão do domicílio.
- O livro (`field_sync_receipts`) aceita `family`, com `result_type: "Family"`.
- **Troca de responsável** (`replace_responsible`) não entra no app.

**`family_client_uuid`:** aceito no lugar de `family_id` em `individual_registration` e em `home_visit` (visita `family_responsible`).

**`version` no resultado:** `accepted` e `duplicate` de `household`, `family` e `individual_registration` levam `version` (inteiro): a versão do domicílio (para `household` e `family`) ou do cadastro (para `individual_registration`) depois da gravação. No `duplicate`, a versão gravada naquele recebimento (guardada no livro: coluna `result_version`). Os outros tipos não levam `version`.

**`location`** em `household` (corpo do 24a) e em `home_visit`:

```json
"location": null | { "latitude": -25.42847123, "longitude": -49.27325411, "accuracy_m": 8.5, "source": "gps" | "map" }
```

- Opcional; ausente = mantém (atualização do domicílio) ou sem coordenada; `null` = apaga (só domicílio).
- Regras do §4. Erro → `rejected invalid_location` com `path`.
- Na visita, `source` só `gps` (o app nunca marca ponto de visita no mapa) → `map` em visita = `invalid_location`.
- Mudar só `location` do domicílio é atualização do 24a: versão nova e ficha de atualização.

### 3.4 Download

- `<household>` em `changes` (e nas rotas do 24a que já o devolvem) ganha `location` (mesma forma; `null` sem ponto). Nada mais muda no `changes`.

### 3.5 Referência

`GET /field/v1/reference` (token do aparelho, sem trilha, sem query; mesmo gate do §9.2 do 24b):

```json
{ "reference_version": "<64 hex>",
  "registration_dictionaries": { <as chaves do registration_dictionaries.yml do 24a, como o dashboard gera territoryDictionaries> },
  "string_domains": { "uf": [...], "tipo_logradouro": [...] },
  "neighborhoods": [ { "id", "name" } ],
  "health_units": [ { "id", "cnes", "name" } ],
  "team_professionals": [ { "team_id", "professional_id", "name", "cbo" } ],
  "home_visit_domains": <o mesmo bloco `domains` do contexto da folha do 24b>,
  "collective_activity_domains": <o mesmo bloco `domains` das opções da atividade do 24b, por equipe (Q57)> }
```

- Bairros ativos do módulo 11; unidades ativas da cidade; profissionais ativos das equipes do aparelho (com CBO). Nome de profissional é dado de equipe, não de cidadão: sem trilha.
- O api monta com as mesmas funções que já servem o dashboard (sem cópia de regra).
- Schema `reference-response-v1.json`; as seções de dicionário ficam `type: object` (fonte é o YAML), como a v1.

### 3.6 Substituir aparelho

- `POST /field_devices` aceita `replaces_device_id` (uuid, opcional). Exige step-up (já exigido). O aparelho tem de ser do próprio usuário e ativo; senão `422 invalid_request` com `path: "replaces_device_id"`. Na mesma transação: revoga o antigo com `reason: "user"` e cria o novo; o limite de 3 conta sem o antigo.
- Resposta igual à do cadastro (`{ device, token, qr_payload }`).

## 4. Coordenada e LEDI

- **Faixas:** latitude −90..90, longitude −180..180, até 8 casas (mais casas → `invalid_location`, sem arredondar); `accuracy_m` 0..10000 com até 1 casa; sempre o par.
- **Cidade:** a coordenada fica cifrada (`CityEncryption::CITY_KEYED_TARGETS`, como um JSON único `location`) no retrato do domicílio e na visita. Nunca em log, evento ou URL.
- **Ficha tipo 3** (`CadastroDomiciliarThrift`): `latitude` (17) e `longitude` (18) do domicílio; **ficha tipo 8** (`FichaVisitaDomiciliarChildThrift`): `latitude` (20) e `longitude` (21) da visita. Sem `location`, sem os campos. Conferir contra a IDL vendorizada (`vendor/ledi/8.7.0/idl/ras`) na Task 1 do api e publicar aqui (Q-n).
- Visita do dashboard: sem `location` (o formulário do 24b não muda).

## 5. Mapa

### 5.1 Plataforma

- **`field_maps`:** `city_id` (único), `version` (data da base, `YYYY-MM-DD`), `size_bytes`, `sha256`, `storage_key` (MinIO), `state` (`building` | `ready` | `failed`), `built_at`, `error_code` (texto curto de lista: `base_missing`, `extract_failed`, `boundary_missing`), timestamps. O arquivo anterior só sai do MinIO depois do novo `ready`.
- **Base:** `FieldMaps::RefreshBaseJob` copia a build do Protomaps (URL em `config/field_maps.yml`, sem segredo) para o MinIO (`field-maps/base/<data>.pmtiles`) e grava a data em `field_map_bases` (ou numa linha de configuração da plataforma; escolha do plano). Disparado pelo maintenance; o maintenance mostra a data da base e "base com mais de 90 dias".
- **Recorte:** `FieldMaps::BuildJob(city_id)` roda `pmtiles extract <base> <saida> --bbox=<caixa> --maxzoom=15`, com a caixa do município (IBGE da cidade; fonte da caixa decidida na Task de mapa do api) mais 2 km; grava em `field-maps/cities/<city_id>/<version>.pmtiles`; calcula SHA-256; `ready`. O CLI `pmtiles` (versão fixa) entra na imagem do api.
- **Teste:** um recorte de exemplo pequeno (fixture `.pmtiles` de poucos KB no repositório do api) e o CLI simulado no spec; um spec de integração opcional marcado, fora da suíte.

### 5.2 Entrega

- **`GET /field/v1/map`** (token do aparelho): o arquivo `ready` da cidade, `Content-Type: application/vnd.pmtiles`, `ETag` = SHA-256, aceita `Range` (206) e `If-None-Match` (304). Sem mapa → `404 map_not_found`. Lê do MinIO por intervalo; sem URL assinada.
- **`GET /territory/map`** (sessão do dashboard; quem lê domicílios do 24a): o mesmo arquivo, mesmas regras, para o `pmtiles` do MapLibre GL JS.
- O arquivo de mapa é público em conteúdo (OSM) e não grava trilha; continua atrás de autenticação.

### 5.3 Pontos no dashboard

- **`GET /territory/map/households?micro_area_id=`** (sessão; escopo do 24a: quem lê a microárea): `{ items: [ { "id", "location": {…}, "address": { "street", "number" }, "families_count", "version" } ] }` só com domicílios com ponto. Nominal (endereço): grava `clinical_record.viewed` com `access: "territory_map"`, `household_ids`. Sem paginação (teto: microárea inteira).
- Corrigir o ponto = `PATCH /territory/households/:id` do 24a com `location` e `version` (§3.3).

### 5.4 maintenance

- GraphQL (padrão do maintenance): `fieldMaps` → por cidade `{ cityId, state, version, sizeBytes, builtAt, errorCode }` e `fieldMapBase { version, ageDays }`; mutações `buildFieldMap(cityId)` e `refreshFieldMapBase`, com `Platform.audit` (`field_map.build_requested`, `field_map.base_refresh_requested`; R18).

## 6. App (`rotasaude/campo`)

### 6.1 Estrutura

- Expo SDK 58, React Native 0.88, TypeScript estrito, nova arquitetura, Expo Router. Dependências nativas: `expo-sqlite` (`useSQLCipher: true`), `expo-secure-store`, `@maplibre/maplibre-react-native` 11.x, `expo-location`, `expo-camera`, `expo-crypto`, `expo-file-system`, `expo-network`, `expo-screen-capture` (ou `FLAG_SECURE` por plugin). Sem Expo Go; `expo prebuild` + Gradle.
- Pastas: `src/contract/` (tipos gerados e validação), `src/db/` (abertura cifrada, migrações locais, repositórios), `src/sync/` (cliente HTTP, fila, download, mescla), `src/map/`, `src/features/<tela>/`, `src/security/` (PIN, bloqueio, apagar).
- `app.json`: `android.allowBackup: false`, permissões só `CAMERA`, `ACCESS_FINE_LOCATION`, `ACCESS_COARSE_LOCATION`, `INTERNET`, `ACCESS_NETWORK_STATE`.

### 6.2 Banco local

- Abrir: chave de 32 bytes do SecureStore (`campo.db_key`, criada no cadastro); `PRAGMA key = "x'<hex>'"`; `PRAGMA cipher_version` precisa voltar uma linha, senão erro fatal "banco sem cifra" e nada é lido nem gravado.
- Tabelas: `session`, `micro_areas`, `households`, `registrations`, `families` (dentro do corpo do domicílio; tabela só de índice), `who_to_visit`, `reference`, `outbox`, `local_visits`, `local_activities`, `map_file`, `meta` (versão do esquema local).
- `households`/`registrations`: `id`, `version`, `body` (JSON atual), `base_version`, `base_body` (último baixado ou aceito), `local_body` (mudança local ainda não aceita, ou nulo), `client_uuid` (se nasceu no tablet), índices de busca (nome normalizado sem acento, nascimento, mãe, rua, número).
- `outbox`: `seq` (ordem), `client_uuid`, `type`, `body`, `depends_on` (lista de `client_uuid`), `state` (`pending` | `sending` | `needs_attention` | `conflict` | `blocked`), `error`, `path`, `attempts`, `created_at`, `updated_at`.

### 6.3 Sincronização

1. `GET session` (atualiza escopo, interruptores, `min_date`, diferença de relógio; `reference_version` diferente → `GET reference`; `map.sha256` diferente → baixar mapa em segundo plano da tela, §6.6).
2. **Envio:** lotes de até 50 itens `pending` em ordem de `seq`, pulando os que dependem de item não aceito (`blocked`). Por resultado: `accepted`/`duplicate` → remove da fila, grava `id` no mapa `client_uuid → id`, atualiza `base_*` com `version`; `rejected` → `needs_attention` (dependentes → `blocked`); `stale_version` → mescla (§6.4); `rejected too_many_requests` → `pending`, para a rodada (sem repetir).
3. **Download:** `GET changes` com o cursor até `has_more: false`; aplica itens (substitui `body` e `base_*` quando não há `local_body`; com `local_body`, só `base_*` muda e o item da fila segue), `removals`, `reset`, `who_to_visit`.
4. Erros HTTP: `429` → para e tenta na próxima rodada; `5xx`/rede → para, sem perder nada; `401 device_revoked` → apaga tudo (§6.5); `401 device_suspended`/`invalid_token` → apaga território, guarda fila, tela "acesso suspenso"; `403 feature_disabled` → mostra "visita desligada pela secretaria", fila fica.
- Quando: ao abrir, ao voltar ao primeiro plano, a cada 15 min com o app aberto e rede, e no botão. Uma rodada por vez (trava).
- Carga inicial e mapa só por Wi-Fi, salvo "usar dados móveis" (preferência local).

### 6.4 Mescla (3B)

- Entrada: `base` (`base_body`), `local` (`local_body`), `current` (do `stale_version`).
- Campos: cada chave de primeiro nível; dentro de `living_conditions`, `identification`, `sociodemographic`, `health_conditions`, `street_situation`, cada chave; listas e objetos fora dessas seções (ex.: `address`, `location`, membros) contam como um campo; igualdade por JSON canônico.
- Para cada campo: `local == base` → `current`; `current == base` → `local`; `local == current` → esse; senão → conflito.
- Sem conflito: novo corpo, `version = current_version`, item volta a `pending` com o mesmo `client_uuid`, `base_* = current`.
- Com conflito: item `conflict`; tela "Conflitos" mostra cada campo (rótulo da referência, "no posto" × "no tablet"); depois de escolher todos → `pending`.
- Função pura `merge(base, local, current): { body, conflicts: Path[] }` em `src/sync/merge.ts`, com testes por caso.

### 6.5 Segurança local

- **PIN:** 6 dígitos, resumo `SHA-256(sal + PIN)` com sal aleatório no SecureStore; pedido ao abrir e após 5 min em segundo plano; 5 erros → trava ("leia um QR novo no dashboard: Aparelhos de campo → Substituir").
- **QR novo:** mesmo `user.id` → troca o token, mantém tudo; outro usuário → recusado se houver fila; sem fila → apaga tudo e cadastra.
- **Apagar tudo:** fecha o banco, apaga o arquivo do banco, o mapa, a chave e o token do SecureStore.
- **Apagar território:** `micro_areas`, `households`, `registrations`, `who_to_visit`, `local_visits`, `local_activities`, cursor; mantém `outbox`, `reference`, mapa.
- 30 dias sem download bem-sucedido → apagar território; aviso desde o 25º.
- `FLAG_SECURE`; sem log com dado de pessoa (só `client_uuid`, tipo, código); sem SDK de terceiros.

### 6.6 Mapa no app

- Baixa `GET /field/v1/map` com `Range` para continuar; arquivo temporário; confere SHA-256; troca atomicamente; guarda em `FileSystem.documentDirectory/map/<version>.pmtiles`.
- Estilo local (Protomaps `light` e `dark`, em português) com glyphs e sprites dentro do app; fonte `pmtiles://file://<caminho>`.
- Camada GeoJSON dos domicílios do banco (agrupada), cor por situação; atribuição "© OpenStreetMap" fixa.
- Sem mapa baixado: a tela mostra "mapa ainda não baixado" e a lista segue.

### 6.7 Localização

- `expo-location` sob demanda; `getCurrentPositionAsync({ accuracy: High })` com tempo limite de 15 s; aceita só `accuracy ≤ 50` e permissão precisa; senão sem `location` (visita) ou oferece marcar no mapa (domicílio). Arredonda a 8 casas antes de gravar.

### 6.8 Telas

As do spec §4. Rótulos e frases em português; frases de erro por código, a partir de `domains-v1.json` + tabela local de frases (todos os códigos de `errors.http` e `errors.item` precisam de frase; teste confere).

## 7. Testes do app

- `bin/test campo`: Jest 30 + `jest-expo` + React Native Testing Library, num container Node 22 (como os frontends), sem emulador. SQLite com adaptador de teste (`better-sqlite3` ou mock do `expo-sqlite`), MapLibre simulado.
- Contrato: fixtures dos `examples/` da tag; toda requisição e resposta dos testes de sincronização valida contra os schemas (Ajv 2020).
- Maestro (fora do `bin/test`): APK release no emulador API 34; local contra o api de dev (`10.0.2.2`, host da cidade por cabeçalho ou `curitiba.localhost` mapeado); na CI do `campo`, contra um servidor falso (`scripts/fake-field-server`) que responde a partir dos exemplos do contrato.
- Teste de cifra no aparelho (Maestro ou teste instrumentado): o arquivo do banco não abre sem a chave e `cipher_version` responde.

## 8. dashboard

- **Mapa da microárea:** rota nova no grupo do território (24a), `maplibre-gl` + `pmtiles` (versões fixas), estilo local igual ao do app; lê `GET /territory/map` e `GET /territory/map/households`; "corrigir o ponto" (arrasta ou toca) para quem pode editar o domicílio, grava com `PATCH` e `version`; `409/422 stale_version` relê. Sem mapa da cidade → aviso.
- **Aparelhos de campo:** "Substituir" (step-up) → `POST /field_devices` com `replaces_device_id`; revogar mostra "o que não foi sincronizado neste aparelho será perdido" com a data de `last_used_at`; status `inactive` mostra "suspenso" (o vínculo voltou? cadastre de novo com Substituir).
- Testes de componente (`bin/test dashboard`), MapLibre simulado.

## 9. maintenance

- Aba ou seção "Mapas de campo" (no padrão das abas por cidade): estado, versão, tamanho, data; "Gerar de novo" por cidade; "Atualizar base" com a idade da base. Testes de componente (`bin/test maintenance`).

## 10. Códigos

| Código | Onde | Quando |
|---|---|---|
| `device_suspended` | 401 `/field/v1/*` | aparelho revogado por `membership_ended` ou `user_disabled` |
| `map_not_found` | 404 `/field/v1/map`, `/territory/map` | cidade sem mapa `ready` |
| `invalid_location` | item (`path`), 422 no `PATCH` do 24a | par incompleto, fora da faixa, mais de 8 casas, `source` inválido |
| `family_not_found` | item | `family_id` desconhecido ou de outro domicílio |
| `cpf_or_reason_required` | item e rotas da equipe | Q71 do 24b |

Os demais seguem o §13 do 24b.

## 11. Repositório, CI e entrega

- **Criar `rotasaude/campo`** (com autorização do usuário): público, `main` protegida como os outros, os 31 labels, os 5 templates do `.github`, README, LICENSE igual aos outros. CI: lint + typecheck + `jest`; job Maestro com `reactivecircus/android-emulator-runner@v2` (API 34, x86_64, KVM) contra o servidor falso; imagens base pelo `mirror.gcr.io`.
- **`bin/test`:** alvo `campo` (container Node 22 do compose de dev ou `docker run` com o worktree montado; escolha do plano do campo, sem tocar no `postgres-test`).
- **Ordem:** prova técnica do campo (Task 1 do plano do campo; resultado publicado aqui) → contracts v1.2.0 (sem tag) → api → contracts (SHA-256 e tag) → campo → dashboard → maintenance → catálogo.
- **Prova:** no emulador (QR do dashboard de dev, carga, modo avião, visita, família nova, sincronização, conflito com e sem escolha, revogação manual e automática, mapa); no navegador (mapa da microárea, corrigir ponto, substituir aparelho, maintenance).
- **Runbook:** `operacao/rollout-app-de-campo.md` (spec §7).
- **Gates de go-live:** spec §9.

## 12. Fora do 24c

iOS, tablet compartilhado, sincronização em segundo plano, fechar o dia pelo app, troca de responsável pelo app, rota até o domicílio, PSE, IVCF-20, agendamento.

## 13. Decisões (Q-n)

| Q | Decisão |
|---|---|
| Q1 (C-2, G-2, QC-1) | Responsável do item `family`: forma aninhada `responsible: { patient_id \| patient_client_uuid }`, igual ao `SaveFamily`. O §2 e a decisão 13 do spec ficam corrigidos (não existe `responsible_patient_client_uuid` plano). |
| Q2 (C-1) | O api traduz erro de schema sob `/body/location` em `rejected invalid_location`, com o mesmo `path`; os demais erros de schema seguem `invalid_request` (Q31 do 24b). |
| Q3 (C-3, G-19, QC-9) | `location`: obrigatórios `latitude`, `longitude` e `source`; `accuracy_m` obrigatório com `source: "gps"` e opcional com `"map"` (o app manda a resolução do toque, metros por pixel × 24). Na visita, `location` ausente = sem coordenada e `null` = `invalid_location`; no domicílio, `null` apaga. |
| Q4 (C-4) | `stale_version` de `family` nova: `id` = o domicílio. |
| Q5 (C-5, G-4) | `version` obrigatório no `accepted` dos cadastros e opcional no `duplicate` (recibos anteriores à coluna `result_version` não a têm). O app, ao receber `duplicate` sem `version`, marca o domicílio ou cadastro para recarga no próximo download (sem mescla). |
| Q6 (C-6, G-6, QC-11) | `team_professionals` = a função do dashboard (`professional_memberships`) filtrada a `in_team`, com `has_cns`; `health_units` com `cnes` anulável. O app oferece a atividade a todo aparelho com `collective_activities` e o api decide (`cbo_not_allowed`). |
| Q7 (C-7, G-7) | `collective_activity_domains` é um bloco único (como `collective_activities/json.rb`); o recorte por equipe fica em `team_professionals.team_id`. |
| Q8 (C-8, G-15, QC-8) | `errors.http` da v1.2.0 ganha `device_suspended`, `map_not_found`, `range_not_satisfiable` e os códigos de borda da cidade que o `/field/v1` pode devolver (`unknown_city`, `city_suspended`, `city_schema_behind`); todos com frase no app. |
| Q9 (C-9) | `cpf_or_reason_required` já está em `errors.item` desde a v1.0.0; nada a acrescentar (a Q71 do 24b fica atendida). |
| Q10 (C-10, D-9) | Superfícies: F-24.18 campo e dashboard (substituir); F-24.19 contracts, api, campo e dashboard (aviso de perda, `device_suspended`); F-24.20 contracts, api, campo, dashboard e maintenance. Spec §8, módulo e CSV corrigidos. |
| Q11 (D-1) | Estilo do mapa num domínio novo do contracts, `map-style/` (estilo `light` e `dark` em português, lista de glyphs e sprites com versão do `@protomaps/basemaps`), tag `map-style-v1.0.0` publicada depois da prova técnica; campo e dashboard copiam por lock com SHA-256, como os tokens. |
| Q12 (D-2) | Versões: a Task 1 do campo fixa no contrato as de `@maplibre/maplibre-react-native`, `@protomaps/basemaps` e do CLI `pmtiles`; o dashboard fixa `maplibre-gl` e `pmtiles` (JS) na sua Task 0, a versão estável com mais de duas semanas, e registra aqui. |
| Q13 (D-3) | Sem `PATCH` parcial: corrigir o ponto no dashboard relê o domicílio, confere a versão e manda o corpo inteiro do 24a com `location`. |
| Q14 (D-4) | `GET /territory/map/households` também devolve `map: null \| { version, size_bytes, sha256 }`; o dashboard não sonda o arquivo. |
| Q15 (D-5) | O mapa do dashboard só mostra e corrige pontos existentes; o primeiro ponto é marcado no app. |
| Q16 (D-6, G-17) | `replaces_device_id` só para aparelho ativo. Aparelho revogado automaticamente: o dashboard oferece "Cadastrar novo". `inactive` (vínculo ausente, calculado na leitura) aparece como "suspenso enquanto sem vínculo"; revogado por `membership_ended`/`user_disabled` como "revogado automaticamente: cadastre de novo". O app troca o token ao ler QR do mesmo host e usuário, venha de "Substituir" ou de "Cadastrar novo". |
| Q17 (D-7) | Motivo novo `replaced` em `REVOKED_REASONS` (revogação por pessoa, com `by`), gravado pelo substituir; a borda responde `device_suspended` para `replaced`, `membership_ended` e `user_disabled`, e `device_revoked` só para `user` e `admin`. O tablet que falar com o token velho antes de ler o QR novo guarda a fila. |
| Q18 (D-8) | Dashboard: mapa só no estilo `light`; o app tem `light` e `dark`. |
| Q19 (M-1, G-14) | maintenance: cidade por `citySlug`; auditoria no padrão `MaintenanceAudit`: `maintenance.field_map.build_requested` e `maintenance.field_map.base_refresh_requested`. Os jobs gravam `field_map.built` e `field_map.failed` (`Platform.audit`, R18). |
| Q20 (M-2) | "Mapas de campo" é tela própria no trilho do maintenance (molde de Medicamentos), não aba da cidade. |
| Q21 (M-3) | GraphQL: `state` e `errorCode` `String`; `sizeBytes` escalar `BigInt` (mapeado no `codegen.ts`); `fieldMapBase` anulável. |
| Q22 (M-4) | As duas mutações não pedem TOTP (nada é apagado; precedente `importAnvisaLists`). |
| Q23 (M-5, G-11) | Sem job recorrente. "Atualizar base" pega a build mais recente do Protomaps pelo índice e extrai só a caixa do Brasil (em dev, a do Paraná) para o MinIO; o maintenance avisa quando a base passa de 90 dias. A decisão 18 do spec ("a cada trimestre") fica atendida pelo aviso. |
| Q24 (M-6) | Sem base, o maintenance não oferece "Gerar de novo"; o api responde `base_missing` se for chamado. |
| Q25 (G-1) | Sem regra nova: o cadastro ativo do responsável no domicílio, se houver, entra na família (L28 do 24a). O §3.3 fica corrigido; o app segue mandando o cadastro do responsável antes quando ele é novo. |
| Q26 (G-3, QC-5) | O livro resolve `patient_client_uuid` também contra recibos de `individual_registration` com `new_patient` (o `client_uuid` do cadastro vira o paciente criado por ele). Assim o app cria a pessoa nova pelo cadastro individual, com nome social, e a cita como responsável. |
| Q27 (G-5, QC-6) | `registration_dictionaries` vem de `TerritoryRegistry::Dictionary::DATA`, na forma do gerador do dashboard (`ledi-dictionaries.mjs`): listas `[{ code, label }]` na ordem do YAML. |
| Q28 (QC-7) | Regras de tela do cadastro, da visita e da atividade: o app porta os arquivos de regra do dashboard, com o commit de origem anotado e os mesmos testes. Divergência futura entre os dois vira card. |
| Q29 (G-8) | O cache guarda só o hex de `reference_version`, nunca o conteúdo. |
| Q30 (G-9) | `field_maps` guarda o arquivo publicado (`version`, `size_bytes`, `sha256`, `storage_key`); `state`/`error_code`/`built_at` descrevem a última geração. A sessão serve o publicado mesmo durante `building` ou depois de `failed`; `map` é `null` só sem arquivo publicado. |
| Q31 (G-10) | Chave do objeto: `field-maps/cities/<city_id>/<version>-<sha12>.pmtiles`; SHA igual ao publicado não sobe nada. |
| Q32 (G-12) | O recorte lê de um cache local da base no worker, conferido por SHA-256; o espaço em disco entra no runbook. |
| Q33 (G-13) | Caixa do município: CSV vendorizado com a caixa dos 5.570 municípios, tirado da API de malhas do IBGE, com SHA-256 e origem no `vendor/`. |
| Q34 (G-16) | Entrega: um intervalo só por requisição; `416` com `Content-Range: bytes */<size>` e `{ "error": "range_not_satisfiable" }`; `Cache-Control: private, no-cache`; `If-Range` aceito com o `ETag`. |
| Q35 (G-18) | A versão nova do cadastro do responsável gravada pela família não volta no `accepted`; o próximo envio desse cadastro recebe `stale_version` e a mescla resolve. Documentado no README do `field-sync`. |
| Q36 (G-20) | O ponto vai na ficha 3 também com recusa do domicílio. |
| Q37 (G-21) | `location` do domicílio aparece em todas as rotas do `<household>` do 24a; a `location` da visita não aparece em nenhuma rota da equipe (só vai à ficha). |
| Q38 (G-22) | `/territory/map/households` só com domicílios ativos. |
| Q39 (G-23) | Predicado novo `TerritoryRegistry::Access.reads_households?(user)` (lê ao menos uma microárea); `/territory/map` exige ele, `/territory/map/households` exige a leitura da microárea pedida. |
| Q40 (G-24) | O contrato cita os números da IDL (17/18 na ficha 3, 20/21 na ficha 8); o dicionário numera diferente. |
| Q41 (QC-2) | Cada item de cadastro na fila guarda a base da edição (`edit_base_version`, `edit_base_body`) e envia sempre essa versão; o download atualiza só `body`/`base_*` da linha, nunca a base do item. Assim, mudança do posto no meio do caminho sempre volta como `stale_version` e passa pela mescla. O §6.2 ganha `edit_base_version`, `edit_base_body`, `target_id` e a tabela `client_ids` (`client_uuid → id`). |
| Q42 (QC-3) | A mescla compara as projeções na forma do envio (sem `families`, que têm item próprio, e sem `patient`). |
| Q43 (QC-4) | Mescla de `family`: compara campo a campo a família de mesmo `id` no `current`; família nova só troca a versão do domicílio; família que sumiu do `current` vira `needs_attention` com `family_not_found`. |
| Q44 (QC-10) | QR: build de produção aceita só `https` e host com sufixo de uma lista fixada no build (`CAMPO_ALLOWED_HOST_SUFFIXES`); QR fora da lista é recusado sem chamar a rede. Os sufixos de staging e produção entram no runbook e são gate do build de release. |
| Q45 (QC-11) | `household_changed` e `internal_error` são transitórios: o item volta a `pending` por até 3 rodadas, depois `needs_attention`. |
| Q46 (QC-12) | "Mesmo usuário" no QR novo = mesmo host e mesmo `user.id`. |
| Q47 (QC-13) | Saída do cadastro (mudança, óbito) e edição de identidade ficam fora do app no 24c ("Faça pelo dashboard"); entram no §12 e viram card no fechamento. |
| Q48 (QC-14) | Maestro roda num APK release de variante `e2e` (campo "Colar código", diagnóstico da cifra, tráfego sem TLS só para `10.0.2.2`); um script de verificação prova que o APK de produção não tem nada disso. |
| Q49 (nota do contracts) | O Ajv do app valida `multipleOf: 1e-8` com `multipleOfPrecision: 8`, para bater com o `json_schemer` do api. |
| Q50 (cruzamento) | `fieldMaps` (GraphQL) lista só as cidades com linha em `field_maps`; o maintenance junta com a lista de cidades. Sem `cityId` nem `stale` no SDL. |
| Q51 (cruzamento, Q26) | Forma mais simples da Q26: `patient_client_uuid` resolve qualquer recibo aceito de `individual_registration` para o `patient_id` daquele cadastro (além do recibo de `patient`). **Sem** a coluna `result_patient_id` e **sem** reescrever os dependentes no app: em "É esta pessoa", o próprio cadastro passa a ser do paciente existente e quem o cita resolve para ele. Prevalece sobre o texto dos planos do api (Tasks 3 e 9) e do campo (Tasks 21–22). |
| Q52 (cruzamento, Q11) | Interface do `map-style/`: estilos com os marcadores `__PMTILES__`, `__GLYPHS__` e `__SPRITE__` (`sprite: "__SPRITE__/<tema>"`), `dist/manifest.json` com versão e SHA-256 de cada arquivo; dashboard copia para `src/lib/vendor/map-style/` + `public/map/` com `src/lib/vendor/map-style.lock`; campo para `assets/map/` com `assets/map/map-style.lock`. |
| Q53 (cruzamento, Q5) | Aceito: a marca de recarga da Q5 só some quando a linha vier no download; até lá, a próxima edição cai em `stale_version` e mescla. Afeta só recibos de dev/test. |
| Q54 (cruzamento) | Só o coordenador escreve no contrato: a Task 1 do campo (prova e versões) e a Task 0 do dashboard (versões) relatam ao coordenador, que transcreve no §13. |
