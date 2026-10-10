# Rollout — documentos clínicos (módulo 19c)

Runbook do operador para publicar o módulo 19c (ADR 0033), carregar o catálogo
de medicamentos, ligar os documentos clínicos numa cidade e operar a página
pública de conferência. Bases:
[`rollout-prontuario.md`](rollout-prontuario.md) (19a) e
[`rollout-assinatura.md`](rollout-assinatura.md) (19b).

O 19c cobre atestado, declaração de comparecimento, receita comum, requisição
de exames, a lista de medicamentos em uso e a conferência por QR code. Tudo
fica atrás do interruptor `clinical_documents`. Controlado e antimicrobiano
digital ficam para o 19d.

## 1. Publicação

Ordem obrigatória. O api vai **antes** dos frontends.

1. `contracts` com a tag `clinical-v1.1.0`: o esquema
   `rotasaude.clinical_document.v1`, que é o JSON canônico que se assina.
2. `api` (`db8f04a` ou posterior). **Imagem nova**: o QR code usa a gem
   `rqrcode_core`. Por cidade, antes de cortar o tráfego:
   1. publique a imagem;
   2. faça backup dos bancos (plataforma e cidades);
   3. pare o worker;
   4. rode `bin/rails db:migrate` na plataforma. A migração `20261009600001`
      cria o catálogo de medicamentos e as listas Anvisa. **Não existe**
      `db:migrate:platform`;
   5. rode `bin/rails city:migrate:all` **da imagem nova**. A
      `20261009600001` de cidade cria documentos, itens de receita,
      medicamentos em uso e seus eventos, REMUME, protocolos de enfermagem e
      tentativas de conferência, e põe `cnpj` no perfil da cidade. É
      **irreversível**;
   6. religue o worker.
3. Carga do catálogo, na imagem publicada, **nesta ordem** (as marcas de
   antimicrobiano e controlado saem das listas):
   1. `bin/rails medications:import_anvisa`;
   2. `bin/rails medications:import_catmat`.

   Produção não tem maintenance: a carga é sempre por estas tarefas. Rodar de
   novo enquanto outra importação corre devolve `import_in_progress` (uma
   importação presa há mais de 1 h passa a `failed`).
4. **Proxy do host de cada cidade** encaminhando `/v` ao api, como já faz com
   `/r`. Sem isso o QR code dos documentos cai em 404.
5. `dashboard` (`74791e0` ou posterior), depois do api.
6. `maintenance` (`bc51914` ou posterior), depois do api.

Publicar não muda nenhuma cidade: `clinical_documents` nasce desligado.

## 2. Ligar numa cidade

1. Pré-requisito: `clinical_record` ligado (modo `record`, 19a).
2. O `municipal_admin` prepara a cidade no dashboard, painel **Documentos
   clínicos** (escritas com step-up):
   - **CNPJ** da secretaria ou do fundo municipal de saúde. Aceita CNPJ
     numérico ou alfanumérico (IN RFB 2.229/2024), com dígito verificador. Sem
     ele a enfermagem não prescreve (`city_cnpj_missing`);
   - **REMUME**: itens do catálogo que a rede tem, opcionalmente por unidade.
     Eles aparecem primeiro na busca;
   - **protocolos de enfermagem**: título, número, ano, vigência (início e fim
     opcional), itens e dose máxima por item (quantidade e unidade). Número e
     ano repetidos dão `already_exists`; versão nova entra pelo próprio
     protocolo. Controlado não entra.
