# Módulo 27b — Envio à BNAFAR e saldo inicial vindo do Hórus — design

**Data:** 2026-11-24
**Status:** rascunho para aprovação do usuário

**Afeta:**
- `apps/api`, migração de cidade:
  - campos novos no lote e no local;
  - saídas para fora da rede;
  - envio diário ao SI-BNAFAR, com protocolo e registros enviados;
  - credencial `bnafar`;
  - área de conferência da importação.
- `apps/api`, plataforma:
  - VMP no catálogo de medicamentos;
  - código OBM das vacinas no catálogo de imunobiológicos.
- `apps/api`, cidade: CATMAT nos produtos de endemias e zoonoses (`vector_products`).
- `apps/dashboard`:
  - os campos novos na entrada e no inventário de abertura;
  - complemento dos lotes antigos;
  - saída para fora da rede;
  - situação do envio e pendências.
- `apps/maintenance`:
  - interruptor `bnafar`;
  - importação da planilha com conferência.
- A credencial é gravada no dashboard pelo `municipal_admin`, como as outras (ADR 0028).

**Módulo:** `docs/modulos/27--farmacia-e-almoxarifado.md` · **ADR base:** 0045 (27a); ADR novo do 27b na execução.

**Pesquisas:**
- `docs/pesquisa/2026-11-23-insumos-e-bnafar-27.md` (normas, campos, lacunas do 27a);
- `docs/pesquisa/2026-11-24-horus-e-si-bnafar-27b.md` (API v1.25, base do Hórus, achados 1–16).

**Fora desta entrega:**
- **Histórico de dispensação vindo do Hórus.** O Ministério ainda não publicou o formato da base. Vira card e tem desenho próprio quando o formato sair.
- **Dispensação ao paciente pela RNDS (REDFM).** O modelo ainda não foi publicado. Vira card.
- **Insumos do 27.** O catálogo de materiais é a entrega seguinte. Quando ele chegar, os materiais entram no envio com CATMAT. Os produtos de endemias e zoonoses que já existem entram agora (decisão 8).
- Entradas e avaliação do CEAF, que o SI-BNAFAR não recebe mais.
- Envio retroativo, que o SI-BNAFAR recusa.
- Compras e empenho.

## 1. Intenção

A cidade que usa a farmácia do Rota Saúde cumpre a BNAFAR sem trabalho manual. A cidade que sai do Hórus abre o estoque no Rota Saúde sem refazer o inventário.

**Por quê.**
- **Envio obrigatório:** o envio diário à BNAFAR é obrigatório para todos os municípios (Portaria GM/MS 5.713/2024). O prazo de adequação venceu em 17/06/2025. A sanção é suspender os incentivos ligados ao envio.
- **Saída do Hórus:** o Hórus para de aceitar lançamentos por volta de dez/2026 (Portaria 11.585/2026). Quem sai dele precisa de um sistema que envie à BNAFAR.
- **Ponto de partida comum:** o e-SUS AF também não migra nada. Ele manda fazer inventário físico e lançar o "saldo de implantação".

**Sucesso:**
- Todo dia a cidade envia ao SI-BNAFAR a posição de estoque de cada lote, em cada local com CNES, e as saídas do dia anterior que não são dispensação.
- A equipe vê o que foi aceito, o que foi recusado e por quê, e corrige dentro dos 30 dias.
- Uma cidade vinda do Hórus entra com o saldo de abertura carregado de uma planilha, conferido antes de valer.

**Antiobjetivos:**
- Perder um dia de envio sem aviso. A janela é D-1, e o atraso só volta por exceção.
- Mandar dispensação como saída.
- Mandar quantidade fracionária ou dado incompleto e descobrir só na recusa.
- A importação mexer no estoque antes de conferida.
- Senha, token ou dado de paciente em log ou URL.
- Prometer no catálogo público o histórico do Hórus antes de existir.

## 2. Decisões

