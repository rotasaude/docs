# Módulo 12 — Campanhas

- **Estado:** Fechado
- **Tipo:** Estratégico (pós-MVP)

## Escopo

Aviso ativo da cidade para um grupo de cidadãos (vacinação, prevenção
sazonal, chamada de quem faltou ou abandonou a triagem), distinto da conversa
reativa do módulo 02 (ADR 0024). O aviso chega sempre pelo **wpda**, numa
caixa atrás do login; um **SMS** com texto fixo e link é opcional, ligado por
cidade. A premissa original (templates do WhatsApp aprovados pelo Meta) caiu
com a descontinuação do WhatsApp.

**Entregue (F-12.1 a F-12.7):**
- papel `campaign_manager` (privilegiado, step-up) e chave de SMS da cidade,
  ligada pelo `municipal_admin`;
- público = recorte geográfico (cidade toda, área de unidade de referência ou
  bairros; ADR 0023) × critérios clínicos combinados por E (protocolo e
  período, faixa, triagem não concluída, desfecho de atendimento, triado e
  não atendido, falta, pedido de agendamento aberto);
- mínimo de 5 telefones distintos, sempre; prévia ao vivo no construtor;
- envio agora ou agendado; público congelado no envio;
- caixa de avisos no wpda, com selo de não lidos e silêncio;
- opt-in explícito de SMS, desligado por padrão, conferido a cada envio;
- SMS por gateway plugável, entre 8h e 20h, texto fixo sem identificador;
- painel da campanha só com agregados.

**Fora, por enquanto:** grupos "OU" no público, provedor de SMS real e sua
tela no maintenance, reenvio de SMS não entregue, expiração de aviso, anexos,
texto livre no SMS.

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0024 | Canal (wpda + SMS opcional por cidade), público congelado, mínimo de 5, opt-in de SMS, texto fixo, papel `campaign_manager` |
| 0023 | Bairros e cobertura: base do recorte geográfico |
| 0016 | Step-up e papéis privilegiados |
| 0017 | Sessão do cidadão por telefone e consentimento |

## Superfícies

| Superfície | O que aparece |
|---|---|
| `api` | `campaigns`, `campaign_recipients`, `citizen_contact_preferences`, `city_profile.campaigns_sms_enabled`; rotas `/campaigns`; `/citizen/notices` e `/citizen/contact_preferences`; `SmsGateway`; jobs de envio |
| `admin` | — |
| `dashboard` | Campanhas (lista, editor com construtor de público, envio, painel), chave de SMS, papel em Equipe |
| `wpda` | Caixa de avisos, selo de não lidos, preferências |

## Pré-requisitos

- Módulo 11 (Território) — recorte por bairro e por cobertura de unidade.
- Módulos 03, 08 e 13 — dados dos critérios clínicos.
- Provedor de SMS — só para ligar o SMS em produção; o módulo funciona sem ele.

## Funcionalidades planejadas

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-12.1 | Papel `campaign_manager` e chave de SMS por cidade | api, dashboard | 0024 |
| F-12.2 | Construtor de público: 3 recortes, 7 critérios clínicos, mínimo de 5 telefones e prévia | api, dashboard | 0024, 0023 |
| F-12.3 | Criar, agendar, desagendar, cancelar e enviar campanha com step-up | api, dashboard | 0024, 0016 |
| F-12.4 | Público congelado no envio e aviso no wpda (caixa, selo, lido) | api, wpda | 0024 |
| F-12.5 | Preferências do cidadão: opt-in de SMS e silenciar avisos | api, wpda | 0024, 0017 |
| F-12.6 | Envio de SMS: gateway plugável, janela 8h–20h e estado por destinatário | api | 0024 |
| F-12.7 | Painel da campanha: agregados e alertas de SMS | api, dashboard | 0024 |

## Riscos herdados

- **Subtração entre campanhas:** públicos quase iguais podem revelar, pela
  diferença, um grupo menor que 5; aceito no Ciclo 1, a rever com o módulo 14.
- **Fuso fixo** em `America/Sao_Paulo` (api#27): a janela do SMS herda.
- **Chave ligada sem provedor:** SMS vira `unavailable`, sem reenvio.
- **`tier` é texto do protocolo:** renomear a faixa separa triagens antigas e
  novas no critério.

## Critério de fechamento do módulo

- F-12.1 a F-12.7 verificadas.
- Suíte de invariante (`spec/invariants/campaign_invariants_spec.rb`): mínimo
  de 5 telefones; revogado fora do público; nenhum SMS sem opt-in e chave;
  texto fixo sem identificador; nenhuma lista de destinatários no dashboard;
  campanha enviada imutável; anonimização apaga destinatários; eventos sem PII.
- Runbook [`operacao/rollout-campanhas.md`](../operacao/rollout-campanhas.md).

## Histórico

- 2026-09-29 — Escopo decidido com o usuário; ADR 0024 e spec
  `superpowers/specs/2026-09-29-module-12-campaigns-design.md`. F-12.1 a
  F-12.7 criados. Módulo passa de `Stub` a `Planejado`.
- 2026-09-29 — F-12.1 a F-12.7 entregues e publicados: api `4425347` (suíte
  2742/0; invariantes com mutação), dashboard `c69a932` (667 testes), wpda
  `32e6420` (249 testes). Prova no navegador feita com o usuário (construtor
  com prévia e "menos de 5", envio com step-up, aviso no wpda com CPF
  mascarado e selo, preferências, SMS no log com texto fixo, alerta de SMS sem
  provedor). Decisões da execução no ADR 0024, seção Revisão. Módulo passa a
  `Entregue`; falta a verificação (dossiê por F-ID).
- 2026-09-30 — Verificação do módulo (dossiê por F-ID,
  [`relatorios/2026-09-29-verificacao-modulo-12.md`](../relatorios/2026-09-29-verificacao-modulo-12.md)):
  lacunas fechadas antes de subir — spec do `down`/`up` da migração, teste
  negativo do `appointment_no_show`, trava do `DispatchJob` com threads,
  `unavailable` fora da janela, evento quando o provedor cai no meio do lote e
  reenfileiramento do SMS parado com teto de 48 h (api `476084d`, suíte
  2760/0). F-12.1 a F-12.7 `Verified` por aprovação do usuário. Critério de
  fechamento cumprido; módulo `Fechado`.
