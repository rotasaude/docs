# Atender uma requisição LGPD

Como a plataforma responde quando um cidadão (titular) exerce um direito da
LGPD numa cidade: acesso aos dados (Art. 18, II), revogação do consentimento
(Art. 8º, §5º; Art. 18, IX), eliminação (Art. 18, VI) e revisão de decisão
automatizada (Art. 20). O **por quê** está nos ADRs 0013, 0014 e 0020. Este
runbook não decide nada: onde falta decisão, ele diz que falta.

## Quando aplicar

A cidade (controladora) recebe o pedido do titular e o repassa à plataforma
(operadora) por canal formal. A plataforma não atende o cidadão diretamente e
não responde pela cidade.

## Pré-requisitos

- Pedido formal da cidade, com o **slug da cidade**, o **CPF** e o **celular**
  do titular, e o direito exercido. O dado do titular vive só no banco da
  cidade dele (ADR 0020): uma requisição nunca atravessa cidades.
- Acesso de console ao papel `worker` do deploy
  (`kamal app exec --roles=worker -i 'bin/rails console'`). Toda leitura
  acontece dentro de `CityConnection.with(city)`: CPF, celular, telefone das
  mensagens, `raw` e evidência de consentimento são cifrados com a chave da
  cidade e só se leem ali dentro.
- Registro do atendimento (quem, quando, qual pedido) no sistema de chamados
  da operação. **Não** anote CPF, celular nem conteúdo clínico no chamado.

## Onde mora o dado do titular (banco da cidade)

| Tabela | Como achar | O que guarda |
|---|---|---|
| `citizens` | `Citizen.find_by(cpf:)` (cifra determinística, busca por igualdade) | CPF, celular |
| `conversations` | `citizen_id`, ou `phone` | canal, estado, telefone |
| `consents` | `conversation_id` | versão do termo, hash do texto, evidência (cifrada), revogação |
| `triages` | `conversation_id` | respostas, resultado, classificação |
| `report_snapshots` | `triage_id` | relatório congelado e assinado |
| `attendances`, `appointments`, `appointment_requests` | `citizen_id` | check-in, desfecho, agenda |
| `citizen_verifications` | `citizen_id` | validação presencial |
| `citizen_sessions`, `otp_challenges` | `phone` | sessão e código do canal web |
| `inbound_messages` | `from` (cifrado; busca por igualdade) | mensagem recebida; `raw` cifrado e zerado aos 90 dias |
| `outbound_messages` | `to` (cifrado; busca por igualdade) | mensagem enviada |
| `domain_events` | `payload` (por `*_id`) | trilha de auditoria; só referências, 12 meses |

No banco de plataforma (`platform_events`) não há dado pessoal: a guarda de
payload recusa CPF, e-mail, telefone e nome.

## Passos

### Acesso (Art. 18, II)

1. Abra o console no papel `worker` e entre na cidade:
   `city = City.find_by!(slug: "<slug>")`, depois `CityConnection.with(city) { ... }`.
2. Ache o titular por CPF e confira o celular. Se não bater, pare e devolva à
   cidade: não confirme a existência de cadastro para quem não é o titular.
3. Reúna, pelas chaves da tabela acima, as linhas de cada tabela. Para
   `triages`, inclua o relatório (`report_snapshots`). O `raw` das mensagens
   com mais de 90 dias já é `NULL`.
4. Gere o arquivo para a cidade fora do console (JSON), cifrado para o canal
   combinado com ela. Nunca em log, chamado ou e-mail aberto.

### Revogação do consentimento

O próprio titular revoga pela conversa (canal web) ou pela triagem
(`POST /citizen/triages/:id/revoke_consent`). Pela plataforma, só a pedido
da cidade: `RevokeConsent` dentro de `CityConnection.with(city)`, para a
conversa do titular.

- Publica `consent.revoked`. Os assinantes zeram o conteúdo clínico da triagem
  interrompida (`AnonymizeRevokedTriageJob`) e contam a métrica. A casca da
  triagem e o registro do consentimento ficam, com `revoked_at`.
- Efeitos anteriores à revogação continuam: relatório emitido, atendimento
  feito, alerta enviado.

### Eliminação (Art. 18, VI)

**Sem decisão.** O que pode ser apagado e o que fica retido por base legal
(prontuário, auditoria) é o ADR pendente "Art. 18 LGPD — exclusão do cadastro
do cidadão" (card no board, módulo 07). Até ele:

1. Não apague nada à mão. As tabelas de auditoria recusam por trigger:
   `domain_events` e `platform_events` são só acréscimo (DELETE só depois de
   12 meses), assim como `consents`/`consent_terms`, `citizen_verifications`,
   `attendances`, `appointments`, `memberships` e `users`.
2. Responda à cidade com o que existe e o prazo de retenção de cada parte
   (tabela acima e seção "Retenção").
3. Se o titular também revoga, aplique a revogação (acima).

### Revisão de decisão automatizada (Art. 20)

A classificação da triagem é automatizada. A explicação fica congelada no
resultado (`triages.outcome`, módulo 03) e no relatório. A plataforma entrega
essa trilha à cidade. **Quem revisa é a cidade**, por profissional dela; o
fluxo de revisão ainda não tem dono no produto (risco do módulo 07).

## Retenção (o que se apaga sozinho)

| Dado | Prazo | Quem apaga |
|---|---|---|
| `inbound_messages.raw` | 90 dias | `PurgeInboundRawJob`, diário |
| `domain_events` | 12 meses | `PurgeDomainEventsJob`, diário; o trigger recusa antes disso |
| `report_snapshots` | 30 dias depois de expirar | `PurgeExpiredReportsJob`, diário |
| `processed_events` (dedup, sem dado pessoal) | 60 dias | `PurgeProcessedEventsJob`, diário |
| O resto do cadastro | sem prazo | depende do ADR do Art. 18 |

Com o worker parado, a purga não roda. Uma cidade atrasada só atrasa a si
mesma (um supervisor por cidade).

## Validação

- O arquivo de acesso tem linhas de todas as tabelas acima em que o titular
  aparece, e nenhuma de outro titular nem de outra cidade.
- Depois da revogação: o consentimento tem `revoked_at`; a triagem interrompida
  tem `answers` vazio; existe `consent.revoked` em `domain_events`.

## Rollback

Revogação não tem desfazer: o titular dá consentimento novo por uma conversa
nova. Acesso não altera dado.

## Riscos

- O telefone das mensagens (`inbound_messages.from`, `outbound_messages.to`)
  é cifrado, mas fica sem prazo de retenção.
- O payload de `user.invited` em `domain_events` guarda o e-mail do servidor
  convidado (não é titular cidadão, mas é dado pessoal), por 12 meses.
- Eliminação e revisão humana seguem sem decisão (ver acima).

## ADRs relacionados

0013 (cifra em repouso), 0014 (auditoria, retenção e purga), 0017 (identidade
declarada do cidadão), 0020 (banco por cidade).
