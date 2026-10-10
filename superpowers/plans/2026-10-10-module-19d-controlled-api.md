# Módulo 19 (19d) — Receita de controlado e antimicrobiano com SNCR (api) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **PRÉ-REQUISITO: 19c entregue em `origin/main` do api** (`ClinicalDocuments::{Issue,Cancel,Pdf,Canonical,Verification,Page,Json}`, `ClinicalDocuments::Content::Prescription`, `Medications::{Substances,AnvisaImport}`, `Patients::ApplyMedicationEvent`, migrações `20261009600001`), a tag **`clinical-v1.1.0`** e a tag **`clinical-v1.2.0`** publicadas no `contracts` (esta última é a do 19d: esquema com `category`, `sncr`, `patient_identification`, `prescriber_contact` e o vetor JCS de uma RCE e uma RET). Quando este plano foi escrito (2026-10-10) o 19c estava em execução noutra sessão e a branch `feat/mod-19c-documents` tinha só o interruptor; o plano usa os nomes dos blocos **Produces** do plano do api do 19c (`docs/superpowers/plans/2026-10-09-module-19c-documents-api.md`) e o código real de 19a/19b em `origin/main` (`c9ccd88`+). **A Task 0 é a prova técnica do token do SNCR e a conferência da base: se a prova falhar, ou se um nome do 19c tiver mudado, PARE e reporte.**

**Goal:** O lado api do 19d (F-19.24 a F-19.29, ADR 0034): interruptores `controlled_prescriptions` e `sncr_mock`; credenciais e cliente do SNCR (login gov.br do prescritor pelo serviço da Anvisa, troca do `session_id` pelo token **no servidor** e pedido do bloco de 1.000 números dentro dos 30 s); SNCR simulado (`fake-sncr` no compose de dev, porta 8092, e o mesmo app nas specs); estoque de números por prescritor com consumo por lock; categoria da receita do 19c (`common` | `special_control` | `antimicrobial`) com as regras da Portaria 344 e da RDC 471; lista da Portaria 344 e marca de anticonvulsivante no catálogo; contato do prescritor; identificação do paciente; registro da Notificação de papel; PDF nos leiautes da Anvisa (RCE e RET) com as 2 vias, rodapé Anvisa e faixa de simulado; JSON canônico contra `clinical-v1.2.0`; página pública; painel do admin; `sncrStatus` no maintenance; eventos; invariantes; semente; prova e rollout.

**Architecture:** O SNCR não emprega credencial de plataforma nem PKCE nosso: o navegador vai a `<auth>/auth/login?client_url=…`, a Anvisa faz o OAuth com o gov.br e volta ao `client_url` com `?session_id` (uso único, 30 s); o dashboard manda `{ state, session_id }` ao api, que troca o `session_id` pelo `access_token` (`GET <auth>/auth/token`, cabeçalho `Origin` do host da cidade) e pede o bloco (`POST <base>/numeracoes/receita-especial-retencao`) na mesma requisição — o token vive numa variável local e nunca é gravado. O `state` de uso único reaproveita o mecanismo do 19b (`Signatures::OauthStates` + `signature_oauth_states`, com os propósitos `sncr_rce`/`sncr_ret`). Cidade (migração `20261010700001`): `sncr_number_batches` (só acréscimo) e `sncr_numbers` (`free → used → voided` por trigger, único por `(kind, number)`), `professionals.prescriber_address` (cifrado), o tipo `controlled_notification_record` em `clinical_documents`. Plataforma (migração `20261010700001`): `medication_catalog_items.anticonvulsant`, o tipo `anticonvulsant` nas listas Anvisa e `sncr_checks`. A emissão do 19c ganha a categoria, as regras e o modo com motivo (`paper_reason`); o número é tirado com `FOR UPDATE SKIP LOCKED` dentro da transação da emissão (a falha devolve por rollback) e fica `voided` no cancelamento e na volta ao papel.

**Tech Stack:** Rails 8.1 (API), PostgreSQL 16 (banco por cidade + plataforma), RSpec, WebMock (`to_rack`), Rack/Puma (fake-sncr), Prawn + `rqrcode_core` (do 19c), `json_schemer` e `Signatures::Jcs` (do 19b). Nenhuma gem nova.

**Spec:** `docs/superpowers/specs/2026-10-10-module-19d-controlled-prescriptions-design.md` e `docs/adr/0034.md` (leia os dois antes). Contrato entre apps — **fonte única de formatos**: `docs/superpowers/plans/2026-10-10-module-19d-controlled-contracts.md` (estende o do 19c, `2026-10-09-module-19c-documents-contracts.md`, cuja §12 prevalece). Pesquisa: `docs/pesquisa/2026-10-10-receita-controlada-19d.md`; Manual da API SNCR 3ª ed. (set/2026) e Instruções de Integração v1.0 (jul/2026), lidos ao escrever este plano (Desvio 1). Contratos do 19a (§9) e do 19b (§13) prevalecem sobre os planos antigos.

## Desvios e precisões

1. **O que os documentos oficiais dizem (lidos em 2026-10-10, Manual da API 3ª ed. §2.2 e Instruções v1.0 §4):** não há `client_id`/`client_secret` nem credenciamento de plataforma ("não solicita chaves fixas ou credenciais de parceiro"); a troca do `code` do gov.br pelo token é feita **pelo backend da Anvisa** (Keycloak, `kc_idp_hint=govbr`), que devolve ao `client_url` só `?session_id={token}` (uso único, 30 s); `GET /auth/token?session_id=` → `{ "access_token", "token_type": "Bearer" }` ou `{ "error": "Sessão inválida ou expirada" }`; o `access_token` vale 30 s, sem refresh; `client_url` só em domínio `.br` (em dev, `localhost` e `127.0.0.1` em qualquer porta); "O `session_id` fica vinculado ao `Origin` que iniciou a autenticação, sendo exigido o mesmo `Origin` para a obtenção do access_token"; o `state` do cliente é opcional e entra assinado no `state` da Anvisa — **a volta ao `client_url` não o menciona**. Os exemplos oficiais trocam o token e chamam a API **pelo navegador**; nenhum documento proíbe ou confirma a chamada pelo servidor. Por isso a Task 0 prova na homologação; os pontos não confirmados viram constantes do cliente (Task 7) ajustadas pelo resultado dela.
2. **Consequência para o contrato:** sem PKCE nosso e sem `code`: o callback é `{ state, session_id }` (ou `{ state, error }`), e como a Anvisa pode não devolver o nosso `state`, `POST /sncr/requests` devolve também o `state`, que o dashboard guarda em `sessionStorage` antes de sair e reenvia no callback (o api amarra o `state` ao usuário, à cidade e ao uso único — o mecanismo do 19b). As credenciais são `sncr.{base_url, auth_url, maintainer_cnpj}` (sem `client_id`/`client_secret`, que não existem). Ver "Divergências".
3. **Endpoint e campos reais (Manual §2.3.2):** `POST numeracoes/receita-especial-retencao` com `{ "conselho": "CRM"|"CRMV"|"CRO", "tipo": "RCE"|"RET", "documento": "<inscrição>", "uf": "<UF do conselho>", "cnpj": "<CNPJ da mantenedora>" }` → **201** `{ "inicio": "2602.6-53.0000001", "fim": "2602.6-53.0001000", "quantidade": 1000, "mensagem": "Numeração gerada com sucesso." }`. Erros 400 com mensagens fixas (o limite mensal é "Usuário atingiu o limite máximo de receita para o tipo de receita no mês atual."), 404 "Inscrição fornecida é diferente da autenticada." / "O usuário informado não possui vínculo ativo no conselho …" / "Prescritor não encontrado com o CPF fornecido.", 401 não autorizado. O esgotamento da faixa (informe de 09/10) **não tem mensagem documentada**: o cliente o reconhece por `/esgot/i` e o falso usa "Faixa de numeração esgotada para a UF." (pendência de go-live: confirmar a mensagem real). O pedido do SNCR é por **inscrição no conselho**, não por CPF: o `409 professional_cpf_missing` do contrato não se aplica; sem perfil de profissional ou com conselho fora de CRM/CRO → 403 `cbo_not_allowed`.
4. **Os 3 pedidos por mês são por inscrição em TODAS as plataformas.** O api conta os seus (`sncr_number_batches` do mês, fuso da cidade) e recusa o 4º antes de mandar ao gov.br (409 `monthly_limit_reached`); quando o SNCR recusa antes (pedidos feitos por outra plataforma), a mesma resposta.
5. **Número repetido vindo do SNCR** (incidente de 09/10: números de outro prescritor): `sncr_numbers` é único por `(kind, number)`; o lote grava `ON CONFLICT DO NOTHING` e `received` conta só os novos; a contagem dos repetidos vai ao log (só o número inteiro, nunca o número SNCR). O lote registra o que o SNCR disse (`first_number`, `last_number`, `quantity`).
6. **Simulado nunca se mistura com real:** cada número guarda `simulated`; a emissão só tira número do modo corrente da cidade (`Sncr::Mock.on?`), e em produção nunca tira simulado (o interruptor nem existe lá). Desligar o `sncr_mock` deixa os simulados parados no estoque, sem uso.
7. **Categoria e conteúdo:** com `controlled_prescriptions` utilizável a receita é classificada pelos itens (listas da Portaria 344 do 19c e marca de antimicrobiano); desligado, o comportamento do 19c fica igual (controlado → `controlled_not_allowed`; antimicrobiano → papel), e a categoria é `common` ou `antimicrobial`. O `content` gravado e devolvido traz sempre `category`, e `sncr`, `patient_identification`, `prescriber_contact` (null quando não se aplicam). Controlado em **texto livre** continua recusado (`controlled_not_allowed`): só item do catálogo tem lista confiável. A lista C4 (antirretrovirais) é tratada como C2/C3 → `not_supported` (o contrato §1 não a lista).
8. **JSON canônico:** na receita `common` o canônico **omite** `category` (ausente = `common`) e omite `sncr`, `patient_identification`, `prescriber_contact` nulos — os vetores da `clinical-v1.1.0` seguem byte a byte (Divergência D7). Na RCE/RET entram os quatro. `controlled_notification_record` não é assinável.
9. **`prescriber_contact`:** obrigatório em `special_control` (digital ou papel; Lei 5.991 art. 35); em `antimicrobial` só quando o modo sai `digital` (decidido dentro da transação; falta → 422 `prescriber_address_missing` e rollback, o número volta a `free`). Enfermeiro nunca precisa (antimicrobiano dele é sempre papel).
10. **Modo e motivo (`paper_reason`):** o modo continua decidido na emissão e nunca convertido (só a volta ao papel do 19b). Ordem do motivo: `feature_disabled` (antimicrobiano com o interruptor desligado) → `nurse_antimicrobial` → `signature_unavailable` (`digital_signature` não utilizável) → `no_certificate` → `no_sncr_number` (sem número livre no modo corrente, visto com o lock). Vale para qualquer documento da consulta em papel (atestado sem certificado sai com `no_certificate`); a declaração da recepção e o registro da Notificação saem com `paper_reason: null`. O motivo é **gravado** na coluna nova `clinical_documents.paper_reason` (imutável como o resto do documento) e devolvido em toda leitura do `<document>`, não só no `POST`. A volta ao papel do 19b (digital → paper) não grava motivo: o bloco `signature` do 19b já diz por quê (`returned_to_paper` + `reason_code`).
11. **Volta ao papel e cancelamento anulam o número:** `Signatures::ToPaper` de uma RCE/RET digital deixa o número `voided` (nunca foi impresso nem assinado, mas o `content` imutável o guarda; reusar violaria "nunca duas vezes") e o PDF de papel não imprime número; o cancelamento faz o mesmo (`sncr.number_voided`).
12. **Registro da Notificação:** `controlled_notification_record` nasce sempre `paper`, sem pedido de assinatura e sem canônico; as colunas `verification_token`/`short_code` do 19c continuam preenchidas (NOT NULL), mas a API devolve `short_code: null`, `verification_url: null`, a página pública responde 404 para ele e o impresso responde 404 `not_found`. O item vai à lista de medicamentos com `continuous: false`.
13. **Dentista:** a consulta do 19a ainda recusa CBO `2232` (Desvio 9 do 19c); a regra "médico e dentista" é provada na matriz (`ClinicalDocuments::Content::Controlled`) e o dentista da semente tem contato e estoque, mas não emite em dev.
14. **Título da RCE:** "RECEITA DE CONTROLE ESPECIAL" (P&R RDC 1.000, itens 7 e 67: o leiaute próprio tem de manter o título "Receita de Controle Especial"); a spec §6 diz "Receituário" — vale a norma.
15. **Contato do prescritor:** o telefone é o `professionals.phone` que já existe (cifrado, editável pelo profissional desde o módulo 10); o endereço é a coluna nova `professionals.prescriber_address` (JSON cifrado). A rota do contrato §5 usa `PUT`.

## Valores fixados por este plano

1. **Número SNCR:** `/\A\d{4}\.\d-\d{2}\.\d{7}\z/` (ex.: `2602.6-53.0000001`); o bloco é `inicio..fim` no mesmo prefixo, com `quantidade` (1 a 1.000) números.
2. **Endereço** (`patient_identification.address` e `prescriber_contact.address`): `street` 1–120, `number` 1–10, `complement` opcional ≤ 60, `district` 1–80, `city` 1–80, `uf` `^[A-Z]{2}$` e em `Professional::UFS`, `zip` `^[0-9]{8}$` — **os formatos do esquema da `clinical-v1.2.0`, sem máscara: CEP com hífen, UF minúscula ou telefone com pontuação são recusados com o campo** (uma representação por valor; o complemento só com espaço vira ausente). **Telefone:** `^[0-9]{10,11}$`. **CPF da identificação do paciente:** sempre o do cadastro do paciente, preenchido pelo api (a entrada não envia CPF; sem `no_cpf` nem passaporte no 19d). **Número SNCR:** `^\d{4}\.\d-\d{2}\.\d{7}$`. **`paper_number`:** 1–30 `[0-9A-Za-z./-]`.
3. **Limites:** `special_control` — até 3 itens C1 (`too_many_c1_substances`), `duration_days` obrigatório e ≤ 60 por item (≤ 180 se o item é anticonvulsivante), validade 30 dias, 2 vias; `antimicrobial` — validade 10 dias, 2 vias; Notificação — `A` ≤ 30 dias, `B`/`B2` ≤ 60 dias, uma substância.
4. **Listas da Portaria 344 por categoria:** `C1`, `C5` → `special_control`; `A1`, `A2`, `A3`, `B1`, `B2` → `requires_notification` (na receita) e aceitas no registro (A1/A2/A3 ↔ `A`, B1 ↔ `B`, B2 ↔ `B2`); `C2`, `C3`, `C4` → `not_supported`.
5. **Estoque:** `low_threshold` 50 por tipo; 3 pedidos por tipo por mês civil do fuso da cidade.
6. **SNCR simulado:** `FAKE_SNCR_URL=http://fake-sncr:8092/api/v1` (api/worker), `FAKE_SNCR_PUBLIC_URL=http://localhost:8092/api/v1` (navegador); CNPJ simulado `11222333000181`; números `"<AAMM>.<1 RCE|2 RET>-<código IBGE da UF>.<7 dígitos>"`; `client_url` aceito em `*.br`, `localhost`, `*.localhost` e `127.0.0.1`.
7. **Retorno do dashboard:** `client_url = "#{CityPublicUrl.dashboard(city)}sncr/callback"`; `Origin` do servidor = `CityPublicUrl.base(city)`.

## Global Constraints

- Banco de cidade: `db/city_migrate/20261010700001_add_controlled_prescriptions.rb` (acima de `20261009600001`), dump à mão em `db/city_schema.rb` (`define(version: 2026_10_10_700001)`; paridade em `spec/services/city_schema_spec.rb`), triggers em `db/city_triggers.sql` (blocos novos no fim, guardados por `to_regclass`; a migração termina com `execute File.read(Rails.root.join("db/city_triggers.sql"))`). Irreversível. Rollout: publicar a imagem e rodar `city:migrate:all` dela **antes** de cortar tráfego; **nunca** migrar cidade fora do rake.
- Banco de plataforma: `db/platform_migrate/20261010700001_add_controlled_prescriptions_platform.rb`, `db/platform_schema.rb` (`define(version: 2026_10_10_700001)`), `db/platform_triggers.sql` se tocar trigger. **A migração de plataforma mexe no banco de teste de plataforma COMPARTILHADO**: avise as sessões "API", "MVP Module 19" e as do 19d antes de rodá-la em teste, e no balanço.
- Valores, exatamente: `category` `common` | `special_control` | `antimicrobial`; `sncr.kind` `rce` | `ret`; número `free` | `used` | `voided`; `notification_type` `A` | `B` | `B2`; `controlled_list` `A1`…`C5`; `paper_reason` `no_certificate` | `signature_unavailable` | `no_sncr_number` | `nurse_antimicrobial` | `feature_disabled`; `kind` novo `controlled_notification_record`; propósitos do state `sncr_rce` | `sncr_ret`, provedor `sncr` | `sncr_simulated`.
- Erros do contrato, exatamente: 422 `mixed_categories`, `requires_notification` (`index`), `not_supported` (`index`), `too_many_c1_substances`, `duration_exceeded` (`index` na receita), `patient_identification_required` (`field`), `prescriber_address_missing`, `item_not_in_notification_list`, `notification_type_mismatch`, `invalid_content` (`field`), `invalid_state`, `invalid_address` (`field`), `invalid_phone`; 403 `cbo_not_allowed`, `authorization_denied`, `feature_disabled` (`{ "error": "feature_disabled", "feature": "controlled_prescriptions" }`), `missing_role`; 409 `monthly_limit_reached`, `authorization_expired`; 503 `sncr_unavailable`, `sncr_exhausted`. Novo (Divergências): 403 `registration_mismatch`. Escrita por cookie exige `Content-Type: application/json`, inclusive PUT e DELETE (guarda existente).
- Cifra por cidade: `encrypts :prescriber_address` (não determinístico) entra em `CityEncryption::CITY_KEYED_TARGETS` (`spec/architecture/city_encrypted_attributes_guard_spec.rb`). `signature_oauth_states.code_verifier` já é cifrado (o SNCR não usa o verifier; a linha o recebe aleatório e ele nunca sai).
- **Token gov.br/SNCR nunca persistido** (nem coluna, nem log, nem evento, nem `inspect`); `session_id`, número SNCR, CPF, passaporte, endereço e telefone fora de log (`filter_parameters`), evento, Analytics, mensagem de erro e `inspect`.
- Eventos só com ids (contrato §10), em `config/initializers/domain_events.rb` (`to: []`) e `spec/initializers/domain_events_bindings_spec.rb`: `sncr.numbers_requested { user_id, kind, count, simulated }`, `sncr.number_used { document_id, kind }`, `sncr.number_voided { document_id, kind }`, `controlled_notification.recorded { document_id, notification_type }`, e `clinical_document.issued`/`.cancelled` ganham `category` (só na receita). Nenhum `Platform.audit` novo. `DomainEvents.publish(` é multilinha: confira cada chamada inteira.
- `Current.city` **nunca** é atribuído em `app/` ou `lib/`; nenhum job novo. Leitura de plataforma (interruptores, catálogo, credenciais) **fora** da transação da cidade.
- Specs de request com `type: :request` explícito; arquivo novo em `spec/support/` entra com `require_relative` em `spec/rails_helper.rb`. `spec/adr_pointers_spec.rb`: `VALID_RANGE = (1..34)`.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). `git add` com caminhos explícitos (nunca `-A`). Merge, push, tag e cards só com autorização explícita do usuário. Senha gov.br, CPF real e token nunca no chat nem em arquivo.

## Ambiente de execução

- Antes de tocar `apps/api`, avise a sessão dona do api (sessão "API"), a sessão "MVP Module 19" (19c) e as sessões do dashboard e do maintenance do 19d. Ordem de merge: `contracts` (tag `clinical-v1.2.0`) → **api** → dashboard → maintenance.
- Worktree (raiz do monorepo `/Users/eduardovrocha/Development/ioit.solutions/rota-saude`):

  ```bash
  /opt/homebrew/bin/git -C apps/api fetch origin
  /opt/homebrew/bin/git -C apps/api worktree add .claude/mod19d -b feat/mod-19d-controlled origin/main
  cp apps/api/config/master.key apps/api/.claude/mod19d/config/master.key
  ```

- `./apps/api` é montado em `/rails` no container `api`; o worktree é `/rails/.claude/mod19d`. **Convenção dos steps:** `rspec <arquivos>` abrevia

  ```bash
  docker compose exec -T -e ROTA_TEST_DB_SUFFIX=_mod19d -w /rails/.claude/mod19d api bundle exec rspec <arquivos>
  ```

  e `git <...>` abrevia `/opt/homebrew/bin/git -C apps/api/.claude/mod19d <...>`.
- Bancos de teste de cidade da branch (sufixo `_mod19d`), criados no começo e recriados depois da migração de cidade (Task 4):

  ```bash
  psql -U rota_saude -d postgres -c "DROP DATABASE IF EXISTS rota_saude_test_city_a_mod19d" -c "DROP DATABASE IF EXISTS rota_saude_test_city_b_mod19d"
  docker compose exec -T -e RAILS_ENV=test -e ROTA_TEST_DB_SUFFIX=_mod19d -w /rails/.claude/mod19d api bin/rails city:test_databases
  ```

- Migração de plataforma em teste (Task 2; banco compartilhado — avise antes): `docker compose exec -T -e RAILS_ENV=test -w /rails/.claude/mod19d api bin/rails db:migrate:platform`.
- Suíte completa só com o worker parado e sem outra sessão rodando suíte: `docker compose stop worker`, a suíte, `docker compose start worker`.
- `docker-compose.yml` mora na raiz do monorepo, **fora de git**: a Task 6 o edita no lugar (sem commit) e o balanço avisa.
- Prova no navegador: api do worktree na porta **3039** (Task 22), com `CITY_PUBLIC_BASE_TEMPLATE=http://%{slug}.localhost:5189` (Vite do dashboard do 19d) e o `fake-sncr` na 8092. O prefixo `/sncr` precisa de entrada no proxy de dev do dashboard (plano do dashboard).

### Arquivos em comum com o 19c (e como rebasear)

Se a main receber correções, rebase sobre `origin/main` e resolva **somando** os dois lados em: `db/city_schema.rb`, `db/platform_schema.rb` (`define(version:)` com o maior número), `db/city_triggers.sql`, `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`, `config/initializers/filter_parameter_logging.rb`, `app/services/city_encryption.rb`, `spec/adr_pointers_spec.rb` (fica `1..34`), `config/routes.rb`, `app/services/platform/features.rb`, `app/services/medications/{substances,anvisa_import}.rb`, `app/services/clinical_documents/{json,pdf,canonical,verification,page}.rb`, `app/services/clinical_documents/content/prescription.rb`, `app/commands/clinical_documents/{issue,cancel}.rb`, `app/commands/signatures/to_paper.rb`, `app/graphql/maintenance/**`, `spec/architecture/maintenance_schema_spec.rb`, as listas fixas do catálogo de interruptores nas specs.

## Review Focus

1. **O callback do gov.br chega tarde, duas vezes ou com o `state` de outra pessoa** (o médico demora no gov.br, recarrega a página de retorno, abre duas abas "Obter números"): o segundo uso do mesmo `state` → 422 `invalid_state`; `session_id` vencido → 409 `authorization_expired` com o `return_to`; nenhum lote parcial gravado, nenhum 500, nenhum token no banco ou no log. Testes: Task 9 ("callback repetido, vencido, de outro usuário").
2. **O SNCR devolve números que já estão no estoque** (o incidente de 09/10, ou o falso reiniciado): nenhum número duplicado, o lote grava só os novos e `received` diz quantos, sem 500. Testes: Task 8 ("bloco com números repetidos").
3. **Duas abas emitindo RCE ao mesmo tempo com 1 número livre** (ou o médico clicando duas vezes em "Emitir"): uma sai `digital` com o número, a outra sai `paper` com `no_sncr_number`; nunca o mesmo número em dois documentos; o rollback de uma emissão que falha devolve o número. Testes: Task 8 (corrida com threads) e Task 12 ("falha depois de tirar o número").
4. **A cidade desliga o `sncr_mock` (ou o ambiente é produção) com números simulados no estoque:** eles nunca são consumidos no modo real, o saldo mostrado é o do modo corrente, e documento com número simulado sempre sai com a faixa "NUMERAÇÃO SIMULADA — SEM VALIDADE". Testes: Task 8 ("modo corrente") e Task 20 (invariantes).
5. **Endereço digitado por gente e paciente sem CPF no cadastro:** CEP com hífen, UF minúscula, telefone com parênteses (recusados com o `field` exato — o formato é o do esquema), complemento só com espaço (vira ausente), acento e emoji no logradouro, CPF mandado na entrada (ignorado: vale o do cadastro), cadastro sem CPF numa RCE (422 com o campo), 3 C1 + 1 C5 de 60 dias — o PDF sai (com `?` no que a fonte não tem) e nada vai ao log. Testes: Task 11 ("endereço digitado") e Task 15 ("RCE cheia").

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| `docs/pesquisa/2026-10-10-sncr-token-no-servidor.md` (repo docs) | prova técnica do token no servidor | 0 |
| `app/services/platform/features.rb`, `app/services/controlled_prescriptions/gate.rb`, `app/controllers/concerns/controlled_prescriptions_gate.rb`, `app/services/sncr/mock.rb`, `spec/adr_pointers_spec.rb`, listas fixas | interruptores | 1 |
| `db/platform_migrate/20261010700001_add_controlled_prescriptions_platform.rb`, `db/platform_schema.rb`, `app/models/sncr_check.rb` | plataforma | 2 |
| `config/medications/anvisa/{anticonvulsants.csv,SOURCES.yml}`, `app/services/medications/{substances,anvisa_import}.rb`, `spec/fixtures/medications/anvisa/{anticonvulsants.csv,SOURCES.yml}` | lista por item e anticonvulsivante | 3 |
| `db/city_migrate/20261010700001_add_controlled_prescriptions.rb`, `db/city_schema.rb`, `db/city_triggers.sql`, `app/models/{sncr_number_batch,sncr_number}.rb`, `app/models/{professional,clinical_document,signature_oauth_state}.rb`, `app/services/city_encryption.rb`, `config/initializers/{domain_events,filter_parameter_logging}.rb`, `spec/support/controlled_prescription_helpers.rb` | dados da cidade | 4 |
| `app/services/sncr/{config,checks}.rb` | credenciais e checagem | 5 |
| `lib/fake_sncr/{app,config.ru}`, `docker-compose.yml` (raiz, fora de git) | SNCR simulado | 6 |
| `app/services/sncr/{errors,numbers,client}.rb` | cliente do SNCR | 7 |
| `app/services/sncr/stock.rb` | estoque com lock | 8 |
| `app/commands/sncr/{start_request,complete_request}.rb`, `app/controllers/sncr/{base,stock,requests,oauth}_controller.rb`, `config/routes.rb` | pedido de números | 9 |
| `app/services/clinical_documents/address.rb`, `app/controllers/professional_contacts_controller.rb` | contato do prescritor | 10 |
| `app/services/clinical_documents/content/{controlled,patient_identification}.rb`, `app/services/clinical_documents/content/prescription.rb` | categoria e regras | 11 |
| `app/commands/clinical_documents/issue.rb`, `app/controllers/clinical_documents_controller.rb`, `app/services/clinical_documents/json.rb` | emissão com número e motivo; última identificação | 12 |
| `app/services/clinical_documents/content/notification_record.rb`, `issue.rb`, `json.rb`, `verification.rb`, controller | registro da Notificação | 13 |
| `app/commands/clinical_documents/cancel.rb`, `app/commands/signatures/to_paper.rb` | número anulado | 14 |
| `app/services/clinical_documents/pdf.rb` | leiautes Anvisa RCE/RET | 15 |
| `config/clinical/clinical-document-v1.json`, `spec/fixtures/clinical/**`, `app/services/clinical_documents/canonical.rb` | canônico `clinical-v1.2.0` | 16 |
| `app/services/clinical_documents/{verification,page}.rb` | página pública | 17 |
| `app/services/sncr/admin_overview.rb`, `app/controllers/sncr/admin_controller.rb` | painel do admin | 18 |
| `app/graphql/maintenance/types/{sncr_status_type,query_type}.rb`, `app/graphql/maintenance/analyzers/human_only.rb` | `sncrStatus` | 19 |
| `spec/invariants/controlled_prescriptions_invariants_spec.rb` | invariantes do ADR 0034 | 20 |
| `lib/controlled_prescriptions_crew.rb`, `db/seeds.rb` | semente de dev | 21 |
| — | revisão, suíte, porta 3039, prova, rollout | 22 |

---

## Fatia 0 — Prova técnica e base

### Task 0: Prova técnica do token do SNCR no servidor e conferência da base

Esta task não escreve código do api. **Se o Step 4 não confirmar a troca no servidor dentro de 30 s, PARE e reporte ao coordenador e ao usuário** com as opções: (a) seguir com o SNCR simulado e manter a troca no servidor como gate de go-live (risco: refazer o cliente se o plano B vier); (b) plano B do ADR 0034 (o navegador chama o SNCR e entrega os números ao api). Nada das Tasks 1–22 começa sem a decisão.

**Files:**
- Create: `docs/pesquisa/2026-10-10-sncr-token-no-servidor.md` (repo `docs`)
- Create (fora de git, descartável): `$SCRATCH/sncr_probe/config.ru`, onde `$SCRATCH` é o diretório de rascunho da sessão

- [ ] **Step 1: Registre a leitura dos documentos oficiais** no arquivo da pesquisa, com este conteúdo (já conferido ao escrever o plano; releia o Manual da API 3ª ed. §2.2–2.3 e as Instruções v1.0 §4 e corrija se a Anvisa publicou edição nova — a lista de arquivos está em `https://www.gov.br/anvisa/++api++/pt-br/assuntos/medicamentos/controlados/sncr/documentos-do-sncr`):

```markdown
# Prova técnica 19d — troca do token do SNCR no servidor

Data: <AAAA-MM-DD da execução>. Fontes: Manual da API SNCR 3ª ed. (set/2026), §2.2 e §2.3; Instruções de Integração v1.0 (jul/2026), §3 e §4.

## O que os documentos dizem [CONFIRMADO]
- Sem credencial de plataforma: "não solicita chaves fixas ou credenciais de parceiro" (Manual §2.2.2); credenciamento de plataforma "não será necessário" (Instruções §3).
- `GET /auth/login?client_url=…[&state=…]`: o backend da Anvisa gera nonce e state assinados (HMAC-SHA256, 5 min) e redireciona ao Keycloak com `kc_idp_hint=govbr`; o callback dele troca o `code` pelo token e redireciona ao `client_url` com `?session_id={token}` (uso único, 30 s).
- `GET /auth/token?session_id=` → 200 `{ "access_token", "token_type": "Bearer" }`; erro `{ "error": "Sessão inválida ou expirada" }`; o token sai da sessão após a leitura.
- Access token de 30 s, sem refresh token.
- `client_url` só em domínio `.br`; em dev, `localhost` e `127.0.0.1` em qualquer porta.
- "O session_id fica vinculado ao Origin que iniciou a autenticação, sendo exigido o mesmo Origin para a obtenção do access_token."
- O `state` do cliente é opcional; a volta ao `client_url` só menciona `session_id`.
- Os exemplos oficiais chamam `/auth/token` e a API pelo navegador (CORS liberado para `*.br`). Nenhum trecho proíbe ou confirma a chamada pelo servidor.
- Hosts citados: API em `https://api-gateway.{hmg,prd}.apps.anvisa.gov.br/api-sncr/api/v1/`; auth nos exemplos em `https://sncr-api.apps.anvisa.gov.br/api/v1/auth/...`.

## O que falta provar na homologação
1. O servidor consegue trocar o `session_id` (com o cabeçalho `Origin` igual ao do `client_url`)?
2. O pedido de bloco com o token, feito pelo servidor, responde 201 dentro dos 30 s?
3. O `state` volta no `client_url`?
4. `*.localhost` (ex.: `curitiba.localhost`) é aceito como `client_url` em dev?
5. Qual host serve `/auth/*` na homologação (o do gateway ou `sncr-api`)?
```

- [ ] **Step 2: Monte a sonda** (descartável, fora de git; nunca imprime token, `session_id` ou número):

```ruby
# $SCRATCH/sncr_probe/config.ru
# Sonda da prova técnica do 19d. Variáveis: SNCR_AUTH_URL (base do /auth),
# SNCR_BASE_URL (base de /numeracoes), SNCR_CONSELHO, SNCR_DOCUMENTO, SNCR_UF,
# SNCR_CNPJ. Mostra só status, tempos e NOMES de chaves.
require "net/http"
require "json"
require "uri"
require "cgi"

AUTH = ENV.fetch("SNCR_AUTH_URL").chomp("/")
BASE = ENV.fetch("SNCR_BASE_URL").chomp("/")
INSCRICAO = { "conselho" => ENV.fetch("SNCR_CONSELHO"), "documento" => ENV.fetch("SNCR_DOCUMENTO"), "uf" => ENV.fetch("SNCR_UF") }.freeze
CNPJ = ENV.fetch("SNCR_CNPJ")

def ms_since(start) = ((Process.clock_gettime(Process::CLOCK_MONOTONIC) - start) * 1000).round

def http(request, uri)
  Net::HTTP.start(uri.host, uri.port, use_ssl: uri.scheme == "https", open_timeout: 5, read_timeout: 10) { |h| h.request(request) }
end

run lambda { |env|
  req = Rack::Request.new(env)
  host = req.params["host"] == "curitiba" ? "http://curitiba.localhost:4300" : "http://localhost:4300"
  case req.path
  when "/start"
    query = URI.encode_www_form(client_url: "#{host}/cb#{req.params['no_origin'] ? '-no-origin' : ''}", state: "probe-state-1")
    [ 302, { "location" => "#{AUTH}/auth/login?#{query}" }, [] ]
  when "/cb", "/cb-no-origin"
    start = Process.clock_gettime(Process::CLOCK_MONOTONIC)
    report = { client_url_host: req.host, query_keys: req.params.keys.sort, state_returned: req.params["state"] == "probe-state-1" }
    token_uri = URI("#{AUTH}/auth/token?session_id=#{CGI.escape(req.params['session_id'].to_s)}")
    token_req = Net::HTTP::Get.new(token_uri)
    token_req["Accept"] = "application/json"
    token_req["Origin"] = "#{req.scheme}://#{req.host_with_port}" unless req.path == "/cb-no-origin"
    token_res = http(token_req, token_uri)
    report.merge!(token_status: token_res.code.to_i, token_ms: ms_since(start))
    token = (JSON.parse(token_res.body)["access_token"] rescue nil)
    report[:token_body_keys] = (JSON.parse(token_res.body).keys rescue [ "not_json" ])
    if token
      numbers_uri = URI("#{BASE}/numeracoes/receita-especial-retencao")
      numbers_req = Net::HTTP::Post.new(numbers_uri)
      numbers_req["Authorization"] = "Bearer #{token}"
      numbers_req["Content-Type"] = "application/json"
      numbers_req.body = JSON.generate(INSCRICAO.merge("tipo" => "RET", "cnpj" => CNPJ))
      numbers_res = http(numbers_req, numbers_uri)
      body = (JSON.parse(numbers_res.body) rescue {})
      report.merge!(numbers_status: numbers_res.code.to_i, total_ms: ms_since(start), numbers_body_keys: body.keys.sort,
                    quantidade: body["quantidade"], mensagem: body["mensagem"])
    end
    [ 200, { "content-type" => "application/json" }, [ JSON.pretty_generate(report) ] ]
  else
    [ 404, {}, [ "" ] ]
  end
}
```

- [ ] **Step 3: Rode a sonda contra a homologação, com o usuário.** Pergunte ao usuário se há acesso: conta gov.br de teste do portal de treinamento (`https://sncr-treinamento.anvisa.gov.br/`) ligada a uma inscrição ativa em CRM/CRO, e qual CNPJ usar (o da mantenedora). **O login gov.br é feito pelo usuário no navegador; você nunca digita senha nem pede CPF no chat.** Os dados da inscrição entram por variável de ambiente digitada pelo usuário no terminal dele. Comando:

```bash
docker compose run --rm -p 127.0.0.1:4300:4300 -v "$SCRATCH/sncr_probe:/probe" \
  -e SNCR_AUTH_URL -e SNCR_BASE_URL -e SNCR_CONSELHO -e SNCR_DOCUMENTO -e SNCR_UF -e SNCR_CNPJ \
  api bundle exec puma -b tcp://0.0.0.0:4300 /probe/config.ru
```
O usuário abre, uma de cada vez: `http://localhost:4300/start` (caso principal), `http://localhost:4300/start?no_origin=1` (sem `Origin`), `http://curitiba.localhost:4300/start?host=curitiba` (`*.localhost`). Cada volta mostra o relatório JSON. **Sem acesso à homologação → registre "não confirmado: sem acesso à homologação" e PARE** (cabeçalho desta task).

- [ ] **Step 4: Registre o resultado e decida.** Acrescente ao arquivo da pesquisa a seção `## Resultado na homologação` com os três relatórios (só status, tempos, nomes de chave, `quantidade` e `mensagem`) e as respostas às perguntas 1–5. **Confirmado** = caso principal com `token_status` 200 e `numbers_status` 201 e `total_ms` < 30000. Anote para a Task 7: o host de `/auth` (→ `sncr.auth_url`), se o `Origin` é exigido (→ `Sncr::Client::SEND_ORIGIN`), se o `state` volta (→ nota para o dashboard), se `*.localhost` vale. Não confirmado → PARE (cabeçalho).

- [ ] **Step 5: Commit no repo docs**

```bash
/opt/homebrew/bin/git -C docs add pesquisa/2026-10-10-sncr-token-no-servidor.md
/opt/homebrew/bin/git -C docs commit -m "docs: record the SNCR server-side token exchange proof

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

- [ ] **Step 6: Confira o 19c na main do api e os nomes que este plano usa**

```bash
/opt/homebrew/bin/git -C apps/api fetch origin
for f in app/commands/clinical_documents/issue.rb app/commands/clinical_documents/cancel.rb app/services/clinical_documents/content/prescription.rb \
         app/services/clinical_documents/pdf.rb app/services/clinical_documents/canonical.rb app/services/clinical_documents/verification.rb \
         app/services/clinical_documents/page.rb app/services/clinical_documents/json.rb app/services/medications/substances.rb \
         app/services/medications/anvisa_import.rb app/commands/patients/apply_medication_event.rb app/commands/signatures/to_paper.rb \
         app/services/signatures/oauth_states.rb spec/support/clinical_document_helpers.rb db/seeds/medications/catmat-dev.json \
         lib/clinical_documents_crew.rb; do
  /opt/homebrew/bin/git -C apps/api cat-file -e "origin/main:$f" && echo "ok $f" || echo "FALTA $f"
done
/opt/homebrew/bin/git -C apps/api grep -n "def call(input, role:, patient:, catalog:, matcher:" origin/main -- app/services/clinical_documents/content/prescription.rb
/opt/homebrew/bin/git -C apps/api grep -n "def issue(kind, input, by, consultation, attendance, replaces_id, signing, platform, now)" origin/main -- app/commands/clinical_documents/issue.rb
/opt/homebrew/bin/git -C apps/api grep -n "Flags = Data.define" origin/main -- app/services/medications/substances.rb
/opt/homebrew/bin/git -C apps/api grep -n "ck_anvisa_list_substances_kind\|ck_clinical_documents_kind\|ck_signature_oauth_states_purpose\|ck_signature_oauth_states_provider\|PROVIDERS =" origin/main -- db/
/opt/homebrew/bin/git -C apps/api grep -n "def documents_city!\|def import_test_catalog!\|def catalog_item(" origin/main -- spec/support
```
Expected: todos `ok`; as assinaturas de `Content::Prescription.call` e `Issue#issue` como acima; `Flags = Data.define(:antimicrobial, :controlled, :controlled_list)`; os nomes dos CHECKs e a lista `PROVIDERS` da migração do 19b (anote os valores: a Task 4 os repete somando os novos); os helpers. Nome diferente → **adapte** as Tasks 3, 11–14 ao nome real (mesmo comportamento) e anote no balanço; método ou arquivo ausente → **pare** e reporte.

- [ ] **Step 7: Confira as tags do `contracts` e o recorte do catálogo**

```bash
/opt/homebrew/bin/git -C contracts fetch --tags origin
/opt/homebrew/bin/git -C contracts tag -l 'clinical-v1.*'
/opt/homebrew/bin/git -C contracts ls-tree -r --name-only clinical-v1.2.0 clinical/
/opt/homebrew/bin/git -C contracts show clinical-v1.2.0:clinical/examples/canonical/SHA256SUMS
grep -o '"codigoItem":\(271089\|273009\|267197\|268856\)' apps/api/.claude/mod19d/db/seeds/medications/catmat-dev.json 2>/dev/null || \
  /opt/homebrew/bin/git -C apps/api show origin/main:db/seeds/medications/catmat-dev.json | grep -o '"codigoItem":\(271089\|273009\|267197\|268856\)'
```
Expected: `clinical-v1.1.0` e `clinical-v1.2.0`; no `SHA256SUMS` da 1.2.0 os `.jcs` da 1.1.0 **com os mesmos hashes** e dois novos — `prescription-special-control.jcs` e `prescription-ret.jcs` (`prescription-antimicrobial` já é exemplo publicado da 1.1.0), com os SHA-256 do plano `docs/superpowers/plans/2026-10-10-module-19d-controlled-contracts-repo.md`: `prescription-special-control.jcs` 1617 bytes `25538e7fb3c406a8a84a785069307297097b338c460d15ed2599cf7df2bb2e0a` e `prescription-ret.jcs` 1390 bytes `e4397d409d2e9bc5a625e6e8bce4ab0d1539e4f1d7f4c2256c7812a95f170464` (a Task 16 os fixa); a tag é MINOR da v1. Confira também no esquema: `patient_identification` = `{ cpf, address }` (sem `no_cpf` nem `passport`), `zip` `^[0-9]{8}$`, `uf` `^[A-Z]{2}$`, `phone` `^[0-9]{10,11}$`, número SNCR `^\d{4}\.\d-\d{2}\.\d{7}$`; diferença → **pare** e reporte. Leia `clinical/clinical-document-v1.json` da tag e compare com o contrato §2 e com o Desvio 8 (receita `common` sem `category` no canônico): se a tag exigir `category` na receita comum, **pare** e reporte (os vetores da 1.1.0 mudariam). Os quatro códigos do catálogo existem (amoxicilina 500 mg — antimicrobiano; fluoxetina — C1; diazepam — B1; losartana — comum). Sem a tag `clinical-v1.2.0`, **pare**.

- [ ] **Step 8: Crie o worktree e os bancos de teste da branch** (Ambiente de execução) e confira a base:

Run: `ls apps/api/.claude/mod19d/db/city_migrate | tail -1; ls apps/api/.claude/mod19d/db/platform_migrate | tail -1`
Expected: `20261009600001_…` nas duas. Se houver mais nova que `20261010700001`, use um número maior nas Tasks 2 e 4 (e no `define(version:)`).

Run: `rspec spec/services/platform spec/adr_pointers_spec.rb spec/commands/clinical_documents`
Expected: PASS (a base está verde antes de mexer).

---
## Fatia 1 — Interruptores, plataforma e catálogo (F-19.24)

### Task 1: Interruptores `controlled_prescriptions` e `sncr_mock`

**Files:**
- Modify: `app/services/platform/features.rb`, `spec/adr_pointers_spec.rb`, `spec/models/city_record_settings_spec.rb`, `spec/requests/operators/city_record_settings_spec.rb`, `spec/requests/maintenance/city_features_spec.rb`
- Create: `app/services/controlled_prescriptions/gate.rb`, `app/controllers/concerns/controlled_prescriptions_gate.rb`, `app/services/sncr/mock.rb`
- Test: `spec/services/platform/controlled_prescriptions_feature_spec.rb`

**Interfaces:**
- Consumes: `Platform::Features` (`feature:<key>` → `"<key>_disabled"`, `environments:`), `ClinicalDocuments::Gate::KEY`, `clinical_city!`, `ledi_maintainer!`.
- Produces: `ControlledPrescriptions::Gate::KEY == "controlled_prescriptions"`, `ControlledPrescriptions::Gate.usable?(city) -> bool`; `Sncr::Mock::KEY == "sncr_mock"`, `Sncr::Mock.on?(city, env: Rails.env) -> bool`; concern `ControlledPrescriptionsGate#require_controlled_prescriptions!` (403 `{ error: "feature_disabled", feature: "controlled_prescriptions" }`).

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/services/platform/controlled_prescriptions_feature_spec.rb
require "rails_helper"

# ADR 0034 (spec §3; contrato §8): controlled_prescriptions exige
# clinical_documents utilizável; sncr_mock exige controlled_prescriptions e
# só existe fora de produção (como o signature_psc_mock do 19b).
RSpec.describe "Interruptores do 19d" do
  let(:city) { clinical_city! }

  def set(key, enabled, env: Rails.env)
    Platform::Features.set!(city: city, key: key, enabled: enabled, maintainer: ledi_maintainer!, env: env)
  end

  it "controlled_prescriptions exige os documentos clínicos utilizáveis" do
    set("controlled_prescriptions", true)
    expect(Platform::Features.missing(city, "controlled_prescriptions")).to eq([ "clinical_documents_disabled" ])
    expect(ControlledPrescriptions::Gate.usable?(city)).to be(false)

    set("clinical_documents", true)
    expect(Platform::Features.missing(city, "controlled_prescriptions")).to eq([])
    expect(ControlledPrescriptions::Gate.usable?(city)).to be(true)
    expect(ControlledPrescriptions::Gate.usable?(nil)).to be(false)
  end

  it "sncr_mock exige controlled_prescriptions e não depende da assinatura" do
    set("clinical_documents", true)
    set("sncr_mock", true)
    expect(Platform::Features.missing(city, "sncr_mock")).to eq([ "controlled_prescriptions_disabled" ])
    expect(Sncr::Mock.on?(city)).to be(false)

    set("controlled_prescriptions", true)
    expect(Sncr::Mock.on?(city)).to be(true)
    expect(Platform::Features.enabled?(city, "digital_signature")).to be(false)
  end

  it "sncr_mock não existe em produção: fora do catálogo, recusa ligar, linha órfã não vale" do
    set("clinical_documents", true)
    set("controlled_prescriptions", true)
    set("sncr_mock", true)
    expect(Platform::Features.catalog(env: "production").map(&:key)).to include("controlled_prescriptions")
    expect(Platform::Features.catalog(env: "production").map(&:key)).not_to include("sncr_mock")
    expect { set("sncr_mock", true, env: "production") }.to raise_error(Platform::Features::UnknownFeature)
    expect(Sncr::Mock.on?(city, env: "production")).to be(false)
    expect(Sncr::Mock.on?(nil)).to be(false)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `rspec spec/services/platform/controlled_prescriptions_feature_spec.rb`
Expected: FAIL (`uninitialized constant ControlledPrescriptions` / interruptor fora do catálogo).

- [ ] **Step 3: Implemente**

Em `app/services/platform/features.rb`, acrescente ao fim de `CATALOG` (depois de `clinical_documents`):

```ruby
      # ADR 0034: receita de controle especial e de antimicrobiano digitais
      # (números do SNCR) e registro da Notificação de papel; só com os
      # documentos clínicos utilizáveis. Desligado: comportamento do 19c.
      Entry.new(key: "controlled_prescriptions",
                description: "Receita de controle especial e de antimicrobiano digitais (SNCR) e registro da Notificação de papel",
                requires: %w[feature:clinical_documents]),
      # ADR 0034: SNCR simulado, só fora de produção. Ligado, os números vêm do
      # SNCR simulado (numeração simulada — sem validade).
      Entry.new(key: "sncr_mock",
                description: "SNCR simulado (desenvolvimento) — numeração simulada, sem validade",
                requires: %w[feature:controlled_prescriptions],
                environments: SIMULATION_ENVS)
```

```ruby
# app/services/controlled_prescriptions/gate.rb
# Receita de controle especial e de antimicrobiano digitais e registro da
# Notificação (ADR 0034) só com controlled_prescriptions LIGADO e UTILIZÁVEL
# (requer clinical_documents utilizável). Relido da plataforma a cada chamada.
module ControlledPrescriptions
  module Gate
    KEY = "controlled_prescriptions".freeze

    module_function

    def usable?(city)
      return false if city.nil? || city.id.nil?

      Platform::Features.usable?(city, KEY)
    end
  end
end
```

```ruby
# app/services/sncr/mock.rb
# ADR 0034: SNCR simulado. Só existe fora de produção; ligado, a cidade pede
# números ao fake-sncr e usa só os números simulados do estoque.
module Sncr
  module Mock
    KEY = "sncr_mock".freeze

    module_function

    def on?(city, env: Rails.env)
      return false if city.nil? || city.id.nil?

      Platform::Features.usable?(city, KEY, env: env)
    end
  end
end
```

```ruby
# app/controllers/concerns/controlled_prescriptions_gate.rb
# Rotas do 19d (ADR 0034; contrato): interruptor desligado ou sem os
# documentos clínicos utilizáveis → 403 feature_disabled.
module ControlledPrescriptionsGate
  extend ActiveSupport::Concern

  private

  def require_controlled_prescriptions!
    return if ControlledPrescriptions::Gate.usable?(Current.city)

    render json: { error: "feature_disabled", feature: ControlledPrescriptions::Gate::KEY }, status: :forbidden
  end
end
```

Nas três specs com a lista fixa do catálogo (`spec/models/city_record_settings_spec.rb`, `spec/requests/operators/city_record_settings_spec.rb`, `spec/requests/maintenance/city_features_spec.rb`), a lista passa a
`%w[ledi_export cadsus_lookup clinical_record digital_signature signature_psc_mock clinical_documents controlled_prescriptions sncr_mock]`.
Em `spec/adr_pointers_spec.rb`: `VALID_RANGE = (1..34)`.

- [ ] **Step 4: Rode e veja passar**

Run: `rspec spec/services/platform spec/models/city_record_settings_spec.rb spec/requests/operators/city_record_settings_spec.rb spec/requests/maintenance/city_features_spec.rb spec/adr_pointers_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add app/services/platform/features.rb app/services/controlled_prescriptions/gate.rb app/controllers/concerns/controlled_prescriptions_gate.rb app/services/sncr/mock.rb spec/services/platform/controlled_prescriptions_feature_spec.rb spec/adr_pointers_spec.rb spec/models/city_record_settings_spec.rb spec/requests/operators/city_record_settings_spec.rb spec/requests/maintenance/city_features_spec.rb
git commit -m "feat: add the controlled_prescriptions and sncr_mock switches

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Plataforma — marca de anticonvulsivante, tipo novo das listas e checagem do SNCR

**Files:**
- Create: `db/platform_migrate/20261010700001_add_controlled_prescriptions_platform.rb`, `app/models/sncr_check.rb`
- Modify: `db/platform_schema.rb`, `app/models/anvisa_list_substance.rb` (se ele tiver `KINDS`)
- Test: `spec/models/controlled_platform_spec.rb`

**Interfaces:**
- Consumes: `PlatformRecord`, `MedicationCatalogItem`, `AnvisaListSubstance`, `AnvisaListRelease` (19c).
- Produces: coluna `medication_catalog_items.anticonvulsant` (bool, padrão false); `anvisa_list_substances.kind` aceita `anticonvulsant`; `anvisa_list_releases.anticonvulsants_count`; `SncrCheck` (`TARGETS = %w[real simulated]`, `.record!(target, ok:, at: Time.current)`, `.last(target) -> SncrCheck|nil`).

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/models/controlled_platform_spec.rb
require "rails_helper"

# ADR 0034: o catálogo da plataforma ganha a marca de anticonvulsivante
# (Portaria 344, art. 59: até 6 meses), as listas Anvisa aceitam o tipo
# anticonvulsant, e a última conversa com o SNCR fica registrada por alvo.
RSpec.describe "Plataforma do 19d" do
  it "item do catálogo nasce sem a marca de anticonvulsivante" do
    item = import_test_catalog!.values.first
    expect(item.anticonvulsant).to be(false)
  end

  it "a lista aceita o tipo anticonvulsant, sem lista da Portaria" do
    import_test_catalog!
    release = AnvisaListRelease.current
    row = AnvisaListSubstance.create!(release_id: release.id, kind: "anticonvulsant", substance: "CARBAMAZEPINA",
                                      substance_key: "CARBAMAZEPINA", controlled_list: nil)
    expect(row).to be_persisted
    expect { AnvisaListSubstance.create!(release_id: release.id, kind: "anticonvulsant", substance: "X", substance_key: "X", controlled_list: "C1") }
      .to raise_error(ActiveRecord::StatementInvalid)
    expect { AnvisaListSubstance.create!(release_id: release.id, kind: "outro", substance: "Y", substance_key: "Y") }
      .to raise_error(ActiveRecord::StatementInvalid)
  end

  it "SncrCheck guarda a última conversa por alvo e nunca derruba quem chamou" do
    SncrCheck.where(target: %w[real simulated]).delete_all
    SncrCheck.record!("real", ok: false, at: 2.minutes.ago)
    SncrCheck.record!("real", ok: true)
    expect(SncrCheck.last("real")).to have_attributes(last_check_ok: true)
    expect(SncrCheck.last("simulated")).to be_nil
    expect { SncrCheck.record!("outro", ok: true) }.not_to raise_error
    expect(SncrCheck.last("outro")).to be_nil
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `rspec spec/models/controlled_platform_spec.rb`
Expected: FAIL (`undefined method 'anticonvulsant'` / `uninitialized constant SncrCheck`).

- [ ] **Step 3: Escreva a migração e o modelo**

```ruby
# db/platform_migrate/20261010700001_add_controlled_prescriptions_platform.rb
# Plataforma do 19d (ADR 0034): marca de anticonvulsivante no catálogo (o
# limite de 180 dias da Portaria 344, art. 59), o tipo anticonvulsant nas
# listas Anvisa e a última conversa com o SNCR (real e simulado). O nome do
# CHECK de kind das listas é o conferido na Task 0 (Step 6).
class AddControlledPrescriptionsPlatform < ActiveRecord::Migration[8.1]
  def up
    add_column :medication_catalog_items, :anticonvulsant, :boolean, null: false, default: false
    add_column :anvisa_list_releases, :anticonvulsants_count, :integer, null: false, default: 0

    remove_check_constraint :anvisa_list_substances, name: "ck_anvisa_list_substances_kind"
    add_check_constraint :anvisa_list_substances,
                         "kind::text = ANY (ARRAY['antimicrobial'::text, 'controlled'::text, 'anticonvulsant'::text])",
                         name: "ck_anvisa_list_substances_kind"

    create_table :sncr_checks, id: :uuid do |t|
      t.string :target, null: false
      t.datetime :last_check_at, null: false
      t.boolean :last_check_ok, null: false
    end
    add_index :sncr_checks, :target, unique: true
    add_check_constraint :sncr_checks, "target::text = ANY (ARRAY['real'::text, 'simulated'::text])", name: "ck_sncr_checks_target"
  end

  def down
    raise ActiveRecord::IrreversibleMigration
  end
end
```

```ruby
# app/models/sncr_check.rb
# Última conversa com o SNCR (plataforma; ADR 0034; contrato §8): uma linha
# por alvo (real, simulated). Escrita por toda chamada do Sncr::Client; uma
# falha aqui nunca derruba o pedido de números (savepoint).
class SncrCheck < PlatformRecord
  TARGETS = %w[real simulated].freeze

  def self.record!(target, ok:, at: Time.current)
    return nil unless TARGETS.include?(target.to_s)

    transaction(requires_new: true) do
      upsert({ target: target.to_s, last_check_at: at, last_check_ok: ok }, unique_by: :target)
    end
  rescue ActiveRecord::ActiveRecordError => e
    Rails.logger.warn("[sncr_check] #{e.class}")
    nil
  end

  def self.last(target) = find_by(target: target.to_s)
end
```

No `db/platform_schema.rb`: `define(version: 2026_10_10_700001)`; em `medication_catalog_items`, `t.boolean "anticonvulsant", default: false, null: false`; em `anvisa_list_releases`, `t.integer "anticonvulsants_count", default: 0, null: false`; o CHECK novo de `anvisa_list_substances` no lugar do antigo; e a tabela:

```ruby
  create_table "sncr_checks", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.string "target", null: false
    t.datetime "last_check_at", null: false
    t.boolean "last_check_ok", null: false
    t.index [ "target" ], name: "index_sncr_checks_on_target", unique: true
    t.check_constraint "target::text = ANY (ARRAY['real'::text, 'simulated'::text])", name: "ck_sncr_checks_target"
  end
```
Se `AnvisaListSubstance` tiver `KINDS`, acrescente `anticonvulsant`.

- [ ] **Step 4: Migre a plataforma de teste** (avise antes — banco compartilhado)

Run: `docker compose exec -T -e RAILS_ENV=test -w /rails/.claude/mod19d api bin/rails db:migrate:platform`
Expected: `AddControlledPrescriptionsPlatform: migrated`.

- [ ] **Step 5: Rode e veja passar**

Run: `rspec spec/models/controlled_platform_spec.rb spec/models/medication_catalog_spec.rb`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add db/platform_migrate/20261010700001_add_controlled_prescriptions_platform.rb db/platform_schema.rb app/models/sncr_check.rb app/models/anvisa_list_substance.rb spec/models/controlled_platform_spec.rb
git commit -m "feat: add the anticonvulsant mark, the anticonvulsant list kind and SNCR checks to the platform

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Lista da Portaria 344 por item e marca de anticonvulsivante nas listas Anvisa

O 19c já grava, por item do catálogo, a lista de controle mais restritiva (`controlled_list`, A1…C5). Esta task acrescenta a lista de anticonvulsivantes (arquivo versionado como as do 19c), a marca no `Flags`, na varredura e no catálogo, e prova a lista por item que a categoria (Task 11) usa.

**Files:**
- Create: `config/medications/anvisa/anticonvulsants.csv`, `spec/fixtures/medications/anvisa/anticonvulsants.csv`
- Modify: `config/medications/anvisa/SOURCES.yml`, `spec/fixtures/medications/anvisa/SOURCES.yml`, `app/services/medications/substances.rb`, `app/services/medications/anvisa_import.rb`, `app/services/clinical_documents/json.rb` (item da busca), `spec/config/anvisa_lists_spec.rb`
- Test: `spec/services/medications/anticonvulsant_marks_spec.rb`

**Interfaces:**
- Consumes: `Medications::Substances` (`Flags`, `Matcher`, `matcher`, `apply_to_catalog!`), `Medications::AnvisaImport` (19c), Task 2.
- Produces: `Medications::Substances::Flags` com `anticonvulsant` (padrão `false` — chamadas do 19c com três chaves continuam valendo); `Matcher.new(antimicrobials:, controlled:, anticonvulsants: [])`; `Matcher#flags_for`/`#scan` marcam `anticonvulsant`; `apply_to_catalog!` grava `anticonvulsant`; `AnvisaImport` lê `anticonvulsants.csv` (contagem conferida com `SOURCES.yml` → `anticonvulsants.count`), grava `kind: "anticonvulsant"` e `anticonvulsants_count`; `ClinicalDocuments::Json.catalog_item(item, in_network: nil)` (busca e REMUME, contrato §1) ganha `controlled_list` e `anticonvulsant` (`catalog_ref`, que vai ao canônico, não muda).

- [ ] **Step 1: Transcreva a lista (dados, não código)**

Fonte: a classificação ATC/OMS, grupo **N03A (antiepilépticos)**, na versão vigente (`https://atcddd.fhi.no/atc_ddd_index/?code=N03A`, só leitura), cruzada com a RENAME vigente (seção de antiepilépticos). Uma substância por linha, nome DCB em maiúsculas, como nas listas do 19c; inclua as que estão na lista C1 do `controlled.csv` do 19c (é a elas que o limite de 180 dias se aplica) e as demais do N03A (a marca não muda nada fora do C1):

```csv
substance
CARBAMAZEPINA
```
(cabeçalho e o formato; o arquivo final tem todas as substâncias lidas)

Em `config/medications/anvisa/SOURCES.yml` acrescente (sem nenhum `<…>` no arquivo final):

```yaml
anticonvulsants:
  label: "ATC/OMS N03A (antiepilépticos), cruzada com a RENAME <ano> — limite de 6 meses da Portaria 344/1998, art. 59"
  read_at: "<AAAA-MM-DD>"
  source: "<endereço lido>"
  count: <linhas sem o cabeçalho>
```

Fixtures de teste: `spec/fixtures/medications/anvisa/anticonvulsants.csv`

```csv
substance
CARBAMAZEPINA
ACIDO VALPROICO
```
e, em `spec/fixtures/medications/anvisa/SOURCES.yml`:

```yaml
anticonvulsants:
  label: "Fixture de teste (recorte ATC N03A)"
  read_at: "2026-10-10"
  source: "spec/fixtures"
  count: 2
```

- [ ] **Step 2: Escreva as specs que falham**

Em `spec/config/anvisa_lists_spec.rb`, acrescente:

```ruby
  it "os anticonvulsivantes batem com a fonte declarada e cobrem os da APS" do
    names = CSV.read(dir.join("anticonvulsants.csv"), headers: true).map { |row| row["substance"] }
    expect(names.size).to eq(sources.dig("anticonvulsants", "count"))
    expect(names).to eq(names.uniq)
    keys = names.map { |name| Medications::Substances.key(name) }
    expect(keys).to include("CARBAMAZEPINA", "ACIDO VALPROICO", "FENITOINA", "LAMOTRIGINA", "TOPIRAMATO", "OXCARBAZEPINA")
  end
```

```ruby
# spec/services/medications/anticonvulsant_marks_spec.rb
require "rails_helper"

# ADR 0034: a lista da Portaria 344 por item (a mais restritiva, do 19c) e a
# marca de anticonvulsivante (art. 59) chegam ao catálogo e à varredura de
# texto livre; quem chama o Flags com três chaves (19c) segue funcionando.
RSpec.describe "Lista por item e anticonvulsivante" do
  let(:matcher) do
    Medications::Substances::Matcher.new(antimicrobials: %w[AMOXICILINA],
                                         controlled: { "CARBAMAZEPINA" => "C1", "DIAZEPAM" => "B1", "TALIDOMIDA" => "C3" },
                                         anticonvulsants: %w[CARBAMAZEPINA])
  end

  it "marca o anticonvulsivante por ingrediente e no texto livre" do
    expect(matcher.flags_for([ "CARBAMAZEPINA" ])).to have_attributes(controlled: true, controlled_list: "C1", anticonvulsant: true)
    expect(matcher.flags_for([ "DIAZEPAM" ])).to have_attributes(controlled_list: "B1", anticonvulsant: false)
    expect(matcher.scan("carbamazepina 200 mg").anticonvulsant).to be(true)
    expect(matcher.flags_for([ "TALIDOMIDA" ]).controlled_list).to eq("C3")
  end

  it "Flags com três chaves (19c) segue valendo, anticonvulsant false" do
    flags = Medications::Substances::Flags.new(antimicrobial: false, controlled: false, controlled_list: nil)
    expect(flags.anticonvulsant).to be(false)
    expect(Medications::Substances::NONE.anticonvulsant).to be(false)
  end

  it "a importação grava o tipo novo, a contagem e a marca do catálogo" do
    import_test_catalog!
    release = Medications::AnvisaImport.call(by: "rspec", dir: anvisa_fixture_dir).payload.fetch(:release)
    expect(release.anticonvulsants_count).to eq(2)
    expect(AnvisaListSubstance.where(release_id: release.id, kind: "anticonvulsant").pluck(:substance_key))
      .to contain_exactly("CARBAMAZEPINA", "ACIDO VALPROICO")
    item = controlled_item!(code: 9_900_101, ingredient: "CARBAMAZEPINA", list: nil)
    Medications::Substances.apply_to_catalog!(Medications::Substances.matcher(release))
    expect(item.reload).to have_attributes(anticonvulsant: true)
    expect(ClinicalDocuments::Json.catalog_item(item.reload)).to include(controlled_list: nil, anticonvulsant: true)
    expect(ClinicalDocuments::Json.catalog_ref(item)).not_to have_key(:anticonvulsant)
  end

  it "contagem diferente do SOURCES.yml falha a importação" do
    Dir.mktmpdir do |dir|
      FileUtils.cp(Dir[anvisa_fixture_dir.join("*")], dir)
      File.write(File.join(dir, "anticonvulsants.csv"), "substance\nCARBAMAZEPINA\n")
      result = Medications::AnvisaImport.call(by: "rspec", dir: dir)
      expect(result.reason).to eq(:invalid_file)
    end
  end
end
```
(`controlled_item!` e `anvisa_fixture_dir` são os helpers da Task 4 e do 19c; esta spec passa ao fim da Task 4.)

- [ ] **Step 3: Rode e veja falhar**

Run: `rspec spec/config/anvisa_lists_spec.rb spec/services/medications/anticonvulsant_marks_spec.rb`
Expected: FAIL (`unknown keyword: :anticonvulsants`).

- [ ] **Step 4: Implemente**

Em `app/services/medications/substances.rb`, troque `Flags`, `NONE`, o `apply_to_catalog!` e o `Matcher` por:

```ruby
    # anticonvulsant (ADR 0034): padrão false — o 19c cria Flags com três chaves.
    Flags = Data.define(:antimicrobial, :controlled, :controlled_list, :anticonvulsant) do
      def initialize(antimicrobial:, controlled:, controlled_list:, anticonvulsant: false) = super
    end
    NONE = Flags.new(antimicrobial: false, controlled: false, controlled_list: nil)

    def matcher(release = AnvisaListRelease.current)
      return Matcher.new(antimicrobials: [], controlled: {}) unless release

      rows = AnvisaListSubstance.where(release_id: release.id).pluck(:kind, :substance_key, :controlled_list)
      Matcher.new(antimicrobials: rows.select { |kind, *| kind == "antimicrobial" }.map(&:second),
                  controlled: rows.select { |kind, *| kind == "controlled" }.to_h { |_kind, substance, list| [ substance, list ] },
                  anticonvulsants: rows.select { |kind, *| kind == "anticonvulsant" }.map(&:second))
    end

    def apply_to_catalog!(matcher)
      changed = 0
      MedicationCatalogItem.find_each do |item|
        flags = matcher.flags_for(item.ingredients)
        current = [ item.antimicrobial, item.controlled, item.controlled_list, item.anticonvulsant ]
        next if current == [ flags.antimicrobial, flags.controlled, flags.controlled_list, flags.anticonvulsant ]

        item.update!(antimicrobial: flags.antimicrobial, controlled: flags.controlled, controlled_list: flags.controlled_list,
                     anticonvulsant: flags.anticonvulsant)
        changed += 1
      end
      changed
    end

    class Matcher
      def initialize(antimicrobials:, controlled:, anticonvulsants: [])
        @antimicrobials = antimicrobials.map { |name| Substances.key(name) }.uniq
        @controlled = controlled.transform_keys { |name| Substances.key(name) }
        @anticonvulsants = anticonvulsants.map { |name| Substances.key(name) }.uniq
      end

      def empty? = @antimicrobials.empty? && @controlled.empty?

      def flags_for(ingredients)
        keys = Array(ingredients).map { |name| Substances.key(name) }
        lists = keys.flat_map { |ingredient| @controlled.filter_map { |substance, list| list if starts?(ingredient, substance) } }
        Flags.new(antimicrobial: keys.any? { |ingredient| @antimicrobials.any? { |substance| starts?(ingredient, substance) } },
                  controlled: lists.any?, controlled_list: lists.min,
                  anticonvulsant: keys.any? { |ingredient| @anticonvulsants.any? { |substance| starts?(ingredient, substance) } })
      end

      def scan(text)
        padded = " #{Substances.key(text)} "
        lists = @controlled.filter_map { |substance, list| list if padded.include?(" #{substance} ") }
        Flags.new(antimicrobial: @antimicrobials.any? { |substance| padded.include?(" #{substance} ") },
                  controlled: lists.any?, controlled_list: lists.min,
                  anticonvulsant: @anticonvulsants.any? { |substance| padded.include?(" #{substance} ") })
      end

      def inspect = "#<Medications::Substances::Matcher #{@antimicrobials.size}/#{@controlled.size}/#{@anticonvulsants.size}>"

      private

      def starts?(ingredient, substance) = ingredient == substance || ingredient.start_with?("#{substance} ")
    end
```

Em `app/services/medications/anvisa_import.rb`:
- `read(dir)` lê também `anticonvulsants = CSV.read(dir.join("anticonvulsants.csv"), headers: true, encoding: "UTF-8").map { |row| row.fetch("substance").to_s.strip }`, levanta `Invalid, "anticonvulsants: contagem #{anticonvulsants.size} ≠ SOURCES …"` quando `anticonvulsants.size != sources.dig("anticonvulsants", "count")` e devolve `[ sources, antimicrobials, controlled, anticonvulsants ]`;
- `call` desestrutura os quatro, soma às `rows` `anticonvulsants.map { |name| { release_id: release.id, kind: "anticonvulsant", substance: name, substance_key: Substances.key(name), controlled_list: nil } }` e grava `anticonvulsants_count: anticonvulsants.size` no `release.update!`;
- `sha256(dir)` passa a ler `%w[SOURCES.yml antimicrobials.csv controlled.csv anticonvulsants.csv]`.
O evento `anvisa_lists.imported` não muda (payload fixado no 19c).

Em `app/services/clinical_documents/json.rb`, o item da busca e da REMUME (contrato §1 do 19d) ganha a lista e a marca:

```ruby
    def catalog_item(item, in_network: nil)
      json = catalog_ref(item).merge(antimicrobial: item.antimicrobial, controlled: item.controlled,
                                     controlled_list: item.controlled_list, anticonvulsant: item.anticonvulsant)
      in_network.nil? ? json : json.merge(in_network: in_network)
    end
```

- [ ] **Step 5: Rode e veja passar (a spec das marcas passa ao fim da Task 4)**

Run: `rspec spec/config/anvisa_lists_spec.rb spec/services/medications`
Expected: PASS, menos `anticonvulsant_marks_spec.rb` (helpers da Task 4).

- [ ] **Step 6: Commit**

```bash
git add config/medications/anvisa/anticonvulsants.csv config/medications/anvisa/SOURCES.yml spec/fixtures/medications/anvisa/anticonvulsants.csv spec/fixtures/medications/anvisa/SOURCES.yml app/services/medications/substances.rb app/services/medications/anvisa_import.rb app/services/clinical_documents/json.rb spec/config/anvisa_lists_spec.rb spec/services/medications/anticonvulsant_marks_spec.rb
git commit -m "feat: mark anticonvulsants in the medication catalog from a versioned list

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
## Fatia 2 — Dados da cidade e SNCR (F-19.24, F-19.25)

### Task 4: Tabelas da cidade, triggers, cifra, eventos e helpers

**Files:**
- Create: `db/city_migrate/20261010700001_add_controlled_prescriptions.rb`, `app/models/sncr_number_batch.rb`, `app/models/sncr_number.rb`, `spec/support/controlled_prescription_helpers.rb`
- Modify: `db/city_schema.rb`, `db/city_triggers.sql`, `app/models/professional.rb`, `app/models/clinical_document.rb`, `app/models/signature_oauth_state.rb`, `app/services/city_encryption.rb`, `config/initializers/domain_events.rb`, `config/initializers/filter_parameter_logging.rb`, `spec/initializers/domain_events_bindings_spec.rb`, `spec/rails_helper.rb`
- Test: `spec/models/sncr_tables_guard_spec.rb`

**Interfaces:**
- Consumes: `documents_city!`, `ledi_maintainer!`, `import_test_catalog!`, `doctor!`, `create_unit`, `CityEncryption::CITY_KEYED_TARGETS`, `ClinicalDocument::KINDS`, `SignatureOauthState::PURPOSES`.
- Produces:
  - `SncrNumberBatch` (`KINDS = %w[rce ret]`, `has_many :numbers`, colunas `user_id`, `kind`, `first_number`, `last_number`, `quantity`, `month` (date, 1º dia do mês), `month_request`, `simulated`, `requested_at`).
  - `SncrNumber` (`KINDS`, `STATUSES = %w[free used voided]`, `FORMAT`, `scope :available`, colunas `kind`, `number`, `batch_id`, `user_id`, `simulated`, `status`, `document_id`, `used_at`, `voided_at`).
  - `Professional#prescriber_address` (JSON cifrado), `Professional#prescriber_contact -> Hash|nil` (`{ "address" => {…}, "phone" => "…" }` só com os dois).
  - `ClinicalDocument::KINDS` com `controlled_notification_record`; `ClinicalDocument::PAPER_REASONS`; coluna `clinical_documents.paper_reason` (nula no digital; imutável); `ClinicalDocument#notification_record?`.
  - `SignatureOauthState::PURPOSES` com `sncr_rce`, `sncr_ret`.
  - Eventos declarados: `sncr.numbers_requested`, `sncr.number_used`, `sncr.number_voided`, `controlled_notification.recorded`.
  - Helpers: `controlled_city!(mock: true, signature: true)`, `controlled_item!(code:, ingredient:, list: nil, antimicrobial: false, anticonvulsant: false)`, `prescriber_contact!(user, phone: "41999990000")`, `patient_identification(**over) -> Hash` (só `address`), `fake_sncr`, `stub_sncr_mock!`, `stub_sncr_real!`, `sncr_stock!(user, kind: "rce", count: 3, simulated: true)`; constantes `ControlledPrescriptionHelpers::{ADDRESS, PATIENT_CPF, FAKE_SNCR_BASE, SNCR_REAL_BASE, SNCR_REAL_AUTH, MAINTAINER_CNPJ}`.

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/models/sncr_tables_guard_spec.rb
require "rails_helper"

# ADR 0034 (Invariantes): o banco é a última palavra. Um número nasce free,
# passa a used com o documento e a voided depois; nunca volta, nunca muda de
# documento, nunca é apagado; (kind, number) único; o lote é só acréscimo.
RSpec.describe "Tabelas do estoque SNCR (triggers)" do
  before { documents_city!; ciap2_release!; cid10_release!; sigtap_release! }

  let(:unit) { create_unit }
  let(:user) { doctor!(unit) }
  # O número aponta para um documento de verdade (FK); os gatilhos rodam antes da FK.
  let(:document_id) { raw_document!(consultation: finalized_consultation!(unit: unit, doctor: user, citizen: verified_citizen!(1))).id }

  def sql(statement) = ApplicationRecord.connection.execute(statement)

  def batch!(kind: "rce")
    SncrNumberBatch.create!(user: user, kind: kind, first_number: "2610.1-41.0000001", last_number: "2610.1-41.0000002",
                            quantity: 2, month: Date.current.beginning_of_month, month_request: 1, simulated: true,
                            requested_at: Time.current)
  end

  def number!(batch, value = "2610.1-41.0000001", kind: "rce")
    SncrNumber.create!(batch: batch, user: user, kind: kind, number: value, simulated: true)
  end

  it "free → used → voided; nunca volta, nunca troca de documento, nunca some" do
    number = number!(batch!)
    expect { number.update!(status: "voided", voided_at: Time.current) }.to raise_error(ActiveRecord::StatementInvalid, /refused/)
    number.update!(status: "used", document_id: document_id, used_at: Time.current)
    expect { number.update!(document_id: SecureRandom.uuid) }.to raise_error(ActiveRecord::StatementInvalid, /refused/)
    expect { number.update!(status: "free", document_id: nil, used_at: nil) }.to raise_error(ActiveRecord::StatementInvalid)
    number.update!(status: "voided", voided_at: Time.current)
    expect { number.update!(status: "used", voided_at: nil) }.to raise_error(ActiveRecord::StatementInvalid)
    expect { number.update_columns(number: "2610.1-41.0000009") }.to raise_error(ActiveRecord::StatementInvalid, /never changes/)
    expect { number.destroy }.to raise_error(ActiveRecord::StatementInvalid, /DELETE refused/)
    expect { sql("TRUNCATE sncr_numbers CASCADE") }.to raise_error(ActiveRecord::StatementInvalid, /append-only/)
  end

  it "nasce free, sem documento; (kind, number) único; formato do número" do
    batch = batch!
    number!(batch)
    expect { number!(batch) }.to raise_error(ActiveRecord::RecordNotUnique)
    expect(number!(batch!(kind: "ret"), kind: "ret")).to be_persisted # o mesmo texto noutro tipo é outro número
    expect { SncrNumber.create!(batch: batch, user: user, kind: "rce", number: "2610.1-41.0000003", simulated: true, status: "used", document_id: document_id, used_at: Time.current) }
      .to raise_error(ActiveRecord::StatementInvalid, /born free/)
    expect { number!(batch, "12345") }.to raise_error(ActiveRecord::StatementInvalid, /ck_sncr_numbers_format/)
  end

  it "o lote é só acréscimo" do
    batch = batch!
    expect { batch.update!(quantity: 3) }.to raise_error(ActiveRecord::StatementInvalid, /append-only/)
    expect { batch.destroy }.to raise_error(ActiveRecord::StatementInvalid, /append-only/)
  end

  it "o motivo do papel é gravado, só no papel, com os valores do contrato, e nunca muda" do
    document = raw_document!(consultation: finalized_consultation!(unit: unit, doctor: user, citizen: verified_citizen!(2)))
    expect(document.paper_reason).to be_nil
    expect { document.update_columns(paper_reason: "no_certificate") }.to raise_error(ActiveRecord::StatementInvalid, /never changes/)
    attrs = document.attributes.except("id", "txid").merge("verification_token" => ClinicalDocuments::Codes.token,
                                                           "short_code" => ClinicalDocuments::Codes.short_code)
    expect { ClinicalDocument.create!(attrs.merge("paper_reason" => "outro")) }.to raise_error(ActiveRecord::StatementInvalid, /ck_clinical_documents_paper_reason/)
    expect { ClinicalDocument.create!(attrs.merge("issue_mode" => "digital", "paper_reason" => "no_certificate")) }
      .to raise_error(ActiveRecord::StatementInvalid, /ck_clinical_documents_paper_reason_mode/)
    expect(ClinicalDocument::PAPER_REASONS).to eq(%w[no_certificate signature_unavailable no_sncr_number nurse_antimicrobial feature_disabled])
  end

  it "o tipo novo de documento é sempre papel; o state aceita os propósitos do SNCR" do
    expect(ClinicalDocument::KINDS).to include("controlled_notification_record")
    expect(SignatureOauthState::PURPOSES).to include("sncr_rce", "sncr_ret")
    state = SignatureOauthState.create!(user: user, purpose: "sncr_rce", provider: "sncr_simulated", code_verifier: "v" * 43,
                                        return_to: "/dashboard/", expires_at: 10.minutes.from_now, created_at: Time.current)
    expect(state).to be_persisted
  end

  it "o endereço do prescritor é cifrado e o contato só existe completo" do
    professional = user.professional
    expect(professional.prescriber_contact).to be_nil
    prescriber_contact!(user)
    raw = ApplicationRecord.connection.select_value("SELECT prescriber_address FROM professionals WHERE id = #{ApplicationRecord.connection.quote(professional.id)}")
    expect(raw).not_to include("Araucárias")
    expect(professional.reload.prescriber_contact).to eq("address" => ControlledPrescriptionHelpers::ADDRESS, "phone" => "41999990000")
  end
end
```

Em `spec/initializers/domain_events_bindings_spec.rb`, acrescente:

```ruby
# Módulo 19d (ADR 0034): estoque SNCR e registro da Notificação, só trilha.
RSpec.describe "controlled prescription event bindings (ADR 0034)" do
  it "declares every module 19d event with no consumer" do
    names = %w[sncr.numbers_requested sncr.number_used sncr.number_voided controlled_notification.recorded]
    expect(DomainEvents.registry.keys).to include(*names)
    expect(names.flat_map { |n| DomainEvents.registry[n] }).to be_empty
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `rspec spec/models/sncr_tables_guard_spec.rb spec/initializers/domain_events_bindings_spec.rb`
Expected: FAIL (`uninitialized constant SncrNumberBatch`).

- [ ] **Step 3: Escreva a migração**

```ruby
# db/city_migrate/20261010700001_add_controlled_prescriptions.rb
# Receita de controlado e antimicrobiano com SNCR (ADR 0034; spec 2026-10-10
# §3–§5). Estoque de números por prescritor (lotes só acréscimo; número
# free → used → voided por trigger, único por tipo), endereço do prescritor
# (cifrado), o documento controlled_notification_record (sempre papel) e os
# propósitos do state do SNCR no state de uso único do 19b. Triggers em
# db/city_triggers.sql. Irreversível.
class AddControlledPrescriptions < ActiveRecord::Migration[8.1]
  KINDS = %w[sick_note attendance_declaration prescription exam_requisition controlled_notification_record].freeze
  PURPOSES = %w[link session batch sncr_rce sncr_ret].freeze
  PAPER_REASONS = %w[no_certificate signature_unavailable no_sncr_number nurse_antimicrobial feature_disabled].freeze
  # A lista do 19b (conferida na Task 0, Step 6) somada aos provedores do SNCR.
  PROVIDERS = %w[vidaas birdid safeid neoid remoteid simulated sncr sncr_simulated].freeze

  def text_in(column, values) = "#{column}::text = ANY (ARRAY[#{values.map { |v| "'#{v}'::text" }.join(', ')}])"

  def up
    create_table :sncr_number_batches, id: :uuid do |t|
      t.uuid :user_id, null: false
      t.string :kind, null: false
      t.string :first_number, null: false
      t.string :last_number, null: false
      t.integer :quantity, null: false
      t.date :month, null: false
      t.integer :month_request, null: false
      t.boolean :simulated, null: false
      t.datetime :requested_at, null: false
      t.datetime :created_at, null: false
    end
    add_index :sncr_number_batches, %i[user_id kind month], name: "idx_sncr_number_batches_month"
    add_foreign_key :sncr_number_batches, :users
    add_check_constraint :sncr_number_batches, text_in("kind", %w[rce ret]), name: "ck_sncr_number_batches_kind"
    add_check_constraint :sncr_number_batches, "quantity BETWEEN 1 AND 1000", name: "ck_sncr_number_batches_quantity"
    add_check_constraint :sncr_number_batches, "month_request >= 1", name: "ck_sncr_number_batches_month_request"
    add_check_constraint :sncr_number_batches, "month = date_trunc('month', month)::date", name: "ck_sncr_number_batches_month"

    create_table :sncr_numbers, id: :uuid do |t|
      t.string :kind, null: false
      t.string :number, null: false
      t.uuid :batch_id, null: false
      t.uuid :user_id, null: false
      t.boolean :simulated, null: false
      t.string :status, null: false, default: "free"
      t.uuid :document_id
      t.datetime :used_at
      t.datetime :voided_at
      t.datetime :created_at, null: false
    end
    add_index :sncr_numbers, %i[kind number], unique: true, name: "idx_sncr_numbers_kind_number"
    add_index :sncr_numbers, %i[user_id kind simulated number], where: "status = 'free'", name: "idx_sncr_numbers_free"
    add_index :sncr_numbers, :document_id, unique: true, where: "document_id IS NOT NULL", name: "idx_sncr_numbers_document"
    add_index :sncr_numbers, :batch_id
    add_foreign_key :sncr_numbers, :sncr_number_batches, column: :batch_id
    add_foreign_key :sncr_numbers, :users
    add_foreign_key :sncr_numbers, :clinical_documents, column: :document_id
    add_check_constraint :sncr_numbers, text_in("kind", %w[rce ret]), name: "ck_sncr_numbers_kind"
    add_check_constraint :sncr_numbers, text_in("status", %w[free used voided]), name: "ck_sncr_numbers_status"
    add_check_constraint :sncr_numbers, "number::text ~ '^[0-9]{4}\\.[0-9]-[0-9]{2}\\.[0-9]{7}$'::text", name: "ck_sncr_numbers_format"
    add_check_constraint :sncr_numbers,
                         "(status::text = 'free'::text) = (document_id IS NULL) AND (status::text = 'free'::text) = (used_at IS NULL) " \
                         "AND (status::text = 'voided'::text) = (voided_at IS NOT NULL)",
                         name: "ck_sncr_numbers_state"

    add_column :professionals, :prescriber_address, :text

    remove_check_constraint :clinical_documents, name: "ck_clinical_documents_kind"
    add_check_constraint :clinical_documents, text_in("kind", KINDS), name: "ck_clinical_documents_kind"
    add_check_constraint :clinical_documents,
                         "kind::text <> 'controlled_notification_record'::text OR issue_mode::text = 'paper'::text",
                         name: "ck_clinical_documents_notification_paper"
    # Desvio 10: o motivo do papel é gravado (nulo no digital e onde não se aplica).
    add_column :clinical_documents, :paper_reason, :string
    add_check_constraint :clinical_documents, "paper_reason IS NULL OR #{text_in('paper_reason', PAPER_REASONS)}",
                         name: "ck_clinical_documents_paper_reason"
    add_check_constraint :clinical_documents, "paper_reason IS NULL OR issue_mode::text = 'paper'::text",
                         name: "ck_clinical_documents_paper_reason_mode"

    remove_check_constraint :signature_oauth_states, name: "ck_signature_oauth_states_purpose"
    add_check_constraint :signature_oauth_states, text_in("purpose", PURPOSES), name: "ck_signature_oauth_states_purpose"
    remove_check_constraint :signature_oauth_states, name: "ck_signature_oauth_states_provider"
    add_check_constraint :signature_oauth_states, text_in("provider", PROVIDERS), name: "ck_signature_oauth_states_provider"

    execute File.read(Rails.root.join("db/city_triggers.sql"))
  end

  def down
    raise ActiveRecord::IrreversibleMigration
  end
end
```

No fim de `db/city_triggers.sql`:

```sql
-- 19d (ADR 0034): estoque de números SNCR. Número nasce free; só
-- free → used (com documento) e used → voided; número, tipo, lote, dono e
-- simulado nunca mudam; nada é apagado. Lote só acréscimo.
CREATE OR REPLACE FUNCTION rota_sncr_number_guard() RETURNS trigger LANGUAGE plpgsql AS $$
BEGIN
  IF TG_OP = 'DELETE' THEN
    RAISE EXCEPTION 'sncr_numbers: DELETE refused';
  END IF;
  IF TG_OP = 'INSERT' THEN
    IF NEW.status <> 'free' OR NEW.document_id IS NOT NULL THEN
      RAISE EXCEPTION 'sncr_numbers: a number is born free';
    END IF;
    RETURN NEW;
  END IF;
  IF NEW.number <> OLD.number OR NEW.kind <> OLD.kind OR NEW.batch_id <> OLD.batch_id OR NEW.user_id <> OLD.user_id
     OR NEW.simulated <> OLD.simulated OR NEW.created_at <> OLD.created_at THEN
    RAISE EXCEPTION 'sncr_numbers: a number never changes';
  END IF;
  IF OLD.status = 'free' AND NEW.status = 'used' AND NEW.document_id IS NOT NULL THEN
    RETURN NEW;
  END IF;
  IF OLD.status = 'used' AND NEW.status = 'voided' AND NEW.document_id = OLD.document_id AND NEW.used_at = OLD.used_at THEN
    RETURN NEW;
  END IF;
  RAISE EXCEPTION 'sncr_numbers: transition % -> % refused', OLD.status, NEW.status;
END
$$;

CREATE OR REPLACE FUNCTION rota_sncr_append_only() RETURNS trigger LANGUAGE plpgsql AS $$
BEGIN
  RAISE EXCEPTION '%: append-only (% refused)', TG_TABLE_NAME, TG_OP;
END
$$;

DO $$
BEGIN
  IF to_regclass('public.sncr_numbers') IS NOT NULL THEN
    DROP TRIGGER IF EXISTS sncr_numbers_guard ON sncr_numbers;
    CREATE TRIGGER sncr_numbers_guard BEFORE INSERT OR UPDATE OR DELETE ON sncr_numbers
      FOR EACH ROW EXECUTE FUNCTION rota_sncr_number_guard();
    DROP TRIGGER IF EXISTS sncr_numbers_no_truncate ON sncr_numbers;
    CREATE TRIGGER sncr_numbers_no_truncate BEFORE TRUNCATE ON sncr_numbers
      FOR EACH STATEMENT EXECUTE FUNCTION rota_sncr_append_only();
  END IF;
  IF to_regclass('public.sncr_number_batches') IS NOT NULL THEN
    DROP TRIGGER IF EXISTS sncr_number_batches_append_only ON sncr_number_batches;
    CREATE TRIGGER sncr_number_batches_append_only BEFORE UPDATE OR DELETE ON sncr_number_batches
      FOR EACH ROW EXECUTE FUNCTION rota_sncr_append_only();
    DROP TRIGGER IF EXISTS sncr_number_batches_no_truncate ON sncr_number_batches;
    CREATE TRIGGER sncr_number_batches_no_truncate BEFORE TRUNCATE ON sncr_number_batches
      FOR EACH STATEMENT EXECUTE FUNCTION rota_sncr_append_only();
  END IF;
END
$$;
```

> O `TRUNCATE sncr_numbers CASCADE` da spec bate primeiro no gatilho de `sncr_numbers` ("append-only (TRUNCATE refused)"). O texto `/refused/` da primeira spec casa as duas mensagens de transição.

No `db/city_schema.rb`: `define(version: 2026_10_10_700001)`; as duas tabelas com índices e CHECKs iguais aos da migração; `t.text "prescriber_address"` em `professionals`; os CHECKs trocados de `clinical_documents` e `signature_oauth_states`; os novos `ck_clinical_documents_notification_paper`, `ck_clinical_documents_paper_reason` e `ck_clinical_documents_paper_reason_mode`; `t.string "paper_reason"` em `clinical_documents`. Se o gatilho de imutabilidade do 19c em `clinical_documents` listar as colunas uma a uma (em vez de comparar a linha inteira), acrescente `paper_reason` à lista no bloco novo do `db/city_triggers.sql` (a spec "o motivo do papel … nunca muda" acusa). A paridade (`spec/services/city_schema_spec.rb`) acusa o que faltar.

- [ ] **Step 4: Modelos, cifra, eventos, filtros**

```ruby
# app/models/sncr_number_batch.rb
# Lote de números do SNCR de um prescritor (ADR 0034; spec §4): o que o SNCR
# respondeu (início, fim, quantidade), o mês e o número do pedido no mês, e se
# veio do SNCR simulado. Só acréscimo.
class SncrNumberBatch < ApplicationRecord
  KINDS = %w[rce ret].freeze

  belongs_to :user
  has_many :numbers, class_name: "SncrNumber", foreign_key: :batch_id, inverse_of: :batch, dependent: :restrict_with_exception

  def inspect = "#<SncrNumberBatch id=#{id} kind=#{kind} quantity=#{quantity} simulated=#{simulated}>"
end
```

```ruby
# app/models/sncr_number.rb
# Número SNCR do estoque de um prescritor (ADR 0034; spec §4). free → used
# (na transação da emissão, com o documento) → voided (cancelamento ou volta
# ao papel). O número nunca vai a log, evento ou inspect.
class SncrNumber < ApplicationRecord
  KINDS = %w[rce ret].freeze
  STATUSES = %w[free used voided].freeze
  FORMAT = /\A\d{4}\.\d-\d{2}\.\d{7}\z/

  belongs_to :batch, class_name: "SncrNumberBatch", inverse_of: :numbers
  belongs_to :user

  scope :available, -> { where(status: "free") }

  def inspect = "#<SncrNumber id=#{id} kind=#{kind} status=#{status} simulated=#{simulated}>"
  alias_method :to_s, :inspect
end
```

Em `app/models/professional.rb`:

```ruby
  # ADR 0034 (Lei 5.991, art. 35): endereço do prescritor impresso na RCE e na
  # RET digitais; JSON cifrado com a chave da cidade. O telefone é o `phone`.
  encrypts :prescriber_address

  def prescriber_address_data
    data = prescriber_address.present? ? JSON.parse(prescriber_address) : nil
    data.is_a?(Hash) ? data : nil
  rescue JSON::ParserError
    nil
  end

  # { "address" => {…}, "phone" => "…" } só quando os dois existem.
  def prescriber_contact
    address = prescriber_address_data
    address && phone.present? ? { "address" => address, "phone" => phone } : nil
  end
```
e `prescriber_address` entra em `SELF_EDITABLE` (o profissional edita o próprio contato).

Em `app/models/clinical_document.rb`: `KINDS` passa a `%w[sick_note attendance_declaration prescription exam_requisition controlled_notification_record]`, `PAPER_REASONS = %w[no_certificate signature_unavailable no_sncr_number nurse_antimicrobial feature_disabled].freeze` e `def notification_record? = kind == "controlled_notification_record"`. Em `app/models/signature_oauth_state.rb`: `PURPOSES = %w[link session batch sncr_rce sncr_ret].freeze`.

Em `app/services/city_encryption.rb`, acrescente `[ "Professional", :prescriber_address ]` a `CITY_KEYED_TARGETS` (no formato das entradas existentes). Se o trigger de `professionals` recusar update sem `rota.reencrypting`, nada muda: não há trigger de imutabilidade em `professionals` (módulo 10).

Em `config/initializers/domain_events.rb`:

```ruby
  # Módulo 19d (ADR 0034): estoque SNCR e registro da Notificação — só ids.
  DomainEvents.bind "sncr.numbers_requested", to: []
  DomainEvents.bind "sncr.number_used", to: []
  DomainEvents.bind "sncr.number_voided", to: []
  DomainEvents.bind "controlled_notification.recorded", to: []
```

Em `config/initializers/filter_parameter_logging.rb`, acrescente à lista:
`/\A(session_id|access_token|street|zip|address|phone|paper_number|sncr|patient_identification|prescriber_contact)\z/`
(âncora: `email_address` continua como está).

- [ ] **Step 5: Helpers de spec**

```ruby
# spec/support/controlled_prescription_helpers.rb
require_relative "../../lib/fake_sncr/app"

# Módulo 19d (ADR 0034): cidade com receita controlada, SNCR simulado e real
# servidos pelo FakeSncr::App (WebMock#to_rack), itens do catálogo com lista e
# marcas, contato do prescritor e identificação do paciente.
module ControlledPrescriptionHelpers
  ADDRESS = { "street" => "Rua das Araucárias", "number" => "120", "district" => "Centro", "city" => "Curitiba",
              "uf" => "PR", "zip" => "80010000" }.freeze
  # CPF mandado na entrada por engano: o api ignora (vale o do cadastro).
  PATIENT_CPF = "52998224725".freeze
  FAKE_SNCR_BASE = "http://fake-sncr.test/api/v1".freeze
  SNCR_REAL_BASE = "https://sncr-gateway.test/api-sncr/api/v1".freeze
  SNCR_REAL_AUTH = "https://sncr-auth.test/api/v1".freeze
  MAINTAINER_CNPJ = "11222333000181".freeze

  def controlled_city!(mock: true, signature: true)
    city = documents_city!
    maintainer = ledi_maintainer!
    Platform::Features.set!(city: city, key: "digital_signature", enabled: true, maintainer: maintainer) if signature
    Platform::Features.set!(city: city, key: "controlled_prescriptions", enabled: true, maintainer: maintainer)
    Platform::Features.set!(city: city, key: "sncr_mock", enabled: true, maintainer: maintainer) if mock
    CityCatalog.reset_cache!
    city
  end

  def fake_sncr = @fake_sncr ||= FakeSncr::App.new(start_block: 1)

  def stub_sncr_mock!
    allow(ENV).to receive(:[]).and_call_original
    allow(ENV).to receive(:[]).with("FAKE_SNCR_URL").and_return(FAKE_SNCR_BASE)
    allow(ENV).to receive(:[]).with("FAKE_SNCR_PUBLIC_URL").and_return("http://localhost:8092/api/v1")
    stub_request(:any, /\A#{Regexp.escape(FAKE_SNCR_BASE)}/).to_rack(fake_sncr)
  end

  def stub_sncr_real!
    allow(Sncr::Config).to receive(:credentials)
      .and_return("base_url" => SNCR_REAL_BASE, "auth_url" => SNCR_REAL_AUTH, "maintainer_cnpj" => MAINTAINER_CNPJ)
    [ SNCR_REAL_BASE, SNCR_REAL_AUTH ].each { |base| stub_request(:any, /\A#{Regexp.escape(base)}/).to_rack(fake_sncr) }
  end

  # Estoque direto (sem SNCR): um lote de `count` números a partir do próximo livre.
  def sncr_stock!(user, kind: "rce", count: 3, simulated: true)
    prefix = "2610.#{kind == 'rce' ? 1 : 2}-41."
    start = SncrNumber.where(kind: kind).where("number LIKE ?", "#{prefix}%").count + 1
    block = Sncr::Client::Block.new(first: "#{prefix}#{start.to_s.rjust(7, '0')}",
                                    last: "#{prefix}#{(start + count - 1).to_s.rjust(7, '0')}", quantity: count)
    Sncr::Stock.record_batch!(user: user, kind: kind, block: block, simulated: simulated)
  end

  # Item do catálogo da plataforma com lista/marcas fixadas (códigos 9_900_xxx não existem no CATMAT).
  # O banco de plataforma é compartilhado e uma reaplicação de marcas
  # (apply_to_catalog!) pode ter mexido no item: as marcas são regravadas a cada chamada.
  def controlled_item!(code:, ingredient:, list: nil, antimicrobial: false, anticonvulsant: false)
    flags = { "antimicrobial" => antimicrobial, "controlled" => !list.nil?, "controlled_list" => list, "anticonvulsant" => anticonvulsant }
    existing = MedicationCatalogItem.find_by(catmat_code: code)
    return existing.tap { |item| item.update!(flags) } if existing

    base = import_test_catalog!.values.first
    MedicationCatalogItem.create!(
      base.attributes.except("id", "created_at", "updated_at").merge(flags).merge(
        "catmat_code" => code, "source_description" => "#{ingredient}, DOSAGEM: 200 MG", "active_ingredient" => ingredient,
        "strength" => "200 MG", "dosage_form" => "comprimido", "unit" => "mg", "hidden" => false,
        "parse_status" => "ok", "review_reason" => nil, "status" => "active", "search_text" => ingredient.downcase
      )
    )
  end

  def prescriber_contact!(user, phone: "41999990000")
    user.professional.update!(prescriber_address: ADDRESS.to_json, phone: phone)
    user
  end

  def patient_identification(**over)
    # Entrada (contrato §2 do 19d): só o endereço; o CPF é o do cadastro, preenchido pelo api.
    { "address" => ADDRESS }.merge(over.stringify_keys)
  end
end

RSpec.configure { |config| config.include ControlledPrescriptionHelpers }
```

Em `spec/rails_helper.rb`: `require_relative "support/controlled_prescription_helpers"` (junto dos outros `support/`). O `lib/fake_sncr/app.rb` nasce na Task 6: até lá, crie o arquivo com só `module FakeSncr; end` para o `require_relative` não falhar (a Task 6 o completa).

- [ ] **Step 6: Migre os bancos de teste da branch**

Run: `psql -U rota_saude -d postgres -c "DROP DATABASE IF EXISTS rota_saude_test_city_a_mod19d" -c "DROP DATABASE IF EXISTS rota_saude_test_city_b_mod19d" && docker compose exec -T -e RAILS_ENV=test -e ROTA_TEST_DB_SUFFIX=_mod19d -w /rails/.claude/mod19d api bin/rails city:test_databases`
Expected: os dois bancos recriados com a migração nova.

- [ ] **Step 7: Rode e veja passar**

Run: `rspec spec/models/sncr_tables_guard_spec.rb spec/initializers/domain_events_bindings_spec.rb spec/services/city_schema_spec.rb spec/architecture/city_encrypted_attributes_guard_spec.rb spec/models/clinical_document_tables_guard_spec.rb spec/models/signature_tables_guard_spec.rb spec/services/medications/anticonvulsant_marks_spec.rb`
Expected: PASS (a última passa ao fim da Task 8, que cria `Sncr::Stock` — só `sncr_stock!` depende dela; se falhar só por isso, siga).

- [ ] **Step 8: Commit**

```bash
git add db/city_migrate/20261010700001_add_controlled_prescriptions.rb db/city_schema.rb db/city_triggers.sql app/models/sncr_number_batch.rb app/models/sncr_number.rb app/models/professional.rb app/models/clinical_document.rb app/models/signature_oauth_state.rb app/services/city_encryption.rb config/initializers/domain_events.rb config/initializers/filter_parameter_logging.rb spec/initializers/domain_events_bindings_spec.rb spec/support/controlled_prescription_helpers.rb spec/rails_helper.rb spec/models/sncr_tables_guard_spec.rb lib/fake_sncr/app.rb
git commit -m "feat: add the SNCR number stock tables, prescriber address and notification record kind

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Credenciais do SNCR e checagem

**Files:**
- Create: `app/services/sncr/config.rb`, `app/services/sncr/checks.rb`
- Test: `spec/services/sncr/config_spec.rb`

**Interfaces:**
- Consumes: `Sncr::Mock` (Task 1), `SncrCheck` (Task 2), `Cnpj.normalize` (19c), `Platform::Features.find`.
- Produces:
  - `Sncr::Config::Settings = Data.define(:base_url, :auth_url, :public_auth_url, :cnpj, :simulated)` (`inspect` sem URL nem CNPJ).
  - `Sncr::Config.credentials -> Hash` (`Rails.application.credentials.sncr`), `.real -> Settings|nil` (as três chaves `base_url`, `auth_url`, `maintainer_cnpj`; URL `https` fora de dev/test; CNPJ com DV), `.simulated(env: Rails.env) -> Settings|nil` (só com `sncr_mock` no catálogo do ambiente e `FAKE_SNCR_URL`), `.for_city(city = Current.city, env: Rails.env) -> Settings|nil`, `.configured? -> bool`, `.simulated_available?(env: Rails.env) -> bool`; `Sncr::Config::SIMULATED_CNPJ == "11222333000181"`.
  - `Sncr::Checks.record!(settings, ok:)`, `.status -> { configured:, reachable:, last_check_at:, simulated_available: }`.

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/services/sncr/config_spec.rb
require "rails_helper"

# ADR 0034 (spec §3; Desvio 2): credenciais da PLATAFORMA por ambiente —
# sncr.{base_url, auth_url, maintainer_cnpj} (o SNCR não tem client_id nem
# secret). O simulado só fora de produção, com FAKE_SNCR_URL. Nunca URL ou
# CNPJ em inspect.
RSpec.describe Sncr::Config do
  let(:creds) do
    { "base_url" => "https://api-gateway.prd.apps.anvisa.gov.br/api-sncr/api/v1/", "auth_url" => "https://sncr-api.apps.anvisa.gov.br/api/v1",
      "maintainer_cnpj" => "11.222.333/0001-81" }
  end

  it "real: as três chaves, sem barra final, CNPJ normalizado" do
    allow(described_class).to receive(:credentials).and_return(creds)
    real = described_class.real
    expect(real).to have_attributes(base_url: "https://api-gateway.prd.apps.anvisa.gov.br/api-sncr/api/v1",
                                    auth_url: "https://sncr-api.apps.anvisa.gov.br/api/v1",
                                    public_auth_url: "https://sncr-api.apps.anvisa.gov.br/api/v1",
                                    cnpj: "11222333000181", simulated: false)
    expect(real.inspect).not_to include("anvisa", "11222333000181")
    expect(described_class.configured?).to be(true)
  end

  it "faltando chave, CNPJ inválido ou http em produção: não configurado" do
    allow(described_class).to receive(:credentials).and_return(creds.except("auth_url"))
    expect(described_class.real).to be_nil
    allow(described_class).to receive(:credentials).and_return(creds.merge("maintainer_cnpj" => "11222333000182"))
    expect(described_class.real).to be_nil
    allow(described_class).to receive(:credentials).and_return(creds.merge("base_url" => "http://x.gov.br"))
    expect(described_class.real(env: "production")).to be_nil
    allow(described_class).to receive(:credentials).and_return({})
    expect(described_class.configured?).to be(false)
  end

  it "simulado só com o interruptor no catálogo do ambiente e FAKE_SNCR_URL; a cidade com sncr_mock usa só ele" do
    stub_sncr_mock!
    expect(described_class.simulated).to have_attributes(base_url: ControlledPrescriptionHelpers::FAKE_SNCR_BASE,
                                                         public_auth_url: "http://localhost:8092/api/v1",
                                                         cnpj: described_class::SIMULATED_CNPJ, simulated: true)
    expect(described_class.simulated(env: "production")).to be_nil
    expect(described_class.simulated_available?).to be(true)

    allow(described_class).to receive(:credentials).and_return(creds)
    city = controlled_city!(mock: true)
    expect(described_class.for_city(city).simulated).to be(true)
    Platform::Features.set!(city: city, key: "sncr_mock", enabled: false, maintainer: ledi_maintainer!)
    expect(described_class.for_city(city).simulated).to be(false)
  end

  it "Checks.status: configurado, alcançável pela última conversa real, simulado disponível" do
    allow(described_class).to receive(:credentials).and_return(creds)
    SncrCheck.where(target: "real").delete_all
    expect(Sncr::Checks.status).to include(configured: true, reachable: false, last_check_at: nil)
    Sncr::Checks.record!(described_class.real, ok: true)
    expect(Sncr::Checks.status).to include(reachable: true, last_check_at: be_present)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `rspec spec/services/sncr/config_spec.rb`
Expected: FAIL (`uninitialized constant Sncr::Config`).

- [ ] **Step 3: Implemente**

```ruby
# app/services/sncr/config.rb
# Credenciais do SNCR (ADR 0034; spec §3; Desvio 2). Da PLATAFORMA, por
# ambiente, nas credenciais cifradas do Rails:
#   sncr.{base_url, auth_url, maintainer_cnpj}
# base_url = .../api-sncr/api/v1 (numeracoes/*); auth_url = base do /auth
# (login e token; o host conferido na Task 0). O SNCR não tem client_id nem
# secret. O simulado existe só fora de produção, com FAKE_SNCR_URL (o que o
# api usa) e FAKE_SNCR_PUBLIC_URL (o que o navegador abre). Nenhuma URL ou
# CNPJ em inspect, log ou erro. Lê Current.city, nunca a atribui.
module Sncr
  module Config
    REQUIRED = %w[base_url auth_url maintainer_cnpj].freeze
    SIMULATED_CNPJ = "11222333000181".freeze

    Settings = Data.define(:base_url, :auth_url, :public_auth_url, :cnpj, :simulated) do
      def inspect = "#<Sncr::Config::Settings #{simulated ? 'simulated' : 'real'}>"
      alias_method :to_s, :inspect
      def pretty_print(pp) = pp.text(inspect)
      def target = simulated ? "simulated" : "real"
    end

    module_function

    def credentials = Rails.application.credentials.dig(:sncr).to_h.deep_stringify_keys

    def real(env: Rails.env)
      raw = credentials
      return nil unless REQUIRED.all? { |key| raw[key].present? }

      base, auth = raw.values_at("base_url", "auth_url").map { |url| url.to_s.chomp("/") }
      return nil unless [ base, auth ].all? { |url| url_ok?(url, env) }

      cnpj = Cnpj.normalize(raw["maintainer_cnpj"])
      return nil unless cnpj

      Settings.new(base_url: base, auth_url: auth, public_auth_url: auth, cnpj: cnpj, simulated: false)
    end

    def simulated(env: Rails.env)
      return nil unless Platform::Features.find(Mock::KEY, env: env)

      base = ENV["FAKE_SNCR_URL"].presence&.chomp("/")
      return nil unless base

      Settings.new(base_url: base, auth_url: base, public_auth_url: ENV["FAKE_SNCR_PUBLIC_URL"].presence&.chomp("/") || base,
                   cnpj: SIMULATED_CNPJ, simulated: true)
    end

    def for_city(city = Current.city, env: Rails.env) = Mock.on?(city, env: env) ? simulated(env: env) : real(env: env)
    def configured? = !real.nil?
    def simulated_available?(env: Rails.env) = !simulated(env: env).nil?

    def url_ok?(url, env)
      uri = URI.parse(url)
      uri.host.present? && (uri.scheme == "https" || (%w[development test].include?(env.to_s) && uri.scheme == "http"))
    rescue URI::InvalidURIError
      false
    end
    private_class_method :url_ok?
  end
end
```

```ruby
# app/services/sncr/checks.rb
# Estado do SNCR para o maintenance (contrato §8): configurado no ambiente,
# alcançável pela última conversa REAL, e se o simulado existe aqui.
module Sncr
  module Checks
    module_function

    def record!(settings, ok:) = settings && SncrCheck.record!(settings.target, ok: ok)

    def status
      check = SncrCheck.last("real")
      { configured: Config.configured?, reachable: check&.last_check_ok || false, last_check_at: check&.last_check_at,
        simulated_available: Config.simulated_available? }
    end
  end
end
```

- [ ] **Step 4: Rode e veja passar**

Run: `rspec spec/services/sncr/config_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add app/services/sncr/config.rb app/services/sncr/checks.rb spec/services/sncr/config_spec.rb
git commit -m "feat: read the SNCR platform settings and record its last check

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
### Task 6: SNCR simulado (`FakeSncr::App`) e o serviço `fake-sncr` do compose

**Files:**
- Create: `lib/fake_sncr/app.rb` (completa o esqueleto da Task 4), `lib/fake_sncr/config.ru`
- Modify: `docker-compose.yml` da raiz do monorepo (fora de git; sem commit)
- Test: `spec/lib/fake_sncr_spec.rb`

**Interfaces:**
- Consumes: nada do Rails (roda sozinho no puma, como o `fake-psc`).
- Produces: `FakeSncr::App.new(start_block: nil, clock: -> { Time.now })`, Rack app com `GET …/auth/login`, `GET …/auth/login/decision`, `GET …/auth/token`, `POST …/numeracoes/receita-especial-retencao` (qualquer prefixo antes de `/auth` ou `/numeracoes`); ganchos `approve!(client_url) -> session_id`, `issued_tokens -> Array<String>`, `repeat_last_block!`, `log`, `failures` (`:refused` ou status HTTP), `exhausted_ufs`, `mismatch_documents`, `clock=`; constantes `MESSAGES`, `UF_CODES`, `TOKEN_TTL = 30`, `SESSION_TTL = 30`, `MONTHLY_LIMIT = 3`, `BLOCK = 1000`.

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/lib/fake_sncr_spec.rb
require "rails_helper"

# O SNCR simulado imita a API real (Manual 3ª ed.; Desvio 1): login com
# página "SNCR SIMULADO", volta com session_id de uso único e 30 s, token
# preso ao Origin, token de 30 s, bloco de 1.000 com 201, 3 pedidos por mês
# por inscrição e tipo, esgotamento, inscrição divergente, CNPJ inválido.
RSpec.describe FakeSncr::App do
  let(:now) { Time.utc(2026, 10, 10, 12) }
  let(:app) { described_class.new(start_block: 1, clock: -> { now }) }
  let(:mock) { Rack::MockRequest.new(app) }
  let(:client_url) { "http://curitiba.localhost:5189/dashboard/sncr/callback" }
  let(:origin) { "http://curitiba.localhost:5189" }
  let(:inscricao) { { "conselho" => "CRM", "documento" => "12345", "uf" => "PR", "cnpj" => "11222333000181" } }

  def token_for(session_id, origin: self.origin) = mock.get("/api/v1/auth/token?session_id=#{session_id}", "HTTP_ORIGIN" => origin)

  def bearer
    JSON.parse(token_for(app.approve!(client_url)).body).fetch("access_token")
  end

  def block(tipo: "RCE", token: bearer, body: inscricao)
    mock.post("/api/v1/numeracoes/receita-especial-retencao", input: JSON.generate(body.merge("tipo" => tipo)),
                                                               "CONTENT_TYPE" => "application/json", "HTTP_AUTHORIZATION" => "Bearer #{token}")
  end

  it "login mostra a página do simulado e volta com session_id; cancelar volta com error" do
    page = mock.get("/api/v1/auth/login?client_url=#{CGI.escape(client_url)}&state=abc")
    expect(page.status).to eq(200)
    expect(page.body).to include("SNCR SIMULADO — desenvolvimento", "numeração simulada")
    approve = page.body[%r{href="([^"]+approve=1[^"]*)"}, 1]
    back = mock.get(CGI.unescapeHTML(approve))
    expect(back.status).to eq(302)
    expect(back.location).to match(%r{\A#{Regexp.escape(client_url)}\?session_id=[\w-]+\z})
    deny = mock.get(CGI.unescapeHTML(mock.get("/api/v1/auth/login?client_url=#{CGI.escape(client_url)}").body[%r{href="([^"]+approve=0[^"]*)"}, 1]))
    expect(deny.location).to eq("#{client_url}?error=access_denied")
    expect(mock.get("/api/v1/auth/login?client_url=#{CGI.escape('https://evil.example.com/cb')}").status).to eq(400)
  end

  it "session_id: uso único, 30 s, preso ao Origin" do
    session_id = app.approve!(client_url)
    expect(token_for(session_id, origin: "http://outra.localhost:5189").status).to eq(403)
    session_id = app.approve!(client_url)
    response = token_for(session_id)
    expect(response.status).to eq(200)
    expect(JSON.parse(response.body)).to include("token_type" => "Bearer", "access_token" => be_a(String))
    expect(JSON.parse(token_for(session_id).body)).to eq("error" => "Sessão inválida ou expirada")
    late = app.approve!(client_url)
    app.clock = -> { now + 31 }
    expect(token_for(late).status).to eq(400)
  end

  it "bloco de 1.000 com 201; token de 30 s" do
    token = bearer
    response = block(token: token)
    expect(response.status).to eq(201)
    body = JSON.parse(response.body)
    expect(body).to eq("inicio" => "2610.1-41.0001001", "fim" => "2610.1-41.0002000", "quantidade" => 1000,
                       "mensagem" => "Numeração gerada com sucesso.")
    expect(JSON.parse(block(tipo: "RET").body)["inicio"]).to eq("2610.2-41.0001001")
    app.clock = -> { now + 31 }
    expect(block(token: token).status).to eq(401)
  end

  it "3 pedidos por mês por inscrição e tipo; esgotamento; inscrição divergente; validação" do
    3.times { expect(block.status).to eq(201) }
    limit = block
    expect([ limit.status, JSON.parse(limit.body)["mensagem"] ])
      .to eq([ 400, "Usuário atingiu o limite máximo de receita para o tipo de receita no mês atual." ])
    expect(block(tipo: "RET").status).to eq(201)
    app.exhausted_ufs << "SP"
    expect(JSON.parse(block(body: inscricao.merge("uf" => "SP")).body)["mensagem"]).to match(/esgotada/)
    app.mismatch_documents << "999"
    expect(block(body: inscricao.merge("documento" => "999")).status).to eq(404)
    expect(JSON.parse(block(body: inscricao.merge("cnpj" => "11222333000182")).body)["mensagem"]).to eq("CNPJ inválido.")
    expect(JSON.parse(block(body: inscricao.merge("conselho" => "COREN")).body)["mensagem"]).to eq("Conselho inválido")
  end

  it "repeat_last_block! devolve de novo a última faixa (incidente de 09/10); failures simulam queda" do
    first = JSON.parse(block.body)["inicio"]
    app.repeat_last_block!
    expect(JSON.parse(block.body)["inicio"]).to eq(first)
    app.failures << 503
    expect(block.status).to eq(503)
    app.failures << :refused
    expect { block }.to raise_error(Errno::ECONNREFUSED)
  end

  it "o config.ru sobe fora de produção" do
    expect(Rack::Builder.parse_file(Rails.root.join("lib/fake_sncr/config.ru").to_s)).to be_present
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `rspec spec/lib/fake_sncr_spec.rb`
Expected: FAIL (`undefined method 'approve!'`).

- [ ] **Step 3: Implemente**

```ruby
# lib/fake_sncr/app.rb
# SNCR SIMULADO (ADR 0034; Manual da API SNCR 3ª ed.): login do prescritor
# (página que faz as vezes do gov.br), session_id de uso único e 30 s preso
# ao Origin do client_url, token de 30 s sem refresh, bloco de 1.000 números
# de RCE/RET com 201, 3 pedidos por mês por inscrição e tipo, esgotamento por
# UF e inscrição divergente. Usado pelas specs (WebMock#to_rack) e pelo
# serviço fake-sncr do compose de dev (config.ru). Nunca em produção. Sem
# ActiveSupport: roda sozinho no puma.
require "json"
require "securerandom"
require "uri"
require "cgi"
require "rack"

module FakeSncr
  class App
    TOKEN_TTL = 30
    SESSION_TTL = 30
    MONTHLY_LIMIT = 3
    BLOCK = 1000
    CONSELHOS = %w[CRM CRMV CRO].freeze
    TIPOS = { "RCE" => 1, "RET" => 2 }.freeze
    UF_CODES = { "RO" => 11, "AC" => 12, "AM" => 13, "RR" => 14, "PA" => 15, "AP" => 16, "TO" => 17, "MA" => 21, "PI" => 22,
                 "CE" => 23, "RN" => 24, "PB" => 25, "PE" => 26, "AL" => 27, "SE" => 28, "BA" => 29, "MG" => 31, "ES" => 32,
                 "RJ" => 33, "SP" => 35, "PR" => 41, "SC" => 42, "RS" => 43, "MS" => 50, "MT" => 51, "GO" => 52, "DF" => 53 }.freeze
    MESSAGES = {
      ok: "Numeração gerada com sucesso.", session: "Sessão inválida ou expirada", domain: "Domínio não autorizado",
      unauthorized: "Não autorizado.", limit: "Usuário atingiu o limite máximo de receita para o tipo de receita no mês atual.",
      exhausted: "Faixa de numeração esgotada para a UF.", mismatch: "Inscrição fornecida é diferente da autenticada.",
      conselho_nulo: "Conselho não pode ser nulo.", conselho: "Conselho inválido", tipo_nulo: "Tipo de receita não pode ser nulo.",
      tipo: "Tipo da receita inválido. Valores permitidos: [RCE, RET]", uf_nulo: "UF não pode ser nulo.",
      documento_nulo: "Documento não pode ser nulo.", cnpj_nulo: "CNPJ não pode ser nulo.", cnpj: "CNPJ inválido."
    }.freeze

    attr_reader :log
    attr_accessor :clock, :failures, :exhausted_ufs, :mismatch_documents

    def initialize(start_block: nil, clock: -> { Time.now })
      @start_block = start_block
      @clock = clock
      @mutex = Mutex.new
      reset!
    end

    def reset!
      @mutex.synchronize do
        @pending = {}
        @sessions = {}
        @tokens = {}
        @requests = Hash.new(0)
        @blocks = {}
        @last_block = nil
        @log = []
      end
      @failures = []
      @exhausted_ufs = []
      @mismatch_documents = []
    end

    # — Ganchos de teste (a pessoa passando pelo gov.br e o estado do SNCR) —

    def approve!(client_url)
      origin = origin_of(client_url) or raise ArgumentError, "client_url fora dos domínios aceitos"
      new_session(origin)
    end

    def issued_tokens = @mutex.synchronize { @tokens.keys }
    def repeat_last_block! = @mutex.synchronize { @blocks[@last_block] -= 1 if @last_block }

    def call(env)
      request = Rack::Request.new(env)
      failure = @mutex.synchronize do
        @log << [ request.request_method, request.path ]
        @failures.shift
      end
      raise Errno::ECONNREFUSED, "fake-sncr" if failure == :refused
      return json(failure, { mensagem: "Erro interno do servidor." }) if failure.is_a?(Integer)

      route(request)
    end

    def inspect = "#<FakeSncr::App>"

    private

    def now = @clock.call

    def route(request)
      path = request.path.sub(%r{\A.*?/(auth|numeracoes)/}, '/\1/')
      case [ request.request_method, path ]
      in [ "GET", "/auth/login" ] then login(request)
      in [ "GET", "/auth/login/decision" ] then decision(request)
      in [ "GET", "/auth/token" ] then token(request)
      in [ "POST", "/numeracoes/receita-especial-retencao" ] then numbers(request)
      else json(404, { mensagem: "Página não foi encontrada." })
      end
    end

    def login(request)
      client_url = request.params["client_url"].to_s
      origin = origin_of(client_url)
      return html(400, "<p>#{MESSAGES[:domain]}</p>") unless origin

      id = SecureRandom.hex(8)
      @mutex.synchronize { @pending[id] = { client_url: client_url, origin: origin } }
      decision = "#{request.script_name}#{request.path}/decision?id=#{id}"
      html(200, <<~HTML)
        <h1>SNCR SIMULADO — desenvolvimento</h1>
        <p>Este é o SNCR simulado do ambiente de desenvolvimento. Não há gov.br aqui: continue como o profissional
        que pediu os números. Os números entregues são de <strong>numeração simulada — sem validade</strong>.</p>
        <p><a href="#{CGI.escapeHTML("#{decision}&approve=1")}">Entrar (simulado)</a> ·
        <a href="#{CGI.escapeHTML("#{decision}&approve=0")}">Cancelar</a></p>
      HTML
    end

    def decision(request)
      pending = @mutex.synchronize { @pending.delete(request.params["id"].to_s) }
      return html(400, "<p>#{MESSAGES[:session]}</p>") unless pending

      separator = pending[:client_url].include?("?") ? "&" : "?"
      location = if request.params["approve"] == "1"
                   "#{pending[:client_url]}#{separator}session_id=#{new_session(pending[:origin])}"
                 else
                   "#{pending[:client_url]}#{separator}error=access_denied"
                 end
      [ 302, { "location" => location }, [] ]
    end

    def token(request)
      session = @mutex.synchronize { @sessions.delete(request.params["session_id"].to_s) }
      return json(400, { error: MESSAGES[:session] }) unless session && session[:expires_at] > now
      return json(403, { error: MESSAGES[:domain] }) unless request.get_header("HTTP_ORIGIN") == session[:origin]

      token = SecureRandom.urlsafe_base64(32)
      @mutex.synchronize { @tokens[token] = now + TOKEN_TTL }
      json(200, { access_token: token, token_type: "Bearer" })
    end

    def numbers(request)
      token = request.get_header("HTTP_AUTHORIZATION").to_s.delete_prefix("Bearer ")
      expires_at = @mutex.synchronize { @tokens[token] }
      return json(401, { mensagem: MESSAGES[:unauthorized] }) unless expires_at && expires_at > now

      body = parse(request)
      error = validation_error(body)
      return json(400, { mensagem: error }) if error
      return json(404, { mensagem: MESSAGES[:mismatch] }) if @mismatch_documents.include?(body["documento"])
      return json(400, { mensagem: MESSAGES[:exhausted] }) if @exhausted_ufs.include?(body["uf"])

      first, last = @mutex.synchronize do
        key = [ body["conselho"], body["documento"], body["uf"], body["tipo"], now.strftime("%Y%m") ]
        next nil if @requests[key] >= MONTHLY_LIMIT

        @requests[key] += 1
        range(body["tipo"], body["uf"])
      end
      return json(400, { mensagem: MESSAGES[:limit] }) unless first

      json(201, { inicio: first, fim: last, quantidade: BLOCK, mensagem: MESSAGES[:ok] })
    end

    # Chamado com o mutex tomado.
    def range(tipo, uf)
      key = [ tipo, uf ]
      @blocks[key] = (@start_block || rand(100..8_999)) unless @blocks.key?(key)
      index = @blocks[key]
      @blocks[key] += 1
      @last_block = key
      prefix = "#{now.strftime('%y%m')}.#{TIPOS.fetch(tipo)}-#{UF_CODES.fetch(uf)}."
      [ "#{prefix}#{format('%07d', (index * BLOCK) + 1)}", "#{prefix}#{format('%07d', (index + 1) * BLOCK)}" ]
    end

    def validation_error(body)
      return MESSAGES[:conselho_nulo] if body["conselho"].to_s.empty?
      return MESSAGES[:conselho] unless CONSELHOS.include?(body["conselho"])
      return MESSAGES[:tipo_nulo] if body["tipo"].to_s.empty?
      return MESSAGES[:tipo] unless TIPOS.key?(body["tipo"])
      return MESSAGES[:uf_nulo] if body["uf"].to_s.empty?
      return "UF não encontrada: #{body['uf']}" unless UF_CODES.key?(body["uf"])
      return MESSAGES[:documento_nulo] if body["documento"].to_s.empty?
      return MESSAGES[:cnpj_nulo] if body["cnpj"].to_s.empty?

      MESSAGES[:cnpj] unless cnpj?(body["cnpj"].to_s)
    end

    # DV do CNPJ (numérico e alfanumérico, IN RFB 2.229/2024: valor = código ASCII − 48).
    def cnpj?(value)
      return false unless value.match?(/\A[0-9A-Z]{12}[0-9]{2}\z/)

      digits = value.chars.map { |char| char.ord - 48 }
      [ 12, 13 ].all? do |size|
        weights = (2..9).cycle.first(size).reverse
        sum = digits.first(size).zip(weights).sum { |digit, weight| digit * weight }
        check = sum % 11 < 2 ? 0 : 11 - (sum % 11)
        check == digits[size]
      end
    end

    def new_session(origin)
      id = SecureRandom.urlsafe_base64(24)
      @mutex.synchronize { @sessions[id] = { origin: origin, expires_at: now + SESSION_TTL } }
      id
    end

    # O SNCR aceita *.br (e localhost/127.0.0.1 em dev); o simulado aceita também *.localhost.
    def origin_of(url)
      uri = URI.parse(url)
      return nil unless %w[http https].include?(uri.scheme) && uri.host
      return nil unless uri.host.end_with?(".br") || uri.host == "localhost" || uri.host.end_with?(".localhost") || uri.host == "127.0.0.1"

      default = (uri.scheme == "https" && uri.port == 443) || (uri.scheme == "http" && uri.port == 80)
      "#{uri.scheme}://#{uri.host}#{default ? '' : ":#{uri.port}"}"
    rescue URI::InvalidURIError
      nil
    end

    def parse(request)
      parsed = JSON.parse(request.body.read.to_s)
      parsed.is_a?(Hash) ? parsed : {}
    rescue JSON::ParserError
      {}
    end

    def json(status, body) = [ status, { "content-type" => "application/json; charset=utf-8" }, [ JSON.generate(body) ] ]

    def html(status, body)
      page = "<!doctype html><html lang=\"pt-BR\"><head><meta charset=\"utf-8\"><title>SNCR SIMULADO — desenvolvimento</title></head>" \
             "<body style=\"font-family:system-ui;max-width:40rem;margin:2rem auto\">#{body}</body></html>"
      [ status, { "content-type" => "text/html; charset=utf-8" }, [ page ] ]
    end
  end
end
```

```ruby
# lib/fake_sncr/config.ru
# SNCR SIMULADO do compose de dev (ADR 0034). Nunca em produção.
# FAKE_SNCR_EXHAUSTED_UFS e FAKE_SNCR_MISMATCH_DOCUMENTS (vírgulas) simulam
# faixa esgotada e inscrição divergente.
require_relative "app"

abort "fake-sncr nunca sobe em produção" if (ENV["RAILS_ENV"] || ENV["RACK_ENV"]).to_s == "production"

app = FakeSncr::App.new
app.exhausted_ufs = ENV.fetch("FAKE_SNCR_EXHAUSTED_UFS", "").split(",").map(&:strip).reject(&:empty?)
app.mismatch_documents = ENV.fetch("FAKE_SNCR_MISMATCH_DOCUMENTS", "").split(",").map(&:strip).reject(&:empty?)
run app
```

> `repeat_last_block!` decrementa o índice do último `(tipo, UF)`: o pedido seguinte devolve a mesma faixa. O teste do esgotamento usa SP para não gastar os pedidos de PR do mesmo teste.

- [ ] **Step 4: Rode e veja passar**

Run: `rspec spec/lib/fake_sncr_spec.rb`
Expected: PASS.

- [ ] **Step 5: O serviço no compose de dev** (arquivo da raiz, fora de git — sem commit; avise no balanço)

Em `docker-compose.yml`, no bloco `x-api-env`, depois de `FAKE_PSC_PUBLIC_URL`:

```yaml
  # SNCR simulado de dev (serviço fake-sncr, ADR 0034): endereço que o api usa e o que o navegador abre.
  FAKE_SNCR_URL: ${FAKE_SNCR_URL:-http://fake-sncr:8092/api/v1}
  FAKE_SNCR_PUBLIC_URL: ${FAKE_SNCR_PUBLIC_URL:-http://localhost:8092/api/v1}
```
e o serviço, depois de `fake-psc`:

```yaml
  fake-sncr:
    container_name: fake-sncr-dev
    # SNCR simulado (ADR 0034, módulo 19d): o mesmo FakeSncr::App das specs. Página "SNCR SIMULADO — desenvolvimento".
    image: rota-saude/api:dev
    working_dir: /rails
    command: bundle exec puma -b tcp://0.0.0.0:8092 lib/fake_sncr/config.ru
    environment:
      FAKE_SNCR_EXHAUSTED_UFS: ${FAKE_SNCR_EXHAUSTED_UFS:-}
      FAKE_SNCR_MISMATCH_DOCUMENTS: ${FAKE_SNCR_MISMATCH_DOCUMENTS:-}
    volumes:
      - ./apps/api:/rails
    ports:
      - "127.0.0.1:8092:8092"
    restart: unless-stopped
```
O serviço lê `./apps/api` (o checkout principal): enquanto a branch não estiver na main, suba o falso do worktree com `docker compose run -d --name fake-sncr-mod19d -p 127.0.0.1:8092:8092 -w /rails/.claude/mod19d api bundle exec puma -b tcp://0.0.0.0:8092 lib/fake_sncr/config.ru` e aponte `FAKE_SNCR_URL=http://fake-sncr-mod19d:8092/api/v1` no api da porta 3039 (Task 22).

Run: `docker compose config --services | grep fake-sncr`
Expected: `fake-sncr`.

- [ ] **Step 6: Commit**

```bash
git add lib/fake_sncr/app.rb lib/fake_sncr/config.ru spec/lib/fake_sncr_spec.rb
git commit -m "feat: add a simulated SNCR that mimics the 30-second token and the monthly limit

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: `Sncr::Client` — login, troca do `session_id` no servidor e pedido do bloco

**Files:**
- Create: `app/services/sncr/numbers.rb`, `app/services/sncr/client.rb`
- Test: `spec/services/sncr/client_spec.rb`

**Interfaces:**
- Consumes: `Sncr::Config` (`Settings`, `.for_city`), `Sncr::Checks.record!` (Task 5), `FakeSncr::App` (Task 6).
- Produces:
  - `Sncr::Numbers.expand(first, last, quantity) -> Array<String>|nil` (mesmo prefixo, 1 a 1.000, tamanho = `quantity`).
  - `Sncr::Client.for(city = Current.city, env: Rails.env) -> Client` (levanta `Unavailable` sem configuração); `#simulated?`, `#provider` (`"sncr"` | `"sncr_simulated"`), `#settings`, `#login_url(client_url:, state:) -> String`, `#exchange(session_id:, origin:) -> Token`, `#request_block(token:, kind:, council:, registration:, uf:) -> Block`.
  - `Sncr::Client::Token = Data.define(:access_token)` e `Block = Data.define(:first, :last, :quantity)` (`inspect` sem token nem número); erros `Sncr::Client::{Error, Unavailable, Expired, LimitReached, Exhausted, RegistrationMismatch}`; `KINDS = { "rce" => "RCE", "ret" => "RET" }`, `COUNCILS = %w[CRM CRMV CRO]`, `SEND_ORIGIN` (resultado da Task 0), `TIMEOUT = 8`.

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/services/sncr/client_spec.rb
require "rails_helper"

# ADR 0034 (spec §3; Desvios 1–3): o api monta o login, troca o session_id
# pelo token no servidor (Origin do host da cidade) e pede o bloco com os
# nomes reais do Manual; cada recusa vira um erro com nome; o token nunca
# aparece em inspect; toda chamada registra a checagem.
RSpec.describe Sncr::Client do
  before do
    controlled_city!(mock: false)
    stub_sncr_real!
  end

  let(:client) { described_class.for(Current.city) }
  let(:callback) { "#{CityPublicUrl.dashboard(Current.city)}sncr/callback" }
  let(:origin) { CityPublicUrl.base(Current.city) }

  def token = client.exchange(session_id: fake_sncr.approve!(callback), origin: origin)
  def block(kind: "rce", registration: "12345", uf: "PR", token: self.token)
    client.request_block(token: token, kind: kind, council: "CRM", registration: registration, uf: uf)
  end

  it "monta o login no auth público com client_url e state" do
    url = client.login_url(client_url: callback, state: "st-1")
    expect(url).to start_with("#{ControlledPrescriptionHelpers::SNCR_REAL_AUTH}/auth/login?")
    expect(URI.decode_www_form(URI(url).query).to_h).to eq("client_url" => callback, "state" => "st-1")
    expect(client).to have_attributes(simulated?: false, provider: "sncr")
  end

  it "troca o session_id no servidor e pede o bloco: 1.000 números no mesmo prefixo" do
    result = block
    expect(result.quantity).to eq(1000)
    numbers = Sncr::Numbers.expand(result.first, result.last, result.quantity)
    expect(numbers.size).to eq(1000)
    expect(numbers).to all(match(SncrNumber::FORMAT))
    expect(result.inspect).not_to include(result.first)
    expect(token.inspect).not_to include(*fake_sncr.issued_tokens)
    expect(SncrCheck.last("real")).to have_attributes(last_check_ok: true)
  end

  it "session_id vencido, repetido ou de outro Origin" do
    session_id = fake_sncr.approve!(callback)
    client.exchange(session_id: session_id, origin: origin)
    expect { client.exchange(session_id: session_id, origin: origin) }.to raise_error(described_class::Expired)
    expect { client.exchange(session_id: fake_sncr.approve!(callback), origin: "http://outra.localhost:5189") }
      .to raise_error(described_class::Unavailable)
    late = fake_sncr.approve!(callback)
    fake_sncr.clock = -> { Time.now + 31 }
    expect { client.exchange(session_id: late, origin: origin) }.to raise_error(described_class::Expired)
  end

  it "token vencido, limite do mês, esgotamento, inscrição divergente" do
    fresh = token
    fake_sncr.clock = -> { Time.now + 31 }
    expect { block(token: fresh) }.to raise_error(described_class::Expired)
    fake_sncr.clock = -> { Time.now }
    3.times { block }
    expect { block }.to raise_error(described_class::LimitReached)
    fake_sncr.exhausted_ufs << "SC"
    expect { block(uf: "SC") }.to raise_error(described_class::Exhausted)
    fake_sncr.mismatch_documents << "777"
    expect { block(registration: "777") }.to raise_error(described_class::RegistrationMismatch)
  end

  it "queda, recusa de conexão e resposta sem bloco válido: Unavailable, checagem falsa" do
    fake_sncr.failures << 503
    expect { token }.to raise_error(described_class::Unavailable)
    expect(SncrCheck.last("real")).to have_attributes(last_check_ok: false)
    fake_sncr.failures << :refused
    expect { token }.to raise_error(described_class::Unavailable)
    fresh = token
    stub_request(:post, "#{ControlledPrescriptionHelpers::SNCR_REAL_BASE}/numeracoes/receita-especial-retencao")
      .to_return(status: 201, body: { inicio: "2610.1-41.0000001", fim: "2610.2-41.0001000", quantidade: 1000 }.to_json)
    expect { block(token: fresh) }.to raise_error(described_class::Unavailable)
  end

  it "sem configuração: Unavailable; com sncr_mock, o simulado" do
    allow(Sncr::Config).to receive(:credentials).and_return({})
    expect { described_class.for(Current.city) }.to raise_error(described_class::Unavailable)
    stub_sncr_mock!
    Platform::Features.set!(city: Current.city, key: "sncr_mock", enabled: true, maintainer: ledi_maintainer!)
    expect(described_class.for(Current.city)).to have_attributes(simulated?: true, provider: "sncr_simulated")
  end

  it "Numbers.expand recusa prefixos diferentes, faixa invertida ou maior que 1.000" do
    expect(Sncr::Numbers.expand("2610.1-41.0000001", "2610.1-41.0000003", 3)).to eq(%w[2610.1-41.0000001 2610.1-41.0000002 2610.1-41.0000003])
    expect(Sncr::Numbers.expand("2610.1-41.0000001", "2610.2-41.0000003", 3)).to be_nil
    expect(Sncr::Numbers.expand("2610.1-41.0000005", "2610.1-41.0000001", 5)).to be_nil
    expect(Sncr::Numbers.expand("2610.1-41.0000001", "2610.1-41.0001001", 1001)).to be_nil
    expect(Sncr::Numbers.expand("2610.1-41.0000001", "2610.1-41.0000003", 1000)).to be_nil
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `rspec spec/services/sncr/client_spec.rb`
Expected: FAIL (`uninitialized constant Sncr::Client`).

- [ ] **Step 3: Implemente**

```ruby
# app/services/sncr/numbers.rb
# Número SNCR (Manual §2.3.2: "2602.6-53.0000001"): o bloco vem como início e
# fim no mesmo prefixo; o estoque guarda cada número.
module Sncr
  module Numbers
    PATTERN = /\A(\d{4}\.\d-\d{2}\.)(\d{7})\z/
    MAX = 1000

    module_function

    def expand(first, last, quantity)
      head = PATTERN.match(first.to_s)
      tail = PATTERN.match(last.to_s)
      return nil unless head && tail && head[1] == tail[1] && quantity.is_a?(Integer)

      from = head[2].to_i
      to = tail[2].to_i
      size = to - from + 1
      return nil unless size.between?(1, MAX) && size == quantity

      (from..to).map { |sequence| "#{head[1]}#{format('%07d', sequence)}" }
    end
  end
end
```

```ruby
# app/services/sncr/client.rb
# Cliente da API do SNCR (ADR 0034; Manual da API 3ª ed.; Desvios 1–3). O
# navegador faz o login (gov.br pela Anvisa) e volta com session_id; o api
# troca o session_id pelo token (GET /auth/token, cabeçalho Origin = host da
# cidade, prova da Task 0) e pede o bloco (POST
# numeracoes/receita-especial-retencao) na mesma requisição — o token de 30 s
# vive só nesta chamada. Nomes de caminho e de campo moram SÓ aqui e no
# FakeSncr::App. Toda chamada registra a checagem (Sncr::Checks). Nada de
# token, session_id, número ou inscrição em log, erro ou inspect.
require "net/http"

module Sncr
  class Client
    TIMEOUT = 8 # duas chamadas com folga dentro dos 30 s do token
    KINDS = { "rce" => "RCE", "ret" => "RET" }.freeze
    COUNCILS = %w[CRM CRMV CRO].freeze
    SEND_ORIGIN = true # Task 0: o session_id está preso ao Origin que iniciou o login
    PATHS = { login: "/auth/login", token: "/auth/token", block: "/numeracoes/receita-especial-retencao" }.freeze
    LIMIT = /limite m[aá]ximo de receita/i
    EXHAUSTED = /esgot/i
    REGISTRATION = /Inscri[cç][aã]o fornecida|n[aã]o possui v[ií]nculo|Prescritor n[aã]o encontrado|Usu[aá]rio autenticado n[aã]o encontrado/i
    NETWORK_ERRORS = [ SocketError, IOError, EOFError, SystemCallError, Net::OpenTimeout, Net::ReadTimeout,
                       Net::WriteTimeout, Net::ProtocolError, OpenSSL::SSL::SSLError ].freeze

    class Error < StandardError; end
    class Unavailable < Error; end           # rede, 5xx, configuração da plataforma, resposta inválida
    class Expired < Error; end               # session_id ou token vencido/repetido
    class LimitReached < Error; end          # 3 pedidos no mês
    class Exhausted < Error; end             # faixa esgotada na UF
    class RegistrationMismatch < Error; end  # gov.br ≠ inscrição, ou prescritor sem cadastro no SNCR

    Token = Data.define(:access_token) do
      def inspect = "#<Sncr::Client::Token [FILTERED]>"
      alias_method :to_s, :inspect
      def pretty_print(pp) = pp.text(inspect)
    end

    Block = Data.define(:first, :last, :quantity) do
      def inspect = "#<Sncr::Client::Block quantity=#{quantity}>"
      alias_method :to_s, :inspect
      def pretty_print(pp) = pp.text(inspect)
    end

    def self.for(city = Current.city, env: Rails.env)
      settings = Config.for_city(city, env: env)
      raise Unavailable, "SNCR não configurado" unless settings

      new(settings)
    end

    attr_reader :settings

    def initialize(settings, timeout: TIMEOUT)
      @settings = settings
      @timeout = timeout
    end

    def simulated? = @settings.simulated
    def provider = simulated? ? "sncr_simulated" : "sncr"

    def login_url(client_url:, state:)
      "#{@settings.public_auth_url}#{PATHS[:login]}?#{URI.encode_www_form(client_url: client_url, state: state)}"
    end

    def exchange(session_id:, origin:)
      uri = URI("#{@settings.auth_url}#{PATHS[:token]}?#{URI.encode_www_form(session_id: session_id)}")
      request = Net::HTTP::Get.new(uri)
      request["Accept"] = "application/json"
      request["Origin"] = origin if SEND_ORIGIN
      response = perform(uri, request)
      code = response.code.to_i
      raise Unavailable, "sncr_token_#{code}" if code >= 500 || code == 403 # 403 = domínio: configuração da plataforma

      access = parse(response.body)["access_token"]
      raise Expired, "sncr_session" unless code == 200 && access.is_a?(String) && access.present?

      Token.new(access_token: access)
    end

    def request_block(token:, kind:, council:, registration:, uf:)
      raise ArgumentError, "tipo desconhecido" unless KINDS.key?(kind)

      uri = URI("#{@settings.base_url}#{PATHS[:block]}")
      request = Net::HTTP::Post.new(uri)
      request["Accept"] = "application/json"
      request["Content-Type"] = "application/json"
      request["Authorization"] = "Bearer #{token.access_token}"
      request.body = JSON.generate(conselho: council, tipo: KINDS.fetch(kind), documento: registration, uf: uf,
                                   cnpj: @settings.cnpj)
      response = perform(uri, request)
      interpret_block(response.code.to_i, response.body.to_s.dup.force_encoding("UTF-8").scrub)
    end

    def inspect = "#<Sncr::Client #{provider}>"

    private

    def interpret_block(code, text)
      case code
      when 200, 201 then block(parse(text))
      when 401 then raise Expired, "sncr_token"
      when 404 then raise(REGISTRATION.match?(text) ? RegistrationMismatch : Unavailable, "sncr_404")
      when 400
        raise LimitReached, "sncr_limit" if LIMIT.match?(text)
        raise Exhausted, "sncr_exhausted" if EXHAUSTED.match?(text)
        raise RegistrationMismatch, "sncr_unregistered" if REGISTRATION.match?(text)

        raise Unavailable, "sncr_400" # CNPJ, conselho, UF: dados da plataforma
      else
        raise Unavailable, "sncr_#{code}"
      end
    end

    def block(body)
      first, last, quantity = body.values_at("inicio", "fim", "quantidade")
      raise Unavailable, "sncr_invalid_block" unless Numbers.expand(first, last, quantity)

      Block.new(first: first, last: last, quantity: quantity)
    end

    def perform(uri, request)
      response = Net::HTTP.start(uri.host, uri.port, use_ssl: uri.scheme == "https", open_timeout: @timeout,
                                                     read_timeout: @timeout, write_timeout: @timeout) { |http| http.request(request) }
      Checks.record!(@settings, ok: response.code.to_i < 500)
      response
    rescue *NETWORK_ERRORS => e
      Checks.record!(@settings, ok: false)
      raise Unavailable, "SNCR inalcançável (#{e.class.name})"
    end

    def parse(text)
      parsed = JSON.parse(text.to_s.presence || "{}")
      parsed.is_a?(Hash) ? parsed : {}
    rescue JSON::ParserError
      {}
    end
  end
end
```

> Se a Task 0 mostrou que o `Origin` não é exigido, `SEND_ORIGIN` continua `true` (o real aceita o cabeçalho de quem iniciou); se mostrou que o auth fica noutro host, é só o valor de `sncr.auth_url`. Se mostrou que o `state` volta no `client_url`, nada muda aqui (o dashboard pode usá-lo).

- [ ] **Step 4: Rode e veja passar**

Run: `rspec spec/services/sncr/client_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add app/services/sncr/numbers.rb app/services/sncr/client.rb spec/services/sncr/client_spec.rb
git commit -m "feat: exchange the SNCR session on the server and request a number block

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 8: Estoque de números com lock, lote com repetidos e a corrida real

**Files:**
- Create: `app/services/sncr/stock.rb`
- Test: `spec/services/sncr/stock_spec.rb`, `spec/services/sncr/stock_race_spec.rb`

**Interfaces:**
- Consumes: `SncrNumber`, `SncrNumberBatch` (Task 4), `Sncr::Numbers.expand`, `Sncr::Client::Block` (Task 7), `DomainEvents`.
- Produces:
  - `Sncr::Stock::MONTHLY_LIMIT = 3`, `LOW_THRESHOLD = 50`, `Received = Data.define(:batch, :received)`.
  - `.month(now) -> Date`; `.requests_this_month(user:, kind:, now: Time.current) -> Integer`.
  - `.summary(user:, simulated:, now: Time.current) -> { "rce" => { free:, used:, voided:, requests_this_month: }, "ret" => {…} }`.
  - `.record_batch!(user:, kind:, block:, simulated:, now: Time.current) -> Received` (transação própria; `pg_advisory_xact_lock` por usuário+tipo; `ON CONFLICT DO NOTHING`; publica `sncr.numbers_requested { user_id, kind, count, simulated }`).
  - `.take!(user:, kind:, simulated:, env: Rails.env) -> SncrNumber|nil` (na transação de quem chama; `FOR UPDATE SKIP LOCKED`, menor número primeiro; simulado em produção → nil).
  - `.use!(number, document:, now: Time.current) -> SncrNumber` (publica `sncr.number_used { document_id, kind }`).
  - `.void!(document:, now: Time.current) -> SncrNumber|nil` (`used → voided`; publica `sncr.number_voided { document_id, kind }`).

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/services/sncr/stock_spec.rb
require "rails_helper"

# ADR 0034 (spec §4; Desvios 4–6): estoque por prescritor e modo; o lote grava
# só números novos (Review Focus 2); o consumo é do modo corrente (Review
# Focus 4); used só com documento; voided não volta.
RSpec.describe Sncr::Stock do
  before { documents_city!; ciap2_release!; cid10_release!; sigtap_release! }

  let(:unit) { create_unit }
  let(:doctor) { doctor!(unit) }
  let(:document) { raw_document!(consultation: finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1))) }

  def block(first, last, quantity) = Sncr::Client::Block.new(first: first, last: last, quantity: quantity)

  it "grava o lote, o mês e o número do pedido; publica só a contagem" do
    received = described_class.record_batch!(user: doctor, kind: "rce", block: block("2610.1-41.0000001", "2610.1-41.0000010", 10),
                                             simulated: true)
    expect(received.received).to eq(10)
    expect(received.batch).to have_attributes(kind: "rce", quantity: 10, month_request: 1, month: Date.current.beginning_of_month)
    second = described_class.record_batch!(user: doctor, kind: "rce", block: block("2610.1-41.0000011", "2610.1-41.0000012", 2),
                                           simulated: true)
    expect(second.batch.month_request).to eq(2)
    expect(described_class.requests_this_month(user: doctor, kind: "rce")).to eq(2)
    expect(described_class.requests_this_month(user: doctor, kind: "ret")).to eq(0)
    event = DomainEvent.where(name: "sncr.numbers_requested").order(:created_at).last
    expect(event.payload).to eq("user_id" => doctor.id, "kind" => "rce", "count" => 2, "simulated" => true)
  end

  it "bloco com números repetidos: grava só os novos, sem erro (Review Focus 2)" do
    described_class.record_batch!(user: doctor, kind: "rce", block: block("2610.1-41.0000001", "2610.1-41.0000005", 5), simulated: true)
    other = doctor!(unit)
    received = nil
    log = capture_log do
      received = described_class.record_batch!(user: other, kind: "rce", block: block("2610.1-41.0000004", "2610.1-41.0000008", 5),
                                               simulated: true)
    end
    expect(received.received).to eq(3)
    expect(SncrNumber.where(kind: "rce").count).to eq(8)
    expect(SncrNumber.where(number: "2610.1-41.0000004").sole.user_id).to eq(doctor.id)
    expect(log).to include("2 número(s) já existente(s)")
    expect(log).not_to include("2610.1-41")
  end

  it "saldo e consumo do modo corrente; menor número primeiro; produção nunca tira simulado (Review Focus 4)" do
    sncr_stock!(doctor, kind: "rce", count: 2, simulated: true)
    described_class.record_batch!(user: doctor, kind: "rce", block: block("2610.1-41.0009001", "2610.1-41.0009001", 1), simulated: false)
    expect(described_class.summary(user: doctor, simulated: true)["rce"]).to include(free: 2, used: 0, voided: 0)
    expect(described_class.summary(user: doctor, simulated: false)["rce"]).to include(free: 1)
    expect(described_class.summary(user: doctor, simulated: true)["ret"]).to include(free: 0, requests_this_month: 0)

    ApplicationRecord.transaction do
      expect(described_class.take!(user: doctor, kind: "rce", simulated: false).number).to eq("2610.1-41.0009001")
      expect(described_class.take!(user: doctor, kind: "rce", simulated: true).number).to eq("2610.1-41.0000001")
      expect(described_class.take!(user: doctor, kind: "ret", simulated: true)).to be_nil
      expect(described_class.take!(user: doctor, kind: "rce", simulated: true, env: "production")).to be_nil
      expect(described_class.take!(user: doctor!(unit), kind: "rce", simulated: true)).to be_nil # o estoque é do prescritor
    end
  end

  it "use! e void!: used com documento, voided não volta; eventos só com ids" do
    sncr_stock!(doctor, count: 1)
    number = ApplicationRecord.transaction { described_class.use!(described_class.take!(user: doctor, kind: "rce", simulated: true), document: document) }
    expect(number.reload).to have_attributes(status: "used", document_id: document.id)
    voided = described_class.void!(document: document)
    expect(voided.reload.status).to eq("voided")
    expect(described_class.void!(document: document)).to be_nil
    payloads = DomainEvent.where(name: %w[sncr.number_used sncr.number_voided]).pluck(:payload)
    expect(payloads).to all(eq("document_id" => document.id, "kind" => "rce"))
    expect(described_class.summary(user: doctor, simulated: true)["rce"]).to include(free: 0, used: 0, voided: 1)
  end

  it "rollback devolve o número (falha da emissão)" do
    sncr_stock!(doctor, count: 1)
    ApplicationRecord.transaction(requires_new: true) do
      described_class.use!(described_class.take!(user: doctor, kind: "rce", simulated: true), document: document)
      raise ActiveRecord::Rollback
    end
    expect(SncrNumber.where(user_id: doctor.id).pluck(:status)).to eq([ "free" ])
  end
end
```

```ruby
# spec/services/sncr/stock_race_spec.rb
require "rails_helper"

# Review Focus 3 (ADR 0034, "um número SNCR nunca é usado duas vezes"): duas
# emissões ao mesmo tempo em conexões reais. Quem chega segundo pula o número
# travado (SKIP LOCKED) sem esperar: com 2 livres pega o outro; com 1, nil
# (a receita dele sai em papel com no_sncr_number).
RSpec.describe "Corrida no estoque SNCR" do
  self.use_transactional_tests = false

  let(:city) { TEST_CITY_A }
  let(:entered) { Queue.new }
  let(:release) { Queue.new }
  let(:threads) { [] }

  before do
    @user = CityConnection.with(city) do
      User.create!(email_address: "sncr-corrida-#{SecureRandom.hex(4)}@cidade.gov.br", password: "senha-segura-123")
    end
  end

  after do
    3.times { release << true }
    threads.each { |thread| (thread.join(5) rescue nil) || thread.kill }
    CityConnection.with(city) do
      ApplicationRecord.transaction do
        ApplicationRecord.connection.execute("SET LOCAL session_replication_role = replica")
        SncrNumber.where(user_id: @user.id).delete_all
        SncrNumberBatch.where(user_id: @user.id).delete_all
        DomainEvent.where("payload->>'user_id' = ?", @user.id).delete_all
        User.where(id: @user.id).delete_all
      end
    end
  end

  def stock!(count)
    CityConnection.with(city) do
      first = "2610.1-41.#{format('%07d', 1)}"
      last = "2610.1-41.#{format('%07d', count)}"
      Sncr::Stock.record_batch!(user: @user, kind: "rce", block: Sncr::Client::Block.new(first: first, last: last, quantity: count),
                                simulated: true)
    end
  end

  def holder
    track(Thread.new do
      CityConnection.with(city) do
        ApplicationRecord.transaction do
          taken = Sncr::Stock.take!(user: @user, kind: "rce", simulated: true)
          entered << true
          release.pop(timeout: 5)
          taken&.number
        end
      end
    end)
  end

  def second
    track(Thread.new do
      CityConnection.with(city) do
        ApplicationRecord.transaction { Sncr::Stock.take!(user: @user, kind: "rce", simulated: true)&.number }
      end
    end)
  end

  def track(thread) = thread.tap { threads << thread }

  it "com 2 livres: o segundo pega o outro, sem esperar" do
    stock!(2)
    first = holder
    expect(entered.pop(timeout: 5)).to be(true)
    other = second
    expect(other.join(5)).to be_truthy
    expect(first).to be_alive
    release << true
    expect([ first.value, other.value ]).to eq(%w[2610.1-41.0000001 2610.1-41.0000002])
  end

  it "com 1 livre: o segundo recebe nil na hora (papel), nunca o mesmo número" do
    stock!(1)
    first = holder
    expect(entered.pop(timeout: 5)).to be(true)
    other = second
    expect(other.join(5)).to be_truthy
    expect(other.value).to be_nil
    release << true
    expect(first.value).to eq("2610.1-41.0000001")
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `rspec spec/services/sncr/stock_spec.rb spec/services/sncr/stock_race_spec.rb`
Expected: FAIL (`uninitialized constant Sncr::Stock`).

- [ ] **Step 3: Implemente**

```ruby
# app/services/sncr/stock.rb
# Estoque de números SNCR por prescritor (ADR 0034; spec §4; Desvios 4–6). O
# lote grava o que o SNCR deu (só os números novos: o SNCR já entregou
# repetidos); a emissão tira o menor número livre do modo corrente com
# FOR UPDATE SKIP LOCKED NA TRANSAÇÃO DELA (falha → rollback devolve); usado
# só com documento; cancelamento e volta ao papel anulam (voided, sem volta).
# Nenhum número SNCR em log ou evento.
module Sncr
  module Stock
    MONTHLY_LIMIT = 3
    LOW_THRESHOLD = 50
    Received = Data.define(:batch, :received)

    module_function

    def month(now) = now.in_time_zone.to_date.beginning_of_month

    def requests_this_month(user:, kind:, now: Time.current)
      SncrNumberBatch.where(user_id: user.id, kind: kind, month: month(now)).count
    end

    def summary(user:, simulated:, now: Time.current)
      counts = SncrNumber.where(user_id: user.id, simulated: simulated).group(:kind, :status).count
      SncrNumber::KINDS.to_h do |kind|
        [ kind, { free: counts.fetch([ kind, "free" ], 0), used: counts.fetch([ kind, "used" ], 0),
                  voided: counts.fetch([ kind, "voided" ], 0), requests_this_month: requests_this_month(user: user, kind: kind, now: now) } ]
      end
    end

    def record_batch!(user:, kind:, block:, simulated:, now: Time.current)
      numbers = Numbers.expand(block.first, block.last, block.quantity)
      raise ArgumentError, "bloco inválido" unless numbers && SncrNumber::KINDS.include?(kind)

      ApplicationRecord.transaction(requires_new: true) do
        lock_key = ApplicationRecord.sanitize_sql_array([ "SELECT pg_advisory_xact_lock(hashtext(?))", "sncr:#{user.id}:#{kind}" ])
        ApplicationRecord.connection.execute(lock_key)
        current = month(now)
        batch = SncrNumberBatch.create!(user: user, kind: kind, first_number: block.first, last_number: block.last,
                                        quantity: block.quantity, month: current,
                                        month_request: SncrNumberBatch.where(user_id: user.id, kind: kind, month: current).count + 1,
                                        simulated: simulated, requested_at: now, created_at: now)
        rows = numbers.map do |number|
          { kind: kind, number: number, batch_id: batch.id, user_id: user.id, simulated: simulated, status: "free", created_at: now }
        end
        inserted = SncrNumber.insert_all(rows, unique_by: :idx_sncr_numbers_kind_number, returning: %w[id], record_timestamps: false).length
        repeated = numbers.size - inserted
        Rails.logger.warn("[sncr] lote com #{repeated} número(s) já existente(s)") if repeated.positive?
        DomainEvents.publish("sncr.numbers_requested", user_id: user.id, kind: kind, count: inserted, simulated: simulated)
        Received.new(batch: batch, received: inserted)
      end
    end

    def take!(user:, kind:, simulated:, env: Rails.env)
      return nil if simulated && env.to_s == "production"

      SncrNumber.available.where(user_id: user.id, kind: kind, simulated: simulated)
                .order(:number).lock("FOR UPDATE SKIP LOCKED").first
    end

    def use!(number, document:, now: Time.current)
      number.update!(status: "used", document_id: document.id, used_at: now)
      DomainEvents.publish("sncr.number_used", document_id: document.id, kind: number.kind)
      number
    end

    def void!(document:, now: Time.current)
      number = SncrNumber.lock.find_by(document_id: document.id, status: "used")
      return nil unless number

      number.update!(status: "voided", voided_at: now)
      DomainEvents.publish("sncr.number_voided", document_id: document.id, kind: number.kind)
      number
    end
  end
end
```

- [ ] **Step 4: Rode e veja passar**

Run: `rspec spec/services/sncr/stock_spec.rb spec/services/sncr/stock_race_spec.rb spec/services/medications/anticonvulsant_marks_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add app/services/sncr/stock.rb spec/services/sncr/stock_spec.rb spec/services/sncr/stock_race_spec.rb
git commit -m "feat: keep a per-prescriber SNCR number stock taken with SKIP LOCKED

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
### Task 9: Pedido de números — `GET /sncr/stock`, `POST /sncr/requests`, `POST /sncr/oauth/callback`

**Files:**
- Create: `app/commands/sncr/start_request.rb`, `app/commands/sncr/complete_request.rb`, `app/controllers/sncr/base_controller.rb`, `app/controllers/sncr/stock_controller.rb`, `app/controllers/sncr/requests_controller.rb`, `app/controllers/sncr/oauth_controller.rb`
- Modify: `config/routes.rb`
- Test: `spec/requests/sncr_requests_spec.rb`

**Interfaces:**
- Consumes: `Signatures::OauthStates.{issue!,consume}` (19b), `Sncr::{Client,Stock,Mock}` (Tasks 1, 7, 8), `ControlledPrescriptionsGate`, `AttendanceAccess#render_failure`, `CitizenVerificationPolicy#care?`, `CityPublicUrl.{base,dashboard}`.
- Produces:
  - `Sncr::StartRequest.call(user:, kind:, return_to:, now: Time.current) -> Result` — `ok(authorize_url:, state:)` | `fail(:invalid_content (field "kind") | :cbo_not_allowed | :monthly_limit_reached | :sncr_unavailable)`. `Sncr::StartRequest.callback_url(city) -> String`.
  - `Sncr::CompleteRequest.call(user:, state:, session_id:, error: nil, now: Time.current) -> Result` — `ok(kind:, received:, batch_id:, return_to:)` | `fail(:invalid_state | :authorization_expired | :authorization_denied | :registration_mismatch | :monthly_limit_reached | :sncr_unavailable | :sncr_exhausted | :cbo_not_allowed)` com `return_to` nos detalhes depois de achar o state.
  - Rotas: `GET /sncr/stock` → `{ rce: { free, used, voided, requests_this_month }, ret: {…}, low_threshold: 50, simulated }` (403 `cbo_not_allowed` sem perfil ou com conselho fora de CRM/CRO, como o pedido); `POST /sncr/requests { kind, return_to }` → `{ authorize_url, state }`; `POST /sncr/oauth/callback { state, session_id }` ou `{ state, error }` → `{ kind, received, batch_id, return_to }`. `Sncr::BaseController::ERROR_STATUS`.

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/requests/sncr_requests_spec.rb
require "rails_helper"

# Contrato §4 (spec §3–§4; Desvios 1–4): o profissional (CRM/CRO) pede um
# bloco pelo gov.br; o api troca o session_id e grava o lote na mesma
# requisição; nada de token no banco, no log ou na resposta. Review Focus 1:
# callback repetido, vencido, de outro usuário.
RSpec.describe "Pedido de números SNCR", type: :request do
  before do
    controlled_city!(mock: true)
    stub_sncr_mock!
  end

  let(:unit) { create_unit }
  let(:doctor) { doctor!(unit) }
  def body = JSON.parse(response.body)

  def start(kind = "rce", return_to: "/dashboard/minha-conta/sncr")
    post "/sncr/requests", params: { kind: kind, return_to: return_to }, as: :json
    body
  end

  def client_url_of(url) = URI.decode_www_form(URI(url).query).to_h.fetch("client_url")

  def callback(state, session_id: nil, error: nil)
    post "/sncr/oauth/callback", params: { state: state, session_id: session_id, error: error }.compact, as: :json
  end

  it "ponta a ponta: URL do login simulado, troca no servidor, lote gravado, saldo" do
    sign_in_as(doctor)
    started = start
    expect(started["authorize_url"]).to start_with("http://localhost:8092/api/v1/auth/login?")
    expect(client_url_of(started["authorize_url"])).to eq("#{CityPublicUrl.dashboard(Current.city)}sncr/callback")
    log = capture_log do
      callback(started["state"], session_id: fake_sncr.approve!(client_url_of(started["authorize_url"])))
    end
    expect(response).to have_http_status(:ok)
    expect(body).to include("kind" => "rce", "received" => 1000, "return_to" => "/dashboard/minha-conta/sncr")
    expect(SncrNumberBatch.find(body["batch_id"])).to have_attributes(user_id: doctor.id, simulated: true, month_request: 1)
    expect(log).not_to include(*fake_sncr.issued_tokens)
    expect(SignatureOauthState.where(user_id: doctor.id).pluck(:purpose, :provider)).to eq([ %w[sncr_rce sncr_simulated] ])

    get "/sncr/stock"
    expect(body).to eq("rce" => { "free" => 1000, "used" => 0, "voided" => 0, "requests_this_month" => 1 },
                       "ret" => { "free" => 0, "used" => 0, "voided" => 0, "requests_this_month" => 0 },
                       "low_threshold" => 50, "simulated" => true)
  end

  it "callback repetido, vencido, de outro usuário, recusado no gov.br (Review Focus 1)" do
    sign_in_as(doctor)
    started = start
    url = client_url_of(started["authorize_url"])
    callback(started["state"], session_id: fake_sncr.approve!(url))
    expect(response).to have_http_status(:ok)
    callback(started["state"], session_id: fake_sncr.approve!(url))
    expect([ response.status, body["error"] ]).to eq([ 422, "invalid_state" ])

    late = start
    session_id = fake_sncr.approve!(url)
    fake_sncr.clock = -> { Time.now + 31 }
    callback(late["state"], session_id: session_id)
    expect([ response.status, body["error"], body["return_to"] ]).to eq([ 409, "authorization_expired", "/dashboard/minha-conta/sncr" ])
    fake_sncr.clock = -> { Time.now }

    denied = start
    callback(denied["state"], error: "access_denied")
    expect([ response.status, body["error"] ]).to eq([ 403, "authorization_denied" ])

    other = start
    sign_in_as(doctor!(unit))
    callback(other["state"], session_id: fake_sncr.approve!(url))
    expect([ response.status, body["error"] ]).to eq([ 422, "invalid_state" ])
    expect(SncrNumberBatch.count).to eq(1)
    expect(SignatureOauthState.where(purpose: %w[sncr_rce sncr_ret]).pluck(:code_verifier)).to all(be_present)
  end

  it "limite do mês antes do gov.br; o SNCR recusando também; esgotamento; inscrição divergente; queda" do
    sign_in_as(doctor)
    3.times { sncr_stock!(doctor, kind: "ret", count: 1) }
    start("ret")
    expect([ response.status, body["error"] ]).to eq([ 409, "monthly_limit_reached" ])

    url = client_url_of(start["authorize_url"])
    fake_sncr.exhausted_ufs << doctor.professional.council_state
    callback(start["state"], session_id: fake_sncr.approve!(url))
    expect([ response.status, body["error"] ]).to eq([ 503, "sncr_exhausted" ])
    fake_sncr.exhausted_ufs.clear

    fake_sncr.mismatch_documents << doctor.professional.registration_number
    callback(start["state"], session_id: fake_sncr.approve!(url))
    expect([ response.status, body["error"] ]).to eq([ 403, "registration_mismatch" ])
    fake_sncr.mismatch_documents.clear

    fake_sncr.failures << 502
    callback(start["state"], session_id: fake_sncr.approve!(url))
    expect([ response.status, body["error"] ]).to eq([ 503, "sncr_unavailable" ])
    expect(SncrNumberBatch.where(kind: "rce").count).to eq(0)
  end

  it "enfermeiro, recepção, tipo inválido, interruptor desligado" do
    sign_in_as(nurse!(unit))
    start
    expect([ response.status, body["error"] ]).to eq([ 403, "cbo_not_allowed" ])
    get "/sncr/stock"
    expect([ response.status, body["error"] ]).to eq([ 403, "cbo_not_allowed" ])
    sign_in_as(verifier!)
    start
    expect([ response.status, body["error"] ]).to eq([ 403, "missing_role" ])
    sign_in_as(doctor)
    start("nr")
    expect([ response.status, body["error"], body["field"] ]).to eq([ 422, "invalid_content", "kind" ])
    Platform::Features.set!(city: Current.city, key: "controlled_prescriptions", enabled: false, maintainer: ledi_maintainer!)
    get "/sncr/stock"
    expect([ response.status, body ]).to eq([ 403, { "error" => "feature_disabled", "feature" => "controlled_prescriptions" } ])
  end

  it "SNCR real sem credencial: 503 sncr_unavailable, sem state gravado" do
    Platform::Features.set!(city: Current.city, key: "sncr_mock", enabled: false, maintainer: ledi_maintainer!)
    allow(Sncr::Config).to receive(:credentials).and_return({})
    sign_in_as(doctor)
    start
    expect([ response.status, body["error"] ]).to eq([ 503, "sncr_unavailable" ])
    expect(SignatureOauthState.where(user_id: doctor.id).count).to eq(0)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `rspec spec/requests/sncr_requests_spec.rb`
Expected: FAIL (`No route matches [POST] "/sncr/requests"`).

- [ ] **Step 3: Implemente os comandos**

```ruby
# app/commands/sncr/start_request.rb
# Começa o pedido de números (ADR 0034; contrato §4; Desvios 1–4): só quem
# tem inscrição em CRM/CRO (o SNCR não aceita outro conselho para humanos), no
# máximo 3 pedidos por tipo no mês, com o SNCR da cidade (real ou simulado).
# O state de uso único é o do 19b (signature_oauth_states, propósito
# sncr_<kind>); o code_verifier da linha nunca sai (o SNCR não usa PKCE nosso).
module Sncr
  module StartRequest
    COUNCILS = %w[CRM CRO].freeze

    module_function

    def call(user:, kind:, return_to:, now: Time.current)
      return Result.fail(:invalid_content, details: { field: "kind" }) unless Client::KINDS.key?(kind)
      return Result.fail(:cbo_not_allowed) unless COUNCILS.include?(user.professional&.council)
      return Result.fail(:monthly_limit_reached) if Stock.requests_this_month(user: user, kind: kind, now: now) >= Stock::MONTHLY_LIMIT

      client = Client.for(Current.city)
      issued = Signatures::OauthStates.issue!(user: user, purpose: "sncr_#{kind}", provider: client.provider,
                                              return_to: return_to, now: now)
      Result.ok(authorize_url: client.login_url(client_url: callback_url(Current.city), state: issued.state), state: issued.state)
    rescue Client::Unavailable
      Result.fail(:sncr_unavailable)
    end

    def callback_url(city) = "#{CityPublicUrl.dashboard(city)}sncr/callback"
  end
end
```

```ruby
# app/commands/sncr/complete_request.rb
# Volta do login do SNCR (ADR 0034; contrato §4; Desvios 1–5). O state é
# consumido ANTES de tudo, na transação própria do 19b (uso único mesmo que o
# passo seguinte falhe). Depois, na mesma requisição e dentro dos 30 s: troca
# o session_id pelo token (Origin = host da cidade) e pede o bloco; o token é
# uma variável local e nunca é gravado. O lote é gravado só com resposta 201.
# Toda falha depois de achar o state leva o return_to dele.
module Sncr
  module CompleteRequest
    PURPOSES = { "sncr_rce" => "rce", "sncr_ret" => "ret" }.freeze
    MAX_SESSION_ID = 512

    module_function

    def call(user:, state:, session_id:, error: nil, now: Time.current)
      consumed = Signatures::OauthStates.consume(state, user: user, now: now)
      return consumed if consumed.failure?

      row = consumed.payload[:state]
      kind = PURPOSES[row.purpose]
      return with_return_to(Result.fail(:invalid_state), row) unless kind

      outcome = proceed(row, kind, user: user, session_id: session_id, error: error, now: now)
      return with_return_to(outcome, row) if outcome.failure?

      Result.ok(**outcome.payload, return_to: row.return_to)
    end

    def proceed(row, kind, user:, session_id:, error:, now:)
      return Result.fail(:authorization_denied) if error.present?
      return Result.fail(:invalid_state) unless session_id.is_a?(String) && session_id.size.between?(1, MAX_SESSION_ID)

      professional = user.professional
      return Result.fail(:cbo_not_allowed) unless StartRequest::COUNCILS.include?(professional&.council)

      client = Client.for(Current.city)
      return Result.fail(:sncr_unavailable) unless client.provider == row.provider # o sncr_mock virou no meio do caminho

      started = Process.clock_gettime(Process::CLOCK_MONOTONIC)
      token = client.exchange(session_id: session_id, origin: CityPublicUrl.base(Current.city))
      block = client.request_block(token: token, kind: kind, council: professional.council,
                                   registration: professional.registration_number, uf: professional.council_state)
      elapsed = Process.clock_gettime(Process::CLOCK_MONOTONIC) - started
      Rails.logger.warn("[sncr] troca e pedido levaram #{elapsed.round(1)} s") if elapsed > 20
      received = Stock.record_batch!(user: user, kind: kind, block: block, simulated: client.simulated?, now: now)
      Result.ok(kind: kind, received: received.received, batch_id: received.batch.id)
    rescue Client::Expired
      Result.fail(:authorization_expired)
    rescue Client::LimitReached
      Result.fail(:monthly_limit_reached)
    rescue Client::Exhausted
      Result.fail(:sncr_exhausted)
    rescue Client::RegistrationMismatch
      Result.fail(:registration_mismatch)
    rescue Client::Unavailable
      Result.fail(:sncr_unavailable)
    end

    def with_return_to(result, row)
      Result.fail(result.reason, message: result.message, details: result.details.merge(return_to: row.return_to))
    end
    private_class_method :proceed, :with_return_to
  end
end
```

- [ ] **Step 4: Controllers e rotas**

```ruby
# app/controllers/sncr/base_controller.rb
# Rotas do SNCR (ADR 0034; contrato §4, §6): interruptor controlled_prescriptions
# utilizável e papel de profissional (a recepção recebe 403 missing_role); o
# admin troca o before_action.
module Sncr
  class BaseController < ApplicationController
    include Authentication
    include AttendanceAccess
    include ControlledPrescriptionsGate

    wrap_parameters false

    ERROR_STATUS = {
      missing_role: :forbidden, cbo_not_allowed: :forbidden, authorization_denied: :forbidden, registration_mismatch: :forbidden,
      invalid_state: :unprocessable_entity, invalid_content: :unprocessable_entity,
      authorization_expired: :conflict, monthly_limit_reached: :conflict,
      sncr_unavailable: :service_unavailable, sncr_exhausted: :service_unavailable
    }.freeze

    before_action :require_controlled_prescriptions!
    before_action :require_professional

    private

    def require_professional
      forbid("missing_role") unless CitizenVerificationPolicy.new(Current.user, nil).care?
    end

    def failure(result) = render_failure(result, ERROR_STATUS)
    def body = params.to_unsafe_h.except("controller", "action", "id")
  end
end
```

```ruby
# app/controllers/sncr/stock_controller.rb
# Minha conta → SNCR (contrato §4): saldo do modo corrente da cidade.
module Sncr
  class StockController < BaseController
    def show
      return failure(Result.fail(:cbo_not_allowed)) unless StartRequest::COUNCILS.include?(Current.user.professional&.council)

      simulated = Mock.on?(Current.city)
      render json: Stock.summary(user: Current.user, simulated: simulated)
                        .merge(low_threshold: Stock::LOW_THRESHOLD, simulated: simulated)
    end
  end
end
```

```ruby
# app/controllers/sncr/requests_controller.rb
# "Obter números do SNCR" (contrato §4; Divergência D2: devolve também o state,
# que o dashboard guarda e reenvia no callback).
module Sncr
  class RequestsController < BaseController
    def create
      result = StartRequest.call(user: Current.user, kind: body["kind"].to_s, return_to: body["return_to"])
      return failure(result) if result.failure?

      render json: { authorize_url: result.payload[:authorize_url], state: result.payload[:state] }
    end
  end
end
```

```ruby
# app/controllers/sncr/oauth_controller.rb
# Volta do login do SNCR pelo dashboard (contrato §4; Divergência D1):
# { state, session_id } ou { state, error }.
module Sncr
  class OauthController < BaseController
    def callback
      result = CompleteRequest.call(user: Current.user, state: body["state"], session_id: body["session_id"], error: body["error"])
      return failure(result) if result.failure?

      render json: result.payload.slice(:kind, :received, :batch_id, :return_to)
    end
  end
end
```

Em `config/routes.rb`, junto do bloco `/signature`:

```ruby
  # SNCR (ADR 0034; contrato §4, §6). Prefixo único: uma entrada no proxy de dev do dashboard.
  scope "/sncr", module: :sncr, as: :sncr do
    get  "stock",           to: "stock#show"
    post "requests",        to: "requests#create"
    post "oauth/callback",  to: "oauth#callback"
    get  "admin/overview",  to: "admin#overview"
  end
```
(`admin#overview` nasce na Task 18; até lá a rota existe e o controller não — a spec desta task não a chama.)

- [ ] **Step 5: Rode e veja passar**

Run: `rspec spec/requests/sncr_requests_spec.rb spec/requests/signature_certificates_spec.rb`
Expected: PASS (o state do 19b segue funcionando com os propósitos novos).

- [ ] **Step 6: Commit**

```bash
git add app/commands/sncr/start_request.rb app/commands/sncr/complete_request.rb app/controllers/sncr/base_controller.rb app/controllers/sncr/stock_controller.rb app/controllers/sncr/requests_controller.rb app/controllers/sncr/oauth_controller.rb config/routes.rb spec/requests/sncr_requests_spec.rb
git commit -m "feat: let prescribers request SNCR number blocks through their gov.br login

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 10: Contato do prescritor (endereço e telefone)

**Files:**
- Create: `app/services/clinical_documents/address.rb`, `app/controllers/professional_contacts_controller.rb`
- Modify: `config/routes.rb`
- Test: `spec/services/clinical_documents/address_spec.rb`, `spec/requests/professional_contact_spec.rb`

**Interfaces:**
- Consumes: `Professional#prescriber_address`, `#phone`, `Professional::UFS`, `ClinicalRecordGate`, `AttendanceAccess`.
- Produces:
  - `ClinicalDocuments::Address.normalize(value) -> [Hash, nil] | [nil, field]` (chaves `street`, `number`, `complement` (só se presente), `district`, `city`, `uf` `^[A-Z]{2}$`, `zip` `^[0-9]{8}$` — formatos do esquema, sem máscara); `.phone(value) -> String|nil` (`^[0-9]{10,11}$`, sem máscara); `.line(address) -> String` (uma linha para o PDF).
  - `GET /attendance/professional_profile/contact` → `{ address: {…}|null, phone: "…"|null }`; `PUT` com `{ address, phone }` → o mesmo; 422 `invalid_address` (`field`), `invalid_phone`; 404 `not_found` sem perfil de profissional.

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/services/clinical_documents/address_spec.rb
require "rails_helper"

# Valores fixados 2; Review Focus 5: endereço digitado por gente.
RSpec.describe ClinicalDocuments::Address do
  let(:base) { ControlledPrescriptionHelpers::ADDRESS }

  it "aceita só os formatos do esquema; apara espaços; complemento só com espaço vira ausente" do
    address, field = described_class.normalize(base.merge("street" => "  Rua  das Araucárias ", "complement" => " "))
    expect(field).to be_nil
    expect(address).to eq(base)
    expect(address).not_to have_key("complement")
  end

  it "máscara é recusada com o campo: CEP com hífen ou ponto, UF minúscula (Review Focus 5)" do
    expect(described_class.normalize(base.merge("zip" => "80010-000"))).to eq([ nil, "zip" ])
    expect(described_class.normalize(base.merge("zip" => "80.010000"))).to eq([ nil, "zip" ])
    expect(described_class.normalize(base.merge("uf" => "pr"))).to eq([ nil, "uf" ])
  end

  it "devolve o campo exato que falhou" do
    expect(described_class.normalize(base.merge("zip" => "8001000"))).to eq([ nil, "zip" ])
    expect(described_class.normalize(base.merge("uf" => "XX"))).to eq([ nil, "uf" ])
    expect(described_class.normalize(base.except("district"))).to eq([ nil, "district" ])
    expect(described_class.normalize(base.merge("street" => "x" * 121))).to eq([ nil, "street" ])
    expect(described_class.normalize(base.merge("complement" => "c" * 61))).to eq([ nil, "complement" ])
    expect(described_class.normalize("rua tal")).to eq([ nil, "address" ])
  end

  it "telefone só com dígitos (10 ou 11); acento e emoji passam (o PDF troca o que não tem)" do
    expect(described_class.phone("41999990000")).to eq("41999990000")
    expect(described_class.phone("4133330000")).to eq("4133330000")
    expect(described_class.phone("(41) 99999-0000")).to be_nil
    expect(described_class.phone("4133")).to be_nil
    expect(described_class.phone(4_133_330_000)).to be_nil
    address, = described_class.normalize(base.merge("street" => "Rua São João 🌳"))
    expect(described_class.line(address)).to eq("Rua São João 🌳, 120 — Centro — Curitiba/PR — CEP 80010-000")
  end
end
```

```ruby
# spec/requests/professional_contact_spec.rb
require "rails_helper"

# Contrato §5: o próprio profissional lê e grava endereço e telefone de
# prescritor; o endereço vai cifrado; nada no log.
RSpec.describe "Contato do prescritor", type: :request do
  before { documents_city! }

  let(:unit) { create_unit }
  let(:doctor) { doctor!(unit) }
  def body = JSON.parse(response.body)

  it "lê vazio, grava, lê de volta; PUT exige JSON" do
    sign_in_as(doctor)
    get "/attendance/professional_profile/contact"
    expect(body).to eq("address" => nil, "phone" => doctor.professional.phone)
    log = capture_log do
      put "/attendance/professional_profile/contact",
          params: { address: ControlledPrescriptionHelpers::ADDRESS, phone: "4133330000" }, as: :json
    end
    expect(response).to have_http_status(:ok)
    expect(body).to eq("address" => ControlledPrescriptionHelpers::ADDRESS, "phone" => "4133330000")
    expect(log).not_to include("Araucárias", "3333")
    expect(doctor.professional.reload.prescriber_contact).to include("phone" => "4133330000")
    put "/attendance/professional_profile/contact", params: { phone: "4133330000" }
    expect(response).to have_http_status(:unsupported_media_type).or have_http_status(:forbidden)
  end

  it "422 com o campo; recepção sem perfil → 404" do
    sign_in_as(doctor)
    put "/attendance/professional_profile/contact", params: { address: ControlledPrescriptionHelpers::ADDRESS.merge("uf" => "ZZ"), phone: "4133330000" }, as: :json
    expect([ response.status, body ]).to eq([ 422, { "error" => "invalid_address", "field" => "uf" } ])
    put "/attendance/professional_profile/contact", params: { address: ControlledPrescriptionHelpers::ADDRESS, phone: "12" }, as: :json
    expect([ response.status, body["error"] ]).to eq([ 422, "invalid_phone" ])
    put "/attendance/professional_profile/contact", params: { address: ControlledPrescriptionHelpers::ADDRESS, phone: "(41) 3333-0000" }, as: :json
    expect([ response.status, body["error"] ]).to eq([ 422, "invalid_phone" ])
    put "/attendance/professional_profile/contact", params: { address: ControlledPrescriptionHelpers::ADDRESS.merge("zip" => "80010-000"), phone: "4133330000" }, as: :json
    expect([ response.status, body ]).to eq([ 422, { "error" => "invalid_address", "field" => "zip" } ])
    sign_in_as(verifier!)
    get "/attendance/professional_profile/contact"
    expect([ response.status, body["error"] ]).to eq([ 404, "not_found" ])
  end
end
```

> O status da guarda de Content-Type é o que a guarda `require_json_for_cookie_writes` já devolve (confira a spec dela e fixe o status exato na asserção).

- [ ] **Step 2: Rode e veja falhar**

Run: `rspec spec/services/clinical_documents/address_spec.rb spec/requests/professional_contact_spec.rb`
Expected: FAIL.

- [ ] **Step 3: Implemente**

```ruby
# app/services/clinical_documents/address.rb
# Endereço completo da RCE/RET (ADR 0034; Lei 5.991, art. 35; Valores fixados
# 2): do prescritor (perfil) e do paciente (digitado na receita). Devolve o
# campo exato que falhou; nunca loga o valor.
module ClinicalDocuments
  module Address
    LIMITS = { "street" => 1..120, "number" => 1..10, "district" => 1..80, "city" => 1..80 }.freeze
    COMPLEMENT = 1..60

    module_function

    def normalize(value)
      return [ nil, "address" ] unless value.is_a?(Hash)

      value = value.to_h.transform_keys(&:to_s)
      out = {}
      LIMITS.each do |key, range|
        text = value[key].is_a?(String) ? value[key].squish : ""
        return [ nil, key ] unless range.cover?(text.size)

        out[key] = text
      end
      complement = value["complement"].is_a?(String) ? value["complement"].squish : ""
      return [ nil, "complement" ] if complement.size > COMPLEMENT.max

      out["complement"] = complement if complement.present?
      # Formatos do esquema (clinical-v1.2.0): sem máscara, uma representação por valor.
      uf = value["uf"]
      return [ nil, "uf" ] unless uf.is_a?(String) && uf.match?(/\A[A-Z]{2}\z/) && Professional::UFS.include?(uf)

      zip = value["zip"]
      return [ nil, "zip" ] unless zip.is_a?(String) && zip.match?(/\A[0-9]{8}\z/)

      [ out.merge("uf" => uf, "zip" => zip).slice("street", "number", "complement", "district", "city", "uf", "zip"), nil ]
    end

    def phone(value) = value.is_a?(String) && value.match?(/\A[0-9]{10,11}\z/) ? value : nil

    def line(address)
      return "" unless address.is_a?(Hash)

      street = [ address["street"], address["number"], address["complement"] ].compact_blank.join(", ")
      zip = address["zip"].to_s.sub(/\A(\d{5})(\d{3})\z/, '\1-\2')
      "#{street} — #{address['district']} — #{address['city']}/#{address['uf']} — CEP #{zip}"
    end
  end
end
```

```ruby
# app/controllers/professional_contacts_controller.rb
# Contato do prescritor (ADR 0034; contrato §5): o próprio profissional lê e
# grava endereço (cifrado) e telefone. Atrás do prontuário, não do interruptor
# (a cidade prepara antes de ligar, como o CNPJ do 19c).
class ProfessionalContactsController < ApplicationController
  include Authentication
  include AttendanceAccess
  include ClinicalRecordGate

  wrap_parameters false
  before_action :require_clinical_record!

  def show
    professional = Current.user.professional
    return render(json: { error: "not_found" }, status: :not_found) unless professional

    render json: payload(professional)
  end

  def update
    professional = Current.user.professional
    return render(json: { error: "not_found" }, status: :not_found) unless professional

    body = params.to_unsafe_h
    address, field = ClinicalDocuments::Address.normalize(body["address"])
    return render(json: { error: "invalid_address", field: field }, status: :unprocessable_entity) unless address

    phone = ClinicalDocuments::Address.phone(body["phone"])
    return render(json: { error: "invalid_phone" }, status: :unprocessable_entity) unless phone

    professional.update!(prescriber_address: address.to_json, phone: phone)
    render json: payload(professional)
  end

  private

  def payload(professional) = { address: professional.prescriber_address_data, phone: professional.phone }
end
```

Em `config/routes.rb`:

```ruby
  # Contato do prescritor (ADR 0034; contrato §5).
  get "/attendance/professional_profile/contact", to: "professional_contacts#show"
  put "/attendance/professional_profile/contact", to: "professional_contacts#update"
```

- [ ] **Step 4: Rode e veja passar**

Run: `rspec spec/services/clinical_documents/address_spec.rb spec/requests/professional_contact_spec.rb spec/requests/professionals_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add app/services/clinical_documents/address.rb app/controllers/professional_contacts_controller.rb config/routes.rb spec/services/clinical_documents/address_spec.rb spec/requests/professional_contact_spec.rb
git commit -m "feat: let prescribers keep the address and phone printed on controlled prescriptions

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
## Fatia 3 — Receita de controle especial e de antimicrobiano (F-19.26, F-19.27)

### Task 11: Categoria da receita, regras da Portaria 344/RDC 471 e identificação do paciente

**Files:**
- Create: `app/services/clinical_documents/content/controlled.rb`, `app/services/clinical_documents/content/patient_identification.rb`
- Modify: `app/services/clinical_documents/content/prescription.rb`
- Test: `spec/services/clinical_documents/controlled_content_spec.rb`

**Interfaces:**
- Consumes: `ClinicalDocuments::Content::{Input,Prescription}` (19c), `ClinicalDocuments::Address` (Task 10), `CitizenIdentity::Cpf.normalize`, `MedicationCatalogItem#controlled_list`/`#anticonvulsant` (Tasks 2–3), `Medications::Substances::Flags#anticonvulsant`.
- Produces:
  - `ClinicalDocuments::Content::Controlled` — `SPECIAL = %w[C1 C5]`, `NOTIFICATION = %w[A1 A2 A3 B1 B2]`, `UNSUPPORTED = %w[C2 C3 C4]`, `MAX_C1 = 3`, `SPECIAL_DAYS = 60`, `ANTICONVULSANT_DAYS = 180`, `VALIDITY = { "special_control" => 30, "antimicrobial" => 10 }`, `PRESCRIBERS = %i[physician dentist]`; `.classify(items) -> Result ok(category:)` | `fail(:controlled_not_allowed | :requires_notification | :not_supported (index) | :mixed_categories)`; `.legacy(items) -> Result` (interruptor desligado: o 19c); `.check(category, items, role:, identification:, prescriber_contact:, patient:) -> Result ok(patient_identification:, prescriber_contact:)` | `fail(:cbo_not_allowed | :too_many_c1_substances | :duration_exceeded (index) | :invalid_item (index, field) | :patient_identification_required (field) | :invalid_content (field) | :prescriber_address_missing)`.
  - `ClinicalDocuments::Content::PatientIdentification.call(value, required:, patient:) -> Result ok(value: Hash|nil)` — `{ "cpf" => "<11 dígitos do cadastro>", "address" => {…} }`; a entrada manda só `{ address }` (CPF, `no_cpf` e `passport` da entrada são ignorados); cadastro sem CPF válido → `patient_identification_required` (`field: "patient_identification.cpf"`).
  - `Content::Prescription.call(input, role:, patient:, catalog:, matcher:, today:, city_cnpj: nil, controlled: false, prescriber_contact: nil) -> Result ok(content:, items:, antimicrobial:, category:)`; o `content` ganha `category`, `sncr` (sempre `nil` aqui; a emissão preenche), `patient_identification`, `prescriber_contact`; `copies` 2 e `valid_until` fixo em `special_control`/`antimicrobial`. Item interno ganha `controlled_list` e `anticonvulsant`. Ordem das checagens: forma dos itens → categoria → regras da categoria → enfermagem → validade.

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/services/clinical_documents/controlled_content_spec.rb
require "rails_helper"

# Spec §5; ADR 0034 (Invariantes; Valores fixados 3–4): categoria calculada
# pelos itens; misturar → mixed_categories; listas A/B → requires_notification;
# C2/C3/C4 → not_supported; RCE só médico/dentista, ≤ 3 C1, ≤ 60 dias (180
# anticonvulsivante), identificação do paciente e contato do prescritor;
# antimicrobiano 10 dias; desligado, o 19c. Review Focus 5.
RSpec.describe ClinicalDocuments::Content::Prescription do
  before { documents_city!; ciap2_release!; cid10_release!; sigtap_release! }

  let(:catalog_items) do
    import_test_catalog!
    { fluoxetina: controlled_item!(code: 9_900_001, ingredient: "FLUOXETINA", list: "C1"),
      carbamazepina: controlled_item!(code: 9_900_002, ingredient: "CARBAMAZEPINA", list: "C1", anticonvulsant: true),
      diazepam: controlled_item!(code: 9_900_003, ingredient: "DIAZEPAM", list: "B1"),
      talidomida: controlled_item!(code: 9_900_004, ingredient: "TALIDOMIDA", list: "C3"),
      zidovudina: controlled_item!(code: 9_900_005, ingredient: "ZIDOVUDINA", list: "C4"),
      sertralina: controlled_item!(code: 9_900_006, ingredient: "SERTRALINA", list: "C1"),
      amitriptilina: controlled_item!(code: 9_900_007, ingredient: "AMITRIPTILINA", list: "C1"),
      nandrolona: controlled_item!(code: 9_900_009, ingredient: "NANDROLONA", list: "C5"),
      amoxicilina: catalog_item(271_089), losartana: catalog_item(268_856) }
  end
  let(:patient) { finalized_consultation!(unit: create_unit, doctor: doctor!(create_unit), citizen: verified_citizen!(1)).patient }
  let(:contact) { { "address" => ControlledPrescriptionHelpers::ADDRESS, "phone" => "41999990000" } }
  let(:today) { Date.new(2026, 10, 10) }

  def item(name, duration: 30, **over)
    { "catalog_item" => { "id" => catalog_items.fetch(name).id }, "quantity" => 30, "quantity_unit" => "comprimido", "route" => "oral",
      "dosage_instructions" => "1 comprimido ao dia", "duration_days" => duration }.merge(over.stringify_keys)
  end

  def call(items, role: :physician, controlled: true, identification: patient_identification, prescriber_contact: contact)
    input = { "items" => items, "patient_identification" => identification }.compact
    described_class.call(input, role: role, patient: patient, catalog: catalog_items.values.index_by(&:id),
                                matcher: Medications::Substances.matcher, today: today, controlled: controlled,
                                prescriber_contact: prescriber_contact)
  end

  it "RCE: categoria, identificação normalizada, contato, 30 dias, 2 vias, sncr ainda nulo" do
    # CPF mandado na entrada é ignorado: vale o do cadastro do paciente (Review Focus 5).
    result = call([ item(:fluoxetina) ], identification: patient_identification("cpf" => ControlledPrescriptionHelpers::PATIENT_CPF,
                                                                                "no_cpf" => true, "passport" => "AB123456"))
    expect(result).to be_ok
    expect(result.payload[:category]).to eq("special_control")
    expect(result.payload[:content]).to include("category" => "special_control", "sncr" => nil, "copies" => 2,
                                                "valid_until" => "2026-11-09", "prescriber_contact" => contact,
                                                "patient_identification" => { "cpf" => patient.cpf,
                                                                              "address" => ControlledPrescriptionHelpers::ADDRESS })
  end

  it "misturar categorias, listas A/B e C2/C3/C4 (com o índice)" do
    expect(call([ item(:fluoxetina), item(:losartana) ]).reason).to eq(:mixed_categories)
    expect(call([ item(:fluoxetina), item(:amoxicilina) ]).reason).to eq(:mixed_categories)
    expect(call([ item(:losartana), item(:diazepam) ]).then { |r| [ r.reason, r.details ] }).to eq([ :requires_notification, { index: 1 } ])
    expect(call([ item(:talidomida) ]).then { |r| [ r.reason, r.details ] }).to eq([ :not_supported, { index: 0 } ])
    expect(call([ item(:zidovudina) ]).reason).to eq(:not_supported)
    free = { "free_text" => "fluoxetina 20 mg", "quantity" => 30, "quantity_unit" => "cápsula", "route" => "oral",
             "dosage_instructions" => "1 ao dia", "duration_days" => 30 }
    expect(call([ free ]).reason).to eq(:controlled_not_allowed)
  end

  it "limites: 3 C1 + 1 C5 de 60 dias passa; 4 C1 não; 61 dias não; anticonvulsivante até 180; duração obrigatória (Review Focus 5)" do
    expect(call([ item(:fluoxetina, duration: 60), item(:sertralina, duration: 60), item(:amitriptilina, duration: 60),
                  item(:nandrolona, duration: 60) ])).to be_ok
    expect(call([ item(:fluoxetina), item(:sertralina), item(:amitriptilina), item(:carbamazepina) ]).reason).to eq(:too_many_c1_substances)
    expect(call([ item(:fluoxetina, duration: 61) ]).then { |r| [ r.reason, r.details ] }).to eq([ :duration_exceeded, { index: 0 } ])
    expect(call([ item(:carbamazepina, duration: 180) ])).to be_ok
    expect(call([ item(:carbamazepina, duration: 181) ]).reason).to eq(:duration_exceeded)
    expect(call([ item(:fluoxetina, duration: nil) ]).then { |r| [ r.reason, r.details ] }).to eq([ :invalid_item, { index: 0, field: "duration_days" } ])
  end

  it "quem prescreve: médico e dentista; enfermeiro não" do
    expect(call([ item(:fluoxetina) ], role: :dentist)).to be_ok
    expect(call([ item(:fluoxetina) ], role: :nurse).reason).to eq(:cbo_not_allowed)
    expect(call([ item(:fluoxetina) ], role: :other).reason).to eq(:cbo_not_allowed)
  end

  it "identificação e contato: obrigatórios na RCE, com o campo exato (Review Focus 5)" do
    expect(call([ item(:fluoxetina) ], identification: nil).then { |r| [ r.reason, r.details ] })
      .to eq([ :patient_identification_required, { field: "patient_identification" } ])
    expect(call([ item(:fluoxetina) ], identification: patient_identification("address" => nil)).details).to eq(field: "patient_identification.address")
    expect(call([ item(:fluoxetina) ], identification: patient_identification("address" => ControlledPrescriptionHelpers::ADDRESS.merge("zip" => "80010-000"))).details)
      .to eq(field: "patient_identification.address.zip")
    allow(patient).to receive(:cpf).and_return(nil) # cadastro sem CPF: a RCE não sai (o 19d não tem "não possui")
    expect(call([ item(:fluoxetina) ]).then { |r| [ r.reason, r.details ] })
      .to eq([ :patient_identification_required, { field: "patient_identification.cpf" } ])
    expect(call([ item(:fluoxetina) ], prescriber_contact: nil).reason).to eq(:prescriber_address_missing)
  end

  it "antimicrobiano: 10 dias, 2 vias, identificação opcional; contato só de quem não é enfermeiro" do
    result = call([ item(:amoxicilina, duration: 7) ], identification: nil)
    expect(result.payload[:content]).to include("category" => "antimicrobial", "valid_until" => "2026-10-20", "copies" => 2,
                                                "patient_identification" => nil, "prescriber_contact" => contact, "antimicrobial" => true)
    # O enfermeiro passa ainda pela trava do protocolo do 19c; a regra do contato é da categoria:
    nurse = ClinicalDocuments::Content::Controlled.check("antimicrobial", [], role: :nurse, identification: nil, prescriber_contact: contact,
                                                                              patient: patient)
    expect(nurse.payload[:prescriber_contact]).to be_nil
  end

  it "comum: sem identificação nem contato, validade do 19c" do
    content = call([ item(:losartana, duration: nil, continuous: true) ], identification: nil).payload[:content]
    expect(content).to include("category" => "common", "copies" => 1, "sncr" => nil, "patient_identification" => nil, "prescriber_contact" => nil)
    expect(content).not_to have_key("valid_until")
  end

  it "interruptor desligado: o 19c (controlado recusado, antimicrobiano marcado), identificação ignorada" do
    expect(call([ item(:fluoxetina) ], controlled: false).then { |r| [ r.reason, r.details ] }).to eq([ :controlled_not_allowed, { index: 0 } ])
    content = call([ item(:amoxicilina) ], controlled: false, prescriber_contact: nil).payload[:content]
    expect(content).to include("category" => "antimicrobial", "copies" => 2, "patient_identification" => nil, "prescriber_contact" => nil)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `rspec spec/services/clinical_documents/controlled_content_spec.rb`
Expected: FAIL (`unknown keyword: :controlled`).

- [ ] **Step 3: Implemente**

```ruby
# app/services/clinical_documents/content/patient_identification.rb
# Identificação do paciente na RCE (ADR 0034; spec §5; decisão do usuário,
# 2026-10-10): o CPF é SEMPRE o do cadastro do paciente, preenchido aqui (a
# entrada não manda CPF; o 19d não tem "não possui" nem passaporte), e o
# endereço completo, digitado na receita. Na RET é opcional (P&R 23): quando
# vem, vem inteira. Erros com o campo exato; nunca o valor.
module ClinicalDocuments
  module Content
    module PatientIdentification
      ROOT = "patient_identification".freeze

      module_function

      def call(value, required:, patient:)
        if value.nil?
          return required ? missing(ROOT) : Result.ok(value: nil)
        end
        return invalid(ROOT) unless value.is_a?(Hash)

        value = value.to_h.transform_keys(&:to_s)
        cpf = CitizenIdentity::Cpf.normalize(patient&.cpf.to_s)
        return missing("#{ROOT}.cpf") unless cpf
        return missing("#{ROOT}.address") if value["address"].nil?

        address, field = Address.normalize(value["address"])
        return invalid("#{ROOT}.address.#{field}") unless address

        Result.ok(value: { "cpf" => cpf, "address" => address })
      end

      def missing(field) = Result.fail(:patient_identification_required, details: { field: field })
      def invalid(field) = Result.fail(:invalid_content, details: { field: field })
      private_class_method :missing, :invalid
    end
  end
end
```

```ruby
# app/services/clinical_documents/content/controlled.rb
# Categoria da receita e as regras de cada uma (ADR 0034; spec §5; Portaria
# 344/1998 arts. 52, 57, 59; RDC 471/2021; Valores fixados 3–4). A categoria
# sai dos itens do catálogo (lista mais restritiva e marcas do 19c); controlado
# em texto livre é recusado. Desligado o interruptor, `legacy` repete o 19c.
module ClinicalDocuments
  module Content
    module Controlled
      SPECIAL = %w[C1 C5].freeze
      NOTIFICATION = %w[A1 A2 A3 B1 B2].freeze
      UNSUPPORTED = %w[C2 C3 C4].freeze
      MAX_C1 = 3
      SPECIAL_DAYS = 60
      ANTICONVULSANT_DAYS = 180
      VALIDITY = { "special_control" => 30, "antimicrobial" => 10 }.freeze
      PRESCRIBERS = %i[physician dentist].freeze

      module_function

      def classify(items)
        categories = items.each_with_index.map do |it, index|
          list = it[:controlled_list]
          if it[:controlled] && it[:free_text] then return Result.fail(:controlled_not_allowed, details: { index: index })
          elsif NOTIFICATION.include?(list) then return Result.fail(:requires_notification, details: { index: index })
          elsif UNSUPPORTED.include?(list) then return Result.fail(:not_supported, details: { index: index })
          elsif SPECIAL.include?(list) then "special_control"
          elsif it[:antimicrobial] then "antimicrobial"
          else "common"
          end
        end
        categories.uniq.one? ? Result.ok(category: categories.first) : Result.fail(:mixed_categories)
      end

      def legacy(items)
        items.each_with_index { |it, index| return Result.fail(:controlled_not_allowed, details: { index: index }) if it[:controlled] }
        Result.ok(category: items.any? { |it| it[:antimicrobial] } ? "antimicrobial" : "common")
      end

      def check(category, items, role:, identification:, prescriber_contact:, patient:)
        case category
        when "special_control" then special(items, role, identification, prescriber_contact, patient)
        when "antimicrobial"
          identified = PatientIdentification.call(identification, required: false, patient: patient)
          return identified if identified.failure?

          Result.ok(patient_identification: identified.payload[:value], prescriber_contact: (prescriber_contact unless role == :nurse))
        else
          Result.ok(patient_identification: nil, prescriber_contact: nil)
        end
      end

      def special(items, role, identification, prescriber_contact, patient)
        return Result.fail(:cbo_not_allowed) unless PRESCRIBERS.include?(role)
        return Result.fail(:too_many_c1_substances) if items.count { |it| it[:controlled_list] == "C1" } > MAX_C1

        items.each_with_index do |it, index|
          return Result.fail(:invalid_item, details: { index: index, field: "duration_days" }) if it[:duration_days].nil?

          limit = it[:anticonvulsant] ? ANTICONVULSANT_DAYS : SPECIAL_DAYS
          return Result.fail(:duration_exceeded, details: { index: index }) if it[:duration_days] > limit
        end
        identified = PatientIdentification.call(identification, required: true, patient: patient)
        return identified if identified.failure?
        return Result.fail(:prescriber_address_missing) unless prescriber_contact

        Result.ok(patient_identification: identified.payload[:value], prescriber_contact: prescriber_contact)
      end

      def validity(category, today) = VALIDITY.key?(category) ? today + VALIDITY.fetch(category) : nil
      private_class_method :special
    end
  end
end
```

Em `app/services/clinical_documents/content/prescription.rb`:

1. A assinatura de `call` ganha `controlled: false, prescriber_contact: nil`.
2. Troque a linha `items.each_with_index { |it, index| return Result.fail(:controlled_not_allowed, details: { index: index }) if it[:controlled] }` por:

```ruby
        classified = controlled ? Controlled.classify(items) : Controlled.legacy(items)
        return classified if classified.failure?

        category = classified.payload[:category]
        rules = Controlled.check(category, items, role: role, identification: (input["patient_identification"] if controlled),
                                                  prescriber_contact: (prescriber_contact if controlled), patient: patient)
        return rules if rules.failure?
```
3. Troque `antimicrobial = items.any? { |it| it[:antimicrobial] }` e `valid_until = validity(input, antimicrobial, today)` por:

```ruby
        antimicrobial = items.any? { |it| it[:antimicrobial] }
        valid_until = Controlled.validity(category, today) || validity(input, false, today)
        return valid_until if valid_until.is_a?(Result)
```
4. Troque `"copies" => antimicrobial ? 2 : 1` por `"copies" => category == "common" ? 1 : 2` e, depois do `.compact` do `content` (os nulos abaixo são do contrato §2 e não podem ser compactados):

```ruby
        content.merge!("category" => category, "sncr" => nil, "patient_identification" => rules.payload[:patient_identification],
                       "prescriber_contact" => rules.payload[:prescriber_contact])
```
5. O `Result.ok` final passa a `Result.ok(content: content, items: rows, antimicrobial: antimicrobial, category: category)`.
6. No hash de `item(...)`, acrescente `controlled_list: flags.controlled_list, anticonvulsant: flags.anticonvulsant == true` (o item do catálogo responde `controlled_list`/`anticonvulsant` como o `Flags`).
7. O comentário do topo passa a citar o ADR 0034 ("controlado pela categoria quando `controlled_prescriptions` está utilizável; desligado, nenhum controlado").

- [ ] **Step 4: Rode e veja passar**

Run: `rspec spec/services/clinical_documents/controlled_content_spec.rb spec/services/clinical_documents/prescription_content_spec.rb`
Expected: PASS. Se uma asserção do 19c fixar o `content` inteiro (`eq`), acrescente as quatro chaves novas (é o contrato §2 do 19d); nunca afrouxe para `include` só para passar.

- [ ] **Step 5: Commit**

```bash
git add app/services/clinical_documents/content/controlled.rb app/services/clinical_documents/content/patient_identification.rb app/services/clinical_documents/content/prescription.rb spec/services/clinical_documents/controlled_content_spec.rb spec/services/clinical_documents/prescription_content_spec.rb
git commit -m "feat: classify prescriptions by category and apply the special control and antimicrobial rules

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
### Task 12: Emissão com número SNCR, modo com motivo e a última identificação do paciente

**Files:**
- Modify: `app/commands/clinical_documents/issue.rb`, `app/controllers/clinical_documents_controller.rb`, `config/routes.rb`
- Test: `spec/commands/clinical_documents/controlled_issue_spec.rb`, `spec/requests/controlled_documents_spec.rb`

**Interfaces:**
- Consumes: `ControlledPrescriptions::Gate`, `Sncr::{Mock,Stock}`, `Content::Prescription` (Task 11), `Professional#prescriber_contact`, `Signatures::{Gate,OpenRequest}`, `SignerCertificate.active`, `Issuers.role`.
- Produces:
  - `ClinicalDocuments::Issue.call(...)` (mesma assinatura do 19c) → `ok(document:, paper_reason: String|nil)`; falhas novas `:feature_disabled` (com `feature`), `:prescriber_address_missing` e as da Task 11.
  - `ClinicalDocuments::Issue::Context = Data.define(:signing, :controlled, :simulated, :platform, :contact)`; `ClinicalDocuments::Issue.decide(kind:, category:, role:, ctx:, certificate:) -> [Symbol, Symbol|nil]` (público, Desvio 10); `SNCR_KINDS = { "special_control" => "rce", "antimicrobial" => "ret" }`.
  - `content.sncr = { "kind", "number", "simulated" }` no modo digital de RCE/RET; número `used` com o documento na mesma transação; `clinical_document.issued` com `category` na receita.
  - `paper_reason` gravado no documento e devolvido em todo `<document>` (`ClinicalDocuments::Json.document`), no `POST` e nas leituras (`GET /attendance/documents/:id`, a lista da consulta); `GET /attendance/consultations/:id/patient_identification` (autora da consulta) → `{ patient_identification: {…}|null }` da última receita emitida do paciente que tenha identificação.

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/commands/clinical_documents/controlled_issue_spec.rb
require "rails_helper"

# Spec §5–§6; ADR 0034 (Invariantes; Desvios 9–11): a RCE/RET digital tira um
# número do estoque do modo corrente na transação da emissão e nasce com o
# pedido de assinatura; sem número, certificado ou assinatura, papel com o
# motivo; a falha devolve o número (Review Focus 3); nada de número no evento.
RSpec.describe "Emissão de receita controlada" do
  before do
    controlled_city!(mock: true)
    ciap2_release!; cid10_release!; sigtap_release!; city_profile!
    import_test_catalog!
  end

  let(:unit) { create_unit }
  let(:doctor) { signer_doctor!(unit).tap { |user| prescriber_contact!(user) } }
  let(:consultation) { finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1)) }
  let(:fluoxetina) { controlled_item!(code: 9_900_001, ingredient: "FLUOXETINA", list: "C1") }

  def rce_content
    { "items" => [ { "catalog_item" => { "id" => fluoxetina.id }, "quantity" => 30, "quantity_unit" => "cápsula", "route" => "oral",
                     "dosage_instructions" => "1 cápsula pela manhã", "duration_days" => 30, "continuous" => true } ],
      "patient_identification" => patient_identification }
  end

  def ret_content
    { "items" => [ { "catalog_item" => { "id" => catalog_item(271_089).id }, "quantity" => 21, "quantity_unit" => "cápsula",
                     "route" => "oral", "dosage_instructions" => "1 cápsula de 8 em 8 horas", "duration_days" => 7 } ] }
  end

  def issue!(content = rce_content, kind: "prescription", by: doctor)
    ClinicalDocuments::Issue.call(kind: kind, content: content, by: by, consultation: consultation)
  end

  it "RCE digital: número do estoque, pedido de assinatura, eventos só com ids" do
    linked_certificate!(doctor)
    sncr_stock!(doctor, kind: "rce", count: 2)
    result = issue!
    document = result.payload[:document]
    expect(result.payload[:paper_reason]).to be_nil
    expect(document.issue_mode).to eq("digital")
    expect(document.content_data["sncr"]).to eq("kind" => "rce", "number" => "2610.1-41.0000001", "simulated" => true)
    expect(document.content_data["category"]).to eq("special_control")
    expect(SncrNumber.find_by(document_id: document.id)).to have_attributes(status: "used", number: "2610.1-41.0000001")
    expect(document.signature_request).to be_present
    issued = DomainEvent.where(name: "clinical_document.issued").order(:created_at).last.payload
    expect(issued).to include("document_id" => document.id, "category" => "special_control", "issue_mode" => "digital")
    used = DomainEvent.where(name: "sncr.number_used").order(:created_at).last.payload
    expect(used).to eq("document_id" => document.id, "kind" => "rce")
    expect(DomainEvent.where(name: %w[clinical_document.issued sncr.number_used]).pluck(:payload).to_json).not_to include("2610.1-41")
  end

  it "duas emissões com 1 número: a segunda sai em papel com no_sncr_number (Review Focus 3)" do
    linked_certificate!(doctor)
    sncr_stock!(doctor, kind: "rce", count: 1)
    first = issue!.payload
    second = issue!.payload
    expect([ first[:document].issue_mode, first[:paper_reason] ]).to eq([ "digital", nil ])
    expect([ second[:document].issue_mode, second[:paper_reason] ]).to eq([ "paper", "no_sncr_number" ])
    expect(second[:document].content_data["sncr"]).to be_nil
    expect(second[:document].signature_request).to be_nil
    expect(SncrNumber.where(document_id: [ first[:document].id, second[:document].id ]).count).to eq(1)
  end

  it "falha depois de tirar o número: rollback, número livre, nenhum documento (Review Focus 3)" do
    linked_certificate!(doctor)
    sncr_stock!(doctor, kind: "rce", count: 1)
    allow(Signatures::OpenRequest).to receive(:call).and_raise(ActiveRecord::StatementInvalid, "falha simulada")
    expect { issue! }.to raise_error(ActiveRecord::StatementInvalid)
    expect(SncrNumber.where(user_id: doctor.id).pluck(:status)).to eq([ "free" ])
    expect(ClinicalDocument.where(consultation_id: consultation.id).count).to eq(0)
  end

  it "sem certificado: papel no_certificate e o número fica livre; sem assinatura: signature_unavailable" do
    sncr_stock!(doctor, kind: "rce", count: 1)
    result = issue!.payload
    expect([ result[:document].issue_mode, result[:paper_reason] ]).to eq([ "paper", "no_certificate" ])
    expect(SncrNumber.where(user_id: doctor.id).pluck(:status)).to eq([ "free" ])
    Platform::Features.set!(city: Current.city, key: "digital_signature", enabled: false, maintainer: ledi_maintainer!)
    expect(issue!.payload[:paper_reason]).to eq("signature_unavailable")
  end

  it "número simulado não serve com o sncr_mock desligado (Review Focus 4)" do
    linked_certificate!(doctor)
    sncr_stock!(doctor, kind: "rce", count: 1, simulated: true)
    Platform::Features.set!(city: Current.city, key: "sncr_mock", enabled: false, maintainer: ledi_maintainer!)
    result = issue!.payload
    expect([ result[:document].issue_mode, result[:paper_reason] ]).to eq([ "paper", "no_sncr_number" ])
  end

  it "RET digital: número ret; sem contato → 422 e o número volta" do
    linked_certificate!(doctor)
    sncr_stock!(doctor, kind: "ret", count: 1)
    result = issue!(ret_content).payload
    expect(result[:document].content_data).to include("category" => "antimicrobial", "copies" => 2,
                                                      "sncr" => include("kind" => "ret", "number" => "2610.2-41.0000001"))
    sncr_stock!(doctor, kind: "ret", count: 1)
    doctor.professional.update!(prescriber_address: nil)
    expect(issue!(ret_content).reason).to eq(:prescriber_address_missing)
    expect(SncrNumber.where(user_id: doctor.id, kind: "ret", status: "free").count).to eq(1)
  end

  it "interruptor desligado: antimicrobiano em papel (feature_disabled), controlado recusado" do
    linked_certificate!(doctor)
    Platform::Features.set!(city: Current.city, key: "controlled_prescriptions", enabled: false, maintainer: ledi_maintainer!)
    result = issue!(ret_content).payload
    expect([ result[:document].issue_mode, result[:paper_reason] ]).to eq([ "paper", "feature_disabled" ])
    expect(issue!.reason).to eq(:controlled_not_allowed)
  end

  it "a decisão do modo (Desvio 10)" do
    ctx = ->(signing: true, controlled: true) { ClinicalDocuments::Issue::Context.new(signing: signing, controlled: controlled, simulated: true, platform: nil, contact: nil) }
    decide = ->(category, role: :physician, certificate: true, **opts) do
      ClinicalDocuments::Issue.decide(kind: "prescription", category: category, role: role, ctx: ctx.(**opts), certificate: certificate)
    end
    expect(decide.("antimicrobial", controlled: false)).to eq([ :paper, :feature_disabled ])
    expect(decide.("antimicrobial", role: :nurse)).to eq([ :paper, :nurse_antimicrobial ])
    expect(decide.("special_control", signing: false)).to eq([ :paper, :signature_unavailable ])
    expect(decide.("special_control", certificate: false)).to eq([ :paper, :no_certificate ])
    expect(decide.("common")).to eq([ :digital, nil ])
    expect(ClinicalDocuments::Issue.decide(kind: "controlled_notification_record", category: nil, role: :physician, ctx: ctx.(), certificate: true))
      .to eq([ :paper, nil ])
  end
end
```

```ruby
# spec/requests/controlled_documents_spec.rb
require "rails_helper"

# Contrato §2: a emissão grava e devolve paper_reason; erros novos com status e
# campo; a última identificação do paciente para "reaproveitar o último".
RSpec.describe "Receita controlada pela API", type: :request do
  before do
    controlled_city!(mock: true)
    ciap2_release!; cid10_release!; sigtap_release!; city_profile!
    import_test_catalog!
  end

  let(:unit) { create_unit }
  let(:doctor) { doctor!(unit).tap { |user| prescriber_contact!(user) } }
  let(:consultation) { finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1)) }
  let(:fluoxetina) { controlled_item!(code: 9_900_001, ingredient: "FLUOXETINA", list: "C1") }
  def body = JSON.parse(response.body)

  def rce(**over)
    { "items" => [ { "catalog_item" => { "id" => fluoxetina.id }, "quantity" => 30, "quantity_unit" => "cápsula", "route" => "oral",
                     "dosage_instructions" => "1 cápsula pela manhã", "duration_days" => 30 } ],
      "patient_identification" => patient_identification }.merge(over.stringify_keys)
  end

  def post_document(content, kind: "prescription")
    post "/attendance/consultations/#{consultation.id}/documents", params: { kind: kind, content: content }, as: :json
  end

  it "201 com paper_reason gravado e devolvido na leitura; a última identificação volta para a próxima receita" do
    sign_in_as(doctor)
    get "/attendance/consultations/#{consultation.id}/patient_identification"
    expect(body).to eq("patient_identification" => nil)
    post_document(rce)
    expect(response).to have_http_status(:created)
    expect(body).to include("issue_mode" => "paper", "paper_reason" => "no_certificate")
    expect(ClinicalDocument.find(body["id"]).paper_reason).to eq("no_certificate")
    get "/attendance/documents/#{body['id']}"
    expect(body["paper_reason"]).to eq("no_certificate")
    get "/attendance/consultations/#{consultation.id}/documents"
    expect(body["items"].first["paper_reason"]).to eq("no_certificate")
    expect(body["content"]).to include("category" => "special_control", "sncr" => nil)
    get "/attendance/consultations/#{consultation.id}/patient_identification"
    expect(body["patient_identification"]).to eq("cpf" => consultation.patient.cpf, "address" => ControlledPrescriptionHelpers::ADDRESS)
  end

  it "erros com status e campo; nada do paciente no log" do
    sign_in_as(doctor)
    log = capture_log do
      post_document(rce("patient_identification" => patient_identification("address" => ControlledPrescriptionHelpers::ADDRESS.merge("zip" => "80010-000"))))
    end
    expect([ response.status, body ]).to eq([ 422, { "error" => "invalid_content", "field" => "patient_identification.address.zip" } ])
    expect(log).not_to include(consultation.patient.cpf, "Araucárias")
    post_document(rce.merge("items" => rce["items"] * 4))
    expect([ response.status, body["error"] ]).to eq([ 422, "too_many_c1_substances" ])
    doctor.professional.update!(prescriber_address: nil)
    post_document(rce)
    expect([ response.status, body["error"] ]).to eq([ 422, "prescriber_address_missing" ])
  end

  it "outra pessoa não lê a identificação" do
    sign_in_as(doctor!(unit))
    get "/attendance/consultations/#{consultation.id}/patient_identification"
    expect([ response.status, body["error"] ]).to eq([ 403, "not_author" ])
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `rspec spec/commands/clinical_documents/controlled_issue_spec.rb spec/requests/controlled_documents_spec.rb`
Expected: FAIL (`uninitialized constant ClinicalDocuments::Issue::Context`).

- [ ] **Step 3: Implemente — `app/commands/clinical_documents/issue.rb`**

Substitua `call`, `prescription_platform`, `issue` e `build` (e acrescente `decide` e `notification_medication!`) por estas versões; `replaced`, `create!`, `medications!` e `invalid` ficam como no 19c (se o 19c entregou nomes diferentes, mantenha os dele e anote — Task 0, Step 6):

```ruby
# app/commands/clinical_documents/issue.rb
# Emissão de documento clínico (ADR 0033 + ADR 0034; spec 19c §4–§5, 19d
# §5–§6). Interruptores, catálogo, listas, credenciais e o contato do
# prescritor lidos FORA da transação. Numa transação só: documento, itens da
# receita, número SNCR (tirado com SKIP LOCKED e marcado used — a falha
# devolve por rollback), eventos da lista de medicamentos, pedido de
# assinatura e eventos. O modo é decidido aqui e nunca convertido; o motivo
# do papel volta na resposta (Desvio 10).
module ClinicalDocuments
  module Issue
    UUID = /\A\h{8}-\h{4}-\h{4}-\h{4}-\h{12}\z/
    DECLARATION_DAYS = 30
    SNCR_KINDS = { "special_control" => "rce", "antimicrobial" => "ret" }.freeze
    CATALOG_KINDS = %w[prescription controlled_notification_record].freeze

    Context = Data.define(:signing, :controlled, :simulated, :platform, :contact) do
      def inspect = "#<ClinicalDocuments::Issue::Context signing=#{signing} controlled=#{controlled} simulated=#{simulated}>"
    end

    module_function

    def call(kind:, content:, by:, consultation: nil, attendance: nil, replaces_document_id: nil, now: Time.current)
      kind = kind.to_s
      return invalid("kind") unless ClinicalDocument::KINDS.include?(kind)
      return invalid("kind") if attendance && kind != "attendance_declaration"

      controlled = consultation ? ControlledPrescriptions::Gate.usable?(Current.city) : false
      if kind == "controlled_notification_record" && !controlled
        return Result.fail(:feature_disabled, details: { feature: ControlledPrescriptions::Gate::KEY })
      end

      platform = CATALOG_KINDS.include?(kind) ? prescription_platform(content) : nil
      return platform if platform.is_a?(Result)

      ctx = Context.new(signing: consultation ? Signatures::Gate.usable?(Current.city) : false, controlled: controlled,
                        simulated: controlled && Sncr::Mock.on?(Current.city), platform: platform,
                        contact: (by.professional&.prescriber_contact if consultation))
      result = nil
      ApplicationRecord.transaction(requires_new: true) do
        result = issue(kind, content, by, consultation, attendance, replaces_document_id, ctx, now)
        raise ActiveRecord::Rollback if result.failure?
      end
      result
    end

    # Desvio 10: o motivo do papel, na ordem fixada.
    def decide(kind:, category:, role:, ctx:, certificate:)
      return [ :paper, nil ] if kind == "controlled_notification_record"

      if category == "antimicrobial"
        return [ :paper, :feature_disabled ] unless ctx.controlled
        return [ :paper, :nurse_antimicrobial ] if role == :nurse
      end
      return [ :paper, :signature_unavailable ] unless ctx.signing
      return [ :paper, :no_certificate ] unless certificate

      [ :digital, nil ]
    end

    def prescription_platform(content)
      return Result.fail(:catalog_unavailable) unless Medications::Search.available?

      input = Content::Input.hash(content)
      entries = Array(input["items"]) + [ input["item"] ].compact
      ids = entries.filter_map do |item|
        reference = item.is_a?(Hash) && item["catalog_item"].is_a?(Hash) ? item["catalog_item"]["id"].to_s : nil
        reference if reference&.match?(UUID)
      end
      { catalog: MedicationCatalogItem.where(id: ids.uniq).index_by(&:id), matcher: Medications::Substances.matcher }
    end

    def issue(kind, input, by, consultation, attendance, replaces_id, ctx, now)
      if consultation
        return Result.fail(:not_author) unless consultation.author_user_id == by.id

        attendance = consultation.attendance
        cbo = consultation.cbo_code
        return Result.fail(:cbo_not_allowed) unless Issuers.allowed?(kind, cbo)

        patient = Patient.lock.find(consultation.patient_id)
      else
        status, cbo = Issuers.declaration_staff(by, attendance)
        return Result.fail(status) unless status == :ok
        return Result.fail(:attendance_not_found_or_closed_long_ago) if attendance.checked_in_at < now - DECLARATION_DAYS.days

        patient = attendance.citizen.patient_id && Patient.lock.find(attendance.citizen.patient_id)
      end
      replaces = replaced(replaces_id, kind, attendance, by)
      return replaces if replaces.is_a?(Result)

      built = build(kind, input, consultation, attendance, patient, cbo, ctx, now)
      return built if built.failure?

      content, rows, category = built.payload.values_at(:content, :items, :category)
      mode, reason =
        if consultation
          decide(kind: kind, category: category, role: Issuers.role(cbo), ctx: ctx,
                 certificate: SignerCertificate.active.exists?(user_id: by.id))
        else
          [ :paper, nil ]
        end
      number = nil
      if mode == :digital && SNCR_KINDS.key?(category)
        number = Sncr::Stock.take!(user: by, kind: SNCR_KINDS.fetch(category), simulated: ctx.simulated)
        mode, reason = :paper, :no_sncr_number unless number
      end
      return Result.fail(:prescriber_address_missing) if category == "antimicrobial" && mode == :digital && ctx.contact.nil?

      content = content.merge("sncr" => { "kind" => number.kind, "number" => number.number, "simulated" => number.simulated }) if number
      document = create!(kind: kind, patient_id: patient&.id, citizen_id: attendance.citizen_id, consultation: consultation,
                         attendance: attendance, author_user_id: by.id, cbo_code: cbo, issue_mode: mode.to_s,
                         paper_reason: (reason&.to_s if consultation && mode == :paper),
                         replaces_document_id: replaces&.id, issued_at: now, content: content.to_json)
      rows.each { |row| PrescriptionItem.create!(row.merge(clinical_document: document)) }
      Sncr::Stock.use!(number, document: document, now: now) if number
      failure = kind == "controlled_notification_record" ? notification_medication!(document, patient, content, ctx, by, now) : medications!(document, patient, rows, ctx.platform, by, now)
      return failure if failure

      Signatures::OpenRequest.call(document, now: now, usable: true) if mode == :digital
      publish!(document, kind, category, consultation, attendance, content)
      Result.ok(document: document, paper_reason: (reason&.to_s if consultation))
    end

    def publish!(document, kind, category, consultation, attendance, content)
      link = consultation ? { consultation_id: consultation.id } : { attendance_id: attendance.id }
      payload = { document_id: document.id, kind: kind, **link, issue_mode: document.issue_mode }
      payload[:category] = category if kind == "prescription"
      DomainEvents.publish("clinical_document.issued", **payload)
      return unless kind == "controlled_notification_record"

      DomainEvents.publish("controlled_notification.recorded", document_id: document.id,
                                                               notification_type: content.fetch("notification_type"))
    end

    def build(kind, input, consultation, attendance, patient, cbo, ctx, now)
      today = now.in_time_zone.to_date
      result =
        case kind
        when "sick_note" then Content::SickNote.call(input, today: today)
        when "attendance_declaration" then Content::Declaration.call(input, attendance: attendance)
        when "exam_requisition" then Content::ExamRequisition.call(input, consultation: consultation)
        when "controlled_notification_record"
          return Content::NotificationRecord.call(input, role: Issuers.role(cbo), catalog: ctx.platform[:catalog])
        when "prescription"
          return Content::Prescription.call(input, role: Issuers.role(cbo), patient: patient, catalog: ctx.platform[:catalog],
                                                   matcher: ctx.platform[:matcher], today: today, city_cnpj: CityProfile.current&.cnpj,
                                                   controlled: ctx.controlled, prescriber_contact: ctx.contact)
        end
      result.ok? ? Result.ok(content: result.payload[:content], items: [], category: nil) : result
    end

    # O registro da Notificação entra na lista de medicamentos (spec §5), sem uso contínuo.
    def notification_medication!(document, patient, content, ctx, by, now)
      item = content.fetch("item")
      result = Patients::ApplyMedicationEvent.call(
        patient: patient, action: "add", by: by, source: { document: document }, origin: "prescription",
        catalog_item: ctx.platform[:catalog].fetch(item.dig("catalog_item", "id")), dosage_summary: item["dosage_instructions"],
        continuous: false, on_active: :change, on: now.in_time_zone.to_date
      )
      result.failure? ? result : nil
    end
    private_class_method :prescription_platform, :issue, :publish!, :build, :notification_medication!
  end
end
```

(`replaced`, `create!`, `medications!` e `invalid` do 19c continuam no módulo, com o `private_class_method` deles.)

> `Content::NotificationRecord` nasce na Task 13; até lá o ramo não é exercitado (o `KINDS` aceita o tipo, mas `Issuers.allowed?` o recusa — Task 13 libera).

- [ ] **Step 4: Controller e rota**

Em `app/controllers/clinical_documents_controller.rb`:

```ruby
  def respond_created(result)
    return render_failure(result, ERROR_STATUS) if result.failure?

    render json: ClinicalDocuments::Json.document(result.payload[:document]), status: :created
  end

  # (o 19c já tinha este respond_created; não muda: o paper_reason vem do Json.document)

  # "Reaproveitar o último" (spec §5; Divergência D4): a identificação da
  # última receita emitida do paciente, só para a autora da consulta.
  def patient_identification
    consultation = find(Consultation, params[:id])
    return not_found unless consultation
    return forbid("not_author") unless consultation.author_user_id == Current.user.id

    last = ClinicalDocument.where(patient_id: consultation.patient_id, kind: "prescription", status: "issued")
                           .order(issued_at: :desc).limit(50).lazy.map(&:content_data)
                           .find { |content| content["patient_identification"].present? }
    render json: { patient_identification: last && last["patient_identification"] }
  end
```
e `require_staff` passa a aceitar `patient_identification` como as escritas (`policy.care?`).

Em `app/services/clinical_documents/json.rb#document`, acrescente ao hash `paper_reason: document.paper_reason` (Desvio 10: gravado e devolvido em toda leitura). Em `config/routes.rb`, junto das rotas de documentos do 19c:

```ruby
  get "/attendance/consultations/:id/patient_identification", to: "clinical_documents#patient_identification"
```

- [ ] **Step 5: Rode e veja passar**

Run: `rspec spec/commands/clinical_documents spec/requests/controlled_documents_spec.rb spec/requests/clinical_documents_spec.rb`
Expected: PASS (as specs do 19c continuam verdes; a resposta da emissão ganhou `paper_reason`).

- [ ] **Step 6: Commit**

```bash
git add app/commands/clinical_documents/issue.rb app/controllers/clinical_documents_controller.rb config/routes.rb spec/commands/clinical_documents/controlled_issue_spec.rb spec/requests/controlled_documents_spec.rb
git commit -m "feat: issue special control and antimicrobial prescriptions with an SNCR number or a paper reason

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

## Fatia 4 — Registro da Notificação (F-19.28)

### Task 13: `controlled_notification_record`

**Files:**
- Create: `app/services/clinical_documents/content/notification_record.rb`
- Modify: `app/services/clinical_documents/issuers.rb`, `app/services/clinical_documents/json.rb`, `app/services/clinical_documents/verification.rb`, `app/controllers/clinical_documents_controller.rb` (impresso), `app/services/signatures/canonical.rb` (guarda)
- Test: `spec/services/clinical_documents/notification_record_spec.rb`, `spec/requests/notification_record_spec.rb`

**Interfaces:**
- Consumes: `ClinicalDocuments::Content::Input`, `ClinicalDocuments::Json.{catalog_ref,number}`, `PrescriptionItem::ROUTES`, `Professional::UFS`, `ClinicalDocuments::Issue` (Task 12), `Patients::ApplyMedicationEvent`.
- Produces:
  - `ClinicalDocuments::Content::NotificationRecord.call(input, role:, catalog:) -> Result ok(content:, items: [], category: nil)` — `content = { notification_type, paper_number, numbering_uf, item: { catalog_item: <ref>, printed_description, quantity, quantity_unit, route, dosage_instructions, duration_days } }`; falhas `cbo_not_allowed`, `invalid_content` (`field`), `item_not_in_notification_list`, `notification_type_mismatch`, `duration_exceeded`.
  - `NotificationRecord::TYPES = { "A" => %w[A1 A2 A3], "B" => %w[B1], "B2" => %w[B2] }`, `MAX_DAYS = { "A" => 30, "B" => 60, "B2" => 60 }`.
  - `Issuers.allowed?("controlled_notification_record", cbo)` só médico e dentista.
  - `<document>` do registro com `short_code: null`, `verification_url: null`, `signature: null`; `/v/:token` e `/v/lookup` não o acham; `GET /attendance/documents/:id/print` → 404 `not_found`.

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/services/clinical_documents/notification_record_spec.rb
require "rails_helper"

# Spec §5; contrato §3: o registro da Notificação de papel (talão da VISA) —
# tipo, número do talão, UF da numeração, uma substância das listas A/B do
# tipo certo, posologia e duração (A ≤ 30; B/B2 ≤ 60).
RSpec.describe ClinicalDocuments::Content::NotificationRecord do
  before { documents_city! }

  let(:items) do
    import_test_catalog!
    { morfina: controlled_item!(code: 9_900_010, ingredient: "MORFINA", list: "A1"),
      diazepam: controlled_item!(code: 9_900_003, ingredient: "DIAZEPAM", list: "B1"),
      sibutramina: controlled_item!(code: 9_900_011, ingredient: "SIBUTRAMINA", list: "B2"),
      fluoxetina: controlled_item!(code: 9_900_001, ingredient: "FLUOXETINA", list: "C1") }
  end

  def input(type: "B", item: :diazepam, duration: 30, **over)
    { "notification_type" => type, "paper_number" => "PR-1234567", "numbering_uf" => "PR",
      "item" => { "catalog_item" => { "id" => items.fetch(item).id }, "quantity" => 30, "quantity_unit" => "comprimido",
                  "route" => "oral", "dosage_instructions" => "1 comprimido à noite", "duration_days" => duration } }
      .merge(over.stringify_keys)
  end

  def call(value, role: :physician) = described_class.call(value, role: role, catalog: items.values.index_by(&:id))

  it "grava o conteúdo do contrato §3" do
    content = call(input).payload[:content]
    expect(content).to include("notification_type" => "B", "paper_number" => "PR-1234567", "numbering_uf" => "PR")
    expect(content["item"]).to include("quantity" => 30, "route" => "oral", "duration_days" => 30,
                                       "catalog_item" => include("id" => items[:diazepam].id))
  end

  it "item fora das listas A/B, tipo errado, duração além do limite" do
    expect(call(input(item: :fluoxetina)).reason).to eq(:item_not_in_notification_list)
    expect(call(input(type: "A", item: :diazepam)).reason).to eq(:notification_type_mismatch)
    expect(call(input(type: "B", item: :morfina)).reason).to eq(:notification_type_mismatch)
    expect(call(input(type: "B2", item: :sibutramina, duration: 60))).to be_ok
    expect(call(input(type: "A", item: :morfina, duration: 31)).reason).to eq(:duration_exceeded)
    expect(call(input(type: "B", duration: 61)).reason).to eq(:duration_exceeded)
  end

  it "campos inválidos com o nome; só médico e dentista" do
    expect(call(input(type: "C")).details).to eq(field: "notification_type")
    expect(call(input(paper_number: "")).details).to eq(field: "paper_number")
    expect(call(input(paper_number: "x" * 31)).details).to eq(field: "paper_number")
    expect(call(input(numbering_uf: "XX")).details).to eq(field: "numbering_uf")
    expect(call(input(duration: nil)).details).to eq(field: "item.duration_days")
    expect(call(input, role: :nurse).reason).to eq(:cbo_not_allowed)
    expect(call(input, role: :dentist)).to be_ok
  end
end
```

```ruby
# spec/requests/notification_record_spec.rb
require "rails_helper"

# Contrato §3, §7; spec §5: emitido pela autora da consulta, sempre papel,
# sem código nem conferência pública nem impresso; entra na lista de
# medicamentos; cancelável; evento só com ids.
RSpec.describe "Registro da Notificação de papel", type: :request do
  before do
    controlled_city!(mock: true)
    ciap2_release!; cid10_release!; sigtap_release!; city_profile!
    import_test_catalog!
  end

  let(:unit) { create_unit }
  let(:doctor) { signer_doctor!(unit) }
  let(:consultation) { finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1)) }
  let(:diazepam) { controlled_item!(code: 9_900_003, ingredient: "DIAZEPAM", list: "B1") }
  def body = JSON.parse(response.body)

  def record!
    post "/attendance/consultations/#{consultation.id}/documents",
         params: { kind: "controlled_notification_record",
                   content: { notification_type: "B", paper_number: "PR-1234567", numbering_uf: "PR",
                              item: { catalog_item: { id: diazepam.id }, quantity: 30, quantity_unit: "comprimido", route: "oral",
                                      dosage_instructions: "1 comprimido à noite", duration_days: 30 } } }, as: :json
  end

  it "registra em papel, sem código, na lista de medicamentos; evento só com ids" do
    linked_certificate!(doctor) # mesmo com certificado, papel
    sign_in_as(doctor)
    record!
    expect(response).to have_http_status(:created)
    expect(body).to include("kind" => "controlled_notification_record", "issue_mode" => "paper", "short_code" => nil,
                            "verification_url" => nil, "signature" => nil, "paper_reason" => nil)
    document = ClinicalDocument.find(body["id"])
    expect(document.signature_request).to be_nil
    expect(PatientMedication.where(patient_id: consultation.patient_id).map(&:label).join).to include("DIAZEPAM")
    event = DomainEvent.where(name: "controlled_notification.recorded").order(:created_at).last
    expect(event.payload).to eq("document_id" => document.id, "notification_type" => "B")

    get "/attendance/documents/#{document.id}/print"
    expect([ response.status, body["error"] ]).to eq([ 404, "not_found" ])
    get "/v/#{document.verification_token}", headers: { "Accept" => "application/json" }
    expect(response).to have_http_status(:not_found)

    sign_in_as(doctor).update!(mfa_verified_at: Time.current) # step-up do cancelamento (19c)
    post "/attendance/documents/#{document.id}/cancel", params: { reason: "Talão registrado errado" }, as: :json
    expect([ response.status, body["status"] ]).to eq([ 200, "cancelled" ])
  end

  it "interruptor desligado: 403 feature_disabled com a chave" do
    Platform::Features.set!(city: Current.city, key: "controlled_prescriptions", enabled: false, maintainer: ledi_maintainer!)
    sign_in_as(doctor)
    record!
    expect([ response.status, body ]).to eq([ 403, { "error" => "feature_disabled", "feature" => "controlled_prescriptions" } ])
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `rspec spec/services/clinical_documents/notification_record_spec.rb spec/requests/notification_record_spec.rb`
Expected: FAIL (`uninitialized constant ClinicalDocuments::Content::NotificationRecord`).

- [ ] **Step 3: Implemente**

```ruby
# app/services/clinical_documents/content/notification_record.rb
# Registro da Notificação de Receita de papel (ADR 0034; spec §5; contrato §3):
# o número do talão da VISA, a UF da numeração e UMA substância das listas
# A (A1–A3) ou B (B1, B2) do tipo certo, com quantidade, posologia e duração
# (Portaria 344: A ≤ 30 dias; B ≤ 60; B2 ≤ 60 — Valores fixados 3). Sem PDF,
# sem assinatura, sem conferência pública.
module ClinicalDocuments
  module Content
    module NotificationRecord
      TYPES = { "A" => %w[A1 A2 A3], "B" => %w[B1], "B2" => %w[B2] }.freeze
      MAX_DAYS = { "A" => 30, "B" => 60, "B2" => 60 }.freeze
      PAPER_NUMBER = %r{\A[0-9A-Za-z./-]{1,30}\z}
      PRESCRIBERS = %i[physician dentist].freeze

      module_function

      def call(input, role:, catalog:)
        return Result.fail(:cbo_not_allowed) unless PRESCRIBERS.include?(role)

        input = Input.hash(input)
        type = input["notification_type"]
        return Input.invalid("notification_type") unless TYPES.key?(type)

        paper = input["paper_number"].is_a?(String) ? input["paper_number"].strip : ""
        return Input.invalid("paper_number") unless paper.match?(PAPER_NUMBER)

        uf = input["numbering_uf"].to_s.upcase
        return Input.invalid("numbering_uf") unless Professional::UFS.include?(uf)

        item = Input.hash(input["item"])
        reference = item["catalog_item"].is_a?(Hash) ? item["catalog_item"]["id"].to_s : nil
        entry = reference && catalog[reference]
        return Input.invalid("item.catalog_item") unless entry && entry.status == "active" && !entry.hidden
        return Result.fail(:item_not_in_notification_list) unless TYPES.values.flatten.include?(entry.controlled_list)
        return Result.fail(:notification_type_mismatch) unless TYPES.fetch(type).include?(entry.controlled_list)

        fields = item_fields(item)
        return fields if fields.is_a?(Result)
        return Result.fail(:duration_exceeded) if fields[:duration_days] > MAX_DAYS.fetch(type)

        content = { "notification_type" => type, "paper_number" => paper, "numbering_uf" => uf,
                    "item" => { "catalog_item" => Json.catalog_ref(entry).deep_stringify_keys, "printed_description" => entry.label,
                                "quantity" => Json.number(fields[:quantity]), "quantity_unit" => fields[:quantity_unit],
                                "route" => fields[:route], "dosage_instructions" => fields[:dosage_instructions],
                                "duration_days" => fields[:duration_days] } }
        Result.ok(content: content, items: [], category: nil)
      end

      def item_fields(item)
        quantity = item["quantity"].is_a?(Numeric) || item["quantity"].to_s.match?(/\A\d+(\.\d+)?\z/) ? BigDecimal(item["quantity"].to_s) : nil
        return Input.invalid("item.quantity") unless quantity && quantity.positive? && quantity <= 9999 && quantity.round(2) == quantity

        unit = Input.text(item["quantity_unit"], 1..30)
        return Input.invalid("item.quantity_unit") unless unit.is_a?(String)
        return Input.invalid("item.route") unless PrescriptionItem::ROUTES.include?(item["route"])

        dosage = Input.text(item["dosage_instructions"], 3..500)
        return Input.invalid("item.dosage_instructions") unless dosage.is_a?(String)

        duration = item["duration_days"]
        return Input.invalid("item.duration_days") unless duration.is_a?(Integer) && duration.between?(1, 365)

        { quantity: quantity, quantity_unit: unit, route: item["route"], dosage_instructions: dosage, duration_days: duration }
      end
      private_class_method :item_fields
    end
  end
end
```

Em `app/services/clinical_documents/issuers.rb`, no começo de `allowed?`:

```ruby
    def allowed?(kind, cbo)
      # ADR 0034: o registro da Notificação, como a RCE, só médico e dentista.
      return %i[physician dentist].include?(role(cbo)) if kind == "controlled_notification_record"
```
(o resto do corpo do 19c continua).

Em `app/services/clinical_documents/json.rb#document`: `short_code: (Codes.display(document.short_code) unless document.notification_record?)` e `verification_url: (document.verification_url unless document.notification_record?)`.

Em `app/services/clinical_documents/verification.rb`: `by_token` e `lookup` tratam o registro como inexistente — troque o `find_by` dos dois por `ClinicalDocument.where.not(kind: "controlled_notification_record").find_by(...)` (a trilha grava `not_found`).

Em `ClinicalDocumentsController#print` (19c), logo depois de achar o documento legível: `return not_found if document.notification_record?`.

Em `Signatures::Canonical.clinical_document` (19c Task 18), junto da guarda `not_signable`: `raise Invalid, "not_signable" if document.notification_record?`.

- [ ] **Step 4: Rode e veja passar**

Run: `rspec spec/services/clinical_documents/notification_record_spec.rb spec/requests/notification_record_spec.rb spec/services/clinical_documents/issuers_spec.rb spec/requests/document_verification_spec.rb`
Expected: PASS. Se a matriz do 19c (`issuers_spec`) listar os tipos fechados, acrescente `controlled_notification_record` (médico e dentista `true`, demais `false`).

- [ ] **Step 5: Commit**

```bash
git add app/services/clinical_documents/content/notification_record.rb app/services/clinical_documents/issuers.rb app/services/clinical_documents/json.rb app/services/clinical_documents/verification.rb app/controllers/clinical_documents_controller.rb app/services/signatures/canonical.rb spec/services/clinical_documents/notification_record_spec.rb spec/requests/notification_record_spec.rb spec/services/clinical_documents/issuers_spec.rb
git commit -m "feat: record paper notification prescriptions in the consultation and the medication list

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 14: Cancelamento e volta ao papel anulam o número

**Files:**
- Modify: `app/commands/clinical_documents/cancel.rb`, `app/commands/signatures/to_paper.rb`
- Test: `spec/commands/clinical_documents/controlled_cancel_spec.rb`

**Interfaces:**
- Consumes: `Sncr::Stock.void!` (Task 8), `ClinicalDocuments::Cancel` (19c), `Signatures::ToPaper` (19b/19c).
- Produces: `ClinicalDocuments::Cancel` deixa o número `voided` na mesma transação do cancelamento e publica `clinical_document.cancelled { document_id, kind, category? }` (category só na receita); `Signatures::ToPaper` de documento clínico com número (exceto `document_cancelled`) também anula.

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/commands/clinical_documents/controlled_cancel_spec.rb
require "rails_helper"

# ADR 0034 (Invariantes: número de receita cancelada não volta; Desvio 11):
# cancelar e voltar ao papel anulam o número; o PDF de papel não o imprime.
RSpec.describe "Número SNCR no cancelamento e na volta ao papel" do
  before do
    controlled_city!(mock: true)
    ciap2_release!; cid10_release!; sigtap_release!; city_profile!
    import_test_catalog!
  end

  let(:unit) { create_unit }
  let(:doctor) { signer_doctor!(unit).tap { |user| prescriber_contact!(user); linked_certificate!(user) } }
  let(:consultation) { finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1)) }
  let(:fluoxetina) { controlled_item!(code: 9_900_001, ingredient: "FLUOXETINA", list: "C1") }

  def rce!
    sncr_stock!(doctor, kind: "rce", count: 1)
    ClinicalDocuments::Issue.call(kind: "prescription", by: doctor, consultation: consultation, content: {
      "items" => [ { "catalog_item" => { "id" => fluoxetina.id }, "quantity" => 30, "quantity_unit" => "cápsula", "route" => "oral",
                     "dosage_instructions" => "1 cápsula pela manhã", "duration_days" => 30 } ],
      "patient_identification" => patient_identification
    }).payload.fetch(:document)
  end

  it "cancelar anula o número, com eventos só com ids e a categoria" do
    document = rce!
    ClinicalDocuments::Cancel.call(document_id: document.id, by: doctor, reason: "Dose errada, vou reemitir")
    expect(SncrNumber.find_by(document_id: document.id).status).to eq("voided")
    cancelled = DomainEvent.where(name: "clinical_document.cancelled").order(:created_at).last.payload
    expect(cancelled).to eq("document_id" => document.id, "kind" => "prescription", "category" => "special_control")
    voided = DomainEvent.where(name: "sncr.number_voided").order(:created_at).last.payload
    expect(voided).to eq("document_id" => document.id, "kind" => "rce")
  end

  it "voltar ao papel anula o número; o PDF de papel não traz o número" do
    document = rce!
    Signatures::ReturnToPaper.call(request_id: document.signature_request.id, by: doctor, reason: "Paciente pediu o papel agora")
    expect(document.reload.issue_mode).to eq("paper")
    expect(SncrNumber.find_by(document_id: document.id).status).to eq("voided")
    text = PDF::Reader.new(StringIO.new(ClinicalDocuments::Pdf.call(document))).pages.map(&:text).join("\n")
    expect(text).not_to include("2610.1-41.0000001")
    ClinicalDocuments::Cancel.call(document_id: document.id, by: doctor, reason: "Cancelando a de papel também")
    expect(SncrNumber.find_by(document_id: document.id).status).to eq("voided")
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `rspec spec/commands/clinical_documents/controlled_cancel_spec.rb`
Expected: FAIL (o número continua `used`).

- [ ] **Step 3: Implemente**

Em `ClinicalDocuments::Cancel.call`, dentro da transação, troque as duas linhas do update e do evento por:

```ruby
        document.update!(status: "cancelled", cancel_reason: note, cancelled_at: now, cancelled_by_user_id: by.id)
        Sncr::Stock.void!(document: document, now: now)
        payload = { document_id: document.id, kind: document.kind }
        payload[:category] = document.content_data["category"] if document.prescription? && document.content_data["category"]
        DomainEvents.publish("clinical_document.cancelled", **payload)
```

Em `Signatures::ToPaper.call`, no ramo que passa o documento clínico a `paper` (19c Task 18; o `document_cancelled` não passa por ele), logo depois de gravar `issue_mode: "paper"`:

```ruby
          Sncr::Stock.void!(document: document, now: now) # ADR 0034, Desvio 11
```
(use o nome da variável e do `now` que o `ToPaper` real tiver.)

O PDF (Task 15) só imprime o número quando `document.digital?`.

- [ ] **Step 4: Rode e veja passar**

Run: `rspec spec/commands/clinical_documents/controlled_cancel_spec.rb spec/commands/clinical_documents spec/commands/signatures`
Expected: PASS, menos a asserção do PDF (passa ao fim da Task 15).

- [ ] **Step 5: Commit**

```bash
git add app/commands/clinical_documents/cancel.rb app/commands/signatures/to_paper.rb spec/commands/clinical_documents/controlled_cancel_spec.rb
git commit -m "feat: void the SNCR number when a prescription is cancelled or returned to paper

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
## Fatia 5 — PDF, canônico e conferência (F-19.26, F-19.27)

### Task 15: PDF nos leiautes da Anvisa (RCE e RET), 2 vias, rodapé Anvisa e faixa de simulado

**Files:**
- Modify: `app/services/clinical_documents/pdf.rb`
- Test: `spec/services/clinical_documents/controlled_pdf_spec.rb`

**Interfaces:**
- Consumes: `ClinicalDocuments::Pdf` (19c: `call`, `copy`, `header`, `person`, `professional`, `verification`, `s`, `date`), `ClinicalDocuments::Address.line`, `Sncr::Config.for_city`, `Cnpj.display`.
- Produces: `ClinicalDocuments::Pdf::{SPECIAL_TITLE, RET_TITLE, SIMULATED_BAND, ANVISA_LINE, PLATFORM_NAME}`; `Pdf.title(document, content) -> String`; `Pdf.sncr_shown?(document, content) -> bool` (digital com `sncr`). RCE/RET digitais: título + "— ELETRÔNICA", "Nº <número>" em destaque em cada via, emitente com endereço e telefone, paciente com CPF/passaporte e endereço, rodapé Anvisa (plataforma + CNPJ, URL de conferência, validar.iti.gov.br, sncr.anvisa.gov.br) além do QR e do rodapé NGS2 do 19c, faixa "NUMERAÇÃO SIMULADA — SEM VALIDADE" em **toda** página quando simulado. Papel: o leiaute do 19c com o título da categoria, endereços impressos, sem número, com assinatura à mão.

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/services/clinical_documents/controlled_pdf_spec.rb
require "rails_helper"
require "pdf/reader"

# Spec §6; P&R RDC 1.000 (itens 7, 67); Desvio 14: título "Receita de
# Controle Especial", número SNCR em destaque, 1ª via farmácia / 2ª via
# paciente, emitente e paciente com endereço, rodapé Anvisa, faixa de
# simulado em toda página. Review Focus 5: receita cheia com emoji.
RSpec.describe "PDF da receita controlada" do
  before do
    ciap2_release!; cid10_release!; sigtap_release!; city_profile!
    import_test_catalog!
  end

  let(:unit) { create_unit }
  let(:doctor) { signer_doctor!(unit).tap { |user| prescriber_contact!(user) } }
  let(:consultation) { finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1)) }
  let(:fluoxetina) { controlled_item!(code: 9_900_001, ingredient: "FLUOXETINA", list: "C1") }

  def rce_item(item = fluoxetina, dosage: "1 cápsula pela manhã")
    { "catalog_item" => { "id" => item.id }, "quantity" => 30, "quantity_unit" => "cápsula", "route" => "oral",
      "dosage_instructions" => dosage, "duration_days" => 60 }
  end

  def issue!(content)
    ClinicalDocuments::Issue.call(kind: "prescription", by: doctor, consultation: consultation, content: content).payload.fetch(:document)
  end

  def pages(document) = PDF::Reader.new(StringIO.new(ClinicalDocuments::Pdf.call(document))).pages.map(&:text)

  context "com o SNCR simulado" do
    before { controlled_city!(mock: true) }

    it "RCE digital simulada: título, número em cada via, contatos, rodapé Anvisa, faixa em toda página" do
      linked_certificate!(doctor)
      sncr_stock!(doctor, kind: "rce", count: 1)
      document = issue!("items" => [ rce_item ], "patient_identification" => patient_identification)
      texts = pages(document)
      expect(texts.size).to eq(2)
      texts.each_with_index do |text, index|
        expect(text).to include("RECEITA DE CONTROLE ESPECIAL — ELETRÔNICA", "Nº 2610.1-41.0000001",
                                index.zero? ? "1ª via — farmácia" : "2ª via — paciente",
                                "NUMERAÇÃO SIMULADA — SEM VALIDADE", "sncr.anvisa.gov.br", "validar.iti.gov.br",
                                "Rua das Araucárias, 120 — Centro — Curitiba/PR — CEP 80010-000", "Telefone: (41) 99999-0000",
                                "CPF: #{consultation.patient.cpf.sub(/\A(\d{3})(\d{3})(\d{3})(\d{2})\z/, '\1.\2.\3-\4')}",
                                "Esta prescrição pode ser verificada em:")
      end
    end

    it "RET digital: título de antimicrobiano eletrônica, validade de 10 dias" do
      linked_certificate!(doctor)
      sncr_stock!(doctor, kind: "ret", count: 1)
      document = issue!("items" => [ { "catalog_item" => { "id" => catalog_item(271_089).id }, "quantity" => 21, "quantity_unit" => "cápsula",
                                       "route" => "oral", "dosage_instructions" => "1 cápsula de 8 em 8 horas", "duration_days" => 7 } ])
      text = pages(document).join("\n")
      expect(text).to include("RECEITA DE ANTIMICROBIANO — ELETRÔNICA", "Nº 2610.2-41.0000001",
                              "Validade: até #{(Time.zone.today + 10).strftime('%d/%m/%Y')}")
    end

    it "RCE de papel: sem número nem faixa, endereços impressos, assinatura à mão, 2 vias" do
      document = issue!("items" => [ rce_item ], "patient_identification" => patient_identification)
      texts = pages(document)
      expect(texts.size).to eq(2)
      expect(texts.first).to include("RECEITA DE CONTROLE ESPECIAL", "CPF: #{consultation.patient.cpf[0, 3]}.", "Assinatura e carimbo",
                                     "Rua das Araucárias")
      expect(texts.join).not_to include("ELETRÔNICA", "Nº 2610", "NUMERAÇÃO SIMULADA")
    end

    it "RCE cheia: 3 C1 + 1 C5, posologia longa com emoji, endereço com emoji (Review Focus 5)" do
      linked_certificate!(doctor)
      sncr_stock!(doctor, kind: "rce", count: 1)
      items = [ fluoxetina, controlled_item!(code: 9_900_006, ingredient: "SERTRALINA", list: "C1"),
                controlled_item!(code: 9_900_007, ingredient: "AMITRIPTILINA", list: "C1"),
                controlled_item!(code: 9_900_009, ingredient: "NANDROLONA", list: "C5") ]
                .map { |item| rce_item(item, dosage: "Tomar 💊 ≥ 1 vez ao dia\r\n#{'x' * 470}") }
      address = ControlledPrescriptionHelpers::ADDRESS.merge("street" => "Rua São João 🌳")
      document = issue!("items" => items, "patient_identification" => patient_identification("address" => address))
      texts = pages(document)
      expect(texts.size).to be >= 2
      expect(texts).to all(include("NUMERAÇÃO SIMULADA — SEM VALIDADE"))
      expect(texts.join).to include("Rua São João ?", "Nº 2610.1-41.0000001", "2ª via — paciente")
    end
  end

  it "RCE digital com número real: sem faixa de simulado" do
    controlled_city!(mock: false)
    linked_certificate!(doctor)
    sncr_stock!(doctor, kind: "rce", count: 1, simulated: false)
    document = issue!("items" => [ rce_item ], "patient_identification" => patient_identification)
    expect(pages(document).join).to include("Nº 2610.1-41.0000001")
    expect(pages(document).join).not_to include("NUMERAÇÃO SIMULADA")
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `rspec spec/services/clinical_documents/controlled_pdf_spec.rb`
Expected: FAIL (o título ainda é o do 19c).

- [ ] **Step 3: Implemente — em `app/services/clinical_documents/pdf.rb`**

Constantes novas:

```ruby
    # ADR 0034 (Desvio 14; P&R RDC 1.000 itens 7 e 67; modelos eletrônicos da Anvisa).
    SPECIAL_TITLE = "RECEITA DE CONTROLE ESPECIAL".freeze
    RET_TITLE = "RECEITA DE ANTIMICROBIANO".freeze
    ELECTRONIC = " — ELETRÔNICA".freeze
    SIMULATED_BAND = "NUMERAÇÃO SIMULADA — SEM VALIDADE".freeze
    ANVISA_LINE = "Verifique a numeração e registre a utilização desta prescrição em sncr.anvisa.gov.br".freeze
    PLATFORM_NAME = "Rota Saúde".freeze
```

Em `call`, logo depois de criar o `pdf` e antes do `stamp_footer`:

```ruby
      simulated_band(pdf) if sncr_shown?(document, content) && content.dig("sncr", "simulated")
```

Em `header`, troque a linha do `title = …` e a do `pdf.text s(title)` por:

```ruby
      pdf.text s(title(document, content)), style: :bold, size: 13
      pdf.text s("Nº #{content.dig('sncr', 'number')}"), style: :bold, size: 16 if sncr_shown?(document, content)
```

Substitua `person` por:

```ruby
    def person(pdf, document)
      person = document.patient || document.citizen
      identification = document.prescription? ? document.content_data["patient_identification"] : nil
      birth = person.birth_date.present? ? date(person.birth_date) : "não informado"
      pdf.move_down 8
      pdf.text s("Paciente: #{person.display_name}"), style: :bold
      if identification
        # O CPF da identificação é o do cadastro (preenchido pelo api na emissão).
        pdf.text s("CPF: #{cpf(identification['cpf'])} — Nascimento: #{birth}")
        pdf.text s("Endereço: #{Address.line(identification['address'])}") if identification["address"]
      else
        # O impresso é do paciente: CPF completo, como o impresso da consulta do 19a.
        pdf.text s("CPF: #{cpf(person.cpf)} — Nascimento: #{birth}")
        pdf.text s("Endereço: ________________________________________________") if document.prescription?
      end
    end

    def cpf(value) = value.to_s.sub(/\A(\d{3})(\d{3})(\d{3})(\d{2})\z/, '\1.\2.\3-\4')
```

Em `professional`, depois da linha do conselho:

```ruby
      if (contact = content["prescriber_contact"])
        pdf.text s("Endereço: #{Address.line(contact['address'])}")
        pdf.text s("Telefone: #{phone(contact['phone'])}")
      end
```
e o helper `def phone(value) = value.to_s.sub(/\A(\d{2})(\d{4,5})(\d{4})\z/, '(\1) \2-\3')`.

No fim de `verification`, acrescente o rodapé Anvisa:

```ruby
      anvisa_footer(pdf, document) if sncr_shown?(document, document.content_data)
```

E os métodos novos:

```ruby
    def title(document, content)
      return TITLES.fetch(document.kind) unless document.prescription?

      case content["category"]
      when "special_control" then document.digital? ? "#{SPECIAL_TITLE}#{ELECTRONIC}" : SPECIAL_TITLE
      when "antimicrobial" then document.digital? ? "#{RET_TITLE}#{ELECTRONIC}" : ANTIMICROBIAL_TITLE
      else content["antimicrobial"] ? ANTIMICROBIAL_TITLE : TITLES.fetch("prescription")
      end
    end

    # O número só sai no documento digital (Desvio 11: a volta ao papel anula).
    def sncr_shown?(document, content) = document.digital? && content["sncr"].is_a?(Hash)

    def simulated_band(pdf)
      pdf.repeat(:all) do
        pdf.bounding_box([ 0, pdf.bounds.top + 28 ], width: pdf.bounds.width, height: 18) do
          pdf.fill_color "B00020"
          pdf.text s(SIMULATED_BAND), style: :bold, size: 11, align: :center
          pdf.fill_color "000000"
        end
      end
    end

    def anvisa_footer(pdf, document)
      cnpj = Sncr::Config.for_city(Current.city)&.cnpj
      pdf.move_down 6
      pdf.text s("Plataforma de prescrição: #{PLATFORM_NAME}#{cnpj ? " — CNPJ #{Cnpj.display(cnpj)}" : ''}"), size: 8
      pdf.text s("Esta prescrição pode ser verificada em: #{document.verification_url}"), size: 8
      pdf.text s("Assinatura digital ICP-Brasil: valide em validar.iti.gov.br"), size: 8
      pdf.text s(ANVISA_LINE), size: 8
    end
```
Acrescente `:title, :sncr_shown?, :simulated_band, :anvisa_footer, :cpf, :phone` ao `private_class_method` (deixe `title` e `sncr_shown?` públicos se a Task 17 os usar — ela não usa). O comentário do topo do arquivo ganha: "RCE/RET (ADR 0034): título da Anvisa, número SNCR em destaque, endereços, rodapé Anvisa e faixa de simulado".

> A margem de cima do PDF é 40 pt: a faixa (`bounds.top + 28`) cabe na margem e não empurra o texto. `Consultations::Print.safe` troca emoji e "≥" por "?".

- [ ] **Step 4: Rode e veja passar**

Run: `rspec spec/services/clinical_documents/controlled_pdf_spec.rb spec/services/clinical_documents/pdf_spec.rb spec/commands/clinical_documents/controlled_cancel_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add app/services/clinical_documents/pdf.rb spec/services/clinical_documents/controlled_pdf_spec.rb
git commit -m "feat: print special control and antimicrobial prescriptions in the Anvisa layouts

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 16: JSON canônico estendido contra `clinical-v1.2.0`

**Files:**
- Modify: `config/clinical/clinical-document-v1.json`, `spec/fixtures/clinical/**` (cópias da tag), `app/services/clinical_documents/canonical.rb`
- Test: `spec/services/clinical_documents/controlled_canonical_spec.rb`

**Interfaces:**
- Consumes: `Signatures::Canonical.{clinical_document,validate!}`, `Signatures::Jcs.dump` (19b/19c), `ClinicalDocuments::Canonical.content` (19c).
- Produces: `ClinicalDocuments::Canonical::CONTROLLED_KEYS = %w[sncr patient_identification prescriber_contact]`; na receita, o canônico leva `category` só quando ≠ `common` e cada chave de `CONTROLLED_KEYS` só quando não nula (Desvio 8); os vetores da 1.1.0 e os dois novos da 1.2.0 reproduzidos byte a byte.

- [ ] **Step 1: Copie o esquema e os exemplos da tag**

```bash
/opt/homebrew/bin/git -C contracts show clinical-v1.2.0:clinical/clinical-document-v1.json > apps/api/.claude/mod19d/config/clinical/clinical-document-v1.json
for f in $(/opt/homebrew/bin/git -C contracts ls-tree -r --name-only clinical-v1.2.0 clinical/examples/ | grep -v "examples/consultation"); do
  target="apps/api/.claude/mod19d/spec/fixtures/clinical/${f#clinical/examples/}"
  mkdir -p "$(dirname "$target")"
  /opt/homebrew/bin/git -C contracts show "clinical-v1.2.0:$f" > "$target"
done
cat apps/api/.claude/mod19d/spec/fixtures/clinical/canonical/SHA256SUMS
```
Expected: o esquema novo; os exemplos do documento (o diretório anotado no 19c, ex.: `clinical-document/`) com o `manifest.json` maior; o `SHA256SUMS` com as linhas antigas **iguais** e as duas novas.

- [ ] **Step 2: Escreva a spec que falha**

```ruby
# spec/services/clinical_documents/controlled_canonical_spec.rb
require "rails_helper"

# Contrato §9; Desvio 8: a RCE e a RET digitais assinam com category, sncr,
# patient_identification e prescriber_contact, válidas no esquema da tag
# clinical-v1.2.0; a receita comum continua byte a byte como na 1.1.0; os
# vetores novos são reproduzidos.
RSpec.describe "JSON canônico da receita controlada (clinical-v1.2.0)" do
  dir = Rails.root.join("spec/fixtures/clinical")
  examples = "clinical-document"
  sums = File.read(dir.join("canonical/SHA256SUMS")).lines.to_h { |line| line.split.then { |sha, name| [ name, sha ] } }

  it "os vetores da 1.1.0 não mudaram e os novos existem" do
    expect(sums).to include("prescription-doctor.jcs" => "62bd59faef736229bfc314adb643a128fa557f4434c6277085f031efa1c170f5",
                            "sick-note-leave.jcs" => "171c46109f960cdebb7c78dba67a29d2d6e548cb84bd29961145a8cb3bc77afc")
    # Hashes do plano docs/superpowers/plans/2026-10-10-module-19d-controlled-contracts-repo.md (1617 e 1390 bytes).
    expect(sums).to include("prescription-special-control.jcs" => "25538e7fb3c406a8a84a785069307297097b338c460d15ed2599cf7df2bb2e0a",
                            "prescription-ret.jcs" => "e4397d409d2e9bc5a625e6e8bce4ab0d1539e4f1d7f4c2256c7812a95f170464")
    expect(File.size(dir.join("canonical/prescription-special-control.jcs"))).to eq(1617)
    expect(File.size(dir.join("canonical/prescription-ret.jcs"))).to eq(1390)
  end

  %w[prescription-special-control prescription-ret prescription-doctor].each do |name|
    it "#{name}: a normalização do construtor preserva os bytes do vetor" do
      example = JSON.parse(File.read(dir.join(examples, "#{name}.json")))
      document = instance_double(ClinicalDocument, kind: "prescription", content_data: example["content"].deep_dup)
      rebuilt = example.merge("content" => ClinicalDocuments::Canonical.content(document))
      expect(Signatures::Jcs.dump(rebuilt).b).to eq(File.binread(dir.join("canonical", "#{name}.jcs")))
    end
  end

  describe "construtor" do
    before do
      controlled_city!(mock: true)
      ciap2_release!; cid10_release!; sigtap_release!; city_profile!
      import_test_catalog!
    end

    let(:unit) { create_unit }
    let(:doctor) { signer_doctor!(unit).tap { |user| prescriber_contact!(user); linked_certificate!(user) } }
    let(:consultation) { finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1)) }

    def issue!(content) = ClinicalDocuments::Issue.call(kind: "prescription", by: doctor, consultation: consultation, content: content).payload.fetch(:document)

    it "RCE digital: válida no esquema, com as quatro chaves" do
      sncr_stock!(doctor, kind: "rce", count: 1)
      document = issue!("items" => [ { "catalog_item" => { "id" => controlled_item!(code: 9_900_001, ingredient: "FLUOXETINA", list: "C1").id },
                                       "quantity" => 30, "quantity_unit" => "cápsula", "route" => "oral",
                                       "dosage_instructions" => "1 cápsula pela manhã", "duration_days" => 30 } ],
                        "patient_identification" => patient_identification)
      content = Signatures::Canonical.clinical_document(document).document["content"]
      expect(content).to include("category" => "special_control",
                                 "sncr" => { "kind" => "rce", "number" => "2610.1-41.0000001", "simulated" => true },
                                 "patient_identification" => { "cpf" => consultation.patient.cpf, "address" => ControlledPrescriptionHelpers::ADDRESS },
                                 "prescriber_contact" => { "address" => ControlledPrescriptionHelpers::ADDRESS, "phone" => "41999990000" })
    end

    it "receita comum: sem category nem as chaves nulas (bytes da 1.1.0)" do
      document = issue!("items" => [ { "catalog_item" => { "id" => catalog_item(268_856).id }, "quantity" => 30, "quantity_unit" => "comprimido",
                                       "route" => "oral", "dosage_instructions" => "1 comprimido pela manhã" } ])
      content = Signatures::Canonical.clinical_document(document).document["content"]
      expect(content.keys).not_to include("category", "sncr", "patient_identification", "prescriber_contact")
    end
  end
end
```

- [ ] **Step 3: Rode e veja falhar**

Run: `rspec spec/services/clinical_documents/controlled_canonical_spec.rb`
Expected: FAIL (o construtor ainda leva `category: "common"` e as chaves nulas, ou o esquema recusa).

- [ ] **Step 4: Implemente — `app/services/clinical_documents/canonical.rb`**

```ruby
module ClinicalDocuments
  module Canonical
    # ADR 0034 (clinical-v1.2.0; Desvio 8): só na RCE/RET; ausentes quando nulos.
    CONTROLLED_KEYS = %w[sncr patient_identification prescriber_contact].freeze

    module_function

    def content(document)
      data = document.content_data
      case document.kind
      when "sick_note", "exam_requisition" then data.except("note").merge("note" => data["note"].presence)
      when "attendance_declaration" then data
      when "prescription"
        items = data["items"].map(&:compact) # catalog_item inteiro: dosage_form pode ser null
        out = data.except("items", "catalog_release", "category", *CONTROLLED_KEYS)
                  .merge("items" => items, "catalog_release" => data["catalog_release"])
        out["category"] = data["category"] if data["category"].present? && data["category"] != "common"
        CONTROLLED_KEYS.each { |key| out[key] = data[key] unless data[key].nil? }
        out
      end
    end
  end
end
```
(o comentário do topo do 19c fica, acrescido de "ADR 0034: category só fora de common; sncr, patient_identification e prescriber_contact só quando presentes".)

- [ ] **Step 5: Rode e veja passar**

Run: `rspec spec/services/clinical_documents/controlled_canonical_spec.rb spec/services/clinical_documents/canonical_spec.rb spec/services/signatures/canonical_vector_spec.rb`
Expected: PASS. Se um exemplo **válido** da tag não passar no construtor por nome de campo, o esquema da tag prevalece: ajuste o api e anote; se for forma (campo obrigatório que o api não tem), **pare** e reporte.

- [ ] **Step 6: Commit**

```bash
git add config/clinical/clinical-document-v1.json spec/fixtures/clinical app/services/clinical_documents/canonical.rb spec/services/clinical_documents/controlled_canonical_spec.rb
git commit -m "feat: sign special control and antimicrobial prescriptions against clinical-v1.2.0

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 17: Página pública com a categoria e o número SNCR

**Files:**
- Modify: `app/services/clinical_documents/verification.rb`, `app/services/clinical_documents/page.rb`
- Test: `spec/requests/controlled_verification_spec.rb`

**Interfaces:**
- Consumes: `ClinicalDocuments::Verification.public_json`, `ClinicalDocuments::Page.result` (19c).
- Produces: `public_json` da receita ganha `category` e, no digital com número, `sncr: { kind, number, simulated }` (Divergência D6); nunca medicamento, CPF, endereço ou telefone. A página HTML mostra "Receita de controle especial"/"Receita de antimicrobiano", a numeração (com "simulada — sem validade") e, cancelada com número, o aviso de não dispensar.

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/requests/controlled_verification_spec.rb
require "rails_helper"

# Contrato §7; spec §6; ADR 0034 (Invariantes): a conferência mostra a
# categoria e o número SNCR; nunca medicamento, CPF, endereço ou telefone; a
# cancelada avisa que o SNCR não registra o cancelamento.
RSpec.describe "Conferência pública da receita controlada", type: :request do
  before do
    controlled_city!(mock: true)
    ciap2_release!; cid10_release!; sigtap_release!; city_profile!
    import_test_catalog!
  end

  let(:unit) { create_unit }
  let(:doctor) { signer_doctor!(unit).tap { |user| prescriber_contact!(user); linked_certificate!(user) } }
  let(:consultation) { finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1)) }

  def rce!
    sncr_stock!(doctor, kind: "rce", count: 1)
    ClinicalDocuments::Issue.call(kind: "prescription", by: doctor, consultation: consultation, content: {
      "items" => [ { "catalog_item" => { "id" => controlled_item!(code: 9_900_001, ingredient: "FLUOXETINA", list: "C1").id },
                     "quantity" => 30, "quantity_unit" => "cápsula", "route" => "oral", "dosage_instructions" => "1 cápsula",
                     "duration_days" => 30 } ],
      "patient_identification" => patient_identification
    }).payload.fetch(:document)
  end

  it "JSON e HTML: categoria e numeração, nada do paciente nem do medicamento" do
    document = rce!
    get "/v/#{document.verification_token}", headers: { "Accept" => "application/json" }
    json = JSON.parse(response.body)
    expect(json).to include("kind" => "prescription", "category" => "special_control",
                            "sncr" => { "kind" => "rce", "number" => "2610.1-41.0000001", "simulated" => true })
    expect(response.body).not_to include("FLUOXETINA", consultation.patient.cpf, "Araucárias", "99999")
    get "/v/#{document.verification_token}"
    expect(response.body).to include("Receita de controle especial", "2610.1-41.0000001", "simulada — sem validade")
    expect(response.body).not_to include("FLUOXETINA", "Araucárias")
  end

  it "cancelada: aviso de não dispensar; receita comum sem sncr" do
    document = rce!
    ClinicalDocuments::Cancel.call(document_id: document.id, by: doctor, reason: "Dose errada, vou reemitir")
    get "/v/#{document.verification_token}"
    expect(response.body).to include("Cancelado", "não dispense", "O SNCR não registra o cancelamento")
    common = ClinicalDocuments::Issue.call(kind: "prescription", by: doctor, consultation: consultation, content: {
      "items" => [ { "catalog_item" => { "id" => catalog_item(268_856).id }, "quantity" => 30, "quantity_unit" => "comprimido",
                     "route" => "oral", "dosage_instructions" => "1 comprimido" } ]
    }).payload.fetch(:document)
    get "/v/#{common.verification_token}", headers: { "Accept" => "application/json" }
    expect(JSON.parse(response.body)).to include("category" => "common")
    expect(JSON.parse(response.body)).not_to have_key("sncr")
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `rspec spec/requests/controlled_verification_spec.rb`
Expected: FAIL (`category` ausente).

- [ ] **Step 3: Implemente**

Em `ClinicalDocuments::Verification.public_json`, antes do `json` final ser devolvido:

```ruby
      if document.prescription?
        content = document.content_data
        json[:category] = content["category"] || (content["antimicrobial"] ? "antimicrobial" : "common")
        if document.digital? && content["sncr"].is_a?(Hash)
          json[:sncr] = { kind: content.dig("sncr", "kind"), number: content.dig("sncr", "number"),
                          simulated: content.dig("sncr", "simulated") == true }
        end
      end
```

Em `ClinicalDocuments::Page`:

```ruby
    CATEGORIES = { "special_control" => "Receita de controle especial", "antimicrobial" => "Receita de antimicrobiano" }.freeze
```
e, em `result`, troque `[ "Documento", KINDS.fetch(data[:kind]) ]` por `[ "Documento", CATEGORIES[data[:category]] || KINDS.fetch(data[:kind]) ]`, e acrescente às `rows`, depois da linha "Modo":

```ruby
      if data[:sncr]
        rows << [ "Numeração SNCR", "#{data.dig(:sncr, :number)}#{' (simulada — sem validade)' if data.dig(:sncr, :simulated)}" ]
        if data[:status] == "cancelled"
          rows << [ "Atenção", "Receita cancelada: não dispense. O SNCR não registra o cancelamento; vale esta conferência." ]
        end
      end
```

- [ ] **Step 4: Rode e veja passar**

Run: `rspec spec/requests/controlled_verification_spec.rb spec/requests/document_verification_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add app/services/clinical_documents/verification.rb app/services/clinical_documents/page.rb spec/requests/controlled_verification_spec.rb
git commit -m "feat: show the prescription category and SNCR number on the public verification page

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
## Fatia 6 — Admin, maintenance, invariantes e semente (F-19.24, F-19.29)

### Task 18: Painel do SNCR do admin municipal

**Files:**
- Create: `app/services/sncr/admin_overview.rb`, `app/controllers/sncr/admin_controller.rb`
- Test: `spec/requests/sncr_admin_spec.rb`

**Interfaces:**
- Consumes: `Sncr::BaseController` (Task 9), `Sncr::{Mock,Stock,StartRequest::COUNCILS}`, `CitizenVerificationPolicy#manage?`, `Professional`.
- Produces: `Sncr::AdminOverview.call(simulated:) -> { professionals: [ { user_id, name, rce_free, ret_free, low, last_request_at } ], simulated: }` (só CRM/CRO de usuários ativos, saldo do modo corrente, `low` = algum tipo abaixo de 50); `GET /sncr/admin/overview` (municipal_admin, só leitura; 403 `missing_role` para os demais; 403 `feature_disabled` com o interruptor desligado).

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/requests/sncr_admin_spec.rb
require "rails_helper"

# Contrato §6; spec §7: o admin municipal vê o saldo de cada prescritor (CRM
# e CRO) no modo corrente e quem está com saldo baixo; só leitura; nada de
# número SNCR, CPF ou inscrição.
RSpec.describe "Painel do SNCR do admin", type: :request do
  before { controlled_city!(mock: true) }

  let(:unit) { create_unit }
  let(:doctor) { doctor!(unit) }
  let(:dentist) { dentist!(unit) }
  def body = JSON.parse(response.body)

  it "saldo por prescritor, baixo, último pedido; sem enfermeiro; sem número" do
    sncr_stock!(doctor, kind: "rce", count: 60)
    sncr_stock!(doctor, kind: "ret", count: 60)
    sncr_stock!(dentist, kind: "rce", count: 10)
    sncr_stock!(doctor, kind: "rce", count: 5, simulated: false)
    nurse!(unit)
    sign_in_as(municipal_admin!)
    get "/sncr/admin/overview"
    expect(response).to have_http_status(:ok)
    rows = body["professionals"].index_by { |row| row["user_id"] }
    expect(rows.keys).to contain_exactly(doctor.id, dentist.id)
    expect(rows[doctor.id]).to include("rce_free" => 60, "ret_free" => 60, "low" => false, "last_request_at" => be_present)
    expect(rows[dentist.id]).to include("rce_free" => 10, "ret_free" => 0, "low" => true)
    expect(body["simulated"]).to be(true)
    expect(response.body).not_to include("2610.", doctor.professional.registration_number)
  end

  it "profissional não vê; interruptor desligado recusa" do
    sign_in_as(doctor)
    get "/sncr/admin/overview"
    expect([ response.status, body["error"] ]).to eq([ 403, "missing_role" ])
    Platform::Features.set!(city: Current.city, key: "controlled_prescriptions", enabled: false, maintainer: ledi_maintainer!)
    sign_in_as(municipal_admin!)
    get "/sncr/admin/overview"
    expect([ response.status, body["feature"] ]).to eq([ 403, "controlled_prescriptions" ])
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `rspec spec/requests/sncr_admin_spec.rb`
Expected: FAIL (`uninitialized constant Sncr::AdminController`).

- [ ] **Step 3: Implemente**

```ruby
# app/services/sncr/admin_overview.rb
# Painel do SNCR do admin municipal (ADR 0034; spec §7; contrato §6). Só
# leitura; o saldo do modo corrente (simulado ou real); nada de número SNCR,
# CPF ou inscrição. Nenhuma coluna cifrada é lida.
module Sncr
  module AdminOverview
    LIMIT = 500

    module_function

    def call(simulated:)
      free = SncrNumber.available.where(simulated: simulated).group(:user_id, :kind).count
      last = SncrNumberBatch.where(simulated: simulated).group(:user_id).maximum(:requested_at)
      professionals = Professional.joins(:user).where(users: { deactivated_at: nil }, council: StartRequest::COUNCILS)
                                  .order(:professional_name).limit(LIMIT).map do |professional|
        rce = free.fetch([ professional.user_id, "rce" ], 0)
        ret = free.fetch([ professional.user_id, "ret" ], 0)
        { user_id: professional.user_id, name: professional.professional_name, rce_free: rce, ret_free: ret,
          low: rce < Stock::LOW_THRESHOLD || ret < Stock::LOW_THRESHOLD, last_request_at: last[professional.user_id]&.iso8601 }
      end
      { professionals: professionals, simulated: simulated }
    end
  end
end
```

```ruby
# app/controllers/sncr/admin_controller.rb
# Painel do SNCR (contrato §6): municipal_admin, só leitura.
module Sncr
  class AdminController < BaseController
    skip_before_action :require_professional
    before_action :require_manager

    def overview
      render json: AdminOverview.call(simulated: Mock.on?(Current.city))
    end

    private

    def require_manager
      forbid("missing_role") unless CitizenVerificationPolicy.new(Current.user, nil).manage?
    end
  end
end
```

- [ ] **Step 4: Rode e veja passar**

Run: `rspec spec/requests/sncr_admin_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add app/services/sncr/admin_overview.rb app/controllers/sncr/admin_controller.rb spec/requests/sncr_admin_spec.rb
git commit -m "feat: show the municipal admin each prescriber's SNCR number balance

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 19: `sncrStatus` na API de manutenção

**Files:**
- Create: `app/graphql/maintenance/types/sncr_status_type.rb`
- Modify: `app/graphql/maintenance/types/query_type.rb`, `app/graphql/maintenance/analyzers/human_only.rb`, `spec/architecture/maintenance_schema_spec.rb`
- Test: `spec/requests/maintenance/sncr_status_spec.rb`

**Interfaces:**
- Consumes: `Sncr::Checks.status` (Task 5); interruptores pelo mecanismo genérico (`city { features }` e `setCityFeature`, Task 1).
- Produces: `Query.sncrStatus: SncrStatus!` com `configured: Boolean!`, `reachable: Boolean!`, `lastCheckAt: ISO8601DateTime`, `simulatedAvailable: Boolean!`; só sessão humana (`HumanOnly::RESTRICTED`); nenhuma cidade aberta; nunca URL, CNPJ ou segredo.

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/requests/maintenance/sncr_status_spec.rb
require "rails_helper"

# Contrato §8: o maintenance vê se o SNCR está configurado no ambiente, se a
# última conversa real deu certo e se o simulado existe aqui. Nunca URL, CNPJ
# ou segredo; nenhuma cidade é aberta; token de serviço não alcança.
RSpec.describe "Maintenance: SNCR", type: :request do
  let(:frontend) { "https://maintenance.rotasaude.app" }
  let(:password) { "s3nha-forte-1" }
  let!(:maintainer) do
    Maintainer.create!(email_address: "sncr-#{SecureRandom.hex(3)}@rotasaude.app", password: password,
                       otp_secret: ROTP::Base32.random, otp_enabled_at: Time.current)
  end

  def browser = { "Origin" => frontend, "X-Rota-Maintenance" => "1" }
  def json = JSON.parse(response.body)

  def login!
    post "/session", params: { email_address: maintainer.email_address, password: password }, headers: browser
    post "/session/challenge", params: { session_id: json["session_id"], code: ROTP::TOTP.new(maintainer.otp_secret).now },
         headers: browser
    expect(response).to have_http_status(:ok)
  end

  def gql!(query) = post("/graphql", params: { query: query, variables: "{}" }, headers: browser)

  before do
    host! "maintenance-api.rotasaude.app"
    allow(ENV).to receive(:[]).and_call_original
    allow(ENV).to receive(:[]).with(MaintenanceApi::ORIGIN).and_return(frontend)
    allow(ENV).to receive(:[]).with("FAKE_SNCR_URL").and_return(ControlledPrescriptionHelpers::FAKE_SNCR_BASE)
    allow(Sncr::Config).to receive(:credentials)
      .and_return("base_url" => "https://api-gateway.prd.apps.anvisa.gov.br/api-sncr/api/v1",
                  "auth_url" => "https://sncr-api.apps.anvisa.gov.br/api/v1", "maintainer_cnpj" => "11222333000181")
    SncrCheck.where(target: "real").delete_all
    SncrCheck.record!("real", ok: true)
    login!
  end

  it "configurado, alcançável, simulado disponível; sem URL nem CNPJ; nenhuma cidade aberta" do
    expect(Maintenance::CityReader).not_to receive(:call)
    gql!("{ sncrStatus { configured reachable lastCheckAt simulatedAvailable } }")
    expect(json.dig("data", "sncrStatus")).to include("configured" => true, "reachable" => true, "simulatedAvailable" => true,
                                                      "lastCheckAt" => be_present)
    expect(response.body).not_to include("anvisa", "11222333000181")
  end

  it "sem credencial nem conversa: falso, sem erro" do
    allow(Sncr::Config).to receive(:credentials).and_return({})
    SncrCheck.where(target: "real").delete_all
    gql!("{ sncrStatus { configured reachable lastCheckAt } }")
    expect(json.dig("data", "sncrStatus")).to eq("configured" => false, "reachable" => false, "lastCheckAt" => nil)
  end
end
```

Em `spec/architecture/maintenance_schema_spec.rb`: `"Query"` ganha `sncrStatus`; `"SncrStatus" => %w[configured reachable lastCheckAt simulatedAvailable]`; em `field_selections`, `"sncrStatus" => "{ configured }"`.

- [ ] **Step 2: Rode e veja falhar**

Run: `rspec spec/requests/maintenance/sncr_status_spec.rb spec/architecture/maintenance_schema_spec.rb`
Expected: FAIL (`Field 'sncrStatus' doesn't exist on type 'Query'`).

- [ ] **Step 3: Implemente**

```ruby
# app/graphql/maintenance/types/sncr_status_type.rb
module Maintenance
  module Types
    class SncrStatusType < BaseObject
      description "SNCR da Anvisa no ambiente (ADR 0034; contrato §8): credencial presente, última conversa real e se " \
                  "o SNCR simulado existe aqui. Nunca segredo, endereço ou CNPJ."

      field :configured, Boolean, null: false
      field :reachable, Boolean, null: false
      field :last_check_at, GraphQL::Types::ISO8601DateTime, null: true
      field :simulated_available, Boolean, null: false
    end
  end
end
```

Em `app/graphql/maintenance/types/query_type.rb`, junto de `signer_status`:

```ruby
      # ADR 0034 (contrato §8): plataforma, sem abrir cidade, sem segredo.
      field :sncr_status, Types::SncrStatusType, null: false,
            description: "SNCR do ambiente: configurado, alcançável e simulado disponível"

      def sncr_status = Sncr::Checks.status
```
Em `app/graphql/maintenance/analyzers/human_only.rb`, `RESTRICTED` ganha `sncrStatus` (na linha de `signatureProviders signerStatus`).

- [ ] **Step 4: Rode e veja passar**

Run: `rspec spec/requests/maintenance/sncr_status_spec.rb spec/architecture/maintenance_schema_spec.rb spec/graphql/maintenance`
Expected: PASS (a guarda de nomes proibidos não reclama: nenhum campo com url/secret/key).

- [ ] **Step 5: Commit**

```bash
git add app/graphql/maintenance/types/sncr_status_type.rb app/graphql/maintenance/types/query_type.rb app/graphql/maintenance/analyzers/human_only.rb spec/architecture/maintenance_schema_spec.rb spec/requests/maintenance/sncr_status_spec.rb
git commit -m "feat: expose the SNCR configuration status to the maintenance API

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 20: Suíte de invariantes do ADR 0034

**Files:**
- Create: `spec/invariants/controlled_prescriptions_invariants_spec.rb`

**Interfaces:**
- Consumes: tudo das Tasks 1–19. Produces: nada novo. Cada bloco diz a mutação que o deixa vermelho.

- [ ] **Step 1: Escreva a suíte**

```ruby
# spec/invariants/controlled_prescriptions_invariants_spec.rb
require "rails_helper"

# ADR 0034 — Invariantes. Cada bloco nomeia a mutação que o deixa vermelho.
RSpec.describe "Invariantes do ADR 0034" do
  before do
    controlled_city!(mock: true)
    ciap2_release!; cid10_release!; sigtap_release!; city_profile!
    import_test_catalog!
  end

  let(:unit) { create_unit }
  let(:doctor) { signer_doctor!(unit).tap { |user| prescriber_contact!(user); linked_certificate!(user) } }
  let(:consultation) { finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1)) }

  def item(code, ingredient, list) = controlled_item!(code: code, ingredient: ingredient, list: list)

  def rce_content(catalog = item(9_900_001, "FLUOXETINA", "C1"))
    { "items" => [ { "catalog_item" => { "id" => catalog.id }, "quantity" => 30, "quantity_unit" => "comprimido", "route" => "oral",
                     "dosage_instructions" => "1 ao dia", "duration_days" => 30 } ],
      "patient_identification" => patient_identification }
  end

  def issue(content = rce_content) = ClinicalDocuments::Issue.call(kind: "prescription", by: doctor, consultation: consultation, content: content)

  # Mutação: tirar o SKIP LOCKED/o índice único, ou o gatilho de transição.
  it "um número SNCR nunca é usado duas vezes" do
    sncr_stock!(doctor, kind: "rce", count: 2)
    documents = 3.times.map { issue.payload[:document] }
    numbers = documents.filter_map { |document| document.content_data.dig("sncr", "number") }
    expect(numbers).to eq(numbers.uniq)
    expect(numbers.size).to eq(2)
    used = SncrNumber.where(status: "used").first
    expect { used.update!(document_id: documents.last.id) }.to raise_error(ActiveRecord::StatementInvalid)
    expect { SncrNumber.create!(batch: used.batch, user: doctor, kind: used.kind, number: used.number, simulated: true) }
      .to raise_error(ActiveRecord::RecordNotUnique)
  end

  # Mutação: cancelar sem void!, ou um gatilho que aceite voided → free.
  it "número de receita cancelada não volta" do
    sncr_stock!(doctor, kind: "rce", count: 1)
    document = issue.payload[:document]
    ClinicalDocuments::Cancel.call(document_id: document.id, by: doctor, reason: "Emitida para o paciente errado")
    number = SncrNumber.find_by(document_id: document.id)
    expect(number.status).to eq("voided")
    expect { number.update!(status: "free", document_id: nil, used_at: nil, voided_at: nil) }.to raise_error(ActiveRecord::StatementInvalid)
    expect(issue.payload[:paper_reason]).to eq("no_sncr_number")
  end

  # Mutação: tirar `environments:` do sncr_mock, a guarda de env do Stock ou do Config, ou o abort do config.ru.
  it "nenhum número simulado em produção" do
    allow(ENV).to receive(:[]).and_call_original
    allow(ENV).to receive(:[]).with("FAKE_SNCR_URL").and_return(ControlledPrescriptionHelpers::FAKE_SNCR_BASE)
    expect(Platform::Features.catalog(env: "production").map(&:key)).not_to include("sncr_mock")
    expect(Sncr::Mock.on?(Current.city, env: "production")).to be(false)
    expect(Sncr::Config.simulated(env: "production")).to be_nil
    allow(Sncr::Config).to receive(:credentials).and_return({})
    expect { Sncr::Client.for(Current.city, env: "production") }.to raise_error(Sncr::Client::Unavailable)
    sncr_stock!(doctor, kind: "rce", count: 1, simulated: true)
    expect(ApplicationRecord.transaction { Sncr::Stock.take!(user: doctor, kind: "rce", simulated: true, env: "production") }).to be_nil
    allow(ENV).to receive(:[]).with("RAILS_ENV").and_return("production")
    allow(ENV).to receive(:[]).with("RACK_ENV").and_return("production")
    expect { Rack::Builder.parse_file(Rails.root.join("lib/fake_sncr/config.ru").to_s) }.to raise_error(SystemExit)
  end

  # Mutação: aceitar enfermeiro em special_control, ou tirar NOTIFICATION/UNSUPPORTED do Controlled.
  it "controle especial só por médico ou dentista; nunca A, B, C2, C3 em receita" do
    { "A1" => :requires_notification, "A2" => :requires_notification, "A3" => :requires_notification,
      "B1" => :requires_notification, "B2" => :requires_notification, "C2" => :not_supported, "C3" => :not_supported }
      .each_with_index do |(list, reason), index|
        expect(issue(rce_content(item(9_900_200 + index, "SUBSTANCIA #{list}", list))).reason).to eq(reason)
      end
    expect(ClinicalDocuments::Content::Controlled.check("special_control", [], role: :nurse, identification: nil, prescriber_contact: nil,
                                                        patient: nil).reason)
      .to eq(:cbo_not_allowed)
    issued = ClinicalDocument.where(kind: "prescription").flat_map { |document| document.content_data["items"].map { |i| i.dig("catalog_item", "id") } }
    lists = MedicationCatalogItem.where(id: issued.compact).pluck(:controlled_list).compact
    expect(lists - %w[C1 C5]).to eq([])
  end

  # Mutação: decidir digital sem certificado, ou não abrir o pedido de assinatura junto com o número.
  it "receita digital de controle especial só com assinatura ICP (pedido nasce na emissão)" do
    sncr_stock!(doctor, kind: "rce", count: 2)
    issue
    other = signer_doctor!(unit, cpf: "11144477735").tap { |user| prescriber_contact!(user) }
    sncr_stock!(other, kind: "rce", count: 1)
    ClinicalDocuments::Issue.call(kind: "prescription", by: other, content: rce_content,
                                  consultation: finalized_consultation!(unit: unit, doctor: other, citizen: verified_citizen!(2)))
    special = ClinicalDocument.where(kind: "prescription").select { |document| document.content_data["category"] == "special_control" }
    special.select(&:digital?).each do |document|
      expect(document.signature_request).to be_present
      expect(document.content_data["sncr"]).to be_present
      expect(SignerCertificate.active.exists?(user_id: document.author_user_id)).to be(true)
    end
    expect(special.reject(&:digital?).map { |document| document.content_data["sncr"] }).to all(be_nil)
    expect(special.count(&:digital?)).to eq(1)
  end
end

# Mutação: gravar o token numa coluna, logar o session_id, pôr o número ou o CPF num evento.
RSpec.describe "Invariantes do ADR 0034 — nada sensível fora do lugar", type: :request do
  before do
    controlled_city!(mock: true)
    stub_sncr_mock!
    ciap2_release!; cid10_release!; sigtap_release!; city_profile!
    import_test_catalog!
  end

  let(:unit) { create_unit }
  let(:doctor) { signer_doctor!(unit).tap { |user| prescriber_contact!(user); linked_certificate!(user) } }

  it "token e session_id nunca persistidos; número, CPF e endereço fora de log e evento" do
    sign_in_as(doctor)
    session_id = nil
    log = capture_log do
      post "/sncr/requests", params: { kind: "rce" }, as: :json
      started = JSON.parse(response.body)
      url = URI.decode_www_form(URI(started["authorize_url"]).query).to_h.fetch("client_url")
      session_id = fake_sncr.approve!(url)
      post "/sncr/oauth/callback", params: { state: started["state"], session_id: session_id }, as: :json
      consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1))
      post "/attendance/consultations/#{consultation.id}/documents", as: :json, params: {
        kind: "prescription",
        content: { items: [ { catalog_item: { id: controlled_item!(code: 9_900_001, ingredient: "FLUOXETINA", list: "C1").id },
                              quantity: 30, quantity_unit: "comprimido", route: "oral", dosage_instructions: "1 ao dia",
                              duration_days: 30 } ],
                   patient_identification: patient_identification }
      }
    end
    number = SncrNumber.where(status: "used").pick(:number)
    patient_cpf = ClinicalDocument.where(kind: "prescription").last.patient.cpf
    secrets = fake_sncr.issued_tokens + [ session_id, number, patient_cpf, "Araucárias" ]
    expect(log).not_to include(*secrets)
    expect(DomainEvent.pluck(:payload).to_json).not_to include(*secrets)
    rows = SignatureOauthState.where(user_id: doctor.id).map { |row| row.attributes.values.join("|") }.join
    expect(rows).not_to include(*fake_sncr.issued_tokens, session_id)
  end
end
```

- [ ] **Step 2: Rode**

Run: `rspec spec/invariants/controlled_prescriptions_invariants_spec.rb`
Expected: PASS. Confira cada mutação dita no comentário (aplique, rode, veja vermelho, desfaça) — pelo menos a do `SKIP LOCKED` e a do `environments:`.

- [ ] **Step 3: Commit**

```bash
git add spec/invariants/controlled_prescriptions_invariants_spec.rb
git commit -m "test: pin the ADR 0034 invariants for SNCR numbers and controlled prescriptions

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 21: Semente de dev

**Files:**
- Create: `lib/controlled_prescriptions_crew.rb`
- Modify: `db/seeds.rb`
- Test: `spec/lib/controlled_prescriptions_crew_spec.rb`

**Interfaces:**
- Consumes: `Platform::Features.set!`, `Maintainer` (`dev@local`), `SignatureCrew.{ensure_totp,otpauth_uri}`, `Professionals::{Create,OpenLink,Cns}`, `DigitalSignatureCrew.free_cpf_for`, `Sncr::Stock.record_batch!`, `Sncr::Client::Block`.
- Produces: `ControlledPrescriptionsCrew.seed_current_city(slug:, password:, admin:) -> { switch:, dentist:, contacts:, block: }` — só Curitiba: liga `controlled_prescriptions` e `sncr_mock` (mantenedor `dev@local`), garante o dentista `dentista@curitiba.demo` (CRO-PR, CBO 223208 na UBS Jardim das Flores, TOTP fixo `DEV_DENTIST_OTP_SECRET`), grava endereço e telefone da médica (`profissional@`) e do dentista, e um bloco simulado de 1.000 RCE e 1.000 RET da médica. Idempotente.

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/lib/controlled_prescriptions_crew_spec.rb
require "rails_helper"
require Rails.root.join("lib/controlled_prescriptions_crew")

# Semente de dev do 19d (ADR 0034): dev é fictício mas imita o real — só
# Curitiba liga, dentista com CRO e CBO real, contatos plausíveis, um bloco
# simulado; reexecutar não duplica nada.
RSpec.describe ControlledPrescriptionsCrew do
  before do
    documents_city!
    Maintainer.find_or_create_by!(email_address: "dev@local") do |maintainer|
      maintainer.password = "dev-password"
      maintainer.otp_secret = ROTP::Base32.random
      maintainer.otp_enabled_at = Time.current
    end
  end

  let!(:unit) { HealthUnit.create!(name: described_class::UNIT, kind: "ubs") }
  let!(:doctor) { doctor!(unit).tap { |user| user.update!(email_address: "profissional@curitiba.demo") } }
  let(:admin) { municipal_admin! }

  it "liga, cria o dentista, os contatos e o bloco; idempotente" do
    first = described_class.seed_current_city(slug: "curitiba", password: "dev-password", admin: admin)
    expect(first[:switch]).to eq("ligado")
    expect(Sncr::Mock.on?(Current.city)).to be(true)
    dentist = User.find_by!(email_address: "dentista@curitiba.demo")
    expect(dentist.professional).to have_attributes(council: "CRO")
    expect(dentist.professional.cpf).to be_present
    expect(dentist.professional.links.active.pluck(:cbo_code)).to eq([ "223208" ])
    expect([ doctor, dentist ].map { |user| user.professional.reload.prescriber_contact }).to all(be_present)
    expect(Sncr::Stock.summary(user: doctor, simulated: true).transform_values { |row| row[:free] }).to eq("rce" => 1000, "ret" => 1000)

    described_class.seed_current_city(slug: "curitiba", password: "dev-password", admin: admin)
    expect(SncrNumberBatch.where(user_id: doctor.id).count).to eq(2)
    expect(User.where(email_address: "dentista@curitiba.demo").count).to eq(1)
  end

  it "outra cidade: nada liga" do
    expect(described_class.seed_current_city(slug: "maringa", password: "x", admin: admin)[:switch]).to eq("desligado (só Curitiba liga)")
    expect(Platform::Features.enabled?(Current.city, "controlled_prescriptions")).to be(false)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `rspec spec/lib/controlled_prescriptions_crew_spec.rb`
Expected: FAIL (`cannot load such file -- …/lib/controlled_prescriptions_crew`).

- [ ] **Step 3: Implemente**

```ruby
# lib/controlled_prescriptions_crew.rb
require "zlib"
require_relative "signature_crew"
require_relative "digital_signature_crew"

# Semente de dev do 19d (ADR 0034). Dev é fictício mas imita o real: só
# Curitiba liga, e liga os DOIS interruptores (controlled_prescriptions e o
# SNCR simulado sncr_mock) com o mantenedor de dev, pelo mesmo caminho do
# maintenance; o dentista da APS (CRO, CBO 223208) com TOTP fixo; endereço e
# telefone de prescritor da médica e do dentista; um bloco SIMULADO de 1.000
# RCE e 1.000 RET da médica (o 1º pedido do mês). Roda depois do
# DigitalSignatureCrew e do ClinicalDocumentsCrew. Idempotente.
module ControlledPrescriptionsCrew
  DEV_MAINTAINER = "dev@local".freeze
  SWITCHES = %w[controlled_prescriptions sncr_mock].freeze
  UNIT = "UBS Jardim das Flores".freeze
  DENTIST = { prefix: "dentista", name: "Marcos Vinícius Teixeira", cbo: "223208",
              secret_env: "DEV_DENTIST_OTP_SECRET", default_secret: "MRSW45DJON2GC5DFON2GCZDFEBZG65DB" }.freeze
  ADDRESS = { "street" => "Rua Desembargador Westphalen", "number" => "1500", "complement" => "sala 4", "district" => "Rebouças",
              "city" => "Curitiba", "uf" => "PR", "zip" => "80230100" }.freeze
  PHONES = { "profissional" => "4132221100", "dentista" => "4132221101" }.freeze

  module_function

  def seed_current_city(slug:, password:, admin:)
    return { switch: "desligado (só Curitiba liga)" } unless slug == "curitiba"

    maintainer = Maintainer.find_by(email_address: DEV_MAINTAINER)
    unless maintainer
      warn "[seeds] receita controlada: sem mantenedor de dev (#{DEV_MAINTAINER}) — interruptores não ligados"
      return { switch: "desligado (sem mantenedor)" }
    end

    city = City.find(Current.city.id)
    SWITCHES.each { |key| Platform::Features.set!(city: city, key: key, enabled: true, maintainer: maintainer) }
    dentist = ensure_dentist(slug: slug, password: password, admin: admin)
    doctor = User.find_by!(email_address: "profissional@#{slug}.demo")
    { "profissional" => doctor, "dentista" => dentist }.each do |prefix, user|
      next if user.professional.prescriber_contact

      user.professional.update!(prescriber_address: ADDRESS.to_json, phone: PHONES.fetch(prefix))
    end
    missing = SWITCHES.flat_map { |key| Platform::Features.missing(city, key) }
    { switch: missing.empty? ? "ligado" : "ligado, falta: #{missing.join(', ')}", dentist: dentist.email_address,
      contacts: 2, block: ensure_block(doctor) }
  end

  def ensure_dentist(slug:, password:, admin:)
    user = User.find_or_initialize_by(email_address: "#{DENTIST[:prefix]}@#{slug}.demo")
    user.password = password
    user.save!
    Membership.find_or_create_by!(user: user, role: "health_professional") { |membership| membership.granted_at = Time.current }
    SignatureCrew.ensure_totp(user, secret_env: DENTIST[:secret_env], default_secret: DENTIST[:default_secret])
    unless user.professional
      seed = "#{slug}:#{DENTIST[:prefix]}"
      attrs = { professional_name: DENTIST[:name], council: "CRO", council_state: CityProfile.current&.uf.presence || "PR",
                cns: Professionals::Cns.generate(seed), registration_number: (Zlib.crc32(seed) % 90_000 + 10_000).to_s }
      result = Professionals::Create.call(user_id: user.id, attrs: attrs, by: admin)
      raise "semente: perfil do dentista recusado (#{result.reason})" if result.failure?

      user.reload
    end
    user.professional.update!(cpf: DigitalSignatureCrew.free_cpf_for(slug, user.professional)) if user.professional.cpf.blank?
    unit = HealthUnit.find_by!(name: UNIT)
    unless user.professional.links.active.exists?(health_unit_id: unit.id, cbo_code: DENTIST[:cbo])
      result = Professionals::OpenLink.call(professional: user.professional, health_unit_id: unit.id, cbo_code: DENTIST[:cbo], by: admin)
      raise "semente: vínculo do dentista recusado (#{result.reason})" if result.failure?
    end
    user
  end

  # Bloco 0 de cada tipo (o fake-sncr começa do bloco 100 em diante: não colide).
  def ensure_block(doctor)
    return "já tinha" if SncrNumberBatch.exists?(user_id: doctor.id, simulated: true)

    %w[rce ret].each do |kind|
      prefix = "#{Time.zone.today.strftime('%y%m')}.#{kind == 'rce' ? 1 : 2}-41."
      block = Sncr::Client::Block.new(first: "#{prefix}0000001", last: "#{prefix}0001000", quantity: 1000)
      Sncr::Stock.record_batch!(user: doctor, kind: kind, block: block, simulated: true)
    end
    "1.000 RCE + 1.000 RET simulados"
  end
end
```

Em `db/seeds.rb`, depois da linha do `DigitalSignatureCrew` (e da do `ClinicalDocumentsCrew` do 19c), com o mesmo `require` que o arquivo usa para os outros crews:

```ruby
        # Depois do DigitalSignatureCrew e do ClinicalDocumentsCrew (19d, ADR 0034).
        controlled = ControlledPrescriptionsCrew.seed_current_city(slug: slug, password: password, admin: muni_admin)
        puts "[seeds] receita controlada .. #{controlled[:switch]}#{controlled[:dentist] ? " — #{controlled[:dentist]} — estoque: #{controlled[:block]}" : ''}"
```

- [ ] **Step 4: Rode e veja passar**

Run: `rspec spec/lib/controlled_prescriptions_crew_spec.rb spec/lib/digital_signature_crew_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add lib/controlled_prescriptions_crew.rb db/seeds.rb spec/lib/controlled_prescriptions_crew_spec.rb
git commit -m "feat: seed Curitiba with controlled prescriptions, a dentist and a simulated SNCR block

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 22: Revisão final, suíte completa, api na porta 3039, prova no navegador e rollout

- [ ] **Step 1: Varredura de vazamento.**
  - `grep -rn 'DomainEvents.publish("sncr\|DomainEvents.publish("controlled_notification\|DomainEvents.publish("clinical_document' -A3 apps/api/.claude/mod19d/app` — só ids, `kind`, `count`, `simulated`, `issue_mode`, `category`, `notification_type` (multilinha: confira cada chamada inteira).
  - `grep -rn "Rails.logger" apps/api/.claude/mod19d/app/services/sncr apps/api/.claude/mod19d/app/commands/sncr apps/api/.claude/mod19d/app/services/clinical_documents` — só contagens, classe de erro e segundos.
  - `grep -rn "access_token\|session_id" apps/api/.claude/mod19d/app apps/api/.claude/mod19d/db` — nenhuma coluna, nenhum `update!`/`create!` com eles.
- [ ] **Step 2: Suíte completa** (worker parado; avise as outras sessões; bancos `_mod19d` recriados):

  ```bash
  docker compose stop worker
  docker compose exec -T -e ROTA_TEST_DB_SUFFIX=_mod19d -w /rails/.claude/mod19d api bundle exec rspec
  docker compose start worker
  ```
  Expected: verde. Spec antiga que fixa listas (catálogo de interruptores, bindings, schema do maintenance, rotas, matriz de emissores, `content` da receita) → acrescente o do 19d (é o contrato); nunca afrouxe asserção de texto livre.
- [ ] **Step 3:** `docker compose exec -T -w /rails/.claude/mod19d api bundle exec rubocop <arquivos tocados>`; corrija só o que a regra do projeto aponta.
- [ ] **Step 4: api na porta 3039 e o fake-sncr na 8092.** Com autorização do usuário, uma etapa de cada vez: `bin/rails db:migrate:platform`, `bin/rails city:migrate:all`, `bin/rails medications:import_anvisa` (as listas com `anticonvulsants.csv`) e `bin/rails db:seed` (Task 21). Depois, sem derrubar o principal:

  ```bash
  docker compose run -d --name fake-sncr-mod19d -p 127.0.0.1:8092:8092 -w /rails/.claude/mod19d api \
    bundle exec puma -b tcp://0.0.0.0:8092 lib/fake_sncr/config.ru
  docker network connect rota-saude_default fake-sncr-mod19d 2>/dev/null || true
  docker compose exec -d -w /rails/.claude/mod19d -e CITY_PUBLIC_BASE_TEMPLATE=http://%{slug}.localhost:5189 \
    -e FAKE_SNCR_URL=http://fake-sncr-mod19d:8092/api/v1 -e FAKE_SNCR_PUBLIC_URL=http://localhost:8092/api/v1 api \
    bin/rails server -b 0.0.0.0 -p 3039 -P tmp/pids/server-mod19d.pid
  ```
  O Vite do worktree do dashboard aponta `VITE_API_PROXY_TARGET` para `http://api:3039`, com `/sncr` no proxy (plano do dashboard). O job de assinatura roda na mão no console do worktree (`Signatures::SignJob.perform_now(city_slug: "curitiba", request_id: …)`) ou num worker do worktree, como no 19b/19c.
- [ ] **Step 5: Prova manual no navegador** (`http://curitiba.localhost:5189/dashboard/`; SNCR e PSC simulados; login, OTP e TOTP são do usuário — as senhas e o TOTP da semente de dev podem ser mostrados no chat se ele pedir):
  1. **Obter números:** a médica (`profissional@curitiba.demo`) em Minha conta → SNCR vê 1.000 RCE e 1.000 RET simulados (a semente) e "pedidos do mês 1"; "Obter números do SNCR" (RCE) abre `http://localhost:8092/api/v1/auth/login…` com "SNCR SIMULADO — desenvolvimento"; "Entrar (simulado)" volta a `/dashboard/sncr/callback` e o saldo vai a 2.000; o 3º pedido passa e o 4º diz "limite do mês"; "Cancelar" no simulado volta com "autorização recusada". Contato (endereço e telefone) gravado no perfil.
  2. **RCE assinada:** com certificado e sessão do PSC simulado (19b), receita com fluoxetina (60 dias) e endereço do paciente: sai digital com o número; assinada pelo job, "Imprimir" baixa o PAdES com "RECEITA DE CONTROLE ESPECIAL — ELETRÔNICA", o número, as 2 vias, os endereços, o rodapé Anvisa, o NGS2 e a faixa "NUMERAÇÃO SIMULADA — SEM VALIDADE" em toda página; o QR abre a conferência com a categoria e o número (simulada), sem medicamento nem CPF.
  3. **RET assinada:** amoxicilina 7 dias → "RECEITA DE ANTIMICROBIANO — ELETRÔNICA", validade de 10 dias, número RET.
  4. **Bloqueios e limites:** diazepam → "exige Notificação"; fluoxetina + losartana → "categorias misturadas"; 61 dias → "duração acima do limite"; sem endereço do paciente → o campo aponta o erro; enfermeira com fluoxetina → recusado.
  5. **Fallback:** sem sessão/certificado, a RCE sai em papel ("vai sair em papel — sem certificado"), sem número, com assinatura à mão.
  6. **Registro de Notificação:** "Registrar Notificação de papel" com diazepam (tipo B, talão `PR-1234567`): aparece na aba Documentos e em "Medicamentos em uso", sem impresso e sem conferência pública.
  7. **Cancelamento:** cancelar a RCE (step-up, motivo) → número anulado no saldo (voided), a conferência mostra "Cancelado — não dispense. O SNCR não registra o cancelamento".
  8. **Painel do admin** (`admin@curitiba.demo`): saldo por prescritor; o dentista com saldo baixo.
  9. **Maintenance** (`maintenance.localhost:5177`, `dev@local`): interruptores `controlled_prescriptions` e `sncr_mock` ligados em Curitiba; `sncrStatus` mostra configurado falso (sem credencial em dev), simulado disponível; desligar `sncr_mock` deixa os simulados parados (a RCE nova sai em papel com "sem número").
- [ ] **Step 6: Pare.** Merge, push, board e docs (página de status do módulo no padrão do módulo 01) só com autorização explícita do usuário, uma etapa de cada vez. Ordem: `contracts` (tag `clinical-v1.2.0`) → **api** → dashboard → maintenance. Rollout: publicar a imagem nova e rodar `db:migrate:platform` e `city:migrate:all` dela **antes** de cortar tráfego (a migração de cidade é irreversível); depois, na imagem publicada, `bin/rails medications:import_anvisa` (com `anticonvulsants.csv`); credenciais `sncr.{base_url, auth_url, maintainer_cnpj}` no `production.yml.enc` (o usuário cria); o interruptor nasce desligado e é ligado por cidade (em produção não há maintenance: pela rake/console, como os interruptores do 19c). Ao voltar o checkout para a main: `DROP DATABASE` dos bancos `_mod19d` (sem pedir de novo, conferindo conexões ativas), `docker rm -f fake-sncr-mod19d` e derrube o servidor da 3039 (`kill $(cat tmp/pids/server-mod19d.pid)` no container). Avise que o `docker-compose.yml` da raiz (fora de git) ganhou o `fake-sncr`.
- [ ] **Step 7: Gates de go-live por cidade (antes de ligar `controlled_prescriptions`):** prova com o SNCR real em homologação e produção (Task 0 com a URL `.br` de produção); domínio `.br` de cada cidade como `client_url` (o `CITY_PUBLIC_BASE_TEMPLATE` de produção); CNPJ da mantenedora confirmado com `sncr.integracao@anvisa.gov.br`; mensagem real de "faixa esgotada"; lista de anticonvulsivantes conferida por farmacêutico (com as listas do 19c); contatos dos prescritores preenchidos; regra da VISA local para Notificação.
- [ ] **Step 8: Pendências para o board de pendências de ciclo** (cards só com autorização): Notificação A/B/B2 eletrônica (endpoint `notificacao-receita` já mapeado); retinoides (C2) e talidomida (C3); QR de consulta do SNCR (formato não publicado); cancelamento no SNCR (pergunta à Anvisa); RCE/RET de enfermeiro em papel depois de 30/10/2026; antiparkinsonianos no limite de 6 meses (art. 59) — hoje só anticonvulsivantes.

---

## Self-review (feito ao escrever o plano)

**Cobertura da spec:**
- §3 interruptores → Task 1; credenciais → Task 5; `Sncr::Client` (login, troca no servidor, `request_rce/ret` = `request_block(kind:)`) → Task 7; `fake-sncr` (30 s, 3/mês, esgotamento) → Task 6; state de uso único do 19b → Task 9; prova técnica → Task 0.
- §4 estoque (lotes, números, lock, devolução, voided, Minha conta, saldo baixo, saldo zero → papel) → Tasks 4, 8, 9, 12, 14.
- §5 categoria, `mixed_categories`, RCE (quem, ≤ 3 C1, 60/180 dias, validade 30, CPF/passaporte, endereços), RET (10 dias, enfermeiro em papel), bloqueios A/B e C2/C3, modo, Notificação (tipos, talão, UF, item, duração, lista de medicamentos, cancelável, fora da conferência), catálogo com lista e anticonvulsivante → Tasks 3, 10, 11, 12, 13.
- §6 PDF Anvisa, assinatura e canônico `clinical-v1.2.0`, conferência, cancelamento → Tasks 14–17.
- §7 telas (lado api) → rotas das Tasks 9, 10, 12, 13, 18; maintenance → Task 19.
- §8 segurança e LGPD → Tasks 4 (cifra, filtros, eventos), 7 (`inspect`), 9 (state), 20.
- §9 testes da spec → categoria (11), bloqueio por lista (11, 20), limites (11), quem prescreve (11, 13, 20), lock e corrida (8), devolução e voided (8, 12, 14), fallback (12), contrato do fake (6), Notificação na lista (13), canônico (16), invariantes (20); prova no navegador (22).
- ADR 0034 invariantes → Task 20; `adr_pointers` → Task 1.

**Placeholders:** a lista de anticonvulsivantes é **dado** transcrito da fonte oficial na Task 3 (formato, fonte, contagem conferida, substâncias de referência); os nomes dos `.jcs` novos são conferidos na tag na Task 0; o `SEND_ORIGIN` e o host do auth vêm da prova da Task 0. Nenhum "TBD" em código.

**Consistência de nomes:** `ControlledPrescriptions::Gate`, `Sncr::{Mock,Config,Checks,Numbers,Client,Stock,StartRequest,CompleteRequest,AdminOverview}`, `Sncr::Client::{Token,Block,Unavailable,Expired,LimitReached,Exhausted,RegistrationMismatch}`, `ClinicalDocuments::{Address,Issue::Context}`, `ClinicalDocuments::Content::{Controlled,PatientIdentification,NotificationRecord}`, `FakeSncr::App`, `SncrCheck`, `SncrNumber`, `SncrNumberBatch`, `ControlledPrescriptionsCrew` — conferidos entre as tasks.

**Review Focus:** as cinco linhas têm teste na task dona (9, 8, 8 + 12, 8 + 20, 11 + 15).

## Divergências propostas ao contrato

- **D1 — Callback do SNCR é `{ state, session_id }`, não `{ state, code }`** (Manual da API §2.2.4: a Anvisa troca o `code` do gov.br no backend dela e devolve só `?session_id`). Não há PKCE nosso; `{ state, error }` segue valendo quando a volta não traz `session_id`. O `code_verifier` da linha do 19b recebe um valor aleatório que nunca sai.
- **D2 — `POST /sncr/requests` devolve `{ authorize_url, state }`** (não só `authorize_url`): a volta oficial não menciona o `state` do cliente; o dashboard guarda o `state` em `sessionStorage` antes de sair e o reenvia no callback. Se a Task 0 provar que o `state` volta no `client_url`, o dashboard pode usar o da URL.
- **D3 — Credenciais `sncr.{base_url, auth_url, maintainer_cnpj}`**, sem `client_id`/`client_secret` (o SNCR "não solicita chaves fixas ou credenciais de parceiro"). `auth_url` = base do `/auth` (o host é confirmado na Task 0).
- **D4 — Rota nova `GET /attendance/consultations/:id/patient_identification`** (autora da consulta) → `{ patient_identification|null }` da última receita emitida do paciente, para o "reaproveitar o último" da spec §5.
- **D5 — Erros do pedido de números e do saldo:** `GET /sncr/stock` também responde 403 `cbo_not_allowed`; 403 **`registration_mismatch`** (novo: gov.br ≠ inscrição, prescritor sem vínculo ou sem cadastro no SNCR); **sai o 409 `professional_cpf_missing`** (o SNCR pede inscrição no conselho, não CPF) — sem perfil ou com conselho fora de CRM/CRO → 403 `cbo_not_allowed`; `kind` inválido → 422 `invalid_content` (`field: "kind"`).
- **D6 — Página pública:** `sncr` leva também `simulated` (`{ kind, number, simulated }`) e a receita cancelada com número mostra o aviso "não dispense — o SNCR não registra o cancelamento"; `category` sai em toda receita (`common` inclusive).
- **D7 — JSON canônico da receita comum sem `category`** (ausente = `common`) e **sem `sncr`/`patient_identification`/`prescriber_contact` nulos**, para os vetores da `clinical-v1.1.0` continuarem byte a byte; na RCE/RET os quatro presentes. O `content` gravado e devolvido pela API traz sempre os quatro (nulos quando não se aplicam). Vetores novos: `prescription-special-control.jcs` (1617 bytes, `25538e7f…`) e `prescription-ret.jcs` (1390 bytes, `e4397d40…`); tag `clinical-v1.2.0`, MINOR da v1.
- **D8 — Lista C4** (antirretrovirais, importada pelo 19c) → 422 `not_supported`, como C2/C3; o §1 do contrato não lista C4.
- **D9 — Controlado em texto livre** continua 422 `controlled_not_allowed` (com `index`) mesmo com o interruptor ligado: só item do catálogo tem lista confiável.
- **D10 — `paper_reason`** vem em toda emissão da consulta que sai em papel (atestado sem certificado → `no_certificate`), `null` no digital, na declaração da recepção, no registro da Notificação e na volta ao papel do 19b; **gravado** em `clinical_documents.paper_reason` e devolvido em todo `<document>` (decisão do usuário, 2026-10-10). Ordem: `feature_disabled` → `nurse_antimicrobial` → `signature_unavailable` → `no_certificate` → `no_sncr_number`.
- **D11 — Registro da Notificação:** `GET /attendance/documents/:id/print` → 404 `not_found`; `/v/:token` → 404; a API devolve `short_code: null` e `verification_url: null` (as colunas do 19c continuam preenchidas por serem NOT NULL).
- **D12 — `prescriber_contact` na RET** é exigido só quando o modo sai `digital` (decidido com o número na mão; faltando → 422 `prescriber_address_missing` e o número volta); na RCE, sempre (digital ou papel).
- **D13 — Volta ao papel (19b) de RCE/RET digital anula o número** (`voided`, com `sncr.number_voided`), como o cancelamento; o PDF de papel nunca imprime número.
- **D14 — Título da RCE "RECEITA DE CONTROLE ESPECIAL"** (P&R RDC 1.000, itens 7 e 67), não "Receituário de Controle Especial" (spec §6).
- **D15 — `duration_days` obrigatório em todo item da RCE** (sem ele não há como provar o limite de 60/180 dias) → 422 `invalid_item` (`field: "duration_days"`).
- **D16 — Contato do prescritor:** `PUT` devolve `{ address, phone }`; 404 `not_found` sem perfil de profissional; o telefone é o `professionals.phone` do módulo 10 (o mesmo editado em `/professionals/me`).
- **D17 — Saldo e painel são do modo corrente da cidade** (`simulated` na resposta); números simulados não contam no modo real e vice-versa. `received` do callback conta só números novos (o SNCR já entregou repetidos).
- **D18 — `clinical_document.cancelled`** leva `category` só na receita (como o `issued`); o item do catálogo na busca/REMUME ganha `controlled_list` e `anticonvulsant` (contrato §1) — o `catalog_ref` do canônico não muda.
- **D19 — Identificação do paciente sem "não possui CPF"** (decisão do usuário, 2026-10-10): `patient_identification` = `{ cpf, address }`; a entrada manda só `{ address }` e o api preenche o `cpf` com o do cadastro do paciente (CPF, `no_cpf` ou `passport` mandados são ignorados); cadastro sem CPF numa RCE → 422 `patient_identification_required` (`field: "patient_identification.cpf"`).
- **D20 — Formatos iguais aos do esquema na entrada:** CEP `^[0-9]{8}$`, UF `^[A-Z]{2}$`, telefone `^[0-9]{10,11}$` — com máscara → 422 com o campo (`invalid_content`/`invalid_address` com `field`, `invalid_phone`); número SNCR `^\d{4}\.\d-\d{2}\.\d{7}$`.
