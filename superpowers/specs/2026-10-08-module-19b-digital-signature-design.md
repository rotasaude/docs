# Módulo 19 — Consulta, subprojeto 19b (assinatura digital) — design

**Data:** 2026-10-08
**Status:** aprovado (2026-10-08)
**Afeta:**
- `apps/api`: cidade — `signer_certificates`, `signature_sessions`, `signatures`, `signature_requests`; interruptor `digital_signature`; `Signatures::*`, cliente da API de PSC do ITI (`Signatures::Psc::*`), cliente do serviço `signer`; rotas em `/signature` e campos novos nas respostas da consulta e do adendo.
- `apps/signer` (repositório novo `rotasaude/signer`): serviço interno Java 21 + Demoiselle Signer, sem estado.
- `apps/dashboard`: Minha conta → Assinatura digital, selo da sessão, estado da assinatura na consulta e no adendo, pendentes de assinatura, impresso assinado, painel do admin.
- `apps/maintenance`: interruptor `digital_signature` (mecanismo genérico), prestadores configurados e estado do `signer` (só leitura).
- `contracts`: esquema do JSON canônico `rotasaude.consultation.v1` (e do adendo) e formas da API.
- `apps/wpda`, `apps/admin`: nada.

**ADR:** `docs/adr/0032.md` · **Módulo:** `docs/modulos/19--consulta.md` · **Pesquisa:** `docs/pesquisa/2026-10-08-assinatura-digital-19b.md`

**Pré-requisito de execução:** o 19a fechado (todas as F-19.1..7 Verified). O 19b não muda os planos do 19a.

**Fora desta entrega:** assinatura gov.br e certificado A3 (token/cartão); carimbo do tempo (AD-RT) ligado; documentos clínicos (19c); receita de controlado/SNCR (19d); prontuário visível ao cidadão (20).

## 1. Ponto de partida

