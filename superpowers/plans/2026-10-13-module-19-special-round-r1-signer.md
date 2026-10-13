# Rodada especial do módulo 19 — fatia R1: signer — plano de implementação

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fechar os 9 endurecimentos do signer#1 antes do go-live da assinatura digital, sem mudar o contrato do signer com o api.

**Architecture:** Tudo no repo `apps/signer` (Java 21, Demoiselle 4.6.2, PDFBox, JDK HttpServer), em quatro frentes:
- a verificação (`SignatureVerifier`, `CertificateCheck`);
- o download de LCR, que passa a ser nosso (`CrlFetcher` atrás do `TrackingCrlRepository`, já registrado por ServiceLoader), com política de URL, cache e contagem de falhas;
- o `/assemble` e a partida;
- o HTTP e a imagem.

**Tech Stack:** Java 21, Maven (via `bin/mvn`, em container), JUnit 5.10, Demoiselle Signer 4.6.2, BouncyCastle, PDFBox 3.

**Spec:** `docs/superpowers/specs/2026-10-13-module-19-special-round-design.md` (§4). Card: rotasaude/signer#1.

## Global Constraints

- **Contrato com o api:**
  - os campos do `/verify` não mudam: `status`, `signer_cpf`, `signer_name`, `policy_oid`, `signed_at` e `reasons`;
  - os códigos HTTP existentes também não;
  - o api trata 400/422 como recusa e o resto como indisponível.
- **Tolerância de relógio:** 5 minutos. Motivo novo: `signed_in_future`, que dá status `invalid`.
- **CPF e nome:** só quando `Chain.of(certificate)` encadeia numa âncora confiável. Sem cadeia, nulos.
- **Política de URL da LCR:**
  - só `http` e `https`;
  - em `SIGNER_ENV=production`, recusa host que resolva para loopback, link-local, site-local (privado), any-local ou multicast;
  - em development e staging, a AC de teste é servida em localhost e passa.
- **Concorrência:** `SIGNER_MAX_CONCURRENCY`, padrão 8, inteiro ≥ 1. O excesso recebe 503 `{"error":"busy"}` com `Retry-After: 1`. O `/health` e o `/livez` não passam pelo semáforo.
- **Corpo:** `Content-Length` acima de 48 MiB recebe 400 `invalid_request` antes de ler o corpo.
- **Cache de LCR:**
  - vale até o menor entre o `nextUpdate` e `buscada + signer.crlCacheMaxSeconds`;
  - a propriedade tem padrão 3600 e lê a variável `SIGNER_CRL_CACHE_MAX_SECONDS`;
  - no Surefire, a propriedade vale 0, para os testes existentes verem cada mudança da AC de teste;
  - limite de 256 entradas;
  - corpo da LCR com no máximo 10 MiB.
- **ByteRange:**
  - 4 valores, `range[0] == 0`, todos ≥ 0;
  - `range[0] + range[1] < range[2]`;
  - `pdf[range[1]] == '<'`, `pdf[range[2]-1] == '>'` e só hexadecimal entre eles;
  - `range[2] + range[3] == length`.

  Motivo novo: `pdf_byte_range_invalid`, que dá `invalid`. O motivo `pdf_modified_after_signing` continua para a soma diferente do tamanho.
- **/assemble:**
  - reconfere o certificado (cadeia, validade agora e LCR). Revogado dá 422 `invalid_certificate`; LCR fora do ar dá 500 `internal`, como no `/prepare`;
  - no PAdES, verifica o PDF produzido; status `invalid` dá 422 `assembled_signature_invalid`.
- **Partida:** sem nenhuma âncora de confiança carregada, o boot falha com mensagem e código 1.
- **Imagem:**
  - `FROM` por digest:
    - `mirror.gcr.io/library/maven@sha256:3a4ab3276a087bf276f79cae96b1af04f53731bec53fb2e651aca79e4b10211e`
    - `mirror.gcr.io/library/eclipse-temurin@sha256:cff19e6215689161eb6162c11b86b0c60ddf802164f2eaf48d570f8fb79a36c5`
  - `GET /livez` sem token responde 200 `ok`;
  - `HEALTHCHECK` com `java -cp /app/signer.jar rotasaude.signer.HealthProbe`.
- **LCR indisponível:**
  - log `WARN revocation_unavailable host=<host>`, no máximo um por host por minuto, nunca com DN, CPF ou nome;
  - o `/health` ganha `revocation_unavailable_count` (total desde a partida) e `revocation_unavailable_since` (início da sequência de falhas atual, ou null).
- **Testes:**
  - JUnit com `bin/mvn verify`;
  - todo item ganha um teste que falha antes da correção;
  - uma asserção existente só muda com uma linha `Ruling:` no ledger.
- **Git:**
  - `/opt/homebrew/bin/git`;
  - Conventional Commits em inglês, terminando com `Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>`;
  - worktree em `rota-saude/.worktrees/signer-r1`, no branch `hardening/signer-r1` a partir de `origin/main`.

## Review Focus

1. **Assinatura com relógio do cliente adiantado 4 minutos.**
   - Esperado: `valid`, e não `signed_in_future`.
   - Prova: Task 1.
2. **Certificado confiável com a LCR fora do ar (`indeterminate`).**
   - Esperado: `signer_cpf` presente, para o api continuar forçando `invalid` quando o CPF não bate.
   - Prova: Task 1.
