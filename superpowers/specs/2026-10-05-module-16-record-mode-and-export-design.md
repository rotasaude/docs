# Módulo 16 — Modo de prontuário e exportação — design

**Data:** 2026-10-05
**Status:** aprovado (2026-10-05)
**Afeta:**
- `contracts`: `session-v1.1.0` (`features` no contrato de sessão).
- `apps/api`:
  - plataforma: `cities.record_mode`, `cities.pec_url` (o IBGE já existe em `city_profile.ibge_code`); `city_features`; terminologias (`terminology_releases`, CID-10, CIAP-2, SIGTAP); retratos do CNES;
  - cidade: `integration_credentials`, `ledi_outbox`, `health_teams`, `health_team_members`, `health_units.cnes`, `professionals.cpf`;
  - `Platform::Features`, `Terminology::*`, `Cnes::*`, `Ledi::*`, `Cadsus::*`; jobs `Ledi::DeliverJob` e alertas de competência; comandos do operador `terminology:import`, `cnes:import`;
  - mutation de interruptor na API de manutenção; rotas novas na cidade e no console.
- `apps/dashboard`: telas Integrações, CNES e Produção e-SUS; CADSUS na validação presencial; uso de `features`.
- `apps/admin`: modo, IBGE e endereço do PEC por cidade; visão de produção de todas as cidades; alerta de SIGTAP não importada.
- `apps/maintenance`: interruptores por cidade.
- `apps/wpda`: nada.

**ADR:** `docs/adr/0028.md` · **Módulo:** `docs/modulos/16--modo-de-prontuario.md`

**Fora desta entrega:** fichas reais (nascem nos módulos 18, 19, 22, 24), RNDS, `maintenance` em produção, migração do CBO para terminologias, telas de busca de terminologia (usadas a partir do 19).

## 1. Ponto de partida

- **Cidade** (`cities`, plataforma): `slug`, `name`, `uf`, `time_zone`, `status`, `database_url`, `encryption_key`. O **código IBGE** mora no `city_profile.ibge_code` do banco da cidade (opcional, gravado no provisionamento).
- **Unidades** (`health_units`, cidade): nome, tipo, endereço, bairro, ativa. Sem CNES.
- **Profissionais** (`professionals`, cidade, ADR 0021): já têm `cns` (obrigatório), conselho, registro, `user_id`; vínculo com unidade e CBO em `professional_links`; CBO num arquivo (`config/professionals/cbo_saude.yml`). Sem CPF, sem equipe.
- **Perfil do cidadão** (ADR 0027): nascimento, sexo, identidade de gênero, `declared`/`verified`. Sem CNS.
- **API de manutenção** (GraphQL): mutations por cidade em `app/graphql/maintenance/mutations` (`CityMutation`); só existe em development, staging e test (`MaintenanceApi.check_boot!` derruba o boot em produção).
- **Contrato de sessão** (`contracts/session/schema.json`, `session-v1.0.0`): `city_slug`, `city_name`, `city_uf`, `scope.city`.
- **Jobs por cidade:** `CityScopedJob`, `EachCityJob`; Solid Queue.
- **Pesquisa:** LEDI 8.7.0 (13/08/2026, PEC ≥ 5.5.26); API do PEC (≥ 5.3.19): `POST /api/recebimento/login` (usuário/senha → cookie) e `POST /api/v1/recebimento/ficha` (binário `.esus`, Thrift TBinaryProtocol); resposta 200 ou 400/500 com mensagem; credencial gerada em "Integração > Credenciais de integração" e não recuperável. Prazo da competência: 10º dia útil do mês seguinte.

## 2. Decisões desta conversa

| # | Decisão |
|---|---|
| 1 | Sem piloto definido: desenho para cidade pequena, PEC centralizador, sem TI. |
| 2 | Exportador entra no 16, provado com fichas sintéticas contra PEC local. |
| 3 | Modo e endereço do PEC pelo operador (console); credenciais pelo `municipal_admin` (dashboard, step-up). |
| 4 | Envio contínuo, com painel da competência e alertas. |
| 5 | Terminologias (CID-10, CIAP-2, SIGTAP) no banco da plataforma, importadas pelo operador. |
| 6 | Identificadores importados do CNES com confirmação da cidade. |
| 7 | CADSUS entra no 16, atrás de interruptor. |
| 8 | Interruptores genéricos por cidade; só o `maintenance` escreve (hoje dev/staging). |
| 9 | Exportador em Thrift/Ruby, com prova técnica primeiro. |

