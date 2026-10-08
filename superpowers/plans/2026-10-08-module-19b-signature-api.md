# Módulo 19 (19b) — Assinatura digital do prontuário (api) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **PRÉ-REQUISITO: 19a fechado em `origin/main` do api** (F-19.1..F-19.7 Verified; branch `feat/mod-19-consultation` mergeado), **tag `clinical-v1.0.0` publicada no `contracts`** e **serviço `signer` com a prova técnica aprovada** (Task 0 do plano do `signer`: CAdES e PAdES AD-RB do sandbox VIDaaS aceitos no validar.iti.gov.br). Este plano usa o código do 19a pelos nomes que os blocos "Produces" do plano do api do 19a fixam (`docs/superpowers/plans/2026-10-07-module-19-consultation-api.md`): `Consultation` (`#finalized?`, `#vitals`, `problem_items`, `conducts`, `exam_requests`, `addenda`, `author_user`, `attendance`, `patient`), `ConsultationAddendum` (`consultation`, `author_user`, `reason`, `text`, `changes`), `Patient` (`#display_name`, `#cpf_masked`, `cpf`, `birth_date`), `ClinicalRecord::{Gate,Access,Trail}`, `ClinicalRecordGate`, `Consultations::{Finalize,AddAddendum,Json,Print}`, `ConsultationsController#read_grant`, `CityEncryption.allowing_reencryption`, os eventos `consultation.finalized { consultation_id, attendance_id }` e `consultation.addendum_added { consultation_id, addendum_id }` e os helpers de spec `ClinicalRecordHelpers` (`clinical_city!`, `doctor!`, `verified_citizen!`, `consulting_attendance!`, `started_consultation!`, `finalized_consultation!`, `draft_body`, `capture_log`). A Task 0 confere isso antes de tudo.

**Goal:** O lado api da assinatura digital do prontuário (F-19.8 a F-19.14, ADR 0032): interruptor `digital_signature`, vínculo do certificado ICP-Brasil em nuvem localizado por CPF nos PSC da plataforma (API v0 do ITI, OAuth2 + PKCE S256), sessão de assinatura por turno, pedido de assinatura nascido da finalização da consulta e do adendo, assinatura automática por job (JSON canônico RFC 8785 em CAdES destacado + PDF com rodapé NGS2 em PAdES, AD-RB, montados e validados pelo serviço interno `signer`), fila de pendentes, lote com uma aprovação, volta ao papel, validação e revalidação, exportação (PDF assinado e `.zip` com JSON + `.p7s`), painel do admin e leitura de prestadores e do `signer` na API de manutenção.

**Architecture:** O Rails conduz o fluxo e o `signer` (Java + Demoiselle, sem estado) prepara, monta e valida. Cinco tabelas novas no banco da cidade (migração `20261008500001`), uma de plataforma (`signature_provider_checks`, migração `20261008500001` em `db/platform_migrate`). O banco é a última palavra: `signatures` só aceita UPDATE das colunas de validação (e da re-cifra), um pedido por documento e uma assinatura por pedido (índices únicos), pedido em estado final nunca volta. A finalização do 19a não muda: um consumidor dos eventos `consultation.finalized`/`consultation.addendum_added` abre o pedido e enfileira `Signatures::SignJob`, que trava o pedido (`FOR UPDATE SKIP LOCKED`) e assina com a sessão do turno; o lote faz o mesmo com um token `multi_signature`. O PSC é falado por um cliente só (`Signatures::Psc::Client`), testado contra um PSC falso Rack (`FakePsc::App`) servido por WebMock `to_rack` nos testes e como serviço `fake-psc` no compose de dev, emitindo e-CPF de teste com a AC de dev do `signer` (volume `signer-dev-pki`; CPF no otherName 2.16.76.1.3.1). O retorno do PSC vai à rota `/dashboard/signature/callback` da cidade, que chama `POST /signature/oauth/callback`.

**Tech Stack:** Rails 8.1 (API), PostgreSQL 16 (banco por cidade + plataforma), RSpec, Solid Queue, Active Record Encryption (chave da cidade, ADR 0007), OpenSSL (Ruby stdlib: PKCE, leitura do certificado, AC de teste), Net::HTTP, `json_schemer` (já no Gemfile), `rubyzip` (já no Gemfile), Prawn (do 19a), WebMock (`to_rack`), Rack (PSC falso). Nenhuma gem nova.

**Spec:** `docs/superpowers/specs/2026-10-08-module-19b-digital-signature-design.md` e `docs/adr/0032.md` (leia os dois antes de começar). Contratos entre apps: `docs/superpowers/plans/2026-10-08-module-19b-signature-contracts.md` — fonte única dos formatos; os planos do dashboard, do maintenance e do `signer` são escritos contra ele: **não mude nomes, formatos nem códigos de erro**. O que o código real obrigou a precisar está em "Desvios da spec" e, quando toca formato, em "Divergências propostas ao contrato" (fim do arquivo). Pesquisa: `docs/pesquisa/2026-10-08-assinatura-digital-19b.md` (§2: API de PSC do ITI).

## Desvios da spec (e precisões)

1. **`previous_sha256` do adendo** segue o esquema do `contracts`: o `canonical_sha256` do documento **assinado** anterior da mesma consulta (consulta ou adendo, na ordem de criação); sem nenhum assinado, o sha256 do JSON canônico da consulta. **No lote** (decisão do usuário): os documentos são assinados em ordem cronológica de criação e os já preparados no mesmo lote contam como assinados — o segundo adendo leva o hash canônico do primeiro, já conhecido antes de assinar; se o primeiro falha no lote, o segundo não é assinado e fica `pending` com o mesmo motivo (Task 11 `Signing`, teste na Task 13).
2. **Status `failed` não é produzido pelo 19b.** Toda falha deixa o pedido `pending` com `reason_code` (o profissional assina em lote ou devolve ao papel). O valor continua no CHECK e no contrato, reservado.
3. **`signer_certificates.certificate_alias`** (coluna a mais): a API do ITI assina por `certificate_alias` (devolvido pela recuperação de certificado); sem ele não há chamada de assinatura.
4. **`signature_oauth_states.provider`, `.request_ids` (uuid[]) e `.return_to`** (colunas a mais): o lote precisa lembrar quais pedidos a aprovação cobre, e o retorno precisa do caminho do dashboard.
5. **`signature_requests.consultation_id`** (para listar e para a forma `<request>`) e **`return_note`** (o motivo livre da volta ao papel, ≥ 10 caracteres, cifrado com a chave da cidade). `signature_requests.consultation_id` e `document_id` sem FK (o documento é polimórfico e aberto ao 19c).
6. **Binários cifrados como texto base64.** `cades`, `signed_pdf`, `validation_material` e `certificate_der` são `text` com base64 cifrado pelo Active Record Encryption (a spec diz `bytea`; o `encrypts` do projeto trabalha com texto e o `CityEncryption`/`ReencryptionJob` só re-cifram texto). `validation_material` e `certificate_der` também são cifrados: a cadeia carrega o certificado do profissional, com CPF e nome. `validation_material` guarda `{"cades": <b64 do .p7c>, "pades": <b64 do .p7c>}` (o `signer` devolve um CMS `.p7c` com a cadeia e as LCRs por montagem).
7. **Trigger de `signatures`** aceita UPDATE só de `last_verification`, `last_verification_at`, `last_verification_reasons` e, sob a marca `rota.reencrypting` (a mesma do 19a), só das colunas cifradas. `signature_requests` recusa DELETE, recusa mudar identidade (`document_type`, `document_id`, `consultation_id`, `author_user_id`) e recusa sair de `signed`/`returned_to_paper` (exceto a re-cifra de `return_note`).
8. **Desligar o interruptor** (`digital_signature`, ou o `clinical_record` de que ele depende, ou o `record_mode`) devolve ao papel os pedidos `pending` com `feature_disabled` em até 10 minutos (`Signatures::SweepJob`, recorrente por cidade) — e na hora, se um job ou lote tocar o pedido antes. O interruptor é gravado na plataforma; a fila mora na cidade; a varredura evita enfileirar job de cidade a partir do maintenance.
9. **Retorno do PSC: um por cidade** (decisão do usuário). A `redirect_uri` de cada pedido é `https://<host do dashboard da cidade>/dashboard/signature/callback`, montada de `CityPublicUrl.dashboard(city)`; o dashboard chama `POST /signature/oauth/callback`. Não há retorno único no `auth.*`. Cadastrar o endereço de cada cidade em cada PSC é passo de go-live por cidade (Task 19).
10. **Revogação no vínculo** (decisão do usuário): o `signer` ganha `POST /certificates/check { certificate_der_base64 }` → `{ status: valid|invalid|indeterminate, signer_cpf, not_after, reasons }` (cadeia + LCR; contrato D1, obrigatória). O vínculo (e a renovação na abertura de sessão) recusa `invalid` com 422 `certificate_revoked` / `certificate_expired` / `certificate_untrusted` (`untrusted_chain`, código novo); `indeterminate` (`revocation_unavailable`) aceita e grava o motivo em `signer_certificates.link_check_status`/`link_check_reasons` — a revogação volta a ser conferida no `/verify` da primeira assinatura; 422 `invalid_certificate` ou `signer` fora do ar → 503 `signer_unavailable`.
11. **Certificado renovado no PSC.** A abertura de sessão relê o certificado; serial diferente do vinculado (o profissional renovou) → passa pelas mesmas conferências do vínculo e vira o `active` (o anterior, `replaced`). A cada assinatura o CPF do certificado é conferido de novo contra o CPF do profissional (NGS2.01.02); diferente → `pending` com `certificate_cpf_mismatch` (Divergência D4).
12. **`signer_name` do bloco `signature`** = nome do profissional autor (o mesmo de `author.name` no 19a); o rodapé do PDF usa o nome do certificado (o que foi assinado).
13. **Impresso com adendos.** `GET /attendance/consultations/:id/print` devolve o PDF assinado (PAdES da consulta) quando a consulta é `digital` **e não tem adendo**; nos demais casos, o impresso do 19a ganha a seção "Assinaturas" (modo e validação de cada parte) e só mantém o espaço de assinatura à mão quando alguma parte é `manual` ou `pending`.
14. **Prestadores em dev.** O PSC falso responde como `vidaas` em development (`config.x.signature_dev_providers`, só development; as credenciais cifradas vencem). A tela do PSC falso diz "PSC SIMULADO — desenvolvimento".
15. **Leitura de assinatura fora de contexto** segue o 19a: abertura justificada válida libera (`ClinicalRecord::Access`), senão 403 `opening_required` (o dashboard trata `out_of_context` e `opening_required` igual).

## Valores fixados por este plano (para o contrato e para o esquema do `contracts`)

1. **JSON canônico** = o esquema da tag `clinical-v1.0.0` do `contracts` (`clinical/consultation-v1.json`, `clinical/consultation-addendum-v1.json`; fechados, `additionalProperties: false`). O construtor da Task 9 produz exatamente: `city { ibge_code|null, name }`, `unit { cnes|null, name }`, `professional { name, cpf, cbo_code, council { name, state, registration_number } }` (no adendo, o autor do adendo), `patient { display_name, cpf, birth_date|null }`, e `consultation { id, started_at, finalized_at, care_type, subjective, objective, assessment, plan (texto vazio = null), vitals { só as medidas preenchidas + bmi }, evaluated_problems [ { terminology, code, label, release, action, onset_on?, onset_precision? } ], conducts, exam_requests [ { sigtap_code, label, cid10_justification? } ], outcome { code, referral_unit_cnes?, referral_note? } }` ou `addendum { consultation_id, id, created_at, reason, text, changes { evaluated_problems?, conducts?, exam_requests? } (mesmas formas), previous_sha256 }`. Horários UTC `AAAA-MM-DDTHH:MM:SSZ`; CPF, CNES, IBGE e CBO só dígitos; `release` = `terminology_releases.version`. O gerador RFC 8785 é provado byte a byte contra o vetor `clinical/examples/canonical/`.
2. **CBO do autor do adendo** (o esquema exige `cbo_code`): o da consulta quando o autor é o mesmo; senão o do vínculo permitido do autor (`Consultations::Authorization.any_allowed_link`).
3. **Tentativas do job:** 3, com espera de 30 s e 2 min entre elas; lote até 50 documentos (100 hashes numa chamada); lista de pendentes até 200; `state` vale 10 min; sessão até 12 h (o menor entre o concedido pelo PSC e o teto).
4. **Varredura** (`Signatures::SweepJob`) a cada 10 minutos por cidade.
5. **Política** `AD-RB`; `policy_oid` = o que o `/verify` do `signer` devolve para o CAdES.
6. **Rodapé NGS2** (toda página do PDF assinado): `Documento assinado digitalmente por <NOME DO CERTIFICADO> (CPF ***.NNN.NNN-**) em DD/MM/AAAA HH:MM UTC — ICP-Brasil, política AD-RB. Verifique em https://validar.iti.gov.br`.
7. **Prestadores** (`Signatures::Providers::CATALOG`): `vidaas`, `birdid`, `safeid`, `neoid`, `remoteid` (rótulos `VIDaaS`, `BirdID`, `SafeID`, `NeoID`, `RemoteID`).

## Global Constraints

- Tudo de domínio no banco de cada cidade: migração em `db/city_migrate`, dump à mão em `db/city_schema.rb` (a paridade compara o schema normalizado, `spec/services/city_schema_spec.rb`), triggers em `db/city_triggers.sql` (a migração termina com `execute File.read(Rails.root.join("db/city_triggers.sql"))`; blocos novos guardados por `to_regclass`). Migração irreversível (`down` levanta `ActiveRecord::IrreversibleMigration`). Número `20261008500001` — maior que as do 19a (`20261007400001/2/3`). A tabela de plataforma vai em `db/platform_migrate` + `db/platform_schema.rb`. Rollout: publicar a imagem e rodar `city:migrate:all` dela antes de cortar tráfego; **nunca** migrar cidade fora do rake (a cidade trava em 503).
- As tabelas do 19a não mudam (ADR 0032, invariante): nenhuma migração toca `consultations`, `consultation_*`, `patients`, `patient_*`, `clinical_record_openings`.
- Cifra por cidade (ADR 0007): `encrypts` sem `key_provider` para token do PSC, `code_verifier`, JSON canônico, CAdES, PDF assinado, provas de validade, certificado e motivo da volta ao papel; `deterministic: true, key_provider: CityDeterministicKeyProvider.new` para `signer_certificates.subject_cpf` e `signatures.signer_cpf`. Todo `encrypts` novo entra em `CityEncryption::CITY_KEYED_TARGETS` (`spec/architecture/city_encrypted_attributes_guard_spec.rb`).
- Valores, exatamente: prestador `vidaas` | `birdid` | `safeid` | `neoid` | `remoteid`; certificado `active` | `replaced` | `unlinked` | `revoked` | `expired`; sessão `active` | `expired` | `revoked`; propósito `link` | `session` | `batch`; pedido `pending` | `signed` | `failed` | `returned_to_paper`; documento no banco `Consultation` | `ConsultationAddendum`, na API `consultation` | `consultation_addendum`; modo `digital` | `manual` | `pending`; validação `valid` | `invalid` | `indeterminate`; política `AD-RB` | `AD-RT`; motivos `no_session`, `session_expired`, `provider_unavailable`, `provider_rejected`, `signer_unavailable`, `verification_failed`, `certificate_expired`, `certificate_revoked`, `certificate_cpf_mismatch`, `feature_disabled`, `user_request`.
- Erros `{ "error": "<reason>" }`; step-up 401 `{ "error": "mfa_required" }`; interruptor 403 `{ "error": "feature_disabled", "feature": "digital_signature" }` (as leituras de assinatura já gravada respondem ao `clinical_record`, não ao `digital_signature` — "assinaturas existentes continuam visíveis"); papel ausente 403 `missing_role`; escrita devolve o objeto puro.
- Eventos só com ids (contrato §11), declarados em `config/initializers/domain_events.rb` e em `spec/initializers/domain_events_bindings_spec.rb`: `signature.certificate_linked`, `signature.certificate_unlinked`, `signature.session_opened`, `signature.signed`, `signature.failed`, `signature.returned_to_paper`, `signature.verified`. Nenhum é de plataforma: nenhum `Platform.audit` novo (o `city.feature_changed` do interruptor já existe). Token do PSC, `code`, `code_verifier`, `state`, CPF, nome, texto clínico e corpo de PSC/`signer` nunca em log (`filter_parameters`), evento, Analytics, mensagem de erro ou `inspect`.
- Jobs por cidade só com `include CityScopedJob` (ou `prepend EachCityJob`); `Current.city` **nunca** é atribuído em `app/` ou `lib/` (`spec/architecture/current_city_assignment_spec.rb`). Consumidor de evento com `include IdempotentConsumer`, sem efeito HTTP (ADR-0005): ele só abre o pedido e enfileira.
- Specs de request com `type: :request` (`infer_spec_type_from_file_location!` está desligado); arquivo novo em `spec/support/` precisa de `require_relative` em `spec/rails_helper.rb`. Specs com threads: `self.use_transactional_tests = false`, `Queue#pop(timeout:)`, `after` que solta as threads e apaga o que commitou (`session_replication_role = replica`).
- `spec/adr_pointers_spec.rb`: `VALID_RANGE = (1..32)`.
- Commits em inglês, Conventional Commits com o tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). `git add` sempre com caminhos explícitos (nunca `-A`). Merge, push, tag e criação de repositório só com autorização explícita do usuário.

## Ambiente de execução

- Antes de tocar `apps/api`, avise a sessão dona do api (sessão "API") e as sessões do `signer` e do dashboard do 19b. Ordem de merge: `contracts` (tag) → `signer` → **api** → dashboard → maintenance.
- Worktree (a partir da raiz do monorepo, `/Users/eduardovrocha/Development/ioit.solutions/rota-saude`):

  ```bash
  /opt/homebrew/bin/git -C apps/api fetch origin
  /opt/homebrew/bin/git -C apps/api worktree add .claude/mod19b -b feat/mod-19b-signature origin/main
  cp apps/api/config/master.key apps/api/.claude/mod19b/config/master.key
  ```

- `./apps/api` é montado em `/rails` no container `api`; o worktree é `/rails/.claude/mod19b`. Todo comando Rails/RSpec:

  ```bash
  docker compose exec -T -w /rails/.claude/mod19b api bundle exec rspec <arquivos>
  ```

- Todo `git add`/`git commit` usa `-C apps/api/.claude/mod19b`.
- Depois da migração de cidade (Task 3): `DROP DATABASE` dos dois bancos de teste de cidade (com o sufixo da branch, se o projeto o aplicar — `lib/test_database_suffix.rb`) e `city:test_databases` de novo; a migração de plataforma (Task 4) roda com `bin/rails db:migrate:platform` no ambiente de teste:

  ```bash
  psql -U rota_saude -d postgres -c "DROP DATABASE rota_saude_test_city_a" -c "DROP DATABASE rota_saude_test_city_b"
  docker compose exec -T -e RAILS_ENV=test -w /rails/.claude/mod19b api bin/rails city:test_databases
  ```

  Ao voltar para a main, repita (o banco de teste fica à frente).
- **`signer` no compose** (Tasks 2, 5 e 18): as specs marcadas `:signer` falam com o `signer` real em `SIGNER_URL` (`http://signer:8090`, com `SIGNER_TOKEN`) e usam a AC de dev do volume `signer-dev-pki` montado no api em `/signer-dev-pki` (`SIGNER_DEV_PKI_DIR`); o `signer` serve as LCRs em `/dev-pki/`. Sem `SIGNER_URL` no ambiente, essas specs ficam fora da corrida com aviso no topo da saída — **nunca** em silêncio (a CI do api ganha o `signer` como serviço numa pendência registrada no fim).
- Suíte completa só com o worker parado e sem outra sessão rodando suíte:

  ```bash
  docker compose stop worker
  docker compose exec -T -w /rails/.claude/mod19b api bundle exec rspec
  docker compose start worker
  ```

- A Task 0 pode usar WebFetch (só leitura) no DOC-ICP-17.01 do ITI e na documentação pública de desenvolvedor do VIDaaS.
- Prova no navegador: o api do worktree sobe na porta **3037** (Task 19), com `CITY_PUBLIC_BASE_TEMPLATE=http://%{slug}.localhost:5187` (o Vite do dashboard do 19b). O prefixo `/signature` precisa de entrada no proxy de dev do dashboard (plano do dashboard).

### Arquivos em comum com o 19a (e como rebasear)

Se o 19a ainda receber correções depois do merge, rebase sobre `origin/main` e resolva **somando** os dois lados em: `db/city_schema.rb` (o `define(version:)` fica com o maior número; tabelas em ordem alfabética), `db/city_triggers.sql` (blocos novos no fim), `config/initializers/domain_events.rb` (o 19b troca o `to: []` de `consultation.finalized`/`consultation.addendum_added` por `to: [Signatures::RequestJob]`), `spec/initializers/domain_events_bindings_spec.rb`, `config/initializers/filter_parameter_logging.rb`, `app/services/city_encryption.rb`, `spec/adr_pointers_spec.rb` (fica `1..32`), `config/routes.rb`, `app/services/platform/features.rb`, `app/services/consultations/{print,json}.rb`, `app/controllers/consultations_controller.rb`.

## Review Focus

1. **Job e lote sobre o mesmo pedido ao mesmo tempo** (o profissional toca "Assinar todas" enquanto o job da finalização ainda assina): nunca duas assinaturas do mesmo documento, nunca pedido `signed` sem linha em `signatures`, nenhum dos dois espera o outro terminar a chamada ao PSC. Testes: Task 13 ("corrida job × lote", com threads).
2. **Rotação de chave da cidade com assinaturas gravadas:** `ReencryptionJob`/`CityRekey` regravam JSON canônico, CAdES, PDF, provas, certificado, token e motivo; o conteúdo decifrado continua byte a byte o mesmo (o sha256 bate); um UPDATE comum continua recusado. Nunca uma rotação que para no meio por causa do trigger. Testes: Task 3 ("re-cifra de assinatura gravada").
3. **PSC ou `signer` fora do ar na hora de finalizar e de assinar:** finalizar continua respondendo como no 19a; o pedido tenta 3 vezes com espera crescente e fica `pending` com o motivo; o token, o `code_verifier` e o corpo da resposta nunca aparecem em log, evento ou erro. Testes: Task 12 ("tentativas e motivos") e Task 17 ("finalizar nunca espera", "nada de segredo em log").
4. **Volta do celular nas bordas:** callback repetido, `state` de outra cidade, de outro usuário ou adulterado, `state` vencido (o profissional aprovou depois de 10 min), `error=access_denied`, código já trocado → 422 `invalid_state` / 409 `authorization_expired` / 403 `authorization_denied`; nunca troca o mesmo código duas vezes, nunca grava sessão ou certificado em nome de outro. Testes: Task 6 (`OauthStates`) e Task 7 ("callback nas bordas").
5. **Documento que a fonte do PDF não tem e JSON estável:** texto com emoji, "≥", `\r\n` e 20.000 caracteres, nome do certificado com acento — o PDF assinável sai (com `?` no que a fonte não tem) com o rodapé em todas as páginas; o JSON canônico do mesmo documento dá o mesmo sha256 em duas gerações e independe da ordem das chaves de `changes`. Testes: Task 9 ("determinístico") e Task 10 ("rodapé em toda página").

---

## Mapa de arquivos

| Arquivo | Responsabilidade | Task |
|---|---|---|
| — | conferir 19a, `contracts`, `signer` e os nomes da API do ITI | 0 |
| `app/services/platform/features.rb`, `app/services/signatures/gate.rb`, `app/controllers/concerns/digital_signature_gate.rb`, `spec/adr_pointers_spec.rb` | interruptor `digital_signature` (requer `feature:clinical_record`) | 1 |
| `lib/fake_psc/pki.rb`, `spec/fixtures/signature_pki/{root,intermediate,intermediate.key}.pem`, `app/services/signatures/certificate_info.rb` | e-CPF de teste na AC do `signer` de dev (ou a cópia das specs) e leitura do certificado | 2 |
| `db/city_migrate/20261008500001_add_digital_signatures.rb`, `db/city_schema.rb`, `db/city_triggers.sql`, `app/models/{signer_certificate,signature_session,signature_oauth_state,signature_request,signature}.rb`, `app/services/signatures/document_types.rb`, `app/services/city_encryption.rb`, `config/initializers/{domain_events,filter_parameter_logging}.rb`, `spec/support/signature_helpers.rb` | dados, triggers, cifra, eventos, helpers | 3 |
| `db/platform_migrate/20261008500001_create_signature_provider_checks.rb`, `db/platform_schema.rb`, `app/models/signature_provider_check.rb`, `app/services/signatures/providers.rb`, `app/services/signatures/psc.rb`, `app/services/signatures/psc/client.rb`, `lib/fake_psc/app.rb` | prestadores e cliente da API v0 do ITI; PSC falso | 4 |
| `app/services/signatures/signer.rb`, `app/services/signatures/signer/client.rb`, `spec/support/fake_signer.rb` | cliente do `signer` (contrato §9) | 5 |
| `app/services/signatures/oauth_states.rb` | `state` de uso único do OAuth | 6 |
| `app/commands/signatures/{discover,start_link,accept_certificate,unlink,complete_oauth}.rb`, `app/services/signatures/json.rb`, `app/controllers/signatures/{base,certificates,oauth}_controller.rb` | vínculo do certificado e o callback | 7 |
| `app/commands/signatures/{start_session,open_session,close_session}.rb`, `app/controllers/signatures/sessions_controller.rb` | sessão do turno | 8 |
| `app/services/signatures/{jcs,canonical}.rb`, `config/clinical/{consultation-v1,consultation-addendum-v1}.json`, `spec/fixtures/clinical/**` | JSON canônico RFC 8785 (provado contra o vetor do `contracts`) e a cadeia | 9 |
| `app/services/consultations/print.rb`, `app/services/signatures/pdf_footer.rb` | PDF assinável com rodapé NGS2; PDF do adendo | 10 |
| `app/services/signatures/{documents,signing,certificate_rules}.rb`, `app/commands/signatures/{sign_pending,park,to_paper}.rb`, `app/jobs/signatures/sign_job.rb` | assinatura (job, tentativas, lock) | 11 |
| `app/commands/signatures/open_request.rb`, `app/jobs/signatures/request_job.rb`, `config/initializers/domain_events.rb` | pedido nascido da finalização e do adendo | 12 |
| `app/commands/signatures/{return_to_paper,start_batch,run_batch}.rb`, `app/jobs/signatures/sweep_job.rb`, `app/controllers/signatures/{requests,batches}_controller.rb`, `config/recurring.yml` | fila, lote, volta ao papel, varredura | 13 |
| `app/services/signatures/{mode,verify,package,print_report}.rb`, `app/services/consultations/json.rb`, `app/controllers/signatures/signatures_controller.rb`, `app/controllers/consultations_controller.rb` | bloco `signature`, validação, conteúdo, PDF, pacote, impresso | 14 |
| `app/services/signatures/admin_overview.rb`, `app/controllers/signatures/admin_controller.rb` | painel do admin | 15 |
| `app/graphql/maintenance/types/{signature_provider_type,signer_status_type,query_type}.rb`, `app/services/signatures/signer_status.rb`, `spec/architecture/maintenance_schema_spec.rb` | leitura no maintenance | 16 |
| `spec/invariants/digital_signature_invariants_spec.rb` | invariantes do ADR 0032 | 17 |
| `config/environments/development.rb`, `lib/fake_psc/config.ru`, `lib/digital_signature_crew.rb`, `db/seeds.rb`, `docker-compose.yml` (raiz, fora do git) | semente e PSC falso de dev | 18 |
| — | revisão final, suíte, porta 3037, prova no navegador, rollout | 19 |

---
## Fatia 0 — Pré-requisitos e interruptor (F-19.8)

### Task 0: Conferir o 19a na main, o `contracts`, o `signer` e os nomes da API do ITI

Nada deste plano compila sem o 19a, e o lado PSC é escrito contra os nomes da API v0 do ITI. Esta task não escreve código.

- [ ] **Step 1: Confira os nomes do 19a em `origin/main`**

```bash
/opt/homebrew/bin/git -C apps/api fetch origin
/opt/homebrew/bin/git -C apps/api log --oneline -1 origin/main
for f in app/models/consultation.rb app/models/consultation_addendum.rb app/models/patient.rb \
         app/services/clinical_record/gate.rb app/services/clinical_record/access.rb app/services/clinical_record/trail.rb \
         app/commands/consultations/finalize.rb app/commands/consultations/add_addendum.rb \
         app/services/consultations/print.rb app/services/consultations/json.rb app/controllers/consultations_controller.rb \
         app/controllers/concerns/clinical_record_gate.rb spec/support/clinical_record_helpers.rb \
         db/city_migrate/20261007400003_add_ledi_correction_pending.rb; do
  /opt/homebrew/bin/git -C apps/api cat-file -e "origin/main:$f" && echo "ok $f" || echo "FALTA $f"
done
/opt/homebrew/bin/git -C apps/api grep -n '"record_mode:record"' origin/main -- app/services/platform/features.rb
/opt/homebrew/bin/git -C apps/api grep -n 'DomainEvents.bind "consultation.finalized"\|DomainEvents.bind "consultation.addendum_added"' origin/main -- config/initializers/domain_events.rb
/opt/homebrew/bin/git -C apps/api grep -n "def call(consultation)\|def safe(text)\|class NotPrintable" origin/main -- app/services/consultations/print.rb
/opt/homebrew/bin/git -C apps/api grep -n "def allowing_reencryption\|def read_grant\|def addendum(a)\|def consultation(c)" origin/main -- app/services/city_encryption.rb app/controllers/consultations_controller.rb app/services/consultations/json.rb
/opt/homebrew/bin/git -C apps/api grep -n "def clinical_city!\|def doctor!\|def finalized_consultation!\|def capture_log" origin/main -- spec/support/clinical_record_helpers.rb
```
Expected: todos `ok`; `record_mode:record` no catálogo; os dois `bind` com `to: []`; `Print.call(consultation)`, `safe`, `NotPrintable`; `allowing_reencryption`, `read_grant`, `Json.consultation`/`addendum`; os quatro helpers. **Pare** e reporte ao coordenador se algo `FALTA` ou se um nome mudou (ajuste este plano antes de seguir).

- [ ] **Step 2: Confira o `contracts` e o `signer`**

```bash
/opt/homebrew/bin/git -C contracts fetch --tags origin
/opt/homebrew/bin/git -C contracts tag -l clinical-v1.0.0
/opt/homebrew/bin/git -C contracts ls-tree -r --name-only clinical-v1.0.0 clinical/ | grep -v "examples/consultation"
ls apps/signer && /opt/homebrew/bin/git -C apps/signer log --oneline -3 origin/main
/opt/homebrew/bin/git -C apps/signer grep -n '"/prepare"\|"/assemble"\|"/verify"\|"/health"\|"/certificates/check"\|SIGNER_DEV_PKI_DIR\|/dev-pki/' origin/main -- src
grep -n "signer-dev-pki\|SIGNER_DEV_PKI_DIR" docker-compose.yml
```
Expected: a tag existe com `clinical/consultation-v1.json`, `clinical/consultation-addendum-v1.json` e `clinical/examples/canonical/{consultation-full.jcs,addendum-structured.jcs,SHA256SUMS}`; `apps/signer` existe com as rotas do contrato §9, `POST /certificates/check` (D1) e o `DevPki` (`SIGNER_DEV_PKI_DIR`, LCRs em `/dev-pki/`); o compose tem o volume `signer-dev-pki` (o plano do `signer` o cria; o api o monta na Task 18). Se algo faltar, **pare** e reporte ao coordenador. Confira também, no registro da Task 0 do plano do `signer`, que a prova técnica (sandbox VIDaaS → validar.iti.gov.br) foi aprovada; sem ela, **pare** (spec §12).

- [ ] **Step 3: Confira os nomes da API v0 do ITI (WebFetch, só leitura)**

Leia o DOC-ICP-17.01 v3.0 (`https://www.gov.br/iti/pt-br/assuntos/legislacao/instrucoes-normativas/IN_20_2020_DOC_17.01_assinada.pdf`) e, se precisar, a documentação pública de desenvolvedor do VIDaaS. Confira, um a um, contra a tabela abaixo (é o que a Task 4 grava em `Signatures::Psc::Client::PATHS` e no `FakePsc::App`):

| Serviço | Método e caminho | Entrada | Saída |
|---|---|---|---|
| Localização do titular | `POST /v0/oauth/user-discovery` (JSON) | `client_id`, `client_secret`, `user_cpf_cnpj: "CPF"`, `val_cpf_cnpj` | `status` (`S`/`N`), `slots[]` (`slot_alias`, `label`) |
| Autorização | `GET /v0/oauth/authorize` | `response_type=code`, `client_id`, `redirect_uri`, `state`, `scope`, `code_challenge`, `code_challenge_method=S256`, `login_hint` (CPF), `lifetime` (s) | 302 para `redirect_uri` com `code` e `state` (ou `error`) |
| Token | `POST /v0/oauth/token` (form) | `grant_type=authorization_code`, `client_id`, `client_secret`, `code`, `redirect_uri`, `code_verifier` | `access_token`, `token_type`, `expires_in`, `scope`, `authorized_identification_type`, `authorized_identification` |
| Recuperação de certificado | `GET /v0/oauth/certificate-discovery` (Bearer) | — | `status`, `certificates[]` (`alias`, `certificate` em base64) |
| Assinatura | `POST /v0/oauth/signature` (Bearer, JSON) | `certificate_alias`, `hashes[]` (`id`, `alias`, `hash` base64, `hash_algorithm` = `2.16.840.1.101.3.4.2.1`, `signature_format` = `RAW`) | `certificate_alias`, `signatures[]` (`id`, `raw_signature` base64) |

Expected: os nomes batem. Diferença de **nome** (caminho, parâmetro ou campo): anote aqui e, na Task 4, ajuste só `Signatures::Psc::Client::PATHS`/os métodos do cliente e o `FakePsc::App` (os dois mudam juntos; o resto do plano não conhece esses nomes). Diferença de **forma** (por exemplo, o `multi_signature` exigir os hashes já na autorização): **pare** e reporte ao coordenador — muda o lote (Task 13).

- [ ] **Step 4: Crie o worktree (Ambiente de execução) e confira a última migração de cidade**

Run: `ls apps/api/.claude/mod19b/db/city_migrate | tail -2`
Expected: a última é `20261007400003_add_ledi_correction_pending.rb` (ou outra menor que `20261008500001`). Se houver mais nova que `20261008500001`, use um número maior que ela na Task 3 (e no `define(version:)`).

---

### Task 1: Interruptor `digital_signature` (requer `feature:clinical_record`)

**Files:**
- Modify: `app/services/platform/features.rb`
- Create: `app/services/signatures/gate.rb`, `app/controllers/concerns/digital_signature_gate.rb`
- Modify: `spec/adr_pointers_spec.rb`
- Test: `spec/services/platform/digital_signature_feature_spec.rb`

**Interfaces:**
- Consumes: `Platform::Features` (ADR 0028; `clinical_record` com `requires: ["record_mode:record"]`, do 19a), `clinical_city!`, `ledi_maintainer!` (helpers existentes).
- Produces: `Platform::Features::CATALOG` com `digital_signature` (`requires: ["feature:clinical_record"]`); pré-requisito genérico `feature:<chave>` → ausência `"<chave>_disabled"` (ligado **e** utilizável); `Signatures::Gate::KEY == "digital_signature"`, `Signatures::Gate.usable?(city) -> bool` (nil ou cidade sem id → false); concern `DigitalSignatureGate` com `require_digital_signature!` (403 `{ error: "feature_disabled", feature: "digital_signature" }`).

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/services/platform/digital_signature_feature_spec.rb
require "rails_helper"

# ADR 0032 (spec §3): assinatura digital atrás do interruptor
# digital_signature, que só é utilizável com o prontuário (clinical_record)
# utilizável. O maintenance liga pelo mecanismo genérico (catálogo).
RSpec.describe "Interruptor digital_signature" do
  let(:city) { clinical_city! } # clinical_record ligado, record_mode = record

  def set(key, enabled) = Platform::Features.set!(city: city, key: key, enabled: enabled, maintainer: ledi_maintainer!)

  it "está no catálogo e exige o prontuário utilizável" do
    expect(Platform::Features::KEYS).to include("digital_signature")
    set("digital_signature", true)
    expect(Platform::Features.missing(city, "digital_signature")).to eq([])
    expect(Signatures::Gate.usable?(city)).to be(true)

    set("clinical_record", false)
    expect(Platform::Features.missing(city, "digital_signature")).to eq([ "clinical_record_disabled" ])
    expect(Signatures::Gate.usable?(city)).to be(false)

    set("clinical_record", true)
    city.update!(record_mode: "integrated") # clinical_record ligado mas não utilizável
    expect(Platform::Features.missing(city, "digital_signature")).to eq([ "clinical_record_disabled" ])
    expect(Signatures::Gate.usable?(city)).to be(false)
  end

  it "desligado não é utilizável; sem cidade também não" do
    expect(Signatures::Gate.usable?(city)).to be(false)
    expect(Signatures::Gate.usable?(nil)).to be(false)
  end

  it "o resumo do maintenance mostra o interruptor e o que falta" do
    set("digital_signature", true)
    set("clinical_record", false)
    row = Platform::Features.summary(city).find { |r| r[:key] == "digital_signature" }
    expect(row).to include(enabled: true, usable: false, missing: [ "clinical_record_disabled" ])
  end

  it "pré-requisito desconhecido continua levantando" do
    expect { Platform::Features.missing_for("feature-sem-dois-pontos", {}, {}, city: city) }.to raise_error(ArgumentError)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod19b api bundle exec rspec spec/services/platform/digital_signature_feature_spec.rb`
Expected: FAIL (`interruptor fora do catálogo: digital_signature` / `uninitialized constant Signatures`).

- [ ] **Step 3: Implemente**

Em `app/services/platform/features.rb`, acrescente ao `CATALOG` (depois de `clinical_record`):

```ruby
      # ADR 0032: assinatura digital do prontuário (19b); só com o prontuário
      # utilizável. Desligado: tudo como no 19a; o que já foi assinado continua
      # válido e visível.
      Entry.new(key: "digital_signature",
                description: "Assinatura digital ICP-Brasil do prontuário (consulta e adendo, sem papel)",
                requires: %w[feature:clinical_record])
```

Troque a linha de `missing` que monta a lista:

```ruby
      entry.requires.filter_map { |requirement| missing_for(requirement, platform, state, city: city) }
```

e a assinatura e o `case` de `missing_for` (o resto fica igual):

```ruby
    def missing_for(requirement, platform, state, city: nil)
      case requirement
      when "record_mode" then "record_mode_off" if platform[:record_mode] == "off"
      when "record_mode:record" then "record_mode_not_record" unless platform[:record_mode] == "record"
      when "pec_url" then "pec_url_missing" if platform[:pec_url].blank?
      when "ibge_code" then "ibge_code_missing" if state[:ibge_code].blank?
      when /\Acredential:(\w+)\z/
        kind = Regexp.last_match(1)
        if !state[:credentials].key?(kind) then "credential_missing:#{kind}"
        elsif state[:credentials][kind] == "unauthorized" then "credential_unauthorized:#{kind}"
        end
      # ADR 0032: uma funcionalidade que depende de outra exige a outra LIGADA E
      # UTILIZÁVEL (o mesmo state, sem abrir a cidade de novo).
      when /\Afeature:(\w+)\z/
        dependency = Regexp.last_match(1)
        "#{dependency}_disabled" unless usable?(city, dependency, state: state)
      else
        raise ArgumentError, "pré-requisito desconhecido no catálogo: #{requirement}"
      end
    end
```

Confira com `git grep -n "missing_for(" app lib spec` que não há outro chamador (se houver, passe `city:`).

```ruby
# app/services/signatures/gate.rb
# A assinatura digital só existe com o interruptor digital_signature LIGADO e
# UTILIZÁVEL (ADR 0032; requer clinical_record utilizável). Relido da
# plataforma a cada chamada.
module Signatures
  module Gate
    KEY = "digital_signature".freeze

    module_function

    def usable?(city)
      return false if city.nil? || city.id.nil?

      Platform::Features.usable?(city, KEY)
    end
  end
end
```

```ruby
# app/controllers/concerns/digital_signature_gate.rb
# Rotas da assinatura (ADR 0032; contrato): interruptor desligado ou sem o
# prontuário utilizável → 403 feature_disabled, antes de qualquer leitura.
module DigitalSignatureGate
  extend ActiveSupport::Concern

  private

  def require_digital_signature!
    return if Signatures::Gate.usable?(Current.city)

    render json: { error: "feature_disabled", feature: Signatures::Gate::KEY }, status: :forbidden
  end
end
```

Em `spec/adr_pointers_spec.rb`: `VALID_RANGE = (1..32).freeze`.

- [ ] **Step 4: Rode e veja passar (e as specs do catálogo e do maintenance)**

Run: `docker compose exec -T -w /rails/.claude/mod19b api bundle exec rspec spec/services/platform spec/requests/maintenance spec/adr_pointers_spec.rb`
Expected: PASS. Se alguma spec do maintenance fixa a lista de chaves do catálogo (`%w[ledi_export cadsus_lookup clinical_record]`), acrescente `digital_signature` (o mecanismo é genérico; é o contrato §8).

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod19b add app/services/platform/features.rb app/services/signatures/gate.rb app/controllers/concerns/digital_signature_gate.rb spec/adr_pointers_spec.rb spec/services/platform/digital_signature_feature_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod19b commit -m "feat: add the digital_signature switch that requires the clinical record

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

(Se alguma spec do maintenance mudou no Step 4, inclua o caminho no `git add`.)

---
## Fatia 1 — Fundação: certificado, dados, PSC e `signer` (F-19.8, F-19.9, F-19.11)

### Task 2: AC de teste no formato ICP e leitura do certificado

**Por que uma AC de teste, e qual.** O vínculo confere o CPF do certificado (NGS2.01.02) e o `signer` valida cadeia e revogação (NGS2.01.03). Testar isso sem certificado real pede uma AC com o formato da ICP-Brasil. A AC de **dev** é a do `signer` (`DevPki`, Task 2 do plano do `signer`): o container do `signer` a gera no volume `signer-dev-pki` (`root.pem`, `intermediate.pem`, `intermediate.key.pem`, `anchors.pem`, LCRs) e serve as LCRs em `http://signer:8090/dev-pki/`; o api monta o mesmo volume em `/signer-dev-pki` (`SIGNER_DEV_PKI_DIR`) e o PSC falso emite e-CPF de teste com a chave da intermediária, no **mesmo formato** do `DevPki`: RSA 2048, SHA256withRSA, `CN=<NOME>:<CPF>,OU=Teste,O=ICP-Brasil,C=BR`, `keyUsage` digitalSignature + nonRepudiation (crítica), SAN `otherName` 2.16.76.1.3.1 = `[0] EXPLICIT OCTET STRING` com `ddMMaaaa` + CPF + 11 zeros + 15 zeros + 6 zeros, AKI, ponto de distribuição `<base>intermediate.crl`. Sem o volume (CI, ou antes de subir o `signer`), as specs usam uma cópia com os mesmos nomes de arquivo em `spec/fixtures/signature_pki/` (gerada uma vez por `FakePsc::Pki.generate!` e commitada — material de teste, não protege nada). Só as specs `:signer` precisam da AC do volume (o `signer` real confia nela).

**Files:**
- Create: `lib/fake_psc/pki.rb`, `spec/fixtures/signature_pki/{root.pem,intermediate.pem,intermediate.key.pem}` (gerados), `app/services/signatures/certificate_info.rb`
- Test: `spec/services/signatures/certificate_info_spec.rb`

**Interfaces:**
- Produces:
  - `FakePsc::Pki::FIXTURES`, `::ICP_PF_OID == "2.16.76.1.3.1"`, `::DEFAULT_CRL_BASE == "http://signer:8090/dev-pki/"`, `::Leaf = Data(:certificate, :key)` (`#der`, `#serial_hex`); `FakePsc::Pki.generate!(dir = FIXTURES, crl_base: DEFAULT_CRL_BASE)` (escreve `root.pem`, `intermediate.pem`, `intermediate.key.pem`, `anchors.pem`), `.load(dir = ENV["SIGNER_DEV_PKI_DIR"] || FIXTURES, crl_base: ENV["SIGNER_DEV_PKI_CRL_BASE"] || DEFAULT_CRL_BASE)`; `#root`, `#ca` (a intermediária), `#issue(cpf:, name:, birth_date: "01011980", not_before:, not_after:, key_usage: "digitalSignature,nonRepudiation") -> Leaf`, `#leaf_for(cpf, name: "PROFISSIONAL DE TESTE <4 últimos>") -> Leaf` (memorizado por CPF), `#replace_leaf!(cpf, leaf)`.
  - `Signatures::CertificateInfo.parse(der) -> CertificateInfo` (levanta `Signatures::CertificateInfo::Invalid`, sem eco do conteúdo); `#cpf -> String|nil`, `#holder_name`, `#serial_number` (hex maiúsculo), `#issuer_dn` (RFC 2253), `#issuer_name` (CN do emissor), `#not_before`, `#not_after`, `#expired?(now)`, `#signing_usage?` (digitalSignature + nonRepudiation), `#der`.

- [ ] **Step 1: Escreva a AC de teste**

```ruby
# lib/fake_psc/pki.rb
# AC de TESTE no formato ICP-Brasil (DOC-ICP-04), com os arquivos do DevPki do
# signer (root.pem, intermediate.pem, intermediate.key.pem). Em dev, lê o volume
# do signer (SIGNER_DEV_PKI_DIR): o signer confia nela e serve as LCRs. Sem o
# volume, a cópia commitada em spec/fixtures/signature_pki. NUNCA é carregada
# em produção: só por require explícito (specs, lib/fake_psc/config.ru).
require "openssl"
require "fileutils"

module FakePsc
  class Pki
    FIXTURES = File.expand_path("../../spec/fixtures/signature_pki", __dir__)
    ICP_PF_OID = "2.16.76.1.3.1".freeze
    DEFAULT_CRL_BASE = "http://signer:8090/dev-pki/".freeze

    Leaf = Data.define(:certificate, :key) do
      def der = certificate.to_der
      def serial_hex = certificate.serial.to_s(16).upcase
      def inspect = "#<FakePsc::Pki::Leaf #{serial_hex}>"
    end

    attr_reader :root, :ca

    def self.load(dir = ENV["SIGNER_DEV_PKI_DIR"].to_s.empty? ? FIXTURES : ENV["SIGNER_DEV_PKI_DIR"],
                  crl_base: ENV.fetch("SIGNER_DEV_PKI_CRL_BASE", DEFAULT_CRL_BASE))
      read = ->(name) { File.read(File.join(dir, name)) }
      new(root: OpenSSL::X509::Certificate.new(read.call("root.pem")),
          ca: OpenSSL::X509::Certificate.new(read.call("intermediate.pem")),
          ca_key: OpenSSL::PKey.read(read.call("intermediate.key.pem")), crl_base: crl_base)
    end

    # Uma vez só, para a cópia de spec/fixtures (em dev quem gera é o signer).
    def self.generate!(dir = FIXTURES, crl_base: DEFAULT_CRL_BASE)
      FileUtils.mkdir_p(dir)
      root_key = OpenSSL::PKey::RSA.new(2048)
      root = authority("/C=BR/O=ICP-Brasil/CN=AC Raiz de Teste Rota Saude", root_key, nil, root_key, 20, nil, nil)
      ca_key = OpenSSL::PKey::RSA.new(2048)
      ca = authority("/C=BR/O=ICP-Brasil/OU=Teste/CN=AC Rota Saude Teste v1", ca_key, root, root_key, 10, 0,
                     "#{crl_base}root.crl")
      { "root.pem" => root.to_pem, "intermediate.pem" => ca.to_pem, "intermediate.key.pem" => ca_key.private_to_pem,
        "anchors.pem" => root.to_pem + ca.to_pem }.each { |name, pem| File.write(File.join(dir, name), pem) }
      dir
    end

    def self.authority(subject, key, issuer, issuer_key, years, path_len, crl_url)
      cert = OpenSSL::X509::Certificate.new
      cert.version = 2
      cert.serial = OpenSSL::BN.rand(64)
      cert.subject = OpenSSL::X509::Name.parse(subject)
      cert.issuer = issuer ? issuer.subject : cert.subject
      cert.public_key = key.public_key
      cert.not_before = Time.now - 3600
      cert.not_after = Time.now + (years * 365 * 86_400)
      ef = OpenSSL::X509::ExtensionFactory.new(issuer || cert, cert)
      cert.add_extension(ef.create_extension("basicConstraints", path_len ? "CA:TRUE,pathlen:#{path_len}" : "CA:TRUE", true))
      cert.add_extension(ef.create_extension("keyUsage", "keyCertSign,cRLSign", true))
      cert.add_extension(ef.create_extension("subjectKeyIdentifier", "hash"))
      cert.add_extension(ef.create_extension("authorityKeyIdentifier", "keyid:always")) if issuer
      cert.add_extension(ef.create_extension("crlDistributionPoints", "URI:#{crl_url}")) if crl_url
      cert.sign(issuer_key, OpenSSL::Digest.new("SHA256"))
      cert
    end
    private_class_method :authority

    def initialize(root:, ca:, ca_key:, crl_base: DEFAULT_CRL_BASE)
      @root = root
      @ca = ca
      @ca_key = ca_key
      @crl_base = crl_base.end_with?("/") ? crl_base : "#{crl_base}/"
      @leaves = {}
      @mutex = Mutex.new
    end

    # e-CPF de teste no formato do DevPki do signer.
    def issue(cpf:, name:, birth_date: "01011980", not_before: Time.now - 60, not_after: Time.now + (365 * 86_400),
              key_usage: "digitalSignature,nonRepudiation")
      key = OpenSSL::PKey::RSA.new(2048)
      cert = OpenSSL::X509::Certificate.new
      cert.version = 2
      cert.serial = OpenSSL::BN.rand(64)
      cert.subject = OpenSSL::X509::Name.new([ [ "C", "BR" ], [ "O", "ICP-Brasil" ], [ "OU", "Teste" ],
                                               [ "CN", "#{name}:#{cpf}", OpenSSL::ASN1::UTF8STRING ] ])
      cert.issuer = @ca.subject
      cert.public_key = key.public_key
      cert.not_before = not_before
      cert.not_after = not_after
      ef = OpenSSL::X509::ExtensionFactory.new(@ca, cert)
      cert.add_extension(ef.create_extension("keyUsage", key_usage, true))
      cert.add_extension(ef.create_extension("authorityKeyIdentifier", "keyid:always"))
      cert.add_extension(ef.create_extension("crlDistributionPoints", "URI:#{@crl_base}intermediate.crl"))
      cert.add_extension(icp_san(cpf, birth_date))
      cert.sign(@ca_key, OpenSSL::Digest.new("SHA256"))
      Leaf.new(certificate: cert, key: key)
    end

    def leaf_for(cpf, name: "PROFISSIONAL DE TESTE #{cpf.to_s[-4..]}")
      @mutex.synchronize { @leaves[cpf] ||= issue(cpf: cpf, name: name) }
    end

    def replace_leaf!(cpf, leaf) = @mutex.synchronize { @leaves[cpf] = leaf }

    def inspect = "#<FakePsc::Pki>"

    private

    # [0] otherName { 2.16.76.1.3.1, [0] EXPLICIT OCTET STRING } — nascimento
    # (ddMMaaaa) + CPF + NIS (11 zeros) + RG (15 zeros) + órgão/UF (6 zeros).
    def icp_san(cpf, birth_date)
      value = "#{birth_date}#{cpf}#{'0' * 11}#{'0' * 15}#{'0' * 6}"
      other_name = OpenSSL::ASN1::ASN1Data.new(
        [ OpenSSL::ASN1::ObjectId.new(ICP_PF_OID),
          OpenSSL::ASN1::ASN1Data.new([ OpenSSL::ASN1::OctetString.new(value) ], 0, :CONTEXT_SPECIFIC) ],
        0, :CONTEXT_SPECIFIC
      )
      OpenSSL::X509::Extension.new("subjectAltName", OpenSSL::ASN1::Sequence.new([ other_name ]).to_der)
    end
  end
end
```

- [ ] **Step 2: Gere a cópia de teste (uma vez) e confira**

```bash
docker compose exec -T -w /rails/.claude/mod19b api bin/rails runner 'require "./lib/fake_psc/pki"; puts FakePsc::Pki.generate!'
docker compose exec -T -w /rails/.claude/mod19b api openssl verify -CAfile spec/fixtures/signature_pki/root.pem spec/fixtures/signature_pki/intermediate.pem
```
Expected: o caminho de `spec/fixtures/signature_pki` e `intermediate.pem: OK`. Não commite `anchors.pem` (só serve ao `signer`, que em dev usa o próprio volume); apague-o: `rm apps/api/.claude/mod19b/spec/fixtures/signature_pki/anchors.pem`.

- [ ] **Step 3: Escreva a spec que falha**

```ruby
# spec/services/signatures/certificate_info_spec.rb
require "rails_helper"
require_relative "../../../lib/fake_psc/pki"

# ADR 0032 (spec §5; NGS2.01.02/02.02): do certificado ICP-Brasil o api lê o
# CPF (otherName 2.16.76.1.3.1), o nome (CN "NOME:CPF"), o serial, o emissor, a
# validade e o uso de chave.
RSpec.describe Signatures::CertificateInfo do
  let(:pki) { FakePsc::Pki.load(FakePsc::Pki::FIXTURES) }

  it "lê CPF, nome, serial, emissor e validade do certificado de teste" do
    leaf = pki.issue(cpf: "52998224725", name: "MARIA DA SILVA")
    info = described_class.parse(leaf.der)
    expect(info.cpf).to eq("52998224725")
    expect(info.holder_name).to eq("MARIA DA SILVA")
    expect(info.serial_number).to eq(leaf.serial_hex)
    expect(info.issuer_name).to eq("AC Rota Saude Teste v1")
    expect(info.issuer_dn).to include("CN=AC Rota Saude Teste v1")
    expect(info.signing_usage?).to be(true)
    expect(info.expired?).to be(false)
  end

  it "nome com acento; certificado vencido; uso de chave sem não repúdio" do
    leaf = pki.issue(cpf: "11144477735", name: "JOÃO CONCEIÇÃO", not_before: Time.now - 7200, not_after: Time.now - 60,
                     key_usage: "digitalSignature")
    info = described_class.parse(leaf.der)
    expect(info.holder_name).to eq("JOÃO CONCEIÇÃO")
    expect(info.expired?).to be(true)
    expect(info.signing_usage?).to be(false)
  end

  it "sem o otherName ICP não há CPF; bytes que não são certificado levantam Invalid sem eco" do
    expect(described_class.new(pki.ca).cpf).to be_nil
    expect { described_class.parse("lixo-52998224725") }
      .to raise_error(described_class::Invalid) { |error| expect(error.message).not_to include("5299") }
    expect(described_class.parse(pki.issue(cpf: "52998224725", name: "X").der).inspect).not_to include("52998224725")
  end

  it "a cadeia de teste fecha: folha → AC → raiz" do
    leaf = pki.leaf_for("52998224725")
    expect(pki.leaf_for("52998224725")).to equal(leaf) # memorizado por CPF
    store = OpenSSL::X509::Store.new
    store.add_cert(pki.root)
    expect(store.verify(leaf.certificate, [ pki.ca ])).to be(true)
  end
end
```

- [ ] **Step 4: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod19b api bundle exec rspec spec/services/signatures/certificate_info_spec.rb`
Expected: FAIL (`uninitialized constant Signatures::CertificateInfo`).

- [ ] **Step 5: Implemente**

```ruby
# app/services/signatures/certificate_info.rb
# Leitura do certificado ICP-Brasil de pessoa física (DOC-ICP-04; ADR 0032):
# CPF e nascimento no otherName 2.16.76.1.3.1 do subjectAltName (OCTET STRING
# ou texto, conforme a AC), nome no CN ("NOME:CPF"). Nenhuma mensagem de erro e
# nenhum inspect carregam CPF, nome ou bytes do certificado.
require "openssl"

module Signatures
  class CertificateInfo
    ICP_PF_OID = "2.16.76.1.3.1".freeze
    SIGNING_USAGE = [ "Digital Signature", "Non Repudiation" ].freeze

    class Invalid < StandardError; end

    attr_reader :certificate

    def self.parse(der)
      new(OpenSSL::X509::Certificate.new(der))
    rescue OpenSSL::X509::CertificateError, TypeError
      raise Invalid, "certificado ilegível"
    end

    def initialize(certificate)
      @certificate = certificate
    end

    def der = certificate.to_der
    def serial_number = certificate.serial.to_s(16).upcase
    def issuer_dn = certificate.issuer.to_s(OpenSSL::X509::Name::RFC2253)
    def issuer_name = entry(certificate.issuer, "CN") || issuer_dn
    def not_before = certificate.not_before
    def not_after = certificate.not_after
    def expired?(now = Time.current) = now >= not_after || now < not_before
    def holder_name = entry(certificate.subject, "CN").to_s.split(":").first.to_s.strip

    def cpf
      san = certificate.extensions.find { |extension| extension.oid == "subjectAltName" }
      return nil unless san

      names = OpenSSL::ASN1.decode(OpenSSL::ASN1.decode(san.to_der).value.last.value)
      names.value.each do |general_name|
        next unless general_name.tag_class == :CONTEXT_SPECIFIC && general_name.tag.zero?

        type_id, wrapped = general_name.value
        next unless type_id.respond_to?(:oid) && type_id.oid == ICP_PF_OID

        digits = Array(wrapped.value).first&.value.to_s[8, 11]
        return digits if digits&.match?(/\A\d{11}\z/)
      end
      nil
    rescue OpenSSL::ASN1::ASN1Error, NoMethodError, TypeError
      nil
    end

    # NGS2.02.02: digitalSignature + nonRepudiation.
    def signing_usage?
      usage = certificate.extensions.find { |extension| extension.oid == "keyUsage" }&.value.to_s.split(", ")
      (SIGNING_USAGE - Array(usage)).empty?
    end

    def inspect = "#<Signatures::CertificateInfo serial=#{serial_number}>"

    private

    def entry(name, key) = name.to_a.find { |(k, _value, _type)| k == key }&.at(1)
  end
end
```

- [ ] **Step 6: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod19b api bundle exec rspec spec/services/signatures/certificate_info_spec.rb`
Expected: PASS.

- [ ] **Step 7: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod19b add lib/fake_psc/pki.rb spec/fixtures/signature_pki/root.pem spec/fixtures/signature_pki/intermediate.pem spec/fixtures/signature_pki/intermediate.key.pem app/services/signatures/certificate_info.rb spec/services/signatures/certificate_info_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod19b commit -m "feat: read ICP-Brasil certificates and add a test certificate authority

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Dados da assinatura — tabelas, triggers, cifra, eventos e helpers

**Files:**
- Create: `db/city_migrate/20261008500001_add_digital_signatures.rb`, `app/models/{signer_certificate,signature_session,signature_oauth_state,signature_request,signature}.rb`, `app/services/signatures/document_types.rb`, `spec/support/signature_helpers.rb`
- Modify: `db/city_schema.rb`, `db/city_triggers.sql`, `app/services/city_encryption.rb`, `config/initializers/domain_events.rb`, `config/initializers/filter_parameter_logging.rb`, `spec/initializers/domain_events_bindings_spec.rb`, `spec/rails_helper.rb`
- Test: `spec/models/signature_tables_guard_spec.rb`

**Interfaces:**
- Consumes: `CityEncryption.allowing_reencryption` (19a), `rota_append_only()` (função existente em `db/city_triggers.sql`), `FakePsc::Pki` (Task 2), `doctor!`/`clinical_city!` (19a).
- Produces:
  - Tabelas `signer_certificates`, `signature_sessions`, `signature_oauth_states`, `signature_requests`, `signatures` (colunas no Step 3); índices `idx_signer_certificates_one_active`, `idx_signature_sessions_one_active`, `idx_signature_requests_document` (único), `idx_signature_requests_queue`, `idx_signatures_document` (único), `index_signatures_on_signature_request_id` (único); triggers `signatures_guard`, `signatures_append_only_truncate`, `signature_requests_guard`, `signature_requests_append_only_truncate`.
  - `signer_certificates.link_check_status` (`valid`|`indeterminate`) e `.link_check_reasons` (string[]): o resultado do `POST /certificates/check` no vínculo.
  - Modelos: `SignerCertificate` (`PROVIDERS`, `STATUSES`, `EXPIRING_WITHIN == 30.days`, `scope :active`, `#der`, `#info -> Signatures::CertificateInfo`, `#expires_in_days(now)`, `#expiring?(now)`); `SignatureSession` (`STATUSES`, `SCOPE == "signature_session"`, `MAX_LIFETIME == 12.hours`, `.usable_for(user_id, now:)`, `.lapsed?(user_id, now:)`); `SignatureOauthState` (`PURPOSES`, `TTL == 10.minutes`); `SignatureRequest` (`STATUSES`, `REASONS`, `TRANSIENT_REASONS`, `MIN_RETURN_NOTE == 10`, `MAX_RETURN_NOTE == 500`, `scope :pending`, `#pending?`, `#document`, `has_one :signature`); `Signature` (`POLICIES`, `VERIFICATIONS`, `#cades_bytes`, `#signed_pdf_bytes`, `#material -> Hash`).
  - `Signatures::DocumentTypes::MODELS`, `.api(db_type) -> "consultation"|"consultation_addendum"`, `.db(document) -> "Consultation"|"ConsultationAddendum"`, `.find(db_type, id)`, `.consultation_id(document)`.
  - Eventos declarados (`to: []`): `signature.certificate_linked`, `signature.certificate_unlinked`, `signature.session_opened`, `signature.signed`, `signature.failed`, `signature.returned_to_paper`, `signature.verified`.
  - Helpers (`spec/support/signature_helpers.rb`, módulo `SignatureHelpers`): `DOCTOR_CPF == "52998224725"`, `OTHER_CPF == "11144477735"`, `test_pki`, `signature_city!(enabled: true) -> City`, `signer_doctor!(unit, cpf: DOCTOR_CPF)`, `linked_certificate!(user, provider: "vidaas", status: "active", leaf: nil) -> SignerCertificate`, `signature_session!(user, certificate:, token: "token-de-teste", expires_at: 8.hours.from_now) -> SignatureSession`, `signature_request!(document = nil, author:, status: "pending", reason_code: nil) -> SignatureRequest`, `signature_row!(request, certificate:, canonical_json: "{\"a\":1}") -> Signature`, `attempt { }` (savepoint).

- [ ] **Step 1: Os helpers de spec**

```ruby
# spec/support/signature_helpers.rb
# Assinatura digital (ADR 0032). Os helpers que falam com o PSC falso e com o
# signer falso entram nas Tasks 4 e 5, neste mesmo módulo.
require_relative "../../lib/fake_psc/pki"

module SignatureHelpers
  DOCTOR_CPF = "52998224725".freeze
  OTHER_CPF = "11144477735".freeze

  # Uma AC de teste por processo (as folhas são memorizadas por CPF).
  def test_pki = ($signature_test_pki ||= FakePsc::Pki.load)

  # TEST_CITY_A com o prontuário e a assinatura ligados (ou não).
  def signature_city!(enabled: true)
    city = clinical_city!
    Platform::Features.set!(city: city, key: "digital_signature", enabled: enabled, maintainer: ledi_maintainer!)
    CityCatalog.reset_cache!
    city
  end

  def signer_doctor!(unit, cpf: DOCTOR_CPF)
    doctor!(unit).tap { |user| user.professional.update!(cpf: cpf) }
  end

  def linked_certificate!(user, provider: "vidaas", status: "active", leaf: nil)
    leaf ||= test_pki.leaf_for(user.professional.cpf)
    info = Signatures::CertificateInfo.new(leaf.certificate)
    SignerCertificate.create!(user: user, provider: provider, certificate_alias: info.cpf, serial_number: info.serial_number,
                              issuer_dn: info.issuer_dn, subject_cpf: info.cpf, not_before: info.not_before,
                              not_after: info.not_after, status: status, certificate_der: Base64.strict_encode64(leaf.der))
  end

  def signature_session!(user, certificate:, token: "token-de-teste", expires_at: 8.hours.from_now, started_at: Time.current)
    SignatureSession.create!(user: user, signer_certificate: certificate, provider: certificate.provider, access_token: token,
                             scope: SignatureSession::SCOPE, started_at: started_at, expires_at: expires_at)
  end

  # Sem documento: um id qualquer (a tabela não tem FK para o documento).
  def signature_request!(document = nil, author:, status: "pending", reason_code: nil)
    type = document ? Signatures::DocumentTypes.db(document) : "Consultation"
    SignatureRequest.create!(document_type: type, document_id: document&.id || SecureRandom.uuid,
                             consultation_id: document ? Signatures::DocumentTypes.consultation_id(document) : SecureRandom.uuid,
                             author_user_id: author.id, status: status, reason_code: reason_code,
                             resolved_at: %w[signed returned_to_paper].include?(status) ? Time.current : nil)
  end

  def signature_row!(request, certificate:, canonical_json: "{\"a\":1}")
    Signature.create!(signature_request: request, document_type: request.document_type, document_id: request.document_id,
                      canonical_json: canonical_json, canonical_sha256: Digest::SHA256.hexdigest(canonical_json),
                      cades: Base64.strict_encode64("cades"), signed_pdf: Base64.strict_encode64("%PDF-1.7 assinado"),
                      pdf_sha256: Digest::SHA256.hexdigest("%PDF-1.7 assinado"), policy: "AD-RB",
                      policy_oid: "2.16.76.1.7.1.1.2.3", validation_material: { "cades" => "", "pades" => "" }.to_json,
                      signer_certificate: certificate, signer_cpf: certificate.subject_cpf, signed_at: Time.current,
                      last_verification: "valid", last_verification_at: Time.current)
  end

  def attempt(&) = ApplicationRecord.transaction(requires_new: true, &)
end

RSpec.configure { |c| c.include SignatureHelpers }
```

Em `spec/rails_helper.rb`, depois de `require_relative "support/clinical_record_helpers"`: `require_relative "support/signature_helpers"`.

- [ ] **Step 2: Escreva a spec de guarda que falha**

```ruby
# spec/models/signature_tables_guard_spec.rb
require "rails_helper"

# ADR 0032 (spec §4; invariantes): um certificado e uma sessão ativos por
# usuário; um pedido por documento; uma assinatura por pedido e por documento;
# assinatura gravada não muda (só a validação e a re-cifra); pedido resolvido
# não volta; segredos e documento cifrados com a chave da cidade.
RSpec.describe "Tabelas da assinatura digital" do
  before { Current.city = signature_city! }
  after { Current.reset }

  let(:unit) { create_unit }
  let(:doctor) { signer_doctor!(unit) }
  let(:certificate) { linked_certificate!(doctor) }

  def raw(table, column, id)
    ApplicationRecord.connection.select_value("SELECT #{column} FROM #{table} WHERE id = #{ApplicationRecord.connection.quote(id)}")
  end

  describe "signer_certificates e signature_sessions" do
    it "um ativo por usuário; o CPF e o certificado ficam cifrados" do
      certificate
      expect { attempt { linked_certificate!(doctor) } }.to raise_error(ActiveRecord::RecordNotUnique)
      expect { attempt { linked_certificate!(doctor, status: "replaced") } }.not_to raise_error
      expect(raw("signer_certificates", "subject_cpf", certificate.id)).not_to include(SignatureHelpers::DOCTOR_CPF)
      expect(certificate.reload.info.cpf).to eq(SignatureHelpers::DOCTOR_CPF)
      expect(certificate.inspect).not_to include(SignatureHelpers::DOCTOR_CPF)
    end

    it "uma sessão ativa por usuário; até 12 h; o token fica cifrado e fora do inspect" do
      session = signature_session!(doctor, certificate: certificate, token: "TOKEN-MARCADOR")
      expect { attempt { signature_session!(doctor, certificate: certificate) } }.to raise_error(ActiveRecord::RecordNotUnique)
      session.update!(status: "revoked")
      expect { attempt { signature_session!(doctor, certificate: certificate, expires_at: 13.hours.from_now) } }
        .to raise_error(ActiveRecord::StatementInvalid, /ck_signature_sessions_lifetime/)
      expect(raw("signature_sessions", "access_token", session.id)).not_to include("TOKEN-MARCADOR")
      expect(session.inspect).not_to include("TOKEN-MARCADOR")
    end
  end

  describe "signature_requests" do
    it "um por documento; identidade fixa; nunca some; resolvido não volta" do
      request = signature_request!(author: doctor)
      dup = request.attributes.slice("document_type", "document_id", "consultation_id", "author_user_id")
      expect { attempt { SignatureRequest.create!(dup) } }.to raise_error(ActiveRecord::RecordNotUnique)
      expect { attempt { request.update_columns(author_user_id: verifier!.id) } }
        .to raise_error(ActiveRecord::StatementInvalid, /identity columns never change/)
      expect { attempt { request.delete } }.to raise_error(ActiveRecord::StatementInvalid, /DELETE refused/)
      request.update!(reason_code: "no_session", attempts: 1)
      request.update!(status: "returned_to_paper", reason_code: "user_request", return_note: "sem certificado hoje", resolved_at: Time.current)
      expect { attempt { request.update!(status: "pending", resolved_at: nil) } }
        .to raise_error(ActiveRecord::StatementInvalid, /resolved signature request never changes/)
      expect(raw("signature_requests", "return_note", request.id)).not_to include("sem certificado")
    end

    it "CHECKs: motivo do catálogo; resolvido exige resolved_at; nota só na volta ao papel" do
      expect { attempt { signature_request!(author: doctor, reason_code: "porque_sim") } }
        .to raise_error(ActiveRecord::StatementInvalid, /ck_signature_requests_reason/)
      expect { attempt { SignatureRequest.create!(document_type: "Consultation", document_id: SecureRandom.uuid, consultation_id: SecureRandom.uuid, author_user_id: doctor.id, status: "signed") } }
        .to raise_error(ActiveRecord::StatementInvalid, /ck_signature_requests_resolution/)
      expect { attempt { signature_request!(author: doctor).update!(return_note: "nota fora de hora") } }
        .to raise_error(ActiveRecord::StatementInvalid, /ck_signature_requests_return_note/)
    end
  end

  describe "signatures" do
    let(:request) { signature_request!(author: doctor) }

    it "uma por pedido e por documento; conteúdo nunca muda; validação muda; não some" do
      signature = signature_row!(request, certificate: certificate, canonical_json: "{\"MARCADOR\":1}")
      expect { attempt { signature_row!(request, certificate: certificate) } }.to raise_error(ActiveRecord::RecordNotUnique)
      expect { attempt { signature.update!(policy: "AD-RT") } }.to raise_error(ActiveRecord::StatementInvalid, /never changes/)
      expect { attempt { signature.update!(canonical_json: "{}") } }.to raise_error(ActiveRecord::StatementInvalid, /never changes/)
      expect { signature.update!(last_verification: "indeterminate", last_verification_at: Time.current, last_verification_reasons: [ "crl_unavailable" ]) }
        .not_to raise_error
      expect { attempt { signature.delete } }.to raise_error(ActiveRecord::StatementInvalid, /DELETE refused/)
      expect { attempt { ApplicationRecord.connection.execute("TRUNCATE signatures CASCADE") } }.to raise_error(ActiveRecord::StatementInvalid)
      expect(raw("signatures", "canonical_json", signature.id)).not_to include("MARCADOR")
    end

    it "re-cifra de assinatura gravada: só as colunas cifradas, conteúdo idêntico (Review Focus 2)" do
      signature = signature_row!(request, certificate: certificate, canonical_json: "{\"MARCADOR\":1}")
      before = raw("signatures", "canonical_json", signature.id)
      expect { CityEncryption.allowing_reencryption { signature.encrypt } }.not_to raise_error
      expect { CityEncryption.allowing_reencryption { certificate.encrypt } }.not_to raise_error
      request.update!(status: "returned_to_paper", reason_code: "user_request", return_note: "devolvido ao papel", resolved_at: Time.current)
      expect { CityEncryption.allowing_reencryption { request.encrypt } }.not_to raise_error
      reloaded = signature.reload
      expect(raw("signatures", "canonical_json", signature.id)).not_to eq(before)
      expect(Digest::SHA256.hexdigest(reloaded.canonical_json)).to eq(signature.canonical_sha256)
      expect do
        attempt do
          ApplicationRecord.connection.execute("SET LOCAL rota.reencrypting = 'on'")
          signature.update_columns(policy_oid: "1.2.3")
        end
      end.to raise_error(ActiveRecord::StatementInvalid, /never changes/)
    end
  end

  it "todo atributo cifrado novo está na lista da re-cifra" do
    expect(CityEncryption::CITY_KEYED_TARGETS).to include(
      [ SignerCertificate, :subject_cpf ], [ SignerCertificate, :certificate_der ], [ SignatureSession, :access_token ],
      [ SignatureOauthState, :code_verifier ], [ SignatureRequest, :return_note ], [ Signature, :canonical_json ],
      [ Signature, :cades ], [ Signature, :signed_pdf ], [ Signature, :validation_material ], [ Signature, :signer_cpf ]
    )
  end
end
```

- [ ] **Step 3: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod19b api bundle exec rspec spec/models/signature_tables_guard_spec.rb`
Expected: FAIL (`uninitialized constant SignerCertificate`).

- [ ] **Step 4: A migração**

```ruby
# db/city_migrate/20261008500001_add_digital_signatures.rb
# Assinatura digital do prontuário (ADR 0032; spec §4). O certificado
# vinculado, a sessão do turno, o state de uso único do OAuth, o pedido (a
# fila) e a assinatura (só acréscimo). O banco garante: um certificado e uma
# sessão ativos por usuário, um pedido por documento, uma assinatura por pedido
# e por documento, assinatura gravada nunca muda (só a validação e a re-cifra),
# pedido resolvido nunca volta. As tabelas do 19a não mudam.
class AddDigitalSignatures < ActiveRecord::Migration[8.1]
  PROVIDERS = %w[vidaas birdid safeid neoid remoteid].freeze
  REASONS = %w[no_session session_expired provider_unavailable provider_rejected signer_unavailable verification_failed
               certificate_expired certificate_revoked certificate_cpf_mismatch feature_disabled user_request].freeze
  DOCUMENT_TYPES = %w[Consultation ConsultationAddendum].freeze

  def text_in(column, values) = "#{column}::text = ANY (ARRAY[#{values.map { |v| "'#{v}'::text" }.join(', ')}])"

  def checks(table, map) = map.each { |name, expression| add_check_constraint table, expression, name: name }

  def up
    create_table :signer_certificates, id: :uuid do |t|
      t.uuid :user_id, null: false
      t.string :provider, null: false
      t.string :certificate_alias, null: false
      t.string :serial_number, null: false
      t.text :issuer_dn, null: false
      t.text :subject_cpf, null: false
      t.datetime :not_before, null: false
      t.datetime :not_after, null: false
      t.string :status, null: false, default: "active"
      t.text :certificate_der, null: false
      # Resultado do POST /certificates/check do signer no vínculo (indeterminate =
      # LCR fora do ar; a revogação volta a ser conferida na primeira assinatura).
      t.string :link_check_status, null: false, default: "valid"
      t.string :link_check_reasons, array: true, null: false, default: []
      t.timestamps
    end
    add_index :signer_certificates, :user_id, unique: true, where: "((status)::text = 'active'::text)",
                                              name: "idx_signer_certificates_one_active"
    add_index :signer_certificates, :user_id
    add_foreign_key :signer_certificates, :users
    checks(:signer_certificates,
           "ck_signer_certificates_provider" => text_in("provider", PROVIDERS),
           "ck_signer_certificates_status" => text_in("status", %w[active replaced unlinked revoked expired]),
           "ck_signer_certificates_validity" => "not_after > not_before",
           "ck_signer_certificates_link_check" => text_in("link_check_status", %w[valid indeterminate]))

    create_table :signature_sessions, id: :uuid do |t|
      t.uuid :user_id, null: false
      t.uuid :signer_certificate_id, null: false
      t.string :provider, null: false
      t.text :access_token, null: false
      t.string :scope, null: false
      t.datetime :started_at, null: false
      t.datetime :expires_at, null: false
      t.string :status, null: false, default: "active"
      t.timestamps
    end
    add_index :signature_sessions, :user_id, unique: true, where: "((status)::text = 'active'::text)",
                                             name: "idx_signature_sessions_one_active"
    add_index :signature_sessions, :signer_certificate_id
    add_foreign_key :signature_sessions, :users
    add_foreign_key :signature_sessions, :signer_certificates
    checks(:signature_sessions,
           "ck_signature_sessions_provider" => text_in("provider", PROVIDERS),
           "ck_signature_sessions_status" => text_in("status", %w[active expired revoked]),
           "ck_signature_sessions_scope" => "scope::text = 'signature_session'::text",
           "ck_signature_sessions_lifetime" => "expires_at > started_at AND expires_at <= (started_at + '12:00:00'::interval)")

    create_table :signature_oauth_states, id: :uuid do |t|
      t.uuid :user_id, null: false
      t.string :purpose, null: false
      t.string :provider, null: false
      t.text :code_verifier, null: false
      t.uuid :request_ids, array: true, null: false, default: []
      t.string :return_to, null: false, default: "/"
      t.datetime :expires_at, null: false
      t.datetime :consumed_at
      t.datetime :created_at, null: false
    end
    add_index :signature_oauth_states, :user_id
    add_index :signature_oauth_states, :expires_at
    add_foreign_key :signature_oauth_states, :users
    checks(:signature_oauth_states,
           "ck_signature_oauth_states_purpose" => text_in("purpose", %w[link session batch]),
           "ck_signature_oauth_states_provider" => text_in("provider", PROVIDERS),
           "ck_signature_oauth_states_batch" => "((purpose)::text = 'batch'::text) = (cardinality(request_ids) > 0) AND cardinality(request_ids) <= 50",
           "ck_signature_oauth_states_return_to" => "(return_to)::text ~ '^/([^/].*)?$'::text")

    create_table :signature_requests, id: :uuid do |t|
      t.string :document_type, null: false
      t.uuid :document_id, null: false
      t.uuid :consultation_id, null: false
      t.uuid :author_user_id, null: false
      t.string :status, null: false, default: "pending"
      t.string :reason_code
      t.text :return_note
      t.integer :attempts, null: false, default: 0
      t.datetime :resolved_at
      t.timestamps
    end
    add_index :signature_requests, %i[document_type document_id], unique: true, name: "idx_signature_requests_document"
    add_index :signature_requests, %i[author_user_id status created_at], name: "idx_signature_requests_queue"
    add_index :signature_requests, :consultation_id
    add_foreign_key :signature_requests, :users, column: :author_user_id
    checks(:signature_requests,
           "ck_signature_requests_document_type" => text_in("document_type", DOCUMENT_TYPES),
           "ck_signature_requests_status" => text_in("status", %w[pending signed failed returned_to_paper]),
           "ck_signature_requests_reason" => "reason_code IS NULL OR #{text_in('reason_code', REASONS)}",
           "ck_signature_requests_attempts" => "attempts >= 0",
           "ck_signature_requests_resolution" => "(#{text_in('status', %w[signed returned_to_paper])}) = (resolved_at IS NOT NULL)",
           "ck_signature_requests_return_note" => "(status)::text = 'returned_to_paper'::text OR return_note IS NULL")

    create_table :signatures, id: :uuid do |t|
      t.uuid :signature_request_id, null: false
      t.string :document_type, null: false
      t.uuid :document_id, null: false
      t.text :canonical_json, null: false
      t.string :canonical_sha256, null: false
      t.text :cades, null: false
      t.text :signed_pdf, null: false
      t.string :pdf_sha256, null: false
      t.string :policy, null: false, default: "AD-RB"
      t.string :policy_oid, null: false
      t.text :validation_material, null: false
      t.uuid :signer_certificate_id, null: false
      t.text :signer_cpf, null: false
      t.datetime :signed_at, null: false
      t.string :last_verification, null: false
      t.datetime :last_verification_at, null: false
      t.string :last_verification_reasons, array: true, null: false, default: []
      t.datetime :created_at, null: false
    end
    add_index :signatures, :signature_request_id, unique: true
    add_index :signatures, %i[document_type document_id], unique: true, name: "idx_signatures_document"
    add_index :signatures, :signer_certificate_id
    add_index :signatures, :last_verification
    add_foreign_key :signatures, :signature_requests
    add_foreign_key :signatures, :signer_certificates
    checks(:signatures,
           "ck_signatures_document_type" => text_in("document_type", DOCUMENT_TYPES),
           "ck_signatures_policy" => text_in("policy", %w[AD-RB AD-RT]),
           "ck_signatures_verification" => text_in("last_verification", %w[valid invalid indeterminate]),
           "ck_signatures_sha256" => "(canonical_sha256)::text ~ '^[0-9a-f]{64}$'::text AND (pdf_sha256)::text ~ '^[0-9a-f]{64}$'::text")

    execute File.read(Rails.root.join("db/city_triggers.sql"))
  end

  def down
    raise ActiveRecord::IrreversibleMigration
  end
end
```

- [ ] **Step 5: Os triggers (fim de `db/city_triggers.sql`)**

```sql
-- ADR 0032: assinatura gravada não muda. Só a validação (estado, instante e
-- motivos) é regravada; sob rota.reencrypting (CityEncryption.allowing_reencryption),
-- só as colunas cifradas. DELETE e TRUNCATE recusados.
CREATE OR REPLACE FUNCTION rota_signature_guard() RETURNS trigger AS $fn$
DECLARE
  verification text[] := ARRAY['last_verification', 'last_verification_at', 'last_verification_reasons'];
  encrypted text[] := ARRAY['canonical_json', 'cades', 'signed_pdf', 'validation_material', 'signer_cpf'];
BEGIN
  IF TG_OP = 'DELETE' THEN
    RAISE EXCEPTION 'signatures is append-only: DELETE refused';
  END IF;
  IF (to_jsonb(NEW) - verification) = (to_jsonb(OLD) - verification) THEN
    RETURN NEW;
  END IF;
  IF current_setting('rota.reencrypting', true) = 'on' AND (to_jsonb(NEW) - encrypted) = (to_jsonb(OLD) - encrypted) THEN
    RETURN NEW;
  END IF;
  RAISE EXCEPTION 'a recorded signature never changes';
END;
$fn$ LANGUAGE plpgsql;

-- ADR 0032: o pedido não troca de documento nem de autor, não some e não sai de
-- estado resolvido (signed, returned_to_paper) — exceto a re-cifra do motivo.
CREATE OR REPLACE FUNCTION rota_signature_request_guard() RETURNS trigger AS $fn$
BEGIN
  IF TG_OP = 'DELETE' THEN
    RAISE EXCEPTION 'signature_requests is append-only: DELETE refused';
  END IF;
  IF (NEW.document_type, NEW.document_id, NEW.consultation_id, NEW.author_user_id)
     IS DISTINCT FROM (OLD.document_type, OLD.document_id, OLD.consultation_id, OLD.author_user_id) THEN
    RAISE EXCEPTION 'signature request identity columns never change';
  END IF;
  IF OLD.status IN ('signed', 'returned_to_paper') THEN
    IF current_setting('rota.reencrypting', true) = 'on' AND (to_jsonb(NEW) - 'return_note') = (to_jsonb(OLD) - 'return_note') THEN
      RETURN NEW;
    END IF;
    RAISE EXCEPTION 'a resolved signature request never changes';
  END IF;
  RETURN NEW;
END;
$fn$ LANGUAGE plpgsql;

DO $do$
BEGIN
  IF to_regclass('public.signatures') IS NOT NULL THEN
    EXECUTE 'DROP TRIGGER IF EXISTS signatures_guard ON signatures';
    EXECUTE 'CREATE TRIGGER signatures_guard
      BEFORE UPDATE OR DELETE ON signatures
      FOR EACH ROW EXECUTE FUNCTION rota_signature_guard()';
    EXECUTE 'DROP TRIGGER IF EXISTS signatures_append_only_truncate ON signatures';
    EXECUTE 'CREATE TRIGGER signatures_append_only_truncate
      BEFORE TRUNCATE ON signatures
      FOR EACH STATEMENT EXECUTE FUNCTION rota_append_only()';
  END IF;
  IF to_regclass('public.signature_requests') IS NOT NULL THEN
    EXECUTE 'DROP TRIGGER IF EXISTS signature_requests_guard ON signature_requests';
    EXECUTE 'CREATE TRIGGER signature_requests_guard
      BEFORE UPDATE OR DELETE ON signature_requests
      FOR EACH ROW EXECUTE FUNCTION rota_signature_request_guard()';
    EXECUTE 'DROP TRIGGER IF EXISTS signature_requests_append_only_truncate ON signature_requests';
    EXECUTE 'CREATE TRIGGER signature_requests_append_only_truncate
      BEFORE TRUNCATE ON signature_requests
      FOR EACH STATEMENT EXECUTE FUNCTION rota_append_only()';
  END IF;
END
$do$;
```

- [ ] **Step 6: Dump à mão em `db/city_schema.rb`**

Troque `define(version: 2026_10_07_400003)` por `define(version: 2026_10_08_500001)`. Tabelas novas (ordem alfabética entre as existentes; colunas em ordem alfabética, como as do 19a):

```ruby
  create_table "signature_oauth_states", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.text "code_verifier", null: false
    t.datetime "consumed_at"
    t.datetime "created_at", null: false
    t.datetime "expires_at", null: false
    t.string "provider", null: false
    t.string "purpose", null: false
    t.uuid "request_ids", default: [], null: false, array: true
    t.string "return_to", default: "/", null: false
    t.uuid "user_id", null: false
    t.index ["expires_at"], name: "index_signature_oauth_states_on_expires_at"
    t.index ["user_id"], name: "index_signature_oauth_states_on_user_id"
    t.check_constraint "((purpose)::text = 'batch'::text) = (cardinality(request_ids) > 0) AND cardinality(request_ids) <= 50", name: "ck_signature_oauth_states_batch"
    t.check_constraint "provider::text = ANY (ARRAY['vidaas'::text, 'birdid'::text, 'safeid'::text, 'neoid'::text, 'remoteid'::text])", name: "ck_signature_oauth_states_provider"
    t.check_constraint "purpose::text = ANY (ARRAY['link'::text, 'session'::text, 'batch'::text])", name: "ck_signature_oauth_states_purpose"
    t.check_constraint "(return_to)::text ~ '^/([^/].*)?$'::text", name: "ck_signature_oauth_states_return_to"
  end

  create_table "signature_requests", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.integer "attempts", default: 0, null: false
    t.uuid "author_user_id", null: false
    t.uuid "consultation_id", null: false
    t.datetime "created_at", null: false
    t.uuid "document_id", null: false
    t.string "document_type", null: false
    t.string "reason_code"
    t.datetime "resolved_at"
    t.text "return_note"
    t.string "status", default: "pending", null: false
    t.datetime "updated_at", null: false
    t.index ["author_user_id", "status", "created_at"], name: "idx_signature_requests_queue"
    t.index ["consultation_id"], name: "index_signature_requests_on_consultation_id"
    t.index ["document_type", "document_id"], name: "idx_signature_requests_document", unique: true
    t.check_constraint "attempts >= 0", name: "ck_signature_requests_attempts"
    t.check_constraint "document_type::text = ANY (ARRAY['Consultation'::text, 'ConsultationAddendum'::text])", name: "ck_signature_requests_document_type"
    t.check_constraint "reason_code IS NULL OR reason_code::text = ANY (ARRAY['no_session'::text, 'session_expired'::text, 'provider_unavailable'::text, 'provider_rejected'::text, 'signer_unavailable'::text, 'verification_failed'::text, 'certificate_expired'::text, 'certificate_revoked'::text, 'certificate_cpf_mismatch'::text, 'feature_disabled'::text, 'user_request'::text])", name: "ck_signature_requests_reason"
    t.check_constraint "(status::text = ANY (ARRAY['signed'::text, 'returned_to_paper'::text])) = (resolved_at IS NOT NULL)", name: "ck_signature_requests_resolution"
    t.check_constraint "(status)::text = 'returned_to_paper'::text OR return_note IS NULL", name: "ck_signature_requests_return_note"
    t.check_constraint "status::text = ANY (ARRAY['pending'::text, 'signed'::text, 'failed'::text, 'returned_to_paper'::text])", name: "ck_signature_requests_status"
  end

  create_table "signature_sessions", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.text "access_token", null: false
    t.datetime "created_at", null: false
    t.datetime "expires_at", null: false
    t.string "provider", null: false
    t.string "scope", null: false
    t.uuid "signer_certificate_id", null: false
    t.datetime "started_at", null: false
    t.string "status", default: "active", null: false
    t.datetime "updated_at", null: false
    t.uuid "user_id", null: false
    t.index ["signer_certificate_id"], name: "index_signature_sessions_on_signer_certificate_id"
    t.index ["user_id"], name: "idx_signature_sessions_one_active", unique: true, where: "((status)::text = 'active'::text)"
    t.check_constraint "expires_at > started_at AND expires_at <= (started_at + '12:00:00'::interval)", name: "ck_signature_sessions_lifetime"
    t.check_constraint "provider::text = ANY (ARRAY['vidaas'::text, 'birdid'::text, 'safeid'::text, 'neoid'::text, 'remoteid'::text])", name: "ck_signature_sessions_provider"
    t.check_constraint "scope::text = 'signature_session'::text", name: "ck_signature_sessions_scope"
    t.check_constraint "status::text = ANY (ARRAY['active'::text, 'expired'::text, 'revoked'::text])", name: "ck_signature_sessions_status"
  end

  create_table "signatures", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.text "cades", null: false
    t.text "canonical_json", null: false
    t.string "canonical_sha256", null: false
    t.datetime "created_at", null: false
    t.uuid "document_id", null: false
    t.string "document_type", null: false
    t.string "last_verification", null: false
    t.datetime "last_verification_at", null: false
    t.string "last_verification_reasons", default: [], null: false, array: true
    t.string "pdf_sha256", null: false
    t.string "policy", default: "AD-RB", null: false
    t.string "policy_oid", null: false
    t.datetime "signed_at", null: false
    t.text "signed_pdf", null: false
    t.uuid "signature_request_id", null: false
    t.uuid "signer_certificate_id", null: false
    t.text "signer_cpf", null: false
    t.text "validation_material", null: false
    t.index ["document_type", "document_id"], name: "idx_signatures_document", unique: true
    t.index ["last_verification"], name: "index_signatures_on_last_verification"
    t.index ["signature_request_id"], name: "index_signatures_on_signature_request_id", unique: true
    t.index ["signer_certificate_id"], name: "index_signatures_on_signer_certificate_id"
    t.check_constraint "document_type::text = ANY (ARRAY['Consultation'::text, 'ConsultationAddendum'::text])", name: "ck_signatures_document_type"
    t.check_constraint "last_verification::text = ANY (ARRAY['valid'::text, 'invalid'::text, 'indeterminate'::text])", name: "ck_signatures_verification"
    t.check_constraint "policy::text = ANY (ARRAY['AD-RB'::text, 'AD-RT'::text])", name: "ck_signatures_policy"
    t.check_constraint "(canonical_sha256)::text ~ '^[0-9a-f]{64}$'::text AND (pdf_sha256)::text ~ '^[0-9a-f]{64}$'::text", name: "ck_signatures_sha256"
  end

  create_table "signer_certificates", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.string "certificate_alias", null: false
    t.text "certificate_der", null: false
    t.datetime "created_at", null: false
    t.text "issuer_dn", null: false
    t.string "link_check_reasons", default: [], null: false, array: true
    t.string "link_check_status", default: "valid", null: false
    t.datetime "not_after", null: false
    t.datetime "not_before", null: false
    t.string "provider", null: false
    t.string "serial_number", null: false
    t.string "status", default: "active", null: false
    t.text "subject_cpf", null: false
    t.datetime "updated_at", null: false
    t.uuid "user_id", null: false
    t.index ["user_id"], name: "idx_signer_certificates_one_active", unique: true, where: "((status)::text = 'active'::text)"
    t.index ["user_id"], name: "index_signer_certificates_on_user_id"
    t.check_constraint "not_after > not_before", name: "ck_signer_certificates_validity"
    t.check_constraint "link_check_status::text = ANY (ARRAY['valid'::text, 'indeterminate'::text])", name: "ck_signer_certificates_link_check"
    t.check_constraint "provider::text = ANY (ARRAY['vidaas'::text, 'birdid'::text, 'safeid'::text, 'neoid'::text, 'remoteid'::text])", name: "ck_signer_certificates_provider"
    t.check_constraint "status::text = ANY (ARRAY['active'::text, 'replaced'::text, 'unlinked'::text, 'revoked'::text, 'expired'::text])", name: "ck_signer_certificates_status"
  end
```

E, no bloco de chaves estrangeiras (ordem alfabética):

```ruby
  add_foreign_key "signature_oauth_states", "users"
  add_foreign_key "signature_requests", "users", column: "author_user_id"
  add_foreign_key "signature_sessions", "signer_certificates"
  add_foreign_key "signature_sessions", "users"
  add_foreign_key "signatures", "signature_requests"
  add_foreign_key "signatures", "signer_certificates"
  add_foreign_key "signer_certificates", "users"
```

Se `spec/services/city_schema_spec.rb` acusar diferença só de forma (parênteses, `::text`), copie a forma que a spec imprime para o dump — a migração é a fonte.

- [ ] **Step 7: Modelos e tipos de documento**

```ruby
# app/models/signer_certificate.rb
# Certificado ICP-Brasil em nuvem vinculado ao profissional (ADR 0032; spec
# §4). Um `active` por usuário (índice parcial). CPF e certificado cifrados com
# a chave da cidade; o CPF é determinístico para conferir sem decifrar tudo.
class SignerCertificate < ApplicationRecord
  PROVIDERS = %w[vidaas birdid safeid neoid remoteid].freeze
  STATUSES = %w[active replaced unlinked revoked expired].freeze
  EXPIRING_WITHIN = 30.days

  encrypts :subject_cpf, deterministic: true, key_provider: CityDeterministicKeyProvider.new
  encrypts :certificate_der

  belongs_to :user

  scope :active, -> { where(status: "active") }

  validates :provider, inclusion: { in: PROVIDERS }
  validates :status, inclusion: { in: STATUSES }

  def der = Base64.strict_decode64(certificate_der)
  def info = (@info ||= Signatures::CertificateInfo.parse(der))
  def expires_in_days(now = Time.current) = ((not_after - now) / 1.day).floor
  def expiring?(now = Time.current) = not_after <= now + EXPIRING_WITHIN
  def inspect = "#<SignerCertificate id=#{id} provider=#{provider} status=#{status}>"
end
```

```ruby
# app/models/signature_session.rb
# Sessão de assinatura do turno (ADR 0032; spec §5): token signature_session do
# PSC, cifrado com a chave da cidade, até 12 h. Uma ativa por usuário.
class SignatureSession < ApplicationRecord
  STATUSES = %w[active expired revoked].freeze
  SCOPE = "signature_session".freeze
  MAX_LIFETIME = 12.hours

  encrypts :access_token

  belongs_to :user
  belongs_to :signer_certificate

  scope :active, -> { where(status: "active") }

  def self.usable_for(user_id, now: Time.current) = active.where(user_id: user_id).where("expires_at > ?", now).first

  # Houve sessão que venceu nas últimas 24 h (motivo session_expired, e não no_session).
  def self.lapsed?(user_id, now: Time.current)
    where(user_id: user_id, status: %w[active expired]).where(expires_at: (now - 1.day)..now).exists?
  end

  def inspect = "#<SignatureSession id=#{id} status=#{status} expires_at=#{expires_at&.iso8601}>"
end
```

```ruby
# app/models/signature_oauth_state.rb
# State de uso único do OAuth com o PSC (ADR 0032; spec §4, §10): amarrado à
# cidade (pelo token assinado), ao usuário e ao propósito; o code_verifier do
# PKCE cifrado com a chave da cidade.
class SignatureOauthState < ApplicationRecord
  PURPOSES = %w[link session batch].freeze
  TTL = 10.minutes

  encrypts :code_verifier

  belongs_to :user

  def inspect = "#<SignatureOauthState id=#{id} purpose=#{purpose}>"
end
```

```ruby
# app/models/signature_request.rb
# Pedido de assinatura de um documento (a fila; ADR 0032; spec §4). Um por
# documento; resolvido (signed, returned_to_paper) nunca volta (trigger).
class SignatureRequest < ApplicationRecord
  STATUSES = %w[pending signed failed returned_to_paper].freeze
  REASONS = %w[no_session session_expired provider_unavailable provider_rejected signer_unavailable verification_failed
               certificate_expired certificate_revoked certificate_cpf_mismatch feature_disabled user_request].freeze
  TRANSIENT_REASONS = %w[provider_unavailable signer_unavailable verification_failed].freeze
  MIN_RETURN_NOTE = 10
  MAX_RETURN_NOTE = 500

  encrypts :return_note

  belongs_to :author_user, class_name: "User"
  has_one :signature, dependent: :restrict_with_exception

  scope :pending, -> { where(status: "pending") }

  def pending? = status == "pending"
  def document = Signatures::DocumentTypes.find(document_type, document_id)
end
```

```ruby
# app/models/signature.rb
# Assinatura gravada (ADR 0032; spec §4, §7): o JSON canônico assinado (CAdES
# destacado, fonte de verdade), o PDF assinado (PAdES) e as provas de validade
# do ato. Só acréscimo: muda só a validação (trigger signatures_guard).
class Signature < ApplicationRecord
  POLICIES = %w[AD-RB AD-RT].freeze
  VERIFICATIONS = %w[valid invalid indeterminate].freeze

  encrypts :canonical_json, :cades, :signed_pdf, :validation_material
  encrypts :signer_cpf, deterministic: true, key_provider: CityDeterministicKeyProvider.new

  belongs_to :signature_request
  belongs_to :signer_certificate

  def cades_bytes = Base64.strict_decode64(cades)
  def signed_pdf_bytes = Base64.strict_decode64(signed_pdf)
  def material = JSON.parse(validation_material)
  def inspect = "#<Signature id=#{id} document_type=#{document_type} last_verification=#{last_verification}>"
end
```

```ruby
# app/services/signatures/document_types.rb
# Documentos assináveis (ADR 0032; aberto ao 19c). No banco, o nome do modelo;
# na API e nos eventos, o nome do contrato.
module Signatures
  module DocumentTypes
    MODELS = { "Consultation" => "consultation", "ConsultationAddendum" => "consultation_addendum" }.freeze

    module_function

    def api(db_type) = MODELS.fetch(db_type)

    def db(document)
      name = document.class.name
      raise ArgumentError, "documento fora do catálogo: #{name}" unless MODELS.key?(name)

      name
    end

    def find(db_type, id)
      return nil unless MODELS.key?(db_type)

      db_type.constantize.find_by(id: id)
    end

    def consultation_id(document) = document.is_a?(Consultation) ? document.id : document.consultation_id
  end
end
```

- [ ] **Step 8: Cifra, eventos e filtro de log**

Em `app/services/city_encryption.rb`, no fim de `CITY_KEYED_TARGETS`:

```ruby
    # ADR 0032: assinatura digital.
    [ SignerCertificate, :subject_cpf ],
    [ SignerCertificate, :certificate_der ],
    [ SignatureSession, :access_token ],
    [ SignatureOauthState, :code_verifier ],
    [ SignatureRequest, :return_note ],
    [ Signature, :canonical_json ],
    [ Signature, :cades ],
    [ Signature, :signed_pdf ],
    [ Signature, :validation_material ],
    [ Signature, :signer_cpf ]
```

Em `config/initializers/domain_events.rb`, no fim do bloco:

```ruby
  # Módulo 19b (ADR 0032; contrato §11): assinatura digital; trilha, só ids.
  # Nenhum token, CPF, nome ou conteúdo assinado entra em evento.
  DomainEvents.bind "signature.certificate_linked", to: []
  DomainEvents.bind "signature.certificate_unlinked", to: []
  DomainEvents.bind "signature.session_opened", to: []
  DomainEvents.bind "signature.signed", to: []
  DomainEvents.bind "signature.failed", to: []
  DomainEvents.bind "signature.returned_to_paper", to: []
  DomainEvents.bind "signature.verified", to: []
```

Em `spec/initializers/domain_events_bindings_spec.rb`:

```ruby
# Módulo 19b (ADR 0032): assinatura digital, só trilha.
RSpec.describe "digital signature event bindings (ADR 0032)" do
  it "declares every module 19b event with no consumer" do
    names = %w[signature.certificate_linked signature.certificate_unlinked signature.session_opened signature.signed
               signature.failed signature.returned_to_paper signature.verified]
    expect(DomainEvents.registry.keys).to include(*names)
    expect(names.flat_map { |n| DomainEvents.registry[n] }).to be_empty
  end
end
```

Em `config/initializers/filter_parameter_logging.rb`, depois da última entrada da lista (com vírgula nela):

```ruby
  # ADR 0032: PKCE, estado opaco do signer, valor da assinatura e o JSON
  # canônico (`code`, `state`, `token` e `cpf` já estavam na lista).
  :verifier, :prepared, :signature_value, :raw_signature, :canonical
```

e, no fim do arquivo:

```ruby
# ADR 0032: o retorno do PSC traz code e state na URL do dashboard.
Rails.application.config.filter_redirect += [ /code=/ ]
```

- [ ] **Step 9: Migre os bancos de teste e rode**

Rode o `DROP DATABASE` + `city:test_databases` do Ambiente de execução e depois:

Run: `docker compose exec -T -w /rails/.claude/mod19b api bundle exec rspec spec/models/signature_tables_guard_spec.rb spec/services/city_schema_spec.rb spec/architecture/city_encrypted_attributes_guard_spec.rb spec/initializers/domain_events_bindings_spec.rb`
Expected: PASS.

- [ ] **Step 10: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod19b add db/city_migrate/20261008500001_add_digital_signatures.rb db/city_schema.rb db/city_triggers.sql app/models/signer_certificate.rb app/models/signature_session.rb app/models/signature_oauth_state.rb app/models/signature_request.rb app/models/signature.rb app/services/signatures/document_types.rb app/services/city_encryption.rb config/initializers/domain_events.rb config/initializers/filter_parameter_logging.rb spec/initializers/domain_events_bindings_spec.rb spec/support/signature_helpers.rb spec/rails_helper.rb spec/models/signature_tables_guard_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod19b commit -m "feat: add the digital signature tables with immutability and city-key encryption

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
### Task 4: Prestadores e cliente da API de PSC do ITI (com o PSC falso)

**PSC falso: servidor Rack, servido por WebMock nos testes.** Escolha: um `FakePsc::App` (Rack puro, `lib/fake_psc/app.rb`) que implementa a API v0 do ITI de verdade — PKCE S256 conferido, token de uso único para `single_signature`/`multi_signature`, assinatura RAW com a chave do e-CPF de teste (AC do `signer` de dev, Task 2). Nos testes ele entra por `stub_request(...).to_rack(app)` (WebMock), e no dev roda como serviço `fake-psc` do compose (Task 18). Por que não só WebMock com respostas fixas: a assinatura RAW precisa ser **verdadeira** (o `signer` real confere contra o certificado), o PKCE precisa ser conferido para o teste valer, e o mesmo falso serve o fluxo no navegador. Por que não um servidor separado nos testes: WebMock `to_rack` exercita o `Net::HTTP` do cliente sem rede, sem porta e sem processo, e deixa injetar falhas (`503`, conexão recusada, demora) por exemplo. Respostas literais do DOC-ICP-17.01 continuam numa spec própria (a do cliente), para o cliente não ficar testado só contra o nosso falso.

**Files:**
- Create: `db/platform_migrate/20261008500001_create_signature_provider_checks.rb`, `app/models/signature_provider_check.rb`, `app/services/signatures/providers.rb`, `app/services/signatures/psc.rb`, `app/services/signatures/psc/client.rb`, `lib/fake_psc/app.rb`
- Modify: `db/platform_schema.rb`, `spec/support/signature_helpers.rb`
- Test: `spec/services/signatures/psc_client_spec.rb`, `spec/services/signatures/providers_spec.rb`

**Interfaces:**
- Consumes: `FakePsc::Pki` (Task 2), `SignerCertificate::PROVIDERS` (Task 3), `CityPublicUrl.dashboard(city)`, `Rota.deployed?` (existentes).
- Produces:
  - `SignatureProviderCheck` (plataforma; `provider` único, `last_check_at`, `last_check_ok`), `.record!(provider, ok:, at: Time.current)` (nunca levanta).
  - `Signatures::Providers::CATALOG`, `::LABELS`, `::Provider = Data(:key, :client_id, :client_secret, :base_url, :authorize_base_url)`; `.credentials -> Hash{String => Hash}` (credenciais cifradas `signature.providers.<key>`; em development, sobre `config.x.signature_dev_providers`), `.find(key) -> Provider|nil` (nil sem as três chaves), `.configured -> Array<Provider>`, `.configured?(key)`, `.redirect_uri(city) -> String` (`https://<host do dashboard da cidade>/dashboard/signature/callback`, de `CityPublicUrl.dashboard(city)`; um por cidade, Desvio 9), `.record_check!(key, ok:)`, `.checks -> Hash{key => SignatureProviderCheck}`.
  - `Signatures::Psc::{Error,Unavailable,Rejected(code),Unauthorized(code)}`, `::Token = Data(:access_token, :expires_in, :scope)` (`inspect` sem o token), `::CertificateEntry = Data(:certificate_alias, :der)`, `::SCOPES`.
  - `Signatures::Psc::Client.for(key) -> Client` (levanta `Unavailable` se o PSC não está configurado), `::PATHS`, `::SHA256_OID`; `#discover(cpf) -> bool`, `#authorize_url(state:, challenge:, scope:, login_hint:, redirect_uri:, lifetime: nil) -> String`, `#exchange(code:, verifier:, redirect_uri:) -> Token`, `#certificates(access_token) -> Array<CertificateEntry>`, `#sign(access_token:, certificate_alias:, digests: Hash{String => 32 bytes}) -> Hash{String => bytes}`. Toda chamada registra `Providers.record_check!` (ok = não foi `Unavailable`).
  - `FakePsc::App.new(pki:)` (Rack; `CLIENT_ID`, `CLIENT_SECRET`); ganchos: `#decide!(state, approve: true) -> url de retorno`, `#approve!(state) -> code`, `#leaf(cpf)`, `#token_for!(cpf:, scope: "signature_session", ttl: 3600) -> access_token`, `#expire_tokens!`, `#issued_tokens -> Array<String>`, `#reset!`, acessores `absent_cpfs`, `failures` (fila de `Integer` status ou `:refused`), `max_lifetime`, `forced_lifetime`, `before_sign` (proc), `certificate_overrides` (Hash{cpf => Leaf}), `log`.
  - Helpers: `SignatureHelpers::PSC_BASES`, `fake_psc(key = "vidaas") -> FakePsc::App`, `stub_psc!(keys = %w[vidaas])`, `authorize_and_approve!(url) -> code` (GET da URL no falso + aprovação pelo `state` da URL).

- [ ] **Step 1: A tabela de plataforma**

```ruby
# db/platform_migrate/20261008500001_create_signature_provider_checks.rb
# Última conversa da plataforma com cada PSC (ADR 0032; contrato §8): o
# maintenance mostra "credencial presente" e "última checagem". Só a chave do
# catálogo, o instante e se deu certo — nunca segredo, URL, CPF ou resposta.
class CreateSignatureProviderChecks < ActiveRecord::Migration[8.1]
  def change
    create_table :signature_provider_checks, id: :uuid, default: -> { "gen_random_uuid()" } do |t|
      t.string :provider, null: false
      t.datetime :last_check_at, null: false
      t.boolean :last_check_ok, null: false
      t.timestamps
      t.index :provider, unique: true
      t.check_constraint "provider::text = ANY (ARRAY['vidaas'::text, 'birdid'::text, 'safeid'::text, 'neoid'::text, 'remoteid'::text])",
                         name: "ck_signature_provider_checks_provider"
    end
  end
end
```

Em `db/platform_schema.rb`, troque o `define(version:)` por `2026_10_08_500001` e acrescente (ordem alfabética):

```ruby
  create_table "signature_provider_checks", id: :uuid, default: -> { "gen_random_uuid()" }, force: :cascade do |t|
    t.datetime "created_at", null: false
    t.datetime "last_check_at", null: false
    t.boolean "last_check_ok", null: false
    t.string "provider", null: false
    t.datetime "updated_at", null: false
    t.index ["provider"], name: "index_signature_provider_checks_on_provider", unique: true
    t.check_constraint "provider::text = ANY (ARRAY['vidaas'::text, 'birdid'::text, 'safeid'::text, 'neoid'::text, 'remoteid'::text])", name: "ck_signature_provider_checks_provider"
  end
```

Run: `docker compose exec -T -e RAILS_ENV=test -w /rails/.claude/mod19b api bin/rails db:migrate:platform`
Expected: a tabela nasce no banco de plataforma de teste (o de dev só na Task 19, com autorização).

```ruby
# app/models/signature_provider_check.rb
# Última checagem de cada PSC (plataforma; ADR 0032). Escrita por toda chamada
# ao PSC; uma falha aqui nunca derruba a assinatura.
class SignatureProviderCheck < PlatformRecord
  def self.record!(provider, ok:, at: Time.current)
    upsert({ provider: provider.to_s, last_check_at: at, last_check_ok: ok }, unique_by: :provider)
  rescue ActiveRecord::ActiveRecordError => e
    Rails.logger.warn("[signature_provider_check] #{e.class}")
    nil
  end
end
```

- [ ] **Step 2: O PSC falso**

```ruby
# lib/fake_psc/app.rb
# PSC falso da API v0 do ITI (DOC-ICP-17.01): user-discovery, authorize, token
# (PKCE S256), certificate-discovery e signature (RAW), com e-CPF de teste da
# AC do signer de dev (FakePsc::Pki; as LCRs quem serve é o signer). Usado pelas specs (WebMock#to_rack) e pelo serviço fake-psc do compose
# de dev (lib/fake_psc/config.ru). Nunca carregado em produção. Sem
# ActiveSupport: roda sozinho no puma.
require "json"
require "openssl"
require "securerandom"
require "base64"
require "digest"
require "uri"
require "rack"
require_relative "pki"

module FakePsc
  class App
    CLIENT_ID = "rota-dev".freeze
    CLIENT_SECRET = "dev-secret".freeze
    SHA256_OID = "2.16.840.1.101.3.4.2.1".freeze
    SCOPES = %w[single_signature multi_signature signature_session authentication_session].freeze
    ONE_SHOT_TTL = 300

    attr_reader :pki, :log
    attr_accessor :absent_cpfs, :failures, :max_lifetime, :forced_lifetime, :before_sign, :certificate_overrides

    def initialize(pki:, client_id: CLIENT_ID, client_secret: CLIENT_SECRET)
      @pki = pki
      @client_id = client_id
      @client_secret = client_secret
      @mutex = Mutex.new
      reset!
    end

    def reset!
      @mutex.synchronize do
        @pending = {}
        @codes = {}
        @tokens = {}
        @log = []
      end
      @absent_cpfs = []
      @failures = []
      @certificate_overrides = {}
      @max_lifetime = 7 * 86_400
      @forced_lifetime = nil
      @before_sign = nil
    end

    # — Ganchos de teste (o "celular" do titular e o estado do PSC) —

    def decide!(state, approve: true)
      id = @mutex.synchronize { @pending.find { |_id, pending| pending[:state] == state }&.first }
      raise ArgumentError, "autorização desconhecida" unless id

      decision(id, approve)
    end

    def approve!(state)
      URI.decode_www_form(URI(decide!(state)).query).to_h.fetch("code")
    end

    def leaf(cpf) = @certificate_overrides[cpf] || @pki.leaf_for(cpf)
    def token_for!(cpf:, scope: "signature_session", ttl: 3600) = issue_token(cpf, scope, ttl)
    def expire_tokens! = @mutex.synchronize { @tokens.each_value { |token| token[:expires_at] = Time.now - 1 } }
    def issued_tokens = @mutex.synchronize { @tokens.keys }

    def call(env)
      request = Rack::Request.new(env)
      failure = @mutex.synchronize do
        @log << [ request.request_method, request.path ]
        @failures.shift
      end
      raise Errno::ECONNREFUSED, "fake-psc" if failure == :refused
      return json(failure, { error: "server_error" }) if failure.is_a?(Integer)

      route(request)
    rescue ArgumentError
      json(400, { error: "invalid_request" })
    end

    def inspect = "#<FakePsc::App>"

    private

    def route(request)
      case [ request.request_method, request.path ]
      in [ "POST", "/v0/oauth/user-discovery" ] then user_discovery(request)
      in [ "GET", "/v0/oauth/authorize" ] then authorize(request)
      in [ "GET", "/v0/oauth/authorize/decision" ]
        [ 302, { "location" => decision(request.params["id"].to_s, request.params["approve"] == "1") }, [] ]
      in [ "POST", "/v0/oauth/token" ] then token(request)
      in [ "GET", "/v0/oauth/certificate-discovery" ] then certificates(request)
      in [ "POST", "/v0/oauth/signature" ] then signature(request)
      else json(404, { error: "not_found" })
      end
    end

    def user_discovery(request)
      body = parse_json(request)
      return json(401, { error: "invalid_client" }) unless client?(body["client_id"], body["client_secret"])

      cpf = body["val_cpf_cnpj"].to_s
      found = body["user_cpf_cnpj"] == "CPF" && cpf.match?(/\A\d{11}\z/) && !@absent_cpfs.include?(cpf)
      json(200, found ? { status: "S", slots: [ { slot_alias: cpf, label: "Certificado de teste" } ] } : { status: "N", slots: [] })
    end

    def authorize(request)
      params = request.params
      valid = params["response_type"] == "code" && params["client_id"] == @client_id &&
              params["code_challenge_method"] == "S256" && SCOPES.include?(params["scope"]) &&
              params["code_challenge"].to_s.size >= 43 && params["redirect_uri"].to_s.start_with?("http") &&
              params["login_hint"].to_s.match?(/\A\d{11}\z/)
      return json(400, { error: "invalid_request" }) unless valid

      id = SecureRandom.hex(8)
      @mutex.synchronize do
        @pending[id] = { state: params["state"], redirect_uri: params["redirect_uri"], scope: params["scope"],
                         challenge: params["code_challenge"], cpf: params["login_hint"],
                         lifetime: params["lifetime"]&.to_i }
      end
      [ 200, { "content-type" => "text/html; charset=utf-8" }, [ page(id, params["scope"]) ] ]
    end

    def page(id, scope)
      <<~HTML
        <!doctype html><html lang="pt-BR"><head><meta charset="utf-8"><title>PSC SIMULADO</title></head>
        <body style="font-family:sans-serif;max-width:32rem;margin:3rem auto">
        <h1>PSC SIMULADO — desenvolvimento</h1>
        <p>Pedido de autorização: <strong>#{Rack::Utils.escape_html(scope)}</strong>. Nenhum certificado real é usado.</p>
        <p><a href="/v0/oauth/authorize/decision?id=#{id}&amp;approve=1">Aprovar</a> ·
           <a href="/v0/oauth/authorize/decision?id=#{id}&amp;approve=0">Recusar</a></p>
        </body></html>
      HTML
    end

    def decision(id, approve)
      pending = @mutex.synchronize { @pending.delete(id) }
      raise ArgumentError, "autorização desconhecida" unless pending

      query = if approve
                code = SecureRandom.urlsafe_base64(24)
                @mutex.synchronize { @codes[code] = pending }
                { code: code, state: pending[:state] }
              else
                { error: "access_denied", state: pending[:state] }
              end
      separator = pending[:redirect_uri].include?("?") ? "&" : "?"
      "#{pending[:redirect_uri]}#{separator}#{URI.encode_www_form(query)}"
    end

    def token(request)
      params = request.POST
      return json(401, { error: "invalid_client" }) unless client?(params["client_id"], params["client_secret"])
      return json(400, { error: "unsupported_grant_type" }) unless params["grant_type"] == "authorization_code"

      grant = @mutex.synchronize { @codes.delete(params["code"].to_s) }
      return json(400, { error: "invalid_grant" }) unless grant && grant[:redirect_uri] == params["redirect_uri"]

      challenge = Base64.urlsafe_encode64(Digest::SHA256.digest(params["code_verifier"].to_s), padding: false)
      return json(400, { error: "invalid_grant" }) unless Rack::Utils.secure_compare(challenge, grant[:challenge])

      ttl = if grant[:scope] == "signature_session"
              @forced_lifetime || [ grant[:lifetime] || @max_lifetime, @max_lifetime ].min
            else
              ONE_SHOT_TTL
            end
      access = issue_token(grant[:cpf], grant[:scope], ttl)
      json(200, { access_token: access, token_type: "Bearer", expires_in: ttl, scope: grant[:scope],
                  authorized_identification_type: "CPF", authorized_identification: grant[:cpf] })
    end

    def certificates(request)
      token = bearer(request)
      return json(401, { error: "invalid_token" }) unless token

      json(200, { status: "S",
                  certificates: [ { alias: token[:cpf], certificate: Base64.strict_encode64(leaf(token[:cpf]).der) } ] })
    end

    def signature(request)
      token = bearer(request)
      return json(401, { error: "invalid_token" }) unless token

      body = parse_json(request)
      hashes = Array(body["hashes"])
      return json(400, { error: "invalid_request" }) if hashes.empty? || body["certificate_alias"] != token[:cpf]
      return json(400, { error: "invalid_request" }) if token[:scope] == "single_signature" && hashes.size != 1
      return json(401, { error: "invalid_token" }) if %w[single_signature multi_signature].include?(token[:scope]) && token[:uses].positive?

      @before_sign&.call
      key = leaf(token[:cpf]).key
      signatures = hashes.map do |item|
        digest = Base64.strict_decode64(item["hash"].to_s)
        unless item["hash_algorithm"] == SHA256_OID && item["signature_format"] == "RAW" && digest.bytesize == 32
          return json(400, { error: "invalid_request" })
        end

        { id: item["id"], raw_signature: Base64.strict_encode64(key.sign_raw("SHA256", digest)) }
      end
      @mutex.synchronize { token[:uses] += 1 }
      json(200, { certificate_alias: token[:cpf], signatures: signatures })
    end

    def bearer(request)
      value = request.get_header("HTTP_AUTHORIZATION").to_s[/\ABearer (.+)\z/, 1]
      token = value && @mutex.synchronize { @tokens[value] }
      token if token && token[:expires_at] > Time.now
    end

    def issue_token(cpf, scope, ttl)
      access = SecureRandom.urlsafe_base64(32)
      @mutex.synchronize { @tokens[access] = { cpf: cpf, scope: scope, expires_at: Time.now + ttl, uses: 0 } }
      access
    end

    def client?(id, secret) = id == @client_id && secret == @client_secret

    def parse_json(request)
      text = request.body.read.to_s
      parsed = JSON.parse(text.empty? ? "{}" : text)
      parsed.is_a?(Hash) ? parsed : {}
    rescue JSON::ParserError
      {}
    end

    def json(status, object) = [ status, { "content-type" => "application/json" }, [ JSON.generate(object) ] ]
  end
end
```

- [ ] **Step 3: Os helpers do PSC (acrescente ao módulo `SignatureHelpers`)**

No topo de `spec/support/signature_helpers.rb`, depois do `require_relative` da Pki:

```ruby
require_relative "../../lib/fake_psc/app"
require "webmock/rspec"
```

Dentro do módulo:

```ruby
  PSC_BASES = { "vidaas" => "https://psc-vidaas.test", "birdid" => "https://psc-birdid.test" }.freeze

  def fake_psc(key = "vidaas") = (@fake_pscs ||= {})[key] ||= FakePsc::App.new(pki: test_pki)

  # Credenciais da plataforma só destes PSC, cada um servido pelo seu falso.
  def stub_psc!(keys = %w[vidaas])
    credentials = keys.to_h do |key|
      [ key, { "client_id" => FakePsc::App::CLIENT_ID, "client_secret" => FakePsc::App::CLIENT_SECRET,
               "base_url" => PSC_BASES.fetch(key) } ]
    end
    allow(Signatures::Providers).to receive(:credentials).and_return(credentials)
    keys.each { |key| stub_request(:any, /\A#{Regexp.escape(PSC_BASES.fetch(key))}/).to_rack(fake_psc(key)) }
  end

  # O navegador abre a URL de autorização (o falso registra o pedido) e o
  # titular aprova no "celular". Devolve o code.
  def authorize_and_approve!(url, key: "vidaas")
    Net::HTTP.get_response(URI(url))
    state = URI.decode_www_form(URI(url).query).to_h.fetch("state")
    fake_psc(key).approve!(state)
  end
```

- [ ] **Step 4: Escreva as specs que falham**

```ruby
# spec/services/signatures/providers_spec.rb
require "rails_helper"

# ADR 0032 (spec §3): credenciais da PLATAFORMA por PSC; sem as três chaves o
# PSC não está habilitado. Nada de segredo em inspect.
RSpec.describe Signatures::Providers do
  it "só os PSC do catálogo com client_id, client_secret e base_url" do
    allow(described_class).to receive(:credentials).and_return(
      "vidaas" => { "client_id" => "a", "client_secret" => "SEGREDO-X", "base_url" => "https://v.test/" },
      "birdid" => { "client_id" => "b", "base_url" => "https://b.test" },
      "outro" => { "client_id" => "c", "client_secret" => "d", "base_url" => "https://o.test" }
    )
    expect(described_class.configured.map(&:key)).to eq([ "vidaas" ])
    expect(described_class.configured?("birdid")).to be(false)
    expect(described_class.find("outro")).to be_nil
    expect(described_class.find("vidaas").base_url).to eq("https://v.test")
    expect(described_class.find("vidaas").inspect).not_to include("SEGREDO-X")
  end

  it "redirect_uri: uma por cidade, a rota de retorno do dashboard dela" do
    city = clinical_city!
    expect(described_class.redirect_uri(city)).to eq("#{CityPublicUrl.dashboard(city)}signature/callback")
    expect(described_class.redirect_uri(city)).to end_with("/dashboard/signature/callback")
    expect(described_class.redirect_uri(city)).to start_with(CityPublicUrl.base(city))
  end

  it "a checagem fica na plataforma e nunca levanta" do
    described_class.record_check!("vidaas", ok: false)
    described_class.record_check!("vidaas", ok: true)
    expect(described_class.checks["vidaas"]).to have_attributes(last_check_ok: true)
    allow(SignatureProviderCheck).to receive(:upsert).and_raise(ActiveRecord::ConnectionNotEstablished)
    expect { described_class.record_check!("vidaas", ok: true) }.not_to raise_error
  end
end
```

```ruby
# spec/services/signatures/psc_client_spec.rb
require "rails_helper"

# Contrato da API v0 do ITI (DOC-ICP-17.01; pesquisa §2; Task 0 Step 3):
# localização por CPF, autorização com PKCE S256, token, recuperação do
# certificado e assinatura RAW de hash. O falso confere PKCE, uso único e
# assina de verdade; a última spec usa respostas literais do documento.
RSpec.describe Signatures::Psc::Client do
  before { stub_psc! }

  let(:client) { described_class.for("vidaas") }
  let(:fake) { fake_psc }
  let(:cpf) { SignatureHelpers::DOCTOR_CPF }
  let(:redirect_uri) { "https://auth.rotasaude.test/signature/psc/callback" }
  let(:verifier) { SecureRandom.urlsafe_base64(48) }
  let(:challenge) { Base64.urlsafe_encode64(Digest::SHA256.digest(verifier), padding: false) }

  def url(scope, state: "st-#{SecureRandom.hex(4)}", lifetime: nil)
    client.authorize_url(state: state, challenge: challenge, scope: scope, login_hint: cpf, redirect_uri: redirect_uri,
                         lifetime: lifetime)
  end

  def token(scope, lifetime: nil)
    code = authorize_and_approve!(url(scope, lifetime: lifetime))
    client.exchange(code: code, verifier: verifier, redirect_uri: redirect_uri)
  end

  it "localiza o titular por CPF e registra a checagem" do
    expect(client.discover(cpf)).to be(true)
    fake.absent_cpfs << cpf
    expect(client.discover(cpf)).to be(false)
    expect(SignatureProviderCheck.find_by(provider: "vidaas")).to have_attributes(last_check_ok: true)
  end

  it "monta a URL de autorização com PKCE S256, CPF e tempo de vida" do
    query = URI.decode_www_form(URI(url("signature_session", state: "abc", lifetime: 43_200)).query).to_h
    expect(query).to include("response_type" => "code", "client_id" => FakePsc::App::CLIENT_ID, "state" => "abc",
                             "scope" => "signature_session", "code_challenge" => challenge,
                             "code_challenge_method" => "S256", "login_hint" => cpf, "lifetime" => "43200",
                             "redirect_uri" => redirect_uri)
    expect { url("qualquer") }.to raise_error(ArgumentError)
  end

  it "troca o código, lê o certificado e assina RAW o hash" do
    issued = token("signature_session", lifetime: 43_200)
    expect(issued.expires_in).to eq(43_200)
    expect(issued.inspect).not_to include(issued.access_token)
    entry = client.certificates(issued.access_token).sole
    expect(Signatures::CertificateInfo.parse(entry.der).cpf).to eq(cpf)
    digests = { "d1" => Digest::SHA256.digest("um"), "d2" => Digest::SHA256.digest("dois") }
    raw = client.sign(access_token: issued.access_token, certificate_alias: entry.certificate_alias, digests: digests)
    public_key = fake.leaf(cpf).certificate.public_key
    expect(digests.all? { |id, digest| public_key.verify_raw("SHA256", raw.fetch(id), digest) }).to be(true)
    expect(client.sign(access_token: issued.access_token, certificate_alias: cpf, digests: { "d3" => digests["d1"] }).keys)
      .to eq([ "d3" ]) # signature_session: várias chamadas
  end

  it "PKCE errado e código reusado são recusados (invalid_grant)" do
    code = authorize_and_approve!(url("single_signature"))
    expect { client.exchange(code: code, verifier: "outro-#{verifier}", redirect_uri: redirect_uri) }
      .to raise_error(Signatures::Psc::Rejected) { |e| expect(e.code).to eq("invalid_grant") }
    expect { client.exchange(code: code, verifier: verifier, redirect_uri: redirect_uri) }
      .to raise_error(Signatures::Psc::Rejected)
  end

  it "single_signature vale uma assinatura; multi_signature, uma chamada com vários hashes" do
    single = token("single_signature")
    client.sign(access_token: single.access_token, certificate_alias: cpf, digests: { "a" => Digest::SHA256.digest("a") })
    expect { client.sign(access_token: single.access_token, certificate_alias: cpf, digests: { "b" => Digest::SHA256.digest("b") }) }
      .to raise_error(Signatures::Psc::Unauthorized)
    multi = token("multi_signature")
    many = (1..100).to_h { |i| [ "h#{i}", Digest::SHA256.digest(i.to_s) ] }
    expect(client.sign(access_token: multi.access_token, certificate_alias: cpf, digests: many).size).to eq(100)
  end

  it "503, conexão recusada e token vencido: Unavailable/Unauthorized sem segredo na mensagem" do
    fake.failures.push(503, :refused)
    log = capture_log do
      expect { client.discover(cpf) }.to raise_error(Signatures::Psc::Unavailable) { |e| expect(e.message).not_to include(FakePsc::App::CLIENT_SECRET) }
      expect { client.discover(cpf) }.to raise_error(Signatures::Psc::Unavailable)
    end
    expect(SignatureProviderCheck.find_by(provider: "vidaas")).to have_attributes(last_check_ok: false)
    issued = token("signature_session")
    fake.expire_tokens!
    expect { client.certificates(issued.access_token) }.to raise_error(Signatures::Psc::Unauthorized)
    expect(log).not_to include(FakePsc::App::CLIENT_SECRET, cpf)
  end

  it "lê as respostas no formato literal do DOC-ICP-17.01 (sem o falso)" do
    WebMock.reset!
    base = SignatureHelpers::PSC_BASES["vidaas"]
    der = test_pki.leaf_for(cpf).der
    stub_request(:post, "#{base}/v0/oauth/token")
      .to_return(status: 200, headers: { "Content-Type" => "application/json" },
                 body: { access_token: "eyJ0eXAi", token_type: "Bearer", expires_in: 43_200, scope: "signature_session",
                         authorized_identification_type: "CPF", authorized_identification: cpf }.to_json)
    stub_request(:get, "#{base}/v0/oauth/certificate-discovery")
      .with(headers: { "Authorization" => "Bearer eyJ0eXAi" })
      .to_return(status: 200, body: { status: "S", certificates: [ { alias: "slot-1", certificate: Base64.strict_encode64(der) } ] }.to_json)
    stub_request(:post, "#{base}/v0/oauth/signature")
      .with(body: hash_including("certificate_alias" => "slot-1",
                                 "hashes" => [ hash_including("id" => "x", "hash_algorithm" => "2.16.840.1.101.3.4.2.1", "signature_format" => "RAW") ]))
      .to_return(status: 200, body: { certificate_alias: "slot-1", signatures: [ { id: "x", raw_signature: Base64.strict_encode64("RAW") } ] }.to_json)

    issued = client.exchange(code: "c", verifier: verifier, redirect_uri: redirect_uri)
    expect(issued).to have_attributes(access_token: "eyJ0eXAi", expires_in: 43_200, scope: "signature_session")
    expect(client.certificates("eyJ0eXAi").sole).to have_attributes(certificate_alias: "slot-1", der: der)
    expect(client.sign(access_token: "eyJ0eXAi", certificate_alias: "slot-1", digests: { "x" => Digest::SHA256.digest("x") }))
      .to eq("x" => "RAW")
  end
end
```

- [ ] **Step 5: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod19b api bundle exec rspec spec/services/signatures/providers_spec.rb spec/services/signatures/psc_client_spec.rb`
Expected: FAIL (`uninitialized constant Signatures::Providers`).

- [ ] **Step 6: Implemente**

```ruby
# app/services/signatures/providers.rb
# PSC de nuvem da ICP-Brasil (ADR 0032; spec §3). Credenciais da PLATAFORMA,
# por ambiente, nas credenciais cifradas do Rails:
#   signature.providers.<key>.{client_id, client_secret, base_url[, authorize_base_url]}
# authorize_base_url só existe no dev (o navegador fala com o PSC falso por um
# endereço, o api/worker por outro). Sem as três chaves o PSC não está
# habilitado. Nenhum segredo em inspect, log ou erro.
module Signatures
  module Providers
    CATALOG = SignerCertificate::PROVIDERS
    LABELS = { "vidaas" => "VIDaaS", "birdid" => "BirdID", "safeid" => "SafeID", "neoid" => "NeoID",
               "remoteid" => "RemoteID" }.freeze
    REQUIRED = %w[client_id client_secret base_url].freeze

    Provider = Data.define(:key, :client_id, :client_secret, :base_url, :authorize_base_url) do
      def inspect = "#<Signatures::Providers::Provider #{key}>"
      alias_method :to_s, :inspect
    end

    module_function

    def credentials
      dev = Rails.configuration.x.signature_dev_providers.to_h.deep_stringify_keys
      real = Rails.application.credentials.dig(:signature, :providers).to_h.deep_stringify_keys
      dev.merge(real)
    end

    def find(key)
      key = key.to_s
      return nil unless CATALOG.include?(key)

      raw = credentials[key]
      return nil unless raw.is_a?(Hash) && REQUIRED.all? { |name| raw[name].present? }

      Provider.new(key: key, client_id: raw["client_id"].to_s, client_secret: raw["client_secret"].to_s,
                   base_url: raw["base_url"].to_s.chomp("/"), authorize_base_url: raw["authorize_base_url"].presence&.chomp("/"))
    end

    def configured = CATALOG.filter_map { |key| find(key) }
    def configured?(key) = !find(key).nil?

    # Desvio 9: um retorno por cidade, montado do host público do dashboard dela
    # (https://<host da cidade>/dashboard/signature/callback). Cada endereço é
    # cadastrado em cada PSC no go-live da cidade (Task 19).
    def redirect_uri(city) = "#{CityPublicUrl.dashboard(city)}signature/callback"

    def record_check!(key, ok:) = SignatureProviderCheck.record!(key, ok: ok)
    def checks = SignatureProviderCheck.where(provider: CATALOG).index_by(&:provider)
  end
end
```

```ruby
# app/services/signatures/psc.rb
# API de PSC do ITI (DOC-ICP-17.01; ADR 0032). Nenhuma mensagem de erro
# carrega token, code, verifier, CPF, segredo ou corpo de resposta.
module Signatures
  module Psc
    class Error < StandardError; end
    # Rede, timeout, 5xx: tentar de novo depois.
    class Unavailable < Error; end

    # 4xx: o PSC recusou (o código é o "error" da resposta, ou http_<status>).
    class Rejected < Error
      attr_reader :code

      def initialize(code)
        @code = code.to_s
        super("o PSC recusou: #{@code}")
      end
    end

    # 401/403: token vencido, revogado ou credencial da plataforma recusada.
    class Unauthorized < Rejected; end

    SCOPES = %w[single_signature multi_signature signature_session].freeze

    Token = Data.define(:access_token, :expires_in, :scope) do
      def inspect = "#<Signatures::Psc::Token scope=#{scope} expires_in=#{expires_in}>"
      alias_method :to_s, :inspect
    end

    CertificateEntry = Data.define(:certificate_alias, :der) do
      def inspect = "#<Signatures::Psc::CertificateEntry>"
    end
  end
end
```

```ruby
# app/services/signatures/psc/client.rb
# Cliente único da API v0 do ITI para todos os PSC (pesquisa §2; Task 0 Step 3
# conferiu os nomes). Nomes de caminho e de campo moram SÓ aqui e no
# FakePsc::App. Toda chamada registra a checagem do PSC na plataforma.
require "net/http"

module Signatures
  module Psc
    class Client
      TIMEOUT = 15
      SHA256_OID = "2.16.840.1.101.3.4.2.1".freeze
      PATHS = { discovery: "/v0/oauth/user-discovery", authorize: "/v0/oauth/authorize", token: "/v0/oauth/token",
                certificates: "/v0/oauth/certificate-discovery", signature: "/v0/oauth/signature" }.freeze
      NETWORK_ERRORS = [ SocketError, IOError, EOFError, SystemCallError, Net::OpenTimeout, Net::ReadTimeout,
                         Net::WriteTimeout, Net::ProtocolError, OpenSSL::SSL::SSLError ].freeze

      def self.for(key)
        provider = Providers.find(key)
        raise Unavailable, "PSC não configurado" unless provider

        new(provider)
      end

      def initialize(provider, timeout: TIMEOUT)
        @provider = provider
        @timeout = timeout
      end

      def discover(cpf)
        body = call(:discovery, json: { client_id: @provider.client_id, client_secret: @provider.client_secret,
                                        user_cpf_cnpj: "CPF", val_cpf_cnpj: cpf })
        body["status"] == "S" && Array(body["slots"]).any?
      end

      def authorize_url(state:, challenge:, scope:, login_hint:, redirect_uri:, lifetime: nil)
        raise ArgumentError, "escopo desconhecido" unless SCOPES.include?(scope)

        query = { response_type: "code", client_id: @provider.client_id, redirect_uri: redirect_uri, state: state,
                  scope: scope, code_challenge: challenge, code_challenge_method: "S256", login_hint: login_hint,
                  lifetime: lifetime }.compact
        "#{@provider.authorize_base_url || @provider.base_url}#{PATHS[:authorize]}?#{URI.encode_www_form(query)}"
      end

      def exchange(code:, verifier:, redirect_uri:)
        body = call(:token, form: { grant_type: "authorization_code", client_id: @provider.client_id,
                                    client_secret: @provider.client_secret, code: code, redirect_uri: redirect_uri,
                                    code_verifier: verifier })
        access = body["access_token"]
        raise Rejected, "invalid_token_response" unless access.is_a?(String) && access.present?

        Token.new(access_token: access, expires_in: Integer(body["expires_in"]), scope: body["scope"].to_s)
      rescue ArgumentError, TypeError
        raise Rejected, "invalid_token_response"
      end

      def certificates(access_token)
        body = call(:certificates, bearer: access_token)
        Array(body["certificates"]).filter_map do |entry|
          der = decode_certificate(entry["certificate"])
          der && CertificateEntry.new(certificate_alias: entry["alias"].to_s, der: der)
        end
      end

      def sign(access_token:, certificate_alias:, digests:)
        hashes = digests.map do |id, digest|
          { id: id.to_s, alias: id.to_s, hash: Base64.strict_encode64(digest), hash_algorithm: SHA256_OID,
            signature_format: "RAW" }
        end
        body = call(:signature, json: { certificate_alias: certificate_alias, hashes: hashes }, bearer: access_token)
        result = Array(body["signatures"]).to_h { |item| [ item["id"].to_s, Base64.strict_decode64(item["raw_signature"].to_s) ] }
        raise Rejected, "incomplete_signatures" unless result.keys.sort == digests.keys.map(&:to_s).sort

        result
      rescue ArgumentError
        raise Rejected, "invalid_signature_response"
      end

      def inspect = "#<Signatures::Psc::Client #{@provider.key}>"

      private

      def call(name, json: nil, form: nil, bearer: nil)
        uri = URI("#{@provider.base_url}#{PATHS.fetch(name)}")
        request = json || form ? Net::HTTP::Post.new(uri) : Net::HTTP::Get.new(uri)
        request["Accept"] = "application/json"
        request["Authorization"] = "Bearer #{bearer}" if bearer
        if json
          request["Content-Type"] = "application/json"
          request.body = JSON.generate(json)
        elsif form
          request.set_form_data(form)
        end
        response = Net::HTTP.start(uri.host, uri.port, use_ssl: uri.scheme == "https", open_timeout: @timeout,
                                                       read_timeout: @timeout, write_timeout: @timeout) { |http| http.request(request) }
        interpret(response)
      rescue *NETWORK_ERRORS => e
        Providers.record_check!(@provider.key, ok: false)
        raise Unavailable, "PSC inalcançável (#{e.class.name})"
      end

      def interpret(response)
        code = response.code.to_i
        body = parse(response.body)
        if code >= 500
          Providers.record_check!(@provider.key, ok: false)
          raise Unavailable, "o PSC respondeu #{code}"
        end
        Providers.record_check!(@provider.key, ok: true)
        return body if code.between?(200, 299)

        error = body["error"].is_a?(String) && body["error"].match?(/\A[a-z_]{1,64}\z/) ? body["error"] : "http_#{code}"
        raise Unauthorized, error if [ 401, 403 ].include?(code)

        raise Rejected, error
      end

      def parse(text)
        parsed = JSON.parse(text.to_s.presence || "{}")
        parsed.is_a?(Hash) ? parsed : {}
      rescue JSON::ParserError
        {}
      end

      # Base64 de DER ou PEM, conforme o PSC.
      def decode_certificate(value)
        text = value.to_s
        return OpenSSL::X509::Certificate.new(text).to_der if text.include?("-----BEGIN")

        Base64.strict_decode64(text.gsub(/\s/, ""))
      rescue ArgumentError, OpenSSL::X509::CertificateError
        nil
      end
    end
  end
end
```

(`raise Rejected, "x"` chama `Rejected.new("x")`, que guarda `code`.)

- [ ] **Step 7: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod19b api bundle exec rspec spec/services/signatures/providers_spec.rb spec/services/signatures/psc_client_spec.rb`
Expected: PASS.

- [ ] **Step 8: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod19b add db/platform_migrate/20261008500001_create_signature_provider_checks.rb db/platform_schema.rb app/models/signature_provider_check.rb app/services/signatures/providers.rb app/services/signatures/psc.rb app/services/signatures/psc/client.rb lib/fake_psc/app.rb spec/support/signature_helpers.rb spec/services/signatures/providers_spec.rb spec/services/signatures/psc_client_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod19b commit -m "feat: talk to ICP-Brasil cloud signature providers through the ITI API

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
### Task 5: Cliente do serviço `signer` (contrato §9) e o `signer` falso das specs

**Dois níveis de teste.** As specs de regra (Tasks 7–17) usam um `FakeSigner` em processo, com a mesma interface do cliente: ele confere de verdade a assinatura RAW contra a chave pública do certificado (o PSC falso assina de verdade) e recusa documento adulterado, mas o formato das "assinaturas" é dele. A prova de CAdES/PAdES AD-RB é das specs `:signer`, contra o serviço real do compose (AC de dev do volume `signer-dev-pki`), que esta task escreve e que voltam na Task 12 (fluxo inteiro) e na Task 19.

**Files:**
- Create: `app/services/signatures/signer.rb`, `app/services/signatures/signer/client.rb`, `spec/support/fake_signer.rb`
- Modify: `spec/support/signature_helpers.rb`, `spec/rails_helper.rb`
- Test: `spec/services/signatures/signer_client_spec.rb`, `spec/integration/signer_service_spec.rb` (`:signer`)

**Interfaces:**
- Consumes: `Signatures::CertificateInfo` (Task 2), `test_pki` (Task 3).
- Produces:
  - `Signatures::Signer::{Error,Unavailable,Rejected(code)}`, `::KINDS == %w[cades pades]`, `::Prepared = Data(:digest, :state)` (`digest` = 32 bytes), `::Assembled = Data(:signature, :validation_material)` (bytes), `::Verification = Data(:status, :signer_cpf, :signer_name, :policy_oid, :signed_at, :reasons)` (`inspect` sem CPF/nome), `::CertificateCheck = Data(:status, :signer_cpf, :not_after, :reasons)` (`inspect` sem CPF); `Signatures::Signer.client -> Client` (de `SIGNER_URL`/`SIGNER_TOKEN`).
  - `Signatures::Signer::Client#prepare(kind:, document:, certificate_der:) -> Prepared`, `#assemble(kind:, state:, signature_value:) -> Assembled`, `#verify(kind:, signature:, document: nil) -> Verification` (`document` obrigatório no CAdES, omitido no PAdES), `#check_certificate(certificate_der:) -> CertificateCheck` (`POST /certificates/check`, contrato D1: cadeia + LCR; `status` `valid`|`invalid`|`indeterminate`), `#health -> { version:, crl_updated_at: }`. 400/422 → `Rejected(code)`; 401, 5xx, rede, `SIGNER_URL` ausente → `Unavailable`.
  - `FakeSigner` (spec/support): mesma interface; `#unavailable=`, `#revoked_serials`, `#untrusted_serials`, `#revocation_unavailable=`, `#calls`; `FakeSigner::POLICY_OID`.
  - Helpers: `stub_signer!(fake = FakeSigner.new) -> FakeSigner`; specs `:signer` só rodam com `SIGNER_URL` (aviso no início da corrida quando ficam de fora) e liberam no WebMock só o host do `signer`.

- [ ] **Step 1: O `signer` falso e os helpers**

```ruby
# spec/support/fake_signer.rb
# Signer falso, em processo, com a interface de Signatures::Signer::Client
# (contrato §9). Confere de verdade a assinatura RAW contra a chave pública do
# certificado e recusa documento adulterado; o formato das "assinaturas" é
# dele. A prova de CAdES/PAdES ICP é das specs :signer, contra o serviço.
class FakeSigner
  POLICY_OID = "2.16.76.1.7.1.1.2.3".freeze
  MARKER = "\n%FAKE-PADES ".b.freeze

  attr_accessor :unavailable, :revoked_serials, :untrusted_serials, :revocation_unavailable
  attr_reader :calls

  def initialize
    @unavailable = false
    @revoked_serials = []
    @untrusted_serials = []
    @revocation_unavailable = false
    @calls = []
  end

  def prepare(kind:, document:, certificate_der:)
    touch!(:prepare)
    doc = Digest::SHA256.hexdigest(document)
    state = { "kind" => kind, "doc" => doc, "cert" => Base64.strict_encode64(certificate_der),
              "pdf" => kind == "pades" ? Base64.strict_encode64(document) : nil }
    Signatures::Signer::Prepared.new(digest: Digest::SHA256.digest("#{kind}|#{doc}"),
                                     state: Base64.strict_encode64(JSON.generate(state)))
  end

  def assemble(kind:, state:, signature_value:)
    touch!(:assemble)
    data = JSON.parse(Base64.strict_decode64(state))
    raise Signatures::Signer::Rejected, "invalid_request" unless data["kind"] == kind

    certificate = OpenSSL::X509::Certificate.new(Base64.strict_decode64(data["cert"]))
    digest = Digest::SHA256.digest("#{kind}|#{data['doc']}")
    unless certificate.public_key.verify_raw("SHA256", signature_value, digest)
      raise Signatures::Signer::Rejected, "invalid_signature_value"
    end

    envelope = JSON.generate("fake" => kind, "doc" => data["doc"], "cert" => data["cert"],
                             "sig" => Base64.strict_encode64(signature_value))
    signature = kind == "pades" ? Base64.strict_decode64(data["pdf"]).b + MARKER + Base64.strict_encode64(envelope).b : envelope.b
    Signatures::Signer::Assembled.new(signature: signature, validation_material: "fake-chain-#{kind}".b)
  end

  def verify(kind:, signature:, document: nil)
    touch!(:verify)
    envelope, original = kind == "pades" ? split_pades(signature) : [ JSON.parse(signature), document ]
    info = Signatures::CertificateInfo.parse(Base64.strict_decode64(envelope["cert"]))
    reasons = []
    reasons << "document_altered" unless original && Digest::SHA256.hexdigest(original) == envelope["doc"]
    reasons << "certificate_revoked" if @revoked_serials.include?(info.serial_number)
    Signatures::Signer::Verification.new(status: reasons.empty? ? "valid" : "invalid", signer_cpf: info.cpf,
                                         signer_name: info.holder_name, policy_oid: POLICY_OID,
                                         signed_at: Time.current, reasons: reasons)
  rescue JSON::ParserError, ArgumentError, Signatures::CertificateInfo::Invalid
    Signatures::Signer::Verification.new(status: "invalid", signer_cpf: nil, signer_name: nil, policy_oid: nil,
                                         signed_at: nil, reasons: [ "malformed" ])
  end

  # Contrato D1: cadeia + LCR do certificado, sem assinar.
  def check_certificate(certificate_der:)
    touch!(:check)
    info = Signatures::CertificateInfo.parse(certificate_der)
    reasons = []
    reasons << "untrusted_chain" if @untrusted_serials.include?(info.serial_number)
    reasons << "certificate_expired" if info.expired?
    reasons << "certificate_revoked" if @revoked_serials.include?(info.serial_number)
    status = reasons.any? ? "invalid" : "valid"
    if status == "valid" && @revocation_unavailable
      status = "indeterminate"
      reasons << "revocation_unavailable"
    end
    Signatures::Signer::CertificateCheck.new(status: status, signer_cpf: info.cpf, not_after: info.not_after, reasons: reasons)
  rescue Signatures::CertificateInfo::Invalid
    raise Signatures::Signer::Rejected, "invalid_certificate"
  end

  def health
    touch!(:health)
    { version: "fake-1", crl_updated_at: Time.current }
  end

  private

  def touch!(name)
    raise Signatures::Signer::Unavailable, "signer falso fora do ar" if @unavailable

    @calls << name
  end

  def split_pades(bytes)
    bytes = bytes.b
    at = bytes.rindex(MARKER)
    raise ArgumentError, "sem assinatura" unless at

    [ JSON.parse(Base64.strict_decode64(bytes[(at + MARKER.bytesize)..])), bytes[0...at] ]
  end
end
```

Em `spec/rails_helper.rb`, junto dos outros: `require_relative "support/fake_signer"` (antes de `support/signature_helpers`).

No módulo `SignatureHelpers` (`spec/support/signature_helpers.rb`):

```ruby
  def stub_signer!(fake = FakeSigner.new)
    allow(Signatures::Signer).to receive(:client).and_return(fake)
    fake
  end
```

e, no fim do arquivo, troque `RSpec.configure { |c| c.include SignatureHelpers }` por:

```ruby
RSpec.configure do |config|
  config.include SignatureHelpers

  # Specs :signer falam com o serviço real (compose: http://signer:8090). Sem
  # SIGNER_URL ficam fora — com aviso, nunca em silêncio.
  if ENV["SIGNER_URL"].to_s.empty?
    config.filter_run_excluding(:signer)
    config.before(:suite) { warn "[signer] SIGNER_URL ausente: specs :signer fora desta corrida" }
  end

  config.around(:each, :signer) do |example|
    WebMock.disable_net_connect!(allow: URI(ENV.fetch("SIGNER_URL")).host)
    example.run
  ensure
    WebMock.disable_net_connect!
  end
end
```

- [ ] **Step 2: Escreva as specs que falham**

```ruby
# spec/services/signatures/signer_client_spec.rb
require "rails_helper"

# Contrato §9 (api/worker → signer): Bearer, JSON em base64, corpo nunca
# logado; 400/422 = recusa com código, 401/5xx/rede = indisponível.
RSpec.describe Signatures::Signer::Client do
  let(:base) { "http://signer.test:8090" }
  let(:client) { described_class.new(url: base, token: "SEGREDO-DO-SIGNER") }
  let(:auth) { { "Authorization" => "Bearer SEGREDO-DO-SIGNER" } }

  def reply(body, status: 200) = { status: status, headers: { "Content-Type" => "application/json" }, body: body.to_json }

  it "prepare e assemble: base64 nas duas pontas" do
    stub_request(:post, "#{base}/prepare").with(headers: auth, body: { kind: "cades", document_base64: Base64.strict_encode64("doc"),
                                                                       certificate_der_base64: Base64.strict_encode64("der"), policy: "AD-RB" })
      .to_return(reply(to_be_signed_sha256_base64: Base64.strict_encode64("h" * 32), prepared_state: "ESTADO"))
    prepared = client.prepare(kind: "cades", document: "doc", certificate_der: "der")
    expect(prepared).to have_attributes(digest: "h" * 32, state: "ESTADO")

    stub_request(:post, "#{base}/assemble").with(body: { kind: "cades", prepared_state: "ESTADO", signature_value_base64: Base64.strict_encode64("RAW") })
      .to_return(reply(signature_base64: Base64.strict_encode64("P7S"), validation_material_base64: Base64.strict_encode64("LCR")))
    expect(client.assemble(kind: "cades", state: "ESTADO", signature_value: "RAW"))
      .to have_attributes(signature: "P7S", validation_material: "LCR")
  end

  it "verify, check do certificado e health (o /health também leva o token)" do
    stub_request(:post, "#{base}/verify").with(body: hash_including("kind" => "pades", "signature_base64" => Base64.strict_encode64("PDF")))
      .to_return(reply(status: "valid", signer_cpf: "52998224725", signer_name: "MARIA", policy_oid: "2.16.76.1.7.1.11.1.1",
                       signed_at: "2026-10-08T12:00:00Z", reasons: []))
    result = client.verify(kind: "pades", signature: "PDF")
    expect(result).to have_attributes(status: "valid", signer_cpf: "52998224725", policy_oid: "2.16.76.1.7.1.11.1.1",
                                      signed_at: Time.utc(2026, 10, 8, 12))
    expect(result.inspect).not_to include("52998224725", "MARIA")

    stub_request(:post, "#{base}/certificates/check").with(headers: auth, body: { certificate_der_base64: Base64.strict_encode64("der") })
      .to_return(reply(status: "invalid", signer_cpf: "52998224725", not_after: "2027-10-08T12:00:00Z", reasons: [ "certificate_revoked" ]))
    check = client.check_certificate(certificate_der: "der")
    expect(check).to have_attributes(status: "invalid", signer_cpf: "52998224725", not_after: Time.utc(2027, 10, 8, 12),
                                     reasons: [ "certificate_revoked" ])
    expect(check.inspect).not_to include("52998224725")
    stub_request(:get, "#{base}/health").with(headers: auth).to_return(reply(version: "1.0.0", crl_updated_at: "2026-10-08T03:00:00Z"))
    expect(client.health).to eq(version: "1.0.0", crl_updated_at: Time.utc(2026, 10, 8, 3))
  end

  it "422 é recusa com código; 401, 500, rede e URL ausente são indisponibilidade; nada vaza" do
    stub_request(:post, "#{base}/assemble").to_return(reply({ error: "invalid_signature_value" }, status: 422))
    expect { client.assemble(kind: "cades", state: "E", signature_value: "R") }
      .to raise_error(Signatures::Signer::Rejected) { |e| expect(e.code).to eq("invalid_signature_value") }
    [ 401, 500 ].each do |status|
      stub_request(:get, "#{base}/health").to_return(reply({ error: "x" }, status: status))
      expect { client.health }.to raise_error(Signatures::Signer::Unavailable) { |e| expect(e.message).not_to include("SEGREDO") }
    end
    stub_request(:get, "#{base}/health").to_raise(Errno::ECONNREFUSED)
    expect { client.health }.to raise_error(Signatures::Signer::Unavailable)
    expect { described_class.new(url: nil, token: nil).health }.to raise_error(Signatures::Signer::Unavailable)
    expect(client.inspect).not_to include("SEGREDO")
  end

  it "status fora do contrato no verify é indisponibilidade (nunca 'valid' por engano)" do
    stub_request(:post, "#{base}/verify").to_return(reply(status: "ok"))
    expect { client.verify(kind: "cades", signature: "S", document: "D") }.to raise_error(Signatures::Signer::Unavailable)
  end
end
```

```ruby
# spec/integration/signer_service_spec.rb
require "rails_helper"
require "prawn"

# Contrato §9 contra o SERVIÇO real (compose): AD-RB em CAdES destacado e em
# PAdES, com a assinatura RAW feita pela chave da folha da AC de teste (como o
# PSC faria). A AC é a do DevPki do signer (volume signer-dev-pki; Task 2).
RSpec.describe "Serviço signer (real)", :signer do
  let(:client) { Signatures::Signer.client }
  let(:leaf) { test_pki.leaf_for(SignatureHelpers::DOCTOR_CPF) }

  def raw(prepared) = leaf.key.sign_raw("SHA256", prepared.digest)

  it "CAdES destacado AD-RB: prepara, monta e verifica; JSON adulterado não verifica" do
    document = '{"schema":"rotasaude.consultation.v1","texto":"Ação ≥ 1"}'
    prepared = client.prepare(kind: "cades", document: document, certificate_der: leaf.der)
    assembled = client.assemble(kind: "cades", state: prepared.state, signature_value: raw(prepared))
    result = client.verify(kind: "cades", signature: assembled.signature, document: document)
    expect(result.status).to eq("valid")
    expect(result.signer_cpf).to eq(SignatureHelpers::DOCTOR_CPF)
    expect(result.policy_oid).to start_with("2.16.76.1.7.1")
    expect(assembled.validation_material.bytesize).to be > 0
    expect(client.verify(kind: "cades", signature: assembled.signature, document: document.sub("1", "2")).status)
      .not_to eq("valid")
  end

  it "PAdES AD-RB sobre um PDF do Prawn" do
    pdf = Prawn::Document.new.tap { |d| d.text "Registro de consulta" }.render
    prepared = client.prepare(kind: "pades", document: pdf, certificate_der: leaf.der)
    assembled = client.assemble(kind: "pades", state: prepared.state, signature_value: raw(prepared))
    expect(assembled.signature).to start_with("%PDF")
    expect(client.verify(kind: "pades", signature: assembled.signature).status).to eq("valid")
  end

  it "valor de assinatura que não confere é recusado; o certificado de teste confere; health responde" do
    prepared = client.prepare(kind: "cades", document: "{}", certificate_der: leaf.der)
    expect { client.assemble(kind: "cades", state: prepared.state, signature_value: "x" * 256) }
      .to raise_error(Signatures::Signer::Rejected)
    check = client.check_certificate(certificate_der: leaf.der)
    expect([ check.status, check.signer_cpf ]).to eq([ "valid", SignatureHelpers::DOCTOR_CPF ])
    expect(client.health[:version]).to be_present
  end
end
```

- [ ] **Step 3: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod19b api bundle exec rspec spec/services/signatures/signer_client_spec.rb`
Expected: FAIL (`uninitialized constant Signatures::Signer`).

- [ ] **Step 4: Implemente**

```ruby
# app/services/signatures/signer.rb
# Serviço interno signer (ADR 0032; contrato §9): prepara, monta e valida
# CAdES/PAdES AD-RB. Sem estado; o documento nunca sai da infraestrutura.
module Signatures
  module Signer
    class Error < StandardError; end
    # Rede, 5xx, 401 (token errado é configuração) ou URL ausente.
    class Unavailable < Error; end

    class Rejected < Error
      attr_reader :code

      def initialize(code)
        @code = code.to_s
        super("o signer recusou: #{@code}")
      end
    end

    KINDS = %w[cades pades].freeze
    STATUSES = %w[valid invalid indeterminate].freeze

    Prepared = Data.define(:digest, :state) do
      def inspect = "#<Signatures::Signer::Prepared>"
    end
    Assembled = Data.define(:signature, :validation_material) do
      def inspect = "#<Signatures::Signer::Assembled #{signature.bytesize} bytes>"
    end
    Verification = Data.define(:status, :signer_cpf, :signer_name, :policy_oid, :signed_at, :reasons) do
      def valid? = status == "valid"
      def inspect = "#<Signatures::Signer::Verification #{status} #{reasons.join(',')}>"
    end
    CertificateCheck = Data.define(:status, :signer_cpf, :not_after, :reasons) do
      def inspect = "#<Signatures::Signer::CertificateCheck #{status} #{reasons.join(',')}>"
    end

    module_function

    def client = Client.new(url: ENV["SIGNER_URL"], token: ENV["SIGNER_TOKEN"])
  end
end
```

```ruby
# app/services/signatures/signer/client.rb
# Cliente HTTP do signer (contrato §9). Corpo nunca logado; nenhuma mensagem
# carrega token, documento ou CPF.
require "net/http"

module Signatures
  module Signer
    class Client
      TIMEOUT = 30
      NETWORK_ERRORS = [ SocketError, IOError, EOFError, SystemCallError, Net::OpenTimeout, Net::ReadTimeout,
                         Net::WriteTimeout, Net::ProtocolError, OpenSSL::SSL::SSLError ].freeze

      def initialize(url:, token:, timeout: TIMEOUT)
        @url = url.to_s.chomp("/")
        @token = token.to_s
        @timeout = timeout
      end

      def prepare(kind:, document:, certificate_der:)
        body = post("/prepare", kind: kind!(kind), document_base64: b64(document),
                                certificate_der_base64: b64(certificate_der), policy: "AD-RB")
        Prepared.new(digest: decode(body, "to_be_signed_sha256_base64"), state: body["prepared_state"].to_s.presence || bad!)
      end

      def assemble(kind:, state:, signature_value:)
        body = post("/assemble", kind: kind!(kind), prepared_state: state, signature_value_base64: b64(signature_value))
        Assembled.new(signature: decode(body, "signature_base64"), validation_material: decode(body, "validation_material_base64"))
      end

      def verify(kind:, signature:, document: nil)
        payload = { kind: kind!(kind), signature_base64: b64(signature) }
        payload[:document_base64] = b64(document) if document
        body = post("/verify", **payload)
        bad! unless STATUSES.include?(body["status"])

        Verification.new(status: body["status"], signer_cpf: body["signer_cpf"].presence, signer_name: body["signer_name"].presence,
                         policy_oid: body["policy_oid"].presence, signed_at: time(body["signed_at"]),
                         reasons: Array(body["reasons"]).map(&:to_s))
      end

      # Contrato D1: cadeia + LCR do certificado, sem assinar (vínculo e renovação).
      def check_certificate(certificate_der:)
        body = post("/certificates/check", certificate_der_base64: b64(certificate_der))
        bad! unless STATUSES.include?(body["status"])

        CertificateCheck.new(status: body["status"], signer_cpf: body["signer_cpf"].presence, not_after: time(body["not_after"]),
                             reasons: Array(body["reasons"]).map(&:to_s))
      end

      def health
        body = perform(Net::HTTP::Get, "/health")
        { version: body["version"].to_s, crl_updated_at: time(body["crl_updated_at"]) }
      end

      def inspect = "#<Signatures::Signer::Client>"

      private

      def post(path, **payload) = perform(Net::HTTP::Post, path, JSON.generate(payload))

      def perform(verb, path, body = nil)
        raise Unavailable, "SIGNER_URL ausente" if @url.empty?

        uri = URI("#{@url}#{path}")
        request = verb.new(uri)
        request["Authorization"] = "Bearer #{@token}"
        request["Accept"] = "application/json"
        if body
          request["Content-Type"] = "application/json"
          request.body = body
        end
        response = Net::HTTP.start(uri.host, uri.port, use_ssl: uri.scheme == "https", open_timeout: @timeout,
                                                       read_timeout: @timeout, write_timeout: @timeout) { |http| http.request(request) }
        interpret(response)
      rescue *NETWORK_ERRORS => e
        raise Unavailable, "signer inalcançável (#{e.class.name})"
      end

      def interpret(response)
        code = response.code.to_i
        parsed = parse(response.body)
        return parsed if code == 200
        raise Rejected, (parsed["error"].to_s.match?(/\A[a-z_]{1,64}\z/) ? parsed["error"] : "http_#{code}") if [ 400, 422 ].include?(code)

        raise Unavailable, "o signer respondeu #{code}"
      end

      def parse(text)
        parsed = JSON.parse(text.to_s.presence || "{}")
        parsed.is_a?(Hash) ? parsed : {}
      rescue JSON::ParserError
        {}
      end

      def kind!(kind)
        raise ArgumentError, "kind fora do contrato" unless KINDS.include?(kind)

        kind
      end

      def b64(bytes) = Base64.strict_encode64(bytes.to_s.b)

      def decode(body, key)
        Base64.strict_decode64(body[key].to_s.presence || bad!)
      rescue ArgumentError
        bad!
      end

      def time(value)
        value.present? ? Time.iso8601(value.to_s) : nil
      rescue ArgumentError
        bad!
      end

      def bad! = raise(Unavailable, "resposta do signer ilegível")
    end
  end
end
```

- [ ] **Step 5: Rode e veja passar (e a integração, com o `signer` do compose)**

Run: `docker compose exec -T -w /rails/.claude/mod19b api bundle exec rspec spec/services/signatures/signer_client_spec.rb`
Expected: PASS.

Run (com o `signer` e o `fake-psc` de pé no compose — Task 18 Step 1 descreve os serviços; se ainda não existirem, rode esta parte depois da Task 18): `docker compose exec -T -e SIGNER_URL=http://signer:8090 -w /rails/.claude/mod19b api bundle exec rspec spec/integration/signer_service_spec.rb`
Expected: PASS (3 exemplos). `indeterminate` ou `untrusted_chain` no lugar de `valid` = o api não está lendo a AC do volume (`SIGNER_DEV_PKI_DIR=/signer-dev-pki`) ou o `signer` não serve as LCRs em `/dev-pki/`.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod19b add app/services/signatures/signer.rb app/services/signatures/signer/client.rb spec/support/fake_signer.rb spec/support/signature_helpers.rb spec/rails_helper.rb spec/services/signatures/signer_client_spec.rb spec/integration/signer_service_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod19b commit -m "feat: add the signer service client with a real-service contract spec

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
## Fatia 2 — Vínculo e sessão do turno (F-19.9, F-19.10)

### Task 6: `state` de uso único do OAuth

**Files:**
- Create: `app/services/signatures/oauth_states.rb`
- Test: `spec/services/signatures/oauth_states_spec.rb`

**Interfaces:**
- Consumes: `SignatureOauthState` (Task 3).
- Produces:
  - `Signatures::OauthStates::Issued = Data(:state, :verifier, :challenge, :row)`; `.issue!(user:, purpose:, provider:, return_to: nil, request_ids: [], city: Current.city, now: Time.current) -> Issued` (o `state` é um token assinado `{ c: slug, s: id }`; a linha guarda o `code_verifier` cifrado e vale 10 min); `.challenge(verifier) -> String` (S256, base64url sem `=`); `.safe_return_to(value) -> String` (`/^\/([^\/]\S{0,198})?$/`, senão `"/"`); `.consume(state, user:, now:) -> Result` (ok `{ state: SignatureOauthState }`; falhas `:invalid_state` (assinatura, cidade, usuário, já usado) e `:authorization_expired` (vencido — e fica consumido)). Chame `consume` numa transação própria: o consumo vale mesmo que o passo seguinte falhe.

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/services/signatures/oauth_states_spec.rb
require "rails_helper"

# ADR 0032 (spec §10; Review Focus 4): state de uso único, amarrado à cidade, ao
# usuário e ao propósito; PKCE S256; 10 minutos.
RSpec.describe Signatures::OauthStates do
  before { Current.city = signature_city! }
  after { Current.reset }

  let(:doctor) { signer_doctor!(create_unit) }

  def issue(**over) = described_class.issue!(user: doctor, purpose: "link", provider: "vidaas", return_to: "/conta/assinatura", **over)

  it "emite state assinado, verifier com o challenge S256 e guarda o verifier cifrado" do
    issued = issue
    expect(issued.challenge).to eq(Base64.urlsafe_encode64(Digest::SHA256.digest(issued.verifier), padding: false))
    expect(issued.verifier.size).to be >= 43
    raw = ApplicationRecord.connection.select_value("SELECT code_verifier FROM signature_oauth_states WHERE id = #{ApplicationRecord.connection.quote(issued.row.id)}")
    expect(raw).not_to include(issued.verifier)
    expect(issued.state).not_to include(issued.verifier)
  end

  it "consome uma vez só; outro usuário, outra cidade e adulterado não consomem" do
    issued = issue
    other = signer_doctor!(create_unit("UBS Dois"), cpf: SignatureHelpers::OTHER_CPF)
    expect(described_class.consume(issued.state, user: other).reason).to eq(:invalid_state)
    Current.set(city: City.new(slug: "outra-cidade")) do
      expect(described_class.consume(issued.state, user: doctor).reason).to eq(:invalid_state)
    end
    expect(described_class.consume("#{issued.state}x", user: doctor).reason).to eq(:invalid_state)
    expect(described_class.consume(nil, user: doctor).reason).to eq(:invalid_state)
    expect(described_class.consume("a" * 5000, user: doctor).reason).to eq(:invalid_state)
    first = described_class.consume(issued.state, user: doctor)
    expect(first).to be_ok
    expect(first.payload[:state].code_verifier).to eq(issued.verifier)
    expect(described_class.consume(issued.state, user: doctor).reason).to eq(:invalid_state)
  end

  it "vencido: authorization_expired, e fica consumido" do
    issued = issue
    travel 11.minutes do
      expect(described_class.consume(issued.state, user: doctor).reason).to eq(:authorization_expired)
      expect(described_class.consume(issued.state, user: doctor).reason).to eq(:invalid_state)
    end
  end

  it "return_to só aceita caminho relativo do dashboard" do
    expect(described_class.safe_return_to("/signature/pendentes")).to eq("/signature/pendentes")
    [ "//evil.test/x", "https://evil.test", "assinatura", "/a b", nil, 42, "/#{'x' * 300}" ].each do |value|
      expect(described_class.safe_return_to(value)).to eq("/"), value.inspect
    end
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod19b api bundle exec rspec spec/services/signatures/oauth_states_spec.rb`
Expected: FAIL (`uninitialized constant Signatures::OauthStates`).

- [ ] **Step 3: Implemente**

```ruby
# app/services/signatures/oauth_states.rb
# State do OAuth com o PSC (ADR 0032; spec §10). O valor que viaja é um token
# assinado { c: cidade, s: id da linha } — a cidade do token tem de ser a do
# host que recebe o callback; a linha, no banco da cidade, guarda o code_verifier (PKCE)
# cifrado, o usuário, o propósito e o uso único. O token não leva o verifier.
module Signatures
  module OauthStates
    PURPOSE = :signature_oauth_state
    SIGNATURE_TTL = 1.day # a validade de USO é a da linha (10 min); esta só separa forjado de vencido
    RETURN_TO = %r{\A/(?:[^/]\S{0,198})?\z}
    MAX_STATE = 1024

    Issued = Data.define(:state, :verifier, :challenge, :row) do
      def inspect = "#<Signatures::OauthStates::Issued purpose=#{row.purpose}>"
    end

    module_function

    def issue!(user:, purpose:, provider:, return_to: nil, request_ids: [], city: Current.city, now: Time.current)
      verifier = SecureRandom.urlsafe_base64(48)
      row = SignatureOauthState.create!(user_id: user.id, purpose: purpose, provider: provider, code_verifier: verifier,
                                        request_ids: request_ids, return_to: safe_return_to(return_to),
                                        expires_at: now + SignatureOauthState::TTL, created_at: now)
      state = verifier_service.generate({ "c" => city.slug, "s" => row.id }, purpose: PURPOSE, expires_in: SIGNATURE_TTL)
      Issued.new(state: state, verifier: verifier, challenge: challenge(verifier), row: row)
    end

    def challenge(verifier) = Base64.urlsafe_encode64(Digest::SHA256.digest(verifier), padding: false)

    def safe_return_to(value) = value.is_a?(String) && value.match?(RETURN_TO) ? value : "/"

    def consume(state, user:, now: Time.current)
      payload = decode(state)
      return Result.fail(:invalid_state) unless payload && payload["c"] == Current.city&.slug && payload["s"].is_a?(String)

      row = SignatureOauthState.lock.find_by(id: payload["s"])
      return Result.fail(:invalid_state) unless row && row.user_id == user.id && row.consumed_at.nil?

      row.update!(consumed_at: now)
      return Result.fail(:authorization_expired) if row.expires_at <= now

      Result.ok(state: row)
    end

    def decode(state)
      return nil unless state.is_a?(String) && state.present? && state.size <= MAX_STATE

      payload = verifier_service.verified(state, purpose: PURPOSE)
      payload.is_a?(Hash) ? payload : nil
    rescue ActiveSupport::MessageVerifier::InvalidSignature, ArgumentError
      nil
    end

    def verifier_service = Rails.application.message_verifier("signature_oauth_state")
  end
end
```

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod19b api bundle exec rspec spec/services/signatures/oauth_states_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod19b add app/services/signatures/oauth_states.rb spec/services/signatures/oauth_states_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod19b commit -m "feat: add single-use PKCE states for the provider authorization

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 7: Vínculo do certificado (localizar, vincular, desvincular) e o callback

**Files:**
- Create: `app/commands/signatures/{discover,start_link,accept_certificate,unlink,complete_oauth}.rb`, `app/services/signatures/json.rb`, `app/controllers/signatures/{base,certificates,oauth}_controller.rb`
- Modify: `config/routes.rb`
- Test: `spec/requests/signature_certificates_spec.rb`

**Interfaces:**
- Consumes: Tasks 1–6 (`Signatures::Gate`, `DigitalSignatureGate`, `SignerCertificate`, `SignatureSession`, `Providers`, `Psc::Client`, `Signer.client#check_certificate`, `OauthStates`, `CertificateInfo`); `MfaStepUp`, `AttendanceAccess#require_professional`/`render_failure` (existentes).
- Produces:
  - `Signatures::Discover.call(user:) -> Result` (ok `{ providers: [{ provider:, found: }], unavailable: [key] }`; falha `:professional_cpf_missing`).
  - `Signatures::StartLink.call(user:, provider:, return_to:) -> Result` (ok `{ authorize_url: }`, escopo `single_signature`; falhas `:invalid_provider`, `:professional_cpf_missing`).
  - `Signatures::AcceptCertificate.call(user:, provider:, entries:, now: Time.current, signer: Signer.client) -> Result` (ok `{ certificate: SignerCertificate, changed: bool }`; falhas `:professional_cpf_missing`, `:certificate_not_found`, `:certificate_cpf_mismatch`, `:certificate_expired`, `:certificate_revoked`, `:certificate_untrusted` (o `POST /certificates/check` do `signer` diz `invalid` com `certificate_revoked` / `certificate_expired` / `untrusted_chain`), `:signer_unavailable` (`signer` fora do ar ou 422 `invalid_certificate`)); `indeterminate` aceita e grava `link_check_status`/`link_check_reasons`. Troca o `active` só quando o serial (ou o PSC) muda; o anterior vira `replaced` e a sessão ativa é revogada; evento `signature.certificate_linked { certificate_id, user_id, provider }`.
  - `Signatures::Unlink.call(user:) -> Result` (ok `{ certificate: }`; falha `:certificate_not_linked`); evento `signature.certificate_unlinked`.
  - `Signatures::CompleteOauth.call(user:, state:, code:, error: nil, now: Time.current) -> Result` (ok `{ purpose: "link"|"session"|"batch", record:, return_to: }`; falhas `:invalid_state`, `:authorization_expired`, `:authorization_denied`, `:provider_unavailable` e as do propósito). Nesta task só `link`; `session` entra na Task 8 e `batch` na Task 13.
  - `Signatures::Json.certificate(certificate, now: Time.current) -> Hash` (contrato §3).
  - `Signatures::BaseController` (`ERROR_STATUS`, `#failure(result, overrides = {})`, `#body`; `before_action :require_digital_signature!, :require_professional`).
  - Rotas: `GET|DELETE /signature/certificates/current`, `POST /signature/certificates/discover`, `POST /signature/certificates/link`, `POST /signature/oauth/callback`.

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/requests/signature_certificates_spec.rb
require "rails_helper"

# Contrato §3–§4 (spec §5, vínculo; NGS2.01.02/02.02): localizar por CPF em
# cada PSC habilitado, vincular com step-up e PKCE, conferir CPF, validade, uso
# e revogação (signer, contrato D1); desvincular com step-up. Review Focus 4: callback nas bordas.
RSpec.describe "Vínculo do certificado", type: :request do
  before do
    signature_city!
    stub_psc!(%w[vidaas birdid])
    @signer = stub_signer!
  end

  let(:unit) { create_unit }
  let(:doctor) { signer_doctor!(unit) }
  let(:cpf) { SignatureHelpers::DOCTOR_CPF }
  def body = JSON.parse(response.body)

  def stepped_in(user = doctor) = sign_in_as(user).update!(mfa_verified_at: Time.current)

  def start_link(provider = "vidaas")
    post "/signature/certificates/link", params: { provider: provider, return_to: "/conta/assinatura" }, as: :json
    body.fetch("authorize_url")
  end

  def state_of(url) = URI.decode_www_form(URI(url).query).to_h.fetch("state")

  def callback(state, code: nil, error: nil)
    post "/signature/oauth/callback", params: { state: state, code: code, error: error }.compact, as: :json
  end

  it "localiza o CPF do profissional em cada PSC habilitado; PSC fora do ar vai para unavailable" do
    sign_in_as(doctor)
    fake_psc("birdid").absent_cpfs << cpf
    post "/signature/certificates/discover", as: :json
    expect(body).to eq("providers" => [ { "provider" => "vidaas", "found" => true }, { "provider" => "birdid", "found" => false } ],
                       "unavailable" => [])
    fake_psc("birdid").failures << 503
    post "/signature/certificates/discover", as: :json
    expect(body).to eq("providers" => [ { "provider" => "vidaas", "found" => true } ], "unavailable" => [ "birdid" ])
  end

  it "vincula: step-up, URL com PKCE e CPF, callback grava o certificado; GET current" do
    sign_in_as(doctor)
    post "/signature/certificates/link", params: { provider: "vidaas" }, as: :json
    expect([ response.status, body["error"] ]).to eq([ 401, "mfa_required" ])
    stepped_in
    post "/signature/certificates/link", params: { provider: "safeid" }, as: :json
    expect([ response.status, body["error"] ]).to eq([ 422, "invalid_provider" ])

    url = start_link
    query = URI.decode_www_form(URI(url).query).to_h
    expect(query).to include("scope" => "single_signature", "code_challenge_method" => "S256", "login_hint" => cpf)
    code = authorize_and_approve!(url)
    callback(state_of(url), code: code)
    expect(response).to have_http_status(:ok)
    expect(body.slice("purpose", "return_to")).to eq("purpose" => "link", "return_to" => "/conta/assinatura")
    leaf = fake_psc.leaf(cpf)
    expect(body["result"]).to include("provider" => "vidaas", "issuer" => Signatures::CertificateInfo.new(fake_psc.leaf(cpf).certificate).issuer_name,
                                      "serial_number" => leaf.serial_hex, "status" => "active")
    expect(body["result"]["expires_in_days"]).to be_between(360, 366)
    expect(response.body).not_to include(*fake_psc.issued_tokens)

    get "/signature/certificates/current"
    expect(body["serial_number"]).to eq(leaf.serial_hex)
    expect(DomainEvent.where(name: "signature.certificate_linked").sole.payload.keys).to match_array(%w[certificate_id user_id provider])
  end

  it "callback nas bordas: reuso, outro usuário, vencido, recusado no celular, código já trocado (Review Focus 4)" do
    stepped_in
    url = start_link
    code = authorize_and_approve!(url)
    callback(state_of(url), code: code)
    callback(state_of(url), code: code)
    expect([ response.status, body["error"] ]).to eq([ 422, "invalid_state" ])

    url = start_link
    other = signer_doctor!(create_unit("UBS Dois"), cpf: SignatureHelpers::OTHER_CPF)
    stepped_in(other)
    callback(state_of(url), code: authorize_and_approve!(url))
    expect([ response.status, body["error"] ]).to eq([ 422, "invalid_state" ])
    expect(SignerCertificate.where(user_id: other.id)).to be_empty

    stepped_in
    url = start_link
    code = authorize_and_approve!(url)
    travel 11.minutes do
      callback(state_of(url), code: code)
      expect([ response.status, body["error"] ]).to eq([ 409, "authorization_expired" ])
    end

    url = start_link
    Net::HTTP.get_response(URI(url))
    fake_psc.decide!(state_of(url), approve: false)
    callback(state_of(url), error: "access_denied")
    expect([ response.status, body["error"] ]).to eq([ 403, "authorization_denied" ])

    url = start_link
    code = authorize_and_approve!(url)
    Signatures::Psc::Client.for("vidaas").exchange(code: code, verifier: "qualquer", redirect_uri: "x") rescue nil # o falso apaga o código no 1º uso
    callback(state_of(url), code: code)
    expect([ response.status, body["error"] ]).to eq([ 409, "authorization_expired" ])
  end

  it "certificado de outro CPF, vencido, sem não repúdio, revogado ou de cadeia desconhecida não é vinculado" do
    stepped_in
    {
      test_pki.issue(cpf: SignatureHelpers::OTHER_CPF, name: "OUTRA PESSOA") => "certificate_cpf_mismatch",
      test_pki.issue(cpf: cpf, name: "VENCIDO", not_before: 2.years.ago, not_after: 1.day.ago) => "certificate_expired",
      test_pki.issue(cpf: cpf, name: "SEM NR", key_usage: "digitalSignature") => "certificate_not_found"
    }.each do |leaf, error|
      fake_psc.certificate_overrides[cpf] = leaf
      url = start_link
      callback(state_of(url), code: authorize_and_approve!(url))
      expect([ response.status, body["error"] ]).to eq([ 422, error ]), error
    end
    leaf = test_pki.issue(cpf: cpf, name: "REVOGADO")
    fake_psc.certificate_overrides[cpf] = leaf
    @signer.revoked_serials << leaf.serial_hex # o check do signer diz invalid/certificate_revoked
    url = start_link
    callback(state_of(url), code: authorize_and_approve!(url))
    expect([ response.status, body["error"] ]).to eq([ 422, "certificate_revoked" ])
    expect(SignerCertificate.where(user_id: doctor.id)).to be_empty
    expect(@signer.calls).to include(:check)

    leaf = test_pki.issue(cpf: cpf, name: "CADEIA DESCONHECIDA")
    fake_psc.certificate_overrides[cpf] = leaf
    @signer.untrusted_serials << leaf.serial_hex
    url = start_link
    callback(state_of(url), code: authorize_and_approve!(url))
    expect([ response.status, body["error"] ]).to eq([ 422, "certificate_untrusted" ])
  end

  it "LCR fora do ar no vínculo: aceita e guarda o motivo; signer fora do ar: 503 signer_unavailable" do
    stepped_in
    @signer.unavailable = true
    url = start_link
    callback(state_of(url), code: authorize_and_approve!(url))
    expect([ response.status, body["error"] ]).to eq([ 503, "signer_unavailable" ])
    @signer.unavailable = false
    @signer.revocation_unavailable = true
    url = start_link
    callback(state_of(url), code: authorize_and_approve!(url))
    expect(response).to have_http_status(:ok)
    expect(SignerCertificate.active.find_by(user_id: doctor.id))
      .to have_attributes(link_check_status: "indeterminate", link_check_reasons: [ "revocation_unavailable" ])
  end

  it "PSC fora do ar na troca do código: 503 provider_unavailable" do
    stepped_in
    url = start_link
    code = authorize_and_approve!(url)
    fake_psc.failures << 503
    callback(state_of(url), code: code)
    expect([ response.status, body["error"] ]).to eq([ 503, "provider_unavailable" ])
  end

  it "desvincula com step-up; a sessão ativa cai; sem certificado: 404" do
    certificate = linked_certificate!(doctor)
    session = signature_session!(doctor, certificate: certificate)
    sign_in_as(doctor)
    delete "/signature/certificates/current"
    expect(response).to have_http_status(:unauthorized)
    stepped_in
    delete "/signature/certificates/current"
    expect(response).to have_http_status(:no_content)
    expect([ certificate.reload.status, session.reload.status ]).to eq(%w[unlinked revoked])
    get "/signature/certificates/current"
    expect([ response.status, body["error"] ]).to eq([ 404, "certificate_not_linked" ])
    delete "/signature/certificates/current"
    expect(response).to have_http_status(:not_found)
  end

  it "interruptor desligado: 403 feature_disabled; recepção: 403 missing_role; sem CPF no perfil: 409" do
    signature_city!(enabled: false)
    sign_in_as(doctor)
    get "/signature/certificates/current"
    expect([ response.status, body ]).to eq([ 403, { "error" => "feature_disabled", "feature" => "digital_signature" } ])
    signature_city!
    sign_in_as(reception!)
    post "/signature/certificates/discover", as: :json
    expect([ response.status, body["error"] ]).to eq([ 403, "missing_role" ])
    doctor.professional.update!(cpf: nil)
    sign_in_as(doctor)
    post "/signature/certificates/discover", as: :json
    expect([ response.status, body["error"] ]).to eq([ 409, "professional_cpf_missing" ])
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod19b api bundle exec rspec spec/requests/signature_certificates_spec.rb`
Expected: FAIL (`No route matches [POST] "/signature/certificates/discover"`).

- [ ] **Step 3: Implemente os comandos**

```ruby
# app/commands/signatures/discover.rb
# Localização do titular por CPF em cada PSC habilitado (ADR 0032; spec §5;
# serviço obrigatório do DOC-ICP-17.01). PSC fora do ar não derruba a lista.
module Signatures
  module Discover
    module_function

    def call(user:)
      cpf = user.professional&.cpf
      return Result.fail(:professional_cpf_missing) if cpf.blank?

      providers = []
      unavailable = []
      Providers.configured.each do |provider|
        providers << { provider: provider.key, found: Psc::Client.new(provider).discover(cpf) }
      rescue Psc::Error
        unavailable << provider.key
      end
      Result.ok(providers: providers, unavailable: unavailable)
    end
  end
end
```

```ruby
# app/commands/signatures/start_link.rb
# Começa o vínculo (ADR 0032; spec §5): OAuth com PKCE, escopo
# single_signature, CPF do profissional como login_hint.
module Signatures
  module StartLink
    module_function

    def call(user:, provider:, return_to:)
      return Result.fail(:invalid_provider) unless provider.is_a?(String) && Providers.configured?(provider)

      cpf = user.professional&.cpf
      return Result.fail(:professional_cpf_missing) if cpf.blank?

      issued = OauthStates.issue!(user: user, purpose: "link", provider: provider, return_to: return_to)
      url = Psc::Client.for(provider).authorize_url(state: issued.state, challenge: issued.challenge, scope: "single_signature",
                                                    login_hint: cpf, redirect_uri: Providers.redirect_uri(Current.city))
      Result.ok(authorize_url: url)
    end
  end
end
```

```ruby
# app/commands/signatures/accept_certificate.rb
# Aceita o certificado lido do PSC (ADR 0032; spec §5; NGS2.01.02, 01.03,
# 02.02): do CPF do profissional, dentro da validade, com digitalSignature +
# nonRepudiation e, no signer (contrato D1: cadeia + LCR), de cadeia conhecida e
# não revogado (indeterminate aceita e guarda o motivo). Mesmo serial do active:
# nada muda. Outro: o anterior vira replaced e a sessão
# ativa cai.
module Signatures
  module AcceptCertificate
    module_function

    def call(user:, provider:, entries:, now: Time.current, signer: Signer.client)
      cpf = user.professional&.cpf
      return Result.fail(:professional_cpf_missing) if cpf.blank?

      infos = entries.filter_map { |entry| parse(entry) }
      return Result.fail(:certificate_not_found) if infos.empty?

      mine = infos.select { |_entry, info| info.cpf == cpf }
      return Result.fail(:certificate_cpf_mismatch) if mine.empty?

      usable = mine.select { |_entry, info| info.signing_usage? }
      return Result.fail(:certificate_not_found) if usable.empty?

      current = usable.reject { |_entry, info| info.expired?(now) }
      return Result.fail(:certificate_expired) if current.empty?

      entry, info = current.max_by { |_entry, candidate| candidate.not_after }
      check = signer.check_certificate(certificate_der: info.der)
      return Result.fail(:certificate_cpf_mismatch) if check.signer_cpf.present? && check.signer_cpf != cpf
      if check.status == "invalid"
        return Result.fail(:certificate_revoked) if check.reasons.include?("certificate_revoked")
        return Result.fail(:certificate_expired) if check.reasons.include?("certificate_expired")

        return Result.fail(:certificate_untrusted) # untrusted_chain (e qualquer outra recusa da cadeia)
      end

      # indeterminate (LCR fora do ar): aceita e guarda o motivo; a revogação
      # volta a ser conferida no /verify da primeira assinatura.
      store(user, provider, entry, info, now, check)
    rescue Signer::Unavailable, Signer::Rejected # 422 invalid_certificate do signer também
      Result.fail(:signer_unavailable)
    end

    def parse(entry)
      [ entry, CertificateInfo.parse(entry.der) ]
    rescue CertificateInfo::Invalid
      nil
    end

    def store(user, provider, entry, info, now, check)
      ApplicationRecord.transaction do
        active = SignerCertificate.active.lock.find_by(user_id: user.id)
        next Result.ok(certificate: active, changed: false) if active && active.serial_number == info.serial_number && active.provider == provider

        active&.update!(status: "replaced")
        SignatureSession.active.where(user_id: user.id).update_all(status: "revoked", updated_at: now)
        certificate = SignerCertificate.create!(
          user: user, provider: provider, certificate_alias: entry.certificate_alias, serial_number: info.serial_number,
          issuer_dn: info.issuer_dn, subject_cpf: info.cpf, not_before: info.not_before, not_after: info.not_after,
          status: "active", certificate_der: Base64.strict_encode64(info.der),
          link_check_status: check.status, link_check_reasons: check.reasons
        )
        DomainEvents.publish("signature.certificate_linked", certificate_id: certificate.id, user_id: user.id, provider: provider)
        Result.ok(certificate: certificate, changed: true)
      end
    end
    private_class_method :parse, :store
  end
end
```

```ruby
# app/commands/signatures/unlink.rb
# Desvínculo (ADR 0032; spec §5, step-up no controller): o certificado vira
# unlinked e a sessão ativa cai. Pedidos pendentes ficam pendentes (o autor
# pode vincular de novo ou devolver ao papel).
module Signatures
  module Unlink
    module_function

    def call(user:, now: Time.current)
      ApplicationRecord.transaction do
        certificate = SignerCertificate.active.lock.find_by(user_id: user.id)
        next Result.fail(:certificate_not_linked) unless certificate

        certificate.update!(status: "unlinked")
        SignatureSession.active.where(user_id: user.id).update_all(status: "revoked", updated_at: now)
        DomainEvents.publish("signature.certificate_unlinked", certificate_id: certificate.id, user_id: user.id,
                                                                provider: certificate.provider)
        Result.ok(certificate: certificate)
      end
    end
  end
end
```

```ruby
# app/commands/signatures/complete_oauth.rb
# Volta do PSC (ADR 0032; contrato §4). O state é consumido numa transação
# própria ANTES de qualquer outra coisa: o mesmo código nunca é trocado duas
# vezes, mesmo que o passo seguinte falhe. Depois, troca o código (PKCE) e
# segue pelo propósito.
module Signatures
  module CompleteOauth
    MAX_CODE = 2048

    module_function

    def call(user:, state:, code:, error: nil, now: Time.current)
      consumed = ApplicationRecord.transaction { OauthStates.consume(state, user: user, now: now) }
      return consumed if consumed.failure?

      row = consumed.payload[:state]
      return Result.fail(:authorization_denied) if error.present?
      return Result.fail(:invalid_state) unless code.is_a?(String) && code.present? && code.size <= MAX_CODE

      client = Psc::Client.for(row.provider)
      token = client.exchange(code: code, verifier: row.code_verifier, redirect_uri: Providers.redirect_uri(Current.city))
      outcome = dispatch(row, user: user, client: client, token: token, now: now)
      return outcome if outcome.failure?

      Result.ok(purpose: row.purpose, record: outcome.payload.fetch(:record), return_to: row.return_to)
    rescue Psc::Unavailable
      Result.fail(:provider_unavailable)
    rescue Psc::Unauthorized
      Result.fail(:provider_unavailable) # a credencial da plataforma foi recusada: não é culpa do profissional
    rescue Psc::Rejected
      Result.fail(:authorization_expired) # código vencido ou já trocado no PSC
    end

    def dispatch(row, user:, client:, token:, now:)
      case row.purpose
      when "link"
        accepted = AcceptCertificate.call(user: user, provider: row.provider, entries: client.certificates(token.access_token), now: now)
        accepted.ok? ? Result.ok(record: accepted.payload[:certificate]) : accepted
      else
        Result.fail(:invalid_state)
      end
    end
    private_class_method :dispatch
  end
end
```

- [ ] **Step 4: JSON, controllers e rotas**

```ruby
# app/services/signatures/json.rb
# Formas do contrato do 19b (contrato §3–§5).
module Signatures
  module Json
    module_function

    def certificate(certificate, now: Time.current)
      { id: certificate.id, provider: certificate.provider, issuer: certificate.info.issuer_name,
        serial_number: certificate.serial_number, not_after: certificate.not_after.iso8601, status: certificate.status,
        expires_in_days: certificate.expires_in_days(now) }
    end
  end
end
```

```ruby
# app/controllers/signatures/base_controller.rb
# Rotas da assinatura digital (ADR 0032; contrato). Interruptor utilizável e
# papel health_professional (a recepção recebe 403 missing_role); as rotas do
# admin e as de leitura de assinatura gravada trocam os before_actions.
module Signatures
  class BaseController < ApplicationController
    include Authentication
    include AttendanceAccess
    include DigitalSignatureGate
    include MfaStepUp

    wrap_parameters false

    ERROR_STATUS = {
      missing_role: :forbidden, not_author: :forbidden, authorization_denied: :forbidden, out_of_context: :forbidden,
      invalid_provider: :unprocessable_entity, invalid_state: :unprocessable_entity, invalid_reason: :unprocessable_entity,
      invalid_period: :unprocessable_entity, certificate_cpf_mismatch: :unprocessable_entity,
      certificate_expired: :unprocessable_entity, certificate_revoked: :unprocessable_entity,
      certificate_not_found: :unprocessable_entity, certificate_untrusted: :unprocessable_entity,
      authorization_expired: :conflict, certificate_not_linked: :conflict, nothing_pending: :conflict,
      not_pending: :conflict, professional_cpf_missing: :conflict,
      provider_unavailable: :service_unavailable, signer_unavailable: :service_unavailable
    }.freeze

    before_action :require_digital_signature!
    before_action :require_professional

    private

    def failure(result, overrides = {})
      if result.reason == :feature_disabled
        return render(json: { error: "feature_disabled", feature: Signatures::Gate::KEY }, status: :forbidden)
      end

      render_failure(result, ERROR_STATUS.merge(overrides))
    end

    def body = params.to_unsafe_h.except("controller", "action", "id")
  end
end
```

```ruby
# app/controllers/signatures/certificates_controller.rb
# Minha conta → Assinatura digital (contrato §3). Vincular e desvincular pedem
# step-up (ADR 0016).
module Signatures
  class CertificatesController < BaseController
    rate_limit to: 10, within: 1.minute, only: :discover,
               with: -> { render json: { error: "too_many_requests" }, status: :too_many_requests }

    def show
      certificate = SignerCertificate.active.find_by(user_id: Current.user.id)
      return render(json: { error: "certificate_not_linked" }, status: :not_found) unless certificate

      render json: Json.certificate(certificate)
    end

    def discover
      result = Discover.call(user: Current.user)
      return failure(result) if result.failure?

      render json: result.payload
    end

    def link
      return require_step_up! unless reauthenticated_recently?

      result = StartLink.call(user: Current.user, provider: body["provider"], return_to: body["return_to"])
      return failure(result) if result.failure?

      render json: result.payload
    end

    def destroy
      return require_step_up! unless reauthenticated_recently?

      result = Unlink.call(user: Current.user)
      return failure(result, certificate_not_linked: :not_found) if result.failure?

      head :no_content
    end
  end
end
```

```ruby
# app/controllers/signatures/oauth_controller.rb
# Volta do PSC pelo dashboard (contrato §4): { state, code } ou { state, error }.
module Signatures
  class OauthController < BaseController
    def callback
      result = CompleteOauth.call(user: Current.user, state: body["state"], code: body["code"], error: body["error"])
      return failure(result) if result.failure?

      payload = result.payload
      render json: { purpose: payload[:purpose], result: rendered(payload[:purpose], payload[:record]),
                     return_to: payload[:return_to] }
    end

    private

    def rendered(purpose, record)
      case purpose
      when "link" then Json.certificate(record)
      when "session" then { expires_at: record.expires_at.iso8601 }
      when "batch" then record
      end
    end
  end
end
```

Em `config/routes.rb`, nas rotas de cidade (depois do bloco `/clinical_record` do 19a):

```ruby
  # Assinatura digital do prontuário (ADR 0032; contrato §3–§7).
  scope "/signature", module: :signatures, as: :signature do
    get    "certificates/current",  to: "certificates#show"
    delete "certificates/current",  to: "certificates#destroy"
    post   "certificates/discover", to: "certificates#discover"
    post   "certificates/link",     to: "certificates#link"
    post   "oauth/callback",        to: "oauth#callback"
  end
```

- [ ] **Step 5: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod19b api bundle exec rspec spec/requests/signature_certificates_spec.rb`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod19b add app/commands/signatures/discover.rb app/commands/signatures/start_link.rb app/commands/signatures/accept_certificate.rb app/commands/signatures/unlink.rb app/commands/signatures/complete_oauth.rb app/services/signatures/json.rb app/controllers/signatures/base_controller.rb app/controllers/signatures/certificates_controller.rb app/controllers/signatures/oauth_controller.rb config/routes.rb spec/requests/signature_certificates_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod19b commit -m "feat: link a cloud ICP-Brasil certificate found by the professional CPF

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 8: Sessão de assinatura do turno

**Files:**
- Create: `app/commands/signatures/{start_session,open_session,close_session}.rb`, `app/controllers/signatures/sessions_controller.rb`
- Modify: `app/commands/signatures/complete_oauth.rb`, `app/services/signatures/json.rb`, `config/routes.rb`
- Test: `spec/requests/signature_sessions_spec.rb`

**Interfaces:**
- Consumes: `AcceptCertificate`, `CompleteOauth`, `OauthStates`, `Providers`, `Psc::Client` (Tasks 4–7).
- Produces:
  - `Signatures::StartSession::LIFETIME == 12.hours`, `.call(user:, return_to:) -> Result` (ok `{ authorize_url: }`, escopo `signature_session`, `lifetime` 43200; falhas `:certificate_not_linked`, `:invalid_provider`, `:professional_cpf_missing`).
  - `Signatures::OpenSession.call(user:, provider:, client:, token:, now: Time.current) -> Result` (ok `{ record: SignatureSession }`; relê o certificado pelo `AcceptCertificate` — renovado no PSC vira o `active`; `expires_at = now + min(expires_in, 12 h)`; a sessão ativa anterior cai; evento `signature.session_opened { session_id, user_id, provider }`; falhas `:authorization_denied` (escopo errado), `:authorization_expired` (`expires_in` ≤ 0) e as do `AcceptCertificate`).
  - `Signatures::CloseSession.call(user:) -> Result` (sempre ok).
  - `Signatures::Json.session(session_or_nil) -> { active:, expires_at?, provider? }`.
  - Rotas: `POST /signature/sessions`, `GET|DELETE /signature/sessions/current`; `CompleteOauth` passa a aceitar `session`.

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/requests/signature_sessions_spec.rb
require "rails_helper"

# Contrato §4 (spec §5, sessão do turno): uma aprovação no celular abre até
# 12 h de assinatura; o certificado é relido (renovado no PSC vira o ativo);
# o token nunca sai da cidade.
RSpec.describe "Sessão de assinatura", type: :request do
  before do
    signature_city!
    stub_psc!
    stub_signer!
  end

  let(:doctor) { signer_doctor!(create_unit) }
  let(:cpf) { SignatureHelpers::DOCTOR_CPF }
  def body = JSON.parse(response.body)

  def open_session!
    post "/signature/sessions", params: { return_to: "/fila" }, as: :json
    url = body.fetch("authorize_url")
    state = URI.decode_www_form(URI(url).query).to_h.fetch("state")
    post "/signature/oauth/callback", params: { state: state, code: authorize_and_approve!(url) }, as: :json
    url
  end

  it "sem certificado vinculado: 409" do
    sign_in_as(doctor)
    post "/signature/sessions", params: { return_to: "/fila" }, as: :json
    expect([ response.status, body["error"] ]).to eq([ 409, "certificate_not_linked" ])
  end

  it "abre a sessão do turno: escopo, 12 h pedidas, sessão ativa, evento só com ids" do
    linked_certificate!(doctor, leaf: fake_psc.leaf(cpf))
    sign_in_as(doctor)
    freeze_time do
      url = open_session!
      expect(URI.decode_www_form(URI(url).query).to_h).to include("scope" => "signature_session", "lifetime" => "43200")
      expect(response).to have_http_status(:ok)
      expect(body).to eq("purpose" => "session", "result" => { "expires_at" => 12.hours.from_now.iso8601 }, "return_to" => "/fila")
      get "/signature/sessions/current"
      expect(body).to eq("active" => true, "expires_at" => 12.hours.from_now.iso8601, "provider" => "vidaas")
    end
    expect(DomainEvent.where(name: "signature.session_opened").sole.payload.keys).to match_array(%w[session_id user_id provider])
    expect(response.body).not_to include(*fake_psc.issued_tokens)
  end

  it "o menor entre o que o PSC concede e o teto de 12 h" do
    linked_certificate!(doctor, leaf: fake_psc.leaf(cpf))
    sign_in_as(doctor)
    freeze_time do
      fake_psc.forced_lifetime = 7 * 86_400
      open_session!
      expect(body.dig("result", "expires_at")).to eq(12.hours.from_now.iso8601)
      fake_psc.forced_lifetime = 3600
      open_session! # a anterior cai
      expect(body.dig("result", "expires_at")).to eq(1.hour.from_now.iso8601)
      expect(SignatureSession.where(user_id: doctor.id).pluck(:status)).to match_array(%w[revoked active])
    end
  end

  it "certificado renovado no PSC vira o ativo; o anterior, replaced" do
    old = linked_certificate!(doctor, leaf: fake_psc.leaf(cpf))
    renewed = test_pki.issue(cpf: cpf, name: "PROFISSIONAL RENOVADO")
    fake_psc.certificate_overrides[cpf] = renewed
    sign_in_as(doctor)
    open_session!
    expect(old.reload.status).to eq("replaced")
    active = SignerCertificate.active.find_by(user_id: doctor.id)
    expect(active.serial_number).to eq(renewed.serial_hex)
    expect(SignatureSession.usable_for(doctor.id).signer_certificate_id).to eq(active.id)
  end

  it "vence sozinha; encerrar derruba; GET current diz inativa" do
    linked_certificate!(doctor, leaf: fake_psc.leaf(cpf))
    sign_in_as(doctor)
    open_session!
    travel 13.hours do
      get "/signature/sessions/current"
      expect(body).to eq("active" => false)
    end
    open_session!
    delete "/signature/sessions/current"
    expect(response).to have_http_status(:no_content)
    get "/signature/sessions/current"
    expect(body).to eq("active" => false)
    expect(SignatureSession.where(user_id: doctor.id, status: "active")).to be_empty
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod19b api bundle exec rspec spec/requests/signature_sessions_spec.rb`
Expected: FAIL (`No route matches [POST] "/signature/sessions"`).

- [ ] **Step 3: Implemente**

```ruby
# app/commands/signatures/start_session.rb
# Pede ao PSC a sessão do turno (ADR 0032; spec §5): escopo signature_session,
# 12 h pedidas (o PSC pode conceder menos; teto de 7 dias do ITI para PF).
module Signatures
  module StartSession
    LIFETIME = SignatureSession::MAX_LIFETIME

    module_function

    def call(user:, return_to:)
      certificate = SignerCertificate.active.find_by(user_id: user.id)
      return Result.fail(:certificate_not_linked) unless certificate
      return Result.fail(:invalid_provider) unless Providers.configured?(certificate.provider)

      cpf = user.professional&.cpf
      return Result.fail(:professional_cpf_missing) if cpf.blank?

      issued = OauthStates.issue!(user: user, purpose: "session", provider: certificate.provider, return_to: return_to)
      url = Psc::Client.for(certificate.provider).authorize_url(
        state: issued.state, challenge: issued.challenge, scope: SignatureSession::SCOPE, login_hint: cpf,
        redirect_uri: Providers.redirect_uri(Current.city), lifetime: LIFETIME.to_i
      )
      Result.ok(authorize_url: url)
    end
  end
end
```

```ruby
# app/commands/signatures/open_session.rb
# Grava a sessão do turno (ADR 0032; spec §5; Desvio 11). O certificado é
# relido do PSC e passa pelas conferências do vínculo (renovado vira o ativo).
module Signatures
  module OpenSession
    module_function

    def call(user:, provider:, client:, token:, now: Time.current)
      return Result.fail(:authorization_denied) unless token.scope.split.include?(SignatureSession::SCOPE)
      return Result.fail(:authorization_expired) unless token.expires_in.positive?

      accepted = AcceptCertificate.call(user: user, provider: provider, entries: client.certificates(token.access_token), now: now)
      return accepted if accepted.failure?

      ApplicationRecord.transaction do
        SignatureSession.active.where(user_id: user.id).lock.each do |previous|
          previous.update!(status: previous.expires_at > now ? "revoked" : "expired")
        end
        session = SignatureSession.create!(
          user: user, signer_certificate: accepted.payload[:certificate], provider: provider,
          access_token: token.access_token, scope: SignatureSession::SCOPE, started_at: now,
          expires_at: now + [ token.expires_in.seconds, SignatureSession::MAX_LIFETIME ].min
        )
        DomainEvents.publish("signature.session_opened", session_id: session.id, user_id: user.id, provider: provider)
        Result.ok(record: session)
      end
    end
  end
end
```

```ruby
# app/commands/signatures/close_session.rb
# Encerrar a sessão do turno (contrato §4). O token é descartado (a API v0 não
# tem revogação de token).
module Signatures
  module CloseSession
    module_function

    def call(user:, now: Time.current)
      SignatureSession.active.where(user_id: user.id).update_all(status: "revoked", updated_at: now)
      Result.ok
    end
  end
end
```

```ruby
# app/controllers/signatures/sessions_controller.rb
# Sessão do turno (contrato §4).
module Signatures
  class SessionsController < BaseController
    def create
      result = StartSession.call(user: Current.user, return_to: body["return_to"])
      return failure(result) if result.failure?

      render json: result.payload
    end

    def show
      render json: Json.session(SignatureSession.usable_for(Current.user.id))
    end

    def destroy
      CloseSession.call(user: Current.user)
      head :no_content
    end
  end
end
```

Em `app/services/signatures/json.rb`, dentro do módulo:

```ruby
    def session(session)
      return { active: false } unless session

      { active: true, expires_at: session.expires_at.iso8601, provider: session.provider }
    end
```

Em `app/commands/signatures/complete_oauth.rb`, em `dispatch`, antes do `else`:

```ruby
      when "session"
        OpenSession.call(user: user, provider: row.provider, client: client, token: token, now: now)
```

Em `config/routes.rb`, dentro do `scope "/signature"`:

```ruby
    post   "sessions",         to: "sessions#create"
    get    "sessions/current", to: "sessions#show"
    delete "sessions/current", to: "sessions#destroy"
```

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod19b api bundle exec rspec spec/requests/signature_sessions_spec.rb spec/requests/signature_certificates_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod19b add app/commands/signatures/start_session.rb app/commands/signatures/open_session.rb app/commands/signatures/close_session.rb app/commands/signatures/complete_oauth.rb app/controllers/signatures/sessions_controller.rb app/services/signatures/json.rb config/routes.rb spec/requests/signature_sessions_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod19b commit -m "feat: open a per-shift signing session with the cloud provider

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
## Fatia 3 — O que se assina e como se assina (F-19.11, F-19.12)

### Task 9: JSON canônico RFC 8785 da consulta e do adendo, com a cadeia

**Files:**
- Create: `app/services/signatures/jcs.rb`, `app/services/signatures/canonical.rb`, `config/clinical/consultation-v1.json`, `config/clinical/consultation-addendum-v1.json` (cópias dos esquemas da tag `clinical-v1.0.0` do `contracts`), `spec/fixtures/clinical/canonical/{consultation-full.jcs,addendum-structured.jcs,SHA256SUMS}`, `spec/fixtures/clinical/consultation/consultation-full.json`, `spec/fixtures/clinical/consultation-addendum/addendum-structured.json` (o vetor e os exemplos de origem)
- Test: `spec/services/signatures/jcs_spec.rb`, `spec/services/signatures/canonical_spec.rb`, `spec/services/signatures/canonical_vector_spec.rb`

**Interfaces:**
- Consumes: `Consultation`, `ConsultationAddendum`, `Patient`, `TerminologyRelease`, `CityProfile`, `HealthUnit`, `ClinicalTerms.label`, `ClinicalTerms::SigtapExams.{label,release}`, `Screenings::VitalSigns.{json,bmi}`, `Consultations::Authorization.any_allowed_link` (19a e anteriores); `Signature` (Task 3); `json_schemer` (Gemfile).
- Produces:
  - `Signatures::Jcs.dump(value) -> String` (RFC 8785: chaves ordenadas por unidades UTF-16, números na forma do ECMAScript, só os escapes obrigatórios); `Signatures::Jcs::Unsupported` (NaN/Infinity, inteiro fora de ±2^53−1, tipo fora do JSON, chave repetida, UTF-8 inválido).
  - `Signatures::Canonical::CONSULTATION_SCHEMA == "rotasaude.consultation.v1"`, `::ADDENDUM_SCHEMA == "rotasaude.consultation_addendum.v1"`, `::Built = Data(:document, :json, :sha256)` (`document` Hash de chaves string; `json` String RFC 8785; `sha256` hex), `::Invalid` (mensagem só com ponteiros do esquema, nunca valores); `.consultation(consultation) -> Built` (levanta `Invalid` se não finalizada), `.addendum(addendum, chain: {}) -> Built` (com `previous_sha256`, Desvio 1: depende das assinaturas já gravadas e, no lote, dos documentos já preparados — `chain` = `Hash{[document_type, document_id] => sha256}`), `.previous_sha256(addendum, chain: {}) -> String`, `.for(document) -> Built`, `.validate!(schema_name, document)`.

- [ ] **Step 1: Copie os esquemas e o vetor do `contracts`**

```bash
W=apps/api/.claude/mod19b
mkdir -p $W/config/clinical $W/spec/fixtures/clinical/canonical $W/spec/fixtures/clinical/consultation $W/spec/fixtures/clinical/consultation-addendum
T=clinical-v1.0.0
/opt/homebrew/bin/git -C contracts show $T:clinical/consultation-v1.json > $W/config/clinical/consultation-v1.json
/opt/homebrew/bin/git -C contracts show $T:clinical/consultation-addendum-v1.json > $W/config/clinical/consultation-addendum-v1.json
for f in consultation-full.jcs addendum-structured.jcs SHA256SUMS; do
  /opt/homebrew/bin/git -C contracts show $T:clinical/examples/canonical/$f > $W/spec/fixtures/clinical/canonical/$f
done
/opt/homebrew/bin/git -C contracts show $T:clinical/examples/consultation/consultation-full.json > $W/spec/fixtures/clinical/consultation/consultation-full.json
/opt/homebrew/bin/git -C contracts show $T:clinical/examples/consultation-addendum/addendum-structured.json > $W/spec/fixtures/clinical/consultation-addendum/addendum-structured.json
(cd $W/spec/fixtures/clinical/canonical && shasum -a 256 -c SHA256SUMS)
```
Expected: os dois esquemas JSON 2020-12 e `consultation-full.jcs: OK`, `addendum-structured.jcs: OK`. Leia os dois esquemas: o construtor do Step 4 segue "Valores fixados" 1 e 2, que os transcrevem; onde a tag publicada divergir (nome, aninhamento, obrigatório), **o esquema prevalece** — ajuste o construtor e o teste e avise o coordenador.

- [ ] **Step 2: Escreva as specs que falham**

```ruby
# spec/services/signatures/jcs_spec.rb
require "rails_helper"

# RFC 8785 (JCS): os vetores do próprio RFC (§3.2.2 números, §3.2.3 ordem das
# chaves) e as recusas.
RSpec.describe Signatures::Jcs do
  it "números na forma do ECMAScript (RFC 8785 §3.2.2.3)" do
    expect(described_class.dump([ 333_333_333.33333329, 1e30, 4.50, 2e-3, 0.000000000000000000000000001 ]))
      .to eq("[333333333.3333333,1e+30,4.5,0.002,1e-27]")
    expect(described_class.dump([ 0.0, -0.0, 1e21, 1e20, 1e-7, 0.00001, 82.5, BigDecimal("36.7"), 10, -3 ]))
      .to eq("[0,0,1e+21,100000000000000000000,1e-7,0.00001,82.5,36.7,10,-3]")
  end

  it "ordena as chaves por unidades UTF-16 (RFC 8785 §3.2.3)" do
    input = { "€" => "Euro Sign", "\r" => "Carriage Return", "דּ" => "Hebrew Letter Dalet With Dagesh",
              "1" => "One", "\u{1F600}" => "Emoji: Grinning Face", "\u0080" => "Control", "ö" => "Latin Small Letter O With Diaeresis" }
    expect(JSON.parse(described_class.dump(input)).keys).to eq([ "\r", "1", "\u0080", "ö", "€", "\u{1F600}", "דּ" ])
    expect(described_class.dump({ "b" => [ true, false, nil ], "a" => { "d" => 1, "c" => "x" } })).to eq('{"a":{"c":"x","d":1},"b":[true,false,null]}')
  end

  it "escapa só o obrigatório; o resto vai literal" do
    expect(described_class.dump("aspas \" barra \\ / linha\n tab\t \u000f é ≥ 😷  "))
      .to eq("\"aspas \\\" barra \\\\ / linha\\n tab\\t \\u000f é ≥ 😷  \"")
  end

  it "recusa o que o JSON canônico não representa" do
    [ Float::NAN, Float::INFINITY, 2**53, Object.new, { a: 1, "a" => 2 }, "\xFF".dup.force_encoding("UTF-8") ].each do |value|
      expect { described_class.dump(value) }.to raise_error(described_class::Unsupported), value.inspect
    end
  end
end
```

```ruby
# spec/services/signatures/canonical_spec.rb
require "rails_helper"

# ADR 0032 (spec §7; contrato §10): o JSON canônico da consulta e do adendo na
# forma dos esquemas da tag clinical-v1.0.0 (fechados), determinístico, com a
# cadeia previous_sha256 (Desvio 1). Review Focus 5.
RSpec.describe Signatures::Canonical do
  before { Current.city = signature_city!; ciap2_release!; cid10_release!; sigtap_release! }
  after { Current.reset }

  let(:unit) { create_unit("UBS Jardim das Flores") }
  let(:doctor) { signer_doctor!(unit) }
  let(:citizen) { verified_citizen!(1, social_name: "Mariana") }

  it "consulta: cabeçalho, conteúdo clínico com rótulos, horários UTC, CPF só dígitos" do
    consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: citizen, objective: "")
    built = described_class.consultation(consultation)
    doc = built.document
    expect(doc.keys).to match_array(%w[schema city unit professional patient consultation])
    expect(doc["schema"]).to eq("rotasaude.consultation.v1")
    expect(doc["patient"]).to eq("display_name" => "Mariana", "cpf" => citizen.cpf, "birth_date" => citizen.birth_date)
    professional = doctor.professional
    expect(doc["professional"]).to eq("name" => professional.professional_name, "cpf" => SignatureHelpers::DOCTOR_CPF,
                                      "cbo_code" => "225125",
                                      "council" => { "name" => professional.council, "state" => professional.council_state,
                                                     "registration_number" => professional.registration_number })
    expect(doc["unit"]).to include("name" => "UBS Jardim das Flores")
    expect(doc["unit"]).to have_key("cnes")
    expect(doc["city"]).to have_key("ibge_code")
    body = doc["consultation"]
    expect(body).to include("id" => consultation.id, "care_type" => 5, "subjective" => "Refere sede e poliúria há dois meses",
                            "objective" => nil, "conducts" => [ 1 ], "outcome" => { "code" => "discharged" })
    expect(body["finalized_at"]).to match(/\A\d{4}-\d{2}-\d{2}T\d{2}:\d{2}:\d{2}Z\z/)
    expect(body["evaluated_problems"].sole)
      .to eq("terminology" => "ciap2", "code" => "T90", "label" => "Diabetes não insulino-dependente",
             "release" => TerminologyRelease.active.find_by(kind: "ciap2").version, "action" => "add",
             "onset_on" => "2025-08-01", "onset_precision" => "month")
    expect(body["exam_requests"].sole).to eq("sigtap_code" => "0202010503", "label" => "DOSAGEM DE HEMOGLOBINA GLICOSILADA")
    expect(body["vitals"]).to eq("systolic" => 130, "diastolic" => 85, "weight_kg" => 82.5, "height_cm" => 170, "bmi" => 28.5)
    expect(built.json).to eq(Signatures::Jcs.dump(doc))
    expect(built.sha256).to eq(Digest::SHA256.hexdigest(built.json))
  end

  it "determinístico: o mesmo documento, o mesmo sha256; texto difícil não quebra (Review Focus 5)" do
    consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: citizen,
                                           subjective: "PA ≥ 140 😷 “aspas” \r\nlinha nova", plan: "y" * 20_000)
    first = described_class.consultation(consultation)
    second = described_class.consultation(Consultation.find(consultation.id))
    expect(second.sha256).to eq(first.sha256)
    expect(first.json).to include("PA ≥ 140 😷 “aspas” \\r\\nlinha nova")
    expect(first.json.encoding).to eq(Encoding::UTF_8)
  end

  it "adendo: mesmo cabeçalho (autor do adendo), changes na forma do esquema, cadeia previous_sha256" do
    consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: citizen)
    one = Consultations::AddAddendum.call(consultation: consultation, by: doctor, reason: "primeiro adendo aqui",
                                          text: "Texto UM").payload[:addendum]
    built_one = described_class.addendum(one)
    expect(built_one.document["schema"]).to eq("rotasaude.consultation_addendum.v1")
    expect(built_one.document["professional"]).to include("cpf" => SignatureHelpers::DOCTOR_CPF, "cbo_code" => "225125")
    expect(built_one.document["addendum"])
      .to include("consultation_id" => consultation.id, "reason" => "primeiro adendo aqui", "text" => "Texto UM", "changes" => {},
                  "previous_sha256" => described_class.consultation(consultation).sha256) # nada assinado: a consulta

    # o adendo 1 assinado: o 2 aponta para ele
    signature_row!(signature_request!(one, author: doctor, status: "signed"), certificate: linked_certificate!(doctor),
                   canonical_json: built_one.json)
    two = Consultations::AddAddendum.call(consultation: consultation, by: doctor, reason: "segundo adendo aqui", text: "Texto DOIS",
                                          changes: { "conducts" => [ 1, 2 ],
                                                     "exam_requests" => [ { "sigtap_code" => "0202010503" }, { "sigtap_code" => "0202010317" } ] })
                                    .payload[:addendum]
    built_two = described_class.addendum(two)
    expect(built_two.document.dig("addendum", "previous_sha256")).to eq(built_one.sha256)
    changes = built_two.document.dig("addendum", "changes")
    expect(changes["conducts"]).to eq([ 1, 2 ])
    expect(changes["exam_requests"].map(&:keys).flatten.uniq).to match_array(%w[sigtap_code label])
    expect(described_class.addendum(two).sha256).to eq(built_two.sha256)
  end

  it "no lote: o adendo seguinte leva o hash do anterior já preparado (chain), mesmo sem assinatura gravada" do
    consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: citizen)
    one = Consultations::AddAddendum.call(consultation: consultation, by: doctor, reason: "primeiro adendo aqui", text: "UM").payload[:addendum]
    two = Consultations::AddAddendum.call(consultation: consultation, by: doctor, reason: "segundo adendo aqui", text: "DOIS").payload[:addendum]
    built_one = described_class.addendum(one)
    expect(described_class.addendum(two).document.dig("addendum", "previous_sha256"))
      .to eq(described_class.consultation(consultation).sha256) # sem chain: nada assinado
    chain = { [ "ConsultationAddendum", one.id ] => built_one.sha256 }
    expect(described_class.addendum(two, chain: chain).document.dig("addendum", "previous_sha256")).to eq(built_one.sha256)
  end

  it "rascunho não tem JSON canônico; documento fora do esquema levanta só com ponteiros" do
    draft = started_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(2))
    expect { described_class.consultation(draft) }.to raise_error(described_class::Invalid)
    doc = described_class.consultation(finalized_consultation!(unit: unit, doctor: doctor, citizen: citizen)).document
    expect { described_class.validate!(described_class::CONSULTATION_SCHEMA, doc.merge("signature" => { "x" => citizen.cpf })) }
      .to raise_error(described_class::Invalid) { |e| expect(e.message).not_to include(citizen.cpf) }
  end
end
```

```ruby
# spec/services/signatures/canonical_vector_spec.rb
require "rails_helper"

# O vetor de canonicalização do contracts (clinical/examples/canonical/): o
# gerador RFC 8785 do api reproduz os bytes e o SHA-256 publicados, e os
# exemplos de origem passam nos esquemas copiados.
RSpec.describe "Vetor RFC 8785 do contracts (clinical-v1.0.0)" do
  let(:dir) { Rails.root.join("spec/fixtures/clinical") }
  let(:sums) do
    File.read(dir.join("canonical/SHA256SUMS")).lines.to_h { |line| line.split.then { |sha, name| [ name, sha ] } }
  end

  {
    "consultation-full" => [ "consultation/consultation-full.json", Signatures::Canonical::CONSULTATION_SCHEMA ],
    "addendum-structured" => [ "consultation-addendum/addendum-structured.json", Signatures::Canonical::ADDENDUM_SCHEMA ]
  }.each do |name, (source, schema)|
    it "#{name}: mesmos bytes e mesmo SHA-256; o exemplo é válido no esquema" do
      document = JSON.parse(File.read(dir.join(source)))
      expect { Signatures::Canonical.validate!(schema, document) }.not_to raise_error
      out = Signatures::Jcs.dump(document)
      expect(out.b).to eq(File.binread(dir.join("canonical/#{name}.jcs")))
      expect(Digest::SHA256.hexdigest(out)).to eq(sums.fetch("#{name}.jcs"))
    end
  end
end
```

- [ ] **Step 3: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod19b api bundle exec rspec spec/services/signatures/jcs_spec.rb spec/services/signatures/canonical_spec.rb spec/services/signatures/canonical_vector_spec.rb`
Expected: FAIL (`uninitialized constant Signatures::Jcs`).

- [ ] **Step 4: Implemente**

```ruby
# app/services/signatures/jcs.rb
# JSON Canonicalization Scheme (RFC 8785): o que se assina em CAdES é ESTA
# serialização (ADR 0032). Chaves ordenadas por unidades UTF-16; números na
# forma do Number.prototype.toString do ECMAScript; strings com só os escapes
# obrigatórios (aspas, barra invertida e controles < 0x20).
module Signatures
  module Jcs
    class Unsupported < StandardError; end

    MAX_SAFE_INTEGER = (2**53) - 1
    ESCAPES = { "\b" => "\\b", "\t" => "\\t", "\n" => "\\n", "\f" => "\\f", "\r" => "\\r", "\"" => "\\\"",
                "\\" => "\\\\" }.freeze

    module_function

    def dump(value)
      out = +""
      write(value, out)
      out
    end

    def write(value, out)
      case value
      when Hash then object(value, out)
      when Array
        out << "["
        value.each_with_index do |item, index|
          out << "," if index.positive?
          write(item, out)
        end
        out << "]"
      when String then string(value, out)
      when true then out << "true"
      when false then out << "false"
      when nil then out << "null"
      when Integer
        raise Unsupported, "inteiro fora do intervalo seguro" if value.abs > MAX_SAFE_INTEGER

        out << value.to_s
      when Float, BigDecimal then out << number(value.to_f)
      else raise Unsupported, "tipo fora do JSON: #{value.class}"
      end
    end

    def object(hash, out)
      pairs = hash.map { |key, item| [ key!(key), item ] }
      raise Unsupported, "chave repetida" if pairs.map(&:first).uniq.size != pairs.size

      out << "{"
      pairs.sort_by { |key, _item| key.encode("UTF-16BE").unpack("n*") }.each_with_index do |(key, item), index|
        out << "," if index.positive?
        string(key, out)
        out << ":"
        write(item, out)
      end
      out << "}"
    end

    def key!(key)
      raise Unsupported, "chave que não é texto" unless key.is_a?(String) || key.is_a?(Symbol)

      key.to_s
    end

    def string(text, out)
      text = text.to_s
      raise Unsupported, "texto que não é UTF-8 válido" unless text.encoding == Encoding::UTF_8 && text.valid_encoding?

      out << "\""
      text.each_char do |char|
        escaped = ESCAPES[char]
        if escaped then out << escaped
        elsif char.ord < 0x20 then out << format("\\u%04x", char.ord)
        else out << char
        end
      end
      out << "\""
    end

    def number(float)
      raise Unsupported, "NaN ou infinito" unless float.finite?
      return "0" if float.zero?

      mantissa, exponent = float.abs.to_s.split("e")
      int, frac = mantissa.split(".")
      frac = "" if frac.nil? || frac == "0"
      point = (int == "0" ? -frac[/\A0*/].size : int.size) + exponent.to_i
      digits = (int + frac).sub(/\A0+/, "").sub(/0+\z/, "")
      (float.negative? ? "-" : "") + ecmascript(digits, point)
    end

    # Number::toString (ECMA-262 §6.1.6.1.20): k dígitos, ponto decimal em n.
    def ecmascript(digits, point)
      size = digits.size
      if size <= point && point <= 21 then digits + ("0" * (point - size))
      elsif point.positive? && point <= 21 then "#{digits[0, point]}.#{digits[point..]}"
      elsif point > -6 && point <= 0 then "0.#{'0' * -point}#{digits}"
      else
        exponent = point - 1
        mantissa = size == 1 ? digits : "#{digits[0]}.#{digits[1..]}"
        "#{mantissa}e#{exponent.negative? ? '-' : '+'}#{exponent.abs}"
      end
    end
    private_class_method :write, :object, :key!, :string, :number, :ecmascript
  end
end
```

```ruby
# app/services/signatures/canonical.rb
# JSON canônico da consulta e do adendo (ADR 0032; spec §7; contrato §10;
# "Valores fixados" 1 e 2): o registro estruturado assinado em CAdES (fonte de
# verdade), na forma EXATA dos esquemas da tag clinical-v1.0.0 do contracts
# (cópias em config/clinical/*.json; mude os dois juntos). Fechados: o que não
# está no esquema não entra. Texto vazio = null; horários UTC com Z; CPF, CNES,
# IBGE e CBO só dígitos. Gerado de documentos imutáveis → determinístico.
require "json_schemer"

module Signatures
  module Canonical
    CONSULTATION_SCHEMA = "rotasaude.consultation.v1".freeze
    ADDENDUM_SCHEMA = "rotasaude.consultation_addendum.v1".freeze
    SCHEMA_FILES = { CONSULTATION_SCHEMA => "config/clinical/consultation-v1.json",
                     ADDENDUM_SCHEMA => "config/clinical/consultation-addendum-v1.json" }.freeze

    class Invalid < StandardError; end

    Built = Data.define(:document, :json, :sha256) do
      def inspect = "#<Signatures::Canonical::Built #{sha256}>"
    end

    module_function

    def for(document)
      case document
      when Consultation then consultation(document)
      when ConsultationAddendum then addendum(document)
      else raise ArgumentError, "documento fora do catálogo"
      end
    end

    def consultation(consultation)
      raise Invalid, "not_finalized" unless consultation.finalized?

      own = ->(relation) { relation.where(addendum_id: nil).order(:created_at, :id) }
      body = {
        "id" => consultation.id, "started_at" => time(consultation.started_at),
        "finalized_at" => time(consultation.finalized_at), "care_type" => consultation.care_type,
        "subjective" => text(consultation.subjective), "objective" => text(consultation.objective),
        "assessment" => text(consultation.assessment), "plan" => text(consultation.plan), "vitals" => vitals(consultation),
        "evaluated_problems" => own.call(consultation.problem_items).map do |row|
          problem(row.terminology, row.code, row.terminology_release_id, row.action, row.onset_on, row.onset_precision)
        end,
        "conducts" => own.call(consultation.conducts).pluck(:code),
        "exam_requests" => own.call(consultation.exam_requests).map { |row| exam(row.sigtap_code, row.sigtap_competence, row.cid10_justification) },
        "outcome" => outcome(consultation.attendance)
      }
      built(CONSULTATION_SCHEMA, header(consultation, consultation.author_user, consultation.cbo_code)
                                   .merge("schema" => CONSULTATION_SCHEMA, "consultation" => body))
    end

    def addendum(addendum, chain: {})
      consultation = addendum.consultation
      body = { "consultation_id" => consultation.id, "id" => addendum.id, "created_at" => time(addendum.created_at),
               "reason" => addendum.reason, "text" => addendum.text, "changes" => changes(addendum),
               "previous_sha256" => previous_sha256(addendum, chain: chain) }
      built(ADDENDUM_SCHEMA, header(consultation, addendum.author_user, addendum_cbo(addendum))
                               .merge("schema" => ADDENDUM_SCHEMA, "addendum" => body))
    end

    # Desvio 1 (esquema do contracts): o canonical_sha256 do documento ASSINADO
    # anterior da mesma consulta, na ordem de criação; os já preparados no MESMO
    # lote (chain) contam como assinados (se um falhar, os seguintes da consulta
    # não são assinados — Signing); nenhum → o sha256 do JSON canônico da consulta.
    def previous_sha256(addendum, chain: {})
      consultation = addendum.consultation
      earlier = consultation.addenda
                            .where("(consultation_addenda.created_at, consultation_addenda.id) < (?, ?)", addendum.created_at, addendum.id)
                            .order(created_at: :desc, id: :desc).pluck(:id)
      signed = Signature.where(document_type: "ConsultationAddendum", document_id: earlier).pluck(:document_id, :canonical_sha256).to_h
      earlier.each do |id|
        sha = chain[[ "ConsultationAddendum", id ]] || signed[id]
        return sha if sha
      end

      chain[[ "Consultation", consultation.id ]] ||
        Signature.where(document_type: "Consultation", document_id: consultation.id).pick(:canonical_sha256) ||
        consultation(consultation).sha256
    end

    def validate!(schema_name, document)
      errors = schema(schema_name).validate(document).first(5)
      return document if errors.empty?

      raise Invalid, "documento fora do esquema #{schema_name}: #{errors.map { |e| "#{e['data_pointer']} #{e['type']}" }.join('; ')}"
    end

    def built(schema_name, document)
      validate!(schema_name, document)
      json = Jcs.dump(document)
      Built.new(document: document, json: json, sha256: Digest::SHA256.hexdigest(json))
    end

    def header(consultation, author, cbo_code)
      profile = CityProfile.current
      unit = consultation.attendance.health_unit
      professional = author.professional
      patient = consultation.patient
      council = professional && { "name" => professional.council, "state" => professional.council_state,
                                  "registration_number" => professional.registration_number }
      { "city" => { "ibge_code" => profile&.ibge_code.presence, "name" => profile&.name.presence || Current.city&.name },
        "unit" => { "cnes" => unit.cnes.presence, "name" => unit.name },
        "professional" => { "name" => professional&.professional_name, "cpf" => professional&.cpf, "cbo_code" => cbo_code,
                            "council" => council },
        "patient" => { "display_name" => patient.display_name, "cpf" => patient.cpf, "birth_date" => patient.birth_date.presence } }
    end

    # "Valores fixados" 2: o esquema exige o CBO de quem assina.
    def addendum_cbo(addendum)
      consultation = addendum.consultation
      return consultation.cbo_code if addendum.author_user_id == consultation.author_user_id

      _status, link = Consultations::Authorization.any_allowed_link(user: addendum.author_user)
      link&.cbo_code
    end

    # Itens do adendo (contrato do 19a §9), na forma do esquema; lista vazia sai.
    def changes(addendum)
      raw = addendum.changes.to_h
      result = {}
      if (items = Array(raw["evaluated_problems"])).any?
        result["evaluated_problems"] = items.map do |item|
          problem(item["terminology"], item["code"], item["release_id"], item["action"], item["onset_on"], item["onset_precision"])
        end
      end
      result["conducts"] = Array(raw["conducts"]).map(&:to_i) if Array(raw["conducts"]).any?
      if (exams = Array(raw["exam_requests"])).any?
        result["exam_requests"] = exams.map do |item|
          exam(item["sigtap_code"], item["sigtap_competence"] || competence_on(addendum.created_at), item["cid10_justification"])
        end
      end
      result
    end

    def problem(terminology, code, release_id, action, onset_on, onset_precision)
      { "terminology" => terminology, "code" => code, "label" => ClinicalTerms.label(terminology, code, release_id),
        "release" => release_version(release_id), "action" => action,
        "onset_on" => onset_on.respond_to?(:iso8601) ? onset_on.iso8601 : onset_on.presence,
        "onset_precision" => onset_precision.presence }.compact
    end

    def exam(code, competence, cid10)
      { "sigtap_code" => code, "label" => ClinicalTerms::SigtapExams.label(code, competence),
        "cid10_justification" => cid10.presence }.compact
    end

    def vitals(consultation)
      Screenings::VitalSigns.json(consultation.vitals).merge("bmi" => Screenings::VitalSigns.bmi(consultation.vitals))
                            .compact.transform_keys(&:to_s)
    end

    def outcome(attendance)
      { "code" => attendance.outcome, "referral_unit_cnes" => attendance.referral_unit&.cnes.presence,
        "referral_note" => text(attendance.referral_note) }.compact
    end

    def release_version(id) = id && TerminologyRelease.where(id: id).pick(:version)
    def competence_on(time) = ClinicalTerms::SigtapExams.release(on: time.to_date)&.version
    def text(value) = value.presence
    def time(value) = value&.utc&.iso8601

    def schema(name)
      @schemas ||= {}
      @schemas[name] ||= JSONSchemer.schema(Rails.root.join(SCHEMA_FILES.fetch(name)))
    end
    private_class_method :built, :header, :addendum_cbo, :changes, :problem, :exam, :vitals, :outcome, :release_version,
                         :competence_on, :text, :time, :schema
  end
end
```

- [ ] **Step 5: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod19b api bundle exec rspec spec/services/signatures/jcs_spec.rb spec/services/signatures/canonical_spec.rb spec/services/signatures/canonical_vector_spec.rb`
Expected: PASS — o vetor do `contracts` sai byte a byte. Falha de esquema por nome/aninhamento → o esquema prevalece (Step 1).

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod19b add app/services/signatures/jcs.rb app/services/signatures/canonical.rb config/clinical/consultation-v1.json config/clinical/consultation-addendum-v1.json spec/fixtures/clinical/canonical/consultation-full.jcs spec/fixtures/clinical/canonical/addendum-structured.jcs spec/fixtures/clinical/canonical/SHA256SUMS spec/fixtures/clinical/consultation/consultation-full.json spec/fixtures/clinical/consultation-addendum/addendum-structured.json spec/services/signatures/jcs_spec.rb spec/services/signatures/canonical_spec.rb spec/services/signatures/canonical_vector_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod19b commit -m "feat: build the RFC 8785 canonical JSON of consultations and addenda

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 10: PDF assinável — rodapé NGS2, PDF do adendo e seção de assinaturas

**Files:**
- Create: `app/services/signatures/pdf_footer.rb`
- Modify: `app/services/consultations/print.rb`
- Test: `spec/services/consultations/print_signed_spec.rb` (e a do 19a, `spec/services/consultations/print_spec.rb`, continua passando sem mudança)

**Interfaces:**
- Consumes: `Consultations::Print` (19a: `call`, `safe`, `NotPrintable` e os blocos privados), `CitizenIdentity::Cpf.mask`.
- Produces:
  - `Signatures::PdfFooter = Data(:signer_name, :signer_cpf, :signed_at)` com `#text` ("Valores fixados" 6).
  - `Consultations::Print.call(consultation, footer: nil, addenda: true, report: nil) -> String` — `footer` (um `PdfFooter`): rodapé em toda página e **sem** o espaço de assinatura à mão; `addenda: false`: só a consulta; `report` (responde a `#lines -> Array<String>` e `#hand_signature? -> bool`): seção "Assinaturas" e espaço à mão só se `hand_signature?`. Sem nenhum dos três: o impresso do 19a, igual.
  - `Consultations::Print.addendum(addendum, footer:) -> String` (PDF só do adendo: cabeçalho da unidade, referência à consulta, paciente, autor, motivo, texto, mudanças).

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/services/consultations/print_signed_spec.rb
require "rails_helper"

# ADR 0032 (spec §7; NGS2.06.05–07): o PDF que vai ao PAdES leva o rodapé
# padronizado em TODA página e não tem espaço de assinatura à mão; o adendo tem
# PDF próprio; o impresso com adendos lista o estado de cada parte. Review
# Focus 5: texto que a fonte não tem nunca derruba.
RSpec.describe "Impresso assinável" do
  before { Current.city = signature_city!; ciap2_release!; cid10_release!; sigtap_release! }
  after { Current.reset }

  let(:unit) { create_unit("UBS Jardim das Flores") }
  let(:doctor) { signer_doctor!(unit) }
  let(:citizen) { verified_citizen!(1, social_name: "Mariana") }
  let(:footer) { Signatures::PdfFooter.new(signer_name: "MARIA ≥ SOUZA", signer_cpf: SignatureHelpers::DOCTOR_CPF, signed_at: Time.utc(2026, 10, 8, 13, 45)) }

  def pages(bytes) = PDF::Reader.new(StringIO.new(bytes)).pages.map { |page| page.text.gsub(/\s+/, " ") }

  it "rodapé NGS2 em toda página, sem assinatura à mão, com texto longo e caracteres fora da fonte" do
    consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: citizen, plan: "linha de plano " * 2_000)
    texts = pages(Consultations::Print.call(consultation, footer: footer, addenda: false))
    expect(texts.size).to be > 1
    texts.each do |text|
      expect(text).to include("assinado digitalmente por MARIA ? SOUZA", "***.982.247-**", "08/10/2026 13:45 UTC", "AD-RB",
                              "validar.iti.gov.br")
    end
    expect(texts.join).not_to include("Assinatura e carimbo")
  end

  it "addenda: false deixa o adendo de fora; o PDF do adendo é só dele" do
    consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: citizen)
    addendum = Consultations::AddAddendum.call(consultation: consultation, by: doctor, reason: "exame adicional pedido",
                                               text: "Texto do ADENDO").payload[:addendum]
    expect(pages(Consultations::Print.call(consultation.reload, footer: footer, addenda: false)).join).not_to include("Texto do ADENDO")
    text = pages(Consultations::Print.addendum(addendum, footer: footer)).join("\n")
    expect(text).to include("Adendo à consulta de", "Mariana", "exame adicional pedido", "Texto do ADENDO",
                            "assinado digitalmente por", "UBS Jardim das Flores")
    expect(text).not_to include("Refere sede e poliúria")
  end

  it "seção de assinaturas: espaço à mão só quando alguma parte não é digital" do
    consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: citizen)
    report = Struct.new(:lines, :hand_signature?).new([ "Consulta: assinada digitalmente — válida" ], false)
    text = pages(Consultations::Print.call(consultation, report: report)).join
    expect(text).to include("Assinaturas", "Consulta: assinada digitalmente")
    expect(text).not_to include("Assinatura e carimbo")
    report = Struct.new(:lines, :hand_signature?).new([ "Adendo de 08/10/2026: sem assinatura digital" ], true)
    expect(pages(Consultations::Print.call(consultation, report: report)).join).to include("Assinatura e carimbo")
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod19b api bundle exec rspec spec/services/consultations/print_signed_spec.rb`
Expected: FAIL (`uninitialized constant Signatures::PdfFooter`).

- [ ] **Step 3: Implemente**

```ruby
# app/services/signatures/pdf_footer.rb
# Rodapé padronizado do PDF assinado (NGS2.06.05–07; "Valores fixados" 6):
# quem assinou (nome do certificado, CPF mascarado), quando (UTC), política e
# onde verificar.
module Signatures
  PdfFooter = Data.define(:signer_name, :signer_cpf, :signed_at) do
    def text
      "Documento assinado digitalmente por #{signer_name} (CPF #{CitizenIdentity::Cpf.mask(signer_cpf)}) em " \
        "#{signed_at.utc.strftime('%d/%m/%Y %H:%M')} UTC — ICP-Brasil, política AD-RB. Verifique em https://validar.iti.gov.br"
    end

    def inspect = "#<Signatures::PdfFooter>"
  end
end
```

Em `app/services/consultations/print.rb`, troque `call` por:

```ruby
    # footer: PDF que vai ao PAdES (rodapé em toda página, sem assinatura à
    # mão). report: impresso com a seção "Assinaturas" (19b). Sem os dois: o
    # impresso do 19a.
    def call(consultation, footer: nil, addenda: true, report: nil)
      patient = consultation.patient
      raise NotPrintable, "not_finalized" unless consultation.finalized?
      raise NotPrintable, "patient_name_missing" if patient.full_name.blank?

      pdf = document(footer)
      header(pdf, consultation)
      patient_block(pdf, patient)
      professional_block(pdf, consultation)
      record(pdf, consultation)
      addenda(pdf, consultation) if addenda
      signatures(pdf, report) if report
      signature(pdf) if footer.nil? && (report.nil? || report.hand_signature?)
      pdf.render
    end

    # ADR 0032: o adendo tem documento próprio (JSON canônico e PDF).
    def addendum(addendum, footer:)
      consultation = addendum.consultation
      patient = consultation.patient
      raise NotPrintable, "patient_name_missing" if patient.full_name.blank?

      pdf = document(footer)
      header(pdf, consultation)
      pdf.move_down 6
      pdf.text safe("Adendo à consulta de #{consultation.finalized_at.in_time_zone.strftime('%d/%m/%Y %H:%M')}"),
               style: :bold, size: 12
      line(pdf, "Registrado em", addendum.created_at.in_time_zone.strftime("%d/%m/%Y %H:%M"))
      patient_block(pdf, patient)
      title(pdf, "Profissional")
      line(pdf, "Nome", addendum.author_user.professional&.professional_name || addendum.author_user.email_address)
      title(pdf, "Motivo")
      pdf.text safe(addendum.reason)
      title(pdf, "Texto")
      pdf.text safe(addendum.text)
      if addendum.changes.present?
        title(pdf, "Mudanças estruturadas")
        pdf.text safe(JSON.generate(addendum.changes)), size: 8
      end
      pdf.render
    end
```

e acrescente, antes do `private_class_method`:

```ruby
    def document(footer)
      pdf = Prawn::Document.new(page_size: "A4", margin: footer ? [ 40, 40, 80, 40 ] : 40,
                                info: { Title: "Registro de consulta", Producer: "Rota Saúde" })
      pdf.font_size(10)
      stamp_footer(pdf, footer) if footer
      pdf
    end

    # NGS2.06.05: o rodapé sai em toda página (repeater do Prawn, aplicado na
    # renderização a todas as páginas, inclusive as criadas depois).
    def stamp_footer(pdf, footer)
      width = pdf.bounds.width
      pdf.repeat(:all) do
        pdf.canvas do
          pdf.bounding_box([ 40, 70 ], width: width, height: 56) do
            pdf.stroke_horizontal_rule
            pdf.move_down 4
            pdf.text safe(footer.text), size: 7
          end
        end
      end
    end

    def signatures(pdf, report)
      title(pdf, "Assinaturas")
      report.lines.each { |text| pdf.text safe(text) }
    end
```

e inclua `:document, :stamp_footer, :signatures` na lista do `private_class_method`.

- [ ] **Step 4: Rode e veja passar (e a spec do 19a, sem mudança)**

Run: `docker compose exec -T -w /rails/.claude/mod19b api bundle exec rspec spec/services/consultations/print_signed_spec.rb spec/services/consultations/print_spec.rb spec/requests/consultation_print_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod19b add app/services/signatures/pdf_footer.rb app/services/consultations/print.rb spec/services/consultations/print_signed_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod19b commit -m "feat: render signable consultation and addendum PDFs with the NGS2 footer

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
### Task 11: Assinar — documentos, PSC, `signer`, gravação; o job com tentativas e lock

**Files:**
- Create: `app/services/signatures/{documents,signing,certificate_rules}.rb`, `app/commands/signatures/{sign_pending,park,to_paper}.rb`, `app/jobs/signatures/sign_job.rb`
- Test: `spec/commands/signatures/sign_pending_spec.rb`, `spec/jobs/signatures/sign_job_spec.rb`, `spec/integration/signing_flow_spec.rb` (`:signer`)

**Interfaces:**
- Consumes: `Canonical` (Task 9), `Consultations::Print.call(..., footer:, addenda: false)`/`.addendum` e `PdfFooter` (Task 10), `Psc::Client#sign` (Task 4), `Signer.client` (Task 5), `SignerCertificate#info`, `SignatureSession.usable_for/.lapsed?` (Task 3), `Signatures::Gate` (Task 1).
- Produces:
  - `Signatures::Documents::Prepared = Data(:request, :canonical, :pdf)`, `.for(request, info:, signed_at:, chain: {}) -> Prepared` (levanta `Canonical::Invalid` sem documento).
  - `Signatures::Signing::Outcome = Data(:signed, :failed)` (`signed`: `Array<Signature>`; `failed`: `Array<[SignatureRequest, reason]>`); `.call(requests:, access_token:, certificate:, now: Time.current, signer: Signer.client) -> Outcome` — chame dentro da transação da cidade com os pedidos travados; em ordem cronológica de criação dos documentos (consulta pela finalização, adendo pela criação), prepara CAdES (JSON canônico, com a cadeia dos já preparados no lote) e PAdES (PDF com rodapé) de cada pedido — se um documento falha, os seguintes da mesma consulta ficam em `failed` com o mesmo motivo e não são assinados —, pede ao PSC **uma** assinatura RAW de todos os hashes (`"<request_id>:cades"`, `"<request_id>:pades"`), monta, verifica e grava cada um num savepoint (assinatura + pedido `signed` + evento `signature.signed { signature_id, request_id, document_type, document_id }`). Levanta `Psc::Error`/`Signer::Unavailable` (este só **antes** da chamada ao PSC) para quem chama; motivos por pedido: `verification_failed`, `signer_unavailable` (depois do PSC), `certificate_revoked`, `certificate_cpf_mismatch`.
  - `Signatures::CertificateRules.reason(certificate, now:) -> String|nil` (`certificate_expired` — e marca o certificado `expired` —, `certificate_revoked`, `certificate_cpf_mismatch`).
  - `Signatures::Park.call(request, reason, transient: false, now:)` (grava o motivo; `transient` soma uma tentativa; evento `signature.failed { request_id, reason_code }`).
  - `Signatures::ToPaper.call(request, reason_code:, note: nil, now:)` (`returned_to_paper` + evento `signature.returned_to_paper { request_id, reason_code }`).
  - `Signatures::SignPending::MAX_ATTEMPTS == 3`, `.call(request_id:, now: Time.current, signer: Signer.client) -> :signed | :pending | :retry | :returned_to_paper | :skipped` (trava com `FOR UPDATE SKIP LOCKED`; pedido travado por outro ou já resolvido → `:skipped`).
  - `Signatures::SignJob` (`CityScopedJob`), `::BACKOFF == [30.seconds, 2.minutes]`, `#perform(city_slug:, request_id:)` (em `:retry` reenfileira a si mesmo com a espera da tentativa).

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/commands/signatures/sign_pending_spec.rb
require "rails_helper"

# ADR 0032 (spec §5, finalização e regras; Review Focus 3): com sessão ativa,
# o job assina o JSON canônico (CAdES) e o PDF (PAdES) com UMA chamada ao PSC;
# sem sessão, certificado vencido/revogado/de outro CPF ou PSC/signer fora do
# ar, o pedido fica pending com o motivo (3 tentativas para o passageiro).
RSpec.describe Signatures::SignPending do
  before do
    Current.city = signature_city!
    ciap2_release!; cid10_release!; sigtap_release!
    stub_psc!
    @signer = stub_signer!
  end
  after { Current.reset }

  let(:unit) { create_unit }
  let(:doctor) { signer_doctor!(unit) }
  let(:cpf) { SignatureHelpers::DOCTOR_CPF }
  let(:certificate) { linked_certificate!(doctor, leaf: fake_psc.leaf(cpf)) }
  let(:consultation) { finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1)) }

  def session!(**over) = signature_session!(doctor, certificate: certificate, token: fake_psc.token_for!(cpf: cpf), **over)
  def sign(request) = ApplicationRecord.transaction { described_class.call(request_id: request.id) }
  def psc_signature_calls = fake_psc.log.count { |_verb, path| path == "/v0/oauth/signature" }

  it "assina a consulta: CAdES do JSON canônico e PAdES do PDF numa chamada ao PSC" do
    session!
    request = signature_request!(consultation, author: doctor)
    expect(sign(request)).to eq(:signed)
    request.reload
    signature = request.signature
    canonical = Signatures::Canonical.consultation(consultation)
    expect(request).to have_attributes(status: "signed", reason_code: nil)
    expect(request.resolved_at).to be_present
    expect(signature.canonical_json).to eq(canonical.json)
    expect(signature.canonical_sha256).to eq(canonical.sha256)
    expect(signature.signed_pdf_bytes).to start_with("%PDF")
    expect(signature.pdf_sha256).to eq(Digest::SHA256.hexdigest(signature.signed_pdf_bytes))
    expect(signature).to have_attributes(policy: "AD-RB", policy_oid: FakeSigner::POLICY_OID, last_verification: "valid",
                                         signer_cpf: cpf, signer_certificate_id: certificate.id)
    expect(signature.material.keys).to match_array(%w[cades pades])
    expect(psc_signature_calls).to eq(1)
    expect(DomainEvent.where(name: "signature.signed").sole.payload)
      .to eq("signature_id" => signature.id, "request_id" => request.id, "document_type" => "consultation",
             "document_id" => consultation.id)
  end

  it "assina o adendo com o JSON do adendo (cadeia) e o PDF só dele" do
    session!
    addendum = Consultations::AddAddendum.call(consultation: consultation, by: doctor, reason: "exame adicional pedido",
                                               text: "Pedido creatinina").payload[:addendum]
    request = signature_request!(addendum, author: doctor)
    expect(sign(request)).to eq(:signed)
    expect(JSON.parse(request.reload.signature.canonical_json).dig("addendum", "previous_sha256"))
      .to eq(Signatures::Canonical.consultation(consultation).sha256)
  end

  it "sem sessão: no_session; sessão vencida: session_expired; certificado vencido: certificate_expired" do
    request = signature_request!(consultation, author: doctor)
    certificate
    expect(sign(request)).to eq(:pending)
    expect(request.reload.reason_code).to eq("no_session")
    session!(started_at: 13.hours.ago, expires_at: 1.hour.ago)
    expect(sign(request)).to eq(:pending)
    expect(request.reload.reason_code).to eq("session_expired")

    other = signature_request!(finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(2)), author: doctor)
    certificate.update!(not_after: 1.minute.ago)
    expect(sign(other)).to eq(:pending)
    expect([ other.reload.reason_code, certificate.reload.status ]).to eq(%w[certificate_expired expired])
    expect(psc_signature_calls).to eq(0)
    expect(DomainEvent.where(name: "signature.failed").pluck(:payload).map { |payload| payload.keys.sort }.uniq).to eq([ %w[reason_code request_id] ])
  end

  it "tentativas: PSC e signer fora do ar → retry, retry, pending com o motivo (Review Focus 3)" do
    session!
    request = signature_request!(consultation, author: doctor)
    fake_psc.failures.push(503, :refused, 503)
    expect([ sign(request), sign(request), sign(request) ]).to eq(%i[retry retry pending])
    expect(request.reload).to have_attributes(status: "pending", reason_code: "provider_unavailable", attempts: 3)

    other = signature_request!(finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(2)), author: doctor)
    @signer.unavailable = true
    expect(sign(other)).to eq(:retry)
    expect(other.reload.reason_code).to eq("signer_unavailable")
    @signer.unavailable = false
    expect(sign(other)).to eq(:signed)
  end

  it "token recusado pelo PSC: a sessão cai e o pedido fica session_expired" do
    session = session!
    request = signature_request!(consultation, author: doctor)
    fake_psc.expire_tokens!
    expect(sign(request)).to eq(:pending)
    expect([ request.reload.reason_code, session.reload.status ]).to eq(%w[session_expired expired])
  end

  it "revogado na verificação: certificate_revoked, nada gravado, certificado marcado" do
    session!
    request = signature_request!(consultation, author: doctor)
    @signer.revoked_serials << certificate.serial_number
    expect(sign(request)).to eq(:pending)
    expect([ request.reload.reason_code, certificate.reload.status, Signature.count ]).to eq([ "certificate_revoked", "revoked", 0 ])
  end

  it "CPF do profissional diferente do certificado (NGS2.01.02): certificate_cpf_mismatch" do
    session!
    request = signature_request!(consultation, author: doctor)
    doctor.professional.update!(cpf: SignatureHelpers::OTHER_CPF)
    expect(sign(request)).to eq(:pending)
    expect(request.reload.reason_code).to eq("certificate_cpf_mismatch")
  end

  it "interruptor desligado: volta ao papel com feature_disabled; resolvido: skipped" do
    session!
    request = signature_request!(consultation, author: doctor)
    signature_city!(enabled: false)
    expect(sign(request)).to eq(:returned_to_paper)
    expect(request.reload).to have_attributes(status: "returned_to_paper", reason_code: "feature_disabled")
    expect(DomainEvent.where(name: "signature.returned_to_paper").sole.payload).to eq("request_id" => request.id, "reason_code" => "feature_disabled")
    expect(sign(request)).to eq(:skipped)
  end
end
```

```ruby
# spec/jobs/signatures/sign_job_spec.rb
require "rails_helper"

# O job reenfileira a si mesmo na falha passageira, com espera crescente.
RSpec.describe Signatures::SignJob do
  include ActiveJob::TestHelper

  before do
    Current.city = signature_city!
    stub_psc!
    stub_signer!
  end
  after { Current.reset }

  it "falha passageira: reenfileira com 30 s e depois 2 min; sucesso: não reenfileira" do
    doctor = signer_doctor!(create_unit)
    certificate = linked_certificate!(doctor, leaf: fake_psc.leaf(SignatureHelpers::DOCTOR_CPF))
    signature_session!(doctor, certificate: certificate, token: fake_psc.token_for!(cpf: SignatureHelpers::DOCTOR_CPF))
    request = signature_request!(author: doctor) # documento inexistente: verification_failed (passageiro)
    slug = Current.city.slug
    freeze_time do
      expect { described_class.perform_now(city_slug: slug, request_id: request.id) }
        .to have_enqueued_job(described_class).with(city_slug: slug, request_id: request.id).at(30.seconds.from_now)
      expect { described_class.perform_now(city_slug: slug, request_id: request.id) }
        .to have_enqueued_job(described_class).at(2.minutes.from_now)
      expect { described_class.perform_now(city_slug: slug, request_id: request.id) }.not_to have_enqueued_job(described_class)
    end
    expect(request.reload).to have_attributes(attempts: 3, reason_code: "verification_failed", status: "pending")
  end
end
```

```ruby
# spec/integration/signing_flow_spec.rb
require "rails_helper"

# O fluxo inteiro contra o SIGNER REAL (compose) e o PSC falso: consulta
# finalizada → JSON canônico + PDF → prepare → RAW no PSC → assemble → verify →
# gravado; e o que foi gravado verifica de novo no signer.
RSpec.describe "Assinatura de ponta a ponta (signer real)", :signer do
  before do
    Current.city = signature_city!
    ciap2_release!; cid10_release!; sigtap_release!
    stub_psc!
  end
  after { Current.reset }

  it "assina a consulta e o adendo; CAdES e PAdES verificam no signer" do
    unit = create_unit
    doctor = signer_doctor!(unit)
    cpf = SignatureHelpers::DOCTOR_CPF
    certificate = linked_certificate!(doctor, leaf: fake_psc.leaf(cpf))
    signature_session!(doctor, certificate: certificate, token: fake_psc.token_for!(cpf: cpf))
    consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1))
    addendum = Consultations::AddAddendum.call(consultation: consultation, by: doctor, reason: "exame adicional pedido",
                                               text: "Texto do adendo").payload[:addendum]
    [ consultation, addendum ].each do |document|
      request = signature_request!(document, author: doctor)
      expect(ApplicationRecord.transaction { Signatures::SignPending.call(request_id: request.id) }).to eq(:signed)
      signature = request.reload.signature
      client = Signatures::Signer.client
      expect(client.verify(kind: "cades", signature: signature.cades_bytes, document: signature.canonical_json).status).to eq("valid")
      expect(client.verify(kind: "pades", signature: signature.signed_pdf_bytes).status).to eq("valid")
    end
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod19b api bundle exec rspec spec/commands/signatures/sign_pending_spec.rb spec/jobs/signatures/sign_job_spec.rb`
Expected: FAIL (`uninitialized constant Signatures::SignPending`).

- [ ] **Step 3: Implemente**

```ruby
# app/services/signatures/documents.rb
# Os dois documentos de cada pedido (ADR 0032; spec §7): o JSON canônico (vai
# ao CAdES destacado) e o PDF com o rodapé NGS2 (vai ao PAdES). O rodapé leva o
# nome e o CPF do CERTIFICADO, que é o que se assina.
module Signatures
  module Documents
    Prepared = Data.define(:request, :canonical, :pdf) do
      def inspect = "#<Signatures::Documents::Prepared #{request.id}>"
    end

    module_function

    # chain: os hashes canônicos dos documentos já preparados NESTE lote
    # (Canonical.previous_sha256 os conta como assinados).
    def for(request, info:, signed_at:, chain: {})
      document = request.document
      raise Canonical::Invalid, "documento não encontrado" unless document

      footer = PdfFooter.new(signer_name: info.holder_name, signer_cpf: info.cpf, signed_at: signed_at)
      case document
      when Consultation
        Prepared.new(request: request, canonical: Canonical.consultation(document),
                     pdf: Consultations::Print.call(document, footer: footer, addenda: false))
      when ConsultationAddendum
        Prepared.new(request: request, canonical: Canonical.addendum(document, chain: chain),
                     pdf: Consultations::Print.addendum(document, footer: footer))
      end
    end
  end
end
```

```ruby
# app/services/signatures/certificate_rules.rb
# Regras do certificado a CADA uso (ADR 0032; spec §5; NGS2.01.02): dentro da
# validade, não revogado, do CPF do profissional AGORA. nil = pode assinar.
module Signatures
  module CertificateRules
    module_function

    def reason(certificate, now: Time.current)
      if certificate.not_after <= now
        certificate.update!(status: "expired")
        return "certificate_expired"
      end
      return "certificate_revoked" if certificate.status == "revoked"

      cpf = certificate.user.professional&.cpf
      return "certificate_cpf_mismatch" unless cpf.present? && certificate.info.cpf == cpf

      nil
    end
  end
end
```

```ruby
# app/services/signatures/signing.rb
# O ato de assinar (ADR 0032; spec §5, §7): para cada pedido, prepara o CAdES
# do JSON canônico e o PAdES do PDF no signer; pede ao PSC UMA assinatura RAW
# de todos os hashes (sessão do turno ou token do lote); monta, verifica (antes
# de gravar, NGS2.03.01) e grava num savepoint por pedido. Chame dentro da
# transação da cidade, com os pedidos travados. Erros do PSC e indisponibilidade
# do signer sobem para quem chama (job ou lote decidem a tentativa).
module Signatures
  module Signing
    KINDS = Signer::KINDS

    Outcome = Data.define(:signed, :failed)

    module_function

    def call(requests:, access_token:, certificate:, now: Time.current, signer: Signer.client)
      info = certificate.info
      ready = []
      failed = []
      chain = {}   # [document_type, document_id] => canonical_sha256 dos já preparados neste lote
      blocked = {} # consultation_id => motivo da primeira falha (os seguintes da mesma consulta não assinam)
      # Ordem cronológica de criação dos documentos: a consulta, depois os adendos.
      chronological(requests).each do |request|
        next failed << [ request, blocked[request.consultation_id] ] if blocked.key?(request.consultation_id)

        docs = Documents.for(request, info: info, signed_at: now, chain: chain)
        prepared = KINDS.to_h do |kind|
          [ kind, signer.prepare(kind: kind, document: kind == "cades" ? docs.canonical.json : docs.pdf, certificate_der: certificate.der) ]
        end
        chain[[ request.document_type, request.document_id ]] = docs.canonical.sha256
        ready << [ request, docs, prepared ]
      rescue Canonical::Invalid, Consultations::Print::NotPrintable, Signer::Rejected
        blocked[request.consultation_id] ||= "verification_failed"
        failed << [ request, "verification_failed" ]
      end
      return Outcome.new(signed: [], failed: failed) if ready.empty?

      digests = ready.each_with_object({}) do |(request, _docs, prepared), map|
        KINDS.each { |kind| map["#{request.id}:#{kind}"] = prepared[kind].digest }
      end
      raw = Psc::Client.for(certificate.provider).sign(access_token: access_token, certificate_alias: certificate.certificate_alias,
                                                       digests: digests)
      signed = []
      ready.each do |request, docs, prepared|
        next failed << [ request, blocked[request.consultation_id] ] if blocked.key?(request.consultation_id)

        result = store(request, docs, prepared, raw, certificate, info, signer, now)
        if result.is_a?(Signature)
          signed << result
        else
          blocked[request.consultation_id] ||= result
          failed << [ request, result ]
        end
      end
      Outcome.new(signed: signed, failed: failed)
    end

    # Consulta pela finalização, adendo pela criação; desempate pelo id do pedido.
    def chronological(requests)
      requests.sort_by do |request|
        document = request.document
        time = case document
               when Consultation then document.finalized_at
               when ConsultationAddendum then document.created_at
               end
        [ time || request.created_at, request.id ]
      end
    end

    def store(request, docs, prepared, raw, certificate, info, signer, now)
      cades = signer.assemble(kind: "cades", state: prepared["cades"].state, signature_value: raw.fetch("#{request.id}:cades"))
      pades = signer.assemble(kind: "pades", state: prepared["pades"].state, signature_value: raw.fetch("#{request.id}:pades"))
      checks = [ signer.verify(kind: "cades", signature: cades.signature, document: docs.canonical.json),
                 signer.verify(kind: "pades", signature: pades.signature) ]
      reason = rejection(checks, info)
      return reason if reason

      ApplicationRecord.transaction(requires_new: true) do
        signature = Signature.create!(
          signature_request: request, document_type: request.document_type, document_id: request.document_id,
          canonical_json: docs.canonical.json, canonical_sha256: docs.canonical.sha256,
          cades: Base64.strict_encode64(cades.signature), signed_pdf: Base64.strict_encode64(pades.signature),
          pdf_sha256: Digest::SHA256.hexdigest(pades.signature), policy: "AD-RB", policy_oid: checks.first.policy_oid,
          validation_material: { "cades" => Base64.strict_encode64(cades.validation_material),
                                 "pades" => Base64.strict_encode64(pades.validation_material) }.to_json,
          signer_certificate: certificate, signer_cpf: info.cpf, signed_at: checks.first.signed_at || now,
          last_verification: "valid", last_verification_at: now, last_verification_reasons: []
        )
        request.update!(status: "signed", reason_code: nil, resolved_at: now)
        DomainEvents.publish("signature.signed", signature_id: signature.id, request_id: request.id,
                                                 document_type: DocumentTypes.api(request.document_type), document_id: request.document_id)
        signature
      end
    rescue Signer::Rejected
      "verification_failed"
    rescue Signer::Unavailable
      "signer_unavailable" # depois do PSC: o lote não perde os já gravados; o job tenta de novo
    end

    def rejection(checks, info)
      return "certificate_revoked" if checks.any? { |check| check.reasons.include?("certificate_revoked") }
      return "certificate_cpf_mismatch" if checks.any? { |check| check.signer_cpf.present? && check.signer_cpf != info.cpf }
      return "verification_failed" unless checks.all?(&:valid?)

      nil
    end
    private_class_method :store, :rejection, :chronological
  end
end
```

```ruby
# app/commands/signatures/park.rb
# O pedido continua pending, agora com o motivo (ADR 0032; spec §5). Falha
# passageira soma uma tentativa.
module Signatures
  module Park
    module_function

    def call(request, reason, transient: false, now: Time.current)
      attributes = { reason_code: reason, updated_at: now }
      attributes[:attempts] = request.attempts + 1 if transient
      request.update!(attributes)
      DomainEvents.publish("signature.failed", request_id: request.id, reason_code: reason)
      request
    end
  end
end
```

```ruby
# app/commands/signatures/to_paper.rb
# Volta ao papel (ADR 0032; spec §5): o impresso do 19a volta a ter espaço de
# assinatura à mão. Pelo autor (user_request, com nota) ou pelo sistema
# (feature_disabled). Quem chama já travou o pedido e conferiu que está pending.
module Signatures
  module ToPaper
    module_function

    def call(request, reason_code:, note: nil, now: Time.current)
      request.update!(status: "returned_to_paper", reason_code: reason_code, return_note: note, resolved_at: now)
      DomainEvents.publish("signature.returned_to_paper", request_id: request.id, reason_code: reason_code)
      request
    end
  end
end
```

```ruby
# app/commands/signatures/sign_pending.rb
# Uma tentativa de assinar um pedido (ADR 0032; spec §5). FOR UPDATE SKIP
# LOCKED: se o lote (ou outro job) já trava o pedido, este sai sem esperar a
# chamada ao PSC dele (Review Focus 1). Chame dentro da transação da cidade.
module Signatures
  module SignPending
    MAX_ATTEMPTS = 3

    module_function

    def call(request_id:, now: Time.current, signer: Signer.client)
      request = SignatureRequest.lock("FOR UPDATE SKIP LOCKED").find_by(id: request_id)
      return :skipped unless request&.pending?

      unless Gate.usable?(Current.city)
        ToPaper.call(request, reason_code: "feature_disabled", now: now)
        return :returned_to_paper
      end

      certificate = SignerCertificate.active.find_by(user_id: request.author_user_id)
      return park(request, "no_session", now) unless certificate

      reason = CertificateRules.reason(certificate, now: now)
      return park(request, reason, now) if reason

      session = SignatureSession.usable_for(request.author_user_id, now: now)
      unless session
        return park(request, SignatureSession.lapsed?(request.author_user_id, now: now) ? "session_expired" : "no_session", now)
      end

      outcome = Signing.call(requests: [ request ], access_token: session.access_token, certificate: certificate, now: now,
                             signer: signer)
      return :signed if outcome.signed.any?

      settle(request, outcome.failed.sole.last, certificate, now)
    rescue Psc::Unauthorized
      session&.update!(status: "expired")
      park(request, "session_expired", now)
    rescue Psc::Unavailable
      retry_later(request, "provider_unavailable", now)
    rescue Psc::Rejected
      park(request, "provider_rejected", now)
    rescue Signer::Unavailable
      retry_later(request, "signer_unavailable", now)
    end

    def settle(request, reason, certificate, now)
      certificate.update!(status: "revoked") if reason == "certificate_revoked"
      SignatureRequest::TRANSIENT_REASONS.include?(reason) ? retry_later(request, reason, now) : park(request, reason, now)
    end

    def park(request, reason, now)
      Park.call(request, reason, now: now)
      :pending
    end

    def retry_later(request, reason, now)
      Park.call(request, reason, transient: true, now: now)
      request.attempts < MAX_ATTEMPTS ? :retry : :pending
    end
    private_class_method :settle, :park, :retry_later
  end
end
```

```ruby
# app/jobs/signatures/sign_job.rb
# Assina um pedido na fila da cidade (ADR 0032; spec §5). Falha passageira:
# até 3 tentativas com espera crescente (30 s, 2 min); depois o pedido fica
# pending com o motivo, à espera do lote ou da volta ao papel. Nunca levanta
# por PSC/signer (a transação do CityScopedJob commita o motivo).
module Signatures
  class SignJob < ApplicationJob
    include CityScopedJob

    queue_as :default

    BACKOFF = [ 30.seconds, 2.minutes ].freeze

    def perform(city_slug:, request_id:)
      with_city(city_slug) do
        next unless SignPending.call(request_id: request_id) == :retry

        attempts = SignatureRequest.where(id: request_id).pick(:attempts).to_i
        self.class.set(wait: BACKOFF.fetch(attempts - 1, BACKOFF.last)).perform_later(city_slug: city_slug, request_id: request_id)
      end
    end
  end
end
```

- [ ] **Step 4: Rode e veja passar (e o fluxo com o `signer` real)**

Run: `docker compose exec -T -w /rails/.claude/mod19b api bundle exec rspec spec/commands/signatures/sign_pending_spec.rb spec/jobs/signatures/sign_job_spec.rb`
Expected: PASS.

Run: `docker compose exec -T -e SIGNER_URL=http://signer:8090 -w /rails/.claude/mod19b api bundle exec rspec spec/integration/signing_flow_spec.rb`
Expected: PASS (com o `signer` e o `fake-psc` de pé; senão, rode na Task 19).

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod19b add app/services/signatures/documents.rb app/services/signatures/signing.rb app/services/signatures/certificate_rules.rb app/commands/signatures/sign_pending.rb app/commands/signatures/park.rb app/commands/signatures/to_paper.rb app/jobs/signatures/sign_job.rb spec/commands/signatures/sign_pending_spec.rb spec/jobs/signatures/sign_job_spec.rb spec/integration/signing_flow_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod19b commit -m "feat: sign consultations and addenda in CAdES and PAdES with retries

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 12: O pedido nasce da finalização e do adendo (adoção por profissional)

**Files:**
- Create: `app/commands/signatures/open_request.rb`, `app/jobs/signatures/request_job.rb`
- Modify: `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`
- Test: `spec/jobs/signatures/request_job_spec.rb`

**Interfaces:**
- Consumes: eventos `consultation.finalized { consultation_id, attendance_id }` e `consultation.addendum_added { consultation_id, addendum_id }` (19a), `IdempotentConsumer` (existente), `Signatures::{Gate,DocumentTypes,SignJob}`.
- Produces:
  - `Signatures::OpenRequest.call(document, now: Time.current) -> :created | :exists | :manual | :unusable` (interruptor utilizável **e** autor com certificado `active` → pedido `pending` + `SignJob` enfileirado; sem certificado → nada nasce, o documento é `manual`).
  - `Signatures::RequestJob` (`IdempotentConsumer`, sem HTTP), ligado a `consultation.finalized` e `consultation.addendum_added`.

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/jobs/signatures/request_job_spec.rb
require "rails_helper"

# ADR 0032 (spec §5, finalização; adoção por profissional): o 19a não muda; o
# consumidor dos eventos abre o pedido só com o interruptor utilizável e o autor
# com certificado ativo, e enfileira o job. Finalizar nunca espera.
RSpec.describe Signatures::RequestJob do
  include ActiveJob::TestHelper

  before do
    Current.city = signature_city!
    ciap2_release!; cid10_release!; sigtap_release!
  end
  after { Current.reset }

  let(:unit) { create_unit }
  let(:doctor) { signer_doctor!(unit) }

  def consume!(name)
    DomainEvent.where(name: name).order(:occurred_at).each do |event|
      described_class.perform_now(event_id: event.id, event_name: event.name, city_slug: Current.city.slug, payload: event.payload)
    end
  end

  it "está ligado aos dois eventos do 19a" do
    %w[consultation.finalized consultation.addendum_added].each do |name|
      expect(DomainEvents.registry[name].map(&:job)).to include("Signatures::RequestJob")
    end
  end

  it "autor com certificado: pedido pending e job enfileirado; consumir de novo não duplica" do
    linked_certificate!(doctor)
    consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1))
    expect { consume!("consultation.finalized") }.to have_enqueued_job(Signatures::SignJob)
    request = SignatureRequest.sole
    expect(request).to have_attributes(document_type: "Consultation", document_id: consultation.id,
                                       consultation_id: consultation.id, author_user_id: doctor.id, status: "pending", attempts: 0)
    expect { consume!("consultation.finalized") }.not_to change(SignatureRequest, :count)
    expect(Signatures::OpenRequest.call(consultation)).to eq(:exists)
  end

  it "autor sem certificado: nada nasce (manual); interruptor desligado: nada nasce" do
    finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1))
    consume!("consultation.finalized")
    expect(SignatureRequest.count).to eq(0)

    linked_certificate!(doctor)
    signature_city!(enabled: false)
    consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(2))
    expect(Signatures::OpenRequest.call(consultation)).to eq(:unusable)
    expect(SignatureRequest.count).to eq(0)
  end

  it "adendo: pelo autor do adendo; outro autor sem certificado fica no papel" do
    linked_certificate!(doctor)
    consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1))
    Consultations::AddAddendum.call(consultation: consultation, by: doctor, reason: "primeiro adendo aqui", text: "UM")
    nurse = signer_doctor!(unit, cpf: SignatureHelpers::OTHER_CPF)
    consulting_attendance!(unit, citizen: consultation.attendance.citizen, doctor: nurse)
    Consultations::AddAddendum.call(consultation: consultation, by: nurse, reason: "segundo adendo aqui", text: "DOIS")
    consume!("consultation.addendum_added")
    expect(SignatureRequest.pluck(:document_type, :author_user_id)).to eq([ [ "ConsultationAddendum", doctor.id ] ])
  end

  it "finalizar nunca espera o PSC nem o signer (o consumidor só enfileira)" do
    stub_psc!
    fake_psc.failures.push(503, 503, 503)
    stub_signer!.unavailable = true
    linked_certificate!(doctor)
    expect { finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1)) }.not_to raise_error
    expect { consume!("consultation.finalized") }.not_to raise_error
    expect(fake_psc.log).to be_empty
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod19b api bundle exec rspec spec/jobs/signatures/request_job_spec.rb`
Expected: FAIL (`uninitialized constant Signatures::RequestJob`).

- [ ] **Step 3: Implemente**

```ruby
# app/commands/signatures/open_request.rb
# Abre o pedido de assinatura de um documento recém-finalizado (ADR 0032; spec
# §5): só com o interruptor utilizável e o autor com certificado ativo (adoção
# por profissional — sem certificado, o documento é manual). Um por documento
# (índice único; o consumidor pode repetir).
module Signatures
  module OpenRequest
    module_function

    def call(document, now: Time.current)
      return :unusable unless Gate.usable?(Current.city)
      return :manual unless SignerCertificate.active.exists?(user_id: document.author_user_id)

      inserted = SignatureRequest.insert_all(
        [ { document_type: DocumentTypes.db(document), document_id: document.id,
            consultation_id: DocumentTypes.consultation_id(document), author_user_id: document.author_user_id,
            status: "pending", attempts: 0, created_at: now, updated_at: now } ],
        unique_by: :idx_signature_requests_document, returning: [ :id ]
      )
      return :exists if inserted.rows.empty?

      SignJob.perform_later(city_slug: Current.city.slug, request_id: inserted.rows.first.first)
      :created
    end
  end
end
```

```ruby
# app/jobs/signatures/request_job.rb
# Consumidor de consultation.finalized e consultation.addendum_added (ADR 0032;
# spec §5). Só abre o pedido e enfileira o SignJob: nada de HTTP aqui
# (ADR-0005). O 19a não muda.
module Signatures
  class RequestJob < ApplicationJob
    include IdempotentConsumer

    queue_as :default

    def handle(consultation_id:, addendum_id: nil, **)
      document = addendum_id ? ConsultationAddendum.find_by(id: addendum_id) : Consultation.find_by(id: consultation_id)
      return if document.nil?
      return if document.is_a?(Consultation) && !document.finalized?

      OpenRequest.call(document)
    end
  end
end
```

Em `config/initializers/domain_events.rb`, troque as duas linhas do 19a por:

```ruby
  # ADR 0032 (19b): a finalização e o adendo abrem o pedido de assinatura
  # (Signatures::RequestJob só abre o pedido e enfileira; o 19a não muda).
  DomainEvents.bind "consultation.finalized", to: [ Signatures::RequestJob ]
  DomainEvents.bind "consultation.addendum_added", to: [ Signatures::RequestJob ]
```

Em `spec/initializers/domain_events_bindings_spec.rb`, no bloco do 19a ("declares every module 19 city event with no consumer"), tire `consultation.finalized` e `consultation.addendum_added` da lista `names` (agora têm consumidor) e acrescente:

```ruby
# Módulo 19b (ADR 0032): a finalização e o adendo abrem o pedido de assinatura.
RSpec.describe "consultation signature bindings (ADR 0032)" do
  it "binds consultation.finalized and consultation.addendum_added to Signatures::RequestJob" do
    %w[consultation.finalized consultation.addendum_added].each do |name|
      expect(DomainEvents.registry[name].map(&:job)).to eq([ "Signatures::RequestJob" ])
    end
  end
end
```

- [ ] **Step 4: Rode e veja passar (e as specs do 19a que finalizam e adicionam adendo)**

Run: `docker compose exec -T -w /rails/.claude/mod19b api bundle exec rspec spec/jobs/signatures/request_job_spec.rb spec/initializers/domain_events_bindings_spec.rb spec/commands/consultations`
Expected: PASS. (Spec do 19a que conte jobs enfileirados depois de finalizar passa a ver também o `Signatures::RequestJob`: filtre a expectativa pela classe que ela verifica — nunca afrouxe o que ela prova.)

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod19b add app/commands/signatures/open_request.rb app/jobs/signatures/request_job.rb config/initializers/domain_events.rb spec/initializers/domain_events_bindings_spec.rb spec/jobs/signatures/request_job_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod19b commit -m "feat: open a signature request when a consultation or addendum is finalized

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

(Se alguma spec do 19a mudou no Step 4, inclua o caminho no `git add`.)

---
### Task 13: Fila de pendentes, lote com uma aprovação, volta ao papel e varredura

**Files:**
- Create: `app/commands/signatures/{return_to_paper,start_batch,run_batch}.rb`, `app/jobs/signatures/sweep_job.rb`, `app/controllers/signatures/{requests,batches}_controller.rb`
- Modify: `app/commands/signatures/complete_oauth.rb`, `app/services/signatures/json.rb`, `config/routes.rb`, `config/recurring.yml`
- Test: `spec/requests/signature_queue_spec.rb`, `spec/jobs/signatures/sweep_job_spec.rb`, `spec/commands/signatures/sign_race_spec.rb`, `spec/commands/signatures/run_batch_chain_spec.rb`

**Interfaces:**
- Consumes: `Signing`, `SignPending`, `Park`, `ToPaper`, `CertificateRules` (Task 11), `OauthStates`, `CompleteOauth` (Tasks 6–8), `EachCityJob`.
- Produces:
  - `Signatures::ReturnToPaper.call(request_id:, by:, reason:, now: Time.current) -> Result` (ok `{ request: }`; falhas `:invalid_reason` (motivo 10–500 após `squish`), `:not_found`, `:not_author`, `:not_pending`); trava com `FOR UPDATE` (espera o job em voo e então vê `signed` → `not_pending`).
  - `Signatures::StartBatch::LIMIT == 50`, `.call(user:, request_ids: nil, return_to: nil) -> Result` (ok `{ authorize_url:, count: }`, escopo `multi_signature`; só pedidos `pending` do próprio autor; ids que não são uuid ignorados; falhas `:certificate_not_linked`, `:invalid_provider`, `:professional_cpf_missing`, `:nothing_pending`).
  - `Signatures::RunBatch.call(user:, request_ids:, token:, now: Time.current, signer: Signer.client) -> Result` (abre a própria transação; `FOR UPDATE SKIP LOCKED` — pedido travado pelo job fica de fora sem espera; ok `{ record: { signed: n, failed: [{ request_id:, reason_code: }] } }`; PSC/`signer` fora do ar viram itens de `failed`; falhas `:feature_disabled`, `:certificate_not_linked`).
  - `CompleteOauth` aceita `batch`.
  - `Signatures::Json.requests(rows) -> Array<Hash>` (forma `<request>` do contrato §5).
  - `Signatures::SweepJob` (`EachCityJob`, a cada 10 min): interruptor não utilizável → `pending` vira `returned_to_paper`/`feature_disabled`; sessões vencidas → `expired`; `state` com mais de 1 dia apagado.
  - Rotas: `GET /signature/requests`, `POST /signature/requests/:id/return_to_paper`, `POST /signature/batches`.

- [ ] **Step 1: Escreva as specs que falham**

```ruby
# spec/requests/signature_queue_spec.rb
require "rails_helper"

# Contrato §5 (spec §5, lote e volta ao papel; NGS2.02.05/06): a fila do
# próprio autor, o lote com UMA aprovação (multi_signature, os dois hashes de
# cada documento numa chamada) e a volta ao papel com motivo.
RSpec.describe "Pendentes, lote e volta ao papel", type: :request do
  before do
    signature_city!
    ciap2_release!; cid10_release!; sigtap_release!
    stub_psc!
    stub_signer!
  end

  let(:unit) { create_unit }
  let(:doctor) { signer_doctor!(unit) }
  let(:cpf) { SignatureHelpers::DOCTOR_CPF }
  def body = JSON.parse(response.body)

  def pending_for!(user, n, at: unit, first_citizen: 10)
    Array.new(n) do |i|
      consultation = finalized_consultation!(unit: at, doctor: user, citizen: verified_citizen!(first_citizen + i))
      signature_request!(consultation, author: user, reason_code: "no_session")
    end
  end

  def other_professional!
    other_unit = create_unit("UBS Dois")
    [ signer_doctor!(other_unit, cpf: SignatureHelpers::OTHER_CPF), other_unit ]
  end

  def run_batch!(params = {})
    post "/signature/batches", params: { return_to: "/pendentes" }.merge(params), as: :json
    url = body.fetch("authorize_url")
    state = URI.decode_www_form(URI(url).query).to_h.fetch("state")
    post "/signature/oauth/callback", params: { state: state, code: authorize_and_approve!(url) }, as: :json
    url
  end

  it "lista só os pendentes do autor, mais antigos primeiro, na forma do contrato" do
    linked_certificate!(doctor)
    first, second = pending_for!(doctor, 2)
    other, other_unit = other_professional!
    pending_for!(other, 1, at: other_unit, first_citizen: 60)
    signature_request!(author: doctor, status: "signed")
    sign_in_as(doctor)
    get "/signature/requests", params: { status: "pending" }
    expect(body["items"].map { |i| i["id"] }).to eq([ first.id, second.id ])
    item = body["items"].first
    consultation = Consultation.find(first.document_id)
    expect(item).to eq("id" => first.id, "document_type" => "consultation", "document_id" => consultation.id,
                       "consultation_id" => consultation.id, "patient_display_name" => consultation.patient.display_name,
                       "finalized_at" => consultation.finalized_at.iso8601, "status" => "pending",
                       "reason_code" => "no_session", "attempts" => 0)
  end

  it "volta ao papel: motivo de 10+, só o autor, só pendente" do
    request = pending_for!(doctor, 1).first
    sign_in_as(doctor)
    post "/signature/requests/#{request.id}/return_to_paper", params: { reason: "curto" }, as: :json
    expect([ response.status, body["error"] ]).to eq([ 422, "invalid_reason" ])
    sign_in_as(signer_doctor!(create_unit("UBS Dois"), cpf: SignatureHelpers::OTHER_CPF))
    post "/signature/requests/#{request.id}/return_to_paper", params: { reason: "sem certificado hoje" }, as: :json
    expect([ response.status, body["error"] ]).to eq([ 403, "not_author" ])
    sign_in_as(doctor)
    post "/signature/requests/#{request.id}/return_to_paper", params: { reason: "  sem   certificado hoje  " }, as: :json
    expect(response).to have_http_status(:ok)
    expect(body).to include("id" => request.id, "status" => "returned_to_paper", "reason_code" => "user_request")
    expect(request.reload.return_note).to eq("sem certificado hoje")
    post "/signature/requests/#{request.id}/return_to_paper", params: { reason: "sem certificado hoje" }, as: :json
    expect([ response.status, body["error"] ]).to eq([ 409, "not_pending" ])
    post "/signature/requests/#{SecureRandom.uuid}/return_to_paper", params: { reason: "sem certificado hoje" }, as: :json
    expect(response).to have_http_status(:not_found)
  end

  it "lote: uma aprovação multi_signature assina todos (dois hashes por documento numa chamada)" do
    linked_certificate!(doctor, leaf: fake_psc.leaf(cpf))
    requests = pending_for!(doctor, 2)
    sign_in_as(doctor)
    url = run_batch!
    expect(URI.decode_www_form(URI(url).query).to_h).to include("scope" => "multi_signature", "login_hint" => cpf)
    expect(body).to eq("purpose" => "batch", "result" => { "signed" => 2, "failed" => [] }, "return_to" => "/pendentes")
    expect(requests.map { |r| r.reload.status }).to eq(%w[signed signed])
    expect(fake_psc.log.count { |_verb, path| path == "/v0/oauth/signature" }).to eq(1)
  end

  it "lote: ids escolhidos; ids de outro autor e lixo ficam de fora; sem nada: 409; sem certificado: 409" do
    sign_in_as(doctor)
    post "/signature/batches", params: { return_to: "/" }, as: :json
    expect([ response.status, body["error"] ]).to eq([ 409, "certificate_not_linked" ])
    linked_certificate!(doctor, leaf: fake_psc.leaf(cpf))
    post "/signature/batches", params: { return_to: "/" }, as: :json
    expect([ response.status, body["error"] ]).to eq([ 409, "nothing_pending" ])
    mine = pending_for!(doctor, 2)
    other, other_unit = other_professional!
    theirs = pending_for!(other, 1, at: other_unit, first_citizen: 60)
    post "/signature/batches", params: { request_ids: [ mine.first.id, theirs.first.id, "lixo" ], return_to: "/" }, as: :json
    expect(body["count"]).to eq(1)
  end

  it "lote com o PSC fora do ar na assinatura: itens em failed, pedidos continuam pendentes com o motivo" do
    linked_certificate!(doctor, leaf: fake_psc.leaf(cpf))
    requests = pending_for!(doctor, 2)
    sign_in_as(doctor)
    post "/signature/batches", params: { return_to: "/" }, as: :json
    url = body.fetch("authorize_url")
    state = URI.decode_www_form(URI(url).query).to_h.fetch("state")
    code = authorize_and_approve!(url)
    # a troca do código passa; a assinatura cai
    allow_any_instance_of(Signatures::Psc::Client).to receive(:sign).and_raise(Signatures::Psc::Unavailable, "fora")
    post "/signature/oauth/callback", params: { state: state, code: code }, as: :json
    expect(body["result"]).to eq("signed" => 0,
                                 "failed" => requests.map { |r| { "request_id" => r.id, "reason_code" => "provider_unavailable" } })
    expect(requests.map { |r| r.reload.slice(:status, :reason_code).values }).to all(eq(%w[pending provider_unavailable]))
  end

  it "o lote pega no máximo 50" do
    linked_certificate!(doctor)
    51.times { signature_request!(author: doctor) }
    sign_in_as(doctor)
    post "/signature/batches", params: { return_to: "/" }, as: :json
    expect(body["count"]).to eq(50)
  end
end
```

```ruby
# spec/jobs/signatures/sweep_job_spec.rb
require "rails_helper"

# Desvio 8 (spec §3): desligado o interruptor (ou o prontuário), pendentes
# voltam ao papel com feature_disabled; o assinado continua; sessões vencidas e
# states velhos são limpos.
RSpec.describe Signatures::SweepJob do
  before { Current.city = signature_city! }
  after { Current.reset }

  it "desligado: pendentes ao papel; o resto intacto; sessões e states" do
    doctor = signer_doctor!(create_unit)
    certificate = linked_certificate!(doctor)
    pending = signature_request!(author: doctor)
    signed = signature_request!(author: doctor, status: "signed")
    signature_session!(doctor, certificate: certificate, started_at: 13.hours.ago, expires_at: 1.hour.ago)
    old_state = Signatures::OauthStates.issue!(user: doctor, purpose: "link", provider: "vidaas", now: 2.days.ago).row

    described_class.perform_now
    expect(pending.reload.status).to eq("pending") # ligado: nada muda na fila
    expect(SignatureSession.where(user_id: doctor.id).pluck(:status)).to eq([ "expired" ])
    expect(SignatureOauthState.exists?(old_state.id)).to be(false)

    signature_city!(enabled: false)
    described_class.perform_now
    expect(pending.reload).to have_attributes(status: "returned_to_paper", reason_code: "feature_disabled")
    expect(signed.reload.status).to eq("signed")
  end
end
```

```ruby
# spec/commands/signatures/run_batch_chain_spec.rb
require "rails_helper"

# Decisão do usuário (cadeia no lote): assinar em ordem cronológica de criação;
# o adendo seguinte leva o hash canônico do anterior do MESMO lote; se o
# anterior falha, o seguinte não é assinado e fica pending com o mesmo motivo.
RSpec.describe "Cadeia no lote" do
  before do
    Current.city = signature_city!
    ciap2_release!; cid10_release!; sigtap_release!
    stub_psc!
    @signer = stub_signer!
    linked_certificate!(doctor, leaf: fake_psc.leaf(cpf))
  end
  after { Current.reset }

  let(:unit) { create_unit }
  let(:doctor) { signer_doctor!(unit) }
  let(:cpf) { SignatureHelpers::DOCTOR_CPF }
  let(:consultation) { finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1)) }

  def addendum!(text)
    Consultations::AddAddendum.call(consultation: consultation, by: doctor, reason: "adendo #{text} aqui", text: text).payload[:addendum]
  end

  def token
    Signatures::Psc::Token.new(access_token: fake_psc.token_for!(cpf: cpf, scope: "multi_signature"), expires_in: 300,
                               scope: "multi_signature")
  end

  it "dois adendos no mesmo lote (pedidos criados fora de ordem): o segundo leva o hash do primeiro" do
    one = addendum!("um")
    two = addendum!("dois")
    second_request = signature_request!(two, author: doctor) # o pedido do segundo nasce antes
    first_request = signature_request!(one, author: doctor)
    result = Signatures::RunBatch.call(user: doctor, request_ids: [ second_request.id, first_request.id ], token: token)
    expect(result.payload[:record]).to eq(signed: 2, failed: [])
    first = first_request.reload.signature
    second = second_request.reload.signature
    expect(JSON.parse(first.canonical_json).dig("addendum", "previous_sha256"))
      .to eq(Signatures::Canonical.consultation(consultation).sha256)
    expect(JSON.parse(second.canonical_json).dig("addendum", "previous_sha256")).to eq(first.canonical_sha256)
  end

  it "consulta e adendo no mesmo lote: o adendo aponta para a consulta" do
    addendum = addendum!("um")
    requests = [ signature_request!(addendum, author: doctor), signature_request!(consultation, author: doctor) ]
    Signatures::RunBatch.call(user: doctor, request_ids: requests.map(&:id), token: token)
    signed = requests.map { |request| request.reload.signature }
    expect(JSON.parse(signed.first.canonical_json).dig("addendum", "previous_sha256")).to eq(signed.last.canonical_sha256)
  end

  it "o primeiro falha: o segundo não é assinado e fica pending com o mesmo motivo" do
    one = addendum!("um")
    two = addendum!("dois")
    first_request = signature_request!(one, author: doctor)
    second_request = signature_request!(two, author: doctor)
    allow(Consultations::Print).to receive(:addendum).and_wrap_original do |original, addendum, **options|
      raise Consultations::Print::NotPrintable, "teste" if addendum.id == one.id

      original.call(addendum, **options)
    end
    result = Signatures::RunBatch.call(user: doctor, request_ids: [ first_request.id, second_request.id ], token: token)
    expect(result.payload[:record]).to eq(signed: 0, failed: [ { request_id: first_request.id, reason_code: "verification_failed" },
                                                               { request_id: second_request.id, reason_code: "verification_failed" } ])
    expect([ first_request, second_request ].map { |r| r.reload.slice(:status, :reason_code).values }).to all(eq(%w[pending verification_failed]))
    expect(Signature.count).to eq(0)
  end

  it "o primeiro falha depois do PSC (verificação): o segundo também não é gravado" do
    one = addendum!("um")
    two = addendum!("dois")
    first_request = signature_request!(one, author: doctor)
    second_request = signature_request!(two, author: doctor)
    first_json = Signatures::Canonical.addendum(one).json
    allow(@signer).to receive(:verify).and_wrap_original do |original, **options|
      result = original.call(**options)
      options[:document] == first_json ? result.with(status: "invalid", reasons: [ "signature_mismatch" ]) : result
    end
    result = Signatures::RunBatch.call(user: doctor, request_ids: [ first_request.id, second_request.id ], token: token)
    expect(result.payload[:record][:signed]).to eq(0)
    expect(result.payload[:record][:failed].map { |item| item[:reason_code] }.uniq).to eq([ "verification_failed" ])
    expect(Signature.count).to eq(0)
  end
end
```

```ruby
# spec/commands/signatures/sign_race_spec.rb
require "rails_helper"

# Review Focus 1 (ADR 0032, invariante "nenhum documento é assinado duas
# vezes"): job e lote sobre o MESMO pedido, em conexões reais. Quem chega
# segundo pula o pedido (SKIP LOCKED) sem esperar a chamada ao PSC do outro.
RSpec.describe "Corrida job × lote" do
  self.use_transactional_tests = false

  let(:city) { TEST_CITY_A }
  let(:entered) { Queue.new }
  let(:release) { Queue.new }
  let(:calls) { Queue.new }

  before do
    allow(Signatures::Gate).to receive(:usable?).and_return(true)
    allow(Signatures::CertificateRules).to receive(:reason).and_return(nil)
    allow(Signatures::Signing).to receive(:call) do |requests:, **|
      calls << requests.map(&:id)
      entered << true
      release.pop(timeout: 5)
      requests.each { |request| request.update!(status: "signed", resolved_at: Time.current) }
      Signatures::Signing::Outcome.new(signed: requests, failed: [])
    end
    @rows = CityConnection.with(city) do
      user = User.create!(email_address: "corrida-#{SecureRandom.hex(4)}@cidade.gov.br", password: "senha-segura-123")
      certificate = linked_certificate!(user, leaf: test_pki.leaf_for(SignatureHelpers::DOCTOR_CPF))
      session = signature_session!(user, certificate: certificate)
      request = signature_request!(author: user)
      { user: user, certificate: certificate, session: session, request: request }
    end
  end

  after do
    3.times { release << true }
    CityConnection.with(city) do
      ApplicationRecord.transaction do
        ApplicationRecord.connection.execute("SET LOCAL session_replication_role = replica")
        SignatureRequest.where(id: @rows[:request].id).delete_all
        SignatureSession.where(id: @rows[:session].id).delete_all
        SignerCertificate.where(id: @rows[:certificate].id).delete_all
        DomainEvent.where("payload->>'request_id' = ?", @rows[:request].id).delete_all
        User.where(id: @rows[:user].id).delete_all
      end
    end
  end

  def in_city(&block) = Thread.new { CityConnection.with(city) { ApplicationRecord.transaction(&block) } }

  def job = in_city { Signatures::SignPending.call(request_id: @rows[:request].id) }

  def batch
    token = Signatures::Psc::Token.new(access_token: "t", expires_in: 300, scope: "multi_signature")
    Thread.new { CityConnection.with(city) { Signatures::RunBatch.call(user: @rows[:user], request_ids: [ @rows[:request].id], token: token) } }
  end

  it "job primeiro: o lote pula sem esperar; uma assinatura só" do
    first = job
    expect(entered.pop(timeout: 5)).to be(true)
    second = batch
    expect(second.join(5)).to be_truthy # terminou com o job ainda dentro do PSC
    expect(second.value.payload[:record]).to eq(signed: 0, failed: [])
    release << true
    expect(first.value).to eq(:signed)
    expect(calls.size).to eq(1)
  end

  it "lote primeiro: o job pula sem esperar; uma assinatura só" do
    first = batch
    expect(entered.pop(timeout: 5)).to be(true)
    second = job
    expect(second.join(5)).to be_truthy
    expect(second.value).to eq(:skipped)
    release << true
    expect(first.value.payload[:record][:signed]).to eq(1)
    expect(calls.size).to eq(1)
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod19b api bundle exec rspec spec/requests/signature_queue_spec.rb spec/jobs/signatures/sweep_job_spec.rb spec/commands/signatures/sign_race_spec.rb spec/commands/signatures/run_batch_chain_spec.rb`
Expected: FAIL (`No route matches`, `uninitialized constant Signatures::SweepJob`, `Signatures::RunBatch`).

- [ ] **Step 3: Implemente**

```ruby
# app/commands/signatures/return_to_paper.rb
# Volta ao papel pelo autor (ADR 0032; spec §5; contrato §5). FOR UPDATE: se o
# job está assinando, espera e vê o resultado (signed → not_pending).
module Signatures
  module ReturnToPaper
    module_function

    def call(request_id:, by:, reason:, now: Time.current)
      note = reason.is_a?(String) ? reason.squish : ""
      unless note.size.between?(SignatureRequest::MIN_RETURN_NOTE, SignatureRequest::MAX_RETURN_NOTE)
        return Result.fail(:invalid_reason)
      end

      ApplicationRecord.transaction do
        request = SignatureRequest.lock.find_by(id: request_id)
        next Result.fail(:not_found) unless request
        next Result.fail(:not_author) unless request.author_user_id == by.id
        next Result.fail(:not_pending) unless request.pending?

        Result.ok(request: ToPaper.call(request, reason_code: "user_request", note: note, now: now))
      end
    end
  end
end
```

```ruby
# app/commands/signatures/start_batch.rb
# Começa o lote (ADR 0032; spec §5; contrato §5): os pendentes do próprio
# autor (todos ou os escolhidos), até 50, mais antigos primeiro, numa
# aprovação multi_signature.
module Signatures
  module StartBatch
    LIMIT = 50
    UUID = /\A\h{8}-\h{4}-\h{4}-\h{4}-\h{12}\z/

    module_function

    def call(user:, request_ids: nil, return_to: nil)
      certificate = SignerCertificate.active.find_by(user_id: user.id)
      return Result.fail(:certificate_not_linked) unless certificate
      return Result.fail(:invalid_provider) unless Providers.configured?(certificate.provider)

      cpf = user.professional&.cpf
      return Result.fail(:professional_cpf_missing) if cpf.blank?

      scope = SignatureRequest.pending.where(author_user_id: user.id)
      scope = scope.where(id: Array(request_ids).grep(UUID)) unless request_ids.nil?
      ids = scope.order(:created_at, :id).limit(LIMIT).pluck(:id)
      return Result.fail(:nothing_pending) if ids.empty?

      issued = OauthStates.issue!(user: user, purpose: "batch", provider: certificate.provider, request_ids: ids, return_to: return_to)
      url = Psc::Client.for(certificate.provider).authorize_url(state: issued.state, challenge: issued.challenge,
                                                                scope: "multi_signature", login_hint: cpf,
                                                                redirect_uri: Providers.redirect_uri(Current.city))
      Result.ok(authorize_url: url, count: ids.size)
    end
  end
end
```

```ruby
# app/commands/signatures/run_batch.rb
# Assina o lote aprovado (ADR 0032; spec §5): SKIP LOCKED — o que o job já
# está assinando fica de fora sem espera (Review Focus 1); uma chamada ao PSC
# com todos os hashes; falha do PSC/signer vira item de `failed` e o pedido
# continua pending com o motivo.
module Signatures
  module RunBatch
    module_function

    def call(user:, request_ids:, token:, now: Time.current, signer: Signer.client)
      ApplicationRecord.transaction do
        requests = SignatureRequest.where(id: request_ids, author_user_id: user.id, status: "pending")
                                   .order(:created_at, :id).lock("FOR UPDATE SKIP LOCKED").to_a
        next Result.ok(record: { signed: 0, failed: [] }) if requests.empty?
        next Result.fail(:feature_disabled) unless Gate.usable?(Current.city)

        certificate = SignerCertificate.active.find_by(user_id: user.id)
        next Result.fail(:certificate_not_linked) unless certificate

        reason = CertificateRules.reason(certificate, now: now)
        next Result.ok(record: report([], requests.map { |request| [ request, reason ] }, now)) if reason

        outcome = begin
          Signing.call(requests: requests, access_token: token.access_token, certificate: certificate, now: now, signer: signer)
        rescue Psc::Unavailable, Psc::Rejected, Signer::Unavailable => e
          Signing::Outcome.new(signed: [], failed: requests.map { |request| [ request, reason_for(e) ] })
        end
        certificate.update!(status: "revoked") if outcome.failed.any? { |_request, code| code == "certificate_revoked" }
        Result.ok(record: report(outcome.signed, outcome.failed, now))
      end
    end

    def reason_for(error)
      case error
      when Psc::Unavailable then "provider_unavailable"
      when Psc::Rejected then "provider_rejected"
      else "signer_unavailable"
      end
    end

    def report(signed, failed, now)
      failed.each { |request, reason| Park.call(request, reason, now: now) }
      { signed: signed.size, failed: failed.map { |request, reason| { request_id: request.id, reason_code: reason } } }
    end
    private_class_method :reason_for, :report
  end
end
```

Em `app/commands/signatures/complete_oauth.rb`, em `dispatch`, antes do `else`:

```ruby
      when "batch"
        RunBatch.call(user: user, request_ids: row.request_ids, token: token, now: now)
```

```ruby
# app/jobs/signatures/sweep_job.rb
# Varredura da assinatura em cada cidade (ADR 0032; Desvio 8), a cada 10 min:
# interruptor não utilizável → pendentes voltam ao papel (feature_disabled);
# sessões vencidas → expired; states com mais de 1 dia apagados.
module Signatures
  class SweepJob < ApplicationJob
    prepend EachCityJob

    queue_as :housekeeping

    def perform
      now = Time.current
      return_pending_to_paper(now) unless Gate.usable?(Current.city)
      SignatureSession.active.where(expires_at: ..now).update_all(status: "expired", updated_at: now)
      SignatureOauthState.where(expires_at: ...(now - 1.day)).delete_all
    end

    private

    def return_pending_to_paper(now)
      SignatureRequest.pending.pluck(:id).each do |id|
        ApplicationRecord.transaction do
          request = SignatureRequest.lock("FOR UPDATE SKIP LOCKED").find_by(id: id)
          ToPaper.call(request, reason_code: "feature_disabled", now: now) if request&.pending?
        end
      end
    end
  end
end
```

Em `config/recurring.yml`, no bloco `default`:

```yaml
  # ADR 0032: pendentes ao papel com o interruptor desligado; sessões e states.
  signature_sweep:
    class: Signatures::SweepJob
    queue: housekeeping
    schedule: every 10 minutes
```

Em `app/services/signatures/json.rb`, dentro do módulo:

```ruby
    # <request> (contrato §5), sem N+1: consultas e adendos numa consulta só.
    def requests(rows)
      consultations = Consultation.where(id: rows.map(&:consultation_id)).includes(:patient).index_by(&:id)
      addendum_ids = rows.select { |row| row.document_type == "ConsultationAddendum" }.map(&:document_id)
      addenda = ConsultationAddendum.where(id: addendum_ids).pluck(:id, :created_at).to_h
      rows.map do |row|
        consultation = consultations[row.consultation_id]
        finalized_at = row.document_type == "Consultation" ? consultation&.finalized_at : addenda[row.document_id]
        { id: row.id, document_type: DocumentTypes.api(row.document_type), document_id: row.document_id,
          consultation_id: row.consultation_id, patient_display_name: consultation&.patient&.display_name,
          finalized_at: finalized_at&.iso8601, status: row.status, reason_code: row.reason_code, attempts: row.attempts }
          .reject { |key, value| value.nil? && %i[reason_code patient_display_name finalized_at].include?(key) }
      end
    end
```

```ruby
# app/controllers/signatures/requests_controller.rb
# Pendentes de assinatura do próprio autor (contrato §5; NGS2.02.06).
module Signatures
  class RequestsController < BaseController
    LIMIT = 200

    def index
      rows = SignatureRequest.pending.where(author_user_id: Current.user.id).order(:created_at, :id).limit(LIMIT).to_a
      render json: { items: Json.requests(rows) }
    end

    def return_to_paper
      result = ReturnToPaper.call(request_id: params[:id], by: Current.user, reason: body["reason"])
      return failure(result, not_found: :not_found) if result.failure?

      render json: Json.requests([ result.payload[:request] ]).sole
    end
  end
end
```

```ruby
# app/controllers/signatures/batches_controller.rb
# "Assinar todas" (contrato §5).
module Signatures
  class BatchesController < BaseController
    def create
      result = StartBatch.call(user: Current.user, request_ids: body["request_ids"], return_to: body["return_to"])
      return failure(result) if result.failure?

      render json: result.payload
    end
  end
end
```

Em `config/routes.rb`, dentro do `scope "/signature"`:

```ruby
    get  "requests",                     to: "requests#index"
    post "requests/:id/return_to_paper", to: "requests#return_to_paper"
    post "batches",                      to: "batches#create"
```

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod19b api bundle exec rspec spec/requests/signature_queue_spec.rb spec/jobs/signatures/sweep_job_spec.rb spec/commands/signatures/sign_race_spec.rb spec/requests/signature_sessions_spec.rb`
Expected: PASS. Se alguma spec do projeto confere `config/recurring.yml` (classes existentes, `EachCityJob`), rode-a também (`git grep -ln "recurring.yml" spec`).

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod19b add spec/commands/signatures/run_batch_chain_spec.rb app/commands/signatures/return_to_paper.rb app/commands/signatures/start_batch.rb app/commands/signatures/run_batch.rb app/commands/signatures/complete_oauth.rb app/jobs/signatures/sweep_job.rb app/controllers/signatures/requests_controller.rb app/controllers/signatures/batches_controller.rb app/services/signatures/json.rb config/routes.rb config/recurring.yml spec/requests/signature_queue_spec.rb spec/jobs/signatures/sweep_job_spec.rb spec/commands/signatures/sign_race_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod19b commit -m "feat: add the pending signature queue, batch signing and return to paper

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
## Fatia 4 — Leitura, validação, admin e maintenance (F-19.8, F-19.13, F-19.14)

### Task 14: Bloco `signature`, validação, conteúdo assinado, PDF, pacote e o impresso

**Files:**
- Create: `app/services/signatures/{mode,verify,package,print_report}.rb`, `app/controllers/signatures/signatures_controller.rb`
- Modify: `app/services/consultations/json.rb`, `app/controllers/consultations_controller.rb`, `app/services/signatures/json.rb`, `config/routes.rb`, `spec/support/signature_helpers.rb`, `spec/services/consultations/json_spec.rb` (19a: chaves com `signature`)
- Test: `spec/requests/signature_reading_spec.rb`

**Interfaces:**
- Consumes: `ClinicalRecord::{Access,Trail}`, `ClinicalRecordGate`, `ConsultationsController#read_grant`, `Screenings::Json.staff_name` (19a/18); `SignPending` (Task 11); `Consultations::Print.call(..., report:)` (Task 10).
- Produces:
  - `Signatures::Mode.blocks(documents) -> Hash{[db_type, id] => Hash}` (pedidos e assinaturas em duas consultas, sem ler colunas cifradas), `.for(document) -> Hash` — `{ mode: "manual" }` sem pedido; `{ mode: "digital", request_id, signature_id, signed_at, signer_name, verification }`; `{ mode: "pending", request_id, reason_code? }`; `{ mode: "manual", request_id, reason_code }` na volta ao papel (contrato §2).
  - `Consultations::Json.consultation` ganha `signature` (finalizada) e cada `addenda[]` também; `Consultations::Json.addendum(a, signature: :load)`.
  - `Signatures::Verify.call(signature, now: Time.current, signer: Signer.client, explicit: false) -> Signature` (revalida CAdES e PAdES, grava `last_verification*`; publica `signature.verified { signature_id, verification }` quando o estado muda ou `explicit`; `signer` fora do ar → devolve o guardado, sem mudar).
  - `Signatures::Package.zip(signature) -> String` (`document.json` = os bytes assinados; `document.json.p7s`).
  - `Signatures::PrintReport::Report = Data(:lines, :hand_signature)` (`#hand_signature?`), `.for(consultation) -> Report|nil` (nil quando nenhuma parte tem pedido — o impresso do 19a, igual), `.signed_pdf(consultation) -> String|nil` (PAdES da consulta quando ela é `digital` e não tem adendo; revalida antes).
  - `Signatures::Json.signature(signature) -> Hash` (contrato §6).
  - Rotas: `GET /signature/signatures/:id`, `GET /signature/signatures/:id/pdf`, `GET /signature/signatures/:id/package`, `POST /signature/signatures/:id/verify` — atrás do `clinical_record` (não do `digital_signature`: o assinado continua visível), papel `health_professional`, `ClinicalRecord::Access` (abertura justificada vale; senão 403 `opening_required`, Desvio 15) e trilha `clinical_record.viewed` em toda leitura.
  - `GET /attendance/consultations/:id/print`: PDF assinado ou impresso com a seção "Assinaturas" (Desvio 13).
  - Helper: `sign_document!(document, author:) -> Signature` (certificado, sessão no PSC falso e `SignPending`).

- [ ] **Step 1: O helper de assinatura (acrescente ao módulo `SignatureHelpers`)**

```ruby
  # Assina de verdade (PSC falso + signer stubado pelo chamador) um documento finalizado.
  def sign_document!(document, author:)
    cpf = author.professional.cpf
    certificate = SignerCertificate.active.find_by(user_id: author.id) || linked_certificate!(author, leaf: fake_psc.leaf(cpf))
    SignatureSession.usable_for(author.id) ||
      signature_session!(author, certificate: certificate, token: fake_psc.token_for!(cpf: cpf))
    request = signature_request!(document, author: author)
    outcome = ApplicationRecord.transaction { Signatures::SignPending.call(request_id: request.id) }
    raise "não assinou: #{outcome} #{request.reload.reason_code}" unless outcome == :signed

    request.reload.signature
  end
```

- [ ] **Step 2: Escreva a spec que falha**

```ruby
# spec/requests/signature_reading_spec.rb
require "rails_helper"

# Contrato §2 e §6 (spec §5 validação, §7 exportação, §9 impresso; NGS2.03.01,
# 06.03): o modo de cada documento, o conteúdo assinado com trilha, a
# revalidação, o PDF, o pacote e o impresso.
RSpec.describe "Leitura da assinatura", type: :request do
  before do
    signature_city!
    ciap2_release!; cid10_release!; sigtap_release!
    stub_psc!
    @signer = stub_signer!
  end

  let(:unit) { create_unit }
  let(:doctor) { signer_doctor!(unit) }
  def body = JSON.parse(response.body)

  def in_context!(consultation) = consulting_attendance!(unit, citizen: consultation.attendance.citizen, doctor: doctor)
  def text_of(bytes) = PDF::Reader.new(StringIO.new(bytes)).pages.map(&:text).join(" ").gsub(/\s+/, " ")

  it "o bloco signature na consulta e no adendo: manual, pending e digital" do
    manual = finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1))
    pending = finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(2))
    pending_request = signature_request!(pending, author: doctor, reason_code: "no_session")
    signed = finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(3))
    signature = sign_document!(signed, author: doctor)
    addendum = Consultations::AddAddendum.call(consultation: signed, by: doctor, reason: "exame adicional pedido", text: "UM").payload[:addendum]

    expect(Consultations::Json.consultation(manual)[:signature]).to eq(mode: "manual")
    expect(Consultations::Json.consultation(pending)[:signature]).to eq(mode: "pending", request_id: pending_request.id, reason_code: "no_session")
    json = Consultations::Json.consultation(signed.reload)
    expect(json[:signature]).to eq(mode: "digital", request_id: signature.signature_request_id, signature_id: signature.id,
                                   signed_at: signature.signed_at.iso8601, signer_name: doctor.professional.professional_name,
                                   verification: "valid")
    expect(json[:addenda].sole[:signature]).to eq(mode: "manual")
    expect(Consultations::Json.addendum(addendum)[:signature]).to eq(mode: "manual")
    draft = started_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(4))
    expect(Consultations::Json.consultation(draft)).not_to have_key(:signature)
  end

  it "conteúdo assinado: forma do contrato, trilha, revalidação e evento; fora de contexto 403" do
    consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1))
    signature = sign_document!(consultation, author: doctor)
    in_context!(consultation)
    sign_in_as(doctor)
    get "/signature/signatures/#{signature.id}"
    expect(response).to have_http_status(:ok)
    expect(body.keys).to match_array(%w[id document_type document_id signed_at signer_name signer_cpf_masked policy verification
                                        verification_reasons verified_at content])
    expect(body).to include("document_type" => "consultation", "document_id" => consultation.id, "policy" => "AD-RB",
                            "signer_cpf_masked" => "***.982.247-**", "verification" => "valid")
    expect(body["content"]).to eq(JSON.parse(signature.canonical_json))
    expect(DomainEvent.where(name: "clinical_record.viewed").pluck(:payload).last).to include("patient_id" => consultation.patient_id)

    @signer.revoked_serials << signature.signer_certificate.serial_number
    post "/signature/signatures/#{signature.id}/verify"
    expect(body.values_at("verification", "verification_reasons")).to eq([ "invalid", [ "certificate_revoked" ] ])
    expect(DomainEvent.where(name: "signature.verified").pluck(:payload).last)
      .to eq("signature_id" => signature.id, "verification" => "invalid")
    expect(signature.reload.canonical_sha256).to eq(Digest::SHA256.hexdigest(signature.canonical_json)) # o registro não muda

    @signer.unavailable = true
    get "/signature/signatures/#{signature.id}"
    expect(body["verification"]).to eq("invalid") # o guardado, sem cair

    sign_in_as(signer_doctor!(create_unit("UBS Dois"), cpf: SignatureHelpers::OTHER_CPF))
    get "/signature/signatures/#{signature.id}"
    expect([ response.status, body["error"] ]).to eq([ 403, "opening_required" ])
  end

  it "PDF assinado e pacote .zip (JSON + .p7s), sem cache; visíveis com o interruptor desligado" do
    consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1))
    signature = sign_document!(consultation, author: doctor)
    in_context!(consultation)
    signature_city!(enabled: false)
    sign_in_as(doctor)
    get "/signature/signatures/#{signature.id}/pdf"
    expect([ response.status, response.media_type ]).to eq([ 200, "application/pdf" ])
    expect(response.body.b).to eq(signature.signed_pdf_bytes.b)
    expect(response.headers["Cache-Control"]).to include("no-store")
    get "/signature/signatures/#{signature.id}/package"
    expect(response.media_type).to eq("application/zip")
    entries = {}
    Zip::InputStream.open(StringIO.new(response.body)) do |zip|
      while (entry = zip.get_next_entry)
        entries[entry.name] = zip.read
      end
    end
    expect(entries.keys).to eq(%w[document.json document.json.p7s])
    expect(entries["document.json"].force_encoding("UTF-8")).to eq(signature.canonical_json)
    expect(entries["document.json.p7s"].b).to eq(signature.cades_bytes.b)
    expect(response.headers["Content-Disposition"]).not_to include(consultation.patient.display_name.to_s)
  end

  it "prontuário desligado: 403 do clinical_record; recepção: 403" do
    consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1))
    signature = sign_document!(consultation, author: doctor)
    sign_in_as(reception!)
    get "/signature/signatures/#{signature.id}"
    expect([ response.status, body["error"] ]).to eq([ 403, "missing_role" ])
    Platform::Features.set!(city: City.find_by(slug: TEST_CITY_A.slug), key: "clinical_record", enabled: false, maintainer: ledi_maintainer!)
    sign_in_as(doctor)
    get "/signature/signatures/#{signature.id}"
    expect(body).to eq("error" => "feature_disabled", "feature" => "clinical_record")
  end

  it "impresso: assinado → o PAdES; com adendo → seção Assinaturas; sem pedido nenhum → o impresso do 19a" do
    plain = finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1))
    in_context!(plain)
    sign_in_as(doctor)
    get "/attendance/consultations/#{plain.id}/print"
    expect(text_of(response.body)).to include("Assinatura e carimbo")
    expect(text_of(response.body)).not_to include("Assinaturas")

    signed = finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(2))
    signature = sign_document!(signed, author: doctor)
    in_context!(signed)
    get "/attendance/consultations/#{signed.id}/print"
    expect(response.body.b).to eq(signature.signed_pdf_bytes.b)

    Consultations::AddAddendum.call(consultation: signed, by: doctor, reason: "exame adicional pedido", text: "Adendo sem assinatura")
    get "/attendance/consultations/#{signed.id}/print"
    text = text_of(response.body)
    expect(text).to include("Assinaturas", "Consulta: assinada digitalmente", "Adendo de", "sem assinatura digital", "Assinatura e carimbo")
  end
end
```

Em `spec/services/consultations/json_spec.rb` (19a), no exemplo "finalizada…", troque `%w[id author_name created_at reason text changes]` por `%w[id author_name created_at reason text changes signature]` (contrato §2 do 19b). Se alguma spec de request do 19a fixa as chaves da consulta finalizada, acrescente `signature` do mesmo jeito.

- [ ] **Step 3: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod19b api bundle exec rspec spec/requests/signature_reading_spec.rb`
Expected: FAIL (`uninitialized constant Signatures::Mode` / `No route matches`).

- [ ] **Step 4: Implemente**

```ruby
# app/services/signatures/mode.rb
# Modo de cada documento (ADR 0032; contrato §2): manual (sem pedido, ou de
# volta ao papel), pending, digital. Calculado; as tabelas do 19a não mudam.
module Signatures
  module Mode
    module_function

    def blocks(documents)
      return {} if documents.empty?

      requests = SignatureRequest.where(document_id: documents.map(&:id)).index_by { |r| [ r.document_type, r.document_id ] }
      signatures = Signature.where(signature_request_id: requests.values.map(&:id))
                            .select(:id, :signature_request_id, :signed_at, :last_verification).index_by(&:signature_request_id)
      documents.to_h do |document|
        key = [ DocumentTypes.db(document), document.id ]
        [ key, block(requests[key], signatures, document) ]
      end
    end

    def for(document) = blocks([ document ]).values.first

    def block(request, signatures, document)
      return { mode: "manual" } unless request

      case request.status
      when "signed"
        signature = signatures[request.id]
        { mode: "digital", request_id: request.id, signature_id: signature&.id, signed_at: signature&.signed_at&.iso8601,
          signer_name: Screenings::Json.staff_name(document.author_user), verification: signature&.last_verification }.compact
      when "returned_to_paper" then { mode: "manual", request_id: request.id, reason_code: request.reason_code }
      else { mode: "pending", request_id: request.id, reason_code: request.reason_code }.compact
      end
    end
    private_class_method :block
  end
end
```

```ruby
# app/services/signatures/verify.rb
# Revalidação (ADR 0032; spec §5; NGS2.03.01: ao abrir, imprimir e sob
# demanda). Revogação posterior muda só o estado de validação; o registro não
# muda (trigger). signer fora do ar: devolve o estado guardado.
module Signatures
  module Verify
    module_function

    def call(signature, now: Time.current, signer: Signer.client, explicit: false)
      checks = [ signer.verify(kind: "cades", signature: signature.cades_bytes, document: signature.canonical_json),
                 signer.verify(kind: "pades", signature: signature.signed_pdf_bytes) ]
      status = combine(checks, signature)
      changed = status != signature.last_verification
      signature.update!(last_verification: status, last_verification_at: now,
                        last_verification_reasons: checks.flat_map(&:reasons).uniq)
      DomainEvents.publish("signature.verified", signature_id: signature.id, verification: status) if changed || explicit
      signature
    rescue Signer::Unavailable, Signer::Rejected
      signature
    end

    def combine(checks, signature)
      return "invalid" if checks.any? { |check| check.signer_cpf.present? && check.signer_cpf != signature.signer_cpf }
      return "valid" if checks.all?(&:valid?)
      return "invalid" if checks.any? { |check| check.status == "invalid" }

      "indeterminate"
    end
    private_class_method :combine
  end
end
```

```ruby
# app/services/signatures/package.rb
# Exportação validável fora do sistema (spec §7; NGS2.06.03): o JSON canônico
# exatamente como foi assinado e o CAdES destacado (.p7s) — o par que o
# validar.iti.gov.br aceita.
module Signatures
  module Package
    module_function

    def zip(signature)
      Zip::OutputStream.write_buffer(StringIO.new) do |zip|
        zip.put_next_entry("document.json")
        zip.write(signature.canonical_json)
        zip.put_next_entry("document.json.p7s")
        zip.write(signature.cades_bytes)
      end.string
    end
  end
end
```

```ruby
# app/services/signatures/print_report.rb
# O impresso da consulta (Desvio 13): consulta digital sem adendo → o próprio
# PAdES; com alguma parte assinada ou pendente → o impresso do 19a com a seção
# "Assinaturas" (NGS2.03.01/06.06: o estado sai na impressão) e espaço à mão só
# se alguma parte não é digital; nenhum pedido → o impresso do 19a, igual.
module Signatures
  module PrintReport
    VERIFICATION = { "valid" => "válida", "invalid" => "inválida", "indeterminate" => "indeterminada" }.freeze

    Report = Data.define(:lines, :hand_signature) do
      def hand_signature? = hand_signature
    end

    module_function

    def signed_pdf(consultation)
      return nil if consultation.addenda.exists?

      request = SignatureRequest.find_by(document_type: "Consultation", document_id: consultation.id, status: "signed")
      request && Verify.call(request.signature).signed_pdf_bytes
    end

    def for(consultation)
      return nil unless SignatureRequest.exists?(consultation_id: consultation.id)

      Signature.where(signature_request_id: SignatureRequest.where(consultation_id: consultation.id, status: "signed").select(:id))
               .find_each { |signature| Verify.call(signature) }
      addenda = consultation.addenda.order(:created_at, :id).to_a
      blocks = Mode.blocks([ consultation, *addenda ])
      parts = [ [ "Consulta", blocks[[ "Consultation", consultation.id ]] ] ] +
              addenda.map { |a| [ "Adendo de #{a.created_at.in_time_zone.strftime('%d/%m/%Y %H:%M')}", blocks[[ "ConsultationAddendum", a.id ]] ] }
      Report.new(lines: parts.map { |label, block| "#{label}: #{describe(block)}" },
                 hand_signature: parts.any? { |_label, block| block[:mode] != "digital" })
    end

    def describe(block)
      case block[:mode]
      when "digital"
        "assinada digitalmente por #{block[:signer_name]} em " \
          "#{Time.iso8601(block[:signed_at]).in_time_zone.strftime('%d/%m/%Y %H:%M')} — validação " \
          "#{VERIFICATION.fetch(block[:verification].to_s, block[:verification].to_s)}"
      when "pending" then "assinatura digital pendente — assinar à mão"
      else "sem assinatura digital — assinar à mão"
      end
    end
    private_class_method :describe
  end
end
```

Em `app/services/signatures/json.rb`, dentro do módulo:

```ruby
    # <signature> (contrato §6). O CPF só mascarado; o conteúdo é o JSON canônico.
    def signature(signature)
      { id: signature.id, document_type: DocumentTypes.api(signature.document_type), document_id: signature.document_id,
        signed_at: signature.signed_at.iso8601, signer_name: Screenings::Json.staff_name(signature.signature_request.author_user),
        signer_cpf_masked: CitizenIdentity::Cpf.mask(signature.signer_cpf), policy: signature.policy,
        verification: signature.last_verification, verification_reasons: signature.last_verification_reasons,
        verified_at: signature.last_verification_at.iso8601, content: JSON.parse(signature.canonical_json) }
    end
```

```ruby
# app/controllers/signatures/signatures_controller.rb
# Assinatura gravada (contrato §6): conteúdo legível, PDF, pacote e
# revalidação. Atrás do clinical_record (o assinado continua visível com o
# digital_signature desligado), papel health_professional, ClinicalRecord::Access
# e trilha clinical_record.viewed em toda leitura.
module Signatures
  class SignaturesController < BaseController
    include ClinicalRecordGate

    skip_before_action :require_digital_signature!
    before_action :require_clinical_record!
    before_action :set_signature

    def show = render(json: Json.signature(Verify.call(@signature)))

    def verify = render(json: Json.signature(Verify.call(@signature, explicit: true)))

    def pdf
      Verify.call(@signature)
      response.headers["Cache-Control"] = "no-store"
      send_data @signature.signed_pdf_bytes, type: "application/pdf", disposition: "attachment", filename: "documento-assinado.pdf"
    end

    def package
      response.headers["Cache-Control"] = "no-store"
      send_data Package.zip(@signature), type: "application/zip", disposition: "attachment", filename: "documento-assinado.zip"
    end

    private

    def set_signature
      @signature = Signature.find_by(id: params[:id])
      return render(json: { error: "not_found" }, status: :not_found) unless @signature

      patient = Consultation.find(@signature.signature_request.consultation_id).patient
      grant = ClinicalRecord::Access.call(user: Current.user, patient: patient)
      # Desvio 15: fora de contexto e sem abertura válida → opening_required (19a).
      return forbid(grant.reason == :out_of_context ? "opening_required" : grant.reason.to_s) unless grant.allowed?

      ClinicalRecord::Trail.viewed!(patient: patient, user: Current.user, grant: grant)
    end
  end
end
```

Em `config/routes.rb`, dentro do `scope "/signature"`:

```ruby
    get  "signatures/:id",         to: "signatures#show"
    get  "signatures/:id/pdf",     to: "signatures#pdf"
    get  "signatures/:id/package", to: "signatures#package"
    post "signatures/:id/verify",  to: "signatures#verify"
```

Em `app/services/consultations/json.rb`, troque `consultation` e `addendum` por:

```ruby
    def consultation(c)
      addenda_rows = c.addenda.order(:created_at, :id).to_a
      blocks = c.draft? ? {} : Signatures::Mode.blocks([ c, *addenda_rows ])
      json = { id: c.id, attendance_id: c.attendance_id, patient_id: c.patient_id, status: c.status,
               author: { id: c.author_user_id, name: Screenings::Json.staff_name(c.author_user) }, cbo_code: c.cbo_code,
               subjective: c.subjective, objective: c.objective, assessment: c.assessment, plan: c.plan, vitals: vitals(c),
               care_type: c.care_type, evaluated_problems: evaluated_problems(c), conducts: conducts(c),
               exam_requests: exam_requests(c), started_at: c.started_at.iso8601, finalized_at: c.finalized_at&.iso8601,
               addenda: addenda_rows.map { |a| addendum(a, signature: blocks[[ "ConsultationAddendum", a.id ]]) } }
      # ADR 0032 (contrato §2 do 19b): o modo da assinatura; rascunho não tem.
      json[:signature] = blocks[[ "Consultation", c.id ]] unless c.draft?
      json
    end

    def addendum(a, signature: :load)
      { id: a.id, author_name: Screenings::Json.staff_name(a.author_user), created_at: a.created_at.iso8601,
        reason: a.reason, text: a.text, changes: a.changes,
        signature: signature == :load ? Signatures::Mode.for(a) : signature }
    end
```

Em `app/controllers/consultations_controller.rb`, troque o corpo de `print` (depois da trilha) por:

```ruby
    ClinicalRecord::Trail.viewed!(patient: @consultation.patient, user: Current.user, grant: grant)
    response.headers["Cache-Control"] = "no-store"
    # ADR 0032 (Desvio 13): digital sem adendo → o próprio PAdES; senão o
    # impresso, com a seção de assinaturas quando houver pedido.
    bytes = Signatures::PrintReport.signed_pdf(@consultation) ||
            Consultations::Print.call(@consultation, report: Signatures::PrintReport.for(@consultation))
    send_data bytes, type: "application/pdf", disposition: "inline", filename: "consulta.pdf"
```

- [ ] **Step 5: Rode e veja passar (e as specs do 19a que leem a consulta e o impresso)**

Run: `docker compose exec -T -w /rails/.claude/mod19b api bundle exec rspec spec/requests/signature_reading_spec.rb spec/services/consultations spec/requests/consultations_spec.rb spec/requests/consultation_print_spec.rb spec/requests/clinical_record_spec.rb`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod19b add app/services/signatures/mode.rb app/services/signatures/verify.rb app/services/signatures/package.rb app/services/signatures/print_report.rb app/services/signatures/json.rb app/controllers/signatures/signatures_controller.rb app/services/consultations/json.rb app/controllers/consultations_controller.rb config/routes.rb spec/support/signature_helpers.rb spec/services/consultations/json_spec.rb spec/requests/signature_reading_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod19b commit -m "feat: show, revalidate and export signed consultations and addenda

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

(Inclua no `git add` qualquer spec de request do 19a que ganhou a chave `signature` no Step 2.)

---
### Task 15: Painel de assinatura do admin municipal

**Files:**
- Create: `app/services/signatures/admin_overview.rb`, `app/controllers/signatures/admin_controller.rb`
- Modify: `config/routes.rb`
- Test: `spec/requests/signature_admin_spec.rb`

**Interfaces:**
- Consumes: `SignerCertificate#expiring?`, `SignatureRequest`, `Signature` (Task 3), `CitizenVerificationPolicy#manage?` (existente; o mesmo critério do relatório de aberturas do 19a).
- Produces:
  - `Signatures::AdminOverview::LIMIT == 500`, `::INVALID_LIMIT == 200`, `.call(range:, now: Time.current) -> Hash` (contrato §7: `professionals` — profissionais ativos por nome, `certificate_status` `active`|`none`|`expiring` (≤ 30 dias), `not_after?`, `pending_count`, `oldest_pending_at?`; `documents_by_mode` — consultas finalizadas e adendos criados no período; `invalid_or_indeterminate` — as mais recentes).
  - `GET /signature/admin/overview?from=&to=` (`municipal_admin`; `AAAA-MM-DD` no fuso da cidade; padrão: os últimos 30 dias até hoje; 422 `invalid_period`; 403 `missing_role`; 403 `feature_disabled`).

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/requests/signature_admin_spec.rb
require "rails_helper"

# Contrato §7 (spec §9, painel do admin; NGS2.02.05/06): quem tem certificado,
# quem vence em 30 dias, pendentes e o mais antigo, documentos por modo no
# período, assinaturas inválidas ou indeterminadas. Só leitura.
RSpec.describe "Painel de assinatura do admin", type: :request do
  before do
    signature_city!
    ciap2_release!; cid10_release!; sigtap_release!
  end

  let(:unit) { create_unit }
  let(:doctor) { signer_doctor!(unit) }
  def body = JSON.parse(response.body)

  it "profissionais, documentos por modo e assinaturas inválidas" do
    certificate = linked_certificate!(doctor)
    expiring = signer_doctor!(create_unit("UBS Dois"), cpf: SignatureHelpers::OTHER_CPF)
    linked_certificate!(expiring, leaf: test_pki.issue(cpf: SignatureHelpers::OTHER_CPF, name: "VENCE LOGO", not_after: 10.days.from_now))
    without = doctor!(create_unit("UBS Tres"))

    signed = finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1))
    pending = finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(2))
    manual = finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(3))
    signed_request = signature_request!(signed, author: doctor, status: "signed")
    pending_request = signature_request!(pending, author: doctor, reason_code: "no_session")
    Consultations::AddAddendum.call(consultation: manual, by: doctor, reason: "adendo sem pedido", text: "x")
    signature_row!(signed_request, certificate: certificate)
      .update!(last_verification: "indeterminate", last_verification_at: Time.current, last_verification_reasons: [ "crl_unavailable" ])

    sign_in_as(ledi_admin!)
    get "/signature/admin/overview"
    expect(response).to have_http_status(:ok)
    rows = body["professionals"].index_by { |row| row["user_id"] }
    expect(rows[doctor.id]).to include("certificate_status" => "active", "pending_count" => 1,
                                       "oldest_pending_at" => pending_request.created_at.iso8601)
    expect(rows[expiring.id]).to include("certificate_status" => "expiring", "pending_count" => 0)
    expect(rows[without.id]).to include("certificate_status" => "none")
    expect(rows[without.id]).not_to have_key("not_after")
    expect(body["documents_by_mode"]).to eq("digital" => 1, "pending" => 1, "manual" => 2) # manual: 1 consulta + 1 adendo
    expect(body["invalid_or_indeterminate"].sole)
      .to include("document_type" => "consultation", "verification" => "indeterminate",
                  "signer_name" => doctor.professional.professional_name)
  end

  it "período: fora dele não conta; data inválida 422; não admin 403; interruptor desligado 403" do
    travel_to(40.days.ago) { finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1)) }
    sign_in_as(ledi_admin!)
    get "/signature/admin/overview"
    expect(body["documents_by_mode"]).to eq("digital" => 0, "pending" => 0, "manual" => 0)
    get "/signature/admin/overview", params: { from: (Time.zone.today - 60).iso8601, to: Time.zone.today.iso8601 }
    expect(body["documents_by_mode"]["manual"]).to eq(1)
    get "/signature/admin/overview", params: { from: "2026-13-01" }
    expect([ response.status, body["error"] ]).to eq([ 422, "invalid_period" ])
    sign_in_as(doctor)
    get "/signature/admin/overview"
    expect([ response.status, body["error"] ]).to eq([ 403, "missing_role" ])
    signature_city!(enabled: false)
    sign_in_as(ledi_admin!)
    get "/signature/admin/overview"
    expect(body["error"]).to eq("feature_disabled")
  end
end
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod19b api bundle exec rspec spec/requests/signature_admin_spec.rb`
Expected: FAIL (`No route matches [GET] "/signature/admin/overview"`).

- [ ] **Step 3: Implemente**

```ruby
# app/services/signatures/admin_overview.rb
# Painel do admin municipal (ADR 0032; spec §9; contrato §7). Só leitura; sem
# CPF, sem conteúdo clínico. Nenhuma coluna cifrada é lida.
module Signatures
  module AdminOverview
    LIMIT = 500
    INVALID_LIMIT = 200

    module_function

    def call(range:, now: Time.current)
      { professionals: professionals(now), documents_by_mode: documents_by_mode(range), invalid_or_indeterminate: invalid }
    end

    def professionals(now)
      certificates = SignerCertificate.active.select(:id, :user_id, :not_after).index_by(&:user_id)
      pending = SignatureRequest.pending.group(:author_user_id)
                                .pluck(:author_user_id, Arel.sql("count(*)"), Arel.sql("min(created_at)"))
                                .to_h { |user_id, count, oldest| [ user_id, [ count, oldest ] ] }
      Professional.joins(:user).where(users: { deactivated_at: nil }).order(:professional_name).limit(LIMIT).map do |professional|
        certificate = certificates[professional.user_id]
        count, oldest = pending[professional.user_id]
        { user_id: professional.user_id, name: professional.professional_name,
          certificate_status: certificate_status(certificate, now), not_after: certificate&.not_after&.iso8601,
          pending_count: count.to_i, oldest_pending_at: oldest&.iso8601 }.compact
      end
    end

    def certificate_status(certificate, now)
      return "none" unless certificate

      certificate.expiring?(now) ? "expiring" : "active"
    end

    def documents_by_mode(range)
      consultation_ids = Consultation.where(status: "finalized", finalized_at: range).pluck(:id)
      addendum_ids = ConsultationAddendum.where(created_at: range).pluck(:id)
      statuses = SignatureRequest.where(document_type: "Consultation", document_id: consultation_ids)
                                 .or(SignatureRequest.where(document_type: "ConsultationAddendum", document_id: addendum_ids))
                                 .group(:status).count
      digital = statuses.fetch("signed", 0)
      pending = statuses.fetch("pending", 0) + statuses.fetch("failed", 0)
      { digital: digital, pending: pending, manual: consultation_ids.size + addendum_ids.size - digital - pending }
    end

    def invalid
      Signature.where.not(last_verification: "valid").order(last_verification_at: :desc).limit(INVALID_LIMIT)
               .select(:id, :document_type, :signature_request_id, :last_verification, :last_verification_at)
               .includes(signature_request: :author_user).map do |signature|
        { signature_id: signature.id, document_type: DocumentTypes.api(signature.document_type),
          signer_name: Screenings::Json.staff_name(signature.signature_request.author_user),
          verification: signature.last_verification, verified_at: signature.last_verification_at.iso8601 }
      end
    end
    private_class_method :professionals, :certificate_status, :documents_by_mode, :invalid
  end
end
```

```ruby
# app/controllers/signatures/admin_controller.rb
# Painel do admin municipal (contrato §7): municipal_admin, só leitura.
module Signatures
  class AdminController < BaseController
    DATE = /\A\d{4}-\d{2}-\d{2}\z/
    DEFAULT_DAYS = 30

    skip_before_action :require_professional
    before_action :require_manager

    def overview
      range = period
      return render(json: { error: "invalid_period" }, status: :unprocessable_entity) unless range

      render json: AdminOverview.call(range: range)
    end

    private

    def require_manager
      forbid("missing_role") unless CitizenVerificationPolicy.new(Current.user, nil).manage?
    end

    # from/to "AAAA-MM-DD" no fuso da cidade (Time.zone dentro do request).
    def period
      from = date(params[:from])
      to = date(params[:to])
      return nil if from == :invalid || to == :invalid

      finish = (to || Time.zone.today).in_time_zone.end_of_day
      start = (from || (finish.to_date - (DEFAULT_DAYS - 1))).in_time_zone.beginning_of_day
      start <= finish ? start..finish : nil
    end

    def date(value)
      return nil if value.blank?
      return :invalid unless value.is_a?(String) && value.match?(DATE)

      Date.iso8601(value)
    rescue Date::Error
      :invalid
    end
  end
end
```

Em `config/routes.rb`, dentro do `scope "/signature"`: `get "admin/overview", to: "admin#overview"`.

- [ ] **Step 4: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod19b api bundle exec rspec spec/requests/signature_admin_spec.rb`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod19b add app/services/signatures/admin_overview.rb app/controllers/signatures/admin_controller.rb config/routes.rb spec/requests/signature_admin_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod19b commit -m "feat: add the municipal admin digital signature overview

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 16: Leitura na API de manutenção — prestadores e estado do `signer`

**Files:**
- Create: `app/graphql/maintenance/types/signature_provider_type.rb`, `app/graphql/maintenance/types/signer_status_type.rb`, `app/services/signatures/signer_status.rb`
- Modify: `app/graphql/maintenance/types/query_type.rb`, `spec/architecture/maintenance_schema_spec.rb`
- Test: `spec/requests/maintenance/signature_status_spec.rb`

**Interfaces:**
- Consumes: `Signatures::Providers.{configured?,checks}`, `Signatures::Providers::CATALOG` (Task 4), `Signatures::Signer.client#health` (Task 5).
- Produces:
  - `Signatures::SignerStatus::Row = Data(:reachable, :version, :crl_updated_at)`, `.call(signer: Signer.client) -> Row` (fora do ar → `reachable: false`, resto nulo).
  - GraphQL (plataforma, sem abrir cidade, sem segredo): `Query.signatureProviders: [SignatureProvider!]!` (`key String!`, `configured Boolean!`, `lastCheckAt ISO8601DateTime`, `lastCheckOk Boolean`), `Query.signerStatus: SignerStatus!` (`reachable Boolean!`, `version String`, `crlUpdatedAt ISO8601DateTime`) — contrato §8. O interruptor `digital_signature` já aparece em `City.features` pelo catálogo (Task 1).

- [ ] **Step 1: Escreva a spec que falha**

```ruby
# spec/requests/maintenance/signature_status_spec.rb
require "rails_helper"

# Contrato §8 (spec §3, §9 maintenance): prestadores do ambiente (só se há
# credencial e a última checagem) e o estado do signer. Nunca segredo, URL ou
# CPF; nenhuma cidade é aberta.
RSpec.describe "Maintenance: assinatura digital", type: :request do
  let(:frontend) { "https://maintenance.rotasaude.app" }
  let(:password) { "s3nha-forte-1" }
  let!(:maintainer) do
    Maintainer.create!(email_address: "sig-#{SecureRandom.hex(3)}@rotasaude.app", password: password,
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
    allow(Signatures::Providers).to receive(:credentials).and_return(
      "vidaas" => { "client_id" => "id", "client_secret" => "SEGREDO-PSC", "base_url" => "https://psc.test" }
    )
    Signatures::Providers.record_check!("vidaas", ok: true)
    login!
  end

  it "prestadores do catálogo com credencial presente e última checagem; nada de segredo" do
    expect(Maintenance::CityReader).not_to receive(:call)
    gql!("{ signatureProviders { key configured lastCheckAt lastCheckOk } }")
    rows = json.dig("data", "signatureProviders")
    expect(rows.map { |row| row["key"] }).to eq(%w[vidaas birdid safeid neoid remoteid])
    expect(rows.first).to include("configured" => true, "lastCheckOk" => true)
    expect(rows.first["lastCheckAt"]).to be_present
    expect(rows.second).to eq("key" => "birdid", "configured" => false, "lastCheckAt" => nil, "lastCheckOk" => nil)
    expect(response.body).not_to include("SEGREDO-PSC", "psc.test")
  end

  it "estado do signer: alcançável com versão e LCR; fora do ar sem cair a consulta" do
    stub_signer!
    gql!("{ signerStatus { reachable version crlUpdatedAt } }")
    expect(json.dig("data", "signerStatus")).to include("reachable" => true, "version" => "fake-1")
    stub_signer!.unavailable = true
    gql!("{ signerStatus { reachable version crlUpdatedAt } }")
    expect(json.dig("data", "signerStatus")).to eq("reachable" => false, "version" => nil, "crlUpdatedAt" => nil)
  end
end
```

Em `spec/architecture/maintenance_schema_spec.rb`: em `EXPECTED_TYPES`, acrescente `signatureProviders signerStatus` à lista de `"Query"` e as entradas

```ruby
    "SignatureProvider" => %w[key configured lastCheckAt lastCheckOk],
    "SignerStatus" => %w[reachable version crlUpdatedAt],
```

e, em `forbidden_name_exempt_fields`, acrescente `SignatureProvider.key` com o comentário:

```ruby
  # ADR 0032: `key` do prestador é a chave do catálogo em código ("vidaas"),
  # não uma chave criptográfica — o par exato, nunca o nome.
```

- [ ] **Step 2: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod19b api bundle exec rspec spec/requests/maintenance/signature_status_spec.rb spec/architecture/maintenance_schema_spec.rb`
Expected: FAIL (`Field 'signatureProviders' doesn't exist on type 'Query'`; tipos declarados que o schema não publica).

- [ ] **Step 3: Implemente**

```ruby
# app/services/signatures/signer_status.rb
# Estado do serviço signer para o maintenance (contrato §8). Fora do ar nunca
# derruba a consulta.
module Signatures
  module SignerStatus
    Row = Data.define(:reachable, :version, :crl_updated_at)

    module_function

    def call(signer: Signer.client)
      health = signer.health
      Row.new(reachable: true, version: health[:version].presence, crl_updated_at: health[:crl_updated_at])
    rescue Signer::Error
      Row.new(reachable: false, version: nil, crl_updated_at: nil)
    end
  end
end
```

```ruby
# app/graphql/maintenance/types/signature_provider_type.rb
module Maintenance
  module Types
    class SignatureProviderType < BaseObject
      description "PSC de assinatura em nuvem da plataforma (ADR 0032; contrato §8): a chave do catálogo, se há " \
                  "credencial no ambiente e a última conversa. Nunca segredo, URL ou resposta do PSC."

      field :key, String, null: false
      field :configured, Boolean, null: false
      field :last_check_at, GraphQL::Types::ISO8601DateTime, null: true
      field :last_check_ok, Boolean, null: true
    end
  end
end
```

```ruby
# app/graphql/maintenance/types/signer_status_type.rb
module Maintenance
  module Types
    class SignerStatusType < BaseObject
      description "Serviço interno signer (ADR 0032; contrato §8): alcançável, versão e última atualização das LCRs."

      field :reachable, Boolean, null: false
      field :version, String, null: true
      field :crl_updated_at, GraphQL::Types::ISO8601DateTime, null: true
    end
  end
end
```

Em `app/graphql/maintenance/types/query_type.rb`, depois de `field :city`:

```ruby
      # ADR 0032 (contrato §8): plataforma, sem abrir cidade, sem segredo.
      field :signature_providers, [ Types::SignatureProviderType ], null: false,
            description: "PSC de assinatura do ambiente: credencial presente e última checagem"
      field :signer_status, Types::SignerStatusType, null: false,
            description: "Estado do serviço interno de assinatura (signer)"
```

e os resolvers, junto dos outros:

```ruby
      ProviderRow = Data.define(:key, :configured, :last_check_at, :last_check_ok)

      def signature_providers
        checks = Signatures::Providers.checks
        Signatures::Providers::CATALOG.map do |key|
          check = checks[key]
          ProviderRow.new(key: key, configured: Signatures::Providers.configured?(key), last_check_at: check&.last_check_at,
                          last_check_ok: check&.last_check_ok)
        end
      end

      def signer_status = Signatures::SignerStatus.call
```

- [ ] **Step 4: Rode e veja passar (e a API de manutenção inteira)**

Run: `docker compose exec -T -w /rails/.claude/mod19b api bundle exec rspec spec/requests/maintenance spec/architecture/maintenance_schema_spec.rb spec/graphql`
Expected: PASS. Se um analisador (`app/graphql/maintenance/analyzers/*`) lista campos de raiz por classe (só humano, escopo de escrita, orçamento de cidade), declare os dois como **leitura de plataforma** — o mesmo tratamento de `cities` — e acrescente o arquivo ao commit. Se o projeto guarda o SDL da API de manutenção fora do api (ADR-0015), avise o plano do maintenance (o SDL ganha os dois campos).

- [ ] **Step 5: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod19b add app/services/signatures/signer_status.rb app/graphql/maintenance/types/signature_provider_type.rb app/graphql/maintenance/types/signer_status_type.rb app/graphql/maintenance/types/query_type.rb spec/architecture/maintenance_schema_spec.rb spec/requests/maintenance/signature_status_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod19b commit -m "feat: expose signature providers and signer status to the maintenance API

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---
## Fatia 5 — Invariantes, semente, prova e fechamento (F-19.8 a F-19.14)

### Task 17: Suíte de invariantes do ADR 0032

**Files:**
- Test: `spec/invariants/digital_signature_invariants_spec.rb`

**Interfaces:**
- Consumes: tudo das Tasks 1–16; `capture_log` (19a), `sign_document!`, `stub_psc!`, `stub_signer!`, `fake_psc` (Tasks 3–14).

- [ ] **Step 1: Escreva a suíte**

```ruby
# spec/invariants/digital_signature_invariants_spec.rb
require "rails_helper"

# ADR 0032, "Invariantes": cada linha é um exemplo. Esta suíte não traz código
# novo: prova que as tasks anteriores, juntas, seguram o que o ADR promete.
RSpec.describe "Invariantes do ADR 0032" do
  include ActiveJob::TestHelper

  before do
    Current.city = signature_city!
    ciap2_release!; cid10_release!; sigtap_release!
    stub_psc!
    @signer = stub_signer!
  end
  after { Current.reset }

  let(:unit) { create_unit }
  let(:doctor) { signer_doctor!(unit) }
  let(:cpf) { SignatureHelpers::DOCTOR_CPF }

  def consume!(name)
    DomainEvent.where(name: name).order(:occurred_at).each do |event|
      Signatures::RequestJob.perform_now(event_id: event.id, event_name: event.name, city_slug: Current.city.slug, payload: event.payload)
    end
  end

  it "finalizar nunca espera a assinatura: PSC e signer fora do ar; o pedido fica pending com o motivo (Review Focus 3)" do
    certificate = linked_certificate!(doctor, leaf: fake_psc.leaf(cpf))
    signature_session!(doctor, certificate: certificate, token: fake_psc.token_for!(cpf: cpf))
    fake_psc.failures.push(*Array.new(10, 503))
    consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1))
    expect(consultation.reload).to be_finalized
    consume!("consultation.finalized")
    request = SignatureRequest.sole
    3.times { Signatures::SignJob.perform_now(city_slug: Current.city.slug, request_id: request.id) }
    expect(request.reload).to have_attributes(status: "pending", reason_code: "provider_unavailable", attempts: 3)
  end

  it "assinatura gravada não muda; revogação posterior muda só a validação" do
    consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1))
    signature = sign_document!(consultation, author: doctor)
    before = signature.slice(:canonical_sha256, :pdf_sha256, :signed_at)
    @signer.revoked_serials << signature.signer_certificate.serial_number
    Signatures::Verify.call(signature, explicit: true)
    expect(signature.reload.last_verification).to eq("invalid")
    expect(signature.slice(:canonical_sha256, :pdf_sha256, :signed_at)).to eq(before)
    expect { attempt { signature.update!(signed_at: 1.day.ago) } }.to raise_error(ActiveRecord::StatementInvalid, /never changes/)
  end

  it "só o autor assina o próprio documento, com o certificado do próprio CPF" do
    other_unit = create_unit("UBS Dois")
    other = signer_doctor!(other_unit, cpf: SignatureHelpers::OTHER_CPF)
    other_certificate = linked_certificate!(other, leaf: fake_psc.leaf(SignatureHelpers::OTHER_CPF))
    signature_session!(other, certificate: other_certificate, token: fake_psc.token_for!(cpf: SignatureHelpers::OTHER_CPF))
    linked_certificate!(doctor, leaf: fake_psc.leaf(cpf)) # sem sessão
    request = signature_request!(finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1)), author: doctor)

    expect(ApplicationRecord.transaction { Signatures::SignPending.call(request_id: request.id) }).to eq(:pending)
    expect(request.reload.reason_code).to eq("no_session") # a sessão do outro não serve
    token = Signatures::Psc::Token.new(access_token: fake_psc.token_for!(cpf: SignatureHelpers::OTHER_CPF, scope: "multi_signature"),
                                       expires_in: 300, scope: "multi_signature")
    batch = Signatures::RunBatch.call(user: other, request_ids: [ request.id ], token: token)
    expect(batch.payload[:record]).to eq(signed: 0, failed: [])
    expect(Signatures::ReturnToPaper.call(request_id: request.id, by: other, reason: "não é meu documento").reason).to eq(:not_author)
    expect(Signature.count).to eq(0)
  end

  it "nenhum documento é assinado duas vezes" do
    consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1))
    signature = sign_document!(consultation, author: doctor)
    expect(Signatures::OpenRequest.call(consultation)).to eq(:exists)
    expect { attempt { signature_row!(signature.signature_request, certificate: signature.signer_certificate) } }
      .to raise_error(ActiveRecord::RecordNotUnique)
    expect(ApplicationRecord.transaction { Signatures::SignPending.call(request_id: signature.signature_request_id) }).to eq(:skipped)
  end

  it "token, code, code_verifier, CPF e texto clínico fora de log, evento e erro" do
    codes = []
    log = capture_log do
      link = Signatures::StartLink.call(user: doctor, provider: "vidaas", return_to: "/conta")
      url = link.payload[:authorize_url]
      codes << authorize_and_approve!(url)
      state = URI.decode_www_form(URI(url).query).to_h["state"]
      expect(Signatures::CompleteOauth.call(user: doctor, state: state, code: codes.last)).to be_ok
      session = Signatures::StartSession.call(user: doctor, return_to: "/fila")
      url = session.payload[:authorize_url]
      codes << authorize_and_approve!(url)
      state = URI.decode_www_form(URI(url).query).to_h["state"]
      expect(Signatures::CompleteOauth.call(user: doctor, state: state, code: codes.last)).to be_ok
      consultation = finalized_consultation!(unit: unit, doctor: doctor, citizen: verified_citizen!(1))
      request = signature_request!(consultation, author: doctor)
      expect(ApplicationRecord.transaction { Signatures::SignPending.call(request_id: request.id) }).to eq(:signed)
    end
    verifiers = SignatureOauthState.all.map(&:code_verifier)
    secrets = fake_psc.issued_tokens + codes + verifiers + [ cpf, FakePsc::App::CLIENT_SECRET, "Refere sede" ]
    expect(secrets.select { |secret| log.include?(secret) }).to be_empty
    payloads = DomainEvent.where("name LIKE 'signature.%'").pluck(:payload)
    expect(payloads.map(&:to_json).join).not_to include(*secrets)
    allowed = %w[certificate_id user_id provider session_id signature_id request_id document_type document_id reason_code verification]
    expect(payloads.flat_map(&:keys).uniq - allowed).to be_empty
  end

  it "as tabelas do 19a não mudam" do
    migration = File.read(Dir[Rails.root.join("db/city_migrate/20261008500001_*.rb")].sole)
    tables = /:(consultations|consultation_\w+|patients|patient_\w+|clinical_record_openings|citizens)\b/
    expect(migration.scan(/(?:add_column|remove_column|change_column|change_table|rename_column|add_reference)\s+#{tables}/)).to be_empty
  end
end
```

- [ ] **Step 2: Rode**

Run: `docker compose exec -T -w /rails/.claude/mod19b api bundle exec rspec spec/invariants/digital_signature_invariants_spec.rb`
Expected: PASS. Um exemplo vermelho aqui é defeito de uma task anterior: corrija lá (com o teste da task) e volte; nunca afrouxe a expectativa.

- [ ] **Step 3: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod19b add spec/invariants/digital_signature_invariants_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod19b commit -m "test: pin the ADR 0032 digital signature invariants

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 18: Semente de dev, PSC simulado e o `signer` no compose

**Proposta de dev (spec §11, brief):** em development, um **PSC simulado local** (o mesmo `FakePsc::App` das specs, num serviço `fake-psc` do compose) + o **`signer` real** com a AC de teste do `DevPki` dele (volume `signer-dev-pki`). Assim o fluxo inteiro roda no navegador sem certificado real: localizar → vincular (página "PSC SIMULADO" com Aprovar/Recusar) → abrir a sessão do turno → finalizar → assinado em CAdES/PAdES AD-RB de verdade, validado pelo `signer`. O validar.iti.gov.br **não** aceita esta AC (ela não é ICP-Brasil): a prova contra o ITI é a do sandbox do VIDaaS (Task 0 do plano do `signer`).

**Files:**
- Create: `lib/fake_psc/config.ru`, `lib/digital_signature_crew.rb`
- Modify: `config/environments/development.rb`, `db/seeds.rb`, `docker-compose.yml` (raiz do monorepo — fora do git; mudança local, com aviso ao usuário)
- Test: `spec/lib/digital_signature_crew_spec.rb`, `spec/lib/fake_psc_rackup_spec.rb`

**Interfaces:**
- Consumes: `Platform::Features.set!`, `FakePsc::{Pki,App}`, `Signatures::Providers` (Tasks 1–4).
- Produces:
  - `config.x.signature_dev_providers` (só development): `vidaas` → `client_id: "rota-dev"`, `client_secret: "dev-secret"`, `base_url: ENV["FAKE_PSC_URL"] || "http://fake-psc:8091"`, `authorize_base_url: ENV["FAKE_PSC_PUBLIC_URL"] || "http://localhost:8091"`.
  - `lib/fake_psc/config.ru` (Rack; lê `SIGNER_DEV_PKI_DIR` e `FAKE_PSC_ABSENT_CPFS`).
  - `DigitalSignatureCrew.seed_current_city(slug:) -> { switch: String, professionals: Array<String> }` (liga `digital_signature` com o mantenedor de dev `dev@local`; sem ele, avisa e não liga; garante CPF válido nos profissionais da semente que não têm).

- [ ] **Step 1: O compose (com autorização do usuário; arquivo fora do git)**

O plano do `signer` cria o serviço `signer` (porta interna 8090, `SIGNER_TOKEN`, `SIGNER_DEV_PKI_DIR`, volume `signer-dev-pki`). Confira com `grep -n "signer" docker-compose.yml`. Depois **peça autorização** e acrescente ao `docker-compose.yml` da raiz:

no `x-api-env` (o mesmo bloco de env do `api` e do `worker`):

```yaml
  # ADR 0032: serviço interno de assinatura e o PSC simulado de dev.
  SIGNER_URL: ${SIGNER_URL:-http://signer:8090}
  SIGNER_TOKEN: ${SIGNER_TOKEN:-dev-signer-token-com-pelo-menos-32-caracteres}
  SIGNER_DEV_PKI_DIR: /signer-dev-pki
  FAKE_PSC_URL: ${FAKE_PSC_URL:-http://fake-psc:8091}
  FAKE_PSC_PUBLIC_URL: ${FAKE_PSC_PUBLIC_URL:-http://localhost:8091}
```

(o `SIGNER_TOKEN` tem de ser o mesmo do serviço `signer`; use a variável que o plano do `signer` fixou), em `api` e `worker`, na lista `volumes`:

```yaml
      - signer-dev-pki:/signer-dev-pki:ro
```

e o serviço novo:

```yaml
  fake-psc:
    container_name: fake-psc-dev
    image: rota-saude/api:dev
    # Enquanto o 19b não está na main, aponte para o worktree: /rails/.claude/mod19b
    working_dir: /rails
    command: bundle exec puma -b tcp://0.0.0.0:8091 lib/fake_psc/config.ru
    environment:
      SIGNER_DEV_PKI_DIR: /signer-dev-pki
      FAKE_PSC_ABSENT_CPFS: ${FAKE_PSC_ABSENT_CPFS:-}
    volumes:
      - ./apps/api:/rails
      - signer-dev-pki:/signer-dev-pki:ro
    ports:
      - "8091:8091"
    depends_on:
      signer:
        condition: service_started
    restart: unless-stopped
```

Run: `docker compose up -d signer fake-psc && curl -s -o /dev/null -w "%{http_code}\n" "http://localhost:8091/v0/oauth/authorize"`
Expected: `400` (o PSC simulado responde; sem parâmetros é `invalid_request`).

- [ ] **Step 2: Escreva as specs que falham**

```ruby
# spec/lib/digital_signature_crew_spec.rb
require "rails_helper"
require Rails.root.join("lib/digital_signature_crew")

# Semente de dev (Task 18): liga o interruptor com o mantenedor de dev e dá CPF
# válido a quem não tem; sem o mantenedor, avisa e não liga.
RSpec.describe DigitalSignatureCrew do
  before { Current.city = clinical_city! }
  after { Current.reset }

  it "sem o mantenedor de dev, não liga" do
    expect(described_class.seed_current_city(slug: Current.city.slug)[:switch]).to include("dev@local")
    expect(Platform::Features.enabled?(Current.city, "digital_signature")).to be(false)
  end

  it "com o mantenedor de dev, liga; profissional sem CPF ganha um válido" do
    Maintainer.create!(email_address: "dev@local", password: "dev-password", otp_secret: ROTP::Base32.random, otp_enabled_at: Time.current)
    doctor = doctor!(create_unit)
    doctor.professional.update!(cpf: nil)
    result = described_class.seed_current_city(slug: Current.city.slug)
    expect(result[:switch]).to eq("ligado")
    expect(Signatures::Gate.usable?(Current.city)).to be(true)
    expect(CitizenIdentity::Cpf.normalize(doctor.professional.reload.cpf)).to be_present
  end
end
```

```ruby
# spec/lib/fake_psc_rackup_spec.rb
require "rails_helper"
require "rack"

# O serviço fake-psc do compose sobe do config.ru com a AC do diretório dado.
RSpec.describe "lib/fake_psc/config.ru" do
  it "monta o PSC simulado e responde à localização por CPF" do
    allow(ENV).to receive(:fetch).and_call_original
    allow(ENV).to receive(:fetch).with("SIGNER_DEV_PKI_DIR").and_return(FakePsc::Pki::FIXTURES)
    app = Rack::Builder.parse_file(Rails.root.join("lib/fake_psc/config.ru").to_s)
    app = app.first if app.is_a?(Array) # Rack 2 devolve [app, options]
    body = { client_id: FakePsc::App::CLIENT_ID, client_secret: FakePsc::App::CLIENT_SECRET, user_cpf_cnpj: "CPF",
             val_cpf_cnpj: SignatureHelpers::DOCTOR_CPF }.to_json
    status, _headers, response = app.call(Rack::MockRequest.env_for("/v0/oauth/user-discovery", method: "POST", input: body))
    expect(status).to eq(200)
    expect(JSON.parse(response.join)["status"]).to eq("S")
  end
end
```

- [ ] **Step 3: Rode e veja falhar**

Run: `docker compose exec -T -w /rails/.claude/mod19b api bundle exec rspec spec/lib/digital_signature_crew_spec.rb spec/lib/fake_psc_rackup_spec.rb`
Expected: FAIL (arquivos não existem).

- [ ] **Step 4: Implemente**

```ruby
# lib/fake_psc/config.ru
# PSC SIMULADO do compose de dev (ADR 0032; Task 18). Nunca em produção: a
# imagem de produção não sobe este arquivo. e-CPF de teste da AC de dev do
# signer (volume signer-dev-pki); FAKE_PSC_ABSENT_CPFS (vírgulas) simula
# profissional sem certificado.
require_relative "app"

pki = FakePsc::Pki.load(ENV.fetch("SIGNER_DEV_PKI_DIR"))
app = FakePsc::App.new(pki: pki)
app.absent_cpfs = ENV.fetch("FAKE_PSC_ABSENT_CPFS", "").split(",").map(&:strip).reject(&:empty?)
run app
```

Em `config/environments/development.rb`, dentro do bloco `configure`:

```ruby
  # ADR 0032 (Desvio 14): o PSC simulado do compose (lib/fake_psc) responde
  # como vidaas em dev. As credenciais cifradas, se houver, vencem.
  config.x.signature_dev_providers = {
    "vidaas" => { "client_id" => "rota-dev", "client_secret" => "dev-secret",
                  "base_url" => ENV.fetch("FAKE_PSC_URL", "http://fake-psc:8091"),
                  "authorize_base_url" => ENV.fetch("FAKE_PSC_PUBLIC_URL", "http://localhost:8091") }
  }
```

```ruby
# lib/digital_signature_crew.rb
# Semente de dev da assinatura digital (ADR 0032; Task 18). Dev é fictício mas
# imita o real: liga o interruptor (o prontuário já vem ligado pela semente do
# 19a) e garante CPF com dígito válido nos profissionais que não têm. O vínculo
# do certificado e a sessão do turno são feitos no navegador (a prova do fluxo).
module DigitalSignatureCrew
  DEV_MAINTAINER = "dev@local".freeze

  module_function

  def seed_current_city(slug:)
    city = City.find_by!(slug: slug)
    maintainer = Maintainer.find_by(email_address: DEV_MAINTAINER)
    return { switch: "não ligado: mantenedor de dev (#{DEV_MAINTAINER}) ausente", professionals: [] } unless maintainer

    Platform::Features.set!(city: city, key: "digital_signature", enabled: true, maintainer: maintainer)
    professionals = CityConnection.with(city) do
      Professional.where(cpf: nil).find_each.with_index.map do |professional, index|
        professional.update!(cpf: cpf_from(900_000_000 + index))
        professional.professional_name
      end
    end
    missing = Platform::Features.missing(city, "digital_signature")
    { switch: missing.empty? ? "ligado" : "ligado, falta: #{missing.join(', ')}", professionals: professionals }
  end

  # CPF fictício com dígitos verificadores válidos a partir de 9 dígitos.
  def cpf_from(base)
    digits = format("%09d", base).chars.map(&:to_i)
    2.times do
      weights = (digits.size + 1).downto(2).to_a
      sum = digits.zip(weights).sum { |digit, weight| digit * weight }
      digits << ((sum * 10) % 11) % 10
    end
    digits.join
  end
end
```

Em `db/seeds.rb`: acrescente `require_relative "../lib/digital_signature_crew"` junto dos outros `require_relative` das crews e, logo depois da chamada `ClinicalRecordCrew.seed_current_city(...)` do 19a (mesmo bloco, mesma cidade), `puts "[assinatura digital] #{DigitalSignatureCrew.seed_current_city(slug: <a mesma variável de slug usada na chamada do 19a>).inspect}"`.

- [ ] **Step 5: Rode e veja passar**

Run: `docker compose exec -T -w /rails/.claude/mod19b api bundle exec rspec spec/lib/digital_signature_crew_spec.rb spec/lib/fake_psc_rackup_spec.rb`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod19b add lib/fake_psc/config.ru lib/digital_signature_crew.rb config/environments/development.rb db/seeds.rb spec/lib/digital_signature_crew_spec.rb spec/lib/fake_psc_rackup_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod19b commit -m "chore: add the simulated signature provider and digital signature dev seed

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 19: Revisão final, suíte completa, api na porta 3037, prova no navegador e rollout

- [ ] **Step 1: Varredura de vazamento.** `grep -rn "access_token\|code_verifier\|raw_signature\|canonical_json\|subject_cpf\|signer_cpf" apps/api/.claude/mod19b/app/services/analytics apps/api/.claude/mod19b/app/events apps/api/.claude/mod19b/config/initializers/domain_events.rb` — nada. `grep -rn 'DomainEvents.publish("signature' -A2 apps/api/.claude/mod19b/app` — só ids, `provider`, `document_type`, `reason_code`, `verification` (lembre: `publish` é multilinha — confira cada chamada inteira). `grep -rn "Rails.logger" apps/api/.claude/mod19b/app/services/signatures apps/api/.claude/mod19b/app/commands/signatures` — só classe de erro, nunca corpo, token ou CPF.
- [ ] **Step 2: Bancos de teste refeitos (Ambiente) e suíte completa com o worker parado, com o `signer` de pé (as specs `:signer` entram):**

  ```bash
  docker compose stop worker
  docker compose exec -T -w /rails/.claude/mod19b api bundle exec rspec
  docker compose start worker
  ```
  Expected: verde, sem o aviso `[signer] SIGNER_URL ausente`. Falha em spec antiga que fixa chaves (consulta finalizada, catálogo de interruptores, bindings de `consultation.*`, schema do maintenance) → acrescente `signature`, `digital_signature`, `Signatures::RequestJob`, `signatureProviders`/`signerStatus` (é o contrato); nunca afrouxe asserção de texto livre.
- [ ] **Step 3:** `docker compose exec -T -w /rails/.claude/mod19b api bundle exec rubocop <arquivos tocados>`; corrija só o que a regra do projeto aponta.
- [ ] **Step 4: api na porta 3037.** **Antes**, com autorização do usuário, uma etapa de cada vez: `bin/rails db:migrate:platform` (a tabela `signature_provider_checks` no banco de plataforma de dev), `city:migrate:all` (as cidades de dev ganham as tabelas da assinatura) e `db:seed` (Task 18 — liga `digital_signature` em Curitiba). Depois, sem derrubar o principal:

  ```bash
  docker compose exec -d -w /rails/.claude/mod19b -e CITY_PUBLIC_BASE_TEMPLATE=http://%{slug}.localhost:5187 api \
    bin/rails server -b 0.0.0.0 -p 3037 -P tmp/pids/server-mod19b.pid
  ```
  O Vite do worktree do dashboard aponta `VITE_API_PROXY_TARGET` para `http://api:3037`, com o prefixo `/signature` no proxy (plano do dashboard). O worker que roda o `SignJob` é o do compose: para a prova, ele precisa do código do worktree — rode um worker do worktree só para Curitiba (`docker compose exec -d -w /rails/.claude/mod19b worker bin/city_workers` com o `CITY_WORKERS_ONLY`/equivalente que o projeto usar; confira em `bin/city_workers`) ou, no mínimo, execute o job na mão no console do worktree (`Signatures::SignJob.perform_now(city_slug: "curitiba", request_id: …)`).
- [ ] **Step 5: Prova manual no navegador** (`http://curitiba.localhost:5187/dashboard/`; login, OTP e TOTP são do usuário — as senhas e o TOTP da semente de dev podem ser mostrados no chat se ele pedir):
  1. A médica (`profissional@curitiba.demo`) abre Conta → Assinatura digital → "Procurar meu certificado": VIDaaS encontrado. "Vincular" pede step-up, abre a página **PSC SIMULADO** (`localhost:8091`), "Aprovar" volta a `/dashboard/signature/callback` e a tela mostra o certificado (emissor da AC de dev, vence em ~1 ano).
  2. "Abrir sessão de assinatura" → Aprovar → o selo do topo mostra a sessão até +12 h.
  3. Ela chama da fila a cidadã da semente do 19a, faz a consulta e finaliza: o marcador da consulta passa de "pendente" a "assinada digitalmente — válida" (depois do job). "Ver o que foi assinado" mostra o JSON canônico; "Baixar PDF assinado" abre o PDF com o rodapé NGS2 em toda página; "Baixar .p7s" baixa o `.zip`; "Revalidar" responde válida. O impresso da consulta é o PDF assinado.
  4. Encerra a sessão, finaliza outra consulta: ela aparece em "Pendentes de assinatura" (`no_session`) e no contador do menu. "Assinar todas" → Aprovar → assinada. Numa terceira, "Voltar ao papel" com motivo: o impresso volta a ter "Assinatura e carimbo".
  5. A enfermeira (`enfermeira@curitiba.demo`), sem vincular certificado, finaliza uma consulta: marcador "sem assinatura digital", impresso do 19a.
  6. O admin (`admin@curitiba.demo`) abre o painel de assinatura: médica com certificado e 0 pendentes, enfermeira "sem certificado", documentos por modo.
  7. No maintenance (`maintenance.localhost:5177`, `dev@local`): o interruptor `digital_signature` aparece ligado; `signatureProviders` mostra `vidaas` configurado com a última checagem; `signerStatus` alcançável. Desligar o interruptor: a pendente que sobrou volta ao papel em até 10 min (`feature_disabled`); as assinadas continuam visíveis e baixáveis.
  8. Conferência independente do `.zip`: `openssl cms -verify -binary -inform DER -in document.json.p7s -content document.json -CAfile <(docker compose exec -T signer cat /signer-dev-pki/anchors.pem) -purpose any -noout` → `Verification successful` (o validar.iti.gov.br não conhece a AC de dev; a prova no ITI é a do sandbox VIDaaS, Task 0 do `signer`).
- [ ] **Step 6: Pare.** Merge, push, board e docs (página de status do módulo no padrão do módulo 01) só com autorização explícita do usuário, uma etapa de cada vez. Ordem: `contracts` (tag) → `signer` → **api** → dashboard → maintenance. Rollout por cidade: publicar a imagem nova e rodar `city:migrate:all` dela (e a migração de plataforma) **antes** de cortar tráfego — a migração de cidade é irreversível; o `signer` sobe antes do api (`SIGNER_URL`/`SIGNER_TOKEN` no api e no worker; **nunca** `SIGNER_DEV_PKI_DIR` nem `SIGNER_EXTRA_TRUST_ANCHORS` fora de dev — o `signer` recusa o boot com `SIGNER_ENV=production`); credenciais `signature.providers.<key>` só depois da homologação em cada PSC; o interruptor nasce desligado e é ligado por cidade pelo maintenance (a dispensa do papel é decisão da cidade, spec §14). Ao voltar o checkout para a main: `DROP DATABASE` dos dois bancos de teste e `city:test_databases`; derrube o servidor da 3037 (`kill $(cat tmp/pids/server-mod19b.pid)` no container) e volte o `working_dir` do `fake-psc` para `/rails` depois do merge.
- [ ] **Step 7: Go-live por cidade (antes de ligar `digital_signature` nela).** Para cada PSC configurado no ambiente, cadastrar na aplicação do Rota Saúde no PSC a `redirect_uri` da cidade, `https://<host do dashboard da cidade>/dashboard/signature/callback` — o valor exato sai de `bin/rails runner 'puts Signatures::Providers.redirect_uri(City.find_by!(slug: "<slug>"))'` no ambiente publicado (Desvio 9). Conferir com um vínculo de teste por PSC (o PSC recusa `redirect_uri` não cadastrada) e só então ligar o interruptor pelo maintenance. Cidade nova = repetir este passo em todo PSC; registre-o no checklist de provisionamento.
- [ ] **Step 8: Pendências para o board de pendências de ciclo** (cards só com autorização): CI do api com o `signer` como serviço (hoje as specs `:signer` só rodam no compose de dev); homologação do Rota Saúde em cada PSC e escolha do segundo PSC (NGS2.01.05); contrato de carimbo do tempo (AD-RT); volume do PDF assinado no banco da cidade.

---

## Self-review (feito ao escrever o plano)

**Cobertura da spec:**
- §3 interruptor e configuração → Tasks 1 (catálogo, `feature:`), 4 (credenciais da plataforma, catálogo de PSC), 13 (desligar devolve ao papel, Desvio 8), 16 (maintenance), 18 (dev).
- §4 dados → Task 3 (cinco tabelas, triggers, cifra), Task 4 (checagem na plataforma); bloco `signature` → Task 14.
- §5 vínculo → Tasks 6–7; sessão → Task 8; finalização → Tasks 11–12; lote e volta ao papel → Task 13; adendo e cadeia → Tasks 9, 11; validação → Tasks 11 (antes de gravar), 14 (ao abrir, imprimir, sob demanda); regras (CPF a cada uso, vencido, revogado, só o autor) → Tasks 11, 13, 17.
- §6 `signer` → Task 5 (cliente, falso, specs reais), Tasks 11/19 (fluxo real).
- §7 formatos e guarda → Tasks 9 (JCS, esquema, vetor), 10 (PDF com rodapé), 11 (CAdES + PAdES, provas), 14 (exportação PDF e `.zip`).
- §8 API → Tasks 7, 8, 13, 14, 15. §9 telas (lado api) → Tasks 13–16. §10 segurança e LGPD → Tasks 3 (cifra, filtro), 6 (state), 14 (trilha), 17 (vazamento). §11 testes → PSC falso (Task 4), `signer` real (Tasks 5, 11), contrato da API do ITI (Task 4), trigger (Task 3), corrida (Task 13), interruptor (Tasks 1, 13), adoção por profissional (Task 12), volta ao papel (Task 13), ausência de segredo (Task 17).
- `spec/adr_pointers_spec.rb` → Task 1.

**Placeholders:** nenhum "TBD"/"similar à Task N"; os dumps de schema estão por extenso. Deixado ao executor, com critério: conferir os nomes da API do ITI no DOC-ICP-17.01 (Task 0, com a tabela esperada e o critério de parada), a forma dos esquemas publicados (Task 9: o esquema prevalece), a variável de slug do `db/seeds.rb` do 19a (Task 18) e o modo de subir um worker do worktree (Task 19).

**Consistência de nomes:** `Signatures::{Gate,Providers,Psc::Client,Signer::Client,CertificateInfo,OauthStates,Canonical,Jcs,PdfFooter,Documents,Signing,CertificateRules,Mode,Verify,Package,PrintReport,AdminOverview,SignerStatus,Json,DocumentTypes}`, comandos `Signatures::{Discover,StartLink,AcceptCertificate,Unlink,CompleteOauth,StartSession,OpenSession,CloseSession,SignPending,Park,ToPaper,OpenRequest,ReturnToPaper,StartBatch,RunBatch}`, jobs `Signatures::{SignJob,RequestJob,SweepJob}` — conferidos entre as tasks. `Signing.call(requests:, access_token:, certificate:, now:, signer:)` é o mesmo no job (Task 11) e no lote (Task 13); `CompleteOauth` ganha `session` na Task 8 e `batch` na Task 13.

**Review Focus:** as cinco linhas têm teste na task dona (Tasks 13, 3, 11/17, 6/7, 9/10).

## Decisões do usuário (incorporadas)

1. **Cadeia no lote:** ordem cronológica de criação; o seguinte leva o hash do anterior do mesmo lote; anterior que falha segura os seguintes da consulta (Desvio 1; Tasks 9, 11, 13).
2. **Revogação no vínculo:** `POST /certificates/check` no `signer` (D1, obrigatória); `indeterminate` aceita e registra (Desvio 10; Tasks 3, 5, 7).
3. **Retorno do PSC:** um por cidade, no dashboard dela; cadastro por cidade em cada PSC no go-live (Desvio 9; Tasks 4, 6, 19).

## Divergências propostas ao contrato

- **D1 — `signer`: `POST /certificates/check { certificate_der_base64 }` → 200 `{ "status": "valid"|"invalid"|"indeterminate", "signer_cpf", "not_after", "reasons": [] }`** (cadeia + LCR, sem assinar; `invalid` com `certificate_revoked`/`certificate_expired`/`untrusted_chain`, `indeterminate` com `revocation_unavailable`; 422 `invalid_certificate` e os demais códigos do §9). Obrigatória (decisão do usuário). No vínculo: 422 `certificate_revoked`, `certificate_expired` e o código novo **`certificate_untrusted`** em `POST /signature/oauth/callback` (§4); `indeterminate` vincula e registra o motivo; `signer` fora do ar ou 422 → **503 `signer_unavailable`** (código novo no §4).
- **D2 — `redirect_uri` por cidade:** `https://<host do dashboard da cidade>/dashboard/signature/callback` (o que o §4 e o plano do dashboard já usam); o `state` é um token assinado opaco (cidade + id), que o dashboard só repassa.
- **D3 — `POST /signature/oauth/callback`** aceita `{ state, error }` (recusa no celular → 403 `authorization_denied`, e o `state` fica consumido) e devolve `return_to` (o guardado com o `state`) junto de `{ purpose, result }` — já combinado com o plano do dashboard.
- **D4 — Motivo `certificate_cpf_mismatch`** na lista de `reason_code` do §1 (NGS2.01.02: o CPF do profissional mudou depois do vínculo).
- **D5 — 409 `professional_cpf_missing`** em `discover`, `link`, `sessions` e `batches` quando o perfil do profissional não tem CPF.
- **D6 — 422 `invalid_provider`** também em `POST /signature/sessions` e `POST /signature/batches` quando o PSC do certificado vinculado deixou de estar configurado no ambiente.
- **D7 — Leitura de assinatura gravada** (`/signature/signatures/:id`, `/pdf`, `/package`, `/verify`) responde ao interruptor `clinical_record` (403 `feature_disabled` com `feature: "clinical_record"`), não ao `digital_signature` — o assinado continua visível com a assinatura desligada; fora de contexto e sem abertura → 403 `opening_required` (19a); id desconhecido → 404 `not_found`.
- **D8 — Lote:** falha do PSC ou do `signer` **durante a assinatura** volta como itens de `failed` (com `provider_unavailable`/`provider_rejected`/`signer_unavailable`), e o callback responde 200; só a troca do código com o PSC fora do ar dá 503 `provider_unavailable`. Pedido travado pelo job no momento do lote fica fora do resultado (nem `signed` nem `failed`).
- **D9 — `GET /signature/requests`** lista sempre os `pending` (o parâmetro `status` só aceita `pending`; outro valor é ignorado). `POST /signature/requests/:id/return_to_paper` com id desconhecido → 404 `not_found`.
- **D10 — Status `failed`** não é produzido pelo 19b (Desvio 2): toda falha fica `pending` com motivo.
- **D11 — Admin:** `GET /signature/admin/overview` → 403 `missing_role` fora do `municipal_admin`; 422 `invalid_period`; sem `from`/`to`, os últimos 30 dias até hoje; `documents_by_mode` conta consultas finalizadas e adendos criados no período (`pending` inclui `failed`).
- **D12 — Bloco `signature`:** `signer_name` = nome do profissional autor (o do cadastro); o nome do certificado vai só no rodapé do PDF e no `content`.
