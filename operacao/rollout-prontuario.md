# Rollout — prontuário da APS (consulta) (módulo 19a)

Runbook do operador para publicar o módulo 19a (ADR 0031), ligar o prontuário
numa cidade, validar o cidadão com nome, usar a abertura justificada e
acompanhar as leituras. Base do modo `record` e da exportação:
[`rollout-modo-de-prontuario.md`](rollout-modo-de-prontuario.md). Fichas e
varredor do acolhimento: [`rollout-acolhimento.md`](rollout-acolhimento.md).

## 1. Publicação

Ordem obrigatória:

1. `api` (`2a3ec97` ou posterior). **A imagem precisa ser reconstruída**: o
   impresso usa a gem Prawn, e o `Gemfile.lock` rebaixou o `bigdecimal` para
   3.3.1. Por cidade, antes de cortar o tráfego:
   1. publique a imagem nova;
   2. pare o worker;
   3. **faça backup do banco da cidade**, porque as quatro migrações são
      irreversíveis;
   4. rode `bin/rails city:migrate:all` **da imagem nova**:
      - `20261007400001` cria `patients`, os nomes em `citizens` (completo,
        social, da mãe) e a lista de problemas por eventos;
      - `20261007400002` cria consultas, itens, adendos e aberturas, com as
        guardas de imutabilidade. A única exceção é a re-cifra da chave da
        cidade;
      - `20261007400003` acrescenta `correction_pending` à fila LEDI;
      - `20261007400004` cria `clinical_record_administrative_reads`, que
        guarda para sempre as leituras do administrador;
   5. religue o worker.

   Nunca migre uma cidade fora do rake, porque ela fica em 503.
2. `dashboard` (`c4f7d5a` ou posterior), **depois** do `api`.

Publicar não muda nenhuma cidade. O interruptor `clinical_record` nasce
desligado, e sem ele todas as rotas do prontuário respondem
`403 feature_disabled`.

O `Ledi::ScreeningFichaSweepJob` (de hora em hora, age às 23h no fuso da
cidade) passa a gerar também a ficha atrasada da consulta finalizada quando a
exportação estava inutilizável na finalização. Ele só alcança a competência
atual e a anterior.

## 2. Ligar o prontuário numa cidade

1. **Modo `record`.** O operador, no console (`admin`), põe a cidade em
   `record` (Cidades → Ficha). O interruptor exige isso: em outro modo, o
   `maintenance` mostra `record_mode_not_record`.
2. **Terminologias (plataforma).** A consulta usa CIAP-2 e CID-10 nos
   problemas e o grupo 02 da SIGTAP nos exames:
   ```bash
   bin/rails 'terminology:import[ciap2,<versão>,<zip ou pasta>]'
   bin/rails 'terminology:import[cid10,<versão>,<zip ou pasta>]'
   bin/rails 'terminology:import[sigtap,AAAAMM,<zip ou pasta>]'
   ```
   As listas da semente de dev são recortes e não valem fora do dev.
3. **Interruptor.** O mantenedor, no `maintenance`, liga `clinical_record` na
   cidade.
4. **Quem registra a consulta.** Só quem tem vínculo ativo na unidade (módulo
   10) com CBO da Tabela 3 do MIAI, exceto o dentista (`2232`). O técnico de
   enfermagem (`3222`) não registra consulta.
5. **CID-10 só para médicos** (grupos `2251`, `2252` e `2253`), como no PEC.
   - Os demais profissionais registram problemas em CIAP-2.
   - Eles podem avaliar ou resolver um CID-10 que já está na lista.
   - A ficha deles sai sem os problemas em CID-10.
   - Por isso, o não médico precisa avaliar ao menos um problema em CIAP-2 para
     finalizar (`422 ciap2_required_for_cbo`).

## 3. Validação presencial com nome

O prontuário é por CPF e só aceita par **validado** no balcão. A consulta exige
o nome completo do paciente.

- **Validação nova** (Atendimento → validação presencial): a recepção informa
  o nome completo (3 a 200 caracteres), o nome social e o nome da mãe
  (opcionais).
- **Par validado antes do 19a, sem nome:** no check-in, a tela oferece
  "completar nomes" (`citizen_verifier`). Até completar, a consulta recusa com
  `patient_name_missing`.
- O nome de exibição é o social, se houver, senão o completo. A fila da
  unidade mostra só o nome de exibição e a cor.
- O paciente nasce na primeira consulta. Antes dela não há prontuário.

## 4. Quem lê o quê

| Quem | Lê | Imprime | Adendo |
|---|---|---|---|
| Autora da consulta finalizada | Sempre, por "Minhas consultas" ou pelo atendimento | Sim | Sim |
| Outro profissional | Com o paciente em atendimento com ele (ou na fila da unidade dele), ou por abertura justificada | Não | Não |
| `municipal_admin` | "Consultas por profissional", conteúdo completo, com step-up | Não | Não |
| Recepção | Nunca | — | — |

Toda leitura de conteúdo clínico deixa trilha (`clinical_record.viewed`, com
`access` = `in_context`, `justified`, `author` ou `administrative`). As listas
("Minhas consultas" e a lista por profissional) não têm conteúdo clínico e não
deixam trilha.

## 5. Abertura justificada

Para ler o prontuário de quem não está em atendimento:

1. Prontuário → "Abrir prontuário fora do atendimento".
2. CPF do paciente e motivo: revisão de caso, busca ativa, continuidade do
   cuidado ou outro (com descrição de 10 a 500 caracteres, sem dado pessoal).
3. Step-up (código do autenticador).
4. A abertura vale 30 minutos. Quando vence, a tela apaga o prontuário e
   oferece abrir de novo.

CPF inexistente e CPF sem paciente respondem igual (`404 patient_not_found`),
para não revelar quem tem prontuário.

## 6. Acompanhar

- **Relatório "Aberturas fora de contexto"** (Prontuário, `municipal_admin`):
  aberturas justificadas e leituras administrativas, com a coluna "Tipo", até
  500 linhas, mais novas primeiro. O relatório mostra o motivo da abertura, mas
  nunca a descrição. Aberturas e leituras administrativas ficam guardadas para
  sempre, em tabelas próprias.
- **Produção e-SUS:** a ficha de Atendimento Individual da consulta
  (`source_type: "Consultation"`) entra nas mesmas listas da escuta: "não
  geradas" (com motivo e "gerar de novo"), recusas e "Reenviar". Adendo depois
  do aceite cria uma ficha `correction_pending`, que fica fora das contagens e
  não é enviada até a api#41 provar o reenvio após aceite.
- **Impresso:** PDF para assinatura manual, só da autora, até a assinatura
  digital do 19b.

## 7. Gates e pendências antes de uma cidade real

| Item | Bloqueia? |
|---|---|
| rotasaude/api#41 — aceite (2xx) e reenvio depois do aceite num PEC com CNES real | **Sim** |
| rotasaude/api#51 — atendimento preso quando a autora sai com rascunho aberto | **Sim** (Alta, antes do go-live) |
| CIAP-2, CID-10 e SIGTAP importados dos arquivos oficiais | **Sim** |
| Imagem do api reconstruída (Prawn) | **Sim** |
| rotasaude/api#52 — adendo que não chega à ficha em corrida rara, sem registro | Não (Média) |
| rotasaude/dashboard#12 — lista de problemas do paciente no adendo de "Minhas consultas" | Não (Baixa) |
