# Módulo 35a — App de campo (`rotasaude/campo`): boletim, ovitrampa, LIRAa e bloqueio offline — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Reconciliado com o §20 do contrato até a Q13 (docs c2d755c).** As D1–D15 foram decididas (Q6, Q7, Q10, Q11) e a N1 virou a Q13(4); o plano já as aplica, e a coluna "Decisão" da tabela do fim diz onde. A Q13(1) (imóvel não residencial pelo agente) é a Task 9b; a Q13(2) (`path` da ovitrampa) está na Task 9. Nada aberto.

**Goal:** Dar ao agente de combate às endemias (CBO `515140`) no app de campo o boletim na visita, a folha "Imóveis a visitar" com LIRAa e bloqueio, e a instalação e coleta de ovitrampa, tudo offline pela fila do 24c, sobre o `field-sync` v1.3.0 (F-35.10).

**Architecture:** O app segue o 24c sem mudar a mecânica: tipos gerados da cópia presa da tag `field-sync-v1.3.0`; esquema local v2 (quatro tabelas novas, migração 1→2 que não apaga linha e zera o cursor); download que guarda as entidades novas e **ignora tipo desconhecido**; regras do boletim e da ovitrampa portadas do dashboard (Q76 do 24c, Q9/Q11 do 35a); folha calculada no aparelho com a regra do api; envio pelos itens `home_visit` (com `vector_inspection`), `ovitrap_installation` e `ovitrap_collection`. Quem decide é sempre o api.

**Tech Stack:** o do 24c, sem dependência nova — Expo SDK 58.0.25, React Native 0.88.1, TypeScript estrito, Expo Router (`root: "app"`), `expo-sqlite` com SQLCipher 4.17.0, Jest 30 + `jest-expo` + RNTL + `sql.js` + Ajv 2020 (`multipleOfPrecision: 8`), Maestro 2.x, Gradle local.

**Spec:** `docs/superpowers/specs/2026-11-14-module-35-zoonoses-endemics-design.md` · **Contrato (fonte única):** `docs/superpowers/plans/2026-11-16-module-35a-vector-control-contracts.md` (§1, §3, §5, §6, §9, §13, §14, §17; o §20 prevalece) · **Base do app:** contrato do 24c `2026-11-13-module-24c-field-app-contracts.md` (§13, até a Q86) e o plano `2026-11-13-module-24c-field-app-campo.md` · **Origem do porte (Q9, Q11):** plano `2026-11-16-module-35a-vector-control-dashboard.md` — Task 2 (`apps/dashboard/src/lib/vectorErrors.ts` e `src/lib/vectorInspection.ts`) e, para a ovitrampa, Task 3 (`installProblems` e `collectionProblems` em `src/lib/vectorControl.ts`).

## Global Constraints

- Repositório `rotasaude/campo`, base `main` em `8429bd5`; branch `feat/mod-35a-vector-control`; worktree `rota-saude/.worktrees/campo-mod35a` (nunca `node_modules` dentro de `apps/<outro app>`).
- Contrato: tag `field-sync-v1.3.0` (MINOR); `session.contract` continua `"field-sync-v1"`. Cópia em `contracts/field-sync/` presa por `SHA256SUMS`, `TAG` e os seis SHA-256 que o api fixa em `FieldSync::Schemas::SHA256`.
- Nada fica obrigatório: `features.vector_control` ausente = `false`; `vector_control_domains` ausente = sem boletim; `<household>.vector` **ausente** = gate desligado na cidade, **`null`** = imóvel sem perfil (Q6). Ligar ou desligar o interruptor chega como página `reset: true` (Q6), que o app trata como qualquer reset.
- O boletim aparece só com `session.features.vector_control === true`, `session.professional.cbo === ENDEMIC_AGENT_CBO` (`"515140"`) e `reference.vectorDomains !== null`. O ACS e os outros CBOs não veem nada novo; o "Quem visitar" do 24b não muda. O agente pode salvar a visita sem boletim: chave "Registrar boletim", ligada por padrão (Q8, como o dashboard).
- Cadastro pelo agente de endemias (Q13(1)): com `features.vector_control`, `features.territory_registry` e CBO `515140`, o app deixa criar e editar domicílio **só** dos tipos não residenciais `2 3 4 5 6 12 13`, sem família nem morador, nas microáreas da sessão (as atribuídas a ele). Tipo residencial, família, cadastro individual e troca para tipo residencial seguem só do ACS (`canEditRegistry` não muda); o servidor responde `rejected cbo_not_allowed` ao resto.
- Esquema local: `SCHEMA_VERSION = 2`; versão desconhecida continua indo para `UnknownSchemaError`; a migração nunca apaga linha e zera `meta.cursor` (Q6).
- Banco (Q75 do 24c): toda transação é `db.tx(fn)` (`BEGIN IMMEDIATE` na conexão única, fila `src/db/serial.ts`); dentro do `fn`, só o `tx` recebido; nunca `withExclusiveTransactionAsync`.
- Regras de tela (Q76 do 24c; Q9 e Q11 do 35a): porte do dashboard, com o commit de origem no cabeçalho, sem regra nova. Nomes, `path` e frases são os da Task 2 do plano do dashboard: as regras devolvem `{ error, path }` (`InspectionIssue`) e a tela mostra `VECTOR_ERRORS[error]`. **O campo não redige frase:** copia a tabela `VECTOR_ERRORS` (com `invalid_transition` no lugar de `status_changed` e `sample_label_taken`, Q9). O que faltar no dashboard vai ao coordenador.
- Etiqueta da amostra (§9): `sampleLabel(clientUuid, n) = clientUuid.slice(0, 8).toUpperCase() + "-" + String(n).padStart(2, "0")`, com o `client_uuid` da visita fixado ao abrir o formulário. No app a etiqueta é definitiva desde a tela (a visita é offline); só muda por "Gerar etiquetas novas" depois de `sample_label_taken` (Q7(1)).
- Limites da referência (Q7(3)): `limits.deposit_count_max` 999, `charges_max` 999, `samples_max` 99, `larvae_collected_max` 10, `measures_max` 20; no máximo 7 grupos de depósito. Número da ovitrampa `^[0-9A-Za-z-]{1,12}$` (Q7(6)).
- Semana epidemiológica, onde aparecer, é o texto `"YYYY-WW"` do api (Q10); o app não a calcula (o feed de `ovitrap` não a traz).
- Sigilo (§17): nada do 35a mostra, grava ou loga paciente; `transmission_block` nunca traz caso; `log()` só com `client_uuid`, tipo, código, `seq`.
- Testes: só `bin/test campo -w campo-mod35a [args do jest]`. Lint e tipos pelo comando da CI (`npx eslint .`, `npx tsc --noEmit`), que não são teste. Maestro fica fora do `bin/test`.
- Git: `/opt/homebrew/bin/git`; nunca `git add -A`/`git add .`; Conventional Commits em inglês, tipo por extenso; toda mensagem termina com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Forma de todo commit:
  `G=/opt/homebrew/bin/git; W=/Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/campo-mod35a; $G -C $W add <arquivos>; $G -C $W commit -m "<tipo>: <descrição>" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"`
- Execução coordenada: ao fim de cada task, relato curto à sessão "Próximos módulos" (memória "relato por task ao coordenador"). Divergência nova vai para o fim deste plano e para o coordenador; só ele escreve no contrato.
- Push, PR e merge só com autorização do usuário; a Task 13 para antes do merge.

## Ambiente de execução

- **Dependências de ordem (§19):** tag `field-sync-v1.3.0` publicada (Task 2); api do 35a com o feed e o envio novos commitados (prova da Task 12); Tasks 2 e 3 do dashboard commitadas no worktree `rota-saude/.worktrees/dashboard-mod35a` (Tasks 6 e 9 daqui). Sem um deles, a task que depende **para** e avisa o coordenador.
- **Gate de go-live (Q6(4)):** a cidade só liga `vector_control` com todos os aparelhos no app do 35a; em dev, o APK do 35a vai ao emulador **antes** do `db:seed` que liga o interruptor.
- **Worktree:** `.worktrees/campo-mod35a`, com `npm ci` no host (o Gradle usa esse `node_modules`); o `bin/test` usa volume próprio quando o lock difere.
- **Android** (memórias "emulador Android do campo" e "24c build Android"), só no shell do comando: `ANDROID_HOME=~/Library/Android/sdk`, `JAVA_HOME=/opt/homebrew/opt/openjdk@21/libexec/openjdk.jdk/Contents/Home`; AVD `campo_api34` (`emulator -avd campo_api34 -no-window -no-audio -gpu swiftshader_indirect -no-snapshot-save`, ou `-no-snapshot` se subir sem rota para `10.0.2.2`); build `npx expo prebuild --platform android --clean` + `./gradlew :app:assembleRelease -PreactNativeArchitectures=arm64-v8a`. Offline e Maestro **só com APK release** (JS embutido). Rodada longa sob `caffeinate -i`; nunca `adb root` com fluxo rodando; rolar até o fim antes do último botão (barra de tarefas do tablet). Variável nova em `maestro/scripts/` entra nas globais do `eslint.config.js`.
- **Dev do 35a:** api em `10.0.2.2:3059` (porta do contrato), dashboard em `5220`; contas `ace@curitiba.demo`, `endemias@curitiba.demo`, `entomologia@curitiba.demo` (senha e TOTP da semente).

## Review Focus

