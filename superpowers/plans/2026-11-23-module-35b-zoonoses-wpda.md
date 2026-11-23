# Módulo 35b — Centro de zoonoses (wpda) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisito:** o **api do 35b** na branch `feat/mod-35b-zoonoses` (worktree `apps/api/.claude/mod35b`, porta **3061**), com o registro do 33 aceitando `zoonoses_subtype` e `place_address` (contrato §4) e com `public_config.zoonoses` (Q10). O api aceita as duas chaves sob a mesma condição. Ordem (contrato §18): contracts → api → **dashboard, wpda e maintenance** → campo. As Tasks 0–4 não dependem do api rodando (dublês de `fetch`, `vi.spyOn(citizenApi, …)` e Playwright com rotas respondidas); a prova da Task 5 depende.

**Goal:** Lado wpda de F-35.13: na "Nova manifestação" (Solicitação e Denúncia) e na "Denúncia anônima", com o `zoonoses` ligado na cidade, a pergunta "É sobre animais, insetos ou bichos?" leva o subtipo e o endereço do local ao registro do 33, que o api encaminha sozinho ao centro de zoonoses. Desligado, nada muda. "Minhas manifestações" segue o 33 e nunca mostra a demanda nem a vistoria.

**Architecture:** Conferido em `origin/main` do wpda (`cfa6675`) e do api (`85dc80ee`).
- O wpda sabe do `ombudsman` pela sessão (`hasSessionFeature(OMBUDSMAN_FEATURE)`, `Flow.tsx:115,281`), que o api monta **sem** as chaves de `CITIZEN_HIDDEN` (`sessions_controller.rb:44-45`). O `zoonoses` entra em `CITIZEN_HIDDEN` (contrato §1), e a denúncia anônima nem tem sessão (`publicCall`, `AnonymousReportStep.tsx:37`). As duas telas já leem `GET /citizen/ombudsman/public_config` (`NewManifestationStep.tsx:83`, `AnonymousReportStep.tsx:37`). O plano lê o interruptor de lá: `public_config.zoonoses` (bool) = `Zoonoses::Gate.usable?` com a ouvidoria utilizável (**Q10**). O api aceita `zoonoses_subtype` e `place_address` nessa mesma condição. O `zoonoses` continua fora da sessão.
- **Cliente** (`src/lib/citizenApi.ts`): `OmbudsmanPublicConfig.zoonoses`, o tipo `ZoonosesSubtype` e a chave opcional `zoonoses` em `createManifestation` e `sendAnonymousReport`. As chaves vão só no corpo e só quando houver subtipo. A normalização da manifestação continua campo a campo: nada da demanda, da vistoria nem o `place_address` chega à tela.
- **Regras puras** (`src/lib/ombudsman.ts`): as frases fixas do §14, os oito subtipos do §4, a faixa do endereço (10–300) e as frases de `path` novas.
- **Telas:** `NewManifestationStep` ganha dois modos ("animal" e "subtype") entre o tipo e o modo de identificação, e o endereço com o aviso na tela do texto. `AnonymousReportStep` ganha as mesmas perguntas no próprio formulário: a pergunta é opcional, mas com subtipo o endereço é obrigatório (Q15). Nenhum componente comum (`ui.tsx`, `OmbudsmanFields.tsx`) muda.
- Os rótulos dos subtipos são os fixos do §4 no wpda: o cidadão manda o `code`, e renomear o assunto no dashboard não muda o canal (Q15).

**Tech Stack:** React 18 + TypeScript, Vitest 2 + Testing Library 16 + user-event 14 + jest-dom (jsdom, `src/test/setup.ts`), `vi.spyOn(citizenApi, …)` e `vi.stubGlobal("fetch")`, Playwright (5 perfis, `e2e/`), Vite 5 (base `/wpda/`).

**Spec:** `docs/superpowers/specs/2026-11-14-module-35-zoonoses-endemics-design.md` (decisão 31; Afeta "`apps/wpda`"; F-35.13); `docs/adr/0054.md`; o contrato `docs/superpowers/plans/2026-11-23-module-35b-zoonoses-contracts.md` §1, §4, §14, §16, §17 (roteiro 1 e 4), §18, Q6 e as decisões do §19 que valem aqui: **Q10** (interruptor no `public_config`), **Q13** (extrato só com o texto do cidadão) e **Q15** (assunto, endereço e rótulos). As Q-n prevalecem sobre o texto do §4 e do §14. Molde: `docs/superpowers/plans/2026-11-11-module-33-ombudsman-wpda.md` (Tasks 4, 5, 9 e 10).

## Global Constraints

- **Frases fixas** (contrato §14), copiadas sem mudança:
  - pergunta: "É sobre animais, insetos ou bichos?";
  - campo: "Onde fica o problema? (endereço do local)";
  - aviso: "Seu pedido vai direto ao centro de zoonoses. Não escreva seu nome nem telefone no texto."
- **Subtipos** (contrato §4), nesta ordem, `code` → rótulo:
  `scorpion` Escorpião · `bat` Morcego · `rodent` Rato ou outro roedor · `tick` Carrapato · `aggressive_animal` Animal que mordeu ou atacou · `neurological_animal` Animal com comportamento estranho ou paralisia · `sporotrichosis` Gato com feridas (esporotricose) · `other` Outro problema com animal.
- **Quando aparece:** só com `zoonosesOn` = `public_config.zoonoses === true` (Q10). Na nova manifestação, só nos tipos `request` (Solicitação) e `report` (Denúncia); na anônima, sempre que `zoonosesOn` (o `kind` continua `report`, Q6). Sem `zoonosesOn`, a tela e o corpo são exatamente os do 33.
- **Corpo:** `zoonoses_subtype` e `place_address` só no corpo, nunca em URL, storage ou console. Com subtipo, `place_address` é obrigatório nos três modos, inclusive na anônima (Q15): 10–300 code points depois do `trim` (`textLength`). Os dois vão sempre juntos. Na anônima, a pergunta continua opcional: sem subtipo, nenhuma das duas chaves vai, e o endereço nunca vai sozinho. Trocar o tipo para um que não aceita a pergunta, ou responder "Não", apaga subtipo e endereço do estado.
- **O cidadão nunca vê** a demanda, a vistoria, o assunto interno nem o encaminhamento (contrato §14; 33 W5). "Minhas manifestações" fica como no 33, sem o assunto, e o cidadão não vê de volta o `place_address` (Q15).
- **Herdado do 33:** uma pergunta por tela na nova manifestação; alvos de 48 px e texto de 18 px; cores só por papel do tema (`var(--warn-bg)`); guardas `inFlight` e `alive`; sem polling e sem `setTimeout` novo; texto só no estado do React; `publicCall` com `credentials: "omit"` na anônima; bloqueio por dispositivo intacto (`e2e/device-gate.spec.ts` verde).
- **Testes:** só `bin/test wpda -w wpda-mod35b <arquivos>` (de `$R`), nunca `vitest` direto. Tipos: `cd $W && npx tsc --noEmit`. E2e: `cd $W && WPDA_URL=http://curitiba.localhost:5222 npx playwright test <arquivos>`.
- **Git:** `/opt/homebrew/bin/git`; nunca `git add -A` nem `git add .`; commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Merge e push só com autorização explícita.
- **Board #1:** o wpda é superfície de F-35.13, com o api e o dashboard. O card vai a In Progress pelo plano do api; a Task 0 só confere. Done quando as três partes terminarem; Verified depois da prova.
- **Execução coordenada:** ao fim de cada task, relato de uma linha à sessão "Próximos módulos" (`SendMessage`).

