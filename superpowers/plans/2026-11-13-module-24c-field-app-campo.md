# Módulo 24c — App de campo (`rotasaude/campo`) Implementation Plan

> **Q51 do contrato prevalece:** sem `field_sync_receipts.result_patient_id`; `patient_client_uuid` resolve qualquer recibo aceito de `individual_registration` para o `patient_id` do cadastro; o app não reescreve dependentes em "É esta pessoa". Onde este plano diz o contrário (api Tasks 3 e 9; campo Tasks 21–22), vale a Q51.

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Construir o app Android offline do ACS e dos demais agentes da Tabela 7 (cadastro do aparelho por QR, banco cifrado, sincronização `field-sync` v1.2.0 com mescla campo a campo, mapa PMTiles local, cadastros do 24a, visita e atividade coletiva do 24b), testado pelo `bin/test campo` e pelo Maestro, com build release assinado e runbook.

**Architecture:** Expo SDK 58 com prebuild (sem Expo Go, sem build na nuvem), Expo Router nas telas, SQLCipher como única persistência de dado de pessoa. A lógica fica em módulos puros (`src/sync`, `src/rules`, `src/security`) que recebem as portas nativas por injeção (`AppServices`), para o Jest rodar no container sem emulador. O app espelha o `api`: tipos gerados dos schemas da tag `field-sync-v1.2.0`, regras de tela portadas do dashboard, dicionários vindos de `GET /field/v1/reference`; quem decide é sempre o `api`.

**Tech Stack:** Expo SDK 58, React Native 0.88, TypeScript estrito, Expo Router, `expo-sqlite` (`useSQLCipher: true`), `expo-secure-store`, `@maplibre/maplibre-react-native` 11.x (versão exata fixada pela Task 1, Q12), `expo-location`, `expo-camera`, `expo-crypto`, `expo-file-system`, `expo-network`; estilo do mapa copiado do `map-style-v1.0.0` do `contracts` (Q11); Jest 30 + `jest-expo` + React Native Testing Library + `sql.js` (adaptador de teste) + Ajv 2020 (só em teste, `multipleOfPrecision: 8`, Q49) + `json-schema-to-typescript`; Maestro 2.x; Gradle local.

**Spec:** `docs/superpowers/specs/2026-11-13-module-24c-field-app-design.md` · **Contrato (fonte única):** `docs/superpowers/plans/2026-11-13-module-24c-field-app-contracts.md` (§1, §2, §3, §6, §7, §11 e §13 — Q1–Q49, que prevalece) · **Base:** contrato do 24b (`2026-11-12-module-24b-home-visit-contracts.md`, §9 e §16 até a Q75) · **Pesquisa:** `docs/pesquisa/2026-11-13-app-de-campo-24c.md`.

## Global Constraints

