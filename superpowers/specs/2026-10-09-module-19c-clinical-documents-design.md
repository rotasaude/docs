# Módulo 19 — Consulta, subprojeto 19c (documentos clínicos) — design

**Data:** 2026-10-09
**Status:** aprovado (2026-10-09)
**Afeta:**
- `apps/api`: plataforma — `medication_catalog_releases`, `medication_catalog_items`, listas Anvisa (antimicrobianos RDC 471, controlados Portaria 344); cidade — `clinical_documents`, `prescription_items`, `patient_medications`, `patient_medication_events`, `city_medication_list`, `nursing_protocols` (+ versões e itens), `document_verification_lookups`, `cnpj` no perfil da cidade; interruptor `clinical_documents`; página pública de conferência.
- `apps/dashboard`: aba Documentos na consulta, lista de medicamentos em uso, declaração na recepção, documentos em "Minhas consultas", REMUME, protocolos de enfermagem, CNPJ da cidade.
- `apps/maintenance`: interruptor (mecanismo genérico), importação do CATMAT e das listas Anvisa, revisão de itens.
- `contracts`: esquema `rotasaude.clinical_document.v1`, tag `clinical-v1.1.0`.
- `apps/signer`, `apps/wpda`, `apps/admin`: nada.

**ADR:** `docs/adr/0033.md` · **Módulo:** `docs/modulos/19--consulta.md` · **Pesquisa:** `docs/pesquisa/2026-10-09-documentos-clinicos-19c.md`

**Fora desta entrega:** receita de controlado e antimicrobiano digital (19d, SNCR); dispensação (assistência farmacêutica); exames de alto custo e OCI (módulo 21 e regulação); documentos no wpda (módulo 20); assinatura gov.br; integração com o Atesta CFM; registro de prescrição na RNDS.

## 1. Ponto de partida

- 19a (ADR 0031): consulta imutável com adendos, lista de problemas por eventos, pedidos de exame estruturados (SIGTAP com competência), leitura em contexto/autora/administrativa/abertura, impresso com Prawn.
- 19b (ADR 0032): assinatura ICP em nuvem, `signature_requests` aberto a novos tipos, sessão/fila/lote/volta ao papel, PSC simulado fora de produção, pedido nasce na emissão.
- Pesquisa de 2026-10-09: o PEC e-SUS APS é o modelo (atestado, declaração, receita por CATMAT + registro manual, QR code e código); Lei 14.063 (qualificada para atestado médico e controlado), CFM 2.299 (ICP em documento médico); antimicrobiano digital depende do SNCR; sem registro central de dispensação da receita comum; REPM/REDFM na RNDS vem (OBM, motivo, CNES, conselho); COFEN 801/2026 (protocolo, CNPJ, Coren); atestado de afastamento só médico e dentista; declaração de presença por qualquer colaborador com norma interna; Atesta CFM suspenso por liminar.

## 2. Decisões desta conversa

| # | Decisão |
|---|---|
| 1 | Escopo: atestado, declaração, receita comum, requisição de exames **e** lista estruturada de medicamentos. |
| 2 | Documentos clínicos só da consulta (pela autora); declaração de comparecimento também pela recepção. |
| 3 | Lista de medicamentos em uso do paciente por eventos, como a de problemas. |
| 4 | Entrega impressa com QR code e página pública de conferência. |
| 5 | Sem dispensação no 19c (assistência farmacêutica). |
| 6 | Catálogo da plataforma a partir do CATMAT, REMUME por cidade, texto livre permitido, `obm_code` reservado. |
| 7 | Prescrição de enfermagem com cadastro de protocolos municipais (trava de itens). |
| 8 | Cancelamento pela autora, com motivo; a página mostra "cancelado". |
| 9 | Abordagem 1: cabeçalho comum `clinical_documents` + tabela própria para os itens da receita. |

## 3. Interruptor e bases