## Ambiente de execução

- `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `G=/opt/homebrew/bin/git`, `W=$R/.worktrees/wpda-mod35b`, `A=$R/apps/api/.claude/mod35b`. Worktree **fora de `apps/`**, na branch `feat/mod-35b-zoonoses` a partir de `origin/main`. Caminhos das tasks relativos a `$W`.
- O api do 35b roda em dev na **3061** (container na rede `rota-saude_default`, apelido `api`). O Vite do worktree sobe num container dessa rede na **5222**, com `WPDA_ALLOW_DESKTOP=1` e `node_modules` em volume próprio (Task 4). Abra sempre `http://curitiba.localhost:5222/wpda/`, nunca `localhost`. O banco de dev só é migrado e semeado **com autorização do usuário** (plano do api).

## Review Focus

1. **Subtipo que sobra no corpo depois de mudar de ideia.** Cidadão escolhe Solicitação → Sim → Escorpião, volta e troca para Reclamação (ou responde Não): o corpo não leva `zoonoses_subtype` nem `place_address`. Teste: Task 2, "trocar o tipo ou responder Não apaga o subtipo e o endereço".
2. **Pergunta pulada porque o `public_config` chegou tarde.** Hoje a tela sai de "loading" só com `people()`; com o `public_config` lento, o "Continuar" do tipo pularia a pergunta. A tela espera as duas leituras (falha do `public_config` = desligado). Teste: Task 2, "public_config lento: a pergunta não é pulada".
3. **Interruptor desligado entre a leitura e o envio** (422 `invalid_request`, `path` `zoonoses_subtype`). A tela relê o `public_config`, tira a escolha, mostra a frase e mantém o texto. Teste: Task 2, "422 zoonoses_subtype: relê, tira a escolha e mantém o texto".
4. **Endereço do local que identifica a pessoa na anônima.** O endereço é do lugar (decisão 31) e, com subtipo, é obrigatório também na anônima (Q15); o aviso fixo fica à vista antes do envio, e nada de contato aparece. Teste: Task 3, "anônima com Sim: aviso antes do envio; corpo com subtipo e endereço; sem cookie".
5. **Demanda ou vistoria chegando a "Minhas manifestações"** (um api com defeito mandando `zoonoses_demand`, `inspection`, `place_address`, `subject`). Some na borda. Teste: Task 1, "manifestation: nada da demanda nem da vistoria".

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | card, âncoras, forma real do api, worktree, base | 0 |
| `src/lib/citizenApi.ts`, `src/lib/citizenApi.ombudsman.test.ts`, `src/lib/ombudsman.ts`, `src/lib/ombudsman.test.ts` | `zoonoses` no `public_config`, chaves novas nos dois envios, frases, subtipos, faixa do endereço | 1 |
| `src/modules/citizen/NewManifestationStep.tsx`, `src/modules/citizen/NewManifestationStep.test.tsx` | pergunta, subtipo, endereço e aviso na nova manifestação | 2 |
| `src/modules/citizen/AnonymousReportStep.tsx`, `src/modules/citizen/AnonymousReportStep.test.tsx`, `README.md` | as mesmas perguntas na anônima (endereço obrigatório com subtipo, Q15); parágrafo do módulo | 3 |
| `e2e/zoonoses.spec.ts` | os dois caminhos nos celulares | 4 |
| — | suíte, build, e2e, revisão, prova na 5222 | 5 |

**Estratégia de teste:**
- **Cliente e regras** (Task 1), com `fetch` falso e tabela de casos: o booleano do `public_config` (só `true` liga), os corpos exatos com e sem zoonoses, o descarte de campos da demanda e as faixas do endereço.
- **Telas** (Tasks 2 e 3), com `vi.spyOn(citizenApi, …)`: o caminho feliz, o desligado (tela e corpo iguais ao 33), a troca de ideia, o `public_config` lento ou recusado, cada 422 novo.
- **Playwright** (Task 4) nos perfis `iphone` e `pixel`, com rotas `/citizen/*` respondidas: identificada com escorpião e anônima com morcego, corpo exato, sem rolagem lateral, e o desligado.
- **Prova** (Task 5) contra o api 3061: os dois caminhos chegam à fila do centro no dashboard (5221) e o desligar pelo maintenance esconde a pergunta.

---

### Task 0: Card, âncoras, forma real do api, worktree e base

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do wpda (`cfa6675` ou mais novo); board #1 (`PVT_kwDOEbfGRc4BbxiB`, Status `PVTSSF_lADOEbfGRc4BbxiBzhWfnC0`); `$A`.
- Produces: `$W`; a base anotada; a confirmação de que o api expõe `public_config.zoonoses` (Q10) e a forma das recusas novas.

- [ ] **Step 1: Conferir o card F-35.13** (um `gh` simples por chamada, sem `$(...)` nem laço)

```bash
gh project item-list 1 --owner rotasaude --limit 600 --format json --jq '.items[] | select(.title | test("^F-35\\.13 ")) | {id, title, status}'
```

Expected: In Progress ou além. Em Todo, mova com `gh project field-list 1 --owner rotasaude --format json --jq '.fields[] | select(.name == "Status") | {id, options}'` e `gh project item-edit --id <PVTI_do_card> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <id_In_Progress>`. Card ausente: **pare e reporte**.

