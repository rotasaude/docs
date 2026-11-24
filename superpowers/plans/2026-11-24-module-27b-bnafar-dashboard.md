# Módulo 27b — Envio à BNAFAR e saldo inicial (dashboard) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **Pré-requisitos:**
> 1. O **contrato** `2026-11-24-module-27b-bnafar-contracts.md` (fonte única), com as decisões Q6–Q18 do §11 (docs `ae1cfbc`). As divergências D1–D10 deste plano foram resolvidas por elas (seção final). D11 e D12 são novas e esperam decisão.
> 2. A spec, com a decisão 6 revista pela Q11: a importação do saldo inicial é feita pela cidade, no dashboard.
> 3. O **api do 27b** na branch `feat/mod-27b-bnafar` (`apps/api/.claude/mod27b`, porta **3062**). As Tasks 1–11 seguem pelas formas deste plano e pelas fixtures. Só a Task 12 (prova) depende do api rodando.
> 4. A `main` do dashboard **com o 35b mergeado**. O contrato diz que o 27b começa depois dele; quando este plano foi escrito, o `origin/main` estava em `2e3a2a9`, só com o 35a.
>
> Ordem do contrato §10: api → **dashboard** e maintenance → suíte completa → prova contra o simulado.

**Goal:** Lado dashboard do 27b. No grupo **Farmácia**, quatro telas:
- "Envio à BNAFAR";
- "Pendências do envio à BNAFAR", que inclui o complemento dos lotes;
- "Saída externa";
- "Saldo inicial", a importação da planilha feita pela cidade (Q11).

Também entram:
- os campos do envio na entrada e na abertura (medicamento, vacina e produto de campo), com o AMPP escolhido entre as apresentações do VMP e o fabricante internacional;
- o CNES do almoxarifado central e o da rede de frio;
- o CATMAT dos produtos de endemias;
- a credencial `bnafar` e o teste de conexão em Integrações;
- a lista das aberturas do local, com o autor.

**Architecture:** O cliente ganha a seção "Envio à BNAFAR, saída externa e saldo inicial (módulo 27b)" no fim de `src/lib/api.ts`. As formas seguem o contrato (Q8, Q9, Q11, Q13, Q15, Q16, Q17) e o plano do api (`2026-11-24-module-27b-bnafar-api.md`, Valores 8–11). O que nenhum dos dois fixa está marcado como proposta, e a Task 0 confere tudo contra o api real.

A seção estende os tipos de módulos anteriores:
- do 16: `CredentialKind` e `IntegrationCredential`;
- do 27a: `StockLot`, `StockItem`, `ReceiptItemInput`, `StockMovementKind` e `AdminLocation`;
- do 29: `VaccineReceiptInput` e o catálogo;
- do 35a: `VectorProduct`.

Onde fica cada coisa:
- portões, chaves, regras de tela e rótulos: `src/lib/bnafar.ts`;
- as frases do envio: uma tabela única, `src/lib/bnafarErrors.ts` (`BNAFAR_ERRORS`, contrato §7);
- as regras e frases da planilha: `src/lib/openingImport.ts`;
- as telas: raízes em `src/modules/Bnafar*.tsx`, `src/modules/ExternalExits.tsx` e `src/modules/OpeningImport.tsx`, e painéis em `src/modules/bnafar/`, `src/modules/openingImport/` e `src/modules/pharmacy/`.

O dashboard não decide o que o api calcula: pendência, alerta, prazo de reenvio, prazo de correção, divisão em registros e remessas, DM01/DM12/DM13, validação de linha da planilha e conversão de unidade. Mostra o que vem e só não oferece nem envia o que já sabe que será recusado.

**Tech Stack:** React 18 + TypeScript, TanStack Query v5, Vitest 2 + Testing Library 16 (jsdom, sem jest-dom, `vi.mock("…/lib/api")`), Vite 5 (base `/dashboard/`), `@tabler/icons-react` 3.

**Spec:** `docs/superpowers/specs/2026-11-24-module-27b-bnafar-design.md` (decisão 6 revista; decisões 8–14; §4.1–§4.6). **Contrato:** `docs/superpowers/plans/2026-11-24-module-27b-bnafar-contracts.md` §1–§7, §9 e Q1–Q18. **Molde:** `docs/superpowers/plans/2026-11-23-module-35b-zoonoses-dashboard.md`.

## Global Constraints

- **Nomes no código** (contrato §1, §4, §6, §7, Q6, Q11):
  - interruptores `bnafar`, `bnafar_medications`, `bnafar_vector_supplies`, `bnafar_vaccines` e `bnafar_mock`;
  - credencial `bnafar` com `environment` `production` | `homologation`;
  - telas `bnafar-status`, `bnafar-pending`, `external-exits`, `opening-import`;
  - movimentos `external_exit` (documento `stock_external_exit`) e `opening_reversal` (Q10).

  **Código desconhecido aparece cru, nunca some.**
- **Frases fixas** (letra por letra):
  - os rótulos curtos dos interruptores, "Envio à BNAFAR", "BNAFAR: medicamentos", "BNAFAR: endemias e zoonoses" e "BNAFAR: vacinas", e "BNAFAR simulado" (Q6);
  - os itens "Envio à BNAFAR", "Pendências do envio à BNAFAR" (Q16), "Saída externa" e "Saldo inicial" (Q11, pelo coordenador);
  - os botões "Reenviar recusados" e "Corrigir";
  - as telas existentes "Entradas e abertura" e "Locais e configurações".

  Toda outra frase deste plano é **proposta** e está marcada assim. O código e a frase de cada pendência e de cada recusa MSG ficam em `BNAFAR_ERRORS`.
- **Portões** (contrato §6 e Q16):
  - `canUseBnafar(user)` = não operador e `bnafar` na sessão. A sessão traz as chaves **ligadas**, não as utilizáveis (`src/lib/features.ts:1-4`), e o api responde `403 feature_disabled` quando falta a credencial ou o IBGE.
  - Status, remessas e pendências: leem o `municipal_admin` e o `stock_keeper`.
  - Reenvio e correção: só o `municipal_admin`.
  - Complemento do lote, dentro de "Pendências do envio à BNAFAR" (Q16): `stock_keeper` ou `pharmacist` do local, ou `municipal_admin`. A rota não exige `bnafar` (§6), mas a tela, sim.
  - Saída externa (Q16): **sem** `bnafar`, com o estoque do domínio do local utilizável. Grava o `stock_keeper` do local, de qualquer domínio; o `municipal_admin` só lê.
  - Saldo inicial (Q11): o `municipal_admin` ou o `stock_keeper` do almoxarifado central. Pelo §6, a rota exige `bnafar`; D12 propõe tirar.
  - Credencial: o `municipal_admin`, sem `bnafar`.
- **Domínio ligado na entrada** (contrato §2; Q6: ligado quer dizer **utilizável**):
  - `bnafarFieldsFor(enabledDomains, user, locationKind)` usa os domínios utilizáveis do `GET /pharmacy/bnafar/domains` (`enabled_domains`, proposta D11). Sem esse campo, usa as chaves ligadas da sessão: `bnafar` e a chave do domínio.
  - Chave por tipo de local: `central` e `unit` → `bnafar_medications`; `vaccine_room` e `cold_chain_central` → `bnafar_vaccines`; `vector_control` → `bnafar_vector_supplies`.
  - Campos obrigatórios:
    - medicamento e vacina: `unit_price_cents`, o fabricante, `anvisa_registration`, `health_program` e `ampp_code`;
    - produto de campo: os mesmos, sem `ampp_code`.
  - O AMPP só é exigido quando o item tem VMP (Q15). O fabricante é o CNPJ **ou**, para fabricante internacional, o nome dele com o CNPJ vazio (Q15).
  - Com o domínio desligado, os campos aparecem como opcionais e só vão se preenchidos.
  - O IUM não tem campo na tela: o SNCM não está em vigor, e o api aceita nulo.
- **Formato** (recusar, nunca cortar; o api confere de novo):
  - valor unitário em reais com vírgula, até 2 casas, `> 0` e até `20.000.000,00` (o `UNIT_PRICE_CENTS` do plano do api), em centavos inteiros: `"1.234,56"` vira `123456`, e `"12,345"` é recusado;
  - CNPJ com 14 caracteres alfanuméricos, no formato novo da Receita (Q7): 12 de `[0-9A-Z]` e 2 dígitos;
  - registro ANVISA com 11 ou 13 dígitos;
  - AMPP até 25 caracteres (Q7), sempre escolhido da lista;
  - programa só da lista DM14 (Q7);
  - nome do fabricante internacional com 1–200 caracteres (pesquisa, `nomeFabricanteInternacional`);
  - CNES com 7 dígitos;
  - CATMAT inteiro positivo de até 9 dígitos (proposta);
  - nos documentos, pontos, barras, hífens e espaços saem antes de contar, e as letras do CNPJ viram maiúsculas;
  - quantidade da saída externa inteira, `≥ 1` e até o saldo do lote;
  - motivo da saída com 10–500 *code points* depois do `trim`.
- **Step-up** (`SensitiveAction` com `requiresStepUp`) em oito escritas:
  - gravar a credencial;
  - "Reenviar recusados";
  - "Corrigir";
  - registrar a saída externa;
  - o CNES do central e o da rede de frio (Q13);
  - aplicar e desfazer o saldo inicial (Q11).

  O teste de conexão, o complemento do lote, o envio da planilha para conferência e o descarte não pedem step-up. Toda escrita leva corpo JSON (`{}` quando não há campo).
- **Concorrência:**
  - O complemento manda `expected_version`, o `bnafar_version` do lote; lote antigo manda `0` (Q16).
  - A correção manda a `version` da saída externa.
  - Aplicar, descartar e desfazer o saldo inicial mandam a `version` da importação (proposta, D12).
  - Um `409 *_changed` mostra a frase, faz **uma** releitura e mantém o formulário com o digitado, sem reenvio.
  - Botão de escrita fica desabilitado enquanto a chamada corre: o duplo clique manda um pedido só.
- **Sigilo** (contrato §9):
  - usuário e senha da BNAFAR só vivem no estado do `SensitiveAction`, nunca em URL, cache, log, storage ou `console`;
  - a planilha e o base64 só vivem no estado do formulário do saldo inicial, e o nome do arquivo não vai ao api;
  - nenhum dado de pessoa nestas telas;
  - na URL vão só ids, datas, códigos de assunto, situação e pendência, `vmp_code` e o cursor;
  - o motivo da saída externa vai no corpo.
- **Leituras:** tudo aqui é sem pessoa → `STATIC_QUERY`, relido ao abrir, depois de cada escrita e no "Atualizar". **Nenhum polling.**
- **Interface:**
  - toda tela nova entra em `NAV_GROUPS` **e** em `CATALOG.screens` (o `catalog.test.ts` cobra);
  - `DataTable` com `card: "title"`/`"actions"`, e 375 px sem rolagem horizontal;
  - cores só por papéis do tema (`Tag` com `tone`), com as guardas `src/theme/*.test.ts` verdes;
  - componentes existentes: `PageHeader`, `Panel`, `DataTable`, `Tag`, `EmptyState`, `KeyValue`, `SensitiveAction`, `ConfirmButton`, `SegmentedControl`, `formStyles`, os estilos de `src/modules/exams/styles.ts`, `PharmacyGate`, `LocationPick`, `VaccineLocationPick`, `saveFile` (`src/modules/lab/saveFile.ts`).
- **CI (lição do `443faca`):** todo `vi.mock("…/lib/api", async (orig) => …)` inclui `fetchCurrentSession: vi.fn()` resolvendo o usuário.
- **Datas:** "agora" é `new Date()`, e as telas recebem `today`/`now` por prop. Os testes usam `NOW27B = new Date("2026-11-26T10:00:00-03:00")` e `TODAY27B = "2026-11-26"` (`TZ=America/Sao_Paulo`). A data da cidade vem de `todayInCity` (`src/lib/campaigns.ts:111`).
- **Testes:** só `$R/bin/test dashboard -w dashboard-mod27b [arquivos do vitest]`, nunca `vitest` direto. Tipos antes de cada commit: `cd $W && npx tsc --noEmit`.
- **Git:**
  - sempre `/opt/homebrew/bin/git`, e nunca `git add -A` nem `git add .`;
  - commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`;
  - merge, push e tag só com autorização explícita.
- **Board #1:** o dashboard é superfície dos cards do 27b (F-27.8 em diante, criados pelo plano do api). Card vai a Done só quando o **último** app dele termina, e a Verified depois da prova (Task 12).
- **Relato ao coordenador:** ao fim de cada task, uma linha à sessão "Próximos módulos" (`SendMessage`): `dashboard · dashboard-mod27b/feat/mod-27b-bnafar · Task N · <pronto> · <divergência nova>`.

## Ambiente de execução

- Variáveis: `R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, `W=$R/.worktrees/dashboard-mod27b`, `G=/opt/homebrew/bin/git`, `A=$R/apps/api/.claude/mod27b`.
- Worktree **fora de `apps/dashboard`**, porque o compose monta `./apps/dashboard:/app` e um `node_modules` ali duplica o React no Vite. Branch `feat/mod-27b-bnafar` a partir de `origin/main`, com `npm ci`. Os caminhos das tasks são relativos a `$W`.
- Testes sem jest-dom (`not.toBeNull()`, `toBe()`), com `globals: false` e `cleanup` no `afterEach`.
- Fixtures:
  - `renderWithProviders`/`sessionWith`: `src/test/campaignFixtures.tsx`;
  - do 27a: `src/test/pharmacyFixtures.ts`;
  - do 29: `src/test/vaccinationFixtures.ts`;
  - do 35a: `src/test/vectorControlFixtures.ts`;
  - novas: `src/test/bnafarFixtures.ts` (Task 1).

## Review Focus

1. **"Reenviar recusados" fora do D+1, com duplo clique ou com o step-up vencido.** Espera-se:
   - um pedido só;
   - `409 retry_window_closed` → a frase e uma releitura, e a remessa aparece perdida, com os registros `lost` e o motivo (Q16);
   - `401 mfa_required` volta ao step-up sem reenviar;
   - a tela nunca calcula o prazo, só mostra o `retry_until` do api.

   Teste: Task 8, "reenviar: step-up, um pedido, janela fechada relê e mostra perdidos".
