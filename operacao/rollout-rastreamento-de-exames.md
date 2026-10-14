# Rollout — rastreamento do exame (módulo 21a, ADR 0036)

Contrato: `docs/superpowers/plans/2026-10-13-module-21a-exams-contracts.md` (§15, A1–A41, prevalece).

## Ordem
1. contracts: tag `exams-v1.0.0` (autorização do usuário).
2. api: publicar a imagem; rodar `bin/rails city:migrate:all` DESSA imagem antes de cortar tráfego
   (migração de cidade 20261013900001, irreversível; triggers por `db/city_triggers.sql` no mesmo rake).
   Não há migração de plataforma.
3. dashboard e wpda; 4. maintenance; 5. catálogo público.

## Ligar numa cidade (produção não tem maintenance: rake/console, como os interruptores do 19c)
- `exam_tracking` (modo record ou integrado) → catálogo de exames e prestadores pelo admin municipal
  (step-up) → usuários do portal (convite do prestador) e do regulador (`exam_regulator`).
  Papéis `exam_regulator` e `exam_provider` exigem step-up para conceder.
- `exam_fhir_inbound` só com laboratório real homologado: token gerado no admin (mostrado uma vez),
  entregue ao laboratório por canal seguro; teste com um laudo de homologação.
  Regras da entrada (A36): só `Content-Type: application/fhir+json`; `status: final` no primeiro envio
  (`amended` → 422 `amendment_not_supported`; o laboratório fala com a unidade); reenvio do mesmo
  `identifier`, mesma guia, `final` → 200 com o mesmo `result_id`; mesmo `identifier` em outra guia →
  409 `identifier_conflict`; SIGTAP do `code` diferente do da guia → 422 `invalid_resource` (`/code`);
  PDF acima de 10 MB → `missing_pdf`. Ator na trilha: só o prestador (`actor_provider_id`).
- `exam_pec_bridge` só com `pec_external_view` já em produção na cidade e a prova do PEC da cidade.
  O botão "Solicitar exames" no PEC deduplica pedidos repetidos em janela de 15 minutos (A41).
- `fake_lab` (prestador simulado) existe só em development, test e staging (`SIMULATION_ENVS`); não existe
  em produção nem aparece no catálogo de interruptores. Prestador `simulated` com `fake_lab` desligado
  não agenda nem autoriza (422 `provider_not_enabled_for_exam`, A34) e não recebe FHIR (403 `feature_disabled`).
  A semente de produção não cria prestador simulado.
- Atores (A35): no portal, a trilha grava o prestador e o funcionário do laboratório; o cidadão nunca vê
  a pessoa nem motivo algum.

## Gates de go-live
- Primeiro laboratório real na entrada FHIR: perfil, mapeamento SIGTAP↔LOINC dos campos do catálogo,
  identificação do paciente por CPF/CNS (sistemas RNDS), limite de 60/min e corpo até 15 MB no proxy.
- Botão no PEC: prova no PEC real da cidade (versão de produção; sandbox do iframe; envio de formulário) — api#64.
- Arranjo de exames e regulação de cada cidade-piloto (SISREG, consórcio, planilha).
- Volume de PDFs por banco de cidade (até 10 MB por arquivo; acompanhar o tamanho de `exam_files`).
- Proxy repassa `/fhir/*` e `/pec/exams*` ao api, com Host e X-Forwarded-Proto.

## Desligar
- `exam_tracking` desligado: o pedido do 19a segue como requisição; exames existentes ficam guardados.
- Token FHIR revogado no admin → 401 imediato para o laboratório.
- `fake_lab` desligado fora de produção: o prestador simulado deixa de receber agendamentos.
