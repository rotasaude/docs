# Rollout — cadastro individual e domiciliar (módulo 24a, território)

Deploy, ligação por cidade e limites conhecidos do cadastro territorial (api; dashboard e maintenance seguem o api; o
wpda não muda). Contrato: `superpowers/plans/2026-10-23-module-24a-territory-registry-contracts.md` (§14 e §15).
Interruptor: `territory_registry` ("Cadastro individual e domiciliar").

## Pré-requisitos

- Cidade em modo registro (`record_mode = record`) com `clinical_record` **e** `ledi_export` ligados **antes** de
  `territory_registry`. Sem `ledi_export` a versão grava e nenhuma ficha nasce, e **não há varredura retroativa**:
  cadastro feito nesse intervalo não gera ficha depois.
- `CityProfile.uf` e `CityProfile.ibge_code` preenchidos (L20). O maintenance mostra `uf_missing` /
  `ibge_code_missing` como faltas do interruptor. Sem eles a versão grava, mas a ficha fica "não gerada" com motivo
  `city_profile_incomplete` (o PEC recusaria a domiciliar sem `numeroDneUf`); ao completar o perfil, as fichas saem no
  **reenvio**. Conferir staging e produção.
- Equipes (INE de 10 dígitos, tipo 70/76) importadas do CNES, com as unidades (módulo 16, retrato do CNES).
- ACS (CBO 515105) e agente de combate às endemias (515140) com perfil **sem conselho** e **com CNS** (o CNS segue
  obrigatório; só o conselho foi dispensado). Demais CBOs não mudam.
- Nenhuma gem nova.

## Ordem de deploy

Migração de cidade **irreversível** (o `down` levanta `IrreversibleMigration`): não há rollback de schema, só do código
(expand/contract). Publicar a imagem e rodar o `bin/migrate` **dela**, antes de cortar o tráfego. Nunca migrar cidade
fora do rake.

1. **api**: publicar a imagem; rodar o `bin/migrate` da imagem (`db:migrate`, `platform:triggers`, `city:migrate:all`).
   A migração de cidade `20261023900001`:
   - torna `patients.cpf` opcional e o conselho do profissional opcional (CBO 515105/515140);
   - cria as tabelas do cadastro (microáreas, domicílios, famílias, cadastros individuais e as versões imutáveis);
   - cria `patients.match_key` (chave cega de duplicados) e seu índice, mais o índice de óbito por paciente;
   - cria os triggers de guarda (`db/city_triggers.sql`).

   ```bash
   bin/migrate
   ```

2. **Backfill das chaves de duplicados, por cidade (OBRIGATÓRIO).** A `match_key` nasce nula; paciente antigo sem chave
   não é achado como duplicado e não ganha chave em save alheio. Idempotente; sem argumento percorre toda cidade
   ativa/suspensa; uma falha aborta no fim listando as cidades.

   ```bash
   bin/rails patients:backfill_match_keys          # todas
   bin/rails 'patients:backfill_match_keys[<slug>]' # uma cidade
   ```

3. **dashboard** (depois do api).
4. **maintenance** (depois do dashboard): rótulo do interruptor e do módulo no catálogo.
5. **Só depois**, o interruptor `territory_registry`, por cidade, no maintenance.

## Conferências pós-deploy

- Pacientes sem chave = 0, em cada cidade:

  ```sql
  SELECT count(*) FROM patients WHERE match_key IS NULL;
  ```

- Triggers presentes em cada banco de cidade (esperados: `household_versions_reencryption_only`,
  `individual_registration_versions_reencryption_only`, `individual_registrations_guard`, `micro_areas_guard`,
  `families_guard`):

  ```sql
  SELECT tgname FROM pg_trigger
   WHERE tgname LIKE '%registration%' OR tgname LIKE '%household%'
      OR tgname LIKE '%micro_area%' OR tgname LIKE '%famil%';
  ```