- [ ] **Step 2: Âncoras e o que o api do 35b fixou**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git; A=$R/apps/api/.claude/mod35b
$G -C $R/apps/wpda fetch -q origin && $G -C $R/apps/wpda log --oneline -3 origin/main
$G -C $R/apps/wpda show origin/main:src/lib/citizenApi.ts | grep -n 'export interface OmbudsmanPublicConfig\|function normalizePublicConfig\|sendAnonymousReport: async\|createManifestation: async'
$G -C $R/apps/wpda show origin/main:src/lib/ombudsman.ts | grep -n '^const PATH_MESSAGE\|^export const ANONYMOUS_WARNING'
$G -C $R/apps/wpda show origin/main:README.md | grep -n '^\*\*Ouvidoria (módulo 33).\*\*\|^### Relatório público'
grep -n 'CITIZEN_HIDDEN =' $A/app/services/platform/features.rb
grep -n 'zoonoses' $A/app/controllers/citizen_api/ombudsman/public_configs_controller.rb $A/app/controllers/citizen_api/sessions_controller.rb
grep -rn 'zoonoses_subtype\|place_address' $A/app/controllers/citizen_api/ombudsman/ | head -20
```

Expected:
- as âncoras do wpda nas linhas de `cfa6675` (ou deslocadas por outro módulo: use-as como estão);
- no api, `zoonoses` em `CITIZEN_HIDDEN`; `public_config` com a chave `zoonoses` (Q10); os dois controladores do 33 aceitando `zoonoses_subtype` e `place_address`.

Se o api ainda não tiver a chave no `public_config`, as Tasks 1–4 seguem com a forma da Q10 e a Task 5 espera; avise a sessão do api. Outra forma: **pare e reporte** ao coordenador.

- [ ] **Step 3: Worktree e base**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/wpda worktree add -q $R/.worktrees/wpda-mod35b -b feat/mod-35b-zoonoses origin/main
cd $R/.worktrees/wpda-mod35b && npm ci --silent && npx tsc --noEmit && echo tsc-ok
$R/bin/test wpda -w wpda-mod35b
```

Expected: verde e `tsc-ok`. **Anote** a base (arquivos e casos). A Task 5 espera o **mesmo número de arquivos de vitest**, com casos novos só nos quatro arquivos de teste das Tasks 1–3, e **+1 e2e** (`e2e/zoonoses.spec.ts`).

**Relato ao coordenador:** "wpda-mod35b Task 0: worktree e base verdes (N arquivos/M casos); `public_config.zoonoses` <presente | ausente, Task 5 espera>; F-35.13 em <estado>."

---

### Task 1: Cliente e regras — `zoonoses` no `public_config` e nos dois envios

**Files:**
- Modify: `src/lib/citizenApi.ts` (bloco "Ouvidoria (módulo 33)"), `src/lib/ombudsman.ts`
- Test: `src/lib/citizenApi.ombudsman.test.ts`, `src/lib/ombudsman.test.ts`

**Interfaces:**
- Consumes: `publicCall`, `call`, `rec`, `normalizePublicConfig`, `normalizeManifestation` (`citizenApi.ts`); `textLength`, `rangeError`, `PATH_MESSAGE` (`ombudsman.ts`).
- Produces:
  - `export type ZoonosesSubtype = "scorpion" | "bat" | "rodent" | "tick" | "aggressive_animal" | "neurological_animal" | "sporotrichosis" | "other"` (`citizenApi.ts`);
  - `OmbudsmanPublicConfig = { anonymous_web: boolean; zoonoses: boolean; units: { id: string; name: string }[] }`, com `zoonoses: d.zoonoses === true`;
  - `createManifestation(m: { …33…; zoonoses?: { subtype: ZoonosesSubtype; placeAddress: string } | null })`: com `zoonoses`, o corpo ganha `zoonoses_subtype` e `place_address` (aparado), depois de `body`;
  - `sendAnonymousReport(r: { body: string; healthUnitId: string | null; zoonoses?: { subtype: ZoonosesSubtype; placeAddress: string } | null })`: com `zoonoses`, ganha `zoonoses_subtype` e `place_address` (aparado), sempre juntos (Q15);
  - em `ombudsman.ts`: `ZOONOSES_QUESTION`, `ZOONOSES_ADDRESS_LABEL`, `ZOONOSES_NOTICE` (as três frases das Global Constraints), `ZOONOSES_SUBTYPES: readonly { value: ZoonosesSubtype; label: string }[]` (a ordem e os rótulos do §4), `YES_NO_OPTIONS = [{ value: "yes", label: "Sim" }, { value: "no", label: "Não" }]`, `PLACE_ADDRESS_MIN = 10`, `PLACE_ADDRESS_MAX = 300`, `placeAddressError(s: string): string | null` (= `rangeError(s, 10, 300)`), `zoonosesAllowed(kind: string | undefined): boolean` (`request` ou `report`), e em `PATH_MESSAGE`: `zoonoses_subtype: "O envio direto ao centro de zoonoses não está disponível agora. Sua manifestação pode seguir sem essa escolha."`, `place_address: "Escreva o endereço do local com 10 a 300 caracteres."`.

- [ ] **Step 1: Write the failing tests**

Em `src/lib/citizenApi.ombudsman.test.ts`, os dois `toEqual` do caso "publicConfig: sem cookie; …" ganham `zoonoses: false`, e entram estes casos (com os helpers `stubFetch`, `sent`, `headerNames` do topo):

| Caso | Asserções |
|---|---|
| "publicConfig: zoonoses só com true" | `{ zoonoses: "yes" }` → `zoonoses: false`; `{ zoonoses: true }` → `true`; sem a chave → `false` |
| "createManifestation: com zoonoses, subtipo e endereço aparado no corpo, nada na URL" | `{ who: { citizenId: "p1" }, kind: "request", mode: "identified", body: TEXT, healthUnitId: null, attachmentIds: [], zoonoses: { subtype: "scorpion", placeAddress: "  Rua das Flores, 120, fundos  " } }` → `sent(m).body` igual a `{ citizen_id: "p1", kind: "request", identification_mode: "identified", body: TEXT, zoonoses_subtype: "scorpion", place_address: "Rua das Flores, 120, fundos" }`; `sent(m).url` = `"/citizen/ombudsman/manifestations"` |
| "createManifestation: sem zoonoses (ou null), o corpo do 33" | as duas chamadas: o corpo não tem `zoonoses_subtype` nem `place_address` |
| "anonymousReport: com zoonoses, corpo exato e sem cookie" | `{ body: TEXT, healthUnitId: null, zoonoses: { subtype: "bat", placeAddress: "Praça Central, perto do coreto" } }` → `init.body` = `JSON.stringify({ body: TEXT, zoonoses_subtype: "bat", place_address: "Praça Central, perto do coreto" })`; `credentials` = `"omit"`; sem cabeçalho `cookie` |
| "anonymousReport: sem zoonoses (ou null), o corpo do 33" | as duas chamadas: `init.body` = `JSON.stringify({ body: TEXT })` |
| "manifestation: nada da demanda nem da vistoria" | resposta com os campos do 33 e mais `zoonoses_subtype`, `place_address`, `zoonoses_demand: { state: "scheduled" }`, `inspection: { note: "x" }`, `subject: { name: "Zoonoses" }` → o objeto devolvido tem exatamente as chaves do `CitizenManifestation` do 33 (`Object.keys(...).sort()` igual à lista dele); `JSON.stringify(r)` sem `/scheduled\|inspection\|Zoonoses\|place_address/` |