3. **LCR em localhost com `SIGNER_ENV=development` ou staging.**
   - Esperado: download permitido. Em production, a mesma URL dá `revocation_unavailable` (`indeterminate`), nunca `invalid`.
   - Prova: Task 2.
4. **Entrada do cache depois do `nextUpdate`.**
   - Esperado: baixa de novo e nunca entrega LCR vencida.
   - Prova: Task 2.
5. **Signer saturado com 8 assinaturas em curso.**
   - Esperado: `/livez` e `/health` respondem, e só a 9ª operação recebe 503.
   - Prova: Task 4.

---

### Task 0: Preparação

```bash
R=/Users/eduardovrocha/Development/ioit.solutions/rota-saude; G=/opt/homebrew/bin/git
$G -C $R/apps/signer fetch -q && $G -C $R/apps/signer worktree add -q $R/.worktrees/signer-r1 -b hardening/signer-r1 origin/main
cd $R/.worktrees/signer-r1 && bin/mvn -q verify 2>&1 | tail -5
```

Expected: `BUILD SUCCESS`. Anote no ledger a contagem de testes do relatório do Surefire (`grep -h "Tests run" target/surefire-reports/*.txt`).

No `pom.xml`, nas `systemPropertyVariables` do Surefire (junto de `SIGNER_DISABLE_CHAIN_ICP_BRASIL`), acrescente:

```xml
<signer.crlCacheMaxSeconds>0</signer.crlCacheMaxSeconds>
```

Isso desliga o cache para os testes existentes, que mudam a AC de teste (revogam, derrubam a LCR) dentro da mesma JVM. Faça o commit junto com a Task 2, que é quem lê a propriedade.

---

### Task 1: Verificação (itens 1, 2 e 3a)

**Files:**
- Modify: `src/main/java/rotasaude/signer/engine/SignatureVerifier.java`
- Modify: `src/main/java/rotasaude/signer/engine/CertificateCheck.java`
- Test: `src/test/java/rotasaude/signer/engine/CadesRoundTripTest.java`, `src/test/java/rotasaude/signer/engine/CertificateCheckTest.java`

**Interfaces:**
- Produces:
  - `static final java.time.Duration SignatureVerifier.CLOCK_SKEW = Duration.ofMinutes(5)`;
  - motivo `signed_in_future`;
  - CPF e nome nulos quando a cadeia não é confiável.

- [ ] **Step 1: testes que falham** (em `CadesRoundTripTest`):

```java
@Test
void signingTimeBeyondTheClockSkewIsInvalid() throws Exception {
    byte[] p7s = sign(PKI.professional(), JSON);
    Instant now = Instant.now();
    // Verificar "6 minutos antes" equivale a uma assinatura 6 minutos no futuro.
    Verification future = SignatureVerifier.verify(Kind.CADES, JSON, p7s, now.minus(Duration.ofMinutes(6)));
    assertEquals(Verification.INVALID, future.status());
    assertTrue(future.reasons().contains("signed_in_future"), future.reasons().toString());
}

@Test
void signingTimeWithinTheClockSkewIsValid() throws Exception {
    byte[] p7s = sign(PKI.professional(), JSON);
    Verification skewed = SignatureVerifier.verify(Kind.CADES, JSON, p7s, Instant.now().minus(Duration.ofMinutes(4)));
    assertEquals(Verification.VALID, skewed.status(), skewed.reasons().toString());
}

@Test
void untrustedChainNeverReturnsCpfOrName() throws Exception {
    DevPki.Issued stranger = TestPki.untrustedProfessional();
    // Monta a assinatura sem o /prepare (que recusa a AC desconhecida): os atributos e o CMS direto.
    byte[] attributes = SignedAttributes.build(Kind.CADES, JSON, stranger.certificate(), Instant.now());
    byte[] raw = TestPki.rawSign(stranger.privateKey(), attributes);
    byte[] p7s = CmsAssembler.assemble(attributes, stranger.certificate(), List.of(stranger.certificate()), raw);

    Verification result = engine.verify(Kind.CADES, JSON, p7s);

    assertEquals(Verification.INVALID, result.status());
    assertTrue(result.reasons().contains("untrusted_chain"), result.reasons().toString());
    assertNull(result.signerCpf());
    assertNull(result.signerName());
}

@Test
void indeterminateKeepsTheCpf() throws Exception {
    byte[] p7s = sign(PKI.professional(), JSON);
    PKI.crlAvailable(false);
    Verification result = engine.verify(Kind.CADES, JSON, p7s);
    assertEquals(Verification.INDETERMINATE, result.status(), result.reasons().toString());
    assertEquals(TestPki.CPF, result.signerCpf());
}

@Test
void untrustedChainMakesNoNetworkRequest() throws Exception {
    // Um certificado cujo ponto de distribuição aponta para um servidor que conta acessos.
    java.util.concurrent.atomic.AtomicInteger hits = new java.util.concurrent.atomic.AtomicInteger();
    com.sun.net.httpserver.HttpServer counter = com.sun.net.httpserver.HttpServer.create(new java.net.InetSocketAddress("127.0.0.1", 0), 0);
    counter.createContext("/", exchange -> { hits.incrementAndGet(); exchange.sendResponseHeaders(404, -1); exchange.close(); });
    counter.start();
    try {
        DevPki foreign = DevPki.generate("http://127.0.0.1:" + counter.getAddress().getPort() + "/");
        DevPki.Issued stranger = foreign.issue(TestPki.CPF, TestPki.NAME, java.time.LocalDate.of(1980, 5, 17),
                Instant.now().minus(Duration.ofDays(1)), Instant.now().plus(Duration.ofDays(30)));
        byte[] attributes = SignedAttributes.build(Kind.CADES, JSON, stranger.certificate(), Instant.now());
        byte[] p7s = CmsAssembler.assemble(attributes, stranger.certificate(), List.of(stranger.certificate()),
                TestPki.rawSign(stranger.privateKey(), attributes));

        engine.verify(Kind.CADES, JSON, p7s);

        assertEquals(0, hits.get());
    } finally {
        counter.stop(0);
    }
}
```

