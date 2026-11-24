# Módulo 27b — Contratos entre apps

**Fonte única** do que o api, o dashboard e o maintenance combinam no 27b. As linhas do §11 prevalecem sobre o texto das seções anteriores. Os planos de cada repositório (`-api.md`, `-dashboard.md`, `-maintenance.md`) seguem este documento. Uma divergência vai para a seção "Divergências propostas ao contrato" do plano, e a decisão volta aqui como Q-n.

- **Spec:** `docs/superpowers/specs/2026-11-24-module-27b-bnafar-design.md`, aprovada em 2026-11-24, decisões 1–14 e §4.
- **Pesquisas:**
  - `docs/pesquisa/2026-11-24-horus-e-si-bnafar-27b.md`, com a API v1.25, os domínios DM01/DM12/DM13/DM14 e os achados 1–16;
  - `docs/pesquisa/2026-11-23-insumos-e-bnafar-27.md`.
- **Base:** contrato e spec do 27a (ADR 0045), mais o que o 29, o 32b, o 35a e o 35b acrescentaram ao estoque. Nada disso muda, salvo o que este documento diz.
- **Sem contracts nem campo.** O 27b não toca o `field-sync`.

Convenções herdadas (27a, 33, 35a e 35b):
- erros `{ "error" }`, com `path` nas recusas de validação;
- step-up `401 mfa_required`;
- interruptor desligado `403 feature_disabled`, com `feature`;
- listas `{ items, next_before }`;
- registro de outro escopo por id → 404;
- concorrência otimista por `expected_version` → 409 `<entidade>_changed`;
- texto longo é recusado, nunca cortado;
- segredo nunca em log, URL, evento nem resposta.

**Bases.** Origin/main de cada repo no dia da Task 0. O 27b começa depois do merge do 35b.

**Valores fixos:**

| Item | Valor |
|---|---|
| Porta do api | 3062 |
| Vite do dashboard | 5223 |
| maintenance | 5177, com o serviço do compose parado durante a prova |
| Migrações da cidade | a partir de `20261125900001` (confira a última na main na Task 0) |
| Migração de plataforma | a partir de `20261125600001` |
| Branch (os três repos) | `feat/mod-27b-bnafar` |
| Worktree do api | `apps/api/.claude/mod27b`, com `config/master.key` copiada |
| Worktrees dos fronts | `rota-saude/.worktrees/<repo>-mod27b` |
| Simulado do SI-BNAFAR | `lib/fake_bnafar`, no molde do `lib/fake_rnds`; container `fake-bnafar-dev` na porta 8095 |
| Testes | só `bin/test api -w mod27b …` e `bin/test <app> -w <repo>-mod27b`; suíte completa pela sessão "Testes" |

**Mudança transversal.** O 27b toca:
- migração de cidade e de plataforma;
- `CityEncryption::CITY_KEYED_TARGETS`;
- `Platform::Features`;
- `IntegrationCredential`;
- o estoque inteiro (`stock_lots`, `stock_locations`, `stock_movements`, `stock_entries`), que serve ao 27a, 29, 32b, 35a e 35b;
- o catálogo de medicamentos da plataforma;
- o YAML de imunobiológicos.

A suíte completa do api é obrigatória antes do merge.

## 1. Interruptores e credencial

**Interruptores** (fim do `CATALOG` de `Platform::Features`, nesta ordem):

| Chave | Rótulo curto | `requires` |
|---|---|---|
| `bnafar` | "Envio à BNAFAR" | `%w[credential:bnafar ibge_code]` |
| `bnafar_medications` | "BNAFAR: medicamentos" | `%w[feature:bnafar feature:pharmacy]` |
| `bnafar_vector_supplies` | "BNAFAR: endemias e zoonoses" | `%w[feature:bnafar]` e o domínio vetorial utilizável: o `missing_for` aceita `vector_control` **ou** `zoonoses`, com o motivo `vector_supplies_disabled` |
| `bnafar_vaccines` | "BNAFAR: vacinas" | `%w[feature:bnafar feature:vaccination]` |