- Repositório `rotasaude/campo` (público), checkout `apps/campo`, worktree `rota-saude/.worktrees/campo-mod24c` (branch `feat/mod-24c-field-app`); nunca `node_modules` dentro de `apps/<outro app>`.
- Pacote Android `br.rotasaude.campo`; um app só para todas as cidades (a cidade vem do QR).
- Expo SDK 58, React Native 0.88, TypeScript `strict: true`, nova arquitetura; `@maplibre/maplibre-react-native` 11.x; versões fixas (`save-exact`), escolhidas por `npx expo install`.
- Sem Expo Go, sem EAS Build: `expo prebuild` + Gradle local. `android/` é gerado e fica no `.gitignore`.
- `app.config.ts`: `android.allowBackup: false`; permissões exatamente `CAMERA`, `ACCESS_FINE_LOCATION`, `ACCESS_COARSE_LOCATION`, `INTERNET`, `ACCESS_NETWORK_STATE` (mais a `DYNAMIC_RECEIVER_NOT_EXPORTED_PERMISSION` de assinatura que o AGP gera).
- Contrato: `field-sync` v1.2.0, tag `field-sync-v1.2.0`; `session.contract` = `"field-sync-v1"`; cópia em `contracts/field-sync/` presa por SHA-256.
- Envio em lotes de 1 a 50 itens; download com `limit` 200 (máximo 500).
- PIN de 6 dígitos, resumo `SHA-256(sal + PIN)`; bloqueio ao abrir e após 5 min em segundo plano; 5 erros seguidos travam.
- Chave do banco: 32 bytes aleatórios no SecureStore (`campo.db_key`), `PRAGMA key = "x'<hex>'"`, sem `requireAuthentication`; `PRAGMA cipher_version` sem linha = erro fatal "banco sem cifra".
- Território apagado após 30 dias sem download completo; aviso a partir do 25º dia. Nunca apagar a fila, salvo `device_revoked` ou "Sair" confirmado.
- Sincronização automática: ao abrir (depois do PIN), ao voltar ao primeiro plano, a cada 15 min com rede; uma rodada por vez. Carga inicial e mapa só por Wi-Fi, salvo a preferência "usar dados móveis".
- Relógio: diferença acima de 5 min avisa e corrige `occurred_at`; `occurred_at` sempre com fuso (o da cidade, tirado do `server_time`).
- Localização: `Accuracy.High`, tempo limite 15 s, só com permissão precisa e `accuracy ≤ 50` m; latitude e longitude arredondadas a 8 casas; `accuracy_m` 0..10000 com 1 casa, sempre presente com `gps` (obrigatório, Q3) e, no ponto marcado no mapa, a resolução do toque (metros por pixel × 24; opcional no contrato, o app sempre manda); visita só `source: "gps"`.
- Mapa: estilo, fontes e ícones dentro do app, copiados da tag `map-style-v1.0.0` do `contracts` por lock com SHA-256 (Protomaps `light`/`dark`, em português; Q11); fonte `pmtiles://file://…`; atribuição "© OpenStreetMap" sempre visível. Nenhuma URL `http(s)` de terceiro no app.
- Logs sem dado de pessoa (só `client_uuid`, tipo, código, `seq`); sem SDK de telemetria, de falhas ou de mapa de terceiros; `FLAG_SECURE`; o token só no cabeçalho `Authorization: Bearer`, nunca em URL, log ou tela.
- Testes: só `bin/test campo -w campo-mod24c [args do jest]` (Jest 30 + `jest-expo` + RNTL em container Node 22). Nunca `jest` direto no host. Maestro fica fora do `bin/test` (API 34).
- Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta); nunca `git add -A`/`git add .` — sempre a lista de arquivos; Conventional Commits em inglês, tipo por extenso; toda mensagem termina com a linha `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Forma de todo passo de commit:
  `G=/opt/homebrew/bin/git; W=/Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/campo-mod24c; $G -C $W add <arquivos>; $G -C $W commit -m "<tipo>: <descrição>" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`
- Textos da interface em português; frases de erro por código (todo código de `errors.http` e `errors.item` tem frase).
- QR (Q44): o build de produção aceita só `https` e host com sufixo da lista fixada no build (`CAMPO_ALLOWED_HOST_SUFFIXES`); release sem a lista não compila. "Mesmo usuário" = mesmo host e mesmo `user.id` (Q46).
- As lacunas QC-1 a QC-14 foram decididas no §13 do contrato; o plano aplica cada decisão onde ela cai, com o número da Q, e o resumo está em **"Decisões aplicadas (§13 do contrato)"**, no fim.

## Review Focus

1. **App morto no meio da rodada** (itens em `sending`, página de download pela metade): na volta nada some nem duplica — `sending` volta a `pending` na abertura (o `client_uuid` dá `duplicate`), a página é aplicada numa transação e o cursor só avança com ela. Testes na Task 9 (`recoverSending`) e na Task 11 (`applyPage` falha no meio → nada aplicado, cursor antigo).
2. **Mudança do posto sobrescrita sem aviso** quando o item esperou (recusado, conflito, bloqueado, 429) e o download trouxe versão nova: o envio usa a versão-base da edição, nunca a da linha atualizada, e a mescla roda com essa base (Q41). Teste na Task 9 (`sends the edit base version after a newer download`) e na Task 10.
3. **QR forjado, malformado ou de outro host/usuário**: host fora de `CAMPO_ALLOWED_HOST_SUFFIXES` é recusado sem chamar a rede (Q44); payload ruim dá frase, não exceção; outro usuário com fila é recusado; o token nunca aparece em log nem em tela. Testes na Task 8.
4. **Relógio ou fuso errados no tablet** (horas de diferença, fuso do aparelho diferente do da cidade): `occurred_at` sai no fuso da cidade, corrigido pela diferença conhecida, e a data mínima/hoje usam o dia da cidade. Testes na Task 13 (`cityNowIso`) e na Task 20.
5. **Download do mapa interrompido, arquivo trocado no servidor no meio, SHA-256 diferente ou disco cheio**: o parcial nunca é usado, o mapa anterior continua, o parcial de outro `sha256` é descartado; a continuação manda `If-Range` com o SHA da sessão e um `200` inteiro ou `416 range_not_satisfiable` recomeça do zero (Q34). Testes na Task 14.
6. **Pessoa nova citada pelo cadastro individual** (Q26): a família, a visita e a atividade citam o `client_uuid` do `individual_registration` com `new_patient`; "É esta pessoa" num `duplicate_suspected` troca o `new_patient` por `patient_id` e reescreve os dependentes para `patient_id`. Testes nas Tasks 19 e 22.

---

## Estrutura de arquivos (`rotasaude/campo`)

```
app.config.ts                      config Expo (pacote, permissões, plugins, flags de build)
plugins/withFieldHardening.js      FLAG_SECURE, backup/extração desligados, cleartext, permissões
plugins/withReleaseSigning.js      signingConfigs a partir de ~/.gradle/gradle.properties
plugins/withMapAssets.js           copia assets/map/{fonts,sprites} para android assets (se a prova pedir asset://)
app/                               rotas do Expo Router (finas: só montam a tela de src/features)
  _layout.tsx  lock.tsx  suspended.tsx  (setup)/scan.tsx  (setup)/pin.tsx  (setup)/initial-load.tsx
  (main)/index.tsx who.tsx map.tsx outbox.tsx about.tsx
  (main)/households/[key].tsx households/new.tsx households/[key]/edit.tsx
  (main)/families/new.tsx families/[key]/edit.tsx
  (main)/registrations/new.tsx registrations/[key]/edit.tsx
  (main)/visits/new.tsx visits/today.tsx activities/new.tsx
  (main)/attention/[seq].tsx conflicts/[seq].tsx  diagnostics.tsx (só build e2e)
contracts/field-sync/              cópia da tag field-sync-v1.2.0 (schemas, domains, examples) + SHA256SUMS
scripts/contract-types.mjs         gera src/contract/generated/*.ts
scripts/map-style-sync.mjs         copia map-style/dist da tag map-style-v1.0.0 para assets/map/ e grava assets/map/map-style.lock (Q11)
scripts/fake-field-server/         servidor falso do contrato para o Maestro da CI
scripts/check-apk.mjs              confere permissões e flags do APK (apkanalyzer)
maestro/                           fluxos
src/app/services.ts                AppServices (portas injetadas) e o provider
src/contract/                      generated/, validate.ts (Ajv, só teste), fixtures.ts
src/db/                            types.ts, open.ts, expoDb.ts, migrations.ts, events.ts, testing/sqljsDb.ts, repos/*.ts
src/security/                      secureStore.ts, pin.ts, lock.ts, wipe.ts
src/log.ts                         único ponto de log
src/setup/                         qr.ts, register.ts
src/sync/                          http.ts, outbox.ts, dispatch.ts, editable.ts, merge.ts, download.ts, reference.ts, round.ts, scheduler.ts, clock.ts, retention.ts
src/map/                           download.ts, style.ts, householdLayer.ts
src/location/capture.ts
src/rules/                         registry.ts, homeVisit.ts, collectiveActivity.ts (portes do dashboard)
src/features/<tela>/               componentes de tela e seus testes
src/features/messages/errorPhrases.ts
src/theme/tokens.ts                gerado dos design-tokens (tokens-v1.1.0: govbr e govbr-escuro)
assets/map/                        cópia do map-style-v1.0.0: style-light.json, style-dark.json, manifest.json, fonts/, sprites/, map-style.lock
```

Regra de fronteira: nada em `src/sync`, `src/rules`, `src/security`, `src/map/{download,style,householdLayer}.ts` importa módulo nativo diretamente; recebe a porta pelo `AppServices`.

## Esquema local (SQLCipher, `SCHEMA_VERSION = 1`)

| Tabela | Colunas (todas `NOT NULL` salvo `?`) |
|---|---|
| `meta` | `key` PK, `value` — `schema_version`, `cursor?`, `last_download_ok_at?`, `clock_skew_ms`, `city_offset_min`, `mobile_data` (`0/1`), `session_host`, `session_user_id` |
| `session` | `id` = 1, `body` (JSON da última `GET session`), `fetched_at` |
| `micro_areas` | `id` PK, `body`, `team_id` |
| `households` | `key` PK (id do servidor ou `client_uuid` de quem nasceu no tablet), `id?` único, `client_uuid?`, `micro_area_id`, `version?`, `body` (forma `<household>` do download, ou sintetizada), `base_version?`, `base_body?`, `local_body?` (projeção editável pendente), `needs_reload` (`0/1`; Q5), `street_norm`, `number`, `has_location` (`0/1`) |
| `families` | `id` PK ou `client_uuid`, `household_key`, `responsible_patient_id?` — só índice; o corpo vive em `households.body.families` |
| `registrations` | `key` PK, `id?` único, `client_uuid?`, `household_key?`, `family_id?`, `patient_id?`, `version?`, `body`, `base_version?`, `base_body?`, `local_body?`, `needs_reload` (`0/1`; Q5), `name_norm`, `birth_date?`, `mother_norm?` |
| `who_to_visit` | `micro_area_id` PK, `people` (JSON), `generated_at` |
| `reference` | `id` = 1, `version`, `body` |
| `outbox` | `seq` PK autoincremento, `client_uuid` único, `type`, `body`, `target_id?` (`key` da linha alvo; a tabela sai do `type`: `household`/`family` → `households`, `individual_registration` → `registrations`), `edit_base_version?`, `edit_base_body?`, `depends_on` (JSON `[{ client_uuid, kind: "cite" \| "version" }]`), `state`, `error?`, `path?`, `candidates?`, `confirmable?`, `conflicts?` (JSON), `sent_version?`, `attempts`, `created_at`, `updated_at` |
| `client_ids` | `client_uuid` PK, `type`, `id` — o mapa `client_uuid → id` do §6.3 |
| `local_visits` | `client_uuid` PK, `city_day`, `occurred_at`, `summary` (JSON sem observação), `state` (`pending` \| `accepted` \| `withdrawn`), `server_id?`, `accepted_at?` |
| `local_activities` | `client_uuid` PK, `held_on`, `summary`, `state`, `server_id?`, `accepted_at?` |
| `map_file` | `id` = 1, `version`, `sha256`, `size_bytes`, `path`, `partial_path?`, `partial_sha256?`, `resume_data?` |

`edit_base_version`, `edit_base_body`, `target_id` e a tabela `client_ids` são os acréscimos do §6.2 decididos na Q41; `depends_on.kind`, `sent_version` e `needs_reload` (Q5) são internos do app. Índices: `households(micro_area_id)`, `households(street_norm, number)`, `registrations(household_key)`, `registrations(name_norm, birth_date)`, `outbox(state, seq)`.

---

### Task 1: Prova técnica — PMTiles local e SQLCipher no emulador (decisão 23)

Diretório descartável, fora de qualquer repositório: `/Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/campo-proof/` (apagar depois de publicar o resultado). Esta task **bloqueia todas as outras**: se um critério falhar, **pare** e devolva ao coordenador com o resultado (decisão 5A volta ao usuário).

**Files:**
- Create: `.worktrees/campo-proof/` (app Expo mínimo, sem git)
- Modify (docs worktree): `superpowers/plans/2026-11-13-module-24c-field-app-contracts.md` — nova seção "Prova técnica do campo" (só esta task escreve ali)

**Interfaces:**
- Produces: o resultado publicado no contrato — versões exatas que funcionaram (Expo, RN, `@maplibre/maplibre-react-native`, `@protomaps/basemaps`, `expo-sqlite`, `cipher_version`, JDK, go-pmtiles — as de `@maplibre/maplibre-react-native`, `@protomaps/basemaps` e do CLI `pmtiles` ficam fixadas para todos, Q12: o api usa o CLI, o `map-style/` usa o basemaps), o commit de `protomaps/basemaps-assets` das fontes e sprites, o esquema de URL que funcionou para glyphs e sprites (`asset://` ou `file://`), o caminho do `.pmtiles`, tamanhos medidos de Curitiba e Maringá, o tempo do SHA-256, e o `sample.pmtiles` (≤ 1 MB) guardado para a Task 24 (e para o fixture do api, se couber). As fontes e os sprites `light`/`dark` guardados alimentam a Task 6 do plano do contracts-repo (`map-style-v1.0.0`).

- [ ] **Step 1: Preparar o ambiente Android (não tocar nos AVDs existentes)**

```bash
export ANDROID_HOME=$HOME/Library/Android/sdk
export PATH=$ANDROID_HOME/platform-tools:$ANDROID_HOME/emulator:$ANDROID_HOME/cmdline-tools/latest/bin:$PATH
sdkmanager --list_installed | grep -E 'system-images;android-34;google_apis;arm64-v8a|platforms;android-3[56]|build-tools;35'
avdmanager list device | grep -i -E 'pixel_tablet|pixel_c'
avdmanager create avd -n campo_api34 -k 'system-images;android-34;google_apis;arm64-v8a' -d pixel_tablet   # sem pixel_tablet: -d pixel_c
emulator -avd campo_api34 -no-snapshot-save -no-boot-anim -netdelay none -netspeed full &
adb wait-for-device && adb shell getprop sys.boot_completed
```
Expected: a imagem `android-34;google_apis;arm64-v8a` listada (já está no SDK); `sys.boot_completed` = `1`. Exporte `ANDROID_HOME` só no shell da task (não mude `~/.zshrc` sem pedir). JDK: use o que o RN 0.88 pede (`npx expo-doctor` e o README do template); a máquina tem 21 (Homebrew), 22 e 23 em `/usr/libexec/java_home -V`. Se a versão pedida não estiver instalada, **pare e peça ao usuário** (download).

- [ ] **Step 2: Medir o mapa por município e recortar a amostra**

Baixe o binário do `go-pmtiles` (release estável mais recente; anote a versão) — pedir autorização do download ao usuário. Use a build diária do Protomaps de uma data dos últimos 7 dias (`https://build.protomaps.com/<AAAAMMDD>.pmtiles`).

```bash
pmtiles extract https://build.protomaps.com/<AAAAMMDD>.pmtiles cwb.pmtiles --bbox=-49.40,-25.65,-49.18,-25.34 --maxzoom=15 --dry-run
pmtiles extract https://build.protomaps.com/<AAAAMMDD>.pmtiles mga.pmtiles --bbox=-52.10,-23.53,-51.85,-23.30 --maxzoom=15 --dry-run
pmtiles extract https://build.protomaps.com/<AAAAMMDD>.pmtiles sample.pmtiles --bbox=-49.2760,-25.4380,-49.2640,-25.4260 --maxzoom=15
pmtiles extract https://build.protomaps.com/<AAAAMMDD>.pmtiles cwb.pmtiles --bbox=-49.40,-25.65,-49.18,-25.34 --maxzoom=15
```
Expected: os dois `--dry-run` imprimem o tamanho estimado (anotar); `sample.pmtiles` ≤ 1 MB; `cwb.pmtiles` completo (para medir o SHA-256 no aparelho). Caixas aproximadas (com ~2 km de margem); a do api vem do IBGE.

- [ ] **Step 3: Criar o app mínimo com as dependências nativas**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees
npx create-expo-app@latest campo-proof --template blank-typescript
cd campo-proof && npm config set save-exact true --location=project
npx expo install expo-dev-client expo-sqlite expo-secure-store expo-file-system expo-asset expo-crypto @maplibre/maplibre-react-native
```
`app.json`: `android.package` `br.rotasaude.campo.proof`; plugins `["expo-sqlite", { "useSQLCipher": true }]` e o plugin do `@maplibre/maplibre-react-native`. Copie `sample.pmtiles` e `cwb.pmtiles` para `assets/`, e de `protomaps/basemaps-assets` (anote o commit) as fontes `Noto Sans Regular`, `Noto Sans Medium`, `Noto Sans Italic` (faixas `0-255`, `256-511`, `8192-8447`) e os sprites `light` e `dark` (`.json`, `.png`, `@2x`). Gere o estilo `light` com `@protomaps/basemaps` (`layers("protomaps", namedFlavor("light"), { lang: "pt" })`).

- [ ] **Step 4: Tela de mapa com duas variantes de glyphs/sprites**

Na abertura, copie `sample.pmtiles` para `FileSystem.documentDirectory + "map/sample.pmtiles"` e use a fonte `"pmtiles://" + documentDirectory + "map/sample.pmtiles"` (resulta em `pmtiles://file:///data/user/0/…/files/map/sample.pmtiles`). Variante A: `glyphs` e `sprite` em `asset://map/…` (arquivos copiados para `android/app/src/main/assets/map/` por um plugin local). Variante B: copiar `fonts/` e `sprites/` de `FileSystem.bundleDirectory` para `documentDirectory` e usar `file:///…/fonts/{fontstack}/{range}.pbf`. Um botão alterna as variantes.

```bash
npx expo prebuild --platform android --clean && npx expo run:android --device campo_api34
adb shell svc wifi disable; adb shell svc data disable      # sem rede
adb exec-out screencap -p > /tmp/claude-proof-map-A.png     # depois de tocar na variante B: map-B.png
```
Expected: com a rede desligada, ruas e nomes em português aparecem (glyphs ok) e os ícones de POI aparecem (sprites ok) em pelo menos uma variante; `adb logcat | grep -i maplibre` sem erro de fonte ausente na variante escolhida.

- [ ] **Step 5: Tela de SQLCipher**

Ao tocar "Testar cifra": gera 32 bytes (`Crypto.getRandomBytes(32)`), guarda no SecureStore, abre `proof.db`, `PRAGMA key = "x'<hex>'"`, `SELECT * FROM pragma_cipher_version` (ou `PRAGMA cipher_version`), cria tabela, grava uma linha, fecha; reabre com a chave e lê; reabre **sem** `PRAGMA key` e tenta `SELECT count(*) FROM sqlite_master`; reabre com chave errada e tenta o mesmo. Cada resultado vai para `console.log("CAMPO_PROOF", …)` (só nesta prova).

```bash
adb logcat -s ReactNativeJS | grep CAMPO_PROOF
adb exec-out run-as br.rotasaude.campo.proof sh -c 'head -c 16 files/SQLite/proof.db' | xxd
```
Expected: `cipher_version` devolve uma linha (anotar, ex.: `4.x.x community`); reabrir com a chave lê a linha; sem chave e com chave errada → erro `file is not a database`; o cabeçalho do arquivo **não** é `53514c69746520666f726d6174203300` ("SQLite format 3\0").

- [ ] **Step 6: Medir o SHA-256 do mapa da cidade**

Copie `cwb.pmtiles` para `documentDirectory` e meça: (a) `Crypto.digest(SHA256, await new File(uri).bytes())`; (b) `@noble/hashes/sha256` incremental lendo blocos de 1 MB. Anote tempo e se houve falta de memória.
Expected: ao menos uma das duas termina sem erro; anote qual a Task 14 usa (preferência: (a) se ≤ 15 s e sem erro).

- [ ] **Step 7: Publicar o resultado e avisar o coordenador**

Escreva no contrato a seção "Prova técnica do campo (Task 1)": data, critérios (mapa offline com glyphs e sprites; `cipher_version`; sem chave falha; cabeçalho cifrado), resultado de cada um, as versões e as escolhas do bloco Produces (inclusive as três versões fixadas da Q12 e o commit dos assets), os tamanhos de Curitiba e Maringá. Commit no worktree do docs:

```bash
G=/opt/homebrew/bin/git; D=/private/tmp/claude-501/-Users-eduardovrocha-Development-ioit-solutions-rota-saude/fd670630-7d57-4330-811c-0f40fb741325/scratchpad/docs-wt
$G -C $D add superpowers/plans/2026-11-13-module-24c-field-app-contracts.md
$G -C $D commit -m "docs: publish the field app technical proof result" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```
Avise o coordenador ("Próximos módulos") com o resultado — ele libera a Task 6 do contracts-repo (`map-style-v1.0.0`). Guarde `sample.pmtiles` e as fontes/sprites escolhidos fora do diretório da prova (scratchpad) para a Task 6 do contracts-repo e a Task 24. **Falhou um critério: pare aqui.**

---

### Task 2: Esqueleto do app e repositório `rotasaude/campo`

**Os passos 6–8 (criar o repositório remoto, labels, templates, proteção, push) exigem autorização explícita do usuário antes de rodar.** Os passos 1–5 são locais.

**Files:**
- Create: `apps/campo/{package.json, package-lock.json, app.config.ts, tsconfig.json, eslint.config.js, jest.config.js, jest.setup.ts, .gitignore, .npmrc, README.md, LICENSE}`
- Create: `app/_layout.tsx` (só `<Slot />` por enquanto), `src/app/services.ts`, `src/log.ts`
- Create: `.github/workflows/ci.yml` (lint, typecheck, jest)
- Test: `src/app/appConfig.test.ts`, `src/log.test.ts`

**Interfaces:**
- Produces: `buildConfig(env: NodeJS.ProcessEnv): ExpoConfig` (default export de `app.config.ts` chama esta); flags de build lidas só aqui: `EXPO_PUBLIC_E2E` (`"1"` = build e2e), `CAMPO_ALLOWED_HOST_SUFFIXES` (lista separada por vírgula, Q44; build de release sem ela lança "CAMPO_ALLOWED_HOST_SUFFIXES ausente"), `EXPO_PUBLIC_DEV_ORIGIN` (só dev/e2e). A lista vai para `extra.allowedHostSuffixes`.
- Produces: `log(event: LogEvent, fields?: { client_uuid?: string; type?: string; code?: string; seq?: number; status?: number }): void` em `src/log.ts`; `LogEvent` é união de literais (`"sync.round.start"`, `"sync.round.end"`, `"sync.item.result"`, `"db.open.failed"`, `"map.download.failed"`, …). ESLint `no-console: error` fora de `src/log.ts`.
- Produces: `AppServices` (interface) com as portas `db`, `secure`, `http`, `fs`, `net`, `location`, `camera`, `crypto` (`digestHex(text): Promise<string>`, `randomUUID(): string`), `now: () => number`, `random: (n: number) => Uint8Array` — as tasks seguintes acrescentam os tipos das portas.

- [ ] **Step 1: Criar o projeto**

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps
npx create-expo-app@latest campo --template blank-typescript
cd campo && printf 'save-exact=true\n' > .npmrc
npx expo install expo-router expo-dev-client expo-sqlite expo-secure-store expo-file-system expo-crypto expo-network expo-location expo-camera expo-asset @maplibre/maplibre-react-native
npm i -D jest@30 jest-expo @testing-library/react-native sql.js ajv ajv-formats json-schema-to-typescript @types/jest eslint eslint-config-expo
```
Versões dos pacotes Expo: as que o `expo install` escolher para o SDK 58 (anotar no README). Ajv e `sql.js` são só `devDependencies`.

- [ ] **Step 2: Escrever o teste de configuração**

```ts
// src/app/appConfig.test.ts
import { buildConfig } from "../../app.config";
const RELEASE = { CAMPO_ALLOWED_HOST_SUFFIXES: ".rotasaude.example" };
test("android hardening and identity", () => {
  const c = buildConfig(RELEASE);
  expect(c.android?.package).toBe("br.rotasaude.campo");
  expect(c.android?.allowBackup).toBe(false);
  expect([...(c.android?.permissions ?? [])].sort()).toEqual(
    ["ACCESS_COARSE_LOCATION", "ACCESS_FINE_LOCATION", "ACCESS_NETWORK_STATE", "CAMERA", "INTERNET"]);
  expect(c.plugins).toContainEqual(["expo-sqlite", { useSQLCipher: true }]);
  expect(JSON.stringify(c)).not.toMatch(/sentry|crashlytics|firebase|bugsnag|mapbox\.com|googleapis/i);
});
test("production build has no dev origin nor e2e entry", () => {
  const c = buildConfig({ ...RELEASE, EXPO_PUBLIC_DEV_ORIGIN: "http://10.0.2.2:3030" });
  expect(c.extra?.devOrigin).toBeUndefined();       // só com EXPO_PUBLIC_E2E=1 ou APP_VARIANT=development
  expect(c.extra?.e2e).toBe(false);
  expect(c.extra?.allowedHostSuffixes).toEqual([".rotasaude.example"]);
});
test("release build without the host suffix list fails (Q44 gate)", () => {
  expect(() => buildConfig({})).toThrow(/CAMPO_ALLOWED_HOST_SUFFIXES ausente/);
});
```
E `src/log.test.ts`: `log("sync.item.result", { client_uuid, type: "household", code: "stale_version" })` escreve uma linha com exatamente essas chaves; um teste de tipo (`// @ts-expect-error`) prova que `log("x", { full_name: "…" })` não compila.

- [ ] **Step 3: Rodar e ver falhar** — ainda sem `bin/test campo` (Task 3): rode só a checagem de tipos `npx tsc --noEmit` no host (não é teste). Expected: erro "buildConfig not exported".

- [ ] **Step 4: Implementar `app.config.ts`, `src/log.ts`, `src/app/services.ts`, configs de lint/jest/ts**

`app.config.ts`: `name` "Rota Saúde Campo", `slug` `campo`, `scheme` ausente (sem deep link), `newArchEnabled: true`, `orientation: "default"`, `userInterfaceStyle: "automatic"`, `android.blockedPermissions` com `RECORD_AUDIO`, `READ_EXTERNAL_STORAGE`, `WRITE_EXTERNAL_STORAGE`, `SYSTEM_ALERT_WINDOW`, `VIBRATE`, `WAKE_LOCK`, `FOREGROUND_SERVICE`, `READ_MEDIA_IMAGES`, `READ_MEDIA_VIDEO`, `ACCESS_BACKGROUND_LOCATION`; `expo-camera` com `recordAudioAndroid: false`; `expo-location` sem segundo plano; `extra.allowedHostSuffixes`, `extra.e2e`, `extra.devOrigin` (este só quando `APP_VARIANT=development` ou `EXPO_PUBLIC_E2E=1`). `jest.config.js`: preset `jest-expo/android`, `setupFilesAfterEnv` com `jest.setup.ts` (mocks de `expo-secure-store` em memória, `@maplibre/maplibre-react-native`, `expo-location`, `expo-camera`, `expo-network`). `.gitignore`: `android/`, `ios/`, `node_modules/`, `.expo/`, `*.jks`, `*.keystore`, `.env*`.

- [ ] **Step 5: README, LICENSE, CI**

README: o que é, stack com as versões fixadas, `bin/test campo`, build local (Task 25), "sem Expo Go, sem EAS". LICENSE: a mesma dos outros repositórios da org (copiar de `apps/api/LICENSE` ou do repo `contracts`). `.github/workflows/ci.yml`: job `test` em `container: mirror.gcr.io/library/node:22-alpine` com `npm ci`, `npx eslint .`, `npx tsc --noEmit`, `npx jest --ci`. (O job Maestro entra na Task 24.)

- [ ] **Step 6 (AUTORIZAÇÃO): Criar o repositório remoto e o primeiro commit em `main`**

Peça ao usuário: "Posso criar `rotasaude/campo` (público), com os 31 labels, os 5 templates do `.github`, `main` protegida como nos outros, e empurrar o esqueleto?". Com o sim:

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/campo
G=/opt/homebrew/bin/git; $G init -b main
$G add package.json package-lock.json .npmrc app.config.ts tsconfig.json eslint.config.js jest.config.js jest.setup.ts .gitignore README.md LICENSE app/_layout.tsx src/app/services.ts src/log.ts src/app/appConfig.test.ts src/log.test.ts .github/workflows/ci.yml assets
$G commit -m "chore: scaffold the field app" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
gh repo create rotasaude/campo --public --source . --remote origin --push
gh label clone rotasaude/api --repo rotasaude/campo --force
gh api repos/rotasaude/api/branches/main/protection > /tmp/claude-protection.json   # copiar a mesma regra
```
Templates: se os outros repositórios têm `.github/ISSUE_TEMPLATE/` próprios, copie os 5 de `rotasaude/api` num commit `chore: add issue templates`; se vêm do repositório `.github` da org, nada a fazer (conferir num issue novo). Aplique a proteção de `main` com `gh api -X PUT repos/rotasaude/campo/branches/main/protection --input <json ajustado>`. Abra o card no board #1 (F-24.18) em In Progress, um `gh` por comando.

- [ ] **Step 7: Worktree de trabalho**

```bash
G=/opt/homebrew/bin/git; $G -C /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/campo worktree add ../../.worktrees/campo-mod24c -b feat/mod-24c-field-app
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/campo-mod24c && npm ci
```
Expected: `.worktrees/campo-mod24c` existe com `node_modules` do host (para o Gradle); o container usa o volume próprio (Task 3).

- [ ] **Step 8: CI verde** — `gh run watch` no primeiro push. Expected: job `test` verde.

---

### Task 3: Alvo `campo` no `bin/test`

`bin/test`, `docker-compose.yml` e `CLAUDE.md` ficam na raiz do monorepo, **fora de git**: sem commit; avise a sessão "Testes" por `SendMessage` depois. A linha nova do `CLAUDE.md` só com autorização do usuário.

**Files:**
- Modify: `/Users/eduardovrocha/Development/ioit.solutions/rota-saude/docker-compose.yml` (serviço `campo-test` e volume `campo-node-modules`)
- Modify: `/Users/eduardovrocha/Development/ioit.solutions/rota-saude/bin/test` (uso, `case campo`, `jest_summary`)
- Modify (com autorização): `/Users/eduardovrocha/Development/ioit.solutions/rota-saude/CLAUDE.md` (linha `bin/test campo …` no bloco de uso)

**Interfaces:**
- Produces: `bin/test campo [-w <wt>] [args do jest]` — monta `.worktrees/<wt>` (ou `apps/campo`) em `/app`; roda `npm ci` só quando o SHA-256 do `package-lock.json` difere do carimbo `node_modules/.campo-lock.sha256`; depois `npx jest --ci <args>`; imprime falhas, totais e `resultado: ok|FALHOU (exit N)`; log em `.test-runs/campo[-wt]-<ts>.log`. Nunca sobe nem toca o `postgres-test`.

- [ ] **Step 1: Serviço no compose**

```yaml
  campo-test:
    profiles: ["test"]
    image: node:22-alpine
    working_dir: /app
    volumes:
      - ./apps/campo:/app
      - campo-node-modules:/app/node_modules
```
e `campo-node-modules:` em `volumes:`.

- [ ] **Step 2: `case campo)` no `bin/test`**

Mesmo padrão do bloco dos frontends (`run_locked`, rótulo `rs.test.owner`, `--no-deps`), com `docker compose --profile test run … campo-test sh -c 'want=$(sha256sum package-lock.json | cut -d" " -f1); have=$(cat node_modules/.campo-lock.sha256 2>/dev/null); [ "$want" = "$have" ] || { npm ci --no-audit --no-fund --silent && echo "$want" > node_modules/.campo-lock.sha256; }; npx jest --ci "$@"' sh <args>`. `jest_summary`: linhas `FAIL <arquivo>` (até 30), `● <suíte> › <teste>` (até 30), `Test Suites:`, `Tests:`, `Time:`; sem `Tests:` no log → "jest não rodou (exit N)" e as 20 últimas linhas. Atualize as linhas 4–6 do cabeçalho (uso).

- [ ] **Step 3: Verificar**

Run: `bin/test campo -w campo-mod24c`
Expected: `Tests: 4 passed` (Task 2: três de configuração e um de log) e `resultado: ok`.
Run (temporário, não commitar): acrescente `test("x", () => expect(1).toBe(2))` num arquivo novo e rode `bin/test campo -w campo-mod24c src/tmp.test.ts`; Expected: `FAIL src/tmp.test.ts`, `resultado: FALHOU (exit 1)`. Apague o arquivo.

- [ ] **Step 4: Avisar** — `SendMessage` à sessão "Testes": "bin/test ganhou o alvo campo (Jest no container campo-test, profile test, sem postgres-test); uso: bin/test campo [-w <wt>] [args do jest]". Proponha ao usuário a linha do `CLAUDE.md` e só edite com o sim.

---

### Task 4: Contrato `field-sync` v1.2.0 e tipos gerados

Depende da tag `field-sync-v1.2.0` no `rotasaude/contracts` (ordem do contrato §11). Sem a tag: pare e avise o coordenador.

**Files:**
- Create: `contracts/field-sync/` (cópia exata da tag: os 5 schemas, `reference-response-v1.json`, `domains-v1.json`, `examples/` com `manifest.json`), `contracts/field-sync/SHA256SUMS`, `contracts/field-sync/TAG` (`field-sync-v1.2.0 <commit>`)
- Create: `scripts/contract-types.mjs`, `src/contract/generated/{session,uploadRequest,uploadResponse,changes,reference,domains}.ts`, `src/contract/index.ts` (reexporta e define os apelidos usados no app)
- Create: `src/contract/validate.ts` (só teste), `src/contract/fixtures.ts`
- Test: `src/contract/contract.sha.test.ts`, `src/contract/contract.examples.test.ts`, `src/contract/contract.types.test.ts`

**Interfaces:**
- Produces (`src/contract/index.ts`): `SessionResponse`, `UploadRequest`, `UploadItem`, `ItemType` (`"home_visit" | "home_visit_withdrawal" | "collective_activity" | "patient" | "household" | "family" | "individual_registration"`), `UploadResult`, `ChangesResponse`, `HouseholdBody` (forma `<household>`), `RegistrationBody` (forma `<registration>`), `MicroAreaBody`, `WhoItem`, `ReferenceResponse`, `Location` (`{ latitude: number; longitude: number; accuracy_m?: number; source: "gps" | "map" }` — `accuracy_m` obrigatório com `gps` pelo esquema, Q3), `KNOWN_HTTP_ERRORS: readonly string[]` (com `device_suspended`, `map_not_found`, `range_not_satisfiable`, `unknown_city`, `city_suspended`, `city_schema_behind`, Q8), `KNOWN_ITEM_ERRORS: readonly string[]` (lidos de `domains-v1.json`).
- Produces (`src/contract/validate.ts`, só para testes): `expectValid(schema: SchemaName, value: unknown): void` (lança com os erros do Ajv 2020, criado com `multipleOfPrecision: 8` para bater com o `json_schemer` do api, Q49), `SchemaName = "session-response" | "upload-request" | "upload-response" | "changes-response" | "reference-response"`.
- Produces (`src/contract/fixtures.ts`): `example<T>(file: string): T` (lê `contracts/field-sync/examples/<file>`).

- [ ] **Step 1: Copiar a tag**

```bash
G=/opt/homebrew/bin/git; C=/Users/eduardovrocha/Development/ioit.solutions/rota-saude/contracts; W=/Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/campo-mod24c
$G -C $C fetch --tags && $G -C $C rev-parse field-sync-v1.2.0^{commit}
mkdir -p $W/contracts/field-sync && $G -C $C archive field-sync-v1.2.0 field-sync | tar -x -C $W/contracts --exclude='field-sync/tools'
(cd $W/contracts/field-sync && find . -type f ! -name SHA256SUMS ! -name TAG | sort | xargs shasum -a 256 > SHA256SUMS)
```

- [ ] **Step 2: Escrever os testes**

```ts
// contract.sha.test.ts — os arquivos batem com SHA256SUMS e TAG nomeia field-sync-v1.2.0
// contract.examples.test.ts — para cada suíte do manifest: expect "valid" → expectValid passa; senão lança
test.each(cases)("%s %s", (schema, file, expected) => { /* valid passa; inválido lança */ });
test("session example carries map and reference_version", () => {
  const s = example<SessionResponse>("session-acs.json");      // a v1.2.0 põe map e reference_version nele; session-without-map.json tem map null
  expect(s.reference_version).toMatch(/^[0-9a-f]{64}$/);
  expect(s.map?.sha256).toMatch(/^[0-9a-f]{64}$/);
});
test("Ajv agrees with json_schemer on 8 decimals (Q49)", () => {
  // -25.42847123 passa e -25.428471234 falha em location.latitude, como nos exemplos invalid-location-nine-decimals.json
});
test("http error list carries the 24c codes (Q8)", () => {
  expect(KNOWN_HTTP_ERRORS).toEqual(expect.arrayContaining(["device_suspended", "map_not_found", "range_not_satisfiable", "unknown_city", "city_suspended", "city_schema_behind"]));
});
// contract.types.test.ts — rodar compile() de json-schema-to-typescript em memória para cada schema e comparar com src/contract/generated/*.ts
```

- [ ] **Step 3: Rodar e ver falhar** — `bin/test campo -w campo-mod24c src/contract` · Expected: FAIL (`validate.ts`/`generated` ausentes).

- [ ] **Step 4: Implementar** `scripts/contract-types.mjs` (`compile(schema, title, { bannerComment: "// gerado de contracts/field-sync (field-sync-v1.2.0) — não editar", additionalProperties: false, strictIndexSignatures: true })` para cada schema → `src/contract/generated/<nome>.ts`), `validate.ts` (Ajv2020 + `ajv-formats`, `strict: false`, `multipleOfPrecision: 8`, schemas carregados por `$id`), `fixtures.ts`, `index.ts`. Rode `node scripts/contract-types.mjs` no host e commite o gerado.

- [ ] **Step 5: Rodar e ver passar** — `bin/test campo -w campo-mod24c src/contract` · Expected: PASS, todas as suítes do manifest.

- [ ] **Step 6: Commit** — arquivos: `contracts/field-sync` (lista explícita gerada por `$G -C $W status --porcelain contracts`, um `add` com os caminhos), `scripts/contract-types.mjs`, `src/contract/*`. Mensagem: `feat: pin the field-sync v1.2.0 contract and generate its types`.

---

### Task 5: Banco local cifrado e migrações

**Files:**
- Create: `src/db/types.ts`, `src/db/open.ts`, `src/db/expoDb.ts`, `src/db/migrations.ts`, `src/db/events.ts`, `src/db/testing/sqljsDb.ts`
- Create: `src/security/secureStore.ts` (porta + chaves)
- Test: `src/db/open.test.ts`, `src/db/migrations.test.ts`

**Interfaces:**
- Produces (`src/db/types.ts`):
```ts
export type SqlParam = string | number | null;
export interface Db {
  exec(sql: string): Promise<void>;
  run(sql: string, params?: SqlParam[]): Promise<{ changes: number; lastInsertRowId: number }>;
  all<T>(sql: string, params?: SqlParam[]): Promise<T[]>;
  get<T>(sql: string, params?: SqlParam[]): Promise<T | null>;
  tx<T>(fn: (db: Db) => Promise<T>): Promise<T>;   // BEGIN IMMEDIATE … COMMIT; ROLLBACK em exceção
  close(): Promise<void>;
}
```
- Produces (`src/security/secureStore.ts`): `SecureStorePort { get(k): Promise<string | null>; set(k, v): Promise<void>; delete(k): Promise<void> }`; `SECURE_KEYS = { dbKey: "campo.db_key", token: "campo.token", host: "campo.host", userId: "campo.user_id", pinHash: "campo.pin_hash", pinSalt: "campo.pin_salt", pinFailures: "campo.pin_failures", locked: "campo.locked" }`.
- Produces (`src/db/open.ts`): `openEncryptedDb(deps: { secure: SecureStorePort; random: (n: number) => Uint8Array; openRaw: (name: string) => Promise<Db> }, opts?: { create: boolean }): Promise<Db>`; `class UnencryptedDatabaseError extends Error`; `DB_NAME = "campo.db"`. Sem chave e `create: false` → lança `MissingKeyError`.
- Produces (`src/db/migrations.ts`): `SCHEMA_VERSION = 1`; `migrate(db: Db): Promise<void>` (cria as tabelas do "Esquema local"; idempotente).
- Produces (`src/db/events.ts`): `dbEvents.emit(table: string)`, `dbEvents.on(table, fn): () => void`, e o hook `useLive<T>(tables: string[], read: (db: Db) => Promise<T>): T | undefined`.
- Produces (`src/db/testing/sqljsDb.ts`): `createTestDb(): Promise<Db>` (`sql.js` em memória, já migrado).

- [ ] **Step 1: Escrever os testes**

```ts
// open.test.ts — openRaw falso que grava os SQL executados e responde cipher_version conforme o caso
test("creates a 32-byte key once and keys the database before anything else", async () => {
  const sqls: string[] = []; const db = await openEncryptedDb(fakeDeps({ sqls, cipher: "4.6.1 community" }), { create: true });
  expect(sqls[0]).toMatch(/^PRAGMA key = "x'[0-9a-f]{64}'"$/);
  expect(sqls[1]).toMatch(/cipher_version/i);
  expect((await secure.get("campo.db_key"))!).toHaveLength(64);
});
test("refuses an unencrypted build: no row from cipher_version", async () => {
  await expect(openEncryptedDb(fakeDeps({ sqls, cipher: null }))).rejects.toBeInstanceOf(UnencryptedDatabaseError);
  expect(sqls.some((s) => /CREATE TABLE|SELECT .* FROM (outbox|households)/i.test(s))).toBe(false);
});
test("reuses the stored key", …);                    // segunda abertura não chama random
test("missing key without create → MissingKeyError", …);
// migrations.test.ts — createTestDb(): todas as tabelas e índices do esquema existem; migrate duas vezes não falha; meta.schema_version = "1"
```

- [ ] **Step 2: Rodar e ver falhar** — `bin/test campo -w campo-mod24c src/db` · Expected: FAIL.

- [ ] **Step 3: Implementar.** `expoDb.ts` adapta `SQLite.openDatabaseAsync(DB_NAME)` (`execAsync`, `runAsync`, `getAllAsync`, `getFirstAsync`, `withExclusiveTransactionAsync`) ao `Db`; uma conexão só por processo. A chave em hex vem de `random(32)`. A mensagem do erro fatal na tela (Task 16): "Banco sem cifra: o app não abre. Reinstale pela loja da secretaria."

- [ ] **Step 4: Rodar e ver passar** — mesmo comando · Expected: PASS.

- [ ] **Step 5: Commit** — `src/db/*`, `src/security/secureStore.ts` e testes · `feat: open the local database only when SQLCipher answers`.

---

### Task 6: PIN, bloqueio e apagar

**Files:**
- Create: `src/security/pin.ts`, `src/security/lock.ts`, `src/security/wipe.ts`
- Test: `src/security/pin.test.ts`, `src/security/lock.test.ts`, `src/security/wipe.test.ts`

**Interfaces:**
- Consumes: `SecureStorePort`, `SECURE_KEYS` (Task 5), `Db`, `createTestDb`.
- Produces: `PIN_LENGTH = 6`, `MAX_PIN_FAILURES = 5`, `LOCK_AFTER_MS = 5 * 60_000`; `setPin(secure, pin: string, random): Promise<void>` (recusa ≠ 6 dígitos; zera falhas e `locked`); `checkPin(secure, pin): Promise<"ok" | "wrong" | "locked">` (5ª falha seguida grava `campo.locked = "1"` e devolve `"locked"`; acerto zera); `isLocked(secure): Promise<boolean>`; `shouldLock(backgroundedAt: number | null, now: number): boolean`.
- Produces: `wipeTerritory(db): Promise<void>` (apaga `micro_areas`, `households`, `families`, `registrations`, `who_to_visit`, `local_visits`, `local_activities`, e `meta.cursor`/`last_download_ok_at`; mantém `outbox`, `client_ids`, `reference`, `map_file`, `session`); `wipeAll(deps: { db: Db; secure: SecureStorePort; fs: FsPort; closeDb: () => Promise<void> }): Promise<void>` (fecha o banco, apaga o arquivo do banco e o diretório `map/`, apaga todas as `SECURE_KEYS`). `FsPort` (`deleteAsync(uri, { idempotent: true })`, `documentDirectory`) é definida aqui.

- [ ] **Step 1: Escrever os testes**

```ts
test("stores only salt and SHA-256(salt+pin)", async () => { await setPin(secure, "123456", rnd); expect(await secure.get("campo.pin_hash")).not.toContain("123456"); });
test("five wrong in a row lock, and the lock survives a restart", async () => {
  for (let i = 0; i < 4; i++) expect(await checkPin(secure, "000000")).toBe("wrong");
  expect(await checkPin(secure, "000000")).toBe("locked");
  expect(await checkPin(secure, "123456")).toBe("locked");          // o PIN certo não destrava
  expect(await isLocked(secureReloaded)).toBe(true);
});
test("a right PIN resets the failure count", …);
test("shouldLock: 4:59 no, 5:00 yes, cold start (null) yes", …);
test("wipeTerritory keeps outbox, reference and map", …);
test("wipeAll deletes the db file, map dir and every secure key", …);
```

- [ ] **Step 2: Rodar e ver falhar** — `bin/test campo -w campo-mod24c src/security` · Expected: FAIL.
- [ ] **Step 3: Implementar** (`expo-crypto` `digestStringAsync(SHA256, salt + pin)` pela porta `crypto` do `AppServices`; sal de 16 bytes em hex).
- [ ] **Step 4: Rodar e ver passar** · Expected: PASS.
- [ ] **Step 5: Commit** — `src/security/{pin,lock,wipe}.ts` e testes · `feat: add the app PIN, background lock and data wipe`.

---

### Task 7: Cliente HTTP e frases de erro

**Files:**
- Create: `src/sync/http.ts`, `src/features/messages/errorPhrases.ts`
- Test: `src/sync/http.test.ts`, `src/features/messages/errorPhrases.test.ts`

**Interfaces:**
- Consumes: tipos do contrato (Task 4), `KNOWN_HTTP_ERRORS`, `KNOWN_ITEM_ERRORS`.
- Produces:
```ts
export type HttpFailure =
  | { kind: "revoked" }                 // 401 device_revoked
  | { kind: "suspended" }               // 401 device_suspended ou invalid_token (decisão 11)
  | { kind: "feature_disabled" }        // 403 feature_disabled
  | { kind: "rate_limited" }            // 429
  | { kind: "not_found"; code: string } // 404 (map_not_found, unknown_city)
  | { kind: "range" }                   // 416 range_not_satisfiable (o mapa recomeça do zero, Q34)
  | { kind: "server"; status: number; code: string | null }   // 5xx, 403/404 de cidade, outros
  | { kind: "network" }                 // sem resposta / tempo esgotado (20 s)
  | { kind: "invalid"; status: number; code: string | null }; // 4xx restantes (422 invalid_request …)
export class FieldHttpError extends Error { constructor(public failure: HttpFailure) }
export interface FieldClient {
  session(): Promise<SessionResponse>;
  uploads(items: UploadItem[]): Promise<UploadResponse>;
  changes(cursor: string | null, limit?: number): Promise<ChangesResponse>;
  reference(): Promise<ReferenceResponse>;
  mapRequest(resume?: { ifRange: string }): { url: string; headers: Record<string, string> };   // com resume: If-Range: "<sha256>" (Q34)
}
export function createFieldClient(o: { host: string; getToken: () => Promise<string | null>; fetch: typeof fetch; devOrigin?: string; timeoutMs?: number }): FieldClient;
```
  Origem: `https://<host>`; com `devOrigin` (só builds dev/e2e) a requisição vai para `devOrigin` com o cabeçalho `Host: <host>` (a cidade é resolvida pelo host, `CityResolution`). Cabeçalhos: `Authorization: Bearer <token>`, `Accept: application/json`, `Content-Type: application/json` nas escritas. `changes`: `?limit=` e `?cursor=` (o cursor é opaco e sem dado pessoal).
- Produces: `phraseFor(code: string | null | undefined): string` e `ERROR_PHRASES: Record<string, string>`; desconhecido → "O Rota Saúde recusou este registro (código <code>). Fale com o enfermeiro da equipe." Fontes das frases: `HOME_VISIT_ERRORS` (`apps/dashboard/src/lib/homeVisits.ts`) e as do `territoryRegistry.ts` do dashboard, adaptadas ao campo. Os códigos de borda e do mapa (`map_not_found`, `range_not_satisfiable`, `unknown_city`, `city_suspended`, `city_schema_behind`) estão em `errors.http` da v1.2.0 (Q8) e têm frase como os demais; `device_suspended` cobre também o aparelho substituído (Q17): "Acesso suspenso: procure o enfermeiro da equipe ou leia o QR novo".

- [ ] **Step 1: Escrever os testes**

```ts
test("token only in the Authorization header, never in the URL", async () => {
  const calls = recordFetch(200, example("session-acs.json"));
  await client.session(); await client.changes("abc", 200);
  for (const c of calls) { expect(c.url).not.toContain("rsf_"); expect(c.headers.Authorization).toMatch(/^Bearer rsf_/); }
});
test.each([
  [401, { error: "device_revoked" }, { kind: "revoked" }],
  [401, { error: "device_suspended" }, { kind: "suspended" }],
  [401, { error: "invalid_token" }, { kind: "suspended" }],
  [403, { error: "feature_disabled" }, { kind: "feature_disabled" }],
  [429, { error: "too_many_requests" }, { kind: "rate_limited" }],
  [503, { error: "city_schema_behind" }, { kind: "server", status: 503, code: "city_schema_behind" }],
  [404, { error: "map_not_found" }, { kind: "not_found", code: "map_not_found" }],
  [416, { error: "range_not_satisfiable" }, { kind: "range" }],
])("maps %s %j", …);
test("network failure and 20 s timeout → network", …);
test("devOrigin sends Host header with the QR host", …);
// errorPhrases.test.ts
test("every known http and item code has its own phrase", () => {
  for (const c of [...KNOWN_HTTP_ERRORS, ...KNOWN_ITEM_ERRORS]) expect(ERROR_PHRASES[c]).toBeTruthy();   // a lista do domains já tem os códigos de borda (Q8)
});
test("phrases never echo values", () => { for (const p of Object.values(ERROR_PHRASES)) expect(p).not.toMatch(/\$\{|%s/); });
```

- [ ] **Step 2: Rodar e ver falhar** — `bin/test campo -w campo-mod24c src/sync/http.test.ts src/features/messages` · Expected: FAIL.
- [ ] **Step 3: Implementar** (`AbortController` com 20 s; corpo de erro lido como `{ error }`, ausente → `code: null`).
- [ ] **Step 4: Rodar e ver passar** · Expected: PASS.
- [ ] **Step 5: Commit** — `feat: add the field sync HTTP client and error phrases`.

---

### Task 8: Cadastro do aparelho por QR e PIN novo

**Files:**
- Create: `src/setup/qr.ts`, `src/setup/register.ts`, `src/features/setup/ScanScreen.tsx`, `src/features/setup/PinCreateScreen.tsx`, `app/(setup)/scan.tsx`, `app/(setup)/pin.tsx`
- Test: `src/setup/qr.test.ts`, `src/setup/register.test.ts`, `src/features/setup/PinCreateScreen.test.tsx`

**Interfaces:**
- Consumes: `createFieldClient`, `SECURE_KEYS`, `openEncryptedDb`, `wipeAll`, `setPin`.
- Produces: `QR_PREFIX = "rotasaude-field:v1:"`; `parseQrPayload(text: string, allowedHostSuffixes: string[]): { ok: true; host: string; token: string } | { ok: false; reason: "not_rotasaude" | "malformed" | "host_not_allowed" }` (base64url → JSON `{ host, token }`; `token` casa `^rsf_[A-Za-z0-9_-]{43}$`; `host` sem esquema, sem porta, minúsculo, termina num sufixo de `CAMPO_ALLOWED_HOST_SUFFIXES`; a origem é sempre `https://<host>` no build de produção — Q44).
- Produces: `registerDevice(deps, qr): Promise<RegisterOutcome>`, `RegisterOutcome = { kind: "registered" } | { kind: "token_replaced"; needsNewPin: boolean } | { kind: "refused_pending" } | { kind: "failed"; failure: HttpFailure }`. Regras: chama `GET session` com o token novo **antes** de gravar qualquer coisa; mesmo host **e** mesmo `user.id` do guardado (Q46) → troca só o token (`needsNewPin = isLocked`), mantém banco e fila — vale para o QR de "Substituir" e de "Cadastrar novo" do dashboard (Q16); outro host ou usuário com item na fila (estado ≠ removido) → `refused_pending`, nada muda; sem fila → `wipeAll`, depois cria chave e banco, grava token/host/userId, sessão em `session`.
- Produces: `PinCreateScreen` pede o PIN duas vezes ("Crie um PIN de 6 números" / "Repita o PIN"); diferentes → "Os PINs não são iguais".

- [ ] **Step 1: Escrever os testes**

```ts
test("valid payload", () => expect(parseQrPayload(QR_PREFIX + b64url({ host: "curitiba.rotasaude.example", token: TOKEN }), [".rotasaude.example"])).toEqual({ ok: true, host: "curitiba.rotasaude.example", token: TOKEN }));
test.each(["", "http://x", QR_PREFIX + "%%%", QR_PREFIX + b64url("[]"), QR_PREFIX + b64url({ host: "x" })])("malformed %s", …);
test("foreign host refused without network", …);                  // fetch nunca chamado
test("host with scheme or port refused", …);
test("same host and user: swaps token, keeps outbox and db", …);
test("same host and user from 'Cadastrar novo' (other device id) also swaps the token and keeps the outbox (Q16)", …);
test("same user id on another host is another user (Q46)", …);
test("other user with pending items: refused and nothing changes", …);
test("other user without pending: wipes everything then registers", …);
test("session failure leaves the previous token in place", …);
test("token never reaches log", () => { /* espia src/log; nenhuma chamada contém "rsf_" */ });
```

- [ ] **Step 2: Rodar e ver falhar** — `bin/test campo -w campo-mod24c src/setup src/features/setup` · Expected: FAIL.
- [ ] **Step 3: Implementar.** `ScanScreen`: `CameraView` com `barcodeScannerSettings={{ barcodeTypes: ["qr"] }}`, para no primeiro código; frases por `reason` ("Este QR não é do Rota Saúde", "QR ilegível — gere outro no dashboard", "Este QR aponta para um endereço que o app não reconhece"). Com `refused_pending`: "Há registros deste tablet ainda não enviados. Sincronize com o usuário atual antes de cadastrar outro." No build e2e (`extra.e2e`), um campo "Colar código" faz o mesmo que a câmera (Maestro; Q48).
- [ ] **Step 4: Rodar e ver passar** · Expected: PASS.
- [ ] **Step 5: Commit** — `feat: register the device from the dashboard QR and create the PIN`.

---

### Task 9: Fila de envio (ordem, dependências, versão, resultados)

**Files:**
- Create: `src/sync/outbox.ts`, `src/sync/dispatch.ts`, `src/db/repos/outboxRepo.ts`
- Test: `src/sync/outbox.test.ts`, `src/sync/dispatch.test.ts`

**Interfaces:**
- Consumes: `Db`, `createTestDb`, tipos do contrato, `expectValid` (testes).
- Produces (`outbox.ts`):
```ts
export type OutboxState = "pending" | "sending" | "needs_attention" | "conflict" | "blocked";
export interface Dep { client_uuid: string; kind: "cite" | "version" }
export function targetTable(type: ItemType): "households" | "registrations" | null; // household/family → households; individual_registration → registrations
export interface OutboxItem { seq: number; client_uuid: string; type: ItemType; body: Record<string, unknown>; target_id: string | null;
  edit_base_version: number | null; edit_base_body: Record<string, unknown> | null; depends_on: Dep[]; state: OutboxState;
  error: string | null; path: string | null; candidates: unknown[] | null; confirmable: boolean | null; conflicts: ConflictField[] | null;
  sent_version: number | null; attempts: number; created_at: string; updated_at: string }