Em `src/lib/ombudsman.test.ts`, um `describe("zoonoses (módulo 35b)")`:
- `ZOONOSES_QUESTION`, `ZOONOSES_ADDRESS_LABEL` e `ZOONOSES_NOTICE` iguais às frases das Global Constraints (`toBe`);
- `ZOONOSES_SUBTYPES.map(s => s.value)` igual à ordem do §4, e `.map(s => s.label)` igual aos oito rótulos;
- `placeAddressError("Rua A, 1")` (8) = `"Escreva pelo menos 10 caracteres."`; `placeAddressError("  Rua A, 12  ")` (10 depois do trim) = `null`; `"a".repeat(301)` = `"Escreva no máximo 300 caracteres."`; `"🦂".repeat(300)` = `null`;
- `zoonosesAllowed`: `request` e `report` → `true`; `complaint`, `praise`, `suggestion`, `undefined` → `false`;
- `ombudsmanMessage(new ApiError(422, "invalid_request", "zoonoses_subtype"))` e o de `place_address` = as frases do Produces (`ApiError(status, code, path?, feature?)`, `citizenApi.ts:11`).

- [ ] **Step 2: Run tests to verify they fail**

Run (de `$R`): `bin/test wpda -w wpda-mod35b src/lib/citizenApi.ombudsman.test.ts src/lib/ombudsman.test.ts`
Expected: FAIL nos casos novos (`zoonoses` ausente do config; exports inexistentes); os do 33 passam, menos os dois `toEqual` ajustados.

- [ ] **Step 3: Implement** os tipos, a normalização e as chaves do Produces em `citizenApi.ts`, e as constantes e funções em `ombudsman.ts`, com o comentário de uma linha "Módulo 35b (contrato §4, §14): …". O comentário de `sendAnonymousReport` passa a dizer o corpo novo: `{ body }`, mais `health_unit_id` opcional e, com subtipo, `zoonoses_subtype` e `place_address` juntos (Q15). `normalizeManifestation` não muda (o caso novo prova que a demanda não passa).

- [ ] **Step 4: Run tests to verify they pass**

Run (de `$R`): `bin/test wpda -w wpda-mod35b src/lib/citizenApi.ombudsman.test.ts src/lib/ombudsman.test.ts && cd $W && npx tsc --noEmit`
Expected: PASS; `tsc` sem erro.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/citizenApi.ts src/lib/citizenApi.ombudsman.test.ts src/lib/ombudsman.ts src/lib/ombudsman.test.ts
$G commit -m "feat: send the zoonoses subtype and place address with ombudsman reports" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** "wpda-mod35b Task 1: cliente e regras de zoonoses (public_config.zoonoses, chaves nos dois envios, frases do §14); commit <sha>."

---

### Task 2: Nova manifestação — pergunta, subtipo, endereço e aviso

**Files:**
- Modify: `src/modules/citizen/NewManifestationStep.tsx`
- Test: `src/modules/citizen/NewManifestationStep.test.tsx`

**Interfaces:**
- Consumes: Task 1 (`ZoonosesSubtype`, `zoonoses` no config e em `createManifestation`, `ZOONOSES_*`, `YES_NO_OPTIONS`, `placeAddressError`, `zoonosesAllowed`, `PLACE_ADDRESS_MAX`); `Field`, `RadioGroup`, `Screen` de `ui.tsx`.
- Produces: o tipo `Mode` ganha `"animal"` e `"subtype"`. Sequência: … → `kind` → (`zoonosesOn && zoonosesAllowed(kind)`) `animal` → (Sim) `subtype` → `mode` → `text`.

Regras da tela:
- **Loading:** sai de `"loading"` só depois de `people()` **e** `ombudsmanPublicConfig()` terminarem; a falha do config continua "sem unidade, sem anônima" e passa a ser também `zoonosesOn = false`.
- **`animal`:** título e legenda = `ZOONOSES_QUESTION`; opções `YES_NO_OPTIONS`; sem escolha, "Escolha uma opção." no grupo. "Não" apaga subtipo e endereço e vai a `mode`.
- **`subtype`:** título "O que está acontecendo?", legenda "Assunto", opções `ZOONOSES_SUBTYPES`; sem escolha, "Escolha o assunto." no grupo.
- **`continueKind`** com um tipo fora de `zoonosesAllowed`, ou com `zoonosesOn` falso: apaga resposta, subtipo e endereço e vai a `mode`.
- **Voltar** de `mode`: a `subtype` (com Sim), a `animal` (com Não), ou a `kind`.
- **`text`** com subtipo: `<p style={warnNote}>{ZOONOSES_NOTICE}</p>` antes dos campos, e o `Field` com `label={ZOONOSES_ADDRESS_LABEL}` (ref próprio, `maxLength` não; `autoComplete="off"`) antes da unidade. No envio, `placeAddressError` local foca o campo e não manda nada.
- **Recusas novas:** `invalidPath(e, "place_address")` → erro no campo do endereço, com foco; `invalidPath(e, "zoonoses_subtype")` → relê o `public_config` (atualiza `zoonosesOn` e as unidades), apaga resposta, subtipo e endereço, mostra a frase em `error` e mantém o texto.

- [ ] **Step 1: Write the failing tests**

No `describe("NewManifestationStep")`, com o `stub` do topo recebendo um `CONFIG_ZOO = { ...CONFIG, zoonoses: true }` e o `CONFIG` do 33 ganhando `zoonoses: false`. Helper novo `answerAnimal(user, answer: "Sim" | "Não", subtype?: string)` (clica no rádio, Continuar, e no subtipo pelo rótulo exato, Continuar).