- As quatro chaves entram em `CITIZEN_HIDDEN`.
- Gate: `Bnafar::Gate.usable?(city)` = `bnafar` utilizável. `Bnafar::Gate.domains(city)` devolve os domínios de `Stock::Domain` ligados: `MEDICATION`, `VECTOR_CONTROL`, `IMMUNOBIOLOGICAL`.
- **Desligar:** o job não monta nada novo, e o que já foi enviado fica.
- **Ligar um domínio:** vale a partir do dia seguinte. O job só monta D-1 de domínios que estavam ligados no fim de D-1.

**Credencial `bnafar`:**
- `IntegrationCredential::KINDS` ganha `bnafar`, com `SECRET_KEYS` `username` e `password`, cifrados como os da `ledi`.
- A tabela ganha `environment` (`homologation` | `production`, padrão `production`). O índice único passa de `kind` para (`kind`, `environment`).
- As credenciais existentes ficam `production`.
- Quem grava é o `municipal_admin`, pelo `PUT /integrations/credentials/bnafar`, com step-up e o corpo `{ username, password, environment }`.
- A credencial em uso é a do ambiente escolhido em `bnafar_settings.environment` (§3). O padrão é `production`, e só o maintenance troca, em cidade de staging.
- O teste de conexão faz `POST …/check`, que pede um token novo.
- `Platform.audit("bnafar.credential_set")` entra no R18.

**Cliente** (`Bnafar::Client`, `Bnafar::Config`):
- As URLs vêm de `Rails.application.credentials.bnafar`, por ambiente: `auth_url`, `api_url`.
- `FAKE_BNAFAR_URL` vale com o interruptor de simulado `bnafar_mock` (`environments: development, test, staging`, como o `rnds_mock`).
- **Autenticação:** `POST {auth_url}` com `Authorization: Basic`, que devolve `access_token` e `expires_in`. As chamadas levam `Authorization: jwt <token>`.
- **Cache:** em memória, pela chave `[city.id, credential.set_at, environment]`. A validade é `expires_in` menos 2 minutos. O manual diz que `expires_in` vem em ms, e o cliente aceita ms e s: valor acima de 86.400 é ms.
- **401:** repete uma vez com token novo.
- **Respostas:**
  - 400 e 422 → `rejected`, com os códigos MSG;
  - 401 e 403 → `unauthorized`;
  - o resto → transitório.

## 2. Dados novos do lote, do local e do produto

**`stock_lots`**:
- Colunas novas: `unit_price_cents` (int > 0), `manufacturer_cnpj` (14 dígitos), `anvisa_registration` (11 ou 13 dígitos), `health_program` (sigla do DM14, texto até 20), `ium` (text[], opcional), `ampp_code` (texto até 20, medicamento e vacina), `bnafar_version` (int ≥ 1).
- Todas aceitam nulo: o lote anterior ao 27b nasce incompleto.
- **Na entrada** (`Pharmacy::RecordEntry`): cada item aceita os campos novos. Com o domínio do local ligado ao BNAFAR, os obrigatórios para o envio passam a ser exigidos, com `path` `items.<i>.<campo>`:
  - medicamento e vacina: `unit_price_cents`, `manufacturer_cnpj`, `anvisa_registration`, `health_program`, `ampp_code`;
  - produto vetorial: os mesmos, sem `ampp_code`.
- **Lote já existente** que recebe nova entrada com valor diferente: o lote guarda o valor da entrada mais recente.
- **Complemento:** `PATCH /pharmacy/lots/:id/bnafar` com `expected_version` → 409 `lot_bnafar_changed`. Quem pode: `stock_keeper` ou `pharmacist` do local, ou `municipal_admin`.

**`stock_locations`**:
- `cnes` (7 dígitos), só para `central` e `cold_chain_central`. Os demais usam o `cnes` da unidade.
- O tipo DM01 do local fica em `Bnafar::LocationTypes`, no código:
  - `central` → `A`;
  - `unit` → `F`;
  - `cold_chain_central`, `vaccine_room` e `vector_control`: o plano do api mapeia contra o DM01 da pesquisa, com spec.

