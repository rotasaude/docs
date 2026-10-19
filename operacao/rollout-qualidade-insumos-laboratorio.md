# Rollout — qualidade e insumos do laboratório municipal (módulo 21c, ADR 0038)

## Pré-requisitos
- 21b em produção na cidade (`municipal_lab` utilizável; liberadores com RT e substituto designados).
- Tag do contracts publicada: `lab-v1.1.0` (seção "Controle" do perfil HL7; exemplos copiados em
  `apps/api/spec/fixtures/lab/`).
- Imagem do api **reconstruída**. A gem `barby` do 21b continua necessária (container antigo não tem a gem).
  O 21c não traz gem nova nem migração de plataforma.

## Ordem de deploy
1. contracts (tag `lab-v1.1.0`) → 2. api: publicar a imagem e rodar `bin/migrate` dela ANTES de cortar tráfego
   (`city:migrate:all` — 20261018900001, **irreversível**). A migração:
   - cria as tabelas novas (aparelhos, CIQ, CEQ, estoque, relatório anual do CEQ);
   - acrescenta `lab_exam_versions.lab_instrument_id` e `lab_results.reagent_lot_id`;
   - acrescenta a coluna nula `purpose` na tabela `exam_files` do 21a (C-E26);
   - o deploy relê `db/city_triggers.sql` (triggers novos de só-acréscimo e de saldo).
3. dashboard → 4. maintenance → 5. catálogo público. O wpda não muda.

Nunca migrar cidade fora do rake. A migração mexe em `exam_files` (21a): rode fora do pico.

## Interruptores (maintenance)
- `lab_quality_control` (CIQ e CEQ) e `lab_inventory` (estoque, lote em uso, rastreio, consumo).
- Os dois são independentes e existem em todos os ambientes. Os dois exigem `municipal_lab` ligado.
- Ligar por cidade, no maintenance, um de cada vez. Ligue o CIQ primeiro: o estoque não depende dele.
- O que muda para o laboratório:
  - com `lab_quality_control`: aparece a área de qualidade; a liberação de laudo passa a respeitar o bloqueio
    por falha de controle; a HL7 passa a aceitar controles;
  - com `lab_inventory`: aparece o estoque; o lote em uso passa a ser anotado em cada resultado digitado ou
    recebido do aparelho.
- Os aparelhos (cadastro e vínculo com exame) abrem com qualquer um dos dois interruptores.

## Preparar a cidade
Quem faz: admin da cidade (aparelhos e vínculos) e RT ou substituto (CIQ). Ordem:
1. **Aparelhos.** Cadastrar cada aparelho com o `hl7_equipment_id` que o middleware manda em `OBX-18`
   (até 50 caracteres; sem diferenciar maiúsculas de minúsculas).
2. **Exame × aparelho.** Criar uma nova versão de cada exame ligada ao aparelho. Só itens de versão com
   aparelho entram no bloqueio e no rastreio. Exame sem aparelho continua liberando como no 21b.
   Versão nova sem o campo `lab_instrument_id` herda o aparelho da versão vigente (C-E44), mesmo com os
   interruptores do 21c desligados ou com o aparelho já inativo; só `null` explícito desvincula. Desligar o CIQ
   por um tempo não solta o vínculo.
3. **Material de controle e lote.** Cadastrar o material (um analito por `analyte_key`; o mesmo analito em dois
   exames é recusado) e o lote, com um código de barras por nível.
   - Formato: `^[A-Za-z0-9._-]{3,30}$`.
   - Nunca só 10 dígitos: esse é o espaço das amostras (recusado).
   - Único na cidade e sem diferenciar maiúsculas de minúsculas; guardado como digitado.
4. **Referência da bula** por nível, analito e aparelho (média e desvio-padrão). Troca de referência cria versão
   nova com motivo; o gráfico mostra a troca.
5. **Regras de Westgard** por analito. Sem linha, todas ficam ligadas. Desligar uma regra ligada pede
   justificativa (até 1000 caracteres). As versões das regras só crescem.
6. **Papéis.** Referência aceita, regras, exceção e revisão: só RT e substituto. Ação corretiva: qualquer
   membro do laboratório. Gráfico de Levey-Jennings: liberador ligado, RT, substituto e admin (`lab_staff` recebe 403).
7. **Estoque** (se `lab_inventory`): itens (rendimento do reagente, mínimo, dias de alerta de validade, exames
   ligados), entrada dos lotes e lote em uso por aparelho e exame.

