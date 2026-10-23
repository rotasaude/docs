# Módulo 24a — Cadastro individual e domiciliar (api) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **PRÉ-REQUISITO: módulo 23 em `origin/main` do api** (conferido ao escrever este plano em `9d35b6e`: `Platform::Features::CATALOG` terminando em `care_lines`; última migração de cidade `20261022900003_add_care_line_origin_to_appointment_requests.rb`; `db/city_schema.rb` em `define(version: 2026_10_22_900003)`; `spec/adr_pointers_spec.rb` com `VALID_RANGE = (1..40)`; `Ledi::FichaTypes::REGISTRY` com `procedimento` (7) e `atendimento_individual` (4); `Ledi::Version::TYPES` com três arquivos; `Ledi::FichaSources::REGISTRY` com `Screening` e `Consultation`; `LediGenerationFailure::REASONS` terminando em `still_sending`; `patients.cpf` `null: false` com `index_patients_on_cpf` único; `professionals.council/council_state/registration_number` `null: false`; `Citizens::RequestErasure.examined_patients?` terminando em `CareLineEnrollment`). **A Task 0 confere a base: se um nome mudou, PARE e reporte.**

**Goal:** O lado api do módulo 24a (F-24.1 a F-24.8, ADR 0042): interruptor `territory_registry`; microáreas por equipe com o ACS responsável; domicílio, famílias e cadastro individual com versões imutáveis que viram as fichas LEDI de Cadastro Domiciliar e Territorial (tipo 3) e de Cadastro Individual (tipo 2); paciente sem CPF (justificativa da ficha, CNS, índice cego de duplicados, `cpf_taken` ao informar o CPF); validade de 24 meses e quem falta cadastrar; equipe da pessoa e óbito pelo cadastro (regra do 23); ACS e agente de endemias sem conselho (módulo 10); exclusão LGPD; semente de dev em Curitiba; runbook.

**Architecture:** Uma migração de cidade (irreversível) cria `micro_areas`, `households` + `household_versions`, `families`, `individual_registrations` + `individual_registration_versions`, torna `patients.cpf` opcional (com `no_cpf_reason`, `cns` e `match_key`) e afrouxa o conselho em `professionals` só para o perfil sem conselho. Cada gravação de domicílio/família ou de cadastro individual cria, numa transação, a versão imutável (trigger em `db/city_triggers.sql`) e enfileira a ficha gerada **da versão** (fonte única do que foi enviado). Os dicionários das fichas vivem num YAML (`config/ledi/registration_dictionaries.yml`) produzido pela prova técnica (Task 0) e validado contra o IDL vendorizado; um único validador (`TerritoryRegistry::Dictionary` + `TerritoryRegistry::Rules`) serve ao api e é a referência do espelho no dashboard.

**Tech Stack:** Rails 8.1 (API), PostgreSQL 16 (banco por cidade), RSpec via `bin/test`, Thrift 0.17 (classes LEDI 8.7.0 já geradas em `vendor/ledi/8.7.0/gen-rb`). Nenhuma gem nova.

**Spec:** `docs/superpowers/specs/2026-10-23-module-24a-territory-registry-design.md` e `docs/adr/0042.md` (divisão: `docs/adr/0041.md`; linhas de cuidado: `docs/adr/0040.md`). Contrato entre apps — **fonte única de formatos**: `docs/superpowers/plans/2026-10-23-module-24a-territory-registry-contracts.md` (convenções dos contratos do 21a §15, 21b §14, 21c §11, 22 §12–§13 e 23 §11–§12); toda diferença está em "Divergências propostas ao contrato", no fim (prefixo **R**). Pesquisa: `docs/pesquisa/2026-10-05-integracoes/frente-1-aps-sisab.md` (§2 fichas, §5 componente de Vínculo, §7 identificação). Lições que valem aqui: contadores sem trilha; ordem de travas estável; recusar em vez de cortar (texto longo → 422 com `path`); todas as recusas de uma vez não se aplicam (aqui não há HL7) — o validador devolve a **primeira** recusa com `path`, como o 22; todo `Data`/`Struct` com dado pessoal sem dado em `inspect`/`to_s`/`pretty_print`/`as_json`.

## Leitura preliminar das regras oficiais (feita ao escrever este plano)

Lidas em 2026-10-23 na documentação LEDI 8.7.0 (`https://integracao.esusaps.bridge.ufsc.tech/ledi/documentacao/`): `estrutura_arquivos/dicionario-fci.html`, `estrutura_arquivos/dicionario-fcd.html`, `referencias/dicionario.html`, `regras/cbo.html`. **A Task 0 baixa as páginas, grava os SHA-256 e confirma cada item abaixo; o que divergir, ela corrige aqui e no YAML antes de qualquer outra task.**

1. **`tipoDadoSerializado`:** `2` = Cadastro Individual; `3` = Cadastro Domiciliar e Territorial (4 = Atendimento Individual e 7 = Procedimentos, já no `FichaTypes`, conferem).
2. **Comuns às duas fichas:** `tpCdsOrigem = 3`; `uuid` com prefixo do CNES e hífen (o `Ledi::Enqueue` já gera `"#{cnes}-#{uuid}"`); `fichaAtualizada` obrigatório; **`uuidFichaOriginadora` obrigatório: igual ao `uuid` na ficha de cadastro novo, o `uuid` da originadora na atualização**; `headerTransport` = `UnicaLotacaoHeaderThrift` com CBO restrito à Tabela 1 (individual) / Tabela 2 (domiciliar) de `regras/cbo.html` — códigos de 6 posições (alguns com letra, ex. `2231F9`). `515105`, `515140`, `223505`, `225125`, `225130`, `225142`, `225170`, `322205`, `322245` estão nas duas tabelas.
3. **Identificação (individual):** `cpfCidadao` e `cnsCidadao` **nunca juntos**; CPF obrigatório se não há CNS e `stNaoPossuiCpf = false`; **CNS obrigatório se `stNaoPossuiCpf = true`**; `justificativaNaoPossuiCpf` obrigatória com `stNaoPossuiCpf = true`, códigos `2 3 4 5 6 7 8 9 10 11 12 13` (3 recém-nascido, 5 situação de rua sem documentos, 8 pessoa indocumentada, 9 criança sem documentação…). `nomeCidadao` 3–70 (regra "validar nome"); `nomeMaeCidadao` 3–70 salvo `desconheceNomeMae`; `nomePaiCidadao` obrigatório salvo `desconheceNomePai`; `racaCorCidadao` obrigatório (**1 Branca, 2 Preta, 3 Amarela, 4 Parda, 5 Indígena** — ≠ BPA, onde 03 é Parda e 04 Amarela); `etnia` só e sempre com raça 5; `nacionalidadeCidadao` obrigatório (1, 2, 3) com `codigoIbgeMunicipioNascimento` (1), `dtNaturalizacao`+`portariaNaturalizacao` (2), `paisNascimento`+`dtEntradaBrasil` (3); `paisNascimento = 31` com nacionalidade 1; `telefoneCelular` obrigatório para nacionalidade 1 ou 2; `sexoCidadao` obrigatório (**0 Masculino, 1 Feminino**, 4 Ignorado, 5 Indeterminado); `dataNascimentoCidadao` em epoch ms, nunca depois do atendimento nem 130 anos antes; `microArea` (2) proibida com `stForaArea`; `cpfResponsavelFamiliar`/`cnsResponsavelFamiliar` só com `statusEhResponsavel = false`, nunca juntos.
4. **Recusa (individual):** com `statusTermoRecusaCadastroIndividualAtencaoBasica = true`, ficam proibidos `condicoesDeSaude`, `emSituacaoDeRua`, `informacoesSocioDemograficas`, `informacoesSocioEconomicas` e `saidaCidadaoCadastro`; a identificação fica opcional.
5. **Saída:** `motivoSaidaCidadao` **135 óbito, 136 mudança de território**; `dataObito` só e sempre com 135; `numeroDO` (9) só com 135.
6. **Condições de saúde / situação de rua / sociodemográficos:** regras condicionais listadas na Task 0 (ex.: `statusEhGestante` proibido para sexo 0 e fora de 9–60 anos; listas de doença cardíaca/respiratória/rins obrigatórias com o status true e proibidas com false; `statusSituacaoRua` obrigatório sem recusa; demais campos de rua proibidos com `statusSituacaoRua = false`; `responsavelPorCrianca` proibido a partir de 10 anos; `relacaoParentescoCidadao` só para não responsável).
7. **Domiciliar:** `tipoDeImovel` obrigatório (1 Domicílio, 2 Comércio, 3 Terreno baldio, 4 Ponto estratégico, 5 Escola, 6 Creche, 7 Abrigo, 8 ILPI, 9 Unidade prisional, 10 Medida socioeducativa, 11 Delegacia, 12 Estabelecimento religioso, 13 CASAI, 99 Outros); `familias`, animais e `telefoneResidencia` só com tipo 1; `condicaoMoradia` proibida para 2, 3, 4, 5, 6, 12, 13, 99 e (sem recusa) com `localizacao` e `situacaoMoradiaPosseTerra` obrigatórios; `instituicaoPermanencia` só para 7–11 (`nomeResponsavelTecnico` obrigatório); `enderecoLocalPermanencia` obrigatório sem recusa: `bairro` (≤ 72), `cep` (8 dígitos), `codigoIbgeMunicipio`, `nomeLogradouro` (≤ 72), `numero` (1–10, letras e dígitos; proibido com `stSemNumero`), `numeroDneUf`, `tipoLogradouroNumeroDne` (código DNE), `microArea` (salvo `stForaArea`), `complemento` ≤ 30, `pontoReferencia` ≤ 40, telefones 10–11 dígitos; `tipoEndereco = 3`; `FamiliaRow`: `numeroCnsResponsavel` ou `cpfResponsavel` (um só), `numeroMembrosFamilia` ≠ 0, `rendaFamiliar` (**1 ¼ SM, 2 ½ SM, 3 1 SM, 4 2 SM, 7 3 SM, 5 4 SM, 6 acima de 4 SM, 8 sem renda**), `resideDesde` (mês/ano, epoch ms, não futuro), `stMudanca`; famílias com o mesmo responsável recusadas; `nuMoradores` ≥ número de famílias e ≥ Σ membros.
8. **Achado:** a documentação lista `stAtivo` em `CadastroDomiciliar`, mas o IDL vendorizado (8.7.0, commit `9316f280f5`) **não tem esse campo** (vai até o id 20 sem ele). O binário segue o IDL: domicílio inativo vai como ficha de atualização com todas as famílias `stMudanca = true` (Desvio 9). A Task 0 registra o achado na tabela do §14.
9. **Conferido na Task 0 (2026-10-23, api `c1665bf`):** os itens 1–8 batem com as páginas baixadas. Achados novos no §15.5 do contrato: nomes que divergem entre documentação e IDL (vale o IDL), etnia com tradução (`0304` → 406, `0315` → 407; `TerritoryRegistry::Catalog.ethnicity_to_ledi`), UF e tipo de logradouro como domínios de texto (`string_domains` do YAML) e o campo 350 (situação de rua) do BPA-I.

## Desvios e precisões

1. **Índice cego = cifra determinística, não HMAC próprio.** `patients.match_key` guarda `"<nome normalizado>|<AAAA-MM-DD>"` cifrado com `encrypts deterministic: true, key_provider: CityDeterministicKeyProvider.new` — a mesma técnica de `patients.cpf`. Dá a busca por igualdade sem texto claro (o que a spec pede) e entra em `CITY_KEYED_TARGETS`, então `city:rotate_key`/`ReencryptionJob` a regravam sozinhos; um HMAC com chave derivada exigiria um passo novo no `CityRekey` (que hoje não regrava nada fora da lista). Normalização: `I18n.transliterate`, minúsculas, só `[a-z ]`, espaços colapsados. O nome da mãe não entra na chave (pode faltar): filtra os candidatos depois de decifrados.
2. **Retenção LGPD por versão gravada, não só por ficha enfileirada.** A spec define "enviado" = ficha no `ledi_outbox`. Mas a versão é imutável (o trigger recusa DELETE) e a ficha só nasce com `ledi_export` utilizável. Retém quem tem **qualquer** `individual_registration_versions` ou é responsável de família (`families.responsible_patient_id`) — superconjunto do "enfileirado", sem caso em que o dado fique e o pedido diga "excluído". Paciente sem CPF nasce sempre com versão, logo sempre retido; não há pedido de exclusão por paciente (o fluxo do posto é por CPF) — o runbook registra.
3. **Equipe do cadastro sem domicílio.** O cadastro individual guarda `team_id` (e `micro_area_id` quando há domicílio). Com domicílio: a equipe da microárea do domicílio. Sem domicílio (situação de rua, morador ainda sem domicílio cadastrado): a equipe informada em `team_id` no corpo, onde quem grava é membro — a ficha vai com `stForaArea = true` e sem `microArea` (R2).
4. **Quem grava → cabeçalho.** CNS do profissional de quem grava, o CBO do vínculo dele **na equipe** (`health_team_members.cbo_code`), INE da equipe, CNES da unidade da equipe, `dataAtendimento` = instante da versão, `codigoIbgeMunicipio` do `CityProfile`. CBO fora da Tabela 1/2 da prova técnica → 403 `cbo_not_allowed` na gravação (a ficha nunca nasceria aceitável).
5. **Validação no posto depois do cadastro (decisão 13).** `Patients::Resolve.refresh!` que muda nome, nascimento ou sexo de paciente com cadastro ativo chama `TerritoryRegistry::IdentityRefresh`: nova versão (`origin = identity_refresh`) com a identidade nova e o resto da versão anterior, `identity_changed_at` gravado e a ficha de atualização. Quem está no cabeçalho é **quem gravou a versão anterior** (o validador da recepção não tem CBO de cadastro); a data é a de agora. Sem `territory_registry` utilizável, só grava `identity_changed_at` (sem versão).
6. **Originadora.** A originadora de um cadastro é a **mais antiga** ficha dele (qualquer versão) que não está `rejected` nem `correction_pending`. Sem nenhuma, a ficha nova sai como cadastro novo (`fichaAtualizada = false`, `uuidFichaOriginadora = uuid`). Reenvio de recusada passa pela regeneração da fonte (`FichaSources`), nunca por `Transport.rewrap` — e o `rewrap` passa a trocar também `uuidFichaOriginadora` quando ele era igual ao `uuid` antigo (Review Focus 3).
7. **Paciente sem CPF e sem CNS.** O cadastro grava (decisão 1: "CNS quando houver"), mas a ficha individual exige CNS sem CPF: vira "não gerada" com `patient_without_identifier`; informar o CNS (nova versão) gera a ficha. Família cujo responsável não tem CPF nem CNS: ficha domiciliar "não gerada" com `responsible_without_identifier`. Nunca CPF inventado.
8. **Endereço do domicílio no paciente.** Gravar o domicílio regrava, nos pacientes com cadastro ativo naquele domicílio, `address_zip`, `address_street`, `address_number` (`SN` sem número), `address_complement` e `address_district` (o nome do bairro do 11, ou o texto `district`). Os limites do domicílio são o **menor** entre LEDI e BPA (logradouro 30, número 5, complemento 10, bairro 30): mais longo → 422 com `path`, nunca cortado. `Patients::Demographics.save!` passa a recusar chave de endereço de paciente com cadastro ativo em domicílio (`address_from_household`, R6); raça/cor e etnia continuam livres.
9. **Domicílio inativo.** `status = inactive` só sem famílias ativas (todas `moved_out`); a ficha sai como atualização com as famílias em `stMudanca = true` (o IDL não tem `stAtivo`).
10. **Ficha sem `ledi_export`.** `Ledi::Enqueue` devolve `nil` com a exportação desligada: a versão grava e nenhuma ficha nasce. Não há varredura retroativa: o runbook manda ligar `ledi_export` **antes** de `territory_registry`.
11. **Trilha.** Leitura nominal de pessoa (cadastro, lista de cadastros, duplicados, validade): um `clinical_record.viewed` por pessoa mostrada (`access: "territory_registry"`, `person_ref`), como o 23 (L7). Domicílio: um evento por consulta — `territory_household.viewed` (`household_id`) e `territory_households.listed` (`micro_area_id`, `household_ids`) — com `user_id` (R7). Contador sem trilha.
12. **Ordem de travas.** CPF (lock consultivo de `Patients::Resolve.lock_key`) → pares do CPF → paciente(s) por id → domicílio → família → cadastro. Gravar domicílio trava os pacientes moradores **antes** do domicílio: lê os moradores, trava os pacientes por id, trava o domicílio, relê; se o conjunto cresceu, recomeça (até 3 vezes, como o `Citizens::Erase`).

## Valores fixados por este plano

1. **Interruptor** (depois de `care_lines` no `CATALOG`): `territory_registry`, `requires: %w[feature:clinical_record]`, descrição "Cadastro territorial: microáreas, domicílios, famílias e cadastro individual com fichas LEDI de cadastro".
2. **Códigos (`TerritoryRegistry::Catalog`):** `ACS_CBO = "515105"`, `ENDEMIC_AGENT_CBO = "515140"`, `REGISTRAR_CBO_PREFIXES = %w[2235 2251 2252 2253]`, `VALIDITY_MONTHS = 24`, `DUE_SOON_DAYS = 60`, `EXIT_REASONS = %w[moved death]`, `LEDI_EXIT = { "death" => 135, "moved" => 136 }`, `NO_CPF_REASONS = %w[2 3 4 5 6 7 8 9 10 11 12 13]`, `PROPERTY_TYPES = %w[1 2 3 4 5 6 7 8 9 10 11 12 13 99]`, `INCOME_BANDS = %w[1 2 3 4 5 6 7 8]`, `RACE_TO_LEDI = { "01" => 1, "02" => 2, "03" => 4, "04" => 3, "05" => 5 }`, `SEX_TO_LEDI = { "male" => 0, "female" => 1 }`, `STATUSES = %w[valid due_soon expired missing refused exited]`, `VERSION_ORIGINS = %w[save identity_refresh exit]`.
3. **LEDI:** `Ledi::FichaTypes` `"cadastro_individual" => { code: 2, klass: "Br::Gov::Saude::Esusab::Ras::Cadastroindividual::CadastroIndividualThrift", uuid_field: :uuid }`, `"cadastro_domiciliar" => { code: 3, klass: "Br::Gov::Saude::Esusab::Ras::Cadastrodomiciliar::CadastroDomiciliarThrift", uuid_field: :uuid }`; `Ledi::Version::TYPES` + `cadastro_individual_types cadastro_domiciliar_types`; fontes `IndividualRegistrationVersion` e `HouseholdVersion`; motivos novos de "não gerada" `patient_without_identifier responsible_without_identifier cbo_not_in_ledi_table`.
4. **Limites:** microárea `code` `/\A(0[1-9]|[1-9]\d)\z/`, `name` 1..60; endereço `zip` 8 dígitos, `street` 1..30, `number` 1..5 `[A-Za-z0-9]`, `complement` ≤ 10, `district` ≤ 30, `reference` ≤ 40, `street_type` código do domínio `TipoLogradouro`; telefone 10–11 dígitos; `full_name`/`mother_name`/nome do pai 3..70 (regra "validar nome" da Task 0); `social_name` ≤ 70; textos dos dicionários ≤ 100; `members_count` 1..999; `limit` das listas: padrão 50, máximo 100; duplicados: no máximo 10.
5. **Dev:** api do worktree na porta **3046**; worktree `apps/api/.claude/mod24a` (com `config/master.key`); branch `feat/mod-24a-territory-registry`; semente em Curitiba na "UBS Jardim das Flores", equipe "ESF JARDIM DAS FLORES" (`CareLinesCrew::TEAM_NAME`), ACS "Rosana Aparecida Moreira" (CBO 515105, sem conselho), microáreas `01` "Jardim das Flores — norte" e `02` "Jardim das Flores — sul", telefones com prefixo `96666`.

