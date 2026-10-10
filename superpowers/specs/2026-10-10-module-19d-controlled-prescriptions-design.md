# Módulo 19 — Consulta, subprojeto 19d (receita de controlado e antimicrobiano digital) — design

**Data:** 2026-10-10
**Status:** aprovado (2026-10-10)
**Afeta:**
- `apps/api`: cidade — `sncr_number_batches`, `sncr_numbers`, categoria e campos novos na receita (`clinical_documents`), documento `controlled_notification_record`, endereço e telefone no perfil do profissional; plataforma — listas da Portaria 344 por item (A1–A3, B1, B2, C1, C2, C3, C5) e marca de anticonvulsivante; interruptores `controlled_prescriptions` e `sncr_mock`; `Sncr::Client` e SNCR simulado.
- `apps/dashboard`: Minha conta → SNCR, endereço/telefone no perfil, categoria e campos da receita, registro da Notificação, painel do SNCR do admin.
- `apps/maintenance`: interruptores (mecanismo genérico), configuração do SNCR (só leitura), aviso de simulado.
- `contracts`: `clinical-document-v1` com categoria e SNCR — tag `clinical-v1.2.0`.
- compose de dev: serviço `fake-sncr`.

**ADR:** `docs/adr/0034.md` · **Módulo:** `docs/modulos/19--consulta.md` · **Pesquisa:** `docs/pesquisa/2026-10-10-receita-controlada-19d.md`

**Fora desta entrega:** Notificação A/B/B2 eletrônica e impressa; retinoides (C2) e talidomida (C3); dispensação e baixa no SNCR; SNCR real (gate de go-live).

## 1. Ponto de partida

- 19b (ADR 0032): assinatura ICP em nuvem, pedido nasce na emissão, PSC simulado fora de produção.
- 19c (ADR 0033): receita estruturada (`clinical_documents` + `prescription_items`), catálogo CATMAT com marcas de antimicrobiano/controlado das listas Anvisa, controlado bloqueado, antimicrobiano só em papel 2 vias, página pública de conferência, cancelamento, lista de medicamentos em uso.
- Pesquisa de 2026-10-10: SNCR em produção desde 30/09/2026, instável; desde 30/10 RCE e RET eletrônicas só com número SNCR; papel segue válido. A API só entrega números: RCE/RET em blocos de 1.000 (até 3 pedidos/mês, sem VISA); Notificação 10–50/dia dentro do saldo da VISA. Autenticação pelo gov.br do prescritor (OAuth2/OIDC), token de 30 s sem refresh, retorno em domínio `.br`, `cnpj` da mantenedora. Só CRM/CRMV/CRO. Controlado exige ICP qualificada; sem formato de dados, só leiaute visual (rodapé com plataforma, URL de verificação e sncr.anvisa.gov.br). Sem cancelamento na API; baixa pela farmácia no portal.

## 2. Decisões desta conversa

| # | Decisão |
|---|---|
| 1 | Controlado em papel agora e SNCR preparado. |
| 2 | Digital já para receita de controle especial (RCE) e antimicrobiano (RET); Notificações A/B/B2 em papel. |
| 3 | Estoque de números por prescritor, reabastecido por gov.br de tempos em tempos. |
| 4 | SNCR simulado primeiro (interruptor `sncr_mock` fora de produção); SNCR real é gate de go-live. |
| 5 | Notificação de papel registrada (número do talão), não impressa. |
| 6 | Abordagem 1: estender a receita do 19c (categoria) + documento de registro da Notificação + tabelas do estoque. |

## 3. Interruptores e SNCR

- `controlled_prescriptions` (`requires: ["feature:clinical_documents"]`): libera RCE/RET digitais e o registro da Notificação. Desligado: comportamento do 19c.
- `sncr_mock` (`requires: ["feature:controlled_prescriptions"]`, só fora de produção; em produção não aparece no catálogo e o api recusa ligar): números do SNCR simulado, marcados "numeração simulada — sem validade".
- Credenciais da plataforma por ambiente (credenciais cifradas do Rails): `sncr.{client_id, client_secret, base_url, auth_url, maintainer_cnpj}`. Maintenance mostra só configurado/alcançável.
- `Sncr::Client`: login gov.br do prescritor (PKCE, `state` de uso único — mecanismo do 19b), troca do código no servidor e pedido no mesmo fluxo, dentro de 30 s; `request_rce_numbers`, `request_ret_numbers`. Token nunca guardado.
- `fake-sncr` no compose de dev: mesma forma da API real, token de 30 s, limite de 3 pedidos/mês, erro de esgotamento.

## 4. Estoque de números (cidade)

- `sncr_number_batches`: prescritor (usuário), `kind` (`rce` | `ret`), faixa recebida, `requested_at`, `simulated`, pedido do mês (1–3).
- `sncr_numbers`: `number`, `kind`, `batch_id`, prescritor, `status` (`free` | `used` | `voided`), `document_id`; índice único em (`kind`, `number`).
- Consumo: `SELECT … FOR UPDATE SKIP LOCKED` na transação da emissão; falha da emissão devolve a `free`; cancelamento da receita deixa `voided` (não volta).
- Minha conta → SNCR: saldo por tipo, pedidos do mês, "Obter números do SNCR"; aviso com saldo < 50. Saldo zero → RCE/RET saem em papel.

## 5. Receitas e Notificação

