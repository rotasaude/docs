# Módulo 32 — Urgência (UPA) — design

**Data:** 2026-11-05
**Status:** aprovado (2026-11-05)
**Afeta:**
- `apps/api`:
  - **cidade:** encontro de urgência sobre o atendimento do 13, paciente não identificado, classificação de 5 cores por protocolo assinado, atendimento médico, desfecho e contrarreferência, senha e chamada com local, painel de TV, BPA pelo 22, fonte `urgency` da linha do tempo, painel do gestor; no 32b, leitos de observação, prescrição interna, checagem e baixa de estoque;
  - **plataforma:** interruptores.
- `apps/dashboard`:
  - recepção da UPA, classificação, fila por cor com tempo estourado, atendimento médico, desfecho, cadastro de TV e locais de chamada, tela da TV em tela cheia, painel do gestor;
  - no 32b, mapa de observação, prescrição e checagem.
- `apps/wpda`: "Meu histórico" mostra o atendimento de urgência, sem tela nova.
- `apps/maintenance`: interruptores, pelo mecanismo genérico.
- `contracts`: novo `kind` de protocolo `urgency_classification` no schema de protocolos do 18.

**ADR:** `docs/adr/0050.md` · **Módulo:** `docs/modulos/32--urgencia.md` · **Pesquisa:** `docs/pesquisa/2026-11-05-urgencia-upa-32.md`. Relacionados:
- ADRs 0020, 0026, 0030 (escuta e protocolo assinado), 0031 e 0035;
- módulos 13 (atendimento e chamada), 19 (documentos), 21 (exames), 22 (BPA), 25 (vigilância) e 27a (estoque).

**Fora desta entrega:**
- SAMU e regulação de urgência;
- pronto-socorro hospitalar e internação;
- central estadual de leitos (a transferência só registra o pedido de vaga);
- farmácia hospitalar;
- fluxogramas licenciados do Manchester;
- voz sintetizada e mais de uma fila por TV;
- espera de cada UPA no wpda.

## 1. Intenção

**O que muda no dia a dia:**
- Quem chega à UPA é recebido em segundos, mesmo sem documento.
- A enfermagem classifica pela cor que o protocolo da cidade sugere.
- O médico vê a fila por gravidade e sabe quem já passou do tempo máximo.
- Todo atendimento termina com um desfecho registrado.
- Na sala de espera, a TV chama pela senha e explica por que o mais grave passa na frente.
- A UBS do paciente sabe que ele passou pela urgência e precisa de retorno.
- A produção sai no BPA.

**Por quê:**
- Urgência aparece em 10 dos 15 editais.
- A UPA é a porta mais cobrada pela população.
- Sem tempo por cor, a gestão não consegue cumprir nem provar a Portaria 10/2017.

**Sucesso:**
- Todo paciente tem cor e horário de classificação.
- O gestor sabe, por cor, quanto tempo se esperou.
- Nenhum atendimento fica sem desfecho.
- O painel público não expõe ninguém.

**Antiobjetivos:**
- Exigir CPF para atender.
- Painel com nome ou cor de pessoa.
- Cor alterada sem registro.
- Atendimento de urgência que some sem desfecho.

## 2. Decisões