1. **Etiqueta escrita no tubito que muda depois** (amostra apagada no meio, `client_uuid` trocado sem aviso, formulário reaberto): a etiqueta mostrada é a que vai ao api. Só a última amostra pode ser apagada; o `client_uuid` é fixado ao abrir e é o que `saveVisit` grava; a troca só acontece pela ação explícita da Task 10, que mostra as etiquetas novas. Testes nas Tasks 6, 7 e 10.
2. **Horários na borda** (entrada com segundos depois do `occurred_at` em minutos, visita aberta às 23h55 e salva depois da meia-noite, relógio do tablet errado): entrada e saída usam o relógio da cidade corrigido; a entrada padrão é o mesmo minuto do `occurred_at` (Q11); fora do dia ou acima de 12 h a tela recusa antes de gravar. Testes nas Tasks 6 e 7.
3. **Sessão e referência desencontradas** (interruptor ligado sem `vector_control_domains`, produto desativado depois de a visita esperar na fila, CBO trocado): sem domínios o boletim some com aviso; a recusa do servidor vira "precisa de atenção" com a frase, sem perder a visita. Testes nas Tasks 5 e 7.
4. **Tablet com fila v1 atualizado no meio do trabalho, e página com tipo que o app não conhece**: a migração 1→2 guarda fila e território e zera o cursor (Q6), sem prender o tablet que já baixou à espera de Wi-Fi (Q13(4)); o download pula o tipo desconhecido em vez de travar o cursor. Testes nas Tasks 1 e 3.
5. **Cadeia offline da ovitrampa** (instalar e coletar antes de sincronizar, duas coletas na fila, instalação recusada): a coleta cita a instalação e espera por ela; a recusa da instalação bloqueia a coleta; a tela mostra a palheta aberta pela fila. Testes na Task 9.

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| `src/sync/download.ts` | `applyPage` com as entidades novas e o descarte de tipo desconhecido | 1, 4 |
| `contracts/field-sync/**`, `scripts/contract-types.mjs`, `src/contract/{index.ts,generated/*}`, `src/contract/contract.sha.test.ts` | cópia presa da v1.3.0 e tipos | 2 |
| `src/db/migrations.ts`, `src/db/testing/schemaV1.ts` | esquema v2 e migração 1→2 | 3 |
| `src/sync/round.ts` | carga inicial = nunca terminou download (Q13(4)) | 3 |
| `src/security/wipe.ts` | `wipeTerritory` cobre as tabelas novas | 3 |
| `src/db/repos/vectorRepo.ts` | `city_blocks`, `ovitraps`, `liraa_items`, `transmission_blocks` | 4 |
| `src/db/repos/territoryRepo.ts` | cascata de `removeHousehold` | 4 |
| `src/sync/reference.ts`, `src/rules/vectorDomains.ts` | `vector_control_domains` | 5 |
| `src/features/session.ts` | `canRecordInspection` | 5 |
| `src/rules/vectorErrors.ts`, `src/rules/vectorInspection.ts` | porte do `vectorErrors.ts` e do `vectorInspection.ts` do dashboard | 6 |
| `src/features/messages/errorPhrases.ts` | frases dos códigos novos (de `VECTOR_ERRORS`) | 6 |
| `src/features/visits/{InspectionSection.tsx,VisitForm.tsx,save.ts}`, `src/ui/counter.tsx`, `src/features/vector/profile.ts` | boletim na visita | 7 |
| `src/rules/vectorDue.ts`, `src/features/vector/{dueData.ts,DueScreen.tsx}`, `app/(main)/vector/due.tsx`, `src/features/home/*` | folha "Imóveis a visitar" | 8 |
| `src/rules/ovitrap.ts`, `src/features/ovitraps/{save,view}.ts`, `src/features/ovitraps/{OvitrapList,InstallForm,CollectForm}.tsx`, `app/(main)/ovitraps/**`, `src/sync/dispatch.ts`, `src/features/attention/AttentionScreen.tsx`, `src/features/households/HouseholdScreen.tsx` | ovitrampa | 9 |
| `src/features/session.ts`, `src/features/households/{HouseholdForm,HouseholdList,HouseholdScreen}.tsx`, `src/features/households/save.ts` | imóvel não residencial pelo agente (Q13(1)) | 9b |
| `src/features/visits/relabel.ts`, `src/features/attention/AttentionScreen.tsx` | etiqueta repetida (`sample_label_taken`) | 10 |
| `scripts/fake-field-server/**`, `maestro/{vector-inspection-offline,ovitrap-install,ovitrap-collect,strategic-point-offline}.yaml`, `maestro/README.md`, `eslint.config.js` | Maestro | 11 |

## Esquema local v2 (acréscimos ao v1)

| Tabela | Colunas (`NOT NULL` salvo `?`) |
|---|---|
| `city_blocks` | `id` PK, `neighborhood_id`, `number`, `active` (`0/1`), `version`, `body` (forma `city_block` do download) |
| `ovitraps` | `key` PK (id do servidor ou `client_uuid` da instalação feita no tablet), `id?` único, `client_uuid?`, `household_key`, `number`, `status` (`active`\|`removed`\|`lost`), `version?`, `body` (forma `ovitrap`, ou sintetizada na instalação) |
| `liraa_items` | `id` PK, `survey_id`, `household_id` (id do servidor), `city_block_id`, `status`, `version`, `body` |
| `transmission_blocks` | `id` PK, `status`, `version`, `body` |

Índices: `idx_city_blocks_neighborhood (neighborhood_id)`, `idx_ovitraps_household (household_key)`, `idx_liraa_items_household (household_id)`. O perfil do imóvel (`vector`) fica no `households.body`, sem coluna (§14). `local_visits.summary` ganha `vector?: { activity, situation, city_block_id?, risk_class? }` (Q11).

---

### Task 0: Base, card e worktree

**Files:** nenhum código.

- [ ] **Step 1: Conferir a base** — `G=/opt/homebrew/bin/git; $G -C /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/campo fetch origin && $G -C /Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/campo rev-parse --short origin/main`. Expected: `8429bd5`. Outro commit: anote e avise o coordenador antes de seguir (o plano foi escrito sobre `8429bd5`).
- [ ] **Step 2: Card** — card F-35.10 do board #1 (mod-35) para In Progress, um `gh` por comando, começando por `gh` (memória "board: gh em comando simples"): `gh project item-list 1 --owner rotasaude --format json --limit 500` para achar o id, depois `gh project item-edit …` com o campo Status. Se o card já estiver em In Progress por outra sessão, não mexa.
- [ ] **Step 3: Worktree**

```bash
G=/opt/homebrew/bin/git; R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
$G -C $R/apps/campo worktree add $R/.worktrees/campo-mod35a -b feat/mod-35a-vector-control origin/main
cd $R/.worktrees/campo-mod35a && npm ci
```
- [ ] **Step 4: Linha de base** — `bin/test campo -w campo-mod35a`. Expected: `resultado: ok` (anote o total de testes para o relato final).

---

### Task 1: Download pula tipo desconhecido (Q6(3))

Na `main` (`8429bd5`), `applyPage` manda todo item que não é `micro_area` nem `household` para `putRegistration`: um `city_block` lança `TypeError` em `body.patient.id`, a transação volta, o cursor não anda e toda rodada falha igual. Esta task corrige isso ainda sobre a v1.2.0, isolada do resto.

**Files:**
- Modify: `src/sync/download.ts`, `src/log.ts` (evento `sync.change.skipped`)
- Test: `src/sync/download.test.ts`

**Interfaces:**
- Produces: `KNOWN_CHANGE_TYPES: readonly string[]` em `download.ts` (nesta task `["micro_area", "household", "registration"]`; a Task 4 acrescenta os novos). Item ou remoção de tipo fora da lista é pulado, com `log("sync.change.skipped", { type })`; o resto da página e o cursor são aplicados.

- [ ] **Step 1: Escrever os testes**

```ts
test("an unknown change type is skipped and the cursor moves on", async () => {
  const page = { ...example<ChangesResponse>("changes-last-page.json"),
    items: [...lastPage.items, { type: "lighthouse", id: UUID_A, version: 1, body: { id: UUID_A } }] } as unknown as ChangesResponse;
  await applyPage(db, page);
  expect(await territoryRepo.getMeta(db, "cursor")).toBe(page.next_cursor);
  expect(await db.get("SELECT key FROM registrations WHERE key = ?", [UUID_A])).toBeNull();
  expect(logged).toContainEqual(["sync.change.skipped", { type: "lighthouse" }]);
});
test("an unknown removal type is skipped", …);       // removals: [{ type: "lighthouse", id, reason: "moved" }] → nada removido, cursor gravado
test("known items of the same page are still applied", …);
```
- [ ] **Step 2: Rodar e ver falhar** — `bin/test campo -w campo-mod35a src/sync/download.test.ts` · Expected: FAIL (`TypeError … patient`).
- [ ] **Step 3: Implementar** — filtrar por `KNOWN_CHANGE_TYPES` antes de ordenar; o despacho vira um `switch` com `default` que nunca grava.
- [ ] **Step 4: Rodar e ver passar** — mesmo comando · Expected: PASS.
- [ ] **Step 5: Commit** — `src/sync/download.ts`, `src/log.ts`, `src/sync/download.test.ts` · `fix: skip change types the app does not know instead of stalling the download`.

---

### Task 2: Contrato `field-sync` v1.3.0 e tipos gerados

Depende da tag `field-sync-v1.3.0`. Sem ela: pare e avise o coordenador.

**Files:**
- Modify: `contracts/field-sync/**` (cópia da tag), `contracts/field-sync/{SHA256SUMS,TAG}`, `scripts/contract-types.mjs` (banner `field-sync-v1.3.0`), `src/contract/generated/*.ts`, `src/contract/index.ts`
- Test: `src/contract/contract.sha.test.ts`, `src/contract/contract.examples.test.ts`, `src/contract/contract.types.test.ts`

