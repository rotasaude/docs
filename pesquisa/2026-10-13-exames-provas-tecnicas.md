# Módulo 21a — provas técnicas (feitas em 2026-10-14)

Task 0 do plano do api (`superpowers/plans/2026-10-13-module-21a-exams-api.md`). Base do api conferida em
`origin/main` 4ffe7c6 (última migração de cidade `20261011800002`, `VALID_RANGE = (1..35)`).

## (a) Formulário dentro da tela de "sistemas externos" do e-SUS PEC

**Resultado: PASSOU**, por reprodução fiel do iframe do PEC. O usuário aceitou essa evidência em 2026-10-14.
A abertura dentro de um atendimento real vira gate de go-live, junto com o botão no PEC real.

### Documentação oficial

- Páginas `index.html` ("Alterado em" 11/09/2026) e `iframe.html` (16/09/2025): não mudaram desde o plano.
- Elas não falam de sandbox, envio de formulário, POST nem cookies. Só dizem que o conteúdo não carrega com
  `X-Frame-Options` DENY/SAMEORIGIN nem com `frame-ancestors 'none'`.
- O cadastro é feito pelo **Administrador da Instalação**, em Integração > Sistemas externos, e só existe com
  HTTPS configurado.

### PEC local (treinamento, versão 5.5.28)

- **Cadastro:** feito pela tela `/integracao/configuracoes/sistemas-externos`, com o nome "Rota Saúde — prova 21a" e
  local de acesso Atendimento. A URL `https://localhost:8099/?documento-profissional=[documentoProfissional]&documento-cidadao=[documentoCidadao]&cnes=[CNES]`
  foi aceita com porta e `localhost`. Responsável: pessoa jurídica fictícia.
- **Locais de acesso:** o PEC oferece "Atendimento" e "Menu lateral".
- **Limite do PEC local:**
  - O banco tem 0 unidades de saúde, 0 lotações, 0 cidadãos e 1 profissional, o instalador.
  - O CNES não foi importado: o XML só sai da área restrita do e-Gestor (ver `operacao/pec-local-dev.md`).
  - Sem unidade e sem lotação, nenhum atendimento individual pode ser aberto, então a aba do sistema externo
    não pôde ser exibida no PEC local.

### Código do PEC 5.5.28

Fonte: o bundle `static/js/index.*.js`, servido pela própria instalação.

- **O iframe:** o componente que exibe o sistema externo cria `<iframe src={url} title={nome do sistema}
  referrerPolicy="strict-origin">`. É o único iframe de sistema externo do bundle.
  - Sem `sandbox` (a palavra não aparece no bundle principal).
  - Sem `allow`.
- **A URL:** vem da mutação GraphQL `gerarLinkSistemaExterno` (input: `providerId`, `sistemaExternoId`, `cidadaoId`).
  A substituição dos termos e a assinatura ficam no servidor.
- **Cabeçalhos de toda página do PEC:**
  - `Content-Security-Policy` com `frame-src 'self' blob: https://localhost:8099`: o PEC acrescentou a origem do
    sistema cadastrado.
  - Ainda na CSP: `form-action 'self'` e `frame-ancestors 'self'`.
  - `X-Frame-Options: DENY`.
- **Consequências:**
  - O `form-action 'self'` vale para formulários do documento do PEC, não para a página carregada no iframe, que
    tem a própria política.
  - A navegação do iframe para a mesma origem cadastrada (POST de formulário) é permitida pelo `frame-src`.
  - O `referrerPolicy="strict-origin"` faz o sistema externo receber só a origem do PEC como `Referer`.

### Reprodução

Um script de prova num diretório temporário, fora de qualquer repo, apagado no fim.

- **Montagem:**
  - Página "pai", com a CSP literal do PEC (`frame-src` com a origem do filho, `form-action 'self'`,
    `frame-ancestors 'self'`) e `X-Frame-Options: DENY`.
  - Dentro dela, o mesmo `<iframe>` do PEC (sem `sandbox`, `referrerpolicy="strict-origin"`), apontando para uma
    página de outro site com um formulário POST e um `fetch` POST.
- **Execução:** Chromium headless (Chrome for Testing do Playwright).
- **Resultados:**
  - POST de formulário dentro do iframe: **recebido** pelo servidor do filho.
  - `fetch` POST dentro do iframe: **recebido**.

### Limite

- A prova local não garante a versão do PEC da cidade.
- A abertura dentro de um atendimento real e o botão "Solicitar exames" no PEC real são gates de go-live (api#64).

## (b) Perfil mínimo do DiagnosticReport de entrada contra o FHIR R4

**Resultado: confere com o Desvio 13 do plano do api.**

### O que o FHIR R4 diz

| Elemento | Regra | Fonte |
|---|---|---|
| `DiagnosticReport.status` | 1..1 | `hl7.org/fhir/R4/diagnosticreport.html` |
| `DiagnosticReport.code` | 1..1 (entra no perfil) | idem |
| `DiagnosticReport.subject` | 0..1 | idem |
| `DiagnosticReport.basedOn`, `.performer`, `.result`, `.presentedForm` | 0..* | idem |
| `DiagnosticReport.issued` | 0..1 | idem |
| `Reference.identifier`, `Reference.display` | 0..1 cada; ao menos um entre `reference`, `identifier` e `display` | `references.html` |
| Recurso `contained` | precisa estar ligado ao recurso que o contém (dom-3); no perfil, referenciado em `result[]` como `"#<id>"` | `domainresource.html` |
| `Observation.status`, `Observation.code` | 1..1 | `observation.html` |
| `Observation.value[x]` | 0..1, inclui `Quantity` e `string` | idem |
| `Observation.interpretation` | 0..* | idem |
| `ServiceRequest.status`, `.intent`, `.subject` | 1..1 | `servicerequest.html` |

- **Valores aceitos de `status` no perfil:** `final` e `amended`. O `amended` faz parte do conjunto de valores do
  `final`.
- **Interpretação crítica:** `HH` (crítico alto), `LL` (crítico baixo) e `AA` (crítico anormal) existem em
  `http://terminology.hl7.org/CodeSystem/v3-ObservationInterpretation`.
- **Sistemas de identificador (RNDS, `uniqueId`):**
  - `http://rnds.saude.gov.br/fhir/r4/NamingSystem/cpf` (`rnds-fhir.saude.gov.br/NamingSystem-cpf.json.html`);
  - `http://rnds.saude.gov.br/fhir/r4/NamingSystem/cns` (`NamingSystem-cns.json.html`).

### Consequência

Valem as divergências D7 do plano do api em relação ao contrato §7. Os esquemas estão no repo `contracts`, em
`exams/fhir-diagnostic-report-inbound.json` (branch `feat/mod-21a-exams`).