| # | Decisão |
|---|---|
| 1 | **Protocolo:** a própria cidade cadastra o protocolo de classificação, com 5 cores e tempo máximo configurável por cor (padrão 0/10/60/120/240 min). Usa o motor de protocolo assinado do 18 (ADR 0030) com um `kind` novo, sem conteúdo licenciado (escolha do usuário, opção A). |
| 2 | **Medicação:** a prescrição interna tem aprazamento e checagem pela enfermagem. O interruptor do maintenance **`urgency_stock_deduction`** ("Atendimentos da UPA com prescrição de medicação atualizam o estoque") decide o que acontece com o estoque: ligado, cada administração dá baixa no estoque `unit` da UPA, pelo razão do 27a; desligado, a administração é só registrada e o estoque não muda (escolha do usuário, opção A com interruptor). |
| 3 | **Observação:** leitos de observação com mapa de ocupação e alerta ao se aproximar de 24 h (escolha do usuário, opção A). |
| 4 | **Cidadão:** painel de chamada para TV, com senha, local e tempo médio por cor, sem dado pessoal, para UPA e UBS. Mais o atendimento de urgência em "Meu histórico" no wpda (escolha do usuário, opção B). |
| 5 | **Entregas:** **32a** cobre chegada, classificação, atendimento, desfecho, painel de TV, histórico e BPA. **32b** cobre observação com leitos, prescrição, checagem e estoque (escolha do usuário, opção B). |
| 6 | **Encontro sobre o atendimento:** numa unidade `upa`, o check-in do 13 cria o `attendance` e, junto, um `urgency_encounter` (1:1). A fila, a chamada e o fechamento seguem o 13. O encontro guarda o que é clínico da urgência. |
| 7 | **Paciente não identificado:** a recepção abre o atendimento sem cidadão, com `unidentified = true` e uma descrição (sexo e idade aparentes, sinais, onde foi encontrado). `attendances.citizen_id` aceita nulo só nesse caso, por CHECK. A identificação posterior liga o cidadão uma vez só, com trilha. O desfecho e o BPA exigem identificação, salvo em óbito ou evasão. |
| 8 | **Classificação:** queixa, sinais vitais (como no 18), escala de dor, a cor sugerida pelo protocolo com o critério que levou a ela, e a cor final. Mudar a cor sugerida exige justificativa. Reclassificar é sempre possível e fica registrado (`urgency_classifications`, só acréscimos). A cor vigente é a última. |
| 9 | **Fila da UPA:** ordem por cor e, dentro da cor, por chegada. Cada paciente mostra o tempo de espera e o estouro do tempo máximo da cor. O "Chamar próximo" do 13 segue essa ordem nas unidades `upa`. Paciente sem classificação fica numa faixa "aguardando classificação" com o tempo desde a chegada. |
| 10 | **Atendimento médico:** anamnese, exame físico, hipótese com CID, conduta e evolução em texto cifrado. Exames de urgência pelo 21, com prioridade urgente. Documentos pelo 19 (receita, atestado, declaração), presos ao atendimento. |
| 11 | **Desfecho:** `discharged`, `discharged_to_primary_care` (contrarreferência), `transferred` (destino e pedido de vaga em texto, sem integração), `left_without_care` (evasão, antes ou depois da classificação), `left_against_advice`, `death`. Todo encontro termina com desfecho; o fechamento do 13 exige um. |
| 12 | **Contrarreferência:** no `discharged_to_primary_care`, a orientação ("procure sua UBS em até N dias", com motivo) vai à equipe da UBS do paciente (território do 11/24a) pela linha do tempo profissional e por um aviso interno. O cidadão vê a orientação no histórico. |
| 13 | **Senha e chamada (UPA e UBS):** o check-in gera uma **senha** por unidade e dia, com prefixo configurável (ex.: "U-042"). Na chamada (`Call`, `CallNext`), o profissional escolhe o **local**, um dos locais de chamada da unidade (Consultório 1, Sala de medicação…), lembrado na sessão. A chamada grava `attendance_calls` (senha, local, hora). "Chamar de novo" grava outra chamada sem mudar o estado. |
| 14 | **TV da sala de espera:** o admin cadastra o dispositivo da unidade e recebe um código de pareamento. A TV abre uma página do dashboard em tela cheia, fora do shell, autenticada pelo token do dispositivo e sem login de pessoa. A página mostra as últimas chamadas da unidade (senha → local) e, se a unidade escolher, o tempo médio de espera do momento por cor, com uma frase explicando a prioridade. Nada de nome, CPF, cor da pessoa ou queixa. Não grava trilha. Interruptor `waiting_room_display`. |
| 15 | **BPA (22):** o encontro gera `performed_procedures` com fonte `urgency`: o acolhimento com classificação de risco (03.01.06.011-8) pelo profissional que classificou e o atendimento médico (03.01.06.009-6) pelo médico. No 32b, também a observação (03.01.06.002-9). O padrão é evento → job → `Billing::Record.insert!`. O instrumento e os CBOs vêm da SIGTAP; conferir os códigos é gate. |
| 16 | **Vigilância (25):** CID do atendimento que esteja na lista de notificação compulsória gera a sugestão de notificação ao módulo 25, como já acontece na consulta. |
| 17 | **Linha do tempo e wpda:** fonte `urgency`, com unidade, data, hora, desfecho e orientação de retorno. Entra na visão do cidadão (`Timeline::KINDS`), sem CID, cor ou evolução. A visão profissional traz cor, CID e desfecho. Fica fora do LEDI (UPA não é APS). |
| 18 | **Painel do gestor:** tempos porta→classificação e classificação→médico por cor (mediana e p90), % fora do tempo máximo, atendimentos por cor e desfecho, evasão, permanência e, no 32b, ocupação da observação. Só contagens, com "menos de 5". |
| 19 | **Papéis:** os da unidade. A recepção usa `citizen_verifier`. A classificação fica com `health_professional` de enfermagem (CBO 2235 ou 3222) e o atendimento com o médico (225*), vinculados à UPA. A configuração do protocolo, dos locais de chamada e das TVs é do `municipal_admin`. Nenhum papel novo. |
| 20 | **32b — observação:** leitos de observação da UPA (número, ativo), com um ocupante por vez (exclusion), entrada e saída. Alerta aos 20 h e aos 24 h de permanência. Reavaliações e evolução no encontro. Desfecho ao sair da observação. |
| 21 | **32b — prescrição interna:** itens do catálogo do 27a, com dose, via, frequência e horários aprazados, prescritos pelo médico. A enfermagem checa cada horário: administrado (lote, quando o estoque estiver ligado), não administrado com motivo, ou recusado. Prescrição suspensa ou alterada vira versão nova. A administração não pode ser alterada. |
| 22 | **32b — estoque:** com `urgency_stock_deduction` ligado, a administração gera `stock_movements` com `kind = administration` no `stock_location` `unit` da UPA, lote por FEFO. Sem saldo, a administração é registrada e o estoque fica com pendência para a farmácia, nunca negativo. Desligado, não há movimento. |
| 23 | **Interruptores:** `urgency_care` (requer `feature:clinical_record`), `waiting_room_display` (independente; vale para UPA e UBS) e `urgency_stock_deduction` (requer `urgency_care` e `feature:pharmacy`). |