**Interfaces:**
- Produces (`src/contract/index.ts`), além do que já existe: `ItemType` passa a incluir `"ovitrap_installation" | "ovitrap_collection"`; `VectorInspectionBody` (o `$defs/vector_inspection` do envio), `OvitrapInstallationItemBody`, `OvitrapCollectionItemBody`; do download `CityBlockBody`, `OvitrapBody`, `LiraaItemBody`, `TransmissionBlockBody`, `HouseholdVector` (`HouseholdBody["vector"]`); `ChangeType = ChangeItem["type"]`; `VectorControlDomainsRaw` (o bloco da referência). Os nomes `$defs` saem do esquema da tag; se o gerador der outro nome, o apelido em `index.ts` mantém estes.

- [ ] **Step 1: Copiar a tag** — o Step 1 da Task 4 do plano do 24c, trocando `field-sync-v1.2.0` por `field-sync-v1.3.0` e `campo-mod24c` por `campo-mod35a`; antes, `rm -rf $W/contracts/field-sync` (para não sobrar arquivo da v1.2.0), extraia de novo e refaça `SHA256SUMS`. Os arquivos removidos entram no commit do Step 6 por `git rm` de cada um.
- [ ] **Step 2: Atualizar os testes**

```ts
// contract.sha.test.ts
test("TAG names field-sync-v1.3.0 and a commit", () => expect(tag).toMatch(/^field-sync-v1\.3\.0 [0-9a-f]{40}$/));
// PINNED: os seis SHA-256 de FieldSync::Schemas::SHA256 no api do 35a (o plano do api ou o relato do contracts-repo os traz)
// contract.examples.test.ts
test("session without vector_control is valid (optional key)", …);       // exemplo da v1.2.0 continua válido
test("home_visit with vector_inspection, ovitrap_installation and ovitrap_collection examples validate", …);
test("vector_inspection is closed: an unknown key is refused", …);        // exemplo inválido da tag
test("reference limits carry the Q7 keys", () => {
  expect(Object.keys(example<ReferenceResponse>("reference.json").vector_control_domains!.limits).sort())
    .toEqual(["charges_max", "deposit_count_max", "larvae_collected_max", "measures_max", "samples_max"]);
});
test("the new item codes are in errors.item", () => {
  expect(KNOWN_ITEM_ERRORS).toEqual(expect.arrayContaining(["situation_outcome_mismatch", "inspection_not_worked", "activity_property_mismatch",
    "liraa_item_not_found", "transmission_block_not_active", "deposit_counts_inconsistent", "product_not_allowed", "samples_exceed_larvae",
    "inspection_times_invalid", "reason_derived", "block_already_set", "household_without_block", "consent_required",
    "ovitrap_number_taken", "collection_out_of_range", "reading_not_open", "ovitrap_not_active", "vector_inspection_invalid", "sample_label_taken"]));
});
```
- [ ] **Step 3: Rodar e ver falhar** — `bin/test campo -w campo-mod35a src/contract` · Expected: FAIL (TAG, PINNED, tipos velhos).
- [ ] **Step 4: Gerar** — `node scripts/contract-types.mjs` no host; acrescente os apelidos em `index.ts`.
- [ ] **Step 5: Rodar e ver passar** — `bin/test campo -w campo-mod35a src/contract` e depois a suíte inteira (`bin/test campo -w campo-mod35a`), porque `ItemType` e `ChangeType` cresceram e `errorPhrases.test.ts` vai acusar os códigos novos sem frase · Expected: `src/contract` PASS; a suíte só com o `every known http and item code has its own phrase` vermelho — **vermelho de propósito até a Task 6** (anote no relato). Qualquer outro vermelho: corrija aqui (tipicamente um `switch` exaustivo sobre `ItemType`).
- [ ] **Step 6: Commit** — `contracts/field-sync` (lista de `$G -C $W status --porcelain contracts`), `scripts/contract-types.mjs`, `src/contract/*` · `feat: pin the field-sync v1.3.0 contract and regenerate its types`.

---

### Task 3: Esquema local v2 e migração 1→2

**Files:**
- Modify: `src/db/migrations.ts`, `src/security/wipe.ts`, `src/sync/round.ts` (Q13(4))
- Create: `src/db/testing/schemaV1.ts` (os `CREATE` do v1 congelados de `8429bd5`, só para teste)
- Test: `src/db/migrations.test.ts`, `src/security/wipe.test.ts`, `src/sync/round.test.ts`

**Interfaces:**
- Produces: `SCHEMA_VERSION = 2`; `migrate(db)` — cria o v1 (idempotente), grava `schema_version` = `SCHEMA_VERSION` só se ausente, e então: `"1"` → cria as tabelas e índices do v2, **apaga `meta.cursor`** (Q6: o próximo download traz as entidades novas e o `vector` de domicílios anteriores ao cursor) e grava `"2"`, tudo na mesma transação; `"2"` → cria o v2 (idempotente); outro → `UnknownSchemaError` e nada muda (a transação volta). `last_download_ok_at` fica: a retenção de 30 dias não recomeça.
- Produces (Q13(4), `round.ts`): a "carga inicial" que espera Wi-Fi passa a ser `cursor === null && last_download_ok_at === null` (nunca terminou um download). O tablet recém-instalado e o que perdeu o território pelos 30 dias continuam esperando Wi-Fi; o migrado baixa em qualquer rede.
- Produces: `wipeTerritory` passa a apagar `city_blocks`, `liraa_items`, `transmission_blocks` e as `ovitraps` que a fila não cita (`client_uuid IS NULL OR client_uuid NOT IN (SELECT client_uuid FROM outbox)`, e as que não são `body.ovitrap_client_uuid` de coleta na fila).

- [ ] **Step 1: Escrever os testes**

```ts
test("creates every table and index of the local schema v2", …);   // TABLES + city_blocks, liraa_items, ovitraps, transmission_blocks; INDEXES + os três
test("upgrades a v1 database with data to v2, keeps everything and resets the cursor (Q6)", async () => {
  const db = await createEmptyTestDb();
  await db.tx(async (tx) => { for (const s of SCHEMA_V1) await tx.exec(s); });
  await db.run("INSERT INTO meta (key, value) VALUES ('schema_version', '1'), ('cursor', 'c1'), ('last_download_ok_at', '2026-11-15T10:00:00Z')");
  await seedV1(db);   // um domicílio, um cadastro, dois itens na fila (um pending, um needs_attention), uma visita local
  await migrate(db);
  expect(await db.get("SELECT value FROM meta WHERE key = 'schema_version'")).toEqual({ value: "2" });
  expect(await db.get("SELECT value FROM meta WHERE key = 'cursor'")).toBeNull();
  expect(await db.get("SELECT value FROM meta WHERE key = 'last_download_ok_at'")).toEqual({ value: "2026-11-15T10:00:00Z" });
  expect(await db.all("SELECT seq, state FROM outbox ORDER BY seq")).toEqual([{ seq: 1, state: "pending" }, { seq: 2, state: "needs_attention" }]);
  expect(await db.get("SELECT count(*) AS n FROM households")).toEqual({ n: 1 });
  expect(await names(db, "table")).toEqual(expect.arrayContaining(["city_blocks", "liraa_items", "ovitraps", "transmission_blocks"]));
});
test("migrating a v2 database again keeps its cursor", …);
test("a database from a newer app is refused, untouched", …);      // o teste existente passa a usar '3'
test("a fresh install starts at 2", …);
// round.test.ts (Q13(4))
test("a migrated tablet downloads on mobile data; a fresh one still waits for wifi", …);
// wipe.test.ts
test("wipeTerritory empties the vector tables but keeps an ovitrap the queue still installs or collects", …);
```
- [ ] **Step 2: Rodar e ver falhar** — `bin/test campo -w campo-mod35a src/db/migrations.test.ts src/security/wipe.test.ts src/sync/round.test.ts` · Expected: FAIL.
- [ ] **Step 3: Implementar** — `STATEMENTS_V1` (as de hoje) e `STATEMENTS_V2` (as quatro tabelas do "Esquema local v2", `status` de `ovitraps` com `CHECK`).
- [ ] **Step 4: Rodar e ver passar** — mesmo comando · Expected: PASS.
- [ ] **Step 5: Commit** — `src/db/migrations.ts`, `src/db/testing/schemaV1.ts`, `src/security/wipe.ts`, `src/sync/round.ts` e testes · `feat: migrate the local schema to v2 with the vector control tables`.

---

### Task 4: Download das entidades novas e do perfil do imóvel

**Files:**
- Create: `src/db/repos/vectorRepo.ts`
- Modify: `src/sync/download.ts`, `src/db/repos/territoryRepo.ts` (`removeHousehold`)
- Test: `src/sync/download.test.ts`, `src/db/repos/vectorRepo.test.ts`

**Interfaces:**
- Consumes: `CityBlockBody`, `OvitrapBody`, `LiraaItemBody`, `TransmissionBlockBody` (Task 2); tabelas da Task 3.
- Produces (`vectorRepo`):
  - `putCityBlock(db, b: CityBlockBody)` — substitui; inativo fica (vizinhos e histórico).
  - `putOvitrap(db, o: OvitrapBody)` — acha a linha pelo `id` ou por `client_ids` (tipo `ovitrap_installation`); `household_key` pelo `territoryRepo.householdKey(o.household_id) ?? o.household_id`; `status ≠ active` → apaga a linha.
  - `putLiraaItem(db, i: LiraaItemBody)` — `status ≠ "pending"` (inclusive `"closed"`) → apaga; senão substitui.
  - `putTransmissionBlock(db, t: TransmissionBlockBody)` — `status = "closed"` → apaga; senão substitui.
  - `blocksOf(db, neighborhoodId): Promise<CityBlockBody[]>` (ativos, ordenados por `number` natural), `pendingLiraaItem(db, householdId): Promise<LiraaItemBody | null>`, `activeBlockFor(db, cityBlockId): Promise<TransmissionBlockBody | null>`, `ovitrapsOf(db, householdKey?): Promise<OvitrapRow[]>`.
