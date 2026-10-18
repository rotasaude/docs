# Módulo 21c — Qualidade e insumos do laboratório municipal (api, contracts e fake-middleware) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **PRÉ-REQUISITO: módulo 21b em `origin/main` do api** (conferido ao escrever este plano em `2deea81`: `Platform::Features::CATALOG` com 17 chaves terminando em `municipal_lab`, `lab_hl7_inbound`, `fake_middleware`; `Lab::Gate.{usable?,hl7_usable?,fake_middleware_usable?}`; `Lab::Access.{labs,staff?,admin?,releaser_link,releaser_provider_ids,provider_ids,member?,council}`; `Lab::Release.call(requisition:, item_ids:, signature:, totp:, by:, now:, rectifies:, reason:, contacts:)`; `Lab::EnterResults.{call,recalculate!,context}` com `create_row` privado; `Lab::ChooseResult`; `Lab::Hl7::{Message,Builder,Inbound}` com `Inbound::Application`; `Lab::CreateVersion`; `Lab::AdminJson.version`; `Lab::EntryJson.result_row`; `lib/fake_middleware/{client,guard,messages,run,runner}.rb`; `lib/lab_crew.rb` (`LabCrew.seed_current_city`); `spec/support/lab_helpers.rb`; última migração de cidade `20261015900002_add_municipal_lab.rb` e de plataforma `20261015900001_create_lab_exam_templates.rb`; `VALID_RANGE = (1..37)`). **A Task 0 confere a base e faz a prova técnica da HL7: se um nome mudou, ou se a prova contradiz este plano fora do que a regra da Task 0 cobre, PARE e reporte.**

**Goal:** O lado api do módulo 21c (F-21.25 a F-21.34, ADR 0038): interruptores `lab_quality_control` e `lab_inventory`; aparelhos e o vínculo exame × aparelho; CIQ completo (materiais, lotes com códigos de barras por nível, referências versionadas da bula e do laboratório, corridas, Westgard avaliado na gravação, regras por analito, bloqueio da liberação do 21b, exceção do RT, revisão, sugestão da referência após 20 corridas, Levey-Jennings); controle pela HL7 (`OUL^R22`); CEQ; estoque com lotes, movimentos, lote em uso, lote de reagente em cada resultado e valor de controle, rastreio, alertas e consumo estimado; a seção "Controle" do perfil HL7 no repo `contracts`; o `fake-middleware` mandando controles; semente realista; runbook.

**Architecture:** Tudo na cidade, numa migração (`20261018900001`). O valor de controle entra por um único comando (`Lab::Qc::RecordValue`), digitado ou pela HL7: trava o aparelho, acha a referência vigente, calcula o z, roda `Lab::Qc::Westgard` (função pura) sobre a série do mesmo lote, nível, analito e aparelho e sobre os outros níveis da mesma corrida, grava o valor (só acréscimo), piora a situação da corrida e, na reprovação, abre o bloqueio (`lab_qc_blocks`) e enfileira a revisão dos itens liberados desde a última corrida aceita. `Lab::Release` (21b) passa a travar os aparelhos dos itens (FOR SHARE) antes da requisição e recusa com 409 `qc_blocked` o item cujo analito tem bloqueio ativo sem exceção. A HL7 reconhece o controle pelo `SPM-2` (código de barras de nível de lote cadastrado) antes do caminho do paciente, na mesma transação. O estoque é só de acréscimo (saldo = soma dos movimentos) e o lote em uso do par aparelho × exame é gravado em `lab_results.reagent_lot_id` e `lab_qc_values.reagent_lot_id`.

**Tech Stack:** Rails 8.1 (API), PostgreSQL 16 (banco por cidade), RSpec, `BigDecimal` (nunca `Float` em valor, média, dp ou z), `csv` (stdlib), `Net::HTTP` (fake-middleware). Nenhuma gem nova.

**Spec:** `docs/superpowers/specs/2026-10-18-module-21c-lab-quality-supplies-design.md` e `docs/adr/0038.md` (leia os dois antes). Contrato entre apps — **fonte única de formatos**: `docs/superpowers/plans/2026-10-18-module-21c-lab-qc-contracts.md` (com as bases do 21b §14 B1–B64 e do 21a §15); toda diferença está em "Divergências propostas ao contrato", no fim (prefixo **Q**). Pesquisa: `docs/pesquisa/2026-10-15-laboratorio-municipal-21b.md` (RDC 978, arts. 171–188) e `docs/pesquisa/2026-10-15-laboratorio-analisadores.md` (aparelhos, middleware, HL7). Lições que valem aqui: vírgula decimal na digitação; contadores sem trilha; ordem de travas estável; recusar em vez de cortar; todas as recusas HL7 de uma vez (B26); estrutura HL7 sempre `AR` (B50).

## Desvios e precisões

1. **Códigos de barras dos níveis numa tabela própria.** `level_barcodes` (spec §4) é guardado em `lab_qc_lot_barcodes` (`lab_qc_lot_id`, `level`, `barcode` com índice único na cidade) e montado como mapa na API. Assim a unicidade é do banco, não de uma consulta. Código de barras de controle: `^[A-Za-z0-9._-]{3,30}$` e **nunca só 10 dígitos** (o espaço `^[0-9]{10}$` é o das amostras do 21b): com isso controle e amostra nunca colidem, hoje ou no futuro.
2. **Referência guarda o id, não só o número da versão.** `lab_qc_values.lab_qc_target_id` aponta a referência usada; `target_version` sai na API pela junção.
3. **Regras como versões só de acréscimo.** `lab_qc_rules` não é editada: cada `PUT` acrescenta uma linha (regras, justificativa, quem, quando); vale a mais recente do par aparelho × analito; sem linha = todas ligadas.
4. **"Corrida aprovada" = aceita:** situação `approved` ou `warning` (1-2s é só aviso, corrida aceita, regra de Westgard). Vale para fechar bloqueio, para a "última corrida aprovada" da revisão e para contar as 20 corridas da sugestão.
5. **Série de Westgard:** valores do mesmo lote, nível, analito e aparelho em corridas **não reprovadas**, do mais antigo ao mais recente (até 9 anteriores). Valor de corrida reprovada não alimenta regra futura (senão um 2-2s se repetiria na corrida de conferência). Limites estritos: `|z| > 2` (1-2s), `> 3` (1-3s); z exatamente 2 não dispara.
6. **Situação da corrida só piora** (`open` → `approved` → `warning` → `rejected`; trigger). Depois de reprovada, o próximo valor do aparelho no dia abre corrida nova; `run_id` de corrida reprovada → 409 `invalid_transition`. Pela HL7, todos os valores de uma mensagem para o mesmo aparelho ficam na mesma corrida (foram medidos juntos).
7. **Fechamento do bloqueio:** ação corretiva registrada (pelo RT ou substituto) **e** corrida aceita, depois da abertura, com o valor mais recente de **todos** os níveis do material para o analito, todos sem reprovação e **gravados depois da ação**. Sem ação, o valor entra (201) e a resposta traz `notice: "corrective_action_required"` (Divergência Q3).
8. **Revisão desde quando:** itens do aparelho cujo exame (versão) tem o analito e cujo laudo foi liberado (`released_at`) depois da última corrida aceita com o analito no aparelho; sem corrida aceita anterior, depois da primeira referência aceita do analito no aparelho.
9. **Interruptor lido fora da transação:** `Lab::Release` lê `lab_quality_control` antes da transação (como `icp_available?`); desligado, o bloqueio não é consultado. `Lab::Hl7::Inbound` recebe `qc_allowed:` do controller. O lote em uso é lido sem interruptor (consulta simples à tabela): com `lab_inventory` desligado não há lote em uso novo, e o rastreio que já existia continua valendo.
10. **Ordem de travas (estende E2/B38 do 21b): aparelho → requisição → itens → pedido.** A avaliação do CIQ trava o aparelho FOR UPDATE; a liberação trava os aparelhos dos itens FOR SHARE (por id) **antes** da requisição; a HL7 trata os controles (aparelhos) antes das amostras de paciente. Assim a liberação e o CIQ nunca se cruzam ao contrário (prova na Task 8).
11. **Decimais.** Entrada digitada (`value`, `mean`, `sd`, `submitted`, `target`, `acceptable_*`) é string com **vírgula** (`^-?\d{1,9}(,\d{1,4})?$`); ponto → 422 `invalid_value`. HL7 usa ponto (`NM`). Saída: string com ponto, sem zeros à direita (`"98.5"`, `"100"`), para o dashboard formatar. Valor de controle **não** é dado de paciente (não é cifrado), mas nunca vai a log nem a evento.
12. **Relatório anual do CEQ** (C12 do dashboard): envio por `POST /lab/eqa/uploads` (só PDF, base64, até 10 MB, guardado em `exam_files` cifrado, como os laudos) e leitura por `GET /lab/eqa/rounds/:id/annual_report.pdf`; a rodada recebe `annual_report_upload_id`. O `POST /exams/uploads` do 21a não muda.
13. **Consumo:** um teste = uma entrada de resultado de um item (linhas não escolhidas do mesmo `entered_at`; a repetição conta) ou um valor de controle de um nível numa corrida; a linha total do item vem com `instrument_id: null` e o desvio; as linhas por aparelho trazem só `tests_performed`.
14. **Eventos a mais (só ids):** `lab_qc_block.action_recorded` e `lab_supply_lot.traced` (trilha da leitura do rastreio). Nenhum evento de plataforma novo: os interruptores usam o `city.feature_changed` que já existe (nada novo em `R18_PLATFORM_EVENT_NAMES`).

## Valores fixados por este plano

1. **Interruptores** (depois de `fake_middleware` no `CATALOG`): `lab_quality_control` e `lab_inventory`, ambos `requires: ["feature:municipal_lab"]`; `missing` `municipal_lab_disabled` (mecanismo genérico).
2. **Códigos:** corrida `open|approved|warning|rejected`; referência `package_insert|laboratory`; regras `1_2s|1_3s|2_2s|R_4s|4_1s|10x`; desfecho `ok|warning|reject`; revisão `pending|kept|rectify`; CEQ `adequate|inadequate`; item de estoque `reagent|calibrator|control|consumable`; origem do lote `purchase|donation|loan`; movimento `receipt|opening|loss|expiry|adjustment`; lote de controle `in_use|closed`; fonte do valor `typed|hl7`.
3. **Limites:** níveis por material 1..6 (nome 1..40); analitos por material 1..60; lote 1..40; código de barras de controle 3..30 (`[A-Za-z0-9._-]`, nunca só 10 dígitos); valor e média até 9 inteiros e 4 casas; dp > 0; z gravado com 3 casas (`ROUND_HALF_UP`); série até 9 anteriores; Levey-Jennings até 200 pontos (padrão: últimos 31 dias); justificativa, ação corretiva e nota 1..1000; CEQ até 60 itens, `program` 1..60, `round_code` 1..40, ação 1..500; item de estoque `name` 1..120, `unit` 1..20, `expiry_alert_days` 1..365 (padrão 30), `minimum_stock` 0..100000, `tests_per_package` 1..100000 (só reagente); quantidade de movimento 1..100000 em valor absoluto; rastreio até 5000 linhas por lista.
4. **Dev:** api do worktree na porta **3043**; Vite do dashboard **5196**; worktree `apps/api/.claude/mod21c`; branch `feat/mod-21c-lab-qc`; bancos de teste `_mod21c`; aparelhos `BIO-01` (Mindray BS-240, bioquímica) e `HEM-01` (Sysmex XN-350, hematologia) no "Laboratório Municipal Central" de Curitiba; material "Lyphochek Assayed Chemistry Control" (Bio-Rad), níveis "Nível 1" e "Nível 2", lote `CQB-2611`, códigos `CQB2611N1` e `CQB2611N2` (constantes em `lib/fake_middleware/qc_plan.rb`, fonte única da semente e do simulado).

## Global Constraints

- Banco de cidade: `db/city_migrate/20261018900001_add_lab_quality_supplies.rb`, dump à mão em `db/city_schema.rb` (`define(version: 2026_10_18_900001)`; paridade em `spec/services/city_schema_spec.rb`); triggers novos em `db/city_triggers.sql` (o dump não representa trigger). Irreversível. Sem migração de plataforma. Rollout: publicar a imagem e rodar `bin/migrate` dela (`city:migrate:all`) **antes** de cortar tráfego; **nunca** migrar cidade fora do rake.
- Erros `{ "error": "<reason>" }`; interruptor 403 `{ "error": "feature_disabled", "feature": "<key>" }`; step-up 401 `mfa_required`; Content-Type JSON em toda escrita por cookie (inclusive DELETE, corpo `{}`); números digitados com vírgula (Desvio 11).
- **Valor de resultado e de controle, CPF, CNS, nome do paciente junto com exame, PDF, mensagem HL7 e token nunca em log, evento, Analytics, mensagem de erro ou `inspect`.** Filtro novo: `/\A(justification)\z/` (`text`, `note`, `reason`, `value`/`values`, `data`, `barcode` e `number` já estão). Rastreio por lote mostra número da requisição e código do exame, nunca o nome do paciente.
- Eventos só com ids, declarados em `config/initializers/domain_events.rb` (`to: []`) e em `spec/initializers/domain_events_bindings_spec.rb`. `DomainEvents.publish(` e `Platform.audit(` são multilinha: confira cada chamada inteira. Nenhum `Platform.audit` novo (se surgir, declare em `R18_PLATFORM_EVENT_NAMES`, nunca afrouxe).
- Cifra por cidade (não determinística) em `CityEncryption::CITY_KEYED_TARGETS`: `LabQcException#justification`, `LabQcReviewItem#note` (`spec/architecture/city_encrypted_attributes_guard_spec.rb`).
- `Current.city` **nunca** é atribuído em `app/` ou `lib/` (só lido). `Platform::Features` engole `StatementInvalid`: interruptores lidos **fora** de transação.
- Specs de request com `type: :request` explícito; helpers novos em `spec/support/lab_qc_helpers.rb` (`RSpec.configure { |c| c.include LabQcHelpers }`). `spec/adr_pointers_spec.rb`: `VALID_RANGE = (1..38)`.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). `git add` com caminhos explícitos (**nunca** `-A`). Merge, push, tag (`lab-v1.1.0` inclusive) e mudança de card só com autorização explícita do usuário — exceto mover os cards F-21.25..F-21.34 para **In Progress** no início e para **Done** ao fim de cada um, que o brief manda.

## Ambiente de execução

- **Primeira ação:** mover os cards F-21.25..F-21.34 do board #1 para **In Progress** (comandos `gh project item-edit` começando com `gh`, um por vez — memória "Board: gh em comando simples"). Ao terminar a task dona de cada F-ID, mover o card para **Done**: F-21.25 → Task 4; F-21.26 → Task 9; F-21.27 → Task 7; F-21.28 → Task 11; F-21.29 → Task 8; F-21.30 → Task 9 (rota do gráfico; a tela é do dashboard); F-21.31 → Task 12; F-21.32 → Task 13; F-21.33 → Task 14; F-21.34 → Task 15.
- Antes de tocar `apps/api`, avise a sessão dona do api (sessão "API") e as sessões do dashboard e do maintenance do 21c. Ordem de merge: **contracts** (tag) → **api** → dashboard → maintenance → catálogo público.
- Worktree (raiz do monorepo `/Users/eduardovrocha/Development/ioit.solutions/rota-saude`):

  ```bash
  /opt/homebrew/bin/git -C apps/api fetch origin
  /opt/homebrew/bin/git -C apps/api worktree add .claude/mod21c -b feat/mod-21c-lab-qc origin/main
  cp apps/api/config/master.key apps/api/.claude/mod21c/config/master.key
  ```

- `./apps/api` é montado em `/rails` no container `api`; o worktree é `/rails/.claude/mod21c` (memória "Worktree do api no container": `tmp/` é volume nomeado, por isso `exec -w`). **Convenção dos steps:** `rspec <arquivos>` abrevia

  ```bash
  docker compose exec -T -e ROTA_TEST_DB_SUFFIX=_mod21c -w /rails/.claude/mod21c api bundle exec rspec <arquivos>
  ```

  `git <...>` abrevia `/opt/homebrew/bin/git -C apps/api/.claude/mod21c <...>` e `rails <...>` abrevia `docker compose exec -T -w /rails/.claude/mod21c api bin/rails <...>`.
- Bancos de teste de cidade da branch (sufixo `_mod21c`), criados no começo e recriados depois da migração de cidade (Task 3):

  ```bash
  psql -U rota_saude -d postgres -c "DROP DATABASE IF EXISTS rota_saude_test_city_a_mod21c" -c "DROP DATABASE IF EXISTS rota_saude_test_city_b_mod21c"
  docker compose exec -T -e RAILS_ENV=test -e ROTA_TEST_DB_SUFFIX=_mod21c -w /rails/.claude/mod21c api bin/rails city:test_databases
  ```

  Memória "Bancos de teste entre branches": a migração de cidade desta branch deixa os bancos `_mod21c` à frente; nunca rode a suíte de `main` neles.
- Suíte completa só com o worker parado e sem outra sessão rodando suíte: `docker compose stop worker`, a suíte, `docker compose start worker`.
- O repo `contracts` (`rota-saude/contracts`) é um clone: edite e commite ali (branch `feat/mod-21c-lab-qc`); o repo `docs` idem. O `docker-compose.yml` da raiz do monorepo **não** está em git (a Task 11 só documenta o comando; não muda o compose). No fim, os bancos `_mod21c` são apagados (memória "Apagar bancos com sufixo de módulo").

### Arquivos em comum com outros módulos (rebase)

Se a main andar, rebase sobre `origin/main` e resolva **somando** os dois lados em: `db/city_schema.rb` (`define(version:)` com o maior número; se outra branch usou `20261018900001`, renumere para o próximo livre e ajuste o `define`), `db/city_triggers.sql`, `config/initializers/{domain_events,filter_parameter_logging}.rb`, `spec/initializers/domain_events_bindings_spec.rb`, `app/services/city_encryption.rb`, `spec/adr_pointers_spec.rb` (fica `1..38`), `config/routes.rb`, `app/services/platform/features.rb`, as três specs com a lista fixa do catálogo, `app/commands/lab/{release,enter_results,choose_result,create_version}.rb`, `app/services/lab/hl7/{message,builder,inbound}.rb`, `app/controllers/lab/base_controller.rb`, `app/controllers/lab/{context_controller,lookups_controller}.rb`, `lib/fake_middleware/*`, `db/seeds.rb`.

## Review Focus

1. **Referência trocada no meio do dia** (o RT aceita a referência do laboratório entre dois valores da mesma corrida, ou um valor chega enquanto a bula é corrigida): o valor novo usa a versão vigente na gravação e a guarda (`target_version`); os valores antigos não são recalculados; o Levey-Jennings mostra cada ponto com a versão dele. Teste: Task 9 ("referência trocada no meio da corrida").
2. **Controle tratado como paciente** (o middleware manda o código de barras de controle sem `SPM-11`, ou com `SPM-11` = `P`; ou manda `SPM-11` = `Q` com o código de uma amostra de paciente): o controle vai para o CIQ e **nunca** cria `lab_results`; `Q` com código de paciente é recusado (`AE` `unknown_barcode`) e nada daquela amostra é gravado. Teste: Task 10 ("controle nunca vira paciente").
3. **Exceção velha num bloqueio novo** (o RT libera exceção para um item, o bloqueio fecha, o mesmo analito reprova de novo antes de o item ser liberado): a exceção antiga não vale para o bloqueio novo; a liberação volta a dar 409 `qc_blocked` com o id do bloqueio novo. Teste: Task 8 ("exceção não atravessa bloqueio").
4. **Duas saídas ao mesmo tempo do último kit** (duas técnicas registram abertura do mesmo lote com saldo 1): uma passa, a outra leva 422 `insufficient_balance`; o saldo nunca fica negativo (e o trigger recusa se o código falhar). Teste: Task 13 ("saídas concorrentes").
5. **Ponto e vírgula** (técnica digita `"1.234"` pensando em mil, ou `"98.5"` copiado de planilha; o middleware manda `"98.5"`): a digitação com ponto é recusada (`invalid_value`, nada gravado), `"98,5"` grava 98.5; pela HL7 `"98.5"` grava 98.5 e `"98,5"` é `AE` `invalid_value`. Teste: Task 7 ("vírgula na digitação, ponto na HL7").

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| `docs/pesquisa/2026-10-18-laboratorio-qualidade-provas-tecnicas.md` (repo docs) | registro da prova técnica da HL7 | 0 |
| `contracts/lab/{hl7-v2-profile.md,CHANGELOG.md,README.md}`, `contracts/lab/examples/{oul-r22-qc.hl7,oul-r22-qc-2-2s.hl7,ack-ae-qc.hl7,manifest.json}` | seção "Controle" do perfil e exemplos | 1 |
| `app/services/platform/features.rb`, `app/services/lab/gate.rb`, `spec/adr_pointers_spec.rb`, 3 specs da lista do catálogo, `spec/services/lab/gate_spec.rb` | interruptores | 2 |
| `db/city_migrate/20261018900001_add_lab_quality_supplies.rb`, `db/city_schema.rb`, `db/city_triggers.sql`, `app/models/lab_{instrument,qc_material,qc_lot,qc_lot_barcode,qc_target,qc_run,qc_value,qc_rule,qc_block,qc_exception,qc_review_item,eqa_round,supply,supply_lot,supply_movement,supply_in_use}.rb`, `app/models/{lab_result,lab_exam_version}.rb`, `app/services/city_encryption.rb`, `config/initializers/{domain_events,filter_parameter_logging}.rb`, `spec/initializers/domain_events_bindings_spec.rb`, `spec/support/lab_qc_helpers.rb`, `spec/models/lab_quality_schema_spec.rb` | dados da cidade | 3 |
| `app/commands/lab/{save_instrument,create_version}.rb`, `app/services/lab/{admin_json,access}.rb`, `app/services/lab/qc/{json,decimal}.rb`, `app/controllers/lab/{instruments_controller,admin/instruments_controller}.rb`, `config/routes.rb`, specs | aparelhos e vínculo (F-21.25) | 4 |
| `app/controllers/lab/qc/{base_controller,materials_controller,lots_controller,targets_controller}.rb`, `app/commands/lab/qc/{save_material,create_lot,close_lot,create_target}.rb`, specs | materiais, lotes, referência da bula | 5 |
| `app/services/lab/qc/westgard.rb`, `spec/services/lab/qc/westgard_spec.rb` | regras de Westgard | 6 |
| `app/commands/lab/qc/{record_value,save_rules,record_corrective_action}.rb`, `app/services/lab/qc/{blocks,reviews}.rb`, `app/controllers/lab/qc/{values_controller,runs_controller,rules_controller,blocks_controller}.rb`, specs | corridas, avaliação, bloqueio (F-21.27) | 7 |
| `app/services/lab/qc/release_check.rb`, `app/commands/lab/release.rb`, `app/controllers/lab/base_controller.rb`, `app/commands/lab/qc/{grant_exception,decide_review}.rb`, `app/controllers/lab/qc/{exceptions_controller,reviews_controller}.rb`, `spec/commands/lab/qc_block_release_concurrency_spec.rb`, specs | bloqueio na liberação, exceção, revisão (F-21.29) | 8 |
| `app/services/lab/qc/{suggestions,chart}.rb`, `app/commands/lab/qc/accept_suggestion.rb`, `app/controllers/lab/qc/{targets_controller,charts_controller}.rb`, specs | sugestão, aceite, Levey-Jennings (F-21.26, F-21.30) | 9 |
| `app/services/lab/qc/hl7_inbound.rb`, `app/services/lab/hl7/{message,builder,inbound}.rb`, `app/controllers/hl7/v2_controller.rb`, `spec/fixtures/lab/{oul-r22-qc,oul-r22-qc-2-2s,ack-ae-qc}.hl7`, specs | controle pela HL7 (F-21.28) | 10 |
| `lib/fake_middleware/{qc_plan,messages,runner,run}.rb`, `spec/lib/fake_middleware_qc_spec.rb` | simulado mandando controles | 11 |
| `app/commands/lab/eqa/record_round.rb`, `app/services/lab/eqa/{summary,json}.rb`, `app/controllers/lab/eqa/rounds_controller.rb`, specs | CEQ (F-21.31) | 12 |
| `app/commands/lab/inventory/{save_supply,receive_lot,record_movement}.rb`, `app/services/lab/inventory/{json,alerts}.rb`, `app/controllers/lab/inventory/{base_controller,supplies_controller,lots_controller,movements_controller,alerts_controller}.rb`, specs | estoque (F-21.32) | 13 |
| `app/commands/lab/inventory/set_in_use.rb`, `app/services/lab/inventory/trace.rb`, `app/commands/lab/{enter_results,choose_result}.rb`, `app/services/lab/entry_json.rb`, `app/controllers/lab/inventory/{in_use_controller,traces_controller}.rb`, specs | lote em uso e rastreio (F-21.33) | 14 |
| `app/services/lab/inventory/consumption.rb`, `app/controllers/lab/inventory/consumption_controller.rb`, specs | consumo estimado (F-21.34) | 15 |
| `spec/invariants/lab_quality_invariants_spec.rb` | invariantes do ADR 0038 | 16 |
| `lib/lab_qc_crew.rb`, `db/seeds.rb`, `spec/lib/lab_qc_crew_spec.rb` | semente de dev | 17 |
| `docs/operacao/rollout-qualidade-insumos-laboratorio.md` (repo docs) | revisão, suíte, porta 3043, prova, runbook | 18 |

---

## Fatia 0 — Base e prova técnica

### Task 0: Conferir a base e fazer a prova técnica da HL7 do controle

Esta task não escreve código do api. A outra prova do spec §11 (Levey-Jennings com a biblioteca de gráficos) é do plano do dashboard. **Regra:** a convenção abaixo foi pré-conferida ao escrever este plano no IHE PaLM TF Vol. 2b, rev. 11.0 (2024-04-08). (a) As fontes confirmam → siga. (b) As fontes não fecham (página fora do ar, norma sem acesso) → **adote a convenção "SPM-2 = código de barras do nível de controle"** (que este plano já usa como chave) e registre a decisão; siga. (c) As fontes **contradizem** um ponto (por exemplo, `EQU` faz parte do `OUL^R22` na 2.5.1, ou o aparelho não vai em `OBX-18`) → **PARE e reporte** (perfil, código e teste nas Tasks 1 e 10).

**Files:**
- Create: `docs/pesquisa/2026-10-18-laboratorio-qualidade-provas-tecnicas.md` (repo `docs`)

- [ ] **Step 1: Confira os nomes da base** (anote o hash de `origin/main`):

  ```bash
  /opt/homebrew/bin/git -C apps/api fetch origin
  /opt/homebrew/bin/git -C apps/api log -1 --format=%h origin/main
  /opt/homebrew/bin/git -C apps/api ls-tree --name-only origin/main db/city_migrate/ | tail -1
  /opt/homebrew/bin/git -C apps/api grep -n 'Entry.new(key: "fake_middleware"' origin/main -- app/services/platform/features.rb
  /opt/homebrew/bin/git -C apps/api grep -n "def call(requisition:, item_ids:, signature:, totp:, by:, now: Time.current, rectifies: nil" origin/main -- app/commands/lab/release.rb
  /opt/homebrew/bin/git -C apps/api grep -n "def create_row(item, analyte, normalized, ctx, source, by, receipt, now)" origin/main -- app/commands/lab/enter_results.rb
  /opt/homebrew/bin/git -C apps/api grep -n "def initialize(message, token, receipt, now, write:)" origin/main -- app/services/lab/hl7/inbound.rb
  /opt/homebrew/bin/git -C apps/api grep -n "def call(raw:, token:, city_slug:, fake_allowed:, now: Time.current)" origin/main -- app/services/lab/hl7/inbound.rb
  /opt/homebrew/bin/git -C apps/api grep -n "def result_row(row" origin/main -- app/services/lab/entry_json.rb
  /opt/homebrew/bin/git -C apps/api grep -n "def self.tick\|def tick(client:, logger:)" origin/main -- lib/fake_middleware/runner.rb
  /opt/homebrew/bin/git -C apps/api grep -n "lab = LabCrew.seed_current_city" origin/main -- db/seeds.rb
  /opt/homebrew/bin/git -C apps/api grep -n "VALID_RANGE" origin/main -- spec/adr_pointers_spec.rb
  ls docs/adr/0038.md
  ```
  Expected: cada grep acha a linha; última migração de cidade `20261015900002_add_municipal_lab.rb` (se houver maior, use o próximo número livre **acima** dela e ajuste o plano inteiro; Rails 8.1 recusa migração datada > 1 dia no futuro); `VALID_RANGE = (1..37)`; o ADR 0038 existe.

- [ ] **Step 2 (prova, HL7 v2.5.1 do controle):** confira cada linha em fonte pública e anote URL e data. Fontes: IHE PaLM TF Vol. 2b (`https://ihe.net/uploadedFiles/Documents/PaLM/IHE_PaLM_TF_Vol2b.pdf`, rev. 11.0 — transações LAW LAB-27/28/29 e o Apêndice C, segmentos `INV`, `OBX`, `SPM`; o PDF tem texto: `pdftotext` e `grep -n "Specimen Role\|INV-3\|OBX-18"`), a tabela HL7 0369 (`https://terminology.hl7.org/CodeSystem-v2-0369.html`), a tabela 0384 (`https://terminology.hl7.org/CodeSystem-v2-0384.html`), a estrutura do `OUL^R22` na 2.5.1 (norma, capítulo 7 — `https://www.hl7.org/implement/standards/product_brief.cfm?product_id=144`; ou as definições da Caristix, que exigem navegador), o capítulo 13 (segmentos `EQU`, `INV`, `SAC`) e a documentação pública de middleware (Data Innovations Instrument Manager, Siemens Atellica Data Manager — só o que estiver público). A conferir:
  1. A marca de controle na 2.5.1/LAW é **`SPM-11` (Specimen Role, CWE, tabela 0369) = `Q`** ("Control specimen"); `P` paciente, `L` pool, `U` desconhecido. Pré-conferido: PaLM TF-2b, LAB-28 e LAB-29, "SPM-11 … SHALL be coded Q in the case of a QC AWOS".
  2. Na LAW o material de controle pode vir num **`INV`** dentro do grupo do recipiente (`SAC`), com `INV-3` (Substance Type, tabela 0384) = `CO` (Control Reagent), `INV-12` validade e `INV-16` lote do fabricante. Pré-conferido: PaLM TF-2b, Apêndice C.2 ("Identify control material when QC results are transmitted").
  3. `SPM-4` (tipo do espécime) vem **vazio** (`""`) quando `SPM-11` = `Q` (PaLM TF-2b, SPM-4).
  4. O aparelho vai em **`OBX-18` (Equipment Instance Identifier, EI, repetível)**: na LAW a 1ª repetição leva modelo^fabricante e a 2ª o número de série (PaLM TF-2b, Apêndice B.2). **`EQU` não faz parte do `OUL^R22` na 2.5.1** (é segmento do capítulo 13 para status de equipamento) → Divergência Q1 (o contrato §6 cita `EQU-1`).
  5. `SPM-2` (Specimen ID, EIP) identifica o espécime de controle (a etiqueta do frasco/nível).

  **Decisão a registrar** (a que este plano implementa): o controle é reconhecido pelo **`SPM-2` = código de barras de um nível de lote de controle cadastrado** (fonte de verdade, porque middlewares reais nem sempre mandam `SPM-11`/`INV`); `SPM-11` = `Q` é recomendado e, quando vem com código que não é de controle, a amostra é recusada (`unknown_barcode`) — nunca cai no caminho do paciente; `INV` é aceito e ignorado (o lote vem do cadastro); `SPM-4` é ignorado no controle; o aparelho é a repetição de `OBX-18` cujo componente 1 casa (sem diferenciar caixa) com `lab_instruments.hl7_equipment_id`.

- [ ] **Step 3: Registre** em `docs/pesquisa/2026-10-18-laboratorio-qualidade-provas-tecnicas.md` (palavras do registro, sem copiar texto da norma):

  ```markdown
  # Módulo 21c — prova técnica da HL7 do controle (2026-10-18)

  ## Convenção HL7 v2.5.1 para resultado de controle de qualidade
  - Fontes (URL e data): <IHE PaLM TF Vol 2b rev. 11.0>, <HL7 tabela 0369>, <HL7 tabela 0384>, <norma cap. 7 e 13 ou Caristix>, <middleware público, se houver>.
  - Marca do controle: SPM-11 (tabela 0369) = Q — <confere | diferença>.
  - Material de controle: INV no grupo SAC, INV-3 = CO, INV-12 validade, INV-16 lote — <confere | diferença>.
  - SPM-4 vazio para Q — <confere | diferença>.
  - Aparelho: OBX-18 (EI, repetível; modelo^fabricante e número de série na LAW) — <confere | diferença>.
  - EQU fora do OUL^R22 na 2.5.1 → Divergência Q1 (contrato §6 cita EQU-1).
  - Decisão do Rota Saúde: controle pelo SPM-2 = código de barras do nível do lote cadastrado; SPM-11 = Q recomendado
    (Q com código que não é de controle é recusado); INV aceito e ignorado; aparelho pela repetição de OBX-18 cujo
    componente 1 casa com hl7_equipment_id.
  - Resultado: <CONFIRMADO | INCONCLUSIVO → convenção SPM-2 adotada | CONTRADIZ (PARE)>.
  ```

- [ ] **Step 4:** commite no repo docs (sem push):

  ```bash
  /opt/homebrew/bin/git -C docs add pesquisa/2026-10-18-laboratorio-qualidade-provas-tecnicas.md
  /opt/homebrew/bin/git -C docs commit -m "docs: record module 21c technical proof (HL7 quality control convention)

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

- [ ] **Step 5:** aplique a regra (a)/(b)/(c). Depois crie o worktree e os bancos de teste ("Ambiente de execução") e rode `rspec spec/adr_pointers_spec.rb spec/requests/lab/release_spec.rb spec/requests/hl7/v2_spec.rb` no worktree limpo. Expected: verde.

---

## Fatia 1 — Contrato, interruptores e dados

### Task 1: Repo `contracts` — seção "Controle" do perfil HL7 e exemplos

**Files (repo `contracts`, branch `feat/mod-21c-lab-qc`):**
- Modify: `lab/hl7-v2-profile.md` (seção "Controle", tabela de recusas), `lab/CHANGELOG.md`, `lab/README.md`, `lab/examples/manifest.json`
- Create: `lab/examples/{oul-r22-qc.hl7,oul-r22-qc-2-2s.hl7,ack-ae-qc.hl7}`

**Interfaces:**
- Produces: os exemplos `.hl7` de controle — a Task 10 copia para `apps/api/spec/fixtures/lab/` e os lê nos testes; a seção "Controle" do perfil, que a Task 10 implementa.

- [ ] **Step 1: Branch** `/opt/homebrew/bin/git -C contracts fetch origin && /opt/homebrew/bin/git -C contracts switch -c feat/mod-21c-lab-qc origin/main`.

- [ ] **Step 2: Acrescente ao `lab/hl7-v2-profile.md`**, depois da seção "### Resultado: OUL^R22 → ACK^R22" e antes de "### ACK":

  ````markdown
  ### Controle: OUL^R22 → ACK^R22 (ADR 0038, módulo 21c)
  Resultado de controle interno da qualidade (CIQ) vai na mesma mensagem `OUL^R22`,
  num `SPM` próprio. Conferido contra a HL7 v2.5.1 e o IHE LAW em 2026-10-18
  (docs/pesquisa/2026-10-18-laboratorio-qualidade-provas-tecnicas.md).
  ```
  MSH|^~\&|<app>|<instalação>|ROTA-SAUDE|<cidade>|<agora>||OUL^R22^OUL_R22|<id>|P|2.5.1
  SPM|1|<código de barras do nível>|||||||||Q
  OBR|1|||<instrument_test_code>^<nome>^99RSL
  OBX|1|NM|<analyte_key>^<nome>^99RSL||<valor>|<unidade>^^UCUM|||||F|||<AAAAMMDDhhmmss>||||<hl7_equipment_id>
  ```
  - **Quem decide que é controle é o `SPM-2`:** código de barras de um nível de lote de
    controle cadastrado no Rota Saúde. O lote, o nível e o material vêm do cadastro.
  - `SPM-11` = `Q` (tabela 0369, "Control specimen") é recomendado. `SPM-11` = `Q` com um
    código que não é de controle é recusado (`unknown_barcode`): um controle nunca vira
    resultado de paciente. `SPM-4` é ignorado no controle; `INV` é aceito e ignorado.
  - `OBR-4.1` = `instrument_test_code` da versão vigente de um exame ligado ao material.
  - `OBX-3.1` = `analyte_key` (um analito do material para aquele exame); `OBX-2` `NM`;
    `OBX-5.1` com **ponto** decimal (até 4 casas); `OBX-6.1`, quando vem, igual à unidade
    do componente.
  - **Aparelho em `OBX-18`** (EI, repetível): vale a repetição cuja 1ª parte casa, sem
    diferenciar maiúsculas, com o identificador HL7 do aparelho cadastrado. `EQU` não faz
    parte do `OUL^R22` na 2.5.1.
  - Os valores de uma mensagem para o mesmo aparelho ficam na mesma corrida. Cada valor é
    avaliado por Westgard ao entrar; a corrida reprovada **não** muda a resposta (`AA`):
    a reprovação é resultado do CIQ, não erro da mensagem.
  - Mensagem inteira ou nada, com as amostras de paciente (B26).

  Exemplo: `examples/oul-r22-qc.hl7` (aprovada) e `examples/oul-r22-qc-2-2s.hl7`
  (reprovada por 2-2s entre níveis: os dois níveis acima de +2 dp na mesma corrida;
  responde `AA`). Recusa: `examples/ack-ae-qc.hl7`.
  ````

  E, na tabela "Códigos de recusa", acrescente as linhas (antes de `invalid_message`):

  ```markdown
  | `qc_lot_expired` | AE | 207 | `SPM^n^2` |
  | `qc_lot_closed` | AE | 207 | `SPM^n^2` |
  | `qc_disabled` | AE | 207 | `SPM^n^2` |
  | `qc_no_target` | AE | 207 | `OBX^n^3` |
  | `unknown_instrument` | AE | 204 | `OBX^n^18` |
  ```

  e, logo abaixo da tabela: "`qc_disabled` = controle com o interruptor `lab_quality_control` desligado na cidade. `qc_no_target` = analito do material sem referência vigente para o lote, o nível e o aparelho."

- [ ] **Step 3: Escreva os exemplos** (segmentos separados por `\r` — gere com `printf`):

  ```bash
  cd contracts/lab/examples
  printf 'MSH|^~\\&|MW-SIM|LAB-MUNICIPAL|ROTA-SAUDE|curitiba|20261018073000||OUL^R22^OUL_R22|C0001|P|2.5.1\rSPM|1|CQB2611N1|||||||||Q\rOBR|1|||GLI^Glicose^99RSL\rOBX|1|NM|glicose^Glicose^99RSL||91.8|mg/dL^^UCUM|||||F|||20261018072500||||BIO-01\rSPM|2|CQB2611N2|||||||||Q\rOBR|2|||GLI^Glicose^99RSL\rOBX|2|NM|glicose^Glicose^99RSL||261.9|mg/dL^^UCUM|||||F|||20261018072500||||BIO-01\r' > oul-r22-qc.hl7
  printf 'MSH|^~\\&|MW-SIM|LAB-MUNICIPAL|ROTA-SAUDE|curitiba|20261018093000||OUL^R22^OUL_R22|C0002|P|2.5.1\rSPM|1|CQB2611N1|||||||||Q\rOBR|1|||GLI^Glicose^99RSL\rOBX|1|NM|glicose^Glicose^99RSL||100.8|mg/dL^^UCUM|||||F|||20261018092500||||BIO-01\rSPM|2|CQB2611N2|||||||||Q\rOBR|2|||GLI^Glicose^99RSL\rOBX|2|NM|glicose^Glicose^99RSL||305.1|mg/dL^^UCUM|||||F|||20261018092500||||BIO-01\r' > oul-r22-qc-2-2s.hl7
  printf 'MSH|^~\\&|ROTA-SAUDE|curitiba|MW-SIM|LAB-MUNICIPAL|20261018073001||ACK^R22^ACK|A0004|P|2.5.1\rMSA|AE|C0001\rERR||SPM^1^2|207^Application internal error^HL70357|E|qc_lot_expired^Lote de controle vencido^99RSL\rERR||OBX^2^18|204^Unknown key identifier^HL70357|E|unknown_instrument^Aparelho desconhecido^99RSL\r' > ack-ae-qc.hl7
  for f in oul-r22-qc.hl7 oul-r22-qc-2-2s.hl7; do tr '\r' '\n' < "$f" | awk -F'|' '$1=="SPM"{print "SPM-2="$3" SPM-11="$12} $1=="OBX"{print "OBX-5="$6" OBX-18="$19}'; done
  od -c oul-r22-qc.hl7 | grep -c '\\n'
  ```
  Expected: para cada arquivo, `SPM-2=CQB2611N1 SPM-11=Q`, `OBX-5=<valor> OBX-18=BIO-01`, `SPM-2=CQB2611N2 SPM-11=Q`, `OBX-5=<valor> OBX-18=BIO-01`; o `grep -c` imprime `0` (nenhum `\n`).

  (Os valores casam com a semente: Nível 1 média 90 dp 4,5 → 91,8 é z +0,4 e 100,8 é z +2,4; Nível 2 média 270 dp 13,5 → 261,9 é z −0,6 e 305,1 é z +2,6.)

- [ ] **Step 4: Acrescente ao `lab/examples/manifest.json`**, no fim da lista `cases` da suíte `"profile": "hl7-v2-profile.md"`:

  ```json
  {
    "file": "oul-r22-qc.hl7",
    "message": "OUL^R22^OUL_R22",
    "specimen_role": "Q",
    "answered_by": "ack-aa.hl7",
    "note": "controle aprovado: dois níveis do lote CQB-2611, aparelho em OBX-18 (ADR 0038)"
  },
  {
    "file": "oul-r22-qc-2-2s.hl7",
    "message": "OUL^R22^OUL_R22",
    "specimen_role": "Q",
    "answered_by": "ack-aa.hl7",
    "note": "controle reprovado por 2-2s entre níveis; a mensagem é aceita (AA), a corrida fica rejected"
  },
  {
    "file": "ack-ae-qc.hl7",
    "message": "ACK^R22^ACK",
    "msa": "AE",
    "err5": [
      "qc_lot_expired",
      "unknown_instrument"
    ]
  }
  ```
  Rode `python3 -m json.tool lab/examples/manifest.json > /dev/null` (de `contracts/`). Expected: sem erro.

- [ ] **Step 5: `lab/CHANGELOG.md`** — acrescente no topo (abaixo do título):

  ```markdown
  ## lab-v1.1.0 — 2026-10-18 (não publicada até a tag)

  Controle interno da qualidade pela HL7 (ADR 0038, módulo 21c). MINOR: nada do
  perfil anterior muda.

  - `hl7-v2-profile.md`: seção "Controle" — o controle é reconhecido pelo `SPM-2`
    (código de barras do nível do lote cadastrado); `SPM-11` = `Q` recomendado e
    recusado com código que não é de controle; aparelho em `OBX-18` (sem `EQU`);
    recusas `qc_lot_expired`, `qc_lot_closed`, `qc_disabled`, `qc_no_target`,
    `unknown_instrument`.
  - `examples/`: `oul-r22-qc.hl7` (aprovada), `oul-r22-qc-2-2s.hl7` (reprovada por
    2-2s entre níveis, aceita como mensagem) e `ack-ae-qc.hl7`.
  ```

  E em `lab/README.md`, na lista de arquivos, a linha: "- `hl7-v2-profile.md` também descreve o controle de qualidade (seção \"Controle\", desde `lab-v1.1.0`)."

- [ ] **Step 6: Commit** (sem push; a tag `lab-v1.1.0` **só com autorização do usuário**, depois do merge do contracts):

  ```bash
  /opt/homebrew/bin/git -C contracts add lab/hl7-v2-profile.md lab/CHANGELOG.md lab/README.md lab/examples/manifest.json lab/examples/oul-r22-qc.hl7 lab/examples/oul-r22-qc-2-2s.hl7 lab/examples/ack-ae-qc.hl7
  /opt/homebrew/bin/git -C contracts commit -m "feat: add quality control section to the lab HL7 profile

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 2: Interruptores `lab_quality_control` e `lab_inventory`

**Files:**
- Modify: `app/services/platform/features.rb`, `app/services/lab/gate.rb`, `spec/services/lab/gate_spec.rb`, `spec/adr_pointers_spec.rb`, `spec/models/city_record_settings_spec.rb:46`, `spec/requests/maintenance/city_features_spec.rb:103`, `spec/requests/operators/city_record_settings_spec.rb:30`

**Interfaces:**
- Produces: `Lab::Gate::QC_KEY = "lab_quality_control"`, `Lab::Gate::INVENTORY_KEY = "lab_inventory"`, `Lab::Gate.qc_usable?(city, env: Rails.env)`, `Lab::Gate.inventory_usable?(city, env: Rails.env)`.

- [ ] **Step 1: Teste que falha** — acrescente a `spec/services/lab/gate_spec.rb`, dentro do `describe`:

  ```ruby
  # ADR 0038: qualidade e estoque, independentes entre si, ambos sobre municipal_lab.
  it "lab_quality_control e lab_inventory dependem de municipal_lab e não um do outro" do
    set("municipal_lab", true)
    set("lab_quality_control", true)
    expect(described_class.qc_usable?(city)).to be(true)
    expect(described_class.inventory_usable?(city)).to be(false)
    set("lab_inventory", true)
    expect(described_class.inventory_usable?(city)).to be(true)
    set("lab_quality_control", false)
    expect(described_class.inventory_usable?(city)).to be(true)
    set("municipal_lab", false)
    expect(described_class.inventory_usable?(city)).to be(false)
    expect(Platform::Features.missing(city, "lab_quality_control")).to eq([ "municipal_lab_disabled" ])
    expect(Platform::Features.missing(city, "lab_inventory")).to eq([ "municipal_lab_disabled" ])
    expect(described_class.qc_usable?(nil)).to be(false)
  end
  ```

- [ ] **Step 2: Rode e veja falhar:** `rspec spec/services/lab/gate_spec.rb` → `Platform::Features::UnknownFeature` (ou `NoMethodError: qc_usable?`).

- [ ] **Step 3: Implemente.** Em `app/services/platform/features.rb`, depois da entrada `fake_middleware` (troque o `)` final dela por `),`):

  ```ruby
      # ADR 0038: controle de qualidade do laboratório municipal (módulo 21c).
      Entry.new(key: "lab_quality_control",
                description: "Controle de qualidade do laboratório: CIQ (Westgard, Levey-Jennings, bloqueio da liberação) e CEQ",
                requires: %w[feature:municipal_lab]),
      # ADR 0038: estoque de reagentes e insumos do laboratório municipal.
      Entry.new(key: "lab_inventory",
                description: "Estoque do laboratório: reagentes e insumos com lotes, lote em uso e consumo estimado",
                requires: %w[feature:municipal_lab])
  ```

  Em `app/services/lab/gate.rb`, troque o comentário de topo e acrescente as chaves e os métodos:

  ```ruby
  # Interruptores dos módulos 21b (ADR 0037) e 21c (ADR 0038): LIGADO E
  # UTILIZÁVEL, relido da plataforma a cada chamada. Leia FORA de transação
  # (Platform::Features engole StatementInvalid). fake_middleware só fora de
  # produção. lab_quality_control e lab_inventory: independentes, sobre municipal_lab.
  module Lab
    module Gate
      KEY = "municipal_lab".freeze
      HL7_KEY = "lab_hl7_inbound".freeze
      FAKE_KEY = "fake_middleware".freeze
      QC_KEY = "lab_quality_control".freeze
      INVENTORY_KEY = "lab_inventory".freeze

      module_function

      def usable?(city, env: Rails.env) = check(city, KEY, env)
      def hl7_usable?(city, env: Rails.env) = check(city, HL7_KEY, env)
      def fake_middleware_usable?(city, env: Rails.env) = check(city, FAKE_KEY, env)
      def qc_usable?(city, env: Rails.env) = check(city, QC_KEY, env)
      def inventory_usable?(city, env: Rails.env) = check(city, INVENTORY_KEY, env)
  ```
  (o resto do arquivo não muda).

  Nas três specs com a lista fixa do catálogo, acrescente `lab_quality_control lab_inventory` no fim do `%w[...]` (depois de `fake_middleware`). Em `spec/adr_pointers_spec.rb`: `VALID_RANGE = (1..38).freeze`.

- [ ] **Step 4: Rode:** `rspec spec/services/lab/gate_spec.rb spec/models/city_record_settings_spec.rb spec/requests/maintenance/city_features_spec.rb spec/requests/operators/city_record_settings_spec.rb spec/adr_pointers_spec.rb` → verde.

- [ ] **Step 5: Commit:**

  ```bash
  git add app/services/platform/features.rb app/services/lab/gate.rb spec/services/lab/gate_spec.rb spec/adr_pointers_spec.rb spec/models/city_record_settings_spec.rb spec/requests/maintenance/city_features_spec.rb spec/requests/operators/city_record_settings_spec.rb
  git commit -m "feat: add lab quality control and inventory feature switches

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 3: Migração de cidade, triggers, modelos, eventos, filtros e helpers

**Files:**
- Create: `db/city_migrate/20261018900001_add_lab_quality_supplies.rb`
- Modify: `db/city_schema.rb` (versão, `lab_exam_versions.lab_instrument_id`, `lab_results.reagent_lot_id`, as tabelas novas e FKs), `db/city_triggers.sql`
- Create: `app/models/{lab_instrument,lab_qc_material,lab_qc_lot,lab_qc_lot_barcode,lab_qc_target,lab_qc_run,lab_qc_value,lab_qc_rule,lab_qc_block,lab_qc_exception,lab_qc_review_item,lab_eqa_round,lab_supply,lab_supply_lot,lab_supply_movement,lab_supply_in_use}.rb`
- Modify: `app/models/{lab_result,lab_exam_version}.rb`, `app/services/city_encryption.rb`, `config/initializers/{domain_events,filter_parameter_logging}.rb`, `spec/initializers/domain_events_bindings_spec.rb`
- Create: `spec/support/lab_qc_helpers.rb`, `spec/models/lab_quality_schema_spec.rb`

**Interfaces:**
- Produces: as tabelas e modelos (colunas no Step 3); `LabQcRun::{STATUSES,SEVERITY,ACCEPTED}`; `LabQcRule::RULES`, `LabQcRule.enabled_for(instrument_id, analyte_key) → [String]`; `LabQcTarget.current(lot_id:, level:, analyte_key:, instrument_id:) → LabQcTarget|nil`; `LabQcMaterial#analyte_entry(key, exam_id: nil) → Hash|nil`; `LabQcLot#level_barcodes → { level => barcode }`, `#expired?(on)`; `LabQcLotBarcode::FORMAT`; `LabQcBlock.active`; `LabSupplyLot#balance → Integer`, `#expired?(on)`; `LabSupplyInUse.current`, `LabSupplyInUse.current_lot_id(instrument_id:, lab_exam_id:) → uuid|nil`; `LabResult#reagent_lot`; `LabExamVersion#lab_instrument`. Helpers `LabQcHelpers#qc_city!(inventory: false, hl7: false)`, `#inventory_city!`, `#qc_instrument!(provider, name:, hl7_equipment_id:, sector:)`, `#qc_exam!(provider:, instrument:, sigtap_code:, name:, analytes:, instrument_test_code:)`, `#qc_material!(provider, exams:, levels:, name:)`, `#qc_lot!(material, lot_number:, expires_on:, barcodes:)`, `#qc_target!(lot, level:, analyte_key:, instrument:, mean:, sd:, source:, runs:, at:, by:)`, `#qc_scenario!(provider:) → QcScenario(provider, instrument, exam, material, lot)`, `#qc_glucose(level, z) → BigDecimal`, `#qc_value!(scenario, level, z, by:, now:, analyte_key:, value:) → payload`, `#comma(number) → "98,5"`, `GLUCOSE_QC`.

- [ ] **Step 1: Teste que falha** — `spec/models/lab_quality_schema_spec.rb`:

  ```ruby
  require "rails_helper"

  # ADR 0038 (Invariantes): referências, valores, códigos de barras, regras,
  # exceções, rodadas do CEQ, lotes e movimentos de estoque nunca mudam nem
  # somem; a corrida só piora; o lote de controle só encerra; o bloqueio só
  # ganha ação corretiva e fecha uma vez, com ação; a revisão decide uma vez;
  # o lote em uso só termina; saldo nunca negativo; justificativa cifrada.
  RSpec.describe "Banco da qualidade e dos insumos do laboratório" do
    before { qc_city!(inventory: true) }

    let(:s) { qc_scenario! }
    let(:staff) { lab_staff! }

    def run!(status: "open") = LabQcRun.create!(lab_instrument: s.instrument, sector: "biochemistry", status: status, started_at: Time.current)

    def value!(run) = LabQcValue.create!(lab_qc_run: run, lab_qc_lot: s.lot, level: "Nível 1", analyte_key: "glicose", value: 101, source: "typed",
                                         z_score: BigDecimal("0.2"), lab_qc_target: LabQcTarget.first, rules_fired: [], outcome: "ok",
                                         entered_by_user: staff, entered_at: Time.current)

    it "referência, valor, código de barras e regra são só acréscimo" do
      target = LabQcTarget.first
      expect { target.update!(mean: 1) }.to raise_error(ActiveRecord::StatementInvalid, /append-only/)
      expect { target.destroy! }.to raise_error(ActiveRecord::StatementInvalid, /append-only/)
      value = value!(run!)
      expect { value.update!(value: 1) }.to raise_error(ActiveRecord::StatementInvalid, /append-only/)
      expect { s.lot.barcodes.first.update!(barcode: "OUTRO123") }.to raise_error(ActiveRecord::StatementInvalid, /append-only/)
      rule = LabQcRule.create!(lab_instrument: s.instrument, analyte_key: "glicose", rules: LabQcRule::RULES.index_with(true),
                               changed_by_user: staff, created_at: Time.current)
      expect { rule.update!(justification: "x") }.to raise_error(ActiveRecord::StatementInvalid, /append-only/)
    end

    it "corrida só piora; lote de controle só encerra" do
      run = run!
      run.update!(status: "warning", evaluated_at: Time.current)
      expect { run.update!(status: "approved") }.to raise_error(ActiveRecord::StatementInvalid, /status warning -> approved refused/)
      expect { run.reload.update!(started_at: 1.day.ago) }.to raise_error(ActiveRecord::StatementInvalid, /never changes/)
      run.reload.update!(status: "rejected")
      expect { run.destroy! }.to raise_error(ActiveRecord::StatementInvalid, /append-only/)
      expect { s.lot.update!(lot_number: "X") }.to raise_error(ActiveRecord::StatementInvalid, /lab_qc_lots/)
      s.lot.reload.update!(status: "closed", closed_on: Date.current)
      expect { s.lot.update!(status: "in_use", closed_on: nil) }.to raise_error(ActiveRecord::StatementInvalid, /lab_qc_lots/)
    end

    it "bloqueio: uma ação corretiva, fecha uma vez e só com ação; um ativo por aparelho e analito" do
      run = run!(status: "rejected")
      block = LabQcBlock.create!(lab_instrument: s.instrument, analyte_key: "glicose", opened_by_run: run, opened_at: Time.current)
      expect { LabQcBlock.create!(lab_instrument: s.instrument, analyte_key: "glicose", opened_by_run: run, opened_at: Time.current) }
        .to raise_error(ActiveRecord::RecordNotUnique)
      closer = run!
      expect { block.update!(closed_by_run: closer, closed_at: Time.current) }.to raise_error(ActiveRecord::StatementInvalid, /lab_qc_blocks/)
      block.reload.update!(corrective_action: "Calibração refeita", corrective_action_by_user: staff, corrective_action_at: Time.current)
      expect { block.update!(corrective_action: "Outra") }.to raise_error(ActiveRecord::StatementInvalid, /lab_qc_blocks/)
      block.reload.update!(closed_by_run: closer, closed_at: Time.current)
      expect { block.update!(closed_at: Time.current + 1) }.to raise_error(ActiveRecord::StatementInvalid, /lab_qc_blocks/)
      expect { block.destroy! }.to raise_error(ActiveRecord::StatementInvalid, /append-only/)
    end

    it "código de barras de controle nunca só 10 dígitos e único na cidade" do
      expect { LabQcLotBarcode.create!(lab_qc_lot: s.lot, level: "Nível 3", barcode: "0000012301", created_at: Time.current) }
        .to raise_error(ActiveRecord::StatementInvalid, /ck_lab_qc_lot_barcodes_barcode/)
      expect { LabQcLotBarcode.create!(lab_qc_lot: s.lot, level: "Nível 3", barcode: s.lot.barcodes.first.barcode, created_at: Time.current) }
        .to raise_error(ActiveRecord::RecordNotUnique)
    end

    it "estoque: lote e movimento só acréscimo; saldo nunca negativo; lote em uso só termina" do
      supply = LabSupply.create!(exam_provider: s.provider, name: "Glicose GOD-PAP", kind: "reagent", unit: "kit", tests_per_package: 500,
                                 minimum_stock: 2, expiry_alert_days: 30, lab_exam_ids: [ s.exam.id ])
      lot = LabSupplyLot.create!(lab_supply: supply, lot_number: "GL1", expires_on: Date.current + 200, supplier: "Fornecedor", origin: "purchase",
                                 created_by_user: staff, created_at: Time.current)
      LabSupplyMovement.create!(lab_supply_lot: lot, kind: "receipt", quantity: 1, recorded_by_user: staff, recorded_at: Time.current, created_at: Time.current)
      expect { LabSupplyMovement.create!(lab_supply_lot: lot, kind: "opening", quantity: -2, recorded_by_user: staff, recorded_at: Time.current, created_at: Time.current) }
        .to raise_error(ActiveRecord::StatementInvalid, /negative balance/)
      expect { LabSupplyMovement.create!(lab_supply_lot: lot, kind: "opening", quantity: 1, recorded_by_user: staff, recorded_at: Time.current, created_at: Time.current) }
        .to raise_error(ActiveRecord::StatementInvalid, /ck_lab_supply_movements_sign/)
      expect { lot.update!(supplier: "Outro") }.to raise_error(ActiveRecord::StatementInvalid, /append-only/)
      expect { LabSupplyMovement.first.destroy! }.to raise_error(ActiveRecord::StatementInvalid, /append-only/)
      use = LabSupplyInUse.create!(lab_supply_lot: lot, lab_instrument: s.instrument, lab_exam: s.exam, started_at: Time.current, started_by_user: staff)
      expect(LabSupplyInUse.current_lot_id(instrument_id: s.instrument.id, lab_exam_id: s.exam.id)).to eq(lot.id)
      expect(LabSupplyInUse.current_lot_id(instrument_id: nil, lab_exam_id: s.exam.id)).to be_nil
      use.update!(ended_at: Time.current, ended_by_user: staff)
      expect { use.update!(ended_at: nil) }.to raise_error(ActiveRecord::StatementInvalid, /lab_supply_in_use/)
      expect(LabSupplyInUse.current_lot_id(instrument_id: s.instrument.id, lab_exam_id: s.exam.id)).to be_nil
    end

    it "revisão decide uma vez; exceção cifrada e só acréscimo" do
      run = run!(status: "rejected")
      block = LabQcBlock.create!(lab_instrument: s.instrument, analyte_key: "glicose", opened_by_run: run, opened_at: Time.current)
      req = lab_requisition_row!(provider: s.provider, version: s.exam.current_version)
      item = req.items.first
      review = LabQcReviewItem.create!(lab_qc_block: block, lab_requisition_item: item)
      review.update!(decision: "kept", decided_by_user: staff, decided_at: Time.current, note: "Valor coerente")
      expect { review.update!(decision: "rectify") }.to raise_error(ActiveRecord::StatementInvalid, /lab_qc_review_items/)
      exception = LabQcException.create!(lab_qc_block: block, lab_requisition_item: item, justification: "Urgência obstétrica",
                                         granted_by_user: staff, granted_at: Time.current)
      raw = LabQcException.connection.select_value("SELECT justification FROM lab_qc_exceptions WHERE id = #{LabQcException.connection.quote(exception.id)}")
      expect(raw).not_to include("Urgência")
      expect { exception.update!(granted_at: Time.current + 1) }.to raise_error(ActiveRecord::StatementInvalid, /append-only/)
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar:** `rspec spec/models/lab_quality_schema_spec.rb` → `NoMethodError: qc_city!`.

- [ ] **Step 3: A migração** `db/city_migrate/20261018900001_add_lab_quality_supplies.rb`:

  ```ruby
  # Módulo 21c (ADR 0038): qualidade e insumos do laboratório municipal.
  # Aparelhos e vínculo exame × aparelho; CIQ (materiais, lotes com códigos
  # por nível, referências versionadas, corridas, valores, regras, bloqueios,
  # exceções, revisões); CEQ; estoque com lotes, movimentos e lote em uso; o
  # lote de reagente em cada resultado. Triggers em db/city_triggers.sql. Irreversível.
  class AddLabQualitySupplies < ActiveRecord::Migration[8.1]
    SECTORS = %w[biochemistry hematology immunology urinalysis parasitology microbiology].freeze

    def up
      create_table :lab_instruments, id: :uuid do |t|
        t.references :exam_provider, type: :uuid, null: false, foreign_key: true, index: false
        t.string :name, limit: 80, null: false
        t.string :manufacturer, limit: 80
        t.string :model, limit: 80
        t.string :serial_number, limit: 60
        t.string :sector, null: false
        t.string :hl7_equipment_id, limit: 50
        t.boolean :active, null: false, default: true
        t.timestamps
      end
      add_index :lab_instruments, "exam_provider_id, lower((name)::text)", unique: true, name: "idx_lab_instruments_name"
      add_index :lab_instruments, "exam_provider_id, upper((hl7_equipment_id)::text)", unique: true,
                where: "hl7_equipment_id IS NOT NULL", name: "idx_lab_instruments_hl7"
      add_check_constraint :lab_instruments, "sector::text = ANY (ARRAY[#{quoted(SECTORS)}]::text[])", name: "ck_lab_instruments_sector"
      add_check_constraint :lab_instruments, "hl7_equipment_id IS NULL OR hl7_equipment_id::text ~ '^[A-Za-z0-9._-]{1,50}$'::text",
                           name: "ck_lab_instruments_hl7"

      add_reference :lab_exam_versions, :lab_instrument, type: :uuid, foreign_key: true, index: true

      create_table :lab_qc_materials, id: :uuid do |t|
        t.references :exam_provider, type: :uuid, null: false, foreign_key: true, index: true
        t.string :name, limit: 80, null: false
        t.string :manufacturer, limit: 80
        t.jsonb :levels, null: false
        t.jsonb :analytes, null: false
        t.boolean :active, null: false, default: true
        t.timestamps
      end
      add_check_constraint :lab_qc_materials, "jsonb_typeof(levels) = 'array' AND jsonb_array_length(levels) BETWEEN 1 AND 6", name: "ck_lab_qc_materials_levels"
      add_check_constraint :lab_qc_materials, "jsonb_typeof(analytes) = 'array' AND jsonb_array_length(analytes) BETWEEN 1 AND 60", name: "ck_lab_qc_materials_analytes"

      create_table :lab_qc_lots, id: :uuid do |t|
        t.references :lab_qc_material, type: :uuid, null: false, foreign_key: true, index: false
        t.string :lot_number, limit: 40, null: false
        t.date :expires_on, null: false
        t.string :status, null: false, default: "in_use"
        t.date :opened_on, null: false
        t.date :closed_on
        t.references :created_by_user, type: :uuid, foreign_key: { to_table: :users }, index: false
        t.timestamps
      end
      add_index :lab_qc_lots, %i[lab_qc_material_id lot_number], unique: true, name: "idx_lab_qc_lots_number"
      add_check_constraint :lab_qc_lots, "status::text = ANY (ARRAY['in_use'::text, 'closed'::text])", name: "ck_lab_qc_lots_status"
      add_check_constraint :lab_qc_lots, "(status::text = 'closed'::text) = (closed_on IS NOT NULL)", name: "ck_lab_qc_lots_closed"

      create_table :lab_qc_lot_barcodes, id: :uuid do |t|
        t.references :lab_qc_lot, type: :uuid, null: false, foreign_key: true, index: false
        t.string :level, limit: 40, null: false
        t.string :barcode, limit: 30, null: false
        t.datetime :created_at, null: false
      end
      add_index :lab_qc_lot_barcodes, :barcode, unique: true
      add_index :lab_qc_lot_barcodes, %i[lab_qc_lot_id level], unique: true, name: "idx_lab_qc_lot_barcodes_level"
      add_check_constraint :lab_qc_lot_barcodes, "barcode::text ~ '^[A-Za-z0-9._-]{3,30}$'::text AND barcode::text !~ '^[0-9]{10}$'::text",
                           name: "ck_lab_qc_lot_barcodes_barcode"

      create_table :lab_qc_targets, id: :uuid do |t|
        t.references :lab_qc_lot, type: :uuid, null: false, foreign_key: true, index: false
        t.string :level, limit: 40, null: false
        t.string :analyte_key, limit: 40, null: false
        t.references :lab_instrument, type: :uuid, null: false, foreign_key: true, index: true
        t.integer :version, null: false
        t.string :source, null: false
        t.decimal :mean, precision: 14, scale: 4, null: false
        t.decimal :sd, precision: 14, scale: 4, null: false
        t.integer :based_on_runs
        t.text :reason
        t.references :accepted_by_user, type: :uuid, null: false, foreign_key: { to_table: :users }, index: false
        t.datetime :accepted_at, null: false
        t.datetime :created_at, null: false
      end
      add_index :lab_qc_targets, %i[lab_qc_lot_id level analyte_key lab_instrument_id version], unique: true, name: "idx_lab_qc_targets_version"
      add_check_constraint :lab_qc_targets, "source::text = ANY (ARRAY['package_insert'::text, 'laboratory'::text])", name: "ck_lab_qc_targets_source"
      add_check_constraint :lab_qc_targets, "(source::text = 'laboratory'::text) = (based_on_runs IS NOT NULL)", name: "ck_lab_qc_targets_runs"
      add_check_constraint :lab_qc_targets, "sd > 0 AND version >= 1", name: "ck_lab_qc_targets_numbers"

      create_table :lab_qc_runs, id: :uuid do |t|
        t.references :lab_instrument, type: :uuid, null: false, foreign_key: true, index: false
        t.string :sector, null: false
        t.string :status, null: false, default: "open"
        t.datetime :started_at, null: false
        t.datetime :evaluated_at
        t.references :opened_by_user, type: :uuid, foreign_key: { to_table: :users }, index: false
        t.timestamps
      end
      add_index :lab_qc_runs, %i[lab_instrument_id started_at], name: "idx_lab_qc_runs_instrument"
      add_check_constraint :lab_qc_runs, "status::text = ANY (ARRAY['open'::text, 'approved'::text, 'warning'::text, 'rejected'::text])", name: "ck_lab_qc_runs_status"
      add_check_constraint :lab_qc_runs, "status::text = 'open'::text OR evaluated_at IS NOT NULL", name: "ck_lab_qc_runs_evaluated"

      create_table :lab_supplies, id: :uuid do |t|
        t.references :exam_provider, type: :uuid, null: false, foreign_key: true, index: false
        t.string :name, limit: 120, null: false
        t.string :kind, null: false
        t.string :unit, limit: 20, null: false
        t.integer :tests_per_package
        t.integer :minimum_stock, null: false, default: 0
        t.integer :expiry_alert_days, null: false, default: 30
        t.jsonb :lab_exam_ids, null: false, default: []
        t.boolean :active, null: false, default: true
        t.timestamps
      end
      add_index :lab_supplies, "exam_provider_id, lower((name)::text)", unique: true, name: "idx_lab_supplies_name"
      add_check_constraint :lab_supplies, "kind::text = ANY (ARRAY['reagent'::text, 'calibrator'::text, 'control'::text, 'consumable'::text])", name: "ck_lab_supplies_kind"
      add_check_constraint :lab_supplies, "(kind::text = 'reagent'::text) = (tests_per_package IS NOT NULL) AND (tests_per_package IS NULL OR tests_per_package BETWEEN 1 AND 100000)",
                           name: "ck_lab_supplies_tests"
      add_check_constraint :lab_supplies, "minimum_stock BETWEEN 0 AND 100000 AND expiry_alert_days BETWEEN 1 AND 365 AND jsonb_typeof(lab_exam_ids) = 'array'",
                           name: "ck_lab_supplies_numbers"

      create_table :lab_supply_lots, id: :uuid do |t|
        t.references :lab_supply, type: :uuid, null: false, foreign_key: true, index: false
        t.string :lot_number, limit: 40, null: false
        t.date :expires_on, null: false
        t.string :supplier, limit: 120, null: false
        t.string :origin, null: false
        t.references :created_by_user, type: :uuid, null: false, foreign_key: { to_table: :users }, index: false
        t.datetime :created_at, null: false
      end
      add_index :lab_supply_lots, %i[lab_supply_id lot_number], unique: true, name: "idx_lab_supply_lots_number"
      add_check_constraint :lab_supply_lots, "origin::text = ANY (ARRAY['purchase'::text, 'donation'::text, 'loan'::text])", name: "ck_lab_supply_lots_origin"

      create_table :lab_supply_movements, id: :uuid do |t|
        t.references :lab_supply_lot, type: :uuid, null: false, foreign_key: true, index: true
        t.string :kind, null: false
        t.integer :quantity, null: false
        t.text :reason
        t.string :document, limit: 60
        t.references :recorded_by_user, type: :uuid, null: false, foreign_key: { to_table: :users }, index: false
        t.datetime :recorded_at, null: false
        t.datetime :created_at, null: false
      end
      add_index :lab_supply_movements, %i[kind recorded_at], name: "idx_lab_supply_movements_kind"
      add_check_constraint :lab_supply_movements,
                           "kind::text = ANY (ARRAY['receipt'::text, 'opening'::text, 'loss'::text, 'expiry'::text, 'adjustment'::text])",
                           name: "ck_lab_supply_movements_kind"
      add_check_constraint :lab_supply_movements,
                           "quantity <> 0 AND abs(quantity) <= 100000 AND (kind::text <> 'receipt'::text OR quantity > 0) " \
                           "AND (kind::text NOT IN ('opening', 'loss', 'expiry') OR quantity < 0)",
                           name: "ck_lab_supply_movements_sign"
      add_check_constraint :lab_supply_movements, "kind::text IN ('receipt', 'opening') OR reason IS NOT NULL", name: "ck_lab_supply_movements_reason"

      create_table :lab_supply_in_use, id: :uuid do |t|
        t.references :lab_supply_lot, type: :uuid, null: false, foreign_key: true, index: true
        t.references :lab_instrument, type: :uuid, null: false, foreign_key: true, index: false
        t.references :lab_exam, type: :uuid, null: false, foreign_key: true, index: false
        t.datetime :started_at, null: false
        t.datetime :ended_at
        t.references :started_by_user, type: :uuid, null: false, foreign_key: { to_table: :users }, index: false
        t.references :ended_by_user, type: :uuid, foreign_key: { to_table: :users }, index: false
        t.timestamps
      end
      add_index :lab_supply_in_use, %i[lab_instrument_id lab_exam_id], unique: true, where: "ended_at IS NULL", name: "idx_lab_supply_in_use_current"
      add_index :lab_supply_in_use, %i[lab_instrument_id started_at], name: "idx_lab_supply_in_use_instrument"

      add_reference :lab_results, :reagent_lot, type: :uuid, foreign_key: { to_table: :lab_supply_lots }, index: true

      create_table :lab_qc_values, id: :uuid do |t|
        t.references :lab_qc_run, type: :uuid, null: false, foreign_key: true, index: true
        t.references :lab_qc_lot, type: :uuid, null: false, foreign_key: true, index: false
        t.string :level, limit: 40, null: false
        t.string :analyte_key, limit: 40, null: false
        t.decimal :value, precision: 14, scale: 4, null: false
        t.string :source, null: false
        t.references :lab_hl7_receipt, type: :uuid, foreign_key: true, index: true
        t.decimal :z_score, precision: 8, scale: 3, null: false
        t.references :lab_qc_target, type: :uuid, null: false, foreign_key: true, index: false
        t.jsonb :rules_fired, null: false, default: []
        t.string :outcome, null: false
        t.references :reagent_lot, type: :uuid, foreign_key: { to_table: :lab_supply_lots }, index: true
        t.references :entered_by_user, type: :uuid, foreign_key: { to_table: :users }, index: false
        t.datetime :entered_at, null: false
        t.datetime :created_at, null: false
      end
      add_index :lab_qc_values, %i[lab_qc_lot_id level analyte_key entered_at], name: "idx_lab_qc_values_series"
      add_check_constraint :lab_qc_values, "source::text = ANY (ARRAY['typed'::text, 'hl7'::text])", name: "ck_lab_qc_values_source"
      add_check_constraint :lab_qc_values, "outcome::text = ANY (ARRAY['ok'::text, 'warning'::text, 'reject'::text])", name: "ck_lab_qc_values_outcome"
      add_check_constraint :lab_qc_values, "entered_by_user_id IS NOT NULL OR lab_hl7_receipt_id IS NOT NULL", name: "ck_lab_qc_values_actor"
      add_check_constraint :lab_qc_values, "source::text = 'typed'::text OR lab_hl7_receipt_id IS NOT NULL", name: "ck_lab_qc_values_hl7"
      add_check_constraint :lab_qc_values, "jsonb_typeof(rules_fired) = 'array'", name: "ck_lab_qc_values_rules"

      create_table :lab_qc_rules, id: :uuid do |t|
        t.references :lab_instrument, type: :uuid, null: false, foreign_key: true, index: false
        t.string :analyte_key, limit: 40, null: false
        t.jsonb :rules, null: false
        t.text :justification
        t.references :changed_by_user, type: :uuid, null: false, foreign_key: { to_table: :users }, index: false
        t.datetime :created_at, null: false
      end
      add_index :lab_qc_rules, %i[lab_instrument_id analyte_key created_at], name: "idx_lab_qc_rules_pair"
      add_check_constraint :lab_qc_rules, "jsonb_typeof(rules) = 'object'", name: "ck_lab_qc_rules_rules"

      create_table :lab_qc_blocks, id: :uuid do |t|
        t.references :lab_instrument, type: :uuid, null: false, foreign_key: true, index: false
        t.string :analyte_key, limit: 40, null: false
        t.references :opened_by_run, type: :uuid, null: false, foreign_key: { to_table: :lab_qc_runs }, index: false
        t.datetime :opened_at, null: false
        t.text :corrective_action
        t.references :corrective_action_by_user, type: :uuid, foreign_key: { to_table: :users }, index: false
        t.datetime :corrective_action_at
        t.references :closed_by_run, type: :uuid, foreign_key: { to_table: :lab_qc_runs }, index: false
        t.references :closed_by_user, type: :uuid, foreign_key: { to_table: :users }, index: false
        t.datetime :closed_at
        t.timestamps
      end
      add_index :lab_qc_blocks, %i[lab_instrument_id analyte_key], unique: true, where: "closed_at IS NULL", name: "idx_lab_qc_blocks_active"
      add_index :lab_qc_blocks, %i[lab_instrument_id opened_at], name: "idx_lab_qc_blocks_instrument"
      add_check_constraint :lab_qc_blocks, "(corrective_action IS NULL) = (corrective_action_at IS NULL)", name: "ck_lab_qc_blocks_action"
      add_check_constraint :lab_qc_blocks, "closed_at IS NULL OR (corrective_action IS NOT NULL AND closed_by_run_id IS NOT NULL)", name: "ck_lab_qc_blocks_closed"

      create_table :lab_qc_exceptions, id: :uuid do |t|
        t.references :lab_qc_block, type: :uuid, null: false, foreign_key: true, index: false
        t.references :lab_requisition_item, type: :uuid, null: false, foreign_key: true, index: true
        t.text :justification, null: false
        t.references :granted_by_user, type: :uuid, null: false, foreign_key: { to_table: :users }, index: false
        t.datetime :granted_at, null: false
        t.datetime :created_at, null: false
      end
      add_index :lab_qc_exceptions, %i[lab_qc_block_id lab_requisition_item_id], unique: true, name: "idx_lab_qc_exceptions_pair"

      create_table :lab_qc_review_items, id: :uuid do |t|
        t.references :lab_qc_block, type: :uuid, null: false, foreign_key: true, index: false
        t.references :lab_requisition_item, type: :uuid, null: false, foreign_key: true, index: true
        t.string :decision, null: false, default: "pending"
        t.references :decided_by_user, type: :uuid, foreign_key: { to_table: :users }, index: false
        t.datetime :decided_at
        t.text :note
        t.timestamps
      end
      add_index :lab_qc_review_items, %i[lab_qc_block_id lab_requisition_item_id], unique: true, name: "idx_lab_qc_review_items_pair"
      add_index :lab_qc_review_items, :decision, where: "decision::text = 'pending'::text", name: "idx_lab_qc_review_items_pending"
      add_check_constraint :lab_qc_review_items, "decision::text = ANY (ARRAY['pending'::text, 'kept'::text, 'rectify'::text])", name: "ck_lab_qc_review_items_decision"
      add_check_constraint :lab_qc_review_items, "(decision::text = 'pending'::text) = (decided_at IS NULL)", name: "ck_lab_qc_review_items_decided"

      create_table :lab_eqa_rounds, id: :uuid do |t|
        t.references :exam_provider, type: :uuid, null: false, foreign_key: true, index: false
        t.string :program, limit: 60, null: false
        t.string :round_code, limit: 40, null: false
        t.date :round_date, null: false
        t.jsonb :items, null: false
        t.references :annual_report_file, type: :uuid, foreign_key: { to_table: :exam_files }, index: { unique: true }
        t.references :recorded_by_user, type: :uuid, null: false, foreign_key: { to_table: :users }, index: false
        t.datetime :created_at, null: false
      end
      add_index :lab_eqa_rounds, %i[exam_provider_id program round_code], unique: true, name: "idx_lab_eqa_rounds_code"
      add_index :lab_eqa_rounds, %i[exam_provider_id round_date], name: "idx_lab_eqa_rounds_date"
      add_check_constraint :lab_eqa_rounds, "jsonb_typeof(items) = 'array' AND jsonb_array_length(items) BETWEEN 1 AND 60", name: "ck_lab_eqa_rounds_items"

      execute File.read(Rails.root.join("db/city_triggers.sql"))
    end

    def down
      raise ActiveRecord::IrreversibleMigration
    end

    private

    def quoted(list) = list.map { |value| "'#{value}'::text" }.join(", ")
  end
  ```

- [ ] **Step 4: Triggers** — acrescente ao fim de `db/city_triggers.sql`:

  ```sql
  -- Módulo 21c (ADR 0038, Invariantes): referências, valores, códigos de
  -- barras, regras, rodadas do CEQ, lotes e movimentos de estoque só crescem;
  -- exceção só cresce (justificativa cifrada aceita re-cifra); a corrida só
  -- piora; o lote de controle só encerra; o bloqueio ganha UMA ação corretiva
  -- e fecha UMA vez, com ação; a revisão decide uma vez; o lote em uso só
  -- termina; aparelho, material e item de estoque nunca somem; saldo de lote
  -- nunca negativo.
  CREATE OR REPLACE FUNCTION rota_lab_qc_run_guard() RETURNS trigger AS $fn$
  DECLARE
    moving text[] := ARRAY['status', 'evaluated_at', 'updated_at'];
    severity jsonb := '{"open": 0, "approved": 1, "warning": 2, "rejected": 3}';
  BEGIN
    IF TG_OP = 'DELETE' THEN
      RAISE EXCEPTION 'lab_qc_runs is append-only: DELETE refused';
    END IF;
    IF (to_jsonb(NEW) - moving) IS DISTINCT FROM (to_jsonb(OLD) - moving) THEN
      RAISE EXCEPTION 'lab_qc_runs: a run never changes (only its status worsens)';
    END IF;
    IF (severity ->> NEW.status::text)::int < (severity ->> OLD.status::text)::int THEN
      RAISE EXCEPTION 'lab_qc_runs: status % -> % refused', OLD.status, NEW.status;
    END IF;
    RETURN NEW;
  END;
  $fn$ LANGUAGE plpgsql;

  CREATE OR REPLACE FUNCTION rota_lab_qc_lot_guard() RETURNS trigger AS $fn$
  DECLARE
    moving text[] := ARRAY['status', 'closed_on', 'updated_at'];
  BEGIN
    IF TG_OP = 'DELETE' THEN
      RAISE EXCEPTION 'lab_qc_lots is append-only: DELETE refused';
    END IF;
    IF (to_jsonb(NEW) - moving) IS NOT DISTINCT FROM (to_jsonb(OLD) - moving)
       AND OLD.status = 'in_use' AND NEW.status = 'closed' AND NEW.closed_on IS NOT NULL THEN
      RETURN NEW;
    END IF;
    RAISE EXCEPTION 'lab_qc_lots: only in_use -> closed is allowed';
  END;
  $fn$ LANGUAGE plpgsql;

  CREATE OR REPLACE FUNCTION rota_lab_qc_block_guard() RETURNS trigger AS $fn$
  DECLARE
    action_cols text[] := ARRAY['corrective_action', 'corrective_action_by_user_id', 'corrective_action_at'];
    close_cols text[] := ARRAY['closed_by_run_id', 'closed_by_user_id', 'closed_at'];
  BEGIN
    IF TG_OP = 'DELETE' THEN
      RAISE EXCEPTION 'lab_qc_blocks is append-only: DELETE refused';
    END IF;
    IF (to_jsonb(NEW) - action_cols - close_cols - 'updated_at') IS DISTINCT FROM (to_jsonb(OLD) - action_cols - close_cols - 'updated_at')
       OR OLD.closed_at IS NOT NULL
       OR (OLD.corrective_action IS NOT NULL
           AND (NEW.corrective_action, NEW.corrective_action_by_user_id, NEW.corrective_action_at)
               IS DISTINCT FROM (OLD.corrective_action, OLD.corrective_action_by_user_id, OLD.corrective_action_at))
       OR (NEW.closed_at IS NOT NULL AND NEW.corrective_action IS NULL) THEN
      RAISE EXCEPTION 'lab_qc_blocks: one corrective action, closed once and only with it';
    END IF;
    RETURN NEW;
  END;
  $fn$ LANGUAGE plpgsql;

  CREATE OR REPLACE FUNCTION rota_lab_qc_review_guard() RETURNS trigger AS $fn$
  DECLARE
    moving text[] := ARRAY['decision', 'decided_by_user_id', 'decided_at', 'note', 'updated_at'];
  BEGIN
    IF TG_OP = 'DELETE' THEN
      RAISE EXCEPTION 'lab_qc_review_items is append-only: DELETE refused';
    END IF;
    IF current_setting('rota.reencrypting', true) = 'on' AND (to_jsonb(NEW) - 'note') = (to_jsonb(OLD) - 'note') THEN
      RETURN NEW;
    END IF;
    IF (to_jsonb(NEW) - moving) IS NOT DISTINCT FROM (to_jsonb(OLD) - moving)
       AND OLD.decision = 'pending' AND NEW.decision IN ('kept', 'rectify')
       AND NEW.decided_by_user_id IS NOT NULL AND NEW.decided_at IS NOT NULL THEN
      RETURN NEW;
    END IF;
    RAISE EXCEPTION 'lab_qc_review_items: a review is decided once';
  END;
  $fn$ LANGUAGE plpgsql;

  CREATE OR REPLACE FUNCTION rota_lab_supply_in_use_guard() RETURNS trigger AS $fn$
  DECLARE
    moving text[] := ARRAY['ended_at', 'ended_by_user_id', 'updated_at'];
  BEGIN
    IF TG_OP = 'DELETE' THEN
      RAISE EXCEPTION 'lab_supply_in_use is append-only: DELETE refused';
    END IF;
    IF (to_jsonb(NEW) - moving) IS NOT DISTINCT FROM (to_jsonb(OLD) - moving)
       AND OLD.ended_at IS NULL AND NEW.ended_at IS NOT NULL THEN
      RETURN NEW;
    END IF;
    RAISE EXCEPTION 'lab_supply_in_use: a lot in use only ends, once';
  END;
  $fn$ LANGUAGE plpgsql;

  -- Saldo do lote (soma dos movimentos) nunca negativo. A aplicação já trava o
  -- lote antes de gravar; este é o cinto de segurança.
  CREATE OR REPLACE FUNCTION rota_lab_supply_balance_guard() RETURNS trigger AS $fn$
  BEGIN
    IF (SELECT COALESCE(SUM(quantity), 0) FROM lab_supply_movements WHERE lab_supply_lot_id = NEW.lab_supply_lot_id) < 0 THEN
      RAISE EXCEPTION 'lab_supply_movements: negative balance refused';
    END IF;
    RETURN NEW;
  END;
  $fn$ LANGUAGE plpgsql;

  DO $do$
  DECLARE
    item text;
  BEGIN
    IF to_regclass('public.lab_qc_runs') IS NOT NULL THEN
      EXECUTE 'DROP TRIGGER IF EXISTS lab_qc_runs_guard ON lab_qc_runs';
      EXECUTE 'CREATE TRIGGER lab_qc_runs_guard BEFORE UPDATE OR DELETE ON lab_qc_runs FOR EACH ROW EXECUTE FUNCTION rota_lab_qc_run_guard()';
      EXECUTE 'DROP TRIGGER IF EXISTS lab_qc_lots_guard ON lab_qc_lots';
      EXECUTE 'CREATE TRIGGER lab_qc_lots_guard BEFORE UPDATE OR DELETE ON lab_qc_lots FOR EACH ROW EXECUTE FUNCTION rota_lab_qc_lot_guard()';
      EXECUTE 'DROP TRIGGER IF EXISTS lab_qc_blocks_guard ON lab_qc_blocks';
      EXECUTE 'CREATE TRIGGER lab_qc_blocks_guard BEFORE UPDATE OR DELETE ON lab_qc_blocks FOR EACH ROW EXECUTE FUNCTION rota_lab_qc_block_guard()';
      EXECUTE 'DROP TRIGGER IF EXISTS lab_qc_review_items_guard ON lab_qc_review_items';
      EXECUTE 'CREATE TRIGGER lab_qc_review_items_guard BEFORE UPDATE OR DELETE ON lab_qc_review_items FOR EACH ROW EXECUTE FUNCTION rota_lab_qc_review_guard()';
      EXECUTE 'DROP TRIGGER IF EXISTS lab_supply_in_use_guard ON lab_supply_in_use';
      EXECUTE 'CREATE TRIGGER lab_supply_in_use_guard BEFORE UPDATE OR DELETE ON lab_supply_in_use FOR EACH ROW EXECUTE FUNCTION rota_lab_supply_in_use_guard()';
      EXECUTE 'DROP TRIGGER IF EXISTS lab_supply_movements_balance ON lab_supply_movements';
      EXECUTE 'CREATE TRIGGER lab_supply_movements_balance AFTER INSERT ON lab_supply_movements FOR EACH ROW EXECUTE FUNCTION rota_lab_supply_balance_guard()';
      EXECUTE 'DROP TRIGGER IF EXISTS lab_qc_exceptions_reencryption_only ON lab_qc_exceptions';
      EXECUTE 'CREATE TRIGGER lab_qc_exceptions_reencryption_only BEFORE UPDATE OR DELETE ON lab_qc_exceptions
        FOR EACH ROW EXECUTE FUNCTION rota_reencryption_only(''justification'')';
      FOREACH item IN ARRAY ARRAY['lab_qc_lot_barcodes', 'lab_qc_targets', 'lab_qc_values', 'lab_qc_rules', 'lab_eqa_rounds',
                                  'lab_supply_lots', 'lab_supply_movements'] LOOP
        EXECUTE format('DROP TRIGGER IF EXISTS %I ON %I', item || '_append_only', item);
        EXECUTE format('CREATE TRIGGER %I BEFORE UPDATE OR DELETE ON %I FOR EACH ROW EXECUTE FUNCTION rota_append_only()',
                       item || '_append_only', item);
      END LOOP;
      FOREACH item IN ARRAY ARRAY['lab_instruments', 'lab_qc_materials', 'lab_supplies'] LOOP
        EXECUTE format('DROP TRIGGER IF EXISTS %I ON %I', item || '_no_delete', item);
        EXECUTE format('CREATE TRIGGER %I BEFORE DELETE ON %I FOR EACH ROW EXECUTE FUNCTION rota_append_only()',
                       item || '_no_delete', item);
      END LOOP;
      FOREACH item IN ARRAY ARRAY['lab_instruments', 'lab_qc_materials', 'lab_qc_lots', 'lab_qc_lot_barcodes', 'lab_qc_targets',
                                  'lab_qc_runs', 'lab_qc_values', 'lab_qc_rules', 'lab_qc_blocks', 'lab_qc_exceptions',
                                  'lab_qc_review_items', 'lab_eqa_rounds', 'lab_supplies', 'lab_supply_lots',
                                  'lab_supply_movements', 'lab_supply_in_use'] LOOP
        EXECUTE format('DROP TRIGGER IF EXISTS %I ON %I', item || '_append_only_truncate', item);
        EXECUTE format('CREATE TRIGGER %I BEFORE TRUNCATE ON %I FOR EACH STATEMENT EXECUTE FUNCTION rota_append_only()',
                       item || '_append_only_truncate', item);
      END LOOP;
    END IF;
  END
  $do$;
  ```

  Antes de seguir, confira que `rota_append_only()` levanta uma mensagem com "append-only" (`sed -n 16,24p db/city_triggers.sql`); o teste do Step 1 depende disso. A coluna nova `lab_results.reagent_lot_id` e `lab_exam_versions.lab_instrument_id` não esbarram nos triggers existentes (são `ADD COLUMN`, não `UPDATE` de linha).

- [ ] **Step 5: Modelos.** Crie cada arquivo:

  ```ruby
  # app/models/lab_instrument.rb
  # Aparelho do laboratório (ADR 0038): identificado na HL7 pelo hl7_equipment_id (OBX-18).
  class LabInstrument < ApplicationRecord
    belongs_to :exam_provider

    scope :active, -> { where(active: true) }

    def inspect = "#<LabInstrument id=#{id} name=#{name} sector=#{sector}>"
  end
  ```

  ```ruby
  # app/models/lab_qc_material.rb
  # Material de controle (ADR 0038): níveis e analitos (lab_exam_id + analyte_key).
  class LabQcMaterial < ApplicationRecord
    belongs_to :exam_provider
    has_many :lots, -> { order(:created_at) }, class_name: "LabQcLot", inverse_of: :lab_qc_material

    def analyte_entry(key, exam_id: nil)
      analytes.find { |a| a["analyte_key"] == key.to_s && (exam_id.nil? || a["lab_exam_id"] == exam_id) }
    end
  end
  ```

  ```ruby
  # app/models/lab_qc_lot.rb
  # Lote do material de controle (ADR 0038): só encerra (trigger).
  class LabQcLot < ApplicationRecord
    STATUSES = %w[in_use closed].freeze

    belongs_to :lab_qc_material
    belongs_to :created_by_user, class_name: "User", optional: true
    has_many :barcodes, -> { order(:level) }, class_name: "LabQcLotBarcode", inverse_of: :lab_qc_lot

    def level_barcodes = barcodes.to_h { |b| [ b.level, b.barcode ] }
    def expired?(on = Time.current.in_time_zone.to_date) = expires_on < on
  end
  ```

  ```ruby
  # app/models/lab_qc_lot_barcode.rb
  # Código de barras de um nível do lote (Desvio 1): único na cidade; nunca só 10 dígitos (espaço das amostras).
  class LabQcLotBarcode < ApplicationRecord
    FORMAT = /\A[A-Za-z0-9._-]{3,30}\z/
    SAMPLE_SPACE = /\A[0-9]{10}\z/

    belongs_to :lab_qc_lot

    def self.valid_code?(code) = code.is_a?(String) && code.match?(FORMAT) && !code.match?(SAMPLE_SPACE)
  end
  ```

  ```ruby
  # app/models/lab_qc_target.rb
  # Referência (média e dp) por lote, nível, analito e aparelho, versionada e imutável (ADR 0038).
  class LabQcTarget < ApplicationRecord
    SOURCES = %w[package_insert laboratory].freeze

    belongs_to :lab_qc_lot
    belongs_to :lab_instrument
    belongs_to :accepted_by_user, class_name: "User"

    def self.current(lot_id:, level:, analyte_key:, instrument_id:)
      where(lab_qc_lot_id: lot_id, level: level, analyte_key: analyte_key, lab_instrument_id: instrument_id).order(:version).last
    end
  end
  ```

  ```ruby
  # app/models/lab_qc_run.rb
  # Corrida de CIQ do aparelho (ADR 0038): a situação só piora (trigger; Desvio 6).
  class LabQcRun < ApplicationRecord
    STATUSES = %w[open approved warning rejected].freeze
    SEVERITY = STATUSES.each_with_index.to_h.freeze
    ACCEPTED = %w[approved warning].freeze

    belongs_to :lab_instrument
    belongs_to :opened_by_user, class_name: "User", optional: true
    has_many :values, -> { order(:entered_at, :created_at) }, class_name: "LabQcValue", inverse_of: :lab_qc_run
  end
  ```

  ```ruby
  # app/models/lab_qc_value.rb
  # Valor de controle (ADR 0038): só acréscimo; avaliado por Westgard na gravação. O valor nunca vai a log.
  class LabQcValue < ApplicationRecord
    OUTCOMES = %w[ok warning reject].freeze

    belongs_to :lab_qc_run
    belongs_to :lab_qc_lot
    belongs_to :lab_qc_target
    belongs_to :lab_hl7_receipt, optional: true
    belongs_to :reagent_lot, class_name: "LabSupplyLot", optional: true
    belongs_to :entered_by_user, class_name: "User", optional: true

    def inspect = "#<LabQcValue id=#{id} analyte_key=#{analyte_key} level=#{level} outcome=#{outcome}>"
  end
  ```

  ```ruby
  # app/models/lab_qc_rule.rb
  # Regras de Westgard ligadas por aparelho e analito (Desvio 3): versões só de
  # acréscimo; vale a mais recente; sem linha = todas ligadas.
  class LabQcRule < ApplicationRecord
    RULES = %w[1_2s 1_3s 2_2s R_4s 4_1s 10x].freeze

    belongs_to :lab_instrument
    belongs_to :changed_by_user, class_name: "User"

    def self.current_for(instrument_id, analyte_key)
      where(lab_instrument_id: instrument_id, analyte_key: analyte_key).order(:created_at, :id).last
    end

    def self.enabled_for(instrument_id, analyte_key)
      row = current_for(instrument_id, analyte_key)
      row ? RULES.select { |rule| row.rules[rule] == true } : RULES
    end
  end
  ```

  ```ruby
  # app/models/lab_qc_block.rb
  # Bloqueio da liberação por aparelho e analito (ADR 0038): um ativo por par;
  # uma ação corretiva; fecha uma vez, com ação (trigger).
  class LabQcBlock < ApplicationRecord
    belongs_to :lab_instrument
    belongs_to :opened_by_run, class_name: "LabQcRun"
    belongs_to :closed_by_run, class_name: "LabQcRun", optional: true
    belongs_to :closed_by_user, class_name: "User", optional: true
    belongs_to :corrective_action_by_user, class_name: "User", optional: true
    has_many :exceptions, class_name: "LabQcException", inverse_of: :lab_qc_block
    has_many :review_items, class_name: "LabQcReviewItem", inverse_of: :lab_qc_block

    scope :active, -> { where(closed_at: nil) }
  end
  ```

  ```ruby
  # app/models/lab_qc_exception.rb
  # Exceção do RT ou substituto para liberar um item bloqueado (ADR 0038): só acréscimo; justificativa cifrada.
  class LabQcException < ApplicationRecord
    encrypts :justification

    belongs_to :lab_qc_block
    belongs_to :lab_requisition_item
    belongs_to :granted_by_user, class_name: "User"

    def inspect = "#<LabQcException id=#{id} lab_qc_block_id=#{lab_qc_block_id} lab_requisition_item_id=#{lab_requisition_item_id}>"
  end
  ```

  ```ruby
  # app/models/lab_qc_review_item.rb
  # Item liberado desde a última corrida aceita, a revisar pelo RT (ADR 0038): decide uma vez; nota cifrada.
  class LabQcReviewItem < ApplicationRecord
    DECISIONS = %w[pending kept rectify].freeze

    encrypts :note

    belongs_to :lab_qc_block
    belongs_to :lab_requisition_item
    belongs_to :decided_by_user, class_name: "User", optional: true

    def inspect = "#<LabQcReviewItem id=#{id} decision=#{decision}>"
  end
  ```

  ```ruby
  # app/models/lab_eqa_round.rb
  # Rodada do controle externo (CEQ; ADR 0038): só acréscimo; relatório anual em exam_files (cifrado).
  class LabEqaRound < ApplicationRecord
    ADEQUACY = %w[adequate inadequate].freeze

    belongs_to :exam_provider
    belongs_to :recorded_by_user, class_name: "User"
    belongs_to :annual_report_file, class_name: "ExamFile", optional: true
  end
  ```

  ```ruby
  # app/models/lab_supply.rb
  # Item de estoque do laboratório (ADR 0038): reagente tem rendimento em testes.
  class LabSupply < ApplicationRecord
    KINDS = %w[reagent calibrator control consumable].freeze

    belongs_to :exam_provider
    has_many :lots, -> { order(:expires_on, :lot_number) }, class_name: "LabSupplyLot", inverse_of: :lab_supply
  end
  ```

  ```ruby
  # app/models/lab_supply_lot.rb
  # Lote de um item (ADR 0038): só acréscimo; saldo = soma dos movimentos (nunca negativo).
  class LabSupplyLot < ApplicationRecord
    ORIGINS = %w[purchase donation loan].freeze

    belongs_to :lab_supply
    belongs_to :created_by_user, class_name: "User"
    has_many :movements, -> { order(:recorded_at, :created_at) }, class_name: "LabSupplyMovement", inverse_of: :lab_supply_lot

    def balance = movements.sum(:quantity)
    def expired?(on = Time.current.in_time_zone.to_date) = expires_on < on
  end
  ```

  ```ruby
  # app/models/lab_supply_movement.rb
  # Movimento de estoque (ADR 0038): só acréscimo; quantidade com sinal (entrada +, saída −).
  class LabSupplyMovement < ApplicationRecord
    KINDS = %w[receipt opening loss expiry adjustment].freeze

    belongs_to :lab_supply_lot
    belongs_to :recorded_by_user, class_name: "User"
  end
  ```

  ```ruby
  # app/models/lab_supply_in_use.rb
  # Lote de reagente em uso no par aparelho × exame (ADR 0038): um corrente por par; só termina.
  class LabSupplyInUse < ApplicationRecord
    self.table_name = "lab_supply_in_use"

    belongs_to :lab_supply_lot
    belongs_to :lab_instrument
    belongs_to :lab_exam
    belongs_to :started_by_user, class_name: "User"
    belongs_to :ended_by_user, class_name: "User", optional: true

    scope :current, -> { where(ended_at: nil) }

    # Lido sem interruptor (Desvio 9): nil quando o exame não tem aparelho ou o par não tem lote em uso.
    def self.current_lot_id(instrument_id:, lab_exam_id:)
      return nil if instrument_id.nil? || lab_exam_id.nil?

      current.where(lab_instrument_id: instrument_id, lab_exam_id: lab_exam_id).pick(:lab_supply_lot_id)
    end
  end
  ```

  Em `app/models/lab_result.rb`, depois de `belongs_to :chosen_from ...`: `belongs_to :reagent_lot, class_name: "LabSupplyLot", optional: true`. Em `app/models/lab_exam_version.rb`, depois de `belongs_to :created_by_user ...`: `belongs_to :lab_instrument, optional: true`.

- [ ] **Step 6: Cifra, eventos, filtro e helpers.**

  `app/services/city_encryption.rb` — no fim de `CITY_KEYED_TARGETS`, depois de `[ LabCriticalNotice, :notified_person ]` (com vírgula):

  ```ruby
      [ LabQcException, :justification ],
      [ LabQcReviewItem, :note ]
  ```

  `config/initializers/domain_events.rb` — depois do bloco do 21b:

  ```ruby
    # Módulo 21c (ADR 0038): qualidade e insumos, só trilha, só ids. Nunca
    # valor de controle, valor de resultado, justificativa ou nome.
    %w[lab_qc_run.evaluated lab_qc_block.opened lab_qc_block.closed lab_qc_block.action_recorded lab_qc_exception.granted
       lab_qc_review.decided lab_qc_target.accepted lab_qc_rule.changed lab_eqa_round.recorded lab_supply_lot.received
       lab_supply_lot.traced lab_supply_movement.recorded lab_supply_in_use.changed].each { |name| DomainEvents.bind name, to: [] }
  ```
  e a mesma lista no `spec/initializers/domain_events_bindings_spec.rb` (um bloco `names = %w[...]` como o do 21b, com o mesmo `expect` que o bloco do 21b usa).

  `config/initializers/filter_parameter_logging.rb` — no fim da lista:

  ```ruby
    # ADR 0038: a justificativa da exceção do CIQ fala do paciente (urgência, quadro).
    # (`:text` — ação corretiva —, `:note`, `:reason`, `values` e `barcode` já estão.)
    /\A(justification)\z/
  ```
  (troque o `]` do item anterior por `,` conforme o arquivo).

  `spec/support/lab_qc_helpers.rb`:

  ```ruby
  # Módulo 21c (ADR 0038): cidade com lab_quality_control (e lab_inventory),
  # aparelho, exame ligado ao aparelho, material de dois níveis, lote e a
  # referência da bula da glicose (Nível 1: 100 ± 5; Nível 2: 250 ± 10).
  module LabQcHelpers
    GLUCOSE_QC = { "Nível 1" => %w[100 5], "Nível 2" => %w[250 10] }.freeze
    QcScenario = Struct.new(:provider, :instrument, :exam, :material, :lot, keyword_init: true)

    def qc_city!(inventory: false, hl7: false)
      city = lab_city!(hl7: hl7)
      maintainer = ledi_maintainer!
      Platform::Features.set!(city: city, key: "lab_quality_control", enabled: true, maintainer: maintainer)
      Platform::Features.set!(city: city, key: "lab_inventory", enabled: true, maintainer: maintainer) if inventory
      CityCatalog.reset_cache!
      city
    end

    def inventory_city!
      city = lab_city!
      Platform::Features.set!(city: city, key: "lab_inventory", enabled: true, maintainer: ledi_maintainer!)
      CityCatalog.reset_cache!
      city
    end

    def qc_instrument!(provider, name: "Analisador bioquímico 1", hl7_equipment_id: "BIO-01", sector: "biochemistry")
      LabInstrument.create!(exam_provider: provider, name: name, manufacturer: "Mindray", model: "BS-240",
                            serial_number: "BS24-#{SecureRandom.hex(3)}", sector: sector, hl7_equipment_id: hl7_equipment_id, active: true)
    end

    # Exame cuja versão vigente está ligada ao aparelho.
    def qc_exam!(provider:, instrument:, sigtap_code: "0202010473", name: "Glicose", analytes: LabHelpers::GLUCOSE, instrument_test_code: "GLI")
      exam = LabExam.create!(exam_provider: provider, sigtap_code: sigtap_code, name: name, sector: instrument.sector, delivery_days: 2)
      version = LabExamVersion.create!(lab_exam: exam, version: 1, analysis_method: "Enzimático", material: "Plasma", container: "Tubo cinza (fluoreto)",
                                       instrument_test_code: instrument_test_code, analytes: analytes, created_by_user: lab_admin!,
                                       lab_instrument: instrument)
      exam.update!(current_version: version)
      exam
    end

    def qc_material!(provider, exams:, levels: [ "Nível 1", "Nível 2" ], name: "Controle bioquímico")
      analytes = exams.flat_map { |exam| exam.current_version.entered_analytes.map { |a| { "lab_exam_id" => exam.id, "analyte_key" => a["key"] } } }
      LabQcMaterial.create!(exam_provider: provider, name: name, manufacturer: "Bio-Rad", levels: levels, analytes: analytes, active: true)
    end

    def qc_lot!(material, lot_number: "L#{SecureRandom.hex(2).upcase}", expires_on: Time.zone.today + 300, barcodes: nil)
      lot = LabQcLot.create!(lab_qc_material: material, lot_number: lot_number, expires_on: expires_on, status: "in_use", opened_on: Time.zone.today)
      codes = barcodes || material.levels.each_with_index.to_h { |level, i| [ level, "CQ#{SecureRandom.hex(3).upcase}N#{i + 1}" ] }
      codes.each { |level, code| LabQcLotBarcode.create!(lab_qc_lot: lot, level: level, barcode: code, created_at: Time.current) }
      lot
    end

    def qc_target!(lot, level:, analyte_key:, instrument:, mean:, sd:, source: "package_insert", runs: nil, at: Time.current - 30.days, by: lab_admin!)
      version = LabQcTarget.where(lab_qc_lot_id: lot.id, level: level, analyte_key: analyte_key, lab_instrument_id: instrument.id).maximum(:version).to_i + 1
      LabQcTarget.create!(lab_qc_lot: lot, level: level, analyte_key: analyte_key, lab_instrument: instrument, version: version, source: source,
                          mean: BigDecimal(mean.to_s), sd: BigDecimal(sd.to_s), based_on_runs: runs, accepted_by_user: by, accepted_at: at, created_at: at)
    end

    def qc_scenario!(provider: lab_provider!)
      instrument = qc_instrument!(provider)
      exam = qc_exam!(provider: provider, instrument: instrument)
      material = qc_material!(provider, exams: [ exam ])
      lot = qc_lot!(material)
      GLUCOSE_QC.each { |level, (mean, sd)| qc_target!(lot, level: level, analyte_key: "glicose", instrument: instrument, mean: mean, sd: sd) }
      QcScenario.new(provider: provider, instrument: instrument, exam: exam, material: material, lot: lot)
    end

    # Valor da glicose do nível para um z da bula do cenário.
    def qc_glucose(level, z)
      mean, sd = GLUCOSE_QC.fetch(level)
      BigDecimal(mean) + (BigDecimal(sd) * BigDecimal(z.to_s))
    end

    def qc_value!(scenario, level, z = 0, by: nil, now: Time.current, analyte_key: "glicose", value: nil)
      result = Lab::Qc::RecordValue.call(instrument: scenario.instrument, lot: scenario.lot, level: level, analyte_key: analyte_key,
                                         value: value || qc_glucose(level, z), source: "typed", by: by || (@qc_staff ||= lab_staff!), now: now)
      raise "valor de controle recusado: #{result.reason}" if result.failure?

      result.payload
    end

    def comma(number) = BigDecimal(number.to_s).to_s("F").sub(/\.0\z/, "").tr(".", ",")
  end

  RSpec.configure { |c| c.include LabQcHelpers }
  ```

- [ ] **Step 7: Rode a migração e escreva o dump.** Recrie os bancos de teste `_mod21c` ("Ambiente") e migre o de dev do worktree **só pelo rake**: `rails city:migrate:all`. **Escreva à mão** `db/city_schema.rb`: `define(version: 2026_10_18_900001)`; `t.uuid "lab_instrument_id"` + índice em `lab_exam_versions`; `t.uuid "reagent_lot_id"` + índice em `lab_results`; os blocos `create_table` novos em ordem alfabética (`lab_eqa_rounds`, `lab_instruments`, `lab_qc_blocks`, `lab_qc_exceptions`, `lab_qc_lot_barcodes`, `lab_qc_lots`, `lab_qc_materials`, `lab_qc_review_items`, `lab_qc_rules`, `lab_qc_runs`, `lab_qc_targets`, `lab_qc_values`, `lab_supplies`, `lab_supply_in_use`, `lab_supply_lots`, `lab_supply_movements`) com colunas em ordem alfabética, índices e checks como o `pg_dump` os devolve — compare com

  ```bash
  docker compose exec -T -e RAILS_ENV=test -e ROTA_TEST_DB_SUFFIX=_mod21c -w /rails/.claude/mod21c api bin/rails runner 'CityConnection.with(TEST_CITY_A) { puts ActiveRecord::SchemaDumper.dump }' > /tmp/city_dump_21c.rb
  ```
  (se `TEST_CITY_A` não existir fora do rspec, use o `CitySchema.db_config_for` do banco `rota_saude_test_city_a_mod21c`) — e as `add_foreign_key` novas em ordem.

- [ ] **Step 8: Rode:** `rspec spec/models/lab_quality_schema_spec.rb spec/services/city_schema_spec.rb spec/architecture spec/initializers/domain_events_bindings_spec.rb spec/models/municipal_lab_schema_spec.rb` → verde.

- [ ] **Step 9: Commit:**

  ```bash
  git add db/city_migrate/20261018900001_add_lab_quality_supplies.rb db/city_schema.rb db/city_triggers.sql app/models/lab_instrument.rb app/models/lab_qc_material.rb app/models/lab_qc_lot.rb app/models/lab_qc_lot_barcode.rb app/models/lab_qc_target.rb app/models/lab_qc_run.rb app/models/lab_qc_value.rb app/models/lab_qc_rule.rb app/models/lab_qc_block.rb app/models/lab_qc_exception.rb app/models/lab_qc_review_item.rb app/models/lab_eqa_round.rb app/models/lab_supply.rb app/models/lab_supply_lot.rb app/models/lab_supply_movement.rb app/models/lab_supply_in_use.rb app/models/lab_result.rb app/models/lab_exam_version.rb app/services/city_encryption.rb config/initializers/domain_events.rb config/initializers/filter_parameter_logging.rb spec/initializers/domain_events_bindings_spec.rb spec/support/lab_qc_helpers.rb spec/models/lab_quality_schema_spec.rb
  git commit -m "feat: add lab quality and supplies city tables with immutability triggers

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

---

## Fatia 2 — Aparelhos e cadastro do CIQ

### Task 4: Aparelhos e vínculo exame × aparelho (F-21.25)

**Files:**
- Create: `app/commands/lab/save_instrument.rb`, `app/controllers/lab/instruments_controller.rb`, `app/controllers/lab/admin/instruments_controller.rb`, `app/services/lab/qc/decimal.rb`, `spec/requests/lab/instruments_spec.rb`, `spec/services/lab/qc/decimal_spec.rb`
- Modify: `app/commands/lab/create_version.rb`, `app/services/lab/admin_json.rb`, `app/services/lab/access.rb`, `app/services/lab/gate.rb`, `app/controllers/lab/{context_controller,lookups_controller}.rb`, `config/routes.rb`, `spec/requests/lab/context_spec.rb`

**Interfaces:**
- Consumes: `LabInstrument` (Task 3); `Lab::Admin::BaseController` (21b: `municipal_admin`, step-up nas escritas, `admin_provider!`); `Lab::Access.{labs,member?,admin?}`.
- Produces: `Lab::SaveInstrument.call(params, instrument: nil) → Result(instrument:)`; `Lab::AdminJson.instrument(i) → { id, provider_id, name, manufacturer, model, serial_number, sector, hl7_equipment_id, active }`; `Lab::AdminJson.version` ganha `lab_instrument_id`; `Lab::Access.technical?(user, provider_id) → Boolean` (RT ou substituto com ligação ativa); `Lab::Qc::Decimal.typed(raw) → BigDecimal|nil` (vírgula), `.hl7(raw) → BigDecimal|nil` (ponto), `.out(number) → String|nil`. Rotas `GET /lab/instruments?provider_id=`, `GET|POST /lab/admin/instruments`, `PATCH /lab/admin/instruments/:id`; `POST /lab/admin/exams/:id/versions` aceita `lab_instrument_id`. Formas do dashboard adotadas aqui: **C1** (`GET /lab/context` ganha `technical_lead_of: [provider_id]`), **C5** (`GET /lab/exams?provider_id=` ganha `id` e `analytes: [{ key, name }]` da versão vigente — o portão `provider_param!` do 21b já deixa liberador e admin), **C13** (aparelhos respondem com `lab_quality_control` **ou** `lab_inventory` utilizável; os dois desligados → 403 `{ "error": "feature_disabled", "feature": "lab_quality_control" }`): `Lab::Gate.instruments_usable?(city) → Boolean`.

- [ ] **Step 1: Testes que falham.** `spec/services/lab/qc/decimal_spec.rb`:

  ```ruby
  require "rails_helper"

  # Desvio 11: digitação com vírgula (ponto recusado); HL7 com ponto; saída com ponto, sem zeros à direita.
  RSpec.describe Lab::Qc::Decimal do
    it "digitado: só vírgula, até 9 inteiros e 4 casas" do
      expect(described_class.typed("98,5")).to eq(BigDecimal("98.5"))
      expect(described_class.typed(" -0,0125 ")).to eq(BigDecimal("-0.0125"))
      expect(described_class.typed("100")).to eq(BigDecimal("100"))
      [ "98.5", "1.234", "1,23456", "1234567890", "", "abc", nil, 98.5 ].each { |raw| expect(described_class.typed(raw)).to be_nil }
    end

    it "HL7: só ponto" do
      expect(described_class.hl7("98.5")).to eq(BigDecimal("98.5"))
      [ "98,5", "1e3", "" ].each { |raw| expect(described_class.hl7(raw)).to be_nil }
    end

    it "saída" do
      expect(described_class.out(BigDecimal("98.5000"))).to eq("98.5")
      expect(described_class.out(BigDecimal("100.0"))).to eq("100")
      expect(described_class.out(nil)).to be_nil
    end
  end
  ```

  `spec/requests/lab/instruments_spec.rb`:

  ```ruby
  require "rails_helper"

  # Contrato §2 (F-21.25): aparelhos do laboratório (admin com step-up),
  # leitura para os membros, identificador HL7 único no laboratório (sem
  # diferenciar caixa) e o vínculo exame × aparelho na versão do exame.
  RSpec.describe "Laboratório — aparelhos", type: :request do
    before { qc_city! }

    let(:provider) { lab_provider! }
    let(:admin) { lab_admin! }
    let(:payload) do
      { provider_id: provider.id, name: "Analisador bioquímico 1", manufacturer: "Mindray", model: "BS-240", serial_number: "BS24-0001",
        sector: "biochemistry", hl7_equipment_id: "BIO-01", active: true }
    end

    it "admin cadastra, edita e desativa; recusas de nome, setor, identificador e duplicado" do
      stepped_up!(admin)
      json_post "/lab/admin/instruments", payload
      expect(response).to have_http_status(:created)
      created = json_body
      expect(created).to include("name" => "Analisador bioquímico 1", "sector" => "biochemistry", "hl7_equipment_id" => "BIO-01", "active" => true)
      json_post "/lab/admin/instruments", payload.merge(name: "Outro", hl7_equipment_id: "bio-01")
      expect([ response.status, json_body["error"] ]).to eq([ 422, "duplicate_hl7_equipment_id" ])
      json_post "/lab/admin/instruments", payload.merge(hl7_equipment_id: "BIO-02")
      expect(json_body["error"]).to eq("duplicate_name")
      json_post "/lab/admin/instruments", payload.merge(name: "", hl7_equipment_id: nil)
      expect(json_body["error"]).to eq("invalid_name")
      json_post "/lab/admin/instruments", payload.merge(name: "X", sector: "radiology", hl7_equipment_id: nil)
      expect(json_body["error"]).to eq("invalid_sector")
      json_post "/lab/admin/instruments", payload.merge(name: "X", hl7_equipment_id: "BIO 01")
      expect(json_body["error"]).to eq("invalid_instrument")
      json_patch "/lab/admin/instruments/#{created['id']}", payload.except(:provider_id).merge(active: false, hl7_equipment_id: nil)
      expect(json_body).to include("active" => false, "hl7_equipment_id" => nil)
      get "/lab/admin/instruments", params: { provider_id: provider.id }
      expect(json_body["instruments"].map { |i| i["name"] }).to eq([ "Analisador bioquímico 1" ])
    end

    it "sem step-up 401; sem papel 403; membro lê; quem não é do laboratório não lê" do
      sign_in_as(admin)
      json_post "/lab/admin/instruments", payload
      expect([ response.status, json_body["error"] ]).to eq([ 401, "mfa_required" ])
      qc_instrument!(provider)
      staff = lab_staff!
      sign_in_as(staff)
      json_post "/lab/admin/instruments", payload
      expect(response).to have_http_status(:forbidden)
      get "/lab/instruments", params: { provider_id: provider.id }
      expect(json_body["instruments"].sole).to include("hl7_equipment_id" => "BIO-01", "provider_id" => provider.id)
      sign_in_as(staff_with("recepcao-#{SecureRandom.hex(2)}@cidade.gov.br", "citizen_verifier"))
      get "/lab/instruments", params: { provider_id: provider.id }
      expect(response).to have_http_status(:forbidden)
    end

    it "C13: com os dois interruptores do 21c desligados, aparelhos 403; com só o estoque, respondem" do
      city = City.find(Current.city.id)
      Platform::Features.set!(city: city, key: "lab_quality_control", enabled: false, maintainer: ledi_maintainer!)
      sign_in_as(lab_staff!)
      get "/lab/instruments", params: { provider_id: provider.id }
      expect(json_body).to eq("error" => "feature_disabled", "feature" => "lab_quality_control")
      Platform::Features.set!(city: city, key: "lab_inventory", enabled: true, maintainer: ledi_maintainer!)
      get "/lab/instruments", params: { provider_id: provider.id }
      expect(response).to have_http_status(:ok)
    end

    it "C1 e C5: contexto diz de quais laboratórios a sessão é RT/substituto; /lab/exams traz id e componentes" do
      lead = lab_releaser!(provider, lead: true)
      exam = lab_exam!(provider: provider)
      sign_in_as(lead)
      get "/lab/context"
      expect(json_body["technical_lead_of"]).to eq([ provider.id ])
      get "/lab/exams", params: { provider_id: provider.id }
      expect(json_body["exams"].sole).to include("id" => exam.id, "analytes" => [ { "key" => "glicose", "name" => "Glicose" } ])
      sign_in_as(lab_staff!)
      get "/lab/context"
      expect(json_body["technical_lead_of"]).to eq([])
    end

    it "versão do exame ligada a um aparelho do laboratório; aparelho alheio ou inativo recusado" do
      instrument = qc_instrument!(provider)
      other = qc_instrument!(lab_provider!, hl7_equipment_id: "BIO-09")
      exam = lab_exam!(provider: provider)
      base = { method: "Enzimático", material: "Plasma", container: "Tubo cinza", instrument_test_code: "GLI", analytes: LabHelpers::GLUCOSE }
      stepped_up!(admin)
      json_post "/lab/admin/exams/#{exam.id}/versions", base.merge(lab_instrument_id: instrument.id)
      expect(response).to have_http_status(:created)
      expect(json_body).to include("version" => 2, "lab_instrument_id" => instrument.id)
      json_post "/lab/admin/exams/#{exam.id}/versions", base.merge(lab_instrument_id: other.id)
      expect(json_body["error"]).to eq("invalid_version")
      instrument.update!(active: false)
      json_post "/lab/admin/exams/#{exam.id}/versions", base.merge(lab_instrument_id: instrument.id)
      expect(json_body["error"]).to eq("invalid_version")
      json_post "/lab/admin/exams/#{exam.id}/versions", base.merge(lab_instrument_id: nil)
      expect(json_body["lab_instrument_id"]).to be_nil
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar:** `rspec spec/services/lab/qc/decimal_spec.rb spec/requests/lab/instruments_spec.rb` → `NameError: Lab::Qc::Decimal` e 404 nas rotas.

- [ ] **Step 3: Implemente.** `app/services/lab/qc/decimal.rb`:

  ```ruby
  # Números do CIQ, do CEQ e do estoque (Desvio 11): digitação com vírgula
  # (ponto recusado — "1.234" é ambíguo); HL7 com ponto (NM); saída com ponto
  # e sem zeros à direita. Sempre BigDecimal, nunca Float.
  module Lab
    module Qc
      module Decimal
        TYPED = /\A-?\d{1,9}(,\d{1,4})?\z/
        HL7 = /\A-?\d{1,9}(\.\d{1,4})?\z/

        module_function

        def typed(raw) = raw.is_a?(String) && raw.strip.match?(TYPED) ? BigDecimal(raw.strip.tr(",", ".")) : nil
        def hl7(raw) = raw.is_a?(String) && raw.strip.match?(HL7) ? BigDecimal(raw.strip) : nil

        def out(number)
          return nil if number.nil?

          BigDecimal(number.to_s).to_s("F").sub(/\.0\z/, "")
        end
      end
    end
  end
  ```

  `app/commands/lab/save_instrument.rb`:

  ```ruby
  # Aparelho do laboratório (contrato §2; F-21.25): nome único no laboratório;
  # identificador HL7 (OBX-18) único no laboratório sem diferenciar caixa.
  # Nunca apagado (desativar). Trava do prestador contra duplicado concorrente.
  module Lab
    module SaveInstrument
      LIMITS = { "manufacturer" => 80, "model" => 80, "serial_number" => 60 }.freeze
      HL7_ID = /\A[A-Za-z0-9._-]{1,50}\z/

      module_function

      def call(params, instrument: nil)
        params = params.to_h.stringify_keys
        provider = instrument&.exam_provider || provider_for(params["provider_id"])
        return Result.fail(:invalid_provider) unless provider

        name = params["name"].is_a?(String) ? params["name"].strip : ""
        return Result.fail(:invalid_name) unless name.length.between?(1, 80)
        return Result.fail(:invalid_sector) unless LabExam::SECTORS.include?(params["sector"])
        return Result.fail(:invalid_instrument) unless (LIMITS.keys + [ "hl7_equipment_id" ]).all? { |k| params[k].nil? || params[k].is_a?(String) }

        extras = LIMITS.to_h { |key, _| [ key.to_sym, params[key]&.strip.presence ] }
        return Result.fail(:invalid_instrument) if LIMITS.any? { |key, max| extras[key.to_sym].to_s.length > max }

        hl7 = params["hl7_equipment_id"]&.strip.presence
        return Result.fail(:invalid_instrument) unless hl7.nil? || hl7.match?(HL7_ID)
        return Result.fail(:invalid_active) unless [ true, false ].include?(params["active"])

        ApplicationRecord.transaction(requires_new: true) do
          provider.lock!
          others = LabInstrument.where(exam_provider_id: provider.id).where.not(id: instrument&.id)
          next Result.fail(:duplicate_hl7_equipment_id) if hl7 && others.where("upper(hl7_equipment_id) = ?", hl7.upcase).exists?
          next Result.fail(:duplicate_name) if others.where("lower(name) = ?", name.downcase).exists?

          record = instrument || LabInstrument.new(exam_provider: provider)
          record.update!(name: name, sector: params["sector"], hl7_equipment_id: hl7, active: params["active"], **extras)
          Result.ok(instrument: record)
        end
      end

      def provider_for(id)
        id.is_a?(String) && id.match?(Exams::BaseController::UUID) ? Access.labs.find_by(id: id) : nil
      end
      private_class_method :provider_for
    end
  end
  ```

  `app/services/lab/admin_json.rb` — em `version(v)` acrescente `lab_instrument_id: v.lab_instrument_id,` depois de `instrument_test_code:`; e o método novo:

  ```ruby
      def instrument(i)
        { id: i.id, provider_id: i.exam_provider_id, name: i.name, manufacturer: i.manufacturer, model: i.model,
          serial_number: i.serial_number, sector: i.sector, hl7_equipment_id: i.hl7_equipment_id, active: i.active }
      end
  ```

  `app/services/lab/access.rb` — depois de `releaser_link`:

  ```ruby
      # RT ou substituto com ligação ativa ao laboratório (ADR 0038 §8: aceitar
      # referência, exceção, revisão, regras, ação corretiva).
      def technical?(user, provider_id)
        return false unless user.has_role?("lab_releaser")

        LabReleaser.active.where(user_id: user.id, exam_provider_id: provider_id).where("technical_lead OR technical_deputy").exists?
      end
  ```

  `app/commands/lab/create_version.rb` — depois da validação de `code` e antes de `analytes = Analytes.parse(...)`:

  ```ruby
        # ADR 0038: aparelho do mesmo laboratório, ativo (ou nenhum).
        raw_instrument = params["lab_instrument_id"]
        unless raw_instrument.nil?
          valid = raw_instrument.is_a?(String) && raw_instrument.match?(Exams::BaseController::UUID) &&
                  LabInstrument.active.exists?(id: raw_instrument, exam_provider_id: exam.exam_provider_id)
          return Result.fail(:invalid_version) unless valid
        end
  ```
  e no `LabExamVersion.create!(...)` acrescente `lab_instrument_id: raw_instrument,`.

  `app/services/lab/gate.rb` — acrescente (C13):

  ```ruby
      # C13: aparelhos servem ao CIQ e ao estoque; basta um dos dois.
      def instruments_usable?(city, env: Rails.env) = qc_usable?(city, env: env) || inventory_usable?(city, env: env)
  ```

  `app/controllers/lab/context_controller.rb` — no `render json:`, acrescente a chave (C1):

  ```ruby
                       technical_lead_of: Lab::Access.labs.select { |lab| Lab::Access.technical?(user, lab.id) }.map(&:id),
  ```

  `app/controllers/lab/lookups_controller.rb` — em `exams`, cada exame passa a (C5):

  ```ruby
        { id: e.id, sigtap_code: e.sigtap_code, name: e.name, material: e.current_version.material, delivery_days: e.delivery_days,
          analytes: e.current_version.analytes.map { |a| { key: a["key"], name: a["name"] } } }
  ```
  e acrescente a `spec/requests/lab/context_spec.rb` uma linha no teste existente de contexto vazio: `expect(json_body["technical_lead_of"]).to eq([])`.

  `app/controllers/lab/instruments_controller.rb`:

  ```ruby
  # GET /lab/instruments?provider_id= (contrato §2): leitura para membros do laboratório e admin (C2).
  # C13: com lab_quality_control ou lab_inventory utilizável.
  module Lab
    class InstrumentsController < BaseController
      before_action do
        next if Lab::Gate.instruments_usable?(Current.city)

        render json: { error: "feature_disabled", feature: Lab::Gate::QC_KEY }, status: :forbidden
      end

      def index
        provider = uuid?(params[:provider_id]) && Lab::Access.labs.find_by(id: params[:provider_id])
        return not_found unless provider
        return forbid unless Lab::Access.member?(Current.user, provider.id) || Lab::Access.admin?(Current.user)

        render json: { instruments: LabInstrument.where(exam_provider_id: provider.id).order(:name).map { |i| Lab::AdminJson.instrument(i) } }
      end
    end
  end
  ```

  `app/controllers/lab/admin/instruments_controller.rb`:

  ```ruby
  # /lab/admin/instruments (contrato §2; F-21.25): municipal_admin, step-up nas escritas; C13 como em /lab/instruments.
  module Lab
    module Admin
      class InstrumentsController < BaseController
        prepend_before_action do
          next if Lab::Gate.instruments_usable?(Current.city)

          render json: { error: "feature_disabled", feature: Lab::Gate::QC_KEY }, status: :forbidden
        end
        before_action :admin_provider!, only: :index
        before_action :load_instrument, only: :update

        def index
          render json: { instruments: LabInstrument.where(exam_provider_id: @provider.id).order(:name).map { |i| Lab::AdminJson.instrument(i) } }
        end

        def create = respond(Lab::SaveInstrument.call(body_params), :created)
        def update = respond(Lab::SaveInstrument.call(body_params, instrument: @instrument), :ok)

        private

        def respond(result, status)
          return render_failure(result) if result.failure?

          render json: Lab::AdminJson.instrument(result.payload[:instrument]), status: status
        end

        def load_instrument
          @instrument = uuid?(params[:id]) && LabInstrument.find_by(id: params[:id])
          not_found unless @instrument
        end
      end
    end
  end
  ```

  `config/routes.rb` — dentro de `scope "/lab", module: :lab, as: :lab do`, logo depois de `get "context", ...`: `get "instruments", to: "instruments#index"`; e dentro de `scope "admin", module: :admin, as: :admin do`:

  ```ruby
        get   "instruments",               to: "instruments#index"
        post  "instruments",               to: "instruments#create"
        patch "instruments/:id",           to: "instruments#update"
  ```

- [ ] **Step 4: Rode:** `rspec spec/services/lab/qc/decimal_spec.rb spec/requests/lab/instruments_spec.rb spec/requests/lab/admin_catalog_spec.rb spec/requests/lab/context_spec.rb spec/requests/lab/entry_and_worklist_spec.rb` → verde.

- [ ] **Step 5: Commit** e mova o card F-21.25 para **Done**:

  ```bash
  git add app/commands/lab/save_instrument.rb app/controllers/lab/instruments_controller.rb app/controllers/lab/admin/instruments_controller.rb app/services/lab/qc/decimal.rb app/commands/lab/create_version.rb app/services/lab/admin_json.rb app/services/lab/access.rb app/services/lab/gate.rb app/controllers/lab/context_controller.rb app/controllers/lab/lookups_controller.rb config/routes.rb spec/requests/lab/instruments_spec.rb spec/requests/lab/context_spec.rb spec/services/lab/qc/decimal_spec.rb
  git commit -m "feat: add lab instruments and link exam versions to an instrument

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 5: Materiais, lotes e referência da bula

**Files:**
- Create: `app/controllers/lab/qc/{base_controller,materials_controller,lots_controller,targets_controller}.rb`, `app/commands/lab/qc/{save_material,create_lot,close_lot,create_target}.rb`, `app/services/lab/qc/json.rb`, `spec/requests/lab/qc_setup_spec.rb`
- Modify: `config/routes.rb`

**Interfaces:**
- Consumes: modelos da Task 3; `Lab::Access.technical?`, `Lab::Qc::Decimal` (Task 4).
- Produces: `Lab::Qc::BaseController` (403 `feature_disabled` com `lab_quality_control` desligado; `lab_param(key)`, `reader?(provider_id)` = membro ou admin, `configurer?(provider_id)` = admin ou RT/substituto, `technical?(provider_id)`, `releaser?(provider_id)` = liberador ligado ou admin); `Lab::Qc::SaveMaterial.call(params, provider: nil, material: nil) → Result(material:)`; `Lab::Qc::CreateLot.call(params, by:, now:) → Result(lot:)`; `Lab::Qc::CloseLot.call(lot:, now:) → Result(lot:)`; `Lab::Qc::CreateTarget.call(params, by:, now:) → Result(target:)` e `Lab::Qc::CreateTarget.next_version(lot_id, level, analyte_key, instrument_id) → Integer`; `Lab::Qc::Json.{material,lot,target,value,run,block,rule,exception,review}` (as quatro últimas entram nas Tasks 7–8). Rotas `GET|POST /lab/qc/materials`, `PATCH /lab/qc/materials/:id`, `GET|POST /lab/qc/lots`, `POST /lab/qc/lots/:id/close`, `GET|POST /lab/qc/targets`.

- [ ] **Step 1: Teste que falha** — `spec/requests/lab/qc_setup_spec.rb`:

  ```ruby
  require "rails_helper"

  # Contrato §3 (F-21.26, parte da bula): material com níveis e analitos;
  # lote com um código de barras por nível (único na cidade, nunca só 10
  # dígitos); referência da bula versionada; quem configura é o admin ou o
  # RT/substituto, com step-up; o interruptor fecha tudo.
  RSpec.describe "CIQ — cadastro", type: :request do
    before { qc_city! }

    let(:provider) { lab_provider! }
    let(:instrument) { qc_instrument!(provider) }
    let(:exam) { qc_exam!(provider: provider, instrument: instrument) }
    let(:lipids) { qc_exam!(provider: provider, instrument: instrument, sigtap_code: "0202010295", name: "Lipidograma", analytes: LabHelpers::LIPIDS, instrument_test_code: "LIP") }
    let(:lead) { lab_releaser!(provider, lead: true) }
    let(:material_body) do
      { provider_id: provider.id, name: "Lyphochek Assayed Chemistry Control", manufacturer: "Bio-Rad", levels: [ "Nível 1", "Nível 2" ],
        analytes: [ { lab_exam_id: exam.id, analyte_key: "glicose" } ], active: true }
    end

    def create_material(body = material_body)
      json_post "/lab/qc/materials", body
      json_body
    end

    def create_lot(material_id, codes = { "Nível 1" => "CQB2611N1", "Nível 2" => "CQB2611N2" }, number: "CQB-2611", expires: (Time.zone.today + 300).iso8601)
      json_post "/lab/qc/lots", material_id: material_id, lot_number: number, expires_on: expires, level_barcodes: codes
      json_body
    end

    it "interruptor desligado: 403" do
      Platform::Features.set!(city: City.find(Current.city.id), key: "lab_quality_control", enabled: false, maintainer: ledi_maintainer!)
      stepped_up!(lead)
      get "/lab/qc/materials", params: { provider_id: provider.id }
      expect(json_body).to eq("error" => "feature_disabled", "feature" => "lab_quality_control")
    end

    it "RT cadastra material e lote; recusas de níveis, analitos e códigos" do
      stepped_up!(lead)
      material = create_material
      expect(response).to have_http_status(:created)
      expect(material).to include("levels" => [ "Nível 1", "Nível 2" ], "has_lots" => false,
                                  "analytes" => [ { "lab_exam_id" => exam.id, "analyte_key" => "glicose" } ])
      create_material(material_body.merge(levels: [ "Nível 1", "Nível 1" ]))
      expect(json_body["error"]).to eq("invalid_levels")
      create_material(material_body.merge(analytes: [ { lab_exam_id: lipids.id, analyte_key: "ldl" } ]))
      expect(json_body["error"]).to eq("invalid_analytes") # calculado não tem controle próprio

      lot = create_lot(material["id"])
      expect(response).to have_http_status(:created)
      expect(lot).to include("lot_number" => "CQB-2611", "status" => "in_use", "expired" => false, "material_name" => "Lyphochek Assayed Chemistry Control",
                             "level_barcodes" => { "Nível 1" => "CQB2611N1", "Nível 2" => "CQB2611N2" })
      create_lot(material["id"], { "Nível 1" => "CQB2611N1", "Nível 2" => "OUTRO2" }, number: "CQB-2612")
      expect(json_body["error"]).to eq("barcode_taken")
      create_lot(material["id"], { "Nível 1" => "0000012301", "Nível 2" => "OUTRO2" }, number: "CQB-2612")
      expect(json_body["error"]).to eq("barcode_taken") # espaço das amostras
      create_lot(material["id"], { "Nível 1" => "com espaço", "Nível 2" => "OUTRO2" }, number: "CQB-2612")
      expect(json_body["error"]).to eq("invalid_barcode")
      create_lot(material["id"], { "Nível 1" => "OUTRO1" }, number: "CQB-2612")
      expect(json_body["error"]).to eq("invalid_levels")
      create_lot(material["id"], { "Nível 1" => "OUTRO1", "Nível 2" => "OUTRO2" }, number: "CQB-2611")
      expect(json_body["error"]).to eq("duplicate_lot")
      create_lot(material["id"], { "Nível 1" => "OUTRO1", "Nível 2" => "OUTRO2" }, number: "CQB-2612", expires: (Time.zone.today - 1).iso8601)
      expect(json_body["error"]).to eq("expired_lot")

      json_patch "/lab/qc/materials/#{material['id']}", material_body.except(:provider_id).merge(levels: [ "Nível 1", "Nível 2", "Nível 3" ])
      expect(json_body["error"]).to eq("invalid_levels") # já tem lote
      json_patch "/lab/qc/materials/#{material['id']}", material_body.except(:provider_id).merge(name: "Lyphochek (2 níveis)")
      expect(json_body).to include("name" => "Lyphochek (2 níveis)", "has_lots" => true)

      json_post "/lab/qc/lots/#{lot['id']}/close", {}
      expect(json_body).to include("status" => "closed")
      json_post "/lab/qc/lots/#{lot['id']}/close", {}
      expect(response).to have_http_status(:conflict)
      get "/lab/qc/lots", params: { provider_id: provider.id }
      expect(json_body["lots"].map { |l| l["lot_number"] }).to eq([ "CQB-2611" ])
    end

    it "quem configura: admin e RT/substituto com step-up; liberador comum e técnica não" do
      sign_in_as(lead)
      create_material
      expect([ response.status, json_body["error"] ]).to eq([ 401, "mfa_required" ])
      stepped_up!(lab_releaser!(provider))
      create_material
      expect(response).to have_http_status(:forbidden)
      stepped_up!(lab_staff!) # com step-up, para a recusa ser do papel
      create_material
      expect(response).to have_http_status(:forbidden)
      get "/lab/qc/materials", params: { provider_id: provider.id }
      expect(response).to have_http_status(:ok)
      stepped_up!(lab_admin!)
      create_material
      expect(response).to have_http_status(:created)
    end

    it "referência da bula: vírgula, dp positivo, versões, lote encerrado; evento só com ids" do
      stepped_up!(lead)
      material = create_material
      lot = create_lot(material["id"])
      body = { lot_id: lot["id"], level: "Nível 1", analyte_key: "glicose", instrument_id: instrument.id, mean: "90,5", sd: "4,5" }
      json_post "/lab/qc/targets", body
      expect(response).to have_http_status(:created)
      expect(json_body).to include("version" => 1, "source" => "package_insert", "mean" => "90.5", "sd" => "4.5", "based_on_runs" => nil,
                                   "reason" => nil)
      json_post "/lab/qc/targets", body.merge(mean: "90.5")
      expect(json_body["error"]).to eq("invalid_value")
      json_post "/lab/qc/targets", body.merge(sd: "0")
      expect(json_body["error"]).to eq("invalid_value")
      json_post "/lab/qc/targets", body.merge(level: "Nível 9")
      expect(json_body["error"]).to eq("invalid_levels")
      json_post "/lab/qc/targets", body.merge(analyte_key: "ureia")
      expect(json_body["error"]).to eq("invalid_analytes")
      json_post "/lab/qc/targets", body.merge(instrument_id: qc_instrument!(lab_provider!, hl7_equipment_id: "X1").id)
      expect(json_body["error"]).to eq("invalid_instrument")
      json_post "/lab/qc/targets", body.merge(mean: "91")
      expect(json_body["error"]).to eq("reason_required") # C6: troca da vigente
      json_post "/lab/qc/targets", body.merge(mean: "91", reason: "Bula do lote corrigida pelo fabricante")
      expect(json_body).to include("version" => 2, "reason" => "Bula do lote corrigida pelo fabricante")
      get "/lab/qc/targets", params: { lot_id: lot["id"] }
      expect(json_body["current"].map { |t| [ t["level"], t["version"], t["mean"] ] }).to eq([ [ "Nível 1", 2, "91" ] ])
      expect(json_body["history"].map { |t| t["version"] }).to eq([ 1, 2 ])
      expect(DomainEvent.where(name: "lab_qc_target.accepted").map { |e| e.payload.keys.sort }.uniq).to eq([ %w[lab_qc_target_id source] ])
      json_post "/lab/qc/lots/#{lot['id']}/close", {}
      json_post "/lab/qc/targets", body
      expect(json_body["error"]).to eq("lot_closed")
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar:** `rspec spec/requests/lab/qc_setup_spec.rb` → 404 (rotas).

- [ ] **Step 3: Base e JSON.** `app/controllers/lab/qc/base_controller.rb`:

  ```ruby
  # Base das rotas /lab/qc/* e /lab/eqa/* (contrato §3–§4; ADR 0038 §8):
  # municipal_lab (herdado) e lab_quality_control utilizáveis. Quem lê:
  # membro do laboratório ou admin. Quem configura: admin ou RT/substituto.
  module Lab
    module Qc
      class BaseController < Lab::BaseController
        before_action :require_quality_control!

        private

        def require_quality_control!
          return if Lab::Gate.qc_usable?(Current.city)

          render json: { error: "feature_disabled", feature: Lab::Gate::QC_KEY }, status: :forbidden
        end

        def lab_param(key = :provider_id) = uuid?(params[key]) ? Lab::Access.labs.find_by(id: params[key]) : nil
        def reader?(provider_id) = Lab::Access.member?(Current.user, provider_id) || Lab::Access.admin?(Current.user)
        def configurer?(provider_id) = Lab::Access.admin?(Current.user) || Lab::Access.technical?(Current.user, provider_id)
        def technical?(provider_id) = Lab::Access.technical?(Current.user, provider_id)
        def releaser?(provider_id) = Lab::Access.releaser_link(Current.user, provider_id).present? || Lab::Access.admin?(Current.user)

        # before_action: @provider do parâmetro, com a permissão pedida.
        def provider!(key = :provider_id, allow: :reader?)
          @provider = lab_param(key)
          return not_found unless @provider

          forbid unless send(allow, @provider.id)
        end
      end
    end
  end
  ```

  `app/services/lab/qc/json.rb`:

  ```ruby
  # Formas do CIQ (contrato §3). Números saem como string com ponto (Desvio 11).
  # Nada de paciente: revisão e exceção levam número da requisição e código do exame.
  module Lab
    module Qc
      module Json
        module_function

        def out(number) = Decimal.out(number)
        def at(time) = time&.iso8601

        def material(m)
          { id: m.id, provider_id: m.exam_provider_id, name: m.name, manufacturer: m.manufacturer, levels: m.levels,
            analytes: m.analytes, active: m.active, has_lots: m.lots.exists? }
        end

        def lot(l)
          { id: l.id, material_id: l.lab_qc_material_id, material_name: l.lab_qc_material.name, lot_number: l.lot_number, expires_on: l.expires_on.iso8601,
            level_barcodes: l.level_barcodes, status: l.status, opened_on: l.opened_on.iso8601, closed_on: l.closed_on&.iso8601,
            expired: l.expired? }
        end

        def target(t)
          { id: t.id, lot_id: t.lab_qc_lot_id, level: t.level, analyte_key: t.analyte_key, instrument_id: t.lab_instrument_id,
            version: t.version, source: t.source, mean: out(t.mean), sd: out(t.sd), based_on_runs: t.based_on_runs, reason: t.reason,
            accepted_by: Screenings::Json.staff_name(t.accepted_by_user), accepted_at: at(t.accepted_at) }
        end
      end
    end
  end
  ```

- [ ] **Step 4: Comandos.** `app/commands/lab/qc/save_material.rb`:

  ```ruby
  # Material de controle (contrato §3): níveis (1..6, distintos) e analitos
  # numéricos não calculados da versão vigente de exames do laboratório.
  # Depois do primeiro lote, os níveis não mudam e nenhum analito sai.
  module Lab
    module Qc
      module SaveMaterial
        module_function

        def call(params, provider: nil, material: nil)
          params = params.to_h.stringify_keys
          provider ||= material.exam_provider
          name = params["name"].is_a?(String) ? params["name"].strip : ""
          manufacturer = params["manufacturer"]
          return Result.fail(:invalid_name) unless name.length.between?(1, 80)
          return Result.fail(:invalid_name) unless manufacturer.nil? || (manufacturer.is_a?(String) && manufacturer.strip.length <= 80)

          levels = parse_levels(params["levels"])
          return Result.fail(:invalid_levels) unless levels

          analytes = parse_analytes(params["analytes"], provider)
          return Result.fail(:invalid_analytes) unless analytes
          return Result.fail(:invalid_active) unless [ true, false ].include?(params["active"])

          ApplicationRecord.transaction(requires_new: true) do
            if material
              material.lock!
              if material.lots.exists?
                next Result.fail(:invalid_levels) unless levels == material.levels
                next Result.fail(:invalid_analytes) unless (material.analytes - analytes).empty?
              end
            end
            record = material || LabQcMaterial.new(exam_provider: provider)
            record.update!(name: name, manufacturer: manufacturer&.strip.presence, levels: levels, analytes: analytes, active: params["active"])
            Result.ok(material: record)
          end
        end

        def parse_levels(raw)
          return nil unless raw.is_a?(Array) && raw.size.between?(1, 6) && raw.all? { |l| l.is_a?(String) && l.strip.length.between?(1, 40) }

          levels = raw.map(&:strip)
          levels.uniq.size == levels.size ? levels : nil
        end

        def parse_analytes(raw, provider)
          return nil unless raw.is_a?(Array) && raw.size.between?(1, 60)

          exams = LabExam.where(exam_provider_id: provider.id).includes(:current_version).index_by(&:id)
          list = raw.map do |entry|
            entry = entry.to_unsafe_h if entry.respond_to?(:to_unsafe_h)
            return nil unless entry.is_a?(Hash)

            entry = entry.stringify_keys
            analyte = exams[entry["lab_exam_id"]]&.current_version&.analyte(entry["analyte_key"])
            return nil unless analyte && analyte["value_type"] == "numeric" && analyte["formula"].blank?

            { "lab_exam_id" => entry["lab_exam_id"], "analyte_key" => analyte["key"] }
          end
          list.uniq.size == list.size ? list : nil
        end
        private_class_method :parse_levels, :parse_analytes
      end
    end
  end
  ```

  `app/commands/lab/qc/create_lot.rb`:

  ```ruby
  # Lote do material (contrato §3): número único no material; validade não
  # vencida; um código de barras por nível do material (Desvio 1): formato
  # [A-Za-z0-9._-]{3,30}; só 10 dígitos é o espaço das amostras (barcode_taken);
  # único na cidade (barcode_taken).
  module Lab
    module Qc
      module CreateLot
        module_function

        def call(params, by:, now: Time.current)
          params = params.to_h.stringify_keys
          material = params["material_id"].to_s.match?(Exams::BaseController::UUID) ? LabQcMaterial.find_by(id: params["material_id"]) : nil
          return Result.fail(:not_found) unless material

          number = params["lot_number"].is_a?(String) ? params["lot_number"].strip : ""
          return Result.fail(:invalid_lot_number) unless number.length.between?(1, 40)

          expires_on = parse_date(params["expires_on"])
          return Result.fail(:invalid_expires_on) unless expires_on
          return Result.fail(:expired_lot) if expires_on < now.in_time_zone.to_date

          codes = params["level_barcodes"]
          codes = codes.to_unsafe_h if codes.respond_to?(:to_unsafe_h)
          return Result.fail(:invalid_levels) unless codes.is_a?(Hash) && codes.keys.map(&:to_s).sort == material.levels.sort

          codes = codes.to_h { |level, code| [ level.to_s, code.is_a?(String) ? code.strip : code ] }
          return Result.fail(:barcode_taken) if codes.values.any? { |c| c.is_a?(String) && c.match?(LabQcLotBarcode::SAMPLE_SPACE) }
          return Result.fail(:invalid_barcode) unless codes.values.all? { |c| LabQcLotBarcode.valid_code?(c) }
          return Result.fail(:barcode_taken) unless codes.values.uniq.size == codes.size

          ApplicationRecord.transaction(requires_new: true) do
            material.lock!
            next Result.fail(:duplicate_lot) if material.lots.exists?(lot_number: number)
            next Result.fail(:barcode_taken) if LabQcLotBarcode.exists?(barcode: codes.values)

            lot = LabQcLot.create!(lab_qc_material: material, lot_number: number, expires_on: expires_on, status: "in_use",
                                   opened_on: now.in_time_zone.to_date, created_by_user: by)
            codes.each { |level, code| LabQcLotBarcode.create!(lab_qc_lot: lot, level: level, barcode: code, created_at: now) }
            Result.ok(lot: lot)
          end
        rescue ActiveRecord::RecordNotUnique
          Result.fail(:barcode_taken) # o mesmo código gravado por outro lote ao mesmo tempo
        end

        def parse_date(raw)
          raw.is_a?(String) && raw.match?(/\A\d{4}-\d{2}-\d{2}\z/) ? Date.iso8601(raw) : nil
        rescue Date::Error
          nil
        end
      end
    end
  end
  ```

  `app/commands/lab/qc/close_lot.rb`:

  ```ruby
  # Encerrar o lote de controle (contrato §3): só in_use → closed (trigger).
  module Lab
    module Qc
      module CloseLot
        module_function

        def call(lot:, now: Time.current)
          ApplicationRecord.transaction(requires_new: true) do
            lot.lock!
            next Result.fail(:invalid_transition, details: { status: lot.status }) unless lot.status == "in_use"

            lot.update!(status: "closed", closed_on: now.in_time_zone.to_date)
            Result.ok(lot: lot)
          end
        end
      end
    end
  end
  ```

  `app/commands/lab/qc/create_target.rb`:

  ```ruby
  # Referência da bula (contrato §3): por lote, nível, analito do material e
  # aparelho ativo do laboratório; média e dp com vírgula (dp > 0); nova
  # versão a cada gravação (nunca edita), com motivo quando troca a vigente
  # (C6). Lote travado: versões sem buraco.
  module Lab
    module Qc
      module CreateTarget
        module_function

        def call(params, by:, now: Time.current)
          params = params.to_h.stringify_keys
          lot = params["lot_id"].to_s.match?(Exams::BaseController::UUID) ? LabQcLot.find_by(id: params["lot_id"]) : nil
          return Result.fail(:not_found) unless lot

          material = lot.lab_qc_material
          return Result.fail(:invalid_levels) unless material.levels.include?(params["level"])
          return Result.fail(:invalid_analytes) unless material.analyte_entry(params["analyte_key"])

          instrument = instrument_for(params["instrument_id"], material.exam_provider_id)
          return Result.fail(:invalid_instrument) unless instrument

          mean = Decimal.typed(params["mean"])
          sd = Decimal.typed(params["sd"])
          return Result.fail(:invalid_value) unless mean && sd && sd.positive?

          reason = params["reason"].is_a?(String) ? params["reason"].strip.presence : nil
          return Result.fail(:reason_required) if reason && reason.length > 1000

          ApplicationRecord.transaction(requires_new: true) do
            lot.lock!
            next Result.fail(:lot_closed) if lot.status == "closed"
            next Result.fail(:expired_lot) if lot.expired?(now.in_time_zone.to_date)

            # C6 (spec §9): trocar a referência vigente exige motivo; a primeira, não.
            replacing = LabQcTarget.current(lot_id: lot.id, level: params["level"], analyte_key: params["analyte_key"], instrument_id: instrument.id)
            next Result.fail(:reason_required) if replacing && reason.nil?

            target = LabQcTarget.create!(lab_qc_lot: lot, level: params["level"], analyte_key: params["analyte_key"], lab_instrument: instrument,
                                         version: next_version(lot.id, params["level"], params["analyte_key"], instrument.id),
                                         source: "package_insert", mean: mean, sd: sd, reason: reason, accepted_by_user: by, accepted_at: now,
                                         created_at: now)
            DomainEvents.publish("lab_qc_target.accepted", lab_qc_target_id: target.id, source: "package_insert")
            Result.ok(target: target)
          end
        end

        # Chame com o lote travado.
        def next_version(lot_id, level, analyte_key, instrument_id)
          LabQcTarget.where(lab_qc_lot_id: lot_id, level: level, analyte_key: analyte_key, lab_instrument_id: instrument_id).maximum(:version).to_i + 1
        end

        def instrument_for(id, provider_id)
          return nil unless id.to_s.match?(Exams::BaseController::UUID)

          LabInstrument.active.find_by(id: id, exam_provider_id: provider_id)
        end
        private_class_method :instrument_for
      end
    end
  end
  ```

- [ ] **Step 5: Controllers.** `app/controllers/lab/qc/materials_controller.rb`:

  ```ruby
  # /lab/qc/materials (contrato §3): lê membro ou admin; grava admin ou RT/substituto, com step-up.
  module Lab
    module Qc
      class MaterialsController < BaseController
        before_action :require_step_up!, only: %i[create update]
        before_action -> { provider! }, only: :index

        def index
          render json: { materials: LabQcMaterial.where(exam_provider_id: @provider.id).order(:name).map { |m| Json.material(m) } }
        end

        def create
          provider = lab_param
          return render_failure(Result.fail(:invalid_provider)) unless provider
          return forbid unless configurer?(provider.id)

          respond(SaveMaterial.call(body_params, provider: provider), :created)
        end

        def update
          material = uuid?(params[:id]) && LabQcMaterial.find_by(id: params[:id])
          return not_found unless material
          return forbid unless configurer?(material.exam_provider_id)

          respond(SaveMaterial.call(body_params, material: material), :ok)
        end

        private

        def respond(result, status)
          return render_failure(result) if result.failure?

          render json: Json.material(result.payload[:material]), status: status
        end
      end
    end
  end
  ```

  `app/controllers/lab/qc/lots_controller.rb`:

  ```ruby
  # /lab/qc/lots (contrato §3): lista por laboratório (ou material); criar e encerrar: admin ou RT/substituto, com step-up.
  module Lab
    module Qc
      class LotsController < BaseController
        before_action :require_step_up!, only: %i[create close]
        before_action -> { provider! }, only: :index

        def index
          lots = LabQcLot.joins(:lab_qc_material).where(lab_qc_materials: { exam_provider_id: @provider.id })
          lots = lots.where(lab_qc_material_id: params[:material_id]) if uuid?(params[:material_id])
          lots = lots.where(status: params[:status]) if LabQcLot::STATUSES.include?(params[:status]) # C2
          render json: { lots: lots.includes(:barcodes, :lab_qc_material).order(:status, :expires_on, :lot_number).map { |l| Json.lot(l) } }
        end

        def create
          material = uuid?(params[:material_id]) && LabQcMaterial.find_by(id: params[:material_id])
          return render_failure(Result.fail(:not_found)) unless material
          return forbid unless configurer?(material.exam_provider_id)

          result = CreateLot.call(body_params, by: Current.user)
          return render_failure(result) if result.failure?

          render json: Json.lot(result.payload[:lot]), status: :created
        end

        def close
          lot = uuid?(params[:id]) && LabQcLot.find_by(id: params[:id])
          return not_found unless lot
          return forbid unless configurer?(lot.lab_qc_material.exam_provider_id)

          result = CloseLot.call(lot: lot)
          return render_failure(result) if result.failure?

          render json: Json.lot(result.payload[:lot])
        end
      end
    end
  end
  ```

  `app/controllers/lab/qc/targets_controller.rb` (a Task 9 acrescenta `suggestions` e `accept`):

  ```ruby
  # /lab/qc/targets (contrato §3): vigentes e histórico do lote (leitura);
  # referência da bula (admin ou RT/substituto, step-up).
  module Lab
    module Qc
      class TargetsController < BaseController
        before_action :require_step_up!, only: %i[create]

        def index
          lot = uuid?(params[:lot_id]) && LabQcLot.find_by(id: params[:lot_id])
          return not_found unless lot
          return forbid unless reader?(lot.lab_qc_material.exam_provider_id)

          history = LabQcTarget.where(lab_qc_lot_id: lot.id).includes(:accepted_by_user).order(:analyte_key, :level, :lab_instrument_id, :version).to_a
          current = history.group_by { |t| [ t.analyte_key, t.level, t.lab_instrument_id ] }.values.map(&:last)
          render json: { current: current.map { |t| Json.target(t) }, history: history.map { |t| Json.target(t) } }
        end

        def create
          lot = uuid?(params[:lot_id]) && LabQcLot.find_by(id: params[:lot_id])
          return render_failure(Result.fail(:not_found)) unless lot
          return forbid unless configurer?(lot.lab_qc_material.exam_provider_id)

          result = CreateTarget.call(body_params, by: Current.user)
          return render_failure(result) if result.failure?

          render json: Json.target(result.payload[:target]), status: :created
        end
      end
    end
  end
  ```

  `config/routes.rb` — dentro de `scope "/lab", module: :lab, as: :lab do`, antes de `scope "admin"`:

  ```ruby
      # Módulo 21c (ADR 0038): CIQ. As literais antes de `:id`.
      scope "qc", module: :qc, as: :qc do
        get   "materials",            to: "materials#index"
        post  "materials",            to: "materials#create"
        patch "materials/:id",        to: "materials#update"
        get   "lots",                 to: "lots#index"
        post  "lots",                 to: "lots#create"
        post  "lots/:id/close",       to: "lots#close"
        get   "targets",              to: "targets#index"
        post  "targets",              to: "targets#create"
      end
  ```

- [ ] **Step 6: Rode:** `rspec spec/requests/lab/qc_setup_spec.rb` → verde.

- [ ] **Step 7: Commit:**

  ```bash
  git add app/controllers/lab/qc/base_controller.rb app/controllers/lab/qc/materials_controller.rb app/controllers/lab/qc/lots_controller.rb app/controllers/lab/qc/targets_controller.rb app/commands/lab/qc/save_material.rb app/commands/lab/qc/create_lot.rb app/commands/lab/qc/close_lot.rb app/commands/lab/qc/create_target.rb app/services/lab/qc/json.rb config/routes.rb spec/requests/lab/qc_setup_spec.rb
  git commit -m "feat: add quality control materials, lots and package insert targets

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

---

## Fatia 3 — Westgard, corridas e bloqueio

### Task 6: Regras de Westgard (função pura)

**Files:**
- Create: `app/services/lab/qc/westgard.rb`, `spec/services/lab/qc/westgard_spec.rb`

**Interfaces:**
- Consumes: `LabQcRule::RULES` (Task 3).
- Produces: `Lab::Qc::Westgard.evaluate(z: BigDecimal, series: [BigDecimal], peers: [BigDecimal], enabled: [String]) → Evaluation(rules_fired: [String] na ordem de RULES, outcome: "ok"|"warning"|"reject")`. `series` = z anteriores do mesmo lote, nível, analito e aparelho em corridas não reprovadas, do mais antigo ao mais recente (até 9); `peers` = z mais recente de cada outro nível do mesmo analito na mesma corrida.

- [ ] **Step 1: Teste que falha** — `spec/services/lab/qc/westgard_spec.rb` (séries conhecidas, um caso que dispara e um que não dispara por regra):

  ```ruby
  require "rails_helper"

  # Spec §5.3 e §10: cada regra com série conhecida (dispara e não dispara);
  # R-4s e 2-2s também entre níveis da mesma corrida; limites estritos
  # (Desvio 5); regra desligada não dispara; rules_fired em ordem fixa.
  RSpec.describe Lab::Qc::Westgard do
    def ev(z, series: [], peers: [], enabled: LabQcRule::RULES)
      described_class.evaluate(z: BigDecimal(z.to_s), series: series.map { |v| BigDecimal(v.to_s) },
                               peers: peers.map { |v| BigDecimal(v.to_s) }, enabled: enabled)
    end

    def fired(...) = ev(...).rules_fired

    it "1-2s: aviso acima de 2 dp; 2,0 exato não dispara" do
      expect(ev(2.1)).to have_attributes(rules_fired: [ "1_2s" ], outcome: "warning")
      expect(ev(-2.5)).to have_attributes(rules_fired: [ "1_2s" ], outcome: "warning")
      expect(ev(2.0)).to have_attributes(rules_fired: [], outcome: "ok")
      expect(ev(0)).to have_attributes(rules_fired: [], outcome: "ok")
    end

    it "1-3s: reprova acima de 3 dp; 3,0 exato só avisa" do
      expect(ev(3.1)).to have_attributes(rules_fired: %w[1_2s 1_3s], outcome: "reject")
      expect(ev(-3.2)).to have_attributes(rules_fired: %w[1_2s 1_3s], outcome: "reject")
      expect(ev(3.0)).to have_attributes(rules_fired: [ "1_2s" ], outcome: "warning")
    end

    it "2-2s no mesmo nível: dois consecutivos além de 2 dp do mesmo lado" do
      expect(ev(2.3, series: [ 0.1, 2.2 ])).to have_attributes(rules_fired: %w[1_2s 2_2s], outcome: "reject")
      expect(fired(-2.4, series: [ -2.1 ])).to eq(%w[1_2s 2_2s])
      expect(fired(2.3, series: [ 2.2, 0.1 ])).to eq([ "1_2s" ]) # não consecutivo
      expect(fired(2.3, series: [ -2.2 ])).to eq([ "1_2s" ])     # lados opostos
      expect(fired(2.3, series: [ 2.0 ])).to eq([ "1_2s" ])      # 2,0 não passa de 2
    end

    it "2-2s entre níveis na mesma corrida" do
      expect(ev(2.4, peers: [ 2.6 ])).to have_attributes(rules_fired: %w[1_2s 2_2s], outcome: "reject")
      expect(fired(2.4, peers: [ 1.9 ])).to eq([ "1_2s" ])
    end

    it "R-4s entre níveis: um acima de +2 e outro abaixo de −2 na mesma corrida" do
      expect(ev(2.1, peers: [ -2.2 ])).to have_attributes(rules_fired: %w[1_2s R_4s], outcome: "reject")
      expect(fired(-2.3, peers: [ 2.05 ])).to eq(%w[1_2s R_4s])
      expect(fired(2.1, peers: [ -1.9 ])).to eq([ "1_2s" ]) # amplitude 4,0 sem passar de −2
      expect(fired(1.5, peers: [ -2.6 ])).to eq([])          # o atual não passa de +2
      expect(fired(2.5, series: [ -2.5 ])).to eq([ "1_2s" ]) # R-4s não olha a série
    end

    it "4-1s: quatro consecutivos além de 1 dp do mesmo lado" do
      expect(ev(1.2, series: [ 1.1, 1.5, 1.3 ])).to have_attributes(rules_fired: [ "4_1s" ], outcome: "reject")
      expect(fired(-1.4, series: [ 0.2, -1.1, -1.2, -1.05 ])).to eq([ "4_1s" ])
      expect(fired(1.2, series: [ 1.1, 0.9, 1.3 ])).to eq([])
      expect(fired(1.2, series: [ 1.5, 1.3 ])).to eq([]) # só três valores
      expect(fired(1.2, series: [ 1.0, 1.5, 1.3 ])).to eq([]) # 1,0 não passa de 1
    end

    it "10x: dez consecutivos do mesmo lado da média" do
      expect(ev(0.3, series: [ 0.1 ] * 9)).to have_attributes(rules_fired: [ "10x" ], outcome: "reject")
      expect(fired(-0.2, series: [ -0.5 ] * 9)).to eq([ "10x" ])
      expect(fired(0.3, series: [ 0.1 ] * 8 + [ 0.0 ])).to eq([]) # zero quebra a sequência
      expect(fired(0.3, series: [ 0.1 ] * 8)).to eq([])           # só nove valores
      expect(fired(-0.3, series: [ 0.1 ] * 9)).to eq([])
    end

    it "regra desligada não dispara; ordem fixa" do
      expect(ev(3.5, enabled: LabQcRule::RULES - [ "1_3s" ])).to have_attributes(rules_fired: [ "1_2s" ], outcome: "warning")
      expect(ev(3.5, enabled: [])).to have_attributes(rules_fired: [], outcome: "ok")
      expect(fired(3.5, series: [ 2.5 ], peers: [ -2.5 ])).to eq(%w[1_2s 1_3s 2_2s R_4s])
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar:** `rspec spec/services/lab/qc/westgard_spec.rb` → `NameError: Lab::Qc::Westgard`.

- [ ] **Step 3: Implemente** `app/services/lab/qc/westgard.rb`:

  ```ruby
  # Regras de Westgard sobre z-scores (spec §5.3; Desvio 5). Função pura:
  # quem chama monta a série (mesmo lote, nível, analito e aparelho, corridas
  # não reprovadas, mais antigo primeiro) e os pares (outros níveis do mesmo
  # analito na mesma corrida). 1-2s é aviso; o resto reprova. Limites estritos.
  module Lab
    module Qc
      module Westgard
        RULES = LabQcRule::RULES
        REJECT = %w[1_3s 2_2s R_4s 4_1s 10x].freeze
        ONE = BigDecimal("1")
        TWO = BigDecimal("2")
        THREE = BigDecimal("3")
        Evaluation = Data.define(:rules_fired, :outcome)

        module_function

        def evaluate(z:, series:, peers:, enabled: RULES)
          checks = {
            "1_2s" => -> { z.abs > TWO },
            "1_3s" => -> { z.abs > THREE },
            "2_2s" => -> { same_side?(z, series.last, TWO) || peers.any? { |p| same_side?(z, p, TWO) } },
            "R_4s" => -> { peers.any? { |p| (z > TWO && p < -TWO) || (z < -TWO && p > TWO) } },
            "4_1s" => -> { streak?([ *series.last(3), z ], 4, ONE) },
            "10x" => -> { streak?([ *series.last(9), z ], 10, 0) }
          }
          fired = RULES.select { |rule| enabled.include?(rule) && checks.fetch(rule).call }
          outcome = fired.intersect?(REJECT) ? "reject" : (fired.include?("1_2s") ? "warning" : "ok")
          Evaluation.new(rules_fired: fired, outcome: outcome)
        end

        def same_side?(a, b, limit) = !b.nil? && ((a > limit && b > limit) || (a < -limit && b < -limit))
        def streak?(values, size, limit) = values.size == size && (values.all? { |v| v > limit } || values.all? { |v| v < -limit })
        private_class_method :same_side?, :streak?
      end
    end
  end
  ```

- [ ] **Step 4: Rode:** `rspec spec/services/lab/qc/westgard_spec.rb` → verde.

- [ ] **Step 5: Commit:**

  ```bash
  git add app/services/lab/qc/westgard.rb spec/services/lab/qc/westgard_spec.rb
  git commit -m "feat: add Westgard rule evaluation for quality control values

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 7: Valores e corridas, avaliação na gravação, regras, bloqueio e ação corretiva (F-21.27)

**Files:**
- Create: `app/commands/lab/qc/{record_value,open_run,save_rules,record_corrective_action}.rb`, `app/services/lab/qc/{blocks,reviews}.rb`, `app/controllers/lab/qc/{values_controller,runs_controller,rules_controller,blocks_controller}.rb`, `spec/commands/lab/qc/record_value_spec.rb`, `spec/requests/lab/qc_runs_spec.rb`
- Modify: `app/services/lab/qc/json.rb`, `config/routes.rb`

**Interfaces:**
- Consumes: `Lab::Qc::Westgard.evaluate` (Task 6); `LabQcTarget.current`, `LabQcRule.enabled_for`, `LabSupplyInUse.current_lot_id` (Task 3); `Lab::Qc::Decimal` (Task 4); `Lab::Qc::BaseController` (Task 5).
- Produces: `Lab::Qc::RecordValue.call(instrument:, lot:, level:, analyte_key:, value: BigDecimal, source: "typed"|"hl7", by: nil, receipt: nil, run_id: nil, run: nil, now: Time.current) → Result(value:, run:, block: LabQcBlock|nil, notice: nil|"corrective_action_required")`; falhas `invalid_instrument`, `invalid_levels`, `no_target`, `lot_closed`, `expired_lot`, `not_found`, `invalid_transition`. **`run:` é interno (HL7, Desvio 6): usa a corrida dada mesmo reprovada.** `Lab::Qc::Blocks.open!(instrument, analyte_key, run, now) → LabQcBlock`, `.try_close!(instrument:, analyte_key:, run:, material:, by:, now:) → [LabQcBlock|nil, notice|nil]`; `Lab::Qc::Reviews.enqueue!(block, run:) → Integer`; `Lab::Qc::OpenRun.call(instrument:, by:, now:)`; `Lab::Qc::SaveRules.call(instrument:, analyte_key:, rules:, justification:, by:, now:) → Result(rule:)`; `Lab::Qc::RecordCorrectiveAction.call(block:, text:, by:, now:) → Result(block:)`; `Lab::Qc::Json.{value,run,block,rule}`. Rotas `POST /lab/qc/values`, `GET|POST /lab/qc/runs`, `GET|PUT /lab/qc/rules`, `GET /lab/qc/blocks`, `GET /lab/qc/blocks/active/count`, `POST /lab/qc/blocks/:id/corrective_action`.

- [ ] **Step 1: Testes que falham.** `spec/commands/lab/qc/record_value_spec.rb`:

  ```ruby
  require "rails_helper"

  # Spec §5.2–§5.7 e Desvios 5–7: corrida do dia aberta no primeiro valor;
  # z contra a referência vigente; Westgard na gravação; 1-2s avisa; a
  # reprovação abre o bloqueio e a próxima corrida é nova; a série ignora a
  # corrida reprovada; o bloqueio fecha só com ação corretiva e corrida aceita
  # com todos os níveis gravados depois da ação.
  RSpec.describe Lab::Qc::RecordValue do
    include ActiveSupport::Testing::TimeHelpers

    before { qc_city! }

    let(:s) { qc_scenario! }
    let(:staff) { lab_staff! }
    let(:lead) { lab_releaser!(s.provider, lead: true) }

    def record(level, z, **opts)
      described_class.call(instrument: s.instrument, lot: s.lot, level: level, analyte_key: "glicose", value: qc_glucose(level, z),
                           source: "typed", by: staff, **opts)
    end

    it "abre a corrida do dia, calcula z e aprova; o segundo nível entra na mesma corrida" do
      first = record("Nível 1", 0.4).payload
      expect(first[:value]).to have_attributes(z_score: BigDecimal("0.4"), outcome: "ok", rules_fired: [], source: "typed", reagent_lot_id: nil)
      expect(first[:value].lab_qc_target.version).to eq(1)
      expect(first[:run]).to have_attributes(status: "approved", lab_instrument_id: s.instrument.id, sector: "biochemistry")
      expect(first[:block]).to be_nil
      second = record("Nível 2", -0.6).payload
      expect(second[:run].id).to eq(first[:run].id)
      expect(DomainEvent.where(name: "lab_qc_run.evaluated").map(&:payload)).to eq([ { "lab_qc_run_id" => first[:run].id, "status" => "approved" } ])
    end

    it "1-2s só avisa: a corrida fica warning" do
      warned = record("Nível 1", 2.2).payload
      expect([ warned[:value].outcome, warned[:value].rules_fired, warned[:run].status ]).to eq([ "warning", [ "1_2s" ], "warning" ])
    end

    it "reprovação: corrida rejected, bloqueio aberto, próxima corrida nova; série sem a reprovada" do
      record("Nível 1", 0.5)
      bad = record("Nível 2", 3.4).payload
      expect(bad[:value].rules_fired).to eq(%w[1_2s 1_3s])
      expect(bad[:run].status).to eq("rejected")
      expect(bad[:block]).to have_attributes(lab_instrument_id: s.instrument.id, analyte_key: "glicose", opened_by_run_id: bad[:run].id, closed_at: nil)
      refused = record("Nível 1", 0.1, run_id: bad[:run].id)
      expect([ refused.reason, refused.details ]).to eq([ :invalid_transition, { status: "rejected" } ])
      nxt = record("Nível 1", 0.1).payload
      expect(nxt[:run].id).not_to eq(bad[:run].id)
      again = record("Nível 2", 2.3).payload
      expect(again[:value].rules_fired).to eq([ "1_2s" ]) # o 3,4 reprovado não forma 2-2s
      expect(again[:notice]).to eq("corrective_action_required")
      expect(again[:block].reload.closed_at).to be_nil
      expect(LabQcBlock.active.count).to eq(1)
    end

    it "fecha só com ação corretiva e corrida aceita com todos os níveis gravados depois dela" do
      travel_to(Time.zone.now.change(hour: 9))
      record("Nível 1", 0.2)
      block = record("Nível 2", 3.5).payload[:block]
      record("Nível 1", 0.3)
      expect(record("Nível 2", -0.2).payload[:notice]).to eq("corrective_action_required")
      Lab::Qc::RecordCorrectiveAction.call(block: block, text: "Calibração refeita e reagente trocado", by: staff, now: Time.current + 60)
      after_l1 = record("Nível 1", 0.1, now: Time.current + 120).payload
      expect([ after_l1[:block].reload.closed_at, after_l1[:notice] ]).to eq([ nil, nil ]) # o Nível 2 da corrida é de antes da ação
      closing = record("Nível 2", 0.1, now: Time.current + 180).payload
      expect(closing[:block].reload).to have_attributes(closed_by_run_id: closing[:run].id, closed_by_user_id: staff.id)
      expect(closing[:block].closed_at).to be_present
      expect(DomainEvent.where(name: "lab_qc_block.closed").map { |e| e.payload.keys.sort }).to eq([ %w[lab_instrument_id lab_qc_block_id] ])
    end

    it "recusas: nível, analito, aparelho, lote encerrado ou vencido, sem referência" do
      expect(described_class.call(instrument: s.instrument, lot: s.lot, level: "Nível 9", analyte_key: "glicose", value: 1, source: "typed", by: staff).reason)
        .to eq(:invalid_levels)
      expect(described_class.call(instrument: s.instrument, lot: s.lot, level: "Nível 1", analyte_key: "ureia", value: 1, source: "typed", by: staff).reason)
        .to eq(:no_target)
      other = qc_instrument!(s.provider, name: "Analisador bioquímico 2", hl7_equipment_id: "BIO-02")
      expect(described_class.call(instrument: other, lot: s.lot, level: "Nível 1", analyte_key: "glicose", value: 1, source: "typed", by: staff).reason)
        .to eq(:no_target)
      alien = qc_instrument!(lab_provider!, hl7_equipment_id: "BIO-77")
      expect(described_class.call(instrument: alien, lot: s.lot, level: "Nível 1", analyte_key: "glicose", value: 1, source: "typed", by: staff).reason)
        .to eq(:invalid_instrument)
      expired = qc_lot!(s.material, expires_on: Time.zone.today - 1)
      qc_target!(expired, level: "Nível 1", analyte_key: "glicose", instrument: s.instrument, mean: 100, sd: 5)
      expect(described_class.call(instrument: s.instrument, lot: expired, level: "Nível 1", analyte_key: "glicose", value: 100, source: "typed", by: staff).reason)
        .to eq(:expired_lot)
      s.lot.update!(status: "closed", closed_on: Time.zone.today)
      expect(record("Nível 1", 0).reason).to eq(:lot_closed)
      expect(LabQcValue.count).to eq(0)
    end

    it "regra desligada pelo RT não reprova; valor nunca vai a evento" do
      Lab::Qc::SaveRules.call(instrument: s.instrument, analyte_key: "glicose", rules: LabQcRule::RULES.index_with(true).merge("1_3s" => false),
                              justification: "Faixa da bula larga demais para este lote", by: lead)
      value = record("Nível 1", 3.5).payload[:value]
      expect([ value.rules_fired, value.outcome ]).to eq([ [ "1_2s" ], "warning" ])
      expect(DomainEvent.where("name LIKE 'lab_qc%'").pluck(:payload).to_json).not_to include(qc_glucose("Nível 1", 3.5).to_s("F"))
    end
  end
  ```

  `spec/requests/lab/qc_runs_spec.rb`:

  ```ruby
  require "rails_helper"

  # Contrato §3 (F-21.27): valor digitado com vírgula, corrida, regras por
  # analito (RT, com justificativa ao desligar), bloqueios, contador sem
  # trilha e ação corretiva do RT.
  RSpec.describe "CIQ — corridas e bloqueios", type: :request do
    before { qc_city! }

    let(:s) { qc_scenario! }
    let(:staff) { lab_staff! }
    let(:lead) { lab_releaser!(s.provider, lead: true) }

    def post_value(level, value, run_id: nil)
      json_post "/lab/qc/values", run_id: run_id, instrument_id: s.instrument.id, lot_id: s.lot.id, level: level, analyte_key: "glicose", value: value
    end

    it "digitado com vírgula; corrida do dia; corrida a pedido; lista do dia" do
      sign_in_as(staff)
      post_value("Nível 1", "100,5")
      expect(response).to have_http_status(:created)
      expect(json_body["value"]).to include("value" => "100.5", "z_score" => 0.1, "outcome" => "ok", "rules_fired" => [], "source" => "typed",
                                            "target_version" => 1, "reagent_lot_id" => nil, "reagent_lot" => nil, "lot_id" => s.lot.id)
      expect(json_body["run"]).to include("status" => "approved", "instrument_id" => s.instrument.id)
      expect(json_body["run"]["values"].size).to eq(1)
      expect(json_body["block"]).to be_nil
      json_post "/lab/qc/runs", instrument_id: s.instrument.id
      opened = json_body
      expect(opened).to include("status" => "open", "values" => [])
      post_value("Nível 2", "250", run_id: opened["id"])
      expect(json_body["run"]["id"]).to eq(opened["id"])
      get "/lab/qc/runs", params: { instrument_id: s.instrument.id, date: Time.zone.today.iso8601 }
      expect(json_body["runs"].map { |r| r["values"].size }).to eq([ 1, 1 ])
    end

    it "vírgula na digitação: ponto recusado e nada gravado" do
      sign_in_as(staff)
      [ "98.5", "1.234", "98,55555", "" ].each do |raw|
        post_value("Nível 1", raw)
        expect([ response.status, json_body["error"] ]).to eq([ 422, "invalid_value" ])
      end
      expect(LabQcValue.count).to eq(0)
    end

    it "reprovação → bloqueio listado e contado; ação corretiva de quem opera (C14), uma vez" do
      sign_in_as(staff)
      post_value("Nível 1", comma(qc_glucose("Nível 1", 3.5)))
      block = json_body["block"]
      expect(block).to include("analyte_key" => "glicose", "instrument_id" => s.instrument.id, "closed_at" => nil, "corrective_action" => nil)
      get "/lab/qc/blocks", params: { provider_id: s.provider.id }
      expect(json_body["blocks"].map { |b| b["id"] }).to eq([ block["id"] ])
      get "/lab/qc/blocks/active/count"
      expect(json_body).to eq("count" => 1)
      stepped_up!(lab_releaser!(lab_provider!))
      json_post "/lab/qc/blocks/#{block['id']}/corrective_action", text: "Calibração refeita"
      expect(response).to have_http_status(:forbidden) # liberador de outro laboratório
      sign_in_as(staff) # C14: técnica do laboratório registra, sem step-up
      json_post "/lab/qc/blocks/#{block['id']}/corrective_action", text: "  "
      expect(json_body["error"]).to eq("corrective_action_required")
      json_post "/lab/qc/blocks/#{block['id']}/corrective_action", text: "Calibração refeita"
      expect(json_body).to include("corrective_action" => "Calibração refeita")
      json_post "/lab/qc/blocks/#{block['id']}/corrective_action", text: "Outra"
      expect(response).to have_http_status(:conflict)
      expect(DomainEvent.where(name: "lab_qc_block.action_recorded").map(&:payload)).to eq([ { "lab_qc_block_id" => block["id"] } ])
    end

    it "regras: padrão todas ligadas; só o RT muda; desligar exige justificativa" do
      sign_in_as(staff)
      get "/lab/qc/rules", params: { instrument_id: s.instrument.id, analyte_key: "glicose" }
      expect(json_body).to include("is_default" => true, "rules" => LabQcRule::RULES.index_with(true))
      stepped_up!(staff)
      json_put "/lab/qc/rules", instrument_id: s.instrument.id, analyte_key: "glicose", rules: LabQcRule::RULES.index_with(true), justification: nil
      expect(response).to have_http_status(:forbidden)
      stepped_up!(lead)
      off = LabQcRule::RULES.index_with(true).merge("10x" => false)
      json_put "/lab/qc/rules", instrument_id: s.instrument.id, analyte_key: "glicose", rules: off, justification: ""
      expect(json_body["error"]).to eq("justification_required")
      json_put "/lab/qc/rules", instrument_id: s.instrument.id, analyte_key: "glicose", rules: off.except("10x"), justification: "x"
      expect(json_body["error"]).to eq("invalid_rules")
      json_put "/lab/qc/rules", instrument_id: s.instrument.id, analyte_key: "glicose", rules: off, justification: "Lote novo em validação"
      expect(json_body).to include("is_default" => false, "justification" => "Lote novo em validação")
      expect(json_body["rules"]["10x"]).to be(false)
      # C16: religar não exige justificativa; manter desligada a mesma regra também não.
      json_put "/lab/qc/rules", instrument_id: s.instrument.id, analyte_key: "glicose", rules: off.merge("2_2s" => true), justification: nil
      expect(response).to have_http_status(:ok)
      json_put "/lab/qc/rules", instrument_id: s.instrument.id, analyte_key: "glicose", rules: LabQcRule::RULES.index_with(true), justification: nil
      expect(json_body["is_default"]).to be(false)
      expect(DomainEvent.where(name: "lab_qc_rule.changed").map { |e| e.payload.keys.sort }).to eq([ %w[lab_instrument_id lab_qc_rule_id] ])
    end

    it "quem não é do laboratório não grava valor" do
      stepped_up!(lab_releaser!(lab_provider!))
      post_value("Nível 1", "100")
      expect(response).to have_http_status(:forbidden)
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar:** `rspec spec/commands/lab/qc/record_value_spec.rb spec/requests/lab/qc_runs_spec.rb` → `NameError: Lab::Qc::RecordValue`.

- [ ] **Step 3: O comando** `app/commands/lab/qc/record_value.rb`:

  ```ruby
  # Valor de controle, digitado ou pela HL7 (spec §5.2–§5.7; ADR 0038):
  # trava o APARELHO (ordem: aparelho → requisição → itens; Desvio 10), acha
  # a corrida (a dada, a do dia não reprovada ou uma nova), a referência
  # vigente, calcula z e roda Westgard sobre a série e os outros níveis; grava
  # o valor (só acréscimo) com o lote de reagente em uso; piora a situação da
  # corrida; reprovação abre o bloqueio (e enfileira a revisão); corrida
  # aceita completa depois da ação corretiva fecha. Valor nunca em log/evento.
  module Lab
    module Qc
      module RecordValue
        OUTCOME_STATUS = { "ok" => "approved", "warning" => "warning", "reject" => "rejected" }.freeze

        module_function

        def call(instrument:, lot:, level:, analyte_key:, value:, source:, by: nil, receipt: nil, run_id: nil, run: nil, now: Time.current)
          material = lot.lab_qc_material
          return Result.fail(:invalid_instrument) unless instrument.active && instrument.exam_provider_id == material.exam_provider_id
          return Result.fail(:invalid_levels) unless material.levels.include?(level)

          entry = material.analyte_entry(analyte_key)
          return Result.fail(:no_target) unless entry

          result = nil
          # Savepoint próprio: dentro da transação da HL7, a recusa desfaz só o que é dela.
          ApplicationRecord.transaction(requires_new: true) do
            instrument.lock!
            lot.reload
            next result = Result.fail(:lot_closed) if lot.status == "closed"
            next result = Result.fail(:expired_lot) if lot.expired?(now.in_time_zone.to_date)

            target = LabQcTarget.current(lot_id: lot.id, level: level, analyte_key: analyte_key, instrument_id: instrument.id)
            next result = Result.fail(:no_target) unless target

            run = resolve_run(instrument, run, run_id, by, now)
            next result = run if run.is_a?(Result)

            z = ((value - target.mean) / target.sd).round(3, BigDecimal::ROUND_HALF_UP)
            evaluation = Westgard.evaluate(z: z, series: series(instrument, lot, level, analyte_key), peers: peers(run, lot, level, analyte_key),
                                           enabled: LabQcRule.enabled_for(instrument.id, analyte_key))
            row = LabQcValue.create!(lab_qc_run: run, lab_qc_lot: lot, level: level, analyte_key: analyte_key, value: value, source: source,
                                     lab_hl7_receipt: receipt, z_score: z, lab_qc_target: target, rules_fired: evaluation.rules_fired,
                                     outcome: evaluation.outcome, entered_by_user: by, entered_at: now,
                                     reagent_lot_id: LabSupplyInUse.current_lot_id(instrument_id: instrument.id, lab_exam_id: entry["lab_exam_id"]))
            changed = advance!(run, evaluation.outcome, now)
            DomainEvents.publish("lab_qc_run.evaluated", lab_qc_run_id: run.id, status: run.status) if changed

            notice = nil
            if evaluation.outcome == "reject"
              block = Blocks.open!(instrument, analyte_key, run, now)
            else
              block, notice = Blocks.try_close!(instrument: instrument, analyte_key: analyte_key, run: run, material: material, by: by, now: now)
            end
            result = Result.ok(value: row, run: run, block: block, notice: notice)
          end
          result
        end

        # A corrida dada (HL7: mesmo reprovada); run_id (não reprovada); a do dia não reprovada; ou nova.
        def resolve_run(instrument, run, run_id, by, now)
          return run if run

          if run_id
            found = LabQcRun.find_by(id: run_id, lab_instrument_id: instrument.id)
            return Result.fail(:not_found) unless found
            return Result.fail(:invalid_transition, details: { status: found.status }) if found.status == "rejected"

            return found
          end
          day = now.in_time_zone.to_date
          LabQcRun.where(lab_instrument_id: instrument.id, started_at: day.beginning_of_day..day.end_of_day).where.not(status: "rejected")
                  .order(:started_at, :id).last ||
            LabQcRun.create!(lab_instrument: instrument, sector: instrument.sector, status: "open", started_at: now, opened_by_user: by)
        end

        # Desvio 5: até 9 z anteriores, corridas não reprovadas, mais antigo primeiro.
        def series(instrument, lot, level, analyte_key)
          LabQcValue.joins(:lab_qc_run).where(lab_qc_runs: { lab_instrument_id: instrument.id }).where.not(lab_qc_runs: { status: "rejected" })
                    .where(lab_qc_lot_id: lot.id, level: level, analyte_key: analyte_key)
                    .order(entered_at: :desc, created_at: :desc).limit(9).pluck(:z_score).reverse
        end

        # O z mais recente de cada OUTRO nível do analito nesta corrida.
        def peers(run, lot, level, analyte_key)
          LabQcValue.where(lab_qc_run_id: run.id, lab_qc_lot_id: lot.id, analyte_key: analyte_key).where.not(level: level)
                    .order(:entered_at, :created_at).to_a.group_by(&:level).values.map { |rows| rows.last.z_score }
        end

        def advance!(run, outcome, now)
          wanted = OUTCOME_STATUS.fetch(outcome)
          status = LabQcRun::SEVERITY[wanted] > LabQcRun::SEVERITY[run.status] ? wanted : run.status
          changed = status != run.status
          run.update!(status: status, evaluated_at: now)
          changed
        end
        private_class_method :resolve_run, :series, :peers, :advance!
      end
    end
  end
  ```

`app/services/lab/qc/blocks.rb`:

  ```ruby
  # Bloqueio por aparelho e analito (spec §5.3–§5.7; Desvio 7). Chamado com o
  # aparelho travado (RecordValue). Abrir: um ativo por par (índice), revisão
  # enfileirada. Fechar: corrida aceita, depois da abertura, com o valor mais
  # recente de todos os níveis do material, nenhum reprovado e todos gravados
  # depois da ação corretiva; sem ação, devolve o aviso.
  module Lab
    module Qc
      module Blocks
        module_function

        def open!(instrument, analyte_key, run, now)
          active = LabQcBlock.active.find_by(lab_instrument_id: instrument.id, analyte_key: analyte_key)
          return active if active

          block = LabQcBlock.create!(lab_instrument: instrument, analyte_key: analyte_key, opened_by_run: run, opened_at: now)
          Reviews.enqueue!(block, run: run)
          DomainEvents.publish("lab_qc_block.opened", lab_qc_block_id: block.id, lab_instrument_id: instrument.id)
          block
        end

        def try_close!(instrument:, analyte_key:, run:, material:, by:, now:)
          block = LabQcBlock.active.find_by(lab_instrument_id: instrument.id, analyte_key: analyte_key)
          return [ nil, nil ] unless block
          return [ block, nil ] if run.id == block.opened_by_run_id || run.started_at < block.opened_at || !LabQcRun::ACCEPTED.include?(run.status)

          latest = LabQcValue.where(lab_qc_run_id: run.id, analyte_key: analyte_key, lab_qc_lot_id: material.lots.select(:id))
                             .order(:entered_at, :created_at).to_a.group_by(&:level).transform_values(&:last)
          return [ block, nil ] unless (material.levels - latest.keys).empty? && latest.values.none? { |v| v.outcome == "reject" }

          block.lock!
          return [ block, "corrective_action_required" ] if block.corrective_action.nil?
          return [ block, nil ] unless latest.values.all? { |v| v.entered_at >= block.corrective_action_at }

          block.update!(closed_by_run: run, closed_by_user: by, closed_at: now)
          DomainEvents.publish("lab_qc_block.closed", lab_qc_block_id: block.id, lab_instrument_id: instrument.id)
          [ block, nil ]
        end
      end
    end
  end
  ```

  `app/services/lab/qc/reviews.rb`:

  ```ruby
  # Revisão (spec §5.6; Desvio 8): na abertura do bloqueio, os itens do aparelho
  # cujo exame tem o analito e cujo laudo foi liberado depois da última corrida
  # aceita com o analito (ou, sem ela, da primeira referência aceita) entram
  # pendentes. Só leitura do 21b; cria linhas de revisão.
  module Lab
    module Qc
      module Reviews
        module_function

        def enqueue!(block, run:)
          since = since_for(block, run)
          versions = LabExamVersion.where(lab_instrument_id: block.lab_instrument_id)
                                   .where("analytes @> ?::jsonb", [ { key: block.analyte_key } ].to_json).select(:id)
          item_ids = LabReportItem.joins(:lab_report).where(lab_reports: { status: %w[released rectified] })
                                  .where("lab_reports.released_at > ?", since)
                                  .where(lab_requisition_item_id: LabRequisitionItem.where(lab_exam_version_id: versions).select(:id))
                                  .distinct.pluck(:lab_requisition_item_id).sort
          item_ids.each { |id| LabQcReviewItem.create!(lab_qc_block: block, lab_requisition_item_id: id) }
          item_ids.size
        end

        def since_for(block, run)
          accepted = LabQcRun.where(lab_instrument_id: block.lab_instrument_id, status: LabQcRun::ACCEPTED).where.not(id: run.id)
                             .where(id: LabQcValue.where(analyte_key: block.analyte_key).select(:lab_qc_run_id)).maximum(:evaluated_at)
          accepted || LabQcTarget.where(lab_instrument_id: block.lab_instrument_id, analyte_key: block.analyte_key).minimum(:accepted_at) || run.started_at
        end
      end
    end
  end
  ```

  `app/commands/lab/qc/open_run.rb`:

  ```ruby
  # Corrida aberta a pedido (spec §5.2): sempre nova; vira a "do dia" (a mais recente não reprovada).
  module Lab
    module Qc
      module OpenRun
        module_function

        def call(instrument:, by:, now: Time.current)
          ApplicationRecord.transaction(requires_new: true) do
            instrument.lock!
            Result.ok(run: LabQcRun.create!(lab_instrument: instrument, sector: instrument.sector, status: "open", started_at: now, opened_by_user: by))
          end
        end
      end
    end
  end
  ```

  `app/commands/lab/qc/save_rules.rb`:

  ```ruby
  # Regras por aparelho e analito (contrato §3; Desvio 3; C16): as seis
  # chaves, booleanas; desligar uma regra que estava ligada exige
  # justificativa (1..1000). Acrescenta uma versão; vale para os valores seguintes.
  module Lab
    module Qc
      module SaveRules
        KEY = /\A[a-z0-9_]{1,40}\z/

        module_function

        def call(instrument:, analyte_key:, rules:, justification:, by:, now: Time.current)
          rules = rules.to_unsafe_h if rules.respond_to?(:to_unsafe_h)
          valid = analyte_key.is_a?(String) && analyte_key.match?(KEY) && rules.is_a?(Hash) &&
                  rules.keys.map(&:to_s).sort == LabQcRule::RULES.sort && rules.values.all? { |v| [ true, false ].include?(v) }
          return Result.fail(:invalid_rules) unless valid

          rules = rules.transform_keys(&:to_s)
          text = justification.is_a?(String) ? justification.strip : ""
          before = LabQcRule.enabled_for(instrument.id, analyte_key)
          turning_off = rules.any? { |rule, on| on == false && before.include?(rule) }
          return Result.fail(:justification_required) if turning_off && !text.length.between?(1, 1000)
          return Result.fail(:justification_required) if text.length > 1000

          rule = LabQcRule.create!(lab_instrument: instrument, analyte_key: analyte_key, rules: rules, justification: text.presence,
                                   changed_by_user: by, created_at: now)
          DomainEvents.publish("lab_qc_rule.changed", lab_qc_rule_id: rule.id, lab_instrument_id: instrument.id)
          Result.ok(rule: rule)
        end
      end
    end
  end
  ```

  `app/commands/lab/qc/record_corrective_action.rb`:

  ```ruby
  # Ação corretiva do bloqueio (spec §5.7; C14): lab_staff ou liberador do
  # laboratório (quem chama confere), uma vez, texto 1..1000. Trava só o bloqueio (o fechamento, que
  # trava aparelho → bloqueio, nunca espera esta trava ao contrário).
  module Lab
    module Qc
      module RecordCorrectiveAction
        module_function

        def call(block:, text:, by:, now: Time.current)
          text = text.is_a?(String) ? text.strip : ""
          return Result.fail(:corrective_action_required) unless text.length.between?(1, 1000)

          ApplicationRecord.transaction(requires_new: true) do
            block.lock!
            next Result.fail(:invalid_transition, details: { status: "closed" }) if block.closed_at
            next Result.fail(:conflict) if block.corrective_action

            block.update!(corrective_action: text, corrective_action_by_user: by, corrective_action_at: now)
            DomainEvents.publish("lab_qc_block.action_recorded", lab_qc_block_id: block.id)
            Result.ok(block: block)
          end
        end
      end
    end
  end
  ```

- [ ] **Step 4: JSON e controllers.** Acrescente a `app/services/lab/qc/json.rb`:

  ```ruby
        def value(v)
          { id: v.id, run_id: v.lab_qc_run_id, lot_id: v.lab_qc_lot_id, level: v.level, analyte_key: v.analyte_key, value: out(v.value),
            z_score: v.z_score.to_f, rules_fired: v.rules_fired, outcome: v.outcome, source: v.source, target_version: v.lab_qc_target.version,
            reagent_lot_id: v.reagent_lot_id, reagent_lot: v.reagent_lot && { id: v.reagent_lot.id, lot_number: v.reagent_lot.lot_number },
            entered_at: at(v.entered_at), entered_by: Screenings::Json.staff_name(v.entered_by_user) }
        end

        def run(r)
          { id: r.id, instrument_id: r.lab_instrument_id, sector: r.sector, status: r.status, started_at: at(r.started_at),
            evaluated_at: at(r.evaluated_at), values: r.values.includes(:lab_qc_target, :entered_by_user, :reagent_lot).map { |v| value(v) } }
        end

        def block(b)
          { id: b.id, instrument_id: b.lab_instrument_id, analyte_key: b.analyte_key, opened_at: at(b.opened_at), opened_by_run_id: b.opened_by_run_id,
            corrective_action: b.corrective_action, corrective_action_at: at(b.corrective_action_at), closed_at: at(b.closed_at),
            closed_by_run_id: b.closed_by_run_id }
        end

        def rule(instrument_id, analyte_key, row)
          { instrument_id: instrument_id, analyte_key: analyte_key, is_default: row.nil?,
            rules: LabQcRule::RULES.index_with { |r| row ? row.rules[r] == true : true },
            justification: row&.justification, changed_by: row && Screenings::Json.staff_name(row.changed_by_user), changed_at: at(row&.created_at) }
        end
  ```

  `app/controllers/lab/qc/values_controller.rb`:

  ```ruby
  # POST /lab/qc/values (contrato §3): membro do laboratório do aparelho; valor com vírgula.
  module Lab
    module Qc
      class ValuesController < BaseController
        def create
          instrument = uuid?(params[:instrument_id]) && LabInstrument.find_by(id: params[:instrument_id])
          return not_found unless instrument
          return forbid unless Lab::Access.member?(Current.user, instrument.exam_provider_id)

          lot = uuid?(params[:lot_id]) && LabQcLot.find_by(id: params[:lot_id])
          return render_failure(Result.fail(:not_found)) unless lot

          value = Decimal.typed(params[:value])
          return render_failure(Result.fail(:invalid_value)) unless value

          run_id = params[:run_id].presence
          return render_failure(Result.fail(:not_found)) if run_id && !uuid?(run_id)

          result = RecordValue.call(instrument: instrument, lot: lot, level: params[:level].to_s, analyte_key: params[:analyte_key].to_s,
                                    value: value, source: "typed", by: Current.user, run_id: run_id)
          return render_failure(result) if result.failure?

          payload = result.payload
          body = { value: Json.value(payload[:value]), run: Json.run(payload[:run].reload), block: payload[:block] && Json.block(payload[:block].reload) }
          body[:notice] = payload[:notice] if payload[:notice]
          render json: body, status: :created
        end
      end
    end
  end
  ```

  `app/controllers/lab/qc/runs_controller.rb`:

  ```ruby
  # /lab/qc/runs (contrato §3): corridas do dia do aparelho (leitura, sem
  # trilha: nada de paciente); abrir corrida a pedido (membro).
  module Lab
    module Qc
      class RunsController < BaseController
        before_action :load_instrument

        def index
          return forbid unless reader?(@instrument.exam_provider_id)

          day = params[:date].to_s.match?(/\A\d{4}-\d{2}-\d{2}\z/) ? (Date.iso8601(params[:date]) rescue nil) : Time.zone.today
          return render_failure(Result.fail(:invalid_date)) unless day

          runs = LabQcRun.where(lab_instrument_id: @instrument.id, started_at: day.beginning_of_day..day.end_of_day).order(:started_at, :id)
          render json: { runs: runs.map { |r| Json.run(r) } }
        end

        def create
          return forbid unless Lab::Access.member?(Current.user, @instrument.exam_provider_id)

          render json: Json.run(OpenRun.call(instrument: @instrument, by: Current.user).payload[:run]), status: :created
        end

        private

        def load_instrument
          @instrument = uuid?(params[:instrument_id]) && LabInstrument.find_by(id: params[:instrument_id])
          not_found unless @instrument
        end
      end
    end
  end
  ```

  `app/controllers/lab/qc/rules_controller.rb`:

  ```ruby
  # /lab/qc/rules (contrato §3): leitura para membros e admin; mudar só RT/substituto, com step-up.
  module Lab
    module Qc
      class RulesController < BaseController
        before_action :require_step_up!, only: :update
        before_action :load_instrument

        def show
          return forbid unless reader?(@instrument.exam_provider_id)

          key = params[:analyte_key].to_s
          render json: Json.rule(@instrument.id, key, LabQcRule.current_for(@instrument.id, key))
        end

        def update
          return forbid unless technical?(@instrument.exam_provider_id)

          result = SaveRules.call(instrument: @instrument, analyte_key: params[:analyte_key], rules: params[:rules],
                                  justification: params[:justification], by: Current.user)
          return render_failure(result) if result.failure?

          rule = result.payload[:rule]
          render json: Json.rule(@instrument.id, rule.analyte_key, rule)
        end

        private

        def load_instrument
          @instrument = uuid?(params[:instrument_id]) && LabInstrument.find_by(id: params[:instrument_id])
          not_found unless @instrument
        end
      end
    end
  end
  ```

  `app/controllers/lab/qc/blocks_controller.rb`:

  ```ruby
  # /lab/qc/blocks (contrato §3): lista por laboratório (ativos ou fechados);
  # contador sem trilha nos laboratórios da sessão (molde A46/D11; C11); ação
  # corretiva de quem opera o aparelho (C14: lab_staff ou liberador do laboratório).
  module Lab
    module Qc
      class BlocksController < BaseController
        before_action -> { provider! }, only: :index

        def index
          scope = LabQcBlock.joins(:lab_instrument).where(lab_instruments: { exam_provider_id: @provider.id })
          scope = params[:status] == "closed" ? scope.where.not(closed_at: nil) : scope.where(closed_at: nil)
          render json: { blocks: scope.order(opened_at: :desc).limit(200).map { |b| Json.block(b) } }
        end

        def count
          ids = Lab::Access.admin?(Current.user) ? Lab::Access.labs.pluck(:id) : Lab::Access.provider_ids(Current.user)
          render json: { count: LabQcBlock.active.joins(:lab_instrument).where(lab_instruments: { exam_provider_id: ids }).count }
        end

        def corrective_action
          block = uuid?(params[:id]) && LabQcBlock.find_by(id: params[:id])
          return not_found unless block
          return forbid unless Lab::Access.member?(Current.user, block.lab_instrument.exam_provider_id)

          result = RecordCorrectiveAction.call(block: block, text: params[:text], by: Current.user)
          return render_failure(result) if result.failure?

          render json: Json.block(result.payload[:block])
        end
      end
    end
  end
  ```

  `config/routes.rb` — dentro de `scope "qc"`:

  ```ruby
        get   "runs",                 to: "runs#index"
        post  "runs",                 to: "runs#create"
        post  "values",               to: "values#create"
        get   "rules",                to: "rules#show"
        put   "rules",                to: "rules#update"
        get   "blocks/active/count",  to: "blocks#count"
        get   "blocks",               to: "blocks#index"
        post  "blocks/:id/corrective_action", to: "blocks#corrective_action"
  ```

- [ ] **Step 5: Rode:** `rspec spec/commands/lab/qc/record_value_spec.rb spec/requests/lab/qc_runs_spec.rb spec/services/lab/qc/westgard_spec.rb` → verde.

- [ ] **Step 6: Commit** e mova o card F-21.27 para **Done**:

  ```bash
  git add app/commands/lab/qc/record_value.rb app/commands/lab/qc/open_run.rb app/commands/lab/qc/save_rules.rb app/commands/lab/qc/record_corrective_action.rb app/services/lab/qc/blocks.rb app/services/lab/qc/reviews.rb app/services/lab/qc/json.rb app/controllers/lab/qc/values_controller.rb app/controllers/lab/qc/runs_controller.rb app/controllers/lab/qc/rules_controller.rb app/controllers/lab/qc/blocks_controller.rb config/routes.rb spec/commands/lab/qc/record_value_spec.rb spec/requests/lab/qc_runs_spec.rb
  git commit -m "feat: evaluate quality control values on write and open release blocks

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 8: Bloqueio na liberação do 21b, exceção do RT e revisão (F-21.29)

**Files:**
- Create: `app/services/lab/qc/release_check.rb`, `app/commands/lab/qc/{grant_exception,decide_review}.rb`, `app/controllers/lab/qc/{exceptions_controller,reviews_controller}.rb`, `spec/requests/lab/qc_release_block_spec.rb`, `spec/commands/lab/qc_block_release_concurrency_spec.rb`
- Modify: `app/commands/lab/release.rb`, `app/controllers/lab/base_controller.rb`, `app/services/lab/qc/json.rb`, `config/routes.rb`

**Interfaces:**
- Consumes: `LabQcBlock.active`, `LabQcException`, `LabQcReviewItem` (Task 3); `Lab::Qc::Reviews.enqueue!`, `Lab::Qc::RecordValue`, `Lab::Qc::RecordCorrectiveAction`, `Lab::Qc::OpenRun` (Task 7); `Lab::Release` (21b).
- Produces: `Lab::Qc::ReleaseCheck.lock_instruments!(item_ids)` (FOR SHARE, por id) e `.blocked(items) → [{ item_id:, block_id:, analyte_key:, instrument_id:, instrument_name: }]` (C9); `Lab::Release` recusa `:qc_blocked` com `details: { items: [...] }` (409 com `items` no corpo); `Lab::Qc::GrantException.call(block_id:, item_id:, justification:, by:, now:) → Result(exception:)`; `Lab::Qc::DecideReview.call(review:, decision:, note:, by:, now:) → Result(review:, report_id:)`; `Lab::Qc::Json.{exception,review}`. Rotas `GET|POST /lab/qc/exceptions`, `GET /lab/qc/reviews`, `POST /lab/qc/reviews/:id`.

- [ ] **Step 1: Testes que falham.** `spec/requests/lab/qc_release_block_spec.rb`:

  ```ruby
  require "rails_helper"

  # Spec §5.4–§5.6 e contrato §3 (F-21.29): com bloqueio ativo, o item do
  # analito no aparelho não é liberado (409 qc_blocked, com os itens); o RT ou
  # substituto libera exceção com justificativa (a liberação segue normal); o
  # que saiu desde a última corrida aceita vai para revisão; a exceção é do
  # bloqueio, não atravessa para o próximo; interruptor desligado não bloqueia.
  RSpec.describe "CIQ — liberação bloqueada, exceção e revisão", type: :request do
    include ActiveSupport::Testing::TimeHelpers

    before do
      travel_to(Time.zone.now.change(hour: 12))
      qc_city!
    end

    let(:s) { qc_scenario! }
    let!(:lead) { lab_releaser!(s.provider, council: "CRBM", lead: true) }
    let(:staff) { lab_staff! }

    # Tempo parado: um segundo entre valores (ordem da série) e 31 s entre liberações (TOTP novo).
    def qc!(level, z)
      travel 1.second
      qc_value!(s, level, z, by: staff)
    end

    def typed_item!
      req = received_requisition!(provider: s.provider, exams: [ s.exam ])
      type!(req.items.first, { glicose: "98" })
    end

    def release(item, user: lead)
      travel 31.seconds
      sign_in_as(user)
      json_post "/lab/requisitions/#{item.lab_requisition_id}/release", item_ids: [ item.id ], signature: "advanced", totp: totp_for(user)
    end

    it "reprovação bloqueia; exceção do RT libera; o já liberado entra na revisão; decisão uma vez" do
      qc!("Nível 1", 0.2)
      qc!("Nível 2", -0.3)
      first = typed_item!
      release(first)
      expect(response).to have_http_status(:created)
      report_id = json_body["id"]
      second = typed_item!
      Lab::Qc::OpenRun.call(instrument: s.instrument, by: staff)
      qc!("Nível 1", 3.6)
      block = LabQcBlock.active.sole
      expect(LabQcReviewItem.where(lab_qc_block: block).pluck(:lab_requisition_item_id, :decision)).to eq([ [ first.id, "pending" ] ])

      release(second)
      expect(response).to have_http_status(:conflict)
      expect(json_body).to eq("error" => "qc_blocked",
                              "items" => [ { "item_id" => second.id, "block_id" => block.id, "analyte_key" => "glicose", "instrument_id" => s.instrument.id,
                                             "instrument_name" => "Analisador bioquímico 1" } ])
      expect(second.reload.status).to eq("typed")

      stepped_up!(staff)
      json_post "/lab/qc/exceptions", block_id: block.id, item_id: second.id, justification: "Gestante em trabalho de parto"
      expect(response).to have_http_status(:forbidden)
      stepped_up!(lead)
      json_post "/lab/qc/exceptions", block_id: block.id, item_id: second.id, justification: " "
      expect(json_body["error"]).to eq("justification_required")
      json_post "/lab/qc/exceptions", block_id: block.id, item_id: second.id, justification: "Gestante em trabalho de parto"
      expect(response).to have_http_status(:created)
      expect(json_body).to include("block_id" => block.id, "item_id" => second.id, "justification" => "Gestante em trabalho de parto",
                                   "requisition_number" => second.lab_requisition.number, "sigtap_code" => "0202010473")
      json_post "/lab/qc/exceptions", block_id: block.id, item_id: second.id, justification: "De novo"
      expect(response).to have_http_status(:conflict)
      release(second)
      expect(response).to have_http_status(:created)

      stepped_up!(lead)
      get "/lab/qc/exceptions", params: { provider_id: s.provider.id, month: Time.zone.today.strftime("%Y-%m") }
      expect(json_body["exceptions"].map { |e| e["item_id"] }).to eq([ second.id ])
      get "/lab/qc/exceptions", params: { provider_id: s.provider.id, month: (Time.zone.today << 1).strftime("%Y-%m") }
      expect(json_body["exceptions"]).to eq([])
      get "/lab/qc/reviews", params: { provider_id: s.provider.id }
      review = json_body["reviews"].sole
      expect(review).to include("item_id" => first.id, "decision" => "pending", "analyte_key" => "glicose", "requisition_number" => first.lab_requisition.number,
                                "requisition_id" => first.lab_requisition_id, "exam_label" => "Glicose")
      json_post "/lab/qc/reviews/#{review['id']}", decision: "maybe"
      expect(json_body["error"]).to eq("invalid_decision")
      json_post "/lab/qc/reviews/#{review['id']}", decision: "rectify", note: "Repetir a dosagem"
      expect(json_body).to include("decision" => "rectify", "note" => "Repetir a dosagem", "report_id" => report_id)
      json_post "/lab/qc/reviews/#{review['id']}", decision: "kept"
      expect(response).to have_http_status(:conflict)
      payloads = DomainEvent.where("name LIKE 'lab_qc_%'").pluck(:payload).to_json
      expect(payloads).not_to include("Gestante", "Repetir")
      expect(DomainEvent.where(name: "lab_qc_review.decided").map(&:payload).sole.keys.sort).to eq(%w[decision lab_qc_review_item_id])
    end

    it "exceção não atravessa bloqueio: fechado e reaberto, o item volta a ser recusado com o bloqueio novo" do
      qc!("Nível 1", 3.6)
      old_block = LabQcBlock.active.sole
      item = typed_item!
      expect(Lab::Qc::GrantException.call(block_id: old_block.id, item_id: item.id, justification: "Urgência", by: lead)).to be_ok
      travel 1.second
      Lab::Qc::RecordCorrectiveAction.call(block: old_block, text: "Calibração refeita", by: lead)
      qc!("Nível 1", 0.1)
      qc!("Nível 2", 0.2)
      expect(old_block.reload.closed_at).to be_present
      Lab::Qc::OpenRun.call(instrument: s.instrument, by: staff)
      qc!("Nível 2", -3.4)
      new_block = LabQcBlock.active.sole
      release(item)
      expect(response).to have_http_status(:conflict)
      expect(json_body["items"].map { |i| i["block_id"] }).to eq([ new_block.id ])
    end

    it "exceção só para item do aparelho e do analito, ainda não liberado" do
      qc!("Nível 1", 3.6)
      block = LabQcBlock.active.sole
      untracked = lab_exam!(provider: s.provider, sigtap_code: "0202010317", name: "Creatinina", instrument_test_code: "CRE",
                            analytes: [ LabHelpers::GLUCOSE.first.merge("key" => "creatinina", "name" => "Creatinina", "decimals" => 2, "entry_min" => nil,
                                                                        "entry_max" => nil, "critical_low" => nil, "critical_high" => nil) ])
      req = received_requisition!(provider: s.provider, exams: [ untracked ])
      item = type!(req.items.first, { creatinina: "0,9" })
      expect(Lab::Qc::GrantException.call(block_id: block.id, item_id: item.id, justification: "x", by: lead).reason).to eq(:not_blocked)
      release(item)
      expect(response).to have_http_status(:created) # sem aparelho no exame: nunca bloqueado
    end

    it "com lab_quality_control desligado o bloqueio não é consultado" do
      qc!("Nível 1", 3.6)
      item = typed_item!
      Platform::Features.set!(city: City.find(Current.city.id), key: "lab_quality_control", enabled: false, maintainer: ledi_maintainer!)
      release(item)
      expect(response).to have_http_status(:created)
    end
  end
  ```

  `spec/commands/lab/qc_block_release_concurrency_spec.rb` — **corrida real entre o bloqueio e a liberação** (molde de `release_concurrency_spec.rb`):

  ```ruby
  require "rails_helper"

  # Desvio 10 / Review: a reprovação do CIQ e a liberação do mesmo analito no
  # mesmo aparelho ao mesmo tempo, nas duas ordens. Aparelho FOR UPDATE (CIQ)
  # × FOR SHARE (liberação, antes da requisição): sem deadlock e sem buraco —
  # ou o item não sai (qc_blocked), ou sai e entra na revisão do bloqueio.
  RSpec.describe "Bloqueio do CIQ × liberação sob disputa" do
    include LabQcHelpers

    self.use_transactional_tests = false

    let(:release_gate) { Queue.new }
    let(:threads) { [] }

    def in_city(&) = CityConnection.with(TEST_CITY_A) { Current.set(city: TEST_CITY_A, &) }

    def empty_tables
      connection = ApplicationRecord.connection
      connection.tables.reject { |t| %w[schema_migrations ar_internal_metadata].include?(t) }
                .select { |t| connection.select_value("SELECT NOT EXISTS (SELECT 1 FROM #{connection.quote_table_name(t)})") }
    end

    before do
      allow(ClinicalTerms::SigtapExams).to receive(:find).and_return(Struct.new(:competence).new(Time.zone.today.strftime("%Y%m")))
      # Sem fixture transacional a plataforma não é tocada: o interruptor vem de um dublê.
      allow(Lab::Gate).to receive(:qc_usable?).and_return(true)
      @rows = in_city do
        @empty = empty_tables
        provider = lab_provider!
        lead = lab_releaser!(provider, lead: true)
        scenario = qc_scenario!(provider: provider)
        staff = lab_staff!
        qc_value!(scenario, "Nível 1", 0.1, by: staff, now: 2.minutes.ago)
        qc_value!(scenario, "Nível 2", 0.2, by: staff, now: 2.minutes.ago + 1)
        Lab::Qc::OpenRun.call(instrument: scenario.instrument, by: staff, now: 1.minute.ago)
        req = received_requisition!(provider: provider, exams: [ scenario.exam ])
        item = type!(req.items.first, { glicose: "98" }, by: staff)
        { req_id: req.id, item_id: item.id, lead: lead.id, staff: staff.id, instrument_id: scenario.instrument.id, lot_id: scenario.lot.id }
      end
    end

    after do
      3.times { release_gate << true }
      threads.each do |thread|
        thread.join(5) || thread.kill
      rescue StandardError
        nil
      end
      in_city do
        ApplicationRecord.transaction do
          connection = ApplicationRecord.connection
          connection.execute("SET LOCAL session_replication_role = replica")
          @empty.to_a.each { |table| connection.execute("DELETE FROM #{connection.quote_table_name(table)}") }
        end
      end
    end

    def release
      user = User.find(@rows[:lead])
      Lab::Release.call(requisition: LabRequisition.find(@rows[:req_id]), item_ids: [ @rows[:item_id] ], signature: "advanced",
                        totp: ROTP::TOTP.new(user.otp_secret).now, by: user)
    end

    def reject
      Lab::Qc::RecordValue.call(instrument: LabInstrument.find(@rows[:instrument_id]), lot: LabQcLot.find(@rows[:lot_id]), level: "Nível 1",
                                analyte_key: "glicose", value: BigDecimal("118"), source: "typed", by: User.find(@rows[:staff]))
    end

    # A primeira segura a transação aberta até o release_gate; a segunda espera a trava.
    def race(first, second)
      holding = Queue.new
      first_out = Queue.new
      threads << Thread.new do
        in_city do
          ApplicationRecord.transaction do
            first_out << first.call
            holding << true
            release_gate.pop(timeout: 10) or raise "timeout"
          end
        end
      end
      holding.pop(timeout: 10) or raise "timeout"
      second_out = Queue.new
      threads << Thread.new { in_city { second_out << second.call } }
      sleep 0.3
      release_gate << true
      [ first_out.pop(timeout: 10), second_out.pop(timeout: 10) ]
    end

    it "reprovação primeiro: a liberação espera o aparelho e leva qc_blocked" do
      first, second = race(-> { reject }, -> { release })
      expect(first).to be_ok
      expect(first.payload[:block]).to be_present
      expect(second.reason).to eq(:qc_blocked)
      expect(second.details[:items].map { |i| i[:item_id] }).to eq([ @rows[:item_id] ])
      in_city do
        expect(LabRequisitionItem.find(@rows[:item_id]).status).to eq("typed")
        expect(LabReportItem.where(lab_requisition_item_id: @rows[:item_id])).to be_empty
      end
    end

    it "liberação primeiro: a reprovação espera, o item sai e entra na revisão do bloqueio" do
      first, second = race(-> { release }, -> { reject })
      expect(first).to be_ok
      expect(second).to be_ok
      in_city do
        block = LabQcBlock.active.sole
        expect(LabRequisitionItem.find(@rows[:item_id]).status).to eq("released")
        expect(LabQcReviewItem.where(lab_qc_block: block).pluck(:lab_requisition_item_id)).to eq([ @rows[:item_id] ])
      end
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar:** `rspec spec/requests/lab/qc_release_block_spec.rb spec/commands/lab/qc_block_release_concurrency_spec.rb` → a liberação passa (201/ok) onde o teste espera 409/`qc_blocked`.

- [ ] **Step 3: A conferência** `app/services/lab/qc/release_check.rb`:

  ```ruby
  # Bloqueio do CIQ na liberação do 21b (spec §5.4; ADR 0038 Invariantes):
  # item cujo exame (versão) está ligado a um aparelho com bloqueio ativo de
  # um analito da versão, sem exceção daquele bloqueio para o item, não sai.
  # A trava FOR SHARE dos aparelhos vem ANTES da requisição (Desvio 10): a
  # reprovação concorrente (FOR UPDATE no aparelho) espera ou é esperada.
  module Lab
    module Qc
      module ReleaseCheck
        module_function

        def lock_instruments!(item_ids)
          ids = LabRequisitionItem.where(id: item_ids).joins(:lab_exam_version).where.not(lab_exam_versions: { lab_instrument_id: nil })
                                  .distinct.pluck("lab_exam_versions.lab_instrument_id").sort
          LabInstrument.where(id: ids).order(:id).lock("FOR SHARE").to_a if ids.any?
        end

        # items com lab_exam_version carregada; chame depois de lock_instruments!.
        def blocked(items)
          pairs = items.filter_map { |item| [ item, item.lab_exam_version ] if item.lab_exam_version.lab_instrument_id }
          return [] if pairs.empty?

          blocks = LabQcBlock.active.includes(:lab_instrument).where(lab_instrument_id: pairs.map { |_, v| v.lab_instrument_id }.uniq).to_a
          return [] if blocks.empty?

          excepted = LabQcException.where(lab_qc_block_id: blocks.map(&:id), lab_requisition_item_id: items.map(&:id))
                                   .pluck(:lab_qc_block_id, :lab_requisition_item_id).to_set
          pairs.flat_map do |item, version|
            blocks.select { |b| b.lab_instrument_id == version.lab_instrument_id && version.analyte(b.analyte_key) && !excepted.include?([ b.id, item.id ]) }
                  .map do |b|
                    { item_id: item.id, block_id: b.id, analyte_key: b.analyte_key, instrument_id: b.lab_instrument_id,
                      instrument_name: b.lab_instrument.name } # C9
                  end
          end.sort_by { |h| [ h[:item_id], h[:analyte_key] ] }
        end
      end
    end
  end
  ```

- [ ] **Step 4: A liberação.** Em `app/commands/lab/release.rb`:
  - no comentário de topo, acrescente: "CIQ (ADR 0038): com `lab_quality_control` utilizável (lido fora da transação), os aparelhos dos itens são travados FOR SHARE antes da requisição e o item com bloqueio ativo sem exceção é recusado (`qc_blocked`, com os itens) depois da conferência dos itens e antes do RT; na retificação só os itens que mudaram (`item_ids`) são conferidos."
  - logo depois de `return Result.fail(:invalid_transition) unless ids.size.between?(1, MAX_ITEMS)`:

    ```ruby
        qc = Lab::Gate.qc_usable?(Current.city) # fora da transação (Platform::Features)
    ```
  - dentro do `ApplicationRecord.transaction(requires_new: true) do`, **antes** de `requisition.lock!`:

    ```ruby
          Qc::ReleaseCheck.lock_instruments!(ids) if qc # aparelho → requisição → itens (Desvio 10)
    ```
  - logo depois de `next result = refusal if refusal`:

    ```ruby
          if qc
            blocked = Qc::ReleaseCheck.blocked(items.select { |item| ids.include?(item.id) })
            next result = Result.fail(:qc_blocked, details: { items: blocked }) if blocked.any?
          end
    ```

  Em `app/controllers/lab/base_controller.rb`: em `FAILURE_STATUS` acrescente `qc_blocked: :conflict,`; em `render_failure`, depois da linha do `rectify_items_mismatch`:

  ```ruby
      payload[:items] = result.details[:items] if result.reason == :qc_blocked
  ```

- [ ] **Step 5: Exceção e revisão.** `app/commands/lab/qc/grant_exception.rb`:

  ```ruby
  # Exceção do RT ou substituto (spec §5.5; contrato §3): justificativa
  # (1..1000, cifrada); o item é do laboratório, ainda não liberado, do
  # aparelho e de um exame com o analito do bloqueio ativo (senão
  # not_blocked). Uma por bloqueio e item (409). A liberação segue normal.
  module Lab
    module Qc
      module GrantException
        module_function

        def call(block_id:, item_id:, justification:, by:, now: Time.current)
          text = justification.is_a?(String) ? justification.strip : ""
          return Result.fail(:justification_required) unless text.length.between?(1, 1000)

          block = uuid?(block_id) && LabQcBlock.find_by(id: block_id)
          return Result.fail(:not_found) unless block

          provider_id = block.lab_instrument.exam_provider_id
          return Result.fail(:missing_role) unless Lab::Access.technical?(by, provider_id)

          item = uuid?(item_id) && LabRequisitionItem.includes(:lab_requisition, :lab_exam_version).find_by(id: item_id)
          return Result.fail(:not_found) unless item && item.lab_requisition.exam_provider_id == provider_id

          ApplicationRecord.transaction(requires_new: true) do
            block.lock!
            version = item.lab_exam_version
            covered = block.closed_at.nil? && version.lab_instrument_id == block.lab_instrument_id && version.analyte(block.analyte_key) &&
                      %w[pending typed].include?(item.reload.status)
            next Result.fail(:not_blocked) unless covered
            next Result.fail(:conflict) if LabQcException.exists?(lab_qc_block_id: block.id, lab_requisition_item_id: item.id)

            exception = LabQcException.create!(lab_qc_block: block, lab_requisition_item: item, justification: text, granted_by_user: by,
                                               granted_at: now, created_at: now)
            DomainEvents.publish("lab_qc_exception.granted", lab_qc_exception_id: exception.id, lab_qc_block_id: block.id,
                                                             lab_requisition_item_id: item.id)
            Result.ok(exception: exception)
          end
        end

        def uuid?(value) = value.is_a?(String) && value.match?(Exams::BaseController::UUID)
        private_class_method :uuid?
      end
    end
  end
  ```

  `app/commands/lab/qc/decide_review.rb`:

  ```ruby
  # Revisão (spec §5.6; contrato §3): RT ou substituto decide uma vez `kept`
  # ou `rectify` (nota 1..1000 opcional, cifrada). `rectify` devolve o laudo
  # liberado mais recente do item; a retificação é a do 21b.
  module Lab
    module Qc
      module DecideReview
        DECISIONS = %w[kept rectify].freeze

        module_function

        def call(review:, decision:, note:, by:, now: Time.current)
          return Result.fail(:invalid_decision) unless DECISIONS.include?(decision)

          text = note.is_a?(String) ? note.strip.presence : nil
          return Result.fail(:invalid_note) unless note.nil? || note.is_a?(String)
          return Result.fail(:invalid_note) if text && text.length > 1000
          return Result.fail(:missing_role) unless Lab::Access.technical?(by, review.lab_qc_block.lab_instrument.exam_provider_id)

          ApplicationRecord.transaction(requires_new: true) do
            review.lock!
            next Result.fail(:invalid_transition, details: { status: review.decision }) unless review.decision == "pending"

            review.update!(decision: decision, decided_by_user: by, decided_at: now, note: text)
            DomainEvents.publish("lab_qc_review.decided", lab_qc_review_item_id: review.id, decision: decision)
            report_id = if decision == "rectify"
                          LabReport.joins(:report_items).where(status: "released", lab_report_items: { lab_requisition_item_id: review.lab_requisition_item_id })
                                   .order(:version).last&.id
                        end
            Result.ok(review: review, report_id: report_id)
          end
        end
      end
    end
  end
  ```

  Acrescente a `app/services/lab/qc/json.rb`:

  ```ruby
        def item_ref(item)
          { item_id: item.id, requisition_id: item.lab_requisition_id, requisition_number: item.lab_requisition.number,
            sigtap_code: item.lab_exam_version.lab_exam.sigtap_code, exam_label: item.lab_exam_version.lab_exam.name } # C3
        end

        def exception(e)
          block = e.lab_qc_block
          { id: e.id, block_id: block.id, analyte_key: block.analyte_key, instrument_id: block.lab_instrument_id,
            justification: e.justification, granted_by: Screenings::Json.staff_name(e.granted_by_user), granted_at: at(e.granted_at) }
            .merge(item_ref(e.lab_requisition_item))
        end

        def review(r, report_id: nil)
          block = r.lab_qc_block
          released_at = LabReport.joins(:report_items).where(status: %w[released rectified], lab_report_items: { lab_requisition_item_id: r.lab_requisition_item_id })
                                 .maximum(:released_at)
          body = { id: r.id, block_id: block.id, analyte_key: block.analyte_key, instrument_id: block.lab_instrument_id, released_at: at(released_at),
                   decision: r.decision, decided_by: r.decided_by_user && Screenings::Json.staff_name(r.decided_by_user),
                   decided_at: at(r.decided_at), note: r.note }.merge(item_ref(r.lab_requisition_item))
          body[:report_id] = report_id if r.decision == "rectify"
          body
        end
  ```

  `app/controllers/lab/qc/exceptions_controller.rb`:

  ```ruby
  # /lab/qc/exceptions (contrato §3): conceder (RT/substituto, step-up); lista
  # por laboratório para membros e admin (a justificativa é do laboratório).
  module Lab
    module Qc
      class ExceptionsController < BaseController
        PRELOAD = [ :granted_by_user, :lab_qc_block, { lab_requisition_item: [ :lab_requisition, { lab_exam_version: :lab_exam } ] } ].freeze

        before_action :require_step_up!, only: :create
        before_action -> { provider! }, only: :index

        def index
          rows = LabQcException.joins(lab_qc_block: :lab_instrument).where(lab_instruments: { exam_provider_id: @provider.id })
          if params[:month].to_s.match?(/\A\d{4}-(0[1-9]|1[0-2])\z/) # C7: relatório do mês
            first = Date.parse("#{params[:month]}-01")
            rows = rows.where(granted_at: first.beginning_of_day..first.end_of_month.end_of_day)
          end
          rows = rows.includes(*PRELOAD).order(granted_at: :desc).limit(500)
          render json: { exceptions: rows.map { |e| Json.exception(e) } }
        end

        def create
          result = GrantException.call(block_id: params[:block_id], item_id: params[:item_id], justification: params[:justification], by: Current.user)
          return render_failure(result) if result.failure?

          render json: Json.exception(result.payload[:exception]), status: :created
        end
      end
    end
  end
  ```

  `app/controllers/lab/qc/reviews_controller.rb`:

  ```ruby
  # /lab/qc/reviews (contrato §3): pendentes (ou decididas) por laboratório;
  # decidir: RT/substituto, step-up.
  module Lab
    module Qc
      class ReviewsController < BaseController
        PRELOAD = [ :decided_by_user, :lab_qc_block, { lab_requisition_item: [ :lab_requisition, { lab_exam_version: :lab_exam } ] } ].freeze

        before_action :require_step_up!, only: :decide
        before_action -> { provider! }, only: :index

        def index
          rows = LabQcReviewItem.joins(lab_qc_block: :lab_instrument).where(lab_instruments: { exam_provider_id: @provider.id })
          rows = params[:status] == "decided" ? rows.where.not(decision: "pending") : rows.where(decision: "pending")
          render json: { reviews: rows.includes(*PRELOAD).order(:created_at, :id).limit(500).map { |r| Json.review(r) } }
        end

        def decide
          review = uuid?(params[:id]) && LabQcReviewItem.find_by(id: params[:id])
          return not_found unless review

          result = DecideReview.call(review: review, decision: params[:decision].to_s, note: params[:note], by: Current.user)
          return render_failure(result) if result.failure?

          render json: Json.review(result.payload[:review], report_id: result.payload[:report_id])
        end
      end
    end
  end
  ```

  `config/routes.rb` — dentro de `scope "qc"`:

  ```ruby
        get   "exceptions",           to: "exceptions#index"
        post  "exceptions",           to: "exceptions#create"
        get   "reviews",              to: "reviews#index"
        post  "reviews/:id",          to: "reviews#decide"
  ```

- [ ] **Step 6: Rode:** `rspec spec/requests/lab/qc_release_block_spec.rb spec/commands/lab/qc_block_release_concurrency_spec.rb spec/requests/lab/release_spec.rb spec/commands/lab/release_concurrency_spec.rb spec/requests/lab/rectification_spec.rb` → verde (as specs do 21b provam que a liberação sem CIQ não mudou).

- [ ] **Step 7: Commit** e mova o card F-21.29 para **Done**:

  ```bash
  git add app/services/lab/qc/release_check.rb app/commands/lab/qc/grant_exception.rb app/commands/lab/qc/decide_review.rb app/controllers/lab/qc/exceptions_controller.rb app/controllers/lab/qc/reviews_controller.rb app/commands/lab/release.rb app/controllers/lab/base_controller.rb app/services/lab/qc/json.rb config/routes.rb spec/requests/lab/qc_release_block_spec.rb spec/commands/lab/qc_block_release_concurrency_spec.rb
  git commit -m "feat: block lab release on failed quality control with technical lead exceptions and reviews

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 9: Sugestão da referência do laboratório, aceite do RT e Levey-Jennings (F-21.26, F-21.30)

**Files:**
- Create: `app/services/lab/qc/{suggestions,chart}.rb`, `app/commands/lab/qc/accept_suggestion.rb`, `app/controllers/lab/qc/charts_controller.rb`, `spec/requests/lab/qc_targets_chart_spec.rb`
- Modify: `app/controllers/lab/qc/targets_controller.rb`, `config/routes.rb`

**Interfaces:**
- Consumes: `LabQcTarget.current`, `LabQcRun::ACCEPTED` (Task 3); `Lab::Qc::CreateTarget.next_version` (Task 5); `Lab::Qc::RecordValue` (Task 7).
- Produces: `Lab::Qc::Suggestions::MIN_RUNS = 20`; `Lab::Qc::Suggestions.list(provider_id:) → [Suggestion]`, `.find(lot_id:, level:, analyte_key:, instrument_id:) → Suggestion|nil`, `Suggestion = Data.define(:lot_id, :level, :analyte_key, :instrument_id, :mean, :sd, :runs, :current_version, :current_source)` (média e dp amostral, 4 casas, do **último valor aceito de cada corrida aceita** gravado depois da referência vigente); `Lab::Qc::AcceptSuggestion.call(lot_id:, level:, analyte_key:, instrument_id:, reason:, by:, now:) → Result(target:)` (C6: `reason` obrigatório); `Lab::Qc::Chart.call(lot:, level:, analyte_key:, instrument:, from:, to:) → { target:, points: }`. Rotas `GET /lab/qc/targets/suggestions?provider_id=`, `POST /lab/qc/targets/suggestions/accept`, `GET /lab/qc/chart`.

- [ ] **Step 1: Teste que falha** — `spec/requests/lab/qc_targets_chart_spec.rb`:

  ```ruby
  require "rails_helper"

  # Spec §5.8–§5.9 (F-21.26, F-21.30): com 20 corridas aceitas no lote (por
  # nível, analito e aparelho) depois da referência vigente, o sistema sugere
  # média e dp; o RT aceita e vira nova versão `laboratory`. Levey-Jennings
  # com a versão de cada ponto (Review Focus 1: referência trocada no meio da
  # corrida não recalcula o que já entrou).
  RSpec.describe "CIQ — referência do laboratório e Levey-Jennings", type: :request do
    include ActiveSupport::Testing::TimeHelpers

    before do
      travel_to(Time.zone.now.change(hour: 12))
      qc_city!
    end

    let(:s) { qc_scenario! } # bula aceita há 30 dias
    let(:lead) { lab_releaser!(s.provider, lead: true) }
    let(:staff) { lab_staff! }
    # z alternando de lado: nenhuma regra dispara na história.
    let(:offsets) { [ 0.3, -0.5, 0.8, -0.2, 0.6, -0.9, 0.4, -0.3, 1.1, -0.6, 0.2, -0.4, 0.7, -0.8, 0.5, -0.1, 0.9, -0.7, 0.3, -0.2 ] }

    # Uma corrida por dia (dias diferentes = corridas diferentes), os dois níveis.
    def run_on!(day, offset)
      qc_value!(s, "Nível 1", offset, by: staff, now: day)
      qc_value!(s, "Nível 2", -offset, by: staff, now: day + 1)
    end

    def expected(level, offsets)
      values = offsets.map { |o| qc_glucose(level, o) }
      mean = values.sum / values.size
      sd = (values.sum { |v| (v - mean)**2 } / (values.size - 1)).sqrt(20)
      [ Lab::Qc::Decimal.out(mean.round(4)), Lab::Qc::Decimal.out(sd.round(4)) ]
    end

    it "19 corridas: nada; 20: sugestão por nível; o RT aceita; a sugestão aceita some" do
      offsets.first(19).each_with_index { |o, i| run_on!(Time.current - (20 - i).days, o) }
      stepped_up!(lead)
      get "/lab/qc/targets/suggestions", params: { provider_id: s.provider.id }
      expect(json_body["suggestions"]).to eq([])
      run_on!(Time.current - 1.hour, offsets.last)
      get "/lab/qc/targets/suggestions", params: { provider_id: s.provider.id }
      level1 = json_body["suggestions"].find { |x| x["level"] == "Nível 1" }
      mean, sd = expected("Nível 1", offsets)
      expect(level1).to include("lot_id" => s.lot.id, "analyte_key" => "glicose", "instrument_id" => s.instrument.id, "runs" => 20,
                                "mean" => mean, "sd" => sd, "current_version" => 1, "current_source" => "package_insert")
      expect(json_body["suggestions"].size).to eq(2)

      body = { lot_id: s.lot.id, level: "Nível 1", analyte_key: "glicose", instrument_id: s.instrument.id }
      json_post "/lab/qc/targets/suggestions/accept", body
      expect(json_body["error"]).to eq("reason_required") # C6
      body[:reason] = "20 corridas do lote; faixa da bula larga demais"
      json_post "/lab/qc/targets/suggestions/accept", body
      expect(response).to have_http_status(:created)
      expect(json_body).to include("version" => 2, "source" => "laboratory", "based_on_runs" => 20, "mean" => mean, "sd" => sd,
                                   "reason" => "20 corridas do lote; faixa da bula larga demais")
      json_post "/lab/qc/targets/suggestions/accept", body
      expect(response).to have_http_status(:not_found) # nenhuma corrida depois da nova referência
      get "/lab/qc/targets/suggestions", params: { provider_id: s.provider.id }
      expect(json_body["suggestions"].map { |x| x["level"] }).to eq([ "Nível 2" ])
      expect(DomainEvent.where(name: "lab_qc_target.accepted").map(&:payload).last).to include("source" => "laboratory")
      stepped_up!(staff)
      json_post "/lab/qc/targets/suggestions/accept", body.merge(level: "Nível 2")
      expect(response).to have_http_status(:forbidden)
      get "/lab/qc/targets/suggestions", params: { provider_id: s.provider.id }
      expect(response).to have_http_status(:forbidden)
    end

    it "corrida reprovada não conta para a sugestão" do
      offsets.first(19).each_with_index { |o, i| run_on!(Time.current - (20 - i).days, o) }
      qc_value!(s, "Nível 1", 3.5, by: staff, now: Time.current - 2.hours)
      expect(Lab::Qc::Suggestions.find(lot_id: s.lot.id, level: "Nível 1", analyte_key: "glicose", instrument_id: s.instrument.id)).to be_nil
    end

    it "referência trocada no meio da corrida: o valor antigo fica com a versão dele; o gráfico mostra as duas" do
      qc_value!(s, "Nível 1", 0.4, by: staff)
      travel 1.second
      qc_target!(s.lot, level: "Nível 1", analyte_key: "glicose", instrument: s.instrument, mean: 101, sd: 2, source: "laboratory", runs: 20, at: Time.current)
      travel 1.second
      second = qc_value!(s, "Nível 1", value: BigDecimal("105"), by: staff)
      expect(second[:value]).to have_attributes(z_score: BigDecimal("2"), outcome: "ok")
      expect(LabQcValue.order(:entered_at).map { |v| [ v.z_score, v.lab_qc_target.version ] }).to eq([ [ BigDecimal("0.4"), 1 ], [ BigDecimal("2"), 2 ] ])
      stepped_up!(lead)
      period = { from: (Time.zone.today - 31).iso8601, to: Time.zone.today.iso8601 }
      get "/lab/qc/chart", params: { lot_id: s.lot.id, level: "Nível 1", analyte_key: "glicose", instrument_id: s.instrument.id, **period }
      expect(json_body["target"]).to eq("mean" => "101", "sd" => "2", "version" => 2, "source" => "laboratory")
      expect(json_body["points"].map { |p| p.values_at("value", "z_score", "outcome", "target_version") })
        .to eq([ [ "102", 0.4, "ok", 1 ], [ "105", 2.0, "ok", 2 ] ])
      get "/lab/qc/chart", params: { lot_id: s.lot.id, level: "Nível 1", analyte_key: "glicose", instrument_id: s.instrument.id }
      expect(json_body["error"]).to eq("filter_required") # C17: período obrigatório
      get "/lab/qc/chart", params: { lot_id: s.lot.id, level: "Nível 2", analyte_key: "glicose",
                                     instrument_id: qc_instrument!(s.provider, name: "Outro", hl7_equipment_id: "BIO-03").id, **period }
      expect(json_body).to eq("target" => nil, "points" => [])
      expect(json_body["points"].first.keys.sort).to eq(%w[at outcome rules_fired target_version value z_score])
      get "/lab/qc/chart", params: { lot_id: s.lot.id, level: "Nível 1", analyte_key: "glicose", instrument_id: s.instrument.id,
                                     from: (Time.zone.today - 60).iso8601, to: (Time.zone.today - 40).iso8601 }
      expect(json_body["points"]).to eq([])
      get "/lab/qc/chart", params: { lot_id: s.lot.id, level: "Nível 1" }
      expect(json_body["error"]).to eq("filter_required")
      stepped_up!(staff)
      get "/lab/qc/chart", params: { lot_id: s.lot.id, level: "Nível 1", analyte_key: "glicose", instrument_id: s.instrument.id, **period }
      expect(response).to have_http_status(:forbidden)
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar:** `rspec spec/requests/lab/qc_targets_chart_spec.rb` → 404 (rotas) e `NameError: Lab::Qc::Suggestions`.

- [ ] **Step 3: Implemente.** `app/services/lab/qc/suggestions.rb`:

  ```ruby
  # Sugestão da referência do laboratório (spec §5.8; Desvio 4): por lote em
  # uso, nível, analito e aparelho, com a referência vigente; conta as
  # corridas aceitas com valor não reprovado gravado DEPOIS da referência
  # vigente (o último valor de cada corrida); com 20 ou mais, média e dp
  # amostral (n − 1), 4 casas, BigDecimal.
  module Lab
    module Qc
      module Suggestions
        MIN_RUNS = 20
        Suggestion = Data.define(:lot_id, :level, :analyte_key, :instrument_id, :mean, :sd, :runs, :current_version, :current_source)

        module_function

        def list(provider_id:)
          lots = LabQcLot.joins(:lab_qc_material).where(status: "in_use", lab_qc_materials: { exam_provider_id: provider_id }).select(:id)
          current = LabQcTarget.where(lab_qc_lot_id: lots).order(:version).to_a
                               .group_by { |t| [ t.lab_qc_lot_id, t.level, t.analyte_key, t.lab_instrument_id ] }.values.map(&:last)
          current.filter_map { |target| for_target(target) }.sort_by { |x| [ x.analyte_key, x.level, x.lot_id ] }
        end

        def find(lot_id:, level:, analyte_key:, instrument_id:)
          target = LabQcTarget.current(lot_id: lot_id, level: level, analyte_key: analyte_key, instrument_id: instrument_id)
          target && for_target(target)
        end

        def for_target(target)
          rows = LabQcValue.joins(:lab_qc_run)
                           .where(lab_qc_runs: { lab_instrument_id: target.lab_instrument_id, status: LabQcRun::ACCEPTED })
                           .where(lab_qc_lot_id: target.lab_qc_lot_id, level: target.level, analyte_key: target.analyte_key)
                           .where.not(outcome: "reject").where("lab_qc_values.entered_at > ?", target.accepted_at)
                           .order(:entered_at, :created_at).pluck(:lab_qc_run_id, :value)
          values = rows.each_with_object({}) { |(run_id, value), last| last[run_id] = BigDecimal(value.to_s) }.values
          return nil if values.size < MIN_RUNS

          mean = values.sum / values.size
          sd = (values.sum { |v| (v - mean)**2 } / (values.size - 1)).sqrt(20)
          return nil unless sd.positive?

          Suggestion.new(lot_id: target.lab_qc_lot_id, level: target.level, analyte_key: target.analyte_key, instrument_id: target.lab_instrument_id,
                         mean: mean.round(4), sd: sd.round(4), runs: values.size, current_version: target.version, current_source: target.source)
        end
      end
    end
  end
  ```

  `app/commands/lab/qc/accept_suggestion.rb`:

  ```ruby
  # Aceite do RT/substituto (spec §5.8): a sugestão é recalculada no servidor
  # com o lote travado (nunca vem do cliente); vira nova versão `laboratory`
  # com based_on_runs. Sem sugestão → not_found.
  module Lab
    module Qc
      module AcceptSuggestion
        module_function

        def call(lot_id:, level:, analyte_key:, instrument_id:, reason:, by:, now: Time.current)
          text = reason.is_a?(String) ? reason.strip : ""
          return Result.fail(:reason_required) unless text.length.between?(1, 1000) # C6

          lot = lot_id.to_s.match?(Exams::BaseController::UUID) ? LabQcLot.find_by(id: lot_id) : nil
          return Result.fail(:not_found) unless lot
          return Result.fail(:not_found) unless instrument_id.to_s.match?(Exams::BaseController::UUID)

          ApplicationRecord.transaction(requires_new: true) do
            lot.lock!
            next Result.fail(:lot_closed) if lot.status == "closed"

            suggestion = Suggestions.find(lot_id: lot.id, level: level.to_s, analyte_key: analyte_key.to_s, instrument_id: instrument_id)
            next Result.fail(:not_found) unless suggestion

            target = LabQcTarget.create!(lab_qc_lot: lot, level: suggestion.level, analyte_key: suggestion.analyte_key,
                                         lab_instrument_id: suggestion.instrument_id,
                                         version: CreateTarget.next_version(lot.id, suggestion.level, suggestion.analyte_key, suggestion.instrument_id),
                                         source: "laboratory", mean: suggestion.mean, sd: suggestion.sd, based_on_runs: suggestion.runs, reason: text,
                                         accepted_by_user: by, accepted_at: now, created_at: now)
            DomainEvents.publish("lab_qc_target.accepted", lab_qc_target_id: target.id, source: "laboratory")
            Result.ok(target: target)
          end
        end
      end
    end
  end
  ```

  `app/services/lab/qc/chart.rb`:

  ```ruby
  # Levey-Jennings (contrato §3; spec §5.9; C17): referência vigente (null se
  # não há) e até 200 pontos do período (mais recentes), do mais antigo ao mais
  # novo, cada um com o z e a versão da referência com que foi avaliado (o
  # dashboard plota em desvios-padrão). Sem trilha (nada de paciente).
  module Lab
    module Qc
      module Chart
        MAX_POINTS = 200

        module_function

        def call(lot:, level:, analyte_key:, instrument:, from:, to:)
          target = LabQcTarget.current(lot_id: lot.id, level: level, analyte_key: analyte_key, instrument_id: instrument.id)
          points = LabQcValue.joins(:lab_qc_run).includes(:lab_qc_target)
                             .where(lab_qc_runs: { lab_instrument_id: instrument.id })
                             .where(lab_qc_lot_id: lot.id, level: level, analyte_key: analyte_key, entered_at: from..to)
                             .order(entered_at: :desc, created_at: :desc).limit(MAX_POINTS).to_a.reverse
          { target: target && { mean: Decimal.out(target.mean), sd: Decimal.out(target.sd), version: target.version, source: target.source },
            points: points.map do |v|
              { at: v.entered_at.iso8601, value: Decimal.out(v.value), z_score: v.z_score.to_f, outcome: v.outcome,
                rules_fired: v.rules_fired, target_version: v.lab_qc_target.version }
            end }
        end
      end
    end
  end
  ```

  `app/controllers/lab/qc/charts_controller.rb`:

  ```ruby
  # GET /lab/qc/chart (contrato §3; spec §8; C17): liberador ligado, RT/substituto ou admin; from e to obrigatórios (até 366 dias).
  module Lab
    module Qc
      class ChartsController < BaseController
        def show
          lot = uuid?(params[:lot_id]) && LabQcLot.find_by(id: params[:lot_id])
          instrument = uuid?(params[:instrument_id]) && LabInstrument.find_by(id: params[:instrument_id])
          return render_failure(Result.fail(:filter_required)) unless lot && instrument && params[:level].present? && params[:analyte_key].present?
          return forbid unless releaser?(lot.lab_qc_material.exam_provider_id)

          from = date_param(:from)
          to = date_param(:to)
          return render_failure(Result.fail(:filter_required)) unless from && to # C17
          return render_failure(Result.fail(:invalid_period)) if from > to || (to - from) > 366

          render json: Chart.call(lot: lot, level: params[:level].to_s, analyte_key: params[:analyte_key].to_s, instrument: instrument,
                                  from: from.beginning_of_day, to: to.end_of_day)
        end

        private

        def date_param(key)
          params[key].to_s.match?(/\A\d{4}-\d{2}-\d{2}\z/) ? Date.iso8601(params[key]) : nil
        rescue Date::Error
          nil
        end
      end
    end
  end
  ```

  Em `app/controllers/lab/qc/targets_controller.rb`, troque `before_action :require_step_up!, only: %i[create]` por `before_action :require_step_up!, only: %i[create accept]` e acrescente as ações:

  ```ruby
        def suggestions
          provider = lab_param
          return not_found unless provider
          return forbid unless releaser?(provider.id)

          render json: { suggestions: Suggestions.list(provider_id: provider.id).map { |x| suggestion_json(x) } }
        end

        def accept
          lot = uuid?(params[:lot_id]) && LabQcLot.find_by(id: params[:lot_id])
          return not_found unless lot
          return forbid unless technical?(lot.lab_qc_material.exam_provider_id)

          result = AcceptSuggestion.call(lot_id: lot.id, level: params[:level], analyte_key: params[:analyte_key], instrument_id: params[:instrument_id],
                                         reason: params[:reason], by: Current.user)
          return render_failure(result) if result.failure?

          render json: Json.target(result.payload[:target]), status: :created
        end

        private

        def suggestion_json(x)
          { lot_id: x.lot_id, level: x.level, analyte_key: x.analyte_key, instrument_id: x.instrument_id, mean: Json.out(x.mean), sd: Json.out(x.sd),
            runs: x.runs, current_version: x.current_version, current_source: x.current_source }
        end
  ```

  `config/routes.rb` — dentro de `scope "qc"`, **antes** de `get "targets"`:

  ```ruby
        get   "targets/suggestions",        to: "targets#suggestions"
        post  "targets/suggestions/accept", to: "targets#accept"
        get   "chart",                      to: "charts#show"
  ```

- [ ] **Step 4: Rode:** `rspec spec/requests/lab/qc_targets_chart_spec.rb spec/requests/lab/qc_setup_spec.rb` → verde.

- [ ] **Step 5: Commit** e mova os cards F-21.26 e F-21.30 para **Done** (F-21.30 só a parte do api; anote no card que a tela é do plano do dashboard):

  ```bash
  git add app/services/lab/qc/suggestions.rb app/services/lab/qc/chart.rb app/commands/lab/qc/accept_suggestion.rb app/controllers/lab/qc/charts_controller.rb app/controllers/lab/qc/targets_controller.rb config/routes.rb spec/requests/lab/qc_targets_chart_spec.rb
  git commit -m "feat: suggest laboratory quality control targets and serve Levey-Jennings data

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

---

## Fatia 4 — HL7 do controle e middleware simulado

### Task 10: Controle pela HL7 (F-21.28)

**Files:**
- Create: `app/services/lab/qc/hl7_inbound.rb`, `spec/requests/hl7/qc_spec.rb`, `spec/fixtures/lab/{oul-r22-qc,oul-r22-qc-2-2s,ack-ae-qc}.hl7` (cópias do contracts)
- Modify: `app/services/lab/hl7/{message,builder,inbound}.rb`, `app/controllers/hl7/v2_controller.rb`

**Interfaces:**
- Consumes: `Lab::Qc::RecordValue` (com `run:` interno), `Lab::Qc::Decimal.hl7` (Tasks 4, 7); `LabQcLotBarcode`, `LabQcTarget.current` (Task 3); exemplos da Task 1.
- Produces: `Lab::Hl7::Message#specimen_role(specimen) → String` (SPM-11.1) e `#repetitions(segment, n) → [String]` (componente 1 de cada repetição); `Lab::Hl7::Builder::ERRORS` com `qc_lot_expired`, `qc_lot_closed`, `qc_disabled`, `qc_no_target`, `unknown_instrument`; `Lab::Hl7::Inbound.call(raw:, token:, city_slug:, fake_allowed:, qc_allowed: false, now:)`; `Lab::Hl7::Inbound::Application.new(message, token, receipt, now, write:, skip: Set.new)`; `Lab::Qc::Hl7Inbound.new(message, token, receipt, now, enabled:, write:)` com `#run → [Builder::Error]` e `#claimed → Set` (índices de `SPM` tratados como controle).

- [ ] **Step 1: Copie os exemplos** do contracts (branch da Task 1) para os fixtures:

  ```bash
  cp contracts/lab/examples/oul-r22-qc.hl7 contracts/lab/examples/oul-r22-qc-2-2s.hl7 contracts/lab/examples/ack-ae-qc.hl7 apps/api/.claude/mod21c/spec/fixtures/lab/
  ```

- [ ] **Step 2: Teste que falha** — `spec/requests/hl7/qc_spec.rb`:

  ```ruby
  require "rails_helper"

  # Spec §7 e contrato §6 (F-21.28; Task 0): o SPM-2 = código de barras de
  # nível de lote de controle cadastrado decide que é controle; aparelho pela
  # repetição de OBX-18; cada valor avaliado por Westgard; reprovação não muda
  # o ACK; controle nunca vira resultado de paciente (Review Focus 2);
  # recusas AE todas de uma vez; mensagem inteira ou nada.
  RSpec.describe "HL7 — controle de qualidade", type: :request do
    before { qc_city!(hl7: true) }

    let(:provider) { lab_provider! }
    let(:instrument) { qc_instrument!(provider) } # BIO-01
    let(:exam) { qc_exam!(provider: provider, instrument: instrument) } # GLI
    let(:material) { qc_material!(provider, exams: [ exam ]) }
    let(:lot) { qc_lot!(material, lot_number: "CQB-2611", barcodes: { "Nível 1" => "CQB2611N1", "Nível 2" => "CQB2611N2" }) }
    let!(:targets) do
      [ [ "Nível 1", "90", "4.5" ], [ "Nível 2", "270", "13.5" ] ].map do |level, mean, sd|
        qc_target!(lot, level: level, analyte_key: "glicose", instrument: instrument, mean: mean, sd: sd)
      end
    end
    let(:token) { middleware_token!(provider) }

    def fixture(name) = File.binread(Rails.root.join("spec/fixtures/lab", name))
    def oul(*segments) = ([ "MSH|^~\\&|MW|LAB|ROTA-SAUDE|t|20261018073000||OUL^R22^OUL_R22|C#{SecureRandom.hex(4)}|P|2.5.1" ] + segments).join("\r") + "\r"
    def spm(i, barcode, role: "Q") = "SPM|#{i}|#{barcode}|||||||||#{role}"
    def obr(i, code = "GLI") = "OBR|#{i}|||#{code}^#{code}^99RSL"
    def obx(i, value, equipment: "BIO-01", key: "glicose") = "OBX|#{i}|NM|#{key}^G^99RSL||#{value}|mg/dL^^UCUM|||||F|||20261018072500||||#{equipment}"
    def msa = response.body.split("\r").find { |s| s.start_with?("MSA|") }.split("|")[1]
    def errs = response.body.split("\r").select { |s| s.start_with?("ERR|") }
    def err5 = errs.map { |e| [ e.split("|")[2], e.split("|")[5].split("^").first ] }

    it "exemplo aprovado do contracts: AA, dois valores na mesma corrida, nada de paciente" do
      hl7_post(fixture("oul-r22-qc.hl7"), token: token)
      expect(msa).to eq("AA")
      values = LabQcValue.order(:level).to_a
      expect(values.map { |v| [ v.level, v.z_score, v.outcome, v.source ] })
        .to eq([ [ "Nível 1", BigDecimal("0.4"), "ok", "hl7" ], [ "Nível 2", BigDecimal("-0.6"), "ok", "hl7" ] ])
      expect(values.map(&:lab_qc_run_id).uniq.size).to eq(1)
      expect(values.first.lab_qc_run.status).to eq("approved")
      expect(values.map(&:lab_hl7_receipt_id).uniq.size).to eq(1)
      expect(LabResult.count).to eq(0)
    end

    it "exemplo 2-2s do contracts: AA (a mensagem é aceita), corrida reprovada e bloqueio" do
      hl7_post(fixture("oul-r22-qc-2-2s.hl7"), token: token)
      expect(msa).to eq("AA")
      level2 = LabQcValue.find_by!(level: "Nível 2")
      expect(level2.rules_fired).to eq(%w[1_2s 2_2s])
      expect(level2.lab_qc_run.status).to eq("rejected")
      expect(LabQcBlock.active.sole).to have_attributes(lab_instrument_id: instrument.id, analyte_key: "glicose")
    end

    it "controle nunca vira paciente" do
      # Sem SPM-11, ou com P: o código de barras de controle decide.
      hl7_post(oul(spm(1, "CQB2611N1", role: ""), obr(1), obx(1, "90.0")), token: token)
      expect(msa).to eq("AA")
      hl7_post(oul(spm(1, "CQB2611N2", role: "P"), obr(1), obx(1, "270.0")), token: token)
      expect(msa).to eq("AA")
      expect(LabQcValue.count).to eq(2)
      # Q com código de amostra de paciente: recusado, nada gravado.
      req = received_requisition!(provider: provider, exams: [ exam ])
      patient_barcode = req.samples.first.barcode
      hl7_post(oul(spm(1, patient_barcode, role: "Q"), obr(1), "OBX|1|NM|glicose^G^99RSL||98|mg/dL|||||F"), token: token)
      expect([ msa, err5 ]).to eq([ "AE", [ [ "SPM^1^2", "unknown_barcode" ] ] ])
      expect(LabResult.count).to eq(0)
      # Controle e paciente juntos: os dois entram; erro no controle derruba os dois.
      hl7_post(oul(spm(1, "CQB2611N1"), obr(1), obx(1, "91.8"), "SPM|2|#{patient_barcode}", obr(1), "OBX|1|NM|glicose^G^99RSL||98|mg/dL|||||F"),
               token: token)
      expect(msa).to eq("AA")
      expect([ LabResult.count, LabQcValue.count ]).to eq([ 1, 3 ])
      hl7_post(oul(spm(1, "CQB2611N1"), obr(1), obx(1, "91.8", equipment: "XX"), "SPM|2|#{patient_barcode}", obr(1),
                   "OBX|1|NM|glicose^G^99RSL||99|mg/dL|||||F"), token: token)
      expect(msa).to eq("AE")
      expect([ LabResult.count, LabQcValue.count ]).to eq([ 1, 3 ])
    end

    it "todas as recusas de uma vez, como o exemplo ack-ae-qc do contracts" do
      expired = qc_lot!(material, lot_number: "CQV-1", expires_on: Time.zone.today - 1, barcodes: { "Nível 1" => "CQV1N1", "Nível 2" => "CQV1N2" })
      qc_target!(expired, level: "Nível 1", analyte_key: "glicose", instrument: instrument, mean: 90, sd: 4.5)
      hl7_post(oul(spm(1, "CQV1N1"), obr(1), obx(1, "90.0"), spm(2, "CQB2611N2"), obr(2), obx(2, "270.0", equipment: "XX")), token: token)
      expect(msa).to eq("AE")
      expect(errs).to eq(fixture("ack-ae-qc.hl7").split("\r").select { |s| s.start_with?("ERR|") })
      expect(LabQcValue.count).to eq(0)
    end

    it "recusas: lote encerrado, sem referência, analito, aparelho ausente, vírgula, controle desligado" do
      qc_instrument!(provider, name: "Analisador bioquímico 2", hl7_equipment_id: "BIO-02")
      hl7_post(oul(spm(1, "CQB2611N1"), obr(1), obx(1, "90.0", equipment: "BIO-02"), obx(2, "90.0", key: "ureia"), obx(3, "90,5"),
                   obx(4, "90.0", equipment: "")), token: token)
      expect(msa).to eq("AE")
      expect(err5).to eq([ [ "OBX^1^3", "qc_no_target" ], [ "OBX^2^3", "unknown_analyte" ], [ "OBX^3^5", "invalid_value" ],
                           [ "OBX^4^18", "unknown_instrument" ] ])
      hl7_post(oul(spm(1, "CQB2611N1"), obr(1, "XYZ"), obx(1, "90.0")), token: token)
      expect(err5).to eq([ [ "OBR^1^4", "exam_not_in_version" ] ])
      lot.update!(status: "closed", closed_on: Time.zone.today)
      hl7_post(oul(spm(1, "CQB2611N1"), obr(1), obx(1, "90.0")), token: token)
      expect(err5).to eq([ [ "SPM^1^2", "qc_lot_closed" ] ])
      Platform::Features.set!(city: City.find(Current.city.id), key: "lab_quality_control", enabled: false, maintainer: ledi_maintainer!)
      hl7_post(oul(spm(1, "CQB2611N2"), obr(1), obx(1, "270.0")), token: token)
      expect(err5).to eq([ [ "SPM^1^2", "qc_disabled" ] ])
      expect(LabQcValue.count).to eq(0)
    end
  end
  ```

- [ ] **Step 3: Rode e veja falhar:** `rspec spec/requests/hl7/qc_spec.rb` → `AE` `unknown_barcode` (o código de controle cai no caminho do paciente).

- [ ] **Step 4: Leitura da mensagem.** Em `app/services/lab/hl7/message.rb`, depois de `def barcode(specimen) ...`:

  ```ruby
      # SPM-11 (Specimen Role, tabela 0369): "Q" = controle (Task 0).
      def specimen_role(specimen) = field(specimen.spm, 11, 1).to_s

      # Componente 1 de cada repetição do campo (OBX-18 é repetível; Task 0).
      def repetitions(segment, n)
        return [] unless segment

        segment[n].to_s.split("~", -1).map { |rep| self.class.unescape(rep.split("^", -1).first.to_s) }.reject(&:empty?)
      end
  ```

  Em `app/services/lab/hl7/builder.rb`, acrescente a `ERRORS` (antes de `"invalid_message"`):

  ```ruby
          "qc_lot_expired" => [ "207", "Application internal error", "Lote de controle vencido" ],
          "qc_lot_closed" => [ "207", "Application internal error", "Lote de controle encerrado" ],
          "qc_disabled" => [ "207", "Application internal error", "Controle de qualidade desligado" ],
          "qc_no_target" => [ "207", "Application internal error", "Controle sem referencia" ],
          "unknown_instrument" => [ "204", "Unknown key identifier", "Aparelho desconhecido" ],
  ```

- [ ] **Step 5: O controle na HL7** — `app/services/lab/qc/hl7_inbound.rb`:

  ```ruby
  # Controle pela HL7 (spec §7; contrato §6; Task 0): dentro da transação da
  # mensagem e ANTES das amostras de paciente (Desvio 10: aparelho primeiro).
  # SPM-2 = código de barras de nível de lote cadastrado do laboratório do
  # token → controle (claimed: o caminho do paciente pula esse SPM). SPM-11 = Q
  # com código que não é de controle → unknown_barcode (claimed também: nunca
  # vira paciente). Confere tudo antes de gravar (B26); os valores de um
  # aparelho na mensagem ficam na mesma corrida (Desvio 6).
  module Lab
    module Qc
      class Hl7Inbound
        FAILURES = { lot_closed: "qc_lot_closed", expired_lot: "qc_lot_expired", no_target: "qc_no_target",
                     invalid_instrument: "unknown_instrument" }.freeze

        attr_reader :claimed

        def initialize(message, token, receipt, now, enabled:, write:)
          @message = message
          @token = token
          @receipt = receipt
          @now = now
          @enabled = enabled
          @write = write
          @claimed = Set.new
          @errors = []
        end

        def run
          plan = @message.specimens.flat_map { |specimen| plan_for(specimen) }
          return @errors if @errors.any? || !@write || plan.empty?

          LabInstrument.where(id: plan.map { |step| step[:instrument].id }.uniq).order(:id).lock.to_a
          runs = {}
          plan.each do |step|
            recorded = RecordValue.call(instrument: step[:instrument], lot: step[:lot], level: step[:level], analyte_key: step[:analyte_key],
                                        value: step[:value], source: "hl7", receipt: @receipt, run: runs[step[:instrument].id], now: @now)
            if recorded.failure?
              @errors << error(FAILURES.fetch(recorded.reason, "invalid_value"), step[:where])
            else
              runs[step[:instrument].id] ||= recorded.payload[:run]
            end
          end
          @errors
        end

        private

        def error(code, location) = Lab::Hl7::Builder::Error.new(code: code, location: location)

        def plan_for(specimen)
          where = "SPM^#{specimen.index}^2"
          entry = LabQcLotBarcode.joins(lab_qc_lot: :lab_qc_material).where(lab_qc_materials: { exam_provider_id: @token.exam_provider_id })
                                 .includes(lab_qc_lot: :lab_qc_material).find_by(barcode: @message.barcode(specimen))
          unless entry
            if @message.specimen_role(specimen) == "Q"
              @claimed << specimen.index
              @errors << error("unknown_barcode", where)
            end
            return []
          end

          @claimed << specimen.index
          lot = entry.lab_qc_lot
          refusal = if !@enabled then "qc_disabled"
                    elsif lot.status == "closed" then "qc_lot_closed"
                    elsif lot.expired?(@now.in_time_zone.to_date) then "qc_lot_expired"
                    end
          if refusal
            @errors << error(refusal, where)
            return []
          end
          specimen.orders.flat_map { |order| order_plan(order, lot, entry.level) }
        end

        def order_plan(order, lot, level)
          material = lot.lab_qc_material
          code = @message.field(order.obr, 4, 1).to_s
          exam = exam_for(material, code)
          unless exam
            @errors << error("exam_not_in_version", "OBR^#{order.index}^4") if code.present? # vazio: já é erro de forma
            return []
          end

          seen = Set.new
          steps = []
          order.results.each do |obx|
            step = obx_step(obx, exam, material, lot, level, seen)
            steps << step if step
          end
          steps
        end

        def obx_step(obx, exam, material, lot, level, seen)
          segment = obx.segment
          at = ->(field) { "OBX^#{obx.index}^#{field}" }
          key = @message.field(segment, 3, 1).to_s
          return nil if key.blank? # forma: structure_errors

          analyte = exam.current_version.analyte(key)
          unless analyte && material.analyte_entry(key, exam_id: exam.id)
            @errors << error("unknown_analyte", at.call(3))
            return nil
          end
          unit = @message.field(segment, 6, 1).to_s
          if unit.present? && analyte["unit"].present? && unit != analyte["unit"]
            @errors << error("unit_mismatch", at.call(6))
            return nil
          end
          value = Decimal.hl7(@message.field(segment, 5, 1).to_s)
          unless value
            @errors << error("invalid_value", at.call(5))
            return nil
          end
          instrument = instrument_for(@message.repetitions(segment, 18))
          unless instrument
            @errors << error("unknown_instrument", at.call(18))
            return nil
          end
          unless LabQcTarget.current(lot_id: lot.id, level: level, analyte_key: key, instrument_id: instrument.id)
            @errors << error("qc_no_target", at.call(3))
            return nil
          end
          # Dois OBX do mesmo analito no mesmo OBR: a ordem não decide qual vale (como E5/B26).
          unless seen.add?(key)
            @errors << error("invalid_value", at.call(5))
            return nil
          end
          { instrument: instrument, lot: lot, level: level, analyte_key: key, value: value, where: at.call(5) }
        end

        def exam_for(material, code)
          return nil if code.blank?

          LabExam.where(id: material.analytes.map { |a| a["lab_exam_id"] }, exam_provider_id: @token.exam_provider_id, active: true)
                 .joins(:current_version).includes(:current_version)
                 .find_by("upper(lab_exam_versions.instrument_test_code) = ?", code.upcase)
        end

        # A repetição de OBX-18 que casa com UM aparelho ativo do laboratório (sem diferenciar caixa).
        def instrument_for(ids)
          return nil if ids.empty?

          found = LabInstrument.active.where(exam_provider_id: @token.exam_provider_id)
                               .where("upper(hl7_equipment_id) IN (?)", ids.map(&:upcase)).to_a
          found.size == 1 ? found.first : nil
        end
      end
    end
  end
  ```

- [ ] **Step 6: Ligue na entrada.** Em `app/services/lab/hl7/inbound.rb`:
  - comentário de topo: acrescente "Controle (ADR 0038): tratado antes das amostras de paciente por `Lab::Qc::Hl7Inbound`, na mesma transação; o SPM de controle nunca chega ao caminho do paciente."
  - assinatura: `def call(raw:, token:, city_slug:, fake_allowed:, qc_allowed: false, now: Time.current)`; a última linha do `call` passa a `message.type.start_with?("QBP") ? worklist(message, token, city_slug, now) : results(message, raw, token, city_slug, now, qc_allowed)`.
  - `def results(message, raw, token, city_slug, now, qc_allowed)` e, dentro da transação, troque a linha `errors.concat(Application.new(message, token, receipt, now, write: structural.empty?).run)` por:

    ```ruby
          qc = Lab::Qc::Hl7Inbound.new(message, token, receipt, now, enabled: qc_allowed, write: structural.empty?)
          errors.concat(qc.run) # aparelho antes da requisição (Desvio 10)
          errors.concat(Application.new(message, token, receipt, now, write: structural.empty?, skip: qc.claimed).run)
    ```
  - em `Application#initialize`, acrescente `skip: Set.new` aos parâmetros (`def initialize(message, token, receipt, now, write:, skip: Set.new)`) e `@skip = skip`; em `resolve`, a primeira linha do `@message.specimens.each do |specimen|` passa a ser `next if @skip.include?(specimen.index)`.

  Em `app/controllers/hl7/v2_controller.rb`, a chamada passa a:

  ```ruby
      reply = Lab::Hl7::Inbound.call(raw: body, token: @token, city_slug: Current.city.slug, fake_allowed: fake_allowed?,
                                     qc_allowed: Lab::Gate.qc_usable?(Current.city))
  ```

- [ ] **Step 7: Rode:** `rspec spec/requests/hl7/qc_spec.rb spec/requests/hl7/v2_spec.rb spec/services/lab/hl7_spec.rb spec/lib/fake_middleware_spec.rb spec/invariants/municipal_lab_invariants_spec.rb` → verde (as do 21b provam que a amostra de paciente não mudou).

- [ ] **Step 8: Commit:**

  ```bash
  git add app/services/lab/qc/hl7_inbound.rb app/services/lab/hl7/message.rb app/services/lab/hl7/builder.rb app/services/lab/hl7/inbound.rb app/controllers/hl7/v2_controller.rb spec/requests/hl7/qc_spec.rb spec/fixtures/lab/oul-r22-qc.hl7 spec/fixtures/lab/oul-r22-qc-2-2s.hl7 spec/fixtures/lab/ack-ae-qc.hl7
  git commit -m "feat: accept quality control results over HL7 identified by control barcode

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 11: `fake-middleware` mandando corridas de controle (aprovada e reprovada por 2-2s)

**Files:**
- Create: `lib/fake_middleware/qc_plan.rb`, `spec/lib/fake_middleware_qc_spec.rb`
- Modify: `lib/fake_middleware/{messages,runner,run}.rb`

**Interfaces:**
- Consumes: `POST /hl7/v2` com controle (Task 10).
- Produces: `FakeMiddleware::QcPlan::{INSTRUMENT,LOT,BARCODES,ANALYTES,TARGETS,SCENARIOS}`, `QcPlan.z(scenario, level, code, on) → Float`, `QcPlan.value(level, code, z) → String` (ponto decimal) — **fonte única** da semente (Task 17) e do simulado; `FakeMiddleware::Messages.qc(scenario:, on:, control:) → String` (ER7); `FakeMiddleware::Runner.qc(client:, logger:, scenario:, on: Date.today) → MSA|nil`. `run.rb qc <approved|reject_2_2s>` manda uma corrida e sai; o laço manda uma corrida aprovada por dia.

- [ ] **Step 1: Teste que falha** — `spec/lib/fake_middleware_qc_spec.rb`:

  ```ruby
  require "rails_helper"
  require Rails.root.join("lib/fake_middleware/runner").to_s

  # Spec §7 e contrato §6: o simulado manda corridas de controle — níveis 1 e
  # 2 do lote CQB-2611 no BIO-01, glicose e creatinina —, uma aprovada e uma
  # reprovada por 2-2s entre níveis na glicose. Log sem código de barras nem valor.
  RSpec.describe "FakeMiddleware — controle", type: :request do
    let(:plan) { FakeMiddleware::QcPlan }

    before do
      city = lab_city!(fake_middleware: true)
      Platform::Features.set!(city: city, key: "lab_quality_control", enabled: true, maintainer: ledi_maintainer!)
      CityCatalog.reset_cache!
      provider = lab_provider!
      instrument = qc_instrument!(provider, hl7_equipment_id: plan::INSTRUMENT)
      glucose = qc_exam!(provider: provider, instrument: instrument)
      creat = qc_exam!(provider: provider, instrument: instrument, sigtap_code: "0202010317", name: "Creatinina", instrument_test_code: "CRE",
                       analytes: [ LabHelpers::GLUCOSE.first.merge("key" => "creatinina", "name" => "Creatinina", "decimals" => 2, "entry_min" => nil,
                                                                  "entry_max" => nil, "critical_low" => nil, "critical_high" => nil) ])
      lot = qc_lot!(qc_material!(provider, exams: [ glucose, creat ]), lot_number: plan::LOT, barcodes: plan::BARCODES)
      plan::TARGETS.each do |level, by_code|
        by_code.each do |code, (mean, sd, _decimals)|
          qc_target!(lot, level: level, analyte_key: plan::ANALYTES.fetch(code).first, instrument: instrument, mean: mean, sd: sd)
        end
      end
      @token = middleware_token!(provider, simulated: true)
    end

    let(:log_io) { StringIO.new }
    let(:logger) { Logger.new(log_io) }
    let(:client) do
      session = ActionDispatch::Integration::Session.new(Rails.application)
      session.host! test_city_host
      Class.new do
        define_method(:initialize) { |s, t| @s = s; @t = t }
        attr_reader :last_status

        define_method(:post) do |body|
          @s.post "/hl7/v2", params: body, headers: { "CONTENT_TYPE" => "application/hl7-v2", "Authorization" => "Bearer #{@t}" }
          @last_status = @s.response.status
          [ @last_status, @s.response.body ]
        end
      end.new(session, @token)
    end

    it "corrida aprovada: AA, quatro valores, corrida aprovada" do
      expect(FakeMiddleware::Runner.qc(client: client, logger: logger, scenario: "approved", on: Date.new(2026, 10, 18))).to eq("AA")
      expect(LabQcValue.pluck(:outcome)).to eq(%w[ok ok ok ok])
      expect(LabQcRun.sole.status).to eq("approved")
    end

    it "corrida reprovada por 2-2s: AA, bloqueio da glicose; creatinina aceita" do
      expect(FakeMiddleware::Runner.qc(client: client, logger: logger, scenario: "reject_2_2s", on: Date.new(2026, 10, 18))).to eq("AA")
      expect(LabQcValue.find_by!(level: "Nível 2", analyte_key: "glicose").rules_fired).to eq(%w[1_2s 2_2s])
      expect(LabQcValue.where(analyte_key: "creatinina").pluck(:outcome)).to eq(%w[ok ok])
      expect(LabQcBlock.active.pluck(:analyte_key)).to eq([ "glicose" ])
    end

    it "loga só cenário, MSH-10 e MSA; cenário desconhecido é recusado" do
      FakeMiddleware::Runner.qc(client: client, logger: logger, scenario: "approved", on: Date.new(2026, 10, 18))
      text = log_io.string.lines.map { |line| line.split(" -- : ", 2).last }.join.gsub(/MSH-10 M[0-9a-f]+/, "MSH-10 M")
      expect(text).to include("controle approved MSH-10 M MSA AA")
      [ *plan::BARCODES.values, "88.2", "278.1", @token ].each { |secret| expect(text).not_to include(secret) }
      expect { FakeMiddleware::Runner.qc(client: client, logger: logger, scenario: "x") }.to raise_error(ArgumentError)
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar:** `rspec spec/lib/fake_middleware_qc_spec.rb` → `NameError: FakeMiddleware::QcPlan`.

- [ ] **Step 3: Implemente.** `lib/fake_middleware/qc_plan.rb`:

  ```ruby
  require "bigdecimal"

  # Plano do controle simulado (ADR 0038; Task 0): aparelho, lote, códigos de
  # barras por nível e a bula do lote. Fonte única: a semente de dev
  # (lib/lab_qc_crew.rb) cadastra exatamente isto, e o simulado manda valores
  # calculados daqui. Sem Rails (o simulado roda sem carregar o app).
  module FakeMiddleware
    module QcPlan
      INSTRUMENT = "BIO-01".freeze
      LOT = "CQB-2611".freeze
      BARCODES = { "Nível 1" => "CQB2611N1", "Nível 2" => "CQB2611N2" }.freeze
      # código do aparelho → [analyte_key, unidade]
      ANALYTES = { "GLI" => [ "glicose", "mg/dL" ], "CRE" => [ "creatinina", "mg/dL" ] }.freeze
      # bula: nível → código → [média, dp, casas]
      TARGETS = { "Nível 1" => { "GLI" => [ "90", "4.5", 1 ], "CRE" => [ "1.00", "0.06", 2 ] },
                  "Nível 2" => { "GLI" => [ "270", "13.5", 1 ], "CRE" => [ "4.80", "0.25", 2 ] } }.freeze
      # Aprovada: z pequenos, de lados opostos, que trocam de sinal a cada dia (nada de 10x ou 4-1s ao longo dos dias).
      APPROVED_Z = { "Nível 1" => 0.4, "Nível 2" => -0.6 }.freeze
      # Reprovada: os dois níveis da glicose acima de +2 dp na mesma corrida → 2-2s entre níveis.
      REJECT_Z = { "Nível 1" => 2.4, "Nível 2" => 2.6 }.freeze
      SCENARIOS = %w[approved reject_2_2s].freeze

      module_function

      def z(scenario, level, code, on)
        return REJECT_Z.fetch(level) if scenario == "reject_2_2s" && code == "GLI"

        APPROVED_Z.fetch(level) * (on.yday.even? ? 1 : -1)
      end

      def value(level, code, z)
        mean, sd, decimals = TARGETS.fetch(level).fetch(code)
        number = (BigDecimal(mean) + (BigDecimal(sd) * BigDecimal(z.to_s))).round(decimals, BigDecimal::ROUND_HALF_UP)
        format("%.#{decimals}f", number)
      end
    end
  end
  ```

  Em `lib/fake_middleware/messages.rb`: no topo, `require_relative "qc_plan"`; no comentário, acrescente "e corridas de controle (ADR 0038): SPM-11 = Q, aparelho em OBX-18"; e o método:

  ```ruby
      # Uma corrida de controle: um SPM por nível (SPM-2 = código do nível, SPM-11 = Q), um OBR por exame, aparelho em OBX-18.
      def qc(scenario:, on:, control: control_id)
        stamp = Time.now.utc.strftime("%Y%m%d%H%M%S")
        segments = [ header("OUL^R22^OUL_R22", control) ]
        QcPlan::BARCODES.each_with_index do |(level, barcode), i|
          segments << [ "SPM", (i + 1).to_s, barcode, *Array.new(8, ""), "Q" ].join("|")
          QcPlan::ANALYTES.each_with_index do |(code, (key, unit)), j|
            value = QcPlan.value(level, code, QcPlan.z(scenario, level, code, on))
            segments << "OBR|#{j + 1}|||#{code}^#{code}^99RSL"
            segments << "OBX|1|NM|#{key}^#{key}^99RSL||#{value}|#{unit}^^UCUM|||||F|||#{stamp}||||#{QcPlan::INSTRUMENT}"
          end
        end
        segments.join("\r") + "\r"
      end
  ```

  Em `lib/fake_middleware/runner.rb`, depois de `tick`:

  ```ruby
      # Uma corrida de controle (ADR 0038). Log só com cenário, MSH-10 e MSA.
      def qc(client:, logger:, scenario:, on: Date.today)
        raise ArgumentError, "cenário desconhecido" unless QcPlan::SCENARIOS.include?(scenario)

        control = Messages.control_id
        _status, reply = client.post(Messages.qc(scenario: scenario, on: on, control: control))
        msa = Messages.msa(reply)
        logger.info("[fake-middleware] controle #{scenario} MSH-10 #{control} MSA #{msa}")
        msa
      end
  ```

  Em `lib/fake_middleware/run.rb`, troque o laço final por:

  ```ruby
  # `run.rb qc <approved|reject_2_2s>`: manda UMA corrida de controle e sai (prova no navegador).
  if ARGV.first == "qc"
    scenario = ARGV[1] || "approved"
    abort "cenário desconhecido: use #{FakeMiddleware::QcPlan::SCENARIOS.join(' ou ')}" unless FakeMiddleware::QcPlan::SCENARIOS.include?(scenario)
    FakeMiddleware::Runner.qc(client: client, logger: logger, scenario: scenario)
    exit 0
  end

  require "date"
  last_qc_on = nil
  loop do
    begin
      FakeMiddleware::Runner.tick(client: client, logger: logger)
      # Uma corrida de controle aprovada por dia (como a rotina de abertura do laboratório).
      unless last_qc_on == Date.today
        FakeMiddleware::Runner.qc(client: client, logger: logger, scenario: "approved")
        last_qc_on = Date.today
      end
    rescue StandardError => e
      logger.warn("[fake-middleware] ciclo falhou: #{e.class}")
    end
    sleep interval
  end
  ```
  (as linhas acima do laço — guarda, `client`, `logger`, `interval` — não mudam; o `require_relative "runner"` já carrega `messages` e, por ele, `qc_plan`).

- [ ] **Step 4: Rode:** `rspec spec/lib/fake_middleware_qc_spec.rb spec/lib/fake_middleware_spec.rb` → verde. Confira também o guarda: `docker compose exec -T -w /rails/.claude/mod21c api ruby lib/fake_middleware/run.rb qc nada` → aborta com "cenário desconhecido" (sem `FAKE_MIDDLEWARE_TOKEN` ele aborta antes no `ENV.fetch`; rode com `-e FAKE_MIDDLEWARE_TOKEN=x`).

- [ ] **Step 5: Commit** e mova o card F-21.28 para **Done**:

  ```bash
  git add lib/fake_middleware/qc_plan.rb lib/fake_middleware/messages.rb lib/fake_middleware/runner.rb lib/fake_middleware/run.rb spec/lib/fake_middleware_qc_spec.rb
  git commit -m "feat: send quality control runs from the fake middleware

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

---

## Fatia 5 — CEQ

### Task 12: Registro do CEQ (F-21.31)

**Files:**
- Create: `app/commands/lab/eqa/record_round.rb`, `app/services/lab/eqa/{summary,json}.rb`, `app/controllers/lab/eqa/rounds_controller.rb`, `spec/requests/lab/eqa_spec.rb`
- Modify: `config/routes.rb`

**Interfaces:**
- Consumes: `LabEqaRound` (Task 3); `Lab::Qc::BaseController` (Task 5; o CEQ fica atrás de `lab_quality_control`); `Lab::Qc::Decimal`; `Exams::Files.{store_base64,claim!}` (21a).
- Produces: `Lab::Eqa::RecordRound.call(params, by:, now:) → Result(round:)`; `Lab::Eqa::Summary.call(provider_id:, year:) → { year:, rows: [{ analyte_key, lab_exam_id, rounds, adequate, inadequate }] }` (forma C3 do dashboard); `Lab::Eqa::Json.round(round)`. Rotas (C12 do dashboard): `POST /lab/eqa/uploads { content_type: "application/pdf", data }` → `{ upload_id }`; `GET|POST /lab/eqa/rounds` (`year` opcional na lista); `GET /lab/eqa/rounds/:id`; `GET /lab/eqa/rounds/:id/annual_report.pdf`; `GET /lab/eqa/summary?provider_id=&year=`.

- [ ] **Step 1: Teste que falha** — `spec/requests/lab/eqa_spec.rb`:

  ```ruby
  require "rails_helper"

  # Contrato §4 e C12 do dashboard (F-21.31): rodada do programa externo com
  # itens por analito (enviado, alvo, faixa aceitável, adequação, ação
  # obrigatória no inadequado), relatório anual em PDF (envio e leitura no
  # próprio /lab/eqa); adequação por analito no ano; liberador ligado, RT ou admin.
  RSpec.describe "CEQ — rodadas", type: :request do
    before { qc_city! }

    let(:provider) { lab_provider! }
    let(:exam) { lab_exam!(provider: provider) }
    let(:lead) { lab_releaser!(provider, lead: true) }
    let(:pdf) { Base64.strict_encode64("%PDF-1.4\n% relatório anual\n%%EOF") }

    def item(adequacy: "adequate", action: nil, submitted: "98,5")
      { analyte_key: "glicose", lab_exam_id: exam.id, submitted: submitted, target: "100", acceptable_low: "90", acceptable_high: "110",
        adequacy: adequacy, action: action }
    end

    def round(code, items, date: "2026-03-10", upload_id: nil)
      json_post "/lab/eqa/rounds", provider_id: provider.id, program: "PNCQ", round_code: code, round_date: date, items: items,
                                   annual_report_upload_id: upload_id
    end

    it "registra rodadas, anexa e lê o relatório anual e resume a adequação do ano" do
      stepped_up!(lead)
      json_post "/lab/eqa/uploads", content_type: "application/pdf", data: pdf
      expect(response).to have_http_status(:created)
      round("2026-01", [ item(adequacy: "inadequate", action: "Recalibração e nova dosagem") ], upload_id: json_body["upload_id"])
      expect(response).to have_http_status(:created)
      created = json_body
      expect(created).to include("program" => "PNCQ", "round_code" => "2026-01", "round_date" => "2026-03-10",
                                 "annual_report_url" => "/lab/eqa/rounds/#{created['id']}/annual_report.pdf")
      expect(created["items"].sole).to include("submitted" => "98.5", "target" => "100", "acceptable_low" => "90", "adequacy" => "inadequate")
      get "/lab/eqa/rounds/#{created['id']}/annual_report.pdf"
      expect([ response.media_type, response.body.b[0, 5] ]).to eq([ "application/pdf", "%PDF-" ])
      round("2026-02", [ item ], date: "2026-06-10")
      expect(json_body["annual_report_url"]).to be_nil
      get "/lab/eqa/summary", params: { provider_id: provider.id, year: 2026 }
      expect(json_body).to eq("year" => 2026, "rows" => [ { "analyte_key" => "glicose", "lab_exam_id" => exam.id, "rounds" => 2, "adequate" => 1,
                                                          "inadequate" => 1 } ])
      get "/lab/eqa/rounds", params: { provider_id: provider.id, year: 2026 }
      expect(json_body["rounds"].map { |r| r["round_code"] }).to eq(%w[2026-02 2026-01])
      get "/lab/eqa/rounds", params: { provider_id: provider.id, year: 2025 }
      expect(json_body["rounds"]).to eq([])
      expect(DomainEvent.where(name: "lab_eqa_round.recorded").map { |e| e.payload.keys }.uniq).to eq([ [ "lab_eqa_round_id" ] ])
    end

    it "recusas: ação no inadequado, vírgula, analito repetido, rodada repetida, data futura, arquivo que não é PDF ou já usado" do
      stepped_up!(lead)
      round("R1", [ item(adequacy: "inadequate") ])
      expect(json_body["error"]).to eq("action_required")
      round("R1", [ item(submitted: "98.5") ])
      expect(json_body["error"]).to eq("invalid_value")
      round("R1", [ item.merge(analyte_key: "ureia") ])
      expect(json_body["error"]).to eq("invalid_items")
      round("R1", [ item, item ])
      expect(json_body["error"]).to eq("invalid_items")
      round("R1", [ item ], date: (Time.zone.today + 1).iso8601)
      expect(json_body["error"]).to eq("invalid_round_date")
      round("R1", [ item ])
      round("R1", [ item ])
      expect(json_body["error"]).to eq("duplicate_round")
      json_post "/lab/eqa/uploads", content_type: "image/png", data: Base64.strict_encode64("\x89PNG\r\n\x1A\nimagem".b)
      expect(json_body["error"]).to eq("invalid_file")
      json_post "/lab/eqa/uploads", content_type: "application/pdf", data: pdf
      used = json_body["upload_id"]
      round("R3", [ item ], upload_id: used)
      round("R4", [ item ], upload_id: used)
      expect(json_body["error"]).to eq("invalid_file")
    end

    it "técnica não registra nem lê; CEQ desligado com lab_quality_control" do
      stepped_up!(lab_staff!)
      get "/lab/eqa/rounds", params: { provider_id: provider.id }
      expect(response).to have_http_status(:forbidden)
      json_post "/lab/eqa/uploads", content_type: "application/pdf", data: pdf
      expect(response).to have_http_status(:forbidden)
      Platform::Features.set!(city: City.find(Current.city.id), key: "lab_quality_control", enabled: false, maintainer: ledi_maintainer!)
      stepped_up!(lead)
      get "/lab/eqa/rounds", params: { provider_id: provider.id }
      expect(json_body["feature"]).to eq("lab_quality_control")
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar:** `rspec spec/requests/lab/eqa_spec.rb` → 404 nas rotas.

- [ ] **Step 3: Implemente.** `app/commands/lab/eqa/record_round.rb`:

  ```ruby
  # Rodada do CEQ (contrato §4; spec §5.10): programa (PNCQ, Controllab ou
  # texto), código único por laboratório e programa, data não futura, 1..60
  # itens de analitos numéricos de exames do laboratório (um item por
  # analito); números com vírgula; inadequado exige ação; relatório anual
  # opcional (PDF enviado por /lab/eqa/uploads, consumido aqui — a recusa
  # desfaz o consumo). Só acréscimo.
  module Lab
    module Eqa
      module RecordRound
        module_function

        def call(params, by:, now: Time.current)
          params = params.to_h.stringify_keys
          provider = params["provider_id"].to_s.match?(Exams::BaseController::UUID) ? Lab::Access.labs.find_by(id: params["provider_id"]) : nil
          return Result.fail(:invalid_provider) unless provider

          program = params["program"].is_a?(String) ? params["program"].strip : ""
          code = params["round_code"].is_a?(String) ? params["round_code"].strip : ""
          return Result.fail(:invalid_program) unless program.length.between?(1, 60)
          return Result.fail(:invalid_round_code) unless code.length.between?(1, 40)

          date = parse_date(params["round_date"])
          return Result.fail(:invalid_round_date) unless date && date <= now.in_time_zone.to_date

          items = parse_items(params["items"], provider)
          return items if items.is_a?(Result)

          result = nil
          ApplicationRecord.transaction(requires_new: true) do
            provider.lock!
            next result = Result.fail(:duplicate_round) if LabEqaRound.exists?(exam_provider_id: provider.id, program: program, round_code: code)

            file = nil
            if params["annual_report_upload_id"].present?
              file = Exams::Files.claim!(file_id: params["annual_report_upload_id"], user: by)
              unless file&.content_type == "application/pdf"
                result = Result.fail(:invalid_file)
                raise ActiveRecord::Rollback # desfaz o consumo do upload que não serve
              end
            end
            round = LabEqaRound.create!(exam_provider: provider, program: program, round_code: code, round_date: date, items: items,
                                        annual_report_file: file, recorded_by_user: by, created_at: now)
            DomainEvents.publish("lab_eqa_round.recorded", lab_eqa_round_id: round.id)
            result = Result.ok(round: round)
          end
          result
        end

        def parse_items(raw, provider)
          return Result.fail(:invalid_items) unless raw.is_a?(Array) && raw.size.between?(1, 60)

          exams = LabExam.where(exam_provider_id: provider.id).includes(:current_version).index_by(&:id)
          items = raw.map do |entry|
            entry = entry.to_unsafe_h if entry.respond_to?(:to_unsafe_h)
            return Result.fail(:invalid_items) unless entry.is_a?(Hash)

            entry = entry.stringify_keys
            analyte = exams[entry["lab_exam_id"]]&.current_version&.analyte(entry["analyte_key"])
            return Result.fail(:invalid_items) unless analyte && analyte["value_type"] == "numeric"
            return Result.fail(:invalid_items) unless LabEqaRound::ADEQUACY.include?(entry["adequacy"])

            submitted = Lab::Qc::Decimal.typed(entry["submitted"])
            target = Lab::Qc::Decimal.typed(entry["target"])
            low = entry["acceptable_low"].nil? ? nil : Lab::Qc::Decimal.typed(entry["acceptable_low"])
            high = entry["acceptable_high"].nil? ? nil : Lab::Qc::Decimal.typed(entry["acceptable_high"])
            return Result.fail(:invalid_value) unless submitted && target
            return Result.fail(:invalid_value) if (!entry["acceptable_low"].nil? && low.nil?) || (!entry["acceptable_high"].nil? && high.nil?)
            return Result.fail(:invalid_value) if low && high && low > high

            action = entry["action"].is_a?(String) ? entry["action"].strip.presence : nil
            return Result.fail(:action_required) if entry["adequacy"] == "inadequate" && action.nil?
            return Result.fail(:invalid_items) if action && action.length > 500

            { "lab_exam_id" => entry["lab_exam_id"], "analyte_key" => analyte["key"], "submitted" => Lab::Qc::Decimal.out(submitted),
              "target" => Lab::Qc::Decimal.out(target), "acceptable_low" => Lab::Qc::Decimal.out(low),
              "acceptable_high" => Lab::Qc::Decimal.out(high), "adequacy" => entry["adequacy"], "action" => action }
          end
          pairs = items.map { |i| i.values_at("lab_exam_id", "analyte_key") }
          pairs.uniq.size == pairs.size ? items : Result.fail(:invalid_items)
        end

        def parse_date(raw)
          raw.is_a?(String) && raw.match?(/\A\d{4}-\d{2}-\d{2}\z/) ? Date.iso8601(raw) : nil
        rescue Date::Error
          nil
        end
        private_class_method :parse_items, :parse_date
      end
    end
  end
  ```

  `app/services/lab/eqa/summary.rb`:

  ```ruby
  # Adequação do CEQ por analito no ano (contrato §4; forma C3 do dashboard):
  # rodadas em que o analito apareceu, adequadas e inadequadas.
  module Lab
    module Eqa
      module Summary
        module_function

        def call(provider_id:, year:)
          items = LabEqaRound.where(exam_provider_id: provider_id, round_date: Date.new(year, 1, 1)..Date.new(year, 12, 31)).flat_map(&:items)
          rows = items.group_by { |i| [ i["analyte_key"], i["lab_exam_id"] ] }.map do |(key, exam_id), list|
            adequate = list.count { |i| i["adequacy"] == "adequate" }
            { analyte_key: key, lab_exam_id: exam_id, rounds: list.size, adequate: adequate, inadequate: list.size - adequate }
          end
          { year: year, rows: rows.sort_by { |r| [ r[:analyte_key], r[:lab_exam_id] ] } }
        end
      end
    end
  end
  ```

  `app/services/lab/eqa/json.rb`:

  ```ruby
  # Forma da rodada do CEQ (contrato §4; C12): o relatório anual sai pela rota do próprio CEQ.
  module Lab
    module Eqa
      module Json
        module_function

        def round(r)
          { id: r.id, provider_id: r.exam_provider_id, program: r.program, round_code: r.round_code, round_date: r.round_date.iso8601,
            items: r.items, annual_report_url: r.annual_report_file_id && "/lab/eqa/rounds/#{r.id}/annual_report.pdf",
            recorded_by: Screenings::Json.staff_name(r.recorded_by_user), recorded_at: r.created_at.iso8601 }
        end
      end
    end
  end
  ```

  `app/controllers/lab/eqa/rounds_controller.rb`:

  ```ruby
  # /lab/eqa (contrato §4; spec §8; C12): liberador ligado ao laboratório (RT
  # incluso) ou admin; escrita com step-up. O PDF do relatório anual: envio
  # (só PDF, até 10 MB, cifrado como os do 21a) e leitura. Sem trilha de
  # leitura: o relatório anual do programa externo não tem dado de paciente.
  module Lab
    module Eqa
      class RoundsController < Lab::Qc::BaseController
        before_action :require_step_up!, only: %i[create upload]
        before_action -> { provider!(allow: :releaser?) }, only: %i[index summary]
        before_action :load_round, only: %i[show annual_report]

        def index
          rounds = LabEqaRound.where(exam_provider_id: @provider.id)
          rounds = rounds.where(round_date: Date.new(params[:year].to_i, 1, 1)..Date.new(params[:year].to_i, 12, 31)) if params[:year].to_s.match?(/\A\d{4}\z/)
          render json: { rounds: rounds.includes(:recorded_by_user).order(round_date: :desc).limit(200).map { |r| Json.round(r) } }
        end

        def show = render(json: Json.round(@round))

        def annual_report
          file = @round.annual_report_file
          return not_found unless file

          send_data file.bytes, type: "application/pdf", disposition: "inline", filename: "ceq-#{@round.round_code.parameterize}-relatorio-anual.pdf"
        end

        def upload
          return forbid unless Lab::Access.releaser_provider_ids(Current.user).any? || Lab::Access.admin?(Current.user)
          return render_failure(Result.fail(:invalid_file)) unless params[:content_type] == "application/pdf"

          result = Exams::Files.store_base64(data: params[:data], content_type: "application/pdf", user: Current.user)
          return render_failure(result) if result.failure?

          render json: { upload_id: result.payload[:file].id }, status: :created
        end

        def create
          provider = lab_param
          return render_failure(Result.fail(:invalid_provider)) unless provider
          return forbid unless releaser?(provider.id)

          result = RecordRound.call(body_params, by: Current.user)
          return render_failure(result) if result.failure?

          render json: Json.round(result.payload[:round]), status: :created
        end

        def summary
          year = params[:year].to_s.match?(/\A\d{4}\z/) ? params[:year].to_i : Time.zone.today.year
          render json: Summary.call(provider_id: @provider.id, year: year)
        end

        private

        def load_round
          @round = uuid?(params[:id]) && LabEqaRound.find_by(id: params[:id])
          return not_found unless @round

          forbid unless releaser?(@round.exam_provider_id)
        end
      end
    end
  end
  ```

  `config/routes.rb` — dentro de `scope "/lab"`, depois do `scope "qc"`:

  ```ruby
      # Módulo 21c (ADR 0038): CEQ. As literais antes de `:id`.
      scope "eqa", module: :eqa, as: :eqa do
        get  "summary",                         to: "rounds#summary"
        post "uploads",                         to: "rounds#upload"
        get  "rounds",                          to: "rounds#index"
        post "rounds",                          to: "rounds#create"
        get  "rounds/:id/annual_report.pdf",    to: "rounds#annual_report", format: false
        get  "rounds/:id",                      to: "rounds#show"
      end
  ```

- [ ] **Step 4: Rode:** `rspec spec/requests/lab/eqa_spec.rb` → verde.

- [ ] **Step 5: Commit** e mova o card F-21.31 para **Done**:

  ```bash
  git add app/commands/lab/eqa/record_round.rb app/services/lab/eqa/summary.rb app/services/lab/eqa/json.rb app/controllers/lab/eqa/rounds_controller.rb config/routes.rb spec/requests/lab/eqa_spec.rb
  git commit -m "feat: record external quality assessment rounds with annual report

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

---


## Fatia 6 — Estoque, lote em uso, rastreio e consumo

### Task 13: Itens, lotes, movimentos, saldo e alertas (F-21.32)

**Files:**
- Create: `app/controllers/lab/inventory/{base_controller,supplies_controller,lots_controller,movements_controller,alerts_controller}.rb`, `app/commands/lab/inventory/{save_supply,receive_lot,record_movement}.rb`, `app/services/lab/inventory/{json,alerts}.rb`, `spec/requests/lab/inventory_spec.rb`, `spec/commands/lab/inventory_concurrency_spec.rb`
- Modify: `config/routes.rb`

**Interfaces:**
- Consumes: `LabSupply`, `LabSupplyLot`, `LabSupplyMovement` (Task 3; trigger de saldo); `Lab::Access`.
- Produces: `Lab::Inventory::BaseController` (403 `feature_disabled` com `lab_inventory` desligado; `lab_param`, `reader?` = membro ou admin, `admin?`, `provider!(allow:)`); `Lab::Inventory::SaveSupply.call(params, provider: nil, supply: nil) → Result(supply:)`; `Lab::Inventory::ReceiveLot.call(params, by:, now:) → Result(lot:)`; `Lab::Inventory::RecordMovement.call(lot:, kind:, quantity:, reason:, by:, admin:, now:) → Result(movement:, balance:)`; `Lab::Inventory::Alerts.call(provider_id:, now:) → { expiring:, below_minimum:, untraced_results: }`; `Lab::Inventory::Json.{supply,lot,movement}`. Rotas `GET|POST /lab/inventory/supplies`, `PATCH /lab/inventory/supplies/:id`, `GET|POST /lab/inventory/lots`, `GET|POST /lab/inventory/movements`, `GET /lab/inventory/alerts`.

- [ ] **Step 1: Testes que falham.** `spec/requests/lab/inventory_spec.rb`:

  ```ruby
  require "rails_helper"

  # Contrato §5 (F-21.32): itens (admin), entrada de lote (vencido recusado),
  # abertura, perda, vencimento e ajuste (só admin) com motivo; saldo = soma
  # dos movimentos e nunca negativo; alertas sem trilha (vencendo, abaixo do
  # mínimo, resultados sem rastreio).
  RSpec.describe "Estoque do laboratório", type: :request do
    before { inventory_city! }

    let(:provider) { lab_provider! }
    let(:exam) { lab_exam!(provider: provider) }
    let(:admin) { lab_admin! }
    let(:staff) { lab_staff! }
    let(:supply_body) do
      { provider_id: provider.id, name: "Glicose GOD-PAP", kind: "reagent", unit: "kit", tests_per_package: 500, minimum_stock: 3,
        expiry_alert_days: 30, lab_exam_ids: [ exam.id ], active: true }
    end

    def create_supply(body = supply_body)
      json_post "/lab/inventory/supplies", body
      json_body
    end

    def receive(supply_id, lot_number: "GL2609", expires_on: (Time.zone.today + 240).iso8601, quantity: 4)
      json_post "/lab/inventory/lots", supply_id: supply_id, lot_number: lot_number, expires_on: expires_on, supplier: "Distribuidora Paraná",
                                       origin: "purchase", quantity: quantity, document: "NF 4471"
      json_body
    end

    it "admin cadastra itens; recusas de tipo, rendimento e exames" do
      stepped_up!(admin)
      supply = create_supply
      expect(response).to have_http_status(:created)
      expect(supply).to include("kind" => "reagent", "tests_per_package" => 500, "balance" => 0, "below_minimum" => true)
      create_supply(supply_body.merge(name: "Outro", kind: "consumable"))
      expect(json_body["error"]).to eq("invalid_tests_per_package") # só reagente tem rendimento
      create_supply(supply_body.merge(name: "Outro", tests_per_package: nil))
      expect(json_body["error"]).to eq("invalid_tests_per_package")
      create_supply(supply_body.merge(name: "Outro", kind: "solvente"))
      expect(json_body["error"]).to eq("invalid_kind")
      create_supply(supply_body.merge(name: "Outro", lab_exam_ids: [ lab_exam!(provider: lab_provider!).id ]))
      expect(json_body["error"]).to eq("invalid_exams")
      create_supply
      expect(json_body["error"]).to eq("duplicate_name")
      json_patch "/lab/inventory/supplies/#{supply['id']}", supply_body.except(:provider_id).merge(minimum_stock: 2)
      expect(json_body["minimum_stock"]).to eq(2)
      sign_in_as(staff)
      create_supply(supply_body.merge(name: "Da técnica"))
      expect(response).to have_http_status(:unauthorized) # sem step-up
      stepped_up!(staff)
      create_supply(supply_body.merge(name: "Da técnica"))
      expect(response).to have_http_status(:forbidden)
    end

    it "entrada, abertura, perda, vencimento, ajuste; saldo nunca negativo; motivos" do
      stepped_up!(admin)
      supply = create_supply
      sign_in_as(staff)
      lot = receive(supply["id"])
      expect(response).to have_http_status(:created)
      expect(lot).to include("lot_number" => "GL2609", "balance" => 4, "origin" => "purchase", "expired" => false)
      again = receive(supply["id"], quantity: 2)
      expect([ again["id"], again["balance"] ]).to eq([ lot["id"], 6 ]) # nova entrada do mesmo lote
      receive(supply["id"], expires_on: (Time.zone.today + 100).iso8601)
      expect(json_body["error"]).to eq("duplicate_lot") # mesmo número com outra validade
      receive(supply["id"], lot_number: "GL2508", expires_on: (Time.zone.today - 1).iso8601)
      expect(json_body["error"]).to eq("expired_lot")
      receive(supply["id"], lot_number: "GL2610", quantity: 0)
      expect(json_body["error"]).to eq("invalid_quantity")

      json_post "/lab/inventory/movements", lot_id: lot["id"], kind: "opening", quantity: 2
      expect(json_body).to include("kind" => "opening", "quantity" => -2, "balance" => 4)
      json_post "/lab/inventory/movements", lot_id: lot["id"], kind: "loss", quantity: 1, reason: ""
      expect(json_body["error"]).to eq("reason_required")
      json_post "/lab/inventory/movements", lot_id: lot["id"], kind: "loss", quantity: 1, reason: "Frasco quebrado"
      expect(json_body["balance"]).to eq(3)
      json_post "/lab/inventory/movements", lot_id: lot["id"], kind: "opening", quantity: 9
      expect([ response.status, json_body["error"] ]).to eq([ 422, "insufficient_balance" ])
      json_post "/lab/inventory/movements", lot_id: lot["id"], kind: "adjustment", quantity: -1, reason: "Contagem"
      expect(response).to have_http_status(:forbidden) # ajuste só admin
      json_post "/lab/inventory/movements", lot_id: lot["id"], kind: "receipt", quantity: 1
      expect(json_body["error"]).to eq("invalid_kind")
      stepped_up!(admin)
      json_post "/lab/inventory/movements", lot_id: lot["id"], kind: "adjustment", quantity: -1, reason: "Contagem de inventário"
      expect(json_body["balance"]).to eq(2)
      json_post "/lab/inventory/movements", lot_id: lot["id"], kind: "adjustment", quantity: -5, reason: "Contagem"
      expect(json_body["error"]).to eq("insufficient_balance")
      get "/lab/inventory/movements", params: { lot_id: lot["id"] }
      expect(json_body["movements"].map { |m| [ m["kind"], m["quantity"] ] })
        .to eq([ [ "adjustment", -1 ], [ "loss", -1 ], [ "opening", -2 ], [ "receipt", 2 ], [ "receipt", 4 ] ])
      get "/lab/inventory/lots", params: { supply_id: supply["id"], with_balance: "true" }
      expect(json_body["lots"].map { |l| [ l["lot_number"], l["balance"] ] }).to eq([ [ "GL2609", 2 ] ])
      expect(DomainEvent.where(name: "lab_supply_movement.recorded").map { |e| e.payload.keys.sort }.uniq).to eq([ %w[kind lab_supply_movement_id] ])
      expect(DomainEvent.where(name: "lab_supply_lot.received").count).to eq(2)
    end

    it "alertas: vencendo, abaixo do mínimo e resultados sem rastreio; sem trilha" do
      stepped_up!(admin)
      supply = create_supply
      receive(supply["id"], lot_number: "GL2603", expires_on: (Time.zone.today + 20).iso8601, quantity: 1)
      receive(supply["id"], lot_number: "GL2609", quantity: 1)
      instrument = qc_instrument!(provider)
      tracked = qc_exam!(provider: provider, instrument: instrument, sigtap_code: "0202010317", name: "Creatinina", instrument_test_code: "CRE",
                         analytes: [ LabHelpers::GLUCOSE.first.merge("key" => "creatinina", "name" => "Creatinina", "decimals" => 2, "entry_min" => nil,
                                                                    "entry_max" => nil, "critical_low" => nil, "critical_high" => nil) ])
      req = received_requisition!(provider: provider, exams: [ tracked ])
      type!(req.items.first, { creatinina: "0,9" })
      sign_in_as(staff)
      expect { get "/lab/inventory/alerts", params: { provider_id: provider.id } }.not_to(change { DomainEvent.count })
      expect(json_body["expiring"].map { |l| l.values_at("lot_number", "supply_name", "balance") }).to eq([ [ "GL2603", "Glicose GOD-PAP", 1 ] ])
      expect(json_body["expiring"].first["lot_id"]).to be_present
      expect(json_body["below_minimum"].map { |x| x.values_at("supply_id", "name", "unit", "balance", "minimum_stock") })
        .to eq([ [ supply["id"], "Glicose GOD-PAP", "kit", 2, 3 ] ])
      expect(json_body["untraced_results"]).to eq(1)
    end

    it "estoque desligado: 403" do
      Platform::Features.set!(city: City.find(Current.city.id), key: "lab_inventory", enabled: false, maintainer: ledi_maintainer!)
      sign_in_as(staff)
      get "/lab/inventory/alerts", params: { provider_id: provider.id }
      expect(json_body).to eq("error" => "feature_disabled", "feature" => "lab_inventory")
    end
  end
  ```

  `spec/commands/lab/inventory_concurrency_spec.rb` (Review Focus 4):

  ```ruby
  require "rails_helper"

  # Review Focus 4: duas aberturas ao mesmo tempo do último kit — a trava do
  # lote serializa; uma passa, a outra leva insufficient_balance; saldo 0.
  RSpec.describe "Saídas concorrentes do estoque" do
    self.use_transactional_tests = false

    let(:threads) { [] }

    def in_city(&) = CityConnection.with(TEST_CITY_A) { Current.set(city: TEST_CITY_A, &) }

    before do
      @rows = in_city do
        @empty = ApplicationRecord.connection.tables.reject { |t| %w[schema_migrations ar_internal_metadata].include?(t) }
                                  .select { |t| ApplicationRecord.connection.select_value("SELECT NOT EXISTS (SELECT 1 FROM #{t})") }
        provider = lab_provider!
        staff = lab_staff!
        supply = LabSupply.create!(exam_provider: provider, name: "Creatinina Jaffé", kind: "reagent", unit: "kit", tests_per_package: 300,
                                   minimum_stock: 1, expiry_alert_days: 30, lab_exam_ids: [])
        lot = Lab::Inventory::ReceiveLot.call({ "supply_id" => supply.id, "lot_number" => "CR1", "expires_on" => (Time.zone.today + 90).iso8601,
                                                "supplier" => "Distribuidora", "origin" => "purchase", "quantity" => 1 }, by: staff).payload[:lot]
        { lot_id: lot.id, staff: [ staff.id, lab_staff!.id ] }
      end
    end

    after do
      threads.each { |t| t.join(5) || t.kill }
      in_city do
        ApplicationRecord.transaction do
          ApplicationRecord.connection.execute("SET LOCAL session_replication_role = replica")
          @empty.to_a.each { |table| ApplicationRecord.connection.execute("DELETE FROM #{ApplicationRecord.connection.quote_table_name(table)}") }
        end
      end
    end

    it "uma abertura passa, a outra é recusada; o saldo fica 0" do
      start = Queue.new
      out = Queue.new
      @rows[:staff].each do |user_id|
        threads << Thread.new do
          in_city do
            start.pop(timeout: 10)
            out << Lab::Inventory::RecordMovement.call(lot: LabSupplyLot.find(@rows[:lot_id]), kind: "opening", quantity: 1, reason: nil,
                                                       by: User.find(user_id), admin: false)
          end
        end
      end
      2.times { start << true }
      results = [ out.pop(timeout: 10), out.pop(timeout: 10) ]
      expect(results.map { |r| r.ok? ? :ok : r.reason }).to contain_exactly(:ok, :insufficient_balance)
      in_city { expect(LabSupplyLot.find(@rows[:lot_id]).balance).to eq(0) }
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar:** `rspec spec/requests/lab/inventory_spec.rb spec/commands/lab/inventory_concurrency_spec.rb` → 404 e `NameError: Lab::Inventory::ReceiveLot`.

- [ ] **Step 3: Base, JSON e alertas.** `app/controllers/lab/inventory/base_controller.rb`:

  ```ruby
  # Base das rotas /lab/inventory/* (contrato §5; ADR 0038 §8): municipal_lab
  # (herdado) e lab_inventory utilizáveis. Lê e movimenta: membro do
  # laboratório (lab_staff, liberador) ou admin; cadastro e ajuste: admin.
  module Lab
    module Inventory
      class BaseController < Lab::BaseController
        before_action :require_inventory!

        private

        def require_inventory!
          return if Lab::Gate.inventory_usable?(Current.city)

          render json: { error: "feature_disabled", feature: Lab::Gate::INVENTORY_KEY }, status: :forbidden
        end

        def lab_param(key = :provider_id) = uuid?(params[key]) ? Lab::Access.labs.find_by(id: params[key]) : nil
        def admin? = Lab::Access.admin?(Current.user)
        def reader?(provider_id) = Lab::Access.member?(Current.user, provider_id) || admin?

        def provider!(key = :provider_id, allow: :reader?)
          @provider = lab_param(key)
          return not_found unless @provider

          forbid unless send(allow, @provider.id)
        end
      end
    end
  end
  ```

  `app/services/lab/inventory/json.rb`:

  ```ruby
  # Formas do estoque (contrato §5). Quantidades inteiras (embalagens), saldo = soma dos movimentos.
  module Lab
    module Inventory
      module Json
        module_function

        def supply(s, balance: nil)
          balance ||= LabSupplyMovement.joins(:lab_supply_lot).where(lab_supply_lots: { lab_supply_id: s.id }).sum(:quantity)
          { id: s.id, provider_id: s.exam_provider_id, name: s.name, kind: s.kind, unit: s.unit, tests_per_package: s.tests_per_package,
            minimum_stock: s.minimum_stock, expiry_alert_days: s.expiry_alert_days, lab_exam_ids: s.lab_exam_ids, active: s.active,
            balance: balance, below_minimum: balance < s.minimum_stock }
        end

        def lot(l, balance: l.balance)
          { id: l.id, supply_id: l.lab_supply_id, supply_name: l.lab_supply.name, lot_number: l.lot_number, expires_on: l.expires_on.iso8601,
            supplier: l.supplier,
            origin: l.origin, balance: balance, expired: l.expired?,
            in_use: LabSupplyInUse.current.where(lab_supply_lot_id: l.id).map { |u| { instrument_id: u.lab_instrument_id, lab_exam_id: u.lab_exam_id } } }
        end

        def movement(m, balance: nil)
          body = { id: m.id, lot_id: m.lab_supply_lot_id, kind: m.kind, quantity: m.quantity, reason: m.reason, document: m.document,
                   recorded_at: m.recorded_at.iso8601, recorded_by: Screenings::Json.staff_name(m.recorded_by_user) }
          body[:balance] = balance unless balance.nil?
          body
        end
      end
    end
  end
  ```

  `app/services/lab/inventory/alerts.rb`:

  ```ruby
  # Alertas do estoque (spec §6.6; contrato §5), sem trilha: lotes com saldo
  # vencendo dentro do expiry_alert_days do item (ou já vencidos); itens
  # ativos abaixo do mínimo (saldo dos lotes não vencidos); resultados dos
  # últimos 30 dias de exame com aparelho sem lote de reagente (sem rastreio).
  module Lab
    module Inventory
      module Alerts
        UNTRACED_WINDOW = 30.days

        module_function

        def call(provider_id:, now: Time.current)
          today = now.in_time_zone.to_date
          balances = LabSupplyMovement.joins(lab_supply_lot: :lab_supply).where(lab_supplies: { exam_provider_id: provider_id })
                                      .group(:lab_supply_lot_id).sum(:quantity)
          lots = LabSupplyLot.where(id: balances.select { |_, b| b.positive? }.keys).includes(:lab_supply).to_a
          expiring = lots.select { |l| l.expires_on <= today + l.lab_supply.expiry_alert_days }.sort_by(&:expires_on)
          available = lots.reject { |l| l.expired?(today) }.group_by(&:lab_supply_id).transform_values { |ls| ls.sum { |l| balances[l.id] } }
          below = LabSupply.where(exam_provider_id: provider_id, active: true).order(:name).select { |s| available.fetch(s.id, 0) < s.minimum_stock }
          untraced = LabResult.joins(lab_requisition_item: [ :lab_requisition, :lab_exam_version ])
                              .where(lab_requisitions: { exam_provider_id: provider_id }, reagent_lot_id: nil, chosen: false)
                              .where.not(lab_exam_versions: { lab_instrument_id: nil })
                              .where("lab_results.created_at >= ?", now - UNTRACED_WINDOW).count
          { expiring: expiring.map { |l| Json.lot(l, balance: balances[l.id]).merge(lot_id: l.id) }, # forma C3: lot_id
            below_minimum: below.map { |s| Json.supply(s, balance: available.fetch(s.id, 0)).merge(supply_id: s.id) },
            untraced_results: untraced }
        end
      end
    end
  end
  ```

- [ ] **Step 4: Comandos.** `app/commands/lab/inventory/save_supply.rb`:

  ```ruby
  # Item de estoque (contrato §5): nome único no laboratório; tipo; unidade;
  # rendimento só (e sempre) para reagente; mínimo; dias de alerta; exames do
  # laboratório. Nunca apagado (desativar).
  module Lab
    module Inventory
      module SaveSupply
        module_function

        def call(params, provider: nil, supply: nil)
          params = params.to_h.stringify_keys
          provider ||= supply.exam_provider
          name = params["name"].is_a?(String) ? params["name"].strip : ""
          unit = params["unit"].is_a?(String) ? params["unit"].strip : ""
          return Result.fail(:invalid_name) unless name.length.between?(1, 120)
          return Result.fail(:invalid_kind) unless LabSupply::KINDS.include?(params["kind"])
          return Result.fail(:invalid_unit) unless unit.length.between?(1, 20)

          tests = params["tests_per_package"]
          reagent = params["kind"] == "reagent"
          return Result.fail(:invalid_tests_per_package) unless reagent ? (tests.is_a?(Integer) && tests.between?(1, 100_000)) : tests.nil?
          return Result.fail(:invalid_minimum_stock) unless params["minimum_stock"].is_a?(Integer) && params["minimum_stock"].between?(0, 100_000)

          alert_days = params.fetch("expiry_alert_days", 30)
          return Result.fail(:invalid_expiry_alert_days) unless alert_days.is_a?(Integer) && alert_days.between?(1, 365)

          exam_ids = params["lab_exam_ids"]
          valid_exams = exam_ids.is_a?(Array) && exam_ids.all?(String) && exam_ids.uniq.size == exam_ids.size &&
                        LabExam.where(exam_provider_id: provider.id, id: exam_ids).count == exam_ids.size
          return Result.fail(:invalid_exams) unless valid_exams
          return Result.fail(:invalid_active) unless [ true, false ].include?(params["active"])

          ApplicationRecord.transaction(requires_new: true) do
            provider.lock!
            next Result.fail(:duplicate_name) if LabSupply.where(exam_provider_id: provider.id).where.not(id: supply&.id).where("lower(name) = ?", name.downcase).exists?

            record = supply || LabSupply.new(exam_provider: provider)
            record.update!(name: name, kind: params["kind"], unit: unit, tests_per_package: tests, minimum_stock: params["minimum_stock"],
                           expiry_alert_days: alert_days, lab_exam_ids: exam_ids, active: params["active"])
            Result.ok(supply: record)
          end
        end
      end
    end
  end
  ```

  `app/commands/lab/inventory/receive_lot.rb`:

  ```ruby
  # Entrada de lote (spec §6.2; contrato §5): número, validade (vencido
  # recusado), fornecedor, origem, quantidade (1..100000) e documento
  # opcional. Mesmo número com a mesma validade = nova entrada no lote
  # existente; com outra validade = duplicate_lot. Item travado.
  module Lab
    module Inventory
      module ReceiveLot
        module_function

        def call(params, by:, now: Time.current)
          params = params.to_h.stringify_keys
          supply = params["supply_id"].to_s.match?(Exams::BaseController::UUID) ? LabSupply.find_by(id: params["supply_id"]) : nil
          return Result.fail(:not_found) unless supply

          number = params["lot_number"].is_a?(String) ? params["lot_number"].strip : ""
          supplier = params["supplier"].is_a?(String) ? params["supplier"].strip : ""
          document = params["document"].is_a?(String) ? params["document"].strip.presence : nil
          return Result.fail(:invalid_lot_number) unless number.length.between?(1, 40)
          return Result.fail(:invalid_supplier) unless supplier.length.between?(1, 120)
          return Result.fail(:invalid_origin) unless LabSupplyLot::ORIGINS.include?(params["origin"])
          return Result.fail(:invalid_quantity) unless params["quantity"].is_a?(Integer) && params["quantity"].between?(1, 100_000)
          return Result.fail(:invalid_document) if document && document.length > 60

          expires_on = parse_date(params["expires_on"])
          return Result.fail(:invalid_expires_on) unless expires_on
          return Result.fail(:expired_lot) if expires_on < now.in_time_zone.to_date

          ApplicationRecord.transaction(requires_new: true) do
            supply.lock!
            lot = LabSupplyLot.find_by(lab_supply_id: supply.id, lot_number: number)
            next Result.fail(:duplicate_lot) if lot && lot.expires_on != expires_on

            lot ||= LabSupplyLot.create!(lab_supply: supply, lot_number: number, expires_on: expires_on, supplier: supplier,
                                         origin: params["origin"], created_by_user: by, created_at: now)
            lot.lock!
            LabSupplyMovement.create!(lab_supply_lot: lot, kind: "receipt", quantity: params["quantity"], document: document,
                                      recorded_by_user: by, recorded_at: now, created_at: now)
            DomainEvents.publish("lab_supply_lot.received", lab_supply_lot_id: lot.id, lab_supply_id: supply.id)
            Result.ok(lot: lot)
          end
        end

        def parse_date(raw)
          raw.is_a?(String) && raw.match?(/\A\d{4}-\d{2}-\d{2}\z/) ? Date.iso8601(raw) : nil
        rescue Date::Error
          nil
        end
      end
    end
  end
  ```

  `app/commands/lab/inventory/record_movement.rb`:

  ```ruby
  # Saída e acerto (spec §6.3–§6.5; contrato §5): abertura, perda e
  # vencimento recebem quantidade positiva e gravam negativa; ajuste (só
  # admin) aceita sinal; perda, vencimento e ajuste exigem motivo; saldo
  # nunca negativo, conferido com o lote travado (e o trigger como cinto).
  module Lab
    module Inventory
      module RecordMovement
        KINDS = %w[opening loss expiry adjustment].freeze

        module_function

        def call(lot:, kind:, quantity:, reason:, by:, admin:, now: Time.current)
          return Result.fail(:invalid_kind) unless KINDS.include?(kind)
          return Result.fail(:missing_role) if kind == "adjustment" && !admin

          valid = quantity.is_a?(Integer) && quantity.abs.between?(1, 100_000) && (kind == "adjustment" || quantity.positive?)
          return Result.fail(:invalid_quantity) unless valid

          text = reason.is_a?(String) ? reason.strip.presence : nil
          return Result.fail(:reason_required) if kind != "opening" && text.nil?
          return Result.fail(:reason_required) if text && text.length > 500

          signed = kind == "adjustment" ? quantity : -quantity
          ApplicationRecord.transaction(requires_new: true) do
            lot.lock!
            balance = lot.balance
            next Result.fail(:insufficient_balance) if balance + signed < 0

            movement = LabSupplyMovement.create!(lab_supply_lot: lot, kind: kind, quantity: signed, reason: text, recorded_by_user: by,
                                                 recorded_at: now, created_at: now)
            DomainEvents.publish("lab_supply_movement.recorded", lab_supply_movement_id: movement.id, kind: kind)
            Result.ok(movement: movement, balance: balance + signed)
          end
        end
      end
    end
  end
  ```

- [ ] **Step 5: Controllers.** `app/controllers/lab/inventory/supplies_controller.rb`:

  ```ruby
  # /lab/inventory/supplies (contrato §5): lê membro ou admin; cadastro só admin, com step-up.
  module Lab
    module Inventory
      class SuppliesController < BaseController
        before_action :require_step_up!, only: %i[create update]
        before_action -> { provider! }, only: :index

        def index
          render json: { supplies: LabSupply.where(exam_provider_id: @provider.id).order(:name).map { |s| Json.supply(s) } }
        end

        def create
          return forbid unless admin?

          provider = lab_param
          return render_failure(Result.fail(:invalid_provider)) unless provider

          respond(SaveSupply.call(body_params, provider: provider), :created)
        end

        def update
          return forbid unless admin?

          supply = uuid?(params[:id]) && LabSupply.find_by(id: params[:id])
          return not_found unless supply

          respond(SaveSupply.call(body_params, supply: supply), :ok)
        end

        private

        def respond(result, status)
          return render_failure(result) if result.failure?

          render json: Json.supply(result.payload[:supply]), status: status
        end
      end
    end
  end
  ```

  `app/controllers/lab/inventory/lots_controller.rb`:

  ```ruby
  # /lab/inventory/lots (contrato §5): lotes do item (ou do laboratório), com
  # saldo; entrada de lote por membro do laboratório ou admin.
  module Lab
    module Inventory
      class LotsController < BaseController
        def index
          scope = if uuid?(params[:supply_id]) && (supply = LabSupply.find_by(id: params[:supply_id]))
                    return forbid unless reader?(supply.exam_provider_id)

                    LabSupplyLot.where(lab_supply_id: supply.id)
                  elsif (provider = lab_param)
                    return forbid unless reader?(provider.id)

                    LabSupplyLot.joins(:lab_supply).where(lab_supplies: { exam_provider_id: provider.id })
                  else
                    return render_failure(Result.fail(:filter_required))
                  end
          balances = LabSupplyMovement.where(lab_supply_lot_id: scope.select(:id)).group(:lab_supply_lot_id).sum(:quantity)
          lots = scope.includes(:lab_supply).order(:expires_on, :lot_number).to_a
          lots = lots.select { |l| balances.fetch(l.id, 0).positive? } if params[:with_balance] == "true"
          render json: { lots: lots.map { |l| Json.lot(l, balance: balances.fetch(l.id, 0)) } }
        end

        def create
          supply = uuid?(params[:supply_id]) && LabSupply.find_by(id: params[:supply_id])
          return render_failure(Result.fail(:not_found)) unless supply
          return forbid unless reader?(supply.exam_provider_id)

          result = ReceiveLot.call(body_params, by: Current.user)
          return render_failure(result) if result.failure?

          render json: Json.lot(result.payload[:lot].reload), status: :created
        end
      end
    end
  end
  ```

  `app/controllers/lab/inventory/movements_controller.rb`:

  ```ruby
  # /lab/inventory/movements (contrato §5): histórico do lote; saída e
  # acerto por membro ou admin (ajuste só admin, com step-up).
  module Lab
    module Inventory
      class MovementsController < BaseController
        before_action :load_lot

        def index
          # C8: do mais novo ao mais antigo.
          render json: { movements: @lot.movements.includes(:recorded_by_user).reorder(recorded_at: :desc, created_at: :desc).map { |m| Json.movement(m) } }
        end

        def create
          if params[:kind] == "adjustment"
            return forbid unless admin?
            return render(json: { error: "mfa_required" }, status: :unauthorized) unless reauthenticated_recently?
          end
          result = RecordMovement.call(lot: @lot, kind: params[:kind].to_s, quantity: params[:quantity], reason: params[:reason],
                                       by: Current.user, admin: admin?)
          return render_failure(result) if result.failure?

          render json: Json.movement(result.payload[:movement], balance: result.payload[:balance]), status: :created
        end

        private

        def load_lot
          @lot = uuid?(params[:lot_id]) && LabSupplyLot.includes(:lab_supply).find_by(id: params[:lot_id])
          return not_found unless @lot

          forbid unless reader?(@lot.lab_supply.exam_provider_id)
        end
      end
    end
  end
  ```
  (`reauthenticated_recently?` é o predicado do `MfaStepUp` que o `require_step_up!` usa; confira o nome no concern antes de usar.)

  `app/controllers/lab/inventory/alerts_controller.rb`:

  ```ruby
  # GET /lab/inventory/alerts (contrato §5): sem trilha (só contagens e lotes), para membros e admin.
  module Lab
    module Inventory
      class AlertsController < BaseController
        before_action -> { provider! }

        def show = render(json: Alerts.call(provider_id: @provider.id))
      end
    end
  end
  ```

  `config/routes.rb` — dentro de `scope "/lab"`, depois do `scope "eqa"`:

  ```ruby
      # Módulo 21c (ADR 0038): estoque. As literais antes de `:id`.
      scope "inventory", module: :inventory, as: :inventory do
        get   "supplies",      to: "supplies#index"
        post  "supplies",      to: "supplies#create"
        patch "supplies/:id",  to: "supplies#update"
        get   "lots",          to: "lots#index"
        post  "lots",          to: "lots#create"
        get   "movements",     to: "movements#index"
        post  "movements",     to: "movements#create"
        get   "alerts",        to: "alerts#show"
      end
  ```

- [ ] **Step 6: Rode:** `rspec spec/requests/lab/inventory_spec.rb spec/commands/lab/inventory_concurrency_spec.rb spec/models/lab_quality_schema_spec.rb` → verde.

- [ ] **Step 7: Commit** e mova o card F-21.32 para **Done**:

  ```bash
  git add app/controllers/lab/inventory/base_controller.rb app/controllers/lab/inventory/supplies_controller.rb app/controllers/lab/inventory/lots_controller.rb app/controllers/lab/inventory/movements_controller.rb app/controllers/lab/inventory/alerts_controller.rb app/commands/lab/inventory/save_supply.rb app/commands/lab/inventory/receive_lot.rb app/commands/lab/inventory/record_movement.rb app/services/lab/inventory/json.rb app/services/lab/inventory/alerts.rb config/routes.rb spec/requests/lab/inventory_spec.rb spec/commands/lab/inventory_concurrency_spec.rb
  git commit -m "feat: add lab supplies inventory with lots, movements and alerts

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 14: Lote em uso, lote no resultado e no controle, rastreio (F-21.33)

**Files:**
- Create: `app/commands/lab/inventory/set_in_use.rb`, `app/services/lab/inventory/trace.rb`, `app/controllers/lab/inventory/{in_use_controller,traces_controller}.rb`, `spec/requests/lab/inventory_trace_spec.rb`
- Modify: `app/commands/lab/{enter_results,choose_result}.rb`, `app/services/lab/entry_json.rb`, `app/services/lab/inventory/json.rb`, `config/routes.rb`

**Interfaces:**
- Consumes: `LabSupplyInUse.current_lot_id` (Task 3); `Lab::Inventory::BaseController`, `ReceiveLot` (Task 13); `Lab::Qc::RecordValue` (Task 7, já grava `reagent_lot_id`).
- Produces: `Lab::Inventory::SetInUse.call(lot_id:, instrument_id:, lab_exam_id:, by:, now:) → Result(in_use:)` (encerra o anterior do par; o mesmo lote de novo é idempotente); `Lab::Inventory::Trace.call(lot:) → { lot_id:, lot_number:, results: [{ requisition_number, sigtap_code, entered_at }], qc_values: [{ run_id, analyte_key, entered_at }] }`; `Lab::Inventory::Json.in_use(u) → { id, lot_id, lot_number, supply_name, instrument_id, lab_exam_id, exam_name, started_at, ended_at }` (forma C3); `<lab_result_row>` ganha `reagent_lot: { id, lot_number }|null` (contrato §7). Rotas `GET|POST /lab/inventory/in_use`, `GET /lab/inventory/lots/:id/trace`, `GET /lab/inventory/lots/:id/trace.csv`.

- [ ] **Step 1: Teste que falha** — `spec/requests/lab/inventory_trace_spec.rb`:

  ```ruby
  require "rails_helper"

  # Spec §6.3, §6.7 e contrato §5, §7 (F-21.33): lote em uso por aparelho e
  # exame (marcar outro encerra o anterior); resultado e valor de controle
  # guardam o lote em uso no momento em que entram (ou nada: "sem rastreio");
  # escolher repetição mantém o lote da medida; rastreio por lote sem nome de
  # paciente, com trilha, em JSON e CSV.
  RSpec.describe "Estoque — lote em uso e rastreio", type: :request do
    before { qc_city!(inventory: true) }

    let(:s) { qc_scenario! }
    let(:staff) { lab_staff! }
    let(:supply) do
      LabSupply.create!(exam_provider: s.provider, name: "Glicose GOD-PAP", kind: "reagent", unit: "kit", tests_per_package: 500, minimum_stock: 1,
                        expiry_alert_days: 30, lab_exam_ids: [ s.exam.id ])
    end

    def receive!(number, supply: self.supply)
      Lab::Inventory::ReceiveLot.call({ "supply_id" => supply.id, "lot_number" => number, "expires_on" => (Time.zone.today + 200).iso8601,
                                        "supplier" => "Distribuidora", "origin" => "purchase", "quantity" => 3 }, by: staff).payload[:lot]
    end

    def use!(lot, exam: s.exam)
      json_post "/lab/inventory/in_use", lot_id: lot.id, instrument_id: s.instrument.id, lab_exam_id: exam.id
    end

    it "marcar o lote em uso encerra o anterior; recusas de tipo, validade e exame" do
      first = receive!("GL1")
      second = receive!("GL2")
      sign_in_as(staff)
      use!(first)
      expect(response).to have_http_status(:created)
      expect(json_body).to include("lot_number" => "GL1", "supply_name" => "Glicose GOD-PAP", "exam_name" => "Glicose", "ended_at" => nil)
      use!(second)
      get "/lab/inventory/in_use", params: { instrument_id: s.instrument.id }
      expect(json_body["in_use"].map { |u| u["lot_number"] }).to eq([ "GL2" ])
      expect(LabSupplyInUse.where(lab_supply_lot_id: first.id).sole.ended_at).to be_present
      tube = LabSupply.create!(exam_provider: s.provider, name: "Tubo fluoreto", kind: "consumable", unit: "caixa", minimum_stock: 1,
                               expiry_alert_days: 30, lab_exam_ids: [ s.exam.id ])
      use!(receive!("TB1", supply: tube))
      expect(json_body["error"]).to eq("lot_not_reagent")
      other_exam = qc_exam!(provider: s.provider, instrument: s.instrument, sigtap_code: "0202010317", name: "Creatinina", instrument_test_code: "CRE")
      use!(second, exam: other_exam)
      expect(json_body["error"]).to eq("exam_not_linked")
      old = LabSupplyLot.create!(lab_supply: supply, lot_number: "GL0", expires_on: Time.zone.today - 1, supplier: "D", origin: "purchase",
                                 created_by_user: staff, created_at: Time.current)
      use!(old)
      expect(json_body["error"]).to eq("expired_lot")
      expect(DomainEvent.where(name: "lab_supply_in_use.changed").count).to eq(2)
    end

    it "resultado e controle guardam o lote em uso; repetição escolhida mantém o lote; sem lote = sem rastreio" do
      req = received_requisition!(provider: s.provider, exams: [ s.exam ])
      item = req.items.first
      type!(item, { glicose: "98" })
      expect(item.results.last.reagent_lot_id).to be_nil # antes de haver lote em uso
      lot = receive!("GL1")
      sign_in_as(staff)
      use!(lot)
      type!(item, { glicose: "99" })
      measured = item.results.reload.last
      expect(measured.reagent_lot_id).to eq(lot.id)
      get "/lab/entry", params: { item_id: item.id }
      expect(json_body["results"].map { |r| r["reagent_lot"] }).to eq([ nil, { "id" => lot.id, "lot_number" => "GL1" } ])
      value = qc_value!(s, "Nível 1", 0.1, by: staff)[:value]
      expect(value.reagent_lot_id).to eq(lot.id)
      use!(receive!("GL2"))
      lead = lab_releaser!(s.provider, lead: true)
      copy = Lab::ChooseResult.call(result: measured, by: lead).payload[:result]
      expect(copy.reagent_lot_id).to eq(lot.id) # a medida foi com o GL1

      get "/lab/inventory/lots/#{lot.id}/trace"
      expect(json_body["results"]).to eq([ { "requisition_number" => req.number, "sigtap_code" => "0202010473", "entered_at" => measured.entered_at.iso8601 } ])
      expect(json_body["qc_values"].map { |v| [ v["run_id"], v["analyte_key"] ] }).to eq([ [ value.lab_qc_run_id, "glicose" ] ])
      expect(DomainEvent.where(name: "lab_supply_lot.traced").map(&:payload)).to eq([ { "lab_supply_lot_id" => lot.id, "user_id" => staff.id } ])
      get "/lab/inventory/lots/#{lot.id}/trace.csv"
      expect(response.media_type).to eq("text/csv")
      expect(response.body.lines.first.strip).to eq("tipo,requisicao_ou_corrida,exame_ou_analito,data")
      expect(response.body).to include(req.number)
      expect(response.body).not_to include(req.patient.display_name)
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar:** `rspec spec/requests/lab/inventory_trace_spec.rb` → 404 e `reagent_lot_id` nulo no resultado.

- [ ] **Step 3: O lote no resultado.** Em `app/commands/lab/enter_results.rb`, no `LabResult.create!` de `create_row`, acrescente:

  ```ruby
                          reagent_lot_id: LabSupplyInUse.current_lot_id(instrument_id: item.lab_exam_version.lab_instrument_id,
                                                                        lab_exam_id: item.lab_exam_version.lab_exam_id),
  ```
  e no comentário de topo: "Lote de reagente (ADR 0038): cada linha guarda o lote em uso do par aparelho × exame no momento em que entra (nulo = sem rastreio)." Em `app/commands/lab/choose_result.rb`, no `LabResult.create!` da cópia, acrescente `reagent_lot_id: result.reagent_lot_id,` (a escolha não é medida nova: guarda o lote da medida escolhida). Em `app/services/lab/entry_json.rb`: em `item`, troque `item.results.includes(:entered_by_user)` por `item.results.includes(:entered_by_user, :reagent_lot)`; em `result_row`, depois de `effective: effective.include?(row.id)`, acrescente `, reagent_lot: row.reagent_lot && { id: row.reagent_lot.id, lot_number: row.reagent_lot.lot_number }`.

- [ ] **Step 4: Lote em uso e rastreio.** `app/commands/lab/inventory/set_in_use.rb`:

  ```ruby
  # Lote em uso (spec §6.3; contrato §5): só reagente, não vencido, de um
  # exame ligado ao item e de aparelho ativo do mesmo laboratório. Trava o
  # APARELHO (ordem do Desvio 10), encerra o lote corrente do par e abre o
  # novo; o mesmo lote de novo é idempotente.
  module Lab
    module Inventory
      module SetInUse
        module_function

        def call(lot_id:, instrument_id:, lab_exam_id:, by:, now: Time.current)
          lot = uuid?(lot_id) && LabSupplyLot.includes(:lab_supply).find_by(id: lot_id)
          return Result.fail(:not_found) unless lot

          supply = lot.lab_supply
          return Result.fail(:lot_not_reagent) unless supply.kind == "reagent"
          return Result.fail(:expired_lot) if lot.expired?(now.in_time_zone.to_date)

          instrument = uuid?(instrument_id) && LabInstrument.active.find_by(id: instrument_id, exam_provider_id: supply.exam_provider_id)
          return Result.fail(:invalid_instrument) unless instrument

          exam = uuid?(lab_exam_id) && LabExam.find_by(id: lab_exam_id, exam_provider_id: supply.exam_provider_id)
          return Result.fail(:exam_not_linked) unless exam && supply.lab_exam_ids.include?(exam.id)

          ApplicationRecord.transaction(requires_new: true) do
            instrument.lock!
            current = LabSupplyInUse.current.find_by(lab_instrument_id: instrument.id, lab_exam_id: exam.id)
            next Result.ok(in_use: current) if current&.lab_supply_lot_id == lot.id

            current&.update!(ended_at: now, ended_by_user: by)
            created = LabSupplyInUse.create!(lab_supply_lot: lot, lab_instrument: instrument, lab_exam: exam, started_at: now, started_by_user: by)
            DomainEvents.publish("lab_supply_in_use.changed", lab_supply_in_use_id: created.id, lab_instrument_id: instrument.id, lab_exam_id: exam.id)
            Result.ok(in_use: created)
          end
        end

        def uuid?(value) = value.is_a?(String) && value.match?(Exams::BaseController::UUID)
        private_class_method :uuid?
      end
    end
  end
  ```

  `app/services/lab/inventory/trace.rb`:

  ```ruby
  # Rastreio por lote (spec §6.7; contrato §5): resultados (número da
  # requisição, código do exame, data da entrada — uma linha por entrada, sem
  # as cópias da escolha) e valores de controle com aquele lote. Nunca o nome
  # do paciente. Até 5000 linhas por lista.
  module Lab
    module Inventory
      module Trace
        MAX = 5000
        CSV_HEADERS = %w[tipo requisicao_ou_corrida exame_ou_analito data].freeze

        module_function

        def call(lot:)
          results = LabResult.where(reagent_lot_id: lot.id, chosen: false)
                             .joins(lab_requisition_item: [ :lab_requisition, { lab_exam_version: :lab_exam } ])
                             .order("lab_results.entered_at").limit(MAX)
                             .pluck("lab_requisitions.number", "lab_exams.sigtap_code", "lab_results.entered_at").uniq
          qc = LabQcValue.where(reagent_lot_id: lot.id).order(:entered_at).limit(MAX).pluck(:lab_qc_run_id, :analyte_key, :entered_at)
          { lot_id: lot.id, lot_number: lot.lot_number,
            results: results.map { |number, code, at| { requisition_number: number, sigtap_code: code, entered_at: at.iso8601 } },
            qc_values: qc.map { |run_id, key, at| { run_id: run_id, analyte_key: key, entered_at: at.iso8601 } } }
        end
      end
    end
  end
  ```

  Acrescente a `app/services/lab/inventory/json.rb`:

  ```ruby
        def in_use(u)
          { id: u.id, lot_id: u.lab_supply_lot_id, lot_number: u.lab_supply_lot.lot_number, supply_name: u.lab_supply_lot.lab_supply.name,
            instrument_id: u.lab_instrument_id, lab_exam_id: u.lab_exam_id, exam_name: u.lab_exam.name, started_at: u.started_at.iso8601,
            ended_at: u.ended_at&.iso8601 }
        end
  ```

  `app/controllers/lab/inventory/in_use_controller.rb`:

  ```ruby
  # /lab/inventory/in_use (contrato §5): lotes em uso do aparelho (corrente);
  # marcar lote em uso: membro do laboratório ou admin.
  module Lab
    module Inventory
      class InUseController < BaseController
        PRELOAD = [ :lab_exam, { lab_supply_lot: :lab_supply } ].freeze

        def index
          instrument = uuid?(params[:instrument_id]) && LabInstrument.find_by(id: params[:instrument_id])
          return not_found unless instrument
          return forbid unless reader?(instrument.exam_provider_id)

          rows = LabSupplyInUse.current.where(lab_instrument_id: instrument.id).includes(*PRELOAD).order(:started_at)
          render json: { in_use: rows.map { |u| Json.in_use(u) } }
        end

        def create
          instrument = uuid?(params[:instrument_id]) && LabInstrument.find_by(id: params[:instrument_id])
          return render_failure(Result.fail(:invalid_instrument)) unless instrument
          return forbid unless reader?(instrument.exam_provider_id)

          result = SetInUse.call(lot_id: params[:lot_id], instrument_id: instrument.id, lab_exam_id: params[:lab_exam_id], by: Current.user)
          return render_failure(result) if result.failure?

          render json: Json.in_use(result.payload[:in_use]), status: :created
        end
      end
    end
  end
  ```

  `app/controllers/lab/inventory/traces_controller.rb`:

  ```ruby
  # Rastreio por lote (contrato §5): membros do laboratório e admin; sem nome
  # de paciente; cada leitura grava lab_supply_lot.traced (só ids).
  require "csv"

  module Lab
    module Inventory
      class TracesController < BaseController
        before_action :load_lot

        def show
          render json: Trace.call(lot: @lot)
        end

        def csv
          data = Trace.call(lot: @lot)
          body = CSV.generate do |out|
            out << Trace::CSV_HEADERS
            data[:results].each { |r| out << [ "resultado", r[:requisition_number], r[:sigtap_code], r[:entered_at] ] }
            data[:qc_values].each { |v| out << [ "controle", v[:run_id], v[:analyte_key], v[:entered_at] ] }
          end
          send_data body, type: "text/csv; charset=utf-8", disposition: "attachment", filename: "rastreio-lote-#{@lot.lot_number.parameterize}.csv"
        end

        private

        def load_lot
          @lot = uuid?(params[:id]) && LabSupplyLot.includes(:lab_supply).find_by(id: params[:id])
          return not_found unless @lot
          return forbid unless reader?(@lot.lab_supply.exam_provider_id)

          DomainEvents.publish("lab_supply_lot.traced", lab_supply_lot_id: @lot.id, user_id: Current.user.id)
        end
      end
    end
  end
  ```

  `config/routes.rb` — dentro de `scope "inventory"`:

  ```ruby
        get   "in_use",               to: "in_use#index"
        post  "in_use",               to: "in_use#create"
        get   "lots/:id/trace.csv",   to: "traces#csv", format: false
        get   "lots/:id/trace",       to: "traces#show"
  ```

- [ ] **Step 5: Rode:** `rspec spec/requests/lab/inventory_trace_spec.rb spec/commands/lab/enter_results_spec.rb spec/requests/lab/entry_and_worklist_spec.rb spec/requests/lab/release_spec.rb` → verde.

- [ ] **Step 6: Commit** e mova o card F-21.33 para **Done**:

  ```bash
  git add app/commands/lab/inventory/set_in_use.rb app/services/lab/inventory/trace.rb app/services/lab/inventory/json.rb app/controllers/lab/inventory/in_use_controller.rb app/controllers/lab/inventory/traces_controller.rb app/commands/lab/enter_results.rb app/commands/lab/choose_result.rb app/services/lab/entry_json.rb config/routes.rb spec/requests/lab/inventory_trace_spec.rb
  git commit -m "feat: trace the reagent lot in use on every lab result and control value

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 15: Relatório de consumo estimado (F-21.34)

**Files:**
- Create: `app/services/lab/inventory/consumption.rb`, `app/controllers/lab/inventory/consumption_controller.rb`, `spec/requests/lab/inventory_consumption_spec.rb`
- Modify: `config/routes.rb`

**Interfaces:**
- Consumes: movimentos `opening` (Task 13); `lab_results` e `lab_qc_values` (Tasks 7, 14).
- Produces: `Lab::Inventory::Consumption.call(provider_id:, month: Date) → [{ supply_id, name, instrument_id, tests_performed, tests_opened, deviation_pct }]` (Desvio 13: por item reagente, uma linha total com `instrument_id: nil`, `tests_opened` e `deviation_pct`, e uma por aparelho com só `tests_performed`); `HEADERS`. Rotas `GET /lab/inventory/consumption?provider_id=&month=YYYY-MM` → `{ "rows" }` e `.csv`.

- [ ] **Step 1: Teste que falha** — `spec/requests/lab/inventory_consumption_spec.rb`:

  ```ruby
  require "rails_helper"

  # Spec §6.8 e Desvio 13 (F-21.34): testes realizados no mês (entradas de
  # resultado — a repetição conta, a cópia escolhida não — e controles por
  # nível) × testes das aberturas (embalagens abertas × rendimento); desvio
  # em %; nada é baixado; tela e CSV.
  RSpec.describe "Estoque — consumo estimado", type: :request do
    before { qc_city!(inventory: true) }

    let(:s) { qc_scenario! }
    let(:staff) { lab_staff! }

    it "conta testes e aberturas do mês e calcula o desvio" do
      supply = LabSupply.create!(exam_provider: s.provider, name: "Glicose GOD-PAP", kind: "reagent", unit: "kit", tests_per_package: 100,
                                 minimum_stock: 1, expiry_alert_days: 30, lab_exam_ids: [ s.exam.id ])
      lot = Lab::Inventory::ReceiveLot.call({ "supply_id" => supply.id, "lot_number" => "GL1", "expires_on" => (Time.zone.today + 200).iso8601,
                                              "supplier" => "Distribuidora", "origin" => "purchase", "quantity" => 5 }, by: staff).payload[:lot]
      Lab::Inventory::RecordMovement.call(lot: lot, kind: "opening", quantity: 1, reason: nil, by: staff, admin: false)
      req = received_requisition!(provider: s.provider, exams: [ s.exam ])
      item = req.items.first
      type!(item, { glicose: "98" })
      type!(item, { glicose: "99" }) # repetição: conta
      other = received_requisition!(provider: s.provider, exams: [ s.exam ]).items.first
      type!(other, { glicose: "101" })
      Lab::ChooseResult.call(result: item.results.reload.first, by: lab_releaser!(s.provider, lead: true)) # cópia: não conta
      qc_value!(s, "Nível 1", 0.1, by: staff)
      qc_value!(s, "Nível 2", -0.1, by: staff)
      sign_in_as(staff)
      month = Time.zone.today.strftime("%Y-%m")
      get "/lab/inventory/consumption", params: { provider_id: s.provider.id, month: month }
      expect(json_body["rows"]).to eq([
        { "supply_id" => supply.id, "name" => "Glicose GOD-PAP", "instrument_id" => nil, "tests_performed" => 5, "tests_opened" => 100,
          "deviation_pct" => -95.0 },
        { "supply_id" => supply.id, "name" => "Glicose GOD-PAP", "instrument_id" => s.instrument.id, "tests_performed" => 5,
          "tests_opened" => nil, "deviation_pct" => nil }
      ])
      expect(lot.reload.balance).to eq(4) # nada baixado pelo consumo
      get "/lab/inventory/consumption.csv", params: { provider_id: s.provider.id, month: month }
      expect(response.body.lines.first.strip).to eq("item,aparelho,testes_realizados,testes_das_aberturas,desvio_pct")
      get "/lab/inventory/consumption", params: { provider_id: s.provider.id, month: "2026-13" }
      expect(json_body["error"]).to eq("invalid_month")
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar:** `rspec spec/requests/lab/inventory_consumption_spec.rb` → 404.

- [ ] **Step 3: Implemente.** `app/services/lab/inventory/consumption.rb`:

  ```ruby
  # Consumo estimado (spec §6.8; Desvio 13): por item reagente do laboratório
  # no mês, testes realizados (entradas de resultado dos exames ligados — uma
  # por item e entered_at, sem cópias escolhidas — e valores de controle — um
  # por corrida, nível e entered_at — de analitos ligados a esses exames)
  # contra testes das aberturas (embalagens abertas × rendimento). Só leitura.
  module Lab
    module Inventory
      module Consumption
        HEADERS = %w[item aparelho testes_realizados testes_das_aberturas desvio_pct].freeze

        module_function

        def call(provider_id:, month:)
          range = month.beginning_of_month.beginning_of_day...month.next_month.beginning_of_month.beginning_of_day
          LabSupply.where(exam_provider_id: provider_id, kind: "reagent").order(:name).flat_map do |supply|
            performed = performed(provider_id, supply.lab_exam_ids, range)
            opened = -LabSupplyMovement.joins(:lab_supply_lot).where(lab_supply_lots: { lab_supply_id: supply.id }, kind: "opening", recorded_at: range)
                                       .sum(:quantity) * supply.tests_per_package
            total = performed.values.sum
            deviation = opened.zero? ? nil : ((total - opened) * 100.0 / opened).round(1)
            [ row(supply, nil, total, opened, deviation) ] +
              performed.reject { |id, _| id.nil? }.sort_by { |id, _| id }.map { |id, count| row(supply, id, count, nil, nil) }
          end
        end

        def performed(provider_id, exam_ids, range)
          counts = Hash.new(0)
          LabResult.joins(lab_requisition_item: :lab_exam_version)
                   .where(chosen: false, entered_at: range, lab_exam_versions: { lab_exam_id: exam_ids })
                   .distinct.pluck("lab_exam_versions.lab_instrument_id", "lab_results.lab_requisition_item_id", "lab_results.entered_at")
                   .each { |instrument_id, _, _| counts[instrument_id] += 1 }
          LabQcValue.joins(:lab_qc_run, lab_qc_lot: :lab_qc_material)
                    .where(entered_at: range, lab_qc_materials: { exam_provider_id: provider_id })
                    .pluck("lab_qc_runs.lab_instrument_id", :lab_qc_run_id, :level, :entered_at, :analyte_key, "lab_qc_materials.analytes")
                    .select { |*, key, analytes| analytes.any? { |a| a["analyte_key"] == key && exam_ids.include?(a["lab_exam_id"]) } }
                    .map { |instrument_id, run_id, level, at, *| [ instrument_id, run_id, level, at ] }.uniq
                    .each { |instrument_id, *| counts[instrument_id] += 1 }
          counts
        end

        def row(supply, instrument_id, performed, opened, deviation)
          { supply_id: supply.id, name: supply.name, instrument_id: instrument_id, tests_performed: performed, tests_opened: opened,
            deviation_pct: deviation }
        end
        private_class_method :performed, :row
      end
    end
  end
  ```

  `app/controllers/lab/inventory/consumption_controller.rb`:

  ```ruby
  # /lab/inventory/consumption (contrato §5): membros do laboratório e admin; sem dado de paciente; JSON e CSV.
  require "csv"

  module Lab
    module Inventory
      class ConsumptionController < BaseController
        before_action -> { provider! }
        before_action :load_month

        def show = render(json: { rows: Consumption.call(provider_id: @provider.id, month: @month) })

        def csv
          names = LabInstrument.where(exam_provider_id: @provider.id).pluck(:id, :name).to_h
          body = CSV.generate do |out|
            out << Consumption::HEADERS
            Consumption.call(provider_id: @provider.id, month: @month).each do |r|
              out << [ Exams::Report.safe_cell(r[:name]), r[:instrument_id] ? Exams::Report.safe_cell(names[r[:instrument_id]]) : "total",
                       r[:tests_performed], r[:tests_opened], r[:deviation_pct]&.to_s&.tr(".", ",") ]
            end
          end
          send_data body, type: "text/csv; charset=utf-8", disposition: "attachment", filename: "consumo-#{params[:month]}.csv"
        end

        private

        def load_month
          @month = params[:month].to_s.match?(/\A\d{4}-(0[1-9]|1[0-2])\z/) ? Date.parse("#{params[:month]}-01") : nil
          render_failure(Result.fail(:invalid_month)) unless @month
        end
      end
    end
  end
  ```

  `config/routes.rb` — dentro de `scope "inventory"`:

  ```ruby
        get   "consumption.csv",      to: "consumption#csv", format: false
        get   "consumption",          to: "consumption#show"
  ```

- [ ] **Step 4: Rode:** `rspec spec/requests/lab/inventory_consumption_spec.rb` → verde.

- [ ] **Step 5: Commit** e mova o card F-21.34 para **Done**:

  ```bash
  git add app/services/lab/inventory/consumption.rb app/controllers/lab/inventory/consumption_controller.rb config/routes.rb spec/requests/lab/inventory_consumption_spec.rb
  git commit -m "feat: report estimated reagent consumption per month without automatic deduction

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

---

## Fatia 7 — Invariantes, semente, prova e rollout

### Task 16: Invariantes do ADR 0038 e varreduras de log e evento

**Files:**
- Create: `spec/invariants/lab_quality_invariants_spec.rb`

**Interfaces:**
- Consumes: tudo das Tasks 3–15.
- Produces: nada novo (prova de ponta a ponta).

- [ ] **Step 1: Escreva o teste** `spec/invariants/lab_quality_invariants_spec.rb`:

  ```ruby
  require "rails_helper"

  # ADR 0038, Invariantes, provados de ponta a ponta: cada um tenta a escrita
  # proibida e espera a recusa do banco ou da aplicação; valor de controle,
  # justificativa e código de barras fora de log e de evento.
  RSpec.describe "Invariantes da qualidade e dos insumos (ADR 0038)", type: :request do
    include ActiveSupport::Testing::TimeHelpers

    before { qc_city!(inventory: true, hl7: true) }

    let(:s) { qc_scenario! }
    let(:staff) { lab_staff! }
    let(:lead) { lab_releaser!(s.provider, lead: true) }

    it "valor de controle nunca vira resultado de paciente (digitado ou pela HL7)" do
      qc_value!(s, "Nível 1", 0.3, by: staff)
      body = "MSH|^~\\&|MW|LAB|ROTA-SAUDE|t|20261018073000||OUL^R22^OUL_R22|INV1|P|2.5.1\rSPM|1|#{s.lot.barcodes.first.barcode}|||||||||Q\r" \
             "OBR|1|||GLI^G^99RSL\rOBX|1|NM|glicose^G^99RSL||101.5|mg/dL|||||F|||20261018072500||||BIO-01\r"
      log = capture_log { hl7_post(body, token: middleware_token!(s.provider)) }
      expect(LabQcValue.count).to eq(2)
      expect([ LabResult.count, ExamResult.count ]).to eq([ 0, 0 ])
      expect(log).not_to include("101.5", s.lot.barcodes.first.barcode)
    end

    it "com bloqueio ativo, só exceção do RT libera; a exceção fica registrada" do
      qc_value!(s, "Nível 1", 3.6, by: staff)
      req = received_requisition!(provider: s.provider, exams: [ s.exam ])
      item = type!(req.items.first, { glicose: "98" })
      refused = Lab::Release.call(requisition: req, item_ids: [ item.id ], signature: "advanced", totp: totp_for(lead), by: lead)
      expect(refused.reason).to eq(:qc_blocked)
      Lab::Qc::GrantException.call(block_id: LabQcBlock.active.sole.id, item_id: item.id, justification: "Paciente em sala de parto", by: lead)
      travel 31.seconds do
        expect(Lab::Release.call(requisition: req, item_ids: [ item.id ], signature: "advanced", totp: totp_for(lead), by: lead)).to be_ok
      end
      expect(LabQcException.sole.justification).to eq("Paciente em sala de parto")
      expect(DomainEvent.where("name LIKE 'lab_%'").pluck(:payload).to_json).not_to include("sala de parto")
    end

    it "corridas, valores, referências, exceções, revisões e movimentos nunca mudam nem somem" do
      qc_value!(s, "Nível 1", 3.6, by: staff)
      [ LabQcRun.first, LabQcValue.first, LabQcTarget.first ].each do |row|
        expect { row.destroy! }.to raise_error(ActiveRecord::StatementInvalid, /append-only/)
      end
      expect { LabQcValue.first.update!(outcome: "ok") }.to raise_error(ActiveRecord::StatementInvalid, /append-only/)
      expect { LabQcRun.first.update!(status: "approved") }.to raise_error(ActiveRecord::StatementInvalid, /refused/)
      expect { ActiveRecord::Base.connection.execute("TRUNCATE lab_qc_values") }.to raise_error(ActiveRecord::StatementInvalid)
    end

    it "saldo de lote nunca negativo e resultado guarda o lote em uso (ou nada)" do
      supply = LabSupply.create!(exam_provider: s.provider, name: "Glicose", kind: "reagent", unit: "kit", tests_per_package: 100, minimum_stock: 1,
                                 expiry_alert_days: 30, lab_exam_ids: [ s.exam.id ])
      lot = Lab::Inventory::ReceiveLot.call({ "supply_id" => supply.id, "lot_number" => "L1", "expires_on" => (Time.zone.today + 90).iso8601,
                                              "supplier" => "D", "origin" => "donation", "quantity" => 1 }, by: staff).payload[:lot]
      expect(Lab::Inventory::RecordMovement.call(lot: lot, kind: "loss", quantity: 2, reason: "Quebra", by: staff, admin: false).reason)
        .to eq(:insufficient_balance)
      expect(lot.balance).to eq(1)
      req = received_requisition!(provider: s.provider, exams: [ s.exam ])
      expect(type!(req.items.first, { glicose: "98" }).results.last.reagent_lot_id).to be_nil
      Lab::Inventory::SetInUse.call(lot_id: lot.id, instrument_id: s.instrument.id, lab_exam_id: s.exam.id, by: staff)
      expect(type!(req.items.first, { glicose: "97" }).results.last.reagent_lot_id).to eq(lot.id)
    end
  end
  ```

- [ ] **Step 2: Rode:** `rspec spec/invariants/lab_quality_invariants_spec.rb` → verde (todas as peças já existem; vermelho aqui é defeito de uma task anterior — corrija lá, com teste).

- [ ] **Step 3: Varreduras** (multilinha: confira cada chamada inteira):

  ```bash
  git grep -n "DomainEvents.publish(" -- app lib | grep -i "lab_qc\|lab_eqa\|lab_supply"
  git grep -n "Platform.audit(" -- app/commands/lab app/services/lab app/controllers/lab
  git grep -n "Current.city =" -- app lib
  git grep -n "Rails.logger" -- app/services/lab app/commands/lab app/controllers/lab app/controllers/hl7 lib/fake_middleware
  ```
  Expected: eventos só com ids e os campos do contrato §9 (`status`, `decision`, `source`, `kind`) mais `user_id` do rastreio; nenhum `Platform.audit` novo; nenhuma atribuição de `Current.city`; nenhum log com valor, código de barras, justificativa ou corpo HL7.

- [ ] **Step 4: Commit:**

  ```bash
  git add spec/invariants/lab_quality_invariants_spec.rb
  git commit -m "test: prove ADR 0038 invariants end to end

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 17: Semente de dev realista no laboratório municipal de Curitiba

**Files:**
- Create: `lib/lab_qc_crew.rb`, `spec/lib/lab_qc_crew_spec.rb`
- Modify: `db/seeds.rb`

**Interfaces:**
- Consumes: `LabCrew` (21b: laboratório, exames v2 com códigos de aparelho, contas `tecnica-lab@`, `biomedica@` e `patologista@`); `FakeMiddleware::QcPlan` (Task 11); comandos das Tasks 4, 5, 7, 13, 14.
- Produces: `LabQcCrew.seed_current_city(slug:, admin:) → { switch:, created: [String] }` (idempotente; só Curitiba). Nenhuma conta nova: as do 21b operam o 21c.

- [ ] **Step 1: Teste que falha** — `spec/lib/lab_qc_crew_spec.rb`:

  ```ruby
  require "rails_helper"
  require Rails.root.join("lib/lab_crew").to_s
  require Rails.root.join("lib/lab_qc_crew").to_s

  # Semente de dev do 21c (dev fictício que imita o real): dois aparelhos
  # (bioquímica e hematologia) ligados às versões dos exames; material de dois
  # níveis com lote e referência da bula (os mesmos códigos e alvos do
  # fake-middleware); 20 corridas aprovadas de história (sugestão de
  # referência); itens de estoque com lotes, um vencendo e um abaixo do
  # mínimo; lote em uso. Idempotente.
  RSpec.describe LabQcCrew do
    before do
      lab_city!(fake_middleware: true)
      Lab::Templates::Load.sigtap_codes.each { |c| SigtapProcedure.find_or_create_by!(release: sigtap_release!, code: c) { |p| p.name = "EXAME #{c}" } }
      Maintainer.find_by(email_address: "dev@local") ||
        Maintainer.create!(email_address: "dev@local", password: "dev-password-123", otp_secret: ROTP::Base32.random, otp_enabled_at: Time.current)
      ExamProvider.create!(name: LabCrew::LAB, kind: "municipal", cnes: "2385123")
      %w[UBS\ Jardim\ das\ Flores UBS\ Vila\ Esperança].each { |name| HealthUnit.find_or_create_by!(name: name) { |u| u.kind = "ubs" } }
      LabCrew.seed_platform!
      LabCrew.seed_current_city(slug: "curitiba", password: "dev-password", admin: lab_admin!)
    end

    let(:lab) { ExamProvider.find_by!(name: LabCrew::LAB) }

    it "aparelhos, CIQ com história, estoque com alertas e lote em uso; segunda rodada não muda nada" do
      result = described_class.seed_current_city(slug: "curitiba", admin: lab_admin!)
      expect(result[:switch]).to start_with("ligado")
      expect(LabInstrument.where(exam_provider: lab).pluck(:hl7_equipment_id)).to contain_exactly("BIO-01", "HEM-01")
      expect(LabExam.find_by!(exam_provider: lab, sigtap_code: "0202010473").current_version.lab_instrument.hl7_equipment_id).to eq("BIO-01")
      expect(LabExam.find_by!(exam_provider: lab, sigtap_code: "0202020380").current_version.lab_instrument.hl7_equipment_id).to eq("HEM-01")
      lot = LabQcLot.find_by!(lot_number: FakeMiddleware::QcPlan::LOT)
      expect(lot.level_barcodes).to eq(FakeMiddleware::QcPlan::BARCODES)
      expect(LabQcTarget.where(lab_qc_lot: lot).pluck(:source)).to eq(%w[package_insert] * 4)
      expect(LabQcRun.where(status: "approved").count).to eq(20)
      expect([ LabQcBlock.count, LabQcValue.where.not(outcome: "ok").count ]).to eq([ 0, 0 ])
      expect(Lab::Qc::Suggestions.list(provider_id: lab.id).map(&:runs)).to eq([ 20 ] * 4)
      alerts = Lab::Inventory::Alerts.call(provider_id: lab.id)
      expect(alerts[:expiring].map { |l| l[:lot_number] }).to eq([ "GL2603" ])
      expect(alerts[:below_minimum].map { |x| x[:name] }).to eq([ "Creatinina Jaffé — kit 300 testes" ])
      expect(LabSupplyInUse.current.joins(:lab_supply_lot).pluck("lab_supply_lots.lot_number")).to contain_exactly("GL2609", "CR2611")
      snapshot = -> { [ LabInstrument, LabExamVersion, LabQcMaterial, LabQcLot, LabQcTarget, LabQcValue, LabSupply, LabSupplyLot, LabSupplyMovement, LabSupplyInUse ].map(&:count) }
      expect { described_class.seed_current_city(slug: "curitiba", admin: lab_admin!) }.not_to(change { snapshot.call })
    end

    it "fora de Curitiba não liga nada" do
      expect(described_class.seed_current_city(slug: "maringa", admin: lab_admin!)[:switch]).to start_with("desligado")
    end
  end
  ```

- [ ] **Step 2: Rode e veja falhar:** `rspec spec/lib/lab_qc_crew_spec.rb` → `LoadError` (`lib/lab_qc_crew`).

- [ ] **Step 3: Implemente** `lib/lab_qc_crew.rb`:

  ```ruby
  require "zlib"
  require_relative "fake_middleware/qc_plan"

  # Semente de dev do módulo 21c (ADR 0038). Dev é fictício mas imita o real:
  # no "Laboratório Municipal Central" de Curitiba (do LabCrew), um Mindray
  # BS-240 (bioquímica, BIO-01) e um Sysmex XN-350 (hematologia, HEM-01),
  # classe de aparelho dos editais municipais; as versões dos exames de
  # bioquímica e hematologia ganham o aparelho; material Lyphochek (Bio-Rad)
  # de dois níveis com o lote, os códigos e a bula de FakeMiddleware::QcPlan
  # (o simulado manda corridas contra eles); referência da bula aceita pela
  # RT há 25 dias; 20 corridas aprovadas de história (uma por dia, z
  # alternando de lado), que liberam a sugestão da referência do laboratório;
  # estoque com reagentes de glicose (um lote vencendo) e creatinina (abaixo
  # do mínimo) e tubos; lote em uso no BIO-01. Opera com as contas do 21b.
  # Só Curitiba. Idempotente. Roda depois do LabCrew.
  class LabQcCrew
    SWITCHES = %w[lab_quality_control lab_inventory].freeze
    PLAN = FakeMiddleware::QcPlan
    INSTRUMENTS = [
      { "name" => "Analisador bioquímico 1", "manufacturer" => "Mindray", "model" => "BS-240", "serial_number" => "BS240-2023-0417",
        "sector" => "biochemistry", "hl7_equipment_id" => "BIO-01" },
      { "name" => "Analisador hematológico 1", "manufacturer" => "Sysmex", "model" => "XN-350", "serial_number" => "XN350-2022-1189",
        "sector" => "hematology", "hl7_equipment_id" => "HEM-01" }
    ].freeze
    SECTOR_INSTRUMENT = { "biochemistry" => "BIO-01", "hematology" => "HEM-01" }.freeze
    MATERIAL = { "name" => "Lyphochek Assayed Chemistry Control", "manufacturer" => "Bio-Rad" }.freeze
    # z de cada corrida da história contra a bula (Nível 1; o Nível 2 leva o sinal trocado): alternando de lado, nenhuma regra dispara.
    HISTORY_Z = [ 0.3, -0.5, 0.8, -0.2, 0.6, -0.9, 0.4, -0.3, 1.1, -0.6, 0.2, -0.4, 0.7, -0.8, 0.5, -0.1, 0.9, -0.7, 0.3, -0.2 ].freeze
    TARGETS_DAYS_AGO = 25
    SUPPLIER = "Distribuidora Paranaense de Diagnósticos Ltda.".freeze
    # lots: [número, dias até vencer, embalagens]
    SUPPLIES = [
      { "name" => "Glicose GOD-PAP — kit 500 testes", "kind" => "reagent", "unit" => "kit", "tests_per_package" => 500, "minimum_stock" => 2,
        "sigtap" => %w[0202010473], "lots" => [ [ "GL2609", 240, 4 ], [ "GL2603", 20, 1 ] ], "in_use" => "GL2609", "opened" => { "GL2609" => 1 } },
      { "name" => "Creatinina Jaffé — kit 300 testes", "kind" => "reagent", "unit" => "kit", "tests_per_package" => 300, "minimum_stock" => 3,
        "sigtap" => %w[0202010317], "lots" => [ [ "CR2611", 300, 2 ] ], "in_use" => "CR2611", "opened" => {} },
      { "name" => "Tubo de coleta 4 mL com fluoreto", "kind" => "consumable", "unit" => "caixa com 100", "tests_per_package" => nil, "minimum_stock" => 10,
        "sigtap" => [], "lots" => [ [ "TF2512", 400, 25 ] ], "in_use" => nil, "opened" => {} }
    ].freeze

    class << self
      def seed_current_city(slug:, admin:)
        raise "semente da qualidade: ambiente implantado; a semente é só de dev" if Rota.deployed?
        return { switch: "desligado (só Curitiba liga)", created: [] } unless slug == "curitiba"

        maintainer = Maintainer.find_by(email_address: LabCrew::DEV_MAINTAINER)
        return { switch: "desligado (sem mantenedor #{LabCrew::DEV_MAINTAINER})", created: [] } unless maintainer

        city = City.find(Current.city.id)
        SWITCHES.each { |key| Platform::Features.set!(city: city, key: key, enabled: true, maintainer: maintainer) }
        lab = ExamProvider.find_by!(name: LabCrew::LAB)
        staff = User.find_by!(email_address: "tecnica-lab@#{slug}.demo")
        lead = User.find_by!(email_address: "biomedica@#{slug}.demo")
        created = []
        instruments = instruments!(lab, created)
        link_versions!(lab, instruments, admin, created)
        quality_control!(lab, instruments.fetch("BIO-01"), lead, staff, created)
        inventory!(lab, instruments.fetch("BIO-01"), staff, created)
        { switch: "ligado (#{SWITCHES.join(', ')})", created: created }
      end

      private

      def check!(result, what)
        raise "semente da qualidade: #{what} recusado (#{result.reason})" if result.failure?

        result.payload
      end

      def instruments!(lab, created)
        INSTRUMENTS.to_h do |spec|
          found = LabInstrument.find_by(exam_provider_id: lab.id, hl7_equipment_id: spec["hl7_equipment_id"])
          unless found
            found = check!(Lab::SaveInstrument.call(spec.merge("provider_id" => lab.id, "active" => true)), "aparelho #{spec['hl7_equipment_id']}")[:instrument]
            created << "aparelho #{spec['hl7_equipment_id']}"
          end
          [ spec["hl7_equipment_id"], found ]
        end
      end

      # Nova versão (nunca edita) com o aparelho do setor, para os exames que ainda não têm.
      def link_versions!(lab, instruments, admin, created)
        LabExam.where(exam_provider_id: lab.id, active: true, sector: SECTOR_INSTRUMENT.keys).includes(:current_version).order(:name).each do |exam|
          version = exam.current_version
          next if version.nil? || version.lab_instrument_id

          params = { "method" => version.analysis_method, "material" => version.material, "container" => version.container,
                     "instrument_test_code" => version.instrument_test_code, "analytes" => version.analytes,
                     "lab_instrument_id" => instruments.fetch(SECTOR_INSTRUMENT.fetch(exam.sector)).id }
          check!(Lab::CreateVersion.call(exam: exam, params: params, by: admin), "versão de #{exam.sigtap_code}")
          created << "versão #{exam.sigtap_code}"
        end
      end

      def quality_control!(lab, instrument, lead, staff, created)
        since = Time.current - TARGETS_DAYS_AGO.days
        exams = PLAN::ANALYTES.keys.to_h do |code|
          [ code, LabExam.where(exam_provider_id: lab.id).joins(:current_version).find_by!(lab_exam_versions: { instrument_test_code: code }) ]
        end
        material = LabQcMaterial.find_by(exam_provider_id: lab.id, name: MATERIAL["name"]) || begin
          analytes = PLAN::ANALYTES.map { |code, (key, _unit)| { "lab_exam_id" => exams.fetch(code).id, "analyte_key" => key } }
          created << "material de controle"
          check!(Lab::Qc::SaveMaterial.call(MATERIAL.merge("levels" => PLAN::BARCODES.keys, "analytes" => analytes, "active" => true), provider: lab),
                 "material")[:material]
        end
        lot = material.lots.find_by(lot_number: PLAN::LOT) || begin
          created << "lote #{PLAN::LOT}"
          check!(Lab::Qc::CreateLot.call({ "material_id" => material.id, "lot_number" => PLAN::LOT, "expires_on" => (Time.zone.today + 300).iso8601,
                                           "level_barcodes" => PLAN::BARCODES }, by: lead, now: since), "lote")[:lot]
        end
        PLAN::TARGETS.each do |level, by_code|
          by_code.each do |code, (mean, sd, _decimals)|
            key = PLAN::ANALYTES.fetch(code).first
            next if LabQcTarget.current(lot_id: lot.id, level: level, analyte_key: key, instrument_id: instrument.id)

            check!(Lab::Qc::CreateTarget.call({ "lot_id" => lot.id, "level" => level, "analyte_key" => key, "instrument_id" => instrument.id,
                                                "mean" => mean.tr(".", ","), "sd" => sd.tr(".", ",") }, by: lead, now: since), "referência #{key} #{level}")
          end
        end
        history!(lot, instrument, staff, created) unless LabQcValue.exists?(lab_qc_lot_id: lot.id)
      end

      # Uma corrida por dia, de 21 a 2 dias atrás, às 7h30 (abertura da rotina), digitada pela técnica.
      def history!(lot, instrument, staff, created)
        HISTORY_Z.each_with_index do |z, i|
          day = (Time.zone.today - (HISTORY_Z.size + 1 - i)).in_time_zone.change(hour: 7, min: 30)
          PLAN::BARCODES.keys.each_with_index do |level, li|
            PLAN::ANALYTES.each do |code, (key, _unit)|
              value = BigDecimal(PLAN.value(level, code, li.zero? ? z : -z))
              payload = check!(Lab::Qc::RecordValue.call(instrument: instrument, lot: lot, level: level, analyte_key: key, value: value,
                                                         source: "typed", by: staff, now: day + li.minutes), "controle #{key} #{level}")
              raise "semente da qualidade: a história reprovou (#{key} #{level})" unless payload[:value].outcome == "ok"
            end
          end
        end
        created << "#{HISTORY_Z.size} corridas aprovadas"
      end

      def inventory!(lab, instrument, staff, created)
        SUPPLIES.each do |spec|
          exam_ids = LabExam.where(exam_provider_id: lab.id, sigtap_code: spec["sigtap"]).pluck(:id)
          supply = LabSupply.where(exam_provider_id: lab.id).find_by("lower(name) = ?", spec["name"].downcase) || begin
            created << "item #{spec['name']}"
            attrs = spec.slice("name", "kind", "unit", "tests_per_package", "minimum_stock").merge("expiry_alert_days" => 30, "lab_exam_ids" => exam_ids,
                                                                                                    "active" => true)
            check!(Lab::Inventory::SaveSupply.call(attrs, provider: lab), "item #{spec['name']}")[:supply]
          end
          spec["lots"].each do |number, days, quantity|
            next if LabSupplyLot.exists?(lab_supply_id: supply.id, lot_number: number)

            lot = check!(Lab::Inventory::ReceiveLot.call({ "supply_id" => supply.id, "lot_number" => number, "expires_on" => (Time.zone.today + days).iso8601,
                                                           "supplier" => SUPPLIER, "origin" => "purchase", "quantity" => quantity,
                                                           "document" => "NF-e #{(Zlib.crc32(number) % 900_000) + 100_000}" }, by: staff), "lote #{number}")[:lot]
            opened = spec["opened"].fetch(number, 0)
            check!(Lab::Inventory::RecordMovement.call(lot: lot, kind: "opening", quantity: opened, reason: nil, by: staff, admin: false), "abertura") if opened.positive?
          end
          next unless spec["in_use"]

          lot = LabSupplyLot.find_by!(lab_supply_id: supply.id, lot_number: spec["in_use"])
          exam_ids.each { |exam_id| check!(Lab::Inventory::SetInUse.call(lot_id: lot.id, instrument_id: instrument.id, lab_exam_id: exam_id, by: staff), "lote em uso") }
        end
      end
    end
  end
  ```

  Em `db/seeds.rb`: ao lado de `require Rails.root.join("lib/lab_crew").to_s`, acrescente `require Rails.root.join("lib/lab_qc_crew").to_s`; logo depois do bloco do laboratório municipal (as linhas do `LabCrew.seed_current_city`):

  ```ruby
          # ── Qualidade e insumos do laboratório (módulo 21c, ADR 0038) ─────────
          # Depois do LabCrew (laboratório, exames e as contas que operam o 21c).
          qc = LabQcCrew.seed_current_city(slug: slug, admin: muni_admin)
          puts "[seeds] qualidade ... #{qc[:switch]}#{"; #{qc[:created].join(', ')}" if qc[:created].any?}"
  ```

- [ ] **Step 4: Rode:** `rspec spec/lib/lab_qc_crew_spec.rb spec/lib/lab_crew_spec.rb` → verde.

- [ ] **Step 5: Commit:**

  ```bash
  git add lib/lab_qc_crew.rb db/seeds.rb spec/lib/lab_qc_crew_spec.rb
  git commit -m "feat: seed realistic lab quality control and supplies data for development

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

### Task 18: Revisão final, suíte completa, api na porta 3043, prova no navegador e runbook de rollout

**Files:**
- Create: `docs/operacao/rollout-qualidade-insumos-laboratorio.md` (repo `docs`)

- [ ] **Step 1: Varreduras** (as da Task 16, Step 3) e `docker compose exec -T -w /rails/.claude/mod21c api bin/rubocop`. Expected: limpo.

- [ ] **Step 2: Suíte completa** com o worker parado e sem outra sessão rodando suíte:

  ```bash
  docker compose stop worker
  docker compose exec -T -e ROTA_TEST_DB_SUFFIX=_mod21c -w /rails/.claude/mod21c api bundle exec rspec
  docker compose start worker
  ```
  Expected: verde; ~0,10 s por exemplo com o host calmo (memória "BCrypt e custo em teste": acima de 0,15 s investigue).

- [ ] **Step 3: api do worktree na porta 3043** (container `api-dev`, serviço `api`):

  ```bash
  docker compose exec -d -w /rails/.claude/mod21c api bin/rails server -b 0.0.0.0 -p 3043
  docker compose exec -T -w /rails/.claude/mod21c api bin/rails city:migrate:all
  docker compose exec -T -w /rails/.claude/mod21c api bin/rails db:seed
  docker compose exec -T api curl -s -o /dev/null -w '%{http_code}\n' -H "Host: curitiba.localhost" http://localhost:3043/up
  ```
  Expected: `200`; o `db:seed` imprime `[seeds] qualidade ... ligado (lab_quality_control, lab_inventory); aparelho BIO-01, ...`. O dashboard do worktree (`.worktrees/dashboard-mod21c`, Vite 5196) aponta o proxy para `http://api:3043`. Memória "Env do arquivo ≠ env do processo": confira com `printenv` no container do simulado.

- [ ] **Step 4: Prova no navegador** (spec §10), em `http://curitiba.localhost:5196`, com o simulado do worktree:

  ```bash
  export FAKE_MIDDLEWARE_WORKDIR=/rails/.claude/mod21c FAKE_MIDDLEWARE_API_URL=http://api:3043
  docker compose --profile fake-middleware up -d fake-middleware   # manda a corrida aprovada do dia ao subir
  ```
  1. biomedica@ (RT) abre o lote `CQB-2611` e confere a referência da bula (Nível 1 e 2 de glicose e creatinina no BIO-01);
  2. corrida aprovada: `docker compose logs fake-middleware | grep controle` mostra `MSA AA`; a corrida do dia aparece `approved`;
  3. corrida reprovada: `docker compose --profile fake-middleware run --rm fake-middleware bundle exec ruby lib/fake_middleware/run.rb qc reject_2_2s` → corrida `rejected`, 2-2s marcado no Levey-Jennings, bloqueio ativo da glicose no BIO-01, contador no menu;
  4. tecnica-lab@ digita a glicose de uma requisição recebida (exame com a versão v3, ligada ao BIO-01); biomedica@ tenta liberar → 409 `qc_blocked` com o item e o aparelho; concede a exceção com justificativa (TOTP do step-up) e libera; a revisão lista os laudos liberados desde a última corrida aceita; decide `kept` num e `rectify` noutro (abre a retificação do 21b);
  5. tecnica-lab@ registra a ação corretiva; `docker compose --profile fake-middleware run --rm fake-middleware bundle exec ruby lib/fake_middleware/run.rb qc approved` → o bloqueio fecha (data e corrida no bloqueio);
  6. sugestão de referência (semente com 20 corridas) para os 4 pares; biomedica@ aceita uma com motivo → versão 2 `laboratory`; o gráfico mostra a troca de versão;
  7. CEQ: biomedica@ registra uma rodada PNCQ com um item inadequado (ação obrigatória) e anexa o PDF do relatório anual; abre o PDF; resumo do ano;
  8. estoque: entrada de um lote novo de glicose, abertura de um kit, troca do lote em uso no BIO-01; alertas (GL2603 vencendo, creatinina abaixo do mínimo);
  9. nova glicose digitada → a linha mostra o lote; rastreio do lote (tela e CSV, sem nome de paciente);
  10. consumo do mês (tela e CSV).
  Anote cada passo (OK/falha). Falha de api → corrija com teste antes; achado só visual → ciclo de interface (memória "Interface em ciclo próprio").

- [ ] **Step 5: Runbook** `docs/operacao/rollout-qualidade-insumos-laboratorio.md` (repo docs):

  ```markdown
  # Rollout — qualidade e insumos do laboratório municipal (módulo 21c, ADR 0038)

  ## Pré-requisitos
  - 21b em produção na cidade (`municipal_lab` utilizável; liberadores com RT e substituto designados).
  - Tag do contracts publicada: `lab-v1.1.0` (seção "Controle" do perfil HL7; exemplos copiados em
    `apps/api/spec/fixtures/lab/`).
  - Nenhuma gem nova; nenhuma migração de plataforma.

  ## Ordem de deploy
  1. contracts (tag `lab-v1.1.0`) → 2. api: publicar a imagem e rodar `bin/migrate` dela ANTES de cortar tráfego
     (`city:migrate:all` — 20261018900001, irreversível; tabelas novas, `lab_exam_versions.lab_instrument_id`,
     `lab_results.reagent_lot_id` e triggers em `db/city_triggers.sql`).
  3. dashboard → 4. maintenance → 5. catálogo público. O wpda não muda.
  Nunca migrar cidade fora do rake.

  ## Ligar numa cidade (maintenance)
  1. `lab_quality_control` e/ou `lab_inventory` (independentes; ambos exigem `municipal_lab`; existem em todos os ambientes).
  2. Admin da cidade: cadastrar os aparelhos (com o identificador HL7 que o middleware manda em `OBX-18`) e criar
     nova versão de cada exame ligada ao aparelho (só itens de versão com aparelho entram no bloqueio e no rastreio).
  3. RT/substituto: material de controle, lote com os códigos de barras por nível (nunca só 10 dígitos), referência
     da bula por nível, analito e aparelho; regras de Westgard (todas ligadas por padrão).
  4. Estoque: itens (rendimento do reagente, mínimo, exames ligados), entrada dos lotes e lote em uso por aparelho e exame.
  5. Middleware real mandando controle: homologar com o fornecedor a convenção do perfil (`SPM-2` = código do nível;
     `SPM-11` = `Q`; aparelho em `OBX-18`) antes de depender dele; até lá, os valores entram digitados.

  ## Gates de go-live (spec §13)
  - Middleware real mandando controles (convenção de cada fornecedor): mensagem aprovada, reprovada e as recusas
    (`qc_lot_expired`, `qc_lot_closed`, `qc_no_target`, `unknown_instrument`) conferidas com o fornecedor.
  - Validação do CIQ pelo RT do laboratório (entra no dossiê do art. 106 da RDC 978).
  - Laboratórios com CIQ no software do provedor de controle: convivem digitando os valores até a importação CSV (api#87).
  - Volume: acompanhar o crescimento de `lab_qc_values` por banco de cidade no primeiro mês.
  - `fake_middleware` em staging manda corridas de controle, inclusive reprovadas: isso ABRE BLOQUEIO de liberação
    no laboratório da cidade de staging. Não ligue `fake_middleware` onde há amostras de teste em liberação sem saber disso.

  ## Verificação depois de ligar
  - `GET /lab/qc/blocks/active/count` com uma conta do laboratório (sem trilha); corrida de controle digitada; liberação
    de um item de teste com e sem bloqueio; `GET /lab/inventory/alerts`.
  - Nenhum valor de controle nem justificativa no log do dia (`grep` do valor de teste).

  ## Desligar
  - `lab_quality_control` desligado: rotas `/lab/qc` e `/lab/eqa` respondem 403; a liberação do 21b deixa de consultar
    bloqueios (os registros ficam); a HL7 recusa controles com `qc_disabled`.
  - `lab_inventory` desligado: rotas `/lab/inventory` respondem 403; o lote em uso já gravado continua sendo anotado
    nos resultados até ser encerrado (Desvio 9) — encerre os lotes em uso antes de desligar se quiser parar o rastreio.
  - Migração irreversível: não há rollback de schema; volte só o código (expand/contract).
  ```
  Commit no repo docs (sem push):

  ```bash
  /opt/homebrew/bin/git -C docs add operacao/rollout-qualidade-insumos-laboratorio.md
  /opt/homebrew/bin/git -C docs commit -m "docs: add lab quality and supplies rollout runbook (module 21c)

  Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
  ```

- [ ] **Step 6: Fechamento.** Pare o servidor da 3043 e o `fake-middleware` (`docker compose --profile fake-middleware stop fake-middleware`); apague os bancos de teste `_mod21c` (conferindo conexões e sessões ativas — memória "Apagar bancos com sufixo de módulo"); confira os cards F-21.25..F-21.34 em **Done**. Merge, push e a tag `lab-v1.1.0` **só com autorização do usuário**, uma etapa de cada vez (memória "Sessões paralelas no push": `origin/main..main` antes de push).

---

## Self-review (feito ao escrever o plano)

- **Cobertura da spec:** §3 interruptores → Task 2 (e C13 dos aparelhos na Task 4); §4 aparelhos e vínculo → Task 4; CIQ (materiais, lotes, referências, corridas, valores, regras, bloqueios, exceções, revisões) → Tasks 3, 5–9; CEQ → Task 12; estoque → Tasks 13–15; `lab_results.reagent_lot_id` → Tasks 3 e 14; §5.1 cadastro → Task 5; §5.2 corrida → Task 7; §5.3 Westgard na gravação, entre níveis → Tasks 6–7; §5.4 bloqueio na liberação → Task 8; §5.5 exceção → Task 8; §5.6 revisão → Tasks 7–8; §5.7 fechamento → Task 7; §5.8 sugestão após 20 → Task 9; §5.9 telas (formas) → Tasks 5–9; §5.10 CEQ → Task 12; §6.1–6.5 → Task 13; §6.6 alertas → Task 13; §6.7 rastreio → Task 14; §6.8 consumo → Task 15; §7 HL7 e simulado → Tasks 0, 1, 10, 11; §8 papéis → controllers de cada task (C14 muda a ação corretiva); §9 trilha e eventos → Task 3 e as varreduras da Task 16; §10 testes → cada task, inclusive a corrida bloqueio × liberação (Task 8) e a prova no navegador (Task 18); §11 entrega → Tasks 0, 1 e 18; semente → Task 17; F-21.25..F-21.34 → mapa do "Ambiente de execução".
- **Placeholders:** nenhum.
- **Tipos e nomes:** `Lab::Qc::RecordValue.call(... run:, run_id:, now:)`, `Lab::Qc::Blocks.{open!,try_close!}`, `Lab::Qc::ReleaseCheck.{lock_instruments!,blocked}`, `Lab::Qc::Westgard.evaluate`, `Lab::Qc::Decimal.{typed,hl7,out}`, `Lab::Qc::CreateTarget.next_version`, `Lab::Qc::Suggestions.{list,find,for_target}`, `LabSupplyInUse.current_lot_id`, `Lab::Inventory::{ReceiveLot,RecordMovement,SetInUse,Trace,Consumption,Alerts}`, `FakeMiddleware::QcPlan`, helpers `qc_city!`, `qc_scenario!`, `qc_value!`, `qc_glucose`, `comma` são os mesmos em todas as tasks.
- **Review Focus:** 1 → Task 9 ("referência trocada no meio da corrida"); 2 → Task 10 ("controle nunca vira paciente"); 3 → Task 8 ("exceção não atravessa bloqueio"); 4 → Task 13 (`inventory_concurrency_spec`); 5 → Task 7 ("vírgula na digitação") e Task 10 ("recusas", `90,5` pela HL7).

## Divergências propostas ao contrato

Numeradas como **Q** (do plano do api). As C do dashboard e as M do maintenance estão na tabela logo abaixo.

1. **Q1 — Aparelho só por `OBX-18`** (Task 0): `EQU` não faz parte do `OUL^R22` na 2.5.1; o §6 cita `EQU-1`. Vale a repetição de `OBX-18` cujo componente 1 casa (sem caixa) com `hl7_equipment_id` de um aparelho ativo; nenhuma ou mais de uma → `unknown_instrument` (`OBX^n^18`).
2. **Q2 — Convenção do controle** (Task 0): o `SPM-2` (código do nível de lote cadastrado) decide; `SPM-11` = `Q` recomendado e, com código que não é de controle, `AE` `unknown_barcode` (nunca cai no caminho do paciente); `INV` aceito e ignorado; `SPM-4` ignorado no controle; ponto decimal em `OBX-5`. Recusa nova `qc_disabled` (`AE`, 207, `SPM^n^2`) com `lab_quality_control` desligado; `unit_mismatch`, `exam_not_in_version`, `unknown_analyte`, `invalid_value` valem também no controle. A reprovação de Westgard não muda o `ACK` (`AA`).
3. **Q3 — Corrida aceita sem ação corretiva:** o valor entra (201) e a resposta traz `"notice": "corrective_action_required"` com o bloqueio ainda ativo (o §3 fala em 422, que daria a entender que o valor foi recusado). Fechamento exige os valores de todos os níveis gravados **depois** da ação.
4. **Q4 — "Aprovada" = `approved` ou `warning`** (1-2s é aviso) para fechar bloqueio, para a revisão e para as 20 corridas.
5. **Q5 — Corridas:** `run_id` de corrida reprovada → 409 `invalid_transition`; depois de uma reprovada, o próximo valor do dia abre corrida nova; `POST /lab/qc/runs` sempre cria; pela HL7, os valores de um aparelho numa mensagem ficam na mesma corrida.
6. **Q6 — Série de Westgard** sem valores de corridas reprovadas; até 9 anteriores; limites estritos (`|z| > 2`, `> 3`; R-4s = um acima de +2 e outro abaixo de −2 na mesma corrida); 2-2s e R-4s entre níveis usam o valor mais recente de cada outro nível na corrida.
7. **Q7 — Regras como versões só de acréscimo** (`lab_qc_rules`), sem linha = todas ligadas; `invalid_rules` quando faltam chaves ou não são booleanas.
8. **Q8 — Códigos de barras de controle** em `lab_qc_lot_barcodes` (único na cidade); formato `[A-Za-z0-9._-]{3,30}`; só 10 dígitos é o espaço das amostras → `barcode_taken`; formato inválido → `invalid_barcode` (novo).
9. **Q9 — Referência:** `lab_qc_values.lab_qc_target_id` no valor (a versão sai na API); coluna `reason` (C6); aceite recalcula no servidor (nada vem do cliente); sem sugestão → 404 `not_found`.
10. **Q10 — Revisão:** itens do aparelho cujo exame tem o analito, com laudo liberado depois da última corrida aceita com o analito (ou da primeira referência); exceção só para item `pending`/`typed`, uma por bloqueio e item (409 `conflict`); `invalid_decision`, `invalid_note`.
11. **Q11 — Eventos a mais (só ids):** `lab_qc_block.action_recorded`, `lab_supply_lot.traced` (`user_id`). Nenhum evento de plataforma.
12. **Q12 — Estoque:** entrada com o mesmo número e a mesma validade soma no lote existente; outra validade → `duplicate_lot`; `POST /lab/inventory/in_use` com o lote já corrente é idempotente; o lote em uso é lido sem interruptor (Desvio 9).
13. **Q13 — Consumo:** um teste = uma entrada de resultado por item (repetição conta, cópia escolhida não) ou um valor de controle por corrida e nível; linha total com `instrument_id: null`, linhas por aparelho com `tests_opened`/`deviation_pct` nulos; `month` = `YYYY-MM` (`invalid_month`).
14. **Q14 — Códigos de erro novos (422 salvo indicação):** `invalid_name`, `invalid_instrument`, `duplicate_name`, `invalid_active`, `invalid_lot_number`, `invalid_expires_on`, `invalid_barcode`, `duplicate_lot`, `invalid_analytes`, `invalid_rules`, `invalid_decision`, `invalid_note`, `invalid_date`, `invalid_period`, `filter_required`, `reason_required`, `invalid_program`, `invalid_round_code`, `invalid_round_date`, `invalid_items`, `duplicate_round`, `invalid_file`, `invalid_kind`, `invalid_unit`, `invalid_tests_per_package`, `invalid_minimum_stock`, `invalid_expiry_alert_days`, `invalid_exams`, `invalid_quantity`, `invalid_origin`, `invalid_supplier`, `invalid_document`, `invalid_month`; `not_blocked` e `qc_blocked` (409) como no contrato.
15. **Q15 — Ponto decimal (citação errada no contrato):** o preâmbulo atribui "ponto recusado" à D11 do 21b, mas a D11 é o contador sem trilha e a digitação do 21b aceita vírgula e ponto (`Lab::Values::NUMBER`). Este plano segue o 21c: nas rotas novas, ponto recusado (`invalid_value`); a digitação de resultado do 21b não muda.
16. **Q16 — Levey-Jennings** só para liberador ligado, RT/substituto e admin (spec §8); `lab_staff` → 403. Período até 366 dias (`invalid_period`).
17. **Q17 — CEQ:** um item por analito por rodada (`invalid_items`); data não futura; relatório anual lido sem trilha (não tem dado de paciente).

### Formas do dashboard (C1–C17) e do maintenance (M1–M3)

| Item | Decisão | Onde |
|---|---|---|
| C1 `technical_lead_of` no contexto | adotada | Task 4 |
| C2 `provider_id` e envelopes | adotada; `/lab/qc/runs` filtra por `instrument_id`; `status` opcional em lotes, bloqueios e revisões; `year` nas rodadas; `/lab/instruments` também para admin | Tasks 4–15 |
| C3 formas | adotadas, com campos a mais que não quebram: `expired` no lote, `reason` na referência, `requisition_id`/`exam_label`/`report_id` na revisão, `ended_at` no lote em uso, `in_use` no lote de estoque | Tasks 5, 8, 12–14 |
| C4 decimais string com ponto, `z_score` número | adotada | Tasks 7, 9 |
| C5 `/lab/exams` com `id` e `analytes` | adotada (liberador e admin já passavam no portão do 21b) | Task 4 |
| C6 `reason` na troca e no aceite | adotada (`reason_required`; guardado em `lab_qc_targets.reason`) | Tasks 3, 5, 9 |
| C7 `GET /lab/qc/exceptions` | adotada com `month` opcional; **troca:** lê também `lab_staff` (membro do laboratório), não só liberador e admin | Task 8 |
| C8 `GET /lab/inventory/movements?lot_id=` | adotada, do mais novo ao mais antigo | Task 13 |
| C9 `instrument_name` no `qc_blocked` | adotada | Task 8 |
| C10 `reagent_lot` e `lot_id` em `<qc_value>` | adotada (mantém `reagent_lot_id`) | Task 7 |
| C11 contador `{ count }` sem trilha | adotada; **troca:** o admin também lê (todos os laboratórios) | Task 7 |
| C12 envio e leitura do PDF do CEQ | adotada (`POST /lab/eqa/uploads`, `GET /lab/eqa/rounds/:id/annual_report.pdf`); **troca:** leitura sem trilha (não há dado de paciente) | Task 12 |
| C13 aparelhos com qualquer dos dois interruptores; `lab_instrument_id` na versão | adotada; 403 leva `feature: "lab_quality_control"` | Task 4 |
| C14 ação corretiva por `lab_staff` e `lab_releaser` | adotada (sem step-up) | Task 7 |
| C15 quantidade inteira, sinal do api salvo no ajuste | igual ao plano | Task 13 |
| C16 `is_default`; justificativa só ao desligar regra ligada | adotada | Task 7 |
| C17 `from`/`to` obrigatórios; `target` null | adotada (`filter_required`; até 366 dias) | Task 9 |
| M1 todos os ambientes, independentes | igual ao plano (sem `environments`; nenhum `requires` entre eles) | Task 2 |
| M2 aviso do `fake-middleware` cita a corrida reprovada | concordo; reforçado no runbook (abre bloqueio em staging) | Task 18 |
| M3 reusa `municipal_lab_disabled` | igual ao plano | Task 2 |