As decisões 1 a 5 são do usuário. As demais são do coordenador, apresentadas no desenho.

**Decisões do coordenador na revisão da spec.** Estas decisões prevalecem sobre as decisões 6, 7, 10, 13 e 22.

- **R1. Origem `walk_in`:**
  - `attendances` ganha a origem `walk_in`, aceita só em unidade `upa`. O CHECK `ck_attendances_origin` passa a aceitar exatamente um entre `triage_id`, `appointment_id` e `walk_in`.
  - A recepção da UPA tem um caminho próprio, `Urgency::Arrive`, que cria o atendimento e o encontro numa só transação. A ficha de chegada segue o 13 e passa pelas guardas do 31: nada em CAPS.
- **R2. O paciente é `Patient`, não `Citizen`:**
  - `attendances` ganha `patient_id`. Na origem `walk_in`, `citizen_id` pode ser nulo e `patient_id` é obrigatório.
  - A recepção acha ou cria o `Patient` por CPF ou CNS. A busca usa `person_lookup`, no padrão do 29, do 30 e do 31. Paciente sem CPF segue o `no_cpf_reason` do 24a.
  - O wpda e o "Meu histórico" acham o atendimento pelo CPF do paciente, como as demais fontes. Não é preciso criar o par CPF + telefone na recepção.
