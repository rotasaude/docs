# Chamada, retorno e agendamento (canal web, subprojeto 4) — design

**Data:** 2026-09-25
**Status:** aprovado em conversa (2026-09-25), aguardando revisão do texto
**Afeta:**
- `apps/api`:
  - banco de cada cidade: `appointment_requests` e `appointments` novas, ajustes em `attendances` e `citizen_verification_codes`;
  - papel `health_professional`;
  - rotas `/attendance/*` e `/citizen/appointments*`;
  - jobs de expiração e falta.
- `apps/dashboard`: módulo "Atendimento" por papel, com fila em duas partes, pedidos e agenda do dia.
- `apps/wpda`: "Seus agendamentos", com confirmar, cancelar e "Cheguei" no dia.

**Card:** [rotasaude/api#9](https://github.com/rotasaude/api/issues/9) · **ADR:** `docs/adr/0019.md`

**Origem:** o subprojeto 3 abriu o atendimento no check-in e o encerrou com um desfecho, mas deixou dois buracos.
1. **Falta o passo "ser atendido".** O atendimento fica aberto até alguém registrar o desfecho, sem saber quem está sendo atendido.
2. **O encaminhamento termina num texto.** Não vira um horário.

O pedido: quem chega depois de uma triagem continua sendo atendido naquele dia, sem agenda. A unidade chama o cidadão da fila, e só **depois** do atendimento nasce, quando for o caso, um horário marcado de retorno ou de encaminhamento.

## 1. Ponto de partida

- **Subprojeto 3 (ADR 0018):**
  - `health_units` mínima;
  - `attendances` com `triage_id` único e NOT NULL, `status` `open | closed` e desfechos `discharged | referred | left`, com encaminhamento descritivo;
  - check-in por código (`citizen_verification_codes.purpose = 'check_in'` + `triage_id`) ou por exceção de CPF;
  - papel `citizen_verifier` faz tudo;
  - unidade escolhida guardada no `localStorage`.
- **Subprojeto 2:** cidadão `verified`, balcão de validação e código do balcão (6 dígitos, 10 min, 5 tentativas, um ativo por cidadão).
- **Módulo 08 (agendamento):** é um esboço, sem agenda de vagas. O módulo 10 (profissionais) não existe.
- **Fuso:** `config.time_zone = "America/Sao_Paulo"`. Os jobs recorrentes de cidade ficam em `config/recurring.yml`.

## 2. Decisões

1. **Chamada (parte B):** papel novo **`health_professional`**, privilegiado.
   - Ele chama o cidadão da fila (`waiting → in_care`) e registra o desfecho.
   - O `citizen_verifier` fica na recepção: check-in, validação e pedidos de agendamento.
   - Recepção e profissional podem marcar "saiu sem atendimento" em quem ainda aguarda.
2. **Unidade do profissional:** escolhida no início do turno (seletor guardado no `localStorage`, como o atendente). Não há vínculo formal com unidade (módulo 10).
3. **Desfechos que geram pedido:**
   - `return`, novo: retorno na mesma unidade;
   - `referred` com unidade de destino, a própria ou outra.
   - `referred` só com descrição não gera pedido.
4. **Quem marca (opção 2):** o desfecho cria um **pedido de agendamento** na unidade de destino, e a recepção de lá escolhe data e hora à mão.
   - Não existe agenda de vagas: fica para o módulo 08, que depois só acrescenta a escolha de vaga.
5. **Cidadão no wpda:** **vê, confirma ou cancela**. Cancelar exige motivo de pelo menos 10 caracteres. Não remarca.
6. **Confirmação com prazo:** o cidadão confirma até **24h antes** do horário. Sem confirmação no prazo, o horário é **cancelado automaticamente** e a unidade vê isso na fila de pedidos.
7. **Horário próximo nasce confirmado:** se faltarem **menos de 48h** quando a unidade marca, o horário já nasce `confirmed` e sem prazo. Todo horário que exige confirmação dá ao cidadão pelo menos 24h para confirmar.
8. **Depois do cancelamento:**
   - pelo cidadão, com motivo: o **pedido se encerra**;
   - pelo prazo (`expired`) ou por falta (`no_show`): o **pedido volta para a fila** da unidade, marcado "sem confirmação" ou "faltou". A recepção remarca ou encerra o pedido com justificativa.
9. **Modelo (abordagem A):** pedido (`appointment_requests`) e horário (`appointments`) em tabelas separadas.
   - O atendimento nasce de uma **triagem ou de um horário**.
   - Remarcar cria um horário novo, nunca edita o anterior.
10. **Check-in de um horário:** pelo mesmo balcão (código ou exceção).
    - O horário precisa estar `confirmed`, ser **hoje** no fuso da cidade e ser **desta unidade**. Horário de outra unidade é recusado, dizendo qual.
    - A falta só é marcada depois que o dia termina, então quem chega atrasado ainda é aceito.
11. **Nível do cadastro:** o pedido e o horário pertencem ao par (CPF, celular) do atendimento de origem, e esse par os vê no wpda, seja `declared` ou `verified`.
    - O "agendar" do ADR 0017, que exige `verified`, é o **cidadão marcando por conta própria**, e continua fora do escopo.
12. **Prioridade na fila:** o atendimento vindo de um horário entra com a prioridade da **triagem raiz** da cadeia.

## 3. Dados (banco de cada cidade)

- **`appointment_requests`** (nova):
  - `origin_attendance_id` (FK `attendances`, **único**), `citizen_id` (FK), `root_triage_id` (FK `triages`: a triagem que começou a cadeia, copiada da origem);
  - `origin_unit_id` e `target_unit_id` (FK `health_units`), `kind` (`return` | `referral`), `note` (texto, opcional);
  - `status` (`open` | `scheduled` | `closed`, padrão `open`) e `reopened_reason` (`expired` | `no_show` | nulo), que marca a volta à fila;
  - `closed_reason` (`fulfilled` | `citizen_cancelled` | `dismissed`), `dismiss_reason` (texto), `closed_by_user_id` (FK users, nulo quando é o cidadão ou o sistema) e `closed_at`;
  - `created_at` e `updated_at`.
  - **CHECKs:**
    - `kind = 'return'` ⇒ `origin_unit_id = target_unit_id`;
    - `status = 'closed'` ⇔ `closed_reason` e `closed_at` preenchidos;
    - `closed_reason = 'dismissed'` ⇔ `dismiss_reason` com pelo menos 10 caracteres depois de `btrim`.
  - **Trigger:** recusa `DELETE`, recusa mudar um pedido `closed` e recusa mudar as colunas de origem (`origin_*`, `citizen_id`, `root_triage_id`, `target_unit_id`, `kind`, `note`, `created_at`).
- **`appointments`** (nova):
  - `request_id` (FK), `citizen_id` (FK) e `health_unit_id` (FK, sempre igual a `target_unit_id` do pedido);
  - `scheduled_at` e `scheduled_by_user_id` (FK users);
  - `confirmation_deadline_at`: vale `scheduled_at − 24h` quando a marcação acontece com 48h ou mais de antecedência. Fica nulo quando o horário nasce confirmado.
  - `status`: `scheduled`, `confirmed`, `checked_in`, `cancelled_by_citizen`, `expired` ou `no_show`;
  - `confirmed_at`, `cancel_reason` e `ended_at`;
  - `created_at` e `updated_at`.
  - **CHECKs:**
    - `status = 'scheduled'` ⇒ `confirmation_deadline_at IS NOT NULL`;
    - `cancelled_by_citizen` ⇔ `cancel_reason` com pelo menos 10 caracteres depois de `btrim`;
    - os estados finais (`checked_in`, `cancelled_by_citizen`, `expired` e `no_show`) ⇔ `ended_at` preenchido.
  - **Índice único parcial:** um só horário vivo (`scheduled` ou `confirmed`) por `request_id`.
  - **Trigger:**
    - recusa `DELETE`;
    - aceita só as transições `scheduled → confirmed | cancelled_by_citizen | expired` e `confirmed → checked_in | cancelled_by_citizen | no_show`;
    - um horário em estado final não muda mais;
    - `request_id`, `citizen_id`, `health_unit_id`, `scheduled_at`, `scheduled_by_user_id`, `confirmation_deadline_at` e `created_at` nunca mudam.
- **`attendances`** (existe):
  - `triage_id` passa a aceitar nulo, e o índice único continua.
  - Entra `appointment_id` (FK `appointments`, **único**), com CHECK de **exatamente um** entre `triage_id` e `appointment_id`.
  - Entram `called_by_user_id` (FK users) e `called_at`.
  - `status` passa a ser `waiting | in_care | closed`. A migração troca `open` por `waiting`.
  - `outcome` ganha `return`, e a nota do retorno usa `referral_note`.
  - **CHECK de encerramento:**
    - `in_care` ⇔ `called_*` preenchidos e sem desfecho;
    - `closed` com `outcome = 'left'` pode vir sem `called_*`;
    - `closed` com qualquer outro desfecho exige `called_*`.
  - **Trigger:** passa a aceitar só `waiting → in_care`, `waiting → closed` (apenas `left`) e `in_care → closed`. `appointment_id`, `called_by_user_id` e `called_at` entram nas colunas que não mudam depois de preenchidas.
- **`citizen_verification_codes`** (existe): entra `appointment_id` (FK). O CHECK passa a ser `purpose = 'check_in'` ⇔ exatamente um entre `triage_id` e `appointment_id`.
- **Sem CPF nem celular nas tabelas novas:** nada entra em `CityEncryption::CITY_KEYED_TARGETS`.
- **`db/city_triggers.sql`:** as triggers novas vão guardadas por `to_regclass`, como as dos subprojetos 2 e 3.

## 4. Rotas

```
dashboard (sessão da cidade; health_unit_id vem da unidade escolhida)
  GET  /attendance/units/:id/queue                   → {waiting: [...], in_care: [...]}   (substitui /open)
       item: {id, cpf_masked, checked_in_at, protocol_name, priority, source: "triage"|"appointment",
              appointment_time?, called_at?, called_by_name?}
  POST /attendance/attendances/:id/call              → 200 {attendance}          health_professional
  POST /attendance/units/:id/call_next               → 200 {attendance} | 404 queue_empty   health_professional
  POST /attendance/attendances/:id/close {outcome, referral_unit_id?, referral_note?}
       left: citizen_verifier ou health_professional, só a partir de waiting
       discharged | referred | return: health_professional, só a partir de in_care
       → 200 {attendance, appointment_request?}
  GET  /attendance/units/:id/requests                → [{id, kind, origin_unit_name, created_at, cpf_masked,
                                                         priority, note?, reopened_reason?}]   citizen_verifier
  POST /attendance/requests/:id/appointments {scheduled_at}
                                                     → 201 {appointment: {id, scheduled_at, status,
                                                         confirmation_deadline_at}}   citizen_verifier
  POST /attendance/requests/:id/dismiss {reason}     → 200   citizen_verifier
  GET  /attendance/units/:id/agenda?date=AAAA-MM-DD  → [{id, scheduled_at, cpf_masked, kind, status}]
  POST /attendance/check_ins/lookup · check_ins · check_ins/search · check_ins/exception
       ganham o caminho do horário: lookup devolve `appointment` no lugar de `triage` quando o código
       aponta para um horário; search devolve {triages: [...], appointments: [...]}; exception aceita
       `triage_id` ou `appointment_id`

wpda (sessão do cidadão; só pares do celular da sessão, senão 404)
  GET  /citizen/appointments                         → [{request: {id, kind, target_unit_name, status,
                                                         closed_reason?, reopened_reason?},
                                                         appointment?: {id, scheduled_at, status,
                                                         confirmation_deadline_at?, check_in_available}}]
  POST /citizen/appointments/:id/confirm             → 200
  POST /citizen/appointments/:id/cancel {reason}     → 200
  POST /citizen/appointments/:id/check_in_code       → 201 {code, expires_at}
  GET  /citizen/triages → o `attendance` de cada triagem ganha `called_at` e `request_kind?`
```

**Regras das rotas:**
- **Chamar:** `call` e `call_next` exigem atendimento `waiting` da unidade informada, e travam a linha antes de conferir. Se dois profissionais chamarem o mesmo atendimento, um vence e o outro recebe 409 `already_called`. O `call_next` pega o primeiro por prioridade e depois por chegada.
- **Encerrar:** `close` com `return` cria o pedido com destino igual à unidade do atendimento. Com `referred` e `referral_unit_id`, cria o pedido para a unidade de destino. As duas coisas acontecem na mesma transação do encerramento.
- **Marcar horário** (`appointments`):
  - só com pedido `open`;
  - `scheduled_at` precisa estar no futuro e a no máximo 180 dias;
  - calcula o prazo de confirmação pela decisão 7;
  - o pedido vai para `scheduled` e `reopened_reason` volta a nulo.
- **Encerrar pedido** (`dismiss`): só com pedido `open`. Grava `dismiss_reason` e o encerra como `dismissed`.
- **Cidadão:**
  - `confirm` vale só para `scheduled` antes do prazo;
  - `cancel` vale para `scheduled` ou `confirmed`, com motivo. Encerra o pedido como `citizen_cancelled`;
  - `check_in_code` vale só para `confirmed` no dia do horário. O código gerado tem `purpose = 'check_in'` e `appointment_id`.
- **Check-in de um horário:**
  - cria o atendimento com `appointment_id`;
  - o horário vai para `checked_in` e o pedido se encerra como `fulfilled`, tudo na mesma transação.
  - Com código e a caixa "Conferi o documento" marcada, o cadastro `declared` é validado junto, como no subprojeto 3.
- **Limites:**
  - as rotas de balcão mantêm os 30 a cada 10 minutos por servidor;
  - confirmar e cancelar entram no limite de escrita do cidadão;
  - o código de check-in do horário segue os 10 por hora.
- O CPF nunca vai na URL, e toda escrita é JSON.

## 5. Jobs (por cidade, a cada 15 minutos, em `config/recurring.yml`)

- **`ExpireUnconfirmedAppointmentsJob`:** pega os horários `scheduled` com `confirmation_deadline_at <= agora`.
  - O horário vai para `expired`, e o pedido volta a `open` com `reopened_reason = 'expired'`.
  - Publica `appointment.expired`.
- **`MarkNoShowAppointmentsJob`:** pega os horários `confirmed` cujo dia já terminou no fuso da cidade (`scheduled_at < início do dia de hoje`).
  - O horário vai para `no_show`, e o pedido volta a `open` com `reopened_reason = 'no_show'`.
  - Publica `appointment.no_show`.
- **Concorrência:** cada linha é tratada com `lock!` e o estado é conferido de novo. Se o cidadão confirmar, cancelar ou fizer check-in no mesmo instante, quem chegar primeiro vence, e o outro não faz nada nem levanta erro.

## 6. Telas

**dashboard — "Atendimento"**. Os painéis aparecem conforme o papel.
- **Recepção (`citizen_verifier`)** continua com o que já tem: unidade, check-in, balcão e histórico. A mudança:
  - **Fila:** a lista de abertos vira a fila, em duas partes: "Aguardando" (ordenada por prioridade e depois por chegada) e "Em atendimento" (com quem chamou e desde quando). Em "Aguardando", a recepção só marca "Saiu sem atendimento".
  - **Pedidos de agendamento:** lista os pedidos `open` da unidade: tipo ("Retorno" ou "Encaminhado de *unidade*"), data, CPF mascarado, prioridade, nota e a marca "novo", "sem confirmação" ou "faltou".
    - **"Marcar horário"** tem um campo de data e hora. Com menos de 48h, avisa "O horário nasce confirmado". Com 48h ou mais, avisa "O cidadão precisa confirmar até *dd/mm hh:mm*".
    - **"Encerrar pedido"** pede justificativa de pelo menos 10 caracteres.
  - **Agenda do dia:** os horários de hoje na unidade, com hora, CPF mascarado, tipo e estado (aguardando confirmação, confirmado ou check-in feito).
  - **Check-in:** o cartão do código e a busca da exceção mostram também "Agendamento *hh:mm* · retorno/encaminhamento".
- **Profissional (`health_professional`):** unidade, a mesma fila e as ações:
  - **"Chamar próximo"**, e **"Chamar"** em cada linha de "Aguardando";
  - **"Encerrar"** em cada linha de "Em atendimento", com os desfechos:
    - Atendido e liberado;
    - Encaminhado, com unidade de destino (entre as ativas, incluindo a própria) e/ou descrição;
    - Retorno, com nota opcional.
    - Quando o desfecho gera pedido, o painel avisa: "Gera pedido de agendamento na *unidade*";
  - "Saiu sem atendimento" em "Aguardando".
- **Semente de dev:** entra `profissional@<cidade>.demo` com TOTP fixo, como as contas do ciclo assinado (`SignatureCrew`).

**wpda — "Seus agendamentos"**: uma seção acima de "Minhas triagens", visível quando houver pedidos.
- Pedido aberto: "Retorno pedido na *unidade* — a unidade vai marcar o horário" (ou "Encaminhamento para *unidade*…").
- Horário `scheduled`: "Agendado: *qui, 02/10, 14h30* — *unidade*. Confirme até *qua, 01/10, 14h30*", com os botões **Confirmar** e **Cancelar**.
  - Cancelar abre um campo de motivo com pelo menos 10 caracteres e o botão "Cancelar agendamento".
- Horário `confirmed`: "Confirmado: *qui, 02/10, 14h30* — *unidade*". No dia, aparece **"Cheguei na unidade"**, que abre a mesma tela de código do subprojeto 3. Continua com **Cancelar**.
- Estados finais:
  - "Cancelado por você";
  - "Cancelado: sem confirmação no prazo — a unidade pode marcar outro horário";
  - "Você não compareceu — a unidade pode marcar outro horário";
  - "Atendido na *unidade*".
- **Em "Minhas triagens":** "Em atendimento" passa a distinguir "Aguardando atendimento" de "Em atendimento". O desfecho Retorno mostra "Retorno — veja em Seus agendamentos".
- **Padrão do wpda:**
  - texto interativo com pelo menos 18px e alvos de toque com pelo menos 48px;
  - os campos novos da API são opcionais na entrada e normalizados num só lugar;
  - datas no fuso da cidade.

## 7. Erros

| Situação | Resposta |
|---|---|
| Chamar atendimento que não está `waiting` | 409 `already_called` |
| "Chamar próximo" com a fila vazia | 404 `queue_empty` |
| Atendimento de outra unidade | 422 `wrong_unit` |
| Desfecho clínico a partir de `waiting`, ou `left` a partir de `in_care` | 422 `invalid_transition` |
| Recepção tentando registrar desfecho clínico, ou profissional fazendo check-in | 403 `forbidden` |
| Marcar horário em pedido que não está `open` | 409 `request_not_open` |
| Horário no passado ou a mais de 180 dias | 422 `invalid_time` |
| Encerrar pedido sem justificativa ou com justificativa curta | 422 `reason_too_short` |
| Confirmar depois do prazo | 409 `confirmation_closed` |
| Confirmar ou cancelar um horário já encerrado | 409 `appointment_ended` |
| Cancelar sem motivo ou com motivo curto | 422 `reason_too_short` |
| Código de check-in fora do dia do horário | 422 `not_today` |
| Check-in de horário de outra unidade | 422 `wrong_unit`, com `unit_name` |
| Horário ou pedido de outro celular | 404 |
| Erros do subprojeto 3 | iguais |

## 8. Segurança e LGPD

- `health_professional` entra em `PRIVILEGED_ROLES`: conceder exige step-up, e o mantenedor nunca concede.
- O profissional vê o que a fila já mostra (CPF mascarado, protocolo e prioridade). Respostas e relatório continuam fora do dashboard do atendimento.
- A recepção vê, nos pedidos, a nota do retorno ou do encaminhamento. Não há dado clínico.
- As tabelas são a prova, e os eventos são trilha. Todo evento carrega só ids:
  - `attendance.called`;
  - `appointment_request.created` e `appointment_request.closed`;
  - `appointment.scheduled`, `.confirmed`, `.cancelled`, `.expired`, `.no_show` e `.checked_in`.
  - Cada um é declarado com `DomainEvents.bind ..., to: []`.
- **Logs:** `cancel_reason`, `dismiss_reason` e `note` entram no filtro.

## 9. Testes

### 9.1 API (RSpec, TDD; request specs com `type: :request`)

- **Modelo:**
  - os CHECKs das três tabelas;
  - "exatamente um" entre triagem e horário, no atendimento e no código;
  - o índice de um horário vivo por pedido;
  - as triggers, com cada transição permitida e cada proibida;
  - a migração de `open` para `waiting`.
- **Chamada:**
  - `call` e `call_next`, pela ordem da fila;
  - dois profissionais chamando o mesmo atendimento resultam em uma chamada e um 409;
  - `wrong_unit`, e 403 por papel.
- **Encerramento:**
  - `left` só a partir de `waiting`;
  - os desfechos clínicos só a partir de `in_care`;
  - `return` e `referred` com unidade criam o pedido na mesma transação, e `referred` só com descrição não cria.
- **Marcação:**
  - antecedência exatamente em 48h, e 1 segundo antes e depois disso;
  - prazo calculado;
  - horário no passado e a mais de 180 dias;
  - pedido fora de `open`.
- **Cidadão:**
  - confirmar antes do prazo e 1 segundo depois;
  - cancelar com e sem motivo;
  - 404 para horário de outro celular;
  - código de check-in só no dia.
- **Check-in de horário:**
  - por código e por exceção;
  - `wrong_unit` e `not_today`;
  - validação junto para `declared`;
  - o pedido se encerra como `fulfilled`;
  - a prioridade vem da triagem raiz.
- **Jobs:**
  - `expired` e `no_show` devolvem o pedido à fila com o motivo;
  - a virada do dia no fuso da cidade (`travel_to`);
  - corrida entre job e confirmar ou check-in, com um vencedor só;
  - rodar duas vezes seguidas não muda nada.
- **Contrato:** nenhum passo muda `triages`, `consents`, relatórios nem métricas.
- **Guardas:**
  - paridade do schema de cidade;
  - trigger guardada por `to_regclass`;
  - eventos declarados e sem CPF;
  - `PRIVILEGED_ROLES`;
  - suíte completa com o worker parado.

### 9.2 dashboard (sem jest-dom) e wpda (jest-dom)

- **dashboard:**
  - painéis por papel;
  - a fila em duas partes;
  - "Chamar próximo" e "Chamar";
  - encerrar com cada desfecho e o aviso de pedido;
  - "Marcar horário" com os dois avisos (nasce confirmado ou prazo);
  - encerrar pedido com justificativa;
  - agenda do dia;
  - mensagem para cada erro.
- **wpda:**
  - cada estado do pedido e do horário;
  - confirmar;
  - cancelar com motivo;
  - "Cheguei" só no dia;
  - normalização dos campos opcionais.

### 9.3 Verificação em desenvolvimento

Em Curitiba, com as contas `profissional@curitiba.demo` e `admin@curitiba.demo` (recepção):
1. Check-in, chamar, encerrar como Retorno e ver o pedido.
2. Marcar o horário com 3 dias de antecedência e confirmar no wpda.
3. Marcar outro horário para amanhã e ver que nasce confirmado.
4. Forçar o prazo com `travel_to` no `rails runner` e ver o pedido voltar como "sem confirmação".
5. Fazer check-in do horário no dia.

Prints salvos em `evidencias/<data>-agendamento/`, com índice.

## 10. Fora do escopo

- Agenda de vagas publicada pela unidade (módulo 08, depois), e o cidadão escolhendo horário.
- Remarcação pedida pelo cidadão.
- Lembrete por SMS ou push.
- Vínculo formal do profissional com a unidade, especialidades e escala (módulo 10).
- Dado clínico (módulo 13).
- Encaminhamento para fora da rede da cidade como pedido.
- Horário de funcionamento da unidade (módulo 09).
- Expiração de atendimentos abertos esquecidos (pendência do subprojeto 3).
