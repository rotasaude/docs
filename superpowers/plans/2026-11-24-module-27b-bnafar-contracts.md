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
| Q6 (api D1, dashboard D10, maintenance DM6) | **Simulado.** `bnafar_mock` é a quinta chave do `CATALOG`, só fora de produção (`environments: development, test, staging`), com o rótulo "BNAFAR simulado" e o mesmo aviso de modo simulado do `rnds_mock` no maintenance. "Domínio ligado" quer dizer interruptor utilizável. |
| Q7 (api D2–D4) | **Lote e formatos.** O lote continua só de acréscimo, com guard próprio que aceita o complemento (de nulo a valor) e a troca do valor unitário. Uma entrada nova no mesmo lote com CNPJ, registro ANVISA ou AMPP diferentes volta 422 `lot_mismatch`. O CNPJ tem 14 caracteres alfanuméricos (formato novo da Receita). O AMPP tem até 25 caracteres. O programa vem só da lista DM14 (até 10). O número de lote acima de 30 caracteres vira pendência `lot_number_too_long`. |
| Q8 (api D5–D7, D21) | **Remessa e registro.** Uma remessa pode ter vários protocolos: tabela `bnafar_batches` (protocolo, situação e tentativas por parte). A evidência cURL é por chamada, em `bnafar_batches`. O registro de posição aceito em parte fica `accepted`, com `rejected_items`. A correção de saída externa vale para a saída inteira, ou seja, para todos os registros dela. |
| Q9 (api D8–D9, maintenance DM3) | **Ambiente e credencial.** Em produção, o ambiente é sempre `production`. A credencial e o ambiente `homologation` só existem fora de produção (staging e dev), onde o maintenance troca o ambiente. `GET /integrations` mostra a credencial `bnafar` do ambiente em uso, e o `city_state` confere essa. |
| Q10 (api D10–D11) | **Desfazer a abertura.** O desfazer é um movimento negativo novo, `opening_reversal`, e não um ajuste. `stock_openings` ganha `reverted_at`, e os índices únicos passam a parciais (`WHERE reverted_at IS NULL`), para permitir reaplicar. |
| Q11 (api D12, D14, D20; maintenance DM1; muda a decisão 6 da spec) | **A importação do saldo inicial é feita pela cidade, no dashboard.** O maintenance não existe em produção, então a importação pela plataforma não chegaria à cidade real. Quem importa é o `municipal_admin`, ou o `stock_keeper` do central. A rota é REST: `POST /pharmacy/opening_imports` (JSON com `file_base64`, até 5 MB), `GET /pharmacy/opening_imports/:id` (relatório e linhas), `POST …/:id/apply` e `POST …/:id/revert` com step-up, `POST …/:id/discard` e `GET /pharmacy/opening_imports/template.csv`. O autor das entradas é o usuário da cidade, e a Q3 sai. O §5 vale com essa troca de lugar, e a GraphQL de importação sai do maintenance. |
| Q12 (api D13) | **Carga OBM.** O portal OBM não tem API nem download automático, então a carga é por arquivo, com mapa de colunas. Em produção, pela rake `medications:import_obm[path]`, rodada pelo operador da plataforma. Fora de produção, também pela mutation do maintenance. O `vmp_code` do catálogo é preenchido pela coluna CATMAT do arquivo OBM. O item sem correspondência vai para a lista de revisão. |
| Q13 (api D15–D17, D19; dashboard D5, D7) | **Rotas e dados que faltavam.** O CNPJ do fabricante da vacina fica por fabricante no YAML. O CNES do central é gravado por `PATCH /pharmacy/admin/locations/:id`, e o da rede de frio por `PATCH /vaccination/admin/cold_chain_central`, os dois pelo `municipal_admin`. As aberturas se leem por `GET /pharmacy/openings`. O destino é obrigatório também na transferência interna (S-T leva o CNES do local de destino). |
| Q14 (api D18) | **Estorno de uso direto.** O estorno de uma saída já aceita há menos de 30 dias apaga o registro original na BNAFAR (`DELETE`). Depois de 30 dias, a fotografia de estoque seguinte já corrige o saldo, e o estorno fica registrado como "estorno fora da janela". |
| Q15 (api D22; dashboard D3) | **Produto sem código.** O payload leva `numero` e `codigoVmp`. O medicamento sem VMP no catálogo pode entrar no estoque: o AMPP só é exigido quando o item tem VMP, e o lote fica com a pendência `product_without_code`. O fabricante internacional sem CNPJ entra pelo campo de fabricante internacional, com o CNPJ vazio, como o manual prevê. |
| Q16 (dashboard D1, D4, D6, D8–D10) | **Formas para as telas.** Registros e pendências trazem `location_name` e o rótulo do item, sem pessoa. O `protocol` vem por parte (Q8). Entram `GET /pharmacy/bnafar/domains` (programas DM14 e tipos de saída) e `GET /pharmacy/obm_presentations?vmp_code=`. O horário do alerta é "15:00 de D+1 no fuso da cidade", e o registro perdido traz `lost` e o motivo. A saída externa exige só o estoque do domínio utilizável, sem o `bnafar`, e o seletor mostra os locais de todos os domínios. O complemento do lote fica dentro de "Pendências do envio à BNAFAR", sem tela própria. O lote antigo começa com `bnafar_version` 0. O rótulo do menu é "Pendências do envio à BNAFAR". |
| Q17 (dashboard D2; maintenance DM2, DM4, DM5) | **Convenções.** O `path` das recusas é o do api real, `items[i].campo`. No SDL do maintenance, o argumento de TOTP é `code`, com `expectedVersion` onde houver versão. Os códigos de erro de linha da planilha e a aceitação do BOM do Excel são os do plano do api. O maintenance fica só com interruptores, ambiente (fora de produção), situação do envio e carga OBM. |
| Q18 (api D20) | **Gate de go-live.** Os interruptores do 27b, como todos os outros, dependem do maintenance em produção, que ainda é uma pendência da plataforma. O 27b herda esse gate e não cria outro. |
| Q19 (api D23–D24) | **Gate das rotas de apoio e auditoria da rake.** A importação do saldo inicial, o `obm_presentations` e o `bnafar/domains` exigem só o estoque do domínio utilizável (`pharmacy`, `vaccination` ou o vetorial), sem o `bnafar`. A cidade que sai do Hórus abre o estoque antes de ligar o envio. A rake `medications:import_obm` grava `Platform.audit("obm_catalog.imported")`, nome novo no R18, com o ator `system`. |
| Q20 (dashboard D11–D12) | **Domínios utilizáveis e lista de importações.** O `GET /pharmacy/bnafar/domains` traz `enabled_domains`, os domínios com o interruptor utilizável. A tela de entrada exige os campos de envio só nesses domínios. Entra `GET /pharmacy/opening_imports`, a lista de importações da cidade. Os atos `apply`, `revert` e `discard` levam `expected_version` → 409 `opening_import_changed`. As rotas do saldo inicial não exigem o `bnafar` (Q19). |
| Q21 (api pré-voo Q-A) | **CNPJ do fabricante da vacina.** Na vacina, o CNPJ vem do YAML de fabricantes pela `manufacturer_key` (Q13), e não é pedido na entrada, no complemento nem na planilha. O payload usa o CNPJ do YAML. Sem CNPJ no YAML, o `LotFields.missing` aponta `manufacturer_cnpj`, e o lote fica com a pendência `lot_incomplete`. O formulário do dashboard não mostra o campo para a vacina. Medicamento e produto vetorial seguem com o CNPJ digitado (§2). |
| Q22 (api pré-voo Q-B) | **Pendências antes de ligar o envio.** O `GET /pharmacy/bnafar/pendencies` exige só o estoque de um domínio utilizável, sem o `bnafar` (como a Q19). Com o `bnafar` desligado, a conta é feita contra os domínios de estoque utilizáveis, como se o envio fosse ligado, e o `domain_off` é omitido. Assim a cidade completa os lotes antes de ligar. Com o `bnafar` ligado e a credencial recusada, vale a regra normal, com `domain_off` (Q41). |
| Q23 (api pré-voo Q-C) | **Dados de origem sem o `bnafar`.** A exceção da Q19 vale também para o `GET /pharmacy/openings`, para o `cnes` do `PATCH /pharmacy/admin/locations/:id` e do `PATCH /vaccination/admin/cold_chain_central`, e para o `catmat_code` do produto vetorial. A regra geral do §6 passa a ser: o dado de origem (lote, local, produto, abertura) exige só o estoque do domínio. O que é do envio (status, remessas, retry, correção) exige o `bnafar`. |
| Q24 (api pré-voo Q-D) | **Regra do desfazer.** O §5 ("nenhum outro movimento nem envio") passa a ser lido assim: o desfazer é bloqueado (`revert_blocked`) por movimento em um local da importação criado depois do `applied_at`, fora os da própria importação, ou por qualquer `bnafar_records` desses locais criado depois do `applied_at`, enviado ou não (comparação por `created_at`). O movimento anterior ao `applied_at` não bloqueia. |
| Q25 (api pré-voo Q-E) | **Âncora dos 30 dias.** A janela de 30 dias conta do `sent_at` do registro, como diz o manual (pesquisa, achado 6), e não do aceite. Isso vale para todo o 27b: a correção da saída externa (§3), o `DELETE` do estorno (Q14), as telas e o simulado. |
| Q26 (api pré-voo Q-F) | **Campos de envio na planilha.** O `Validator` da planilha segue a regra da entrada: exige os campos de envio só nos `enabled_domains` (`Gate.required_for?`). O relatório ganha a contagem de linhas sem os campos de envio, só como aviso. Esses lotes aparecem depois como `lot_incomplete` nas pendências (Q22). |
| Q27 (api Task 9, resumo mensal de vacinas) | **Saída externa e desfazer no resumo mensal (V14).** O `Vaccination::MonthlySummary` ganha duas colunas, em doses positivas: `external_out` (os movimentos `external_exit` do mês) e `opening_reverted` (os `opening_reversal` do mês). A abertura continua somada em `received`, como hoje. A conta por linha fica `closing = opening + received + transferred_in − transferred_out − external_out − opening_reverted − vial_doses − unit_doses + adjustment − Σ physical_loss`. A mudança no JSON só acrescenta campos, e a tela de resumo de vacinas do dashboard mostra as duas colunas novas. |
| Q28 (api Task 9) | **Lote bloqueado na saída externa.** A saída externa recusa, em todos os tipos, o lote com bloqueio ativo por excursão de temperatura (`422 lot_blocked`, ADR 0047), como a transferência. A devolução ao fornecedor de um lote bloqueado só acontece depois da conduta: descartar vira perda, liberar permite a saída. |
| Q29 (api Task 9) | **Lote vencido na saída externa.** O lote vencido só sai em S-DF (devolução ao fornecedor). Em S-TR, S-D e S-E a saída volta `422 lot_expired`, como a transferência. |
| Q30 (api Task 11) | **Purga do payload.** O §9 vale como está: todo payload cifrado de `bnafar_records` é apagado 30 dias depois do `sent_at`, seja qual for a situação (aceito, recusado, perdido ou estorno fora da janela). O registro nunca enviado é apagado 30 dias depois do `created_at`. Os metadados, como situação, códigos, protocolo e evidência cURL sem segredo, continuam. Depois da purga, a correção e o `DELETE` só usam os códigos guardados, e essa janela também é de 30 dias (Q25). |
| Q31 (api Task 11) | **O fechamento não perde remessa em processamento.** O `close!` só marca `lost` a parte que não foi enviada ou que foi recusada e não reenviada até o fim de D+1. A parte com protocolo em `processing` continua sendo consultada pelo `DeliverJob` até a BNAFAR responder. Enquanto isso, o alerta `late` (15:00 de D+1) avisa a cidade, e a parte não vira `lost`. |
| Q32 (api Task 12) | **Quem lê a situação e as pendências.** Os leitores de `GET /pharmacy/bnafar/status`, das remessas e de `GET /pharmacy/bnafar/pendencies` são: `municipal_admin`, `stock_keeper` ou `pharmacist` de qualquer local, e o responsável de qualquer sala de vacina (`room_lead`). Assim quem completa o lote (complemento, §2) enxerga a tela em que o complemento fica (Q16). O reenvio e a correção continuam só com o `municipal_admin`, com step-up. |
| Q33 (api Task 12) | **Pendências paginadas.** `GET /pharmacy/bnafar/pendencies` volta `{ items, next_before, totals }`, com padrão de 50 itens e máximo de 100, como as outras listas. Os filtros continuam (`location_id`, `code`). `totals` traz a contagem por `code` e por local, sem filtro de página, para a cidade que vem do Hórus ver o tamanho do que falta. |
| Q34 (api Task 12) | **Alerta da parte em processamento.** Pela Q31, a parte em `processing` nunca vira `lost`. Por isso o `Bnafar::Alert.late` vale enquanto houver parte de remessa de D-1 ou anterior não `finished` depois das 15:00 de D+1, sem limite em R+2. |
| Q35 (api Task 12) | **Correção no ambiente em uso.** O `POST /pharmacy/bnafar/records/:id/correct` só acha registro do ambiente em uso (Q9). Registro de outro ambiente volta 404, como as leituras. |
| Q36 (api Task 14) | **AMPP de outro produto na planilha.** Entra o código de linha `ampp_not_in_vmp`, com o mesmo texto da entrada manual. Ele vale para o AMPP que existe no catálogo OBM mas pertence a outro VMP. O `ampp_unknown` fica para o AMPP que não existe no catálogo. |
| Q37 (api Task 14) | **Conflito de lote na conferência.** A conferência detecta, antes de aplicar, o código de linha `lot_mismatch` com a regra da Q7: o preço pode diferir, e os outros campos preenchidos (CNPJ, registro ANVISA, AMPP, validade) não. São dois casos: (a) linhas do mesmo lote (mesmo item e número), em qualquer local, com campos divergentes, e aí todas as linhas do conflito viram erro; (b) linha divergente de um lote já gravado na cidade. |
| Q38 (api Task 14) | **Fabricante internacional fora da planilha.** O cabeçalho do §5 continua exato e não ganha coluna. A linha com `cnpj_fabricante` vazio é aceita. O lote nasce sem fabricante e aparece como `lot_incomplete` quando o domínio exige o campo (Q26). O fabricante internacional é completado depois, pelo complemento do lote (Q15). Na vacina, o CNPJ vem do YAML (Q21). Limite da planilha: 20.000 linhas, acima disso `413 file_too_large`. A cidade grande divide a importação por local, e o modelo da planilha diz isso. |
| Q39 (maintenance, Task 15 do api) | **Troca de ambiente e leitura da situação.** O `setBnafarEnvironment` pede sessão humana (CityMutation). Fora de produção, trocar para `homologation` não pede TOTP. Trocar para `production` fora de produção pede o step-up TOTP (`code`), porque faria um staging enviar dado fictício à BNAFAR real. O `cityBnafarStatus` aceita token de leitura, já que só traz ambiente, domínios, alerta e contagens, sem credencial e sem dado de pessoa. |
| Q40 (api, revisão final) | **A situação aparece com a credencial recusada.** `GET /pharmacy/bnafar/status`, `GET /pharmacy/bnafar/submissions` e `GET /pharmacy/bnafar/submissions/:id` exigem o `bnafar` LIGADO, e não utilizável. Assim o dashboard mostra a faixa e o alerta justamente quando a credencial está recusada. `retry` e `correct` continuam exigindo o `bnafar` utilizável. O alerta `late` conta os domínios devidos pelo interruptor ligado (`Gate.domains_due`), no status e no `cityBnafarStatus`. |
| Q41 (api, revisão final; ajusta Q6 e Q20) | **Campos de envio pelo interruptor, não pela credencial.** A entrada (`required_for?`), a planilha (Q26) e o `enabled_domains` do `GET /pharmacy/bnafar/domains` exigem os campos de envio quando o `bnafar` e o interruptor do domínio estão ligados (e o estoque do domínio utilizável), seja qual for o estado da credencial. Um problema de credencial passageiro não pode fazer nascer lotes `lot_incomplete`. |
| Q42 (api Task 18; ajusta Q25 e Q31) | **Fim do reenvio em fuso a oeste de Brasília.** O SI-BNAFAR roda em Brasília e a MSG73 aceita só a data atual ou a do dia anterior. Por isso o reenvio (R+1) termina no MENOR entre o fim de R+1 no fuso da cidade e o fim de R+1 em Brasília (`Bnafar::Window.retry_until`, a mesma régua para o `RetrySubmission`, a entrega e a tela). O `close!`/`lost` usa a mesma régua, e a parte em voo continua sem virar `lost` (Q31). Em cidade no horário de Brasília, nada muda. Na de Manaus, o reenvio termina às 23:00 local, e na do Acre, às 22:00. O simulado calcula as datas em Brasília. |
