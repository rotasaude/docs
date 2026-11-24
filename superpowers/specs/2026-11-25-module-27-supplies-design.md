# Módulo 27 — Insumos (catálogo de materiais, diabetes e estomia) — design

**Data:** 2026-11-25
**Status:** aprovado pelo usuário (2026-11-25)

**Afeta:**
- **`apps/api`, plataforma:** catálogo de materiais CATMAT (classes 6515 e 6550).
- **`apps/api`, cidade:**
  - quarta chave de item do estoque (`supply`);
  - inscrição em programa;
  - cotas por programa;
  - dispensação por inscrição;
  - empréstimo de aparelho (comodato);
  - procedimento faturável da bolsa de estomia.
- **`apps/dashboard`:**
  - inscrição na consulta;
  - dispensação por inscrição na farmácia;
  - catálogo de insumos nas telas de estoque;
  - comodato;
  - configuração das cotas.
- **`apps/maintenance`:** carga do catálogo de materiais.
- **`apps/catalog`:** o teste que hoje proíbe "insumo|material" no catálogo público muda.

**Módulo:** `docs/modulos/27--farmacia-e-almoxarifado.md` · **ADR base:** 0045 (27a).
**Pesquisa:** `docs/pesquisa/2026-11-23-insumos-e-bnafar-27.md`, achados de insumos 1–8.

**Fora desta entrega:**
- Fralda, que vai pelo Farmácia Popular.
- Preservativo e outra saída sem paciente.
- Incontinência e outros programas, que entram depois no mesmo molde.
- Glicosímetro e sensor contínuo como item de estoque: são patrimônio, não consumo.
- Compras.

## 1. Intenção

A farmácia da unidade entrega tiras, lancetas, seringas e bolsas de estomia a quem está inscrito no programa, dentro da cota definida pela cidade, e o estoque desses materiais é controlado como o dos medicamentos.

**Por quê:**
- Os editais pedem almoxarifado completo, e o 27a só cobre medicamento.
- Os insumos de diabetes são obrigação nacional (Lei 11.347/2006 e Portaria 2.583/2007).
- A bolsa de estomia é procedimento SIGTAP, registrado em BPA-I.
- O 27b precisa dos insumos no estoque para enviar a posição completa à BNAFAR.

**Sucesso:**
- O médico ou o enfermeiro inscreve o paciente no programa na consulta.
- A farmácia entrega o que cabe no mês, sem receita a cada retirada, e o sistema bloqueia o que passa da cota.
- O glicosímetro emprestado fica registrado com número de série, termo e devolução.
- A bolsa entregue entra no faturamento do mês.

**Antiobjetivos:**
- Cota fixa no código.
- Entrega sem inscrição válida.
- Glicosímetro saindo como consumo de estoque.
- Bolsa entregue sem virar produção.

## 2. Decisões

| # | Decisão | De quem |
|---|---|---|
| 1 | Entram o catálogo de materiais no estoque, para todos os insumos, e a dispensação ao paciente nos programas de diabetes e de estomia. | usuário |
| 2 | A inscrição no programa nasce na consulta, pelo médico ou pelo enfermeiro, no prontuário. | usuário |
| 3 | **O insumo é a quarta chave de item do estoque** (`supply`), dentro do domínio de medicamentos. Ele usa o central e as farmácias das unidades, e o molde é o 35a (`vector_products`). Entrada, transferência, inventário, perda e envio à BNAFAR valem para ele. Um estoque separado, como o do laboratório do 21c, ficou de fora: o ADR 0045 já descartou essa via, e ela duplicaria o envio à BNAFAR. | coordenador, apresentada ao usuário |
| 4 | **Catálogo de materiais da plataforma** (`supply_catalog_items`), carregado do CATMAT, classes 6515 e 6550, pelo mesmo cliente do catálogo de medicamentos. O item é descrito pelo PDM, e não por princípio ativo, com a unidade de fornecimento do CATMAT. | coordenador |
| 5 | **A inscrição no programa** (`supply_enrollments`) traz: programa (`diabetes` ou `ostomy`), paciente, CID, perfil (no diabetes: tipo de insulina e esquema), validade de 3 a 12 meses, unidade de retirada e quem inscreveu. Ela é renovada na consulta. | coordenador |
| 6 | **Cota por programa e por perfil, configurável pela cidade** (`supply_quotas`): item ou grupo de itens, quantidade por período, período em dias. Não há padrão nacional. A cidade parte de um modelo vazio e preenche pelo protocolo dela. | coordenador |
| 7 | **Dispensação por inscrição.** A dispensação do 27a passa a aceitar a inscrição como origem, no lugar da receita (receita **ou** inscrição). O item dispensado passa a aceitar o insumo. Acima da cota, a entrega é recusada. A exceção passa por aprovação do farmacêutico, com justificativa. | coordenador |
| 8 | **Inativação:** a inscrição sem retirada por 6 meses fica inativa, e a reativação é feita na consulta. | coordenador |
| 9 | **Comodato** (`device_loans`): aparelho (glicosímetro, leitor), número de série, paciente, termo aceito, data de entrega e de devolução, e troca contra o aparelho com defeito. O aparelho não é item de estoque. A tira compatível fica ligada ao modelo do aparelho, como informação para a farmácia. | coordenador |
| 10 | **Estomia no faturamento:** o item de bolsa tem o código SIGTAP da OPM. A entrega gera um procedimento realizado no 22, com CNS, CID e CBO, no BPA-I da competência. O teto mensal do SIGTAP (`qt_maxima`) é conferido. | coordenador |
| 11 | **Papéis:** o farmacêutico e o auxiliar entregam. Só o farmacêutico aprova exceção de cota. O médico e o enfermeiro inscrevem. O `municipal_admin` configura as cotas. | coordenador |
| 12 | **Inscrição sem consulta.** Além da consulta, o farmacêutico inscreve a partir de um laudo de fora, como a receita externa do 27a. Essa inscrição vale no máximo 3 meses e é renovada na consulta. Na entrada da cidade, os pacientes atuais são carregados por planilha (CSV), com área de conferência e aplicação com step-up, como o saldo inicial do 27b, pelo `municipal_admin` no dashboard. | usuário |
| 13 | **Bolsa de estomia no faturamento.** A regra do 22 não gera BPA de UBS (ADR 0039). Por isso a bolsa só é entregue nas unidades que a cidade marca como ponto de atenção à pessoa estomizada (não UBS, com CNES), e o faturamento sai por essa unidade. O CBO no BPA-I é o do profissional responsável do serviço, configurado na unidade. | coordenador |
| 14 | **Regra da cota, por perfil:** `fixed` (quantidade por período, como Campinas) ou `per_prescription` (medições ou aplicações por dia × dias do período, como São Paulo), com teto opcional. Quem prescreve pode fixar, na inscrição, uma cota acima da regra, com justificativa. A cota fica gravada na inscrição e só é recalculada na renovação. O período é contado da última retirada (padrão de 30 dias), sem acumular o que não foi retirado. | coordenador |
| 15 | **Exceção de cota na hora:** o farmacêutico confirma no balcão, com a própria sessão ou com step-up se quem atende é o auxiliar. Não há fila de pedidos. | coordenador |
| 16 | **Unidade de retirada** é só a preferida: o paciente retira em qualquer unidade, e a cota é somada entre elas. | coordenador |
| 17 | **Devolução:** tira, lanceta e seringa podem ser devolvidas, e a quantidade volta à cota. A bolsa de estomia não é devolvida depois de entregue. | coordenador |
| 18 | **A inscrição é registro estruturado** ligado à consulta, sem assinatura digital, como o plano odontológico. A farmácia só a vê depois da consulta finalizada. | coordenador |