- Primeira ficha de cadastro individual (tipo 2) e primeira de cadastro domiciliar (tipo 3) **aceitas pelo PEC de
  homologação**.
- No maintenance, `territory_registry` sem `uf_missing`/`ibge_code_missing`. Em dev, ligar `ledi_export` **antes** do
  `db:seed`: a ficha "não gerada" da pessoa em situação de rua só aparece assim.

## O que muda para os outros módulos

- **Linhas de cuidado (23)**: a equipe da pessoa passa a vir do cadastro individual ativo (equipe da microárea do
  domicílio); sem cadastro, vale a regra de antes (inscrição, consulta recente com equipe, bairro). **Inscrição ativa
  numa linha vence o cadastro naquela mesma linha** (L22); o cadastro vale para o resto. Cadastro recusado também
  define a equipe. Óbito registrado na saída do cadastro conta como óbito mesmo com o interruptor desligado depois.
- **Faturamento e recepção (22/17)**: paciente com cadastro ativo em domicílio tem endereço **só leitura**
  ("do cadastro domiciliar"); gravar endereço devolve 422 `address_from_household`.
- **BPA-I sem CPF**: vai com o CNS em `Prd-cnspac` (colunas 60–74), CPF em branco e `prd_sem_cpf = S`. O campo
  `prd_situacao_rua` (350) **continua em branco** (pendência do faturamento, L18).
- **Paciente sem CPF** aparece como "sem CPF" com o CNS (4 últimos dígitos); recepção, prontuário e linhas de cuidado
  não quebram com `cpf` nulo. Sem CPF **e** sem CNS o cadastro grava, mas a ficha fica "não gerada"
  (`patient_without_identifier`) e a pessoa aparece como `missing_identifier` na validade, para a equipe obter o CNS.
- **Rotação de chave** (`city:rotate_key`): cobre as colunas cifradas novas e a `match_key`/índice de duplicados
  (cifra determinística, como o CPF).
- **Campanhas, agenda, analytics**: sem mudança.

## Motivos de ficha "não gerada" (comunicar às equipes)

`city_profile_incomplete` (UF/IBGE da cidade), `patient_without_identifier`, `responsible_without_identifier`,
`cbo_not_in_ledi_table`, mais os de profissional/unidade sem CNS/CNES. Corrigido o dado, a ficha sai no reenvio.

## LGPD

- O cadastro **retém** o pedido de exclusão: retém quem tem qualquer versão de cadastro gravada **ou** é responsável de
  família.
- **Paciente sem CPF não tem pedido de exclusão pelo celular** (o wpda é por CPF + OTP). O fluxo é presencial, no
  posto, por CPF (`Patients::AddCpf` leva o CPF ao paciente e então o pedido segue o fluxo normal).
- CPF, CNS, nome, nascimento, endereço, condições de saúde e situação de rua nunca vão para log, evento ou Analytics;
  os eventos do módulo levam só ids. Listas e detalhe de domicílio gravam `clinical_record.viewed` com
  `access: territory_registry`.

## Desligar

- Desligar o interruptor `territory_registry` (maintenance): as rotas `/territory/registrations`, `/territory/households`
  e `/territory/registry_status` passam a responder 403 `feature_disabled`; nenhuma ficha nova nasce. **Nada se apaga**:
  cadastros, versões e fichas ficam.
- A migração **não volta**: sem rollback de schema; volte só o código (expand/contract). `patients.cpf` e o conselho
  seguem opcionais no banco.

## Gates de go-live

- **Prova no PEC real** das fichas 2 (individual) e 3 (domiciliar), com `numeroDneUf`/IBGE da cidade.
- **SIAPS** aceitando cadastro **sem CPF**, identificado só por CNS.
- **Nomes nas fichas** seguem o conjunto oficial do "Validar nome" do e-SUS: na ficha, `Ç` vira `C`; o nome no paciente **mantém** o `Ç`. Conferir com um nome real antes de liberar.
