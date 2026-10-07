# Módulo 19 — Consulta (prontuário da APS), subprojeto 19a — design

**Data:** 2026-10-07
**Status:** aprovado em conversa (2026-10-07), aguardando revisão do texto
**Afeta:**
- `apps/api`: cidade — `patients`, `patient_problems`, `patient_problem_events`, `consultations`, `consultation_problems`, `consultation_conducts`, `consultation_exam_requests`, `consultation_addenda`, `clinical_record_openings`, nome em `citizens`, `citizens.patient_id`; interruptor `clinical_record`; `Patients::*`, `Consultations::*`, `ClinicalRecord::Access`, `Ledi::Fichas::IndividualCare`; rotas em `/attendance` e `/clinical_record`.
- `apps/dashboard`: painel do paciente e editor da consulta no atendimento, consulta finalizada (adendo, imprimir), prontuário fora de contexto, relatório de aberturas, nomes na validação presencial, correção pendente na Produção.
- `apps/maintenance`: só o interruptor `clinical_record` (mecanismo genérico, sem código novo).
- `apps/wpda`, `apps/admin`, `contracts`: nada.

**ADR:** `docs/adr/0031.md` · **Módulo:** `docs/modulos/19--consulta.md`

**Fora desta entrega (subprojetos seguintes):** 19b assinatura ICP-Brasil/NGS2; 19c documentos clínicos e CATMAT; 19d receita de controlado (SNCR). Também fora: prontuário visível ao cidadão (20), odontologia (28).

## 1. Ponto de partida

