# Rollout — farmácia e estoque (módulo 27a, ADR 0045)

Deploy, ligação por cidade, inventário de abertura, operação e limites conhecidos da farmácia municipal (api;
dashboard e maintenance seguem o api). Spec: `superpowers/specs/2026-10-29-module-27a-pharmacy-stock-design.md`.
Contrato: `superpowers/plans/2026-10-29-module-27a-pharmacy-stock-contracts.md` (decisões H1–H14).
Interruptor: `pharmacy` ("Farmácia e estoque"). Papéis: `pharmacist`, `pharmacy_assistant`, `stock_keeper`.

## Pré-requisitos

- **Catálogo CATMAT e listas Anvisa ativos na plataforma.** O item do estoque é um item CATMAT do catálogo; a marca
  de controlado vem da lista da Portaria 344 e a de antimicrobiano (`antimicrobial`) vem da lista Anvisa. Sem as
  listas a dispensação não sabe pedir retenção. Sem o catálogo a busca de item responde 503 `catalog_unavailable`.
- **Itens CATMAT sem `dosage_form`: medir antes do go-live.** A unidade de dispensação (comprimido, ampola, frasco...)
  sai de `dosage_form` pela tabela YAML do api (`config/pharmacy/dispensing_units.yml`) e fica gravada no lote na
  primeira entrada (`stock_lots.dispensing_unit`). Item sem forma cai em "unidade". Conte quantos itens CATMAT ativos
  têm `dosage_form` nulo e leve a lista à assistência farmacêutica: esses itens aparecem como "unidade" no estoque
  (ver gates).
- **Unidades que dispensam com cadastro** no módulo 09 (ativas; CNES conferido). A farmácia da unidade é um local
  de estoque do tipo `unit` e liga a marca `health_units.dispenses`.
- **Farmacêuticos com vínculo profissional ativo e CBO `2234*`** (módulo 10). Vincular um `pharmacist` a um local
  sem esse CBO responde 422 `cbo_not_allowed`.
- **Quem terá cada papel.**
  - `pharmacist` e `pharmacy_assistant` são **privilegiados**: concedidos pelo `municipal_admin` **com step-up**; o
    mantenedor não concede. Quem tem o papel lê e dispensa receitas de pacientes da cidade (ver LGPD).
  - `stock_keeper` (almoxarife/central) não é privilegiado.
  - O papel efetivo no local é a `Membership` ativa **e** o vínculo ativo ao local (`role_not_granted`).
- **Perfil profissional para toda a equipe da farmácia** (módulo 10), inclusive auxiliares e almoxarifes. O nome de
  quem dispensou, recebeu ou ajustou vem do perfil profissional (o usuário não tem nome próprio): sem perfil, o
  histórico de retiradas, os ajustes e o extrato mostram "—". Basta o perfil com nome; o vínculo de CBO é exigido só
  do farmacêutico.
- **Worker na imagem nova** (consumidores de eventos do módulo; worker antigo falha com
  `ActiveJob::UnknownJobClassError`).
- Nenhuma gem nova; nenhuma migração de plataforma.

## Ordem de deploy

Migração de **cidade** `20261029900001` **irreversível** (o `down` levanta `IrreversibleMigration`): não há rollback
de schema, só do código (expand/contract). Ela mexe no CHECK de papéis (`ck_memberships_role` passa a aceitar
`pharmacist`, `pharmacy_assistant` e `stock_keeper`) e em `health_units` (coluna `dispenses`), e cria 15 tabelas,
triggers de guarda (`stock_*`, `dispensation*`, `external_prescription*`, incluindo `stock_movements_balance`) e o
índice parcial `idx_stock_adjustments_one_pending` (no máximo um ajuste pendente por item e local). Publicar a imagem
e rodar o `bin/migrate` **dela**, antes de cortar o tráfego. Nunca migrar cidade fora do rake.

1. **api**: publicar a imagem (api e worker) e rodar o `bin/migrate` da imagem (`db:migrate`, `platform:triggers`,
   `city:migrate:all`). Fora do pico; meça antes em staging com volume real (a migração troca o CHECK de
   `memberships` e altera `health_units`).

   ```bash
   bin/migrate
   ```

2. **dashboard** (depois do api).
3. **maintenance** (depois do dashboard): rótulo do interruptor e do módulo no catálogo.
4. **Ligar e preparar, nesta ordem** (a ordem é do go-live, H12):
   1. maintenance: ligar `pharmacy` na cidade. O interruptor vale para **todo `/pharmacy/*`, inclusive o admin**:
      sem ele nem o cadastro de locais abre. Ligar não anuncia nada ao público interno.
   2. Admin municipal: criar o **central** e as **farmácias das unidades**; vincular as pessoas aos locais
      (`pharmacy/admin/locations`, equipe com os papéis acima).
   3. Conferir os **parâmetros** (valores de partida: validade comum 30 dias e contínuo 180; intervalo do uso
      contínuo 25 dias; alerta de vencimento 90 dias; aprovação de ajuste acima de 10%) e cadastrar os
      **mínimos** por item e local.
   4. **Inventário de abertura** (próxima seção).
   5. **Só então anunciar a dispensação** às farmácias. Sem saldo, a dispensação recusa com `insufficient_balance`;
      o anúncio antes da abertura só produz recusas.

