# Módulo 33 — Ouvidoria do SUS municipal — design

**Data:** 2026-11-11
**Status:** aprovado (2026-11-11)
**Afeta:**
- `apps/api`:
  - **cidade:** setores da secretaria, assuntos, manifestações com protocolo, encaminhamentos com extrato, respostas, prazos, identidade preservada com trilha, anexos, avisos ao cidadão, pesquisa de satisfação, painel e relatório anual;
  - **plataforma:** interruptor `ombudsman`.
- `apps/dashboard`: Ouvidoria (caixa, manifestação, encaminhamento, resposta, configuração), caixa do ponto de resposta, cadastro de setores, painel e relatório anual.
- `apps/wpda`: nova manifestação identificada ou com identidade preservada, denúncia anônima sem login, "Minhas manifestações", complemento, aviso de resposta e pesquisa de satisfação.
- `apps/maintenance`: interruptor, pelo mecanismo genérico.

**ADR:** `docs/adr/0051.md` · **Módulo:** `docs/modulos/33--ouvidoria.md` · **Pesquisa:** `docs/pesquisa/2026-11-11-ouvidoria-33.md`
- **ADRs relacionados:** 0010 (relatório público), 0017 (identidade do cidadão), 0020 (banco por cidade), 0026 (exclusão LGPD).
- **Módulos relacionados:** 06 (identidade e acesso), 10 (profissionais), 14 (analytics).