- **R3. O atendimento médico é uma `Consultation` do 19a de `kind: urgency`, presa ao atendimento:**
  - Herda rascunho, finalização, adendos, CID, exames do 21, documentos do 19 e vigilância do 25, sem nova origem.
  - O `urgency_encounter` guarda só o que é da urgência: classificações, estado, desfecho e, no 32b, observação e prescrição.
  - A consulta de urgência fica fora do LEDI.
- **R4. Paciente não identificado (substitui a decisão 7):**
  - A recepção cria um **`Patient` provisório**, com `no_cpf_reason: "unidentified"` e a descrição cifrada. Assim o médico atende, prescreve e pede exame sem esperar.
  - **Identificar** acontece uma vez só, com trilha `urgency.patient_identified`. Há dois casos:
    - se não existe paciente com aquele CPF ou CNS, o provisório recebe os dados;
    - se existe, os registros do atendimento passam para o paciente real (encontro, consulta, documentos, exames, BPA e, no 32b, prescrição e administrações) e o provisório fica marcado `merged_into`, sem apagar nada.
  - O BPA do atendimento só sai depois da identificação. Óbito e evasão sem identificação ficam pendentes no faturamento, sem linha.
- **R5. Na unidade `upa`, a classificação do 32 substitui a escuta do 18:**
  - O `screening_scope` da UPA fica em `none`.
  - A fila da UPA é uma variante do `UnitQueue`: ordena pelas 5 cores e mostra a faixa "aguardando classificação".
  - O protocolo `urgency_classification` fica fora do `triage_protocols` e de `Screenings::ActiveProtocol`.
- **R6. Senha e local da chamada:**
  - Só existem com `waiting_room_display` ligado na cidade. Desligado, o check-in e a chamada do 13 não mudam.
  - Ligado, o local é obrigatório na chamada das unidades que têm TV cadastrada.
- **R7. (32b) Falta de saldo:**
  - Gera uma linha em `urgency_stock_shortfalls`: item, quantidade, administração e local. Ela fica numa fila "Faltas da UPA" na tela da farmácia do 27a.
  - O razão (`stock_movements`) não muda. A farmácia resolve com o que o 27a já tem, por ajuste ou transferência, e marca a falta como conciliada.

## 3. Dados

**Cidade (32a):**
- `attendances`:
  - ganha `ticket` (único por unidade e dia, só com TV), origem `walk_in` e `patient_id`;
  - `citizen_id` aceita nulo na origem `walk_in`.
- `patients` ganha `merged_into_id`, e `no_cpf_reason` ganha o valor `unidentified`.
- `consultations.kind` ganha `urgency`.
- `urgency_stock_shortfalls` (32b).
- `urgency_encounters`: atendimento 1:1, texto clínico cifrado, CID, estado.
- `urgency_classifications`: só acréscimos, com cor, critério, sinais, justificativa e quem classificou.
- `urgency_outcomes`: tipo, destino, orientação e quando.
- `call_locations`: locais de chamada da unidade.
- `attendance_calls`: senha, local, hora e quem chamou.
- `waiting_room_displays`: unidade, nome, token cifrado, ativo e opção de mostrar o tempo médio.
- `performed_procedures.source` ganha `urgency`.
- Configuração: prefixo da senha por unidade e tempos máximos por cor (no protocolo).

**Cidade (32b):**
- `urgency_beds` e `urgency_bed_stays` (exclusion);
- `urgency_prescriptions`, `urgency_prescription_items` e `urgency_administrations`;
- `stock_movements.kind` ganha `administration`.

**contracts:** `kind = urgency_classification` no schema de protocolos (5 cores com tempos). Sai uma tag nova.

## 4. Telas

