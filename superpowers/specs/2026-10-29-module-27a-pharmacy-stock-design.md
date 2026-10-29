# Módulo 27a — Estoque e dispensação — design

**Data:** 2026-10-29
**Status:** aprovado (2026-10-29)
**Afeta:**
- `apps/api`: cidade — locais de estoque, lotes, movimentos (imutáveis), transferências, dispensações e itens dispensados, receitas externas, retenção de receita controlada, mínimos e alertas; papéis `pharmacist`, `pharmacy_assistant`, `stock_keeper`; interruptor `pharmacy`; dispensação na linha do tempo e na lista de medicamentos do paciente.
- `apps/dashboard`: almoxarifado (entradas, transferências, inventário, ajustes); farmácia da unidade (recebimento, dispensação pela receita, receita externa, controlados); alertas de validade e mínimo; extratos e posição de estoque.
- `apps/maintenance`: interruptor (mecanismo genérico).

**ADR:** `docs/adr/0045.md` · **Módulo:** `docs/modulos/27--farmacia-e-almoxarifado.md` · **Pesquisa:** `docs/pesquisa/2026-10-05-integracoes/frente-6-mercado.md` (e-SUS AF, concorrentes), ADR 0033 (receita), ADR 0038 (estoque do laboratório; padrão de movimentos)

**Fora desta entrega:** insumos e catálogo de materiais (entrega seguinte); envio ao BNAFAR/Hórus (27b, com pesquisa antes); SNGPC (obrigação do estabelecimento); compras, empenho e licitação; componente especializado (estadual); Farmácia Popular; disponibilidade para o cidadão no wpda; estoque do laboratório do 21c (continua separado).

## 1. Intenção

Todo medicamento que sai da farmácia da unidade sai de um lote conhecido, contra uma receita válida, para um paciente identificado; e a secretaria sabe, por local, o que tem, o que vence e o que falta, sem planilha.

**Sucesso:** o farmacêutico dispensa a receita digital pelo código em segundos; a unidade recebe a transferência do central conferindo lote a lote; o almoxarifado vê o que vence em 90 dias e o que está abaixo do mínimo; o prontuário mostra o que o paciente retirou.

**Antiobjetivos:** saldo negativo; dispensar receita cancelada ou vencida; dispensar controlado sem farmacêutico; apagar ou editar movimento; dispensar sem lote.

## 2. Decisões

| # | Decisão |
|---|---|
| 1 | **Construir** estoque e dispensação próprios, ligados à receita (19c), ao prontuário e à linha do tempo, em vez de integrar com o e-SUS AF (escolha do usuário, opção A). |
| 2 | O 27 se divide em 27a (estoque e dispensação, agora) e 27b (envio ao BNAFAR, depois de pesquisa). |
| 3 | **Locais:** almoxarifado central e a farmácia de cada unidade que dispensa. |
| 4 | **Itens:** só medicamentos do catálogo CATMAT já carregado; lote e validade obrigatórios. Insumos (seringa, fita de glicemia, lanceta, curativo) entram numa entrega seguinte, com catálogo de materiais próprio, sobre o mesmo estoque (escolha do usuário, opção A). |
| 5 | **Quantidade** na unidade de dispensação do catálogo (comprimido, frasco, ampola…), até 2 casas decimais. |
| 6 | **Movimentos imutáveis:** entrada (nota fiscal ou doação), transferência (saída do central → "em trânsito" → recebimento na unidade; diferença com motivo), dispensação, perda, vencimento, ajuste de inventário (motivo e segundo fator; acima de 10% do saldo do lote, segundo responsável) e devolução. Saldo = soma; nunca negativo. A tela sugere o lote que vence primeiro. |
| 7 | **Alertas** por local: validade em 90 dias e estoque abaixo do mínimo. |
| 8 | **Dispensação** pela receita digital (código impresso ou paciente), total ou parcial; receita cancelada ou vencida não dispensa; uso contínuo por até 30 dias de cada vez, próxima retirada a partir do 25º dia; receita externa digitada com profissional, conselho e data (auditada). |
| 9 | **Controlados (Portaria 344):** só o farmacêutico dispensa e registra a retenção da receita. |
| 10 | **Papéis:** `pharmacist` (CBO 2234; dispensa tudo), `pharmacy_assistant` (dispensa o que não é controlado), `stock_keeper` (entradas, transferências, inventário). |
| 11 | Dispensação na linha do tempo (20) e na lista de medicamentos do paciente. |
| 12 | Interruptor `pharmacy`; exclusão LGPD retém a dispensação, como o prontuário. |

Decisões 1 e 4 são do usuário; as demais são do coordenador, apresentadas e aprovadas.

