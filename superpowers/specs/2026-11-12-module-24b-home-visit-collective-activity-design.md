# Módulo 24b — Visita domiciliar e atividade coletiva — design

**Data:** 2026-11-12
**Status:** aprovado pelo usuário (2026-11-12)
**Afeta:**
- `apps/api`:
  - **cidade:** folha de visitas do dia, visitas, atividades coletivas e participantes; fichas LEDI de visita domiciliar e territorial (tipo 8) e de atividade coletiva (tipo 6); visita como fonte das linhas de cuidado (23) e da linha do tempo (20); lista "quem visitar"; aparelhos de campo e as rotas de sincronização do app;
  - **plataforma:** interruptores `home_visits` e `collective_activities`.
- `apps/dashboard`: folha do dia e visita (no celular), atividade coletiva, "quem visitar", visitas da equipe, aparelhos de campo.
- `apps/wpda`: a visita realizada na linha do tempo e em "Meu acompanhamento".
- `apps/maintenance`: os dois interruptores, pelo mecanismo genérico.
- `rotasaude/contracts`: contrato de sincronização do app de campo (versão 1).

**ADR:** `docs/adr/0052.md` · **Módulo:** `docs/modulos/24--domiciliar-e-coletivo.md` · **Pesquisa:** `docs/pesquisa/2026-11-12-visita-e-coletiva-24b.md`
- **ADRs relacionados:** 0020 (banco por cidade), 0028 (modo de prontuário e LEDI), 0035 (linha do tempo), 0040 (linhas de cuidado), 0041 (núcleo e ferramenta), 0042 (cadastro territorial), 0047 (sala de vacina, precedente da fonte nas linhas de cuidado).
- **Módulos relacionados:** 10 (profissionais), 20 (linha do tempo), 23 (linhas de cuidado), 24a (cadastro).

**Fora desta entrega:**
- o app de campo offline (24c), que só consome o contrato;
- georreferenciamento do imóvel e latitude/longitude da visita (24c);
- Programa Saúde na Escola (PSE): cadastro de escolas com INEP e os campos de PSE da atividade coletiva (próximo ciclo);
- agendamento de visita;
- trabalho de campo do agente de endemias além dos campos da ficha de visita (módulo 35);
- ficha de Atendimento Domiciliar (eMulti e Atenção Domiciliar);
- IVCF-20 na visita (próximo ciclo, junto com a Nota Informativa 1/2026);
- papel de digitador.

## 1. Intenção

O ACS registra cada visita que faz e a equipe registra suas atividades coletivas, e isso chega ao SIAPS como ficha LEDI aceita. A visita passa a contar onde ela já é exigida: nas boas práticas das linhas de cuidado e na dimensão de Acompanhamento do cofinanciamento federal.

**Por quê:**
- A visita domiciliar e a atividade coletiva são práticas de cuidado da dimensão de Acompanhamento (70% do componente de Vínculo e Acompanhamento Territorial; NT 30/2025).
- Seis boas práticas do catálogo-base pedem visita do ACS (C2, C3 duas vezes, C4, C5 e C6). Hoje elas aparecem como "não acompanhado" (`acs_visit` em `UNTRACKED_SOURCE`).
- O app de campo (24c) precisa de um contrato pronto e de um api que já o atenda.

**Sucesso:**
- A visita registrada pelo ACS no celular vira ficha de visita aceita pelo SIAPS, uma por profissional e dia.
- A boa prática de visita das linhas de cuidado mede a visita real, com a regra da nota metodológica.
- O ACS abre a microárea e vê quem precisa de visita e por quê.
- O contrato de sincronização é testado de ponta a ponta sem o app.

**Antiobjetivos:**
- Motivo da visita, medidas, observação, nome, CPF ou CNS em URL, log ou evento.
- Visita recusada ou de ausente visível para o cidadão.
- Dado de cidadão em visita a imóvel não residencial.
- Ficha com CPF ou CNS inventado, ou com o documento de outra pessoa fora da regra do responsável familiar.
- O app de campo como fonte de verdade (ADR 0041).

## 2. Decisões

### Decisões do usuário

