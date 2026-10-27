# Módulo 25 — Vigilância epidemiológica — design

**Data:** 2026-10-27
**Status:** aprovado (2026-10-27)
**Afeta:**
- `apps/api`: plataforma — lista de doenças e agravos de notificação compulsória, versionada; cidade — doenças locais, notificações, andamento, notificação negativa semanal; sugestão na consulta; rede de segurança na finalização; alertas; ficha, CSV e relatório; painel por bairro e semana epidemiológica; papel `surveillance`; interruptor `surveillance`.
- `apps/dashboard`: sugestão de notificação na consulta; fila da vigilância; ficha e andamento; notificação manual; notificação negativa; relatórios; painel.
- `apps/maintenance`: interruptor (mecanismo genérico) e lista de doenças da plataforma.

**ADR:** `docs/adr/0043.md` · **Módulo:** `docs/modulos/25--vigilancia-epidemiologica.md` · **Pesquisa:** `docs/pesquisa/2026-10-05-integracoes/frente-5-vigilancia.md` (A5, v1)

**Fora desta entrega:** envio automático ao Sinan, e-SUS Sinan ou e-SUS Notifica (não há interface para terceiros); registro de dose de vacina (módulo 29; obrigaria a RNDS); declaração de óbito e de nascido vivo; integração com o GAL (só o número da requisição, que já existe no 21a); investigação de campo e controle de vetores (módulo 35); detecção espaço-temporal (v3 da pesquisa); o cidadão não vê a notificação no wpda.

## 1. Intenção

Nenhum caso de doença de notificação obrigatória atendido no Rota Saúde passa sem a vigilância saber, e nenhum caso imediato passa das 24 horas sem alguém ver. A vigilância municipal acompanha cada caso até estar digitado no Sinan e enxerga onde a doença está crescendo.

**Sucesso:** o médico confirma a suspeita em dois campos sem sair da consulta; a vigilância recebe o caso na fila, com prazo visível, completa a ficha, investiga, encerra e registra o número do Sinan; a unidade sem casos na semana registra a notificação negativa; o painel mostra casos por bairro e semana epidemiológica.

**Antiobjetivos:** bloquear a finalização da consulta por causa da notificação; prometer envio automático ao Sinan; dado do paciente em e-mail, log ou URL; apresentar o aumento atípico como surto declarado; tratar a lista de doenças como código.

## 2. Decisões

| # | Decisão |
|---|---|
| 1 | **Lista da plataforma, versionada** (Portaria 11.211/2026, anexo 1 do anexo V da PRC 4/2017): cada doença com vigência, prazo (imediata = 24 h; semanal = 7 dias) e CIDs que geram a sugestão. A cidade pode acrescentar doenças locais. |
| 2 | **Na consulta, o médico ou o enfermeiro só confirma a suspeita** (doença e data dos primeiros sintomas); a vigilância completa a ficha (escolha do usuário, opção A). A sugestão aparece quando um CID da lista é registrado e nunca bloqueia a finalização. |
| 3 | **Rede de segurança:** consulta finalizada com CID da lista e sem notificação gera "possível notificação não feita" na fila da vigilância. |
| 4 | **Notificação manual** pela vigilância, para casos de hospital, laboratório, outra cidade ou cidade sem modo prontuário. |
| 5 | **Papel `surveillance`** (vigilância municipal), privilegiado, com segundo fator; fila nominal com trilha. |
| 6 | **Andamento:** notificado → em investigação → encerrado (confirmado, descartado, inconclusivo) → digitado no sistema oficial, com o número informado à mão. |
| 7 | **Prazo** conta da data da suspeita: imediata 24 h, semanal 7 dias; vencida em destaque. **Alertas:** imediata gera alerta no painel e e-mail sem dado do paciente ("nova notificação imediata: dengue, UBS X"). **Notificação negativa** semanal por unidade, com lembrete. |
| 8 | **Saídas:** ficha em PDF no formato da ficha de notificação individual do Sinan; CSV por semana epidemiológica; relatório de casos sem número oficial. |
| 9 | **Painel** por bairro e semana epidemiológica (domingo a sábado), com a supressão do analytics (1–4 escondidos) e sinal de aumento atípico (semana × média das 4 anteriores), sempre rotulado como sinal para avaliar. |
| 10 | **Bairro da pessoa:** domicílio do cadastro territorial (24a); sem ele, o bairro do cidadão; sem nenhum, "não informado". |
| 11 | **Interruptor `surveillance`** sem exigir o modo prontuário; a sugestão na consulta só existe com `clinical_record`. |
| 12 | **Exclusão LGPD:** a notificação é obrigação legal e fica retida. |

Decisão 2 é do usuário; as demais são do coordenador, apresentadas e aprovadas.

