# Módulo 26b — APAC, OCI, teto em dinheiro e AIH das eletivas — design

**Data:** 2026-11-25
**Status:** rascunho para aprovação do usuário

**Afeta:**
- **`apps/api`, cidade:**
  - laudo e autorização numerada (APAC e AIH);
  - faixas de numeração;
  - OCI com procedimentos secundários;
  - espelho da programação (FPO) e reserva de teto;
  - arquivo da APAC da rede própria;
  - fila nominal da OCI.
- **`apps/api`, plataforma:** a carga completa da SIGTAP (financiamento, valores SA/SH/SP, compatibilidades, atributos).
- **`apps/dashboard`:** pedido por laudo na consulta, autorização na fila, faixas, FPO, OCI, remessa da APAC.
- **`apps/maintenance`:** interruptores.

**Módulo:** `docs/modulos/26--regulacao.md` · **ADR base:** 0044 (26a); ADR novo do 26b na execução.
**Pesquisa:** `docs/pesquisa/2026-11-24-apac-oci-teto-mac-26b.md`, achados 1–18.

**Fora desta entrega:**
- O arquivo de AIH: o hospital fatura no SISAIH01 com o número autorizado.
- O arquivo de APAC dos prestadores contratados: eles faturam no APAC-Mag ou no sistema deles com o número autorizado.
- Escrita no SIA, no SIHD ou em sistema do estado.
- Editor de layout de arquivo, por decisão do usuário.

## 1. Intenção

A regulação municipal autoriza no Rota Saúde também o que sai por APAC ou por AIH, e entre isso estão as OCI e as cirurgias eletivas. Ela numera cada autorização na faixa recebida do estado, controla o teto em dinheiro de cada prestador e gera o arquivo da APAC da rede própria.

**Por quê:**
- Os editais pedem esse fluxo.
- O 26a só regula procedimento de BPA: no grupo 04, apenas 222 de 1.697 procedimentos entram.
- A OCI só é paga pelo FAEC se o município enviar a fila nominal todo mês.
- O teto na programação federal é só informativo, e o controle antes da autorização tem de ficar no sistema municipal.

**Sucesso:**
- O médico pede, na consulta, a APAC, a OCI ou a AIH com o laudo.
- O regulador autoriza na mesma fila do 26a.
- O sistema numera na faixa certa, sem repetir número.
- O sistema reserva o valor no teto do prestador e bloqueia o que passa dele.
- A rede própria recebe o arquivo da APAC da competência.
- A fila nominal da OCI sai até o 5º dia útil.

**Antiobjetivos:**
- Número de APAC ou AIH repetido.
- Autorizar acima do teto sem aviso.
- O próprio solicitante autorizar o pedido.
- Duas filas para a mesma regulação.

## 2. Decisões

