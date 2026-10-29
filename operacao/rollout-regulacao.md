# Rollout — regulação municipal (módulo 26a, ADR 0044)

Deploy, ligação por cidade, operação da fila e limites conhecidos da regulação municipal (api; dashboard, wpda e
maintenance seguem o api). Spec: `superpowers/specs/2026-10-28-module-26a-care-regulation-design.md`. Contrato:
`superpowers/plans/2026-10-28-module-26a-care-regulation-contracts.md` (§13 G1–G14).
Interruptor: `care_regulation` (exige `clinical_record`). Papel: `care_regulator`.

## Pré-requisitos

- **`clinical_record` ligado e utilizável.** Sem ele o interruptor fica "ligado, falta pré-requisito" e as rotas
  respondem 403 (`feature_disabled`; `feature: "clinical_record"` nas rotas da consulta).
- **Unidades e prestadores com CNES.** A guia em PDF e o prestador usam o CNES. Prestador da rede própria (`own`)
  exige unidade ativa e herda o CNES dela; CNES e unidade não se repetem entre prestadores.
- **SIGTAP da competência importada.** Sem ela, procedimento é recusado como alvo (`invalid_target`).
- **Órgão de saúde da cidade preenchido** (`City.health_department_name`): é o cabeçalho da guia impressa.
- **Quem terá o papel `care_regulator`.** Papel privilegiado: concedido pelo `municipal_admin` **com step-up**; o
  mantenedor não concede. Quem tem o papel vê nome, CID e justificativa de todos os pedidos da cidade e autoriza vaga.
- **Worker na imagem nova.** O consumidor dos eventos do módulo 17 (`CareRegulation::AppointmentSyncJob`) só existe na
  imagem nova; worker antigo falha com `ActiveJob::UnknownJobClassError`.
- Nenhuma gem nova.

## Ordem de deploy

Migração de **cidade** `20261028900001_add_care_regulation` **irreversível** (o `down` levanta
`IrreversibleMigration`): não há rollback de schema, só do código (expand/contract). Não há migração de plataforma.
Publicar a imagem e rodar o `bin/migrate` **dela**, antes de cortar o tráfego. Nunca migrar cidade fora do rake.

1. **api**: publicar a imagem (api e worker) e rodar o `bin/migrate` da imagem (`db:migrate`, `platform:triggers`,
   `city:migrate:all`). **Fora do pico**: a migração revalida `appointment_requests` (módulo 17) sob `ACCESS
   EXCLUSIVE`, na mesma transação.

   ```bash
   bin/migrate
   ```

   - Cria `regulation_requests`, `regulation_request_events`, `regulation_authorizations`, `care_providers`,
     `care_offers`, `regulation_risk_deadlines` e `nurse_requestable_items`, com os triggers de guarda
     (`db/city_triggers.sql`).
   - **Mexe em `appointment_requests`**: três `replace_check` reescrevem os CHECK de tipo, origem e linha de cuidado
     (`kind = 'regulated'`, `origin_regulation_request_id`, índice de no máximo um pedido vivo por pedido de
     regulação). Linhas existentes continuam válidas.
   - **Mexe em `attendances`** (módulo 18): acrescenta `referral_regulated boolean NOT NULL DEFAULT false` e reescreve
     `ck_attendances_referral` (G13). Linhas existentes continuam válidas.
   - Acrescenta `care_regulator` ao CHECK `ck_memberships_role`.
   - É mudança transversal: rode junto os specs do 17 (agendamento, agenda, check-in, `Drain`) e a suíte completa antes
     do merge.

2. **dashboard** (depois do api), **wpda**, **maintenance** (por último; rótulo do interruptor e do módulo `26a`).
3. **Antes de ligar**, preparar a cidade (admin municipal, com step-up; ver "Preparar a cidade").
4. **Só depois**, ligar `care_regulation`, **por cidade**, no maintenance.
5. Conceder o papel `care_regulator` à regulação da cidade (Equipe, com step-up).

## Preparar a cidade (antes de ligar)

- **Prestadores** (`/regulation/admin/providers`): próprios (`own`, com a unidade), contratados, consórcio e PPI, cada
  um com CNES.