**Decisões do coordenador (revisão da spec):**
- **Enfermeiro e CIAP-2:** a lista guarda, por doença, os CID-10 **e** os CIAP-2 correspondentes (quando existirem), porque o enfermeiro registra CIAP-2 e não pode registrar CID-10 novo. Além da sugestão por código, a consulta sempre oferece "Notificar suspeita" à mão, para médico e enfermeiro.
- **O que dispara e duplicidade:** só problema novo ou reativado na consulta (`add`, não reavaliação). Não há sugestão nem "possível não feita" quando o paciente já tem notificação da mesma doença aberta, ou encerrada há menos que a janela da doença (`repeat_after_days`, padrão 30; doenças de acompanhamento longo como tuberculose, hanseníase, HIV e sífilis usam janela própria definida na lista). No banco, no máximo uma notificação aberta por paciente e doença.
- **Paciente da notificação manual:** sempre um paciente da cidade, encontrado pela busca de CPF ou de duplicados do 24a ou criado como no cadastro (`CreateTyped`, sem CPF com justificativa). A ficha vem pré-preenchida com o que o paciente e o cadastro já têm (mãe, raça/cor, endereço, ocupação); o resto é digitado.
- **Morador de outra cidade:** a notificação fica na cidade do atendimento, marcada "residente em outro município", com o município de residência; é digitada no Sinan aqui, e o Sinan cuida da transferência. Nenhum fluxo entre bancos na v1.
- **Notificação negativa:** cada unidade ganha a marca "unidade notificadora" (padrão: todas as unidades de APS ativas); a vigilância registra a negativa em nome da unidade. Lembrete e alertas por e-mail vão para todos os membros ativos com papel `surveillance`.
- **Painel:** consulta direta, só na cidade (fora do pipeline do analytics e da plataforma), com semana epidemiológica própria (domingo a sábado, regra do Sinan para a semana 1); vê `surveillance` e `analyst` (agregado suprimido, sem nome).
- **Lista e CIDs da carga inicial:** a Task 0 do api monta a lista a partir da portaria e dos CID-10 que as próprias fichas do Sinan atribuem a cada doença, mais os CIAP-2 equivalentes, com fonte por linha. No maintenance, a lista é editada por um mantenedor com segundo fator e auditoria, publicando versão nova (como o catálogo-base do 23).
- **Doenças estaduais:** entram como doenças locais de cada cidade.

## 3. Interruptor

`surveillance`. Desligado: nenhuma sugestão, fila, rota ou alerta; o que foi gravado fica.

## 4. Dados

Plataforma: `notifiable_disease_versions` (versão da lista, portaria, vigência) e `notifiable_diseases` (código, nome, prazo, CIDs, ficha do Sinan correspondente).

Cidade:
- `local_notifiable_diseases` (doenças da cidade, mesma forma).
- `disease_notifications`: paciente (pode não ter CPF, ADR 0042), doença (código e versão da lista), data da suspeita, data dos primeiros sintomas, unidade (CNES), notificador, origem (`consultation` \| `manual` \| `safety_net`), consulta (quando houver), ficha complementar (cifrada), situação, classificação final, número oficial, prazo.
- `disease_notification_events` (andamento, imutável, com autor e motivo).
- `weekly_negative_reports` (unidade, semana epidemiológica, autor).

## 5. Telas (dashboard)

- **Consulta:** aviso "CID de notificação obrigatória: notificar suspeita de X" com data dos primeiros sintomas; também ao finalizar, sem bloquear.
- **Vigilância:** fila com prazo (vencidas e imediatas no topo), ficha, andamento, número oficial, notificação manual, possíveis não feitas.
- **Notificação negativa:** por unidade e semana.
- **Relatórios:** ficha em PDF, CSV por semana, sem número oficial.
- **Painel:** bairro × semana epidemiológica, com supressão e sinal.

## 6. Segurança e LGPD

Trilha `clinical_record.viewed` em leitura nominal (sem polling); contagens sem trilha. Ficha complementar cifrada. Nada pessoal em e-mail, evento, log, URL, `inspect` ou `as_json`. Eventos novos de `Platform.audit` declarados em R18.

## 7. Testes

- api: lista versionada (vigência); sugestão por CID; rede de segurança idempotente; andamento e transições inválidas; prazos no fuso da cidade; semana epidemiológica (virada de ano); e-mail sem dado pessoal; supressão e sinal; retenção LGPD; escopo do papel.
- dashboard: telas com testes de componente; sugestão que não bloqueia.
- Suíte completa do api no fim.

## 8. Entrega

api → dashboard → maintenance; runbook `operacao/rollout-vigilancia.md`.

## 9. F-IDs (propostos)

| ID | Funcionalidade | Superfície |
|---|---|---|
| F-25.1 | Interruptor, papel de vigilância e lista de doenças | api, dashboard, maintenance |
| F-25.2 | Sugestão de notificação na consulta | api, dashboard |
| F-25.3 | Rede de segurança na finalização | api |
| F-25.4 | Fila, notificação manual e ficha | api, dashboard |
| F-25.5 | Andamento, encerramento e número oficial | api, dashboard |
| F-25.6 | Prazos, alertas e notificação negativa | api, dashboard |
| F-25.7 | Ficha em PDF, CSV por semana e relatório sem número | api, dashboard |
| F-25.8 | Painel por bairro e semana epidemiológica | api, dashboard |

## 10. Riscos

- Lista e CIDs mudam (mitigação: prova técnica na Task 0 e lista versionada editável no maintenance).
- Ficha do Sinan varia por doença (mitigação: a v1 gera a ficha de notificação individual comum; as fichas de investigação específicas ficam fora).
- Subnotificação se o médico ignorar a sugestão (mitigação: rede de segurança).
