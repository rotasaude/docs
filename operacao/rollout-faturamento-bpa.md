# Rollout — procedimentos e faturamento BPA (módulo 22, ADR 0039)

## Pré-requisitos
- SIGTAP da competência importada na plataforma (módulo 16; `terminology:import` com o ZIP do DATASUS) — sem ela,
  a crítica usa a última ativa (aviso `sigtap_fallback`) ou marca `not_in_sigtap`.
- Cada unidade não-APS (UPA, hospital, outro) com CNES no cadastro (módulo 09); o laboratório municipal com CNES (21a/21b).
- Profissionais com perfil (CNS com DV válido) e vínculo com CBO na unidade (módulo 10); liberadores do laboratório
  com vínculo em alguma unidade (o CBO do laudo vem dele).
- IBGE da cidade no perfil (`city_profile.ibge_code`) — sem ele o interruptor fica "ligado, falta pré-requisito" e o
  BPA-I sai com o IBGE em brancos.
- Tag do contracts publicada: `billing-v1.0.0` (perfil e exemplos golden, copiados em `apps/api/spec/fixtures/billing/`).
- Nenhuma gem nova; nenhuma migração de plataforma.

## Ordem de deploy
1. contracts (tag `billing-v1.0.0`) → 2. api: publicar a imagem e rodar `bin/migrate` dela ANTES de cortar tráfego
   (`city:migrate:all` — 20261021900001, irreversível; papel `billing_clerk`, `performed_procedures`, `billing_periods`,
   `billing_exports`, `billing_settings` e triggers em `db/city_triggers.sql`).
3. dashboard → 4. maintenance → 5. catálogo público. Nunca migrar cidade fora do rake.

## Ligar numa cidade
1. maintenance: `procedure_billing` ("Faturamento de procedimentos (BPA)"); vale nos dois modos de prontuário.
2. Admin municipal: configuração do faturamento (órgão de origem até 30, sigla até 6, CNPJ só numérico, órgão de
   destino até 40, indicador M/E) — com step-up. As larguras contam o texto já como sai no arquivo (ASCII maiúsculo,
   sem acento e sem dígito: "Æ" vira "AE"); passou, `too_long` com o campo (§13 E20).
3. Admin municipal: convidar/conceder `billing_clerk` (papel privilegiado: step-up; o mantenedor não concede). O
   faturista cadastra o TOTP no primeiro acesso, **antes do primeiro fechamento**: fechar, reabrir e baixar exigem
   step-up, e sem TOTP cadastrado o fechamento do mês fica parado.
4. Recepção: ao validar o par ou completar o cadastro, preencher raça/cor (sem "sem informação"; etnia se indígena) e
   endereço (CEP, logradouro, número ou SN, complemento, bairro) — D-U1. Quem não tem fica pendente no fechamento
   (`missing_race_color`, `missing_address`) e o faturista corrige em "dados do paciente".
5. A partir daí: procedimentos no atendimento, na consulta e avulsos; os laudos liberados do laboratório municipal
   viram produção automaticamente. **Sem retroativo:** laudo liberado com o interruptor desligado não entra; procedimento
   anterior ao interruptor entra só por lançamento manual do faturista.

## Rotina mensal do faturista
1. Conferir o `BPA_LEIAME.txt` vigente em https://sia.datasus.gov.br/versao/listar_ftp_bpa.php antes de fechar: versão
   nova do BPA Magnético com mudança de layout (posição/tamanho de campo): **suspenda o fechamento** até o api ser atualizado
   (o layout implementado é o de 09/07/2026, BPA 05.00 — SHA-256 do PDF no perfil do contracts).
2. Competências: o filtro "abertas" (`GET /billing/periods?status=open`) mostra todas as abertas, de qualquer mês (§13
   E18). O resumo da competência aberta vem do mesmo recálculo das pendências (bate com a lista; §13 E17).
3. Pendências da competência (por CNES); corrigir cadastro (CNS, CBO, CNES, perfil do paciente) — o fechamento recalcula.
   A lista de procedimentos mostra nome de paciente e grava trilha de leitura (`billing_procedures.listed`, só ids: quem
   leu e os procedimentos da página; §13 E19).
