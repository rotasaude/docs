# Módulo 35b — App de campo (`rotasaude/campo`): animal, vacinação antirrábica, coleta de LV e coleira offline — Implementation Plan

> **Nota do coordenador (Q29, docs d4728f1):** a N1 foi resolvida pela Q29. O `tutor_client_uuid` cita o `client_uuid` de um item `patient` **ou** de um `individual_registration` com `new_patient` do mesmo lote. A recusa fora disso é `tutor_not_found`, e o caso "tutor novo no mesmo lote" fica na prova.

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Reconciliado com o §19 do contrato até a Q27 (2026-11-23).** As D1–D14 foram decididas pelas Q10 e Q19–Q24, e a coluna "Decisão" da tabela do fim diz onde o plano as aplica. Fica aberta só a **N1**: o tutor novo do ACS nasce como `individual_registration` com `new_patient`, e não como item `patient`. A Task 6 (tutor nascido no tablet) e o caso 1 da Task 12 aguardam a N1.

**Goal:** Dar ao agente de campo (CBO `515140` e `515105`) no app o cadastro do animal do domicílio, a vacinação antirrábica casa a casa, a coleira e a lista "Coleiras a trocar", e ao agente de endemias (`515140`) a coleta de sangue do inquérito de leishmaniose, tudo offline pela fila do 24c, sobre o `field-sync` v1.4.0.

**Architecture:**
- A mecânica do 24c e do 35a fica como está, e os tipos são gerados da cópia presa da tag `field-sync-v1.4.0`.
- O esquema local passa a v3 (`animals`, `leish_surveys` e `local_leish_collections`), com uma migração 2→3 que não apaga linha e zera o cursor.
- O download guarda `animal` e `leish_survey` e aplica as remoções dos dois (`out_of_scope`, Q21).
- As edições do animal usam o caminho versionado do 24c (`targetTable`, `VERSIONED`, `editBase`), com `stale_version` sem mescla (Q20).
- Os quatro itens novos citam o animal nascido no tablet por `animal_client_uuid`, com dependência `cite`. O descarte do animal leva os dependentes pelo `doomedBy` de hoje.
- A etiqueta da coleta é do api e só existe depois do aceite (Q21). "Coleiras a trocar" é uma lista própria na área de animais (Q23).
- Quem decide é sempre o api.

**Tech Stack:** a do 24c e do 35a, sem dependência nova: Expo SDK 58.0.25, React Native 0.88.1, TypeScript estrito, Expo Router (`root: "app"`), `expo-sqlite` com SQLCipher 4.17.0, Jest 30 + `jest-expo` + RNTL + `sql.js` + Ajv 2020, Maestro 2.x, Gradle local.

**Spec:** `docs/superpowers/specs/2026-11-14-module-35-zoonoses-endemics-design.md`

**Contrato (fonte única):** `docs/superpowers/plans/2026-11-23-module-35b-zoonoses-contracts.md`, §1, §2, §3, §6, §11, §12, §16 e §17. O §19 (Q1–Q27) prevalece.

**Base:**
- contrato do 35a `2026-11-16-module-35a-vector-control-contracts.md`, §13, §14 e §20 até a Q63;
- plano `2026-11-16-module-35a-vector-control-campo.md`, que é o molde deste.

**Origem das frases (Q24):** `ZOONOSES_ERRORS` em `apps/dashboard/src/lib/zoonosesErrors.ts`, no commit do dashboard do 35b que o cria.

## Global Constraints

- Repositório `rotasaude/campo`, base `main` em `dbc54b3`, branch `feat/mod-35b-zoonoses`, worktree `/Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/campo-mod35b`. Nunca `node_modules` dentro de `apps/<outro app>`. Os worktrees do 35a ficam de pé e não são usados.
- Contrato: tag `field-sync-v1.4.0` (MINOR). `session.contract` continua `"field-sync-v1"`. A cópia em `contracts/field-sync/` é presa por `SHA256SUMS`, `TAG` e pelos seis SHA-256 que o api fixa em `FieldSync::Schemas::SHA256`.
- Nada fica obrigatório:
  - `features.zoonoses` ausente = `false`;
  - `zoonoses_domains` ausente = sem telas de zoonoses;
  - um código desconhecido só no `zoonoses_domains` derruba só esse bloco, como a Q44(1) do 35a faz com o `vector_control_domains`.
- Quem usa (§12, Q10):
  - `features.zoonoses` = `Zoonoses::Gate.field_usable?`;
  - `canUseZoonoses` = `features.zoonoses === true`, CBO `515140` ou `515105` e `reference.zoonosesDomains !== null`;
  - só o `515140` coleta (`canCollectLeish`);
  - o app oferece só os domicílios das microáreas da sessão, e os outros CBOs não veem nada novo.
- Formas do envio (Q20):
  - **`animal`:**
    - leva o corpo inteiro, e `animal_id` vai na edição junto com `version`;
    - `microchip` vai só na criação;
    - `status` vai só na edição e nunca volta a `active`;
    - tutor é proibido com `community: true`;
    - o nascimento vai em `birth_estimate_month` (`"AAAA-MM"`).
  - **`rabies_vaccination`:** `adverse_note` vai só com `adverse_event: true`.
  - **`leish_collection`:** `dpp_result` e `dpp_on` vão sempre juntos.
  - `last_rabies_vaccination_on` e `collar_due_on` mudam sem `version` nova, e o app sobrescreve o animal a cada recebimento (Q29 do 35a).
- Esquema local:
  - `SCHEMA_VERSION = 3`;
  - a migração nunca apaga linha, zera `meta.cursor` e mantém `last_download_ok_at` e o horizonte (Q13(4), Q61);
  - a primeira página sem cursor segue a Q42: `wipeTerritory(tx, { restart: true })`;
  - versão desconhecida continua indo para `UnknownSchemaError`.
- Banco (Q75 do 24c): toda transação é `db.tx(fn)`. Dentro do `fn`, só o `tx` recebido.
- Sigilo (§16, Q9, Q20):
  - o número do microchip vai **só** no corpo do item `animal` de criação. O app não oferece `registry_number`;
  - o número nunca entra em `animals.body`, em `log()`, em lista, em "precisa de atenção" nem em script do Maestro. O app guarda só `has_microchip`;
  - o tutor sai só como `display_name`;
  - `log()` leva só `client_uuid`, tipo, código e `seq`.
- Frases (§12, Q24):
  - as recusas e as regras de tela são copiadas byte a byte de `ZOONOSES_ERRORS` (`apps/dashboard/src/lib/zoonosesErrors.ts`), no molde `AGENT_*` de `src/features/messages/errorPhrases.ts`;
  - **o campo não redige frase**;
  - os textos que o contrato fixa são usados como estão: "Animais", "Vacinar", "Colocar coleira", "Coletar sangue (LV)", "Etiqueta depois de enviar", "Coleiras a trocar" e "Notifique o evento adverso no formulário do Ministério da Saúde (EATAV).".
- Testes:
  - só `bin/test campo -w campo-mod35b [args do jest]`;
  - lint e tipos pelo comando da CI (`npx eslint .`, `npx tsc --noEmit`), que não são teste;
  - o Maestro fica fora do `bin/test` e só digita ASCII no Android: nada de acento em `inputText` nem na asserção que lê esse texto.
- Git: `/opt/homebrew/bin/git`. Nunca `git add -A` nem `git add .`. Conventional Commits em inglês, com o tipo por extenso. Forma de todo commit:
  `G=/opt/homebrew/bin/git; W=/Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/campo-mod35b; $G -C $W add <arquivos>; $G -C $W commit -m "<tipo>: <descrição>" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`
- Execução coordenada: ao fim de cada task, um relato curto à sessão "Próximos módulos". Divergência nova vai para o fim deste plano e para o coordenador. Só ele escreve no contrato.
- Push, PR e merge só com autorização do usuário. A Task 13 para antes do merge.

## Ambiente de execução

- **Dependências de ordem (§18).** Sem uma delas, a task que depende **para** e avisa o coordenador:
  - tag `field-sync-v1.4.0` publicada (Task 1);
  - `apps/dashboard/src/lib/zoonosesErrors.ts` commitado no `dashboard-mod35b` (Task 5, Q24);
  - api do 35b inteiro commitado (prova da Task 12).
- **Gate de go-live (§11):** nenhum aparelho com app anterior ao 35a numa cidade com `zoonoses` ligado. Em dev, vale a ordem da Q6 do 35a (Q24): desligar o `zoonoses`, parear o APK do 35b, religar e conferir o `reset: true`.
- **Worktree:** `.worktrees/campo-mod35b`, com `npm ci` no host. O Gradle usa esse `node_modules`.
- **Android** (memórias "emulador Android do campo" e "24c build Android"), só no shell do comando:
  - `ANDROID_HOME=~/Library/Android/sdk`, `JAVA_HOME=/opt/homebrew/opt/openjdk@21/libexec/openjdk.jdk/Contents/Home` e o AVD `campo_api34`;
  - build: `npx expo prebuild --platform android --clean` + `./gradlew :app:assembleRelease -PreactNativeArchitectures=arm64-v8a`;
  - offline e Maestro **só com APK release**, e rodada longa sob `caffeinate -i`;
  - variável nova em `maestro/scripts/` entra nas globais do `eslint.config.js`.
- **Dev do 35b:**
  - api em `10.0.2.2:3061`, dashboard em `5221` e maintenance em `5177`, com o serviço do compose parado;
  - contas da semente (§17): `ace@curitiba.demo` (endemias), `zoonoses@curitiba.demo`, `veterinaria@curitiba.demo`, `endemias@curitiba.demo` (coordenação) e o ACS da semente do 24c, para o caso do tutor novo (Q22, N1);
  - senha e TOTP da semente podem aparecer no chat (memória "senhas de dev no chat").