- Produces: `KNOWN_CHANGE_TYPES` = `micro_area`, `city_block`, `household`, `registration`, `ovitrap`, `liraa_item`, `transmission_block` — aplicados nessa ordem (quarteirão antes do domicílio; ovitrampa e item do LIRAa depois dele). O `<household>.vector` fica no `body` pelo `putHousehold` de hoje, sem mudança.
- Produces: `removeHousehold` também apaga as `ovitraps` do `household_key` que a fila não cita e os `liraa_items` do `household_id` (Q11).

- [ ] **Step 1: Escrever os testes** (páginas = exemplos novos da v1.3.0, por `expectValid("changes-response", …)`)

```ts
test("a page with city blocks, ovitraps, LIRAa items and a transmission block is stored", …);
test("household vector: present is kept, null means no profile, absent means the switch is off (Q6)", …);
test("ovitrap filed under the tablet-born row by client_ids after its installation was accepted", …);
test("liraa_item with status closed deletes the row; inspected and substituted too", …);
test("transmission_block closed deletes the row and never stores a case", () => { /* body sem chave de caso: expect(Object.keys(stored)).not.toContain("cases") */ });
test("ovitrap removed or lost deletes the row", …);
test("moved household takes its ovitraps and LIRAa items with it, but not an ovitrap the queue still collects", …);
test("the reset that comes when the switch toggles empties the vector tables and refills them (Q6)", …);
```
- [ ] **Step 2: Rodar e ver falhar** — `bin/test campo -w campo-mod35a src/sync/download.test.ts src/db/repos/vectorRepo.test.ts` · Expected: FAIL.
- [ ] **Step 3: Implementar.**
- [ ] **Step 4: Rodar e ver passar** — mesmo comando · Expected: PASS.
- [ ] **Step 5: Commit** — `src/db/repos/vectorRepo.ts`, `src/db/repos/territoryRepo.ts`, `src/sync/download.ts` e testes · `feat: download city blocks, ovitraps, LIRAa items and transmission blocks`.

---

### Task 5: Referência `vector_control_domains` e quem registra o boletim

**Files:**
- Create: `src/rules/vectorDomains.ts`
- Modify: `src/sync/reference.ts`, `src/features/session.ts`
- Test: `src/sync/reference.test.ts`, `src/features/session.test.ts` (novo)

**Interfaces:**
- Produces (`vectorDomains.ts`), os tipos com os nomes da Task 1 do dashboard:
```ts
export type VectorActivity = "li" | "li_t" | "pe" | "ie" | "transmission_block" | "liraa";
export type VectorWorkType = "routine" | "pending";
export type VectorSituation = "worked" | "closed" | "refused" | "vacant";
export type DepositGroup = "A1" | "A2" | "B" | "C" | "D1" | "D2" | "E";
export type MeasureMethod = "mechanical" | "focal" | "perifocal" | "spatial";
export type RiskClass = "green" | "yellow" | "red" | "gray" | "purple";
export interface Cadence { neighborhood_id: string | null; risk_class: RiskClass | null; strategic_point: boolean; days: number }
export interface VectorControlDomains {
  activities: { code: VectorActivity; label: string }[]; work_types: …; situations: …; deposit_groups: …; measure_methods: …; risk_classes: …;
  products: { id: string; code: string; name: string; unit: "g" | "ml" | "un"; charge_size: string; uses: Exclude<MeasureMethod, "mechanical">[] }[];
  cadences: Cadence[];
  parameters: { default_days: number; strategic_point_days: number; ovitrap_collection_days_min: number; ovitrap_collection_days_max: number };
  limits: { deposit_count_max: number; charges_max: number; samples_max: number; larvae_collected_max: number; measures_max: number };   // Q7(3)
}
export function parseVectorDomains(raw: unknown): VectorControlDomains | null;   // ausente → null; presente e fora da forma → ReferenceShapeError
```
  `charge_size` é texto decimal, como o dashboard (Q8: decimais como texto). Se o esquema da v1.3.0 disser número, vale o esquema, e o `measureQuantity` portado recebe `String(x)`.
- Produces: `Reference.vectorDomains: VectorControlDomains | null` (preenchido em `parseReference`).
- Produces (`session.ts`): `canRecordInspection(session, reference): boolean` (as três condições das Global Constraints, com `ENDEMIC_AGENT_CBO` importado de `src/rules/vectorInspection.ts`, Task 6 — até lá, constante local removida na Task 6); `vectorControlOn(session): boolean` (`features.vector_control === true`).

- [ ] **Step 1: Escrever os testes**

```ts
test("reference without vector_control_domains parses with vectorDomains null", …);   // exemplo da v1.2.0
test("vector_control_domains of the v1.3.0 example is parsed with the five limits", …);
test("a malformed vector block keeps the previous reference (kept_previous)", …);
test("canRecordInspection: ACE with the switch and the domains → true", …);
test.each([["515105", true, true], ["515140", false, true], ["515140", true, false], ["515140", undefined, true]])(
  "canRecordInspection(cbo %s, vector_control %s, domains %s) → false", …);
```
- [ ] **Step 2–4:** `bin/test campo -w campo-mod35a src/sync/reference.test.ts src/features/session.test.ts` FAIL → implementar → PASS.
- [ ] **Step 5: Commit** — `feat: read the vector control domains from the offline reference`.

---

### Task 6: Regras do boletim e frases, portadas do dashboard

Porte (cópia adaptada, sem mudar regra) dos dois módulos da Task 2 do plano do dashboard: `apps/dashboard/src/lib/vectorErrors.ts` e `apps/dashboard/src/lib/vectorInspection.ts`, no commit do dashboard do 35a que os criou (anotado no cabeçalho, como no `homeVisit.ts`). Adaptação permitida, como a Q76(2) do 24c: o "dia da visita" sai do `offsetMin` da cidade (o dashboard usa `cityIsoDate` da sessão), então `checkInspection` e `inspectionInput` nascem de `createInspectionRules({ offsetMin })`; o resto fica solto, com a assinatura do dashboard. Se o dashboard mudar um nome depois desta task, o campo segue no mesmo commit em que portar.

**Files:**
- Create: `src/rules/vectorErrors.ts`, `src/rules/vectorInspection.ts`, `src/rules/testing/vectorFixtures.ts`
- Modify: `src/features/messages/errorPhrases.ts`, `src/features/session.ts` (importa `ENDEMIC_AGENT_CBO`)
- Test: `src/rules/vectorErrors.test.ts`, `src/rules/vectorInspection.test.ts` (porte de `vectorErrors.test.ts` e `vectorInspection.test.ts` do dashboard), `src/features/messages/errorPhrases.test.ts`

**Interfaces:**
- Consumes: `VectorControlDomains` (Task 5), `VisitOutcome`, `parseCityLocal` (24c).
- Produces (`src/rules/vectorErrors.ts`): `VECTOR_CONTROL_DISABLED`, `VECTOR_ERROR_CODES`, `VECTOR_ERRORS: Record<VectorErrorCode, string>` — **a tabela do dashboard**, já com a Q9: `invalid_transition` no lugar de `status_changed` e `sample_label_taken` = "esta etiqueta já foi usada — gere etiquetas novas para este boletim".
- Produces (`src/rules/vectorInspection.ts`), com os nomes da Task 2 do dashboard:
  - `ENDEMIC_AGENT_CBO = "515140"`, `DERIVED_REASONS = [34, 35, 36, 37] as const`, `PE_PROPERTY = "4"`, `IE_PROPERTIES` (`"5"`…`"13"`).
  - `InspectionIssue = { error: VectorErrorCode; path: string }`; `InspectionCtx` (a do dashboard: `visit`, `household`, `property_type`, `profile`, `agent`, `gateUsable`, `products`, `blocks?`, `liraaItems?`, `transmissionBlocks?`). No app os opcionais vêm **sempre** preenchidos do banco local (Task 4), então a tela confere também os vínculos.
  - `createInspectionRules({ offsetMin }): { checkInspection(input: VectorInspectionBody, ctx: InspectionCtx): InspectionIssue[]; inspectionInput(draft: InspectionDraft, day: string): VectorInspectionBody }` — mesma ordem e mesmos `path` da tabela de `path` do dashboard (Q9), inclusive sem domicílio → `vector_inspection_invalid` em `household_id` (Q10) e os limites dos `limits` da referência (Q7(3)).
  - Soltos: `derivedReasons(input)`, `outcomeForSituation(s)`, `sampleLabel(clientUuid, n)`, `measureQuantity(charges, chargeSize)`, `InspectionDraft`, `emptyInspectionDraft(start?: VectorStart)`, `pruneInspection(draft)`, `VectorStart = { activity?; workType?; liraaDrawItemId?; transmissionBlockId? }`.
  - `issueText(issue): VisitProblem` (`{ path: issue.path, text: VECTOR_ERRORS[issue.error] }`), o adaptador para o formato `{ path, text }` das telas do 24c.
- `errorPhrases.ts`: cada código novo de `errors.item` recebe `VECTOR_ERRORS[code]`; o teste existente fecha o vermelho deixado pela Task 2.

- [ ] **Step 1: Portar os testes** do dashboard (os dois arquivos inteiros) trocando os imports e alimentando os domínios por `vectorFixtures.ts` (montado do exemplo de referência da v1.3.0). Mesmo `error` e mesmo `path` em cada caso. Acréscimos do campo (se o dashboard não os tiver, vão ao coordenador como achado):

