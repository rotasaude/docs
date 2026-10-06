# PEC local para a prova técnica do exportador LEDI (módulo 16)

Como subir um PEC e-SUS APS 5.5.28 em Docker, só para provar a entrega Thrift
do exportador LEDI 8.7.0. Observado em 2026-10-06. Tudo aqui é de
**desenvolvimento**: nada disso toca banco de cidade nem dado real.

Fontes oficiais:

- https://integracao.esusaps.bridge.ufsc.tech/ledi/documentacao/api_transmissao/api_transmissao_registro_LEDI.html
- https://sisaps.saude.gov.br/sistemas/esusaps/docs/manual/APOIO/API_transmissao/
- https://sisaps.saude.gov.br/sistemas/esusaps/docs/manual/APOIO/Certificado_Https_Linux/
- https://github.com/laboratoriobridge/esusaps-integracao (IDLs Thrift)

## Pré-requisitos

- Docker com **pelo menos 4 GB** de memória para o PEC.
- Em Mac arm64 o PEC roda **emulado em amd64**; a instalação levou ~10 min.
- `openssl` (a LibreSSL 3.3.6 do macOS serviu).

## Instalador

| Item | Valor |
| --- | --- |
| Arquivo | `eSUS-AB-PEC-5.5.28-Linux64.jar` |
| URL | https://arquivos.esusaps.ufsc.br/PEC/5ad5d7ee6e32a785/5.5.28/eSUS-AB-PEC-5.5.28-Linux64.jar |
| Tamanho | 779582762 bytes |
| sha256 | `82982c9bd28d587d54601022ebade47bde9250f210dc946e8c76b2c182c9cdc1` |
| Página que lista | https://sisaps.saude.gov.br/sistemas/esusaps/ (a página antiga `/esus/` está desatualizada e mostra 5.3.26) |

Baixe para `deploy/development/pec/local/` (fora do git) e confira o sha256.

## Passos executados

Tudo a partir de `apps/api/.claude/mod16-exporter/deploy/development/pec`.

1. **Arquivo de ambiente** `local/pec.env` com `PEC_JAR` (nome do `.jar`) e
   `PEC_KEYSTORE_PASSWORD` (senha aleatória). Nunca vai ao git.
2. **CA e certificado** em `local/` (comandos do Step 3 do plano):

   ```bash
   openssl req -x509 -newkey rsa:2048 -nodes -days 365 -subj "/CN=Rota Saude Dev CA" -keyout ca.key -out ca.pem
   openssl req -newkey rsa:2048 -nodes -subj "/CN=pec.rota.test" -keyout pec.key -out pec.csr
   printf "subjectAltName=DNS:pec.rota.test,DNS:localhost\n" > san.ext
   openssl x509 -req -in pec.csr -CA ca.pem -CAkey ca.key -CAcreateserial -days 365 -extfile san.ext -out pec.crt
   openssl pkcs12 -export -in pec.crt -inkey pec.key -name esusaps -out esusaps.p12 -passout "pass:$PEC_KEYSTORE_PASSWORD"
   ```

   Adicione `127.0.0.1 pec.rota.test` ao `/etc/hosts`.
3. **Subir:** `docker compose --env-file local/pec.env up -d --build`.
   Acompanhe com `docker compose logs -f pec` e espere
   `https://localhost:8443` responder (polling com `curl -k`). A primeira
   subida instala o PEC em modo treinamento.
4. **Três correções de contêiner** descobertas na instalação (já no repositório):
   - `WORKDIR /opt` no Dockerfile: o instalador converte o cwd `/` em `""` e
     falha com `Cannot run program /bin/sh (in directory "")`, que aparece
     como "Esta ferramenta necessita executar com privilégios de
     administrador" (mensagem enganosa).
   - `pec-db` com autenticação **md5**: o driver JDBC do instalador rejeita
     SCRAM ("O tipo de autenticação 10 não é suportado").
   - A **CA local importada no cacerts da JRE do PEC** (`keytool -importcert
     -cacerts`, no entrypoint): o PEC valida o próprio link HTTPS e, sem a CA,
     falha com PKIX.
