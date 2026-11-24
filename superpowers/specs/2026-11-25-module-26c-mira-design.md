# Módulo 26c — Envio da regulação à RNDS (MIRA) — design

**Data:** 2026-11-25
**Status:** rascunho para aprovação do usuário

**Afeta:**
- **`apps/api`, cidade:**
  - estado de sincronização por pedido;
  - credencial própria do modelo de regulação;
  - configurações da regulação (conversão das cores, de-para de especialidade, envio dos externos);
  - CBO do executante na oferta;
  - envio contínuo com alarme D-1;
  - simulado do fake-rnds com os cenários de regulação.
- **`apps/dashboard`:**
  - tela de situação e pendências do envio, no grupo da regulação;
  - configurações da regulação;
  - credencial nas Integrações.
- **`apps/maintenance`:** o interruptor.

**Módulo:** `docs/modulos/26--regulacao.md` · **ADRs base:** 0044 (26a) e 0047 (RNDS da vacina); ADR novo do 26c na execução.
**Pesquisa:** `docs/pesquisa/2026-11-23-mira-regulacao-rnds-26c.md`, achados 1–15.

**Fora desta entrega:**
- APAC, OCI e AIH (26b). Quando o 26b trouxer a AIH, a modalidade hospitalar (04) entra aqui.
- O envio da fila nominal da OCI ao Ministério, que é do 26b.
- Escrita no SISREG ou em sistema do estado.
- A correção do identificador do solicitante no envio da vacina (29): vira card ligado a este.

## 1. Intenção

A cidade que regula no Rota Saúde cumpre o envio da regulação à RNDS sem trabalho manual, e vê o que não foi aceito e por quê.

**Por quê.**
- **Envio diário:** a Portaria GM/MS 6.656/2025 obriga o ente com sistema próprio a enviar todo dia, com os dados até o dia anterior.
- **Prazo:** o Plano Operativo da PNR-SUS (Portaria 9.262/2025) pede o envio integral, de preferência síncrono, até dez/2026.
- **Sanção:** a possível perda de acesso aos programas de cirurgias eletivas.

**Sucesso:**
- Cada pedido regulado no Rota Saúde chega à RNDS com a situação certa, em poucos minutos depois de cada mudança, e nunca depois do fim do dia seguinte.
- A cidade vê os pedidos aceitos, recusados e parados, com o motivo, e corrige o dado de origem.
- O pedido corrigido volta a andar sozinho.

**Antiobjetivos:**
- Duplicar no MIRA um pedido que o estado já envia.
- Mandar documento que a RNDS vai recusar, sem antes avisar a cidade.
- Quebrar a cadeia de substituição de um pedido por envio fora de ordem.
- CPF, CNS, CID ou justificativa do paciente em log, URL ou evento.

## 2. Decisões