**Decisões do coordenador (revisão da spec):**
- **Validade da receita:** a data `valid_until` da receita vale quando existe; sem ela, comum vale 30 dias da emissão e de uso contínuo 180 dias; os dois prazos são editáveis por cidade.
- **Quantidade:** o farmacêutico ou auxiliar digita a quantidade dispensada, na unidade de dispensação do catálogo, vendo o texto da receita (quantidade e posologia); o sistema não converte. Guarda o total dispensado por item da receita e mostra o histórico; a receita pode ser marcada como "dispensada por completo" por item. No uso contínuo, a próxima retirada do mesmo item só é liberada 25 dias depois da anterior; a quantidade de "30 dias" é julgamento do profissional.
- **Item em texto livre:** só o farmacêutico dispensa, escolhendo o item do CATMAT na hora; a escolha fica registrada.
- **Controlados:** a receita de controle especial digital (19c) é dispensada pelo farmacêutico com registro da retenção (marca e número, sem anexo). Notificação A/B em papel e receita de controle especial de fora da rede entram pela receita externa, só pelo farmacêutico, com o número da notificação quando houver.
- **Segundo fator e segundo responsável:** segundo fator é o step-up TOTP que o sistema já usa; o ajuste acima de 10% fica pendente até outro `stock_keeper` ou `pharmacist` do mesmo local aprovar.
- **Lista de medicamentos do paciente:** a dispensação aparece na leitura (última retirada por item), sem gravar em `patient_medications`.
- **Paciente:** sempre um paciente da cidade; na receita externa, encontrado pela busca de pessoa ou criado como no cadastro do 24a (inclusive sem CPF).
- **Quem atua onde:** os papéis da farmácia são vinculados a locais de estoque (`stock_location_staff`); a unidade ganha a marca "dispensa"; o `stock_keeper` do almoxarifado central opera o central e consulta a posição de todos os locais.
- **Exclusão LGPD:** a dispensação é retida inteira, como o prontuário (ADR 0026); os movimentos de estoque não dependem do paciente.

## 3. Dados

Cidade:
- `stock_locations` (central ou unidade; ativo).
- `stock_lots` (item CATMAT, lote, validade, fabricante).
- `stock_movements` (imutável): local, lote, tipo, quantidade com sinal, documento (nota, transferência, dispensação), motivo, autor, segundo responsável.
- `stock_transfers` (+ itens): origem, destino, situação (`sent`, `received`, `received_with_difference`).
- `dispensations` (+ itens): paciente, receita (digital ou externa), local, profissional, itens com lote e quantidade, retenção (controlado).
- `external_prescriptions`: profissional, conselho, data, itens.
- `stock_minimums` por local e item.

## 4. Telas (dashboard)

- **Almoxarifado:** entrada; transferência; inventário e ajuste; posição por local e lote; extrato.
- **Farmácia da unidade:** recebimento; dispensação (receita, itens, lote sugerido, quantidade, retenção); receita externa; histórico do paciente.
- **Alertas:** vencendo e abaixo do mínimo.

## 5. Segurança e LGPD

Trilha `clinical_record.viewed` ao abrir a receita e o histórico do paciente; posição de estoque sem dado pessoal e sem trilha. Nada pessoal em evento, log, URL, `inspect` ou `as_json`. Ordem de travas: lote → local → receita. Eventos novos de `Platform.audit` em R18.

## 6. Testes

- api: saldo nunca negativo sob concorrência (duas dispensações do último comprimido); transferência em trânsito e diferença; ajuste com segundo responsável; receita cancelada, vencida e contínua (25º dia); controlado só por farmacêutico; imutabilidade; retenção LGPD.
- dashboard: telas com testes de componente.
- Suíte completa do api no fim.

## 7. Entrega

api → dashboard → maintenance; runbook `operacao/rollout-farmacia.md` (inclui carga do saldo inicial por inventário de abertura).

## 8. F-IDs (propostos)

| ID | Funcionalidade | Superfície |
|---|---|---|
| F-27.1 | Interruptor, papéis e locais de estoque | api, dashboard, maintenance |
| F-27.2 | Entradas, lotes e validade | api, dashboard |
| F-27.3 | Transferência do almoxarifado para a unidade | api, dashboard |
| F-27.4 | Inventário, ajustes e perdas | api, dashboard |
| F-27.5 | Dispensação pela receita | api, dashboard |
| F-27.6 | Receita externa e controlados | api, dashboard |
| F-27.7 | Alertas, posição e extrato | api, dashboard |

## 9. Riscos

- Saldo inicial errado (mitigação: inventário de abertura por local antes de ligar).
- Unidade do catálogo diferente da unidade de compra (mitigação: entrada registra a conversão embalagem → unidade de dispensação).
- Concorrência no último item (mitigação: trava no lote e teste com threads).