Imports: `java.time.Duration` e `static org.junit.jupiter.api.Assertions.assertNull`.

- **Teste do `CertificateCheck`.** Em `CertificateCheckTest`, um teste que chama `CertificateCheck.of(TestPki.untrustedProfessional().certificate().getEncoded(), Instant.now())` e afirma `signerCpf() == null` e `reasons().contains("untrusted_chain")`.
- **Dependências do `SignedAttributes.build`.** Se ele tocar a LCR do certificado (ele chama o Demoiselle), e por isso o teste do estranho falhar no build, monte os atributos com a mesma chamada que o `CadesRoundTripTest.demoiselleCrlFailureAfterOurCheckIsTransient` já usa e registre um `Ruling`. O objetivo do teste é a verificação.

  Run: `bin/mvn -q -Dtest=CadesRoundTripTest,CertificateCheckTest test`
  Expected: FAIL nos testes novos (`signed_in_future` ausente, CPF não nulo, `hits > 0`).

- [ ] **Step 2: implementar** em `SignatureVerifier.verify`, logo depois de ler `certificate`, `policyOid` e `signedAt`:

```java
List<X509Certificate> chain;
try {
    chain = Chain.of(certificate);
} catch (SignerFailure untrusted) {
    // Cadeia desconhecida: nada de rede (nem o checker do Demoiselle, que baixa LCR) e nada do titular.
    reasons.add("untrusted_chain");
    if (policyOid == null || !Policies.acceptedOids(kind).contains(policyOid)) reasons.add("policy_mismatch");
    List<String> unique = List.copyOf(new LinkedHashSet<>(reasons));
    return new Verification(status(unique), null, null, policyOid, signedAt, unique);
}
```

- O checker do Demoiselle passa a rodar só depois disso.
- No bloco de `signedAt`, acrescente:

  ```java
  else if (signedAt.isAfter(now.plus(CLOCK_SKEW))) reasons.add("signed_in_future");
  ```

  como outro ramo, antes do `certificate_expired`.
- A chamada final de revogação usa a `chain` já montada: `reasons.addAll(Revocation.reasons(chain, signedAt, now));`.
- O `try/catch` antigo de `Chain.of` sai.
- Em `CertificateCheck.of`: `chain == null ? null : SignatureVerifier.cpf(certificate)`.

  Run: `bin/mvn -q verify 2>&1 | tail -5`
  Expected: `BUILD SUCCESS`.

- [ ] **Step 3: commit.**

```bash
/opt/homebrew/bin/git add src/main/java/rotasaude/signer/engine/SignatureVerifier.java src/main/java/rotasaude/signer/engine/CertificateCheck.java src/test/java/rotasaude/signer/engine/CadesRoundTripTest.java src/test/java/rotasaude/signer/engine/CertificateCheckTest.java
/opt/homebrew/bin/git commit -m "fix: check trust before any network access and refuse future signing times

Co-Authored-By: Claude Opus 5.5 <noreply@anthropic.com>"
```

---

### Task 2: Download de LCR nosso (itens 3b, 6 e 9)

**Files:**
- Create: `src/main/java/rotasaude/signer/engine/CrlUrlPolicy.java`, `src/main/java/rotasaude/signer/engine/CrlFetcher.java`, `src/main/java/rotasaude/signer/engine/RevocationHealth.java`
- Modify: `src/main/java/rotasaude/signer/engine/TrackingCrlRepository.java`, `src/main/java/rotasaude/signer/http/SignerServer.java` (`health()`), `src/main/java/rotasaude/signer/App.java`, `pom.xml` (da Task 0)
- Test: `src/test/java/rotasaude/signer/engine/CrlFetcherTest.java` (novo), `src/test/java/rotasaude/signer/http/SignerServerTest.java`

**Interfaces:**
- Produces:
  - `CrlUrlPolicy(boolean production, java.util.function.Function<String, java.net.InetAddress[]> resolver)` com `boolean allowed(String url)`. O resolver padrão é `InetAddress::getAllByName`, embrulhado.
  - `CrlFetcher(CrlUrlPolicy policy, java.time.Clock clock, java.time.Duration maxAge, int timeoutMillis)` com `List<ICPBR_CRL> fetch(X509Certificate certificate)`. Nunca lança: falha vira lista vazia e conta em `RevocationHealth`.
  - `RevocationHealth` com os estáticos `recordSuccess(Instant)`, `recordFailure(String host, Instant)`, `lastSuccess()`, `unavailableCount()` e `unavailableSince()`.
  - `TrackingCrlRepository.configure(CrlFetcher fetcher)`, estático, chamado pelo `App` na partida. O padrão (testes) é `CrlFetcher` com `production=false`.

- [ ] **Step 1: testes que falham** (`CrlFetcherTest`):