export function enqueue(db: Db, input: { type: ItemType; body: Record<string, unknown>; targetId?: string; editBase?: { version: number; body: Record<string, unknown> }; deps?: Dep[]; clientUuid?: string }): Promise<OutboxItem>;
export function coalesceOrEnqueue(db: Db, input: …): Promise<OutboxItem>; // item pending/needs_attention do mesmo target_id e tipo → troca o body, mantém client_uuid e edit_base_*
export function recoverSending(db: Db): Promise<number>;                  // sending → pending, na abertura
export function pendingCount(db: Db): Promise<number>;                   // tudo que não saiu da fila
export function removeUnsent(db: Db, clientUuid: string): Promise<void>; // só pending/needs_attention, e cascata nos dependentes "cite"
```
- Produces (`dispatch.ts`):
```ts
export function selectBatch(items: OutboxItem[], max = 50): OutboxItem[];   // pura
export function resolveVersion(item: OutboxItem, rowBaseVersion: number | null): number | null; // pura
export type ApplySignal = { stop: "rate_limited" | null; progressed: boolean };
export function applyResults(db: Db, sent: OutboxItem[], results: UploadResult[], hooks: { onStale(item: OutboxItem, r: UploadResult): Promise<void> }): Promise<ApplySignal>;
export function sendQueue(deps: { db: Db; client: FieldClient; onStale: …}): Promise<{ stop: "rate_limited" | "http" | null; failure?: HttpFailure }>;
```

Regras de `selectBatch` (em ordem de `seq`, até `max`):
1. Só `pending`. Um item cujo dep está em `needs_attention`, `conflict` ou `blocked` é marcado `blocked` (em `sendQueue`, antes de selecionar) e não entra.
2. Dep `cite`: satisfeito se o dep já saiu da fila (está em `client_ids`) **ou** está neste lote antes dele.
3. Dep `version`: satisfeito só se o dep já saiu da fila (precisa da `version` do `accepted`; nunca no mesmo lote).
4. Dois itens com o mesmo `target_id` nunca no mesmo lote (o segundo espera o próximo).

`resolveVersion` (Q41): item com `target_id` e `edit_base_version` → `edit_base_version` (sempre a base da edição); sem `edit_base_version` (o alvo nasceu no tablet ou o item foi criado antes de existir versão) → `rowBaseVersion` (a da linha, atualizada pelo `accepted.version`). Nunca a versão trazida pelo download para uma edição já feita: mudança do posto no meio do caminho volta como `stale_version` e passa pela mescla. O `body.version` é preenchido só no envio, e `sent_version` grava o que foi.

`applyResults`, por resultado (casado pelo `client_uuid`; sem `client_uuid` no resultado → `needs_attention` `invalid_request` na posição):
| Resultado | Efeito |
|---|---|
| `accepted` / `duplicate` com `version` | remove da fila; `client_ids(client_uuid, type, id)`; com `target_id`: linha alvo `id = id`, `base_version = version`, `base_body` = a projeção enviada; itens seguintes `pending` do mesmo `target_id` com `edit_base_version = sent_version` passam a `edit_base_version = version` (são edições feitas por cima desta, não vindas do download); `local_visits`/`local_activities` → `accepted` |
| `duplicate` sem `version` (recibo anterior ao v1.2.0, Q5) | remove da fila e grava `client_ids` como acima; a linha alvo fica com `needs_reload = 1` e `base_*` intocados, sem mescla; o próximo download que trouxer a linha a substitui e zera a marca (Task 11) |
| `stale_version` | `hooks.onStale` (Task 10) |
| `rejected` `too_many_requests` | volta `pending`, `attempts + 1`; sinal `stop: "rate_limited"` (os seguintes do lote também voltam `pending`) |
| `rejected` `feature_disabled` | volta `pending` (decisão 22) |
| `rejected` `household_changed` ou `internal_error` | transitórios (Q45): volta `pending` com `attempts + 1`; na 3ª rodada → `needs_attention` |
| `rejected` `not_found` com `path` terminando em `_client_uuid` e o citado recusado neste lote | `blocked` |
| outro `rejected` | `needs_attention` com `error`, `path`, `candidates`, `confirmable`; dependentes → `blocked` |

- [ ] **Step 1: Escrever os testes** (todo corpo enviado passa por `expectValid("upload-request", { items })`; toda resposta fabricada por `expectValid("upload-response", …)`)

```ts
test("keeps creation order and sends at most 50", …);
test("cite dependency may travel in the same batch after its target", …);   // patient → individual_registration(patient_client_uuid)
test("version dependency never shares a batch", …);                         // household novo → family(household_client_uuid, version)
test("two items on the same target go in successive batches", …);
test("accepted removes, maps client_uuid → id and fills base_version from result.version", …);
test("duplicate leaves the queue like accepted", …);
test("duplicate without version marks the row for reload, no merge (Q5)", …);
test("household_changed goes back to pending twice and needs attention on the third round (Q45)", …);
test("rejected stays as needs_attention with phrase code and path; dependents become blocked", …);
test("resend after fix keeps the same client_uuid", …);
test("too_many_requests puts the rest back to pending and stops the round", …);
test("recoverSending turns sending into pending after a crash", async () => { /* estado sending no banco → recoverSending → pending; reenvio aceita duplicate */ });
test("sends the edit base version after a newer download", async () => {
  // edição feita na v3; download trouxe v4 para a linha (sem tocar no item); o envio leva version: 3
});
test("later edit on top of an accepted one is rebased to the accepted version", …);
test("coalesce: a second edit of a pending creation replaces its body, same client_uuid", …);
```

- [ ] **Step 2: Rodar e ver falhar** — `bin/test campo -w campo-mod24c src/sync/outbox.test.ts src/sync/dispatch.test.ts` · Expected: FAIL.
- [ ] **Step 3: Implementar** (`client_uuid` = `Crypto.randomUUID()` minúsculo; `sendQueue` repete lotes até não haver item elegível, até `stop`, ou até falha HTTP; marca `sending` na transação antes do POST).
- [ ] **Step 4: Rodar e ver passar** · Expected: PASS.
- [ ] **Step 5: Commit** — `feat: send the outbox in ordered batches with dependencies and versions`.

---

### Task 10: Mescla campo a campo (3B)

**Files:**
- Create: `src/sync/editable.ts`, `src/sync/merge.ts`, `src/sync/stale.ts`
- Test: `src/sync/merge.test.ts`, `src/sync/editable.test.ts`, `src/sync/stale.test.ts`

**Interfaces:**
- Consumes: `OutboxItem`, `enqueue`/repos (Task 9), `HouseholdBody`, `RegistrationBody`.
- Produces (`editable.ts`, projeção editável = forma do corpo de envio, sem `families` e sem `patient`; Q42):
  - `householdEditable(h: HouseholdBody): HouseholdEditable` = `{ micro_area_id: h.micro_area.id, address, property_type, living_conditions, contact_phone, refused, status, location }`.
  - `familyEditable(h: HouseholdBody, familyId: string): FamilyEditable | null` = `{ income_band, members_count, living_since, moved_out }` da família com esse `id` (Q43).
  - `registrationEditable(r: RegistrationBody): RegistrationEditable` = `{ household_id, family_id, is_responsible, team_id: r.team.id, patient_demographics, identification, sociodemographic, health_conditions, street_situation, refused }` (o `patient` fica fora: a edição de identidade e a saída do cadastro ficam fora do app no 24c, "Faça pelo dashboard"; Q47).
- Produces (`merge.ts`):
```ts
export const DICTIONARY_SECTIONS = ["living_conditions", "identification", "sociodemographic", "health_conditions", "street_situation"] as const;
export interface ConflictField { path: string; station: unknown; tablet: unknown }
export function merge(base: Record<string, unknown>, local: Record<string, unknown>, current: Record<string, unknown>): { body: Record<string, unknown>; conflicts: ConflictField[] };
export function applyChoices(body: Record<string, unknown>, conflicts: ConflictField[], choices: Record<string, "station" | "tablet">): Record<string, unknown>;
export function canonical(v: unknown): string;   // JSON com chaves ordenadas; undefined → "∅"
```
  Campos: cada chave de primeiro nível; dentro de cada seção de `DICTIONARY_SECTIONS` que é objeto nos três lados, cada chave da seção (`path` = `seção.chave`); seção nula ou ausente em algum lado conta como um campo só; listas e objetos fora das seções (`address`, `location`, `patient_demographics`) são um campo. Por campo: `local == base` → `current`; `current == base` → `local`; `local == current` → esse; senão conflito (o corpo leva `current` até a escolha). Chave ausente no resultado é omitida.
- Produces (`stale.ts`): `handleStale(db: Db, item: OutboxItem, r: UploadResult): Promise<void>` — base = `item.edit_base_body`, local = projeção do `item.body` (sem ids/version), current = projeção de `r.current` (para `family`: `familyEditable(r.current, family_id)`; família nova sem `family_id` → só troca a versão). Sem conflito: `body` = mesclado + ids, `edit_base_version = r.current_version`, `edit_base_body` = projeção de `current`, `pending`, mesmo `client_uuid`. Com conflito: `conflict` com `conflicts`. Família com `family_id` que sumiu do `current` → `needs_attention` `family_not_found` (Q43). Em todos os casos a linha alvo recebe `body = r.current`, `version`/`base_* = current` e `needs_reload = 0`; `r.id` de `family` é o domicílio (Q4).
- Produces: `resolveConflict(db, seq: number, choices: Record<string, "station" | "tablet">): Promise<void>` — exige uma escolha por `path`; volta a `pending`.

- [ ] **Step 1: Escrever os testes (um por caso)**

```ts
const base = { property_type: "1", contact_phone: "4100", address: { street: "A", number: "1" }, living_conditions: { abastecimento_de_agua: 117, destino_do_lixo: 1 } };
test("only tablet changed → tablet wins", () => expect(merge(base, { ...base, contact_phone: "4199" }, base)).toEqual({ body: { ...base, contact_phone: "4199" }, conflicts: [] }));
test("only station changed → station wins", …);
test("both changed to the same value → no conflict", …);
test("both changed differently → conflict with station and tablet values", () => {
  const r = merge(base, { ...base, contact_phone: "1" }, { ...base, contact_phone: "2" });
  expect(r.conflicts).toEqual([{ path: "contact_phone", station: "2", tablet: "1" }]);
});
test("dictionary section merges key by key", () => {
  const local = { ...base, living_conditions: { ...base.living_conditions, abastecimento_de_agua: 118 } };
  const current = { ...base, living_conditions: { ...base.living_conditions, destino_do_lixo: 2 } };
  expect(merge(base, local, current).body.living_conditions).toEqual({ abastecimento_de_agua: 118, destino_do_lixo: 2 });
});
test("lists are one field (deficiencias_cidadao changed on both → one conflict)", …);
test("address is one field: street on tablet, number on station → conflict on address", …);
test("location is one field", …);
test("key added inside a section by station only is kept", …);
test("section null on one side counts as a single field", …);
test("key order does not create a false conflict", …);
test("absent vs null are different values", …);
test("applyChoices requires every path", …);
// stale.test.ts: household sem conflito volta pending com current_version; com conflito vira conflict; family existente mescla os campos da família; family nova só troca a versão; family sumida → family_not_found; a linha recebe current
```

- [ ] **Step 2: Rodar e ver falhar** — `bin/test campo -w campo-mod24c src/sync/merge.test.ts src/sync/editable.test.ts src/sync/stale.test.ts` · Expected: FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4: Rodar e ver passar** · Expected: PASS.
- [ ] **Step 5: Commit** — `feat: merge field edits with the station version field by field`.

---

### Task 11: Download (changes, reset, remoções, quem visitar)

**Files:**
- Create: `src/sync/download.ts`, `src/db/repos/territoryRepo.ts`, `src/sync/text.ts` (normalização)
- Test: `src/sync/download.test.ts`, `src/sync/text.test.ts`

**Interfaces:**
- Consumes: `FieldClient.changes`, `wipeTerritory`, `Db`.
- Produces: `normalizeText(s: string): string` (sem acento, minúsculo, espaços simples).
- Produces: `applyPage(db: Db, page: ChangesResponse): Promise<void>` — numa transação: `reset: true` → `wipeTerritory` primeiro; itens (`micro_area` substitui; `household`/`registration`: acha a linha pelo `id`, ou por `client_ids` quando nasceu no tablet; sem `local_body` → `body`, `version`, `base_*` = o item; com `local_body` → só `body`/`version`/`base_*` da linha, o item da fila (e o seu `edit_base_*`) nunca muda (Q41); em todo caso `needs_reload = 0` (Q5); reindexa `families` e campos de busca; `has_location` de `body.location`); `removals` (`moved`, `erased`, `out_of_scope`) apagam a linha e os cadastros do domicílio removido; `who_to_visit` não nulo substitui a tabela inteira; grava `meta.cursor = next_cursor` na mesma transação.
- Produces: `downloadAll(deps: { db; client; now }): Promise<{ complete: boolean }>` — `changes(cursor)` até `has_more: false`; completo grava `meta.last_download_ok_at`; falha HTTP propaga `FieldHttpError` (o que já foi aplicado fica).

- [ ] **Step 1: Escrever os testes** (páginas = exemplos do contrato, validados por `expectValid("changes-response", …)`)

```ts
test("full load from changes-first-page + changes-last-page", …);
test("reset wipes territory but never the outbox", …);
test("moved removal deletes the household and its registrations", …);
test("erased and out_of_scope are handled like moved", …);
test("who_to_visit null on middle pages keeps the old list; last page replaces it", …);
test("item with pending local edit: row updates, outbox item and its edit base untouched (Q41)", …);
test("row marked needs_reload is replaced and unmarked when the item arrives (Q5)", …);
test("server item that was born on the tablet replaces the local-born row by client_ids", …);
test("page failing midway applies nothing and keeps the previous cursor", async () => { /* db.run lança no 3º item → cursor igual, nenhuma linha nova */ });
test("next_cursor is stored even on the last page", …);
test("member whose family is missing (Q66) is shown without family, no error", …);
test("normalizeText('  Ána  Júlia ') === 'ana julia'", …);
```

- [ ] **Step 2: Rodar e ver falhar** — `bin/test campo -w campo-mod24c src/sync/download.test.ts src/sync/text.test.ts` · Expected: FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4: Rodar e ver passar** · Expected: PASS.
- [ ] **Step 5: Commit** — `feat: download the device scope with reset, removals and who to visit`.

---

### Task 12: Referência offline e dicionários

**Files:**
- Create: `src/sync/reference.ts`, `src/rules/dictionaries.ts`
- Test: `src/sync/reference.test.ts`, `src/rules/dictionaries.test.ts`

**Interfaces:**
- Consumes: `FieldClient.reference`, `ReferenceResponse`, exemplo `reference` do contrato.
- Produces: `RegistrationDictionaries = { domains: Record<string, { code: number; label: string }[]>; stringDomains: Record<"uf" | "tipo_logradouro", { code: string; label: string }[]>; sections: Record<SectionKey, FieldSpec[]> }` (`FieldSpec` com a forma de `territoryDictionaries.generated.ts` do dashboard).
- Produces: `parseRegistrationDictionaries(raw: unknown, stringDomains: unknown): RegistrationDictionaries` — domínio só como lista `[{ code, label }]` na ordem do YAML (Q27; o esquema da v1.2.0 recusa objeto); domínio fora dessa forma ou campo desconhecido → lança `ReferenceShapeError`.
- Produces: `Reference = { version: string; dictionaries: RegistrationDictionaries; neighborhoods: { id: string; name: string }[]; healthUnits: { id: string; cnes: string | null; name: string }[]; teamProfessionals: { team_id; professional_id; name; cbo; has_cns: boolean }[]; homeVisitDomains: HomeVisitDomains; collectiveDomains: CollectiveDomains }` (os tipos `HomeVisitDomains`/`CollectiveDomains` = forma do §7 do 24b, portados do `lib/api.ts` do dashboard; `collectiveDomains` é um bloco só, o recorte por equipe vem de `teamProfessionals.team_id`, Q7). `healthUnitOptions(ref)` e `professionalOptions(ref, teamId)` adaptam à forma que as regras portadas do dashboard esperam (`has_cnes = cnes !== null`; `has_cns`; Q6).
- Produces: `refreshReference(db, client, sessionVersion: string): Promise<"unchanged" | "updated" | "kept_previous">` — só baixa com `reference_version` diferente do guardado; forma inválida → mantém a anterior; `loadReference(db): Promise<Reference | null>`.

- [ ] **Step 1: Escrever os testes**

```ts
test("downloads only when reference_version changes", …);
test("keeps LEDI order of a domain sent as a list", () => expect(dict.domains.abastecimento_de_agua.map((d) => d.code).slice(0, 3)).toEqual([117, 214, 118]));
test("invalid shape keeps the previous reference", …);
test("a domain sent as an object is refused and the previous reference stays (Q27)", …);
test("professionalOptions keeps has_cns and filters by team; unit without CNES has has_cnes false (Q6)", …);
test("reference example validates against reference-response-v1.json", …);
test("home visit and activity codes in the reference equal those of domains-v1.json", …); // regras de tela e domains, um dado só (Q13 do 24b)
```
- [ ] **Step 2–4:** `bin/test campo -w campo-mod24c src/sync/reference.test.ts src/rules/dictionaries.test.ts` FAIL → implementar → PASS.
- [ ] **Step 5: Commit** — `feat: keep the offline reference and parse registration dictionaries`.

---

### Task 13: Rodada de sincronização, revogação, 30 dias e relógio

**Files:**
- Create: `src/sync/round.ts`, `src/sync/scheduler.ts`, `src/sync/clock.ts`, `src/sync/retention.ts`
- Test: `src/sync/round.test.ts`, `src/sync/clock.test.ts`, `src/sync/retention.test.ts`, `src/sync/scheduler.test.ts`

**Interfaces:**
- Consumes: `sendQueue`, `handleStale`, `downloadAll`, `refreshReference`, `wipeAll`, `wipeTerritory`, `recoverSending`, `syncMapFile` (Task 14; injetado, opcional até lá).
- Produces (`clock.ts`): `SKEW_WARN_MS = 5 * 60_000`; `measureSkew(serverTime: string, deviceNowAtResponse: number): number`; `cityOffsetMinutes(serverTime: string): number` (do sufixo `±hh:mm` ou `Z`); `cityNowIso(deviceNow: number, skewMs: number, offsetMin: number): string` (ex.: `2026-11-13T09:15:00-03:00`, segundos inteiros); `cityToday(deviceNow, skewMs, offsetMin): string` (`AAAA-MM-DD`).
- Produces (`retention.ts`): `TERRITORY_MAX_DAYS = 30`, `TERRITORY_WARN_DAY = 25`; `territoryAge(lastOkAt: string | null, now: number): { days: number | null; warn: boolean; expired: boolean }`; `enforceTerritoryAge(db, now): Promise<boolean>` (expirado → `wipeTerritory`); `purgeAccepted(db, cityToday: string)` (visitas e atividades `accepted` de dia < hoje saem de `local_visits`/`local_activities`, decisão 27); `validityBand(registeredAt: string, cityToday: string): "valid" | "due_soon" | "expired"` (Q65: dia de `registered_at` + 24 meses de calendário, 29/02 → 28/02; `expired` se ≤ hoje; `due_soon` se ≤ 60 dias).
- Produces (`round.ts`):
```ts
export type RoundTrigger = "open" | "foreground" | "timer" | "button" | "after_pairing";
export type RoundOutcome =
  | { kind: "ok"; downloaded: boolean } | { kind: "skipped"; reason: "running" | "offline" | "locked" | "needs_wifi" }
  | { kind: "stopped"; reason: "rate_limited" | "server" | "network" }
  | { kind: "revoked" } | { kind: "suspended" } | { kind: "feature_disabled" };