```ts
test("checkInspection uses the city offset: 23:50 entry at -03:00 is another day for a 00:10 visit", …);
test("merged list (reasons + derivedReasons) over reasons_max is refused by the 24b rule at reasons", …);   // createVisitRules com reasons = merged
test("limits come from the reference: 8 deposit groups, measures over measures_max and samples over samples_max → vector_inspection_invalid at the list", …);   // já no dashboard (Q7(3))
// vectorErrors.test.ts
test("sample_label_taken and invalid_transition have the Q9 phrases; status_changed is gone", …);
// errorPhrases.test.ts
test("vector item codes use VECTOR_ERRORS", () => { for (const code of VECTOR_ERROR_CODES) if (KNOWN_ITEM_ERRORS.includes(code)) expect(ERROR_PHRASES[code]).toBe(VECTOR_ERRORS[code]); });
```
- [ ] **Step 2: Rodar e ver falhar** — `bin/test campo -w campo-mod35a src/rules/vectorErrors.test.ts src/rules/vectorInspection.test.ts src/features/messages` · Expected: FAIL.
- [ ] **Step 3: Portar o código** (só imports, `offsetMin` pela fábrica, tipos do contrato).
- [ ] **Step 4: Rodar e ver passar** — o mesmo comando e depois `bin/test campo -w campo-mod35a` · Expected: PASS, sem vermelho de propósito restante.
- [ ] **Step 5: Commit** — `src/rules/vectorErrors.ts`, `src/rules/vectorInspection.ts`, `src/rules/testing/vectorFixtures.ts`, `src/features/messages/errorPhrases.ts`, `src/features/session.ts` e testes · `feat: port the dashboard vector inspection rules and phrases`.

---

### Task 7: Boletim na visita

O comportamento da tela é o da Task 5 do plano do dashboard, adaptado ao tablet; os rótulos e as frases de tela são copiados de `InspectionSection.tsx` dele.

**Files:**
- Create: `src/features/visits/InspectionSection.tsx`, `src/ui/counter.tsx`, `src/features/vector/profile.ts`
- Modify: `src/features/visits/VisitForm.tsx`, `src/features/visits/save.ts`, `app/(main)/visits/new.tsx` (params `vector_activity`, `work_type`, `liraa_draw_item_id`, `transmission_block_id` → `VectorStart`)
- Test: `src/features/visits/save.test.ts`, `src/features/visits/InspectionSection.test.tsx`, `src/features/visits/VisitForm.test.tsx`, `src/features/vector/profile.test.ts`

**Interfaces:**
- Consumes: `createInspectionRules`, `derivedReasons`, `outcomeForSituation`, `sampleLabel`, `measureQuantity`, `emptyInspectionDraft`, `pruneInspection`, `issueText`, `DERIVED_REASONS` (Task 6); `canRecordInspection` (Task 5); `vectorRepo` (Task 4); `cityNowIso`.
- Produces (`profile.ts`): `readProfile(db, householdKey): Promise<{ city_block_id: string | null; risk_class: RiskClass | null; last_worked_on: string | null; last_situation: VectorSituation | null } | null>` — o `body.vector` da linha (ausente ou `null` → `null`) com, por cima, o quarteirão e o risco de visitas com boletim ainda na fila ou aceitas hoje (`local_visits.summary.vector`).
- Produces (`save.ts`): `SaveVisitContext` ganha `clientUuid: string` (o do formulário, passado ao `enqueue`) e `vectorInspection: VectorInspectionBody | null`; com boletim, `body.vector_inspection` e `summary.vector = { activity, situation, city_block_id?, risk_class? }`; `body.reasons` nunca leva 34–37.
- Produces (`counter.tsx`): `Counter({ label, value, max, onChange, testID })` — "−" e "+" de 48 dp e o número digitável (texto, como o rascunho do dashboard); nunca abaixo de 0 nem acima de `max`.
- Tela: com `canRecordInspection`, a chave "Registrar boletim" (ligada); com ela, a seção vem depois de "Quem foi visitado", nesta ordem (§14): atividade (sem `liraa` e `transmission_block`, salvo com o vínculo do `VectorStart`, Q8; `pe` só em `"4"`, `ie` só em `"5"`–`"13"`), tipo de trabalho, situação (o desfecho vira a linha "Desfecho: <rótulo> (da situação do imóvel)" por `outcomeForSituation`; trocar passa por `pruneInspection` e a tela diz o que saiu), entrada e saída (`hh:mm`; entrada = o minuto do `occurred_at` oferecido, saída = agora ao salvar se não foi tocada, Q11), depósitos ("Adicionar grupo" e cinco `Counter` com `max` = `limits.deposit_count_max`; no máximo 7 grupos), medidas (até `limits.measures_max`; método, produto filtrado por `uses`, cargas, e ao lado a quantidade de `measureQuantity` com a unidade), amostras (até `limits.samples_max`; grupo só entre os com larva; larvas até `limits.larvae_collected_max`; a etiqueta `sampleLabel(clientUuid, n)` aparece assim que a linha nasce; "Apagar" só na última), ação educativa, quarteirão (só sem quarteirão no perfil: os ativos do `address.neighborhood_id`; sem bairro, `VECTOR_ERRORS.household_without_neighborhood` e nenhuma escolha) e risco. Os motivos 34–37 somem da lista do 24b e, abaixo dela, "O boletim acrescenta: 34 Ação educativa, …". Sem domicílio cadastrado (Q10): "O boletim precisa de um imóvel cadastrado." e nenhuma seção. Sem domínios: "O boletim chega na próxima sincronização."
- Ao salvar: `visitProblems` do 24b com `reasons = reasons + derivedReasons` (limite `reasons_max`, `path` `reasons`) mais `checkInspection(...).map(issueText)`; a primeira recusa rola até o campo (`testID` `field-<path>`).

- [ ] **Step 1: Escrever os testes**

```ts
// save.test.ts
test("the visit is enqueued with the client_uuid chosen when the form opened", …);
test("vector_inspection goes in the body and reasons never carry 34–37", …);
test("summary keeps activity and situation, nothing said in the house", …);
test("visit body with vector_inspection validates against upload-request-v1.json", …);
// InspectionSection.test.tsx
test("hidden for the ACS; hidden with a notice when the domains are missing; off when 'Registrar boletim' is unchecked", …);
test("the label of a new sample shows at once and stays the same after other edits", …);
test("only the last sample can be removed", …);
test("closed after worked: prune says what went and the outcome line becomes absent", …);
test("block choice only when the profile has none, from the household neighbourhood", …);
test("liraa and transmission_block offered only with the link of the start (Q8); pe only for property type 4", …);
test("entry defaults to the minute of occurred_at, so entered_at ≤ occurred_at", …);
test("saving after midnight with entry the day before is refused on screen at vector_inspection.entered_at", …);
test("no more than 7 deposit groups and limits.measures_max measures", …);
// VisitForm.test.tsx
test("ACE visit with inspection saves and shows today's sheet", …);
test("reasons 34–37 are not offered while the inspection is on, and the preview lists the derived ones", …);
// profile.test.ts
test("a queued inspection that sets the block shows it in the profile before the download", …);
test("vector absent or null → no profile", …);
```
- [ ] **Step 2–4:** `bin/test campo -w campo-mod35a src/features/visits src/features/vector/profile.test.ts src/ui` FAIL → implementar → PASS.
- [ ] **Step 5: Commit** — `feat: record the vector inspection inside the home visit offline`.

---

### Task 8: Folha "Imóveis a visitar"

Não há regra do dashboard para portar: o dashboard lê `GET /vector_control/due`. A regra vem do api (`VectorControl::Cadence.days_for` e a lista de vencidos do §5), portada para TS com os casos do spec do api (Q11); a ordem é a do `/due` (Q8).

**Files:**
- Create: `src/rules/vectorDue.ts`, `src/features/vector/dueData.ts`, `src/features/vector/DueScreen.tsx`, `app/(main)/vector/due.tsx`
- Modify: `src/features/home/homeData.ts`, `src/features/home/HomeScreen.tsx`, `app/(main)/index.tsx`
- Test: `src/rules/vectorDue.test.ts`, `src/features/vector/DueScreen.test.tsx`, `src/features/home/HomeScreen.test.tsx`

**Interfaces:**
- Consumes: `Cadence`, `parameters` (Task 5); `readProfile` (Task 7); `vectorRepo` (Task 4); `VectorStart` (Task 6).
- Produces (`vectorDue.ts`):
```ts
export type DueReason = "transmission_block" | "liraa" | "pending" | "due";
export function cadenceDays(h: { propertyType: string; neighborhoodId: string | null; riskClass: RiskClass | null }, cadences: Cadence[], p: { default_days: number; strategic_point_days: number }): number;
export interface DueInput { householdKey: string; householdId: string | null; status: "active" | "inactive"; propertyType: string; neighborhoodId: string | null;
  profile: { city_block_id: string | null; risk_class: RiskClass | null; last_worked_on: string | null; last_situation: VectorSituation | null } | null;
  liraaItemId: string | null; transmissionBlockId: string | null }
export interface DueItem { householdKey: string; reason: DueReason; dueSince: string | null; lastWorkedOn: string | null; riskClass: RiskClass | null;
  liraaDrawItemId: string | null; transmissionBlockId: string | null }
export function dueList(rows: DueInput[], today: string, cadences: Cadence[], p: …): DueItem[];   // um motivo por imóvel; ordem: motivo, dueSince (null primeiro), chave
export function startFor(item: DueItem, propertyType: string): VectorStart;   // o startFor da Task 6 do dashboard
```
  PE (`"4"`): (bairro, PE) → (cidade, PE) → `strategic_point_days`; demais: (bairro, risco) → (bairro, qualquer) → (cidade, risco) → **(cidade, qualquer)** → `default_days` (Q10). Vencido: ativo e `last_worked_on` nulo ou `last_worked_on + days ≤ today`. `dueSince` = `last_worked_on + days`, `null` sem trabalho anterior (Q8/Q11). `startFor`: bloqueio → `{ activity: "transmission_block", transmissionBlockId }`; LIRAa → `{ activity: "liraa", liraaDrawItemId }`; pendência → `{ workType: "pending" }`; imóvel `"4"` → `{ activity: "pe" }`; senão `{}`.
