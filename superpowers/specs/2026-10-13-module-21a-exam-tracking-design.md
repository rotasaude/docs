# Módulo 21 — Exames e laboratório, subprojeto 21a (rastreamento do exame) — design

**Data:** 2026-10-13
**Status:** aprovado (2026-10-13)
**Afeta:**
- `apps/api`: cidade — `exam_providers`, `exam_catalog_items`, `exam_provider_items`, `exam_orders`, `exam_order_events`, `exam_results`; prioridade no pedido do 19a; papéis `regulador` e `prestador`; interruptores `exam_tracking`, `exam_fhir_inbound`, `exam_pec_bridge`, `fake_lab`; entrada FHIR; job diário de prazos; fornecedor `Timeline::Sources::Exams`.
- `apps/dashboard`: catálogo e prestadores, fila de regulação, painel de atrasos, ciência e fila da equipe, guia da recepção, resultado externo, portal do prestador, relatório de conferência, tela do botão no PEC.
- `apps/wpda`: "Meus exames".
- `apps/maintenance`: interruptores (mecanismo genérico).
- `contracts`: `exam-order-v1` e perfil FHIR de entrada — tag `exams-v1.0.0`.
- compose de dev: serviço `fake-lab`.

**ADR:** `docs/adr/0036.md` · **Módulo:** `docs/modulos/21--exames.md` · **Pesquisa:** `docs/pesquisa/2026-10-05-integracoes/frente-3-exames.md`, `docs/pesquisa/2026-10-05-mapa-de-integracoes.md`

**Fora desta entrega:** laboratório municipal (coleta com etiqueta, mapa de trabalho, digitação e liberação de laudo — subprojeto 21b, brainstorm próprio); regulação completa (módulo 26); arquivo BPA-I; HL7 v2; envio REL/RELC à RNDS; GAL; DICOM; aviso por SMS ou e-mail.

## 1. Ponto de partida

- 19a (ADR 0031): pedido de exame estruturado na consulta (`consultation_exam_requests`: SIGTAP com competência, CID opcional, cancelamento por adendo); sem prioridade e sem ciclo de vida.
- 19c (ADR 0033): requisição de exames em PDF (F-19.22).
- 20 (ADR 0035): linha do tempo montada na leitura por fornecedores; ponte com a tela "sistemas externos" do PEC (HMAC, sessão curta, só leitura); pendência api#71 (fornecedor de exames).
- Pesquisa de 2026-10-05: sem padrão nacional de pedido; quatro arranjos (laboratório municipal, credenciado, consórcio, LACEN/GAL); resultado volta em PDF ou papel; REL na RNDS só cobre COVID/Mpox e não há leitura por integrador; GAL sem API; LIS só por contrato. SIGTAP é o código do pedido; LOINC o do resultado. Recomendação: v1 sem integração + portal do prestador; v2 FHIR por laboratório; v3 RNDS.

## 2. Decisões desta conversa

| # | Decisão |
|---|---|
| 1 | Módulo 21 em dois subprojetos: 21a rastreamento, 21b laboratório municipal. |
| 2 | Regulação simples no 21a: exame marcado "exige autorização" entra numa fila decidida pelo regulador da cidade; o 26 amplia. |
| 3 | Resultado entra pela unidade, pelo portal do prestador e por uma entrada FHIR (LOINC), provada com laboratório simulado. |
| 4 | Liberação ao cidadão por política de cada exame no catálogo da cidade (`imediata`, `apos_ciencia`, `nunca_wpda`); padrão `apos_ciencia`. |
| 5 | Prazos por exame e por prestador; ordem prestador → catálogo → padrão da plataforma; urgente usa metade. |
| 6 | Ciência por quem pediu; após o prazo, ou se crítico, fila da equipe de referência. |
| 7 | Cidades em modo integrado: recepção cadastra pela guia + botão no PEC (ponte do módulo 20), sujeito a prova técnica. |
| 8 | Relatório de conferência por prestador e mês (tela + CSV); sem BPA-I. |
| 9 | Portal do prestador dentro do dashboard, papel `prestador`, um usuário por cidade. |
| 10 | Resultado sem pedido entra como "resultado externo". |
| 11 | Abordagem 1: exame com ciclo de vida em tabelas da cidade, trilha só de acréscimo, FHIR só na borda. |
| 12 | Revisão dos planos (2026-10-14): `exam_tracking` nos dois modos de prontuário; equipe de referência = equipes ativas da unidade do exame; botão no PEC dentro da página da linha do tempo; `fake_lab` em dev e staging. |

