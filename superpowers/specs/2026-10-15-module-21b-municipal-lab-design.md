# Módulo 21 — Exames e laboratório, subprojeto 21b (laboratório municipal: fluxo analítico) — design

**Data:** 2026-10-15
**Status:** aprovado (2026-10-15)
**Afeta:**
- `apps/api`: plataforma — catálogo-base (`lab_exam_templates`, `lab_analyte_templates`); cidade — `lab_exams`, `lab_exam_versions`, `lab_requisitions`, `lab_requisition_items`, `lab_samples`, `lab_shipments`, `lab_results`, `lab_reports`, `lab_critical_notices`, `lab_rejection_reasons`, `lab_middleware_tokens`; `exam_providers.uses_municipal_lab`; `units.collects_samples`; dados do liberador no perfil profissional; `exam_orders.gal_requisition`; papéis `lab_staff` e `lab_releaser`; interruptores `municipal_lab`, `lab_hl7_inbound`, `fake_middleware`; entrada HL7; assinatura avançada; serviço `fake-middleware`.
- `apps/dashboard`: grupo "Laboratório" (requisição, etiquetas, coleta, remessa, recebimento, mapa, digitação, liberação, críticos, retificação, entrega, situação, admin do laboratório, relatórios); requisição e remessa na UBS que coleta.
- `apps/maintenance`: interruptores (mecanismo genérico) e catálogo-base.
- `contracts`: perfil HL7 do laboratório e `lab-report-v1` — tag `lab-v1.0.0`.
- compose de dev: serviço `fake-middleware`.

**ADR:** `docs/adr/0037.md` · **Módulo:** `docs/modulos/21--exames.md` · **Pesquisa:** `docs/pesquisa/2026-10-15-laboratorio-municipal-21b.md`, `docs/pesquisa/2026-10-15-laboratorio-analisadores.md`

**Fora desta entrega:** controle de qualidade interno e estoque de reagentes (subprojeto 21c, brainstorm próprio); driver direto de aparelho (ASTM, serial, agente local); SMS de resultado; envio REL/RELC à RNDS; integração com o GAL (só registro); laboratório de apoio; coleta domiciliar.

## 1. Ponto de partida