**Catálogo de medicamentos (plataforma):**
- `medication_catalog_items.obm_code` passa a se chamar `vmp_code`, e o CHECK que o reservava sai.
- Tabela de plataforma nova `obm_presentations`: `ampp_code` (único), `vmp_code`, `anvisa_registration`, `manufacturer_cnpj`, `description`, `active`.
- A carga vem do portal OBM pela mutation de maintenance `importObmCatalog`, no molde do `importMedicationCatalog`, com auditoria `maintenance.obm_catalog.imported` (R18).
- O plano do api confere a API do portal. Sem acesso automático, a carga aceita o arquivo oficial pela mesma mutation, com `fileBase64` (§6).

**Vacinas:** `config/vaccination/immunobiologicals.yml` ganha `obm_vmp` por produto. Uma vacina sem `obm_vmp` fica fora do envio (§4.3).

**Produtos vetoriais:** `vector_products.catmat_code` (int, opcional), editado na tela de produtos do 35a.

## 3. Envio diário

**Tabelas novas da cidade:**
- **`bnafar_settings`** (linha única): `environment`, `started_on` (o primeiro dia enviado, gravado no primeiro envio) e `updated_by`.
- **`bnafar_submissions`** (uma por cidade, dia de referência e assunto):
  - `reference_on`, `subject` (`position` | `exit`), `environment`;
  - `status`: `building` | `queued` | `sending` | `processing` | `finished` | `failed` | `lost`;
  - `protocol`, `attempts`, `next_attempt_at`, `sent_at`, `finished_at`, `last_error` (só o código);
  - único por (`reference_on`, `subject`, `environment`).
- **`bnafar_records`**:
  - `bnafar_submission_id`, `origin_code` (uuid, o `codigoOrigem`), `subject`, `stock_location_id`;
  - `item_key` (domínio e chave do item), `payload` (cifrado; nulo depois de 30 dias do aceite);
  - `bnafar_code`, `status` (`pending` | `accepted` | `rejected`), `error_codes` (text[], só códigos MSG);
  - `replaces_record_id` (no PUT), `sent_at`.
- **`bnafar_movement_links`:** `stock_movement_id` (único), `bnafar_record_id`. É o rastro de cada saída.

**Granularidade** (achado 5):
- Posição de estoque: um registro por (estabelecimento, item), com um item de payload por lote. Até 60 lotes por registro; acima disso, mais registros com o mesmo par. O teto de 60 ou 20 se confirma na homologação e fica em `Bnafar::Limits`.
- Saída: um registro por movimento.

**Job:**
- `Bnafar::DailyJob` (`EachCityJob`), na agenda de cidade `config/recurring.yml`, às 01:00 no fuso da cidade. Ele monta e enfileira as remessas de D-1 dos domínios ligados.
- `Bnafar::DeliverJob`, a cada 5 minutos: envia as remessas `queued`, consulta o processamento das `processing` até o status final, lê as inconsistências e grava `bnafar_code` e erros por `origin_code`.
- Claim com `SKIP LOCKED` e backoff no padrão da fila da RNDS, com tabela própria.
- Lote de até 1.000 registros por remessa. Acima disso, a remessa se divide em partes, com o protocolo de cada parte.

**Pendências** (`Bnafar::Pendencies`, calculada na leitura, sem tabela):
- `lot_incomplete`, `location_without_cnes`, `product_without_code`, `fractional_balance`, `exit_type_missing`, `domain_off`.
- Cada pendência traz local, item e lote, sem pessoa. Um item pendente não entra no registro, e o resto do estabelecimento é enviado.

**Recusas e correções** (decisão 10):
- **Registro recusado em D+1:** a correção é feita no dado de origem (complemento do lote, CNES do local, código do produto). `POST /pharmacy/bnafar/submissions/:id/retry` remonta e reenvia só os registros recusados, desde que ainda seja D+1 no fuso da cidade. Depois disso, a remessa vira `lost` para esses registros, com o motivo.
- **Saída externa já aceita:** a correção do destino ou do motivo, até 30 dias do aceite, é um PUT (`POST /pharmacy/bnafar/records/:id/correct`). Ele gera um registro novo com `replaces_record_id`.
- **Posição de estoque:** nunca tem PUT. A fotografia seguinte já sai corrigida.

