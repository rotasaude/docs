# Módulo 06 — Identidade/Acesso

- **Estado:** Planejado
- **Tipo:** MVP

## Escopo

Autenticação (staff e plataforma), sessão, MFA, memberships, RBAC, convites,
provisionamento de município. Identidade do cidadão (no wpda) usa token
assinado — não há senha de cidadão neste módulo.

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0011 | `has_secure_password`, `Session`, MFA TOTP (operador no login, publisher em step-up), seam gov.br |
| 0012 | Memberships, roles, append-only, control plane vs data plane, fallback de quatro olhos, eventos platform-scope |
| 0013 | Provisionamento de município (control plane + data plane, command único) |

## Superfícies

| Superfície | O que aparece |
|---|---|
| `api` | `users`, `sessions`, `identities`, `memberships`, `invitations`, `municipalities`, `ProvisionMunicipality`, policies, MFA, `Platform.audit` |
| `admin` | Provisionar cidade, custódia de chave (visão), gestão de operadores, MFA obrigatória no login, visão cross-tenant |
| `dashboard` | Login (cidade), gestão de usuários da cidade (`municipal_admin`), convites, RBAC visível, publicação de protocolo com step-up |
| `wpda` | Acesso via token assinado (sem login), revogação de token |

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

## Dependências

- Módulo 07 (LGPD/Auditoria) — eventos de identidade são platform-scope,
  caminho irmão do `DomainEvents.publish`.
- Módulo 03 (Triagem) — backstop e step-up MFA na publicação.

## Riscos herdados

- **(ADR 0013):** chave única do AR Encryption protege secrets de todas
  as cidades. Aceito no piloto; endurecimento (envelope encryption por
  tenant) fica fora.
- **(ADR 0012):** quatro olhos colapsa em município pequeno; fallback via
  backstop do `platform_operator` desenhado mas exige operador disponível.
- **Em aberto:** integração gov.br OIDC; recovery assistido de MFA;
  desprovisionamento de cidade (inverso de `ProvisionMunicipality`).

## Critério de fechamento do módulo

- F-06.1 a F-06.17 verificadas.
- Suíte de invariante: RLS-exempt apenas no control plane (users, sessions,
  identities, memberships, municipality_channels, municipalities);
  membership revogado não autoriza; step-up MFA bloqueia publicação sem TOTP
  recente; convite expirado não cria usuário.
- Documentação de custódia/rotação de chave registrada em `docs/operacao/`.

## Histórico