## Estratégia de teste

- **Unidade (Jest, `sql.js`):**
  - migração 2→3 e 1→3 a partir de bancos congelados;
  - download com as páginas de exemplo da tag;
  - regras de tela;
  - corpos validados contra `upload-request-v1.json`;
  - aceite, `stale_version`, descarte e bloqueio na fila.
- **Tela (RNTL):** quem vê cada botão, por CBO e por interruptor; a etiqueta antes e depois do aceite; o microchip fora da tela depois de salvo.
- **Servidor falso (`server.test.ts`, no `bin/test`):** o perfil de zoonoses responde válido pelos esquemas da v1.4.0.
- **Maestro:** os quatro fluxos novos offline e os quinze de antes, no APK release.
- **Prova integrada:** emulador contra o api 3061, com os casos do §17, passo 9.

## Review Focus

1. **Microchip vazando.** O risco é o número aparecer:
   - no `body` local ou num `log`;
   - na lista do domicílio;
   - na tela "precisa de atenção" depois de `microchip_taken`;
   - no texto do `phraseFor`;
   - no `current` de um `stale_version`.

   O número fica só no corpo do item de criação, e a tela mostra só "tem microchip". Testes nas Tasks 6 e 9 e no `noLeak.test.ts`.
2. **Cadeia offline do animal.** Os casos: animal novo + vacina + coleira + coleta antes de sincronizar; animal recusado; animal aceito com o download ainda por fazer; descarte do animal.
   - Os três itens citam o animal e esperam por ele, e a recusa do animal os bloqueia.
   - Depois do aceite, os próximos vão por `animal_id`.
   - O descarte avisa quantos registros vão junto.

   Testes nas Tasks 6, 7, 8 e 9.
3. **Interruptor e migração no meio do trabalho.** Os casos: tablet v2 com fila; `reset` por ligar ou desligar o `zoonoses`; item na fila com o interruptor desligado.
   - A migração guarda a fila e o território.
   - O `reset` apaga os animais, menos os que a fila cita.
   - O item espera em `pending` com `feature_disabled`.

   Testes nas Tasks 2, 3 e 9.
4. **Etiqueta da coleta.** Os casos: tela fechada antes do aceite; `duplicate` no reenvio; recomeço da Q42 depois do aceite; troca de dia.
   - A etiqueta nunca aparece antes do aceite e é sempre o `label` da resposta.
   - Ela sobrevive ao fechamento da tela e ao recomeço.
   - Ela sai na purga do dia seguinte, como as visitas.

   Testes na Task 8.
5. **Referência e feed desencontrados.** Os casos: `zoonoses` ligado sem `zoonoses_domains`; `open_campaign` nulo ou trocado com vacina na fila; inquérito removido com coleta na fila; animal que sai de `active` com vacina na fila; segunda dose já aceita na mesma campanha.
   - Os botões somem sem perder a fila.
   - A recusa do servidor vira "precisa de atenção", com a frase e "Descartar". Os códigos: `campaign_not_open`, `survey_not_open`, `animal_not_active`, `already_vaccinated_in_campaign`, `date_out_of_range` e `household_not_in_survey`.
   - A segunda dose é barrada só pelo api (Q19).

   Testes nas Tasks 4, 7 e 8.

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| `contracts/field-sync/**`, `scripts/contract-types.mjs`, `src/contract/{index.ts,generated/*}`, `src/contract/*.test.ts`, `src/features/outbox/OutboxScreen.tsx` (`ITEM_TYPE_LABEL`) | cópia presa da v1.4.0 e tipos | 1 |
| `src/db/migrations.ts`, `src/db/testing/schemaV2.ts`, `src/db/repos/queued.ts`, `src/security/wipe.ts` | esquema v3 e migração 2→3 | 2 |
| `src/db/repos/zoonosesRepo.ts`, `src/db/repos/territoryRepo.ts`, `src/sync/download.ts` | `animal`, `leish_survey` e as duas remoções | 3 |
| `src/rules/zoonosesDomains.ts`, `src/sync/reference.ts`, `src/features/session.ts` | `zoonoses_domains` e quem usa | 4 |
| `src/rules/zoonosesErrors.ts`, `src/rules/animal.ts`, `src/features/messages/errorPhrases.ts`, `src/features/conflicts/fieldText.ts` | frases (porte do dashboard) e regras de tela | 5 |
| `src/features/animals/{save,view}.ts`, `src/features/animals/{AnimalForm,AnimalSection}.tsx`, `app/(main)/animals/**`, `src/features/households/HouseholdScreen.tsx`, `src/sync/{outbox,dispatch,stale}.ts` | cadastro e edição do animal | 6 |
| `src/features/animals/{VaccinateForm,CollarForm}.tsx`, `src/features/animals/save.ts`, `src/sync/dispatch.ts` | vacinação e coleira | 7 |
| `src/features/animals/LeishCollectForm.tsx`, `src/features/animals/{save,view}.ts`, `src/sync/{dispatch,outbox,retention}.ts`, `src/features/visits/TodayScreen.tsx` | coleta de LV e etiqueta | 8 |
| `src/features/attention/AttentionScreen.tsx`, `src/features/outbox/OutboxScreen.tsx`, `src/sync/outbox.ts` (`forget`) | descarte e "precisa de atenção" dos itens novos | 9 |
| `src/features/animals/{collars.ts,CollarsDueScreen.tsx}`, `app/(main)/animals/collars.tsx`, `src/features/home/*` | "Coleiras a trocar", lista própria (Q23) | 10 |
| `scripts/fake-field-server/**`, `maestro/{animal-register-offline,rabies-vaccination-offline,leish-collection-offline,collar-offline}.yaml`, `maestro/scripts/expect-zoonoses.js`, `maestro/README.md`, `eslint.config.js` | Maestro | 11 |

## Esquema local v3 (acréscimos ao v2)

| Tabela | Colunas (`NOT NULL` salvo `?`) |
|---|---|
| `animals` | `key` PK (id do servidor ou `client_uuid` da criação no tablet), `id?` único, `client_uuid?`, `household_key`, `species` (`CHECK dog\|cat\|other`), `status`, `version?`, `base_version?`, `needs_reload` (`0/1`), `body` (forma `animal` do download, sintetizada na criação, **sem microchip**) |
| `leish_surveys` | `id` PK, `version`, `body` (forma `leish_survey`, com `household_ids`, Q21) |
| `local_leish_collections` (Q21) | `client_uuid` PK, `animal_key`, `city_day`, `label?`, `state` (`CHECK pending\|accepted`), `accepted_at?` |

Índices: `idx_animals_household (household_key)`, `idx_local_leish_collections_day (city_day)`.

---

### Task 0: Base, card e worktree

**Files:** nenhum código.

- [ ] **Step 1: Conferir a base** — `G=/opt/homebrew/bin/git; $G -C /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/campo fetch origin && $G -C /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/campo rev-parse --short origin/main`. Expected: `dbc54b3`. Outro commit: anote e avise o coordenador antes de seguir.
- [ ] **Step 2: Card** — o card do campo do 35b (entre F-35.11 e F-35.18, pelo título) no board #1 vai para In Progress.
  - Um `gh` por comando, começando por `gh`: `gh project item-list 1 --owner rotasaude --format json --limit 500`, depois `gh project item-edit …`.
  - Se outra sessão já o pôs em In Progress, não mexa.
- [ ] **Step 3: Worktree**

```bash
G=/opt/homebrew/bin/git; R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
$G -C $R/apps/campo worktree add $R/.worktrees/campo-mod35b -b feat/mod-35b-zoonoses origin/main
cd $R/.worktrees/campo-mod35b && npm ci
```
- [ ] **Step 4: Linha de base** — `bin/test campo -w campo-mod35b`. Expected: `resultado: ok`. Anote o total de testes para o relato final.

---

### Task 1: Contrato `field-sync` v1.4.0 e tipos gerados

Depende da tag `field-sync-v1.4.0`. Sem ela, pare e avise o coordenador.

**Files:**
- Modify: `contracts/field-sync/**` (cópia da tag), `contracts/field-sync/{SHA256SUMS,TAG}`, `scripts/contract-types.mjs` (banner `field-sync-v1.4.0`), `src/contract/generated/*.ts`, `src/contract/index.ts`, `src/features/outbox/OutboxScreen.tsx` (`ITEM_TYPE_LABEL`)
- Test: `src/contract/contract.sha.test.ts`, `src/contract/contract.examples.test.ts`, `src/contract/contract.types.test.ts`

**Interfaces:**
- Produces (`src/contract/index.ts`), além do que já existe:
  - `ItemType` passa a incluir `"animal" | "rabies_vaccination" | "leish_collection" | "collar"`;
  - do envio: `AnimalItemBody`, `RabiesVaccinationItemBody`, `LeishCollectionItemBody`, `CollarItemBody`;
  - do download: `AnimalBody`, `LeishSurveyBody`; `ChangeType` passa a incluir `"animal" | "leish_survey"`; `RemovalType = Removal["type"]` inclui `"animal" | "leish_survey"`;
  - da referência: `ZoonosesDomainsRaw = NonNullable<ReferenceResponse["zoonoses_domains"]>`;
  - a seção `zoonoses` do `domains-v1.json` sai por `domains.zoonoses`, com rótulos, as chaves de `limits`, os tipos de inquérito e as finalidades de produto (Q20).
  - Os nomes `$defs` saem do esquema da tag. Se o gerador der outro nome, o apelido em `index.ts` mantém estes.