## Inventário de abertura

O estoque nasce do inventário. Não existe saldo por planilha importada.

- **Contagem física por local e lote**, com validade e fabricante, digitada em `POST /pharmacy/openings`, **um local
  por vez**, antes de liberar a dispensação.
- **Uma abertura por item e local** (409 `opening_exists` na segunda). Corrigir depois **só por ajuste**.
- A abertura grava a unidade de dispensação no lote (regra de `dosage_form`; item sem forma vira "unidade"). Confira
  a unidade de cada item **antes** de digitar: depois de gravada no lote ela não muda.
- **Conferir a posição contra a contagem** com o farmacêutico responsável do local (`GET /pharmacy/positions`) e
  **guardar a planilha assinada da contagem fora do sistema**.
- Quantidade: string com até 5 inteiros e 2 casas; a saída sempre vem com duas casas e ponto (`"30.00"`).

## Conferências pós-deploy

- Parâmetros da cidade:

  ```sql
  SELECT * FROM pharmacy_settings;
  ```

- Triggers (com `stock_movements_balance` presente):

  ```sql
  SELECT tgname FROM pg_trigger
   WHERE tgname LIKE 'stock_%' OR tgname LIKE 'dispensation%' OR tgname LIKE 'external_prescription%';
  ```

- Índice do ajuste pendente (esperado: uma linha, índice parcial):

  ```sql
  SELECT indexname, indexdef FROM pg_indexes WHERE indexname = 'idx_stock_adjustments_one_pending';
  ```

- Nenhum saldo negativo (esperado: vazio):

  ```sql
  SELECT stock_location_id, stock_lot_id, SUM(quantity) FROM stock_movements
   GROUP BY 1, 2 HAVING SUM(quantity) < 0;
  ```

- CHECK de papéis aceita os três (`ck_memberships_role`) e `health_units.dispenses` existe.
- Logs: `prescriber_name`, `council`, `council_state`, `council_number` e `supplier` saem **filtrados em todo log da
  aplicação**, inclusive do cadastro de profissionais (módulo 10). Faça `grep` de um valor de teste nos logs do dia.
- **Só em STAGING:** uma dispensação de teste, da busca da receita ao extrato (posição, movimento, linha do tempo).
  **Em produção, nada de dispensação de teste:** o movimento é só acréscimo, entra no saldo real e retém o paciente na
  exclusão LGPD. Em produção verifique só leitura (`GET /pharmacy/locations`, `GET /pharmacy/positions`).
- Falha no trigger de saldo ou saldo negativo: **não ligue a dispensação**; o diagnóstico é da sessão dona do api.

## Operação

- **Transferência:** fica **em trânsito** (`sent`) até a unidade de destino receber. Recebida inteira, uma vez
  (`already_received`, `invalid_items`). Diferença entre enviado e recebido vira **perda com motivo** no destino;
  recebido acima do enviado é recusado.
- **Ajuste:** perda e vencimento (`expiry`, só de lote já vencido) são aplicados na hora, com motivo e step-up. O
  **ajuste de contagem acima de 10%** (saldo zero conta como acima) fica `pending` até **outra pessoa do local**
  aprovar (`same_user` se for a mesma). Entrada por nota ou doação é só do `stock_keeper`.
- **Devolução:** volta ao **mesmo lote** (`POST /pharmacy/returns`, `return_exceeds` se passar do dispensado).
- **Desativar um local de estoque:** exige **saldo zero**, nenhuma transferência `sent` e nenhum ajuste `pending`
  (409 `location_has_stock`).
- **Desativar uma unidade de saúde (módulo 09) depois do go-live (H14):** se a farmácia dela tem transferência em
  trânsito ou ajuste pendente, responde **409 `pharmacy_pending`**. **Receba as transferências e decida os ajustes
  antes**, depois desative a unidade.
- **Dispensação:** total ou parcial; receita cancelada ou vencida é marcada e recusada. Lote que vence primeiro é
  sugestão da tela (o api devolve `expires_on` na posição). Uso contínuo respeita o intervalo de 25 dias por item
  da receita.
- **Antimicrobiano (RDC 471/2021):** a **retenção da 2ª via** é registrada por **quem dispensa, farmacêutico ou
  auxiliar**. A marca vem do catálogo da plataforma (`antimicrobial`, lista Anvisa), tanto na receita digital quanto
  na externa; quem digita não marca. O api calcula `retention_required` por item.
