# Módulo 26b — Contratos entre apps

**Fonte única** do que o api, o dashboard, o maintenance e o contracts combinam no 26b. As linhas do §12 prevalecem sobre o texto das seções anteriores. Os planos (`-api.md`, `-dashboard.md`, `-maintenance.md`, `-contracts-repo.md`) seguem este documento. Uma divergência vai para "Divergências propostas ao contrato" no plano, e a decisão volta aqui como Q-n.

- **Spec:** `docs/superpowers/specs/2026-11-25-module-26b-apac-oci-aih-design.md`, aprovada em 2026-11-25 com as decisões 1–22.
- **Pesquisa:** `docs/pesquisa/2026-11-24-apac-oci-teto-mac-26b.md`, achados 1–18.
- **Base:** 26a (ADR 0044), 26c (contrato de 2026-11-25), 22 (faturamento) e 19b (assinatura). Nada deles muda, salvo o que este documento diz.

**Convenções herdadas:**
- erros `{ "error" }` com `path`;
- step-up `401 mfa_required`;
- `403 feature_disabled` com `feature`;
- listas `{ items, next_before }`;
- `expected_version` → 409 `<entidade>_changed`;
- laudo, CID e justificativa só no corpo: nunca em log, URL, evento ou lista.

**Bases:** origin/main no dia da Task 0. O 26b é executado depois dos insumos do 27, que vêm depois do 26c (ordem do usuário: 27b → 26c → insumos → 26b).

**Valores fixos:**

| Item | Valor |
|---|---|
| Porta do api | 3064 |
| Vite do dashboard | 5225 |
| maintenance | 5177 |
| Migrações | as seguintes às do 26c, de cidade e de plataforma (confira na Task 0) |
| Branch | `feat/mod-26b-apac` |
| Worktrees | `apps/api/.claude/mod26b` (com `master.key`) e `rota-saude/.worktrees/<repo>-mod26b` |
| Testes | `bin/test api -w mod26b …`, `bin/test <app> -w <repo>-mod26b …`; suíte completa pela sessão "Testes" |
| Contracts | o esquema canônico do laudo entra no domínio `clinical`, como MINOR `clinical-v1.3.0` |

**Task 0 do api: o que precisa existir antes de seguir.** Sem estes três itens a execução para:
1. o layout do arquivo de APAC do SIA (FTP do DATASUS), com a versão;
2. o algoritmo do dígito verificador do número de APAC e de AIH;
3. o ZIP completo da SIGTAP da competência, com `tb_financiamento`, `rl_procedimento_compativel`, `rl_procedimento_habilitacao`, `tb_detalhe`, `VL_SH` e `VL_SP`.

O download desses arquivos precisa da autorização do usuário. Eles ficam em `vendor/regulation/<data>/`, com `SOURCE` e SHA-256, como a pasta do 26a.

**Mudança transversal.** O 26b toca:
- a SIGTAP da plataforma;
- a regra G11;
- o `Authorize` do 26a;
- o faturamento do 22 (`performed_procedures`, `BpaFile`, `Critic`);
- o signer;
- o envio ao MIRA (26c);
- o contracts.

A suíte completa do api é obrigatória.

## 1. Interruptor

`care_regulation_apac`, com o rótulo "Regulação por laudo (APAC, OCI e AIH)" e `requires: %w[feature:care_regulation sigtap_billing_data]`:
- `sigtap_billing_data` = existe uma release SIGTAP ativa com financiamento carregado (`missing`: `sigtap_billing_data_missing`).
- Sem o interruptor, o 26a continua como está e o pedido por laudo é recusado com 422 `invalid_target`.
- O teto em dinheiro (§4) vale com o interruptor ligado. Ele vale também para os pedidos BPA, quando há FPO.

## 2. SIGTAP completa (plataforma)

- **`sigtap_procedures` ganha colunas:** `amount_sh_cents`, `amount_sp_cents` e `financing_code` (tb_financiamento: `01` atenção básica, `04` FAEC, `06` MAC etc., com o nome em `sigtap_financings`).
- **Tabelas novas:**
  - `sigtap_financings` (código, nome, por release);
  - `sigtap_procedure_compatibilities` (principal, secundário, registro, tipo de compatibilidade, quantidade);
  - `sigtap_procedure_habilitations` (procedimento, habilitação);
  - `sigtap_attribute_catalog` (tb_detalhe, código e nome).