export function runRound(deps: RoundDeps, trigger: RoundTrigger): Promise<RoundOutcome>;
```
  Ordem: trava (uma por vez) → `recoverSending` → `GET session` (grava `session`, skew, offset; `reference_version` diferente → `refreshReference`; guarda `map` para a Task 14) → envio (`sendQueue`) → download (`downloadAll`; sem cursor e sem Wi-Fi e sem `mobile_data` → `skipped needs_wifi` depois do envio) → `purgeAccepted` → mapa em segundo plano da tela (não bloqueia o resultado). `features.home_visits = false` → envia nada (`feature_disabled`), download segue. Falhas: `revoked` → `wipeAll` e volta ao cadastro; `suspended` (`device_suspended` — revogação automática ou aparelho substituído, Q17 — e `invalid_token`) → `wipeTerritory`, marca `meta.suspended = 1` (tela "Acesso suspenso: procure o enfermeiro da equipe"), fila fica até um QR novo do mesmo host e usuário; `feature_disabled` (HTTP 403) → para, fila fica; `rate_limited`/`server`/`network` → para sem perder nada.
- Produces (`scheduler.ts`): `startScheduler(deps: { run: (t: RoundTrigger) => Promise<unknown>; appState; net; intervalMs?: number }): () => void` — dispara `foreground` ao voltar ao primeiro plano, `timer` a cada 15 min com rede; nunca com o app bloqueado.

- [ ] **Step 1: Escrever os testes**

```ts
test("upload before download", …);
test("second round while one runs is skipped", …);
test("device_revoked wipes everything including the outbox", …);
test("device_suspended and invalid_token wipe territory but keep the outbox", …);
test("403 feature_disabled keeps the queue", …);
test("5xx and network keep everything and stop", …);
test("initial load without wifi waits unless mobile data is allowed", …);
test("reference refreshed only when the version changes", …);
test("cityNowIso uses the city offset, not the device zone", () => {
  expect(cityNowIso(Date.parse("2026-11-13T12:00:00Z"), 0, -180)).toBe("2026-11-13T09:00:00-03:00");
});
test("skew of 7 min corrects occurred_at and asks for a warning", …);
test("day 24 no warn, day 25 warn, day 30 wipes territory but not the outbox", …);
test("validityBand: 2024-02-29 + 24 months = 2026-02-28", …);
test("accepted visits from yesterday are purged, today's stay", …);
test("scheduler: foreground and 15-min timer, never while locked", …);
```
- [ ] **Step 2–4:** `bin/test campo -w campo-mod24c src/sync/round.test.ts src/sync/clock.test.ts src/sync/retention.test.ts src/sync/scheduler.test.ts` FAIL → implementar → PASS.
- [ ] **Step 5: Commit** — `feat: run sync rounds with revocation, 30-day limit and clock correction`.

---

### Task 14: Mapa — download por intervalo e SHA-256

**Files:**
- Create: `src/map/download.ts`
- Test: `src/map/download.test.ts`

**Interfaces:**
- Consumes: `FieldClient.mapRequest` (com `If-Range` na continuação), sessão `map: { version, size_bytes, sha256 } | null` (o publicado, mesmo durante uma regeração no servidor, Q30), porta `FsPort` + `downloadResumable` (`expo-file-system/legacy` `createDownloadResumable`, que continua com `Range`), porta `sha256File(uri): Promise<string>` (método escolhido na Task 1).
- Produces: `syncMapFile(deps, manifest): Promise<"none" | "current" | "downloaded" | "resumed" | "needs_wifi" | "failed">` — `manifest = null` → `none` (tela: "Mapa ainda não baixado"; o arquivo antigo, se houver, fica); `sha256` igual ao de `map_file` → `current`; senão baixa para `map/<version>.pmtiles.part` (parcial de outro `sha256` é apagado antes), guarda `resume_data` em `map_file` a cada pausa, confere tamanho e SHA-256, renomeia para `map/<version>.pmtiles`, troca `map_file` na transação e só então apaga o arquivo anterior. SHA diferente → apaga o parcial, `failed`, mapa anterior intacto. Continuação sempre com `If-Range: "<manifest.sha256>"` (Q34): resposta `200` (o arquivo mudou no servidor) ou `416 range_not_satisfiable` → apaga o parcial e recomeça do zero na mesma rodada, uma vez. Sem Wi-Fi e sem `mobile_data` → `needs_wifi`. 404 `map_not_found` → `none`. `mapUri(row): string` = `"pmtiles://" + documentDirectory + "map/<version>.pmtiles"`.

- [ ] **Step 1: Escrever os testes**

```ts
test("current file is not downloaded again", …);
test("interrupted download resumes from resume_data with If-Range set to the session sha256 (Q34)", …);
test("200 or 416 on a resume discards the partial and starts over once", …);
test("partial of another sha is discarded before starting", …);
test("sha mismatch keeps the previous map and deletes the partial", …);
test("disk full error keeps the previous map", …);
test("swap is atomic: map_file changes only after the check", …);
test("no wifi and no mobile-data preference → needs_wifi", …);
test("map_not_found → none", …);
test("pmtiles uri is absolute file:// under the document directory", () => expect(mapUri(row)).toMatch(/^pmtiles:\/\/file:\/\/\/.+\/map\/2026-11-01\.pmtiles$/));
```
- [ ] **Step 2–4:** `bin/test campo -w campo-mod24c src/map/download.test.ts` FAIL → implementar → PASS.
- [ ] **Step 5: Commit** — `feat: download the city map with resume and SHA-256 check`.

---

### Task 15: Mapa — estilo do `map-style-v1.0.0` e camada dos domicílios

Depende da tag `map-style-v1.0.0` (Task 6 do plano do contracts-repo; Q11). Sem ela: pare e avise o coordenador.

**Files:**
- Create: `scripts/map-style-sync.mjs`, `assets/map/{style-light.json,style-dark.json,manifest.json,map-style.lock}`, `assets/map/fonts/…`, `assets/map/sprites/…` (cópia da tag), `plugins/withMapAssets.js` (se a Task 1 escolheu `asset://`)
- Create: `src/map/style.ts`, `src/map/householdLayer.ts`, `src/features/map/MapScreen.tsx`, `app/(main)/map.tsx`
- Test: `src/map/style.test.ts`, `src/map/mapStyle.lock.test.ts`, `src/map/householdLayer.test.ts`, `src/features/map/MapScreen.test.tsx`