- 19a (ADR 0031): consulta finalizada imutável (trigger), adendos só acréscimo, impresso PDF gerado na hora (Prawn) e assinado à mão, trilha de toda leitura (`clinical_record.viewed`), texto clínico cifrado com a chave da cidade (ADR 0007).
- Interruptores por cidade com pré-requisitos (`Platform::Features`, ADR 0028), escritos só pelo maintenance.
- Step-up (ADR 0016).
- Pesquisa de 2026-10-08: a API de PSC do ITI (DOC-ICP-17.01, IN ITI 20/2020) é comum a todos os prestadores de certificado em nuvem: OAuth2 + PKCE S256, escopos `single_signature`/`multi_signature`/`signature_session` (teto de 7 dias para pessoa física), assinatura de hash com retorno RAW (PKCS#1) ou CMS, localização do titular por CPF obrigatória. O CMS devolvido não traz política ICP → a assinatura AD-RB é montada do nosso lado. NGS2 (manual SBIS v5.1) exige: CPF do certificado = CPF do usuário a cada uso; validação no servidor (antes, depois, na impressão, sob demanda) com estado Válida/Inválida/Indeterminada; certificados de ≥ 2 ACs distintas; fila de pendentes com aviso; mostrar o que será assinado; rodapé de impressão padronizado; exportação validável pelo ITI. COFEN 754/2024: ICP do profissional para prontuário sem papel; o certificado não é pago pelo profissional de enfermagem.

## 2. Decisões desta conversa

| # | Decisão |
|---|---|
| 1 | Objetivo: dispensar o papel da consulta **e** deixar a assinatura reaproveitável pelo 19c/19d. |
| 2 | Interruptor `digital_signature` no maintenance (liga/desliga), como nas funcionalidades anteriores. |
| 3 | Só certificado ICP-Brasil em nuvem, pela API de PSC do ITI. gov.br e A3 depois, sem mudar o desenho. |
| 4 | Sessão de assinatura por turno; sem sessão ou com falha, fila de pendentes e lote. Finalizar nunca depende da assinatura. |
| 5 | Adoção por profissional: com certificado assina digital, sem certificado segue no papel; cada documento registra o modo. |
| 6 | Assina o JSON canônico (CAdES destacado, fonte de verdade) **e** o PDF (PAdES), numa única aprovação. |
| 7 | AD-RB agora, guardando as provas de validade; carimbo do tempo (AD-RT) entra como configuração quando houver contrato. |
| 8 | A plataforma se cadastra em cada PSC; localização do certificado pelo CPF; VIDaaS primeiro, mais um segundo PSC. |
| 9 | O 19a não muda; o 19b só executa depois que o 19a fechar. |
| 10 | Abordagem 1: Rails conduz o fluxo; serviço interno `signer` (Java + Demoiselle Signer) monta e valida. Prova técnica primeiro. |

## 3. Interruptor e configuração

- `digital_signature` no catálogo de `Platform::Features`, `requires: ["feature:clinical_record"]`; faltando → `missing: ["clinical_record_disabled"]` (o `clinical_record` já carrega `record_mode_not_record`).
- Desligado: tudo como no 19a. Ligado e depois desligado: assinaturas existentes continuam válidas e visíveis; nada novo é assinado; pedidos `pending` passam a `returned_to_paper` com motivo `feature_disabled`.
- Credenciais dos PSCs são **da plataforma**, por ambiente, nas credenciais cifradas do Rails: `signature.providers.<key>.{client_id, client_secret, base_url}`; `<key>` ∈ catálogo fixo (`vidaas`, `birdid`, `safeid`, `neoid`, `remoteid`). PSC sem credencial = não habilitado. O maintenance só lê: nome, credencial presente, última checagem.
- Serviço `signer`: `SIGNER_URL` e `SIGNER_TOKEN` no ambiente do api/worker.

## 4. Dados (banco da cidade)

- `signer_certificates`: `user_id`, `provider` (key), `serial_number`, `issuer_dn`, `subject_cpf` (cifra determinística), `not_before`, `not_after`, `status` (`active`|`replaced`|`unlinked`|`revoked`|`expired`), `certificate_der`, `created_at`. Um `active` por usuário (índice parcial).
- `signature_sessions`: `user_id`, `signer_certificate_id`, `provider`, `access_token` (cifrado com a chave da cidade), `scope`, `started_at`, `expires_at` (≤ 12 h, mínimo entre o concedido e o teto da plataforma), `status` (`active`|`expired`|`revoked`). Um `active` por usuário.
- `signature_oauth_states`: `state`, `code_verifier` (cifrado), `user_id`, `purpose` (`link`|`session`|`batch`), `expires_at` (10 min), `consumed_at`. Uso único.
- `signature_requests` (a fila): `document_type` (`Consultation`|`ConsultationAddendum`; aberto ao 19c), `document_id`, `author_user_id`, `status` (`pending`|`signed`|`failed`|`returned_to_paper`), `reason_code`, `attempts`, `created_at`, `resolved_at`; único por documento.
- `signatures` (só acréscimos; trigger impede update/delete): `signature_request_id`, `document_type`, `document_id`, `canonical_json` (cifrado), `canonical_sha256`, `cades` (bytea cifrado), `signed_pdf` (bytea cifrado), `pdf_sha256`, `policy` (`AD-RB`; futuro `AD-RT`), `policy_oid`, `validation_material` (cadeia + LCR/OCSP do ato, bytea), `signer_certificate_id`, `signer_cpf` (cifra determinística), `signed_at`, `last_verification` (`valid`|`invalid`|`indeterminate`), `last_verification_at`, `last_verification_reasons` (códigos).
- Respostas: consulta e adendo ganham `signature: { "mode": "digital"|"manual"|"pending", "status"?, "signed_at"?, "signer_name"?, "verification"?, "reason_code"? }` — calculado; tabelas do 19a intocadas.

## 5. Fluxo

**Vínculo (Minha conta → Assinatura digital; step-up).** `POST /signature/certificates/discover` localiza o CPF do usuário em cada PSC habilitado (endpoint de localização da API do ITI); devolve os prestadores onde há certificado. `POST /signature/certificates/link { provider }` inicia OAuth (`purpose: link`, escopo `single_signature`, PKCE); no retorno, o api lê o certificado, confere CPF do titular = CPF do usuário (`certificate_cpf_mismatch`), validade e revogação, e grava `active` (o anterior vira `replaced`). Sem certificado: "você continua no papel", com instrução (médico: certificado gratuito do CFM).

**Sessão do turno.** `POST /signature/sessions` → URL de autorização (`purpose: session`, escopo `signature_session`, `lifetime` pedido = 12 h). Retorno no dashboard (`/signature/callback`) → `POST /signature/oauth/callback { state, code }` troca o código, grava a sessão. `DELETE /signature/sessions/current` encerra.

**Finalização.** `Consultations::Finalize` (19a) não muda; um assinante de `consultation.finalized` e de `consultation.addendum_added` cria o `signature_request` quando o interruptor está utilizável **e** o autor tem certificado `active`; sem certificado, nada nasce (modo `manual`). `Signatures::SignJob` (fila da cidade): com sessão ativa → monta JSON canônico e PDF → `signer /prepare` (dois documentos) → PSC (lote com os dois hashes) → `signer /assemble` → `signer /verify` → grava `signatures` e `signed`. Sem sessão → fica `pending` (`no_session`). Falha passageira (PSC/`signer`) → até 3 tentativas com espera crescente, depois `pending` com o motivo. Lock no `signature_request` (`FOR UPDATE`) impede assinatura dupla entre job e lote.

**Lote.** `GET /signature/requests?status=pending` (do próprio usuário). `POST /signature/batches` → OAuth `multi_signature` com todos os hashes pendentes (máx. 50 documentos por lote) → assina cada um como acima. `POST /signature/requests/:id/return_to_paper { reason }` (≥ 10 caracteres; só o autor) → `returned_to_paper`; o impresso volta a ter espaço para assinatura à mão.

**Adendo.** Mesmo fluxo, pelo autor do adendo. O JSON canônico do adendo leva `previous_sha256`: o `canonical_sha256` do documento assinado anterior da mesma consulta (consulta ou adendo), ou o hash do JSON canônico da consulta quando ela não foi assinada.

**Validação.** Antes de gravar (`/verify`); de novo ao abrir, imprimir e em `POST /signature/signatures/:id/verify`; estado e motivos guardados em `last_verification*`. Revogação posterior → `indeterminate`/`invalid`; o registro não muda.

**Regras.** Certificado do CPF do autor; certificado vencido/revogado não assina (`pending` com `certificate_expired`/`certificate_revoked`); ninguém assina documento de outro (`not_author`).

## 6. Serviço `signer`

- Java 21, Demoiselle Signer (LGPL), HTTP interno, token compartilhado (`Authorization: Bearer`), sem banco, sem log de corpo; saída de rede só para LCR/OCSP das ACs ICP-Brasil.
- `POST /prepare { kind: "cades"|"pades", document_base64, certificate_der, policy: "AD-RB" }` → `{ to_be_signed_sha256, prepared_state }`.
- `POST /assemble { kind, prepared_state, signature_value (RAW) }` → `{ signature_base64 (p7s ou PDF), validation_material_base64 }`.
- `POST /verify { kind, document_base64, signature_base64 }` → `{ status, signer_cpf, signer_name, policy_oid, signed_at, reasons }`.
- `GET /health` → versão, data da última atualização das LCRs.
- Fallback documentado: pyHanko (Python, MIT) no mesmo lugar, mesmas rotas.

## 7. Formatos e guarda

- JSON canônico (RFC 8785), `schema: "rotasaude.consultation.v1"` / `"rotasaude.consultation_addendum.v1"`, no `contracts`: conteúdo clínico (S, O, A, P, problemas com terminologia/código/release, condutas, exames, sinais vitais, tipo de atendimento, desfecho, horários), paciente (nome de exibição, CPF, nascimento), profissional (nome, CPF, CBO, conselho), unidade/CNES, cidade (IBGE).
- CAdES destacado AD-RB sobre o JSON (fonte de verdade); PAdES AD-RB sobre o PDF com rodapé NGS2 (nome e CPF mascarado de quem assinou, data/hora, política, "verifique em validar.iti.gov.br"), sem espaço para assinatura à mão.
- Guarda: no banco da cidade, cifrado; provas de validade no ato. Carimbo do tempo: configuração futura (AD-RT para novas; recarimbo em lote das antigas).
- Exportação: PDF assinado; pacote `.zip` com JSON + `.p7s`.

## 8. API (resumo; formas no contrato)

`/signature/certificates` (GET, discover, link, DELETE com step-up), `/signature/sessions` (POST, current GET/DELETE), `/signature/oauth/callback`, `/signature/requests` (GET, return_to_paper), `/signature/batches` (POST), `/signature/signatures/:id` (GET conteúdo legível, `pdf`, `package`, `verify`), `/signature/admin/overview` (`municipal_admin`). Erros: `feature_disabled` (feature `digital_signature`), `certificate_not_found`, `certificate_cpf_mismatch`, `certificate_expired`, `certificate_revoked`, `provider_unavailable`, `authorization_denied`, `authorization_expired`, `invalid_state`, `not_author`, `already_signed`, `not_pending`. Motivos da pendente: `no_session`, `session_expired`, `provider_unavailable`, `provider_rejected`, `signer_unavailable`, `verification_failed`, `certificate_expired`, `certificate_revoked`.

## 9. Telas

- **Profissional:** Minha conta → Assinatura digital (vínculo, troca, desvínculo, aviso 30 dias antes do vencimento); selo da sessão no topo; marcador de assinatura na consulta e no adendo; "Ver o que foi assinado", "Baixar PDF assinado", "Baixar .p7s", "Revalidar"; Pendentes de assinatura com "Assinar todas" e "Voltar ao papel"; contador no menu. Impresso: assinado → PDF assinado; manual/voltou ao papel → impresso do 19a.
- **Admin municipal:** painel Assinatura digital (com/sem certificado, vencendo em 30 dias, pendentes > 24 h, consultas por modo, assinaturas inválidas/indeterminadas); só leitura.
- **Maintenance:** interruptor por cidade; prestadores do ambiente; estado do `signer`.

## 10. Segurança e LGPD

- Token do PSC e `code_verifier` cifrados; nunca em log, evento, Analytics ou erro. `state` de uso único, amarrado a cidade, usuário e propósito.
- Eventos só com ids: `signature.certificate_linked`, `signature.certificate_unlinked`, `signature.session_opened`, `signature.signed`, `signature.failed`, `signature.returned_to_paper`, `signature.verified` (declarados em `R18_PLATFORM_EVENT_NAMES` quando forem de plataforma).
- Ler conteúdo assinado, PDF ou pacote passa por `ClinicalRecord::Access` e gera `clinical_record.viewed`.
- `signer` nunca grava documento.

## 11. Testes

- **api:** PSC falso (servidor fake da API do ITI) no ambiente de teste; `signer` real no compose de teste com AC de teste no formato ICP; specs de contrato da API do ITI; trigger de imutabilidade; corrida job × lote (nada assinado duas vezes); interruptor e pré-requisitos; adoção por profissional; volta ao papel; ausência de token/texto clínico em log e evento.
- **signer:** JUnit com certificados de teste; ida e volta (montar → verificar → adulterar falha) para CAdES e PAdES.
- **dashboard:** Vitest nas telas.
- **Prova manual:** sandbox do VIDaaS + validar.iti.gov.br.

## 12. Entrega

1. **Task 0 — prova técnica:** sandbox VIDaaS → CAdES e PAdES AD-RB aprovados no validar.iti.gov.br. Falhou → parar e rever com o usuário.
2. `contracts` (esquemas canônicos + formas).
3. Repositório `rotasaude/signer` (criação com autorização do usuário).
4. `api` → 5. `dashboard` → 6. `maintenance`.

## 13. F-IDs (propostos)

| F-ID | Funcionalidade | Apps |
|---|---|---|
| F-19.8 | Interruptor `digital_signature` e prestadores no maintenance | api, maintenance |
| F-19.9 | Vínculo de certificado ICP em nuvem por CPF | api, dashboard |
| F-19.10 | Sessão de assinatura por turno | api, dashboard |
| F-19.11 | Assinatura automática da consulta e do adendo (CAdES + PAdES AD-RB) | api, signer |
| F-19.12 | Fila de pendentes, lote e volta ao papel | api, dashboard |
| F-19.13 | Validação, conteúdo assinado e exportação | api, dashboard |
| F-19.14 | Painel de assinatura do admin | api, dashboard |
| F-19.15 | Serviço `signer` | signer |

## 14. Riscos e pendências de go-live

- Prova técnica pode falhar (política, cadeia, validador) → Task 0 antes de tudo.
- Homologação do Rota Saúde em cada PSC (prazo e exigências por prestador).
- Escolha do segundo PSC (NGS2 pede ≥ 2 ACs).
- Contrato de carimbo do tempo (custo, quem contrata) antes da primeira cidade sem papel.
- Leitura jurídica da COFEN 801/2026 (assinatura avançada) frente à 754/2024.
- Quem paga o certificado (COFEN: município, para enfermagem).
- Sem selo SBIS, a conformidade ao NGS2 (inclusive NGS1: backup, trilha, sessão) é responsabilidade do município e do diretor técnico; a dispensa do papel é decisão da cidade ao ligar o interruptor.
- Volume do PDF assinado no banco da cidade; armazenamento de arquivos quando pedir.