| Caso | Arranjo e ações | Asserções |
|---|---|---|
| "zoonoses ligado, Solicitação: pergunta, subtipo, aviso e endereço vão no corpo" | `[P1]`, `CONFIG_ZOO`; Solicitação → Sim → Escorpião → Identificada → endereço `"  Rua das Flores, 120, fundos  "` → texto → Enviar | `heading` "É sobre animais, insetos ou bichos?" visto; `getByText(ZOONOSES_NOTICE)` antes do botão "Enviar manifestação" (`compareDocumentPosition`); `create` chamado com `expect.objectContaining({ kind: "request", zoonoses: { subtype: "scorpion", placeAddress: "Rua das Flores, 120, fundos" } })`; tela "Manifestação registrada" |
| "Denúncia também pergunta; Reclamação, Elogio e Sugestão não" | `CONFIG_ZOO`; para cada um dos três tipos, Continuar | Denúncia → `heading` da pergunta; os outros três → `heading` "Como você quer se identificar?" direto |
| "zoonoses desligado: a tela e o corpo do 33" | `CONFIG` (sem zoonoses); Solicitação → Continuar | vai a "Como você quer se identificar?"; `queryByText(ZOONOSES_QUESTION)` nulo; `create.mock.calls[0][0].zoonoses` `undefined` ou `null` |
| "Não: segue sem subtipo e sem endereço" | `CONFIG_ZOO`; Solicitação → Não → … → Enviar | sem a tela de subtipo; sem o campo do endereço; sem o aviso; `zoonoses` ausente ou nulo no `create` |
| "trocar o tipo ou responder Não apaga o subtipo e o endereço" | `CONFIG_ZOO`; Solicitação → Sim → Morcego → Identificada → digita endereço → Voltar até o tipo → Reclamação → Continuar → … → Enviar | `create` sem `zoonoses`; e, numa segunda renderização, Sim → Morcego → Voltar → Não → … → Enviar, também sem |
| "endereço obrigatório: curto ou vazio, erro no campo com foco e nada enviado" | `CONFIG_ZOO`; Sim → Rato ou outro roedor → endereço `"Rua A"` → texto válido → Enviar | `getByRole("textbox", { name: ZOONOSES_ADDRESS_LABEL })` com `aria-invalid="true"` e foco; texto "Escreva pelo menos 10 caracteres."; `create` não chamado |
| "sem escolha na pergunta e no subtipo: erro no grupo" | Continuar sem escolher em cada tela | "Escolha uma opção." e "Escolha o assunto." |
| "public_config lento: a pergunta não é pulada" | `configSpy` com uma promessa resolvida só depois de `people()` | enquanto pendente, `role="status"` "Carregando…" e nenhum rádio de tipo; depois de resolver com `CONFIG_ZOO`, Solicitação → Continuar mostra a pergunta |
| "public_config recusado: sem a pergunta" | `configSpy.mockRejectedValue(new ApiError(403, "feature_disabled"))` | Solicitação → Continuar vai ao modo; sem a pergunta |
| "422 place_address: erro no campo do endereço, com foco" | `create.mockRejectedValue(new ApiError(422, "invalid_request", "place_address"))` | frase de `PATH_MESSAGE.place_address` no campo, com foco; o texto continua |
| "422 zoonoses_subtype: relê, tira a escolha e mantém o texto" | `create` rejeita com `path` `zoonoses_subtype` e depois resolve; `configSpy` responde `CONFIG_ZOO` e depois `CONFIG` | `configSpy` chamado 2 vezes; frase de `PATH_MESSAGE.zoonoses_subtype` visível; o campo do endereço e o aviso somem; o texto continua; Enviar de novo chama `create` sem `zoonoses` |

- [ ] **Step 2: Run tests to verify they fail**

Run (de `$R`): `bin/test wpda -w wpda-mod35b src/modules/citizen/NewManifestationStep.test.tsx`
Expected: FAIL nos casos novos; os do 33 passam.

- [ ] **Step 3: Implement** os modos, o estado (`zoonosesOn`, `aboutAnimals: "yes" | "no" | undefined`, `subtype: ZoonosesSubtype | undefined`, `placeAddress`, `addressError`, `animalError`, `subtypeError`) e as regras acima em `NewManifestationStep.tsx`. O `warnNote` é o estilo de `AnonymousReportStep.tsx:15` (copie a constante; nada em `ui.tsx` muda). O comentário do topo ganha uma linha: "Módulo 35b (contrato §4, §14): com zoonoses, Solicitação e Denúncia perguntam se é sobre animais; subtipo e endereço do local vão ao registro do 33."

- [ ] **Step 4: Run tests to verify they pass**

Run (de `$R`): `bin/test wpda -w wpda-mod35b src/modules/citizen/NewManifestationStep.test.tsx src/modules/citizen/Flow.ombudsman.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS (o `Flow.ombudsman` prova que a navegação do 33 não mudou); `tsc` sem erro.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/citizen/NewManifestationStep.tsx src/modules/citizen/NewManifestationStep.test.tsx
$G commit -m "feat: ask about animals in new ombudsman requests and reports" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** "wpda-mod35b Task 2: nova manifestação com a pergunta, o subtipo, o endereço obrigatório e o aviso; desligado igual ao 33; commit <sha>."

---

### Task 3: Denúncia anônima — as mesmas perguntas, endereço obrigatório com subtipo; README

**Files:**
- Modify: `src/modules/citizen/AnonymousReportStep.tsx`, `README.md` (fim do parágrafo **Ouvidoria (módulo 33).**, antes de `### Relatório público`)
- Test: `src/modules/citizen/AnonymousReportStep.test.tsx`

**Interfaces:**
- Consumes: Task 1 (`zoonoses` no config e em `sendAnonymousReport`, `ZOONOSES_*`, `YES_NO_OPTIONS`, `placeAddressError`); `RadioGroup`, `Field` de `ui.tsx`.
- Produces: no formulário, com `config.zoonoses`, depois da unidade e antes do texto:
  - `RadioGroup` com legenda `ZOONOSES_QUESTION` e `YES_NO_OPTIONS`, sem valor inicial (sem resposta = Não);
  - com Sim: `<p style={warnNote}>{ZOONOSES_NOTICE}</p>`, `RadioGroup` com legenda "Assunto" e `ZOONOSES_SUBTYPES`, e `Field` com `label={ZOONOSES_ADDRESS_LABEL}`, sem dica de opcional (com subtipo, é obrigatório, Q15);
  - "Não" apaga subtipo e endereço.
- Envio (Q15): com Sim e subtipo → `placeAddressError` local (vazio ou curto foca o campo e não envia) e `zoonoses: { subtype, placeAddress: <aparado> }`; endereço preenchido sem subtipo → "Escolha o assunto." no grupo e nada enviado (o endereço nunca vai sozinho); Sim sem subtipo e sem endereço → envia sem `zoonoses` (a pergunta continua opcional). Recusas `place_address` e `zoonoses_subtype` como na Task 2 (a segunda relê o config e tira a escolha).

- [ ] **Step 1: Write the failing tests**

No `describe("AnonymousReportStep")`, com `CONFIG_ZOO = { ...CONFIG, zoonoses: true }` e o `CONFIG` do 33 ganhando `zoonoses: false`:

| Caso | Asserções |
|---|---|
| "anônima com Sim: aviso antes do envio; corpo com subtipo e endereço; sem cookie" | com `fetch` falso (molde do caso "no fio: …") e `CONFIG_ZOO`: Sim → Morcego → endereço `"Praça Central, perto do coreto"` → texto → Enviar; `getByText(ZOONOSES_NOTICE)` antes do botão; `init.body` = `JSON.stringify({ body: TEXT, zoonoses_subtype: "bat", place_address: "Praça Central, perto do coreto" })`; `credentials` `"omit"`; tela "Denúncia enviada" sem protocolo |
| "anônima com subtipo e sem endereço: erro no campo com foco e nada enviado (Q15)" | Sim → Carrapato → texto válido → Enviar: o campo `ZOONOSES_ADDRESS_LABEL` com `aria-invalid="true"` e foco; "Escreva pelo menos 10 caracteres."; `send` não chamado |
| "anônima com Sim e sem subtipo nem endereço: envia sem zoonoses" | `send` chamado com `{ body: TEXT, healthUnitId: null }` (sem `zoonoses`, ou com `zoonoses` nulo) |
| "endereço sem subtipo: pede o assunto e não envia" | "Escolha o assunto." no grupo; `send` não chamado |
| "endereço curto: erro no campo com foco" | `"Rua A"` → "Escreva pelo menos 10 caracteres."; `send` não chamado |
| "Sim e depois Não: nada de zoonoses no corpo" | `send` sem `zoonoses` (ausente ou nulo); o campo do endereço some |
| "zoonoses desligado: o formulário e o corpo do 33" | `CONFIG`: `queryByText(ZOONOSES_QUESTION)` nulo; `getAllByRole("textbox")` com 1; `send` sem `zoonoses` |
| "com Sim, nenhum campo de contato" | `CONFIG_ZOO`, Sim: `queryByLabelText(/telefone\|celular\|e-mail\|nome\|cpf/i)` nulo; `getAllByRole("textbox")` com 2 (endereço e texto) |
| "422 zoonoses_subtype: relê, tira a escolha e mantém o texto" | `ombudsmanPublicConfig` chamado 2 vezes; frase de `PATH_MESSAGE.zoonoses_subtype`; texto continua; a pergunta some se o config relido vier sem `zoonoses` |

- [ ] **Step 2: Run tests to verify they fail**

Run (de `$R`): `bin/test wpda -w wpda-mod35b src/modules/citizen/AnonymousReportStep.test.tsx`
Expected: FAIL nos casos novos; os do 33 passam.

- [ ] **Step 3: Implement** as regras do Produces em `AnonymousReportStep.tsx`; o comentário do topo troca "o corpo é exatamente { body } ou { body, health_unit_id }" por "o corpo é { body }, com health_unit_id opcional e, com subtipo, zoonoses_subtype e place_address juntos (35b, contrato §4, Q15)".

README, no fim do parágrafo **Ouvidoria (módulo 33).**, um parágrafo **Zoonoses na ouvidoria (módulo 35b).** com:
- com o centro de zoonoses ligado na cidade (`public_config.zoonoses`, Q10), Solicitação e Denúncia perguntam "É sobre animais, insetos ou bichos?"; com "Sim", o assunto (os oito do §4) e o campo obrigatório "Onde fica o problema? (endereço do local)", com o aviso "Seu pedido vai direto ao centro de zoonoses. Não escreva seu nome nem telefone no texto.";
- na denúncia anônima, as mesmas perguntas; a pergunta é opcional, mas com o assunto escolhido o endereço é obrigatório, como na manifestação;
- o api encaminha sozinho ao centro; "Minhas manifestações" mostra só o que o 33 mostra, nunca a demanda nem a vistoria;
- desligado, nada muda; contra um api sem o 35b, a pergunta só não aparece;
- ordem de deploy: contracts → api → dashboard, wpda e maintenance → campo.

- [ ] **Step 4: Run tests to verify they pass**

Run (de `$R`): `bin/test wpda -w wpda-mod35b src/modules/citizen/AnonymousReportStep.test.tsx && cd $W && npx tsc --noEmit`
Expected: PASS; `tsc` sem erro.

- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/citizen/AnonymousReportStep.tsx src/modules/citizen/AnonymousReportStep.test.tsx README.md
$G commit -m "feat: let anonymous reports reach the zoonoses center" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** "wpda-mod35b Task 3: denúncia anônima com as perguntas (endereço obrigatório com subtipo, Q15), README; commit <sha>."

---

### Task 4: Playwright — os dois caminhos nos celulares

**Files:**
- Create: `e2e/zoonoses.spec.ts`

**Interfaces:**
- Consumes: o molde de `e2e/ombudsman.spec.ts` (`PHONES`, `noOverflow`, `expectNoCookie`, `timezoneId: "UTC"`, `page.clock.setFixedTime`, rotas `/citizen/*` respondidas); `playwright.config.ts` (`WPDA_URL`).
- Produces: a prova no aparelho de que a identificada (Solicitação de escorpião) e a anônima (Denúncia de morcego) levam o subtipo e o endereço no corpo, sem rolagem lateral, e de que o desligado não mostra a pergunta.

- [ ] **Step 1: Suba o Vite do worktree** (fica no ar até a Task 5)

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude
docker run -d --rm --name wpda-mod35b --network rota-saude_default -p 5222:5222 \
  -v $R/.worktrees/wpda-mod35b:/app -v wpda-mod35b-node-modules:/app/node_modules -w /app \
  -e VITE_API_PROXY_TARGET=http://api:3061 -e WPDA_ALLOW_DESKTOP=1 node:22-alpine \
  sh -c "npm ci --silent --no-audit --no-fund && npx vite --port 5222 --host 0.0.0.0"
docker logs -f wpda-mod35b   # espere o "ready"
```

- [ ] **Step 2: Escreva o teste.** `mockApi(page, { signedIn: boolean; zoonoses: boolean })` responde:
  - `/session`: com `signedIn`, `{ phone_masked: "(41) *****-4321", time_zone: "America/Sao_Paulo", features: [ "ombudsman", "ombudsman_anonymous_web" ] }`; sem, 401 `unauthenticated` (guardando o pedido, controle positivo do cookie);
  - `/people`: `{ people: [ { id: "p1", cpf_masked: "***.982.247-**", verification_level: "declared", neighborhood: null, profile: null } ] }`;
  - `/notices`: `{ notices: [], unread_count: 0 }`; `/ombudsman/manifestations` GET: `{ manifestations: [] }`;
  - `/ombudsman/public_config`: `{ anonymous_web: true, zoonoses: <opts.zoonoses>, units: [ { id: "u1", name: "UBS Centro" } ] }`;
  - POST `/ombudsman/manifestations` → 201 `{ id: "m1", protocol: "2026-000321", due_on: "2026-12-23" }` (guarda o pedido); POST `/ombudsman/anonymous_reports` → 201 `{ ok: true }` (guarda); o resto, 404 `not_found`.

  Três testes, só em `iphone` e `pixel`, relógio em `2026-11-23T10:00:00-03:00`:

| Teste | Caminho | Asserções |
|---|---|---|
| "Solicitação de escorpião: pergunta, assunto, endereço e aviso; corpo exato" | `signedIn`, `zoonoses: true`; `goto("/wpda/ouvidoria")` → "Nova manifestação" → Solicitação → Continuar → Sim → Continuar → Escorpião → Continuar → Identificada → Continuar → endereço `"Rua das Flores, 120, fundos"` → texto `"Tem escorpião saindo do bueiro da calçada todas as noites."` → Enviar | `getByText(ZOONOSES_NOTICE)` visível na tela do texto; 1 POST de manifestação, `search` vazio, `postDataJSON()` igual a `{ citizen_id: "p1", kind: "request", identification_mode: "identified", body: <texto>, zoonoses_subtype: "scorpion", place_address: "Rua das Flores, 120, fundos" }`; "Manifestação registrada"; `noOverflow` ≤ 0 nas telas da pergunta, do assunto e do texto |
| "Denúncia anônima de morcego: assunto e endereço, corpo exato, sem cookie" | sem `signedIn`, `zoonoses: true`, cookie `citizen_session=plantado`; `goto("/wpda/")` → "Denúncia anônima" → Sim → Morcego → endereço `"Praça Central, perto do coreto"` → texto → Enviar | aviso fixo visível antes do botão; `postDataJSON()` igual a `{ body: <texto>, zoonoses_subtype: "bat", place_address: "Praça Central, perto do coreto" }`; `expectNoCookie(report, session)` (o mesmo tratamento do WebKit do 33); "Denúncia enviada", sem protocolo; `noOverflow` ≤ 0 |
| "Zoonoses desligado: nenhuma pergunta nos dois caminhos" | `zoonoses: false`; anônima sem login e Solicitação logado | `getByText("É sobre animais, insetos ou bichos?")` com `toHaveCount(0)` nos dois; os corpos sem `zoonoses_subtype` |

As frases ficam como constantes no topo do arquivo, copiadas das Global Constraints (o e2e não importa `src/`).

- [ ] **Step 3: Run** `cd $W && WPDA_URL=http://curitiba.localhost:5222 npx playwright test e2e/zoonoses.spec.ts e2e/ombudsman.spec.ts e2e/device-gate.spec.ts`
Expected: PASS em `iphone` e `pixel`; os outros perfis pulam com o motivo; `ombudsman` e `device-gate` verdes.