| # | Decisão |
|---|---|
| 1A | **O ACS registra a própria visita no dashboard** até o app de campo (24c), inclusive pelo celular: uma folha do dia com várias visitas. Não há papel de digitador. |
| 2A | **Atividade coletiva com os 7 tipos**, mas sem PSE: o cadastro de escolas com INEP fica para o próximo ciclo. |
| 3A | **wpda:** o cidadão vê só a visita realizada, com a data e "Visita do agente de saúde", sem motivo nem observação. Recusa e ausência nunca aparecem. "Meu acompanhamento" passa a contar a visita. |
| 4A | **O 24b entrega o contrato de sincronização** do app de campo em `rotasaude/contracts` **e o lado do api**: envio idempotente e download incremental por microárea, testados sem o app. O 24c só consome. |
| 5A | **"Quem visitar":** lista só de leitura por microárea, com três fontes: pendências das linhas de cuidado que pedem visita, cadastros perto de vencer os 24 meses e pessoas nunca visitadas. Sem agendamento. |
| 6A | **Quem tem o app de campo e o que baixa.** O ACS baixa só as microáreas pelas quais é responsável. Os outros CBOs de campo da Tabela 7 (TACS, ACE, agente de ação social, agentes indígenas, visitador sanitário) baixam só as microáreas que o enfermeiro da equipe atribui a cada um no dashboard (atribuição nova, com trilha). Médico e enfermeiro não cadastram aparelho. O download e as remoções seguem esse escopo. |
| 7A | **Participante ou morador sem cadastro.** Na atividade coletiva e na visita, quem ainda não é paciente entra por um registro rápido (nome, nascimento, sexo e CPF ou CNS, ou a justificativa de não ter CPF do 24a), pelo dashboard e pelo envio do app. Vale a guarda de duplicado do 24a: a tela avisa quando a pessoa parece já existir e oferece abrir o existente. |

### Decisões do coordenador (desenho)