**32a:**
- **Recepção da UPA:** check-in rápido, paciente não identificado e senha.
- **Classificação:** queixa, sinais, cor sugerida e justificativa; reclassificação.
- **Fila da UPA:** por cor, com tempo e estouro; "aguardando classificação".
- **Atendimento:** médico, documentos e exames, desfecho.
- **Identificar paciente:** liga o cidadão ao atendimento.
- **Locais de chamada e TVs:** admin.
- **TV:** tela cheia.
- **Painel da urgência.**

**32b:**
- mapa de observação;
- prescrição;
- checagem da enfermagem.

## 5. Segurança e LGPD

- **Texto clínico cifrado:** queixa, evolução, conduta e descrição do não identificado. O CID também.
- **Fora de URL, log e evento:** nome, CPF, CNS, CID e queixa.
- **Painel público:** só senha, local e médias. O token da TV é revogável e não lê nada fora da própria unidade.
- **Identificação posterior:** trilha `urgency.patient_identified`. Um não identificado nunca é ligado a dois cidadãos.
- **Exclusão (ADR 0026):** o encontro é registro de saúde e fica retido. Atendimento com evasão antes da classificação segue a regra da triagem sem atendimento.
- **Imutabilidade:** classificações, desfechos, chamadas e administrações não podem ser alterados (trigger). Os novos `Platform.audit` entram em R18.

## 6. Testes

**api:**
- encontro nasce com o check-in na UPA, e só lá;
- não identificado e identificação única;
- cor sugerida pelo protocolo, justificativa e reclassificação;
- ordem da fila e "Chamar próximo" por cor;
- desfecho obrigatório no fechamento;
- contrarreferência;
- senha única por dia;
- chamada com local e "chamar de novo";
- TV: token, escopo da unidade e nada pessoal na resposta;
- BPA com fonte `urgency`;
- linha do tempo: cidadão sem CID e sem cor;
- vigilância;
- painel com "menos de 5";
- 32b: leito sem sobreposição, alertas, prescrição em versões, checagem, estoque ligado e desligado, sem saldo.

**dashboard e wpda:** testes de componente; tela da TV.

**Suíte completa do api no fim**, porque o módulo mexe em `attendances` (13) e em `performed_procedures` (22).

## 7. Entrega

- **32a:** contracts → api → dashboard → wpda → maintenance.
- **32b:** api → dashboard → maintenance.
- **Runbook** `operacao/rollout-urgencia.md`, com:
  - protocolo da cidade e revisão clínica;
  - tempos máximos;
  - locais de chamada e TVs;
  - códigos SIGTAP;
  - estoque da UPA e o interruptor.

## 8. F-IDs (propostos)

| ID | Funcionalidade | Entrega | Superfície |
|---|---|---|---|
| F-32.1 | Interruptores, protocolo de 5 cores e configuração da UPA | 32a | api, dashboard, maintenance, contracts |
| F-32.2 | Chegada, senha e paciente não identificado | 32a | api, dashboard |
| F-32.3 | Classificação de risco e fila por cor | 32a | api, dashboard |
| F-32.4 | Atendimento médico e desfecho com contrarreferência | 32a | api, dashboard |
| F-32.5 | Chamada com local e painel da TV (UPA e UBS) | 32a | api, dashboard |
| F-32.6 | BPA, vigilância, histórico no wpda e painel do gestor | 32a | api, dashboard, wpda |
| F-32.7 | Observação com leitos e alerta de 24 h | 32b | api, dashboard |
| F-32.8 | Prescrição interna, checagem e baixa de estoque | 32b | api, dashboard, maintenance |

## 9. Riscos e pendências

- **SIGTAP da urgência:** conferir o instrumento, os CBOs e os valores do 03.01.06.011-8, do 009-6 e do 002-9 no SIGTAP oficial.
- **Protocolo da cidade:** a semente de dev é exemplo. As regras de cor precisam de revisão clínica antes de cidade real, como no 18.
- **Licença do Manchester:** a cidade licenciada que cadastrar as regras dele responde pela licença.
- **Painel da TV:** o token do dispositivo precisa de rotação e revogação. A TV fica numa rede pública da unidade.