**Alerta** (`Bnafar::Alert`, calculado na leitura, como o `Rnds::Alert`):
- `late`: remessa de D-1 não `finished` depois das 15:00 de D+1 no fuso da cidade.
- `lost`: registros perdidos nos últimos 30 dias.
- `rejected`: registros recusados ainda corrigíveis.
- O dashboard e o maintenance mostram a faixa (§7 e §8).

**Mapeamentos no código, com spec:**
- **`Bnafar::ExitTypes`:**
  - `transfer_out` → S-T;
  - `loss` → S-PE;
  - `expiry` → S-VV;
  - `adjustment` negativo → S-AE;
  - saída externa → o tipo escolhido.
  - O plano do api decide `administration`, `vial_opening`, `immunization`, `vector_application`, `animal_vaccination` e `collar_placement` contra os 21 tipos do DM12. Movimento sem tipo vira pendência `exit_type_missing` e não é enviado.
  - `dispensing` nunca vai.
- **`Bnafar::ProductTypes`:** o DM13 derivado do DM14 do lote (decisão 11).
- **`Bnafar::LocationTypes`:** o DM01 (§2).

**Quantidade:** é inteira na menor unidade de fornecimento. O saldo ou movimento com fração fica como pendência `fractional_balance`. A conversão de embalagem, quando a unidade local difere da de fornecimento, fica para o plano do api.

**IBGE:** `CityProfile.current.ibge_code[0, 6]`.

**Homologação** (decisão 13):
- Com `environment` `homologation`, o cliente troca CNES, produtos e programas pela massa de dados do Ministério, segundo `config/bnafar/homologation.yml`.
- Cada chamada grava a evidência em cURL, sem token nem senha, em `bnafar_records.evidence` (texto). Essa coluna só existe em homologação: em produção ela fica nula.

## 4. Saída externa

- **Tabela nova `stock_external_exits`:**
  - `stock_location_id`;
  - `exit_type` (os tipos do DM12 cobertos: devolução ao fornecedor, remanejamento a outro ente, doação e empréstimo, com os códigos da pesquisa);
  - `destination_kind` (`cnes` | `cnpj`), `destination_code`;
  - `reason` (cifrado, 10–500), `recorded_by_user_id`, `recorded_at`.
- **`stock_external_exit_items`:** `stock_lot_id`, `quantity` (inteira).
- **Movimento negativo novo** `external_exit`, com o documento `stock_external_exit_id`. Os CHECKs de kind, de documento e de sinal são trocados (`replace_check`).
- **Rotas:** `POST /pharmacy/external_exits`, com step-up, e `GET /pharmacy/external_exits`. Quem pode: o `stock_keeper` do local, e o `municipal_admin` só lê.
- **Saldo:** o saldo insuficiente recusa a saída (`insufficient_balance`), como na transferência. A saída externa não é consumo de campo.

## 5. Saldo inicial pela planilha

**Planilha padrão:** CSV UTF-8, separado por `;`, com cabeçalho exato:

`cnes;tipo_produto;codigo_produto;ampp;lote;validade;quantidade;valor_unitario;cnpj_fabricante;registro_anvisa;programa`

- `tipo_produto`: `medicamento` | `vacina` | `vetorial`.
- `codigo_produto`:
  - CATMAT para medicamento;
  - código do imunobiológico (YAML) para vacina;
  - `code` do produto vetorial.
- `validade`: `AAAA-MM-DD`.
- `quantidade`: inteira.
- `valor_unitario`: decimal com vírgula.
- O api publica um modelo vazio em `GET /maintenance/opening_import_template.csv`.

**Tabelas novas da cidade:**
- **`opening_imports`:**
  - `status`: `uploaded` | `validated` | `applied` | `discarded` | `reverted`;
  - `file_sha256`, `file_data` (cifrado; apagado ao aplicar ou descartar);
  - `rows_total`, `rows_valid`;
  - `uploaded_by_maintainer_id`, `applied_by_maintainer_id`, `applied_at`.