| # | Decisão | De quem |
|---|---|---|
| 1 | Ordem das entregas restantes do 26 e do 27: 27b → 26c → insumos → 26b. O 34 fica por último. | usuário |
| 2 | A importação do Hórus traz o saldo inicial agora e o histórico de dispensação por paciente depois. | usuário |
| 3 | Paciente do histórico fora do cadastro é criado pela busca de CPF/CNS (o caminho da farmácia do 27a), e o que não casa vai a um relatório de pendências. Vale para a entrega do histórico. | usuário |
| 4 | **Envio em lote diário** (abordagem A): um job por cidade monta as remessas de D-1 a partir do razão do estoque, guarda o que foi enviado e acompanha o protocolo. A abordagem B, com saída na hora e posição diária, ficou de fora: são dois mecanismos e a retificação fica mais difícil. | usuário |
| 5 | **Importação por arquivo com área de conferência** (abordagem A), com dois leitores: a planilha padrão do Rota Saúde agora e a base oficial do Ministério depois. A abordagem B, de busca direta numa API do Ministério, ficou de fora porque essa API não existe hoje. | usuário |
| 6 | A importação é feita pela equipe da plataforma, no maintenance, uma vez por cidade, e a secretaria confere o resultado no dashboard. | coordenador |
| 7 | O REDFM fica fora, em card. | coordenador |
| 8 | **O envio cobre tudo o que tem estoque:** medicamentos (27a), vacinas e rede de frio (29) e produtos de endemias e zoonoses (35a e 35b). O mecanismo e as migrações já servem a todos. Cada tipo é ligado à parte, no maintenance, pelos interruptores da cidade. A ordem prevista é: medicamentos; depois os produtos de endemias e zoonoses; por último as vacinas, quando a homologação confirmar o código e os tipos de saída. | usuário (escopo e interruptores no maintenance); coordenador (ordem) |
| 9 | **Os códigos de produto da BNAFAR:**<br>- o VMP fica no catálogo da plataforma;<br>- o AMPP, que é a apresentação comercial de um fabricante e se confere contra o registro ANVISA, vai em cada entrada de lote, escolhido entre as opções do VMP;<br>- vacinas ganham o código OBM no catálogo de imunobiológicos;<br>- produtos de endemias e zoonoses ganham CATMAT. | coordenador |
| 10 | **Correções e recusas:**<br>- a posição de estoque é uma fotografia por dia, e corrigir um lote vale para as próximas, sem PUT das passadas;<br>- o PUT, dentro de 30 dias, vale só para a saída externa, quando o destino ou o motivo forem corrigidos;<br>- o registro recusado é corrigido e reenviado no mesmo dia (o alerta existe para isso);<br>- passada a janela, fica como perdido, com o motivo, e entra no relatório. | coordenador |
| 11 | **Tipo de produto (DM13)** derivado do programa de saúde (DM14) por uma tabela de regras no código, sem campo digitado. | coordenador |
| 12 | **Credencial** gravada pelo `municipal_admin` no dashboard, com step-up, como as outras credenciais (ADR 0028). A equipe da plataforma não manuseia a senha da cidade. | coordenador |
| 13 | **Homologação** feita uma vez pelo Rota Saúde, numa cidade de staging, com credencial de homologação. A evidência gerada serve para cada cidade pedir a sua credencial de produção. A credencial passa a ter ambiente (`homologation` \| `production`), um de cada por `kind`. | coordenador |
| 14 | **A conferência do almoxarife não bloqueia.** O envio começa no dia seguinte à aplicação da planilha, e o desfazer só vale antes do primeiro envio do local. | coordenador |

## 3. O que o SI-BNAFAR pede (resumo da pesquisa)

O detalhe e as fontes estão na pesquisa de 2026-11-24, que segue o manual v1.25 (03/06/2026), o Swagger e a coleção Postman.

- **Dois assuntos:**
  - **Posição de Estoque:** uma fotografia por estabelecimento, com itens por produto e lote.
  - **Saída:** transferência e perda, com o motivo (domínio DM12, 21 tipos) e o destino, por CNES ou por CNPJ.
- **Modo de envio:** síncrono ou em lote, este último assíncrono, com protocolo.
- **Autenticação:** `POST /jwtauth/auth` com HTTP Basic (usuário e senha do Portal de Serviços), que devolve um JWT. As chamadas levam `Authorization: jwt <token>`. O tempo de vida vem em `expires_in`, a confirmar em homologação.
- **Data:** só hoje ou ontem (MSG73). Reenvio é proibido.
- **Correção:** PUT ou DELETE até 30 dias depois do envio. O PUT substitui o registro inteiro.
- **Limites:** até 1.000 registros por lote e até 60 itens por registro. A MSG46 diz 20; conferir em homologação.
- **Campos por item:**
  - produto: AMPP/VMP para medicamento, CATMAT para insumo;
  - CNPJ do fabricante e registro ANVISA;
  - lote, validade, valor unitário (2 casas, > 0), quantidade inteira na menor unidade (até 8 dígitos);
  - programa de saúde (DM14) e IUM.
- **Ente:** IBGE com 6 dígitos. Os CNES precisam pertencer a esse IBGE (MSG51).
- **Divergências:** manual, Swagger e Postman divergem em 9 pontos. O validador local usa a união das regras (achado 12), e a homologação decide o resto.

## 4. Desenho