| # | Decisão | De quem |
|---|---|---|
| 1 | Entram juntos APAC com OCI, teto em dinheiro e AIH das eletivas. | usuário |
| 2 | Sem editor de layout. O layout do arquivo de APAC é fixado no código a partir do documento oficial, conseguido na Task 0 da execução. | usuário |
| 3 | O Rota Saúde autoriza, numera e emite o laudo para todos os prestadores, e gera o arquivo da APAC da rede própria, como o BPA do 22. Os contratados faturam com o número autorizado. | usuário |
| 4 | **O pedido do 26a é o item da fila** (abordagem A). O laudo, a autorização numerada, os procedimentos da OCI e a reserva de teto ficam ligados a ele. Isso preserva o 26c, o TFD (30) e o cruzamento do Ministério. A abordagem B, um autorizador separado, ficou de fora: criaria duas filas. | usuário |
| 5 | **Teto em tabela própria, espelho da FPO** (abordagem A): competência × prestador × procedimento × financiamento (MAC, FAEC por programa, fora do teto), com valor e quantidade. O valor é reservado na autorização. A abordagem B, valor na oferta, ficou de fora porque não separa o financiamento. | usuário |
| 6 | **Alvo por laudo.** O pedido passa a aceitar procedimento cujo instrumento seja APAC (06) ou AIH (03 e 04), e as OCI (grupo 09), pelo segundo caminho de alvo da regra G11, que hoje é só um gancho. | coordenador |
| 7 | **Faixas de numeração** por tipo (5º dígito: APAC 2 geral, 7 OCI, 6 eletiva, 9 créditos; AIH 1 geral, 5 eletiva, 8 créditos), recebidas do estado e cadastradas pela cidade. O número sai na autorização, em sequência, com trava. O dígito verificador segue o algoritmo do documento oficial (Task 0). | coordenador |
| 8 | **Autorizador.** É o regulador com nível superior e vínculo público, e nunca o solicitante do pedido. O 26a não confere isso hoje e passa a conferir. | coordenador |
| 9 | **Momento da autorização, configurável por cidade:** antes da execução (modelo nacional e São Paulo) ou depois, com os secundários comprovados (Rio). O padrão é antes. | coordenador |
| 10 | **OCI.** É uma APAC única do tipo 3, com o principal do grupo 09, ao menos dois secundários de valor zero (um deles consulta ou teleconsulta) e validade de 2 competências, do primeiro ao último atendimento. Os secundários são registrados conforme acontecem. Se a OCI não for concluída, o que foi feito vai para o BPA-I. Na oncologia, o prazo é de 30 dias corridos. | coordenador |
| 11 | **Fila nominal da OCI.** O envio mensal sai pelo MIRA do 26c, se o Ministério confirmar esse canal. Até lá, sai em exportação no formato do e-SUS Captação de Filas, até o 5º dia útil. | coordenador |
| 12 | **AIH das eletivas.** Laudo de AIH na consulta, autorização na fila com conferência de vaga e de teto FAEC, e número da faixa de AIH. A modalidade 04 do MIRA passa a ser usada pelo 26c. | coordenador |
| 13 | **Laudo em PDF**, assinado com ICP-Brasil pelo signer do 19b quando o profissional tiver certificado. Sem certificado, sai impresso para assinatura. | coordenador |
| 14 | **Carga completa da SIGTAP:** financiamento, VL_SH e VL_SP, habilitações, compatibilidades entre principal e secundário e os atributos 051 a 058. É pré-requisito. | coordenador |
| 15 | **APAC de continuidade (tipos 1 e 2, tratamento de vários meses) fica para depois.** O 26b entrega a APAC Única, a OCI, o teto e a AIH. | usuário |
| 16 | **Competência da reserva:** a do mês previsto para o atendimento (na AIH, o da internação). Quando o atendimento real cai em outro mês, a reserva migra para ele. A regra do 26a de só aceitar oferta do mês corrente continua valendo para o BPA. | coordenador |
| 17 | **Vaga e teto, nesta ordem:** a vaga é a oferta em quantidade do 26a (`care_offers`), e a trava vem antes da linha da FPO. Na OCI, o alvo da oferta é o principal; na AIH, o procedimento. | coordenador |
| 18 | **Onde vale o teto:** em toda autorização para prestador `own`, `contracted` ou `consortium`, inclusive as de BPA, quando a cidade tem a FPO da competência cadastrada para aquele prestador. Prestador `ppi` e pedido `state_regulated` ficam fora da numeração e do teto, porque o gestor é outro. | coordenador |
| 19 | **Autorizar depois (modo Rio):** na liberação do acesso, o valor é reservado. O número só sai num segundo passo, "Autorizar a APAC", depois do atendimento comprovado. Se o regulador negar nesse segundo passo, a reserva é devolvida, e os procedimentos feitos viram BPA-I. É um estado novo na autorização, e não no pedido. | coordenador |
| 20 | **Fonte do procedimento realizado:** na rede própria, os procedimentos realizados do 22 (prontuário), ligados à autorização. No externo, a regulação marca na tela. O BPA passa a levar o número da autorização (hoje em branco), e o secundário de OCI ativa fica bloqueado no BPA-I, para não cobrar duas vezes. | coordenador |
| 21 | **Autorizador:** o nível superior sai do CBO (a tabela de `consultations/cbos.rb`), e o vínculo público é um vínculo ativo do profissional numa unidade própria. A regra vale para APAC e AIH. O BPA do 26a fica como está, para não travar os reguladores atuais. | coordenador |
| 22 | **Faixa, 5º dígito e financiamento:** a faixa é da cidade, por tipo. O 5º dígito sai do tipo do pedido. O financiamento é a linha da FPO que o regulador escolhe na autorização, com o padrão vindo do financiamento SIGTAP do procedimento; os créditos do programa federal levam o 9 ou o 8. A FPO aceita linha por procedimento ou por grupo, subgrupo ou forma de organização, e a reserva usa a linha mais específica. | coordenador |