## Global Constraints

- git `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta); nunca `git add -A`; commits em inglês, Conventional Commits por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`; merge, push e tag só com autorização do usuário.
- **Primeira ação da execução:** mover os cards F-24.1..F-24.8 do board #1 para In Progress (`gh project item-edit …`, um comando `gh` por chamada). Um card só vai para Done quando o ÚLTIMO app dele terminar; Verified depois da prova no navegador.
- Testes só por `bin/test api -w mod24a <specs>` (a partir de `/Users/eduardovrocha/Development/ioit.solutions/rota-saude`); nunca `rspec` direto nem o Postgres do host; a suíte completa é pedida à sessão "Testes" (Task 15).
- Banco de cidade: `db/city_migrate/20261023900001_add_territory_registry.rb`, dump à mão em `db/city_schema.rb` (`define(version: 2026_10_23_900001)`; paridade em `spec/services/city_schema_spec.rb`); triggers em `db/city_triggers.sql`, executado no fim do `up` (o dump não representa trigger). Irreversível. Rollout: publicar a imagem e rodar `bin/migrate` dela antes de cortar tráfego; nunca migrar cidade fora do rake; **o banco de dev só é migrado/semeado na prova no navegador, com autorização do usuário.**
- Specs de requisição com `type: :request` (a inferência está desligada); nunca simule cidade inalcançável com URL real.
- Erros `{ "error": "<reason>" }` (com `path` ou `field` quando o contrato manda); interruptor 403 `{ "error": "feature_disabled", "feature": "territory_registry" }`; step-up 401 `mfa_required`; Content-Type JSON em toda escrita por cookie; registro de outro escopo por id → 404; lista/criação com escopo alheio → 403 `out_of_scope`.
- **CPF, CNS, nome, nome da mãe/pai, nascimento, condições de saúde, recusa, situação de rua, moradia, renda, justificativa sem CPF e telefone nunca em log, evento, Analytics, mensagem de erro, URL ou `inspect`.** Só no corpo JSON (leitura com `request.request_parameters`, como o `CareLines::BaseController#body_param`). Filtros de log novos: `full_name`, `social_name`, `mother_name`, `no_cpf_reason`, `sociodemographic`, `health_conditions`, `street_situation`, `living_conditions`, `identification`, `refused`, `address`, `income_band`, `living_since`.
- Eventos declarados em `config/initializers/domain_events.rb` e em `spec/initializers/domain_events_bindings_spec.rb`; `DomainEvents.publish(` é multilinha — confira cada chamada inteira (memória "Grep de linha única engana"). Este módulo não cria `Platform.audit` (o interruptor usa o `city.feature_changed` existente); o guarda R18 (`spec/events/platform_event_payload_guard_spec.rb`) roda na Task 15.
- Nunca `Current.city =` solto: leia `Current.city`; em seed/rake, `CityConnection.with`.
- Cifra por cidade em `CityEncryption::CITY_KEYED_TARGETS` para todo `encrypts` novo (o guarda `spec/architecture/city_encrypted_attributes_guard_spec.rb` exige).
- `spec/adr_pointers_spec.rb` sobe para `VALID_RANGE = (1..42)`.

## Review Focus

1. **Raça/cor do paciente (códigos do BPA, `03` Parda e `04` Amarela) vai para a ficha individual pelos códigos LEDI (`4` Parda, `3` Amarela)** — uma cópia direta troca parda por amarela em toda ficha. Teste na Task 8 ("raça/cor 03 do paciente sai 4 na ficha").
2. **Endereço do domicílio maior que o layout do BPA** (logradouro de 31 caracteres, número `1234A6`, complemento de 11) → 422 `invalid_household` com `path`, nada gravado; nunca cortar nem deixar o BPA-I do morador quebrar depois. Teste na Task 9.
3. **Reenvio de ficha de cadastro NOVO recusada:** o `uuid` muda e `uuidFichaOriginadora` muda junto (regeneração e `Transport.rewrap`); a atualização seguinte aponta para a nova. Teste na Task 8.
4. **Paciente sem CPF e sem CNS:** o cadastro grava (201), a ficha vira "não gerada" `patient_without_identifier`, e o PATCH com `cns` gera a ficha da nova versão — nunca 500 nem CPF inventado. Teste na Task 10.
5. **Dois profissionais cadastram o mesmo paciente ao mesmo tempo:** um 201, outro 409 `already_registered`; nunca dois cadastros ativos nem 500 (índice único parcial + `rescue RecordNotUnique`). Teste na Task 10.

## Inventário dos leitores de `patients.cpf` (paciente sem CPF)

Alcance hoje de um paciente sem CPF: ele nasce só pelo cadastro (Task 10) e **nunca tem par** (o par exige CPF); `attendances.citizen_id` é `null: false`, então não há atendimento, consulta (19a/19b), documento (19c/19d), pedido de exame ou requisição do laboratório (21) dele. Alcança: telas do 24a, linhas de cuidado (23, pela equipe do cadastro — Task 11) e, em tese, o faturamento (22) se um procedimento for lançado com o `patient_id`. Até ganhar CPF (Task 4), tudo que busca por CPF simplesmente não o acha.

| # | Local (origin/main `9d35b6e`) | Módulo | Leitura hoje | Mudança | Task |
|---|---|---|---|---|---|
| 1 | `app/models/patient.rb:24` `validates :cpf, presence: true` | 19a | exige CPF | `num_nonnulls(cpf, no_cpf_reason) = 1` (validação e CHECK) | 2, 4 |
| 2 | `app/models/patient.rb:27` `cpf_masked` | 19a | `mask(nil)` levanta | `cpf && CitizenIdentity::Cpf.mask(cpf)` | 4 |
| 3 | `app/commands/patients/resolve.rb` (`find_by(cpf:)`, `lock_key`, `refresh!`) | 19a | acha/cria pelo CPF do par | sem mudança na busca; `refresh!` chama `TerritoryRegistry::IdentityRefresh` (Desvio 5) | 10 |
| 4 | `app/services/clinical_record/access.rb:55,62` `where(cpf: patient.cpf)`, `citizen.cpf != patient.cpf` | 19a | pares pelo CPF | `Patients::Pairs.of(patient)`; `in_context?` compara `patient_id` do par quando o paciente não tem CPF | 5 |
| 5 | `app/controllers/clinical_records_controller.rb:39` | 19a | par pelo CPF | `Patients::Pairs.of(patient).order(:created_at).first` | 5 |
| 6 | `app/services/clinical_record/json.rb:21` `cpf_masked` | 19a | — | nulo-seguro pelo #2 (tela mostra "sem CPF", contrato §10) | 4 |
| 7 | `app/controllers/clinical_record_openings_controller.rb:68,86` | 19a | `cpf_masked` | nulo-seguro pelo #2 | 4 |
| 8 | `app/commands/clinical_record/open.rb:25`, `app/controllers/attendance_controller.rb:57` | 19a/18 | `Patient.find_by(cpf: digits)` | sem mudança (busca pelo CPF digitado) | — |
| 9 | `app/services/ledi/consultation_ficha.rb:92` `citizen_cpf: patient.cpf` | 16/19a | CPF na ficha | `citizen_cpf` ou `citizen_cns` (um só); sem os dois → "não gerada" `patient_without_identifier` | 5 |
| 10 | `app/services/ledi/fichas/individual_care.rb:45` `stCidadaoNaoPossuiCpf: false` | 16 | fixo | `cpfCidadao`/`cnsCidadao` exclusivos; `stCidadaoNaoPossuiCpf = cpf.nil?` | 5 |
| 11 | `app/services/ledi/fichas/screening_identity.rb` + `screening_ficha.rb:62` | 18 | CPF do par | `ScreeningIdentity` ganha `citizen_cns` (a escuta passa `nil`: o par sempre tem CPF) | 5 |
| 12 | `app/services/consultations/print.rb:94` | 19a | `cpf.to_s.sub` | `"sem CPF"` quando nulo (inalcançável hoje; barato) | 5 |
| 13 | `app/controllers/clinical_documents_controller.rb:122`, `app/services/clinical_documents/content/patient_identification.rb:21`, `app/services/clinical_documents/pdf.rb:92,96`, `app/services/signatures/canonical.rb:154` | 19b/19c/19d | CPF do paciente no documento/assinatura | **sem mudança**: exigem consulta (inalcançável); `patient_identification` já recusa (`missing`) sem CPF — spec de guarda na Task 5 | 5 |
| 14 | `app/services/exams/fhir/identifiers.rb:20,23,29` | 21 | CPF e CNS dos pares | `Patients::Pairs.of`; `for_patient` usa o CNS do paciente sem CPF (`CNS_SYSTEM`) e `nil` sem os dois | 5 |
| 15 | `app/services/exams/json.rb:117` `cpf_last3` | 21 | `cpf.to_s[-3,3]` | já nulo-seguro; spec | 5 |
| 16 | `app/commands/care_lines/enroll.rb:65` | 23 | pares pelo CPF | `Patients::Pairs.of(patient)` | 5 |
| 17 | `app/services/care_lines/facts.rb:73`, `app/commands/care_lines/accept_suggestion.rb:51`, `app/controllers/care_lines/person_lookups_controller.rb:25`, `app/services/care_lines/json.rb:66` | 23 | CPF **do par** | sem mudança (o paciente sem CPF entra pelo `patient_id`); `Json.person` não lê CPF | — |
| 18 | `app/services/timeline/person.rb:10` | 20 | pares pelo CPF | `Patients::Pairs.of(patient)` | 5 |
| 19 | `app/services/billing/critic.rb:77` `missing_patient_id` | 22 | sem CPF → crítica | crítica só sem CPF **e** sem CNS | 5 |
| 20 | `app/services/billing/lines.rb:31`, `app/services/billing/bpa_file.rb:72,78` | 22 | CPF em 339–349, CNS em brancos, `prd_sem_cpf = N` | `ILine` ganha `cns`; sem CPF: CNS em 60–74, CPF em brancos, `prd_sem_cpf` conforme a Task 0 (`S`, "Pessoa sem CPF/Registro Civil") | 5 |
| 21 | `app/controllers/billing/patients_controller.rb`, `app/controllers/attendance_controller.rb:152,169` (`Patients::Demographics`) | 22/18 | escrevem endereço | recusa `address_from_household` com cadastro ativo em domicílio (Desvio 8); `flags` ganha `address_from_household` | 9 |
| 22 | `app/commands/citizens/request_erasure.rb:71`, `app/commands/citizens/erase.rb` | ADR 0026 | paciente pelo CPF do pedido | sem mudança na busca; `examined_patients?` retém cadastro (Desvio 2) | 13 |
| 23 | Módulo 17 (`app/services/scheduling/request_json.rb:42`, `appointment_presenter.rb:28`, `unit_agenda.rb:38`) | 17 | `citizen.cpf_masked` (par) | sem mudança — o 17 não lê `patients.cpf` | — |
| 24 | `index_patients_on_cpf` (único) | 19a | — | continua único; NULL não colide | 2 |

## Mapa de arquivos

**Criar**
- `vendor/ledi/8.7.0/docs/` — páginas da documentação LEDI baixadas na Task 0 (`dicionario-fci.html`, `dicionario-fcd.html`, `dicionario.html`, `cbo.html`, `header-transport.html`, `validar_nome.html`, `algoritmo_CNS.html`) + `vendor/ledi/8.7.0/SOURCE` (acréscimo `docs_sha256`).
- `config/ledi/registration_dictionaries.yml` — seções, campos (chave IDL em snake_case), tipos, domínios e limites (Task 0).
- `config/ledi/registration_cbos.yml` — Tabela 1 e Tabela 2 de `regras/cbo.html` (Task 0).
- `db/city_migrate/20261023900001_add_territory_registry.rb`.
- `app/models/micro_area.rb`, `household.rb`, `household_version.rb`, `family.rb`, `individual_registration.rb`, `individual_registration_version.rb`.
- `app/services/territory_registry/{gate,catalog,access,dictionary,rules,identity,address,validity,teams,registry_status,json,trail}.rb`.
- `app/controllers/concerns/territory_registry_gate.rb`; `app/controllers/territory_registry/{base,micro_areas,households,families,registrations,duplicates,registry_status}_controller.rb`.
- `app/commands/territory_registry/{save_micro_area,save_household,save_family,save_registration,exit_registration,identity_refresh}.rb`.
- `app/commands/patients/{add_cpf,create_typed}.rb`; `app/services/patients/{match_key,duplicates,pairs}.rb`.
- `app/services/ledi/fichas/{registration_header,individual_registration,household_registration}.rb`; `app/services/ledi/{registration_ficha,individual_registration_ficha,household_registration_ficha}.rb`.
- `lib/territory_registry_crew.rb`; `lib/tasks/territory_registry.rake` (`patients:backfill_match_keys`).
- Specs: `spec/services/ledi/registration_dictionaries_spec.rb`, `spec/models/territory_registry_triggers_spec.rb`, `spec/models/patient_without_cpf_spec.rb`, `spec/commands/professionals/field_agent_spec.rb`, `spec/commands/patients/add_cpf_spec.rb`, `spec/services/patients/duplicates_spec.rb`, `spec/services/patient_cpf_readers_spec.rb`, `spec/requests/territory_registry/{micro_areas,households,registrations,duplicates,registry_status}_spec.rb`, `spec/services/territory_registry/{dictionary,rules,validity,teams}_spec.rb`, `spec/services/ledi/fichas/{individual_registration,household_registration}_spec.rb`, `spec/services/ledi/registration_ficha_spec.rb`, `spec/commands/territory_registry/{save_household,save_registration,identity_refresh}_spec.rb`, `spec/commands/territory_registry/locks_spec.rb`, `spec/commands/citizens/territory_registry_erasure_spec.rb`, `spec/lib/territory_registry_crew_spec.rb`, `spec/support/territory_registry_helpers.rb`.