- `prescription` ganha `category`: `common` | `special_control` | `antimicrobial`, calculada pelos itens; misturar → 422 `mixed_categories`.
- `special_control` (itens C1/C5): médico e dentista; ≤ 3 substâncias C1; duração ≤ 60 dias por item (≤ 180 para anticonvulsivante); validade 30 dias; CPF do paciente (ou "não possui" + passaporte opcional) e endereço do paciente (digitado, com reaproveitar o último); endereço e telefone do prescritor (perfil do profissional; 422 `prescriber_address_missing`).
- `antimicrobial`: validade 10 dias; médico/dentista digital com número RET; enfermeiro sempre em papel (19c).
- Bloqueados: listas A e B (`requires_notification`), C2 e C3 (`not_supported`).
- Modo: digital se a autora tem certificado ativo, `digital_signature` utilizável e há número livre (consumido na emissão; pedido de assinatura nasce na mesma transação); senão papel (2 vias, sem número). Modo fixo.
- `controlled_notification_record` (médico e dentista): `notification_type` (`A` | `B` | `B2`), `paper_number`, `numbering_uf`, item do catálogo (listas A/B), quantidade, posologia, duração (≤ 30 dias A; ≤ 60 B/B2); uma substância; sem PDF nem assinatura; inclui na lista de medicamentos em uso; trilha; cancelável com motivo; não aparece na página de conferência.
- Catálogo: as listas da Portaria 344 importadas no 19c ganham a lista de cada item e a marca de anticonvulsivante.

## 6. PDF, assinatura, conferência, cancelamento

- PDF nos leiautes visuais da Anvisa (Prawn): RCE com título "Receituário de Controle Especial", número SNCR em destaque, 1ª via farmácia / 2ª via paciente; RET "Receita de Antimicrobiano", validade 10 dias; rodapé Anvisa (plataforma, URL de conferência, sncr.anvisa.gov.br) + QR do 19c + rodapé NGS2; faixa "NUMERAÇÃO SIMULADA — SEM VALIDADE" quando simulado. Papel: como no 19c.
- Assinatura do 19b (PAdES, ICP qualificada); JSON canônico com `category`, `sncr: { kind, number, simulated }` e endereços — `clinical-v1.2.0`. Pendente → "aguardando assinatura", sem impressão.
- Página de conferência: o do 19c + categoria e número SNCR; nunca medicamento ou CPF.
- Cancelamento: o do 19c; número fica `voided`; aviso de que o SNCR não tem cancelamento (a farmácia só vê na conferência do Rota Saúde).

## 7. Telas

- Profissional: Minha conta → SNCR; endereço e telefone no perfil; na receita, categoria calculada, campos de endereço/CPF do paciente em RCE, limites, aviso "vai sair em papel" (sem número/certificado/assinatura); "Registrar Notificação de papel" na aba Documentos.
- Admin municipal: painel do SNCR (saldo por prescritor, saldo baixo), só leitura.
- Maintenance: interruptores, configuração do SNCR, aviso de simulado.

## 8. Segurança e LGPD

- Token gov.br nunca persistido; `state`/`code_verifier` cifrados e de uso único.
- Número SNCR, CPF e endereço fora de log, evento, Analytics e erro.
- Eventos só com ids: `sncr.numbers_requested` (kind, count), `sncr.number_used`, `controlled_notification.recorded`, `clinical_document.issued/cancelled` (com `category`).
- Leitura pelas regras do 19a, com trilha.

## 9. Testes

- **api:** categoria e `mixed_categories`; bloqueio por lista; limites (duração, substâncias, validade); quem prescreve; consumo do número com lock e corrida real; devolução na falha, `voided` no cancelamento; fallback para papel; contrato do `fake-sncr` (30 s, 3/mês, esgotamento); registro da Notificação na lista de medicamentos; JSON canônico contra `clinical-v1.2.0`; invariantes (nenhum número simulado em produção).
- **dashboard:** Vitest.
- **Prova no navegador** (SNCR e PSC simulados): obter números; RCE e RET assinadas; registro de Notificação; cancelamento com número anulado; painel do admin.

## 10. Entrega

0. Prova técnica: troca do token no servidor dentro dos 30 s (documentação + homologação, se houver acesso). Falhou → parar e rever com o usuário (plano B: navegador chama o SNCR).
1. `contracts` `clinical-v1.2.0`. 2. `api` (migrações acima das do 19c; `fake-sncr` no compose). 3. `dashboard`. 4. `maintenance`.

## 11. F-IDs (propostos)

| F-ID | Funcionalidade | Apps |
|---|---|---|
| F-19.24 | Interruptores `controlled_prescriptions` e `sncr_mock` e configuração do SNCR | api, maintenance |
| F-19.25 | Estoque de números SNCR por prescritor | api, dashboard |
| F-19.26 | Receita de controle especial digital | api, dashboard |
| F-19.27 | Receita de antimicrobiano digital | api, dashboard |
| F-19.28 | Registro da Notificação de papel | api, dashboard |
| F-19.29 | Painel do SNCR do admin | api, dashboard |

## 12. Riscos e pendências de go-live

- Cadastro da integração na Anvisa (CNPJ da mantenedora num SaaS municipal; domínio `.br` de retorno) e prova com o SNCR real.
- Troca do token no servidor (prova técnica).
- Cancelamento e QR do SNCR — pergunta à Anvisa.
- Regra de cada VISA para Notificação impressa.
- Retinoides e talidomida.
- Instabilidade da API do SNCR (queda, esgotamento, números de outro prescritor).