| # | Decisão | De quem |
|---|---|---|
| 1 | Vão à RNDS só os pedidos que a cidade regula no Rota Saúde. Os pedidos `external` ficam fora. A configuração "Enviar também os pedidos regulados pelo estado" fica pronta, desligada por padrão, para a cidade que pactuar o envio com o estado. | usuário |
| 2 | **Sincronizar o estado de cada pedido** (abordagem A). Uma tabela guarda, por pedido, o identificador local, o último documento aceito, a situação enviada e a situação desejada. Um job por minuto envia o próximo documento, em ordem por pedido, só depois do aceite do anterior. As abordagens B (um envio por mudança) e C (lote diário) ficaram de fora: a B quebra a regra de substituir uma vez só quando duas mudanças chegam juntas, e a C deixa o pedido até um dia defasado. | usuário |
| 3 | **Conferência leve no envio** (um esqueleto em Ruby, como o `Rnds::Validation` da vacina) e **validação oficial nos testes**: a CI roda o validador do Ministério sobre documentos de exemplo gerados pelo próprio código, e o resultado serve de evidência na homologação. A validação oficial em cada envio ficou de fora porque põe Java em produção. | usuário |
| 4 | Correspondência das situações do 26a com o MIRA: pendente e autorizado → `pending`; devolvido → `returned-to-requester`; agendado → `booked`; realizado → `attended`; faltou → `absence`; negado, cancelado e encerrado → `cancelled`. O rascunho não vai. | coordenador, apresentada ao usuário |
| 5 | Cor de risco → caráter do MIRA: vermelho → `urgent`; amarelo, verde e azul → `routine`. A conversão é configurável por cidade. | coordenador, apresentada ao usuário |
| 6 | **Credencial própria** `rnds_regulation`. O DATASUS libera a credencial por modelo de informação, e o identificador do solicitante (`BRRNDS-{id}`) é por credencial. Pode ser o mesmo certificado A1 da vacina, mas a credencial e o identificador são outros. Ela guarda o PFX, a senha, o identificador do solicitante e o CNS do responsável pelo envio. | coordenador |
| 7 | O header `Authorization` leva o **CNS do responsável pelo envio**, um profissional com CBO autorizado num dos CNES da credencial, informado na credencial. A homologação confirma se a RNDS aceita assim. | coordenador |
| 8 | **Pedido por especialidade.** O MIRA só aceita procedimento no `ServiceRequest.code`. O pedido por CBO vai com o código SIGTAP da consulta especializada que a cidade configura por especialidade. O padrão é 03.01.01.007-2 (consulta médica em atenção especializada), e o CBO segue em `performerType`. | coordenador |
| 9 | **CBO do executante** (regra RN03 nos grupos 03 e 04). Ele entra na oferta do prestador (`care_offers.performer_cbo`), opcional no 26a e exigido para enviar pedido desses grupos. Sem ele, o pedido fica parado, com motivo. | coordenador |
| 10 | A **modalidade** fica fixa em 09 (ambulatorial), porque o 26a só regula procedimento com BPA. O 26b traz a 04. | coordenador |
| 12 | **Falta.** A primeira falta gera dois documentos, em ordem: `absence` e depois `pending`. No 26a o pedido volta a `pending` na hora, mas o MIRA precisa registrar a falta. A segunda falta (`closed` com desfecho `missed`) vai como `absence`, final. Por isso o pedido guarda uma pequena lista de passos pendentes, montada a partir do andamento (`regulation_request_events`) desde o último aceite. Dos passos intermediários, só a falta é preservada; os outros se juntam no último (§3.1). | coordenador |
| 13 | **Pedido que sai da regulação municipal** (marcado externo, ou a configuração dos externos desligada depois de enviar) recebe uma substituição com `excluded`, e a cadeia para. Com a configuração dos externos ligada, o `external` acompanha o andamento: espera → `pending`, agendado → `booked`, realizado → `attended`. O `DELETE` não é usado em nenhum fluxo do produto, só na reconciliação de um documento duplicado, achado pelo `GET /identifier`. | coordenador |
| 14 | **CBO do executante (RN03)** só é exigido quando há executante (`booked` ou `attended`) num procedimento dos grupos 03 ou 04. A fonte é, nesta ordem: o profissional da agenda (rede própria, pelo 17); o `performer_cbo` da oferta; o CBO padrão do prestador (`care_providers.default_performer_cbo`, novo). Sem nenhum, o pedido fica parado, com motivo. O documento `pending` vai sem executante. | coordenador |
| 15 | **Carga inicial ao ligar.** Ganham linha e são enviados, na situação atual, todos os pedidos não terminais, mais os terminais encerrados nos últimos 30 dias. O ritmo é limitado (padrão de 60 por minuto, configurável). Os pedidos da carga ficam fora do alarme de atraso nas primeiras 72 horas. Com o interruptor desligado ou sem credencial, as linhas são mantidas e calculadas, mas nada é enviado. | coordenador |
| 16 | **Retificação.** Uma mudança de conteúdo num documento já aceito, sem troca de situação, gera substituição. Entram nessa conta a remarcação, a troca de prestador, a troca de cor, o CID, a identificação do paciente, o CNES e a mudança das configurações de conversão. A linha guarda uma impressão digital do conteúdo enviado e compara com a atual. Os ganchos vêm dos eventos do pedido, da oferta, do prestador, da unidade e das configurações. | coordenador |
| 17 | **Paciente apagado por LGPD** (`Citizens::Erase`, ADR 0026). A cadeia do pedido para em silêncio, na situação `erased`, fora do alarme. O que já foi enviado fica na RNDS como registro do SUS, como a dispensação fica retida no 27a. Nada novo é enviado. | coordenador |
| 18 | **Datas.** `scheduled_at` vai no `Appointment.start`, e `requested_at` no `ServiceRequest.authoredOn`, como no exemplo oficial do guia. A correspondência fica num parâmetro do perfil. Como o guia rotula esses campos de forma trocada, a confirmação com o suporte da RNDS é gate de go-live. | coordenador |
| 11 | **Interruptor** `rnds_regulation`, que exige `feature:care_regulation` e `credential:rnds_regulation`. O simulado `rnds_mock` passa a valer para os dois envios, vacina e regulação, sem depender do `rnds_immunization`. | coordenador |

## 3. Desenho

### 3.1 Estado por pedido

- **Tabela da cidade `regulation_rnds_syncs`**, 1:1 com `regulation_requests`:
  - `bundle_identifier`, o identificador local, estável e gravado na primeira vez;
  - `last_rnds_id`, o ID RNDS do último documento aceito;
  - `sent_status` e `desired_status`, nas situações do MIRA;
  - `status`: `idle` | `queued` | `sending` | `blocked` | `rejected` | `failed`;
  - `blocked_reasons` e `last_error_codes` (só códigos);
  - `attempts`, `next_attempt_at`, `last_accepted_at`.