- **`Terminology::SigtapReader`** lê os arquivos novos. Todos são opcionais na leitura, mas o `sigtap_billing_data` só vale quando o financiamento entrou.

## 3. Pedido por laudo

**Alvo (G11, segundo caminho).** `config/regulation/procedures.yml` ganha:
- `apac_instruments: ["06"]` → `authorization_kind` `apac`;
- `aih_instruments` (os códigos de AIH da tb_registro, conferidos na Task 0) → `aih`;
- `oci_group: "09"` → `oci`.

`CareRegulation::Targets.resolve_procedure` aceita esses alvos com o interruptor ligado. O BPA continua na regra de hoje (`authorization_kind` `bpa`).

**`regulation_requests` ganha** `authorization_kind` (`bpa` | `apac` | `oci` | `aih`, padrão `bpa`).

**Tabela nova `regulation_laudos`** (1:1 com o pedido, cifrada):
- `clinical_summary` (20–4000), `cid_principal`, `cid_secondary` (opcional), `findings` (opcional);
- AIH: `character` (`elective` | `urgent`), `clinic` (código da clínica do SIH);
- `version`.

**Tabela nova `regulation_requested_procedures`:**
- `regulation_request_id`, `sigtap_code`, `role` (`principal` | `secondary`), `quantity`, `performed_at`;
- `performed_source` (`performed_procedure` | `manual`), `performed_procedure_id`.

**Rotas:** o `POST/PATCH consultations/:id/regulation_requests` aceita `authorization_kind`, `laudo` e `procedures`.

**Regras do OCI:**
- O principal é do grupo 09.
- Há pelo menos 2 secundários.
- Os secundários seguem as compatibilidades da SIGTAP (`sigtap_procedure_compatibilities`).

Sem compatibilidade, a recusa é 422 `incompatible_secondary` (`path` `procedures[i]`).

## 4. Autorização numerada e teto

**Faixas** (`authorization_number_ranges`, da cidade):
- `kind` (`apac` | `aih`), `fifth_digit` (APAC 2, 7, 6, 9; AIH 1, 5, 8), `year`;
- `first_number`, `last_number` e `next_number` (sem o dígito verificador);
- `active`, `created_by_user_id`.

As faixas são gravadas pelo `municipal_admin`, com step-up. Duas faixas não podem se sobrepor.

**Número:** `Regulation::AuthorizationNumber.next!(kind:, fifth_digit:)` trava a faixa (`FOR UPDATE`), monta UF + ano + tipo + sequencial e o dígito verificador (Task 0), e avança `next_number`.
- Faixa esgotada → 409 `number_range_exhausted`.
- O número é único na cidade (índice único em `regulation_authorizations.number`).

**`regulation_authorizations` ganha colunas:**
- `number` (13), `authorization_kind`, `stage` (`issued` | `awaiting_review` | `review_denied`);
- `valid_from_competence`, `valid_until_competence` (APAC Única e OCI: 2 competências; AIH: até 3 depois da alta);
- `financing` (`mac` | `faec` | `credits`, com `faec_program`), `fpo_line_id`, `reserved_cents`;
- `authorizer_professional_id`, `ceiling_override_reason` (cifrado).
- O `care_offer_id` continua obrigatório, porque a vaga é a oferta (decisão 17).

**Espelho da FPO:**
- **`fpo_lines`** (da cidade): `competence`, `care_provider_id`, `level` (`procedure` | `organization_form` | `subgroup` | `group`), `code`, `financing`, `faec_program`, `quantity`, `amount_cents`, `version`.
  - É único por (competência, prestador, nível, código, financiamento, programa).
  - É importado por CSV no dashboard (área de conferência e aplicação com step-up, como o saldo inicial do 27b) ou editado linha a linha.
- **`fpo_reservations`:** `fpo_line_id`, `regulation_authorization_id`, `competence`, `cents`, `quantity`, `released_at`, `moved_to_reservation_id`.

