# Módulo 06 — Identidade/Acesso

- **Estado:** Fechado
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
| F-06.13 | Termos de consentimento por cidade (`consent_terms`) | api | 0013 |
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

## Riscos que continuam

- **Produção sem credentials próprias:** desde este fechamento produção só
  sobe com `config/credentials/production.yml.enc`. O arquivo e a
  `production.key` ainda não existem; até lá o job `production-boot` da CI
  fica vermelho e produção não sobe. Passos em `operacao/custodia-de-chave.md`.
- **Chave da plataforma sem rotação:** a de cada cidade rotaciona
  (`city:rotate_key`); a da plataforma, da qual todas derivam, não.
- **Destinatário de alerta (F-06.14):** só é definido no provisionamento; não
  há tela nem endpoint para editar. O despacho usa só o primeiro destinatário
  de e-mail; o canal `whatsapp` e a ordem de escalonamento ficam sem uso.
- **Usuário desativado não volta:** não há reativação, e o e-mail dele não
  pode ser convidado de novo na cidade. Quem foi desativado antes deste
  fechamento ainda tem memberships ativas (a lista já o esconde).
- **Termo de consentimento sem tela:** versão nova só por
  `city:consent_term:publish`, rodada pela plataforma.
- **Operador sem autocadastro:** nasce por `operator:create`, com TOTP já
  ativo; a tela `MfaEnroll` do `admin` fica sem uso (rota da cidade).
- **Convite:** o token fica em claro em `invitations` e nos argumentos do job
  de e-mail (`solid_queue_jobs`) até a limpeza da fila.
- **Suíte de invariante espalhada:** os exemplos estão em `requests/`,
  `commands/` e `architecture/`, sem tag nem pasta própria.

## Critério de fechamento do módulo

- ✓ F-06.1 a F-06.25 verificadas.
- ✓ Suíte de invariante, no `api`:
  - sessão de uma cidade não autentica em outra:
    `requests/city_session_isolation_spec.rb`, `architecture/cookie_domain_spec.rb`;
  - grant expirado ou de outra cidade é recusado: `requests/session_grant_spec.rb`;
  - o mantenedor nunca assina nem concede papel privilegiado:
    `commands/protocols_authoring_spec.rb`, `commands/grant_role_spec.rb`,
    `architecture/protocol_signatures_guard_spec.rb`;
  - sessão do cidadão e sessão de servidor nunca autenticam uma à outra:
    `requests/citizen_api/otp_and_session_spec.rb`,
    `requests/citizen_session_on_staff_endpoints_spec.rb`;
  - um par (CPF, celular) não vê triagem de outro par:
    `requests/citizen_api/isolation_spec.rb`;
  - membership revogado não autoriza: `requests/admin/api/membership_gate_spec.rb`;
  - step-up MFA bloqueia publicação sem TOTP recente:
    `requests/protocol_lifecycle_spec.rb`;
  - convite expirado não cria usuário: `commands/accept_invitation_spec.rb`,
    `requests/setup_accept_invitation_spec.rb`.
- ✓ Custódia e rotação de chave em `operacao/custodia-de-chave.md`.

## Histórico

- 2026-09-27 — Módulo fechado: 25/25 `Verified`. A verificação por F-ID
  achou cinco lacunas reais, consertadas com TDD:
  - F-06.10: a desativação respondia sempre 403; agora é do `municipal_admin`,
    com step-up, e revoga os papéis ativos. O dashboard ganhou o botão.
  - F-06.9: o convite de membro não era entregue; agora sai por e-mail
    (`MemberInvitationMailer`), substitui o pendente do mesmo e-mail, e o
    aceite recusa convite vencido, senha curta e e-mail já cadastrado, com
    trava na linha. O dashboard ganhou o formulário de convite.
  - F-06.12: o initializer lia `AR_ENCRYPTION_*` e o deploy injetava
    `ACTIVE_RECORD_ENCRYPTION_*`; agora lê os dois, derruba o boot publicado
    sem chave, e produção exige credentials próprias.
  - F-06.4: `operator:create` cria o operador com TOTP; o `admin` explica
    `mfa_enrollment_required` e o fim das tentativas no login.
  - F-06.13: `city:consent_term:publish` publica versão nova; o hash do
    consentimento passa a ser o do texto do termo da cidade.
  Triggers de só acréscimos em `consent_terms`, `memberships` e `users`
  (migrações de cidade `20260927200001` e `20260927200002`). Testes novos
  para limite por IP, step-up dos papéis `citizen_verifier` e
  `health_professional`, e as invariantes de sessão cruzada e convite vencido.
  Commits: api `c36cd01` (2125 exemplos, 0 falhas), dashboard `b248fba`
  (321 testes), admin `9a8c0f6` (46 testes).

- 2026-10-01: códigos SMS (`otp_challenges`) e sessões do cidadão
  (`citizen_sessions`) passam a ter prazo de retenção (api#31): os códigos saem
  7 dias depois de vencer e as sessões 30 dias depois de vencer ou de serem
  encerradas, por `PurgeCitizenChannelJob` (módulo 07). A cota de 5 códigos por
  celular em 24 h não muda.
- 2026-10-04: o contrato de sessão troca `municipality_*` por `city_slug`,
  `city_name` e `city_uf`, e o envelope de `/admin/api` troca `municipality`
  por `city` (api#35; Revisão do ADR 0020). Ordem de deploy: api com as duas
  formas → dashboard e admin → api sem as antigas. O contrato passa a ser
  documentado e versionado no contracts (`session-v1.0.0`). Fora deste
  contrato e sem consumidor hoje: `GET /admin/api/municipalities`.