## 3. Interruptores, modo e credenciais

### 3.1 Interruptores
- `Platform::Features::CATALOG`: `{ key, description, requires: [...] }`. Chaves: `ledi_export` (requer `record_mode != off`, `pec_url`, credencial `ledi`), `cadsus_lookup` (requer credencial `cadsus`).
- `city_features` (plataforma): `city_id`, `key` (texto, conferido contra o catálogo), `enabled`, `changed_by_maintainer_id`, `changed_at`; único `(city_id, key)`.
- Mutation `setCityFeature(citySlug, key, enabled)` na API de manutenção; `Platform.audit("city.feature_changed", city_id, key, enabled)` (declarar em `R18_PLATFORM_EVENT_NAMES`).
- `Platform::Features.enabled?(city, key)` e `.usable?(city, key)` (ligado **e** pré-requisitos); `.missing(city, key)` → lista do que falta (para a tela).
- Sessão: `features` = chaves **ligadas** (não necessariamente utilizáveis); o dashboard pergunta o que falta à rota da funcionalidade.
- Recusa: `403 { error: "feature_disabled", feature: <key> }`.

### 3.2 Modo e endereço
- `cities.record_mode` (`off`|`integrated`|`record`, default `off`), `cities.pec_url` (HTTPS, nulo). O código IBGE é o `city_profile.ibge_code` existente (banco da cidade, gravado no provisionamento); a edição pelo console grava lá.
- Console `admin`: edição dos três na ficha da cidade; `Platform.audit("city.record_settings_changed")` (campos mudados, nunca valores de URL).

### 3.3 Credenciais
- `integration_credentials` (cidade): `kind` (`ledi`|`cadsus`, único), `secret` (jsonb cifrado: `ledi` = `{ username, password }`; `cadsus` = `{ username, password }`), `set_by_user_id`, `set_at`, `last_check_at`, `last_check_status` (`ok`|`unauthorized`|`unreachable`|`error`), `last_check_message` (sem segredo).
- Dashboard "Integrações" (`municipal_admin`, step-up): cadastrar/trocar (escrita só), testar conexão (LEDI = login no PEC; CADSUS = consulta de saúde do serviço), ver estado. `GET` nunca devolve `secret`.
- Evento `integration_credential.changed` só com `kind` e `user_id`.

## 4. Terminologias (plataforma)

- `terminology_releases`: `kind` (`cid10`|`ciap2`|`sigtap`), `version` (SIGTAP = `AAAAMM`), `source_sha256`, `imported_by`, `imported_at`, `status` (`importing`|`active`|`superseded`|`failed`); no máximo uma `active` por `kind` e versão.
- `cid10_codes` (`release_id`, `code`, `description`, `sex_restriction`), `ciap2_codes` (`release_id`, `code`, `description`).
- `sigtap_procedures` (`release_id`, `code`, `name`, `sex`, `age_min_months`, `age_max_months`, `complexity`), `sigtap_procedure_cbos`, `sigtap_procedure_cids`, `sigtap_procedure_instruments` (BPA-C, BPA-I, APAC…).
- `terminology:import[kind,version,path]`: lê o arquivo oficial (ZIP SIGTAP do DATASUS; tabelas CID-10 e CIAP-2), valida, grava em transação com `status=importing`, ativa no fim (a anterior vira `superseded`); falha → `failed`, nada ativo muda. Trigger: release `active`/`superseded` imutável.
- `Terminology::Sigtap.compatible?(code, competence:, cbo:, age_months:, sex:, cid:)` → `{ ok:, reasons: [...] }`; competência ausente cai na última `active` anterior a ela.
- Alerta no console: dia 5 sem SIGTAP da competência corrente.

## 5. CNES