**Interfaces:**
- Consumes: `map-style/dist` da tag `map-style-v1.0.0` (estilos `light` e `dark` com os marcadores `__PMTILES__`, `__GLYPHS__`, `__SPRITE__`; `manifest.json`; fontes; sprites `light` e `dark`); `mapUri` (Task 14), `households`, `who_to_visit`, `local_visits`, `captureGps`/`mapTapLocation` (Task 18 os usa; aqui só o evento de toque).
- Produces: `node scripts/map-style-sync.mjs map-style-v1.0.0 --contracts <clone>` — copia `map-style/dist` da tag para `assets/map/` e grava `assets/map/map-style.lock` (`{ "ref": "map-style-v1.0.0", "files": { "<caminho>": "<sha256>" } }`, formato do `tokens.lock`).
- Produces: `buildStyle(theme: "light" | "dark", p: { pmtilesUri: string; glyphs: string; sprite: string }): StyleSpecification` — sobre uma cópia do `assets/map/style-<tema>.json`, troca só os três marcadores: `__PMTILES__` por `pmtilesUri`, `__GLYPHS__` pela base das fontes e `__SPRITE__` pela base dos sprites (o esquema `asset://` ou `file://` escolhido na prova).
- Produces: `householdFeatures(rows, whoIds: Set<string>, visitedToday: Set<string>): FeatureCollection` — um ponto por domicílio com `location`; `properties.status` = `"visited_today"` > `"to_visit"` > `"registered"`; nenhuma propriedade com nome, CPF ou motivo (só `key` e `status`).
- Produces: `MapScreen` — fonte GeoJSON com `cluster`, cores por `status` (tokens), botão "Minha posição" (só ao tocar), modo "Marcar o ponto do imóvel" que devolve `{ latitude, longitude, zoom }` ao chamador, atribuição fixa "© OpenStreetMap" no canto, sem mapa → "Mapa ainda não baixado" e a lista continua. Nomes dos componentes: os do `@maplibre/maplibre-react-native` 11.x fixado (conferir na documentação da versão instalada).