2. **Entrada com o domínio ligado e uma linha sem os campos do envio, ou a recusa do api numa linha do meio.** Espera-se:
   - a tela recusa antes de enviar, na linha e no campo;
   - a recusa do api com `problems` (`items[2].unit_price_cents`, Q17) cai na linha 3, com a frase do campo;
   - item sem VMP entra sem AMPP, e o fabricante internacional entra sem CNPJ (Q15);
   - com o domínio desligado, nada é exigido.

   Teste: Task 4, "domínio ligado exige os campos; recusa do api na linha certa; sem VMP e internacional entram; desligado não exige".
3. **Valor unitário digitado como as pessoas digitam:** `"12,5"`, `"1.234,56"`, `"1234.56"`, `"0,00"`, `"12,345"` e `" 3 "`. Espera-se centavos certos (`1250`, `123456`, `300`) ou a recusa com a frase, nunca corte nem arredondamento. Teste: Task 2, `parseUnitPrice` por tabela.
4. **Saída externa com quantidade fracionária, acima do saldo ou com o documento do destino com pontuação.** Espera-se:
   - `"2,5"` e `"0"` recusados antes de enviar;
   - acima do saldo recusado antes de enviar; na corrida, `422 insufficient_balance` relê os lotes uma vez;
   - `"12.abc.345/01de-35"` vira `"12ABC34501DE35"` no corpo, e CNES com 8 dígitos é recusado.

   Teste: Task 10, "saída: inteira, até o saldo, documento normalizado, corrida de saldo relê uma vez".
5. **Saldo inicial: arquivo errado, Excel com BOM, aplicar duas vezes, desfazer bloqueado.** Espera-se:
   - `.xlsx`, 6 MB e cabeçalho com vírgula recusados antes de enviar, sem chamada;
   - CSV com BOM aceito;
   - aplicar com duplo clique manda um pedido;
   - `revert_blocked` mostra a frase e mantém "aplicada".

   Teste: Task 11, os quatro casos de mesmo nome.

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | board, worktree, base, âncoras, ícones, conferência do api | 0 |
| `src/lib/api.ts`, `src/lib/api.bnafar.test.ts`, `src/lib/features.ts`, `src/lib/features.test.ts`, `src/test/bnafarFixtures.ts` | tipos e cliente do 27b; extensões do 16, 27a, 29 e 35a; rótulos dos interruptores; fixtures | 1 |
| `src/lib/bnafarErrors.ts`, `src/lib/bnafarErrors.test.ts`, `src/lib/bnafar.ts`, `src/lib/bnafar.test.ts`, `src/lib/pharmacy.ts`, `src/lib/pharmacy.test.ts` | frases; portões, chaves, rótulos, regras de formato; rótulos do movimento e caminhos de recusa | 2 |
| `src/shell/modules.ts`, `src/shell/modules.bnafar.test.ts`, `src/shell/catalog.ts`, `src/App.tsx`, `src/modules/bnafar/BnafarGate.tsx`, `src/modules/bnafar/BnafarGate.test.tsx`, cascas das 4 telas | quatro itens no grupo Farmácia e o portão | 3 |
| `src/modules/pharmacy/BnafarLotFields.tsx`, `src/modules/pharmacy/AmppPick.tsx`, `src/modules/pharmacy/ReceiptForm.tsx`, `src/modules/vaccination/VaccineReceiptForm.tsx`, `src/lib/vaccination.ts`, `src/modules/pharmacy/ReceiptForm.bnafar.test.tsx`, `src/modules/vaccination/VaccineReceiptForm.bnafar.test.tsx` | campos do envio na entrada e na abertura | 4 |
| `src/modules/bnafar/LotBnafarForm.tsx`, `src/modules/bnafar/LotBnafarForm.test.tsx` | formulário do complemento do lote | 5 |
| `src/modules/pharmacy/LocationsPanel.tsx`, `src/modules/vaccination/RoomsPanel.tsx`, `src/modules/vectorControl/ProductsPanel.tsx`, `src/modules/PharmacyAdmin.bnafar.test.tsx`, `src/modules/VaccinationAdmin.bnafar.test.tsx`, `src/modules/VectorSettings.bnafar.test.tsx` | CNES do central e da rede de frio; CATMAT do produto de campo | 6 |
| `src/lib/integrations.ts`, `src/lib/integrations.test.ts`, `src/modules/Integrations.tsx`, `src/modules/integrations/BnafarCredentialForm.tsx`, `src/modules/Integrations.bnafar.test.tsx` | credencial `bnafar` do ambiente em uso e teste | 7 |
| `src/modules/BnafarStatus.tsx`, `src/modules/bnafar/AlertBand.tsx`, `src/modules/bnafar/SubmissionDetail.tsx`, `src/modules/bnafar/CorrectionForm.tsx`, `src/modules/BnafarStatus.test.tsx` | "Envio à BNAFAR" | 8 |
| `src/modules/BnafarPending.tsx`, `src/modules/BnafarPending.test.tsx` | "Pendências do envio à BNAFAR", com o complemento | 9 |
| `src/modules/ExternalExits.tsx`, `src/modules/pharmacy/ExternalExitForm.tsx`, `src/modules/pharmacy/ExitLocationPick.tsx`, `src/modules/ExternalExits.test.tsx` | "Saída externa" | 10 |
| `src/lib/openingImport.ts`, `src/lib/openingImport.test.ts`, `src/modules/OpeningImport.tsx`, `src/modules/openingImport/ImportReport.tsx`, `src/modules/OpeningImport.test.tsx`, `src/modules/pharmacy/OpeningsPanel.tsx`, `src/modules/PharmacyEntries.tsx`, `src/modules/PharmacyEntries.openings.test.tsx` | "Saldo inicial" e a lista das aberturas do local | 11 |
| — | suíte, build, varreduras, revisão, prova no navegador (5223 → 3062) | 12 |
| — | board, relato, parada antes do merge | 13 |

**Estratégia de teste:**
- **Cliente** (Task 1): `fetch` falso (`stub`, `last`, `bodyOf`, como em `api.pharmacy.test.ts`), conferindo URL, método e corpo; senha, motivo, documentos e planilha só no corpo.
- **Regras puras** (Tasks 2 e 11): tabela de casos, com as bordas de formato do Review Focus.
- **Menu** (Task 3): por papel e por interruptor.
- **Telas** (Tasks 4–11): pelo ponto de entrada, com `renderWithProviders`, `vi.mock` do api com `fetchCurrentSession` e sessão por papel e por interruptor. Cobrem:
  - o caminho feliz;
  - cada recusa que muda a tela;
  - o duplo envio;
  - o 409 com uma releitura;
  - o que **não** pode aparecer: senha no cache, botão de escrita para quem só lê, campo exigido com o domínio desligado, planilha enviada quando o arquivo é recusado.
- **Fim** (Task 12): `tsc`, build, suíte, varreduras e prova no navegador, com o api real na 3062 e o Vite na 5223.

O dashboard não testa o que é do api:
- a montagem do registro e a divisão em 20 itens por registro e 1.000 registros por remessa;
- DM01/DM12/DM13, a fila e o backoff, o `expires_in` e o cURL;
- a validação de cada linha da planilha e a conversão de unidade;
- o sigilo de evento;
- D+1 e `lost` no relógio do servidor.

---

### Task 0: Board, worktree, base, âncoras, ícones e conferência do api

**Files:** nenhum.

**Interfaces:**
- Consumes: `origin/main` do dashboard (com o 35b); board #1; o api `feat/mod-27b-bnafar`.
- Produces: cards em In Progress; `$W` na branch; base, âncoras, ícones e formas do api anotados.

- [ ] **Step 1: Cards.** Um `gh` simples por vez, sem `$(...)` nem laço:

  ```bash
  gh project item-list 1 --owner rotasaude --limit 900 --format json --jq '.items[] | select(.title | test("^F-27\\.([89]|[1-9][0-9]) ")) | {id, title, status}'
  ```

  Os cards do 27b que o dashboard toca vão a In Progress, com `gh project item-edit --id <PVTI> --project-id PVT_kwDOEbfGRc4BbxiB --field-id PVTSSF_lADOEbfGRc4BbxiBzhWfnC0 --single-select-option-id <In Progress>`. Card ausente: **pare e pergunte** (quem cria é o plano do api).

- [ ] **Step 2: Base e âncoras**

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/dashboard fetch -q origin && $G -C $R/apps/dashboard log --oneline -3 origin/main
$G -C $R/apps/dashboard log --oneline origin/main | grep -ci zoonos
$G -C $R/apps/dashboard show origin/main:src/lib/api.ts | grep -n '^export type CredentialKind\|^export interface IntegrationCredential {\|^export interface StockItem {\|^export interface StockLot {\|^export type StockMovementKind =\|^export type ReceiptItemInput\|^export interface AdminLocation \|^export interface AdminLocationInput\|^export function createColdChainCentral\|^export interface VectorProduct extends\|VaccineReceiptInput\|^function readBase64\|^async function binaryFetch\|^export async function listMyVaccineLocations'
$G -C $R/apps/dashboard show origin/main:src/lib/features.ts | grep -n '| "vector_control"\|zoonoses'
$G -C $R/apps/dashboard show origin/main:src/shell/modules.ts | grep -n '{ id: "pharmacy-admin", label: "Locais e configurações"\|if (item.id === "pharmacy-admin") return'
$G -C $R/apps/dashboard show origin/main:src/lib/pharmacy.ts | grep -n '^const MOVEMENT_LABEL\|^const DOCUMENT_LABEL\|^const PATH_RULES\|^function receiptLineProblem\|^export function receiptItems\|^export function problemItemIndex'
$G -C $R/apps/dashboard show origin/main:src/lib/integrations.ts | grep -n '^const CREDENTIAL_LABEL\|case "credential_unauthorized":\|^const MESSAGES'
$G -C $R/apps/dashboard show origin/main:src/modules/Integrations.tsx | grep -n 'editing?.kind === "rnds"\|e-SUS PEC · CADSUS · RNDS'
```

Expected:
- o 35b presente na `main` (contagem > 0). Se for 0, **pare**: o contrato manda começar depois do merge dele;
- cada linha achada. Linha mudada: a task dona aplica a mesma intenção sobre o texto real e anota no commit.

- [ ] **Step 3: Worktree, base e ícones**

```bash
$G -C $R/apps/dashboard worktree add -q $R/.worktrees/dashboard-mod27b -b feat/mod-27b-bnafar origin/main
cd $R/.worktrees/dashboard-mod27b && npm ci --silent && npx tsc --noEmit && echo tsc-ok
$R/bin/test dashboard -w dashboard-mod27b
ls node_modules/@tabler/icons-react/dist/esm/icons | grep -E '^Icon(CloudUpload|ListDetails|TruckReturn|FileImport)\.mjs$'
```

Expected:
- `tsc-ok`;
- suíte verde (**anote** arquivos e testes);
- os quatro ícones. Faltou algum: use outro que exista, e o mesmo na Task 3.

- [ ] **Step 4: Formas do api.** Leia, na branch `feat/mod-27b-bnafar` do api:
  - as rotas: `$G -C $R/apps/api show feat/mod-27b-bnafar:config/routes.rb | grep -n 'bnafar\|external_exits\|lots/:id\|openings\|opening_imports\|cold_chain_central\|obm_presentations'`;
  - os serializadores de `Bnafar` e de `OpeningImports`, e o `Pharmacy::Json` do lote;
  - os specs de request;
  - o `Platform::Features::CATALOG`: as cinco chaves e os `requires`.

  Confira tudo contra as formas da Task 1. **Onde o api diz outra coisa, vale o api**: a task dona ajusta tipo, fixture e teste e registra no commit. Sem branch ainda: siga pelas formas deste plano e repita este passo antes da Task 12.

  Confira também:
  - os códigos de recusa do plano do api (Valores 8 e 11);
  - o nome do campo do fabricante internacional (Q15);
  - o nome do rótulo do item (Q16);
  - se o `GET /pharmacy/bnafar/domains` traz os domínios utilizáveis (D11).

---

### Task 1: Tipos, cliente HTTP, rótulos dos interruptores e fixtures

**Files:**
- Modify: `src/lib/api.ts` (seção nova no fim; extensões nas seções do 16, 27a, 29 e 35a), `src/lib/features.ts`
- Create: `src/lib/api.bnafar.test.ts`, `src/test/bnafarFixtures.ts`
- Test: `src/lib/features.test.ts`

**Interfaces:**
- Produces: em `src/lib/api.ts`, a seção "Envio à BNAFAR, saída externa e saldo inicial (módulo 27b; contrato 2026-11-24 §1–§7, Q6–Q17)". Caminhos por `phPath`/`phWrite` do 27a e `INTEGRATIONS_BASE` do 16; listas `{ items, next_before }`. O que é proposta está marcado:

```ts
export type BnafarEnvironment = "production" | "homologation";
export type BnafarSubject = "position" | "exit";
export type BnafarSubmissionStatus = "building" | "queued" | "sending" | "processing" | "finished" | "failed" | "lost";
export type BnafarRecordStatus = "pending" | "accepted" | "rejected" | "lost";                  // Q16
export type BnafarPendencyCode =
  | "lot_incomplete" | "location_without_cnes" | "product_without_code" | "fractional_balance" | "exit_type_missing" | "domain_off"
  | "lot_number_too_long";                                                                       // Q7