## 3. Interruptores

- `exam_tracking` (`requires: ["record_mode"]` — modo `record` **ou** `integrated`): liga o 21a. No modo `record` os pedidos também nascem da consulta; no `integrated`, pela guia da recepção e pelo botão no PEC. Desligado: o pedido do 19a segue como hoje (só requisição).
- `exam_fhir_inbound` (`requires: ["feature:exam_tracking"]`): liga `POST /fhir/DiagnosticReport`.
- `exam_pec_bridge` (`requires: ["feature:exam_tracking", "feature:pec_external_view"]`): liga "Solicitar exames" pela ponte do PEC.
- `fake_lab` (`requires: ["feature:exam_fhir_inbound"]`, `environments: SIMULATION_ENVS` — dev e staging, como o PSC e o SNCR simulados; em produção não aparece no catálogo e o api recusa ligar): o laboratório simulado responde às guias agendadas dele.

## 4. Dados (banco da cidade)

**Catálogo e prestadores**
- `exam_providers`: `name`, `cnes` (opcional), `kind` (`municipal` | `credenciado` | `consorcio` | `lacen` | `imagem`), `active`; `fhir_token_digest` (SHA-256), `fhir_token_revoked_at`.
- `exam_catalog_items`: `sigtap_code` (único na cidade), `requires_authorization`, `release_policy` (`imediata` | `apos_ciencia` | `nunca_wpda`, padrão `apos_ciencia`), prazos em dias por etapa (`authorize_days`, `schedule_days`, `collect_days`, `result_days`, `acknowledge_days`, todos opcionais), `result_fields` (lista de `{ key, label, unit, loinc }`), `active`.
- `exam_provider_items`: (`exam_provider_id`, `exam_catalog_item_id`) único; prazos por etapa opcionais que se sobrepõem aos do catálogo.

**O exame**
- `consultation_exam_requests` ganha `priority` (`rotina` | `urgente`, padrão `rotina`).
- `exam_orders`: `patient_id`, `sigtap_code`, `sigtap_competence`, `priority`, `cid10_justification`, `requester_user_id` (nulo quando o solicitante foi digitado), `requester_name` e `requester_cns` (quando digitado), `unit_id`, `origin` (`consulta` | `guia_recepcao` | `pec`), `consultation_exam_request_id` (único, quando `consulta`), `exam_provider_id`, `scheduled_at`, `scheduled_place`, `external_reference`, `guide_attachment` (cifrado, quando `guia_recepcao`), `short_code` (10 caracteres, sem ambíguos, índice único), `status` (cache da trilha), `status_changed_at`, `escalated_at`.
- `exam_order_events` (só acréscimo; trigger impede UPDATE e DELETE): `exam_order_id`, `kind` (`solicitado` | `autorizado` | `negado` | `agendado` | `coletado` | `resultado_registrado` | `ciencia` | `escalado_equipe` | `cancelado`), `actor_user_id` ou `actor_provider_id`, `reason` (obrigatório em `negado`, `cancelado`, `escalado_equipe`, `ciencia` pela equipe), `occurred_at`.
- `exam_results`: `exam_order_id` (nulo quando externo), `patient_id`, `sigtap_code`, `origin` (`pedido` | `externo`), `source` (`unidade` | `prestador` | `fhir` | `consulta`), `pdf` (cifrado com a chave da cidade), `values` (JSON cifrado: chave do campo do catálogo ou LOINC desconhecido → valor, unidade), `released_on`, `laboratory_name`, `critical`, `fhir_identifier` (único por prestador, idempotência), `registered_by_user_id` ou `registered_by_provider_id`.

**Ligação com o 19a.** Finalizar a consulta cria um `exam_order` (origem `consulta`, evento `solicitado`) para cada pedido `requested`, na mesma transação. Cancelar o pedido por adendo cancela o exame (evento `cancelado`), se ainda não houver resultado.

## 5. Fluxo, prazos e alertas

**Etapas**
1. `solicitado` — consulta, guia da recepção ou PEC.
2. Autorização — só para `requires_authorization`. Fila de regulação da cidade (papel `regulador` ou admin municipal), urgente primeiro e depois o mais antigo. `autorizado` exige o prestador; `negado` exige motivo, avisa quem pediu (contador no início do dashboard) e fecha o exame.
3. `agendado` — unidade ou prestador informam data e local. Exame sem autorização chega aqui direto; o prestador é escolhido pela unidade.
4. `coletado` — prestador ou unidade.
5. `resultado_registrado` — unidade, portal do prestador ou FHIR.
6. `ciencia` — quem pediu. Se o prazo de ciência estourar, ou o resultado vier `critical`, evento `escalado_equipe` e o exame entra na fila da equipe de referência — as equipes ativas da unidade do exame (o código não liga paciente a equipe); qualquer médico ou enfermeiro dessas equipes dá ciência com motivo. Solicitante digitado sem vínculo: escalado já no resultado.