- Atendimento (ADR 0018/0019/0030): `waiting` → `in_care` → `closed`; desfecho na chamada do profissional; escuta inicial com trilha `screening.viewed`.
- Par (CPF, celular) do ADR 0017; validação presencial (`Citizens::Verify`) confere nascimento e sexo (ADR 0027), CNS via CADSUS (ADR 0028). `citizens` não tem nome.
- Exportação (ADR 0028/0030): `Ledi::Ficha`, `ledi_outbox` com códigos de erro, "não gerada", regeneração de recusada; terminologias CID-10/CIAP-2/SIGTAP; busca de CIAP-2 (`POST /attendance/ciap2/search`); aceite real do PEC não observado (api#41).
- Interruptores por cidade (`Platform::Features`, `city_features`), escritos pelo maintenance.
- Pedido de agendamento (ADR 0029).

## 2. Decisões desta conversa

| # | Decisão |
|---|---|
| 1 | 19a sozinho primeiro (registro e ficha), impresso e assinado à mão até o 19b. |
| 2 | Lista de problemas do paciente, alimentada pela consulta. |
| 3 | Prontuário do CPF; consulta exige par validado. |
| 4 | SOAP em texto; problemas, condutas, exames (SIGTAP) e encaminhamento estruturados; medicamentos em texto até o 19c. |
| 5 | Rascunho → finalizado imutável → adendos. |
| 6 | Leitura em contexto + abertura justificada com relatório. |
| 7 | Tabelas relacionais com texto clínico cifrado. |
| 8 | Nome completo, nome social e nome da mãe entram agora, na validação presencial. |

## 3. Paciente, nome e lista de problemas

- `citizens` ganha `full_name`, `social_name`, `mother_name` (cifrados, não determinísticos) e `patient_id`. `Citizens::Verify` passa a receber e conferir os nomes (nome completo obrigatório; social e mãe opcionais). Nome de exibição = `social_name` se presente, senão `full_name`.
- `patients`: `cpf` (cifra determinística, único), `full_name`, `social_name`, `mother_name`, `birth_date`, `sex` (do par validado mais recente), `created_at`; `patient_profile_divergences` (registro quando dois pares validados divergem em nascimento/sexo).
- `Patients::Resolve.call(citizen)`: exige `verification_level = verified`; acha ou cria o paciente pelo CPF (lock por CPF); liga o par.
- `patient_problems`: `patient_id`, `terminology` (`ciap2`|`cid10`), `code`, `terminology_release_id`, `status` (`active`|`resolved`), `onset_on`, `onset_precision` (`day`|`month`|`year`|null), `resolved_on`; índice único parcial `(patient_id, terminology, code) WHERE status = 'active'`.
- `patient_problem_events` (só acréscimos): `patient_problem_id`, `kind` (`added`|`resolved`|`reactivated`|`onset_corrected`), `consultation_id` ou `addendum_id`, `user_id`, `created_at`, valores novos; trigger impede update/delete.
- `Patients::ApplyProblemEvent` é o único caminho de escrita da lista.

## 4. Consulta

- Interruptor `clinical_record` no catálogo de `Platform::Features` (requer `record_mode = record`).
- `consultations`: `attendance_id` (único), `patient_id`, `author_user_id`, `professional_link_id`, `cbo_code`, `status` (`draft`|`finalized`), `subjective`/`objective`/`assessment`/`plan` (cifrados), sinais vitais estruturados (mesmas colunas e validador do módulo 18), `care_type` (código LEDI), `started_at`, `finalized_at`.
- `consultation_problems` (avaliados, com situação no momento), `consultation_conducts` (códigos LEDI), `consultation_exam_requests` (`sigtap_code`, `sigtap_competence`, `cid10_justification`, `status` `requested`), encaminhamento reaproveita os campos do desfecho do atendimento.
- `Consultations::Start.call(attendance:, by:)`: interruptor utilizável; atendimento `in_care` chamado por `by`; CBO permitido (nível superior, sem 2232); par validado (`409 citizen_not_verified`); paciente resolvido.
- `Consultations::SaveDraft` (autosave; só o autor; só `draft`).
- `Consultations::Finalize.call(consultation:, outcome_params:, by:)`: exige ≥1 problema avaliado, ≥1 conduta, A ou P não vazio, nome do paciente presente, CID-10 permitido ao CBO (`cid10_not_allowed_for_cbo`); numa transação: aplica eventos de problemas, `status = finalized`, fecha o atendimento com o desfecho (comando existente; retorno/encaminhamento geram pedido), enfileira a ficha.
- Trigger: consulta `finalized` imutável.
- `consultation_addenda` (só acréscimos): `consultation_id`, `author_user_id`, `text` (cifrado), `reason` (≥ 10), `changes` (problemas/condutas/exames), `opening_id` quando o autor não é o da consulta; `Consultations::AddAddendum` aplica os eventos e dispara a regeneração da ficha (§6).
- Tipo de atendimento sugerido: horário marcado → consulta agendada; escuta `same_day` → consulta no dia; editável.
- Impresso: `GET /attendance/consultations/:id/print` gera PDF na hora (não gravado) com a consulta finalizada, adendos em ordem, unidade, profissional (nome, conselho, CBO), paciente (nome de exibição, CPF, nascimento) e espaço para assinatura e carimbo; exige nome do paciente.

## 5. Leitura e LGPD

- `ClinicalRecord::Access.call(user:, patient:, attendance: nil)` → `:in_context` (atendimento `in_care` chamado pelo usuário, ou `waiting` na unidade de um vínculo ativo dele, com CBO permitido), `:justified` (abertura válida) ou `:denied`.
- Abertura justificada: `POST /clinical_record/openings { cpf, reason_code, reason_note? }` com step-up; `reason_code` ∈ `case_review`, `active_search`, `continuity_of_care`, `other` (nota ≥ 10 quando `other`); válida 30 minutos para aquele paciente e usuário; `clinical_record_openings` (só acréscimos).
- Trilha: toda leitura publica `clinical_record.viewed` (`patient_id`, `user_id`, `access` `in_context`|`justified`, `reason_code`), nunca a nota.
- Relatório: `GET /clinical_record/openings?from=&to=&user_id=` (`municipal_admin`): quem, quando, CPF mascarado, motivo.
- Recepção: 403 em toda rota do prontuário; fila mostra só nome de exibição e cor.
- Texto clínico e nomes: cifrados; fora de evento, log, Analytics, mensagem de erro; sem busca por texto.
- ADR 0026: paciente com consulta é retido na exclusão; revogação não afeta o prontuário.

## 6. Ficha LEDI

- Tarefa 1 do plano: confirmar nos IDLs 8.7.0 e na documentação os códigos de tipo de atendimento, condutas, exames solicitados, problemas com situação, medições e a regra de CID-10 por CBO; gravar `config/ledi/consultation_mapping.yml` com fontes; parar se contradizer este desenho.
- `Ledi::Fichas::IndividualCare` (fonte: consulta finalizada + adendos): tipo de atendimento, problemas avaliados com situação, medições (da consulta ou, na falta, da escuta do mesmo atendimento), condutas, exames solicitados, encaminhamento, início/fim; identificação como no módulo 18 (só CPF do cidadão; nascimento e sexo do paciente). Sem medicamentos nem texto SOAP.
- Nasce na finalização (exportação utilizável); falta de identificação → "não gerada" do módulo 18.
- Adendo com mudança estruturada: ficha `pending`/`rejected` → regerada; ficha `accepted` → `ledi_outbox` ganha linha `correction_pending` (não enviada) até a regra de reenvio após aceite ser confirmada (api#41); a Produção mostra.

## 7. Telas (dashboard)

- Atendimento/consulta: painel do paciente (nome de exibição, idade, sexo, problemas ativos, escuta do dia, últimas consultas) e editor (S, O, A, P; sinais; problemas com avaliar/incluir/resolver/corrigir início; busca CIAP-2/CID-10 por nome — ampliar a busca do módulo 18 para CID-10; condutas; exames com busca SIGTAP; encaminhamento; tipo de atendimento; desfecho); autosave; Finalizar.
- Consulta finalizada: leitura, Adendo, Imprimir.
- Prontuário fora de contexto: busca por CPF + motivo + step-up; leitura marcada "abertura justificada".
- Relatório "Aberturas fora de contexto" (`municipal_admin`).
- Validação presencial: nome completo, nome social, nome da mãe.
- Produção e-SUS: `correction_pending`.

## 8. Testes

- Tarefa 1 (mapeamento com fonte; parada).
- Paciente: nasce na primeira consulta; sem duplicar CPF (threads); par declarado nunca ligado; divergência registrada.
- Lista: eventos reconstroem estado; sem dois ativos; só por consulta/adendo.
- Consulta: interruptor/modo; não validado; rascunho só do autor; finalização (requisitos, transação com desfecho e pedido); imutável; adendo do autor e de terceiro com abertura.
- Leitura: contexto; abertura (motivo, step-up, 30 min); trilha; recepção 403; relatório.
- Ficha: IDLs; CID-10 por CBO; duas fichas no mesmo atendimento; regeneração antes do aceite; `correction_pending` depois.
- Cifra e varredura de log/evento.
- Invariantes (`spec/invariants/clinical_record_invariants_spec.rb`): os do ADR 0031.
- Front (relógio fixo). Prova no navegador conforme a parte 5 do brainstorm.

## 9. Semente de dev

Interruptor `clinical_record` e `record_mode = record` em Curitiba (dev); cidadã validada com nome completo e social; médica e enfermeira da semente; uma consulta finalizada com diabetes (CIAP-2 T90) ativo e um adendo.

## 10. Rollout

1. `api`: migração de cidade; interruptor no catálogo; `VALID_RANGE` → `(1..31)`.
2. `dashboard` depois do `api`.
3. Tudo desligado; modo `record` e interruptor `clinical_record` por cidade, pelo maintenance.

## 11. F-IDs (propostos)

| ID | Funcionalidade | Superfície |
|---|---|---|
| F-19.1 | Paciente por CPF ligado aos pares validados, e nome no cadastro | api, dashboard |
| F-19.2 | Lista de problemas do paciente por eventos, alimentada pela consulta | api, dashboard |
| F-19.3 | Consulta SOAP: rascunho, itens estruturados, finalização com desfecho | api, dashboard |
| F-19.4 | Adendos e impresso para assinatura manual | api, dashboard |
| F-19.5 | Leitura em contexto, abertura justificada com step-up e relatório | api, dashboard |
| F-19.6 | Ficha LEDI de atendimento individual, com confirmação do layout e correção pendente após aceite | api, dashboard |
| F-19.7 | Interruptor `clinical_record` e pré-condições | api, maintenance |

## 12. Riscos

- Prontuário sem NGS2 exige papel assinado até o 19b.
- Aceite e reenvio no PEC não observados (api#41).
- Nome só para pares validados a partir do deploy; pares já validados precisam completar.
- Conflito de arquivos com o módulo 18 em execução (atendimento, validação presencial, `ledi_outbox`, `city_schema.rb`, `adr_pointers_spec.rb`).