- [ ] **Step 1: Escrever os testes**

```ts
test("style has no remote URL", () => { for (const t of ["light", "dark"] as const) expect(JSON.stringify(buildStyle(t, paths))).not.toMatch(/https?:\/\//); });
test("style source is the local pmtiles uri and no placeholder is left", …);
// mapStyle.lock.test.ts: cada arquivo de assets/map/ bate com o map-style.lock; nada fora do lock; ref = map-style-v1.0.0
test("labels are Portuguese", () => expect(JSON.stringify(styleLight)).toMatch(/name:pt/));
test("features carry no personal data", () => { const f = householdFeatures(rows, who, today); expect(Object.keys(f.features[0].properties).sort()).toEqual(["key", "status"]); });
test("visited today wins over to-visit", …);
test("MapScreen shows OSM attribution and the not-downloaded notice", …);
```
- [ ] **Step 2: Rodar e ver falhar** — `bin/test campo -w campo-mod24c src/map src/features/map` · Expected: FAIL.
- [ ] **Step 3: Implementar** `scripts/map-style-sync.mjs` e rode `node scripts/map-style-sync.mjs map-style-v1.0.0 --contracts /Users/eduardovrocha/Development/ioit.solutions/rota-saude/contracts` (sem `@protomaps/basemaps` no app: o estilo vem pronto, Q11); `style.ts`, `householdLayer.ts`, `MapScreen`.
- [ ] **Step 4: Rodar e ver passar** · Expected: PASS.
- [ ] **Step 5: Commit** — `feat: render the offline city map with the household layer`.

---

### Task 16: Casca das telas — bloqueio, Início, Quem visitar, Envio, Sobre, Acesso suspenso

**Files:**
- Create: `app/_layout.tsx` (substitui), `app/lock.tsx`, `app/suspended.tsx`, `app/(setup)/initial-load.tsx`, `app/(main)/{_layout,index,who,outbox,about}.tsx`, `app/(main)/attention/[seq].tsx`
- Create: `src/app/AppProvider.tsx`, `src/features/{lock,home,who,outbox,about,suspended,setup}/*.tsx`, `src/theme/tokens.ts`, `scripts/tokens.mjs`
- Test: `src/features/home/HomeScreen.test.tsx`, `src/features/lock/LockScreen.test.tsx`, `src/features/who/WhoScreen.test.tsx`, `src/features/outbox/OutboxScreen.test.tsx`, `src/features/about/AboutScreen.test.tsx`, `src/app/AppProvider.test.tsx`

**Interfaces:**
- Consumes: tudo das Tasks 5–15.
- Produces: `AppProvider` — abre o banco (erro de cifra → tela fatal), `recoverSending`, bloqueio (`shouldLock` com `AppState`), `startScheduler` depois do PIN, roteia: sem token → `(setup)/scan`; `campo.locked` → `lock` em modo "leia um QR novo no dashboard: Aparelhos de campo → Substituir"; `meta.suspended` → `suspended`.
- Produces: `useSyncState(): { running: boolean; lastOutcome: RoundOutcome | null; run(): void }`.
- Telas (frases exatas onde o spec/contrato fixa):
  - **Início:** microáreas; "A visitar: N"; "Pendentes de envio: N"; "Precisam de atenção: N"; "Conflitos: N"; "Última sincronização: <data hora>"; aviso dos 30 dias desde o 25º ("Sincronize até <data>: depois disso o território some do tablet"); aviso de relógio ("O relógio do tablet está X min diferente do servidor"); botão "Sincronizar". `home_visits` desligado → só a fila e "Sincronizar", com "Visita desligada pela secretaria".
  - **Quem visitar:** por microárea, cada pessoa com os motivos (24b §8: linha de cuidado com prática e prazo, `due_on: null` → "nunca feita"; cadastro a vencer pela `validityBand`; nunca visitado), idade (sem idade quando `null`), atalho "Registrar visita".
  - **Envio:** cada item com tipo, estado e a frase de `phraseFor(error)`; "Corrigir" abre o formulário do item; "Reenviar" volta `needs_attention` → `pending`; "Descartar" só para `needs_attention` e `blocked`, com confirmação "Este registro nunca chegou ao Rota Saúde e será apagado".
  - **Sobre:** versão, cidade (host), usuário, "© OpenStreetMap", preferência "Usar dados móveis para a carga e o mapa", "Sair" (com pendente: "Há N registros não enviados. Sair apaga tudo deste tablet, inclusive eles." e confirmação dupla).
  - **Carga inicial:** progresso por página, aviso de Wi-Fi, depois o mapa.
  - Tema claro/escuro pelos tokens `govbr`/`govbr-escuro` (`design-tokens` tag `tokens-v1.1.0`); alvos de toque ≥ 48 dp, texto base ≥ 18 sp; retrato e paisagem.

- [ ] **Step 1: Escrever os testes** (RNTL; serviços falsos pelo `AppServices`)

```tsx
test("cold start asks for the PIN", …);
test("after 5 min in background the app locks", …);
test("locked by 5 errors shows the QR replacement message", …);
test("home shows counters, last sync and the day-25 warning", …);
test("home with home_visits off shows only queue and Sync", …);
test("who-to-visit shows 'nunca feita' for null due_on and no age when null", …);
test("outbox shows the phrase of needs_attention and allows resend", …);
test("discard asks for confirmation and removes dependents", …);
test("sign out with pending items warns with the count", …);
test("encryption failure shows the fatal screen and never opens the main stack", …);
```
- [ ] **Step 2–4:** `bin/test campo -w campo-mod24c src/features/home src/features/lock src/features/who src/features/outbox src/features/about src/app` FAIL → implementar → PASS.
- [ ] **Step 5: Commit** — `feat: add the app shell with lock, home, who to visit, outbox and about`.

---

### Task 17: Regras de tela portadas do dashboard

Porte (cópia adaptada, sem mudar regra) de três arquivos do dashboard, para os formulários do app recusarem o que o api recusaria (Q28: o app porta os arquivos de regra do dashboard, com o commit de origem anotado e os mesmos testes; divergência futura entre os dois vira card). Origem: `apps/dashboard/src/lib/{territoryDictionaries.ts, homeVisits.ts, collectiveActivities.ts}` da main com o 24b mergeado (hoje na branch `feat/mod-24b-home-visits`, `.worktrees/dashboard-mod24b`, commit 88747fe); anote o commit no cabeçalho de cada porte.

**Files:**
- Create: `src/rules/registry.ts`, `src/rules/homeVisit.ts`, `src/rules/collectiveActivity.ts`, `src/rules/support.ts` (`isValidCpf`, `isValidCns`, larguras de endereço — copiados de `attendance.ts`, `professionals.ts`, `demographics.ts`)
- Test: `src/rules/registry.test.ts`, `src/rules/homeVisit.test.ts`, `src/rules/collectiveActivity.test.ts` (porte dos testes do dashboard: `territoryDictionaries.test.ts`, `homeVisits.test.ts`, `collectiveActivities.test.ts`)

**Interfaces:**
- Consumes: `RegistrationDictionaries`, `HomeVisitDomains`, `CollectiveDomains` (Task 12).
- Produces: `createRegistryRules(dict: RegistrationDictionaries)` → `{ dictLabel, isValidLediName, sectionProblem, addressProblem, allowsLivingConditions, allowsFamilies, livingFieldAllowed, householdProblem, familyProblem, canBePregnant, individualRulesProblem, newPatientProblem, registrationProblem }` — mesmas assinaturas do dashboard, com os dicionários vindos da referência em vez do `.generated.ts`.
- Produces: `homeVisit.ts` — `offeredReasons`, `allowedMeasurements`, `pruneVisitDraft`, `familyVisitNotice`, `measurementsProblems`, `visitProblems`, `visitBody`, `quickPatientProblems`, `FAMILY_NOTICE`, `NO_FAMILY_NOTICE`; `collectiveActivity.ts` — `fieldRule`, `offeredHealthTopics`, `offeredPractices`, `needsProcedure`, `smokingAllowed`, `activityProblems`, `activityWarnings`, `activityBody`. `Problem = { path: string; message: string }`.
- O que não vem: chamadas de API, chaves de cache, textos de folha do dashboard (`DAY_CHANGED` …).

- [ ] **Step 1: Portar os testes** do dashboard trocando os imports e alimentando os dicionários a partir do exemplo `reference` do contrato. Expected em cada caso: mesmo `path` e mesma mensagem que no dashboard.
- [ ] **Step 2: Rodar e ver falhar** — `bin/test campo -w campo-mod24c src/rules` · Expected: FAIL.
- [ ] **Step 3: Portar o código** (só imports, dicionários por parâmetro e tipos do contrato; nenhuma regra nova; `visitBody` emite o corpo do item `home_visit` sem `client_uuid` e sem `shift`).
- [ ] **Step 4: Rodar e ver passar** · Expected: PASS.
- [ ] **Step 5: Commit** — `feat: port the dashboard screen rules for registry, visit and activity`.

---

### Task 18: Domicílio — detalhe, cadastro domiciliar e ponto do imóvel