- **Ofertas do mês** (`/regulation/admin/offers`): prestador, alvo, mês e quantidade. Só a oferta do mês corrente no
  fuso da cidade é autorizável; o mês passado não se cadastra.
- **Prazos por cor** (`/regulation/admin/deadlines`). **Cidade nova, provisionada agora, nasce sem linhas em
  `regulation_risk_deadlines`**, e o `Deadlines` cai nos padrões 7/30/90/180 dias (vermelho, amarelo, verde, azul).
  Funciona, mas a **coordenação deve revisar os prazos antes de ligar o interruptor**; mudar a tabela depois não mexe
  em pedido já feito.
- **Lista do enfermeiro** (`/regulation/admin/nurse_items`): os alvos que o enfermeiro pode pedir. Enfermeiro (2235*)
  só pede o que está na lista; médico (225*) pede qualquer alvo; dentista (2232*) só especialidades 2232*.

## Conferências pós-deploy

- Prazos presentes (vazio em cidade sem o `ensure_defaults!` da semente; o `Deadlines` usa os padrões mesmo assim):

  ```sql
  SELECT color, days FROM regulation_risk_deadlines ORDER BY days;
  ```

- Triggers presentes em **cada banco de cidade**:

  ```sql
  SELECT tgname FROM pg_trigger
   WHERE tgname LIKE '%regulation%' OR tgname LIKE '%care_offer%' OR tgname LIKE '%care_provider%';
  ```

- CHECKs novos e reescritos:

  ```sql
  SELECT conname FROM pg_constraint
   WHERE conname IN ('ck_appointment_requests_regulated', 'ck_attendances_referral', 'ck_memberships_role');
  ```

- `care_regulator` aceito no CHECK de papéis (`pg_get_constraintdef` de `ck_memberships_role`).
- **Em staging**, um pedido de teste ponta a ponta: pedido na consulta, finalização, autorização em oferta de unidade
  própria, horário no módulo 17 e check-in (a sincronização leva o pedido a `scheduled` e `completed`).
- Execuções falhas de `CareRegulation::AppointmentSyncJob` no Solid Queue: nenhuma.

## Operação

- **Ofertas são mensais.** Cadastrar a do mês seguinte **antes da virada**. A oferta de outro mês é 422
  `invalid_offer`. Cota esgotada trava a autorização daquele alvo (409 `quota_exhausted`).
- **Cota e falta.** A falta **não devolve a vaga**: a primeira volta o pedido à fila com `misses: 1`, a segunda
  encerra (`closed`). Uma autorização que já teve falta contada **não devolve a vaga em nenhum cancelamento**,
  seja do cidadão, da unidade, do consentimento ou do regulador. Reautorizar depois da falta consome de novo.
- **Cancelar com horário marcado na rede própria** exige cancelar o horário antes, no módulo 17 (422
  `appointment_scheduled`), pelo cidadão (wpda) ou pela recepção; a falta ou a expiração do horário também o reabre.
- **Janela estreita na rede própria** (cancelar × esvaziar a unidade com `HealthUnits::Drain`): um pedido cancelado
  pode ficar com horário marcado vivo no 17, porque a rede de segurança só encerra pedido de agendamento ainda
  `open`. A **recepção cancela esse horário pelo 17**; o atendimento ocorrido acima da cota é aceito.
- **Encaminhado.** Com a regulação ligada, "encaminhado" sem unidade passa a valer com um pedido de regulação
  presente (a flag `attendances.referral_regulated`, gravada pelo fechamento do atendimento). O "encaminhado" da
  **escuta** (módulo 18) continua exigindo unidade ou nota: pedido de regulação só nasce na consulta. O documento
  clínico assinado **ainda não lista** os pedidos de regulação (pendência registrada).
- **CSV do estado** (`GET /regulation/exports/state.csv`, nome `regulacao-estado-AAAA-MM-DD.csv`): é para
  **digitação manual** no sistema do estado. Leva justificativa, nome, nascimento e CNS só com os 4 últimos dígitos, e
  **nenhum CPF**. **Não tem limite de volume**: use o status padrão `external`; o histórico completo
  (`completed`/`closed`) pode ser grande.