```java
class CrlFetcherTest {

    private static final TestPki PKI = TestPki.get();

    @AfterEach void back() { PKI.crlAvailable(true); PKI.crlMode(TestPki.CrlMode.GOOD); }

    @Test
    void productionRefusesPrivateLoopbackAndLinkLocalTargets() {
        CrlUrlPolicy policy = new CrlUrlPolicy(true, host -> switch (host) {
            case "lcr.iti.gov.br" -> addresses("200.152.32.10");
            case "intranet" -> addresses("10.0.0.5");
            case "meta" -> addresses("169.254.169.254");
            default -> addresses("127.0.0.1");
        });
        assertTrue(policy.allowed("http://lcr.iti.gov.br/ac.crl"));
        assertFalse(policy.allowed("http://intranet/ac.crl"));
        assertFalse(policy.allowed("http://meta/latest"));
        assertFalse(policy.allowed("http://localhost:8080/ac.crl"));
        assertFalse(policy.allowed("ldap://lcr.iti.gov.br/cn=ac"));
        assertFalse(policy.allowed("file:///etc/passwd"));
    }

    @Test
    void developmentAllowsLocalhostButStillOnlyHttp() {
        CrlUrlPolicy policy = new CrlUrlPolicy(false, host -> addresses("127.0.0.1"));
        assertTrue(policy.allowed("http://127.0.0.1:9000/intermediate.crl"));
        assertFalse(policy.allowed("ldap://127.0.0.1/cn=ac"));
    }

    @Test
    void cachesUntilTheConfiguredAgeAndDownloadsAgainAfterIt() throws Exception {
        MutableClock clock = new MutableClock(Instant.now());
        CrlFetcher fetcher = new CrlFetcher(new CrlUrlPolicy(false, h -> addresses("127.0.0.1")), clock, Duration.ofMinutes(10), 2000);
        X509Certificate certificate = PKI.professional().certificate();

        assertEquals(1, fetcher.fetch(certificate).size());
        PKI.crlAvailable(false);
        assertEquals(1, fetcher.fetch(certificate).size(), "dentro da validade do cache: não baixa de novo");
        clock.advance(Duration.ofMinutes(11));
        assertEquals(0, fetcher.fetch(certificate).size(), "venceu o cache: baixa, falha e não entrega a velha");
    }

    @Test
    void neverServesAcrlPastItsNextUpdate() throws Exception {
        MutableClock clock = new MutableClock(Instant.now());
        CrlFetcher fetcher = new CrlFetcher(new CrlUrlPolicy(false, h -> addresses("127.0.0.1")), clock, Duration.ofDays(30), 2000);
        X509Certificate certificate = PKI.professional().certificate();
        fetcher.fetch(certificate);
        PKI.crlAvailable(false);
        clock.advance(Duration.ofDays(2)); // a LCR da AC de teste vale 1 dia (DevPki)
        assertEquals(0, fetcher.fetch(certificate).size());
    }

    @Test
    void failuresAreCountedAndLoggedWithoutCertificateData() throws Exception {
        long before = RevocationHealth.unavailableCount();
        PKI.crlAvailable(false);
        CrlFetcher fetcher = new CrlFetcher(new CrlUrlPolicy(false, h -> addresses("127.0.0.1")), Clock.systemUTC(), Duration.ZERO, 2000);

        fetcher.fetch(PKI.professional().certificate());

        assertTrue(RevocationHealth.unavailableCount() > before);
        assertNotNull(RevocationHealth.unavailableSince());
    }

    private static InetAddress[] addresses(String ip) {
        try { return new InetAddress[] { InetAddress.getByName(ip) }; } catch (Exception e) { throw new IllegalStateException(e); }
    }

    /** Relógio que o teste adianta. */
    static final class MutableClock extends Clock {
        private Instant now;
        MutableClock(Instant now) { this.now = now; }
        void advance(Duration d) { now = now.plus(d); }
        @Override public Instant instant() { return now; }
        @Override public ZoneId getZone() { return ZoneOffset.UTC; }
        @Override public Clock withZone(ZoneId zone) { return this; }
    }
}
```

**Teste do log.** Ainda no `CrlFetcherTest`, capture o `System.out`, como faz `SignerServerTest.logsNeverCarryTokenCpfNameOrDocument`, durante 3 falhas seguidas. Confira:
- o log contém `revocation_unavailable host=127.0.0.1` exatamente uma vez;
- não contém `TestPki.CPF` nem `TestPki.NAME`.

**Teste do /health.** Em `SignerServerTest`, `GET /health` traz os campos `revocation_unavailable_count` (número) e `revocation_unavailable_since` (presente, pode ser null).

  Run: `bin/mvn -q -Dtest=CrlFetcherTest,SignerServerTest test`
  Expected: FAIL. As classes não existem e o `/health` não tem os campos.

- [ ] **Step 2: implementar.**

**`CrlUrlPolicy.allowed(url)`:**
- `URI.create(url)`; o esquema precisa ser `http` ou `https`, e o host não pode faltar;
- fora de produção, devolve `true`;
- em produção, resolve com o resolver e recusa se qualquer endereço for `isLoopbackAddress`, `isLinkLocalAddress`, `isSiteLocalAddress`, `isAnyLocalAddress` ou `isMulticastAddress`, ou se for IPv6 ULA (`fc00::/7`, primeiro byte `& 0xfe == 0xfc`);
- erro de resolução ou de URI devolve `false`.