export type StockDomain = "medication" | "immunobiological" | "vector_control";
export type BnafarField = "unit_price_cents" | "manufacturer_cnpj" | "anvisa_registration" | "health_program" | "ampp_code";
export type Coded = { code: string; label: string }[];
// Na posição e no extrato o lote só traz `{ version, missing }` (plano do api, Task 5); lote antigo: version 0 (Q16).
export interface LotBnafarState { version: number; missing: string[] }
export interface LotBnafar extends LotBnafarState {                                              // resposta do PATCH
  unit_price_cents: number | null; manufacturer_cnpj: string | null; manufacturer_international_name: string | null;
  anvisa_registration: string | null; health_program: string | null; ampp_code: string | null; ium: string[] | null;
}
// Q15: o fabricante internacional entra pelo nome, com o CNPJ vazio. O nome do campo é proposta; a Task 0 confere.
export interface BnafarLotInput {
  unit_price_cents?: number; manufacturer_cnpj?: string; manufacturer_international_name?: string;
  anvisa_registration?: string; health_program?: string; ampp_code?: string;
}
export interface LotBnafarInput extends BnafarLotInput { location_id: string; expected_version: number }
export interface ObmPresentation { ampp_code: string; description: string; anvisa_registration: string | null; manufacturer_cnpj: string | null }
export interface BnafarDomains {
  health_programs: Coded; external_exit_types: Coded;                                            // Q16
  enabled_domains?: StockDomain[];                                                               // proposta (D11)
}
export interface BnafarAlert { late: number; rejected: number; lost: number; last_finished_on: string | null }
export interface BnafarDaySubject { subject: BnafarSubject | (string & {}); status: BnafarSubmissionStatus | (string & {}); accepted: number; rejected: number; pending: number }
export interface BnafarStatus { environment: BnafarEnvironment; alert: BnafarAlert; days: { reference_on: string; subjects: BnafarDaySubject[] }[] }
export interface BnafarBatch { protocol: string | null; status: string; attempts: number }        // Q8
export interface BnafarSubmission {
  id: string; reference_on: string; subject: BnafarSubject | (string & {}); environment: BnafarEnvironment;
  status: BnafarSubmissionStatus | (string & {}); batches: BnafarBatch[];
  counts: { accepted: number; rejected: number; pending: number; lost: number };                 // proposta
}
// Q16: o nome do local e o rótulo do item, sem pessoa. Q8: aceito em parte = `accepted` com `rejected_items`.
export interface BnafarRecord {
  id: string; subject: BnafarSubject | (string & {}); status: BnafarRecordStatus | (string & {});
  stock_location_id: string; location_name: string; item_label: string;
  error_codes: string[]; rejected_items: { lot_number: string; error_codes: string[] }[]; lost_reason: string | null;
  bnafar_code: string | null; retry_until: string | null; correct_until: string | null; external_exit_id: string | null;
}
export interface BnafarSubmissionDetail extends BnafarSubmission { records: BnafarRecord[] }
export interface BnafarPendency {
  code: BnafarPendencyCode | (string & {}); stock_location_id: string; location_name: string; item_label: string;
  location_kind: string; stock_lot_id: string | null; lot_number: string | null; lot_version: number | null;  // os três: proposta
  fields: string[];
}
export interface BnafarPendencies { items: BnafarPendency[]; counts: Record<string, number> }
export type ExternalExitType = "S-DF" | "S-TR" | "S-D" | "S-E";
export interface ExternalExitInput {
  location_id: string; exit_type: ExternalExitType; destination_kind: "cnes" | "cnpj"; destination_code: string;
  reason: string; items: { lot_id: string; quantity: number }[];
}
// A lista não traz o motivo; o detalhe traz (plano do api, Task 9).
export interface StockExternalExit {
  id: string; version: number; location: LocationRef; exit_type: ExternalExitType | (string & {}); destination_kind: "cnes" | "cnpj";
  destination_code: string; reason?: string; recorded_at: string; recorded_by: { name: string | null } | null;
  items: { lot: StockLot; quantity: number }[];
}
// Q8: a correção vale para a saída inteira, ou seja, para todos os registros dela.
export interface BnafarCorrectionInput {
  expected_version: number; exit_type?: ExternalExitType; destination_kind?: "cnes" | "cnpj"; destination_code?: string; reason: string;
}
export interface StockOpening { stock_entry_id: string; created_at: string; recorded_by: { name: string | null } | null; reverted: boolean; items_count: number }
// Q11: o relatório da planilha (os campos do relatório são proposta, D12).
export type OpeningImportStatus = "uploaded" | "validated" | "applied" | "discarded" | "reverted";
export interface OpeningImport {
  id: string; status: OpeningImportStatus | (string & {}); version: number; rows_total: number; rows_valid: number;
  uploaded_at: string; uploaded_by: { name: string | null } | null; applied_at: string | null;
  report: {
    errors_by_code: { code: string; count: number }[]; products_without_catalog: string[]; cnes_without_location: string[];
    totals_by_location: { cnes: string; location_name: string | null; rows: number; quantity: string; value_cents: number }[];
  };
}
export interface OpeningImportRow {
  line: number; status: "valid" | "error" | (string & {}); error_codes: string[];
  cnes: string | null; product_type: string | null; product_code: string | null; lot: string | null; expires_on: string | null; quantity: string | null;
}
export interface OpeningImportDetail extends OpeningImport { rows: OpeningImportRow[]; next_after: number | null }
```

- Funções (todas `Promise`):
  - `getBnafarStatus(): BnafarStatus` → `GET /pharmacy/bnafar/status`.
  - `listBnafarSubmissions(q: { from?; to?; subject?; status?; before? })` → `GET /pharmacy/bnafar/submissions?…`, só com as chaves preenchidas.
  - `getBnafarSubmission(id)`.
  - `retryBnafarSubmission(id)` → `POST …/submissions/:id/retry` com `{}`.
  - `correctBnafarRecord(id, i): BnafarRecord[]` → `POST /pharmacy/bnafar/records/:id/correct`.
  - `listBnafarPendencies(q: { locationId?; code? }): BnafarPendencies` → `GET /pharmacy/bnafar/pendencies?location_id=&code=`.
  - `getBnafarDomains(): BnafarDomains` → `GET /pharmacy/bnafar/domains` (Q16).
  - `listObmPresentations(vmpCode): ObmPresentation[]` → `GET /pharmacy/obm_presentations?vmp_code=` (Q16), com envelope `{ items }`.
  - `updateLotBnafar(lotId, i: LotBnafarInput): StockLot & { bnafar: LotBnafar }` → `PATCH /pharmacy/lots/:id/bnafar`.
  - `recordExternalExit(i)` → `POST /pharmacy/external_exits`.
  - `listExternalExits(q: { locationId?; before? })` → `GET /pharmacy/external_exits?location_id=&before=`.
  - `getExternalExit(id)` → `GET /pharmacy/external_exits/:id`.
  - `listOpenings(locationId, before?)` → `GET /pharmacy/openings?location_id=&before=` (Q13).
  - `uploadOpeningImport(file: Blob): OpeningImportDetail` → `POST /pharmacy/opening_imports` com `{ file_base64 }`. O `readBase64` do módulo é reaproveitado, e o nome do arquivo não vai.
  - `getOpeningImport(id, q: { after?: number | null; status?: "error" })` → `GET /pharmacy/opening_imports/:id?after=&status=`.
  - `applyOpeningImport(id, expectedVersion)`, `revertOpeningImport(id, expectedVersion)` e `discardOpeningImport(id, expectedVersion)` → `POST …/:id/apply|revert|discard` com `{ expected_version }`.
  - `fetchOpeningImportTemplate(): SignatureFile` → `binaryFetch` em `/pharmacy/opening_imports/template.csv` (`text/csv`).
  - `listOpeningImports()` → `GET /pharmacy/opening_imports` (proposta, D12), com envelope `{ items }`.
  - `setBnafarCredential(username, password, environment)` → `PUT /integrations/credentials/bnafar` com `{ username, password, environment }`.
  - `checkBnafarCredential(environment)` → `POST …/bnafar/check` com `{ environment }`.
  - `setColdChainCentralCnes(cnes: string | null): AdminLocation` → `PATCH /vaccination/admin/cold_chain_central` com `{ cnes }` (Q13).
- Extensões:
  - 16: `CredentialKind` + `"bnafar"`; `IntegrationCredential.environment?: BnafarEnvironment | null`. O `GET /integrations` mostra uma linha `bnafar`, a do ambiente em uso (Q9).
  - 27a:
    - `StockItem.vmp_code?: string | null` (Q15, Q16);
    - `StockLot.bnafar?: LotBnafarState | null`;
    - `StockMovementKind` + `"external_exit" | "opening_reversal"`;
    - `ReceiptItemInput` ganha os campos de `BnafarLotInput`;
    - `AdminLocation.cnes?: string | null` e `AdminLocationInput.cnes?: string | null` (Q13).
  - 29: os itens de `VaccineReceiptInput` ganham os campos de `BnafarLotInput`; o imunobiológico do catálogo ganha `obm_vmp?: string | null`; `VaccineRoomsAdmin.cold_chain_central` ganha `cnes`.
  - 35a: `VectorProduct.catmat_code: number | null`; `VectorProductInput.catmat_code?: number | null`.
- `src/lib/features.ts`:
  - `FeatureKey` + `"bnafar" | "bnafar_medications" | "bnafar_vector_supplies" | "bnafar_vaccines" | "bnafar_mock"`;
  - `FEATURE_LABEL` com os rótulos curtos do §1 e `bnafar_mock: "BNAFAR simulado"` (Q6), com o comentário de uma linha do módulo.
- Fixtures (`src/test/bnafarFixtures.ts`):
  - `NOW27B` e `TODAY27B`;
  - `BNAFAR_FEATURES = [ "pharmacy", "bnafar", "bnafar_medications" ]` e `bnafarUser(roles = [ "stock_keeper" ], features = BNAFAR_FEATURES)`;
  - `BNAFAR_DOMAINS`: os programas `AF` "Assistência Farmacêutica básica", `HIPERTEN` "Hipertensão" e `INSULDEP I` "Insulina", os quatro tipos de saída e `enabled_domains: [ "medication" ]`;
  - `PRESENTATIONS`: dois AMPP do VMP `"VMP0001"`;
  - `lotBnafar(o?)`, `incompleteLot(o?)` (`bnafar: { version: 0, missing: [ os cinco ] }`);
  - `status(o?)`, `submission(o?)`, `rejectedSubmission(o?)` (`retry_until` nos registros recusados);
  - `record(o?)`, `exitRecord(o?)` (`correct_until`), `lostRecord(o?)` (`status: "lost"`, `lost_reason: "retry_window_closed"`);
  - `pendencies(codes, o?)`;
  - `externalExit(o?)` (`version: 1`, sem `reason`) e `externalExitDetail(o?)` (com `reason`);
  - `opening(o?)`, `openingImport(o?)` (`validated`, `version: 1`, 10 linhas, 8 válidas, o relatório) e `importRows(n, o?)`;
  - `bnafarCredential(environment, o?)`.

- [ ] **Step 1: Testes que falham** — `src/lib/api.bnafar.test.ts` (molde `src/lib/api.pharmacy.test.ts`):

```ts
it("credencial: senha e ambiente só no corpo; o teste leva o ambiente", async () => {
  stub(bnafarCredential("production")); await setBnafarCredential("usuario.ente", " s3nh@ ", "production");
  expect([ last().url, last().init.method, bodyOf() ]).toEqual([ "/integrations/credentials/bnafar", "PUT", { username: "usuario.ente", password: " s3nh@ ", environment: "production" } ]);
  stub(bnafarCredential("production")); await checkBnafarCredential("production");
  expect([ last().url, bodyOf() ]).toEqual([ "/integrations/credentials/bnafar/check", { environment: "production" } ]);
});
it("remessas, reenvio e correção", async () => {
  stub({}); await listBnafarSubmissions({ from: "2026-11-01", subject: "exit" });
  expect(last().url).toBe("/pharmacy/bnafar/submissions?from=2026-11-01&subject=exit");
  stub(rejectedSubmission()); await retryBnafarSubmission("bs1");
  expect([ last().url, bodyOf() ]).toEqual([ "/pharmacy/bnafar/submissions/bs1/retry", {} ]);
  stub([ exitRecord() ]); await correctBnafarRecord("br1", { expected_version: 1, destination_kind: "cnpj", destination_code: "12ABC34501DE35", reason: "destino informado errado" });
  expect(last().url).toBe("/pharmacy/bnafar/records/br1/correct");
});
it("domínios, apresentações e complemento", async () => {
  stub(BNAFAR_DOMAINS); await getBnafarDomains(); expect(last().url).toBe("/pharmacy/bnafar/domains");
  stub({ items: PRESENTATIONS }); expect(await listObmPresentations("VMP0001")).toEqual(PRESENTATIONS);
  expect(last().url).toBe("/pharmacy/obm_presentations?vmp_code=VMP0001");
  stub({ ...incompleteLot(), bnafar: lotBnafar() }); await updateLotBnafar("lt1", { location_id: "l1", expected_version: 0, unit_price_cents: 1250, manufacturer_international_name: "Pharma GmbH" });
  expect(bodyOf()).toEqual({ location_id: "l1", expected_version: 0, unit_price_cents: 1250, manufacturer_international_name: "Pharma GmbH" });
});
it("saldo inicial: base64 sem nome; atos com expected_version; modelo pelo binaryFetch", async () => {
  stub(openingImport()); await uploadOpeningImport(new File([ "abc" ], "saldo.csv"));
  expect([ last().url, bodyOf() ]).toEqual([ "/pharmacy/opening_imports", { file_base64: "YWJj" } ]);
  stub(openingImport({ status: "applied" })); await applyOpeningImport("oi1", 1);
  expect([ last().url, bodyOf() ]).toEqual([ "/pharmacy/opening_imports/oi1/apply", { expected_version: 1 } ]);
  stub({ ...openingImport(), rows: [], next_after: null }); await getOpeningImport("oi1", { after: 100, status: "error" });
  expect(last().url).toBe("/pharmacy/opening_imports/oi1?after=100&status=error");
  stubBlob(200, "cnes;…\r\n", 'attachment; filename="modelo-saldo-inicial.csv"');
  expect((await fetchOpeningImportTemplate()).filename).toBe("modelo-saldo-inicial.csv");
});
```

Em `src/lib/features.test.ts`: os quatro rótulos do §1 e `featureLabel("bnafar_mock")` = `"BNAFAR simulado"`.

- [ ] **Step 2: Rode e veja falhar** — `$R/bin/test dashboard -w dashboard-mod27b src/lib/api.bnafar.test.ts src/lib/features.test.ts`. Expected: FAIL.
- [ ] **Step 3: Implemente** pelas Interfaces.
- [ ] **Step 4: Rode e veja passar; `tsc`.** Expected: PASS. Se o `tsc` pedir o campo nas fixtures do 35a, acrescente `catmat_code: null` nelas.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/api.ts src/lib/api.bnafar.test.ts src/lib/features.ts src/lib/features.test.ts src/test/bnafarFixtures.ts src/test/vectorControlFixtures.ts
$G commit -m "feat: add the BNAFAR, external exit and opening import API client with the lot, credential and product type extensions" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Frases, portões, rótulos e regras de formato

**Files:**
- Create: `src/lib/bnafarErrors.ts`, `src/lib/bnafarErrors.test.ts`, `src/lib/bnafar.ts`, `src/lib/bnafar.test.ts`
- Modify: `src/lib/pharmacy.ts` (`MOVEMENT_LABEL`, `DOCUMENT_LABEL`, `PATH_RULES`, `ERRORS`), `src/lib/pharmacy.test.ts`

**Interfaces:**
- Consumes: `hasFeature`, `featureLabel` (Task 1); `pharmacyError`, `Problem` (`src/lib/pharmacy.ts`); `STATIC_QUERY`.
- Produces (`src/lib/bnafarErrors.ts`):
  - `BNAFAR_DISABLED = "o envio à BNAFAR está desligado nesta cidade"`;
  - `BNAFAR_ERRORS: Record<string, string>` (contrato §7): uma frase por código, todas distintas salvo os grupos declarados, sem `_`, sem dado de pessoa;
  - `bnafarErrorText(code)`: código ausente → `"recusa da BNAFAR: <código>"`.

  Os códigos seguem o contrato (Q7, Q16) e o plano do api (Valor 8). **As frases são todas proposta:**

| Código | Frase |
|---|---|
| `lot_incomplete` | "lote sem os dados do envio (valor, fabricante, registro ANVISA, programa ou AMPP)" |
| `location_without_cnes` | "local sem CNES — o envio deste local espera o CNES" |
| `product_without_code` | "produto sem código da BNAFAR (VMP do medicamento, código OBM da vacina ou CATMAT do produto de campo)" |
| `fractional_balance` | "saldo com fração — a BNAFAR só aceita quantidade inteira; ajuste o saldo" |
| `exit_type_missing` | "saída sem tipo BNAFAR correspondente — fica fora do envio" |
| `domain_off` | "tipo de produto ainda não ligado ao envio" |
| `lot_number_too_long` | "número do lote com mais de 30 caracteres — a BNAFAR não aceita este lote" |
| `feature_disabled` | `BNAFAR_DISABLED` |
| `lot_bnafar_changed` | "os dados do lote foram salvos por outra pessoa — o lote foi relido; confira antes de salvar" |
| `external_exit_changed` | "a saída foi corrigida por outra pessoa — os dados foram relidos; confira antes de corrigir" |
| `retry_window_closed` | "passou o dia seguinte ao da remessa — os registros recusados ficaram perdidos" |
| `correction_window_closed` | "passaram os 30 dias depois do aceite — esta saída não pode mais ser corrigida" |
| `not_correctable` | "só a saída externa aceita pode ser corrigida; a posição de estoque sai corrigida no dia seguinte" |
| `nothing_to_retry` | "esta remessa não tem registros recusados para reenviar" |
| `ampp_unknown` | "esta apresentação (AMPP) não está no catálogo OBM" |
| `ampp_not_in_vmp` | "esta apresentação (AMPP) é de outro medicamento" |
| `anvisa_mismatch` | "o registro ANVISA não é o da apresentação (AMPP) escolhida" |
| `program_unknown` | "programa de saúde fora da tabela da BNAFAR" |
| `invalid_cnpj` | "CNPJ inválido — confira os 14 caracteres" |
| `cnes_in_use` | "este CNES já é de uma unidade de saúde — use o CNES próprio do almoxarifado" |
| `invalid_environment` | "ambiente desconhecido" |
| `MSG06` | "a BNAFAR não encontrou este CNES" |
| `MSG08` | "um valor está fora da tabela da BNAFAR" |
| `MSG09` | "a BNAFAR não reconheceu o produto (código)" |
| `MSG10` | "a BNAFAR não reconheceu o programa de saúde" |
| `MSG11` | "data no futuro" |
| `MSG12`, `MSG59` | "CNPJ do fabricante não encontrado na Receita Federal" |
| `MSG13` | "fabricante: informe só o CNPJ ou só o nome do fabricante internacional" |
| `MSG15` | "registro igual a um já enviado — contado como aceito" |
| `MSG16`, `MSG17`, `MSG18` | "passou o prazo de 30 dias para corrigir na BNAFAR" |
| `MSG21` | "tipo de saída não aceito pela BNAFAR" |
| `MSG22` | "a remessa anterior ainda está em processamento na BNAFAR" |
| `MSG44` | "serviço do Ministério indisponível — a BNAFAR reprocessa sozinha" |
| `MSG46` | "registro com itens demais" |
| `MSG47` | "nenhum item do registro foi aceito" |
| `MSG51` | "o CNES não pertence a esta cidade" |
| `MSG62` | "remessa com mais de 1.000 registros" |
| `MSG63` | "remessa sem itens válidos" |
| `MSG71` | "a BNAFAR não reconheceu o AMPP" |
| `MSG72` | "o registro ANVISA não confere com o AMPP" |
| `MSG73` | "a BNAFAR só aceita a data de hoje ou de ontem" |
| `MSG01`, `MSG02`, `MSG03`, `MSG04`, `MSG64` | "a BNAFAR recusou a credencial da cidade — o administrador troca e testa em Integrações" |

- Produces (`src/lib/bnafar.ts`):
  - **Portões:**
    - `canUseBnafar(user)`;
    - `bnafarRoles(user): { admin; keeper; pharmacist; reader }`, com `reader` = `admin || keeper`;
    - `BNAFAR_NO_ROLE = "seu papel não permite ver o envio à BNAFAR"` (proposta);
    - `DOMAIN_FEATURE: Record<string, FeatureKey>` e `KIND_DOMAIN: Record<string, StockDomain>` (por `location.kind`);
    - `bnafarFieldsFor(enabled: StockDomain[] | null | undefined, user, locationKind: string): BnafarField[]` (D11): com `enabled`, usa o domínio do local; sem `enabled`, as chaves ligadas da sessão. Devolve `[]`, os cinco ou os quatro sem `ampp_code` no `vector_control`.
  - **Chaves:** `BNAFAR_STATUS_KEY`, `BNAFAR_SUBMISSIONS_KEY`, `BNAFAR_SUBMISSION_KEY`, `BNAFAR_PENDENCIES_KEY`, `BNAFAR_DOMAINS_KEY`, `BNAFAR_PRESENTATIONS_KEY`, `EXTERNAL_EXITS_KEY`, `PH_OPENINGS_KEY`, `OPENING_IMPORTS_KEY`; reexporta `STATIC_QUERY`.
  - **Formato:**
    - `parseUnitPrice(raw): number | null`: centavos; aceita `^\d+(,\d{1,2})?$` e `^\d{1,3}(\.\d{3})+(,\d{1,2})?$` depois do `trim`; `> 0` e `≤ 2_000_000_000`;
    - `fmtCents(cents): string` (`"1.234,56"`);
    - `docOf(raw): string | null`: tira espaço, `.`, `/` e `-`, e passa a maiúsculas; com outro caractere fora de `[0-9A-Z]`, `null`;
    - `cnpjProblem` (Q7: `^[0-9A-Z]{12}\d{2}$`), `anvisaProblem`, `cnesProblem`, `catmatProblem` e `internationalNameProblem` (1–200): texto ou `null`.
  - **Campos do lote:**
    - `type LotBnafarDraft = { unitPrice: string; international: boolean; cnpj: string; internationalName: string; anvisa: string; program: string; ampp: string }` e `EMPTY_LOT_BNAFAR`;
    - `missingFields(state): BnafarField[]`: os cinco que o `missing` do api cita;
    - `lotBnafarProblems(d, required: BnafarField[], opts: { hasVmp: boolean }): { field: string; text: string }[]`:
      - campo exigido vazio → `"informe <rótulo>"`;
      - o fabricante vale pelo CNPJ ou, com `international`, pelo nome (Q15);
      - `ampp_code` só é exigido com `hasVmp` (Q15);
      - campo preenchido e inválido → a frase do formato, exigido ou não;
    - `lotBnafarBody(d): BnafarLotInput`: só os preenchidos. Com `international`, vai `manufacturer_international_name` e não vai `manufacturer_cnpj`. Documentos vão por `docOf`, e o valor em centavos;
    - `BNAFAR_FIELD_LABEL` (proposta): `unit_price_cents` "Valor unitário (R$)", `manufacturer_cnpj` "CNPJ do fabricante", `manufacturer_international_name` "Nome do fabricante internacional", `anvisa_registration` "Registro ANVISA", `health_program` "Programa de saúde", `ampp_code` "Apresentação (AMPP)".
  - **Saída externa:**
    - `type ExitLine = { key; lot: LotBalance | null; quantity: string }`;
    - `externalExitProblems(d: { exitType; destinationKind; destination; reason; lines }): Problem[]`: tipo escolhido; CNES com 7 dígitos ou CNPJ pelo Q7; motivo com 10–500; ao menos uma linha; quantidade `^\d+$`, `≥ 1` e `≤` a parte inteira do saldo;
    - `externalExitBody(...)`.
  - **Rótulos** (todos proposta; código desconhecido aparece cru):
    - `subjectLabel`: `position` "Posição de estoque", `exit` "Saídas";
    - `environmentLabel`: `production` "Produção", `homologation` "Homologação";
    - `submissionStatusLabel`/`Tone`: `building` "montando", `queued` "na fila", `sending` "enviando", `processing` "em processamento na BNAFAR", `finished` "concluída", `failed` "falhou", `lost` "perdida";
    - `recordStatusLabel(r)`/`Tone`: `pending` "aguardando", `accepted` "aceito", `accepted` com `rejected_items` "aceito em parte" (Q8), `rejected` "recusado", `lost` "perdido" (Q16);
    - `exitTypeLabel(domains, code)`: do api; sem ele, `S-DF` "Devolução ao fornecedor", `S-TR` "Remanejamento a outro ente", `S-D` "Doação", `S-E` "Empréstimo";
    - `pendencyLabel` = `bnafarErrorText`.
  - **Alerta** (Q16: 15:00 de D+1 no fuso da cidade; frases proposta): `alertLines(a)` devolve uma linha por condição:
    - `late > 0` → "A remessa de ontem ainda não foi aceita pela BNAFAR. Depois da meia-noite de hoje, o dia só volta por exceção.";
    - `rejected > 0` → "N registros recusados ainda podem ser corrigidos e reenviados hoje.", com o singular para 1;
    - `lost > 0` → "N registros perdidos nos últimos 30 dias — veja o motivo na lista.".
  - **Recusas:** `bnafarError(err)`; `isStale(err, code)`.
- `src/lib/pharmacy.ts`:
  - `MOVEMENT_LABEL.external_exit = "saída para fora da rede"` e `MOVEMENT_LABEL.opening_reversal = "abertura desfeita"` (proposta);
  - `DOCUMENT_LABEL.stock_external_exit = "saída externa"`;
  - `PATH_RULES` ganha, antes da regra genérica de `items`, uma frase por campo:
    - `unit_price_cents` → "valor unitário inválido — use reais com vírgula, maior que zero";
    - `manufacturer_cnpj` → "o CNPJ do fabricante tem 14 caracteres";
    - `manufacturer_international_name` → "o nome do fabricante internacional tem até 200 caracteres";
    - `anvisa_registration` → "o registro ANVISA tem 11 ou 13 dígitos";
    - `health_program` → "escolha o programa de saúde";
    - `ampp_code` → "escolha a apresentação (AMPP) do produto";
    - `destination_code` → "o destino é um CNES de 7 dígitos ou um CNPJ de 14 caracteres";
    - `cnes` → "o CNES tem 7 dígitos";
  - o código `required` ganha `requiredText(path)` → `"informe <o campo> — obrigatório para o envio à BNAFAR"`;
  - os códigos do lote vêm de `BNAFAR_ERRORS`;
  - o `path` é o do api real, `items[i].campo` (Q17), e `problemItemIndex` não muda.

- [ ] **Step 1: Testes que falham.**
  - `src/lib/bnafarErrors.test.ts`: cada código da tabela tem frase; as frases são distintas, salvo os grupos declarados; nenhuma tem `_`; código desconhecido volta cru.
  - `src/lib/bnafar.test.ts`:

```ts
it.each([
  [ "12,5", 1250 ], [ "1.234,56", 123456 ], [ " 3 ", 300 ], [ "0,01", 1 ], [ "20.000.000,00", 2000000000 ],
  [ "0,00", null ], [ "12,345", null ], [ "1234.56", null ], [ "1.23,4", null ], [ "-1", null ], [ "", null ], [ "20.000.000,01", null ]
])("parseUnitPrice(%j) = %j", (raw, cents) => expect(parseUnitPrice(raw)).toBe(cents));
it("CNPJ alfanumérico (Q7): pontuação sai, minúscula sobe, DV com letra recusa", () => {
  expect(docOf("12.abc.345/01de-35")).toBe("12ABC34501DE35"); expect(cnpjProblem("12ABC34501DE35")).toBeNull();
  expect(cnpjProblem("12ABC34501DE3X")).not.toBeNull(); expect(docOf("12#345")).toBeNull();
});
it("campos exigidos seguem o domínio utilizável do api; sem ele, a sessão", () => {
  expect(bnafarFieldsFor([ "medication" ], bnafarUser(), "central")).toHaveLength(5);
  expect(bnafarFieldsFor([ "vector_control" ], bnafarUser(), "vector_control")).not.toContain("ampp_code");
  expect(bnafarFieldsFor([], bnafarUser(), "unit")).toEqual([]);
  expect(bnafarFieldsFor(undefined, bnafarUser([ "stock_keeper" ], [ "pharmacy", "bnafar" ]), "unit")).toEqual([]);
});
it("lote: AMPP só com VMP; fabricante internacional sem CNPJ (Q15); corpo só com o preenchido", () => {
  const all: BnafarField[] = [ "unit_price_cents", "manufacturer_cnpj", "anvisa_registration", "health_program", "ampp_code" ];
  const d = { ...EMPTY_LOT_BNAFAR, unitPrice: "1,00", international: true, internationalName: "Pharma GmbH", anvisa: "12345678901", program: "AF" };
  expect(lotBnafarProblems(d, all, { hasVmp: false })).toEqual([]);
  expect(lotBnafarProblems(d, all, { hasVmp: true }).map((p) => p.field)).toEqual([ "ampp_code" ]);
  expect(lotBnafarBody(d)).toEqual({ unit_price_cents: 100, manufacturer_international_name: "Pharma GmbH", anvisa_registration: "12345678901", health_program: "AF" });
});
it("saída externa: inteira, até a parte inteira do saldo, motivo 10–500, destino pelo tipo", () => { /* "2,5", "0", saldo "3.50" aceita 3 e recusa 4; CNES 8 dígitos; motivo de 9 */ });
it("alerta e situação do registro", () => {
  expect(alertLines({ late: 1, rejected: 1, lost: 0, last_finished_on: "2026-11-24" })).toEqual([
    "A remessa de ontem ainda não foi aceita pela BNAFAR. Depois da meia-noite de hoje, o dia só volta por exceção.",
    "1 registro recusado ainda pode ser corrigido e reenviado hoje."
  ]);
  expect(recordStatusLabel(record({ status: "accepted", rejected_items: [ { lot_number: "L1", error_codes: [ "MSG72" ] } ] }))).toBe("aceito em parte");
});
```

  - `src/lib/pharmacy.test.ts`:
    - `movementLabel("external_exit")` e `movementLabel("opening_reversal")`;
    - `required` com `path` `items[0].unit_price_cents` → "informe o valor unitário — obrigatório para o envio à BNAFAR";
    - `problemItemIndex("items[3].ampp_code")` = `3`.

- [ ] **Step 2: Rode e veja falhar** — `$R/bin/test dashboard -w dashboard-mod27b src/lib/bnafarErrors.test.ts src/lib/bnafar.test.ts src/lib/pharmacy.test.ts`. Expected: FAIL.
- [ ] **Step 3: Implemente** pelas Interfaces.
- [ ] **Step 4: Rode e veja passar; `tsc`.** Expected: PASS.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/bnafarErrors.ts src/lib/bnafarErrors.test.ts src/lib/bnafar.ts src/lib/bnafar.test.ts src/lib/pharmacy.ts src/lib/pharmacy.test.ts
$G commit -m "feat: add the BNAFAR phrases, gates and format rules" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Quatro itens no grupo Farmácia e o portão

**Files:**
- Create: `src/modules/bnafar/BnafarGate.tsx`, `src/modules/bnafar/BnafarGate.test.tsx`, `src/shell/modules.bnafar.test.ts`, e as cascas `src/modules/BnafarStatus.tsx`, `src/modules/BnafarPending.tsx`, `src/modules/ExternalExits.tsx` e `src/modules/OpeningImport.tsx` (portão e título; o corpo vem nas Tasks 8–11)
- Modify: `src/shell/modules.ts`, `src/shell/catalog.ts`, `src/App.tsx`

**Interfaces:**
- Consumes: `canUseBnafar`, `bnafarRoles`, `BNAFAR_DISABLED`, `BNAFAR_NO_ROLE` (Task 2); `canUseStockScreens` (27a/35a); `canUseVaccination` (29).
- Produces:
  - `ModuleId` + `"bnafar-status" | "bnafar-pending" | "external-exits" | "opening-import"`.
  - No grupo **Farmácia**:
    - depois de `pharmacy-adjustments`: `{ id: "external-exits", label: "Saída externa", icon: "⇱" }` e `{ id: "opening-import", label: "Saldo inicial", icon: "⇲" }`;
    - antes de `pharmacy-admin`: `{ id: "bnafar-status", label: "Envio à BNAFAR", icon: "⇪" }` e `{ id: "bnafar-pending", label: "Pendências do envio à BNAFAR", icon: "⚑" }` (Q16).
  - Regras do menu:
    - `bnafar-status` e `bnafar-pending` → `canUseBnafar(user) && (isAdmin || isStockKeeper)`;
    - `external-exits` → `(canUseStockScreens(user) || canUseVaccination(user)) && (isStockKeeper || isAdmin)`, sem `bnafar` (Q16);
    - `opening-import` → `canUseBnafar(user) && (canUseStockScreens(user) || canUseVaccination(user)) && (isAdmin || isStockKeeper)`. Pelo §6, a rota exige o `bnafar`; D12 propõe tirar. O almoxarife do central é conferido na tela (Task 11) e no api.
  - `CATALOG.screens` (descrições proposta):
    - `bnafar-status` (`IconCloudUpload`): "Situação de cada dia, recusas e prazo de correção";
    - `bnafar-pending` (`IconListDetails`): "Lotes incompletos, locais sem CNES, produtos sem código e saldos com fração";
    - `external-exits` (`IconTruckReturn`): "Devolução, remanejamento, doação e empréstimo para fora da rede";
    - `opening-import` (`IconFileImport`): "Planilha do saldo inicial, conferência e aplicação".
  - `BnafarGate({ title, sub, allow, children })`, no molde do `PharmacyGate`: sem `bnafar` → `EmptyState` com `BNAFAR_DISABLED`; sem papel → `BNAFAR_NO_ROLE`.
  - `App.tsx`:
    - `case "bnafar-status": return <BnafarStatus onGoToSecurity={() => setActive("security")} />;`
    - `case "bnafar-pending": return <BnafarPending onGoToLocations={() => setActive("pharmacy-admin")} onGoToColdChain={() => setActive("vaccination-admin")} onGoToVectorProducts={() => setActive("vector-settings")} onGoToAdjustments={() => setActive("pharmacy-adjustments")} />;`
    - `case "external-exits": return <ExternalExits onGoToSecurity={() => setActive("security")} />;`
    - `case "opening-import": return <OpeningImport onGoToSecurity={() => setActive("security")} />;`

- [ ] **Step 1: Testes que falham.**
  - `src/shell/modules.bnafar.test.ts`:
    - com `bnafar` e `stock_keeper`, os quatro itens aparecem na ordem relativa acima;
    - com `municipal_admin`, os quatro;
    - com `pharmacist`, nenhum;
    - sem `bnafar`, só "Saída externa" (com `pharmacy`);
    - só com `vaccination` e `stock_keeper`, "Saída externa" aparece;
    - o operador não vê nenhum;
    - o rótulo "Pendências do envio à BNAFAR" é diferente do "Pendências do envio" da Vacinação;
    - o `CATALOG` tem as quatro descrições.
  - `BnafarGate.test.tsx`: as duas frases e o `children` com os papéis.
- [ ] **Step 2–4:** rode e veja falhar (`src/shell/modules.bnafar.test.ts src/modules/bnafar/BnafarGate.test.tsx src/shell/catalog.test.ts src/shell/modules.pharmacy.test.ts`); implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/shell/modules.ts src/shell/modules.bnafar.test.ts src/shell/catalog.ts src/App.tsx src/modules/bnafar/BnafarGate.tsx src/modules/bnafar/BnafarGate.test.tsx src/modules/BnafarStatus.tsx src/modules/BnafarPending.tsx src/modules/ExternalExits.tsx src/modules/OpeningImport.tsx
$G commit -m "feat: add the BNAFAR, external exit and opening stock screens to the pharmacy group" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: Campos do envio na entrada e na abertura

**Files:**
- Create: `src/modules/pharmacy/BnafarLotFields.tsx`, `src/modules/pharmacy/AmppPick.tsx`, `src/modules/pharmacy/ReceiptForm.bnafar.test.tsx`, `src/modules/vaccination/VaccineReceiptForm.bnafar.test.tsx`
- Modify: `src/lib/pharmacy.ts` (`ReceiptLine`, `newReceiptLine`, `receiptProblems`, `receiptItems`), `src/lib/vaccination.ts` (a linha e os itens da entrada de vacina), `src/modules/pharmacy/ReceiptForm.tsx`, `src/modules/vaccination/VaccineReceiptForm.tsx`

**Interfaces:**
- Consumes: `bnafarFieldsFor`, `LotBnafarDraft`, `EMPTY_LOT_BNAFAR`, `lotBnafarProblems`, `lotBnafarBody`, `BNAFAR_FIELD_LABEL`, `BNAFAR_DOMAINS_KEY`, `BNAFAR_PRESENTATIONS_KEY` (Task 2); `getBnafarDomains`, `listObmPresentations`, `StockItem.vmp_code`, `obm_vmp` (Task 1).
- Produces:
  - `ReceiptLine` e a linha da vacina ganham `bnafar: LotBnafarDraft`.
  - `receiptProblems(lines, today, doc, vector, required: BnafarField[] = [])` acrescenta, por linha, a primeira recusa de `lotBnafarProblems(l.bnafar, required, { hasVmp: !!l.item?.vmp_code })`, com o nome do item na frente.
  - `receiptItems` espalha `lotBnafarBody(l.bnafar)` em cada item.
  - `BnafarLotFields({ value, onChange, required, vmpCode, label, disabled })`:
    - campos: valor unitário; "Fabricante internacional (sem CNPJ)" (proposta), que troca o CNPJ pelo nome; registro ANVISA; programa de saúde (`select` com os `health_programs` dos domínios; código fora da lista aparece cru); e, fora do `vector_control`, o `AmppPick`;
    - "(opcional)" no rótulo do campo não exigido;
    - `aria-label` pelo `label()`.
  - `AmppPick({ vmpCode, value, onPick, disabled })`:
    - com `vmpCode`, lê `listObmPresentations` (`STATIC_QUERY`) e oferece `"<descrição> · AMPP <código>"`;
    - escolher preenche o ANVISA e o CNPJ **vazios** com os da apresentação;
    - sem `vmpCode`, mostra "este produto ainda não tem VMP no catálogo — a entrada segue, e o lote fica nas pendências do envio" (proposta), e o AMPP não é exigido (Q15).
  - `ReceiptForm` lê os domínios (`getBnafarDomains`, só com `bnafar` na sessão) e calcula `required = bnafarFieldsFor(domains?.enabled_domains, user, location.kind)`. O `VaccineReceiptForm` faz o mesmo, com o `obm_vmp` do imunobiológico.

- [ ] **Step 1: Testes que falham** — `ReceiptForm.bnafar.test.tsx`, com sessão `bnafarUser([ "stock_keeper" ])` e local `central`:

```ts
it("domínio ligado exige os campos; recusa do api na linha certa; sem VMP e internacional entram; desligado não exige", async () => { /*
  1) três linhas, a 2ª sem valor: nada enviado; alerta "Losartana 50 mg: informe Valor unitário (R$)";
  2) preenche; o AMPP escolhido preenche ANVISA e CNPJ vazios; corpo com unit_price_cents 123456, documentos por docOf, ampp_code;
  3) recordReceipt rejeita { error: "required", path: "items[2].unit_price_cents", problems: [ { path: "items[2].unit_price_cents", error: "required" }, { path: "items[0].lot_number", error: "lot_mismatch" } ] } → linha 3 "informe o valor unitário…", linha 1 a frase do lote;
  4) item com vmp_code null: aviso na linha, sem AMPP, envia; fabricante internacional: corpo com manufacturer_international_name e sem manufacturer_cnpj;
  5) enabled_domains: [] → nada exigido, e o corpo não tem nenhuma das chaves vazias. */ });