**Files:**
- Create: `src/location/capture.ts`, `src/features/households/{HouseholdScreen,HouseholdForm,HouseholdList}.tsx`, `src/features/households/save.ts`, `app/(main)/households/[key].tsx`, `households/new.tsx`, `households/[key]/edit.tsx`
- Test: `src/location/capture.test.ts`, `src/features/households/save.test.ts`, `src/features/households/HouseholdForm.test.tsx`

**Interfaces:**
- Consumes: `createRegistryRules`, `coalesceOrEnqueue`, `householdEditable`, `MapScreen` (modo marcar), `Reference.neighborhoods`.
- Produces (`capture.ts`): `GPS_TIMEOUT_MS = 15_000`, `MAX_ACCURACY_M = 50`; `roundCoord(x: number): number` (8 casas); `roundAccuracy(m: number): number` (1 casa, 0..10000); `captureGps(loc: LocationPort): Promise<Location | null>` — permissão concedida e precisa (`android.accuracy === "fine"`), `getCurrentPositionAsync({ accuracy: High })` com 15 s; `accuracy` nula ou > 50 → `null`; `source: "gps"`; `mapTapLocation(lat, lng, zoom): Location` (`source: "map"`, `accuracy_m` = `roundAccuracy(156543.03392 * cos(lat) / 2^zoom * 24)`, a resolução do toque; Q3 — opcional com `map` no contrato, o app sempre manda).
- Produces (`save.ts`): `saveHousehold(db, input: { key?: string; editable: HouseholdEditable }): Promise<{ key: string; clientUuid: string }>` — novo: linha `key = client_uuid`, item `household` sem `household_id`; existente: item `household` com `household_id` + `targetId` (a `key` da linha) + `editBase` (a `base_*` da linha no momento da edição, Q41), coalescido se já houver um pendente; recusa se a linha tiver item em `conflict` ("Resolva o conflito deste domicílio antes de editar").
- Tela: endereço (tipo de logradouro e UF da referência, bairro da lista do módulo 11 ou texto), tipo de imóvel, condições de moradia só quando `allowsLivingConditions`, telefone, recusa; "Ponto do imóvel": "Usar o GPS aqui na frente" (sem precisão → "Sem posição precisa. Marque no mapa.") ou "Marcar no mapa"; famílias, moradores e visitas locais no detalhe. Sem `territory_registry` ou CBO ≠ 515105 → só leitura (decisão 22).

- [ ] **Step 1: Escrever os testes**

```ts
test("roundCoord keeps at most 8 decimals in JSON", () => { for (const x of randomCoords(10_000)) expect(JSON.stringify(roundCoord(x))).toMatch(/^-?\d+(\.\d{1,8})?$/); });
test("coarse permission → null", …);
test("accuracy 51 m → null, 50 m → location", …);
test("timeout after 15 s → null", …);
test("GPS location always carries accuracy_m; map tap location has source map and bounded accuracy_m (Q3)", …);
test("household body with location validates against upload-request-v1.json, gps and map", …);
test("new household enqueues a household item with location and no household_id", …);
test("edit enqueues with household_id, target_id and the row base as edit base", …);
test("exit (move, death) and identity edits show 'Faça pelo dashboard' (Q47)", …);
test("second edit while pending is coalesced", …);
test("form hides living conditions for non-residential types and blocks on householdProblem", …);
test("non-ACS sees the household read-only", …);
```
- [ ] **Step 2–4:** `bin/test campo -w campo-mod24c src/location src/features/households` FAIL → implementar → PASS.
- [ ] **Step 5: Commit** — `feat: register and edit households offline with their location`.

---

### Task 19: Família e cadastro individual com duplicado no tablet

**Files:**
- Create: `src/features/families/{FamilyForm,save}.ts(x)`, `src/features/registrations/{RegistrationForm,save,localDuplicates}.ts(x)`, `app/(main)/families/new.tsx`, `families/[key]/edit.tsx`, `registrations/new.tsx`, `registrations/[key]/edit.tsx`
- Test: `src/features/registrations/localDuplicates.test.ts`, `src/features/registrations/save.test.ts`, `src/features/families/save.test.ts`, `src/features/registrations/RegistrationForm.test.tsx`

**Interfaces:**
- Consumes: `createRegistryRules`, `enqueue`/`coalesceOrEnqueue`, `registrationEditable`, `familyEditable`, `normalizeText`.
- Produces (`localDuplicates.ts`): `findSimilar(db, q: { full_name: string; birth_date: string; mother_name?: string | null }): Promise<{ key: string; display_name: string; birth_date: string | null }[]>` (nome normalizado igual ou com distância ≤ 2 **e** mesmo nascimento; mãe, se as duas existirem, igual normalizada) — avisa; `findDocumentClash(db, q: { cpf?: string; cns?: string }): Promise<string | null>` — só entre os `new_patient` dos itens `individual_registration` e `patient` que o tablet criou e ainda estão na fila — bloqueia ("Este CPF já foi usado em outro cadastro deste tablet").
- Produces (`registrations/save.ts`): pessoa nova = **um** item `individual_registration` com `new_patient` (com nome social e `duplicates_reviewed`), `household_id` ou `household_client_uuid` (dep `cite`), `family_id` ou `family_client_uuid` (dep `cite`); o `client_uuid` desse item é o que a família, a visita e a atividade citam como `patient_client_uuid` (Q26: o livro do api resolve o paciente criado por ele). O item `patient` fica só para o registro rápido de pessoa sem cadastro na visita e na atividade (24b). Existente: item com `registration_id`, `targetId`, `editBase`.
- Produces (`families/save.ts`): `saveFamily(db, input)` — família nova: item `family` com `household_id`/`household_client_uuid` (dep `version` se o domicílio nasceu no tablet e ainda não foi aceito; senão `targetId` = domicílio, sem `editBase` de família), `responsible: { patient_id | patient_client_uuid }` (forma aninhada, Q1). O responsável é escolhido entre os moradores do domicílio — baixados (`patient_id`) ou criados no tablet (`patient_client_uuid` = `client_uuid` do `individual_registration` dele, dep `cite`, que já está antes na fila); o api liga o cadastro ativo dele, se houver (Q25). Existente: `family_id`, `targetId` = domicílio, `editBase` = `familyEditable(base_body, family_id)`. A troca de responsável não existe no app.
- Resultado `duplicate_suspected` (Task 22 mostra): candidatos do resultado, marcando os que existem localmente; "É esta pessoa" troca, no próprio `individual_registration`, o `new_patient` pelo `patient_id` escolhido e reescreve os dependentes que citavam o `client_uuid` dele (`patient_client_uuid` → `patient_id`, dep `cite` removido), porque o api só resolve a citação quando o cadastro criou a pessoa (Q26); no item `patient` do registro rápido, troca o item pelo `patient_id` nos dependentes e o apaga; "É outra pessoa" reenvia com `duplicates_reviewed: true`. `already_registered` com `id` → oferece abrir o cadastro existente e descartar o item.

- [ ] **Step 1: Escrever os testes**

```ts
test("similar name + same birth warns, different birth does not", …);
test("CPF clash only among tablet-created pending people", …);
test("new person enqueues one individual_registration with new_patient and social name (Q26)", …);
test("new family on a tablet-born household waits for its version", …);
test("new family offers as responsible only residents of the household, downloaded or created on the tablet", …);
test("family body cites the responsible's registration client_uuid in responsible.patient_client_uuid, with a cite dependency", …);
test("'É esta pessoa' rewires dependents from patient_client_uuid to patient_id", …);
test("registration on a new family uses family_client_uuid", …);
test("upload of registration(new_patient) → family → visit citing it validates against upload-request-v1.json", …);
test("RegistrationForm shows pregnant only when canBePregnant", …);
```
- [ ] **Step 2–4:** `bin/test campo -w campo-mod24c src/features/registrations src/features/families` FAIL → implementar → PASS.
- [ ] **Step 5: Commit** — `feat: register families and residents offline with the local duplicate check`.

---

### Task 20: Visita com GPS e "minhas visitas de hoje"

**Files:**
- Create: `src/features/visits/{VisitForm,TodayScreen,save}.ts(x)`, `app/(main)/visits/new.tsx`, `visits/today.tsx`
- Test: `src/features/visits/save.test.ts`, `src/features/visits/VisitForm.test.tsx`, `src/features/visits/TodayScreen.test.tsx`

**Interfaces:**
- Consumes: `homeVisit.ts` (Task 17), `captureGps`, `cityNowIso`, `cityToday`, sessão (`teams`, `min_date`, `out_of_area_allowed`, `features`), `Reference.homeVisitDomains`.
- Produces (`save.ts`): `saveVisit(db, draft, ctx): Promise<string>` — corpo de `visitBody` + `team_id` + `occurred_at = cityNowIso(...)` (ou o horário escolhido, no dia da cidade) + `location` só quando `captureGps` devolveu; `household_client_uuid`/`patient_client_uuid` com dep `cite` quando o alvo nasceu no tablet (a pessoa criada no tablet é citada pelo `client_uuid` do `individual_registration` com `new_patient`, Q26; a do registro rápido, pelo do item `patient`); `family_id`/`family_client_uuid` para `family_responsible`; grava `local_visits` (`pending`, `city_day`, resumo sem observação).
- Produces: `withdrawVisit(db, clientUuid, reason: string)` — visita ainda não enviada → `removeUnsent` (com confirmação "Esta visita ainda não foi enviada e será apagada"); aceita → item `home_visit_withdrawal` `{ visit_client_uuid, reason }` (10–500), `local_visits.state = withdrawn` só depois do `accepted`; `day_closed` → frase "O dia já foi fechado no Rota Saúde; a visita não pode mais ser retirada".
- Tela: origem (domicílio, família, morador ou imóvel sem cadastro), motivos por grupo e medidas com as regras portadas, aviso `FAMILY_NOTICE`/`NO_FAMILY_NOTICE`, data entre `min_date` e hoje da cidade; ao salvar, "Pegando a posição…" (até 15 s) e grava com ou sem `location` sem perguntar. "Minhas visitas de hoje": `local_visits` do dia da cidade, com o estado de envio e "Retirar".

- [ ] **Step 1: Escrever os testes**

```ts
test("visit saved with GPS has location source gps", …);
test("visit without precise position saves without location, never a bad one", …);
test("occurred_at has the city offset and is corrected by the known skew", …);
test("date before min_date is refused on screen", …);
test("withdraw before sending deletes the pending item after confirmation", …);
test("withdraw after acceptance enqueues home_visit_withdrawal with visit_client_uuid", …);
test("non-ACS can register visits only in assigned micro areas, no out-of-area", …);
test("visit body validates against upload-request-v1.json", …);
test("visit to a resident created on the tablet cites the registration client_uuid (Q26)", …);
test("visit location is gps with accuracy_m, or absent — never null (Q3)", …);
```
- [ ] **Step 2–4:** `bin/test campo -w campo-mod24c src/features/visits` FAIL → implementar → PASS.
- [ ] **Step 5: Commit** — `feat: record home visits offline with GPS and today's sheet`.

---

### Task 21: Atividade coletiva

**Files:**
- Create: `src/features/activities/{ActivityForm,save}.ts(x)`, `app/(main)/activities/new.tsx`
- Test: `src/features/activities/save.test.ts`, `src/features/activities/ActivityForm.test.tsx`

**Interfaces:**
- Consumes: `collectiveActivity.ts` (Task 17), `healthUnitOptions`/`professionalOptions` (Task 12: `has_cnes`, `has_cns`, recorte por `team_id`; Q6, Q7), `Reference.collectiveDomains`, `registrations` locais.
- Produces: `saveActivity(db, draft, ctx): Promise<string>` — corpo de `activityBody` + `team_id`; local: unidade da referência (`place_health_unit_id`; sem CNES, a regra portada recusa como no dashboard) ou texto; profissionais da equipe da atividade (sem CNS, a regra portada recusa: "este profissional não tem CNS no cadastro"); participantes: morador baixado (`patient_id`), pessoa criada no tablet (`patient_client_uuid` = `client_uuid` do cadastro ou do item `patient`, dep `cite`; Q26) ou registro rápido (`new_patient` + `duplicates_reviewed` no participante); grava `local_activities`.
- Tela: oculta com `features.collective_activities = false`; tipos e campos por `fieldRule`; busca de participante por nome entre os moradores baixados; avisos de `activityWarnings` (sem bloquear, Q74); resultado `duplicate_participant` com `confirmable` mostra "É outra pessoa" só quando `true`. Oferecida a todo aparelho com `collective_activities` ligado; o api decide (`cbo_not_allowed`; Q6).

- [ ] **Step 1: Escrever os testes**

```ts
test("type 5 without participants is blocked by activityProblems", …);
test("participant created on the tablet is cited by patient_client_uuid", …);
test("health unit without CNES and professional without CNS are refused on screen like the dashboard (Q6)", …);
test("hidden when collective_activities is off", …);
test("activity body validates against upload-request-v1.json", …);
```
- [ ] **Step 2–4:** `bin/test campo -w campo-mod24c src/features/activities` FAIL → implementar → PASS.
- [ ] **Step 5: Commit** — `feat: record collective activities offline`.

---

### Task 22: Conflitos e itens que precisam de atenção

**Files:**
- Create: `src/features/conflicts/ConflictScreen.tsx`, `src/features/attention/AttentionScreen.tsx`, `app/(main)/conflicts/[seq].tsx` (e a rota de atenção da Task 16 passa a usar esta tela)
- Test: `src/features/conflicts/ConflictScreen.test.tsx`, `src/features/attention/AttentionScreen.test.tsx`

**Interfaces:**
- Consumes: `resolveConflict` (Task 10), `OutboxItem`, `createRegistryRules().dictLabel` e rótulos de seção da referência, `phraseFor`.
- Produces: **Conflitos** — um bloco por `ConflictField`: rótulo do campo (da referência; `address` → "Endereço", `location` → "Ponto do imóvel"), valor "No posto" × "No tablet" (códigos pelos rótulos), escolha obrigatória de cada um; "Enviar" só com tudo escolhido → `resolveConflict` → item `pending`. **Atenção** — a frase do código, o campo do `path` em destaque, "Corrigir" (abre o formulário do item com o corpo da fila), "Reenviar", e os casos `duplicate_suspected` (candidatos; "É esta pessoa" / "É outra pessoa", regra da Task 19), `already_registered` ("Abrir o cadastro existente"), `duplicate_participant` (`confirmable`).

- [ ] **Step 1: Escrever os testes**

```tsx
test("shows station and tablet values with dictionary labels", …);
test("send stays disabled until every field is chosen", …);
test("choosing resolves and returns the item to pending", …);
test("duplicate_suspected: 'É esta pessoa' replaces new_patient by patient_id and rewires every dependent to patient_id (Q26)", …);
test("duplicate_suspected: 'É outra pessoa' resends with duplicates_reviewed true", …);
```
- [ ] **Step 2–4:** `bin/test campo -w campo-mod24c src/features/conflicts src/features/attention` FAIL → implementar → PASS.
- [ ] **Step 5: Commit** — `feat: resolve conflicts and refused items on the tablet`.

---

### Task 23: Endurecimento do Android (tela, backup, rede, permissões, logs)

**Files:**
- Create: `plugins/withFieldHardening.js`, `scripts/check-apk.mjs`
- Modify: `app.config.ts` (plugin), `eslint.config.js` (`no-console`, proibição de `fetch` fora de `src/sync/http.ts` e `src/map/download.ts`)
- Test: `plugins/withFieldHardening.test.js`, `src/security/noLeak.test.ts`

**Interfaces:**
- Produces: `withFieldHardening(config, { e2e: boolean; devCleartextHosts: string[] })` — em `MainActivity.kt` acrescenta `window.setFlags(WindowManager.LayoutParams.FLAG_SECURE, WindowManager.LayoutParams.FLAG_SECURE)` antes do `super.onCreate`; no manifesto `android:allowBackup="false"`, `android:fullBackupContent="false"`, `android:dataExtractionRules="@xml/campo_no_backup"` (exclui tudo de `cloud-backup` e `device-transfer`); `network_security_config` com `cleartextTrafficPermitted="false"` e, só com `e2e`/dev, `domain-config` cleartext só para `10.0.2.2` (Q48).
- Produces: `node scripts/check-apk.mjs <apk>` — `apkanalyzer manifest permissions` = exatamente as 5 permissões + `br.rotasaude.campo.DYNAMIC_RECEIVER_NOT_EXPORTED_PERMISSION`; `apkanalyzer manifest print` com `allowBackup="false"` e `debuggable` ausente; nenhuma string `10.0.2.2`, "Colar código" nem "sem chave:" no APK de produção (`unzip -p … index.android.bundle | grep`); a lista de sufixos embutida é a do build (`.localhost`/`.test` nunca num APK de produção; Q44); exit ≠ 0 com a lista do que falhou.