- `cnes:import[competence,path]` (operador): lê a base mensal e grava só os municípios com `ibge_code` de cidade ativa em `cnes_snapshots` (`competence`, `ibge_code`, `imported_at`), `cnes_establishments`, `cnes_teams` (INE, tipo, estabelecimento, ativa), `cnes_professional_bonds` (CNS, CPF, CBO, estabelecimento, INE). Retenção: últimas 13 competências por município.
- Cidade: `health_units.cnes` (7 dígitos, único), `health_teams` (`ine` único, `kind` `70`|`76`, `health_unit_id`, `active`), `health_team_members` (`professional_id`, `health_team_id`, `cbo_code`, `started_on`, `ended_on`), `professionals.cpf` (cifrado, dígito verificador).
- `Cnes::Proposal.for(city)`: compara o retrato mais recente com o cadastro e devolve propostas `{ kind: unit|team|member, action: link|create|end, ... , confidence }` e divergências (`no_bond_in_cnes`, `cbo_mismatch`, `team_inactive_in_cnes`, `unit_without_cnes`).
- Dashboard "CNES" (`municipal_admin`): lista propostas e divergências; confirmar item ou lote (`POST /cnes/apply`), com step-up; edição manual pontual nos cadastros.

## 6. Exportador LEDI

### 6.1 Prova técnica (primeira tarefa)
PEC local em container de dev (instalador oficial Linux + PostgreSQL), credencial de integração gerada, classes Thrift 8.7.0 geradas, uma ficha sintética serializada em Ruby e enviada. Critério: 200 e ficha visível na instalação. Registra: compactação exigida, formato do erro 400, efeito de reenviar o mesmo `uuidDadoSerializado`. Falha de serialização → parar e rever a abordagem (serviço Java).

### 6.2 Layout
- `vendor/ledi/<versão>/` (classes geradas) + `bin/ledi-generate <versão>`; `Ledi::Version::ACTIVE = "8.7.0"`.
- `Ledi::Transport.wrap(ficha, city:)`: monta `DadoTransporteThrift` (uuid `CNES-UUID`, tipo, CNES, `codIbge`, INE, lote, `remetente`/`originadora` com `contraChave` "Rota Saúde - <versão>", `uuidInstalacao`, CNPJ do responsável).
- Interface `Ledi::Ficha`: `type`, `competence`, `cnes`, `ine`, `to_thrift`, `source` (tipo/id). No 16: `Ledi::Fichas::Synthetic` (só dev/test).

### 6.3 Fila (`ledi_outbox`, cidade)
`uuid`, `ficha_type`, `competence`, `source_type`, `source_id`, `status` (`pending`|`sending`|`accepted`|`rejected`|`failed`), `attempts`, `next_attempt_at`, `last_error` (sem dado de cidadão), `ledi_version`, `payload` (bytea cifrado; apagado em `accepted`), `accepted_at`. Trigger: `accepted` imutável; `payload` só pode ir a nulo.

### 6.4 Envio (`Ledi::DeliverJob`, por cidade)
- Recorrente (a cada minuto) e disparado ao enfileirar; só roda com `usable?(:ledi_export)`.
- Lote `pending` com `next_attempt_at <= now`, `FOR UPDATE SKIP LOCKED`; login com cookie em cache de processo por cidade.
- 200 → `accepted` (apaga `payload`); 400 → `rejected` (mensagem); 5xx/timeout → nova tentativa com espera crescente até 24 h, depois `failed`; 401 → novo login uma vez; segunda 401 → credencial `unauthorized`, pausa o envio da cidade, aviso em Integrações.
- Eventos `ledi.ficha_accepted` / `ledi.ficha_rejected` só com ids.

### 6.5 Painel "Produção e-SUS"
- Dashboard (`municipal_admin`): por competência — aceitas, recusadas (motivos agrupados), pendentes, falhas; prazo estimado (10º dia útil do mês seguinte, feriados nacionais fixos e móveis, fuso da cidade).
- Alertas: 5 dias úteis antes com pendente/recusada; forte com `record_mode=record` e zero aceitas a 3 dias úteis.
- Console: tabela de todas as cidades com o mesmo resumo.
- "Reenviar" ficha recusada: regra definida pela prova técnica.

## 7. CADSUS

- `Cadsus::Client.for(city)` → backend `soap_pdq` (PDQv3, credencial `cadsus`) ou `simulated` (casos fixos; dev/test).
- `Cadsus::Lookup.call(cpf:, by:, citizen:)` → `{ cns:, birth_date:, sex: }` só em memória; grava `citizens.cns` (cifrado) e `citizens.cadsus_checked_at`; trilha `citizen.cadsus_looked_up` (`citizen_id`, `user_id`).
- Validação presencial: botão "Consultar CADSUS" com `usable?(:cadsus_lookup)`; mostra divergência de nascimento/sexo para o atendente decidir; indisponível → segue pelo documento.