- [ ] **Step 4: Commit**

```bash
cd $W && $G add e2e/zoonoses.spec.ts
$G commit -m "test: cover zoonoses ombudsman requests and anonymous reports on phone profiles" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

**Relato ao coordenador:** "wpda-mod35b Task 4: e2e dos dois caminhos verde no iphone e no pixel (corpo exato, sem cookie na anônima, sem rolagem); ombudsman e device-gate verdes."

---

### Task 5: Suíte, build, e2e, revisão e prova no navegador

**Files:** nenhum novo.

- [ ] **Step 1: Suíte, tipos, build e e2e**

Run (de `$R`): `bin/test wpda -w wpda-mod35b && cd $W && npx tsc --noEmit && npm run build && WPDA_URL=http://curitiba.localhost:5222 npx playwright test && $G status --short && $G log --stat origin/main..HEAD | grep -c node_modules`

Expected: verde, com o mesmo número de arquivos de vitest da base e só os casos novos; e2e verde; `status` vazio; contagem `0`. Se o número de testes dobrar, é artefato de build: pare e reporte.

- [ ] **Step 2: Nada vaza**

```bash
cd $W
grep -n "console\.\|localStorage\|sessionStorage\|setTimeout\|setInterval" src/modules/citizen/NewManifestationStep.tsx src/modules/citizen/AnonymousReportStep.tsx src/lib/ombudsman.ts
grep -n 'place_address\|zoonoses_subtype' src/lib/citizenApi.ts | grep -n '`/\|?'
```

Expected: sem saída nos dois.

- [ ] **Step 3: Revisão.** Um subagente revisor lê `origin/main..HEAD` contra a spec (decisão 31, F-35.13) e o contrato §4, §14 e Q6. Ele confere: as três frases idênticas às do §14; os oito subtipos e a ordem do §4; a pergunta só em `request`/`report` e só com `zoonosesOn`; o endereço obrigatório com subtipo nos três modos e nunca sozinho (Q15); nenhuma chave nova com o interruptor desligado; nada da demanda na normalização; `credentials: "omit"` intacto; `ui.tsx` e `OmbudsmanFields.tsx` sem mudança; cores só por papel do tema.

- [ ] **Step 4: Prova no navegador na 5222 (com o usuário)**

Depende do api do 35b na **3061**, com a semente `lib/zoonoses_crew.rb` (contrato §17): Curitiba com `ombudsman`, `ombudsman_anonymous_web` e `zoonoses` ligados, a "UVZ Curitiba" ativa e `zoonoses@curitiba.demo`. O Vite da Task 4 segue no ar. Abra `http://curitiba.localhost:5222/wpda/` a 375 px, com toque. Screenshots em `evidencias/2026-11-23-modulo-35b/wpda/`. Roteiro (contrato §17, passos 1 e 4):
1. **Solicitação de escorpião** (logado, código SMS de dev): pergunta, assunto, aviso e endereço; anote o protocolo. Na rede (`read_network_requests`), o POST `/citizen/ombudsman/manifestations` leva `zoonoses_subtype` e `place_address` só no corpo.
2. **Denúncia anônima de morcego**, sem login, com endereço: o POST `/citizen/ombudsman/anonymous_reports` vai sem `Cookie`, com as duas chaves.
3. **Fila do centro** (dashboard 5221, com a sessão do dashboard e `zoonoses@`): as duas demandas aparecem com o subtipo, o prazo e a cor; o extrato (só o texto do cidadão, Q13) e o endereço saem no detalhe. A UVZ fecha a de escorpião com a resposta; a ouvidoria responde ao cidadão.
4. **"Minhas manifestações"**: a de escorpião mostra o estado e a "Resposta da Ouvidoria" do 33; nenhuma resposta `/citizen/ombudsman/*` contém `zoonoses_demand`, `inspection`, `scheduled` nem o nome do assunto.
5. **Desligar** (maintenance 5177, na prova do plano do maintenance, ou com a sessão dele): com `zoonoses` desligado, a pergunta some nos dois caminhos e o corpo volta ao do 33; religue.
6. Alto contraste legível; a 375 px, nenhuma tela com rolagem lateral.

Ao terminar: `docker rm -f wpda-mod35b && docker volume rm wpda-mod35b-node-modules`.

- [ ] **Step 5: Pare.** O merge vem depois do api do 35b (suíte completa pela sessão "Testes", pedida pelo plano do api) e só com autorização explícita (merge, push e card, uma etapa de cada vez). Antes do push, `origin/main..main` só pode ter os commits desta entrega. Quando autorizado: `fetch`, `merge --ff-only feat/mod-35b-zoonoses`, `push origin main`, `worktree remove $W`, `branch -d`. F-35.13 vai a Done com as partes do api e do dashboard, e a Verified se a prova passou.

**Relato ao coordenador:** "wpda-mod35b Task 5: suíte/build/e2e verdes, revisão limpa, prova na 5222 <feita/pendente: …> (evidências em evidencias/2026-11-23-modulo-35b/wpda/); aguardando autorização de merge."

