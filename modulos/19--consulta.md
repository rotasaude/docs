# Módulo 19 — Consulta (prontuário da APS)

- **Estado:** Em andamento (19a Fechado; 19b Entregue — gate api#53 aberto; 19c Entregue — gates api#56/#57 antes de cidade real; 19d Planejado)
- **Tipo:** Ciclo 2

## Escopo

Prontuário da atenção primária, dividido em subprojetos. O **19a** (este
planejamento) registra a consulta e gera a ficha LEDI de atendimento individual
(ADR 0031); o registro finalizado é impresso e assinado à mão até o 19b.

**Planejado (F-19.1 a F-19.7):**
- paciente por CPF ligado aos pares validados; nome completo, nome social e
  nome da mãe na validação presencial;
- lista de problemas do paciente por eventos, alimentada pela consulta;
- consulta SOAP: rascunho, problemas, condutas, exames (SIGTAP), encaminhamento,
  finalização com o desfecho do atendimento;
- adendos e impresso para assinatura manual;
- leitura em contexto, abertura justificada com step-up e relatório;
- ficha LEDI de atendimento individual, com correção pendente após aceite;
- interruptor `clinical_record` e pré-condições.

**Planejado no 19b (F-19.8 a F-19.15, ADR 0032; executa depois do 19a fechado):**
assinatura ICP-Brasil em nuvem (API de PSC do ITI) da consulta e do adendo,
atrás do interruptor `digital_signature`; sessão por turno com fila de
pendentes e lote; adoção por profissional (sem certificado segue no papel);
CAdES do JSON canônico + PAdES do PDF; serviço interno `signer`.

**Planejado no 19c (F-19.16 a F-19.23, ADR 0033):** atestado, declaração de
comparecimento (também pela recepção), receita comum com catálogo do CATMAT,
REMUME e protocolos de enfermagem (antimicrobiano em papel), requisição de
exames; lista de medicamentos em uso por eventos; página pública de conferência
por QR code e cancelamento, atrás do interruptor `clinical_documents`.

**Planejado no 19d (F-19.24 a F-19.29, ADR 0034):** receita de controle
especial (C1/C5) e de antimicrobiano digitais com número do SNCR (estoque por
prescritor via gov.br), Notificação A/B/B2 em papel registrada na consulta,
SNCR simulado primeiro (`sncr_mock`), atrás do interruptor
`controlled_prescriptions`.

**Fora, por enquanto:** prontuário visível ao cidadão (20), odontologia (28).

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0031 | Paciente, nome, lista de problemas, consulta, adendos, leitura, ficha |
| 0032 | Assinatura digital (19b): PSC em nuvem, sessão, fila, formatos, `signer` |
| 0033 | Documentos clínicos (19c): cabeçalho comum, receita estruturada, medicamentos em uso, conferência |
| 0034 | Controlado e antimicrobiano (19d): categoria, estoque SNCR, Notificação registrada |
| 0028 | Modo `record`, interruptores, exportação, terminologias |
| 0030 | Escuta inicial, trilha de leitura, fichas não geradas |
| 0017, 0027 | Par e validação presencial, perfil |
| 0026 | Retenção do registro de saúde |

## Superfícies

| Superfície | O que aparece |
|---|---|
| `api` | `patients`, lista de problemas, consultas, adendos, aberturas, nomes, `Ledi::Fichas::IndividualCare` |
| `admin` | — |
| `dashboard` | Painel do paciente, editor da consulta, adendo, impresso, prontuário fora de contexto, relatório, nomes na validação |
| `maintenance` | Interruptores `clinical_record`, `digital_signature` e `clinical_documents`; prestadores e estado do `signer` (19b); tela Medicamentos (CATMAT e listas Anvisa, 19c) |
| `signer` | Serviço interno de assinatura (19b) |
| `wpda` | — |

## Pré-requisitos

- Módulos 13, 15, 16 (modo `record`, exportação), 18 (escuta, busca CIAP-2).

## Funcionalidades planejadas

| ID | Funcionalidade | Superfície | ADRs |
|---|---|---|---|
| F-19.1 | Paciente por CPF ligado aos pares validados, e nome no cadastro | api, dashboard | 0031, 0017 |
| F-19.2 | Lista de problemas do paciente por eventos, alimentada pela consulta | api, dashboard | 0031 |
| F-19.3 | Consulta SOAP: rascunho, itens estruturados, finalização com desfecho | api, dashboard | 0031, 0019 |
| F-19.4 | Adendos e impresso para assinatura manual | api, dashboard | 0031 |
| F-19.5 | Leitura em contexto, abertura justificada com step-up e relatório | api, dashboard | 0031, 0016 |
| F-19.6 | Ficha LEDI de atendimento individual, com confirmação do layout e correção pendente após aceite | api, dashboard | 0031, 0028 |
| F-19.7 | Interruptor `clinical_record` e pré-condições | api, maintenance | 0031, 0028 |
| F-19.8 | Interruptor `digital_signature` e prestadores no maintenance | api, maintenance | 0032, 0028 |
| F-19.9 | Vínculo de certificado ICP em nuvem por CPF | api, dashboard | 0032, 0016 |
| F-19.10 | Sessão de assinatura por turno | api, dashboard | 0032 |
| F-19.11 | Assinatura automática da consulta e do adendo (CAdES + PAdES AD-RB) | api, signer | 0032, 0031 |
| F-19.12 | Fila de pendentes, lote e volta ao papel | api, dashboard | 0032 |
| F-19.13 | Validação, conteúdo assinado e exportação | api, dashboard | 0032 |
| F-19.14 | Painel de assinatura do admin | api, dashboard | 0032 |
| F-19.15 | Serviço `signer` | signer | 0032 |
| F-19.16 | Interruptor `clinical_documents` e importação do catálogo de medicamentos | api, maintenance | 0033, 0028 |
| F-19.17 | REMUME e protocolos de enfermagem | api, dashboard | 0033 |
| F-19.18 | Lista de medicamentos em uso por eventos | api, dashboard | 0033 |
| F-19.19 | Receita comum (enfermagem por protocolo; antimicrobiano em papel) | api, dashboard | 0033, 0032 |
| F-19.20 | Atestado | api, dashboard | 0033, 0032 |
| F-19.21 | Declaração de comparecimento (consulta e recepção) | api, dashboard | 0033 |
| F-19.22 | Requisição de exames em PDF | api, dashboard | 0033, 0031 |
| F-19.23 | Página pública de conferência e cancelamento | api, dashboard | 0033 |
| F-19.24 | Interruptores `controlled_prescriptions` e `sncr_mock` e configuração do SNCR | api, maintenance | 0034, 0028 |
| F-19.25 | Estoque de números SNCR por prescritor | api, dashboard | 0034 |
| F-19.26 | Receita de controle especial digital | api, dashboard | 0034, 0032 |
| F-19.27 | Receita de antimicrobiano digital | api, dashboard | 0034, 0032 |
| F-19.28 | Registro da Notificação de papel | api, dashboard | 0034 |
| F-19.29 | Painel do SNCR do admin | api, dashboard | 0034 |

## Riscos herdados

- Papel assinado até o 19b.
- 19b: prova técnica (sandbox VIDaaS → validar.iti.gov.br), homologação em cada PSC, segundo PSC, carimbo do tempo, COFEN 801/2026, custo do certificado.
- Aceite e reenvio no PEC não observados (api#41).
- Pares já validados sem nome precisam completar.

## Critério de fechamento do módulo (19a)

- F-19.1 a F-19.7 verificadas.
- Suíte de invariante (`spec/invariants/clinical_record_invariants_spec.rb`).
- Runbook do prontuário (ligar, validação com nome, abertura justificada):
  [`operacao/rollout-prontuario.md`](../operacao/rollout-prontuario.md).

## Critério de fechamento do 19b

- F-19.8 a F-19.15 verificadas.
- Gate de go-live api#53: assinatura real aprovada no validar.iti.gov.br (até lá o 19b fica `Entregue`).
- Runbook da assinatura (ligar, vincular, sessão, pendentes, revalidação):
  [`operacao/rollout-assinatura.md`](../operacao/rollout-assinatura.md).

## Critério de fechamento do 19c

- F-19.16 a F-19.23 verificadas.
- Prova no navegador (atestado e receita assinados, receita de enfermagem no
  protocolo, declaração da recepção, conferência pelo QR, cancelamento).
- Runbook dos documentos (catálogo, REMUME, protocolos, conferência):
  [`operacao/rollout-documentos-clinicos.md`](../operacao/rollout-documentos-clinicos.md).

## Critério de fechamento do 19d

- F-19.24 a F-19.29 verificadas.
- Prova no navegador com SNCR e PSC simulados.
- Gate de go-live: prova com o SNCR real (até lá o 19d fica `Entregue`).

## Histórico

- 2026-10-05 — Registrado como `Stub` no portfólio do Ciclo 2.
- 2026-10-07 — Escopo do 19a decidido com o usuário; ADR 0031 e spec
  `superpowers/specs/2026-10-07-module-19-consultation-design.md`. F-19.1 a
  F-19.7 criados (board #1, `Not Started`). Módulo passa a `Planejado`.
- 2026-10-08 — Escopo do 19b decidido com o usuário; ADR 0032 e spec
  `superpowers/specs/2026-10-08-module-19b-digital-signature-design.md`.
  F-19.8 a F-19.15 criados (board #1, `Not Started`).
- 2026-10-09 — **19a implementado, publicado e fechado:**
  - api `2a3ec97` (36 commits sobre `eb732e8`; suíte 4396/0);
  - dashboard `c4f7d5a` (31 commits sobre `4bf895b`; 1570 testes).

  Migrações de cidade `20261007400001` (paciente, nomes, lista de problemas),
  `20261007400002` (consulta, itens, adendos, aberturas), `20261007400003`
  (`correction_pending`) e `20261007400004` (leituras administrativas), todas
  **irreversíveis**. O impresso usa a gem Prawn, então a imagem do api precisa
  ser reconstruída. Suíte de invariante
  `spec/invariants/clinical_record_invariants_spec.rb`; runbook
  [`operacao/rollout-prontuario.md`](../operacao/rollout-prontuario.md).

  **Prova no navegador aprovada** (Curitiba, UBS Jardim das Flores, 2026-10-09):
  - "Minhas consultas" e impresso da autora fora do atendimento;
  - consulta nova de ponta a ponta (texto com "≥" e emoji, PA, K86 incluído,
    T90 avaliado, conduta);
  - impresso logo depois de finalizar;
  - ficha registrada como "não gerada" (`unit_without_cnes`,
    `professional_without_team`), como na prova do 18;
  - leitura administrativa com step-up, sem impresso nem adendo, e registrada
    no relatório de aberturas.

  **Decisões do usuário durante a execução** (contrato, bloco do mesmo nome;
  revisão do ADR 0031 de 2026-10-09):
  - CID-10 só para médicos (`2251`–`2253`); o não médico avalia ou resolve o
    CID-10 que já está na lista, mas a ficha dele sai sem CID-10. Ele precisa
    avaliar ao menos um problema em CIAP-2 para finalizar
    (`ciap2_required_for_cbo`);
  - chave desconhecida no salvamento automático é ignorada;
  - o varredor das 23h passa a gerar também a ficha atrasada da consulta;
  - a autora lê e imprime a própria consulta a qualquer momento ("Minhas
    consultas"); o `municipal_admin` lê o conteúdo completo de cada
    profissional, só leitura e com step-up; o outro profissional só lê.
    Impresso e adendo passam a ser só da autora;
  - leituras administrativas guardadas para sempre em tabela própria.

  F-19.1 a F-19.7 `Verified`. O 19b segue `Planejado` (F-19.8 a F-19.15).

  **Em aberto:**
  - rotasaude/api#51 (atendimento preso com rascunho órfão; Alta, antes do
    go-live);
  - rotasaude/api#52 (adendo que não chega à ficha em corrida rara; Média);
  - rotasaude/dashboard#12 (lista de problemas no adendo de "Minhas
    consultas"; Baixa);
  - a SIGTAP da semente de dev tem só 5 procedimentos (1 do grupo 02).
- 2026-10-09 — **19b implementado, publicado e Entregue** (o gate rotasaude/api#53
  continua aberto):
  - contracts `ff7df1d` (tag `clinical-v1.0.0`; `competence` AAAAMM nos exames);
  - signer — repositório novo `rotasaude/signer`, `70475f7` (63 testes);
  - api `c9ccd88` (31 commits sobre `2a3ec97`; suíte 4675/0);
  - dashboard `8a0c0de` (16 commits sobre `c4f7d5a`; 1735 testes);
  - maintenance `0f0a779` (3 commits; 217 testes).

  Migrações `20261008500001`, de plataforma e de cidade (esta **irreversível**).
  Fila `signatures` com worker próprio por cidade. Runbook
  [`operacao/rollout-assinatura.md`](../operacao/rollout-assinatura.md).

  **Decisões do usuário durante a execução** (contrato do 19b, §13; revisão do
  ADR 0032 de 2026-10-09):
  - o 19b foi verificado contra o **PSC simulado** e a AC de teste do `signer`;
    a prova com prestador real virou o gate de go-live api#53;
  - interruptor `signature_psc_mock` no maintenance, só fora de produção, com
    o aviso "simulada — sem validade jurídica" em toda tela, PDF e pacote;
  - competência da SIGTAP no JSON canônico dos exames;
  - o pedido de assinatura nasce na finalização e no adendo, quando há
    certificado ativo, com fila própria `signatures` e worker dedicado.

  **Prova no navegador aprovada** (Curitiba, 2026-10-09, PSC simulado):
  - vínculo do certificado e sessão do turno;
  - consulta finalizada já pendente e assinada sozinha;
  - detalhe, PDF, pacote e Revalidar (`valid`, AD-RB);
  - pendentes, volta ao papel e "Assinar todas" (`multi_signature`);
  - desvincular;
  - painel e leitura do admin;
  - aviso, interruptor, prestadores e estado do `signer` no maintenance.

  A prova achou os DELETE de sessão e de certificado sem `Content-Type` JSON
  (415, guarda CSRF), corrigido no dashboard `8a0c0de`.

  F-19.8 a F-19.15 `Verified`.

  **Em aberto:**
  - rotasaude/api#53 (prova real no validar.iti; Crítica, bloqueia o go-live);
  - rotasaude/signer#1 (endurecimento do `signer`; Alta);
  - rotasaude/dashboard#14 (marcador "assinando…" sem sessão; Baixa);
  - conexões do worker `signatures` no `max_connections`;
  - chamadas ao PSC e ao `signer` dentro da transação com lock;
  - sessão de usuário desativado vale até 12 h.
- 2026-10-09 — Escopo do 19c decidido com o usuário; ADR 0033 e spec
  `superpowers/specs/2026-10-09-module-19c-clinical-documents-design.md`.
  F-19.16 a F-19.23 criados (board #1, `Not Started`).
- 2026-10-10 — **19c implementado, publicado e Entregue:**
  - contracts `ea08b05` (tag `clinical-v1.1.0`; `dosage_form` nulo aceito);
  - api `db8f04a` (sobre `c9ccd88`; revisão final em `cff6539` com suíte
    4968/0, mais `248ecbf`, `ba80ae9` e `db8f04a` depois da prova);
  - dashboard `74791e0`;
  - maintenance `bc51914`.

  Migrações `20261009600001` de plataforma (catálogo de medicamentos e listas
  Anvisa, por `bin/rails db:migrate`) e de cidade (documentos, receita,
  medicamentos em uso, REMUME, protocolos, conferência; **irreversível**). O
  QR code usa a gem `rqrcode_core`, então a imagem do api precisa ser
  reconstruída. Carga do catálogo por `medications:import_anvisa` e depois
  `medications:import_catmat`; proxy da cidade encaminhando `/v`. Runbook
  [`operacao/rollout-documentos-clinicos.md`](../operacao/rollout-documentos-clinicos.md).

  **Decisões do usuário durante a execução** (contrato do 19c, §12):
  - `catalog_item.dosage_form` pode ser nulo já na `clinical-v1.1.0`; o nome
    impresso do item é o rótulo do catálogo;
  - declaração pela recepção vale para a cidade inteira (restringir à unidade
    é api#59);
  - QR code no fim de cada via do documento;
  - atestado de acompanhante com início opcional e sem dias; renovação só em
    contexto; `/v` sem interruptor e com token mascarado no log;
  - requisição de exames só depois de finalizar a consulta.

  **Prova no navegador feita** (Curitiba, 2026-10-10). Ela levou a corrigir o
  item completo do catálogo no protocolo (com o aviso de antimicrobiano antes
  de emitir, inclusive para a enfermagem), o timeout de 300 s das importações
  do maintenance, o texto do PDF e a requisição de exames em rascunho.

  F-19.16 a F-19.23 `Verified`.

  **Em aberto:**
  - rotasaude/api#56 (conferência farmacêutica das listas Anvisa; Crítica,
    bloqueia cidade real);
  - rotasaude/api#57 (curadoria do CATMAT; Alta, bloqueia cidade real);
  - rotasaude/api#62 (access log do proxy sem o token de `/v` e `/r`, e
    `trusted_proxies`; bloqueia cidade real);
  - norma da cidade para a declaração pela recepção (bloqueia cidade real);
  - rotasaude/api#58 (RNDS e OBM), api#59 (declarações da recepção e
    restrição à unidade), api#60 (REMUME e protocolos antes do interruptor),
    api#61 (histórico de versões dos protocolos).
- 2026-10-10 — Escopo do 19d decidido com o usuário; ADR 0034 e spec
  `superpowers/specs/2026-10-10-module-19d-controlled-prescriptions-design.md`.
  F-19.24 a F-19.29 criados (board #1, `Not Started`).
