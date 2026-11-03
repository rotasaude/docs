# Módulo 30 — Transporte sanitário e TFD — design

**Data:** 2026-11-03
**Status:** aprovado (2026-11-03)
**Afeta:**
- `apps/api`: cidade — frota (veículos e motoristas), destinos com distância, viagens (avulsas e recorrentes) com passageiros e acompanhantes, presença e fechamento da viagem; processo de TFD (laudo, autorização pela comissão, atendimento marcado no destino, ajuda de custo, pagamento registrado, prestação de contas); produção do grupo 08.03.01 no BPA-I (módulo 22); aviso de viagem ao cidadão; fonte `transport` da linha do tempo; painel; plataforma — interruptores `sanitary_transport` e `tfd`.
- `apps/dashboard`: central de transporte (frota, destinos, viagens, séries, lista de passageiros impressa), tela do motorista (celular), comissão de TFD (fila de laudos, autorização), ajuda de custo e prestação de contas, painel.
- `apps/wpda`: "Minhas viagens" (data, hora, embarque, destino, situação do TFD).
- `apps/maintenance`: interruptores (mecanismo genérico).

**ADR:** `docs/adr/0048.md` · **Módulo:** `docs/modulos/30--transporte-e-tfd.md` · **Pesquisa:** `docs/pesquisa/2026-11-03-transporte-e-tfd-30.md` · ADR 0020 (banco por cidade), 0026 (exclusão LGPD), 0031 (leitura do prontuário), 0035 (linha do tempo), 0044 (regulação, 26a), módulo 17 (agenda), módulo 22 (BPA)

**Fora desta entrega:** urgência e SAMU, ambulâncias de suporte B a D; rastreador GPS e aplicativo próprio do motorista (sem trabalho offline); contratos de frota, manutenção, peças e pneus; TFD interestadual pela CNRAC/SISCNRAC (entra como prestador externo, sem integração); empenho, adiantamento financeiro e pagamento na tesouraria (o sistema calcula e registra, não paga); compra de passagem integrada a empresa de ônibus ou aérea.

## 1. Intenção

A central de transporte da secretaria monta a viagem do dia com os pacientes que têm consulta, exame ou tratamento marcados em outra cidade, sem planilha, e o motorista sai com a lista de quem embarca e onde. Quem precisa de Tratamento Fora de Domicílio tem o laudo do médico avaliado pela comissão, a viagem e a ajuda de custo calculadas pela regra, e a produção sai no BPA sem digitar de novo.

**Por quê:** transporte está nos 15 editais analisados e o TFD em 11 (pesquisa de mercado, frente 6); é "mínimo competitivo"; nas prefeituras o TFD costuma ficar em planilha. O módulo 26a deixou transporte e TFD explicitamente para o 30.

**Sucesso:** toda viagem da frota sai com lista de passageiros do sistema, sem veículo ou motorista em duas viagens ao mesmo tempo; nenhum TFD é pago sem autorização da comissão e atendimento marcado no destino; a produção do TFD entra no BPA do mês; o cidadão sabe pelo wpda quando e onde embarcar.

**Antiobjetivos:** lista de passageiros com CID ou diagnóstico; TFD pago para menos de 50 km; passageiro sem atendimento marcado entrando sem motivo registrado; motorista com CNH vencida escalado sem aviso.

## 2. Decisões