3. O mantenedor liga `clinical_documents` no `maintenance` (aba
   Funcionalidades). Faça isso só depois do passo 2 (rotasaude/api#60).

Com o interruptor desligado, CNPJ, REMUME e protocolos continuam editáveis, e
imprimir e cancelar documentos já emitidos continua permitido.

## 3. Quem emite o quê

O CBO é o do vínculo com que a autora atendeu a consulta.

| Documento | Médico (2251–2253) | Dentista (2232) | Enfermeiro (2235) | Outros CBOs | Recepção |
|---|---|---|---|---|---|
| Atestado | Sim | Sim | Não | Não | Não |
| Receita comum | Sim | Sim | Só itens do protocolo vigente, com CNPJ e Coren | Não | Não |
| Requisição de exames | Sim | Sim | Sim | Sim | Não |
| Declaração de comparecimento | Sim | Sim | Sim | Sim | Sim |

- **Origem:** documento clínico nasce só da consulta, pela autora, também
  depois de finalizada. A declaração também sai pelo atendimento.
- **Atestado:** CID só com autorização do paciente; o de acompanhante não tem
  dias, aceita início opcional (até 30 dias antes) e não leva CID.
- **Enfermagem:** sem texto livre; item fora do protocolo vigente dá
  `not_in_nursing_protocol`, dose acima da máxima (ou em outra unidade) dá
  `above_protocol_max_dose`.
- **Antimicrobiano:** sai só em **papel**, em 2 vias, com validade de 10 dias,
  até o 19d. O aviso aparece antes de emitir.
- **Controlado:** bloqueado (`controlled_not_allowed`) até o 19d.
- **Requisição de exames:** só depois de finalizar a consulta, com os exames
  da consulta e dos adendos (`no_exam_requests` quando não há nenhum).
- **Declaração pela recepção** (`citizen_verifier`): sai sempre em papel, sem
  assinatura digital, com "Emitido pela recepção" e a matrícula, nunca o
  e-mail. Vale para a **cidade inteira**: a recepção de outra unidade também
  emite (restringir à unidade é rotasaude/api#59). Atendimento encerrado há
  mais de 30 dias dá `attendance_not_found_or_closed_long_ago`.

## 4. Assinatura

Reaproveita o 19b. O modo é escolhido na emissão e **nunca** é convertido:

- autora com certificado ativo e `digital_signature` utilizável: o pedido de
  assinatura nasce na emissão e sai na fila `signatures`, como a consulta;
- nos demais casos (e sempre no antimicrobiano e na declaração da recepção):
  papel, com espaço para a assinatura à mão.

Um pedido de documento cancelado sai da fila (`document_cancelled`); "Voltar
ao papel" passa o documento a `paper`. O PSC simulado só existe fora de
produção e todo documento assinado com ele diz "simulada — sem validade
jurídica" (§5 do runbook do 19b).

## 5. Impresso e página pública `/v`

- Todo impresso leva **QR code** no fim de cada via e o **código curto**
  `XXXXX-XXXXX`. Documento digital imprime o PAdES gravado, sem revalidar;
  documento aguardando assinatura dá `awaiting_signature`.
- A página `/v` fica no host da cidade, **sem login** e sem depender de
  interruptor. Mostra só tipo, data, profissional, unidade, iniciais e ano de
  nascimento do paciente, situação (válido, cancelado, aguardando assinatura)
  e modo. Nunca mostra CID, medicamento, dias de afastamento ou CPF.
- Busca manual por código curto + ano de nascimento (4 dígitos). Qualquer erro
  devolve o mesmo 404, para não revelar se o código existe.
- Limites por IP, por 10 min: 10 buscas, 60 aberturas pelo QR, 30 downloads de
  `signed.pdf`. Estourou: 429 `rate_limited`.
- `signed.pdf` só existe para documento digital, assinado e não cancelado.
- O token de `/v` e `/r` sai mascarado no log do api; o access log do proxy
  precisa do mesmo cuidado (§9, rotasaude/api#62).

## 6. Cancelamento, medicamentos em uso e leitura administrativa

- **Cancelamento:** só a autora, com motivo de 10 a 500 caracteres e step-up.
  O documento passa a "cancelado" e a página pública mostra isso. "Cancelar e
  emitir outro" emite um documento novo que aponta o cancelado.
- **Medicamentos em uso:** estado atual reconstruído de eventos (só
  acréscimos), alimentado pela receita, pelo medicamento externo e por
  suspender/reativar na consulta em rascunho. A leitura segue a regra do
  prontuário, com trilha; renovar só dentro da consulta em andamento.
- **Leitura administrativa:** o `municipal_admin` lê os documentos de uma
  consulta com step-up, gravado em `clinical_record_administrative_reads` e na
  trilha `administrative`. Não imprime.

## 7. Maintenance: tela Medicamentos

- **Importar CATMAT:** classe 6505 da API aberta do Compras.gov, cerca de
  25 s, 6.778 itens. Cerca de 44% ficam **ocultos para revisão** (o analisador
  não entendeu a descrição); a revisão é só leitura.
- **Listas Anvisa:**
  - antimicrobianos da IN 360/2025: 141 linhas (131 oficiais mais 10 grafias
    DCB);
  - controlados da Portaria 344, atualização 101 (RDC 1.036/2026); C4 não
    existe no texto vigente.

  As listas não têm API aberta: são arquivos transcritos em
  `config/medications/anvisa` do api.
- As mutations `importMedicationCatalog` e `importAnvisaLists` têm timeout
  GraphQL próprio de **300 s** (as demais seguem em 10 s). Falha devolve
  `ok: false` com `source_unavailable`, `incomplete_source`,
  `suspicious_drop`, `import_in_progress` ou `import_failed`.

## 8. Validação

- `bin/rails medications:import_anvisa` imprime as contagens das listas, e
  `medications:import_catmat` a diferença do catálogo, sem `abort`.
- Na cidade ligada: emitir uma receita de teste com item da REMUME, abrir o QR
  no celular fora da rede interna e conferir que `/v` responde pelo host da
  cidade sem dado clínico; cancelar e ver "cancelado" na página.
- O access log do proxy não traz o token de `/v` e `/r`.

## 9. Gates e pendências antes de uma cidade real

| Item | Bloqueia? |
|---|---|
| rotasaude/api#56 — conferência farmacêutica das listas Anvisa (o nº 42 falta na fonte; benzilpenicilina não casa com penicilina G). Controlado fora da lista passa na receita | **Sim** (Crítica) |
| rotasaude/api#57 — curadoria do catálogo CATMAT (itens ocultos para revisão) | **Sim** (Alta) |
| rotasaude/api#62 — access log do proxy sem o token de `/v` e `/r` (o token abre o `signed.pdf`) | **Sim** |
| rotasaude/api#62 — `trusted_proxies` configurado, senão o limite por IP vê só o IP do proxy | **Sim** |
| Norma da cidade para a declaração emitida pela recepção | **Sim** |
| `max_connections` com o worker `signatures` de cada cidade (já do 19b) | **Sim** |
| rotasaude/api#58 — RNDS e OBM | Não |
| rotasaude/api#59 — recepção listar, reimprimir e cancelar declarações, e restringir à unidade do atendimento | Não |
| rotasaude/api#60 — preparar REMUME e protocolos antes de ligar o interruptor | Não |
| rotasaude/api#61 — histórico de versões dos protocolos de enfermagem | Não |

## ADRs relacionados

0028 (interruptores, terminologias), 0031 (prontuário, leitura), 0032
(assinatura), 0033 (documentos clínicos).
