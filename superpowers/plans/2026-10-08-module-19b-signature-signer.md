# Módulo 19b — Serviço `signer` (repo novo `rotasaude/signer`) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

> **PRÉ-REQUISITO: 19a fechado em `origin/main`** (api e dashboard; F-19.1..7 Verified). O `signer` não usa código do 19a, mas o 19b só executa depois dele (spec, "Pré-requisito de execução"); a Task 0 confere isso antes da prova.

**Goal:** Criar o serviço interno `signer` (Java 21 + Demoiselle Signer, repo novo `rotasaude/signer`, local em `apps/signer`) que prepara, monta e valida assinaturas CAdES destacadas e PAdES na política AD-RB, com o hash assinado fora pelo certificado em nuvem do profissional (F-19.15; ADR 0032; contrato §9) — depois de provar com o sandbox do VIDaaS e o validar.iti.gov.br que o caminho funciona.

**Architecture:** O `api` fala com o PSC; o `signer` só trabalha com bytes. `/prepare` monta os atributos assinados da política com `CAdESSigner#prepareSignedAttributes` do Demoiselle (público, sem chave privada) e devolve o SHA-256 deles e um `prepared_state` opaco selado com HMAC; o PSC assina esse hash em `RAW` (PKCS#1 v1.5); `/assemble` confere a RAW contra o certificado e monta o `SignerInfo` com o BouncyCastle (dependência do próprio Demoiselle), por um `ContentSigner` que só devolve o valor já calculado; no PAdES o PDFBox reserva a assinatura no PDF no `/prepare` e o CMS é injetado no `/assemble`. `/verify` usa o checker do Demoiselle e confere política, cadeia, hora, keyUsage e revogação no instante da assinatura. HTTP do próprio JDK, sem framework; nenhum estado entre chamadas; log só de acesso.

**Tech Stack:** Java 21; Maven 3.9 (rodado em container, `bin/mvn`); Demoiselle Signer 4.6.2 (`policy-impl-cades`, `policy-impl-pades`, `chain-icp-brasil`, `chain-icp-brasil-homolog`); BouncyCastle 1.80 (`bcpkix-jdk18on`); Apache PDFBox 3.0.3; Jackson 2.17.2; Log4j 2.25.5; JUnit 5.10.3; `com.sun.net.httpserver`; Docker (`maven:3.9.9-eclipse-temurin-21` → `eclipse-temurin:21-jre`); GitHub Actions.

**Spec:** `docs/superpowers/specs/2026-10-08-module-19b-digital-signature-design.md` (§2 decisão 10, §6, §7, §11, §12); ADR `docs/adr/0032.md`; contrato entre apps (fonte única dos formatos): `docs/superpowers/plans/2026-10-08-module-19b-signature-contracts.md` §9 e §12; pesquisa `docs/pesquisa/2026-10-08-assinatura-digital-19b.md` §2, §4 e §5.

## Decisões deste plano (com trade-offs)

1. **Maven, não Gradle.** O Demoiselle é Maven, o build é declarativo e cabe num `pom.xml` sem plugin próprio; não há necessidade de build incremental rápido num serviço de ~20 classes. O host não tem Maven nem Gradle: `bin/mvn` roda `maven:3.9.9-eclipse-temurin-21` num container (o mesmo da imagem), e a CI usa o Maven do runner. Contra: sem wrapper versionado (a versão vem da imagem fixada no script e no Dockerfile).
2. **Assinatura externa no Demoiselle — o que existe e o que falta (conferido no código-fonte 4.6.2, `github.com/demoiselle/signer`).** `CAdESSigner` tem `prepareDetachedSign`/`envelopDetachedSign` e `doHashSign`, mas **todos exigem `PrivateKey`** (`JcaSimpleSignerInfoGeneratorBuilder.build(..., getPrivateKey(), ...)`): não há assinatura em duas fases com o hash assinado fora. Existe, pública e sem chave, `CAdESSigner#prepareSignedAttributes(byte[] content, byte[] previewSignature)`, que monta os atributos obrigatórios da política (contentType, messageDigest, signaturePolicyIdentifier, signingCertificateV2) a partir só da cadeia. O plano usa essa chamada e fecha a lacuna com o BouncyCastle, que já vem com o Demoiselle. **PAdES:** o `policy-impl-pades` não lê nem escreve PDF (a própria documentação manda usar o Apache PDFBox); o `PAdESSigner` é um `CAdESSigner(..., pades = true)` sobre os bytes cobertos pelo `ByteRange`. Por isso **nem pyHanko nem EU DSS** são necessários: o PDFBox (Apache-2.0) faz só a estrutura do PDF e o Demoiselle continua dono da política. Validação: `CAdESChecker`/`PAdESChecker#checkDetachedSignature`.
3. **Políticas:** as padrão do Demoiselle 4.6.2 — CAdES `AD_RB_CADES_2_4` (OID `2.16.76.1.7.1.1.2.4`) e PAdES `AD_RB_PADES_1_3` (OID `2.16.76.1.7.1.11.1.3`), lidas do `.der` embutido (sem rede). O OID nunca é fixado no código. O `/verify` aceita todas as versões AD-RB do formato (assinatura antiga continua verificável). A checagem de vigência da política usa o `signingPeriod` do próprio `.der`; o `PolicyValidator` do Demoiselle não é usado porque pode baixar a LPA da rede no meio de uma assinatura.
4. **HTTP do JDK** (`com.sun.net.httpserver`, threads virtuais), não um framework: quatro rotas internas, JSON pequeno, nenhuma sessão. Contra: roteamento à mão (um `switch`).
5. **Logs:** só acesso (método, caminho, status, duração). O logger `org.demoiselle` fica **desligado** — o Demoiselle escreve o DN do titular (`CN=NOME:CPF`) em erro e debug; isso foi visto ao escrever o plano (o teste de higiene quebra com o Demoiselle em `debug`).
6. **AC de teste no formato ICP** (`DevPki`, código de produção, nunca ligado em produção): raiz → intermediária → e-CPF com o `otherName` 2.16.76.1.3.1, LCRs com ponto de distribuição HTTP. Os testes a servem num HTTP local; em dev o container a gera num volume e serve as LCRs em `/dev-pki/`, e o `api` monta o mesmo volume para o PSC falso dele emitir e-CPF de teste.

## Global Constraints

- Repositório novo `rotasaude/signer`, local em `/Users/eduardovrocha/Development/ioit.solutions/rota-saude/apps/signer`. `git init` local é livre; **criar o repositório remoto, configurar o `origin` e dar push só com autorização explícita do usuário** (hard stop, Task 6).
- Worktree `apps/signer/.claude/mod19b`, branch `feat/mod-19b-signature` (a partir do primeiro commit da `main` local, que só tem o `.gitignore`).
- Java 21 (`maven.compiler.release` 21). Pacote raiz `rotasaude.signer`. Porta interna **8090**.
- Contrato §9 literal: `Authorization: Bearer <SIGNER_TOKEN>` em toda rota; JSON; `POST /prepare { kind: "cades"|"pades", document_base64, certificate_der_base64, policy: "AD-RB" }` → `{ to_be_signed_sha256_base64, prepared_state }`; `POST /assemble { kind, prepared_state, signature_value_base64 }` → `{ signature_base64, validation_material_base64 }`; `POST /verify { kind, document_base64?, signature_base64 }` → `{ status, signer_cpf, signer_name, policy_oid, signed_at, reasons }`; `GET /health` → `{ version, crl_updated_at }`; erros `400 invalid_request`, `401 unauthorized`, `422 invalid_certificate`, `422 invalid_signature_value`, `500 internal`, sempre `{ "error": "<código>" }`.
- Corpo nunca logado. Token, `code_verifier`, CPF, nome e texto clínico nunca em log, mensagem de erro ou resposta de erro. A mensagem de erro é só o código.
- O `signer` nunca grava documento: nada em disco além do cache de LCR/LPA do Demoiselle em `/tmp`.
- Segredos de sandbox (Task 0) só no arquivo `~/.config/rotasaude/vidaas-sandbox.env` do usuário (`chmod 600`, fora de qualquer repositório), passado ao container com `--env-file`. Nunca inventar credencial, nunca gravar segredo em arquivo versionado, nunca imprimir o arquivo.
- `SIGNER_ENV=production` proíbe `SIGNER_EXTRA_TRUST_ANCHORS` e `SIGNER_DEV_PKI_DIR` (o boot falha) e, pelo próprio Demoiselle, desliga as cadeias de homologação da ICP-Brasil.
- Testes herméticos: sem rede externa. O `pom.xml` desliga as cadeias ICP reais e a LPA online só no Surefire (`SIGNER_DISABLE_CHAIN_ICP_BRASIL`, `SIGNER_DISABLE_CHAIN_ICP_BRASIL_HOMOLOG`, `SIGNER_REPOSITORY_LPA_ONLINE=false`, `SIGNER_CRL_CONNECTION_TIMEOUT=2000`).
- `docker-compose.yml` da raiz do monorepo **não é versionado** (a raiz está fora de git) e é compartilhado: avise a sessão dona do `api` (sessão "API") **antes** de mexer em `x-api-env` e nos serviços `api`/`worker`.
- Git: `/opt/homebrew/bin/git` (o `/usr/bin/git` aborta). Nunca `git add -A`: caminhos explícitos. Commits em inglês, Conventional Commits com tipo por extenso, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`. Merge, push, tag e criação de repositório só com autorização explícita, uma etapa de cada vez.
- Todos os comandos a partir da raiz do monorepo `/Users/eduardovrocha/Development/ioit.solutions/rota-saude`, salvo indicação.

## Review Focus

1. **Nome e CPF do titular no log** — o Demoiselle escreve o DN (`CN=NOME:CPF`) do certificado em vários níveis, inclusive em erro de cadeia; um `log4j2.xml` mexido para "depurar" vaza dado pessoal sem nenhum teste funcional perceber. Teste: `SignerServerTest#logsNeverCarryTokenCpfNameOrDocument` (Task 4), que captura a saída de assinaturas CAdES e PAdES, de uma verificação e de um certificado recusado e exige que não apareçam token, CPF, nome nem texto clínico.
2. **AC de teste ligada em produção** — `SIGNER_EXTRA_TRUST_ANCHORS` ou `SIGNER_DEV_PKI_DIR` esquecidos no ambiente de produção fariam o `signer` confiar numa AC cuja chave está num volume. Testes: `ConfigTest#refusesTestAnchorsAndDevPkiInProduction` (Task 1) e a subida do container com `SIGNER_ENV=production` saindo com código 1 (Task 5, Step 3); e `SignerServerTest#devPkiServesOnlyTheCrlsWithoutToken` (Task 4) garante que a chave da intermediária nunca é servida.
3. **`prepared_state` adulterado ou trocado de tipo** — quem segura o estado entre `/prepare` e `/assemble` é o `api`; um estado alterado (ou um PAdES apresentado como CAdES) não pode virar assinatura. Testes: `PreparedStateCodecTest` (Task 1: byte trocado, outro token, lixo) e `SignerServerTest#contractErrors` (Task 4: `kind` diferente do estado → 400; RAW que não confere → 422 `invalid_signature_value`).
4. **PDF alterado depois de assinado** — atualização incremental acrescentada ao fim (o PDF continua abrindo e a assinatura "parece" válida num leitor) ou byte trocado dentro da faixa assinada. Testes: `PadesRoundTripTest#bytesAppendedAfterSigningAreInvalid` (`pdf_modified_after_signing`) e `#alteredSignedByteIsInvalid` (`signature_mismatch`) (Task 3).
5. **Revogação e LCR fora do ar** — certificado revogado antes da assinatura (inválida), revogado depois (sem carimbo do tempo não há como provar a ordem: indeterminada) e LCR inacessível (indeterminada, nunca "válida"). Testes: `CadesRoundTripTest#certificateRevokedBeforeSigningIsInvalid`, `#certificateRevokedAfterSigningIsIndeterminate`, `#unreachableCrlIsIndeterminate` (Task 3).

## Estratégia de teste

- **Unidade e integração no JUnit, sem rede:** a AC de teste (`DevPki`) é gerada no próprio teste, com as LCRs servidas num HTTP local; as âncoras entram pela configuração (`signer.extraTrustAnchors`), o mesmo caminho que dev usa. Ida e volta real: `/prepare` → RAW calculada com a chave privada de teste (o que o PSC faz) → `/assemble` → `/verify` `valid`; depois adulteração do documento, da assinatura, do PDF e do certificado.
- **HTTP:** servidor real numa porta livre, cliente `java.net.http`, todos os erros do contrato e a higiene de log.
- **Container:** `docker build`, subida com AC de dev, `/health` com e sem token, recusa de boot em produção; chamada a partir do container do `api` pela rede do compose.
- **Prova manual (Task 0):** sandbox do VIDaaS + validar.iti.gov.br, antes de qualquer commit.
- O código das Tasks 1–4 foi executado ao escrever o plano (36 testes verdes, 3 corridas seguidas) e a prova contra um PSC falso local deu `valid` nas duas assinaturas; o que só a Task 0 pode provar é o PSC real e o validador do ITI.

## Ambiente de execução

- Prova: `.claude/signer-proof/` na raiz do monorepo (fora de qualquer git; descartável).
- Repo: `apps/signer` (`git init -b main`), worktree `apps/signer/.claude/mod19b`.
- Maven: `apps/signer/.claude/mod19b/bin/mvn <goals>` (container com cache no volume `rotasaude-signer-m2`). Rode sempre de dentro do worktree: `(cd apps/signer/.claude/mod19b && bin/mvn test)`.
- A suíte leva ~25 s depois do primeiro download de dependências.

## Mapa de arquivos

| Arquivo (repo `signer`, salvo indicação) | Responsabilidade | Task |
|---|---|---|
| `.claude/signer-proof/` (raiz do monorepo, fora de git) | prova técnica: motor + `Proof` + `PscClient` | 0 |
| `docs/pesquisa/<data>-prova-tecnica-signer.md` (repo `docs`) | achados da prova | 0 |
| `.gitignore`, `.dockerignore`, `pom.xml`, `bin/mvn` | esqueleto e build | 1 |
| `src/main/java/rotasaude/signer/SignerFailure.java` | códigos de erro do contrato | 1 |
| `src/main/java/rotasaude/signer/Config.java` | ambiente, guarda de produção | 1 |
| `src/main/java/rotasaude/signer/engine/{Kind,PreparedSignature,PreparedStateCodec}.java` | tipos e o estado selado | 1 |
| `src/main/java/rotasaude/signer/devpki/DevPki.java` | AC de teste no formato ICP | 2 |
| `src/main/java/rotasaude/signer/trust/ConfiguredAnchorsProviderCA.java` + `META-INF/services/org.demoiselle.signer.core.ca.provider.ProviderCA` | âncoras extras por configuração | 2 |
| `src/main/java/rotasaude/signer/engine/Chain.java` | cadeia até âncora confiável | 2 |
| `src/test/java/rotasaude/signer/support/TestPki.java` | AC de teste + LCR local para os testes | 2 |
| `src/main/java/rotasaude/signer/engine/{Policies,SignedAttributes,CmsAssembler,PdfPlaceholder,ValidationMaterial,TrackingCrlRepository,Verification,SignatureVerifier,SignatureEngine}.java` + `META-INF/services/org.demoiselle.signer.core.repository.CRLRepository` | motor CAdES/PAdES | 3 |
| `src/main/java/rotasaude/signer/http/SignerServer.java`, `App.java`, `src/main/resources/log4j2.xml` | HTTP, boot, log | 4 |
| `Dockerfile`, `README.md`, `.github/workflows/ci.yml` | imagem, documentação, CI | 5 |
| `docker-compose.yml` (raiz do monorepo, fora de git) | serviço `signer`, env e volume do `api`/`worker` | 5 |

---

### Task 0: Prova técnica — sandbox do VIDaaS → CAdES e PAdES AD-RB no validar.iti.gov.br

Nada de repositório antes desta task passar. **Critério de parada:** qualquer um destes → pare, registre na pesquisa (Step 8) e reporte ao coordenador e ao usuário, sem seguir para a Task 1: (a) o PSC não aceita `multi_signature` com dois hashes RAW; (b) a RAW não confere como PKCS#1 v1.5 SHA-256 com DigestInfo sobre os atributos (o `Proof` diz o que ela é); (c) o validar.iti.gov.br não reconhece a política AD-RB ou acusa erro de estrutura em qualquer dos dois arquivos; (d) a cadeia do certificado de sandbox não é montada pelo Demoiselle.

**Files:**
- Create (fora de git): `.claude/signer-proof/pom.xml`, `.claude/signer-proof/src/main/java/rotasaude/signer/SignerFailure.java`, `.claude/signer-proof/src/main/java/rotasaude/signer/engine/{Kind,Policies,PreparedSignature,Chain,SignedAttributes,CmsAssembler,PdfPlaceholder,TrackingCrlRepository,ValidationMaterial,Verification,SignatureVerifier,SignatureEngine}.java`, `.claude/signer-proof/src/main/java/rotasaude/signer/proof/{PscClient,Proof}.java`, `.claude/signer-proof/src/main/resources/log4j2.xml`, `.claude/signer-proof/src/main/resources/META-INF/services/org.demoiselle.signer.core.repository.CRLRepository`
- Create (repo `docs`): `pesquisa/<AAAA-MM-DD da execução>-prova-tecnica-signer.md`

**Interfaces:**
- Consumes: API de PSC do ITI (DOC-ICP-17.01 v3.0 §6.4.5, lida ao escrever o plano): `GET <base>/oauth/authorize` (`response_type=code`, `client_id`, `redirect_uri`, `state`, `scope`, `lifetime`, `code_challenge`, `code_challenge_method=S256`, `login_hint`); `POST <base>/oauth/token` (form: `grant_type=authorization_code`, `client_id`, `client_secret`, `code`, `redirect_uri`, `code_verifier`) → `{ access_token, expires_in, authorized_identification }`; `GET <base>/oauth/certificate-discovery` (Bearer) → `{ status, certificates: [ { alias, certificate: <PEM> } ] }`; `POST <base>/oauth/signature` (Bearer) `{ hashes: [ { id, alias, hash, hash_algorithm: "2.16.840.1.101.3.4.2.1", signature_format: "RAW" } ] }` → `{ certificate_alias, signatures: [ { id, raw_signature } ] }`. `<base>` é a `base_url` do PSC já com o prefixo de versão (ex.: termina em `/v0`).
- Produces: os arquivos do motor (`rotasaude.signer.engine.*`, `rotasaude.signer.SignerFailure`) que as Tasks 1–3 copiam para o repo; a pesquisa com o formato da RAW, a cadeia, as políticas e o resultado do ITI.

- [ ] **Step 1: Confira o 19a em `origin/main` do api**

```bash
/opt/homebrew/bin/git -C apps/api fetch origin
/opt/homebrew/bin/git -C apps/api log --oneline -1 origin/main
for f in app/models/consultation.rb app/models/consultation_addendum.rb app/services/clinical_record/access.rb \
         app/services/ledi/consultation_mapping.rb spec/support/clinical_record_helpers.rb \
         db/city_migrate/20261007400002_add_consultations.rb; do
  /opt/homebrew/bin/git -C apps/api cat-file -e "origin/main:$f" && echo "ok $f" || echo "FALTA $f"
done
/opt/homebrew/bin/git -C apps/api grep -n "consultation.finalized\|consultation.addendum_added" origin/main -- app | head -4
```
Expected: todos `ok` e os dois eventos aparecem. **Pare** e reporte ao coordenador se algo `FALTA` (o 19a ainda não fechou).

- [ ] **Step 2: HARD STOP — peça as credenciais de sandbox ao usuário**

Não invente nenhum valor. Peça ao usuário que crie, no próprio terminal, o arquivo `~/.config/rotasaude/vidaas-sandbox.env` com `chmod 600`, contendo:

```
PSC_BASE_URL=<URL base da API de PSC do sandbox, com o prefixo de versão>
PSC_CLIENT_ID=<client_id da aplicação cadastrada no sandbox>
PSC_CLIENT_SECRET=<client_secret>
PSC_REDIRECT_URI=<redirect_uri cadastrada; de preferência http://localhost:8765/callback>
PSC_CPF=<CPF do titular do certificado de teste, só dígitos>
```

e que confirme quando estiver pronto, dizendo também: o host da `base_url` (só o host, para a pesquisa) e se a `redirect_uri` é `localhost`. Você nunca lê, imprime nem copia esse arquivo; ele só entra no container por `--env-file`. Sem credencial: pare aqui e reporte.

- [ ] **Step 3: Crie o projeto da prova**

Em `.claude/signer-proof/` (raiz do monorepo; `mkdir -p` dos diretórios). `pom.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>rotasaude</groupId>
  <artifactId>signer-proof</artifactId>
  <version>0.0.0</version>
  <packaging>jar</packaging>
  <name>signer-proof</name>
  <description>Prova técnica do 19b (descartável, fora de qualquer repositório): sandbox de PSC → CAdES e PAdES AD-RB para o validar.iti.gov.br.</description>

  <properties>
    <maven.compiler.release>21</maven.compiler.release>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <demoiselle.version>4.6.2</demoiselle.version>
    <bouncycastle.version>1.80</bouncycastle.version>
    <pdfbox.version>3.0.3</pdfbox.version>
    <jackson.version>2.17.2</jackson.version>
    <log4j.version>2.25.5</log4j.version>
  </properties>

  <dependencies>
    <dependency>
      <groupId>org.demoiselle.signer</groupId>
      <artifactId>policy-impl-cades</artifactId>
      <version>${demoiselle.version}</version>
    </dependency>
    <dependency>
      <groupId>org.demoiselle.signer</groupId>
      <artifactId>policy-impl-pades</artifactId>
      <version>${demoiselle.version}</version>
      <exclusions>
        <!-- o Demoiselle não manipula PDF; o signer usa PDFBox 3 -->
        <exclusion>
          <groupId>org.apache.pdfbox</groupId>
          <artifactId>pdfbox</artifactId>
        </exclusion>
      </exclusions>
    </dependency>
    <dependency>
      <groupId>org.demoiselle.signer</groupId>
      <artifactId>chain-icp-brasil</artifactId>
      <version>${demoiselle.version}</version>
    </dependency>
    <dependency>
      <!-- desligada sozinha com SIGNER_ENV=production (DisablingUtil do Demoiselle) -->
      <groupId>org.demoiselle.signer</groupId>
      <artifactId>chain-icp-brasil-homolog</artifactId>
      <version>${demoiselle.version}</version>
    </dependency>
    <dependency>
      <groupId>org.bouncycastle</groupId>
      <artifactId>bcpkix-jdk18on</artifactId>
      <version>${bouncycastle.version}</version>
    </dependency>
    <dependency>
      <groupId>org.apache.pdfbox</groupId>
      <artifactId>pdfbox</artifactId>
      <version>${pdfbox.version}</version>
    </dependency>
    <dependency>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-databind</artifactId>
      <version>${jackson.version}</version>
    </dependency>
    <dependency>
      <groupId>org.apache.logging.log4j</groupId>
      <artifactId>log4j-core</artifactId>
      <version>${log4j.version}</version>
    </dependency>
  </dependencies>

  <build>
    <finalName>signer-proof</finalName>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-dependency-plugin</artifactId>
        <version>3.7.1</version>
        <executions>
          <execution>
            <id>copy-dependencies</id>
            <phase>package</phase>
            <goals><goal>copy-dependencies</goal></goals>
            <configuration>
              <includeScope>runtime</includeScope>
              <outputDirectory>${project.build.directory}/lib</outputDirectory>
            </configuration>
          </execution>
        </executions>
      </plugin>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-jar-plugin</artifactId>
        <version>3.4.2</version>
        <configuration>
          <archive>
            <manifest>
              <mainClass>rotasaude.signer.proof.Proof</mainClass>
              <addClasspath>true</addClasspath>
              <classpathPrefix>lib/</classpathPrefix>
              <addDefaultImplementationEntries>true</addDefaultImplementationEntries>
            </manifest>
          </archive>
        </configuration>
      </plugin>
    </plugins>
  </build>
</project>
```

`src/main/java/rotasaude/signer/SignerFailure.java`:

```java
package rotasaude.signer;

/**
 * Recusa com código do contrato (§9). A mensagem é o próprio código: nunca carrega
 * texto de exceção de terceiros (que pode trazer DN com nome e CPF do titular).
 */
public final class SignerFailure extends RuntimeException {

    public enum Code {
        INVALID_REQUEST(400, "invalid_request"),
        UNAUTHORIZED(401, "unauthorized"),
        INVALID_CERTIFICATE(422, "invalid_certificate"),
        INVALID_SIGNATURE_VALUE(422, "invalid_signature_value"),
        INTERNAL(500, "internal");

        public final int status;
        public final String wire;

        Code(int status, String wire) {
            this.status = status;
            this.wire = wire;
        }
    }

    public final Code code;

    public SignerFailure(Code code) {
        super(code.wire, null, false, false);
        this.code = code;
    }
}
```

`src/main/java/rotasaude/signer/engine/Kind.java`:

```java
package rotasaude.signer.engine;

import java.util.Locale;
import rotasaude.signer.SignerFailure;

public enum Kind {
    CADES, PADES;

    public static Kind parse(Object value) {
        if ("cades".equals(value)) return CADES;
        if ("pades".equals(value)) return PADES;
        throw new SignerFailure(SignerFailure.Code.INVALID_REQUEST);
    }

    public String wire() {
        return name().toLowerCase(Locale.ROOT);
    }
}
```

`src/main/java/rotasaude/signer/engine/Policies.java`:

```java
package rotasaude.signer.engine;

import java.time.Instant;
import java.util.Date;
import java.util.EnumMap;
import java.util.HashSet;
import java.util.Map;
import java.util.Set;
import org.demoiselle.signer.policy.engine.asn1.etsi.SignaturePolicy;
import org.demoiselle.signer.policy.engine.asn1.etsi.SigningPeriod;
import org.demoiselle.signer.policy.engine.factory.PolicyFactory;
import rotasaude.signer.SignerFailure;

/**
 * Políticas AD-RB do Demoiselle (artefatos da LPA embutidos no jar, sem rede).
 * CAdES: a padrão do CAdESSigner 4.6.2; PAdES: a padrão do PAdESSigner 4.6.2.
 * O OID nunca é fixado no código: sai do .der da política.
 */
public final class Policies {

    public static final String SIGNATURE_ALGORITHM = "SHA256withRSA";

    private static final Map<Kind, PolicyFactory.Policies> CURRENT = new EnumMap<>(Map.of(
            Kind.CADES, PolicyFactory.Policies.AD_RB_CADES_2_4,
            Kind.PADES, PolicyFactory.Policies.AD_RB_PADES_1_3));

    private static final Map<Kind, Set<String>> ACCEPTED = new EnumMap<>(Kind.class);

    private Policies() {}

    public static PolicyFactory.Policies of(Kind kind) {
        return CURRENT.get(kind);
    }

    public static SignaturePolicy load(PolicyFactory.Policies policy) {
        return PolicyFactory.getInstance().loadPolicy(policy);
    }

    public static String oid(Kind kind) {
        return load(of(kind)).getSignPolicyInfo().getSignPolicyIdentifier().getValue();
    }

    /** OIDs de todas as versões AD-RB do formato: assinatura antiga continua verificável. */
    public static synchronized Set<String> acceptedOids(Kind kind) {
        return ACCEPTED.computeIfAbsent(kind, k -> {
            String prefix = k == Kind.CADES ? "AD_RB_CADES_" : "AD_RB_PADES_";
            Set<String> oids = new HashSet<>();
            for (PolicyFactory.Policies p : PolicyFactory.Policies.values()) {
                if (p.name().startsWith(prefix)) {
                    oids.add(load(p).getSignPolicyInfo().getSignPolicyIdentifier().getValue());
                }
            }
            return Set.copyOf(oids);
        });
    }

    /** Política fora do período de assinatura é configuração vencida do serviço: 500. */
    public static void requireInForce(Kind kind, Instant at) {
        SigningPeriod period = load(of(kind)).getSignPolicyInfo().getSignatureValidationPolicy().getSigningPeriod();
        Date when = Date.from(at);
        if (period.getNotBefore() != null && when.before(period.getNotBefore().getDate())) {
            throw new SignerFailure(SignerFailure.Code.INTERNAL);
        }
        if (period.getNotAfter() != null && when.after(period.getNotAfter().getDate())) {
            throw new SignerFailure(SignerFailure.Code.INTERNAL);
        }
    }
}
```

`src/main/java/rotasaude/signer/engine/PreparedSignature.java`:

```java
package rotasaude.signer.engine;

import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;

/**
 * O que o /prepare devolve e o /assemble recebe de volta (pelo prepared_state).
 * signedAttributesDer é exatamente o que o PSC assina (RAW = SHA256withRSA sobre esses bytes).
 * preparedPdf só existe em PAdES: o PDF com a assinatura reservada (zeros) e a data M gravada.
 */
public record PreparedSignature(Kind kind, byte[] certificateDer, byte[] signedAttributesDer, byte[] preparedPdf) {

    public byte[] toBeSignedSha256() {
        try {
            return MessageDigest.getInstance("SHA-256").digest(signedAttributesDer);
        } catch (NoSuchAlgorithmException e) {
            throw new IllegalStateException(e);
        }
    }
}
```

`src/main/java/rotasaude/signer/engine/Chain.java`:

```java
package rotasaude.signer.engine;

import java.security.cert.X509Certificate;
import java.util.List;
import org.demoiselle.signer.core.ca.manager.CAManager;
import rotasaude.signer.SignerFailure;

/** Cadeia montada pelas âncoras do Demoiselle (ICP-Brasil e, se configuradas, as extras). */
public final class Chain {

    private Chain() {}

    /** Titular, intermediária(s) e raiz confiável; menos que 3 elos = certificado recusado. */
    public static List<X509Certificate> of(X509Certificate certificate) {
        try {
            List<X509Certificate> chain = List.copyOf(CAManager.getInstance().getCertificateChain(certificate));
            if (chain.size() < 3 || !CAManager.getInstance().isRootCA(chain.get(chain.size() - 1))) {
                throw new SignerFailure(SignerFailure.Code.INVALID_CERTIFICATE);
            }
            return chain;
        } catch (SignerFailure e) {
            throw e;
        } catch (RuntimeException e) {
            throw new SignerFailure(SignerFailure.Code.INVALID_CERTIFICATE);
        }
    }
}
```

`src/main/java/rotasaude/signer/engine/SignedAttributes.java`:

```java
package rotasaude.signer.engine;

import java.io.IOException;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;
import java.security.cert.Certificate;
import java.security.cert.X509Certificate;
import java.time.Instant;
import java.time.temporal.ChronoUnit;
import java.util.Date;
import org.bouncycastle.asn1.ASN1Encoding;
import org.bouncycastle.asn1.DERSet;
import org.bouncycastle.asn1.cms.AttributeTable;
import org.bouncycastle.asn1.cms.CMSAttributes;
import org.bouncycastle.asn1.cms.Time;
import org.demoiselle.signer.policy.impl.cades.pkcs7.impl.CAdESSigner;
import rotasaude.signer.SignerFailure;

/**
 * Atributos assinados da política AD-RB, montados pelo Demoiselle sem chave privada
 * (CAdESSigner#prepareSignedAttributes é público e só precisa do certificado).
 * CAdES ganha signingTime (NGS2.02.03); PAdES não (o instante vai na entrada M do PDF).
 */
public final class SignedAttributes {

    private SignedAttributes() {}

    public static byte[] build(Kind kind, byte[] content, X509Certificate certificate, Instant signingTime) {
        CAdESSigner signer = new CAdESSigner(Policies.SIGNATURE_ALGORITHM, Policies.of(kind), kind == Kind.PADES);
        signer.setCertificates(new Certificate[] {certificate});
        signer.setHash(sha256(content));
        AttributeTable table;
        try {
            table = signer.prepareSignedAttributes(content, null);
            signer.prepareAlgAndLength();
        } catch (RuntimeException e) {
            // cadeia, validade, LCR ou tamanho de chave recusados pelo Demoiselle
            throw new SignerFailure(SignerFailure.Code.INVALID_CERTIFICATE);
        }
        if (kind == Kind.CADES && table.get(CMSAttributes.signingTime) == null) {
            table = table.add(CMSAttributes.signingTime, new Time(Date.from(signingTime.truncatedTo(ChronoUnit.SECONDS))));
        }
        try {
            return new DERSet(table.toASN1EncodableVector()).getEncoded(ASN1Encoding.DER);
        } catch (IOException e) {
            throw new SignerFailure(SignerFailure.Code.INTERNAL);
        }
    }

    static byte[] sha256(byte[] content) {
        try {
            return MessageDigest.getInstance("SHA-256").digest(content);
        } catch (NoSuchAlgorithmException e) {
            throw new IllegalStateException(e);
        }
    }
}
```

`src/main/java/rotasaude/signer/engine/CmsAssembler.java`:

```java
package rotasaude.signer.engine;

import java.io.ByteArrayOutputStream;
import java.io.OutputStream;
import java.security.GeneralSecurityException;
import java.security.Signature;
import java.security.cert.X509Certificate;
import java.util.Arrays;
import java.util.List;
import org.bouncycastle.asn1.ASN1Encoding;
import org.bouncycastle.asn1.ASN1Set;
import org.bouncycastle.asn1.cms.AttributeTable;
import org.bouncycastle.asn1.x509.AlgorithmIdentifier;
import org.bouncycastle.cert.jcajce.JcaCertStore;
import org.bouncycastle.cms.CMSAbsentContent;
import org.bouncycastle.cms.CMSSignedDataGenerator;
import org.bouncycastle.cms.SignerInfoGenerator;
import org.bouncycastle.cms.SimpleAttributeTableGenerator;
import org.bouncycastle.cms.jcajce.JcaSignerInfoGeneratorBuilder;
import org.bouncycastle.operator.ContentSigner;
import org.bouncycastle.operator.DefaultSignatureAlgorithmIdentifierFinder;
import org.bouncycastle.operator.jcajce.JcaDigestCalculatorProviderBuilder;
import rotasaude.signer.SignerFailure;

/**
 * Monta o CMS destacado com a assinatura RAW devolvida pelo PSC (PKCS#1 v1.5 sobre os
 * atributos assinados). O Demoiselle não tem assinatura em duas fases sem chave privada
 * (prepare*Sign/envelop* exigem PrivateKey); o BouncyCastle (dependência do próprio
 * Demoiselle) fecha a lacuna com um ContentSigner que só devolve o valor já calculado.
 */
public final class CmsAssembler {

    private CmsAssembler() {}

    public static byte[] assemble(byte[] signedAttributesDer, X509Certificate certificate,
                                  List<X509Certificate> chain, byte[] rawSignature) {
        requireMatches(signedAttributesDer, certificate, rawSignature);
        try {
            AttributeTable table = new AttributeTable(ASN1Set.getInstance(signedAttributesDer));
            SignerInfoGenerator signerInfo = new JcaSignerInfoGeneratorBuilder(new JcaDigestCalculatorProviderBuilder().build())
                    .setSignedAttributeGenerator(new SimpleAttributeTableGenerator(table))
                    .build(new Precomputed(signedAttributesDer, rawSignature), certificate);
            CMSSignedDataGenerator generator = new CMSSignedDataGenerator();
            generator.addSignerInfoGenerator(signerInfo);
            generator.addCertificates(new JcaCertStore(chain));
            return generator.generate(new CMSAbsentContent(), false).getEncoded(ASN1Encoding.DER);
        } catch (Exception e) {
            throw new SignerFailure(SignerFailure.Code.INTERNAL);
        }
    }

    /** RAW que não confere com a chave pública do certificado = 422 (nada é montado). */
    static void requireMatches(byte[] signedAttributesDer, X509Certificate certificate, byte[] rawSignature) {
        try {
            Signature verifier = Signature.getInstance(Policies.SIGNATURE_ALGORITHM);
            verifier.initVerify(certificate.getPublicKey());
            verifier.update(signedAttributesDer);
            if (verifier.verify(rawSignature)) return;
        } catch (GeneralSecurityException | IllegalArgumentException e) {
            // valor malformado cai no mesmo 422
        }
        throw new SignerFailure(SignerFailure.Code.INVALID_SIGNATURE_VALUE);
    }

    private static final class Precomputed implements ContentSigner {
        private final byte[] expected;
        private final byte[] signature;
        private final ByteArrayOutputStream written = new ByteArrayOutputStream();

        Precomputed(byte[] expected, byte[] signature) {
            this.expected = expected;
            this.signature = signature;
        }

        @Override
        public AlgorithmIdentifier getAlgorithmIdentifier() {
            return new DefaultSignatureAlgorithmIdentifierFinder().find(Policies.SIGNATURE_ALGORITHM);
        }

        @Override
        public OutputStream getOutputStream() {
            return written;
        }

        @Override
        public byte[] getSignature() {
            if (!Arrays.equals(written.toByteArray(), expected)) {
                throw new IllegalStateException("signed attributes differ from the prepared ones");
            }
            return signature.clone();
        }
    }
}
```

`src/main/java/rotasaude/signer/engine/PdfPlaceholder.java`:

```java
package rotasaude.signer.engine;

import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.time.Instant;
import java.util.Arrays;
import java.util.Calendar;
import java.util.HexFormat;
import java.util.TimeZone;
import org.apache.pdfbox.Loader;
import org.apache.pdfbox.pdmodel.PDDocument;
import org.apache.pdfbox.pdmodel.interactive.digitalsignature.ExternalSigningSupport;
import org.apache.pdfbox.pdmodel.interactive.digitalsignature.PDSignature;
import org.apache.pdfbox.pdmodel.interactive.digitalsignature.SignatureOptions;
import rotasaude.signer.SignerFailure;

/**
 * PDF com a assinatura reservada (assinatura invisível, SubFilter ETSI.CAdES.detached,
 * entrada M = instante da assinatura). O Demoiselle não manipula PDF (a documentação do
 * policy-impl-pades manda usar o PDFBox); a assinatura é em duas fases porque entre o
 * /prepare e o /assemble o hash vai ao PSC.
 */
public final class PdfPlaceholder {

    /** Bytes reservados para o CMS (cadeia ICP de 3–4 certificados cabe com folga). */
    public static final int SIGNATURE_SIZE = 24_576;

    private PdfPlaceholder() {}

    public static byte[] prepare(byte[] original, Instant signingTime) {
        try (PDDocument document = Loader.loadPDF(original);
             SignatureOptions options = new SignatureOptions()) {
            PDSignature signature = new PDSignature();
            signature.setFilter(PDSignature.FILTER_ADOBE_PPKLITE);
            signature.setSubFilter(PDSignature.SUBFILTER_ETSI_CADES_DETACHED);
            Calendar when = Calendar.getInstance(TimeZone.getTimeZone("UTC"));
            when.setTimeInMillis(signingTime.toEpochMilli());
            signature.setSignDate(when);
            options.setPreferredSignatureSize(SIGNATURE_SIZE);
            document.addSignature(signature, options);
            ByteArrayOutputStream out = new ByteArrayOutputStream();
            ExternalSigningSupport external = document.saveIncrementalForExternalSigning(out);
            external.setSignature(new byte[0]); // grava o PDF com o espaço reservado em zeros
            return out.toByteArray();
        } catch (IOException | RuntimeException e) {
            throw new SignerFailure(SignerFailure.Code.INVALID_REQUEST);
        }
    }

    /** Bytes cobertos pela assinatura (tudo menos o valor de /Contents). */
    public static byte[] signedContent(byte[] preparedPdf) {
        int[] range = byteRange(preparedPdf);
        byte[] content = Arrays.copyOfRange(preparedPdf, range[0], range[0] + range[1]);
        byte[] tail = Arrays.copyOfRange(preparedPdf, range[2], range[2] + range[3]);
        byte[] joined = Arrays.copyOf(content, content.length + tail.length);
        System.arraycopy(tail, 0, joined, content.length, tail.length);
        return joined;
    }

    public static byte[] inject(byte[] preparedPdf, byte[] cms) {
        int[] range = byteRange(preparedPdf);
        int start = range[0] + range[1] + 1;      // depois do '<'
        int capacity = range[2] - 1 - start;      // antes do '>'
        byte[] hex = HexFormat.of().withUpperCase().formatHex(cms).getBytes(StandardCharsets.US_ASCII);
        if (hex.length > capacity) throw new SignerFailure(SignerFailure.Code.INTERNAL);
        byte[] signed = preparedPdf.clone();
        System.arraycopy(hex, 0, signed, start, hex.length);
        return signed;
    }

    static int[] byteRange(byte[] pdf) {
        try (PDDocument document = Loader.loadPDF(pdf)) {
            PDSignature signature = document.getLastSignatureDictionary();
            if (signature == null) throw new SignerFailure(SignerFailure.Code.INVALID_REQUEST);
            int[] range = signature.getByteRange();
            if (range == null || range.length != 4) throw new SignerFailure(SignerFailure.Code.INVALID_REQUEST);
            return range;
        } catch (IOException e) {
            throw new SignerFailure(SignerFailure.Code.INVALID_REQUEST);
        }
    }
}
```

`src/main/java/rotasaude/signer/engine/TrackingCrlRepository.java`:

```java
package rotasaude.signer.engine;

import java.security.NoSuchProviderException;
import java.security.cert.X509Certificate;
import java.time.Instant;
import java.util.Collection;
import java.util.concurrent.atomic.AtomicReference;
import org.demoiselle.signer.core.extension.ICPBR_CRL;
import org.demoiselle.signer.core.repository.CRLRepository;
import org.demoiselle.signer.core.repository.ConfigurationRepo;
import org.demoiselle.signer.core.repository.OnLineCRLRepository;

/**
 * Repositório de LCR do Demoiselle (registrado por ServiceLoader): baixa como o
 * OnLineCRLRepository e anota o instante do último download bem-sucedido (/health).
 */
public final class TrackingCrlRepository implements CRLRepository {

    private static final AtomicReference<Instant> LAST_SUCCESS = new AtomicReference<>();

    public static Instant lastSuccess() {
        return LAST_SUCCESS.get();
    }

    @Override
    public Collection<ICPBR_CRL> getX509CRL(X509Certificate certificate) throws NoSuchProviderException {
        Collection<ICPBR_CRL> crls = new OnLineCRLRepository(ConfigurationRepo.getInstance().getProxy()).getX509CRL(certificate);
        if (crls != null && !crls.isEmpty()) LAST_SUCCESS.set(Instant.now());
        return crls;
    }
}
```

`src/main/java/rotasaude/signer/engine/ValidationMaterial.java`:

```java
package rotasaude.signer.engine;

import java.security.cert.X509Certificate;
import java.util.List;
import org.bouncycastle.asn1.ASN1Encoding;
import org.bouncycastle.cert.jcajce.JcaCertStore;
import org.bouncycastle.cert.jcajce.JcaX509CRLHolder;
import org.bouncycastle.cms.CMSAbsentContent;
import org.bouncycastle.cms.CMSSignedDataGenerator;
import org.demoiselle.signer.core.extension.ICPBR_CRL;
import org.demoiselle.signer.core.repository.CRLRepositoryFactory;
import rotasaude.signer.SignerFailure;

/**
 * Provas de validade do ato (ADR 0032): cadeia e LCRs vigentes, num CMS degenerado
 * (certs-only, RFC 5652 §5.1, sem signatário). LCR indisponível não impede: vai sem ela.
 */
public final class ValidationMaterial {

    private ValidationMaterial() {}

    public static byte[] collect(List<X509Certificate> chain) {
        try {
            CMSSignedDataGenerator generator = new CMSSignedDataGenerator();
            generator.addCertificates(new JcaCertStore(chain));
            for (X509Certificate certificate : chain) {
                if (certificate.getSubjectX500Principal().equals(certificate.getIssuerX500Principal())) continue;
                try {
                    for (ICPBR_CRL crl : CRLRepositoryFactory.factoryCRLRepository().getX509CRL(certificate)) {
                        generator.addCRL(new JcaX509CRLHolder(crl.getCRL()));
                    }
                } catch (Exception unavailable) {
                    // segue sem a LCR deste elo; o /verify diz indeterminate
                }
            }
            return generator.generate(new CMSAbsentContent()).getEncoded(ASN1Encoding.DER);
        } catch (Exception e) {
            throw new SignerFailure(SignerFailure.Code.INTERNAL);
        }
    }
}
```

`src/main/java/rotasaude/signer/engine/Verification.java`:

```java
package rotasaude.signer.engine;

import java.time.Instant;
import java.util.List;

/** Resultado do /verify (contrato §9). Campos do titular são null quando não há como lê-los. */
public record Verification(String status, String signerCpf, String signerName, String policyOid,
                           Instant signedAt, List<String> reasons) {

    public static final String VALID = "valid";
    public static final String INVALID = "invalid";
    public static final String INDETERMINATE = "indeterminate";

    static Verification malformed() {
        return new Verification(INVALID, null, null, null, null, List.of("malformed_signature"));
    }
}
```

`src/main/java/rotasaude/signer/engine/SignatureVerifier.java`:

```java
package rotasaude.signer.engine;

import java.security.cert.X509Certificate;
import java.time.Instant;
import java.util.ArrayList;
import java.util.Collection;
import java.util.Date;
import java.util.LinkedHashSet;
import java.util.List;
import java.util.Locale;
import java.util.Set;
import javax.naming.ldap.LdapName;
import javax.naming.ldap.Rdn;
import org.bouncycastle.asn1.ASN1Encodable;
import org.bouncycastle.asn1.ASN1InputStream;
import org.bouncycastle.asn1.ASN1Primitive;
import org.bouncycastle.asn1.cms.Attribute;
import org.bouncycastle.asn1.cms.AttributeTable;
import org.bouncycastle.asn1.cms.CMSAttributes;
import org.bouncycastle.asn1.cms.ContentInfo;
import org.bouncycastle.asn1.cms.Time;
import org.bouncycastle.asn1.esf.SignaturePolicyId;
import org.bouncycastle.asn1.pkcs.PKCSObjectIdentifiers;
import org.bouncycastle.cert.X509CertificateHolder;
import org.bouncycastle.cert.jcajce.JcaX509CertificateConverter;
import org.bouncycastle.cms.CMSProcessableByteArray;
import org.bouncycastle.cms.CMSSignedData;
import org.bouncycastle.cms.SignerInformation;
import org.demoiselle.signer.core.exception.CertificateRevocationException;
import org.demoiselle.signer.core.exception.CertificateValidatorCRLException;
import org.demoiselle.signer.core.extension.BasicCertificate;
import org.demoiselle.signer.core.extension.ICPBRCertificatePF;
import org.demoiselle.signer.core.validator.CRLValidator;
import org.demoiselle.signer.policy.impl.cades.SignatureInformations;
import org.demoiselle.signer.policy.impl.cades.ValidationMessageCode;
import org.demoiselle.signer.policy.impl.cades.pkcs7.impl.CAdESChecker;
import org.demoiselle.signer.policy.impl.pades.pkcs7.impl.PAdESChecker;
import org.apache.pdfbox.Loader;
import org.apache.pdfbox.pdmodel.PDDocument;
import org.apache.pdfbox.pdmodel.interactive.digitalsignature.PDSignature;

/**
 * Validação no servidor (NGS2.03): criptografia e atributos obrigatórios pelo checker do
 * Demoiselle; política, cadeia, instante, keyUsage e revogação conferidos aqui, com motivo.
 * Qualquer motivo de recusa → invalid; só motivos de dúvida → indeterminate.
 */
public final class SignatureVerifier {

    private static final Set<String> INDETERMINATE_REASONS = Set.of("revocation_unavailable", "certificate_revoked_after_signing");

    private SignatureVerifier() {}

    public static Verification verify(Kind kind, byte[] document, byte[] signature) {
        byte[] content;
        byte[] cms;
        Instant pdfSignDate = null;
        List<String> reasons = new ArrayList<>();
        if (kind == Kind.CADES) {
            content = document;
            cms = signature;
        } else {
            try (PDDocument pdf = Loader.loadPDF(signature)) {
                PDSignature dictionary = pdf.getLastSignatureDictionary();
                if (dictionary == null) return Verification.malformed();
                int[] range = dictionary.getByteRange();
                if (range == null || range.length != 4) return Verification.malformed();
                if ((long) range[2] + range[3] != signature.length) reasons.add("pdf_modified_after_signing");
                content = dictionary.getSignedContent(signature);
                cms = canonical(dictionary.getContents(signature));
                if (dictionary.getSignDate() != null) pdfSignDate = dictionary.getSignDate().toInstant();
            } catch (Exception e) {
                return Verification.malformed();
            }
        }

        SignerInformation signer;
        X509Certificate certificate;
        try {
            CMSSignedData data = new CMSSignedData(new CMSProcessableByteArray(content), cms);
            Collection<SignerInformation> signers = data.getSignerInfos().getSigners();
            if (signers.size() != 1) return Verification.malformed();
            signer = signers.iterator().next();
            @SuppressWarnings("unchecked")
            Collection<X509CertificateHolder> matches = data.getCertificates().getMatches(signer.getSID());
            if (matches.isEmpty()) return Verification.malformed();
            certificate = new JcaX509CertificateConverter().getCertificate(matches.iterator().next());
        } catch (Exception e) {
            return Verification.malformed();
        }

        String policyOid = policyOid(signer.getSignedAttributes());
        Instant signedAt = kind == Kind.CADES ? signingTime(signer.getSignedAttributes()) : pdfSignDate;

        List<SignatureInformations> infos;
        try {
            infos = (kind == Kind.CADES ? new CAdESChecker() : new PAdESChecker()).checkDetachedSignature(content, cms);
        } catch (RuntimeException mismatch) {
            infos = null;
            reasons.add("signature_mismatch");
        }
        if (infos != null) {
            if (infos.size() != 1) return Verification.malformed();
            SignatureInformations info = infos.get(0);
            if (info.isInvalidSignature()) reasons.add("signature_mismatch");
            for (ValidationMessageCode code : info.getValidatorErrorCodes()) reasons.add(reasonFor(code));
            for (ValidationMessageCode code : info.getValidatorWarningCodes()) {
                if (code == ValidationMessageCode.POLICY_NOT_FOUND || code == ValidationMessageCode.PKCS7_ATTRIBUTE_NOT_FOUND) {
                    reasons.add("policy_mismatch");
                }
            }
            if (info.getChain() == null || info.getChain().size() < 3) reasons.add("untrusted_chain");
        }

        if (policyOid == null || !Policies.acceptedOids(kind).contains(policyOid)) reasons.add("policy_mismatch");
        if (signedAt == null) {
            reasons.add("signing_time_missing");
        } else if (signedAt.isBefore(certificate.getNotBefore().toInstant()) || signedAt.isAfter(certificate.getNotAfter().toInstant())) {
            reasons.add("certificate_expired");
        }
        boolean[] keyUsage = certificate.getKeyUsage();
        if (keyUsage == null || !keyUsage[0] || !keyUsage[1]) reasons.add("key_usage");
        reasons.remove("revocation_unavailable");
        String revocation = revocation(certificate, signedAt);
        if (revocation != null) reasons.add(revocation);

        List<String> unique = List.copyOf(new LinkedHashSet<>(reasons));
        String status = unique.isEmpty() ? Verification.VALID
                : unique.stream().allMatch(INDETERMINATE_REASONS::contains) ? Verification.INDETERMINATE : Verification.INVALID;
        return new Verification(status, cpf(certificate), name(certificate), policyOid, signedAt, unique);
    }

    private static String reasonFor(ValidationMessageCode code) {
        return switch (code) {
            case CRL_NOT_ACCESS -> "revocation_unavailable"; // refeito abaixo por revocation()
            case CERTIFICATE_CHAIN_FAILED -> "untrusted_chain";
            case SIGNED_ATTRIBUTE_NOT_FOUND, PKCS7_ATTRIBUTE_NOT_FOUND, POLICY_NOT_FOUND -> "policy_mismatch";
            case INVALID_SIGNATURE, SIGNATURE_INVALID, SIGNATURE_MISMATCH, SIGNATURE_MISMATCH_DIGEST -> "signature_mismatch";
            default -> code.name().toLowerCase(Locale.ROOT);
        };
    }

    /** NGS2.03.02: revogação conferida no instante da assinatura (sem carimbo, o signingTime). */
    private static String revocation(X509Certificate certificate, Instant signedAt) {
        CRLValidator validator = new CRLValidator();
        try {
            validator.validate(certificate, null);
            return null;
        } catch (CertificateRevocationException revoked) {
            if (signedAt == null) return "certificate_revoked";
            try {
                validator.validate(certificate, Date.from(signedAt));
                return "certificate_revoked_after_signing";
            } catch (CertificateRevocationException | CertificateValidatorCRLException before) {
                return "certificate_revoked";
            }
        } catch (CertificateValidatorCRLException unavailable) {
            return "revocation_unavailable";
        }
    }

    private static String policyOid(AttributeTable attributes) {
        if (attributes == null) return null;
        Attribute attribute = attributes.get(PKCSObjectIdentifiers.id_aa_ets_sigPolicyId);
        if (attribute == null) return null;
        try {
            ASN1Encodable value = attribute.getAttrValues().getObjectAt(0);
            return SignaturePolicyId.getInstance(value).getSigPolicyId().getId();
        } catch (RuntimeException e) {
            return null;
        }
    }

    private static Instant signingTime(AttributeTable attributes) {
        if (attributes == null) return null;
        Attribute attribute = attributes.get(CMSAttributes.signingTime);
        if (attribute == null) return null;
        try {
            return Time.getInstance(attribute.getAttrValues().getObjectAt(0)).getDate().toInstant();
        } catch (RuntimeException e) {
            return null;
        }
    }

    private static String cpf(X509Certificate certificate) {
        try {
            BasicCertificate basic = new BasicCertificate(certificate);
            ICPBRCertificatePF pf = basic.hasCertificatePF() ? basic.getICPBRCertificatePF() : null;
            String cpf = pf == null ? null : pf.getCPF();
            return cpf == null ? null : cpf.replaceAll("\\D", "");
        } catch (RuntimeException e) {
            return null;
        }
    }

    /** CN da ICP-Brasil é "NOME DO TITULAR:CPF"; devolve só o nome. */
    private static String name(X509Certificate certificate) {
        try {
            for (Rdn rdn : new LdapName(certificate.getSubjectX500Principal().getName()).getRdns()) {
                if ("CN".equalsIgnoreCase(rdn.getType())) {
                    String cn = rdn.getValue().toString();
                    int colon = cn.indexOf(':');
                    return colon < 0 ? cn : cn.substring(0, colon);
                }
            }
        } catch (Exception e) {
            return null;
        }
        return null;
    }

    /** /Contents vem com zeros de enchimento depois do DER; o checker recebe só o DER. */
    private static byte[] canonical(byte[] padded) throws java.io.IOException {
        try (ASN1InputStream in = new ASN1InputStream(padded)) {
            ASN1Primitive object = in.readObject();
            return ContentInfo.getInstance(object).getEncoded("DER");
        }
    }
}
```

`src/main/java/rotasaude/signer/engine/SignatureEngine.java`:

```java
package rotasaude.signer.engine;

import java.io.ByteArrayInputStream;
import java.security.cert.CertificateException;
import java.security.cert.CertificateFactory;
import java.security.cert.X509Certificate;
import java.time.Clock;
import java.time.Instant;
import java.util.List;
import rotasaude.signer.SignerFailure;

/** As três operações do contrato §9 sobre bytes; o HTTP fica em rotasaude.signer.http. */
public final class SignatureEngine {

    public record Assembled(byte[] signature, byte[] validationMaterial) {}

    private final Clock clock;

    public SignatureEngine(Clock clock) {
        this.clock = clock;
    }

    public PreparedSignature prepare(Kind kind, byte[] document, byte[] certificateDer) {
        X509Certificate certificate = certificate(certificateDer);
        if (!"RSA".equals(certificate.getPublicKey().getAlgorithm())) {
            throw new SignerFailure(SignerFailure.Code.INVALID_CERTIFICATE);
        }
        Instant now = clock.instant();
        Policies.requireInForce(kind, now);
        Chain.of(certificate);
        if (kind == Kind.CADES) {
            return new PreparedSignature(kind, certificateDer, SignedAttributes.build(kind, document, certificate, now), null);
        }
        byte[] preparedPdf = PdfPlaceholder.prepare(document, now);
        byte[] content = PdfPlaceholder.signedContent(preparedPdf);
        return new PreparedSignature(kind, certificateDer, SignedAttributes.build(kind, content, certificate, now), preparedPdf);
    }

    public Assembled assemble(PreparedSignature prepared, byte[] rawSignature) {
        X509Certificate certificate = certificate(prepared.certificateDer());
        List<X509Certificate> chain = Chain.of(certificate);
        byte[] cms = CmsAssembler.assemble(prepared.signedAttributesDer(), certificate, chain, rawSignature);
        byte[] signature = prepared.kind() == Kind.CADES ? cms : PdfPlaceholder.inject(prepared.preparedPdf(), cms);
        return new Assembled(signature, ValidationMaterial.collect(chain));
    }

    public Verification verify(Kind kind, byte[] document, byte[] signature) {
        if (kind == Kind.CADES && document == null) throw new SignerFailure(SignerFailure.Code.INVALID_REQUEST);
        return SignatureVerifier.verify(kind, document, signature);
    }

    static X509Certificate certificate(byte[] der) {
        try {
            return (X509Certificate) CertificateFactory.getInstance("X.509").generateCertificate(new ByteArrayInputStream(der));
        } catch (CertificateException | RuntimeException e) {
            throw new SignerFailure(SignerFailure.Code.INVALID_CERTIFICATE);
        }
    }
}
```

`src/main/java/rotasaude/signer/proof/PscClient.java`:

```java
package rotasaude.signer.proof;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ArrayNode;
import com.fasterxml.jackson.databind.node.ObjectNode;
import java.io.ByteArrayInputStream;
import java.net.URI;
import java.net.URLEncoder;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.nio.charset.StandardCharsets;
import java.security.cert.CertificateFactory;
import java.security.cert.X509Certificate;
import java.util.ArrayList;
import java.util.Base64;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

/**
 * Cliente mínimo da API de PSC do ITI (DOC-ICP-17.01 v3.0, §6.4.5): authorize, token,
 * certificate-discovery e signature. Só para a prova técnica; em produção quem fala
 * com o PSC é o api (Signatures::Psc::*). Nunca imprime token, segredo nem code_verifier.
 */
final class PscClient {

    record Token(String accessToken, long expiresIn, String authorizedIdentification) {}

    record HashToSign(String id, String alias, byte[] sha256) {}

    private static final ObjectMapper JSON = new ObjectMapper();
    private static final String SHA256_OID = "2.16.840.1.101.3.4.2.1";

    private final HttpClient http = HttpClient.newHttpClient();
    private final String baseUrl;
    private final String clientId;
    private final String clientSecret;
    private final String redirectUri;

    PscClient(String baseUrl, String clientId, String clientSecret, String redirectUri) {
        this.baseUrl = baseUrl.endsWith("/") ? baseUrl.substring(0, baseUrl.length() - 1) : baseUrl;
        this.clientId = clientId;
        this.clientSecret = clientSecret;
        this.redirectUri = redirectUri;
    }

    String authorizeUrl(String state, String codeChallenge, String scope, int lifetimeSeconds, String cpf) {
        Map<String, String> query = new LinkedHashMap<>();
        query.put("response_type", "code");
        query.put("client_id", clientId);
        query.put("redirect_uri", redirectUri);
        query.put("state", state);
        query.put("scope", scope);
        query.put("lifetime", String.valueOf(lifetimeSeconds));
        query.put("code_challenge", codeChallenge);
        query.put("code_challenge_method", "S256");
        query.put("login_hint", cpf);
        return baseUrl + "/oauth/authorize?" + form(query);
    }

    Token token(String code, String codeVerifier) throws Exception {
        Map<String, String> body = new LinkedHashMap<>();
        body.put("grant_type", "authorization_code");
        body.put("client_id", clientId);
        body.put("client_secret", clientSecret);
        body.put("code", code);
        body.put("redirect_uri", redirectUri);
        body.put("code_verifier", codeVerifier);
        JsonNode response = send(HttpRequest.newBuilder(URI.create(baseUrl + "/oauth/token"))
                .header("Content-Type", "application/x-www-form-urlencoded")
                .POST(HttpRequest.BodyPublishers.ofString(form(body))).build(), "token");
        return new Token(response.path("access_token").asText(), response.path("expires_in").asLong(),
                response.path("authorized_identification").asText());
    }

    List<X509Certificate> certificates(String accessToken) throws Exception {
        JsonNode response = send(HttpRequest.newBuilder(URI.create(baseUrl + "/oauth/certificate-discovery"))
                .header("Accept", "application/json").header("Authorization", "Bearer " + accessToken)
                .GET().build(), "certificate-discovery");
        List<X509Certificate> certificates = new ArrayList<>();
        CertificateFactory factory = CertificateFactory.getInstance("X.509");
        for (JsonNode item : response.path("certificates")) {
            String pem = item.path("certificate").asText();
            certificates.add((X509Certificate) factory.generateCertificate(new ByteArrayInputStream(pem.getBytes(StandardCharsets.US_ASCII))));
        }
        return certificates;
    }

    /** signature_format RAW; devolve id → assinatura PKCS#1 decodificada. */
    Map<String, byte[]> signRaw(String accessToken, List<HashToSign> hashes) throws Exception {
        ObjectNode body = JSON.createObjectNode();
        ArrayNode items = body.putArray("hashes");
        for (HashToSign hash : hashes) {
            items.addObject().put("id", hash.id()).put("alias", hash.alias())
                    .put("hash", Base64.getEncoder().encodeToString(hash.sha256()))
                    .put("hash_algorithm", SHA256_OID).put("signature_format", "RAW");
        }
        JsonNode response = send(HttpRequest.newBuilder(URI.create(baseUrl + "/oauth/signature"))
                .header("Content-Type", "application/json").header("Accept", "application/json")
                .header("Authorization", "Bearer " + accessToken)
                .POST(HttpRequest.BodyPublishers.ofString(body.toString())).build(), "signature");
        Map<String, byte[]> signatures = new LinkedHashMap<>();
        for (JsonNode item : response.path("signatures")) {
            signatures.put(item.path("id").asText(), Base64.getMimeDecoder().decode(item.path("raw_signature").asText()));
        }
        return signatures;
    }

    private JsonNode send(HttpRequest request, String step) throws Exception {
        HttpResponse<String> response = http.send(request, HttpResponse.BodyHandlers.ofString());
        if (response.statusCode() / 100 != 2) {
            JsonNode error = response.body().isBlank() ? JSON.createObjectNode() : JSON.readTree(response.body());
            throw new IllegalStateException(step + " → HTTP " + response.statusCode() + " error=" + error.path("error").asText()
                    + " description=" + error.path("error_description").asText());
        }
        return JSON.readTree(response.body());
    }

    private static String form(Map<String, String> values) {
        return values.entrySet().stream()
                .map(e -> URLEncoder.encode(e.getKey(), StandardCharsets.UTF_8) + "=" + URLEncoder.encode(e.getValue(), StandardCharsets.UTF_8))
                .collect(Collectors.joining("&"));
    }
}
```

`src/main/java/rotasaude/signer/proof/Proof.java`:

```java
package rotasaude.signer.proof;

import com.sun.net.httpserver.HttpServer;
import java.io.BufferedReader;
import java.io.ByteArrayOutputStream;
import java.io.InputStreamReader;
import java.io.OutputStream;
import java.net.InetSocketAddress;
import java.net.URI;
import java.net.URLDecoder;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;
import java.security.MessageDigest;
import java.security.SecureRandom;
import java.security.Signature;
import java.security.cert.X509Certificate;
import java.time.Clock;
import java.util.Base64;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.concurrent.CompletableFuture;
import java.util.concurrent.TimeUnit;
import org.apache.pdfbox.pdmodel.PDDocument;
import org.apache.pdfbox.pdmodel.PDPage;
import org.apache.pdfbox.pdmodel.PDPageContentStream;
import org.apache.pdfbox.pdmodel.font.PDType1Font;
import org.apache.pdfbox.pdmodel.font.Standard14Fonts;
import rotasaude.signer.engine.Chain;
import rotasaude.signer.engine.Kind;
import rotasaude.signer.engine.Policies;
import rotasaude.signer.engine.PreparedSignature;
import rotasaude.signer.engine.SignatureEngine;
import rotasaude.signer.engine.Verification;

/**
 * Prova técnica (Task 0): um certificado em nuvem de sandbox assina, numa aprovação
 * (multi_signature), o hash dos atributos de um CAdES AD-RB (JSON) e de um PAdES AD-RB (PDF).
 * Grava em OUT_DIR os arquivos para o upload manual no validar.iti.gov.br.
 * Variáveis: PSC_BASE_URL, PSC_CLIENT_ID, PSC_CLIENT_SECRET, PSC_REDIRECT_URI, PSC_CPF, OUT_DIR.
 */
public final class Proof {

    private static final SecureRandom RANDOM = new SecureRandom();

    private Proof() {}

    public static void main(String[] args) throws Exception {
        run(System.getenv());
    }

    static void run(Map<String, String> env) throws Exception {
        PscClient psc = new PscClient(require(env, "PSC_BASE_URL"), require(env, "PSC_CLIENT_ID"),
                require(env, "PSC_CLIENT_SECRET"), require(env, "PSC_REDIRECT_URI"));
        String cpf = require(env, "PSC_CPF");
        Path out = Path.of(env.getOrDefault("OUT_DIR", "/out"));
        Files.createDirectories(out);

        String verifier = base64Url(random(32));
        String challenge = base64Url(MessageDigest.getInstance("SHA-256").digest(verifier.getBytes(StandardCharsets.US_ASCII)));
        String state = base64Url(random(16));
        System.out.println("1) Abra no navegador e aprove no app do PSC:\n" + psc.authorizeUrl(state, challenge, "multi_signature", 600, cpf));
        Map<String, String> callback = awaitCallback(require(env, "PSC_REDIRECT_URI"));
        if (callback.containsKey("error")) throw new IllegalStateException("autorização negada: " + callback.get("error"));
        if (!state.equals(callback.get("state"))) throw new IllegalStateException("state diferente do enviado");

        PscClient.Token token = psc.token(callback.get("code"), verifier);
        System.out.println("2) token ok; expires_in=" + token.expiresIn() + "s; CPF autorizado = PSC_CPF? "
                + cpf.equals(token.authorizedIdentification().replaceAll("\\D", "")));

        List<X509Certificate> certificates = psc.certificates(token.accessToken());
        if (certificates.isEmpty()) throw new IllegalStateException("certificate-discovery sem certificado");
        X509Certificate certificate = certificates.get(0);
        List<X509Certificate> chain = Chain.of(certificate);
        System.out.println("3) certificado: chave " + certificate.getPublicKey().getAlgorithm() + ", vence " + certificate.getNotAfter()
                + "; cadeia com " + chain.size() + " elos; raiz: " + chain.get(chain.size() - 1).getSubjectX500Principal().getName());

        SignatureEngine engine = new SignatureEngine(Clock.systemUTC());
        byte[] json = ("{\"schema\":\"rotasaude.consultation.v1\",\"prova\":\"assinatura tecnica 19b\",\"gerado_em\":\""
                + java.time.Instant.now() + "\"}").getBytes(StandardCharsets.UTF_8);
        byte[] pdf = samplePdf();
        PreparedSignature cades = engine.prepare(Kind.CADES, json, certificate.getEncoded());
        PreparedSignature pades = engine.prepare(Kind.PADES, pdf, certificate.getEncoded());
        System.out.println("4) atributos montados; política CAdES " + Policies.oid(Kind.CADES) + ", PAdES " + Policies.oid(Kind.PADES));

        Map<String, byte[]> raw = psc.signRaw(token.accessToken(), List.of(
                new PscClient.HashToSign("cades", "Prova 19b - JSON", cades.toBeSignedSha256()),
                new PscClient.HashToSign("pades", "Prova 19b - PDF", pades.toBeSignedSha256())));
        System.out.println("5) PSC devolveu " + raw.size() + " assinaturas RAW");
        diagnoseRaw("cades", certificate, cades, raw.get("cades"));
        diagnoseRaw("pades", certificate, pades, raw.get("pades"));

        SignatureEngine.Assembled signedJson = engine.assemble(cades, raw.get("cades"));
        SignatureEngine.Assembled signedPdf = engine.assemble(pades, raw.get("pades"));
        Files.write(out.resolve("documento.json"), json);
        Files.write(out.resolve("documento.json.p7s"), signedJson.signature());
        Files.write(out.resolve("documento-assinado.pdf"), signedPdf.signature());
        Files.write(out.resolve("material-de-validacao.p7c"), signedJson.validationMaterial());
        report("CAdES", engine.verify(Kind.CADES, json, signedJson.signature()));
        report("PAdES", engine.verify(Kind.PADES, null, signedPdf.signature()));
        System.out.println("7) Arquivos em " + out + ". Envie documento.json.p7s (com documento.json) e documento-assinado.pdf ao validar.iti.gov.br.");
    }

    /** RAW esperado: PKCS#1 v1.5 com DigestInfo(SHA-256) sobre os atributos. Se não, diz o que é. */
    private static void diagnoseRaw(String id, X509Certificate certificate, PreparedSignature prepared, byte[] raw) throws Exception {
        if (raw == null) throw new IllegalStateException("PSC não devolveu a assinatura " + id);
        Signature digestInfo = Signature.getInstance("SHA256withRSA");
        digestInfo.initVerify(certificate.getPublicKey());
        digestInfo.update(prepared.signedAttributesDer());
        if (digestInfo.verify(raw)) {
            System.out.println("   " + id + ": RAW = PKCS#1 v1.5 SHA-256 com DigestInfo (o esperado)");
            return;
        }
        Signature bare = Signature.getInstance("NONEwithRSA");
        bare.initVerify(certificate.getPublicKey());
        bare.update(prepared.toBeSignedSha256());
        System.out.println("   " + id + ": RAW NÃO confere como SHA256withRSA; como hash puro sem DigestInfo: " + bare.verify(raw)
                + " — registre na pesquisa e PARE");
    }

    private static void report(String label, Verification result) {
        System.out.println("6) " + label + " no verificador do signer: " + result.status() + " " + result.reasons()
                + " política " + result.policyOid() + " em " + result.signedAt());
    }

    /** redirect_uri em localhost: escuta a volta; senão, pede a URL de retorno colada no terminal. */
    private static Map<String, String> awaitCallback(String redirectUri) throws Exception {
        URI uri = URI.create(redirectUri);
        if (!"localhost".equals(uri.getHost()) && !"127.0.0.1".equals(uri.getHost())) {
            System.out.println("Cole aqui a URL completa para onde o PSC redirecionou:");
            return query(URI.create(new BufferedReader(new InputStreamReader(System.in, StandardCharsets.UTF_8)).readLine().trim()));
        }
        CompletableFuture<Map<String, String>> result = new CompletableFuture<>();
        HttpServer server = HttpServer.create(new InetSocketAddress("0.0.0.0", uri.getPort()), 0);
        server.createContext(uri.getPath().isEmpty() ? "/" : uri.getPath(), exchange -> {
            byte[] page = "Autorizacao recebida. Pode fechar esta aba.".getBytes(StandardCharsets.UTF_8);
            exchange.sendResponseHeaders(200, page.length);
            try (OutputStream body = exchange.getResponseBody()) {
                body.write(page);
            }
            result.complete(query(exchange.getRequestURI()));
        });
        server.start();
        try {
            return result.get(10, TimeUnit.MINUTES);
        } finally {
            server.stop(0);
        }
    }

    private static Map<String, String> query(URI uri) {
        Map<String, String> values = new HashMap<>();
        if (uri.getRawQuery() == null) return values;
        for (String pair : uri.getRawQuery().split("&")) {
            int eq = pair.indexOf('=');
            if (eq > 0) values.put(pair.substring(0, eq), URLDecoder.decode(pair.substring(eq + 1), StandardCharsets.UTF_8));
        }
        return values;
    }

    private static byte[] samplePdf() throws Exception {
        try (PDDocument document = new PDDocument()) {
            PDPage page = new PDPage();
            document.addPage(page);
            try (PDPageContentStream content = new PDPageContentStream(document, page)) {
                content.beginText();
                content.setFont(new PDType1Font(Standard14Fonts.FontName.HELVETICA), 12);
                content.newLineAtOffset(72, 720);
                content.showText("Rota Saude - prova tecnica 19b (documento de teste, sem dado real)");
                content.endText();
            }
            ByteArrayOutputStream out = new ByteArrayOutputStream();
            document.save(out);
            return out.toByteArray();
        }
    }

    private static String require(Map<String, String> env, String name) {
        String value = env.get(name);
        if (value == null || value.isBlank()) throw new IllegalStateException("falta a variável " + name);
        return value;
    }

    private static byte[] random(int size) {
        byte[] bytes = new byte[size];
        RANDOM.nextBytes(bytes);
        return bytes;
    }

    private static String base64Url(byte[] bytes) {
        return Base64.getUrlEncoder().withoutPadding().encodeToString(bytes);
    }
}
```

`src/main/resources/log4j2.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!--
  Só o log de acesso do signer (método, caminho, status, duração). O Demoiselle fica
  desligado: ele escreve o DN do titular (nome e CPF) em vários níveis.
-->
<Configuration status="WARN">
  <Appenders>
    <Console name="out" target="SYSTEM_OUT" follow="true">
      <PatternLayout pattern="%d{ISO8601}{UTC}Z %-5level %c{1} - %msg%n"/>
    </Console>
  </Appenders>
  <Loggers>
    <Logger name="rotasaude.signer" level="info" additivity="false">
      <AppenderRef ref="out"/>
    </Logger>
    <Logger name="org.demoiselle" level="off" additivity="false"/>
    <Logger name="org.apache.pdfbox" level="error" additivity="false">
      <AppenderRef ref="out"/>
    </Logger>
    <Root level="warn">
      <AppenderRef ref="out"/>
    </Root>
  </Loggers>
</Configuration>
```

`src/main/resources/META-INF/services/org.demoiselle.signer.core.repository.CRLRepository`:

```
rotasaude.signer.engine.TrackingCrlRepository
```

- [ ] **Step 4: Compile a prova**

```bash
(cd .claude/signer-proof && docker run --rm -v "$PWD":/src -w /src -v rotasaude-signer-m2:/root/.m2 \
  maven:3.9.9-eclipse-temurin-21 mvn -B -q -DskipTests package)
ls .claude/signer-proof/target/signer-proof.jar
docker run --rm -v "$PWD/.claude/signer-proof":/w -w /w eclipse-temurin:21-jre java -jar target/signer-proof.jar 2>&1 | grep -m1 "falta a variável"
```
Expected: o jar existe; sem variáveis, o `Proof` para com `falta a variável PSC_BASE_URL`. (Ao escrever o plano, este mesmo `Proof` rodou contra um PSC falso local com um e-CPF de teste e imprimiu `CAdES ... valid []` e `PAdES ... valid []`.)

- [ ] **Step 5: Peça ao usuário para rodar a prova no terminal dele**

É interativo (aprovação no celular) e usa o arquivo de credenciais dele. Passe este comando ao usuário (com `redirect_uri` em `localhost:8765`; para outra porta, troque o `-p`):

```bash
cd /Users/eduardovrocha/Development/ioit.solutions/rota-saude/.claude/signer-proof && mkdir -p out && \
docker run --rm -it --env-file ~/.config/rotasaude/vidaas-sandbox.env -e OUT_DIR=/out \
  -p 127.0.0.1:8765:8765 -v "$PWD":/w -v "$PWD/out":/out -w /w eclipse-temurin:21-jre \
  java -jar target/signer-proof.jar
```

O `Proof` imprime a URL de autorização (abrir no navegador, aprovar no app VIDaaS), recebe o retorno em `localhost:8765` (ou pede a URL de retorno colada, se a `redirect_uri` não for `localhost`), pede uma única aprovação `multi_signature` para os dois hashes e escreve em `out/`: `documento.json`, `documento.json.p7s`, `documento-assinado.pdf`, `material-de-validacao.p7c`. Peça ao usuário as linhas `2)` a `6)` da saída (não contêm segredo nem CPF; a `3)` traz o DN da AC raiz). Se ele preferir, leia o terminal dele com a ferramenta de leitura de terminal.

Expected: `2) ... CPF autorizado = PSC_CPF? true`; `3)` cadeia com 3 ou mais elos; `5)` duas assinaturas; as duas linhas de diagnóstico `RAW = PKCS#1 v1.5 SHA-256 com DigestInfo (o esperado)`; `6) CAdES ... valid` e `6) PAdES ... valid`. Uma linha `RAW NÃO confere` ou um erro HTTP em `token`/`signature` é critério de parada (b)/(a).

- [ ] **Step 6: Upload manual no validar.iti.gov.br**

Peça ao usuário que envie ao validar.iti.gov.br (a) `out/documento.json.p7s` com o conteúdo `out/documento.json` (assinatura destacada) e (b) `out/documento-assinado.pdf`, e que traga o relatório de cada um (print ou PDF do relatório). Os arquivos de `out/` são de teste e não têm dado real, mas ficam só na máquina do usuário.

Expected: para os dois, assinatura reconhecida, íntegra e com a política AD-RB identificada (CAdES v2.4 e PAdES v1.3). Se o certificado de sandbox for de cadeia de **homologação** da ICP-Brasil, o validador pode não reconhecer a raiz como confiável: registre exatamente o que ele disse; estrutura e política reconhecidas com raiz de homologação **não** é parada, mas a decisão de seguir é do usuário (pergunte). Erro de estrutura, política não reconhecida ou integridade recusada é critério de parada (c).

- [ ] **Step 7: Se a prova pediu ajuste no motor**

Qualquer mudança nos arquivos de `.claude/signer-proof/src/main/java/rotasaude/signer/` feita para a prova passar fica lá (as Tasks 1–3 copiam desta pasta) e vai, arquivo por arquivo e com o motivo, para a seção "Achados e ajustes" da pesquisa. Recompile (Step 4) e repita os Steps 5–6.

- [ ] **Step 8: Registre a prova em `docs/pesquisa/`**

Crie `docs/pesquisa/<AAAA-MM-DD da execução>-prova-tecnica-signer.md` com este conteúdo, preenchendo cada `<...>` com o que foi observado (sem segredo, sem CPF, sem nome do titular):

```markdown
# Prova técnica do `signer` (19b, Task 0): sandbox de PSC → validar.iti.gov.br

Data: <data da execução>. Plano: `docs/superpowers/plans/2026-10-08-module-19b-signature-signer.md`, Task 0.
Sem segredo neste arquivo: nada de client_id, client_secret, token, code_verifier, CPF ou nome do titular de teste.

## Ambiente

- PSC: VIDaaS, sandbox (host da `base_url`: <só o host>).
- Escopo pedido: `multi_signature`, `lifetime` 600 s; `expires_in` devolvido: <n> s.
- `authorized_identification` conferiu com o CPF pedido: <sim/não>.
- Certificado: algoritmo <RSA n bits>; cadeia de <n> elos; raiz: <DN da AC raiz> (<produção | homologação ICP-Brasil>).

## Resultado do PSC

- `/oauth/signature` com 2 hashes RAW numa aprovação: <ok / erro + error/error_description>.
- Formato do RAW (diagnóstico do Proof): <PKCS#1 v1.5 SHA-256 com DigestInfo | hash puro sem DigestInfo | outro>.

## Políticas e montagem

- CAdES: AD-RB v2.4, OID 2.16.76.1.7.1.1.2.4; PAdES: AD-RB v1.3, OID 2.16.76.1.7.1.11.1.3 (Demoiselle 4.6.2).
- Verificador do signer: CAdES <status + motivos>; PAdES <status + motivos>.

## validar.iti.gov.br (upload manual)

| Arquivo | Resultado | Política reconhecida | Observações (cadeia, revogação, avisos) |
|---|---|---|---|
| `documento.json.p7s` + `documento.json` | <...> | <...> | <...> |
| `documento-assinado.pdf` | <...> | <...> | <...> |

## Achados e ajustes

- <cada diferença entre o esperado e o obtido; o que mudou no motor (arquivo e linha) e por quê>.

## Decisão

<seguir com o plano | parado: motivo e o que precisa ser revisto com o usuário>.
```

Commit no repo `docs` (o checkout local é um clone; commite ali mesmo):

```bash
/opt/homebrew/bin/git -C docs status --short
/opt/homebrew/bin/git -C docs add pesquisa/<AAAA-MM-DD>-prova-tecnica-signer.md
/opt/homebrew/bin/git -C docs commit -m "docs: record the signer technical proof against the PSC sandbox

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```
Expected: `status --short` antes do `add` mostra o arquivo novo (e nada desta task além dele). Push do `docs` só com autorização (junto com a Task 6).

- [ ] **Step 9: Decida**

Prova aprovada (critérios (a)–(d) livres e o usuário de acordo, inclusive sobre raiz de homologação): siga para a Task 1. Senão: **pare** e reporte ao coordenador e ao usuário com o link da pesquisa.

---

### Task 1: Repositório local, esqueleto Maven, configuração e `prepared_state`

**Files:**
- Create: `apps/signer/.gitignore` (na `main` local)
- Create (worktree `apps/signer/.claude/mod19b`): `pom.xml`, `bin/mvn`, `.dockerignore`, `src/main/java/rotasaude/signer/{SignerFailure,Config}.java`, `src/main/java/rotasaude/signer/engine/{Kind,PreparedSignature,PreparedStateCodec}.java`, `src/main/java/rotasaude/signer/trust/ConfiguredAnchorsProviderCA.java` (o `Config` usa a constante `ENV`; o registro no `ServiceLoader` e o teste vêm na Task 2)
- Test: `src/test/java/rotasaude/signer/ConfigTest.java`, `src/test/java/rotasaude/signer/engine/PreparedStateCodecTest.java`

**Interfaces:**
- Consumes: `SignerFailure`, `Kind`, `PreparedSignature` da prova (Task 0, Step 3).
- Produces:
  - `rotasaude.signer.SignerFailure(Code)` com `Code { INVALID_REQUEST(400,"invalid_request"), UNAUTHORIZED(401,"unauthorized"), INVALID_CERTIFICATE(422,"invalid_certificate"), INVALID_SIGNATURE_VALUE(422,"invalid_signature_value"), INTERNAL(500,"internal") }`, campos `code`, `code.status`, `code.wire`.
  - `rotasaude.signer.Config(String token, int port, String environment, String extraTrustAnchors, String devPkiDir, String version)`; `static Config fromEnv(Map<String,String>)` (lança `IllegalArgumentException`); `DEFAULT_PORT = 8090`; `MIN_TOKEN_LENGTH = 32`.
  - `rotasaude.signer.engine.Kind { CADES, PADES }`, `static Kind parse(Object)` (400 se não for `"cades"`/`"pades"`), `String wire()`.
  - `rotasaude.signer.engine.PreparedSignature(Kind kind, byte[] certificateDer, byte[] signedAttributesDer, byte[] preparedPdf)`, `byte[] toBeSignedSha256()`.
  - `rotasaude.signer.engine.PreparedStateCodec(String sharedToken)`, `String encode(PreparedSignature)`, `PreparedSignature decode(String)` (400 se adulterado, de outro token ou malformado).
  - `rotasaude.signer.trust.ConfiguredAnchorsProviderCA` (`ENV = "SIGNER_EXTRA_TRUST_ANCHORS"`, `PROPERTY = "signer.extraTrustAnchors"`; lê o PEM a cada `getCAs()`).

- [ ] **Step 1: Crie o repositório local e o worktree**

```bash
test -e apps/signer && echo "JA EXISTE" || mkdir apps/signer
/opt/homebrew/bin/git init -b main apps/signer
printf 'target/\n/.claude/\n*.env\n.DS_Store\n' > apps/signer/.gitignore
/opt/homebrew/bin/git -C apps/signer add .gitignore
/opt/homebrew/bin/git -C apps/signer commit -m "chore: initialize repository

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
/opt/homebrew/bin/git -C apps/signer worktree add .claude/mod19b -b feat/mod-19b-signature main
```
Expected: `apps/signer` não existia (se `JA EXISTE`, pare e reporte); repositório com um commit; worktree criado. Nenhum `remote` (`git -C apps/signer remote -v` vazio).

- [ ] **Step 2: Esqueleto de build**

`apps/signer/.claude/mod19b/pom.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>rotasaude</groupId>
  <artifactId>signer</artifactId>
  <version>0.1.0</version>
  <packaging>jar</packaging>
  <name>signer</name>
  <description>Serviço interno do Rota Saúde que prepara, monta e valida assinaturas ICP-Brasil (CAdES e PAdES AD-RB) com o hash assinado fora (PSC em nuvem).</description>

  <properties>
    <maven.compiler.release>21</maven.compiler.release>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <demoiselle.version>4.6.2</demoiselle.version>
    <bouncycastle.version>1.80</bouncycastle.version>
    <pdfbox.version>3.0.3</pdfbox.version>
    <jackson.version>2.17.2</jackson.version>
    <log4j.version>2.25.5</log4j.version>
    <junit.version>5.10.3</junit.version>
  </properties>

  <dependencies>
    <dependency>
      <groupId>org.demoiselle.signer</groupId>
      <artifactId>policy-impl-cades</artifactId>
      <version>${demoiselle.version}</version>
    </dependency>
    <dependency>
      <groupId>org.demoiselle.signer</groupId>
      <artifactId>policy-impl-pades</artifactId>
      <version>${demoiselle.version}</version>
      <exclusions>
        <!-- o Demoiselle não manipula PDF; o signer usa PDFBox 3 -->
        <exclusion>
          <groupId>org.apache.pdfbox</groupId>
          <artifactId>pdfbox</artifactId>
        </exclusion>
      </exclusions>
    </dependency>
    <dependency>
      <groupId>org.demoiselle.signer</groupId>
      <artifactId>chain-icp-brasil</artifactId>
      <version>${demoiselle.version}</version>
    </dependency>
    <dependency>
      <!-- desligada sozinha com SIGNER_ENV=production (DisablingUtil do Demoiselle) -->
      <groupId>org.demoiselle.signer</groupId>
      <artifactId>chain-icp-brasil-homolog</artifactId>
      <version>${demoiselle.version}</version>
    </dependency>
    <dependency>
      <groupId>org.bouncycastle</groupId>
      <artifactId>bcpkix-jdk18on</artifactId>
      <version>${bouncycastle.version}</version>
    </dependency>
    <dependency>
      <groupId>org.apache.pdfbox</groupId>
      <artifactId>pdfbox</artifactId>
      <version>${pdfbox.version}</version>
    </dependency>
    <dependency>
      <groupId>com.fasterxml.jackson.core</groupId>
      <artifactId>jackson-databind</artifactId>
      <version>${jackson.version}</version>
    </dependency>
    <dependency>
      <groupId>org.apache.logging.log4j</groupId>
      <artifactId>log4j-core</artifactId>
      <version>${log4j.version}</version>
    </dependency>
    <dependency>
      <groupId>org.junit.jupiter</groupId>
      <artifactId>junit-jupiter</artifactId>
      <version>${junit.version}</version>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <build>
    <finalName>signer</finalName>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-surefire-plugin</artifactId>
        <version>3.2.5</version>
        <configuration>
          <environmentVariables>
            <!-- testes herméticos: só a AC de teste é âncora; nada de rede para cadeias reais -->
            <SIGNER_DISABLE_CHAIN_ICP_BRASIL>true</SIGNER_DISABLE_CHAIN_ICP_BRASIL>
            <SIGNER_DISABLE_CHAIN_ICP_BRASIL_HOMOLOG>true</SIGNER_DISABLE_CHAIN_ICP_BRASIL_HOMOLOG>
            <SIGNER_REPOSITORY_LPA_ONLINE>false</SIGNER_REPOSITORY_LPA_ONLINE>
            <SIGNER_CRL_CONNECTION_TIMEOUT>2000</SIGNER_CRL_CONNECTION_TIMEOUT>
          </environmentVariables>
        </configuration>
      </plugin>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-dependency-plugin</artifactId>
        <version>3.7.1</version>
        <executions>
          <execution>
            <id>copy-dependencies</id>
            <phase>package</phase>
            <goals><goal>copy-dependencies</goal></goals>
            <configuration>
              <includeScope>runtime</includeScope>
              <outputDirectory>${project.build.directory}/lib</outputDirectory>
            </configuration>
          </execution>
        </executions>
      </plugin>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-jar-plugin</artifactId>
        <version>3.4.2</version>
        <configuration>
          <archive>
            <manifest>
              <mainClass>rotasaude.signer.App</mainClass>
              <addClasspath>true</addClasspath>
              <classpathPrefix>lib/</classpathPrefix>
              <addDefaultImplementationEntries>true</addDefaultImplementationEntries>
            </manifest>
          </archive>
        </configuration>
      </plugin>
    </plugins>
  </build>
</project>
```

`apps/signer/.claude/mod19b/bin/mvn` (e `chmod +x`):

```bash
#!/usr/bin/env bash
# Maven 3.9 + JDK 21 em container: o host não precisa de Maven. Cache em volume nomeado.
# Uso: bin/mvn test | bin/mvn verify | bin/mvn -Dtest=CadesRoundTripTest test
set -euo pipefail
cd "$(dirname "$0")/.."
exec docker run --rm -v "$PWD":/src -w /src -v rotasaude-signer-m2:/root/.m2 \
  maven:3.9.9-eclipse-temurin-21 mvn -B "$@"
```

`apps/signer/.claude/mod19b/.dockerignore`:

```
target/
.git/
.claude/
```

Copie da prova os três tipos de base (sem mudança):

```bash
W=apps/signer/.claude/mod19b; PR=.claude/signer-proof/src/main/java/rotasaude/signer
mkdir -p $W/src/main/java/rotasaude/signer/engine $W/src/test/java/rotasaude/signer/engine
cp $PR/SignerFailure.java $W/src/main/java/rotasaude/signer/
cp $PR/engine/Kind.java $PR/engine/PreparedSignature.java $W/src/main/java/rotasaude/signer/engine/
```
(Se a pasta da prova não existir mais, o conteúdo dos três está na Task 0, Step 3.)

- [ ] **Step 3: Escreva os testes que falham**

`src/test/java/rotasaude/signer/ConfigTest.java`:

```java
package rotasaude.signer;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertFalse;
import static org.junit.jupiter.api.Assertions.assertThrows;

import java.util.Map;
import org.junit.jupiter.api.Test;

class ConfigTest {

    private static final String TOKEN = "t".repeat(32);

    @Test
    void defaultsToPort8090() {
        Config config = Config.fromEnv(Map.of("SIGNER_TOKEN", TOKEN));
        assertEquals(8090, config.port());
        assertFalse(config.toString().contains(TOKEN));
    }

    @Test
    void refusesMissingOrShortToken() {
        assertThrows(IllegalArgumentException.class, () -> Config.fromEnv(Map.of()));
        assertThrows(IllegalArgumentException.class, () -> Config.fromEnv(Map.of("SIGNER_TOKEN", "t".repeat(31))));
    }

    @Test
    void refusesTestAnchorsAndDevPkiInProduction() {
        assertThrows(IllegalArgumentException.class, () -> Config.fromEnv(Map.of(
                "SIGNER_TOKEN", TOKEN, "SIGNER_ENV", "production", "SIGNER_EXTRA_TRUST_ANCHORS", "/x/anchors.pem")));
        assertThrows(IllegalArgumentException.class, () -> Config.fromEnv(Map.of(
                "SIGNER_TOKEN", TOKEN, "SIGNER_ENV", "prod", "SIGNER_DEV_PKI_DIR", "/dev-pki")));
        assertEquals("/dev-pki", Config.fromEnv(Map.of("SIGNER_TOKEN", TOKEN, "SIGNER_DEV_PKI_DIR", "/dev-pki")).devPkiDir());
    }
}
```

`src/test/java/rotasaude/signer/engine/PreparedStateCodecTest.java`:

```java
package rotasaude.signer.engine;

import static org.junit.jupiter.api.Assertions.assertArrayEquals;
import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertNull;
import static org.junit.jupiter.api.Assertions.assertThrows;

import java.util.Base64;
import org.junit.jupiter.api.Test;
import rotasaude.signer.SignerFailure;

class PreparedStateCodecTest {

    private static final String TOKEN = "a".repeat(40);
    private final PreparedStateCodec codec = new PreparedStateCodec(TOKEN);
    private final PreparedSignature cades = new PreparedSignature(Kind.CADES, new byte[] {1, 2, 3}, new byte[] {4, 5}, null);

    @Test
    void roundTrips() {
        PreparedSignature back = codec.decode(codec.encode(new PreparedSignature(Kind.PADES, new byte[] {1}, new byte[] {2}, new byte[] {3, 4})));
        assertEquals(Kind.PADES, back.kind());
        assertArrayEquals(new byte[] {1}, back.certificateDer());
        assertArrayEquals(new byte[] {2}, back.signedAttributesDer());
        assertArrayEquals(new byte[] {3, 4}, back.preparedPdf());
        assertNull(codec.decode(codec.encode(cades)).preparedPdf());
    }

    @Test
    void refusesATamperedState() {
        byte[] raw = Base64.getDecoder().decode(codec.encode(cades));
        raw[5] ^= 0x01;
        assertInvalid(Base64.getEncoder().encodeToString(raw));
    }

    @Test
    void refusesAStateSealedWithAnotherToken() {
        assertInvalid(new PreparedStateCodec("b".repeat(40)).encode(cades));
    }

    @Test
    void refusesGarbage() {
        assertInvalid("%%not-base64%%");
        assertInvalid(Base64.getEncoder().encodeToString(new byte[10]));
    }

    private void assertInvalid(String state) {
        SignerFailure failure = assertThrows(SignerFailure.class, () -> codec.decode(state));
        assertEquals(SignerFailure.Code.INVALID_REQUEST, failure.code);
    }
}
```

- [ ] **Step 4: Rode e veja falhar**

Run: `(cd apps/signer/.claude/mod19b && bin/mvn -q test)`
Expected: FAIL de compilação — `cannot find symbol ... class Config` e `class PreparedStateCodec`.

- [ ] **Step 5: Implemente**

`src/main/java/rotasaude/signer/Config.java`:

```java
package rotasaude.signer;

import java.util.Map;
import rotasaude.signer.trust.ConfiguredAnchorsProviderCA;

/**
 * Configuração por ambiente. SIGNER_ENV é a mesma variável que o Demoiselle lê
 * (production desliga as cadeias de homologação); aqui ela também proíbe AC de teste.
 */
public record Config(String token, int port, String environment, String extraTrustAnchors, String devPkiDir, String version) {

    public static final int DEFAULT_PORT = 8090;
    public static final int MIN_TOKEN_LENGTH = 32;

    public static Config fromEnv(Map<String, String> env) {
        String token = env.get("SIGNER_TOKEN");
        if (token == null || token.length() < MIN_TOKEN_LENGTH) {
            throw new IllegalArgumentException("SIGNER_TOKEN ausente ou com menos de " + MIN_TOKEN_LENGTH + " caracteres");
        }
        int port = DEFAULT_PORT;
        String rawPort = env.get("SIGNER_PORT");
        if (rawPort != null && !rawPort.isBlank()) {
            try {
                port = Integer.parseInt(rawPort.trim());
            } catch (NumberFormatException e) {
                throw new IllegalArgumentException("SIGNER_PORT inválida");
            }
        }
        String environment = env.getOrDefault("SIGNER_ENV", "").trim();
        String extra = blankToNull(env.get(ConfiguredAnchorsProviderCA.ENV));
        String devPki = blankToNull(env.get("SIGNER_DEV_PKI_DIR"));
        boolean production = "production".equalsIgnoreCase(environment) || "prod".equalsIgnoreCase(environment);
        if (production && (extra != null || devPki != null)) {
            throw new IllegalArgumentException("SIGNER_EXTRA_TRUST_ANCHORS e SIGNER_DEV_PKI_DIR são proibidos com SIGNER_ENV=production");
        }
        String version = Config.class.getPackage().getImplementationVersion();
        return new Config(token, port, environment, extra, devPki, version == null ? "dev" : version);
    }

    private static String blankToNull(String value) {
        return value == null || value.isBlank() ? null : value;
    }

    @Override
    public String toString() {
        return "Config[port=" + port + ", environment=" + environment + "]"; // nunca o token
    }
}
```

`src/main/java/rotasaude/signer/engine/PreparedStateCodec.java`:

```java
package rotasaude.signer.engine;

import java.io.ByteArrayOutputStream;
import java.io.DataInputStream;
import java.io.DataOutputStream;
import java.io.IOException;
import java.nio.charset.StandardCharsets;
import java.security.GeneralSecurityException;
import java.security.MessageDigest;
import java.util.Arrays;
import java.util.Base64;
import javax.crypto.Mac;
import javax.crypto.spec.SecretKeySpec;
import rotasaude.signer.SignerFailure;

/**
 * prepared_state opaco: versão, tipo, certificado, atributos assinados e PDF preparado,
 * selados com HMAC-SHA256 (chave derivada do SIGNER_TOKEN). Estado adulterado = 400.
 */
public final class PreparedStateCodec {

    private static final int VERSION = 1;
    private static final int MAC_LENGTH = 32;
    private static final int MAX_FIELD = 32 * 1024 * 1024;

    private final byte[] key;

    public PreparedStateCodec(String sharedToken) {
        this.key = hmac(sharedToken.getBytes(StandardCharsets.UTF_8),
                "rotasaude.signer.prepared_state.v1".getBytes(StandardCharsets.UTF_8));
    }

    public String encode(PreparedSignature prepared) {
        try {
            ByteArrayOutputStream buffer = new ByteArrayOutputStream();
            DataOutputStream out = new DataOutputStream(buffer);
            out.writeByte(VERSION);
            out.writeByte(prepared.kind().ordinal());
            writeField(out, prepared.certificateDer());
            writeField(out, prepared.signedAttributesDer());
            writeField(out, prepared.preparedPdf() == null ? new byte[0] : prepared.preparedPdf());
            out.flush();
            byte[] body = buffer.toByteArray();
            byte[] sealed = Arrays.copyOf(body, body.length + MAC_LENGTH);
            System.arraycopy(hmac(key, body), 0, sealed, body.length, MAC_LENGTH);
            return Base64.getEncoder().encodeToString(sealed);
        } catch (IOException e) {
            throw new SignerFailure(SignerFailure.Code.INTERNAL);
        }
    }

    public PreparedSignature decode(String encoded) {
        try {
            byte[] sealed = Base64.getDecoder().decode(encoded);
            if (sealed.length <= MAC_LENGTH) throw invalid();
            byte[] body = Arrays.copyOf(sealed, sealed.length - MAC_LENGTH);
            byte[] mac = Arrays.copyOfRange(sealed, body.length, sealed.length);
            if (!MessageDigest.isEqual(mac, hmac(key, body))) throw invalid();
            DataInputStream in = new DataInputStream(new java.io.ByteArrayInputStream(body));
            if (in.readUnsignedByte() != VERSION) throw invalid();
            int kindOrdinal = in.readUnsignedByte();
            if (kindOrdinal >= Kind.values().length) throw invalid();
            Kind kind = Kind.values()[kindOrdinal];
            byte[] certificate = readField(in);
            byte[] attributes = readField(in);
            byte[] pdf = readField(in);
            if (in.available() != 0) throw invalid();
            return new PreparedSignature(kind, certificate, attributes, pdf.length == 0 ? null : pdf);
        } catch (IllegalArgumentException | IOException e) {
            throw invalid();
        }
    }

    private static SignerFailure invalid() {
        return new SignerFailure(SignerFailure.Code.INVALID_REQUEST);
    }

    private static void writeField(DataOutputStream out, byte[] value) throws IOException {
        out.writeInt(value.length);
        out.write(value);
    }

    private static byte[] readField(DataInputStream in) throws IOException {
        int length = in.readInt();
        if (length < 0 || length > MAX_FIELD || length > in.available()) throw invalid();
        return in.readNBytes(length);
    }

    private static byte[] hmac(byte[] key, byte[] data) {
        try {
            Mac mac = Mac.getInstance("HmacSHA256");
            mac.init(new SecretKeySpec(key, "HmacSHA256"));
            return mac.doFinal(data);
        } catch (GeneralSecurityException e) {
            throw new IllegalStateException(e);
        }
    }
}
```

`src/main/java/rotasaude/signer/trust/ConfiguredAnchorsProviderCA.java` (o `Config` usa a constante `ENV`; o registro no `ServiceLoader` vem na Task 2):

```java
package rotasaude.signer.trust;

import java.io.InputStream;
import java.nio.file.Files;
import java.nio.file.Path;
import java.security.cert.Certificate;
import java.security.cert.CertificateFactory;
import java.security.cert.X509Certificate;
import java.util.ArrayList;
import java.util.Collection;
import java.util.List;
import org.demoiselle.signer.core.ca.provider.ProviderCA;

/**
 * Âncoras extras (PEM) por configuração: SIGNER_EXTRA_TRUST_ANCHORS ou a propriedade
 * signer.extraTrustAnchors. Só AC de teste/desenvolvimento; proibido com SIGNER_ENV=production
 * (Config recusa o boot). Lido a cada chamada: o teste escreve o arquivo antes de usar.
 */
public final class ConfiguredAnchorsProviderCA implements ProviderCA {

    public static final String ENV = "SIGNER_EXTRA_TRUST_ANCHORS";
    public static final String PROPERTY = "signer.extraTrustAnchors";

    @Override
    public String getName() {
        return "rotasaude-extra-anchors";
    }

    @Override
    public Collection<X509Certificate> getCAs() {
        String path = System.getenv(ENV);
        if (path == null || path.isBlank()) path = System.getProperty(PROPERTY);
        if (path == null || path.isBlank()) return List.of();
        try (InputStream in = Files.newInputStream(Path.of(path))) {
            List<X509Certificate> anchors = new ArrayList<>();
            for (Certificate certificate : CertificateFactory.getInstance("X.509").generateCertificates(in)) {
                anchors.add((X509Certificate) certificate);
            }
            return anchors;
        } catch (Exception e) {
            return List.of();
        }
    }
}
```

- [ ] **Step 6: Rode e veja passar**

Run: `(cd apps/signer/.claude/mod19b && bin/mvn test)`
Expected: `Tests run: 7, Failures: 0, Errors: 0` (3 de `ConfigTest`, 4 de `PreparedStateCodecTest`), `BUILD SUCCESS`.

- [ ] **Step 7: Commit**

```bash
W=apps/signer/.claude/mod19b
/opt/homebrew/bin/git -C $W add pom.xml bin/mvn .dockerignore \
  src/main/java/rotasaude/signer/SignerFailure.java src/main/java/rotasaude/signer/Config.java \
  src/main/java/rotasaude/signer/engine/Kind.java src/main/java/rotasaude/signer/engine/PreparedSignature.java \
  src/main/java/rotasaude/signer/engine/PreparedStateCodec.java \
  src/main/java/rotasaude/signer/trust/ConfiguredAnchorsProviderCA.java \
  src/test/java/rotasaude/signer/ConfigTest.java src/test/java/rotasaude/signer/engine/PreparedStateCodecTest.java
/opt/homebrew/bin/git -C $W status --short
/opt/homebrew/bin/git -C $W commit -m "feat: scaffold signer service with config and sealed prepared state

Maven build run in a container (bin/mvn), environment config that refuses
test trust anchors and the dev PKI under SIGNER_ENV=production, and the
HMAC-sealed prepared_state carried by the api between /prepare and
/assemble.

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```
Expected: `status --short` vazio depois do commit (nada esquecido; `target/` ignorado).

---

### Task 2: AC de teste no formato ICP e âncoras por configuração

**Files:**
- Create: `src/main/java/rotasaude/signer/devpki/DevPki.java`, `src/main/resources/META-INF/services/org.demoiselle.signer.core.ca.provider.ProviderCA`, `src/main/java/rotasaude/signer/engine/Chain.java` (cópia da prova)
- Test: `src/test/java/rotasaude/signer/support/TestPki.java`, `src/test/java/rotasaude/signer/devpki/DevPkiTest.java`

**Interfaces:**
- Consumes: `SignerFailure`, `ConfiguredAnchorsProviderCA` (Task 1); `org.demoiselle.signer.core.ca.provider.ProviderCA` (`getName()`, `getCAs()`), `CAManager.getInstance().getCertificateChain/isRootCA`, `BasicCertificate#hasCertificatePF/getICPBRCertificatePF().getCPF()` (Demoiselle 4.6.2).
- Produces:
  - `rotasaude.signer.devpki.DevPki`: `static DevPki generate(String crlBaseUrl)`; `record Issued(X509Certificate certificate, PrivateKey privateKey)`; `Issued issue(String cpf, String name, LocalDate birthDate, Instant notBefore, Instant notAfter)`; `void revoke(BigInteger serial, Instant when)`; `byte[] crl(String name)` (`"root.crl"` | `"intermediate.crl"`); `List<X509Certificate> anchors()` (raiz, intermediária); `void writeTo(Path dir)` (`anchors.pem`, `root.pem`, `intermediate.pem`, `intermediate.key.pem` PKCS#8, `root.crl`, `intermediate.crl`); `main(<dir> <crlBaseUrl>)` que não sobrescreve.
  - Formato do e-CPF de teste (o PSC falso do `api` emite igual): RSA 2048, SHA256withRSA pela intermediária; `CN=<NOME>:<CPF>,OU=Teste,O=ICP-Brasil,C=BR`; `keyUsage` digitalSignature + nonRepudiation (crítica); SAN `otherName` 2.16.76.1.3.1 = `[0] EXPLICIT OCTET STRING` com `ddMMaaaa` + CPF + 11 zeros (NIS) + 15 zeros (RG) + 6 zeros (órgão/UF); AKI; ponto de distribuição `<crlBaseUrl>intermediate.crl`.
  - `rotasaude.signer.trust.ConfiguredAnchorsProviderCA` (Task 1) registrada no `ServiceLoader` do Demoiselle.
  - `rotasaude.signer.engine.Chain.of(X509Certificate) -> List<X509Certificate>` (titular … raiz; 422 `invalid_certificate` com menos de 3 elos ou sem raiz confiável).
  - Teste: `rotasaude.signer.support.TestPki.get()` (uma por JVM), `CPF`, `NAME`, `professional()`, `expiredProfessional()`, `static untrustedProfessional()`, `crlAvailable(boolean)`, `static rawSign(PrivateKey, byte[])`, `static samplePdf(String)`, campo `pki`.

- [ ] **Step 1: Escreva o apoio de teste e o teste que falha**

`src/test/java/rotasaude/signer/support/TestPki.java`:

```java
package rotasaude.signer.support;

import com.sun.net.httpserver.HttpServer;
import java.io.ByteArrayOutputStream;
import java.io.IOException;
import java.io.OutputStream;
import java.net.InetSocketAddress;
import java.nio.file.Files;
import java.nio.file.Path;
import java.security.PrivateKey;
import java.security.Signature;
import java.time.Duration;
import java.time.Instant;
import java.time.LocalDate;
import org.apache.pdfbox.pdmodel.PDDocument;
import org.apache.pdfbox.pdmodel.PDPage;
import org.apache.pdfbox.pdmodel.PDPageContentStream;
import org.apache.pdfbox.pdmodel.font.PDType1Font;
import org.apache.pdfbox.pdmodel.font.Standard14Fonts;
import rotasaude.signer.devpki.DevPki;
import rotasaude.signer.trust.ConfiguredAnchorsProviderCA;

/**
 * AC de teste no formato ICP, uma por JVM, com as LCRs servidas num HTTP local
 * (o Demoiselle baixa a LCR pelo ponto de distribuição do certificado). As âncoras
 * entram pela configuração (propriedade signer.extraTrustAnchors), como em produção
 * entraria SIGNER_EXTRA_TRUST_ANCHORS em dev.
 */
public final class TestPki {

    public static final String CPF = "52998224725";
    public static final String NAME = "ANA BEATRIZ SOUZA";

    private static TestPki instance;

    public final DevPki pki;
    private volatile boolean crlAvailable = true;

    private TestPki() throws IOException {
        HttpServer crlServer = HttpServer.create(new InetSocketAddress("127.0.0.1", 0), 0);
        pki = DevPki.generate("http://127.0.0.1:" + crlServer.getAddress().getPort() + "/");
        crlServer.createContext("/", exchange -> {
            String name = exchange.getRequestURI().getPath().substring(1);
            byte[] body = crlAvailable ? pki.crl(name) : new byte[0];
            exchange.getResponseHeaders().set("Connection", "close"); // sem keep-alive: conexão velha reaproveitada estoura o timeout da LCR
            exchange.sendResponseHeaders(crlAvailable ? 200 : 503, crlAvailable ? body.length : -1);
            try (OutputStream out = exchange.getResponseBody()) {
                out.write(body);
            }
        });
        crlServer.setExecutor(java.util.concurrent.Executors.newCachedThreadPool());
        crlServer.start();
        Path dir = Files.createTempDirectory("signer-test-pki");
        pki.writeTo(dir);
        System.setProperty(ConfiguredAnchorsProviderCA.PROPERTY, dir.resolve("anchors.pem").toString());
    }

    public static synchronized TestPki get() {
        if (instance == null) {
            try {
                instance = new TestPki();
            } catch (IOException e) {
                throw new IllegalStateException(e);
            }
        }
        return instance;
    }

    public DevPki.Issued professional() {
        Instant now = Instant.now();
        return pki.issue(CPF, NAME, LocalDate.of(1980, 5, 17), now.minus(Duration.ofDays(1)), now.plus(Duration.ofDays(365)));
    }

    public DevPki.Issued expiredProfessional() {
        Instant now = Instant.now();
        return pki.issue(CPF, NAME, LocalDate.of(1980, 5, 17), now.minus(Duration.ofDays(730)), now.minus(Duration.ofDays(365)));
    }

    /** O mesmo e-CPF, emitido por uma AC que não está nas âncoras. */
    public static DevPki.Issued untrustedProfessional() {
        Instant now = Instant.now();
        return DevPki.generate("http://127.0.0.1:9/").issue(CPF, NAME, LocalDate.of(1980, 5, 17),
                now.minus(Duration.ofDays(1)), now.plus(Duration.ofDays(365)));
    }

    public void crlAvailable(boolean available) {
        crlAvailable = available;
    }

    /** O que o PSC devolve em signature_format RAW: PKCS#1 v1.5 SHA-256 sobre os atributos assinados. */
    public static byte[] rawSign(PrivateKey key, byte[] signedAttributesDer) {
        try {
            Signature signature = Signature.getInstance("SHA256withRSA");
            signature.initSign(key);
            signature.update(signedAttributesDer);
            return signature.sign();
        } catch (Exception e) {
            throw new IllegalStateException(e);
        }
    }

    public static byte[] samplePdf(String text) {
        try (PDDocument document = new PDDocument()) {
            PDPage page = new PDPage();
            document.addPage(page);
            try (PDPageContentStream content = new PDPageContentStream(document, page)) {
                content.beginText();
                content.setFont(new PDType1Font(Standard14Fonts.FontName.HELVETICA), 12);
                content.newLineAtOffset(72, 720);
                content.showText(text);
                content.endText();
            }
            ByteArrayOutputStream out = new ByteArrayOutputStream();
            document.save(out);
            return out.toByteArray();
        } catch (IOException e) {
            throw new IllegalStateException(e);
        }
    }
}
```

`src/test/java/rotasaude/signer/devpki/DevPkiTest.java`:

```java
package rotasaude.signer.devpki;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertThrows;
import static org.junit.jupiter.api.Assertions.assertTrue;

import java.nio.file.Files;
import java.nio.file.Path;
import java.security.cert.X509Certificate;
import java.util.List;
import java.util.Set;
import java.util.stream.Collectors;
import java.util.stream.Stream;
import org.demoiselle.signer.core.extension.BasicCertificate;
import org.junit.jupiter.api.Test;
import rotasaude.signer.SignerFailure;
import rotasaude.signer.engine.Chain;
import rotasaude.signer.support.TestPki;

class DevPkiTest {

    private static final TestPki PKI = TestPki.get();

    @Test
    void issuesAnIcpBrasilNaturalPersonCertificate() {
        X509Certificate certificate = PKI.professional().certificate();
        BasicCertificate basic = new BasicCertificate(certificate);

        assertTrue(basic.hasCertificatePF());
        assertEquals(TestPki.CPF, basic.getICPBRCertificatePF().getCPF());
        assertTrue(certificate.getKeyUsage()[0] && certificate.getKeyUsage()[1], "digitalSignature + nonRepudiation");
    }

    @Test
    void chainResolvesThroughTheConfiguredAnchors() {
        List<X509Certificate> chain = Chain.of(PKI.professional().certificate());
        assertEquals(3, chain.size());
        assertEquals(PKI.pki.anchors().get(0), chain.get(2));
    }

    @Test
    void certificateFromAnUnknownAuthorityHasNoChain() {
        SignerFailure failure = assertThrows(SignerFailure.class, () -> Chain.of(TestPki.untrustedProfessional().certificate()));
        assertEquals(SignerFailure.Code.INVALID_CERTIFICATE, failure.code);
    }

    @Test
    void mainWritesTheFilesOnceAndNeverOverwrites() throws Exception {
        Path dir = Files.createTempDirectory("dev-pki");
        DevPki.main(new String[] {dir.toString(), "http://signer:8090/dev-pki/"});
        String anchors = Files.readString(dir.resolve("anchors.pem"));
        DevPki.main(new String[] {dir.toString(), "http://signer:8090/dev-pki/"});

        try (Stream<Path> files = Files.list(dir)) {
            assertEquals(Set.of("anchors.pem", "root.pem", "intermediate.pem", "intermediate.key.pem", "root.crl", "intermediate.crl"),
                    files.map(p -> p.getFileName().toString()).collect(Collectors.toSet()));
        }
        assertEquals(anchors, Files.readString(dir.resolve("anchors.pem")));
    }
}
```

- [ ] **Step 2: Rode e veja falhar**

Run: `(cd apps/signer/.claude/mod19b && bin/mvn -q test)`
Expected: FAIL de compilação — `package rotasaude.signer.devpki does not exist` / `cannot find symbol ... Chain`.

- [ ] **Step 3: A AC de teste**

`src/main/java/rotasaude/signer/devpki/DevPki.java`:

```java
package rotasaude.signer.devpki;

import java.io.IOException;
import java.io.StringWriter;
import java.math.BigInteger;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;
import java.security.KeyPair;
import java.security.KeyPairGenerator;
import java.security.PrivateKey;
import java.security.SecureRandom;
import java.security.cert.X509Certificate;
import java.time.Duration;
import java.time.Instant;
import java.time.LocalDate;
import java.time.format.DateTimeFormatter;
import java.util.Date;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import org.bouncycastle.asn1.ASN1EncodableVector;
import org.bouncycastle.asn1.ASN1ObjectIdentifier;
import org.bouncycastle.asn1.DEROctetString;
import org.bouncycastle.asn1.DERSequence;
import org.bouncycastle.asn1.DERTaggedObject;
import org.bouncycastle.asn1.x500.X500Name;
import org.bouncycastle.asn1.x509.BasicConstraints;
import org.bouncycastle.asn1.x509.CRLDistPoint;
import org.bouncycastle.asn1.x509.CRLReason;
import org.bouncycastle.asn1.x509.DistributionPoint;
import org.bouncycastle.asn1.x509.DistributionPointName;
import org.bouncycastle.asn1.x509.Extension;
import org.bouncycastle.asn1.x509.GeneralName;
import org.bouncycastle.asn1.x509.GeneralNames;
import org.bouncycastle.asn1.x509.KeyUsage;
import org.bouncycastle.cert.X509CRLHolder;
import org.bouncycastle.cert.X509v2CRLBuilder;
import org.bouncycastle.cert.X509v3CertificateBuilder;
import org.bouncycastle.cert.jcajce.JcaX509CertificateConverter;
import org.bouncycastle.cert.jcajce.JcaX509CertificateHolder;
import org.bouncycastle.cert.jcajce.JcaX509ExtensionUtils;
import org.bouncycastle.cert.jcajce.JcaX509v3CertificateBuilder;
import org.bouncycastle.openssl.jcajce.JcaPEMWriter;
import org.bouncycastle.operator.jcajce.JcaContentSignerBuilder;

/**
 * AC de teste no formato ICP-Brasil (raiz → intermediária → titular com o otherName
 * 2.16.76.1.3.1 de pessoa física), com LCRs. Usada pelos testes JUnit e, em desenvolvimento,
 * para o api montar o PSC falso. Nunca em produção (Config recusa o boot).
 */
public final class DevPki {

    public record Issued(X509Certificate certificate, PrivateKey privateKey) {}

    private static final ASN1ObjectIdentifier ICP_PF = new ASN1ObjectIdentifier("2.16.76.1.3.1");
    private static final SecureRandom RANDOM = new SecureRandom();

    private final String crlBaseUrl;
    private final KeyPair rootKeys;
    private final KeyPair intermediateKeys;
    private final X509Certificate root;
    private final X509Certificate intermediate;
    private final Map<BigInteger, Instant> revoked = new LinkedHashMap<>();

    private DevPki(String crlBaseUrl) throws Exception {
        this.crlBaseUrl = crlBaseUrl.endsWith("/") ? crlBaseUrl : crlBaseUrl + "/";
        Instant now = Instant.now();
        rootKeys = keys();
        intermediateKeys = keys();
        X500Name rootName = new X500Name("CN=AC Raiz de Teste Rota Saude,OU=Teste,O=ICP-Brasil,C=BR");
        X500Name intermediateName = new X500Name("CN=AC Intermediaria de Teste Rota Saude,OU=Teste,O=ICP-Brasil,C=BR");
        root = sign(new JcaX509v3CertificateBuilder(rootName, serial(), Date.from(now.minus(Duration.ofDays(1))),
                        Date.from(now.plus(Duration.ofDays(3650))), rootName, rootKeys.getPublic())
                        .addExtension(Extension.basicConstraints, true, new BasicConstraints(true))
                        .addExtension(Extension.keyUsage, true, new KeyUsage(KeyUsage.keyCertSign | KeyUsage.cRLSign))
                        .addExtension(Extension.subjectKeyIdentifier, false, new JcaX509ExtensionUtils().createSubjectKeyIdentifier(rootKeys.getPublic())),
                rootKeys.getPrivate());
        intermediate = sign(new JcaX509v3CertificateBuilder(root, serial(), Date.from(now.minus(Duration.ofDays(1))),
                        Date.from(now.plus(Duration.ofDays(1825))), intermediateName, intermediateKeys.getPublic())
                        .addExtension(Extension.basicConstraints, true, new BasicConstraints(0))
                        .addExtension(Extension.keyUsage, true, new KeyUsage(KeyUsage.keyCertSign | KeyUsage.cRLSign))
                        .addExtension(Extension.authorityKeyIdentifier, false, new JcaX509ExtensionUtils().createAuthorityKeyIdentifier(root))
                        .addExtension(Extension.subjectKeyIdentifier, false, new JcaX509ExtensionUtils().createSubjectKeyIdentifier(intermediateKeys.getPublic()))
                        .addExtension(Extension.cRLDistributionPoints, false, distributionPoint("root.crl")),
                rootKeys.getPrivate());
    }

    public static DevPki generate(String crlBaseUrl) {
        try {
            return new DevPki(crlBaseUrl);
        } catch (Exception e) {
            throw new IllegalStateException(e);
        }
    }

    public List<X509Certificate> anchors() {
        return List.of(root, intermediate);
    }

    /** e-CPF de teste: CN "NOME:CPF", otherName ICP PF com nascimento e CPF, digitalSignature + nonRepudiation. */
    public Issued issue(String cpf, String name, LocalDate birthDate, Instant notBefore, Instant notAfter) {
        try {
            KeyPair keys = keys();
            String pf = birthDate.format(DateTimeFormatter.ofPattern("ddMMyyyy")) + cpf + "0".repeat(11) + "0".repeat(15) + "0".repeat(6);
            GeneralName otherName = new GeneralName(GeneralName.otherName,
                    new DERSequence(new ASN1EncodableVector() {{
                        add(ICP_PF);
                        add(new DERTaggedObject(true, 0, new DEROctetString(pf.getBytes(StandardCharsets.US_ASCII))));
                    }}));
            X509v3CertificateBuilder builder = new JcaX509v3CertificateBuilder(intermediate, serial(), Date.from(notBefore),
                    Date.from(notAfter), new X500Name("CN=" + name + ":" + cpf + ",OU=Teste,O=ICP-Brasil,C=BR"), keys.getPublic())
                    .addExtension(Extension.basicConstraints, true, new BasicConstraints(false))
                    .addExtension(Extension.keyUsage, true, new KeyUsage(KeyUsage.digitalSignature | KeyUsage.nonRepudiation))
                    .addExtension(Extension.subjectAlternativeName, false, new GeneralNames(otherName))
                    .addExtension(Extension.authorityKeyIdentifier, false, new JcaX509ExtensionUtils().createAuthorityKeyIdentifier(intermediate))
                    .addExtension(Extension.cRLDistributionPoints, false, distributionPoint("intermediate.crl"));
            return new Issued(sign(builder, intermediateKeys.getPrivate()), keys.getPrivate());
        } catch (Exception e) {
            throw new IllegalStateException(e);
        }
    }

    public synchronized void revoke(BigInteger serial, Instant when) {
        revoked.put(serial, when);
    }

    /** "root.crl" (emitida pela raiz) ou "intermediate.crl" (pela intermediária, com as revogações). */
    public synchronized byte[] crl(String name) {
        try {
            boolean ofRoot = "root.crl".equals(name);
            X509Certificate issuer = ofRoot ? root : intermediate;
            Instant now = Instant.now();
            X509v2CRLBuilder builder = new X509v2CRLBuilder(new JcaX509CertificateHolder(issuer).getSubject(), Date.from(now.minusSeconds(60)));
            builder.setNextUpdate(Date.from(now.plus(Duration.ofDays(3650)))); // AC de teste: LCR longa, para o volume de dev não vencer
            if (!ofRoot) {
                revoked.forEach((serial, when) -> builder.addCRLEntry(serial, Date.from(when), CRLReason.keyCompromise));
            }
            X509CRLHolder holder = builder.build(new JcaContentSignerBuilder("SHA256withRSA")
                    .build(ofRoot ? rootKeys.getPrivate() : intermediateKeys.getPrivate()));
            return holder.getEncoded();
        } catch (Exception e) {
            throw new IllegalStateException(e);
        }
    }

    /** Escreve anchors.pem, root.pem, intermediate.pem, intermediate.key.pem e as duas LCRs. */
    public void writeTo(Path dir) throws IOException {
        Files.createDirectories(dir);
        Files.writeString(dir.resolve("anchors.pem"), pem(root) + pem(intermediate));
        Files.writeString(dir.resolve("root.pem"), pem(root));
        Files.writeString(dir.resolve("intermediate.pem"), pem(intermediate));
        Files.writeString(dir.resolve("intermediate.key.pem"), pem(intermediateKeys.getPrivate()));
        Files.write(dir.resolve("root.crl"), crl("root.crl"));
        Files.write(dir.resolve("intermediate.crl"), crl("intermediate.crl"));
    }

    /** Uso: DevPki <dir> <crlBaseUrl>. Não sobrescreve: com anchors.pem no diretório, não faz nada. */
    public static void main(String[] args) throws IOException {
        if (args.length != 2) {
            System.err.println("uso: DevPki <dir> <crlBaseUrl>");
            System.exit(2);
        }
        Path dir = Path.of(args[0]);
        if (Files.exists(dir.resolve("anchors.pem"))) return;
        generate(args[1]).writeTo(dir);
    }

    private CRLDistPoint distributionPoint(String file) {
        DistributionPoint point = new DistributionPoint(new DistributionPointName(
                new GeneralNames(new GeneralName(GeneralName.uniformResourceIdentifier, crlBaseUrl + file))), null, null);
        return new CRLDistPoint(new DistributionPoint[] {point});
    }

    private static KeyPair keys() throws Exception {
        KeyPairGenerator generator = KeyPairGenerator.getInstance("RSA");
        generator.initialize(2048, RANDOM);
        return generator.generateKeyPair();
    }

    private static BigInteger serial() {
        return new BigInteger(64, RANDOM).abs().add(BigInteger.ONE);
    }

    private static X509Certificate sign(X509v3CertificateBuilder builder, PrivateKey key) throws Exception {
        return new JcaX509CertificateConverter().getCertificate(builder.build(new JcaContentSignerBuilder("SHA256withRSA").build(key)));
    }

    private static String pem(Object object) throws IOException {
        StringWriter out = new StringWriter();
        try (JcaPEMWriter writer = new JcaPEMWriter(out)) {
            writer.writeObject(object instanceof PrivateKey key ? new org.bouncycastle.openssl.jcajce.JcaPKCS8Generator(key, null) : object);
        }
        return out.toString();
    }
}
```

- [ ] **Step 4: A cadeia (cópia da prova)**

```bash
cp .claude/signer-proof/src/main/java/rotasaude/signer/engine/Chain.java apps/signer/.claude/mod19b/src/main/java/rotasaude/signer/engine/
```

- [ ] **Step 5: Registre as âncoras por configuração no Demoiselle**

A classe `ConfiguredAnchorsProviderCA` existe desde a Task 1; o `CAManager` só a enxerga pelo `ServiceLoader`. Crie `src/main/resources/META-INF/services/org.demoiselle.signer.core.ca.provider.ProviderCA`:

```
rotasaude.signer.trust.ConfiguredAnchorsProviderCA
```

- [ ] **Step 6: Rode e veja passar**

Run: `(cd apps/signer/.claude/mod19b && bin/mvn test)`
Expected: `Tests run: 11, Failures: 0, Errors: 0` (7 da Task 1 + 4 de `DevPkiTest`).

- [ ] **Step 7: Commit**

```bash
W=apps/signer/.claude/mod19b
/opt/homebrew/bin/git -C $W add src/main/java/rotasaude/signer/devpki/DevPki.java src/main/java/rotasaude/signer/engine/Chain.java \
  src/main/resources/META-INF/services/org.demoiselle.signer.core.ca.provider.ProviderCA \
  src/test/java/rotasaude/signer/support/TestPki.java src/test/java/rotasaude/signer/devpki/DevPkiTest.java
/opt/homebrew/bin/git -C $W status --short
/opt/homebrew/bin/git -C $W commit -m "feat: add ICP-format test authority and configured trust anchors

DevPki issues a root, an intermediate and natural-person certificates
with the ICP-Brasil otherName and HTTP CRL distribution points, for the
JUnit suite and for the dev compose. Extra anchors come only from
configuration and resolve through Demoiselle's CAManager.

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 3: Motor de assinatura — CAdES destacado e PAdES AD-RB, ida e volta e adulteração

**Files:**
- Create (cópias da prova): `src/main/java/rotasaude/signer/engine/{Policies,SignedAttributes,CmsAssembler,PdfPlaceholder,TrackingCrlRepository,ValidationMaterial,Verification,SignatureVerifier,SignatureEngine}.java`, `src/main/resources/META-INF/services/org.demoiselle.signer.core.repository.CRLRepository`
- Test: `src/test/java/rotasaude/signer/engine/CadesRoundTripTest.java`, `src/test/java/rotasaude/signer/engine/PadesRoundTripTest.java`

**Interfaces:**
- Consumes: `Kind`, `PreparedSignature`, `SignerFailure` (Task 1); `Chain`, `DevPki`, `TestPki` (Task 2).
- Produces:
  - `rotasaude.signer.engine.SignatureEngine(Clock)`: `PreparedSignature prepare(Kind, byte[] document, byte[] certificateDer)` (422 certificado não RSA, fora da validade, sem cadeia ou revogado; 400 PDF ilegível; 500 política fora da vigência); `Assembled assemble(PreparedSignature, byte[] rawSignature)` → `record Assembled(byte[] signature, byte[] validationMaterial)` (422 `invalid_signature_value`); `Verification verify(Kind, byte[] document, byte[] signature)` (400 se CAdES sem documento).
  - `rotasaude.signer.engine.Verification(String status, String signerCpf, String signerName, String policyOid, Instant signedAt, List<String> reasons)`; `VALID`, `INVALID`, `INDETERMINATE`.
  - Motivos: `malformed_signature`, `signature_mismatch`, `untrusted_chain`, `policy_mismatch`, `signing_time_missing`, `certificate_expired`, `key_usage`, `certificate_revoked`, `pdf_modified_after_signing` (→ `invalid`); `revocation_unavailable`, `certificate_revoked_after_signing` (sozinhos → `indeterminate`); demais `ValidationMessageCode` do Demoiselle em minúsculas (→ `invalid`).
  - `Policies.oid(Kind)`, `Policies.acceptedOids(Kind)`, `Policies.SIGNATURE_ALGORITHM = "SHA256withRSA"`; `PdfPlaceholder.SIGNATURE_SIZE = 24_576`; `TrackingCrlRepository.lastSuccess() -> Instant|null`.
  - `validation_material`: CMS `SignedData` degenerado (sem signatário) com a cadeia e as LCRs do ato.

- [ ] **Step 1: Escreva os testes que falham**

`src/test/java/rotasaude/signer/engine/CadesRoundTripTest.java`:

```java
package rotasaude.signer.engine;

import static org.junit.jupiter.api.Assertions.assertArrayEquals;
import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertThrows;
import static org.junit.jupiter.api.Assertions.assertTrue;

import java.nio.charset.StandardCharsets;
import java.security.KeyPairGenerator;
import java.time.Clock;
import java.time.Duration;
import java.time.Instant;
import java.util.List;
import org.junit.jupiter.api.AfterEach;
import org.junit.jupiter.api.Test;
import rotasaude.signer.SignerFailure;
import rotasaude.signer.devpki.DevPki;
import rotasaude.signer.support.TestPki;

class CadesRoundTripTest {

    private static final TestPki PKI = TestPki.get();
    private static final byte[] JSON = "{\"schema\":\"rotasaude.consultation.v1\",\"text\":\"Glicemia 245\"}".getBytes(StandardCharsets.UTF_8);

    private final SignatureEngine engine = new SignatureEngine(Clock.systemUTC());

    @AfterEach
    void crlBack() {
        PKI.crlAvailable(true);
    }

    private byte[] sign(DevPki.Issued professional, byte[] document) throws Exception {
        PreparedSignature prepared = engine.prepare(Kind.CADES, document, professional.certificate().getEncoded());
        byte[] raw = TestPki.rawSign(professional.privateKey(), prepared.signedAttributesDer());
        return engine.assemble(prepared, raw).signature();
    }

    @Test
    void signsAndVerifiesDetachedCadesAdRb() throws Exception {
        Instant before = Instant.now().minusSeconds(1);
        byte[] p7s = sign(PKI.professional(), JSON);

        Verification result = engine.verify(Kind.CADES, JSON, p7s);

        assertEquals(List.of(), result.reasons());
        assertEquals(Verification.VALID, result.status());
        assertEquals(TestPki.CPF, result.signerCpf());
        assertEquals(TestPki.NAME, result.signerName());
        assertEquals(Policies.oid(Kind.CADES), result.policyOid());
        assertTrue(!result.signedAt().isBefore(before.minusSeconds(1)) && !result.signedAt().isAfter(Instant.now()));
    }

    @Test
    void toBeSignedIsTheSha256OfTheSignedAttributes() throws Exception {
        PreparedSignature prepared = engine.prepare(Kind.CADES, JSON, PKI.professional().certificate().getEncoded());
        assertArrayEquals(java.security.MessageDigest.getInstance("SHA-256").digest(prepared.signedAttributesDer()),
                prepared.toBeSignedSha256());
    }

    @Test
    void alteredDocumentIsInvalid() throws Exception {
        byte[] p7s = sign(PKI.professional(), JSON);
        byte[] altered = JSON.clone();
        altered[altered.length - 3] ^= 0x01;

        Verification result = engine.verify(Kind.CADES, altered, p7s);

        assertEquals(Verification.INVALID, result.status());
        assertTrue(result.reasons().contains("signature_mismatch"), result.reasons().toString());
    }

    @Test
    void alteredSignatureValueIsInvalid() throws Exception {
        byte[] p7s = sign(PKI.professional(), JSON);
        byte[] altered = p7s.clone();
        altered[altered.length - 10] ^= 0x01; // dentro do valor RSA, no fim do SignerInfo

        Verification result = engine.verify(Kind.CADES, JSON, altered);

        assertEquals(Verification.INVALID, result.status());
    }

    @Test
    void rawFromAnotherKeyIsRefusedAtAssemble() throws Exception {
        DevPki.Issued professional = PKI.professional();
        PreparedSignature prepared = engine.prepare(Kind.CADES, JSON, professional.certificate().getEncoded());
        KeyPairGenerator generator = KeyPairGenerator.getInstance("RSA");
        generator.initialize(2048);
        byte[] foreign = TestPki.rawSign(generator.generateKeyPair().getPrivate(), prepared.signedAttributesDer());

        SignerFailure failure = assertThrows(SignerFailure.class, () -> engine.assemble(prepared, foreign));
        assertEquals(SignerFailure.Code.INVALID_SIGNATURE_VALUE, failure.code);
    }

    @Test
    void expiredCertificateIsRefusedAtPrepare() throws Exception {
        byte[] der = PKI.expiredProfessional().certificate().getEncoded();
        SignerFailure failure = assertThrows(SignerFailure.class, () -> engine.prepare(Kind.CADES, JSON, der));
        assertEquals(SignerFailure.Code.INVALID_CERTIFICATE, failure.code);
    }

    @Test
    void certificateOutsideTheTrustAnchorsIsRefusedAtPrepare() throws Exception {
        byte[] der = TestPki.untrustedProfessional().certificate().getEncoded();
        SignerFailure failure = assertThrows(SignerFailure.class, () -> engine.prepare(Kind.CADES, JSON, der));
        assertEquals(SignerFailure.Code.INVALID_CERTIFICATE, failure.code);
    }

    @Test
    void certificateRevokedBeforeSigningIsInvalid() throws Exception {
        DevPki.Issued professional = PKI.professional();
        byte[] p7s = sign(professional, JSON);
        PKI.pki.revoke(professional.certificate().getSerialNumber(), Instant.now().minus(Duration.ofHours(1)));

        Verification result = engine.verify(Kind.CADES, JSON, p7s);

        assertEquals(Verification.INVALID, result.status(), result.reasons() + " " + result.signedAt());
        assertTrue(result.reasons().contains("certificate_revoked"), result.reasons().toString());
    }

    @Test
    void certificateRevokedAfterSigningIsIndeterminate() throws Exception {
        DevPki.Issued professional = PKI.professional();
        byte[] p7s = sign(professional, JSON);
        PKI.pki.revoke(professional.certificate().getSerialNumber(), Instant.now().plus(Duration.ofMinutes(1)));

        Verification result = engine.verify(Kind.CADES, JSON, p7s);

        assertEquals(Verification.INDETERMINATE, result.status());
        assertEquals(List.of("certificate_revoked_after_signing"), result.reasons());
    }

    @Test
    void unreachableCrlIsIndeterminate() throws Exception {
        byte[] p7s = sign(PKI.professional(), JSON);
        PKI.crlAvailable(false);

        Verification result = engine.verify(Kind.CADES, JSON, p7s);

        assertEquals(Verification.INDETERMINATE, result.status());
        assertEquals(List.of("revocation_unavailable"), result.reasons());
    }

    @Test
    void garbageSignatureIsMalformed() {
        Verification result = engine.verify(Kind.CADES, JSON, "not a cms".getBytes(StandardCharsets.UTF_8));
        assertEquals(Verification.INVALID, result.status());
        assertEquals(List.of("malformed_signature"), result.reasons());
    }

    @Test
    void cadesWithoutDocumentIsAnInvalidRequest() throws Exception {
        byte[] p7s = sign(PKI.professional(), JSON);
        SignerFailure failure = assertThrows(SignerFailure.class, () -> engine.verify(Kind.CADES, null, p7s));
        assertEquals(SignerFailure.Code.INVALID_REQUEST, failure.code);
    }

    @Test
    void validationMaterialCarriesTheChainAndTheCrls() throws Exception {
        DevPki.Issued professional = PKI.professional();
        PreparedSignature prepared = engine.prepare(Kind.CADES, JSON, professional.certificate().getEncoded());
        byte[] material = engine.assemble(prepared, TestPki.rawSign(professional.privateKey(), prepared.signedAttributesDer()))
                .validationMaterial();

        org.bouncycastle.cms.CMSSignedData bundle = new org.bouncycastle.cms.CMSSignedData(material);
        assertEquals(3, bundle.getCertificates().getMatches(null).size());
        assertEquals(2, bundle.getCRLs().getMatches(null).size());
        assertEquals(0, bundle.getSignerInfos().size());
    }
}
```

`src/test/java/rotasaude/signer/engine/PadesRoundTripTest.java`:

```java
package rotasaude.signer.engine;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertThrows;
import static org.junit.jupiter.api.Assertions.assertTrue;

import java.nio.charset.StandardCharsets;
import java.time.Clock;
import java.util.Arrays;
import java.util.List;
import org.apache.pdfbox.Loader;
import org.apache.pdfbox.pdmodel.PDDocument;
import org.apache.pdfbox.pdmodel.interactive.digitalsignature.PDSignature;
import org.junit.jupiter.api.Test;
import rotasaude.signer.SignerFailure;
import rotasaude.signer.devpki.DevPki;
import rotasaude.signer.support.TestPki;

class PadesRoundTripTest {

    private static final TestPki PKI = TestPki.get();

    private final SignatureEngine engine = new SignatureEngine(Clock.systemUTC());

    private byte[] signedPdf() throws Exception {
        DevPki.Issued professional = PKI.professional();
        PreparedSignature prepared = engine.prepare(Kind.PADES, TestPki.samplePdf("Consulta de teste"), professional.certificate().getEncoded());
        return engine.assemble(prepared, TestPki.rawSign(professional.privateKey(), prepared.signedAttributesDer())).signature();
    }

    @Test
    void signsAndVerifiesPadesAdRb() throws Exception {
        byte[] pdf = signedPdf();

        Verification result = engine.verify(Kind.PADES, null, pdf);

        assertEquals(List.of(), result.reasons());
        assertEquals(Verification.VALID, result.status());
        assertEquals(TestPki.CPF, result.signerCpf());
        assertEquals(Policies.oid(Kind.PADES), result.policyOid());
        try (PDDocument document = Loader.loadPDF(pdf)) {
            PDSignature signature = document.getLastSignatureDictionary();
            assertEquals(PDSignature.SUBFILTER_ETSI_CADES_DETACHED.getName(), signature.getSubFilter());
            assertEquals(signature.getSignDate().toInstant(), result.signedAt());
        }
    }

    @Test
    void bytesAppendedAfterSigningAreInvalid() throws Exception {
        byte[] pdf = signedPdf();
        byte[] appended = Arrays.copyOf(pdf, pdf.length + 12);
        System.arraycopy("\n%tampered\n".getBytes(StandardCharsets.US_ASCII), 0, appended, pdf.length, 11);

        Verification result = engine.verify(Kind.PADES, null, appended);

        assertEquals(Verification.INVALID, result.status());
        assertTrue(result.reasons().contains("pdf_modified_after_signing"), result.reasons().toString());
    }

    @Test
    void alteredSignedByteIsInvalid() throws Exception {
        byte[] pdf = signedPdf();
        byte[] altered = pdf.clone();
        altered[11] ^= 0x01; // comentário binário da 2ª linha do cabeçalho: o PDF continua legível

        Verification result = engine.verify(Kind.PADES, null, altered);

        assertEquals(Verification.INVALID, result.status());
        assertTrue(result.reasons().contains("signature_mismatch"), result.reasons().toString());
    }

    @Test
    void notAPdfIsAnInvalidRequestAtPrepare() throws Exception {
        byte[] der = PKI.professional().certificate().getEncoded();
        SignerFailure failure = assertThrows(SignerFailure.class,
                () -> engine.prepare(Kind.PADES, "not a pdf".getBytes(StandardCharsets.US_ASCII), der));
        assertEquals(SignerFailure.Code.INVALID_REQUEST, failure.code);
    }

    @Test
    void unsignedPdfIsMalformed() {
        Verification result = engine.verify(Kind.PADES, null, TestPki.samplePdf("sem assinatura"));
        assertEquals(List.of("malformed_signature"), result.reasons());
    }
}
```

- [ ] **Step 2: Rode e veja falhar**

Run: `(cd apps/signer/.claude/mod19b && bin/mvn -q test)`
Expected: FAIL de compilação — `cannot find symbol ... SignatureEngine`, `Verification`, `Policies`.

- [ ] **Step 3: Copie o motor da prova**

```bash
PR=.claude/signer-proof/src/main; W=apps/signer/.claude/mod19b/src/main
for f in Policies SignedAttributes CmsAssembler PdfPlaceholder TrackingCrlRepository ValidationMaterial Verification SignatureVerifier SignatureEngine; do
  cp $PR/java/rotasaude/signer/engine/$f.java $W/java/rotasaude/signer/engine/
done
mkdir -p $W/resources/META-INF/services
cp $PR/resources/META-INF/services/org.demoiselle.signer.core.repository.CRLRepository $W/resources/META-INF/services/
for f in SignerFailure.java engine/Kind.java engine/PreparedSignature.java engine/Chain.java; do
  diff -q $PR/java/rotasaude/signer/$f $W/java/rotasaude/signer/$f && echo "igual $f"
done
```
Expected: `igual` para os quatro (o que veio antes da prova não divergiu). O conteúdo de cada arquivo copiado está na Task 0, Step 3 — com os ajustes que a prova tiver pedido (registrados na pesquisa). Se a prova mudou um teste esperado deste plano (por exemplo, o motivo de uma recusa), ajuste o teste aqui e registre no commit.

- [ ] **Step 4: Rode e veja passar**

Run: `(cd apps/signer/.claude/mod19b && bin/mvn test)`
Expected: `Tests run: 29, Failures: 0, Errors: 0` (11 anteriores + 13 de `CadesRoundTripTest` + 5 de `PadesRoundTripTest`). `CadesRoundTripTest` leva ~6 s (a LCR indisponível espera o timeout de 2 s duas vezes).

- [ ] **Step 5: Commit**

```bash
W=apps/signer/.claude/mod19b
/opt/homebrew/bin/git -C $W add src/main/java/rotasaude/signer/engine/Policies.java src/main/java/rotasaude/signer/engine/SignedAttributes.java \
  src/main/java/rotasaude/signer/engine/CmsAssembler.java src/main/java/rotasaude/signer/engine/PdfPlaceholder.java \
  src/main/java/rotasaude/signer/engine/TrackingCrlRepository.java src/main/java/rotasaude/signer/engine/ValidationMaterial.java \
  src/main/java/rotasaude/signer/engine/Verification.java src/main/java/rotasaude/signer/engine/SignatureVerifier.java \
  src/main/java/rotasaude/signer/engine/SignatureEngine.java \
  src/main/resources/META-INF/services/org.demoiselle.signer.core.repository.CRLRepository \
  src/test/java/rotasaude/signer/engine/CadesRoundTripTest.java src/test/java/rotasaude/signer/engine/PadesRoundTripTest.java
/opt/homebrew/bin/git -C $W status --short
/opt/homebrew/bin/git -C $W commit -m "feat: prepare, assemble and verify AD-RB CAdES and PAdES with an external signature

Demoiselle builds the policy's signed attributes without a private key;
BouncyCastle wraps the RAW value returned by the PSC into the SignerInfo,
and PDFBox reserves and fills the PDF signature. Verification checks
policy, chain, signing time, key usage and revocation at signing time,
with reasons; revocation after signing or an unreachable CRL is
indeterminate, never valid.

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 4: HTTP do contrato §9, boot e higiene de log

**Files:**
- Create: `src/main/java/rotasaude/signer/http/SignerServer.java`, `src/main/java/rotasaude/signer/App.java`, `src/main/resources/log4j2.xml` (cópia da prova)
- Test: `src/test/java/rotasaude/signer/http/SignerServerTest.java`

**Interfaces:**
- Consumes: `Config` (Task 1), `PreparedStateCodec` (Task 1), `SignatureEngine`, `Verification`, `TrackingCrlRepository` (Task 3), `TestPki` (Task 2).
- Produces:
  - `rotasaude.signer.http.SignerServer.start(Config, SignatureEngine) -> com.sun.net.httpserver.HttpServer` (já iniciado).
  - Rotas do contrato §9; rota desconhecida → 404 `{ "error": "not_found" }`; corpo acima de 48 MB, JSON inválido, campo ausente ou base64 inválido → 400 `invalid_request`; `signed_at` ISO 8601 UTC (`...Z`) ou `null`.
  - Com `SIGNER_DEV_PKI_DIR`: `GET /dev-pki/root.crl` e `/dev-pki/intermediate.crl` sem token, `application/pkix-crl`; qualquer outro nome → 404.
  - `rotasaude.signer.App.main` (sai com 1 e mensagem sem segredo se a configuração for recusada).

- [ ] **Step 1: Escreva o teste que falha**

`src/test/java/rotasaude/signer/http/SignerServerTest.java`:

```java
package rotasaude.signer.http;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertFalse;
import static org.junit.jupiter.api.Assertions.assertTrue;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import com.sun.net.httpserver.HttpServer;
import java.io.ByteArrayOutputStream;
import java.io.PrintStream;
import java.net.ServerSocket;
import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;
import java.time.Clock;
import java.util.Base64;
import java.util.Map;
import org.junit.jupiter.api.AfterAll;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;
import rotasaude.signer.Config;
import rotasaude.signer.devpki.DevPki;
import rotasaude.signer.engine.PreparedStateCodec;
import rotasaude.signer.engine.SignatureEngine;
import rotasaude.signer.support.TestPki;

class SignerServerTest {

    private static final String TOKEN = "dev-signer-token-0123456789abcdef";
    private static final String CLINICAL_MARKER = "Glicemia capilar 245 mg/dL";
    private static final ObjectMapper JSON = new ObjectMapper();
    private static final TestPki PKI = TestPki.get();
    private static final HttpClient CLIENT = HttpClient.newHttpClient();
    private static final B64 B = new B64();

    private static HttpServer server;
    private static String base;
    private static Path devPkiDir;

    @BeforeAll
    static void start() throws Exception {
        int port;
        try (ServerSocket socket = new ServerSocket(0)) {
            port = socket.getLocalPort();
        }
        devPkiDir = Files.createTempDirectory("dev-pki");
        PKI.pki.writeTo(devPkiDir);
        server = SignerServer.start(Config.fromEnv(Map.of("SIGNER_TOKEN", TOKEN, "SIGNER_PORT", String.valueOf(port),
                "SIGNER_DEV_PKI_DIR", devPkiDir.toString())), new SignatureEngine(Clock.systemUTC()));
        base = "http://127.0.0.1:" + port;
    }

    @AfterAll
    static void stop() {
        server.stop(0);
    }

    private static HttpResponse<String> call(String method, String path, String token, String body) throws Exception {
        HttpRequest.Builder request = HttpRequest.newBuilder(URI.create(base + path))
                .method(method, body == null ? HttpRequest.BodyPublishers.noBody() : HttpRequest.BodyPublishers.ofString(body))
                .header("Content-Type", "application/json");
        if (token != null) request.header("Authorization", "Bearer " + token);
        return CLIENT.send(request.build(), HttpResponse.BodyHandlers.ofString());
    }

    private static JsonNode post(String path, ObjectNode body, int expectedStatus) throws Exception {
        HttpResponse<String> response = call("POST", path, TOKEN, body.toString());
        assertEquals(expectedStatus, response.statusCode(), response.body());
        return JSON.readTree(response.body());
    }

    private static ObjectNode prepareBody(String kind, byte[] document, DevPki.Issued professional) throws Exception {
        return JSON.createObjectNode().put("kind", kind).put("policy", "AD-RB")
                .put("document_base64", B.e(document)).put("certificate_der_base64", B.e(professional.certificate().getEncoded()));
    }

    private static JsonNode signOverHttp(String kind, byte[] document, DevPki.Issued professional) throws Exception {
        JsonNode prepared = post("/prepare", prepareBody(kind, document, professional), 200);
        String state = prepared.get("prepared_state").asText();
        byte[] attributes = new PreparedStateCodec(TOKEN).decode(state).signedAttributesDer();
        assertEquals(B.e(java.security.MessageDigest.getInstance("SHA-256").digest(attributes)),
                prepared.get("to_be_signed_sha256_base64").asText());
        return post("/assemble", JSON.createObjectNode().put("kind", kind).put("prepared_state", state)
                .put("signature_value_base64", B.e(TestPki.rawSign(professional.privateKey(), attributes))), 200);
    }

    @Test
    void refusesMissingAndWrongToken() throws Exception {
        assertEquals(401, call("GET", "/health", null, null).statusCode());
        HttpResponse<String> wrong = call("GET", "/health", "x".repeat(40), null);
        assertEquals(401, wrong.statusCode());
        assertEquals("{\"error\":\"unauthorized\"}", wrong.body());
    }

    @Test
    void healthReportsVersionAndCrlTime() throws Exception {
        HttpResponse<String> response = call("GET", "/health", TOKEN, null);
        assertEquals(200, response.statusCode());
        JsonNode body = JSON.readTree(response.body());
        assertEquals("dev", body.get("version").asText());
        assertTrue(body.has("crl_updated_at"));
    }

    @Test
    void cadesRoundTripOverHttp() throws Exception {
        byte[] document = ("{\"text\":\"" + CLINICAL_MARKER + "\"}").getBytes(StandardCharsets.UTF_8);
        JsonNode assembled = signOverHttp("cades", document, PKI.professional());
        JsonNode verified = post("/verify", JSON.createObjectNode().put("kind", "cades").put("document_base64", B.e(document))
                .put("signature_base64", assembled.get("signature_base64").asText()), 200);

        assertEquals("valid", verified.get("status").asText(), verified.toString());
        assertEquals(TestPki.CPF, verified.get("signer_cpf").asText());
        assertEquals(TestPki.NAME, verified.get("signer_name").asText());
        assertTrue(verified.get("signed_at").asText().endsWith("Z"));
        assertEquals(0, verified.get("reasons").size());
        assertFalse(assembled.get("validation_material_base64").asText().isEmpty());
    }

    @Test
    void padesRoundTripOverHttpWithoutDocument() throws Exception {
        JsonNode assembled = signOverHttp("pades", TestPki.samplePdf("Consulta"), PKI.professional());
        JsonNode verified = post("/verify", JSON.createObjectNode().put("kind", "pades")
                .put("signature_base64", assembled.get("signature_base64").asText()), 200);
        assertEquals("valid", verified.get("status").asText(), verified.toString());
    }

    @Test
    void contractErrors() throws Exception {
        DevPki.Issued professional = PKI.professional();
        byte[] json = "{}".getBytes(StandardCharsets.UTF_8);
        assertEquals("invalid_request", post("/prepare", prepareBody("xades", json, professional), 400).get("error").asText());
        assertEquals("invalid_request", post("/prepare", prepareBody("cades", json, professional).put("policy", "AD-RT"), 400).get("error").asText());
        assertEquals("invalid_request", post("/prepare", prepareBody("cades", json, professional).put("document_base64", "%%%"), 400).get("error").asText());
        assertEquals("invalid_certificate", post("/prepare", prepareBody("cades", json, professional)
                .put("certificate_der_base64", B.e("not a certificate".getBytes(StandardCharsets.UTF_8))), 422).get("error").asText());
        assertEquals("invalid_certificate", post("/prepare", prepareBody("cades", json, TestPki.untrustedProfessional()), 422).get("error").asText());

        JsonNode prepared = post("/prepare", prepareBody("cades", json, professional), 200);
        String state = prepared.get("prepared_state").asText();
        assertEquals("invalid_request", post("/assemble", JSON.createObjectNode().put("kind", "pades").put("prepared_state", state)
                .put("signature_value_base64", B.e(new byte[256])), 400).get("error").asText());
        assertEquals("invalid_signature_value", post("/assemble", JSON.createObjectNode().put("kind", "cades").put("prepared_state", state)
                .put("signature_value_base64", B.e(new byte[256])), 422).get("error").asText());
        assertEquals("invalid_request", post("/verify", JSON.createObjectNode().put("kind", "cades")
                .put("signature_base64", B.e(new byte[] {1})), 400).get("error").asText());
        assertEquals(400, call("POST", "/prepare", TOKEN, "not json").statusCode());
        assertEquals(404, call("GET", "/nothing", TOKEN, null).statusCode());
    }

    @Test
    void devPkiServesOnlyTheCrlsWithoutToken() throws Exception {
        assertEquals(200, call("GET", "/dev-pki/intermediate.crl", null, null).statusCode());
        assertEquals(200, call("GET", "/dev-pki/root.crl", null, null).statusCode());
        assertEquals(404, call("GET", "/dev-pki/intermediate.key.pem", null, null).statusCode());
        assertEquals(404, call("GET", "/dev-pki/../anchors.pem", null, null).statusCode());
    }

    @Test
    void logsNeverCarryTokenCpfNameOrDocument() throws Exception {
        PrintStream original = System.out;
        ByteArrayOutputStream captured = new ByteArrayOutputStream();
        System.setOut(new PrintStream(captured, true, StandardCharsets.UTF_8));
        try {
            byte[] document = ("{\"text\":\"" + CLINICAL_MARKER + "\"}").getBytes(StandardCharsets.UTF_8);
            JsonNode assembled = signOverHttp("cades", document, PKI.professional());
            post("/verify", JSON.createObjectNode().put("kind", "cades").put("document_base64", B.e(document))
                    .put("signature_base64", assembled.get("signature_base64").asText()), 200);
            post("/prepare", prepareBody("cades", document, TestPki.untrustedProfessional()), 422);
            signOverHttp("pades", TestPki.samplePdf(CLINICAL_MARKER), PKI.professional());
        } finally {
            System.setOut(original);
        }
        String log = captured.toString(StandardCharsets.UTF_8);
        assertTrue(log.contains("POST /prepare 200"), log);
        for (String secret : new String[] {TOKEN, TestPki.CPF, TestPki.NAME, CLINICAL_MARKER, "Glicemia"}) {
            assertFalse(log.contains(secret), "log contém " + secret + ":\n" + log);
        }
    }

    private static final class B64 {
        String e(byte[] bytes) {
            return Base64.getEncoder().encodeToString(bytes);
        }
    }
}
```

- [ ] **Step 2: Rode e veja falhar**

Run: `(cd apps/signer/.claude/mod19b && bin/mvn -q test)`
Expected: FAIL de compilação — `package rotasaude.signer.http does not exist`.

- [ ] **Step 3: Implemente o servidor e o boot**

`src/main/java/rotasaude/signer/http/SignerServer.java`:

```java
package rotasaude.signer.http;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ArrayNode;
import com.fasterxml.jackson.databind.node.ObjectNode;
import com.sun.net.httpserver.HttpExchange;
import com.sun.net.httpserver.HttpServer;
import java.io.IOException;
import java.io.InputStream;
import java.io.OutputStream;
import java.net.InetSocketAddress;
import java.nio.charset.StandardCharsets;
import java.nio.file.Files;
import java.nio.file.Path;
import java.security.MessageDigest;
import java.time.Instant;
import java.util.Base64;
import java.util.Set;
import java.util.concurrent.Executors;
import org.apache.logging.log4j.LogManager;
import org.apache.logging.log4j.Logger;
import rotasaude.signer.Config;
import rotasaude.signer.SignerFailure;
import rotasaude.signer.engine.Kind;
import rotasaude.signer.engine.PreparedSignature;
import rotasaude.signer.engine.PreparedStateCodec;
import rotasaude.signer.engine.SignatureEngine;
import rotasaude.signer.engine.TrackingCrlRepository;
import rotasaude.signer.engine.Verification;

/**
 * HTTP interno (contrato §9): JSON, Bearer compartilhado, corpo nunca logado.
 * O log de acesso tem só método, caminho, status e duração.
 */
public final class SignerServer {

    private static final Logger LOG = LogManager.getLogger(SignerServer.class);
    private static final ObjectMapper JSON = new ObjectMapper();
    private static final int MAX_BODY = 48 * 1024 * 1024;
    private static final Set<String> DEV_PKI_FILES = Set.of("root.crl", "intermediate.crl");

    private final Config config;
    private final SignatureEngine engine;
    private final PreparedStateCodec codec;
    private final byte[] expectedAuthorization;

    private SignerServer(Config config, SignatureEngine engine) {
        this.config = config;
        this.engine = engine;
        this.codec = new PreparedStateCodec(config.token());
        this.expectedAuthorization = ("Bearer " + config.token()).getBytes(StandardCharsets.UTF_8);
    }

    public static HttpServer start(Config config, SignatureEngine engine) throws IOException {
        SignerServer handler = new SignerServer(config, engine);
        HttpServer server = HttpServer.create(new InetSocketAddress(config.port()), 0);
        server.setExecutor(Executors.newVirtualThreadPerTaskExecutor());
        server.createContext("/", handler::handle);
        if (config.devPkiDir() != null) server.createContext("/dev-pki/", handler::devPki);
        server.start();
        return server;
    }

    private void handle(HttpExchange exchange) throws IOException {
        long started = System.nanoTime();
        String route = exchange.getRequestMethod() + " " + exchange.getRequestURI().getPath();
        int status = 500;
        try {
            if (!authorized(exchange)) throw new SignerFailure(SignerFailure.Code.UNAUTHORIZED);
            ObjectNode body = switch (route) {
                case "POST /prepare" -> prepare(read(exchange));
                case "POST /assemble" -> assemble(read(exchange));
                case "POST /verify" -> verify(read(exchange));
                case "GET /health" -> health();
                default -> null;
            };
            if (body == null) {
                status = 404;
                send(exchange, status, error("not_found"));
            } else {
                status = 200;
                send(exchange, status, body);
            }
        } catch (SignerFailure failure) {
            status = failure.code.status;
            send(exchange, status, error(failure.code.wire));
        } catch (Exception unexpected) {
            status = 500;
            LOG.error("{} failed: {}", route, unexpected.getClass().getSimpleName());
            send(exchange, status, error(SignerFailure.Code.INTERNAL.wire));
        } finally {
            LOG.info("{} {} {}ms", route, status, (System.nanoTime() - started) / 1_000_000);
        }
    }

    private ObjectNode prepare(JsonNode request) {
        Kind kind = Kind.parse(text(request, "kind"));
        if (!"AD-RB".equals(text(request, "policy"))) throw invalidRequest();
        PreparedSignature prepared = engine.prepare(kind, base64(request, "document_base64"), base64(request, "certificate_der_base64"));
        ObjectNode out = JSON.createObjectNode();
        out.put("to_be_signed_sha256_base64", Base64.getEncoder().encodeToString(prepared.toBeSignedSha256()));
        out.put("prepared_state", codec.encode(prepared));
        return out;
    }

    private ObjectNode assemble(JsonNode request) {
        Kind kind = Kind.parse(text(request, "kind"));
        PreparedSignature prepared = codec.decode(text(request, "prepared_state"));
        if (prepared.kind() != kind) throw invalidRequest();
        SignatureEngine.Assembled assembled = engine.assemble(prepared, base64(request, "signature_value_base64"));
        ObjectNode out = JSON.createObjectNode();
        out.put("signature_base64", Base64.getEncoder().encodeToString(assembled.signature()));
        out.put("validation_material_base64", Base64.getEncoder().encodeToString(assembled.validationMaterial()));
        return out;
    }

    private ObjectNode verify(JsonNode request) {
        Kind kind = Kind.parse(text(request, "kind"));
        JsonNode document = request.get("document_base64");
        byte[] documentBytes = document == null || document.isNull() ? null : base64(request, "document_base64");
        Verification result = engine.verify(kind, documentBytes, base64(request, "signature_base64"));
        ObjectNode out = JSON.createObjectNode();
        out.put("status", result.status());
        out.put("signer_cpf", result.signerCpf());
        out.put("signer_name", result.signerName());
        out.put("policy_oid", result.policyOid());
        out.put("signed_at", result.signedAt() == null ? null : result.signedAt().toString());
        ArrayNode reasons = out.putArray("reasons");
        result.reasons().forEach(reasons::add);
        return out;
    }

    private ObjectNode health() {
        Instant crl = TrackingCrlRepository.lastSuccess();
        ObjectNode out = JSON.createObjectNode();
        out.put("version", config.version());
        out.put("crl_updated_at", crl == null ? null : crl.toString());
        return out;
    }

    /** LCRs da AC de desenvolvimento, sem token (LCR é pública por natureza); nunca a chave. */
    private void devPki(HttpExchange exchange) throws IOException {
        String name = exchange.getRequestURI().getPath().substring("/dev-pki/".length());
        Path file = Path.of(config.devPkiDir(), name);
        if (!"GET".equals(exchange.getRequestMethod()) || !DEV_PKI_FILES.contains(name) || !Files.isRegularFile(file)) {
            send(exchange, 404, error("not_found"));
            return;
        }
        byte[] crl = Files.readAllBytes(file);
        exchange.getResponseHeaders().set("Content-Type", "application/pkix-crl");
        exchange.sendResponseHeaders(200, crl.length);
        try (OutputStream out = exchange.getResponseBody()) {
            out.write(crl);
        }
    }

    private boolean authorized(HttpExchange exchange) {
        String header = exchange.getRequestHeaders().getFirst("Authorization");
        return header != null && MessageDigest.isEqual(header.getBytes(StandardCharsets.UTF_8), expectedAuthorization);
    }

    private static JsonNode read(HttpExchange exchange) throws IOException {
        try (InputStream in = exchange.getRequestBody()) {
            byte[] raw = in.readNBytes(MAX_BODY + 1);
            if (raw.length > MAX_BODY) throw invalidRequest();
            JsonNode node = JSON.readTree(raw);
            if (node == null || !node.isObject()) throw invalidRequest();
            return node;
        } catch (com.fasterxml.jackson.core.JsonProcessingException e) {
            throw invalidRequest();
        }
    }

    private static String text(JsonNode request, String field) {
        JsonNode value = request.get(field);
        if (value == null || !value.isTextual()) throw invalidRequest();
        return value.asText();
    }

    private static byte[] base64(JsonNode request, String field) {
        try {
            byte[] decoded = Base64.getDecoder().decode(text(request, field));
            if (decoded.length == 0) throw invalidRequest();
            return decoded;
        } catch (IllegalArgumentException e) {
            throw invalidRequest();
        }
    }

    private static SignerFailure invalidRequest() {
        return new SignerFailure(SignerFailure.Code.INVALID_REQUEST);
    }

    private static ObjectNode error(String code) {
        return JSON.createObjectNode().put("error", code);
    }

    private static void send(HttpExchange exchange, int status, ObjectNode body) throws IOException {
        byte[] bytes = JSON.writeValueAsBytes(body);
        exchange.getResponseHeaders().set("Content-Type", "application/json");
        exchange.sendResponseHeaders(status, bytes.length);
        try (OutputStream out = exchange.getResponseBody()) {
            out.write(bytes);
        }
    }
}
```

`src/main/java/rotasaude/signer/App.java`:

```java
package rotasaude.signer;

import java.time.Clock;
import org.apache.logging.log4j.LogManager;
import rotasaude.signer.engine.SignatureEngine;
import rotasaude.signer.http.SignerServer;

public final class App {

    private App() {}

    public static void main(String[] args) throws Exception {
        Config config;
        try {
            config = Config.fromEnv(System.getenv());
        } catch (IllegalArgumentException e) {
            System.err.println("signer: " + e.getMessage());
            System.exit(1);
            return;
        }
        SignerServer.start(config, new SignatureEngine(Clock.systemUTC()));
        LogManager.getLogger(App.class).info("signer {} listening on {}", config.version(), config.port());
    }
}
```

O `log4j2.xml` da prova (Task 0, Step 3), sem mudança:

```bash
cp .claude/signer-proof/src/main/resources/log4j2.xml apps/signer/.claude/mod19b/src/main/resources/
```

- [ ] **Step 4: Rode e veja passar**

Run: `(cd apps/signer/.claude/mod19b && bin/mvn test)`
Expected: `Tests run: 36, Failures: 0, Errors: 0`, `BUILD SUCCESS`.

- [ ] **Step 5: Prove que o teste de higiene morde**

```bash
W=apps/signer/.claude/mod19b
sed -i '' 's#<Logger name="org.demoiselle" level="off" additivity="false"/>#<Logger name="org.demoiselle" level="debug"><AppenderRef ref="out"/></Logger>#' $W/src/main/resources/log4j2.xml
(cd $W && bin/mvn -q test -Dtest='SignerServerTest#logsNeverCarryTokenCpfNameOrDocument') 2>&1 | grep -m1 "log contém"
/opt/homebrew/bin/git -C $W checkout -- src/main/resources/log4j2.xml 2>/dev/null || cp .claude/signer-proof/src/main/resources/log4j2.xml $W/src/main/resources/
grep -c 'org.demoiselle" level="off"' $W/src/main/resources/log4j2.xml
```
Expected: `log contém 52998224725` (o Demoiselle em `debug` escreve o CPF do titular); depois da restauração, `1`. (O `checkout` falha porque o arquivo ainda não foi commitado; o `cp` restaura da prova.)

- [ ] **Step 6: Commit**

```bash
W=apps/signer/.claude/mod19b
/opt/homebrew/bin/git -C $W add src/main/java/rotasaude/signer/http/SignerServer.java src/main/java/rotasaude/signer/App.java \
  src/main/resources/log4j2.xml src/test/java/rotasaude/signer/http/SignerServerTest.java
/opt/homebrew/bin/git -C $W status --short
/opt/homebrew/bin/git -C $W commit -m "feat: expose prepare, assemble, verify and health over internal HTTP

JDK HTTP server with a shared bearer token on every route, contract error
codes only, an access log without bodies, and Demoiselle logging switched
off because it writes the holder's name and CPF. The dev PKI CRLs are
served without a token only when SIGNER_DEV_PKI_DIR is set.

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 5: Imagem, README, CI e o serviço no compose de dev

**Files:**
- Create: `Dockerfile`, `README.md`, `.github/workflows/ci.yml`
- Modify (raiz do monorepo, fora de git): `docker-compose.yml` (serviço `signer`, `x-api-env`, volumes de `api`/`worker`, volume `signer-dev-pki`)

**Interfaces:**
- Consumes: o jar e `target/lib/` do `pom.xml` (Task 1); `DevPki.main` (Task 2); `App` (Task 4).
- Produces (para o plano do `api`):
  - Serviço `signer` na rede do compose em `http://signer:8090`; no host, `127.0.0.1:${SIGNER_HOST_PORT:-8090}`.
  - Env do `api` e do `worker` (via `x-api-env`): `SIGNER_URL` (padrão `http://signer:8090`), `SIGNER_TOKEN` (padrão de dev `dev-signer-token-change-me-0123456789`, o mesmo do `signer`), `SIGNER_DEV_PKI_DIR=/signer-dev-pki`.
  - Volume `signer-dev-pki` montado no `signer` em `/dev-pki` e no `api`/`worker` em `/signer-dev-pki` (leitura e escrita: o PSC falso do `api` lê `intermediate.pem`/`intermediate.key.pem` e pode reescrever `intermediate.crl` para testar revogação).
  - Imagem `rota-saude/signer:dev`; produção: `SIGNER_ENV=production`, sem `SIGNER_EXTRA_TRUST_ANCHORS`/`SIGNER_DEV_PKI_DIR`.

- [ ] **Step 1: Dockerfile, README e CI**

`Dockerfile`:

```dockerfile
# signer — serviço interno de assinatura (Java 21 + Demoiselle Signer).
# Build com Maven no primeiro estágio; o runtime só tem a JRE, o jar e as dependências.
FROM maven:3.9.9-eclipse-temurin-21 AS build
WORKDIR /src
COPY pom.xml .
RUN mvn -B -q dependency:go-offline
COPY src ./src
RUN mvn -B -q -DskipTests package

FROM eclipse-temurin:21-jre
RUN useradd --system --uid 10001 --home-dir /app signer \
 && mkdir -p /app /dev-pki \
 && chown signer /dev-pki
WORKDIR /app
COPY --from=build /src/target/signer.jar /app/signer.jar
COPY --from=build /src/target/lib /app/lib
USER signer
ENV SIGNER_PORT=8090
EXPOSE 8090
ENTRYPOINT ["java", "-XX:MaxRAMPercentage=75", "-jar", "/app/signer.jar"]
```

`README.md`:

````markdown
# signer — assinatura ICP-Brasil do Rota Saúde

Serviço interno, sem estado e sem banco (ADR 0032, módulo 19b). O `api` conduz o
fluxo (OAuth com o PSC, sessões, fila); o `signer` **prepara**, **monta** e
**valida** assinaturas AD-RB:

- CAdES destacado sobre o JSON canônico (`contracts/clinical`);
- PAdES sobre o PDF da consulta (assinatura invisível; o rodapé NGS2 vem desenhado pelo `api`).

O hash é assinado fora, pelo certificado em nuvem do profissional (API de PSC do
ITI, DOC-ICP-17.01, `signature_format: RAW`). O documento nunca sai da
infraestrutura e o `signer` nunca grava documento.

## Como funciona

| Rota | Faz |
|---|---|
| `POST /prepare` | Monta os atributos assinados da política (Demoiselle, sem chave privada) e devolve o SHA-256 que vai ao PSC e um `prepared_state` opaco (selado com HMAC). Em PAdES, reserva a assinatura no PDF e grava a hora (entrada `M`). |
| `POST /assemble` | Confere a RAW contra o certificado (PKCS#1 v1.5 SHA-256 sobre os atributos), monta o CMS (BouncyCastle) e devolve o `.p7s` ou o PDF assinado, mais o material de validação (cadeia + LCRs do ato, CMS certs-only). |
| `POST /verify` | Valida no servidor (NGS2.03): criptografia e atributos (checker do Demoiselle), política AD-RB, cadeia até âncora confiável, hora da assinatura dentro da validade, keyUsage e revogação no instante da assinatura. `valid`, `invalid` ou `indeterminate`, com motivos. |
| `GET /health` | Versão e hora do último download de LCR bem-sucedido. |

Formas e erros: `docs/superpowers/plans/2026-10-08-module-19b-signature-contracts.md` §9
(repo `docs`). Toda rota exige `Authorization: Bearer $SIGNER_TOKEN`.

Políticas (padrões do Demoiselle Signer 4.6.2, lidas do `.der` da política no jar):
CAdES AD-RB v2.4 (`2.16.76.1.7.1.1.2.4`) e PAdES AD-RB v1.3 (`2.16.76.1.7.1.11.1.3`).
O `/verify` aceita todas as versões AD-RB do formato.

Motivos do `/verify`: `malformed_signature`, `signature_mismatch`, `untrusted_chain`,
`policy_mismatch`, `signing_time_missing`, `certificate_expired`, `key_usage`,
`certificate_revoked`, `pdf_modified_after_signing` (todos `invalid`);
`revocation_unavailable` e `certificate_revoked_after_signing` (sozinhos, `indeterminate`);
outros códigos do Demoiselle aparecem em minúsculas.

## Configuração

| Variável | Padrão | O que é |
|---|---|---|
| `SIGNER_TOKEN` | — (obrigatório, ≥ 32 caracteres) | Token compartilhado com o `api`/worker; também deriva a chave do `prepared_state`. |
| `SIGNER_PORT` | `8090` | Porta HTTP. |
| `SIGNER_ENV` | vazio | Variável do próprio Demoiselle: `production` desliga as cadeias de homologação da ICP-Brasil. Aqui, também proíbe as duas abaixo (o boot falha). |
| `SIGNER_EXTRA_TRUST_ANCHORS` | — | PEM com âncoras extras (AC de teste/dev). Nunca em produção. |
| `SIGNER_DEV_PKI_DIR` | — | Diretório da AC de desenvolvimento; liga `GET /dev-pki/{root.crl,intermediate.crl}` sem token. Nunca em produção. |

Rede de saída: só as LCRs das ACs (pontos de distribuição dos certificados).
Variáveis do Demoiselle que valem aqui: `SIGNER_CRL_CONNECTION_TIMEOUT`, `SIGNER_PROXY_*`,
`SIGNER_DISABLE_CHAIN_<CADEIA>`.

## Logs

Só o acesso: método, caminho, status e duração. Corpo de requisição, token, CPF,
nome e texto clínico nunca vão ao log. O Demoiselle fica **desligado** no
`log4j2.xml` porque escreve o DN do titular (nome e CPF). O teste
`SignerServerTest#logsNeverCarryTokenCpfNameOrDocument` quebra se isso mudar.

## Desenvolvimento

O host não precisa de Maven: `bin/mvn` roda o Maven 3.9 com JDK 21 num container.

```bash
bin/mvn test      # JUnit: AC de teste gerada no próprio teste (DevPki), LCR servida localmente
bin/mvn verify    # o mesmo da CI
docker build -t rota-saude/signer:dev .
```

No compose do monorepo o serviço chama `signer` (porta 8090; no host só em
`127.0.0.1:${SIGNER_HOST_PORT:-8090}`). Em dev, o container gera na primeira
subida uma AC de teste no formato ICP-Brasil no volume `signer-dev-pki`
(`anchors.pem`, `root.pem`, `intermediate.pem`, `intermediate.key.pem`,
`root.crl`, `intermediate.crl`) e confia nela. O `api` monta o mesmo volume em
`/signer-dev-pki` para o PSC falso emitir e-CPF de teste com a chave da
intermediária. Para recomeçar a AC: `docker compose rm -sf signer && docker volume rm rota-saude_signer-dev-pki`
(assinaturas de dev feitas com a AC antiga passam a `invalid`/`untrusted_chain`).

## Dependências e licenças

- Demoiselle Signer 4.6.2 (Serpro, LGPL-3.0): políticas ICP-Brasil, cadeias, atributos, checker. Usado como biblioteca, sem alteração.
- BouncyCastle 1.80 (MIT): o `SignerInfo` com a assinatura calculada fora — o Demoiselle não assina em duas fases sem chave privada.
- Apache PDFBox 3 (Apache-2.0): a estrutura de assinatura do PDF — o Demoiselle não manipula PDF.
- Jackson e Log4j 2 (Apache-2.0).
````

`.github/workflows/ci.yml`:

```yaml
name: CI
on:
  push:
    branches: [ main ]
  pull_request:

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-java@v4
        with:
          distribution: temurin
          java-version: "21"
          cache: maven
      - run: mvn -B verify
      - run: docker build -t rotasaude/signer:ci .
```

- [ ] **Step 2: Construa a imagem**

Run: `docker build -t rota-saude/signer:dev apps/signer/.claude/mod19b`
Expected: build conclui (o estágio de build roda `mvn package` sem testes; os testes rodam na CI e no `bin/mvn`).

- [ ] **Step 3: Suba o container isolado e confira o boot**

```bash
docker rm -f signer-check >/dev/null 2>&1; docker volume rm signer-check-pki >/dev/null 2>&1
docker run -d --name signer-check -p 127.0.0.1:18090:8090 -v signer-check-pki:/dev-pki \
  -e SIGNER_TOKEN=dev-signer-token-change-me-0123456789 -e SIGNER_EXTRA_TRUST_ANCHORS=/dev-pki/anchors.pem -e SIGNER_DEV_PKI_DIR=/dev-pki \
  --entrypoint sh rota-saude/signer:dev -c 'java -cp /app/signer.jar rotasaude.signer.devpki.DevPki /dev-pki http://signer:8090/dev-pki/ && exec java -jar /app/signer.jar'
sleep 6
curl -s -H "Authorization: Bearer dev-signer-token-change-me-0123456789" 127.0.0.1:18090/health; echo
curl -s -o /dev/null -w "%{http_code}\n" 127.0.0.1:18090/health
curl -s -o /dev/null -w "%{http_code}\n" 127.0.0.1:18090/dev-pki/intermediate.crl
docker exec signer-check ls /dev-pki
docker run --rm -e SIGNER_TOKEN=dev-signer-token-change-me-0123456789 -e SIGNER_ENV=production -e SIGNER_DEV_PKI_DIR=/x rota-saude/signer:dev; echo "exit=$?"
docker rm -f signer-check >/dev/null; docker volume rm signer-check-pki >/dev/null
```
Expected: `{"version":"0.1.0","crl_updated_at":null}`; `401`; `200`; os seis arquivos da AC (`anchors.pem intermediate.crl intermediate.key.pem intermediate.pem root.crl root.pem`); a linha `signer: SIGNER_EXTRA_TRUST_ANCHORS e SIGNER_DEV_PKI_DIR são proibidos com SIGNER_ENV=production` e `exit=1`. (Conferido ao escrever o plano.)

- [ ] **Step 4: Avise a sessão do `api` antes de mexer no compose**

O `docker-compose.yml` da raiz é compartilhado e não versionado; `x-api-env` e os serviços `api`/`worker` são da sessão "API". Mande a ela (SendMessage) o resumo do Step 5 — serviço novo `signer`, três variáveis novas em `x-api-env`, um volume novo em `api`/`worker`, e que o Step 6 recria os containers `api` e `worker` — e espere o "ok" (ou o ajuste pedido) antes de editar.

- [ ] **Step 5: O serviço no compose**

Em `docker-compose.yml` (raiz do monorepo):

(a) No fim do bloco `x-api-env`, depois de `CITY_WPDA_BASE_TEMPLATE: ...`, acrescente:

```yaml
  # Serviço interno de assinatura (ADR 0032, módulo 19b). O token de dev é o mesmo do serviço signer.
  SIGNER_URL: ${SIGNER_URL:-http://signer:8090}
  SIGNER_TOKEN: ${SIGNER_TOKEN:-dev-signer-token-change-me-0123456789}
  # AC de teste do signer (só dev): o PSC falso do api emite e-CPF de teste com a chave da intermediária.
  SIGNER_DEV_PKI_DIR: /signer-dev-pki
```

(b) Em `services.api.volumes` e em `services.worker.volumes`, depois de `- rails-tmp:/rails/tmp`, acrescente `- signer-dev-pki:/signer-dev-pki`.

(c) Logo antes de `volumes:` (o bloco de volumes do fim do arquivo), acrescente o serviço:

```yaml
  signer:
    container_name: signer-dev
    # Serviço interno de assinatura (ADR 0032) — Java 21 + Demoiselle Signer (apps/signer).
    # Sem banco. Em dev gera e confia numa AC de teste no formato ICP (volume signer-dev-pki).
    build:
      context: ./apps/signer
    image: rota-saude/signer:dev
    entrypoint: ["sh", "-c", "java -cp /app/signer.jar rotasaude.signer.devpki.DevPki /dev-pki http://signer:8090/dev-pki/ && exec java -XX:MaxRAMPercentage=75 -jar /app/signer.jar"]
    environment:
      SIGNER_TOKEN: ${SIGNER_TOKEN:-dev-signer-token-change-me-0123456789}
      SIGNER_EXTRA_TRUST_ANCHORS: /dev-pki/anchors.pem
      SIGNER_DEV_PKI_DIR: /dev-pki
    volumes:
      - signer-dev-pki:/dev-pki
    ports:
      - "127.0.0.1:${SIGNER_HOST_PORT:-8090}:8090"
    restart: unless-stopped

```

(d) No bloco `volumes:` do fim, acrescente `signer-dev-pki:`.

(e) No comentário de "Uso:" do topo, acrescente a linha `#   docker compose up signer                # assinatura (ADR 0032); o api fala com ele em http://signer:8090` e, no comentário de nomes fixos, `signer-dev`.

Confira: `docker compose config --quiet && echo compose-ok`.

- [ ] **Step 6: Suba no compose e chame a partir do `api`**

```bash
docker compose up -d --no-build signer
sleep 6
docker compose ps signer --format '{{.Name}} {{.State}}'
docker compose up -d api worker
docker compose exec -T api printenv SIGNER_URL
docker compose exec -T api ls /signer-dev-pki
docker compose exec -T api ruby -rnet/http -e '
uri = URI(ENV.fetch("SIGNER_URL") + "/health")
req = Net::HTTP::Get.new(uri); req["Authorization"] = "Bearer #{ENV.fetch("SIGNER_TOKEN")}"
res = Net::HTTP.start(uri.host, uri.port) { |h| h.request(req) }
puts res.code, res.body'
docker compose logs signer --tail 5
```
(Até o merge da Task 6, `apps/signer` só tem o `.gitignore` na `main`: use `--no-build` com a imagem do Step 2; um `docker compose up --build signer` antes do merge falha por falta de `pom.xml`.)

Expected: `signer-dev running`; `http://signer:8090`; os seis arquivos da AC; `200` e `{"version":"0.1.0","crl_updated_at":null}`; o log do `signer` com uma linha `GET /health 200 <n>ms` e nada além de linhas de acesso e o `listening on 8090`. (O `up -d api worker` recria os containers para pegarem as variáveis novas: variável nova do compose não chega a container já rodando — `printenv` é a prova.)

- [ ] **Step 7: Commit (só o repo `signer`; o compose não é versionado)**

```bash
W=apps/signer/.claude/mod19b
/opt/homebrew/bin/git -C $W add Dockerfile README.md .github/workflows/ci.yml
/opt/homebrew/bin/git -C $W status --short
/opt/homebrew/bin/git -C $W commit -m "build: add container image, CI workflow and README

Two-stage image (Maven build, JRE runtime, non-root user, dev PKI folder
owned by the service), GitHub Actions running mvn verify and the image
build, and a README with routes, configuration, logging rules and the
dev compose setup.

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 6: Entrega — revisão final e paradas de autorização

**Files:** nenhum novo. Só leitura e, com autorização, merge, criação do repositório remoto e push.

**Interfaces:**
- Consumes: a branch `feat/mod-19b-signature` com os commits das Tasks 1–5; o commit da pesquisa no `docs` (Task 0).
- Produces: `rotasaude/signer` publicado com CI verde; os hashes para o plano do `api` citar; card F-19.15 movido.

- [ ] **Step 1: Verificação final da branch**

```bash
W=apps/signer/.claude/mod19b
/opt/homebrew/bin/git -C $W log --oneline main..HEAD
(cd $W && bin/mvn verify) 2>&1 | grep -E "Tests run: [0-9]+, F.*$|BUILD"
/opt/homebrew/bin/git -C $W grep -n -i "client_secret\|BEGIN PRIVATE KEY\|vidaas-sandbox.env" -- . ':!README.md' ':!src/main/java/rotasaude/signer/devpki' || echo "sem-segredo"
```
Expected: 5 commits; `Tests run: 36, Failures: 0, Errors: 0` e `BUILD SUCCESS`; `sem-segredo` (nenhuma credencial nem chave versionada; o `DevPki` só gera chave em tempo de execução).

- [ ] **Step 2: Peça a revisão final**

Use superpowers:requesting-code-review sobre `main..feat/mod-19b-signature` com foco no Review Focus deste plano. Corrija o que for Critical/Important na branch (um commit por correção) e repita o Step 1.

- [ ] **Step 3: Pare e peça autorização — uma etapa de cada vez**

Reporte ao usuário: resultado da prova (link da pesquisa), os 5 commits, 36 testes verdes e o compose de dev funcionando. Pergunte, separadamente:
1. merge `--ff-only` de `feat/mod-19b-signature` na `main` local de `apps/signer`;
2. **criação do repositório remoto `rotasaude/signer`** (público, como os outros da organização) e o `origin`;
3. push da `main`;
4. clonar os labels de `rotasaude/api` para o repo novo;
5. push do commit da pesquisa no `docs`.

- [ ] **Step 4: Com cada autorização, execute só aquela etapa**

```bash
# 1. merge local
/opt/homebrew/bin/git -C apps/signer merge --ff-only feat/mod-19b-signature
# 2. repositório remoto (só depois do "sim" para criar)
gh repo create rotasaude/signer --public --description "Serviço interno de assinatura ICP-Brasil (CAdES/PAdES AD-RB) do Rota Saúde" 
/opt/homebrew/bin/git -C apps/signer remote add origin git@github.com:rotasaude/signer.git
# 3. push
/opt/homebrew/bin/git -C apps/signer push -u origin main
gh run list --repo rotasaude/signer --limit 1
# 4. labels
gh label clone rotasaude/api --repo rotasaude/signer
# 5. docs
/opt/homebrew/bin/git -C docs log --oneline origin/main..main
/opt/homebrew/bin/git -C docs push origin main
```
Expected: merge fast-forward; repositório criado; push aceito; a corrida de CI (`CI`) termina `completed success` (acompanhe com `gh run watch` ou repita o `gh run list`); labels clonados; no `docs`, `origin/main..main` mostra só o commit da pesquisa desta sessão (se houver commit de outra sessão, publique só o seu num worktree a partir de `origin/main` e avise a outra sessão).

- [ ] **Step 5: Limpeza e card**

```bash
/opt/homebrew/bin/git -C apps/signer worktree remove .claude/mod19b
/opt/homebrew/bin/git -C apps/signer branch -d feat/mod-19b-signature
```
Mova o card F-19.15 ("Serviço `signer`") do board #1 para Done (o coordenador move para Verified depois da passada do `api`). A pasta `.claude/signer-proof/` pode ficar até o fim do 19b (o plano do `api` pode querer rodar a prova de novo); apague-a ao fechar o módulo.

---

## Self-review (feito ao escrever o plano)

1. **Cobertura:** contrato §9 (quatro rotas, token, erros, sem log de corpo) → Tasks 3 e 4; spec §6 (Java 21, Demoiselle, sem banco, só saída para LCR) → Decisões 1–5, Tasks 1–4; spec §11 (JUnit com certificados de teste, ida e volta e adulteração em CAdES e PAdES) → Tasks 2 e 3; spec §12 passo 1 (prova antes de tudo; falhou → parar) → Task 0; brief (repo novo com autorização, Dockerfile, compose, CI, README, Maven justificado, classes reais do Demoiselle e a alternativa para o que falta) → Decisões 1–2, Tasks 1, 5 e 6.
2. **Placeholders:** os `<...>` que restam são valores que só a execução conhece (credenciais do usuário na Task 0, Step 2; resultados da prova no modelo da pesquisa; a data da execução no nome do arquivo). Todo código está no texto e foi compilado e executado ao escrever o plano.
3. **Consistência de nomes:** `SignatureEngine.prepare/assemble/verify`, `PreparedSignature`, `PreparedStateCodec.encode/decode`, `DevPki.generate/issue/revoke/crl/writeTo`, `TestPki.get/professional/rawSign` — os mesmos em todas as tasks e nos testes.
4. **Review Focus:** as cinco linhas têm teste nas Tasks 1, 3, 4 e 5.
5. **Ordem das tasks e compilação:** a Task 1 cria `ConfiguredAnchorsProviderCA` porque `Config` usa a constante `ENV`; a Task 2 a registra no `ServiceLoader` e a exercita (`DevPkiTest`).

## Divergências propostas ao contrato

- **S1 — vocabulário de `reasons` do `/verify`.** O §9 não lista motivos. Fixados aqui: `malformed_signature`, `signature_mismatch`, `untrusted_chain`, `policy_mismatch`, `signing_time_missing`, `certificate_expired`, `key_usage`, `certificate_revoked`, `pdf_modified_after_signing` (→ `invalid`); `revocation_unavailable`, `certificate_revoked_after_signing` (sozinhos → `indeterminate`); outros códigos do Demoiselle em minúsculas (→ `invalid`). O `api` guarda esses códigos em `last_verification_reasons`.
- **S2 — revogação depois da assinatura.** Sem carimbo do tempo (AD-RB) não há como provar que a assinatura veio antes da revogação: revogado depois do `signingTime` → `indeterminate` (`certificate_revoked_after_signing`); antes → `invalid`. Coerente com a spec §5 ("Revogação posterior → indeterminate/invalid").
- **S3 — `document_base64` no `/verify`.** Obrigatório em CAdES (400 sem ele); ignorado em PAdES (o PDF é a assinatura).
- **S4 — `validation_material_base64`.** CMS `SignedData` degenerado (certs-only, `.p7c`) com a cadeia e as LCRs obtidas no ato; LCR indisponível não impede o `/assemble` (vai sem ela; o `/verify` do mesmo ato diz `indeterminate`).
- **S5 — `crl_updated_at`.** Instante do último download de LCR bem-sucedido deste processo (`null` até o primeiro). `version` vem do manifesto do jar (`0.1.0` hoje).
- **S6 — rotas fora do §9.** Rota desconhecida → 404 `{ "error": "not_found" }`; `/health` também exige o token; em dev (só com `SIGNER_DEV_PKI_DIR`), `GET /dev-pki/{root.crl,intermediate.crl}` sem token.
- **S7 — limites e recusas.** Corpo acima de 48 MB → 400; certificado não RSA → 422 `invalid_certificate`; PDF ilegível no `/prepare` → 400; política fora do período de assinatura → 500 `internal`.
- **S8 — PAdES invisível.** O `signer` não desenha nada no PDF: o rodapé NGS2 (nome, CPF mascarado, data/hora, política, "verifique em validar.iti.gov.br") tem de vir no PDF que o `api` manda ao `/prepare`.
- **S9 — ambiente de dev para o `api`.** `SIGNER_URL`, `SIGNER_TOKEN` e `SIGNER_DEV_PKI_DIR=/signer-dev-pki` em `x-api-env`; volume `signer-dev-pki` no `api`/`worker`; formato do e-CPF de teste na Task 2 (Produces). O plano do `api` deve usar esses nomes para o PSC falso e para o `signer` real no compose de teste (spec §11).
- **S10 — caminho dos esquemas canônicos.** O `signer` não lê os esquemas; ver a divergência C1 do plano do `contracts` (`clinical/…` em vez de `schemas/clinical/…`).