**Prazos.** Cada etapa tem prazo resolvido nesta ordem: `exam_provider_items` → `exam_catalog_items` → padrão da plataforma (autorizar 15, agendar 15, coletar 30, resultado 30, ciência 15 dias). `urgente` usa a metade, arredondada para cima. Contagem em dias corridos desde o evento anterior.

**Alertas.** Job diário (`EachCityJob`) calcula o atraso por etapa e escala ciências vencidas. Mostra:
- painel "Exames atrasados" da unidade e da coordenação, com filtros por etapa, prestador e exame;
- para quem pediu: "resultados aguardando ciência" no início do dashboard;
- para o prestador: "coletas e resultados em atraso" no portal.
Sem SMS ou e-mail.

**Cancelamento.** Em qualquer etapa antes do resultado, por quem pediu ou pela unidade, com motivo. Depois do resultado, não.

## 6. Prestador, entradas e cidadão

**Portal do prestador (dashboard, papel `prestador`).** O admin municipal cria o usuário (e-mail, senha, TOTP) ligado a um `exam_provider`. Vê as guias do seu prestador (autorizadas/agendadas, coletadas, aguardando resultado, em atraso) com nome, data de nascimento e os 3 últimos dígitos do CPF. Agenda, registra coleta, registra resultado (PDF + campos do catálogo, marca "crítico"). Não acessa prontuário, linha do tempo nem guias de outro prestador; o isolamento é verificado no servidor.

**Entrada FHIR (`exam_fhir_inbound`).** `POST /fhir/DiagnosticReport` no host da cidade, `Authorization: Bearer <token do prestador>`.
- `basedOn` traz o `short_code` do `exam_order`; ele precisa existir, estar agendado ou coletado e pertencer ao prestador do token.
- O paciente do recurso (CPF ou CNS) precisa bater com o do exame; senão 422 `patient_mismatch`.
- `Observation` com LOINC preenche o campo do catálogo com o mesmo LOINC; LOINC desconhecido fica guardado sem campo.
- `presentedForm` (PDF) vira o PDF do resultado; sem PDF, 422 `missing_pdf`.
- `interpretation` crítica (`HH`, `LL`, `AA`) marca `critical`.
- Repetir o mesmo `identifier` não duplica (200 com o mesmo resultado).
- Limite de taxa por token; token revogado → 401.

**Laboratório simulado (`fake-lab`).** Serviço do compose de dev com o token de um prestador da semente; com `fake_lab` ligado, consulta as guias agendadas dele e envia um `DiagnosticReport` de exemplo (hemograma, HbA1c) para prova no navegador e CI.

**Botão no PEC (`exam_pec_bridge`).** Dentro da página da linha do tempo do PEC (módulo 20, aberta pela chamada assinada), um botão "Solicitar exames" abre um formulário que usa a sessão curta da página (cada cadastro no PEC tem chave própria e a cidade guarda uma só, por isso não há chamada assinada nova); profissional e paciente vêm da sessão; o pedido nasce com origem `pec`. Depende da prova técnica (§10).

**Guia da recepção.** A recepção cadastra paciente, exame, prioridade, solicitante (profissional com vínculo, ou nome e CNS digitados) e a foto ou PDF da guia; origem `guia_recepcao`.

**Resultado externo.** Profissional (na consulta) ou unidade registra exame SIGTAP, data, laboratório, PDF e valores; origem `externo`, ciência dada por quem registra; não aparece para prestador.

**Cidadão (wpda), "Meus exames".** Só par validado (regra do módulo 20). Situação: pedido, aguardando autorização, agendado (data e local), coletado, resultado disponível. Laudo por política: `imediata` mostra ao registrar; `apos_ciencia` mostra após a ciência; `nunca_wpda` mostra "retire o resultado na sua unidade". Nunca mostra motivo de negação nem a marca "crítico". Resultado externo segue a política do exame.

**Relatório de conferência (admin municipal e regulador).** Por prestador e mês: guias autorizadas, coletadas, com resultado no prazo e em atraso. Tela e CSV; o CSV detalhado leva só `short_code`, SIGTAP e datas.