- Produces (`dueData.ts`): `loadDue(db, now): Promise<{ items: (DueItem & { address: string; block: { id: string; number: string } | null; propertyType: string })[] }>` — os domicílios do escopo com o perfil da Task 7; nunca morador.
- Tela: cada imóvel com endereço curto, tipo, "Quarteirão 12" ou "sem quarteirão", risco, "trabalhado em dd/mm/aaaa" ou "nunca trabalhado", "vencido desde dd/mm/aaaa" e o motivo (rótulos de `dueReasonLabel` do dashboard: bloqueio, LIRAa, pendência, vencido); "Registrar visita" abre `/visits/new` com `household_id` e o `startFor` em params. Início: "Imóveis a visitar: N" e o botão, só com `canRecordInspection`.

- [ ] **Step 1: Escrever os testes**

```ts
test.each([
  ["PE with (neighbourhood, PE) 10", "4", 10], ["PE falls back to (city, PE) 12", "4", 12], ["PE falls back to strategic_point_days 15", "4", 15],
  ["(neighbourhood, red) 30", "1", 30], ["(neighbourhood, any) 45", "1", 45], ["(city, red) 50", "1", 50], ["(city, any) 55 (Q10)", "1", 55], ["default_days 60", "1", 60],
])("cadenceDays: %s", …);
test("worked on 2026-09-17 with 60 days is due on 2026-11-16; on 2026-09-18 it is not", …);
test("never worked is due with dueSince null and comes first", …);
test("one reason per household: block wins over LIRAa, LIRAa over pending, pending over due", …);
test("startFor: block, LIRAa, pending and PE", …);
test("inactive households never appear", …);
test("an inspection saved today on the tablet takes the household out of the list", …);
// DueScreen.test.tsx: mostra motivo e quarteirão, nunca nome de morador; "Registrar visita" do LIRAa leva activity e o vínculo
// HomeScreen.test.tsx: contador e botão só para o ACE com o interruptor
```
- [ ] **Step 2–4:** `bin/test campo -w campo-mod35a src/rules/vectorDue.test.ts src/features/vector src/features/home` FAIL → implementar → PASS.
- [ ] **Step 5: Commit** — `feat: list the properties due for an inspection on the tablet`.

---

### Task 9: Ovitrampas — instalar e coletar

Porte de `installProblems` e `collectionProblems` da Task 3 do plano do dashboard (`apps/dashboard/src/lib/vectorControl.ts`), com as frases de `VECTOR_ERRORS`; as telas seguem a Task 7 dele.

**Files:**
- Create: `src/rules/ovitrap.ts`, `src/features/ovitraps/{save.ts,view.ts,OvitrapList.tsx,InstallForm.tsx,CollectForm.tsx}`, `app/(main)/ovitraps/{index.tsx,new.tsx,[key]/collect.tsx}`
- Modify: `src/sync/dispatch.ts` (aceite da instalação), `src/features/conflicts/fieldText.ts` (Q13(2)), `src/db/repos/vectorRepo.ts` (`acceptInstallation`), `src/features/households/HouseholdScreen.tsx` ("Instalar ovitrampa"), `src/features/home/*` ("Ovitrampas"), `src/features/attention/AttentionScreen.tsx` ("Corrigir número", Q11)
- Test: `src/rules/ovitrap.test.ts`, `src/features/ovitraps/save.test.ts`, `src/features/ovitraps/view.test.ts`, `src/features/ovitraps/OvitrapScreens.test.tsx`, `src/sync/dispatch.test.ts`

**Interfaces:**
- Consumes: `vectorRepo`, `readProfile`, `enqueue`, `resend` (24c), `cityTodayOf`, `parameters` da referência, `VECTOR_ERRORS`.
- Produces (`ovitrap.ts`), com a assinatura do dashboard:
```ts
export function installProblems(d: { number: string; consent: boolean; installedOn: string; householdHasBlock: boolean | null }, today: string): { code: string | null; path: string; text: string }[];
  // consent_required, household_without_block quando false, número ^[0-9A-Za-z-]{1,12}$ (Q7(6)), data ≤ hoje
export function collectionProblems(d: { installedOn: string; collectedOn: string; condition: ReadingCondition | null }, p: { ovitrap_collection_days_min: number; ovitrap_collection_days_max: number }, today: string): { code: string | null; path: string; text: string }[];
  // collection_out_of_range em collected_on fora de [min, max] dias; data futura; condição
export function numberTaken(number: string, taken: string[]): boolean;   // do campo: ativas do escopo e instalações na fila, sem caixa → ovitrap_number_taken
```
  `path` das recusas (Q13(2)): `household_without_block` → `household_id`; `consent_required` → `consent`; `ovitrap_number_taken` → `number`; `reading_not_open` e `ovitrap_not_active` → `ovitrap_id`; `collection_out_of_range` → `collected_on`. As mesmas chaves entram em `src/features/conflicts/fieldText.ts` (`pathLabel` de `ovitrap_installation` e `ovitrap_collection`: "Imóvel", "Termo (TCLE)", "Número", "Ovitrampa", "Data da coleta"), para "precisa de atenção" pôr em palavras o campo da recusa do servidor (Q81(4) do 24c).
- Produces (`save.ts`):
  - `installOvitrap(db, input: { householdKey: string; number: string; clientUuid: string; today: string }): Promise<string>` — numa transação: item `ovitrap_installation` `{ household_id | household_client_uuid, number, consent: true, installed_on: today }` e a linha local `ovitraps` (`key = clientUuid`, `status active`, `body.open_reading = { installed_on: today }`); dep `cite` no domicílio nascido no tablet e na visita da fila que deu o quarteirão (o api aplica o lote em ordem, Q10).
  - `collectOvitrap(db, input: { ovitrapKey: string; condition: ReadingCondition; remove: boolean; today: string; newUuid(): string }): Promise<string>` — item `ovitrap_collection` `{ ovitrap_id | ovitrap_client_uuid, collected_on: today, condition, reinstall: !remove, remove }` (o api ignora `reinstall` com `remove`, Q7(5)); dep `cite` na instalação ainda na fila e na coleta anterior da mesma ovitrampa ainda na fila.
- Produces (`view.ts`): `readOvitraps(db, householdKey?)` — a linha com as coletas da fila por cima (`remove: false` → palheta aberta em `collected_on`; `remove: true` → some da lista); `OVITRAP_TABLES = ["ovitraps", "outbox"]`.
- Produces (`dispatch.ts`): aceite de `ovitrap_installation` → `client_ids(client_uuid, "ovitrap_installation", result.id)` e `vectorRepo.acceptInstallation(db, key, result.id)`. O `id` do `accepted`/`duplicate` da instalação e da coleta é o da ovitrampa (Q7(2)); não há `ovitrap_id`.
- Telas (rótulos e frases da Task 7 do dashboard): "Instalar ovitrampa" no detalhe do domicílio (só ACE com o interruptor; sem quarteirão, `VECTOR_ERRORS.household_without_block` e o botão desligado); formulário com número e "O morador assinou o termo de consentimento (TCLE)"; lista "Ovitrampas" com número, imóvel, "palheta desde dd/mm — colete entre dd/mm e dd/mm (ideal dd/mm)" (min, max e o alvo de 5 dias) e "Coletar palheta"; coleta com condição, "Reinstalar palheta" (marcado) ou "Retirar a ovitrampa" (com "perdida", "a ovitrampa sai como perdida"). Nada de contagem no app.
- "Corrigir número" (Q11): em "precisa de atenção" com `ovitrap_number_taken`, um campo de número e "Reenviar", que chama `resend(db, seq, { ...body, number })` com o mesmo `client_uuid` e troca o número da linha local.

- [ ] **Step 1: Escrever os testes**

```ts
test.each([["2026-11-02", "2026-11-04", true], ["2026-11-02", "2026-11-05", false], ["2026-11-02", "2026-11-12", false], ["2026-11-02", "2026-11-13", true]])(
  "collection installed %s, collected %s → out of range = %s (2 and 11 days)", …);
test("number OV-12 ok; 'OV 12' and 13 characters refused (Q7(6))", …);
test("number taken by an active ovitrap or a queued installation, case-insensitive → ovitrap_number_taken", …);
test("install then collect offline: the collection cites ovitrap_client_uuid with a cite dependency", …);
test("two queued collections of the same ovitrap: the second depends on the first", …);
test("held installation blocks its collection", …);   // selectBatch + markBlocked do 24c
test("remove sends reinstall false; the ovitrap leaves the list at once", …);
test("installation and collection bodies validate against upload-request-v1.json", …);
test("accepted installation maps client_ids to result.id and the row gets it", …);
test("the download of the server ovitrap replaces the tablet-born row by client_ids", …);
test("'Corrigir número' resends the same client_uuid with the new number", …);
test("ACS sees no ovitrap button", …);
test.each([["household_without_block", "household_id"], ["consent_required", "consent"], ["ovitrap_number_taken", "number"], ["reading_not_open", "ovitrap_id"], ["ovitrap_not_active", "ovitrap_id"], ["collection_out_of_range", "collected_on"]])(
  "server refusal %s at %s shows its phrase and the field in words (Q13(2))", …);
```
- [ ] **Step 2–4:** `bin/test campo -w campo-mod35a src/rules/ovitrap.test.ts src/features/ovitraps src/sync/dispatch.test.ts src/features/attention src/features/conflicts src/features/households` FAIL → implementar → PASS.
- [ ] **Step 5: Commit** — `feat: install ovitraps and collect their paddles offline`.

