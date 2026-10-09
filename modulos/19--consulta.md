# Módulo 19 — Consulta (prontuário da APS)

- **Estado:** Em andamento (19a Fechado; 19b Planejado)
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
- Runbook do prontuário (ligar, validação com nome, abertura justificada):
  [`operacao/rollout-prontuario.md`](../operacao/rollout-prontuario.md).

## Critério de fechamento do 19b

- F-19.8 a F-19.15 verificadas.
- Gate de go-live api#53: assinatura real aprovada no validar.iti.gov.br (até lá o 19b fica `Entregue`).
- Runbook da assinatura (ligar, vincular, sessão, pendentes, revalidação).

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