## HL7 — controle
- O controle é reconhecido por `SPM-2` = código de barras de um nível de lote cadastrado. Aparelho em `OBX-18`.
- Recomendado: `SPM-11` = `Q`. `INV` é aceito e ignorado; `SPM-4` é ignorado no controle.
- Decimal com **ponto** (`98.5`). Vírgula (`98,5`) → `AE invalid_value`.
- Mensagem com controle e paciente: os controles entram antes das amostras. A mensagem entra inteira ou nada.
- Reprovação de Westgard não muda o `ACK` (continua `AA`): o efeito é o bloqueio, não a recusa.
- Recusas novas: `qc_disabled` (interruptor desligado), `qc_lot_expired`, `qc_lot_closed`, `qc_no_target`,
  `unknown_instrument`, `unknown_barcode`. Corrida em aparelho inativo é recusada.

**Mudança de comportamento do 21b (C-E38).** `SPM-11 = Q` com código de **paciente** passa a ser recusado com
`AE unknown_barcode`, mesmo com `lab_quality_control` desligado (o controle nunca vira paciente). Um middleware
real que mande `Q` por engano começa a receber `AE` assim que o api novo subir. **Confira com o fornecedor antes
do go-live** se algum fluxo de paciente manda `Q`.

## Simulado (`fake_middleware`)
- NUNCA em produção (não existe lá). Em staging, só com dados de teste.
- `run.rb qc reject_2_2s` manda corrida reprovada: isso **ABRE UM BLOQUEIO** de liberação daquele analito no
  aparelho. Em staging, use só com amostras de teste e saiba que a liberação do analito fica travada até uma
  corrida aprovada fechar o bloqueio (plano M2). Não rode onde há laudo de teste em liberação sem avisar o time.

## Bloqueio, exceção e revisão
- **Bloqueio.** Corrida reprovada abre bloqueio do analito no aparelho. Liberar item bloqueado → 409
  `qc_blocked` (com o item e o aparelho). Fecha quando há ação corretiva registrada e uma corrida aprovada
  (`approved` ou `warning`) com valores de todos os níveis gravados depois da ação.
- **Valor aceito sem ação corretiva.** O valor entra (201) com `notice: corrective_action_required`; o bloqueio
  segue ativo. Texto em branco ou acima de 1000 caracteres → 422 `invalid_text`.
- **Exceção do RT/substituto.** Com justificativa e TOTP (step-up), uma por bloqueio e item. Vale para item
  `pending`/`typed` e também para a **retificação** de item já liberado (C-E19). Fica na trilha.
- **Revisão.** Quando o CIQ reprova, os itens liberados desde o `started_at` da última corrida aceita do
  analito (C-E21) entram na revisão. Laudos ICP ainda `pending_signature` entram também (C-E22); a
  finalização ICP não consulta o CIQ, para não jogar fora uma assinatura feita. O RT decide `kept` ou
  `rectify` (abre a retificação do 21b).

## CEQ
- Rodada: programa, código, data (não futura) e um item por analito, com ação obrigatória para item inadequado.
- Relatório anual: um por laboratório, programa e ano, só de acréscimo (repetido → 409 `already_recorded`).
- O PDF entra **só** por `POST /lab/eqa/uploads` (C-E24; propósito do arquivo em C-E25/C-E26). Upload de outra origem (laudo, guia) → 422
  `invalid_file`. Depois `POST /lab/eqa/annual_reports`; leitura em `GET /lab/eqa/annual_reports/:id/pdf`, sem trilha.
- Rodada ou relatório de outro laboratório → 404.

## Estoque
- Movimentos só de acréscimo, com estes tipos: `receipt` (entrada, só pela entrada de lote), `opening` (abertura
  de kit), `loss` (perda), `expiry` (vencido descartado) e `adjustment` (ajuste, com sinal, só admin). Fora `opening`,
  todo movimento pede motivo (`reason_required`; acima de 1000 → `invalid_reason`). O saldo nunca fica negativo
  (`insufficient_balance`). Não há exclusão.
- Entrada com mesmo número e mesma validade soma no lote existente; outra validade → `duplicate_lot`.
- Alertas (`GET /lab/inventory/alerts`), três chaves: `expiring` (lotes com saldo que vencem dentro do
  `expiry_alert_days` do item, ou já vencidos), `below_minimum` (itens ativos abaixo do mínimo; conta só lotes
  **não vencidos**, C-E28) e `untraced_results` (contagem de resultados medidos dos últimos 30 dias, de exame com
  aparelho, sem lote de reagente).
  O contador `GET /lab/inventory/alerts` não grava trilha.