| # | Decisão |
|---|---|
| 1 | **Folha do dia = ficha LEDI.** A folha (`home_visit_days`) é de um profissional, uma equipe e uma data, e vira **uma** ficha de visita (tipo 8, `FichaVisitaDomiciliarMasterThrift`) com até 99 visitas, como o e-SUS Território faz. A 100ª visita do dia abre uma folha complementar. É o que a decisão 1A descreve e o que a ficha prevê. |
| 2 | **Fechamento da folha.** A folha fica aberta no dia; o ACS a fecha ("Fechar o dia") ou um job diário (`EachCityJob`) fecha as folhas de dias anteriores de madrugada. A ficha é gerada no fechamento, dentro da transação. Visita que chega depois (o app de campo, decisão 4A) para um dia já fechado entra numa folha complementar do mesmo dia, que o job fecha. Depende de o PEC aceitar mais de uma ficha do mesmo profissional no mesmo dia (seção 9). |
| 3 | **Correção.** Enquanto a folha está aberta, a visita pode ser **retirada** (com motivo, fica registrada) e registrada de novo; a linha da visita não muda. Fechada, a folha e suas visitas são imutáveis (trigger em `db/city_triggers.sql`, exceção `rota.reencrypting`). Ficha recusada é regerada da folha, como no 24a. |
| 4 | **Quem registra a visita:** membro ativo da equipe com CBO da Tabela 7 do LEDI (ACS 515105, TACS 322255, ACE 515140, agente de ação social 515310, agentes indígenas 515125 e 515130, visitador sanitário 515120), no `config/ledi/registration_cbos.yml` como `home_visit`. O ACS registra só na microárea em que é responsável, ou fora de área; os outros CBOs da tabela, nas microáreas atribuídas a eles (6A). Enfermeiro e médico **não** registram (a Tabela 7 não os aceita), mas leem as visitas da equipe. |
| 5 | **Quem é visitado.** A visita é a um imóvel, com ou sem pessoa. Ligada a um domicílio do 24a, herda o tipo de imóvel e a microárea; sem domicílio, o profissional escolhe o tipo de imóvel e a microárea (ou fora de área). Em imóvel não residencial (tipos 2, 3, 4, 5, 6, 12 e 13), nenhum dado de pessoa. Em imóvel residencial, a visita pode ser a um morador (paciente) ou à família; na família, o documento da ficha é o do responsável familiar, como a ficha pede para visita periódica e controle vetorial. |
| 6 | **Documento na ficha:** o do paciente que a ficha leva, CPF se houver, senão CNS (nunca os dois); sem nenhum, o marcador "não possui CPF" com a justificativa do paciente (24a). Nascimento e sexo vêm do paciente. O `uuidOrigemCadastroDomiciliar` é a ficha originadora do domicílio (`Ledi::HouseholdRegistrationFicha#originating_uuid`); sem ficha utilizável, a visita sai sem ele, nunca com uuid inventado. |
| 7 | **Campos da visita:** desfecho (realizada, recusada, ausente), motivos (lista oficial, com as regras: nenhum em recusada e ausente; só os de controle ambiental e "outros" em não residencial; gestante e puérpera proibidos para sexo masculino), visita compartilhada, peso, altura, pressão, temperatura, glicemia com tipo, e observação interna (até 500 caracteres, nunca vai ao LEDI nem ao cidadão). O turno sai da hora da visita no fuso da cidade (antes de 12h manhã, antes de 18h tarde, depois noite). Medidas são proibidas em recusada, ausente e não residencial. |
| 8 | **Data da visita:** no máximo hoje e não anterior ao prazo da competência no SIAPS (`config/ledi/siaps_deadlines.yml`), para o app offline não mandar o que o SIAPS já não aceita. |
| 9 | **Atividade coletiva:** uma atividade = uma ficha (tipo 6). Equipe, unidade, data, turno, tipo (1 a 7), local (a unidade pelo CNES ou outra localidade em texto; INEP fora, decisão 2A), número de participantes, temas para reunião, público-alvo, temas para saúde, práticas em saúde, procedimento SIGTAP (só com a prática 30), profissionais participantes (do cadastro do 10, com CNS e CBO) e lista de participantes (paciente, avaliação alterada, peso, altura, e os marcadores do tabagismo nas práticas 25–28). As regras cruzadas da ficha valem no api e são espelhadas no dashboard. Os dois campos de PSE vão sempre `false`; o tema 18 (Semana saúde na escola), que exige PSE, não é oferecido. |
| 10 | **Participantes:** obrigatórios nos tipos 5 e 6, como a ficha pede; nos outros tipos, opcionais e incentivados na tela, porque só participante identificado conta no Acompanhamento. Na ficha, o CPF do paciente; o CNS só quando ele não tem CPF (marcador e justificativa). Sem CPF e sem CNS num tipo 5 ou 6, a ficha fica "não gerada" (`participant_without_identifier`). |
| 11 | **Quem registra a atividade:** membro ativo da equipe com CBO da Tabela 5 (`collective_activity` no `registration_cbos.yml`); é o profissional do cabeçalho. Uma gravação é a atividade inteira e gera a ficha na hora; sem rascunho, como no 24a. Erro depois de gravada: "registrada por engano", que tira a atividade das telas e das fontes, mas não da ficha já enfileirada (seção 9). |
| 12 | **Interruptores:** `home_visits` ("Visita domiciliar e territorial"), que requer `territory_registry`; e `collective_activities` ("Atividade coletiva"), que requer `clinical_record`. A atividade coletiva não depende do cadastro territorial e a cidade pode querer uma sem a outra. Desligado: nenhuma tela, rota ou ficha nova; o que foi gravado fica. A sincronização do app exige `home_visits`. |
| 13 | **Linhas de cuidado.** Com `home_visits` utilizável, `acs_visit` deixa de ser não rastreada (precedente do `vaccines!` do 29): `CareLines::Catalog.untracked?` ganha `home_visits:` e `CareLines::Sources` ganha `home_visits!`. Conta como evento a visita **realizada**, a imóvel residencial, **com pelo menos um motivo**, por CBO 515105 ou 322255, para o paciente cujo documento vai na ficha. Sem o interruptor, segue "não acompanhado". |
| 14 | **Intervalo de 30 dias.** A boa prática ganha o atributo opcional `min_gap_days`: duas visitas só contam se tiverem esse intervalo. O catálogo-base sai em versão nova com `min_gap_days: 30` na C5 (nota metodológica lida). C2, C3, C4 e C6 ficam sem o atributo até as notas serem lidas (seção 9). |
| 15 | **Medidas da visita** (pressão, peso e altura) alimentam também as fontes `blood_pressure` e `weight_height`, como a C5 aceita; só de visita realizada. |
| 16 | **Linha do tempo:** kind novo `home_visit` em `Timeline::KINDS`. O profissional vê todos os desfechos, com motivos e medidas; a observação só na tela da visita. O cidadão (`CITIZEN_FILTERED_KINDS`) vê só a realizada, com data e "Visita do agente de saúde", sem nome do profissional, motivo, medida ou observação. A participação em atividade coletiva entra como `collective_activity` só na lista do profissional (`PROFESSIONAL_ONLY_KINDS`). |
| 17 | **"Quem visitar":** por microárea, para quem registra visita ali e para o enfermeiro e o médico da equipe. Cada pessoa vem com os motivos da lista: `care_line` (boa prática com fonte `acs_visit` vencida ou a vencer, com o nome da linha e da prática, sem CID), `registration_due` (cadastro a vencer em 60 dias ou vencido, pela regra do 24a) e `never_visited` (cadastro ativo sem nenhuma visita realizada). Leitura nominal com trilha; contagem sem trilha. |
| 18 | **Aparelho de campo.** O app precisa de credencial própria: o profissional cadastra o aparelho no dashboard, com segundo fator, e recebe o token uma vez (QR). O token é guardado só como resumo, vale para aquele usuário e aquela cidade, e o profissional ou o administrador municipal revoga. Perder o vínculo com a equipe revoga. |
| 19 | **Sincronização (contrato v1):** envio em lote idempotente pelo `client_uuid` de cada item (visita, retirada de visita, atividade coletiva, e cadastro individual e domiciliar com os corpos do 24a e a versão esperada); cada item volta `accepted`, `duplicate` ou `rejected` com o código de erro, e o cadastro com versão vencida volta `stale_version` com a versão atual, para o app resolver o conflito (24c). Download incremental pelas microáreas do aparelho (6A), por cursor opaco, em páginas: domicílios, famílias, moradores com o cadastro atual, "quem visitar" e remoções. Toda página baixada grava uma trilha com os ids. |
| 20 | **Sensível:** motivos, medidas, observação, temas, público, práticas, procedimento, avaliação e medidas dos participantes ficam cifrados (`CityEncryption::CITY_KEYED_TARGETS`). A visita guarda à parte, sem cifra, só se tem motivo (`has_reason`), para a regra da linha de cuidado. |
| 21 | **Primeira task do api é prova técnica:** vendorizar `dicionario-fvd.html` e `dicionario-fac.html` com SHA-256 no `vendor/ledi/8.7.0/SOURCE`, ler as regras oficiais das duas fichas, conferir contra o IDL e o `gen-rb` e fixar os códigos (inclusive o "Outros" do público-alvo), publicando a tabela no contrato antes das tasks do dashboard. |
| 22 | **Interruptor desligado com folha aberta.** O job de madrugada fecha as folhas abertas de dias anteriores mesmo com `home_visits` desligado, e a ficha é gerada no fechamento: o que foi visitado chega ao SIAPS. Desligado, nenhuma visita nova entra. Folha cujo dia já passou do prazo da competência no SIAPS fecha com a ficha "não gerada" (`siaps_deadline_passed`), nunca descartada. |
| 23 | **Contagem com a folha aberta.** A visita conta nas linhas de cuidado, em "Meu acompanhamento" e em "quem visitar" assim que é gravada, sem esperar o fechamento. A retirada (decisão 3) tira a visita das fontes, e o status recalcula. |

