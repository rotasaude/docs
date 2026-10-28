# Decisões de Arquitetura — Rota Saúde

Corpus consolidado (v2). Cada ADR documenta uma decisão vigente, completa, sem cadeia
de emendas. Numeração linear. O histórico pré-consolidação vive na pasta de arquivo
(ver [DE-PARA.md](DE-PARA.md)).

## Índice

| # | Item | Tema |
|---|------|------|
| 0001 | [Platform & application stack](0001.md) | Rails 8 + Solid Queue no Postgres; imagem única, papéis web/worker; identificadores em inglês |
| 0002 | [Repository topology (multi-repo)](0002.md) | Um repo por app + `contracts` + `docs`; compartilhar contrato, não código |
| 0003 | [Multi-tenant isolation (RLS)](0003.md) | **Substituído pelo ADR 0020.** Isolamento pelo Postgres; `SET LOCAL` por transação; `rota_app`/`rota_admin`; control plane vs data plane |
| 0004 | [Domain events, commands & write atomicity](0004.md) | Command muta + publica evento carimbado; `domain_events` imutável; enqueue após COMMIT |
| 0005 | [Consumer idempotency & side-effect isolation](0005.md) | `processed_events` exactly-once; corpo transacional; HTTP fora do lock |
| 0006 | [Queues & clinical priority](0006.md) | Filas por SLA; trabalho pesado nunca atrasa alerta urgente |
| 0007 | [WhatsApp ingestion & channel routing](0007.md) | Borda HMAC → rota → persiste sob tenant; dedup na borda |
| 0008 | [Conversation state & consent](0008.md) | Máquina de estados da conversa; `Consent` append-only, versionado, revogável |
| 0009 | [Protocol engine](0009.md) | Motor puro; `Outcome`; storage + façade; scoring weighted/decision_table |
| 0010 | [CQRS: projections & snapshots](0010.md) | `ReportSnapshot` imutável (prova); `DashboardMetric` reconstrutível |
| 0011 | [Identity, session & MFA](0011.md) | Auth nativa Rails 8; identidade global RLS-exempt; MFA operador/publisher |
| 0012 | [Authorization: RBAC & memberships](0012.md) | `memberships (user, city, role)`; operador = cidade nula; append-only; quatro-olhos de protocolo (ADR 0016) |
| 0013 | [Provisioning, secrets & key custody](0013.md) | Secrets via Kamal; AR Encryption; uma chave no env do `api`; `ProvisionMunicipality` |
| 0014 | [Audit log, retention & LGPD](0014.md) | `domain_events` como evidência imutável; replay; purga por retenção |
| 0015 | [Contracts & versioning](0015.md) | SemVer por domínio; expand/contract para MAJOR; tolerância do consumidor |
| 0016 | [Protocol signatures: two city reviewers per publication and per activation](0016.md) | `protocol_reviewer`; duas assinaturas para publicar e para ativar; substitui o quatro-olhos do ADR 0012 |
| 0017 | [Web citizen channel: declared identity and SMS-confirmed session](0017.md) | Cidadão entra pelo `wpda` com CPF declarado + celular por SMS; mesmo contrato de dados; WhatsApp desligável por cidade |
| 0018 | [Health units and attendances: from remote triage to in-person care](0018.md) | `health_units` mínima; check-in na unidade abre `attendances` a partir da triagem; desfecho e encaminhamento descritivo; código do balcão com finalidade |
| 0019 | [Calling, follow-up requests and appointments: care continues after the outcome](0019.md) | Papel `health_professional` chama e dá o desfecho; retorno e encaminhamento geram pedido; recepção marca horário; cidadão confirma (24h antes) ou cancela; atendimento nasce de triagem ou de horário |
| 0020 | [Database per city: the city is the database, not a column](0020.md) | Um banco e um role por cidade + banco de plataforma; Host escolhe a conexão antes da autenticação; worker por cidade; grant de operador; chave derivada por cidade; substitui o ADR 0003 |
| 0021 | [Health professionals: a user profile linked to units, with dated shifts](0021.md) | Perfil do profissional 1:1 com o usuário; vínculo com unidade e CBO com início e fim; turnos com data; chamada e desfecho exigem vínculo ativo com a unidade; o turno não bloqueia ato clínico |
| 0022 | [City dashboard panels read live, not from the projection](0022.md) | Painéis da cidade agregam ao vivo por janela de período; `as_of` = instante da leitura, carimbado em todo agregado; `DashboardMetric` só para manutenção e painel de Saúde |
| 0023 | [Territory by declared neighborhood, without geometry](0023.md) | Bairros por cidade (semente + edição do admin), cobertura bairro→unidades, bairro declarado pelo cidadão e copiado na triagem; unidade de referência informa e sugere, nunca restringe; filtro por bairro nos painéis com supressão de 1 a 4; CEP pelo navegador; sem PostGIS |
| 0024 | [Campaigns as wpda notices, with optional SMS per city](0024.md) | Aviso da cidade no wpda para público por recorte geográfico × critérios clínicos (E); público congelado no envio; mínimo de 5 telefones; SMS opcional por cidade, com opt-in e texto fixo; papel `campaign_manager` com step-up |
| 0025 | [Analytics as daily anonymous consolidation per city, with a fixed suppressed set for the platform](0025.md) | Consolidação diária anônima no banco da cidade (janela de 30 dias, D-1, 5 anos); supressão de 1 a 4 sempre; epidemiologia só por perguntas `analytic` (boolean/enum) marcadas no protocolo assinado; papel `analyst`; plataforma lê só 6 indicadores semanais suprimidos |
| 0026 | [Citizen erasure as an in-place tombstone, and revocation that keeps only what became a health record](0026.md) | Revogação anonimiza a triagem concluída sem atendimento e retém a que virou atendimento; trilha só com referência; exclusão no posto por duas pessoas (verificador pede, admin confirma com step-up), casca no lugar do CPF e celular; quem foi atendido fica retido |
| 0027 | [Triage catalog per citizen profile, with signed eligibility, city restriction and suggestions](0027.md) | Perfil do par (nascimento, sexo; declarado/conferido; obrigatório no wpda); elegibilidade e sugestões assinadas no protocolo na linguagem de condição; catálogo da cidade só restringe (soma com E); intervalo de repetição; sugestão nunca com resultado urgente |
| 0028 | [Record mode per city, per-city feature switches and integration credentials, and continuous LEDI export to the city's PEC](0028.md) | Modo `off`/`integrated`/`record` pelo operador; interruptores genéricos por cidade escritos só pelo `maintenance`; credenciais por cidade cifradas (`municipal_admin`); terminologias versionadas na plataforma; CNES importado com confirmação; envio contínuo do LEDI (Thrift/Ruby) ao PEC da cidade; CADSUS atrás de interruptor |
| 0029 | [Professional schedules from shifts and templates, computed slots, fit-ins, and appointment requests from triage](0029.md) | Agenda por turno com modelo opcional (demanda do dia, agendável, bloqueada); tipos de atendimento base + cidade com CBO; vagas calculadas com trava de sobreposição; encaixe com justificativa e limite; protocolo gera pedido (tipo, prioridade, prazo); recepção sempre marca; cidadão pede outro horário e recebe lembrete |
| 0030 | [Initial listening (acolhimento) with CAB 28 risk colours suggested by a signed rule set, its destination, and its LEDI record](0030.md) | Escuta inicial em registro próprio (queixa CIAP-2, sinais vitais), cor CAB 28 sugerida por protocolo assinado `screening`, destino (consulta no dia, agendar, orientação, encaminhar) com exceção na trava do módulo 13, ficha LEDI por CBO, fila de fichas só com códigos e retenção de 90 dias (api#43), trilha de leitura |
| 0031 | [Primary care clinical record without digital signature: patient per CPF, event-sourced problem list, finalized consultations with addenda, contextual reading](0031.md) | Prontuário da APS (19a) só no modo `record` e atrás de `clinical_record`; paciente por CPF de par validado; nome no cadastro; lista de problemas por eventos; consulta SOAP rascunho → finalizada imutável → adendos; leitura em contexto + abertura justificada com step-up e relatório; ficha de atendimento individual com correção pendente após aceite |
| 0032 | [Paperless clinical record with ICP-Brasil cloud signatures: per-shift signing session, pending queue, canonical JSON plus PDF, internal signer service](0032.md) | Assinatura digital do prontuário (19b) atrás de `digital_signature`; só ICP em nuvem pela API de PSC do ITI; sessão por turno com fila de pendentes e lote; adoção por profissional (sem certificado segue no papel); CAdES do JSON canônico + PAdES do PDF numa aprovação; AD-RB com provas de validade; serviço interno `signer` (Demoiselle) |
| 0033 | [Clinical documents issued from the consultation: common document header, structured prescriptions over a CATMAT-based catalog, event-sourced medication list, public verification page](0033.md) | Documentos clínicos (19c) atrás de `clinical_documents`: atestado, declaração (também pela recepção), receita comum (CATMAT + REMUME; enfermagem por protocolo; antimicrobiano em papel), requisição de exames; lista de medicamentos em uso por eventos; modo de emissão fixo; cancelamento pela autora; página pública de conferência por QR code |
| 0034 | [Controlled and antimicrobial prescriptions with SNCR numbers: prescription category, per-prescriber number stock via the prescriber's gov.br, paper notifications recorded, simulated SNCR first](0034.md) | Receita de controle especial (C1/C5) e de antimicrobiano digitais com número SNCR (19d) atrás de `controlled_prescriptions`; categoria na receita do 19c; estoque de números por prescritor via gov.br; Notificação A/B/B2 em papel registrada; SNCR simulado (`sncr_mock`) primeiro, real como gate |
| 0035 | [Patient timeline assembled at read time from per-module sources, with a professional view, a citizen view and a page for the PEC external systems screen](0035.md) | Linha do tempo do paciente (módulo 20) atrás de `patient_timeline`: visão do profissional (regras de leitura do 19a) e do cidadão (administrativo, documentos e resumo clínico só para par validado); "a pessoa" = paciente ou pares validados do mesmo CPF; montada na leitura por fornecedores; página para os "sistemas externos" do PEC (`pec_external_view`, HMAC, vínculo na unidade do CNES) |
| 0036 | [Exam tracking: exam orders with an append-only lifecycle in the city database, simple regulation queue, provider portal, FHIR inbound at the edge, per-exam release policy](0036.md) | Rastreamento do exame (módulo 21, subprojeto 21a) atrás de `exam_tracking`: `exam_order` vindo da consulta, da guia da recepção ou do PEC, com trilha só de acréscimo; regulação simples por exame; prestador com papel próprio no dashboard; entrada FHIR com token por prestador (`fake_lab` fora de produção); prazos por prestador e exame; ciência com fila da equipe; laudo ao cidadão por política do exame |
| 0037 | [Municipal laboratory workflow on top of exam tracking: versioned lab catalog from a platform base, samples and shipments, HL7 v2.5.1 inbound for middleware, release with ICP or advanced signature](0037.md) | Fluxo analítico do laboratório municipal (módulo 21, subprojeto 21b) atrás de `municipal_lab`: o 21a é a fonte única do exame; catálogo-base da plataforma com versões por laboratório; coleta na UBS com remessa ou no laboratório; etiqueta Code 128, rejeição e recoleta; resultados digitados ou por HL7 v2.5.1 do middleware (`fake_middleware` fora de produção); `lab_staff` e `lab_releaser`; laudo da RDC 978 com ICP ou assinatura avançada (médico só ICP); valores críticos; retificação por novo laudo |
| 0038 | [Municipal laboratory quality and supplies: internal QC with Westgard evaluated on write and a release block, EQA records, reagent lots traced to every result, estimated consumption without automatic deduction](0038.md) | Qualidade e insumos do laboratório municipal (módulo 21, subprojeto 21c) atrás de `lab_quality_control` e `lab_inventory`: CIQ com lotes, níveis e referência (bula, depois laboratório com aceite do RT), valores digitados ou pela HL7, Westgard avaliado na gravação com bloqueio da liberação, exceção do RT e revisão; registro do CEQ; estoque com lotes e movimentos só de acréscimo, lote em uso guardado em cada resultado e controle, consumo estimado sem baixa automática |
| 0039 | [Ambulatory billing: performed procedures checked against SIGTAP without blocking care, only BPA-instrument procedures exported, monthly BPA-I/BPA-C interface file per establishment](0039.md) | Procedimentos e faturamento ambulatorial (módulo 22) atrás de `procedure_billing`: procedimento realizado (atendimento, consulta, avulso, laudo do laboratório municipal, lançamento manual) com crítica SIGTAP que marca e não bloqueia; BPA só de estabelecimentos não-APS e do laboratório municipal (UBS nunca); fechamento por estabelecimento pelo `billing_clerk` com críticas recalculadas; arquivo de interface do BPA Magnético cifrado, com versões e download na trilha |
| 0040 | [Care lines: platform templates from the federal quality indicators, clinician-confirmed enrollment, follow-up computed at read time from existing sources, outreach panel and a non-official indicator preview](0040.md) | Linhas de cuidado (módulo 23) atrás de `care_lines` (só modo registro): seis linhas C2–C7 da plataforma e linhas próprias da cidade; sugestão por problema ativo, idade/sexo e condição declarada (inclusive cidadão validado que nunca consultou) com confirmação de médico ou enfermeiro; boas práticas avaliadas na leitura sobre consulta, escuta, exames e procedimentos (visita do ACS e vacina "não acompanhadas"); painel de busca ativa; prévia C2–C7 sobre a população elegível, sempre estimativa |
| 0041 | [Field app and telehealth video as separate projects: clinical domain stays in the Rota Saúde api, the specialised tool lives in its own repository behind a versioned contract](0041.md) | Módulos 24 e 34 divididos em núcleo e ferramenta: visita, cadastro, atividade coletiva, LEDI e teleconsulta no api/dashboard/wpda; app de campo offline (`rotasaude/campo`) e serviço de vídeo em repositórios próprios, integrados por contrato, sem nunca serem fonte de verdade de dado clínico |
| 0042 | [Territory registry: micro-areas, households and families, individual registration on the patient (who may have no CPF), immutable versions exported as LEDI registration records, team membership by registration](0042.md) | Cadastro territorial (módulo 24a) atrás de `territory_registry` (só modo registro): microáreas com ACS; domicílio e famílias; cadastro individual no paciente, que pode não ter CPF e nasce do cadastro; versões imutáveis viram fichas LEDI de cadastro; equipe pela microárea; óbito do cadastro é a marca de óbito das linhas; ACS e agente de endemias sem conselho |
| 0043 | [Disease surveillance: versioned platform list of notifiable diseases, suspicion confirmed at the consultation and completed by the surveillance team, safety net on finalization, manual entry into the official system](0043.md) | Vigilância epidemiológica (módulo 25) atrás de `surveillance`: lista nacional versionada com CID-10/CIAP-2 e doenças locais; suspeita confirmada na consulta sem bloquear e completada pela vigilância; rede de segurança na finalização; fila com andamento até a digitação no Sinan (número manual); prazos, alertas sem dado do paciente, negativa semanal; PDF, CSV e painel por bairro e semana epidemiológica com supressão |
| 0044 | [Municipal care regulation: structured referral from the consultation, regulated queue by risk colour and waiting time, quantity quotas per provider, coexistence with state systems](0044.md) | Regulação municipal (módulo 26a) atrás de `care_regulation` (requer `clinical_record`): pedido na consulta (especialidade ou SIGTAP, CID, justificativa, cor, guia); fila por cor e espera com o papel `care_regulator`; rede própria e prestadores externos com cota mensal em quantidade; agendamento pelo 17 ou externo; convivência com o estado por número externo e exportação; cidadão vê só a situação. 26b (APAC/OCI) e 26c (MIRA) depois |

## Itens em aberto

Reconhecidos como **não decididos** e ainda válidos no estado final do corpus. Cada um
vira ADR próprio quando entrar no ciclo.

### Arquitetura

- **UI de autoria de protocolos** — formato de edição, fluxo de revisão, preview lado
  a lado (ADR 0009).
- **Estado `:aborted` no `Outcome`** — a triagem revogada vive como
  `aborted_by_revocation` na `Triage`, sem estado correspondente no `Outcome`
  (ADR 0008/0009).
- **Mesclar as duas estratégias de scoring** no mesmo fluxo (ADR 0009).
- **Granularidade de consentimento por finalidade** (`Consent.scope`) e **canal de
  entrada por telefone (voz)** (ADR 0008; a web foi resolvida pelo ADR 0017).
- **Versionamento de schema de payload** de evento e **particionamento de
  `domain_events`** por volume (ADR 0004).
- **Seleção de canal no outbound** quando a cidade tiver mais de um número (ADR 0007).
- **Fila por município** (isolamento de carga) (ADR 0006).
- **Granularidade de `viewer`** e **delegação de `municipal_admin`** (ADR 0012).
- **Fatias do dashboard** por médico/posto e **payload para object storage** se o
  snapshot crescer (ADR 0010).
- **Reavaliação de Solid Queue / tamanho da imagem** sob carga (ADR 0001); **métricas
  agregadas do operador** (ADR 0003; o ADR 0020 removeu a visão entre cidades do console).
- **Mecanismo de distribuição de `contracts`** por domínio, **validação automática de
  categoria** e **compatibilidade de eventos persistidos** (ADR 0015).
- **Nome da organização**, **preservação de histórico do `api`** na extração e
  **release coordenado cross-repo** (ADR 0002).

### Confiabilidade e segurança clínica

- **Mecanismo de alerta urgente** — SLA + escalonamento + monitoramento +
  dead-letter-para-humano + plantão. O destino do alerta é config (ADR 0013); o
  mecanismo fim-a-fim é aberto (ADR 0006).
- **Idempotência HTTP** dos consumers de efeito externo (`NotifyCitizenJob`,
  `AlertMunicipalityJob`) (ADR 0005).
- **Corrida de primeiro contato** na ingestão (índice + tratamento explícito do
  conflito) (ADR 0007).

### Operação e custódia

- **Chave por tenant** (envelope encryption via KMS) — endurecimento pós-piloto
  (ADR 0013).
- **Rotação sem janela de break** de `WHATSAPP_APP_SECRET` e `REPORT_SIGNING_KEY`
  (ADR 0013).
- **Inventário de secrets por repo** (ADR 0013).
- **Suspensão/desprovisionamento de cidade** — retenção pós-saída, base legal
  (ADR 0013).
- **Biblioteca de templates de protocolo** (herança municipal/estadual) (ADR 0013/0009).
- **Retenção / TTL por município** (ADR 0003; com banco por cidade, ADR 0020).

### Identidade e LGPD

- **Integração gov.br OIDC** e **recovery de MFA** além dos recovery codes (ADR 0011).
- **Imutabilidade × Art. 18 LGPD** — o que é apagável vs retido por base legal
  (ADR 0014). Inclui a **exclusão do cadastro do cidadão** do canal web (ADR 0017):
  `citizens` guarda CPF e celular, além de `citizen_sessions` e `otp_challenges`;
  decidido fora do subprojeto 2 (validação presencial) para análise própria.
- **Art. 20 LGPD** — revisão humana de decisão automatizada (ADR 0014).
- **Assinante real de `consent.revoked`** e base legal de retenção pós-revogação
  (ADR 0014).
- **Reconciliação de identidade do cidadão com CNS/SUS** (ADR 0014).
- **Mascaramento de dados pessoais na API de manutenção** — adiado; até existir, a
  API não expõe dado de cidadão
  ([registro](../superpowers/specs/2026-09-17-mascaramento-dados-sensiveis-decisao.md))
  (ADR 0014).

## Proveniência

Este corpus sucede o anterior (26 ADRs com cadeia de emendas + uma nota de custódia). O
mapeamento completo está em [DE-PARA.md](DE-PARA.md). O corpus antigo é preservado como
arquivo histórico, read-only, em [`_v1/adr/`](../_v1/README.md). Como o corpus
v2 foi construído está em [`../../_refundacao/`](../../_refundacao/PROVENIENCIA.md).