**`Authorize` estendido**, nesta ordem de travas: oferta → linha da FPO → faixa → pedido.
1. Ele confere o autorizador (APAC/AIH): CBO do vínculo ativo começando por `2`, nível superior; vínculo ativo numa unidade da cidade; e não pode ser o solicitante. Fora disso, 403 `authorizer_not_allowed`.
2. Escolhe a linha da FPO, a mais específica (procedimento → forma → subgrupo → grupo), para a competência prevista, o prestador e o financiamento escolhido. O padrão do financiamento vem do `financing_code` do SIGTAP.
3. Reserva `valor SIGTAP (SA + SH + SP) × quantidade`. Sem saldo, 409 `ceiling_exhausted`. Com `ceiling_override_reason` e o papel `care_regulator`, passa com step-up, marcado.
4. Tira o número (modo "antes") ou deixa `stage` `awaiting_review` (modo "depois", §5).

- **Onde vale o teto:** prestadores `own`, `contracted` e `consortium` que tenham FPO na competência. `ppi` e pedido `state_regulated` ficam fora da faixa e do teto (decisão 18).
- **Cancelamento e liberação:** devolvem a reserva (`released_at`).
- **Atendimento em outra competência:** a reserva migra, numa transação (`moved_to_reservation_id`).

**Configurações** (`regulation_apac_settings`, linha única): `authorization_moment` (`before` | `after`, padrão `before`) e `version`. Rota `GET/PUT /regulation/admin/apac_settings`, pelo `municipal_admin`, com step-up.

## 5. Modo "autorizar depois"

- **Liberação do acesso:** reserva o teto e deixa `stage` `awaiting_review`, sem número.
- **Atendimento:** a unidade registra o atendimento e os secundários.
- **Revisão:** o regulador faz `POST /regulation/authorizations/:id/review` com `decision` `issue` ou `deny`.
  - `issue`: tira o número.
  - `deny`: devolve a reserva, e os procedimentos realizados viram BPA-I (§7).

## 6. OCI

- **Andamento:** cada secundário é marcado como realizado, pelo `performed_procedure` ligado (rede própria) ou à mão pela regulação (externo: `POST /regulation/requests/:id/procedures/:pid/performed`).
- **Encerramento:** `POST /regulation/requests/:id/oci_close`, com o `exit_reason` da lista oficial (`config/regulation/oci_exit_reasons.yml`, Task 0).
- **OCI não concluída:** os secundários feitos viram BPA-I.
- **Oncologia:** o alarme mostra a OCI de oncologia aberta há mais de 25 dias corridos (prazo de 30).

## 7. Faturamento (22)

- **`performed_procedures`:**
  - ganha `regulation_authorization_id` (opcional);
  - `authorization_number` passa a ser preenchido a partir da autorização.
- **`Billing::BpaFile`:** escreve o `authorization_number` no BPA-I quando existir (hoje sai em branco).
- **`Billing::Critic`:** ganha `oci_secondary_active`. Um secundário de OCI ainda ativa não entra no BPA-I. Quando a OCI se encerra sem conclusão, o bloqueio cai.
- **Remessa APAC da rede própria:**
  - tabela `apac_exports` (competência, `health_unit_id`, `file` cifrado, `sha256`, `layout_version`, `version`, `generated_by_user_id`);
  - gerada por `POST /regulation/apac_exports` (unidade e competência), com `Regulation::ApacFile`, no layout da Task 0;
  - só das autorizações `apac` e `oci` emitidas para prestador `own`.

## 8. Fila nominal da OCI

- **`oci_queue_exports`** (competência, CSV cifrado, `sha256`, `generated_at`): no layout do e-SUS Captação de Filas (Task 0). É gerada pelo job no 1º dia útil ou à mão, e o `municipal_admin` baixa.
- **26c:** os pedidos `oci` já vão ao MIRA pelo 26c. Quando o Ministério confirmar que o MIRA cobre a fila nominal, a exportação vira só consulta.

## 9. AIH e MIRA

- **Laudo de AIH:** com `character` e `clinic`. O número sai da faixa de AIH.
- **Sem arquivo de AIH:** o hospital fatura no SISAIH01.
- **26c:** o pedido `aih` vai com a modalidade `04`. O `Regulation::RiraBundle` lê o `authorization_kind`.

## 10. Laudo em PDF e assinatura

- **PDF:** `Regulation::LaudoPdf` (Prawn, na folha do `Consultations::Print`), gerado na hora.
- **Assinatura:** o `RegulationLaudo` entra em `Signatures::DocumentTypes`, com o canônico `rotasaude.regulation_laudo.v1`, esquema novo no contracts (`clinical-v1.3.0`), e com o PDF assinado (PAdES).
- **Sem certificado:** `OpenRequest` devolve `:manual`, e o PDF sai para assinar à mão.