### 4.1 Dados do lote e do local

- **Lote ou entrada, campos novos:** valor unitário, CNPJ do fabricante, registro ANVISA, programa de saúde (DM14) e IUM, quando houver.
  - Com o envio ligado, a entrada e o inventário de abertura recusam linha sem esses campos, com `path`.
  - O lote antigo, de antes do 27b, fica com eles vazios.
- **Complemento:** uma tela de complemento lista os lotes com saldo e campo faltando.
  - Enquanto um lote estiver incompleto, o item dele fica fora da posição de estoque, como pendência com motivo. O resto do estabelecimento é enviado.
- **Local:**
  - Cada local de estoque ganha um tipo DM01. `central` → A e `unit` → F são diretos. `cold_chain_central`, `vaccine_room` e `vector_control` são mapeados no plano contra o DM01, com spec (achado 9).
  - Cada local precisa de CNES do ente. O almoxarifado central e a rede de frio central, que hoje não têm unidade, ganham um campo de CNES próprio.
  - Local sem CNES fica fora do envio, como pendência.
- **Quantidade:** é inteira na menor unidade de fornecimento. Saldo fracionário (frasco aberto) fica fora da posição, como pendência "saldo fracionário", até ser ajustado. O razão continua com 2 casas.
- **IBGE:** sai do perfil da cidade sem o dígito verificador. Não há dado novo.

### 4.2 Códigos de produto

- **Medicamento:**
  - O VMP fica no catálogo da plataforma, carregado do portal OBM do Ministério, que tem API, e substitui a coluna `obm_code` reservada.
  - O AMPP vai na entrada do lote, escolhido entre as apresentações do VMP, e é conferido contra o registro ANVISA do lote (MSG72).
- **Vacina:** o catálogo de imunobiológicos (YAML do 29) ganha o código OBM de cada vacina.
- **Produtos de endemias e zoonoses:** `vector_products` ganha o CATMAT, informado pela cidade no cadastro do produto.
- **Sem código:** item sem o código do seu tipo fica fora do envio, como pendência de catálogo.

### 4.3 Saídas para fora da rede

O 27a só transfere entre locais da cidade. O 27b acrescenta uma **saída externa**, um movimento negativo novo, com destino por CNES ou CNPJ e motivo do DM12. Os motivos cobertos são os que a farmácia municipal usa:
- devolução ao fornecedor;
- remanejamento a outro ente;
- doação;
- empréstimo.

A tabela de correspondência entre os movimentos do estoque e o DM12 fica no código, com spec:
- perda → S-PE;
- vencimento → S-VV;
- ajuste negativo → S-AE;
- transferência → S-T.
- Os movimentos de uso direto são mapeados no plano contra os 21 tipos do DM12: administração na urgência (32), abertura de frasco e dose aplicada (29), aplicação de produto vetorial (35a), dose antirrábica e coleira (35b). Um movimento sem tipo correspondente fica fora, registrado como "sem tipo BNAFAR", e isso vai para a lista de lacunas.

Dispensação não entra. A dispensação a paciente sem identificação (S-PA) fica fora até a homologação confirmar (achado 15).

### 4.4 Envio diário

- **Interruptores** por cidade, no maintenance, no mesmo mecanismo do `Platform::Features`: cada um tem pré-requisito, auditoria e a frase de pré-requisito faltando.
  - `bnafar`: liga o envio. Exige `credential:bnafar`.
  - `bnafar_medications`: medicamentos. Exige `feature:bnafar` e `feature:pharmacy`.
  - `bnafar_vector_supplies`: produtos de endemias e zoonoses. Exige `feature:bnafar` e o domínio vetorial utilizável (`vector_control` ou `zoonoses`).
  - `bnafar_vaccines`: vacinas e rede de frio. Exige `feature:bnafar` e `feature:vaccination`.
  - O job só monta registros dos tipos ligados. Item de tipo desligado aparece nas pendências como "tipo não ligado ao envio". Ligar um tipo vale a partir do dia seguinte, porque a BNAFAR não aceita data retroativa.
- **Credencial:**
  - `IntegrationCredential` ganha o `kind` `bnafar`, com usuário e senha do Portal de Serviços, cifrados, e o ambiente (decisão 13). É migração de cidade.
  - Quem grava é o `municipal_admin`, no dashboard, com step-up (decisão 12).
  - A conexão se testa como as outras credenciais.
- **Job:** um por cidade, todo dia, logo depois da meia-noite local. Ele monta, para D-1:
  - a posição de estoque de cada estabelecimento com CNES, um registro por estabelecimento e item (achado 5);
  - as saídas do dia que não são dispensação.
