# Módulo 29 — Sala de vacina — design

**Data:** 2026-11-02
**Status:** aprovado (2026-11-02)
**Afeta:**
- `apps/api`: cidade — sala de vacina e vacinadores, registro da dose aplicada (e transcrição de caderneta), cancelamento e substituição, aprazamento pelo calendário, envio RIA-R 2.0 à RNDS com guarda do retorno, estoque de imunobiológicos (reaproveita o razão do 27a) com abertura de frasco e perdas, cadeia de frio, caderneta, fonte `vaccination` da linha do tempo (fecha api#71), marcos de vacina das linhas de cuidado (23) passam a ser medidos, critério "vacina em atraso" nas campanhas (12), ESAVI pré-preenchido na vigilância (25), painel; plataforma — credencial `rnds`, interruptores `vaccination`, `rnds_immunization`, `rnds_mock`.
- `apps/dashboard`: sala de vacina (busca, caderneta, registro da dose, transcrição), estoque e frascos, cadeia de frio, painel de vacinação, credencial RNDS na tela de integrações, critério novo na campanha.
- `apps/wpda`: caderneta do cidadão (doses, próximas doses, situação do envio à RNDS).
- `apps/maintenance`: interruptores (mecanismo genérico).

**ADR:** `docs/adr/0047.md` · **Módulo:** `docs/modulos/29--sala-de-vacina.md` · **Pesquisa:** `docs/pesquisa/2026-11-02-sala-de-vacina-29.md` · ADR 0024 (campanhas), 0028 (modo de prontuário, credenciais, RNDS prevista), 0035 (linha do tempo), 0040 (linhas de cuidado), 0043 (vigilância), 0045 (farmácia e estoque) · card api#71

**Fora desta entrega:** consulta à RNDS do histórico nacional do cidadão (doses de fora entram como transcrição); vacinação offline extramuros (fica para o app de campo, ADR 0041); agenda da sala (demanda espontânea; NT 236/2026 proíbe exigir agendamento); integração com SIES/SIPNI (não há API: o sistema gera o resumo para lançamento manual); envio do ESAVI ao e-SUS Notifica (manual); registro de temperatura por sensor.

## 1. Intenção

A técnica de enfermagem da sala de vacina abre o cidadão, vê na caderneta o que ele deve tomar hoje e o que está atrasado, registra a dose escolhendo o lote que está na geladeira, e a dose chega à RNDS em até 24 horas, sem digitar de novo em outro sistema. A sala controla frascos abertos, perdas e a temperatura das geladeiras no mesmo lugar.

**Por quê:** sala de vacina é lacuna nos 15 editais; desde a Portaria GM/MS 5.663/2024 a dose aplicada vai **só pela RNDS** (prazo de 24 h; o envio de vacina pelo LEDI de terceiros foi desligado), e o indicador C2 do cofinanciamento da APS (vacinas até 2 anos) conta pela RNDS.

**Sucesso:** toda dose registrada no Rota Saúde aparece como aceita pela RNDS no dia seguinte; a coordenação vê quem está atrasado e convoca por campanha; os marcos de vacina das linhas de cuidado deixam de aparecer como "não medidos"; a sala fecha o mês com o resumo de movimentação pronto para o SIES/SIPNI.

**Antiobjetivos:** dose que fica só no Rota Saúde sem o cidadão saber; dose registrada com lote que não está no estoque da sala; caderneta que diverge do Meu SUS Digital sem explicação; exigir agendamento ou número mínimo de pessoas para abrir frasco.

## 2. Decisões

| # | Decisão |
|---|---|
| 1 | **Um módulo só**: dose, RNDS, calendário, caderneta, estoque de imunobiológicos e cadeia de frio (escolha do usuário, opção A). |
| 2 | **Sem consulta ao histórico nacional na RNDS**; dose de fora entra como transcrição da caderneta (escolha do usuário, opção B). |
| 3a | **Caderneta não conferida:** o atraso só é calculado depois que o vacinador confere a caderneta (transcreve o que falta ou confirma que está completa); quem nasceu depois do go-live da cidade já nasce conferido. Antes disso, a caderneta mostra "não conferida" e o calendário mostra só as doses registradas e as próximas pela idade, sem "em atraso"; os marcos `vaccine:*` das linhas de cuidado ficam "não conferido" (não contam como pendentes). A campanha ganha dois alvos: `vaccine_overdue` (só cadernetas conferidas) e `vaccination_record_unchecked` (convocar para trazer a caderneta). A conferência é um registro imutável (quem, quando, sala). (Escolha do usuário, opção A.) |
| 3 | **Campanhas ganham o critério "vacina em atraso"** e a contagem de convocados vacinados (escolha do usuário, opção A). |
| 4 | **Registro da dose:** cidadão, sala (unidade), vacinador, imunobiológico, lote do estoque (fabricante vem do lote), dose, estratégia, grupo de atendimento, via, local de aplicação, condição maternal, data/hora da aplicação (nunca futura). Imutável. Correção = cancelamento com motivo + novo registro; os dois vão à RNDS. |
| 5 | **Transcrição de caderneta:** o mesmo registro com `transcription = true`; lote e fabricante opcionais (texto livre, fora do estoque), data passada, vacinador = quem transcreve; vai à RNDS com o flag de transcrição. |
| 6 | **Quem vacina:** técnico e auxiliar de enfermagem (3222\*), enfermeiro (2235\*) e médico (2251\*, 2252\*, 2253\*), com vínculo à sala de vacina da unidade. Não há papel novo: a sala tem uma lista de vacinadores, como a farmácia tem a equipe do local. |
| 7 | **RNDS RIA-R 2.0** (também para campanha), FHIR R4, autenticação por certificado A1 ICP-Brasil da cidade trocado por token no EHR-AUTH. Credencial nova `rnds` (`IntegrationCredential`), cadastrada pelo administrador municipal na tela de integrações. |
| 8 | **Fila de envio** no padrão da fila do LEDI (idempotente por registro, payload cifrado e anulado no aceite, backoff, alerta), mais o **identificador e o status devolvidos pela RNDS** guardados por registro. Alerta à coordenação quando um registro passa de 24 h sem aceite. Cancelamento só vai depois que o registro original foi aceito. |
| 9 | **Validação antes do envio** (NT 115/2024): CNS do vacinador no CNES importado; CPF/CNS do cidadão conferido no CADSUS quando `cadsus_lookup` está ligado. Falha vira pendência visível na sala, com motivo de lista fechada. |
| 10 | **Calendário** em arquivo versionado (Calendário Nacional 2026), curado do PDF: por imunobiológico e dose, idade mínima e máxima, intervalo mínimo e recomendado, público (criança, adolescente, adulto, idoso, gestante), sexo quando houver. Códigos de imunobiológico, dose, estratégia, via, local e fabricante seguem os CodeSystems da RNDS. Aprazamento e atraso são calculados (cópia recalculável, ver decisões do coordenador). |
| 11 | **Estoque reaproveita o razão do 27a** (lotes, validade, entradas, transferências, perdas, inventário, saldo por soma de movimentos com trigger contra negativo). O lote ganha o tipo de item `immunobiological` e o código BRImunobiologico; a unidade de estoque é a **dose**. |
| 12 | **Frasco multidose:** abrir frasco é uma saída do estoque (`vial_opening`, as doses do frasco). A dose aplicada consome do frasco aberto; o frasco vence pelo prazo **por produto e fabricante** (NT 236/2026); as doses que sobram viram **perda técnica** calculada. Perda física (frasco fechado quebrado, vencido, fora da temperatura) é lançada pela sala. Frasco unidose: a dose aplicada é a própria saída. |
| 13 | **Resumo mensal** por sala no formato da movimentação do SIES/SIPNI (entradas, abertura de frascos, doses aplicadas, perdas técnica e física), para lançamento manual. |
| 14 | **Cadeia de frio:** cada sala cadastra seus equipamentos (geladeira, câmara); registro de temperatura atual, mínima e máxima duas vezes ao dia; fora de +2 a +8 °C gera alerta e pede a conduta (lotes afetados ficam marcados até a decisão: usar ou dar perda). |
| 15 | **Caderneta:** no dashboard, dentro da sala; no wpda, para o cidadão, com doses, próximas doses e a situação do envio à RNDS ("enviada ao Ministério" / "em envio"). |
| 16 | **Ligações:** fonte `vaccination` na linha do tempo (fecha api#71); marcos `vaccine:*` das linhas de cuidado saem de `untracked` e passam a ser medidos pela dose registrada (inclusive transcrição); critério `vaccine_overdue` nas campanhas e contagem de convocados vacinados; botão "notificar ESAVI" na dose abre a notificação da vigilância (código `esavi`) já presa à dose. |
| 17 | **Painel:** doses aplicadas por vacina e sala, cobertura das vacinas do C2 por equipe, envio à RNDS (pendentes, recusados, fora do prazo), perdas. Só contagens. |
| 18 | **Interruptores:** `vaccination` (requer `record_mode:record`; cidade em modo PEC vacina no PEC, que já envia à RNDS); `rnds_immunization` (requer `feature:vaccination` e `credential:rnds`); `rnds_mock` (dev/test, servidor falso, padrão do `sncr_mock`). Com `vaccination` ligado e o envio desligado, a dose é registrada e fica na fila. Exclusão LGPD retém a dose (registro oficial de saúde, como o prontuário — ADR 0026). |

Decisões 1, 2, 3 e 3a são do usuário; as demais são do coordenador, apresentadas no desenho.

**Decisões do coordenador (revisão da spec):**
- **Situação vacinal materializada:** a regra do calendário roda em Ruby; o resultado por cidadão e imunobiológico (aplicadas, devida, em atraso, próxima data) fica numa tabela `vaccination_statuses`, só como cópia recalculável: recalculada na dose, no cancelamento e por um job diário (a idade muda). A fonte da verdade continua sendo as doses + o calendário. O critério `vaccine_overdue` das campanhas é uma relação SQL sobre essa tabela, com parâmetros: imunobiológico (um ou vários) e faixa etária opcional.
- **Local de estoque próprio:** a sala tem local próprio, novo tipo `vaccine_room` (um por unidade; o índice único passa a ser unidade + tipo). A rede de frio municipal é um local novo do tipo `cold_chain_central` (um por cidade), onde entra a vacina recebida da regional/estado e de onde sai a transferência para as salas. Vacina e medicamento nunca dividem local, mínimo nem inventário.
- **Sem dependência da farmácia:** `vaccination` não exige `pharmacy`. Os serviços do razão (lotes, movimentos, transferência, perda, inventário) são usados atrás do interruptor `vaccination`, com telas próprias da rede de frio e da sala (reaproveitando componentes do 27a).
- **Identidade da vacina no razão:** o lote de imunobiológico não usa CATMAT: `catalog_item_id` e `catmat_code` passam a ser nulos quando `item_kind = immunobiological`, com check (medicamento exige CATMAT; imunobiológico exige `immunobiological_code`); mínimos e aberturas chaveados pela chave do item (CATMAT ou código do imunobiológico).
- **Papéis na sala e na rede de frio:** a equipe da sala tem `vaccinator` e `room_lead` (enfermeiro responsável, CBO 2235\*); a rede de frio usa `stock_keeper`. Qualquer vacinador abre frasco, aplica, registra temperatura e recebe transferência; perda física, inventário e a conduta depois de temperatura fora da faixa são do `room_lead` (na sala) ou do `stock_keeper` (na central). A conduta "dar perda" segue o limite de aprovação de ajuste do 27a.
- **Credencial RNDS por cidade:** um certificado A1 (e-CNPJ da secretaria, `.pfx` + senha, cifrados) por cidade; o CNES de cada dose vai no payload. Se a homologação mostrar que a RNDS exige credenciamento ou token por CNES, vira pendência de go-live e a credencial ganha o CNES como chave, sem mudar o resto.
- **Linhas de cuidado:** os códigos `vaccine:*` do catálogo são convertidos para BRImunobiologico na tarefa (conferindo cada um); `vaccine:c2_schedule` segue a **nota metodológica do C2** (lista de códigos, intervalo mínimo de 30 dias, SCR a partir de 12 meses), não o calendário curado.
- **Sem ficha LEDI da dose:** a dose não gera ficha LEDI (Portaria 5.663 veda o envio de vacina por LEDI de terceiros). Se, na mesma ida à unidade, houver outro atendimento, ele segue o seu fluxo normal; a sala de vacina não gera atendimento individual.

## 3. Dados

Cidade:
- `vaccination_record_checks` (imutável): paciente, sala, autor, hora, modo (`transcribed` | `confirmed_complete` | `born_after_go_live`).
- `vaccination_statuses` (cópia recalculável): paciente, imunobiológico, aplicadas, devida, em atraso, próxima data.
- `vaccine_rooms`: unidade, nome, local de estoque (`stock_locations`), ativa; `vaccine_room_staff` (vacinadores).
- `immunizations` (imutável): paciente, sala, vacinador (e CBO), imunobiológico, lote (FK ao `stock_lots`, nulo na transcrição), lote e fabricante em texto (transcrição), dose, estratégia, grupo de atendimento, via, local, condição maternal, `applied_at`, `transcription`, frasco aberto de origem, campanha (opcional), `cancelled_by_id`/`replaces_id`, motivo do cancelamento.
- `rnds_outbox`: registro (aplicação ou cancelamento), payload cifrado, status (`pending sending accepted rejected failed`), identificador RNDS, mensagem de erro (sem dado pessoal), tentativas, prazo.
- `vial_openings`: lote, sala, aberto em, vence em, doses do frasco, doses aplicadas (calculado), perda técnica (calculado no vencimento).
- `stock_lots`: `item_kind` (`medication` | `immunobiological`), `immunobiological_code`; `stock_movements` ganha `vial_opening`.
- `cold_chain_equipments` e `cold_chain_readings` (imutável): equipamento, hora, atual, mínima, máxima, autor; `cold_chain_excursions`: leitura, lotes afetados, conduta, autor.
- Configuração versionada: `config/vaccination/calendar-2026.yml`, catálogo de imunobiológicos (código, nome, doses por frasco, validade após aberto por fabricante, via e local padrão), terminologias RNDS.

Plataforma: `IntegrationCredential::KINDS` ganha `rnds`; entradas no catálogo de interruptores.

## 4. Telas

- **Dashboard — sala de vacina:** busca do cidadão → caderneta (aplicadas, devidas hoje, em atraso, próximas) → registro da dose (lote em lista do estoque da sala, só lotes válidos e não bloqueados; frasco aberto sugerido) → confirmação com aprazamento. Transcrição. Cancelamento com motivo. Situação do envio à RNDS por dose e lista de pendências da sala.
- **Dashboard — estoque e frio:** frascos abertos com tempo restante, perdas, resumo mensal; equipamentos e leituras de temperatura.
- **Dashboard — painel e campanhas:** painel de vacinação; critério "vacina em atraso" e "convocados vacinados" na campanha.
- **Dashboard — integrações:** credencial RNDS (certificado A1 e senha, CNES solicitante).
- **wpda:** "Minhas vacinas" (caderneta e próximas doses).

## 5. Segurança e LGPD

- A caderneta é dado de saúde: leitura nominal com trilha `clinical_record.viewed`; o vacinador da sala lê no contexto da sala (registro de dose), sem abertura justificada; fora disso, a regra do prontuário (ADR 0031).
- Painéis, resumo mensal e alertas só com contagens.
- Certificado e senha cifrados, nunca em log, resposta ou evento; o payload FHIR (CPF, CNS, nome) cifrado na fila e anulado no aceite.
- Nada pessoal ou clínico (vacina, lote, dose) em URL, log, evento, `inspect` ou `as_json`.
- Imutabilidade por trigger em `db/city_triggers.sql` (`immunizations`, `cold_chain_readings`); novos `Platform.audit` em R18.

## 6. Testes

- api: regras do calendário (idade, intervalo mínimo, atraso, aprazamento, gestante, sexo) contra casos do PDF; registro, transcrição, cancelamento e substituição; lote fora da sala ou vencido recusado; abertura de frasco, consumo, vencimento e perda técnica; razão sem saldo negativo; montagem do RIA-R validada contra o perfil oficial; fila (backoff, prazo de 24 h, cancelamento só após aceite, payload anulado); validação NT 115; marcos das linhas de cuidado; critério da campanha; fonte da linha do tempo; trilha.
- dashboard e wpda: testes de componente das telas novas.
- Prova no ambiente de homologação da RNDS (gate de go-live; em dev, `rnds_mock`).
- Suíte completa do api no fim.

## 7. Entrega

api → dashboard → wpda → maintenance; runbook `operacao/rollout-sala-de-vacina.md` (credenciamento do estabelecimento na RNDS e certificado A1; cadastro das salas, vacinadores, equipamentos; inventário inicial dos lotes; transcrição das cadernetas sob demanda).

## 8. F-IDs (propostos)

| ID | Funcionalidade | Superfície |
|---|---|---|
| F-29.1 | Interruptores, credencial RNDS, sala de vacina e vacinadores | api, dashboard, maintenance |
| F-29.2 | Calendário, catálogo de imunobiológicos e caderneta | api, dashboard |
| F-29.3 | Registro da dose, transcrição e cancelamento | api, dashboard |
| F-29.4 | Envio à RNDS (RIA-R 2.0) com guarda do retorno | api, dashboard |
| F-29.5 | Estoque de imunobiológicos, frascos e perdas, resumo mensal | api, dashboard |
| F-29.6 | Cadeia de frio | api, dashboard |
| F-29.7 | Caderneta do cidadão e linha do tempo | api, wpda, dashboard |
| F-29.8 | Linhas de cuidado, campanhas, ESAVI e painel | api, dashboard |

## 9. Riscos e pendências

- **Perfil RIA-R 2.0 não lido na fonte:** o guia da RNDS não abriu; baixar o perfil no Portal de Serviços DATASUS antes da tarefa de montagem (gate de go-live: homologação).
- **Autenticação:** e-CNPJ ou e-CPF, validade do token e cabeçalhos não confirmados; credenciamento do estabelecimento é trabalho do município.
- **Retificação e cancelamento na RNDS:** o fluxo não foi achado em normativo; confirmar no perfil.
- **Calendário curado à mão:** risco de erro de transcrição do PDF; precisa de revisão por enfermagem antes do go-live, e de nova versão a cada nota técnica (adequação em 15 dias, Portaria 5.663).
- **Desativação do RIA-R 1.0/RIA-C 2.0** sem cronograma: nascemos no 2.0.