- **Controlados (Portaria 344):** **só o farmacêutico dispensa e registra a retenção.** A receita digital de
  controle especial usa o **número SNCR do documento** (`sncr.number`; número divergente é recusado). **Notificação
  A/B em papel** e **receita de fora da rede** entram como **receita externa**, com o número da notificação
  (`notification_required` para item de lista A/B); receita externa com item controlado vale 30 dias.
- **Texto livre** na dispensação: só o farmacêutico.
- **Receita externa:** conselho em `CRM`, `CRO` ou `COREN`; os dados do prescritor e do fornecedor não vão a log.

## O que a v1 não faz

- **Não envia ao BNAFAR/Hórus** (é o módulo 27b) e **não substitui o SNGPC**: a escrituração continua onde já é feita.
- Sem **compras, empenho ou licitação**.
- Sem **componente especializado** nem **Farmácia Popular**.
- Sem **insumos e materiais** (entrega seguinte).
- Sem **disponibilidade de medicamento para o cidadão** (o wpda não muda).
- Sem saldo por planilha importada; sem conversão de embalagem (a quantidade digitada é a da unidade de dispensação).

## LGPD

- **Dispensação e receita externa retêm o cadastro** do paciente na exclusão (como o prontuário). Os movimentos de
  estoque não dependem do paciente.
- **Escopo de leitura (permitido pelo contrato §7):** farmacêutico ou auxiliar **de qualquer unidade** lê **um ano de
  receitas de qualquer paciente da cidade** (`POST /pharmacy/prescriptions/lookup` por paciente: receitas dos últimos 365 dias e
  externas, até 20). É um alcance amplo, escolhido para o paciente ser atendido em qualquer farmácia da rede.
- **Cada leitura deixa trilha `clinical_record.viewed`** (`access: "pharmacy"`). **Recomendação:** o encarregado
  revisa a trilha periodicamente (acessos fora da unidade de lotação, volume por pessoa, horários) e a secretaria
  decide a periodicidade antes do go-live.
- A busca de receita e de pessoa tem limite (429 `too_many_requests`).
- Linha do tempo: `dispensation` aparece **só para o profissional** (fora de `citizen_kinds`; o wpda não muda), com
  os nomes dos medicamentos, sem lote nem quantidade, e **segue visível com o interruptor desligado** (registro
  retido).

## Desligar e reverter

- Antes de desligar o interruptor `pharmacy` de uma cidade, receba todas as transferências em trânsito e decida os
  ajustes pendentes. Com o interruptor desligado, a guarda H14 não vale. Uma unidade de saúde desativada nessa
  situação deixa a transferência presa em `location_inactive`, sem saída pelo sistema.
- Desligar `pharmacy` (maintenance): **todo `/pharmacy/*`** responde 403 `feature_disabled`, inclusive o admin; as
  telas somem. **Nada se apaga:** locais, lotes, movimentos, dispensações, retenções. O estoque fica **parado como
  está**: nada vence, nada transfere, nada é dispensado, e ligar de novo não reprocessa nada.
- A linha do tempo e a **última retirada** na lista de medicamentos do paciente seguem.
- A migração **não volta**: sem rollback de schema; volte só o código (expand/contract). Os papéis seguem aceitos no
  CHECK e `health_units.dispenses` segue existindo.

## Gates de go-live

- **Inventário de abertura conferido por local**, com a planilha assinada guardada fora do sistema.
- **Regra de unidade de dispensação conferida pelo farmacêutico responsável contra a REMUME**, com a contagem de itens
  CATMAT ativos de `dosage_form` nulo em mãos e uma decisão para cada um.
- **Prazos de validade** (30 e 180 dias) **aprovados pela assistência farmacêutica da cidade**.
- **Impressão e retenção de controlados** (e do antimicrobiano) **conferidas com a vigilância sanitária**: o que se
  retém, em que via, e quem assina.
- Revisão periódica da trilha `clinical_record.viewed` combinada com o encarregado (LGPD).
- Quem recebe `pharmacist`, `pharmacy_assistant` e `stock_keeper` definido e concedido pelo admin com step-up, cada
  um com perfil profissional com nome.
- **SNCR simulado provado** com a imagem atual do api (o `fake-sncr` de dev estava parado na prova de 2026-10-30):
  dispensar uma receita de controle especial emitida com certificado e conferir o número do documento na retenção.
- `CityProfile` e unidades com CNES conferidos; o central e as farmácias criados.

## Riscos conhecidos

- **Item sem forma farmacêutica** cai em "unidade" e a unidade, uma vez gravada no lote, só muda por novo lote.
- **Abertura errada** só se corrige por ajuste, e o ajuste acima de 10% espera uma segunda pessoa.
- **Estoque parado ao desligar** o interruptor: lotes continuam vencendo no calendário sem o sistema registrar.
- **Dev:** só é migrado e semeado na prova no navegador. A migração de cidade é irreversível e deixa os bancos de dev
  à frente da main (a parity spec pode quebrar em main até o DROP + `city:test_databases`).
