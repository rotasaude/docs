# Rollout — laboratório municipal (módulo 21b, ADR 0037)

## Pré-requisitos
- Tags do contracts publicadas: `lab-v1.0.0` e `exams-v1.1.0` (cópias em `apps/api/config/lab/lab-report-v1.json`
  e `spec/fixtures/exams/exam-order-v1.json`, SHA-256 preso).
- 21a em produção; `exam_tracking` utilizável na cidade.
- Imagem do api **reconstruída** com a gem `barby` 0.7.0 (B41: está no `Gemfile` e no `Gemfile.lock`; o
  `bundle install` acontece no build da imagem — container antigo não tem a gem).

## Ordem de deploy
1. contracts (tags) → 2. api: publicar a imagem e rodar `bin/migrate` dela ANTES de cortar tráfego
   (`db:migrate` da plataforma — 20261015900001 —, `platform:triggers`, `city:migrate:all` — 20261015900002,
   irreversível; o deploy relê `db/city_triggers.sql`); depois `bin/rails lab:templates:load` (catálogo-base;
   idempotente; para com erro se uma entrada do catálogo for recusada por outro motivo que não SIGTAP fora da
   competência ativa).
3. dashboard → 4. maintenance → 5. catálogo público. O wpda não muda.
Nunca migrar cidade fora do rake. A migração de cidade troca checks e um índice de `exam_results`
em tabelas existentes: rode fora do pico.

## Ligar numa cidade (maintenance)
1. `municipal_lab` (exige `exam_tracking`).
2. Admin da cidade:
   - marcar o prestador `uses_municipal_lab` — o api recusa sem endereço e telefone (B45; CNES de 7 dígitos,
     telefone de 10 ou 11). Marcado, o prestador não recebe mais resultado pelo portal, FHIR nem unidade (B35);
   - marcar as UBS que coletam (`collects_samples`);
   - adotar exames e ajustar versões (método, limites, críticos, código do aparelho);
   - motivos de rejeição;
   - liberadores: um responsável técnico e um substituto, com perfil e conselho (CRF, CRBM ou CRM). Para trocar o
     RT, encerre o atual e designe o novo; entre um passo e outro a liberação de laudos fica bloqueada
     (`technical_lead_missing`, B46). Encerrar um liberador devolve os laudos ICP dele que aguardavam assinatura
     (`withdrawn`), e os itens voltam à fila de liberação.
3. `lab_hl7_inbound` só depois do middleware real homologado; gerar o token (aparece uma vez) e
   entregá-lo ao fornecedor por canal seguro. O middleware lista o que espera resultado em
   `GET /lab/hl7/pending_barcodes`, com **corte em 200** códigos por chamada (Ciclo 1): o fornecedor precisa
   consumir a fila com frequência.
4. `fake_middleware` NUNCA em produção (não existe lá); em staging, só com dados de teste.

## Gates de go-live (spec §12)
- Primeiro middleware real: mapeamento de códigos de aparelho e componentes; teste de troca de
  paciente (amostra de outro laboratório, rejeitada, versão antiga) com o fornecedor.
- Validação do sistema pelo laboratório (dossiê do art. 106 da RDC 978).
- Certificados ICP dos liberadores médicos (médico só libera com ICP); aceitação da assinatura avançada pela
  VISA local.
- Impressoras térmicas (203 dpi, etiqueta 50 x 25 mm) e leitores nas UBS e no laboratório: imprimir e
  ler uma etiqueta real de cada modelo.
- Valor SIGTAP: conferir o `VL_SA` da competência importada (Task 3, Step 1).
- Retificação de laudo de prestador externo continua em rotasaude/api#85.

## Retenção
- Mensagem HL7 bruta: 5 anos (RDC 978) e depois purga para NULL; o recibo fica (B44). O job de purga **não**
  entra no 21b: pendência rotasaude/api#86 (board #2). Até lá a coluna cresce.
- Laudos, resultados, requisições e amostras do laboratório retêm o paciente na exclusão LGPD (api#84).

## Verificação depois de ligar
- `GET /lab/context` com uma conta do laboratório; abrir uma requisição de teste com paciente de teste,
  imprimir e ler a etiqueta; `POST /hl7/v2` com `QBP^Q11` do fornecedor → `RSP^K11`.
- Conferir que nenhum valor aparece em log (`grep` do código de barras de teste no log do dia).

## Desligar
- `municipal_lab` desligado: rotas /lab respondem 403; nada é apagado; laudos liberados seguem no 21a e na
  conferência pública. Laudo ICP pendente com `digital_signature` desligado sai (`withdrawn`) e os itens voltam à
  fila. Uma falha permanente da finalização ICP também devolve o laudo (B56).
- Migrações irreversíveis: não há rollback de schema; volte só o código (expand/contract).

## Conhecido (Ciclo 1)
- A conferência de reimpressão de etiqueta lê `domain_events` sem índice próprio: acompanhe o tempo de
  `GET /lab/requisitions/:id/labels.pdf` em cidade grande.
- A semente de dev não vincula CBO ao `patologista@` (o código de patologista clínico não está em
  `config/professionals/cbo_saude.yml`).