## 3. Dados

**Cidade:**
- `home_visit_days`: profissional, equipe, data (dia da cidade), sequência (1 e as complementares), estado (`open`, `closed`), fechada em, fechada por (`user` ou `job`). Única por profissional, equipe, data e sequência.
- `home_visits`: folha, `client_uuid` (único), instante, turno, domicílio (opcional), família (opcional), paciente (opcional), quem a ficha leva (`resident` ou `family_responsible`), tipo de imóvel, microárea ou fora de área, desfecho, `has_reason`, motivos, visita compartilhada, medidas e observação (cifrados), retirada em, motivo da retirada, origem (`dashboard` ou `field_app`).
- `collective_activities`: `client_uuid` (único), equipe, unidade, data, turno, tipo, local, número de participantes, número de avaliações alteradas, temas, público, práticas e procedimento (cifrados), registrada por, por engano em.
- `collective_activity_professionals`: atividade, profissional, CBO.
- `collective_activity_participants`: atividade, paciente, avaliação alterada, medidas e marcadores do tabagismo (cifrados).
- `field_devices`: usuário, nome do aparelho, resumo do token, último uso, revogado em, revogado por.
- `ledi_outbox` ganha as fontes `HomeVisitDay` e `CollectiveActivity` em `Ledi::FichaSources`.
- `Ledi::FichaTypes` ganha `visita_domiciliar` (código 8, `Br::Gov::Saude::Esusab::Ras::Visitadomiciliar::FichaVisitaDomiciliarMasterThrift`, `uuidFicha`) e `atividade_coletiva` (código 6, `Br::Gov::Saude::Esusab::Ras::Atividadecoletiva::FichaAtividadeColetivaThrift`, `uuidFicha`).
- `config/ledi/registration_cbos.yml` ganha `home_visit` (Tabela 7) e `collective_activity` (Tabela 5).
- **Plataforma:** interruptores `home_visits` e `collective_activities`; catálogo-base das linhas de cuidado em versão nova (`min_gap_days`).