- Produces (`ITEM_TYPE_LABEL`): `animal` "Cadastro de animal", `rabies_vaccination` "Vacina antirrábica", `leish_collection` "Coleta de sangue (LV)" e `collar` "Coleira". Ficam até a Task 5, que os troca pelos do dashboard se ele tiver outros.

- [ ] **Step 1: Copiar a tag** — siga o Step 1 da Task 2 do plano do campo do 35a, trocando `field-sync-v1.3.0` por `field-sync-v1.4.0` e `campo-mod35a` por `campo-mod35b`.
  - Antes, `rm -rf $W/contracts/field-sync`, extraia de novo e refaça `SHA256SUMS`.
  - Os arquivos removidos entram no commit por `git rm` de cada um.
- [ ] **Step 2: Atualizar os testes**

```ts
// contract.sha.test.ts
test("TAG names field-sync-v1.4.0 and a commit", () => expect(tag).toMatch(/^field-sync-v1\.4\.0 [0-9a-f]{40}$/));
// PINNED: os seis SHA-256 de FieldSync::Schemas::SHA256 no api do 35b (o relato do contracts os traz)
// contract.examples.test.ts
test("session and reference of v1.3.0 stay valid without the zoonoses keys", …);
test("animal, rabies_vaccination, leish_collection and collar examples validate; each invalid example is refused", …);
test("the animal change body has has_microchip and no microchip key", () => {
  const animal = page.items.find((i) => i.type === "animal")!.body;
  expect(animal).toHaveProperty("has_microchip");
  expect(Object.keys(animal)).not.toContain("microchip");
});
test("leish_survey carries household_ids (Q21)", …);
test("removals accept animal and leish_survey with out_of_scope", …);
test("accepted animal carries version; leish_collection accepted and duplicate carry label", …);
test("the new item codes are in errors.item", () => {
  expect(KNOWN_ITEM_ERRORS).toEqual(expect.arrayContaining(["microchip_taken", "registry_number_taken", "already_vaccinated_in_campaign",
    "campaign_not_open", "survey_not_open", "animal_not_dog", "animal_not_active", "date_out_of_range", "household_not_in_survey"]));
});
```
- [ ] **Step 3: Rodar e ver falhar** — `bin/test campo -w campo-mod35b src/contract` · Expected: FAIL (TAG, PINNED, tipos velhos).
- [ ] **Step 4: Gerar** — `node scripts/contract-types.mjs` no host. Acrescente os apelidos em `index.ts` e as quatro linhas de `ITEM_TYPE_LABEL`.
- [ ] **Step 5: Rodar e ver passar** — `bin/test campo -w campo-mod35b src/contract`, depois a suíte inteira e `npx tsc --noEmit`.
  - Expected: `src/contract` PASS.
  - Na suíte, só o `every known http and item code has its own phrase` fica vermelho, **de propósito até a Task 5**. Anote no relato.
  - Qualquer outro vermelho se corrige aqui, inclusive o erro de tipo de um `switch` exaustivo sobre `ItemType` ou `ChangeType`.
- [ ] **Step 6: Commit** — `contracts/field-sync` (lista de `$G -C $W status --porcelain contracts`), `scripts/contract-types.mjs`, `src/contract/*`, `src/features/outbox/OutboxScreen.tsx` · `feat: pin the field-sync v1.4.0 contract and regenerate its types`.

---

### Task 2: Esquema local v3 e migração 2→3

**Files:**
- Modify: `src/db/migrations.ts`, `src/db/repos/queued.ts`, `src/security/wipe.ts`
- Create: `src/db/testing/schemaV2.ts` (os `CREATE` do v1 e do v2 congelados de `dbc54b3`, só para teste)
- Test: `src/db/migrations.test.ts`, `src/security/wipe.test.ts`

**Interfaces:**
- Produces: `SCHEMA_VERSION = 3` e `STATEMENTS_V3`, com as tabelas e os índices do "Esquema local v3". `migrate(db)` roda numa transação:
  - `"1"` → cria v2 e v3, apaga `meta.cursor` e grava `"3"`;
  - `"2"` → cria v3, apaga `meta.cursor` e grava `"3"`;
  - `"3"` → cria v2 e v3 (idempotente);
  - outro valor → `UnknownSchemaError`, e nada muda;
  - `last_download_ok_at` e `HORIZON_KEY` ficam.
- Produces (`queued.ts`): `QUEUED_ANIMAL`, a regra do `QUEUED_OVITRAP` aplicada ao animal. Vale para `client_uuid` não nulo que a fila cita, seja como item próprio, seja como `json_extract(body, '$.animal_client_uuid')` de qualquer item.
- Produces (`wipe.ts`): `wipeTerritory` apaga `animals WHERE NOT (QUEUED_ANIMAL)` e `leish_surveys`.
  - Com `restart` (Q42), `local_leish_collections` fica.
  - Sem `restart`, apaga as linhas que a fila não cita (`client_uuid NOT IN (SELECT client_uuid FROM outbox)`), como `local_visits`.

- [ ] **Step 1: Escrever os testes**

```ts
test("creates every table and index of the local schema v3", …);   // TABLES + animals, leish_surveys, local_leish_collections; INDEXES + os dois
test("upgrades a v2 database with data to v3, keeps everything and resets the cursor", async () => {
  const db = await createEmptyTestDb();
  await db.tx(async (tx) => { for (const s of [...SCHEMA_V1, ...SCHEMA_V2]) await tx.exec(s); });
  await db.run("INSERT INTO meta (key, value) VALUES ('schema_version', '2'), ('cursor', 'c2'), ('last_download_ok_at', '2026-11-20T10:00:00Z'), ('last_download_horizon_at', '2026-11-20T09:59:58Z')");
  await seedV2(db);   // um domicílio, uma ovitrampa, dois itens na fila (um pending, um needs_attention), uma visita local
  await migrate(db);
  expect(await db.get("SELECT value FROM meta WHERE key = 'schema_version'")).toEqual({ value: "3" });
  expect(await db.get("SELECT value FROM meta WHERE key = 'cursor'")).toBeNull();
  expect(await db.get("SELECT value FROM meta WHERE key = 'last_download_ok_at'")).toEqual({ value: "2026-11-20T10:00:00Z" });
  expect(await db.get("SELECT value FROM meta WHERE key = 'last_download_horizon_at'")).toEqual({ value: "2026-11-20T09:59:58Z" });
  expect(await db.all("SELECT seq, state FROM outbox ORDER BY seq")).toEqual([{ seq: 1, state: "pending" }, { seq: 2, state: "needs_attention" }]);
  expect(await db.get("SELECT count(*) AS n FROM ovitraps")).toEqual({ n: 1 });
});
test("a v1 database goes straight to v3", …);
test("migrating a v3 database again keeps its cursor", …);
test("a database from a newer app is refused, untouched", …);   // o teste existente passa a usar '4'
test("a fresh install starts at 3", …);
// wipe.test.ts
test("wipeTerritory empties animals and leish surveys but keeps an animal the queue creates or cites by animal_client_uuid", …);
test("the Q42 restart keeps local_leish_collections; a server reset keeps only the ones the queue still has", …);
```
- [ ] **Step 2: Rodar e ver falhar** — `bin/test campo -w campo-mod35b src/db/migrations.test.ts src/security/wipe.test.ts` · Expected: FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4: Rodar e ver passar** — mesmo comando · Expected: PASS.
- [ ] **Step 5: Commit** — `src/db/migrations.ts`, `src/db/testing/schemaV2.ts`, `src/db/repos/queued.ts`, `src/security/wipe.ts` e testes · `feat: migrate the local schema to v3 with the zoonoses tables`.

---

### Task 3: Download de `animal`, `leish_survey` e das remoções

**Files:**
- Create: `src/db/repos/zoonosesRepo.ts`
- Modify: `src/sync/download.ts`, `src/db/repos/territoryRepo.ts` (`removeHousehold`)
- Test: `src/sync/download.test.ts`, `src/db/repos/zoonosesRepo.test.ts`

**Interfaces:**
- Consumes: `AnimalBody`, `LeishSurveyBody` (Task 1); as tabelas e `QUEUED_ANIMAL` (Task 2).
- Produces (`zoonosesRepo`):
  - `AnimalRow { key; id: string | null; client_uuid: string | null; household_key: string; species: AnimalBody["species"]; status: string; version: number | null; body: AnimalBody }`.
  - `putAnimal(db, a: AnimalBody)`:
    - acha a linha pelo `id` ou por `client_ids` (tipo `animal`);
    - `household_key` = `territoryRepo.householdKey(a.household_id) ?? a.household_id`;
    - sobrescreve qualquer que seja a `version` (Q20, Q29 do 35a) e grava `version`, `base_version` = `a.version` e `needs_reload = 0`;
    - `status ≠ "active"` apaga a linha (é uma defesa: o fim normal vem pela remoção, Q21).
  - `removeAnimal(db, id)`.
  - `putLeishSurvey(db, s: LeishSurveyBody)`: substitui qualquer que seja a `version`. `removeLeishSurvey(db, id)`.
  - `animalsOf(db, householdKey): Promise<AnimalRow[]>`, ordenados pelo `name` sem caixa e depois pela `key`.
  - `openSurveysFor(db, householdId: string | null): Promise<LeishSurveyBody[]>`: os inquéritos com `householdId` em `household_ids` (Q21). Com `householdId` nulo (domicílio nascido no tablet), devolve vazio, e só a coleta avulsa fica.