**`CrlFetcher.fetch(certificate)`:**
- `new BasicCertificate(certificate).getCRLDistributionPoint()`;
- para cada URL permitida:
  - procura no cache (`ConcurrentHashMap<String, Entry(ICPBR_CRL crl, Instant expires)>`); serve se `clock.instant()` for anterior a `expires`;
  - senão, baixa com `HttpURLConnection` usando `ConfigurationRepo.getInstance().getProxy()` quando não for nulo, com `connectTimeout` e `readTimeout` iguais a `timeoutMillis`, lendo no máximo 10 MiB, e faz `new ICPBR_CRL(bytes)`;
  - grava no cache com `expires = min(crl.getNextUpdate(), agora + maxAge)` (sem `nextUpdate`, ou com `maxAge` zero, não grava);
  - com mais de 256 entradas, limpa o mapa inteiro antes de gravar.
- Sucesso: `RevocationHealth.recordSuccess(agora)`.
- URL recusada, ou falha de rede ou de parse: `RevocationHealth.recordFailure(host, agora)`.
- O `timeoutMillis` padrão vem de `SIGNER_CRL_CONNECTION_TIMEOUT` (padrão 5000).

**`RevocationHealth`:**
- `AtomicLong` com o total de falhas;
- `AtomicReference<Instant>` com o início da sequência atual (vira null no sucesso);
- `AtomicReference<Instant>` com o último sucesso;
- `ConcurrentHashMap<String, Instant>` com o último log por host;
- o log sai em `LogManager.getLogger(RevocationHealth.class).warn("revocation_unavailable host={}", host)` quando passou mais de 60 s do último log daquele host.

**`TrackingCrlRepository.getX509CRL`** passa a devolver `fetcher.fetch(certificate)`. O `fetcher` é um campo estático `volatile`, com o padrão `new CrlFetcher(new CrlUrlPolicy(false, …), Clock.systemUTC(), maxAgeFromProperty(), timeout)`. O `maxAgeFromProperty()` lê `System.getProperty("signer.crlCacheMaxSeconds", System.getenv().getOrDefault("SIGNER_CRL_CACHE_MAX_SECONDS", "3600"))`. O `lastSuccess()` passa a delegar para `RevocationHealth`.

**`App.main`**, depois do `Config`:

```java
TrackingCrlRepository.configure(new CrlFetcher(new CrlUrlPolicy("production".equals(config.environment()), CrlUrlPolicy::resolve),
        Clock.systemUTC(), TrackingCrlRepository.maxAgeFromProperty(), TrackingCrlRepository.timeoutFromEnv()));
```

**`SignerServer.health()`** acrescenta:

```java
out.put("revocation_unavailable_count", RevocationHealth.unavailableCount());
Instant since = RevocationHealth.unavailableSince();
out.put("revocation_unavailable_since", since == null ? null : since.toString());
```

  Run: `bin/mvn -q verify 2>&1 | tail -5`
  Expected: `BUILD SUCCESS`, com os testes antigos de LCR fora do ar, chave errada e vencida passando (o cache está desligado no Surefire).

- [ ] **Step 3: commit** (`fix: own CRL download with URL policy, cache and unavailability metrics`), incluindo o `pom.xml`.

---

### Task 3: /assemble e partida (item 4)

**Files:**
- Modify: `src/main/java/rotasaude/signer/engine/SignatureEngine.java`, `src/main/java/rotasaude/signer/SignerFailure.java`, `src/main/java/rotasaude/signer/App.java`
- Create: `src/main/java/rotasaude/signer/trust/TrustAnchors.java`
- Test: `src/test/java/rotasaude/signer/engine/CadesRoundTripTest.java`, `src/test/java/rotasaude/signer/engine/PadesRoundTripTest.java`, `src/test/java/rotasaude/signer/trust/TrustAnchorsTest.java` (novo)

**Interfaces:**
- Produces:
  - `SignerFailure.Code.ASSEMBLED_SIGNATURE_INVALID(422, "assembled_signature_invalid")`;
  - `TrustAnchors.count()` (soma de `getCAs().size()` de todo `ProviderCA` do `ServiceLoader`);
  - `TrustAnchors.requireAny()`, que lança `IllegalStateException("nenhuma âncora de confiança carregada")`.

- [ ] **Step 1: testes que falham.**
  - **CAdES, revogado entre os dois passos:** `prepare` → `PKI.pki.revoke(serial, agora - 1h)` → `assemble` lança `SignerFailure` `INVALID_CERTIFICATE`.
  - **CAdES, LCR fora do ar entre os dois passos:** `prepare` → `PKI.crlAvailable(false)` → `assemble` lança `INTERNAL`.
  - **PAdES, PDF produzido inválido** (`PadesRoundTripTest`): troque `preparedPdf` por um PDF preparado de *outro* documento, com o mesmo certificado e os atributos assinados do primeiro. Monte `new PreparedSignature(Kind.PADES, der, prepared.signedAttributesDer(), outroPreparado.preparedPdf())`. O `assemble` lança `ASSEMBLED_SIGNATURE_INVALID`.
  - **Âncoras** (`TrustAnchorsTest`): com a propriedade `signer.extraTrustAnchors` apontando para a AC de teste (`TestPki.get()`), `TrustAnchors.count() > 0` e `requireAny()` não lança. `TrustAnchors.requireAny(0)`, uma sobrecarga de pacote que recebe a contagem, lança.

  Run: `bin/mvn -q -Dtest=CadesRoundTripTest,PadesRoundTripTest,TrustAnchorsTest test`
  Expected: FAIL.