it("local vector_control com o domínio utilizável: sem AMPP, quatro exigidos", async () => { /* */ });
it("abertura exige os mesmos campos com o domínio utilizável", async () => { /* mode opening */ });
```

  `VaccineReceiptForm.bnafar.test.tsx`:
  - com `enabled_domains: [ "immunobiological" ]`, a rede de frio exige os campos, com o AMPP pelo `obm_vmp`;
  - sem o domínio, nada é exigido;
  - o corpo de `recordVaccineOpening` leva os campos preenchidos.

- [ ] **Step 2–4:** rode e veja falhar (os dois arquivos e `src/modules/PharmacyEntries.test.tsx src/modules/VaccinationStock.test.tsx`); implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/pharmacy.ts src/lib/vaccination.ts src/modules/pharmacy/BnafarLotFields.tsx src/modules/pharmacy/AmppPick.tsx src/modules/pharmacy/ReceiptForm.tsx src/modules/pharmacy/ReceiptForm.bnafar.test.tsx src/modules/vaccination/VaccineReceiptForm.tsx src/modules/vaccination/VaccineReceiptForm.bnafar.test.tsx
$G commit -m "feat: ask for the BNAFAR lot fields on receipts and openings when the domain is usable" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Formulário do complemento do lote

**Files:**
- Create: `src/modules/bnafar/LotBnafarForm.tsx`, `src/modules/bnafar/LotBnafarForm.test.tsx`

**Interfaces:**
- Consumes: `BnafarLotFields` (Task 4); `missingFields`, `EMPTY_LOT_BNAFAR`, `lotBnafarProblems`, `lotBnafarBody`, `isStale`, `bnafarError` (Task 2); `updateLotBnafar` (Task 1).
- Produces: `LotBnafarForm({ lotId, lotNumber, locationId, state, hasVmp, onSaved, onCancel, reload }: { lotId: string; lotNumber: string; locationId: string; state: LotBnafarState; hasVmp: boolean; onSaved(l: LotBnafar): void; onCancel(): void; reload(): Promise<LotBnafarState | null> })`. Ele só é usado em "Pendências do envio à BNAFAR" (Q16; Task 9).
  - Abre vazio, porque a pendência não traz os valores. Os campos de `missingFields(state)` aparecem como exigidos, e os outros como opcionais.
  - "Salvar dados do lote" (proposta) → `updateLotBnafar(lotId, { location_id: locationId, expected_version: state.version, ...body })`, e mostra os valores que a resposta traz.
  - `409 lot_bnafar_changed` → a frase e `reload()` **uma** vez; o digitado fica, e o `expected_version` passa a ser o relido.
  - Recusa com `path` vai ao campo.
  - Sucesso: "Dados do lote salvos. O lote entra na posição de estoque do próximo envio." (proposta).

- [ ] **Step 1: Testes que falham** (`LotBnafarForm.test.tsx`):
  - lote antigo manda `expected_version: 0` (Q16), `location_id` e só os preenchidos;
  - `409 lot_bnafar_changed` mostra a frase, chama `reload` uma vez, mantém o digitado, e o segundo envio leva a versão relida;
  - `422` com `path` `manufacturer_cnpj` aparece no campo;
  - o fabricante internacional manda o nome;
  - o duplo clique envia um pedido.
- [ ] **Step 2–4:** rode e veja falhar; implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/bnafar/LotBnafarForm.tsx src/modules/bnafar/LotBnafarForm.test.tsx
$G commit -m "feat: add the lot complement form for the BNAFAR data" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: CNES do central e da rede de frio; CATMAT do produto de campo

**Files:**
- Create: `src/modules/PharmacyAdmin.bnafar.test.tsx`, `src/modules/VaccinationAdmin.bnafar.test.tsx`, `src/modules/VectorSettings.bnafar.test.tsx`
- Modify: `src/modules/pharmacy/LocationsPanel.tsx`, `src/modules/vaccination/RoomsPanel.tsx`, `src/modules/vectorControl/ProductsPanel.tsx`

**Interfaces:**
- Consumes: `cnesProblem`, `catmatProblem`, `docOf`, `bnafarError` (Task 2); `AdminLocationInput.cnes`, `setColdChainCentralCnes`, `VectorProductInput.catmat_code` (Task 1).
- Produces (Q13):
  - `LocationsPanel`, no formulário do `central` (novo e editar), pelo `PATCH /pharmacy/admin/locations/:id`:
    - campo "CNES do almoxarifado (opcional, 7 dígitos)", com a ajuda "obrigatório para o envio à BNAFAR; precisa ser um CNES desta cidade" (proposta);
    - vazio → `cnes: null`;
    - a tabela mostra o CNES do central, ou "sem CNES";
    - os outros tipos não mostram o campo.
  - `RoomsPanel`, no quadro da rede de frio central: "CNES da rede de frio" e "Salvar CNES" pelo `SensitiveAction` (`requiresStepUp`) → `setColdChainCentralCnes`.
  - `ProductsPanel`:
    - `ProductDraft` + `catmat: string`, e `ProductProblem.path` + `"catmat_code"`;
    - campo "CATMAT (opcional)", com a ajuda "código do produto no CATMAT; sem ele o produto fica fora do envio à BNAFAR" (proposta);
    - a edição manda `catmat_code` só quando mudou (vazio → `null`);
    - a tabela ganha a coluna "CATMAT".

- [ ] **Step 1: Testes que falham.**
  - Central:
    - `"1234-567"` manda `cnes: "1234567"`;
    - `"123456"` recusado antes de enviar;
    - `422 invalid_request` com `path` `cnes` no campo;
    - `409 cnes_in_use` com a frase;
    - a farmácia da unidade não mostra o campo.
  - Rede de frio: step-up, corpo `{ cnes }`, recusa no campo.
  - Produto:
    - CATMAT `"267123"` manda `catmat_code: 267123`;
    - `"26A"` recusado;
    - apagar manda `null`;
    - editar só o nome não manda `catmat_code`.
- [ ] **Step 2–4:** rode e veja falhar (os três arquivos e `src/modules/PharmacyAdmin.test.tsx src/modules/VaccinationAdmin.test.tsx src/modules/VectorSettings.test.tsx`); implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/pharmacy/LocationsPanel.tsx src/modules/vaccination/RoomsPanel.tsx src/modules/vectorControl/ProductsPanel.tsx src/modules/PharmacyAdmin.bnafar.test.tsx src/modules/VaccinationAdmin.bnafar.test.tsx src/modules/VectorSettings.bnafar.test.tsx
$G commit -m "feat: edit the CNES of the central stores and the cold chain and the CATMAT of field products" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: Credencial `bnafar` em Integrações

**Files:**
- Create: `src/modules/integrations/BnafarCredentialForm.tsx`, `src/modules/Integrations.bnafar.test.tsx`
- Modify: `src/lib/integrations.ts`, `src/lib/integrations.test.ts`, `src/modules/Integrations.tsx`

**Interfaces:**
- Consumes: `setBnafarCredential`, `checkBnafarCredential`, `IntegrationCredential.environment` (Task 1); `environmentLabel` (Task 2); `SensitiveAction`.
- Produces (Q9: o `GET /integrations` mostra a credencial `bnafar` do ambiente em uso; em produção, ele é sempre `production`):
  - `src/lib/integrations.ts`:
    - `CREDENTIAL_LABEL.bnafar = "BNAFAR (SI-BNAFAR)"`;
    - `credentialLabel(kind, environment?)`: com `homologation`, acrescenta " — homologação";
    - `missingPhrase("vector_supplies_disabled")` → "ligue antes: Vigilância ambiental (endemias) ou Centro de zoonoses" (proposta);
    - `bnafar_disabled`, `credential_missing:bnafar` e `credential_unauthorized:bnafar` saem pelas regras genéricas, com os rótulos novos;
    - `MESSAGES.invalid_environment = "ambiente desconhecido — recarregue a página"` (proposta).
  - `BnafarCredentialForm({ credential, onDone, onCancel, onGoToSecurity })`:
    - `SensitiveAction` (`requiresStepUp`), com os campos "Usuário do Portal de Serviços" e "Senha" (`type: "password"`);
    - → `setBnafarCredential(username.trim(), password, credential.environment ?? "production")`. O ambiente é o da linha, sem escolha na tela (Q9);
    - frase de sucesso: "Credencial da BNAFAR salva. Teste a conexão para conferir." (proposta).
  - `Integrations.tsx`:
    - a linha `bnafar` abre o `BnafarCredentialForm`;
    - o teste da linha chama `checkBnafarCredential(c.environment ?? "production")`;
    - o subtítulo vira "e-SUS PEC · CADSUS · RNDS · BNAFAR".

- [ ] **Step 1: Testes que falham** (`Integrations.bnafar.test.tsx`):
  - uma linha `bnafar`, com o rótulo do ambiente em uso;
  - "Cadastrar" pede o step-up, e o corpo leva a senha como digitada e o ambiente da linha;
  - nenhuma chave do `queryClient` tem a senha;
  - "Testar conexão" manda `{ environment }`;
  - os `missing` `vector_supplies_disabled` e `credential_missing:bnafar` aparecem com as frases.

  `src/lib/integrations.test.ts`: `credentialLabel("bnafar", "homologation")` e `missingPhrase("bnafar_disabled")` = `"ligue antes: Envio à BNAFAR"`.
- [ ] **Step 2–4:** rode e veja falhar (os dois arquivos e `src/modules/Integrations.test.tsx src/modules/Integrations.rnds.test.tsx`); implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/integrations.ts src/lib/integrations.test.ts src/modules/Integrations.tsx src/modules/integrations/BnafarCredentialForm.tsx src/modules/Integrations.bnafar.test.tsx
$G commit -m "feat: set and test the BNAFAR credential of the environment in use" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 8: "Envio à BNAFAR"

**Files:**
- Create: `src/modules/bnafar/AlertBand.tsx`, `src/modules/bnafar/SubmissionDetail.tsx`, `src/modules/bnafar/CorrectionForm.tsx`, `src/modules/BnafarStatus.test.tsx`
- Modify: `src/modules/BnafarStatus.tsx`

**Interfaces:**
- Consumes: `BnafarGate` (Task 3); `getBnafarStatus`, `listBnafarSubmissions`, `getBnafarSubmission`, `retryBnafarSubmission`, `correctBnafarRecord`, `getExternalExit`, `getBnafarDomains` (Task 1); `alertLines`, os rótulos, `bnafarErrorText`, `bnafarError`, `externalExitProblems`, as chaves (Task 2).
- Produces:
  - `BnafarStatus({ onGoToSecurity? })`, dentro de `BnafarGate title="Envio à BNAFAR" sub="situação de cada dia, recusas e prazo de correção" allow={(r) => r.reader}`:
    - o `AlertBand` (`role="alert"`, uma linha por `alertLines`; nada sem alerta);
    - "Ambiente" e "Última remessa concluída" (`last_finished_on`, ou "nenhuma ainda");
    - filtros "De", "Até", "Assunto" e "Situação", que só valem no "Filtrar";
    - a lista `listBnafarSubmissions` (`useInfiniteQuery`, "Carregar mais"), com as colunas "Dia", "Assunto", "Situação", "Registros" (só as contagens que não são zero), "Protocolos" (os `batches[].protocol`, Q8) e "Ver registros — <dia> <assunto>".
  - `SubmissionDetail({ id, admin, onGoToSecurity, onClose })`:
    - colunas:
      - "Local": o `location_name` (Q16);
      - "Item": o `item_label`;
      - "Situação": pelo `recordStatusLabel`;
      - "Motivo": os `error_codes` e os de `rejected_items` por `bnafarErrorText`, com o lote na frente; no registro `lost`, o `lost_reason` (Q16);
      - "Prazo": o `retry_until` do recusado ou o `correct_until` da saída aceita, sempre do api, nunca calculado.
    - Com `admin` e algum registro `rejected` com `retry_until`, "Reenviar recusados" pelo `SensitiveAction` (`requiresStepUp`; descrição proposta: "Remonta e reenvia só os registros recusados desta remessa, com os dados corrigidos.") → `retryBnafarSubmission`; depois, relê o detalhe, a lista e o status.
  - `CorrectionForm({ record, domains, onDone, onCancel, onGoToSecurity })`:
    - só para registro com `external_exit_id`, `accepted` e `correct_until`, e só para `admin`;
    - abre com o tipo, o destino e a versão atuais (`getExternalExit`);
    - campos tipo, destino e "Por que corrigir" (10–500);
    - "Corrigir" → `correctBnafarRecord`, que vale para a saída inteira (Q8);
    - `409 external_exit_changed` → a frase e uma releitura;
    - sucesso: "Correção enviada. A BNAFAR substitui os registros desta saída." (proposta).

- [ ] **Step 1: Testes que falham** (`BnafarStatus.test.tsx`):

```ts
it("faixa do alerta e lista por dia e assunto, só leitura para o almoxarife", async () => { /* stock_keeper não vê "Reenviar recusados" nem "Corrigir" */ });
it("reenviar: step-up, um pedido, janela fechada relê e mostra perdidos", async () => { /*
  admin; duplo clique → 1 chamada; 401 mfa_required volta ao step-up sem nova chamada;
  409 retry_window_closed → frase; getBnafarSubmission relido uma vez com lostRecord() → "perdido" e o motivo */ });