- Produces (`download.ts`):
  - `KNOWN_CHANGE_TYPES` = `micro_area`, `city_block`, `household`, `registration`, `animal`, `ovitrap`, `liraa_item`, `transmission_block`, `leish_survey`, aplicados nessa ordem (o domicílio antes do animal);
  - as remoções ganham `case "animal"` → `zoonosesRepo.removeAnimal` e `case "leish_survey"` → `zoonosesRepo.removeLeishSurvey` (`out_of_scope`, Q21).
- Produces (`removeHousehold`): apaga também `animals WHERE household_key = ? AND NOT (QUEUED_ANIMAL)`.

- [ ] **Step 1: Escrever os testes.** As páginas são os exemplos novos da v1.4.0, conferidos por `expectValid("changes-response", …)`.

```ts
test("a page with animals and a leish survey is stored; the animal is filed under the household's local key", …);
test("an animal body never stores a microchip number, only has_microchip", …);
test("an animal re-emitted with the same version and a new collar_due_on overwrites the row (Q20)", …);
test("removal animal out_of_scope deletes the row; removal leish_survey out_of_scope deletes the survey (Q21)", …);
test("the animal born on the tablet is replaced by the server's through client_ids once accepted", …);
test("moved household takes its animals with it, but not one the queue still creates or cites", …);
test("openSurveysFor lists only surveys whose household_ids contain the household", …);
test("the reset that comes when zoonoses toggles empties animals and surveys and refills them", …);
test("a v1.3.0 page (no animal type) applies as before", …);
```
- [ ] **Step 2: Rodar e ver falhar** — `bin/test campo -w campo-mod35b src/sync/download.test.ts src/db/repos/zoonosesRepo.test.ts` · Expected: FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4: Rodar e ver passar** — mesmo comando · Expected: PASS.
- [ ] **Step 5: Commit** — `src/db/repos/zoonosesRepo.ts`, `src/db/repos/territoryRepo.ts`, `src/sync/download.ts` e testes · `feat: download animals and leishmaniasis surveys`.

---

### Task 4: Referência `zoonoses_domains` e quem usa

**Files:**
- Create: `src/rules/zoonosesDomains.ts`
- Modify: `src/sync/reference.ts`, `src/features/session.ts`
- Test: `src/sync/reference.test.ts`, `src/features/session.test.ts`

**Interfaces:**
- Produces (`zoonosesDomains.ts`):

```ts
export const ANIMAL_SPECIES = ["dog", "cat", "other"] as const;
export const ANIMAL_SEXES = ["male", "female", "unknown"] as const;
export const ANIMAL_SIZES = ["small", "medium", "large"] as const;
export const ANIMAL_STATUSES = ["active", "deceased", "missing", "transferred"] as const;
export const DPP_RESULTS = ["reactive", "non_reactive", "invalid"] as const;
export const ZOONOSES_PRODUCT_USES = ["rabies_vaccine", "collar"] as const;
export interface ZoonosesDomains {
  species: { code: AnimalSpecies; label: string }[]; sexes: …; sizes: …; animal_statuses: …; dpp_results: …;
  products: { id: string; code: string; name: string; use: ZoonosesProductUse }[];
  open_campaign: { id: string; name: string; starts_on: string; ends_on: string } | null;
  parameters: { collar_months: number };
  limits: ZoonosesLimits;   // as chaves da seção zoonoses do domains-v1.json (Q20)
}
export function parseZoonosesDomains(raw: unknown): ZoonosesDomains | null;
// ausente → null; fora da forma → ReferenceShapeError; código desconhecido só aqui → null (Q44(1) do 35a)
```
- Produces: `Reference.zoonosesDomains: ZoonosesDomains | null`.
- Produces (`session.ts`):
  - `zoonosesOn(session)`: `features.zoonoses === true` (Q10: é o `field_usable?`);
  - `ZOONOSES_CBOS = [ENDEMIC_AGENT_CBO, ACS_CBO] as const`;
  - `canUseZoonoses(session, reference)`: `zoonosesOn`, CBO em `ZOONOSES_CBOS` e `reference?.zoonosesDomains` não nulo;
  - `canCollectLeish(session, reference)`: `canUseZoonoses` e CBO `515140`.

- [ ] **Step 1: Escrever os testes**

```ts
test("reference without zoonoses_domains parses with zoonosesDomains null", …);   // exemplo da v1.3.0
test("zoonoses_domains of the v1.4.0 example is parsed; products only rabies_vaccine and collar", …);
test("open_campaign null is kept as null", …);
test("an unknown species code drops only the zoonoses block; vectorDomains and the forms stay", …);
test("a malformed zoonoses block keeps the previous reference (kept_previous)", …);
test.each([
  ["515140", true, true, true], ["515105", true, true, true], ["223505", true, true, false],
  ["515140", false, true, false], ["515140", undefined, true, false], ["515140", true, false, false],
])("canUseZoonoses(cbo %s, zoonoses %s, domains %s) → %s", …);
test("canCollectLeish only for 515140", …);
test("zoonoses on and vector_control off: canUseZoonoses true, canRecordInspection false", …);
```
- [ ] **Step 2–4:** `bin/test campo -w campo-mod35b src/sync/reference.test.ts src/features/session.test.ts` FAIL → implementar → PASS.
- [ ] **Step 5: Commit** — `feat: read the zoonoses domains from the offline reference`.

---

### Task 5: Frases e regras de tela, portadas do dashboard (Q24)

O porte é uma cópia, sem mudar regra nem frase, de `apps/dashboard/src/lib/zoonosesErrors.ts` e das regras de formulário das telas `zoo-animals`, `zoo-campaigns` e `zoo-leish` do dashboard do 35b. O commit de origem vai no cabeçalho, como no `vectorErrors.ts`. **Espera o commit do `zoonosesErrors.ts` no `dashboard-mod35b`.** Sem ele, pare e avise o coordenador.

**Files:**
- Create: `src/rules/zoonosesErrors.ts`, `src/rules/animal.ts`, `src/rules/testing/zoonosesFixtures.ts`
- Modify: `src/features/messages/errorPhrases.ts`, `src/features/conflicts/fieldText.ts`, `src/features/outbox/OutboxScreen.tsx` (os rótulos da Task 1, se o dashboard tiver outros)
- Test: `src/rules/zoonosesErrors.test.ts`, `src/rules/animal.test.ts`, `src/features/messages/errorPhrases.test.ts`, `src/features/conflicts/fieldText.test.ts` (crie se não houver)

**Interfaces:**
- Consumes: `ZoonosesDomains` (Task 4).
- Produces (`zoonosesErrors.ts`): `ZOONOSES_ERROR_CODES` e `ZOONOSES_ERRORS: Record<ZoonosesErrorCode, string>`, a lista do dashboard, com os nomes de constante dele. A lista cobre:
  - os nove códigos de item da v1.4.0;
  - a frase de animal alterado, a que o dashboard usa no 409 `animal_changed` e que o app mostra no `stale_version` do animal (Task 6);
  - as recusas locais no molde `AGENT_*`.
- Produces (`animal.ts`). Cada função devolve `{ code: string | null; path: string; text: string }[]`, com `text` = `ZOONOSES_ERRORS[code]`. Cada `path` é o do corpo do item:
  - `animalProblems(d: AnimalDraft, limits: ZoonosesLimits, today: string)`:
    - `microchip` com 15 dígitos (`^[0-9]{15}$`), só na criação;
    - `name`, `coat` e `breed` dentro de `limits`;
    - `birth_estimate_month` (`AAAA-MM`) não depois do mês de hoje;
    - tutor com `community: true` → recusa em `tutor_patient_id` (Q20);
    - sem tutor e sem `community` é válido (Q22).
  - `vaccinationProblems(animal: AnimalRow, domains: ZoonosesDomains, d: { adverseEvent: boolean; adverseNote: string | null })`:
    - espécie `dog` ou `cat`, e `animal_not_active`;
    - `campaign_not_open` sem `open_campaign`;
    - `adverse_note` só com evento adverso e dentro de `limits`.
    - A segunda dose na mesma campanha **não** é conferida aqui (Q19).
  - `collarProblems(animal)`: `animal_not_dog` e `animal_not_active`.
  - `collectionProblems(animal, surveyId: string | null, surveys: LeishSurveyBody[])`:
    - `animal_not_dog` e `animal_not_active`;
    - `survey_not_open` para um inquérito que o tablet não tem mais.
  - `AnimalDraft` e `emptyAnimalDraft()`. O `toAnimalBody(draft, ctx)` fica na Task 6.
- `errorPhrases.ts`:
  - cada código novo de `errors.item` recebe `ZOONOSES_ERRORS[code]`;
  - `phraseForItem` usa `ZOONOSES_ERRORS` nos quatro itens novos, como já faz com `VECTOR_ITEMS`;
  - `SHARED_24C_CODES` não muda: `not_found`, `feature_disabled` e `cbo_not_allowed` seguem com a frase do 24c.
- `fieldText.ts`: o `pathLabel` dos quatro itens põe em palavras, com os rótulos do dashboard: `animal_id` e `animal_client_uuid` ("Animal"), `microchip`, `tutor_patient_id`, `tutor_client_uuid`, `birth_estimate_month`, `status`, `campaign_id`, `vector_product_id`, `vaccinated_on`, `adverse_note`, `leish_survey_id`, `collected_on`, `dpp_result`, `dpp_on` e `placed_on`.

- [ ] **Step 1: Escrever os testes**