**Linha do tempo (api#71).** `Timeline::Sources::Exams`: profissional vê pedido, etapas e resultado; cidadão vê andamento e laudo conforme a política.

## 7. Segurança e LGPD

- Leitura do exame pelas regras do 19a (contexto, autoria, administrativa, abertura), com trilha; o prestador só pelas guias dele.
- PDF, guia e valores cifrados com a chave da cidade; download por URL curta e assinada, com trilha.
- Eventos só com ids: `exam_order.requested`, `.authorized`, `.denied`, `.scheduled`, `.collected`, `.resulted`, `.acknowledged`, `.escalated`, `.cancelled`.
- Valor de resultado, CPF e nome do exame junto com o paciente fora de log, evento, Analytics e erro.
- Token FHIR guardado só como hash, revogável no admin; limite de taxa.
- Política de liberação aplicada no servidor.

## 8. Testes

- **api:** ciclo de vida e trigger da trilha; transições inválidas; prazos (ordem de resolução, urgente, job diário); escalonamento por atraso e por crítico; isolamento entre prestadores; contrato FHIR (aceite, `patient_mismatch`, `missing_pdf`, guia de outro prestador, idempotência, LOINC desconhecido, token revogado); pedido do 19a virando `exam_order` e cancelamento por adendo; política de liberação no wpda; relatório; fornecedor da linha do tempo; guarda R18; `fake_lab` recusado em produção.
- **dashboard e wpda:** Vitest.
- **Prova no navegador** (com `fake-lab`): pedido → autorização → agendamento pelo prestador → resultado via FHIR → ciência; atraso e escalonamento; cidadão nas três políticas; guia da recepção; resultado externo; relatório.

## 9. Telas

- Profissional: status dos exames na consulta e na aba do paciente; "resultados aguardando ciência"; fila da equipe; resultado externo.
- Recepção/unidade: guia da recepção; agendar, registrar coleta e resultado; painel de atrasos.
- Regulador: fila de regulação; relatório.
- Admin municipal: catálogo de exames, prestadores, usuários e tokens dos prestadores; relatório.
- Prestador: portal.
- Cidadão: "Meus exames".
- Maintenance: interruptores.

## 10. Entrega

0. Provas técnicas: (a) no PEC local, a tela de "sistemas externos" aceita formulário e envio dentro do atendimento — falhou, F-21.11 sai do escopo e a recepção segue cadastrando; (b) forma do `DiagnosticReport` de entrada (perfil mínimo).
1. `contracts`: `exam-order-v1` e perfil FHIR de entrada — tag `exams-v1.0.0`.
2. `api` (migrações acima das do módulo 20; `fake-lab` no compose).
3. `dashboard`. 4. `wpda`. 5. `maintenance`. 6. catálogo público.

## 11. F-IDs (propostos)

| F-ID | Funcionalidade | Apps |
|---|---|---|
| F-21.1 | Interruptores e configuração | api, maintenance |
| F-21.2 | Catálogo de exames e prestadores da cidade | api, dashboard |
| F-21.3 | Exame com ciclo de vida (consulta e guia da recepção) | api, dashboard |
| F-21.4 | Fila de regulação | api, dashboard |
| F-21.5 | Prazos e painel de atrasos | api, dashboard |
| F-21.6 | Portal do prestador | api, dashboard |
| F-21.7 | Entrada FHIR e laboratório simulado | api, contracts |
| F-21.8 | Resultado, ciência e fila da equipe | api, dashboard |
| F-21.9 | Resultado externo | api, dashboard |
| F-21.10 | Exames do cidadão no wpda | api, wpda |
| F-21.11 | Botão no PEC | api, dashboard |
| F-21.12 | Relatório de conferência | api, dashboard |
| F-21.13 | Fornecedor da linha do tempo (fecha api#71) | api |

## 12. Riscos e pendências de go-live

- Primeiro laboratório real na entrada FHIR (contrato, mapeamento SIGTAP↔LOINC, identificação do paciente).
- Botão no PEC real (versão do PEC da cidade; prova local não garante a de produção).
- Envio REL/RELC à RNDS — fora do módulo; quem é o integrador quando a UBS registra teste rápido.
- Arranjo de exames e regulação de cada cidade-piloto (SISREG, consórcio, planilha).
- Volume de PDFs por banco de cidade.
- Vinculação de laudo ao paciente errado — conferência por CPF/CNS e trilha.