- [ ] **Step 2: implementar.**
  - **`SignatureEngine.assemble`:**

    ```java
    Instant now = clock.instant();
    if (now.isAfter(certificate.getNotAfter().toInstant())) throw new SignerFailure(SignerFailure.Code.INVALID_CERTIFICATE);
    requireNotRevoked(chain, now);
    ```

    Isso vem antes do `CmsAssembler`. Depois de montar, no PAdES:

    ```java
    if (prepared.kind() == Kind.PADES
            && Verification.INVALID.equals(SignatureVerifier.verify(Kind.PADES, null, signature, now).status())) {
        throw new SignerFailure(SignerFailure.Code.ASSEMBLED_SIGNATURE_INVALID);
    }
    ```

  - **CAdES:** a recusa do valor RSA de outra chave já existe (`INVALID_SIGNATURE_VALUE`) e cobre a auto-verificação, porque o conteúdo destacado não está no estado preparado.
    - Ruling: CAdES verifica certificado e valor da assinatura, sem o conteúdo. Mandar o conteúdo no `prepared_state` dobraria o corpo do `/assemble`.
  - **`App.main`:** depois do `Config`, `TrustAnchors.requireAny()` dentro do mesmo `try`, imprimindo `signer: …` e saindo com 1.

  Run: `bin/mvn -q verify 2>&1 | tail -5`
  Expected: `BUILD SUCCESS`.

- [ ] **Step 3: commit** (`fix: recheck the certificate and the produced PDF at assemble and refuse to boot without trust anchors`).

---

### Task 4: HTTP (item 5)

**Files:**
- Modify: `src/main/java/rotasaude/signer/Config.java`, `src/main/java/rotasaude/signer/SignerFailure.java`, `src/main/java/rotasaude/signer/http/SignerServer.java`
- Test: `src/test/java/rotasaude/signer/ConfigTest.java`, `src/test/java/rotasaude/signer/http/SignerServerTest.java`

**Interfaces:**
- Produces:
  - `Config.maxConcurrency()` (int), com `SIGNER_MAX_CONCURRENCY` e padrão 8;
  - valor ≤ 0 ou não numérico lança `IllegalArgumentException("SIGNER_MAX_CONCURRENCY inválida")`;
  - `SignerFailure.Code.BUSY(503, "busy")`;
  - `SignerServer.start(Config, SignatureEngine)`, com a mesma assinatura de hoje;
  - `GET /livez`, sem token, responde 200 com `ok` em `text/plain`; outros métodos recebem 405. Fica fora do token e do semáforo, e a Task 6 o usa no `HEALTHCHECK`.

- [ ] **Step 1: testes que falham.**
  - **`ConfigTest`:** padrão 8; `"3"` dá 3; `"0"` e `"x"` lançam.
  - **`SignerServerTest`, liveness:** `GET /livez` sem token dá 200 e o corpo `ok`, sem `version` nem `crl`. `POST /livez` dá 405.
  - **`SignerServerTest`, saturação:** sobe um segundo servidor com `SIGNER_MAX_CONCURRENCY=1` e um `SignatureEngine` subclasse de teste. O `verify` dessa subclasse bloqueia num `CountDownLatch`. Esse é o motivo de `SignatureEngine` não ser `final`.
    - dispara um `/verify` em outra thread e espera ele entrar no latch;
    - o segundo `/verify` recebe 503, com corpo `{"error":"busy"}` e cabeçalho `Retry-After: 1`;
    - `GET /health` (com token) responde 200, e `GET /livez` responde 200;
    - libera o latch.
  - **`SignerServerTest`, tamanho:**

    ```java
    @Test
    void refusesOversizedContentLengthBeforeReadingTheBody() throws Exception {
        try (java.net.Socket socket = new java.net.Socket("127.0.0.1", java.net.URI.create(base).getPort())) {
            socket.setSoTimeout(5000);
            String head = "POST /prepare HTTP/1.1\r\nHost: x\r\nAuthorization: Bearer " + TOKEN
                    + "\r\nContent-Type: application/json\r\nContent-Length: 60000000\r\n\r\n";
            socket.getOutputStream().write(head.getBytes(StandardCharsets.US_ASCII));
            socket.getOutputStream().flush();
            String status = new java.io.BufferedReader(new java.io.InputStreamReader(socket.getInputStream(), StandardCharsets.US_ASCII)).readLine();
            assertEquals("HTTP/1.1 400 Bad Request", status);
        }
    }
    ```

    Sem nenhum byte de corpo enviado, a resposta precisa chegar antes do timeout de 5 s.

  Run: `bin/mvn -q -Dtest=ConfigTest,SignerServerTest test`
  Expected: FAIL.

- [ ] **Step 2: implementar.**
  - **Liveness.** Em `SignerServer.start`, `server.createContext("/livez", handler::livez)`, com o handler devolvendo `ok` (405 para métodos diferentes de `GET`).
  - **Semáforo.** Em `SignerServer`, o campo `private final Semaphore permits = new Semaphore(config.maxConcurrency());`. No `handle`, depois do `authorized` e só para rotas diferentes de `GET /health`:

    ```java
    if (!permits.tryAcquire()) throw new SignerFailure(SignerFailure.Code.BUSY);
    ```

    O `release()` vai num `finally` próprio, que só roda se adquiriu.
  - **Retry-After.** No `catch (SignerFailure failure)`, se o código for `BUSY`, acrescente `exchange.getResponseHeaders().set("Retry-After", "1")` antes do `send`.
  - **Content-Length.** Em `read(exchange)`, antes de ler:

    ```java
    String declared = exchange.getRequestHeaders().getFirst("Content-Length");
    if (declared != null) {
        try { if (Long.parseLong(declared.trim()) > MAX_BODY) throw invalidRequest(); }
        catch (NumberFormatException e) { throw invalidRequest(); }
    }
    ```

  - **Assinatura do `read`.** O `read` vira método de instância, ou recebe o `exchange` como hoje; o importante é não mudar a assinatura pública do `start`.
  - **Classe do engine.** Se `SignatureEngine` for `final`, tire o `final` (Ruling: necessário para o teste de saturação; nenhum outro uso).

  Run: `bin/mvn -q verify 2>&1 | tail -5`
  Expected: `BUILD SUCCESS`.