- `clinical_documents` no catálogo de `Platform::Features`, `requires: ["feature:clinical_record"]` → `missing: ["clinical_record_disabled"]`. Desligado: nada novo é emitido; documentos existentes seguem legíveis e a página de conferência segue respondendo. A declaração da recepção também depende dele.
- **Catálogo (plataforma):** `medication_catalog_releases` (fonte, data, contagens, status) e `medication_catalog_items` (`catmat_code`, `source_description`, `active_ingredient` (DCB), `strength`, `dosage_form`, `unit`, `antimicrobial` bool, `controlled` bool, `controlled_list`, `obm_code` null, `hidden` bool, `parse_status` `ok`|`review`). Importação do CATMAT classe 6505 pela API aberta do Compras.gov, acionada no maintenance; diferença entrou/mudou/saiu. Item que o analisador não entende fica `hidden` e vai à lista de revisão. Marcas de antimicrobiano (RDC 471) e controlado (Portaria 344) por listas Anvisa importadas junto. Controlado aparece na busca bloqueado ("receita de controle especial — 19d").
- **REMUME (cidade):** `city_medication_list` (item do catálogo, unidades opcionais), editada pelo `municipal_admin` com step-up. Na busca da receita, itens da REMUME primeiro, com "na rede".
- **Protocolos de enfermagem (cidade):** `nursing_protocols` (título, número, ano) → `nursing_protocol_versions` (vigência de/até, itens permitidos com dose máxima opcional); só `municipal_admin`, com step-up; editar cria versão; a receita guarda a versão.
- **Perfil da cidade:** `cnpj` (secretaria/fundo), obrigatório para receita de enfermagem (`city_cnpj_missing`).

## 4. Documentos

- `clinical_documents` (cidade, só acréscimos; trigger só permite `issued` → `cancelled` com motivo/data/usuário): `kind` (`sick_note` | `attendance_declaration` | `prescription` | `exam_requisition`), `patient_id` (nulo só na declaração de quem não é paciente), `citizen_id`, `consultation_id` (nulo na declaração da recepção), `attendance_id`, `author_user_id`, `cbo_code`, `issue_mode` (`digital` | `paper`), `status`, `cancel_reason`, `cancelled_at`, `cancelled_by`, `verification_token` (128 bits, índice único), `short_code` (10 caracteres, sem ambíguos), `replaces_document_id`, `issued_at`, `content` (JSON cifrado com a chave da cidade).
- **Modo de emissão** escolhido no ato e nunca convertido: `digital` se a autora tem certificado ativo e `digital_signature` utilizável (nasce pedido de assinatura `pending`, como no 19b); senão `paper`. Antimicrobiano força `paper` (2 vias). Volta ao papel do 19b passa o documento a `paper`.
- **Quem emite:**
  - Atestado: médico (2251–2253) ou dentista (2232). Afastamento (dias, início) ou acompanhante (nome, parentesco, motivo CLT art. 473 X/XI/XII). CID só com `cid_authorized: true` (trilha); acompanhante nunca com CID.
  - Declaração: qualquer profissional da consulta ou a recepção (papéis da recepção) pelo atendimento; data, chegada/saída ou período, unidade, acompanhante opcional; papel traz nome e matrícula do emissor.
  - Receita: médico; dentista; enfermeiro só com itens de protocolo vigente (sem texto livre). Receita de enfermagem leva protocolo + versão, CNPJ e Coren.
  - Requisição de exames: autora da consulta; renderiza os `consultation_exam_requests` (exames comuns).
- Emissão também depois de finalizada, pela autora (trilha; não muda a consulta).

## 5. Lista de medicamentos e receita

- `patient_medications` (estado atual: item do catálogo ou texto, posologia resumida, `continuous`, `status` `active`|`suspended`, início, `origin` `prescription`|`external`; único ativo por paciente+item) e `patient_medication_events` (só acréscimos: `added`, `suspended`, `reactivated`, `changed`, com consulta/adendo/documento, usuário, hora). `Patients::ApplyMedicationEvent` é o único caminho de escrita. Receita de uso contínuo inclui/atualiza; uso curto não entra.
- Na consulta: lista ao lado dos problemas, com suspender, reativar e "informar medicamento em uso de fora".
- `prescription_items`: posição, `medication_catalog_item_id` ou `free_text` (marcado), descrição impressa, quantidade + unidade, via, posologia (texto), duração (dias), `continuous`, `antimicrobial`, `nursing_protocol_version_item_id`, `reason_problem_id` opcional (motivo para a RNDS futura).
- Validações: controlado recusado (`controlled_not_allowed`); enfermeiro fora do protocolo vigente ou acima da dose máxima (`not_in_nursing_protocol`, `above_protocol_max_dose`); texto livre de enfermeiro (`free_text_not_allowed`); antimicrobiano → papel, 2 vias, validade de 10 dias no PDF.
- "Renovar receita": parte dos contínuos ativos e preenche uma receita nova para revisão, dentro de uma consulta.

## 6. PDF, assinatura, conferência e cancelamento

