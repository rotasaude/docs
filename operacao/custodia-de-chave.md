# Custódia de chave — AR Encryption e credentials

Onde vive cada chave que cifra dado de cidadão e de servidor, quem acessa, como
rotaciona. O inventário completo de segredos do deploy fica em
`apps/api/deploy/SECRETS.md`; este runbook trata só das chaves de cifra.

## Quando aplicar

- Antes do primeiro deploy de produção (criar as credentials próprias).
- Ao suspeitar de vazamento de dump de uma cidade, ou por rotina de rotação.
- Ao restaurar o dump de uma cidade.

## Pré-requisitos

- Acesso ao cofre `rota-saude-prod` (ou `rota-saude-staging`) no 1Password.
- Shell no container do `worker` do ambiente (as rakes de cidade rodam lá).

## Onde cada chave vive

| Chave | Onde | Quem lê |
|---|---|---|
| `RAILS_MASTER_KEY` | cofre, item `rails-master-key`; nunca no git | boot do `api` (web e worker) |
| `config/credentials/<env>.yml.enc` | git (cifrado pela master key do ambiente) | boot |
| `ACTIVE_RECORD_ENCRYPTION_PRIMARY_KEY`, `_DETERMINISTIC_KEY`, `_KEY_DERIVATION_SALT` (chave da plataforma) | cofre, item `active-record-encryption`, ou dentro das credentials do ambiente | `lib/encryption_keys.rb` |
| `cities.encryption_key` (material de cada cidade) | banco de plataforma, cifrado pela chave da plataforma | `CityEncryption` |
| `report_signing_key` | credentials do ambiente | `CityEncryption.report_signing_key` |

Ordem de leitura da chave da plataforma (`lib/encryption_keys.rb`): credentials
do ambiente, depois `ACTIVE_RECORD_ENCRYPTION_*`, depois os nomes legados
`AR_ENCRYPTION_*`. Em ambiente publicado (staging, produção), faltar qualquer
uma das três derruba o boot com `EncryptionKeys::Missing`.

Staging e produção só sobem com o próprio `config/credentials/<env>.yml.enc`
(`Rota::ISOLATED_CREDENTIALS_ENVS`). Lendo o arquivo compartilhado de dev, o
boot falha com `Rota::SharedCredentials`.

A chave efetiva de cada cidade é **derivada** de (chave da plataforma +
`cities.encryption_key`). Um dump de uma cidade não abre outra; quem tem a
chave da plataforma deriva todas.

## Passos

### 1. Credentials próprias de produção (uma vez, antes do go-live)

1. `bin/rails credentials:edit --environment production` — cria
   `config/credentials/production.yml.enc` (commitar) e
   `config/credentials/production.key` (nunca commitar).
2. Preencha `secret_key_base` e `report_signing_key`. As chaves do AR
   Encryption podem ficar só no cofre; se entrarem nas credentials, vencem as
   do cofre.
3. Guarde `production.key` no cofre `rota-saude-prod`, item `rails-master-key`,
   e apague a cópia local.
4. Troque o secret `RAILS_MASTER_KEY` do GitHub (job `production-boot`) pelo
   mesmo valor.

### 2. Rotacionar o material de uma cidade

1. `city:backup[slug]`
2. `city:suspend[slug]` e aguarde o período de silêncio
   (`CityLifecycle::SuspensionGuard::QUIET_PERIOD`).
3. `city:rotate_key[slug]` — gera `cities.encryption_key` novo e reescreve os
   atributos de `CityEncryption::CITY_KEYED_TARGETS`.
4. `city:resign_reports[slug]` — reassina os links de relatório.
5. `city:resume[slug]`

### 3. Restaurar o dump de uma cidade

Precisa de dois materiais **da época do dump**: a chave da plataforma e o
`cities.encryption_key` daquela cidade. Sem os dois, o dado não decifra e as
assinaturas de relatório não conferem.

## Validação

- Boot de produção sem `production.yml.enc` falha com `Rota::SharedCredentials`
  (`spec/lib/rota_spec.rb`).
- Chave faltando em ambiente publicado falha com `EncryptionKeys::Missing`
  (`spec/lib/encryption_keys_spec.rb`).
- Uma linha da cidade A não decifra com o material da cidade B
  (`spec/models/city_connection_encryption_spec.rb`).

## Rollback

Se a reescrita falhar no meio, `city:rotate_key` devolve o
`cities.encryption_key` anterior ao catálogo. Depois de concluída, voltar
exige o dump do passo 2.1 **e** o `cities.encryption_key` anterior, que a
rotação sobrescreve: o `city:backup` grava só o SHA-256 dele (para conferir),
não a chave. Sem um backup do banco de plataforma da mesma hora, não há volta.

## Em aberto

- **Rotação da chave da plataforma:** não há procedimento. Trocá-la exige
  reescrever o `cities.encryption_key` de todas as cidades e todo dado de cada
  cidade.
- **Acesso:** quem tem acesso aos itens do cofre não está registrado aqui.

## ADRs relacionados

0013, 0020.