---

## Divergências propostas ao contrato

> Todas decididas no §19 do contrato: D1 e D2 pela **Q10**; D3 a D6 pela **Q15**. O plano acima já aplica as decisões.

1. **D1 — Como o wpda lê `zoonoses` se ele está em `CITIZEN_HIDDEN`.** O §1 manda usar "o mesmo caminho pelo qual o wpda já lê `ombudsman`". Esse caminho é a sessão: `hasSessionFeature(OMBUDSMAN_FEATURE)` (`apps/wpda/src/modules/citizen/Flow.tsx:115,281`), e a sessão sai **sem** as chaves de `CITIZEN_HIDDEN` (`apps/api/app/controllers/citizen_api/sessions_controller.rb:44-45`). Com `zoonoses` lá, o wpda nunca veria a chave. Além disso, a denúncia anônima não tem sessão: ela só lê `GET /citizen/ombudsman/public_config` (`AnonymousReportStep.tsx:37`; `public_configs_controller.rb:21-25`).
   - **Proposta:** `public_config` ganha `zoonoses: bool`, e `zoonoses` fica em `CITIZEN_HIDDEN` (fora da sessão). As duas telas já leem o `public_config` (`NewManifestationStep.tsx:83`, `AnonymousReportStep.tsx:37`), que é como o wpda já sabe de `anonymous_web`. O plano do api ganha a chave e um request spec.
   - **Resolvida pela Q10:** `public_config.zoonoses` = `Zoonoses::Gate.usable?` com a ouvidoria utilizável; `zoonoses` fica fora da sessão.
2. **D2 — "Ligado" ou "utilizável".** O §4 recusa a chave "com o `zoonoses` desligado", e o §14 fala em "ligado na cidade". Só que `Zoonoses::Gate.usable?` também exige o cadastro territorial (§1), e o wpda mostraria a pergunta numa cidade em que o encaminhamento não age.
   - **Proposta:** `public_config.zoonoses` = `Zoonoses::Gate.usable?(city)`, e o api aceita `zoonoses_subtype` sob a **mesma** condição (ligado e não utilizável → 422 `invalid_request`, `path` `zoonoses_subtype`). O plano trata esse 422 (Task 2 e 3).
   - **Resolvida pela Q10:** a mesma condição do `public_config` (gate utilizável e ouvidoria utilizável) vale para o api aceitar `zoonoses_subtype` e `place_address`.
3. **D3 — "Minhas manifestações: mostra o assunto".** O 33 fixou que o assunto interno nunca chega ao cidadão: `apps/wpda/src/lib/citizenApi.ts:842` ("o assunto … nunca passa da borda"), `ManifestationDetailStep.tsx:2`, e o caso "manifestation: só os campos do contrato" de `citizenApi.ombudsman.test.ts:117`, que descarta `subject`. O api também não manda o assunto ao cidadão.
   - **Proposta:** ler o §14 como "mostra o tipo e o andamento do 33", sem o assunto. É o que o plano faz. Se o coordenador quiser que o cidadão veja o que escolheu, o api manda `zoonoses_subtype` (o código) na lista e no detalhe, e o wpda mostra o rótulo fixo do §4, nunca o nome do assunto (que é renomeável). Isso vira uma task a mais aqui.
   - **Resolvida pela Q15:** "Minhas manifestações" segue o 33, sem o assunto (corrige o §14). Nenhuma task a mais.
4. **D4 — Endereço na denúncia anônima.** O §14 diz "as mesmas duas perguntas, opcionais", e o §4 só torna o endereço obrigatório na identificada e na preservada. Fica aberto: com o subtipo escolhido na anônima, o endereço continua opcional? E o endereço pode ir sem subtipo?
   - **Proposta:** com o subtipo, o endereço é obrigatório nos três modos, porque sem lugar a UVZ não vistoria.
   - **Resolvida pela Q15:** com subtipo, o `place_address` é obrigatório nos três modos, inclusive na anônima (corrige o §4 e o §14). A Task 3 aplica isso; a pergunta continua opcional, e o endereço nunca vai sem subtipo.
5. **D5 — Rótulos dos subtipos.** O §4 deixa renomear o assunto com `code`. O wpda usa os rótulos fixos do §4 (Task 1), porque o cidadão manda só o código e o `public_config` não traz a lista. Renomear no dashboard não muda o canal.
   - **Proposta:** registrar isso no §14. A alternativa é o `public_config` mandar `zoonoses_subtypes: [{ code, label }]`. Ela não é recomendada, porque um rótulo editável chegaria sem revisão de linguagem ao cidadão.
   - **Resolvida pela Q15:** rótulos fixos no wpda, com a lista do §4.
6. **D6 — O cidadão vê o endereço que mandou?** O contrato não diz. O plano não mostra: a normalização do detalhe descarta `place_address` (Task 1, Review Focus 5).
   - **Proposta:** confirmar no §14 que o detalhe do cidadão não traz o `place_address`. O texto dele (`body`) continua visível, como no 33.
   - **Resolvida pela Q15:** o cidadão não vê de volta o `place_address`.

## Self-review

- **Cobertura:**
  - §14: pergunta, subtipo e endereço obrigatório (Task 2);
  - o aviso (Tasks 2 e 3);
  - a anônima com a pergunta opcional e o endereço obrigatório com subtipo (Task 3; Q15);
  - desligado igual ao 33 (Tasks 2, 3 e 4);
  - "Minhas manifestações" sem demanda, sem assunto e sem o endereço de volta (Task 1, Task 5 passo 4; Q15);
  - e2e nos celulares do 33 (Task 4).
  - §4: os tipos aceitos, a anônima continuar `report` (Q6), a recusa com o interruptor desligado (Tasks 2 e 3) e o encaminhamento conferido na prova.
  - §16: o `place_address` só no corpo (Task 1 e grep da Task 5).
  - §17, passos 1 e 4: Task 5.
- **Placeholders:** nenhum. Valores lidos na execução: o id do card e a presença de `public_config.zoonoses` no api (Task 0).
- **Tipos e nomes:** `ZoonosesSubtype`, `OmbudsmanPublicConfig.zoonoses`, `ZOONOSES_QUESTION`, `ZOONOSES_ADDRESS_LABEL`, `ZOONOSES_NOTICE`, `ZOONOSES_SUBTYPES`, `YES_NO_OPTIONS`, `placeAddressError` e `zoonosesAllowed` nascem na Task 1, e as Tasks 2–4 os usam com esses nomes. `invalidPath`, `isCode`, `warnNote`, `stub`, `stubFetch`, `sent` e `expectNoCookie` existem em `cfa6675`.
- **Proporção:** a mudança de produção é um campo e duas chaves no cliente, dez constantes e duas telas. O plano traz os testes em tabelas, não em código.
