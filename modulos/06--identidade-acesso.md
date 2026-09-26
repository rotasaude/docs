# Módulo 06 — Identidade/Acesso

- **Estado:** Planejado
- **Tipo:** MVP

## Escopo

Autenticação (staff da cidade e operador de plataforma), sessão, MFA,
memberships, RBAC, convites, provisionamento de cidade e entrada do operador
numa cidade por grant. Identidade do cidadão, sem senha: **declarada** no
`wpda` (CPF conferido pelo dígito verificador + celular confirmado por código
SMS, ADR 0017) e **comprovada** por um servidor da UBS no primeiro atendimento
presencial (`citizen_verifier`, ADR 0017). O relatório público continua
aberto por token assinado (ADR 0010).

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0011 | `has_secure_password`, `Session`, MFA TOTP (operador no login, publisher em step-up), seam gov.br |
| 0012 | Memberships, roles, append-only, eventos platform-scope |
| 0013 | Provisionamento de cidade, secrets e custódia de chave |
| 0016 | Duas revisoras por publicação e por ativação; substitui o quatro-olhos do 0012 e acaba com o backstop do operador |
| 0017 | Identidade do cidadão: declarada (par CPF + celular, sessão `citizen_session` por celular, código SMS por `OtpSender`) e comprovada no balcão pelo papel `citizen_verifier` |
| 0020 | Banco por cidade: `users`/`memberships` no banco da cidade, `Operator` na plataforma, grant de entrada, chave derivada por cidade |

## Superfícies

| Superfície | O que aparece |
|---|---|
| `api` | Na cidade: `users`, `sessions`, `identities`, `memberships`, `invitations`; do cidadão, `citizens`, `citizen_sessions`, `otp_challenges`, `citizen_verification_codes`, `citizen_verifications`. Na plataforma: `operators`, `operator_sessions`, `cities`, `city_grants`. `POST /cities` + `ProvisionCityJob`, policies, MFA, `Platform.audit` |
| `admin` | Provisionar cidade, registrar canal, entrar numa cidade por grant (sessão só leitura), MFA obrigatória no login; sem visão entre cidades |
| `dashboard` | Login (cidade), gestão de usuários da cidade (`municipal_admin`), convites, RBAC visível, publicação de protocolo com step-up, validação presencial do cidadão (`citizen_verifier`) |
| `wpda` | Entrada do cidadão por CPF declarado + código SMS (sessão por celular, várias pessoas por aparelho); relatório público por token assinado |

## Funcionalidades planejadas

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-06.1 | `has_secure_password` + `Session` (Rails 8 generator) | api | 0011 |
| F-06.2 | Reset de senha por e-mail | api | 0011 |
| F-06.3 | MFA TOTP + recovery codes | api | 0011 |
| F-06.4 | MFA obrigatória no login do operador de plataforma (`Operator`) | api, admin | 0011, 0020 |
| F-06.5 | Step-up MFA no ato de publicar protocolo | api, dashboard | 0011, 0009 |
| F-06.6 | Modelo `memberships` (append-only, end-dating) | api | 0012 |
| F-06.7 | Papéis da cidade (`municipal_admin`, `protocol_author`, `protocol_publisher`, `protocol_reviewer`, `citizen_verifier`, `health_professional`, `viewer`); operador fora das memberships | api | 0012, 0016, 0020 |
| F-06.8 | Policies (defesa em profundidade: policy + banco da cidade escolhido pelo Host) | api | 0012, 0020 |
| F-06.9 | Convites (`invitations`) | api, admin, dashboard | 0012 |
| F-06.10 | Desativação de usuário (append-only) | api, dashboard | 0012 |
| F-06.11 | Provisionamento de cidade em duas fases (`POST /cities` + `ProvisionCityJob`) | api, admin | 0013, 0020 |
| F-06.12 | Custódia de chave AR Encryption no env por ambiente | api (config) | 0013 |
| F-06.13 | Termos de consentimento por cidade (`consent_terms`) | api, dashboard | 0013 |
| F-06.14 | Destinatário de alerta por cidade (`alert_recipients`) | api, admin | 0013 |
| F-06.15 | Sem backstop do operador: publicar e ativar protocolo exigem duas revisoras da cidade | api, dashboard | 0016 |
| F-06.16 | Token assinado de acesso do paciente | api, wpda | — (toca módulo 04) |
| F-06.17 | Eventos platform-scope (`Platform.audit`) | api | 0014 |
| F-06.18 | Identidade declarada do cidadão: par (CPF + celular) com CPF conferido pelo dígito verificador | api, wpda | 0017 |
| F-06.19 | Código SMS de confirmação do celular (`OtpSender`; 6 dígitos; 10 min; limites por celular e IP) | api, wpda | 0017 |
| F-06.20 | Sessão do cidadão por celular (`citizen_session`; 30 dias deslizantes; várias pessoas por aparelho) | api, wpda | 0017 |
| F-06.21 | Papel `citizen_verifier` (concedido e revogado pelo `municipal_admin` com step-up) | api, dashboard | 0017, 0012 |
| F-06.22 | Validação presencial: código "Validar no posto" no wpda + conferência do documento no dashboard | api, wpda, dashboard | 0017 |
| F-06.23 | Desfazer validação (só `municipal_admin`; motivo obrigatório; tabela só de acréscimos) | api, dashboard | 0017 |
| F-06.24 | Histórico do cidadão: par declarado vê só as próprias triagens; verificado vê todo o histórico do CPF | api, wpda | 0017 |
| F-06.25 | Papel `health_professional` (chama e registra o desfecho; concedido pelo `municipal_admin` com step-up) | api, dashboard | 0019, 0012 |

## Dependências

- Módulo 07 (LGPD/Auditoria) — eventos de identidade são platform-scope,
  caminho irmão do `DomainEvents.publish`.
- Módulo 03 (Triagem) — assinaturas das revisoras (ADR 0016) e step-up MFA
  na publicação e na ativação.

## Riscos herdados

- **(ADR 0020):** a chave de cada cidade é derivada da chave da plataforma.
  Um dump de uma cidade não abre outra, mas quem tem a chave da plataforma
  deriva todas; e ainda não há procedimento para rotacioná-la.
- **(ADR 0016):** cidade sem duas revisoras elegíveis fica bloqueada para
  publicar e ativar protocolo. Não há backstop do operador de plataforma.
- **(ADR 0020):** quem atua em duas prefeituras tem duas contas e dois
  cadastros de MFA.
- **(ADR 0017):** CPF declarado não prova nada. Por isso cada par (CPF,
  celular) vê só as próprias triagens até a validação presencial.
- **(ADR 0017):** o provedor real de SMS é pendência de go-live; sem ele o
  envio do código responde 503 (em dev, o código sai no log do `api`).
- **Em aberto:** gates de go-live do gov.br (callback único em `auth.*` já
  implementado); recovery assistido de MFA.

## Critério de fechamento do módulo

- F-06.1 a F-06.25 verificadas.
- Suíte de invariante: sessão de uma cidade não autentica em outra (cookie
  host-only, conexão escolhida antes da autenticação); grant expirado ou de
  outra cidade é recusado; o mantenedor nunca assina nem concede papel
  privilegiado; sessão do cidadão e sessão de servidor nunca autenticam uma
  à outra; um par (CPF, celular) não vê triagem de outro par; membership
  revogado não autoriza; step-up MFA bloqueia publicação sem TOTP
  recente; convite expirado não cria usuário.
- Documentação de custódia/rotação de chave registrada em `docs/operacao/`.

## Histórico