4. Fechar (step-up); baixar o arquivo (conteúdo que não bate com o SHA-256 gravado não sai: 409 `integrity_mismatch` —
   reabra e feche de novo); importar no BPA Magnético ("importar arquivos de sistemas próprios"); consistir
   e exportar a remessa para a secretaria.
5. Erro no arquivo: reabrir com motivo (gera nova versão; a anterior fica no histórico).
6. Competência antiga é atrasada (§13 E18): procedimento com competência igual ou anterior à **última fechada** do
   estabelecimento — mesmo de um mês que nunca teve competência própria — entra na primeira competência aberta depois da
   última fechada, com `late: true` (a data aceita vai até o 1º dia de 3 competências antes). Nunca se abre competência
   no passado. Vale também para o laboratório (coleta antiga).

## Gates de go-live (spec §12)
- **Importação real no BPA Magnético** (VM Windows ou secretaria piloto) de um arquivo gerado em staging: sem rejeição
  de layout; registrar o resultado da consistência. Pontos a conferir ali: `prd-org` `BPA` × `EXT` (F6), código de
  logradouro em brancos, INE em brancos em unidade sem equipe. Raça/cor e endereço vêm do cadastro (D-U1): a
  recepção ou o faturista completam; quem não tem fica pendente (`missing_race_color`, `missing_address`).
- CNS e CBO de todos os profissionais e CNES de todas as unidades faturantes (módulos 09 e 10).
- Transição para RAC/CMD (RNDS) sem data: o BPA é ponte.
- Volume: mais de 999 folhas num estabelecimento recusa o fechamento (`too_many_sheets`). O teto real fica **abaixo de
  19.980 linhas** (999 × 20): o BPA-I abre folha nova a cada profissional (CNS + CBO), então cada profissional deixa uma
  folha incompleta, e as folhas do BPA-C e do BPA-I somam no mesmo limite de 999. Acompanhar UPAs grandes no primeiro
  mês (BPA-I aceita 99 linhas por folha no layout — mudança a decidir se acontecer).

## Verificação depois de ligar
- **Só em STAGING:** um procedimento de teste numa UPA (faturável) e numa UBS (`establishment_is_aps`, sem competência);
  `GET /billing/periods` com o faturista; pendências; nenhum CPF/nome/arquivo no log do dia (`grep` do CPF de teste).
- **Em produção, nada de procedimento de teste:** o procedimento é só acréscimo (não se apaga), entraria no BPA real da
  competência e reteria o paciente na exclusão LGPD para sempre. Verifique só leitura: as rotas `/billing` respondem
  com o faturista (`GET /billing/periods?status=open`, `GET /billing/procedures`) e a lista vem vazia antes da primeira
  produção.

## Desligar
- `procedure_billing` desligado: rotas `/billing` respondem 403; atendimento e consulta não mostram procedimentos; a
  linha do tempo para de mostrar o tipo `procedure`; laudos liberados deixam de virar produção. Os registros e arquivos
  ficam (só acréscimo).
- Migração irreversível: não há rollback de schema; volte só o código (expand/contract).

## Riscos conhecidos
- **Janela residual do E12:** com dois escritores concorrentes no mesmo paciente (ex.: recepção e faturista), o par pode
  ficar validado enquanto a resposta de um deles é 422. Nenhum dado inválido persiste; o paciente só precisa repetir ou
  completar o cadastro.
- **Pontos do layout a conferir na importação real (gate de go-live):** `prd-org` `BPA` × `EXT`, código de logradouro
  (DNE) em brancos e INE em brancos.
- **`billing_export_id` = "incluído no arquivo vigente":** o re-fechamento limpa o id dos procedimentos que não entraram na
  nova versão; o histórico "estava na v1" fica só no arquivo da v1.
- **CNPJ alfanumérico recusado:** o campo do layout é numérico (NUM); CNPJ com letras não pode ser configurado.
- **Dev:** só é migrado e semeado na prova no navegador. A migração de cidade é irreversível e deixa os bancos de dev à
  frente da main (a parity spec pode quebrar em main até o DROP + `city:test_databases`).