| # | Decisão |
|---|---|
| 1 | **Um módulo só**: frota, viagem e TFD. Viagem e TFD dividem a mesma lista de passageiros (escolha do usuário, opção A). |
| 2 | **Dinheiro do TFD:** o sistema calcula a ajuda de custo devida pela regra, registra o pagamento feito (data, valor, forma, recibo, quem registrou) e a prestação de contas; empenho e pagamento ficam com a tesouraria (escolha do usuário, opção A). |
| 3 | **Motorista no dashboard:** tela que funciona no celular, com as viagens do dia, a lista de passageiros, embarque e falta, odômetro e abastecimento; sem aplicativo próprio, sem offline e sem GPS (escolha do usuário, opção A). |
| 4 | **Viagem sem atendimento marcado** (retorno, paciente antigo ainda sem processo) é permitida com motivo de lista fechada e texto, com trilha (escolha do usuário, opção A). |
| 5 | **Veículo:** placa, RENAVAM, modelo, tipo (van, micro-ônibus, ônibus, carro, ambulância tipo A; códigos RENEM quando houver), lotação de passageiros, vagas para cadeirante, CNES onde está cadastrado (Portaria SAES 4.904/2026 exige o cadastro no CNES), ativo. |
| 6 | **Motorista:** usuário da cidade com papel `driver`, com número, categoria e validade da CNH. Veículo com mais de 8 lugares além do motorista exige categoria D (CTB art. 143). CNH vencida bloqueia a escala; a 30 dias do vencimento, aviso na central. |
| 7 | **Sem sobreposição:** o banco recusa o mesmo veículo ou o mesmo motorista em viagens com intervalos de horário que se cruzam (exclusion constraint sobre saída prevista e retorno previsto). Lotação e vagas de cadeirante não podem ser excedidas. |
| 8 | **Destino:** município (código IBGE) com distância rodoviária em km a partir da sede, cadastrada pela cidade, e marca de região metropolitana. O prestador externo do 26a (`care_providers`) ganha o município. |
| 9 | **Viagem:** data, saída prevista, retorno previsto, veículo, motorista, rota com paradas em ordem (embarque e destino), situação (`planned confirmed in_progress done cancelled`). **Passageiro:** paciente, parada de embarque, destino de tratamento (prestador e endereço), hora do atendimento, se ocupa vaga, cadeira de rodas, acompanhante (nome, documento, motivo), origem do atendimento marcado. |
| 10 | **Atendimento marcado** vem de: pedido de regulação agendado (26a, `scheduled`), consulta da agenda (17) — só para viagem dentro do município (zona rural, distrito) —, TFD autorizado, ou série recorrente. Sem nenhum desses, vale a decisão 4. |
| 11 | **Acompanhante:** regra por cidade; padrão da CIT 13/2017 para o transporte eletivo (paciente até 15 anos ou acima de 60), outros casos com justificativa; no TFD, só com indicação médica no laudo (Portaria 55/1999, art. 7º). |
| 12 | **Série recorrente** (hemodiálise, radioterapia, outro tratamento contínuo): paciente, destino, dias da semana, horário, início e fim; um job cria as viagens das próximas semanas e põe o paciente na viagem do dia. Suspender a série tira o paciente das viagens futuras. |
| 13 | **Presença e fechamento:** o motorista marca embarcou ou faltou por passageiro e acompanhante; no fechamento, odômetro de saída e chegada, abastecimentos (litros, valor, posto) e despesas. Falta fica no histórico do paciente e aparece na central. |
| 14 | **Lista de passageiros impressa** (PDF) e tela do motorista: nome, parada de embarque, destino, hora do atendimento, acompanhante, cadeira de rodas; **nunca CID, diagnóstico, procedimento ou especialidade**. |
| 15 | **TFD — laudo:** feito por médico (CBO 225\*) a partir da consulta (19a) ou de um pedido de regulação: procedimento SIGTAP, CID-10, justificativa (por que não pode ser feito na cidade), indicação de acompanhante com motivo, tipo de transporte (frota, passagem terrestre, aéreo, fluvial). Cifrado. |
| 16 | **TFD — regras de recusa:** destino a menos de 50 km ou na mesma região metropolitana (limite configurável por cidade, padrão 50 km); procedimento da atenção básica; sem atendimento marcado no destino (data e hora) não há autorização. |
| 17 | **TFD — comissão:** papel novo `tfd_committee`, com segundo fator. Autoriza, devolve para complementar ou recusa, sempre com motivo; pode pedir exames ou documentos. Processo com situação `draft submitted returned authorized denied scheduled travelling completed cancelled` e histórico de eventos. |
| 18 | **TFD — ajuda de custo:** calculada por viagem a partir das datas de ida e volta: alimentação com pernoite (por noite fora) ou sem pernoite (ida e volta no mesmo dia), para paciente e para acompanhante; sem diária enquanto o paciente está internado no destino. Valor = tabela SIGTAP da competência + complemento da cidade (configurável por código). Passagem comprada é registrada com valor e trecho. |
| 19 | **TFD — pagamento e contas:** o servidor registra o pagamento feito (data, valor, forma, número do recibo); a prestação de contas guarda o comprovante de comparecimento do prestador (arquivo) e fecha o processo. Óbito durante o TFD: a despesa fica com a cidade de origem (art. 9º), sem regra especial além do registro. |
| 20 | **BPA:** a produção do TFD (grupo 08.03.01: deslocamento por unidade de 50 km ou 200 milhas por trecho e por pessoa; ajuda de custo por dia) entra no BPA-I pelo módulo 22, no padrão do laboratório (evento de domínio → job → `Billing::Record.insert!` com fonte nova `tfd`). O estabelecimento é o CNES da secretaria ou da central de regulação, cadastrado na configuração do TFD. Só o TFD gera BPA; a viagem eletiva comum não. |
| 21 | **Cidadão:** "Minhas viagens" no wpda com data, hora, embarque, destino (nome do prestador e cidade) e, no TFD, a situação do processo e a ajuda de custo devida; aviso na caixa do wpda e por SMS (com opt-in e janela de horário do 17) na véspera e quando a viagem muda. |
| 22 | **Ligações:** fonte `transport` da linha do tempo ("Viagem para <cidade>", sem procedimento no cidadão); painel com viagens, ocupação, faltas, km, TFD por situação e valores pagos (só contagens e somas). |
| 23 | **Interruptores:** `sanitary_transport` (frota e viagens) e `tfd` (requer `feature:sanitary_transport` e `feature:clinical_record`, pois o laudo nasce da consulta). Desligado, 403 `feature_disabled`. |