## 11. Telas, eventos e LGPD

**Dashboard:**
- **Consulta:** o pedido por laudo, com o tipo, o laudo e os procedimentos (OCI com secundários).
- **Fila:** o tipo do pedido. A autorização mostra a linha da FPO, o saldo, a reserva, o número emitido e o "autorizar acima do teto".
- **Regulação, configurações:**
  - faixas de numeração;
  - FPO (importação e edição);
  - momento da autorização.
- **"OCI em andamento":** os secundários, o encerramento e o alarme de oncologia.
- **"Remessa APAC":** por unidade própria e competência.
- **"Fila nominal da OCI".**
- **"Revisão de APAC":** só no modo "depois".
- **Frases:** `REGULATION_APAC_ERRORS` em `src/lib/regulationApacErrors.ts`.

**Maintenance:** o interruptor com `FEATURE_LABELS` e `MISSING_LABELS` (`sigtap_billing_data_missing`).

**Eventos** (só ids):
- `regulation_authorization.issued`
- `regulation_authorization.review_denied`
- `fpo_reservation.created`
- `fpo_reservation.released`
- `fpo_reservation.moved`
- `oci.closed`
- `apac_export.generated`
- `oci_queue_export.generated`

**LGPD:**
- O laudo é cifrado e não aparece em log, URL nem lista.
- O PDF é gerado na hora.
- Spec de sigilo: `spec/security/regulation_apac_secrecy_spec.rb`.

## 12. Decisões (Q-n)