```ts
test("every zoonoses item code of the contract has the dashboard phrase", () => {
  for (const code of ["microchip_taken", "registry_number_taken", "already_vaccinated_in_campaign", "campaign_not_open", "survey_not_open",
    "animal_not_dog", "animal_not_active", "date_out_of_range", "household_not_in_survey"])
    expect(ERROR_PHRASES[code]).toBe(ZOONOSES_ERRORS[code as ZoonosesErrorCode]);
});
test("phraseForItem: not_found on a collar keeps the 24c phrase; animal_not_dog on a collar is the zoonoses one", …);
test.each([["98514100012345", false], ["985141000123456", true], ["98514100012345a", false]])("microchip %s valid = %s", …);
test("tutor with community true is refused at tutor_patient_id; no tutor and no community is fine (Q22)", …);
test("vaccination: cat ok, other refused; no open campaign → campaign_not_open at campaign_id", …);
test("adverse_note without adverse_event is refused; with it, under the limit is refused", …);
test("collar and collection on a cat → animal_not_dog at animal_id", …);
test("fieldText puts every path of the four items in words; the raw path never shows", …);
test("no phrase or label carries a value (microchip, name)", …);
```
- [ ] **Step 2: Rodar e ver falhar** — `bin/test campo -w campo-mod35b src/rules/zoonosesErrors.test.ts src/rules/animal.test.ts src/features/messages src/features/conflicts` · Expected: FAIL.
- [ ] **Step 3: Portar** (só imports e tipos do contrato).
- [ ] **Step 4: Rodar e ver passar** — o mesmo comando e depois `bin/test campo -w campo-mod35b` · Expected: PASS, sem o vermelho de propósito da Task 1.
- [ ] **Step 5: Commit** — `src/rules/zoonosesErrors.ts`, `src/rules/animal.ts`, `src/rules/testing/zoonosesFixtures.ts`, `src/features/messages/errorPhrases.ts`, `src/features/conflicts/fieldText.ts`, `src/features/outbox/OutboxScreen.tsx` e testes · `feat: port the dashboard zoonoses phrases and form rules`.

---

### Task 6: Cadastro e edição do animal (Q20, Q22; tutor nascido no tablet pende N1)

**Files:**
- Create: `src/features/animals/{save.ts,view.ts,AnimalForm.tsx,AnimalSection.tsx}`, `app/(main)/animals/new.tsx` (param `household`), `app/(main)/animals/[key]/edit.tsx`
- Modify: `src/features/households/HouseholdScreen.tsx` (seção "Animais"), `src/sync/outbox.ts` (`targetTable`, `forget`), `src/sync/dispatch.ts` (`VERSIONED`, `ID_KEY`, `accept`), `src/sync/stale.ts` (animal sem mescla), `src/db/repos/zoonosesRepo.ts` (`putBorn`, `acceptAnimal`), `src/features/attention/AttentionScreen.tsx` ("Corrigir" abre o `AnimalForm`)
- Test: `src/features/animals/save.test.ts`, `src/features/animals/view.test.ts`, `src/features/animals/AnimalScreens.test.tsx`, `src/sync/dispatch.test.ts`, `src/sync/stale.test.ts` (crie se não houver), `src/security/noLeak.test.ts`

**Interfaces:**
- Consumes: `animalProblems`, `AnimalDraft` e `ZOONOSES_ERRORS` (Task 5); `canUseZoonoses` (Task 4); `zoonosesRepo` (Task 3); `territoryRepo.residentsOf`; `coalesceOrEnqueue` e `enqueue` (24c).
- Produces (`outbox.ts`):
  - `targetTable("animal") = "animals"`;
  - o `forget` de `animal` apaga `animals WHERE key = ? AND id IS NULL AND client_uuid = ?`, porque só a criação é dona da linha.
- Produces (`dispatch.ts`):
  - `VERSIONED` e `ID_KEY` ganham `animal` / `animal_id`;
  - o aceite de `animal` chama `zoonosesRepo.acceptAnimal(db, item.target_id, item.body, { id, version })`, que grava `id`, `version` e `base_version`. A `version` vem no `accepted` (Q20). Sem ela, num `duplicate` antigo, a linha fica `needs_reload = 1` pelo caminho de hoje;
  - o laço que leva `edit_base_version` às edições seguintes vale para o animal sem mudança.
- Produces (`stale.ts`): o `stale_version` de `animal` segue o 24c **sem mescla** (Q20), numa transação:
  - a linha vira o `current`, pelo `putAnimal`;
  - o item vai para `needs_attention`, com `edit_base_version = current_version`, `edit_base_body = current` e o corpo do tablet intacto;
  - a tela mostra a frase de animal alterado de `ZOONOSES_ERRORS` e oferece "Corrigir" (o `AnimalForm` sobre a versão atual, que reenvia o mesmo `client_uuid`) e "Descartar";
  - `household` e `registration` não mudam.
- Produces (`save.ts`): `saveAnimal(db, input: { householdKey: string; draft: AnimalDraft; animalKey?: string; newUuid(): string }): Promise<string>`, numa transação:
  - **Criação:** item `animal` com o corpo inteiro, sem `animal_id` e sem `status`.
    - O domicílio vai por `household_id`, ou por `household_client_uuid` com `cite` enquanto está na fila.
    - O tutor vai por `tutor_patient_id`, ou por nenhum (Q22). O tutor nascido no tablet segue a N1: com a N1 aceita, `tutor_client_uuid` com o `client_uuid` do cadastro e `cite` enquanto ele estiver na fila, o mesmo critério do `patient_client_uuid` em `src/features/visits/save.ts:80-88`; sem a N1, o morador sem `patient.id` não é oferecido como tutor.
    - `community: true` só sem tutor. O `microchip` vai só se foi digitado, e o nascimento em `birth_estimate_month`.
    - Na mesma transação, `zoonosesRepo.putBorn` cria a linha local com `has_microchip` e sem o número. O `tutor` dela é `{ patient_id, display_name }`, lido do cadastro local.
  - **Edição:** `coalesceOrEnqueue`, com `targetId` = a chave do animal e `editBase` = `{ version: row.version, body: row.body }`.
    - O corpo inteiro leva `animal_id` e `status` (o de hoje ou o escolhido, nunca de volta a `active`) e nunca `microchip` (Q20).
    - Uma criação ainda na fila e não enviada recebe a edição no próprio corpo, pelo `coalesce` de hoje.
- Produces (`view.ts`): `readAnimals(db, householdKey): Promise<AnimalView[]>`.
  - `AnimalView { key; id; species; name; sexLabel; status; tutorName: string | null; community: boolean; hasMicrochip: boolean; lastVaccinationOn: string | null; collarDueOn: string | null; waiting: boolean; held: "needs_attention" | "blocked" | null }`.
  - A fila entra por cima, como em `ovitraps/view.ts`.
  - `ANIMAL_TABLES = ["animals", "outbox"]`.
- Tela:
  - `HouseholdScreen` ganha a seção "Animais" (`AnimalSection`), só com `canUseZoonoses` e o domicílio ativo. Ela traz "Cadastrar animal" (rótulo do dashboard, Q24) e um cartão por animal: espécie, nome, tutor ou "comunitário", "tem microchip", "Aguardando envio" (`WAITING`) e "Editar".
  - Campos do `AnimalForm`:
    - espécie, nome, sexo, porte, pelagem, raça (aceita "SRD") e nascimento estimado (mês e ano);
    - microchip só na criação; depois de salvo, só "tem microchip";
    - tutor entre os moradores que o tablet tem no domicílio, ou sem tutor (Q22);
    - "Animal comunitário", que tira o tutor;
    - situação, só na edição, sem volta a "ativo".
  - A recusa da tela rola até o campo (`testID` `field-<path>`).

- [ ] **Step 1: Escrever os testes**

```ts
// save.test.ts
test("new animal in a household born on the tablet cites household_client_uuid", …);
test("tutor known by the server goes by tutor_patient_id, without dependency", …);
test("no tutor and no community is saved as is (Q22); community true drops the tutor", …);
test("a resident registered on the tablet as tutor follows N1", …);   // N1 aceita: tutor_client_uuid + cite; recusada: não oferecido
test("the local row has has_microchip true and no microchip anywhere in animals", async () => {
  await saveAnimal(db, { householdKey, draft: { ...draft, microchip: "985141000123456" }, newUuid });
  expect(JSON.stringify(await db.all("SELECT * FROM animals"))).not.toContain("985141000123456");
});
test("edit carries the whole body with animal_id, version and status, never microchip; status never back to active", …);
test("edit of an animal still waiting in the line rewrites its creation body", …);
test("animal bodies validate against upload-request-v1.json; birth goes as birth_estimate_month", …);
// dispatch.test.ts
test("accepted animal maps client_ids, gives the row id and version, and the next edit goes by animal_id", …);
// stale.test.ts
test("stale_version on an animal replaces the row by current, keeps the tablet body, no merge, needs attention on current_version", …);
test("stale_version on a household still merges as in the 24c", …);
// AnimalScreens.test.tsx
test("Animais shows for 515140 and 515105 with zoonoses, never for another CBO or with the switch off", …);
test("after saving, the card says the animal has a microchip and the number is nowhere on screen", …);
test("tutor list is the residents the tablet has in the household; no tutor is allowed", …);
// noLeak.test.ts
test("a round with an animal carrying a microchip logs no 15-digit number", …);   // o LEAK de hoje já casa \b\d{15}\b
```
- [ ] **Step 2–4:** `bin/test campo -w campo-mod35b src/features/animals src/sync/dispatch.test.ts src/sync/stale.test.ts src/sync/outbox.test.ts src/features/households src/security/noLeak.test.ts` FAIL → implementar → PASS.
- [ ] **Step 5: Commit** — `feat: register and edit household animals offline`.

---

### Task 7: Vacinação antirrábica e coleira