- **`desired_status` e passos pendentes:** são recalculados a cada evento do pedido (e da oferta, do prestador, da unidade e das configurações, decisão 16), pelo gancho que já existe. A linha vira `queued` quando a situação ou a impressão digital do conteúdo diferem do último aceite. Os passos pendentes preservam a falta (decisão 12).
- **Primeiro envio:** é um documento novo.
- **Mudanças seguintes:** cada uma vai como substituição (`relatesTo replaces` do `last_rnds_id`). A RNDS só aceita substituir um documento uma vez, então o próximo envio sempre parte do último aceito.
- **Ordem:** um pedido nunca tem dois envios ao mesmo tempo. O claim é por linha, com `SKIP LOCKED`, como na fila da vacina.
- **Pulo de situação:** quando a situação muda duas vezes antes do envio, vai só a última, salvo a falta (decisão 12).
- **Timeout sem resposta:** o pedido não é reenviado no escuro. Primeiro, uma consulta `GET /identifier` descobre se o documento já foi aceito. Assim, nada vira duplicado.

### 3.2 O documento

- **Bundle RIRA `document`**, com Composition `BRRegulacaoAssistencial`, Appointment, ServiceRequest e Condition. As URLs canônicas são as do guia oficial (`br.gov.saude.rira.fhir#1.0.0-release`), confirmadas.
- **Paciente:** entra por identifier (CPF ou CNS).
- **Estabelecimentos:** entram por CNES. A unidade solicitante é sempre informada. O executante é o prestador da oferta, ou a unidade própria.
- **Appointment:** vai sempre. No `pending`, vai como `proposed`.
- **Motivo:** CID-10 ou CIAP-2 do pedido.
- **Conversões:** a cor vira o caráter pela configuração da cidade, e a especialidade vira o SIGTAP pela configuração (decisão 8).

### 3.3 Conferência e pendências

- **Pedido parado (`blocked`), com motivo**, e não enviado, quando:
  - falta CID/CIAP;
  - a unidade ou o prestador está sem CNES;
  - o paciente está sem CPF e sem CNS (`unidentified`);
  - o procedimento é dos grupos 03 ou 04 sem CBO do executante;
  - a especialidade não tem SIGTAP configurado;
  - o esqueleto acusa campo faltando.
- **Corrigido o dado de origem**, a linha volta a `queued` sozinha, na verificação periódica de parados (como o `BLOCKED_RECHECK` da vacina).
- **Recusa da RNDS (422):** o pedido fica `rejected`, com os códigos. Ele volta quando o dado de origem mudar.
- **Erro transitório:** backoff, como na fila da vacina.

### 3.4 Alarme

`Regulation::RndsAlert`, calculado na leitura, no molde do `Rnds::Alert`, conta:
- os pedidos `blocked` e `rejected`;
- os atrasados: pedido cuja situação mudou e não foi aceita até as 23:00 do dia seguinte no fuso da cidade.

O painel da regulação e a tela de envio mostram a faixa quando há atraso.

### 3.5 Telas

- **Dashboard, grupo da regulação:**
  - "Envio à RNDS": totais aceitos, parados, recusados e atrasados, mais a lista com o motivo e o atalho para corrigir. Mostra o pedido sem nome nem CPF, pelo número do pedido.
  - "Configurações da regulação": a conversão cor → caráter, o SIGTAP por especialidade e o envio dos externos.
  - A oferta do prestador ganha o campo "CBO do executante".
- **Dashboard, Integrações:** a credencial `rnds_regulation` (PFX, senha, identificador do solicitante, CNS do responsável) com teste de conexão, pelo `municipal_admin`, com step-up, como a da vacina.
- **Maintenance:** o interruptor `rnds_regulation` e o aviso do simulado.

### 3.6 Testes e simulado

- **O fake-rnds ganha os cenários da regulação:**
  - aceite com `Location`;
  - recusa 422;
  - substituição repetida (EDS-MSG009);
  - `DELETE`;
  - `GET /identifier`.
  - O marcador de falha vai no `Bundle.identifier.value`, porque o `ServiceRequest.identifier` é 0..0 no perfil (achado 14).
- **CI:** um job roda o validador oficial do Ministério sobre os documentos de exemplo, um por situação e com as duas conversões.

## 4. Restrições

- Migração de cidade irreversível; a suíte completa do api é obrigatória.
- O documento vai só no corpo: o CPF, o CNS e o CID nunca entram em log, URL ou evento. O payload guardado é cifrado.
- O servidor de produção precisa de IP de saída fixo, porque a faixa de IPs é declarada no credenciamento (achado 15). É gate de infraestrutura.
- O envio da vacina (29) não muda nesta entrega.

## 5. O que fica para o plano

- Os nomes de tabelas, rotas e eventos, no contrato entre apps.
- O intervalo da verificação de parados e os limites de backoff.
- O formato dos documentos de exemplo da CI.
- O texto das frases de pendência.

## 6. Gates de go-live

- Credencial `rnds_regulation` de homologação e de produção, pedida pelo gestor no Portal de Serviços. O prazo de análise do DATASUS é desconhecido, então o pedido começa antes do código.
- Homologação com a evidência do validador.
- O header `Authorization` com o CNS do responsável, confirmado na homologação.
- IP de saída fixo.
- A conversão de cores e o SIGTAP por especialidade conferidos com a regulação da cidade piloto.
- A confirmação de que a cidade piloto não envia os mesmos pedidos por outro sistema.