Decisões 1 a 4 são do usuário; as demais são do coordenador, apresentadas no desenho.

**Decisões do coordenador (revisão da spec):**
- **Destino estruturado:** o agendamento externo do 26a (`ScheduleExternal`) ganha `care_provider_id` opcional, e `care_providers` ganha o município (IBGE); o texto livre `schedule_place` continua. O destino do passageiro é sempre um `transport_destination` (município com km) mais o prestador ou endereço: vem do prestador quando o agendamento o tem; senão, a central escolhe. A unidade de saúde da própria cidade (17) é destino de viagem intramunicipal, sem km para TFD.
- **Papéis:** `transport_dispatcher` (central: frota, motoristas, destinos, viagens, séries, lista impressa, faltas; nunca lê o laudo) e `tfd_committee` (laudo, autorização, trechos, ajuda de custo, pagamento e prestação de contas) são privilegiados, com segundo fator. `driver` não é privilegiado e vê só as próprias viagens do dia, com os dados da decisão 14.
- **Trecho do TFD:** o processo tem um ou mais **trechos** (`tfd_legs`: ida ou volta, data, modal, distância, passagem com valor, ligação opcional ao passageiro de uma viagem da frota); tratamento continuado são vários pares de trechos no mesmo processo. A comissão registra a **internação no destino** (início e fim) no processo. A ajuda de custo (`tfd_allowances`) é calculada por par de trechos e pessoa, descontando os dias internado.
- **Viagem sem escala:** viagem `planned` pode existir sem veículo e motorista (a exclusion constraint ignora nulos); `confirmed` exige os dois, com CNH válida e categoria. O job da série põe o paciente na viagem do mesmo dia e destino que tenha vaga (e vaga de cadeirante, se for o caso); sem nenhuma, cria uma viagem `planned` sem escala; sem vaga em viagem confirmada, vira pendência na central, nunca excesso de lotação.
- **BPA do TFD:** a linha sai quando o trecho é cumprido (passageiro `boarded` na viagem da frota, ou trecho de passagem marcado como realizado), com `performed_on` = data do trecho; a ajuda de custo sai junto com a volta. Profissional da linha = médico autor do laudo (CBO médico, compatível com o 08.03.01). Trecho pela frota também gera o deslocamento por unidade de 50 km (a confirmar no SIGTAP oficial, gate). Estabelecimento: unidade de saúde com tipo novo `health_management` (secretaria ou central de regulação), que o `Billing::Establishment` aceita e a crítica `establishment_is_aps` não marca; o fechamento do 22 não muda.
- **Exclusão LGPD:** passageiro ligado a TFD fica retido com o processo; os demais passageiros são anonimizados na revogação, mantendo a contagem de presença e falta sem nome.
- **Avisos:** entrada do paciente na viagem, mudança da saída em 30 minutos ou mais, mudança da parada de embarque, cancelamento ou retirada, e lembrete na véspera; um aviso por evento e viagem, sem repetir; SMS só nos mesmos casos, com as regras do 17.

## 3. Dados

Cidade:
- `transport_vehicles`; `transport_drivers` (usuário, CNH: número cifrado, categoria, validade).
- `transport_destinations` (município IBGE, nome, km, região metropolitana); `care_providers` ganha `ibge_code`.
- `transport_trips` (exclusion por veículo e por motorista); `transport_trip_stops`; `transport_trip_passengers` (paciente, parada, destino, hora do atendimento, vaga, cadeira, acompanhante cifrado, origem: `regulation_request_id` | `appointment_id` | `tfd_case_id` | `series_id` | `exception` com motivo; presença); `transport_trip_logs` (odômetro, abastecimentos, despesas).
- `transport_series` (recorrência).
- `tfd_cases` (laudo cifrado, situação, destino, atendimento no destino, acompanhante, internação) e `tfd_case_events` (imutável); `tfd_legs` (trechos); `tfd_allowances` (calculadas por viagem e pessoa: código SIGTAP, quantidade, valor de tabela, complemento); `tfd_payments` (imutável); `tfd_accountability` (comprovante, fechamento).
- `performed_procedures.source` ganha `tfd`; `health_units.kind` ganha `health_management`; `memberships.role` ganha `transport_dispatcher`, `tfd_committee`, `driver`; `regulation_requests` ganha `care_provider_id` no agendamento externo.
- `transport_trip_notices` (aviso de viagem; o aviso do 17 é preso ao `appointment`).
- Configuração por cidade: distância mínima, regra do acompanhante, complemento por código, CNES do TFD, motivos de exceção.