**Files:**
- Create: `src/features/animals/{VaccinateForm,CollarForm}.tsx`, `app/(main)/animals/[key]/vaccinate.tsx`, `app/(main)/animals/[key]/collar.tsx`
- Modify: `src/features/animals/{save,view}.ts`, `src/features/animals/AnimalSection.tsx`, `src/sync/dispatch.ts` (aceite), `src/db/repos/zoonosesRepo.ts` (`acceptVaccination`, `acceptCollar`)
- Test: `src/features/animals/save.test.ts`, `src/features/animals/view.test.ts`, `src/features/animals/AnimalScreens.test.tsx`, `src/sync/dispatch.test.ts`

**Interfaces:**
- Consumes: `vaccinationProblems` e `collarProblems` (Task 5); `ZoonosesDomains.open_campaign`, `products` e `parameters.collar_months` (Task 4); `cityTodayOf`.
- Produces (`save.ts`). Nas duas funções, o animal vai por `animal_id`, ou por `animal_client_uuid` com `cite` enquanto a criação está na fila:
  - `vaccinate(db, input: { animalKey; productId; adverseEvent: boolean; adverseNote: string | null; today: string; newUuid() }): Promise<string>` → item `rabies_vaccination` `{ animal_id | animal_client_uuid, campaign_id: open_campaign.id, vaccinated_on: today, vector_product_id, adverse_event, adverse_note? }`, com `adverse_note` só com `adverse_event: true` (Q20). Não há modalidade: no app ela é sempre `house`.
  - `placeCollar(db, input: { animalKey; productId; today: string; newUuid() }): Promise<string>` → item `collar` `{ animal_id | animal_client_uuid, vector_product_id, placed_on: today }`.
- Produces (`view.ts`): `AnimalView.lastVaccinationOn` e `collarDueOn`, com a fila por cima:
  - a vacina na fila põe `vaccinated_on`;
  - a coleira na fila põe `placed_on + collar_months`, em meses do calendário e no dia que existir (31 → último dia do mês).
- Produces (aceite): até o download trazer o animal, `acceptVaccination` grava `last_rabies_vaccination_on` e `acceptCollar` grava `collar_due_on` no `body` da linha. O download depois sobrescreve (Q20).
- Tela:
  - "Vacinar" aparece só com `open_campaign`, espécie `dog` ou `cat` e animal ativo. Ele mostra "Última vacina antirrábica: dd/mm/aaaa" quando houver. Não confere dose na mesma campanha: a segunda dose é barrada só pelo api (Q19).
  - "Colocar coleira" aparece só em cão ativo.
  - Produto: a lista do `use` correspondente. Com um só produto, ele vem escolhido.
  - Evento adverso: chave e nota. Com a chave ligada, aparece o texto fixo do §3, "Notifique o evento adverso no formulário do Ministério da Saúde (EATAV).".
  - A falta de estoque não aparece no app: o item volta `accepted` (§11).

- [ ] **Step 1: Escrever os testes**

```ts
test("vaccinate an animal born offline: cites animal_client_uuid and waits for it", …);
test("vaccination body validates; campaign_id is the open campaign of the reference; no modality key; adverse_note only with adverse_event", …);
test("adverse event on shows the EATAV reminder", …);
test("no open campaign: no Vacinar button", …);
test("a second vaccination of the same campaign is offered; the api decides (Q19)", …);
test("collar only for an active dog; a cat has no Colocar coleira", …);
test("queued collar shows the new due date: placed 2026-08-31 with 6 months → 2027-02-28", …);
test("accepted vaccination and collar update the row until the download brings the animal", …);
test("a refused animal blocks its vaccination and collar (selectBatch + refreshBlocked of the 24c)", …);
test.each([["campaign_not_open", "campaign_id"], ["already_vaccinated_in_campaign", "campaign_id"], ["animal_not_active", "animal_id"],
  ["animal_not_dog", "animal_id"], ["date_out_of_range", "vaccinated_on"], ["date_out_of_range", "placed_on"]])(
  "server refusal %s at %s goes to attention with its phrase and the field in words", …);
```
- [ ] **Step 2–4:** `bin/test campo -w campo-mod35b src/features/animals src/sync/dispatch.test.ts` FAIL → implementar → PASS.
- [ ] **Step 5: Commit** — `feat: record rabies vaccinations and collars offline`.

---

### Task 8: Coleta de sangue (LV) e etiqueta depois do aceite (Q21)

**Files:**
- Create: `src/features/animals/LeishCollectForm.tsx`, `app/(main)/animals/[key]/collect.tsx`
- Modify: `src/features/animals/{save,view}.ts`, `src/features/animals/AnimalSection.tsx`, `src/db/repos/zoonosesRepo.ts` (`putCollection`, `acceptLeishCollection`, `collectionsOf`), `src/sync/dispatch.ts`, `src/sync/outbox.ts` (`forget`), `src/sync/retention.ts` (`purgeAccepted`), `src/features/visits/TodayScreen.tsx`
- Test: `src/features/animals/save.test.ts`, `src/features/animals/AnimalScreens.test.tsx`, `src/sync/dispatch.test.ts`, `src/sync/retention.test.ts`, `src/features/visits/TodayScreen.test.tsx`

**Interfaces:**
- Consumes: `collectionProblems` (Task 5); `canCollectLeish` (Task 4); `zoonosesRepo.openSurveysFor` (Task 3).
- Produces (`save.ts`): `collectLeish(db, input: { animalKey; surveyId: string | null; dppResult: DppResult | null; today: string; newUuid() }): Promise<string>`, numa transação:
  - item `leish_collection` `{ leish_survey_id?, animal_id | animal_client_uuid, collected_on: today, dpp_result?, dpp_on? }`. `dpp_result` e `dpp_on` vão juntos, com `dpp_on = today` (Q20);
  - linha `local_leish_collections` `{ client_uuid, animal_key, city_day: today, label: null, state: "pending" }`.
- Produces (aceite, `dispatch.ts`): o `accepted` e o `duplicate` de `leish_collection` chamam `acceptLeishCollection(db, client_uuid, result.label)`, que grava `label`, `state = "accepted"` e `accepted_at`. A etiqueta é do api, em hex maiúsculo (Q21).
- Produces:
  - o `forget` de `leish_collection` apaga a linha local;
  - o `purgeAccepted` apaga as `accepted` de dia anterior, como faz com `local_visits`.
- Produces (`view.ts`): `todayCollections(db, cityDay): Promise<{ clientUuid; animalName: string | null; label: string | null }[]>`.
- Tela:
  - "Coletar sangue (LV)" aparece só com `canCollectLeish`, em cão ativo.
  - Inquérito: os de `openSurveysFor(<id do domicílio>)` (Q21) e "Avulsa", que vai sem `leish_survey_id`.
  - TR-DPP é opcional, com os rótulos de `dpp_results`.
  - Antes do aceite, o cartão do animal e a linha em Hoje mostram "Etiqueta depois de enviar". Depois do aceite, mostram "Etiqueta: <label>".
  - O app nunca calcula nem prevê a etiqueta.

- [ ] **Step 1: Escrever os testes**

```ts
test("collection by 515140 of a dog born offline cites animal_client_uuid; body validates", …);
test("515105 has no Coletar sangue (LV); a cat neither", …);
test("surveys offered are the ones whose household_ids contain the household, plus Avulsa; a household born on the tablet gets only Avulsa", …);
test("no label before acceptance: Etiqueta depois de enviar, in the animal card and in Hoje", …);
test("accepted with label LV-2026-AB12CD-0001 shows it in Hoje after the screen was closed", …);
test("duplicate on resend keeps the same label", …);
test("the Q42 restart keeps the label; the next day's purge takes it", …);
test("dpp_result and dpp_on go together or not at all", …);
test("discarding the collection deletes its local line", …);
test.each([["survey_not_open", "leish_survey_id"], ["household_not_in_survey", "leish_survey_id"], ["date_out_of_range", "collected_on"]])(
  "server refusal %s at %s shows its phrase; Descartar removes item and line", …);
```
- [ ] **Step 2–4:** `bin/test campo -w campo-mod35b src/features/animals src/sync/dispatch.test.ts src/sync/retention.test.ts src/features/visits/TodayScreen.test.tsx` FAIL → implementar → PASS.
- [ ] **Step 5: Commit** — `feat: collect leishmaniasis blood samples offline and show the label after acceptance`.

---

### Task 9: Descarte do animal e "precisa de atenção" dos itens novos

O `doomedBy` (`src/sync/outbox.ts:65`) já leva o que cita o item descartado, e o `dependentsWarning` (`src/features/outbox/OutboxScreen.tsx:22`) já avisa. Esta task prova a cadeia do §12 com os itens reais e fecha as pontas das telas.

**Files:**
- Modify: `src/features/attention/AttentionScreen.tsx` (o `hasForm` de `animal` abre o `AnimalForm`), `src/sync/outbox.ts` (o `forget` da criação do animal, se a Task 6 não fechou todos os casos)
- Test: `src/features/outbox/discard.test.ts`, `src/features/attention/AttentionScreen.test.tsx`, `src/sync/dispatch.test.ts`

**Interfaces:**
- Consumes: os `save*` das Tasks 6–8.
- Produces: nada além do comportamento que os testes abaixo fixam.

- [ ] **Step 1: Escrever os testes**