- **Envio:** em lote de até 1.000 registros. O job guarda o protocolo, consulta o processamento até terminar, lê as inconsistências e grava o `codigoBnafar` de cada registro aceito.
- **Tabelas novas:**
  - remessa: cidade, dia, protocolo, situação;
  - registro enviado: origem, tipo, payload cifrado guardado por 30 dias, `codigoBnafar`, situação, erros.
  - O padrão de claim e backoff da fila da RNDS se reaproveita. A tabela não.
- **Correção:** segue a decisão 10. Cada saída guarda em qual registro foi enviada.
- **Alerta:** se a remessa de D-1 não estiver aceita até o meio da tarde do dia seguinte, a equipe da plataforma e a secretaria recebem aviso, porque depois da meia-noite o dia só volta por exceção.
- **Validação local antes de enviar:** item sem campo, local sem CNES, saldo fracionário ou produto sem código ficam como pendência com motivo e não vão.
- **Homologação:** segue a decisão 13. A cidade de staging usa a massa de dados do Ministério para CNES e produtos. Cada chamada fica registrada em cURL, sem token nem senha, para a evidência da credencial de produção (achado 14).

### 4.5 Saldo inicial pela planilha

- **Planilha padrão:** o Rota Saúde publica uma planilha (CSV) com as colunas do inventário de abertura: local (CNES), produto (CATMAT e AMPP), lote, validade, quantidade, valor unitário, CNPJ do fabricante, registro ANVISA, programa.
  - A cidade preenche com o relatório final de posição do Hórus ou com o inventário físico.
- **Conferência no maintenance:** a equipe da plataforma envia o arquivo. O Rota Saúde guarda as linhas numa área de conferência, sem tocar no estoque, e mostra um relatório:
  - linhas válidas;
  - linhas com erro, com o motivo;
  - produto que não casa com o catálogo;
  - local sem CNES;
  - totais por local.
- **Aplicação:** confirmada pela equipe da plataforma com step-up, a importação vira entradas `opening` (o tipo já existe no 27a), uma por local, numa transação.
  - A importação é aplicada uma vez por local. Repetir exige desfazer antes, e o desfazer só vale enquanto o local não tiver nenhum outro movimento.
- **Conferência da secretaria:** o almoxarife confere o saldo de abertura no dashboard. A partir daí, o envio diário começa no dia seguinte.
- **Leitor do Ministério:** a área de conferência é genérica, com o leitor separado. O leitor da base oficial entra quando o formato for publicado, na entrega do histórico.

### 4.6 Telas

- **Dashboard, grupo Farmácia:**
  - "Envio à BNAFAR": situação de cada dia, aceito, recusado ou pendente, com os motivos e o prazo de correção;
  - "Pendências do envio": lotes incompletos, locais sem CNES, produtos sem código, saldos fracionários;
  - "Complemento de lotes";
  - "Saída externa";
  - os campos novos na entrada e no inventário.
- **Maintenance:**
  - interruptor;
  - "Importar saldo inicial", com o relatório de conferência e a aplicação.
- **Dashboard, Integrações:** credencial `bnafar` com teste de conexão, pelo `municipal_admin`.

## 5. Restrições

- Migrações de cidade irreversíveis; a suíte completa do api é obrigatória.
- Usuário e senha da BNAFAR só cifrados, nunca em log nem em resposta. O token fica só em memória.
- A planilha importada é cifrada em repouso na área de conferência e apagada depois de aplicada ou descartada.
- O catálogo público não promete histórico do Hórus nem REDFM.
- A cidade com e-SUS AF ou com o Hórus não liga o `bnafar` no Rota Saúde. A regra fica no runbook: um envio só por ente.

## 6. O que fica para o plano

- Tipo DM01 dos locais `cold_chain_central`, `vaccine_room` e `vector_control`, e o DM12 dos movimentos de uso direto (§4.3).
- A tabela DM13 por programa (decisão 11).
- O horário exato do job e do alerta.
- O formato do registro de cURL.
- A conversão de embalagem para a menor unidade de fornecimento, quando a unidade de dispensação local for diferente dela.
- O teto de 60 ou 20 itens por registro.
- O tempo de vida do token. Os dois se confirmam na homologação.
- Os nomes de tabelas, rotas e eventos, no contrato entre apps.

## 7. Gates de go-live

- Credencial de homologação e de produção do SI-BNAFAR, pedida pelo gestor no Portal de Serviços.
- Homologação com a massa de dados do Ministério.
- Carga do VMP, do OBM das vacinas e do CATMAT dos produtos de endemias conferida para a cidade piloto.
- CNES do almoxarifado central.
- Lotes de abertura completos.