- [ ] **Step 1: Escrever os testes**

```js
test("injects FLAG_SECURE before super.onCreate", () => expect(applyMainActivity(SAMPLE_KT)).toMatch(/FLAG_SECURE[\s\S]*super\.onCreate/));
test("production network config forbids cleartext everywhere", …);
test("e2e allows cleartext only to 10.0.2.2 (Q48)", …);
```
```ts
// noLeak.test.ts — roda uma rodada completa com dados de exemplo e espia src/log e console:
test("no log line contains names, CPF, CNS, coordinates or the token", () => {
  for (const line of captured) expect(line).not.toMatch(/rsf_|\b\d{11}\b|\b\d{15}\b|-25\.\d{3,}|Rosângela|full_name|latitude/);
});
```
- [ ] **Step 2–4:** `bin/test campo -w campo-mod24c plugins src/security/noLeak.test.ts` FAIL → implementar → PASS; depois `npx expo prebuild --platform android --clean && (cd android && ./gradlew assembleRelease)` e `node scripts/check-apk.mjs android/app/build/outputs/apk/release/app-release.apk` · Expected: exit 0.
- [ ] **Step 5: Commit** — `feat: harden the Android build against capture, backup and cleartext`.

---

### Task 24: Maestro — servidor falso, fluxos e CI

**Files:**
- Create: `scripts/fake-field-server/{server.mjs,state.mjs,qr.mjs,fixtures/sample.pmtiles}` (`qr.mjs <host>` imprime um `qr_payload` com token falso), `scripts/fake-field-server/server.test.ts` (Jest, `@jest-environment node`)
- Create: `maestro/{pairing,initial-load,offline-visit,household-family,conflict-merge,conflict-choice,revocation-manual,revocation-suspended,map,encryption}.yaml`, `maestro/README.md`
- Create: `src/features/diagnostics/DiagnosticsScreen.tsx`, `app/(main)/diagnostics.tsx` (só com `extra.e2e`)
- Modify: `.github/workflows/ci.yml` (job `maestro`)

**Interfaces:**
- Produces: `node scripts/fake-field-server/server.mjs --port 8787` — rotas `GET /field/v1/session` (`session-*.json`), `GET /field/v1/reference`, `GET /field/v1/changes` (primeira e última página dos exemplos), `POST /field/v1/uploads` (valida o pedido contra `upload-request-v1.json`; responde `accepted` com `id` = UUID v5 do `client_uuid` e, para `household`/`family`/`individual_registration`, `version` = 1 + a anterior por alvo; resolve `patient_client_uuid` contra `patient` e `individual_registration` com `new_patient`, como o api, Q26), `GET /field/v1/map` (`sample.pmtiles`, `ETag`, `Range` 206, `If-Range`, 416 `range_not_satisfiable`; Q34); controle por `POST /__script` (`{ next_upload: "stale_version" | "rejected:<code>", next_status: 401, error: "device_revoked" | "device_suspended" }`; `device_suspended` simula também o aparelho substituído, Q17) só no servidor falso. Toda resposta é validada contra o schema antes de sair.
- Produces: `DiagnosticsScreen` (e2e; a rota redireciona para o Início quando `extra.e2e` é falso) — mostra `cipher_version` e o resultado de abrir uma cópia do banco sem chave ("sem chave: recusado").
- Build e2e: `EXPO_PUBLIC_E2E=1 EXPO_PUBLIC_DEV_ORIGIN=http://10.0.2.2:8787 CAMPO_ALLOWED_HOST_SUFFIXES=.localhost,.test npx expo prebuild --clean && cd android && ./gradlew assembleRelease` (APK release de variante `e2e`, assinado com chave descartável na CI; Q48).

- [ ] **Step 1: Teste do servidor falso** (`scripts/fake-field-server/server.test.ts`, no `bin/test campo`): cada rota responde conforme o schema; `Range: bytes=0-99` → 206 com 100 bytes; `If-Range` de outro SHA → 200; upload inválido → 422 `invalid_request`. Run: `bin/test campo -w campo-mod24c scripts` · FAIL → implementar → PASS.

- [ ] **Step 2: Fluxos Maestro** — cada `.yaml` com `appId: br.rotasaude.campo`, `clearState` no início, `inputText` do payload no campo "Colar código" (`${QR_PAYLOAD}`), PIN `123456` duas vezes:
  - `pairing`: cadastra, cria PIN, vê "Início".
  - `initial-load`: carga com progresso; contagem "A visitar" > 0.
  - `offline-visit`: `setAirplaneMode: enabled`; visita a um morador; "Pendentes de envio: 1"; `setAirplaneMode: disabled`; "Sincronizar"; "Pendentes de envio: 0".
  - `household-family`: domicílio novo com ponto no mapa, morador novo, família nova com ele responsável; sincroniza; tudo aceito.
  - `conflict-merge`: script `stale_version` sem campo cruzado → volta sozinho e é aceito.
  - `conflict-choice`: script `stale_version` com campo cruzado → "Conflitos: 1" → escolhe "No tablet" → aceito.
  - `revocation-manual`: script `401 device_revoked` → volta ao cadastro; fila vazia.
  - `revocation-suspended`: script `401 device_suspended` → "Acesso suspenso: procure o enfermeiro da equipe"; parear de novo com o mesmo usuário → fila enviada.
  - `map`: abre o mapa offline; vê "© OpenStreetMap" e o ponto do domicílio.
  - `encryption`: abre Diagnóstico; vê "cifra:" seguido de versão e "sem chave: recusado".

- [ ] **Step 3: Rodar local contra o servidor falso**

```bash
node scripts/fake-field-server/server.mjs --port 8787 &
adb install -r android/app/build/outputs/apk/release/app-release.apk
QR=$(node scripts/fake-field-server/qr.mjs curitiba.localhost); maestro test -e QR_PAYLOAD="$QR" maestro/
```
Expected: todos os fluxos `PASSED`.

- [ ] **Step 4: Rodar local contra o api de dev** (prova, não CI): build e2e com `EXPO_PUBLIC_DEV_ORIGIN=http://10.0.2.2:3030` (porta do api no compose); confira antes `curl -s -H 'Host: curitiba.localhost' -H "Authorization: Bearer <token>" http://localhost:3030/field/v1/session`. O `qr_payload` vem do dashboard de dev (Aparelhos de campo → cadastrar, com o usuário ACS da semente; o payload está na resposta 201 nas ferramentas de rede, só em dev) e vai por `-e QR_PAYLOAD=…`, nunca para arquivo do repositório. Rode `pairing`, `initial-load`, `offline-visit`, `household-family`; conflito e revogação na prova da Task 26. Expected: `PASSED`.

- [ ] **Step 5: Job da CI** — `maestro` em `ubuntu-latest` com KVM: `actions/setup-node` (22), `actions/setup-java` (a versão da Task 1), `npm ci`, keystore descartável (`keytool -genkeypair … -keystore $RUNNER_TEMP/e2e.jks`), build e2e com `EXPO_PUBLIC_DEV_ORIGIN=http://10.0.2.2:8787`, `reactivecircus/android-emulator-runner@v2` (`api-level: 34`, `arch: x86_64`, `target: google_apis`) rodando o servidor falso, `adb install` e `maestro test maestro/` (Maestro CLI na versão fixada). Imagens base, se houver container, por `mirror.gcr.io`. Expected: job verde no push.

- [ ] **Step 6: Commit** — `test: add Maestro flows against a fake field sync server`.

---

### Task 25: Build release assinado e runbook de distribuição

**Files:**
- Create: `plugins/withReleaseSigning.js`, `plugins/withReleaseSigning.test.js`, `docs/build-release.md` (no repo `campo`)
- Create (docs repo): `operacao/rollout-app-de-campo.md`
- Modify: `app.config.ts` (plugin; `versionCode`/`version` lidos de `package.json`)

**Interfaces:**
- Produces: `withReleaseSigning` — no `android/app/build.gradle` gerado, `signingConfigs.release` lendo `CAMPO_UPLOAD_STORE_FILE`, `CAMPO_UPLOAD_STORE_PASSWORD`, `CAMPO_UPLOAD_KEY_ALIAS`, `CAMPO_UPLOAD_KEY_PASSWORD` de `~/.gradle/gradle.properties`; sem as propriedades, o `release` falha com "chave de assinatura ausente" (nunca cai na chave de debug). A chave fica em `~/.rotasaude/campo-upload.jks`, fora do repositório e do `android/`.

- [ ] **Step 1: Teste do plugin** — `bin/test campo -w campo-mod24c plugins/withReleaseSigning.test.js`: injeta `signingConfigs.release` e `buildTypes.release.signingConfig signingConfigs.release`; sem propriedades gera o `throw new GradleException("chave de assinatura ausente")`. FAIL → implementar → PASS.
- [ ] **Step 2: Gerar a chave (o usuário guarda a senha)** — `keytool -genkeypair -v -keystore ~/.rotasaude/campo-upload.jks -alias campo-upload -keyalg RSA -keysize 4096 -validity 10000`; senha e alias em `~/.gradle/gradle.properties` (o executor não vê nem escreve a senha: o usuário digita).
- [ ] **Step 3: Build**

```bash
CAMPO_ALLOWED_HOST_SUFFIXES=<sufixos do runbook> npx expo prebuild --platform android --clean
cd android && ./gradlew bundleRelease assembleRelease
node ../scripts/check-apk.mjs app/build/outputs/apk/release/app-release.apk
$ANDROID_HOME/build-tools/35.0.0/apksigner verify --print-certs app/build/outputs/apk/release/app-release.apk
```
Expected: AAB e APK gerados; `check-apk` exit 0; certificado `CN` da chave de upload.

- [ ] **Step 4: Runbook `operacao/rollout-app-de-campo.md`** (spec §7): conta de desenvolvedor Google verificada (obrigatória no Brasil desde 2026-09-30); app privado no Google Play gerenciado de cada cidade (AAB); tablets por Android Enterprise, totalmente gerenciados, permissão de localização precisa (conceder na primeira vez se o DPC não conceder); gerar o mapa da cidade no maintenance antes de cadastrar aparelhos (e o espaço em disco do worker para a cópia local da base do Brasil, Q32); ligar `home_visits` antes; os sufixos de host de staging e de produção para `CAMPO_ALLOWED_HOST_SUFFIXES` — **gate do build de release**: sem eles o build não sai (Q44); cadastro dos aparelhos pelo dashboard (QR, step-up); substituir aparelho depois de bloqueio do PIN ou troca de tablet (o antigo recebe `device_suspended` e guarda a fila até ler o QR novo, Q17); aparelho revogado automaticamente → "Cadastrar novo" no dashboard e QR no mesmo tablet (Q16); guardar a chave de upload fora do repositório (cópia cifrada em dois lugares); `prebuild --clean` apaga `android/` e a chave nunca mora lá; build local (Gradle), nunca a nuvem do Expo; gates de go-live (spec §9).
- [ ] **Step 5: Commits** — no `campo`: `plugins/withReleaseSigning.js`, o teste, `docs/build-release.md`, `app.config.ts` · `build: sign release builds with an upload key kept outside the repository`; no docs: `operacao/rollout-app-de-campo.md` · `docs: add the field app rollout runbook`.

---

### Task 26: Prova integrada no emulador e fechamento do campo

**Files:** nenhum código novo; relatório ao coordenador.

- [ ] **Step 1: Suíte do app** — `bin/test campo -w campo-mod24c` · Expected: `resultado: ok`.
- [ ] **Step 2: Prova no emulador contra o api de dev** (contrato §11): QR do dashboard de dev, carga, modo avião, visita, família nova, sincronização, conflito com e sem escolha (editar o mesmo domicílio no dashboard enquanto o tablet está em modo avião), revogação manual (dashboard → revogar → `device_revoked` apaga tudo) e automática (encerrar o vínculo do ACS na semente → `device_suspended`; substituir no dashboard → fila enviada), mapa offline com os domicílios. Capturas por `adb exec-out screencap` não funcionam com `FLAG_SECURE` (esperado: tela preta) — use o build e2e sem a flag só para a evidência ou descreva.
- [ ] **Step 3: Revisão** — `superpowers:requesting-code-review` sobre a branch inteira; correções com `bin/test campo` de escopo fechado.
- [ ] **Step 4: Fechamento** — PR `feat/mod-24c-field-app` → `main` no `rotasaude/campo` (descrição com `pr-description`), CI verde (test + maestro), merge com autorização; cards: F-24.22, F-24.23 e F-24.25 (só campo e docs) em Done; F-24.18, F-24.19, F-24.20, F-24.21 e F-24.24 só quando todas as superfícies da Q10 estiverem entregues (o último app move); card novo no board #2 para a saída do cadastro e a edição de identidade pelo app (Q47, com o §12 do contrato); relato ao coordenador com qualquer divergência nova (Q50+).

---

## Decisões aplicadas (§13 do contrato)

| Código | Q | Onde o plano aplica |
|---|---|---|
| QC-1 | Q1 | Task 19: `responsible: { patient_id \| patient_client_uuid }` aninhado |
| QC-2 | Q41 | Esquema local (`edit_base_version`, `edit_base_body`, `target_id`, `client_ids`), Tasks 9, 11, 18: o envio leva sempre a base da edição; o download nunca muda a base do item |
| QC-3 | Q42 | Task 10: a mescla compara as projeções na forma do envio (sem `families` nem `patient`) |
| QC-4 | Q43 | Task 10: família campo a campo pelo `id`; família nova só troca a versão; família sumida → `family_not_found` |
| QC-5 | Q26 | Tasks 19–22, 24: pessoa nova = um `individual_registration` com `new_patient` (com nome social), citado pelo `client_uuid`; "É esta pessoa" reescreve os dependentes |
| QC-6 | Q27 | Task 12: domínios só como listas `[{ code, label }]`; objeto → `ReferenceShapeError` |
| QC-7 | Q28 | Task 17: regras portadas do dashboard, com o commit de origem e os mesmos testes |
| QC-8 | Q8 | Tasks 4 e 7: os códigos de borda e do mapa vêm de `errors.http`; todos com frase |
| QC-9 | Q3 | Global Constraints, Tasks 4, 18, 20: `accuracy_m` obrigatório com `gps`; no mapa, a resolução do toque |
| QC-10 | Q44 | Global Constraints, Tasks 2, 8, 23, 25: `CAMPO_ALLOWED_HOST_SUFFIXES`, só `https`; release sem a lista não compila; sufixos no runbook |
| QC-11 | Q6, Q45 | Tasks 12 e 21: `has_cns`, `cnes` anulável, atividade oferecida a todo aparelho com `collective_activities`; Task 9: `household_changed`/`internal_error` transitórios por 3 rodadas |
| QC-12 | Q46 | Task 8: mesmo host e mesmo `user.id` |
| QC-13 | Q47 | Tasks 10, 18, 26: saída do cadastro e identidade fora do app ("Faça pelo dashboard"); card no fechamento |
| QC-14 | Q48 | Tasks 8, 23, 24: APK release de variante `e2e`; cleartext só para `10.0.2.2`; `check-apk` prova o de produção |
| — | Q5 | Esquema local (`needs_reload`), Tasks 9 e 11: `duplicate` sem `version` marca a linha para recarga, sem mescla |
| — | Q11, Q12 | Task 1 (versões de `@maplibre/maplibre-react-native`, `@protomaps/basemaps` e `pmtiles` no contrato) e Task 15 (cópia do `map-style-v1.0.0` por lock) |
| — | Q16, Q17 | Tasks 7, 8, 13, 25: o QR de "Substituir" ou "Cadastrar novo" do mesmo host e usuário troca o token; o aparelho substituído recebe `device_suspended` e guarda a fila |
| — | Q30, Q34 | Task 14: o `map` da sessão é o publicado; `If-Range` na continuação, `200`/`416` recomeçam |
| — | Q10 | Task 26: cards por superfície |
| — | Q49 | Task 4: Ajv com `multipleOfPrecision: 8` |