```ts
test("discardImpact of a new animal counts its vaccination, collar and collection: 3", …);
test("discarding the animal removes the four items, the local animal row and the collection line", …);
test("discarding a vaccination leaves the animal", …);
test("microchip_taken: Corrigir opens the animal form; the attention text never shows the number", …);
test("animal in needs_attention after stale_version: Corrigir opens the form on the current version and resends the same client_uuid", …);
test("feature_disabled keeps the four items pending; turning zoonoses back on sends them", …);
test("animal refused: vaccination, collar and collection become blocked and return when the animal is resent", …);
```
- [ ] **Step 2–4:** `bin/test campo -w campo-mod35b src/features/outbox src/features/attention src/sync/dispatch.test.ts` FAIL → implementar → PASS.
- [ ] **Step 5: Commit** — `feat: discard an animal with what depends on it and fix its refusals`.

---

### Task 10: "Coleiras a trocar", lista própria (Q23)

A coleira **não** entra em "Imóveis a visitar". O `vectorDue.ts`, o `dueData.ts` e o `DueScreen.tsx` do 35a não mudam.

**Files:**
- Create: `src/features/animals/collars.ts`, `src/features/animals/CollarsDueScreen.tsx`, `app/(main)/animals/collars.tsx`
- Modify: `src/features/home/homeData.ts`, `src/features/home/HomeScreen.tsx`, `app/(main)/index.tsx`
- Test: `src/features/animals/collars.test.ts`, `src/features/animals/CollarsDueScreen.test.tsx`, `src/features/home/HomeScreen.test.tsx`

**Interfaces:**
- Consumes: `readAnimals` (Task 6, com a fila por cima, Task 7); `canUseZoonoses` (Task 4); `cityTodayOf`; `addressShort` e `listHouseholds` (`src/features/households/view.ts`).
- Produces (`collars.ts`):
  - `COLLAR_DUE_DAYS = 15`;
  - `loadCollarsDue(db, now): Promise<{ animalKey: string; animalName: string | null; householdKey: string; address: string; dueOn: string }[]>`:
    - lista os cães ativos dos domicílios da sessão com `collarDueOn ≤ today + 15 dias`, inclusive os vencidos;
    - ordena por `dueOn` e depois pela chave;
    - sem `canUseZoonoses`, devolve vazio;
    - não traz o nome do tutor.
  - `COLLARS_TABLES = ["animals", "outbox", "households", "reference", "session"]`.
- Tela:
  - O Início mostra "Coleiras a trocar: N" e o botão "Coleiras a trocar", só com `canUseZoonoses`. É a área de animais no tablet.
  - Cada linha traz o endereço curto, o nome do animal e "vence em dd/mm/aaaa" (rótulos do dashboard, Q24). Ela abre o detalhe do domicílio na seção "Animais".

- [ ] **Step 1: Escrever os testes**

```ts
test("dog with collar due in 15 days is listed; due in 16 is not; overdue still is", …);
test("a queued collar takes the dog out of the list", …);
test("cats and inactive animals never appear", …);
test("515105 and 515140 with zoonoses see the list; vector_control off changes nothing; another CBO sees nothing", …);
test("Imóveis a visitar is untouched: a dog with collar due adds nothing to loadDue", …);
// HomeScreen.test.tsx: contador e botão só com canUseZoonoses
```
- [ ] **Step 2–4:** `bin/test campo -w campo-mod35b src/features/animals/collars.test.ts src/features/animals/CollarsDueScreen.test.tsx src/features/home src/features/vector` FAIL → implementar → PASS.
- [ ] **Step 5: Commit** — `feat: list the collars due for replacement in the animals area`.

---

### Task 11: Servidor falso e fluxos do Maestro

**Files:**
- Modify: `scripts/fake-field-server/{state.mjs,server.mjs,server.test.ts}`, `maestro/README.md`, `eslint.config.js`
- Create: `scripts/fake-field-server/fixtures/zoonoses-products.json`, `maestro/{animal-register-offline,rabies-vaccination-offline,leish-collection-offline,collar-offline}.yaml`, `maestro/scripts/expect-zoonoses.js`

**Interfaces:**
- Produces (servidor falso):
  - `POST /__script { "zoonoses": true }` vale até o `__reset` e combina com `profile` (`acs` ou `ace`). A sessão ganha `features.zoonoses: true`.
  - `/reference` ganha `zoonoses_domains`, montado do exemplo da tag e de `zoonoses-products.json` (uma vacina e uma coleira, falsas). A `open_campaign` sai aberta hoje, e o bloco é validado contra o esquema antes de sair.
  - `/changes` traz os animais do exemplo da tag, levados ao domicílio do perfil:
    - um cão sai com `collar_due_on` = hoje + 5;
    - no perfil `ace`, há um inquérito com o domicílio em `household_ids`.
  - `/uploads`:
    - `animal` volta com `id` e `version`;
    - `animal_client_uuid` e o tutor citado (pela forma que a N1 decidir) são resolvidos pelo livro (`CITES`);
    - `leish_collection` volta com `label` `LV-2026-<6 HEX do inquérito>-0001` ou `LV-2026-AVUL-0001`, e o mesmo `label` no `duplicate`;
    - `rabies_vaccination` e `collar` voltam `accepted`;
    - `next_upload: "rejected:microchip_taken"` aponta `microchip`;
    - `next_upload: "stale_version"` também vale para `animal`, com `current` e `current_version`.
  - `GET /__state` ganha `zoonoses: { animals, animals_with_tutor, vaccinations, collections, collars }`.
- Fluxos. Todos em ASCII, e cada um começa por `scripts/server.js` e pelo `common/pair.yaml` ou `common/pair-ace.yaml`.

| Fluxo | Perfil | Passos | O script confere |
|---|---|---|---|
| `animal-register-offline` | ACS + zoonoses | modo avião; no domicílio do download, "Cadastrar animal": cão "Rex", raça "SRD", tutor = um morador do download; "Pendentes de envio: 1"; rede; sincroniza; zero | `animals = 1` e `animals_with_tutor = 1` |
| `rabies-vaccination-offline` | ACS + zoonoses | modo avião; o cão do download; "Vacinar", sem evento adverso; rede; sincroniza | `vaccinations = 1` |
| `leish-collection-offline` | ACE + zoonoses | modo avião; o cão; "Coletar sangue (LV)" no inquérito do download, TR-DPP não reagente; "Etiqueta depois de enviar" visível; rede; sincroniza; `LV-2026-` visível em Hoje | `collections = 1` |
| `collar-offline` | ACE + zoonoses | Início, "Coleiras a trocar" com o cão; abre o domicílio; "Colocar coleira"; rede; sincroniza; a lista fica sem o cão | `collars = 1` |

- [ ] **Step 1: Testes do servidor falso** (`server.test.ts`, no `bin/test`):
  - o perfil de zoonoses dá sessão, referência e `/changes` válidos pelos esquemas da v1.4.0;
  - animal + vacina citando `animal_client_uuid` no mesmo lote → dois `accepted`;
  - citação desconhecida → `rejected not_found` com `path` `animal_client_uuid`;
  - o `label` se repete no `duplicate`;
  - o `__reset` desliga o `zoonoses`.
  - Run: `bin/test campo -w campo-mod35b scripts` · FAIL → implementar → PASS.
- [ ] **Step 2: Escrever os quatro fluxos** com as regras do `maestro/README.md`: rolar antes do último botão, opção de busca com `index: 1` e só ASCII. A linha de cada fluxo entra na tabela do README, e o servidor falso ganha a seção "Zoonoses (35b)".
- [ ] **Step 3: Lint como a CI** — `npx eslint .` e `npx tsc --noEmit` no worktree · Expected: sem erro.
- [ ] **Step 4: Commit** — `test: add Maestro flows for animals, rabies vaccination, leishmaniasis collection and collars`.

---

### Task 12: Prova no emulador contra o api 3061

**Exige autorização do usuário.** A prova precisa do build Android, do emulador, do api, do dashboard e do maintenance de dev do 35b de pé, e muda o interruptor no dev de Curitiba. Sem a autorização, pare e relate o que está pronto. O ambiente está na seção "Ambiente de execução".

**Files:** nenhum código; relato ao coordenador.

- [ ] **Step 1: APK e2e release.**
  - Chave descartável no scratchpad.
  - `EXPO_PUBLIC_E2E=1 EXPO_PUBLIC_DEV_ORIGIN=http://10.0.2.2:8787 CAMPO_ALLOWED_HOST_SUFFIXES=.localhost,.test`, exportadas no shell do prebuild **e** no do Gradle.
  - `node scripts/check-apk.mjs <apk> --e2e` · Expected: exit 0.
- [ ] **Step 2: Os 19 fluxos contra o servidor falso** — `caffeinate -i` com o laço do `maestro/README.md` (os 15 de antes e os 4 novos) · Expected: todos `PASSED`. Um fluxo antigo vermelho é regressão desta branch: volte à task dona.
- [ ] **Step 3: APK contra o api 3061** — refaça o build com `EXPO_PUBLIC_DEV_ORIGIN=http://10.0.2.2:3061`. Antes, `curl -s -H 'Host: curitiba.localhost' http://localhost:3061/up`.
- [ ] **Step 4: Ordem da Q6 do 35a** (§11, §17 passo 9 e Q24):
  1. no maintenance (5177), **desligue** o `zoonoses` de Curitiba;
  2. instale o APK e pareie o `ace@curitiba.demo` pelo QR do dashboard (5221, Aparelhos de campo). O QR vai por `-e QR_PAYLOAD=…`, nunca em arquivo. Confira que não há "Animais";
  3. **religue** o `zoonoses`;
  4. na rodada seguinte, confira o `reset: true` (log `sync` com a página `reset`) e os animais da área do `ace@`.