| # | Decisão |
|---|---|
| Q1 (coordenador, contrato) | O nível superior do autorizador sai do CBO do vínculo ativo começando por `2` (grande grupo 2 da CBO, profissionais das ciências e das artes). O vínculo público é um vínculo ativo numa unidade da cidade. Não há tabela nova. |
| Q2 (coordenador, contrato) | A FPO é importada por CSV no dashboard, com área de conferência, no molde da Q11 do 27b. O maintenance não existe em produção. |
| Q3 (coordenador, contrato) | Os pedidos `oci` e `aih` já vão ao MIRA pelo 26c. A exportação da fila nominal é o plano B até o Ministério confirmar o canal. |
| Q4 (coordenador, contrato) | A assinatura do laudo exige um esquema canônico novo no contracts (`clinical-v1.3.0`). O plano do contracts é curto e vai junto com o do api. |
| Q5 (coordenador, contrato) | O teto em dinheiro dos pedidos BPA vale só com o `care_regulation_apac` ligado e com FPO cadastrada. A cidade que não liga o 26b continua só com a quantidade. |
| Q6 (api D1, D2, D3, D13) | **Ordem e alvo.**<br>- As travas seguem a ordem oferta → pedido → linha da FPO → faixa, a mesma dos comandos do 26a.<br>- `aih_instruments` é só `03`.<br>- O pedido por laudo vale só para procedimento sem instrumento BPA, dos grupos 02–04, e para a OCI do grupo 09. `Targets.resolve` e `search` ganham `kind`, e a busca devolve o `authorization_kind`.<br>- `cid_principal` é o `cid10` do pedido. A `justification` do 26a continua. O complemento não troca o CID de laudo já assinado.<br>- Na Task 0, só para a tarefa que é dona de um item faltante.<br>- A SIGTAP vai em ZIP completo no `vendor`, com SHA-256.<br>- O "1º dia útil" desconta os feriados nacionais de uma lista fixa no `config`. |
| Q7 (api D4; dashboard D5) | **OCI.**<br>- Ao menos um secundário é consulta ou teleconsulta, pela lista de códigos em `config/regulation/oci_consultation_codes.yml`.<br>- `oci_close` leva `outcome` (`concluded` \| `not_concluded`) e `exit_reason`.<br>- A OCI de oncologia é a que tem o principal em `config/regulation/oci_oncology.yml`. Os 25 dias contam do primeiro procedimento realizado.<br>- A OCI vencida (validade passada) aparece no alarme e é encerrada pelo regulador.<br>- Recusas: `oci_needs_two_secondaries`, `oci_needs_consultation`, `oci_principal_not_group_09`. |
| Q8 (api D5; dashboard D10–D11) | **Número.**<br>- O 5º dígito é escolhido pelo api. APAC: OCI → 7; créditos → 9; atributo 052 (eletiva) → 6; senão 2. AIH: créditos → 8; atributo 051 → 5; senão 1.<br>- A faixa guarda o sequencial de 7 dígitos, e o api devolve também o número inteiro.<br>- A UF sai do IBGE da cidade. |
| Q9 (api D6–D8) | **Financiamento e reserva.**<br>- O padrão vem do `financing_code` do SIGTAP: `06` → `mac`, `04` → `faec`. Os demais ficam `other`, sem reserva de teto.<br>- `faec_program` vem de `config/regulation/faec_programs.yml`. Créditos exigem a `budget_source` (Fonte Orçamentária) na autorização.<br>- A linha FAEC da FPO é só por procedimento (CHECK).<br>- O valor reservado é SIGTAP (SA + SH + SP) mais `complement_cents_per_unit` da linha da FPO, vezes a quantidade.<br>- Pedido BPA por especialidade usa o SIGTAP do mapeamento do 26c (`specialty_sigtap`).<br>- A reserva fica enquanto o pedido estiver aberto, inclusive depois da primeira falta. Ela sai no cancelamento ou no encerramento e migra quando o procedimento realizado cai em outra competência.<br>- A AIH vale até 3 competências depois da competência da alta. |
| Q10 (api D9–D11; dashboard D13, D15, D16) | **Faturamento, rotas e assinatura.**<br>- O api liga o procedimento realizado à autorização sozinho, quando o paciente, o SIGTAP e a unidade executante batem dentro da validade (gancho no `Billing::Record`).<br>- A remessa APAC da competência leva as autorizações `apac` e `oci` emitidas para unidade própria que tenham procedimento realizado na competência. Quem gera é o `municipal_admin` ou o `billing_clerk`. Unidade que não é própria → 422 `not_own_provider`.<br>- Rotas, corpos, códigos de recusa e a tabela `fpo_imports` de conferência são os do plano do api.<br>- A assinatura do laudo abre na promoção do pedido (finalização da consulta), com autor = solicitante. Os CHECKs de `signature_requests` aceitam `regulation_laudo`, e a rota do PDF é `laudo.pdf`.<br>- A negativa da revisão (modo "depois") leva motivo. |
| Q11 (api D12) | **26c.** Com o 26b, o `RiraBundle` usa a modalidade `04` para `aih` e `09` para os outros. `specialty_without_sigtap` só vale para alvo de especialidade. A mudança vai na Task 16 do plano do api do 26b. |
| Q12 (contracts D1–D3) | **Canônico do laudo.**<br>- O canônico traz cabeçalho comum, `request` e `laudo`, sem o número da autorização.<br>- Paciente sem CPF: o laudo sai impresso, para assinatura à mão, porque o `patient` comum exige CPF.<br>- Limites: `findings` até 4000, `quantity` de 1 a 999, até 30 procedimentos, `clinic` com 2 dígitos. |
| Q13 (dashboard D1–D4, D6–D9, D12, D14; maintenance DM1–DM2) | **Telas e convenções.**<br>- As ações sobre o pedido seguem o `version`/`stale_version` do 26a. As entidades novas usam `expected_version`.<br>- A lista de pedidos da própria consulta mostra o laudo (é o rascunho do autor). A fila nunca mostra.<br>- O complemento aceita laudo e procedimentos.<br>- Os pedidos por laudo aceitam a oferta do mês corrente e dos 2 seguintes.<br>- Há uma rota de prévia da autorização (linha da FPO, saldo, reserva), e o POST aceita o financiamento escolhido.<br>- O regulador lê o momento da autorização e a FPO.<br>- Competência em `AAAAMM`. O mês da oferta continua `YYYY-MM`.<br>- No maintenance, a frase de `care_regulation_disabled` vem do 26c, ou é acrescentada igual. A frase do `sigtap_billing_data_missing` diz "desligada ou incompleta". |
| Q14 (dashboard D17; maintenance DM2) | **Listas das telas e uma frase.**<br>- O api expõe `GET /regulation/apac_options`, com os motivos de saída da OCI, as clínicas do SIH, os programas FAEC e as fontes orçamentárias, lidos dos YAMLs do config.<br>- A frase "desligada ou incompleta" é a do `care_regulation_disabled` (DM2). A do `sigtap_billing_data_missing` diz que a SIGTAP da competência está incompleta. A Q13 citou o motivo errado. |