- **`opening_import_rows`:** `opening_import_id`, `line`, os campos lidos, `stock_location_id` (resolvido pelo CNES), o item resolvido, `status` (`valid` | `error`) e `error_codes`.

**GraphQL do maintenance** (todas com `citySlug`, auditadas, com nomes novos no R18):
- `uploadOpeningImport(fileBase64, filename)`: até 5 MB, lê e valida. Devolve o relatório: linhas válidas, erros por código, produtos sem catálogo, locais sem CNES e totais por local.
- `openingImport(id)`: consulta o relatório e as linhas, paginadas.
- `applyOpeningImport(id, totp)`: com step-up TOTP, como a publicação de lista de agravos. Transforma as linhas válidas em entradas `opening` por local (até 100 itens por entrada, `MAX_ENTRY_ITEMS`), numa transação por local.
  - Linha de item que já tem abertura no local vira erro `opening_exists`.
- `discardOpeningImport(id)`.
- `revertOpeningImport(id, totp)`: só enquanto nenhum local da importação tiver outro movimento nem envio BNAFAR. Grava ajustes negativos que zeram a abertura, com motivo fixo.

**Quem grava a entrada:** a entrada criada pela importação leva `recorded_by_maintainer_id` (uuid da plataforma, sem FK). `stock_entries.recorded_by_user_id` passa a aceitar nulo, com CHECK de exatamente um dos dois. O mesmo vale para os movimentos.

**O dashboard só confere:** a tela "Entradas e abertura" mostra as aberturas, com a marca "importada pela plataforma".

## 6. Rotas do api

| Rota | Quem |
|---|---|
| `GET /pharmacy/bnafar/status` (alerta e últimos 30 dias) | `municipal_admin`, `stock_keeper` de qualquer local |
| `GET /pharmacy/bnafar/submissions` (filtros `from`, `to`, `subject`, `status`) e `GET /pharmacy/bnafar/submissions/:id` (registros com erros) | os mesmos |
| `POST /pharmacy/bnafar/submissions/:id/retry` | `municipal_admin`, com step-up |
| `POST /pharmacy/bnafar/records/:id/correct` | `municipal_admin`, com step-up; só saída externa aceita, até 30 dias |
| `GET /pharmacy/bnafar/pendencies` (filtros `location_id`, `code`) | os mesmos do status |
| `PATCH /pharmacy/lots/:id/bnafar` | `stock_keeper` ou `pharmacist` do local, ou `municipal_admin` |
| `POST /pharmacy/external_exits`, `GET /pharmacy/external_exits` | §4 |
| `PUT /integrations/credentials/bnafar`, `POST …/check` | `municipal_admin` (§1) |
| GraphQL do maintenance: `importObmCatalog`, `uploadOpeningImport`, `openingImport`, `applyOpeningImport`, `discardOpeningImport`, `revertOpeningImport`, `setBnafarEnvironment(citySlug, environment)` e `cityBnafarStatus(citySlug)` | mantenedor |

Todas as rotas de cidade exigem `Bnafar::Gate.usable?`, exceto credencial e complemento do lote, que funcionam antes de ligar.

## 7. Dashboard

- **Grupo "Farmácia":**
  - "Envio à BNAFAR" (`bnafar-status`): a faixa de alerta, a lista por dia e assunto com a situação, os registros recusados com o motivo e o prazo, e "Reenviar recusados" (D+1) e "Corrigir" (saída externa) com step-up.
  - "Pendências do envio" (`bnafar-pending`): as pendências por código, com o atalho para o complemento do lote, o CNES do local e o produto.
  - "Saída externa" (`external-exits`).
- **Na entrada e na abertura:** os campos novos, obrigatórios quando o domínio do local está ligado. O AMPP é escolhido entre as apresentações do VMP do item.
- **Locais e configurações:** o CNES do almoxarifado central e da rede de frio central.
- **Produtos de endemias:** o CATMAT.
- **Integrações:** a credencial `bnafar` (usuário, senha e ambiente) e o teste de conexão.
- **Frases:** `BNAFAR_ERRORS` em `src/lib/bnafarErrors.ts`, com o código e a frase de cada pendência e de cada recusa MSG.