- 21a (ADR 0036): `exam_order` com ciclo de vida e trilha; prestadores (`municipal` entre eles); portal do prestador; entrada FHIR; prazos; ciência e fila da equipe; política de liberação ao cidadão; resultado externo; linha do tempo; retificação ausente (api#85); exame conta na exclusão LGPD (api#84).
- 19b (ADR 0032): assinatura ICP em nuvem (PAdES); 19c (ADR 0033): página de conferência com QR.
- Pesquisa de 2026-10-15:
  - **RDC Anvisa 978/2025** obriga o sistema a:
    - identificação da amostra (4 campos), quem coletou e quem recebeu, aceite/rejeição/restrição com motivo;
    - laudo com os 14 itens do art. 167 (CNES, RT e liberador com conselho, método, unidade, referência, restrição, data de coleta e emissão, assinatura);
    - retificação só por novo laudo com registro de quem foi avisado (art. 170);
    - valores críticos definidos e comunicados;
    - trilha que preserva o original; guarda de 5 anos;
    - assinatura física ou eletrônica avançada/qualificada (ICP não exigida; CFM exige ICP do médico).
  - **Editais** (9 lidos): etiqueta com código de barras, mapa de trabalho, digitação com referência, liberação restrita e laudo na internet em 7–8; interfaceamento com aparelhos em 6; GAL, controle de qualidade, estoque e SMS em 2 ou menos. A prova de conceito cobra 90–100% dos itens.
  - **Aparelhos:** comodato com reagente; o fornecedor entrega middleware e interface; ASTM ou HL7 v2; o LIS fala com o middleware; risco de troca de paciente na interface.

## 2. Decisões desta conversa

| # | Decisão |
|---|---|
| 1 | Objetivo: passar na prova de conceito dos editais, sem driver de aparelho. |
| 2 | Coleta na UBS com remessa **e** no posto do laboratório; a cidade marca por unidade. |
| 3 | Catálogo-base da plataforma; o laboratório adota e ajusta; cada ajuste vira versão imutável que o laudo guarda. |
| 4 | Entrada de resultados de aparelhos por HL7 v2.5.1 sobre HTTPS, para o middleware do fornecedor; middleware simulado na prova. |
| 5 | Dois papéis: `lab_staff` e `lab_releaser` (com RT e substituto marcados). |
| 6 | O 21a é a fonte única do exame; a liberação cria o resultado no 21a; avulso vira `exam_order` com origem `lab_reception`. |
| 7 | Assinatura: ICP (19b) quando houver; senão assinatura eletrônica avançada do Rota Saúde; médico só com ICP. |
| 8 | GAL só como registro; controle de qualidade e estoque no 21c; SMS fora; resultado ao cidadão pelo wpda pela política do 21a. |
| 9 | Abordagem 1: 21b (fluxo analítico) agora, 21c (qualidade e insumos) depois. |

## 3. Interruptores

- `municipal_lab` (`requires: ["feature:exam_tracking"]`): liga o 21b.
- `lab_hl7_inbound` (`requires: ["feature:municipal_lab"]`): liga `POST /hl7/v2`.
- `fake_middleware` (`requires: ["feature:lab_hl7_inbound"]`, `environments: SIMULATION_ENVS` — dev e staging): o middleware simulado responde pelas amostras recebidas.

## 4. Dados

**Plataforma (catálogo-base)**
- `lab_exam_templates`: `sigtap_code`, `name`, `sector` (bioquímica, hematologia, imunologia, urinálise, parasitologia, microbiologia…), `material`, `container`, `suggested_method`, `preparation`, `active`.
- `lab_analyte_templates`: `lab_exam_template_id`, `key`, `name`, `loinc`, `unit`, `value_type` (`numeric` | `text` | `choice`), `choices`, `decimals`, `reference_ranges` (lista de `{ sex: male|female|any, age_min_days, age_max_days, low, high, text }`), `position`.
- Semente: exames da APS — hemograma, glicemia, HbA1c, colesterol total e frações, triglicerídeos, creatinina, ureia, TGO, TGP, TSH, T4 livre, urina tipo 1, parasitológico, beta-hCG — com referência de partida marcada "ajuste ao seu método".

**Cidade**
- `lab_exams`: `exam_provider_id` (o laboratório), `lab_exam_template_id`, `sigtap_code`, `sector`, `bench`, `delivery_days`, `active`; `current_version_id`.
- `lab_exam_versions` (imutável; trigger impede UPDATE e DELETE): `lab_exam_id`, `version`, `method`, `material`, `container`, `analytes` (lista com `key`, `name`, `loinc`, `unit`, `value_type`, `decimals`, `reference_ranges`, `entry_min`, `entry_max`, `critical_low`, `critical_high`, `formula` para calculados), `instrument_test_code` (para a HL7), `created_by`, `created_at`.
- `lab_rejection_reasons`: do laboratório (`hemolisada`, `sem identificação`, `volume insuficiente`…), `active`.
- `lab_requisitions`: `number` (único na cidade), `patient_id`, `exam_provider_id`, `origin_unit_id`, `collection_place` (`ubs` | `lab`), `biological_sex`, `mother_name`, `urgent`, `clinical_info`, `expected_delivery_on`, `status` (`open` | `in_analysis` | `typed` | `released` | `delivered`), `delivered_to`, `delivered_at`.
- `lab_requisition_items`: `lab_requisition_id`, `exam_order_id` (único), `lab_exam_version_id`, `lab_sample_id`, `status` (`pending` | `typed` | `released` | `cancelled`).
- `lab_samples`: `lab_requisition_id`, `barcode` (Code 128, único na cidade), `material`, `container`, `collected_by`, `collected_at`, `lab_shipment_id`, `received_by`, `received_at`, `status` (`awaiting` | `in_transit` | `received` | `rejected` | `accepted_with_restriction` | `not_received`), `lab_rejection_reason_id`, `restriction_note`, `recollection_of_id`.
- `lab_shipments`: `origin_unit_id`, `exam_provider_id`, `sent_by`, `sent_at`, `temperature_celsius`, `received_by`, `received_at`, `status` (`open` | `sent` | `received`).
- `lab_results` (só acréscimo; trigger): `lab_requisition_item_id`, `analyte_key`, `value` (cifrado), `unit`, `flag` (`normal` | `low` | `high` | `critical_low` | `critical_high`), `source` (`typed` | `hl7`), `hl7_message_id`, `raw` (cifrado, só HL7), `entered_by`, `entered_at`, `chosen` (o valor que vale entre repetições; mudar a escolha cria linha nova).
- `lab_reports`: `lab_requisition_id`, `version`, `replaces_report_id`, `partial`, `released_by`, `released_at`, `signature_kind` (`icp` | `advanced`), `signature_request_id` (ICP), `pdf` (cifrado), `pdf_sha256`, `verification_token`, `short_code`, `rectification_reason`, `rectification_contacts` (quem foi avisado, quando, como), `status` (`pending_signature` | `released` | `rectified`).
- `lab_critical_notices`: `lab_result_id`, `notified_person`, `notified_role`, `channel` (`telefone` | `presencial` | `sistema`), `notified_at`, `recorded_by`.
- `lab_middleware_tokens`: `exam_provider_id`, `token_digest`, `revoked_at`.
- `exam_providers.uses_municipal_lab` (bool); `units.collects_samples` (bool); `exam_orders.gal_requisition` (texto opcional).
- Perfil profissional: `council` (`CRF` | `CRBM` | `CRM`), `council_uf`, `council_number`; marcas `lab_technical_lead` e `lab_technical_deputy` no laboratório.

## 5. Fluxo

1. **Requisição.** UBS que coleta (enfermagem) ou recepção do laboratório (`lab_staff`) abre a requisição com os `exam_orders` do paciente para o laboratório (autorizados ou sem autorização exigida). Requisição em papel cria `exam_order` com origem `lab_reception` (solicitante com vínculo ou digitado, como a guia do 21a). Gera comprovante com data prevista (maior `delivery_days` dos exames) e as etiquetas: uma por material/recipiente, Code 128 com nome, nascimento, material e número da requisição; reimpressão com trilha.
2. **Coleta.** Leitura da etiqueta registra quem e quando; no 21a o exame vai a `coletado`.
3. **Remessa** (UBS que coleta). A UBS monta a remessa pelas etiquetas, informa a temperatura e envia. O laboratório confere lendo as etiquetas; amostra da lista que não chegou fica `not_received` e alerta a UBS.
4. **Recebimento.** Cada amostra: aceita, rejeitada (motivo do cadastro) ou aceita com restrição (nota, que vai ao laudo). Rejeitada gera **recoleta** para a UBS de origem (nova amostra ligada à anterior); o exame segue aberto e o prazo de coleta recomeça.
5. **Mapa de trabalho** por setor e bancada, filtros de urgência, data e pendentes; PDF.
6. **Resultado.** Digitado pelo mapa ou pela etiqueta: destaque fora da referência (pela faixa do sexo e da idade na data da coleta), bloqueio fora de `entry_min`/`entry_max`, calculados preenchidos (`formula`). Pela HL7, as mesmas regras, chega "aguardando conferência". Repetição cria linha nova; o liberador escolhe a que vale.
7. **Liberação** (`lab_releaser`). Requisição inteira ou por exame (laudo parcial marcado). Valor crítico exige o registro da comunicação antes de liberar, ou entra na fila "críticos sem aviso" (liberação permitida, aviso obrigatório depois, com alerta). Assinatura: ICP pelo 19b (laudo fica `pending_signature` até assinar) ou avançada (TOTP na hora + SHA-256 do PDF + carimbo de data + trilha); médico (CRM) só ICP. PDF com os 14 itens do art. 167, CNES do laboratório, restrições, laboratório de apoio quando houver, tipo de assinatura e QR de conferência (página do 19c).
8. **Volta ao 21a.** A liberação cria `exam_result` com `source: lab` (PDF, valores com LOINC, `critical`) para cada `exam_order` liberado; o exame vai a `resulted`; ciência, fila da equipe, linha do tempo e wpda seguem o 21a.
9. **Retificação** (parte do laboratório municipal de api#85). Novo laudo com `replaces_report_id`, motivo e contatos registrados; o anterior fica `rectified`. No 21a, o `exam_result` é substituído com trilha (o anterior fica "substituído", visível só ao profissional) e o exame volta à ciência; o cidadão vê só o vigente.
10. **Entrega** do impresso: quem retirou e quando.
11. **Situação** por número de requisição, para UBS e laboratório: recebida → em análise → digitada → liberada → entregue.

**GAL.** `exam_order.gal_requisition` registra o número; o laudo do LACEN entra como resultado externo do 21a.

## 6. Entrada HL7 (`lab_hl7_inbound`)

- `POST /hl7/v2` no host da cidade, `Content-Type: application/hl7-v2`, `Authorization: Bearer <token do middleware>`; corpo = uma mensagem HL7 v2.5.1 (ER7). Resposta sempre `ACK` (`MSA-1` `AA` | `AE` | `AR`), com `ERR` sem dado de paciente.
- **Lista de trabalho:** `QBP^Q11` pelo código de barras → `RSP^K11` com os exames da amostra (`instrument_test_code` da versão). Alternativa: `OML^O33` por lote, quando o middleware preferir receber.
- **Resultado:** `OUL^R22` — `SPM-2` = código de barras; `OBX-3` = `instrument_test_code` + `analyte_key`. Recusas (`AE`): código de barras desconhecido, amostra não recebida, exame fora da versão, componente desconhecido, valor fora do limite de digitação. `MSH-10` repetido do mesmo token → `AA` sem efeito (idempotência). Mensagem bruta guardada cifrada. Limite de taxa por token; token revogado → `AR`.
- `fake-middleware` (compose de dev): lê amostras recebidas, consulta a lista de trabalho e envia resultados de exemplo, com um crítico e uma repetição.

## 7. Telas e relatórios

- `lab_staff`: requisição, etiquetas, coleta, conferência de remessa, recebimento e rejeição, mapa, digitação, entrega, situação.
- `lab_releaser`: fila de liberação, conferência, assinatura, críticos sem aviso, retificação.
- UBS que coleta: requisição, etiquetas, coleta, remessa, recoletas.
- Admin (`municipal_admin`): adotar e ajustar exames (versões), motivos de rejeição, liberadores, RT e substituto, token do middleware.
- Maintenance: interruptores e catálogo-base.
- Relatórios (admin e `lab_releaser`): produção por unidade de origem, exame, setor e liberador por mês, com quantidade e valor SIGTAP (tela + CSV); rejeições por motivo e unidade; prazo de entrega cumprido/atrasado.

## 8. Segurança e LGPD

- Leitura clínica pelas regras do 19a/21a, com trilha; `lab_staff` vê dados da amostra e da requisição, não o prontuário.
- Resultado não liberado não sai do laboratório (21a, wpda e linha do tempo só veem o liberado).
- Valores, CPF, PDF, mensagem HL7 e tokens fora de log, evento, Analytics e erro; mensagem bruta cifrada; token só como hash.
- Requisição, amostras, resultados e laudos contam como documento clínico na exclusão LGPD (regra da api#84).
- Eventos só com ids: `lab_requisition.opened`, `lab_sample.collected`, `.received`, `.rejected`, `lab_shipment.sent`, `.received`, `lab_result.entered`, `lab_report.released`, `.rectified`, `lab_critical.notified`.

## 9. Testes

- **api:** versões do catálogo e versão no laudo; etiqueta (4 campos, código único); remessa (faltante, conferência); rejeição e recoleta; limites e calculados; repetição sem sobrescrever; liberação parcial e total; crítico exigindo comunicação; médico só ICP; assinatura avançada (TOTP, hash); 14 itens do laudo; liberação criando `exam_result` sem duplicar; retificação no 21b e no 21a com volta à ciência; não liberado invisível fora do laboratório; HL7 (aceite, cada `AE`, `MSH-10` repetido, token revogado, bruto cifrado); corrida na liberação; exclusão LGPD; guarda R18.
- **dashboard:** Vitest.
- **Prova no navegador** (`fake-middleware`, PSC simulado): requisição na UBS → etiqueta → coleta e remessa → recebimento com uma rejeição e a recoleta → mapa → digitação e resultado HL7 → crítico e comunicação → liberação com ICP simulado e com avançada → laudo no 21a e no wpda conforme a política → retificação → produção.

## 10. Entrega

0. Provas técnicas: (a) perfil das mensagens `QBP^Q11`/`RSP^K11`/`OUL^R22`/`ACK` conferido contra a HL7 v2.5.1; (b) etiqueta Code 128 gerada em PDF imprimível em impressora térmica (tamanho de etiqueta de tubo) e lida por leitor de código de barras, sem agente local.
1. `contracts`: perfil HL7 do laboratório e `lab-report-v1` — tag `lab-v1.0.0`.
2. `api` (migrações acima das do 21a; `fake-middleware` no compose).
3. `dashboard`. 4. `maintenance`. 5. catálogo público. O wpda não muda.

## 11. F-IDs (propostos)

| F-ID | Funcionalidade | Apps |
|---|---|---|
| F-21.14 | Interruptores e catálogo-base | api, maintenance |
| F-21.15 | Catálogo do laboratório com versões | api, dashboard |
| F-21.16 | Requisição, etiqueta e coleta | api, dashboard |
| F-21.17 | Remessa da UBS e recebimento com rejeição | api, dashboard |
| F-21.18 | Mapa de trabalho e digitação | api, dashboard |
| F-21.19 | Entrada HL7 e middleware simulado | api, contracts |
| F-21.20 | Liberação com assinatura e laudo | api, dashboard |
| F-21.21 | Valores críticos | api, dashboard |
| F-21.22 | Retificação de laudo do laboratório municipal (parte de api#85) | api, dashboard |
| F-21.23 | Relatórios de produção, rejeição e prazo | api, dashboard |
| F-21.24 | Registro do GAL | api, dashboard |

## 12. Riscos e pendências de go-live

- Primeiro middleware real ligado à HL7 (fornecedor do reagente; mapeamento de códigos; risco de troca de paciente).
- Validação do sistema pelo laboratório (dossiê do art. 106 da RDC 978).
- Certificados ICP dos liberadores médicos; aceitação da assinatura avançada pela VISA local.
- REL/RELC à RNDS quando o Rota Saúde for o LIS do laboratório (fora do módulo).
- Impressoras térmicas e leitores nas UBS e no laboratório.
- Retificação de laudo de prestador externo (portal/FHIR do 21a) continua em api#85.