## 3. Desenho

### 3.1 Pedido por laudo

- **Na consulta:** o pedido de regulação ganha o tipo `apac`, `oci` ou `aih`, com o laudo: justificativa clínica, CID principal e secundário, procedimentos solicitados e campos do tipo (na AIH, o caráter e a clínica).
- **Na fila:** o pedido entra na mesma fila, com a cor, e segue os prazos por cor do 26a.

### 3.2 Autorização numerada e teto

- **Ao autorizar,** o sistema:
  - confere o autorizador;
  - reserva o valor (SIGTAP mais complemento) no espelho da FPO do prestador para a competência e o financiamento;
  - tira o próximo número da faixa do tipo;
  - grava a autorização com a validade (1 a 3 competências; OCI com 2).
- **Teto:**
  - sem saldo no teto, a autorização é recusada, com a frase;
  - o regulador com papel pode autorizar acima com justificativa, e isso fica marcado;
  - o cancelamento devolve a reserva.
- **Espelho da FPO:** cadastrado pela cidade por competência, com importação de planilha. Os créditos do programa federal que não oneram o teto ficam como financiamento à parte.

### 3.3 OCI

- **O pedido de OCI** traz o principal (grupo 09) e os secundários previstos.
- **Andamento:** cada secundário é marcado como realizado, com a data, pela unidade executante ou pela regulação.
- **Encerramento:** a OCI se encerra no último atendimento, com motivo de saída da lista restrita. Sem conclusão, os secundários feitos viram BPA-I.
- **Prazo:** o alarme de prazo mostra a OCI de oncologia perto dos 30 dias.

### 3.4 Arquivo da APAC da rede própria

- **Remessa:** gerada por competência e por unidade própria, no layout fixado (Task 0), com versão e SHA-256, no molde do `BillingExport` do 22.
- **Conferência:** o VERSIA confere o arquivo antes do uso real.

### 3.5 Fila nominal da OCI

- **Envio:** até o 5º dia útil de cada mês, pelo canal da decisão 11.
- **Na tela:** a lista do mês e a situação do envio.

### 3.6 Telas

- **Consulta:** o pedido por laudo.
- **Fila:** a autorização com o número, a reserva e o aviso de teto.
- **Regulação, configurações:**
  - faixas de numeração;
  - espelho da FPO;
  - momento da autorização;
  - OCI em andamento;
  - remessa da APAC;
  - fila nominal.

## 4. Restrições

- Migrações de cidade e de plataforma irreversíveis; a suíte completa do api é obrigatória.
- O laudo traz dado clínico: fica cifrado e nunca vai em log, URL ou evento.
- O número de APAC ou AIH nunca se repete na cidade, nem entre faixas.

## 5. O que fica para o plano

- A Task 0 consegue o layout do arquivo de APAC (FTP do SIA), o algoritmo do dígito verificador e a SIGTAP completa. Sem eles, a execução para.
- Os nomes de tabelas, rotas e eventos, no contrato entre apps.
- A lista de motivos de saída da OCI e os campos do laudo de AIH.

## 6. Gates de go-live

- As faixas de numeração recebidas da SES da cidade piloto.
- A FPO da competência cadastrada e conferida.
- O arquivo da APAC aprovado no VERSIA.
- O canal da fila nominal confirmado com o Ministério.
- A habilitação 38.01 e a 29.02 dos prestadores conferidas no CNES.
