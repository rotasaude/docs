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
| Q55 (Task 1, prova) | Fontes e sprites no app por `asset://map/fonts/{fontstack}/{range}.pbf` e `asset://map/sprites/<tema>`, servidos de `android/app/src/main/assets/map` por plugin de config local (variante A da prova; sem cópia na abertura). O `.pmtiles` fica em `file://<documentDirectory>/map/<cidade>.pmtiles` e entra como `pmtiles://file://…`. O dashboard segue a Q52 (`public/map/`). |
| Q56 (Task 1, prova; Task 14) | O SHA-256 do mapa baixado sai do nativo do `expo-file-system` 58 (`new File(uri).digest("SHA-256")`, 12 ms em 14,7 MB, sem carregar o arquivo em memória). Substitui as opções (a) `Crypto.digest` e (b) `@noble/hashes` da Task 14; a porta `fs` do `AppServices` ganha `sha256Hex(uri): Promise<string>`. |
| Q57 (Task 1, prova; Task 24 e api) | Fixture de mapa: `sample-z8-15.pmtiles` (932.293 bytes, z8–15), o único que fica ≤ 1 MB. Serve à Task 24 do campo e ao fixture do api (Range e SHA-256). O `sample-z0-15` não entra em repositório. Mapas reais continuam z0–15. |
| Q58 (Task 1, prova; Task 15) | No mapa do app, o logo e o botão (i) do MapLibre ficam desligados, e um texto fixo próprio "© OpenStreetMap" fica sempre visível (Restrições globais). Não muda contrato. |
| Q59 (Task 1, prova; Tasks 24 e 26) | Prova offline e Maestro sempre com JS embutido (variante release `e2e`): o build de depuração não alcança o Metro com a rede do emulador desligada. |
| Q60 (contracts Task 0, contracts#2) | Na MINOR v1.2.0, numa Task 3b própria, antes do SHA-256 da Task 4: (1) texto e dados — README com UUID em minúsculas e `note`/`reason` já aparados; `errors.http` com o código do 415 com o mesmo nome que o api usa hoje; os dois trechos mal lidos do README corrigidos; exemplos de `overdue` com data e dos pares exclusivos; (2) esquema só de resposta — `duplicate_suspected` exige `candidates` e `results` ganha `minItems: 1`, desde que o api de hoje já cumpra (conferir nos specs do api; se não cumprir, fica para depois). Ficam para depois, com o motivo no CHANGELOG e no contracts#2: candidato discriminado por `type`, `id` do `already_registered` amarrado ao `type`, fusão dos blocos `care_line`, metaesquema no `check.sh`, expects com a chave e forma de `proof`. A cópia do esquema no api acompanha a v1.2.0 (plano do api). |
| Q61 (contracts Task 6; Q55, §14) | **Fontes completas:** o `map-style/` copia do `protomaps/basemaps-assets` (commit `028c18f`) as **quatro** pilhas que o estilo do `@protomaps/basemaps` 5.7.2 cita (Noto Sans Regular, Medium, Italic e Noto Sans Devanagari Regular v1), com as **256 faixas** de cada uma (~11,5 MB), sem gerar binário. Motivo (medido no emulador): faixa ausente faz o MapLibre não terminar de carregar o bloco e apaga ruas, POIs e prédios. O manifesto lista as quatro pilhas; o teste do `map-style` cobra que toda pilha citada pelas camadas (inclusive dentro de expressões de `text-font`) tenha as 256 faixas. Substitui as "três faixas" da Q55 e do §14. A prova integrada (campo Task 26) ganha um caso fixo: rótulo com "Ω" aparece e o mapa termina de carregar. |
| Q62 (contracts Task 6; Q61) | As faixas da pilha Devanagari que no upstream são atalhos para a Noto Sans Regular viram cópias materializadas (atalho não sobrevive ao pacote do app nem ao lock por SHA-256): o `dist/` fica com ~18,4 MB. O estilo do upstream não é alterado (sem trocar a pilha Devanagari). Manifesto: `fonts` é a lista das quatro pilhas e `glyph_ranges: 256`; o teste cobra arquivo real (não atalho, não vazio) em cada faixa e que as pilhas pedidas pelas camadas sejam exatamente as do manifesto. Rótulo `name:pt` cobrado em toda camada que lê algum `name`. |
| Q63 (api Task 9; Q2) | O `path` do `invalid_location` é o mesmo do erro de esquema (`schema_path`): aponta o campo mais fundo que falhou (`location.latitude`, `location.accuracy_m`, `location.longitude`, `location.altitude` para chave a mais; `location` só quando o objeto inteiro é nulo ou inválido). Igual ao do `FieldLocation` na rota da equipe, para app e dashboard verem o mesmo path. Prevalece sobre o exemplo `location` seco da Task 9 do plano do api. |
| Q64 (api Task 18; Q22) | GraphQL do maintenance: `fieldMaps`, `fieldMapBase`, `buildFieldMap` e `refreshFieldMapBase` entram em `HumanOnly::RESTRICTED`, só com sessão humana, sem token (precedente: `medicationCatalog`/`importAnvisaLists`). `buildFieldMap` herda de `CityMutation` com `on_platform`: escopo da credencial, slug conferido antes de auditar, cidade existente e ativa, sem abrir o banco da cidade. Segue em `step_up_exempt`. Recusas no path `citySlug`: "cidade inexistente", "cidade não está ativa (<status>)", "slug de cidade inválido" (este sem linha de auditoria). O SDL não muda. Suíte completa do api só uma vez, no fim, depois de prender os esquemas na tag `field-sync-v1.2.0`. |
| Q65 (dashboard Task 6; Q16, Q17) | Aparelhos no dashboard: (1) o rótulo mantém o `perspective` do 24b; na lista do administrador, o aparelho revogado automaticamente segue "revogado: fim do vínculo na equipe", sem "cadastre de novo" nem "por você"; (2) "Substituir" e "Cadastrar novo" só aparecem para quem pode cadastrar (interruptor ligado e `can_register_device`), porque os dois são um POST de cadastro; "Revogar" aparece sempre (Q75 do 24b); (3) "Cadastrar novo" traz o nome do aparelho antigo pré-preenchido. |
| Q66 (api Task 18) | Suíte completa final do api: 12257/12258 no HEAD 573b478b. A única falha era o spec de sigilo da Task 17, que procurava os eventos só por `audited(` depois da Q64; a correção (065deffa) muda só o regex desse spec e passou no escopo fechado (431/0). Como a correção não pode afetar outro spec, a suíte não é repetida antes do merge. |
| Q67 (campo Tasks 4–9) | Desvios aceitos: (1) `mapRequest` assíncrono (token por `getToken` assíncrono); (2) `setPin` e `checkPin` recebem a porta de hash; (3) `enqueue` recebe o gerador de UUID; item que volta a `pending` só é tentado na rodada seguinte (as 3 rodadas da Q45 não viram 3 lotes da mesma chamada); no `too_many_requests` cada resultado é aplicado como veio e a rodada para; `refreshBlocked` também solta o bloqueado quando a causa some; (4) tipos gerados com `unreachableDefinitions`; os corpos de envio são `<Tipo>ItemBody`, porque `HouseholdBody` e `RegistrationBody` são a forma do download; (5) `registerDevice` recebe portas (apagar e criar banco), e a ligação com `wipeAll`/`openEncryptedDb` fica na casca (Task 16). Vale o esquema sobre o exemplo da Task 9: no cadastro individual, `version` e `registration_id` andam juntos; na família, `version` é sempre obrigatória. |
| Q68 (campo Tasks 23 e 25) | O build `e2e` assina com `CAMPO_E2E_*` e nunca com `CAMPO_UPLOAD_*`, também no build local: um APK e2e com "Colar código" e cleartext, assinado com a chave de envio, instalaria por cima do app de produção. Acréscimos aceitos: `allowScreenCapture` só no `e2e`, para a evidência da Task 26; cleartext para `localhost` só no build de desenvolvimento, por causa do Metro. O `scripts/check-apk.mjs` cobra, no APK de release, FLAG_SECURE ligado, nenhum cleartext e a assinatura com a chave de envio. |
| Q69 (campo Tasks 9 e 10; Q41) | (1) A mescla do `stale_version` compara só as chaves que o item envia: chave ausente no corpo de envio significa "manter o do servidor", então a base e o atual são recortados às chaves do item, e o corpo mesclado tem exatamente essas chaves. Comparar a projeção inteira daria conflito falso quando o posto muda o que o tablet nem mandou. Prevalece sobre o texto da Task 10. (2) Uma segunda edição de registro nascido no tablet, escrita antes de o servidor devolver o id, sai com o `household_id`/`registration_id` da linha local quando o `accepted` anterior já o trouxe; sem id na linha, vai sem `version`, porque o esquema amarra os dois. |
| Q70 (campo Task 11; Q51) | A linha nascida no tablet mantém a chave local (`client_uuid`) quando o registro do servidor chega, e o id do servidor vai para `client_ids`. `registrations.household_key` guarda a chave local do domicílio, e quem aponta para a linha (cadastros e itens da fila) segue válido sem reescrita, como na Q51. Numa página do download, os itens são aplicados na ordem microárea → domicílio → cadastro. |
| Q71 (campo Task 12) | A referência é recusada inteira, e a anterior fica, quando um campo de seção traz uma regra que o app não conhece (chave fora de `key`, `ledi`, `type`, `domain`, `max`, `min`, `max_items`, `required`, `fields`), quando cita um domínio que não veio ou quando chega uma seção de nome desconhecido. Seção ausente é lista vazia. O erro diz onde quebrou, nunca o valor. Consequências: (1) o app mostra "atualize o app" quando a recusa for por regra ou seção nova, sem bloquear o trabalho com a referência anterior; (2) no api, regra ou seção nova na referência é MINOR do `field-sync`, e só vai para produção junto com uma versão do campo que a conheça. Os blocos `home_visit_domains` e `collective_activity_domains` são conferidos contra a forma do api na Task 21. |
| Q72 (campo Task 13) | (1) O limite de 30 dias é conferido no início de toda rodada, mesmo sem rede: vencido, o território sai e a fila fica. (2) `purgeAccepted` apaga também as visitas `withdrawn` de dias anteriores; pendente nunca sai. (3) `home_visits` desligado na sessão: nada é enviado, o download segue e a rodada termina `feature_disabled` ("Visita desligada pela secretaria"); o 403 `feature_disabled` no envio para a rodada antes do download, com a fila intacta. (4) Falha HTTP sem tratamento (`invalid`, `not_found`, `range`) termina `stopped server`, sem perda; a marca de suspenso sai quando uma sessão volta a responder (QR novo do mesmo usuário, 6A). Validade do cadastro pelo dia da cidade em `registered_at`; 29/02 + 24 meses = 28/02, vencido no próprio dia (regra do 24a). |
| Q73 (campo Tasks 14 e 23; Q55, Q31) | (1) O arquivo do mapa no tablet é `map/<version>-<sha12>.pmtiles`, e não `<cidade>.pmtiles` (substitui o nome da Q55). A troca é atômica: renomeia, grava `map_file` e só então apaga o anterior, como na Q31 do api. (2) Em `map_file`, as colunas `version`, `sha256`, `size_bytes` e `path` aceitam NULL para guardar o parcial e o `resume_data` do primeiro download; "tem mapa" é `path` não nulo. (3) O app escreve só estas linhas de log: `sync.round.start` (gatilho), `sync.round.end` (resultado), `sync.item.result` (`client_uuid`, `type`, status ou código) e `map.download.failed` (status HTTP ou `mismatch`). Nunca path, candidatos nem corpo. O teste de vazamento confere essas linhas. |
| Q74 (campo Tasks 8 e 16) | (1) Os portões ficam dentro do layout raiz (`AppRoot`), e não como rotas: tela fatal sem cifra → pareamento (QR) → criar PIN → PIN → Acesso suspenso → Primeira carga, e só então o navegador das rotas principais é montado (`/`, `/who`, `/outbox`, `/about` e as das Tasks 15, 20 e 22). Nenhuma tela com dado de pessoa existe antes do PIN, e nada fala com o servidor antes dele: o agendador e as rodadas começam depois do PIN e param quando o app bloqueia. Deep link para tela interna com o app bloqueado não funciona, e o app não tem deep link. (2) `src/app/platform.ts` é a única interface com o aparelho, e `devicePlatform.tsx` é o único arquivo que importa módulo Expo; a prova dele é da sessão Android, no emulador. (3) Chave do banco ausente com token presente volta ao pareamento, porque os dados se foram. (4) "Quem visitar" mostra os motivos como vêm do servidor, sem recalcular validade na lista. (5) `testTimeout` de 20 s no Jest do campo. As frases das telas ficam no ledger do 24c; o contrato só fixa a da Q71 e a da Q79 do 24b. |
| Q75 (campo Tasks 3, 14, 15 e 18; Q73, Q74) | (1) Toda transação do banco local é `BEGIN IMMEDIATE`/`COMMIT` na conexão única e cifrada, por uma fila (`src/db/serial.ts`). Nunca `withExclusiveTransactionAsync`: ela abre uma segunda conexão sem a chave da cifra. O banco de teste usa a mesma fila; uma leitura de tela pode esperar a transação da sincronização. (2) A retomada do download do mapa parte do tamanho do `.part` em disco. A coluna `map_file.resume_data` sai do esquema local v1, e o resto da Q73(2) fica. (3) `isOnline` é rede conectada, sem exigir a sonda de internet do Android; a requisição decide, e a falha conta como rede. (4) O enquadramento inicial do mapa usa a caixa do cabeçalho do próprio PMTiles. (5) O MapLibre fica só em `src/app/mapCanvas.tsx`, atrás de `MapCanvasProps`. (6) `src/location/capture.ts` nasce na Task 15 (antes da 18). A posição do agente em "Minha posição" fica só na tela: nunca é gravada, enviada nem logada. (7) "Visitado hoje" lê `local_visits.summary` como JSON com `household` (chave local ou id), e a Task 20 grava nesse formato. Cores por token (a visitar = focus, visitado hoje = ok, cadastrado = accent), sempre com legenda escrita. (8) No bloqueio por inatividade, as telas principais ficam montadas sob a tela de PIN, que é opaca e cobre tudo; os filhos ficam fora da acessibilidade, o agendador fica parado e nada fala com o servidor. O que foi digitado continua só na memória e se perde se o app for encerrado. Não há rascunho no banco. No arranque a frio nada muda: nenhuma tela existe antes do PIN. A prova no aparelho confere que a prévia de apps recentes não mostra a tela de baixo. |
| Q76 (campo Task 17; Q67, Q75) | (1) Os testes das regras portadas usam uma fixture com os dicionários completos, gerada de `territoryDictionaries.generated.ts` do dashboard e lida pelo mesmo parser da referência do app. O exemplo `reference` do contrato segue sendo um recorte, e o app não depende dele. (2) As regras nascem de fábricas: `createRegistryRules(dict)`, `createVisitRules({ offsetMin, newPatientProblem })` e `createActivityRules({ quickPatientProblems })`. O que só lê `domains` fica solto, com a assinatura do dashboard. (3) Os formatos de recusa seguem a origem: no cadastro é `{ path, message }`, e na visita e na atividade é `{ path, text }`. (4) O relógio da cidade usa o deslocamento em minutos que o servidor deu. (5) Não vieram as frases de erro do api, os portões, os rótulos da folha e as frases de tela da atividade. Quando uma tela das Tasks 20–21 precisar de uma, ela é copiada do dashboard, e não reescrita. (6) `collective_activity_domains.rules` tem a forma do tipo do dashboard. (7) Bloqueio da Q75(8): só cobre as telas principais. Fora delas o bloqueio desmonta. Quem esconde a prévia de apps recentes é o `FLAG_SECURE` da Task 23. A origem é o dashboard em ceb53af, e nenhuma regra mudou: cadastro 251/251, visita 28/28 e atividade 25/25 casos portados; nas duas partes portadas por script, 8 mutações e 8 pegas. |
| Q77 (campo Task 18; Q51, Q70, Q75, decisão 22, Q47) | (1) A coluna local `local_body` (em `households` e `registrations`) sai do esquema v1. A tela mostra a linha do servidor com os itens da fila aplicados por cima, marcados "Aguardando envio". (2) No aceite de um item `household`, a linha recebe id, versão e os campos aceitos, até o próximo download trazer o registro. A base da mescla segue sendo o corpo enviado. (3) Um domicílio novo é uma linha nascida no tablet (chave = `client_uuid`) mais um item `household` sem `household_id`. Se esse domicílio for editado com a criação ainda em envio, a edição vira um item novo que espera a versão da criação (dep `version`), e nunca uma segunda criação. (4) As seções das fichas são desenhadas pelo dicionário da referência. Rótulos e ordem são cópia de `territoryLabels.ts` do dashboard (ceb53af), e um teste cobra rótulo para todo campo. Lista com mais de 20 opções vira busca. (5) "Visitas locais" no detalhe do domicílio fica para a Task 20. (6) Decisão 22: sem `territory_registry` ou com CBO diferente de 515105, o formulário não abre e o detalhe fica só para consulta. "Registrar visita" continua. (7) Q47 no detalhe: mudança, óbito e correção de identidade são feitas pelo dashboard. (8) A permissão de localização é pedida de novo quando só a aproximada foi dada. (9) Reforço da Q75(8): uma View opaca cobre as telas principais já no evento `background`. No `active`, ela vira a tela de PIN depois de 5 min ou sai, com as mesmas quatro condições. (10) Achado: os exemplos do field-sync levam `address.street_type: "RUA"`, que é o rótulo, mas o api e o dicionário usam o código ("081"). O esquema aceita qualquer texto, então só os exemplos estão errados. O app grava o código. A correção dos exemplos entra no contracts#2. |
| Q78 (campo Task 19; Q69, Q70, Q77) | (1) Na mescla, "Reside desde" é comparado por mês: o download traz o dia, e o envio leva `AAAA-MM`. (2) A edição de família é juntada à que já espera na fila por `family_id`, e nunca pelo alvo, que para toda família é o domicílio. (3) No aceite de `individual_registration` e `family`, a linha local recebe o que foi aceito, como na Q77(2). A família aceita troca o id do tablet pelo do servidor no domicílio e nos cadastros dela. (4) O `is_responsible` de uma pessoa nova é falso quando ela entra numa família que já existe. Sem família, a tela pergunta, e nunca pré-escolhe, nem com um só morador. A edição não muda o campo. (5) Editar uma pessoa ou família criada no tablet, antes do envio, entra na própria criação. Com a criação já em envio ou retida, o cadastro ganha um item novo que espera a versão. A família recusa com "Esta família ainda está sendo enviada. Sincronize e edite depois." (6) Raça/cor usa a codificação do paciente (01 Branca … 05 Indígena), como o dashboard, e não a do domínio LEDI. Etnia só aparece com 05. (7) Duplicado no tablet: o aviso sai com o mesmo nascimento e nome a até 2 letras (e mãe igual, quando as duas existem), e "É outra pessoa: salvar" manda `duplicates_reviewed: true`. O bloqueio sai com um CPF ou CNS já digitado em outro cadastro ainda na fila. Nem o aviso nem o bloqueio mostram o número. (8) Respostas do servidor (a tela é da Task 22): `duplicate_suspected` com "É esta pessoa" troca `new_patient` por `patient_id` e reescreve `patient_client_uuid` → `patient_id` em tudo que cita a pessoa; no registro rápido, o item some. Com "É outra pessoa", o mesmo `client_uuid` é reenviado com `duplicates_reviewed`. `already_registered` descarta o item e o que o cita e abre o cadastro existente, se o tablet o tem. Rótulos e frases de `FamilyForm.tsx` e `RegistrationForm.tsx` são copiados do dashboard (ceb53af). |
| Q79 (campo Task 20; Q3, Q75, Q78) | (1) A tela da visita não oferece o registro rápido de pessoa. A pessoa nova entra por "Novo morador"; em imóvel sem cadastro, ou para quem não é ACS, a visita é sem pessoa. A gravação continua aceitando um item `patient` citado pela visita, para a Task 21. (2) Retirada: a visita nunca enviada sai da fila e da folha, junto com a pessoa digitada só para ela. A que está em envio naquele instante recusa com "A visita está sendo enviada agora. Tente de novo em instantes." A visita aceita vira um `home_visit_withdrawal` com o motivo (10–500), e a folha só muda com o aceite. Pedir duas vezes não enfileira dois. (3) `local_visits.summary` é `{ household, subject, outcome }`, com `household` sendo a chave local ou null. O resumo diz quando e onde, e nada do que foi dito. (4) O momento oferecido é agora na cidade, corrigido pelo desvio do relógio. A pessoa pode mudar, e a regra portada recusa fora de [`min_date`, agora]. (5) Ao salvar, o app tenta a posição por até 15 s e grava com `location` (gps, `accuracy_m`) ou sem a chave; nunca null, nunca posição ruim. Os motivos aparecem todos, por grupo, e os indisponíveis com o porquê. (6) `day_closed` na retirada: vale "A folha deste dia já foi fechada. A visita não pode mais ser retirada.", próxima da frase do dashboard (Q76(5)). Ela substitui a redação do plano do campo. (7) Condição da Q78: o aviso de pessoa parecida diz só "Pode ser alguém que já está neste tablet. Confira antes de salvar." e oferece "Ver cadastro parecido" e "É outra pessoa: salvar", sem nome nem nascimento da outra pessoa. |
| Q80 (campo Task 21; Q6, Q26, Q76, Q79) | (1) Pessoa sem cadastro numa atividade vai dentro do participante (`participants[i].new_patient`, com `duplicates_reviewed` quando revista), e não num item `patient` à parte. Quem foi cadastrado no tablet há pouco é citado pelo `client_uuid` do cadastro, com dep (Q26). O morador baixado vai por `patient_id`. A gravação do item `patient` segue testada, mas nenhuma tela a usa na v1. (2) `duplicate_participant` é tratado na tela de itens que precisam de atenção (Task 22), com "É outra pessoa" só quando `confirmable` é true. Atividade registrada no tablet não tem lista própria: aparece em "Ver envios" enquanto pendente, e `local_activities` guarda só dia, tipo, turno e quantos participantes. (3) O turno e a data oferecidos vêm da hora da cidade. A atividade aparece a todo aparelho da cidade com `collective_activities` ligado, e o api decide quem registra (Q6). Desligada, o Início não tem o botão. (4) Achado: `domains-v1.json` descreve `rules` da atividade com outro vocabulário (`by_activity_type`, `practices_only_in_activity_type`…), mas o api serve em /reference a forma de `CollectiveActivities::Json.domains` (`list_max`, `by_type`, `health_topics_with_meeting_topic`, `practices_only_in_type`, `procedure_practices`, `smoking_practices`, `participant_measurements`). Vale a forma do api, que o app lê (Q76(6)). A correção do `domains-v1.json`, e um esquema que cobre essa forma em `reference`, entram no contracts#2, junto do `street_type` da Q77(10). |
| Q81 (campo Task 22 e tela de diagnóstico; Q68, Q78, Q80) | (1) "Corrigir" existe só para `household`, `family` e `individual_registration`: abre o formulário com o que está na fila, e salvar devolve o mesmo item, com o mesmo `client_uuid`. Para visita, atividade, retirada e pessoa do registro rápido não há formulário de volta na v1; a tela diz "Este registro não pode ser corrigido aqui. Descarte e registre de novo." e oferece Reenviar e Descartar. Isso substitui o plano do campo, e a correção desses tipos fica como pendência de ciclo. (2) No `duplicate_suspected` do servidor, os candidatos aparecem como o contrato os manda: nome de exibição, nascimento e com ou sem CPF/CNS, nunca o número, com "É esta pessoa" em cada um e "É outra pessoa". A condição da Q78 vale para o aviso e o bloqueio locais, e não para os candidatos do servidor, que são feitos para o agente decidir, como no dashboard. (3) No `already_registered`, o `registration_id` devolvido fica guardado com o item (`[{ registration_id }]`). "Abrir o cadastro existente" descarta o item e o que o cita e abre o cadastro. Fora do escopo do tablet, diz "O cadastro existente não está neste tablet. O registro foi descartado." (4) Conflito: um bloco por campo, com "No posto: …" e "No tablet: …" pelos rótulos do dicionário. O ponto do imóvel aparece em palavras (fonte e precisão), nunca a coordenada; id de outro registro aparece como "outro registro". "Enviar" só com um lado escolhido em cada campo. O campo apontado pela recusa aparece em palavras. (5) A tela de diagnóstico existe só no build e2e: a rota redireciona ao Início fora dele, e os módulos ficam atrás do literal `EXPO_PUBLIC_E2E`. As linhas são fixas, "cifra: <versão>" e "sem chave: recusado" (ou "ABRIU"). A sonda abre sem chave uma cópia do arquivo do banco. Nada do conteúdo do banco aparece. |
| Q82 (revisão final do campo e provas no navegador; Q41, Q71, Q72, Q75, Q77, Q81) | (1) Aceitas as correções da revisão: C1 (base da edição lida na abertura do formulário), C2, I1–I6, e as quatro do tablet (margens seguras, mapa, Voltar no "Marcar no mapa", teclado). O Expo fica com `root: "app"`. (2) C2 substitui a Q81(3). O descarte leva o que cita e o que espera a versão, limpa a folha, as atividades e as linhas nascidas no tablet na mesma transação, e avisa antes quantos dependentes saem. "Enviada" aparece só com o aceite. No `already_registered` com o cadastro no tablet, o que citava a pessoa digitada passa a citar o paciente dele, e o cadastro abre. Com o cadastro fora do tablet, nada é descartado sozinho, e a tela diz "O cadastro que esta pessoa já tem não está neste tablet. Descarte este registro, ou peça ao enfermeiro da equipe para conferir." (3) Corrigir antes do merge: (a) uma falha do GET reference que não seja de rede mantém a referência anterior (Q71), e a rodada segue para o envio; (b) uma falha do apagamento total nunca é engolida: o app fica na tela fatal, sem abrir dado, e tenta de novo no próximo arranque; (c) a retirada de visita roda numa transação só; (d) `session.contract` diferente de `field-sync-v1` deixa o app sem rodada e mostra o aviso de atualizar o app; (e) com `home_visits` desligado, somem do Início "Registrar visita" e "Minhas visitas de hoje". Domicílios e Mapa seguem o cadastro territorial, e a atividade segue `collective_activities`; isto substitui a frase do plano. (4) Ficam como estão: a precisão do GPS em número na tela (precisão não é posição) e a base da mescla depois do aceite (Q77(2)). (5) Pendências de ciclo: o conflito entre dois pontos do mapa mostra a mesma frase dos dois lados; a retenção de 30 dias e o bloqueio usam o relógio cru do tablet; `attempts` é compartilhado entre 429 e recusa passageira; o maintenance deixa gerar o mapa de cidade arquivada; o spec `:pmtiles_cli` não roda pelo `bin/test` (imagem sem o binário até o rebuild, e a variável não é repassada). (6) A prova integrada da Task 26 responde: `not_found` em `*_client_uuid` para dependente de item adiado, o feed com os registros do próprio aparelho na mesma rodada, e `already_registered` direto para item com `new_patient`. Ela também acompanha a falha isolada do "Salvar cadastro" no Maestro. (7) Provas no navegador: maintenance 4/4; dashboard 7/8, com o item 5 (cidade sem mapa) coberto pelo teste `map: null`. Na semente não há aparelho com vínculo encerrado, então a prova 8 encerrou e restaurou o vínculo por runner. O dashboard corrigiu a URL absoluta do sprite em 677a511. |
| Q83 (campo, Q82(3); Q71, Q72) | (1) A falha do GET reference com o servidor respondendo (`server`, `invalid`, `not_found`, `range`) mantém a referência anterior, e a rodada segue para o envio e o download. A referência é pedida de novo na rodada seguinte. Continuam encerrando a rodada a rede, o 429 e `revoked`/`suspended`/`feature_disabled`. (2) Contrato diferente de `field-sync-v1` tem frase própria, e não a da Q71: "Atualize o app: esta versão não conversa mais com o servidor da secretaria. O que você registrar fica guardado no tablet até a atualização." O desfecho é `stopped: app_outdated`, e não há marca no banco. O pareamento também confere o contrato: se for diferente, nada é gravado, e a tela diz "Atualize o app antes de cadastrar este tablet." (3) O apagamento total que falha fica devido no cofre (`campo.wipe_owed`, fora das `SECURE_KEYS`), e o próximo arranque o termina antes de ler o token. A frase da tela fatal é "Não foi possível apagar os dados deste tablet. Feche o app e abra de novo." (4) Com `home_visits` desligado, "Registrar visita" some de todo lugar: o "Quem visitar" e "Minhas visitas de hoje" no Início, e o botão no detalhe do domicílio, da família e do morador. Domicílios e Mapa seguem `territory_registry`, e a atividade segue `collective_activities`. |
| Q84 (campo Tasks 23–25, sessão Android; Q34, Q77(10), Q82(1)) | (1) `USE_BIOMETRIC` e `USE_FINGERPRINT` vão em `blockedPermissions`: vêm do androidx.biometric do expo-secure-store, e o app não usa biometria. (2) O 11º fluxo do Maestro, `map-resume`, usa `next_map: "cut:<bytes>"` no roteiro do servidor falso para provar no aparelho a retomada da Q34: um 206, e o SHA final certo. (3) O servidor falso converte o rótulo de `street_type` no código até a correção dos exemplos no contracts#2 (Q77(10)). Essa conversão sai quando os exemplos forem corrigidos. (4) A prova foi feita só em desenvolvimento, no emulador de tablet, com autorização do usuário. Passaram o `check-apk` no APK e2e e num APK de produção de ensaio (chave descartável, sufixo `.rotasaude.example`), a janela SECURE, a miniatura de recentes preta, a ausência de backup e de depuração, o banco cifrado e o cleartext só para 10.0.2.2 no e2e. Os 11 fluxos ficaram verdes no HEAD do app, e o bloqueio de PIN aos 5 min passou com o leitor de tela. A leitura de QR pela câmera fica para aparelho físico. (5) Ficam abertos: o job do Maestro na CI (que só se prova com push), a chave de envio real, os sufixos de produção, o AAB e o Play, o runbook `operacao/rollout-app-de-campo.md` e a Task 26 (prova integrada contra o api de dev). |

## 14. Prova técnica do campo (Task 1)

**Data:** 2026-11-14, sessão "MVP Module 24c App". **Resultado:** PASSOU em todos os critérios; a decisão 5A segue sem volta ao usuário.

| Critério | Resultado |
|---|---|
| Mapa offline com glyphs e sprites | OK: APK release com JS embutido, emulador em modo avião; `pmtiles://file:///data/user/0/<pacote>/files/map/sample.pmtiles`; ruas, nomes em português e ícones de POI nos temas light e dark (71 camadas), sem erro de glyph ou sprite no logcat. As variantes `asset://` e `file://` funcionaram; fica a `asset://` (Q55). |
| `cipher_version` | `4.17.0 community` (SQLite 3.53.3) |
| Sem chave falha | Sem `PRAGMA key` e com chave errada: "Error code 26: file is not a database"; com a chave, lê a linha. |
| Cabeçalho cifrado | `9e9d57454d123e8639e3e7d2e2c05496` (não é `SQLite format 3\0`); `strings` não acha o texto gravado. |

**Versões que funcionaram** (as três marcadas com Q12 ficam fixadas para todos os repos):

- Expo 58.0.25 · React Native 0.88.1 · React 19.3.0 · nova arquitetura
- `@maplibre/maplibre-react-native` **11.5.1** (MapLibre Native Android 13.6.1, variante opengl) — Q12
- `@protomaps/basemaps` **5.7.2** (`layers("protomaps", namedFlavor("light"|"dark"), { lang: "pt" })`) — Q12
- CLI `go-pmtiles` **1.31.2** (commit a3e4951) — Q12
- `expo-sqlite` 58.0.17 (`useSQLCipher: true`) · `expo-file-system` 58.0.13 · `expo-crypto` 58.0.8 · `expo-secure-store` 58.0.3 · `expo-asset` 58.0.21
- JDK 21.0.12 · Gradle 9.4.1 · Kotlin 2.2.10 · compileSdk 37 · targetSdk 36 · minSdk 24 · NDK 27.1.12297006 · Node 22.19.0
- `protomaps/basemaps-assets` commit `028c18f713baecad011301ff7a69acc39bcc2ae7`: fontes Noto Sans Regular, Medium e Italic (na prova, faixas 0-255, 256-511, 8192-8447; no `map-style`, as quatro pilhas completas, Q61) e sprites v4 light e dark (`.json`, `.png`, `@2x`); alimentam a Task 6 do contracts-repo (`map-style-v1.0.0`).
- Build Protomaps usada na prova: `20261113.pmtiles`.

**Tamanhos (z0–15, caixas do plano):**

| Recorte | Tamanho | Blocos | Observação |
|---|---|---|---|
| Curitiba (-49.40,-25.65,-49.18,-25.34) | 14.673.035 bytes (15 MB) | 958 | extração em ~11 s; SHA-256 `575d7431d813161c8b9dd34e8537ed1860c410acd1647359befa33f624970bf1` |
| Maringá (-52.10,-23.53,-51.85,-23.30) | 4,3 MB (dry-run) | 830 | |
| Amostra z0–15 | 1.426.872 bytes | | acima de 1 MB por causa dos zooms baixos |
| Amostra z8–15 | 932.293 bytes | | fixture (Q57) |

**SHA-256 de 14,7 MB no emulador arm64** (os três iguais ao do host): `File.digest` nativo 12 ms (Q56); `Crypto.digest` sobre `file.bytes()` 30 ms; `@noble/hashes` incremental em blocos de 1 MB 5,8 s.

**Ajustes de ambiente:** imagem de sistema `system-images;android-34;google_apis;arm64-v8a` reinstalada (a do SDK estava sem `system.img`), AVD `campo_api34` (pixel_tablet); o Gradle instalou Build-Tools 36 e 37, Platform 37 e o NDK acima. No Android, `Paths.bundle` é `asset:///` e `File.copySync` de asset para documento funciona.

**Guardado fora de repositório** (até as Tasks que consomem): `.worktrees/campo-proof-tools/keep` (fontes, sprites, as duas amostras, `SHA256SUMS`, evidências).

**Versões do dashboard (Task 0 do dashboard, Q12; 2026-11-14):** `maplibre-gl` **6.16.0** (publicada em 2026-10-25) e `pmtiles` **4.5.0** (2026-08-10) — as estáveis mais novas com mais de duas semanas, sem beta nem rc.