it("corrigir só a saída externa aceita dentro do prazo; motivo e destino conferidos antes; 409 relê", async () => { /* */ });
it("aceito em parte, nomes do local e do item, MSG traduzida e desconhecida crua", async () => { /* MSG72; MSG99 */ });
it("sem bnafar: frase do interruptor e nenhuma leitura", async () => { /* */ });
```

- [ ] **Step 2–4:** rode e veja falhar; implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/BnafarStatus.tsx src/modules/BnafarStatus.test.tsx src/modules/bnafar/AlertBand.tsx src/modules/bnafar/SubmissionDetail.tsx src/modules/bnafar/CorrectionForm.tsx
$G commit -m "feat: show the daily BNAFAR submissions with retry of rejected records and correction of external exits" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 9: "Pendências do envio à BNAFAR", com o complemento

**Files:**
- Create: `src/modules/BnafarPending.test.tsx`
- Modify: `src/modules/BnafarPending.tsx`

**Interfaces:**
- Consumes: `BnafarGate` (Task 3); `LotBnafarForm` (Task 5); `listBnafarPendencies` (Task 1); `pendencyLabel`, `bnafarRoles`, `BNAFAR_FIELD_LABEL`, `BNAFAR_PENDENCIES_KEY` (Task 2); `SegmentedControl`.
- Produces: `BnafarPending({ onGoToLocations, onGoToColdChain, onGoToVectorProducts, onGoToAdjustments })`, dentro de `BnafarGate title="Pendências do envio à BNAFAR" sub="o que fica fora do envio de amanhã e como resolver" allow={(r) => r.reader}`:
  - filtro por código (`SegmentedControl`: "Todas" e os sete rótulos);
  - uma seção por código presente, com o título do rótulo e o `counts[code]`;
  - colunas "Local" (`location_name`), "Item" (`item_label`), "Lote", "O que falta" (os `fields` por `BNAFAR_FIELD_LABEL`) e a ação.

  Ações por código:
  - `lot_incomplete` → "Completar dados do lote — <lote>" abre o `LotBnafarForm` abaixo da linha (Q16, sem tela própria), para o almoxarife, o farmacêutico ou o admin; o api confere o local. A `state` vem de `lot_version` e `fields`. Salvar relê as pendências.
  - `location_without_cnes`, pelo `location_kind`:
    - `central` → "Abrir Locais e configurações" (só admin);
    - `cold_chain_central` → "Abrir Salas e rede de frio" (só admin);
    - os demais → "o CNES deste local vem da unidade de saúde — confira o cadastro da unidade" (proposta).
  - `product_without_code`:
    - `vector_control` → "Abrir Produtos e parâmetros";
    - medicamento → "o VMP vem do catálogo de medicamentos da plataforma — peça à equipe do Rota Saúde";
    - vacina → "o código OBM vem do catálogo de imunobiológicos — peça à equipe do Rota Saúde" (proposta).
  - `fractional_balance` → "Abrir Inventário e ajustes".
  - `exit_type_missing`, `domain_off` e `lot_number_too_long` → só a frase.

- [ ] **Step 1: Testes que falham** (`BnafarPending.test.tsx`):
  - as sete pendências aparecem nas seções, com as frases e as contagens do api;
  - o filtro manda `code=`;
  - "Completar dados do lote" abre o formulário, manda `expected_version: 0` num lote antigo e, depois de salvar, relê a lista;
  - o almoxarife não vê "Abrir Locais e configurações";
  - código desconhecido aparece cru, numa seção própria;
  - nenhuma coluna de pessoa.
- [ ] **Step 2–4:** rode e veja falhar; implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/BnafarPending.tsx src/modules/BnafarPending.test.tsx
$G commit -m "feat: list the BNAFAR pending items by code with the lot complement and shortcuts" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 10: "Saída externa"

**Files:**
- Create: `src/modules/pharmacy/ExternalExitForm.tsx`, `src/modules/pharmacy/ExitLocationPick.tsx`, `src/modules/ExternalExits.test.tsx`
- Modify: `src/modules/ExternalExits.tsx`

**Interfaces:**
- Consumes: `listMyPharmacyLocations`, `listPositions`, `lotsWithBalance`, `locationRoles`, `canUseStockScreens`, `PHARMACY_DISABLED` (27a); `listMyVaccineLocations`, `locationAbilities`, `canUseVaccination` (29); `recordExternalExit`, `listExternalExits`, `getExternalExit`, `getBnafarDomains` (Task 1); `externalExitProblems`, `externalExitBody`, `exitTypeLabel` (Task 2). Molde: `TransferForm`.
- Produces:
  - `ExternalExits({ today?, onGoToSecurity? })`: **sem** o `bnafar` (Q16). Sem `canUseStockScreens || canUseVaccination`, mostra `PHARMACY_DISABLED`; sem `stock_keeper` nem `municipal_admin`, mostra o `PHARMACY_NO_ROLE`. O almoxarife escolhe o local e vê o formulário e a lista do local; o admin vê só a lista da cidade.
  - `ExitLocationPick({ children })` (Q16: os locais de todos os domínios): junta os locais da farmácia com `locationRoles(loc).keeper` e os da vacina com `locationAbilities(loc).keeper`, em ordem de nome, com o tipo ao lado; sem nenhum, `NO_LOCATION`.
  - `ExternalExitForm({ location, today, domains, onDone, onGoToSecurity })`:
    - campos: "Tipo de saída", "Destino" (CNES ou CNPJ), "Motivo" (10–500), linhas por lote com saldo e "Quantidade (inteira)";
    - confere tudo antes de enviar;
    - lote de vacina só em múltiplo das doses do frasco: o `invalid_quantity` do api aparece na linha;
    - "Registrar saída" pelo `SensitiveAction` (`requiresStepUp`);
    - `422 insufficient_balance` → a frase do 27a e **uma** releitura das posições;
    - sucesso: "Saída registrada. Ela vai à BNAFAR no envio de amanhã, se o envio estiver ligado." (proposta).
  - A lista: "Data", "Tipo", "Destino", "Itens" e "Por". "Ver motivo" lê `getExternalExit` no clique.

- [ ] **Step 1: Testes que falham** (`ExternalExits.test.tsx`; o caso do Review Focus 4 tem o nome "saída: inteira, até o saldo, documento normalizado, corrida de saldo relê uma vez"):
  - `"2,5"`, `"0"` e quantidade acima do saldo recusados antes;
  - `"12.abc.345/01de-35"` vai `"12ABC34501DE35"`;
  - CNES `"23840430"` recusado;
  - step-up, e um pedido no duplo clique;
  - `insufficient_balance` relê `listPositions` uma vez;
  - sem `bnafar` na sessão, a tela funciona (Q16);
  - uma sala de vacina do almoxarife aparece no seletor;
  - o admin não vê o formulário.
- [ ] **Step 2–4:** rode e veja falhar; implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/modules/ExternalExits.tsx src/modules/ExternalExits.test.tsx src/modules/pharmacy/ExternalExitForm.tsx src/modules/pharmacy/ExitLocationPick.tsx
$G commit -m "feat: record external stock exits from any stock domain with destination, reason and step-up" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 11: "Saldo inicial" e a lista das aberturas

**Files:**
- Create: `src/lib/openingImport.ts`, `src/lib/openingImport.test.ts`, `src/modules/openingImport/ImportReport.tsx`, `src/modules/OpeningImport.test.tsx`, `src/modules/pharmacy/OpeningsPanel.tsx`, `src/modules/PharmacyEntries.openings.test.tsx`
- Modify: `src/modules/OpeningImport.tsx`, `src/modules/PharmacyEntries.tsx`

**Interfaces:**
- Consumes: `uploadOpeningImport`, `getOpeningImport`, `applyOpeningImport`, `revertOpeningImport`, `discardOpeningImport`, `fetchOpeningImportTemplate`, `listOpeningImports`, `listOpenings` (Task 1); `BnafarGate`, `bnafarRoles` (Tasks 2–3); `listMyPharmacyLocations`; `saveFile`; `SensitiveAction`; `ConfirmButton`.
- Produces (`src/lib/openingImport.ts`):
  - `OPENING_IMPORT_HEADER = "cnes;tipo_produto;codigo_produto;ampp;lote;validade;quantidade;valor_unitario;cnpj_fabricante;registro_anvisa;programa"` (contrato §5), `OPENING_IMPORT_MAX_BYTES = 5 * 1024 * 1024` (Q1, Q11) e `ROWS_PAGE = 100`.
  - `fileProblem(file)`, que recusa (frases proposta):
    - sem arquivo → "escolha a planilha (.csv)";
    - outra extensão → "a planilha é um arquivo .csv separado por ponto e vírgula — salve como CSV UTF-8";
    - acima do teto → "o arquivo passa de 5 MB — divida a planilha por local".
  - `headerProblem(firstLine)`: tira o BOM e o `\r` (Q17). Cabeçalho diferente → "o cabeçalho não é o da planilha padrão — baixe o modelo e copie as colunas na mesma ordem".
  - `readFirstLine(file)`.
  - `ROW_ERRORS` e `rowErrorText` (desconhecido cru). Os códigos são os do plano do api (Valor 11; Q17), e as frases são proposta:

    | Código | Frase |
    |---|---|
    | `cnes_unknown` | "CNES sem local de estoque nesta cidade" |
    | `product_unknown` | "produto não encontrado no catálogo" |
    | `invalid_product_type` | "tipo_produto deve ser medicamento, vacina ou vetorial" |
    | `item_not_allowed_here` | "o produto não entra neste tipo de local" |
    | `invalid_lot_number` | "lote vazio ou longo demais" |
    | `invalid_date` | "validade fora do formato AAAA-MM-DD" |
    | `expired_lot` | "lote vencido" |
    | `invalid_quantity` | "quantidade inteira maior que zero, na unidade da BNAFAR (frasco na vacina)" |
    | `invalid_price` | "valor unitário com vírgula, maior que zero" |
    | `invalid_cnpj` | "CNPJ do fabricante inválido" |
    | `invalid_anvisa` | "registro ANVISA com 11 ou 13 dígitos" |
    | `ampp_unknown` | "AMPP fora do catálogo OBM" |
    | `anvisa_mismatch` | "o registro ANVISA não é o do AMPP" |
    | `program_unknown` | "programa fora da tabela DM14" |
    | `manufacturer_not_in_catalog` | "fabricante da vacina fora do catálogo de imunobiológicos" |
    | `duplicate_line` | "linha repetida (mesmo local, produto e lote)" |
    | `opening_exists` | "este item já tem abertura neste local" |
    | `too_many_lots` | "mais de 100 lotes do mesmo item num local" |
  - `IMPORT_ERRORS` e `importErrorText(err)` (frases proposta):

    | Código | Frase |
    |---|---|
    | `invalid_file` | "o arquivo não pôde ser lido — confira se é o CSV UTF-8 da planilha padrão" |
    | `file_too_large` | "o arquivo passa de 5 MB — divida a planilha por local" |
    | `revert_blocked` | "não dá para desfazer: um local da importação já teve outro movimento ou envio à BNAFAR" |
    | `opening_import_changed` (proposta, D12) | "a importação mudou enquanto a tela estava aberta — ela foi relida; confira e repita" |
    | `import_in_progress` (proposta, D12) | "já há uma importação aberta nesta cidade — aplique ou descarte antes" |
    | `missing_role` | "só o administrador da cidade ou o almoxarife do almoxarifado central importa o saldo inicial" |
  - `importStatusView(status)`: `uploaded` "recebida", `validated` "conferida", `applied` "aplicada", `discarded` "descartada", `reverted` "desfeita" (proposta).
  - `fmtCentsBRL`.
  - Frases de resultado (proposta):
    - `appliedText(i)`: "Saldo inicial aplicado: <n> linhas em <m> locais. O envio à BNAFAR desses locais começa amanhã." (decisão 14);
    - `revertedText`: "Importação desfeita: as aberturas foram estornadas.";
    - `discardedText`: "Importação descartada; o arquivo foi apagado.".
- Produces (telas):
  - `OpeningImport({ onGoToSecurity? })`, dentro de `BnafarGate title="Saldo inicial" sub="planilha do saldo de abertura, conferência e aplicação" allow={(r) => r.admin || r.keeper}`. Pelo §6, o `bnafar` é exigido; D12 propõe tirar.
    - O almoxarife sem papel num local `central` (`listMyPharmacyLocations`) vê a frase de `missing_role`, sem leitura.
    - A nota: "A planilha vira o saldo de abertura de cada local depois de conferida e aplicada. O envio à BNAFAR do local começa no dia seguinte, e desfazer só vale antes de qualquer outro movimento ou envio." (proposta).
    - "Baixar o modelo": `fetchOpeningImportTemplate` e `saveFile(file, file.filename ?? "modelo-saldo-inicial.csv")`.
    - A lista `listOpeningImports` (D12), com situação, linhas válidas de total, quem e quando, e "abrir" em cada uma. Sem a lista no api, a tela abre só a importação que acabou de enviar.
    - O envio:
      - `<input type="file" accept=".csv,text/csv" aria-label="Planilha do saldo inicial">`;
      - ao escolher o arquivo, confere `fileProblem` e `headerProblem(await readFirstLine(file))`; se recusado, mostra a frase e não faz **nenhuma** chamada;
      - "enviar e conferir" (`ConfirmButton`) → `uploadOpeningImport(file)`;
      - depois, solta o arquivo do estado e abre o relatório.
  - `ImportReport({ importId, admin, onGoToSecurity, onChanged })`:
    - lê `getOpeningImport(id, { after })`;
    - mostra os totais, os erros por código (`rowErrorText` e a contagem), "Produtos sem catálogo", "CNES sem local", "Totais por local" (com `fmtCentsBRL`) e as linhas em `DataTable`;
    - filtro "só com erro" e "carregar mais" pelo `next_after`.
  - Ações pela situação:
    - `validated` → "Aplicar <válidas> linhas" pelo `SensitiveAction` (`requiresStepUp`) → `applyOpeningImport(id, version)`, e "Descartar" (`ConfirmButton`);
    - `applied` → "Desfazer" pelo `SensitiveAction` → `revertOpeningImport`;
    - `discarded` e `reverted` → só leitura.

    Um `409` de versão mostra a frase e faz uma releitura. Depois de cada ato, relê o relatório, a lista e as aberturas. Rede ou timeout: relê e mantém a tela, sem reenviar.
  - `OpeningsPanel({ location })` em "Entradas e abertura" (Q13; com a Q11, a marca da plataforma saiu):
    - "Aberturas deste local" (proposta), com "Carregar mais";
    - colunas "Data", "Itens" e "Por" (o `recorded_by.name`, o autor normal);
    - abertura `reverted` ganha a `Tag` "desfeita" (proposta).

- [ ] **Step 1: Testes que falham.**
  - `src/lib/openingImport.test.ts`:
    - `fileProblem` por tabela: sem arquivo, `.xlsx`, 5 MB + 1, e `SALDO.CSV` com exatamente 5 MB, que passa;
    - `headerProblem`: o cabeçalho exato passa; com BOM e `\r`, passa; com vírgula ou com colunas trocadas, recusa;
    - as frases não têm `_`, e o código desconhecido volta cru;
    - `fmtCentsBRL(123456)` = `"R$ 1.234,56"`.
  - `OpeningImport.test.tsx`:

  | Caso | Asserções |
  |---|---|
  | "arquivo recusado antes: nenhuma chamada" | `.xlsx`, 6 MB e cabeçalho com vírgula → cada frase; `uploadOpeningImport` não chamado |
  | "envio com BOM: relatório aberto e nome fora do corpo" | `uploadOpeningImport` chamado com o arquivo; totais, erros por código, "CNES sem local" e "Totais por local" com `R$` |
  | "linhas em páginas: after na segunda leitura" | "carregar mais" → `getOpeningImport("oi1", { after: 100 })` |
  | "aplicar: step-up, um pedido, versão na chamada" | duplo clique → `applyOpeningImport("oi1", 1)` uma vez; `401 mfa_required` volta ao step-up; sucesso → `appliedText` e releitura |
  | "desfazer bloqueado: frase e situação mantida" | `422 revert_blocked` → a frase; continua "aplicada" |
  | "descartar: confirmação e só leitura depois" | `discardOpeningImport("oi1", 1)`; nenhum botão de aplicar depois |
  | "almoxarife de unidade não importa" | `stock_keeper` só de `unit` → a frase de `missing_role`, sem leitura |

  - `PharmacyEntries.openings.test.tsx`:
    - a lista mostra o autor;
    - a abertura desfeita tem "desfeita";
    - registrar uma abertura relê o painel.
- [ ] **Step 2–4:** rode e veja falhar (os três arquivos e `src/modules/PharmacyEntries.test.tsx`); implemente; rode e veja passar; `tsc`.
- [ ] **Step 5: Commit**

```bash
cd $W && $G add src/lib/openingImport.ts src/lib/openingImport.test.ts src/modules/OpeningImport.tsx src/modules/OpeningImport.test.tsx src/modules/openingImport/ImportReport.tsx src/modules/pharmacy/OpeningsPanel.tsx src/modules/PharmacyEntries.tsx src/modules/PharmacyEntries.openings.test.tsx
$G commit -m "feat: import, review, apply and revert the opening stock spreadsheet in the city" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 12: Suíte, build, varreduras, revisão e prova no navegador (api na 3062, Vite na 5223)