---

### Task 9b: Imóvel não residencial pelo agente de endemias (Q13(1))

O agente cadastra o ponto estratégico, o comércio, o terreno baldio e os demais não residenciais da área dele, para registrar o boletim neles. Usa o formulário e o item `household` do 24c, sem caminho novo; só a permissão muda. Rótulos e frases: os do dashboard, se o plano dele tiver para este caso; senão, os abaixo, ao coordenador.

**Files:**
- Modify: `src/features/session.ts`, `src/features/households/HouseholdForm.tsx`, `src/features/households/HouseholdList.tsx`, `src/features/households/HouseholdScreen.tsx`, `src/features/households/save.ts`
- Test: `src/features/session.test.ts`, `src/features/households/HouseholdForm.test.tsx`, `src/features/households/HouseholdScreens.test.tsx`, `src/features/households/save.test.ts`

**Interfaces:**
- Produces (`session.ts`): `AGENT_PROPERTY_TYPES = ["2", "3", "4", "5", "6", "12", "13"] as const`; `agentRegistersNonResidential(session): boolean` (`features.vector_control === true`, `features.territory_registry === true`, `professional.cbo === ENDEMIC_AGENT_CBO`); `canEditHousehold(session, propertyType: string | null): boolean` — `canEditRegistry(session)`, ou `agentRegistersNonResidential(session)` com `propertyType === null` (domicílio novo) ou em `AGENT_PROPERTY_TYPES`. `canEditRegistry` fica como está e continua guardando família e cadastro individual.
- `HouseholdForm`: abre com `canEditHousehold(session, body?.property_type ?? null)`; para o agente, "Tipo de imóvel" oferece só `AGENT_PROPERTY_TYPES`, então não há troca para residencial; sem seções de condições de moradia (os sete tipos não as têm em `allowsLivingConditions`); microáreas = as da sessão. Domicílio residencial abre só para consulta, como hoje.
- `HouseholdList`: "Novo imóvel não residencial" para o agente, no lugar do "Novo domicílio" do ACS.
- `HouseholdScreen`: "Editar" por `canEditHousehold`; família e morador continuam por `canEditRegistry` (o agente não os vê); "Registrar visita" e "Instalar ovitrampa" como nas Tasks 7 e 9.
- `save.ts` (`saveHousehold`): recusa na tela, antes de enfileirar, tipo fora de `AGENT_PROPERTY_TYPES` quando quem salva é o agente (`cbo_not_allowed` em `property_type`, frase de `ERROR_PHRASES`). A recusa do servidor (`rejected cbo_not_allowed`, por exemplo um posto que mudou o tipo para residencial e a mescla trouxe a mudança) vai para "precisa de atenção" com Descartar, como no 24c.

- [ ] **Step 1: Escrever os testes**

```ts
// session.test.ts
test.each([["2", true], ["4", true], ["13", true], ["1", false], ["7", false], ["99", false], [null, true]])(
  "canEditHousehold for the endemic agent, type %s → %s", …);
test("ACS keeps editing every type; the agent never edits families nor registrations (canEditRegistry false)", …);
test("agent without vector_control or without territory_registry edits nothing", …);
// HouseholdForm.test.tsx
test("agent sees only the seven non-residential types and no living conditions", …);
test("agent opens a residential household read-only", …);
// HouseholdScreens.test.tsx
test("agent: 'Novo imóvel não residencial' in the list, no family or resident buttons in the detail", …);
// save.test.ts
test("agent saving a residential type is refused on screen and nothing is enqueued", …);
test("agent creates a strategic point (type 4) in an assigned micro area: one household item, body valid for upload-request-v1.json", …);
```
- [ ] **Step 2–4:** `bin/test campo -w campo-mod35a src/features/session.test.ts src/features/households` FAIL → implementar → PASS.
- [ ] **Step 5: Commit** — `feat: let the endemic agent register non-residential properties offline`.

---

### Task 10: Etiqueta repetida (`sample_label_taken`, Q7(1))

**Files:**
- Create: `src/features/visits/relabel.ts`
- Modify: `src/features/attention/AttentionScreen.tsx`
- Test: `src/features/visits/relabel.test.ts`, `src/features/attention/AttentionScreen.test.tsx`

**Interfaces:**
- Produces: `relabelVisit(db, seq: number, newUuid: () => string): Promise<{ clientUuid: string; labels: string[] }>` — só para item `home_visit` em `needs_attention` com `error = "sample_label_taken"`, nunca aceito; numa transação troca o `client_uuid` do item, a chave de `local_visits` e o `visit_client_uuid` de retirada na fila (se houver), e volta o item a `pending`. Nunca automático.
- Tela: `VECTOR_ERRORS.sample_label_taken`, as etiquetas antigas e "Gerar etiquetas novas"; depois, "Etiquete os tubitos: <etiquetas novas>" (o molde do dashboard depois do 201).

- [ ] **Step 1: Escrever os testes**

```ts
test("relabel changes the item uuid, the sheet key and the labels together", …);
test("relabel refuses another code, another item type, or an item not in needs_attention", …);
test("labels after relabel match sampleLabel of the new uuid", …);
test("the attention screen shows old and new labels and never relabels by itself", …);
```
- [ ] **Step 2–4:** `bin/test campo -w campo-mod35a src/features/visits/relabel.test.ts src/features/attention` FAIL → implementar → PASS.
- [ ] **Step 5: Commit** — `feat: let the agent regenerate sample labels after a label clash`.

---

### Task 11: Servidor falso e fluxos do Maestro

**Files:**
- Modify: `scripts/fake-field-server/{state.mjs,server.mjs,server.test.ts}`, `maestro/README.md`, `eslint.config.js` (globais novas dos scripts)
- Create: `scripts/fake-field-server/fixtures/vector-products.json`, `maestro/{vector-inspection-offline,ovitrap-install,ovitrap-collect,strategic-point-offline}.yaml`, `maestro/common/pair-ace.yaml`

**Interfaces:**
- Produces (servidor falso):
  - `POST /__script { "profile": "ace" }` — fica até o `__reset`: a sessão vira a do exemplo de ACE da v1.3.0 (CBO `515140`, `features.vector_control: true`); sem ele, tudo como hoje.
  - `/reference` com `vector_control_domains` montado do `domains-v1.json` (`vector_control`) e de `vector-products.json`, só no perfil ACE; validado contra o esquema antes de sair.
  - `/changes` no perfil ACE: o exemplo de página com as entidades novas da v1.3.0, com `open_reading.installed_on` reescrito para hoje − 5 dias (para a coleta no aparelho caber na faixa de 3 a 10); no perfil ACS, os domicílios sem a chave `vector` (gate desligado, Q6).
  - `/uploads`: `ovitrap_installation` e `ovitrap_collection` aceitos com `id` = id da ovitrampa (Q7(2)); `ovitrap_client_uuid` resolvido pelo livro (`CITES`); `home_visit` com boletim aceito com `vector_inspection_id`; `next_upload: "rejected:sample_label_taken"` vale com `path` `vector_inspection.samples[0]`; `GET /__state` ganha `vector: { inspections, samples, bodies_with_derived_reasons }`.
- Fluxos (cada um começa com `runScript` `profile: ace` e `common/pair-ace.yaml`):
  - `vector-inspection-offline`: modo avião; "Imóveis a visitar"; um imóvel; boletim LI, rotina, trabalhado, A1 existente 2 / com água 1 / com larva 1, uma amostra (etiqueta visível, `[0-9A-F]{8}-01`), focal com produto, ação educativa; "Registrar visita"; "Pendentes de envio: 1"; rede; "Sincronizar"; "Pendentes de envio: 0"; o script confere `vector.inspections = 1`, `samples = 1`, `bodies_with_derived_reasons = 0`.
  - `ovitrap-install`: modo avião; domicílio com quarteirão; "Instalar ovitrampa", número, TCLE; rede; sincroniza; `accepted.ovitrap_installation = 1`.
  - `ovitrap-collect`: "Ovitrampas"; a do download; "Coletar palheta", condição boa, reinstalar; sincroniza; `accepted.ovitrap_collection = 1`.
  - `strategic-point-offline` (Q13(1)): modo avião; "Domicílios" → "Novo imóvel não residencial", tipo ponto estratégico, endereço e microárea; salvar; no detalhe, "Registrar visita" com boletim de atividade PE e situação trabalhado; "Pendentes de envio: 2"; rede; sincroniza; zero; o script confere `accepted.household = 1` e `vector.inspections = 1` (a visita citou `household_client_uuid`).

- [ ] **Step 1: Testes do servidor falso** (`server.test.ts`, no `bin/test`): perfil ACE dá sessão e referência válidas pelos esquemas da v1.3.0; `/changes` do ACE valida e traz `installed_on` de hoje − 5; instalação + coleta citando `ovitrap_client_uuid` no mesmo lote → dois `accepted` com o mesmo `id`; coleta citando uuid desconhecido → `rejected not_found` com `path` `ovitrap_client_uuid`; `__reset` volta ao perfil ACS. Run: `bin/test campo -w campo-mod35a scripts` · FAIL → implementar → PASS.
- [ ] **Step 2: Escrever os quatro fluxos** com as regras do `maestro/README.md` (rolar antes do último botão; opção de busca com `index: 1`) e a linha deles na tabela do README.
- [ ] **Step 3: Lint como a CI** — `npx eslint .` e `npx tsc --noEmit` no worktree · Expected: sem erro.
- [ ] **Step 4: Commit** — `test: add Maestro flows for the vector inspection and the ovitraps`.

---

### Task 12: Prova no emulador

**Exige autorização do usuário** (build Android, emulador, e o api e o dashboard de dev do 35a de pé). Sem ela, pare e relate o que está pronto. Ambiente: a seção "Ambiente de execução".