## 8. Maintenance

- **Interruptores:** os quatro do §1 com `FEATURE_LABELS`. `MISSING_LABELS` ganha:
  - `credential_missing:bnafar`;
  - `credential_unauthorized:bnafar`;
  - `bnafar_disabled`;
  - `vector_supplies_disabled`.
- **"Importar saldo inicial":** na aba da cidade, envio do CSV, relatório de conferência, aplicar e desfazer com TOTP, e descartar. A tela lê o arquivo no navegador e manda em base64, porque o maintenance não tem envio de arquivo.
- **Situação do envio:** na aba da cidade, a faixa do `Bnafar::Alert` e o ambiente, que só se troca em cidade de staging.
- **Catálogo OBM:** a tela de catálogo de medicamentos ganha "Importar OBM".

## 9. Eventos e LGPD

**Eventos de domínio** (só ids):
- `bnafar_submission.queued`
- `bnafar_submission.finished`
- `bnafar_submission.lost`
- `bnafar_record.rejected`
- `bnafar_record.corrected`
- `stock_external_exit.recorded`
- `stock_lot.bnafar_completed`
- `opening_import.applied`
- `opening_import.reverted`

**`Platform.audit`, nomes novos no R18:**
- `bnafar.credential_set`
- `maintenance.obm_catalog.imported`
- `maintenance.opening_import.uploaded`
- `maintenance.opening_import.applied`
- `maintenance.opening_import.discarded`
- `maintenance.opening_import.reverted`
- `maintenance.bnafar.environment_changed`

**Sigilo:**
- Nenhum dado de pessoa vai à BNAFAR: a dispensação não entra, e o S-PA fica fora.
- O payload cifrado é apagado em 30 dias.
- O CSV é cifrado e apagado.
- Usuário e senha nunca aparecem em log, evento nem cURL.
- Spec de sigilo `spec/security/bnafar_secrecy_spec.rb`: log, eventos, evidência cURL e respostas sem segredo.

## 10. Entrega e testes

**Ordem:**
1. api, incluindo a migração de plataforma e o simulado;
2. dashboard e maintenance;
3. suíte completa;
4. prova contra o simulado;
5. homologação real, quando houver credencial;
6. merges com autorização.

**Testes do api:**
- os mapeamentos DM01, DM12 e DM13;
- a granularidade e a divisão em lotes;
- D+1 e `lost`;
- a correção de 30 dias;
- o `expires_in` em ms e s;
- a planilha (cada erro de linha, abertura repetida, local sem CNES, desfazer bloqueado depois de movimento);
- o sigilo;
- a suíte completa.

**Fronts:** vitest e prova no navegador.

**Gates de go-live:** os do spec §7.

## 11. Decisões (Q-n)

| # | Decisão |
|---|---|
| Q1 (coordenador, contrato) | A planilha vai em base64 pela GraphQL do maintenance, com limite de 5 MB, porque o maintenance não tem envio de arquivo. É o mesmo padrão da credencial da RNDS no dashboard e do upload de exames. |
| Q2 (coordenador, contrato) | Pendência e alerta são calculados na leitura, sem tabela, como o `Rnds::Alert`. Só o que foi enviado fica gravado. |
| Q3 (coordenador, contrato) | A entrada vinda da importação leva o mantenedor como autor (`recorded_by_maintainer_id`), e não um usuário da cidade. A trilha mostra "importada pela plataforma". |
| Q4 (coordenador, contrato) | O AMPP fica no lote, e o VMP no catálogo (decisão 9 da spec). A tabela `obm_presentations` é da plataforma, porque é igual para todas as cidades. |
| Q5 (coordenador, contrato) | `expires_in` aceito em ms ou em s, porque o manual diz ms e o exemplo dele está truncado. A homologação confirma, e o valor real vai para o runbook. |