## 4. Telas

- **Central de transporte:** veículos e motoristas (alerta de CNH), destinos, agenda de viagens por dia e veículo, montar viagem com passageiros sugeridos dos atendimentos marcados na data, séries recorrentes, lista de passageiros em PDF, faltas.
- **Motorista (celular):** viagens do dia → passageiros por parada → embarcou/faltou → fechamento.
- **Comissão de TFD:** fila de laudos, leitura do laudo, autorizar/devolver/recusar, marcar atendimento no destino, viagem ou passagem, ajuda de custo, pagamento, prestação de contas.
- **Consulta (19a):** botão "pedir TFD" que abre o laudo pré-preenchido.
- **Painel:** transporte e TFD.
- **wpda:** "Minhas viagens".

## 5. Segurança e LGPD

- Laudo (CID, justificativa) é dado de saúde: só a comissão e o médico autor leem, com trilha `clinical_record.viewed`.
- Central e motorista veem só o necessário à viagem (decisão 14); nada clínico na lista, na tela do motorista, no aviso ou no SMS.
- Painéis e prestação de contas agregadas; nada nominal em relatório de transparência.
- CPF, CNS, nome, CNH, documento do acompanhante, destino de tratamento e CID nunca em URL, log ou evento.
- Imutabilidade por trigger em `db/city_triggers.sql` (`tfd_case_events`, `tfd_payments`, presença depois do fechamento); novos `Platform.audit` em R18.
- Exclusão LGPD: o processo de TFD e o pagamento são retidos (prestação de contas ao controle do SUS, art. 15–16); viagens anonimizam o passageiro.

## 6. Testes

- api: sobreposição de veículo e motorista no banco; lotação e cadeira; CNH vencida e categoria; passageiro sem atendimento marcado exige motivo; série cria e suspende viagens; recusa de TFD abaixo de 50 km, na região metropolitana e sem atendimento marcado; cálculo da ajuda de custo (pernoite, mesmo dia, internado, acompanhante); unidades de 50 km por trecho; BPA-I com fonte `tfd`; aviso e SMS; lista sem dado clínico; trilha.
- dashboard e wpda: testes de componente; tela do motorista em largura de celular.
- Suíte completa do api no fim.

## 7. Entrega

api → dashboard → wpda → maintenance; runbook `operacao/rollout-transporte-e-tfd.md` (cadastro de veículos no CNES, motoristas e CNH, destinos e km, regras do manual estadual de TFD da UF, CNES do TFD para o BPA).

## 8. F-IDs (propostos)

| ID | Funcionalidade | Superfície |
|---|---|---|
| F-30.1 | Interruptores, veículos, motoristas e destinos | api, dashboard, maintenance |
| F-30.2 | Viagem, passageiros e lista sem dado clínico | api, dashboard |
| F-30.3 | Séries recorrentes (hemodiálise, radioterapia) | api, dashboard |
| F-30.4 | Tela do motorista, presença e fechamento | api, dashboard |
| F-30.5 | Laudo e comissão de TFD | api, dashboard |
| F-30.6 | Ajuda de custo, pagamento e prestação de contas | api, dashboard |
| F-30.7 | Produção do TFD no BPA-I | api |
| F-30.8 | Minhas viagens, avisos, linha do tempo e painel | api, wpda, dashboard |

## 9. Riscos e pendências

- **Manual estadual de TFD** varia por UF (laudo, valores, fluxos): o padrão segue a Portaria 55; o manual da UF da cidade piloto precisa ser lido no rollout.
- **SIGTAP 08.03.01:** instrumento BPA-I confirmado só para o 08.03.01.012-5 (fonte de terceiros); conferir no SIGTAP oficial todos os códigos, CBOs compatíveis e a contagem de unidades por trecho antes do go-live.
- **Estabelecimento do BPA do TFD** (CNES da secretaria ou da central) e a crítica `establishment_is_aps` do 22.
- **Distância:** km cadastrado pela cidade, não calculado; erro de cadastro muda o valor.