- [ ] **Step 5: Casos sem sinal** (§17, passo 9):
  1. **Animal novo com tutor novo no mesmo lote.** É caso do ACS (Q22) e depende da N1. Pareie o ACS da semente do 24c e, em modo avião, cadastre um morador novo e um animal com esse tutor. Volte a rede e confira, no dashboard (`zoonoses@`, `zoo-animals`), o animal com o tutor. Sem a decisão da N1, registre o caso como pendente e faça só o animal com tutor já baixado.
  2. **Vacinação** na "Campanha antirrábica 2026", pelo `ace@`. Confira a dose `house` no resultado semanal (`zoo-campaigns`).
  3. **Coleta** do cão de um domicílio do inquérito `open` da semente. A tela mostra "Etiqueta depois de enviar" até o aceite e depois a etiqueta `LV-…` em Hoje, que tem de ser igual à do `zoo-leish`.
  4. **Coleira com falta de estoque, aceita.** Se o lote da coleira da semente tiver saldo, zere-o antes por um ajuste na tela de estoque do 35a (`stock_keeper`), com o usuário. Confira a coleira aceita e a falta com `source` `collar` na tela de faltas (Q17).
  5. **Coleira a trocar:** a coleira vencendo da semente aparece em "Coleiras a trocar" e sai depois da troca.
  6. **Desligar de novo:** chega o `reset`, as telas somem e a vacina feita offline espera na fila. Religar envia.
  - Evidência: com `FLAG_SECURE`, a captura sai preta. Descreva o que viu, ou use `CAMPO_E2E_SCREEN_CAPTURE=1` só para a evidência.
- [ ] **Step 6: Limpeza** — apague a chave e os APKs do scratchpad. Os registros fictícios ficam no dev de Curitiba; diga quais no relato.
- [ ] **Step 7: Relato** ao coordenador: os casos, o resultado de cada um, o que ficou de fora e as divergências novas.

---

### Task 13: Revisão final (para antes do merge)

**Files:** nenhum código novo, salvo as correções da revisão.

- [ ] **Step 1: Suíte do app** — `bin/test campo -w campo-mod35b` · Expected: `resultado: ok`, com total = linha de base da Task 0 + os novos.
- [ ] **Step 2: Comandos da CI** — `npx eslint .`, `npx tsc --noEmit` e `node scripts/contract-types.mjs --check` · Expected: sem erro e "6 files up to date".
- [ ] **Step 3: Revisão** — `superpowers:requesting-code-review` sobre `origin/main..feat/mod-35b-zoonoses`, com este plano e o contrato. Cada correção roda `bin/test campo -w campo-mod35b <arquivos>` de escopo fechado e vai no seu commit.
- [ ] **Step 4: Relato e parada.**
  - Relato ao coordenador: commits, testes, provas e divergências.
  - O card fica em In Progress.
  - **Push, PR e merge só com autorização do usuário.** Este plano termina aqui.

---

## Self-review

- **Cobertura do contrato:**

  | Parte do contrato | Onde |
  |---|---|
  | §12, esquema v3 e migração 2→3 | Task 2 |
  | §12, quem usa (Q10) | Task 4 |
  | §12, telas do domicílio | Tasks 6, 7 e 8 |
  | §12, coleta com "Etiqueta depois de enviar" (Q21) | Task 8 |
  | §12, "Coleiras a trocar" (Q23) | Task 10 |
  | §12, fila e `discardImpact` com aviso | Task 9 |
  | §12, desligado | Tasks 3, 9 e 12 |
  | §12, os quatro fluxos do Maestro | Task 11 |
  | §11 e Q20, tipos, formas e códigos | Tasks 1 e 5 |
  | §11 e Q21, download e remoções | Task 3 |
  | §11, referência | Task 4 |
  | §11, falta de estoque aceita | Tasks 7 e 12 |
  | §11, compatibilidade e `reset` | Tasks 3 e 12 |
  | §2, Q20 e Q22, tutor, `community` e microchip | Tasks 6 e 9 |
  | Q20, `stale_version` sem mescla | Tasks 6 e 9 |
  | §3 e Q19, vacina `house`, evento adverso e segunda dose pelo api | Task 7 |
  | §6, coleta só em cão ativo e coleira com `collar_months` | Tasks 7 e 8 |
  | §16, microchip e tutor | Tasks 3 e 6, `noLeak` |
  | §17, passo 9 | Task 12 |

- **Tipos:**
  - `ZoonosesDomains` (Task 4) é usado nas Tasks 5–10;
  - `AnimalRow` e `zoonosesRepo` (Task 3), nas 6–10;
  - `AnimalView` e `readAnimals` (Task 6), nas 7, 8 e 10;
  - `ItemType` com os quatro tipos (Task 1), no `dispatch` (Tasks 6–8) e no `ITEM_TYPE_LABEL`.
- **Vermelho de propósito:** só entre as Tasks 1 e 5 (frase dos códigos novos), anotado nas duas.
- **Dependências externas:** tag (Task 1), `zoonosesErrors.ts` do dashboard (Task 5), api do 35b (Task 12) e N1 (Task 6, tutor nascido no tablet; Task 12, caso 1).

## Divergências propostas ao contrato

| # | Achado | Decisão | Onde o plano aplica |
|---|---|---|---|
| D1 | Sem `vector_control`, o tablet não sabe o quarteirão do domicílio (`changes-response-v1.json:174`). | **Resolvida pela Q21:** o `leish_survey` traz `household_ids` no lugar de `city_block_ids`. Entra o código `household_not_in_survey` (Q20). | Tasks 3 e 8 |
| D2 | O inquérito fechado nunca saía do tablet: sem `stage` e sem remoção (`changes-response-v1.json:328`). | **Resolvida pela Q21:** `removals` de `leish_survey` (e de `animal`) com `out_of_scope`. | Task 3 |
| D3 | "Como a amostra do 35a" estava errado (`src/rules/vectorInspection.ts:268`), e faltava tabela local. | **Resolvida pela Q21:** a etiqueta é do api, vem no `accepted` e no `duplicate` em hex maiúsculo, e o app guarda em `local_leish_collections`. | Tasks 2 e 8 |
| D4 | O `accepted` do `animal` não podia trazer `version` (`upload-response-v1.json:221`). | **Resolvida pela Q20:** o `accepted` do `animal` traz `version`. | Task 6 |
| D5 | Não se dizia o que volta numa edição de animal com versão vencida (`src/sync/stale.ts:35`). | **Resolvida pela Q20:** `stale_version` com `current_version`, como no 24c, sem mescla. | Tasks 6 e 9 |
| D6 | Formas do envio não fixadas: id da edição, mês do nascimento, `dpp_on` e campos editáveis. | **Resolvida pela Q20:** corpo inteiro; `animal_id` com `version` na edição; `status` só na edição, sem volta a `active`; `birth_estimate_month`; `dpp_result` e `dpp_on` juntos; `adverse_note` só com `adverse_event`. | Tasks 6, 7 e 8 |
| D7 | Não se sabia o que o `tutor_client_uuid` cita, nem quem faz o caso do §17.9. | **Resolvida pela Q22:** cita o item `patient` do 24c; só o ACS cria morador, e o 515140 escolhe entre os moradores que o tablet já tem. A forma do ACS segue aberta na **N1**. | Tasks 6 e 12 |
| D8 | O animal do domicílio sem tutor tinha de ser comunitário (§2)? | **Resolvida pela Q22:** com domicílio, o animal pode ficar sem tutor e sem `community`; `community: true` quer dizer que ninguém responde pelo animal, e o tutor é proibido junto com ele (Q20). | Tasks 5 e 6 |
| D9 | A folha do 35a só abre com `canRecordInspection` e tem um motivo por imóvel (`dueData.ts:59`, `vectorDue.ts:19`). | **Resolvida pela Q23:** "Coleiras a trocar" é lista própria na área de animais; "Imóveis a visitar" não muda. | Task 10 |
| D10 | Não se dizia qual gate o `features.zoonoses` reflete. | **Resolvida pela Q10:** `field_usable?`. | Task 4 |
| D11 | As chaves de `limits` e os limites do envio do `animal` não tinham nome. | **Resolvida pela Q20:** rótulos, chaves de `limits`, tipo de inquérito e finalidade do produto ficam na seção `zoonoses` do `domains-v1.json`. | Tasks 1, 4 e 5 |
| D12 | Não havia fonte para as frases do campo. | **Resolvida pela Q24:** `ZOONOSES_ERRORS` em `apps/dashboard/src/lib/zoonosesErrors.ts`; a Task 5 espera esse commit. | Task 5 |
| D13 | O feed não traz a campanha da última vacina. | **Resolvida pela Q19:** a segunda dose é barrada só pelo api, e isso fica documentado. | Task 7 |
| D14 | O "Q6" do §12 e do §17 era ambíguo. | **Resolvida pela Q24:** é a Q6 do 35a. | Task 12 |
| N1 | **Nova.** O morador novo do ACS nasce no tablet como `individual_registration` com `new_patient`, sem item `patient` (`src/features/registrations/save.ts:83` e `:106`). O item `patient` só nasce na visita a uma pessoa nova (`src/features/visits/save.ts:91`), e essa pessoa não ganha linha em `registrations`. Pela Q22 ao pé da letra, o tutor novo no mesmo lote do §17.9 não tem item `patient` para citar. Pior: o morador já aceito, mas ainda não baixado, também não serve de tutor, porque o tablet não sabe o `patient.id` dele. | Proposta: o `tutor_client_uuid` cita um item `patient` **ou** um `individual_registration` com `new_patient` aceito antes, no mesmo lote ou em outro, como o `patient_client_uuid` da Q26 do 24c. A falta → `rejected not_found` `path` `tutor_client_uuid`. Sem isso, a Task 6 oferece como tutor só o morador com `patient.id`, e o caso 1 do §17.9 sai da prova. | Tasks 6, 11 e 12 |