**Files:** nenhum código; relato ao coordenador.

- [ ] **Step 1: APK e2e release** — chave descartável no scratchpad (`keytool … -keystore <scratchpad>/campo-e2e.jks`), `EXPO_PUBLIC_E2E=1 EXPO_PUBLIC_DEV_ORIGIN=http://10.0.2.2:8787 CAMPO_ALLOWED_HOST_SUFFIXES=.localhost,.test` exportadas no shell do prebuild **e** do Gradle, `./gradlew :app:assembleRelease -PreactNativeArchitectures=arm64-v8a -PCAMPO_E2E_…`, `node scripts/check-apk.mjs <apk> --e2e` · Expected: exit 0.
- [ ] **Step 2: Os 15 fluxos contra o servidor falso** — `caffeinate -i` com o laço do `maestro/README.md` (os 11 do 24c e os 4 novos) · Expected: todos `PASSED`. Um fluxo antigo vermelho é regressão desta branch: volte à task dona.
- [ ] **Step 3: Prova integrada contra o api de dev do 35a** — o APK e2e com `EXPO_PUBLIC_DEV_ORIGIN=http://10.0.2.2:3059` vai ao emulador **antes** do `db:seed` que liga `vector_control` (Q6(4)): instale e pareie com o interruptor desligado, depois o seed (com o usuário), e a rodada seguinte traz o `reset: true`. Antes, `curl -s -H 'Host: curitiba.localhost' http://localhost:3059/up`. QR do dashboard de dev (5220, Aparelhos de campo, para `ace@curitiba.demo`), por `-e QR_PAYLOAD=…`, nunca em arquivo. Casos (roteiro do §18, item 9 e a parte do app dos itens 1, 4 e 5):
  1. o `reset` do interruptor traz quarteirões, ovitrampas, itens do LIRAa e o bloqueio ativo da semente; a folha mostra bloqueio e LIRAa por cima;
  2. boletim offline com amostra num item do LIRAa; volta a rede; no dashboard (`endemias@curitiba.demo`), a visita com o boletim, a etiqueta igual à do tablet e os motivos 34–37 derivados; o item do LIRAa `inspected`;
  3. boletim com medida de um produto do local de estoque (baixa) e outro sem saldo (falta-estoque registrado, visita aceita);
  4. coleta da ovitrampa da semente com leitura aberta (instalada há 3 a 10 dias; se nenhuma couber, ajuste a data pelo runner do api, com o usuário) e contagem pelo `entomologia@curitiba.demo` no dashboard;
  5. instalação de ovitrampa nova num imóvel com quarteirão;
  5b. cadastro offline de um ponto estratégico pelo agente, com boletim PE; o dashboard mostra o imóvel com a ficha 3 no CBO 515140 (Q13(1));
  6. o interruptor desligado de novo: chega o `reset`, o boletim some, a fila guarda o que havia; religar envia.
  Evidência: `FLAG_SECURE` deixa a captura preta; descreva, ou use `CAMPO_E2E_SCREEN_CAPTURE=1` só para a evidência.
- [ ] **Step 4: Limpeza** — apague a chave e os APKs do scratchpad; os registros fictícios ficam no dev de Curitiba (diga quais no relato).
- [ ] **Step 5: Relato** ao coordenador: casos, resultado, o que ficou de fora, divergências novas.

---

### Task 13: Entrega (para antes do merge)

**Files:** nenhum código novo, salvo correções da revisão.

- [ ] **Step 1: Suíte do app** — `bin/test campo -w campo-mod35a` · Expected: `resultado: ok`, total = linha de base da Task 0 + os novos.
- [ ] **Step 2: Comandos da CI** — `npx eslint .`, `npx tsc --noEmit`, `node scripts/contract-types.mjs --check` · Expected: sem erro; "6 files up to date".
- [ ] **Step 3: Revisão** — `superpowers:requesting-code-review` sobre `origin/main..feat/mod-35a-vector-control`, com este plano e o contrato; correções com `bin/test campo -w campo-mod35a <arquivos>` de escopo fechado, cada uma no seu commit.
- [ ] **Step 4: Relato e parada** — ao coordenador: commits, testes, provas, divergências; card F-35.10 fica em In Progress (vai a Done quando contracts, api e campo estiverem entregues, pelo último). **Push, PR e merge só com autorização do usuário**: este plano termina aqui.

---

## Self-review

- **Cobertura do contrato:** §14 esquema v2 (Task 3), boletim com ordem e condições (Tasks 5–7), regras do §3 portadas (Task 6), folha local com LIRAa e bloqueio por cima (Task 8), ovitrampa sem contagem (Task 9), ordem de envio por citação (Task 9), testes de regras/migração/envio e Maestro (Tasks 1–11). §13 tipos e itens novos (Task 2), entidades do download e reemissão (Task 4), referência (Task 5), compatibilidade (Task 1, Q6). §9 etiqueta (Tasks 6, 7, 10). §6 coleta 3–10 dias e TCLE (Task 9). §17 sigilo: `transmission_block` sem caso (Task 4), folha sem morador (Task 8). §1 desligado: `reset` e `feature_disabled` volta a `pending` pela regra do 24c (Task 12, caso 6). §5 cadência com o degrau da Q10 (Task 8). Q13(1) imóvel não residencial pelo agente (Task 9b, fluxo `strategic-point-offline`); Q13(2) `path` da ovitrampa (Task 9); Q13(4) carga inicial (Task 3).
- **Tipos:** nomes da Task 2 do dashboard (`checkInspection`, `InspectionCtx`, `InspectionDraft`, `VectorStart`, …) usados nas Tasks 6–10; `readProfile` (Task 7) nas Tasks 8 e 9; `vectorRepo` (Task 4) nas 7–9; `ItemType` com os tipos novos (Task 2) no `dispatch` (Task 9).
- **Vermelho de propósito:** só entre as Tasks 2 e 6 (frase dos códigos novos), anotado nas duas.
- **Proporção:** nomes, assinaturas, valores do contrato e testes; corpo só onde a regra não sai da assinatura (nenhum).

## Divergências propostas ao contrato

| # | Achado | Decisão | Onde o plano aplica |
|---|---|---|---|
| D1 | Compatibilidade do §13 falsa na `main` do campo: tipo desconhecido trava o download. | **Q6** — api só emite as entidades e o `vector` com o gate; o interruptor vira `reset`; o campo ignora tipo desconhecido; a migração zera o cursor; gate de go-live. | Tasks 1, 3, 4, 12 |
| D2 | Colisão de etiqueta reusava `client_uuid_conflict` e trocava o uuid sozinho. | **Q7(1)** — `sample_label_taken`, `path` `vector_inspection.samples[i]`; troca só por ação do agente. | Tasks 2, 6, 10, 11 |
| D3 | `ovitrap_id` ao lado do `id` no `accepted`. | **Q7(2)** — `id` é o da ovitrampa (instalação e coleta); sem `ovitrap_id`. | Tasks 9, 11 |
| D4 | Nomes das chaves de `limits`. | **Q7(3)** — `deposit_count_max`, `charges_max`, `samples_max`, `larvae_collected_max`, `measures_max` (20); até 7 grupos. | Tasks 2, 5, 7 |
| D5 | `remove: true` com `reinstall: true`. | **Q7(5)** — o api ignora `reinstall` com `remove`, sem acoplar o esquema; o app manda `reinstall: !remove`. | Task 9 |
| D6 | Entrada em segundos depois do `occurred_at` em minutos. | **Q11** — entrada padrão = minuto do `occurred_at`; dia = dia da cidade. | Tasks 6, 7 |
| D7 | Folha sem regra do dashboard; `due_since` sem trabalho anterior. | **Q11** e **Q8** — porte do api; `due_since = null`, primeiro na ordem. | Task 8 |
| D8 | `local_visits.summary` sem o boletim. | **Q11** — `summary.vector { activity, situation, city_block_id?, risk_class? }`. | Tasks 7, 8 |
| D9 | Fim das entidades sem `removal`. | **Q11** — apaga LIRAa não pendente, bloqueio fechado, ovitrampa inativa; cascata do domicílio; quarteirão só no `reset`. | Task 4 |
| D10 | Imóvel no bloqueio e no LIRAa. | **Q11** — um motivo e um vínculo; o bloqueio vence. | Task 8 |
| D11 | Cadência sem (cidade, qualquer). | **Q10** — degrau (cidade, qualquer risco) antes de `default_days`. | Task 8 |
| D12 | Instalação depende de boletim da mesma fila. | **Q10** — o api aplica o lote em ordem, cada item vendo o anterior. | Task 9 |
| D13 | Boletim em visita sem domicílio. | **Q10** — `vector_inspection_invalid`, `path` `household_id`; o app esconde a seção. | Task 7 |
| D14 | Precedência do dashboard no porte. | **Q9** e **Q11** — nomes, `path` e frases da Task 2 do dashboard (`vectorErrors.ts`, `vectorInspection.ts`). | Tasks 6, 7, 9 |
| D15 | `ovitrap_number_taken` sem "Corrigir". | **Q11** — "Corrigir número" para `ovitrap_installation`, mesmo `client_uuid`. | Task 9 |
| N1 | A Q6 zera o cursor na migração, e o 24c trata `cursor === null` como carga inicial (espera Wi-Fi). | **Q13(4)** — carga inicial = `cursor === null && last_download_ok_at === null`; a migração mantém `last_download_ok_at`. | Task 3 |
| — | Imóvel não residencial pelo agente de endemias (conflito do dashboard). | **Q13(1)** — só os tipos `2 3 4 5 6 12 13`, sem família nem morador, nas microáreas dele. | Task 9b, Task 11 |
| — | `path` das recusas da ovitrampa. | **Q13(2)**. | Task 9 |