## 8. LGPD e segurança

- Credenciais e `payload` cifrados com a chave da cidade; nunca em resposta, log, evento ou argumento de job; `filter_parameters` com `password`, `secret`, `cpf`, `cns`.
- Retratos do CNES: só cidades ativas, 13 competências.
- CADSUS: só CNS e data da conferência.
- Eventos só com ids.

## 9. Testes

- **Prova técnica:** `spec/integration/ledi_pec_spec.rb` com tag `:pec` (fora da CI; roda contra o PEC local).
- **Contrato LEDI:** serialização da ficha sintética e do transporte conferida contra os IDLs 8.7.0.
- **Terminologias e CNES:** fixtures reduzidas dos arquivos oficiais; importação ok, falha sem efeito, ativação no fim, retenção.
- **Envio:** tabela de casos (200, 400, 5xx, timeout, 401 + relogin, 401 duplo pausa), `SKIP LOCKED` com duas threads, apagamento do `payload`.
- **Invariantes** (`spec/invariants/record_mode_invariants_spec.rb`): os do ADR 0028.
- **Interruptores:** mutation (só mantenedor, auditada), sessão com `features`, 403 nas rotas, job sem efeito.
- **Front:** dashboard (Integrações, CNES, Produção, CADSUS no balcão), admin (modo/IBGE/PEC, visão geral, alerta SIGTAP), maintenance (interruptores).
- **Prova no navegador:** cidade de dev com PEC local: ligar interruptor no maintenance, cadastrar credencial no dashboard, testar conexão, enfileirar ficha sintética, ver aceita no painel.

## 10. Semente de dev

Curitiba e Maringá com `ibge_code` reais (4106902, 4115200), `record_mode=off`; retrato CNES fictício coerente com as unidades e profissionais da semente; release SIGTAP reduzida ativa; credencial `cadsus` simulada.

## 11. Rollout

1. `contracts` `session-v1.1.0` (push com autorização).
2. `api`: migração de plataforma e de cidade; tudo nasce `off` e desligado; `VALID_RANGE` → `(1..28)`.
3. `maintenance`, `admin`, `dashboard` depois do `api`.
4. Por cidade (quando houver): operador preenche IBGE, modo e PEC; importa SIGTAP e CNES; cidade cadastra credencial e confirma o CNES; mantenedor liga o interruptor.

## 12. Planos

`contracts`, `api` em dois planos (**fundação**: interruptores, modo, credenciais, terminologias, CNES, CADSUS; **exportador**: prova técnica, layout, fila, envio, painel), `dashboard`, `admin`, `maintenance`.

## 13. F-IDs (propostos)

| ID | Funcionalidade | Superfície |
|---|---|---|
| F-16.1 | Interruptores de funcionalidade por cidade, escritos pelo maintenance | api, maintenance, contracts, dashboard, admin |
| F-16.2 | Modo de prontuário, código IBGE e endereço do PEC por cidade | api, admin |
| F-16.3 | Credenciais de integração por cidade, cifradas, com teste de conexão | api, dashboard |
| F-16.4 | Terminologias CID-10, CIAP-2 e SIGTAP versionadas na plataforma | api |
| F-16.5 | Importação do CNES, equipes e casamento confirmado pela cidade | api, dashboard |
| F-16.6 | Exportador LEDI: prova técnica, layout versionado, fila e envio contínuo | api |
| F-16.7 | Painel da competência e alertas de prazo | api, dashboard, admin |
| F-16.8 | Consulta ao CADSUS na validação presencial | api, dashboard |

## 14. Riscos

- **Serialização Thrift em Ruby** sem exemplo oficial — mitigado pela prova técnica primeiro.
- **Esteira do LEDI** (versão a cada 2–6 semanas; > 12 meses invalidado) — manutenção contínua.
- **Dependência do PEC da cidade** como porta: sem HTTPS ou desatualizado, nada chega ao SIAPS.
- **Base do CNES grande** — importação filtrada por IBGE; tempo e disco a medir na prova.
- **`maintenance` só em dev/staging** — em produção tudo desligado até o ADR próprio.
- **Repasse da cidade** passa a depender da qualidade das fichas no modo `record`.