## 4. Telas

**dashboard:**
- **Folha do dia** (no celular primeiro): as visitas do dia, nova visita a partir da microárea (domicílio → família ou morador) ou de imóvel sem cadastro, desfecho, motivos, medidas, observação, retirar, fechar o dia. Mostra o que falta para a ficha sair ("sem CNS do profissional", por exemplo).
- **Visitas da equipe:** por microárea, profissional e período, com a situação da ficha; leitura para o enfermeiro e o médico.
- **Atividade coletiva:** nova (com as regras cruzadas guiando o formulário), lista por equipe e período, detalhe, "registrada por engano".
- **Quem visitar:** por microárea, com o motivo de cada pessoa e atalho para registrar a visita.
- **Aparelhos de campo:** cadastrar com segundo fator e QR, listar, revogar.
- Linha do tempo do paciente: o kind `home_visit` e a participação em atividade coletiva.

**wpda:**
- Linha do tempo: "Visita do agente de saúde" com a data, só a realizada.
- "Meu acompanhamento": a boa prática de visita passa a contar.

## 5. Segurança e LGPD

- **Fora do sistema:** nada de nome, CPF, CNS, motivo, medida, observação, tema ou prática em URL, log, evento ou SMS. Eventos só com ids: `home_visit.recorded`, `.withdrawn`, `home_visit_day.closed`, `collective_activity.recorded`, `.marked_mistaken`, `field_device.registered`, `.revoked`. Os novos `Platform.audit` entram em R18.
- **Trilha:** leitura nominal (visitas, atividade, "quem visitar", download do app) grava `clinical_record.viewed`; contagens e polling sem trilha.
- **Escopo:** ACS só na própria microárea; equipe só na sua; registro de outro escopo por id → 404, lista de escopo alheio → 403.
- **Aparelho:** token só no corpo e no cabeçalho de autorização, nunca na URL; limite de tentativas; o download devolve só a microárea do profissional.
- **Exclusão LGPD (ADR 0026):** folha e atividade enfileiradas no `ledi_outbox` são retidas, como o cadastro do 24a. A visita retirada e a atividade por engano também (são trilha).
- **Imutabilidade:** folha fechada, visitas dela, atividade gravada e participantes não mudam (só `withdrawn_at` em folha aberta e `mistaken_at`).
- **Travas em ordem estável:** paciente → domicílio → folha → visita.

## 6. Testes