- Lote em uso: um por aparelho e exame. Marcar de novo o mesmo lote → 200, nada criado. O lote em uso é lido uma
  vez por chamada e anotado no resultado (valor calculado herda o lote só se as medidas têm o mesmo lote, C-E30).
- Rastreio por lote (tela e CSV, sem nome de paciente): corta em **5000 linhas** por lista e avisa com
  `results_truncated` e `qc_values_truncated` (C-E37). Corte significa: refaça com período menor.
- Consumo do mês (`month` = `YYYY-MM`): **estimado**, sem baixa automática do estoque (C-E32/C-E33). Cada linha traz
  `line` = `total`, `instrument` ou `no_instrument` (exame sem aparelho); `instrument` + `no_instrument` = `total`.
  `tests_opened` e `deviation_pct` podem ser null. A baixa continua sendo ato humano (movimento).

## Gates de go-live
- Middleware real mandando controles: códigos de barras dos níveis e `OBX-18` mapeados com o fornecedor;
  mensagem aprovada, reprovada e as recusas (`qc_lot_expired`, `qc_lot_closed`, `qc_no_target`,
  `unknown_instrument`, `unknown_barcode`) conferidas. Incluir a pergunta do C-E38 (`Q` em amostra de paciente).
- Validação do CIQ pelo RT do laboratório (entra no dossiê do art. 106 da RDC 978).
- Coleta no posto do laboratório (21b): hoje `lab_staff` recebe 403 na lista de unidades de origem e a requisição
  não abre. Não ligue `municipal_lab` numa cidade com coleta no posto do laboratório antes da api#89.
- Etiqueta e leitor: a prova de impressão e leitura herdada do 21b (203 dpi, 50 x 25 mm) vale; repita se trocar
  modelo de impressora ou leitor.
- Laboratório com CIQ no software do provedor de controle: convive digitando os valores até a importação CSV
  (api#87).
- Volume: acompanhar o crescimento de `lab_qc_values` por banco de cidade no primeiro mês.

## Verificação depois de ligar
- `GET /lab/qc/blocks/active/count` com uma conta do laboratório (sem trilha).
- Digitar uma corrida de controle; liberar um item de teste com e sem bloqueio; `GET /lab/inventory/alerts`.
- Nenhum valor de controle nem justificativa no log do dia (`grep` do valor de teste).

## Conhecido (Ciclo 1)
- Listas cortam sem paginação: bloqueios 200, rodadas do CEQ 200, relatórios anuais do CEQ 200, exceções 500,
  revisões 500; o gráfico de Levey-Jennings traz os últimos 200 pontos. Lotes, materiais e corridas não têm corte.
  Acompanhe em cidade grande.
- Todas as listas do 21c exigem `provider_id`; não há laboratório único implícito.
- A semente de dev, ao rodar de novo, re-seleciona o lote em uso da semente; não use re-run para testar troca de lote.
  A semente não cria CEQ, exceção, bloqueio nem consumo.
- CSV do rastreio traz duas linhas finais `results_truncated` e `qc_values_truncated` (`true`/`false`). Esse formato
  não está no texto do contrato (C-E37 só fala dos booleanos).
- `last_qc_on` do simulado vive em memória: reiniciar o simulado no mesmo dia soma corridas do mesmo sinal
  (cerca de 10 disparam 10x).

## Desligar
- `lab_quality_control` desligado: rotas `/lab/qc` e `/lab/eqa` respondem 403; a liberação do 21b deixa de
  consultar bloqueios (os registros ficam); a HL7 recusa controles com `qc_disabled`. A regra do C-E38 (`Q` com
  paciente → `AE`) continua valendo.
- `lab_inventory` desligado: rotas `/lab/inventory` respondem 403; o lote em uso já gravado continua sendo anotado
  nos resultados até ser encerrado — encerre os lotes em uso antes de desligar para parar o rastreio.
- Desligar nunca apaga dados. Ligar de novo retoma de onde parou.

## Rollback
- Migração irreversível (inclui a coluna `purpose` em `exam_files`): não há rollback de schema.
- Volte só o código (expand/contract). A coluna `purpose` é nula e inofensiva para a imagem anterior; as tabelas
  novas ficam sem uso.