- [ ] **Step 3: commit** (`fix: bound concurrent operations and refuse oversized bodies before reading`).

---

### Task 5: ByteRange estrito (item 7)

**Files:**
- Modify: `src/main/java/rotasaude/signer/engine/SignatureVerifier.java`
- Test: `src/test/java/rotasaude/signer/engine/PadesRoundTripTest.java`

**Interfaces:**
- Produces: `static boolean SignatureVerifier.strictByteRange(int[] range, byte[] pdf)` e o motivo `pdf_byte_range_invalid`.

- [ ] **Step 1: testes que falham.** Um teste puro de `strictByteRange`, sobre um PDF assinado de verdade:

```java
@Test
void strictByteRangeAcceptsOnlyTheExactContentsGap() throws Exception {
    byte[] pdf = signedPdf();
    int[] r;
    try (PDDocument document = Loader.loadPDF(pdf)) { r = document.getLastSignatureDictionary().getByteRange(); }

    assertTrue(SignatureVerifier.strictByteRange(r, pdf));
    assertFalse(SignatureVerifier.strictByteRange(new int[] {1, r[1] - 1, r[2], r[3]}, pdf), "não começa no zero");
    assertFalse(SignatureVerifier.strictByteRange(new int[] {0, r[2] + 1, r[2], r[3]}, pdf), "intervalos sobrepostos");
    assertFalse(SignatureVerifier.strictByteRange(new int[] {0, r[1], r[2] + 2, r[3] - 2}, pdf), "vão maior que o /Contents");
    assertFalse(SignatureVerifier.strictByteRange(new int[] {0, r[1] + 1, r[2], r[3]}, pdf), "vão menor que o /Contents");
    assertFalse(SignatureVerifier.strictByteRange(new int[] {0, -5, r[2], r[3]}, pdf), "negativo");
    assertFalse(SignatureVerifier.strictByteRange(new int[] {0, r[1], r[2]}, pdf), "três valores");
}

@Test
void byteRangeRewrittenInTheFileIsInvalid() throws Exception {
    byte[] pdf = signedPdf();
    String text = new String(pdf, StandardCharsets.ISO_8859_1);
    java.util.regex.Matcher m = java.util.regex.Pattern.compile("/ByteRange\\s*\\[\\s*0\\s+(\\d+)\\s+(\\d+)\\s+(\\d+)\\s*\\]").matcher(text);
    assertTrue(m.find());
    String original = m.group();
    String forged = "/ByteRange [0 " + (Integer.parseInt(m.group(1)) + 1) + " " + m.group(2) + " " + m.group(3) + "]";
    forged = forged + " ".repeat(Math.max(0, original.length() - forged.length())); // mesmo tamanho: o xref não se mexe
    byte[] altered = (text.substring(0, m.start()) + forged + text.substring(m.end())).getBytes(StandardCharsets.ISO_8859_1);

    Verification result = engine.verify(Kind.PADES, null, altered);

    assertEquals(Verification.INVALID, result.status());
    assertTrue(result.reasons().contains("pdf_byte_range_invalid"), result.reasons().toString());
}
```

Imports: `java.nio.charset.StandardCharsets`, `org.apache.pdfbox.Loader`, `org.apache.pdfbox.pdmodel.PDDocument` e `assertFalse` (os que ainda faltarem).

  Run: `bin/mvn -q -Dtest=PadesRoundTripTest test`
  Expected: FAIL (método inexistente).

- [ ] **Step 2: implementar.**

```java
/** /ByteRange estrito: começa no zero, sem sobreposição, vão = exatamente o /Contents (<hex>), fim = fim do arquivo. */
static boolean strictByteRange(int[] range, byte[] pdf) {
    if (range == null || range.length != 4) return false;
    for (int value : range) if (value < 0) return false;
    if (range[0] != 0) return false;
    long gapStart = (long) range[0] + range[1];
    long gapEnd = range[2]; // exclusivo
    if (gapStart >= gapEnd || gapEnd > pdf.length) return false;
    if ((long) range[2] + range[3] != pdf.length) return false;
    if (pdf[(int) gapStart] != '<' || pdf[(int) gapEnd - 1] != '>') return false;
    for (long i = gapStart + 1; i < gapEnd - 1; i++) {
        if (Character.digit(pdf[(int) i], 16) < 0) return false;
    }
    return true;
}
```

- **No ramo PAdES do `verify`:** troque a checagem de tamanho por

  ```java
  if ((long) range[2] + range[3] != signature.length) reasons.add("pdf_modified_after_signing");
  else if (!strictByteRange(range, signature)) reasons.add("pdf_byte_range_invalid");
  ```