**api:**
- prova técnica: dicionários vendorizados com SHA-256; códigos dos domínios conferidos com o `gen-rb`;
- ficha de visita conferida campo a campo com um exemplo de referência; uma folha, várias visitas; limite de 99 e folha complementar; fechamento pelo ACS e pelo job; visita tardia em dia fechado;
- regras da visita: desfecho × motivos × medidas; não residencial sem pessoa; sexo × gestante e puérpera; CPF xor CNS; responsável familiar; sem documento com justificativa; `uuidOrigemCadastroDomiciliar` presente e ausente;
- quem registra: Tabela 7, ACS só na sua microárea, enfermeiro e médico só leem;
- atividade coletiva: cada regra cruzada da ficha, participantes nos tipos 5 e 6, CPF antes do CNS, prática 30 com SIGTAP, PSE sempre falso, Tabela 5;
- "não gerada" com motivos de lista fechada; regerar ficha recusada;
- linhas de cuidado: `acs_visit` só com o interruptor; só realizada, residencial, com motivo e CBO de ACS/TACS; `min_gap_days`; medidas nas fontes de pressão e peso/altura; prévia;
- linha do tempo: profissional vê tudo; cidadão só a realizada, sem motivo, nome nem medida;
- "quem visitar": as três fontes, escopo e trilha;
- sincronização: aparelho e revogação; envio repetido não duplica; item recusado não derruba o lote; `stale_version`; download incremental, paginado, só da microárea, com remoções e trilha; tudo pelo contrato, sem o app;
- nada sensível em URL, log e evento; interruptores; R18; imutabilidade; exclusão LGPD.

**dashboard:** testes de componente da folha do dia (largura de celular), da atividade coletiva com as regras espelhadas, de "quem visitar" e dos aparelhos.

**wpda:** teste de componente da entrada de visita na linha do tempo e de "Meu acompanhamento".

**contracts:** os exemplos do contrato validam contra o schema; o api testa as respostas contra o mesmo schema (cópia, como no protocolo).

**Suíte completa do api no fim** (migração de cidade, `CareLines`, `Timeline`, `Ledi`).

## 7. Entrega

- **Uma entrega só.**
- **Ordem:** contracts → api → dashboard → wpda → maintenance → catálogo.
- **Runbook:** `operacao/rollout-visita-e-coletiva.md`, com:
  - ligar `home_visits` só depois do `territory_registry` com microáreas e ACS responsáveis;
  - publicar a versão nova do catálogo-base no maintenance;
  - CNS de todos os ACS e TACS (sem CNS a ficha não sai);
  - cadastro dos aparelhos de campo só quando o 24c existir.

## 8. F-IDs

| ID | Funcionalidade | Superfície |
|---|---|---|
| F-24.9 | Interruptores e quem registra (Tabelas 7 e 5 do LEDI) | api, dashboard, maintenance |
| F-24.10 | Folha do dia e visita domiciliar pelo dashboard | api, dashboard |
| F-24.11 | Ficha LEDI de visita domiciliar e territorial (tipo 8) | api |
| F-24.12 | Atividade coletiva, sem PSE | api, dashboard |
| F-24.13 | Ficha LEDI de atividade coletiva (tipo 6) | api |
| F-24.14 | Visita nas linhas de cuidado e em "Meu acompanhamento" | api, dashboard, wpda |
| F-24.15 | Visita na linha do tempo do profissional e do cidadão | api, dashboard, wpda |
| F-24.16 | "Quem visitar" por microárea | api, dashboard |
| F-24.17 | Contrato de sincronização do app de campo e lado do api (aparelho, envio idempotente, download por microárea) | contracts, api, dashboard |

## 9. Riscos e pendências

- **Várias fichas do mesmo profissional no mesmo dia:** a folha complementar depende de o PEC aceitar (não confirmado). Se recusar, a visita tardia fica "não gerada" e o desenho do fechamento muda. É gate de go-live.
- **Correção depois do envio:** a ficha de visita e a de atividade não têm originadora; não está confirmado como o PEC trata reenvio ou anulação. Até lá, "retirada" só vale em folha aberta e "por engano" não desfaz a ficha.
- **Contagem no SIAPS:** não confirmado se recusada e ausente contam no Acompanhamento, se a visita à família conta só para o responsável e se participantes dos tipos 4 e 7 contam.
- **Notas C2, C3, C4 e C6:** não lidas; o `min_gap_days` delas fica para quando forem.
- **Credencial do aparelho:** o 24c pode pedir outra forma (por exemplo, cadastro pelo administrador); o contrato é versionado para isso.
- **Gates de go-live:** prova no PEC real das fichas 8 e 6; SIAPS contando a visita e a atividade no Acompanhamento; a regra da C5 conferida contra o relatório do SIAPS.
- **Próximo ciclo:** PSE (escolas com INEP), IVCF-20 na visita, agendamento de visita.