**Files:** `$R/.claude/launch.json` (entrada nova; fora de git). Correção que a prova pedir entra na task dona, com teste.

- [ ] **Step 1: Suíte, tipos e build**

```bash
$R/bin/test dashboard -w dashboard-mod27b
cd $W && npx tsc --noEmit && npx vite build >/dev/null && echo build-ok
```

Expected: verde e `build-ok`. A contagem é a base da Task 0 + **18 arquivos de teste novos**:
- `src/lib/api.bnafar.test.ts`, `src/lib/bnafarErrors.test.ts`, `src/lib/bnafar.test.ts`, `src/lib/openingImport.test.ts`;
- `src/shell/modules.bnafar.test.ts`;
- `src/modules/bnafar/BnafarGate.test.tsx`, `src/modules/bnafar/LotBnafarForm.test.tsx`;
- `src/modules/pharmacy/ReceiptForm.bnafar.test.tsx`, `src/modules/vaccination/VaccineReceiptForm.bnafar.test.tsx`;
- `src/modules/PharmacyAdmin.bnafar.test.tsx`, `src/modules/VaccinationAdmin.bnafar.test.tsx`, `src/modules/VectorSettings.bnafar.test.tsx`;
- `src/modules/Integrations.bnafar.test.tsx`;
- `src/modules/BnafarStatus.test.tsx`, `src/modules/BnafarPending.test.tsx`, `src/modules/ExternalExits.test.tsx`;
- `src/modules/OpeningImport.test.tsx`, `src/modules/PharmacyEntries.openings.test.tsx`.