**Fora desta entrega:**
- integração com o Fala.BR (API) e com o OuvidorSUS, incluindo a ficha para digitação no OuvidorSUS (cards no board #2, próximo ciclo);
- totem de satisfação por unidade e perguntas de satisfação cadastradas pela cidade (card no board #2, próximo ciclo);
- pedidos de acesso à informação (LAI, e-SIC), que seguem outro rito;
- conselhos de usuários e Carta de Serviços (Lei 13.460, arts. 7º e 18–22);
- tipo "simplificação" (só existe para entes federais no Fala.BR);
- chat com a ouvidoria;
- gravação de ligação telefônica.

## 1. Intenção

O cidadão fala com a secretaria de saúde por um canal oficial, recebe um protocolo e sabe que vai ter resposta dentro do prazo da lei. A ouvidoria recebe tudo num lugar só, classifica, pede a quem sabe responder (um setor da secretaria ou uma unidade) e responde ao cidadão. Quem denuncia pode se identificar sem se expor, ou não se identificar.

**Por quê:**
- A ouvidoria está em 7 dos 15 editais.
- A Lei 13.460/2017 vale para o município: prazo de 30 dias, prorrogável uma vez por mais 30, com justificativa, relatório de gestão anual e pesquisa de satisfação.
- A Lei 13.608/2018 obriga o município a manter ouvidoria para receber denúncias e a preservar a identidade do denunciante.

**Sucesso:**
- Toda manifestação tem protocolo, prazo visível e uma resposta conclusiva ou um arquivamento com motivo.
- A equipe vê, por cor, o que está perto de vencer e o que já venceu.
- A identidade preservada só aparece para o ouvidor, e cada vez que ele abre fica registrado quem abriu, quando e por quê.
- O relatório anual sai do sistema, sem planilha paralela.

**Antiobjetivos:**
- Identidade de quem pediu sigilo visível para técnico, setor, unidade ou profissional reclamado.
- Texto da manifestação, nome ou CPF em SMS, URL, log ou evento.
- Manifestação recusada ou sem resposta por falta de identificação (Lei 13.460, art. 10, §1º).
- Ponto de resposta (setor ou unidade) falando direto com o cidadão.

## 2. Decisões

| # | Decisão |
|---|---|
| 1 | **Ouvidoria própria:** o Rota Saúde é o sistema da ouvidoria da secretaria, seguindo a Lei 13.460 e usando como modelo operacional o Decreto 9.492/2018, o 10.153/2019 e a Portaria Normativa CGU 116/2024. **Sem integração nesta entrega** (escolha do usuário, opção 3A). Fala.BR (API) e ficha para o OuvidorSUS ficam para o próximo ciclo, com cards no board #2. |
| 2 | **Tipos:** `complaint` (reclamação), `report` (denúncia), `praise` (elogio), `suggestion` (sugestão) e `request` (solicitação de providência). Uma manifestação nunca é recusada; a que não serve é arquivada com motivo (decisão 10). |
| 3 | **Modos de identificação:** `identified`; `preserved` (identidade preservada: só o papel `ombudsman` vê quem é, decisão 12); `anonymous` (só para denúncia). |
| 4 | **Canais:** `web` (wpda), `in_person` e `phone` (registrados pela equipe, reduzidos a termo), e `ouvidorsus` (caso que chega da rede, como o Disque Saúde 136, digitado pela equipe com o número de origem). |
| 5 | **wpda identificado:** o cidadão entra com CPF e código SMS (ADR 0017) e registra a manifestação nos modos `identified` ou `preserved`. O par declarado basta; não exige validação no balcão. Pode anexar até 3 arquivos (PDF, JPEG ou PNG, até 10 MB cada), pelo padrão de `ExamFile`, com finalidade nova. Acompanha tudo em "Minhas manifestações". |
| 6 | **Denúncia anônima sem login no wpda** (escolha do usuário, opção 1A): formulário público, só tipo `report`, só texto (20 a 4.000 caracteres), unidade referida opcional, sem anexo e sem contato. **Sem protocolo e sem acompanhamento**, como a "comunicação" do Fala.BR; a tela diz isso antes do envio. Limite por IP de 3 por hora e 10 por dia, mais um teto diário por cidade configurável (padrão 50), acima do qual o formulário responde "tente mais tarde" e a equipe é avisada. É o primeiro formulário público de escrita do sistema. |
| 7 | **Registro pela equipe:** o atendente reduz a termo a manifestação presencial ou por telefone. Identificado: busca o cidadão pelo CPF ou registra nome e contato só na manifestação, cifrados. O modo `preserved` e o anônimo também valem aqui. Entrega o **comprovante** impresso com protocolo, tipo, data e prazo. |
| 8 | **Protocolo:** ano e sequência da cidade, `2026-000123`, por sequence do Postgres criada em `db/city_triggers.sql`, no padrão de `LabRequisition.next_number`. A denúncia anônima recebe um número interno, que não é mostrado ao cidadão. |
| 9 | **Estados:** `received` → `in_analysis` (classificada) → `forwarded` (aguardando ponto de resposta) ⇄ `in_analysis` → `answered` (resposta conclusiva) ou `archived`. Paralelos: `awaiting_complement` (prazo suspenso). `extended` é um marcador, não um estado. **Reabrir** (`answered`/`archived` → `in_analysis`) é ato da ouvidoria, a pedido do cidadão em até 30 dias da conclusão ou de ofício, e abre um novo prazo de 30 dias. |
| 10 | **Respostas e encerramento:** resposta **intermediária** (não mexe no prazo), pedido de **complemento** (uma vez; suspende o prazo; sem resposta em 20 dias, arquiva), **prorrogação** (uma vez, com justificativa, +30 dias) e resposta **conclusiva**, que marca a resolutividade (`resolved` ou `not_resolved`, alterável depois com trilha). **Arquivamento** com motivo de lista: duplicidade (com o protocolo original), falta de clareza sem complemento, falta de urbanidade, manifestação imprópria, perda de objeto, encaminhada a outro órgão. A denúncia anônima só tem resposta conclusiva interna ou arquivamento. |
| 11 | **Setores e encaminhamento** (escolha do usuário, opção 2B): a cidade cadastra **setores da secretaria** (`departments`: nome, ativo), que valem além das unidades de saúde. Destino de um encaminhamento é um setor ou uma unidade. Cada destino tem **respondentes** (pessoas com o papel `ombudsman_responder`, ligadas ao destino). O encaminhamento leva um **extrato** escrito pela ouvidoria, nunca a identidade nem o texto original. O destino responde à ouvidoria em até 20 dias (configurável), prorrogável uma vez. Pode haver vários encaminhamentos em paralelo. |
| 12 | **Identidade preservada:** nos modos `preserved` e `anonymous` (quando houver dado), nome, CPF, contato e anexos com dado pessoal ficam visíveis só para o papel `ombudsman`, e cada abertura exige justificativa e grava `ombudsman_identity_accesses` (imutável). O técnico vê "identidade preservada". Revelar fora da ouvidoria exige consentimento do cidadão, registrado pelo ouvidor; sem ele, nunca sai. |
| 13 | **Papéis:** `ombudsman` (ouvidor: tudo, inclusive identidade preservada, prorrogação, arquivamento, reabertura, relatório anual; privilegiado, com segundo fator); `ombudsman_staff` (técnico e atendente: registra, classifica, encaminha, redige respostas e envia; vê a identidade só no modo `identified`); `ombudsman_responder` (ponto de resposta: vê só os encaminhamentos do seu destino, pelo extrato, e responde à ouvidoria). |
| 14 | **Classificação:** assunto e subassunto (`ombudsman_subjects`, por cidade, com uma lista inicial semeada), unidade referida e profissional referido (opcional, do cadastro do 10), e marca de urgência. A manifestação urgente tem prazo de 15 dias (Portaria SE/MS 729/2020). Reclassificar fica registrado. |
| 15 | **Prazos:** em dias corridos, calculados na leitura (fonte única em SQL com espelho em Ruby, como em `CareRegulation::Deadlines`), configuráveis por cidade: resposta 30, prorrogação 30, urgente 15, complemento 20, ponto de resposta 20. Cores: verde, amarelo a 5 dias do fim, vermelho vencido. Um job diário (`EachCityJob`) avisa a ouvidoria e os respondentes por e-mail, só com contagens. |
| 16 | **Cidadão no wpda:** "Minhas manifestações" lista protocolo, tipo, estado, prazo e as respostas enviadas a ele. Cada resposta intermediária, pedido de complemento e resposta conclusiva vira um **aviso** na caixa de avisos (nova fonte em `NoticesController`), sem conteúdo no aviso, e SMS opcional pelo padrão de `Transport::Notify` (desligado até o gate de go-live do SMS). O cidadão responde ao complemento e pede reabertura pelo wpda. |
| 17 | **Pesquisa de satisfação** (escolha do usuário, opção 4A): depois da resposta conclusiva, uma vez por manifestação identificada pelo wpda, em até 30 dias, com 3 perguntas fixas: "sua demanda foi atendida?" (sim, parcialmente, não), "a resposta foi fácil de entender?" (sim, não) e satisfação de 1 a 5. Totem e perguntas próprias da cidade ficam para o próximo ciclo. |
| 18 | **Painel e relatório anual:** volume por tipo, assunto, unidade, canal e estado; respostas no prazo; tempo médio; resolutividade; satisfação. Contagens de 1 a 4 escondidas (`Analytics::Suppression`). O ouvidor monta o **relatório de gestão anual** (Lei 13.460, arts. 14–15) com os números do ano e os textos dele (problemas recorrentes, providências), e exporta para publicar. O painel por profissional referido é só do ouvidor. |
| 19 | **Interruptor:** `ombudsman` ("Ouvidoria do SUS municipal"), sem pré-requisito. A denúncia anônima sem login tem o próprio interruptor, `ombudsman_anonymous_web`, que requer `ombudsman`, para a cidade poder desligar o formulário público sem desligar a ouvidoria. |

## 3. Dados

**Cidade:**
- `departments`: setores da secretaria (nome, ativo). Genérico, para outros módulos usarem depois.
- `ombudsman_responders`: pessoa (`user_id`) e destino (`department_id` ou `health_unit_id`, CHECK de um só).
- `ombudsman_subjects`: assunto e subassunto, ativo.
- `ombudsman_settings`: prazos, teto diário anônimo.
- `ombudsman_manifestations`: protocolo, tipo, modo, canal, `citizen_id` (nulo no anônimo e no registro sem cidadão), nome e contato informados (cifrados), texto (cifrado), assunto, unidade e profissional referidos, urgente, estado, datas de recebimento, prazo base, suspensões, prorrogação, resolutividade.
- `ombudsman_events`: linha do tempo da manifestação, só acréscimos (registro, classificação, encaminhamento, resposta do destino, complemento pedido e recebido, prorrogação, resposta intermediária e conclusiva, arquivamento, reabertura, mudança de resolutividade), com texto cifrado quando houver.
- `ombudsman_forwardings`: destino, extrato (cifrado), prazo, prorrogação, resposta (cifrada), estado.
- `ombudsman_identity_accesses`: quem, quando, justificativa (cifrada). Imutável.
- `ombudsman_satisfaction_responses`: uma por manifestação.
- `ombudsman_notices`: avisos ao cidadão, só `read_at` muda (padrão de `transport_trip_notices`).
- `exam_files.purpose` ganha `ombudsman_attachment`, com o anexo ligado à manifestação e marcado com o modo de identificação.
- `memberships.role` ganha `ombudsman`, `ombudsman_staff` e `ombudsman_responder` (`ombudsman` em `PRIVILEGED_ROLES`).
- Sequence `ombudsman_protocol_seq`, com o ano no protocolo.
- **Plataforma:** interruptores `ombudsman` e `ombudsman_anonymous_web`.

## 4. Telas

**dashboard:**
- **Ouvidoria · Caixa:** manifestações por estado e cor de prazo, com filtros por tipo, assunto, unidade, canal e responsável.
- **Ouvidoria · Manifestação:** dados, linha do tempo, encaminhamentos, respostas, ações (classificar, encaminhar, pedir complemento, prorrogar, responder, arquivar, reabrir), anexos, identidade (só ouvidor, com justificativa).
- **Ouvidoria · Nova manifestação:** registro presencial ou por telefone, com comprovante para impressão.
- **Ouvidoria · Configuração:** setores, respondentes, assuntos e prazos.
- **Respostas à ouvidoria:** caixa do ponto de resposta, com o extrato e o campo de resposta.
- **Painel:** Ouvidoria, mais o relatório anual.

**wpda:**
- Entrada "Ouvidoria" na tela inicial, com dois caminhos: "Fazer uma manifestação" (entra com CPF e código) e "Denúncia anônima" (sem login).
- "Minhas manifestações", com detalhe, complemento, pedido de reabertura e pesquisa de satisfação.

## 5. Segurança e LGPD

- **Base legal:** obrigação legal e execução de política pública (LGPD, arts. 7º, II e III, e 23), não consentimento. A manifestação pode ter dado de saúde (sensível).
- **Cifra:** texto, extrato, respostas, nome, contato e justificativas entram em `CityEncryption::CITY_KEYED_TARGETS`.
- **Fora do sistema:** nada de texto, nome, CPF, assunto ou unidade referida em URL, log, evento ou SMS. Os eventos levam só ids. O aviso ao cidadão só diz que há novidade no protocolo.
- **Imutabilidade:** `ombudsman_events`, `ombudsman_identity_accesses` e resposta enviada não mudam (trigger em `db/city_triggers.sql`, `rota_reencryption_only`). Os novos `Platform.audit` entram em R18.
- **Formulário público:** só JSON, limite por IP e teto por cidade (decisão 6), tamanho máximo do corpo, nada do que foi digitado nos logs.
- **Exclusão LGPD (ADR 0026):** a manifestação é registro de obrigação legal e fica **retida** (guarda da Portaria CGU 116: 5+5 anos, denúncia 5+15). O pedido de exclusão do cidadão desliga a manifestação da conta no wpda e marca o motivo de retenção em `Citizens::RequestErasure`. A purga ao fim da guarda é pendência do board #2.
- **Painéis:** agregados, com "menos de 5"; o recorte por profissional é só do ouvidor.

## 6. Testes

**api:**
- protocolo único e sequencial por ano;
- os três modos, com a identidade invisível para técnico e respondente no `preserved`;
- trilha de cada abertura da identidade, com justificativa obrigatória;
- denúncia anônima: sem login, só `report`, sem anexo, limite por IP, teto por cidade, nada no log;
- registro pela equipe com e sem cidadão, e comprovante;
- máquina de estados, incluindo reabertura e arquivamento com motivo;
- prazos: 30, prorrogação única, urgente 15, complemento suspende e arquiva em 20, ponto de resposta 20, cores;
- encaminhamento só com extrato; respondente vê só o próprio destino;
- avisos ao cidadão sem conteúdo; SMS só pelo gate;
- pesquisa de satisfação: uma por manifestação, só depois da conclusiva, só identificada;
- painel com supressão; relatório anual;
- anexos: tipo, tamanho, dono, finalidade;
- exclusão LGPD retém;
- eventos só com ids; R18; interruptores.

**dashboard:** testes de componente; fluxo da caixa até a resposta.

**wpda:** testes de componente das duas entradas e de "Minhas manifestações"; Playwright do formulário anônimo.

**Suíte completa do api no fim.**

## 7. Entrega

- **Uma entrega só.**
- **Ordem:** api → dashboard → wpda → maintenance.
- **Runbook:** `operacao/rollout-ouvidoria.md`, com:
  - decreto ou portaria municipal de ouvidoria (prazos locais);
  - setores, respondentes e ouvidor nomeado;
  - lista de assuntos;
  - decisão da cidade sobre a denúncia anônima sem login;
  - texto do relatório anual.

## 8. F-IDs (propostos)

| ID | Funcionalidade | Superfície |
|---|---|---|
| F-33.1 | Interruptores, papéis, setores, respondentes, assuntos e prazos | api, dashboard, maintenance |
| F-33.2 | Manifestação pelo wpda (identificada ou com identidade preservada) com protocolo e anexos | api, wpda |
| F-33.3 | Denúncia anônima sem login, com limites | api, wpda |
| F-33.4 | Registro pela equipe (presencial, telefone, OuvidorSUS) e comprovante | api, dashboard |
| F-33.5 | Classificação, encaminhamento com extrato e resposta do ponto de resposta | api, dashboard |
| F-33.6 | Prazos, prorrogação, complemento, respostas, arquivamento e reabertura | api, dashboard |
| F-33.7 | Identidade preservada com trilha | api, dashboard |
| F-33.8 | "Minhas manifestações", avisos e pesquisa de satisfação | api, wpda |
| F-33.9 | Painel e relatório de gestão anual | api, dashboard |

## 9. Riscos e pendências

- **Formulário público:** é a primeira superfície de escrita sem login. Sem captcha, o abuso é contido só pelo limite e pelo teto. Se não bastar, captcha vira pendência.
- **Prazos locais:** o município regulamenta a própria ouvidoria; os padrões vêm do modelo federal e da Portaria SE/MS 729/2020. Conferir o decreto local é gate de go-live.
- **Taxonomia:** a lista inicial de assuntos deveria seguir o Manual de Tipificação do OuvidorSUS 3, que não foi obtido.
- **SMS:** desligado até o gate de go-live; até lá o aviso fica só na caixa do wpda.
- **Purga ao fim da guarda:** 10 e 20 anos; fica no board #2.
- **Próximo ciclo (board #2):** integração com o Fala.BR, ficha para o OuvidorSUS, totem e perguntas próprias de satisfação.