- PDF por tipo (Prawn): cabeçalho (unidade, CNES, cidade), corpo do tipo, rodapé com profissional (conselho, CBO), QR code (endereço de conferência) e código curto; digital → rodapé NGS2 (e aviso de simulada); papel → espaço para assinatura e carimbo. Antimicrobiano: "1ª via — farmácia" / "2ª via — paciente".
- Assinatura: tipos novos no `signature_requests` do 19b; JSON canônico `rotasaude.clinical_document.v1` (cabeçalho + conteúdo do tipo; receita com CATMAT, versão do catálogo e protocolo) no `contracts` (`clinical-v1.1.0`); PAdES sobre o PDF.
- **Página pública** `https://<host da cidade>/v/<token>` e formulário com código curto + ano de nascimento do paciente. Mostra tipo, data, profissional e conselho, unidade, iniciais do paciente e ano de nascimento, situação (válido / cancelado em / aguardando assinatura) e modo (assinado digitalmente, com download do PDF assinado; ou papel). Nunca CID, medicamentos, dias de afastamento ou CPF. Limite de tentativas por IP; trilha de consulta sem dado pessoal (`document_verification_lookups`).
- **Cancelamento:** só a autora, motivo ≥ 10, step-up; `cancelled`; pedido de assinatura pendente sai da fila; "cancelar e emitir outro" abre um novo preenchido com `replaces_document_id`. Aviso na tela: cancelar não desfaz medicamento já entregue.

## 7. Telas

- **Consulta:** aba Documentos (emitir os quatro; lista com modo, estado da assinatura, Imprimir, Cancelar, Cancelar e emitir outro); medicamentos em uso ao lado dos problemas.
- **Recepção:** "Emitir declaração de comparecimento" no atendimento.
- **Minhas consultas:** documentos da consulta (reimprimir, cancelar).
- **Admin municipal:** REMUME, protocolos de enfermagem, CNPJ da cidade.
- **Maintenance:** interruptor; importação do CATMAT e das listas Anvisa; revisão de itens.

## 8. Segurança e LGPD

- Conteúdo cifrado; eventos só com ids (`clinical_document.issued`, `clinical_document.cancelled`, `patient_medication.changed`); leitura pelas regras do 19a (`ClinicalRecord::Access`) com trilha; página pública sem dado clínico; CID só com autorização registrada.
- Exclusão LGPD (ADR 0026): paciente com documento é registro retido.

## 9. Testes

- **api:** matriz quem-emite-o-quê; trava do protocolo; controlado recusado; antimicrobiano só papel; modo fixo na emissão; cancelamento; página de conferência (campos, token errado, código curto + ano errado, limite, código de outra cidade); lista de medicamentos por eventos (sem ativo duplicado); JSON canônico contra o vetor do `contracts`; analisador do CATMAT contra amostras reais; assinatura ponta a ponta com o PSC simulado.
- **dashboard:** Vitest nas telas.
- **Prova no navegador:** médica emite atestado e receita assinados; enfermeira prescreve no protocolo; recepção emite declaração; página de conferência pelo QR; cancelamento.

## 10. Entrega

1. `contracts` — `clinical_document.v1`, tag `clinical-v1.1.0`.
2. `api` — migração de cidade acima de 20261008500001; tabelas de plataforma do catálogo.
3. `dashboard`. 4. `maintenance`.

## 11. F-IDs (propostos)

| F-ID | Funcionalidade | Apps |
|---|---|---|
| F-19.16 | Interruptor `clinical_documents` e importação do catálogo de medicamentos | api, maintenance |
| F-19.17 | REMUME e protocolos de enfermagem | api, dashboard |
| F-19.18 | Lista de medicamentos em uso por eventos | api, dashboard |
| F-19.19 | Receita comum (enfermagem por protocolo; antimicrobiano em papel) | api, dashboard |
| F-19.20 | Atestado | api, dashboard |
| F-19.21 | Declaração de comparecimento (consulta e recepção) | api, dashboard |
| F-19.22 | Requisição de exames em PDF | api, dashboard |
| F-19.23 | Página pública de conferência e cancelamento | api, dashboard |

## 12. Riscos e pendências de go-live

- Curadoria real do CATMAT (itens sujos, analisador).
- OBM e registro de prescrição na RNDS (REPM/REDFM).
- Antimicrobiano digital com o SNCR (19d).
- CFO 295/2026 (atestado odontológico) — leitura jurídica.
- Atesta CFM, se a liminar cair.
- Norma interna da cidade para a declaração emitida pela recepção.