Contagem que dobra = artefato de build achado pelo vitest. Erro não tratado com tudo verde = mock sem `fetchCurrentSession`.

- [ ] **Step 2: Varreduras**

```bash
cd $W
O="src/modules/bnafar src/modules/openingImport src/modules/Bnafar*.tsx src/modules/ExternalExits.tsx src/modules/OpeningImport.tsx src/modules/pharmacy/ExternalExitForm.tsx src/modules/pharmacy/ExitLocationPick.tsx src/modules/pharmacy/BnafarLotFields.tsx src/modules/pharmacy/AmppPick.tsx src/modules/pharmacy/OpeningsPanel.tsx src/modules/integrations/BnafarCredentialForm.tsx src/lib/bnafar.ts src/lib/bnafarErrors.ts src/lib/openingImport.ts"
grep -rnE "setInterval|refetchInterval: [0-9]" $O && echo "ERRO: polling" || echo ok-polling
grep -rnE "localStorage|sessionStorage|indexedDB|console\.(log|info|warn|error)" $O && echo "ERRO: storage/log" || echo ok-storage
grep -rnE "innerHTML|dangerouslySetInnerHTML" $O && echo "ERRO: html cru" || echo ok-html
grep -rnE "queryKey:.*(password|username|reason|file)" $O && echo "ERRO: segredo, motivo ou arquivo no cache" || echo ok-cache
sed -n '/Envio à BNAFAR, saída externa e saldo inicial (módulo 27b/,$p' src/lib/api.ts | grep -nE "password|username|reason|destination_code|file_base64" | grep -E "withQuery|URLSearchParams|\?\$\{" && echo "revisar: URL" || echo ok-url
for f in $(grep -rlE 'vi\.mock\("\.\./(\.\./)?lib/api", async' src/modules/*.bnafar.test.tsx src/modules/Bnafar*.test.tsx src/modules/ExternalExits.test.tsx src/modules/OpeningImport.test.tsx src/modules/PharmacyEntries.openings.test.tsx src/modules/pharmacy/*.bnafar.test.tsx src/modules/vaccination/*.bnafar.test.tsx src/modules/bnafar 2>/dev/null); do
  grep -q "fetchCurrentSession" "$f" || echo "ERRO: $f sem fetchCurrentSession"; done; echo ok-mocks
```