- **Guia em PDF** (`GET /regulation/requests/:id/guide.pdf`, nome `guia-regulacao-<8 primeiros do id>.pdf`): leva o
  **CNS completo** (o prestador identifica pelo Cartão SUS) e o órgão de saúde da cidade, e **nunca o CPF**. No JSON
  de tela o CNS vai **mascarado** (`cns_masked`). Quem lê: quem pediu, a unidade solicitante e o regulador.
- **Campanhas.** O público sem filtro de tipo passa a incluir pedidos de agendamento `regulated`. A coordenação deve
  saber disso antes de disparar campanha de agendamento; o filtro de tipo aceita `regulated`.
- **Catálogo de alvos.** Especialidades: CBO `225*` e `2232*`, com rótulo de exibição ("Cardiologia", "Endodontia");
  patologia, radiologia, medicina nuclear, endoscopia e patologia bucal ficam fora, e hemoterapia entra.
  Procedimentos: SIGTAP dos grupos 02–04 **com instrumento BPA** (G11). Exame ativo no catálogo do 21a é recusado
  como regulação (`invalid_target`, `hint: "exam_order"`): segue pelo pedido de exame e pela fila do 21a.
- **Fila**: vencidos primeiro, depois cor, depois data do pedido. A cor efetiva é a confirmada pelo regulador ou, até
  lá, a sugerida; pular um pedido mais antigo exige motivo (`skip_reason`).
- **Sincronização com o 17** independe do interruptor: pedidos `regulated` já existentes continuam sendo reconciliados
  com o interruptor desligado.

## O que o módulo NÃO faz

- **Não escreve no SISREG, no e-SUS Regulação, no CARE nem no CROSS.** O regulado pelo estado é marcado aqui
  (`external`, com sistema e número) e acompanhado à mão; o CSV é para digitação.
- Não faz APAC, OCI nem teto financeiro em dinheiro (módulo 26b).
- Não faz MIRA (módulo 26c).
- A fila de cirurgia eletiva por AIH fica para o 26b.
- Não mostra ao cidadão cor, posição na fila, CID nem justificativa (o wpda mostra só a situação e o agendamento).

## LGPD

- **O pedido de regulação é registro clínico e retém o pedido de exclusão** (`Citizens::RequestErasure`), em qualquer
  situação, rascunho inclusive. Nada é apagado nem anonimizado por esse pedido; informar o titular de que a retenção é
  por obrigação legal.
- CPF, CNS, nome, CID, justificativa, motivos, alvo do paciente e notas nunca vão para log, evento, Analytics, mensagem
  de erro ou URL; os eventos levam só ids. Justificativa, CID e motivos são cifrados por cidade.
- Leitura nominal (fila, detalhe, guia, CSV) grava `clinical_record.viewed` com `access: "regulation"`; contagens
  não gravam trilha.

## Desligar e reverter

- Desligar `care_regulation` (maintenance): a fila some, as rotas respondem 403 `feature_disabled`, e o "encaminhado"
  volta à regra de hoje (unidade ou nota). Os pedidos, as ofertas e a sincronização com o 17 **continuam**; **nada é
  apagado**. Rascunhos ficam como estão na finalização.
- Atendimento já fechado como encaminhado por regulação continua válido (a flag fica gravada).
- As migrações **não voltam**: sem rollback de schema; volte só o código (expand/contract). O papel `care_regulator`
  segue aceito no CHECK.

## Gates de go-live (regulação da cidade piloto)

- **Catálogo de alvos e lista do enfermeiro conferidos pela regulação da cidade piloto** (patologia bucal e
  endoscopia ficam fora por proposta; a curadoria final é da regulação).
- **Prazos por cor aprovados** pela coordenação (os padrões 7/30/90/180 valem até lá).
- **Guia impressa conferida pela recepção do prestador** (inclui o CNS completo e o órgão de saúde).
- Prestadores com CNES, ofertas do mês e do mês seguinte cadastradas.
- Quem recebe o papel `care_regulator` definido e concedido pelo admin com step-up.
- Recepção orientada sobre o horário vivo de pedido cancelado e sobre a falta que não devolve a vaga.
- Coordenação ciente de que o público de campanha sem filtro inclui pedidos `regulated`.