- **Se o `getSignedContent`** lançar com o range adulterado, o `catch` atual devolve `malformed`. Para o teste do arquivo reescrito, mova o `strictByteRange` para antes do `getSignedContent` e, se falhar, devolva `invalid` com só `pdf_byte_range_invalid`, sem tentar ler o conteúdo:

  ```java
  return new Verification(INVALID, null, null, null, null, List.of("pdf_byte_range_invalid"))
  ```

  Registre um Ruling se essa for a forma escolhida.

  Run: `bin/mvn -q verify 2>&1 | tail -5`
  Expected: `BUILD SUCCESS`.

- [ ] **Step 3: commit** (`fix: verify the PAdES byte range strictly`).

---

### Task 6: Imagem (item 8)

**Files:**
- Create: `src/main/java/rotasaude/signer/HealthProbe.java`
- Modify: `Dockerfile`, `bin/mvn`, `README.md` (seção de operação)
- Test: `src/test/java/rotasaude/signer/HealthProbeTest.java` (novo)

**Interfaces:**
- Consumes: `GET /livez` (Task 4).
- Produces:
  - `HealthProbe.main(String[])` sai com 0 quando `GET http://127.0.0.1:${SIGNER_PORT:-8090}/livez` dá 200 e com 1 em qualquer outro caso.
  - `static int HealthProbe.check(int port)`, que devolve o código de saída, para o teste.

- [ ] **Step 1: testes que falham.**
  - **`HealthProbeTest`:**
    - `HealthProbe.check(porta do servidor do teste) == 0`;
    - `HealthProbe.check(porta livre) == 1`, usando uma porta obtida de um `ServerSocket(0)` já fechado.

  Run: `bin/mvn -q -Dtest=HealthProbeTest test`
  Expected: FAIL.

- [ ] **Step 2: implementar.**
  - **`HealthProbe`:** `HttpURLConnection`, com timeout de 2 s e `GET`, sem dependência fora do JDK.
  - **`Dockerfile`:**

    ```dockerfile
    FROM mirror.gcr.io/library/maven@sha256:3a4ab3276a087bf276f79cae96b1af04f53731bec53fb2e651aca79e4b10211e AS build
    # (…)
    FROM mirror.gcr.io/library/eclipse-temurin@sha256:cff19e6215689161eb6162c11b86b0c60ddf802164f2eaf48d570f8fb79a36c5
    # (…)
    HEALTHCHECK --interval=30s --timeout=5s --start-period=20s --retries=3 \
      CMD ["java", "-cp", "/app/signer.jar", "rotasaude.signer.HealthProbe"]
    ```

    O comentário do topo ganha uma linha: a tag de origem (`maven:3.9.9-eclipse-temurin-21`, `eclipse-temurin:21-jre`) e como renovar o digest (`docker pull` mais `docker inspect --format '{{index .RepoDigests 0}}'`).
  - **`bin/mvn`:** troque a tag pelo mesmo digest do maven, para o build local e a CI usarem a mesma imagem.
  - **README:** a tabela de variáveis ganha `SIGNER_MAX_CONCURRENCY`, `SIGNER_CRL_CACHE_MAX_SECONDS` e `SIGNER_CRL_CONNECTION_TIMEOUT`. A seção de operação ganha o `/livez`, os campos novos do `/health` e o log `revocation_unavailable`.

  Run: `bin/mvn -q verify 2>&1 | tail -5 && docker build -q -t rotasaude/signer:r1 . && docker inspect --format '{{json .Config.Healthcheck}}' rotasaude/signer:r1`
  Expected: `BUILD SUCCESS`, imagem construída e o Healthcheck com o comando `java -cp /app/signer.jar rotasaude.signer.HealthProbe`.

- [ ] **Step 3: prova do container.**

```bash
docker run -d --name signer-r1 -e SIGNER_TOKEN=dev-signer-token-0123456789abcdef -e SIGNER_ENV=development rotasaude/signer:r1
```

- Espere cerca de 40 s e rode `docker inspect --format '{{.State.Health.Status}}' signer-r1`.
- Expected: `healthy`. Se ficar `unhealthy` por falta de âncora, que é o comportamento novo da Task 3 sem a AC de teste montada, rode com o volume `signer-dev-pki` e `SIGNER_EXTRA_TRUST_ANCHORS`, como no `docker-compose.yml` da raiz.
- `docker rm -f signer-r1`.

- [ ] **Step 4: commit** (`build: pin base images by digest and add a liveness healthcheck`).

---

### Task 7: Integração, revisão, aprovação e publicação

- [ ] **Signer de dev.** Reconstrua o `signer-dev` do compose com a imagem nova (`docker compose build signer && docker compose up -d signer`, da raiz). Confira, no api-dev, o fluxo de assinatura de dev de ponta a ponta:
  - `docker exec api-dev bin/rails runner 'p Signatures::Signer.client.health'` responde com os campos novos;
  - a spec do cliente passa: `docker compose exec api bundle exec rspec spec/services/signatures/signer_client_spec.rb`.
- [ ] **Revisão final.** Revisor independente, no modelo mais capaz, com a seção Review Focus. Critical e Important são corrigidos com TDD e entram no ledger.
- [ ] **Aprovação do usuário.** Resumo dos 9 itens com a prova de cada um: nome do teste e saída.
- [ ] **Publicação.**
  - avise as sessões que mexem no signer ou no api;
  - confira `origin/main..main`;
  - `merge --ff-only` e `push` do signer;
  - confira a CI do signer (`mvn verify` mais o build da imagem);
  - remova o worktree;
  - mova o signer#1 para "Resolvido" no board #2, com um comentário listando cada item e o teste que o prova.