## 3. Desenho

### 3.1 Estoque e catálogo

- **`stock_lots.item_kind`** ganha `supply`, com a chave `supply_catalog_item_id` e `catmat_code`.
- **CHECKs trocados:** os de lote, mínimo, abertura e movimento, no molde da migração do 35a.
- **`Stock::Domain.of_lot`:** o insumo pertence ao domínio `MEDICATION`, nos mesmos locais. O ramo de medicamento é reescrito para não tratar como medicamento todo lote que não é vacina nem vetorial.
- **Unidade de dispensação:** é a unidade de fornecimento do CATMAT do material.
- **Telas de estoque:** entradas, transferências e posição passam a buscar medicamento e insumo.
- **27b:** o insumo vai no envio com CATMAT, pelo interruptor de medicamentos.

### 3.2 Inscrição na consulta

- **Seção "Programas de insumos" na consulta (19):** inscrever, renovar e encerrar.
  - Diabetes: CID E10 a E14 ou O24, com a lista da cidade; tipo de insulina (NPH, regular, análogo, sem insulina); número de aplicações por dia; medições por dia prescritas.
  - Estomia: CID e tipo de estoma.
- **Inscrição pela farmácia:** a partir de um laudo de fora (decisão 12), com profissional, conselho e data.
- **Carga inicial:** os pacientes atuais entram por planilha, na entrada da cidade (decisão 12).
- **Cálculo da cota:** a inscrição calcula a cota pelo perfil e pelas regras da cidade, e mostra a cota ao profissional.
- **Prontuário:** a inscrição aparece no histórico e na linha do tempo do paciente (20).

### 3.3 Retirada na farmácia

- **Busca e cota:** a farmácia busca o paciente e vê as inscrições ativas, com a cota disponível no período.
- **Entrega:** sai por lote, como a dispensação do 27a.
  - O que passa da cota é recusado, com a frase da cota.
  - A exceção pede o farmacêutico e a justificativa.
- **Retirada por terceiro:** é registrada com o nome e o documento de quem retira, cifrados.
- **Paciente com glicosímetro em comodato:** a tela avisa qual tira é compatível.

### 3.4 Comodato

- **Registro do empréstimo:** aparelho, modelo, número de série, termo aceito e data. O aparelho fica ligado ao paciente.
- **Devolução, troca e perda:** a troca registra o aparelho com defeito e o novo. A perda pede o número do boletim de ocorrência, como texto.
- **Lista de aparelhos emprestados:** por unidade.

### 3.5 Estomia e BPA-I

- **Ao entregar uma bolsa,** nasce um `performed_procedure` do 22, com SIGTAP da OPM, CNS, CID, CBO de quem entregou, unidade e data.
  - A entrega sem CNS do paciente é recusada, com motivo.
- **Teto do SIGTAP:** quantidade acima do `qt_maxima` do mês é recusada.

## 4. Restrições

- Migração de cidade e de plataforma irreversíveis; a suíte completa do api é obrigatória.
- O nome e o documento de quem retira por terceiro ficam cifrados. Nenhum dado do paciente vai em log ou URL.
- O 27a não muda de comportamento para medicamento.

## 5. O que fica para o plano

- Os nomes de tabelas, rotas e eventos, no contrato entre apps.
- Os códigos SIGTAP das bolsas, conferidos na carga SIGTAP.
- A lista de CIDs de diabetes no padrão.
- A frase de cada recusa.

## 6. Gates de go-live

- As cotas conferidas com a assistência farmacêutica da cidade piloto.
- O teto das bolsas conferido no SIGTAP da competência.
- A carga do CATMAT de materiais conferida para os itens da cidade.