5. **Configuração no navegador** (`https://localhost:8443`, aceitando a CA local):
   - Instalação de **treinamento** do tipo **CENTRALIZADORA**, nome
     "Rota Saúde dev – Maringá", com link `https://pec.rota.test:8443`.
   - Município **Maringá (4115200)**, o `city_profile.ibge_code` de `maringa` na semente de dev.
   - **Transmissão de dados → Credenciais para API**: credencial de pessoa
     jurídica ("Rota Saúde dev", CNPJ fictício 11.222.333/0001-81). Usuário e
     senha ficam só no ambiente de quem roda a prova.

## Limite: CNES não importado

O XML do CNES para o e-SUS APS só sai da **área restrita do e-Gestor APS**, e a
lista de municípios da tela de importação do PEC veio **vazia**. Sem CNES
importado o PEC recusa qualquer ficha com CNES/INE/profissional "inválido".
Consequência: **nenhum 2xx foi observado**, nem a recusa de duplicidade após
aceite. O exemplo `:pec` "aceita a ficha sintética" de
`spec/integration/ledi_pec_spec.rb` **fica vermelho neste ambiente**, de
propósito; ele passa quando houver um PEC com CNES importado.

## Geração das classes Thrift

```bash
bin/ledi-generate 8.7.0 9316f280f5
```

- Commit `9316f280f5` do esusaps-integracao: IDLs **idênticos** aos do commit
  `a9e5a203` e coerentes com o dicionário LEDI 8.7.0.
- Compilador `thrift` 0.17.0 (Debian bookworm) contra a gem `thrift` 0.25.0:
  compatíveis, provado pelo spec de ida e volta e pelo PEC desserializando a
  ficha.

## Comando da prova

Variáveis (só os **nomes**; os valores nunca entram em arquivo versionado):
`LEDI_PEC_URL=https://pec.rota.test:8443`,
`LEDI_PEC_CA_FILE=<caminho do local/ca.pem no contêiner>`,
`LEDI_PEC_USERNAME`, `LEDI_PEC_PASSWORD`, `LEDI_PROOF_CNES`, `LEDI_PROOF_INE`,
`LEDI_PROOF_CNS`, `LEDI_PROOF_CBO`, `LEDI_PROOF_IBGE=4115200`.
Rode o Step 12 do plano com essas variáveis no `docker compose exec`.

## Onde ver a ficha no PEC

Não foi observável: sem um 2xx nenhuma ficha é gravada. Quando houver aceite,
registre aqui a tela em que ela aparece.

## Respostas observadas

Espelho de `config/ledi/pec_observations.yml` no api (PEC 5.5.28, LEDI 8.7.0).

| Situação | Resposta |
| --- | --- |
| Login com formulário (`/api/recebimento/login`, campos `usuario`, `senha`) | 200 + cookie de sessão |
| Login com corpo JSON ou credencial errada | 400, texto "Usuário ou senha inválidos." |
| `/api/v1/recebimento/login` | 403 (CSRF) |
| Entrega em `POST /api/v1/recebimento/ficha` (multipart, campo `ficha`, cookie) | caminho correto; `/api/recebimento/ficha` dá 403 |
| Arquivo que não é Thrift | 400 JSON `descricaoErro` com "Erro na desserialização" |
| Ficha sintética com CNES/INE/CNS fictícios | 400 JSON `Erro de validação` com `errosValidacao` (`cnesDadoSerializado`, `ineDadoSerializado`, `headerTransport`): o PEC desserializou transporte e ficha e rodou a validação de negócio |
| Transporte válido, `dadoSerializado` lixo | 400, `errosValidacao.dadoSerializado` = erro de desserialização da Ficha |
| Compressão | nenhuma (TBinaryProtocol cru) |
| Cookie de sessão inválido | 401 |
| Mesmo uuid reenviado após 400 | revalidado (mesmo 400), não recusado como duplicado |
| 2xx de sucesso / duplicidade após aceite | **não observados** (exigem CNES importado) |

## O que nunca entra no git

- O instalador `.jar` e tudo em `deploy/development/pec/local/`.
- Chaves e certificados: `ca.key`, `ca.pem`, `pec.key`, `pec.crt`, `esusaps.p12`.
- `local/pec.env` e a senha do keystore.
- Usuário e senha da credencial de API, cookies de sessão.
- CNES, INE, CNS e CBO reais, e qualquer CPF/CNS/nome de pessoa em exemplos de resposta.
