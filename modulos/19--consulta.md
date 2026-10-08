# Módulo 19 — Consulta (prontuário da APS)

- **Estado:** Planejado
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

**Próximos subprojetos:** 19c documentos clínicos (atestado, receita comum,
pedido de exame) e CATMAT; 19d receita de controlado (SNCR).

**Fora, por enquanto:** prontuário visível ao cidadão (20), odontologia (28).

## ADRs governantes

| ADR | Papel no módulo |
|---|---|
| 0031 | Paciente, nome, lista de problemas, consulta, adendos, leitura, ficha |
| 0032 | Assinatura digital (19b): PSC em nuvem, sessão, fila, formatos, `signer` |
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
| `maintenance` | Interruptores `clinical_record` e `digital_signature`; prestadores e estado do `signer` (19b) |
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

## Riscos herdados

- Papel assinado até o 19b.
- 19b: prova técnica (sandbox VIDaaS → validar.iti.gov.br), homologação em cada PSC, segundo PSC, carimbo do tempo, COFEN 801/2026, custo do certificado.
- Aceite e reenvio no PEC não observados (api#41).
- Pares já validados sem nome precisam completar.

## Critério de fechamento do módulo (19a)

- F-19.1 a F-19.7 verificadas.
- Suíte de invariante (`spec/invariants/clinical_record_invariants_spec.rb`).
- Runbook do prontuário (ligar, validação com nome, abertura justificada).

## Critério de fechamento do 19b

- F-19.8 a F-19.15 verificadas.
- Assinatura real aprovada no validar.iti.gov.br.
- Runbook da assinatura (ligar, vincular, sessão, pendentes, revalidação).

## Histórico

- 2026-10-05 — Registrado como `Stub` no portfólio do Ciclo 2.
- 2026-10-07 — Escopo do 19a decidido com o usuário; ADR 0031 e spec
  `superpowers/specs/2026-10-07-module-19-consultation-design.md`. F-19.1 a
  F-19.7 criados (board #1, `Not Started`). Módulo passa a `Planejado`.
- 2026-10-08 — Escopo do 19b decidido com o usuário; ADR 0032 e spec
  `superpowers/specs/2026-10-08-module-19b-digital-signature-design.md`.
  F-19.8 a F-19.15 criados (board #1, `Not Started`).