**Modificar**
- `app/services/platform/features.rb` (CATALOG), `spec/adr_pointers_spec.rb` (1..42).
- `db/city_schema.rb`, `db/city_triggers.sql`, `app/services/city_encryption.rb` (CITY_KEYED_TARGETS).
- `app/models/patient.rb`, `app/models/professional.rb`, `config/professionals/cbo_saude.yml`, `app/services/clinical_documents/json.rb:62`, `app/services/consultations/print.rb:94,103`.
- Leitores do inventário (#2, #4, #5, #9–#12, #14, #16, #18–#21).
- `app/commands/patients/resolve.rb` (gancho do `refresh!`).
- `app/services/ledi/{ficha_types,ficha_sources,version,transport}.rb`, `app/models/ledi_generation_failure.rb`, `spec/services/ledi/contract_spec.rb`.
- `app/services/care_lines/{team_population,facts}.rb` e `spec/services/care_lines/{team_population,facts,suggest}_spec.rb`.
- `app/commands/citizens/request_erasure.rb`.
- `config/routes.rb`, `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`, `config/initializers/filter_parameter_logging.rb`.
- `db/seeds.rb` (TerritoryRegistryCrew depois do CareLinesCrew).

---

### Task 0: Prova técnica — regras oficiais das fichas de cadastro, dicionários e tabela do contrato

**Files:**
- Create: `vendor/ledi/8.7.0/docs/*.html`, `config/ledi/registration_dictionaries.yml`, `config/ledi/registration_cbos.yml`, `spec/services/ledi/registration_dictionaries_spec.rb`
- Modify: `vendor/ledi/8.7.0/SOURCE`, `app/services/ledi/version.rb`, `app/services/ledi/ficha_types.rb`, `spec/services/ledi/contract_spec.rb`, este plano (seção "Leitura preliminar", se algo divergir)
- Entregável fora do api: a tabela dos dicionários para o contrato (§14, R1) em `/Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/docs-mod24a/superpowers/plans/2026-10-23-module-24a-territory-registry-contracts.md` (não commitado; a coordenadora publica)

**Interfaces:**
- Consumes: IDL `vendor/ledi/8.7.0/idl/ras/cadastro_individual.thrift`, `cadastro_domiciliar.thrift`, `common.thrift`.
- Produces: `Ledi::FichaTypes.code("cadastro_individual") == 2`, `Ledi::FichaTypes.code("cadastro_domiciliar") == 3`; `TerritoryRegistry::Dictionary` (Task 7) lê `config/ledi/registration_dictionaries.yml` com a forma abaixo; `config/ledi/registration_cbos.yml` = `{ "individual" => [<cbo>…], "household" => [<cbo>…] }`.

Forma do YAML (uma seção por bloco da ficha; chave = campo do IDL em snake_case; `ledi:` = nome do campo no IDL):

```yaml
version: "8.7.0"
domains:
  raca_cor: { 1: "Branca", 2: "Preta", 3: "Amarela", 4: "Parda", 5: "Indígena" }
  justificativa_nao_possui_cpf: { 2: "Paciente desacordado, sem documentos", 3: "Recém-nascidos", … 13: "Situações de calamidade ou desastre" }
  motivo_saida: { 135: "Óbito", 136: "Mudança de território" }
  renda_familiar: { 1: "1/4 de salário mínimo", … 8: "Ausência de renda" }
  tipo_de_imovel: { 1: "Domicílio", … 99: "Outros" }
  # … todos os domínios citados nas seções (AbastecimentoDeAgua, DoencaCardiaca, CursoMaisElevado, TipoLogradouro…)
sections:
  identification:          # IdentificacaoUsuarioCidadaoThrift — só os campos que NÃO moram no paciente
    nome_pai_cidadao: { ledi: nomePaiCidadao, type: name }
    desconhece_nome_pai: { ledi: desconheceNomePai, type: bool }
    nacionalidade_cidadao: { ledi: nacionalidadeCidadao, type: code, domain: nacionalidade, required: true }
    # … codigo_ibge_municipio_nascimento, pais_nascimento, dt_naturalizacao, portaria_naturalizacao, dt_entrada_brasil,
    #   telefone_celular, email_cidadao, numero_nis_pis_pasep, estado_civil, tipo_sanguineo, dnv, desconhece_nome_mae
  sociodemographic: { … }  # InformacoesSocioDemograficasThrift + InformacoesSocioEconomicasThrift (prefixo socioeconomic_ não; mesmas chaves do IDL)
  health_conditions: { … } # CondicoesDeSaudeThrift
  street_situation: { … }  # EmSituacaoDeRuaThrift
  living_conditions: { … } # CondicaoMoradiaThrift + animais_no_domicilio, st_animais_no_domicilio, quantos_animais_no_domicilio + instituicao_permanencia (objeto)
types: [bool, code, codes, text, name, digits, date_month, date, count]
```

- [ ] **Step 1: Mover os cards F-24.1..F-24.8 do board #1 para In Progress** (um `gh project item-edit` por card, comando começando com `gh`; ids dos campos na memória "board-91-features-roadmap").

- [ ] **Step 2: Conferir a base** (`/opt/homebrew/bin/git -C apps/api/.claude/mod24a log -1 --format=%h origin/main` e os nomes do PRÉ-REQUISITO). Divergiu → PARE e reporte.

- [ ] **Step 3: Baixar a documentação** para `vendor/ledi/8.7.0/docs/` (cada página num arquivo; diretório novo e vazio; `curl -sSf` de `https://integracao.esusaps.bridge.ufsc.tech/ledi/documentacao/estrutura_arquivos/dicionario-fci.html`, `…/dicionario-fcd.html`, `…/estrutura_arquivos/header-transport.html`, `…/referencias/dicionario.html`, `…/regras/cbo.html`, `…/regras/validar_nome.html`, `…/regras/algoritmo_CNS.html`). Acrescentar a `SOURCE` um bloco `docs_fetched_on: 2026-10-23` + `docs_sha256:` por arquivo (`shasum -a 256`).

- [ ] **Step 4: Confirmar a "Leitura preliminar"** item a item contra as páginas baixadas (códigos 2/3, justificativas, motivos, renda, tipo de imóvel, raça/cor, sexo, obrigatoriedades e condicionais, ausência de `stAtivo` no IDL). Divergência → corrigir a seção deste plano e avisar a coordenadora (SendMessage) antes da Task 1.

- [ ] **Step 5: Confirmar o "sem CPF" do BPA-I** em `contracts/billing/bpa-interface.md` (origin/main do contracts) e no PDF citado ali (Layout_Exportacao_BPA, 05.00): com o paciente identificado pelo CNS, `Prd-cnspac` (60–74) com o CNS, `prd_cpf_pcnte` (339–349) em brancos e `prd_sem_cpf` (351) = `S`. Registrar o resultado na tabela do §14 (linha "BPA-I sem CPF") — a Task 5 usa este valor. Se o PDF disser outra coisa, a Task 5 segue o PDF e a divergência R8 muda.

- [ ] **Step 6: Escrever o teste do YAML**

```ruby
# spec/services/ledi/registration_dictionaries_spec.rb
require "rails_helper"

RSpec.describe "Dicionários das fichas de cadastro (LEDI 8.7.0)" do
  let(:yaml) { YAML.load_file(Rails.root.join("config/ledi/registration_dictionaries.yml")) }

  def idl_fields(file, struct)
    text = Ledi::Version.root.join("idl/ras", file).read
    text[/struct\s+#{struct}\s*\{(.*?)\n\}/m, 1].scan(/^\s*\d+:\s*(?:required|optional)\s+[\w.<>]+\s+(\w+)/).flatten
  end

  {
    "identification" => [ "cadastro_individual.thrift", "IdentificacaoUsuarioCidadaoThrift" ],
    "health_conditions" => [ "cadastro_individual.thrift", "CondicoesDeSaudeThrift" ],
    "street_situation" => [ "cadastro_individual.thrift", "EmSituacaoDeRuaThrift" ]
  }.each do |section, (file, struct)|
    it "#{section}: todo campo aponta para um campo de #{struct}, e a chave é o nome em snake_case" do
      fields = idl_fields(file, struct)
      yaml.fetch("sections").fetch(section).each do |key, spec|
        expect(fields).to include(spec.fetch("ledi"))
        expect(key).to eq(spec.fetch("ledi").underscore)
      end
    end
  end

  it "sociodemographic cobre InformacoesSocioDemograficas e InformacoesSocioEconomicas; living_conditions cobre CondicaoMoradia e os animais" do
    socio = idl_fields("cadastro_individual.thrift", "InformacoesSocioDemograficasThrift") +
            idl_fields("cadastro_individual.thrift", "InformacoesSocioEconomicasThrift")
    living = idl_fields("cadastro_domiciliar.thrift", "CondicaoMoradiaThrift") +
             %w[animaisNoDomicilio stAnimaisNoDomicilio quantosAnimaisNoDomicilio instituicaoPermanencia]
    expect(yaml["sections"]["sociodemographic"].values.map { |s| s["ledi"] }).to all(be_in(socio))
    expect(yaml["sections"]["living_conditions"].values.map { |s| s["ledi"] }).to all(be_in(living))
  end

  it "todo domínio citado existe, com códigos inteiros, e os valores da prova estão lá" do
    cited = yaml["sections"].values.flat_map(&:values).filter_map { |s| s["domain"] }.uniq
    expect(yaml["domains"].keys).to include(*cited)
    expect(yaml["domains"].values.flat_map(&:keys)).to all(be_a(Integer))
    expect(yaml["domains"]["raca_cor"]).to eq(1 => "Branca", 2 => "Preta", 3 => "Amarela", 4 => "Parda", 5 => "Indígena")
    expect(yaml["domains"]["motivo_saida"].keys).to eq([ 135, 136 ])
    expect(yaml["domains"]["justificativa_nao_possui_cpf"].keys.map(&:to_s)).to eq(TerritoryRegistry::Catalog::NO_CPF_REASONS)
    expect(yaml["domains"]["renda_familiar"].keys.map(&:to_s).sort).to eq(TerritoryRegistry::Catalog::INCOME_BANDS.sort)
    expect(yaml["domains"]["tipo_de_imovel"].keys.map(&:to_s)).to eq(TerritoryRegistry::Catalog::PROPERTY_TYPES)
  end

  it "tabelas de CBO: ACS, agente de endemias, enfermeiro e médicos da ESF nas duas" do
    cbos = YAML.load_file(Rails.root.join("config/ledi/registration_cbos.yml"))
    %w[515105 515140 223505 225142 225125 225130 225170].each do |code|
      expect(cbos["individual"]).to include(code)
      expect(cbos["household"]).to include(code)
    end
  end

  it "registra os tipos de ficha 2 e 3 com as classes geradas" do
    expect(Ledi::FichaTypes.code("cadastro_individual")).to eq(2)
    expect(Ledi::FichaTypes.code("cadastro_domiciliar")).to eq(3)
    expect(Ledi::FichaTypes.klass("cadastro_individual")).to eq(Br::Gov::Saude::Esusab::Ras::Cadastroindividual::CadastroIndividualThrift)
    expect(Ledi::FichaTypes.klass("cadastro_domiciliar")).to eq(Br::Gov::Saude::Esusab::Ras::Cadastrodomiciliar::CadastroDomiciliarThrift)
  end
end
```

`TerritoryRegistry::Catalog` (Task 1) ainda não existe: crie-o já aqui com as constantes de "Valores fixados" item 2 (`app/services/territory_registry/catalog.rb`), sem mais nada.

- [ ] **Step 7: Rodar e ver falhar** — `bin/test api -w mod24a spec/services/ledi/registration_dictionaries_spec.rb` → FAIL (YAML ausente; `UnknownType`).

- [ ] **Step 8: Escrever os dois YAML** a partir das páginas baixadas (todas as seções e domínios; `required`, `max`, `max_items` e `forbidden_when`/`required_when` só como **comentário** — as regras condicionais viram código na Task 7, uma por regra, citando o campo). Acrescentar a `Ledi::Version::TYPES` `cadastro_individual_types cadastro_domiciliar_types` e os dois tipos ao `Ledi::FichaTypes::REGISTRY` (Valores fixados, item 3). Em `spec/services/ledi/contract_spec.rb`, acrescentar as 7 structs do individual e as 5 do domiciliar ao mapa (mesma forma das existentes).

- [ ] **Step 9: Rodar** — `bin/test api -w mod24a spec/services/ledi/registration_dictionaries_spec.rb spec/services/ledi/contract_spec.rb spec/services/ledi/thrift_roundtrip_spec.rb` → PASS.

- [ ] **Step 10: Escrever o §14 do contrato** (R1) no worktree `docs-mod24a` (criar com `/opt/homebrew/bin/git -C <rota-saude>/docs worktree add ../.worktrees/docs-mod24a origin/main` se não existir): uma tabela por seção — `chave` | campo IDL | tipo | domínio (códigos: descrição) | obrigatório/condição — mais "Achados" (`stAtivo`, BPA-I sem CPF do Step 5, ausência de `stAtivo`). Avisar a coordenadora por SendMessage: "contrato 24a §14 pronto no worktree docs-mod24a (não commitado) — publique antes das tasks de formulário do dashboard".

- [ ] **Step 11: Commit**

```bash
/opt/homebrew/bin/git -C apps/api/.claude/mod24a add vendor/ledi/8.7.0/docs vendor/ledi/8.7.0/SOURCE config/ledi/registration_dictionaries.yml config/ledi/registration_cbos.yml app/services/territory_registry/catalog.rb app/services/ledi/version.rb app/services/ledi/ficha_types.rb spec/services/ledi/registration_dictionaries_spec.rb spec/services/ledi/contract_spec.rb
/opt/homebrew/bin/git -C apps/api/.claude/mod24a commit -m "feat: vendor LEDI registration rules and dictionaries for module 24a" -m "Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 1: Interruptor `territory_registry` e portão das rotas

**Files:**
- Create: `app/services/territory_registry/gate.rb`, `app/controllers/concerns/territory_registry_gate.rb`, `app/controllers/territory_registry/base_controller.rb`
- Modify: `app/services/platform/features.rb`, `spec/adr_pointers_spec.rb`, `config/initializers/filter_parameter_logging.rb`
- Test: `spec/services/territory_registry/gate_spec.rb`

**Interfaces:**
- Produces: `TerritoryRegistry::Gate::KEY = "territory_registry"`, `Gate.usable?(city) -> bool` (rotas, job), `Gate.enabled?(city) -> bool` (dentro da transação da cidade: `TeamPopulation`, `IdentityRefresh`); concern `TerritoryRegistryGate#require_territory_registry!`; `TerritoryRegistry::BaseController` com `render_failure(result)` (`{ error, path?, field? }`, status por `FAILURE_STATUS`), `render_error`, `forbid(reason = "missing_role")`, `not_found`, `uuid?`, `body_param(key)` (só `request.request_parameters`), `body` (`request.request_parameters` inteiro, sem `controller/action/id`), `page_limit` (padrão 50, máx. 100; inválido → 422 `invalid_limit`).
- `FAILURE_STATUS = { duplicate_code: :conflict, stale_version: :conflict, already_registered: :conflict, cpf_taken: :conflict, cns_taken: :conflict, cbo_not_allowed: :forbidden, out_of_scope: :forbidden, not_found: :not_found }` (o resto 422).

- [ ] **Step 1: Teste**

```ruby
# spec/services/territory_registry/gate_spec.rb
require "rails_helper"

RSpec.describe TerritoryRegistry::Gate do
  it "está no catálogo depois de care_lines e exige o prontuário utilizável" do
    entry = Platform::Features::CATALOG.find { |e| e.key == "territory_registry" }
    expect(entry.requires).to eq(%w[feature:clinical_record])
    expect(Platform::Features::KEYS.last(2)).to eq(%w[care_lines territory_registry])
  end

  it "usable? só com o interruptor ligado e o modo record; enabled? só o interruptor" do
    city = clinical_city!
    expect(described_class.usable?(city)).to be(false)
    Platform::Features.set!(city: city, key: "territory_registry", enabled: true, maintainer: ledi_maintainer!)
    expect(described_class.usable?(city)).to be(true)
    city.update!(record_mode: "off")
    expect(described_class.usable?(city)).to be(false)
    expect(described_class.enabled?(city)).to be(true)
    expect(described_class.usable?(nil)).to be(false)
  end
end
```

- [ ] **Step 2: Rodar** — `bin/test api -w mod24a spec/services/territory_registry/gate_spec.rb` → FAIL (`uninitialized constant TerritoryRegistry::Gate`).

- [ ] **Step 3: Implementar** a entrada do CATALOG (Valores fixados, item 1), `TerritoryRegistry::Gate` (forma do `CareLines::Gate`), o concern (403 `{ error: "feature_disabled", feature: "territory_registry" }`), `TerritoryRegistry::BaseController < ApplicationController` (`include Authentication, MfaStepUp, ::TerritoryRegistryGate`; `wrap_parameters false`; `before_action :require_territory_registry!`), os filtros de log (Global Constraints) e `VALID_RANGE = (1..42)`.

- [ ] **Step 4: Rodar** — `bin/test api -w mod24a spec/services/territory_registry/gate_spec.rb spec/adr_pointers_spec.rb spec/services/platform/features_spec.rb` → PASS.

- [ ] **Step 5: Commit** — `feat: add the territory_registry switch` (arquivos acima).

---

### Task 2: Migração de cidade, modelos e triggers de imutabilidade

**Files:**
- Create: `db/city_migrate/20261023900001_add_territory_registry.rb`, `app/models/{micro_area,household,household_version,family,individual_registration,individual_registration_version}.rb`, `spec/models/territory_registry_triggers_spec.rb`, `spec/support/territory_registry_helpers.rb`
- Modify: `db/city_schema.rb`, `db/city_triggers.sql`, `app/models/patient.rb` (só `encrypts` e a validação nova), `app/services/city_encryption.rb`

**Interfaces:**
- Produces (colunas; `enc` = `encrypts` não determinístico com a chave da cidade, coluna `text`; `det` = determinístico com `CityDeterministicKeyProvider`):
  - `micro_areas`: `id uuid`, `team_id → health_teams` (null false), `code string(2)` (CHECK `^(0[1-9]|[1-9][0-9])$`), `name string(60)` (CHECK `length(btrim(name)) > 0`), `acs_professional_id → professionals` (null), `active bool default true`, timestamps; único `(team_id, code)` `idx_micro_areas_team_code`.
  - `households`: `id`, `micro_area_id` (null false), `status string` (`active|inactive`), `version int` (≥ 1), `registered_at`, `registered_by_user_id → users`, timestamps; índice `(micro_area_id, id)`.
  - `household_versions`: `id`, `household_id`, `version int`, `micro_area_id`, `team_id`, `neighborhood_id → neighborhoods` (null), `property_type string` (CHECK nos `PROPERTY_TYPES`), `address enc`, `living_conditions enc`, `contact_phone enc` (null), `refused enc`, `families_snapshot enc`, `status string`, `origin string` (`save`), `saved_by_user_id`, `saved_by_professional_id`, `saved_cbo string(6)`, `created_at`; único `(household_id, version)`.
  - `families`: `id`, `household_id`, `responsible_patient_id → patients`, `income_band enc` (null), `members_count int` (CHECK 1..999, null), `living_since enc` (null, `"AAAA-MM"`), `moved_out bool default false`, timestamps; único parcial `(household_id, responsible_patient_id) WHERE NOT moved_out` `idx_families_one_responsible`.
  - `individual_registrations`: `id`, `patient_id`, `team_id`, `micro_area_id` (null), `household_id` (null), `family_id` (null), `is_responsible bool`, `version int`, `refused enc`, `registered_at`, `registered_by_user_id`, `exit_reason string` (null; CHECK `moved|death`), `exited_on date` (null; CHECK `(exit_reason IS NULL) = (exited_on IS NULL)`), `identity_changed_at` (null), timestamps; **único parcial `patient_id WHERE exit_reason IS NULL` `idx_individual_registrations_one_active`**; índices `(team_id, registered_at)`, `household_id`; CHECK `household_id IS NULL OR micro_area_id IS NOT NULL`.
  - `individual_registration_versions`: `id`, `registration_id`, `version`, `patient_id`, `team_id`, `micro_area_id`, `household_id`, `family_id`, `is_responsible`, `identity enc`, `identification enc`, `sociodemographic enc` (null), `health_conditions enc` (null), `street_situation enc` (null), `refused enc`, `exit_reason`, `exited_on`, `origin string` (CHECK `VERSION_ORIGINS`), `saved_by_user_id`, `saved_by_professional_id`, `saved_cbo string(6)`, `created_at`; único `(registration_id, version)`; índice `patient_id`.
  - `patients`: `cpf` passa a `null: true`; `no_cpf_reason enc`, `cns det` (único parcial `WHERE cns IS NOT NULL` `idx_patients_cns`), `match_key det` (índice `idx_patients_match_key`); CHECK `num_nonnulls(cpf, no_cpf_reason) = 1` `ck_patients_cpf_or_reason`.
  - `professionals`: `council`, `council_state`, `registration_number` passam a `null: true`; CHECK `num_nonnulls(council, council_state, registration_number) IN (0, 3)` `ck_professionals_council_all_or_none`.
- Modelos: JSON cifrado por `serialize :<col>, coder: JSON` + `encrypts :<col>`; booleano cifrado `refused` guardado como `"true"`/`"false"` com `refused?`; `inspect` de todo modelo novo só com `id` e ids de FK (`#<IndividualRegistrationVersion id=… registration_id=… version=…>`), sem dado. Associações: `MicroArea belongs_to :team, class_name: "HealthTeam"`, `belongs_to :acs_professional, class_name: "Professional", optional: true`; `Household belongs_to :micro_area; has_many :versions, class_name: "HouseholdVersion"; has_many :families`; `IndividualRegistration belongs_to :patient, :team; has_many :versions, class_name: "IndividualRegistrationVersion", foreign_key: :registration_id`; escopo `IndividualRegistration.active_registrations = where(exit_reason: nil)`; `IndividualRegistration#latest_version`, `Household#latest_version`.

Triggers novos em `db/city_triggers.sql` (bloco `DO $do$ … IF to_regclass('public.individual_registrations') IS NOT NULL …`):
- `household_versions`, `individual_registration_versions`: `BEFORE UPDATE OR DELETE … EXECUTE FUNCTION rota_reencryption_only('address','living_conditions','contact_phone','refused','families_snapshot')` / `('identity','identification','sociodemographic','health_conditions','street_situation','refused')`, e `BEFORE TRUNCATE … rota_append_only()`.
- `rota_individual_registration_guard()` (UPDATE OR DELETE): DELETE recusado; `id, patient_id, created_at` nunca mudam; com `OLD.exit_reason IS NOT NULL`, nada muda (salvo re-cifra de `refused`); `NEW.exit_reason` só de NULL para valor.
- `rota_micro_area_guard()`: DELETE recusado; `id, team_id, code` nunca mudam.
- `households` e `families`: DELETE recusado (`rota_no_delete()` nova, genérica: `RAISE EXCEPTION '% : DELETE refused', TG_TABLE_NAME`); `families.household_id` e `responsible_patient_id` nunca mudam (`rota_family_guard()`).
- `rota_patient_guard()` passa a: `encrypted` + `'cns','no_cpf_reason','match_key'`; trocar `cpf` só de NULL para valor (e então `NEW.no_cpf_reason IS NULL`); `no_cpf_reason` só vai a NULL junto com o CPF chegando.

- [ ] **Step 1: Teste dos triggers e das constraints**

```ruby
# spec/models/territory_registry_triggers_spec.rb
require "rails_helper"

RSpec.describe "Cadastro territorial — imutabilidade e constraints" do
  before { registry_city! }

  let(:registration) { registry_registration! }

  it "versão de cadastro individual nunca muda nem some; sob re-cifra só os campos cifrados" do
    version = registration.versions.first
    expect { version.update_columns(version: 9) }.to raise_error(ActiveRecord::StatementInvalid, /append-only/)
    expect { IndividualRegistrationVersion.where(id: version.id).delete_all }.to raise_error(ActiveRecord::StatementInvalid, /append-only/)
    CityEncryption.allowing_reencryption { version.update_columns(health_conditions: version.health_conditions) }
    expect { CityEncryption.allowing_reencryption { version.update_columns(origin: "exit") } }.to raise_error(ActiveRecord::StatementInvalid)
  end

  it "versão de domicílio idem" do
    version = registry_household!.versions.first
    expect { version.update_columns(property_type: "2") }.to raise_error(ActiveRecord::StatementInvalid, /append-only/)
  end

  it "um cadastro ativo por paciente; saída só uma vez; cadastro saído congela" do
    expect { IndividualRegistration.create!(registration.attributes.except("id", "created_at", "updated_at")) }
      .to raise_error(ActiveRecord::RecordNotUnique, /idx_individual_registrations_one_active/)
    registration.update_columns(exit_reason: "moved", exited_on: Time.zone.today)
    expect { registration.update_columns(exit_reason: "death") }.to raise_error(ActiveRecord::StatementInvalid)
    expect { registration.update_columns(version: 5) }.to raise_error(ActiveRecord::StatementInvalid)
  end

  it "patients: CPF ou justificativa, nunca os dois nem nenhum; CPF só de nulo para valor, levando a justificativa" do
    patient = Patient.create!(full_name: "Ana Clara Souza", birth_date: "2026-09-01", sex: "female", no_cpf_reason: "3")
    expect(patient.cpf_masked).to be_nil
    expect { patient.update_columns(no_cpf_reason: nil) }.to raise_error(ActiveRecord::StatementInvalid, /ck_patients_cpf_or_reason/)
    patient.update_columns(cpf: valid_cpf(91), no_cpf_reason: nil)
    expect { patient.update_columns(cpf: valid_cpf(92)) }.to raise_error(ActiveRecord::StatementInvalid, /identity columns never change/)
    expect { Patient.create!(full_name: "Sem Nada", birth_date: "2000-01-01", sex: "male") }.to raise_error(ActiveRecord::RecordInvalid)
  end

  it "microárea: código e equipe nunca mudam; código repetido na equipe recusado" do
    area = registry_micro_area!(code: "01")
    expect { area.update_columns(code: "02") }.to raise_error(ActiveRecord::StatementInvalid)
    expect { MicroArea.create!(team: area.team, code: "01", name: "Outra") }.to raise_error(ActiveRecord::RecordNotUnique)
  end

  it "professionals: conselho todo ou nenhum" do
    professional = registry_acs!.professional
    expect(professional.council).to be_nil
    expect { professional.update_columns(council_state: "PR") }.to raise_error(ActiveRecord::StatementInvalid, /ck_professionals_council_all_or_none/)
  end

  it "nada pessoal no inspect nem no banco em claro" do
    version = registration.versions.first
    expect(version.inspect).not_to include("Diabetes", "true", registration.patient.full_name)
    raw = ApplicationRecord.connection.select_value("SELECT health_conditions FROM individual_registration_versions WHERE id = '#{version.id}'")
    expect(raw).not_to include("status_tem_diabetes")
  end
end
```

`spec/support/territory_registry_helpers.rb` — `registry_city!` (= `care_city!` + `territory_registry` ligado + `ledi_ready!` com o PEC falso), `registry_team!(unit = nil)`, `registry_acs!(unit)` (usuário `health_professional` com perfil **sem conselho**, CNS válido, vínculo `515105` e membro da equipe), `registry_nurse!(unit)` (`223505`, membro), `registry_micro_area!(code:, team: nil, acs: nil)`, `registry_household!(micro_area: nil, **over)` e `registry_registration!(patient: nil, household: nil, **over)` — **cenário de teste** que grava pelas linhas diretas (versão 1 + cabeçalho) até a Task 9/10 existir; depois delas, os helpers passam a chamar `SaveHousehold`/`SaveRegistration` (Step 6 da Task 10). `valid_cpf(n)` e `valid_cns(n)` geram documentos com dígito válido.

- [ ] **Step 2: Rodar** — `bin/test api -w mod24a spec/models/territory_registry_triggers_spec.rb` → FAIL (tabelas ausentes).

- [ ] **Step 3: Escrever a migração** (cabeçalho com o porquê, como a `20261022900002`), o dump à mão em `db/city_schema.rb` (versão `2026_10_23_900001`, colunas em ordem alfabética como o dump), os triggers, os modelos, `CITY_KEYED_TARGETS` (+ `Patient :no_cpf_reason, :cns, :match_key`; `HouseholdVersion :address, :living_conditions, :contact_phone, :refused, :families_snapshot`; `Family :income_band, :living_since`; `IndividualRegistration :refused`; `IndividualRegistrationVersion :identity, :identification, :sociodemographic, :health_conditions, :street_situation, :refused`). `Patient`: `encrypts :no_cpf_reason`; `encrypts :cns, :match_key, deterministic: true, key_provider: CityDeterministicKeyProvider.new`; troque `validates :cpf, presence: true` por `validate { errors.add(:cpf, :blank) unless cpf.present? ^ no_cpf_reason.present? }`; `cpf_masked` nulo-seguro. `Professional`: `validates :council/:council_state/:registration_number` com `allow_nil: true` e `validate { errors.add(:council, :incomplete) unless [ council, council_state, registration_number ].compact.size.in?([ 0, 3 ]) }`. A migração termina com `execute File.read(Rails.root.join("db/city_triggers.sql"))`; `down` levanta `IrreversibleMigration`.

- [ ] **Step 4: Rodar** — `bin/test api -w mod24a spec/models/territory_registry_triggers_spec.rb spec/services/city_schema_spec.rb spec/architecture/city_encrypted_attributes_guard_spec.rb spec/models/patient_spec.rb spec/models/professional_spec.rb` → PASS.

- [ ] **Step 5: Commit** — `feat: add the territory registry tables and immutable versions`.

---

### Task 3: Módulo 10 — ACS e agente de endemias sem conselho

**Files:**
- Modify: `config/professionals/cbo_saude.yml`, `app/services/clinical_documents/json.rb:62`, `app/services/consultations/print.rb:103`, `app/controllers/concerns/professional_rendering.rb` (sem mudança de forma: `council: nil` passa)
- Test: `spec/commands/professionals/field_agent_spec.rb`

**Interfaces:**
- Consumes: `Professional` sem conselho (Task 2); `Professionals::OpenLink` (já recusa `council_mismatch` quando `entry.council && entry.council != professional.council`).
- Produces: CBO `515105` "Agente comunitário de saúde" e `515140` "Agente de combate às endemias", `council: null`.

- [ ] **Step 1: Teste**

```ruby
# spec/commands/professionals/field_agent_spec.rb
require "rails_helper"

RSpec.describe "Módulo 10 — ACS e agente de endemias sem conselho (F-24.2)" do
  before { clinical_city! }

  let(:admin) { staff_with("admin-campo@cidade.gov.br", "municipal_admin") }
  let(:unit) { create_unit("UBS Campo") }
  let(:user) { staff_with("acs-#{SecureRandom.hex(2)}@cidade.gov.br", "health_professional") }

  it "cria o perfil sem conselho e abre vínculo 515105 e 515140; CNS continua obrigatório" do
    result = Professionals::Create.call(user_id: user.id, attrs: { professional_name: "Rosana Moreira", cns: valid_cns(1) }, by: admin)
    expect(result).to be_ok
    professional = result.payload[:professional]
    expect(Professionals::OpenLink.call(professional: professional, health_unit_id: unit.id, cbo_code: "515105", by: admin)).to be_ok
    expect(Professionals::OpenLink.call(professional: professional, health_unit_id: unit.id, cbo_code: "515140", by: admin)).to be_ok
    expect(Professionals::Create.call(user_id: staff_with("x-#{SecureRandom.hex(2)}@c.gov.br", "health_professional").id,
                                      attrs: { professional_name: "Sem CNS" }, by: admin).details[:fields]).to include("cns")
  end

  it "perfil sem conselho não abre CBO que exige conselho; conselho pela metade é inválido" do
    professional = Professionals::Create.call(user_id: user.id, attrs: { professional_name: "Rosana Moreira", cns: valid_cns(2) }, by: admin).payload[:professional]
    expect(Professionals::OpenLink.call(professional: professional, health_unit_id: unit.id, cbo_code: "223505", by: admin).reason).to eq(:council_mismatch)
    other = staff_with("y-#{SecureRandom.hex(2)}@c.gov.br", "health_professional")
    expect(Professionals::Create.call(user_id: other.id, attrs: { professional_name: "Meio", cns: valid_cns(3), council: "COREN" }, by: admin).details[:fields])
      .to include("council")
  end

  it "tirar o conselho de quem tem vínculo que o exige → council_in_use" do
    nurse = care_nurse!(unit)
    result = Professionals::UpdateProfile.call(professional: nurse.professional, attrs: { council: nil, council_state: nil, registration_number: nil }, by: admin)
    expect(result.reason).to eq(:council_in_use)
  end

  it "documento e impressão não quebram com conselho nulo" do
    professional = Professionals::Create.call(user_id: user.id, attrs: { professional_name: "Rosana Moreira", cns: valid_cns(4) }, by: admin).payload[:professional]
    expect(ClinicalDocuments::Json.council(professional)).to be_nil
  end
end
```

- [ ] **Step 2: Rodar** — `bin/test api -w mod24a spec/commands/professionals/field_agent_spec.rb` → FAIL (`invalid_cbo`).

- [ ] **Step 3: Implementar** as duas linhas no YAML; `ClinicalDocuments::Json.council(professional)` → `nil` quando `professional&.council` é nulo; `Consultations::Print` linha "Conselho" omitida quando nulo. `Professional.normalizes :registration_number` devolve `""` para nil: troque para `->(v) { v.to_s.gsub(/\D/, "").presence }, apply_to_nil: true` (o CHECK do banco recusa `""` pela regex).

- [ ] **Step 4: Rodar** — `bin/test api -w mod24a spec/commands/professionals spec/requests/professionals_spec.rb spec/services/clinical_documents` → PASS.

- [ ] **Step 5: Commit** — `feat: let field agents register without a council (module 10)`.

---

### Task 4: Paciente sem CPF — CNS, chave de duplicados, CPF informado depois

**Files:**
- Create: `app/services/patients/{match_key,duplicates,pairs}.rb`, `app/commands/patients/{add_cpf,create_typed}.rb`, `lib/tasks/territory_registry.rake`
- Modify: `app/models/patient.rb`
- Test: `spec/models/patient_without_cpf_spec.rb`, `spec/commands/patients/add_cpf_spec.rb`, `spec/services/patients/duplicates_spec.rb`

**Interfaces:**
- Produces:
  - `Patients::MatchKey.for(full_name, birth_date) -> String|nil` (Desvio 1); `Patient before_save { self.match_key = Patients::MatchKey.for(full_name, birth_date) }`.
  - `Patients::Pairs.of(patient) -> ActiveRecord::Relation<Citizen>` — validados, não excluídos, `patient_id = patient.id` **ou** (só com CPF) `cpf = patient.cpf`.
  - `Patients::CreateTyped.call(attrs:, by:) -> Result` (`attrs`: `full_name social_name birth_date sex mother_name cpf no_cpf_reason cns`; falhas `:cpf_or_reason_required`, `:invalid_cpf`, `:invalid_cns`, `:invalid_patient` com `details: { path: "new_patient.<campo>" }`, `:cpf_taken`, `:cns_taken`). Com `cpf`: delega a `AddCpf` na mesma transação. Publica `patient.created` (`patient_id`, `user_id`).
  - `Patients::AddCpf.call(patient:, cpf:, by:) -> Result` — ordem: lock consultivo `Patients::Resolve.lock_key(cpf)` → pares do CPF (`order(:id).lock`) → `patient.lock!`; recusa `:cpf_taken` se outro paciente tem o CPF, `:invalid_transition` se o paciente já tem CPF; grava `cpf`, `no_cpf_reason = nil`; liga os pares **validados** sem paciente (`patient_id`, `patient.linked`); `Patients::Resolve.refresh!(patient)`; publica `patient.cpf_added` (`patient_id`, `user_id`).
  - `Patients::Duplicates.call(full_name:, birth_date:, mother_name: nil, limit: 10) -> Array<Patient>` — `Patient.where(match_key:)` + (com `mother_name`) filtro por `MatchKey.normalize(mother_name)` igual, ordenado por `created_at`, no máximo `limit`.
  - Rake `patients:backfill_match_keys` (por cidade, `CityConnection.with`, `find_each` + `update_columns(match_key:)`; idempotente; imprime só contagens).

- [ ] **Step 1: Testes**

```ruby
# spec/commands/patients/add_cpf_spec.rb
require "rails_helper"

RSpec.describe Patients::AddCpf do
  before { clinical_city! }

  let(:by) { verifier! }
  let(:typed) { Patients::CreateTyped.call(attrs: { "full_name" => "Joao Pedro Silva", "birth_date" => "1990-02-03", "sex" => "male", "no_cpf_reason" => "8" }, by: by).payload[:patient] }

  it "recusa CPF de outro paciente (cpf_taken) sem mudar nada" do
    other, _citizen = exam_patient!(1, age: 40, sex: "male")
    expect(described_class.call(patient: typed, cpf: other.cpf, by: by).reason).to eq(:cpf_taken)
    expect(typed.reload.cpf).to be_nil
    expect(typed.no_cpf_reason).to eq("8")
  end

  it "liga o par validado do CPF e os dados validados substituem os digitados" do
    citizen = verified_citizen!(7, full_name: "João Pedro da Silva", birth_date: "1990-02-03", sex: "male")
    result = described_class.call(patient: typed, cpf: citizen.cpf, by: by)
    expect(result).to be_ok
    expect(citizen.reload.patient_id).to eq(typed.id)
    expect(typed.reload.full_name).to eq("João Pedro da Silva")
    expect(typed.no_cpf_reason).to be_nil
    event = DomainEvent.where(name: "patient.cpf_added").last
    expect(event.payload.keys).to match_array(%w[patient_id user_id])
  end

  it "CPF inválido não grava; paciente com CPF não troca" do
    expect(described_class.call(patient: typed, cpf: "11111111111", by: by).reason).to eq(:invalid_cpf)
    described_class.call(patient: typed, cpf: valid_cpf(55), by: by)
    expect(described_class.call(patient: typed.reload, cpf: valid_cpf(56), by: by).reason).to eq(:invalid_transition)
  end
end
```

```ruby
# spec/services/patients/duplicates_spec.rb
require "rails_helper"

RSpec.describe Patients::Duplicates do
  before { clinical_city! }

  it "acha pelo nome normalizado + nascimento, filtra pela mãe, nunca pelo texto claro no banco" do
    a = Patients::CreateTyped.call(attrs: { "full_name" => "José  Antônio Pereira", "birth_date" => "1960-05-01", "sex" => "male",
                                            "mother_name" => "Maria Pereira", "no_cpf_reason" => "8" }, by: verifier!).payload[:patient]
    Patients::CreateTyped.call(attrs: { "full_name" => "Jose Antonio Pereira", "birth_date" => "1960-05-01", "sex" => "male",
                                        "mother_name" => "Ana Lima", "no_cpf_reason" => "8" }, by: verifier!)
    expect(described_class.call(full_name: "JOSE ANTONIO PEREIRA", birth_date: "1960-05-01").size).to eq(2)
    expect(described_class.call(full_name: "jose antonio pereira", birth_date: "1960-05-01", mother_name: "maria pereira")).to eq([ a ])
    expect(described_class.call(full_name: "Jose Antonio Pereira", birth_date: "1960-05-02")).to eq([])
    raw = ApplicationRecord.connection.select_value("SELECT match_key FROM patients WHERE id = '#{a.id}'")
    expect(raw).not_to include("jose")
  end
end
```

`spec/models/patient_without_cpf_spec.rb`: `CreateTyped` sem CPF e sem justificativa → `cpf_or_reason_required`; justificativa fora de `NO_CPF_REASONS` → `invalid_patient` `path: "new_patient.no_cpf_reason"`; CNS com dígito inválido → `invalid_cns`; dois pacientes com o mesmo CNS → o segundo `cns_taken`; `cpf_masked` nulo; o rake preenche `match_key` de paciente antigo (`update_columns(match_key: nil)` antes).

- [ ] **Step 2: Rodar** — `bin/test api -w mod24a spec/commands/patients/add_cpf_spec.rb spec/services/patients/duplicates_spec.rb spec/models/patient_without_cpf_spec.rb` → FAIL.

- [ ] **Step 3: Implementar** as interfaces acima. CNS do paciente validado por `Professionals::Cns.valid?` (mesmo algoritmo); nome por `TerritoryRegistry::Dictionary.valid_name?` (Task 7; até lá, 3..70 letras, espaço e apóstrofo — a Task 7 move a regra para lá). `RecordNotUnique` em `idx_patients_cns` → `:cns_taken`; em `index_patients_on_cpf` → `:cpf_taken`.

- [ ] **Step 4: Rodar** — os três specs + `spec/commands/patients/resolve_spec.rb spec/commands/patients/resolve_concurrency_spec.rb` → PASS.

- [ ] **Step 5: Commit** — `feat: allow patients without CPF with CNS and a blind duplicate key`.

---

### Task 5: Leitores de `patients.cpf` — paciente sem CPF nos módulos 16–23

**Files:**
- Modify: as linhas #4, #5, #9, #10, #11, #12, #14, #16, #18, #19, #20 do inventário; `app/models/ledi_generation_failure.rb` (`REASONS` + `patient_without_identifier responsible_without_identifier cbo_not_in_ledi_table`)
- Test: `spec/services/patient_cpf_readers_spec.rb`; specs existentes que mudam: `spec/services/ledi/fichas/individual_care_spec.rb` (stCidadaoNaoPossuiCpf), `spec/services/billing/bpa_file_spec.rb` (linha 03 sem CPF)

**Interfaces:**
- Consumes: `Patients::Pairs.of` (Task 4); valor do `prd_sem_cpf` confirmado na Task 0 Step 5.
- Produces: `Ledi::Fichas::ScreeningIdentity` com `citizen_cns` (todos os construtores passam a chave); `Billing::BpaFile::ILine` com `cns`; `Billing::BpaFile::WITH_CPF = "N"`, `WITHOUT_CPF_FLAG = "S"` (o nome antigo `WITHOUT_CPF` sai).

- [ ] **Step 1: Teste**

```ruby
# spec/services/patient_cpf_readers_spec.rb
require "rails_helper"

# Inventário do plano 24a: cada leitor de patients.cpf com um paciente sem CPF.
RSpec.describe "Leitores de patients.cpf com paciente sem CPF" do
  before { clinical_city! }

  let(:no_cpf) do
    Patients::CreateTyped.call(attrs: { "full_name" => "Lia Santos", "birth_date" => "2026-08-01", "sex" => "female",
                                        "no_cpf_reason" => "3", "cns" => valid_cns(31) }, by: verifier!).payload[:patient]
  end

  it "Patients::Pairs.of nunca casa por cpf nulo" do
    verified_citizen!(1) # um par qualquer
    expect(Patients::Pairs.of(no_cpf)).to be_empty
  end

  it "ficha de atendimento individual: CNS no lugar do CPF, nunca os dois" do
    identity = ledi_identity(citizen_cpf: nil, citizen_cns: no_cpf.cns)
    thrift = Ledi::Fichas::IndividualCare.new(identity: identity, care: ledi_care, source_id: SecureRandom.uuid).to_thrift(uuid: "1234567-#{SecureRandom.uuid}")
    child = thrift.atendimentosIndividuais.first
    expect([ child.cpfCidadao, child.cnsCidadao, child.stCidadaoNaoPossuiCpf ]).to eq([ nil, no_cpf.cns, true ])
  end

  it "BPA-I: crítica só sem CPF e sem CNS; linha 03 com CNS em 60–74, CPF em brancos e prd_sem_cpf S" do
    line = bpa_i_line(cpf: nil, cns: no_cpf.cns)
    row = Billing::BpaFile.build(header: bpa_header, c_lines: [], i_lines: [ line ]).lines.find { |l| l.start_with?("03") }
    expect(row[59, 15]).to eq(no_cpf.cns)
    expect(row[338, 11]).to eq(" " * 11)
    expect(row[350]).to eq("S")
    expect(Billing::Critic.patient_identifier_missing?(no_cpf)).to be(false)
    no_cpf.update_columns(cns: nil)
    expect(Billing::Critic.patient_identifier_missing?(no_cpf.reload)).to be(true)
  end

  it "FHIR do exame: CNS do paciente sem CPF; sem os dois, nil" do
    expect(Exams::Fhir::Identifiers.for_patient(no_cpf)).to eq(system: Exams::Fhir::Identifiers::CNS_SYSTEM, value: no_cpf.cns)
  end

  it "linha do tempo, inscrição em linha de cuidado e acesso ao prontuário não quebram com cpf nulo" do
    expect(Timeline::Person.for_patient(no_cpf).citizen_ids).to eq([])
    expect(ClinicalRecord::Access.open_attendances_of(no_cpf)).to eq([])
    expect(no_cpf.cpf.to_s[-3, 3]).to be_nil # Exams::Json#cpf_last3 (inventário #15)
  end

  it "documento clínico sem CPF continua recusado (inalcançável hoje: exige consulta)" do
    expect(ClinicalDocuments::Content::PatientIdentification.call(patient: no_cpf)).to be_failure
  end
end
```

Os helpers `ledi_identity`, `ledi_care`, `bpa_i_line`, `bpa_header` entram em `spec/support/territory_registry_helpers.rb` (montam os `Data` com valores fixos; `bpa_i_line` aceita `cpf:`/`cns:`). Ajuste o nome real de `Billing::Critic` exposto: extraia `Billing::Critic.patient_identifier_missing?(patient) -> bool` (`patient.nil? || (patient.cpf.blank? && patient.cns.blank?)`) e use-o no lugar da linha 77. Confira a assinatura real de `PatientIdentification` antes de escrever o último exemplo.

- [ ] **Step 2: Rodar** — `bin/test api -w mod24a spec/services/patient_cpf_readers_spec.rb` → FAIL.

- [ ] **Step 3: Implementar** cada linha do inventário (coluna "Mudança"). `ConsultationFicha.build`: `reasons << "patient_without_identifier" if patient.cpf.blank? && patient.cns.blank?`; identidade com `citizen_cpf: patient.cpf, citizen_cns: (patient.cns if patient.cpf.blank?)`. `IndividualCare`: `cpfCidadao: identity.citizen_cpf, cnsCidadao: identity.citizen_cns, stCidadaoNaoPossuiCpf: identity.citizen_cpf.nil?`. `BpaFile#i_row`: `opt_num(l.cpf ? nil : l.cns, 15)` no lugar de `blank(15)`, `opt_num(l.cpf, 11)` no lugar de `num(l.cpf, 11)`, flag `l.cpf ? WITH_CPF : WITHOUT_CPF_FLAG`. `Lines`: `cpf: p.patient.cpf, cns: p.patient.cns`.

- [ ] **Step 4: Rodar** — `bin/test api -w mod24a spec/services/patient_cpf_readers_spec.rb spec/services/ledi spec/services/billing spec/services/exams/fhir spec/services/timeline spec/services/clinical_record spec/commands/care_lines/enrollment_commands_spec.rb spec/requests/clinical_records_spec.rb` → PASS (os vizinhos diretos do inventário).

- [ ] **Step 5: Commit** — `fix: let patient CPF readers handle patients without CPF`.

---

### Task 6: Microáreas (F-24.1)

**Files:**
- Create: `app/services/territory_registry/access.rb`, `app/commands/territory_registry/save_micro_area.rb`, `app/controllers/territory_registry/micro_areas_controller.rb`, `app/services/territory_registry/json.rb` (só `micro_area` nesta task)
- Modify: `config/routes.rb`, `config/initializers/domain_events.rb`, `spec/initializers/domain_events_bindings_spec.rb`
- Test: `spec/requests/territory_registry/micro_areas_spec.rb`

**Interfaces:**
- Produces:
  - Rotas: `scope "/territory", module: "territory_registry"` — `get/post "admin/micro_areas"`, `patch "admin/micro_areas/:id"` (as rotas do 11 em `/territory/neighborhoods` continuam no `TerritoryController`).
  - `TerritoryRegistry::Access.admin?(user)`; `Access.membership(user, team) -> HealthTeamMember|nil` (ativo, perfil do usuário, papel `health_professional`); `Access.registrar_cbo?(cbo, ficha:)` (`:individual`/`:household`, pela `config/ledi/registration_cbos.yml`); `Access.registrar?(user, team, micro_area: nil) -> :ok | :out_of_scope | :cbo_not_allowed` (membro com CBO de `REGISTRAR_CBO_PREFIXES` → qualquer microárea da equipe; membro `515105` → só a microárea onde é o ACS; outro membro → `:cbo_not_allowed`; não membro → `:out_of_scope`); `Access.visible_micro_area_ids(user, team)`.
  - `TerritoryRegistry::SaveMicroArea.call(params:, by:, micro_area: nil) -> Result` — falhas `:invalid_code`, `:duplicate_code`, `:acs_not_in_team` (o profissional não é membro ativo da equipe com CBO `515105`), `:acs_without_cns`, `:team_kind_not_allowed`, `:invalid_micro_area` (`field: "name"`), `:not_found` (equipe). Evento `micro_area.changed` (`micro_area_id`, `team_id`, `user_id`).
  - `TerritoryRegistry::Json.micro_area(area) -> Hash` (contrato §2).

- [ ] **Step 1: Teste**

```ruby
# spec/requests/territory_registry/micro_areas_spec.rb
require "rails_helper"

RSpec.describe "Território — microáreas", type: :request do
  before { registry_city! }

  let(:admin) { care_admin! }
  let(:unit) { care_unit! }
  let(:team) { registry_team!(unit) }
  let(:acs) { registry_acs!(unit, team: team) }

  it "cria com step-up, lista por equipe, troca o ACS e desativa; só ids no evento" do
    sign_in_as(admin)
    json_post "/territory/admin/micro_areas", team_id: team.id, code: "01", name: "Norte", acs_professional_id: acs.professional.id
    expect(json_body).to eq("error" => "mfa_required")
    stepped_up!(admin)
    json_post "/territory/admin/micro_areas", team_id: team.id, code: "01", name: "Norte", acs_professional_id: acs.professional.id
    expect(response).to have_http_status(:created)
    expect(json_body).to include("code" => "01", "name" => "Norte", "active" => true,
                                 "team" => { "id" => team.id, "ine" => team.ine, "name" => team.name },
                                 "acs" => { "professional_id" => acs.professional.id, "name" => acs.professional.professional_name })
    id = json_body["id"]
    json_patch "/territory/admin/micro_areas/#{id}", active: false
    expect(json_body["active"]).to be(false)
    get "/territory/admin/micro_areas", params: { team_id: team.id }
    expect(json_body["micro_areas"].map { |a| a["id"] }).to eq([ id ])
    expect(DomainEvent.where(name: "micro_area.changed").last.payload.keys).to match_array(%w[micro_area_id team_id user_id])
  end

  it "recusas: código, duplicado, ACS de fora, equipe que não é 70/76, quem não é admin" do
    stepped_up!(admin)
    json_post "/territory/admin/micro_areas", team_id: team.id, code: "1", name: "X"
    expect(json_body).to eq("error" => "invalid_code")
    json_post "/territory/admin/micro_areas", team_id: team.id, code: "00", name: "X"
    expect(json_body).to eq("error" => "invalid_code")
    json_post "/territory/admin/micro_areas", team_id: team.id, code: "02", name: "X"
    json_post "/territory/admin/micro_areas", team_id: team.id, code: "02", name: "Y"
    expect(response).to have_http_status(:conflict)
    expect(json_body).to eq("error" => "duplicate_code")
    outsider = registry_acs!(care_unit!)
    json_post "/territory/admin/micro_areas", team_id: team.id, code: "03", name: "X", acs_professional_id: outsider.professional.id
    expect(json_body).to eq("error" => "acs_not_in_team")
    nurse = registry_nurse!(unit, team: team)
    json_post "/territory/admin/micro_areas", team_id: team.id, code: "03", name: "X", acs_professional_id: nurse.professional.id
    expect(json_body).to eq("error" => "acs_not_in_team")
    sign_in_as(nurse)
    get "/territory/admin/micro_areas", params: { team_id: team.id }
    expect(response).to have_http_status(:forbidden)
  end

  it "interruptor desligado → 403 feature_disabled" do
    Platform::Features.set!(city: City.find(Current.city.id), key: "territory_registry", enabled: false, maintainer: ledi_maintainer!)
    CityCatalog.reset_cache!
    sign_in_as(admin)
    get "/territory/admin/micro_areas", params: { team_id: team.id }
    expect(json_body).to eq("error" => "feature_disabled", "feature" => "territory_registry")
  end
end
```

`team_kind_not_allowed`: o CHECK de `health_teams.kind` já só aceita 70/76 hoje — a checagem fica (defesa para tipo novo no CNES) e o teste usa `update_columns(kind: …)` só se o CHECK permitir; senão, documente no spec que é inalcançável e teste a função `SaveMicroArea.team_kind_allowed?` direto. Mesmo raciocínio para `acs_without_cns` (CNS é `null: false` em `professionals`): teste `SaveMicroArea.acs_check` com um `Professional.new(cns: "")`.

- [ ] **Step 2: Rodar** — `bin/test api -w mod24a spec/requests/territory_registry/micro_areas_spec.rb` → FAIL.
- [ ] **Step 3: Implementar** (controller: `before_action { forbid unless Access.admin?(Current.user) }`, `require_step_up!` em `create update`; corpo só JSON; `team_id` inválido → 404 `not_found`).
- [ ] **Step 4: Rodar** — o spec + `spec/initializers/domain_events_bindings_spec.rb` → PASS.
- [ ] **Step 5: Commit** — `feat: add team micro-areas with the responsible community health agent`.

---

### Task 7: Dicionários e regras das fichas (validador único)

**Files:**
- Create: `app/services/territory_registry/{dictionary,rules,identity,address}.rb`
- Test: `spec/services/territory_registry/dictionary_spec.rb`, `spec/services/territory_registry/rules_spec.rb`

**Interfaces:**
- Consumes: `config/ledi/registration_dictionaries.yml` (Task 0).
- Produces:
  - `TerritoryRegistry::Dictionary.section(name) -> Hash`, `.domain(name) -> Hash{Integer=>String}`, `.valid_name?(value) -> bool` (regra "validar nome" baixada), `.parse(section, raw, prefix:) -> Result` — aceita só chaves da seção (desconhecida → `:invalid_registration`/`:invalid_household` com `path: "<prefix>.<chave>"`), converte pelo `type` (`bool` só `true/false`; `code` inteiro do domínio; `codes` lista sem repetição ≤ `max_items`; `text` ≤ `max` (padrão 100) — longo → `too_long` com `path`; `name`; `digits`; `date` ISO; `date_month` `AAAA-MM`; `count` inteiro > 0) e devolve `values` com as chaves do YAML.
  - `TerritoryRegistry::Rules.individual(input, person:, on:) -> Result` — `input` = `{ identification:, sociodemographic:, health_conditions:, street_situation:, refused:, is_responsible: }` já parseado; `person` = `{ sex:, birth_date:, race_color:, has_cpf:, nationality: }`; aplica cada regra condicional da "Leitura preliminar" itens 3, 4 e 6 (uma linha por regra, comentário com o campo LEDI) e devolve a **primeira** recusa (`:invalid_registration`, `path`).
  - `Rules.household(input, families:) -> Result` (itens 7 e 9; `path` `living_conditions.<campo>`, `families`, `property_type`).
  - `Rules.family(input, property_type:) -> Result` (`:invalid_family`, `path`).
  - `TerritoryRegistry::Address.parse(raw) -> Result` — chaves `zip street_type street number no_number complement neighborhood_id district reference` (R3); limites de "Valores fixados" item 4; `number` vazio exige `no_number: true` e vice-versa; `neighborhood_id` de bairro ativo (senão `path: "address.neighborhood_id"`); sem bairro, `district` obrigatório; `.to_patient(address, neighborhood:) -> Hash` com as chaves de `Patient::DEMOGRAPHICS` de endereço (Desvio 8).
  - `TerritoryRegistry::Identity.snapshot(patient) -> Hash` (`full_name social_name birth_date sex mother_name race_color ethnicity cpf cns no_cpf_reason`) e `.changed_fields(old, new) -> Array<String>` (só os nomes).

- [ ] **Step 1: Testes** (um exemplo por regra condicional; os mais importantes abaixo)

```ruby
# spec/services/territory_registry/rules_spec.rb
require "rails_helper"

RSpec.describe TerritoryRegistry::Rules do
  let(:woman) { { sex: "female", birth_date: Date.new(1990, 1, 1), race_color: "03", has_cpf: true } }
  let(:base) do
    { identification: { "nacionalidade_cidadao" => 1, "codigo_ibge_municipio_nascimento" => "4106902", "telefone_celular" => "41999990000",
                        "desconhece_nome_pai" => true },
      sociodemographic: { "status_deseja_informar_orientacao_sexual" => false, "status_deseja_informar_identidade_genero" => false },
      health_conditions: {}, street_situation: { "status_situacao_rua" => false }, refused: false, is_responsible: true }
  end

  def check(over = {}, person: woman, on: Date.new(2026, 10, 23)) = described_class.individual(base.deep_merge(over), person: person, on: on)

  it("aceita o mínimo válido") { expect(check).to be_ok }

  it "gestante: proibido para homem e fora de 9–60 anos" do
    expect(check({ health_conditions: { "status_eh_gestante" => true } }, person: woman.merge(sex: "male")).details[:path]).to eq("health_conditions.status_eh_gestante")
    expect(check({ health_conditions: { "status_eh_gestante" => true } }, person: woman.merge(birth_date: Date.new(1960, 1, 1)))).to be_failure
    expect(check(health_conditions: { "status_eh_gestante" => true })).to be_ok
  end

  it "doença cardíaca: lista obrigatória com o status true e proibida com false" do
    expect(check(health_conditions: { "status_teve_doenca_cardiaca" => true }).details[:path]).to eq("health_conditions.doenca_cardiaca")
    expect(check(health_conditions: { "status_teve_doenca_cardiaca" => false, "doenca_cardiaca" => [ 24 ] })).to be_failure
  end

  it "situação de rua: campos de rua proibidos com status_situacao_rua false" do
    expect(check(street_situation: { "status_situacao_rua" => false, "tempo_situacao_rua" => 1 }).details[:path]).to eq("street_situation.tempo_situacao_rua")
  end

  it "recusa: proíbe condições, sociodemográficos e situação de rua" do
    refused = described_class.individual({ identification: base[:identification], sociodemographic: { "status_frequenta_escola" => true },
                                           health_conditions: {}, street_situation: nil, refused: true, is_responsible: true }, person: woman, on: Date.new(2026, 10, 23))
    expect(refused.details[:path]).to eq("sociodemographic")
  end

  it "nacionalidade 1 exige município de nascimento e celular; 3 exige país e data de entrada" do
    expect(check(identification: { "codigo_ibge_municipio_nascimento" => nil }).details[:path]).to eq("identification.codigo_ibge_municipio_nascimento")
    expect(check(identification: { "nacionalidade_cidadao" => 3, "codigo_ibge_municipio_nascimento" => nil }).details[:path]).to eq("identification.pais_nascimento")
  end

  it "responsável por criança só antes dos 10 anos" do
    expect(check(sociodemographic: { "responsavel_por_crianca" => [ 1 ] }).details[:path]).to eq("sociodemographic.responsavel_por_crianca")
  end

  it "domicílio: famílias só em tipo 1; condição de moradia proibida em comércio; nu_moradores ≥ Σ membros" do
    expect(described_class.household({ property_type: "2", living_conditions: { "localizacao" => 83 }, refused: false }, families: []).details[:path])
      .to eq("living_conditions")
    expect(described_class.household({ property_type: "2", living_conditions: nil, refused: false }, families: [ { members_count: 2 } ]).details[:path])
      .to eq("families")
    expect(described_class.household({ property_type: "1", living_conditions: { "localizacao" => 83, "situacao_moradia_posse_terra" => 75, "nu_moradores" => "2" },
                                       refused: false }, families: [ { members_count: 3 } ]).details[:path]).to eq("living_conditions.nu_moradores")
  end
end
```

`spec/services/territory_registry/dictionary_spec.rb`: chave desconhecida → `path`; código fora do domínio → `path`; texto de 101 → `too_long` com `path`; lista com repetição recusada; `Address.parse` com logradouro de 31 → `invalid_household` `path: "address.street"`, número `1234A6` → `address.number`, sem número sem `no_number` → `address.number`; `Address.to_patient` com bairro do 11 → `address_district` = nome do bairro, sem número → `"SN"`. Códigos usados nos testes (`24`, `83`, `75`, `1`) são substituídos pelos reais do YAML se a Task 0 achar outros.

- [ ] **Step 2: Rodar** — `bin/test api -w mod24a spec/services/territory_registry/rules_spec.rb spec/services/territory_registry/dictionary_spec.rb` → FAIL.
- [ ] **Step 3: Implementar**; mover a regra de nome do `CreateTyped` para `Dictionary.valid_name?`.
- [ ] **Step 4: Rodar** → PASS (mais `spec/models/patient_without_cpf_spec.rb`).
- [ ] **Step 5: Commit** — `feat: validate registration records against the LEDI dictionaries`.

---

### Task 8: Fichas LEDI de cadastro (F-24.7)

**Files:**
- Create: `app/services/ledi/fichas/{registration_header,individual_registration,household_registration}.rb`, `app/services/ledi/{registration_ficha,individual_registration_ficha,household_registration_ficha}.rb`
- Modify: `app/services/ledi/ficha_sources.rb`, `app/services/ledi/transport.rb` (`rewrap`)
- Test: `spec/services/ledi/fichas/individual_registration_spec.rb`, `spec/services/ledi/fichas/household_registration_spec.rb`, `spec/services/ledi/registration_ficha_spec.rb`

**Interfaces:**
- Consumes: modelos de versão (Task 2), `Dictionary` (Task 7), `Catalog::RACE_TO_LEDI/SEX_TO_LEDI/LEDI_EXIT`.
- Produces:
  - `Ledi::Fichas::RegistrationHeader = Data.define(:cnes, :ine, :professional_cns, :cbo, :saved_at, :ibge_code)`; `#to_thrift -> UnicaLotacaoHeaderThrift`.
  - `Ledi::Fichas::IndividualRegistration.new(header:, values:, originating_uuid:, source_id:)` — `type "cadastro_individual"`, `competence` (mês de `header.saved_at` no fuso da cidade), `cnes`, `ine`, `source { type: "IndividualRegistrationVersion", id: }`, `to_thrift(uuid:)`: `tpCdsOrigem 3`, `fichaAtualizada = !originating_uuid.nil?`, `uuidFichaOriginadora = originating_uuid || uuid`, identidade (CPF **ou** CNS + `stNaoPossuiCpf` + `justificativaNaoPossuiCpf`; raça por `RACE_TO_LEDI`; sexo por `SEX_TO_LEDI`; nascimento em epoch ms; `microArea`/`stForaArea`; `statusEhResponsavel`; CPF/CNS do responsável quando não é), seções do dicionário (snake_case → campo LEDI pelo `ledi:` do YAML), recusa (só identificação), saída (`motivoSaidaCidadao`, `dataObito` com 135). `inspect` só com `source_id`.
  - `Ledi::Fichas::HouseholdRegistration.new(header:, values:, originating_uuid:, source_id:)` — `type "cadastro_domiciliar"`, `tipoEndereco 3`, `tpCdsOrigem 3`, endereço (`codigoIbgeMunicipio` e `numeroDneUf` do `CityProfile`; `microArea` da versão), `condicaoMoradia`, animais, `instituicaoPermanencia`, `familias` (`FamiliaRowThrift`: `cpfResponsavel` **ou** `numeroCnsResponsavel`, `dataNascimentoResponsavel`, `numeroMembrosFamilia`, `rendaFamiliar`, `resideDesde` (dia 1 do mês em epoch ms), `stMudanca`), `statusTermoRecusa`.
  - `Ledi::RegistrationFicha` (módulo de comportamento, `extend`): `generate(version, city: Current.city) -> :enqueued|:failed|:unusable|:exists`, `build(version) -> [ficha|nil, reasons]`, `originating_uuid(version) -> String|nil` (Desvio 6), `retry!(failure, by:)`, `regenerate(entry, by:, city:)` — mesmo contrato do `Ledi::ConsultationFicha` (sem `refresh!`/correção: versão nunca muda). Motivos: `unit_without_cnes`, `professional_without_cns`, `cbo_not_in_ledi_table`, `patient_without_identifier` (individual), `responsible_without_identifier` (domiciliar), `citizen_without_birth_date`.
  - `Ledi::IndividualRegistrationFicha` (`SOURCE_TYPE = "IndividualRegistrationVersion"`) e `Ledi::HouseholdRegistrationFicha` (`SOURCE_TYPE = "HouseholdVersion"`); `Ledi::FichaSources::REGISTRY` com as duas.
  - `Ledi::Transport.rewrap`: quando a ficha interna responde a `uuidFichaOriginadora` e ele era igual ao `uuid` antigo, troca os dois.

- [ ] **Step 1: Testes**

```ruby
# spec/services/ledi/fichas/individual_registration_spec.rb
require "rails_helper"

RSpec.describe Ledi::Fichas::IndividualRegistration do
  let(:header) { Ledi::Fichas::RegistrationHeader.new(cnes: "1234567", ine: "0000000001", professional_cns: "700000000000005", cbo: "515105",
                                                       saved_at: Time.zone.parse("2026-10-23 10:00"), ibge_code: "4106902") }
  let(:values) do
    { "identity" => { "full_name" => "Maria das Graças Lima", "birth_date" => "1953-08-20", "sex" => "female", "race_color" => "03",
                      "mother_name" => "Benedita Lima", "cpf" => "52998224725", "cns" => nil, "no_cpf_reason" => nil },
      "identification" => { "nacionalidade_cidadao" => 1, "codigo_ibge_municipio_nascimento" => "4106902", "telefone_celular" => "41999990000",
                            "desconhece_nome_pai" => true },
      "health_conditions" => { "status_tem_hipertensao_arterial" => true }, "sociodemographic" => {},
      "street_situation" => { "status_situacao_rua" => false }, "refused" => false, "micro_area_code" => "01", "is_responsible" => true,
      "exit" => nil }
  end

  def thrift(over = {}, originating: nil) = described_class.new(header: header, values: values.deep_merge(over), originating_uuid: originating, source_id: "v1").to_thrift(uuid: "1234567-u1")

  it "cadastro novo: originadora = uuid; atualização: a originadora; tpCdsOrigem 3" do
    expect(thrift.uuidFichaOriginadora).to eq("1234567-u1")
    expect(thrift.fichaAtualizada).to be(false)
    expect(thrift(originating: "1234567-u0").uuidFichaOriginadora).to eq("1234567-u0")
    expect(thrift.tpCdsOrigem).to eq(3)
  end

  it "raça/cor 03 do paciente (parda no BPA) sai 4 na ficha; 04 sai 3; sexo feminino sai 1" do
    expect(thrift.identificacaoUsuarioCidadao.racaCorCidadao).to eq(4)
    expect(thrift("identity" => { "race_color" => "04" }).identificacaoUsuarioCidadao.racaCorCidadao).to eq(3)
    expect(thrift.identificacaoUsuarioCidadao.sexoCidadao).to eq(1)
  end

  it "CPF ou CNS, nunca os dois; sem CPF: stNaoPossuiCpf e justificativa" do
    id = thrift.identificacaoUsuarioCidadao
    expect([ id.cpfCidadao, id.cnsCidadao, id.stNaoPossuiCpf ]).to eq([ "52998224725", nil, false ])
    id = thrift("identity" => { "cpf" => nil, "cns" => "700000000000005", "no_cpf_reason" => "3" }).identificacaoUsuarioCidadao
    expect([ id.cpfCidadao, id.cnsCidadao, id.stNaoPossuiCpf, id.justificativaNaoPossuiCpf ]).to eq([ nil, "700000000000005", true, 3 ])
  end

  it "recusa só leva a identificação; saída por óbito leva 135 e a data" do
    refused = thrift("refused" => true, "health_conditions" => nil, "sociodemographic" => nil, "street_situation" => nil)
    expect([ refused.statusTermoRecusaCadastroIndividualAtencaoBasica, refused.condicoesDeSaude, refused.informacoesSocioDemograficas ]).to eq([ true, nil, nil ])
    death = thrift("exit" => { "reason" => "death", "on" => "2026-10-20" })
    expect(death.saidaCidadaoCadastro.motivoSaidaCidadao).to eq(135)
    expect(death.saidaCidadaoCadastro.dataObito).to eq(Time.zone.parse("2026-10-20").to_i * 1000)
  end

  it "campo a campo com o exemplo de referência (spec/fixtures/ledi/cadastro_individual_referencia.json)" do
    reference = JSON.parse(Rails.root.join("spec/fixtures/ledi/cadastro_individual_referencia.json").read)
    expect(Ledi::Version.deserialize(Br::Gov::Saude::Esusab::Ras::Cadastroindividual::CadastroIndividualThrift,
                                     Ledi::Version.serialize(thrift)).identificacaoUsuarioCidadao.to_h_for_spec).to eq(reference["identificacaoUsuarioCidadao"])
    expect(described_class.new(header: header, values: values, originating_uuid: nil, source_id: "v1").inspect).not_to include("Maria", "52998224725")
  end
end
```

O "exemplo de referência" é o JSON que a Task 0 monta à mão a partir do dicionário oficial para estes `values` (um campo LEDI por chave, valores esperados); `to_h_for_spec` é um helper de spec (`spec/support/territory_registry_helpers.rb`) que transforma a struct Thrift em `{ "<campo>" => valor }` sem os nulos. `spec/services/ledi/fichas/household_registration_spec.rb`: `tipoEndereco 3`; `familias` com `cpfResponsavel` e sem `numeroCnsResponsavel` (e o contrário sem CPF); `resideDesde` = epoch ms do dia 1; famílias só em tipo 1; endereço com `codigoIbgeMunicipio` do perfil.

```ruby
# spec/services/ledi/registration_ficha_spec.rb
require "rails_helper"

RSpec.describe Ledi::IndividualRegistrationFicha do
  before do
    registry_city!
    stub_pec!
  end

  it "enfileira uma ficha por versão; a segunda versão aponta a originadora" do
    registration = registry_registration!
    first = LediOutboxEntry.find_by!(source_type: "IndividualRegistrationVersion", source_id: registration.versions.first.id)
    second_version = registry_new_version!(registration)
    second = LediOutboxEntry.find_by!(source_id: second_version.id)
    inner = ->(entry) { Ledi::Version.deserialize(Ledi::FichaTypes.klass("cadastro_individual"), Ledi::Transport.read(entry.bytes).dadoSerializado) }
    expect(inner.call(first).uuidFichaOriginadora).to eq(first.uuid)
    expect(inner.call(second).uuidFichaOriginadora).to eq(first.uuid)
    expect(inner.call(second).fichaAtualizada).to be(true)
  end

  it "reenvio da ficha NOVA recusada: uuid e originadora mudam juntos; a atualização seguinte aponta a nova" do
    registration = registry_registration!
    first = LediOutboxEntry.find_by!(source_id: registration.versions.first.id)
    first.update!(status: "sending")
    first.reject!([ Ledi::ErrorCodes::UNKNOWN.dup ])
    status, fresh = described_class.regenerate(first.reload, by: care_admin!)
    expect(status).to eq(:ok)
    inner = Ledi::Version.deserialize(Ledi::FichaTypes.klass("cadastro_individual"), Ledi::Transport.read(fresh.bytes).dadoSerializado)
    expect(inner.uuid).to eq(fresh.uuid)
    expect(inner.uuidFichaOriginadora).to eq(fresh.uuid)
    later = LediOutboxEntry.find_by!(source_id: registry_new_version!(registration).id)
    expect(Ledi::Version.deserialize(Ledi::FichaTypes.klass("cadastro_individual"), Ledi::Transport.read(later.bytes).dadoSerializado).uuidFichaOriginadora)
      .to eq(fresh.uuid)
    rewrapped = Ledi::Transport.read(Ledi::Transport.rewrap(fresh.bytes, uuid: "1234567-novo"))
    expect(Ledi::Version.deserialize(Ledi::FichaTypes.klass("cadastro_individual"), rewrapped.dadoSerializado).uuidFichaOriginadora).to eq("1234567-novo")
  end

  it "paciente sem CPF e sem CNS: não gerada, patient_without_identifier; nada no outbox" do
    registration = registry_registration!(patient: registry_patient_without_cpf!(cns: nil))
    failure = LediGenerationFailure.find_by!(source_type: "IndividualRegistrationVersion", source_id: registration.versions.first.id)
    expect(failure.reason_codes).to eq(%w[patient_without_identifier])
    expect(LediOutboxEntry.where(source_id: registration.versions.first.id)).to be_empty
  end
end
```

`registry_new_version!(registration)` e `registry_patient_without_cpf!(cns:)` entram nos helpers (cenário direto até a Task 10; depois via `SaveRegistration`).

- [ ] **Step 2: Rodar** — `bin/test api -w mod24a spec/services/ledi/fichas/individual_registration_spec.rb spec/services/ledi/fichas/household_registration_spec.rb spec/services/ledi/registration_ficha_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.** `build(version)`: unidade = `version.team.health_unit` (CNES); profissional = `Professional.find(version.saved_by_professional_id)`; `cbo_not_in_ledi_table` se `!Access.registrar_cbo?(version.saved_cbo, ficha:)`; valores decifrados da versão; `originating_uuid` = `LediOutboxEntry.where(source_type: SOURCE_TYPE, source_id: <ids das versões do mesmo cadastro/domicílio>).where.not(status: %w[rejected correction_pending]).order(:created_at, :id).pick(:uuid)`. Os comandos das Tasks 9/10 chamam `generate` **dentro** da transação da gravação (o `Ledi::Enqueue` usa savepoint próprio).
- [ ] **Step 4: Rodar** — os três + `spec/services/ledi spec/commands/ledi spec/requests/production_spec.rb` → PASS.
- [ ] **Step 5: Commit** — `feat: export individual and household registrations as LEDI records`.

---

### Task 9: Domicílio e famílias (F-24.4)

**Files:**
- Create: `app/commands/territory_registry/{save_household,save_family}.rb`, `app/controllers/territory_registry/{households,families}_controller.rb`, `app/services/territory_registry/trail.rb`
- Modify: `app/services/territory_registry/json.rb`, `app/services/patients/demographics.rb`, `app/controllers/billing/patients_controller.rb`, `app/controllers/attendance_controller.rb` (só o `render_failure` do novo motivo), `config/routes.rb`, `config/initializers/domain_events.rb`
- Test: `spec/requests/territory_registry/households_spec.rb`, `spec/commands/territory_registry/save_household_spec.rb`

**Interfaces:**
- Consumes: `Access.registrar?` (Task 6), `Address`, `Dictionary`, `Rules.household/family` (Task 7), `Ledi::HouseholdRegistrationFicha.generate` (Task 8).
- Produces:
  - Rotas: `get/post "households"`, `get/patch "households/:id"`, `post "households/:id/families"`, `patch "families/:id"`.
  - `TerritoryRegistry::SaveHousehold.call(params:, by:, household: nil) -> Result(household:, version:)` — corpo do contrato §4 (+ R3, `version` esperada no PATCH); recusas `:invalid_household` (`path`), `:stale_version`, `:out_of_scope`, `:cbo_not_allowed`. Uma versão nova (`version + 1`) com `families_snapshot` das famílias atuais, `Ledi::HouseholdRegistrationFicha.generate(version)`, endereço nos moradores (Desvio 8), evento `household.saved` (`household_id`, `household_version_id`, `user_id`). Travas: Desvio 12.
  - `TerritoryRegistry::SaveFamily.call(household:, params:, by:, family: nil) -> Result` — `{ responsible: { patient_id }, income_band, members_count, living_since, moved_out, version }`; recusas `:invalid_family` (`path`), `:stale_version`; grava a família e chama `SaveHousehold` para a nova versão do domicílio (mesma transação, mesmas travas).
  - `TerritoryRegistry::Json.household(household, version, families:, full: true)` (sem `living_conditions` com `full: false`), `Json.family(family)`.
  - `TerritoryRegistry::Trail.households_listed!(user:, micro_area_id:, household_ids:)`, `Trail.household_viewed!(user:, household_id:)`, `Trail.person_viewed!(user:, patient_id:, citizen_id: nil)` (Desvio 11).
  - `Patients::Demographics.save!` → `Result.fail(:address_from_household, details: { field: <primeira chave de endereço> })` quando `IndividualRegistration.active_registrations.where(patient_id:).where.not(household_id: nil).exists?` e há chave de endereço; `flags(patient)` ganha `address_from_household: bool`.

- [ ] **Step 1: Testes**

```ruby
# spec/requests/territory_registry/households_spec.rb
require "rails_helper"

RSpec.describe "Território — domicílios e famílias", type: :request do
  before { registry_city! }

  let(:unit) { care_unit! }
  let(:team) { registry_team!(unit) }
  let(:acs) { registry_acs!(unit, team: team) }
  let(:area) { registry_micro_area!(code: "01", team: team, acs: acs) }
  let(:neighborhood) { care_neighborhood!(unit) }
  let(:body) do
    { micro_area_id: area.id, property_type: "1", refused: false, contact_phone: "41999990000",
      address: { zip: "81000000", street_type: "081", street: "Rua das Acácias", number: "120", no_number: false, complement: "casa 2",
                 neighborhood_id: neighborhood.id, reference: "perto da escola" },
      living_conditions: { localizacao: 83, situacao_moradia_posse_terra: 75 } }
  end

  it "ACS da microárea cria e atualiza com versão; cada gravação é versão nova e ficha domiciliar" do
    sign_in_as(acs)
    json_post "/territory/households", body
    expect(response).to have_http_status(:created)
    expect(json_body).to include("version" => 1, "status" => "active", "property_type" => "1", "families" => [])
    id = json_body["id"]
    expect(LediOutboxEntry.where(ficha_type: "cadastro_domiciliar").count).to eq(1)
    json_patch "/territory/households/#{id}", body.merge(version: 1, contact_phone: "41988880000")
    expect(json_body["version"]).to eq(2)
    json_patch "/territory/households/#{id}", body.merge(version: 1)
    expect(response).to have_http_status(:conflict)
    expect(json_body).to eq("error" => "stale_version")
    expect(DomainEvent.where(name: "household.saved").last.payload.keys).to match_array(%w[household_id household_version_id user_id])
  end

  it "endereço maior que o layout do BPA é recusado com path, nada gravado" do
    sign_in_as(acs)
    json_post "/territory/households", body.deep_merge(address: { street: "R" * 31 })
    expect(json_body).to eq("error" => "invalid_household", "path" => "address.street")
    json_post "/territory/households", body.deep_merge(address: { number: "1234A6" })
    expect(json_body).to eq("error" => "invalid_household", "path" => "address.number")
    json_post "/territory/households", body.deep_merge(address: { complement: "C" * 11 })
    expect(json_body).to eq("error" => "invalid_household", "path" => "address.complement")
    expect(Household.count).to eq(0)
  end

  it "família: troca de responsável gera versão do domicílio e ficha; responsável repetido recusado" do
    sign_in_as(acs)
    json_post "/territory/households", body
    id = json_body["id"]
    patient = care_patient!(age: 40, sex: "female")
    json_post "/territory/households/#{id}/families", responsible: { patient_id: patient.id }, income_band: "3", members_count: 3,
                                                      living_since: "2020-03", version: 1
    expect(response).to have_http_status(:created)
    expect(Household.find(id).version).to eq(2)
    json_post "/territory/households/#{id}/families", responsible: { patient_id: patient.id }, members_count: 1, version: 2
    expect(json_body).to eq("error" => "invalid_family", "path" => "responsible.patient_id")
  end

  it "escopo: ACS de outra microárea → 403; domicílio de outra equipe por id → 404; lista com trilha por consulta, sem moradia" do
    other_acs = registry_acs!(unit, team: team)
    registry_micro_area!(code: "02", team: team, acs: other_acs)
    sign_in_as(other_acs)
    json_post "/territory/households", body
    expect(json_body).to eq("error" => "out_of_scope")
    household = registry_household!(micro_area: registry_micro_area!(code: "05", team: registry_team!(care_unit!)))
    get "/territory/households/#{household.id}"
    expect(response).to have_http_status(:not_found)
    sign_in_as(acs)
    json_post "/territory/households", body
    get "/territory/households", params: { micro_area_id: area.id }
    expect(json_body["households"].first).not_to have_key("living_conditions")
    expect(DomainEvent.where(name: "territory_households.listed").last.payload.keys).to match_array(%w[micro_area_id household_ids user_id])
  end
end
```

```ruby
# spec/commands/territory_registry/save_household_spec.rb
require "rails_helper"

RSpec.describe TerritoryRegistry::SaveHousehold do
  before { registry_city! }

  it "endereço do domicílio vai para os moradores com cadastro ativo; o faturamento não o edita mais" do
    household = registry_household!
    registration = registry_registration!(household: household)
    patient = registration.patient
    described_class.call(params: registry_household_params(household).deep_merge("address" => { "street" => "Rua Nova", "number" => "", "no_number" => true },
                                                                                  "version" => household.version), by: registry_acs_of(household), household: household)
    expect(patient.reload.address_street).to eq("Rua Nova")
    expect(patient.address_number).to eq("SN")
    result = ApplicationRecord.transaction { patient.lock!; Patients::Demographics.save!(patient, "address_street" => "Outra") }
    expect(result.reason).to eq(:address_from_household)
    expect(Patients::Demographics.flags(patient)).to include(address_from_household: true)
    expect(Patients::Demographics.save!(patient, "race_color" => "02")).to be_ok
  end

  it "domicílio inativo só sem famílias ativas" do
    household = registry_household!
    registry_family!(household)
    result = described_class.call(params: registry_household_params(household).merge("status" => "inactive", "version" => household.version),
                                  by: registry_acs_of(household), household: household)
    expect(result.details[:path]).to eq("status")
  end
end
```

- [ ] **Step 2: Rodar** — `bin/test api -w mod24a spec/requests/territory_registry/households_spec.rb spec/commands/territory_registry/save_household_spec.rb` → FAIL.
- [ ] **Step 3: Implementar.** Corpo inteiro do JSON (nunca query); `version` obrigatório no PATCH; `status` (`active|inactive`) aceito no PATCH (Desvio 9, R4); `team_id` da versão = equipe da microárea; `saved_cbo` = CBO do membro.
- [ ] **Step 4: Rodar** — os dois + `spec/requests/billing spec/requests/attendance_spec.rb spec/services/patients` → PASS.
- [ ] **Step 5: Commit** — `feat: register households and families as immutable versions`.

---

### Task 10: Cadastro individual, duplicados, recusa, saída e identidade validada (F-24.3, F-24.5)

**Files:**
- Create: `app/commands/territory_registry/{save_registration,exit_registration,identity_refresh}.rb`, `app/controllers/territory_registry/{registrations,duplicates}_controller.rb`
- Modify: `app/commands/patients/resolve.rb` (`refresh!`), `app/services/territory_registry/json.rb`, `config/routes.rb`, `config/initializers/domain_events.rb`, `spec/support/territory_registry_helpers.rb` (helpers passam pelos comandos)
- Test: `spec/requests/territory_registry/registrations_spec.rb`, `spec/requests/territory_registry/duplicates_spec.rb`, `spec/commands/territory_registry/save_registration_spec.rb`, `spec/commands/territory_registry/identity_refresh_spec.rb`, `spec/commands/territory_registry/locks_spec.rb`

**Interfaces:**
- Consumes: `Patients::CreateTyped/AddCpf/Duplicates` (Task 4), `Patients::Resolve`, `Rules.individual`, `Identity` (Task 7), `Ledi::IndividualRegistrationFicha.generate` (Task 8).
- Produces:
  - Rotas: `post "patients/duplicates"`, `get/post "registrations"`, `get/patch "registrations/:id"`, `post "registrations/:id/exit"`.
  - `TerritoryRegistry::SaveRegistration.call(params:, by:, registration: nil) -> Result(registration:, version:)` — corpo do contrato §5 + R2 (`team_id` sem domicílio) + R5 (`identification`); pessoa por `patient_id` | `citizen_id` (par validado: `Patients::Resolve.call`; não validado → `:citizen_not_verified` 409) | `new_patient` (`CreateTyped`); no PATCH, `cpf` → `AddCpf`, `cns` → grava no paciente (`cns_taken`), `version` esperada. Raça/cor e etnia moram **no paciente** (decisão "uma fonte por dado"): vêm em `new_patient.race_color/ethnicity` ou na raiz do corpo (paciente existente), em códigos do BPA, e são gravadas por `Patients::Demographics.save!` (R5). Recusas: `:already_registered`, `:invalid_registration` (`path`), `:cpf_or_reason_required`, `:stale_version`, `:cbo_not_allowed`, `:out_of_scope`, `:cpf_taken`, `:cns_taken`, `:not_found`. Família: `family_id` precisa ser do `household_id`; `is_responsible` precisa bater com `family.responsible_patient_id == patient.id` (`path: "is_responsible"`). Evento `territory_registration.saved` (`registration_id`, `registration_version_id`, `patient_id`, `user_id`).
  - `TerritoryRegistry::ExitRegistration.call(registration:, params:, by:) -> Result` — `{ reason: moved|death, on: AAAA-MM-DD }` (`on` ≤ hoje e ≥ nascimento); versão `origin: "exit"` + ficha; cadastro congela (trigger); evento `territory_registration.exited` (`registration_id`, `registration_version_id`, `patient_id`, `user_id`). Recusa `:invalid_registration` (`path: "reason"`/`"on"`), `:invalid_transition` (já saiu, 409).
  - `TerritoryRegistry::IdentityRefresh.call(patient) -> :versioned|:marked|:none` (Desvio 5); chamado por `Patients::Resolve.refresh!` **depois** do `patient.update!` e só quando algum de `full_name social_name mother_name birth_date sex` mudou.
  - `TerritoryRegistry::Json.registration(registration, version, identity_changed: …)` (contrato §2: `patient.has_cpf`, `patient.no_cpf_reason`, `identity_changed_at`), `Json.duplicate(patient)`.

- [ ] **Step 1: Testes**

```ruby
# spec/requests/territory_registry/registrations_spec.rb
require "rails_helper"

RSpec.describe "Território — cadastro individual", type: :request do
  before { registry_city! }

  let(:unit) { care_unit! }
  let(:team) { registry_team!(unit) }
  let(:acs) { registry_acs!(unit, team: team) }
  let(:area) { registry_micro_area!(code: "01", team: team, acs: acs) }
  let(:household) { registry_household!(micro_area: area) }
  let(:sections) { registry_registration_sections } # mínimo válido (Task 7)

  it "cria paciente sem CPF com CNS direto do cadastro; versão 1 e ficha individual com CNS" do
    sign_in_as(acs)
    json_post "/territory/registrations", sections.merge(household_id: household.id, is_responsible: false,
                                                         new_patient: { full_name: "Lia Santos", birth_date: "2026-08-01", sex: "female",
                                                                        mother_name: "Carla Santos", no_cpf_reason: "3", cns: valid_cns(41), race_color: "03" })
    expect(response).to have_http_status(:created)
    expect(json_body["patient"]).to include("has_cpf" => false, "no_cpf_reason" => "3")
    expect(json_body).to include("version" => 1, "status" => "valid", "exit" => nil)
    expect(LediOutboxEntry.where(ficha_type: "cadastro_individual").count).to eq(1)
    expect(DomainEvent.where(name: "territory_registration.saved").last.payload.keys)
      .to match_array(%w[registration_id registration_version_id patient_id user_id])
  end

  it "sem CPF e sem CNS: grava, ficha não gerada; informar o CNS gera a ficha da nova versão" do
    sign_in_as(acs)
    json_post "/territory/registrations", sections.merge(household_id: household.id, is_responsible: false,
                                                         new_patient: { full_name: "Rui Alves", birth_date: "1970-01-01", sex: "male", no_cpf_reason: "5" })
    expect(response).to have_http_status(:created)
    id = json_body["id"]
    expect(LediGenerationFailure.last.reason_codes).to eq(%w[patient_without_identifier])
    json_patch "/territory/registrations/#{id}", sections.merge(household_id: household.id, is_responsible: false, version: 1, cns: valid_cns(42))
    expect(json_body["version"]).to eq(2)
    expect(LediOutboxEntry.where(ficha_type: "cadastro_individual").count).to eq(1)
  end

  it "CPF de outro paciente → 409 cpf_taken; sem CPF e sem justificativa → 422" do
    other, _ = exam_patient!(3, age: 30, sex: "female")
    sign_in_as(acs)
    json_post "/territory/registrations", sections.merge(household_id: household.id, is_responsible: false,
                                                         new_patient: { full_name: "Ana Paula", birth_date: "1996-01-01", sex: "female", no_cpf_reason: "8" })
    json_patch "/territory/registrations/#{json_body['id']}", sections.merge(household_id: household.id, is_responsible: false, version: 1, cpf: other.cpf)
    expect(response).to have_http_status(:conflict)
    expect(json_body).to eq("error" => "cpf_taken")
    json_post "/territory/registrations", sections.merge(household_id: household.id, is_responsible: false,
                                                         new_patient: { full_name: "Sem Doc", birth_date: "1996-01-01", sex: "female" })
    expect(json_body).to eq("error" => "cpf_or_reason_required")
  end

  it "par validado sem paciente cria o paciente pelo Resolve; segundo cadastro ativo → already_registered" do
    citizen = verified_citizen!(9)
    sign_in_as(acs)
    json_post "/territory/registrations", sections.merge(household_id: household.id, is_responsible: false, citizen_id: citizen.id)
    expect(response).to have_http_status(:created)
    expect(citizen.reload.patient_id).to eq(json_body["patient"]["id"])
    json_post "/territory/registrations", sections.merge(household_id: household.id, is_responsible: false, patient_id: citizen.patient_id)
    expect(json_body).to eq("error" => "already_registered")
  end

  it "recusa com termo; saída por óbito congela; técnico de enfermagem não grava; CPF nunca na URL" do
    sign_in_as(acs)
    patient = care_patient!(age: 50, sex: "male")
    json_post "/territory/registrations", household_id: household.id, is_responsible: false, patient_id: patient.id, refused: true,
                                          identification: sections[:identification]
    expect(json_body).to include("refused" => true, "status" => "refused")
    id = json_body["id"]
    json_post "/territory/registrations/#{id}/exit", reason: "death", on: Time.zone.today.iso8601
    expect(json_body["exit"]).to eq("reason" => "death", "on" => Time.zone.today.iso8601)
    json_post "/territory/registrations/#{id}/exit", reason: "moved", on: Time.zone.today.iso8601
    expect(response).to have_http_status(:conflict)
    technician = screener!(unit, cbo: "322245")
    HealthTeamMember.create!(professional: technician.professional, health_team: team, cbo_code: "322245", started_on: Time.zone.today - 1)
    sign_in_as(technician)
    json_post "/territory/registrations", sections.merge(household_id: household.id, is_responsible: false, patient_id: care_patient!(age: 30, sex: "male").id)
    expect(json_body).to eq("error" => "cbo_not_allowed")
    get "/territory/registrations", params: { patient_id: patient.id, cpf: patient.cpf }
    expect(response.body).not_to include(patient.cpf)
  end

  it "leitura nominal com trilha por pessoa; enfermeiro da equipe lê; profissional de fora → 404" do
    registration = registry_registration!(household: household)
    sign_in_as(registry_nurse!(unit, team: team))
    get "/territory/registrations/#{registration.id}"
    expect(response).to have_http_status(:ok)
    event = DomainEvent.where(name: "clinical_record.viewed").last
    expect(event.payload).to include("access" => "territory_registry", "patient_id" => registration.patient_id)
    sign_in_as(registry_nurse!(care_unit!))
    get "/territory/registrations/#{registration.id}"
    expect(response).to have_http_status(:not_found)
  end
end
```

```ruby
# spec/commands/territory_registry/locks_spec.rb
require "rails_helper"

RSpec.describe "Cadastro individual — concorrência" do
  include LockWait
  before { registry_city! }

  it "dois cadastros do mesmo paciente ao mesmo tempo: um grava, o outro already_registered; nunca dois ativos" do
    household = registry_household!
    patient = care_patient!(age: 40, sex: "female")
    by = registry_acs_of(household)
    params = registry_registration_params(household).merge("patient_id" => patient.id)
    results = run_in_threads(2) { TerritoryRegistry::SaveRegistration.call(params: params, by: by) }
    expect(results.map { |r| r.ok? ? :ok : r.reason }).to match_array([ :ok, :already_registered ])
    expect(IndividualRegistration.active_registrations.where(patient_id: patient.id).count).to eq(1)
  end

  it "gravar o domicílio enquanto um cadastro entra nele: sem deadlock (paciente → domicílio → família → cadastro)" do
    household = registry_household!
    by = registry_acs_of(household)
    patient = care_patient!(age: 40, sex: "female")
    results = run_in_threads(2) do |i|
      if i.zero?
        TerritoryRegistry::SaveRegistration.call(params: registry_registration_params(household).merge("patient_id" => patient.id), by: by)
      else
        TerritoryRegistry::SaveHousehold.call(params: registry_household_params(household).merge("version" => household.version), by: by, household: household)
      end
    end
    expect(results).to all(satisfy { |r| r.ok? || r.reason == :stale_version })
  end
end
```

`run_in_threads(n) { |i| … }` — confira o helper real em `spec/support/lock_wait.rb` (se não houver, use o padrão de `spec/commands/patients/resolve_concurrency_spec.rb`: threads com `CityConnection.with`, `Queue#pop(timeout:)` e limpeza no `after`, memória "Verificação do módulo 09").

```ruby
# spec/commands/territory_registry/identity_refresh_spec.rb
require "rails_helper"

RSpec.describe TerritoryRegistry::IdentityRefresh do
  before { registry_city! }

  it "validação no posto depois do cadastro: dados validados substituem os digitados, nova versão e ficha, o que mudou fica marcado" do
    registration = registry_registration!(patient: Patients::CreateTyped.call(attrs: { "full_name" => "Joao Silva", "birth_date" => "1980-01-01",
                                                                                       "sex" => "male", "no_cpf_reason" => "8", "cns" => valid_cns(5) },
                                                                              by: verifier!).payload[:patient])
    citizen = verified_citizen!(12, full_name: "João da Silva", birth_date: "1980-01-01", sex: "male")
    Patients::AddCpf.call(patient: registration.patient, cpf: citizen.cpf, by: verifier!)
    registration.reload
    expect(registration.version).to eq(2)
    expect(registration.identity_changed_at).to be_present
    expect(registration.latest_version.origin).to eq("identity_refresh")
    expect(registration.latest_version.identity["full_name"]).to eq("João da Silva")
    expect(LediOutboxEntry.where(source_id: registration.latest_version.id)).to exist
  end

  it "sem mudança de identidade, nada; interruptor desligado, só marca" do
    registration = registry_registration!
    expect(described_class.call(registration.patient)).to eq(:none)
    Platform::Features.set!(city: City.find(Current.city.id), key: "territory_registry", enabled: false, maintainer: ledi_maintainer!)
    registration.patient.update!(full_name: "Outro Nome Qualquer")
    expect(described_class.call(registration.patient)).to eq(:marked)
    expect(registration.reload.version).to eq(1)
  end
end
```

`spec/requests/territory_registry/duplicates_spec.rb`: `POST /territory/patients/duplicates` com nome/nascimento/mãe no corpo → `candidates` (`patient_id display_name birth_date has_cpf`), no máximo 10, trilha `clinical_record.viewed` por candidato; com os dados na query (`?full_name=`) → 422 `invalid_request` (o corpo manda; nada lido da URL); ACS de outra equipe também consulta (a busca é da cidade), mas sem papel `health_professional` → 403.

- [ ] **Step 2: Rodar** — os cinco specs → FAIL.
- [ ] **Step 3: Implementar.** Ordem de travas (Desvio 12): `AddCpf` (se `cpf`) → `patient.lock!` → `household.lock("FOR SHARE")` → `family.lock("FOR SHARE")` → `registration.lock!`. `RecordNotUnique` em `idx_individual_registrations_one_active` → `:already_registered`. Escopo (`out_of_scope`): equipe da microárea do domicílio, ou `team_id` sem domicílio, entre as do usuário (`Access.registrar?`). Status no JSON por `TerritoryRegistry::Validity` (Task 12 cria; até lá `"valid"`/`"refused"`/`"exited"` direto — a Task 12 troca pela função). Depois, trocar os helpers de cenário (Task 2) por chamadas aos comandos e rodar de novo os specs das Tasks 2 e 8.
- [ ] **Step 4: Rodar** — os cinco + `spec/models/territory_registry_triggers_spec.rb spec/services/ledi/registration_ficha_spec.rb spec/commands/patients` → PASS.
- [ ] **Step 5: Commit** — `feat: add individual registration with refusal, exit and validated identity refresh`.

---

### Task 11: Equipe e óbito pelo cadastro — mudança no módulo 23 (F-24.8)

**Files:**
- Create: `app/services/territory_registry/teams.rb`
- Modify: `app/services/care_lines/team_population.rb`, `app/services/care_lines/facts.rb`
- Test: `spec/services/territory_registry/teams_spec.rb`; specs do 23 que mudam: `spec/services/care_lines/team_population_spec.rb` (exemplos novos), `spec/services/care_lines/facts_spec.rb` (exemplo novo em `deceased`), `spec/services/care_lines/suggest_spec.rb` (óbito pelo cadastro cala a sugestão), `spec/services/care_lines/preview_spec.rb` (paciente sem CPF da equipe entra no denominador)

**Interfaces:**
- Consumes: `IndividualRegistration` (Task 2), `TerritoryRegistry::Gate.enabled?`.
- Produces:
  - `TerritoryRegistry::Teams.for_patients(patient_ids = nil, team: nil) -> Hash{patient_id => team_id}` — cadastros ativos (`exit_reason IS NULL`) de equipes ativas; com `team:`, só os daquela equipe; uma consulta.
  - `CareLines::TeamPopulation.patient_teams(patient_ids = nil, involving_team: nil)`: com `TerritoryRegistry::Gate.enabled?(Current.city)`, `consulta.merge(território).merge(cadastro)` — o cadastro vence (contrato §9); com `involving_team`, acrescenta os cadastrados daquela equipe ao conjunto; o paciente com cadastro em outra equipe sai da equipe da consulta. Saída por `moved` não conta (as regras antigas voltam). Desligado: a regra de hoje, byte a byte.
  - `CareLines::Facts.deceased(patient_ids)`: inscrição encerrada por `death` **OU** `individual_registrations.exit_reason = 'death'` (uma consulta, `UNION`).

- [ ] **Step 1: Testes** (acréscimos aos specs do 23)

```ruby
# spec/services/care_lines/team_population_spec.rb (novos exemplos)
describe "cadastro individual (módulo 24a, contrato §9)" do
  before { Platform::Features.set!(city: City.find(Current.city.id), key: "territory_registry", enabled: true, maintainer: ledi_maintainer!) }

  it "cadastro ativo vence a consulta: equipe da microárea do domicílio" do
    patient = care_patient!(age: 60, sex: "male", problems: [ %w[ciap2 T90] ], unit: unit_a, doctor: doctor_a)
    registry_registration!(patient: patient, household: registry_household!(micro_area: registry_micro_area!(code: "01", team: team_b)))
    expect(described_class.patient_teams([ patient.id ])).to eq(patient.id => team_b.id)
    expect(described_class.patient_teams(nil, involving_team: team_b)).to include(patient.id => team_b.id)
    expect(described_class.patient_teams(nil, involving_team: team_a)).not_to include(patient.id => team_a.id)
  end

  it "paciente sem CPF só entra pelo cadastro" do
    patient = registry_patient_without_cpf!(cns: valid_cns(8))
    registry_registration!(patient: patient, household: registry_household!(micro_area: registry_micro_area!(code: "01", team: team_a)))
    expect(described_class.patient_teams(nil, involving_team: team_a)).to include(patient.id => team_a.id)
  end

  it "saída por mudança devolve a regra antiga; interruptor desligado ignora o cadastro" do
    patient = care_patient!(age: 60, sex: "male", problems: [ %w[ciap2 T90] ], unit: unit_a, doctor: doctor_a)
    registration = registry_registration!(patient: patient, household: registry_household!(micro_area: registry_micro_area!(code: "01", team: team_b)))
    TerritoryRegistry::ExitRegistration.call(registration: registration, params: { "reason" => "moved", "on" => Time.zone.today.iso8601 }, by: care_admin!)
    expect(described_class.patient_teams([ patient.id ])).to eq(patient.id => team_a.id)
    other = care_patient!(age: 61, sex: "male", problems: [ %w[ciap2 T90] ], unit: unit_a, doctor: doctor_a)
    registry_registration!(patient: other, household: registry_household!(micro_area: registry_micro_area!(code: "02", team: team_b)))
    Platform::Features.set!(city: City.find(Current.city.id), key: "territory_registry", enabled: false, maintainer: ledi_maintainer!)
    expect(described_class.patient_teams([ other.id ])).to eq(other.id => team_a.id)
  end

  it "o número de consultas continua constante com o cadastro ligado" do
    3.times { registry_registration! }
    small = count_queries { described_class.patient_teams(nil, involving_team: team_a) }
    5.times { registry_registration! }
    expect(count_queries { described_class.patient_teams(nil, involving_team: team_a) }).to eq(small)
  end
end
```

```ruby
# spec/services/care_lines/facts_spec.rb (novo exemplo)
it "deceased: saída do cadastro por óbito também marca, numa consulta" do
  registry_city!
  registration = registry_registration!
  TerritoryRegistry::ExitRegistration.call(registration: registration, params: { "reason" => "death", "on" => Time.zone.today.iso8601 }, by: care_admin!)
  moved = registry_registration!
  TerritoryRegistry::ExitRegistration.call(registration: moved, params: { "reason" => "moved", "on" => Time.zone.today.iso8601 }, by: care_admin!)
  expect(described_class.deceased([ registration.patient_id, moved.patient_id ])).to eq(Set[registration.patient_id])
end
```

`spec/services/care_lines/suggest_spec.rb`: paciente com cadastro saído por óbito não recebe sugestão nem sugestão de encerramento. `spec/services/care_lines/preview_spec.rb`: paciente sem CPF de 65 anos com cadastro na equipe entra no denominador de `elderly_c6`.

- [ ] **Step 2: Rodar** — `bin/test api -w mod24a spec/services/care_lines/team_population_spec.rb spec/services/care_lines/facts_spec.rb spec/services/care_lines/suggest_spec.rb spec/services/care_lines/preview_spec.rb spec/services/territory_registry/teams_spec.rb` → FAIL nos exemplos novos.
- [ ] **Step 3: Implementar** (comentário de cabeçalho do `TeamPopulation` atualizado: "Cadastro individual ativo (24a, ADR 0042) → equipe da microárea; P18/P22 para quem não tem").
- [ ] **Step 4: Rodar** — `bin/test api -w mod24a spec/services/care_lines spec/requests/care_lines spec/services/territory_registry/teams_spec.rb` → PASS (todo o 23: a mudança é transversal a ele).
- [ ] **Step 5: Commit** — `feat: derive the person's team and death from the territory registry`.

---

### Task 12: Validade de 24 meses e quem falta cadastrar (F-24.6)

**Files:**
- Create: `app/services/territory_registry/{validity,registry_status}.rb`, `app/controllers/territory_registry/registry_status_controller.rb`
- Modify: `app/services/care_lines/team_population.rb` (extrai `members`), `config/routes.rb`, `app/services/territory_registry/json.rb`
- Test: `spec/services/territory_registry/validity_spec.rb`, `spec/requests/territory_registry/registry_status_spec.rb`

**Interfaces:**
- Consumes: `CareLines::TeamPopulation` (Task 11).
- Produces:
  - `TerritoryRegistry::Validity.status(registration, today:) -> String` — `exited` (saída) · `refused` (recusa) · `expired` (`registered_at.to_date >> 24 <= today`) · `due_soon` (`(registered_at.to_date >> 24) - today <= 60`) · `valid`.
  - `CareLines::TeamPopulation.members(team:) -> { patient_ids: Array, citizen_ids: Array }` — a população da equipe sem regra de linha (o mesmo miolo do `call`, sem `Facts.deceased`/elegibilidade); `call` passa a usá-lo.
  - `TerritoryRegistry::RegistryStatus.page(team:, micro_area: nil, status: nil, before: nil, limit:, today:) -> { people: [<person_row>], next_before: }` — população = `members(team)` ∪ cadastros ativos da equipe, menos óbitos (`Facts.deceased`); `micro_area` filtra pelos cadastros dela (quem não tem cadastro não tem microárea: só entra sem o filtro); ordem por `person_ref` (`patient:<uuid>`/`citizen:<uuid>`) decrescente, `before` = o `person_ref` da última (R9); decifra só a página.
  - `RegistryStatus.count(team:, micro_area: nil, today:) -> { valid:, due_soon:, expired:, missing: }` (sem trilha; recusados e saídos fora das quatro contagens).
  - Rotas: `get "registry_status"`, `get "registry_status/count"`; escopo: membro da equipe (qualquer CBO de cadastro; ACS só a própria microárea → `micro_area_id` obrigatório para ele, senão 403 `out_of_scope`).

- [ ] **Step 1: Testes**

```ruby
# spec/services/territory_registry/validity_spec.rb
require "rails_helper"

RSpec.describe TerritoryRegistry::Validity do
  let(:today) { Date.new(2026, 10, 23) }

  def reg(at, refused: false, exit: nil) = instance_double(IndividualRegistration, registered_at: at.in_time_zone, refused?: refused, exit_reason: exit)

  it "24 meses contados do dia; vence logo = até 60 dias antes" do
    expect(described_class.status(reg(Date.new(2024, 10, 23)), today: today)).to eq("expired")
    expect(described_class.status(reg(Date.new(2024, 10, 24)), today: today)).to eq("due_soon")
    expect(described_class.status(reg(Date.new(2024, 12, 22)), today: today)).to eq("due_soon")
    expect(described_class.status(reg(Date.new(2024, 12, 23)), today: today)).to eq("valid")
    expect(described_class.status(reg(Date.new(2026, 1, 1), refused: true), today: today)).to eq("refused")
    expect(described_class.status(reg(Date.new(2026, 1, 1), exit: "moved"), today: today)).to eq("exited")
  end
end
```

```ruby
# spec/requests/territory_registry/registry_status_spec.rb
require "rails_helper"

RSpec.describe "Território — validade e quem falta cadastrar", type: :request do
  include ActiveSupport::Testing::TimeHelpers
  before { registry_city! }

  let(:unit) { care_unit! }
  let(:doctor) { care_doctor!(unit) }
  let(:team) { care_team!(unit, doctor) }
  let(:nurse) { registry_nurse!(unit, team: team) }

  it "vencidos, a vencer e sem cadastro, com contagem sem trilha e lista com trilha por pessoa" do
    missing = care_patient!(age: 50, sex: "male", problems: [ %w[ciap2 T90] ], unit: unit, doctor: doctor)
    expired = travel_to(25.months.ago) { registry_registration!(household: registry_household!(micro_area: registry_micro_area!(code: "01", team: team))) }
    soon = travel_to((24.months - 30.days).ago) { registry_registration!(household: registry_household!(micro_area: MicroArea.find_by!(team: team, code: "01"))) }
    sign_in_as(nurse)
    before_count = DomainEvent.count
    get "/territory/registry_status/count", params: { team_id: team.id }
    expect(json_body).to eq("valid" => 0, "due_soon" => 1, "expired" => 1, "missing" => 1)
    expect(DomainEvent.count).to eq(before_count)
    get "/territory/registry_status", params: { team_id: team.id, status: "missing" }
    expect(json_body["people"].map { |p| p["patient_id"] }).to eq([ missing.id ])
    expect(json_body["people"].first).to include("status" => "missing", "registered_at" => nil, "micro_area_code" => nil)
    get "/territory/registry_status", params: { team_id: team.id, status: "expired" }
    expect(json_body["people"].map { |p| p["registration_id"] }).to eq([ expired.id ])
    expect(DomainEvent.where(name: "clinical_record.viewed").last.payload["access"]).to eq("territory_registry")
    expect(soon).to be_present
  end

  it "óbito pelo cadastro sai da população; paginação com next_before; equipe alheia → 403" do
    area = registry_micro_area!(code: "01", team: team)
    3.times { registry_registration!(household: registry_household!(micro_area: area)) }
    sign_in_as(nurse)
    get "/territory/registry_status", params: { team_id: team.id, limit: 2 }
    first = json_body
    expect(first["people"].size).to eq(2)
    get "/territory/registry_status", params: { team_id: team.id, limit: 2, before: first["next_before"] }
    expect(json_body["people"].size).to eq(1)
    expect(json_body["next_before"]).to be_nil
    get "/territory/registry_status", params: { team_id: registry_team!(care_unit!).id }
    expect(response).to have_http_status(:forbidden)
  end
end
```

- [ ] **Step 2: Rodar** → FAIL.
- [ ] **Step 3: Implementar**; trocar o status provisório do JSON de cadastro (Task 10) por `Validity.status`.
- [ ] **Step 4: Rodar** — os dois + `spec/services/care_lines/team_population_spec.rb spec/services/care_lines/preview_spec.rb spec/requests/territory_registry` → PASS.
- [ ] **Step 5: Commit** — `feat: list expired, due and missing registrations per team and micro-area`.

---

### Task 13: Exclusão LGPD — cadastro retido

**Files:**
- Modify: `app/commands/citizens/request_erasure.rb`
- Test: `spec/commands/citizens/territory_registry_erasure_spec.rb`

**Interfaces:**
- Consumes: `IndividualRegistrationVersion`, `Family` (Task 2).
- Produces: `Citizens::RequestErasure.examined_patients?(ids)` também verdadeiro com `IndividualRegistrationVersion.where(patient_id: ids).exists? || Family.where(responsible_patient_id: ids).exists?` (Desvio 2; comentário "ADR 0042").

- [ ] **Step 1: Teste**

```ruby
# spec/commands/citizens/territory_registry_erasure_spec.rb
require "rails_helper"

RSpec.describe "Exclusão LGPD com cadastro territorial (ADR 0042)" do
  before { registry_city! }

  it "pedido de quem tem cadastro nasce retido; a confirmação também retém" do
    patient, citizen = exam_patient!(21, age: 40, sex: "female")
    registry_registration!(patient: patient)
    result = Citizens::RequestErasure.call(cpf: citizen.cpf, document_checked: true, by: verifier!)
    expect(result.payload[:request].status).to eq("retained")
  end

  it "responsável de família sem cadastro individual também retém; sem nada, a exclusão segue" do
    patient, citizen = exam_patient!(22, age: 40, sex: "female")
    registry_family!(registry_household!, responsible: patient)
    expect(Citizens::RequestErasure.call(cpf: citizen.cpf, document_checked: true, by: verifier!).payload[:request].status).to eq("retained")
    _free, other = exam_patient!(23, age: 40, sex: "female")
    expect(Citizens::RequestErasure.call(cpf: other.cpf, document_checked: true, by: verifier!).payload[:request].status).to eq("pending")
  end
end
```

- [ ] **Step 2: Rodar** → FAIL. **Step 3: Implementar.** **Step 4: Rodar** — o spec + `spec/commands/citizens/erase_spec.rb spec/commands/citizens/request_erasure_spec.rb spec/commands/citizens/care_lines_erasure_spec.rb` → PASS.
- [ ] **Step 5: Commit** — `feat: retain territory registrations on LGPD erasure`.

---

### Task 14: Semente de dev realista (Curitiba) para a prova no navegador

**Files:**
- Create: `lib/territory_registry_crew.rb`, `spec/lib/territory_registry_crew_spec.rb`
- Modify: `db/seeds.rb` (bloco depois do `CareLinesCrew`)

**Interfaces:**
- Consumes: `CareLinesCrew::UNIT`, `CareLinesCrew::TEAM_NAME`, `CareLinesCrew::PEOPLE`, `CareLinesCrew.team!` (a mesma equipe), comandos das Tasks 6, 9 e 10 (caminho real; só a versão "vencida" é gravada direto com `created_at` no passado, como `lib/campaign_history.rb`).
- Produces: `TerritoryRegistryCrew.seed_current_city(slug:, ddd:, password:) -> { switch:, team:, micro_areas:, households:, registrations:, users: [{ email:, role:, otpauth_uri: }] }` — só Curitiba; `guard_environment!` (nunca produção/staging); idempotente; liga `territory_registry` pelo mantenedor `dev@local`; cria a ACS `acs@curitiba.demo` (Rosana Aparecida Moreira, 515105, sem conselho, CNS com dígito válido, OTP fixo por `DEV_ACS_OTP_SECRET` com padrão `ACSCADASTROTERRITORIALSEMENTE234`); microáreas `01` e `02` (Valores fixados, item 5); 4 domicílios realistas (Rua das Acácias, Rua Ipê Amarelo… com CEP de Curitiba `81…`, bairros do `TerritoryCrew`), famílias com renda e membros coerentes; cadastros: `jose`, `maria`, `osvaldo`, `sebastiao` do `CareLinesCrew::PEOPLE` (válidos), um vencido há 25 meses (`sebastiao`), um a vencer em 40 dias, um recém-nascido sem CPF com CNS válido (justificativa 3, mãe = `ana`), uma pessoa em situação de rua sem CPF e sem CNS (justificativa 5, sem domicílio, `team_id`) — a "não gerada" da prova —, uma recusa com termo, e `antonio` sem cadastro (aparece em "falta cadastrar"); telefones `+55 <ddd> 96666-NNNN`.

- [ ] **Step 1: Teste** (`spec/lib/territory_registry_crew_spec.rb`, padrão do `care_lines_crew_spec.rb`): roda duas vezes e conta o mesmo; o recém-nascido tem `cpf` nulo e CNS com dígito válido; a contagem de `registry_status/count` da equipe bate `{ valid: 4, due_soon: 1, expired: 1, missing: ≥ 1 }`; a pessoa em situação de rua tem `LediGenerationFailure` `patient_without_identifier`; nenhum telefone da semente fora do prefixo `96666`; recusa fora de Curitiba (`switch` "desligado").
- [ ] **Step 2: Rodar** — `bin/test api -w mod24a spec/lib/territory_registry_crew_spec.rb` → FAIL. **Step 3: Implementar** (e o bloco do `db/seeds.rb` com o `puts` no padrão do 23). **Step 4: Rodar** — o spec + `spec/lib/care_lines_crew_spec.rb spec/lib/dev_maintainer_spec.rb` → PASS. **O banco de dev NÃO é migrado nem semeado aqui** (só na prova, com autorização).
- [ ] **Step 5: Commit** — `chore: seed a realistic territory registry in Curitiba for the browser proof`.

---

### Task 15: Revisão, runbook e suíte completa

**Files:**
- Create (worktree `docs-mod24a`, não publicado): `/Users/eduardovrocha/Development/ioit.solutions/rota-saude/.worktrees/docs-mod24a/operacao/rollout-cadastro-territorial.md`

- [ ] **Step 1: Revisão do branch** contra a spec e o contrato (superpowers:requesting-code-review, um revisor no modelo mais capaz): Review Focus 1–5; inventário (cada linha com teste ou justificativa de "inalcançável"); `git -C apps/api/.claude/mod24a grep -n "DomainEvents.publish(" -- app` conferido chamada inteira (payload só com ids); nenhum `Current.city =`; `R18`/`domain_events_bindings`/`city_encrypted_attributes_guard`/`adr_pointers` verdes.

- [ ] **Step 2: Runbook** `operacao/rollout-cadastro-territorial.md` (no padrão dos `rollout-*.md` existentes): pré-requisitos (cidade em modo `record`, `clinical_record` e **`ledi_export` ligados antes** — Desvio 10; `CityProfile.ibge_code`; equipes 70/76 importadas do CNES; ACS com perfil sem conselho e CNS); ordem (imagem do api → `bin/migrate` da imagem, **migração de cidade irreversível**: `patients.cpf` opcional e conselho opcional; `bin/rails patients:backfill_match_keys` por cidade; dashboard; maintenance; ligar `territory_registry` por cidade pelo maintenance); conferências pós-deploy (contagem de pacientes sem `match_key` = 0; triggers presentes: `SELECT tgname FROM pg_trigger WHERE tgname LIKE '%registration%' OR tgname LIKE '%household%'`; primeira ficha tipo 2/3 aceita no PEC de homologação); o que muda para os outros módulos (equipe das linhas de cuidado pelo cadastro; endereço do faturamento só leitura para quem tem cadastro; BPA-I com CNS); LGPD (cadastro retém; paciente sem CPF não tem pedido de exclusão — fluxo no posto é por CPF); reversão (desligar o interruptor; nada se apaga; a migração não volta); gates de go-live (prova no PEC real das fichas 2 e 3; SIAPS aceitando cadastro sem CPF com CNS).

- [ ] **Step 3: Escopo fechado verde** — `bin/test api -w mod24a spec/requests/territory_registry spec/services/territory_registry spec/commands/territory_registry spec/services/ledi spec/services/care_lines spec/commands/citizens spec/commands/patients spec/commands/professionals spec/services/billing spec/models spec/lib/territory_registry_crew_spec.rb spec/architecture spec/initializers spec/events spec/adr_pointers_spec.rb` → PASS.

- [ ] **Step 4: Pedir a suíte completa** à sessão "Testes" (mudança transversal: migração de cidade, `patients`, `spec/support`): SendMessage `api · .claude/mod24a/feat/mod-24a-territory-registry · --full · migração de cidade 20261023900001 (patients.cpf opcional, professionals sem conselho) e módulo 24a`. Antes, `docker compose stop worker` (memória "Suíte com worker parado") e `start` depois. Falha → diagnóstico nesta sessão (a "Testes" só responde o resumo).

- [ ] **Step 5: Commit final** (se a revisão mudou algo) — `fix: address module 24a review findings`; avisar a coordenadora: plano do api concluído, runbook no `docs-mod24a` (não publicado), §14 do contrato pendente de publicação, cards F-24.x seguem In Progress até o dashboard e o maintenance.

---

## Self-review

- **Cobertura da spec:** §2 decisões 1–2 (Tasks 4, 10), 3 (6), 4 (9), 5–6 (10), 7 (12), 8 (3), 9 (6, 10), 10 (1), 11 (2, 13), 12 (nada no wpda), 13 (10); decisões do coordenador: versões imutáveis (2, 8), uma fonte por dado (9, 10), equipe e óbito (11), duplicados (4), CPF só no corpo (Global Constraints, 10), Task 0 prova técnica (0); revisão: endereço (9), saída por mudança (11), sem rascunho (nenhuma rota de rascunho; nada no servidor), família e versões (9), "enviado" (13, Desvio 2), quem falta (12), cabeçalho (8, Desvio 4), sem CPF nos outros envios (5). §6 segurança (Global Constraints, 2, 10, 11). §7 testes (todas). §8 runbook (15). F-24.1..8: 1+6, 3, 4+5+10, 9, 10, 12, 8, 11.
- **Tipos:** `Patients::Pairs.of`, `Patients::AddCpf.call(patient:, cpf:, by:)`, `TerritoryRegistry::Access.registrar?`, `Ledi::RegistrationFicha.generate(version, city:)`, `TerritoryRegistry::Teams.for_patients`, `TeamPopulation.members(team:)` usados com a mesma assinatura em todas as tasks.
- **Proporção:** corpos só onde a regra não se deduz da assinatura e do teste (YAML da Task 0, triggers listados por regra).

## Divergências propostas ao contrato

- **R1 — Numeração do §11.** O contrato já tem §11 (Maintenance) a §13; a tabela dos dicionários que a Task 0 produz entra como **§14 "Dicionários das fichas"** (e o §1 passa a citar "§14"). Inclui os achados: `stAtivo` ausente do IDL 8.7.0 e o "sem CPF" do BPA-I.
- **R2 — Cadastro sem domicílio.** `POST/PATCH /territory/registrations` aceitam `team_id` (obrigatório quando não há `household_id`); a ficha vai com `stForaArea = true`. `<registration>` ganha `team: { id, ine, name }` e `micro_area: { id, code }|null`.
- **R3 — Endereço do domicílio.** `address` ganha `street_type` (código DNE, obrigatório), `no_number` (bool), `district` (texto, obrigatório quando `neighborhood_id` é nulo). Limites: o menor entre LEDI e BPA (`street` ≤ 30, `number` ≤ 5, `complement` ≤ 10, `district` ≤ 30, `reference` ≤ 40); mais longo → 422 `invalid_household` com `path`.
- **R4 — Situação do domicílio.** `PATCH /territory/households/:id` aceita `status` (`active|inactive`); `inactive` só sem famílias ativas (`path: "status"`).
- **R5 — Identificação da ficha individual.** O corpo do cadastro ganha `identification` (campos de `IdentificacaoUsuarioCidadaoThrift` que não moram no paciente: nacionalidade, município/país de nascimento, naturalização/entrada, nome do pai ou "desconhece", celular, e-mail, NIS, estado civil, tipo sanguíneo, DNV, "desconhece nome da mãe"); `race_color` e `ethnicity` vão em `new_patient` (ou na raiz, para paciente existente) e são gravados **no paciente** (códigos do BPA, convertidos na ficha). `<registration>` devolve `identification`.
- **R6 — Endereço só leitura (contrato §10).** Recepção e faturamento recebem 422 `address_from_household` (`field`) ao gravar endereço de paciente com cadastro ativo em domicílio; `flags` ganha `address_from_household: bool` (a tela mostra "do cadastro domiciliar").
- **R7 — Eventos de trilha.** Além do §12: `territory_household.viewed` (`household_id`, `user_id`) e `territory_households.listed` (`micro_area_id`, `household_ids`, `user_id`); leitura nominal de pessoa usa `clinical_record.viewed` com `access: "territory_registry"`. `patient.created` (sem `citizen_id`) na criação direta.
- **R8 — Códigos novos.** 409 `cns_taken` (CNS de outro paciente), 409 `citizen_not_verified` (`citizen_id` de par não validado), 409 `invalid_transition` (saída repetida), 422 `invalid_request` (dado pessoal na query de `duplicates`), 422 `invalid_limit`; motivos de "não gerada" `patient_without_identifier`, `responsible_without_identifier`, `cbo_not_in_ledi_table`. BPA-I sem CPF: `Prd-cnspac` com o CNS, `prd_cpf_pcnte` em brancos, `prd_sem_cpf = S` (a confirmar na Task 0; muda `contracts/billing/bpa-interface.md` linhas 37, 98, 126, 128).
- **R9 — Cursor da validade.** `next_before` de `/territory/registry_status` é o `person_ref` da última linha (`patient:<uuid>`/`citizen:<uuid>`), ordem decrescente; a lista não vem por data de cadastro.
- **R10 — Retenção LGPD.** "Enviado" vira "qualquer versão gravada ou responsável de família" (Desvio 2): superconjunto do enfileirado, sem dado que fique com o pedido dizendo "excluído".