Expected: `ok-polling`, `ok-storage`, `ok-html`, `ok-cache`, `ok-url`, `ok-mocks`, sem `ERRO`.

- [ ] **Step 3: Conferência do api.** Repita a Task 0 Step 4 contra o api final. Forma diferente vira ajuste na task dona antes da prova.

- [ ] **Step 4: Revisão do ramo** (superpowers:requesting-code-review). Foco:
  - as cinco linhas do Review Focus;
  - a tabela `BNAFAR_ERRORS`;
  - os campos exigidos por domínio utilizável;
  - o step-up das oito escritas;
  - a planilha fora de URL e de cache.

- [ ] **Step 5: Entrada do Vite.** Acrescente ao `$R/.claude/launch.json`, depois da `dashboard-mod35b`:

```json
{
  "name": "dashboard-mod27b",
  "runtimeExecutable": "bash",
  "runtimeArgs": [ "-c", "cd .worktrees/dashboard-mod27b && PATH=$HOME/.nvm/versions/node/v22.19.0/bin:$PATH VITE_API_PROXY_TARGET=http://localhost:3062 npx vite --port 5223 --host 0.0.0.0" ],
  "port": 5223
}
```

- [ ] **Step 6: Prova no navegador.** Antes de começar:
  - **Autorização do usuário:** migrar os bancos de dev (migrações de cidade a partir de `20261125900001`, de plataforma a partir de `20261125600001`) e rodar a semente do 27b em Curitiba (plano do api, Task 16) só com autorização dele. Peça antes de qualquer `db:migrate` ou `db:seed`.
  - **Dono de cada serviço:** o api do worktree `apps/api/.claude/mod27b` na **3062** e o simulado `fake-bnafar-dev` na **8095** são da sessão do api; o maintenance na 5177 é da sessão dele.
  - **Abrir:** `curitiba.localhost:5223/dashboard/` (`preview_start` com `dashboard-mod27b`).
  - **Senhas:** a senha e o TOTP de dev podem ser mostrados; os de staging, nunca.

  Roteiro, um passo por linha:

1. **Interruptores** (maintenance 5177): `bnafar`, `bnafar_mock` e `bnafar_medications` ligados. Antes da credencial, "Envio à BNAFAR" mostra a frase do interruptor.
2. **Credencial** (`municipal_admin`): em Integrações, cadastrar com step-up e "Testar conexão" contra o simulado. Na aba de rede, a senha só aparece no corpo do `PUT`.
3. **CNES**: o do almoxarifado central em "Locais e configurações" e o da rede de frio em "Salas e rede de frio".
4. **Saldo inicial** (`stock_keeper` do central):
   - baixar o modelo;
   - um `.xlsx` é recusado antes de enviar;
   - a planilha da semente gera o relatório (CNES sem local, produto sem catálogo, abertura repetida);
   - aplicar com step-up; "Entradas e abertura" mostra as aberturas com o autor;
   - desfazer antes de qualquer movimento.
5. **Entrada**: nota fiscal com três itens: a linha sem valor é recusada antes; o AMPP preenche ANVISA e CNPJ; um item sem VMP entra; um fabricante internacional entra sem CNPJ.
6. **Pendências do envio à BNAFAR**: as da semente; "Completar dados do lote" num lote antigo, e a pendência some depois de salvar; os atalhos levam às telas certas.
7. **Saída externa**: doação com CNPJ, com step-up, inclusive de uma sala de vacina, com o `bnafar` desligado por um momento (depois, religue). O extrato do lote mostra "saída para fora da rede".
8. **Envio**: a sessão do api roda o `Bnafar::DailyJob` e o `Bnafar::DeliverJob` contra o simulado. "Envio à BNAFAR" mostra:
   - a remessa do dia anterior;
   - um registro recusado com a frase da MSG e um aceito em parte;
   - "Reenviar recusados" com step-up;
   - "Corrigir" na saída externa aceita.

  Também:
  - preset mobile 375×812 nos passos 4, 5 e 8;
  - só teclado nos passos 2 e 4;
  - `resize_window` com o preset desktop ao terminar.

  Anote cada passo. Achado que muda comportamento vira teste na task dona antes da correção. Achado só visual vai ao usuário (memória "Interface em ciclo próprio").

---

### Task 13: Entrega — board, relato e parada antes do merge

**Files:** nenhum.

- [ ] **Step 1: Board.** Os cards do 27b que o dashboard toca vão a **Done** só se o api já terminou o deles, e a **Verified** só depois da prova da Task 12, com o usuário.
- [ ] **Step 2: Suíte completa do api.** Não é desta sessão: a sessão dona do api faz o pedido à sessão "Testes" (`api · mod27b/feat/mod-27b-bnafar · --full · mudança transversal do 27b`). A suíte inteira do dashboard neste worktree (Task 12 Step 1) basta para o dashboard.
- [ ] **Step 3: Pare e peça autorização** para merge e push (ordem do §10: depois do api e da suíte completa).
  - Antes do push, confira `$G -C $W log --oneline origin/main..HEAD` e `origin/main..main` (sessões paralelas).
  - O worktree só sai depois do merge: `$G -C $R/apps/dashboard worktree remove $W`.

**Relato:** `dashboard · dashboard-mod27b/feat/mod-27b-bnafar · Task 13 · suíte <n>/<n>, build ok, varreduras ok, prova <passos> · cards movidos · aguardando autorização de merge`.

---

## Self-review

1. **Cobertura do contrato §7, da Q11 e da spec §4.6:**
   - "Envio à BNAFAR", com faixa, lista, recusados e prazo, "Reenviar recusados" e "Corrigir": Task 8;
   - "Pendências do envio à BNAFAR", com o complemento (Q16): Tasks 5 e 9;
   - "Saída externa" sem `bnafar` e com todos os domínios (Q16): Task 10;
   - campos novos com AMPP, VMP e fabricante internacional (Q15): Task 4;
   - CNES (Q13) e CATMAT: Task 6;
   - credencial do ambiente em uso (Q9): Task 7;
   - `BNAFAR_ERRORS`: Task 2;
   - "Saldo inicial" e as aberturas com o autor (Q11, Q13): Task 11.
2. **Bordas que tocam a tela:**
   - valor com milhar e casas: Task 2;
   - CNPJ alfanumérico e com pontuação: Tasks 2 e 10;
   - quantidade fracionária e acima do saldo: Tasks 2 e 10;
   - domínio não utilizável não exige: Tasks 2 e 4;
   - produto sem VMP e fabricante internacional: Task 4;
   - lote com versão 0: Tasks 5 e 9;
   - janela do reenvio e registro perdido: Task 8;
   - planilha com BOM, grande ou errada, e desfazer bloqueado: Task 11.
3. **Tipos:**
   - `LotBnafarState`, `LotBnafarDraft` e `BnafarField` (Tasks 1–2) são os mesmos nas Tasks 4, 5 e 9;
   - `BnafarLotFields` (Task 4) serve à Task 5;
   - `externalExitProblems` (Task 2) serve às Tasks 8 e 10;
   - `bnafarRoles` (Task 2) governa as Tasks 3, 8, 9 e 11.
4. **Review Focus:** as cinco linhas têm teste na task dona (Tasks 8, 4, 2, 10 e 11).
5. **Proporção:** o código do plano é tipo, frase e teste; as telas vão por assinatura, props e comportamento.

## Divergências propostas ao contrato

As D1–D10 foram decididas no §11 do contrato (docs `ae1cfbc`), e o plano já aplica as decisões. O texto de cada uma fica como registro. D11 e D12 são novas e esperam decisão.

- **D1 — Resolvida pela Q16** (com a Q8 e a Q9). Formas das respostas:
  - registros e pendências com `location_name` e o rótulo do item;
  - um protocolo por parte (`bnafar_batches`);
  - a credencial do ambiente em uso.

  Ficam propostas, a conferir na Task 0: as contagens da remessa e o `location_kind`, `lot_number` e `lot_version` da pendência, que a Task 9 usa no atalho e no complemento.
- **D2 — Resolvida pela Q17.** O `path` é o do api real, `items[i].campo`.
- **D3 — Resolvida pela Q15.** O AMPP só é exigido com VMP; o fabricante internacional entra pelo nome, com o CNPJ vazio. O nome do campo (`manufacturer_international_name`) é proposta, e a Task 0 confere.
- **D4 — Resolvida pela Q16.** Entram `GET /pharmacy/bnafar/domains` e `GET /pharmacy/obm_presentations?vmp_code=`.
- **D5 — Resolvida pela Q13.** O CNES do central vai pelo `PATCH /pharmacy/admin/locations/:id`; o da rede de frio, pelo `PATCH /vaccination/admin/cold_chain_central`.
- **D6 — Resolvida pela Q16.** O alerta vale às 15:00 de D+1, no fuso da cidade; o registro perdido traz `lost` e o motivo.
- **D7 — Resolvida pela Q13 e pela Q11.** As aberturas se leem por `GET /pharmacy/openings`. Com a importação feita pela cidade, a marca "importada pela plataforma" sai, e a abertura mostra o autor normal.
- **D8 — Resolvida pela Q16.** A saída externa exige só o estoque do domínio utilizável, sem o `bnafar`, e o seletor mostra os locais de todos os domínios.
- **D9 — Resolvida pela Q16.** O complemento fica dentro de "Pendências do envio à BNAFAR", sem tela própria, e o lote antigo começa com `bnafar_version` 0.
- **D10 — Resolvida pela Q16 e pela Q6.**
  - O rótulo do menu é "Pendências do envio à BNAFAR".
  - O `bnafar_mock` é a quinta chave, com o rótulo "BNAFAR simulado".
  - "Domínio ligado" quer dizer utilizável.
- **D11 — Nova. A tela não sabe quais domínios estão utilizáveis.**
  - A Q6 diz que "domínio ligado" quer dizer interruptor utilizável, e é isso que torna os campos obrigatórios na entrada.
  - A sessão só traz as chaves **ligadas** (`src/lib/features.ts:1-4`).
  - O estado utilizável só vem no `GET /integrations`, que é do `municipal_admin`.
  - Sem saber o domínio utilizável, a tela exigiria os campos com o `bnafar` ligado e ainda sem credencial, e travaria a entrada que o api aceitaria.

  **Proposta:** o `GET /pharmacy/bnafar/domains` traz `enabled_domains` (os domínios de `Bnafar::Gate.domains(city)`), sem exigir o `bnafar`, para o almoxarife também. Até a decisão, a Task 2 usa as chaves ligadas da sessão.
- **D12 — Nova. Pontos do saldo inicial que a Q11 não fixa.**
  - Pelo §6, toda rota de cidade exige `Bnafar::Gate.usable?`, salvo a credencial e o complemento. Isso alcança as rotas da Q11, mas a importação é estoque e vem antes do primeiro envio (decisão 14): a cidade que sai do Hórus abre o estoque antes de ter a credencial.
  - Não há lista das importações da cidade. Quem recarrega a página perde o id da importação conferida.
  - Não há versão para aplicar, descartar e desfazer. Duas abas podem aplicar a mesma importação.
  - Os campos do relatório não estão fixados.

  **Proposta:**
  - as rotas de `/pharmacy/opening_imports` seguem o estoque do domínio (`StockGate`), sem o `bnafar`; o portão da Task 3 muda numa linha;
  - `GET /pharmacy/opening_imports` lista as importações da cidade;
  - uma importação aberta por cidade, com `import_in_progress`;
  - `expected_version` nas três ações, com `409 opening_import_changed`;
  - o relatório com `errors_by_code`, `products_without_catalog`, `cnes_without_location` e `totals_by_location` (Task 1).
